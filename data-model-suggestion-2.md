# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Event Venue Management · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable domain event stored in an append-only event store. The event store is the single source of truth. Current state is reconstructed by replaying events or — more practically — maintained as materialised read models (projections) updated by event handlers. This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from purpose-built projections.

For event venue management this approach is compelling because the domain has natural "events" in both the software and business sense. Every booking goes through a lifecycle (enquiry, tentative hold, confirmed, contracted, completed, cancelled) and stakeholders frequently need to answer temporal questions: "What was the BEO when the client signed the contract?", "When did the guest count change from 150 to 200?", "What was the room rate when we sent the proposal?". Event sourcing answers these questions by design rather than requiring retroactive audit table bolts-on.

The approach is used by financial ledger systems, airline reservation platforms, and healthcare records systems where auditability is non-negotiable. Microsoft's Azure Architecture Center recommends event sourcing for domains requiring full audit trails, temporal queries, and complex event-driven workflows. The trade-off is increased complexity: developers must understand event replay, projections, and eventual consistency between the write and read sides.

**Best for:** Organisations that need complete audit trails, regulatory compliance evidence, temporal state reconstruction, and AI-powered analytics on change patterns over time.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is preserved forever
- (+) Temporal queries are native ("show me the booking state as of March 1")
- (+) Natural fit for event-driven integrations (webhooks, async notifications)
- (+) AI can analyse change patterns (e.g., "bookings that change guest count >3 times are 40% more likely to cancel")
- (+) Read models can be rebuilt from scratch if requirements change
- (-) Higher implementation complexity — developers must understand event replay and projections
- (-) Eventual consistency between event store and read models requires careful handling
- (-) Schema evolution of events requires versioning and upcasting logic
- (-) Debugging production issues requires understanding the event stream, not just current state
- (-) Storage grows faster than a mutable-state model (every change is a new row)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| APEX/ANSI Event Industry Standards | Event type names and BEO field vocabularies used in domain event payloads |
| iCalendar (RFC 5545) | Calendar sync events (`CalendarSyncRequested`, `CalendarEventPublished`) carry iCal UID and VEVENT data |
| AsyncAPI 3.0 | Domain events map directly to AsyncAPI channel definitions for webhook/WebSocket documentation |
| ISO 3166-1/2 | Venue location codes embedded in `VenueCreated` event payloads |
| ISO 4217 | Currency codes in all financial event payloads |
| ISO 20121:2024 | Sustainability measurement events (`SustainabilityMetricRecorded`) track ISO 20121 categories |
| PCI DSS | Payment events reference Stripe intent IDs only — no card data in event store |
| GDPR / CCPA | `ContactConsentGranted`, `ContactConsentWithdrawn`, `ContactDataDeletionRequested` events enable audit of consent lifecycle |
| OAuth 2.0 (RFC 6749) | Integration events track token lifecycle without storing secrets in the event stream |

---

## Event Store

```sql
-- The event store is the single source of truth
-- All business state is derived from replaying these events

CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    stream_type     TEXT NOT NULL,      -- 'Booking', 'Event', 'Lead', 'Contact', 'Invoice', 'Contract', 'BEO'
    stream_id       UUID NOT NULL,      -- aggregate root ID
    event_type      TEXT NOT NULL,      -- e.g., 'BookingCreated', 'GuestCountUpdated', 'PaymentReceived'
    event_version   INT NOT NULL,       -- monotonically increasing per stream
    payload         JSONB NOT NULL,     -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- {"user_id": "...", "ip_address": "...", "user_agent": "...", "correlation_id": "..."}
    causation_id    UUID,              -- the command/event that caused this event
    correlation_id  UUID,              -- groups related events across aggregates
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, event_version)  -- optimistic concurrency control
);

-- Primary query: replay events for a single aggregate
CREATE INDEX idx_es_stream ON event_store(stream_type, stream_id, event_version);

-- Query: all events for an org, ordered by time (for projections rebuild)
CREATE INDEX idx_es_org_created ON event_store(org_id, created_at);

-- Query: all events of a specific type (for cross-aggregate projections)
CREATE INDEX idx_es_event_type ON event_store(event_type, created_at);

-- Partition by month for storage management and retention
-- CREATE TABLE event_store PARTITION BY RANGE (created_at);
```

