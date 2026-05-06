# Standards & API Reference

> Project: Corporate Travel Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 31030:2021 — Travel Risk Management: Guidance for Organizations**
- URL: https://www.iso.org/standard/54204.html
- Provides a structured framework for developing, implementing, evaluating, and reviewing travel risk management (TRM) policy and programmes, covering threat identification, risk assessment, traveller safety protocols, and crisis response planning. Based on the broader ISO 31000 risk management principles. This is the primary international standard for corporate duty of care obligations; any platform featuring traveller safety, real-time location tracking, or emergency response capabilities should align with ISO 31030.

**ISO 31000:2018 — Risk Management: Guidelines**
- URL: https://www.iso.org/standard/65694.html
- Parent risk management standard underpinning ISO 31030. Provides universal principles for risk identification, analysis, and treatment across an organisation. Relevant for the overall risk governance architecture of any corporate travel programme.

### W3C & IETF Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational standard for delegated authorisation used by all major corporate travel APIs (TravelPerk, SAP Concur, Amadeus, Sabre). Any developer integrating with these platforms must implement the OAuth 2.0 authorisation code flow. TravelPerk explicitly uses OAuth 2.0 for its REST API at `https://app.travelperk.com/api/v2`.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is the standard token format used by OAuth 2.0 and OpenID Connect in corporate travel platforms. Used for stateless authentication between travel management tools and third-party ERP, HR, and duty of care integrations.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on OAuth 2.0 providing user authentication for corporate SSO. Used by enterprise identity providers (Okta, Microsoft Entra ID) that corporate travel tools integrate with for employee login. Required for SAML-to-OIDC bridge architectures common in enterprise deployments.

**RFC 7643 / RFC 7644 — SCIM 2.0 (System for Cross-domain Identity Management)**
- URL: https://datatracker.ietf.org/doc/html/rfc7643, https://datatracker.ietf.org/doc/html/rfc7644
- Standard protocol for automated user provisioning and deprovisioning between HR systems and SaaS applications. TravelPerk supports SCIM 2.0 at `https://app.travelperk.com/api/scim/v2` (Premium plan). Enables synchronisation of employee records from HR systems (Workday, BambooHR, SAP SuccessFactors) into the travel platform without manual onboarding.

### Data Model & API Specifications

**IATA NDC (New Distribution Capability) — Schema 21.3 / 24.1**
- URL: https://developer.iata.org/en/ndc/ · Program: https://www.iata.org/en/programs/airline-distribution/retailing/ndc/
- XML-based data transmission standard enabling airlines to communicate directly with travel agents, TMCs, and OTAs, bypassing traditional GDS intermediaries. NDC is a set of standard XML messages (based on W3C XML schemas) enabling offer-and-order-based airline retailing. NDC schema 24.1 (released 2025) introduced enhanced security, accounting, and interoperability features. Platforms adopting NDC report 6–12% cost savings on air spend. Certified by 73+ airlines as of 2026.

**OpenTravel Alliance (OTA) Specification — 2.0 / OAS 3.0**
- URL: https://opentravel.org/ · Download: https://opentravel.org/download-the-opentravel-specification/
- Industry-standard XML and JSON schemas (via OAS 3.0) for electronic exchange of travel booking data across airlines, hotels, car rental, and cruise. OpenTravel's DEx tool transforms the model into both XML (.xsd) and JSON (OAS 3.0) formats. Supports W3C, IATA, and ISO standards. Widely adopted for hotel and car rental data exchange between TMC platforms and suppliers.

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The REST API description standard used by major travel platforms including Amadeus (developer.amadeus.com), Travelport (OAS/Swagger files for Air APIs), and NetSuite REST API Browser. Any new corporate travel platform should publish its API using OpenAPI 3.1 to enable partner integrations and SDK generation.

