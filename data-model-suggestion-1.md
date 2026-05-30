# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Corporate Travel Management · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design where every real-world concept — travellers, trips, bookings, segments, policies, expenses, approvals — gets its own table with explicit foreign-key relationships. Reference data (airlines, airports, currencies, jurisdictions) is stored in dedicated lookup tables aligned with international standards (IATA, ICAO, ISO 3166, ISO 4217). Junction tables handle many-to-many relationships such as traveller-to-trip assignments, policy-to-rule mappings, and expense-to-allocation splits.

This is the architecture that enterprise ERP systems (SAP, Oracle) and legacy TMC platforms use. It provides maximum data integrity through database-enforced constraints, supports complex cross-entity reporting (spend by department by supplier by jurisdiction), and makes regulatory compliance straightforward because every relationship is explicit and auditable via standard SQL joins.

The normalized approach is best for organisations that prioritise data integrity, complex analytical queries, and regulatory compliance over schema flexibility or write throughput. It maps cleanly to the SAP Concur and TravelPerk API entity models documented in their developer portals.

**Best for:** Enterprises requiring complex cross-entity reporting, strong referential integrity, and alignment with established travel industry data standards.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Complex analytical queries are natural (joins across any dimension)
- (+) Standards-aligned reference data enables interoperability with GDS/NDC systems
- (+) Well-understood by most engineering teams and BI tools
- (-) High table count increases migration and schema evolution complexity
- (-) Adding jurisdiction-specific or supplier-specific fields requires ALTER TABLE or new tables
- (-) Write-heavy workflows (real-time booking updates) may face contention on heavily-indexed tables
- (-) Temporal queries ("what was the policy on date X?") require additional versioning tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IATA NDC (Schema 24.1) | Offer/Order entity structure informs `booking`, `booking_segment`, and `offer` table design |
| OpenTravel Alliance (OTA 2.0) | Hotel/car reservation schemas inform `hotel_booking` and `car_booking` segment fields |
| ISO 3166-1 / ISO 3166-2 | `jurisdiction` reference table uses alpha-2 country codes and subdivision codes |
| ISO 4217 | `currency` reference table uses standard 3-letter currency codes for all monetary fields |
| IATA Airline Designator | `airline` reference table uses 2-character IATA codes and 3-character ICAO codes |
| IATA Airport/Location Code | `airport` reference table uses 3-letter IATA and 4-letter ICAO location identifiers |
| ISO 31030:2021 | Duty of care entities (`traveller_location`, `risk_alert`, `emergency_contact`) align with TRM framework |
| PCI DSS v4.0.1 | Payment card data stored in `payment_method` table with tokenized card references, never raw PANs |
| GHG Protocol Scope 3 Cat 6 | `carbon_emission` table captures distance, mode, emission factor, and CO2e per segment |
| SCIM 2.0 (RFC 7643/7644) | `user` table schema supports SCIM-compatible provisioning/deprovisioning from HR systems |
| GDPR / CCPA | PII fields marked in schema; `data_retention_policy` and `consent_record` tables support compliance |

---

## Core Identity & Multi-Tenancy

```sql
-- =============================================================
-- TENANT / ORGANISATION
-- =============================================================
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    tax_id          VARCHAR(50),               -- VAT / EIN
    industry        VARCHAR(100),
    country_code    CHAR(2) NOT NULL,           -- ISO 3166-1 alpha-2
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisation_country ON organisation(country_code);

-- =============================================================
-- USER / EMPLOYEE
-- =============================================================
CREATE TABLE app_user (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    email               VARCHAR(320) NOT NULL,
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    employee_id         VARCHAR(50),             -- internal HR identifier
    department_id       UUID REFERENCES department(id),
    cost_centre_id      UUID REFERENCES cost_centre(id),
    job_title           VARCHAR(200),
    phone               VARCHAR(30),
    date_of_birth       DATE,                    -- for duty of care / passport
    nationality         CHAR(2),                 -- ISO 3166-1
    passport_number     VARCHAR(50),             -- encrypted at rest
    passport_expiry     DATE,
    preferred_language  CHAR(5) DEFAULT 'en',    -- BCP 47
    is_active           BOOLEAN NOT NULL DEFAULT true,
    scim_external_id    VARCHAR(255),            -- SCIM 2.0 externalId
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_department ON app_user(department_id);
CREATE INDEX idx_user_email ON app_user(email);

-- =============================================================
-- DEPARTMENT
-- =============================================================
CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(200) NOT NULL,
    parent_id       UUID REFERENCES department(id), -- hierarchy
    manager_id      UUID REFERENCES app_user(id),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_department_org ON department(organisation_id);
CREATE INDEX idx_department_parent ON department(parent_id);

-- =============================================================
-- COST CENTRE
-- =============================================================
CREATE TABLE cost_centre (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    gl_account      VARCHAR(50),                -- general ledger mapping
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, code)
);
```