---

## Domain Event Catalogue

The following events represent the complete vocabulary of state changes in the system. Each event type has a defined payload schema.

### Booking Lifecycle Events

```sql
-- Example event payloads (stored in event_store.payload JSONB):

-- BookingCreated
-- {
--   "booking_number": "BK-2026-0042",
--   "venue_id": "uuid",
--   "account_id": "uuid",
--   "contact_id": "uuid",
--   "name": "Smith-Jones Wedding Reception",
--   "event_type": "wedding",
--   "guest_count": 150,
--   "start_date": "2026-09-15",
--   "end_date": "2026-09-15",
--   "status": "tentative"
-- }

-- BookingConfirmed
-- {
--   "confirmed_by": "uuid",
--   "deposit_required": 5000.00,
--   "confirmed_at": "2026-05-22T14:30:00Z"
-- }

-- GuestCountUpdated
-- {
--   "previous_count": 150,
--   "new_count": 200,
--   "reason": "Client increased guest list",
--   "updated_by": "uuid"
-- }

-- BookingCancelled
-- {
--   "reason": "Client relocated event to different city",
--   "cancellation_fee": 2500.00,
--   "cancelled_by": "uuid"
-- }
```

### Lead Events

```sql
-- LeadReceived
-- {
--   "source": "website_form",
--   "venue_id": "uuid",
--   "contact_name": "Jane Smith",
--   "contact_email": "jane@example.com",
--   "event_type": "corporate",
--   "preferred_date": "2026-11-20",
--   "guest_count_est": 75,
--   "message": "Looking for a venue for our annual holiday party"
-- }

-- LeadQualified
-- {
--   "ai_score": 82.5,
--   "ai_score_reasons": ["repeat_client_account", "high_budget_signal", "available_date"],
--   "qualified_by": "ai_engine"
-- }

-- LeadConverted
-- {
--   "booking_id": "uuid",
--   "converted_by": "uuid"
-- }
```

### Event (Time Block) Events

```sql
-- EventScheduled
-- {
--   "booking_id": "uuid",
--   "name": "Reception",
--   "start_at": "2026-09-15T17:00:00-04:00",
--   "end_at": "2026-09-15T23:00:00-04:00",
--   "guest_count": 200
-- }

-- SpaceAssigned
-- {
--   "event_id": "uuid",
--   "space_id": "uuid",
--   "setup_style": "Banquet",
--   "start_at": "2026-09-15T15:00:00-04:00",
--   "end_at": "2026-09-15T23:30:00-04:00"
-- }

-- EquipmentReserved
-- {
--   "event_id": "uuid",
--   "equipment_id": "uuid",
--   "quantity": 2,
--   "setup_notes": "Two wireless lapel microphones for speeches"
-- }
```

### Financial Events

```sql
-- InvoiceIssued
-- {
--   "invoice_number": "INV-2026-0089",
--   "booking_id": "uuid",
--   "line_items": [
--     {"description": "Grand Ballroom rental (full day)", "amount": 8500.00, "category": "space_rental"},
--     {"description": "Dinner service (200 guests)", "amount": 15000.00, "category": "catering"},
--     {"description": "AV package", "amount": 2500.00, "category": "av_equipment"}
--   ],
--   "subtotal": 26000.00,
--   "tax_rate": 0.085,
--   "tax_amount": 2210.00,
--   "total": 28210.00,
--   "currency": "USD",
--   "due_date": "2026-08-15"
-- }

-- PaymentReceived
-- {
--   "invoice_id": "uuid",
--   "amount": 5000.00,
--   "currency": "USD",
--   "payment_method": "credit_card",
--   "stripe_payment_intent_id": "pi_3abc123def456",
--   "paid_at": "2026-06-01T10:15:00Z"
-- }

-- PaymentRefunded
-- {
--   "payment_id": "uuid",
--   "refund_amount": 2500.00,
--   "reason": "Partial cancellation — ceremony space no longer needed",
--   "stripe_refund_id": "re_xyz789"
-- }
```

