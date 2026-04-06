# System Design Baseline — Baseline Definition

> Owner: AKN
> Scope: Architecture documentation standards, living docs, new-dev onboarding readiness
> Context: This repo (`agent-arch`) contains the system design. This doc defines what "complete enough" looks like.

---

## What This Is

The minimum set of architecture documentation such that:
1. A new developer can understand the system and start contributing within 1 day
2. Any module can be worked on independently using only its module definition + interface contracts
3. The architecture docs stay current with the actual codebase (not diverge into fiction)

## Current State

| What Exists | What's Missing |
|-------------|----------------|
| `0-main.md` — full architecture vision (finalized) | No "start here" onboarding guide |
| `15-module-ownership-map.md` — 11 modules, boundaries, interfaces, dependencies | No runbook for common operations (deploy, debug, test) |
| `9-memory-architecture.md` — 5-layer memory model | No API contract docs (what endpoints exist, what they accept/return) |
| `13-platform-completion-status.md` — component-level completion tracking | No data model doc (what's in the DB beyond raw schema SQL) |
| `Agent.md` — repo role and file index | Completion status last updated 2026-03-30, may drift |
| `3.1-choosing-skills.md` — skill strategy | No sequence diagrams for core flows |
| DB schema at `docs/entire-schema.sql` | Raw SQL, no annotated data model |
| `AGENTS.md` in backend repo — code conventions | Not linked from architecture docs |

---

## Baseline Definition

**The line:** A developer joining the team can read 4 documents in sequence and be productive. Those 4 documents are current, consistent with each other, and consistent with the actual codebase. Anything beyond this is refinement.

### The 4-Document Onboarding Path

```
1. START-HERE.md        →  "What is this system? Who works on what? How do I get set up?"
2. 0-main.md            →  "How does the architecture work?" (already exists)
3. 15-module-ownership-map.md  →  "What does my module own? What are the interfaces?" (already exists)
4. RUNBOOK.md            →  "How do I build, test, deploy, and debug?"
```

### Baseline Components

#### 1. START-HERE.md (new)

A single-page onboarding entry point:

```markdown
# Trybo — Start Here

## What is Trybo?
[2-3 sentences. Mobile-first personal assistant OS. Agent-of-Agents architecture.]

## Architecture in 30 Seconds
[The simplified flow diagram from 15-module-ownership-map.md: User → GID → Planner → Executor → Skills → Result]

## Repos
| Repo | What | Tech |
|------|------|------|
| `trybot_api` | Backend — agentic engine, skills, APIs | FastAPI, Python, LangGraph, Supabase |
| `agent-arch` | Architecture docs, planning, Jira orchestration | Markdown |
| [mobile repo] | React Native mobile app | React Native |

## Team & Ownership
[Condensed from 15-module-ownership-map.md ownership section]

## Reading Order
1. This doc (you're here)
2. `0-main.md` — full architecture
3. `15-module-ownership-map.md` — your module's definition
4. `RUNBOOK.md` — how to run things

## Key Architecture Decisions
[Top 5 non-obvious decisions with one-line rationale, linking to ADRs when they exist]
- Two-tier LangGraph executor (why: outer lifecycle stability + inner dynamic flexibility)
- `interrupt()` for consent (why: native LangGraph pattern, no custom polling)
- mem0 for semantic memory (why: built-in dedup, pgvector backend)
- `skill_data_store` generic table (why: no feature-specific tables, skills own their data shape)
- `semantic-router` for GID (why: sub-10ms routing without LLM call)
```

#### 2. RUNBOOK.md (new)

Operational guide for the backend:

```markdown
# Runbook

## Local Setup
- Clone, venv, install, env vars, run

## Running Locally
- API server command
- How to trigger a test run via curl/Postman

## Testing
- pytest patterns, test JWT, live API testing

## Debugging
- Where logs are (structlog, local_performance.log)
- How to trace a request (debug endpoints from debug-logs baseline)
- Common failure modes and what to check

## Database
- How to connect (psql with $SUPABASE_DB_URI)
- Schema overview (link to docs/entire-schema.sql)
- How to run migrations
- Key tables and what they contain (annotated, not raw SQL)

## Deployment
- How code gets to production
- Environment variables that matter
- Feature flags

## Common Tasks
- "I need to add a new skill" → steps
- "I need to add a new MCP server" → steps
- "I need to modify the planner prompt" → where to look
- "A skill is failing" → debug checklist
```

#### 3. Data model annotation (enhancement to existing)

The raw `docs/entire-schema.sql` exists but is unreadable for onboarding. Baseline adds an annotated version:

```markdown
# Data Model Guide

## Core Tables by Domain

### Orchestration (owned by M5)
| Table | Purpose | Key Columns | Relationships |
|-------|---------|-------------|---------------|
| `agent_runs` | One row per execution | status, run_id, user_id | → agent_run_events, bot_tasks |
| `agent_run_events` | Event timeline per run | event_type, payload, seq | → agent_runs |
| `bot_tasks` | Per-task SLA tracking | sla_status, deadline_at | → agent_runs |

### Memory (owned by M7)
...

### Communication (owned by M10)
...
```

Not a full ERD — just enough to understand "what table holds what" without reading SQL.

#### 4. Core flow sequence diagrams (3 flows)

Three ASCII sequence diagrams for the most important flows:

**Flow 1: Chat message → plan → execute → response**
```
User → API → GID → [FAST_PATH | PLANNER]
                         │
                    Planner → PlanSpec
                         │
                    Validator → OK
                         │
                    Executor → compile graph
                         │
                    ┌─ Node 1 (no consent) → adapter.invoke() → result
                    ├─ Node 2 (consent) → interrupt() → [user approves] → resume → adapter.invoke()
                    └─ Node 3 (parallel) → ...
                         │
                    Finalize → write episode, extract facts, respond
                         │
User ← API ← SSE events throughout
```

**Flow 2: Inbound call → agentic handling**

**Flow 3: Scheduled job → execution**

#### 5. Currency mechanism

A process to keep docs current:

- `13-platform-completion-status.md` gets a `Last verified: YYYY-MM-DD` stamp per module
- When a significant PR merges that changes module boundaries or interfaces, the PR description must note which architecture doc needs updating
- Monthly 30-min "architecture sync" — walk through completion status, flag drift

This is manual at baseline. The self-learning repo (foundation-setup-claude.md) automates it.

---

## What Counts as Refinement (explicitly deferred)

| Refinement | Why deferred |
|------------|-------------|
| Interactive architecture diagrams (Mermaid, D2) | ASCII diagrams are sufficient. Tooling is overhead. |
| Auto-generated API docs (Swagger/OpenAPI) | FastAPI generates these by default at `/docs`. Link to it. |
| Video onboarding walkthrough | Text is searchable and maintainable. Video is not. |
| Architecture fitness functions (automated tests for arch rules) | Manual review is fine at 3-5 devs. |
| Full ERD diagram | Annotated table list covers 90% of needs. |
| Per-sprint architecture changelog | Monthly sync is sufficient at current pace. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| `0-main.md` finalized | Already done | Core architecture reference |
| `15-module-ownership-map.md` | Already done | Module definitions |
| Backend repo access | `trybot_api` | Runbook references actual commands and paths |
| Debug endpoints | Debug Logs baseline (AKN) | Runbook references debug tools |

## Effort Estimate

- `START-HERE.md`: 0.5 day
- `RUNBOOK.md`: 1 day (requires walking through actual setup process)
- Data model annotation: 1 day
- 3 sequence diagrams: 0.5 day
- Currency mechanism setup: 0.5 day

**Total: ~3-4 dev days to reach baseline.**