## RBAC & Permissions

```sql
-- =============================================================
-- ROLE-BASED ACCESS CONTROL (tenant-scoped)
-- =============================================================
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),  -- NULL = system-wide role
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,     -- built-in vs custom
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        VARCHAR(100) NOT NULL,     -- e.g. 'booking', 'expense_report', 'policy'
    action          VARCHAR(50) NOT NULL,      -- e.g. 'create', 'read', 'approve', 'delete'
    description     TEXT,
    UNIQUE(resource, action)
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, organisation_id)
);

CREATE INDEX idx_user_role_org ON user_role(organisation_id);
```

## Reference Data

```sql
-- =============================================================
-- REFERENCE DATA — Airlines, Airports, Currencies, Countries
-- =============================================================
CREATE TABLE airline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    iata_code       CHAR(2) NOT NULL UNIQUE,       -- IATA 2-letter designator
    icao_code       CHAR(3),                        -- ICAO 3-letter code
    name            VARCHAR(200) NOT NULL,
    country_code    CHAR(2),                         -- ISO 3166-1
    is_ndc_enabled  BOOLEAN NOT NULL DEFAULT false,  -- supports IATA NDC
    alliance        VARCHAR(50),                     -- Star Alliance, oneworld, SkyTeam
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE airport (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    iata_code       CHAR(3) NOT NULL UNIQUE,       -- IATA 3-letter location code
    icao_code       CHAR(4),                        -- ICAO 4-letter code
    name            VARCHAR(300) NOT NULL,
    city            VARCHAR(200),
    country_code    CHAR(2) NOT NULL,                -- ISO 3166-1
    subdivision_code VARCHAR(6),                     -- ISO 3166-2
    latitude        DECIMAL(9,6),
    longitude       DECIMAL(9,6),
    timezone        VARCHAR(50)
);

CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,             -- ISO 4217
    name            VARCHAR(100) NOT NULL,
    numeric_code    CHAR(3),
    minor_units     SMALLINT NOT NULL DEFAULT 2
);

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,                -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),                     -- ISO 3166-2 (nullable for country-level)
    name            VARCHAR(200) NOT NULL,
    parent_id       UUID REFERENCES jurisdiction(id),
    level           SMALLINT NOT NULL DEFAULT 0       -- 0=country, 1=state/province, 2=city
);

CREATE INDEX idx_jurisdiction_country ON jurisdiction(country_code);

CREATE TABLE hotel_chain (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    code            VARCHAR(10),                     -- OTA chain code
    loyalty_program VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE car_rental_company (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    code            VARCHAR(10),                     -- OTA vendor code
    is_active       BOOLEAN NOT NULL DEFAULT true
);
```

## Travel Policy

```sql
-- =============================================================
-- TRAVEL POLICY
-- =============================================================
CREATE TABLE travel_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    effective_from  DATE NOT NULL,
    effective_to    DATE,                           -- NULL = currently active
    is_default      BOOLEAN NOT NULL DEFAULT false,
    version         INTEGER NOT NULL DEFAULT 1,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_org ON travel_policy(organisation_id);
CREATE INDEX idx_policy_effective ON travel_policy(effective_from, effective_to);

CREATE TABLE policy_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES travel_policy(id) ON DELETE CASCADE,
    category        VARCHAR(50) NOT NULL,           -- 'flight', 'hotel', 'car', 'meal', 'general'
    rule_type       VARCHAR(50) NOT NULL,           -- 'max_amount', 'cabin_class', 'advance_booking', 'preferred_supplier'
    description     TEXT NOT NULL,
    condition_field VARCHAR(100),                    -- field this rule checks
    operator        VARCHAR(20),                     -- 'lte', 'gte', 'eq', 'in', 'not_in'
    threshold_value VARCHAR(255),                    -- value to compare against
    currency_code   CHAR(3) REFERENCES currency(code),
    jurisdiction_id UUID REFERENCES jurisdiction(id), -- jurisdiction-specific rules
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning', -- 'block', 'warning', 'info'
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rule_policy ON policy_rule(policy_id);
CREATE INDEX idx_rule_category ON policy_rule(category);

-- Which user groups does a policy apply to?
CREATE TABLE policy_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES travel_policy(id) ON DELETE CASCADE,
    assignment_type VARCHAR(30) NOT NULL,            -- 'organisation', 'department', 'role', 'user'
    target_id       UUID NOT NULL,                   -- references org, dept, role, or user
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_assignment_target ON policy_assignment(assignment_type, target_id);
```

