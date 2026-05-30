# Corporate Travel Management — Phased Development Plan

> Project: 259-corporate-travel-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` documents into a concrete, phased implementation specification for an AI-native, open-source corporate travel and expense platform.

The core thesis from the research: incumbents (SAP Concur, Navan, TravelPerk, Amex GBT) support conversational *search* but still require UI interaction for approval and expense coding. **No incumbent offers a fully conversational, end-to-end booking-to-expense workflow.** This plan therefore treats the conversational AI agent and policy-as-data engine as the heart of the product (Phases 5–7), built on a standards-aligned foundation (NDC, OTA, ISO 31030, PCI DSS, GDPR).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **Python 3.12** | The product is LLM-agent-heavy (conversational booking, receipt OCR, policy extraction, spend-leakage detection). Python has the strongest LLM/agent ecosystem (openai/anthropic SDKs, MCP server SDK, instructor for structured output) and excellent libraries for the supplier-API integration surface. |
| API framework | **FastAPI** | Async-native (essential for fan-out across GDS/NDC supplier calls), auto-generates **OpenAPI 3.1** specs — a hard requirement from `standards.md` for partner integrations — and integrates cleanly with Pydantic v2 for request/response validation. |
| Data validation | **Pydantic v2** | Single source of truth for API schemas, JSONB document shapes (the hybrid model relies on app-layer validation of JSONB), and LLM structured output. |
| ORM / DB toolkit | **SQLAlchemy 2.0 + Alembic** | Mature async ORM with first-class PostgreSQL JSONB and GIN-index support; Alembic handles migrations for the relational columns while JSONB absorbs schema variance. |
| Database | **PostgreSQL 16** | The chosen data model (Hybrid Relational + JSONB, suggestion 3) requires Postgres' mature JSONB, GIN indexes, jsonpath, and **row-level security** for `organisation_id` tenant isolation. Also provides `INET`, `TIMESTAMPTZ`, and `gen_random_uuid()`. |
| Cache / queue broker | **Redis 7** | Session cache, idempotency keys for booking writes, rate-limit counters, and Celery broker. |
| Task queue | **Celery** | Async workloads are central: supplier API fan-out, webhook processing, receipt OCR, card reconciliation, duty-of-care risk matching, carbon calculation. Celery gives retries, scheduled beat jobs, and dead-letter handling. |
| LLM access | **Vercel AI Gateway / provider-agnostic via `litellm`** | The conversational agent must not be locked to one provider (research notes Claude/GPT-4o/Gemini as interchangeable). A gateway abstraction enables failover and cost tracking. |
| Agent tool protocol | **MCP (Model Context Protocol)** | `standards.md` explicitly identifies MCP as directly relevant; Sabre already ships an MCP beta. Exposing search/policy/expense tools as MCP servers lets any LLM act as a booking agent without per-provider wrappers and future-proofs against GDS MCP adoption. |
| Supplier content (MVP) | **Duffel** (air + accommodation aggregator) | `standards.md` recommends a GDS aggregator (Duffel/Kiwi B2B) as a faster path than direct Amadeus/Sabre/Travelport certification. Duffel exposes **NDC content** via REST, covering the MVP "GDS connectivity" requirement without a TMC commercial agreement. A `SupplierAdapter` interface keeps Amadeus/Sabre/Travelport pluggable later. |
| Auth | **Authlib (OAuth 2.0 / OIDC) + python-jose (JWT)** | `standards.md` mandates RFC 6749 (OAuth 2.0), RFC 7519 (JWT), and OpenID Connect for enterprise SSO. SCIM 2.0 (RFC 7643/7644) provisioning is a dedicated endpoint set. |
| Payments | **Stripe (tokenisation + virtual cards via Issuing)** | PCI DSS v4.0.1 scope reduction — raw PANs never touch our servers; we store only Stripe tokens and `last_four`. Stripe Issuing covers virtual-card creation for booking payment. |
| Frontend | **Next.js 16 (App Router) + shadcn/ui + Tailwind** | Needed for the admin/travel-manager dashboard (spend visibility, policy config, approvals) and the chat surface. SSR for dashboards, streaming for the conversational agent. |
| Mobile | **React Native (Expo)** — *deferred to backlog phase* | MVP feature list includes a mobile app; built last, reusing the same REST API. |
| Containerisation | **Docker + docker-compose** | Self-hosted and hybrid deployment is a stated requirement (data-residency / duty-of-care). compose wires api + worker + postgres + redis for local and self-host. |
| Testing | **pytest + pytest-asyncio + respx (HTTP mocking) + testcontainers** | Unit + async integration; respx mocks supplier/LLM HTTP; testcontainers spins real Postgres for integration tests. |
| Code quality | **ruff (lint+format) + mypy (strict) + pre-commit** | Standard, fast Python tooling; mypy strict because the financial/PCI domain demands type safety. |
| Package manager | **uv** | Fast, reproducible Python dependency resolution and locking. |
| Observability | **OpenTelemetry + structlog** | Structured audit-grade logging (GDPR + PCI require auditability); traces across the async supplier fan-out. |

### Data Model Decision

The project provides four data-model proposals. This plan adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB)** as the primary schema, because the dominant engineering challenge in corporate travel is *normalising heterogeneous supplier data* (multiple GDS dialects, NDC schema 21.3/24.1 variants, OTA hotel/car formats, direct-supplier responses) while iterating quickly on an MVP. JSONB columns (`booking.details`, `booking.supplier_data`, `travel_policy.rules`, `expense_entry.ai_extraction`) absorb this variance without migrations.

From **Suggestion 1 (Entity-Centric 3NF)** we adopt the **standards-aligned reference-data tables** verbatim (`airline`, `airport`, `currency`, `jurisdiction`, `hotel_chain`, `car_rental_company`) and the **audit/consent/retention** tables — these benefit from strict relational integrity and standards codes (IATA, ICAO, ISO 3166, ISO 4217). From **Suggestion 2 (Event-Sourced)** we adopt only the `audit_log` JSONB-diff pattern, not full CQRS. From **Suggestion 4 (Graph-Relational)** we adopt the *idea* of correlation queries for duty-of-care, implemented as PostGIS-style proximity queries rather than a graph store.

Net: core identity, booking, financial, and reference tables are relational with `organisation_id` row-level security; variable supplier/policy/AI data lives in documented, Pydantic-validated JSONB.

### Project Structure

```
corporate-travel-management/
├── pyproject.toml                 # uv-managed deps, ruff/mypy config
├── uv.lock
├── .pre-commit-config.yaml
├── Dockerfile                     # api + worker (multi-stage)
├── docker-compose.yml             # api, worker, beat, postgres, redis
├── docker-compose.override.yml    # local dev (hot reload, exposed ports)
├── alembic.ini
├── README.md
├── .env.example
├── migrations/                    # Alembic revisions
│   └── versions/
├── src/
│   └── ctm/
│       ├── __init__.py
│       ├── main.py                # FastAPI app factory, router registration
│       ├── config.py              # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── base.py            # SQLAlchemy declarative base, session factory
│       │   ├── rls.py             # row-level-security session context (org scoping)
│       │   └── models/            # ORM models grouped by concern
│       │       ├── identity.py    # organisation, app_user, department, cost_centre
│       │       ├── rbac.py        # role, user_role, permission
│       │       ├── reference.py   # airline, airport, currency, jurisdiction, chains
│       │       ├── policy.py      # travel_policy, supplier_agreement
│       │       ├── trip.py        # travel_request, trip, trip_traveller, booking
│       │       ├── expense.py     # expense_report, expense_entry, receipt
│       │       ├── payment.py     # payment_method, card_transaction
│       │       ├── dutyofcare.py  # traveller_location, risk_alert, traveller_alert
│       │       ├── sustainability.py # carbon_emission
│       │       └── audit.py       # audit_log, consent_record, data_retention_policy
│       ├── schemas/               # Pydantic v2 models (API + JSONB shapes)
│       │   ├── booking.py
│       │   ├── policy.py
│       │   ├── expense.py
│       │   └── ...
│       ├── api/
│       │   ├── deps.py            # auth, current_user, org context, pagination
│       │   ├── errors.py          # exception handlers -> RFC 7807 problem+json
│       │   └── routes/
│       │       ├── auth.py
│       │       ├── scim.py        # SCIM 2.0 provisioning
│       │       ├── trips.py
│       │       ├── bookings.py
│       │       ├── search.py
│       │       ├── policy.py
│       │       ├── approvals.py
│       │       ├── expenses.py
│       │       ├── cards.py
│       │       ├── dutyofcare.py
│       │       ├── analytics.py
│       │       ├── webhooks.py
│       │       └── chat.py        # conversational agent endpoint (SSE stream)
│       ├── core/                  # business logic (no I/O framework deps)
│       │   ├── policy_engine.py   # rule evaluation
│       │   ├── approval_engine.py # workflow state machine
│       │   ├── reconciliation.py  # card<->expense matching
│       │   ├── carbon.py          # GHG Scope 3 Cat 6 calculation
│       │   ├── leakage.py         # spend-leakage detection
│       │   └── risk_matching.py   # traveller<->risk-alert correlation
│       ├── suppliers/             # supplier integration adapters
│       │   ├── base.py            # SupplierAdapter protocol
│       │   ├── duffel.py          # Duffel air + stays adapter
│       │   ├── normalise.py       # supplier response -> canonical Offer/Order
│       │   └── mock.py            # deterministic mock adapter for tests/demo
│       ├── ai/
│       │   ├── gateway.py         # litellm wrapper, provider failover
│       │   ├── agent.py           # conversational booking agent loop
│       │   ├── prompts.py         # system/user prompt templates
│       │   ├── receipt_ocr.py     # vision-model receipt extraction
│       │   └── policy_extract.py  # policy-doc -> rules extraction
│       ├── mcp/
│       │   └── server.py          # MCP server exposing tools (search/policy/expense)
│       ├── integrations/
│       │   ├── stripe_pay.py      # tokenisation, virtual cards
│       │   ├── emissions_dev.py   # carbon API client
│       │   ├── risk_feed.py       # Everbridge/Intl SOS-style risk client
│       │   └── m365.py            # Teams/Outlook (v1.1)
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks.py           # async jobs + beat schedules
│       └── observability/
│           └── logging.py
├── frontend/                      # Next.js 16 dashboard + chat (built in Phase 9)
└── tests/
    ├── conftest.py                # fixtures: db, client, mock supplier, auth
    ├── fixtures/                  # NDC/OTA sample responses, receipts, policies
    ├── unit/
    ├── integration/
    └── e2e/
