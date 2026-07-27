# API Integration

Implements the client side of the contract defined in [API Specification](../docs/08-api-specification.md). This document does not redefine any endpoint — it covers how the Flutter apps call them correctly.

## Environment configuration

Base URL, and any environment-specific keys, are compiled per build flavor (dev/staging/prod — see [Build, Release & CI/CD](10-build-release-cicd.md)), never hardcoded or switched at runtime by a settings toggle. A citizen app pointed at the wrong environment by a stray debug menu is a real risk for a product handling real incident reports; environment is a build-time decision, not a runtime one.

## Auth flow (client side)

1. Sign-in (`/auth/google`, `/auth/apple`, `/auth/email/request-otp` + verify, `/auth/phone/request-otp` + verify, or `/auth/guest`) returns an access token + refresh token.
2. Access token → `flutter_secure_storage`, attached via the auth interceptor as `Authorization: Bearer <token>` on every request.
3. On a `401`, the interceptor attempts exactly one silent `/auth/refresh` call, replays the original request once on success, and — on failure — clears the session and routes to sign-in (per the route guard rules in [Screens & Navigation](03-screens-and-navigation.md#route-guard-rules-both-apps)). It never silently retries more than once; a second consecutive `401` is a real sign-out, not a retry loop.
4. Guest sessions follow the same interceptor path with a guest-scoped token — the client does not special-case guest requests, since the backend already scopes what a guest token can do ([Security — Authentication](../docs/09-security.md#authentication)).

## Idempotency for report submission

Per [API Specification — Conventions](../docs/08-api-specification.md#conventions), `POST /reports` requires an `Idempotency-Key`. The client generates a UUID v4 **once, when the draft is first created** (not at send time) and persists it alongside the draft in Drift (see [State & Data Layer](05-state-and-data-layer.md#local-persistence)). Every submission attempt for that draft — including retries after a dropped connection or an app restart before the original request confirmed — reuses the same key. Generating a fresh key per retry would defeat the entire point of the header: the backend can only deduplicate a retried submission from a genuinely new one if the key is stable across retries of the *same* citizen intent.

## Error handling

The backend returns RFC 7807 `problem+json` on failure ([API Specification](../docs/08-api-specification.md#sample-error-response-validation-failure)). The networking layer maps this to a small closed set of typed errors before it ever reaches `domain`/`presentation`:

| API condition | Client type | Typical handling |
|---|---|---|
| `422` with `errors[]` | `ValidationError(fields)` | Inline field errors on the form (e.g., "GPS location is required") |
| `401` (after refresh already attempted) | `AuthError` | Route to sign-in |
| `403` | `ForbiddenError` | Generic "not available" screen (e.g., a Worker hitting a job not assigned to them) |
| `404` | `NotFoundError` | "This report/job could not be found" state, not a crash |
| Network failure (no response) | `NetworkError` | Offline-aware handling — see [Offline Support](08-offline-support.md); never shown as if it were a validation failure |
| `5xx` | `ServerError` | Generic retry-able error state with a "Try again" action |

No screen parses `problem+json` directly — if a new error shape is ever needed, it's added once to this mapping layer, not hunted down across every screen that calls an endpoint.

## Photo upload

Photos are sent as `multipart/form-data` alongside the report/job-completion payload (per [API Specification](../docs/08-api-specification.md#sample-submit-report)), not as a separate pre-upload-then-reference step — simpler for MVP volume, and it means a report's photo and its metadata succeed or fail together rather than needing to reconcile a partially-uploaded state. Large-file/background-upload handling for flaky connections is covered in [Offline Support](08-offline-support.md#upload-retry), since it's really the same problem as offline report queueing.

## Pagination

List endpoints (`/reports?mine=true`, `/barangays/{id}/queue` equivalents on the worker side) use cursor pagination per the API spec. The client's list controllers (`AsyncNotifier`s, see [State & Data Layer](05-state-and-data-layer.md#provider-patterns-riverpod)) hold the next cursor as controller state and append pages on scroll-to-bottom — never re-fetch-from-page-1 on every load, which would both waste bandwidth (a real cost on the mobile data plans much of the target user base is on) and risk showing a citizen a shifted list mid-scroll as new reports arrive.

## Request logging

Verbose request/response logging (via a Dio `LogInterceptor`) is enabled only in debug/staging builds, gated by the same build-flavor mechanism as the base URL — never compiled into a production build, since request logs can contain citizen PII (phone numbers, precise GPS) that has no reason to sit in a production device's logs or crash-report breadcrumbs (see [Analytics & Crash Reporting — Consent & PII](12-analytics-and-crash-reporting.md#pii-handling)).
