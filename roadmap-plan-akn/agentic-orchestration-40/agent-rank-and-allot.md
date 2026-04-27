# AgentRank and AgentAllot — Long-Term Vision (Not Baseline)

> Owner: AKN
> Module mapping: M4 (Skill Ranking & Selection) / L2 component **C4**
> Current completion: M4 at ~5% (only hardcoded Perplexity→Tavily fallback exists)
> Research basis: RouteLLM, AgentRank-UC, CoCoMaMa, SkillRouter, CLASSic framework

> **Boundary note (2026-04-25):** C4 owns the *entire* multi-provider routing concern — composite scoring, fallback chains, exploration/A-B, sticky-session memory, cost-budget honoring, and the *fallback walk on failure*. The C2 Execution Tools doc (TB-706) had originally drafted a "C2.5 Multi-Provider Routing Substrate" that intended to split decision (M4) from dispatch (C2). That split has been **removed** — fallback decisions on runtime failure are themselves ranking decisions, so they belong here. **C2 now stops at "execute one chosen call, return one Receipt."** L2 (Execution Engine) runs the fallback loop: it asks C4 for a ranked chain, then calls C2 once per attempt, feeding each Receipt back to C4 via `record_outcome`. See `execution-tools.md` §4 revision note.

---

## Why This Is NOT Baseline — And Where the Line Is

AgentRank as a standalone module is a **refinement**, not a baseline requirement. The Skill Management system (M2) already does basic ranking as a side effect of retrieval. This section draws a clear line between what M2 handles and what justifies AgentRank as its own module.

### What M2 already covers (no AgentRank needed)

The skill hierarchy (namespace/capability tree) in M2 is the same data as the "capability → providers" mapping in AgentRank. When a user says "call John":

1. **Semantic-router** narrows to `communication > voice` namespace — this IS the capability grouping
2. **Bi-encoder** retrieval finds `twilio.call.initiate`, `knowlarity.call.initiate` — these ARE the competing providers
3. Retrieval query filters by `health_status = 'active'` — this IS degradation filtering
4. Results sorted by `success_rate DESC` (or a weighted combo of similarity + health) — this IS basic ranking
5. Top result = best provider. Rest = implicit fallback chain

At 2-3 providers per capability, this is sufficient. The skill retrieval pipeline with health-metric sorting **is** basic AgentRank.

### What AgentRank holds separate (the genuine delta)

AgentRank becomes its own module when ranking requires logic that **cannot live inside a retrieval query**:

| Capability | Why it can't be a sort clause in M2 | Example |
|-----------|--------------------------------------|---------|
| **Context-dependent weights** | The retrieval query has no concept of the *request's* context affecting sort order. M2 finds skills similar to the query — it doesn't know if this is a budget trip or a premium trip. AgentRank takes request-level context (urgency, budget, language) and adjusts weights per request. | "Budget trip to Goa" → weight cost heavily → pick Serper ($0.001/call). "Business client research" → weight quality heavily → pick Perplexity ($0.005/call). Same capability, different winner. |
| **Exploration (Thompson Sampling)** | Retrieval always returns the current best. It never deliberately picks a suboptimal provider to test if it's improved. AgentRank maintains a probability distribution per provider and occasionally samples the #2 to discover improvements. | Tavily was slow last month, so M2 always ranks it last. But Tavily quietly upgraded their infra. Without exploration, the system never discovers this. AgentRank would occasionally route 10% of traffic to Tavily and detect the improvement. |
| **Abstract capability planning** | Today the planner picks concrete skills (`twilio.call.initiate`). AgentRank enables the planner to think in abstract capabilities (`telephony.outbound`) and defer provider resolution to execution time. This decouples planning from provider selection — the plan stays valid even if providers change. | Plan created Monday with `twilio.call`. By Wednesday, Twilio is degraded. Without AgentRank, the plan fails or needs re-planning. With AgentRank, the plan says `telephony.outbound` and the ranker picks Knowlarity at execution time. |
| **Cross-request learning** | M2 tracks per-skill health (did this skill succeed?). AgentRank tracks per-context outcomes (did calling this contact at 10am work better than 3pm? does this user prefer Perplexity results over Tavily?). This is behavioral learning across requests, not single-invocation telemetry. | "Calling Sharma at 10am → answered 8/10 times. Calling Sharma at 3pm → answered 2/10 times." M2 doesn't track this. AgentRank's competence graph does. |
| **Composite scoring formula** | M2 can sort by `success_rate` or `latency` — one dimension at a time. At 5+ providers per capability, you need a weighted formula: `0.4 × success + 0.25 × speed + 0.2 × cost + 0.1 × context + 0.05 × consent_overhead`. The weights themselves are tunable per capability. | For voice calls: weight latency heavily (users notice call setup delay). For web search: weight cost heavily (high volume, low stakes). Same formula, different weights — this is routing policy, not retrieval. |

