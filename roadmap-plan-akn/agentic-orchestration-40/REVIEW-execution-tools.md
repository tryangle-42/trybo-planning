# C2 — Execution Tools: Long-Term Architecture Plan

> **Jira:** TB-706 (parent: TB-704 Arch Reviews)
> **Owner:** AKN
> **Layer:** L1 — Platform Foundation
> **Sibling component:** C1 — Barebone Platform (TB-705)
> **Status:** Solutioning — pre-implementation, pre review
> **Last updated:** 2026-04-25

---

## 0. Purpose of this document

This is the **architecture solutioning doc** for L1 / C2 (Execution Tools), companion to the C1 doc. Same format as TB-705: responsibilities, sub-components, contracts, integration points, current vs. target state, phased roadmap, open questions. Long-term maturity lens; baseline-vs-mature deltas noted per surface.

**Read C1 first.** Many primitives this doc *uses* (HTTP client, observability, idempotency, webhook bridge) are owned by C1. C2 is the layer of capability *on top* of those primitives.

---

## 1. The C1 / C2 dividing line, restated

**C1 (Barebone Platform)** = anything you'd need to run a backend at all, even with zero agents. Inward-facing infrastructure primitives.

**C2 (Execution Tools)** = every uniform-contract interface to something *outside the agent's brain* — channels, third-party APIs, MCPs, sub-agents, and LLMs-as-tools. The agent's hands.

The clean test for C2: *is this a deterministic action against an external system, exposed via a uniform invocation contract that the agent can call?* If yes → C2.

---

## 2. Mission

Provide **deterministic, reusable, observable action interfaces** so that the agent's brain (L2) and its memory (L4) can interact with anything outside themselves — phone, WhatsApp, web, Composio, Slack, Linear, OpenAI, custom MCPs, future sub-agents — without embedding system-specific complexity into decision-making layers.

Three non-negotiables:

1. **One contract.** Every callable thing — Direct Python function, MCP tool, Composio action, external agent, LLM call — implements the same `SkillAdapter.run_skill(skill_name, inputs, context) → (outputs, receipt)` interface. The executor doesn't care which kind it is.
2. **Pure capability, not decision.** A tool is a function: given inputs, do the thing, return outputs. *When* to call it = L2 Planner. *Which provider* = L2 Agent Rank. *Whether the user has consented* = M11. C2 only does the doing.
3. **Side-effect-aware by default.** Every tool declares its `side_effect` and `requires_consent` flags. The framework enforces consent at the adapter boundary — not the skill's responsibility, not optional.

---

## 3. What C2 absorbs from related drafts

| Surface | Where it was | Where it lives now | Why |
|---|---|---|---|
| **LLM Gateway** | KPR's barebone draft (1.11) | C2.8 (under "First-Party Tool Adapters" / LLM-as-tool) | Calling an LLM is structurally identical to calling Tavily — outbound action against an external system. |
| **Notification Service (FCM/APNs send)** | KPR's barebone draft (1.10) | C2.8 (channel adapter) | "Send a push to user X" is a channel action. Push-token registry stays in C1.6/C1.2 (data) and outbound HTTP stays in C1.8 — but the *send* is a tool. |
| **Communication channels** (Voice, WhatsApp, WebRTC, SMS, Email, Slack) | trybo-arch's M10 separately | All inside C2.8 | The April 5 L-model has *no* separate communication layer — channels are *instances* of Execution Tools. Treating them as adapters under one contract is what keeps the L-model clean. |

---

## 4. The 7 surfaces of C2

> **Note (revision 2026-04-25):** Multi-provider routing was originally drafted as C2.5. It has been **removed from C2** and folded into **C4 (Agent Rank & Allot)** — see `agent-rank-and-allot.md`. The split (decision in C4, dispatch in C2) was load-bearing only if M4 produced decisions ahead of time in the PlanSpec. Fallback selection on failure, A/B arm choice, sticky-session memory, and cost-budget honoring are all *ranking decisions that happen at the moment of failure*, which makes them C4's job. C2 now stops at "given a chosen adapter+skill+inputs, execute it once and report a Receipt." C4 owns the loop that walks the fallback chain by calling C2 for each attempt. Surface count reduced 8 → 7.

| # | Surface | One-liner |
|---|---------|-----------|
| **C2.1** | Adapter Framework — the contract | The `SkillAdapter` ABC, `Receipt` shape, error taxonomy, lifecycle hooks |
| **C2.2** | Adapter Type Library | Direct, MCP, Marketplace (Composio), Agent (external bidirectional), Sandbox (untrusted) |
| **C2.3** | Tool Lifecycle & Health | Probes, circuit breakers per tool, health states, auto-degrade, hot-add/remove |
| **C2.4** | Tool Execution Semantics | Idempotency, retries, timeouts, cancellation, partial-result recovery, streaming responses |
| **C2.5** | Tool Telemetry Pipe | Standard signal emission feeding M2 stats and C4 ranking |
| **C2.6** | Approval-Aware Tool Plumbing | The hook that flips a `requires_consent: true` call into a consent gate |
| **C2.7** | First-Party Tool Adapters | Comm channels, notifications, web, files, LLM-as-tool — all under one contract |

---

## 5. How C2 connects

