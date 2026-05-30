# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Corporate Travel Management · Created: 2026-05-22

## Philosophy

This model treats every business action — booking a flight, approving a request, submitting an expense, changing a policy — as an immutable event appended to a central event store. The event store is the single source of truth. Current state is derived by replaying events into materialised read models (projections) optimised for specific query patterns. This is the Command Query Responsibility Segregation (CQRS) pattern combined with event sourcing.

Event sourcing is particularly well-suited to corporate travel management because the domain has strong regulatory requirements for complete audit trails (PCI DSS, GDPR, ISO 31030), temporal queries are inherent ("what was the travel policy when this booking was made?", "what was the traveller's location at 3pm on Tuesday?"), and multiple downstream systems (expense, compliance, carbon reporting, analytics) need to react to the same business events. Airlines themselves are moving toward an event-driven Offer/Order model via IATA NDC and ONE Order, making event sourcing a natural fit for the platform's integration architecture.

The approach is inspired by financial ledger systems (where every transaction is an append-only entry), airline reservation systems (which maintain a history of every PNR modification), and modern event-driven architectures used by companies like Stripe, Uber, and Navan for real-time transaction processing.

**Best for:** Organisations requiring complete audit trails, temporal queries, real-time event-driven integrations, and AI/ML pipelines that consume change streams for anomaly detection and spend pattern analysis.

**Trade-offs:**
- (+) Complete, immutable audit trail of every change — built-in rather than bolted-on
- (+) Temporal queries are trivial: replay to any point in time
- (+) Event stream feeds AI/ML pipelines, real-time dashboards, and downstream integrations naturally
- (+) Decoupled read/write scaling — event ingestion and query projections scale independently
- (+) Schema evolution is easier: add new event types without altering existing ones
- (-) Higher implementation complexity — requires event store, projection builders, and eventual consistency handling
- (-) Eventually consistent read models may show stale data briefly after writes
- (-) Event schema versioning requires careful management (upcasters, version negotiation)
- (-) Debugging requires understanding both events and projections, not just table state
- (-) Storage grows faster than a mutable relational model (events are never deleted)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IATA NDC / ONE Order | NDC's Offer/Order model is natively event-driven; NDC events map directly to event store entries |
| PCI DSS v4.0.1 | Immutable event log satisfies PCI DSS audit trail requirements; payment card tokens are event payloads |
| GDPR / CCPA | Event store supports right-to-erasure via crypto-shredding (encrypt PII per user, delete key on erasure) |
| ISO 31030:2021 | Location and risk events create a complete duty-of-care timeline per traveller |
| GHG Protocol Scope 3 Cat 6 | Carbon calculation events create auditable emission records with full methodology provenance |
| ISO 4217 / ISO 3166 | Reference data codes embedded in event payloads for standards alignment |
| OCSF (Open Cybersecurity Schema Framework) | Audit events follow structured event log patterns compatible with SIEM ingestion |

---

## Event Store (Core)

```sql
-- =============================================================
-- EVENT STORE — the single source of truth
-- =============================================================
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                   -- aggregate root ID (trip, booking, expense_report, etc.)
    stream_type     VARCHAR(50) NOT NULL,             -- 'trip', 'booking', 'expense_report', 'travel_request', 'policy'
    event_type      VARCHAR(100) NOT NULL,            -- 'BookingCreated', 'FlightSegmentAdded', 'ExpenseSubmitted', etc.
    event_version   INTEGER NOT NULL,                 -- sequence number within stream
    organisation_id UUID NOT NULL,                   -- tenant isolation
    actor_id        UUID,                            -- user who triggered the event
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user', -- 'user', 'system', 'ai_agent'
    payload         JSONB NOT NULL,                  -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',     -- correlation IDs, IP, user agent, causation
    schema_version  SMALLINT NOT NULL DEFAULT 1,     -- for event schema evolution
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Ensure ordering within a stream
    UNIQUE(stream_id, event_version)
);

-- Partitioned by month for storage management
-- In production, use declarative partitioning:
-- CREATE TABLE event_store (...) PARTITION BY RANGE (created_at);

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_org ON event_store(organisation_id);
CREATE INDEX idx_event_created ON event_store(created_at);
CREATE INDEX idx_event_actor ON event_store(actor_id);
CREATE INDEX idx_event_stream_type ON event_store(stream_type);

-- GIN index for querying within event payloads
CREATE INDEX idx_event_payload ON event_store USING GIN (payload jsonb_path_ops);

-- =============================================================
-- STREAM SNAPSHOT — periodic snapshots for replay performance
-- =============================================================
CREATE TABLE stream_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,               -- event_version at which snapshot was taken
    state           JSONB NOT NULL,                  -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, snapshot_version)
);

CREATE INDEX idx_snapshot_stream ON stream_snapshot(stream_id, snapshot_version DESC);
```

