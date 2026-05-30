# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

## Overview

This model combines the strengths of normalized relational tables for stable, well-understood entities (facilities, customers, payments) with JSONB columns for data that is inherently variable, configuration-heavy, or changes shape frequently (permit rules, pricing structures, device configurations, sensor metadata). PostgreSQL's native JSONB support -- including GIN indexing, containment operators, and JSON path queries -- enables this hybrid approach within a single database engine, avoiding the operational complexity of running separate relational and document databases.

The key insight driving this design is that a parking management system serves diverse operators with vastly different configurations. A municipality managing on-street meters has different space attributes, pricing rules, and enforcement workflows than a university campus or a residential HOA. Rather than creating an explosion of nullable columns or entity-attribute-value tables to accommodate this variability, JSONB columns capture operator-specific extensions naturally while the relational backbone enforces integrity on the data that is structurally consistent across all deployments.

---

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary Database | PostgreSQL 16+ | Native JSONB with GIN indexes, PostGIS, RLS -- no second database needed |
| Spatial Extension | PostGIS 3.4+ | Geospatial queries for facility locations, zone boundaries, geofencing |
| Connection Pooling | PgBouncer | Multi-tenant connection management |
| Migration Tool | Flyway or Liquibase | Schema migrations for relational columns; JSONB columns are schema-flexible |
| Caching | Redis | Real-time occupancy counters, session tokens, rate caches |
| Validation | JSON Schema (application layer) | Validate JSONB payloads against per-tenant schemas at the application layer |
| Full-Text Search | PostgreSQL tsvector + GIN | Search across relational and JSONB fields |

---

## Design Principles

1. **Normalize data that participates in foreign keys, aggregates, or WHERE clauses.** If you JOIN on it, GROUP BY it, or filter it in most queries, it belongs in a typed column.

2. **Use JSONB for data that varies by operator, changes schema frequently, or is read/written as a unit.** Configuration objects, metadata bags, device telemetry, and operator-specific custom fields go in JSONB.

3. **Index JSONB paths that are queried frequently.** Use GIN indexes for containment queries and expression indexes for specific key lookups.

4. **Validate JSONB structure at the application layer.** Use JSON Schema definitions per tenant or entity type to enforce structure without database-level rigidity.

---

## Core Schema

### Tenant and Organization

```sql
CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'terminated')),
    -- JSONB: Operator-specific configuration that varies widely
    -- Examples:
    -- {
    --   "branding": { "logo_url": "...", "primary_color": "#003366", "portal_name": "City Parking" },
    --   "enforcement": { "grace_period_minutes": 10, "auto_adjudicate": true, "max_citations_before_tow": 5 },
    --   "notifications": { "sms_enabled": true, "email_templates": { ... } },
    --   "integrations": { "payment_gateway": "stripe", "sso_provider": "okta", "lpr_vendor": "genetec" },
    --   "features_enabled": ["dynamic_pricing", "ev_charging", "chatbot", "mobile_app"],
    --   "custom_fields_schema": { ... }  -- JSON Schema for tenant-specific custom fields
    -- }
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_config ON tenants USING GIN(config);

CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    role            VARCHAR(30) NOT NULL DEFAULT 'operator',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- JSONB: User preferences, notification settings, mobile device tokens
    -- {
    --   "preferences": { "language": "en", "date_format": "MM/DD/YYYY", "dashboard_layout": "..." },
    --   "mobile_devices": [{ "token": "...", "platform": "ios", "app_version": "2.1" }],
    --   "sso": { "provider": "okta", "external_id": "...", "groups": ["parking_admins"] }
    -- }
    profile         JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

### Facility and Space Inventory

```sql
CREATE TABLE facilities (
    facility_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    facility_type   VARCHAR(30) NOT NULL
                    CHECK (facility_type IN ('garage', 'surface_lot', 'on_street',
                                             'underground', 'mixed_use')),
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),
    location        GEOGRAPHY(POINT, 4326),
    boundary        GEOGRAPHY(POLYGON, 4326),
    total_spaces    INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- JSONB: Facility-specific configuration that varies by type
    -- For a garage:
    -- {
    --   "operating_hours": {
    --     "mon_fri": { "open": "06:00", "close": "22:00" },
    --     "sat": { "open": "08:00", "close": "20:00" },
    --     "sun": "closed",
    --     "holidays": [{ "date": "2026-12-25", "status": "closed" }]
    --   },
    --   "height_clearance_ft": 7.2,
    --   "levels": [
    --     { "name": "Level 1", "spaces": 120, "clearance_ft": 8.5 },
    --     { "name": "Level 2", "spaces": 150, "clearance_ft": 7.2 }
    --   ],
    --   "amenities": ["elevator", "restroom", "ev_charging", "security_camera", "covered"],
    --   "signage": { "entry_display_id": "...", "level_displays": [...] },
    --   "parcs_config": { "gate_vendor": "skidata", "ticket_format": "barcode_128" },
    --   "accessibility": { "accessible_entrance": true, "ada_spaces": 8 }
    -- }
    --
    -- For on-street meters:
    -- {
    --   "operating_hours": { "enforcement_hours": "08:00-18:00 Mon-Sat" },
    --   "meter_type": "multi-space",
    --   "time_limit_minutes": 120,
    --   "street_segments": [
    --     { "street": "Main St", "block": "100", "side": "north", "spaces": 12 }
    --   ],
    --   "snow_emergency_rules": { "alternate_side": true, "schedule": "..." }
    -- }
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_facilities_tenant ON facilities(tenant_id);
CREATE INDEX idx_facilities_location ON facilities USING GIST(location);
CREATE INDEX idx_facilities_properties ON facilities USING GIN(properties);

