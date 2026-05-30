# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

## Overview

This model uses a fully normalized relational schema in PostgreSQL, following Third Normal Form (3NF) principles throughout. Every entity is decomposed into its own table with explicit foreign key relationships, enforcing referential integrity at the database level. This approach aligns naturally with the APDS / ISO TS 5206-1 data specification, which defines discrete data objects for places, rates, sessions, rights (permits), and observations (occupancy).

The schema is designed for multi-tenancy using a shared-schema, row-level security (RLS) approach with a `tenant_id` column on every table. This supports the platform's goal of allowing operators to manage multiple facilities or municipalities from a single instance.

---

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary Database | PostgreSQL 16+ | Mature, extensible, excellent spatial support via PostGIS, native RLS for multi-tenancy |
| Spatial Extension | PostGIS 3.4+ | Geospatial indexing for parking facility locations, zone boundaries, geofenced virtual permits |
| Connection Pooling | PgBouncer | Connection management for multi-tenant workloads |
| Migration Tool | Flyway or Liquibase | Version-controlled schema migrations |
| Caching Layer | Redis | Session tokens, real-time occupancy counts, rate limit counters |
| Search | PostgreSQL Full-Text Search | Permit lookups, citation searches, customer records |

---

## Core Schema

### Tenant and Organization

```sql
-- Multi-tenancy root: each operator (municipality, university, garage company) is a tenant
CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),
    billing_address TEXT,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    settings        JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'terminated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users who interact with the system (operators, enforcement officers, admins)
CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    phone           VARCHAR(50),
    role            VARCHAR(30) NOT NULL DEFAULT 'operator'
                    CHECK (role IN ('super_admin', 'tenant_admin', 'operator',
                                    'enforcement_officer', 'customer_service', 'auditor')),
    sso_provider    VARCHAR(50),
    sso_external_id VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'locked')),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
```

### Facility and Space Inventory

```sql
-- A parking facility (garage, surface lot, on-street zone, campus area)
CREATE TABLE facilities (
    facility_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    facility_type   VARCHAR(30) NOT NULL
                    CHECK (facility_type IN ('garage', 'surface_lot', 'on_street',
                                             'underground', 'mixed_use')),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),
    location        GEOGRAPHY(POINT, 4326),  -- PostGIS point for lat/lng
    boundary        GEOGRAPHY(POLYGON, 4326), -- PostGIS polygon for facility boundary
    total_spaces    INTEGER NOT NULL DEFAULT 0,
    operating_hours JSONB,  -- structured operating schedule
    contact_phone   VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'maintenance', 'closed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_facilities_tenant ON facilities(tenant_id);
CREATE INDEX idx_facilities_location ON facilities USING GIST(location);

-- Levels within a facility (floors in a garage, sections in a surface lot)
CREATE TABLE levels (
    level_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(100) NOT NULL,  -- e.g., "Level 1", "Basement", "Roof"
    sort_order      INTEGER NOT NULL DEFAULT 0,
    total_spaces    INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_levels_facility ON levels(facility_id);

-- Zones within a level or facility (used for enforcement, pricing, signage grouping)
CREATE TABLE zones (
    zone_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    level_id        UUID REFERENCES levels(level_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(100) NOT NULL,
    zone_code       VARCHAR(20),
    boundary        GEOGRAPHY(POLYGON, 4326),
    total_spaces    INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_zones_facility ON zones(facility_id);
CREATE INDEX idx_zones_level ON zones(level_id);

-- Individual parking spaces
CREATE TABLE spaces (
    space_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    level_id        UUID REFERENCES levels(level_id),
    zone_id         UUID REFERENCES zones(zone_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    space_number    VARCHAR(20) NOT NULL,
    space_type      VARCHAR(30) NOT NULL DEFAULT 'standard'
                    CHECK (space_type IN ('standard', 'compact', 'accessible', 'ev_charging',
                                          'motorcycle', 'oversized', 'reserved', 'loading',
                                          'car_share', 'valet')),
    location        GEOGRAPHY(POINT, 4326),
    has_sensor      BOOLEAN NOT NULL DEFAULT false,
    sensor_id       VARCHAR(100),
    is_covered      BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'available'
                    CHECK (status IN ('available', 'occupied', 'reserved', 'out_of_service',
                                      'maintenance')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (facility_id, space_number)
);

CREATE INDEX idx_spaces_facility ON spaces(facility_id);
CREATE INDEX idx_spaces_zone ON spaces(zone_id);
CREATE INDEX idx_spaces_type ON spaces(space_type);
CREATE INDEX idx_spaces_status ON spaces(status);
CREATE INDEX idx_spaces_sensor ON spaces(sensor_id) WHERE sensor_id IS NOT NULL;
```

### Vehicles and Customers

