# Trybo Planning Repo

## What This Repo Is

Engineering planning scratchpad for Trybo. Meeting notes, sprint plans, gap analysis, milestone tracking, per-owner baseline definitions. This is the **working repo** — docs iterate frequently. Not the source of truth for architecture (that's trybo-arch).

## Repo Ecosystem

| Repo | Purpose | Location |
|------|---------|----------|
| **tryangle-42/trybo-planning** (this repo) | Planning scratchpad — meetings, sprints, milestones, gap analysis | `/Users/avinash.nagar/Documents/git/trybo-planning` |
| **tryangle-42/trybo-arch** | Gold standard finalized architecture — module definitions, memory model, skill strategy | `/Users/avinash.nagar/Documents/git/trybo-arch` |
| **tryangle-42/trybot-api** | Backend codebase — FastAPI, LangGraph, Python | `/Users/avinash.nagar/Documents/git/trybot_api` |
| **tryangle-42/Agentic_Orchestration_Components** | Shared architecture components (KPR's contributions) | — |

## Key Files in This Repo

| File | What it is |
|------|-----------|
| `sprint-plan.md` | Active sprint plan — weekly, 80/20 GTM/arch split, team assignments |
| `gtm-milestone-plan.md` | 9 GTM milestones with owners, estimates, dependencies, demo schedule |
| `baseline-to-user-impact.md` | Maps every baseline component to user-facing capability |
| `full-roadmap-planning.md` | Master roadmap with all module assignments |
| `gap-analysis/` | GTM gap analysis, prosumer analysis, RKP items |
| `meetings/` | Meeting notes (dated) |
| `roadmap-plan-akn/` | AKN's per-module baseline definitions |
| `roadmap-plan-rkp/` | RKP's per-module baseline definitions |
| `roadmap-plan-kpr/` | KPR's per-module baseline definitions |
| `archive/` | Historical planning docs — reference only, not active |

## Rules

- Planning docs live HERE, not duplicated to trybo-arch
- Finalized architecture gets PROMOTED to trybo-arch via reviewed PR
- Share links to this repo directly for team discussions
- Meeting notes go in `meetings/YYYY-MM-DD.md`
