# Vision & Product Strategy

## Core philosophy

We are not building a reporting app. We are building **the operating system for local governments**, and Citizen Incident Reporting is Module #1 — the first proof point, not the product ceiling.

This distinction has concrete design consequences, not just marketing value:

| If we were building "a reporting app" | Because we are building "an LGU OS" |
|---|---|
| Hardcode report categories, statuses, and routing rules | Every category, status, workflow, and routing rule is data, configured per tenant |
| One table per feature, tightly coupled to "incidents" | A generic **Workflow Engine** where "incident report," "permit application," "appointment," and "inspection" are all instances of the same underlying case/workflow abstraction |
| Auth model scoped to "citizen vs staff" | A hierarchical RBAC model scoped to Organization → City → Barangay → Department, ready for Provincial/National tiers |
| Single-tenant deployment per city (classic gov-contractor model) | Single deployment, many tenants, isolated data, shared codebase |

Every architectural decision in this documentation set is evaluated against one question: **"Does this decision survive the platform going from 1 city to 500 cities and from 1 module to 15 modules?"** If not, it gets redesigned now, while the cost of doing so is a schema and a service, not a migration of live government data.

## Vision statement

> Within 5 years, Project Obserba is the default digital backbone that Philippine LGUs use to manage every citizen-facing government process — from a pothole report to a business permit — with citizens able to track any request in one app and LGUs able to run their operations from one console.

## Target users

The platform models six operational actor tiers today, with two more reserved for future government levels:

| Tier | Role | MVP relevance |
|---|---|---|
| 1 | Guest Citizen | Can submit reports without an account (lower friction, higher volume, higher spam risk — mitigated per [Security](09-security.md)) |
| 2 | Verified Citizen | Authenticated via Google/Apple/Email/Phone OTP; gets report history, notifications, reputation score |
| 3 | Barangay Staff | Verifies incoming reports, assigns workers |
| 4 | Barangay Administrator | Manages Barangay Staff, sees barangay-level dashboard |
| 5 | Department Staff (Worker) | Executes assigned jobs (roads, sanitation, utilities, etc.) |
| 6 | Department Head | Manages department staff and workload, department-level SLA visibility |
| 7 | City Hall Administrator | Cross-department, cross-barangay visibility and configuration authority for the city tenant |
| 8 | Super Administrator | Platform operator — onboards new city tenants, manages global configuration and platform health |
| — (future) | Provincial Administrator | Aggregate visibility across cities/municipalities in a province |
| — (future) | National Administrator | Aggregate visibility across provinces — policy and reporting, not operational management |

This tier list is the basis for the RBAC model in [Security Architecture](09-security.md) and is designed so Provincial/National roles slot in as **read-and-policy** tiers above City, without requiring changes to how City-level tenants operate.

## Multi-level government structure

```
Organization
  └─ City / Municipality
       └─ Barangay
            └─ Department
                 └─ Worker
```

- **Organization** is the tenant boundary (see [Architecture — Multi-Tenancy](06-architecture.md#multi-tenancy-strategy)). In the MVP, one Organization typically corresponds to one City LGU, but the model does not assume a 1:1 mapping — a provincial rollout may create one Organization spanning multiple City records.
- **City** owns city-wide configuration: report categories, department list, SLA policy, branding.
- **Barangay** is the first-line responder tier for most citizen reports — it owns intake queues and initial verification.
- **Department** is the execution tier (Public Works, Sanitation, Utilities, etc.) — it owns worker assignment and job completion.
- **Worker** is an individual staff account scoped to assigned jobs only.

Every permission check in the system resolves against this hierarchy: a Barangay Administrator's authority is implicitly scoped to their Barangay's data; a Department Head's to their Department; a City Hall Administrator's to their City. This is enforced structurally (see [Database Design](07-database-design.md#tenant-scoping)), not just in application logic, so a bug in one screen cannot leak another barangay's data.

## Product philosophy — configurability over hardcoding

The founding brief is explicit: **"Nothing should require source code changes"** for anything that may differ between cities. Concretely, these are treated as *tenant configuration data*, not code:

- Departments and their names/icons/routing
- Report categories and subcategories
- Priority levels and SLA targets per category
- Workflow definitions and status vocabularies
- Roles and permissions (within the platform's role *shape*, not unlimited custom RBAC in v1 — see trade-off note below)
- Notification templates and channels
- Business rules (auto-escalation thresholds, auto-close timers, spam thresholds)

**Trade-off called out deliberately:** fully generic, tenant-authorable RBAC (where a city can invent entirely new role types with custom permission sets) is *not* in MVP scope — it is high engineering cost for a capability early customers won't ask for. MVP ships a fixed set of role tiers (above) with **configurable scoping**, not configurable role *shapes*. This is revisited in [Roadmap](11-roadmap.md) once 3+ paying tenants surface real demand for custom roles — building it earlier would be speculative complexity.

## Future modules (beyond MVP)

All of the following are designed to run on the same Workflow Engine and multi-tenant data model established by the MVP, requiring new configuration and UI, not new architecture:

Permit Processing · Barangay Clearance · Business Permit · Building Permit · Disaster Response · Emergency Management · Appointment Scheduling · Public Announcements · Asset Management · Public Works · Community Events · Inspections · Payments · Executive Dashboard · AI Assistant · Analytics

See [Architecture — Workflow Engine](06-architecture.md#workflow-engine) for how a new module is expected to be added without touching core platform code.

## Expansion beyond LGU

Although the first customer is a single city, the Organization/City/Barangay model is designed to also represent:

- Multiple municipalities within a province (Provincial tenant aggregates multiple City tenants)
- National government agencies that need cross-LGU visibility (read/policy layer, not operational)

This is a design constraint carried through [Database Design](07-database-design.md) and [Security](09-security.md), not a feature to be built later — it costs nothing to model correctly now and is expensive to retrofit onto live tenant data later.
