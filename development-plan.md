# Event Venue Management — Phased Development Plan

> Project: 262-event-venue-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four
`data-model-suggestion-*.md` documents. The chosen data architecture is **Suggestion 3 — Hybrid
Relational + JSONB** (typed columns for core booking/financial fields, JSONB for venue-type-specific
data), augmented with the **PostgreSQL exclusion-constraint double-booking guarantee from Suggestion 1**
and an **AsyncAPI-documented webhook/event stream inspired by Suggestion 2**. The graph layer
(Suggestion 4) is deferred to the backlog.

---

## Core Requirements (synthesis)

- **What it does:** A unified, AI-native, open-source platform for venue operators to manage space
  bookings, CRM/leads, contracts, invoicing/payments, catering and BEOs, AV/equipment, and post-event
  reporting — without six-figure enterprise contracts or hospitality-only lock-in.
- **Primary users:** Independent venues (wedding barns, conference suites), hotel/restaurant event
  departments, convention-centre ops teams, corporate campus facility managers, university event offices.
- **Differentiators (AI-native):** Lead qualification + proposal generation; natural-language → BEO
  generation; dynamic space pricing; post-event analysis; an MCP server exposing venue tools to AI agents.
- **Deployment model:** Self-hostable (Docker Compose) SaaS-style multi-tenant web app + REST API +
  documented webhooks + optional MCP server. API-first; no third-party lock-in.
- **Integration surface:** Stripe (payments), Google Calendar + Microsoft Graph + CalDAV (availability
  sync), LLM provider (Anthropic / OpenAI-compatible) for AI features, outbound webhooks.
- **Standards:** OpenAPI 3.1, AsyncAPI 3.0 (webhooks), iCalendar RFC 5545 / CalDAV RFC 4791 / iTIP RFC
  5546, OAuth 2.0 (RFC 6749/6750) + OpenID Connect, PCI DSS (Stripe-delegated), GDPR/CCPA, ISO 3166/4217,
  ISO 20121:2024 (sustainability), APEX ESG (BEO fields), OWASP API Security Top 10, MCP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **Python 3.12** | AI/LLM-heavy domain (lead scoring, BEO generation, dynamic pricing, post-event analysis); first-class Anthropic/OpenAI SDKs; mature Stripe/Google/Microsoft SDKs; Pydantic gives strong typing for OpenAPI generation. |
| API framework | **FastAPI** | Auto-generates OpenAPI 3.1 (a stated requirement); async-native for webhook/LLM I/O; Pydantic request/response models map 1:1 to JSON Schema Draft 2020-12; dependency-injection model fits per-tenant auth. |
| Data validation | **Pydantic v2** | Single source of truth for API schemas, JSONB payload validation, and the published OpenAPI component library (the "open venue booking schema" opportunity noted in standards.md). |
| Database | **PostgreSQL 16** | Required for `btree_gist` exclusion constraints (double-booking prevention), JSONB + GIN indexes (hybrid model), Row-Level Security (multi-tenant isolation), `tstzrange` availability queries. SQLite is inadequate for these. |
| ORM / migrations | **SQLAlchemy 2.0 (async) + Alembic** | Async ORM matches FastAPI; Alembic handles versioned migrations including the JSONB CHECK constraints and exclusion constraints. |
| JSONB validation | **pg_jsonschema** extension + Pydantic | Validates JSONB structure at the DB layer (per Suggestion 3) and the app layer. |
| Task queue | **ARQ (Redis-based)** | Lightweight async job queue for webhook delivery, calendar sync, LLM calls (lead scoring, BEO gen), and email. Simpler ops than Celery; async-native; uses the Redis we already need for caching. |
| Cache / queue broker | **Redis 7** | ARQ broker, rate limiting, idempotency keys, free/busy availability cache. |
| LLM access | **Anthropic SDK** (primary) via a provider-abstraction layer; OpenAI-compatible fallback | AI-native features; abstraction keeps the platform provider-neutral (no lock-in, matching the project ethos). Prompt caching enabled. |
| Auth | **OAuth 2.0 / OIDC** via Authlib; internal JWT sessions | Required for Google/Microsoft/Stripe integration and enterprise SSO (standards.md). Argon2 password hashing for local accounts. |
| Payments | **Stripe** (Python SDK) | PCI DSS scope reduction via tokenisation; webhook signature verification; matches incumbent integration patterns. |
| Calendar | **icalendar** + **caldav** libs; Google/Microsoft Graph SDKs | RFC 5545 export, CalDAV sync, provider-specific delta sync. |
| Frontend | **Next.js 15 (App Router, React 19, TypeScript)** + **shadcn/ui** + **Tailwind** | Operator dashboard + self-serve booking portal. Server Components for data-heavy views; consumes the FastAPI OpenAPI spec via a generated TypeScript client. |
| API client gen | **openapi-typescript** | Generates the frontend's typed client directly from the published OpenAPI 3.1 spec — no hand-written API bindings. |
| MCP server | **mcp Python SDK** | Exposes availability/booking/BEO/lead tools to AI agents (standards.md MCP section). |
| Containerisation | **Docker + docker-compose** | Self-hosted deployment; services: api, worker, db, redis, web. |
| Testing | **pytest + pytest-asyncio + httpx + testcontainers** (Python); **Vitest + Playwright** (frontend) | testcontainers spins real Postgres/Redis for integration tests; Playwright for portal/dashboard E2E. |
| Quality | **ruff** (lint+format), **mypy** (types), **pre-commit** | Standard Python toolchain. **eslint + prettier** for the frontend. |
| Package mgmt | **uv** (Python), **pnpm** (frontend) | Fast, reproducible installs. |
| Observability | **structlog** (JSON logs) + OpenTelemetry hooks | Structured audit/security logging (OWASP). |

