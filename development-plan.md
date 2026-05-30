# Parking Management System — Phased Development Plan

> Project: 465-parking-management-system · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four data-model
suggestions. The canonical schema is **data-model-suggestion-1** (normalized relational PostgreSQL with PostGIS
and row-level-security multi-tenancy), augmented with the **data-model-suggestion-4** insight that high-volume
time-series tables (`lpr_reads`, `occupancy_readings`, `audit_log`) become TimescaleDB hypertables — since
TimescaleDB is a PostgreSQL extension, this stays inside one database engine with no polyglot operational cost.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The product is AI-native (dynamic pricing, occupancy forecasting, LPR confidence routing, citation auto-adjudication, NL config). Python has the richest ML/LLM ecosystem (scikit-learn, statsmodels, OpenAI/Anthropic SDKs) while remaining fully capable for the transactional API tier. |
| API framework | FastAPI 0.115+ | Native OpenAPI 3.1 generation (required by `standards.md`), async I/O for gate/LPR latency, Pydantic v2 validation, dependency-injection for per-request tenant context and auth. |
| Web/admin frontend | Next.js 15 (App Router) + TypeScript + shadcn/ui + MapLibre GL | Operator dashboard, occupancy map, and driver self-service portal. MapLibre renders GeoJSON facility/zone/space layers (RFC 7946). Must meet WCAG 2.1 AA per `standards.md`. |
| Mobile enforcement app | React Native (Expo) + SQLite (offline store) | One codebase for iOS+Android officer app; offline-first citation issuance required by `research.md` §4 (underground/low-signal). Local SQLite queue syncs on reconnect. |
| Database | PostgreSQL 16 + PostGIS 3.4 + TimescaleDB 2.x | 3NF relational core for payments/permits/citations (ACID, FK integrity); PostGIS for facility/zone geometry and geofenced permits; TimescaleDB hypertables for LPR/occupancy/audit volume. RLS for shared-schema multi-tenancy. |
| ORM / data access | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM; Alembic for versioned migrations. GeoAlchemy2 for PostGIS types. |
| Cache / real-time | Redis 7 | Live occupancy counters, session token store, rate-limit counters, pub/sub for occupancy → signage/app fan-out. |
| Task queue | Celery 5 + Redis broker | Async webhook delivery, LPR match processing, dynamic-pricing recompute, forecast batch jobs, notification dispatch. |
| Payment integration | Stripe (primary), pluggable gateway interface | PCI-DSS v4.0 scope reduction via tokenisation — no PAN ever touches our servers (Stripe Elements / PaymentIntents). Gateway abstracted behind an interface so Adyen/Square can be added. |
| Auth | OAuth 2.0 (RFC 6749) + JWT (RFC 7519) + OIDC for SSO | Operator/officer login via JWT bearer; SSO (OIDC) for campus/municipal identity providers per `standards.md`; API keys for machine-to-machine (PARCS, LPR vendors). |
| LLM provider | Anthropic Claude via SDK, pluggable | NL permit-rule config, citation auto-adjudication reasoning, chatbot. Abstracted behind `LLMClient` so provider is swappable. |
| ML libraries | statsmodels + scikit-learn (forecasting), pandas | Occupancy forecasting and dynamic pricing models; no GPU dependency for MVP-scale operators. |
| Object storage | S3-compatible (MinIO self-hosted / AWS S3) | Citation photos, LPR images, appeal documents — only URLs stored in DB. |
| Containerisation | Docker + docker-compose | Self-hosted deployment target per `README.md`; compose bundles Postgres+TimescaleDB, Redis, API, workers, frontend, MinIO. |
| Testing | pytest + pytest-asyncio + testcontainers; Vitest + Playwright (frontend) | Real Postgres via testcontainers for integration tests; Playwright for portal e2e. |
| Code quality | ruff (lint+format), mypy (types), Biome (TS) | Enforced in CI. |
| Package manager | uv (Python), pnpm (JS) | Fast, reproducible installs. |
| API response convention | JSON:API-style envelopes + RFC 8288 Link headers for pagination | Consistent pagination/filtering/error shape across endpoints per `standards.md`. |
| Interop exports | APDS v3 / ISO TS 5206-1 mapping layer; OCPI 2.3 for EV | Differentiator per `standards.md` notes — native APDS export and OCPI parking module. |

### Project Structure