### Contract Events

```sql
-- ContractDrafted
-- {
--   "booking_id": "uuid",
--   "contract_number": "CTR-2026-0042",
--   "template_id": "uuid",
--   "total_amount": 28210.00
-- }

-- ContractSent
-- {
--   "sent_to_email": "jane@example.com",
--   "sent_at": "2026-06-05T09:00:00Z"
-- }

-- ContractSigned
-- {
--   "signer_name": "Jane Smith",
--   "signer_email": "jane@example.com",
--   "signer_ip": "192.168.1.100",
--   "signed_at": "2026-06-06T14:22:00Z",
--   "signature_hash": "sha256:abcdef123456"
-- }
```

### BEO Events

```sql
-- BEOGenerated
-- {
--   "beo_number": "BEO-2026-0042-v1",
--   "booking_id": "uuid",
--   "event_id": "uuid",
--   "generated_by": "ai_engine",
--   "source_brief": "Wedding reception for 200 guests, plated dinner, open bar...",
--   "meal_services": [...],
--   "setup_style": "Banquet",
--   "av_requirements": [...]
-- }

-- BEOApproved
-- {
--   "approved_by": "uuid",
--   "approved_at": "2026-08-01T16:00:00Z",
--   "version": 3
-- }

-- BEODistributed
-- {
--   "distributed_to": ["kitchen", "front_of_house", "av_team", "setup_crew"],
--   "distributed_at": "2026-09-14T08:00:00Z"
-- }
```

### GDPR/Privacy Events

```sql
-- ContactConsentGranted
-- {
--   "contact_id": "uuid",
--   "consent_type": "marketing_email",
--   "granted_at": "2026-05-20T10:00:00Z",
--   "ip_address": "192.168.1.50"
-- }

-- ContactDataDeletionRequested
-- {
--   "contact_id": "uuid",
--   "requested_at": "2026-07-15T09:00:00Z",
--   "reason": "GDPR right to erasure",
--   "deadline": "2026-08-14"
-- }

-- ContactDataDeleted
-- {
--   "contact_id": "uuid",
--   "fields_deleted": ["email", "phone", "mobile", "address"],
--   "deleted_at": "2026-07-20T11:00:00Z",
--   "deleted_by": "system"
-- }
```

---

## Read Model Projections (Materialised Views)

The read side consists of denormalised tables optimised for specific query patterns. These are rebuilt from the event store by projection handlers.