CREATE TABLE zones (
    zone_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(100) NOT NULL,
    zone_code       VARCHAR(20),
    level_name      VARCHAR(100),   -- denormalized: no separate levels table needed
    boundary        GEOGRAPHY(POLYGON, 4326),
    total_spaces    INTEGER NOT NULL DEFAULT 0,
    -- JSONB: Zone-specific rules and attributes
    -- {
    --   "space_types": { "standard": 80, "accessible": 4, "ev": 6, "compact": 10 },
    --   "restrictions": ["permit_only_7am_5pm", "2hr_limit_weekdays"],
    --   "enforcement_priority": "high",
    --   "signage_display_id": "...",
    --   "custom_rules": [
    --     { "rule": "No overnight parking Nov-Mar", "applies": "11/01-03/31" }
    --   ]
    -- }
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_zones_facility ON zones(facility_id);
CREATE INDEX idx_zones_boundary ON zones USING GIST(boundary);

CREATE TABLE spaces (
    space_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    zone_id         UUID REFERENCES zones(zone_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    space_number    VARCHAR(20) NOT NULL,
    space_type      VARCHAR(30) NOT NULL DEFAULT 'standard'
                    CHECK (space_type IN ('standard', 'compact', 'accessible', 'ev_charging',
                                          'motorcycle', 'oversized', 'reserved', 'loading',
                                          'car_share', 'valet')),
    location        GEOGRAPHY(POINT, 4326),
    status          VARCHAR(20) NOT NULL DEFAULT 'available'
                    CHECK (status IN ('available', 'occupied', 'reserved', 'out_of_service',
                                      'maintenance')),
    -- JSONB: Space-specific attributes that vary by facility type
    -- {
    --   "sensor": { "id": "S-001-A23", "type": "ultrasonic", "installed": "2025-06-15" },
    --   "ev_charger": { "station_id": "...", "connector_type": "ccs2", "max_kw": 150 },
    --   "dimensions": { "length_ft": 18, "width_ft": 8.5, "clearance_ft": 7.2 },
    --   "reserved_for": { "permit_type": "faculty", "hours": "7am-5pm Mon-Fri" },
    --   "meter": { "meter_id": "M-1234", "meter_type": "single-space", "accepts": ["coin", "card", "app"] },
    --   "notes": "Near elevator bank A"
    -- }
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (facility_id, space_number)
);

CREATE INDEX idx_spaces_facility ON spaces(facility_id);
CREATE INDEX idx_spaces_zone ON spaces(zone_id);
CREATE INDEX idx_spaces_status ON spaces(status);
CREATE INDEX idx_spaces_type ON spaces(space_type);
CREATE INDEX idx_spaces_attributes ON spaces USING GIN(attributes);
-- Expression index for sensor lookups
CREATE INDEX idx_spaces_sensor_id ON spaces((attributes->>'sensor'->>'id'))
    WHERE attributes ? 'sensor';
```

### Vehicles and Customers

```sql
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
    -- JSONB: Customer profile details that vary by tenant
    -- Municipal tenant:
    -- {
    --   "address": { "line1": "...", "city": "...", "state": "...", "zip": "..." },
    --   "identity_verification": { "verified": true, "method": "drivers_license", "verified_at": "..." },
    --   "employer_validation": { "employer": "City Hall", "employee_id": "E-456" },
    --   "communication_preferences": { "email": true, "sms": true, "push": false },
    --   "language": "en"
    -- }
    --
    -- University tenant:
    -- {
    --   "student_id": "STU-2026-1234",
    --   "department": "Computer Science",
    --   "classification": "graduate",
    --   "advisor": "Dr. Smith",
    --   "housing": "on_campus",
    --   "meal_plan": true
    -- }
    --
    -- Residential (HOA) tenant:
    -- {
    --   "unit_number": "4B",
    --   "building": "Tower A",
    --   "move_in_date": "2024-03-15",
    --   "lease_expires": "2026-08-31",
    --   "guest_parking_credits": 5
    -- }
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_profile ON customers USING GIN(profile);

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
    is_ev           BOOLEAN NOT NULL DEFAULT false,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- JSONB: Vehicle-specific attributes
    -- {
    --   "registration_state": "CA",
    --   "registration_expiry": "2027-03-15",
    --   "vin": "1HGBH41JXMN109186",
    --   "transponder_id": "T-789-456",
    --   "fleet_id": "FLEET-2026-A",
    --   "ev_details": { "battery_kwh": 75, "connector_types": ["ccs2", "type2"], "range_miles": 280 },
    --   "dimensions": { "length_ft": 15.5, "height_ft": 5.2, "weight_lbs": 4200 }
    -- }
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicles_customer ON vehicles(customer_id);
CREATE INDEX idx_vehicles_plate ON vehicles(license_plate, plate_state, plate_country);
CREATE INDEX idx_vehicles_tenant ON vehicles(tenant_id);
```

### Permit Management

```sql
CREATE TABLE permit_types (
    permit_type_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    category        VARCHAR(30) NOT NULL,
    price           NUMERIC(10, 2),
    billing_cycle   VARCHAR(20),
    max_vehicles    INTEGER NOT NULL DEFAULT 1,
    is_virtual      BOOLEAN NOT NULL DEFAULT true,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- JSONB: Full permit rule configuration -- the heart of the hybrid approach
    -- This is where the "natural language configuration" feature maps to structured rules.
    -- {
    --   "eligibility": {
    --     "requires_verification": true,
    --     "verification_types": ["student_id", "employee_badge"],
    --     "required_customer_fields": ["student_id", "department"],
    --     "excluded_classifications": ["freshman_on_campus"]
    --   },
    --   "validity": {
    --     "duration_days": 365,
    --     "valid_days": [1, 2, 3, 4, 5],
    --     "valid_hours": { "start": "06:00", "end": "22:00" },
    --     "blackout_dates": ["2026-12-25", "2027-01-01"],
    --     "semester_aligned": true,
    --     "semester_dates": { "fall": "2026-08-25", "spring": "2027-01-15" }
    --   },
    --   "zones": {
    --     "allowed_zone_ids": ["uuid1", "uuid2"],
    --     "allowed_facility_ids": ["uuid3"],
    --     "overflow_zones": ["uuid4"],
    --     "overflow_after_pct": 95
    --   },
    --   "pricing": {
    --     "base_price": 150.00,
    --     "installment_plan": { "payments": 3, "interval": "monthly" },
    --     "early_bird_discount": { "pct": 10, "before_date": "2026-07-01" },
    --     "prorated": true,
    --     "refund_policy": { "full_refund_before_days": 14, "prorated_after": true }
    --   },
    --   "waitlist": {
    --     "enabled": true,
    --     "max_waitlist": 50,
    --     "offer_expiry_hours": 48,
    --     "priority_groups": ["faculty", "staff", "graduate", "undergraduate"]
    --   },
    --   "restrictions": {
    --     "max_active": 500,
    --     "max_per_customer": 1,
    --     "vehicle_types_allowed": ["sedan", "suv", "compact", "ev"],
    --     "oversized_surcharge": 25.00
    --   },
    --   "geofencing": {
    --     "enabled": true,
    --     "activation_radius_meters": 200,
    --     "deactivation_radius_meters": 500
    --   }
    -- }
    rules           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_permit_types_tenant ON permit_types(tenant_id);
CREATE INDEX idx_permit_types_rules ON permit_types USING GIN(rules);

CREATE TABLE permits (
    permit_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    permit_type_id  UUID NOT NULL REFERENCES permit_types(permit_type_id),
    customer_id     UUID NOT NULL REFERENCES customers(customer_id),
    permit_number   VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'active', 'expired', 'revoked',
                                      'suspended', 'waitlisted')),
    effective_from  TIMESTAMPTZ NOT NULL,
    effective_to    TIMESTAMPTZ,
    issued_at       TIMESTAMPTZ,
    issued_by       UUID REFERENCES users(user_id),
    renewed_from    UUID REFERENCES permits(permit_id),
    -- JSONB: Permit instance data -- filled based on the permit_type rules
    -- {
    --   "vehicles": [
    --     { "vehicle_id": "...", "license_plate": "ABC-123", "added_at": "..." }
    --   ],
    --   "payment_plan": {
    --     "installments": [
    --       { "amount": 50.00, "due_date": "2026-09-01", "paid": true, "txn_id": "..." },
    --       { "amount": 50.00, "due_date": "2026-10-01", "paid": false },
    --       { "amount": 50.00, "due_date": "2026-11-01", "paid": false }
    --     ]
    --   },
    --   "verification": {
    --     "status": "verified",
    --     "verified_at": "2026-08-20T14:30:00Z",
    --     "documents": [{ "type": "student_id", "file_url": "..." }]
    --   },
    --   "geofence": {
    --     "last_activation": "2026-09-15T08:30:00Z",
    --     "activation_count": 42
    --   },
    --   "notes": "Transferred from West Campus lot due to construction"
    -- }
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, permit_number)
);

