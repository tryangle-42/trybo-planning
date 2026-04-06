# Development Workflow — Baseline Definition

> Owner: RKP
> Scope: How the team uses AI coding tools for day-to-day backend + frontend development
> Extends: AKN's foundation-setup-claude.md (CLAUDE.md, skills, self-learning repo). This doc covers the workflow ON TOP of that foundation.

---

## What This Is

The daily development workflow for a team where AI coding assistants (Claude Code, Cursor) handle most implementation work. Covers three concerns:

1. **Backend Development** — how a developer (or AI) picks up a ticket, implements it, and produces a working PR
2. **Frontend Development** — same workflow adapted for React Native mobile app
3. **Coordination Agent** — how multi-repo changes (backend + frontend + architecture docs) stay synchronized

## Current State

| What Exists | What's Missing |
|-------------|----------------|
| `AGENTS.md` in backend repo (code conventions) | No `CLAUDE.md` in backend repo (AKN is creating this) |
| 10 Claude Code skills designed in `12-dev-automation-skills.md` | Skills not implemented yet |
| Jira MCP configured | No standardized dev workflow (each dev does it differently) |
| Manual code review process | No AI-assisted review (see review.md) |

---

## Baseline Definition

**The line:** Every developer follows the same workflow: Jira ticket → branch → implement (with AI) → test → PR. The workflow is documented, reproducible, and produces consistent output regardless of whether a human or AI does most of the coding. Anything beyond this is refinement.

---

## Baseline Components

### 1. Backend Development Workflow

The standard flow for a backend ticket:

```
1. PICKUP
   Read Jira ticket (via MCP) → understand requirements + acceptance criteria
   Check architecture docs → which module, which files, what constraints
   Create branch: feat/TB-NNN-short-description

2. IMPLEMENT
   Read existing code in affected files
   Implement changes (AI does bulk of coding, dev reviews)
   Follow AGENTS.md conventions (structlog, Pydantic models, async patterns)
   Run ruff check + ruff format after changes

3. TEST
   Run pytest for affected tests
   Test against live local API (uvicorn, test JWT, curl/Postman)
   Verify DB state if applicable (psql via $SUPABASE_DB_URI)

4. SHIP
   Run quality gate (lint + import check + secrets scan)
   Create PR with structured description (link to ticket, what changed, test plan)
   Update Jira ticket status → "In Review"
   Post implementation comment on Jira ticket
```

**What AI handles vs what the dev handles:**

| Step | AI does | Dev does |
|------|---------|---------|
| Read ticket + plan approach | Understands requirements, proposes approach | Reviews approach, adjusts if needed |
| Write code | Generates implementation following AGENTS.md conventions | Reviews diff, catches architectural violations |
| Write tests | Generates test cases from acceptance criteria | Verifies test quality, adds edge cases |
| Create PR | Generates PR description from changes | Reviews PR description, approves |

**The dev's role shifts from "write code" to "review + direct."** The AI produces the first draft. The dev ensures it's correct, follows architecture, and doesn't break module boundaries.

### 2. Frontend Development Workflow

Same structure as backend, adapted for React Native:

**Key differences from backend:**
- Code lives in a separate mobile repo
- Testing involves device/simulator testing, not just unit tests
- UI changes need visual review (screenshot comparison)
- API contract changes must be coordinated with backend

**Frontend-specific conventions (to add to mobile repo's CLAUDE.md):**
- Component structure and naming patterns
- State management approach
- API client patterns (how to call backend endpoints)
- SSE event consumption patterns (how to render agent progress)
- Navigation patterns

At baseline, the frontend workflow follows the same 4-step pattern (pickup → implement → test → ship) with frontend-specific test commands and review criteria.

### 3. Coordination Agent

When a feature requires changes across multiple repos (backend API change + frontend consumption + architecture doc update), the coordination agent ensures nothing falls through the cracks.

**At baseline, this is a checklist, not an automated agent:**

```
Multi-repo change checklist:
□ Backend PR created with API changes
□ OpenAPI spec updated (if endpoint changed)
□ Frontend PR created that consumes the new/changed API
□ Frontend PR references backend PR
□ Architecture doc flagged for update (if module boundary changed)
□ Both PRs tested against each other (frontend points to backend branch)
□ Jira ticket links both PRs
```

**Why a checklist, not an automated coordinator:** At 3-5 devs, the overhead of building an automated multi-repo coordination system exceeds the benefit. A checklist in the PR template catches 90% of coordination failures. An automated coordination agent is refinement when the team scales to 8+ devs or 3+ repos.

---

## What Counts as Refinement

| Refinement | Why deferred |
|------------|-------------|
| Automated coordination agent (orchestrates multi-repo PRs) | Checklist works at current team size |
| AI-generated test suggestions on every PR | Manual test writing + AI assistance is sufficient |
| Automated branch management (auto-create, auto-delete stale) | Manual branch management works |
| Performance regression detection per PR | No benchmark suite yet |
| Cross-repo impact analysis | Single backend + single mobile app. Manageable manually. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| `CLAUDE.md` in backend repo | Foundation Setup (AKN) | AI coding tools need project context |
| Claude Code skills implemented | Foundation Setup (AKN) | `/test-local`, `/ship`, `/investigate` workflows |
| Jira MCP configured | Already done | Ticket management |
| AGENTS.md in backend repo | Already exists | Code conventions |

## Effort Estimate

| Component | Effort |
|-----------|--------|
| Document the backend dev workflow (standardize across team) | 0.5 day |
| Document the frontend dev workflow (mobile-specific conventions) | 0.5 day |
| Create multi-repo change checklist (PR template) | 0.5 day |
| Team training session (walk through the workflow with examples) | 0.5 day |

**Total: ~2 dev days to reach baseline.** Mostly documentation and alignment, not code.