```

The structure groups by concern (`core/`, `suppliers/`, `ai/`, `api/`), never by phase, so every phase is additive.

---

## Phase 1: Foundation, Multi-Tenancy & Reference Data

### Purpose
Establish the runnable skeleton: configuration, the database with multi-tenant row-level security, Alembic migrations, the standards-aligned reference data, structured logging, the audit log, and a health endpoint. After this phase the application boots, connects to Postgres/Redis, enforces tenant isolation at the DB layer, and serves an OpenAPI 3.1 spec. Nothing domain-specific works yet, but every later phase plugs into this scaffold.

### Tasks

#### 1.1 — Project scaffolding & configuration

**What**: A `uv`-managed Python project with FastAPI app factory, Pydantic settings, Docker, and CI tooling.

**Design**:
```python
# src/ctm/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    environment: str = "development"          # development|staging|production
    database_url: str                          # postgresql+asyncpg://...
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    access_token_ttl_seconds: int = 3600
    duffel_api_key: str | None = None
    stripe_secret_key: str | None = None
    llm_gateway_url: str | None = None
    default_llm_model: str = "anthropic/claude-sonnet"
    log_level: str = "INFO"

def get_settings() -> Settings: ...           # lru_cached
```

```python
# src/ctm/main.py
def create_app() -> FastAPI:
    app = FastAPI(title="Corporate Travel Management", version="0.1.0",
                  openapi_version="3.1.0")
    register_error_handlers(app)              # RFC 7807 problem+json
    register_routers(app)
    return app
