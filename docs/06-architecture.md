# System Architecture

## Architecture style

**Modular Monolith, feature-first, Clean Architecture, with CQRS where read/write shapes genuinely diverge.**

### Why not microservices from day one

The founding brief asks us to "not think small" and design for nationwide scale — but nationwide *data* scale and nationwide *organizational/deployment* scale are different problems, and only the first is real on day one. A single-city MVP has one team, one release cadence, and no independent-scaling requirement between "report submission" and "dashboard reads." Microservices at this stage would buy:

- Network-call overhead and distributed-transaction complexity for workflows that are currently a handful of joined tables
- Multiple deployment pipelines, service meshes, and inter-service auth to maintain with a small team
- Premature service boundaries that are likely wrong until real usage patterns (which module gets hot, which needs independent scaling) are observed

...while paying none of the benefits microservices exist for (independent team ownership, independent scaling, independent deploy cadence) because there is one team and one deployable at this stage.

**The counter-risk — the one worth taking seriously — is the classic "monolith calcifies and can never be split."** This is mitigated structurally, not by promise:

- **Feature-first folder structure**: code is organized by module (`IncidentReporting/`, `Identity/`, `Workflow/`, future `Permits/`), not by technical layer (`Controllers/`, `Services/`, `Repositories/`) spanning all modules. Each module owns its own domain models, application logic, and data access.
- **No cross-module direct database access.** A module talks to another module's data only through its public application-service interface (in-process today, a network call tomorrow) — never a direct ORM/SQL join across module schemas.
- **Modules communicate via an in-process domain event bus** (e.g., a report being `Verified` publishes an event; the Notification module subscribes) — the same pattern a message broker would use, just without the network hop. This means extracting a module into its own service later is a matter of moving the event transport from in-process to a broker (e.g., Azure Service Bus), not a rewrite of business logic.

This gives us a **path to microservices**, chosen deliberately — extract a module when it has an independent scaling need or an independent team, not before. Candidates in priority order if/when extraction is warranted: the **AI pipeline** (different scaling profile — bursty, GPU/inference-bound — from the CRUD API) and **Notifications** (fan-out, retry-heavy, benefits from decoupled queueing).

### Clean Architecture layering (per module)

```
┌─────────────────────────────────────────┐
│  Presentation (API Controllers / DTOs)   │
├─────────────────────────────────────────┤
│  Application (Commands, Queries,         │
│               Handlers — CQRS)           │
├─────────────────────────────────────────┤
│  Domain (Entities, Value Objects,        │
│          Domain Events, Business Rules)  │
├─────────────────────────────────────────┤
│  Infrastructure (TypeORM, Repositories,  │
│    Azure Blob, Azure OpenAI clients,     │
│    FCM, Redis)                           │
└─────────────────────────────────────────┘
```

Dependency rule: inner layers never reference outer layers. Domain has zero dependency on NestJS decorators or TypeORM — this is what makes the domain (workflow rules, RBAC scoping rules, routing rules) unit-testable without a database or an HTTP framework, and portable if the persistence technology ever changes.

**CQRS is applied selectively, not dogmatically:** write-side operations (submit report, verify, assign, transition state) go through Command handlers enforcing invariants; read-side dashboards (which have very different shape needs — aggregated counts, filters, joins across categories/barangays) go through Query handlers that can read from denormalized/projected views without forcing the write-model to compromise its integrity for read convenience. We do **not** stand up a separate read database/event-sourcing pipeline for MVP — that is justified once dashboard query load or reporting complexity actually demands it, not preemptively.

## Multi-tenancy strategy

**Shared database, shared schema, discriminator column (`OrganizationId`) — not database-per-tenant, not schema-per-tenant.**

| Option | Verdict | Why |
|---|---|---|
| Database-per-tenant | Rejected for MVP | Operationally expensive at hundreds-of-cities scale (migrations run N times, connection pool exhaustion, backup/restore complexity); the isolation benefit is real but achievable more cheaply below |
| Schema-per-tenant | Rejected | Same migration-fanout problem as above, with less mainstream ORM/tooling support than row-level discriminator; complicates cross-tenant platform analytics (Super Admin views) |
| **Shared schema + `OrganizationId` discriminator (chosen)** | **Adopted** | One migration path, one connection pool, trivial platform-level analytics; isolation enforced via mandatory tenant-scoping middleware (a NestJS interceptor paired with a TypeORM query subscriber) so every query is tenant-scoped by construction, not by developer discipline |

