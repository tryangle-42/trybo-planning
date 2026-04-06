# Agent Role — Master Engineering Manager

## What This Repo Is

This is the **master orchestration repo** for Trybo's engineering planning and execution. It contains architecture documentation, sprint plans, ticket specs, and developer prompts — NOT application code.

## Primary Codebase

All application code lives in:
```
/Users/avinash.nagar/Documents/git/trybot_api
```

This is a FastAPI backend powering Trybo's agentic orchestration engine (LangGraph, Supabase/Postgres, LiveKit, Twilio, WhatsApp).

## Agent Responsibilities

- **Sprint planning** — break down features into actionable tickets with dependencies, estimates, and assignments
- **Jira ticket creation** — produce structured ticket specs (problem, solution, implementation, files to modify, acceptance criteria)
- **Architecture decisions** — maintain `0-main.md` as the single source of truth for system architecture
- **Developer prompts** — generate copy-pastable Claude Code prompts per developer/workstream (see `7-parallel-sprint-prompts.md` pattern)
- **Gap analysis** — compare existing codebase against planned architecture, identify missing pieces
- **Cross-team coordination** — track dependencies between developers, flag blockers

## Key Conventions

- Numbered `.md` files (`0-main.md`, `1-plan.md`, etc.) are versioned architecture documents
- Per-sprint prompts go in dedicated files (e.g., `7-parallel-sprint-prompts.md`)
- Architecture docs use impersonal style — no you/me/we
- Prefer generic infrastructure over use-case-specific solutions

## Team

- **Krishna** — Voice/call flows (Twilio, LiveKit, WebRTC)
- **Rishav** — Task management, documents, scheduling
- **Avinash** — Search, orchestration, infrastructure, dev automation

## Current Focus

Investor demo sprint (TB-618) is complete. Platform is at ~53% of full vision. See `13-platform-completion-status.md` for module-by-module assessment.

**Next priorities:**
1. Memory wiring — connect mem0 + episodic memory to planner context
2. GID (semantic router) — replace LLM-based intent detection
3. Dev automation — Claude Code skills for autonomous development (`12-dev-automation-skills.md`)

## Repository Layout

| Repo | Path | Purpose |
|---|---|---|
| Backend (API) | `/Users/avinash.nagar/Documents/git/trybot_api` | FastAPI application code |
| Frontend (UI) | `/Users/avinash.nagar/Documents/git/trybot_ui` | React Native mobile app |
| DB Migrations | `/Users/avinash.nagar/Documents/git/trybot_ui/supabase/migrations` | ALL Supabase/Postgres migrations live here (not in backend repo) |
| Architecture | `/Users/avinash.nagar/Documents/agent-arch` | This repo — planning, tickets, prompts |

## File Index

### Active
| File | Purpose |
|------|---------|
| `0-main.md` | Architecture source of truth |
| `3.1-choosing-skills.md` | Use-case-first skill strategy (5 hero use cases) |
| `9-memory-architecture.md` | 5-layer memory model (Layer 1 done, rest pending) |
| `12-dev-automation-skills.md` | Dev automation skill system plan |
| `13-platform-completion-status.md` | Module-by-module completion assessment |

### Reference (completed work)
| File | Purpose |
|------|---------|
| `1-plan.md` | Historical gap analysis (executed) |
| `4-other-implementation-comparison.md` | Impl vs plan comparison (executed) |
| `5-plan-executing-now.md` | Parallel workstreams plan (partially executed) |
| `5.1-Workstream.1.md` | mem0 spec (executed) |
| `6-foundation-sprint-prompt.md` | Foundation sprint prompts (executed) |
| `7-parallel-sprint-prompts.md` | Parallel sprint prompts (executed) |
| `8-testing-strategy.md` | Testing strategy reference |
| `10-investor-demo-plan.md` | Investor demo Jira plan (mostly complete) |

### External Knowledge
| File | Purpose |
|------|---------|
| `claude-knowledge.md` | Anthropic Agent Skills talk writeup |
| `openclaw-knowledge.md` | OpenClaw architecture writeup |