### The line, simply put

> **M2 answers:** "Which skills are relevant to this request, and which ones are healthy?"
> **AgentRank answers:** "Among healthy, relevant skills, which one is *optimal for this specific request context*, and what should the system try next if it fails?"

M2 is skill-aware. AgentRank is context-aware and strategy-aware.

### When to build AgentRank

| Trigger | Why it means you need AgentRank |
|---------|-------------------------------|
| 5+ providers for any single capability | Sorting by one metric stops being sufficient |
| Product requires "budget mode" vs "premium mode" | Context-dependent weights |
| Provider performance varies by time-of-day or user segment | Cross-request learning |
| You want the planner to stop naming concrete providers | Abstract capability planning |
| You're leaving money on the table by not cost-optimizing across providers | Composite scoring with cost weight |

**Estimated timing:** 3-6 months post-GTM, when the provider ecosystem grows and context-dependent selection becomes a product differentiator.

---

## What This Is (Long-Term)

When multiple skills or providers can fulfill the same capability, the system must **rank** them (score candidates by quality, speed, cost, and context) and **allot** the best one to handle the request. "Rank" and "Allot" are two phases of one decision:

```
User intent: "Search for flights to Goa"
       │
       ▼
Candidate skills: [perplexity.search, tavily.search, google.search, serper.search]
       │
       ▼
  ┌──────────┐
  │  RANK    │  Score each candidate: success_rate, latency, cost, context match
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  ALLOT   │  Select top-1 (or fallback chain: top-1 → top-2 → top-3)
  └────┬─────┘
       │
       ▼
Winner: perplexity.search (score: 0.87)
Fallback chain: [tavily.search, google.search]
```