### Project Structure

```
event-venue-management/
├── README.md
├── docker-compose.yml
├── .env.example
├── Makefile
├── api/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── Dockerfile
│   ├── alembic/
│   │   ├── env.py
│   │   └── versions/
│   ├── src/evm/
│   │   ├── main.py                 # FastAPI app factory, OpenAPI metadata
│   │   ├── config.py               # Pydantic Settings (env-driven)
│   │   ├── db/
│   │   │   ├── base.py             # async engine, session, Base
│   │   │   ├── models/             # SQLAlchemy models (one module per domain)
│   │   │   └── rls.py              # Row-Level Security helpers / tenant context
│   │   ├── schemas/                # Pydantic models = OpenAPI components (the open schema lib)
│   │   ├── auth/                   # OAuth2/OIDC, JWT, password hashing, RBAC deps
│   │   ├── tenancy/                # org context, RLS session vars, membership checks
│   │   ├── domain/                 # business logic services (pure, testable)
│   │   │   ├── availability.py
│   │   │   ├── bookings.py
│   │   │   ├── pricing.py
│   │   │   ├── invoicing.py
│   │   │   ├── beo.py
│   │   │   └── leads.py
│   │   ├── api/                    # FastAPI routers (one per resource)
│   │   ├── integrations/
│   │   │   ├── stripe_gateway.py
│   │   │   ├── calendar/           # ical export, caldav, google, microsoft
│   │   │   └── llm/                # provider abstraction + prompts
│   │   ├── events/                 # domain-event emission + AsyncAPI webhook delivery
│   │   ├── workers/                # ARQ task definitions
│   │   ├── mcp/                    # MCP server exposing venue tools
│   │   └── observability/          # structlog, audit log writer
│   └── tests/
│       ├── unit/
│       ├── integration/            # testcontainers: real PG + Redis
│       ├── e2e/
│       └── fixtures/               # sample briefs, webhook payloads, seed data
├── web/
│   ├── package.json
│   ├── Dockerfile
│   ├── src/
│   │   ├── app/                    # Next.js App Router (dashboard + /book portal)
│   │   ├── lib/api/                # openapi-typescript generated client
│   │   ├── components/             # shadcn/ui components
│   │   └── tests/                  # Vitest + Playwright
└── spec/
    ├── openapi.json                # generated, committed for SDK/doc generation
    └── asyncapi.yaml               # webhook event catalogue (AsyncAPI 3.0)
```

The structure groups by concern (db, schemas, domain, api, integrations, events, workers). Each phase
adds modules without restructuring.

---

## Phase 1: Foundation, Multi-Tenancy & Auth

### Purpose
Stand up the project skeleton, database, configuration, the multi-tenant security model, and
authentication. After this phase the API boots, runs migrations, enforces tenant isolation at the
database layer, and authenticates users — the bedrock every other phase relies on. This directly
addresses OWASP API#1 (BOLA) by making tenant isolation a database invariant rather than an
application afterthought.

### Tasks

#### 1.1 — Project scaffold, config, Docker

**What:** Bootstrap the `api/` package, `docker-compose.yml`, config, and tooling.

**Design:**
- `docker-compose.yml` services: `db` (postgres:16 with `btree_gist`, `pg_jsonschema`), `redis:7`,
  `api`, `worker`, `web`. Healthchecks on db/redis; api depends on healthy db.
- `config.py` using Pydantic `BaseSettings`:
```python
class Settings(BaseSettings):
    database_url: PostgresDsn
    redis_url: RedisDsn
    jwt_secret: SecretStr
    jwt_ttl_minutes: int = 60
    refresh_ttl_days: int = 30
    llm_provider: Literal["anthropic", "openai"] = "anthropic"
    llm_api_key: SecretStr | None = None
    stripe_secret_key: SecretStr | None = None
    stripe_webhook_secret: SecretStr | None = None
    encryption_key: SecretStr            # Fernet key for oauth_tokens at rest
    environment: Literal["dev", "test", "prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="EVM_", env_file=".env")
```
- `main.py`: FastAPI app factory with OpenAPI 3.1 metadata (`title`, `version`, `servers`, tags),
  `/healthz` (liveness) and `/readyz` (db+redis check) endpoints.
- `Makefile` targets: `up`, `migrate`, `test`, `lint`, `typecheck`, `seed`, `openapi` (dump spec).

**Testing:**
- `Unit: Settings loads from env with EVM_ prefix → correct typed values, SecretStr masks secrets`
- `Unit: missing required DATABASE_URL → ValidationError naming the field`
- `Integration: GET /healthz → 200 {"status":"ok"}`
- `Integration (testcontainers): GET /readyz with live PG+Redis → 200; with redis down → 503`

#### 1.2 — Database base, migrations, extensions

**What:** Async SQLAlchemy engine/session, Alembic, and the initial migration enabling required extensions.

**Design:**
- `db/base.py`: `create_async_engine`, `async_sessionmaker`, declarative `Base` with naming
  convention for constraints (deterministic Alembic diffs).
- Alembic `env.py` configured for async + autogenerate against `Base.metadata`.
- Migration `0001_extensions`: `CREATE EXTENSION IF NOT EXISTS "uuid-ossp", btree_gist, pg_jsonschema;`
- FastAPI dependency `get_session()` yielding an `AsyncSession` per request.

**Testing:**
- `Integration (testcontainers): run alembic upgrade head → extensions present (query pg_extension)`
- `Integration: get_session yields a usable session; rolls back on exception`

#### 1.3 — Identity & tenancy schema + Row-Level Security

