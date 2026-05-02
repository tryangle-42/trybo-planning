# Agent Routing & Allocation Control Plane

> Module mapping: M4 (Skill Ranking & Selection — AgentRank + AgentAllot)
> Status: canonical reference for AgentRank and AgentAllot design
> Research basis: RouteLLM (ICLR 2025), AgentRank-UC, CoCoMaMa, CLASSic, NetMCP, Unify.ai routing benchmarks, contextual bandits literature

---

## Why "Control Plane", not "Ranker"

It is tempting to frame M4 as *AgentRank + AgentAllot* — a two-phase decision (score, then pick). That framing is correct but too narrow. Production routing for a multi-tenant agent system is not just ranking; it is a **control plane** with at least eight responsibilities:

1. **Candidate generation** — which providers can fulfill this capability?
2. **Constraint filtering** — which are *legally and operationally* eligible right now?
3. **Scoring** — among the eligible, who is best for this request?
4. **Allocation** — primary + fallback chain, with traffic-shaping rules
5. **Fallback execution** — what to try when the primary fails mid-flight
6. **Exploration** — guarded sampling of underused providers
7. **Outcome learning** — telemetry feedback that updates future decisions
8. **Cost & SLA control** — budget caps, latency SLAs, compliance routing

Calling this "AgentRank" hides 60% of the work. Renaming it **Agent Routing & Allocation Control Plane (ARAC)** forces the design to address all eight from the start, even if most are stubs at baseline.

---

## Position vs M2 (Skill Management)

The line between M2 and M4 must be drawn carefully, because M2's retrieval pipeline already does basic ranking as a side effect.

> **M2 answers:** "Which skills are relevant to this request, and which ones are healthy?"
> **ARAC answers:** "Among healthy, relevant skills, which one is *optimal for this specific request context*, and what should the system try next if it fails?"

M2 is skill-aware. ARAC is context-aware *and* policy-aware *and* strategy-aware.

### What M2 already covers (no ARAC needed for it)

When a user says "call John":

1. Semantic-router narrows to `communication > voice` namespace — this is the capability grouping.
2. Bi-encoder retrieval finds `twilio.call.initiate`, `knowlarity.call.initiate` — these are the competing providers.
3. The retrieval query filters by `health_status = 'active'` — degradation filtering.
4. Results are sorted by `success_rate DESC` (or a weighted similarity + health combo) — basic ranking.
5. Top result = best provider; the rest form an implicit fallback chain.

At 2–3 providers per capability, this is sufficient. The skill retrieval pipeline with health-metric sorting **is** basic ranking.

### What ARAC holds separate (the genuine delta)

ARAC becomes its own module when ranking requires logic that **cannot live inside a retrieval query**:

| Capability | Why it can't be a sort clause in M2 | Example |
|-----------|--------------------------------------|---------|
| **Context-dependent weights** | Retrieval has no concept of the *request's* context affecting sort order. M2 finds skills similar to the query — it doesn't know if this is a budget trip or a premium trip. ARAC takes request-level context (urgency, budget, language) and adjusts weights per request. | "Budget trip to Goa" → weight cost heavily → pick Serper ($0.001/call). "Business client research" → weight quality heavily → pick Perplexity ($0.005/call). Same capability, different winner. |
| **Exploration** | Retrieval always returns the current best. It never deliberately picks a suboptimal provider to test if it has improved. ARAC maintains a probability distribution per provider and occasionally samples the #2 to discover improvements. | Tavily was slow last month, so M2 always ranks it last. But Tavily quietly upgraded their infra. Without exploration, the system never discovers this. ARAC routes ~10% of low-risk traffic to Tavily and detects the improvement. |
| **Abstract capability planning** | Today the planner picks concrete skills (`twilio.call.initiate`). ARAC enables the planner to think in abstract capabilities (`telephony.outbound`) and defer provider resolution to execution time. The plan stays valid even if providers change. | Plan created Monday with `twilio.call`. By Wednesday, Twilio is degraded. Without ARAC, the plan fails or needs re-planning. With ARAC, the plan says `telephony.outbound` and the allocator picks Knowlarity at execution time. |
| **Cross-request learning** | M2 tracks per-skill health (did this skill succeed?). ARAC tracks per-context outcomes (does calling this contact at 10am work better than 3pm? does this user prefer Perplexity over Tavily?). Behavioural learning across requests, not single-invocation telemetry. | "Calling Sharma at 10am → answered 8/10. At 3pm → answered 2/10." M2 doesn't track this. ARAC's competence graph does. |
| **Composite scoring formula** | M2 can sort by `success_rate` *or* `latency` — one dimension at a time. At 5+ providers, a weighted formula is required: `0.4·success + 0.25·speed + 0.2·cost + 0.1·context + 0.05·consent_overhead`. The weights themselves are tunable per capability. | Voice calls weight latency (users notice setup delay). Web search weights cost (high volume, low stakes). Same engine, different weights — routing policy, not retrieval. |