```sql
-- Drivers / customers who use parking services
CREATE TABLE customers (
    customer_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    company_name    VARCHAR(255),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),
    customer_type   VARCHAR(30) NOT NULL DEFAULT 'individual'
                    CHECK (customer_type IN ('individual', 'corporate', 'government',
                                             'educational', 'residential')),
    identity_verified BOOLEAN NOT NULL DEFAULT false,
    sso_provider    VARCHAR(50),
    sso_external_id VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'suspended', 'banned')),
    balance         NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_email ON customers(email);

-- Vehicles registered to customers
CREATE TABLE vehicles (
    vehicle_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(customer_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    license_plate   VARCHAR(20) NOT NULL,
    plate_state     VARCHAR(10),
    plate_country   CHAR(2) NOT NULL DEFAULT 'US',
    make            VARCHAR(50),
    model           VARCHAR(50),
    year            SMALLINT,
    color           VARCHAR(30),
    vehicle_type    VARCHAR(20) DEFAULT 'sedan'
                    CHECK (vehicle_type IN ('sedan', 'suv', 'truck', 'van', 'motorcycle',
                                            'ev', 'hybrid', 'compact', 'oversized')),
    is_ev           BOOLEAN NOT NULL DEFAULT false,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'stolen', 'flagged')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicles_customer ON vehicles(customer_id);
CREATE INDEX idx_vehicles_plate ON vehicles(license_plate, plate_state, plate_country);
CREATE INDEX idx_vehicles_tenant ON vehicles(tenant_id);
```

### Permit Management

```sql
-- Permit types configured by the operator
CREATE TABLE permit_types (
    permit_type_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    description     TEXT,
    category        VARCHAR(30) NOT NULL
                    CHECK (category IN ('resident', 'employee', 'student', 'visitor',
                                        'guest', 'temporary', 'event', 'contractor',
                                        'accessible', 'ev', 'monthly', 'annual')),
    price           NUMERIC(10, 2),
    billing_cycle   VARCHAR(20) CHECK (billing_cycle IN ('one_time', 'daily', 'weekly',
                                                          'monthly', 'quarterly', 'annual')),
    duration_days   INTEGER,
    max_vehicles    INTEGER NOT NULL DEFAULT 1,
    max_active      INTEGER,  -- max active permits of this type (NULL = unlimited)
    is_waitlisted   BOOLEAN NOT NULL DEFAULT false,
    requires_verification BOOLEAN NOT NULL DEFAULT false,
    eligible_zones  UUID[],  -- array of zone_ids where this permit is valid
    eligible_facilities UUID[],  -- array of facility_ids
    valid_days      SMALLINT[] DEFAULT '{1,2,3,4,5}', -- 0=Sun, 6=Sat
    valid_start_time TIME,
    valid_end_time  TIME,
    is_virtual      BOOLEAN NOT NULL DEFAULT true,  -- geofence-based, no physical decal
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'archived')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_permit_types_tenant ON permit_types(tenant_id);

-- Issued permits
CREATE TABLE permits (
    permit_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    permit_type_id  UUID NOT NULL REFERENCES permit_types(permit_type_id),
    customer_id     UUID NOT NULL REFERENCES customers(customer_id),
    permit_number   VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'active', 'expired', 'revoked',
                                      'suspended', 'waitlisted')),
    issued_at       TIMESTAMPTZ,
    effective_from  TIMESTAMPTZ NOT NULL,
    effective_to    TIMESTAMPTZ,
    renewed_from    UUID REFERENCES permits(permit_id),  -- link to prior permit if renewal
    issued_by       UUID REFERENCES users(user_id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, permit_number)
);

CREATE INDEX idx_permits_tenant ON permits(tenant_id);
CREATE INDEX idx_permits_customer ON permits(customer_id);
CREATE INDEX idx_permits_type ON permits(permit_type_id);
CREATE INDEX idx_permits_status ON permits(status);
CREATE INDEX idx_permits_effective ON permits(effective_from, effective_to);

-- Vehicles associated with a permit
CREATE TABLE permit_vehicles (
    permit_vehicle_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id       UUID NOT NULL REFERENCES permits(permit_id),
    vehicle_id      UUID NOT NULL REFERENCES vehicles(vehicle_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    removed_at      TIMESTAMPTZ,
    UNIQUE (permit_id, vehicle_id)
);

CREATE INDEX idx_permit_vehicles_permit ON permit_vehicles(permit_id);
CREATE INDEX idx_permit_vehicles_vehicle ON permit_vehicles(vehicle_id);

-- Waitlist for oversubscribed permit types
CREATE TABLE permit_waitlist (
    waitlist_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    permit_type_id  UUID NOT NULL REFERENCES permit_types(permit_type_id),
    customer_id     UUID NOT NULL REFERENCES customers(customer_id),
    position        INTEGER NOT NULL,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    notified_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'waiting'
                    CHECK (status IN ('waiting', 'offered', 'accepted', 'expired', 'cancelled')),
    UNIQUE (permit_type_id, customer_id)
);

CREATE INDEX idx_waitlist_type ON permit_waitlist(permit_type_id, position);
```

### Parking Sessions and Reservations