## Supplier Management

```sql
-- =============================================================
-- PREFERRED SUPPLIERS & NEGOTIATED RATES
-- =============================================================
CREATE TABLE supplier_agreement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    supplier_type   VARCHAR(30) NOT NULL,           -- 'airline', 'hotel', 'car_rental', 'rail'
    supplier_ref_id UUID NOT NULL,                   -- FK to airline, hotel_chain, or car_rental_company
    agreement_name  VARCHAR(200) NOT NULL,
    contract_number VARCHAR(100),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    discount_pct    DECIMAL(5,2),
    negotiated_rate DECIMAL(12,2),
    currency_code   CHAR(3) REFERENCES currency(code),
    min_volume      INTEGER,                         -- minimum booking volume per period
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_org ON supplier_agreement(organisation_id);
CREATE INDEX idx_supplier_type ON supplier_agreement(supplier_type);
```

## Trip & Booking

```sql
-- =============================================================
-- TRAVEL REQUEST (pre-trip approval)
-- =============================================================
CREATE TABLE travel_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    requester_id    UUID NOT NULL REFERENCES app_user(id),
    trip_name       VARCHAR(300) NOT NULL,
    purpose         TEXT,
    departure_date  DATE NOT NULL,
    return_date     DATE,
    destination     VARCHAR(200),
    estimated_cost  DECIMAL(12,2),
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    policy_id       UUID REFERENCES travel_policy(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- status: draft -> submitted -> approved -> rejected -> cancelled
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_request_org ON travel_request(organisation_id);
CREATE INDEX idx_request_requester ON travel_request(requester_id);
CREATE INDEX idx_request_status ON travel_request(status);
CREATE INDEX idx_request_dates ON travel_request(departure_date, return_date);

-- =============================================================
-- TRIP (container for bookings)
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
    -- status: planned -> booked -> in_progress -> completed -> cancelled
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trip_org ON trip(organisation_id);
CREATE INDEX idx_trip_traveller ON trip(primary_traveller_id);
CREATE INDEX idx_trip_dates ON trip(departure_date, return_date);
CREATE INDEX idx_trip_status ON trip(status);

-- Multiple travellers can share a trip
CREATE TABLE trip_traveller (
    trip_id         UUID NOT NULL REFERENCES trip(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role            VARCHAR(30) NOT NULL DEFAULT 'traveller', -- 'traveller', 'organiser', 'guest'
    PRIMARY KEY (trip_id, user_id)
);

-- =============================================================
-- BOOKING (individual reservations within a trip)
-- =============================================================
CREATE TABLE booking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trip_id         UUID NOT NULL REFERENCES trip(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    booking_type    VARCHAR(30) NOT NULL,           -- 'flight', 'hotel', 'car', 'rail', 'other'
    booking_source  VARCHAR(30) NOT NULL,           -- 'gds', 'ndc', 'direct', 'agent'
    gds_pnr         VARCHAR(20),                    -- GDS Passenger Name Record locator
    supplier_confirmation VARCHAR(50),              -- supplier's own confirmation code
    status          VARCHAR(30) NOT NULL DEFAULT 'confirmed',
    -- status: pending -> confirmed -> ticketed -> checked_in -> completed -> cancelled -> refunded
    total_amount    DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    base_fare       DECIMAL(12,2),
    taxes_fees      DECIMAL(12,2),
    policy_compliant BOOLEAN NOT NULL DEFAULT true,
    policy_violations TEXT[],                        -- array of violated rule descriptions
    booked_by       UUID REFERENCES app_user(id),
    booked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    cancelled_at    TIMESTAMPTZ,
    cancellation_fee DECIMAL(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_booking_trip ON booking(trip_id);
CREATE INDEX idx_booking_org ON booking(organisation_id);
CREATE INDEX idx_booking_type ON booking(booking_type);
CREATE INDEX idx_booking_status ON booking(status);
CREATE INDEX idx_booking_pnr ON booking(gds_pnr);

-- =============================================================
-- FLIGHT SEGMENT (specific to air bookings)
-- =============================================================
CREATE TABLE flight_segment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    segment_order   SMALLINT NOT NULL,
    airline_iata    CHAR(2) NOT NULL REFERENCES airline(iata_code),
    flight_number   VARCHAR(10) NOT NULL,
    origin_iata     CHAR(3) NOT NULL,               -- IATA airport code
    destination_iata CHAR(3) NOT NULL,               -- IATA airport code
    departure_at    TIMESTAMPTZ NOT NULL,
    arrival_at      TIMESTAMPTZ NOT NULL,
    cabin_class     VARCHAR(20) NOT NULL,            -- 'economy', 'premium_economy', 'business', 'first'
    fare_class      CHAR(2),                         -- booking class code
    fare_basis      VARCHAR(20),                     -- fare basis code
    ticket_number   VARCHAR(20),                     -- e-ticket number (IATA 13-digit)
    seat_number     VARCHAR(5),
    baggage_allowance VARCHAR(20),
    is_ndc_booking  BOOLEAN NOT NULL DEFAULT false,  -- booked via IATA NDC
    ndc_offer_id    VARCHAR(100),                    -- NDC OfferID reference
    status          VARCHAR(30) NOT NULL DEFAULT 'confirmed',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_flight_booking ON flight_segment(booking_id);
CREATE INDEX idx_flight_departure ON flight_segment(departure_at);
CREATE INDEX idx_flight_route ON flight_segment(origin_iata, destination_iata);

-- =============================================================
-- HOTEL BOOKING DETAIL (specific to hotel bookings)
-- =============================================================
CREATE TABLE hotel_booking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    hotel_chain_id  UUID REFERENCES hotel_chain(id),
    property_name   VARCHAR(300) NOT NULL,
    property_code   VARCHAR(20),                     -- OTA property code
    address         TEXT,
    city            VARCHAR(200),
    country_code    CHAR(2) NOT NULL,                -- ISO 3166-1
    check_in_date   DATE NOT NULL,
    check_out_date  DATE NOT NULL,
    room_type       VARCHAR(100),
    rate_per_night  DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    num_nights      SMALLINT NOT NULL,
    loyalty_number  VARCHAR(50),
    confirmation_number VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hotel_booking ON hotel_booking(booking_id);
CREATE INDEX idx_hotel_dates ON hotel_booking(check_in_date, check_out_date);

-- =============================================================
-- CAR RENTAL DETAIL (specific to car bookings)
-- =============================================================
CREATE TABLE car_rental (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    rental_company_id UUID REFERENCES car_rental_company(id),
    pickup_location VARCHAR(200) NOT NULL,
    dropoff_location VARCHAR(200),
    pickup_at       TIMESTAMPTZ NOT NULL,
    dropoff_at      TIMESTAMPTZ NOT NULL,
    vehicle_class   VARCHAR(50),                     -- 'economy', 'compact', 'midsize', 'suv', etc.
    rate_per_day    DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    insurance_included BOOLEAN NOT NULL DEFAULT false,
    confirmation_number VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_car_booking ON car_rental(booking_id);
```

