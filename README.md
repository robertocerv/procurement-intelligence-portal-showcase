# Intelligent Procurement Management Portal

> Purpose-built procurement intelligence platform that replaced 7+ years of manual, spreadsheet-based analysis — cutting execution time for critical purchasing processes by up to 99%, without cloud infrastructure or a new ERP license.

## The problem

The procurement department of a mid-sized precision manufacturing company had operated for over 7 years relying entirely on Excel and Word for critical sourcing decisions: reorder calculations, purchase order follow-up, and specialized tooling management. This led to late or poorly sized purchases, no traceability on order status, recurring tooling shortages, and no auditable record behind any purchasing decision.

## Impact

| Process | Before | After | Reduction |
|---|---|---|---|
| Sourcing analysis (5 modules) | ~30 min | ~1 min | ~97% |
| Purchase order follow-up (5 modules) | ~2 hours | ~1 min | ~99% |
| Operational tracking meeting | 2 hours | 5 min | ~96% |

Plus: the equivalent of one full 9-hour workday of capacity freed up per day, complete audit trail on every decision, and new AI-assisted analysis capability that didn't exist before.

## Architecture (To-Be)

The portal acts as a **read-only intelligence layer on top of the existing ERP** — it never writes back to the legacy system. Data is synced periodically into a local database, where a business analysis engine and an AI engine run all calculations. Every decision flows through a formal calculation → review → approval workflow with immutable history.

![To-Be architecture](diagrams/to-be-hld.png)

## Screenshots

| Dashboard | AI-assisted analysis |
|---|---|
| ![Dashboard](screenshots/dashboard.png) | ![AI-assisted analysis](screenshots/analysis.png) |

More screenshots (High Runner tooling, Max/Min reorder, OTD tracking, proforma invoices) are in the [full case study](Procurement-Intelligence-Portal-Case-Study.md#screenshots).

## Tech stack

.NET 8 / ASP.NET Core MVC · SQL Server · Entity Framework Core · Generative AI (API) · Role-based access control (RBAC)

## Key architecture decisions

- **Build vs. buy:** custom-built solution instead of a costly modern ERP, enabled by generative AI and a small internal team.
- **On-premise vs. cloud:** deployed on existing company infrastructure due to budget constraints.
- **Read-only ERP integration:** the legacy ERP is treated as an untouchable source of truth — zero write risk.
- **Local sync over real-time queries:** analyses run against a local copy to protect ERP performance and stay resilient to ERP downtime.

## Full case study

📄 [Read the full case study](Procurement-Intelligence-Portal-Case-Study.md) — business context, requirements, security, detailed results, and lessons learned.
📐 [Read the full arc42 architecture document](arc42/procurement-intelligence-portal-arc42_en.md)

---

> **Confidentiality notice:** This project has been anonymized. Company-specific information, credentials, infrastructure details, and confidential implementation details have been removed or generalized.
