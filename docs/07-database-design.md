# Database Design & ER Model

Database: **PostgreSQL 16 + PostGIS**. Naming: singular PascalCase entities in this doc, mapped to `snake_case` tables via a TypeORM `SnakeNamingStrategy`. All identifiers are UUIDv7 (time-ordered, index-friendly, and safe to generate client-side without a round trip — relevant for offline draft creation, see [NFR — Offline Support](05-non-functional-requirements.md#offline-support)).

## Tenant scoping

Every tenant-owned table includes `organization_id UUID NOT NULL REFERENCES organization(id)`, indexed as the leading column of its primary access-pattern index (e.g., `(organization_id, status, created_at)` on `report`). This is enforced two ways, not one:

1. **Application layer:** a shared NestJS interceptor/guard resolves the current tenant and a TypeORM subscriber/base-repository wrapper injects `WHERE organization_id = @currentTenant` on every query against a tenant-scoped entity (see [Architecture — Multi-Tenancy](06-architecture.md#multi-tenancy-strategy)).
2. **Database layer (defense in depth):** PostgreSQL Row-Level Security (RLS) policies on tenant-scoped tables, keyed to a session variable set at the start of each request's transaction. If the application layer's filter is ever bypassed by a bug, RLS is the second, independent barrier — this is the difference between "we tried to prevent cross-tenant leaks" and "cross-tenant leaks are structurally impossible short of a superuser bypassing RLS deliberately," which matters enormously for a government-data product.

`Organization`, `City`, and reference/seed tables not owned by a single tenant (e.g., a shared national Barangay boundary dataset used as fallback) are the only tables without `organization_id`.

## Core entity groups

### 1. Tenancy & Government Hierarchy

```mermaid
erDiagram
    ORGANIZATION ||--o{ CITY : has
    CITY ||--o{ BARANGAY : has
    CITY ||--o{ DEPARTMENT : has
    BARANGAY ||--o{ BARANGAY_BOUNDARY : "geo boundary"
    DEPARTMENT ||--o{ USER_ACCOUNT : employs

    ORGANIZATION {
        uuid id PK
        string name
        string tier "City | Province | National (future)"
        jsonb settings
        timestamptz created_at
    }
    CITY {
        uuid id PK
        uuid organization_id FK
        string name
        string psgc_code "PH Standard Geographic Code"
        jsonb branding_config
    }
    BARANGAY {
        uuid id PK
        uuid organization_id FK
        uuid city_id FK
        string name
        string psgc_code
    }
    BARANGAY_BOUNDARY {
        uuid id PK
        uuid barangay_id FK
        geometry polygon "PostGIS GEOGRAPHY(Polygon)"
    }
    DEPARTMENT {
        uuid id PK
        uuid organization_id FK
        uuid city_id FK
        string name
        string code
        boolean is_active
    }
```

**Design note — PSGC codes:** every `City` and `Barangay` stores the official Philippine Standard Geographic Code alongside its internal UUID. This is not decorative — it is the join key for any future integration with national datasets (PSA statistics, DILG reporting, disaster-response coordination with national agencies), and retrofitting it after tenants have organically-named barangays would require an error-prone reconciliation pass. Cheap to capture at onboarding, expensive to backfill.

### 2. Identity & Access

```mermaid
erDiagram
    USER_ACCOUNT ||--o{ USER_ROLE_ASSIGNMENT : has
    USER_ROLE_ASSIGNMENT }o--|| ROLE : references
    USER_ACCOUNT {
        uuid id PK
        uuid organization_id FK "nullable for Guest"
        string auth_provider "google|apple|email|phone|guest"
        string external_subject_id
        string display_name
        string phone_number_hash
        string email
        int reputation_score
        timestamptz created_at
    }
    ROLE {
        uuid id PK
        string code "GuestCitizen|VerifiedCitizen|BarangayStaff|BarangayAdmin|DeptStaff|DeptHead|CityHallAdmin|SuperAdmin"
        string display_name
    }
    USER_ROLE_ASSIGNMENT {
        uuid id PK
        uuid user_id FK
        uuid role_id FK
        uuid scope_barangay_id FK "nullable"
        uuid scope_department_id FK "nullable"
        uuid organization_id FK
    }
```

**Design note — scoped role assignments, not global roles:** `UserRoleAssignment` carries an optional `scope_barangay_id`/`scope_department_id` rather than roles being purely global-per-organization. This is what lets one user be, e.g., Barangay Staff for Barangay A only, while another is City Hall Administrator across the whole City — and it's what the RBAC engine in [Security](09-security.md#authorization-model) evaluates against. A flat "role per organization" model (rejected) would force one of two bad outcomes: over-broad access (any Barangay Staff sees all barangays) or a proliferation of organization-per-barangay tenants (defeats the point of the City tenant).

### 3. Workflow Engine (generic)

```mermaid
erDiagram
    WORKFLOW_DEFINITION ||--o{ WORKFLOW_STATE : defines
    WORKFLOW_DEFINITION ||--o{ WORKFLOW_TRANSITION : defines
    WORKFLOW_DEFINITION ||--o{ WORKFLOW_INSTANCE : instantiates
    WORKFLOW_INSTANCE ||--o{ WORKFLOW_EVENT : "audit trail"

    WORKFLOW_DEFINITION {
        uuid id PK
        uuid organization_id FK
        string case_type "IncidentReport|Permit|Appointment (future)"
        int version
        boolean is_active
    }
    WORKFLOW_STATE {
        uuid id PK
        uuid workflow_definition_id FK
        string code "Submitted|Verified|Assigned|..."
        boolean is_terminal
    }
    WORKFLOW_TRANSITION {
        uuid id PK
        uuid workflow_definition_id FK
        uuid from_state_id FK
        uuid to_state_id FK
        string required_role_code
        jsonb required_evidence "e.g. photo_before, photo_after"
        jsonb guard_rule "business-rule predicate reference"
    }
    WORKFLOW_INSTANCE {
        uuid id PK
        uuid organization_id FK
        uuid workflow_definition_id FK
        uuid current_state_id FK
        string case_type
        uuid subject_entity_id "FK to Report, Permit, etc."
        timestamptz created_at
        timestamptz updated_at
    }
    WORKFLOW_EVENT {
        uuid id PK
        uuid workflow_instance_id FK
        uuid actor_user_id FK "nullable for system/AI actor"
        string actor_type "human|ai|system"
        uuid from_state_id FK
        uuid to_state_id FK
        jsonb metadata
        timestamptz occurred_at
    }
```

`WORKFLOW_EVENT` **is** the audit log for every workflow-driven entity (FR-10) — a single, generic, append-only table rather than a separate audit table per module. This directly delivers FR-10.2 (no application-layer update/delete path is ever defined for this table) and means Permits, Appointments, and Inspections get full audit history for free the moment they're modeled as workflows, rather than each module needing to reinvent audit logging.

### 4. Incident Reporting (module-specific)

```mermaid
erDiagram
    REPORT ||--o{ REPORT_ATTACHMENT : has
    REPORT }o--|| REPORT_CATEGORY : categorized_as
    REPORT }o--o| REPORT : "duplicate_of (self-ref)"
    REPORT ||--|| WORKFLOW_INSTANCE : "backed by"
    REPORT_CATEGORY }o--|| DEPARTMENT : "default routes to"

    REPORT {
        uuid id PK
        uuid organization_id FK
        uuid workflow_instance_id FK
        uuid category_id FK
        uuid barangay_id FK
        uuid department_id FK "nullable until routed"
        uuid assigned_worker_id FK "nullable"
        uuid submitted_by_user_id FK "nullable for guest"
        string guest_contact_hash "nullable"
        geography location "PostGIS Point"
        text description
        uuid duplicate_of_report_id FK "nullable"
        float ai_validation_confidence
        string priority
        timestamptz sla_due_at
        timestamptz created_at
    }
    REPORT_CATEGORY {
        uuid id PK
        uuid organization_id FK
        string name
        string default_priority
        interval default_sla
        boolean requires_inspection
        boolean requires_emergency_escalation
        boolean is_active
    }
    REPORT_ATTACHMENT {
        uuid id PK
        uuid report_id FK
        string kind "citizen|before|after"
        string blob_url
        geography captured_at_location "nullable"
        timestamptz captured_at
    }
```

**Design note — `Report` references a `WorkflowInstance` (1:1) rather than embedding status directly:** the report's current status is always read through its `WorkflowInstance.current_state`, not a duplicated `status` column on `Report` itself. This avoids the classic bug class where a workflow-engine-driven status and a denormalized status column on the domain entity drift out of sync. The `Report` table stays focused on domain data (location, category, attachments); the Workflow Engine owns state and transitions exclusively.

**Design note — GPS as `geography`, not two floats:** using PostGIS `GEOGRAPHY(Point, 4326)` rather than separate `latitude`/`longitude` columns is what makes FR-3.1's radius-based duplicate search and FR-4.1's point-in-polygon barangay resolution simple, correct, and index-accelerated (`GIST` index) queries rather than manually implemented haversine math in application code — the latter is a common source of subtle correctness bugs (e.g., mishandling the antimeridian, or radius math that's wrong at scale) that PostGIS has already solved and battle-tested.

## Indexing strategy (MVP-critical)

| Index | Purpose |
|---|---|
| `report(organization_id, barangay_id, current_state, created_at)` (via workflow join or denormalized state cache — see below) | Barangay queue load (FR-5.1), the highest-frequency staff query |
| `report USING GIST(location)` | Duplicate radius search (FR-3.1), barangay point-in-polygon resolution (FR-4.1) |
| `barangay_boundary USING GIST(polygon)` | Same |
| `workflow_event(workflow_instance_id, occurred_at)` | Audit trail retrieval for a single case |
| `user_role_assignment(user_id, organization_id)` | Auth/permission resolution on every authenticated request — must be fast, it's on the hot path |

**Deliberate denormalization called out:** to avoid an expensive join to `WorkflowInstance`/`WorkflowState` on every dashboard query, `Report` maintains a denormalized `current_state_code` column, updated transactionally in the same write as the `WorkflowEvent` insert. This is a standard CQRS-adjacent read-optimization, not a violation of "status lives in the workflow engine" — the `WorkflowInstance` remains the source of truth; the denormalized column is a cache invalidated within the same transaction, never written independently.

## Data retention & deletion

- `WORKFLOW_EVENT` (audit log) is retained indefinitely by default — it is the platform's accountability record and a likely subject of future transparency/FOI requirements.
- Citizen accounts support data-subject erasure requests (RA 10173 compliance, see [Security](09-security.md#compliance)): PII fields on `USER_ACCOUNT` are nulled/anonymized on request, but `WORKFLOW_EVENT` and `REPORT` rows are retained with the actor reference pointing to an anonymized placeholder, preserving audit integrity without retaining erasable PII.