## Approval Workflow

```sql
-- =============================================================
-- APPROVAL WORKFLOW (state machine pattern)
-- =============================================================
CREATE TABLE approval_workflow (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(200) NOT NULL,
    entity_type     VARCHAR(30) NOT NULL,            -- 'travel_request', 'expense_report', 'booking'
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_step (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID NOT NULL REFERENCES approval_workflow(id) ON DELETE CASCADE,
    step_order      SMALLINT NOT NULL,
    approver_type   VARCHAR(30) NOT NULL,            -- 'manager', 'role', 'specific_user', 'cost_centre_owner'
    approver_ref_id UUID,                            -- user or role id
    auto_approve_below DECIMAL(12,2),                -- auto-approve if below this amount
    currency_code   CHAR(3) REFERENCES currency(code),
    escalation_hours INTEGER DEFAULT 48,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_step_workflow ON approval_step(workflow_id);

CREATE TABLE approval_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(30) NOT NULL,            -- 'travel_request', 'expense_report'
    entity_id       UUID NOT NULL,                   -- FK to the entity being approved
    step_id         UUID NOT NULL REFERENCES approval_step(id),
    approver_id     UUID NOT NULL REFERENCES app_user(id),
    decision        VARCHAR(20) NOT NULL,            -- 'approved', 'rejected', 'returned', 'escalated'
    comments        TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_approval_entity ON approval_record(entity_type, entity_id);
CREATE INDEX idx_approval_approver ON approval_record(approver_id);
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
    trip_id         UUID REFERENCES trip(id),        -- optional link to trip
    report_name     VARCHAR(300) NOT NULL,
    purpose         TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- status: draft -> submitted -> approved -> processing -> paid -> rejected -> returned
    total_amount    DECIMAL(12,2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    amount_due_employee DECIMAL(12,2) DEFAULT 0,
    amount_due_company  DECIMAL(12,2) DEFAULT 0,
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    payment_reference VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_expense_org ON expense_report(organisation_id);
CREATE INDEX idx_expense_user ON expense_report(user_id);
CREATE INDEX idx_expense_trip ON expense_report(trip_id);
CREATE INDEX idx_expense_status ON expense_report(status);

-- =============================================================
-- EXPENSE ENTRY (line items)
-- =============================================================
CREATE TABLE expense_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES expense_report(id) ON DELETE CASCADE,
    booking_id      UUID REFERENCES booking(id),     -- link to booking if applicable
    expense_type    VARCHAR(50) NOT NULL,             -- 'airfare', 'hotel', 'meal', 'ground_transport', 'other'
    description     TEXT NOT NULL,
    vendor_name     VARCHAR(200),
    transaction_date DATE NOT NULL,
    amount          DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    exchange_rate   DECIMAL(12,6),                    -- if different from report currency
    amount_in_report_currency DECIMAL(12,2),
    payment_type    VARCHAR(30) NOT NULL,             -- 'corporate_card', 'personal_card', 'cash', 'virtual_card'
    card_transaction_id UUID REFERENCES card_transaction(id),
    receipt_id      UUID REFERENCES receipt(id),
    gl_account      VARCHAR(50),                      -- general ledger account code
    cost_centre_id  UUID REFERENCES cost_centre(id),
    is_personal     BOOLEAN NOT NULL DEFAULT false,   -- personal expense on corp card
    is_policy_compliant BOOLEAN NOT NULL DEFAULT true,
    policy_violation_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entry_report ON expense_entry(report_id);
CREATE INDEX idx_entry_type ON expense_entry(expense_type);
CREATE INDEX idx_entry_date ON expense_entry(transaction_date);

-- =============================================================
-- EXPENSE ALLOCATION (split across cost centres)
-- =============================================================
CREATE TABLE expense_allocation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id        UUID NOT NULL REFERENCES expense_entry(id) ON DELETE CASCADE,
    cost_centre_id  UUID NOT NULL REFERENCES cost_centre(id),
    gl_account      VARCHAR(50),
    percentage      DECIMAL(5,2) NOT NULL,            -- allocation percentage
    amount          DECIMAL(12,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_allocation_entry ON expense_allocation(entry_id);

-- =============================================================
-- RECEIPT
-- =============================================================
CREATE TABLE receipt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    file_name       VARCHAR(300) NOT NULL,
    file_url        TEXT NOT NULL,                    -- S3 / blob storage URL
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    capture_source  VARCHAR(30),                      -- 'camera', 'email', 'sms', 'upload'
    ocr_text        TEXT,                             -- extracted text from AI/OCR
    ocr_vendor      VARCHAR(200),                    -- AI-extracted vendor name
    ocr_amount      DECIMAL(12,2),                   -- AI-extracted amount
    ocr_date        DATE,                            -- AI-extracted date
    ocr_currency    CHAR(3),                         -- AI-extracted currency
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_receipt_user ON receipt(user_id);
```

