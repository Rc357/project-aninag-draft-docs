# Mobile Roadmap

Maps mobile-specific work onto the platform-wide phases in [Roadmap](../docs/11-roadmap.md) — this doesn't introduce new phases, it fills in what "Phase 0," "Phase 1," etc. concretely mean for the two mobile apps.

## Phase 0 — Foundations (mobile scope)

- Repo scaffold per [App Architecture](02-app-architecture.md): flavors, `core/` packages (networking, storage, design_system, routing), feature folder skeletons.
- Design system implementation ([Design System](04-design-system.md)): tokens, component library, one committed theme.
- Citizen app happy path: all 8 wireframed screens ([Screens & Navigation](03-screens-and-navigation.md)) plus the flagged-but-not-yet-wireframed auth screens.
- Worker app happy path: all 5 wireframed screens plus staff sign-in.
- Offline queue ([Offline Support](08-offline-support.md)) and push notifications ([Device Capabilities](07-device-capabilities.md)) wired end-to-end against a real backend, not stubbed.
- CI pipeline producing installable staging builds ([Build, Release & CI/CD](10-build-release-cicd.md)).

**Two corrections to carry from wireframe → implementation, called out explicitly so they aren't silently lost:**

1. **Category list must be fetched, never hardcoded.** The wireframe's category grid shows a fixed 14-category set for illustration, but [FR-11](../docs/04-functional-requirements.md#fr-11--category-management) requires categories to be tenant-configurable, and [FR-13.1](../docs/04-functional-requirements.md#fr-13--tenant-onboarding-platform-admin) specifically targets onboarding a new city with *zero code changes*. If the Citizen app ships with the category list compiled into the client, onboarding city #2 with a different category set would require an app update — directly contradicting that target. The category screen must render from the tenant's configured category list (fetched via the equivalent of `/admin/categories`, scoped by the citizen's resolved tenant), with icon/priority metadata coming from that same config, not from a client-side enum.
2. **The submitted-report status timeline must reflect the real v1 pipeline, not the wireframe's illustrative one.** The wireframe's "Submitted" confirmation screen shows "AI Validation" as the first pipeline stage — but [AI features are deferred post-MVP](../docs/10-ai-design.md) for budget reasons, so v1's actual flow is Submitted → Duplicate check (geospatial-only, no AI) → Barangay Review, with no AI stage in between. The timeline component ([Design System — Timeline](04-design-system.md#component-library-coredesign_systemwidgets)) already renders whatever steps the backend's `WorkflowInstance` reports, so this is a content/copy correction at the backend workflow-definition level, not a client code change — but worth flagging here so nobody implements a client-side assumption that an "AI Validation" step always exists.

## Phase 1 — Single-City Pilot (mobile scope)

- Real-device testing on the low-end Android profile ([Performance & Accessibility](11-performance-and-accessibility.md)) — not just emulators.
- Crashlytics/Analytics wired and actually reviewed weekly against the [PRD success metrics](../docs/03-prd.md#success-metrics), not just installed and ignored.
- Staged Play Store/TestFlight rollout ([Build, Release & CI/CD](10-build-release-cicd.md#store-submission-process)) timed against the pilot's 8–12 week window in the [platform Roadmap](../docs/11-roadmap.md#phase-1--single-city-pilot).
- Fast bug-fix loop via Firebase App Distribution for pilot-partner staff, since store review latency is too slow for pilot-phase iteration speed.

## Phase 2 — Multi-City Expansion (mobile scope)

Mobile should need **no code changes** to onboard city #2, by construction — if it does, that's evidence the Phase 0 category-list correction above (or an equivalent tenant-config assumption) wasn't actually made generic, and is worth treating as a bug against Phase 0, not a Phase 2 feature request. This is the mobile-side instance of the same validation the [platform Roadmap's Phase 2](../docs/11-roadmap.md#phase-2--multi-city-expansion-same-module) already calls for at the backend/config level.

## Phase 3 — Second Module (mobile scope)

When a second backend module ships (Barangay Clearance or Business Permit, per [platform Roadmap Phase 3](../docs/11-roadmap.md#phase-3--second-module-prove-the-workflow-engine-thesis)), its mobile counterpart is a new `features/` folder ([App Architecture](02-app-architecture.md#project-structure--one-codebase-two-app-targets-not-a-monorepo)) plus new screens/routes — not a restructuring of the Incident Reporting feature or its shared `core/` packages. If adding it requires touching `core/networking`, `core/design_system`, or the Incident Reporting feature's own code, that's the same architecture-review trigger the platform-level roadmap flags for the backend equivalent.

## Explicitly not scheduled yet

- **AI-dependent UI** (e.g., surfacing an AI confidence score or AI-suggested category to the citizen) — no mobile work until [AI features are actually funded and built](../docs/10-ai-design.md).
- **Provincial/National-tier mobile screens** — those tiers are dashboard/policy-facing (per [Vision & Strategy](../docs/02-vision-and-strategy.md#target-users)), not mobile-app users; no mobile roadmap item exists for them.
- **Tablet-optimized layouts** — MVP targets phone form factors; revisit only if a specific pilot LGU's Worker deployment is tablet-based in practice.