### Triggers that justify the standalone module

| Trigger | Why it means you need ARAC |
|---------|----------------------------|
| 5+ providers for any single capability | Sorting by one metric stops being sufficient |
| Product requires "budget mode" vs "premium mode" | Context-dependent weights |
| Provider performance varies by time-of-day or user segment | Cross-request learning |
| You want the planner to stop naming concrete providers | Abstract capability planning |
| Cost is being lost by not optimising across providers | Composite scoring with cost weight |

This document's recommendation: **even before those triggers fire, build the thin ARAC interface and constraint filter at baseline** (≈3–4 dev-days). Reasoning in §"Baseline vs Advanced" below.

---

## The Pipeline (Final Architecture)

```
┌────────────────────────────────────────────────────────────────────────┐
│                         Planner (M3)                                    │
│              emits abstract capability + context                         │
│              { capability: "web.search", context: {...} }                │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│  1. Candidate Resolver        capability → [provider_a, provider_b…]    │
│     (capability registry)                                                │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│  2. Constraint Filter         remove ineligible providers BEFORE scoring│
│     - tenant_allowed?                                                    │
│     - credentials_present?                                               │
│     - provider_enabled?                                                  │
│     - region_compliant?                                                  │
│     - risk_policy_passes?                                                │
│     - within_tenant_budget?                                              │
│     - SLA_achievable?                                                    │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│  3. Ranker                    score each survivor; produce sorted list  │
│     - composite weighted score (per-capability policy)                   │
│     - quality signals (not just success_rate)                            │
│     - cost_per_success, latency_p95, retry_rate, manual_override_rate    │
│     - explanations attached to each candidate                            │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│  4. Allocator                 choose primary + build fallback chain     │
│     - apply traffic rules (tenant pinning, A/B splits)                   │
│     - apply guarded exploration (≤10% on low-risk traffic)               │
│     - cap fallback depth per capability policy                           │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│  5. Executor (M5)             invoke primary; on failure walk chain      │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│  6. Outcome Recorder          structured telemetry                      │
│     {provider, capability, success, latency_ms, cost, quality_signal,    │
│      retry_count, fell_back_to, user_override?}                          │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│  7. Learning Loop             EMA → bandit → contextual bandit (later)  │
└────────────────────────────────────────────────────────────────────────┘
```

The pipeline is **constraint-filter → score → allocate**, not "score everything then filter". Filtering first is cheaper, safer, and ensures we never even consider providers we cannot legally use.

---

## Baseline vs Advanced (the explicit split)

A common shortcut is to declare M4 a "0-day" module and let M2 handle provider selection until the triggers above fire. This document rejects that shortcut narrowly: **the interface and constraint filter should ship at baseline**, because they cost almost nothing and prevent a painful refactor later. The sophisticated ranker can land later, behind the same surface.

### Baseline — build now (≈ 3–4 dev-days)

The minimal control plane that lets M3, M5, and M2 talk to a stable surface even when ranking is trivial.