```
parking-management-system/
├── pyproject.toml
├── uv.lock
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── .env.example
├── alembic.ini
├── migrations/                      # Alembic migration scripts
│   └── versions/
├── src/
│   └── pms/
│       ├── main.py                  # FastAPI app factory, router mount, middleware
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py           # async engine, session factory
│       │   ├── base.py              # Declarative base, common mixins (TenantMixin, TimestampMixin)
│       │   └── rls.py               # set_config('app.current_tenant', ...) helper
│       ├── models/                  # SQLAlchemy ORM models (one module per domain)
│       │   ├── tenant.py  facility.py  customer.py  permit.py
│       │   ├── session.py  rate.py  payment.py  citation.py
│       │   ├── lpr.py  occupancy.py  ev.py  hardware.py  audit.py
│       ├── schemas/                 # Pydantic request/response models
│       ├── api/
│       │   ├── deps.py              # get_current_user, get_tenant, require_role
│       │   └── v1/
│       │       ├── facilities.py  spaces.py  permits.py  sessions.py
│       │       ├── reservations.py  rates.py  payments.py  citations.py
│       │       ├── appeals.py  enforcement.py  lpr.py  occupancy.py
│       │       ├── ev.py  hardware.py  webhooks.py  exports.py
│       ├── services/                # Business logic (no HTTP concerns)
│       │   ├── pricing/             # rate resolution + dynamic pricing
│       │   ├── permits/             # eligibility, waitlist, issuance
│       │   ├── enforcement/         # citation lifecycle, adjudication
│       │   ├── payments/            # gateway-agnostic charge/refund
│       │   ├── lpr/                 # vendor-agnostic ingest + matching
│       │   ├── occupancy/           # snapshot computation, forecasting
│       │   └── ai/                  # LLMClient, NL config, adjudication, chatbot
│       ├── integrations/
│       │   ├── gateways/            # stripe.py + base.PaymentGateway
│       │   ├── lpr_vendors/         # genetec.py, vigilant.py + base.LPRAdapter
│       │   ├── parcs/               # gate control adapters + base.PARCSAdapter
│       │   ├── ocpi/                # OCPI 2.3 client/server
│       │   └── apds/                # APDS/ISO 5206 export mappers
│       ├── workers/                 # Celery tasks
│       ├── webhooks/                # outbound webhook signing + delivery
│       └── security/                # JWT, OAuth, OIDC, API keys, password hashing
├── tests/
│   ├── conftest.py                  # testcontainers Postgres+Redis fixtures
│   ├── unit/  integration/  e2e/  fixtures/
├── frontend/                        # Next.js operator + driver portal
│   ├── app/  components/  lib/  tests/
└── mobile/                          # React Native (Expo) enforcement app
    ├── app/  src/offline/  src/sync/  tests/
```

---

## Phase 1: Foundation & Multi-Tenancy

### Purpose
Establish the project skeleton, configuration, database connectivity with PostGIS+TimescaleDB, the multi-tenant
RLS model, and the auth backbone. After this phase a developer can run `docker-compose up`, authenticate as a
tenant user, and have every subsequent request automatically scoped to its tenant.

### Tasks

#### 1.1 — Project scaffolding & config
**What**: Bootstrap the repo, dependency management, Docker, and environment configuration.

**Design**:
- `pyproject.toml` with uv; pin FastAPI, SQLAlchemy[asyncio], asyncpg, alembic, geoalchemy2, redis, celery, pydantic-settings, python-jose, passlib[bcrypt], httpx.
- `config.py`:
```python
class Settings(BaseSettings):
    database_url: str          # postgresql+asyncpg://...
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str
    jwt_alg: str = "HS256"
    access_token_ttl_min: int = 30
    refresh_token_ttl_days: int = 30
    s3_endpoint: str | None = None
    stripe_secret_key: str | None = None
    llm_provider: str = "anthropic"
    llm_api_key: str | None = None
    model_config = SettingsConfigDict(env_file=".env")
```
- `docker-compose.yml`: services `db` (timescale/timescaledb-ha:pg16 with PostGIS), `redis`, `api`, `worker`, `minio`, `frontend`.
- `Dockerfile.api` multi-stage (uv install → slim runtime).

**Testing**:
- Unit: `Settings` loads from env with defaults → correct types; missing `jwt_secret` → ValidationError naming the field.
- Integration: `docker-compose up db` then connection succeeds and `CREATE EXTENSION postgis, timescaledb` run without error.

#### 1.2 — Database base, mixins & migration tooling
**What**: SQLAlchemy declarative base, common mixins, async session factory, Alembic wired to autogenerate.

**Design**:
```python
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class TenantMixin:
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.tenant_id"), index=True)
```
- `db/session.py`: `async_sessionmaker`; a FastAPI dependency yields a session and on entry runs
  `SELECT set_config('app.current_tenant', :tid, true)` for RLS.
- Alembic `env.py` configured for async engine and `target_metadata = Base.metadata`.

**Testing**:
- Integration (testcontainers): `alembic upgrade head` then `downgrade base` round-trips cleanly.
- Unit: mixin columns present on a sample model via mapper inspection.

#### 1.3 — Tenant & User models + RLS policies
**What**: Implement `tenants` and `users` tables (data-model-1) and enable RLS tenant isolation.

**Design**:
- ORM models mirror the `tenants` and `users` DDL (roles: `super_admin, tenant_admin, operator, enforcement_officer, customer_service, auditor`).
- Migration emits `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` and `CREATE POLICY tenant_isolation USING (tenant_id = current_setting('app.current_tenant')::uuid)` on every tenant-scoped table (codified in a reusable migration helper `apply_rls(table)`).
- `super_admin` rows bypass via a `BYPASSRLS` connection role used only by the admin service.

**Testing**:
- Integration: insert two tenants + users; with `app.current_tenant=A`, `SELECT * FROM users` returns only A's users.
- Integration: cross-tenant `UPDATE` of B's user while scoped to A affects 0 rows.
- Unit: role CHECK rejects an invalid role value.

