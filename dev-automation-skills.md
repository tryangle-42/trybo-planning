# Development Automation Skills — Full System Plan -> Completed through 14-backend-setup-prompt.md

> Complete skill system for `/Users/avinash.nagar/Documents/git/trybot_api`
> Goal: Zero-human-coding autonomous development from Jira ticket to merged PR.

---

## Design Principle: Skills vs CLAUDE.md

**CLAUDE.md** handles anything Claude already does by default — it just needs project-specific parameters:
- Branch/commit naming conventions
- Code style and architecture rules (already in AGENTS.md)
- Jira project key, local API port, log file location, DB connection pattern
- "Before committing, run ruff" / "After implementing, run pytest"

**Skills** encode workflows that Claude would NOT figure out on its own:
- Multi-file scaffolding templates with specific registration steps
- Multi-system orchestration (git + gh + Jira in one flow)
- Project-specific testing against a live local API
- Cross-system debugging (logs + DB + code correlation)
- Architecture drift detection against external docs

If a skill's logic is just "read X, then do Y" — it belongs in CLAUDE.md.

---

## 1. Infrastructure Prerequisites

### 1.1 CLAUDE.md (Create — does not exist)

The repo has `AGENTS.md` but no `CLAUDE.md`. Claude Code auto-loads `CLAUDE.md`, not `AGENTS.md`.

**Action:** Create `/trybot_api/CLAUDE.md` that:
- References AGENTS.md (`See AGENTS.md for full project conventions`)
- Branch naming: `<type>/<PROJECT_KEY>-<TICKET_NUMBER>-<short-description>`
- Commit format: `<type>[PROJECT_KEY-TICKET_NUMBER]: <description>`
- Jira project key: `TB` (epic: TB-618)
- Local API: `uvicorn app.main:app --reload --host 0.0.0.0 --port 8000` on port 8000
- Logs: structlog JSON at `local_performance.log`, logger at `app/logging_config.py`
- DB: always use `$SUPABASE_DB_URI` env var for psql, never hardcode credentials
- Venv: `source .venv/bin/activate`
- Before committing: `ruff check . --fix && ruff format .`
- After implementing: `pytest -v` for affected tests
- When working on a Jira ticket: fetch it via MCP, transition to "In Progress", post planning/implementation comments at each phase
- Test JWT: use `$TEST_JWT` env var for local API testing

This single file absorbs what `/onboard`, `/pickup`, `/plan-impl`, `/implement`, `/test-run`, and `/logs` would have done — Claude already knows how to do those things, it just needs the project parameters.

### 1.2 Skills Directory

```
trybot_api/.claude/skills/
├── test-local/SKILL.md        # Live API testing with auth, DB verification, log checks
├── test-write/SKILL.md        # Project-specific pytest patterns (conftest, cleanup, class grouping)
├── quality/SKILL.md           # Multi-step quality gate (ruff + import + secrets + debug prints)
├── ship/SKILL.md              # Full ship: quality → PR → Jira update → transition
├── jira-update/SKILL.md       # Phase-aware structured Jira commenting
├── investigate/SKILL.md       # Cross-system debugging (logs + DB + code + git)
├── db/SKILL.md                # Safety-railed dev database queries
├── standup/SKILL.md           # Git + Jira activity aggregation
├── arch-sync/SKILL.md         # Code vs architecture drift detection
└── dev-loop/SKILL.md          # Full autonomous ticket-to-PR pipeline
```

10 skills. Everything else is CLAUDE.md.

### 1.3 MCP Servers (Current + Recommended)

| Server | Status | Purpose |
|--------|--------|---------|
| Jira (atlassian-jira SSE) | ✅ Configured | Ticket CRUD, comments, transitions |
| GitHub (gh CLI) | ✅ Available | PR creation, reviews, checks |
| Supabase (psql via Bash) | ✅ Configured in settings | DB queries, schema inspection |
| Filesystem (built-in) | ✅ Available | Code read/write |

