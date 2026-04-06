# Backend Dev Automation Setup — Claude Code Prompt -> Ran and Done

> Paste this into a Claude Code session pointed at `/Users/avinash.nagar/Documents/git/trybot_api`

---

```
You are setting up Claude Code development automation for this FastAPI backend repo. This involves creating CLAUDE.md (the project knowledge base), 10 Claude Code skills, and a team onboarding guide.

## Step 0: Read Context

Before creating anything, read these files to understand the full architecture and current state:

FROM THIS REPO (trybot_api):
- AGENTS.md — existing project conventions (code style, architecture rules, structure)
- app/config/__init__.py — all environment variables and settings
- app/main.py — FastAPI app structure, startup sequence, registered routes
- app/agentic_mvp/runtime/contracts.py — core types (SkillDefinition, PlanSpec, ExecutionContext)
- app/services/database_utils.py — DB access helpers (safe_insert, safe_update, etc.)
- app/deps.py — auth dependency (get_authenticated_user)
- tests/conftest.py — pytest fixtures and test patterns
- .claude/settings.local.json — current permissions
- docs/TASK_ORCHESTRATION_SYSTEM.md — batch/task architecture
- docs/entire-schema.md — full DB schema (first 200 lines for key tables)

FROM ARCHITECTURE REPO:
- /Users/avinash.nagar/Documents/agent-arch/0-main.md — architecture source of truth
- /Users/avinash.nagar/Documents/agent-arch/13-platform-completion-status.md — what's built vs planned
- /Users/avinash.nagar/Documents/agent-arch/9-memory-architecture.md — 5-layer memory model (lines 1-100)

Read ALL of these before proceeding. Understanding the existing patterns is critical.

## Step 1: Create CLAUDE.md

Create `/Users/avinash.nagar/Documents/git/trybot_api/CLAUDE.md` — this is the single file Claude Code auto-loads every session.

It must contain:

### Section 1: Project Overview
Brief description: Trybo's FastAPI backend powering an agentic orchestration engine (LangGraph, Supabase, Twilio, WhatsApp, LiveKit). Link to AGENTS.md for full conventions.

### Section 2: Architecture Quick Reference
Summarize from 0-main.md (keep under 40 lines):
- Two-tier LangGraph executor (outer lifecycle graph + inner dynamic graph from PlanSpec)
- Skill registry → planner → executor → consent gates → batch/task execution
- Adapter types: Direct (68 handlers), MCP (Tavily), Composio marketplace
- $memory.<node_id>.<field> for inter-node data flow
- interrupt() + Command(resume=) for consent and batch completion
- Generic skill_data_store for all skill persistence (no feature-specific tables)
- 5-layer memory model (Layer 1: chat buffer ✅, Layer 2: working memory ✅, Layer 3: episodic ❌, Layer 4: mem0 built/not wired, Layer 5: skill_data_store ✅)

### Section 3: Code Conventions (from AGENTS.md + additions)
Reference AGENTS.md for full rules, then add:
- Routes are thin. Business logic in services/. DB via database_utils.py helpers.
- snake_case files/functions, PascalCase classes, UPPER_SNAKE_CASE constants
- 100-char line length (Ruff enforced)
- All application tables in `app` Postgres schema (not `public`)
- Service-role client for system tables, authenticated client for user tables

### Section 4: Git Conventions
- Branch: `<type>/<PROJECT_KEY>-<TICKET_NUMBER>-<short-description>`
  - Types: feat, fix, refactor, chore, test, docs
  - Example: `feat/TB-123-batch-resume`
- Commit: `<type>[PROJECT_KEY-TICKET_NUMBER]: <description>`
  - Example: `feat[TB-123]: Add batch completion resume handler`
- PR title follows same commit format
- Can reference multiple tickets: `fix[TB-123, TB-456]: Resolve API and UI issues`

### Section 5: Jira Workflow
- Project key: TB
- Current epic: TB-618
- When working on a ticket:
  1. Fetch ticket details via Jira MCP
  2. Create branch following naming convention
  3. Transition ticket to "In Progress"
  4. Post planning comment to Jira before implementing
  5. Post implementation comment to Jira after PR is raised
- Comment format: structured markdown with files changed, approach, test results

### Section 6: Local Development
- Start API: `source .venv/bin/activate && uvicorn app.main:app --reload --host 0.0.0.0 --port 8000`
- Logs: structlog JSON output, tee to `local_performance.log`
- Logger: `from app.logging_config import logger` (structlog, JSON format)
- Health check: `GET http://localhost:8000/__ping` → `{"ok": true}`
- Interactive docs: `http://localhost:8000/docs` (Swagger UI)
- DB access: always use `$SUPABASE_DB_URI` env var for psql — NEVER hardcode credentials
- Test JWT: use `$TEST_JWT` env var for authenticated endpoint testing
- Venv: `source .venv/bin/activate`

