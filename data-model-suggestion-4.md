# Data Model Suggestion 4: Time-Series + Relational Polyglot Architecture (PostgreSQL + TimescaleDB + Redis)

## Overview

This model recognizes that a parking management system is fundamentally a dual-natured data problem. Half of the data is transactional and relational (customers, permits, payments, citations) and benefits from ACID guarantees, foreign keys, and normalized structure. The other half -- sensor readings, occupancy events, LPR captures, pricing signals, and demand metrics -- is high-frequency time-series data that needs specialized storage, compression, and aggregation. Forcing both into a single paradigm creates compromises in either direction.

This design uses a polyglot persistence strategy with a carefully chosen set of specialized databases:

- **PostgreSQL** for transactional/relational data (the business domain)
- **TimescaleDB** (PostgreSQL extension) for time-series sensor and event data
- **Redis** for real-time state (live occupancy, session caches, rate caches)

The critical insight is that TimescaleDB is a PostgreSQL extension, not a separate database engine. This means both the relational tables and the time-series hypertables live in the same PostgreSQL instance, share the same connection, participate in the same transactions, and can be JOINed in a single query. You get specialized time-series performance without the operational overhead of a truly separate database.

---

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Relational Core | PostgreSQL 16+ | Business entities, transactions, referential integrity |
| Time-Series Engine | TimescaleDB 2.x (PostgreSQL extension) | Hypertables, compression, continuous aggregates for IoT/sensor data |
| Spatial Extension | PostGIS 3.4+ | Geospatial queries coexisting with TimescaleDB in the same PostgreSQL instance |
| Real-Time Cache | Redis 7+ with Redis Streams | Sub-millisecond occupancy lookups, session state, event streaming |
| Analytics / ML | ClickHouse (optional, for scale) | Columnar analytics for demand forecasting training data at enterprise scale |
| Message Broker | Redis Streams or NATS | Lightweight pub/sub for real-time occupancy updates to signage and apps |
| Connection Pooling | PgBouncer | Multi-tenant connection management |

---

## Why Time-Series is the Right Specialty for Parking

Parking management generates massive volumes of time-stamped data:

| Data Source | Frequency | Volume (100 facilities) | Access Pattern |
|-------------|-----------|------------------------|----------------|
| Occupancy sensors | Every 1-30 seconds per space | 50-500M readings/month | Recent: real-time dashboards. Historical: demand forecasting |
| LPR cameras | Every vehicle entry/exit + patrols | 5-50M reads/month | Recent: enforcement lookups. Historical: traffic analysis |
| Pricing signals | Every price adjustment | 100K-1M/month | Recent: current rate display. Historical: revenue optimization |
| Gate/barrier events | Every vehicle passage | 5-30M events/month | Recent: session tracking. Historical: throughput analysis |
| EV charging telemetry | Every 15-60 seconds per session | 10-100M readings/month | Recent: charging status. Historical: energy analytics |
| Payment events | Per transaction | 1-10M/month | Recent: reconciliation. Historical: financial reporting |

Traditional relational databases handle this volume by partitioning and archiving, but they cannot efficiently:
- Compress old data 10-20x automatically (TimescaleDB native compression)
- Maintain pre-computed aggregates that update incrementally (continuous aggregates)
- Execute time-bucket queries ("average occupancy per 15-minute window") without full table scans
- Retain high-resolution recent data while downsampling historical data automatically

TimescaleDB addresses all four of these natively.

---

## Schema: Relational Core (PostgreSQL)

The relational tables are substantially the same as Data Model Suggestion 1. Below, I include abbreviated versions of the key tables and focus on the integration points with the time-series layer.