```sql
-- ================================================================
-- PROJECTION: Current Booking State
-- Rebuilt by replaying all Booking stream events
-- ================================================================
CREATE TABLE proj_bookings (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    venue_id        UUID NOT NULL,
    account_id      UUID NOT NULL,
    contact_id      UUID NOT NULL,
    booking_number  TEXT NOT NULL,
    name            TEXT NOT NULL,
    event_type      TEXT NOT NULL,
    status          TEXT NOT NULL,
    guest_count     INT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    total_invoiced  NUMERIC(14,2) NOT NULL DEFAULT 0,
    total_paid      NUMERIC(14,2) NOT NULL DEFAULT 0,
    balance_due     NUMERIC(14,2) NOT NULL DEFAULT 0,
    assigned_to     UUID,
    lead_id         UUID,
    event_count     INT NOT NULL DEFAULT 0,
    last_event_version INT NOT NULL,   -- tracks which event was last processed
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_bookings_org ON proj_bookings(org_id);
CREATE INDEX idx_proj_bookings_org_status ON proj_bookings(org_id, status);
CREATE INDEX idx_proj_bookings_dates ON proj_bookings(start_date, end_date);
CREATE INDEX idx_proj_bookings_venue ON proj_bookings(venue_id);

-- ================================================================
-- PROJECTION: Space Availability
-- Optimised for availability queries and double-booking prevention
-- ================================================================
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE proj_space_reservations (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    space_id        UUID NOT NULL,
    event_id        UUID NOT NULL,
    booking_id      UUID NOT NULL,
    booking_name    TEXT NOT NULL,
    setup_style     TEXT,
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL,      -- active, cancelled

    EXCLUDE USING gist (
        space_id WITH =,
        tstzrange(start_at, end_at) WITH &&
    ) WHERE (status = 'active')
);

CREATE INDEX idx_proj_space_res_space ON proj_space_reservations(space_id, start_at);
CREATE INDEX idx_proj_space_res_org ON proj_space_reservations(org_id);

-- ================================================================
-- PROJECTION: Lead Pipeline
-- Optimised for sales dashboard queries
-- ================================================================
CREATE TABLE proj_lead_pipeline (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    venue_id        UUID,
    contact_name    TEXT,
    contact_email   TEXT,
    source          TEXT NOT NULL,
    status          TEXT NOT NULL,
    event_type      TEXT,
    preferred_date  DATE,
    guest_count_est INT,
    budget_est      NUMERIC(12,2),
    ai_score        NUMERIC(5,2),
    assigned_to     UUID,
    converted_booking_id UUID,
    response_time_mins INT,            -- calculated from events
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_leads_org_status ON proj_lead_pipeline(org_id, status);

-- ================================================================
-- PROJECTION: Financial Summary per Booking
-- Optimised for revenue reporting
-- ================================================================
CREATE TABLE proj_booking_financials (
    booking_id      UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    venue_id        UUID NOT NULL,
    booking_number  TEXT NOT NULL,
    event_type      TEXT NOT NULL,
    start_date      DATE NOT NULL,
    total_invoiced  NUMERIC(14,2) NOT NULL DEFAULT 0,
    total_paid      NUMERIC(14,2) NOT NULL DEFAULT 0,
    total_refunded  NUMERIC(14,2) NOT NULL DEFAULT 0,
    balance_due     NUMERIC(14,2) NOT NULL DEFAULT 0,
    invoice_count   INT NOT NULL DEFAULT 0,
    payment_count   INT NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    last_payment_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_financials_org ON proj_booking_financials(org_id);
CREATE INDEX idx_proj_financials_venue ON proj_booking_financials(venue_id);
CREATE INDEX idx_proj_financials_dates ON proj_booking_financials(start_date);

-- ================================================================
-- PROJECTION: BEO Current Version
-- Optimised for kitchen and operations staff
-- ================================================================
CREATE TABLE proj_beos (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    booking_id      UUID NOT NULL,
    event_id        UUID NOT NULL,
    beo_number      TEXT NOT NULL,
    version         INT NOT NULL,
    status          TEXT NOT NULL,
    event_name      TEXT NOT NULL,
    event_date      DATE NOT NULL,
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    guest_count     INT NOT NULL,
    guaranteed_count INT,
    room_name       TEXT NOT NULL,
    setup_style     TEXT,
    meal_services   JSONB NOT NULL DEFAULT '[]',
    av_requirements JSONB NOT NULL DEFAULT '[]',
    setup_notes     TEXT,
    special_requests TEXT,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_beos_booking ON proj_beos(booking_id);
CREATE INDEX idx_proj_beos_date ON proj_beos(event_date);

-- ================================================================
-- PROJECTION: Contact Directory
-- Optimised for CRM lookups
-- ================================================================
CREATE TABLE proj_contacts (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    account_id      UUID,
    account_name    TEXT,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    job_title       TEXT,
    booking_count   INT NOT NULL DEFAULT 0,
    total_revenue   NUMERIC(14,2) NOT NULL DEFAULT 0,
    last_booking_date DATE,
    consent_status  TEXT NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_contacts_org ON proj_contacts(org_id);
CREATE INDEX idx_proj_contacts_email ON proj_contacts(email);
CREATE INDEX idx_proj_contacts_account ON proj_contacts(account_id);

-- ================================================================
-- PROJECTION: Sustainability Dashboard
-- Aggregated sustainability metrics per event, venue, and org
-- ================================================================
CREATE TABLE proj_sustainability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    event_id        UUID NOT NULL,
    venue_id        UUID NOT NULL,
    event_date      DATE NOT NULL,
    carbon_kg       NUMERIC(14,4) DEFAULT 0,
    waste_kg        NUMERIC(14,4) DEFAULT 0,
    recycled_kg     NUMERIC(14,4) DEFAULT 0,
    water_litres    NUMERIC(14,4) DEFAULT 0,
    energy_kwh      NUMERIC(14,4) DEFAULT 0,
    food_waste_kg   NUMERIC(14,4) DEFAULT 0,
    guest_count     INT,
    carbon_per_guest NUMERIC(10,4),     -- calculated
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_sustain_org ON proj_sustainability(org_id);
CREATE INDEX idx_proj_sustain_venue ON proj_sustainability(venue_id);
CREATE INDEX idx_proj_sustain_date ON proj_sustainability(event_date);
```

