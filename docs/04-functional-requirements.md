# Functional Requirements

Requirements use RFC-2119-style **shall/should/may** language and are numbered for traceability into test plans and API design. Each is scoped to the MVP ([PRD](03-prd.md)) unless marked `[FUTURE]`.

## FR-1 — Report Submission

- **FR-1.1** The system shall allow a Guest Citizen to submit a report without creating an account.
- **FR-1.2** The system shall require at least one photo attachment per report.
- **FR-1.3** The system shall require a GPS coordinate per report, captured from device location, with an option for the citizen to manually adjust the pin before submission.
- **FR-1.4** The system shall require the citizen to select a report category from the tenant's configured category list.
- **FR-1.5** The system shall accept an optional free-text description, with a configurable maximum length.
- **FR-1.6** The system shall reject submission if photo or GPS is missing, with a clear client-side error before any network call.
- **FR-1.7** The system should allow a citizen to save an unsubmitted report as a local draft when offline, and submit automatically once connectivity resumes (see [NFR — Offline Support](05-non-functional-requirements.md#offline-support)).

## FR-2 — AI Validation

- **FR-2.1** The system shall run every submitted photo through an AI image-classification check before the report enters any human queue.
- **FR-2.2** The system shall flag a report as `NeedsReview` if AI confidence that the image matches the selected category falls below a tenant-configurable threshold.
- **FR-2.3** The system shall run spam-heuristic checks (rate limiting, duplicate image hash, blank/irrelevant image detection) prior to queueing.
- **FR-2.4** A report failing spam checks shall not be routed to a Barangay queue; it shall be logged and, for repeat offenders, shall count against the submitting account's reputation score (see [AI Design](10-ai-design.md#citizen-reputation)).
- **FR-2.5** AI validation shall never be the sole basis for permanently rejecting a report — a human (Barangay Staff) shall be able to override any AI validation outcome, and the override shall be audit-logged.

## FR-3 — Duplicate Detection

- **FR-3.1** The system shall check new reports against existing open reports within a tenant-configurable radius (default 100m) and time window (default 72 hours) for the same category.
- **FR-3.2** Where AI/geospatial similarity exceeds a configurable threshold, the system shall present the new report to Barangay Staff as a candidate duplicate rather than auto-merging it, unless confidence exceeds a second, higher auto-merge threshold.
- **FR-3.3** When a report is merged as a duplicate, the system shall link the citizen's submission to the parent report so the citizen still receives status notifications.

## FR-4 — GPS Routing

- **FR-4.1** The system shall resolve a submitted GPS coordinate to a Barangay using tenant-configured geospatial boundary data.
- **FR-4.2** The system shall resolve the responsible Department from the (Category × Barangay) configuration mapping.
- **FR-4.3** If no boundary match is found (e.g., coordinate outside city limits), the system shall route the report to a City Hall-level exception queue rather than silently discarding it.

## FR-5 — Barangay Queue & Verification

- **FR-5.1** The system shall present incoming reports to Barangay Staff sorted by configurable priority (category default × SLA time-remaining).
- **FR-5.2** Barangay Staff shall be able to Verify, Reject (with required reason), or Merge-as-duplicate a report.
- **FR-5.3** A Rejected report shall notify the submitting citizen with the reason, unless the citizen is a Guest without a notification channel.
- **FR-5.4** A Verified report shall become assignable to a Department.

## FR-6 — Assignment & Worker Execution

- **FR-6.1** Department Head/Staff with assignment permission shall be able to assign a Verified report to a specific Worker.
- **FR-6.2** A Worker shall see only reports assigned to them, never another worker's queue (see [Security — Authorization](09-security.md#authorization-model)).
- **FR-6.3** A Worker shall be able to transition an assigned report through: `Assigned → InProgress → BeforePhotosUploaded → Completed → AfterPhotosUploaded`.
- **FR-6.4** The system shall require at least one "before" photo before allowing transition to `Completed`, and at least one "after" photo before allowing transition to `AfterPhotosUploaded`.
- **FR-6.5** The system shall record GPS location and timestamp at each Worker-initiated transition, for SLA and fraud-prevention evidence.

## FR-7 — Inspection (Optional Step)

- **FR-7.1** The system shall allow a tenant to configure, per category, whether a Department Head or designated Inspector role must approve completion before the report moves to citizen confirmation.
- **FR-7.2** Where inspection is required and rejected, the report shall return to `Assigned` with the inspection notes attached.

## FR-8 — Citizen Confirmation & Closure

- **FR-8.1** Upon work completion (and inspection, if required), the system shall notify the submitting citizen and request confirmation.
- **FR-8.2** The citizen shall be able to confirm resolution (→ `Closed`) or dispute it (→ reopened, routed back to the responsible Department with the dispute reason).
- **FR-8.3** If the citizen does not respond within a tenant-configurable window (default 7 days), the system shall auto-close the report and log the auto-close reason.

## FR-9 — Notifications

- **FR-9.1** The system shall notify the citizen (push, and SMS/email fallback for verified accounts with contact info) on every status transition of their report.
- **FR-9.2** The system shall notify relevant staff (Barangay Staff on new report, Department on verification, Worker on assignment) via the same mechanism.
- **FR-9.3** Notification templates shall be tenant-configurable text, not hardcoded strings, to support localization and city-specific tone (see [Vision & Strategy](02-vision-and-strategy.md#product-philosophy--configurability-over-hardcoding)).

## FR-10 — Audit Logging

- **FR-10.1** Every state transition on every report shall produce an immutable audit log entry containing: actor (user or system/AI), timestamp, previous state, new state, and reason/metadata.
- **FR-10.2** Audit log entries shall not be editable or deletable through any application-layer API.
- **FR-10.3** City Hall Administrators and Super Administrators shall be able to view the full audit trail of any report within their tenant scope.

## FR-11 — Category Management

- **FR-11.1** A City Hall Administrator shall be able to create, rename, deactivate, and reorder report categories for their tenant without a code deployment.
- **FR-11.2** Deactivating a category shall not delete or hide historical reports filed under it.
- **FR-11.3** Each category shall support tenant-configurable default priority, SLA target, department routing rule, and whether inspection is required.

## FR-12 — Dashboards & Reporting

- **FR-12.1** Barangay, Department, City Hall, and Worker dashboards shall each show only data within the viewing user's authorization scope (see [Security](09-security.md)).
- **FR-12.2** City Hall Dashboard shall show aggregate SLA compliance, volume by category, and volume by barangay/department, filterable by date range.
- **FR-12.3** `[FUTURE]` Executive Dashboard shall provide cross-tenant, trend, and predictive views for Super Administrators and (future) Provincial/National tiers.

## FR-13 — Tenant Onboarding (Platform Admin)

- **FR-13.1** A Super Administrator shall be able to provision a new City/Organization tenant — including seed categories, default departments, and boundary data import — without engineering involvement, targeting the < 1 business day onboarding metric in the [PRD](03-prd.md#success-metrics).
- **FR-13.2** The system shall enforce that a new tenant's data (reports, users, configuration) is inaccessible to any other tenant by default (see [Security — Tenant Isolation](09-security.md#tenant-isolation)).

## FR-14 — Workflow Engine `[FUTURE-enabling, MVP-scoped]`

- **FR-14.1** The Incident Reporting workflow (FR-5 through FR-8) shall be implemented as a configured instance of a generic Workflow Engine, not as bespoke incident-specific code, so that future modules (permits, appointments, inspections) can be added as new workflow definitions (see [Architecture — Workflow Engine](06-architecture.md#workflow-engine)).
- **FR-14.2** The Workflow Engine shall support configurable states, transitions, required-role-per-transition, and required-evidence-per-transition (e.g., "photo required to enter this state") as data, not code.

## FR-15 — Category Taxonomy: Groups, Barangay Overrides & Request Routing

- **FR-15.1** Report categories shall belong to a category group (e.g. Infrastructure & Roads, Sanitation & Waste) for organizing the citizen-facing category picker. Grouping is organizational only — it does not affect routing, priority, or workflow.
- **FR-15.2** A City Hall Administrator shall be able to enable/disable a citywide category per barangay, and override its default priority, SLA target, or department routing for that barangay specifically, without affecting the category's configuration for other barangays (see [Database Design — Incident Reporting](07-database-design.md#4-incident-reporting-module-specific)).
- **FR-15.3** Because the citizen selects a category before GPS/barangay resolution occurs (FR-4.1), a category disabled for the citizen's resolved barangay shall be rejected at submission time with a clear reason — not filtered from the picker in advance, and not silently dropped or rerouted.
- **FR-15.4** Categories may be flagged as `request` rather than `complaint` kind (e.g., "Requests for Assistance"). A `request`-kind submission shall route to a distinct workflow definition rather than the standard Incident Reporting pipeline (FR-2 through FR-8). `[FUTURE]` The specific states/transitions of the request-handling workflow (eligibility, fulfillment tracking, etc.) are not defined by this requirement and need their own requirements pass before implementation — this requirement only obligates the category/routing distinction to exist.
