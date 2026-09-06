# Device Capabilities

## Camera — capture only, no gallery picker

**Decision:** photo/video capture (citizen report media, FR-1.2/FR-20.2; Worker before/after photos, FR-6.4) uses `image_picker` restricted to `ImageSource.camera` — the OS gallery is never offered as a source, on either screen.

This is a deliberate anti-spam and evidence-integrity choice, not an oversight: [AI Design — Spam Detection](../docs/10-ai-design.md#2-spam-detection) already treats recycled/stock images as a known abuse pattern, and [Security — Preventing Spam & Abuse](../docs/09-security.md#preventing-spam--abuse) lists photo capture as the first line of defense. Forcing live capture closes that gap at the source instead of relying entirely on server-side perceptual-hash detection after the fact. The same logic applies even more directly to Worker before/after photos — those are evidence a job was actually done, and allowing a gallery-sourced "after" photo would undermine the entire before/after evidence model this workflow is designed around.

**EXIF handling:** immediately after capture, the `image` package strips all EXIF metadata (device identifiers, embedded GPS) before the file is queued for upload — consistent with [Security — Data Protection](../docs/09-security.md#data-protection), which treats the app's own explicit GPS capture (below), not photo EXIF, as the authoritative location source. The photo is evidence of the issue; it is never trusted as evidence of *where*.

**Video (FR-20.2):** an alternative to photo, not an addition — a report has one or the other, never both. Recorded via `image_picker`'s camera source with a 3-minute `maxDuration`, then compressed on-device before upload (`flutter_compress`'s `forSocialMedia` preset: 1080p cap, H.264) — see [Functional Requirements — FR-20.3](../docs/04-functional-requirements.md#fr-20--feed-visibility--media) for why compression happens client-side rather than server-side. Recording video captures an audio track, which is what actually requires the microphone permission below — there is no separate "record without audio" option, since stripping audio client-side after the fact would cost the same compression pass anyway.

## Location — foreground-only, "when in use"

`geolocator` provides GPS coordinates at the two points that need them: report submission (FR-1.3) and Worker job-transition timestamps (FR-6.5). The app requests **"When In Use" location permission only** — never "Always"/background location:

- The product has no feature that needs location while the app isn't open (no live worker tracking, no geofencing at MVP).
- Background location is both a real user-trust cost and a materially harder App Store/Play Store review process (both platforms require specific justification for background location) — requesting it without a corresponding feature would be pure cost for no benefit.

If a future module (e.g., real-time worker dispatch/ETA) genuinely needs background location, that's a deliberate, separately-justified permission upgrade at that time, not something to request speculatively now.

**Permission denial handling:** since GPS is mandatory for both report submission and job-transitions (FR-1.3, FR-6.5), a denial doesn't just block silently — the app shows a short explanation of *why* it's needed (matching the plain-language copy guidance: name what the user is doing, not what the system needs) with a direct link to the OS settings screen, since neither citizen trust nor a stuck Worker mid-job is served by a dead end.

## Permission manifest reference

| Platform | Permission | Used for |
|---|---|---|
| iOS (`Info.plist`) | `NSCameraUsageDescription` | Report/job photo/video capture |
| iOS | `NSMicrophoneUsageDescription` | Video's audio track (FR-20.2) |
| iOS | `NSLocationWhenInUseUsageDescription` | Report location, job transition location |
| iOS | (no `NSPhotoLibraryUsageDescription`) | Not requested — gallery access is intentionally never used |
| Android (`AndroidManifest.xml`) | `CAMERA` | Report/job photo/video capture |
| Android | `RECORD_AUDIO` | Video's audio track (FR-20.2) |
| Android | `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION` | GPS capture |
| Android | `POST_NOTIFICATIONS` (API 33+) | Push notifications (see below) |
| Both | Internet/network state | Standard API access |

**Just-in-time requests, not upfront:** every permission above is requested at the point the feature is first used (e.g., camera permission when the citizen reaches the photo-capture step of a new report), not all at once on first app launch. This is both better UX practice and yields higher grant rates than a wall of permission dialogs before the user has seen any value.

## Push notifications (Firebase Cloud Messaging)

- **Token-based, not topic-based**, at MVP: every notification (FR-9) is targeted at a specific report or job belonging to a specific user, so per-device FCM tokens registered against the user's account are the right primitive. Topic-based broadcast (e.g., a future "city-wide announcement" module) is a `[FUTURE]` addition layered on top, not a reason to complicate the MVP targeting model now.
- **Foreground display** needs `flutter_local_notifications` alongside FCM — FCM alone does not reliably surface a visible banner while the app is in the foreground on both platforms, so the client displays a local notification when a foreground FCM message arrives.
- **Deep-link payload:** every push carries a data payload with a route path (e.g. `"route": "/reports/9a12..."`), which the notification-tap handler feeds directly into `go_router` (see [Screens & Navigation — Route Guard Rules](03-screens-and-navigation.md#route-guard-rules-both-apps)) rather than the app having to re-derive which screen a given report/job belongs to.
- **Permission timing (iOS explicit prompt, Android 13+ runtime permission):** requested right after a citizen's first successful report submission — the moment "we'll tell you when this updates" is self-evidently valuable — not at cold app launch, where the value proposition isn't yet visible and denial rates are highest.

## Maps

**Google Maps Flutter plugin**, chosen primarily for Philippine map-data coverage and label quality over alternatives (Mapbox, Apple Maps-only) that are weaker in this specific region. Two things worth tracking as real, ongoing costs, not one-time setup:

- **API key provisioning** per platform (iOS/Android) and per environment (dev/staging/prod, see [Build, Release & CI/CD](10-build-release-cicd.md)) — keys should be restricted (by bundle ID/package name and by API) at creation, not left unrestricted.
- **Usage-based billing.** Given [there's no AI budget allocated yet](../docs/10-ai-design.md), Maps API cost is worth the same scrutiny: prefer a lightweight static map image (a single Static Maps API call) for small, non-interactive previews (e.g., a thumbnail on a report card) and reserve the full interactive `GoogleMap` widget for screens where panning/zooming is actually needed (capture-step pin adjustment, job navigation) — not every map-shaped element in the mockup needs to be a live interactive map instance.
