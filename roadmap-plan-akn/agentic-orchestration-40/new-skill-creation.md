# New Skill Creation — Baseline Definition

> Owner: AKN
> Module mapping: M6 (New Skill Creation)
> Current completion: 0% — nothing built
> Research basis: Voyager, SAGE, MCP-Zero, Composio/Gentoro, Docker Dynamic MCP, E2B/microsandbox

---

## What This Is

The system's ability to expand its own capabilities. Owns the full pipeline from discovery to promotion:

```
DISCOVER              GENERATE             VALIDATE             PROMOTE
(find new APIs,  →   (parse specs,    →   (quality gates,  →   (call M2's
 poll registries,     create skills,       sandbox test,        register()
 user composes)       LLM refine)          contract test)       endpoint)
```

**Triggers:** Cron jobs (daily/weekly) and developer-initiated actions. Never per-request — the pipeline takes minutes, not milliseconds.

**Interface with Skill Management (M2):** This module produces verified skills. M2's `register()` endpoint adds them to the live registry and embedding index. M6 creates, M2 stores.

## Current State

| What Exists | What's Missing |
|-------------|----------------|
| LangGraph subgraph composition (M5) | No user-facing skill composition |
| Composio marketplace adapter (M1) | No discovery of new tools at runtime |
| `scheduled_jobs` table | No "user-defined automation" abstraction |
| PlanSpec schema (multi-step task graph) | Not persisted as a reusable skill |

---

## Baseline Definition

**The line:** Users can create reusable automations from chat. The system can poll for new MCP servers and generate skills from OpenAPI specs. All generated skills pass quality gates and sandbox testing before going live. Anything beyond this is refinement.

---

## Baseline Components

### 1. Skill Composition — PlanSpecs as Reusable Skills

The simplest form of skill creation: a PlanSpec that gets saved, named, and made reusable.

```
User: "Every morning at 7am, check weather in Delhi, check my calendar,
       summarize my unread emails, and send me a WhatsApp briefing"

Planner generates PlanSpec:
  Task 1: weather.get(location="Delhi")
  Task 2: calendar.query(date="today")        [parallel with 1]
  Task 3: email.summarize(count=5)             [parallel with 1,2]
  Task 4: llm.synthesize(inputs=[1,2,3])       [depends on 1,2,3]
  Task 5: whatsapp.send(to="self", message=…)  [depends on 4]

System: "Want me to save this as a reusable automation?"
User: "Yes, call it Morning Briefing"

Saved as:
  skill_id: "user.automation.morning_briefing"
  adapter_type: "composed"
  trigger: { cron: "0 7 * * *" }
```

A `composed_skills` table stores the PlanSpec, trigger config, and runtime stats. The composed skill registers in M2's registry with `adapter_type = "composed"`. When invoked, the `ComposedAdapter` loads the PlanSpec and submits it to M5 — a plan within a plan.

Users can also describe automations in natural language. The planner generates the PlanSpec, the user confirms, and a skill handler (`automation.create`) persists it + creates a `scheduled_job` if cron-triggered.

### 2. Discovery — MCP Registry + OpenAPI Crawling

**MCP Registry Poller:** A cron job (daily/weekly) that:
1. Queries `registry.modelcontextprotocol.io/v0/servers` (REST API, cursor-paginated)
2. Filters by capability tags relevant to Trybo (communication, calendar, search, productivity)
3. Compares against already-registered servers
4. Stores new finds in a `skill_candidates` table (`status: pending`)

**OpenAPI Spec Crawling:** Developer-initiated. A developer provides a spec URL, the system fetches and queues it.

**Both go through the same pipeline:** discovery → generation → validation → promotion. Nothing is auto-registered — a developer reviews and approves candidates before generation begins.

### 3. Generation — Specs to Skills

**From MCP servers:** Connect to the server, list its tools, auto-generate a `SkillDefinition` per tool from the MCP schema.

**From OpenAPI specs:** Parse each operation (endpoint + method), extract input/output schemas, set `requires_consent = true` for mutating methods (POST, PUT, DELETE, PATCH), generate a thin HTTP handler template.

**The critical step: LLM refinement.** Raw auto-generation produces machine-readable but planner-hostile names. `GET /api/v1/weather/{city}` becomes "Get current weather for a city." Research (Composio/Gentoro) found this single step raises planner success rate from ~33% to ~100%. Not optional.

