# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Event Venue Management · Created: 2026-05-22

## Philosophy

This model keeps core, frequently queried fields in typed relational columns while using PostgreSQL JSONB columns for variable, venue-type-specific, and rapidly evolving data. The "stable centre, flexible edges" principle means that universal fields (dates, amounts, statuses, foreign keys) are always relational, but fields that vary by venue type, event category, region, or customer preference live in indexed JSONB columns.

This is the architecture increasingly adopted by modern SaaS platforms that must serve diverse customer segments with a single codebase. A wedding venue needs fields for ceremony type, officiant, and bridal suite access times. A convention centre needs session tracks, exhibitor booth assignments, and badge printing specs. A restaurant event space needs table turn times and prix fixe menu selections. Rather than creating nullable columns or EAV tables for every permutation, the hybrid model puts the variable parts in JSONB with GIN indexes for efficient querying.

PostgreSQL's JSONB support has matured to the point where JSONB columns offer full ACID guarantees, GIN indexing for containment queries, partial indexes on JSONB paths, and JSON Schema validation via CHECK constraints. This makes JSONB suitable for production use, not just prototyping. The approach is recommended by AWS for "semi-structured payloads" and by the PostgreSQL community for fields that "vary across record types."

**Best for:** Platforms serving diverse venue types (weddings, corporate, conventions, restaurants) that need rapid feature development without constant schema migrations, and MVP-stage products that expect the data model to evolve significantly.