**IATA BSPlink API / IATA Financial Gateway (IFG)**
- URL: https://www.iata.org/en/services/finance/bsp/
- The Billing and Settlement Plan (BSP) is IATA's electronic billing system for reconciling sales between travel agents and airlines. The IFG provides API services for upfront validation and real-time transaction control. Required for TMC platforms handling airline ticketing settlement outside the US. The US equivalent is ARC (Airline Reporting Corporation) at https://www2.arccorp.com/.

**GHG Protocol — Scope 3 Category 6 (Business Travel)**
- URL: https://ghgprotocol.org/corporate-value-chain-scope-3-standard · Category 6: https://plana.earth/glossary/scope-3-category-6
- The Greenhouse Gas Protocol's Corporate Value Chain Standard defines business travel emissions reporting under Scope 3 Category 6. Mandated by EU CSRD (Corporate Sustainability Reporting Directive) for large companies. APIs such as emissions.dev implement GHG Protocol Scope 3 Category 6 calculations for flights and hotels. Any travel platform offering carbon reporting must align with GHG Protocol methodology and CSRD disclosure requirements.

### Security & Authentication Standards

**PCI DSS v4.0.1 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/standards/
- Global security standard for entities that store, process, or transmit cardholder data. Released June 2024. Travel agencies processing card payments must complete a Self-Assessment Questionnaire (SAQ) and — if systems connect to the internet — quarterly vulnerability scans with a PCI SSC Approved Scanning Vendor. Corporate travel platforms handling virtual cards, expense card reconciliation, and payment processing must achieve and maintain PCI DSS v4 compliance. IATA provides specific PCI DSS guidance for travel agents at https://www.iata.org/en/services/finance/pci-dss/.

**SAML 2.0 — Security Assertion Markup Language**
- URL: https://docs.oasis-open.org/security/saml/v2.0/
- XML-based standard for federated enterprise SSO. Widely used in enterprise deployments where corporate identity providers (Microsoft ADFS, Okta, Ping Identity) authenticate employees before granting access to travel booking platforms. SAP Concur, Navan, and TravelPerk all support SAML 2.0 federation.

**GDPR (EU) / EU PNR Directive / CCPA (California)**
- GDPR: https://gdpr-info.eu/ · EU PNR Directive: https://www.europarl.europa.eu/legislative-train/theme-area-of-justice-and-fundamental-rights/file-eu-passenger-name-record-(european-pnr) · IATA data privacy: https://www.iata.org/en/programs/passenger/data-protection-privacy/
- GDPR governs the collection, processing, and retention of traveller Passenger Name Record (PNR) data, including itinerary details, email addresses, payment methods, and IP addresses. EU PNR Directive mandates collection of passenger data for law enforcement purposes with strict six-month unmasked retention limits. PNR data containing sensitive categories (health, religion, ethnicity) faces additional restrictions. CCPA applies to California-resident traveller data. Corporate travel platforms operating in the EU must implement data minimisation, consent management, and right-to-erasure capabilities.

### MCP Server Specifications

The Model Context Protocol (MCP) is directly relevant to AI-native corporate travel platforms that use LLM agents for conversational booking, expense coding, and policy checking.

- **MCP Specification**: https://modelcontextprotocol.io/
- Provides a standardised interface for LLM agents to interact with external tools and data sources. For a corporate travel platform, MCP servers can expose: GDS flight/hotel search tools, policy compliance checkers, expense GL coding engines, and duty of care risk data feeds — enabling any LLM (Claude, GPT-4o, Gemini) to operate as a travel booking agent without custom API wrappers per provider.
- Sabre has published a beta MCP-compatible Booking Management API endpoint at `https://developer.sabre.com/rest-api/mcp-booking-management-api/v1`, signalling that GDS providers are beginning to support AI-agent access patterns natively.

---

## Similar Products — Developer Documentation & APIs

### SAP Concur

