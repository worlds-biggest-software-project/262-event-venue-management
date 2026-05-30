# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Event Venue Management · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational database design. Every real-world concept — venue, space, booking, event, contact, contract, invoice, menu item, equipment asset — gets its own table with explicit foreign key relationships. Junction tables handle many-to-many relationships (e.g., events-to-rooms, events-to-equipment, menus-to-dietary-tags). Reference data (event types, room setup styles, dietary categories) lives in dedicated lookup tables aligned with APEX/ANSI event industry terminology.

This is the architecture most incumbent venue management platforms (Tripleseat, Momentus/VenueOps, EventPro) use internally. It maps directly to the resource-oriented REST APIs these platforms expose (events, bookings, leads, contacts, locations, rooms) and makes regulatory compliance straightforward — every field has a known type, constraint, and audit surface.

The normalized approach excels when data integrity is paramount, cross-entity reporting is frequent (e.g., "revenue by room by month by event type"), and the schema is expected to remain relatively stable once established. It is the most natural fit for SQL-based reporting tools, BI dashboards, and financial audit requirements.

**Best for:** Teams that value data integrity, need complex cross-entity reporting, and operate in a regulatory environment requiring clear audit surfaces.

**Trade-offs:**
- (+) Strong referential integrity — the database enforces business rules
- (+) Excellent for complex JOIN-based analytics and reporting
- (+) Direct mapping to REST API resources (one table = one endpoint)
- (+) Easy to understand and onboard new developers
- (+) Well-supported by every ORM and migration tool
- (-) Schema migrations required for every new field or concept
- (-) Many tables (60+) can feel heavyweight for an MVP
- (-) Jurisdiction-specific or venue-type-specific fields require nullable columns or EAV patterns
- (-) Historical state queries ("what was the booking on March 1?") require separate versioning mechanisms

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| APEX/ANSI Event Industry Standards | Event type taxonomy, BEO field names, room setup style vocabulary align with APEX glossary |
| iCalendar (RFC 5545) | `events.ical_uid` stores the RFC 5545 UID for calendar sync; `DTSTART`/`DTEND` map to `start_at`/`end_at` |
| CalDAV (RFC 4791) | Calendar sync table tracks per-room CalDAV collection URLs for bi-directional sync |
| ISO 3166-1/2 | `venues.country_code` and `venues.region_code` use ISO 3166 codes |
| ISO 4217 | `invoices.currency` and all monetary columns reference ISO 4217 currency codes |
| ISO 20121:2024 | Sustainability metrics tables track carbon, waste, and social impact per ISO 20121 categories |
| PCI DSS | No raw card data stored; `payments.stripe_payment_intent_id` delegates to Stripe tokenisation |
| GDPR / CCPA | `contacts.consent_status`, `contacts.data_retention_until`, and deletion workflow columns support data subject rights |
| OAuth 2.0 (RFC 6749) | `oauth_tokens` table stores encrypted access/refresh tokens for calendar and payment integrations |
| OpenAPI 3.1 | Table structure maps 1:1 to OpenAPI resource schemas for auto-generated API documentation |

---

## Core Identity & Multi-Tenancy

```sql
-- Every tenant-scoped table includes org_id for row-level security
-- PostgreSQL RLS policies enforce tenant isolation

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    billing_email   TEXT,
    stripe_customer_id TEXT,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    country_code    CHAR(2) NOT NULL,          -- ISO 3166-1 alpha-2
    currency        CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    settings        JSONB NOT NULL DEFAULT '{}',
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
    consent_status  TEXT NOT NULL DEFAULT 'pending',  -- GDPR
    data_retention_until DATE,                        -- GDPR
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    role            TEXT NOT NULL DEFAULT 'member',  -- owner, admin, manager, member, viewer
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);

CREATE INDEX idx_org_memberships_org ON org_memberships(org_id);
CREATE INDEX idx_org_memberships_user ON org_memberships(user_id);
```

---

## Venue & Space Management