### Section 7: Before Committing
- Run: `ruff check . --fix && ruff format .`
- Verify imports: `python -c "from app.main import app"`
- Run affected tests: `pytest tests/test_<feature>.py -v`

### Section 8: Key File Paths
List the critical files (read from the codebase — include actual line counts/descriptions):
- Config: app/config/__init__.py
- Auth: app/deps.py
- DB utils: app/services/database_utils.py
- Supabase client: app/services/supabase_service.py
- LangGraph compiler: app/agentic_mvp/runtime/langgraph/compiler_graph.py
- Planner: app/agentic_mvp/runtime/langgraph/llm_planner.py
- Plan builder: app/agentic_mvp/runtime/langgraph/plan_builder.py
- Skill contracts: app/agentic_mvp/runtime/contracts.py
- Skill registry: app/agentic_mvp/runtime/seed_registry.py
- Direct adapter: app/agentic_mvp/runtime/adapters/direct.py
- Skill data store: app/services/skill_data/store.py
- Memory service: app/services/memory/
- Job scheduler: app/services/job_scheduler/
- DB schema: docs/entire-schema.md

### Section 9: Testing Patterns
Summarize from tests/conftest.py and existing tests:
- Framework: pytest
- Fixtures: sb_client (session scope, service-role Supabase), ensure_test_profile (test user)
- Async: @pytest.mark.asyncio with unittest.mock.AsyncMock
- Class-based grouping: TestClassName for logical organization
- Cleanup: explicit DELETE at end of test, not teardown fixtures
- Env-driven: tests skip gracefully if SUPABASE_URL/SUPABASE_SERVICE_ROLE_KEY not set
- Schema: always .postgrest.schema("app") for queries

### Section 10: Dependencies
- pip-tools managed: edit requirements.in, run pip-compile, never edit requirements.txt directly
- Add new dep: add to requirements.in → `pip-compile requirements.in` → `pip install -r requirements.txt`

IMPORTANT: Keep CLAUDE.md under 300 lines. Be dense and precise. No filler. Every line should be useful to a developer (human or AI) starting work on this codebase.

## Step 2: Create Skills

Create these 10 skill files. Each skill is a directory with a SKILL.md file.

### .claude/skills/test-local/SKILL.md

```yaml
---
name: test-local
description: Test endpoints against the locally running API at localhost:8000. Use when the user says "test locally", "try the API", "call the endpoint", or after implementing a new endpoint.
allowed-tools: Bash, Read, Grep
---
```

Test endpoints against the local API (must already be running).

1. Verify API is up: `curl -s http://localhost:8000/__ping`
2. If not running, tell the user to start it — do NOT auto-start.
3. For each endpoint being tested, construct scenarios:
   - Happy path with valid auth: `curl -s -X POST http://localhost:8000/<path> -H "Authorization: Bearer $TEST_JWT" -H "Content-Type: application/json" -d '<body>'`
   - Missing/invalid fields
   - Unauthorized (no token / bad token)
   - Not found (invalid IDs)
4. After each request, verify:
   - Response status code and structure
   - DB side effects: `psql "$SUPABASE_DB_URI" -c "SELECT ... FROM app.<table> WHERE ... ORDER BY created_at DESC LIMIT 5"`
   - Logs: `tail -50 local_performance.log | grep -i error`
5. Report pass/fail per scenario with response snippets.

### .claude/skills/test-write/SKILL.md

```yaml
---
name: test-write
description: Write pytest tests for recently implemented code. Use when the user says "write tests", "add tests", or after local testing confirms the implementation works.
allowed-tools: Read, Write, Edit, Glob, Grep
---
```

Write pytest tests following this project's patterns.

Before writing, read:
- `tests/conftest.py` — fixtures: `sb_client` (service-role Supabase), `ensure_test_profile` (test user)
- At least 2 existing test files to understand conventions