**What:** `organisations`, `users`, `org_memberships` tables and PostgreSQL RLS enforcing `org_id` isolation.

**Design:** (from Suggestion 1/3 core identity)
```sql
CREATE TABLE organisations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(), name TEXT NOT NULL, slug TEXT NOT NULL UNIQUE,
  billing_email TEXT, stripe_customer_id TEXT, timezone TEXT NOT NULL DEFAULT 'UTC',
  country_code CHAR(2) NOT NULL, currency CHAR(3) NOT NULL DEFAULT 'USD',
  settings JSONB NOT NULL DEFAULT '{}', created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now());
CREATE TABLE users ( id UUID PRIMARY KEY DEFAULT gen_random_uuid(), email TEXT NOT NULL UNIQUE,
  full_name TEXT NOT NULL, phone TEXT, password_hash TEXT, is_active BOOLEAN NOT NULL DEFAULT true,
  consent_status TEXT NOT NULL DEFAULT 'pending', data_retention_until DATE,
  last_login_at TIMESTAMPTZ, created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now());
CREATE TABLE org_memberships ( id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organisations(id), user_id UUID NOT NULL REFERENCES users(id),
  role TEXT NOT NULL DEFAULT 'member', is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (org_id, user_id));
```
- RLS pattern: every tenant-scoped table gets `ALTER TABLE x ENABLE ROW LEVEL SECURITY` and a policy
  `USING (org_id = current_setting('evm.current_org_id')::uuid)`. `tenancy/rls.py` sets the session
  var `SET LOCAL evm.current_org_id = :org` at the start of each request transaction.
- Roles enum: `owner, admin, manager, member, viewer`. RBAC dependency `require_role(min_role)`.

**Testing:**
- `Integration: session scoped to org A cannot SELECT org B rows (RLS) → 0 rows`
- `Integration: insert with mismatched org_id → RLS policy violation`
- `Unit: role hierarchy — manager satisfies require_role('member'), viewer does not`

#### 1.4 — Authentication & authorization

**What:** Local login (email+password), JWT access/refresh tokens, OIDC login, and per-request tenant context.