```sql
-- Active and historical parking sessions
CREATE TABLE parking_sessions (
    session_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    space_id        UUID REFERENCES spaces(space_id),
    zone_id         UUID REFERENCES zones(zone_id),
    customer_id     UUID REFERENCES customers(customer_id),
    vehicle_id      UUID REFERENCES vehicles(vehicle_id),
    license_plate   VARCHAR(20),  -- denormalized for LPR-initiated sessions
    session_type    VARCHAR(20) NOT NULL DEFAULT 'transient'
                    CHECK (session_type IN ('transient', 'reserved', 'permit',
                                            'event', 'monthly', 'validation')),
    entry_time      TIMESTAMPTZ NOT NULL,
    exit_time       TIMESTAMPTZ,
    expected_exit   TIMESTAMPTZ,
    duration_minutes INTEGER GENERATED ALWAYS AS (
        EXTRACT(EPOCH FROM (COALESCE(exit_time, now()) - entry_time)) / 60
    ) STORED,
    entry_method    VARCHAR(20)
                    CHECK (entry_method IN ('lpr', 'ticket', 'mobile', 'gate_card',
                                            'permit', 'validation', 'manual')),
    exit_method     VARCHAR(20)
                    CHECK (exit_method IN ('lpr', 'ticket', 'mobile', 'gate_card',
                                           'permit', 'validation', 'manual')),
    entry_lane_id   UUID,
    exit_lane_id    UUID,
    permit_id       UUID REFERENCES permits(permit_id),
    reservation_id  UUID,  -- FK added after reservations table
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'completed', 'cancelled', 'overstay',
                                      'forced_exit')),
    amount_due      NUMERIC(10, 2) DEFAULT 0.00,
    amount_paid     NUMERIC(10, 2) DEFAULT 0.00,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_tenant ON parking_sessions(tenant_id);
CREATE INDEX idx_sessions_facility ON parking_sessions(facility_id);
CREATE INDEX idx_sessions_customer ON parking_sessions(customer_id);
CREATE INDEX idx_sessions_plate ON parking_sessions(license_plate);
CREATE INDEX idx_sessions_status ON parking_sessions(status);
CREATE INDEX idx_sessions_entry ON parking_sessions(entry_time);
CREATE INDEX idx_sessions_active ON parking_sessions(facility_id, status)
    WHERE status = 'active';

-- Reservations (pre-booked parking)
CREATE TABLE reservations (
    reservation_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    space_id        UUID REFERENCES spaces(space_id),
    zone_id         UUID REFERENCES zones(zone_id),
    customer_id     UUID NOT NULL REFERENCES customers(customer_id),
    vehicle_id      UUID REFERENCES vehicles(vehicle_id),
    reservation_type VARCHAR(20) NOT NULL DEFAULT 'single'
                    CHECK (reservation_type IN ('single', 'recurring', 'event',
                                                 'monthly_contract')),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    recurrence_rule TEXT,  -- iCal RRULE for recurring reservations
    status          VARCHAR(20) NOT NULL DEFAULT 'confirmed'
                    CHECK (status IN ('pending', 'confirmed', 'checked_in', 'completed',
                                      'cancelled', 'no_show')),
    amount          NUMERIC(10, 2) NOT NULL DEFAULT 0.00,
    confirmation_code VARCHAR(20) NOT NULL,
    qr_code_data    TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, confirmation_code)
);

ALTER TABLE parking_sessions
    ADD CONSTRAINT fk_sessions_reservation
    FOREIGN KEY (reservation_id) REFERENCES reservations(reservation_id);

CREATE INDEX idx_reservations_tenant ON reservations(tenant_id);
CREATE INDEX idx_reservations_facility ON reservations(facility_id);
CREATE INDEX idx_reservations_customer ON reservations(customer_id);
CREATE INDEX idx_reservations_time ON reservations(start_time, end_time);
CREATE INDEX idx_reservations_status ON reservations(status);
```

### Pricing and Rate Management

