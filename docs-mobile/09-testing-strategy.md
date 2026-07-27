# Testing Strategy

Mirrors the platform-level principle already established in [NFR — Maintainability](../docs/05-non-functional-requirements.md#maintainability): prioritize coverage on logic that causes real incidents if wrong, not on exhaustively testing every widget.

## Test pyramid

| Layer | Tool | What it covers | Priority |
|---|---|---|---|
| Unit tests | `package:test` | `domain/` use-cases and repository interfaces (mocked) — e.g., "a job cannot transition to Completed without at least one after-photo" | **Highest** — pure Dart, no widget harness, fastest to write and run, and this is exactly the kind of rule (mirrors FR-6.4) whose failure mode is a real workflow-integrity bug, not a cosmetic one |
| Widget tests | `flutter_test` + `ProviderContainer` overrides | Individual screens/components in isolation, with fake repositories | **High** for the report-submission and job-completion flows specifically (the two flows with hard evidence/validation gates); **medium** elsewhere |
| Golden tests | `golden_toolkit` | Design-system components ([Design System](04-design-system.md)) — `StatusChip`, `ReportCard`, `CategoryTile`, buttons | **Medium-high** — cheap to maintain given the app commits to one fixed theme (no light/dark matrix to double the golden set), and catches accidental visual regressions in shared components that would otherwise silently affect every screen using them |
| Integration tests | `integration_test` package | Full flows end-to-end against a staging backend: submit a report start-to-finish, complete a job start-to-finish | **Medium** — expensive to write and run, reserved for the one or two flows whose end-to-end correctness is the actual product (report lifecycle, job lifecycle), not every navigation path |

## What's deliberately not exhaustively tested

- **Every screen's pixel-perfect layout** across every device size — golden tests cover shared *components*, not full-screen snapshots for all 13+ screens × every device size, which would be high-maintenance for low signal (most layout bugs are caught faster by golden component tests plus manual device checks during development).
- **Third-party plugin internals** (`geolocator`, `image_picker`, FCM) — trust the plugin, test the app's *use* of it (e.g., "if geolocator throws, the capture screen shows an explanatory state," per [Device Capabilities](07-device-capabilities.md#location--foreground-only-when-in-use)), not the plugin's own correctness.
- **Offline queue timing edge cases beyond the core retry/backoff logic** — covered functionally (queue → retry → success/fail states), not exhaustively fuzzed against every possible connectivity flake pattern, which is disproportionate effort for MVP.

## Device testing matrix

Given [NFR — Performance](../docs/05-non-functional-requirements.md#performance) explicitly targets low-end Android devices common in the target market, manual/CI device testing should include, at minimum:

- One low-end Android device or emulator profile (2GB RAM class) — this is where cold-start and camera/GPS responsiveness problems actually surface, not on a developer's high-end phone.
- One recent mid-range Android device (the realistic median citizen device).
- One recent iOS device/simulator (iOS share is smaller in-market but Barangay/City Hall staff procurement sometimes skews iOS — worth covering, not skipping).

## CI integration

Unit, widget, and golden tests run on every pull request (see [Build, Release & CI/CD](10-build-release-cicd.md#ci-pipeline)); integration tests run on a slower schedule (e.g., pre-release, not every commit) given their cost — a PR shouldn't be blocked on a full device-farm run for a one-line copy change, but a release build should never ship without the end-to-end flows having passed recently.