```
                            L2 Execution Engine
                                    │
                                    │ for each plan node:
                                    │   ranking = C4.select(capability, ctx)
                                    │   for provider in ranking.chain:
                                    │     receipt = adapter.run_skill(provider, ...)
                                    │     C4.record_outcome(provider, receipt)
                                    │     if receipt.success: break
                                    ▼
        ┌─────────────────────────────────────────────────────────┐
        │  C2 Execution Tools                                     │
        │                                                         │
        │  C2.1 Contract  ──────────────►  C2.5 Telemetry         │
        │      │                                │                 │
        │      ▼                                ▼                 │
        │  C2.2 Type Library          C2.6 Consent Plumbing       │
        │      │                                │                 │
        │      ▼                                ▼                 │
        │  C2.3 Lifecycle/Health      C2.7 First-Party Adapters   │
        │      │                                                  │
        │      ▼                                                  │
        │  C2.4 Execution Semantics                               │
        │                                                         │
        └─────────────────────────────────┬───────────────────────┘
                                          │ Receipts feed C4
                                          ▼
                              ┌──────────────────────────┐
                              │  C4 Agent Rank & Allot   │
                              │  - composite scoring     │
                              │  - fallback chains       │
                              │  - exploration / A/B     │
                              │  - user preferences      │
                              │  - cost-aware routing    │
                              └──────────────────────────┘

        ┌─────────────────────────────────────────────────────────┐
        │  C1 primitives (HTTP, webhooks, idempotency, telemetry, │
        │  config, secrets) — used by every tool                  │
        └─────────────────────────────────────────────────────────┘
                                          │
                                          ▼
                              External: Twilio, Meta, Composio,
                              MCP servers, OpenAI, FCM, Slack, ...
```

**Key relationships:**

- **L2 Execution Engine** is the *only* direct caller of C2. For each plan node, it asks C4 for a ranking, then walks the chain by calling `adapter.run_skill(...)` once per attempt. C2 sees a single chosen adapter+skill+inputs per call.
- **L3 Skill Map (M2)** owns *what* tools exist (registry, schemas, descriptions, health filter). C2 owns *how* tools execute. M2 reads C2.5 telemetry to compute health and stats.
- **C4 Agent Rank & Allot** owns *which* provider to pick *and* the fallback walk on failure — the entire ranking-and-routing concern. C2 stays out of routing. C2 reports outcomes via Receipts (consumed by C4 via `record_outcome`).
- **M11 Governance** owns *whether* a side-effect is allowed. C2.6 is the *plumbing* that asks M11 before executing.
- **C1** provides every primitive C2 stands on. C2 doesn't reinvent HTTP, webhooks, or observability.

---

## 6. The 8 surfaces in detail

For each surface: **Mission** → **Sub-components** → **Key contracts** → **Integration points** → **Current state** → **Long-term target** → **Phased roadmap** → **Open questions**.

Effort bands: S/M/L/XL = ~2 / ~5 / ~10 / ~20+ dev-days.

---

### C2.1 — Adapter Framework (the contract)

**Mission.** Define the *one* interface every external capability must satisfy. No adapter type — Direct, MCP, Composio, external agent, LLM call, future Sandbox — can deviate. The executor stays adapter-agnostic.

**Sub-components.**
- **`SkillAdapter` ABC.** Single async method: `run_skill(skill_name, inputs, context) → (outputs, Receipt)`. Today's contract is roughly right; long-term we tighten it (see below).
- **Receipt shape.** Structured, versioned record of what the adapter did: `started_at, completed_at, duration_ms, status, error_category, attempt_count, breaker_state, cost, provider, raw_request_hash, raw_response_hash`. The receipt is what feeds C2.6 (telemetry) and C2.7 (audit/consent record).
- **Error taxonomy.** Versioned enum of failure categories — `TIMEOUT, RATE_LIMITED, AUTH, SCHEMA_INVALID, PROVIDER_5XX, PROVIDER_4XX, NETWORK, CONSENT_DENIED, CIRCUIT_OPEN, CANCELLED, UNKNOWN`. Every adapter maps its native errors to this taxonomy. M5 (Execution Engine) decides retry/fallback by category; C2.6 categorizes for stats.
- **Lifecycle hooks.** Long-term: `init()`, `health_probe()`, `shutdown()`, in addition to `run_skill()`. `init` lazily warms providers (e.g., MCP sessions); `health_probe` is what C2.3 calls; `shutdown` releases pooled resources during drain (C1.1).
- **Streaming-result extension.** Some tools return streams (voice transcription, SSE search results). Add `run_skill_streaming(...) → AsyncIterator[Chunk]` as an optional capability flag on the adapter.
- **Capability flags.** Adapter declares: `supports_streaming, supports_cancellation, supports_dry_run, idempotent_by_default`. Executor uses these.

**Key contracts.**
- ABC: `class SkillAdapter(ABC): async def run_skill(...)`, plus optional `init / health_probe / shutdown / run_skill_streaming`.
- `Receipt`: versioned Pydantic model. Backward-compatible additions only.
- `ErrorCategory`: enum, versioned. Adding a category is non-breaking; renaming requires migration.

**Integration points.**
- L2 Execution Engine — calls `run_skill` exclusively.
- C2.6 — receipts feed telemetry.
- C2.7 — consent gate sits between executor and adapter, reads `requires_consent` from the skill definition.
- C1.1 Lifecycle — adapters register `shutdown()` hooks.

**Current state.**
```python
class SkillAdapter(ABC):
    async def run_skill(self, *, skill_name, inputs, context) → (Dict, Receipt)
```
Single method. Receipt exists but is shallow. Error taxonomy partial (started in TB-662). No lifecycle hooks. No streaming. No capability flags.

**Long-term target.** Tightened ABC with optional lifecycle + streaming. Receipt rich enough that downstream stats/audit/consent never need to re-derive anything. Error taxonomy stable, versioned, exhaustive.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Tighten Receipt — add `provider, attempt_count, breaker_state, error_category`; backfill in existing adapters | M |
| P2 | Versioned `ErrorCategory` enum; map every adapter's native errors | M |
| P3 | Optional lifecycle hooks (`init`, `health_probe`, `shutdown`); register with C1.1 | M |
| P4 | Streaming extension (`run_skill_streaming`) for voice + search streams | M |
| P5 | Capability flags surface on the adapter; executor reads them | S |

