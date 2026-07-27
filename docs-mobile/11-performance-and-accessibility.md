# Performance & Accessibility

Translates [NFR — Performance](../docs/05-non-functional-requirements.md#performance) and [NFR — Accessibility](../docs/05-non-functional-requirements.md#accessibility) into concrete mobile engineering budgets — a target that isn't a number isn't a target.

## Minimum supported devices

- **Android:** API 24 (Android 7.0) minimum, tested primarily against a 2GB-RAM-class device profile — chosen because this is the realistic low end of the target citizen user base, not the median developer's test device. Higher API levels get progressive feature enhancement (e.g., newer permission dialogs), not a different feature set.
- **iOS:** last 3 major iOS versions, tested on at least one older/smaller-screen device (e.g., iPhone SE class) alongside a current device — iOS device fragmentation is lower than Android's, but screen-size and performance-class variance still matters for the same reasons.

## Performance budgets

| Metric | Budget | Why |
|---|---|---|
| Cold start (app icon tap → first interactive frame) | < 2.5s on the low-end Android profile | A citizen standing at an incident site abandoning the app before it opens defeats the entire product |
| Report-submission photo capture → local save | < 1s perceived (optimistic UI: show "saved" before EXIF-strip/compression finishes in the background) | Matches [NFR — Performance](../docs/05-non-functional-requirements.md#performance)'s submission-latency target; the client shouldn't add its own lag on top of the API's |
| App install size (per app, per platform) | Track and budget explicitly (target: stay under typical low-end-device comfortable install size — flag if either app crosses ~50MB) | Citizens on limited mobile data/storage are a real segment of the target market, not an edge case |
| List scroll (My Reports / Assigned Jobs, 50+ items) | Sustained 60fps | Cheap to get right early (see caching/pagination in [State & Data Layer](05-state-and-data-layer.md#pagination) — actually covered in [API Integration](06-api-integration.md#pagination)), expensive to retrofit once a screen's widget tree has grown organically |

## Photo handling for bandwidth

Photos are **compressed client-side before upload** (target: resize to a sensible max dimension and re-encode at a quality level tuned for legibility of the incident, not full-resolution fidelity) — this is a genuine performance *and* cost lever, not just a nicety: it directly reduces upload time on the mobile data connections much of the target user base has, and it reduces Azure Blob Storage + bandwidth cost at scale (the same cost-consciousness already applied to [AI](../docs/10-ai-design.md#cost--operational-considerations) and [Maps](07-device-capabilities.md#maps) elsewhere in this stack).

## Accessibility

Beyond the [Design System's](04-design-system.md) touch-target and color-contrast decisions:

- **Screen reader support (TalkBack/VoiceOver):** every interactive widget in `core/design_system` carries an explicit `Semantics` label describing the action in plain language ("Report an issue", not "Floating action button") — set once in the shared component, not left to each screen to remember to add.
- **Text scaling:** the app respects the system font-size setting (`MediaQuery.textScaler`) up to a capped maximum multiplier, rather than either ignoring it (a real accessibility failure for low-vision users) or allowing unbounded scaling that breaks the fixed-size phone-frame-style layouts the design system assumes — capped scaling is the practical middle ground.
- **Color is never the only signal:** every status representation pairs a color with a text label and/or icon (the `StatusChip` and `ReportCard` stripe-plus-chip pattern in [Design System](04-design-system.md#component-library-coredesign_systemwidgets)) — this is both a WCAG requirement and directly relevant given a meaningful share of any large population has some form of color vision deficiency.
- **Reduced motion:** any transition/animation (screen transitions, the SLA-countdown color escalation) respects `MediaQuery.disableAnimations` — functional state changes still happen instantly, only the animated presentation is skipped.

## What's measured, and how

Cold-start and frame-rendering metrics are captured via Firebase Performance Monitoring (paired with the [Crashlytics/Analytics](12-analytics-and-crash-reporting.md) setup, same Firebase project, no separate tooling to integrate) specifically on real low-end devices in the field during [Roadmap — Phase 1 pilot](../docs/11-roadmap.md#phase-1--single-city-pilot) — a budget that's only validated against a developer's flagship test phone isn't actually validated.
