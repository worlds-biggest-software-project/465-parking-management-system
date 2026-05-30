# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) to the parking management domain. Every state change -- a vehicle entering a facility, a citation being issued, a permit being activated, a price adjustment being applied -- is recorded as an immutable event in an append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimized projections (materialized views) are maintained separately to serve queries, dashboards, and APIs.

This architecture is a natural fit for parking management because the domain is inherently event-driven: vehicles arrive and depart, sensors fire occupancy changes, LPR cameras capture plate reads, payments are processed, and enforcement actions are taken. Capturing each of these as a first-class event provides a complete audit trail, enables temporal queries ("what was the occupancy at 2pm last Tuesday?"), and supports real-time event streaming to downstream consumers like dynamic pricing engines and digital signage.

---

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Event Store | PostgreSQL with append-only tables (or EventStoreDB) | PostgreSQL provides ACID guarantees and is operationally familiar; EventStoreDB is a purpose-built alternative |
| Message Broker | Apache Kafka or NATS JetStream | Durable event streaming with topic partitioning by tenant/facility |
| Read Model Database | PostgreSQL (for transactional reads) + Redis (for hot read models) | Projections need fast reads; Redis for real-time occupancy, PostgreSQL for complex queries |
| Search / Analytics | OpenSearch or ClickHouse | Full-text search for citations/permits; columnar analytics for revenue reporting |
| Application Framework | Axon Framework (Java), Marten (C#), or custom with Node.js/Python | CQRS/ES framework with saga support |
| Snapshot Store | PostgreSQL or S3 | Periodic aggregate snapshots to avoid replaying long event streams |

---

## Event Store Schema

### Core Event Store Tables

```sql
-- The append-only event log: the single source of truth
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,          -- aggregate root ID (e.g., session_id, permit_id)
    stream_type     VARCHAR(50) NOT NULL,    -- aggregate type (e.g., 'ParkingSession', 'Permit')
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,   -- e.g., 'SessionStarted', 'CitationIssued'
    event_version   INTEGER NOT NULL,        -- version within the stream (for optimistic concurrency)
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation IDs, user context, timestamps
    causation_id    UUID,                    -- the command or event that caused this event
    correlation_id  UUID,                    -- groups related events across aggregates
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Optimistic concurrency: no two events in the same stream can have the same version
    UNIQUE (stream_id, event_version)
);

-- Primary query patterns for the event store
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_tenant ON events(tenant_id, created_at);
CREATE INDEX idx_events_correlation ON events(correlation_id);
CREATE INDEX idx_events_created ON events(created_at);

-- Partition by month for manageability at scale
-- (In production, use pg_partman for automated partition creation)

-- Snapshots to avoid replaying entire streams
CREATE TABLE snapshots (
    snapshot_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    tenant_id       UUID NOT NULL,
    version         INTEGER NOT NULL,        -- event version this snapshot represents
    state           JSONB NOT NULL,          -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, version)
);

CREATE INDEX idx_snapshots_stream ON snapshots(stream_id, version DESC);

-- Dead letter queue for failed event processing
CREATE TABLE dead_letter_events (
    dead_letter_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(event_id),
    projection_name VARCHAR(100) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 5,
    next_retry_at   TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'retrying', 'resolved', 'abandoned')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection checkpoints: track which event each projection has processed up to
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_position   BIGINT NOT NULL,         -- global position in the event log
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Type Catalog

### Parking Session Events

```
SessionStarted
  { session_id, facility_id, space_id, zone_id, license_plate, vehicle_id,
    customer_id, entry_method, entry_lane_id, entry_time }

SessionSpaceAssigned
  { session_id, space_id, zone_id, assigned_at }

SessionPaymentInitiated
  { session_id, amount, currency, payment_method, payment_token }

SessionPaymentCompleted
  { session_id, transaction_id, amount, gateway_txn_id, card_last_four }

SessionPaymentFailed
  { session_id, amount, reason, gateway_error_code }

SessionExtended
  { session_id, new_expected_exit, additional_amount, reason }

SessionValidationApplied
  { session_id, validation_code, discount_type, discount_value, sponsor }

SessionOverstayDetected
  { session_id, expected_exit, detected_at, overstay_minutes }

SessionEnded
  { session_id, exit_time, exit_method, exit_lane_id, total_duration_minutes,
    total_amount, total_paid }

SessionCancelled
  { session_id, reason, cancelled_by }
```

### Permit Events

```
PermitApplicationSubmitted
  { permit_id, customer_id, permit_type_id, vehicle_ids, requested_start }

PermitVerificationRequested
  { permit_id, verification_type, external_system }

PermitVerificationCompleted
  { permit_id, verified, verification_details }

PermitIssued
  { permit_id, permit_number, effective_from, effective_to, issued_by }

PermitActivated
  { permit_id, activated_at }

PermitVehicleAdded
  { permit_id, vehicle_id, license_plate }

PermitVehicleRemoved
  { permit_id, vehicle_id, reason }

PermitRenewed
  { permit_id, new_permit_id, new_effective_from, new_effective_to, amount }

PermitSuspended
  { permit_id, reason, suspended_by }

PermitRevoked
  { permit_id, reason, revoked_by }

PermitExpired
  { permit_id, expired_at }

PermitWaitlisted
  { permit_id, customer_id, permit_type_id, position }

PermitWaitlistOffered
  { permit_id, customer_id, offer_expires_at }
```

### Citation / Enforcement Events

```
CitationIssued
  { citation_id, citation_number, facility_id, zone_id, space_id,
    license_plate, violation_type, fine_amount, issued_by, evidence_urls,
    is_lpr_generated, lpr_confidence, location_point }

CitationMailed
  { citation_id, mailed_to_address, mailed_at }

CitationPaid
  { citation_id, transaction_id, amount, payment_method }

CitationPartiallyPaid
  { citation_id, transaction_id, amount, remaining_balance }

CitationAppealed
  { citation_id, appeal_id, appeal_reason, statement, supporting_docs }

CitationAppealReviewed
  { citation_id, appeal_id, decision, reviewed_by, review_notes,
    auto_reviewed, ai_recommendation, ai_confidence }

CitationDismissed
  { citation_id, reason, dismissed_by }

CitationEscalated
  { citation_id, escalation_type, escalated_at }

CitationLateFeeApplied
  { citation_id, late_fee_amount, new_total_due, applied_at }

EnforcementActionOrdered
  { action_id, citation_id, action_type, assigned_to, vendor }

EnforcementActionCompleted
  { action_id, citation_id, completed_at, cost }

EnforcementActionReleased
  { action_id, citation_id, released_at, release_reason }
```

### Occupancy and Sensor Events

```
SpaceOccupied
  { space_id, facility_id, zone_id, sensor_id, detected_at }

SpaceVacated
  { space_id, facility_id, zone_id, sensor_id, detected_at }

SensorHeartbeat
  { sensor_id, facility_id, battery_level, signal_strength, timestamp }

SensorOffline
  { sensor_id, facility_id, last_seen_at }

OccupancyThresholdCrossed
  { facility_id, zone_id, threshold_pct, current_pct, direction, timestamp }
```

### LPR Events

```
PlateRead
  { lpr_read_id, camera_id, facility_id, license_plate, plate_state,
    confidence_score, image_url, direction, read_at }

PlateMatched
  { lpr_read_id, vehicle_id, customer_id, matched_permit_id, matched_session_id }

PlateMatchFailed
  { lpr_read_id, license_plate, reason, requires_review }

PlateManuallyReviewed
  { lpr_read_id, reviewed_by, corrected_plate, original_plate, reviewed_at }
```

### Pricing Events

```
RateScheduleCreated
  { rate_schedule_id, facility_id, zone_id, name, rate_type, tiers, effective_from }

RateScheduleUpdated
  { rate_schedule_id, changes, updated_by }

RateScheduleDeactivated
  { rate_schedule_id, reason, deactivated_by }

DynamicPriceAdjusted
  { adjustment_id, facility_id, zone_id, adjustment_type, multiplier,
    reason, occupancy_pct, is_ai_generated, ml_model_version, confidence }

DynamicPriceReverted
  { adjustment_id, facility_id, reverted_by, reason }
```

### Facility and Configuration Events

```
FacilityCreated
  { facility_id, tenant_id, name, facility_type, location, total_spaces }

FacilityUpdated
  { facility_id, changes }

SpaceCreated
  { space_id, facility_id, zone_id, space_number, space_type }

SpaceStatusChanged
  { space_id, old_status, new_status, reason, changed_by }

ZoneCreated
  { zone_id, facility_id, name, zone_code, boundary }

DeviceRegistered
  { device_id, facility_id, device_type, manufacturer, model, ip_address }

DeviceStatusChanged
  { device_id, old_status, new_status, reason }

EVStationStatusChanged
  { ev_station_id, old_status, new_status, reason }

EVChargingSessionStarted
  { charging_session_id, ev_station_id, parking_session_id, vehicle_id, start_time }

EVChargingSessionEnded
  { charging_session_id, end_time, energy_kwh, cost }
```

---

## Read Model Projections

### Projection 1: Current Occupancy (Redis)

This is the hottest read model -- updated on every SpaceOccupied/SpaceVacated event and queried by signage, mobile apps, and the operator dashboard.

```
Redis Key Structure:
  occupancy:{tenant_id}:{facility_id}               -> Hash { total, occupied, available, pct }
  occupancy:{tenant_id}:{facility_id}:{zone_id}     -> Hash { total, occupied, available, pct }
  occupancy:{tenant_id}:{facility_id}:{level_id}    -> Hash { total, occupied, available, pct }
  space:{tenant_id}:{space_id}                       -> Hash { status, occupied_since, vehicle_plate }
```

**Projection handler** (pseudocode):
```python
def handle_space_occupied(event):
    redis.hset(f"space:{event.tenant_id}:{event.space_id}", {
        "status": "occupied",
        "occupied_since": event.detected_at,
    })
    redis.hincrby(f"occupancy:{event.tenant_id}:{event.facility_id}", "occupied", 1)
    redis.hincrby(f"occupancy:{event.tenant_id}:{event.facility_id}", "available", -1)
    recalculate_pct(event.tenant_id, event.facility_id)
    # Publish to signage WebSocket channel
    publish_occupancy_update(event.tenant_id, event.facility_id)

def handle_space_vacated(event):
    redis.hset(f"space:{event.tenant_id}:{event.space_id}", {
        "status": "available",
        "occupied_since": None,
    })
    redis.hincrby(f"occupancy:{event.tenant_id}:{event.facility_id}", "occupied", -1)
    redis.hincrby(f"occupancy:{event.tenant_id}:{event.facility_id}", "available", 1)
    recalculate_pct(event.tenant_id, event.facility_id)
    publish_occupancy_update(event.tenant_id, event.facility_id)
```

### Projection 2: Active Sessions (PostgreSQL)

```sql
-- Materialized from SessionStarted, SessionEnded, SessionPayment* events
CREATE TABLE read_active_sessions (
    session_id      UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    facility_name   VARCHAR(255),
    space_id        UUID,
    space_number    VARCHAR(20),
    zone_id         UUID,
    zone_name       VARCHAR(100),
    customer_id     UUID,
    customer_name   VARCHAR(200),
    customer_email  VARCHAR(255),
    vehicle_id      UUID,
    license_plate   VARCHAR(20),
    session_type    VARCHAR(20),
    entry_time      TIMESTAMPTZ NOT NULL,
    expected_exit   TIMESTAMPTZ,
    duration_minutes INTEGER,
    amount_due      NUMERIC(10, 2),
    amount_paid     NUMERIC(10, 2),
    payment_status  VARCHAR(20),
    permit_id       UUID,
    permit_number   VARCHAR(50),
    entry_method    VARCHAR(20),
    is_overstay     BOOLEAN DEFAULT false,
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         INTEGER NOT NULL
);

CREATE INDEX idx_read_sessions_tenant ON read_active_sessions(tenant_id);
CREATE INDEX idx_read_sessions_facility ON read_active_sessions(facility_id);
CREATE INDEX idx_read_sessions_plate ON read_active_sessions(license_plate);
CREATE INDEX idx_read_sessions_entry ON read_active_sessions(entry_time);
```

### Projection 3: Permit Registry (PostgreSQL)

```sql
-- Materialized from Permit* events
CREATE TABLE read_permits (
    permit_id       UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    permit_number   VARCHAR(50) NOT NULL,
    permit_type_name VARCHAR(100),
    permit_category VARCHAR(30),
    customer_id     UUID,
    customer_name   VARCHAR(200),
    customer_email  VARCHAR(255),
    vehicles        JSONB DEFAULT '[]',   -- [{vehicle_id, license_plate, make, model, color}]
    status          VARCHAR(20) NOT NULL,
    effective_from  TIMESTAMPTZ,
    effective_to    TIMESTAMPTZ,
    is_virtual      BOOLEAN,
    eligible_zones  JSONB DEFAULT '[]',
    eligible_facilities JSONB DEFAULT '[]',
    issued_at       TIMESTAMPTZ,
    issued_by_name  VARCHAR(200),
    renewal_count   INTEGER DEFAULT 0,
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         INTEGER NOT NULL
);

CREATE INDEX idx_read_permits_tenant ON read_permits(tenant_id);
CREATE INDEX idx_read_permits_status ON read_permits(status);
CREATE INDEX idx_read_permits_customer ON read_permits(customer_id);
-- GIN index for searching within the vehicles JSONB array
CREATE INDEX idx_read_permits_vehicles ON read_permits USING GIN(vehicles);
```

### Projection 4: Citation Lookup (PostgreSQL)

```sql
-- Materialized from Citation* and EnforcementAction* events
CREATE TABLE read_citations (
    citation_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    citation_number VARCHAR(50) NOT NULL,
    facility_id     UUID,
    facility_name   VARCHAR(255),
    zone_name       VARCHAR(100),
    space_number    VARCHAR(20),
    license_plate   VARCHAR(20) NOT NULL,
    plate_state     VARCHAR(10),
    customer_id     UUID,
    customer_name   VARCHAR(200),
    violation_type  VARCHAR(50),
    fine_amount     NUMERIC(10, 2),
    late_fee        NUMERIC(10, 2),
    total_due       NUMERIC(10, 2),
    amount_paid     NUMERIC(10, 2) DEFAULT 0,
    balance_due     NUMERIC(10, 2),
    status          VARCHAR(20),
    issued_at       TIMESTAMPTZ,
    issued_by_name  VARCHAR(200),
    is_lpr_generated BOOLEAN,
    auto_adjudicated BOOLEAN,
    due_date        DATE,
    appeal_status   VARCHAR(20),
    appeal_count    INTEGER DEFAULT 0,
    enforcement_actions JSONB DEFAULT '[]',
    evidence_urls   JSONB DEFAULT '[]',
    paid_at         TIMESTAMPTZ,
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         INTEGER NOT NULL
);

CREATE INDEX idx_read_citations_tenant ON read_citations(tenant_id);
CREATE INDEX idx_read_citations_plate ON read_citations(license_plate);
CREATE INDEX idx_read_citations_status ON read_citations(status);
CREATE INDEX idx_read_citations_issued ON read_citations(issued_at);
```

### Projection 5: Revenue Dashboard (PostgreSQL + Materialized View)

```sql
-- Materialized from SessionPaymentCompleted, CitationPaid, PermitIssued events
CREATE TABLE read_revenue_entries (
    entry_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    facility_name   VARCHAR(255),
    revenue_type    VARCHAR(30) NOT NULL,  -- 'session', 'permit', 'citation', 'ev_charging', 'validation'
    amount          NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) DEFAULT 'USD',
    payment_method  VARCHAR(20),
    source_id       UUID,    -- session_id, permit_id, or citation_id
    source_type     VARCHAR(30),
    occurred_at     TIMESTAMPTZ NOT NULL,
    hour_of_day     SMALLINT,
    day_of_week     SMALLINT,
    month           SMALLINT,
    year            SMALLINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_revenue_tenant ON read_revenue_entries(tenant_id, occurred_at);
CREATE INDEX idx_revenue_facility ON read_revenue_entries(facility_id, occurred_at);
CREATE INDEX idx_revenue_type ON read_revenue_entries(revenue_type, occurred_at);

-- Pre-aggregated daily revenue materialized view
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT
    tenant_id,
    facility_id,
    facility_name,
    revenue_type,
    DATE(occurred_at) AS revenue_date,
    day_of_week,
    COUNT(*) AS transaction_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_transaction,
    MIN(amount) AS min_transaction,
    MAX(amount) AS max_transaction
FROM read_revenue_entries
GROUP BY tenant_id, facility_id, facility_name, revenue_type,
         DATE(occurred_at), day_of_week;

CREATE INDEX idx_mv_daily_revenue ON mv_daily_revenue(tenant_id, revenue_date);
```

### Projection 6: Enforcement Officer View (Redis + PostgreSQL)

```sql
-- Hot lookup: is this plate authorized right now?
-- Redis key: plate_auth:{tenant_id}:{license_plate}
-- Value: JSON { authorized: bool, permit_id, permit_type, session_id, expires_at }

-- PostgreSQL read model for officer app
CREATE TABLE read_plate_authorization (
    license_plate   VARCHAR(20) NOT NULL,
    plate_state     VARCHAR(10),
    tenant_id       UUID NOT NULL,
    is_authorized   BOOLEAN NOT NULL DEFAULT false,
    authorization_type VARCHAR(20),  -- 'permit', 'session', 'reservation'
    authorization_id UUID,
    facility_ids    UUID[],
    zone_ids        UUID[],
    valid_from      TIMESTAMPTZ,
    valid_to        TIMESTAMPTZ,
    customer_name   VARCHAR(200),
    vehicle_description VARCHAR(200),
    citation_count  INTEGER DEFAULT 0,
    outstanding_balance NUMERIC(10, 2) DEFAULT 0,
    last_updated    TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (tenant_id, license_plate)
);

CREATE INDEX idx_plate_auth_plate ON read_plate_authorization(license_plate);
```

### Projection 7: Occupancy History (ClickHouse or PostgreSQL Time-Series)

```sql
-- For demand analysis, ML training, and occupancy forecasting
CREATE TABLE read_occupancy_history (
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    zone_id         UUID,
    timestamp_bucket TIMESTAMPTZ NOT NULL,  -- rounded to 5-minute intervals
    total_spaces    INTEGER NOT NULL,
    occupied_spaces INTEGER NOT NULL,
    occupancy_pct   NUMERIC(5, 2),
    entry_count     INTEGER DEFAULT 0,     -- entries in this 5-min window
    exit_count      INTEGER DEFAULT 0,     -- exits in this 5-min window
    PRIMARY KEY (tenant_id, facility_id, zone_id, timestamp_bucket)
);

-- Hourly rollup for dashboard queries
CREATE TABLE read_occupancy_hourly (
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    zone_id         UUID,
    hour_bucket     TIMESTAMPTZ NOT NULL,
    avg_occupancy_pct NUMERIC(5, 2),
    peak_occupancy_pct NUMERIC(5, 2),
    min_occupancy_pct NUMERIC(5, 2),
    total_entries   INTEGER DEFAULT 0,
    total_exits     INTEGER DEFAULT 0,
    PRIMARY KEY (tenant_id, facility_id, zone_id, hour_bucket)
);
```

---

## Command Handlers and Sagas

### Example: Start Parking Session (Command Handler)

```python
class StartParkingSessionHandler:
    def handle(self, command: StartParkingSession):
        # Validate: is the space available? Is the facility active?
        facility = self.facility_repo.load(command.facility_id)
        if not facility.is_active():
            raise FacilityInactiveError(command.facility_id)

        space = self.space_repo.load(command.space_id)
        if space.is_occupied():
            raise SpaceOccupiedError(command.space_id)

        # Create the session aggregate
        session = ParkingSession.start(
            session_id=uuid4(),
            tenant_id=command.tenant_id,
            facility_id=command.facility_id,
            space_id=command.space_id,
            zone_id=command.zone_id,
            license_plate=command.license_plate,
            vehicle_id=command.vehicle_id,
            customer_id=command.customer_id,
            entry_method=command.entry_method,
            entry_lane_id=command.entry_lane_id,
            entry_time=command.entry_time,
        )
        # Persist events: [SessionStarted]
        self.event_store.save(session.pending_events)
        return session.session_id
```

### Saga: Citation Auto-Adjudication

This saga coordinates the automated citation resolution workflow, reacting to `CitationIssued` events and cross-referencing permit and payment data.

```
Saga: CitationAutoAdjudication

Triggered by: CitationIssued (where is_lpr_generated = true)

Step 1: Check permit registry
  -> Query read_permits for license_plate
  -> If valid permit found for the zone/facility:
     -> Emit CitationDismissed { reason: "valid_permit_found", permit_id }
     -> END

Step 2: Check active sessions
  -> Query read_active_sessions for license_plate
  -> If active paid session found:
     -> Emit CitationDismissed { reason: "active_paid_session", session_id }
     -> END

Step 3: Check LPR confidence
  -> If lpr_confidence < 0.85:
     -> Emit CitationFlaggedForReview { reason: "low_lpr_confidence" }
     -> END (awaits manual review)

Step 4: Check grace periods
  -> If time_since_expiry < grace_period_minutes:
     -> Emit CitationDismissed { reason: "within_grace_period" }
     -> END

Step 5: Confirm violation
  -> Emit CitationConfirmed { auto_adjudicated: true, checks_performed: [...] }
  -> If customer_id known, emit CitationNotificationSent { method: "email" }
```

### Saga: Dynamic Pricing Adjustment

```
Saga: DynamicPricingUpdate

Triggered by: OccupancyThresholdCrossed

Step 1: Evaluate pricing rules
  -> Load current rate_schedule for facility/zone
  -> Load ML model prediction (if available)
  -> Calculate new multiplier based on:
     - Current occupancy percentage
     - Time of day / day of week
     - Historical demand patterns
     - Active events nearby

Step 2: Apply adjustment
  -> If multiplier change > 10%, require human approval:
     -> Emit DynamicPriceProposed { ... awaiting_approval: true }
  -> Else:
     -> Emit DynamicPriceAdjusted { multiplier, reason, is_ai_generated: true }

Step 3: Update signage
  -> Emit SignageContentUpdated { display_ids, new_rate_display }
```

---

## Event Flow Architecture

```
                    Commands
                       |
                       v
              +------------------+
              | Command Handlers |
              | (Write Side)     |
              +--------+---------+
                       |
                       v
              +------------------+        +-------------------+
              |   Event Store    | -----> |   Message Broker  |
              | (PostgreSQL /    |        |   (Kafka / NATS)  |
              |  EventStoreDB)  |        +---+----+----+------+
              +------------------+            |    |    |
                                              |    |    |
              +-------------------------------+    |    +---------------------------+
              |                                    |                                |
              v                                    v                                v
    +------------------+              +------------------+              +------------------+
    | Occupancy        |              | Session & Permit |              | Revenue          |
    | Projection       |              | Projections      |              | Projection       |
    | (Redis)          |              | (PostgreSQL)     |              | (ClickHouse)     |
    +------------------+              +------------------+              +------------------+
              |                                    |                                |
              v                                    v                                v
    +------------------+              +------------------+              +------------------+
    | Digital Signage  |              | Customer Portal  |              | Analytics        |
    | Mobile App       |              | Enforcement App  |              | Dashboard        |
    | Wayfinding       |              | Admin Portal     |              | ML Pipeline      |
    +------------------+              +------------------+              +------------------+
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by design**: Every state change is an immutable event. You never need to ask "how did this citation get dismissed?" or "who changed the rate at 3am?" -- the event stream tells the full story. This is invaluable for regulatory compliance, financial auditing, and dispute resolution.

2. **Temporal queries**: You can reconstruct the state of any aggregate at any point in time. "What was the occupancy of Zone A at 2pm last Tuesday?" is answered by replaying events up to that timestamp. This directly supports the AI-driven occupancy forecasting feature.

3. **Natural fit for IoT event streams**: Sensor readings, LPR captures, and gate events are inherently events. Rather than transforming them into CRUD operations against normalized tables, they flow naturally into the event store.

4. **Independent scaling of reads and writes**: The event store (write side) can be a small, highly-available PostgreSQL cluster. Read projections scale independently -- Redis for real-time occupancy, ClickHouse for analytics, PostgreSQL for transactional queries. Each read model is optimized for its specific access pattern.

5. **Decoupled consumers**: Adding a new feature (e.g., "send SMS when occupancy exceeds 90%") requires only a new event handler subscribing to `OccupancyThresholdCrossed`. No changes to existing code.

6. **Replay and reprocessing**: If a projection has a bug, fix the projection code and replay the event stream to rebuild the correct state. No data is ever lost.

7. **Dynamic pricing engine integration**: The pricing engine subscribes to occupancy and session events in real-time, enabling sub-second reaction to demand changes. The event stream also provides the historical data needed to train ML pricing models.

8. **Multi-tenancy through event partitioning**: Kafka topics or NATS subjects can be partitioned by tenant_id, providing logical isolation. Individual tenant event streams can be exported for data portability.

### Cons

1. **Significant complexity increase**: Event sourcing adds substantial architectural complexity. The team needs to understand aggregate design, eventual consistency, saga orchestration, projection management, and snapshot strategies. This is a much larger learning curve than traditional CRUD.

2. **Eventual consistency**: Read models are updated asynchronously. After a vehicle enters a facility, the occupancy dashboard may show stale data for a few hundred milliseconds (or longer under load). For most parking use cases, this is acceptable, but enforcement officers may occasionally see slightly outdated permit status.

3. **Schema evolution challenges**: Once events are stored, their schema cannot be changed retroactively. If `SessionStarted` v1 doesn't include a field needed later, you must either introduce a new event type or use upcasting (transforming old events when replayed). This requires disciplined versioning.

4. **Projection rebuild time**: For a system with 100 million events, rebuilding a projection from scratch could take hours. Snapshots mitigate this for aggregate replay, but projection rebuilds require processing the entire event stream.

5. **Debugging difficulty**: Troubleshooting a bug requires understanding the full event chain, potentially across multiple aggregates and sagas. Traditional "query the current row" debugging is not available.

6. **Storage volume**: Storing every event (especially high-frequency sensor and LPR events) consumes significantly more storage than a mutable-state model. A busy facility might generate 50,000+ events per day from sensors alone.

7. **Testing complexity**: Integration tests need to set up event streams, run projections, and verify both write-side invariants and read-side consistency. This is more involved than testing CRUD operations.

8. **Operational overhead**: Running Kafka/NATS, managing projection workers, monitoring consumer lag, handling dead letters -- all add operational burden compared to a single PostgreSQL database.

---

## Migration and Scaling Considerations

### Initial Deployment

- Use PostgreSQL as both the event store and read model database to minimize operational complexity.
- Use a simple in-process event bus (no Kafka) for the first deployment. Events are written to the events table and projections are updated synchronously or via PostgreSQL LISTEN/NOTIFY.
- Start with 3-5 core projections: active sessions, permits, citations, occupancy, revenue.

### Growth Path

- Introduce Kafka or NATS JetStream when you need to decouple projection processing from command handling, or when event volume exceeds what PostgreSQL LISTEN/NOTIFY can handle (~10,000 events/second).
- Move the occupancy projection to Redis for sub-millisecond reads.
- Add ClickHouse for analytics projections when historical queries start impacting the main PostgreSQL database.

### Enterprise Scale

- Partition the event store by tenant_id using Citus or by deploying separate event store databases per region/municipality cluster.
- Implement snapshot strategies: snapshot aggregates every 100 events to cap replay time.
- Automate projection rebuild: a "reset projection" command that replays the event stream through a rebuilt projection in a new table, then swaps.
- Archive old events (>2 years) to cold storage (S3 in Parquet format) with an on-demand replay capability.

### Event Store Sizing

| Metric | Small Operator | Medium Operator | Large Municipality |
|--------|---------------|-----------------|-------------------|
| Facilities | 1-5 | 10-50 | 100-500 |
| Events/day | 5,000 | 50,000 | 500,000 |
| Events/year | 1.8M | 18M | 180M |
| Storage/year (compressed) | 2 GB | 20 GB | 200 GB |
| Projections | 5-8 | 8-12 | 12-20 |

---

## Summary

The event-sourced CQRS model is the most architecturally sophisticated option. It provides an unparalleled audit trail, temporal query capability, and decoupled scalability -- all highly valuable for a parking management platform where regulatory compliance, financial accountability, and real-time IoT data processing are core requirements. The tradeoff is significant complexity in development, testing, and operations. This model is best suited for teams with event-sourcing experience building a platform intended for large-scale, multi-tenant deployment where the audit and analytics capabilities justify the engineering investment.
