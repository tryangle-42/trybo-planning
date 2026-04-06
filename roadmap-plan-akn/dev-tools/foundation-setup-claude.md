# Foundation Setup — Self-Learning Repository & AI Dev Automation Baseline

> Owner: AKN
> Scope: Claude Code team setup, self-learning repository system, and full-spectrum AI development automation
> Extends: `12-dev-automation-skills.md` (10 Claude Code skills — that's the tooling leg; this is the full system)

---

## What This Is

Two things:

1. **Self-Learning Repository** — The codebase automatically maintains its own context. Every PR merge updates the system's understanding of itself. Architecture docs, context files, and module boundaries stay current without manual effort. The repository gets smarter after every merge.

2. **AI Dev Automation** — Everything beyond coding that should be automated for a team doing full-fledged AI-assisted development. The 10 Claude Code skills from `12-dev-automation-skills.md` are the first leg. This doc defines the full system.

## Current State

| What Exists | What's Missing |
|-------------|----------------|
| `12-dev-automation-skills.md` — 10 Claude Code skills designed | Skills not implemented in backend repo yet |
| `AGENTS.md` in backend repo (code conventions) | No `CLAUDE.md` in backend repo (Claude Code doesn't auto-load AGENTS.md) |
| Jira MCP configured | No automated context updates on PR merge |
| `agent-arch` repo with 17+ architecture docs | Docs diverge from code over time — no drift detection |
| structlog for logging | No automated standup/activity summaries |
| `0-main.md` as architecture source of truth | No mechanism to keep it in sync with actual code |
| Manual code review process | No AI-assisted review integrated into workflow |

---

## Baseline Definition

**The line:** After every PR merge, the repository's context files (CLAUDE.md, architecture docs, module completion status) are either auto-updated or flagged for update. A developer starting a new ticket gets accurate, current context without asking anyone. The team has automated standup, dependency management, and quality gates. Anything beyond this is refinement.

---

## Part 1: Self-Learning Repository

### The Core Loop

```
PR Merged to main
       │
       ▼
┌─────────────────────────┐
│ 1. Classify PR           │  Is this architecturally significant?
│    (heuristic + LLM)     │  (new module, new dependency, schema change, new skill, config change)
└──────────┬──────────────┘
           │
     ┌─────┴─────┐
     │ No         │ Yes
     │            ▼
     │  ┌────────────────────────┐
     │  │ 2. Analyze diff         │  What changed semantically?
     │  │    (AST-level, not raw) │  Which modules affected?
     │  └──────────┬─────────────┘
     │             │
     │             ▼
     │  ┌────────────────────────┐
     │  │ 3. Update context       │  Update CLAUDE.md sections
     │  │    (LLM generates diff) │  Update module completion %
     │  │                         │  Flag arch doc sections for review
     │  └──────────┬─────────────┘
     │             │
     │             ▼
     │  ┌────────────────────────┐
     │  │ 4. Open context PR      │  Reviewable, not auto-merged
     │  │    (or post comment)    │  Human approves context update
     │  └────────────────────────┘
     │
     ▼
┌─────────────────────────┐
│ 5. Update staleness      │  Bump "last verified" date
│    tracker               │  for affected modules
└─────────────────────────┘
```

### Baseline Components

#### 1. PR Classification (heuristic, no LLM needed)

A GitHub Action triggered on `push` to `main` that classifies the merge:

**Heuristic rules (cheap, deterministic, no LLM needed):**

| Signal | Classification |
|--------|---------------|
| New file in `app/services/` or `app/skills/` | Architecturally significant |
| Changes to `requirements.txt` / `pyproject.toml` | Dependency change |
| Changes to `app/models/` or migration files | Schema change |
| New skill registered in `seed_registry.py` | Skill catalog change |
| Changes to `app/services/orchestration/` | Orchestration change |
| Changes to `app/config/` or env vars | Config change |
| Pure test files, docstrings, formatting | Not significant — skip |

Use `dorny/paths-filter` GitHub Action for path-based classification. No LLM call needed for 80% of PRs.

#### 2. Combined Code Review + Context Update (single LLM pass)

For architecturally significant PRs, a single LLM call does both review AND context update suggestion. This saves tokens vs. two separate calls.

**Input to LLM:**
- Compressed diff (AST-level summary via `difftastic` or `ast-grep`, not raw diff — 60-80% token reduction)
- Only the relevant section(s) of CLAUDE.md (matched by file path heuristic)
- Module ownership for affected files (from `15-module-ownership-map.md`)

**Output from LLM:** Structured response with review comments, specific CLAUDE.md section updates, module completion status changes, and flags for which architecture docs may need manual review.

**Token efficiency:** The key trick is sending a compressed diff (AST-level summary, not raw line changes — see Key Concepts below) + only the relevant CLAUDE.md section + module context. This totals ~1K tokens input instead of sending full files, making each review + context update cost pennies, not dollars.

#### 3. Context File Management

**Single source of truth:** `CLAUDE.md` in the backend repo. Generated derivatives for other tools.

| File | Generated From | How |
|------|----------------|-----|
| `CLAUDE.md` | Primary — manually maintained + auto-updated | The canonical context file |
| `.cursor/rules/*.md` | `CLAUDE.md` sections | Script splits CLAUDE.md by section into glob-scoped rule files |
| `.github/copilot-instructions.md` | `CLAUDE.md` | Script reformats for Copilot |

**Sync script:** `scripts/sync-ai-context.sh` — runs as a pre-commit hook or manually. Reads CLAUDE.md, generates derivatives. Diffs them. Commits if changed.

#### 4. Architecture Drift Detection

Two layers:

**Layer 1: `import-linter` in CI (structural, deterministic)**

Translates module boundaries from `15-module-ownership-map.md` into machine-enforceable import rules. Example rules: "Skills cannot import from orchestration internals," "Memory module has no upstream dependencies." These run in CI on every PR and fail the build if a developer accidentally crosses a module boundary — catching architecture violations before they're merged, not after.

**Layer 2: Weekly LLM drift scan (semantic, comprehensive)**

GitHub Action cron job (weekly):
1. Read `0-main.md` architecture doc
2. Read current codebase structure (`find` + key file contents)
3. LLM: "Does this codebase match this architecture? Flag divergences."
4. Post results as a GitHub issue tagged `architecture-drift`

#### 5. Staleness Tracking

Every module in `13-platform-completion-status.md` gets a `Last verified: YYYY-MM-DD` stamp. The GitHub Action bumps this date when a PR touching that module's files is merged and the context update passes review.

If a module hasn't been verified in >30 days, the weekly drift scan flags it.

---

## Part 2: AI Dev Automation (Beyond the 10 Skills)

The 10 Claude Code skills in `12-dev-automation-skills.md` cover the developer's inner loop (ticket → code → test → PR). This section covers the **outer loop** — everything around the developer that should be automated.

### Baseline Automations

#### 1. Automated Standup from Git Activity

**What:** Daily summary of yesterday's git activity per developer, posted to Slack at 9 AM.

**How:** GitHub Action cron job:
1. `git log --since="1 day ago" --author=<each dev> --oneline`
2. Fetch Jira ticket status for referenced ticket IDs
3. LLM: "Summarize this developer's day in 3 bullet points"
4. Post to Slack channel via webhook

**Output format:**
```
🔧 Standup — April 1, 2026

**Avinash:**
- Completed TB-650: episodic memory table + generation
- In progress: TB-651 (mem0 → planner wiring)
- 3 commits, 1 PR merged

**Krishna:**
- ...
```

#### 2. Dependency Update with Context-Aware Changelogs

**What:** When Renovate/Dependabot opens a dependency update PR, auto-generate a context-aware summary of what changed upstream and whether it affects our code.

**How:** GitHub Action on `pull_request` from Renovate:
1. Read the upstream changelog for the updated package
2. Grep our codebase for usage of the updated package
3. LLM: "Given this changelog and our usage, what's relevant? Any breaking changes for us?"
4. Post summary as PR comment

#### 3. API Contract Testing

**What:** Auto-generate OpenAPI spec from FastAPI on every PR. Diff against the committed spec. Fail CI if they diverge without explicit acknowledgment.

**How:** FastAPI auto-generates OpenAPI at `/openapi.json`. The CI:
1. Starts the API server
2. Fetches `/openapi.json`
3. Diffs against `docs/openapi.json` (committed)
4. If different: fail with a clear message showing what endpoints changed
5. Developer must update the committed spec to pass

This catches accidental API changes and documents intentional ones.

#### 4. Security Scanning with AI Triage

**What:** Run SAST (Semgrep/Bandit) on every PR. Feed findings to LLM for triage — filter false positives, explain real issues, suggest fixes.

**How:** GitHub Action:
1. Run `semgrep --config=auto` on changed files
2. If findings: send to LLM with code context
3. LLM classifies each finding: `true_positive | false_positive | needs_review`
4. Post only true positives and needs_review as PR comments with explanations

#### 5. Test Coverage for Uncovered Changes

**What:** On PR, identify code paths changed but not covered by tests. Suggest which tests to write.

**How:** GitHub Action:
1. Run `pytest --cov` on the PR branch
2. Compute coverage diff (new uncovered lines in changed files)
3. If uncovered lines > threshold: LLM generates test suggestions
4. Post as PR comment: "These 3 functions in `planner.py` have no test coverage. Suggested tests: ..."

#### 6. PR-Triggered Architecture Doc Reminder

**What:** When a PR touches architecture-significant files, post a comment reminding the author to update architecture docs.

**How:** `dorny/paths-filter` + comment bot:
- Changes to `app/services/orchestration/` → "This PR changes orchestration. Check if `0-main.md` Section 3 needs updating."
- Changes to `seed_registry.py` → "New skill registered. Check if `15-module-ownership-map.md` M2 section needs updating."
- Changes to DB migrations → "Schema change. Check if `13-platform-completion-status.md` DB section needs updating."

---

## Part 3: What You May Have Missed

Automations that AI-native teams benefit from but aren't in the current plan:

| Automation | What It Does | Priority for Trybo |
|------------|-------------|-------------------|
| **Self-healing CI** | When CI fails, LLM analyzes failure, attempts fix, opens PR | Medium — saves time on flaky tests and lint issues |
| **Cross-repo impact analysis** | When `trybot_api` changes an API endpoint, check if the mobile app repo would break | High — you have a backend + mobile client |
| **Automated changelog generation** | On release/tag, generate user-facing changelog from commits + PR descriptions | Medium — needed before beta launch |
| **Incident response automation** | On production error alert, pull recent deploys + diffs + logs, generate initial RCA | Low now — relevant at scale |
| **Complexity hotspot detection** | Weekly scan: flag files with high cyclomatic complexity that are frequently changed | Low — small codebase currently |
| **Automated refactoring suggestions** | Weekly: identify code duplication, dead code, unused imports across the codebase | Low — Claude Code handles this per-session |
| **Onboarding environment setup** | Script that sets up a new dev's machine: clone repos, install deps, configure env vars, run seed data | High — needed when Sunny/Saurav join |
| **PR size guardrails** | Fail CI if PR exceeds N lines changed (configurable). Force decomposition. | Medium — prevents mega-PRs that are hard to review |

### Recommended Priority Order

1. **Onboarding environment setup** — immediate need (May hires)
2. **Cross-repo impact analysis** — prevent mobile app breakage
3. **Standup automation** — quality-of-life, builds team cadence
4. **API contract testing** — prevent silent API breakage
5. **PR-triggered doc reminders** — low effort, high value for doc currency
6. **Security scanning with AI triage** — before beta launch
7. Everything else — post-beta

---

## What Counts as Refinement (explicitly deferred)

| Refinement | Why deferred |
|------------|-------------|
| Full auto-commit of context updates (no human review) | Risk of bad auto-updates. Human-in-the-loop is safer at baseline. |
| Real-time context sync (on every commit, not just merge) | Merge-triggered is sufficient. Per-commit is noisy. |
| ML-based PR classification (trained on your repo) | Heuristic rules cover 80%. ML classifier is over-engineering. |
| Interactive architecture diagrams (Mermaid in CI) | ASCII diagrams are sufficient. Tooling is overhead. |
| Automated refactoring PRs | LLM-generated refactors are unreliable without extensive testing. |
| Full CI/CD pipeline as code review target | Overkill at current team size. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| `CLAUDE.md` created in backend repo | AKN (first action) | Everything else references it |
| GitHub Actions access | DevOps setup | All automations run as Actions |
| Slack webhook | Team setup | Standup automation target |
| `import-linter` added to `requirements-dev.txt` | AKN | Structural drift detection |
| Renovate or Dependabot configured | DevOps setup | Dependency update automation |
| `semgrep` or `bandit` in CI | DevOps setup | Security scanning |

## Effort Estimate

| Component | Effort |
|-----------|--------|
| `CLAUDE.md` creation + context sync script | 1 day |
| PR classification GitHub Action (heuristic) | 1 day |
| Combined review + context update Action (LLM) | 2 days |
| `import-linter` setup with module boundary rules | 1 day |
| Weekly drift scan Action | 1 day |
| Standup automation | 0.5 day |
| API contract testing in CI | 1 day |
| PR doc reminder bot | 0.5 day |
| Onboarding setup script | 1 day |

**Total: ~9-10 dev days to reach baseline.** Can be spread across 2-3 weeks alongside feature work.

---

## Research References

| Topic | Source |
|-------|--------|
| PR classification | `dorny/paths-filter` GitHub Action |
| AST-level diffs | `difftastic`, `ast-grep` |
| Import boundary enforcement | `seddonym/import-linter` (Python) |
| Context file sync | Single-source pattern (CLAUDE.md → derivatives) |
| Code review + doc update in one pass | Token-efficient combined prompt pattern |
| Security scanning | Semgrep (OSS), Bandit (Python-specific) |
| Staleness detection | Heuristic: path-based + timestamp tracking |
| Architecture drift | Structural (import-linter) + semantic (weekly LLM scan) |

---

## Key Concepts Explained

**What is an AST-level diff?**
A normal `git diff` shows raw text changes: "line 42 was removed, line 43 was added." An AST (Abstract Syntax Tree) diff understands code structure: "function `plan_execution` was renamed to `execute_plan`, and a new parameter `timeout_ms` was added." This is much more compact and meaningful than raw diffs. Tools like `difftastic` and `ast-grep` produce these. When feeding code changes to an LLM for review, an AST diff uses 60-80% fewer tokens than a raw diff while conveying more information.

**What is `import-linter`?**
A Python tool that enforces architectural rules as CI checks. You define rules like "module A cannot import from module B" in a config file, and the tool scans your codebase's import statements to verify compliance. It's like having an architecture police bot that catches boundary violations automatically. No ML, no LLM — just deterministic static analysis of Python imports.

**Why "single source of truth" for context files?**
Different AI coding tools read different files: Claude Code reads `CLAUDE.md`, Cursor reads `.cursor/rules/`, Copilot reads `.github/copilot-instructions.md`. If you maintain all three independently, they will drift apart. Instead: maintain one canonical file (`CLAUDE.md`), auto-generate the others from it via a script. One file to update, zero drift.

**Why not auto-commit context updates?**
An LLM analyzing a code diff and suggesting doc updates will occasionally get things wrong — especially for subtle architectural changes. At baseline, all context updates are opened as PRs for human review. This adds a 2-minute review step but prevents bad auto-updates from corrupting the team's shared context. Auto-commit is a refinement after the system proves reliable.
