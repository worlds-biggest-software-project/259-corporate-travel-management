# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Corporate Travel Management · Created: 2026-05-22

## Philosophy

This model uses normalised relational tables for core identity, booking, and financial data — the entities whose structure is well-defined and where referential integrity matters — while using JSONB columns extensively for domain-specific fields that vary by jurisdiction, supplier, booking type, and organisation configuration. The principle is: **columns for things you query across; JSONB for things you filter within or display**.

Corporate travel management spans dozens of airlines (each with unique ancillary services), hundreds of hotel chains (each with different room type taxonomies), multiple GDS systems with differing data schemas, jurisdiction-specific tax rules, and organisation-specific policy configurations. A fully normalised approach forces premature schema decisions and generates table sprawl for every variant. The JSONB hybrid avoids this by letting the schema expand organically inside well-defined document boundaries.

This is the approach used by modern SaaS platforms like Stripe (charges have a `metadata` JSONB field), Shopify (flexible product attributes), and newer travel platforms like Navan (which must normalise across multiple GDS, NDC, and direct supplier schemas). PostgreSQL's mature JSONB support — with GIN indexes, containment operators, and jsonpath queries — makes this practical without sacrificing query performance.

**Best for:** Rapid MVP development, multi-region deployments with jurisdiction-specific variations, teams that need to iterate quickly on booking and expense schemas without database migrations, and platforms that must normalise data from heterogeneous supplier APIs (GDS, NDC, direct).

**Trade-offs:**
- (+) Fast schema evolution — new fields added to JSONB without migrations
- (+) Handles heterogeneous supplier data naturally (different GDS/NDC schemas normalised into JSONB)
- (+) Jurisdiction-specific policy rules and tax fields without table-per-jurisdiction
- (+) Fewer tables and joins than fully normalised model
- (+) PostgreSQL GIN indexes make JSONB queries performant
- (-) JSONB fields lack database-enforced constraints (must validate in application layer)
- (-) JSONB columns can become "junk drawers" without disciplined documentation
- (-) Complex JSONB queries are harder to optimise than simple column queries
- (-) BI tools and reporting layers may struggle with nested JSONB structures
- (-) No foreign key enforcement inside JSONB — referential integrity is application-managed

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IATA NDC (Schema 24.1) | NDC offer/order responses stored in `booking.supplier_data` JSONB preserving original structure |
| OpenTravel Alliance (OTA 2.0) | Hotel/car OTA responses stored in `booking.supplier_data` JSONB |
| ISO 3166-1 / ISO 3166-2 | `country_code` and `subdivision_code` relational columns on core tables |
| ISO 4217 | `currency_code` relational columns for all monetary fields |
| IATA Airline/Airport Codes | Relational `airline` and `airport` reference tables |
| ISO 31030:2021 | Duty of care data in `traveller_location` table with `context` JSONB for variable risk data |
| PCI DSS v4.0.1 | Payment tokens in relational columns; supplier-specific payment metadata in JSONB |
| GHG Protocol Scope 3 Cat 6 | Carbon data in `booking.sustainability` JSONB with standardised field names |
| GDPR / CCPA | PII isolation: relational columns for core PII, JSONB clearly documented with PII markers |

---

## Core Identity & Organisation