- **Description:** Market-leading corporate travel and expense management platform. APIs cover travel booking requests, expense reports, receipt capture, GL coding, and approval workflows.
- **API Documentation:** https://developer.concur.com/
- **API Hub (SAP):** https://api.sap.com/products/SAPConcur/apis/REST
- **GitHub (preview docs):** https://github.com/SAP-docs/preview.developer.concur.com
- **Key APIs:** Travel Request v4, Expense Report v4, Receipt Capture v4, User Provisioning v4
- **Standards:** REST/JSON, OAuth 2.0, OpenAPI; SAML 2.0 for SSO
- **Authentication:** OAuth 2.0 (required for v4 APIs)
- **Notes:** Travel Request v4 requires Concur Request module purchase. Expense API v4 is actively evolving; legacy v3 APIs still in use at many enterprise customers.

### Amadeus for Developers

- **Description:** Self-service and enterprise REST APIs for flight search and booking, hotel availability and booking, car rental, destination content, and ancillary services. Supports 400+ airlines and 2M+ lodging properties.
- **API Documentation:** https://developers.amadeus.com/
- **Postman Collection:** https://www.postman.com/amadeus4dev/amadeus-for-developers-s-public-workspace/documentation/kquqijj/amadeus-for-developers
- **SDKs:** Node.js, Python, Java, Ruby, PHP, iOS, Android — via https://developers.amadeus.com/self-service/apis-docs/guides/developer-guides/sdks/
- **Standards:** REST/JSON; OpenAPI 3.0; also SOAP XML (legacy enterprise APIs)
- **Authentication:** OAuth 2.0 (client credentials flow for self-service)
- **Notes:** Two tiers — Self-Service (sandbox + production, no commercial agreement needed) and Enterprise (requires commercial contract). NDC content available via enterprise tier.

### Sabre Developer Hub

- **Description:** GDS-based REST and SOAP APIs for flight/hotel/car booking, PNR management, seat selection, and ancillary services. Sabre's Corporate Travel Services (CTS) exposes dedicated corporate booking APIs.
- **API Documentation:** https://developer.sabre.com/
- **Booking Management API:** https://developer.sabre.com/docs/rest_apis/trip/orders/booking_management
- **Corporate Travel Services:** https://developer.sabre.com/docs/rest_apis/air/reservation/corporate_travel_services/release-notes
- **MCP Beta API:** https://developer.sabre.com/rest-api/mcp-booking-management-api/v1
- **Standards:** REST/JSON (modern), SOAP XML (legacy); Swagger/OpenAPI for REST endpoints
- **Authentication:** OAuth 2.0 (REST); session tokens (SOAP)
- **Notes:** REST APIs recommended for new integrations. SOAP APIs still required for some legacy operations. MCP beta endpoint signals AI-agent readiness.

### Travelport Developer Portal

- **Description:** GDS connectivity via Universal API (SOAP) and modern RESTful JSON microservices for air, hotel, and payment. Also branded as Travelport TripServices for AI-native travel integrations.
- **API Documentation:** https://developer.travelport.com/
- **REST/JSON APIs:** https://developer.travelport.com/restful-json-api
- **Universal API (SOAP):** https://support.travelport.com/webhelp/uapi/uapi.htm
- **GitHub:** https://github.com/Travelport
- **SDKs:** Java, PHP, C# tutorials via GitHub repos; OAS (Swagger) files and Postman collections for REST APIs
- **Standards:** REST/JSON (modern); SOAP XML (Universal API); OpenAPI 3.0
- **Authentication:** OAuth 2.0 (REST); credential-based session tokens (SOAP)
- **Notes:** Two parallel API generations — Universal API (SOAP, mature, broad coverage) and RESTful JSON microservices (modern, growing). OAS files downloadable from developer portal.

### TravelPerk Developer Hub

- **Description:** Corporate travel booking platform with an open REST API and webhooks for invoices, users, account configuration, trips, and travel information. Designed for integration with HR systems, ERPs, and flight tracking systems.
- **API Documentation:** https://developers.travelperk.com/docs
- **Developer Hub:** https://developers.travelperk.com/
- **Expenses API:** https://support.travelperk.com/hc/en-us/articles/360046079392-Getting-the-Expenses-API
- **Standards:** REST/JSON, OAuth 2.0, SCIM 2.0 (Premium), Webhooks
- **Authentication:** OAuth 2.0 (authorisation code flow); SCIM 2.0 bearer token for user provisioning
- **Notes:** Sandbox environment available. SCIM 2.0 endpoint (`https://app.travelperk.com/api/scim/v2`) on Premium plan. Webhooks for event-driven integrations (e.g., new invoice, booking change).