```sql
-- ============================================================
-- RELATIONAL CORE: Business domain entities
-- These tables use standard PostgreSQL features.
-- ============================================================

CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    config          JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE facilities (
    facility_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    facility_type   VARCHAR(30) NOT NULL,
    location        GEOGRAPHY(POINT, 4326),
    boundary        GEOGRAPHY(POLYGON, 4326),
    total_spaces    INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE zones (
    zone_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(100) NOT NULL,
    zone_code       VARCHAR(20),
    boundary        GEOGRAPHY(POLYGON, 4326),
    total_spaces    INTEGER NOT NULL DEFAULT 0,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE spaces (
    space_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    zone_id         UUID REFERENCES zones(zone_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    space_number    VARCHAR(20) NOT NULL,
    space_type      VARCHAR(30) NOT NULL DEFAULT 'standard',
    sensor_id       VARCHAR(100),
    location        GEOGRAPHY(POINT, 4326),
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (facility_id, space_number)
);

CREATE TABLE customers (
    customer_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    customer_type   VARCHAR(30) NOT NULL DEFAULT 'individual',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    balance         NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE vehicles (
    vehicle_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(customer_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    license_plate   VARCHAR(20) NOT NULL,
    plate_state     VARCHAR(10),
    plate_country   CHAR(2) NOT NULL DEFAULT 'US',
    make            VARCHAR(50),
    model           VARCHAR(50),
    color           VARCHAR(30),
    is_ev           BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicles_plate ON vehicles(license_plate, plate_state, plate_country);

-- Permits, Citations, Payments, Reservations, etc.
-- (See Data Model Suggestion 1 for full definitions of these relational tables.
--  They remain unchanged in this architecture.)

CREATE TABLE permits (
    permit_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    permit_type_id  UUID NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customers(customer_id),
    permit_number   VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    effective_from  TIMESTAMPTZ NOT NULL,
    effective_to    TIMESTAMPTZ,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, permit_number)
);

CREATE TABLE parking_sessions (
    session_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    space_id        UUID REFERENCES spaces(space_id),
    zone_id         UUID REFERENCES zones(zone_id),
    customer_id     UUID REFERENCES customers(customer_id),
    vehicle_id      UUID REFERENCES vehicles(vehicle_id),
    license_plate   VARCHAR(20),
    session_type    VARCHAR(20) NOT NULL DEFAULT 'transient',
    entry_time      TIMESTAMPTZ NOT NULL,
    exit_time       TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    amount_due      NUMERIC(10, 2) DEFAULT 0.00,
    amount_paid     NUMERIC(10, 2) DEFAULT 0.00,
    permit_id       UUID REFERENCES permits(permit_id),
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_plate ON parking_sessions(license_plate);
CREATE INDEX idx_sessions_active ON parking_sessions(facility_id, status)
    WHERE status = 'active';

CREATE TABLE citations (
    citation_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    citation_number VARCHAR(50) NOT NULL,
    license_plate   VARCHAR(20) NOT NULL,
    violation_type  VARCHAR(50) NOT NULL,
    fine_amount     NUMERIC(10, 2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'issued',
    issued_by       UUID,
    issued_at       TIMESTAMPTZ NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, citation_number)
);

CREATE TABLE payment_transactions (
    transaction_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    customer_id     UUID REFERENCES customers(customer_id),
    session_id      UUID REFERENCES parking_sessions(session_id),
    permit_id       UUID REFERENCES permits(permit_id),
    citation_id     UUID REFERENCES citations(citation_id),
    transaction_type VARCHAR(20) NOT NULL,
    payment_method  VARCHAR(20) NOT NULL,
    amount          NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payment_token   VARCHAR(255),
    gateway         VARCHAR(50),
    gateway_txn_id  VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    gateway_data    JSONB NOT NULL DEFAULT '{}',
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Schema: Time-Series Layer (TimescaleDB Hypertables)

```sql
-- ============================================================
-- TIME-SERIES LAYER: High-frequency IoT and event data
-- These tables use TimescaleDB hypertables with automatic
-- partitioning, compression, and continuous aggregates.
-- ============================================================

-- Enable the TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- -------------------------------------------------------
-- 1. OCCUPANCY SENSOR READINGS
-- The highest-volume table. Each sensor reports every 1-30 seconds.
-- -------------------------------------------------------
CREATE TABLE ts_occupancy_readings (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    zone_id         UUID,
    space_id        UUID,
    sensor_id       VARCHAR(100) NOT NULL,
    is_occupied     BOOLEAN NOT NULL,
    battery_pct     SMALLINT,
    signal_rssi     SMALLINT,
    temperature_c   REAL,
    raw_distance_cm REAL
);

