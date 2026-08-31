# Intelligent Procurement Management Portal

> **Value proposition:** Design and implementation of a purpose-built procurement intelligence platform that replaced 7+ years of manual spreadsheet-based analysis, cutting execution time for critical purchasing processes by up to 99% — without investing in cloud infrastructure or licensing a new ERP.

---

## 1. Executive Summary

**Industry:** Precision manufacturing (machined components and assemblies for the aerospace and oil exploration sectors)
**My Role:** Internal Solution Architecture, Development & Deployment
**Project Type:** Production
**Status:** In active production (since the first module went live, with ongoing incremental releases)
**Technologies:** .NET 8 / ASP.NET Core MVC · SQL Server · Generative AI (API) · Entity Framework Core

### Business Impact

- **Procurement analysis time (Max/Min, Weekly Operations, High Runner, Winder, Flagging):** from ~30 minutes per analysis → ~1 minute (**~97% reduction**)
- **Purchase order follow-up time (High Runner, General PO, Max/Min, Special Treatments, Requests):** from ~2 hours per follow-up → ~1 minute (**~99% reduction**)
- **Operational tracking meeting (OTD):** from 2 hours → 5 minutes (**~96% reduction**)
- **Freed-up capacity:** the equivalent of one full workday (9 h/day) for one employee, redirected to higher-value work
- **New capabilities that didn't exist before:** AI-assisted analysis and consolidation of information into a single system

---

## 2. The Business Challenge

### Context

The procurement department of a mid-sized precision manufacturing company (~180 employees) had operated for more than 7 years relying exclusively on office tools (Excel and Word) to run critical sourcing processes: reorder point calculations, purchase order follow-up, strategic materials analysis, and specialized tooling management.

The company had spent two years evaluating the acquisition of a more modern ERP that could address these limitations, but the licensing cost was prohibitive relative to the size of the operation, so the initiative kept getting postponed.

### Problem

The manual process generated daily, highly visible operational friction:

- Purchases that were poorly sized or executed too late.
- Lack of traceability: there was no clear visibility into the real status of each purchase order.
- Recurring shortages of tooling critical to production.
- Delays in export documentation at the country of origin, caused by a lack of timely follow-up.
- Poor communication between the procurement department and related areas.

In addition, every analysis depended on the individual judgment of each analyst, with no standardized, reproducible, or auditable process, and purchasing decisions lacked any formal backing (history, approvals, or supporting data).

### Objectives

- Eliminate manual work in inventory, cost, and purchase order analysis.
- Standardize and automate sourcing calculations (Max/Min, shortage-risk analysis, tooling management).
- Provide full traceability for every purchasing decision, from initial calculation to final approval.
- Incorporate AI-assisted analysis capabilities to support decision-making.
- Do all of this within the real infrastructure and budget constraints of a mid-sized company.

### Constraints

- **Budget:** no economic feasibility for acquiring a modern ERP or contracting dedicated cloud infrastructure.
- **Infrastructure:** mandatory on-premise deployment, on a server already existing within the company.
- **Untouchable legacy system:** the corporate ERP could not be modified or receive writes from any new solution — only read access was permitted.
- **Technical team:** the company's IT department is small and focused on support, not development — the project was executed by a single person responsible for both architecture and development.
- **Timeline:** no hard deadline; an incremental delivery approach was chosen, validating each module against real production data as it progressed.

---

## 3. Requirements

### Functional Requirements

- Automatically and periodically synchronize relevant information from the ERP (items, inventory, suppliers, purchase and sales orders, planning) without altering the source system.
- Calculate reorder points and sourcing needs in a standardized way (Max/Min, materials explosion, specialized tooling consumption management).
- Provide centralized tracking of the status of purchase orders, requests, and special treatments.
- Support a formal Calculation → Review → Approval workflow, with an immutable history of every decision.
- Incorporate an AI-assisted analysis module for complex purchasing scenarios.
- Allow analyses and reports to be exported in standard office formats (Excel, PDF).
- Manage users, roles, and permissions in a fully administrable way from within the platform itself.