**Design:**
- `POST /auth/register` → creates user + org + owner membership.
- `POST /auth/login` {email, password} → {access_token, refresh_token}. Argon2 verify; constant-time.
- `POST /auth/refresh`, `POST /auth/logout` (refresh-token revocation list in Redis).
- `GET /auth/oidc/{provider}/start` + `/callback` via Authlib (Google/Microsoft/Azure AD).
- JWT claims: `sub` (user_id), `orgs` (memberships), `act_org` (active org), `role`, `exp`.
- FastAPI deps: `current_user`, `current_org` (validates membership, sets RLS var), `require_role`.
- Security headers + CORS configured; rate limit on `/auth/*` via Redis (OWASP API#2).

**Testing:**
- `Unit: Argon2 hash/verify round-trip; wrong password → False`
- `Integration: register → login → access protected route 200; no token → 401; expired → 401`
- `Integration: refresh after logout (revoked) → 401`
- `Integration (mocked OIDC): callback with valid code → user provisioned, JWT issued`
- `Integration: user without membership in act_org → 403`

---

## Phase 2: Venues, Spaces & the Open Schema Library

### Purpose
Model the physical inventory — venues, spaces (with divisible sub-rooms), setup styles, amenities — and
establish the Pydantic schema library that doubles as the published OpenAPI component set (the
"open venue booking schema" gap identified in standards.md). After this phase operators can define
their venues and the API exposes a stable, documented resource model.

### Tasks

#### 2.1 — Venue & space schema (hybrid relational + JSONB)

**What:** `venues`, `spaces`, `setup_styles`, `space_setup_configs`, `amenities`, `space_amenities`.

**Design:** Typed columns for universal fields (capacity, rates, ISO codes, FKs); JSONB `attributes`
for venue-type-specific data (e.g. bridal-suite access, exhibitor-booth specs). From Suggestion 3:
```sql
CREATE TABLE venues ( id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organisations(id), name TEXT NOT NULL, slug TEXT NOT NULL,
  description TEXT, address_line1 TEXT, city TEXT, region_code TEXT, postal_code TEXT,
  country_code CHAR(2) NOT NULL, latitude NUMERIC(10,7), longitude NUMERIC(10,7),
  timezone TEXT NOT NULL DEFAULT 'UTC', phone TEXT, email TEXT, website_url TEXT,
  attributes JSONB NOT NULL DEFAULT '{}', is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (org_id, slug));
CREATE TABLE spaces ( id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  venue_id UUID NOT NULL REFERENCES venues(id), parent_space_id UUID REFERENCES spaces(id),
  name TEXT NOT NULL, capacity_min INT, capacity_max INT NOT NULL, area_sqft NUMERIC(10,2),
  floor_level INT, is_outdoor BOOLEAN NOT NULL DEFAULT false, is_divisible BOOLEAN NOT NULL DEFAULT false,
  hourly_rate NUMERIC(12,2), half_day_rate NUMERIC(12,2), full_day_rate NUMERIC(12,2),
  currency CHAR(3) NOT NULL DEFAULT 'USD', attributes JSONB NOT NULL DEFAULT '{}',
  is_active BOOLEAN NOT NULL DEFAULT true, created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now());
-- setup_styles, space_setup_configs (UNIQUE(space_id,setup_style_id)), amenities,
-- space_amenities (PK space_id+amenity_id) per Suggestion 1.
```
- GIN index on `spaces.attributes` and `venues.attributes` for containment queries.
- `pg_jsonschema` CHECK constraint validates `attributes` against a per-org-extensible schema.

**Testing:**
- `Unit: SpaceCreate Pydantic — capacity_max < capacity_min → ValidationError`
- `Integration: create venue + space → GET returns hierarchy with sub-rooms`
- `Integration: GIN containment query attributes @> '{"has_bridal_suite":true}' returns matches`

#### 2.2 — Resource CRUD routers + Pydantic schema library

**What:** REST routers for venues/spaces/setup styles/amenities; the `schemas/` package as the
OpenAPI component library.

**Design:**
- RESTful resources: `GET/POST /venues`, `GET/PATCH/DELETE /venues/{id}`, nested
  `/venues/{id}/spaces`, `/spaces/{id}/setup-configs`, etc.
- Each resource has `XCreate`, `XUpdate`, `XRead` Pydantic models. `XRead` includes `id`, timestamps,
  and `org_id` (read-only). All list endpoints paginate (`limit`/`cursor`) and filter.
- `make openapi` dumps `spec/openapi.json`; CI fails if it drifts from committed spec.
- Object-level authorization: every fetch joins on `org_id` via RLS — no manual `WHERE org_id` (BOLA defence).

**Testing:**
- `Integration: full CRUD lifecycle on /venues (create→read→update→delete)`
- `Integration: viewer role cannot POST (403); manager can`
- `Integration: list pagination — cursor returns next page, no duplicates`
- `Contract: generated openapi.json validates against OAS 3.1 meta-schema`

---

## Phase 3: Bookings, Events & Availability (Core Value)

### Purpose
The heart of the product: bookings (contract-level grouping) containing events (time blocks), space
assignments, and a database-enforced double-booking guarantee. After this phase the platform can
prevent scheduling conflicts and answer availability queries — the single most critical business rule.

### Tasks

#### 3.1 — Booking > Event > space-assignment schema

**What:** `bookings`, `events`, `event_spaces` with the exclusion-constraint double-booking guard.

**Design:** (Booking>Event hierarchy from Suggestion 1; exclusion constraint is the key safety property)
```sql
CREATE TABLE bookings ( id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organisations(id), venue_id UUID NOT NULL REFERENCES venues(id),
  account_id UUID NOT NULL, contact_id UUID NOT NULL, lead_id UUID,
  booking_number TEXT NOT NULL, name TEXT NOT NULL, event_type TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'tentative', -- tentative,confirmed,contracted,in_progress,completed,cancelled
  guest_count INT NOT NULL, start_date DATE NOT NULL, end_date DATE NOT NULL,
  attributes JSONB NOT NULL DEFAULT '{}', notes TEXT, internal_notes TEXT, assigned_to UUID,
  cancelled_at TIMESTAMPTZ, cancelled_reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (org_id, booking_number));
CREATE TABLE events ( id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE, name TEXT NOT NULL,
  start_at TIMESTAMPTZ NOT NULL, end_at TIMESTAMPTZ NOT NULL, setup_start_at TIMESTAMPTZ,
  teardown_end_at TIMESTAMPTZ, guest_count INT, status TEXT NOT NULL DEFAULT 'scheduled',
  ical_uid TEXT, notes TEXT, created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now());
CREATE TABLE event_spaces ( id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  space_id UUID NOT NULL REFERENCES spaces(id), setup_style_id UUID REFERENCES setup_styles(id),
  status TEXT NOT NULL DEFAULT 'active', -- active, cancelled
  start_at TIMESTAMPTZ NOT NULL, end_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(), UNIQUE (event_id, space_id),
  EXCLUDE USING gist (space_id WITH =, tstzrange(start_at, end_at) WITH &&) WHERE (status = 'active'));
```
- `start_at`/`end_at` on `event_spaces` include setup+teardown buffers so reservations block the room
  for the whole occupied window.
- Booking-number generation: `BK-{YYYY}-{seq}` per org via a sequence table (gap-tolerant).

**Testing:**
- `Integration: assign space A to two overlapping active events → second insert raises IntegrityError (exclusion)`
- `Integration: cancelled reservation does not block a new overlapping active one`
- `Integration: booking_number monotonic per org, isolated across orgs`
- `Unit: end_at <= start_at → ValidationError before DB`

#### 3.2 — Availability service & query API

**What:** A service answering "is space S free between T1 and T2?" and "what spaces are free on date D?"

**Design:**
```python
async def is_available(session, space_id: UUID, start: datetime, end: datetime,
                       exclude_event_id: UUID | None = None) -> bool: ...
async def find_available_spaces(session, venue_id: UUID, start: datetime, end: datetime,
                                min_capacity: int, setup_style_id: UUID | None) -> list[SpaceRead]: ...
```
- Implemented via `tstzrange && tstzrange` overlap query against active `event_spaces`, respecting
  divisible-space hierarchy (booking a parent blocks children and vice-versa via `parent_space_id`
  expansion).
- Endpoints: `GET /venues/{id}/availability?start=&end=&min_capacity=&setup_style=`,
  `GET /spaces/{id}/availability?start=&end=`.
- Results cached in Redis keyed by `(space_id, day)`, invalidated on reservation change.

**Testing:**
- `Unit: divisible parent booked → child reported unavailable (and vice-versa)`
- `Integration: availability reflects new reservation immediately after cache invalidation`
- `Integration: find_available_spaces filters by min_capacity and setup style`

#### 3.3 — Booking lifecycle & transactional space assignment

**What:** Services to create/confirm/cancel bookings and atomically assign spaces with conflict checks.

**Design:**
- State machine: `tentative → confirmed → contracted → in_progress → completed`; `→ cancelled` from any
  pre-completed state. Illegal transitions raise `409 Conflict`.
- `create_booking` opens a transaction, inserts booking+events+event_spaces; the exclusion constraint
  is the final guard — on `IntegrityError` return `409` with the conflicting space/time.
- Endpoints: `POST /bookings`, `PATCH /bookings/{id}` (status transitions), `POST /bookings/{id}/cancel`,
  nested `/bookings/{id}/events` and `/events/{id}/spaces`.
- Every state change emits a domain event (`booking.created`, `booking.confirmed`, …) to the event bus
  (Phase 7) and writes an `audit_log` row.

**Testing:**
- `Unit: state machine — confirmed→tentative rejected; tentative→cancelled allowed`
- `Integration: concurrent create of two overlapping bookings → exactly one 201, one 409`
- `Integration: cancel booking → all its event_spaces set cancelled, room freed`
- `E2E: create booking with two events in two rooms → GET returns full nested structure`

---

## Phase 4: CRM, Leads & Calendar Sync

### Purpose
Capture demand and sync availability outward. Adds accounts/contacts/leads (with GDPR consent fields)
and bi-directional calendar integration (iCal export, CalDAV, Google, Microsoft). After this phase
inbound enquiries are tracked and venue calendars sync with external clients — reducing booking friction.

### Tasks

#### 4.1 — Accounts, contacts, leads + GDPR consent

**What:** `accounts`, `contacts`, `leads` tables with consent/retention fields and CRUD routers.

**Design:** Per Suggestion 1 CRM schema. Key GDPR columns on `contacts`: `consent_status`
(granted/denied/pending/withdrawn), `consent_date`, `data_retention_until`. `leads` includes
`ai_score NUMERIC(5,2)` and `ai_score_reasons JSONB` (populated in Phase 6), `status`
(new→contacted→qualified→proposal_sent→won/lost), and `converted_booking_id`.
- `POST /leads` is **public** (booking-portal/website form) — rate-limited, captcha-gated, scoped to a
  venue via a public org/venue token. All other CRM routes require auth.
- Lead→booking conversion endpoint `POST /leads/{id}/convert` creates account/contact/booking.

**Testing:**
- `Integration: public lead submission → lead row, status=new, 201; over rate limit → 429`
- `Integration: convert lead → booking created, lead.status=won, converted_booking_id set`
- `Unit: ContactCreate without consent_status defaults to 'pending'`

#### 4.2 — iCalendar export & free/busy feed (RFC 5545)

**What:** Read-only `.ics` feeds per space and per venue.

**Design:**
- `GET /spaces/{id}/calendar.ics?token=` → `VCALENDAR` with one `VEVENT` per active reservation
  (`UID`=`event_spaces.id@evm`, `DTSTART`/`DTEND`, `SUMMARY`=booking name, `STATUS`). Token-authenticated
  feed URL (no session) so external calendar apps can subscribe.
- `GET /venues/{id}/freebusy.ics` → `VFREEBUSY` aggregating all spaces.
- Built with the `icalendar` library; `DTSTAMP`, `PRODID`, time-zone-aware per venue timezone.

**Testing:**
- `Unit: reservation → VEVENT with correct UID/DTSTART/DTEND, parseable by icalendar round-trip`
- `Integration: GET calendar.ics → valid RFC 5545 text/calendar; invalid token → 401`
- `Fixture: golden .ics output matches committed fixture for a known dataset`

#### 4.3 — CalDAV / Google / Microsoft bi-directional sync

**What:** `calendar_syncs` + `oauth_tokens` tables and a sync worker.

**Design:** Per Suggestion 1 integration tables; `oauth_tokens.access_token_enc/refresh_token_enc`
encrypted with Fernet (`EVM_ENCRYPTION_KEY`). `calendar_syncs` holds per-space provider config and a
`sync_token` (provider delta token).
- ARQ worker `sync_calendar(space_id)`: outbound pushes EVM reservations as events; inbound pulls
  external events and creates `tentative` holds. Conflict resolution: external busy blocks block EVM
  availability; EVM is authoritative for confirmed bookings.
- Providers behind an interface `CalendarProvider.push/pull/refresh_token`. Implementations: Google
  Calendar API, Microsoft Graph (`getSchedule`/events), generic CalDAV (`caldav` lib).

**Testing:**
- `Integration (mocked Google API): pull external event → tentative hold created`
- `Integration (mocked): expired access token → auto-refresh via refresh_token, retry succeeds`
- `Unit: Fernet encrypt/decrypt round-trip on token storage`
- `Integration (mocked): EVM confirmed booking pushed → provider create-event called with correct iCal UID`

---

## Phase 5: Contracts, Invoicing & Payments

### Purpose
Turn confirmed bookings into money: contracts (with e-signature lifecycle), invoices with line items
and tax/service charges, and Stripe-delegated payments. After this phase the platform handles the full
commercial workflow while keeping PCI DSS scope minimal (no card data stored).

### Tasks

#### 5.1 — Contracts & documents

**What:** `contracts`, `documents` tables and a contract lifecycle.

**Design:** Per Suggestion 1. Contract status: `draft→sent→viewed→signed→countersigned→expired/voided`.
`content_html` rendered from a template; `signer_ip INET`, `signed_at`, `signature_data` for the
e-sign audit trail. `documents` stores file metadata + object-storage URL (`doc_type`:
contract/beo/proposal/floor_plan/invoice). Files go to S3-compatible storage (presigned upload URLs).
- Endpoints: `POST /bookings/{id}/contracts`, `POST /contracts/{id}/send`,
  public `GET /sign/{token}` + `POST /sign/{token}` (signer-facing, token-auth).

**Testing:**
- `Integration: send contract → status=sent, sent_at set, signing email enqueued`
- `Integration: sign via token → status=signed, signer_ip + signed_at recorded`
- `Unit: sign an expired contract → 409`

#### 5.2 — Invoicing engine

**What:** `invoices`, `invoice_line_items` with tax/service-charge/discount math.

**Design:** Per Suggestion 1. Totals computed server-side and stored: `subtotal = Σ(line totals)`,
`tax_amount = subtotal_taxable * tax_rate`, `service_charge = subtotal * service_charge_rate`,
`total = subtotal + tax_amount + service_charge - discount_amount`. All money `NUMERIC`; currency ISO
4217. Line categories: space_rental, catering, av_equipment, service_fee, other. An invoice can be
auto-drafted from a booking's space/catering/equipment selections (`POST /bookings/{id}/invoices:draft`).
- Status: draft→sent→viewed→partially_paid→paid→overdue→void→refunded.

**Testing:**
- `Unit: invoice with mixed taxable/non-taxable lines → correct subtotal, tax, total`
- `Unit: discount > subtotal → ValidationError`
- `Integration: auto-draft from booking → line items for each booked space/equipment`

#### 5.3 — Stripe payments & webhooks (PCI DSS)

**What:** `payments` table, Stripe PaymentIntent creation, and verified webhook ingestion.

**Design:** Per Suggestion 1 — only `stripe_payment_intent_id`/`stripe_charge_id` stored, never card
data (PCI scope reduction). `POST /invoices/{id}/pay` creates a PaymentIntent, returns client secret.
Webhook `POST /webhooks/stripe`: verify `Stripe-Signature` (HMAC) against `stripe_webhook_secret`;
on `payment_intent.succeeded`/`charge.refunded`, update payment + invoice status and emit domain event.
Idempotency via Redis on Stripe event id.
- Refund endpoint `POST /payments/{id}/refund`.

**Testing:**
- `Integration (mocked Stripe): create PaymentIntent → payment row pending, client_secret returned`
- `Integration: webhook with invalid signature → 400, no state change`
- `Integration: webhook payment_intent.succeeded → payment succeeded, invoice paid/partially_paid`
- `Integration: duplicate webhook event id → idempotent (single state change)`

---

## Phase 6: AI-Native Layer — Lead Scoring, BEO Generation, Dynamic Pricing

### Purpose
Deliver the differentiating AI features that justify the project's existence. Adds catering/menus and
equipment (BEO inputs), an LLM provider abstraction, and three AI capabilities: lead qualification,
natural-language → structured BEO, and dynamic pricing. After this phase the platform does work that
incumbents charge enterprise rates for.

### Tasks

#### 6.1 — Catering, menus & equipment schema

**What:** `menu_categories`, `menu_items`, `menu_packages`, `menu_package_items`,
`equipment_categories`, `equipment`, `event_equipment` (BEO ingredients).

**Design:** Per Suggestion 1. `menu_items.allergens TEXT[]`, `dietary_tags TEXT[]`; `equipment`
tracks `quantity_total` and rental rates; `event_equipment` reserves quantities against events.
Optional exclusion/availability check on equipment quantity per time window.

**Testing:**
- `Integration: reserve more equipment than quantity_total → 409 over-allocation`
- `Integration: CRUD menu item with dietary_tags → GIN query by tag returns it`

#### 6.2 — LLM provider abstraction + prompt-cached client

**What:** `integrations/llm/` with a provider-neutral interface and structured-output helper.

**Design:**
```python
class LLMClient(Protocol):
    async def complete_structured(self, *, system: str, user: str,
                                  schema: type[BaseModel], cache_prefix: str | None) -> BaseModel: ...
```
- Anthropic implementation uses tool/JSON mode + **prompt caching** on the long system prompt
  (`cache_prefix`); OpenAI-compatible implementation mirrors it. Selected via `EVM_LLM_PROVIDER`.
- All AI calls run in ARQ workers (async, retried), never block the request path. Token usage logged.

**Testing:**
- `Unit (mocked SDK): complete_structured returns validated Pydantic instance; malformed JSON → retry then raise`
- `Unit: provider switch via config returns the correct implementation`

#### 6.3 — AI lead qualification

**What:** Score inbound leads 0–100 with reasons, write back to `leads.ai_score`/`ai_score_reasons`.

**Design:** On `lead.created`, enqueue `score_lead`. Prompt feeds lead fields + venue context +
availability check on `preferred_date`. Output schema:
```python
class LeadScore(BaseModel):
    score: float                    # 0..100
    reasons: list[str]              # e.g. ["available_date","high_budget_signal","repeat_account"]
    suggested_alternative_dates: list[date]
    recommended_action: Literal["fast_track","standard","decline"]
```
System prompt instructs scoring on date availability, budget vs space rates, guest count vs capacity,
and repeat-account signal. Result persisted; high scores notify the assigned user.

**Testing:**
- `Unit (mocked LLM): score_lead persists ai_score and reasons on the lead`
- `Integration: lead with unavailable preferred_date → suggested_alternative_dates populated`
- `Unit: score out of 0..100 → validation rejects, task fails and retries`

#### 6.4 — Natural-language → BEO generation (APEX ESG)

**What:** Generate a structured BEO from a free-text brief; store as first-class entities.

**Design:** `beos`, `beo_meal_services`, `beo_menu_items` per Suggestion 1, fields aligned to APEX
Event Specifications Guide. `POST /events/{id}/beo:generate` {brief: str} enqueues generation.
Output schema (validated, then mapped to rows):
```python
class GeneratedBEO(BaseModel):
    event_name: str; guest_count: int; guaranteed_count: int | None
    setup_style: str; setup_notes: str | None; av_requirements: list[str]
    meal_services: list[MealService]   # service_type, time, menu items w/ dietary tags
    special_requests: str | None
```
The prompt is grounded with the org's actual menu_items/equipment so the model selects real catalogue
items. BEO is versioned (`version` increments), status draft→reviewed→approved→distributed. PDF render
of the approved BEO stored as a document.

**Testing:**
- `Unit (mocked LLM): brief "200-guest plated wedding, open bar" → GeneratedBEO with meal_services`
- `Integration: generated menu items resolve to real menu_item_id where names match catalogue`
- `Integration: regenerate → version increments, prior version retained`
- `Fixture: golden brief → expected structured BEO shape (schema-level assertions)`

#### 6.5 — Dynamic pricing engine

**What:** `pricing_rules` table + a deterministic engine, optionally advised by AI demand signals.

**Design:** Per Suggestion 1 — declarative rules (seasonal, day_of_week, advance_booking, demand,
time_of_day) with `multiplier`/`flat_adjustment`, validity windows, and `priority`. Engine evaluates
applicable rules for a (space, date) and stacks them deterministically by priority. AI layer optionally
proposes a `demand` multiplier from occupancy + lead volume + (future) local-event signals, surfaced as
a suggestion the operator confirms.
- `GET /spaces/{id}/quote?date=&guests=` → base rate + applied rules + final price + breakdown.

**Testing:**
- `Unit: weekend + peak-season rules stack by priority → correct multiplier`
- `Unit: expired rule (outside valid_from/to) not applied`
- `Integration: quote endpoint returns itemised breakdown of applied rules`

---

## Phase 7: Domain Events, Webhooks (AsyncAPI) & Reporting

### Purpose
Make the platform integrable and observable. Adds a domain-event bus, outbound webhooks documented with
AsyncAPI 3.0 (a differentiator per standards.md), the immutable audit log, and reporting/analytics.
After this phase third parties can subscribe to events and operators get dashboards.

### Tasks

#### 7.1 — Domain event bus & audit log

**What:** Internal event emission + `audit_log` table.

**Design:** `events/bus.py` — every state-changing service publishes a typed `DomainEvent(name, org_id,
entity_type, entity_id, payload, actor)`. Bus writes an `audit_log` row (per Suggestion 1) and enqueues
webhook delivery. Event names mirror Suggestion 2's catalogue (`booking.created`, `payment.received`,
`beo.approved`, …). `audit_log.changes JSONB` records old/new field values.