#### 1.4 — Authentication & authorization
**What**: OAuth2 password + refresh flow issuing JWTs, role-based dependencies, API-key auth for machines.

**Design**:
- `security/jwt.py`: `create_access_token(sub, tenant_id, role)`, `decode(token)`; claims `{sub, tid, role, exp, iat}` per RFC 7519.
- Endpoints: `POST /v1/auth/token` (password grant → access+refresh), `POST /v1/auth/refresh`.
- `api/deps.py`: `get_current_user`, `get_tenant`, `require_role(*roles)` → 403 on mismatch.
- API keys: `api_keys` table (hashed key, tenant_id, scopes[]); header `X-API-Key` resolved by `get_api_principal`.
- Passwords hashed with bcrypt via passlib.

**Testing**:
- Unit: valid creds → token whose decoded claims match user; expired token → 401.
- Integration: `require_role("tenant_admin")` on operator token → 403.
- Integration: valid API key with scope `lpr:write` reaches an LPR endpoint; missing scope → 403.

---

## Phase 2: Facility, Space Inventory & Occupancy Core

### Purpose
Model the physical world — facilities, levels, zones, spaces, lanes — with PostGIS geometry, and the
real-time occupancy state that everything else (pricing, enforcement, signage) depends on. After this phase the
system can answer "where are the available accessible spaces in Zone B right now?".

### Tasks

#### 2.1 — Facility hierarchy CRUD
**What**: REST CRUD for facilities, levels, zones, spaces, lanes with geometry.

**Design**:
- ORM models per data-model-1 (`facilities.location GEOGRAPHY(POINT)`, `boundary GEOGRAPHY(POLYGON)`; `spaces.space_type` enum incl. `accessible, ev_charging, reserved`).
- Endpoints (all tenant-scoped, paginated with `?page`/`?limit`, Link headers):
  - `GET/POST /v1/facilities`, `GET/PATCH/DELETE /v1/facilities/{id}`
  - `…/facilities/{id}/levels`, `/zones`, `/spaces`, `/lanes` nested collections
- GeoJSON in/out: request accepts `location` as GeoJSON Point; responses emit Feature/FeatureCollection (RFC 7946) for map consumption.
- Spatial query: `GET /v1/facilities?near=lng,lat&radius_m=500` uses `ST_DWithin`.

**Testing**:
- Integration: create facility with GeoJSON polygon → stored, `ST_Area` non-zero; retrieved geometry round-trips.
- Integration: `?near=` returns only facilities within radius, ordered by distance.
- Unit: invalid space_type → 422; duplicate `(facility_id, space_number)` → 409.

#### 2.2 — Occupancy state & snapshots
**What**: Maintain live occupancy in Redis and periodic `occupancy_snapshots`; expose dashboards.

**Design**:
- Redis keys `occ:{facility_id}:{zone_id}` holding occupied count; `space:{space_id}` holding status.
- `services/occupancy/state.py`:
  - `set_space_status(space_id, status)` → updates Redis + publishes `occupancy.changed` on Redis pub/sub.
  - `snapshot(facility_id)` → writes `occupancy_snapshots` row (generated `available_spaces`, `occupancy_pct`).
- Celery beat task every 60s snapshots each active facility.
- `GET /v1/facilities/{id}/occupancy` → `{total, occupied, available, occupancy_pct, by_zone[]}`.
- WebSocket `GET /v1/ws/occupancy/{facility_id}` streams changes from pub/sub.

**Testing**:
- Integration: mark 3 of 10 spaces occupied → endpoint returns occupancy_pct 30.0.
- Integration: snapshot task writes a row with correct generated columns.
- Unit: occupancy of facility with 0 spaces → 0.0 (no divide-by-zero).
- Integration: WebSocket client receives a message after a `set_space_status` call.

#### 2.3 — Occupancy readings ingestion (TimescaleDB)
**What**: High-volume sensor reading ingest into a TimescaleDB hypertable feeding live state.

**Design**:
- `occupancy_readings` created as hypertable: migration runs `SELECT create_hypertable('occupancy_readings','read_at')` plus a 90-day compression policy.
- `POST /v1/occupancy/readings` (API-key auth, scope `sensor:write`) accepts batched `[{sensor_id, space_id, is_occupied, read_at, battery_level}]`.
- On ingest: upsert space status → call `set_space_status`. Late/duplicate reads (older than current space `updated_at`) are stored but do not regress live state.

**Testing**:
- Integration: batch of 1,000 readings inserts and updates live counts; `is_occupied=false` flips space to available.
- Integration: out-of-order read (older timestamp) is stored but does not change current status.
- Unit: malformed batch (missing `sensor_id`) → 422 with index of offending element.

---

## Phase 3: Permits & Customers

### Purpose
Deliver the first headline MVP capability: customers, vehicles, configurable permit types, the
application→issuance→renewal lifecycle, and waitlists. This is a core differentiator (deep permit configurability
matching T2/AIMS) and a prerequisite for enforcement permit lookups.

### Tasks

#### 3.1 — Customers & vehicles
**What**: CRUD for customers and their vehicles, with plate normalisation.

