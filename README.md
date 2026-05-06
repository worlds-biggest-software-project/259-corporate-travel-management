# Corporate Travel Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source corporate travel and expense platform that unifies booking, policy enforcement, and expense automation through conversational interfaces.

Corporate Travel Management is a candidate project to build an open alternative to the dominant proprietary travel management suites (SAP Concur, Navan, TravelPerk, Amex GBT). It targets corporate travel managers, finance leaders, and HR teams who need policy-compliant booking, real-time spend visibility, and duty-of-care monitoring without the complexity and pricing of legacy TMC platforms.

---

## Why Corporate Travel Management?

- Incumbents like SAP Concur are powerful but carry steep learning curves, legacy UIs in core modules, and per-user pricing that creates scaling friction.
- Tech-native players such as Navan deliver modern UX but use custom enterprise pricing that effectively excludes mid-market companies (typically requiring six-figure annual travel spend).
- Managed-service TMCs (BCD Travel, CWT, Amex GBT) provide global reach but iterate slowly, are service-heavy, and offer limited self-serve technology compared with SaaS-first competitors.
- Most platforms support conversational search but still require UI interaction for approval and expense coding — no incumbent offers a fully conversational, end-to-end booking-to-expense workflow.
- Open standards adoption (IATA NDC, ISO 31030) is fragmented across vendors, and an open-source platform can offer transparent NDC integration with reported 6–12% air-spend savings.

---

## Key Features

### Booking and Policy Enforcement

- Travel booking for flights, hotels, rental cars, and ground transport
- Policy-driven booking workflow with real-time compliance checking before approval
- Multi-level approval workflow automation
- GDS connectivity (Amadeus, Sabre, Travelport) for inventory
- Multi-currency and multi-language support for global programmes

### Expense Automation

- Expense report automation with receipt capture and GL code assignment
- Corporate card integration with auto-reconciliation
- Real-time spend visibility dashboards
- Multi-channel expense capture (SMS, email, card auto-sync)

### Conversational and AI-Native Workflow

- Conversational AI booking assistant for natural language trip requests
- AI expense agent that reads receipts, applies GL codes, and matches card charges
- Personalised travel recommendations learning preferences and budget constraints
- Spend leakage detection identifying policy violations and unused negotiated discounts
- Dynamic policy optimisation recommending rule adjustments from actual booking behaviour

### Duty of Care and Risk Management

- Real-time traveller location tracking against safety intelligence feeds
- Proactive disruption alerts and automated rebooking options
- ISO 31030–aligned travel risk management
- 24/7 traveller support interface

### Standards and Distribution

- IATA NDC integration for direct airline content distribution
- PCI DSS–compliant payment handling for corporate and virtual cards
- GDPR / CCPA–aligned handling of PNR and expense data
- Microsoft 365 (Teams, Outlook) integration for in-app travel and expense tasks

---

## AI-Native Advantage

AI shifts corporate travel from form-driven workflows to conversational, proactive operations. A single natural-language request can search NDC and GDS inventory, check policy compliance, apply preferred-supplier rates, and pre-code an expense — eliminating the traditional booking-tool UI. Continuous AI monitoring tracks traveller locations against real-time risk feeds and triggers automated duty-of-care actions, while spend-pattern analysis surfaces policy leakage and unused discounts directly to programme managers. AI-driven policy optimisation and supplier negotiation modelling let teams test cost and traveller-experience tradeoffs before changes are committed.

---

## Tech Stack & Deployment

The project targets a SaaS-style deployment with self-hosted and hybrid options to suit enterprise data-residency and duty-of-care requirements. Integrations include GDS systems (Amadeus, Sabre, Travelport), IATA NDC airline connections, corporate card processors (Amex, Visa, Mastercard), expense and ERP systems, and Microsoft 365 / Slack / Teams for in-app workflows. Mobile clients support on-trip booking, expense submission, and approvals. Compliance scope includes PCI DSS, GDPR, CCPA, ISO 31030, and IATA BSP / ARC settlement procedures where applicable.

---

## Market Context

The corporate travel management software market was valued at approximately USD 1.17–1.26 billion in 2025–2026 and is projected to reach USD 1.66 billion by 2030 at a CAGR of roughly 7.2% (The Business Research Company, 2026; Research and Markets, 2026). Incumbent pricing ranges from Zoho Expense at $3/user/month and SAP Concur from ~$9/user/month, to TravelPerk from $99/month plus booking fees, Engine at $450/year membership, and custom enterprise pricing for Navan, Amex GBT, CWT, and BCD Travel. Primary buyers are corporate travel managers at companies spending $1M+ on travel annually, plus CFOs, HR leaders responsible for duty of care, and procurement directors managing supplier programmes.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