```

- `GET /health` returns `{"status": "ok", "db": "ok", "redis": "ok"}` after pinging both.
- Error handler maps `AppError(code, status, detail)` to `application/problem+json` per RFC 7807.

**Testing**:
- `Unit: get_settings() with valid env → Settings populated with defaults`
- `Unit: get_settings() missing DATABASE_URL → ValidationError naming database_url`
- `Integration: GET /health with db+redis up → 200, all "ok"`
- `Integration: GET /health with db down → 503, db:"error"`
- `Unit: GET /openapi.json → openapi field == "3.1.0"`

#### 1.2 — Database base, sessions & row-level-security tenant scoping

**What**: SQLAlchemy 2.0 async base, session factory, and a per-request RLS context that scopes all queries to the caller's `organisation_id`.

**Design**:
- Every tenant table includes `organisation_id UUID NOT NULL`.
- Enable Postgres RLS on tenant tables; policy `USING (organisation_id = current_setting('app.current_org')::uuid)`.
- A dependency sets `SET LOCAL app.current_org = :org_id` at the start of each request transaction.

```python
# src/ctm/db/rls.py
@asynccontextmanager
async def org_scoped_session(org_id: UUID) -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as session:
        await session.execute(text("SET LOCAL app.current_org = :o"), {"o": str(org_id)})
        yield session
```

**Testing**:
- `Integration (real PG via testcontainers): insert rows for org A and org B; query under org A context → only A rows returned`
- `Integration: attempt cross-tenant UPDATE under org A context on org B row → 0 rows affected`
- `Unit: org_scoped_session sets app.current_org before any query`

#### 1.3 — Reference-data tables & seed loaders

**What**: Relational reference tables (from Suggestion 1) with seed loaders populating from official registries.

**Design**: Tables `currency` (ISO 4217), `airline` (IATA 2 / ICAO 3, `is_ndc_enabled`), `airport` (IATA 3 / ICAO 4, lat/long, tz), `jurisdiction` (ISO 3166-1/2 hierarchy), `hotel_chain` (OTA chain code), `car_rental_company` (OTA vendor code). A `scripts/seed_reference.py` loads packaged CSV snapshots (OpenFlights-style airport/airline data, ISO 4217 currency list, ISO 3166 country list).

**Testing**:
- `Unit: currency seed loads ISO 4217 → 'USD' has minor_units=2, 'JPY' has minor_units=0`
- `Unit: airline seed → 'BA' resolves to British Airways with icao 'BAW'`
- `Integration: seed idempotency — running loader twice → no duplicate rows (UPSERT on iata_code)`

#### 1.4 — Audit log, consent & retention

**What**: `audit_log` (JSONB before/after diffs), `consent_record`, `data_retention_policy` tables plus an `audit()` helper.

**Design**:
```python
async def audit(session, *, org_id, user_id, entity_type, entity_id,
                action: str, changes: dict | None, request: Request | None): ...
# changes shape: {"field": {"old": <v>, "new": <v>}}
```
- `audit_log.changes JSONB`, `ip_address INET`, `user_agent TEXT`.
- A SQLAlchemy event hook captures `before_flush` dirty fields for audited models.

**Testing**:
- `Unit: audit() with field change → row with changes={"status":{"old":"draft","new":"submitted"}}`
- `Integration: updating a booking's status → audit_log row auto-created with correct diff`
- `Unit: consent_record revoke → revoked_at set, granted stays historical`

#### 1.5 — Alembic baseline migration & docker-compose

**What**: First migration creating Phase-1 tables; compose stack for api+worker+postgres+redis.

**Design**: `alembic revision --autogenerate` baseline; `docker-compose.yml` services with healthchecks; `Dockerfile` multi-stage (builder installs via uv, runtime slim).

**Testing**:
- `Integration: alembic upgrade head on empty PG → all Phase-1 tables exist`
- `Integration: alembic downgrade base → upgrade head round-trips cleanly`
- `E2E: docker-compose up → GET /health returns 200 within healthcheck window`

---

## Phase 2: Identity, Auth, RBAC & SCIM

### Purpose
Add authenticated, role-scoped access. Implements OAuth 2.0 / OIDC login, JWT issuance, RBAC, and SCIM 2.0 provisioning so enterprise HR systems can sync employees. After this phase, real users in real organisations can authenticate and every endpoint can enforce permissions — the prerequisite for any booking/expense action.

### Tasks

#### 2.1 — Identity tables & organisation onboarding

**What**: `organisation`, `app_user`, `department`, `cost_centre` (relational core + `settings`/`profile` JSONB per Suggestion 3).

**Design**: `app_user.profile JSONB` holds variable fields (dietary, seat preference, loyalty numbers); core PII (`email`, names, `passport_number` encrypted) stays relational. `scim_external_id` for provisioning linkage. Endpoint `POST /organisations` bootstraps an org + first admin user.

**Testing**:
- `Unit: create org → default currency 'USD', timezone 'UTC'`
- `Integration: POST /organisations → org + admin user + default role created in one transaction`
- `Unit: duplicate email within org → 409 conflict`

#### 2.2 — OAuth 2.0 / OIDC auth & JWT

**What**: Password + OIDC (Authorization Code) login issuing JWTs (RFC 7519); SSO via enterprise IdP (Entra ID/Okta).

