# Design System

This translates the visual language from the [Mobile UI Concepts artifact](https://claude.ai/code/artifact/e745faa6-fbdc-46c4-ab0d-0a7b8461c9d5) into implementable Flutter tokens and a shared widget library (`core/design_system`), so "what the mockup shows" and "what `ThemeData` produces" cannot silently drift apart.

## One committed theme, not light/dark

The mockup deliberately committed to a single fixed visual language regardless of viewer theme preference (see the artifact's own note on this). That decision carries into the app: **`ThemeData` ships one theme, not a `ThemeData.light()`/`ThemeData.dark()` pair.** This isn't an oversight to fix later — for a civic/public-service app used briefly and functionally (report an issue, check a status), a single, consistent, high-contrast presentation is more valuable than accommodating system dark-mode preference, and it removes an entire class of contrast/legibility QA (every color combination validated once, not twice). Revisit only if pilot user feedback specifically asks for it.

## Color tokens

```dart
class FmtColors {
  static const ink        = Color(0xFF16211D);
  static const paper      = Color(0xFFF4F1E6);
  static const surface    = Color(0xFFFFFFFF);
  static const muted      = Color(0xFF5B6B66);
  static const line       = Color(0xFFE3DDC9);

  static const brand      = Color(0xFF06AED5); // boracay blue
  static const brandInk   = Color(0xFF00576B); // for text/headers/pressed states
  static const brandTint  = Color(0xFFB1E9F6); // light backgrounds, chips

  static const amber      = Color(0xFFC97A2B); // in-progress
  static const amberTint  = Color(0xFFF6E6D3);
  static const green      = Color(0xFF3F7D4C); // resolved
  static const greenTint  = Color(0xFFDFEEE1);
  static const red        = Color(0xFFB23A2E); // urgent / alert
  static const redTint    = Color(0xFFF6DFDC);
  static const blue       = Color(0xFF3A5A7D); // pending / informational
  static const blueTint   = Color(0xFFDEE6EE);
}
```

**Semantic status colors (`amber`/`green`/`red`/`blue`) are separate from the brand accent (`brand` teal)** by design — a status chip's color encodes _what state a report is in_, which must stay legible and consistent even where brand color is also present on screen (e.g., a teal app bar with an amber "in progress" chip inside it). Never repurpose a status color for a non-status UI element (e.g., don't use `amber` for a random highlighted button) — that would erode the one signal citizens and workers are trained to scan for at a glance.

## Typography

**Roboto, one family, across both iOS and Android** — a deliberate divergence from the mockup's "authentic native OS font" reasoning (which was about making static mockup images read as real phone screenshots). In the shipped app, Flutter's Material widgets default to Roboto on both platforms anyway; fighting that with iOS-conditional San Francisco rendering adds platform-branching complexity for a brand-consistency benefit that matters more than the win from platform-native text — a small team should spend that effort elsewhere. Revisit only if design/brand feedback specifically calls for platform-native type.

| Role      | Size / weight                                       | Usage                                                                        |
| --------- | --------------------------------------------------- | ---------------------------------------------------------------------------- |
| `display` | 22sp / 800                                          | Screen titles rarely used ("Report submitted")                               |
| `title`   | 15.5sp / 800                                        | App-bar titles, card titles                                                  |
| `body`    | 13sp / 500                                          | Descriptions, list body text                                                 |
| `label`   | 10.5sp / 800, +0.09em tracking, uppercase           | Section labels ("PHOTO _REQUIRED_", "REVIEW")                                |
| `caption` | 11sp / 600                                          | Metadata (timestamps, secondary info)                                        |
| `mono`    | 12sp / 700, Roboto + `FontFeature.tabularFigures()` | Tracking IDs, SLA countdowns — tabular so digits align when they update live |

No separate monospace font asset is bundled — `FontFeature.tabularFigures()` on Roboto satisfies the tabular-numeral need (SLA timers, tracking IDs) without the app-size cost of an additional font family.

## Spacing & radius scale

```dart
class FmtSpace { static const xs=4.0, sm=8.0, md=12.0, lg=16.0, xl=24.0, xxl=32.0; }
class FmtRadius { static const tile=13.0, card=14.0, button=12.0, pill=999.0; }
```

Applied via `Row`/`Column`/`Wrap` `spacing`/`runSpacing` (Flutter 3.27+) or explicit `SizedBox` gaps between siblings — never accumulated margins on individual children, for the same reason the web design system avoids margin-collapse ambiguity: one source of truth for the gap between two elements, not two children each contributing half.

## Component library (`core/design_system/widgets`)

| Widget                                        | Maps to mockup element                     | Key behavior                                                                                                                                                                                                                                                       |
| --------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `FmtPrimaryButton` / `FmtOutlineButton` | `.btn-primary` / `.btn-outline`            | Full-width by default; disabled state uses a desaturated fill, never just lowered opacity (opacity alone can fail contrast checks)                                                                                                                                 |
| `StatusChip`                                  | `.chip-pending/-progress/-resolved/-alert` | Takes a `ReportStatus` enum, not a raw color — prevents a screen from inventing a new ad hoc status color                                                                                                                                                          |
| `ReportCard`                                  | `.card` with `.stripe`                     | Severity-color left stripe is a required parameter, not optional styling — encodes state in form, not just the chip text, per accessibility redundancy (color is never the _only_ signal — see [Performance & Accessibility](11-performance-and-accessibility.md)) |
| `CategoryTile`                                | `.category-tile`                           | Visual tile can stay compact, but **minimum 48×48dp tap target** is enforced via `InkWell` hit-area padding regardless of visual icon size — see the touch-target note below                                                                                       |
| `PhotoSlot`                                   | `.photo-slot`                              | Empty/filled/required states; wraps the actual camera capture flow (see [Device Capabilities](07-device-capabilities.md))                                                                                                                                          |
| `MapPreview`                                  | `.map-box`                                 | Thin wrapper around the chosen map SDK widget (see [Device Capabilities](07-device-capabilities.md)), not a custom map renderer                                                                                                                                    |
| `Timeline` / `TimelineStep`                   | `.timeline`                                | Renders a `List<WorkflowStepStatus>` from the API's workflow state — presentation-only, no business logic about _which_ steps exist (that's server/workflow-engine-defined, per [Architecture — Workflow Engine](../docs/06-architecture.md#workflow-engine))      |
| `StepperDots`                                 | `.stepper-mini`                            | Multi-step form progress (category → capture → review)                                                                                                                                                                                                             |
| `SlaCountdownText`                            | `.sla`                                     | Ticking countdown using `mono` type; color escalates `blue → amber → red` as the SLA deadline approaches, thresholds sourced from the category's configured SLA, not hardcoded                                                                                     |

## Touch targets — a deliberate departure from the mockup's visual density

The wireframe's category grid (14 tiles in a compact 3-column layout) was sized for a static visual mockup. In the real app, **every interactive element gets a minimum 48×48dp hit area** (Material accessibility baseline, and the concrete implementation of the "Accessible" principle and WCAG 2.1 AA touch-target guidance in [NFR — Accessibility](../docs/05-non-functional-requirements.md#accessibility)), even where the visible icon/label is smaller — achieved via `InkWell`/`GestureDetector` padding, not by simply enlarging the visual tile (which would blow out the layout density the mockup intentionally used). This is the kind of gap that's easy to miss translating a static mockup into a real, tappable app — called out explicitly so it isn't.

## Icons

The mockup's inline SVG line-icon set (category icons, UI chrome icons) ports to Flutter as a small custom icon font or `CustomPainter`/vector-drawable set — not `Icons.*` Material defaults, which don't have matching glyphs for civic-specific concepts (flooding, illegal dumping, barangay pin). Recommend generating a custom icon font from the existing SVG set (via `fluttericon`/`icomoon`-style tooling) once the set is finalized, rather than hand-porting each SVG path into a `CustomPainter` — less code to maintain per icon, consistent sizing/baseline behavior for free.
