# Execution Engine — Baseline Definition

> Owner: RKP
> Module mapping: M5 (Execution Engine)
> Current completion: ~62% — Two-tier graph, fanout, consent gates, checkpointing, batch, SLA, warm restart done. Error recovery, cancellation, timeouts, resource limits, analytics missing.

---

## What This Is

Takes a compiled LangGraph StateGraph (from M3's DAG compiler) and runs it to completion. Handles parallel execution, consent interrupts, error recovery, and real-time progress streaming. Owns the **run → batch → task** hierarchy that gives users and the UI visibility into what the agent is doing.

## Current State

| Built | Not Built |
|-------|-----------|
| Two-tier graph (outer lifecycle + inner dynamic) | Error recovery (single failed node kills the run) |
| Parallel fan-out (LangGraph Send API) | Run cancellation / abort |
| Consent gates (`interrupt()` + `Command(resume=)`) | Per-node and per-plan timeouts |
| Checkpointing to Postgres (`langgraph-checkpoint-postgres`) | Resource limits (max nodes, max API calls) |
| Idempotency (content-addressable hashing) | Execution analytics (per-plan metrics) |
| SSE streaming (~24 event types) | |
| Batch execution (multi-batch resume) | |
| SLA tracking (30s sweep, escalation chain) | |
| Warm restart (resume from checkpoint) | |

---

## Baseline Definition

**The line:** The executor runs plans reliably with proper error handling, timeout management, and user-visible progress. The run→batch→task hierarchy is correctly maintained so the UI can show granular status and offer retry at any level. Single node failures don't kill the entire run. Anything beyond this is refinement.

---

## The Run → Batch → Task Hierarchy

This is the core visibility model. One user prompt creates one agent run. The run contains batches. Batches contain tasks.

```
User: "Research our competitors and call our top 3 partners about the findings"
  │
  └─ agent_run (run_id: "run_abc")
       │
       ├─ batch_1: "Internet research" (batch_id: "b1")
       │    ├─ task: research competitor Alpha    (task_id: "t1")
       │    ├─ task: research competitor Beta     (task_id: "t2")
       │    └─ task: research competitor Gamma    (task_id: "t3")
       │
       ├─ batch_2: "Partner calls" (batch_id: "b2")  [depends on batch_1]
       │    ├─ task: call partner Sharma with findings   (task_id: "t4", requires_consent)
       │    ├─ task: call partner Mehta with findings    (task_id: "t5", requires_consent)
       │    └─ task: call partner Patel with findings    (task_id: "t6", requires_consent)
       │
       └─ batch_3: "Consolidation" (batch_id: "b3")  [depends on batch_2]
            └─ task: synthesize all call outcomes    (task_id: "t7")
```

**Why this matters for UX:**

| Level | What the user sees | Actions available |
|-------|-------------------|-------------------|
| **Run** | "Agent is working on your request" | Cancel entire run |
| **Batch** | "Researching 3 competitors (2/3 done)" | See batch progress, retry failed batch |
| **Task** | "Calling Sharma — waiting for approval" | Approve/deny, retry individual task, see result |

**How IDs flow:**
- The planner creates the PlanSpec with task_ids
- The compiler groups tasks into batches based on dependency structure: tasks with the same dependencies and no inter-dependencies form a batch (they can run in parallel)
- The executor creates `agent_run`, `batch`, and `bot_tasks` records in the DB as it encounters them
- SSE events reference all three IDs so the frontend can render the tree

**Existing schema already supports this:** `agent_runs` (run level), `batches` (batch level with progress tracking, channel_mix, summary), `bot_tasks` (task level with SLA, output, escalation state). The baseline work is ensuring the executor correctly creates and updates these records as the graph executes.

---

## Baseline Components

### 1. LangGraph Runtime with Checkpoints

The two-tier architecture is already built:
- **Outer graph** (fixed): normalize → clarify → plan → validate → execute → finalize
- **Inner graph** (dynamic): compiled from PlanSpec by M3's DAG compiler

All state persisted to Postgres via `AsyncPostgresSaver`. The baseline hardening needed:

**State design for the inner graph:**
- `task_results: dict[str, SkillResult]` — results keyed by task_id (already exists as `TaskContext`)
- `errors: list[ErrorRecord]` — error accumulator for partial failure handling (NEW)
- `pending_approvals: list[ApprovalRequest]` — interrupt state (already exists)
- `batch_progress: dict[str, BatchStatus]` — per-batch completion tracking (NEW)

### 2. Batch Orchestration

**How batches execute:**

```
Batch starts
    │
    ├─ Fan-out: all tasks in the batch dispatch in parallel (Send API)
    │   ├─ task t1 → adapter.invoke() → result
    │   ├─ task t2 → adapter.invoke() → result
    │   └─ task t3 → adapter.invoke() → ERROR
    │
    ▼
Completion detection: all parallel branches converge (LangGraph handles this natively)
    │
    ▼
Reconciliation: check results + errors
    │
    ├─ All succeeded → proceed to next batch
    ├─ Some failed → partial results available, user notified
    │   User can: retry failed tasks, skip them, or abort
    └─ All failed → batch failed, escalate to user
    │
    ▼
LLM Synthesis: summarize batch outcomes
    "Researched 3 competitors. Found pricing data for Alpha and Gamma.
     Beta's website was unreachable (will retry)."
```

**Reconciliation pattern:** Each task writes to either `task_results` (success) or `errors` (failure) in the graph state. The reduce node after a batch reads both and decides: proceed, retry, or escalate. This is the critical gap — today a single failure kills the entire run.

**Progress tracking:** Each task completion fires an SSE event with `run_id`, `batch_id`, `task_id`, status, and a running count (`completed: 2/3`). The frontend renders this as a progress bar per batch.

### 3. Error Recovery & Retry

Today: one node fails → entire run fails. Baseline fixes this.

**Per-node retry with backoff:**

| Attempt | Wait | When to retry |
|---------|------|---------------|
| 1st retry | 1s | Transient errors (timeout, 5xx, rate limit) |
| 2nd retry | 3s | Same transient errors |
| 3rd retry | 10s | Last attempt |
| After 3 failures | — | Try fallback skill (if M2 retrieval returned alternatives). If no fallback: write to `errors`, continue with remaining tasks. |

**Error classification** (determines retry behavior):

| Category | Retry? | Example |
|----------|--------|---------|
| `TRANSIENT` | Yes, with backoff | Timeout, 5xx, rate limit |
| `PERMANENT` | No — skip or fallback | 4xx, skill not found, invalid input |
| `USER_FIXABLE` | No — interrupt, ask user | Auth expired, missing permission |
| `CONSENT_DENIED` | No — respect user's decision | User denied the action |

**Partial plan recovery:** Completed nodes are preserved. Only the failed subtree is affected. The user sees: "2 of 3 research tasks completed. Beta research failed (website unreachable). Continue with what we have, or retry?"

### 4. Adapter Framework

The adapter layer sits between the executor and external systems. Each task node calls an adapter, which calls the actual skill.

**Adapter types already built:**

| Adapter | What it does |
|---------|-------------|
| `DirectAdapter` | Routes to Python handler functions (68+ handlers) |
| `MCPAdapter` | Calls MCP servers via `MultiServerMCPClient` |
| `MarketplaceAdapter` | Calls Composio tools |
| `AgentAdapter` | Manages external agent lifecycle |
| `ComposedAdapter` (NEW, from M6) | Loads a saved PlanSpec and submits it as a sub-graph |

**Baseline hardening:** Each adapter invocation needs:
- Timeout enforcement (configurable per skill, default 30s)
- Error wrapping (adapter errors → typed `ErrorRecord` with category)
- Telemetry emission (skill_id, latency_ms, success/failure → feeds M2 health metrics)

### 5. Execution Observability

What the executor tracks at baseline (feeds into AKN's debug-logs and analytics-setup):

**Per-task:** skill_id, adapter_type, input hash, start/end time, duration_ms, status, error category (if failed), token usage (if LLM call)

**Per-batch:** batch_id, task count, completed count, failed count, total duration, synthesis output

**Per-run:** run_id, user_id, plan_id, total duration, total cost estimate, batch count, final status, error summary

All emitted as SSE events (already have ~24 event types) AND persisted to `agent_run_events`. The debug span model (from AKN's debug-logs baseline) instruments each of these.

### 6. Run Cancellation

User can cancel a run from the UI at any level:

| Cancel level | What happens |
|-------------|-------------|
| Cancel task | Mark task as `cancelled`. Other tasks in the batch continue. |
| Cancel batch | Mark all pending tasks in the batch as `cancelled`. Completed tasks keep their results. |
| Cancel run | Mark all pending batches/tasks as `cancelled`. Finalize with partial results. |

In-flight adapter calls are not force-killed (they may have side effects). Instead, the executor checks a `cancelled` flag before dispatching each node. If cancelled, skip the node and write a `CANCELLED` status.

---

## What Counts as Refinement

| Refinement | Why deferred |
|------------|-------------|
| Resource limits (max concurrent nodes, max API calls) | Not hitting limits at current scale |
| Execution isolation (failures in one user's plan affect others) | Single process, low concurrency today |
| Conditional routing (branch based on intermediate results) | Planner can express this as separate tasks with dependencies |
| Execution analytics dashboards | Data collection is baseline; visualization is refinement |
| Warm restart improvements (smarter checkpoint selection) | Current checkpoint-based restart works |
| SLA-driven execution (deadline-aware scheduling) | SLA tracking exists; adaptive scheduling is refinement |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| Compiled StateGraph | Task Decomposition (M3, RKP) | Executor runs what the compiler produces |
| Adapter framework | Core Platform (M1, KPR) | Adapter invocations for each task node |
| Skill registry | Skill Management (M2, AKN) | Skill lookup, health metric updates |
| Debug span model | Debug Logs (AKN) | Instrumentation points for observability |
| SSE streaming | Already built | Real-time progress to mobile |
| Postgres checkpointer | Already deployed | Durable state for interrupts |

## Effort Estimate

| Component | Effort |
|-----------|--------|
| Run→batch→task ID management (create/update records correctly) | 2-3 days |
| Batch reconciliation (partial failure handling, error accumulator) | 2-3 days |
| Per-node retry with backoff + error classification | 2-3 days |
| Fallback skill invocation on repeated failure | 1-2 days |
| Timeout enforcement per node | 1 day |
| Run/batch/task cancellation | 2 days |
| Adapter telemetry emission (feeds M2 health + AKN debug) | 1 day |
| Progress tracking SSE events (completed X of N per batch) | 1 day |

**Total: ~12-16 dev days to reach baseline.**

---

## Key Concepts Explained

**Why batches, not just tasks?**
A flat list of tasks doesn't capture the structure of a plan. "Research 3 competitors" is one logical unit — a batch. "Call 3 partners" is another. Batches give the UI a natural grouping for progress display, and the executor a natural boundary for reconciliation ("all research done → start calling"). Without batches, the user sees 7 individual tasks with no structure.

**Why partial failure matters:**
Today, if task 2 of 7 fails, the entire run dies. But tasks 1, 3-7 might be perfectly fine. The user loses everything because of one failure. With partial failure handling, the user gets results from 6 tasks and a clear message about the 1 that failed, with options to retry or skip. This is the difference between a brittle demo and a usable product.

**Why error classification, not just try/catch?**
A timeout and a 403 are both "errors" but need completely different handling. Timeouts should retry (transient). 403s should ask the user to re-auth (user-fixable). Invalid inputs should skip the task (permanent). Without classification, the executor either retries everything (wasteful) or retries nothing (fragile).
