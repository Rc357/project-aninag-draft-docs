# Analytics & Crash Reporting

## Tooling

**Firebase Crashlytics + Firebase Analytics + Firebase Performance Monitoring** — one Firebase project already backing [push notifications](07-device-capabilities.md#push-notifications-firebase-cloud-messaging) and [performance monitoring](11-performance-and-accessibility.md#what%27s-measured-and-how), so this adds no new vendor integration, just additional SDKs against infrastructure already in place. Consistent with the platform's general bias toward not adding a tool where an existing one already covers the need (see [Architecture](../docs/06-architecture.md) for the same reasoning applied to the backend stack).

## What gets tracked, and why it's not arbitrary

Every tracked event should trace back to either a [PRD success metric](../docs/03-prd.md#success-metrics) or a concrete debugging need — analytics for its own sake is noise that costs engineering time to maintain and citizen trust to justify.

| Event | Feeds |
|---|---|
| Report submission funnel: category selected → photo added → GPS confirmed → description added → submitted (with drop-off point on abandonment) | Where citizens abandon the flow is the single most actionable mobile-specific signal for improving the [PRD](../docs/03-prd.md)'s core loop — not currently measurable any other way |
| Citizen confirmation action (confirmed resolved / disputed) taken from push notification vs. from opening the app cold | Informs whether push notification copy/timing is actually driving the confirmation-rate target |
| Worker: time from job assignment to "Start Job" tap, and start to completion | Operational signal for [Roadmap — Phase 1](../docs/11-roadmap.md#phase-1--single-city-pilot) pilot review, distinct from the SLA data the backend already tracks — this captures app-side friction (e.g., a worker opening the app but hesitating before starting) that server-side timestamps alone can't distinguish from field conditions |
| Crash-free session rate, per app, per OS version | Standard release-health signal — gates whether a staged rollout ([Build, Release & CI/CD](10-build-release-cicd.md#store-submission-process)) proceeds to the next percentage |

**Not tracked:** granular UI interaction heatmaps, screen-view duration for its own sake, or anything that doesn't map to one of the above categories — resisting analytics sprawl is easier to maintain than pruning it later.

## PII handling

This is the section most worth getting right given [Security — Compliance](../docs/09-security.md#compliance) (RA 10173) applies to analytics/crash data exactly as much as it applies to the primary database:

- **Crashlytics breadcrumbs and analytics events never include:** raw GPS coordinates, phone numbers, email addresses, or free-text report descriptions. Where an event needs to reference *a* report, it uses the report's tracking ID (already designed to be an opaque, non-PII identifier per [Database Design](../docs/07-database-design.md)) — never anything a support engineer reading a crash log could use to identify a specific citizen's home address.
- **User identifiers sent to Firebase are the app's own internal user ID**, not the Verified Citizen's phone/email directly — consistent with treating PII and operational/analytics data as distinct classes, the same separation [Security — Data Protection](../docs/09-security.md#data-protection) establishes for the backend.
- **Guest users** are tracked as anonymous Firebase installations with no cross-session identity linkage beyond what's needed for the submission-funnel event sequence within a single session — there is no attempt to build a persistent guest profile across app reinstalls.
- **Consent:** Analytics collection is disclosed in the app's privacy notice at first launch (both apps), consistent with RA 10173's transparency requirement; it is not gated behind an opt-in toggle at MVP since the data collected is de-identified operational telemetry, not marketing tracking — this is a judgment call worth revisiting if legal counsel reviewing the actual privacy policy for a specific pilot LGU determines otherwise.

## What Crashlytics is for, and what it isn't

Crashlytics answers "did this crash, and how often" — it is not a substitute for the [structured audit log](../docs/07-database-design.md#3-workflow-engine-generic) that records *why* a report reached a given state. A crash in the photo-capture screen and a citizen's report being stuck in `PendingAIValidation` are different classes of problem, diagnosed with different tools (Crashlytics vs. the backend's `WorkflowEvent` audit trail) — conflating them would send whoever's debugging looking in the wrong place.