**Design**:
- Models per data-model-1. `services/vehicles/plate.py:normalize(plate, state, country)` uppercases, strips whitespace/dashes — used everywhere a plate is matched (LPR, citations, sessions).
- Endpoints: `/v1/customers`, `/v1/customers/{id}/vehicles`. Driver self-service uses customer-scoped tokens (Phase 7).

**Testing**:
- Unit: `normalize(" ab-12 cd ", "CA", "US")` → `"AB12CD"`.
- Integration: duplicate `(tenant_id, email)` → 409.
- Integration: vehicle plate index lookup returns the right customer.

#### 3.2 — Permit types & eligibility rules
**What**: Operator-configurable permit types with validity windows, eligible zones/facilities, and limits.

**Design**:
- `permit_types` model per data-model-1 (category enum, `eligible_zones uuid[]`, `valid_days smallint[]`, `valid_start_time/valid_end_time`, `max_active`, `is_virtual`).
- `services/permits/eligibility.py:check(permit_type, customer, vehicle) -> EligibilityResult{eligible: bool, reasons: list[str]}` evaluates `requires_verification`, `max_vehicles`, and `max_active` against current issued count.

**Testing**:
- Unit: permit type with `max_active=100` and 100 active → new issuance ineligible with reason `"quota_full"`.
- Unit: `valid_days={1..5}` rejects a Sunday-only request.
- Integration: create permit type with eligible_zones array; retrieval preserves array.

#### 3.3 — Permit issuance, renewal & waitlist
**What**: Lifecycle endpoints with state machine and oversubscription handling.

**Design**:
- States: `pending → active → expired | revoked | suspended`; `waitlisted → (offered) → active`.
- `POST /v1/permits` → runs eligibility; if type full and `is_waitlisted` → create `permit_waitlist` row at next `position`, return permit `waitlisted`.
- `POST /v1/permits/{id}/renew` → creates new permit with `renewed_from` set, copies vehicles, new `effective_from/to`.
- Celery: nightly task expires permits past `effective_to`; when an active permit of a full type is revoked, offer the next waitlisted customer (`status=offered`, `expires_at=now+72h`, notification).

**Testing**:
- Integration: issue to a full waitlisted type → permit `waitlisted`, waitlist position 1.
- Integration: revoke an active permit → next waitlisted becomes `offered`.
- Integration: renew copies vehicles and links `renewed_from`.
- Unit: expiry task moves only permits with `effective_to < now` to `expired`.

---

## Phase 4: Rating Engine, Sessions & Reservations

### Purpose
Build the pricing engine and the session/reservation lifecycle that produces an amount due. This is the revenue
backbone and the input to payment processing. Dynamic (AI) pricing is layered in Phase 8; here the engine resolves
the correct base/tiered/flat rate deterministically.

### Tasks

#### 4.1 — Rate schedules & resolution engine
**What**: Configurable rate schedules/tiers and a function that prices a session duration.

**Design**:
- `rate_schedules` + `rate_tiers` models per data-model-1 (rate_type `hourly|flat|tiered|daily_max|event|dynamic|validation`; `priority`).
- `services/pricing/engine.py`:
```python
def resolve_rate(tenant_id, facility_id, zone_id, at: datetime) -> RateSchedule
def price_session(schedule, entry: datetime, exit: datetime,
                  adjustments: list[DynamicAdjustment]) -> PriceResult
# PriceResult: {base, tier_breakdown[], dynamic_multiplier, daily_max_applied, total}
```
- Resolution picks highest-`priority` active schedule whose day/time window matches; tiers apply `from_minutes/to_minutes` with `daily_max`/`weekly_max` caps.

**Testing**:
- Unit: 90-minute stay on a tiered schedule (first hour $3, then $2/30min) → $5.
- Unit: stay exceeding `daily_max` caps at the max.
- Unit: two overlapping schedules → higher priority wins.

#### 4.2 — Parking sessions
**What**: Session lifecycle from entry to priced completion.

**Design**:
- `parking_sessions` model (entry/exit method enums, denormalised `license_plate` for LPR-initiated, generated `duration_minutes`).
- `POST /v1/sessions` (entry), `POST /v1/sessions/{id}/exit` → calls `price_session`, sets `amount_due`, transitions `active→completed`. Overstay detection sets `overstay` when `now > expected_exit`.
- Updates occupancy state (Phase 2) on entry/exit.

**Testing**:
- Integration: open then exit a session → `amount_due` matches engine; status `completed`.
- Integration: session past `expected_exit` flagged `overstay` by sweep task.
- Integration: entry increments facility occupancy; exit decrements.

#### 4.3 — Reservations
**What**: Pre-booked single/recurring/event/monthly reservations with confirmation codes.

**Design**:
- `reservations` model (iCal `recurrence_rule`, `confirmation_code`, `qr_code_data`). Availability check counts overlapping confirmed reservations + active sessions against `zone.total_spaces`.
- `POST /v1/reservations` validates no overbooking; generates code + QR; `POST /v1/reservations/{id}/check-in` links to a session.

**Testing**:
- Integration: booking beyond capacity for the window → 409 `no_availability`.
- Unit: confirmation code unique per tenant; QR payload decodes to reservation id.
- Integration: check-in creates a `reserved` session.

---

## Phase 5: Payments (PCI-DSS)

