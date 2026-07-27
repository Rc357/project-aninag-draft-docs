# State & Data Layer

## Provider patterns (Riverpod)

Three recurring shapes cover nearly every screen — picking the right one per case keeps controllers predictable to read across the codebase:

| Pattern | Use for | Example |
|---|---|---|
| `Provider` (plain, no state) | Wiring dependencies (repositories, the Dio client, the current session) | `reportRepositoryProvider` |
| `AsyncNotifierProvider` | Any state backed by an API call with loading/data/error | `MyReportsController`, `JobQueueController` |
| `NotifierProvider` (sync) | Local-only UI state with no network round trip | `NewReportDraftController` (holds in-progress category/photo/location before submission) |

**Rule of thumb:** if a screen's state can be "stale," it's an `AsyncNotifier` wrapping a repository call, not a `Notifier` a widget populates manually — this keeps loading/error handling (retry, offline fallback) implemented once in the controller base pattern rather than reinvented per screen.

## Repository layer

Every feature's `domain/` defines a repository **interface**; `data/` provides the implementation. A repository method's return type is a small sealed `Result<T>` (via `freezed`'s union support) rather than throwing raw exceptions across the domain/presentation boundary — `Success<T> | Failure(AppError)`. This is a deliberate, low-boilerplate discipline: a controller calling `reportRepository.submit(draft)` is forced by the type system to handle the failure case, rather than an uncaught exception surfacing as a generic crash screen deep in the widget tree.

```dart
abstract class ReportRepository {
  Future<Result<ReportSummary>> submit(NewReportDraft draft);
  Future<Result<List<ReportSummary>>> myReports();
  Future<Result<ReportDetail>> detail(String reportId);
  Future<Result<void>> confirmResolved(String reportId);
}
```

## Local persistence

| Data | Storage | Why |
|---|---|---|
| Auth tokens (access + refresh) | `flutter_secure_storage` | Platform keychain/keystore-backed — never plain `SharedPreferences` for anything credential-shaped |
| Draft reports awaiting submission (offline) | Drift (SQLite) | Structured, queryable, and durable across app restarts — a `SharedPreferences` JSON blob doesn't scale past one draft and can't represent a queue; see [Offline Support](08-offline-support.md) |
| Cached "My Reports" / "Assigned Jobs" list (last known good) | Drift | Lets the list screen render instantly from cache while a fresh fetch happens in the background, and still render *something* if the fetch fails offline |
| Photo files pending upload | App-sandboxed filesystem (`path_provider`), referenced by path from Drift | Binary data doesn't belong in a SQLite row; the DB row stores the file path + upload status |
| User preferences (non-sensitive: last-selected category, notification opt-in) | `SharedPreferences` | Genuinely simple key-value, no query needs — using Drift here would be over-engineering |

**Why Drift over Hive or plain `sqflite`:** Drift gives compile-time-checked queries (a typo'd column name is a build error, not a runtime crash) and straightforward migrations as the offline schema evolves — both matter more here than Hive's slightly simpler API, given the offline queue's correctness (a dropped or duplicated citizen report is a real trust failure, not just a bug) is worth the marginally steeper learning curve.

## Cache strategy — cache-then-network, not cache-only or network-only

List screens (My Reports, Assigned Jobs) render the last cached result immediately, then trigger a background fetch and update in place if the result differs — never a blocking spinner on every screen visit for data that was probably still valid. Detail screens (`/reports/:id`) fetch fresh on entry (status can change from a push notification the user just tapped) but fall back to cache with a visible "showing last known status" indicator if offline, rather than an error screen — consistent with the platform's NFR that citizen access degrades gracefully rather than blocking (see [NFR — Availability](../docs/05-non-functional-requirements.md#availability)).

## What state does *not* live on the client

Workflow state (what stage a report or job is in, what transitions are valid next) is never computed or cached as *authoritative* on the client — the client renders whatever the API's `WorkflowInstance.current_state` says ([Database Design — Workflow Engine](../docs/07-database-design.md#3-workflow-engine-generic)) and never derives "the report must now be Assigned" from local logic. This matters specifically because workflow definitions are tenant-configurable and versioned server-side ([Architecture — Workflow Engine](../docs/06-architecture.md#workflow-engine)) — any client-side assumption about what states/transitions exist would break the moment a tenant's configuration differs from what the app was built against.