```sql
-- Rate schedules define pricing structures
CREATE TABLE rate_schedules (
    rate_schedule_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID REFERENCES facilities(facility_id),
    zone_id         UUID REFERENCES zones(zone_id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    rate_type       VARCHAR(20) NOT NULL
                    CHECK (rate_type IN ('hourly', 'flat', 'tiered', 'daily_max',
                                          'event', 'dynamic', 'validation')),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    priority        INTEGER NOT NULL DEFAULT 0,  -- higher priority overrides lower
    effective_from  TIMESTAMPTZ NOT NULL,
    effective_to    TIMESTAMPTZ,
    applicable_days SMALLINT[] DEFAULT '{0,1,2,3,4,5,6}',
    applicable_start_time TIME,
    applicable_end_time TIME,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'draft', 'archived')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rate_schedules_tenant ON rate_schedules(tenant_id);
CREATE INDEX idx_rate_schedules_facility ON rate_schedules(facility_id);

-- Individual rate tiers within a schedule
CREATE TABLE rate_tiers (
    rate_tier_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_schedule_id UUID NOT NULL REFERENCES rate_schedules(rate_schedule_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    from_minutes    INTEGER NOT NULL DEFAULT 0,
    to_minutes      INTEGER,  -- NULL = unlimited
    rate_amount     NUMERIC(10, 2) NOT NULL,
    rate_unit       VARCHAR(20) NOT NULL DEFAULT 'per_hour'
                    CHECK (rate_unit IN ('per_minute', 'per_hour', 'per_day',
                                          'flat', 'per_entry')),
    daily_max       NUMERIC(10, 2),
    weekly_max      NUMERIC(10, 2),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rate_tiers_schedule ON rate_tiers(rate_schedule_id);

-- Dynamic pricing adjustments (AI-driven price modifications)
CREATE TABLE dynamic_pricing_adjustments (
    adjustment_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    zone_id         UUID REFERENCES zones(zone_id),
    rate_schedule_id UUID REFERENCES rate_schedules(rate_schedule_id),
    adjustment_type VARCHAR(30) NOT NULL
                    CHECK (adjustment_type IN ('demand_surge', 'demand_reduction',
                                                'event', 'weather', 'time_of_day',
                                                'promotional', 'manual_override')),
    multiplier      NUMERIC(5, 3) NOT NULL DEFAULT 1.000,  -- e.g., 1.500 = 50% surcharge
    absolute_adjustment NUMERIC(10, 2) DEFAULT 0.00,
    reason          TEXT,
    occupancy_threshold NUMERIC(5, 2),  -- occupancy % that triggered this
    effective_from  TIMESTAMPTZ NOT NULL,
    effective_to    TIMESTAMPTZ,
    approved_by     UUID REFERENCES users(user_id),
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    ml_model_version VARCHAR(50),
    confidence_score NUMERIC(5, 4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dynamic_pricing_facility ON dynamic_pricing_adjustments(facility_id);
CREATE INDEX idx_dynamic_pricing_effective ON dynamic_pricing_adjustments(effective_from, effective_to);

-- Validation codes (employer subsidies, merchant validations)
CREATE TABLE validation_programs (
    validation_program_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(100) NOT NULL,
    sponsor_name    VARCHAR(255),  -- employer or merchant name
    discount_type   VARCHAR(20) NOT NULL
                    CHECK (discount_type IN ('percentage', 'fixed_amount', 'free_hours',
                                              'flat_rate', 'full_validation')),
    discount_value  NUMERIC(10, 2) NOT NULL,
    max_hours       INTEGER,
    budget_limit    NUMERIC(12, 2),
    budget_used     NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    valid_from      TIMESTAMPTZ NOT NULL,
    valid_to        TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE validation_codes (
    validation_code_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    validation_program_id UUID NOT NULL REFERENCES validation_programs(validation_program_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    code            VARCHAR(20) NOT NULL,
    used_at         TIMESTAMPTZ,
    used_by_session UUID REFERENCES parking_sessions(session_id),
    expires_at      TIMESTAMPTZ,
    UNIQUE (tenant_id, code)
);
```

### Payment Processing

```sql
-- Payment transactions (PCI-DSS compliant: NO raw card data stored)
CREATE TABLE payment_transactions (
    transaction_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    customer_id     UUID REFERENCES customers(customer_id),
    session_id      UUID REFERENCES parking_sessions(session_id),
    reservation_id  UUID REFERENCES reservations(reservation_id),
    permit_id       UUID REFERENCES permits(permit_id),
    citation_id     UUID,  -- FK added after citations table
    transaction_type VARCHAR(20) NOT NULL
                    CHECK (transaction_type IN ('charge', 'refund', 'void',
                                                 'adjustment', 'payout')),
    payment_method  VARCHAR(20) NOT NULL
                    CHECK (payment_method IN ('credit_card', 'debit_card', 'mobile_pay',
                                              'pay_by_plate', 'cash', 'account_balance',
                                              'contactless', 'invoice', 'validation')),
    amount          NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    tax_amount      NUMERIC(10, 2) DEFAULT 0.00,
    -- PCI-DSS: only tokenized references, never raw card data
    payment_token   VARCHAR(255),  -- token from payment gateway
    gateway         VARCHAR(50),   -- stripe, square, adyen, etc.
    gateway_txn_id  VARCHAR(255),
    card_last_four  CHAR(4),
    card_brand      VARCHAR(20),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'completed', 'failed',
                                      'refunded', 'disputed', 'voided')),
    failure_reason  TEXT,
    receipt_url     TEXT,
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transactions_tenant ON payment_transactions(tenant_id);
CREATE INDEX idx_transactions_customer ON payment_transactions(customer_id);
CREATE INDEX idx_transactions_session ON payment_transactions(session_id);
CREATE INDEX idx_transactions_status ON payment_transactions(status);
CREATE INDEX idx_transactions_date ON payment_transactions(created_at);

-- Stored payment methods (tokenized only)
CREATE TABLE payment_methods (
    payment_method_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(customer_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    method_type     VARCHAR(20) NOT NULL
                    CHECK (method_type IN ('credit_card', 'debit_card', 'bank_account',
                                            'mobile_wallet')),
    gateway         VARCHAR(50) NOT NULL,
    gateway_customer_id VARCHAR(255),
    payment_token   VARCHAR(255) NOT NULL,  -- tokenized card reference
    card_last_four  CHAR(4),
    card_brand      VARCHAR(20),
    expiry_month    SMALLINT,
    expiry_year     SMALLINT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'expired', 'removed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_methods_customer ON payment_methods(customer_id);
```