```sql
CREATE TABLE venues (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    region_code     TEXT,              -- ISO 3166-2
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL,  -- ISO 3166-1
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    phone           TEXT,
    email           TEXT,
    website_url     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_venues_org ON venues(org_id);

CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    venue_id        UUID NOT NULL REFERENCES venues(id),
    parent_space_id UUID REFERENCES spaces(id),  -- for sub-rooms / divisible spaces
    name            TEXT NOT NULL,
    description     TEXT,
    capacity_min    INT,
    capacity_max    INT NOT NULL,
    area_sqft       NUMERIC(10,2),
    floor_level     INT,
    is_outdoor      BOOLEAN NOT NULL DEFAULT false,
    is_divisible    BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    hourly_rate     NUMERIC(12,2),
    half_day_rate   NUMERIC(12,2),
    full_day_rate   NUMERIC(12,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    calendar_sync_url TEXT,            -- CalDAV collection URL
    ical_feed_url   TEXT,              -- iCal read-only feed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spaces_venue ON spaces(venue_id);
CREATE INDEX idx_spaces_parent ON spaces(parent_space_id);

-- Room setup configurations (theatre, banquet, classroom, U-shape, etc.)
-- Vocabulary aligned with APEX room setup terminology
CREATE TABLE setup_styles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,       -- e.g., 'Theatre', 'Banquet', 'Classroom', 'U-Shape', 'Boardroom'
    description     TEXT,
    icon_url        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE space_setup_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    setup_style_id  UUID NOT NULL REFERENCES setup_styles(id),
    capacity        INT NOT NULL,
    floor_plan_url  TEXT,              -- link to floor plan image/SVG for this configuration
    setup_time_mins INT DEFAULT 30,
    teardown_time_mins INT DEFAULT 30,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (space_id, setup_style_id)
);

CREATE INDEX idx_space_setups_space ON space_setup_configs(space_id);

-- Space amenities (WiFi, projector, whiteboard, etc.)
CREATE TABLE amenities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    category        TEXT,              -- 'AV', 'furniture', 'technology', 'accessibility'
    icon_url        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE space_amenities (
    space_id        UUID NOT NULL REFERENCES spaces(id),
    amenity_id      UUID NOT NULL REFERENCES amenities(id),
    quantity        INT NOT NULL DEFAULT 1,
    notes           TEXT,
    PRIMARY KEY (space_id, amenity_id)
);
```

---

## CRM — Accounts & Contacts

```sql
-- Accounts represent organisations/companies that book events
CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL DEFAULT 'corporate',  -- corporate, wedding, social, non-profit, government
    industry        TEXT,
    website_url     TEXT,
    phone           TEXT,
    email           TEXT,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    region_code     TEXT,
    postal_code     TEXT,
    country_code    CHAR(2),
    tax_id          TEXT,
    notes           TEXT,
    source          TEXT,              -- referral, website, cold_call, repeat
    lifetime_value  NUMERIC(14,2) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_accounts_org ON accounts(org_id);
CREATE INDEX idx_accounts_type ON accounts(org_id, account_type);

-- Contacts are individuals associated with accounts
CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    account_id      UUID REFERENCES accounts(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    mobile          TEXT,
    job_title       TEXT,
    role            TEXT,              -- primary, billing, on-site coordinator
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    consent_status  TEXT NOT NULL DEFAULT 'pending',  -- GDPR: granted, denied, pending, withdrawn
    consent_date    TIMESTAMPTZ,
    data_retention_until DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_org ON contacts(org_id);
CREATE INDEX idx_contacts_account ON contacts(account_id);
CREATE INDEX idx_contacts_email ON contacts(email);

-- Leads represent inbound enquiries before conversion to bookings
CREATE TABLE leads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    venue_id        UUID REFERENCES venues(id),
    contact_id      UUID REFERENCES contacts(id),
    account_id      UUID REFERENCES accounts(id),
    source          TEXT NOT NULL,      -- website_form, phone, email, referral, walk_in
    status          TEXT NOT NULL DEFAULT 'new',  -- new, contacted, qualified, proposal_sent, won, lost
    event_type      TEXT,
    preferred_date  DATE,
    preferred_time  TEXT,
    alt_date        DATE,
    guest_count_est INT,
    budget_est      NUMERIC(12,2),
    notes           TEXT,
    ai_score        NUMERIC(5,2),      -- AI lead qualification score (0-100)
    ai_score_reasons JSONB,
    assigned_to     UUID REFERENCES users(id),
    converted_booking_id UUID,          -- populated when lead converts
    lost_reason     TEXT,
    responded_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leads_org_status ON leads(org_id, status);
CREATE INDEX idx_leads_venue ON leads(venue_id);
CREATE INDEX idx_leads_assigned ON leads(assigned_to);
```

