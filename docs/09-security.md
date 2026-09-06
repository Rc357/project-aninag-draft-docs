# Security Architecture

Government-adjacent products carry a different trust bar than typical consumer SaaS: a breach doesn't just damage a company, it damages public trust in digital government services broadly. Security decisions below are made with that asymmetry in mind, not just conventional best practice.

## Authentication

- Supported identity providers at MVP: Guest (unauthenticated, rate-limited token), Google, Apple, Email + OTP, Phone + OTP, Facebook (per FR-16.1). Government Single Sign-On (e.g., a future PhilSys/national ID integration) is reserved as a `[FUTURE]` provider — the auth abstraction (see below) is provider-pluggable specifically so this doesn't require an auth-model rewrite later.
- Facebook Login specifically requires Meta's App Review for most permissions beyond basic profile — an external approval dependency with its own timeline, not something the auth abstraction's pluggability can shortcut.
- A user's `username` (public-facing) and `display_name` (real name, PII) are never the same field — see [Database Design's design note on this](07-database-design.md#2-identity--access) — which is what keeps the anonymity-by-default policy (FR-16.2) a schema guarantee rather than a per-screen convention.
- All non-guest sessions issue a short-lived JWT access token (target: 15 minutes) plus a longer-lived, rotating refresh token stored server-side (revocable) — short access-token life bounds the damage window of a leaked token; server-side-revocable refresh tokens mean a compromised device's access can be cut off immediately, which a pure-JWT-everywhere design cannot do.
- Guest tokens are scoped narrowly: they can create a report and check its status by tracking ID, nothing else. They are never granted a role claim.
- Phone numbers and emails are stored hashed for lookup/dedup purposes with the plaintext encrypted separately (not both stored in the clear) — this limits blast radius if the lookup index were ever exposed without the encryption keys.

## Authorization model

**Role-Based Access Control, scoped hierarchically** (Organization → City → Barangay/Department), per the `USER_ROLE_ASSIGNMENT` model in [Database Design](07-database-design.md#2-identity--access).

Every authorization check resolves as: *does this user hold a role, at or above the required level, whose scope covers the resource's Barangay/Department/City?* Concretely:

- A **Worker** can only act on `Report`s where `assigned_worker_id = self` (FR-6.2) — enforced as a repository-level filter, not merely a UI restriction, so a crafted API request cannot bypass it.
- A **Barangay Staff/Admin**'s effective scope is the set of `scope_barangay_id`s on their role assignments — cross-barangay queries return nothing outside that set, even for a City-tenant-authenticated user.
- A **City Hall Administrator** has scope over the entire City tenant (all its barangays/departments) but never another City's data, because City itself is a data attribute (of `organization_id`, indirectly), and every query is `organization_id`-filtered first (see [Tenant Isolation](#tenant-isolation) below), role scope second.
- **Super Administrator** is the only role with cross-tenant reach, and every action it takes is audit-logged with elevated scrutiny (see [Audit & Monitoring](#audit--monitoring)) — this role is the platform's single highest-value target and is treated as such (mandatory MFA, no long-lived sessions, alerting on any use).

This two-layer check (tenant scope, then hierarchical role scope) is implemented as reusable authorization middleware/handlers applied per-endpoint, not re-implemented ad hoc per controller — a re-implemented-per-endpoint authorization check is exactly how the OWASP "broken access control" class of vulnerability tends to enter a codebase, and this platform's core value proposition (trustworthy multi-tenant government data separation) cannot afford that risk.

## Tenant isolation

- `organization_id` for the current session is resolved **exclusively from the authenticated JWT's claims**, never from a client-supplied request parameter, path segment, or body field. An endpoint that accepted `organizationId` as client input would allow any authenticated user to attempt cross-tenant access by simply changing that value — this class of bug is closed structurally by never trusting the client for this value at all.
- Enforced redundantly at the database layer via PostgreSQL Row-Level Security in addition to the application-layer NestJS/TypeORM tenant-scoping (see [Database Design — Tenant Scoping](07-database-design.md#tenant-scoping)) — defense in depth against an application-layer bug.
- Tenant-boundary tests (attempting cross-tenant reads/writes with a valid token for a different tenant) are treated as required security regression tests, not optional QA, given how central this guarantee is to the product's trustworthiness.

## Data protection

- TLS 1.2+ enforced in transit; AES-256 at rest for the database and Blob Storage (Azure-managed encryption keys at MVP; customer-managed keys considered if/when a specific enterprise tenant requires it).
- PII fields (name, phone, email, precise home-adjacent GPS where inferable) are treated as a distinct data class from operational data (report category, generic location, timestamps) in the schema and in access logging — this separation is what makes RA 10173 data-subject requests (below) tractable without touching operational/audit data integrity.
- Photo EXIF is sanitized on upload: GPS metadata is stripped from the stored file (the report's location is captured explicitly and separately, per [Database Design](07-database-design.md#4-incident-reporting-module-specific)); device/camera identifiers are stripped. Uploaded photos are never trusted as a source of location truth precisely because EXIF GPS can be absent, wrong, or spoofed — the app's own GPS capture at submission time is authoritative.

## Preventing spam & abuse

Layered, not single-mechanism, per FR-2/NFR — no single control is assumed sufficient on its own:

| Control | Defends against |
|---|---|
| Photo + GPS required (FR-1.2/1.3) | Trivial junk/text-only spam |
| Rate limiting (per device fingerprint + per account, Redis-backed) | Flooding attacks, scripted mass submission |
| AI spam/duplicate-image-hash detection | Repeated/recycled images, obviously irrelevant photos |
| AI duplicate detection (geospatial + visual) | Report-count inflation, whether malicious or well-intentioned over-reporting |
| Citizen reputation score | Cumulative signal across a user's history — a single false report is normal citizen behavior, a pattern is not |
| Community reactions (FR-17: support/dispute) | Crowd-sourced validation signal for ambiguous cases — formerly listed here as a future item, now specified. A `dispute` reaction requires a written reason (FR-17.3) specifically to raise the cost of coordinated dispute-brigading against a legitimate report |
| Manual verification (Barangay Staff, FR-5.2) | Final human backstop — AI and heuristics inform priority/triage, a human always retains override authority (FR-2.5) |

**Deliberate design stance:** no automated control (spam score, AI confidence, reputation) is ever allowed to *permanently* reject a citizen report without a human-reviewable path — see FR-2.5. For a government service, silently and permanently denying a citizen's ability to report an issue based on an opaque automated score is a legitimacy risk, not just a UX rough edge.

## Compliance

- **RA 10173 (Data Privacy Act of 2012):** the platform implements data-subject access and erasure requests (see [Database Design — Data Retention](07-database-design.md#data-retention--deletion)), a designated Data Protection Officer contact per deployment, and breach notification procedures (72-hour NPC notification target, mirroring GDPR-equivalent practice since RA 10173's implementing rules are closely aligned).
- **Data residency:** Azure region selection defaults to a Southeast-Asia region; a Philippines-resident-data requirement from a specific government customer would be satisfied by Azure region choice, not an architecture change.
- **Government procurement security requirements** (e.g., DICT/NPC guidelines for government digital services) are tracked as an evolving compliance checklist against this document as specific LGU/national partnerships develop — not fully enumerable at MVP stage without a named customer's specific requirements.

## Audit & monitoring

- Every workflow state transition is audit-logged immutably (FR-10, [Database Design — Workflow Engine](07-database-design.md#3-workflow-engine-generic)).
- Every authentication event and every Super Administrator action is logged with elevated retention and alerting — given this role's cross-tenant reach, anomalous use (login from a new location, off-hours cross-tenant queries) should page a human, not just log silently.
- Azure Application Insights captures per-tenant error rates and auth failure spikes, feeding basic intrusion-detection alerting (e.g., repeated auth failures against one account, unusual cross-barangay query patterns).

## Explicit non-goals for MVP (revisit when justified)

- Full SOC 2 / ISO 27001 certification — appropriate once the platform has paying enterprise/government contracts that require it as a procurement gate, premature as a day-one engineering target for a pre-pilot product.
- Customer-managed encryption keys (BYOK) — reserved for a specific enterprise/national tenant requirement, not built speculatively.
- Penetration testing — scheduled ahead of the first production pilot going live with real citizen data, not before there's a system to test.