**No additional MCP servers needed.** The `gh` CLI + `psql` via Bash permissions cover GitHub and DB access. Jira MCP is already configured.

### 1.4 Settings Updates Required

Update `.claude/settings.local.json` to add Bash permissions for:
```
Bash(curl http://localhost:*)     # Local API testing
Bash(httpx *)                     # Alternative HTTP client
Bash(pytest *)                    # Test execution
Bash(ruff *)                      # Lint/format
Bash(pip-compile *)               # Dependency management
Bash(source * && uvicorn *)       # Local API startup
```

### 1.5 Environment Access

Skills access DB credentials via environment variables loaded from `.env`:
- `SUPABASE_DB_URI` — Postgres connection string for psql queries
- `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` — Supabase SDK access
- **Never hardcode credentials in skill files or CLAUDE.md**

### 1.6 Hooks (Optional — Phase 2)

| Hook | Trigger | Action |
|------|---------|--------|
| Pre-commit | Before git commit | Run `ruff check --fix` + `ruff format` |
| Post-ship | After `/ship` completes | Notify Slack (if configured) |

---

## 2. Skill Catalog

### Testing Skills

#### `/test-local` — Local API Integration Testing
```yaml
---
name: test-local
description: Test endpoints against the locally running API at localhost:8000. Use when the user says "test locally", "try the API", "call the endpoint", or after implementing a new endpoint. Assumes the API is already running.
allowed-tools: Bash, Read, Grep
---
```