**Trade-offs:**
- (+) Rapid development — new fields added without schema migrations
- (+) Venue-type-specific data handled naturally without nullable column sprawl
- (+) Fewer tables (25-30 vs 37+ for fully normalised)
- (+) Easy to support custom fields per organisation
- (+) JSON Schema validation in CHECK constraints provides type safety
- (+) GIN indexes make JSONB containment queries fast
- (-) JSONB fields are invisible to the database's type system until extracted
- (-) Reporting on JSONB fields requires extraction (`->>`), which is less ergonomic than column references
- (-) Risk of "everything in JSONB" anti-pattern if discipline is not maintained
- (-) JSONB field evolution requires application-level migration logic
- (-) Full-text search on JSONB content needs explicit extraction and indexing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| APEX/ANSI Event Industry Standards | Core APEX fields are relational columns; venue-specific APEX extensions stored in JSONB |
| iCalendar (RFC 5545) | `ical_uid` is a relational column; extended iCal properties stored in `calendar_data` JSONB |
| ISO 3166-1/2 | `country_code` and `region_code` are typed CHAR columns |
| ISO 4217 | `currency` is a typed CHAR(3) column on all financial tables |
| ISO 20121:2024 | Sustainability metrics stored in a flexible JSONB `metrics` column to accommodate evolving ISO 20121:2024 category definitions |
| PCI DSS | Stripe references in relational columns; no card data anywhere |
| GDPR / CCPA | Consent fields are relational for query efficiency; custom privacy preferences in JSONB |
| JSON Schema (Draft 2020-12) | CHECK constraints validate JSONB column structure using pg_jsonschema extension |
| OpenAPI 3.1 | API resource schemas define which fields are relational vs JSONB, documented in OpenAPI components |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    billing_email   TEXT,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    country_code    CHAR(2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    -- JSONB: org-specific settings that vary widely
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "branding": {"logo_url": "...", "primary_color": "#2563eb"},
    --   "booking_defaults": {"deposit_pct": 25, "cancellation_policy": "..."},
    --   "tax_config": {"default_rate": 0.085, "tax_id_label": "EIN"},
    --   "notification_prefs": {"email_beo_approved": true, "sms_payment_received": false},
    --   "custom_fields_config": {"booking": [...], "event": [...], "contact": [...]}
    -- }
    stripe_customer_id TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    full_name       TEXT NOT NULL,
    phone           TEXT,
    password_hash   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    consent_status  TEXT NOT NULL DEFAULT 'pending',
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example preferences:
    -- {
    --   "timezone": "America/New_York",
    --   "date_format": "MM/DD/YYYY",
    --   "notification_channels": ["email", "in_app"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    role            TEXT NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example: ["bookings:read", "bookings:write", "invoices:read", "beo:approve"]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);
```

---

## Venues & Spaces

```sql
CREATE TABLE venues (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    venue_type      TEXT NOT NULL,      -- hotel, restaurant, convention_centre, corporate_campus, wedding_venue, university
    -- Core address fields as relational columns for geospatial queries
    address_line1   TEXT,
    city            TEXT,
    region_code     TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: venue-type-specific attributes
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example for wedding venue:
    -- {
    --   "ceremony_options": ["indoor", "outdoor_garden", "chapel"],
    --   "max_ceremonies_per_day": 2,
    --   "bridal_suite": true,
    --   "rehearsal_dinner_available": true,
    --   "preferred_vendors": ["uuid1", "uuid2"],
    --   "parking_capacity": 200,
    --   "accessibility": {"wheelchair": true, "hearing_loop": true},
    --   "alcohol_license": "full_bar",
    --   "noise_curfew": "23:00",
    --   "sustainability_certifications": ["ISO_20121"]
    -- }
    --
    -- Example for convention centre:
    -- {
    --   "loading_dock": true,
    --   "freight_elevator": true,
    --   "exhibitor_services": true,
    --   "badge_printing": true,
    --   "fiber_internet_gbps": 10,
    --   "backup_generator": true,
    --   "union_labor_required": true
    -- }
    contact_info    JSONB NOT NULL DEFAULT '{}',
    -- {"phone": "...", "email": "...", "website": "...", "social": {"instagram": "...", "facebook": "..."}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_venues_org ON venues(org_id);
CREATE INDEX idx_venues_type ON venues(org_id, venue_type);
CREATE INDEX idx_venues_attributes ON venues USING gin(attributes);

CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    venue_id        UUID NOT NULL REFERENCES venues(id),
    parent_space_id UUID REFERENCES spaces(id),
    name            TEXT NOT NULL,
    capacity_max    INT NOT NULL,
    area_sqft       NUMERIC(10,2),
    is_outdoor      BOOLEAN NOT NULL DEFAULT false,
    is_divisible    BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Core pricing as relational columns
    hourly_rate     NUMERIC(12,2),
    half_day_rate   NUMERIC(12,2),
    full_day_rate   NUMERIC(12,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    -- JSONB: setup configurations, amenities, and space-specific attributes
    setup_configs   JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"style": "Theatre", "capacity": 300, "floor_plan_url": "...", "setup_mins": 45, "teardown_mins": 30},
    --   {"style": "Banquet", "capacity": 200, "floor_plan_url": "...", "setup_mins": 60, "teardown_mins": 45},
    --   {"style": "Classroom", "capacity": 150, "floor_plan_url": "...", "setup_mins": 40, "teardown_mins": 25},
    --   {"style": "U-Shape", "capacity": 60, "floor_plan_url": "...", "setup_mins": 30, "teardown_mins": 20}
    -- ]
    amenities       JSONB NOT NULL DEFAULT '[]',
    -- ["WiFi", "projector", "whiteboard", "video_conferencing", "natural_light", "blackout_curtains"]
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- {"floor_level": 2, "ceiling_height_ft": 14, "columns": false, "natural_light": true}
    calendar_sync   JSONB NOT NULL DEFAULT '{}',
    -- {"provider": "google_calendar", "calendar_id": "...", "sync_direction": "bidirectional"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spaces_venue ON spaces(venue_id);
CREATE INDEX idx_spaces_amenities ON spaces USING gin(amenities);
CREATE INDEX idx_spaces_setup ON spaces USING gin(setup_configs);
```

---

## CRM — Accounts & Contacts

```sql
CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL DEFAULT 'corporate',
    -- Core fields as relational columns
    email           TEXT,
    phone           TEXT,
    website_url     TEXT,
    city            TEXT,
    region_code     TEXT,
    country_code    CHAR(2),
    source          TEXT,
    lifetime_value  NUMERIC(14,2) DEFAULT 0,
    -- JSONB: flexible account-type-specific fields
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Corporate: {"industry": "tech", "company_size": "500-1000", "billing_department": "events@corp.com"}
    -- Wedding: {"wedding_planner": "Jane Doe", "wedding_date": "2026-10-15", "couple_names": "Alex & Jordan"}
    -- Non-profit: {"ein": "12-3456789", "tax_exempt": true, "mission": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_accounts_org ON accounts(org_id);
CREATE INDEX idx_accounts_org_type ON accounts(org_id, account_type);
CREATE INDEX idx_accounts_attrs ON accounts USING gin(attributes);

CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    account_id      UUID REFERENCES accounts(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    job_title       TEXT,
    role            TEXT,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    consent_status  TEXT NOT NULL DEFAULT 'pending',
    consent_date    TIMESTAMPTZ,
    data_retention_until DATE,
    -- JSONB: custom contact fields, communication preferences
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- {"preferred_contact_method": "email", "language": "es", "dietary_restrictions": ["vegan"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_org ON contacts(org_id);
CREATE INDEX idx_contacts_account ON contacts(account_id);
CREATE INDEX idx_contacts_email ON contacts(email);

CREATE TABLE leads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    venue_id        UUID REFERENCES venues(id),
    contact_id      UUID REFERENCES contacts(id),
    account_id      UUID REFERENCES accounts(id),
    source          TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'new',
    event_type      TEXT,
    preferred_date  DATE,
    guest_count_est INT,
    budget_est      NUMERIC(12,2),
    assigned_to     UUID REFERENCES users(id),
    ai_score        NUMERIC(5,2),
    converted_booking_id UUID,
    -- JSONB: source-specific lead data, AI analysis, custom form fields
    source_data     JSONB NOT NULL DEFAULT '{}',
    -- Website form: {"form_id": "wedding-inquiry", "utm_source": "google", "page_url": "..."}
    -- Phone: {"call_duration_mins": 12, "call_recording_url": "..."}
    ai_analysis     JSONB NOT NULL DEFAULT '{}',
    -- {"score_reasons": [...], "suggested_spaces": [...], "similar_past_bookings": [...]}
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Org-defined custom fields: {"referral_source_detail": "Aunt Martha", "color_theme": "sage green"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leads_org_status ON leads(org_id, status);
CREATE INDEX idx_leads_venue ON leads(venue_id);
```

---

## Bookings & Events

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    venue_id        UUID NOT NULL REFERENCES venues(id),
    account_id      UUID NOT NULL REFERENCES accounts(id),
    contact_id      UUID NOT NULL REFERENCES contacts(id),
    lead_id         UUID REFERENCES leads(id),
    booking_number  TEXT NOT NULL,
    name            TEXT NOT NULL,
    event_type      TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'tentative',
    guest_count     INT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    assigned_to     UUID REFERENCES users(id),
    -- JSONB: event-type-specific booking details
    details         JSONB NOT NULL DEFAULT '{}',
    -- Wedding:
    -- {
    --   "ceremony_type": "outdoor_garden",
    --   "rehearsal_dinner": true,
    --   "rehearsal_date": "2026-09-14",
    --   "color_scheme": "sage and gold",
    --   "officiant_name": "Rev. Johnson",
    --   "photographer": "Studio XYZ",
    --   "florist": "Petals & Stems",
    --   "band_dj": "DJ Smooth",
    --   "wedding_party_size": 12,
    --   "special_moments": ["first_dance", "father_daughter_dance", "cake_cutting"]
    -- }
    --
    -- Corporate conference:
    -- {
    --   "session_count": 12,
    --   "keynote_speakers": ["Dr. Smith", "Prof. Jones"],
    --   "exhibitor_count": 25,
    --   "badge_printing": true,
    --   "streaming_required": true,
    --   "recording_consent": "opt_in",
    --   "breakout_rooms_needed": 4,
    --   "sponsor_tiers": [{"name": "Gold", "count": 3}, {"name": "Silver", "count": 8}]
    -- }
    --
    -- Restaurant private dining:
    -- {
    --   "course_count": 4,
    --   "wine_pairing": true,
    --   "prix_fixe_price": 125.00,
    --   "dietary_breakdown": {"vegan": 3, "gluten_free": 5, "nut_allergy": 2},
    --   "cake_cutting_fee_waived": false
    -- }
    internal_notes  TEXT,
    custom_fields   JSONB NOT NULL DEFAULT '{}',  -- org-defined custom fields
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, booking_number)
);

CREATE INDEX idx_bookings_org_status ON bookings(org_id, status);
CREATE INDEX idx_bookings_venue ON bookings(venue_id);
CREATE INDEX idx_bookings_dates ON bookings(start_date, end_date);
CREATE INDEX idx_bookings_details ON bookings USING gin(details);

CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    setup_start_at  TIMESTAMPTZ,
    teardown_end_at TIMESTAMPTZ,
    guest_count     INT,
    status          TEXT NOT NULL DEFAULT 'scheduled',
    ical_uid        TEXT,
    -- JSONB: event-specific configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "spaces": [
    --     {"space_id": "uuid", "setup_style": "Banquet", "capacity": 200, "notes": "Head table at north wall"}
    --   ],
    --   "equipment": [
    --     {"equipment_id": "uuid", "name": "Wireless Mic", "quantity": 2, "notes": "For speeches"},
    --     {"equipment_id": "uuid", "name": "Projector", "quantity": 1, "notes": "For slideshow"}
    --   ],
    --   "timeline": [
    --     {"time": "17:00", "activity": "Guests arrive, cocktail hour"},
    --     {"time": "18:00", "activity": "Dinner service begins"},
    --     {"time": "19:30", "activity": "Speeches and toasts"},
    --     {"time": "20:00", "activity": "First dance"},
    --     {"time": "20:30", "activity": "Open dancing"},
    --     {"time": "22:30", "activity": "Last call"},
    --     {"time": "23:00", "activity": "Event ends"}
    --   ]
    -- }
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_events_booking ON events(booking_id);
CREATE INDEX idx_events_dates ON events(start_at, end_at);

-- Space reservations are tracked relationally for exclusion constraint support
CREATE TABLE space_reservations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    space_id        UUID NOT NULL REFERENCES spaces(id),
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    UNIQUE (event_id, space_id),
    EXCLUDE USING gist (
        space_id WITH =,
        tstzrange(start_at, end_at) WITH &&
    ) WHERE (status = 'active')
);

CREATE INDEX idx_space_res_space ON space_reservations(space_id, start_at);
```

---

## Contracts, BEOs & Documents

```sql
CREATE TABLE contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID NOT NULL REFERENCES bookings(id),
    contract_number TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    total_amount    NUMERIC(14,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    content_html    TEXT,
    -- JSONB: signature data, template variables, terms
    signature_data  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "signer_name": "Jane Smith",
    --   "signer_email": "jane@example.com",
    --   "signed_at": "2026-06-06T14:22:00Z",
    --   "signer_ip": "192.168.1.100",
    --   "signature_image_url": "...",
    --   "countersigner_name": "Venue Manager",
    --   "countersigned_at": "2026-06-07T09:00:00Z"
    -- }
    template_vars   JSONB NOT NULL DEFAULT '{}',
    -- Variables merged into contract template: {"client_name": "...", "event_date": "...", "deposit_amount": 5000}
    terms           JSONB NOT NULL DEFAULT '{}',
    -- {"cancellation_policy": "...", "payment_schedule": [...], "liability_limit": 10000}
    sent_at         TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contracts_booking ON contracts(booking_id);

-- BEO stores structured operational data in JSONB for maximum flexibility
CREATE TABLE beos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID NOT NULL REFERENCES bookings(id),
    event_id        UUID NOT NULL REFERENCES events(id),
    beo_number      TEXT NOT NULL,
    version         INT NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'draft',
    -- Core scheduling fields as relational columns
    event_date      DATE NOT NULL,
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    guest_count     INT NOT NULL,
    guaranteed_count INT,
    room_name       TEXT NOT NULL,
    setup_style     TEXT,
    -- JSONB: the full BEO content — meal services, AV, setup, notes
    content         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "meal_services": [
    --     {
    --       "type": "reception",
    --       "time": "17:00",
    --       "items": [
    --         {"name": "Caprese Skewers", "quantity": 200, "price": 4.50, "dietary": ["vegetarian", "gluten-free"]},
    --         {"name": "Shrimp Cocktail", "quantity": 200, "price": 6.00, "dietary": []}
    --       ]
    --     },
    --     {
    --       "type": "dinner",
    --       "time": "18:30",
    --       "items": [
    --         {"name": "Filet Mignon", "quantity": 120, "price": 65.00, "dietary": []},
    --         {"name": "Pan-Seared Salmon", "quantity": 50, "price": 55.00, "dietary": ["gluten-free"]},
    --         {"name": "Mushroom Risotto", "quantity": 30, "price": 45.00, "dietary": ["vegan", "gluten-free"]}
    --       ]
    --     }
    --   ],
    --   "beverages": {
    --     "type": "open_bar",
    --     "duration_hours": 5,
    --     "includes": ["beer", "wine", "spirits", "soft_drinks"],
    --     "specialty_cocktails": ["Lavender Gin Fizz", "Old Fashioned"]
    --   },
    --   "av_requirements": [
    --     {"item": "Wireless Lapel Mic", "qty": 2, "notes": "For speeches"},
    --     {"item": "DJ Booth Power", "qty": 1, "notes": "Dedicated 20A circuit"}
    --   ],
    --   "setup_instructions": "Head table for 12 at north wall. Round tables of 10. Dance floor centre.",
    --   "parking_notes": "Valet service arranged; 150 spaces reserved in Lot B",
    --   "special_requests": "Sparkler send-off at 23:00 — confirm fire marshal approval"
    -- }
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_beos_booking ON beos(booking_id);
CREATE INDEX idx_beos_event ON beos(event_id);
CREATE INDEX idx_beos_date ON beos(event_date);

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID REFERENCES bookings(id),
    doc_type        TEXT NOT NULL,
    title           TEXT NOT NULL,
    file_url        TEXT NOT NULL,
    mime_type       TEXT,
    file_size_bytes BIGINT,
    version         INT NOT NULL DEFAULT 1,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- {"dimensions": "1920x1080", "duration_secs": 120, "page_count": 5}
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_booking ON documents(booking_id);
```

---

## Catering & Menu Management

```sql
-- Menus stored as structured JSONB — fewer tables, same expressiveness
CREATE TABLE menus (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    menu_type       TEXT NOT NULL,      -- a_la_carte, package, prix_fixe, buffet
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: the full menu structure
    content         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "categories": [
    --     {
    --       "name": "Appetisers",
    --       "items": [
    --         {
    --           "id": "uuid",
    --           "name": "Caprese Skewers",
    --           "description": "Fresh mozzarella, cherry tomatoes, basil, balsamic glaze",
    --           "price": 4.50,
    --           "unit": "per_person",
    --           "dietary_tags": ["vegetarian", "gluten-free"],
    --           "allergens": ["dairy"],
    --           "prep_time_mins": 20,
    --           "image_url": "..."
    --         }
    --       ]
    --     },
    --     {
    --       "name": "Entrees",
    --       "items": [...]
    --     }
    --   ],
    --   "packages": [
    --     {
    --       "name": "Silver Package",
    --       "price_per_person": 85.00,
    --       "min_guests": 50,
    --       "includes": ["2 appetisers", "1 entree choice", "1 dessert", "coffee/tea"],
    --       "upgrades": [
    --         {"name": "Premium bar", "additional_per_person": 25.00}
    --       ]
    --     }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_menus_org ON menus(org_id);
CREATE INDEX idx_menus_content ON menus USING gin(content);
```

---

## Equipment & AV

```sql
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    venue_id        UUID REFERENCES venues(id),
    name            TEXT NOT NULL,
    category        TEXT NOT NULL,     -- audio, video, lighting, staging, furniture
    quantity_total  INT NOT NULL DEFAULT 1,
    rental_rate     NUMERIC(10,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: equipment-specific specs
    specs           JSONB NOT NULL DEFAULT '{}',
    -- Audio: {"brand": "Shure", "model": "SM58", "wireless": true, "frequency_band": "UHF"}
    -- Video: {"resolution": "4K", "lumens": 5000, "throw_ratio": "1.2:1"}
    -- Lighting: {"type": "LED", "watts": 150, "dmx_channels": 12, "color_temp_k": 3200}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_org ON equipment(org_id);
CREATE INDEX idx_equipment_venue ON equipment(venue_id);
```

---

## Invoicing & Payments

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID NOT NULL REFERENCES bookings(id),
    account_id      UUID NOT NULL REFERENCES accounts(id),
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    subtotal        NUMERIC(14,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(14,2) NOT NULL DEFAULT 0,
    service_charge  NUMERIC(14,2) NOT NULL DEFAULT 0,
    discount_amount NUMERIC(14,2) NOT NULL DEFAULT 0,
    total           NUMERIC(14,2) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    due_date        DATE,
    -- JSONB: line items stored inline — avoids separate table
    line_items      JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"description": "Grand Ballroom (full day)", "category": "space_rental", "qty": 1, "unit_price": 8500.00, "total": 8500.00, "taxable": true},
    --   {"description": "Dinner service (200 guests)", "category": "catering", "qty": 200, "unit_price": 75.00, "total": 15000.00, "taxable": true},
    --   {"description": "AV package", "category": "av_equipment", "qty": 1, "unit_price": 2500.00, "total": 2500.00, "taxable": true}
    -- ]
    -- JSONB: tax breakdown for multi-jurisdiction
    tax_breakdown   JSONB NOT NULL DEFAULT '{}',
    -- {"state_tax": {"rate": 0.06, "amount": 1560.00}, "city_tax": {"rate": 0.025, "amount": 650.00}}
    sent_at         TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, invoice_number)
);

CREATE INDEX idx_invoices_booking ON invoices(booking_id);
CREATE INDEX idx_invoices_org_status ON invoices(org_id, status);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    amount          NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    payment_method  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    stripe_payment_intent_id TEXT,
    reference       TEXT,
    -- JSONB: payment-method-specific metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Credit card: {"last_four": "4242", "brand": "visa", "stripe_charge_id": "ch_..."}
    -- Bank transfer: {"bank_name": "Chase", "routing_number": "021000021", "confirmation_number": "..."}
    -- Cheque: {"cheque_number": "1042", "bank": "Wells Fargo", "cleared_at": "2026-06-15"}
    paid_at         TIMESTAMPTZ,
    refunded_at     TIMESTAMPTZ,
    refund_amount   NUMERIC(14,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_stripe ON payments(stripe_payment_intent_id);
```

---

## Tasks, Pricing & Sustainability

```sql
CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID REFERENCES bookings(id),
    event_id        UUID REFERENCES events(id),
    title           TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    priority        TEXT NOT NULL DEFAULT 'medium',
    assigned_to     UUID REFERENCES users(id),
    due_at          TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    -- JSONB: checklist items, dependencies
    checklist       JSONB NOT NULL DEFAULT '[]',
    -- [{"item": "Confirm florist delivery time", "done": true}, {"item": "Print place cards", "done": false}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_booking ON tasks(booking_id);
CREATE INDEX idx_tasks_assigned ON tasks(assigned_to, status);

CREATE TABLE pricing_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    venue_id        UUID REFERENCES venues(id),
    space_id        UUID REFERENCES spaces(id),
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: flexible rule definitions for AI-powered dynamic pricing
    rule_config     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "type": "seasonal",
    --   "multiplier": 1.25,
    --   "conditions": {
    --     "date_range": {"from": "2026-06-01", "to": "2026-09-30"},
    --     "days_of_week": [5, 6],
    --     "min_days_advance": 30
    --   }
    -- }
    priority        INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pricing_org ON pricing_rules(org_id);

-- Sustainability: flexible metrics via JSONB
CREATE TABLE sustainability_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    event_id        UUID NOT NULL REFERENCES events(id),
    -- JSONB: ISO 20121-aligned metrics with room for future categories
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "carbon_kg": 245.5,
    --   "waste_kg": 120.0,
    --   "recycled_kg": 85.0,
    --   "water_litres": 3500,
    --   "energy_kwh": 890,
    --   "food_waste_kg": 15.2,
    --   "food_donated_kg": 25.0,
    --   "single_use_plastic_items": 0,
    --   "local_supplier_pct": 0.72,
    --   "carbon_offset_purchased": true,
    --   "measurement_method": "estimated"
    -- }
    notes           TEXT,
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sustainability_event ON sustainability_reports(event_id);
CREATE INDEX idx_sustainability_metrics ON sustainability_reports USING gin(metrics);
```

---

## Audit Log & Integrations

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_entity ON audit_log(org_id, entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    -- JSONB: provider-specific config and encrypted token references
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "provider": "google_calendar",
    --   "access_token_ref": "vault:google_cal_token_org123",
    --   "refresh_token_ref": "vault:google_cal_refresh_org123",
    --   "scopes": ["https://www.googleapis.com/auth/calendar"],
    --   "last_synced_at": "2026-05-22T10:00:00Z",
    --   "sync_spaces": [{"space_id": "uuid", "calendar_id": "room@resource.calendar.google.com"}]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_org ON integrations(org_id);
```

---

## JSONB Query Examples

```sql
-- Find all venues with outdoor ceremony options
SELECT id, name
FROM venues
WHERE attributes @> '{"ceremony_options": ["outdoor_garden"]}';

-- Find all spaces that have both projector and WiFi amenities
SELECT id, name
FROM spaces
WHERE amenities @> '["projector", "WiFi"]';

-- Find wedding bookings with more than 6 bridesmaids
SELECT id, name, details->>'wedding_party_size' AS party_size
FROM bookings
WHERE event_type = 'wedding'
  AND (details->>'wedding_party_size')::int > 6;

-- Revenue by catering category from invoice line items
SELECT
    item->>'category' AS category,
    SUM((item->>'total')::numeric) AS total_revenue
FROM invoices,
     jsonb_array_elements(line_items) AS item
WHERE org_id = 'org-uuid'
  AND status IN ('paid', 'partially_paid')
GROUP BY item->>'category'
ORDER BY total_revenue DESC;

-- Find events with specific AV requirements from BEO content
SELECT b.beo_number, b.event_date, av_item->>'item' AS equipment
FROM beos b,
     jsonb_array_elements(content->'av_requirements') AS av_item
WHERE av_item->>'item' ILIKE '%wireless%';

-- Sustainability: average carbon per guest across all events
SELECT
    AVG((metrics->>'carbon_kg')::numeric / NULLIF(e.guest_count, 0)) AS avg_carbon_per_guest
FROM sustainability_reports sr
JOIN events e ON e.id = sr.event_id
WHERE sr.org_id = 'org-uuid';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisations, users, org_memberships |
| Venues & Spaces | 2 | venues, spaces (setup configs, amenities in JSONB) |
| CRM | 3 | accounts, contacts, leads |
| Bookings & Events | 3 | bookings, events, space_reservations |
| Contracts & Documents | 3 | contracts, beos, documents |
| Catering | 1 | menus (full structure in JSONB) |
| Equipment | 1 | equipment (specs in JSONB) |
| Invoicing & Payments | 2 | invoices (line items in JSONB), payments |
| Tasks | 1 | tasks (checklists in JSONB) |
| Pricing | 1 | pricing_rules (rule config in JSONB) |
| Sustainability | 1 | sustainability_reports (metrics in JSONB) |
| Audit & Integrations | 2 | audit_log, integrations |
| **Total** | **23** | ~38% fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for venue-type-specific attributes** eliminates the need for separate wedding_venues, convention_centres, and restaurant_venues tables. A single `venues.attributes` JSONB column holds type-specific fields, queried efficiently via GIN indexes and `@>` containment operators.

2. **BEO content as structured JSONB** rather than 3 separate tables (beos, beo_meal_services, beo_menu_items). The BEO is fundamentally a document — it gets generated, reviewed, approved, and distributed as a unit. JSONB preserves the document nature while remaining queryable. AI-generated BEOs can write the complete structure in one operation.

3. **Invoice line items in JSONB** rather than a separate table. Line items are always read and written with their parent invoice. JSONB makes API serialisation trivial (the database column IS the API response shape) and eliminates N+1 query patterns.

4. **Space reservations remain relational** despite the JSONB-heavy approach. The PostgreSQL exclusion constraint (`EXCLUDE USING gist`) that prevents double-booking requires relational columns — JSONB cannot participate in exclusion constraints. This is a deliberate hybrid: use JSONB where flexibility matters, use relational columns where database-level constraints matter.

5. **Menu structure in JSONB** collapses 4 tables (categories, items, packages, package_items) into 1 table. Menus are edited as complete documents and rendered as complete documents. The JSONB structure mirrors the React component tree that will render the menu editor UI.

6. **Organisation-level custom fields configuration** in `organisations.settings.custom_fields_config` allows each venue operator to define custom fields for bookings, events, and contacts without schema migrations. Custom field values are stored in `custom_fields` JSONB columns on the relevant tables.

7. **Integration configuration in JSONB** allows each OAuth provider to store its specific config (scopes, calendar mappings, sync preferences) without a fixed schema. Token secrets are stored by reference to an external vault, not inline.

8. **Sustainability metrics in JSONB** accommodate the evolving ISO 20121:2024 categories. As the standard adds new measurement types, the JSONB column accepts them without migration. GIN indexes enable aggregation queries across any metric type.

9. **23 tables vs 37** in the normalised model — a 38% reduction. Every eliminated table is one fewer migration to manage, one fewer ORM model to maintain, and one fewer API endpoint to secure. The trade-off is that some queries require JSONB extraction operators.

10. **Discipline boundary: what stays relational.** Foreign keys, statuses, dates, amounts, and any field used in exclusion constraints or JOIN conditions remain relational columns. JSONB is reserved for nested structures, variable attributes, and fields that vary by type or organisation. This prevents the "everything in JSONB" anti-pattern.