CREATE INDEX idx_permits_tenant ON permits(tenant_id);
CREATE INDEX idx_permits_customer ON permits(customer_id);
CREATE INDEX idx_permits_status ON permits(status);
CREATE INDEX idx_permits_effective ON permits(effective_from, effective_to);
CREATE INDEX idx_permits_details ON permits USING GIN(details);
-- Expression index for plate lookups within permit details
CREATE INDEX idx_permits_plates ON permits USING GIN((details->'vehicles'));
```

### Parking Sessions

```sql
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
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'completed', 'cancelled', 'overstay')),
    amount_due      NUMERIC(10, 2) DEFAULT 0.00,
    amount_paid     NUMERIC(10, 2) DEFAULT 0.00,
    permit_id       UUID REFERENCES permits(permit_id),
    reservation_id  UUID,
    -- JSONB: Session details that accumulate during the session
    -- {
    --   "entry": {
    --     "method": "lpr",
    --     "lane_id": "...",
    --     "lpr_read_id": "...",
    --     "lpr_confidence": 0.97,
    --     "image_url": "..."
    --   },
    --   "exit": {
    --     "method": "lpr",
    --     "lane_id": "...",
    --     "lpr_confidence": 0.99,
    --     "image_url": "..."
    --   },
    --   "pricing": {
    --     "rate_schedule_id": "...",
    --     "rate_name": "Weekday Hourly",
    --     "base_amount": 12.00,
    --     "dynamic_multiplier": 1.25,
    --     "dynamic_reason": "event_surge",
    --     "final_amount": 15.00,
    --     "breakdown": [
    --       { "period": "0-60min", "rate": 3.00 },
    --       { "period": "60-120min", "rate": 4.00 },
    --       { "period": "120-180min", "rate": 5.00 }
    --     ]
    --   },
    --   "validations": [
    --     { "code": "VAL-ABC", "sponsor": "City Mall", "discount": 2.00, "applied_at": "..." }
    --   ],
    --   "extensions": [
    --     { "requested_at": "...", "additional_minutes": 60, "amount": 4.00 }
    --   ],
    --   "ev_charging": {
    --     "station_id": "...",
    --     "energy_kwh": 32.5,
    --     "charging_cost": 8.13,
    --     "start_time": "...",
    --     "end_time": "..."
    --   }
    -- }
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_tenant ON parking_sessions(tenant_id);
CREATE INDEX idx_sessions_facility ON parking_sessions(facility_id);
CREATE INDEX idx_sessions_plate ON parking_sessions(license_plate);
CREATE INDEX idx_sessions_status ON parking_sessions(status);
CREATE INDEX idx_sessions_entry ON parking_sessions(entry_time);
CREATE INDEX idx_sessions_active ON parking_sessions(facility_id, status)
    WHERE status = 'active';