-- Convert to a hypertable, partitioned by time in 1-day chunks
SELECT create_hypertable('ts_occupancy_readings', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Add space-partitioning for multi-tenant workloads (hash by tenant_id)
SELECT add_dimension('ts_occupancy_readings', 'tenant_id',
    number_partitions => 4);

-- Indexes optimized for typical queries
CREATE INDEX idx_ts_occ_sensor ON ts_occupancy_readings(sensor_id, time DESC);
CREATE INDEX idx_ts_occ_facility ON ts_occupancy_readings(facility_id, time DESC);
CREATE INDEX idx_ts_occ_space ON ts_occupancy_readings(space_id, time DESC);

-- Enable compression after 7 days (10-20x storage reduction)
ALTER TABLE ts_occupancy_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, facility_id, sensor_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_occupancy_readings', INTERVAL '7 days');

-- Retention policy: drop raw data after 90 days (aggregates are preserved)
SELECT add_retention_policy('ts_occupancy_readings', INTERVAL '90 days');


-- -------------------------------------------------------
-- 2. CONTINUOUS AGGREGATES: Pre-computed occupancy summaries
-- These are incrementally maintained materialized views.
-- -------------------------------------------------------

-- 5-minute occupancy summary per facility/zone
CREATE MATERIALIZED VIEW ts_occupancy_5min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('5 minutes', time) AS bucket,
    tenant_id,
    facility_id,
    zone_id,
    COUNT(*) FILTER (WHERE is_occupied) AS occupied_readings,
    COUNT(*) FILTER (WHERE NOT is_occupied) AS vacant_readings,
    COUNT(*) AS total_readings,
    -- Approximate occupancy: ratio of occupied readings
    ROUND(
        COUNT(*) FILTER (WHERE is_occupied)::NUMERIC / NULLIF(COUNT(*), 0) * 100, 2
    ) AS occupancy_pct,
    AVG(battery_pct) FILTER (WHERE battery_pct IS NOT NULL) AS avg_battery_pct,
    MIN(signal_rssi) FILTER (WHERE signal_rssi IS NOT NULL) AS min_signal_rssi
FROM ts_occupancy_readings
GROUP BY bucket, tenant_id, facility_id, zone_id;

-- Refresh policy: update every 5 minutes, covering the last 30 minutes
SELECT add_continuous_aggregate_policy('ts_occupancy_5min',
    start_offset => INTERVAL '30 minutes',
    end_offset => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '5 minutes');

-- Hourly occupancy summary (aggregated from 5-minute buckets)
CREATE MATERIALIZED VIEW ts_occupancy_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', bucket) AS hour_bucket,
    tenant_id,
    facility_id,
    zone_id,
    SUM(occupied_readings) AS total_occupied_readings,
    SUM(vacant_readings) AS total_vacant_readings,
    SUM(total_readings) AS total_readings,
    AVG(occupancy_pct) AS avg_occupancy_pct,
    MAX(occupancy_pct) AS peak_occupancy_pct,
    MIN(occupancy_pct) AS min_occupancy_pct
FROM ts_occupancy_5min
GROUP BY hour_bucket, tenant_id, facility_id, zone_id;

SELECT add_continuous_aggregate_policy('ts_occupancy_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Daily occupancy summary (for long-term trend analysis and ML training)
CREATE MATERIALIZED VIEW ts_occupancy_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', hour_bucket) AS day_bucket,
    tenant_id,
    facility_id,
    zone_id,
    SUM(total_occupied_readings) AS total_occupied_readings,
    SUM(total_readings) AS total_readings,
    AVG(avg_occupancy_pct) AS avg_occupancy_pct,
    MAX(peak_occupancy_pct) AS peak_occupancy_pct,
    MIN(min_occupancy_pct) AS min_occupancy_pct
FROM ts_occupancy_hourly
GROUP BY day_bucket, tenant_id, facility_id, zone_id;

SELECT add_continuous_aggregate_policy('ts_occupancy_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');


-- -------------------------------------------------------
-- 3. LPR CAMERA READS
-- High-volume plate recognition events from fixed and mobile cameras.
-- -------------------------------------------------------
CREATE TABLE ts_lpr_reads (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    camera_id       VARCHAR(100) NOT NULL,
    license_plate   VARCHAR(20) NOT NULL,
    plate_state     VARCHAR(10),
    plate_country   CHAR(2) DEFAULT 'US',
    confidence      REAL NOT NULL,
    direction       VARCHAR(10),  -- 'entry', 'exit', 'patrol'
    image_url       TEXT,
    matched_vehicle_id UUID,
    matched_session_id UUID,
    requires_review BOOLEAN NOT NULL DEFAULT false,
    reviewed        BOOLEAN NOT NULL DEFAULT false,
    corrected_plate VARCHAR(20),
    metadata        JSONB DEFAULT '{}'
);

SELECT create_hypertable('ts_lpr_reads', 'time',
    chunk_time_interval => INTERVAL '1 day');

SELECT add_dimension('ts_lpr_reads', 'tenant_id',
    number_partitions => 4);

CREATE INDEX idx_ts_lpr_plate ON ts_lpr_reads(license_plate, time DESC);
CREATE INDEX idx_ts_lpr_camera ON ts_lpr_reads(camera_id, time DESC);
CREATE INDEX idx_ts_lpr_facility ON ts_lpr_reads(facility_id, time DESC);
CREATE INDEX idx_ts_lpr_review ON ts_lpr_reads(requires_review, time DESC)
    WHERE requires_review = true AND reviewed = false;

ALTER TABLE ts_lpr_reads SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, facility_id, camera_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_lpr_reads', INTERVAL '30 days');
SELECT add_retention_policy('ts_lpr_reads', INTERVAL '1 year');

-- LPR throughput continuous aggregate: entries/exits per hour per facility
CREATE MATERIALIZED VIEW ts_lpr_throughput_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour_bucket,
    tenant_id,
    facility_id,
    direction,
    COUNT(*) AS read_count,
    COUNT(*) FILTER (WHERE confidence >= 0.90) AS high_confidence_count,
    COUNT(*) FILTER (WHERE confidence < 0.85) AS low_confidence_count,
    AVG(confidence) AS avg_confidence,
    COUNT(DISTINCT license_plate) AS unique_plates
FROM ts_lpr_reads
GROUP BY hour_bucket, tenant_id, facility_id, direction;

SELECT add_continuous_aggregate_policy('ts_lpr_throughput_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');


-- -------------------------------------------------------
-- 4. GATE AND BARRIER EVENTS
-- Hardware events from PARCS equipment.
-- -------------------------------------------------------
CREATE TABLE ts_gate_events (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    device_id       UUID NOT NULL,
    lane_id         UUID,
    event_type      VARCHAR(30) NOT NULL,
    -- 'gate_opened', 'gate_closed', 'barrier_raised', 'barrier_lowered',
    -- 'ticket_issued', 'ticket_read', 'card_presented', 'access_granted',
    -- 'access_denied', 'intercom_call', 'sensor_triggered', 'error'
    license_plate   VARCHAR(20),
    session_id      UUID,
    details         JSONB DEFAULT '{}'
);

SELECT create_hypertable('ts_gate_events', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ts_gate_facility ON ts_gate_events(facility_id, time DESC);
CREATE INDEX idx_ts_gate_device ON ts_gate_events(device_id, time DESC);
CREATE INDEX idx_ts_gate_type ON ts_gate_events(event_type, time DESC);

ALTER TABLE ts_gate_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, facility_id, device_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_gate_events', INTERVAL '14 days');
SELECT add_retention_policy('ts_gate_events', INTERVAL '180 days');


-- -------------------------------------------------------
-- 5. PRICING SIGNALS
-- Every dynamic pricing calculation, rate change, and demand signal.
-- Used to train and evaluate the ML pricing model.
-- -------------------------------------------------------
CREATE TABLE ts_pricing_signals (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    zone_id         UUID,
    signal_type     VARCHAR(30) NOT NULL,
    -- 'occupancy_sample', 'price_adjusted', 'event_detected',
    -- 'weather_update', 'demand_forecast', 'model_prediction'
    occupancy_pct   REAL,
    current_rate    NUMERIC(10, 2),
    multiplier      REAL,
    new_rate        NUMERIC(10, 2),
    demand_score    REAL,
    -- Additional context from the ML pipeline
    model_version   VARCHAR(50),
    confidence      REAL,
    features        JSONB DEFAULT '{}'
    -- {
    --   "hour_of_day": 14,
    --   "day_of_week": 2,
    --   "is_holiday": false,
    --   "weather": { "temp_f": 72, "condition": "clear", "precip_pct": 0 },
    --   "nearby_events": [{ "name": "Concert", "venue": "Arena", "expected_attendees": 15000 }],
    --   "historical_avg_occupancy": 0.78,
    --   "entry_rate_per_hour": 45,
    --   "exit_rate_per_hour": 30
    -- }
);

SELECT create_hypertable('ts_pricing_signals', 'time',
    chunk_time_interval => INTERVAL '7 days');

CREATE INDEX idx_ts_pricing_facility ON ts_pricing_signals(facility_id, time DESC);
CREATE INDEX idx_ts_pricing_type ON ts_pricing_signals(signal_type, time DESC);

ALTER TABLE ts_pricing_signals SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, facility_id, signal_type',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_pricing_signals', INTERVAL '30 days');
SELECT add_retention_policy('ts_pricing_signals', INTERVAL '2 years');

-- Pricing effectiveness aggregate: how did price changes affect occupancy?
CREATE MATERIALIZED VIEW ts_pricing_effectiveness
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour_bucket,
    tenant_id,
    facility_id,
    zone_id,
    AVG(occupancy_pct) AS avg_occupancy,
    AVG(multiplier) AS avg_multiplier,
    AVG(new_rate) AS avg_rate,
    AVG(demand_score) AS avg_demand_score,
    COUNT(*) FILTER (WHERE signal_type = 'price_adjusted') AS price_changes
FROM ts_pricing_signals
GROUP BY hour_bucket, tenant_id, facility_id, zone_id;

SELECT add_continuous_aggregate_policy('ts_pricing_effectiveness',
    start_offset => INTERVAL '6 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');


-- -------------------------------------------------------
-- 6. EV CHARGING TELEMETRY
-- Real-time energy consumption from EV charging stations.
-- -------------------------------------------------------
CREATE TABLE ts_ev_telemetry (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    station_id      UUID NOT NULL,
    session_id      UUID,
    power_kw        REAL NOT NULL,
    energy_kwh_total REAL,           -- cumulative for this session
    voltage_v       REAL,
    current_a       REAL,
    soc_pct         REAL,            -- state of charge (if reported by vehicle)
    temperature_c   REAL,
    status          VARCHAR(20)      -- 'charging', 'idle', 'error', 'finishing'
);

SELECT create_hypertable('ts_ev_telemetry', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ts_ev_station ON ts_ev_telemetry(station_id, time DESC);
CREATE INDEX idx_ts_ev_session ON ts_ev_telemetry(session_id, time DESC);

ALTER TABLE ts_ev_telemetry SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, facility_id, station_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_ev_telemetry', INTERVAL '7 days');
SELECT add_retention_policy('ts_ev_telemetry', INTERVAL '1 year');

-- EV energy consumption hourly aggregate
CREATE MATERIALIZED VIEW ts_ev_energy_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour_bucket,
    tenant_id,
    facility_id,
    station_id,
    MAX(energy_kwh_total) - MIN(energy_kwh_total) AS energy_kwh_delivered,
    AVG(power_kw) AS avg_power_kw,
    MAX(power_kw) AS peak_power_kw,
    COUNT(DISTINCT session_id) AS active_sessions
FROM ts_ev_telemetry
GROUP BY hour_bucket, tenant_id, facility_id, station_id;

SELECT add_continuous_aggregate_policy('ts_ev_energy_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');


-- -------------------------------------------------------
-- 7. DEVICE HEALTH / HEARTBEATS
-- Monitor the health of all IoT devices.
-- -------------------------------------------------------
CREATE TABLE ts_device_health (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    device_id       UUID NOT NULL,
    device_type     VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL,  -- 'online', 'degraded', 'offline', 'error'
    cpu_pct         REAL,
    memory_pct      REAL,
    disk_pct        REAL,
    uptime_seconds  BIGINT,
    error_count     INTEGER DEFAULT 0,
    diagnostics     JSONB DEFAULT '{}'
);

SELECT create_hypertable('ts_device_health', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ts_health_device ON ts_device_health(device_id, time DESC);

ALTER TABLE ts_device_health SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, facility_id, device_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_device_health', INTERVAL '3 days');
SELECT add_retention_policy('ts_device_health', INTERVAL '90 days');
```

---

## Redis Real-Time State Layer

```
# ============================================================
# REDIS: Real-time state for sub-millisecond lookups
# ============================================================

# --- Live Occupancy Counters ---
# Updated by the application layer when occupancy events arrive
# Read by signage, mobile apps, and the operator dashboard

HSET occupancy:{tenant_id}:{facility_id} \
    total 500 \
    occupied 342 \
    available 158 \
    pct 68.4 \
    updated_at "2026-09-15T14:30:00Z"

HSET occupancy:{tenant_id}:{facility_id}:zone:{zone_id} \
    total 80 \
    occupied 65 \
    available 15 \
    pct 81.3

# --- Individual Space Status ---
HSET space:{tenant_id}:{space_id} \
    status "occupied" \
    occupied_since "2026-09-15T13:45:00Z" \
    vehicle_plate "ABC-1234" \
    session_id "uuid-xxx"

# --- Active Session Cache ---
# Denormalized session data for fast enforcement lookups
SET session:plate:{tenant_id}:{license_plate} \
    '{"session_id":"...","facility_id":"...","entry_time":"...","amount_due":12.00,"permit_id":"..."}'
    EX 86400  # TTL: 24 hours

# --- Current Rate Cache ---
# Active pricing for each facility/zone
HSET rate:{tenant_id}:{facility_id}:{zone_id} \
    base_rate 3.50 \
    multiplier 1.25 \
    effective_rate 4.38 \
    reason "demand_surge" \
    updated_at "2026-09-15T14:00:00Z"

# --- Plate Authorization Cache ---
# Enforcement officer quick-check: is this plate authorized?
SET auth:plate:{tenant_id}:{license_plate} \
    '{"authorized":true,"type":"permit","permit_id":"...","zones":["z1","z2"],"expires":"..."}'
    EX 3600  # TTL: 1 hour, refreshed on permit changes

# --- Redis Streams for Real-Time Events ---
# Occupancy change stream (consumed by signage, mobile app, dynamic pricing)
XADD stream:occupancy:{tenant_id}:{facility_id} MAXLEN ~10000 * \
    space_id "uuid" \
    action "occupied" \
    zone_id "uuid" \
    timestamp "2026-09-15T14:30:00Z"

# Pricing update stream (consumed by signage and mobile app)
XADD stream:pricing:{tenant_id}:{facility_id} MAXLEN ~1000 * \
    zone_id "uuid" \
    new_rate 4.38 \
    multiplier 1.25 \
    reason "demand_surge"
```

---

## Cross-Layer Query Examples

The power of TimescaleDB being a PostgreSQL extension is that relational and time-series data can be JOINed in a single query:

```sql
-- Demand analysis: average occupancy by hour of day, joined with revenue data
SELECT
    EXTRACT(HOUR FROM oh.hour_bucket) AS hour_of_day,
    f.name AS facility_name,
    ROUND(AVG(oh.avg_occupancy_pct), 1) AS avg_occupancy,
    ROUND(AVG(oh.peak_occupancy_pct), 1) AS avg_peak_occupancy,
    COUNT(DISTINCT ps.session_id) AS avg_sessions,
    COALESCE(SUM(pt.amount), 0) AS total_revenue
FROM ts_occupancy_hourly oh
JOIN facilities f ON f.facility_id = oh.facility_id
LEFT JOIN parking_sessions ps ON ps.facility_id = oh.facility_id
    AND ps.entry_time >= oh.hour_bucket
    AND ps.entry_time < oh.hour_bucket + INTERVAL '1 hour'
LEFT JOIN payment_transactions pt ON pt.session_id = ps.session_id
    AND pt.status = 'completed'
WHERE oh.tenant_id = :tenant_id
    AND oh.hour_bucket >= :start_date
    AND oh.hour_bucket < :end_date
GROUP BY hour_of_day, f.name
ORDER BY hour_of_day;

-- ML training data: pricing signals enriched with outcome data
SELECT
    ps.time,
    ps.facility_id,
    ps.zone_id,
    ps.occupancy_pct,
    ps.multiplier,
    ps.new_rate,
    ps.demand_score,
    ps.features,
    -- Outcome: what happened in the next hour?
    next_occ.avg_occupancy AS next_hour_occupancy,
    next_rev.revenue AS next_hour_revenue
FROM ts_pricing_signals ps
LEFT JOIN LATERAL (
    SELECT AVG(avg_occupancy_pct) AS avg_occupancy
    FROM ts_occupancy_hourly oh
    WHERE oh.facility_id = ps.facility_id
        AND oh.zone_id = ps.zone_id
        AND oh.hour_bucket = time_bucket('1 hour', ps.time) + INTERVAL '1 hour'
) next_occ ON true
LEFT JOIN LATERAL (
    SELECT SUM(pt.amount) AS revenue
    FROM payment_transactions pt
    JOIN parking_sessions sess ON sess.session_id = pt.session_id
    WHERE sess.facility_id = ps.facility_id
        AND pt.status = 'completed'
        AND pt.created_at >= time_bucket('1 hour', ps.time) + INTERVAL '1 hour'
        AND pt.created_at < time_bucket('1 hour', ps.time) + INTERVAL '2 hours'
) next_rev ON true
WHERE ps.tenant_id = :tenant_id
    AND ps.signal_type = 'price_adjusted'
    AND ps.time >= :training_start
    AND ps.time < :training_end;

-- Occupancy forecast data: last 72 hours of 15-minute occupancy with day/hour features
SELECT
    time_bucket('15 minutes', time) AS bucket,
    tenant_id,
    facility_id,
    zone_id,
    AVG(CASE WHEN is_occupied THEN 1.0 ELSE 0.0 END) AS occupancy_ratio,
    EXTRACT(DOW FROM time) AS day_of_week,
    EXTRACT(HOUR FROM time) AS hour_of_day,
    EXTRACT(MINUTE FROM time) / 15 AS quarter_hour
FROM ts_occupancy_readings
WHERE tenant_id = :tenant_id
    AND facility_id = :facility_id
    AND time >= now() - INTERVAL '72 hours'
GROUP BY bucket, tenant_id, facility_id, zone_id
ORDER BY bucket;

-- Device health monitoring: find sensors with deteriorating battery
SELECT
    dh.device_id,
    hd.device_name,
    s.space_number,
    f.name AS facility_name,
    last_value(dh.battery_pct) OVER (
        PARTITION BY dh.device_id ORDER BY dh.time
    ) AS current_battery,
    first_value(dh.battery_pct) OVER (
        PARTITION BY dh.device_id ORDER BY dh.time
    ) AS battery_30d_ago,
    COUNT(*) FILTER (WHERE dh.status = 'error') AS error_count_30d
FROM ts_device_health dh
JOIN hardware_devices hd ON hd.device_id = dh.device_id
JOIN spaces s ON s.sensor_id = dh.device_id::TEXT
JOIN facilities f ON f.facility_id = dh.facility_id
WHERE dh.tenant_id = :tenant_id
    AND dh.time >= now() - INTERVAL '30 days'
GROUP BY dh.device_id, hd.device_name, s.space_number, f.name,
         dh.battery_pct, dh.time, dh.status;
```

---

## Data Flow Architecture

```
IoT Sensors / LPR Cameras / Gate Controllers / EV Chargers
    |
    v
+-------------------+      +-------------------+
| Ingestion Service | ---> |  Redis Streams     |  (real-time pub/sub)
| (MQTT / HTTP)     |      |  + State Updates   |
+--------+----------+      +---+---------------+
         |                      |
         |                      v
         |              +-------------------+
         |              | Signage / Mobile   |  (WebSocket consumers)
         |              | Dynamic Pricing    |
         |              +-------------------+
         |
         v
+-------------------+
| TimescaleDB       |  (time-series hypertables)
| ts_occupancy_*    |
| ts_lpr_reads      |
| ts_gate_events    |
| ts_pricing_*      |
| ts_ev_telemetry   |
| ts_device_health  |
+--------+----------+
         |
         | Continuous Aggregates (auto-maintained)
         v
+-------------------+
| 5min / Hourly /   |  (pre-computed rollups)
| Daily Aggregates  |
+--------+----------+
         |
         | JOINable with relational tables
         v
+-------------------+
| PostgreSQL Core   |  (relational business entities)
| tenants, spaces   |
| permits, sessions |
| citations, txns   |
+-------------------+
         |
         v
+-------------------+
| Reporting / BI    |
| ML Training       |
| API Layer         |
+-------------------+
```

---

## Pros and Cons

### Pros

1. **Purpose-built for each workload**: Relational data gets ACID transactions, foreign keys, and normalized integrity. Time-series data gets automatic partitioning, 10-20x compression, and continuous aggregates. Each workload uses the storage engine optimized for it.

2. **Single operational footprint**: Despite being "polyglot," TimescaleDB runs inside PostgreSQL. You manage one database, one backup strategy (with some TimescaleDB-specific considerations), one monitoring stack. The team needs PostgreSQL skills plus TimescaleDB knowledge, not an entirely separate database technology.

3. **Cross-domain JOINs**: Because hypertables are PostgreSQL tables, you can JOIN occupancy data with facility names, session durations with customer records, and LPR reads with permit status -- all in a single SQL query. This is impossible with truly separate databases (e.g., PostgreSQL + InfluxDB).

4. **Native compression**: TimescaleDB compression reduces storage for old sensor data by 10-20x without external archival processes. A year of occupancy data from 10,000 sensors compresses from ~500GB to ~30-50GB.

5. **Continuous aggregates for dashboards and ML**: Pre-computed 5-minute, hourly, and daily aggregates are maintained incrementally. Dashboard queries that would scan billions of raw readings instead hit compact aggregate tables in milliseconds.

6. **Built-in data lifecycle**: `add_retention_policy()` and `add_compression_policy()` automate the hot/warm/cold data lifecycle. Raw sensor data is compressed after 7 days and dropped after 90 days, while aggregates are retained for years.

7. **ML pipeline integration**: The pricing signals hypertable with enrichment features (weather, events, historical patterns) serves as a ready-made feature store for training demand forecasting and dynamic pricing models. Continuous aggregates provide the training data in the exact time-bucket format ML models need.

8. **Real-time + historical in one system**: Redis provides sub-millisecond real-time state for signage and enforcement. TimescaleDB provides historical analytics. PostgreSQL provides transactional integrity. Each concern is addressed by the right tool, but they integrate seamlessly.

### Cons

1. **TimescaleDB extension limitations**: TimescaleDB hypertables have some restrictions compared to regular PostgreSQL tables: no foreign keys pointing TO a hypertable (only FROM it), no unique constraints that do not include the partitioning column, and some DDL operations behave differently.

2. **Redis as a consistency risk**: Redis is an eventually-consistent cache layer. If Redis crashes and restarts, occupancy counts must be rebuilt from TimescaleDB. The application must handle Redis unavailability gracefully (fall back to querying TimescaleDB directly with slightly higher latency).

3. **Three systems to operate**: PostgreSQL + TimescaleDB + Redis is still three things to deploy, monitor, and maintain. Redis adds a failure mode and a data consistency concern that a pure-PostgreSQL architecture avoids.

4. **Compression tradeoffs**: Compressed TimescaleDB chunks are read-only. If you need to update a compressed LPR read (e.g., manual plate correction), you must decompress the chunk first, update, and recompress. This makes corrections to old data more expensive.

5. **Continuous aggregate limitations**: Continuous aggregates have some query restrictions (limited JOIN support, no HAVING, specific aggregate function requirements). Complex multi-table aggregates may need to be handled at the application layer.

6. **Licensing considerations**: TimescaleDB Community Edition is open-source (Apache 2.0), but some features (multi-node distributed hypertables, continuous aggregate real-time aggregation) require the Timescale License. Evaluate which features you need.

7. **Schema migration complexity**: Migrating hypertable schemas (adding columns, changing compression settings) requires specific TimescaleDB procedures that differ from standard ALTER TABLE. The migration tooling must account for this.

---

## Migration and Scaling Considerations

### Initial Deployment

- Single PostgreSQL 16 instance with TimescaleDB extension and PostGIS.
- Redis as a single node with AOF persistence for real-time state.
- Start with compression after 7 days on sensor data; tune based on query patterns.
- Estimated hardware: 8 vCPU, 32GB RAM, 1TB SSD for up to 50 facilities.

### Growth Path (50-200 facilities)

- Add a PostgreSQL read replica for analytics queries (both relational and time-series queries hit the replica).
- Increase Redis to a 3-node Sentinel setup for high availability.
- Add more continuous aggregates as dashboard requirements grow.
- Tune compression segment-by columns based on actual query patterns.

### Enterprise Scale (200-1000+ facilities)

- Evaluate Timescale Cloud for managed multi-node deployment with automatic chunk distribution.
- Shard relational data by tenant using Citus (which can coexist with TimescaleDB in the same instance).
- Upgrade Redis to a Redis Cluster for horizontal scaling.
- Consider a ClickHouse replica fed by CDC for heavy analytical workloads that benefit from columnar storage (e.g., multi-year revenue trend analysis across all tenants).

### Storage Projections

| Data Type | Raw Volume/Year (100 facilities) | After Compression | After Retention Policy |
|-----------|----------------------------------|-------------------|----------------------|
| Occupancy readings | ~500 GB | ~30 GB | Aggregates only after 90 days |
| LPR reads | ~100 GB | ~8 GB | Dropped after 1 year |
| Gate events | ~50 GB | ~4 GB | Dropped after 6 months |
| Pricing signals | ~20 GB | ~2 GB | Kept 2 years compressed |
| EV telemetry | ~80 GB | ~6 GB | Dropped after 1 year |
| Device health | ~30 GB | ~2 GB | Dropped after 90 days |
| Relational core | ~20 GB | N/A (no compression) | Archived after 7 years |
| **Total** | **~800 GB** | **~72 GB active** | Sustainable long-term |

---

## Summary

The Time-Series + Relational polyglot architecture is the technically strongest choice for a parking management system with AI-native features. The domain generates enormous volumes of time-stamped IoT data (sensors, cameras, gates, chargers) alongside traditional transactional data (permits, payments, citations). Rather than forcing both workloads into a single paradigm with compromises, this model uses TimescaleDB's hypertables, compression, and continuous aggregates for the time-series half while PostgreSQL handles the relational half -- all within a single database connection. Redis adds a real-time state layer for sub-millisecond signage and enforcement lookups. The tradeoff is modest operational complexity (TimescaleDB extension management, Redis availability) in exchange for dramatically better performance, storage efficiency, and ML pipeline integration for the IoT-heavy aspects of the platform.