| Item | What it is at baseline | Why now |
|------|------------------------|---------|
| **Abstract capability IDs** | Planner emits `web.search`, not `perplexity.search`. Capability registry maps capability → provider list. | Plans become stable across provider churn. M5 always calls `arac.select(capability)` regardless of whether ranking is sophisticated yet. |
| **`select_provider(capability, context)` interface** | Returns `{ primary, fallback_chain, decision_id }`. Internally calls M2 sort + capability registry. | Locks the contract so we can swap the implementation later without touching M3/M5. |
| **`record_outcome(decision_id, success, latency_ms, cost, quality_signal?)`** | Writes structured telemetry keyed by `decision_id`. | Without this, you cannot retroactively learn from baseline traffic. Cheap to add now, very expensive to backfill later. |
| **Constraint filter (hard gates)** | Tenant allow-list, credentials present, provider enabled flag, region compliance. | These are correctness/security concerns, not optimisation. They MUST exist day one. |
| **Single fallback chain per capability** | Ordered list, ≤2 fallbacks. Today this is the M2-sorted result; same shape as future scored chain. | Lets M5 retry on transient failure without re-planning. |
| **Capability-level routing policy file** | YAML/JSON per capability declaring: `weights`, `fallback.max_attempts`, `allowed_channels`, `sla_p95_ms`, `cost_cap_per_call`. Even if most fields are unused at baseline, the schema exists. | Prevents per-capability behaviour from getting hardcoded into M2 sort clauses. |
| **Cost config** | Static per-provider cost map (per call / per minute / per 1K chars). | Required for any future cost-aware decision; trivial to maintain. |

### Advanced — build later (when triggers hit, ≈ 9–11 dev-days)

| Item | Trigger to build |
|------|------------------|
| Composite weighted scoring (success / latency / cost / context / quality) | 3+ providers per capability, OR product asks for "budget mode" vs "premium mode" |
| Quality signals beyond success rate (semantic quality, user acceptance, manual override rate) | First customer complaint about "technically succeeded but bad output" |
| `cost_per_success` as the cost metric | First post-mortem where a "cheap" provider was actually expensive due to retries |
| Thompson Sampling, **guarded** | 5+ providers per capability AND enough traffic to learn (>1k calls/capability/day) |
| Contextual bandits (per-segment routing) | Time-of-day, language, or user-segment effects observable in telemetry |
| Preference learning from behaviour | Users start over-riding recommendations >5% of the time |
| SLA-aware multi-step allocation | Voice / real-time use cases with hard P95 latency budgets |
| Learned ranker (RouteLLM-style) | LLM-provider routing becomes a first-class cost lever |
| AgentRank-UC dual-graph | Cross-skill competence patterns matter (composite tasks, multi-skill workflows) |

The point is: **at baseline you ship 7 small things, none of which require ML.** They are interface + config + telemetry. The advanced layer slots in behind the same `select_provider` call when it is justified.

---

## Component Detail

### 1. Abstract Capability Registry

The single most important design upgrade in this document.