**This is revisited, not permanent:** if a specific enterprise customer (e.g., a National Government agency) contractually requires physical data isolation, the architecture supports a hybrid — that tenant's `OrganizationId` is deployed to a dedicated database using the same schema and codebase, via a tenant-to-connection-string routing layer. This is a configuration/ops decision at that point, not a rewrite, because the schema and query filters are already tenant-shaped.

Enforcement mechanism (see [Database Design — Tenant Scoping](07-database-design.md#tenant-scoping) for the schema-level detail):
- A shared NestJS interceptor/guard resolves `currentTenant` once per request and a TypeORM subscriber/query-builder wrapper applies `OrganizationId = @currentTenant` to every tenant-scoped entity's queries automatically — a developer would have to explicitly opt out (bypass the shared repository base class) to skip it, which is a deliberate, reviewable action, not an accidental omission.
- `currentTenant` is resolved once per request from the authenticated principal's claims, not from client-supplied input, closing the obvious "pass a different OrganizationId in the request body" attack (see [Security — Tenant Isolation](09-security.md#tenant-isolation)).

## Workflow Engine

The Incident Reporting lifecycle (FR-5 through FR-8) is **not** modeled as bespoke `IncidentReport` state-machine code. It is a configured instance of a generic engine so that Permits, Appointments, Inspections, and Business Licensing (future modules) are new configuration, not new state-machine implementations.

**Core abstractions:**

- `WorkflowDefinition` — tenant-scoped, versioned definition of states, transitions, and rules for a case type (e.g., "Incident Report v1").
- `WorkflowInstance` — a running case (an actual incident report, permit application, etc.) bound to a `WorkflowDefinition` version, current state, and domain payload.
- `Transition` — from-state → to-state, guarded by: required role(s), required evidence (e.g., "photo attached"), and optional business-rule predicate (e.g., "SLA breach triggers auto-escalation").
- `WorkflowEvent` — every transition emits a domain event (`WorkflowInstanceTransitioned`) that other modules (Notifications, Audit Log, AI reputation scoring) subscribe to, decoupling "the incident workflow" from "what happens as a side effect of its transitions."

**Why version the definition, not just the instance:** an LGU will change its process (e.g., add a new inspection step) after cases are already in flight. Existing `WorkflowInstance`s stay bound to the version they started under; new instances pick up the latest version. Without this, changing a workflow definition would corrupt in-flight cases — a subtle but real production incident class in workflow-engine products that don't version definitions.

**What we are deliberately NOT building for MVP:** a visual, tenant-facing workflow designer UI (drag-and-drop state machine builder). MVP workflows are configured via structured admin forms/JSON against the same engine — the engine's data model supports a visual designer later without migration, but building the designer UI now is speculative effort against a capability only a platform admin (us), not a City Hall Administrator, needs at MVP scale (one workflow: Incident Reporting).

## Technology stack and rationale

| Layer | Choice | Why |
|---|---|---|
| Citizen/Worker mobile | **Flutter** | Single codebase for iOS/Android, strong offline-local-storage story (FR-1.7, NFR offline support), good camera/GPS plugin maturity |
| Admin/Dashboard web | **React + Vite** | Deepest ecosystem for table/filter/chart-heavy internal dashboards (TanStack Table, AG Grid, Ant Design/MUI for fast, decent-looking screens without building components from scratch); matches the team's existing React experience. Talks directly to the NestJS REST API — no separate web backend/BFF (Express or otherwise), since these are authenticated internal tools with no SSR/SEO requirement that would justify one. |
| Backend API | **NestJS (Node.js/TypeScript)** | Decisive factor is team fit: both engineers are already fluent in Node/TypeScript (React, Express, NestJS) and not in .NET — for a two-person team, that fluency outweighs a marginal framework-capability difference. Architecturally it's a like-for-like swap, not a design change: NestJS's module/DI/decorator model and official `@nestjs/cqrs` package map directly onto the same Clean Architecture + CQRS pattern this document already specifies, and Nest guards implement the same hierarchical RBAC checks (see [Security — Authorization](09-security.md#authorization-model)). One backend serves both the Flutter mobile apps and the React dashboard — no second backend framework anywhere in the stack. |
| Primary datastore | **PostgreSQL + PostGIS** | PostGIS is the deciding factor: GPS routing (FR-4) and duplicate-radius detection (FR-3.1) are genuinely geospatial queries (point-in-polygon for barangay boundaries, radius search for duplicates) — reimplementing this in application code on a non-spatial database is slower and more bug-prone than using PostGIS's indexed geospatial operators. TypeORM has first-class NestJS integration (`@nestjs/typeorm`) and supports PostgreSQL geometry/geography columns directly; the handful of genuinely geospatial queries (radius search, point-in-polygon) use TypeORM's raw-query/query-builder escape hatch to call PostGIS operators directly — normal practice for spatial queries regardless of ORM, not a Node-specific workaround. |
| Cache / rate-limiting | **Redis** | Session/token cache, rate-limiting counters (FR-2.3 spam prevention), pub/sub backing for domain events at higher scale |
| File storage | **Azure Blob Storage** | Before/after photos, report evidence — cheap, durable, CDN-fronted for citizen transparency feed images |
| AI | **Azure OpenAI + Azure AI Vision** | See [AI Design](10-ai-design.md) for full rationale — regional availability and enterprise data-handling terms matter for a government-adjacent product |
| Push notifications | **Firebase Cloud Messaging** | Cross-platform (iOS/Android/Web) push, no viable first-party Azure equivalent at comparable maturity |
| Hosting | **Azure App Services** | Managed, scales horizontally, avoids Kubernetes operational overhead the team doesn't need at this stage — revisit as a container platform (Azure Container Apps/AKS) only if App Services' scaling/cost model becomes a genuine constraint |
| Observability | **Azure Application Insights** | Mature Node.js SDK (`applicationinsights` npm package) with the same per-tenant custom-dimensions support needed for the per-tenant SLA reporting NFR — no capability lost moving off .NET |
| CI/CD | **GitHub Actions** | Sufficient for current team size; standard deploy-to-Azure actions exist |
| Containerization | **Docker** | Local dev parity and a clean on-ramp to Container Apps/AKS later without re-deriving the runtime environment |

## Deployment topology (MVP)

```
                         ┌───────────────────────┐
 Citizen / Worker  ───▶  │   Azure App Service    │
 Flutter apps            │     (NestJS API)       │──────┐
                         └───────────────────────┘       │
                                    │                     ▼
 City Hall / Barangay /            │            ┌──────────────────┐
 Department Dashboards ───────────▶│            │  Azure OpenAI /   │
 (React + Vite)                    │            │  Azure AI Vision  │
                                    ▼            └──────────────────┘
                         ┌───────────────────────┐
                         │  PostgreSQL + PostGIS  │
                         │  (Azure Database for   │
                         │   PostgreSQL, HA tier) │
                         └───────────────────────┘
                                    │
                         ┌───────────────────────┐
                         │  Redis (Azure Cache)   │
                         └───────────────────────┘
                                    │
                         ┌───────────────────────┐
                         │  Azure Blob Storage     │
                         │  (photos/evidence)      │
                         └───────────────────────┘
```

The API tier is stateless (session state in Redis, not in-process) so it scales horizontally behind the App Service's built-in load balancing without sticky sessions — a prerequisite for the NFR scalability target, and free to get right now versus expensive to retrofit once session state has leaked into in-process assumptions.

## Cross-cutting: how a future module gets added

To make the "modular, not a reporting app" thesis concrete, adding **Business Permit Processing** later looks like:

1. Define a new `WorkflowDefinition` (states: Submitted → DocumentReview → FeeAssessment → Approved/Rejected → Issued) — configuration, no core engine change.
2. Add a `Permits` feature module (own Application/Domain/Infrastructure per Clean Architecture) implementing permit-specific domain rules (fee calculation, document requirement rules) — new code, but isolated to its module folder, not touching Incident Reporting.
3. Reuse, unmodified: Identity/RBAC, Workflow Engine core, Notification module, Audit Log, multi-tenant data scoping, Azure Blob evidence storage.
4. Add permit-specific screens to the citizen app and City Hall dashboard.

No change to the tenant data model, no change to auth, no change to the deployment topology. This is the concrete test of whether the "OS for LGUs" thesis in [Vision & Strategy](02-vision-and-strategy.md) actually holds — if adding a module required touching Identity, tenancy, or the workflow engine core, that would be a signal the abstraction was wrong.