## Payment & Corporate Card

```sql
-- =============================================================
-- PAYMENT METHOD (PCI DSS compliant — tokenized)
-- =============================================================
CREATE TABLE payment_method (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES app_user(id),    -- NULL for company-level cards
    method_type     VARCHAR(30) NOT NULL,             -- 'corporate_card', 'virtual_card', 'bank_account'
    card_token      VARCHAR(255),                     -- PCI-compliant payment processor token
    last_four       CHAR(4),
    card_network    VARCHAR(20),                      -- 'visa', 'mastercard', 'amex'
    expiry_month    SMALLINT,
    expiry_year     SMALLINT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_org ON payment_method(organisation_id);
CREATE INDEX idx_payment_user ON payment_method(user_id);

-- =============================================================
-- CARD TRANSACTION (auto-imported from card processor)
-- =============================================================
CREATE TABLE card_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_method_id UUID NOT NULL REFERENCES payment_method(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    transaction_date DATE NOT NULL,
    posted_date     DATE,
    merchant_name   VARCHAR(300),
    merchant_category_code VARCHAR(10),               -- MCC code
    amount          DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    original_amount DECIMAL(12,2),                    -- if foreign currency
    original_currency CHAR(3),
    is_reconciled   BOOLEAN NOT NULL DEFAULT false,
    reconciled_entry_id UUID REFERENCES expense_entry(id),
    external_txn_id VARCHAR(100),                     -- processor's transaction ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_card_txn_user ON card_transaction(user_id);
CREATE INDEX idx_card_txn_date ON card_transaction(transaction_date);
CREATE INDEX idx_card_txn_reconciled ON card_transaction(is_reconciled);
```