**Design**:
- `POST /auth/login` → `{access_token, refresh_token, token_type:"bearer", expires_in}`.
- JWT claims: `sub` (user id), `org` (org id), `roles`, `exp`, `iat`, `jti`.
- `GET /auth/oidc/{provider}/start` + `/callback` implement Authorization Code flow via Authlib.
- `deps.get_current_user` decodes JWT, loads user, sets RLS org context.

**Testing**:
- `Unit: valid creds → JWT with correct sub/org/roles claims`
- `Unit: expired token → 401 with problem+json`
- `Integration (mocked IdP): OIDC callback with valid code → user upserted, JWT issued`
- `Integration: request to protected route without token → 401`

#### 2.3 — RBAC enforcement

**What**: `role` (with JSONB `permissions` array per Suggestion 3) + `user_role`; permission dependency.

**Design**: Built-in roles `admin`, `travel_manager`, `approver`, `traveller`, `finance`. Permissions are `resource:action` strings (`booking:create`, `expense_report:approve`, `policy:write`). `require("booking:create")` FastAPI dependency raises 403 if absent.

**Testing**:
- `Unit: traveller role lacks policy:write → require() raises 403`
- `Unit: admin has wildcard → all require() pass`
- `Integration: travel_manager POSTs a policy → 200; traveller POSTs same → 403`

#### 2.4 — SCIM 2.0 provisioning

**What**: SCIM 2.0 (RFC 7643/7644) `/scim/v2/Users` and `/Groups` for HR-system sync.

**Design**: Bearer-token-auth SCIM endpoints supporting `POST/GET/PATCH/DELETE` Users with `externalId` ↔ `scim_external_id`. PATCH supports the SCIM `Operations` patch format; `active=false` deactivates (no hard delete) to preserve audit trail.

**Testing**:
- `Integration: SCIM POST /Users → app_user created, scim_external_id stored, SCIM-shaped response`
- `Integration: SCIM PATCH active=false → user.is_active=false, not deleted`
- `Unit: SCIM filter userName eq "x@y.com" → correct user returned`

---

## Phase 3: Supplier Abstraction & Travel Search

### Purpose
Build the supplier integration layer and a normalised search API. This is the content engine: a `SupplierAdapter` protocol with a Duffel implementation (NDC air + stays) and a deterministic mock, plus a normalisation layer producing canonical `Offer` objects. After this phase, the API can search flights/hotels/cars across suppliers and return a uniform shape — the substrate the conversational agent and booking flow consume.

### Tasks

#### 3.1 — SupplierAdapter protocol & canonical Offer model

**What**: A provider-agnostic interface and canonical Pydantic models for search results.

**Design**:
```python
# src/ctm/suppliers/base.py
class SupplierAdapter(Protocol):
    name: str
    async def search_flights(self, q: FlightQuery) -> list[FlightOffer]: ...
    async def search_stays(self, q: StayQuery) -> list[StayOffer]: ...
    async def search_cars(self, q: CarQuery) -> list[CarOffer]: ...
    async def create_order(self, offer_id: str, passengers: list[Passenger],
                           payment: PaymentRef) -> Order: ...
    async def cancel_order(self, order_id: str) -> CancellationResult: ...

class FlightOffer(BaseModel):
    offer_id: str
    supplier: str
    source: Literal["gds", "ndc", "direct"]
    total_amount: Decimal
    currency: str                  # ISO 4217
    segments: list[FlightSegmentOffer]
    cabin_class: str
    fare_basis: str | None
    is_ndc: bool
    raw: dict                       # original supplier payload -> booking.supplier_data
```
Canonical models mirror NDC offer/order structure (Schema 24.1) and OTA hotel/car schemas so normalisation is lossless. `raw` preserves the original for audit and re-pricing.

**Testing**:
- `Unit: FlightOffer rejects non-ISO currency → ValidationError`
- `Unit: canonical model round-trips Decimal amounts without float drift`

#### 3.2 — Duffel adapter

**What**: Concrete `DuffelAdapter` implementing search + order against Duffel's REST API.

**Design**: Async httpx client, OAuth bearer auth, pagination handling, retry with exponential backoff on 429/5xx. Maps Duffel offer JSON → `FlightOffer`/`StayOffer`, tagging `source="ndc"` when Duffel reports NDC content. Errors mapped to `SupplierError(retryable: bool)`.

**Testing**:
- `Integration (respx-mocked Duffel): flight search response fixture → 3 normalised FlightOffers`
- `Integration (mocked): Duffel 429 → retried then succeeds`
- `Integration (mocked): malformed offer → SupplierError, other offers still returned`

#### 3.3 — Mock adapter & supplier registry

**What**: A deterministic `MockAdapter` (no network) and a registry selecting active adapters per org config.

**Design**: Registry reads `organisation.settings.suppliers`; mock returns fixed offers seeded from `tests/fixtures` for demos/tests/CI. Multiple adapters can be queried concurrently via `asyncio.gather`, results merged + de-duplicated.

**Testing**:
- `Unit: registry with [mock] → search returns mock offers`
- `Unit: two adapters → concurrent fan-out, merged results sorted by price`
- `Unit: one adapter raises → results from healthy adapter still returned, error logged`

#### 3.4 — Search API endpoints

**What**: `POST /search/flights`, `/search/stays`, `/search/cars`.

**Design**:
```
POST /search/flights
{ "origin":"LHR", "destination":"JFK", "depart":"2026-07-01",
  "return":"2026-07-08", "cabin":"economy", "pax":1 }
→ 200 { "offers":[FlightOffer...], "search_id":"uuid" }
```
`search_id` caches the result set in Redis (TTL 15 min) so a later booking references a priced offer without re-search. Each offer is annotated with a preliminary policy badge (filled in Phase 4).

**Testing**:
- `Integration: POST /search/flights (mock supplier) → offers + search_id; search_id retrievable from Redis`
- `Integration: invalid IATA origin "XXX" not in airport table → 422`
- `Integration: search without auth → 401`