## Event Type Catalogue

The following are the primary event types with example payloads:

```sql
-- =============================================================
-- EVENT TYPE REGISTRY — documents all known event types
-- =============================================================
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    schema_version  SMALLINT NOT NULL DEFAULT 1,
    payload_schema  JSONB NOT NULL,                  -- JSON Schema for validation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payloads:
--
-- TravelRequestCreated:
-- {
--   "request_id": "uuid",
--   "requester_id": "uuid",
--   "trip_name": "Q3 Client Visit - London",
--   "purpose": "Client onboarding meetings",
--   "departure_date": "2026-06-15",
--   "return_date": "2026-06-19",
--   "estimated_cost": 4500.00,
--   "currency_code": "USD",
--   "cost_centre_code": "ENG-001"
-- }
--
-- TravelRequestApproved:
-- {
--   "request_id": "uuid",
--   "approver_id": "uuid",
--   "approval_step": 1,
--   "comments": "Approved — within Q3 budget"
-- }
--
-- BookingCreated:
-- {
--   "booking_id": "uuid",
--   "trip_id": "uuid",
--   "booking_type": "flight",
--   "booking_source": "ndc",
--   "total_amount": 1250.00,
--   "currency_code": "USD",
--   "policy_compliant": true,
--   "gds_pnr": "ABC123"
-- }
--
-- FlightSegmentAdded:
-- {
--   "segment_id": "uuid",
--   "booking_id": "uuid",
--   "airline_iata": "BA",
--   "flight_number": "BA178",
--   "origin_iata": "JFK",
--   "destination_iata": "LHR",
--   "departure_at": "2026-06-15T18:30:00Z",
--   "arrival_at": "2026-06-16T06:30:00Z",
--   "cabin_class": "business",
--   "fare_basis": "JFLEX",
--   "is_ndc_booking": true,
--   "ndc_offer_id": "NDC-BA-2026-OFR-789"
-- }
--
-- BookingCancelled:
-- {
--   "booking_id": "uuid",
--   "cancelled_by": "uuid",
--   "reason": "Schedule change",
--   "cancellation_fee": 150.00,
--   "refund_amount": 1100.00
-- }
--
-- ExpenseEntryCreated:
-- {
--   "entry_id": "uuid",
--   "report_id": "uuid",
--   "expense_type": "meal",
--   "vendor_name": "The Ivy, London",
--   "amount": 85.50,
--   "currency_code": "GBP",
--   "transaction_date": "2026-06-16",
--   "receipt_id": "uuid",
--   "gl_account": "6200-MEAL",
--   "ai_coded": true
-- }
--
-- ExpenseReportSubmitted:
-- {
--   "report_id": "uuid",
--   "total_amount": 4320.50,
--   "currency_code": "USD",
--   "entry_count": 12
-- }
--
-- TravellerLocationUpdated:
-- {
--   "user_id": "uuid",
--   "trip_id": "uuid",
--   "country_code": "GB",
--   "city": "London",
--   "latitude": 51.5074,
--   "longitude": -0.1278,
--   "source": "itinerary"
-- }
--
-- RiskAlertTriggered:
-- {
--   "alert_id": "uuid",
--   "alert_type": "security",
--   "severity": "high",
--   "title": "Protest activity near Canary Wharf",
--   "affected_traveller_ids": ["uuid1", "uuid2"],
--   "country_code": "GB"
-- }
--
-- PolicyRuleChanged:
-- {
--   "policy_id": "uuid",
--   "rule_id": "uuid",
--   "category": "flight",
--   "rule_type": "cabin_class",
--   "old_value": "economy",
--   "new_value": "premium_economy",
--   "changed_by": "uuid",
--   "effective_from": "2026-07-01"
-- }
--
-- CarbonEmissionCalculated:
-- {
--   "booking_id": "uuid",
--   "segment_id": "uuid",
--   "transport_mode": "air",
--   "distance_km": 5540.0,
--   "cabin_class": "business",
--   "co2e_kg": 1420.5,
--   "emission_factor": 0.256,
--   "emission_factor_source": "DEFRA 2026",
--   "calculation_method": "distance"
-- }
```

