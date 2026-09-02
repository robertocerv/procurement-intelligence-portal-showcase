# Software Architecture — Purchasing Portal (Portal de Compras)

**Format:** arc42\
**Status:** Complete MVP with the current architecture; operational risks documented in section 11\
**Target environment:** On-premises intranet server (Windows / IIS)

------------------------------------------------------------------------

## 1. Introduction and objectives

The Purchasing Portal is an on-premises intranet web application for the
company's Purchasing Department. Its purpose is to replace manual,
spreadsheet-driven procurement analysis with a centralized, audited,
and AI-assisted intelligence platform.

The system reads data from the corporate ERP through read-only SQL
views, copies it locally, and executes standardized purchasing workflows
against that local copy. It does **not** write to the ERP.

The portal provides the following main functions:

-   **Analysis modules:** Max/Min reorder points, BOM shortage risk
    (Bandera), Winder consumable coverage, Tool Explosion,
    High-Runner management.
-   **AI-assisted analysis:** Claude API integration for commentary on
    complex purchasing scenarios (7 configured modules).
-   **Approval workflow:** Calculate → Review → Approve → immutable
    historical snapshot for every analysis module.
-   **Order tracking:** open PO lines, solicitation follow-up, OTD
    exception management.
-   **Master data:** vendor-item catalog, payment terms, delivery
    addresses, item substitutions and exclusions.
-   **Operational records:** daily production log, weekly operations
    plan, daily tool consumption, RFQ workflow, goods receipt checklist.
-   **Administration:** user management, RBAC, dynamic menu permissions,
    ERP sync monitoring, AI configuration.

### Main objectives

-   Synchronize ERP data automatically into a local, queryable copy.
-   Execute standardized purchasing calculations with full parameter
    traceability.
-   Enforce human review and approval before any analysis becomes a
    recorded decision.
-   Augment analyst judgment with AI-generated commentary on complex
    scenarios.
-   Record every decision, calculation parameter, and AI interaction
    for compliance and reproducibility.
-   Keep the AI provider decoupled so it can be replaced
    (cloud ↔ local) without redesigning the application.

------------------------------------------------------------------------

## 2. Architecture constraints

-   Runs on an **on-premises Windows server within the corporate
    intranet**.
-   The corporate ERP is **strictly read-only**: the portal queries 8
    predefined SQL views and never writes back. This constraint is
    enforced at code level — `ErpDbContext.SaveChanges()` throws by
    design.
-   **SQL Server** is the portal's persistent database.
-   Authentication uses custom cookie-based auth with
    **PBKDF2-SHA256** — no ASP.NET Identity.
-   The AI provider is interchangeable via the `IAIService` interface:
    currently **Claude API (Anthropic)**, originally scoped as
    Gemma 2 / Ollama (local).
-   Credentials (API key, SMTP, database) are managed via
    `appsettings.json` / environment variables — not hardcoded.
-   Background ERP synchronization runs **in-process** inside the web
    application; no external job scheduler is required.
-   Departmental scale — tens of concurrent users; not designed for
    internet-scale traffic.

### Technologies not required by the MVP

For the current solution, in its Minimum Viable Product (MVP) version,
it has not yet been necessary to implement:

-   Kubernetes / Docker orchestration
-   Message brokers (Kafka, RabbitMQ)
-   Redis or distributed cache
-   Vector database / RAG
-   LangChain / LangGraph
-   SPA framework (React, Angular)
-   CI/CD pipeline
-   Formal HA / disaster recovery

The MVP is complete without them; their absence is not an outstanding
gap. However, they are not ruled out in the future if the problem or
the system's scale changes.

------------------------------------------------------------------------

## 3. System scope and context

### In scope

-   ERP data synchronization — read-only, background, periodic
    (4 h / 24 h by group).
-   Procurement analysis workflows: Max/Min, Bandera BOM shortage risk,
    Winder consumable coverage, Tool Explosion, High-Runner management.
-   AI-assisted analysis for complex purchasing scenarios
    (Claude API, 7 configured modules).
-   Formal approval workflow with immutable historical snapshots for all
    calculation modules.
