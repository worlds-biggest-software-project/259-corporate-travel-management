# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Corporate Travel Management · Created: 2026-05-22

## Philosophy

This model uses a relational core for CRUD-heavy operational data (bookings, expenses, payments) combined with a property graph layer for relationship-intensive queries. The graph layer is implemented as `graph_node` and `graph_edge` tables in PostgreSQL (no external graph database required), using recursive CTEs and materialised paths for traversal. An optional Neo4j or Apache AGE extension can be added for production-scale graph queries.

Corporate travel management is fundamentally a **relationship-rich domain**: travellers belong to departments within organisational hierarchies; departments have budgets linked to cost centres; cost centres roll up through approval chains; bookings reference suppliers that have negotiated agreements scoped to specific routes and periods; policies apply to user groups through complex assignment rules; duty-of-care monitoring requires real-time correlation of traveller locations with risk zones. A graph-relational model makes these relationship queries natural and performant.

This approach is inspired by social network platforms (LinkedIn's entity graph), financial compliance systems (relationship mapping for conflict-of-interest detection), and supply chain platforms (supplier relationship networks). In the travel domain, it enables queries like "which travellers are currently in countries with active risk alerts?", "what is the full approval chain for this cost centre?", "which suppliers have agreements expiring this quarter that affect our top 10 routes?", and "show me all spend flowing through this cost centre hierarchy."

**Best for:** Organisations with complex organisational hierarchies, multi-entity approval chains, supplier relationship networks, and duty-of-care requirements that demand real-time correlation of travellers with risk zones.

**Trade-offs:**
- (+) Natural representation of hierarchical and networked relationships
- (+) Approval chains, org hierarchies, and supplier networks are first-class queryable structures
- (+) Duty-of-care queries ("who is affected by this risk alert?") are efficient graph traversals
- (+) Spend analysis across organisational hierarchies uses recursive traversal rather than self-joins
- (+) Can layer Neo4j or Apache AGE for production-scale graph analytics without schema changes
- (-) Graph concepts (nodes, edges, properties) add cognitive overhead for developers unfamiliar with graph databases
- (-) Recursive CTEs in PostgreSQL are slower than native graph engines for deep traversals
- (-) Dual-model (relational + graph) requires synchronisation between operational tables and graph tables
- (-) BI tools may not natively understand graph traversal patterns
- (-) More complex to implement than a pure relational or pure JSONB approach

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IATA NDC (Schema 24.1) | NDC Offer/Order entities are graph nodes; airline-route-offer relationships are graph edges |
| ISO 3166-1 / ISO 3166-2 | Jurisdictions modelled as hierarchical graph nodes (country -> subdivision -> city) |
| ISO 4217 | Currency codes on monetary properties of booking and expense nodes |
| IATA Airline/Airport Codes | Airlines and airports are graph nodes; route edges connect airport pairs |
| ISO 31030:2021 | Traveller-location-risk relationships are graph edges enabling rapid proximity queries |
| PCI DSS v4.0.1 | Payment data in relational tables (not in graph) for compliance isolation |
| GHG Protocol Scope 3 Cat 6 | Carbon emission data attached as properties to booking nodes and route edges |
| GDPR / CCPA | PII restricted to relational user table; graph nodes reference user by ID only |

---

## Graph Layer

```sql
-- =============================================================
-- GRAPH NODE — generic vertex table
-- =============================================================
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Node types:
    --   'organisation', 'department', 'cost_centre', 'user',
    --   'trip', 'booking', 'airline', 'airport', 'hotel_chain',
    --   'supplier_agreement', 'policy', 'risk_zone', 'location'
    label           VARCHAR(300) NOT NULL,           -- display name
    properties      JSONB NOT NULL DEFAULT '{}',     -- type-specific properties
    ref_table       VARCHAR(50),                     -- source relational table name
    ref_id          UUID,                            -- FK to relational table row
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gnode_org ON graph_node(organisation_id);
CREATE INDEX idx_gnode_type ON graph_node(node_type);
CREATE INDEX idx_gnode_ref ON graph_node(ref_table, ref_id);
CREATE INDEX idx_gnode_props ON graph_node USING GIN (properties jsonb_path_ops);

-- =============================================================
-- GRAPH EDGE — generic relationship table
-- =============================================================
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    --   'BELONGS_TO' (user->dept, dept->org)
    --   'REPORTS_TO' (user->user)
    --   'MANAGES' (user->dept)
    --   'CHARGED_TO' (booking->cost_centre)
    --   'APPROVES' (user->cost_centre, user->dept)
    --   'TRAVELS_ON' (user->trip)
    --   'CONTAINS' (trip->booking)
    --   'OPERATED_BY' (booking->airline)
    --   'ROUTE' (airport->airport)
    --   'LOCATED_IN' (airport->jurisdiction, user->location)
    --   'AGREEMENT_WITH' (organisation->supplier)
    --   'GOVERNED_BY' (user/dept->policy)
    --   'AFFECTED_BY' (user->risk_zone)
    --   'PART_OF' (dept->dept hierarchy, jurisdiction->jurisdiction)
    --   'EXPENSED_IN' (expense_entry->expense_report)
    properties      JSONB NOT NULL DEFAULT '{}',     -- relationship-specific properties
    weight          DECIMAL(10,4),                   -- for weighted traversals (cost, distance)
    valid_from      TIMESTAMPTZ DEFAULT now(),       -- temporal edges
    valid_to        TIMESTAMPTZ,                     -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_org ON graph_edge(organisation_id);
CREATE INDEX idx_gedge_source ON graph_edge(source_node_id);
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id);
CREATE INDEX idx_gedge_type ON graph_edge(edge_type);
CREATE INDEX idx_gedge_source_type ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_gedge_target_type ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_gedge_valid ON graph_edge(valid_from, valid_to);
CREATE INDEX idx_gedge_props ON graph_edge USING GIN (properties jsonb_path_ops);

-- =============================================================
-- MATERIALISED PATH — for fast hierarchical queries
-- =============================================================
CREATE TABLE graph_path (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ancestor_id     UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    descendant_id   UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    path_type       VARCHAR(50) NOT NULL,            -- 'org_hierarchy', 'cost_centre_rollup', 'jurisdiction'
    depth           INTEGER NOT NULL,
    path            UUID[] NOT NULL,                  -- ordered array of node IDs from ancestor to descendant
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(ancestor_id, descendant_id, path_type)
);

CREATE INDEX idx_gpath_ancestor ON graph_path(ancestor_id);
CREATE INDEX idx_gpath_descendant ON graph_path(descendant_id);
CREATE INDEX idx_gpath_type ON graph_path(path_type);
```

## Relational Core — Operational Tables

```sql
-- =============================================================
-- ORGANISATION
-- =============================================================
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    country_code    CHAR(2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
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
    job_title       VARCHAR(200),
    phone           VARCHAR(30),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_email ON app_user(email);

-- =============================================================
-- DEPARTMENT
-- =============================================================
CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(200) NOT NULL,
    parent_id       UUID REFERENCES department(id),
    manager_id      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_department_org ON department(organisation_id);

-- =============================================================
-- COST CENTRE
-- =============================================================
CREATE TABLE cost_centre (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    gl_account      VARCHAR(50),
    parent_id       UUID REFERENCES cost_centre(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, code)
);

-- =============================================================
-- REFERENCE DATA
-- =============================================================
CREATE TABLE airline (
    iata_code       CHAR(2) PRIMARY KEY,
    icao_code       CHAR(3),
    name            VARCHAR(200) NOT NULL,
    country_code    CHAR(2),
    is_ndc_enabled  BOOLEAN NOT NULL DEFAULT false,
    alliance        VARCHAR(50)
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

## Travel & Booking (Relational)

```sql
-- =============================================================
-- TRAVEL POLICY
-- =============================================================
CREATE TABLE travel_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(200) NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    rules           JSONB NOT NULL DEFAULT '[]',
    applies_to      JSONB NOT NULL DEFAULT '{"type": "organisation"}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_org ON travel_policy(organisation_id);

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
    cost_centre_id  UUID REFERENCES cost_centre(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    approvals       JSONB NOT NULL DEFAULT '[]',
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_request_org ON travel_request(organisation_id);
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trip_org ON trip(organisation_id);
CREATE INDEX idx_trip_dates ON trip(departure_date, return_date);
CREATE INDEX idx_trip_status ON trip(status);

-- =============================================================
-- BOOKING (unified with JSONB details, same as hybrid model)
-- =============================================================
CREATE TABLE booking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trip_id         UUID NOT NULL REFERENCES trip(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    booking_type    VARCHAR(30) NOT NULL,
    booking_source  VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'confirmed',
    gds_pnr         VARCHAR(20),
    total_amount    DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    base_fare       DECIMAL(12,2),
    taxes_fees      DECIMAL(12,2),
    policy_compliant BOOLEAN NOT NULL DEFAULT true,
    details         JSONB NOT NULL DEFAULT '{}',
    sustainability  JSONB DEFAULT '{}',
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
CREATE INDEX idx_booking_details ON booking USING GIN (details jsonb_path_ops);

-- =============================================================
-- SUPPLIER AGREEMENT
-- =============================================================
CREATE TABLE supplier_agreement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    supplier_type   VARCHAR(30) NOT NULL,
    supplier_name   VARCHAR(200) NOT NULL,
    supplier_code   VARCHAR(20),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    terms           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_org ON supplier_agreement(organisation_id);
```

## Expense & Payment (Relational)

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
    approvals       JSONB NOT NULL DEFAULT '[]',
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
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
    payment_type    VARCHAR(30) NOT NULL,
    gl_account      VARCHAR(50),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    is_policy_compliant BOOLEAN NOT NULL DEFAULT true,
    ai_extraction   JSONB DEFAULT '{}',
    allocations     JSONB DEFAULT '[]',
    tax_details     JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entry_report ON expense_entry(report_id);
CREATE INDEX idx_entry_date ON expense_entry(transaction_date);

-- =============================================================
-- PAYMENT METHOD & CARD TRANSACTION
-- =============================================================
CREATE TABLE payment_method (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES app_user(id),
    method_type     VARCHAR(30) NOT NULL,
    card_token      VARCHAR(255),
    last_four       CHAR(4),
    card_network    VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE card_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_method_id UUID NOT NULL REFERENCES payment_method(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    transaction_date DATE NOT NULL,
    merchant_name   VARCHAR(300),
    amount          DECIMAL(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    is_reconciled   BOOLEAN NOT NULL DEFAULT false,
    reconciled_entry_id UUID REFERENCES expense_entry(id),
    processor_data  JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_card_txn_user ON card_transaction(user_id);
CREATE INDEX idx_card_txn_date ON card_transaction(transaction_date);

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
    capture_source  VARCHAR(30),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- DUTY OF CARE
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
    context         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_location_user ON traveller_location(user_id);
CREATE INDEX idx_location_time ON traveller_location(recorded_at);

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
    source_data     JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_severity ON risk_alert(severity);
CREATE INDEX idx_alert_effective ON risk_alert(effective_from, effective_to);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

## Example Graph Queries

### Organisational hierarchy spend rollup

```sql
-- Total travel spend for a department and all its sub-departments
-- Uses materialised path for efficient hierarchy traversal
WITH dept_hierarchy AS (
    SELECT gp.descendant_id
    FROM graph_path gp
    JOIN graph_node gn ON gn.id = gp.ancestor_id
    WHERE gn.ref_table = 'department'
      AND gn.ref_id = 'target-dept-uuid'
      AND gp.path_type = 'org_hierarchy'
)
SELECT
    d.name AS department,
    SUM(b.total_amount) AS total_spend,
    COUNT(DISTINCT t.id) AS trip_count
FROM dept_hierarchy dh
JOIN graph_node gn ON gn.id = dh.descendant_id
JOIN department d ON d.id = gn.ref_id
JOIN app_user u ON u.organisation_id = d.organisation_id
JOIN trip t ON t.primary_traveller_id = u.id
JOIN booking b ON b.trip_id = t.id
WHERE b.status NOT IN ('cancelled', 'refunded')
GROUP BY d.name
ORDER BY total_spend DESC;
```

### Find approval chain for a cost centre

```sql
-- Walk the APPROVES edges to find who can approve expenses for a cost centre
SELECT
    approver_node.label AS approver_name,
    approver_node.properties->>'job_title' AS job_title,
    e.properties->>'approval_limit' AS approval_limit,
    e.properties->>'currency' AS currency
FROM graph_edge e
JOIN graph_node cc_node ON cc_node.id = e.target_node_id
JOIN graph_node approver_node ON approver_node.id = e.source_node_id
WHERE cc_node.ref_table = 'cost_centre'
  AND cc_node.ref_id = 'target-cost-centre-uuid'
  AND e.edge_type = 'APPROVES'
  AND (e.valid_to IS NULL OR e.valid_to > now())
ORDER BY (e.properties->>'approval_limit')::DECIMAL DESC;
```

### Travellers affected by a risk alert

```sql
-- Find all travellers currently located in a risk zone
-- Uses graph edges to correlate traveller locations with risk alerts
WITH risk_zone AS (
    SELECT
        rn.id AS risk_node_id,
        ra.latitude AS risk_lat,
        ra.longitude AS risk_lng,
        ra.radius_km
    FROM graph_node rn
    JOIN risk_alert ra ON ra.id = rn.ref_id
    WHERE rn.node_type = 'risk_zone'
      AND rn.ref_id = 'alert-uuid-here'
)
SELECT
    u.first_name || ' ' || u.last_name AS traveller,
    u.email,
    u.phone,
    tl.city,
    tl.country_code,
    tl.recorded_at AS last_location_update
FROM risk_zone rz
CROSS JOIN LATERAL (
    SELECT tl.*
    FROM traveller_location tl
    WHERE tl.recorded_at > now() - interval '24 hours'
      AND (
          6371 * acos(
              cos(radians(rz.risk_lat)) * cos(radians(tl.latitude)) *
              cos(radians(tl.longitude) - radians(rz.risk_lng)) +
              sin(radians(rz.risk_lat)) * sin(radians(tl.latitude))
          )
      ) <= rz.radius_km
    ORDER BY tl.recorded_at DESC
    LIMIT 1
) tl
JOIN app_user u ON u.id = tl.user_id;
```

### Supplier agreement network for a route

```sql
-- Find all supplier agreements that cover a specific route
-- Traverses: airport->ROUTE->airport, airline->OPERATES->route, org->AGREEMENT_WITH->airline
SELECT
    sa.supplier_name,
    sa.supplier_type,
    sa.terms->>'discount_pct' AS discount,
    sa.effective_to,
    route_edge.properties->>'frequency' AS weekly_flights
FROM graph_node origin ON origin.node_type = 'airport'
JOIN graph_edge route_edge ON route_edge.source_node_id = origin.id
    AND route_edge.edge_type = 'ROUTE'
JOIN graph_node dest ON dest.id = route_edge.target_node_id
JOIN graph_edge operates ON operates.target_node_id = route_edge.id
    AND operates.edge_type = 'OPERATES'
JOIN graph_node airline_node ON airline_node.id = operates.source_node_id
JOIN graph_edge agreement_edge ON agreement_edge.target_node_id = airline_node.id
    AND agreement_edge.edge_type = 'AGREEMENT_WITH'
JOIN graph_node org_node ON org_node.id = agreement_edge.source_node_id
JOIN supplier_agreement sa ON sa.id = (
    SELECT ref_id FROM graph_node WHERE id = agreement_edge.id -- simplified
)
WHERE origin.properties->>'iata_code' = 'JFK'
  AND dest.properties->>'iata_code' = 'LHR'
  AND (route_edge.valid_to IS NULL OR route_edge.valid_to > now());
```

### Recursive org hierarchy traversal (without materialised paths)

```sql
-- Alternative: use recursive CTE when materialised paths are not available
WITH RECURSIVE org_tree AS (
    -- Base case: start at the target department
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.ref_id,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.ref_table = 'department'
      AND gn.ref_id = 'target-dept-uuid'

    UNION ALL

    -- Recursive case: follow PART_OF edges downward
    SELECT
        child.id,
        child.label,
        child.ref_id,
        ot.depth + 1,
        ot.path || child.id
    FROM org_tree ot
    JOIN graph_edge e ON e.target_node_id = ot.node_id
        AND e.edge_type = 'PART_OF'
    JOIN graph_node child ON child.id = e.source_node_id
    WHERE child.id != ALL(ot.path)  -- prevent cycles
      AND ot.depth < 10             -- depth limit
)
SELECT * FROM org_tree ORDER BY depth, label;
```

## Graph Synchronisation Triggers

```sql
-- =============================================================
-- Trigger: Sync relational data to graph on INSERT/UPDATE
-- =============================================================

-- Example: When a new user is created, create a graph node and edges
CREATE OR REPLACE FUNCTION sync_user_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    user_node_id UUID;
    dept_node_id UUID;
    org_node_id  UUID;
BEGIN
    -- Create or update user node
    INSERT INTO graph_node (organisation_id, node_type, label, ref_table, ref_id, properties)
    VALUES (
        NEW.organisation_id,
        'user',
        NEW.first_name || ' ' || NEW.last_name,
        'app_user',
        NEW.id,
        jsonb_build_object(
            'email', NEW.email,
            'employee_id', NEW.employee_id,
            'job_title', NEW.job_title,
            'is_active', NEW.is_active
        )
    )
    ON CONFLICT (ref_table, ref_id) DO UPDATE
    SET label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        updated_at = now()
    RETURNING id INTO user_node_id;

    -- Note: In production, add a unique partial index on (ref_table, ref_id)
    -- CREATE UNIQUE INDEX idx_gnode_ref_unique ON graph_node(ref_table, ref_id)
    --   WHERE ref_table IS NOT NULL AND ref_id IS NOT NULL;

    -- Create BELONGS_TO edge to organisation
    SELECT id INTO org_node_id
    FROM graph_node
    WHERE ref_table = 'organisation' AND ref_id = NEW.organisation_id
    LIMIT 1;

    IF org_node_id IS NOT NULL THEN
        INSERT INTO graph_edge (organisation_id, source_node_id, target_node_id, edge_type)
        VALUES (NEW.organisation_id, user_node_id, org_node_id, 'BELONGS_TO')
        ON CONFLICT DO NOTHING;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_user_graph
    AFTER INSERT OR UPDATE ON app_user
    FOR EACH ROW
    EXECUTE FUNCTION sync_user_to_graph();

-- Similar triggers for: department, cost_centre, trip, booking,
-- supplier_agreement, travel_policy, risk_alert
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 3 | graph_node, graph_edge, graph_path |
| Identity & Organisation | 4 | organisation, app_user, department, cost_centre |
| Reference Data | 3 | airline, airport, currency |
| Travel Policy | 1 | travel_policy |
| Trip & Booking | 4 | travel_request, trip, booking, supplier_agreement |
| Expense Management | 3 | expense_report, expense_entry, receipt |
| Payment & Card | 2 | payment_method, card_transaction |
| Duty of Care | 3 | traveller_location, risk_alert |
| Audit | 1 | audit_log |
| **Total** | **24** | 3 graph tables + 21 relational tables |

---

## Key Design Decisions

1. **Generic graph tables (graph_node, graph_edge) rather than domain-specific relationship tables** — a single pair of graph tables handles all relationship types: org hierarchies, approval chains, supplier networks, traveller-trip associations, and duty-of-care correlations. New relationship types are added by defining new `edge_type` values rather than creating new junction tables.

2. **Materialised paths for hierarchy performance** — the `graph_path` table pre-computes all ancestor-descendant relationships for hierarchical structures (org tree, cost centre rollup, jurisdiction hierarchy). This trades storage for query speed: hierarchical aggregations become simple joins instead of recursive CTEs.

3. **Temporal edges with `valid_from`/`valid_to`** — graph edges support time-bounded validity. A supplier agreement edge is valid only during the agreement period. An employee's department membership edge has a start and end date. This enables historical graph queries ("who was this person's manager in Q1?") without maintaining separate history tables.

4. **Graph nodes reference relational tables via `ref_table`/`ref_id`** — graph nodes are lightweight pointers to authoritative relational data. The relational tables remain the source of truth for operational CRUD; the graph layer is an index over relationships. This avoids data duplication while providing graph query capabilities.

5. **Trigger-based synchronisation** — PostgreSQL triggers on relational tables automatically create and update corresponding graph nodes and edges. This ensures the graph stays consistent with relational data without requiring application-layer dual-writes. In production, an async event-driven approach (via pg_notify or change data capture) may be preferred for performance.

6. **Haversine distance calculation for duty-of-care proximity queries** — the risk alert query uses the Haversine formula in SQL to find travellers within a risk zone's radius. For production scale, PostGIS extension with spatial indexes would provide better performance, but the SQL-only approach works without additional extensions.

7. **Relational core preserves CRUD efficiency** — bookings, expenses, and payments remain in standard relational tables for efficient transactional operations. The graph layer adds relationship intelligence without replacing the operational data store. This means standard ORM tooling works for day-to-day operations while graph queries handle analytical and relationship-intensive use cases.

8. **Graph properties as JSONB** — both nodes and edges carry JSONB `properties` for type-specific attributes. This avoids the need for separate property tables per node/edge type while keeping the core graph schema simple. GIN indexes on properties enable filtered graph traversals.

9. **Edge weights enable cost-optimised routing** — the `weight` column on `graph_edge` supports weighted graph algorithms. For route optimisation, weight can represent cost or distance. For approval chains, weight can represent approval authority level. This makes the graph layer useful for optimisation problems beyond simple relationship queries.

10. **No external graph database dependency** — the entire graph layer runs in PostgreSQL, eliminating operational complexity of a separate Neo4j or similar deployment. For organisations that outgrow PostgreSQL's graph query performance, the schema maps directly to a property graph model that can be migrated to Neo4j or Apache AGE with minimal transformation.