### Navan (formerly TripActions)

- **Description:** Unified travel, expense, and corporate card platform. API covers travel booking data (flights, hotels), expense transactions, card programme management, and TMC integration.
- **API Documentation:** https://app.navan.com/app/helpcenter/articles/travel/admin/other-integrations/navan-tmc-api-integration-documentation
- **API Tracker:** https://apitracker.io/a/navan
- **Airbyte Connector (travel data export):** https://docs.airbyte.com/integrations/sources/navan
- **Standards:** REST/JSON; OAuth 2.0
- **Authentication:** Client ID and Client Secret (admin-generated); OAuth 2.0
- **Notes:** Enhanced API connection with Booking.com established January 2026, expanding lodging inventory. TMC API allows third-party travel agents to manage Navan bookings.

### Everbridge — Travel Risk Management APIs

- **Description:** Critical event management and duty of care platform. Integration with International SOS TravelTracker provides real-time employee location data from travel itineraries and enables location-based emergency notifications to travelling employees.
- **Product Page:** https://www.everbridge.com/products/travel-protector/
- **Integration Resource:** https://www.everbridge.com/resource/safety-connection-international-sos-traveltracker-integration/
- **Standards:** REST API; event-driven webhooks for risk alert delivery
- **Authentication:** API key + OAuth 2.0
- **Notes:** Integrates with corporate travel booking platforms via itinerary data feeds (flights, hotels, rail). Powers proactive duty of care: location matching against real-time security/weather/health risk intelligence.

### emissions.dev — Business Travel Carbon API

- **Description:** Dedicated API for calculating business travel emissions (flights, hotels) compliant with GHG Protocol Scope 3 Category 6 and EU CSRD requirements, with full audit trail source documentation.
- **API Documentation:** https://emissions.dev/business-travel
- **Standards:** GHG Protocol Scope 3 Category 6; EU CSRD; REST/JSON
- **Authentication:** API key
- **Notes:** Provides verifiable, auditable carbon calculations suitable for corporate sustainability reporting. Increasingly required as CSRD mandates Scope 3 Category 6 quantification for large EU companies from 2025–2026 onwards.

---

## Notes

**GDS Access Complexity:** Direct access to Amadeus Enterprise, Sabre, or Travelport for a new product requires a commercial agreement (certification as an OTA or TMC). Self-service tiers (Amadeus for Developers) provide a lower-barrier path but may carry GDS booking fees per transaction. New entrants building AI-native tools should evaluate a GDS aggregator (e.g., Duffel, Kiwi.com B2B) as a faster path to content without direct GDS certification.

**NDC Fragmentation:** Despite being an IATA standard, NDC implementation varies significantly per airline — each carrier exposes different subsets of NDC schema versions (21.3, 24.1) with proprietary extensions. Platforms must normalise across multiple carrier NDC dialects, which is a significant engineering investment. NDC aggregators (e.g., Verteil, Airgateway) reduce this burden.

**Emerging AI-agent patterns:** Sabre's MCP beta endpoint is an early signal that GDS providers may expose AI-agent-native interfaces alongside traditional REST/SOAP APIs. Building on MCP from the outset would position a new platform to natively orchestrate LLM-based booking agents across multiple GDS and NDC content sources without bespoke API wrappers.

**CSRD and sustainability reporting:** EU CSRD creates a new compliance requirement driving demand for native carbon tracking within travel platforms. This is currently an underserved area; integrating an emissions calculation API (e.g., emissions.dev) as a standard feature — rather than a bolt-on — would differentiate a new entrant in the European market.