```sql
-- =============================================================
-- ORGANISATION
-- =============================================================
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    country_code    CHAR(2) NOT NULL,                -- ISO 3166-1
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',

    -- Organisation-specific configuration stored as JSONB
    -- avoids config tables and enables per-org customisation
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "default_approval_workflow": "two_level",
    --   "expense_receipt_required_above": 25.00,
    --   "allowed_cabin_classes": ["economy", "premium_economy"],
    --   "preferred_gds": "amadeus",
    --   "ndc_enabled": true,
    --   "sustainability_reporting": true,
    --   "tax_config": {
    --     "US": {"sales_tax": true, "vat": false},
    --     "GB": {"sales_tax": false, "vat": true, "vat_rate": 0.20}
    --   },
    --   "integrations": {
    --     "hr_system": "workday",
    --     "erp": "sap",
    --     "card_processor": "amex"
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- USER
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
    job_title       VARCHAR(200),
    phone           VARCHAR(30),
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- User preferences and variable profile data
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "passport": {
    --     "number": "encrypted:...",
    --     "country": "US",
    --     "expiry": "2029-03-15"
    --   },
    --   "known_traveller_number": "123456789",
    --   "loyalty_programs": [
    --     {"program": "AAdvantage", "number": "AA12345678"},
    --     {"program": "Marriott Bonvoy", "number": "MB987654"}
    --   ],
    --   "travel_preferences": {
    --     "seat": "aisle",
    --     "meal": "vegetarian",
    --     "hotel_room": "non-smoking",
    --     "car_class": "midsize"
    --   },
    --   "emergency_contacts": [
    --     {"name": "Jane Doe", "phone": "+1-555-0123", "relationship": "spouse"}
    --   ],
    --   "accessibility_needs": ["wheelchair_assistance"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_email ON app_user(email);

-- =============================================================
-- ROLE (simplified RBAC — roles are flexible via JSONB permissions)
-- =============================================================
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions:
    -- [
    --   {"resource": "booking", "actions": ["create", "read", "update"]},
    --   {"resource": "expense_report", "actions": ["create", "read", "submit"]},
    --   {"resource": "policy", "actions": ["read"]},
    --   {"resource": "analytics", "actions": ["read"]}
    -- ]
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, organisation_id)
);
```

## Reference Data

```sql
-- =============================================================
-- REFERENCE DATA — relational for query performance
-- =============================================================
CREATE TABLE airline (
    iata_code       CHAR(2) PRIMARY KEY,
    icao_code       CHAR(3),
    name            VARCHAR(200) NOT NULL,
    country_code    CHAR(2),
    is_ndc_enabled  BOOLEAN NOT NULL DEFAULT false,
    alliance        VARCHAR(50),
    metadata        JSONB DEFAULT '{}'
    -- metadata example:
    -- {
    --   "ndc_versions": ["21.3", "24.1"],
    --   "baggage_policy_url": "...",
    --   "loyalty_program": "Flying Blue"
    -- }
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
```

## Travel Policy

```sql
-- =============================================================
-- TRAVEL POLICY (rules stored as JSONB for flexibility)
-- =============================================================
CREATE TABLE travel_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(200) NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    version         INTEGER NOT NULL DEFAULT 1,

    -- Policy rules as a structured JSONB array
    rules           JSONB NOT NULL DEFAULT '[]',
    -- Example rules:
    -- [
    --   {
    --     "id": "rule-001",
    --     "category": "flight",
    --     "type": "cabin_class",
    --     "description": "Economy for flights under 6 hours",
    --     "condition": {"field": "duration_hours", "op": "lt", "value": 6},
    --     "allowed_values": ["economy"],
    --     "severity": "warning",
    --     "jurisdiction": null
    --   },
    --   {
    --     "id": "rule-002",
    --     "category": "flight",
    --     "type": "cabin_class",
    --     "description": "Business class for flights 6+ hours",
    --     "condition": {"field": "duration_hours", "op": "gte", "value": 6},
    --     "allowed_values": ["economy", "premium_economy", "business"],
    --     "severity": "block",
    --     "jurisdiction": null
    --   },
    --   {
    --     "id": "rule-003",
    --     "category": "hotel",
    --     "type": "max_rate",
    --     "description": "Max hotel rate by city tier",
    --     "thresholds": {
    --       "tier_1": {"cities": ["NYC", "SFO", "LHR"], "max_rate": 350, "currency": "USD"},
    --       "tier_2": {"cities": ["CHI", "DFW", "CDG"], "max_rate": 250, "currency": "USD"},
    --       "default": {"max_rate": 200, "currency": "USD"}
    --     },
    --     "severity": "warning"
    --   },
    --   {
    --     "id": "rule-004",
    --     "category": "general",
    --     "type": "advance_booking",
    --     "description": "Book flights 14+ days in advance",
    --     "condition": {"field": "days_before_departure", "op": "gte", "value": 14},
    --     "severity": "info"
    --   }
    -- ]

    -- Assignment scope
    applies_to      JSONB NOT NULL DEFAULT '{"type": "organisation"}',
    -- Examples:
    -- {"type": "organisation"}  — applies to all
    -- {"type": "department", "ids": ["dept-uuid-1", "dept-uuid-2"]}
    -- {"type": "role", "names": ["executive", "sales"]}
    -- {"type": "user", "ids": ["user-uuid-1"]}

    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_org ON travel_policy(organisation_id);
CREATE INDEX idx_policy_effective ON travel_policy(effective_from, effective_to);
CREATE INDEX idx_policy_rules ON travel_policy USING GIN (rules jsonb_path_ops);
```

