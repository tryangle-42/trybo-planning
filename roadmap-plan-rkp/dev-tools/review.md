# Review Subagents — Baseline Definition

> Owner: RKP
> Scope: AI-assisted review pipeline — code review, plan review, PR review
> Context: With AI doing most of the coding, review becomes the primary quality gate. These subagents help the reviewer focus on architecture and correctness, not style and formatting.

---

## What This Is

Three specialized review subagents that catch different classes of issues:

| Subagent | What it reviews | When it runs |
|----------|----------------|-------------|
| **Code Review** | Code quality, architecture compliance, security, performance | On every PR, before human review |
| **Plan Review** | Generated PlanSpecs — are they correct, efficient, safe? | On every plan generation (dev/staging only) |
| **PR Review** | PR structure, completeness, ticket alignment, test coverage | On every PR creation |

These are not replacements for human review. They're filters that catch the obvious issues so the human reviewer focuses on the hard problems.

## Current State

Nothing built. All review is manual.

---

## Baseline Definition

**The line:** Every PR gets an automated code review comment highlighting potential issues. Every PlanSpec in dev/staging gets a quality check. PR descriptions are validated for completeness. Anything beyond this is refinement.

---

## Baseline Components

### 1. Code Review Subagent

Runs as a GitHub Action on every `pull_request` event. Analyzes the diff and posts review comments.

**What it checks:**