### Enforcement and Violations

```sql
-- Citations (parking tickets / violations)
CREATE TABLE citations (
    citation_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    zone_id         UUID REFERENCES zones(zone_id),
    space_id        UUID REFERENCES spaces(space_id),
    citation_number VARCHAR(50) NOT NULL,
    license_plate   VARCHAR(20) NOT NULL,
    plate_state     VARCHAR(10),
    plate_country   CHAR(2) DEFAULT 'US',
    vehicle_id      UUID REFERENCES vehicles(vehicle_id),
    customer_id     UUID REFERENCES customers(customer_id),
    violation_type  VARCHAR(50) NOT NULL
                    CHECK (violation_type IN ('expired_meter', 'no_permit', 'wrong_zone',
                                              'overtime', 'no_payment', 'fire_lane',
                                              'accessible_violation', 'double_parking',
                                              'loading_zone', 'expired_registration',
                                              'other')),
    fine_amount     NUMERIC(10, 2) NOT NULL,
    late_fee        NUMERIC(10, 2) DEFAULT 0.00,
    total_due       NUMERIC(10, 2) GENERATED ALWAYS AS (fine_amount + late_fee) STORED,
    issued_by       UUID REFERENCES users(user_id),
    issued_at       TIMESTAMPTZ NOT NULL,
    location_description TEXT,
    location_point  GEOGRAPHY(POINT, 4326),
    status          VARCHAR(20) NOT NULL DEFAULT 'issued'
                    CHECK (status IN ('issued', 'mailed', 'paid', 'partially_paid',
                                      'appealed', 'appeal_approved', 'appeal_denied',
                                      'dismissed', 'escalated', 'collections',
                                      'void', 'adjudicated')),
    is_lpr_generated BOOLEAN NOT NULL DEFAULT false,
    lpr_confidence  NUMERIC(5, 4),
    auto_adjudicated BOOLEAN NOT NULL DEFAULT false,
    due_date        DATE,
    escalation_date DATE,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, citation_number)
);

ALTER TABLE payment_transactions
    ADD CONSTRAINT fk_transactions_citation
    FOREIGN KEY (citation_id) REFERENCES citations(citation_id);

CREATE INDEX idx_citations_tenant ON citations(tenant_id);
CREATE INDEX idx_citations_plate ON citations(license_plate);
CREATE INDEX idx_citations_status ON citations(status);
CREATE INDEX idx_citations_issued ON citations(issued_at);
CREATE INDEX idx_citations_facility ON citations(facility_id);

-- Citation evidence (photos, LPR scans)
CREATE TABLE citation_evidence (
    evidence_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citation_id     UUID NOT NULL REFERENCES citations(citation_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    evidence_type   VARCHAR(20) NOT NULL
                    CHECK (evidence_type IN ('photo', 'lpr_scan', 'video', 'document',
                                              'gps_log', 'sensor_reading')),
    file_url        TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(50),
    description     TEXT,
    captured_at     TIMESTAMPTZ NOT NULL,
    captured_by     UUID REFERENCES users(user_id),
    metadata        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_evidence_citation ON citation_evidence(citation_id);

-- Appeals
CREATE TABLE citation_appeals (
    appeal_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citation_id     UUID NOT NULL REFERENCES citations(citation_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    customer_id     UUID REFERENCES customers(customer_id),
    appeal_reason   VARCHAR(50) NOT NULL
                    CHECK (appeal_reason IN ('incorrect_plate', 'valid_permit', 'paid_meter',
                                              'vehicle_sold', 'medical_emergency',
                                              'signage_unclear', 'meter_malfunction',
                                              'first_offense', 'other')),
    statement       TEXT NOT NULL,
    supporting_docs TEXT[],  -- array of file URLs
    status          VARCHAR(20) NOT NULL DEFAULT 'submitted'
                    CHECK (status IN ('submitted', 'under_review', 'approved', 'denied',
                                      'escalated', 'withdrawn')),
    reviewed_by     UUID REFERENCES users(user_id),
    review_notes    TEXT,
    decision_at     TIMESTAMPTZ,
    auto_reviewed   BOOLEAN NOT NULL DEFAULT false,
    ai_recommendation VARCHAR(20),
    ai_confidence   NUMERIC(5, 4),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appeals_citation ON citation_appeals(citation_id);
CREATE INDEX idx_appeals_status ON citation_appeals(status);

-- Escalation actions (tow, boot/immobilize, collections)
CREATE TABLE enforcement_actions (
    action_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citation_id     UUID NOT NULL REFERENCES citations(citation_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    action_type     VARCHAR(30) NOT NULL
                    CHECK (action_type IN ('warning', 'boot', 'tow', 'collections',
                                            'license_hold', 'registration_hold')),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'in_progress', 'completed', 'cancelled')),
    assigned_to     UUID REFERENCES users(user_id),
    vendor_name     VARCHAR(255),
    vendor_reference VARCHAR(100),
    cost            NUMERIC(10, 2),
    notes           TEXT,
    executed_at     TIMESTAMPTZ,
    released_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_actions_citation ON enforcement_actions(citation_id);
```

