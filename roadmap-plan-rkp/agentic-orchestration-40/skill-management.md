# Skill Management — Control Plane (Registry + Index + Resolver + Health + Governance + Versioning)

> Module mapping: M2 (Skill Management — Advanced Baseline)
> Status: canonical reference for the Skill Management control plane
> Current completion: ~17% (registry + contracts done; resolver, governance, health, versioning, evaluation all missing)

---

## What This Is

The system's **complete control plane for every skill** — not just a registry.

Owns six responsibilities, each a distinct service:

1. **Registry** — source of truth. A skill does not exist unless it is registered here.
2. **Index** — fast discovery layer (semantic + keyword + capability graph + metadata filters).
3. **Resolver** — runtime selection (filtering, ranking, fallback selection, schema-fit, reasoning).
4. **Health** — live reliability tracking, circuit breaker, fallback eligibility.
5. **Governance** — tenant/user permissions, credential availability, risk class, approval policy.
6. **Versioning** — lifecycle, rollout, pinning, rollback, deprecation.

The planner **never sees the full catalog**. It asks the **Resolver** for executable candidates.

---

## What This Module Does NOT Own

- Skill discovery from APIs / synthesis from specs → M6 (New Skill Creation)
- Execution runtime → M7 (Execution Engine)
- Planning / DAG decomposition → M3 (Task Decomposition)
- Telemetry pipeline → M5 (Observability)

This module exposes:

```text
register()
find_skills()         # raw retrieval — internal use
resolve_skill()       # the canonical interface for planner/executor
health.update()
policy.evaluate()
version.lifecycle.transition()
```

---

## Why "Registry + Search" Is Not Enough

A registry-only design works at MVP scale (≤100 skills, single tenant, dev environment). It collapses the moment any of these become real:

| Real-world condition | Why a bare registry fails |
|---|---|
| Multi-tenant | A skill exists globally but isn't authorized for tenant B |
| Per-user credentials | `slack.post_message` exists, but user X hasn't connected Slack |
| Risk-tiered ops | `loan.restructure.offer` (L3) must require explicit approval; registry has no notion of risk |
| Provider degradation | Exotel is failing — need to silently route to Twilio without breaking the plan |
| Plan stability | A skill's behavior changes between versions; in-flight plans break without pinning |
| Schema-fit | Resolver returns `whatsapp.send` but the upstream node doesn't produce a `phone_number` |
| Retrieval quality | Vector search alone misses exact technical-skill matches; need hybrid scoring |

The Control Plane addresses each of these as a first-class concern.

---

## Current State

| What Exists | What's Missing |
|---|---|
| `seed_registry.py` — 70+ `SkillDefinition` entries with schemas, side_effect flags, idempotency | No resolver layer — planner gets full catalog in prompt |
| `SkillsRegistry` lookup by ID + capability tag | No tenant / user permission filtering |
| Namespace taxonomy partially expressed | No risk classification, no approval gating |
| Static seed at startup; restart required for new skills | No versioning, no rollout/rollback |
| Some retry policies wired into nodes | No live health metrics, no circuit breaker, no fallback chain |
| | No hybrid retrieval (semantic + keyword + capability graph) |
| | No schema-fit validation between candidate skill and DAG context |
| | No retrieval evaluation harness (Recall@K / MRR / leak rates) |

---

## Target Definition (Advanced Baseline)

> **The line:** the planner receives a *governed, healthy, permission-safe, ranked* set of executable skills from a registry of 500–2,000+ skills in **<100ms** — without ever seeing the full catalog. Every skill is versioned, every external action is risk-classified, every degraded provider is automatically bypassed, and retrieval quality is measured against a benchmark dataset.

Three tests of the baseline:

1. **No catalog leak** — agent prompt never contains more than 8 skill descriptions.
2. **No permission leak** — a skill the user lacks credentials for never appears in candidates.
3. **No silent regressions** — Recall@5 against the eval set is tracked per release; alarms fire on drop.

---

## Architecture Overview