### Non-Functional Requirements

- **Security:** custom authentication with passwords protected via secure hashing; role-based access control (RBAC), where each role determines both what information is visible and what functionality the user can execute; credentials for external services (such as the AI engine) managed through protected configuration variables, never exposed directly in code; only administrator users can reset passwords.
- **Performance:** analyses must run against a local copy of the data, avoiding any impact on ERP performance.
- **Availability:** the system must continue operating on local data even if the ERP is temporarily unavailable.
- **Scalability:** support growth in the number of analysis modules and data volume without degrading performance.
- **Usability:** a dashboard-style interface, visually appealing, with a clear visual identity (colors and intuitive iconography) that facilitates adoption by non-technical users.
- **Maintainability:** meet baseline security and design best practices expected of an internal, department-level system.

---

## 4. Architecture

### As-Is

The sourcing process depended on manual exports from the ERP into spreadsheets, where each analyst ran their own reorder, follow-up, and risk-analysis calculations. There was no centralized repository for these decisions, and no traceability of who had calculated what, when, or under what criteria.

**Diagram:**
![Diagrama As-Is](diagrams/as-is.png)

### To-Be

A web-based intranet platform was designed to act as an **intelligence layer on top of the existing ERP**, without replacing or modifying it. The portal periodically synchronizes relevant information into a local database, against which all analyses, calculations, and approval workflows are executed. An AI engine complements human analysis in higher-complexity scenarios, and every interaction is logged for audit purposes.

**High-level architecture diagram:**
![Diagrama To-Be](diagrams/to-be-hld.png)

### Main Components

| Component | Responsibility |
|---|---|
| **Corporate ERP (legacy system)** | Source of truth for items, inventory, suppliers, and orders; consumed in read-only mode |
| **Synchronization engine** | Keeps a local copy of ERP information up to date on periodic cycles, without affecting its operation |
| **Local analysis repository** | Stores the synchronized ERP copy, business configuration, and decision history |
| **Business analysis engine** | Executes standardized sourcing calculations (reorder, shortage risk, tooling management) |
| **AI engine** | Provides AI-assisted analysis for complex purchasing scenarios, with independent configuration per analysis type |
| **Web portal (dashboard)** | Central interface for consultation, follow-up, approval, and reporting for the procurement team |

---

## 5. Architecture Decisions

### Decision 1 — Build a custom solution instead of acquiring a modern ERP

**Decision:** develop a purpose-built platform instead of replacing the existing ERP.

**Why:** for two years, the acquisition of a modern ERP was evaluated, but the cost was incompatible with the scale of the operation. The rise of generative AI opened the possibility of building an in-house solution, tailored exactly to the procurement department's needs, with a small internal team.

**Alternatives considered:** acquiring a commercial modern ERP / continuing to operate with spreadsheets.

**Trade-off:** gained a solution 100% tailored to the real business process at a fraction of the cost of a commercial ERP, in exchange for taking on full internal responsibility for design, development, and maintenance.

### Decision 2 — On-premise deployment instead of cloud infrastructure

**Decision:** deploy the solution on the company's own infrastructure instead of a cloud environment.

**Why:** no budget was available to contract dedicated cloud infrastructure or specialized hosting. The company already had an internal server capable of supporting the solution.

**Alternatives considered:** dedicated cloud hosting.

**Trade-off:** avoided a recurring infrastructure cost, in exchange for taking on server management and availability using existing internal IT resources (which are focused on support).

### Decision 3 — The ERP as a strictly read-only source of truth

**Decision:** the portal never writes information back to the ERP; it only consumes it.

**Why:** the corporate ERP is a critical, shared system across the entire operation; allowing writes from a new solution posed an unacceptable risk of data corruption.

**Alternatives considered:** bidirectional integration with the ERP.

