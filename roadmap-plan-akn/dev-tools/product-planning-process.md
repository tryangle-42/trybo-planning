# Product Planning Process — Baseline Definition

> Owner: AKN
> Scope: The pipeline from product requirement → architecture decision → Jira tickets → developer prompt → implementation
> Context: This repo (`agent-arch`) IS the planning system. This doc defines the baseline for the process itself.

---

## What This Is

A repeatable, structured process for how a product requirement becomes working code. Every requirement follows the same pipeline, so that:
- No architectural decision is made in a vacuum
- Every Jira ticket has enough context for autonomous AI-assisted development
- Planning artifacts are reusable (not throwaway Slack messages or meeting notes)
- New team members can trace any feature from "why" to "what" to "how" to "where in code"

## Current State

| What Exists | What's Missing |
|-------------|----------------|
| `agent-arch` repo with 17+ planning docs | No standardized template for new planning docs |
| `0-main.md` as architecture source of truth | No formal "requirement intake" format |
| `15-module-ownership-map.md` with clear module boundaries | No standardized Jira ticket structure with dev prompts |
| `12-dev-automation-skills.md` with Claude Code skills | No traceability from requirement → architecture doc → ticket → PR |
| `13-platform-completion-status.md` with module completion tracking | No review/sign-off process for architecture decisions |
| Jira MCP configured for ticket management | Gap analysis exists (PDF) but no structured intake template |
| `Agent.md` describing repo role | No decision log (ADR) format |

---

## Baseline Definition

**The line:** Every product requirement that results in engineering work follows a documented 4-stage pipeline. Each stage produces a named artifact stored in this repo or Jira. A new team member can follow the chain from requirement to merged PR. Anything beyond this is refinement.

### The 4-Stage Pipeline

```
Stage 1: Intake          Stage 2: Architecture       Stage 3: Ticketing       Stage 4: Dev Prompt
─────────────────── ──▶ ─────────────────────── ──▶ ──────────────────── ──▶ ──────────────────
Requirement doc          Module impact analysis       Jira epics + stories     Per-ticket dev prompt
with context +           + architecture decision      with acceptance criteria  with full context
success criteria         record (ADR)                 + dependencies            for Claude Code
```

### Baseline Components

#### 1. Requirement Intake Template

Every new product requirement gets a file in `agent-arch/requirements/` (or a structured Jira epic description):

```markdown
# [REQ-NNN] Requirement Title

## Context
Why this matters. Business driver, user pain point, or strategic goal.

## Success Criteria
Measurable conditions for "done." Not implementation details.
- [ ] Criterion 1
- [ ] Criterion 2

## Scope
What's in. What's explicitly out.

## Modules Affected
List of M1-M11 modules this touches, referencing 15-module-ownership-map.md.

## Dependencies
What must exist before this can start.

## Open Questions
Unresolved decisions that need architecture review.

## Source
Where this came from: user feedback, gap analysis, team discussion, investor requirement.
```

#### 2. Architecture Decision Record (ADR) format

For non-trivial decisions (new module interaction, new data model, new external dependency), an ADR is created:

```markdown
# ADR-NNN: Decision Title

## Status
Proposed | Accepted | Superseded by ADR-NNN

## Context
What situation requires a decision.

## Decision
What was decided, and the key reasoning.

## Alternatives Considered
| Option | Pros | Cons | Why not |
|--------|------|------|---------|

## Consequences
What changes as a result. Which modules are affected. What new work is created.

## References
Links to requirement, relevant docs, external resources.
```

Stored in `agent-arch/decisions/`. Lightweight — most ADRs are 1 page.

#### 3. Jira Ticket Structure

Every implementation ticket follows a standard structure that's directly consumable by Claude Code:

**Epic:** Maps to a requirement. Contains the "why" and success criteria.

**Story / Task:** Maps to a unit of work. Contains:

```
## Summary
One sentence: what this ticket delivers.

## Context
Link to requirement (REQ-NNN) and ADR (ADR-NNN) if applicable.
Which module(s) this changes. Link to module definition in 15-module-ownership-map.md.

## Acceptance Criteria
- [ ] Specific, testable conditions

## Technical Approach (optional)
If the approach was decided in architecture review, link or summarize it.
If open, the developer (or Claude Code) decides.

## Files Likely Affected
Best-guess list of files/directories. Helps Claude Code narrow its search.

## Dependencies
Other tickets that must be done first (Jira links).

## Test Plan
How to verify this works. Specific test scenarios.
```

#### 4. Dev Prompt Template

For tickets that will be implemented by Claude Code (via `/dev-loop` or similar), a structured prompt accompanies the ticket:

```markdown
## Ticket: TB-NNN — [Title]

### Goal
[One sentence from the ticket summary]

### Context
[Condensed from requirement + ADR. What the developer — human or AI — needs to know.]

### Architecture Constraints
[From ADR or module ownership map. "This must go through M5 execution engine, not a direct API call." "Use $memory references, not hardcoded values."]

### Existing Code References
[Specific files/functions to start from. "See app/services/orchestration/compiler_graph.py:execute_node()"]

### Acceptance Criteria
[Copied from ticket]

### Anti-patterns to Avoid
[Things NOT to do. "Don't create a new table — use skill_data_store." "Don't add a new skill_id — extend existing handler."]
```

#### 5. Traceability chain

Every artifact links to its upstream:
- PR description links to Jira ticket (TB-NNN)
- Jira ticket links to requirement (REQ-NNN) and ADR (ADR-NNN)
- ADR links to requirement
- Requirement links to source (gap analysis item #, user feedback, etc.)

At baseline, this is manual linking (references in markdown). Refinement would automate it.

---

## What Counts as Refinement (explicitly deferred)

| Refinement | Why deferred |
|------------|-------------|
| Automated requirement→ticket generation | Manual is fine for 3-5 devs. Automate at scale. |
| ADR approval workflow (GitHub PR-based) | Informal review in team call is sufficient at baseline. |
| Automated traceability verification | Manual links work. Automated checks are nice-to-have. |
| Requirement prioritization framework (RICE, MoSCoW) | Small team can prioritize in a call. |
| Automated dev prompt generation from ticket | Template + manual fill is baseline. Auto-generation is refinement. |
| Cross-requirement dependency graph | Track in Jira epic links for now. |
| Retrospective / postmortem templates | Valuable but not a planning-process baseline. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| Jira MCP server | Already configured | Ticket creation and management |
| `agent-arch` repo structure | Already exists | Home for requirements, ADRs, planning docs |
| Module ownership map | 15-module-ownership-map.md | Referenced in every ticket for module impact |
| Claude Code skills | 12-dev-automation-skills.md | `/dev-loop` consumes dev prompts |

## Effort Estimate

- Create `requirements/` and `decisions/` directories with README: 0.5 day
- Write templates (4 templates): 1 day
- Backfill 3-5 existing decisions as ADRs (to establish the pattern): 1 day
- Update Jira ticket creation process (team alignment): 0.5 day

**Total: ~3 dev days to reach baseline.**
