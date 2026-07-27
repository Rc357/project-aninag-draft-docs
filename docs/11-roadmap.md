# Roadmap

Phased from single-city MVP to nationwide platform. Each phase's exit criteria are deliberately outcome-based (real usage evidence), not date-based — building Phase N+1 against assumptions instead of Phase N's real pilot data is exactly the kind of premature complexity this documentation set otherwise argues against.

## Phase 0 — Foundations (Engineering, pre-pilot)

**Goal:** the architecture in this document set exists as working software, ready for one real city.

- Multi-tenant data model, Identity/RBAC, Workflow Engine core ([Architecture](06-architecture.md), [Database Design](07-database-design.md))
- Incident Reporting module configured as a Workflow Engine instance (FR-14)
- Citizen app (Flutter): submission, tracking, confirmation
- Barangay, Department, Worker, City Hall interfaces (MVP feature set per [PRD](03-prd.md))
- AI pipeline: validation, spam detection, duplicate detection ([AI Design](10-ai-design.md))
- Core security controls: tenant isolation, RBAC enforcement, audit logging ([Security](09-security.md))

**Exit criteria:** one internal/design-partner city can be fully configured (categories, barangays, departments) and a citizen report can traverse the entire FR-5–FR-8 lifecycle in a staging environment with realistic seed data.

## Phase 1 — Single-City Pilot

**Goal:** prove the product works with a real LGU, real citizens, real staff — validate the [PRD success metrics](03-prd.md#success-metrics) against live usage, not projections.

- Onboard one pilot City (real Barangay boundaries, real Departments, real staff accounts)
- Run a defined pilot period (recommend minimum 8–12 weeks — long enough to see steady-state usage patterns and at least one full SLA-cycle's worth of reports across all categories, not just launch-week novelty volume)
- Instrument and review: time-to-first-response, duplicate rate, confirmation rate, spam-reaching-queue rate, AI validation precision/recall against Barangay Staff overrides
- Tune AI thresholds (duplicate auto-merge, validation confidence) using real precision/recall data — deliberately not tuned aggressively pre-launch per [AI Design](10-ai-design.md#3-duplicate-detection)

**Exit criteria:** success metrics are met or a clear, evidence-based remediation plan exists for whichever aren't; pilot LGU stakeholders (Barangay Admin, Department Head, City Hall) confirm the workflow matches real operational needs.

## Phase 2 — Multi-City Expansion (same module)

**Goal:** prove the multi-tenancy thesis — onboarding city #2 (and #3+) should be materially faster than city #1, using only configuration.

- Onboard 2–5 additional cities, tracking the tenant-onboarding-time metric (target < 1 business day, FR-13.1) as the key validation signal
- Harden anything Phase 1 revealed as under-configurable (if a second city needed a code change to support a workflow variation, that's the signal to fix, not proceed past)
- Stand up the Super Administrator / platform-health tooling properly (Phase 0/1 can operate with lighter internal tooling; multi-tenant scale requires real cross-tenant operational visibility)

**Exit criteria:** a new city can be onboarded by a non-engineer (Super Administrator role) with zero source changes, within the target timeframe, twice in a row.

## Phase 3 — Second Module (prove the Workflow Engine thesis)

**Goal:** prove that a second government process can be added as configuration + module code, not a platform rewrite — this is the concrete test of the entire "OS for LGUs" positioning, not just a feature launch.

- Candidate first additional module: **Barangay Clearance** or **Business Permit** (recommended over higher-complexity candidates like Payments or Disaster Response — these have the simplest external dependencies (no payment processor integration, no emergency-services integration) while still being a genuinely different workflow shape (document review, fee assessment) from Incident Reporting, which is exactly what's needed to stress-test the Workflow Engine's generality)
- Validate the "add a module" checklist in [Architecture](06-architecture.md#cross-cutting-how-a-future-module-gets-added) against reality — if Identity, tenancy, or the Workflow Engine core needed changes to support it, that is the most important finding of this phase and should be treated as an architecture review trigger, not quietly patched around

**Exit criteria:** second module ships touching only its own module folder plus additive (non-breaking) Workflow Engine configuration — no core-platform code changes.

## Phase 4 — Platform Maturity & Scale Hardening

**Goal:** the platform is ready to sell and operate at the scale the founding brief envisions, with evidence behind every claim, not aspiration.

- Executive Dashboard, cross-tenant analytics, AI-driven risk scoring/heatmaps ([AI Design — Future](10-ai-design.md#7-risk-score-future-prediction-heatmaps-future)) — now justified by real historical data volume
- Formal SOC 2 / compliance certification work, if enterprise/national procurement requires it ([Security — Non-goals](09-security.md#explicit-non-goals-for-mvp-revisit-when-justified))
- Revisit database-per-tenant hybrid isolation if a specific large customer contractually requires it ([Architecture — Multi-Tenancy](06-architecture.md#multi-tenancy-strategy))
- Revisit microservice extraction for the AI pipeline and/or Notifications if their independent scaling needs have become real operational pain, not hypothetical

## Phase 5+ — Provincial / National Tiers `[FUTURE]`

**Goal:** the Provincial and National Administrator roles ([Vision & Strategy](02-vision-and-strategy.md#target-users)) go from reserved data-model placeholders to real, used functionality.

- Provincial Organization tenant aggregating multiple City tenants (read/policy scope, per the model established in Phase 0)
- Cross-LGU coordination features for disaster response/emergency management modules, if pursued
- National-level reporting/policy dashboards

**Note on sequencing:** Phase 5 is listed last deliberately. Every phase before it is what makes Phase 5 possible *without* an architecture rewrite — the entire point of the Organization → City → Barangay → Department model, the tenant-scoped RBAC, and the generic Workflow Engine established in Phase 0 was to make this phase a data/configuration/product exercise, not a re-platforming exercise. If Phase 5 turns out to require schema or auth-model changes, that is evidence a decision earlier in this document set needs revisiting — and should be traced back to the specific document/decision responsible, per the cross-references throughout this suite.