---

## Snapshot Table (Performance Optimisation)

```sql
-- Snapshots store periodic aggregate state to avoid full event replay
-- Replay only events after the snapshot version

CREATE TABLE event_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    snapshot_version INT NOT NULL,      -- event_version at time of snapshot
    state           JSONB NOT NULL,     -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, snapshot_version)
);

CREATE INDEX idx_snapshots_stream ON event_snapshots(stream_type, stream_id, snapshot_version DESC);
```

---

## Projection Checkpoint Tracking

```sql
-- Tracks which event each projection has processed up to
-- Used for rebuilding projections and resuming after failures

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,   -- 'proj_bookings', 'proj_space_reservations', etc.
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'running',  -- running, paused, rebuilding, error
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Temporal Query Examples

```sql
-- "What was the guest count for booking X on March 1, 2026?"
-- Replay GuestCountUpdated events up to that date

SELECT payload->>'new_count' AS guest_count
FROM event_store
WHERE stream_type = 'Booking'
  AND stream_id = 'booking-uuid-here'
  AND event_type IN ('BookingCreated', 'GuestCountUpdated')
  AND created_at <= '2026-03-01T23:59:59Z'
ORDER BY event_version DESC
LIMIT 1;

-- "How many times did this booking's guest count change?"
SELECT COUNT(*) AS change_count,
       MIN(payload->>'previous_count') AS original_count,
       MAX(payload->>'new_count') AS final_count
FROM event_store
WHERE stream_type = 'Booking'
  AND stream_id = 'booking-uuid-here'
  AND event_type = 'GuestCountUpdated';

-- "Show me the full history of booking X"
SELECT event_type, payload, metadata, created_at
FROM event_store
WHERE stream_type = 'Booking'
  AND stream_id = 'booking-uuid-here'
ORDER BY event_version ASC;

-- "What was the BEO when the contract was signed?"
-- Find the contract signed timestamp, then replay BEO events up to that point
WITH contract_signed AS (
    SELECT created_at AS signed_at
    FROM event_store
    WHERE stream_type = 'Contract'
      AND stream_id = 'contract-uuid-here'
      AND event_type = 'ContractSigned'
    LIMIT 1
)
SELECT payload
FROM event_store, contract_signed
WHERE stream_type = 'BEO'
  AND stream_id = 'beo-uuid-here'
  AND created_at <= contract_signed.signed_at
ORDER BY event_version DESC
LIMIT 1;

-- "AI insight: Bookings with >3 guest count changes have higher cancellation rates"
WITH change_counts AS (
    SELECT stream_id,
           COUNT(*) FILTER (WHERE event_type = 'GuestCountUpdated') AS changes,
           bool_or(event_type = 'BookingCancelled') AS was_cancelled
    FROM event_store
    WHERE stream_type = 'Booking'
    GROUP BY stream_id
)
SELECT
    CASE WHEN changes > 3 THEN 'high_changes' ELSE 'low_changes' END AS change_group,
    COUNT(*) AS bookings,
    COUNT(*) FILTER (WHERE was_cancelled) AS cancelled,
    ROUND(100.0 * COUNT(*) FILTER (WHERE was_cancelled) / COUNT(*), 1) AS cancel_pct
