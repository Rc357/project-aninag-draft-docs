# Functional Requirements

Requirements use RFC-2119-style **shall/should/may** language and are numbered for traceability into test plans and API design. Each is scoped to the MVP ([PRD](03-prd.md)) unless marked `[FUTURE]`.

## FR-1 — Report Submission

- **FR-1.1** The system shall allow a Guest Citizen to submit a report without creating an account.
- **FR-1.2** The system shall require at least one media attachment per report — one or more photos (up to **5**), or a single video up to 3 minutes long, not both (previously photo-only; extended per the feed-visibility requirement in FR-20, then extended again to multiple photos per the social-post framing).
- **FR-1.3** The system shall require a GPS coordinate per report, captured from device location, with an option for the citizen to manually adjust the pin before submission.
- **FR-1.4** The system shall require the citizen to select a report category from the tenant's configured category list.
- **FR-1.5** The system shall accept an optional free-text description, with a maximum length of **280 characters** — the value already hardcoded in `review_screen.dart`'s `TextField`, a deliberate existing choice this requirement is documenting, not proposing. Tenant-configurable per the original wording.
- **FR-1.6** The system shall reject submission if both photo and video are missing, or GPS is missing, with a clear client-side error before any network call.
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
- **FR-5.5** The system shall record the timestamp and acting Barangay Staff member when a report is Verified, independent of whatever workflow state the report later moves through (FR-6 onward) — this is what lets a "Verified" badge (FR-19.1) remain accurate for a report that's since moved to `InProgress`, `Completed`, or `Closed`, without re-deriving verification status from the current state on every read.

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

## FR-16 — Authentication & Identity