**Trade-off:** gave up the ability for the portal to directly update the ERP (for example, to close out a purchase order cycle), in exchange for completely eliminating the risk of affecting the company's critical operations.

### Decision 4 — Periodic local synchronization instead of real-time ERP queries

**Decision:** copy relevant ERP information into a local repository on periodic cycles, instead of querying the ERP directly for every analysis.

**Why:** the ERP was not designed to support complex, frequent analytical queries. Working against a local copy allows heavy calculations to run without impacting the source system's performance, and also provides operational continuity even if the ERP is temporarily offline.

**Alternatives considered:** real-time queries against the ERP for each analysis.

**Trade-off:** introduces a slight lag between an ERP update and its reflection in the portal, in exchange for greater stability, performance, and resilience overall.

### Decision 5 — Formal Calculation → Review → Approval workflow

**Decision:** every purchasing analysis goes through a three-stage workflow before becoming a formal decision, recorded as immutable history.

**Why:** the department lacked a reproducible, auditable process; decisions depended on individual judgment with no documented backing.

**Alternatives considered:** keeping analysis and approval as a single, informal step.

**Trade-off:** adds an extra step to the operational workflow, in exchange for full traceability and audit capability over every purchasing decision.

---

## 6. Security & Risk Considerations

- **Authentication:** custom authentication mechanism, with passwords protected using secure hashing techniques; only an administrator user has the ability to reset other users' passwords.
- **Authorization:** role-based access control (RBAC) model. Each role granularly defines both the information visible in each view and the functionality the user can execute.
- **Data protection and external integrations:** credentials for external services (such as the AI engine) are managed through protected configuration variables, never exposed directly in the application code.
- **Legacy system protection:** the architecture guarantees, by design, that it is impossible to write information back to the ERP from the portal.
- **Auditing:** every relevant interaction (calculations, AI analyses, operational changes) is logged with user, date, time, and action detail, allowing the origin of any decision to be reconstructed.
- **Baseline security compliance:** the solution was designed to meet baseline security best practices expected of an internal, department-level system (credential management, access control, password hashing, traceability).

### Key Risks

| Risk | Mitigation |
|---|---|
| Accidental corruption or alteration of the corporate ERP | Strictly read-only access, enforced at the architecture level |
| Concurrent synchronization runs duplicating or corrupting data | Concurrency control per synchronization process |
| Temporary ERP unavailability | The portal operates independently against its local data copy |
| Exposure of external service (AI) credentials | Credential management through protected configuration, outside the source code |
| Purchasing decisions without backing or traceability | Formal approval workflow with immutable history |

---

## 7. Implementation

The project was executed incrementally: each module was designed, developed, and validated directly against real production data before moving on to the next, enabling partial deliveries in short cycles and early adjustments based on actual use by the procurement team.

As Internal Solution Architect, I independently led both the architecture definition and the full development of the solution, given the company's small IT team (focused on support). The initiative originated from a proposal I presented to management based on the operational problems identified in the procurement department, which was approved without objections along with the requested resources.

**Deployment:** On-premise, on existing server infrastructure within the company.

**Repository:** Not publicly available (internal corporate project).

**Demo:** Not publicly available.

---

## 8. Results & Business Impact

### Before

- Sourcing analysis (Max/Min, Weekly Operations, High Runner, Winder, Flagging): ~30 minutes per run, done manually in spreadsheets.
- Purchase follow-up (High Runner, General PO, Max/Min, Special Treatments, Requests): ~2 hours per run.
- Operational tracking meeting (OTD): ~2 hours.
- Proforma invoice creation: ~30 minutes.
- No AI-assisted analysis capability.
- Information scattered across multiple files with no central consolidation.

### After

- The same sourcing analyses now run in ~1 minute.
- Purchase follow-up now runs in ~1 minute.
- The operational tracking meeting was reduced to ~5 minutes.
- Proforma invoice creation now runs in ~1 minute.
- AI-assisted analysis was introduced as a new capability in the decision-making process.
- All procurement department information was consolidated into a single system.