**Testing:**
- `Integration: confirming a booking writes one audit_log row with before/after status`
- `Unit: DomainEvent serialises to the AsyncAPI-documented payload shape`

#### 7.2 — Outbound webhooks (AsyncAPI 3.0)

**What:** Subscriptions + signed delivery worker + `spec/asyncapi.yaml`.

**Design:** `webhook_endpoints` (url, secret, subscribed event names, active). Worker `deliver_webhook`
POSTs JSON with `X-EVM-Signature` (HMAC-SHA256, mirroring HoneyBook/Stripe patterns), retries with
exponential backoff, records delivery attempts. `spec/asyncapi.yaml` documents every channel/event
payload; `make asyncapi` validates it. Endpoint mgmt API under `/webhook-endpoints`.

**Testing:**
- `Integration: subscribed event → endpoint receives signed POST; signature verifies`
- `Integration: endpoint 500 → retried with backoff, attempts recorded; permanent fail → marked failed`
- `Contract: asyncapi.yaml validates against AsyncAPI 3.0 schema`

#### 7.3 — Reporting & analytics

**What:** Aggregate reporting endpoints + materialised summary tables.

**Design:** Read-optimised aggregates (revenue by venue/month/event_type, occupancy %, lead
conversion rate, average response time). Implemented as SQL views / periodically refreshed materialised
views. Endpoints `GET /reports/revenue`, `/reports/occupancy`, `/reports/leads`, with date-range and
venue filters. CSV export for each.

