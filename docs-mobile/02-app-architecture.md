# App Architecture

## Project structure — one codebase, two app targets, not a monorepo

```
lib/
  apps/
    citizen/            # Citizen app entry point + app-shell only
      main.dart
    worker/              # Worker app entry point + app-shell only
      main.dart
  core/
    networking/          # Dio client, interceptors, error mapping
    storage/              # Drift database, secure token storage
    auth/                  # Session/token state, auth repository
    design_system/         # Tokens + shared widgets (see Design System doc)
    routing/               # go_router configuration per app
  features/
    incident_reporting/
      data/                # DTOs, API client calls, repository impl
      domain/               # Entities, repository interfaces, use-cases
      presentation/         # Screens, widgets, controllers/providers
    worker_jobs/
      data/ domain/ presentation/
    identity/
      data/ domain/ presentation/
    notifications/
      data/ domain/ presentation/
```

**Feature-first, mirroring the backend's module boundaries** ([Architecture — Modular Monolith](../docs/06-architecture.md#architecture-style)): `incident_reporting` on mobile talks to the `IncidentReporting` module on the backend, `identity` to `Identity`, and so on. When a new backend module (Permits, Appointments) ships, the mobile counterpart is a new `features/permits/` folder — not a restructuring of existing ones. This is the same "adding a module shouldn't touch what already works" principle the backend architecture is built around, applied to the client.

**Two entry points (`apps/citizen`, `apps/worker`), one Flutter project — not a melos monorepo of separate packages.** A monorepo with independently versioned packages is justified when packages have genuinely independent release cadences or ownership; at a two-person team building one MVP, that overhead (package boundaries, cross-package versioning, melos tooling) buys nothing yet. Flutter's build **flavors** (`--flavor citizen` / `--flavor worker`, targeting `apps/citizen/main.dart` / `apps/worker/main.dart`) achieve the same "two distinct installable apps" outcome with one `pubspec.yaml` and one CI pipeline. Revisit a package-based monorepo only if `core/` or `features/` genuinely need independent versioning against each app — not preemptively.

**Each app's entry point is thin.** `apps/citizen/main.dart` wires up the citizen app's theme, router, and root providers, then hands off entirely to `features/*/presentation`. It contains no business logic — its only job is composition, which keeps the two apps' divergence contained to "which features/screens are included," not duplicated logic.

## Layering within a feature (Clean Architecture, Flutter-flavored)

```
presentation/   → Widgets (screens, dialogs) + Riverpod controllers/providers
                  Depends on domain only — never touches data/ directly.
domain/          → Entities (plain Dart classes), repository interfaces,
                    use-cases (a single public method each, e.g. SubmitReportUseCase)
                    Zero Flutter/Dio/Drift imports — pure Dart, unit-testable
                    without a widget test harness or a mocked HTTP client.
data/            → DTOs (freezed + json_serializable), Dio API calls,
                    Drift table definitions, repository implementations
                    that satisfy domain's repository interfaces.
```

This mirrors the backend's own dependency rule ([Architecture](../docs/06-architecture.md#clean-architecture-layering-per-module)): inner layers (`domain`) never depend on outer layers (`data`, `presentation`). The payoff on mobile specifically: workflow rules that matter for correctness — e.g., "before/after photos are required before a job can be marked complete" (FR-6.4) — live in `domain/use-cases` as plain Dart, testable with a fast, non-widget unit test, rather than being buried in a widget's `onPressed` callback where it can only be exercised through a full widget test.

## State management — Riverpod

**Chosen over Bloc/Provider/plain `setState`.** The deciding factors:

