# Design System

This translates the visual language from the [Mobile UI Concepts artifact](https://claude.ai/code/artifact/e745faa6-fbdc-46c4-ab0d-0a7b8461c9d5) into implementable Flutter tokens and a shared widget library (`core/design_system`), so "what the mockup shows" and "what `ThemeData` produces" cannot silently drift apart.

## One committed theme, not light/dark

The mockup deliberately committed to a single fixed visual language regardless of viewer theme preference (see the artifact's own note on this). That decision carries into the app: **`ThemeData` ships one theme, not a `ThemeData.light()`/`ThemeData.dark()` pair.** This isn't an oversight to fix later — for a civic/public-service app used briefly and functionally (report an issue, check a status), a single, consistent, high-contrast presentation is more valuable than accommodating system dark-mode preference, and it removes an entire class of contrast/legibility QA (every color combination validated once, not twice). Revisit only if pilot user feedback specifically asks for it.

## Color tokens

Only 3 colors are independently chosen — `ink` (text), `surface` (background), `brand` (accent). Everything else is either derived from one of those three, a structural neutral needed regardless of brand (`muted`/`line`), or a semantic status color. This mirrors the reference app (bluehive-project/AfexVisitor2025), which derives its own tint/shade family from a single primary hue rather than hand-picking a separate hex per variant.

```dart
class FmtColors {
  static const ink        = Color(0xFF16211D);
  static const surface    = Color(0xFFFFFFFF);
  static const brand      = Color(0xFF06AED5); // boracay blue

  static const muted      = Color(0xFF5B6B66);
  static const line       = Color(0xFFE3DDC9);

  // Derived from `brand`, not separately hand-picked:
  static final brandInk  = Color.lerp(brand, Colors.black, 0.5)!; // text/icons on light bg
  static final brandTint = brand.withValues(alpha: 0.3);          // light backgrounds, chips

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

`paper` was merged into `surface` — both were already the same literal white, kept as two names for no reason. Disabled buttons use `muted` at reduced opacity for the fill (`muted.withValues(alpha: 0.2)`) plus solid `muted` for text/icon — not `line`, which despite the name is a warm khaki tone, not a neutral grey, and read as an odd color for a "disabled" state rather than a clearly greyed-out one.

**Semantic status colors (`amber`/`green`/`red`/`blue`) are separate from the brand accent (`brand` teal)** by design — a status chip's color encodes _what state a report is in_, which must stay legible and consistent even where brand color is also present on screen (e.g., a teal app bar with an amber "in progress" chip inside it). Never repurpose a status color for a non-status UI element (e.g., don't use `amber` for a random highlighted button) — that would erode the one signal citizens and workers are trained to scan for at a glance.

## Typography

**Inter, one family, across both iOS and Android**, via `google_fonts` — matches the reference app (bluehive-project/AfexVisitor2025), whose body text is Inter Regular at the same 16sp/w400. This is a change from a brief Lexend pass tried right before it: Lexend carries noticeably thicker strokes than Inter at the *same* nominal weight, so text kept reading "bold" even after weights were dialed down — the font itself, not just the weight number, was the mismatch. Also a divergence from the mockup's "authentic native OS font" reasoning (which was about making static mockup images read as real phone screenshots) and from an earlier Roboto-default pass. Revisit only if design/brand feedback specifically calls for platform-native type.

| Role      | Size / weight                                      | Usage                                                                        |
| --------- | --------------------------------------------------- | ---------------------------------------------------------------------------- |
| `display` | 26sp / 700                                          | Screen titles rarely used ("Report submitted"), the welcome-screen wordmark |
| `title`   | 18sp / 700                                          | App-bar titles, card titles                                                  |
| `body`    | 16sp / 400                                          | Descriptions, list body text                                                 |
| `label`   | 13sp / 700, +0.6 tracking, uppercase                | Section labels ("PHOTO _REQUIRED_", "REVIEW")                                |
| `caption` | 13sp / 500                                          | Metadata (timestamps, secondary info)                                        |
| `mono`    | 14sp / 700, Inter + `FontFeature.tabularFigures()`  | Tracking IDs, SLA countdowns — tabular so digits align when they update live |

Weights were dialed back from an earlier pass (was: display/title/label all 800, body 500, caption 600) after a direct "too bold" complaint — heavy weight everywhere was contributing to that independent of font choice. Bold is now reserved for roles that should actually stand out (titles, labels, buttons); body/caption read at a normal weight.

No separate monospace font asset is bundled — `FontFeature.tabularFigures()` on Inter satisfies the tabular-numeral need (SLA timers, tracking IDs) without the app-size cost of an additional font family.

`FmtText`'s fields are `static final`, not `static const` — `GoogleFonts.inter(...)` lazily registers/loads the font file on first call, so it isn't a const constructor. Anything that directly embeds an `FmtText` value in an otherwise-`const` widget tree (the app's `ThemeData.textTheme`/`appBarTheme` do this) has to drop that local `const`, same as any other non-const value.

**Sizes bumped from an earlier pass** (was: display 22/title 15.5/body 13/label 10.5/caption 11/mono 12) after a direct readability complaint — those numbers ran meaningfully smaller than typical app body text (Material's own `bodyLarge` default is 16sp; iOS HIG recommends 17pt minimum for body copy), which hits older/low-vision users hardest. The bigger problem wasn't `FmtText` itself, though — most on-screen text was hardcoded `TextStyle(fontSize: N)` scattered across individual widgets, entirely bypassing these shared roles (11, 9.8, 10.5, 12.5... each widget inventing its own slightly-different small number). Fixed at the root with a new token layer:

```dart
class FmtFontSize {
  static const xs = 12.0;      // floor — status chips, Verified badge, never smaller
  static const sm = 13.0;      // captions, meta text, timestamps
  static const md = 14.0;      // secondary reading text
  static const lg = 15.0;      // card/list titles, comment bodies
  static const xl = 16.0;      // body copy and buttons
  static const xxl = 18.0;     // section/screen titles
  static const display = 26.0; // large headlines
}
```

`FmtText`'s own roles are now built from these tokens rather than their own numbers, and every other hardcoded `fontSize` in the app was migrated to reference one of them too — the same relationship `FmtSpace`/`FmtRadius` already have to raw spacing/corner values, applied to type. A future one-off `Text` widget should reach for `FmtFontSize.*`, never a bare number.

## Spacing & radius scale

```dart
class FmtSpace { static const xs=4.0, sm=8.0, md=12.0, lg=16.0, xl=24.0, xxl=32.0; }
class FmtRadius { static const tile=13.0, card=14.0, input=8.0, pill=999.0; }
```

Applied via `Row`/`Column`/`Wrap` `spacing`/`runSpacing` (Flutter 3.27+) or explicit `SizedBox` gaps between siblings — never accumulated margins on individual children, for the same reason the web design system avoids margin-collapse ambiguity: one source of truth for the gap between two elements, not two children each contributing half.

## Component library (`core/design_system/widgets`)

| Widget                                        | Maps to mockup element                     | Key behavior                                                                                                                                                                                                                                                       |
| --------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `FmtPrimaryButton` / `FmtOutlineButton` | `.btn-primary` / `.btn-outline`            | Full-width by default, 50dp min height; pill/stadium shape (`FmtRadius.pill`), not `FmtRadius.input` — buttons and form controls deliberately use different radii, a structural cue borrowed from reviewing bluehive-project. Disabled state uses a desaturated fill, never just lowered opacity (opacity alone can fail contrast checks) |
| `StatusChip`                                  | `.chip-pending/-progress/-resolved/-alert` | Takes a `ReportStatus` enum, not a raw color — prevents a screen from inventing a new ad hoc status color                                                                                                                                                          |
| `VerifiedBadge`       | new — no mockup element yet                | Renders from `report.verifiedAt != null` only, never from `ReportStatus` — takes a separate slot alongside `StatusChip` on `ReportCard`/detail/`ReportFeedPost`, not a replacement for it (FR-19.3); absent entirely (not a greyed-out variant) when unverified, so it can't be mistaken for a "not verified yet" badge |
| `ReportFeedPost`      | new — no mockup element yet                | The home-feed "post" card (FR-20.1), styled like a Facebook post: header (round category "avatar", category name, timestamp, verified badge, status chip), description, `ReportMediaCarousel`, a divider, then an evenly-split action row (`SupportButton` + comment-to-detail button). Separate from `ReportCard` (My Reports' compact list style) — different information density for a different job, not a redesign of one into the other |
| `ReportMediaCarousel` | new — no mockup element yet                | Shared by `ReportFeedPost` and the detail screen so a report's media has one rendering path. Photos (FR-1.2, up to 5) page through a `PageView` with dot indicators + a "n/total" counter, Facebook-album style; a video renders as a static placeholder (no autoplay — FR-20.5 is future scope) |
| `SupportButton`       | new — no mockup element yet                | One-tap toggle for FR-17.1's "support" reaction — a feed-card affordance, not the full reactions row. Dispute (FR-17.3, requires a typed reason) deliberately isn't offered inline here; it stays on the detail screen's `_ReactionsRow` |
| `ReportCard`                                  | `.card` with `.stripe`                     | Severity-color left stripe is a required parameter, not optional styling — encodes state in form, not just the chip text, per accessibility redundancy (color is never the _only_ signal — see [Performance & Accessibility](11-performance-and-accessibility.md)) |
| `CategoryTile`                                | `.category-tile`                           | Visual tile can stay compact, but **minimum 48×48dp tap target** is enforced via `InkWell` hit-area padding regardless of visual icon size — see the touch-target note below                                                                                       |
| `MapPreview`                                  | `.map-box`                                 | Thin wrapper around the chosen map SDK widget (see [Device Capabilities](07-device-capabilities.md)), not a custom map renderer                                                                                                                                    |
| `Timeline` / `TimelineStep`                   | `.timeline`                                | Renders a `List<WorkflowStepStatus>` from the API's workflow state — presentation-only, no business logic about _which_ steps exist (that's server/workflow-engine-defined, per [Architecture — Workflow Engine](../docs/06-architecture.md#workflow-engine))      |
| `StepperDots`                                 | `.stepper-mini`                            | Multi-step form progress (category → capture → review)                                                                                                                                                                                                             |
| `SlaCountdownText`                            | `.sla`                                     | Ticking countdown using `mono` type; color escalates `blue → amber → red` as the SLA deadline approaches, thresholds sourced from the category's configured SLA, not hardcoded                                                                                     |

## Touch targets — a deliberate departure from the mockup's visual density

The wireframe's category grid (14 tiles in a compact 3-column layout) was sized for a static visual mockup. In the real app, **every interactive element gets a minimum 48×48dp hit area** (Material accessibility baseline, and the concrete implementation of the "Accessible" principle and WCAG 2.1 AA touch-target guidance in [NFR — Accessibility](../docs/05-non-functional-requirements.md#accessibility)), even where the visible icon/label is smaller — achieved via `InkWell`/`GestureDetector` padding, not by simply enlarging the visual tile (which would blow out the layout density the mockup intentionally used). This is the kind of gap that's easy to miss translating a static mockup into a real, tappable app — called out explicitly so it isn't.

## Icons

The mockup's inline SVG line-icon set (category icons, UI chrome icons) ports to Flutter as a small custom icon font or `CustomPainter`/vector-drawable set — not `Icons.*` Material defaults, which don't have matching glyphs for civic-specific concepts (flooding, illegal dumping, barangay pin). Recommend generating a custom icon font from the existing SVG set (via `fluttericon`/`icomoon`-style tooling) once the set is finalized, rather than hand-porting each SVG path into a `CustomPainter` — less code to maintain per icon, consistent sizing/baseline behavior for free.