### Purpose
Process payments for sessions, reservations, permits, and citations without bringing card data into PCI scope.
After this phase the platform can take money — the prerequisite for being a real product.

### Tasks

#### 5.1 — Gateway abstraction & Stripe adapter
**What**: Provider-agnostic payment interface with a Stripe implementation; tokenised methods only.

**Design**:
```python
class PaymentGateway(Protocol):
    async def create_intent(self, amount: Decimal, currency: str, customer_ref: str|None,
                            metadata: dict) -> Intent          # returns client_secret, gateway_txn_id
    async def confirm(self, gateway_txn_id: str) -> ChargeResult
    async def refund(self, gateway_txn_id: str, amount: Decimal) -> RefundResult
    async def tokenize_setup(self, customer_ref: str) -> SetupIntent
```
- `payment_transactions` and `payment_methods` per data-model-1 store **only** `payment_token`, `card_last_four`, `card_brand`, `gateway_txn_id` — never PAN (PCI-DSS v4.0 scope reduction). Card capture happens client-side via Stripe Elements.
- `POST /v1/payments/intents` → creates intent for a target (session/reservation/permit/citation); `POST /v1/payments/{id}/confirm`.

**Testing**:
- Integration (Stripe test mode / mocked): create intent → `client_secret` returned, transaction `pending`; confirm → `completed`, `card_last_four` stored, no PAN anywhere in DB (assert by scanning columns).
- Integration: refund a completed charge → `refunded`, refund txn linked.
- Unit: amount ≤ 0 → 422.

#### 5.2 — Stripe webhooks & reconciliation
**What**: Inbound gateway webhooks update transaction state idempotently.

**Design**:
- `POST /v1/webhooks/stripe` verifies signature; maps `payment_intent.succeeded/.payment_failed/charge.refunded` to transaction status. Idempotent on `gateway_txn_id` + event id (dedup table).
- On success for a citation target → marks citation `paid`; for permit → activates pending permit.

**Testing**:
- Integration (mocked): valid signature succeeded event → transaction `completed`; replay same event → no double-apply.
- Integration: invalid signature → 400, no state change.

#### 5.3 — Validations & account balance
**What**: Validation programs/codes and customer account balance debits.

**Design**:
- `validation_programs` + `validation_codes` per data-model-1; redeeming a code reduces session `amount_due` per `discount_type`, decrements `budget_used` (reject if over `budget_limit`).
- `payment_method = account_balance` debits `customers.balance` transactionally.

**Testing**:
- Unit: `free_hours=2` validation on a 3-hour stay charges only the 3rd hour.
- Integration: redeeming beyond `budget_limit` → 409; budget unchanged.
- Integration: balance debit insufficient funds → 402, balance unchanged.

---

## Phase 6: Enforcement, Citations & Appeals

### Purpose
Deliver the enforcement half of the MVP: citation issuance with photo evidence, permit/payment lookup, the
citation lifecycle, appeals, and escalation (boot/tow/collections). This phase produces the API the mobile
officer app (Phase 9) consumes.

### Tasks

#### 6.1 — Citation issuance & lifecycle
**What**: Issue, list, and transition citations with evidence.

**Design**:
- `citations`, `citation_evidence`, `enforcement_actions` models per data-model-1 (violation_type enum, generated `total_due`, status machine `issued→paid|appealed|escalated|dismissed|void`).
- `POST /v1/citations` (officer/operator) → validates plate, computes due_date/escalation_date from tenant config, creates citation.
- `POST /v1/citations/{id}/evidence` → presigned S3 upload, records `citation_evidence` row.
- `GET /v1/lookup/plate/{plate}` → real-time permit + active-session + payment status for officers (the "is this car legal?" call).

**Testing**:
- Integration: issue citation → `total_due = fine + late_fee`; status `issued`.
- Integration: plate lookup returns valid permit when one is active for that plate/zone → officer sees "permitted".
- Unit: invalid violation_type → 422.

#### 6.2 — Appeals workflow
**What**: Customer-submitted appeals with operator review and decision.

**Design**:
- `citation_appeals` model (reason enum, `supporting_docs[]`, status `submitted→under_review→approved|denied`). Approval transitions citation `dismissed`/`appeal_approved` and voids unpaid balance; denial returns to `issued`.
- `POST /v1/citations/{id}/appeals`, `POST /v1/appeals/{id}/decision` (require_role customer_service/tenant_admin).

**Testing**:
- Integration: submit appeal → citation `appealed`; approve → citation `dismissed`, balance voided.
- Integration: deny → citation back to `issued`, balance intact.

#### 6.3 — Escalation actions
**What**: Boot/tow/collections lifecycle with vendor references.

**Design**:
- `enforcement_actions` model (action_type `warning|boot|tow|collections|license_hold`, status machine). Celery task escalates unpaid citations past `escalation_date` to `collections`. Tow/boot create an action row with `vendor_reference`.

**Testing**:
- Integration: citation unpaid past escalation_date → `escalated`, collections action created.
- Integration: boot action `executed_at` then `released_at` transitions to `completed`.

---

## Phase 7: Driver Self-Service Portal & Operator Dashboard (Frontend)