- **FR-16.1** Supported identity providers shall be: Guest, Google, Apple, Email + OTP, Phone + OTP, and Facebook (extends the provider list in [Security — Authentication](09-security.md#authentication) and [Database Design](07-database-design.md#2-identity--access)).
- **FR-16.2** Every non-guest account shall have a unique, account-holder-chosen or auto-generated **username**, visible to other users by default. An account holder's real name (where a provider supplies one) shall never be shown to other users unless the account holder explicitly enables a "show my real name" setting — anonymous/pseudonymous is the default, not opt-in.
- **FR-16.3** For OAuth providers that don't collect a username directly (Google, Apple, Facebook), the system shall gate first-time sign-in behind a mandatory "choose a username" step before any other authenticated action is available. A suggested auto-generated pseudonym (e.g. "Citizen4471") shall be pre-filled so accepting it is a single tap, not friction — this is what keeps FR-16.2's anonymity-by-default priority practical rather than merely aspirational.
- **FR-16.4** Email + password / OTP signup shall collect the username as part of the signup form directly, without a separate post-auth gate (FR-16.3 applies only to providers that bypass a signup form).
- **FR-16.5** A username, once set, may be changed by its owner subject to a tenant-configurable cooldown (default 14 days) — limits impersonation via rapid renaming (e.g., renaming to impersonate a barangay official mid-conversation, then renaming away).
- **FR-16.6** A signed-in citizen shall be able to request deletion of their own account from within the app (Profile → account menu) — required for app store distribution (Apple App Store review guideline 5.1.1(v): an account-creating app must offer in-app account deletion). The request takes effect immediately from the citizen's perspective (signed out, account flagged) even though the underlying hard-delete of their `auth.users` row runs as a separate, asynchronous server-side process — see the design note below.
  - **Design note — request vs. hard delete:** actually erasing an `auth.users` row requires Supabase's admin API (the `service_role` key), which must run on a trusted server, never embedded in the mobile app. Since there's no backend service yet ([Architecture](06-architecture.md)), FR-16.6 is implemented as a flag (`user_account.deletion_requested_at`) that a recurring job or manual pass must still process into an actual delete. This satisfies the citizen-facing requirement (self-service, immediate, no "email us" step) without requiring backend infrastructure that doesn't exist yet — build the real job before this ships to an app store, not after.

## FR-17 — Community Reactions `[PROPOSED — see design note, needs confirmation before build]`

- **FR-17.1** A Verified Citizen (not Guest) may register one reaction per report: **support** (affirms the report — "this affects me too") or **dispute** (contests its accuracy). A user holds at most one reaction per report; casting the other replaces it, it does not add a second. Displayed in the mobile app as "Bump"/"Debump" (Reddit/Stack Overflow-style vote wording) — a UI label choice only; the requirement, the `ReactionKind` enum, and the database values all stay `support`/`dispute`.
  - **Design note:** the originally requested set (thumbsup, like, agree, disagree) collapses to two here — thumbsup/like/agree describe the same underlying intent (affirming the report), so modeling them as three separate reaction types would just fragment one signal for no product benefit. Revisit only if a real, distinct use case for more than two ever emerges.
- **FR-17.2** Reaction counts shall be visible to any viewer, including Guests browsing the transparency feed (`/reports/nearby`) — only casting a reaction requires being signed in, viewing the count does not.
- **FR-17.3** A **dispute** reaction shall require a brief written reason (tenant-configurable max length) — a bare disagree tap is not permitted. This exists specifically to raise the cost of dispute-brigading a legitimate but locally unpopular report (e.g., one implicating a powerful local interest).
- **FR-17.4** Reactions shall never directly affect a report's workflow state — FR-5 through FR-8 remain the sole state-transition authority. Reactions are a visible community signal only, never a moderation or approval mechanism; this closes off "reactions become a backdoor way to suppress reports" as a failure mode. This is the formal version of the "community confirmation" item already anticipated as a future control in [Security — Preventing Spam & Abuse](09-security.md#preventing-spam--abuse).

## FR-18 — Comments & Moderation `[PROPOSED — see design note, needs confirmation before build]`

- **FR-18.1** A Verified Citizen (not Guest) may post a text comment on a report; comments are visible to any viewer.
- **FR-18.2** Any viewer, including Guests, may flag a comment as abusive/inappropriate with a reason. A flagged comment is **not** automatically hidden — it's surfaced to Barangay Staff for that report's barangay (the existing role from FR-5, not a new moderation team) as part of their review queue.
  - **Design note:** pre-moderating every comment before it's visible doesn't scale against the team size this platform is scoped for (see [Vision & Strategy](02-vision-and-strategy.md)) — flag-then-review is the same "human backstop, not automated gate" philosophy FR-2.5 already applies to AI validation.
- **FR-18.3** Barangay Staff may hide a flagged comment. A hidden comment is not deleted (preserves the audit trail per FR-10); its author is notified with the reason.
- **FR-18.4** Comment posting is rate-limited using the same principle as report submission (FR-2.3).
- **FR-18.5** Repeated flagged/hidden comments from one account count against that account's reputation score — the same mechanism FR-2.4 already uses for reports, not a second parallel reputation system.

## FR-19 — Verification Badge (Transparency Feed & Report Detail)

- **FR-19.1** A report shall display a "Verified" badge to any viewer, including Guests, if and only if it has passed Barangay Staff verification (FR-5.2/5.5) — i.e. `verifiedAt` is set. This applies on the transparency feed (`/reports/nearby`), a citizen's own report list, and report detail.
- **FR-19.2** AI validation (FR-2) shall never be surfaced as a citizen-facing badge or signal, verified or otherwise — it is an internal triage input to the human queue (FR-2.1–2.5), not a claim about the report's accuracy that citizens should be shown. A report that has merely passed AI validation but not yet Barangay Staff review shows no badge, the same as a report AI flagged for review — the citizen-facing distinction is binary (human-verified, or not yet), not graduated by AI confidence.
- **FR-19.3** The badge shall be visually and textually distinct from the report's workflow status label (e.g. "In progress," "Resolved," per the `ReportStatus` labels already in the mobile app) — verification and workflow progress are answers to two different questions ("is this real?" vs. "where is it in the process?") and conflating them into one indicator would lose one of the two signals.

## FR-20 — Feed Visibility & Media

Implemented in the mobile app: the home screen is a scrollable feed of `ReportFeedPost` cards (photo/video, description, status, verified badge, one-tap support, comment count → detail), refreshed via pull-to-refresh rather than a live subscription — see the design note on `SupabaseReportRepository.watchNearby`/`watchMine` for why realtime streaming and the joined data a feed card needs (media, reaction counts) don't coexist. `ReportCard`/`ReportListTile` (the earlier compact list style) is kept for My Reports, a separate, lower-density personal-utility view — not replaced everywhere.

- **FR-20.1** Reports shall be browsable in a scrollable public feed (the existing `/reports/nearby` transparency feed, FR-12/FR-17/FR-18 already establish its reaction/comment visibility rules) — this requirement establishes the feed itself as the primary citizen-facing surface, not just an incidental endpoint, per the source request's "feed like Facebook/TikTok" framing.
- **FR-20.2** A report's required media (FR-1.2) may be one or more photos (up to **5**, Facebook-album style) or a single video up to **3 minutes** in length — exactly one of the two kinds, never both, never neither; either satisfies FR-1.2/1.6.
- **FR-20.3** Video shall be compressed client-side before upload — target: capped resolution (1080p) and bitrate, re-encoded to a widely-supported codec (H.264), before the file ever reaches Storage. This is a storage/bandwidth cost control, not a quality feature — see design note below on why client-side, not server-side.
  - **Design note — client-side, not server-side, compression:** transcoding server-side (e.g., an Edge Function invoking ffmpeg) would mean uploading the full uncompressed file first, defeating the bandwidth-saving goal, and adds server-side compute cost + processing latency before a report is even queryable. Client-side compression trades a few seconds of on-device processing time (during which the citizen sees a "preparing video…" state) for a smaller upload and no server transcoding pipeline to build or pay for.
- **FR-20.4** The system shall enforce the 3-minute cap and a maximum post-compression file size **(tenant-configurable, default 50MB)** client-side, before upload — same "reject before any network call" principle FR-1.6 already applies to missing photo/GPS.
- **FR-20.5** `[FUTURE]` Full TikTok/Facebook-style engagement (autoplay video feed, algorithmic ranking, follow/share) is out of scope for this requirement — FR-20.1 through 20.4 establish browsability + media capture only. Treat additional feed-engagement mechanics as their own requirements pass, not an implied extension of this one.

## FR-21 — In-App Notifications

Implemented in the mobile app: a fourth bottom-nav tab, distinct from FCM push (`PushNotificationService` — an ephemeral OS-tray alert). This is the browsable history a citizen can revisit inside the app, populated entirely server-side by triggers on `report_reaction`/`report_comment`/`report` — never written to directly by the client (see `.test_folder/supabase-setup-guide.md` §11).

- **FR-21.1** The system shall notify a report's submitter (never the reacting/commenting citizen themself) when: another citizen reacts to their report (FR-17), another citizen comments on their report (FR-18), their report is verified (FR-19), or their report's workflow status changes (FR-14).
- **FR-21.2** Notifications shall be scoped strictly to activity on reports the citizen themself submitted — not full comment-thread-subscriber semantics (a citizen who comments on someone else's report is not notified of further comments from other participants).
- **FR-21.3** A guest-submitted report (FR-1.1) generates no notifications — there is no `user_account` row to notify against. Consistent with guest submission itself not being wired up against the real backend yet.
- **FR-21.4** `[FUTURE]` Staff-facing notifications (a new report needs review, a comment was flagged) are out of scope for this requirement — FR-21.1 through 21.3 cover citizen-facing notifications on the citizen's own reports only. Needs its own pass once a staff-facing app exists.
