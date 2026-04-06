# Debug Logs — Baseline Definition

> Owner: AKN
> Module mapping: M1 (Observability Stack), M5 (Execution Engine telemetry)
> Scope: Dev-facing only. No user-facing dashboards.

---

## What This Is

End-to-end developer observability for the agentic pipeline. A developer must be able to trace any user request from entry (chat message / scheduled trigger / webhook) through GID → Planner → Executor → Skill invocation → Finalize, identify exactly where something failed or slowed down, and understand why — without reading source code or adding print statements.

## Current State

| What Exists | What's Missing |
|-------------|----------------|
| `structlog` JSON logging | No correlation ID across the pipeline |
| Request timing middleware | No distributed tracing (no spans, no trace IDs) |
| `agent_run_events` table (24 event types, persisted to Postgres) | No way to query "show me everything that happened for this user request" |
| SSE event streaming to mobile | No structured error categorization |
| `local_performance.log` for perf timing | No dashboards, no alerting |
| Basic try/catch logging in handlers | No log levels enforced consistently |

---

## Baseline Definition

**The line:** A developer can take any `run_id` and reconstruct the full lifecycle of that request in under 2 minutes — what happened, in what order, how long each step took, what failed, and what the LLM saw/produced. Anything beyond this is refinement.

### Baseline Components

#### 1. Correlation ID propagation

Every request gets a `trace_id` at the entry point (API endpoint / scheduled job trigger / webhook handler). This ID propagates through every function call, every LLM request, every skill invocation, every DB write. All log lines include it.

**What this means concretely:**
- `trace_id` generated at FastAPI middleware level
- Passed via `contextvars` (Python context) — no manual threading
- Every `structlog` log line includes `trace_id`, `user_id`, `run_id`
- LLM calls log `trace_id` + `model` + `prompt_tokens` + `completion_tokens` + `latency_ms`
- Skill invocations log `trace_id` + `skill_id` + `input_hash` + `latency_ms` + `success`

#### 2. Structured span logging (lightweight tracing)

Not full OpenTelemetry (that's refinement). A lightweight span model where each span records: which operation ran, when it started/ended, how long it took, whether it succeeded, and operation-specific metadata (skill_id, model name, token counts, etc.). Each span also records its parent, so you can reconstruct the full tree: "this LLM call happened *inside* this planning step, which happened *inside* this request."

Persisted to a `debug_spans` table (or appended to `agent_run_events` as span events). Queryable by `trace_id` to reconstruct the full waterfall — like a profiler, but for the agentic pipeline.

**Key spans to instrument (baseline set):**
| Span | Parent | Captures |
|------|--------|----------|
| `request.handle` | root | Total request time, endpoint, user_id |
| `gid.route` | request.handle | Intent classification result, routing decision, confidence |
| `planner.create_plan` | request.handle | Model used, prompt token count, completion tokens, plan_id, task count |
| `planner.validate` | planner.create_plan | Validation pass/fail, error details |
| `executor.compile_graph` | request.handle | Node count, edge count, parallel groups |
| `executor.run_node` | executor.compile_graph | skill_id, input hash, output summary, consent_required |
| `adapter.invoke` | executor.run_node | adapter_type, external_url (if applicable), response_code, latency |
| `memory.assemble_context` | planner.create_plan | Layers queried, token counts per layer, total context tokens |
| `memory.write_episode` | request.handle | Episode stored, facts extracted count |
| `llm.call` | (varies) | Model, prompt_tokens, completion_tokens, latency_ms, temperature |

#### 3. Error categorization

Every error gets a structured category, not just a stack trace. Categories like `PLANNING_FAILED`, `ADAPTER_TIMEOUT`, `CONSENT_DENIED`, `LLM_RATE_LIMITED`, etc. — about 12-15 categories covering the full pipeline. 

**Why this matters:** A stack trace tells you *where* the code crashed. An error category tells you *what kind of problem* this is — which is what you need for metrics ("30% of failures are adapter timeouts") and for deciding who should investigate ("consent failures are a UX problem; adapter timeouts are an infra problem").

Every error log line includes: `error_category` (the bucket), `error_detail` (human-readable explanation), and `stack_trace` (for deep debugging).

#### 4. Request replay / inspection endpoint

Dev-only API endpoint (gated behind admin auth):

```
GET /debug/trace/{trace_id}
```

Returns the full span tree for a trace — ordered by time, nested by parent, with all metadata. This is the "show me everything" endpoint.

```
GET /debug/runs/{run_id}
```

Returns `agent_run_events` + spans for a specific run, merged into a single timeline.

#### 5. Log level discipline

| Level | When to use | Example |
|-------|-------------|---------|
| `ERROR` | Something failed that should not have. Needs investigation. | Adapter timeout, LLM parse failure, checkpoint write failure |
| `WARNING` | Degraded but recovered. Or approaching a limit. | Fallback provider used, retry succeeded, token budget exceeded |
| `INFO` | Normal lifecycle events. One per major step. | Plan created, node started, node completed, run finalized |
| `DEBUG` | Detailed internals. Only on in dev/staging. | Full prompt text, full LLM response, full skill input/output |

Enforce: no `print()` statements. All logging through `structlog` bound logger with context.

#### 6. LLM call logging

Every LLM call (planner, synthesizer, GID, fact extraction) logs:
- `model` used
- `prompt_tokens` + `completion_tokens` (from API response)
- `latency_ms`
- `temperature` / `top_p` if non-default
- `prompt_hash` (for dedup analysis — are we sending the same prompt repeatedly?)
- At DEBUG level: full prompt and response text

---

## What Counts as Refinement (explicitly deferred)

| Refinement | Why deferred |
|------------|-------------|
| Full OpenTelemetry with Jaeger/Zipkin | Heavyweight. Lightweight span model covers 90% of debugging needs. |
| Grafana/Datadog dashboards | Dashboards are visualization. Baseline is data availability. |
| Alerting (PagerDuty, Slack alerts) | Requires defining SLOs first. Baseline is "can debug," not "auto-detect." |
| Log aggregation service (ELK, Loki) | structlog JSON + Postgres spans are queryable for a small team. |
| Performance profiling (flame graphs) | Only needed for deep latency investigation. |
| User-facing error explanations | That's M9 (frontend) territory. |
| Cost tracking dashboards | That's analytics, not debug. |
| Distributed tracing across microservices | Single FastAPI process today. Revisit when horizontal scaling happens. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| `contextvars` propagation in FastAPI | Core Platform (M1) | trace_id must flow through async handlers |
| `agent_run_events` table | Execution Engine (M5) | Already exists. Spans extend this. |
| Admin auth middleware | Security (M11) | Debug endpoints must be gated |

## Effort Estimate

- Correlation ID + contextvars: 1 day
- Span model + instrumentation of 10 key spans: 2-3 days
- Error categorization retrofit: 1-2 days
- Debug API endpoints: 1 day
- Log level audit + enforcement: 1 day

**Total: ~6-8 dev days to reach baseline.**
