# Build, Release & CI/CD

## Flavors

Three environments (dev, staging, prod — matching backend environments) × two apps (Citizen, Worker) = six build variants, using Flutter's native flavor mechanism, not manual config-file swapping:

```
flutter build apk --flavor citizenDev    --target lib/apps/citizen/main.dart
flutter build apk --flavor citizenProd   --target lib/apps/citizen/main.dart
flutter build apk --flavor workerDev     --target lib/apps/worker/main.dart
flutter build apk --flavor workerProd    --target lib/apps/worker/main.dart
... (staging variants analogous)
```

Each flavor gets a distinct app ID/bundle ID suffix (e.g., `ph.obserba.citizen.dev` vs `ph.obserba.citizen`) so dev, staging, and prod builds can be installed side-by-side on the same test device — a real practical need during pilot testing, not a nicety.

## Signing

- **Android:** a single upload keystore per app (Citizen, Worker), generated once and stored as a CI secret (GitHub Actions encrypted secret), never committed to the repository. Google Play App Signing is enabled so Google holds the actual distribution signing key — the upload keystore is only used to authenticate uploads, which limits the blast radius if the CI secret were ever compromised (rotate the upload key without losing the app's identity in-market).
- **iOS:** managed via Fastlane `match` (certificates and provisioning profiles stored encrypted in a private repo, synced across the team and CI) rather than manually-exported `.p12`/`.mobileprovision` files passed around ad hoc — this is the standard, low-friction way to avoid "it only builds on my machine" signing failures, especially relevant since Apple certificates expire and need coordinated renewal.

## CI pipeline (GitHub Actions)

Matches the platform-wide CI/CD choice in [Architecture — Technology Stack](../docs/06-architecture.md#technology-stack-and-rationale):

| Trigger | Jobs |
|---|---|
| Every pull request | `flutter analyze`, unit + widget + golden tests (see [Testing Strategy](09-testing-strategy.md)), build (not sign) both flavors' debug APKs to catch build-breaking errors early |
| Merge to `main` | Same as above, plus a signed **staging** build of both apps uploaded to a distribution channel (Firebase App Distribution) for internal/pilot-partner testing |
| Tagged release | Signed **prod** build submitted to Play Console (internal testing track first, promoted manually) and TestFlight/App Store Connect |

**Firebase App Distribution for staging builds** rather than requiring testers to sideload APKs or wait on a Play Store internal-testing propagation delay — pilot-city staff and internal testers get a build within minutes of a merge, which matters during the pilot phase in [Roadmap — Phase 1](../docs/11-roadmap.md#phase-1--single-city-pilot) where fast feedback loops on real usage are the whole point.

## Versioning

Semantic version (`MAJOR.MINOR.PATCH`) plus a monotonically increasing build number, bumped automatically by CI (not hand-edited in `pubspec.yaml` per release) to guarantee Play Console/App Store Connect never reject a build for a duplicate version code. Citizen and Worker apps version independently — they are different apps to their respective stores and there's no reason to force them to share a version number just because they share a codebase.

## Store submission process

- **Android:** internal testing track → closed testing (pilot partner staff) → production, following Play Console's staged rollout percentages for the first production release rather than 100% on day one — a real incident-reporting app's first production release deserves a slow rollout, not a full blast.
- **iOS:** TestFlight (internal, then external testers capped at the pilot group) → App Store review → release. Apple's review timeline (typically 24–48h, sometimes longer) should be budgeted into the [Roadmap](../docs/11-roadmap.md) pilot timeline explicitly — it's an external dependency outside engineering's control.
- **Government/LGU app store listing considerations:** the Worker app, being staff-only with no public signup, may be a better candidate for a private/internal distribution mechanism (Play Console's internal app sharing, or an enterprise/ad hoc iOS distribution) rather than a public store listing at all — worth deciding per pilot city rather than defaulting both apps to public listings.

## Over-the-air updates — not adopted for MVP

Tools like Shorebird exist for Flutter code-push-style OTA updates without a full store review cycle. **Not adopted now:** it's an additional paid service and operational dependency for a two-person team already managing a from-scratch stack, and the store review turnaround above is acceptable for MVP/pilot cadence. Revisit if a specific pilot incident (e.g., a critical bug needing same-day fix while an App Store review is pending) makes the review-cycle latency a genuine operational risk rather than a hypothetical one.