### Purpose
Ship the web UIs: a driver self-service portal (permit management, violation payment, reservations, account) and
an operator dashboard (occupancy map, revenue, enforcement activity). Completes the MVP feature set.

### Tasks

#### 7.1 — Auth, layout & API client
**What**: Next.js app shell with JWT session handling and a typed API client.

**Design**:
- `lib/api.ts` generated from the FastAPI OpenAPI 3.1 schema (openapi-typescript). Auth via httpOnly cookie holding refresh token; access token in memory.
- Role-aware routing: `/portal/*` (driver), `/admin/*` (operator). shadcn/ui components; WCAG 2.1 AA (focus order, labels, contrast).

**Testing**:
- Playwright: unauthenticated `/admin` → redirect to login.
- Vitest: API client attaches bearer; 401 triggers silent refresh then retry.

#### 7.2 — Driver portal
**What**: Permit purchase/renewal, reservation booking, citation payment, vehicle/account management.

**Design**:
- Pages: Permits (list + apply/renew → Phase 3 + Stripe Elements), Reservations (map search via MapLibre → Phase 4), Citations (lookup by plate + pay → Phases 5/6), Account.
- Stripe Elements card capture so PAN never reaches our backend.

**Testing**:
- Playwright (mocked API): pay a citation end-to-end → confirmation shown, citation reads `paid`.
- Playwright: book a reservation → confirmation code + QR rendered.

#### 7.3 — Operator dashboard
**What**: Occupancy map, revenue analytics, enforcement activity, permit/waitlist admin.

**Design**:
- Live occupancy map (MapLibre + GeoJSON, WebSocket updates from Phase 2). Analytics cards: occupancy trend, revenue by zone, citation counts (queries over snapshots/transactions/citations).
- Permit-type config UI and waitlist management.

**Testing**:
- Playwright: occupancy map updates when a WebSocket occupancy event fires.
- Playwright: revenue dashboard renders correct totals from seeded data.

---

## Phase 8: AI-Native Differentiators

### Purpose
Implement the AI capabilities that set this product apart from rule-based incumbents: dynamic pricing, occupancy
forecasting, LPR confidence routing, automated citation adjudication, and natural-language configuration. Each is
additive and gated behind a per-tenant feature flag.

### Tasks

#### 8.1 — Occupancy forecasting
**What**: Predict facility/zone occupancy up to 72h ahead.

**Design**:
- `services/occupancy/forecast.py`: trains a seasonal model (statsmodels SARIMAX or a gradient-boosted regressor on hour-of-week/holiday/weather features) per facility from `occupancy_snapshots` history. Celery nightly retrain; predictions cached in Redis and stored in `occupancy_forecasts(facility_id, horizon_at, predicted_pct, lower, upper, model_version)` (new table via migration).
- `GET /v1/facilities/{id}/forecast?hours=72`.

**Testing**:
- Unit: forecaster on a synthetic weekly-seasonal series predicts the next peak within tolerance.
- Integration: cold facility (no history) → endpoint returns 200 with `confidence: low` and naive baseline, no crash.

#### 8.2 — Dynamic pricing
**What**: Adjust rates from demand signals, writing auditable `dynamic_pricing_adjustments`.

**Design**:
- `services/pricing/dynamic.py`: maps forecast/live occupancy to a `multiplier` via configurable bands (e.g. >85% → 1.5x, <30% → 0.8x), bounded by tenant min/max. Writes `dynamic_pricing_adjustments` with `is_ai_generated=true`, `confidence_score`, `ml_model_version`, `occupancy_threshold`. `price_session` (Phase 4) already consumes active adjustments.
- Operator approval mode: adjustments above a threshold require `approved_by` before taking effect.

**Testing**:
- Unit: 90% occupancy → multiplier within configured surge band, clamped to max.
- Integration: an active adjustment changes `price_session` output; approval-required adjustment does not apply until approved.

#### 8.3 — LPR confidence routing & matching
**What**: Vendor-agnostic LPR ingest that routes low-confidence reads to a review queue.

**Design**:
- `integrations/lpr_vendors/base.py:LPRAdapter` normalises vendor payloads → `LPRRead{plate, state, confidence, image_url, camera_id, direction, read_at}`. `lpr_reads` is a TimescaleDB hypertable.
- `POST /v1/lpr/reads` (API key, scope `lpr:write`). On ingest: normalise plate, match to `vehicles`/active session; if `confidence < tenant.lpr_threshold` (default 0.85) set `requires_review=true`. Review queue `GET /v1/lpr/reads?requires_review=true`; `POST /v1/lpr/reads/{id}/correct` records `corrected_plate`.

**Testing**:
- Integration: read at 0.70 confidence → `requires_review=true`, appears in queue; at 0.95 → auto-matched to vehicle/session.
- Unit: Genetec vs Vigilant sample payloads both normalise to the same `LPRRead`.

#### 8.4 — Automated citation adjudication
**What**: Auto-resolve straightforward appeals/violations by cross-referencing data + LLM reasoning.