---

## Bookings & Events

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- A booking is a contract-level grouping; an event is a time block within a booking
-- Mirrors the Tripleseat model: Booking > Event(s)
CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    venue_id        UUID NOT NULL REFERENCES venues(id),
    account_id      UUID NOT NULL REFERENCES accounts(id),
    contact_id      UUID NOT NULL REFERENCES contacts(id),
    lead_id         UUID REFERENCES leads(id),
    booking_number  TEXT NOT NULL,      -- human-readable sequential number
    name            TEXT NOT NULL,      -- e.g., "Smith-Jones Wedding Reception"
    event_type      TEXT NOT NULL,      -- wedding, corporate, social, conference, gala
    status          TEXT NOT NULL DEFAULT 'tentative',
        -- tentative, confirmed, contracted, in_progress, completed, cancelled
    guest_count     INT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    setup_date      DATE,
    teardown_date   DATE,
    notes           TEXT,
    internal_notes  TEXT,
    assigned_to     UUID REFERENCES users(id),
    cancelled_at    TIMESTAMPTZ,
    cancelled_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, booking_number)
);

CREATE INDEX idx_bookings_org_status ON bookings(org_id, status);
CREATE INDEX idx_bookings_venue ON bookings(venue_id);
CREATE INDEX idx_bookings_account ON bookings(account_id);
CREATE INDEX idx_bookings_dates ON bookings(start_date, end_date);

-- Events are time blocks within a booking (e.g., ceremony 2pm-3pm, reception 5pm-11pm)
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    setup_start_at  TIMESTAMPTZ,
    teardown_end_at TIMESTAMPTZ,
    guest_count     INT,
    status          TEXT NOT NULL DEFAULT 'scheduled',
        -- scheduled, in_progress, completed, cancelled
    ical_uid        TEXT,              -- RFC 5545 UID for calendar interop
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_events_booking ON events(booking_id);
CREATE INDEX idx_events_dates ON events(start_at, end_at);

-- Junction: which spaces are used by which events
CREATE TABLE event_spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    space_id        UUID NOT NULL REFERENCES spaces(id),
    setup_style_id  UUID REFERENCES setup_styles(id),
    capacity_override INT,
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (event_id, space_id),

    -- Prevent double-booking: no two non-cancelled event_spaces for the same space can overlap
    EXCLUDE USING gist (
        space_id WITH =,
        tstzrange(start_at, end_at) WITH &&
    )
);