```text
                       User Request
                            ↓
                Intent + Context Builder
                            ↓
                ┌───────────────────────────┐
                │      Skill Resolver       │
                │                           │
                │  1. Index Retrieval       │  ← Skill Index (vector + BM25 + graph)
                │  2. Capability Filter     │
                │  3. Permission Filter     │  ← Governance.permissions
                │  4. Credential Filter     │  ← Governance.credentials
                │  5. Health Filter         │  ← Health Service (circuit-breaker)
                │  6. Risk / Approval Filter│  ← Governance.risk
                │  7. Schema-fit Validation │
                │  8. Hybrid Ranking        │
                └───────────────────────────┘
                            ↓
              Top 3–8 candidates + alternatives + reason
                            ↓
                    Planner builds DAG
                            ↓
            Executor validates schemas at bind-time
                            ↓
       Governance enforces approval (if risk requires)
                            ↓
                       Skill execution
                            ↓
       Telemetry → Health Service → Index health field
```

---

## Core Components

### 1. Skill Registry (Source of Truth)

Every skill is a versioned, signed contract. Stored in Postgres (Supabase) — not just YAML / Python module — so it can be queried, indexed, and audited.

```yaml
skill_id: channel.whatsapp.send
version: 1.4.2
status: active                  # draft | staging | active | deprecated | retired | blocked
owner: comms-team
adapter_type: direct            # direct | mcp | api | marketplace | agent

description: Send a WhatsApp message to a recipient.
intent_examples:
  - "WhatsApp Raj the meeting summary"
  - "send a message to my dad"
  - "ping Aarti via WA"

input_schema:    { ...JSON Schema... }
output_schema:   { ...JSON Schema... }

permissions:                    # capability strings; user/tenant must hold these
  - communication.send
  - whatsapp.connected

credential_requirements:        # external auth needed
  - whatsapp_business_token

risk_level: L2                  # L0 read-only … L4 destructive
side_effect: true
idempotent: true

cost_hint: low                  # low | med | high (used in ranking penalty)
latency_hint_ms_p50: 800

tags:    [whatsapp, communication, messaging]
namespace: communication.messaging.whatsapp

fallback_skills:                # ordered fallback chain
  - channel.sms.send
  - channel.email.send

upstream_hints:                 # commonly-paired upstream skills (for schema-fit)
  - contact.lookup
  - text.summarize.llm
```

### 2. Skill Index (Discovery Layer)

Multi-index, queried in a single SQL statement.

| Index | Purpose | Backend |
|---|---|---|
| Vector | Semantic similarity ("phone John" ≈ `voice.call.initiate`) | pgvector + HNSW |
| Keyword (BM25) | Exact technical matches ("get_calendar_events") | Postgres `tsvector` |
| Capability graph | Namespace tree filtering (`communication.messaging.*`) | `ltree` extension |
| Metadata | Tenant, health, permissions, risk, version | regular columns |

**Indexed fields** (mirror of registry): `skill_id`, `version`, `name`, `description`, `tags`, `namespace`, `intent_examples`, `schema_summary`, `health_score`, `latency_p95_ms`, `success_rate_24h`, `risk_level`, `permissions[]`, `embedding`.

#### Hybrid retrieval scoring

```text
final_score =
    w_sem  * semantic_similarity
  + w_kw   * bm25_match
  + w_cap  * namespace_match
  + w_fit  * schema_fit_score
  + w_hth  * health_score
  - w_lat  * latency_penalty
  - w_cst  * cost_penalty
  - w_rsk  * risk_penalty
```

Weights live in config (per-tenant overridable). Defaults derived from the eval set (see §12).

### 3. Skill Resolver (the critical addition)

The single entry point for planner and executor. Replaces direct `find_skills()` use.

```python
def resolve_skill(
    intent: str,
    context: ResolutionContext,
) -> ResolutionResult: ...
```

`ResolutionContext`:

```text
tenant_id, user_id, plan_id, region,
available_credentials, user_permissions,
upstream_node_outputs (for schema-fit),
compliance_mode, dry_run
```

`ResolutionResult`:

```json
{
  "selected_skill": "channel.whatsapp.send@1.4.2",
  "alternatives": [
    "channel.sms.send@2.1.0",
    "channel.email.send@1.0.5"
  ],
  "confidence": 0.92,
  "reason": "Top semantic match; recipient is on WhatsApp; tenant has comms.send; provider healthy.",
  "requires_approval": false,
  "rejected": [
    {"skill": "channel.slack.post_message", "reason": "no slack credential for tenant"},
    {"skill": "channel.exotel.call", "reason": "circuit_open (success_rate_1h=12%)"}
  ]
}
```