## Read Model Projections (CQRS Query Side)

```sql
-- =============================================================
-- PROJECTION: Current Trip State
-- Materialised from: TravelRequestCreated, BookingCreated,
--   FlightSegmentAdded, HotelBookingAdded, BookingCancelled, etc.
-- =============================================================
CREATE TABLE proj_trip (
    trip_id             UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    travel_request_id   UUID,
    primary_traveller_id UUID NOT NULL,
    trip_name           VARCHAR(300) NOT NULL,
    purpose             TEXT,
    departure_date      DATE,
    return_date         DATE,
    status              VARCHAR(30) NOT NULL,
    total_cost          DECIMAL(12,2) NOT NULL DEFAULT 0,
    currency_code       CHAR(3) NOT NULL,
    booking_count       INTEGER NOT NULL DEFAULT 0,
    policy_compliant    BOOLEAN NOT NULL DEFAULT true,
    last_event_version  INTEGER NOT NULL,            -- tracks projection currency
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_trip_org ON proj_trip(organisation_id);
CREATE INDEX idx_proj_trip_traveller ON proj_trip(primary_traveller_id);
CREATE INDEX idx_proj_trip_dates ON proj_trip(departure_date, return_date);
CREATE INDEX idx_proj_trip_status ON proj_trip(status);

-- =============================================================
-- PROJECTION: Current Booking State
-- Materialised from: BookingCreated, FlightSegmentAdded,
--   BookingStatusChanged, BookingCancelled, etc.
-- =============================================================
CREATE TABLE proj_booking (
    booking_id          UUID PRIMARY KEY,
    trip_id             UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    booking_type        VARCHAR(30) NOT NULL,
    booking_source      VARCHAR(30) NOT NULL,
    gds_pnr             VARCHAR(20),
    status              VARCHAR(30) NOT NULL,
    total_amount        DECIMAL(12,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL,
    policy_compliant    BOOLEAN NOT NULL DEFAULT true,
    policy_violations   TEXT[],
    booked_by           UUID,
    booked_at           TIMESTAMPTZ,
    -- Denormalised segment summary for quick display
    segments            JSONB NOT NULL DEFAULT '[]',
    -- Example segments JSONB:
    -- [
    --   {
    --     "type": "flight",
    --     "airline": "BA",
    --     "flight": "BA178",
    --     "from": "JFK",
    --     "to": "LHR",
    --     "depart": "2026-06-15T18:30:00Z",
    --     "arrive": "2026-06-16T06:30:00Z",
    --     "cabin": "business"
    --   }
    -- ]
    co2e_total_kg       DECIMAL(10,4) DEFAULT 0,
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_booking_trip ON proj_booking(trip_id);
CREATE INDEX idx_proj_booking_org ON proj_booking(organisation_id);
CREATE INDEX idx_proj_booking_status ON proj_booking(status);

-- =============================================================
-- PROJECTION: Expense Report State
-- Materialised from: ExpenseReportCreated, ExpenseEntryCreated,
--   ExpenseReportSubmitted, ExpenseReportApproved, ExpenseReportPaid, etc.
-- =============================================================
CREATE TABLE proj_expense_report (
    report_id           UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    user_id             UUID NOT NULL,
    trip_id             UUID,
    report_name         VARCHAR(300) NOT NULL,
    status              VARCHAR(30) NOT NULL,
    total_amount        DECIMAL(12,2) NOT NULL DEFAULT 0,
    currency_code       CHAR(3) NOT NULL,
    entry_count         INTEGER NOT NULL DEFAULT 0,
    amount_due_employee DECIMAL(12,2) DEFAULT 0,
    amount_due_company  DECIMAL(12,2) DEFAULT 0,
    submitted_at        TIMESTAMPTZ,
    approved_at         TIMESTAMPTZ,
    paid_at             TIMESTAMPTZ,
    entries             JSONB NOT NULL DEFAULT '[]',
    -- Example entries JSONB:
    -- [
    --   {
    --     "entry_id": "uuid",
    --     "type": "meal",
    --     "vendor": "The Ivy",
    --     "amount": 85.50,
    --     "currency": "GBP",
    --     "date": "2026-06-16",
    --     "has_receipt": true,
    --     "gl_account": "6200-MEAL"
    --   }
    -- ]
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_expense_org ON proj_expense_report(organisation_id);
CREATE INDEX idx_proj_expense_user ON proj_expense_report(user_id);
CREATE INDEX idx_proj_expense_status ON proj_expense_report(status);

-- =============================================================
-- PROJECTION: Spend Analytics (pre-aggregated)
-- Materialised from: BookingCreated, ExpenseEntryCreated, etc.
-- =============================================================
CREATE TABLE proj_spend_summary (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    period_type         VARCHAR(10) NOT NULL,         -- 'daily', 'weekly', 'monthly'
    period_start        DATE NOT NULL,
    department_id       UUID,
    cost_centre_code    VARCHAR(50),
    spend_category      VARCHAR(30) NOT NULL,         -- 'flight', 'hotel', 'car', 'meal', 'other'
    total_amount        DECIMAL(14,2) NOT NULL DEFAULT 0,
    currency_code       CHAR(3) NOT NULL,
    booking_count       INTEGER NOT NULL DEFAULT 0,
    policy_violation_count INTEGER NOT NULL DEFAULT 0,
    co2e_total_kg       DECIMAL(12,4) DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, period_type, period_start, department_id, cost_centre_code, spend_category)
);

CREATE INDEX idx_proj_spend_org ON proj_spend_summary(organisation_id);
CREATE INDEX idx_proj_spend_period ON proj_spend_summary(period_start);

-- =============================================================
-- PROJECTION: Traveller Location Timeline
-- Materialised from: TravellerLocationUpdated, RiskAlertTriggered, etc.
-- =============================================================
CREATE TABLE proj_traveller_location (
    user_id             UUID NOT NULL,
    recorded_at         TIMESTAMPTZ NOT NULL,
    trip_id             UUID,
    country_code        CHAR(2) NOT NULL,
    city                VARCHAR(200),
    latitude            DECIMAL(9,6),
    longitude           DECIMAL(9,6),
    location_source     VARCHAR(30) NOT NULL,
    active_alerts       JSONB DEFAULT '[]',
    -- Example active_alerts:
    -- [{"alert_id": "uuid", "type": "security", "severity": "high", "title": "..."}]
    PRIMARY KEY (user_id, recorded_at)
);

CREATE INDEX idx_proj_location_country ON proj_traveller_location(country_code);

-- =============================================================
-- PROJECTION: Policy State (point-in-time reconstructable)
-- Materialised from: PolicyCreated, PolicyRuleChanged, PolicyDeactivated, etc.
-- =============================================================
CREATE TABLE proj_travel_policy (
    policy_id           UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    name                VARCHAR(200) NOT NULL,
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    is_active           BOOLEAN NOT NULL,
    rules               JSONB NOT NULL DEFAULT '[]',
    -- Example rules JSONB:
    -- [
    --   {
    --     "rule_id": "uuid",
    --     "category": "flight",
    --     "rule_type": "cabin_class",
    --     "description": "Economy class for flights under 6 hours",
    --     "condition": {"field": "duration_hours", "op": "lt", "value": 6},
    --     "threshold": "economy",
    --     "severity": "warning"
    --   }
    -- ]
    assignments         JSONB NOT NULL DEFAULT '[]',
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_policy_org ON proj_travel_policy(organisation_id);
```