**Testing:**
- `Integration: revenue report sums paid invoices by month correctly against seeded data`
- `Integration: occupancy report computes booked-hours / available-hours per space`
- `Integration: CSV export has correct header + rows`

---

## Phase 8: Frontend — Operator Dashboard & Self-Serve Portal

### Purpose
Deliver the human-facing surfaces: an operator dashboard (calendar, bookings, CRM, BEOs, invoices,
reports) and a public self-serve booking portal. Consumes the OpenAPI spec via a generated typed client.

### Tasks

#### 8.1 — App shell, auth, generated API client

**What:** Next.js app with auth flow and `openapi-typescript`-generated client.

**Design:** App Router; login/OIDC pages hitting Phase 1 auth; tokens in httpOnly cookies via a
route handler proxy. `lib/api/` generated from `spec/openapi.json` (CI regenerates on spec change).
shadcn/ui + Tailwind; role-aware navigation.

**Testing:**
- `E2E (Playwright): login → dashboard; logout → redirected to login`
- `Vitest: API client typed wrapper handles 401 by refreshing token`

#### 8.2 — Calendar, bookings & CRM views

**What:** Availability calendar, booking create/edit wizard, CRM pipeline board.

**Design:** Calendar (week/month) over `/venues/{id}/availability`; drag-to-create booking calling the
Phase 3 conflict-checked create; lead pipeline kanban over `/leads`. Server Components for lists,
client components for interactive calendar.