- **Compile-safe DI without a separate service locator.** Riverpod's providers *are* the dependency graph — a `ReportRepository` provider can depend on the `DioClient` provider and the `AuthSession` provider, and the compiler catches a missing dependency. This removes the need for a separate tool like `get_it`.
- **Team fit.** The team's stronger background is React (per prior discussion on the web stack) — Riverpod's `provider`/`ref.watch` model is conceptually close to React's context + hooks pattern (a widget "subscribes" to a provider the way a component subscribes to a context value), which shortens the ramp-up compared to Bloc's more ceremony-heavy event/state/bloc triad.
- **Testability.** `ProviderContainer` overrides make it straightforward to substitute a fake repository in a widget test without a DI framework's test-mode ceremony.

**What this means concretely:** each feature's `presentation/` layer exposes Riverpod `Notifier`/`AsyncNotifier` classes (e.g., `ReportSubmissionController`) that call into `domain` use-cases and expose UI-consumable state (loading/data/error) — screens are largely declarative, reading provider state and dispatching intents, not holding business logic themselves.

## Navigation — go_router

Declarative routes matching the screen inventory in [Screens & Navigation](03-screens-and-navigation.md), one route table per app (citizen routes vs. worker routes), since the two apps never share a navigation stack. `go_router` is chosen over `Navigator` 1.0/imperative pushes and over `auto_route` because:

- It's Flutter-team-maintained, reducing third-party-package risk for a decision this load-bearing.
- Deep-link support is needed day one, not as an add-on: a push notification (FR-9) linking to a specific report's tracking screen, or a guest's "track this report" flow, both need to resolve a URL-like path (`/reports/:id`) to a screen — exactly go_router's design center.
- Route guards (e.g., redirect to sign-in if a Worker-app route is hit without a valid session) are declarative and centralized, not scattered per-screen `if` checks.

## Networking layer

- **Dio** as the HTTP client, wrapped in a single `core/networking` package — never instantiated ad hoc inside a feature.
- **Interceptor chain** (in order): auth-token attach → tenant-agnostic (mobile never sends `organizationId`, consistent with [Security — Tenant Isolation](../docs/09-security.md#tenant-isolation) — the token alone identifies the tenant) → idempotency-key injection for `POST /reports` (see [API Specification](../docs/08-api-specification.md#conventions)) → retry-with-backoff for network failures → 401 handler that attempts a silent token refresh once before surfacing an auth error.
- **Error mapping:** the backend's RFC 7807 `problem+json` error shape ([API Specification](../docs/08-api-specification.md#conventions)) is deserialized into a small set of typed Dart exceptions (`ValidationError`, `AuthError`, `NotFoundError`, `NetworkError`) at the `data` layer boundary — `domain` and `presentation` never parse raw error JSON, they pattern-match on typed exceptions.
- **DTOs vs. domain entities are distinct types**, mapped explicitly in the repository implementation. This is a deliberate, small amount of boilerplate: it means a backend API field rename only requires a change in one mapping function, not a hunt through every screen that happens to reference that field.

## Code generation

- **`freezed`** for immutable domain entities and DTOs (value equality, `copyWith`, union types for state like `AsyncValue`-style loading/data/error where Riverpod's built-in `AsyncValue` isn't already sufficient).
- **`json_serializable`** for DTO ↔ JSON, paired with `freezed`.
- Both are build-time codegen (`build_runner`), not reflection — keeps release binaries small and startup fast, relevant to the app-size/cold-start budgets in [Performance & Accessibility](11-performance-and-accessibility.md).

## What's deliberately not introduced yet

- **GraphQL client / code-generated API client (e.g., OpenAPI-generated Dio client)** — worth adding once the NestJS API's OpenAPI spec is stable; hand-written repository calls are fine at current API surface size and avoid a codegen pipeline dependency before the contract has stabilized.
- **A second navigation package for nested/shell routes beyond what go_router's `ShellRoute` covers** — not needed until a screen genuinely requires nested tab+stack navigation more complex than the flows in [Screens & Navigation](03-screens-and-navigation.md).
- **Dependency injection framework beyond Riverpod** (e.g., `get_it`, `injectable`) — Riverpod's provider graph already covers this; adding a second DI mechanism would just create two competing ways to wire the same objects.
