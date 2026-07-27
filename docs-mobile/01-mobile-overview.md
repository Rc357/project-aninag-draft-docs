# Mobile Overview

## What ships as "mobile"

Two distinct Flutter apps, both scoped to the Incident Reporting MVP ([PRD](../docs/03-prd.md)):

| App | Users | Core job |
|---|---|---|
| **Citizen app** | Guest Citizen, Verified Citizen | Report an issue, track it, confirm resolution |
| **Worker app** | Department Staff (Workers) | See assigned jobs, document before/after work, mark complete |

These are **two separate app targets built from one Flutter codebase** (shared packages for design system, networking, and models; separate `lib/apps/citizen` and `lib/apps/worker` entry points — see [App Architecture](02-app-architecture.md)). They are not one app with a role switch: a Worker's device should never be able to install or accidentally land on citizen-facing screens, and a citizen has no reason to carry Worker-app bulk (job queues, navigation tooling) on their phone. Splitting at the app-target level, while sharing everything below the presentation layer, gets both the separation and the code reuse.

Barangay Staff/Admin, Department Head, City Hall Administrator, and Super Administrator are **not** mobile users at MVP — those are the web dashboard's audience (separate doc set, not yet started). This doc set only covers the two apps above.

## Implementation status

The Citizen app has been scaffolded as a standalone Flutter project at [`../mobile-citizen/`](../mobile-citizen/README.md), built from `mobile-boilerplate-riverpod-v2` (a more production-oriented Riverpod + go_router starter than the earlier `mobile-boilerplate-riverpod` reference migration) — package `com.aninag.citizen`, display name "Aninag Citizen". See that project's `README.md` for structure/run instructions and `SETUP.md`-equivalent notes for what's still manual (iOS per-flavor schemes, real Firebase config).

**The full MVP flow is implemented and running**, against an in-memory data layer (no backend exists yet): Welcome (guest/sign-in) → Home (nearby feed) → new report (category → photo/GPS capture → review → submit) → Submitted confirmation → My Reports → Report detail/tracking with confirm/dispute → Track by ID (guest) → Profile. Camera and GPS capture are real device integrations (`image_picker`, `geolocator`); the map is a lightweight custom-painted placeholder rather than a real map SDK, since that needs a Google Maps API key nobody's provisioned yet — swap `MapPreview` for a real map widget once one is. The design system in [04](04-design-system.md) is implemented as actual `ThemeData`/widgets under `lib/app/{theme,widgets}`, using Material Icons rather than the custom icon font that doc recommends (a scope trim, not an oversight — revisit once the icon set is finalized). `flutter analyze` and `flutter test` both pass clean.

**This is a deliberate divergence from the plan below**, worth flagging rather than quietly ignoring: this section still describes "two app targets, one Flutter codebase" (`lib/apps/citizen` + `lib/apps/worker` sharing `core/`), but the Citizen app was actually built as its **own independent Flutter project** — a full standalone app, not a target within a shared monorepo package. This is simpler to stand up and matches the boilerplate as given, but means design-system tokens, networking, and domain patterns are currently *conventions to replicate* in a future Worker app project, not *code shared* by importing a common package. When the Worker app is built, this is the decision to revisit explicitly: extract shared pieces (design system, API client, auth) into a shared package the two projects both depend on, or accept the duplication because two small apps evolving independently is simpler for a two-person team than coordinating a shared package's versioning. Don't assume the original one-codebase plan silently still holds.

## Relationship to the platform docs

This mobile doc set assumes and does not re-derive:

- **What** the app does — see [Functional Requirements](../docs/04-functional-requirements.md) (FR-1 through FR-8 cover the citizen/worker-facing lifecycle).
- **What it's measured against** — see [Non-Functional Requirements](../docs/05-non-functional-requirements.md) (performance, offline, accessibility targets referenced throughout this set).
- **What API it talks to** — see [API Specification](../docs/08-api-specification.md); [API Integration](06-api-integration.md) here covers the *client-side* implementation of that contract, not the contract itself.
- **The backend/tenancy model** — see [Architecture](../docs/06-architecture.md); mobile is a pure client of the NestJS API and holds no tenant-scoping logic of its own (see [Security — Tenant Isolation](../docs/09-security.md#tenant-isolation)).

If something here seems to duplicate the main docs, it shouldn't — flag it for a fix rather than let two documents drift.

## Tech stack (mobile-specific recap)

| Concern | Choice | Reference |
|---|---|---|
| Framework | Flutter (Dart) | [Architecture — Technology Stack](../docs/06-architecture.md#technology-stack-and-rationale) |
| Backend consumed | NestJS REST API | [API Specification](../docs/08-api-specification.md) |
| Push notifications | Firebase Cloud Messaging | [Device Capabilities](07-device-capabilities.md) |
| Local persistence | Drift (SQLite) for structured offline data, `flutter_secure_storage` for tokens | [State & Data Layer](05-state-and-data-layer.md) |
| Maps/geolocation | `geolocator` + a map SDK (Google Maps Flutter, given Philippine map data coverage) | [Device Capabilities](07-device-capabilities.md) |
| Crash/analytics | Firebase Crashlytics + Firebase Analytics | [Analytics & Crash Reporting](12-analytics-and-crash-reporting.md) |

## Explicit non-goals for this doc set

- **Web dashboard implementation** — separate documentation, separate codebase target, not covered here even though it may eventually share the design-token source with [Design System](04-design-system.md).
- **Backend implementation detail** — covered in the main `../docs/` set, not duplicated here.
- **Wireframe visual design itself** — already produced as the [Aninag Mobile UI Concepts artifact](https://claude.ai/code/artifact/e745faa6-fbdc-46c4-ab0d-0a7b8461c9d5); this doc set describes how to *build* that concept, not re-justify its design choices.
