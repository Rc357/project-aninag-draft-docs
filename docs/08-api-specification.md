# API Specification

## Conventions

- **Style:** REST over HTTPS, JSON bodies. GraphQL was considered and rejected for MVP — the client set (a handful of purpose-built screens per role) doesn't have the heterogeneous query-shape problem GraphQL solves, and REST's simplicity/tooling maturity wins for a small team; revisit only if the Executive Dashboard's `[FUTURE]` cross-tenant analytics needs genuinely justify it.
- **Versioning:** URL path versioning, `/api/v1/...`. Chosen over header-based versioning for debuggability (visible in logs, curl-able without extra headers) — appropriate given the API has a small, controlled set of first-party clients, not a public third-party developer ecosystem yet.
- **Base URL shape:** `https://api.aninag.gov.ph/v1/...` (illustrative — actual domain pending product naming).
- **Auth:** Bearer JWT in `Authorization` header, issued by the Identity module. Tenant (`organization_id`) and role/scope claims are embedded in the token, never accepted as a request parameter (see [Security — Tenant Isolation](09-security.md#tenant-isolation)).
- **Pagination:** cursor-based (`?cursor=...&limit=...`), not offset — offset pagination degrades on large, frequently-mutated queues like the Barangay report queue.
- **Errors:** RFC 7807 Problem Details (`application/problem+json`) — `type`, `title`, `status`, `detail`, `instance`, plus an `errors` array for field-level validation failures. Standardizing on this now avoids each client team inventing its own error-shape parsing.
- **Idempotency:** all `POST` endpoints that create a resource accept an optional `Idempotency-Key` header, required for report submission specifically (FR-1) since mobile clients on flaky connections commonly retry submissions — without this, a retried request would create duplicate reports indistinguishable from genuine duplicate incidents.

## Authentication

| Endpoint | Method | Purpose |
|---|---|---|
| `/auth/guest` | POST | Issue a short-lived, unauthenticated-but-rate-limited token for Guest report submission (FR-1.1) |
| `/auth/google` | POST | Exchange a Google ID token for a platform session |
| `/auth/apple` | POST | Exchange an Apple identity token for a platform session |
| `/auth/email/request-otp` | POST | Send email OTP |
| `/auth/phone/request-otp` | POST | Send SMS OTP |
| `/auth/{provider}/verify-otp` | POST | Verify OTP, issue session |
| `/auth/refresh` | POST | Refresh an access token using a refresh token |
| `/auth/logout` | POST | Revoke current session |

## Reports (Incident Reporting module)

| Endpoint | Method | Actor | Purpose |
|---|---|---|---|
| `/reports` | POST | Guest, Verified Citizen | Submit a new report (FR-1). Multipart: metadata JSON + photo(s) |
| `/reports/{id}` | GET | Submitter, assigned staff/worker, admins in scope | Fetch full report detail, including current workflow state and audit trail |
| `/reports/{id}/confirm` | POST | Submitter | Confirm resolution (FR-8.2 → `Closed`) |
| `/reports/{id}/dispute` | POST | Submitter | Dispute resolution (FR-8.2 → reopened) |
| `/reports?mine=true` | GET | Verified Citizen | Own report history |
| `/reports/nearby` | GET | Any (public feed) | PII-redacted transparency feed, geo-bounded query |
| `/barangays/{id}/queue` | GET | Barangay Staff/Admin (scoped) | Incoming queue sorted by priority (FR-5.1) |
| `/reports/{id}/verify` | POST | Barangay Staff | Verify a report (FR-5.2 → `Verified`) |
| `/reports/{id}/reject` | POST | Barangay Staff | Reject with required reason (FR-5.2/5.3) |
| `/reports/{id}/merge` | POST | Barangay Staff | Merge as duplicate of another report (FR-3.2, FR-5.2) |
| `/reports/{id}/assign` | POST | Department Head/Staff | Assign to a Worker (FR-6.1) |
| `/reports/{id}/transition` | POST | Worker (assigned only) | Generic state transition through the Workflow Engine (FR-6.3): body specifies target state + required evidence refs (photo attachment IDs) |
| `/departments/{id}/workload` | GET | Department Head | Team workload view |

Note: `/reports/{id}/transition` is deliberately generic (targets *any* valid next state per the Workflow Engine's transition rules for the report's category) rather than one bespoke endpoint per lifecycle step (`/start`, `/upload-before`, `/complete`...). This mirrors the Workflow Engine's design ([Architecture](06-architecture.md#workflow-engine)): the API surface for state changes should not need to grow every time a new workflow state is configured for a tenant.

## Configuration (City Hall Administrator)

| Endpoint | Method | Purpose |
|---|---|---|
| `/admin/categories` | GET/POST/PATCH | Manage report categories (FR-11) |
| `/admin/departments` | GET/POST/PATCH | Manage departments |
| `/admin/barangays` | GET/POST/PATCH | Manage barangays, including boundary GeoJSON import |
| `/admin/workflow-definitions/{caseType}` | GET/PATCH | View/edit the active workflow definition for a case type (FR-14) |
| `/admin/notification-templates` | GET/PATCH | Edit notification copy (FR-9.3) |
| `/admin/users` | GET/POST/PATCH | User and role-assignment management within tenant |

## Dashboards / Reporting

| Endpoint | Method | Purpose |
|---|---|---|
| `/dashboards/city-hall/summary` | GET | Cross-department/barangay SLA + volume (FR-12.2) |
| `/dashboards/department/{id}/summary` | GET | Department-scoped performance |
| `/dashboards/barangay/{id}/summary` | GET | Barangay-scoped performance |

## Platform (Super Administrator)

| Endpoint | Method | Purpose |
|---|---|---|
| `/platform/organizations` | POST | Onboard a new tenant (FR-13.1) |
| `/platform/organizations/{id}/modules` | PATCH | Enable/disable modules/feature flags per tenant |
| `/platform/health` | GET | Cross-tenant platform health/usage |

## Sample: submit report

```http
POST /api/v1/reports
Authorization: Bearer <token-or-guest-token>
Idempotency-Key: 6c1f5e2a-...
Content-Type: multipart/form-data

{
  "categoryId": "b3f...",
  "description": "Large pothole blocking one lane near the market entrance.",
  "location": { "lat": 14.5995, "lng": 120.9842 },
  "photos": [<binary>]
}
```

```json
201 Created
{
  "id": "9a12...",
  "status": "PendingAIValidation",
  "categoryId": "b3f...",
  "createdAt": "2026-07-26T08:15:00Z",
  "trackingUrl": "https://app.aninag.gov.ph/reports/9a12..."
}
```

## Sample: error response (validation failure)

```json
422 Unprocessable Entity
Content-Type: application/problem+json
{
  "type": "https://api.aninag.gov.ph/errors/validation",
  "title": "One or more fields are invalid.",
  "status": 422,
  "errors": [
    { "field": "location", "message": "GPS location is required." },
    { "field": "photos", "message": "At least one photo is required." }
  ]
}
```

## What's intentionally deferred

- **Public developer API / API keys for third-party integration** — no external integrators exist at MVP stage; premature to design a public API contract before any partner has asked for one.
- **Webhooks for tenant-side integration** (e.g., LGU's existing systems subscribing to report events) — natural extension of the domain-event model in [Architecture](06-architecture.md#architecture-style) once a real integration partner requests it.
- **Bulk/batch endpoints** — not needed until an admin workflow (e.g., bulk category import for onboarding) proves manual/one-by-one is a real bottleneck.