---

## Phase 4: Policy-as-Data Engine & Trips/Bookings

### Purpose
Implement the policy engine (the differentiator vs. legacy tools) and the trip/booking lifecycle. Policies are JSONB rule arrays evaluated against offers, producing compliant/violation verdicts before approval. After this phase, a user can create a trip, book a policy-checked offer, and persist a canonical booking with supplier data retained in JSONB.

### Tasks

#### 4.1 — Policy model & rule schema

**What**: `travel_policy` table with JSONB `rules` and `applies_to` (Suggestion 3) + `supplier_agreement`.

**Design**:
```python
class PolicyRule(BaseModel):
    id: str
    category: Literal["flight","hotel","car","meal","general"]
    rule_type: Literal["max_amount","cabin_class","advance_booking",
                        "preferred_supplier","route_restriction"]
    field: str                     # e.g. "total_amount", "cabin_class"
    operator: Literal["lte","gte","eq","in","not_in"]
    value: str | float | list[str]
    currency: str | None
    severity: Literal["block","warning","info"]
    jurisdiction: str | None       # ISO 3166-2 for region-specific rules
```
`applies_to`: `{"type":"organisation"|"department"|"role"|"user","ids":[...]}`. Versioned via `effective_from/effective_to` (temporal resolution at evaluation time).

**Testing**:
- `Unit: parse policy with mixed rules → typed PolicyRule list`
- `Unit: max_amount rule with non-numeric value → ValidationError`
- `Unit: resolve active policy for date X → correct version chosen`

#### 4.2 — Policy evaluation engine

**What**: Pure function evaluating an offer against the applicable resolved policy.

**Design**:
```python
# src/ctm/core/policy_engine.py
def evaluate(offer: FlightOffer | StayOffer | CarOffer,
             rules: list[PolicyRule],
             context: PolicyContext) -> PolicyResult: ...

class PolicyResult(BaseModel):
    compliant: bool                # False if any 'block' violation
    violations: list[Violation]    # rule_id, severity, message
    preferred_supplier_match: bool
    savings_vs_preferred: Decimal | None
```
`block` → non-compliant (requires elevated approval/justification); `warning`/`info` → compliant but surfaced. Preferred-supplier matching uses `supplier_agreement` to flag negotiated-rate availability and compute leakage.

**Testing**:
- `Unit: offer $1200 vs max_amount lte $1000 (block) → compliant=False, 1 violation`
- `Unit: business class vs cabin_class in [economy,premium_economy] → block violation`
- `Unit: warning-severity breach → compliant=True with surfaced violation`
- `Unit: offer matches preferred supplier → preferred_supplier_match=True`

#### 4.3 — Trips, bookings & supplier order placement

**What**: `travel_request`, `trip`, `trip_traveller`, `booking` tables (relational core + `details`/`supplier_data`/`policy_check`/`sustainability` JSONB) and the booking write path.

**Design**:
- `booking.details` holds type-specific fields (flight segments / hotel / car) as JSONB; `supplier_data` retains the raw supplier order; `policy_check` stores the `PolicyResult` at booking time.
- `POST /trips/{id}/bookings` flow: load cached offer by `offer_id` → re-evaluate policy → if `block` violation and no override → 409 with violations → else `adapter.create_order()` → persist booking with PNR/confirmation → emit `booking.created` event + audit.
- Idempotency: `Idempotency-Key` header stored in Redis prevents double-booking on retry.
- Status state machine: `pending → confirmed → ticketed → checked_in → completed → cancelled → refunded`.

**Testing**:
- `Integration (mock supplier): book compliant offer → booking confirmed, supplier_data persisted, audit row`
- `Integration: book offer with block violation, no override → 409 listing violations, no order created`
- `Integration: book with override + justification → booking created, policy_check records override`
- `Integration: duplicate Idempotency-Key → same booking returned, single supplier order`
- `Unit: status transition confirmed→refunded invalid path → error` (illegal transitions rejected)

#### 4.4 — Cancellation & rebooking

**What**: `DELETE /bookings/{id}` and `POST /bookings/{id}/rebook`.

**Design**: Cancellation calls `adapter.cancel_order`, records `cancelled_at`/`cancellation_fee`, transitions status, audits. Rebook creates a linked replacement booking referencing the original.

**Testing**:
- `Integration (mock): cancel confirmed booking → status cancelled, fee recorded`
- `Integration: cancel already-cancelled booking → 409`

---

## Phase 5: Conversational AI Booking Agent (core differentiator)

### Purpose
Deliver the headline capability: a natural-language agent that searches, policy-checks, books, and pre-codes expenses in one conversation. Built on MCP tools so the same tool layer powers any LLM and future GDS MCP endpoints. After this phase, a user can type *"Book me a flight to NYC next Tuesday returning Friday, economy, under $800"* and get an end-to-end booking — the feature no incumbent offers.

### Tasks

#### 5.1 — LLM gateway & MCP tool server

**What**: A provider-agnostic LLM client and an MCP server exposing platform capabilities as tools.

**Design**:
- `ai/gateway.py` wraps `litellm` for model failover + cost logging; configurable via `default_llm_model`.
- `mcp/server.py` exposes tools: `search_flights`, `search_stays`, `search_cars`, `check_policy`, `create_booking`, `get_trips`, `precode_expense`. Each tool wraps Phase 3/4 services and is org-scoped via the calling user's context. Tool I/O schemas are JSON Schema (auto-derived from Pydantic).

**Testing**:
- `Unit: gateway falls back to secondary model when primary raises rate-limit`
- `Integration: MCP search_flights tool → same offers as REST search`
- `Unit: MCP tool rejects call without org context → error`

#### 5.2 — Conversational agent loop

**What**: An agent that runs a tool-use loop, streaming responses, gathering missing trip parameters, and confirming before any booking write.