**MCP connection management:** A `DynamicMCPManager` wraps the MCP client with `add_server()` / `remove_server()`. The MCP protocol natively supports this via `notifications/tools/list_changed`. Current limitation: `langchain-mcp-adapters` doesn't support adding servers after init — workaround is wrapping it or using Docker's MCP Gateway as a proxy.

### 4. Quality Gates

Every generated skill passes three gates before going live:

```
Gate 1: Static Analysis     →  Schema valid? Required fields present? Description > 20 chars?
Gate 2: Dry Run             →  Execute with sample inputs in sandbox. No crash? Valid JSON output?
Gate 3: Contract Test       →  3 sample inputs. Outputs consistently match declared schema?
         │
         ▼
Status: CANDIDATE           →  Available for sandboxed execution only
         │
         ▼ (after 10 successful real invocations)
Status: VERIFIED            →  Promoted to M2's live registry. Sandbox removed.
```

**Key principle:** The quality gate runs in a separate process from the generator. The generator cannot approve its own output — same reason the PR author doesn't review their own PR.

### 5. Safety Sandbox

Generated and third-party skills execute in isolation until verified.

**Trust tiers:**

| Tier | Skills | Environment |
|------|--------|------------|
| Trusted | Core platform skills (seed registry) | Direct execution |
| Verified | Generated skills that passed gates + 10 successful runs | Direct execution |
| Sandboxed | New generated or third-party skills | Docker container |

**Docker sandbox:** Short-lived container, 256MB memory, 50% CPU, 30-second hard timeout, network restricted to whitelisted endpoints. Skill code and input passed as env vars. Container destroyed after execution. Crashes/timeouts have zero impact on the host.

**Why Docker over microVM or WASM:** Already available everywhere, <90ms startup, sufficient isolation for HTTP-calling skills. Upgrade to microVM (E2B, microsandbox) for truly arbitrary third-party code. WASM only works for code compiled to it.

### 6. Composed vs Third-Party Coexistence

A composed skill and a discovered third-party skill can cover the same capability. Both live in M2's registry, same hierarchy, same retrieval pipeline — differentiated by `adapter_type`.

**Example:** "Morning Briefing" in three forms:

| Form | adapter_type | Latency | Visibility |
|------|-------------|---------|------------|
| Primitives (plan from scratch) | — | Highest — planning + 5 calls | Full |
| Composed (saved PlanSpec) | `composed` | Medium — 5 calls, no planning | Full — user can edit |
| Third-party black-box | `mcp` / `direct` | Lowest — 1 call | None — output only |

**Selection at baseline:**
1. User-created composed skills win for that user (explicit preference)
2. Otherwise, M2's health-metric sorting picks the healthiest option
3. Black-boxes trade transparency for speed — can't debug, can't customize, data goes to third party

**Deeper tradeoffs (relevant when AgentRank is built):**

| Factor | Composed | Black-box |
|--------|---------|-----------|
| Debuggability | Trace each sub-skill | Output only |
| Partial failure | 3/5 steps succeed = partial result | All or nothing |
| Customizability | User swaps/reorders steps | Take it or leave it |
| Data privacy | Internal APIs, data stays in system | Third party sees all input |

### 7. Pattern Extraction (Voyager Pattern)

When a multi-step plan succeeds and the user confirms it worked well, the system can extract it as a reusable template:

1. Was the plan multi-step (>2 nodes) and successful?
2. Has the user given positive feedback?
3. Extract the PlanSpec, parameterize it (LLM replaces concrete values like "Delhi" with `{destination}`)
4. Store as a template in `composed_skills` with `type: "template"`

The verification loop is what makes generated skills trustworthy. No successful execution = no template saved.

---

## Full Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    NEW SKILL CREATION (M6)                        │
│                                                                   │
│  DISCOVER                  GENERATE              VALIDATE         │
│  ┌──────────────┐         ┌──────────────┐      ┌─────────────┐  │
│  │ MCP Registry  │────┐   │ Parse spec   │      │ Static check│  │
│  │ Poller (cron) │    │   │ Extract ops  │      │ Dry run     │  │
│  ├──────────────┤    ├──▶│ LLM refine   │─────▶│ Contract    │  │
│  │ OpenAPI spec  │    │   │ Gen handler  │      │ test        │  │
│  │ (dev-trigger) │────┘   └──────────────┘      └──────┬──────┘  │
│  ├──────────────┤                                      │         │
│  │ User compose  │─── (already a PlanSpec) ────────────▶│         │
│  │ (chat)        │                                      │         │
│  └──────────────┘                                      ▼         │
│                                                  ┌───────────┐   │
│                                                  │ SANDBOX   │   │
│                                                  │ (first 10 │   │
│                                                  │  runs)    │   │
│                                                  └─────┬─────┘   │
└────────────────────────────────────────────────────────┼─────────┘
                                                         │
                                                         ▼
                                              ┌──────────────────┐
                                              │ M2: register()   │
                                              │ Live registry +  │
                                              │ embedding index  │
                                              └──────────────────┘