-   Master data management: vendor-item catalog, payment terms, delivery
    addresses, exclusions, substitutions.
-   Order tracking: open PO lines, solicitation follow-up, OTD
    exception management.
-   Operational records: daily production log, weekly plan, daily tool
    consumption.
-   RFQ (Request for Quotation) workflow.
-   Goods receipt quality checklist.
-   Reporting: Excel (ClosedXML) and PDF (QuestPDF) export.
-   User management, RBAC, dynamic menu permissions.
-   ERP sync monitoring dashboard.

### Out of initial scope

-   Writing any data back to the corporate ERP.
-   ERP transaction processing (PO creation, goods receipts, invoice
    management).
-   Financial accounting or accounts payable.
-   Customer-facing portal or internet access.
-   Native mobile application.
-   Integration with external vendor portals.
-   High availability or horizontal scaling.
-   Formal CI/CD pipeline.

These are firm scope decisions for the MVP, not incomplete
functionality.

------------------------------------------------------------------------

## 4. Solution strategy

The solution follows a two-project layered architecture. The
`PortalCompras.Web` presentation layer delegates all business logic to
the `PortalCompras.LogicData` class library. Two flows run in parallel:
synchronous HTTP request handling for analyst interactions, and a
background in-process service for ERP data synchronization.

``` text
                CORPORATE INTRANET
                       │
                Browser / HTTP
                       │
                       ▼
       ┌────────────────────────────────┐
       │    ASP.NET Core MVC .NET 8     │
       │                                │
       │  [Web]    34 Controllers       │
       │           Razor Views (SSR)    │
       │           SyncBackgroundSvc    │
       │                                │
       │  [Logic]  Business Services    │
       │           Repository layer     │
       │           AppDbContext         │
       │           ErpDbContext (RO)    │
       └────────────────────────────────┘
              │             │        │
       SQL R/W│      SQL RO │  HTTPS │
              │             │        │
              ▼             ▼        ▼
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │ SQL Server │  │ ERP SQL    │  │ Claude API │
  │(PortalComp)│  │ Server     │  │ Anthropic  │
  │            │  │(on-prem.)  │  │ (cloud)    │
  │ • Users    │  │            │  └────────────┘
  │ • Analysis │  │ 8 SQL      │
  │ • History  │  │ views (RO) │
  │ • SyncLogs │  └────────────┘
  └────────────┘
```

------------------------------------------------------------------------

## 5. Building block view

### 5.1 Frontend (Razor Views)

**Technologies**

-   ASP.NET Core MVC server-side Razor
-   Bootstrap
-   ClosedXML (Excel export)
-   QuestPDF (PDF export)
-   Vanilla JS / custom CSS

**Responsibilities**

-   Render analysis grids, dashboards, and operational records.
-   Present the Calculate → Review → Approve workflow per analysis
    module.
-   Display AI-generated commentary alongside analyst input.
-   Serve Excel and PDF exports.
-   Expose the administration console: users, roles, menus, AI
    configuration, sync monitoring.

### 5.2 Backend

**Technologies**

-   C# / .NET 8
-   ASP.NET Core MVC
-   Entity Framework Core 8 — Code-First
-   .NET IHostedService (background sync)

The backend is split into two projects with a strict one-way dependency:
`Web → LogicData`.

#### Controllers (PortalCompras.Web)

34 MVC controllers, one per business domain. Each delegates business
logic to the corresponding Service — no business logic in controllers.

#### Business Services (PortalCompras.LogicData)

11 services: authentication, menu, email, AI integration, ERP sync,
Max/Min, Bandera, Tool Explosion, Winder, RFQ, OTD.

#### Repository layer

`IRepository<T>` generic pattern plus 5 specialized repositories
(`UserRepository`, `AIConfigurationRepository`, `FormulaRepository`,
`PromptHistoryRepository`, `MenuRepository`). All database access goes
through repositories — controllers never touch EF Core directly.

#### AppDbContext

Read-write EF Core context for all portal-owned data. 77 migrations
manage the schema. `DataSeeder` bootstraps users, roles, menus, AI
configurations, formulas, and sync table configuration on first run.

