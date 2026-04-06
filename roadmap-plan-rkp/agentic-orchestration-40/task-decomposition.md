# Task Decomposition & Planning — Baseline Definition

> Owner: RKP
> Module mapping: M3 (Task Decomposition)
> Current completion: ~33% — LLM Planner, validation, clarification done. GID, fast path, algorithmic decomposition, templates, confidence scoring missing.

---

## What This Is

Turns a user's natural language goal into a validated, executable plan. The module answers: "What needs to happen, in what order, using which skills?"

Four sub-systems:

1. **LLM Planner + Clarification Engine** — generate a PlanSpec from the user's goal, asking for missing info first
2. **DAG Compilation** — compile the PlanSpec (JSON task graph) into a runnable LangGraph StateGraph
3. **Interrupt System** — approvals, clarifications, OAuth, agent questions — all via durable checkpoints
4. **Plan Validation & Safety** — verify the plan is structurally sound before execution

## Current State

| Built | Not Built |
|-------|-----------|
| LLM Planner (`llm_planner.py`) — generates PlanSpec via structured JSON output | GID semantic router — intent detection is LLM-based today (~200ms per routing decision, should be <10ms) |
| Plan validation — skill_id existence, schema validation, cycle detection | Fast path — all intents go through full planner, even "call John" |
| Clarification engine — interrupt-based, multi-turn | Algorithmic decomposition — all decomposition is LLM-based |
| Partial re-planning — can generate new plans, but can't modify a subtree of an in-flight plan | Plan templates, confidence scoring, explainable plans |

---

## Baseline Definition

**The line:** A user's message is routed in <10ms (not via LLM). Simple intents bypass the planner entirely. Complex intents produce a validated PlanSpec with correct dependencies, consent flags, and skill references. The planner asks for missing information before planning. Plans are compiled into executable LangGraph graphs. All of this survives process restarts via durable checkpoints. Anything beyond this is refinement.

---

## Baseline Components

### 1. GID — Global Intent Detector (Semantic Router)

Today every user message costs an LLM call (~200ms, ~$0.001) just to decide what to do with it. GID replaces this with embedding-based classification in <10ms using `semantic-router`.

**How it works:** Pre-defined "routes" are embedded (e.g., "call someone," "search the web," "schedule a meeting"). The user's message is embedded at runtime and compared to route embeddings. The closest match determines the routing decision.

**Three routing outcomes:**

| Decision | When | What happens |
|----------|------|-------------|
| `FAST_PATH` | High-confidence, single-skill intent ("call John") | Bypass planner. Go directly to executor with a single-node plan. |
| `PLANNER` | Complex or multi-step intent ("research competitors and call partners") | Full LLM planner generates PlanSpec. |
| `CLARIFY` | Ambiguous or missing critical info ("book a flight") | Ask for clarification before planning. |

**Why this matters:** At 100 user messages/day, LLM-based routing costs ~$3/month. At 10,000/day, it's $300/month — and every message has a 200ms tax. GID eliminates both. The `semantic-router` library handles this with a single embedding model and cosine similarity.

### 2. LLM Planner + Clarification Engine

The planner takes a user goal + assembled context (from M7 memory) + skill catalog (from M2 retrieval) and produces a PlanSpec — a JSON task graph with dependencies, skill references, and consent flags.

**The planning loop:**

```
User message
     │
     ▼
GID routes to PLANNER
     │
     ▼
Clarification check: does the planner have enough info?
     │
     ├─ No → interrupt() → ask user → resume with answer → re-check
     │        (max 3 rounds, then plan with what you have)
     │
     └─ Yes → LLM generates PlanSpec (structured output via Pydantic)
                  │
                  ▼
              Validate → if invalid, retry with error context (max 3 retries)
                  │
                  ▼
              Compile into LangGraph StateGraph → execute
```

**What the planner sees:** conversation history + relevant memories + top-K skills from M2 retrieval + persona config (from persona management). The planner generates structured JSON output enforced via `with_structured_output(PlanSpec)`.

**Re-planning:** When a step fails or the user modifies the request mid-execution, the planner can generate a new plan that preserves completed node results and only replans the affected subtree. At baseline, this is a full replan with completed results injected as context. True partial replanning (modifying the live graph) is refinement.

### 3. DAG Compilation

Compiles a validated PlanSpec into a LangGraph `StateGraph` — the executable graph that M5 runs.

**What the compiler does:**
- For each task in the PlanSpec: add a node to the graph with the appropriate wrapper (tool node, consent node, map node, condition node)
- For each dependency: add an edge (upstream → downstream)
- Independent tasks (no dependency edges between them) become parallel branches — LangGraph auto-detects fan-out
- Map nodes use the `Send` API to dispatch N items to the same node function
- Compile with Postgres checkpointer for durability

**Why this lives in Task Decomposition, not Execution:** The compiler translates the *plan's intent* into an executable form. It's the boundary between "what should happen" (M3) and "make it happen" (M5). The compiler understands plan semantics (dependencies, consent requirements, map configs); the executor just runs the compiled graph.

**Mixed node types the compiler handles:**