**Design**:
```python
# src/ctm/ai/agent.py
async def run_turn(conversation_id: str, user_msg: str,
                   user: User) -> AsyncIterator[AgentEvent]: ...
# AgentEvent: token | tool_call | tool_result | confirmation_request | booking_done
```
System prompt template (`ai/prompts.py`) instructs: extract origin/destination/dates/cabin/budget; call `search_flights`; rank by policy compliance then price; **never call `create_booking` without an explicit user confirmation** (human-in-the-loop guard); after booking, call `precode_expense`. Conversation state persisted in Redis (turn history + pending confirmation).

**Testing**:
- `Integration (mocked LLM returning scripted tool calls): "flight to JFK Tue" → agent asks for return date`
- `Integration: full happy path (search→confirm→book) → booking created, expense pre-coded`
- `Integration: agent attempts create_booking without confirmation → blocked by guard, asks to confirm`
- `Unit: param extraction maps "next Tuesday" to correct date given today`

#### 5.3 — Chat API (SSE streaming)

**What**: `POST /chat` streaming agent events over Server-Sent Events.

**Design**: Request `{conversation_id?, message}`; response is an SSE stream of `AgentEvent` JSON frames. A `POST /chat/{conversation_id}/confirm` endpoint resolves pending booking confirmations.

**Testing**:
- `Integration: POST /chat → SSE stream emits token + tool_call + tool_result frames`
- `Integration: confirm pending booking → booking_done frame, persisted booking`
- `Integration: chat without auth → 401`

---

## Phase 6: Expense Automation & AI Expense Agent