#### ErpDbContext

Read-only EF Core context mapping 8 ERP SQL views. `SaveChanges()` and
`SaveChangesAsync()` are overridden to throw `InvalidOperationException`
— the ERP constraint is enforced in code, not just policy.

#### SyncBackgroundService

In-process .NET hosted service (`PortalCompras.Web/Services/`) running
ERP synchronization on two independent `PeriodicTimer` loops:

-   **Group A (every 4 h):** operational data — inventory, open PO
    lines, purchase orders, sales orders, parts inventory, solicitations.
-   **Group B (every 24 h):** master/reference data — items, vendors,
    planning details, current materials, customers, sub-assemblies,
    PO item prices.

UPSERT in batches of 500 rows. Per-table `SemaphoreSlim` prevents
overlapping runs.

#### AIService

Retrieves per-module `AIConfiguration` (system prompt, temperature,
token limits) from the database, builds the Claude API payload, calls
`POST /v1/messages`, and records every call in `PromptHistory` (full
audit: user, prompts, response, token counts, duration, status).

### 5.3 Approval workflow (cross-module pattern)

All analysis modules follow an identical three-phase pattern. No
analysis result enters the permanent record without explicit human
approval:

-   **Calculate:** service writes results to `[Module]Calculo`
    (mutable; replaces prior tentative calculation).
-   **Review:** analyst may edit individual rows (quantities,
    observations).
-   **Approve + Snapshot:** creates `[Module]Aprobacion` record and
    copies all rows into `[Module]HistoricoDetalle` (immutable).

### 5.4 AI integration

`IAIService` interface decouples the application from any specific AI
provider. Currently implemented against the Claude API (Anthropic).
Switching to a local model (Gemma 2 / Ollama) requires only
reimplementing `AIService` — no controller or business service changes.

Configured modules: `STOCK_ANALYSIS`, `PRICE_ANALYSIS`,
`SITUACION_DIARIA`, `RIESGO_DESABASTO`, `PLAN_HIGH_RUNNER`,
`TRIAGE_OCS`, `VIABILIDAD_FABRICACION`.

------------------------------------------------------------------------

## 6. Runtime view

### 6.1 ERP synchronization (background)

``` text
SyncBackgroundService (timer — Group A / B)
      │
      ▼
SyncService
      │
      ├── Acquire SemaphoreSlim (per table)
      │
      ▼
ErpDbContext → ERP SQL View (SELECT only)
      │
      ▼
AppDbContext → Local* table (UPSERT, batches of 500)
      │
      ├── INSERT SyncLog (row count, duration, status)
      │
      └── Release SemaphoreSlim
```

### 6.2 Analysis calculation and approval

``` text
Analyst (browser)
      │
      ▼
MVC Controller → Business Service
      │
      ├── Queries Local* tables (AppDbContext)
      │
      ▼
[Module]Calculo (mutable — DELETE + INSERT)
      │
      ▼
Analyst reviews and edits grid (optional)
      │
      ▼
Approve action
      │
      ├── INSERT [Module]Aprobacion (userId, timestamp)
      │
      └── INSERT [Module]HistoricoDetalle (immutable snapshot)
```

### 6.3 AI-assisted analysis

``` text
Analyst submits question (browser)
      │
      ▼
Controller → AIService.AnalyzeAsync()
      │
      ├── Retrieves AIConfiguration (system prompt, temperature, maxTokens)
      ├── Retrieves Formula context (injected into prompt)
      │
      ▼
POST https://api.anthropic.com/v1/messages
      │
      ▼
INSERT PromptHistory (full audit record)
      │
      ▼
AI commentary rendered to analyst
```

### 6.4 User authentication

``` text
Browser → POST /Account/Login
      │
      ▼
UserService.AuthenticateAsync()
      │
      ├── SELECT AppUser (SQL Server)
      ├── PasswordHasher.Verify() — PBKDF2-SHA256, constant-time
      ├── UPDATE LastLogin
      │
      └── Set auth cookie (8 h sliding, role claims)
            │
            ▼
      Protected routes enforce [Authorize] — redirect to /Login if absent
```