## Duty of Care & Risk Management

```sql
-- =============================================================
-- DUTY OF CARE (ISO 31030 aligned)
-- =============================================================
CREATE TABLE traveller_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    trip_id         UUID REFERENCES trip(id),
    location_source VARCHAR(30) NOT NULL,             -- 'itinerary', 'checkin', 'gps', 'manual'
    country_code    CHAR(2) NOT NULL,
    city            VARCHAR(200),
    latitude        DECIMAL(9,6),
    longitude       DECIMAL(9,6),
    recorded_at     TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_location_user ON traveller_location(user_id);
CREATE INDEX idx_location_time ON traveller_location(recorded_at);
CREATE INDEX idx_location_country ON traveller_location(country_code);

CREATE TABLE risk_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_type      VARCHAR(50) NOT NULL,             -- 'weather', 'security', 'health', 'political', 'natural_disaster'
    severity        VARCHAR(20) NOT NULL,             -- 'low', 'medium', 'high', 'critical'
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    country_code    CHAR(2),
    region          VARCHAR(200),
    latitude        DECIMAL(9,6),
    longitude       DECIMAL(9,6),
    radius_km       DECIMAL(8,2),
    source          VARCHAR(100),                     -- 'everbridge', 'international_sos', 'internal'
    effective_from  TIMESTAMPTZ NOT NULL,
    effective_to    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_country ON risk_alert(country_code);
CREATE INDEX idx_alert_severity ON risk_alert(severity);
CREATE INDEX idx_alert_effective ON risk_alert(effective_from, effective_to);

CREATE TABLE traveller_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_id        UUID NOT NULL REFERENCES risk_alert(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    trip_id         UUID REFERENCES trip(id),
    notification_status VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- status: pending -> sent -> acknowledged -> resolved
    notified_at     TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_traveller_alert_user ON traveller_alert(user_id);

CREATE TABLE emergency_contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    contact_name    VARCHAR(200) NOT NULL,
    relationship    VARCHAR(50),
    phone           VARCHAR(30) NOT NULL,
    email           VARCHAR(320),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Carbon & Sustainability

```sql
-- =============================================================
-- CARBON EMISSIONS (GHG Protocol Scope 3 Category 6)
-- =============================================================
CREATE TABLE carbon_emission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES booking(id),
    segment_id      UUID,                            -- flight_segment or hotel_booking id
    transport_mode  VARCHAR(30) NOT NULL,            -- 'air', 'rail', 'car', 'hotel'
    calculation_method VARCHAR(30) NOT NULL,         -- 'distance', 'fuel', 'spend'
    distance_km     DECIMAL(10,2),
    cabin_class     VARCHAR(20),                     -- affects emission factor for air
    emission_factor DECIMAL(10,6),                   -- kg CO2e per unit
    emission_factor_source VARCHAR(100),             -- e.g. 'DEFRA 2026', 'emissions.dev'
    co2e_kg         DECIMAL(10,4) NOT NULL,          -- total CO2 equivalent in kg
    scope           VARCHAR(10) NOT NULL DEFAULT '3.6', -- GHG Protocol scope
    reporting_year  SMALLINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carbon_booking ON carbon_emission(booking_id);