| Category | What | How |
|----------|------|-----|
| **Architecture compliance** | Does the PR respect module boundaries? | `import-linter` (from AKN's foundation-setup) catches structural violations. The code review agent adds semantic checks: "this PR modifies M5 execution logic from inside an M3 file." |
| **Security** | PII in logs? Hardcoded secrets? SQL injection? Prompt injection vectors? | Semgrep/Bandit rules (from AKN's foundation-setup) + LLM scan of the diff for context-specific issues |
| **Error handling** | Are errors caught and classified? Are retries appropriate? | LLM review of error-handling patterns against the error classification taxonomy (from agent-intelligence.md) |
| **Naming + conventions** | Does the code follow AGENTS.md conventions? | Primarily handled by ruff (already configured). The agent catches semantic issues ruff can't: "this function does X but is named Y." |
| **Test coverage** | Are new code paths covered by tests? | Coverage diff analysis. Flag: "these 3 functions have no test coverage." |

**Output format:** GitHub PR review comments on specific lines, plus a summary comment at the top: "Found 2 issues (1 security, 1 architecture). 3 suggestions. Overall: looks good pending these fixes."

**Not a gatekeeper:** The code review agent posts comments but does NOT block the PR. The human reviewer decides whether to address the comments or dismiss them. Auto-blocking PRs based on LLM judgment is refinement (needs high confidence first).

### 2. Plan Review Subagent

Validates generated PlanSpecs in dev/staging environments. Not in production at baseline — it adds latency to the planning pipeline.

**What it checks:**

| Check | What it catches |
|-------|----------------|
| **Over-decomposition** | Plan has 8 steps for something that needs 2. Penalize unnecessary complexity. |
| **Skill selection quality** | Could a simpler/cheaper/faster skill have achieved the same result? |
| **Missing consent flags** | Side-effecting skills without `requires_consent: true` |
| **Redundant steps** | Two steps that do the same thing (e.g., search twice with different providers) |
| **Dependency correctness** | Task B depends on Task A, but Task A's output doesn't match Task B's input |

**How it works:** After the planner generates a PlanSpec (in dev/staging), the plan review agent runs a set of rule-based checks (structural) + an LLM-as-judge evaluation (semantic). Results are logged to a `plan_reviews` table with scores and flags.

**This feeds into plan quality scoring** (agent-intelligence.md). The plan review agent generates the raw quality signal; the intelligence module tracks trends over time.

**Not in production pipeline at baseline.** In prod, plans are validated by M3's structural validator (fast, deterministic). The plan review agent runs offline on a sample of recent plans — weekly batch analysis. Adding it inline in prod (adding 500ms+ to every plan) is refinement when its accuracy justifies the latency cost.

### 3. PR Review Subagent

Validates PR structure and completeness. Runs as a GitHub Action on PR creation.

**What it checks:**

| Check | What it catches |
|-------|----------------|
| **Ticket reference** | PR description doesn't reference a Jira ticket (TB-NNN) |
| **Description quality** | PR description is empty or boilerplate. Must explain *what* and *why*. |
| **Test plan present** | No test plan section in the PR description |
| **Architecture doc flag** | PR touches architecture-significant files but doesn't mention doc updates (from AKN's PR-triggered doc reminder) |
| **PR size** | PR exceeds N lines changed (configurable, default 500). Warn: "consider splitting this PR." |
| **File coverage** | Tests added for new files? Or just implementation with no tests? |

**Output format:** A checklist comment on the PR:
```
PR Review:
✅ Ticket reference: TB-652
✅ Description: present and substantive
⚠️ Test plan: missing — please add a test plan section
⚠️ PR size: 620 lines changed — consider splitting
✅ Architecture docs: no flag needed (no architecture-significant files changed)
```

This is lightweight (rule-based, no LLM needed for most checks). It enforces PR hygiene without blocking — comments are suggestions, not gates.

---

## What Counts as Refinement

| Refinement | Why deferred |
|------------|-------------|
| Auto-blocking PRs based on review agent findings | Needs high confidence. False positives would slow the team. |
| Plan review in production pipeline (inline, every plan) | Adds 500ms+. Run offline on samples at baseline. |
| AI-generated fix suggestions (not just flagging, but proposing code changes) | Useful but complex. Agent comments are sufficient at baseline. |
| Review agent learning from human reviewer decisions | Needs interaction volume to learn meaningful patterns. |
| Cross-PR review (does this PR conflict with another open PR?) | Rare at 3-5 devs. Relevant at 8+ devs. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| GitHub Actions access | DevOps | All agents run as Actions |
| `import-linter` in CI | Foundation Setup (AKN) | Architecture compliance checks |
| Semgrep/Bandit in CI | Foundation Setup (AKN) | Security scanning |
| Jira MCP | Already configured | Ticket reference validation |
| Plan quality scoring framework | Agent Intelligence (RKP) | Plan review feeds quality scores |

## Effort Estimate

| Component | Effort |
|-----------|--------|
| Code review agent (GitHub Action + LLM review of diffs) | 3-4 days |
| PR review agent (rule-based checklist + GitHub Action) | 1-2 days |
| Plan review agent (rule-based + LLM-as-judge, offline batch) | 2-3 days |
| Testing + tuning (reduce false positives on real PRs) | 2 days |

**Total: ~8-11 dev days to reach baseline.**

---

## Key Concepts Explained

**Why not just use ruff + Semgrep and skip the LLM-based review?**
Ruff catches formatting and style issues. Semgrep catches known vulnerability patterns. Neither catches: "this function modifies M5 execution state from inside an M3 file" (architecture violation) or "this error should be classified as TRANSIENT but it's being treated as PERMANENT" (domain-specific logic error). The LLM-based review catches semantic issues that rule-based tools miss.

**Why separate code review and PR review?**
PR review checks the *wrapper* — is the PR well-structured, does it reference a ticket, is there a test plan? Code review checks the *content* — is the code correct, safe, and architecturally sound? They run at the same time but catch different classes of issues. A PR can have a great description and terrible code, or great code and a missing test plan.

**Why plan review offline, not inline?**
The planning pipeline targets <2s end-to-end. Adding an LLM-as-judge evaluation adds 500ms-1s. At baseline, this latency cost isn't justified — the structural validator (M3) catches the critical issues (missing skills, cycles, consent violations) in <10ms. The semantic quality review (is this plan *good*?) runs weekly on a sample to track trends. If plan quality degrades, it triggers investigation — it doesn't block individual requests.