**Logic:**
1. Verify API is running: `curl -s http://localhost:8000/__ping`
2. If not running, inform user to start it (don't auto-start — side effect)
3. For the implemented ticket, construct test scenarios:
   - Happy path with valid auth token
   - Edge cases (missing fields, invalid data)
   - Error cases (unauthorized, not found)
4. Execute requests via `curl` or `python -c "import httpx; ..."`:
   ```bash
   curl -s -X POST http://localhost:8000/api/endpoint \
     -H "Authorization: Bearer $TEST_JWT" \
     -H "Content-Type: application/json" \
     -d '{"field": "value"}'
   ```
5. Validate response structure, status codes, DB state changes
6. Query DB to verify side effects: `psql $SUPABASE_DB_URI -c "SELECT ..."`
7. Check logs for errors: `tail -50 local_performance.log`
8. Report results with pass/fail per scenario

**Invocation:** Auto (after implementation) + Manual

#### `/test-write` — Write Pytest Tests
```yaml
---
name: test-write
description: Write pytest tests for recently implemented code. Use when the user says "write tests", "add tests", or after local testing confirms the implementation works.
allowed-tools: Read, Write, Edit, Glob, Grep
---
```

**Logic:**
1. Read `tests/conftest.py` for available fixtures (`sb_client`, `ensure_test_profile`)
2. Read existing test files for patterns:
   - `@pytest.mark.asyncio` for async tests
   - `unittest.mock.Mock, AsyncMock, patch` for mocking
   - Class-based grouping (`TestClassName`)
   - Environment-driven skips
3. Generate tests covering:
   - Service layer unit tests (mock DB, test logic)
   - Integration tests (real Supabase, CRUD verification)
   - Edge cases from acceptance criteria
4. Write to `tests/test_<feature>.py`
5. Follow cleanup pattern: explicit DELETE at end, not teardown fixtures

**Invocation:** Auto (after local testing) + Manual

---

### Quality & Review Skills

#### `/quality` — Pre-Ship Quality Checks
```yaml
---
name: quality
description: Run lint, format, and code quality checks. Use before shipping, after implementation, or when the user asks to "lint", "check", or "format" the code.
allowed-tools: Bash, Read, Grep
---
```

**Logic:**
1. `ruff check . --select ALL` — lint issues
2. `ruff format --check .` — format issues
3. `ruff check . --fix && ruff format .` — auto-fix
4. `python -c "from app.main import app"` — import check
5. Check for:
   - Hardcoded secrets (grep for API keys, passwords)
   - TODO/FIXME left behind
   - Debug prints/breakpoints
   - Missing type hints on new public functions
6. Report issues or confirm clean

**Invocation:** Auto (before PR) + Manual

#### `/self-review` — Pre-PR Code Review
```yaml
---
name: self-review
description: Self-review all changes before creating a PR. Use before shipping, or when the user says "review my changes" or "check the diff".
context: fork
allowed-tools: Bash, Read, Grep, Glob
---
```

**Logic:**
1. `git diff main...HEAD` — full diff
2. Review against AGENTS.md conventions:
   - Routes thin? Logic in services?
   - Using `database_utils.py` helpers?
   - Error handling graceful?
   - No hardcoded URLs or secrets?
3. Check for:
   - Leftover debug code
   - Missing error handling at system boundaries
   - Breaking changes to existing APIs
   - Performance concerns (N+1 queries, unbounded loops)
4. Output: list of issues or "LGTM"

**Invocation:** Auto (before PR) + Manual

---

### Shipping Skills

#### `/ship` — Full Ship Flow
```yaml
---
name: ship
description: Complete shipping flow — quality gate, create PR, update Jira. Use when the user says "ship it", "send it", or "we're done".
disable-model-invocation: true
allowed-tools: Bash, Read, Grep, Glob, mcp__jira__*
---
```

**Logic:**
1. Run `/quality` checks — abort if failures
2. Run `/self-review` — flag issues but don't block
3. Run `/test-run` — abort if test failures
4. Stage & commit any uncommitted changes
5. Push branch: `git push -u origin <branch>`
6. Create PR via `gh pr create`:
   - Title: `<type>[TB-XXX]: <description>` (matching commit format)
   - Body: summary of changes, test plan, link to Jira ticket
   - Labels: auto-detect (feature, bugfix, refactor)
7. Update Jira:
   - Add implementation comment with PR link
   - Add test results summary
   - Transition to "In Review" (if workflow supports it)
8. Report: PR URL + Jira status

**Invocation:** Manual only (`disable-model-invocation: true`)

#### `/jira-update` — Update Jira Ticket
```yaml
---
name: jira-update
description: Post an update comment to a Jira ticket. Use when the user says "update jira", "comment on the ticket", or after major milestones.
argument-hint: <ticket-id> [phase: planning|implementation|testing|review]
allowed-tools: Read, Bash, mcp__jira__*
---
```

**Logic:**
1. Determine phase (planning, implementation, testing, review)
2. Gather context:
   - Planning: implementation plan summary
   - Implementation: files changed, approach taken
   - Testing: test results, coverage
   - Review: PR link, review status
3. Post structured comment to Jira ticket
4. Transition ticket status if appropriate

**Invocation:** Auto (at each phase) + Manual

---

### Debug & Investigate Skills

#### `/investigate` — Debug Issues
```yaml
---
name: investigate
description: Investigate a bug or error using logs, database state, and code analysis. Use when the user reports a bug, an error occurs, or says "debug", "investigate", "why is this failing", or "what's wrong".
allowed-tools: Read, Grep, Glob, Bash
---
```

**Logic:**
1. Gather error context (from user description or recent logs)
2. Search logs: `grep -i "error\|exception\|traceback" local_performance.log | tail -50`
3. Read relevant code paths (Glob + Grep for the failing area)
4. Query DB for relevant state:
   ```bash
   psql "$SUPABASE_DB_URI" -c "SELECT * FROM app.<table> WHERE ... ORDER BY created_at DESC LIMIT 5"
   ```
5. Check recent commits: `git log --oneline -10`
6. Cross-reference: log timestamps → DB state → code path
7. Output: root cause analysis + suggested fix

**Invocation:** Auto (matches error/debug patterns) + Manual

#### `/db` — Query Dev Database
```yaml
---
name: db
description: Query the development Supabase database. Use when the user asks to "check the database", "query", "look up", or when debugging requires inspecting DB state.
argument-hint: <table-name or query description>
allowed-tools: Bash, Read
---
```

**Logic:**
1. Load connection from env: `$SUPABASE_DB_URI`
2. Construct query based on user intent:
   - Table inspection: `\d app.<table>`
   - Data query: `SELECT ... FROM app.<table> WHERE ... LIMIT 20`
   - Status check: `SELECT status, count(*) FROM app.<table> GROUP BY status`
3. Execute: `psql "$SUPABASE_DB_URI" -c "<query>"`
4. Format results in readable table
5. **Safety:** Read-only queries only. Never run UPDATE/DELETE/DROP without explicit user confirmation.

**Invocation:** Auto + Manual

---

### Maintenance & Reporting Skills

#### `/standup` — Generate Standup Report
```yaml
---
name: standup
description: Generate a standup report from recent git activity and Jira ticket status. Use when the user says "standup", "what did I do", or "status update".
disable-model-invocation: true
allowed-tools: Bash, Read, mcp__jira__*
---
```

**Logic:**
1. `git log --since="yesterday" --author="$(git config user.name)" --oneline` — recent commits
2. `git diff --stat HEAD~5` — scope of recent changes
3. Fetch assigned Jira tickets and their statuses
4. Generate standup format:
   ```
   **Yesterday:** Completed TB-123 (endpoint + tests), reviewed TB-456
   **Today:** Starting TB-789 (skill scaffold)
   **Blockers:** Waiting on API key for Composio integration
   ```

**Invocation:** Manual only

#### `/arch-sync` — Architecture Drift Detection
```yaml
---
name: arch-sync
description: Compare actual codebase against architecture documentation and flag drift. Use when the user says "check architecture", "is the code in sync", or periodically after major changes.
context: fork
agent: Explore
allowed-tools: Read, Glob, Grep, Bash
---
```

**Logic:**
1. Read architecture source of truth:
   - `/agent-arch/0-main.md` — finalized architecture
   - `/trybot_api/docs/*.md` — feature-level docs
   - `/trybot_api/AGENTS.md` — project structure
2. Scan actual codebase:
   - Directory structure vs documented structure
   - Key patterns (checkpointer, registry, adapters) vs documented patterns
   - API endpoints vs documented endpoints
3. Flag drift:
   - Code exists but not documented
   - Documentation references code that doesn't exist
   - Pattern divergence (e.g., direct DB calls bypassing `database_utils.py`)
4. Output: drift report with actionable items

**Invocation:** Manual only

---

### The Autonomous Dev Loop

#### `/dev-loop` — Full Ticket-to-PR Automation
```yaml
---
name: dev-loop
description: Full autonomous development loop from Jira ticket to merged PR. Use when the user says "dev loop", "auto-develop", or "take this ticket end to end".
argument-hint: <ticket-id>
disable-model-invocation: true
allowed-tools: Read, Edit, Write, Glob, Grep, Bash, mcp__jira__*
---
```

**The Pipeline:**

```
┌──────────────────────────────────────────────────────────┐
│                    /dev-loop TB-XXX                       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  PHASE 1: Intake (Claude default + CLAUDE.md)            │
│  ─────────────────────────────────────────────           │
│  ├─ Fetch ticket from Jira via MCP                       │
│  ├─ Parse requirements + acceptance criteria             │
│  ├─ Create branch: feat/TB-XXX-description               │
│  ├─ Transition ticket → "In Progress"                    │
│  ├─ Analyze codebase for affected areas                  │
│  ├─ Create implementation plan                           │
│  ├─ /jira-update TB-XXX planning                         │
│  └─ ⏸ CHECKPOINT: Present plan for user approval         │
│                                                          │
│  PHASE 2: Implementation (Claude default + CLAUDE.md)    │
│  ─────────────────────────────────────────────           │
│  ├─ Execute plan step by step                            │
│  ├─ Follow AGENTS.md conventions                         │
│  ├─ Scaffold endpoints/skills following existing patterns │
│  ├─ Commit after each logical unit                       │
│  └─ /jira-update TB-XXX implementation                   │
│                                                          │
│  PHASE 3: Testing (Skills)                               │
│  ─────────────────────────────────────────────           │
│  ├─ /test-local — hit live API, verify DB, check logs    │
│  ├─ /test-write — pytest tests per project patterns      │
│  ├─ pytest -v (CLAUDE.md instruction)                    │
│  └─ If failures: fix → rerun (max 3 attempts)           │
│                                                          │
│  PHASE 4: Quality (Skill)                                │
│  ─────────────────────────────────────────────           │
│  ├─ /quality — ruff + import check + secret scan         │
│  └─ ⏸ CHECKPOINT: Present diff for user approval         │
│                                                          │
│  PHASE 5: Ship (Skill)                                   │
│  ─────────────────────────────────────────────           │
│  ├─ /ship — push → PR → Jira update → transition        │
│  └─ ✅ Done: PR URL + Jira ticket updated                │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Checkpoints (Human-in-the-Loop):**
- After Phase 1 (Planning): User approves implementation plan
- After Phase 4 (Quality): User reviews diff before PR
- These are the only two points where human input is required

**What's Claude default vs what's a skill:**
- Phases 1-2 are Claude's natural behavior, guided by conventions in CLAUDE.md
- Phases 3-5 are skills that encode non-obvious, multi-step project-specific workflows

---

## 3. Gaps & Recommendations

### 3.1 Missing Infrastructure

| Gap | Impact | Recommendation |
|-----|--------|----------------|
| No `CLAUDE.md` | AGENTS.md not auto-loaded by Claude Code | Create CLAUDE.md (see section 1.1) |
| No `.claude/skills/` | No skills exist yet | Create 12 skill files (this plan) |
| No `httpx` in requirements | Harder to test locally in `/test-local` | Add `httpx` to dev dependencies |
| No `pytest-asyncio` in requirements.in | Async tests not explicitly configured | Add to dev dependencies |

### 3.2 Future Skills (Not in v1)

| Skill | Purpose | When to add |
|-------|---------|-------------|
| `/batch-dev` | Run `/dev-loop` on N tickets in parallel (worktree isolation) | After v1 is validated — this is the real multiplier |
| `/migrate` | Generate SQL migrations for schema changes | When Alembic is set up |
| `/api-diff` | Compare OpenAPI spec before/after changes | When API contracts become critical |

### 3.3 Security

- DB access: always `$SUPABASE_DB_URI` env var, never inline credentials
- Local API testing: `$TEST_JWT` env var, never hardcoded tokens
- `/ship` and `/dev-loop` are `disable-model-invocation: true` (manual only) — they have external side effects

---

## 4. Implementation Order

### Step 1: Foundation
1. Create `CLAUDE.md` in trybot_api (absorbs all default-behavior conventions)
2. Update `settings.local.json` with new Bash permissions (curl, pytest, ruff)
3. Create `.claude/skills/` directory

### Step 2: Testing Skills
4. `/test-local`, `/test-write` — testing skills
5. Test with a real ticket end-to-end (manually invoke skills)

### Step 3: Quality + Ship
6. `/quality` — pre-ship gate
7. `/ship`, `/jira-update` — shipping pipeline
8. Test full flow: implement → test → quality → ship

### Step 4: Debug + Maintenance
9. `/investigate`, `/db` — debugging skills
10. `/standup`, `/arch-sync` — maintenance skills

### Step 5: Autonomous Loop
11. `/dev-loop` — chains everything
12. End-to-end validation with a real Jira ticket
13. Team rollout to Krishna and Rishav

---

## 5. Success Metrics

| Metric | Before | Target |
|--------|--------|--------|
| Ticket → PR time | 4-8 hours (manual) | 30-60 min (automated) |
| Human intervention points | Continuous | 2 checkpoints (plan approval, pre-PR review) |
| Test coverage per ticket | Ad hoc | 100% of acceptance criteria |
| Jira update frequency | End of task | Real-time (per phase) |
| Architecture drift | Undetected | Caught per PR |
