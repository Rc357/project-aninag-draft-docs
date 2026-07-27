# AI Design

## Why Azure OpenAI + Azure AI Vision

- **Regional availability and enterprise data-handling terms** matter more here than raw model quality differences between providers: Azure's enterprise agreements provide contractual data-handling guarantees (no training on customer data, defined data residency options) that are easier to represent to a government procurement/legal reviewer than a consumer-grade API agreement — a material factor for a product whose customer is, ultimately, government.
- Consistency with the rest of the [Azure-based stack](06-architecture.md#technology-stack-and-rationale) (App Services, Blob Storage, Application Insights) reduces integration and IAM surface area — one cloud identity/networking boundary to secure, not two.
- Azure AI Vision provides pretrained image classification/moderation capability suitable for FR-2's validation and spam checks without training a custom model from zero, which would be unjustified engineering cost before there is a large labeled dataset of real citizen-submitted photos to train on.

## AI is an assistive triage layer, not a gate or an authority

This is the single governing principle behind every design choice below, restated from [Security — Preventing Spam & Abuse](09-security.md#preventing-spam--abuse): **AI outputs inform priority, routing, and human review — they never unilaterally and permanently reject a citizen's access to a government service (FR-2.5).** Every AI feature below is designed as a recommendation with a confidence score and a human override path, not a pass/fail gate.

## Feature-by-feature design

### 1. Image classification (validation)

- **Input:** submitted photo(s) + selected category.
- **Model:** Azure AI Vision image classification/tagging, compared against a category-to-expected-visual-tags mapping (e.g., category `Flooding` expects tags like water, street, flood).
- **Output:** confidence score. Below threshold → `NeedsReview` flag surfaced to Barangay Staff (FR-2.2), never auto-rejected.
- **Why not a custom-trained model at MVP:** no proprietary training set exists yet. Once the platform has accumulated a meaningful volume of Barangay-Staff-verified (photo, category, valid/invalid) pairs across pilot cities, a custom classifier fine-tuned on that ground truth becomes viable and higher-precision than a generic model — this is a deliberate `[FUTURE]` step gated on data volume, not on engineering readiness.

### 2. Spam detection

- **Signals combined:** perceptual image hashing (detect recycled/stock images across submissions), Azure AI Vision content moderation (irrelevant/inappropriate image detection), submission-rate signals from Redis (FR-2.3).
- **Design choice — heuristic + AI ensemble, not AI-only:** perceptual hashing is cheap, deterministic, and catches the single most common real-world spam pattern (same image resubmitted) without any model call; AI classification handles the harder "is this photo even plausibly related to a report" case. Using AI for everything would be slower and costlier for signal a simple hash catches for free.

### 3. Duplicate detection

- **Two signals, combined:** geospatial proximity (PostGIS radius query, FR-3.1) and visual similarity (image embedding comparison via Azure AI Vision) for reports of the same category within the time/radius window.
- **Two-threshold design** (FR-3.2): below the lower threshold, no action; between lower and upper threshold, surfaced to Barangay Staff as a candidate duplicate for a human decision; above the upper threshold, auto-merged. The upper (auto-merge) threshold is intentionally set conservatively high at MVP launch — an incorrectly auto-merged *distinct* incident (e.g., two separate potholes 30 meters apart) silently drops a real citizen report, which is a worse failure mode than asking a human to confirm an obvious duplicate. This threshold is expected to be tuned upward (more auto-merge) only after real precision/recall data from pilot usage justifies it.

### 4. Priority prediction & department recommendation

- Priority starts from the category's configured default (FR-11.3: `ReportCategory.default_priority`) and is adjusted by signals: image-classified severity (e.g., "large pothole blocking a lane" vs. "hairline crack"), proximity to sensitive locations (school, hospital — from a configurable points-of-interest layer), and duplicate-cluster size (many independent reports of the same issue is itself a priority signal).
- Department recommendation for MVP is primarily **deterministic** (Category × Barangay configuration mapping, FR-4.2) — AI's role here is limited to flagging ambiguous cases (a report that could plausibly belong to more than one department, e.g., a flooded road with a downed electric wire) for human routing rather than guessing, since a wrong deterministic route is at least predictable/debuggable, while a wrong AI-guessed route in an emergency-adjacent scenario (downed wires) is a genuine safety risk.

### 5. Automatic categorization & incident summarization

- **Automatic categorization** (suggesting a category from the photo before the citizen manually selects one) is a UX assist, always confirmable/overridable by the citizen before submission — never silently substituted.
- **Incident summarization** (AI-generated one-line summary of a report's free-text description, for staff scanning a dense queue) is generated once at verification time and cached, not regenerated live, to control Azure OpenAI cost at scale (see [Cost & Operational Considerations](#cost--operational-considerations)).

### 6. Citizen reputation

- A per-account score, adjusted by outcomes: verified-genuine reports increase it, reports rejected as spam/invalid decrease it, confirmed resolutions (FR-8.2) increase it modestly.
- **Explicitly not used to gate submission** at MVP — used only as a triage signal (a low-reputation account's reports get a lower default priority for AI-assist purposes, not blocked). Using reputation to block submissions would risk disproportionately silencing citizens who are simply new to the platform (zero history reads identically to bad history under a naive scoring scheme) — an equity failure mode worth explicitly designing against rather than discovering after launch.

### 7. Risk score, future prediction, heatmaps `[FUTURE]`

- Reserved for the Executive Dashboard module: aggregate risk scoring (e.g., flood-prone area prediction from historical report density + weather data), trend heatmaps. Explicitly deferred — these require a meaningful historical dataset the MVP has not yet generated, and building predictive features against synthetic/insufficient data would produce misleading outputs presented as authoritative to government decision-makers, which is a credibility risk worth avoiding by simply waiting for real data.

## Pipeline architecture

```
Report submitted
   │
   ▼
Report persisted (state: PendingAIValidation) ── citizen sees "Submitted" immediately
   │
   ▼ (async, queued)
AI Processing Worker:
   ├─ Azure AI Vision: classification + moderation
   ├─ Perceptual hash check (Redis-backed)
   ├─ PostGIS duplicate candidate query
   ├─ Azure AI Vision: embedding similarity vs. candidates
   └─ Azure OpenAI: summarization (verification-time, cached)
   │
   ▼
WorkflowInstance transitions (Validated / NeedsReview / SpamFlagged)
   │
   ▼
Routed to Barangay Queue (or exception queue)
```

The AI processing worker runs **asynchronously, decoupled from the submission request** — this is what delivers the NFR performance target (submission API responds without waiting on AI latency) and the graceful-degradation NFR (if Azure AI is unavailable, reports still queue as `PendingAIValidation` rather than failing submission). This decoupling is also the architectural seam noted in [Architecture — Modular Monolith](06-architecture.md#architecture-style) as the most likely first candidate for extraction into an independently-scaled service, since its load profile (bursty, latency-tolerant, potentially GPU/inference-bound if a custom model is added later) genuinely differs from the CRUD API's.

## Cost & operational considerations

- Azure OpenAI calls (summarization) are the highest per-call cost item — mitigated by running once per report at verification time and caching the result, not on every queue view.
- Azure AI Vision classification/moderation calls scale linearly with submission volume — tracked per-tenant in Application Insights (per [NFR — Observability](05-non-functional-requirements.md#observability)) both for cost attribution and as an input to future tenant pricing models.
- A circuit breaker around AI pipeline calls ensures a provider outage or rate-limit event degrades to the `PendingAIValidation` fallback path rather than cascading into API-tier failures.

## Model governance

- AI confidence thresholds (validation, duplicate detection) are tenant-configurable, not hardcoded, consistent with the platform's overall configurability philosophy ([Vision & Strategy](02-vision-and-strategy.md#product-philosophy--configurability-over-hardcoding)) — different cities may have different risk tolerances for auto-merge vs. human review, especially in a pilot phase where trust in AI-assisted government tooling is still being established.
- Every AI-driven decision that affects a report's routing/priority is logged as a `WorkflowEvent` with `actor_type = "ai"` and the model/confidence metadata attached (see [Database Design](07-database-design.md#3-workflow-engine-generic)) — this is what makes "why was my report deprioritized" answerable, both to a citizen and to an auditor, rather than an opaque black box.