Patterns to follow:
- `@pytest.mark.asyncio` for async functions
- `unittest.mock.Mock`, `AsyncMock`, `patch` for service isolation
- Class-based grouping: `class TestFeatureName:`
- `setup_method()` for per-test initialization
- Environment-driven skips: `pytest.skip()` if env vars missing
- Schema targeting: `client.postgrest.schema("app")`
- Random test data: `secrets.token_hex(16)` for IDs
- Cleanup: explicit DELETE queries at end of test (NOT teardown fixtures)
- File naming: `tests/test_<feature>.py`

Generate tests covering:
1. Service layer unit tests (mock DB dependencies, test business logic)
2. Integration tests (real Supabase if credentials available, CRUD verification)
3. Edge cases from the ticket's acceptance criteria
4. Error cases (invalid input, missing records, auth failures)

### .claude/skills/quality/SKILL.md

```yaml
---
name: quality
description: Run lint, format, and code quality checks. Use before shipping, after implementation, or when the user asks to "lint", "check", or "format" the code.
allowed-tools: Bash, Read, Grep
---
```

Multi-step quality gate. Run all checks, auto-fix what's fixable, report what's not.

1. **Lint + Format:**
   ```bash
   source .venv/bin/activate && ruff check . --fix && ruff format .
   ```
2. **Import verification:**
   ```bash
   source .venv/bin/activate && python -c "from app.main import app; print('imports OK')"
   ```
3. **Security scan** (grep changed files only — use `git diff --name-only main...HEAD`):
   - Hardcoded API keys or passwords (strings matching `sk-`, `key=`, `password=`)
   - Hardcoded localhost URLs that should be config-driven
   - `print()` or `breakpoint()` debug statements
   - Disabled SSL verification (`verify=False`)
4. **Convention check** (on changed files):
   - Routes calling DB directly instead of through `database_utils.py`
   - Business logic in route handlers instead of services
   - Missing error handling on external API calls
5. Report: list issues found, or "All checks passed."

### .claude/skills/ship/SKILL.md

```yaml
---
name: ship
description: Complete shipping flow — quality checks, PR creation, Jira update. Use when the user says "ship it", "send it", or "we're done".
disable-model-invocation: true
allowed-tools: Bash, Read, Grep, Glob
---
```

Full shipping pipeline. This skill has external side effects — manual invocation only.

1. **Quality gate:** Run `/quality`. If lint or import errors, fix them. If security issues, report and stop.

2. **Test gate:** Run `source .venv/bin/activate && pytest -v` on affected test files. If failures, report and stop.

3. **Stage + commit** any remaining changes:
   ```bash
   ruff check . --fix && ruff format .
   git add -A
   git commit -m "<type>[<TICKET>]: <description>"
   ```

4. **Push:**
   ```bash
   git push -u origin $(git branch --show-current)
   ```

5. **Create PR:**
   ```bash
   gh pr create --title "<type>[<TICKET>]: <description>" --body "## Summary\n<changes>\n\n## Test Plan\n<test results>\n\nJira: https://tryangle42-team.atlassian.net/browse/<TICKET>"
   ```

6. **Update Jira:** Post implementation comment with:
   - PR link
   - Files changed (from `git diff --stat main...HEAD`)
   - Test results summary
   - Transition ticket to "In Review" if possible

7. Report: PR URL + Jira ticket status.

### .claude/skills/jira-update/SKILL.md

```yaml
---
name: jira-update
description: Post a structured update to a Jira ticket. Use when the user says "update jira", "comment on the ticket", or at planning/implementation/testing milestones.
argument-hint: <ticket-id> [planning|implementation|testing|review]
allowed-tools: Read, Bash
---
```

Post a structured comment to a Jira ticket. Determine the phase from context or argument.

**Planning phase comment:**
```markdown
## Planning
**Approach:** <1-2 sentence summary>
**Files to modify:**
- <file path> — <what changes>
**Dependencies:** <other tickets or none>
**Risks:** <edge cases or none>
```

**Implementation phase comment:**
```markdown
## Implementation
**PR:** <link>
**Changes:**
- <file path> — <what was done>
**Key decisions:** <any non-obvious choices made>
```

**Testing phase comment:**
```markdown
## Testing
**Local API tests:** <pass/fail count>
**Pytest:** <pass/fail/skip count>
**Manual verification:** <what was checked>
```