Bad (today's planner):

```json
{ "skill_id": "perplexity.search", "args": { "query": "flights to Goa" } }
```

Good (after baseline ARAC):

```json
{ "capability": "web.search", "args": { "query": "flights to Goa" } }
```

Resolution happens at execution time, not plan time. Concrete benefit: a plan generated Monday with `web.search` is still valid on Wednesday after Perplexity has degraded — the allocator picks Tavily without re-planning.

Registry shape (illustrative):

```yaml
capabilities:
  web.search:
    providers: [perplexity.search, tavily.search, serper.search]
    policy_ref: policies/web.search.yaml
  telephony.outbound:
    providers: [twilio.call, knowlarity.call]
    policy_ref: policies/telephony.outbound.yaml
  voice.tts:
    providers: [elevenlabs.tts, cartesia.tts, sarvam.tts]
    policy_ref: policies/voice.tts.yaml
```

### 2. Constraint Filter (hard gates, before scoring)

Eight checks, in order. First failure removes the candidate.

| Check | Source | Example failure |
|-------|--------|-----------------|
| `tenant_allowed` | Tenant config | Tenant X cannot use Twilio (contractual) |
| `credentials_present` | Secrets store | No Knowlarity API key for tenant Y |
| `provider_enabled` | Global feature flag | Provider disabled by ops during incident |
| `region_compliant` | Tenant region + provider region map | EU tenant cannot use a US-only provider |
| `risk_policy_passes` | Risk engine | Outbound call to a flagged number blocked |
| `within_tenant_budget` | Cost ledger | Tenant has consumed monthly LLM budget |
| `provider_health_active` | M2 health metric | Provider marked `disabled` by EMA |
| `sla_achievable` | Provider P95 vs request SLA | Real-time call needs <300ms; provider P95 is 800ms |

Filtering is O(N) and runs on every request. Do not "score everything and filter later" — that wastes compute and risks accidentally returning ineligible providers if scoring is buggy.

### 3. Ranker (scoring only — no selection)

The ranker assigns a numeric score and rationale to each surviving candidate. It does **not** decide who wins. Allocator does.

#### Per-capability weighted formula (baseline-compatible)

```
score = w_success      × success_rate
      + w_latency      × (1 - normalized_latency)
      + w_cost         × cost_score              // 1 - normalized_cost_per_success
      + w_quality      × quality_score
      + w_context      × context_bonus
      - w_consent      × consent_overhead
```

Per-capability weights, declared in policy file, not hardcoded:

```yaml
# policies/telephony.outbound.yaml
weights:
  success: 0.45
  latency: 0.30
  cost:    0.10
  quality: 0.05
  context: 0.05
  consent: 0.05
fallback:
  max_attempts: 2
  allowed_channels: [call, whatsapp]
sla_p95_ms: 600
cost_cap_per_call_inr: 4.0
```

```yaml
# policies/web.search.yaml
weights:
  success: 0.35
  cost:    0.30
  quality: 0.25
  latency: 0.10
  context: 0.00
  consent: 0.00
fallback:
  max_attempts: 2
sla_p95_ms: 8000
```

Voice weights latency and reliability heavily because a 1-second call setup delay is user-visible. Web search tolerates latency in exchange for quality and cost — the same engine, different policy.

#### Quality signals (beyond success rate)

Success-rate-only ranking is a known anti-pattern: a provider that returns "200 OK with garbage" looks identical to one returning "200 OK with the right answer". Track at minimum:

| Signal | What it captures | How to compute |
|--------|------------------|----------------|
| `task_success_rate` | Did the *task* succeed (planner-level) | M5 finalize hook |
| `provider_success_rate` | Did the provider call return non-error | Adapter-level |
| `semantic_quality_score` | Was the output usable downstream | LLM-as-judge, sampled |
| `user_acceptance_rate` | Did the user accept / not retry | Chat UX signal |
| `retry_rate` | Did the request need retries to succeed | M5 telemetry |
| `manual_override_rate` | How often does the user pick a different provider | Preference store |
| `latency_p95` | Tail latency, not mean | Streaming percentile estimator |
| `cost_per_success` | Total cost / successful tasks | Cost ledger |
| `fallback_frequency` | How often this provider is the primary that fails | Allocator counter |

`task_success_rate` and `provider_success_rate` are different and both matter. A provider can have 99% provider success but 60% task success because its outputs fail downstream validation.

#### `cost_per_success` (replace raw cost)

```
cost_per_success = total_cost_over_window / successful_task_count_over_window
```

A $0.001 provider that succeeds 30% of the time costs $0.0033 per success. A $0.005 provider that succeeds 95% of the time costs $0.0053 per success. Use `cost_per_success` in `cost_score`, never raw per-call cost. This single change usually flips ranker preference for the right reasons.

### 4. Allocator (selection only)

The allocator takes the scored list and decides:

1. **Primary** — usually top-scored, but may differ if exploration is active or tenant pinning applies.
2. **Fallback chain** — next K providers in score order, capped by `policy.fallback.max_attempts`.
3. **Traffic rules** — tenant pinning ("tenant X always gets Twilio"), A/B splits, regional preferences.
4. **Guarded exploration** — described below.

The split between Ranker and Allocator matters at scale because:

- Ranker is a pure function of telemetry + policy. Easy to unit-test.
- Allocator is where traffic-shaping, exploration, and policy overrides live. It is mutable and operationally tuned.
- You can replay a Ranker decision deterministically for debugging. You cannot replay an Allocator decision because of exploration randomness — but the `decision_id` records which branch was taken.

### 5. Guarded Exploration (not blind exploration)

Naive Thompson Sampling on production traffic is dangerous: it may route a real customer's outbound voice call to a provider you wanted to test. Guard rails:

- **Only on active providers** — never explore into a `degraded` provider.
- **Only on low-risk requests** — exclude payment-touching, voice/real-time, and high-value workflows. Idempotent web searches and offline summarisation are good candidates.
- **Only within tenant policy** — tenant must have opted in (default: opt-in for shared providers, opt-out for tenant-pinned).
- **Only if SLA allows** — do not explore into a provider whose P95 exceeds the request's SLA.
- **Hard cap** — ≤5–10% of capability traffic, per capability, per day.
- **Kill-switch** — global feature flag to disable exploration in seconds during incidents.

RouteLLM (ICLR 2025) shows >2× cost reduction without quality loss using learned routing for LLM selection — but the value comes from disciplined evaluation, not unconstrained exploration.

### 6. Outcome Recorder

Every `select_provider` call returns a `decision_id`. Every M5 invocation records its outcome under that `decision_id`. This is the join key that makes learning possible.

Minimum schema:

```
arac_decisions
  decision_id (pk), tenant_id, capability, request_context_hash,
  candidates_considered (json), candidates_filtered_out (json with reasons),
  primary_chosen, fallback_chain, exploration_branch (bool),
  policy_version, ts

arac_outcomes
  decision_id (fk), provider_used, attempt_index, success,
  latency_ms, cost, quality_signal, retry_count, fell_back_to,
  user_overrode (bool), error_class, ts
```

Without this join, you cannot answer "is provider A actually getting better, or are we just routing easier requests to it?" — the most important question in any routed system.

---

## Interfaces

```
arac.select(capability: str, context: RequestContext) -> Decision
  returns { decision_id, primary, fallback_chain, explanations }

arac.record_outcome(decision_id, attempt_index, success, latency_ms,
                    cost, quality_signal=None, error_class=None) -> None

arac.set_user_preference(user_id, capability, provider) -> None
arac.clear_user_preference(user_id, capability) -> None

arac.get_provider_health(capability) -> list[ProviderHealth]
arac.get_decision(decision_id) -> Decision   # for debugging / audit

arac.policy_for(capability) -> RoutingPolicy
```

The shape of `arac.select` does not change between baseline (M2-sort underneath) and advanced (full bandit). M3 and M5 are written against this surface from day one.

---

## Phased Evolution

```
Baseline (now)                         Mid-term (~6 mo)                   Long-term (12+ mo)
───────────────                        ────────────────                    ──────────────────
abstract capabilities      →           composite weighted scoring     →    contextual bandits
constraint filter (hard)               quality signals (multi-axis)        per-segment routing
single-tier fallback       →           cost_per_success ranking       →    SLA-aware multi-step
M2-sort under the hood     →           guarded Thompson Sampling      →    learned router (RouteLLM)
record_outcome telemetry   →           per-capability policy tuning   →    AgentRank-UC dual-graph
static cost config         →           per-tenant budget enforcement  →    preference learning
no exploration             →           ≤10% guarded exploration       →    full A/B infrastructure
```

The interface (`arac.select`, `arac.record_outcome`) does not change across phases. Only the implementation behind it does.

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| `skill_index` with health metrics | M2 | Telemetry source for ranker |
| Capability-level `skill_id` in PlanSpec | M3 | Without this, abstract capability planning cannot happen — the most important M3 contract change. |
| Finalize hook | M5 | Calls `record_outcome` |
| Tenant policy + cost ledger | Tenant management | Constraint filter inputs |
| User preference store | M7 (Memory) | Preference overrides |
| Risk engine hook | Risk/Compliance module | Constraint filter input |

The single hard dependency that must land alongside ARAC baseline is **M3 emitting capability-level `skill_id`**. Everything else can degrade gracefully.

---

## Effort Estimate

**Baseline (build now): 3–4 dev-days**

| Component | Effort |
|-----------|--------|
| Capability registry + abstract IDs in M3 | 0.5 day |
| `arac.select` / `arac.record_outcome` interface (M2-sort underneath) | 1 day |
| Constraint filter (8 hard gates) | 1 day |
| Per-capability policy file schema + 3 example policies | 0.5 day |
| Outcome telemetry tables + writer | 0.5 day |
| M5 wiring (resolve capability → concrete provider; record outcome) | 0.5 day |

**Advanced (when triggers hit): 9–11 dev-days**

| Component | Effort |
|-----------|--------|
| Composite weighted scoring with per-capability weights | 1–2 days |
| Quality signals (semantic_quality, user_acceptance, override_rate) | 2 days |
| `cost_per_success` aggregation and ranker integration | 1 day |
| Guarded Thompson Sampling | 2 days |
| Allocator/Ranker split, traffic rules, A/B hooks | 1 day |
| User preference overrides | 1 day |
| Testing with real provider mix | 1–2 days |

---

## Design Principles (the non-negotiables)

These are the load-bearing decisions in this design. Anything that contradicts them is a regression.

| Principle | Why |
|-----------|-----|
| **Control plane, not just a ranker** | Ranking + selection is one of eight responsibilities; the others (constraints, exploration, learning, SLA, cost, fallback, candidate generation) cannot be bolted on later without a refactor. |
| **Ship the interface at baseline, not the sophistication** | `arac.select` and `arac.record_outcome` cost ~2 days. They lock the contract so M3 and M5 never have to change again. |
| **Plans use abstract capabilities, not concrete providers** | A plan must outlive provider churn between plan time and execution time. |
| **Filter, then score** | Eliminate ineligible providers (tenant, credentials, region, risk, budget, SLA) before any scoring. Cheaper, safer, and avoids accidental routing to invalid providers. |
| **`cost_per_success`, not raw cost** | A cheap-but-flaky provider is not actually cheap. |
| **Multi-axis quality, not just `success_rate`** | A provider can return 200 OK with garbage. Track `task_success_rate`, `semantic_quality_score`, `user_acceptance_rate`, `manual_override_rate`. |
| **Guarded exploration only** | Exploration is opt-in, capped at ≤10%, never on real-time/payment workflows, behind a kill-switch. |
| **Ranker and Allocator are separate components** | Ranker is a pure function (testable, replayable). Allocator owns mutable concerns (traffic rules, exploration randomness, tenant pinning). |
| **Per-capability policy files** | Voice and search cannot share weights, SLA, or fallback depth. Codify per-capability policy as configuration, not code. |

---

## Research References

| Topic | Source | Key takeaway used here |
|-------|--------|------------------------|
| Learned LLM routing | RouteLLM (ICLR 2025) | >2× cost reduction without quality loss; routing is a real cost lever |
| PageRank for agents | AgentRank-UC (arXiv 2509.04979) | Dual-graph (usage + competence), recency decay, cold-start priors |
| Contextual bandits | CoCoMaMa (CEUR-WS 2025) | Context-aware combinatorial bandit for volatile agent sets |
| Real-time provider routing | Unify.ai | Benchmarks updated every 10 min; 3-axis optimisation pattern |
| Five-dimension evaluation | CLASSic (ICLR 2025 Workshop) | Cost, Latency, Accuracy, Stability, Security as ranking axes |
| Network-aware routing | NetMCP (2025) | `score = semantic_relevance × network_health` |
| EMA telemetry | Common pattern | `alpha = 0.1` ≈ 10-invocation recency weighting |

---

## TL;DR

ARAC is a control plane, not a ranker. Ship the **interface, constraint filter, abstract capabilities, per-capability policy file, and outcome telemetry at baseline** (3–4 days). Defer the actual ranker sophistication, quality signals, `cost_per_success`, and guarded exploration until the triggers in §"Baseline vs Advanced" are hit. Because the interface is stable from day one, the advanced layer slots in without touching M3 or M5.