CREATE INDEX idx_sessions_details ON parking_sessions USING GIN(details);
```

### Pricing and Rates

```sql
CREATE TABLE rate_schedules (
    rate_schedule_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID REFERENCES facilities(facility_id),
    zone_id         UUID REFERENCES zones(zone_id),
    name            VARCHAR(100) NOT NULL,
    rate_type       VARCHAR(20) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    priority        INTEGER NOT NULL DEFAULT 0,
    effective_from  TIMESTAMPTZ NOT NULL,
    effective_to    TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- JSONB: Complete rate definition -- supports arbitrarily complex pricing
    -- {
    --   "schedule": {
    --     "applicable_days": [1, 2, 3, 4, 5],
    --     "applicable_hours": { "start": "06:00", "end": "22:00" },
    --     "exceptions": [
    --       { "date": "2026-12-25", "rate_override": "holiday_flat" }
    --     ]
    --   },
    --   "tiers": [
    --     { "from_minutes": 0, "to_minutes": 30, "rate": 0.00, "unit": "flat", "label": "Free first 30 min" },
    --     { "from_minutes": 30, "to_minutes": 60, "rate": 2.50, "unit": "per_hour" },
    --     { "from_minutes": 60, "to_minutes": 180, "rate": 3.00, "unit": "per_hour" },
    --     { "from_minutes": 180, "to_minutes": null, "rate": 4.00, "unit": "per_hour" }
    --   ],
    --   "caps": {
    --     "daily_max": 25.00,
    --     "weekly_max": 100.00,
    --     "monthly_max": 350.00
    --   },
    --   "dynamic_pricing": {
    --     "enabled": true,
    --     "min_multiplier": 0.75,
    --     "max_multiplier": 2.50,
    --     "thresholds": [
    --       { "occupancy_pct": 50, "multiplier": 1.0 },
    --       { "occupancy_pct": 75, "multiplier": 1.25 },
    --       { "occupancy_pct": 90, "multiplier": 1.75 },
    --       { "occupancy_pct": 95, "multiplier": 2.50 }
    --     ],
    --     "event_surcharge": { "detection_method": "calendar_api", "multiplier": 1.50 }
    --   },
    --   "special_rates": {
    --     "ev_discount_pct": 10,
    --     "accessible_free_first_hours": 2,
    --     "motorcycle_discount_pct": 50,
    --     "carpool_discount_pct": 25
    --   },
    --   "early_bird": {
    --     "entry_before": "09:00",
    --     "exit_after": "16:00",
    --     "flat_rate": 12.00
    --   },
    --   "night_rate": {
    --     "after": "18:00",
    --     "before": "06:00",
    --     "flat_rate": 8.00
    --   }
    -- }
    rate_definition JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rates_tenant ON rate_schedules(tenant_id);
CREATE INDEX idx_rates_facility ON rate_schedules(facility_id);
CREATE INDEX idx_rates_effective ON rate_schedules(effective_from, effective_to);
CREATE INDEX idx_rates_definition ON rate_schedules USING GIN(rate_definition);
```

### Enforcement and Citations

```sql
CREATE TABLE citations (
    citation_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    zone_id         UUID REFERENCES zones(zone_id),
    citation_number VARCHAR(50) NOT NULL,
    license_plate   VARCHAR(20) NOT NULL,
    plate_state     VARCHAR(10),
    vehicle_id      UUID REFERENCES vehicles(vehicle_id),
    customer_id     UUID REFERENCES customers(customer_id),
    violation_type  VARCHAR(50) NOT NULL,
    fine_amount     NUMERIC(10, 2) NOT NULL,
    late_fee        NUMERIC(10, 2) DEFAULT 0.00,
    status          VARCHAR(20) NOT NULL DEFAULT 'issued',
    issued_by       UUID REFERENCES users(user_id),
    issued_at       TIMESTAMPTZ NOT NULL,
    due_date        DATE,
    paid_at         TIMESTAMPTZ,
    -- JSONB: All citation details, evidence, appeals, and enforcement history
    -- {
    --   "location": {
    --     "description": "Level 2, Row C, near elevator",
    --     "coordinates": { "lat": 37.7749, "lng": -122.4194 },
    --     "space_number": "2C-15"
    --   },
    --   "evidence": [
    --     {
    --       "type": "photo",
    --       "url": "s3://citations/2026/09/CIT-001-front.jpg",
    --       "captured_at": "2026-09-15T14:30:00Z",
    --       "description": "Front of vehicle showing expired meter"
    --     },
    --     {
    --       "type": "lpr_scan",
    --       "url": "s3://lpr/2026/09/read-uuid.jpg",
    --       "confidence": 0.96,
    --       "camera_id": "CAM-PATROL-3"
    --     }
    --   ],
    --   "lpr": {
    --     "is_lpr_generated": true,
    --     "confidence": 0.96,
    --     "read_id": "...",
    --     "camera_id": "CAM-PATROL-3"
    --   },
    --   "adjudication": {
    --     "auto_adjudicated": true,
    --     "checks_performed": ["permit_check", "session_check", "grace_period"],
    --     "ai_recommendation": "uphold",
    --     "ai_confidence": 0.94
    --   },
    --   "appeals": [
    --     {
    --       "appeal_id": "...",
    --       "submitted_at": "...",
    --       "reason": "valid_permit",
    --       "statement": "I had a valid permit...",
    --       "documents": ["s3://appeals/...pdf"],
    --       "status": "denied",
    --       "reviewed_by": "...",
    --       "review_notes": "Permit expired 3 days prior to citation",
    --       "decision_at": "..."
    --     }
    --   ],
    --   "enforcement_actions": [
    --     {
    --       "action_id": "...",
    --       "type": "boot",
    --       "status": "completed",
    --       "ordered_at": "...",
    --       "completed_at": "...",
    --       "vendor": "CityTow LLC",
    --       "cost": 75.00,
    --       "released_at": "..."
    --     }
    --   ],
    --   "payment_history": [
    --     { "txn_id": "...", "amount": 50.00, "method": "credit_card", "paid_at": "..." }
    --   ],
    --   "escalation": {
    --     "collections_sent_at": "...",
    --     "collections_agency": "ABC Collections",
    --     "collections_reference": "COL-2026-789"
    --   },
    --   "custom_fields": {
    --     "officer_badge_number": "B-1234",
    --     "weather_conditions": "clear",
    --     "was_idling": true
    --   }
    -- }
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, citation_number)
);