**Design**:
- `services/ai/adjudication.py`: gathers citation + LPR scan + permit/payment records, prompts the LLM for a structured decision.
```python
class AdjudicationDecision(BaseModel):
    recommendation: Literal["dismiss", "uphold", "human_review"]
    confidence: float
    rationale: str
```
- Deterministic pre-checks first (valid permit at time/zone → auto-dismiss; confirmed paid session → auto-dismiss). Only ambiguous cases reach the LLM. Decisions with `confidence ≥ tenant.adjudication_threshold` auto-apply; others set appeal `under_review`. Writes `auto_reviewed=true`, `ai_recommendation`, `ai_confidence`.

**Testing**:
- Unit (deterministic): appeal on a citation where vehicle had a valid permit in-zone at issue time → `dismiss` without LLM call.
- Integration (mocked LLM): ambiguous case low confidence → routed to `human_review`, not auto-applied.

#### 8.5 — Natural-language configuration & chatbot
**What**: Operators describe permit rules/rates in plain English; drivers chat for self-service.

**Design**:
- `services/ai/nl_config.py`: LLM converts NL → a proposed `PermitType`/`RateSchedule` payload returned for operator confirmation before persistence (never auto-writes config).
- `services/ai/chatbot.py`: retrieval over the tenant's permits/citations/FAQs; tool-calls into read-only lookup endpoints; escalates to human on low confidence.

**Testing**:
- Integration (mocked LLM): "Staff permits valid Mon–Fri 7am–6pm in Lots A and B, $40/month" → proposed PermitType with correct `valid_days`, times, eligible_facilities, price; requires explicit confirm to save.
- Integration: chatbot answers "how do I pay my ticket?" with a link to the citation flow.

---

## Phase 9: Mobile Enforcement App (Offline-First)

### Purpose
Deliver the officer field app with offline citation issuance and later sync — a hard requirement from
`research.md` §4 for underground/low-signal environments.

### Tasks

#### 9.1 — App shell, auth & plate lookup
**What**: Expo app with login and online plate lookup.

**Design**:
- Auth against `/v1/auth/token`; plate lookup calls `/v1/lookup/plate/{plate}` showing permit/session/payment status. Camera capture for plate + evidence photos.

**Testing**:
- Detox/Playwright-RN: login → lookup a permitted plate shows green "permitted" state.

#### 9.2 — Offline citation issuance & sync
**What**: Issue citations offline into local SQLite; sync queue on reconnect.

**Design**:
- Local SQLite mirrors a citation draft (client UUID, plate, violation_type, photos as local files). `src/sync/queue.ts` flushes to `POST /v1/citations` + evidence presigned upload when connectivity returns; server dedups on client UUID (idempotency key).
- Conflict: if plate was paid/permitted between issue and sync, server still records citation but flags `needs_review` (officer notified).

**Testing**:
- Detox: airplane mode → issue citation stored locally; reconnect → citation appears server-side exactly once (replayed sync does not duplicate).
- Unit: sync queue retries with backoff on 5xx; gives up to local "failed" state after N attempts.

---

## Phase 10: Integrations, Interop & Public API

### Purpose
Open the platform: PARCS gate control, EV charging (OCPI 2.3), digital signage, outbound webhooks, and
standards-compliant data export (APDS/ISO 5206). These are the v1.1 "should-have" features and the
interoperability differentiator.

### Tasks

#### 10.1 — PARCS gate control
**What**: Drive entry/exit gates from permit/payment/session state, fail-safe.

**Design**:
- `integrations/parcs/base.py:PARCSAdapter.open_gate(lane_id)/close_gate/heartbeat`. `hardware_devices` + `lanes` models per data-model-1. On entry LPR/permit match → open gate; on adapter timeout, default to fail-safe mode per `research.md` §4 (barrier stays up / staffed) — never trap.
- `POST /v1/hardware/{device_id}/command` for manual override (operator role).

**Testing**:
- Integration (mocked adapter): valid permit at entry lane → `open_gate` called.
- Integration: adapter timeout → fail-safe path invoked, alert raised; no exception bubbles to caller.

#### 10.2 — EV charging (OCPI 2.3)
**What**: Manage EV stations/sessions and expose/consume OCPI parking + EV objects.

**Design**:
- `ev_stations` + `ev_charging_sessions` models per data-model-1 (OCPI EVSE/connector hierarchy). `integrations/ocpi/` implements OCPI 2.3 Locations + the new Parking module so the facility's parking spot counts are exposed to roaming partners.
- Charging session cost folds into the parking session payment (`included_in_parking` option).

**Testing**:
- Integration: start/stop charging session records `energy_kwh` and cost.
- Integration: OCPI Locations endpoint returns stations with valid OCPI EVSE structure; parking module reports spot counts.

#### 10.3 — Webhooks & public REST API
**What**: Signed outbound webhooks for events; finalise the documented OpenAPI 3.1 public API.

**Design**:
- `webhook_subscriptions(tenant_id, url, events[], secret)`; events `session.completed, citation.issued, occupancy.threshold, permit.activated, payment.completed`. Delivery via Celery with HMAC-SHA256 `X-PMS-Signature`, exponential-backoff retries, dead-letter after N.
- Publish OpenAPI 3.1 at `/openapi.json` and a docs portal; OAuth 2.0 + API-key auth documented; OWASP API Top-10 review (rate limiting via Redis, object-level authorization checks on every `/{id}` route).