### LPR and Sensor Data

```sql
-- LPR camera reads (high-volume table, partitioned by date)
CREATE TABLE lpr_reads (
    lpr_read_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    camera_id       VARCHAR(100) NOT NULL,
    camera_location VARCHAR(50),  -- 'entry', 'exit', 'enforcement_mobile', 'zone_patrol'
    license_plate   VARCHAR(20) NOT NULL,
    plate_state     VARCHAR(10),
    plate_country   CHAR(2) DEFAULT 'US',
    confidence_score NUMERIC(5, 4) NOT NULL,
    image_url       TEXT,
    thumbnail_url   TEXT,
    direction       VARCHAR(10) CHECK (direction IN ('entry', 'exit', 'patrol')),
    matched_vehicle_id UUID REFERENCES vehicles(vehicle_id),
    matched_session_id UUID REFERENCES parking_sessions(session_id),
    requires_review BOOLEAN NOT NULL DEFAULT false,
    reviewed_by     UUID REFERENCES users(user_id),
    reviewed_at     TIMESTAMPTZ,
    corrected_plate VARCHAR(20),
    read_at         TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (read_at);

-- Create monthly partitions
CREATE TABLE lpr_reads_2026_01 PARTITION OF lpr_reads
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE lpr_reads_2026_02 PARTITION OF lpr_reads
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional monthly partitions created via automation

CREATE INDEX idx_lpr_reads_plate ON lpr_reads(license_plate);
CREATE INDEX idx_lpr_reads_camera ON lpr_reads(camera_id, read_at);
CREATE INDEX idx_lpr_reads_review ON lpr_reads(requires_review) WHERE requires_review = true;
CREATE INDEX idx_lpr_reads_facility ON lpr_reads(facility_id, read_at);

-- Occupancy sensor events
CREATE TABLE occupancy_readings (
    reading_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    space_id        UUID REFERENCES spaces(space_id),
    zone_id         UUID REFERENCES zones(zone_id),
    sensor_id       VARCHAR(100) NOT NULL,
    is_occupied     BOOLEAN NOT NULL,
    battery_level   SMALLINT,
    signal_strength SMALLINT,
    read_at         TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (read_at);

CREATE INDEX idx_occupancy_facility ON occupancy_readings(facility_id, read_at);
CREATE INDEX idx_occupancy_space ON occupancy_readings(space_id, read_at);
CREATE INDEX idx_occupancy_sensor ON occupancy_readings(sensor_id, read_at);

-- Occupancy snapshots (aggregated counts, updated in near-real-time)
CREATE TABLE occupancy_snapshots (
    snapshot_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    zone_id         UUID REFERENCES zones(zone_id),
    level_id        UUID REFERENCES levels(level_id),
    total_spaces    INTEGER NOT NULL,
    occupied_spaces INTEGER NOT NULL,
    available_spaces INTEGER GENERATED ALWAYS AS (total_spaces - occupied_spaces) STORED,
    occupancy_pct   NUMERIC(5, 2) GENERATED ALWAYS AS (
        CASE WHEN total_spaces > 0 THEN (occupied_spaces::NUMERIC / total_spaces * 100)
             ELSE 0 END
    ) STORED,
    snapshot_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshots_facility ON occupancy_snapshots(facility_id, snapshot_at);
```

### EV Charging Integration

```sql
-- EV charging stations (aligned with OCPI Location/EVSE/Connector hierarchy)
CREATE TABLE ev_stations (
    ev_station_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    space_id        UUID REFERENCES spaces(space_id),
    ocpi_evse_uid   VARCHAR(100),  -- OCPI EVSE unique identifier
    station_name    VARCHAR(100),
    manufacturer    VARCHAR(100),
    model           VARCHAR(100),
    serial_number   VARCHAR(100),
    max_power_kw    NUMERIC(8, 2),
    connector_type  VARCHAR(30)
                    CHECK (connector_type IN ('type1', 'type2', 'ccs1', 'ccs2',
                                               'chademo', 'tesla', 'nacs')),
    status          VARCHAR(20) NOT NULL DEFAULT 'available'
                    CHECK (status IN ('available', 'charging', 'reserved', 'out_of_service',
                                      'maintenance')),
    ocpi_status     VARCHAR(20),
    last_heartbeat  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ev_stations_facility ON ev_stations(facility_id);
CREATE INDEX idx_ev_stations_space ON ev_stations(space_id);

-- EV charging sessions
CREATE TABLE ev_charging_sessions (
    charging_session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    ev_station_id   UUID NOT NULL REFERENCES ev_stations(ev_station_id),
    parking_session_id UUID REFERENCES parking_sessions(session_id),
    customer_id     UUID REFERENCES customers(customer_id),
    vehicle_id      UUID REFERENCES vehicles(vehicle_id),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ,
    energy_kwh      NUMERIC(10, 3) DEFAULT 0.000,
    max_power_kw    NUMERIC(8, 2),
    cost            NUMERIC(10, 2) DEFAULT 0.00,
    pricing_model   VARCHAR(20)
                    CHECK (pricing_model IN ('per_kwh', 'per_minute', 'flat', 'free',
                                              'included_in_parking')),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'completed', 'stopped', 'error')),
    ocpi_session_id VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ev_sessions_station ON ev_charging_sessions(ev_station_id);
CREATE INDEX idx_ev_sessions_parking ON ev_charging_sessions(parking_session_id);
```