## Supplier Agreements

```sql
-- =============================================================
-- SUPPLIER AGREEMENTS (flexible terms in JSONB)
-- =============================================================
CREATE TABLE supplier_agreement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    supplier_type   VARCHAR(30) NOT NULL,
    supplier_name   VARCHAR(200) NOT NULL,
    supplier_code   VARCHAR(20),                     -- IATA or OTA code
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',

    -- Agreement terms vary significantly by supplier type
    terms           JSONB NOT NULL DEFAULT '{}',
    -- Example for airline:
    -- {
    --   "discount_type": "percentage",
    --   "discount_pct": 12.5,
    --   "fare_classes": ["Y", "B", "M"],
    --   "routes": [{"from": "JFK", "to": "LHR"}, {"from": "SFO", "to": "NRT"}],
    --   "min_annual_segments": 500,
    --   "corporate_fare_code": "ACME2026"
    -- }
    --
    -- Example for hotel chain:
    -- {
    --   "discount_type": "fixed_rate",
    --   "rates": {
    --     "NYC": {"standard": 289, "suite": 450, "currency": "USD"},
    --     "LDN": {"standard": 225, "suite": 380, "currency": "GBP"}
    --   },
    --   "loyalty_program_number": "MB-CORP-12345",
    --   "cancellation_policy": "24h_free"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_org ON supplier_agreement(organisation_id);
CREATE INDEX idx_supplier_type ON supplier_agreement(supplier_type);
CREATE INDEX idx_supplier_terms ON supplier_agreement USING GIN (terms jsonb_path_ops);
```

## Trip & Booking