Use Jira MCP to post the comment. Transition ticket status if appropriate (In Progress → In Review).

### .claude/skills/investigate/SKILL.md

```yaml
---
name: investigate
description: Debug a bug or error using logs, database state, and code analysis. Use when the user reports a bug, an error occurs, or says "debug", "investigate", "why is this failing".
allowed-tools: Read, Grep, Glob, Bash
---
```

Cross-system debugging. Correlate logs, DB state, and code to find root cause.

1. **Gather context:** What's the error? Check recent logs:
   ```bash
   tail -100 local_performance.log | grep -i "error\|exception\|traceback"
   ```

2. **Find the code path:** Use Grep/Glob to locate the failing endpoint, service, or function.

3. **Check DB state:** Query relevant tables:
   ```bash
   psql "$SUPABASE_DB_URI" -c "SELECT id, status, created_at FROM app.<table> WHERE ... ORDER BY created_at DESC LIMIT 10"
   ```

4. **Check recent changes:**
   ```bash
   git log --oneline -10
   git diff HEAD~3 --name-only
   ```

5. **Correlate:** Match log timestamps → DB record states → code paths. Identify where the expected flow diverges from actual.

6. **Output:** Root cause analysis + suggested fix (or fix it directly if the cause is clear).

Key tables for debugging:
- `app.batches` — batch status, summary_text
- `app.bot_tasks` — task status, error info, result_payload
- `app.chat_messages` — message flow, seq ordering
- `app.agent_runs` — graph execution status
- `app.agent_run_events` — granular execution events

### .claude/skills/db/SKILL.md

```yaml
---
name: db
description: Query the dev Supabase database. Use when the user asks to "check the database", "query", "look up", or when debugging needs DB state.
argument-hint: <table-name or natural language query>
allowed-tools: Bash, Read
---
```

Safety-railed database access. Read-only by default.

**Connection:** Always use `$SUPABASE_DB_URI` env var. Never hardcode credentials.

**Common queries:**
- Table structure: `psql "$SUPABASE_DB_URI" -c "\d app.<table>"`
- Recent records: `psql "$SUPABASE_DB_URI" -c "SELECT * FROM app.<table> ORDER BY created_at DESC LIMIT 10"`
- Status counts: `psql "$SUPABASE_DB_URI" -c "SELECT status, count(*) FROM app.<table> GROUP BY status"`
- User's data: `psql "$SUPABASE_DB_URI" -c "SELECT * FROM app.<table> WHERE user_id = '<id>' ORDER BY created_at DESC LIMIT 10"`

**Schema:** All application tables are in the `app` schema. Always prefix: `app.<table_name>`.

**Safety rules:**
- SELECT queries: run freely
- UPDATE/DELETE/DROP: NEVER run without explicit user confirmation. Show the query first and ask.
- INSERT: ask first unless the user explicitly requested it

### .claude/skills/standup/SKILL.md

```yaml
---
name: standup
description: Generate a standup report from recent git activity and Jira tickets. Use when the user says "standup", "what did I do", or "status update".
disable-model-invocation: true
allowed-tools: Bash, Read
---
```

Aggregate git activity and Jira status into a standup report.

1. **Git activity:**
   ```bash
   git log --since="yesterday" --author="$(git config user.name)" --oneline --no-merges
   ```

2. **Files changed:**
   ```bash
   git diff --stat HEAD~5
   ```

3. **Current branch status:**
   ```bash
   git branch --show-current
   git log main..HEAD --oneline
   ```

4. **Jira tickets:** Fetch assigned tickets and their statuses via Jira MCP.

5. **Format output:**
   ```
   **Yesterday:**
   - <completed work with ticket refs>

   **Today:**
   - <planned work>

   **Blockers:**
   - <any blockers or "None">
   ```

### .claude/skills/arch-sync/SKILL.md

```yaml
---
name: arch-sync
description: Compare codebase against architecture docs and flag drift. Use when the user says "check architecture", "is code in sync", or after major changes.
context: fork
agent: Explore
allowed-tools: Read, Glob, Grep, Bash
---
```

Detect drift between the actual codebase and architecture documentation.

**Architecture sources:**
- `/Users/avinash.nagar/Documents/agent-arch/0-main.md` — architecture source of truth
- `/Users/avinash.nagar/Documents/agent-arch/13-platform-completion-status.md` — module completion status
- `AGENTS.md` — project structure and conventions
- `docs/*.md` — feature-level documentation

