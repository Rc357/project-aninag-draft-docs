# Project FixMyTown — Mobile Development Documentation

This folder documents the **mobile surfaces only**: the Flutter **Citizen app** and **Worker app** defined in the platform [PRD](../docs/03-prd.md). It is deliberately separate from [`../docs/`](../docs/README.md), which covers the whole-platform product/architecture story (backend, database, security, AI). Web/admin dashboard development will get its own doc set alongside this one once that stack work starts.

Read this set if you are building, reviewing, or onboarding onto the mobile client codebase. Read `../docs/` for anything about the backend API, data model, or platform-level product decisions this mobile app consumes but doesn't own.

## Reading order

| # | Document | Purpose |
|---|----------|---------|
| 1 | [Mobile Overview](01-mobile-overview.md) | Scope, the two apps, tech stack recap, how this relates to the main docs |
| 2 | [App Architecture](02-app-architecture.md) | Flutter project structure, layering, state management, DI |
| 3 | [Screens & Navigation](03-screens-and-navigation.md) | Full screen inventory and navigation map for both apps |
| 4 | [Design System](04-design-system.md) | Color/type/spacing tokens and reusable widget library, translated from the UI concepts into Flutter theme code |
| 5 | [State & Data Layer](05-state-and-data-layer.md) | State management patterns, local persistence, caching |
| 6 | [API Integration](06-api-integration.md) | How the app talks to the NestJS backend: auth, HTTP client, error handling, idempotency |
| 7 | [Device Capabilities](07-device-capabilities.md) | Camera, GPS, permissions, push notifications per platform |
| 8 | [Offline Support](08-offline-support.md) | Draft queueing, background sync, conflict handling |
| 9 | [Testing Strategy](09-testing-strategy.md) | Unit, widget, integration, and golden tests; device matrix |
| 10 | [Build, Release & CI/CD](10-build-release-cicd.md) | Flavors, signing, store submission, GitHub Actions pipeline |
| 11 | [Performance & Accessibility](11-performance-and-accessibility.md) | Cold-start/app-size budgets, low-end device targets, mobile accessibility |
| 12 | [Analytics & Crash Reporting](12-analytics-and-crash-reporting.md) | Crashlytics, product analytics, consent handling |
| 13 | [Mobile Roadmap](13-roadmap-mobile.md) | Mobile-specific build phases, mapped to the platform roadmap |

## Implementation

The Citizen app's code lives at [`../mobile-citizen/`](../mobile-citizen/README.md) — see [Mobile Overview — Implementation status](01-mobile-overview.md#implementation-status) for how it relates (and where it diverges) from the architecture documented below.

## Reference artifact

The high-fidelity screen concepts referenced throughout this doc set (especially [Screens & Navigation](03-screens-and-navigation.md)) were published as a standalone visual reference: **[Aninag — Mobile UI Concepts](https://claude.ai/code/artifact/e745faa6-fbdc-46c4-ab0d-0a7b8461c9d5)**. Treat that artifact as the visual source of truth these documents describe in writing.

## Source

Traces back to the founding brief ([`initial-project-fixmytown.md`](../initial-project-fixmytown.md)) and the platform documentation suite in [`../docs/`](../docs/README.md), particularly [PRD](../docs/03-prd.md), [Functional Requirements](../docs/04-functional-requirements.md), [Non-Functional Requirements](../docs/05-non-functional-requirements.md), [Architecture](../docs/06-architecture.md), and [API Specification](../docs/08-api-specification.md).