------------------------------------------------------------------------

## 7. Persistence and memory

**Database:** SQL Server (on-premises, same network as ERP).

SQL Server is the system's source of persistence. The AI is **not** the
persistent memory — the model produces analytical commentary; the
application stores and structures the results.

Conceptually:

``` text
ERP SQL Views (read-only)
    │
    ▼
SyncBackgroundService (UPSERT every 4–24 h)
    │
    ▼
Local* tables (PortalCompras DB)
    │
    ▼
Business Services (calculate → approve → snapshot)
    │
    ▼
[Module]Calculo → [Module]Aprobacion → [Module]HistoricoDetalle
```

The stored data is reused when:

-   an analyst triggers a recalculation (reads Local* tables);
-   an AI module is invoked (formula context + prior analysis);
-   a manager reviews historical approvals for audit or compliance.

### Main persisted entities

-   **AppUser / AppRole / AppUserRole:** users, roles, and their
    associations.
-   **MenuItem / RoleMenuPermission:** dynamic navigation and role-scoped
    access control.
-   **AIConfiguration:** per-module AI settings (system prompt,
    temperature, token limits), editable at runtime without redeployment.
-   **Formula / FormulaVariable:** reusable calculation formulas injected
    into AI prompts for context.
-   **PromptHistory:** immutable audit record of every AI API call.
-   **SyncTable / SyncLog:** sync configuration and execution history per
    ERP table.
-   **Local\* tables (13):** local copies of ERP view data — items,
    inventory, vendors, PO lines, orders, planning, etc.
-   **[Module]Calculo:** mutable current calculation state (per module).
-   **[Module]Aprobacion:** approval event (approver, timestamp).
-   **[Module]HistoricoDetalle:** immutable snapshot captured at approval
    time.
-   **ProduccionDiaria, OperacionSemanal, HerramientaConsumoDiario:**
    operational records with their own audit bitácora.

------------------------------------------------------------------------

## 8. Cross-cutting concepts

### Configuration and secrets

`appsettings.json` for non-secret settings; User Secrets in
development. In production, API key (`AI:ApiKey`), admin password, and
SMTP credentials must be supplied via environment variables or a secrets
manager — currently a known gap (see §11).

AI configuration (system prompts, temperature, token limits) is managed
in SQL Server via the Admin console — no redeployment required to tune
AI behavior.

Secrets are not stored in source code or Git.

### Authentication

Custom cookie-based authentication. No ASP.NET Identity. `PasswordHasher`
implements PBKDF2-SHA256 with a random salt and constant-time comparison.

As a result, the following are outside the system's scope:

-   Password reset flow
-   Two-factor authentication
-   External OAuth providers
-   User lockout

### Authorization

Role-Based Access Control (RBAC). Four non-deletable system roles:
`Admin`, `User`, `Ver`, `Almacén`. Menu permissions are role-scoped and
manageable via the Admin console (`RoleMenuPermission` table).
Controllers use `[Authorize]` and `[Authorize(Roles = "...")]`
attributes.

### ERP write protection

`ErpDbContext.SaveChanges()` and `SaveChangesAsync()` throw
`InvalidOperationException`. This is the primary defense against
accidental ERP data mutation — enforced in code, not policy.

### Observability

Business audit logs are persisted in SQL Server:

-   `SyncLog` — every ERP sync execution (table, row count, duration,
    status, error).
-   `PromptHistory` — every AI call (user, module, full prompts,
    response, tokens, duration, status).
-   `ProduccionDiariaBitacora` — every change to production daily
    records (who, what, when, previous value).
-   Approval records per analysis module — formal events with timestamps.

No dedicated metrics or APM solution; this is a scope decision for the
MVP.

------------------------------------------------------------------------

## 9. Architecture decisions

### Frontend: Razor + Bootstrap (server-side)

Chosen for data density and simplicity appropriate for a departmental
intranet tool. No SPA complexity needed; server-side rendering
minimizes client-side overhead for analyst-facing grid-heavy views.

### Backend: C# + ASP.NET Core MVC + .NET 8 LTS

