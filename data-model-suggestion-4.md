# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Event Venue Management · Created: 2026-05-22

## Philosophy

This model combines a conventional relational schema for day-to-day CRUD operations with a property graph layer for relationship-heavy queries. The graph captures the web of connections between people, organisations, venues, events, vendors, and equipment that is central to venue management but awkward to express in pure relational JOINs. The relational tables handle transactional operations (booking creation, invoice generation, payment processing), while the graph answers questions like "show me all events where Company X and Vendor Y have both been involved", "which contacts are connected to multiple accounts?", and "what is the full dependency chain for this event's setup?"

The graph layer is implemented using PostgreSQL's own tables (`graph_nodes` and `graph_edges`) rather than requiring a separate graph database like Neo4j. This keeps the architecture simple — one database, one backup strategy, one deployment — while still enabling recursive CTE graph traversals, shortest-path queries, and relationship pattern matching. For organisations that outgrow PostgreSQL's graph capabilities, the node/edge tables can be exported to Neo4j or Apache AGE (PostgreSQL's graph extension) without changing the relational side.

This architecture is particularly compelling for venue management because the domain is fundamentally about relationships: a contact works for an account, which books events at venues, which use spaces configured with equipment, staffed by team members, catered by vendors, photographed by partners. AI-powered features like "suggest vendors based on past successful events with similar profiles" or "identify conflicts of interest across bookings" require efficient graph traversal.

**Best for:** Multi-venue operations with complex vendor networks, corporate campus environments with recurring client relationships, and platforms that want AI-powered relationship insights and recommendations.

**Trade-offs:**
- (+) Relationship queries are natural and efficient (recursive CTEs, graph traversals)
- (+) AI-powered recommendations ("clients who booked X also booked Y") are trivial
- (+) Vendor/supplier network analysis built into the data model
- (+) Conflict detection (same vendor at competing events) is a simple graph query
- (+) Single PostgreSQL database — no separate graph database infrastructure
- (+) Graph layer is additive — relational tables work independently
- (-) Dual-write pattern: business operations update both relational and graph tables
- (-) Graph queries (recursive CTEs) can be slower than purpose-built graph databases for deep traversals
- (-) More tables than the JSONB hybrid approach
- (-) Developers need to understand both relational and graph query patterns
- (-) Graph consistency must be maintained by application logic or triggers

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| APEX/ANSI Event Industry Standards | Event type taxonomy, BEO terminology, and vendor role classifications aligned with APEX glossary |
| iCalendar (RFC 5545) | `ical_uid` on events for calendar interoperability |
| ISO 3166-1/2 | Venue and account location codes |
| ISO 4217 | Currency codes on all financial entities |
| ISO 20121:2024 | Sustainability metrics linked to events via graph edges for supply-chain-level carbon tracking |
| PCI DSS | Payment processing delegated to Stripe — only intent IDs stored |
| GDPR / CCPA | Contact consent and retention fields; graph edges enable "show me all data linked to this contact" for data subject access requests |
| OAuth 2.0 (RFC 6749) | Integration token management |
| W3C RDF / Property Graph Model | Graph layer follows the labelled property graph model (nodes with types and properties, edges with types, directions, and properties) |

---

## Graph Layer

```sql
-- ================================================================
-- PROPERTY GRAPH: Nodes and Edges
-- Implements a labelled property graph within PostgreSQL
-- ================================================================

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    node_type       TEXT NOT NULL,
        -- 'Contact', 'Account', 'Venue', 'Space', 'Booking', 'Event',
        -- 'Vendor', 'Equipment', 'User', 'Menu', 'Invoice', 'Contract'
    entity_id       UUID NOT NULL,     -- FK to the relational table's primary key
    label           TEXT NOT NULL,      -- human-readable label for display
    properties      JSONB NOT NULL DEFAULT '{}',
        -- Denormalised key properties for graph queries without JOINing back
        -- Contact: {"email": "...", "company": "..."}
        -- Venue: {"city": "...", "capacity": 500}
        -- Booking: {"event_type": "wedding", "guest_count": 200, "status": "confirmed"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (node_type, entity_id)
);

CREATE INDEX idx_gn_org ON graph_nodes(org_id);
CREATE INDEX idx_gn_type ON graph_nodes(node_type);
CREATE INDEX idx_gn_entity ON graph_nodes(entity_id);
CREATE INDEX idx_gn_props ON graph_nodes USING gin(properties);
CREATE INDEX idx_gn_label ON graph_nodes USING gin(label gin_trgm_ops);  -- requires pg_trgm

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,
        -- Relationship types:
        -- 'WORKS_FOR'          Contact -> Account
        -- 'BOOKED_BY'          Booking -> Account
        -- 'PRIMARY_CONTACT'    Booking -> Contact
        -- 'HELD_AT'            Booking -> Venue
        -- 'USES_SPACE'         Event -> Space
        -- 'PART_OF'            Event -> Booking
        -- 'CATERED_BY'         Event -> Vendor
        -- 'PHOTOGRAPHED_BY'    Event -> Vendor
        -- 'AV_PROVIDED_BY'     Event -> Vendor
        -- 'DECORATED_BY'       Event -> Vendor
        -- 'USES_EQUIPMENT'     Event -> Equipment
        -- 'INVOICED_TO'        Invoice -> Account
        -- 'INVOICE_FOR'        Invoice -> Booking
        -- 'REFERRED_BY'        Account -> Contact  (referral tracking)
        -- 'MANAGED_BY'         Venue -> User
        -- 'ASSIGNED_TO'        Booking -> User
        -- 'PREVIOUS_BOOKING'   Booking -> Booking  (repeat client chain)
        -- 'CHILD_SPACE'        Space -> Space  (divisible rooms)
        -- 'SUSTAINABILITY_VENDOR'  Event -> Vendor  (sustainable supply chain)
    properties      JSONB NOT NULL DEFAULT '{}',
        -- Edge-specific data:
        -- CATERED_BY: {"contract_value": 15000, "menu_package": "Gold", "rating": 4.8}
        -- USES_SPACE: {"setup_style": "Banquet", "capacity": 200, "hours": 6}
        -- REFERRED_BY: {"referral_date": "2026-03-15", "referral_bonus_paid": true}
    weight          NUMERIC(10,4),     -- optional weight for path/ranking algorithms
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,       -- NULL = currently active; temporal relationships
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_source ON graph_edges(source_node_id);
CREATE INDEX idx_ge_target ON graph_edges(target_node_id);
CREATE INDEX idx_ge_type ON graph_edges(edge_type);
CREATE INDEX idx_ge_org ON graph_edges(org_id);
CREATE INDEX idx_ge_props ON graph_edges USING gin(properties);
CREATE INDEX idx_ge_temporal ON graph_edges(valid_from, valid_to) WHERE is_active = true;

-- Composite index for common graph traversal patterns
CREATE INDEX idx_ge_source_type ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edges(target_node_id, edge_type);
```

---

## Relational Core — Identity & Multi-Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    billing_email   TEXT,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    country_code    CHAR(2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    settings        JSONB NOT NULL DEFAULT '{}',
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    role            TEXT NOT NULL DEFAULT 'member',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);
```

---

## Relational Core — Venues, Spaces, CRM

```sql
CREATE TABLE venues (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    venue_type      TEXT NOT NULL,
    address_line1   TEXT,
    city            TEXT,
    region_code     TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_venues_org ON venues(org_id);

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
    hourly_rate     NUMERIC(12,2),
    half_day_rate   NUMERIC(12,2),
    full_day_rate   NUMERIC(12,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spaces_venue ON spaces(venue_id);

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL DEFAULT 'corporate',
    email           TEXT,
    phone           TEXT,
    city            TEXT,
    country_code    CHAR(2),
    source          TEXT,
    lifetime_value  NUMERIC(14,2) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_accounts_org ON accounts(org_id);

CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    account_id      UUID REFERENCES accounts(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    job_title       TEXT,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    consent_status  TEXT NOT NULL DEFAULT 'pending',
    consent_date    TIMESTAMPTZ,
    data_retention_until DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_org ON contacts(org_id);
CREATE INDEX idx_contacts_account ON contacts(account_id);
CREATE INDEX idx_contacts_email ON contacts(email);

-- Vendors are first-class entities in the graph model
CREATE TABLE vendors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    vendor_type     TEXT NOT NULL,
        -- caterer, photographer, florist, dj, band, decorator, av_provider,
        -- lighting, valet, security, officiant, bakery, rental_company
    contact_name    TEXT,
    email           TEXT,
    phone           TEXT,
    website_url     TEXT,
    city            TEXT,
    region_code     TEXT,
    country_code    CHAR(2),
    rating          NUMERIC(3,2),      -- aggregate rating (1.00-5.00)
    booking_count   INT NOT NULL DEFAULT 0,
    is_preferred    BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendors_org ON vendors(org_id);
CREATE INDEX idx_vendors_type ON vendors(org_id, vendor_type);
```

---

## Relational Core — Bookings, Events, Leads

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE leads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    venue_id        UUID REFERENCES venues(id),
    contact_id      UUID REFERENCES contacts(id),
    source          TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'new',
    event_type      TEXT,
    preferred_date  DATE,
    guest_count_est INT,
    budget_est      NUMERIC(12,2),
    ai_score        NUMERIC(5,2),
    assigned_to     UUID REFERENCES users(id),
    converted_booking_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leads_org_status ON leads(org_id, status);

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
    notes           TEXT,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, booking_number)
);

CREATE INDEX idx_bookings_org_status ON bookings(org_id, status);
CREATE INDEX idx_bookings_venue ON bookings(venue_id);
CREATE INDEX idx_bookings_dates ON bookings(start_date, end_date);

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
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_events_booking ON events(booking_id);
CREATE INDEX idx_events_dates ON events(start_at, end_at);

-- Space reservations with exclusion constraint for double-booking prevention
CREATE TABLE event_spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    space_id        UUID NOT NULL REFERENCES spaces(id),
    setup_style     TEXT,
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    UNIQUE (event_id, space_id),
    EXCLUDE USING gist (
        space_id WITH =,
        tstzrange(start_at, end_at) WITH &&
    )
);

CREATE INDEX idx_event_spaces_space ON event_spaces(space_id);

-- Vendor assignments to events (relational side)
CREATE TABLE event_vendors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    vendor_id       UUID NOT NULL REFERENCES vendors(id),
    vendor_role     TEXT NOT NULL,      -- caterer, photographer, florist, dj, etc.
    contract_value  NUMERIC(12,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          TEXT NOT NULL DEFAULT 'confirmed',  -- proposed, confirmed, cancelled
    notes           TEXT,
    rating          NUMERIC(3,2),      -- post-event rating for this specific engagement
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (event_id, vendor_id, vendor_role)
);

CREATE INDEX idx_event_vendors_event ON event_vendors(event_id);
CREATE INDEX idx_event_vendors_vendor ON event_vendors(vendor_id);
```

---

## Relational Core — Financial, Contracts, BEOs

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
    signed_at       TIMESTAMPTZ,
    signer_name     TEXT,
    signer_email    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contracts_booking ON contracts(booking_id);

CREATE TABLE beos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID NOT NULL REFERENCES bookings(id),
    event_id        UUID NOT NULL REFERENCES events(id),
    beo_number      TEXT NOT NULL,
    version         INT NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'draft',
    event_date      DATE NOT NULL,
    guest_count     INT NOT NULL,
    room_name       TEXT NOT NULL,
    setup_style     TEXT,
    content         JSONB NOT NULL DEFAULT '{}',  -- full BEO body
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_beos_booking ON beos(booking_id);

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID NOT NULL REFERENCES bookings(id),
    account_id      UUID NOT NULL REFERENCES accounts(id),
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    subtotal        NUMERIC(14,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(14,2) NOT NULL DEFAULT 0,
    total           NUMERIC(14,2) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    due_date        DATE,
    line_items      JSONB NOT NULL DEFAULT '[]',
    sent_at         TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
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
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_stripe ON payments(stripe_payment_intent_id);
```

---

## Relational Core — Equipment, Tasks, Sustainability

```sql
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    venue_id        UUID REFERENCES venues(id),
    name            TEXT NOT NULL,
    category        TEXT NOT NULL,
    quantity_total  INT NOT NULL DEFAULT 1,
    rental_rate     NUMERIC(10,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    specs           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_org ON equipment(org_id);

CREATE TABLE event_equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    quantity        INT NOT NULL DEFAULT 1,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_event_equip_event ON event_equipment(event_id);

CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    booking_id      UUID REFERENCES bookings(id),
    title           TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    priority        TEXT NOT NULL DEFAULT 'medium',
    assigned_to     UUID REFERENCES users(id),
    due_at          TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_booking ON tasks(booking_id);
CREATE INDEX idx_tasks_assigned ON tasks(assigned_to, status);

CREATE TABLE sustainability_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    event_id        UUID NOT NULL REFERENCES events(id),
    metrics         JSONB NOT NULL DEFAULT '{}',
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sustain_event ON sustainability_metrics(event_id);

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
```

---

## Graph Query Examples

```sql
-- Required extensions
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- ================================================================
-- QUERY: Find all vendors who have worked events for a specific account
-- ================================================================
SELECT DISTINCT
    v.id, v.name, v.vendor_type, v.rating,
    COUNT(DISTINCT e2.id) AS event_count,
    AVG(ev.rating) AS avg_event_rating
FROM graph_edges e1
JOIN graph_nodes n_account ON n_account.id = e1.source_node_id AND n_account.node_type = 'Account'
JOIN graph_nodes n_booking ON n_booking.id = e1.target_node_id AND n_booking.node_type = 'Booking'
JOIN graph_edges e2 ON e2.source_node_id IN (
    SELECT gn.id FROM graph_nodes gn
    JOIN graph_edges ge ON ge.source_node_id = gn.id
    WHERE ge.target_node_id = n_booking.id AND ge.edge_type = 'PART_OF'
    AND gn.node_type = 'Event'
)
JOIN graph_nodes n_vendor ON n_vendor.id = e2.target_node_id AND n_vendor.node_type = 'Vendor'
JOIN vendors v ON v.id = n_vendor.entity_id
LEFT JOIN event_vendors ev ON ev.vendor_id = v.id
WHERE n_account.entity_id = 'account-uuid-here'
  AND e1.edge_type = 'BOOKED_BY'
  AND e2.edge_type IN ('CATERED_BY', 'PHOTOGRAPHED_BY', 'AV_PROVIDED_BY', 'DECORATED_BY')
GROUP BY v.id, v.name, v.vendor_type, v.rating;

-- ================================================================
-- QUERY: Vendor co-occurrence — "Vendors frequently booked together"
-- For AI-powered vendor recommendations
-- ================================================================
WITH vendor_events AS (
    SELECT
        ge.source_node_id AS event_node_id,
        gn.entity_id AS vendor_id,
        gn.label AS vendor_name,
        ge.edge_type AS role
    FROM graph_edges ge
    JOIN graph_nodes gn ON gn.id = ge.target_node_id AND gn.node_type = 'Vendor'
    WHERE ge.edge_type IN ('CATERED_BY', 'PHOTOGRAPHED_BY', 'AV_PROVIDED_BY', 'DECORATED_BY')
      AND ge.org_id = 'org-uuid-here'
)
SELECT
    v1.vendor_name AS vendor_a,
    v1.role AS role_a,
    v2.vendor_name AS vendor_b,
    v2.role AS role_b,
    COUNT(*) AS times_together
FROM vendor_events v1
JOIN vendor_events v2
    ON v1.event_node_id = v2.event_node_id
    AND v1.vendor_id < v2.vendor_id  -- avoid duplicates
GROUP BY v1.vendor_name, v1.role, v2.vendor_name, v2.role
HAVING COUNT(*) >= 3
ORDER BY times_together DESC;

-- ================================================================
-- QUERY: Referral chain — trace how a client was referred
-- Uses recursive CTE for multi-hop graph traversal
-- ================================================================
WITH RECURSIVE referral_chain AS (
    -- Start from the target account
    SELECT
        gn.id AS node_id,
        gn.entity_id,
        gn.label,
        gn.node_type,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_id = 'account-uuid-here' AND gn.node_type = 'Account'

    UNION ALL

    -- Follow REFERRED_BY edges backwards
    SELECT
        gn2.id,
        gn2.entity_id,
        gn2.label,
        gn2.node_type,
        rc.depth + 1,
        rc.path || gn2.id
    FROM referral_chain rc
    JOIN graph_edges ge ON ge.source_node_id = rc.node_id AND ge.edge_type = 'REFERRED_BY'
    JOIN graph_nodes gn2 ON gn2.id = ge.target_node_id
    WHERE gn2.id != ALL(rc.path)  -- prevent cycles
      AND rc.depth < 10           -- max depth
)
SELECT entity_id, label, node_type, depth
FROM referral_chain
ORDER BY depth;

-- ================================================================
-- QUERY: "Show me everything connected to this contact" (GDPR DSAR)
-- ================================================================
WITH RECURSIVE contact_graph AS (
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.entity_id,
        gn.label,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_id = 'contact-uuid-here' AND gn.node_type = 'Contact'

    UNION ALL

    SELECT
        gn2.id,
        gn2.node_type,
        gn2.entity_id,
        gn2.label,
        cg.depth + 1,
        cg.path || gn2.id
    FROM contact_graph cg
    JOIN graph_edges ge ON (ge.source_node_id = cg.node_id OR ge.target_node_id = cg.node_id)
    JOIN graph_nodes gn2 ON gn2.id = CASE
        WHEN ge.source_node_id = cg.node_id THEN ge.target_node_id
        ELSE ge.source_node_id
    END
    WHERE gn2.id != ALL(cg.path)
      AND cg.depth < 3  -- limit traversal depth for DSAR
)
SELECT node_type, entity_id, label, depth
FROM contact_graph
ORDER BY depth, node_type;

-- ================================================================
-- QUERY: Conflict detection — same vendor at overlapping events
-- ================================================================
SELECT
    v.name AS vendor_name,
    e1.name AS event_1,
    e1.start_at AS event_1_start,
    e2.name AS event_2,
    e2.start_at AS event_2_start
FROM event_vendors ev1
JOIN event_vendors ev2
    ON ev1.vendor_id = ev2.vendor_id
    AND ev1.event_id < ev2.event_id
JOIN events e1 ON e1.id = ev1.event_id
JOIN events e2 ON e2.id = ev2.event_id
JOIN vendors v ON v.id = ev1.vendor_id
WHERE tstzrange(e1.start_at, e1.end_at) && tstzrange(e2.start_at, e2.end_at)
  AND ev1.status = 'confirmed'
  AND ev2.status = 'confirmed';

-- ================================================================
-- QUERY: AI recommendation — "Clients similar to X also booked these vendors"
-- Collaborative filtering via graph
-- ================================================================
WITH target_vendors AS (
    -- Vendors used by the target account
    SELECT DISTINCT gn_v.entity_id AS vendor_id
    FROM graph_edges ge1
    JOIN graph_nodes gn_b ON gn_b.id = ge1.target_node_id AND gn_b.node_type = 'Booking'
    JOIN graph_edges ge2 ON ge2.edge_type = 'PART_OF'
    JOIN graph_nodes gn_e ON gn_e.id = ge2.source_node_id AND gn_e.node_type = 'Event'
    JOIN graph_edges ge3 ON ge3.source_node_id = gn_e.id
    JOIN graph_nodes gn_v ON gn_v.id = ge3.target_node_id AND gn_v.node_type = 'Vendor'
    WHERE ge1.source_node_id = (SELECT id FROM graph_nodes WHERE entity_id = 'account-uuid' AND node_type = 'Account')
      AND ge1.edge_type = 'BOOKED_BY'
      AND ge3.edge_type IN ('CATERED_BY', 'PHOTOGRAPHED_BY', 'DECORATED_BY')
),
similar_accounts AS (
    -- Accounts that used at least 2 of the same vendors
    SELECT gn_a2.entity_id AS account_id, COUNT(*) AS shared_vendors
    FROM graph_edges ge1
    JOIN graph_nodes gn_a2 ON gn_a2.id = ge1.source_node_id AND gn_a2.node_type = 'Account'
    JOIN graph_edges ge2 ON ge2.edge_type = 'PART_OF'
    JOIN graph_nodes gn_e ON gn_e.id = ge2.source_node_id
    JOIN graph_edges ge3 ON ge3.source_node_id = gn_e.id
    JOIN graph_nodes gn_v ON gn_v.id = ge3.target_node_id AND gn_v.node_type = 'Vendor'
    WHERE gn_v.entity_id IN (SELECT vendor_id FROM target_vendors)
      AND gn_a2.entity_id != 'account-uuid'
      AND ge1.edge_type = 'BOOKED_BY'
    GROUP BY gn_a2.entity_id
    HAVING COUNT(*) >= 2
)
-- Recommend vendors used by similar accounts but NOT by the target
SELECT v.id, v.name, v.vendor_type, v.rating, COUNT(*) AS recommendation_strength
FROM similar_accounts sa
JOIN accounts a ON a.id = sa.account_id
JOIN bookings b ON b.account_id = a.id
JOIN events ev ON ev.booking_id = b.id
JOIN event_vendors evr ON evr.event_id = ev.id
JOIN vendors v ON v.id = evr.vendor_id
WHERE v.id NOT IN (SELECT vendor_id FROM target_vendors)
  AND v.is_active = true
GROUP BY v.id, v.name, v.vendor_type, v.rating
ORDER BY recommendation_strength DESC, v.rating DESC
LIMIT 10;
```

---

## Graph Maintenance Triggers

```sql
-- Auto-create graph nodes when relational entities are created
-- Example for bookings — similar triggers for contacts, accounts, vendors, events

CREATE OR REPLACE FUNCTION sync_booking_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    -- Create/update node
    INSERT INTO graph_nodes (org_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.org_id,
        'Booking',
        NEW.id,
        NEW.name,
        jsonb_build_object(
            'event_type', NEW.event_type,
            'guest_count', NEW.guest_count,
            'status', NEW.status,
            'start_date', NEW.start_date
        )
    )
    ON CONFLICT (node_type, entity_id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        updated_at = now();

    -- Create BOOKED_BY edge (Booking -> Account)
    IF TG_OP = 'INSERT' THEN
        INSERT INTO graph_edges (org_id, source_node_id, target_node_id, edge_type)
        SELECT NEW.org_id,
               (SELECT id FROM graph_nodes WHERE node_type = 'Booking' AND entity_id = NEW.id),
               (SELECT id FROM graph_nodes WHERE node_type = 'Account' AND entity_id = NEW.account_id),
               'BOOKED_BY'
        WHERE EXISTS (SELECT 1 FROM graph_nodes WHERE node_type = 'Account' AND entity_id = NEW.account_id);

        -- Create HELD_AT edge (Booking -> Venue)
        INSERT INTO graph_edges (org_id, source_node_id, target_node_id, edge_type)
        SELECT NEW.org_id,
               (SELECT id FROM graph_nodes WHERE node_type = 'Booking' AND entity_id = NEW.id),
               (SELECT id FROM graph_nodes WHERE node_type = 'Venue' AND entity_id = NEW.venue_id),
               'HELD_AT'
        WHERE EXISTS (SELECT 1 FROM graph_nodes WHERE node_type = 'Venue' AND entity_id = NEW.venue_id);

        -- Create PRIMARY_CONTACT edge (Booking -> Contact)
        INSERT INTO graph_edges (org_id, source_node_id, target_node_id, edge_type)
        SELECT NEW.org_id,
               (SELECT id FROM graph_nodes WHERE node_type = 'Booking' AND entity_id = NEW.id),
               (SELECT id FROM graph_nodes WHERE node_type = 'Contact' AND entity_id = NEW.contact_id),
               'PRIMARY_CONTACT'
        WHERE EXISTS (SELECT 1 FROM graph_nodes WHERE node_type = 'Contact' AND entity_id = NEW.contact_id);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_booking_graph_sync
AFTER INSERT OR UPDATE ON bookings
FOR EACH ROW EXECUTE FUNCTION sync_booking_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Identity & Multi-Tenancy | 3 | organisations, users, org_memberships |
| Venues & Spaces | 2 | venues, spaces |
| CRM | 3 | accounts, contacts, vendors (first-class) |
| Leads | 1 | leads |
| Bookings & Events | 4 | bookings, events, event_spaces, event_vendors |
| Contracts & BEOs | 2 | contracts, beos |
| Invoicing & Payments | 2 | invoices, payments |
| Equipment | 2 | equipment, event_equipment |
| Tasks | 1 | tasks |
| Sustainability | 1 | sustainability_metrics |
| Audit | 1 | audit_log |
| **Total** | **24** | Plus 2 graph tables = **26 tables** |

---

## Key Design Decisions

1. **Property graph in PostgreSQL** rather than a separate graph database. The `graph_nodes` and `graph_edges` tables implement a labelled property graph that supports recursive CTE traversals, path queries, and relationship analytics. This avoids the operational complexity of running Neo4j alongside PostgreSQL while providing 90% of the graph query power needed for this domain.

2. **Vendors as first-class entities** with their own relational table and graph nodes. Unlike the normalised model (which treats vendors as optional), the graph model makes vendor relationships central. The `event_vendors` junction table tracks operational details (contract value, rating), while graph edges capture the broader relationship network.

3. **Dual-write with triggers** keeps the graph layer synchronised with relational tables. Database triggers on bookings, events, contacts, accounts, and vendors automatically create/update graph nodes and edges. This ensures the graph is never stale without requiring application-level dual-write logic.

4. **Temporal edges** via `valid_from` / `valid_to` on `graph_edges`. When a vendor relationship ends or a contact leaves an account, the edge is end-dated rather than deleted. This preserves historical relationship data for analytics ("who used to work for this account?") and enables temporal graph queries.

5. **GDPR data subject access requests** are a graph traversal. "Show me all data related to this contact" is a recursive CTE starting from the contact node and following all edges to a configurable depth. The result set identifies every booking, event, invoice, vendor interaction, and document connected to the contact.

6. **AI-powered vendor recommendations** use collaborative filtering on the graph: find accounts that used similar vendors, then recommend vendors those accounts used but the target account has not. This is a standard graph pattern that would require multiple complex JOINs in a pure relational model but reads naturally as graph traversal.

7. **Conflict detection** (same vendor booked for overlapping events) is a join on `event_vendors` with a `tstzrange` overlap check. The graph makes it easy to extend this to deeper conflicts: "vendor X is booked for Event A, but Vendor X's subcontractor Y is also booked for competing Event B at the same time."

8. **Graph node properties are denormalised** from the relational tables. Key fields (status, event_type, guest_count) are duplicated in `graph_nodes.properties` so graph queries can filter and display results without JOINing back to the relational tables for every traversal step.

9. **Referral chain tracking** via `REFERRED_BY` edges enables the platform to trace multi-hop referral networks. "Account A was referred by Contact B, who works for Account C, which was originally a Lead referred by Vendor D." This supports referral program analytics and commission calculations.

10. **Graph layer is additive and optional.** The relational tables work independently — all CRUD operations, invoicing, booking, and scheduling function without the graph layer. The graph adds relationship intelligence on top. Teams can start with the relational core and add the graph layer when they need vendor analytics, referral tracking, or AI recommendations.