**Testing:**
- `E2E: create booking via wizard → appears on calendar; overlapping attempt shows conflict error`
- `E2E: move lead card new→qualified → status persisted`

#### 8.3 — Self-serve booking portal

**What:** Public portal: browse spaces, check availability, submit enquiry, pay deposit.

**Design:** Public routes under `/book/{orgSlug}`; calls public availability + public lead endpoints
(Phase 4) and Stripe deposit (Phase 5). No auth; rate-limited; GDPR consent checkbox on submission.

**Testing:**
- `E2E: visitor selects space + date → sees availability → submits enquiry → lead created`
- `E2E: deposit payment via Stripe test card → payment succeeded webhook updates booking`

---

## Phase 9: MCP Server, Sustainability, Compliance & Hardening

### Purpose
Complete the AI-native vision and meet compliance obligations. Adds the MCP server (AI agents operate
the venue), ISO 20121 sustainability tracking, GDPR/CCPA data-subject workflows, and a security pass
against the OWASP API Top 10.

### Tasks

#### 9.1 — MCP server

**What:** An MCP server exposing venue tools to AI agents (Claude, etc.).

**Design:** `mcp/server.py` using the MCP Python SDK. Tools: `check_availability`, `find_spaces`,
`create_booking`, `generate_beo`, `qualify_lead`, `quote_price`. Each tool authenticates via a scoped
API token mapped to an org+role; calls the same domain services (no logic duplication). Read tools for
viewer scope, write tools require manager.