| Node type | What it does | Wrapper behavior |
|-----------|-------------|-----------------|
| `tool` | Invoke a skill via adapter | Call M2 registry → M1 adapter → return result |
| `consent` | Require user approval before proceeding | `interrupt()` → checkpoint → wait for `Command(resume=)` |
| `map` | Process a variable-length list in parallel | `Send` API → N parallel invocations → collect results |
| `condition` | Branch based on intermediate results | Evaluate condition → route to one of N downstream nodes |

### 4. Plan Validation & Safety

Every PlanSpec passes 5 deterministic checks before compilation (no LLM needed):

| Check | What it catches |
|-------|----------------|
| **Skill existence** | Plan references a skill that doesn't exist in registry |
| **Schema conformance** | Input parameters don't match the skill's declared input schema |
| **Cycle detection** | Topological sort fails — circular dependencies |
| **Consent flag cross-check** | Plan says `requires_consent: false` but registry says `true` — plan can't escalate privileges |
| **Reference resolution** | `$memory.X.Y` references point to nodes that exist and are upstream in the dependency graph |

On validation failure: retry with error context injected into planner prompt (max 3 retries). If still failing after 3 retries: graceful degradation — remove the invalid node + cascade-remove its dependents, and explain to the user what was dropped.

### 5. Interrupt System (Durable Checkpoints)

All pauses — approvals, clarifications, OAuth flows, agent questions — use LangGraph's `interrupt()` + Postgres checkpointing.

**How it works:** When a node needs to pause (user approval, OAuth redirect, clarification), it calls `interrupt(value)`. The entire graph state is checkpointed to Postgres. No compute is consumed while waiting — it's just rows in a database. When the user responds, `Command(resume=answer)` resumes from the exact point.

**Types of interrupts at baseline:**

| Type | Trigger | Resume with |
|------|---------|-------------|
| **Consent approval** | Skill has `requires_consent: true` | User's decision (approve/edit/deny) |
| **Clarification** | Planner needs more info | User's answer |
| **OAuth** | Skill needs auth token | OAuth token from mobile app |
| **Agent question** | Mid-execution, agent needs user input | User's answer |

**Durability:** Survives process restarts, deployments, even server migrations. A user can approve hours or days later.

**Stale data concern:** If a user approves 6 hours after the interrupt, upstream data (flight prices, availability) may be stale. At baseline, we add a `stale_after` TTL to approval artifacts. On resume, if TTL has expired, the node re-fetches upstream data before proceeding. This is a simple timestamp check, not a full re-plan.

---

## What Counts as Refinement

| Refinement | Why deferred |
|------------|-------------|
| Algorithmic decomposition (decompose known patterns without LLM) | LLM planner handles all patterns today. Algorithmic decomposition saves cost/latency but isn't blocking. |
| Plan templates (pre-built PlanSpec structures for common goals) | Useful for speed but planner generates good plans without them. |
| Confidence scoring (planner self-rates plan quality) | Needs feedback data to calibrate. Build feedback collection first (see agent-intelligence.md). |
| Explainable plans (human-readable explanation alongside PlanSpec) | Nice UX but not functionally blocking. |
| Multi-turn planning (reference yesterday's trip plan) | Requires episodic memory (M7 Layer 3) to be wired. |
| Plan optimization (minimize API calls, maximize parallelism) | Planner generates "good enough" plans. Optimization is a quality-of-life improvement. |
| Partial re-planning (modify live graph subtree without full replan) | Full replan with completed results as context is sufficient at baseline. |
| Abandoned workflow cleanup (cron for stale interrupted graphs) | Manual cleanup works at current scale. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| Skill catalog (find_skills retrieval) | Skill Management (M2, AKN) | Planner needs to know what skills exist |
| Assembled context (conversation + memories) | Memory (M7, KPR) | Planner needs user context |
| Persona config | Persona Management (RKP) | Planner behavior varies by persona |
| `langgraph-checkpoint-postgres` | Already deployed | Durable checkpoints for interrupts |
| `semantic-router` library | New dependency | GID embedding-based routing |

## Effort Estimate

| Component | Effort |
|-----------|--------|
| GID semantic router (routes + embedding + integration) | 2-3 days |
| Fast path (single-skill bypass of planner) | 1-2 days |
| Clarification max-rounds cap + improved UX | 1 day |
| DAG compiler hardening (mixed node types, error edges) | 2-3 days |
| Stale data detection on interrupt resume | 1 day |
| Re-planning with completed results as context | 2 days |

**Total: ~9-12 dev days to reach baseline.**

---

## Key Concepts Explained

**What is a PlanSpec?**
A JSON document that describes a multi-step plan as a directed acyclic graph (DAG). Each node is a task (skill to invoke, inputs, dependencies). The planner *generates* it, the validator *checks* it, the compiler *compiles* it into a runnable graph, the executor *runs* it.

**Why `semantic-router` instead of an LLM for intent detection?**
An LLM call for "should I plan or fast-path this?" costs ~200ms and ~$0.001. Embedding-based classification costs <10ms and ~$0. At scale, the LLM approach is 20x slower and significantly more expensive. The accuracy tradeoff is minimal — intent routing is a classification problem, not a generation problem.

**Why DAG compilation is not trivial:**
A PlanSpec says "task B depends on task A." But what if task A fails? The compiled graph needs conditional edges: on success → proceed to B, on failure → route to error handler. Map nodes need the `Send` API. Consent nodes need `interrupt()`. The compiler translates *plan semantics* into *graph mechanics*.
