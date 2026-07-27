# Product Requirements Document (PRD) — MVP: Citizen Incident Reporting

## MVP scope statement

The MVP delivers **one workflow end to end, for one city, configured (not hardcoded)**: a citizen reports an incident; it is validated, routed, verified, worked, and closed, with full status visibility for the citizen and full operational visibility for LGU staff. Everything else in the founding brief (permits, disaster response, payments, etc.) is explicitly **out of scope** for MVP and exists only to constrain the architecture (see [Vision & Strategy](02-vision-and-strategy.md)).

## Report categories (MVP)

Citizens can report:

Broken roads · Flooding · Garbage · Illegal dumping · Fallen trees · Broken street lights · Electric wires · Water leaks · Drainage · Traffic incidents · Public facility damage · Crime reports · Fire · Other community issues

These ship as **seed configuration data**, not enum constants — a City Hall Administrator can add, rename, or deactivate categories without a deployment (see [Functional Requirements — Category Management](04-functional-requirements.md#category-management)). "Crime reports" and "Fire" are flagged in seed data as `requiresEmergencyEscalation = true`, routing them differently (see below) rather than through the standard barangay queue — treating a fire report identically to a pothole report would be a product and safety failure.

## Personas

| Persona | Goal | Primary pain today |
|---|---|---|
| **Rosa, Verified Citizen** | Report a flooded street near her home and know someone will act | No visibility after posting on the barangay's Facebook page; reports get lost |
| **Guest Citizen (unregistered)** | Report a hazard quickly without creating an account | Friction of signup stops people from reporting at all |
| **Jun, Barangay Staff** | Triage incoming reports and assign them to the right worker fast | Reports arrive via chat/paper with no structure, duplicates waste time |
| **Ate Baby, Department Head (Public Works)** | See workload across her team and prove SLA performance | No structured data — she manually tracks completion in a spreadsheet |
| **Mayor's Office / City Hall Admin** | See city-wide operational performance and constituent sentiment | No dashboard; performance claims are anecdotal |

## End-to-end MVP workflow

```
Citizen submits report (photo + GPS required)
   ↓
AI Validation (is this a real, categorizable incident?)
   ↓
Duplicate Detection (is this already reported nearby?)
   ↓
GPS Routing (which Barangay/Department owns this location + category?)
   ↓
Barangay Queue (pending verification)
   ↓
Barangay Verification (staff confirms legitimacy, may reject/merge)
   ↓
Assign Worker (Department Staff)
   ↓
Worker Starts Job
   ↓
Upload Before Photos
   ↓
Complete Work
   ↓
Upload After Photos
   ↓
Inspection (optional, configurable per category)
   ↓
Citizen Confirmation ("Yes, this is resolved" / "No, reopen")
   ↓
Closed
```

Every transition is an audit-logged event (actor, timestamp, from-status, to-status, reason) — see [Database Design — Audit Log](07-database-design.md#audit-log). This is a hard requirement, not a nice-to-have: LGU accountability and any future FOI/transparency reporting depend on an unforgeable history.

## MVP feature list

### Citizen-facing (mobile app)
- Submit report: category, description, photo (required), GPS location (required, with map pin adjustment)
- Guest submission (no account) and authenticated submission (Google, Apple, Email, Phone OTP)
- Track report status in real time (push notification on each status change)
- View nearby reports (public transparency feed, PII-redacted)
- Confirm resolution or reopen a report
- View personal report history and reputation indicator (verified accounts only)

### Barangay Dashboard
- Incoming queue sorted by priority/SLA risk
- Report detail view: photos, GPS pin, AI classification, duplicate cluster (if any)
- Verify / reject / merge-as-duplicate
- Assign to department/worker

### Department Dashboard
- Team workload view
- Assign/reassign jobs among department workers
- SLA and completion-rate reporting for the department

### Worker App
- List of assigned jobs only (no visibility into other workers' queues)
- Navigate to location
- Start job → upload before photos → complete → upload after photos

### City Hall Dashboard
- Cross-department, cross-barangay report volume and SLA compliance
- Category configuration (add/edit/deactivate report categories, priorities, SLAs)
- Department and Barangay configuration
- User and role management within the city tenant

### Platform (Super Admin)
- Onboard a new City/Organization tenant
- Global platform health and usage metrics
- Feature-flag / module enablement per tenant

## Explicit non-goals for MVP

- No permit/licensing workflows (future module, same Workflow Engine)
- No payments processing
- No offline-first data sync for the citizen app (basic offline draft queueing only — see [Non-Functional Requirements](05-non-functional-requirements.md#offline-support))
- No custom, tenant-authored role types (fixed role tiers only — see [Vision & Strategy](02-vision-and-strategy.md#product-philosophy--configurability-over-hardcoding))
- No multi-language UI beyond English/Filipino toggle (structured for i18n, not fully localized to regional languages at launch)

## Success metrics

| Metric | MVP target | Why it matters |
|---|---|---|
| Median time-to-first-response (submission → barangay verification) | < 24 hours | Primary citizen trust driver |
| Duplicate-report rate reaching human review | < 15% of submitted reports | Validates AI dedup is doing real work |
| Citizen confirmation rate (vs. silent/no response) | > 60% | Signals the loop actually closes, not just "marked done" by staff |
| Spam/invalid reports reaching Barangay queue | < 10% | Validates AI validation + rate limiting |
| Tenant onboarding time (new city, zero code changes) | < 1 business day | Proves the configurability thesis, the platform's core bet |

## Dependencies and assumptions

- LGU partner provides authoritative Barangay boundary GIS data (or the platform ships with a public boundary dataset as a fallback — see [Architecture](06-architecture.md)).
- LGU commits at least one Department Head and one Barangay Administrator for pilot configuration and UAT.
- Azure OpenAI/Vision access is provisioned in a region serving the Philippines within acceptable latency (see [AI Design](10-ai-design.md)).