**Testing:**
- `Integration: MCP check_availability tool returns same result as REST availability endpoint`
- `Integration: create_booking tool with viewer-scoped token → permission denied`

#### 9.2 — Sustainability tracking (ISO 20121:2024)

**What:** `sustainability_metrics` table + capture/report endpoints.

**Design:** Per Suggestion 1 — metric_type (carbon_kg, waste_kg, recycled_kg, water_litres, energy_kwh,
food_waste_kg, attendee_transport_km), value, measurement_method. `GET /reports/sustainability` returns
per-event and per-org aggregates incl. carbon-per-guest, aligned to ISO 20121 categories.

**Testing:**
- `Integration: record metrics for an event → sustainability report aggregates carbon_per_guest`
- `Unit: unknown metric_type → ValidationError`

#### 9.3 — GDPR/CCPA data-subject workflows

**What:** Consent management, data export, and right-to-erasure.

**Design:** `GET /contacts/{id}/export` (portability: JSON of all data linked to a contact);
`POST /contacts/{id}/erase` (anonymises PII fields, retains financial records per legal-hold, logs the
deletion in audit_log); consent endpoints update `consent_status`/`consent_date`. "Do Not Sell" flag
for CCPA.

**Testing:**
- `Integration: erase contact → PII nulled/anonymised, audit_log records deletion, invoices retained`
- `Integration: export returns all linked bookings/contracts/communications as JSON`

#### 9.4 — Security hardening (OWASP API Top 10)

**What:** Security review pass + automated checks.

**Design:** Verify BOLA defence (RLS coverage on every tenant table — automated test enumerates tables),
rate limiting on auth + public endpoints (API#2/#4), input validation everywhere (Pydantic), no
deprecated/unauthenticated routes (API#9), security headers, dependency scanning in CI, secrets only via
env, encrypted tokens at rest. Document the threat model.

**Testing:**
- `Integration: cross-tenant access attempt on every resource type → 404/403 (BOLA suite)`
- `Integration: exceed auth rate limit → 429`
- `CI: ruff + mypy + dependency audit pass; no table lacks an RLS policy (introspection test)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Auth        ─── required by everything
    │
Phase 2: Venues, Spaces & Schema Library   ─── requires Phase 1
    │
Phase 3: Bookings, Events & Availability    ─── requires Phase 2   (CORE VALUE)
    │
    ├── Phase 4: CRM, Leads & Calendar Sync       ─── requires Phase 3 ┐ can parallel
    └── Phase 5: Contracts, Invoicing & Payments  ─── requires Phase 3 ┘
              │
Phase 6: AI Layer (Leads / BEO / Pricing)   ─── requires Phase 4 (leads, menus) + Phase 5 (pricing↔invoicing)
    │
Phase 7: Events, Webhooks & Reporting       ─── requires Phase 5 (financial events) + Phase 6
    │
Phase 8: Frontend (Dashboard + Portal)      ─── requires Phases 3–7 APIs (build incrementally alongside)
    │
Phase 9: MCP / Sustainability / Compliance  ─── requires Phase 6 (AI services) + Phase 7 (events)
```

**Parallelism opportunities:**
- **Phases 4 and 5** can be built concurrently once Phase 3 lands (independent surfaces over the same booking core).
- **Phase 8 (frontend)** can begin as soon as each backend phase publishes its slice of the OpenAPI spec — front and back can progress in lockstep rather than strictly after Phase 7.
- Within Phase 6, **6.3 (lead scoring), 6.4 (BEO), 6.5 (pricing)** are independent once 6.2 (LLM client) exists.
- Within Phase 9, **9.1 (MCP), 9.2 (sustainability), 9.3 (GDPR)** are independent.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`); frontend phases also pass Vitest + Playwright.
3. Linting and formatting pass (`ruff`; `eslint`/`prettier` for frontend).
4. Type checking passes (`mypy`; `tsc --noEmit` for frontend).
5. `docker compose build` succeeds for all affected services.
6. The phase's feature works end-to-end (demonstrated via an integration or E2E test).
7. New configuration options are documented in `.env.example` and the README.
8. New/changed API endpoints appear in the regenerated `spec/openapi.json`; new events appear in `spec/asyncapi.yaml`; CI confirms no spec drift.
9. Alembic migration(s) created, reversible, and applied cleanly on a fresh database.
10. New tenant-scoped tables have RLS enabled and a policy (verified by the introspection test from 9.4).
11. No raw payment-card data is stored anywhere (PCI DSS); secrets only via environment; tokens encrypted at rest.
```