### PARCS Hardware Integration

```sql
-- PARCS hardware devices (gates, barriers, pay stations, signage)
CREATE TABLE hardware_devices (
    device_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    device_type     VARCHAR(30) NOT NULL
                    CHECK (device_type IN ('entry_gate', 'exit_gate', 'pay_station',
                                            'barrier', 'display_sign', 'lpr_camera',
                                            'occupancy_sensor', 'intercom', 'ev_charger')),
    device_name     VARCHAR(100) NOT NULL,
    manufacturer    VARCHAR(100),
    model           VARCHAR(100),
    serial_number   VARCHAR(100),
    firmware_version VARCHAR(50),
    ip_address      INET,
    mac_address     MACADDR,
    location_description VARCHAR(255),
    lane_id         UUID,
    status          VARCHAR(20) NOT NULL DEFAULT 'online'
                    CHECK (status IN ('online', 'offline', 'maintenance', 'error',
                                      'decommissioned')),
    last_heartbeat  TIMESTAMPTZ,
    configuration   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_devices_facility ON hardware_devices(facility_id);
CREATE INDEX idx_devices_type ON hardware_devices(device_type);
CREATE INDEX idx_devices_status ON hardware_devices(status);

-- Lanes (entry/exit lanes with associated devices)
CREATE TABLE lanes (
    lane_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    lane_name       VARCHAR(100) NOT NULL,
    lane_type       VARCHAR(20) NOT NULL
                    CHECK (lane_type IN ('entry', 'exit', 'bidirectional', 'pedestrian')),
    is_accessible   BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'closed', 'maintenance')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lanes_facility ON lanes(facility_id);
```

### Digital Signage and Notifications

```sql
-- Digital signage displays
CREATE TABLE signage_displays (
    display_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    device_id       UUID REFERENCES hardware_devices(device_id),
    display_name    VARCHAR(100) NOT NULL,
    display_type    VARCHAR(20) NOT NULL
                    CHECK (display_type IN ('entrance', 'level', 'zone', 'wayfinding',
                                            'rate_display', 'full_sign')),
    shows_count_for UUID[],  -- zone_ids or level_ids to aggregate
    content_template JSONB,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit log (tracks all significant system actions)
CREATE TABLE audit_log (
    audit_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    user_id         UUID REFERENCES users(user_id),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
```

### Row-Level Security for Multi-Tenancy

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE facilities ENABLE ROW LEVEL SECURITY;
ALTER TABLE levels ENABLE ROW LEVEL SECURITY;
ALTER TABLE zones ENABLE ROW LEVEL SECURITY;
ALTER TABLE spaces ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE vehicles ENABLE ROW LEVEL SECURITY;
ALTER TABLE permit_types ENABLE ROW LEVEL SECURITY;
ALTER TABLE permits ENABLE ROW LEVEL SECURITY;
ALTER TABLE parking_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE reservations ENABLE ROW LEVEL SECURITY;
ALTER TABLE rate_schedules ENABLE ROW LEVEL SECURITY;
ALTER TABLE citations ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE lpr_reads ENABLE ROW LEVEL SECURITY;

-- Example policy (applied to each table)
CREATE POLICY tenant_isolation ON facilities
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

CREATE POLICY tenant_isolation ON parking_sessions
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

-- Repeat for all tenant-scoped tables...
```

---

## Entity Relationship Summary

```
tenants
  |-- facilities
  |     |-- levels
  |     |-- zones
  |     |-- spaces
  |     |-- lanes
  |     |-- hardware_devices
  |     |-- signage_displays
  |     |-- ev_stations
  |     |-- occupancy_snapshots
  |     |-- lpr_reads
  |
  |-- customers
  |     |-- vehicles
  |     |-- payment_methods
  |     |-- permits --> permit_types
  |     |     |-- permit_vehicles --> vehicles
  |     |
  |     |-- reservations --> facilities, spaces
  |     |-- parking_sessions --> facilities, spaces, permits
  |     |     |-- ev_charging_sessions --> ev_stations
  |     |
  |     |-- payment_transactions --> sessions, permits, citations
  |
  |-- citations --> facilities, vehicles
  |     |-- citation_evidence
  |     |-- citation_appeals
  |     |-- enforcement_actions
  |
  |-- rate_schedules --> facilities, zones
  |     |-- rate_tiers
  |     |-- dynamic_pricing_adjustments
  |
  |-- validation_programs
  |     |-- validation_codes
  |
  |-- users
  |-- audit_log
