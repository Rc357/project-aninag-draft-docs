# Offline Support

Implements [NFR — Offline Support](../docs/05-non-functional-requirements.md#offline-support): capture works offline, submission is deferred and queued, full offline browsing is explicitly not a goal.

## What's actually offline-capable

| Action | Offline behavior |
|---|---|
| Compose a new citizen report (category, photo, GPS, description) | Fully offline — saved as a local Drift draft, GPS captured device-locally |
| Submit a report | Queued if offline, sent automatically on reconnect |
| Worker: capture before/after photos, fill checklist | Fully offline — same queueing pattern as citizen drafts |
| Worker: submit job transition (start/complete) | Queued if offline, sent automatically on reconnect |
| Browse My Reports / Assigned Jobs | Last cached list only (see [State & Data Layer — Cache Strategy](05-state-and-data-layer.md#cache-strategy--cache-then-network-not-cache-only-or-network-only)) — no offline pagination beyond what's cached |
| View a report/job detail not previously loaded | Not available offline — this is a genuine gap, not a bug, since there's nothing to show for data never fetched |
| Real-time status updates (push notifications) | Not delivered offline by definition — the app reconciles on next launch/foreground via a fresh fetch |

The MVP's offline need is **capture at the point of observation** — a citizen standing at a flooded street with no signal, a worker in a basement with no reception — not offline *review*. Building full offline-first sync (bidirectional, conflict-resolved) for browsing would be solving a problem the target usage pattern doesn't actually have.

## The submission queue

- Every draft (citizen report or worker job-transition) is a row in a Drift table: payload fields, photo file paths, the idempotency key generated at draft-creation time (see [API Integration — Idempotency](06-api-integration.md#idempotency-for-report-submission)), a status (`draft`, `queued`, `submitting`, `submitted`, `failed`), and a retry count.
- A **connectivity listener** (`connectivity_plus`) triggers queue processing whenever the device regains network — the app doesn't need to be reopened for a queued report to go out.
- A **foreground queue processor** also runs on app resume, so a citizen who reopens the app after a connectivity gap sees their queued reports move to "Submitted" without needing to do anything.
- **No background OS-level sync task (e.g., WorkManager) at MVP** — background execution guarantees on both platforms are unreliable enough (especially iOS) that the connectivity-listener + on-resume pattern above covers the realistic case (user has the app open or reopens it soon after) without taking on the complexity and battery-usage scrutiny of a true background sync job. Revisit only if pilot data shows a meaningful share of queued reports are going stale because the app is never reopened promptly — an evidence-based bar, not a default.

## Upload retry

Each queued item retries with exponential backoff (capped, e.g., 3 attempts before surfacing a "couldn't submit — check connection" state to the user rather than retrying silently forever). A failed item never blocks the rest of the queue — an independent retry state per item, not a single-file-at-a-time gate, since one bad upload shouldn't hold up an otherwise-connected citizen's other pending reports.

## What happens to the UI while something is queued

A queued (not-yet-confirmed-by-server) report appears in My Reports immediately with a distinct, honest status — "Waiting to send" — never disguised as "Submitted" (which the tracking ID and server-assigned workflow state make meaningful only after the server accepts it). This matters for trust: a citizen should never believe a report reached the barangay when it's actually still sitting on their phone.

## Conflict handling — mostly a non-issue by design

Citizen report submission and worker job-transitions are **creates and forward-only state transitions**, not edits to shared mutable state — there's no scenario where two offline edits to the *same* record need reconciling. The one place a real ordering question exists is a Worker attempting a transition (e.g., "complete job") based on stale cached job state; the server is the single source of truth for whether a transition is valid (per [Architecture — Workflow Engine](../docs/06-architecture.md#workflow-engine)), so an offline-queued transition that's no longer valid by the time it's sent (e.g., the job was reassigned) is rejected by the API and surfaced to the worker as a clear, specific error — not silently dropped or force-applied client-side.