```sql
-- =============================================================
-- TRAVEL REQUEST
-- =============================================================
CREATE TABLE travel_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    requester_id    UUID NOT NULL REFERENCES app_user(id),
    trip_name       VARCHAR(300) NOT NULL,
    purpose         TEXT,
    departure_date  DATE NOT NULL,
    return_date     DATE,
    estimated_cost  DECIMAL(12,2),
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    cost_centre     VARCHAR(50),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,

    -- Approval chain state
    approvals       JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"step": 1, "approver_id": "uuid", "decision": "approved", "at": "2026-06-01T10:00:00Z", "comments": "OK"},
    --   {"step": 2, "approver_id": "uuid", "decision": "pending"}
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_request_org ON travel_request(organisation_id);
CREATE INDEX idx_request_requester ON travel_request(requester_id);
CREATE INDEX idx_request_status ON travel_request(status);

-- =============================================================
-- TRIP
-- =============================================================
CREATE TABLE trip (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    travel_request_id UUID REFERENCES travel_request(id),
    primary_traveller_id UUID NOT NULL REFERENCES app_user(id),
    trip_name       VARCHAR(300) NOT NULL,
    purpose         TEXT,
    departure_date  DATE NOT NULL,
    return_date     DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'planned',
    travellers      UUID[] NOT NULL DEFAULT '{}',    -- array of user IDs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trip_org ON trip(organisation_id);
CREATE INDEX idx_trip_traveller ON trip(primary_traveller_id);
CREATE INDEX idx_trip_dates ON trip(departure_date, return_date);
CREATE INDEX idx_trip_status ON trip(status);

-- =============================================================
-- BOOKING (unified table for all booking types)
-- =============================================================
CREATE TABLE booking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trip_id         UUID NOT NULL REFERENCES trip(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    booking_type    VARCHAR(30) NOT NULL,            -- 'flight', 'hotel', 'car', 'rail', 'other'
    booking_source  VARCHAR(30) NOT NULL,            -- 'gds_amadeus', 'gds_sabre', 'ndc', 'direct', 'agent'
    status          VARCHAR(30) NOT NULL DEFAULT 'confirmed',
    gds_pnr         VARCHAR(20),
    supplier_confirmation VARCHAR(50),

    -- Financial
    total_amount    DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    base_fare       DECIMAL(12,2),
    taxes_fees      DECIMAL(12,2),

    -- Policy compliance
    policy_compliant BOOLEAN NOT NULL DEFAULT true,
    policy_check    JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "policy_id": "uuid",
    --   "checked_at": "2026-06-01T09:00:00Z",
    --   "violations": [
    --     {"rule_id": "rule-001", "description": "Business class on 4hr flight", "severity": "warning"}
    --   ],
    --   "overridden": true,
    --   "override_reason": "Client requirement"
    -- }

    -- Booking-type-specific details in JSONB
    -- This is the key hybrid design: a single booking table with
    -- type-specific details in a flexible JSONB column
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example for flight:
    -- {
    --   "segments": [
    --     {
    --       "segment_order": 1,
    --       "airline_iata": "BA",
    --       "flight_number": "BA178",
    --       "origin_iata": "JFK",
    --       "destination_iata": "LHR",
    --       "departure_at": "2026-06-15T18:30:00Z",
    --       "arrival_at": "2026-06-16T06:30:00Z",
    --       "cabin_class": "business",
    --       "fare_class": "J",
    --       "fare_basis": "JFLEX",
    --       "seat": "14A",
    --       "baggage": "2x23kg",
    --       "ticket_number": "125-2345678901",
    --       "is_ndc": true,
    --       "ndc_offer_id": "NDC-BA-2026-OFR-789"
    --     },
    --     {
    --       "segment_order": 2,
    --       "airline_iata": "BA",
    --       "flight_number": "BA179",
    --       "origin_iata": "LHR",
    --       "destination_iata": "JFK",
    --       "departure_at": "2026-06-19T10:00:00Z",
    --       "arrival_at": "2026-06-19T13:00:00Z",
    --       "cabin_class": "business",
    --       "fare_class": "J",
    --       "fare_basis": "JFLEX",
    --       "seat": null,
    --       "baggage": "2x23kg"
    --     }
    --   ],
    --   "ancillaries": [
    --     {"type": "lounge_access", "amount": 0, "included": true}
    --   ]
    -- }
    --
    -- Example for hotel:
    -- {
    --   "property_name": "Hilton London Canary Wharf",
    --   "property_code": "LONHICW",
    --   "chain": "Hilton",
    --   "address": "Marsh Wall, London E14 9SH, UK",
    --   "check_in": "2026-06-15",
    --   "check_out": "2026-06-19",
    --   "num_nights": 4,
    --   "room_type": "King Executive",
    --   "rate_per_night": 289.00,
    --   "rate_currency": "GBP",
    --   "meal_plan": "breakfast_included",
    --   "loyalty_number": "HH987654321",
    --   "cancellation_deadline": "2026-06-14T14:00:00Z"
    -- }
    --
    -- Example for car rental:
    -- {
    --   "company": "Hertz",
    --   "pickup_location": "London Heathrow Airport",
    --   "dropoff_location": "London Heathrow Airport",
    --   "pickup_at": "2026-06-15T08:00:00Z",
    --   "dropoff_at": "2026-06-19T16:00:00Z",
    --   "vehicle_class": "midsize",
    --   "vehicle_model": "Volkswagen Golf or similar",
    --   "rate_per_day": 65.00,
    --   "insurance": "collision_damage_waiver",
    --   "mileage": "unlimited"
    -- }

    -- Raw supplier response preserved for debugging / compliance
    supplier_data   JSONB DEFAULT NULL,
    -- Stores the raw GDS/NDC/supplier API response for this booking.
    -- Never displayed to users; used for dispute resolution and compliance.

    -- Sustainability data
    sustainability  JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "co2e_kg": 1420.5,
    --   "calculation_method": "distance",
    --   "distance_km": 5540,
    --   "emission_factor": 0.256,
    --   "emission_factor_source": "DEFRA 2026",
    --   "scope": "3.6",
    --   "offset_available": true,
    --   "offset_cost_usd": 28.41
    -- }

    booked_by       UUID REFERENCES app_user(id),
    booked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    cancelled_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_booking_trip ON booking(trip_id);
CREATE INDEX idx_booking_org ON booking(organisation_id);
CREATE INDEX idx_booking_type ON booking(booking_type);
CREATE INDEX idx_booking_status ON booking(status);
CREATE INDEX idx_booking_pnr ON booking(gds_pnr);
CREATE INDEX idx_booking_details ON booking USING GIN (details jsonb_path_ops);

-- Query flight segments across bookings:
-- SELECT * FROM booking
-- WHERE booking_type = 'flight'
--   AND details @> '{"segments": [{"origin_iata": "JFK"}]}';

-- Query hotel bookings by city:
-- SELECT * FROM booking
-- WHERE booking_type = 'hotel'
--   AND details->>'chain' = 'Hilton';
```