```

---

## Pros and Cons

### Pros

1. **Strong data integrity**: Foreign keys and check constraints prevent invalid states. A citation cannot reference a nonexistent facility. A permit must reference a valid permit type. Orphaned records are structurally impossible.

2. **Standards alignment**: The entity structure maps directly to APDS / ISO TS 5206-1 concepts (Place = facility/zone/space, Right = permit, Session = parking_session, Observation = occupancy_reading, Rate = rate_schedule/rate_tier), making data exchange with compliant systems straightforward.

3. **Mature tooling ecosystem**: PostgreSQL has decades of operational tooling -- backup (pg_dump, pgBackRest), monitoring (pg_stat_statements, pganalyze), migration (Flyway, Liquibase), ORMs (every language), and hosting options (RDS, Cloud SQL, Crunchy Bridge, Neon).

4. **Multi-tenancy via RLS**: Row-level security provides tenant isolation without schema-per-tenant complexity. A single connection pool serves all tenants. Adding a new tenant is an INSERT, not a DDL migration.

5. **ACID transactions**: Payment processing, citation issuance, and permit state changes all benefit from transactional guarantees. Partial failures (charge succeeded but session not updated) are eliminated.

6. **Rich query capabilities**: PostgreSQL's window functions, CTEs, and lateral joins support the complex analytics needed for revenue reporting, occupancy trending, and enforcement metrics without requiring a separate analytics database.

7. **PostGIS for spatial queries**: Native geospatial support enables "find nearest available space", geofenced virtual permits, and enforcement patrol route optimization without an external GIS system.

### Cons

1. **Schema rigidity**: Adding a new space attribute or permit rule requires an ALTER TABLE migration. In a live system with millions of sessions, some ALTER TABLE operations require careful planning to avoid locking.

2. **High-volume sensor data overhead**: The fully normalized `occupancy_readings` table will accumulate billions of rows from continuous sensor feeds. Partitioning helps, but pure relational storage is less efficient than purpose-built time-series databases for this workload.

3. **Complex queries for reporting**: Revenue reports that span sessions, payments, validations, and dynamic pricing adjustments require multi-table joins that can be slow without materialized views or pre-computed aggregates.

4. **LPR read volume**: At scale (thousands of cameras, reads every few seconds), the `lpr_reads` table can grow to hundreds of millions of rows per month. Partitioning is essential, but archival and purge policies need careful automation.

5. **No built-in event history**: The schema captures current state, not the history of state transitions. To audit "why was this citation dismissed?", you need the separate audit_log table, which requires application-level discipline to populate consistently.

6. **Horizontal scaling limitations**: PostgreSQL scales vertically well but horizontal sharding is complex. For very large multi-tenant deployments (thousands of municipalities), you may eventually need Citus or a shard-per-region strategy.

---

## Migration and Scaling Considerations

### Initial Deployment

- Start with a single PostgreSQL 16 instance with 8 vCPUs, 32GB RAM, 500GB SSD for operators managing up to ~50 facilities and ~10,000 active sessions/day.
- Enable `pg_partman` for automated partition management on `lpr_reads`, `occupancy_readings`, and `audit_log`.
- Set up streaming replication to a read replica for reporting queries.

### Growth Path (10,000-100,000 sessions/day)

- Add a dedicated read replica for analytics/reporting.
- Implement materialized views for frequently-accessed aggregates (daily revenue, hourly occupancy).
- Move LPR images and citation photos to object storage (S3/GCS) with only URLs in the database.
- Consider TimescaleDB extension for `occupancy_readings` to gain hypertable compression and continuous aggregates without leaving PostgreSQL.

### Enterprise Scale (100,000+ sessions/day, 500+ facilities)

- Evaluate Citus for distributed PostgreSQL with sharding on `tenant_id`.
- Implement connection pooling with PgBouncer (or pgcat) to handle thousands of concurrent connections.
- Archive historical sessions and LPR reads older than 2 years to cold storage with a query-on-demand pattern.
- Consider separating the payment service into its own database for PCI-DSS scope reduction.

### Data Retention Policy

| Data Type | Hot Storage | Warm Storage | Cold/Archive |
|-----------|-------------|--------------|--------------|
| Active sessions | Current | N/A | N/A |
| Completed sessions | 90 days | 2 years | 7 years |
| LPR reads | 30 days | 1 year | 3 years |
| Occupancy readings | 7 days | 90 days | 1 year |
| Citations | Until resolved + 2 years | 5 years | 10 years |
| Payment transactions | 1 year | 3 years | 7 years (PCI) |
| Audit log | 90 days | 2 years | 7 years |

---

## Summary

The normalized relational model is the safest, most well-understood choice for a parking management system. It provides the strongest guarantees around data integrity for critical flows like payments, permits, and citations. The main tradeoff is managing high-volume IoT data (sensors, LPR reads) within a relational paradigm, which requires partitioning and lifecycle automation. For most operators -- municipalities, universities, garages with up to a few hundred facilities -- this model will serve well for years before any scaling concerns arise.