This is NOT about finding which skill to use (that's M2 retrieval). This is about choosing the best **provider/implementation** when multiple can do the same thing.

## Long-Term Goal

Every capability with multiple providers has automatic, context-aware, performance-based selection using live telemetry. The system shifts traffic away from degraded providers within minutes, explores underused providers to discover improvements, adapts selection based on request context (urgency, budget, language), and respects user preference overrides.

None of this is needed at baseline. M2's health-metric sorting handles provider selection until the triggers above are hit.

---

## The Ranking System (When Built)

### Architecture

```
                    ┌─────────────────────────┐
                    │   Telemetry Store         │
                    │   (per-skill rolling EMA)  │
                    │                           │
                    │   success_rate: 0.94       │
                    │   latency_p50: 340ms       │
                    │   cost_per_call: $0.002    │
                    │   last_updated: 2m ago     │
                    └────────────┬──────────────┘
                                 │
                                 ▼
┌────────────┐    ┌─────────────────────────────┐    ┌──────────────┐
│ Candidate  │───▶│         RANKER               │───▶│ Ranked List  │
│ Skills     │    │                               │    │ + Fallback   │
│ (from M2)  │    │ 1. Score each candidate       │    │   Chain      │
│            │    │ 2. Apply user preferences      │    │              │
│            │    │ 3. Apply context modifiers      │    │ [skill_a,    │
│            │    │ 4. Rank by composite score      │    │  skill_b,    │
│            │    │ 5. Build fallback chain          │    │  skill_c]    │
└────────────┘    └─────────────────────────────┘    └──────────────┘
                                 ▲
                                 │
                    ┌────────────┴──────────────┐
                    │   Context                  │
                    │   - user_preferences       │
                    │   - urgency level          │
                    │   - cost_sensitivity       │
                    │   - quality_requirement    │
                    └───────────────────────────┘
```

### Baseline Components

#### 1. Multi-Provider Registry

A configuration that maps each capability to its available providers:

| Capability | Providers |
|-----------|-----------|
| `web.search` | Perplexity, Tavily, Serper |
| `telephony.outbound` | Twilio, Knowlarity |
| `voice.tts` | ElevenLabs, CosyVoice, Cartesia |
| `voice.stt` | Deepgram, Whisper |
| `messaging.whatsapp` | Meta BSP, GreenAPI |
| `email.send` | Gmail API, SMTP |

**The key separation:** The planner thinks in capabilities ("I need to search the web"), not providers. It emits `skill_id = "web.search"` in the PlanSpec. The ranker resolves this to a concrete provider (Perplexity, Tavily, etc.) at execution time, based on live performance data. This means the planner never needs to know or care about which provider is fastest/cheapest today — that's the ranker's job.

#### 2. Composite Scoring Formula

Each candidate skill is scored using live telemetry:

```
score = (success_rate × w_success) 
      + (1 / normalized_latency × w_latency) 
      + (cost_score × w_cost) 
      + (context_bonus × w_context)
      - (consent_overhead × w_consent)
```

**Default weights (tunable per capability):**

| Weight | Value | Rationale |
|--------|-------|-----------|
| `w_success` | 0.40 | Reliability is most important |
| `w_latency` | 0.25 | Speed matters for real-time (calls, chat) |
| `w_cost` | 0.20 | Cost-aware but not cost-obsessed |
| `w_context` | 0.10 | Context match bonus (e.g., "budget trip" favors cheap provider) |
| `w_consent` | 0.05 | Fewer consent steps = smoother UX |

**Score components:**

| Component | Source | Computation |
|-----------|--------|-------------|
| `success_rate` | `skill_index.success_rate` (rolling EMA) | Direct value, 0.0–1.0 |
| `normalized_latency` | `skill_index.avg_latency_ms` | `latency / max_latency_in_group` (so 0.0–1.0, lower is better) |
| `cost_score` | `provider_costs` config | `1 - (cost / max_cost_in_group)` (cheaper = higher score) |
| `context_bonus` | User preferences + task context | +0.2 if user prefers this provider, +0.1 if context matches (e.g., Hindi content → Sarvam STT) |
| `consent_overhead` | `skill_definition.requires_consent` | +0.1 penalty if consent required but alternatives don't need it |

#### 3. Live Telemetry (Exponential Moving Average)

After every skill invocation, the system updates that skill's `success_rate` and `avg_latency_ms` using an EMA with `alpha = 0.1`.

**Why EMA and not a simple average?**

Imagine Tavily search worked perfectly for 1000 calls, then their API goes down. With a simple average, the success rate barely drops: 999/1001 = 99.8%. The system keeps routing to a broken provider. With EMA (alpha=0.1), the last ~10 calls dominate the score. After 5 consecutive failures, the score drops to ~50%, triggering auto-degradation. EMA is also stateless — just one number per metric, no need to store a window of recent results.

**The cold start problem:** A brand-new skill has zero data. Should it rank high (give it a chance) or low (don't trust it)? Neither — it gets a neutral prior: `success_rate = 0.8`, `avg_latency_ms = 1000ms`. This keeps it in the middle of the pack. As real invocations accumulate, the prior fades and actual performance takes over.

#### 4. Automatic Degradation Detection

Based on live telemetry, the system detects when a provider is failing and shifts traffic:

| Condition | Action |
|-----------|--------|
| `success_rate < 0.5` for >5 consecutive invocations | Mark skill `degraded` — still in fallback chain but never primary |
| `success_rate < 0.2` | Mark skill `disabled` — removed from ranking entirely |
| `avg_latency_ms > 3× baseline` | Mark skill `degraded` |
| External health check fails (if configured) | Mark skill `degraded` immediately |
| `success_rate > 0.8` after being degraded | Auto-recover to `active` |

**Traffic shift is implicit:** Because the ranker uses live EMA scores, a degraded provider's score drops automatically. The ranker selects alternatives without any manual intervention. The explicit `degraded`/`disabled` status is a safety net for severe failures.

#### 5. Fallback Chains

For each capability, the ranker produces not just a winner but an ordered fallback chain. All available providers (excluding `disabled` ones) are scored and sorted. The top-scored provider is the primary; the rest form the fallback chain in score order.

**Why this matters:** If the primary provider fails mid-execution, the execution engine (M5) can immediately try the next provider in the chain — without going back to the planner to re-plan. This turns a potential user-visible failure into a transparent retry. Example: Perplexity times out → Tavily is tried automatically → user sees results, never knows about the failover.

#### 6. User Preference Overrides

Users can say "always use Perplexity for search" and the system stores this as a preference (in `skill_data_store` under a `user_preferences` namespace). When a preference exists, the preferred provider gets a +0.3 context bonus in the scoring formula, making it almost always the primary selection.

**Safety valve:** If the preferred provider is `disabled` (severely broken), the ranker falls back regardless of user preference. The system won't keep routing to a dead provider just because the user prefers it.

#### 7. Cost Tracking

A static config maps each provider to its cost (per call, per minute, per 1K characters, etc.). Updated manually when providers change pricing. The cost score normalizes within a capability group: cheapest provider gets `cost_score = 1.0`, most expensive gets `cost_score = 0.0`. This means the ranker naturally favors cheaper providers when all else is equal, without hard-coding "use the cheap one."

---

## Ranker Interface

The ranker exposes 5 operations:

| Operation | Purpose | Called by |
|-----------|---------|-----------|
| `select(capability, context)` | Pick best provider + build fallback chain | Execution Engine (M5), at plan execution time |
| `get_fallback_chain(skill_id)` | Get ordered fallback list for a provider | M5, on primary failure |
| `record_outcome(skill_id, success, latency, cost)` | Feed telemetry back after each invocation | M5 finalize hook |
| `set_user_preference(user_id, capability, provider)` | Store explicit user preference | Chat handler (when user says "always use X") |
| `get_provider_health(capability)` | Health status of all providers for a capability | Analytics / debug endpoints |

**Integration with M5:** When the executor encounters a capability-level `skill_id` in the PlanSpec (e.g., `web.search`), it calls `ranker.select("web.search")`. The ranker returns a concrete provider (e.g., `perplexity.search`) plus a fallback chain. The executor uses the concrete provider for the adapter invocation.

**Integration with C2 (Execution Tools):** C2 does not see the ranking, the chain, or the fallback walk. The flow is:

```python
# In L2 Execution Engine, per plan node:
ranking = ranker.select(capability, context)           # C4
for provider in [ranking.primary, *ranking.fallbacks]: # C4-owned chain
    if provider.health == "disabled":                  # C4 filter
        continue
    receipt = adapter.run_skill(                       # C2 — one attempt
        skill_name=provider.skill_id,
        inputs=resolved_inputs,
        context=task_context,
    )
    ranker.record_outcome(provider.skill_id, receipt)  # C4 telemetry sink
    if receipt.success:
        break
    if receipt.error_category in {"CONSENT_DENIED", "AUTH"}:
        break  # not a fallback case; surface to user
```

C2 stays "given X, do X." C4 stays "given a capability and context, here's the ranked chain — and learn from outcomes." The fallback loop itself runs in L2 because L2 is the only layer that holds both halves (the chain from C4 and the executor's per-node lifecycle).

---

## Phased Evolution

The entire module is built incrementally as triggers are hit. There is no baseline effort — M2's retrieval handles provider selection until then.

```
Now (M2 handles it)               When triggers hit (~6mo)        Long-term (1yr+)
───────────────────               ──────────────────────          ─────────────────
M2 sorts by success_rate   →      Composite scoring formula  →   Contextual Bandits
                                  (weighted: success, latency,    (per-context optimal
                                   cost, context)                  selection)

M2 health_status filter    →      Auto-degradation with      →   Time-series trend
                                  EMA + recovery                  detection

Retrieval order = fallback →      Explicit scored fallback   →   SLA-aware escalation
                                  chains consumed by M5           chains

No exploration             →      Thompson Sampling          →   Full A/B testing
                                  (10% exploration traffic)       infrastructure

No user preferences        →      Explicit overrides         →   Preference learning
                                  ("always use Perplexity")       from behavior

No cost awareness          →      Static cost config         →   Budget-aware planning
                                                                  (cost constraints in planner)

Per-skill health only      →      Cross-request learning     →   AgentRank-UC dual-graph
                                  (time-of-day, per-contact)      (usage + competence)
```

## Dependencies (when built)

| Needs | From | Why |
|-------|------|-----|
| `skill_index` table with health metrics | Skill Management (M2) | Telemetry data source — already exists at baseline |
| M5 finalize hook for telemetry | Execution Engine (M5) | Feed outcome data — already exists at baseline |
| `skill_data_store` for user preferences | Memory (M7) | Store user overrides |
| Capability-level skill_ids in PlanSpec | Task Decomposition (M3) | Planner emits abstract capabilities, ranker resolves to concrete providers. Requires M3 change. |

## Effort Estimate (when built)

**Baseline effort: 0 days.** M2 handles provider selection.

**When triggers are hit (~9-11 dev days):**

| Component | Effort |
|-----------|--------|
| Composite scoring formula with tunable weights | 1-2 days |
| Ranker API (`select`, `get_fallback_chain`, `record_outcome`) | 2 days |
| M5 integration (capability → concrete provider resolution) | 2 days |
| User preference overrides | 1 day |
| Cost config + cost scoring | 0.5 day |
| Thompson Sampling (exploration layer) | 2 days |
| Testing with real providers | 1-2 days |

---

## Research References

| Topic | Source | Key Finding |
|-------|--------|-------------|
| LLM routing scoring | RouteLLM (ICLR 2025) | 95% GPT-4 quality at 48% cost using learned router |
| PageRank for agents | AgentRank-UC (arXiv 2509.04979) | Dual-graph (usage + competence) with recency decay and cold-start priors |
| Contextual bandits | CoCoMaMa (CEUR-WS 2025) | Context-aware combinatorial bandit for volatile agent sets |
| Real-time routing | Unify.ai | Benchmarks updated every 10 minutes, 3-axis optimization |
| Five-dimension evaluation | CLASSic (ICLR 2025 Workshop) | Cost, Latency, Accuracy, Stability, Security as ranking axes |
| Network-aware routing | NetMCP (2025) | `score = semantic_relevance × network_health` |
| EMA for live telemetry | Common pattern across all systems | `alpha = 0.1` gives ~10-invocation recency weighting |

---

## Key Concepts Explained

**Why not just hardcode "use Perplexity for search"?**
Because providers degrade, pricing changes, and new providers launch. Hardcoding works until it doesn't — and when it breaks, someone has to manually fix it at 2am. Performance-based ranking means the system adapts automatically. If Perplexity raises prices, its cost score drops and Tavily gets more traffic. If a new provider joins with better latency, it climbs the ranking organically.

**What is Thompson Sampling? (the evolution path)**
Imagine you have 3 search providers but don't know which is best. Thompson Sampling is a "try things, but be smart about it" strategy. For each provider, the system maintains a probability distribution of how good it *thinks* the provider is. Each time it needs to pick one, it randomly samples from each distribution and picks the highest sample. A provider that's been great 9/10 times will usually win — but occasionally a less-tested provider gets a chance, which is how the system discovers that a new provider is actually better. It balances exploitation ("use what works") with exploration ("try new things occasionally"). This is a refinement because the baseline EMA-based scoring already does a simpler version of this.

**What is a Contextual Bandit? (the long-term evolution)**
Thompson Sampling treats all requests the same. A contextual bandit says: "for *this type* of request, provider A is best, but for *that type*, provider B is better." Example: For Hindi-language voice synthesis, Sarvam TTS might outperform ElevenLabs, even though ElevenLabs has a higher overall success rate. The bandit learns these per-context preferences from data. This requires significant traffic to learn meaningful patterns, which is why it's deferred.

**What is AgentRank-UC (PageRank for agents)?**
Google's PageRank ranks websites by looking at which sites link to which. AgentRank-UC applies the same idea to agents/skills: build two graphs — a "usage graph" (which skills does the planner pick for which tasks?) and a "competence graph" (which skills actually succeed?). Solve for a combined ranking that incorporates both popularity and quality. Elegant but complex — the baseline linear scoring formula covers 90% of the value with 10% of the complexity.