The `rejected` list is what makes this debuggable — every exclusion has a reason.

### 4. Capability Hierarchy

```text
root
├── communication
│   ├── voice          (call.initiate, call.answer, call.transfer)
│   ├── messaging      (whatsapp.send, sms.send, slack.post)
│   ├── email          (email.send, email.search)
│   └── notification   (push.send, reminder.set)
├── information
│   ├── search         (web.search, news.search)
│   ├── document       (doc.parse, doc.summarize)
│   └── data_lookup    (weather.get, finance.stock)
├── scheduling
│   ├── calendar       (calendar.create, calendar.query)
│   └── tasks          (task.create, task.update)
├── contacts           (contact.find, contact.details)
├── memory             (memory.search, memory.store)
└── system             (preference.get, audit.log)
```

Used for: planner reasoning at the capability level, retrieval pre-filtering, provider abstraction (multiple providers under one leaf), grouping for evaluation metrics.

### 5. Health System (Live Reliability)

Per skill **and** per provider:

| Metric | Window | Source |
|---|---|---|
| `success_rate_1h` / `_24h` | rolling, EMA α=0.1 | telemetry M5 hook |
| `latency_p50` / `p95` | rolling | telemetry |
| `error_rate` (typed: timeout / 4xx / 5xx) | rolling | telemetry |
| `last_success_at` | timestamp | telemetry |
| `circuit_state` | enum | computed |

**Circuit breaker:**

```text
closed     → normal operation
open       → all calls short-circuit; resolver excludes the skill
half-open  → after cool-down, allow N probe calls; promote to closed on success
```

**Status thresholds (drive resolver eligibility):**

| Success Rate | Status | Resolver behavior |
|---|---|---|
| ≥ 80% | `active` | Eligible as primary |
| 50–80% | `degraded` | Eligible as fallback only |
| 20–50% | `unhealthy` | Probe via half-open; not surfaced to planner |
| < 20% | `disabled` | Excluded entirely |

**Fallback chains** (declared in registry, enforced by executor):

```text
primary:  exotel.call.initiate
fallback: twilio.call.initiate
fallback: channel.whatsapp.send
```

Executor consults health *at bind time* and walks the chain on failure.

### 6. Governance Layer

#### Risk classification (mandatory per skill)

| Level | Meaning | Examples |
|---|---|---|
| **L0** | Read-only, no external visibility | `calendar.search`, `contact.find` |
| **L1** | Internal write, reversible | `memory.store`, `preference.set` |
| **L2** | External communication | `whatsapp.send`, `email.send`, `call.initiate` |
| **L3** | Financial / legal / customer-impacting | `loan.restructure.offer`, `payment.charge` |
| **L4** | Destructive / irreversible | `delete.customer.data`, `account.terminate` |

#### Approval policy

| Risk | Default policy | Tenant override |
|---|---|---|
| L0 | no approval | — |
| L1 | no approval | tenant may require |
| L2 | tenant policy (default: silent) | per-recipient or per-template |
| L3 | **mandatory human approval** | cannot disable |
| L4 | admin-only; blocked unless explicitly granted | — |

#### Filters applied during resolution

```text
tenant_id          → tenant_skill_mapping
user_id            → user_role + user_permissions
credential_state   → credentials_table per (user, provider)
plan_membership    → SKU-gated skills
region             → region-restricted skills (data residency)
compliance_mode    → e.g. SOC2-mode disables certain L4
```

If any filter fails, the skill is added to `rejected` (with reason) and never surfaces in candidates.

### 7. Versioning & Lifecycle

```text
draft → staging → active → deprecated → retired → blocked
```

| Phase | Visible to | Allowed callers |
|---|---|---|
| draft | author only | dry-run only |
| staging | tenant allowlist | dry-run + sandboxed live |
| active | all eligible tenants | all |
| deprecated | all (with warning) | existing pinned plans |
| retired | none | only audit reads |
| blocked | none | none — security action |

**Plan pinning:**

```json
{ "skill_id": "channel.whatsapp.send", "version": "1.4.2" }
```

Plans created in flight are pinned to the version active at plan-creation time. Resolver respects pinning unless explicitly told to upgrade. Rollback = transition the bad version back to `staging` and re-pin affected plans.

### 8. Schema-Fit Validation