## Supporting Tables (shared across command and query sides)

```sql
-- =============================================================
-- ORGANISATION (shared — command side writes, query side reads)
-- =============================================================
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    country_code    CHAR(2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- USER (shared — synced via SCIM events)
-- =============================================================
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(320) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    employee_id     VARCHAR(50),
    department      VARCHAR(200),
    cost_centre     VARCHAR(50),
    encryption_key_id VARCHAR(100),                  -- for crypto-shredding (GDPR erasure)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

-- =============================================================
-- REFERENCE DATA — same as normalised model
-- =============================================================
CREATE TABLE airline (
    iata_code       CHAR(2) PRIMARY KEY,
    icao_code       CHAR(3),
    name            VARCHAR(200) NOT NULL,
    is_ndc_enabled  BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE airport (
    iata_code       CHAR(3) PRIMARY KEY,
    icao_code       CHAR(4),
    name            VARCHAR(300) NOT NULL,
    city            VARCHAR(200),
    country_code    CHAR(2) NOT NULL,
    latitude        DECIMAL(9,6),
    longitude       DECIMAL(9,6),
    timezone        VARCHAR(50)
);

CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    minor_units     SMALLINT NOT NULL DEFAULT 2
);

-- =============================================================
-- IDEMPOTENCY (prevent duplicate command processing)
-- =============================================================
CREATE TABLE idempotency_key (
    key             VARCHAR(255) PRIMARY KEY,
    stream_id       UUID NOT NULL,
    event_id        UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL DEFAULT (now() + interval '24 hours')
);

CREATE INDEX idx_idempotency_expires ON idempotency_key(expires_at);

-- =============================================================
-- PROJECTION CHECKPOINT (tracks projection progress)
-- =============================================================
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## GDPR Crypto-Shredding Support

```sql
-- =============================================================
-- ENCRYPTION KEY STORE (for GDPR right-to-erasure)
-- =============================================================
-- Each user gets a unique encryption key. PII in event payloads
-- is encrypted with this key. When a user exercises right-to-erasure,
-- delete their key — all their PII in the event store becomes
-- unreadable without modifying the immutable events.
-- =============================================================
CREATE TABLE user_encryption_key (
    user_id         UUID PRIMARY KEY REFERENCES app_user(id),
    key_id          VARCHAR(100) NOT NULL UNIQUE,     -- reference to key in KMS (AWS KMS, Vault)
    algorithm       VARCHAR(30) NOT NULL DEFAULT 'AES-256-GCM',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ                      -- set on erasure; key destroyed in KMS
);
```

## Example Queries

### Replay a booking's full history

```sql
-- Get the complete timeline of events for a specific booking
SELECT
    event_type,
    event_version,
    actor_id,
    payload,
    created_at