### Purpose
Automate the second half of the booking-to-expense loop: receipt capture with vision-model extraction, AI GL coding, expense reports, and card auto-reconciliation. After this phase, a card charge + a photographed receipt produce a coded expense entry with zero manual data entry (matching Navan's Expense Agent, the benchmark).

### Tasks

#### 6.1 — Expense reports & entries

**What**: `expense_report`, `expense_entry` (with JSONB `ai_extraction`, `allocations`, `tax_details`), `receipt` tables + CRUD.

**Design**: `expense_entry.allocations JSONB` holds cost-centre splits (Suggestion 3 collapses the allocation table into JSONB). Report status machine: `draft → submitted → approved → processing → paid` (+ `rejected`, `returned`). Entries link optionally to a `booking_id` and `card_transaction_id`.

**Testing**:
- `Unit: report total = sum of entry amounts in report currency`
- `Integration: submit report → status submitted, approval workflow triggered`
- `Unit: allocation percentages must sum to 100 → else 422`

#### 6.2 — Receipt capture & AI OCR extraction

**What**: Receipt upload (multi-channel: upload/email/SMS) + vision-model extraction.

**Design**:
- `POST /receipts` accepts image/PDF → stored in blob storage → `receipt.file_url`.
- `ai/receipt_ocr.py` calls a vision LLM with a structured-output prompt returning `{vendor, amount, currency, date, line_items[], tax}` (via `instructor`), written to `receipt.ocr_*` and `expense_entry.ai_extraction`.
- Email/SMS ingestion via inbound webhook (Phase 8 webhook infra) creating receipts with `capture_source`.

**Testing**:
- `Integration (mocked vision LLM): upload sample receipt fixture → vendor/amount/date extracted into receipt row`
- `Unit: low-confidence extraction → flagged needs_review=true`
- `Integration: unsupported mime type → 415`

#### 6.3 — AI GL coding

**What**: Auto-assign GL account + expense type from receipt/merchant data.

**Design**: `precode_expense` uses merchant name, MCC, and amount + the org's chart-of-accounts (org settings) to predict `gl_account`, `expense_type`, `cost_centre_id`. Below a confidence threshold → suggestion only (employee confirms); above → auto-applied. Used by both the manual flow and the conversational agent (Phase 5 post-booking).

**Testing**:
- `Unit: airline MCC 3000-series → expense_type "airfare"`
- `Unit: ambiguous merchant → suggestion with confidence < threshold, not auto-applied`
- `Integration: book flight via agent → expense entry pre-coded with airfare GL`

#### 6.4 — Card integration & auto-reconciliation

**What**: `payment_method`, `card_transaction` tables; Stripe-token import; matching engine.

**Design**:
- `payment_method` stores Stripe token + `last_four` only (PCI DSS v4.0.1 — no PAN).
- Card feed import (webhook/poll) creates `card_transaction` rows.
- `core/reconciliation.py` matches transactions to expense entries by amount (±tolerance), date window, and merchant similarity; sets `is_reconciled`, links `reconciled_entry_id`. Unmatched charges surface as "needs expense".

**Testing**:
- `Unit: exact amount+date+merchant → matched`
- `Unit: amount within tolerance, date within 3 days → matched with lower confidence`
- `Unit: no candidate → unreconciled, flagged`
- `Integration: import card feed fixture → entries reconciled, dashboard count correct`

---

## Phase 7: Approvals, Spend Analytics & Leakage Detection

### Purpose
Complete the management layer: multi-level approval workflows, real-time spend dashboards, and AI spend-leakage detection (an identified underserved opportunity). After this phase, travel managers get policy-compliant approvals, live spend visibility, and proactive savings recommendations.

### Tasks

#### 7.1 — Approval workflow engine

**What**: Approval workflow config + a state-machine engine driving travel requests, bookings, and expense reports through multi-level sign-off.

**Design**:
- Workflow steps with `approver_type` (manager/role/specific_user/cost_centre_owner), `auto_approve_below` threshold, and `escalation_hours`.
- `core/approval_engine.py` advances entities step-by-step; auto-approves below threshold; escalates on timeout (Celery beat). Decisions recorded in an `approval_record`-style JSONB array on the entity (`approvals`), plus audit log.

**Testing**:
- `Unit: amount below auto_approve_below → auto-approved, no human step`
- `Unit: two-step workflow, first approver approves → advances to step 2`
- `Unit: rejection at any step → entity status rejected, halts`
- `Integration: escalation timeout job → step escalated to next approver`

#### 7.2 — Spend analytics dashboard API

**What**: Aggregation endpoints for spend visibility.

**Design**: `GET /analytics/spend?group_by=department|supplier|cost_centre|month` returns aggregated totals (multi-currency normalised to org currency via stored exchange rates). Backed by SQL aggregation over `booking` + `expense_entry`; cached in Redis (5-min TTL).

**Testing**:
- `Integration: spend grouped by department → correct sums across seeded bookings`
- `Unit: multi-currency entries normalised to org currency`
- `Integration: analytics requires travel_manager/finance role → traveller gets 403`

#### 7.3 — AI spend-leakage detection

**What**: Detect policy violations, preferred-supplier non-compliance, and unused negotiated discounts.

**Design**: `core/leakage.py` scans bookings/expenses for: out-of-policy bookings made anyway, bookings off preferred suppliers where an agreement existed (computing `savings_vs_preferred` from Phase 4), and under-utilised `supplier_agreement.min_volume`. Produces ranked `LeakageFinding` records surfaced via `GET /analytics/leakage`; a Celery job refreshes daily.

**Testing**:
- `Unit: booking off preferred airline with active agreement → leakage finding with computed savings`
- `Unit: agreement min_volume unmet → under-utilisation finding`
- `Integration: leakage endpoint → findings ranked by savings descending`

---

## Phase 8: Duty of Care, Carbon, Webhooks & Disruption Alerts

### Purpose
Add ISO 31030 duty-of-care, GHG-Protocol carbon tracking, the webhook infrastructure, and proactive disruption alerts. After this phase, the platform tracks traveller locations against real-time risk feeds, calculates Scope 3 Cat 6 emissions, and triggers proactive rebooking — meeting the post-pandemic table-stakes and CSRD requirements.

### Tasks

#### 8.1 — Webhook infrastructure (inbound + outbound)

**What**: Signed inbound webhook receiver and outbound webhook dispatcher.

**Design**:
- Inbound `POST /webhooks/{source}` verifies HMAC signature, enqueues a Celery task, returns 200 fast. Sources: card feed, supplier order changes, risk feed, inbound email/SMS receipts.
- Outbound: org-registered webhook URLs receive signed `booking.created`, `expense.approved`, `risk.alert` events with retry + dead-letter.

**Testing**:
- `Integration: inbound webhook valid signature → task enqueued, 200`
- `Integration: invalid signature → 401, no task enqueued`
- `Integration: outbound delivery fails 3x → moved to dead-letter`

#### 8.2 — Duty of care: location tracking & risk matching

**What**: `traveller_location`, `risk_alert`, `traveller_alert`, `emergency_contact` tables + correlation engine.

**Design**:
- Trip itineraries auto-populate `traveller_location` (`source="itinerary"`); check-ins/GPS optional (consent-gated via `consent_record`).
- Risk feed client (Everbridge/Intl-SOS-style) ingests `risk_alert`s with geo + radius.
- `core/risk_matching.py` correlates active traveller locations against alerts (haversine within `radius_km`), creating `traveller_alert`s and dispatching notifications (proactive duty of care, ISO 31030).

**Testing**:
- `Unit: traveller within alert radius → traveller_alert created`
- `Unit: traveller outside radius → no alert`
- `Unit: location tracking without consent → not recorded`
- `Integration: risk feed ingest → matched travellers notified`

#### 8.3 — Proactive disruption alerts & rebooking

**What**: Detect flight disruptions and offer/trigger rebooking.

**Design**: Supplier order-change webhooks (delay/cancellation) → match to booking → notify traveller + (via agent) propose alternative offers. High-severity + opted-in → auto-rebook within policy.

**Testing**:
- `Integration: cancellation webhook for a booked flight → traveller notified, alternatives surfaced`
- `Unit: auto-rebook only when alternative is policy-compliant`

#### 8.4 — Carbon emissions (GHG Protocol Scope 3 Cat 6)

**What**: `carbon_emission` table + calculation on every booking.

**Design**: `core/carbon.py` computes CO2e per segment using distance (great-circle from `airport` lat/long) × cabin-class emission factor, or delegates to the emissions.dev client; stores method, factor, source (e.g. "DEFRA 2026"), `co2e_kg`, `scope="3.6"`, `reporting_year`. `GET /analytics/carbon` aggregates for CSRD reporting.

**Testing**:
- `Unit: LHR→JFK business class → expected CO2e within tolerance using distance method`
- `Integration: booking created → carbon_emission row auto-generated`
- `Integration: carbon analytics → totals by reporting_year`

---

## Phase 9: Web Dashboard & Chat UI

### Purpose
Deliver the human-facing surfaces: a Next.js 16 dashboard for travel managers/finance (policy config, approvals, spend, leakage, duty-of-care map, carbon) and a streaming chat UI for the conversational agent. After this phase the product is usable end-to-end by non-developers.

### Tasks

#### 9.1 — App shell, auth & API client

**What**: Next.js 16 App Router shell with OIDC login, shadcn/ui, and a typed API client generated from the OpenAPI 3.1 spec.

**Design**: Auth via the Phase 2 OIDC flow; tokens in httpOnly cookies; middleware guards protected routes. API client generated from `/openapi.json` for end-to-end types.

**Testing**:
- `E2E (Playwright): login → redirected to dashboard; unauthenticated → login`
- `E2E: expired session → re-auth prompt`

#### 9.2 — Manager dashboards

**What**: Spend visibility, policy editor, approvals queue, leakage, duty-of-care map, carbon report.

**Design**: Server components fetch analytics endpoints; policy editor is a form over the JSONB rule schema (Phase 4.1); approvals queue calls Phase 7.1; duty-of-care renders `traveller_location` vs `risk_alert` on a map.

**Testing**:
- `E2E: travel_manager edits a max_amount rule → persisted, reflected in next booking check`
- `E2E: approver approves a request from the queue → status updates live`

#### 9.3 — Conversational chat UI

**What**: Streaming chat consuming the Phase 5 SSE endpoint with inline confirmation cards.

**Design**: Renders `AgentEvent` frames; offer cards show policy badges (compliant/violation); a confirm button posts to `/chat/{id}/confirm` to finalise a booking.

**Testing**:
- `E2E: type a trip request → offers stream in → confirm → booking appears in trips`
- `E2E: out-of-policy offer shows violation badge and requires justification`

---

## Phase 10: Hardening, Compliance & Deployment

### Purpose
Make the platform production-ready: PCI DSS / GDPR controls, rate limiting, full OpenAPI publication, load testing, and self-hosted + SaaS deployment artefacts. After this phase the platform can be deployed for real organisations.

### Tasks

#### 10.1 — Security & compliance hardening

**What**: PCI DSS v4.0.1 scope controls, GDPR data-subject endpoints, encryption-at-rest for PII, rate limiting.

**Design**:
- PCI: confirm no PAN ever stored (only Stripe tokens); network segmentation documented in `docs/pci-saq.md`; TLS enforced.
- GDPR: `POST /privacy/export` (data portability), `POST /privacy/erase` (right to erasure honouring `data_retention_policy` legal holds), consent enforcement, EU-PNR retention limits applied via retention job.
- Encrypt `passport_number`, `date_of_birth` at rest (pgcrypto or app-layer envelope encryption).
- Redis-backed rate limiting per org/user.

**Testing**:
- `Integration: privacy export → all user data returned as JSON bundle`
- `Integration: erase request → PII anonymised, audit/financial records retained per retention policy`
- `Unit: no code path persists raw PAN (static + runtime assertion)`
- `Integration: exceed rate limit → 429`

#### 10.2 — OpenAPI publication & SDK generation

**What**: Polished OpenAPI 3.1 spec + generated client SDKs.

**Design**: Annotate all routes with descriptions/examples/tags; publish `/openapi.json` and a docs site; generate TypeScript + Python SDKs in CI.

**Testing**:
- `Integration: spec validates against OpenAPI 3.1 schema`
- `CI: generated SDK compiles and round-trips a sample call against the mock supplier`

#### 10.3 — Deployment artefacts (self-hosted + SaaS)

**What**: Production Docker images, compose + Helm/Kubernetes manifests, migration runner, seed job.

**Design**: Hardened multi-stage images; `docker-compose.prod.yml` for self-host; Kubernetes manifests (api Deployment, worker Deployment, beat CronJob, managed Postgres/Redis) for SaaS; init job runs `alembic upgrade head` + reference seed. Data-residency via per-region deployments (stated hybrid requirement).

**Testing**:
- `E2E: fresh deploy via compose.prod → health green, migrations applied, reference data seeded`
- `Smoke: post-deploy suite (auth → search → book mock → expense → analytics) passes`

#### 10.4 — Load & resilience testing

**What**: Performance baselines and failure-mode tests.

**Design**: k6/Locust scenarios for search fan-out and concurrent booking; chaos tests for supplier/LLM timeouts (circuit breaker), DB failover.

**Testing**:
- `Load: 100 concurrent searches → p95 latency within target, no errors`
- `Resilience: supplier timeout → circuit breaker opens, graceful degradation`
- `Resilience: LLM provider down → gateway fails over, chat continues`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Multi-Tenancy & Reference Data   ─── required by everything
    │
Phase 2: Identity, Auth, RBAC & SCIM                  ─── requires 1
    │
Phase 3: Supplier Abstraction & Search               ─── requires 1,2
    │
Phase 4: Policy Engine & Trips/Bookings              ─── requires 3
    ├── Phase 5: Conversational AI Booking Agent     ─── requires 4  (core differentiator)
    ├── Phase 6: Expense Automation & AI Expense Agent ─ requires 4 (can parallel with 5)
    └── Phase 7: Approvals, Analytics & Leakage      ─── requires 4,6 (can parallel with 5)
         │
Phase 8: Duty of Care, Carbon, Webhooks, Alerts      ─── requires 4 (webhooks used by 6,8)
    │
Phase 9: Web Dashboard & Chat UI                     ─── requires 5,6,7,8
    │
Phase 10: Hardening, Compliance & Deployment         ─── requires all
```

**Parallelism opportunities:**
- After Phase 4, **Phases 5, 6, and 7** can be developed concurrently by separate streams (agent, expense, management). Phase 7 depends on Phase 6 for expense aggregation.
- Phase 8's webhook infrastructure (8.1) is a shared dependency for multi-channel receipt capture (6.2) and risk/disruption (8.2–8.3); if Phases 6 and 8 run in parallel, build 8.1 first.
- Frontend (Phase 9) can begin shell + auth (9.1) as soon as Phase 2 ships, with feature pages added as their backends land.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`).
3. Linting and formatting pass (`ruff check` + `ruff format --check`).
4. Type checking passes (`mypy --strict`).
5. New/changed Alembic migrations created and `alembic upgrade head` / `downgrade base` round-trip cleanly.
6. Docker build succeeds and `docker-compose up` yields a green `/health`.
7. The phase's headline capability works end-to-end against the **mock supplier** and **mocked LLM** (no paid external calls required in CI).
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec with descriptions and examples.
9. New config/env variables documented in `.env.example` and `README.md`.
10. Row-level-security tenant isolation verified for any new tenant tables.
11. Audit-log coverage added for any new mutating action (compliance requirement).
12. New JSONB document shapes have a corresponding Pydantic model and validation tests (the hybrid model relies on app-layer validation).
```