The resolver is the *first* place to catch "this skill can't be used here," before the planner builds an unworkable DAG.

```text
Required by skill:    phone_number, message
Available in context: borrower_list (from upstream contact.list)
Missing:              phone_number per item

Resolver decision:    needs_upstream → contact.lookup
                      OR rejects this skill if no upstream can supply
```

Outputs added to `ResolutionResult.alternatives` if a near-fit skill exists with a satisfiable upstream chain.

### 9. Retrieval API

**Internal (raw):**
```python
find_skills(query, namespace=None, top_k=20)
```

**Canonical (planner / executor):**
```python
resolve_skill(intent, context) -> ResolutionResult
```

Latency budget: P95 < 100 ms end-to-end including all filters.

### 10. Registration

```text
POST /skills/register
```

Hot-add path. Behavior by adapter type:

| Type | Steps | User-visible cost |
|---|---|---|
| Direct / API / Composio | validate → embed → store → publish event | None |
| MCP | validate → embed → store → MCP client reload | brief blip (sub-second) |
| Marketplace | validate → sandbox dry-run → embed → store | N/A (out-of-band approval) |
| Generated (M6) | validate quality gates → embed → store as `staging` | None — tenant must opt in |

Every registration emits a `skill.registered` event consumed by the index, the eval harness, and the audit log.

### 11. Database Model (Postgres / Supabase)

Minimum tables — kept normalized so each subsystem owns its rows:

| Table | Owner subsystem |
|---|---|
| `skills` | Registry (one row per skill_id, latest pointer) |
| `skill_versions` | Versioning (one row per skill_id × version) |
| `skill_embeddings` | Index (vector + dense metadata) |
| `skill_health` | Health (rolling counters, circuit state) |
| `skill_permissions` | Governance (capability strings required) |
| `skill_tenant_mapping` | Governance (which tenants have which skills) |
| `skill_credentials_required` | Governance (provider credential keys) |
| `skill_risk_policy` | Governance (risk class + approval rules) |
| `skill_dependencies` | Resolver (fallback chains, upstream hints) |
| `skill_execution_logs` | Telemetry feed → Health |

### 12. Evaluation & Benchmarking

**Without measurement, retrieval silently rots.** A fixed eval dataset is part of the baseline, not refinement.

`skill_eval.jsonl`:

```json
{"query": "ping Raj on WhatsApp",      "expected": ["channel.whatsapp.send"]}
{"query": "find restaurants in Patna", "expected": ["web.search"]}
{"query": "schedule a call tomorrow",  "expected": ["calendar.create", "channel.call.initiate"]}
```

**Tracked metrics (per release):**

| Metric | Why |
|---|---|
| Recall@5 | Did the right skill appear at all? |
| Precision@5 | How clean is the candidate set? |
| MRR | Position quality of the right answer |
| Wrong-skill rate | Hard wrongs (returned skill cannot do the task) |
| Permission-leak rate | Skills surfaced that user lacks permission for — must be 0 |
| Health-leak rate | Degraded skills surfaced as primary — must be 0 |
| Latency p95 | Resolver end-to-end |

**Alarm:** Recall@5 drops by >3pp release-over-release → block release.

---

## Execution Flow (concrete)

```text
User: "Send WhatsApp reminder to all overdue borrowers"

1. Intent + Context Builder
   intent = "send WhatsApp reminder"
   context = { tenant=A, user=X, creds=[wa_token], permissions=[comms.send] }

2. Resolver
   - retrieves: [whatsapp.send, sms.send, email.send, slack.post, ...]
   - permission filter: keeps [whatsapp.send, sms.send, email.send]
   - credential filter: keeps [whatsapp.send, sms.send]   (no smtp creds)
   - health filter: keeps both (twilio degraded → sms.send drops to fallback)
   - risk filter: L2 → tenant policy = silent
   - schema-fit: needs phone_number per borrower → adds upstream contact.lookup
   - rank: whatsapp.send wins (semantic + healthy + creds)
   → returns: { selected: whatsapp.send, alts: [sms.send], reason: "..." }

3. Planner
   builds DAG: borrower.list → contact.lookup → whatsapp.send (per item)

4. Executor
   binds inputs → schema-validate → governance check (L2, no approval)
   executes per item; on whatsapp.send failure for an item → walks fallback to sms.send

5. Telemetry → Health
   per-call success/latency → EMA update → circuit state recompute
```

