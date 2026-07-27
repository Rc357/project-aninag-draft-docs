# Non-Functional Requirements

Each NFR includes its MVP target and the rationale, since "scalable" and "secure" are meaningless without a measurable bar.

## Performance

| Requirement | Target | Rationale |
|---|---|---|
| Report submission API p95 latency | < 800ms (excluding photo upload) | Citizen app must feel responsive on low-end Android devices common in the target market |
| Photo upload (single image, 5MB) | < 5s on 4G | Photo is mandatory (FR-1.2); slow upload is the #1 abandonment risk |
| Dashboard queue load (Barangay, up to 500 open reports) | < 1.5s p95 | Staff usability under real operational load |
| AI validation turnaround (submission → validated/queued) | < 10s p95 | Must not block the citizen from seeing "Submitted" confirmation; validation runs async after acknowledgment |

## Scalability

- The system shall support horizontal scaling of the API tier independent of the database tier (see [Architecture](06-architecture.md#deployment-topology)).
- The system shall be designed so that onboarding an additional City tenant requires **zero additional infrastructure provisioning** for the shared services (API, workflow engine, AI pipeline) — only data rows, not new deployments, per tenant.
- Target design capacity (not MVP launch load, but architectural ceiling to validate against): 500 concurrent tenant cities, 50M citizens, sustained 10K report submissions/day platform-wide, without a data-model change. This number exists to stress-test schema and query design decisions now (see [Database Design](07-database-design.md)) — it is a design constraint, not a launch requirement.

## Availability

- Target **99.5% uptime** for MVP (single-region deployment) — appropriate for a pilot-stage government tool, not yet justifying multi-region active-active cost.
- Planned maintenance windows shall be communicated in-app at least 24 hours in advance.
- The citizen-facing report submission path shall degrade gracefully if the AI pipeline is unavailable: reports shall still be accepted and queued as `PendingAIValidation`, never blocked outright (validation is an enhancement to triage, not a gate on citizen access to government services — treating it otherwise would be a policy failure, not just a technical one).

## Multi-Tenancy & Data Isolation

- Every tenant-scoped table shall carry an `OrganizationId`, enforced at the data-access layer, not only in application code (see [Database Design — Tenant Scoping](07-database-design.md#tenant-scoping)).
- Cross-tenant data access shall be structurally prevented (query-layer enforcement, e.g., row-level security or mandatory repository-level filters), not merely relied upon via UI restrictions.
- A tenant's data export/deletion (offboarding) shall be achievable without affecting other tenants' data or availability.

## Security & Compliance

See full detail in [Security Architecture](09-security.md). NFR-level targets:

- All PII (names, phone numbers, addresses, GPS home-location inference) shall be encrypted at rest and in transit (TLS 1.2+, AES-256 at rest).
- The platform shall comply with the Philippine **Data Privacy Act of 2012 (RA 10173)**, including data subject access/erasure requests and breach notification procedures.
- Photo EXIF metadata shall be stripped of anything beyond what's needed (retain GPS only where the report design explicitly requires it; strip device identifiers).

## Accessibility

- Citizen mobile app shall meet **WCAG 2.1 AA**-equivalent mobile accessibility guidance: minimum touch target sizes, screen-reader labels, color-contrast ratios ≥ 4.5:1.
- This is a genuine target-market requirement, not a checkbox: a meaningful share of LGU constituents are older or have low digital literacy — inaccessible design directly reduces reporting volume, undermining the product's core value proposition.

## Localization

- UI text shall be externalized (no hardcoded strings) from MVP launch, supporting English and Filipino at minimum, even though only these two ship at launch — retrofitting i18n after launch is materially more expensive than building it in from the first screen.

## Offline Support

- The citizen app shall allow composing a report (photo, category, description) while offline and queue it locally for submission once connectivity resumes.
- GPS capture shall work offline (device-local), with submission deferred, not the coordinate.
- Full offline *browsing* of report history/status is **not** an MVP requirement — the primary offline need is capture at the point of observation (e.g., a citizen at a flood site with no signal), not offline review.

## Observability

- All API requests shall be traced with a correlation ID propagated through async AI processing, so a support engineer can reconstruct a single report's full lifecycle across services from one identifier.
- Azure Application Insights shall capture API latency, error rate, and AI pipeline success/failure rate per tenant, enabling per-tenant SLA reporting back to LGU customers — a commercial requirement, not just an ops one.

## Maintainability

- The codebase shall follow Clean Architecture boundaries (see [Architecture](06-architecture.md#architecture-style)) such that a new report category or a new workflow does not require changes to core domain logic, only configuration and, at most, new workflow-step handlers.
- Test coverage on domain/business-rule logic (workflow transitions, routing, RBAC scoping) shall be prioritized over UI test coverage, since these are the components most likely to cause cross-tenant or cross-city incidents if broken.

## Configurability (cross-cutting, restated as an NFR)

- Time-to-configure a new tenant's category list, department map, and SLA policy shall require **no engineering ticket** — this is treated as a measurable non-functional property of the platform (see [PRD success metrics](03-prd.md#success-metrics)), because "configurable" that still requires a developer is not actually configurable.