**Open questions.**
- Should `SkillAdapter` be split per *kind* (sync tools vs. streaming vs. agent vs. consent-required)? My answer: no — one ABC with optional capabilities, otherwise the executor branches everywhere. Capability flags are the right escape valve.
- Should `Receipt` carry the raw request/response or only hashes? Hashes only by default (PII risk); raw is opt-in per adapter, gated by config flag.

---

### C2.2 — Adapter Type Library

**Mission.** Five concrete adapter implementations, each a thin shell over the ABC. Adding a sixth type (e.g., Sandbox for dynamically created skills) shouldn't require executor changes.

**Sub-components.**
- **DirectAdapter.** In-house Python handlers dispatched by `skill_name` (today: ~30 skills in a `_SKILL_DISPATCH` table). Long-term: keep dispatch table; ensure handlers are stateless or use C1 primitives (DB client, Redis, HTTP) — never instantiate their own.
- **MCPAdapter.** Config-driven, multi-server. **Streamable HTTP transport in production** (stdio reserved for local dev — collapses under concurrency at scale, ~20/22 fail at 20 concurrent clients per current research). Pool of 5-10 reusable sessions per MCP server. Hot-add/hot-remove servers via config change + a swap-in primitive in C2.3. Today: `langchain-mcp-adapters` `MultiServerMCPClient` is *stateless by default* (each call opens a fresh session) — for stateful work use `client.session()` context. We extend with pooling.
- **MarketplaceAdapter.** Composio (today) + future marketplace integrations. Handles OAuth refresh per provider, token storage, multi-app dispatch. Each marketplace tool gets a deterministic `skill_id` mapping.
- **AgentAdapter.** External bidirectional agents (today: Deal Hunter). Lifecycle: `start()` + `forward_answer()` + `close()`. Breaks the synchronous request/response shape — intentional. Long-term: standardized agent-card protocol, async message channels, supervised lifetime.
- **SandboxAdapter (future, M6 dependency).** For dynamically generated skills (auto-discovered MCPs, user-defined automations). Resource-limited execution, network policy, output validation, time-boxed. Full M6 design out of scope here, but the *adapter slot* is.

**Key contracts.**
- Each concrete adapter implements `SkillAdapter`. No exceptions.
- MCP server config schema: `{ name, transport: "http"|"stdio"|"sse_legacy", url|command, auth, max_pool_size, health_probe_interval }`.
- Marketplace adapter config: `{ provider: "composio", oauth_config, ... }`.
- Agent adapter config: agent-card JSON describing capabilities and message shapes.

**Integration points.**
- L3 Skill Registry — every registered skill declares its `adapter_type`.
- C1.7 Config — adapter-instance configurations sourced from here (hot-reloadable for MCP server list).
- C2.3 — health probes per adapter type.
- C1.8 Outbound HTTP — Marketplace + MCP-over-HTTP go through it.

**Current state.** Four adapters: Direct, MCP-Tavily-only, Marketplace-Composio, Agent-DealHunter. All single-instance, no pooling, no hot-reload. Direct is ~30 skills via dispatch table. MCP currently bound to one server (Tavily); the multi-server architecture from `architecture.md` is designed but not deployed.

**Long-term target.** Five adapter types with consistent lifecycles. MCP via Streamable HTTP, pooled, multi-server, hot-reloadable. Marketplace adapter generic enough to add a second marketplace alongside Composio. Agent adapter standardized via agent-card. Sandbox slot reserved for M6.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | MCPAdapter generalized to multi-server via `MultiServerMCPClient`; config-driven | M |
| P2 | MCP session pooling (5-10 per server) + Streamable HTTP transport | M |
| P3 | MCP hot-add/remove primitive (paired with C2.3 health) | M |
| P4 | Marketplace adapter abstraction generic enough for a second marketplace; OAuth refresh standardized | M |
| P5 | Agent adapter agent-card protocol formalization | M |
| P6 | Sandbox adapter slot (interface + stub; full impl is M6) | S |
| P7 | Direct adapter handler-instance lifecycle (init/shutdown hooks) | S |

**Open questions.**
- Direct adapter dispatch table will grow large (~100+ skills). Should it move from a single dispatch dict to a plugin-discovery pattern (skills register themselves)? Recommendation: yes, by P7. Reduces merge conflicts and makes "add a skill" a one-file operation.
- Composio is one marketplace; do we expect a second (ActionAI, Bhindi, Apify, etc.) to justify the abstraction now, or defer? Defer abstraction until we have a second concrete need; design the interface but don't generalize prematurely.

---

### C2.3 — Tool Lifecycle & Health

**Mission.** Every tool has an observable health state. Failing tools degrade automatically. Recovering tools come back online without operator intervention. Hot-add and hot-remove are first-class operations.