---

## What Counts as Refinement (deferred)

| Refinement | Why deferred |
|---|---|
| Cross-encoder reranking | Hybrid scoring sufficient up to ~1k skills |
| Skill caching (memoize idempotent results) | Optimization; not correctness |
| Marketplace ranking & monetization | Post-GTM |
| Dynamic MCP server hot-add (sub-second) | Reload blip is acceptable |
| Skill auto-deprecation by usage decay | Manual lifecycle is fine at scale we're targeting |
| Multi-region resolver replication | Single-region is enough until we cross regions |

---

## Dependencies

| Needs | From | Why |
|---|---|---|
| pgvector + ltree extensions in Supabase | DevOps | Index backends |
| Embedding model (`sentence-transformers` or hosted) | Infra | Vector encoding |
| Telemetry post-execution hook | M5 | Drives Health |
| Auth/permission service | M11 (Security) | Drives Governance permission filter |
| Credential vault | M11 | Drives credential filter |
| Approval gateway | M11 | L3 / L4 enforcement |
| Audit log | M5 / M11 | Every governance decision and lifecycle transition |

## Effort Estimate

| Component | Effort |
|---|---|
| Registry tables + migration of `seed_registry.py` to DB | 1–2 days |
| Index (pgvector + BM25 + ltree) + embedding pipeline | 1–2 days |
| Resolver (the meat — filters, ranking, schema-fit) | 2–3 days |
| Health service + circuit breaker + fallback walking | 2 days |
| Governance (permissions, credentials, risk, approval hooks) | 2–3 days |
| Versioning + lifecycle transitions + plan pinning | 1–2 days |
| Eval harness (dataset, metrics, CI alarm) | 1 day |

**Total: ~10–14 dev days to reach the advanced baseline.**

---

## Key Concepts Explained

**Why a Resolver, not just `find_skills`?**
Search returns "skills that look similar to your query." Resolution returns "skills you are *allowed*, *credentialed*, and *able* to execute *right now*, *ranked*." The agent should never get to choose between two skills if one of them was going to fail at execution time.

**What is hybrid retrieval and why is vector alone not enough?**
Vector search is great at fuzzy intent ("phone John" → call.initiate). It is bad at exact technical names ("get_calendar_events_v2") because semantically that's just "calendar stuff." BM25 nails the exact match. Capability-graph filtering ("only `communication.messaging.*`") prunes noise. Combined scoring beats any single signal.

**Why is risk a baseline concern, not a refinement?**
Because the day you ship, an L3 skill (e.g. `payment.charge`) without approval gating is a security incident waiting to happen. Risk classification is metadata; approval enforcement is one decision tree. Both are cheap to add now and impossible to backfill safely once a skill has run unapproved in production.

**Why version from day one?**
Skills are *executed*. A behavioral change in `whatsapp.send@1.5.0` breaks every plan that assumed `@1.4.x` semantics. Pinning is the only way to make in-flight plans safe across rollouts. Adding versioning later forces an unpinned migration of every active plan — a breakage cliff.

**Why an eval set on day one?**
Retrieval quality is invisible until users complain. By the time users complain, the metric has been silently degrading for weeks. A 100-row eval set run in CI catches this before merge.

---

## Final Principle

> The agent should never see the full skill catalog.
> The agent should ask the **Skill Resolver** for executable candidates.
> The Resolver owns search, ranking, health, governance, permissions, fallbacks, schema-fit, and versioning — and explains every selection and rejection.

---

## Research References

| Topic | Source |
|---|---|
| Embedding-based skill retrieval (97.1% hit@K=3) | Semantic Tool Discovery for MCP — arXiv 2603.20313 |
| Skill body / docstring as decisive signal (29–44% lift) | SkillRouter — arXiv 2603.22455 |
| Capability tree at 200 → 200k skills | AgentSkillOS — arXiv 2603.02176 |
| Meta-tool retrieval pattern | Composio architecture — `COMPOSIO_SEARCH_TOOLS` |
| Hybrid retrieval (vector + BM25) | LangChain Hybrid Search docs |
| Circuit breaker patterns | Hystrix / resilience4j |
| Risk-tiered approval | Anthropic Model-context Action Safety, OpenAI tool-use guidelines |