FROM change_counts
GROUP BY 1;
```

---

## Reference Data Tables

```sql
-- Minimal mutable reference data (not event-sourced)
-- These rarely change and don't need full event history

CREATE TABLE ref_venues (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    name            TEXT NOT NULL,
    timezone        TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE ref_spaces (
    id              UUID PRIMARY KEY,
    venue_id        UUID NOT NULL REFERENCES ref_venues(id),
    name            TEXT NOT NULL,
    capacity_max    INT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE ref_menu_items (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    name            TEXT NOT NULL,
    category        TEXT NOT NULL,
    price           NUMERIC(10,2) NOT NULL,
    dietary_tags    TEXT[],
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE ref_equipment (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    name            TEXT NOT NULL,
    category        TEXT NOT NULL,
    rental_rate     NUMERIC(10,2),
    quantity_total  INT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | event_store — single append-only table, partitioned by month |
| Snapshots & Checkpoints | 2 | event_snapshots, projection_checkpoints |
| Read Projections | 7 | proj_bookings, proj_space_reservations, proj_lead_pipeline, proj_booking_financials, proj_beos, proj_contacts, proj_sustainability |
| Reference Data | 4 | ref_venues, ref_spaces, ref_menu_items, ref_equipment |
| **Total** | **14** | Plus event store partitions |

---

## Key Design Decisions

1. **Single event store table** rather than per-aggregate event tables. This simplifies infrastructure (one table to partition, index, and back up) while the `stream_type` + `stream_id` composite provides aggregate isolation. Partitioning by `created_at` enables efficient retention management.

2. **Optimistic concurrency via unique constraint** on `(stream_type, stream_id, event_version)`. When two concurrent commands try to append event version N to the same aggregate, one succeeds and the other gets a unique constraint violation, forcing a retry with the latest state. This prevents lost updates without distributed locks.

3. **Separation of payload and metadata** in the event store. The `payload` contains domain-specific event data (what happened), while `metadata` contains cross-cutting concerns (who did it, from where, correlation IDs). This keeps event schemas clean and makes GDPR data scrubbing possible on metadata without touching domain events.

4. **Projections are disposable and rebuildable.** If a projection's schema changes or its data becomes corrupted, the projection handler can be stopped, the table truncated, and all events replayed from the event store. The `projection_checkpoints` table tracks progress for resume-after-failure.

5. **Snapshots for performance.** Long-lived aggregates (a booking with hundreds of events over months of planning) would be slow to rebuild from scratch. Periodic snapshots store the aggregate state at a known event version, so replay only needs to process events after the snapshot.

6. **Reference data is NOT event-sourced.** Venues, spaces, menu items, and equipment are mutable reference data that changes infrequently. Event-sourcing these would add complexity without proportional benefit. They are stored in simple relational tables.

7. **Exclusion constraint on the space reservation projection** prevents double-booking at the read-model level. The write side validates availability by checking this projection before accepting a `SpaceAssigned` command, providing strong consistency for the most critical business rule.

8. **GDPR compliance through event metadata.** Contact consent events (`ContactConsentGranted`, `ContactConsentWithdrawn`) create an immutable audit trail of consent lifecycle. For right-to-erasure, personal data in event payloads can be crypto-shredded (encrypted with a per-contact key that is destroyed on deletion request) rather than mutating the event store.

9. **AI analytics on event patterns.** The event store enables queries that are impossible in a mutable-state model: "bookings that had >3 BEO revisions correlated with higher client satisfaction scores", "average time from lead receipt to first response by venue", "seasonal patterns in guest count changes". These feed the AI-native features.

10. **AsyncAPI-ready event catalogue.** Every domain event type maps to an AsyncAPI channel definition, making it straightforward to expose real-time webhooks (booking confirmed, payment received, BEO approved) to third-party integrations with formal documentation.