FROM event_store
WHERE stream_id = 'booking-uuid-here'
  AND stream_type = 'booking'
ORDER BY event_version ASC;
```

### Point-in-time policy reconstruction

```sql
-- What policy rules were active when a booking was made?
WITH booking_time AS (
    SELECT created_at
    FROM event_store
    WHERE stream_id = 'booking-uuid-here'
      AND event_type = 'BookingCreated'
)
SELECT
    e.payload
FROM event_store e, booking_time bt
WHERE e.stream_type = 'policy'
  AND e.event_type IN ('PolicyCreated', 'PolicyRuleChanged', 'PolicyRuleRemoved')
  AND e.created_at <= bt.created_at
ORDER BY e.stream_id, e.event_version ASC;
```

### Spend analytics from event stream

```sql
-- Total air spend by department this month (from projections)
SELECT
    u.department,
    SUM((e.payload->>'total_amount')::DECIMAL) as total_spend,
    COUNT(*) as booking_count
FROM event_store e
JOIN app_user u ON (e.payload->>'booked_by')::UUID = u.id
WHERE e.event_type = 'BookingCreated'
  AND e.organisation_id = 'org-uuid-here'
  AND e.payload->>'booking_type' = 'flight'
  AND e.created_at >= date_trunc('month', now())
GROUP BY u.department
ORDER BY total_spend DESC;
```

### Feed for AI spend anomaly detection

```sql
-- Stream of recent booking events for ML pipeline
SELECT
    event_type,
    payload,
    metadata,
    created_at