CREATE INDEX idx_event_spaces_event ON event_spaces(event_id);
CREATE INDEX idx_event_spaces_space ON event_spaces(space_id);
```

---

## Contracts & Documents

```sql
CREATE TABLE contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID NOT NULL REFERENCES bookings(id),
    contract_number TEXT NOT NULL,
    title           TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
        -- draft, sent, viewed, signed, countersigned, expired, voided
    template_id     UUID,
    content_html    TEXT,              -- rendered contract body
    total_amount    NUMERIC(14,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    sent_at         TIMESTAMPTZ,
    viewed_at       TIMESTAMPTZ,
    signed_at       TIMESTAMPTZ,
    signer_name     TEXT,
    signer_email    TEXT,
    signer_ip       INET,
    signature_data  TEXT,              -- base64 signature image or e-sign reference
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contracts_booking ON contracts(booking_id);
CREATE INDEX idx_contracts_org_status ON contracts(org_id, status);

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID REFERENCES bookings(id),
    event_id        UUID REFERENCES events(id),
    doc_type        TEXT NOT NULL,      -- contract, beo, proposal, floor_plan, invoice, photo, other
    title           TEXT NOT NULL,
    file_url        TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type       TEXT,
    version         INT NOT NULL DEFAULT 1,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_org ON documents(org_id);
CREATE INDEX idx_documents_booking ON documents(booking_id);
```

---

## Banquet Event Orders (BEOs)

```sql
-- BEO is the core operational document — structured per APEX ESG field guidance
CREATE TABLE beos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID NOT NULL REFERENCES bookings(id),
    event_id        UUID NOT NULL REFERENCES events(id),
    beo_number      TEXT NOT NULL,
    version         INT NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'draft',  -- draft, reviewed, approved, distributed, archived
    event_name      TEXT NOT NULL,
    event_date      DATE NOT NULL,
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    guest_count     INT NOT NULL,
    guaranteed_count INT,
    room_name       TEXT NOT NULL,
    setup_style     TEXT,
    setup_notes     TEXT,
    general_notes   TEXT,
    av_notes        TEXT,
    parking_notes   TEXT,
    special_requests TEXT,
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    distributed_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_beos_booking ON beos(booking_id);
CREATE INDEX idx_beos_event ON beos(event_id);

-- BEO meal/food service blocks
CREATE TABLE beo_meal_services (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    beo_id          UUID NOT NULL REFERENCES beos(id) ON DELETE CASCADE,
    service_type    TEXT NOT NULL,      -- breakfast, lunch, dinner, reception, break, snack
    service_time    TIME,
    location        TEXT,
    notes           TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE beo_menu_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meal_service_id UUID NOT NULL REFERENCES beo_meal_services(id) ON DELETE CASCADE,
    menu_item_id    UUID REFERENCES menu_items(id),
    name            TEXT NOT NULL,
    description     TEXT,
    quantity         INT,
    price_per_unit  NUMERIC(10,2),
    dietary_tags    TEXT[],            -- {'vegan','gluten-free','nut-free'}
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Catering & Menus

```sql
CREATE TABLE menu_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,       -- appetiser, entree, dessert, beverage, bar_package
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE menu_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    category_id     UUID NOT NULL REFERENCES menu_categories(id),
    name            TEXT NOT NULL,
    description     TEXT,
    price           NUMERIC(10,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    unit            TEXT DEFAULT 'per_person',  -- per_person, per_item, per_dozen, flat_rate
    is_active       BOOLEAN NOT NULL DEFAULT true,
    allergens       TEXT[],            -- {'dairy','nuts','gluten','shellfish'}
    dietary_tags    TEXT[],            -- {'vegan','vegetarian','gluten-free','halal','kosher'}
    prep_time_mins  INT,
    image_url       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_menu_items_org ON menu_items(org_id);
CREATE INDEX idx_menu_items_category ON menu_items(category_id);

-- Menu packages (bundled pricing)
CREATE TABLE menu_packages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    description     TEXT,
    price_per_person NUMERIC(10,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    min_guests      INT,
    max_guests      INT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE menu_package_items (
    package_id      UUID NOT NULL REFERENCES menu_packages(id) ON DELETE CASCADE,
    menu_item_id    UUID NOT NULL REFERENCES menu_items(id),
    is_default      BOOLEAN NOT NULL DEFAULT true,  -- included vs optional upgrade
    upgrade_price   NUMERIC(10,2),
    PRIMARY KEY (package_id, menu_item_id)
);
```

---

## AV & Equipment Management

```sql
CREATE TABLE equipment_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,       -- audio, video, lighting, staging, furniture, signage
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    category_id     UUID NOT NULL REFERENCES equipment_categories(id),
    venue_id        UUID REFERENCES venues(id),
    name            TEXT NOT NULL,
    description     TEXT,
    serial_number   TEXT,
    quantity_total  INT NOT NULL DEFAULT 1,
    rental_rate     NUMERIC(10,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    rental_unit     TEXT DEFAULT 'per_event',  -- per_hour, per_event, per_day
    condition       TEXT DEFAULT 'good',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_org ON equipment(org_id);
CREATE INDEX idx_equipment_venue ON equipment(venue_id);

-- Equipment reserved for specific events
CREATE TABLE event_equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    quantity        INT NOT NULL DEFAULT 1,
    setup_notes     TEXT,
    rate_override   NUMERIC(10,2),
    status          TEXT NOT NULL DEFAULT 'reserved',  -- reserved, deployed, returned
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_event_equipment_event ON event_equipment(event_id);
CREATE INDEX idx_event_equipment_equipment ON event_equipment(equipment_id);
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
        -- draft, sent, viewed, partially_paid, paid, overdue, void, refunded
    subtotal        NUMERIC(14,2) NOT NULL DEFAULT 0,
    tax_rate        NUMERIC(5,4),
    tax_amount      NUMERIC(14,2) NOT NULL DEFAULT 0,
    service_charge_rate NUMERIC(5,4),
    service_charge  NUMERIC(14,2) NOT NULL DEFAULT 0,
    discount_amount NUMERIC(14,2) NOT NULL DEFAULT 0,
    total           NUMERIC(14,2) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    due_date        DATE,
    sent_at         TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, invoice_number)
);

CREATE INDEX idx_invoices_booking ON invoices(booking_id);
CREATE INDEX idx_invoices_org_status ON invoices(org_id, status);

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    description     TEXT NOT NULL,
    category        TEXT,              -- space_rental, catering, av_equipment, service_fee, other
    quantity        NUMERIC(10,2) NOT NULL DEFAULT 1,
    unit_price      NUMERIC(12,2) NOT NULL,
    total           NUMERIC(14,2) NOT NULL,
    taxable         BOOLEAN NOT NULL DEFAULT true,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_line_items_invoice ON invoice_line_items(invoice_id);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    amount          NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    payment_method  TEXT NOT NULL,      -- credit_card, bank_transfer, cheque, cash, other
    status          TEXT NOT NULL DEFAULT 'pending',
        -- pending, processing, succeeded, failed, refunded, partially_refunded
    stripe_payment_intent_id TEXT,      -- PCI DSS: delegate card handling to Stripe
    stripe_charge_id TEXT,
    reference       TEXT,
    notes           TEXT,
    paid_at         TIMESTAMPTZ,
    refunded_at     TIMESTAMPTZ,
    refund_amount   NUMERIC(14,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_org ON payments(org_id);
CREATE INDEX idx_payments_stripe ON payments(stripe_payment_intent_id);
```

---

## Tasks & Timeline

```sql
CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID REFERENCES bookings(id),
    event_id        UUID REFERENCES events(id),
    title           TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'pending',  -- pending, in_progress, completed, cancelled
    priority        TEXT NOT NULL DEFAULT 'medium',   -- low, medium, high, urgent
    assigned_to     UUID REFERENCES users(id),
    due_at          TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_booking ON tasks(booking_id);
CREATE INDEX idx_tasks_assigned ON tasks(assigned_to);
CREATE INDEX idx_tasks_org_status ON tasks(org_id, status);
```

---

## Sustainability Tracking (ISO 20121)

```sql
CREATE TABLE sustainability_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    event_id        UUID NOT NULL REFERENCES events(id),
    metric_type     TEXT NOT NULL,
        -- carbon_kg, waste_kg, recycled_kg, water_litres, energy_kwh, food_waste_kg, attendee_transport_km
    value           NUMERIC(14,4) NOT NULL,
    unit            TEXT NOT NULL,
    measurement_method TEXT,           -- estimated, measured, calculated
    notes           TEXT,
    recorded_by     UUID REFERENCES users(id),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sustainability_event ON sustainability_metrics(event_id);
CREATE INDEX idx_sustainability_org_type ON sustainability_metrics(org_id, metric_type);
```

---

## Dynamic Pricing

```sql
CREATE TABLE pricing_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    venue_id        UUID REFERENCES venues(id),
    space_id        UUID REFERENCES spaces(id),
    name            TEXT NOT NULL,
    rule_type       TEXT NOT NULL,      -- seasonal, day_of_week, advance_booking, demand, time_of_day
    multiplier      NUMERIC(5,3),       -- e.g., 1.25 for 25% premium
    flat_adjustment NUMERIC(12,2),
    valid_from      DATE,
    valid_to        DATE,
    days_of_week    INT[],             -- {0,1,2,3,4,5,6} where 0=Sunday
    min_days_advance INT,
    max_days_advance INT,
    priority        INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pricing_rules_org ON pricing_rules(org_id);
CREATE INDEX idx_pricing_rules_space ON pricing_rules(space_id);
```

---

## Integration & OAuth Tokens

```sql
CREATE TABLE oauth_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    provider        TEXT NOT NULL,      -- google_calendar, microsoft_graph, stripe
    access_token_enc TEXT NOT NULL,     -- encrypted at rest
    refresh_token_enc TEXT,
    token_type      TEXT NOT NULL DEFAULT 'Bearer',
    expires_at      TIMESTAMPTZ,
    scopes          TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE calendar_syncs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    provider        TEXT NOT NULL,      -- google_calendar, microsoft_graph, apple_caldav
    external_calendar_id TEXT NOT NULL,
    sync_direction  TEXT NOT NULL DEFAULT 'bidirectional',  -- inbound, outbound, bidirectional
    last_synced_at  TIMESTAMPTZ,
    sync_status     TEXT NOT NULL DEFAULT 'active',
    sync_token      TEXT,              -- provider's sync/delta token
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_calendar_syncs_space ON calendar_syncs(space_id);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,      -- create, update, delete, sign, approve, send
    entity_type     TEXT NOT NULL,      -- booking, event, contract, invoice, payment, beo
    entity_id       UUID NOT NULL,
    changes         JSONB,             -- {"field": {"old": "...", "new": "..."}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_entity ON audit_log(org_id, entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
-- Partition by month for retention management
-- CREATE TABLE audit_log PARTITION BY RANGE (created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisations, users, org_memberships |
| Venue & Space Management | 5 | venues, spaces, setup_styles, space_setup_configs, amenities + 1 junction |
| CRM | 3 | accounts, contacts, leads |
| Bookings & Events | 3 | bookings, events, event_spaces (with exclusion constraint) |
| Contracts & Documents | 2 | contracts, documents |
| BEOs | 3 | beos, beo_meal_services, beo_menu_items |
| Catering & Menus | 4 | menu_categories, menu_items, menu_packages, menu_package_items |
| AV & Equipment | 3 | equipment_categories, equipment, event_equipment |
| Invoicing & Payments | 3 | invoices, invoice_line_items, payments |
| Tasks | 1 | tasks |
| Sustainability | 1 | sustainability_metrics |
| Dynamic Pricing | 1 | pricing_rules |
| Integration | 2 | oauth_tokens, calendar_syncs |
| Audit | 1 | audit_log |
| **Total** | **36** | Plus 1 junction table (space_amenities) = **37 tables** |

---

## Key Design Decisions

1. **Booking > Event hierarchy** mirrors the Tripleseat/Momentus data model where a booking (contract-level) contains one or more events (time blocks). This matches industry practice where a wedding booking might include a ceremony event, cocktail hour, and reception.

2. **PostgreSQL exclusion constraints** on `event_spaces` prevent double-booking at the database level using `tstzrange` with the `&&` overlap operator and `btree_gist`. This is the strongest possible guarantee against scheduling conflicts.

3. **No raw payment card data** is stored anywhere. The `payments` table only stores Stripe references (`stripe_payment_intent_id`, `stripe_charge_id`), fully delegating PCI DSS scope to Stripe's certified infrastructure.

4. **GDPR compliance columns** (`consent_status`, `consent_date`, `data_retention_until`) on `contacts` and `users` enable data subject rights workflows including consent tracking, data portability, and right-to-erasure.

5. **ISO-aligned reference data** — country codes (ISO 3166-1), region codes (ISO 3166-2), currency codes (ISO 4217), and iCalendar UIDs (RFC 5545) ensure interoperability with external systems.

6. **BEO as a first-class entity** with structured meal services and menu items rather than a document blob. This enables AI-powered BEO generation from natural language and automated production sheet creation for kitchen staff.

7. **Multi-tenant via `org_id`** on every business table, designed for PostgreSQL Row-Level Security policies. A single shared-schema database keeps operational costs low while enforcing tenant isolation at the database level.

8. **Audit log as a separate table** rather than event sourcing — simpler to implement but limited to recording what changed, not reconstructing full historical state. Suitable for compliance but not for temporal queries.

9. **Dynamic pricing rules** are stored declaratively (multipliers, date ranges, day-of-week patterns) so the pricing engine can stack and evaluate rules programmatically, enabling the AI-powered dynamic pricing feature.

10. **Calendar sync infrastructure** supports Google Calendar, Microsoft Graph, and CalDAV providers with per-space sync configuration, enabling the bi-directional availability sync described in the features spec.