## Expense Management

```sql
-- =============================================================
-- EXPENSE REPORT
-- =============================================================
CREATE TABLE expense_report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    trip_id         UUID REFERENCES trip(id),
    report_name     VARCHAR(300) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    total_amount    DECIMAL(12,2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    amount_due_employee DECIMAL(12,2) DEFAULT 0,
    amount_due_company  DECIMAL(12,2) DEFAULT 0,
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,

    -- Approval chain
    approvals       JSONB NOT NULL DEFAULT '[]',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_expense_org ON expense_report(organisation_id);
CREATE INDEX idx_expense_user ON expense_report(user_id);
CREATE INDEX idx_expense_status ON expense_report(status);

-- =============================================================
-- EXPENSE ENTRY
-- =============================================================
CREATE TABLE expense_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES expense_report(id) ON DELETE CASCADE,
    booking_id      UUID REFERENCES booking(id),
    expense_type    VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    vendor_name     VARCHAR(200),
    transaction_date DATE NOT NULL,
    amount          DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    exchange_rate   DECIMAL(12,6),
    amount_in_report_currency DECIMAL(12,2),
    payment_type    VARCHAR(30) NOT NULL,
    gl_account      VARCHAR(50),
    cost_centre     VARCHAR(50),
    is_personal     BOOLEAN NOT NULL DEFAULT false,
    is_policy_compliant BOOLEAN NOT NULL DEFAULT true,

    -- AI-extracted data and variable fields
    ai_extraction   JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "ocr_text": "The Ivy Restaurant...",
    --   "extracted_vendor": "The Ivy",
    --   "extracted_amount": 85.50,
    --   "extracted_currency": "GBP",
    --   "extracted_date": "2026-06-16",
    --   "confidence": 0.95,
    --   "suggested_gl": "6200-MEAL",
    --   "suggested_type": "meal",
    --   "line_items": [
    --     {"description": "Main course", "amount": 42.00},
    --     {"description": "Wine", "amount": 35.00},
    --     {"description": "Service charge", "amount": 8.50}
    --   ]
    -- }

    -- Allocations as JSONB instead of separate table
    allocations     JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   {"cost_centre": "ENG-001", "gl_account": "6200", "pct": 60, "amount": 51.30},
    --   {"cost_centre": "SALES-002", "gl_account": "6200", "pct": 40, "amount": 34.20}
    -- ]

    -- Tax details vary by jurisdiction
    tax_details     JSONB DEFAULT '{}',
    -- Example (UK):
    -- {"vat_rate": 0.20, "vat_amount": 14.25, "net_amount": 71.25, "vat_number": "GB123456789"}
    -- Example (US):
    -- {"sales_tax_rate": 0.08875, "sales_tax_amount": 7.59, "state": "NY"}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entry_report ON expense_entry(report_id);
CREATE INDEX idx_entry_type ON expense_entry(expense_type);
CREATE INDEX idx_entry_date ON expense_entry(transaction_date);

-- =============================================================
-- RECEIPT
-- =============================================================
CREATE TABLE receipt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    entry_id        UUID REFERENCES expense_entry(id),
    file_url        TEXT NOT NULL,
    file_name       VARCHAR(300),
    mime_type       VARCHAR(100),
    file_size_bytes BIGINT,
    capture_source  VARCHAR(30),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_receipt_user ON receipt(user_id);
CREATE INDEX idx_receipt_entry ON receipt(entry_id);
```