CREATE INDEX idx_citations_tenant ON citations(tenant_id);
CREATE INDEX idx_citations_plate ON citations(license_plate);
CREATE INDEX idx_citations_status ON citations(status);
CREATE INDEX idx_citations_issued ON citations(issued_at);
CREATE INDEX idx_citations_details ON citations USING GIN(details);
```

### Payment Processing

```sql
CREATE TABLE payment_transactions (
    transaction_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    customer_id     UUID REFERENCES customers(customer_id),
    session_id      UUID REFERENCES parking_sessions(session_id),
    permit_id       UUID REFERENCES permits(permit_id),
    citation_id     UUID REFERENCES citations(citation_id),
    transaction_type VARCHAR(20) NOT NULL
                    CHECK (transaction_type IN ('charge', 'refund', 'void', 'adjustment')),
    payment_method  VARCHAR(20) NOT NULL,
    amount          NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    tax_amount      NUMERIC(10, 2) DEFAULT 0.00,
    -- PCI-DSS: tokenized payment references only
    payment_token   VARCHAR(255),
    gateway         VARCHAR(50),
    gateway_txn_id  VARCHAR(255),
    card_last_four  CHAR(4),
    card_brand      VARCHAR(20),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'completed', 'failed',
                                      'refunded', 'disputed', 'voided')),
    -- JSONB: Gateway-specific response data (varies by payment provider)
    -- {
    --   "stripe": { "charge_id": "ch_xxx", "receipt_url": "...", "risk_score": 12 },
    --   "metadata": { "invoice_number": "INV-2026-001", "department_code": "PKG-MAIN" },
    --   "failure_details": { "code": "card_declined", "decline_code": "insufficient_funds" },
    --   "3ds": { "authenticated": true, "version": "2.2" },
    --   "reconciliation": { "batch_id": "B-20260915", "settled_at": "..." }
    -- }
    gateway_data    JSONB NOT NULL DEFAULT '{}',
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_txns_tenant ON payment_transactions(tenant_id);
CREATE INDEX idx_txns_customer ON payment_transactions(customer_id);
CREATE INDEX idx_txns_session ON payment_transactions(session_id);
CREATE INDEX idx_txns_status ON payment_transactions(status);
CREATE INDEX idx_txns_date ON payment_transactions(created_at);
```

### Hardware and IoT

```sql
CREATE TABLE hardware_devices (
    device_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    device_type     VARCHAR(30) NOT NULL,
    device_name     VARCHAR(100) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'online',
    last_heartbeat  TIMESTAMPTZ,
    -- JSONB: Device-specific configuration (varies wildly by device type and vendor)
    -- Gate controller:
    -- {
    --   "manufacturer": "SKIDATA",
    --   "model": "Power.Gate",
    --   "firmware": "4.2.1",
    --   "network": { "ip": "10.0.1.50", "mac": "AA:BB:CC:DD:EE:FF", "protocol": "tcp" },
    --   "lane_id": "...",
    --   "lane_type": "entry",
    --   "lpr_camera_id": "...",
    --   "barrier_timing": { "open_delay_ms": 500, "close_delay_ms": 3000 },
    --   "access_modes": ["lpr", "ticket", "card", "qr"]
    -- }
    --
    -- Occupancy sensor:
    -- {
    --   "manufacturer": "ParkAssist",
    --   "model": "M4",
    --   "sensor_type": "ultrasonic",
    --   "space_id": "...",
    --   "calibration": { "threshold_cm": 150, "last_calibrated": "2026-01-15" },
    --   "battery": { "type": "lithium", "installed": "2025-06-01", "expected_life_months": 60 },
    --   "wireless": { "protocol": "lorawan", "dev_eui": "...", "app_key": "..." }
    -- }
    --
    -- Digital sign:
    -- {
    --   "manufacturer": "Daktronics",
    --   "model": "GS6",
    --   "display_type": "led_matrix",
    --   "resolution": "128x32",
    --   "shows_count_for": ["zone-uuid-1", "zone-uuid-2"],
    --   "content_template": "OPEN {{available}} / FULL",
    --   "network": { "ip": "10.0.2.20", "protocol": "modbus" }
    -- }
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_devices_facility ON hardware_devices(facility_id);
CREATE INDEX idx_devices_type ON hardware_devices(device_type);
CREATE INDEX idx_devices_status ON hardware_devices(status);
CREATE INDEX idx_devices_config ON hardware_devices USING GIN(config);
```

### Sensor and LPR Data (Time-Partitioned)

```sql
-- LPR reads: relational columns for queried fields, JSONB for variable metadata
CREATE TABLE lpr_reads (
    lpr_read_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    camera_id       VARCHAR(100) NOT NULL,
    license_plate   VARCHAR(20) NOT NULL,
    confidence_score NUMERIC(5, 4) NOT NULL,
    direction       VARCHAR(10),
    read_at         TIMESTAMPTZ NOT NULL,
    -- JSONB: LPR-specific metadata that varies by vendor
    -- {
    --   "plate_state": "CA",
    --   "plate_country": "US",
    --   "image_url": "s3://lpr/2026/09/15/read-uuid.jpg",
    --   "thumbnail_url": "s3://lpr/2026/09/15/read-uuid-thumb.jpg",
    --   "matched_vehicle_id": "...",
    --   "matched_session_id": "...",
    --   "review": { "required": false, "reviewed_by": null, "corrected_plate": null },
    --   "vendor_data": { "genetec_event_id": "...", "processing_time_ms": 45 },
    --   "plate_region": { "type": "standard", "characters": "ABC1234" },
    --   "ocr_alternatives": [
    --     { "plate": "A8C1234", "confidence": 0.82 },
    --     { "plate": "ABC1Z34", "confidence": 0.71 }
    --   ]
    -- }
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (read_at);

CREATE INDEX idx_lpr_plate ON lpr_reads(license_plate);
CREATE INDEX idx_lpr_camera ON lpr_reads(camera_id, read_at);
CREATE INDEX idx_lpr_facility ON lpr_reads(facility_id, read_at);
CREATE INDEX idx_lpr_confidence ON lpr_reads(confidence_score)
    WHERE confidence_score < 0.85;

-- Occupancy sensor readings
CREATE TABLE occupancy_readings (
    reading_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    sensor_id       VARCHAR(100) NOT NULL,
    space_id        UUID,
    zone_id         UUID,
    is_occupied     BOOLEAN NOT NULL,
    read_at         TIMESTAMPTZ NOT NULL,
    -- JSONB: Sensor telemetry data
    -- {
    --   "battery_pct": 82,
    --   "signal_rssi": -65,
    --   "temperature_c": 22.5,
    --   "detection_method": "ultrasonic",
    --   "raw_distance_cm": 145,
    --   "firmware_version": "3.1.2"
    -- }
    telemetry       JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (read_at);

CREATE INDEX idx_occupancy_sensor ON occupancy_readings(sensor_id, read_at);
CREATE INDEX idx_occupancy_facility ON occupancy_readings(facility_id, read_at);
CREATE INDEX idx_occupancy_space ON occupancy_readings(space_id, read_at);
```

### Reservations

```sql
CREATE TABLE reservations (
    reservation_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    facility_id     UUID NOT NULL REFERENCES facilities(facility_id),
    customer_id     UUID NOT NULL REFERENCES customers(customer_id),
    vehicle_id      UUID REFERENCES vehicles(vehicle_id),
    space_id        UUID REFERENCES spaces(space_id),
    zone_id         UUID REFERENCES zones(zone_id),
    reservation_type VARCHAR(20) NOT NULL DEFAULT 'single',
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'confirmed',
    amount          NUMERIC(10, 2) NOT NULL DEFAULT 0.00,
    confirmation_code VARCHAR(20) NOT NULL,
    -- JSONB: Booking details
    -- {
    --   "recurrence": { "rrule": "FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR", "until": "2026-12-31" },
    --   "qr_code_data": "...",
    --   "special_requests": ["ground_floor", "near_elevator", "ev_charging"],
    --   "event": { "name": "City Concert Series", "event_id": "EVT-2026-42" },
    --   "group_booking": { "group_id": "GRP-001", "total_spaces": 5, "organizer": "..." },
    --   "check_in": { "checked_in_at": "...", "session_id": "..." },
    --   "cancellation": { "reason": "plans_changed", "refund_amount": 12.00, "txn_id": "..." }
    -- }
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, confirmation_code)
);

CREATE INDEX idx_reservations_tenant ON reservations(tenant_id);
CREATE INDEX idx_reservations_facility ON reservations(facility_id);
CREATE INDEX idx_reservations_customer ON reservations(customer_id);
CREATE INDEX idx_reservations_time ON reservations(start_time, end_time);
CREATE INDEX idx_reservations_status ON reservations(status);
```

### Audit Log

```sql
CREATE TABLE audit_log (
    audit_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    -- JSONB: Captures both old and new values plus context
    -- {
    --   "old": { ... },
    --   "new": { ... },
    --   "changed_fields": ["status", "fine_amount"],
    --   "context": { "ip": "192.168.1.100", "user_agent": "...", "source": "admin_portal" }
    -- }
    change_data     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

### Multi-Tenancy (Row-Level Security)

```sql
-- Same RLS pattern as the normalized model
ALTER TABLE facilities ENABLE ROW LEVEL SECURITY;
ALTER TABLE zones ENABLE ROW LEVEL SECURITY;
ALTER TABLE spaces ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE permits ENABLE ROW LEVEL SECURITY;
ALTER TABLE parking_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE citations ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_transactions ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON facilities
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
-- ... applied to all tenant-scoped tables
```

---

## JSONB Query Examples

```sql
-- Find all permits with a specific license plate in the vehicles array
SELECT p.permit_id, p.permit_number, p.status
FROM permits p
WHERE p.tenant_id = :tenant_id
  AND p.details @> '{"vehicles": [{"license_plate": "ABC-1234"}]}';

-- Find facilities with EV charging amenity
SELECT f.facility_id, f.name
FROM facilities f
WHERE f.tenant_id = :tenant_id
  AND f.properties @> '{"amenities": ["ev_charging"]}';

-- Find permit types that allow waitlisting
SELECT pt.name, pt.rules->'waitlist'->>'max_waitlist' AS max_waitlist
FROM permit_types pt
WHERE pt.tenant_id = :tenant_id
  AND pt.rules @> '{"waitlist": {"enabled": true}}';

-- Calculate revenue with dynamic pricing breakdown
SELECT
    ps.session_id,
    ps.license_plate,
    ps.details->'pricing'->>'rate_name' AS rate_applied,
    (ps.details->'pricing'->>'dynamic_multiplier')::NUMERIC AS multiplier,
    ps.amount_due
FROM parking_sessions ps
WHERE ps.tenant_id = :tenant_id
  AND ps.facility_id = :facility_id
  AND ps.entry_time >= :start_date
  AND ps.details->'pricing' ? 'dynamic_multiplier';

-- Find low-confidence LPR reads needing review
SELECT lr.lpr_read_id, lr.license_plate, lr.confidence_score,
       lr.metadata->'ocr_alternatives' AS alternatives
FROM lpr_reads lr
WHERE lr.tenant_id = :tenant_id
  AND lr.confidence_score < 0.85
  AND NOT (lr.metadata->'review'->>'required')::BOOLEAN;

-- Find devices by vendor-specific configuration
SELECT hd.device_id, hd.device_name, hd.config->'wireless'->>'protocol' AS protocol
FROM hardware_devices hd
WHERE hd.tenant_id = :tenant_id
  AND hd.device_type = 'occupancy_sensor'
  AND hd.config @> '{"wireless": {"protocol": "lorawan"}}';
```

---

## Pros and Cons

### Pros

1. **Single database engine**: Everything runs in PostgreSQL. No need to operate MongoDB alongside PostgreSQL or manage data synchronization between systems. One backup strategy, one monitoring stack, one team skill set.

2. **Flexibility where it matters**: Permit rules, pricing structures, device configurations, and operator-specific fields can evolve without schema migrations. Adding a new permit rule parameter is a code change, not a DDL migration.

3. **Reduced table count**: By consolidating related sub-entities (evidence, appeals, enforcement actions) into JSONB within the citation, the schema has ~50% fewer tables than the fully normalized model. This simplifies ORM mapping and reduces JOIN complexity.

4. **Type-safe where it matters**: Critical business data (amounts, dates, foreign keys, license plates) lives in typed columns with constraints. You cannot store a string in `fine_amount` or reference a nonexistent facility.

5. **Rich querying of flexible data**: PostgreSQL's JSONB operators (`@>`, `?`, `->`, `->>`) and GIN indexes allow efficient querying into document fields without sacrificing the relational model. You can JOIN across relational and JSONB data in a single query.

6. **Natural fit for multi-operator variability**: A municipality's permit rules differ completely from a university's or an HOA's. Rather than designing the union of all possible permit attributes as columns, each tenant defines rules in JSON, validated by the application against a JSON Schema.

7. **Progressive normalization**: If a JSONB field turns out to be queried heavily or needs referential integrity, it can be extracted into its own relational column or table in a later migration. The hybrid model is not a one-way door.

8. **Excellent developer experience**: Modern ORMs (Prisma, SQLAlchemy, TypeORM) have strong JSONB support. Application code can work with JSONB fields as native objects.

### Cons

1. **No referential integrity within JSONB**: If the citation's `details.appeals[0].reviewed_by` references a user_id, the database cannot enforce that the user exists. This must be validated at the application layer.

2. **JSONB update granularity**: Updating a single field within a JSONB column requires rewriting the entire JSONB value. For frequently-updated document fields, this creates write amplification. PostgreSQL's `jsonb_set()` function helps but still rewrites the whole column value.

3. **Schema discipline required**: Without database-level enforcement, JSONB fields can drift over time -- different tenants may have inconsistent structures in the same field. Application-layer JSON Schema validation is essential but adds development overhead.

4. **Query performance caveats**: GIN indexes on JSONB support containment (`@>`) and existence (`?`) operators efficiently, but range queries on JSONB values (e.g., "find all citations where details.lpr.confidence > 0.9") require expression indexes and are less efficient than typed column indexes.

5. **Reporting complexity**: Business intelligence tools and report writers may struggle with JSONB fields. Extracting JSONB data into flat columns for reporting views requires explicit `->>`/`#>>` expressions, which can be verbose.

6. **Migration testing**: While JSONB columns do not need DDL migrations for new fields, the application code that reads/writes them does need to handle schema evolution (old records without new fields, new records without deprecated fields). This testing burden shifts from the DBA to the application developer.

7. **Potential for data bloat**: Large JSONB documents (e.g., a citation with many appeals, evidence items, and enforcement actions) can grow to 10-50KB per row. This impacts buffer cache efficiency and vacuum performance.

---

## Migration and Scaling Considerations

### Initial Deployment

- Start with a single PostgreSQL 16 instance. The hybrid model's reduced table count means simpler initial setup.
- Define JSON Schema files for each JSONB column and validate at the application layer from day one.
- Use PostgreSQL's `pg_stat_user_tables` and `pg_stat_user_indexes` to monitor which JSONB fields are queried most and add expression indexes as needed.

### Growth Path

- Extract frequently-queried JSONB fields into typed columns as access patterns solidify. For example, if every query on `parking_sessions` filters by `details.entry.method`, promote it to a relational column.
- Add materialized views for reporting that flatten JSONB into columnar formats.
- Partition high-volume tables (`lpr_reads`, `occupancy_readings`, `audit_log`) by time.

### Enterprise Scale

- Same PostgreSQL scaling strategies as the normalized model: read replicas, Citus for sharding, PgBouncer for connection pooling.
- Consider moving the analytics workload to ClickHouse with a CDC pipeline (Debezium) that extracts and flattens JSONB data for columnar analytics.
- Implement JSONB column compression policies: archive old JSONB data to cold storage, keep only relational columns for historical records.

### JSON Schema Governance

Maintain a JSON Schema registry (stored in-repo or in a `json_schemas` table) that defines the expected structure for each JSONB column by entity type and version:

```sql
CREATE TABLE json_schemas (
    schema_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(50) NOT NULL,  -- 'permit_type_rules', 'facility_properties', etc.
    version         INTEGER NOT NULL,
    schema          JSONB NOT NULL,         -- JSON Schema definition
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (entity_type, version)
);
```

---

## Summary

The hybrid relational + JSONB model is the pragmatic middle ground for a parking management system. It preserves relational integrity for financial, identity, and referential data while embracing the inherent variability of operator configurations, device integrations, and enforcement workflows through JSONB. The single-engine approach (PostgreSQL only) minimizes operational complexity while providing the flexibility that a multi-segment platform demands. The key risk is JSONB schema drift, which is mitigated through application-layer JSON Schema validation and a governance discipline of documenting expected JSONB structures. For most development teams, this model offers the best balance of developer productivity, operational simplicity, and domain flexibility.