CREATE INDEX idx_carbon_year ON carbon_emission(reporting_year);
```

## Audit Log

```sql
-- =============================================================
-- AUDIT LOG
-- =============================================================
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES app_user(id),
    entity_type     VARCHAR(50) NOT NULL,            -- 'booking', 'expense_report', 'travel_request', etc.
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,            -- 'create', 'update', 'delete', 'approve', 'reject'
    changes         JSONB,                           -- {"field": {"old": "X", "new": "Y"}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

-- =============================================================
-- DATA RETENTION & CONSENT (GDPR / CCPA)
-- =============================================================
CREATE TABLE consent_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    consent_type    VARCHAR(50) NOT NULL,            -- 'location_tracking', 'data_processing', 'marketing'
    granted         BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    ip_address      INET,
    source          VARCHAR(50)                      -- 'onboarding', 'settings', 'api'
);

CREATE INDEX idx_consent_user ON consent_record(user_id);

CREATE TABLE data_retention_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    data_category   VARCHAR(50) NOT NULL,            -- 'pnr', 'expense', 'location', 'audit'
    retention_days  INTEGER NOT NULL,
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    legal_basis     TEXT,                            -- GDPR Article 6 basis
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | organisation, app_user, department, cost_centre |
| RBAC & Permissions | 4 | role, permission, role_permission, user_role |
| Reference Data | 6 | airline, airport, currency, jurisdiction, hotel_chain, car_rental_company |
| Travel Policy | 3 | travel_policy, policy_rule, policy_assignment |
| Supplier Management | 1 | supplier_agreement |
| Trip & Booking | 7 | travel_request, trip, trip_traveller, booking, flight_segment, hotel_booking, car_rental |
| Approval Workflow | 3 | approval_workflow, approval_step, approval_record |
| Expense Management | 4 | expense_report, expense_entry, expense_allocation, receipt |
| Payment & Card | 2 | payment_method, card_transaction |
| Duty of Care | 4 | traveller_location, risk_alert, traveller_alert, emergency_contact |
| Carbon & Sustainability | 1 | carbon_emission |
| Audit & Compliance | 3 | audit_log, consent_record, data_retention_policy |
| **Total** | **42** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe for multi-region deployments, and prevents enumeration attacks on sequential IDs.

2. **Tenant isolation via `organisation_id` foreign keys** — every data table includes an `organisation_id` column enabling row-level security policies in PostgreSQL. This supports shared-database multi-tenancy without schema-per-tenant complexity.

3. **Separate booking type tables (flight_segment, hotel_booking, car_rental)** rather than a single polymorphic table — each travel segment type has fundamentally different fields (cabin class vs. room type vs. vehicle class). Separate tables preserve type safety and enable type-specific indexes.

4. **Standards-aligned reference data** — airline (IATA 2-letter), airport (IATA 3-letter / ICAO 4-letter), currency (ISO 4217), jurisdiction (ISO 3166-1/2) tables are populated from official registries and enable interoperability with GDS/NDC systems.

5. **Policy as data, not code** — travel policies are expressed as rows in `policy_rule` with condition/operator/threshold columns. This allows non-technical travel managers to configure rules and supports jurisdiction-specific policy variations without code changes.

6. **State machine status columns** — `travel_request.status`, `booking.status`, `expense_report.status` follow explicit state transitions documented in comments. The `approval_record` table provides a complete audit trail of every approval decision.

7. **PCI DSS compliance by design** — `payment_method` stores only tokenized card references (from payment processor), never raw PANs. Card network, last four digits, and expiry are stored for display purposes only.

8. **GHG Protocol alignment** — `carbon_emission` table captures the three calculation methods (distance, fuel, spend) defined by GHG Protocol Scope 3 Category 6, with explicit emission factor sourcing for CSRD audit trails.

9. **GDPR/CCPA readiness** — `consent_record` tracks opt-in/opt-out for location tracking and data processing. `data_retention_policy` defines per-category, per-jurisdiction retention periods. PII fields (passport, date of birth) are identified for encryption-at-rest.

10. **Audit log with JSONB diffs** — `audit_log.changes` stores before/after field values as JSONB, providing a human-readable change history without the complexity of full event sourcing.