Chosen for: structured data processing, EF Core ORM, AI orchestration
via HTTP, and in-process background service hosting. The two-project
layered architecture (`Web → LogicData`) keeps controllers thin and
business logic independently testable.

### Database: SQL Server

Shared infrastructure with the ERP; simplifies network topology. EF
Core Code-First migrations provide versioned, repeatable schema
evolution. `DataSeeder` bootstraps a fully functional portal from a
blank database.

### ERP: strictly read-only at code level

`ErpDbContext.SaveChanges()` throws by design. The portal queries 8
predefined SQL views — no write permissions to ERP tables are requested
or required.

``` text
SyncBackgroundService
     │
     ▼
ErpDbContext (SELECT only — SaveChanges throws)
     │
     ▼
8 ERP SQL Views
```

### Local sync copy instead of real-time ERP queries

All analytical queries run against local `Local*` tables, never against
the ERP in real time. This protects ERP performance and allows the
portal to operate during ERP maintenance windows, at the cost of a 4–24
hour data lag depending on the sync group.

### Calculate → Review → Approve → Snapshot

No analysis result enters the permanent historical record without
explicit human approval. The mutable `[Module]Calculo` table allows
free recalculation; approval generates an immutable
`[Module]HistoricoDetalle` snapshot with approver and timestamp.

### AI provider: multi-provider by design

`IAIService` decouples the application from any specific provider.

``` text
Business Service
     │
     ▼
IAIService
     │
     ▼
AIService (implementation)
     │
     ├── Claude API (Anthropic) — cloud, actively in use
     └── Ollama / Gemma 2     — local, original target, valid future path
```

Currently using `claude-haiku-4-5-20251001`. Switching back to a local
model requires only reimplementing `AIService` — no controller or
service changes.

------------------------------------------------------------------------

## 10. Quality requirements

### Security

ERP data is protected by a code-level write constraint. Portal access
requires cookie authentication with PBKDF2-SHA256 hashed passwords.
Role-based authorization gates every controller action and menu item.

### Traceability / Auditability

Every calculation, approval, and AI interaction is reconstructable:
`[Module]HistoricoDetalle` captures the exact data at approval time;
`PromptHistory` captures the complete AI prompt and response including
formula context.

### Maintainability

Repository + Service pattern isolates business logic from EF Core. All
analysis modules follow the same Calculate → Approve → History pattern,
making new module additions predictable. AI behavior and calculation
formulas are configurable without code changes.

### Data availability

Local `Local*` tables allow the portal to operate during ERP downtime,
within the sync interval (up to 24 h for master data groups).

### Simplicity

No distributed components, external queues, or additional caching
infrastructure beyond the in-process sync service. Additional
infrastructure is avoided unless a demonstrated need arises.

------------------------------------------------------------------------

## 11. Risks and technical debt

### No high availability or formal DR strategy

The system runs on a single on-premises server. Brief downtime is
tolerable for departmental use; longer outages require a manual recovery
runbook. SQL Server backup schedule and RPO/RTO targets are not yet
defined.

### Purchasing data sent to external Claude API

Every AI analysis call sends analyst questions and context (vendor
names, material codes, prices, quantities) to Anthropic's cloud API —
data leaves the corporate network. The `IAIService` interface allows
switching to a local model (Gemma 2 / Ollama) without upstream changes;
this was the original architecture target.

### Secrets in appsettings.json

API key, admin password, and SMTP credentials are currently stored in
`appsettings.json`. In production these must be moved to environment
variables or a secrets manager before the system is considered hardened.

### HTTPS enforcement not verified in application configuration

HTTPS termination is assumed at the IIS / reverse-proxy layer but not
enforced in `Program.cs`. Must be confirmed and enforced before
production.

### No CI/CD pipeline

Deployment is manual via `dotnet publish`. A basic pipeline
(build + test + publish) would reduce regression risk significantly.

### No automated test suite

With 34 controllers, 11 services, and 70+ domain entities, changes to core
services carry undetected regression risk. Unit tests for calculation
services (MaxMinService, BanderaService, WinderService) are the
highest-priority technical debt item.