**Check for:**
1. **Structure drift:** Does the directory layout match what AGENTS.md documents?
2. **Pattern drift:** Are routes thin? Is business logic in services? Is DB access through database_utils.py?
3. **Undocumented code:** New endpoints, services, or features not in any docs
4. **Dead references:** Docs mention files/functions that no longer exist
5. **Convention violations:** Direct DB calls bypassing helpers, hardcoded URLs, logic in routes

**Output:** Drift report organized by severity (breaking > convention > documentation).

### .claude/skills/dev-loop/SKILL.md

```yaml
---
name: dev-loop
description: Full autonomous development loop — Jira ticket to PR. Use when the user says "dev loop", "auto-develop", or "take this ticket end to end".
argument-hint: <ticket-id>
disable-model-invocation: true
allowed-tools: Read, Edit, Write, Glob, Grep, Bash
---
```

Full ticket-to-PR pipeline with 2 human checkpoints.

**PHASE 1: Intake + Planning**
1. Fetch ticket $ARGUMENTS from Jira via MCP
2. Parse requirements and acceptance criteria
3. Create branch: `feat/$ARGUMENTS-<short-description>` (or fix/ refactor/ as appropriate)
4. Transition ticket to "In Progress"
5. Explore codebase for affected areas (read existing patterns)
6. Create implementation plan
7. Post planning comment to Jira: `/jira-update $ARGUMENTS planning`
8. **CHECKPOINT:** Present plan to user. Wait for approval before proceeding.

**PHASE 2: Implementation**
9. Execute plan step by step following AGENTS.md conventions
10. Commit after each logical unit: `<type>[$ARGUMENTS]: <description>`
11. Post implementation comment: `/jira-update $ARGUMENTS implementation`

**PHASE 3: Testing**
12. `/test-local` — test against running API
13. `/test-write` — write pytest tests
14. Run tests: `source .venv/bin/activate && pytest -v`
15. If failures: diagnose, fix, rerun (max 3 attempts)

**PHASE 4: Quality**
16. `/quality` — lint, format, security scan
17. **CHECKPOINT:** Show `git diff main...HEAD` to user. Wait for approval.

**PHASE 5: Ship**
18. `/ship` — push, create PR, update Jira, transition to "In Review"
19. Report: PR URL + final Jira status

## Step 3: Update settings.local.json

Read the current `.claude/settings.local.json` and ADD these Bash permissions if they don't already exist (merge into the existing allow list, don't replace):

```
"Bash(curl http://localhost:*)",
"Bash(curl -s http://localhost:*)",
"Bash(curl -s -X * http://localhost:*)",
"Bash(pytest *)",
"Bash(ruff *)",
"Bash(pip-compile *)",
"Bash(source .venv/bin/activate && *)"
```

## Step 4: Create Team Onboarding Guide

Create `docs/CLAUDE_CODE_GUIDE.md` — a guide for Krishna and Rishav to use the skills:

Contents:
1. **What's set up:** CLAUDE.md (auto-loaded context), 10 skills, Jira MCP integration
2. **Getting started:** Just open Claude Code in this repo. CLAUDE.md loads automatically. Say "work on TB-XXX" and Claude will fetch the ticket, create a branch, and plan.
3. **Available skills:** Table of all 10 skills with what they do and how to invoke
4. **The dev loop:** Explain `/dev-loop TB-XXX` and the 2 checkpoints
5. **Tips:**
   - Say "test locally" after implementing to run /test-local
   - Say "ship it" when ready to trigger full shipping pipeline
   - Use `/db users` to quickly query a table
   - Use `/investigate` when something breaks
   - Use `/standup` before standup meetings
6. **What NOT to do:**
   - Don't hardcode DB credentials — always use $SUPABASE_DB_URI
   - Don't skip the quality gate before PRs
   - Don't force push to main

## Execution Order

1. Read all context files (Step 0) — understand before writing
2. Create CLAUDE.md (Step 1) — this is the most important file
3. Create all 10 skills (Step 2) — in order listed
4. Update settings.local.json (Step 3) — merge permissions
5. Create team guide (Step 4)
6. Verify: run `ls -la .claude/skills/*/SKILL.md` to confirm all skills exist
7. Run `cat CLAUDE.md | wc -l` to confirm under 300 lines

After completion, show me the CLAUDE.md content and a list of all created files.
```