```

---

## What Counts as Refinement

| Refinement | Why deferred |
|------------|-------------|
| **Skill marketplace** (browse, install, rate) | Requires ecosystem |
| **Per-request discovery** (MCP-Zero pattern) | Pipeline takes minutes, not ms. Can't run mid-conversation. |
| **Multi-step demonstration learning** | Research frontier (HyperWrite, ASI) |
| **Skill versioning** | Single version per skill is fine for GTM |
| **microVM sandbox** (E2B, microsandbox) | Docker sufficient for HTTP skills |
| **WASM sandbox** | Only works for compiled-to-WASM code |
| **Self-evolving skills** (auto-improve from feedback) | Requires mature feedback loop |
| **Visual workflow builder** | Chat-based composition is baseline |
| **Cross-user skill sharing** | Automations private at baseline |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| `register()` endpoint | Skill Management (M2) | Verified skills go to live registry |
| Embedding index (pgvector) | Skill Management (M2) | Generated skills need to be searchable |
| PlanSpec schema + validation | Task Decomposition (M3) | Composed skills are PlanSpecs |
| Execution engine | Execution Engine (M5) | Composed skills execute via M5 |
| `scheduled_jobs` infrastructure | Execution Engine (M5) | Cron-triggered automations + poller |
| Docker access | DevOps | Sandbox isolation |
| Episodic memory | Memory (M7) | Pattern extraction from past executions |

## Effort Estimate

**Phased — largest module. Does not need to ship all at once.**

| Phase | Components | Effort |
|-------|-----------|--------|
| **A** | Composition + user automations (`composed_skills` table, `ComposedAdapter`, `automation.create` handler, cron integration) | 8-10 days |
| **B** | Discovery + generation (MCP poller, OpenAPI parser, LLM refinement, HTTP handler generation, `skill_candidates` table) | 7-9 days |
| **C** | Validation + sandbox (quality gates, Docker sandbox, promotion logic, pattern extraction) | 6-8 days |

**Total: ~21-27 dev days across all phases.**

---

## Key Concepts Explained

**Why "save a PlanSpec as a skill"?**
The planner already breaks multi-step requests into plans. Today those plans run once and are forgotten. Persisting the PlanSpec + giving it a name + trigger makes it a reusable automation — like a Zapier workflow, but created through conversation and executing through the same agentic engine.

**Why does LLM refinement matter for API-generated skills?**
OpenAPI specs describe APIs in machine terms: `GET /api/v1/weather/{city}`. The planner thinks in natural language. The refinement pass rewrites this into "Get current weather for a city." Without it: ~33% planner success. With it: ~100%.

**Why quality gates in a separate process?**
If the same LLM that generates a skill also evaluates it, there's a conflict of interest. Same principle as code review — the author doesn't approve their own PR.

**Why cron, not per-request?**
The generation pipeline (connect → parse → refine → validate → sandbox) takes minutes. Running this mid-conversation while a user waits is unacceptable. And connecting to untested servers during a live request is a reliability risk. Discovery runs in the background; by the time a user needs the skill, it's already registered and healthy.

---

## Research References

| Topic | Source | Key Finding |
|-------|--------|-------------|
| OpenAPI-to-tool generation | Google ADK `OpenAPIToolset` | Auto-creates tools per operation from spec |
| LLM refinement | Composio/Gentoro benchmarks | 33% → 100% planner success with refinement |
| Skill learning | Voyager (Minecraft) | Generate → verify → store loop |
| Safety sandbox | E2B, microsandbox, Docker | microVM = strongest; Docker = most practical |
| Quality gates | Dual gate architecture (2025-2026) | Validation separate from testing |
| Dynamic MCP | Docker Dynamic MCP | Protocol natively supports `tools/list_changed` |
| MCP registry | `registry.modelcontextprotocol.io` | REST API, 2000+ servers |
| User automations | n8n workflow-as-JSON | DAG with trigger + action nodes |
| Active tool request | MCP-Zero (arXiv 2506.01056) | Agent requests tools on-demand (deferred) |