**Sub-components.**
- **Synthetic canaries** — a lightweight probe per tool, on schedule (default every 60s, tunable per tool). Returns `ok | degraded | down`. Probe shape varies: for a search tool, a known query; for a comm channel, an auth check; for an MCP server, `tools/list`.
- **Passive telemetry** — failure rate, p95 latency, schema-drift counter, auth-error counter from C2.6. Window: rolling 5 / 60 / 1440 min.
- **Circuit breakers per tool** — `aiobreaker` instance per (provider, tool). `fail_max=5, reset_timeout=60s` default; override per tool. **Critical: breaker is *per tool*, not just per provider**, because a healthy provider can have one broken tool (e.g., Composio Slack works but Composio Linear is broken).
- **Health-status state machine** — `active → degraded → disabled`, transitions driven by canary + passive telemetry crossing thresholds. `disabled` tools are removed from the planner's available-skills list (C2.6 publishes; M2 consumes).
- **Hot-add / hot-remove** — adding an MCP server should not require restart. Mechanism: config change → C1.7 push event → C2.2 swaps in a new pool atomically → C2.6 starts publishing telemetry for it. Removal is the reverse: drain in-flight calls, close pool, mark disabled.
- **Recovery** — `disabled` tools get probed at backoff intervals (1m, 5m, 15m, 60m). On three successive `ok` probes → re-promoted to `active`.

**Key contracts.**
- Internal API: `health.probe(tool_id)`, `health.get_status(tool_id)`, `health.subscribe_changes()`, `health.force_disable(tool_id, reason)`, `health.swap_in(adapter_config)`.
- Events: `tool.health.degraded`, `tool.health.disabled`, `tool.health.recovered`, `tool.swapped_in`, `tool.swapped_out`.
- Health record schema: `(tool_id, status, last_probe_at, last_probe_result, p95_latency_ms_5m, error_rate_5m, breaker_state)`.

**Integration points.**
- C2.6 — telemetry feeds passive signals.
- M2 Skill Registry — consumes status, hides `disabled` tools from planner.
- M4 Agent Rank — health is a top-priority ranking signal; degraded providers score lower automatically.
- C1.10 — canaries run as scheduled jobs.
- C1.3 — status changes published.
- C1.11 — health events become spans.

**Current state.** No canaries. No tool-level breakers. No `degraded` state — tools are binary "up or 500s." MCP is single-instance, no swap-in. Composio failures cascade.

**Long-term target.** Every tool probed. Every tool gated by a per-tool breaker. Status machine drives planner visibility. Hot-add/remove proven for MCP servers. Recovery automatic.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Canary primitive + per-tool probe registry; first-class breakers per tool | L |
| P2 | Health-status state machine + threshold logic + planner integration | M |
| P3 | Hot-add/remove for MCP servers (paired with C2.2 P3) | M |
| P4 | Recovery probe with backoff | S |
| P5 | Operator UI for force-disable / force-enable + health dashboard | M |

**Open questions.**
- How aggressive on auto-disable? Recommendation: conservative — `degraded` after 3 consecutive failures or p95 > 5x baseline; `disabled` only after an operator confirms or after 30 min of `degraded` with no recovery. Auto-disable on data-loss-class errors only.
- Does the planner need to know about `degraded` (and prefer alternatives) or only `disabled` (and avoid)? Both, but with different semantics — degraded is a M4 ranking signal, disabled is a M2 filter.

---

### C2.4 — Tool Execution Semantics

**Mission.** Define *how* a single tool call behaves: idempotency, retries, timeouts, cancellation, streaming, partial-result recovery. The executor (M5) calls a tool; C2.4 governs what happens between request and response.

**Sub-components.**
- **Tool-level idempotency** — content-addressable hash of `(skill_id, normalized_inputs)`. **Distinct from M5's plan-level idempotency** — that's "did this *plan node* already run?"; this is "is this *external call* safe to retry?" Some tools are naturally idempotent (search, lookups); side-effecting tools must declare an idempotency key the provider understands (Stripe-style `Idempotency-Key` header) or be marked non-retryable.
- **Retry policy per tool** — declared in tool definition: `{max_attempts, backoff: exp|linear|fixed, retryable_categories: [TIMEOUT, NETWORK, PROVIDER_5XX]}`. Default conservative. Honor `Retry-After` always.
- **Timeouts per tool** — declared in tool definition; never inherit "default infinite." Connect / read / total. Long-running tools (voice calls) have their own timeout regime — long total, short per-utterance.
- **Cancellation** — every tool call is cancellable. Adapter must check `context.cancelled` at safe points and abort cleanly. For streaming tools, closing the iterator is the cancel signal. Cancellation cascades from M5 (user clicks "stop" → run cancels → in-flight tool calls cancel).
- **Partial-result recovery** — for batch / multi-step tools (e.g., "search 10 sites, return summaries"), a partial failure shouldn't lose the partial result. Tools return `(outputs, Receipt)` even on partial failure; outputs include `_partial: true` marker. Caller decides whether to accept or retry.
- **Streaming responses** — for tools declaring `supports_streaming`, the adapter yields chunks; the executor either consumes incrementally (passing through to UI via SSE) or accumulates. Backpressure via async iteration is built in.
- **Resource limits per call** — max output bytes, max tokens (for LLM calls), max nested calls (for agent adapters). Prevents a single call from blowing memory.

**Key contracts.**
- Tool definition schema additions: `idempotent_by_default: bool, idempotency_key_header: str|None, retry_policy: {...}, timeout: {connect, read, total}, supports_cancellation: bool, supports_streaming: bool, max_output_bytes: int`.
- Internal API: `execution.run_with_semantics(adapter, skill_name, inputs, context, policy)` — the wrapper that applies retry/timeout/cancellation around a raw `adapter.run_skill`.