## Payment & Corporate Card

```sql
-- =============================================================
-- PAYMENT METHOD (PCI DSS — tokenized)
-- =============================================================
CREATE TABLE payment_method (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES app_user(id),
    method_type     VARCHAR(30) NOT NULL,
    card_token      VARCHAR(255),
    last_four       CHAR(4),
    card_network    VARCHAR(20),
    expiry_month    SMALLINT,
    expiry_year     SMALLINT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB DEFAULT '{}',
    -- Example: {"billing_address": {...}, "processor": "stripe", "processor_id": "pm_xxx"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_org ON payment_method(organisation_id);

-- =============================================================
-- CARD TRANSACTION
-- =============================================================
CREATE TABLE card_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_method_id UUID NOT NULL REFERENCES payment_method(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    transaction_date DATE NOT NULL,
    merchant_name   VARCHAR(300),
    merchant_category_code VARCHAR(10),
    amount          DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    is_reconciled   BOOLEAN NOT NULL DEFAULT false,
    reconciled_entry_id UUID REFERENCES expense_entry(id),
    external_txn_id VARCHAR(100),

    -- Processor-specific data varies by card network
    processor_data  JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "original_amount": 85.50,
    --   "original_currency": "GBP",
    --   "exchange_rate": 1.27,
    --   "merchant_city": "London",
    --   "merchant_country": "GB",
    --   "auth_code": "ABC123"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_card_txn_user ON card_transaction(user_id);
CREATE INDEX idx_card_txn_date ON card_transaction(transaction_date);
CREATE INDEX idx_card_txn_reconciled ON card_transaction(is_reconciled);
```

## Duty of Care

```sql
-- =============================================================
-- TRAVELLER LOCATION (ISO 31030 aligned)
-- =============================================================
CREATE TABLE traveller_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    trip_id         UUID REFERENCES trip(id),
    country_code    CHAR(2) NOT NULL,
    city            VARCHAR(200),
    latitude        DECIMAL(9,6),
    longitude       DECIMAL(9,6),
    location_source VARCHAR(30) NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL,

    -- Variable context depending on source
    context         JSONB DEFAULT '{}',
    -- Example (from itinerary):
    -- {"booking_id": "uuid", "segment": "JFK-LHR", "status": "en_route"}
    -- Example (from check-in):
    -- {"hotel": "Hilton Canary Wharf", "room": "1204"}
    -- Example (from GPS):
    -- {"accuracy_m": 15, "altitude_m": 42, "speed_kmh": 0}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_location_user ON traveller_location(user_id);
CREATE INDEX idx_location_time ON traveller_location(recorded_at);
CREATE INDEX idx_location_country ON traveller_location(country_code);

-- =============================================================
-- RISK ALERT
-- =============================================================
CREATE TABLE risk_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    country_code    CHAR(2),
    latitude        DECIMAL(9,6),
    longitude       DECIMAL(9,6),
    radius_km       DECIMAL(8,2),
    source          VARCHAR(100),
    effective_from  TIMESTAMPTZ NOT NULL,
    effective_to    TIMESTAMPTZ,

    -- Source-specific data varies by provider
    source_data     JSONB DEFAULT '{}',
    -- Example (Everbridge):
    -- {"everbridge_id": "EVB-2026-1234", "category": "civil_unrest", "impact_zones": [...]}
    -- Example (weather):
    -- {"storm_name": "...", "wind_speed_kmh": 120, "forecast_track": [...]}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_country ON risk_alert(country_code);
CREATE INDEX idx_alert_severity ON risk_alert(severity);
CREATE INDEX idx_alert_effective ON risk_alert(effective_from, effective_to);

-- =============================================================
-- TRAVELLER ALERT NOTIFICATION
-- =============================================================
CREATE TABLE traveller_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_id        UUID NOT NULL REFERENCES risk_alert(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    trip_id         UUID REFERENCES trip(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    notified_at     TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    response        JSONB DEFAULT '{}',
    -- Example: {"confirmed_safe": true, "location": "hotel", "needs_assistance": false}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_traveller_alert_user ON traveller_alert(user_id);
```