**Testing**:
- Integration: `citation.issued` fires a signed webhook; receiver validates HMAC; failing receiver triggers retries then dead-letter.
- Integration: OpenAPI doc validates against OAS 3.1 schema; every route enforces tenant object-level auth (attempt cross-tenant id → 404/403).

#### 10.4 — APDS / ISO 5206 export
**What**: Standards-compliant data export for interoperability.

**Design**:
- `integrations/apds/` maps internal entities → APDS v3 objects: facility/zone/space → Place, permit → Right, rate_schedule → Rate, parking_session → Session, occupancy_snapshot → Observation. `GET /v1/exports/apds/facilities/{id}` returns an APDS-conformant JSON document.

**Testing**:
- Integration: export a seeded facility → document validates against the APDS v3 schema; Place/Rate/Occupancy fields map correctly.

---

## Phase 11: Observability, Hardening & Deployment

### Purpose
Make the platform operable and compliant: audit logging, metrics/tracing, data-retention automation, GDPR
controls for LPR/PII, and production deployment artefacts.

### Tasks

#### 11.1 — Audit log & GDPR controls
**What**: Comprehensive audit trail and privacy tooling.

**Design**:
- `audit_log` as a TimescaleDB hypertable; a SQLAlchemy event listener records create/update/delete with old/new values, user, IP. GDPR: data-subject export (`GET /v1/customers/{id}/data-export`), erasure (anonymise PII, retain financial records per retention table), and LPR retention purge (Celery deletes `lpr_reads` older than tenant retention, default 30/90 days) per `standards.md` GDPR notes.

**Testing**:
- Integration: updating a permit writes an audit_log row with correct old/new JSON.
- Integration: erasure anonymises customer PII but preserves payment_transactions financial fields.
- Integration: retention task purges LPR reads older than the configured window.

#### 11.2 — Observability
**What**: Metrics, structured logs, tracing, health checks.

**Design**:
- Prometheus metrics (`/metrics`), OpenTelemetry traces around DB/gateway/LLM calls, structured JSON logs with `tenant_id`/`request_id`. `/healthz` (liveness) and `/readyz` (DB+Redis check).

**Testing**:
- Integration: `/readyz` returns 503 when DB unreachable, 200 when healthy.
- Unit: request middleware attaches `request_id` propagated into logs.

#### 11.3 — Deployment & CI
**What**: Production compose/Helm, migrations on deploy, CI gates.

**Design**:
- CI: ruff + mypy + pytest (testcontainers) + Biome + Vitest/Playwright must pass; build all Docker images. `alembic upgrade head` runs as a release step. Optional Helm chart for Kubernetes. Document PCI-DSS scope boundary (payment service isolation) and SOC 2 control mapping per `standards.md`.

**Testing**:
- CI: pipeline fails on lint/type/test error.
- Integration: fresh `docker-compose up` reaches `/readyz` 200 with migrations applied.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Multi-Tenancy        ─── required by everything
    │
Phase 2: Facility, Space & Occupancy       ─── requires 1
    │
    ├── Phase 3: Permits & Customers        ─── requires 1,2
    │
    └── Phase 4: Rating, Sessions, Reserv.  ─── requires 2 (uses 3 for permit sessions)
            │
            └── Phase 5: Payments           ─── requires 4 (+3 for permit pay)
                    │
                    ├── Phase 6: Enforcement & Citations ─── requires 3,5
                    │
                    └── Phase 7: Frontend portals        ─── requires 3,4,5,6
                            │
                            ├── Phase 8: AI Differentiators ─── requires 2,4,6
                            │
                            └── Phase 9: Mobile Enforcement ─── requires 6 (+8.3 LPR)
                                    │
                                    └── Phase 10: Integrations & Public API ─── requires 2,4,5,6
                                            │
                                            └── Phase 11: Observability & Deploy ─── requires all
```

**Parallelism opportunities:**
- After Phase 2: **Phase 3** and **Phase 4** can be developed concurrently (Phase 4 only needs permit linkage from 3 for `permit` session type — stub until 3 lands).
- After Phases 5/6: **Phase 7 (frontend)**, **Phase 8 (AI)**, and the start of **Phase 9 (mobile)** can run in parallel by separate workstreams; Phase 9.2 LPR conflict handling depends on Phase 8.3.
- Within Phase 10, tasks 10.1–10.4 are independent of each other and can be parallelised once their domain phases exist.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`); frontend phases also pass Vitest + Playwright; mobile passes Detox.
3. Linting and formatting pass: `ruff check`, `ruff format --check`, `biome check`.
4. Type checking passes: `mypy src/` (Python), `tsc --noEmit` (frontend/mobile).
5. Docker images build (`Dockerfile.api`, `Dockerfile.worker`, frontend image).
6. Feature works end-to-end against a real Postgres+Redis via `docker-compose up` (verified by the phase's integration/e2e tests).
7. New configuration options documented in `.env.example`.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 document and validate against the OAS 3.1 schema.
9. Alembic migrations created, and `upgrade head` / `downgrade` round-trip cleanly.
10. RLS verified on any new tenant-scoped table (cross-tenant access returns no rows / 403).
11. No raw cardholder data (PAN) stored anywhere — verified for any phase touching payments.
```