**Integration points.**
- C2.1 — capability flags on the adapter feed semantics.
- L2 Execution Engine — receives the wrapped, semantics-aware result.
- C1.4 — idempotency key store is C1; the *policy* of when to dedupe is C2.4.
- C1.8 — HTTP-level retries handled there; tool-level retries are higher; both layers cooperate (don't double-retry).

**Current state.** Per-skill timeouts partial. No tool-level retry policy. No cancellation. No partial-result protocol. No streaming. Idempotency is plan-level only.

**Long-term target.** Every tool has explicit policy (idempotency, retry, timeout, cancellation). Streaming supported where useful. Partial results never lost. Resource limits prevent runaway calls.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Per-tool retry policy declared in skill definition; wrapper applies + honors `Retry-After` | M |
| P2 | Per-tool timeout declared and enforced (connect/read/total) | S |
| P3 | Cancellation primitive; cooperative-cancel pattern documented; wired through M5 | M |
| P4 | Partial-result protocol (`_partial: true`) + receipt extension | S |
| P5 | Streaming wrapper for streaming-capable tools | M |
| P6 | Tool-level idempotency for known side-effect tools (Stripe-style key) | M |
| P7 | Resource limits (output bytes, tokens, nested calls) | S |

**Open questions.**
- Where does retry decision-making live — at C2.4 (per-tool policy), at M5 (per-node policy), or both? Recommendation: **both, layered**. C2.4 retries on transient/network within a single call (3-5 attempts). M5 retries by switching providers / fallback skill (1-2 attempts). They don't double-count because C2.4 reports "exhausted" and M5 picks up from there.
- Streaming UX through to the mobile client — how much of this is C2 vs. M5 vs. UI? C2 produces a stream; M5 forwards it to C1.3 (event bus) as `node.partial.chunk` events; UI consumes via SSE. Documented as a pattern.

---

### C2.5 — Tool Telemetry Pipe

**Mission.** Every tool invocation emits a standard signal: latency, success, cost, error category, schema-drift. M2 (registry stats), C4 (ranking), C2.3 (health) all consume from here. No tool implements telemetry itself.

**Sub-components.**
- **Standard signal envelope.** From every `Receipt`, derive: `(tool_id, provider, started_at, duration_ms, success, error_category, attempt_count, breaker_tripped, cost_usd, output_size_bytes, schema_drift_detected)`. Published as `tool.invocation.completed` events on C1.3.
- **Cost extraction.** For LLM tools: extract `gen_ai.usage.input_tokens` + `gen_ai.usage.output_tokens` from spans; compute cost via per-model pricing table (config). For per-call-cost tools (Maps, Twilio): provider returns or skill declares unit cost.
- **Schema-drift detection.** Adapter compares actual response shape against declared `output_schema`. Drift logged + counted; over-threshold → triggers C2.3 health degradation.
- **Aggregation.** Background sweeper rolls up raw events into `(tool_id, window)` aggregates: success_rate, p50/p95/p99 latency, error-category breakdown, total cost. Windows: 5 min / 1h / 24h / 7d.
- **Sinks.**
  - M2 reads aggregates for `success_rate`, `p50_latency_ms` on the SkillDefinition.
  - C4 reads aggregates for ranking signals (composite scoring formula, EMA inputs).
  - C2.3 reads recent error rate for health state.
  - C1.11 tracing carries the same data per-call as span attributes.

**Key contracts.**
- Internal API: `telemetry.record(receipt)` (auto-called by adapter framework after every `run_skill`).
- Events: `tool.invocation.completed`, `tool.schema.drift.detected`, `tool.cost.recorded`.
- Aggregate schema: `tool_stats (tool_id, window_start, window_end, success_count, fail_count, p50_ms, p95_ms, p99_ms, total_cost_usd, error_breakdown JSONB)`.

**Integration points.**
- C2.1 — receipt is the input.
- C1.3 — events ride the bus.
- C1.10 — aggregation jobs scheduled.
- C1.11 — same data also goes into spans.
- M2, C4, C2.3 — sinks.

**Current state.** No standardized telemetry. Per-skill stats not tracked. Cost not tracked. Schema drift not detected.

**Long-term target.** Every tool call emits a structured signal. Aggregates available to M2/M4/C2.3 with low latency. Cost tracked per tenant per tool per period (feeds C1.6 cost ceiling).

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Standard signal envelope; auto-emit from adapter wrapper | M |
| P2 | Aggregation sweeper + `tool_stats` table | M |
| P3 | Cost extraction for LLM tools (token-based pricing table) + per-call tools | M |
| P4 | Schema-drift detection + alerting | M |
| P5 | M2/C4/C2.3 read paths | S |
| P6 | Per-tenant cost rollup feeding C1.6 cost ledger | M |

**Open questions.**
- Aggregation in Postgres vs. ClickHouse-style columnar store? At our volume, Postgres is fine for years. Defer columnar until it actually hurts.
- Cost pricing table — config or DB? Config is fine if it changes monthly; DB if it changes per-tenant (e.g., Bring-Your-Own-Key with per-key pricing). Start config; promote to DB when needed.

---

### C2.6 — Approval-Aware Tool Plumbing

**Mission.** Every tool call with `requires_consent: true` automatically passes through a consent gate before executing. The gate machinery lives here in C2; the *policy* (when to auto-approve, what risk level, who's the approver) lives in M11/Governance.

This is the cleanest M11 ↔ C2 boundary. Repeating: **C2 owns the plumbing; M11 owns the rules.**

**Sub-components.**
- **Pre-call hook.** Before any adapter `run_skill` for a `requires_consent: true` skill: (1) gather consent context (action, draft, recipient, risk level from skill def) into a `ConsentRequest`; (2) call M11's `consent.enforce(request) → Allowed | Interrupted`; (3) if `Interrupted`, raise an interrupt that M5 (Execution Engine) catches and routes through the LangGraph `interrupt()` flow.
- **Resume protocol.** M11 returns `ConsentDecision = APPROVED | EDIT_REQUIRED | DENIED` along with optional `edited_inputs`. C2.6 then either: (a) executes with original inputs (APPROVED), (b) executes with edited inputs (EDIT_REQUIRED), or (c) returns a `CONSENT_DENIED` Receipt without executing (DENIED).
- **Audit recording.** Every consent decision recorded in audit log (C1.11) with `(actor, action, target, decision, edited?, timestamp)`. Receipt links to the consent decision ID.
- **Auto-approve evaluation.** M11 may auto-approve based on policy (low risk + previously approved + same recipient). C2.6 calls M11; M11 decides; C2.6 just respects.
- **Mid-call consent (for streaming/long-running tools).** Some tools may need to ask consent *during* execution (e.g., voice agent mid-call: "do I have permission to schedule this?"). C2.6 provides `tool.request_mid_call_consent(...)` — the streaming adapter can pause, request, resume.

**Key contracts.**
- Internal API: pre-call hook is invisible to adapters — wrapper-applied. `tool.request_mid_call_consent(action, draft) → ConsentDecision` for adapters that need it.
- `ConsentRequest`, `ConsentDecision`: shared types owned by M11; C2.7 uses them.
- Events: `consent.gate.intercepted`, `consent.gate.approved`, `consent.gate.denied`.

**Integration points.**
- M11/Governance — owns the policy + UI surfaces.
- L3 Skill Registry (M2) — `requires_consent` flag declared per skill.
- L2 Execution Engine — its `interrupt()` is the runtime mechanism C2.7 piggybacks on.
- C1.11 — every consent decision audited.

**Current state.** Consent enforcement exists at the executor level (`app/consent/domains/execution_approval/{plan_gate, skill_gate}`). It's working — TB-716 / TB-717 / TB-718 / TB-719 / TB-720 landed in the last weeks. What's *not* well-factored: it's intermixed with executor logic; C2.6 wants to formalize it as an adapter-framework hook so it works the same regardless of which adapter invoked the call.

**Long-term target.** Adapter wrapper applies consent gate uniformly. Mid-call consent supported for streaming tools. M11 policy fully decoupled from machinery. Audit trail complete.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Refactor existing consent gate into an adapter wrapper invoked by C2.1 framework | M |
| P2 | Standardize ConsentRequest / ConsentDecision contract; audit linkage | S |
| P3 | Mid-call consent primitive for streaming adapters | M |
| P4 | Consolidate audit recording in C1.11 audit_log | S |

**Open questions.**
- Should consent gate be invokable *from* M2 registry (e.g., disable a skill entirely when consent infrastructure is down) or only at runtime? Runtime-only — consent is a per-call decision.
- How does C2.6 interact with C2.4 retry? On `CONSENT_DENIED`, never retry — it's a hard no. On approval, retry inside the adapter is fine. Codify.

---

### C2.7 — First-Party Tool Adapters

**Mission.** Concrete tool implementations Trybo owns. Each is a *skill* registered in M2 and invoked through the framework — they're not special. But they're a meaningful chunk of work, so they get a surface.

This surface is where the April 5 "Execution Tools" L1 component meets reality — these *are* the execution tools, sitting on top of C2.1-C2.7.

**Categories (long-term, not all built today):**

#### Communication channels
| Tool | Provider(s) | State today | Long-term |
|---|---|---|---|
| `channel.call.outbound` | Twilio, Knowlarity, Vapi/Retell (managed) | Done | Multi-provider via C2.5; latency optimization; mid-call consent via C2.7 P3 |
| `channel.call.inbound` | Same | Done | Mid-call webhook → consent flow standardized |
| `channel.call.webrtc` | LiveKit | Done | Recording, multi-participant, hold |
| `channel.whatsapp.send` | GreenAPI, Meta BSP (Pinnacle) | Done | Template messages, interactive (buttons/lists), media |
| `channel.whatsapp.inbound` | Same | Done | Webhook normalization via C1.9 |
| `channel.email.send` | Gmail (Composio + native) | Partial — read-only | Compose / send / threads / attachments |
| `channel.sms.send` | Twilio | Basic | Two-way, MMS, delivery tracking |
| `channel.slack.post` | Composio | Partial | Native Slack adapter w/ DMs, threads, presence |
| `channel.telegram.*` | TBD | Not built | Bot integration |

#### Notifications
| Tool | Provider(s) | State today | Long-term |
|---|---|---|---|
| `notification.push.send` | FCM (Android), APNs (iOS) | Done | Logging, per-category preferences, scheduled delivery |
| `notification.in_app.deliver` | C1.3 → SSE | Partial | First-class typed notification stream |

#### Web / research
| Tool | Provider(s) | State today | Long-term |
|---|---|---|---|
| `web.search.*` | Perplexity, Tavily, Serper, Google | Partial multi-provider | Routed via C2.5 fallback chain |
| `web.scrape.structured` | Playwright MCP, Firecrawl | Not built | MCP-driven; sandboxed |
| `content.feed.fetch` | feedparser (RSS/Atom) | Done | Generic content ingestion |
| `content.{hn,arxiv,producthunt,reddit}.*` | Direct APIs | Done (UC5) | Steady |
| `maps.{directions,geocode,places}.google` | Google Maps Routes/Places | Built (UC1) | Steady |
| `weather.forecast.openmeteo` | Open-Meteo | Done | Steady |

#### Files / documents
| Tool | Provider(s) | State today | Long-term |
|---|---|---|---|
| `file.text.extract` | Kreuzberg (PDF/DOCX/etc.) | Partial | Generic extraction pipeline |
| `file.audio.transcribe` | Whisper batch / streaming | Partial | Streaming via C2.4 P5 |
| `file.image.analyze` | OpenAI Vision / Claude Vision | Not built | First-class once L4 needs it |
| `file.tabular.read` | CSV/XLSX parsers | Partial | Already wired into planner context |

#### Third-party SaaS via marketplace
| Tool | Provider(s) | State today | Long-term |
|---|---|---|---|
| `tasks.issue.create` | Linear, GitHub Issues | Via Composio | Generic issue-tracker abstraction |
| `knowledge.note.write` | Notion, Obsidian | Via Composio / community MCP | Generic knowledge-base abstraction |
| `meeting.transcript.import` | Zoom, Teams, Meet | Via Composio | Transcript ingestion pipeline (post-beta) |

#### LLM as a tool
| Tool | Provider(s) | State today | Long-term |
|---|---|---|---|
| `llm.complete` | OpenAI, Anthropic, Gemini, Ollama | Per-service ad-hoc | Single LLM Gateway: provider registry, fallback chain via C2.5, structured output across all, streaming across all, multi-model routing by task type, per-tenant cost ceiling via C1.6 |
| `llm.embed` | OpenAI, voyageai | Ad-hoc | Single embedder gateway |
| `llm.classify` | Smaller cheap models | Ad-hoc | Routed via task type |

**Key principle for C2.7.** Every entry above is *just a skill*. No special path. The Adapter Framework (C2.1) + Type Library (C2.2) + Execution Semantics (C2.4) + Telemetry (C2.5) + Consent (C2.6) handle them uniformly. Multi-provider routing (which one of Twilio vs. Knowlarity, OpenAI vs. Anthropic, Perplexity vs. Tavily) lives in **C4 Agent Rank**, not here. The only thing that varies in C2.7 is the handler / adapter binding.

**Phased roadmap.** Not enumerated phase-by-phase here — sequencing is driven by GTM milestones (M1-M9 in `gtm-milestone-plan.md`), not by C2.7 itself. C2.1-C2.6 are the architecture work; C2.7 is *applying* that architecture to specific tools, mostly a GTM-driven workstream.

**Open questions.**
- How aggressive on consolidating LLM Gateway? Today every service that calls an LLM does so directly. Migrating ~15 callers is a Tranche-B item. Should be sequenced after C4 routing infrastructure exists and before any per-tenant cost ceiling work (C1.6 P4) — so the migration buys both fallback (via C4) and cost gates in one go.
- Should `notification.push.send` move to C1 (closer to FCM-as-infra) or stay in C2 (as a channel)? Decision in this doc: **C2** — the *send* is an action; the *device registry* + *outbound HTTP* it depends on are C1. Splits cleanly.

---

## 7. Cross-cutting design principles

1. **One contract, many adapters.** The cost of adding a new external system is bounded — write an adapter, register a skill, declare retry/timeout/consent flags. No executor changes, no special paths.

2. **C1 primitives, always.** No tool reaches for `httpx` directly, instantiates its own logger, or stores its own idempotency state. Use C1.

3. **Side effects are gated.** `requires_consent: true` + side-effect flag → automatic gate. Not the skill's responsibility, not optional.

4. **Telemetry is automatic, not opt-in.** Every `run_skill` produces a Receipt; the framework emits the standard telemetry signal. Tool authors don't have to remember.

5. **Health is observable, automatic, recoverable.** No tool stays "broken" silently. Probes catch it; breakers contain it; recovery un-breaks it.

6. **Routing is C4's job, not C2's.** When multiple providers can satisfy one capability, ranking + fallback walking + A/B + sticky-session + cost-budget honoring all live in C4 Agent Rank. C2 executes a single chosen call and reports a Receipt. The fallback loop is C4's, not C2's. Keeps C2 testable as "given X, do X" and keeps ranking policy debuggable in one place.

7. **The receipt is the contract for everything downstream.** Stats, ranking, audit, consent records, cost — all derive from the Receipt. Make it complete.

---

## 8. Dependencies

### Upward — what consumes C2

| Layer | Depends on | For what |
|---|---|---|
| L2 Execution Engine | C2.1 | The only direct caller; per-attempt `adapter.run_skill` (the fallback loop itself is C4's) |
| L3 Skill Registry (M2) | C2.5 telemetry | Aggregated stats per skill |
| C4 Agent Rank & Allot | C2.5 telemetry | Receipts in (`record_outcome`); ranking decisions back out to L2 |
| M11 Governance | C2.6 | The hook into consent enforcement |
| L4 Memory (synthesis nodes) | C2.7 (LLM-as-tool) | Personalization calls |
| C1.11 Observability | All of C2 | Spans + audit |

### Downward — what C2 depends on

| Dep | What we use | Owner |
|---|---|---|
| C1.8 Outbound HTTP | All HTTP-bound adapters | C1 |
| C1.9 Inbound Protocol Bridge | Webhook normalization for inbound channels | C1 |
| C1.4 Idempotency / Locks | Tool-level idempotency keys | C1 |
| C1.11 Observability | Spans, logs, audit | C1 |
| C1.10 Background Work | Canary jobs, telemetry rollups | C1 |
| C1.7 Config | Provider configs, MCP server list, feature flags | C1 |
| L3 Skill Registry (M2) | Skill definitions, capability declarations | L3 (read-only from C2) |
| M11 Governance | Consent policy callable | M11 |
| C4 Agent Rank & Allot | Ranked candidate + fallback chain consumer at L2 (not at C2) | C4 |

If a dependency above is missing or broken, the corresponding C2 surface is blocked. **C2 cannot ship ahead of its C1 dependencies** — C1.8 (HTTP) and C1.11 (observability) are particularly load-bearing.

---

## 9. Phased delivery

Sequenced to unblock the most code soonest, while respecting C1 dependencies.

**Tranche A — contract + observability (must-have).**
- C2.1 P1-P2 (Receipt tightening + ErrorCategory).
- C2.5 P1-P2 (telemetry envelope + aggregation).
- C2.4 P1-P2 (per-tool retry + timeout policy).
- Depends on C1.11 P1-P2 (OTel) — coordinate sequencing.

**Tranche B — health + adapter type maturity.**
- C2.3 P1-P2 (canaries + breakers + status machine).
- C2.2 P1-P2 (MCP multi-server via Streamable HTTP + pooling).
- Depends on C1.8 P1-P3 (HTTP centralization).
- Pairs with C4 first phase (capability abstraction, fallback-chain semantics) — coordinate with C4 owner.

**Tranche C — consent + cancellation + streaming.**
- C2.6 P1-P2 (consent wrapper + standardized contract).
- C2.4 P3-P5 (cancellation + partial result + streaming).
- C2.2 P3-P5 (MCP hot-add/remove, marketplace abstraction, agent-card).

**Tranche D — long tail + polish.**
- C2.5 P3-P6 (cost extraction + drift + per-tenant rollup).
- C2.7 LLM Gateway consolidation.
- C2.2 P6-P7 (Sandbox slot + Direct adapter plugin discovery).
- C2.3 P3-P5 (hot-add MCP, recovery, operator UI).

**Total rough effort:** ~40-55 dev-days across the architecture surface (down from prior ~50-70 after removing C2.5 routing substrate, which moved to C4). Plus the C2.7 first-party adapter work which is GTM-driven and counted against the 60% bucket.

Tranche A is the unblock for everything else. It can largely run in parallel with C1 Tranche A.

---

## 10. Open questions for review

Ranked by blast radius:

1. **Receipt shape — backward compatibility.** Tightening Receipt means existing receipts in DB don't have new fields. Plan: additive only, all new fields optional, default values for old rows. Confirm acceptable.
2. **C2 ↔ C4 boundary — confirm.** Routing logic (fallback walks, A/B, sticky, cost-budget) lives entirely in C4. C2 executes one call per attempt. The fallback loop runs in L2 (executor), driven by C4's chain, calling C2 each iteration. Confirm this is the cut you want.
3. **MCP transport choice.** Streamable HTTP everywhere in production; stdio only in dev. Confirm.
4. **LLM Gateway consolidation timing.** Tranche D — recommend after C4's routing primitive exists + C1.6 cost ledger so the consolidation buys both fallback and cost gates in one go.
5. **Sandbox adapter slot now or later?** Slot the interface in by P6 of C2.2 even if M6 is years out — cheaper than retrofitting. Confirm.
6. **Per-tool vs. per-provider breakers** (C2.3 open Q). Recommend per-tool. Confirm.
7. **Streaming through to UI architecture.** Documented as: tool yields → C2.4 forwards → C1.3 publishes `node.partial.chunk` → C1.9 SSE delivers. Confirm pattern.

---

## 11. Companion document

C1 Barebone Platform (TB-705) sits alongside this and provides every primitive C2 stands on. See `roadmap-plan-akn/agentic-orchestration-40/barebone-platform.md`. The `requires_consent` flag and `risk_level` policy come from M11/Governance — separate doc.

---

## 12. Source references

- April 5 meeting notes: `meetings/2026-04-05.md`
- C1 companion: `roadmap-plan-akn/agentic-orchestration-40/barebone-platform.md` (TB-705)
- Trybo arch (gold standard): `../../trybo-arch/architecture.md` §7-§9, `../../trybo-arch/module-ownership-map.md` (M1, M4, M10)
- Platform completion snapshot: `../../trybo-arch/platform-completion-status.md` (M1, M4, M10)
- Skill strategy: `../../trybo-arch/skill-strategy.md`
- Codebase anchors:
  - `app/agentic_mvp/runtime/adapters/base.py` — current `SkillAdapter` ABC
  - `app/agentic_mvp/runtime/adapters/direct.py` — `DirectAdapter` w/ ~30 skills in `_SKILL_DISPATCH`
  - `app/agentic_mvp/runtime/adapters/mcp_tavily.py` — single MCP adapter today
  - `app/agentic_mvp/runtime/adapters/marketplace_composio.py` — Composio marketplace adapter
  - `app/agentic_mvp/runtime/adapters/agent.py` — external agent adapter (Deal Hunter)
  - `app/agentic_mvp/runtime/contracts.py` — `Receipt`, `ExecutionContext`
  - `app/consent/domains/execution_approval/` — current consent enforcement
  - `app/services/llm_service.py` — today's ad-hoc LLM caller (target for LLM Gateway consolidation)

External research informing this doc:
- [langchain-mcp-adapters reference](https://reference.langchain.com/python/langchain-mcp-adapters)
- [MCP transport perf — stdio vs HTTP](https://stacklok.com/blog/mcp-server-performance-transport-protocol-matters/)
- [MCP health checks pattern](https://mcpcat.io/guides/building-health-check-endpoint-mcp-server/)
- [aiobreaker async circuit breakers](https://pypi.org/project/aiobreaker/)
- [tenacity async retry](https://tessl.io/registry/tessl/pypi-tenacity/9.1.0/files/docs/async-support.md)
- [OpenAI rate-limit cookbook](https://cookbook.openai.com/examples/how_to_handle_rate_limits)
- [Twilio 429 best practices](https://www.twilio.com/docs/usage/rest-api-best-practices)
- [LLM retries / fallbacks / breakers](https://www.getmaxim.ai/articles/retries-fallbacks-and-circuit-breakers-in-llm-apps-a-production-guide/)
- [OTel GenAI agent spans](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/)