## Audit Log

```sql
-- =============================================================
-- AUDIT LOG
-- =============================================================
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,
    changes         JSONB DEFAULT '{}',
    metadata        JSONB DEFAULT '{}',
    -- Example metadata:
    -- {"ip": "192.168.1.1", "user_agent": "...", "source": "web", "ai_agent": false}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Organisation | 2 | organisation, app_user |
| RBAC | 2 | role, user_role |
| Reference Data | 3 | airline, airport, currency |
| Travel Policy | 1 | travel_policy (rules as JSONB) |
| Supplier Management | 1 | supplier_agreement (terms as JSONB) |
| Trip & Booking | 3 | travel_request, trip, booking (all types unified) |
| Expense Management | 3 | expense_report, expense_entry, receipt |
| Payment & Card | 2 | payment_method, card_transaction |
| Duty of Care | 3 | traveller_location, risk_alert, traveller_alert |
| Audit | 1 | audit_log |
| **Total** | **21** | |

---

## Key Design Decisions

1. **Unified booking table with type-specific JSONB `details`** — instead of separate `flight_segment`, `hotel_booking`, `car_rental` tables (3 tables in the normalised model), a single `booking` table uses the `details` JSONB column for type-specific data. The `booking_type` column distinguishes flight/hotel/car/rail. This reduces table count and eliminates the need for new tables when adding rail, ferry, or other transport modes.

2. **Policy rules as JSONB arrays** — instead of separate `travel_policy`, `policy_rule`, and `policy_assignment` tables (3 tables), a single `travel_policy` table stores rules and assignments as JSONB. This makes policies self-contained documents that are easy to version, clone, and compare. The GIN index enables queries like "find all policies with cabin_class rules."

3. **Approval chains embedded in JSONB** — both `travel_request` and `expense_report` carry their approval history in an `approvals` JSONB array rather than a separate `approval_record` table. This simplifies querying ("show me the full approval chain for this request") and keeps the approval context co-located with the entity it belongs to.

4. **`supplier_data` preserves raw API responses** — the `booking.supplier_data` column stores the complete GDS/NDC/supplier API response for each booking. This is invaluable for dispute resolution, compliance audits, and debugging data normalisation issues. It also future-proofs the platform: as NDC schema versions evolve, the raw responses are always available for re-processing.

5. **AI extraction data co-located with expense entries** — the `expense_entry.ai_extraction` JSONB column stores OCR results, confidence scores, and AI-suggested GL codes alongside the entry. This keeps AI provenance traceable without requiring a separate table, and allows the application to show users what the AI extracted vs. what was manually overridden.

6. **Tax details as jurisdiction-specific JSONB** — `expense_entry.tax_details` handles the significant variation in tax structures across jurisdictions (VAT in UK/EU, sales tax in US, GST in Australia) without requiring a separate tax table per jurisdiction or a complex normalised tax schema.

7. **Traveller preferences and loyalty programs in user profile JSONB** — rather than separate `loyalty_program`, `travel_preference`, and `emergency_contact` tables, all variable user profile data lives in `app_user.profile`. This dramatically reduces the number of tables and simplifies the user profile API.

8. **Organisation settings as JSONB** — `organisation.settings` contains all configurable organisation preferences (default GDS, NDC support, tax configuration, integration settings). This avoids a `setting_key/setting_value` EAV pattern and makes the full configuration available in a single read.

9. **21 tables vs. 42 in normalised model** — the JSONB hybrid halves the table count while preserving relational integrity for all core relationships (organisation -> user -> trip -> booking -> expense). The reduction comes from collapsing type-specific subtables, configuration tables, and junction tables into JSONB columns.

10. **GIN indexes on all major JSONB columns** — PostgreSQL GIN indexes with `jsonb_path_ops` enable efficient containment queries (`@>`) on booking details, policy rules, and supplier terms. This makes JSONB columns queryable at near-relational performance for the most common access patterns.