FROM event_store
WHERE organisation_id = 'org-uuid-here'
  AND event_type IN ('BookingCreated', 'ExpenseEntryCreated')
  AND created_at >= now() - interval '7 days'
ORDER BY created_at ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (write side) | 4 | event_store, stream_snapshot, event_type_registry, idempotency_key |
| Projections (read side) | 6 | proj_trip, proj_booking, proj_expense_report, proj_spend_summary, proj_traveller_location, proj_travel_policy |
| Shared Identity | 2 | organisation, app_user |
| Reference Data | 3 | airline, airport, currency |
| Infrastructure | 1 | projection_checkpoint |
| GDPR | 1 | user_encryption_key |
| **Total** | **17** | Plus projection tables grow as new read models are needed |

---

## Key Design Decisions

1. **Single event store table** — all event types share one table rather than per-aggregate tables. This simplifies cross-aggregate queries (e.g., "all events for this organisation in the last hour"), enables a single CDC stream for downstream consumers, and reduces operational complexity. The `stream_type` + `stream_id` compound index provides efficient per-aggregate replay.

2. **JSONB payloads with schema registry** — event payloads are stored as JSONB for flexibility, but each event type has a registered JSON Schema in `event_type_registry` for validation. This balances schema evolution flexibility with data quality guarantees. GIN indexes on payload enable efficient queries into event data.

3. **Projections are disposable and rebuildable** — every `proj_*` table can be dropped and rebuilt from the event store. The `projection_checkpoint` table tracks how far each projection has been built, enabling incremental updates and full rebuilds. This means read model schemas can be changed freely without data migration.

4. **Snapshot table for replay performance** — for aggregates with many events (e.g., a long-running trip with dozens of booking changes), periodic snapshots avoid replaying hundreds of events on every read. Snapshots are an optimisation, not a requirement — the event store remains authoritative.

5. **Crypto-shredding for GDPR compliance** — PII in event payloads is encrypted with per-user keys stored in an external KMS. When a user exercises their right to erasure, the key is deleted in the KMS and marked as deleted in `user_encryption_key`. The events remain in the store but PII fields become unreadable. This preserves event store immutability while satisfying GDPR Article 17.

6. **Idempotency keys prevent duplicate processing** — commands include an idempotency key that is checked before event creation. This is critical for travel booking where network retries could otherwise create duplicate reservations.

7. **Actor type distinguishes human from AI actions** — `actor_type` can be 'user', 'system', or 'ai_agent', creating an audit trail that distinguishes between human decisions and AI-automated actions (e.g., AI expense coding, AI rebooking suggestions). This is important for compliance review.

8. **Pre-aggregated spend projections** — `proj_spend_summary` materialises spend data into daily/weekly/monthly aggregates by department, cost centre, and category. This avoids expensive real-time aggregation queries on the event store while keeping analytics dashboards fast.

9. **Event metadata carries correlation and causation** — `metadata` JSONB includes `correlation_id` (ties related events across aggregates), `causation_id` (which event caused this one), IP address, and user agent. This enables distributed tracing and forensic investigation.

10. **Fewer tables, more flexibility** — at 17 tables vs. 42 in the normalised model, the event-sourced design is structurally simpler. Complexity moves from schema design to event processing code. New features are added by defining new event types and projections rather than ALTER TABLE migrations.