### Key Results

- **Efficiency:** time reductions of between 96% and 99% across critical purchasing analysis and follow-up processes.
- **Freed-up capacity:** the equivalent of one full workday for one employee, redirected toward higher-responsibility, higher-value activities.
- **Quality:**  reduction of errors associated with manual spreadsheet calculations.
- **Traceability:** complete history of changes and decisions, with date, time, and origin of each calculation — a capability that did not exist before the project.
- **Adoption:** positive acceptance from the procurement team from day one, driven by a tailor-made solution that remained compatible with Excel.
- **Economic impact:** quantifying direct cost savings was not a project objective; the core purpose was operational simplification and standardization for the procurement department.

---

## 9. Lessons Learned

- Designing the solution as an intelligence layer on top of the legacy system — rather than attempting to replace it — delivered high-impact results with much lower risk and without requiring a major investment in a new ERP.
- Incremental deliveries, validated directly against real production data, made it possible to tailor each module to the procurement team's actual needs before scaling to the next one.
- Compatibility with the tools the team already knew (Excel) was key to achieving adoption without resistance.
- Maintaining a strict separation between the legacy system and the new platform (read-only) proved to be a low-cost, high-return decision in terms of risk mitigation.

---

## 10. Next Evolution

- Explore a controlled, audited write-back integration with the ERP to fully close the loop on certain purchase orders.
- Expand the use of the AI engine into new analysis scenarios (e.g., demand forecasting or early shortage-risk detection).
- Evaluate an eventual migration to cloud infrastructure if the growth of the operation justifies it.
- Extend the platform's model to other departments with similar operational processes.

---

## Screenshots

**Dashboard & AI-assisted analysis**

| Main dashboard | AI-assisted analysis catalog |
|---|---|
| ![Dashboard](screenshots/dashboard.png) | ![AI-assisted analysis](screenshots/analysis.png) |

Central module launcher, and the catalog of AI-assisted analyses (daily action plan, shortage-risk detection, High Runner planning, purchase-order triage, manufacturing feasibility).

**Sourcing & reorder analysis**

| Max/Min reorder module | High Runner tooling purchase module |
|---|---|
| ![Max/Min](screenshots/max-min.png) | ![High Runner](screenshots/high-runner.png) |

Automated reorder-point calculation across the full item catalog, and the specialized module for high-rotation tooling purchases by supplier.

**AI-assisted planning**

![AI High Runner plan](screenshots/IA-high-runner.png)

The AI engine determines which critical items will run out of stock, whether an open purchase order will arrive in time, and what action to take for each one.

**Operational tracking**

| Weekly Operations | OTD tracking meeting |
|---|---|
| ![Weekly Operations](screenshots/ops-sem.png) | ![OTD meeting](screenshots/OTD.png) |

Weekly material-explosion validation for the production plan, and the exception-case tracker used in the operational tracking meeting.

**Follow-up & documents**

| Follow-up module | High Runner follow-up view | Proforma invoice |
|---|---|---|
| ![Follow-up dashboard](screenshots/seguimiento-dashboard.png) | ![High Runner follow-up](screenshots/seg-high-runner.png) | ![Proforma invoice](screenshots/prof-inv.png) |

The follow-up module's sub-modules (High Runner, general POs, Max/Min, special treatments, requests), a per-item inventory-days-of-cover view against open purchase orders, and automated proforma-invoice generation by PO number.

---

## Project Resources

**Case Study:** Web Portfolio
**Architecture:** [arc42/procurement-intelligence-portal-arc42_en.md](arc42/procurement-intelligence-portal-arc42_en.md)
**Repository:** Not publicly available
**Demo:** Not publicly available

---

> **Confidentiality Notice:** This case study has been anonymized. Company-specific information, credentials, infrastructure details, proprietary data, and confidential implementation details have been removed or generalized.