### ERP view schema changes

The portal depends on 8 named SQL views. Changes to ERP view schema
require corresponding portal model updates. A change notification
process with the ERP team has not been established.

------------------------------------------------------------------------

## 12. Glossary

**ERP:** Enterprise Resource Planning — the corporate system that is the
authoritative source for items, inventory, purchase orders, and planning
data. Read-only from the portal's perspective.

**Max/Min:** Reorder point analysis — defines minimum and maximum stock
levels (reorder point, fixed order quantity) per item based on planning
codes and current inventory position.

**Bandera:** "Flag" — BOM explosion and shortage risk analysis.
Identifies materials at risk of causing a production stoppage based on
open sales orders and current stock.

**BOM:** Bill of Materials — list of components required to manufacture
a product.

**Winder:** A category of precision consumable manufacturing tools
managed separately due to high consumption rate and strategic supplier
relationships.

**Explosión de Herramienta:** "Tool Explosion" — decomposition of tool
assemblies into component parts for procurement analysis.

**High-Runner:** Items with consistently high consumption volume
requiring priority attention in procurement planning.

**OTD:** On-Time Delivery — module managing exception cases (late or
at-risk deliveries) discussed in operational review meetings.

**RFQ:** Request for Quotation — formal process of soliciting price
quotes from vendors.

**Local\* tables:** SQL Server tables in the PortalCompras database that
are copies of ERP data, updated via the synchronization engine.

**AppDbContext:** Primary EF Core context for all portal-owned tables;
manages schema via Code-First migrations.

**ErpDbContext:** Read-only EF Core context mapping ERP SQL views.
`SaveChanges()` throws by design.

**DataSeeder:** Startup class that applies migrations and seeds initial
roles, menus, admin user, AI configurations, formulas, and sync table
configuration.

**PromptHistory:** Immutable audit log of every AI API call — full
prompt, response, token counts, duration, and status.

**Calculate → Approve → History:** Standard three-phase workflow for all
analysis modules: tentative calculation (mutable), human approval,
immutable historical snapshot.

**PBKDF2:** Password-Based Key Derivation Function 2 — algorithm used
for password hashing, configured with SHA-256 and a random salt.

**RBAC:** Role-Based Access Control — permissions assigned to roles,
roles assigned to users.

**UPSERT:** Insert if not exists (by natural key), update if it does.
Used by the sync engine to keep local tables current without duplicates.

**ComprarSugerido:** "Suggested Buy Quantity" — output of a calculation
module recommending how much of an item to purchase.

------------------------------------------------------------------------

## 13. Defined development environment

-   Visual Studio 2022
-   C# / .NET 8 SDK
-   SQL Server (LocalDB or full instance)
-   Entity Framework Core 8 (Code-First migrations)
-   ClosedXML (Excel) / QuestPDF (PDF)
-   Git
-   Anthropic API (Claude Haiku) — actively used cloud AI provider
-   Ollama / Gemma 2 — supported local provider (original architecture
    target)
-   Claude Code CLI as a development support tool

------------------------------------------------------------------------

## Current status

**The system constitutes a complete MVP with the architecture described
in this document: ERP synchronization into a local SQL Server copy,
five analysis workflows with a formal Calculate → Approve → History
cycle, AI-assisted analysis via the Claude API, and role-based access
control. The out-of-scope points (§2–3) are firm decisions for this
maturity level, not outstanding work.**

### Operational risks to watch (do not block the MVP)

1.  No automated test suite — calculation engine regressions are
    currently detected manually.
2.  Secrets (API key, admin password) stored in `appsettings.json` —
    must be moved to env vars or a secrets manager before production
    hardening.
3.  HTTPS enforcement not verified in application configuration —
    assumed at IIS level but not confirmed.
4.  Purchasing data sent to Claude API (external) — evaluate local
    model (Gemma 2 / Ollama) as data-egress mitigation.
5.  No formal backup / DR strategy — SQL Server backup schedule and
    RPO/RTO targets are undefined.
6.  No CI/CD pipeline — manual deployments increase regression risk.
7.  ERP view schema change notification process with the ERP team not
    established.
