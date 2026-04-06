# Foundation Sprint — Two Prompts -> Complete

---

# PROMPT 1: Database Migration (UI Repo)

> Paste into a Claude Code session pointed at the Trybo **UI/React Native repo** (where Supabase migrations live).

---

## Context

Trybo's Supabase schema migrations are managed from the UI repo. The FastAPI backend assumes tables already exist and never modifies the database schema.

We need 2 new tables for the orchestration engine's application use cases. These are generic platform tables — not feature-specific. Create a single migration file following whatever migration pattern already exists in this repo (check `supabase/migrations/` or similar).

## Migration: Foundation Platform Tables

Create a new migration file with the next sequential timestamp/number following the existing convention.

### Table 1: `skill_data_store`

Generic persistence layer for structured, queryable, per-user data produced by skills. One table for all use cases — namespace isolation keeps them separate.

```sql
-- Skill Data Persistence Layer
-- Generic table for ALL skill-produced persistent data.
-- No feature-specific tables — every use case writes here with a unique namespace.
-- Examples: namespace="commute" for travel measurements, namespace="digest" for daily digests,
--           namespace="briefing" for user preferences, namespace="user_profile" for push tokens.

CREATE TABLE skill_data_store (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  namespace TEXT NOT NULL,
  record_type TEXT NOT NULL,
  payload JSONB NOT NULL,
  recorded_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  metadata JSONB
);

-- Primary lookup: user + namespace + type + time range
CREATE INDEX idx_skill_data_lookup
  ON skill_data_store(user_id, namespace, record_type, recorded_at DESC);

-- JSONB containment queries (e.g., WHERE payload @> '{"day_of_week": 1}')
CREATE INDEX idx_skill_data_payload
  ON skill_data_store USING GIN (payload);

-- RLS: users can only access their own data
ALTER TABLE skill_data_store ENABLE ROW LEVEL SECURITY;

CREATE POLICY skill_data_store_user_policy ON skill_data_store
  FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);
```

### Table 2: `scheduled_jobs`

Cron-like scheduled task executor. Supports two modes: lightweight single-skill invocation (e.g., Maps API query every 15 min) and full graph execution (e.g., daily morning briefing).

```sql
-- Scheduled Job Executor
-- Any skill can register a recurring job. The backend worker polls this table
-- and triggers execution when next_run <= now().

CREATE TABLE scheduled_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  job_type TEXT NOT NULL CHECK (job_type IN ('skill_invocation', 'graph_execution')),
  cron_expression TEXT NOT NULL,
  config JSONB NOT NULL,
  next_run TIMESTAMPTZ NOT NULL,
  last_run TIMESTAMPTZ,
  enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Worker polls for due jobs: WHERE enabled = TRUE AND next_run <= now()
CREATE INDEX idx_scheduled_jobs_next_run
  ON scheduled_jobs(next_run)
  WHERE enabled = TRUE;

-- User lookup
CREATE INDEX idx_scheduled_jobs_user
  ON scheduled_jobs(user_id);

-- RLS: users can only access their own jobs
ALTER TABLE scheduled_jobs ENABLE ROW LEVEL SECURITY;

CREATE POLICY scheduled_jobs_user_policy ON scheduled_jobs
  FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);
```

## Verification

- [ ] Migration file follows existing naming convention in the repo
- [ ] Both tables created successfully in local/dev Supabase
- [ ] RLS policies are active
- [ ] Indexes are created
- [ ] No conflicts with existing tables
- [ ] `users(id)` foreign key reference matches the actual users table in the schema

## What NOT to do

- Do not modify any existing tables
- Do not add seed data
- Do not create any application code — that's the backend repo's job

---
---

# PROMPT 2: Backend Foundation (FastAPI Repo)

> Paste into a Claude Code session pointed at the Trybo **FastAPI backend repo**.

---

## Context

This is a FastAPI backend for Trybo — a mobile-first personal assistant that orchestrates multi-step tasks. The agentic orchestration engine (planner, executor, consent, MCP adapters, mem0 memory) is already built. We are now building 5 application use cases (Commute Optimizer, Morning Briefing, Intelligence Digest, Trip Planner, Purchase Advisor) that require shared platform modules.

This session builds the **foundation modules** that all use cases depend on. After this session completes, 3 parallel Claude Code instances will build the use cases simultaneously — each writing to separate files to avoid merge conflicts.

**DATABASE NOTE:** The following tables already exist in Supabase (migrations are managed by the UI repo, not this backend). Do NOT create migrations or run any DDL. Just use these tables as-is:

- `skill_data_store` — columns: `id` (UUID PK), `user_id` (UUID FK→users), `namespace` (TEXT), `record_type` (TEXT), `payload` (JSONB), `recorded_at` (TIMESTAMPTZ), `created_at` (TIMESTAMPTZ), `metadata` (JSONB). Indexes on `(user_id, namespace, record_type, recorded_at)` and GIN on `payload`.
- `scheduled_jobs` — columns: `id` (UUID PK), `user_id` (UUID FK→users), `job_type` (TEXT: 'skill_invocation' | 'graph_execution'), `cron_expression` (TEXT), `config` (JSONB), `next_run` (TIMESTAMPTZ), `last_run` (TIMESTAMPTZ), `enabled` (BOOLEAN), `created_at` (TIMESTAMPTZ). Index on `next_run WHERE enabled = TRUE`.

**IMPORTANT:** Before writing any code, explore the codebase first. Understand the existing patterns:
- How are services structured? (check `app/services/` — e.g., `memory/`, `llm_service.py`)
- How is config managed? (check `app/config/__init__.py`)
- Where does the orchestration code live? (check `app/agentic_mvp/` or `app/orchestration/`)
- How are skills registered? (check `seed_registry.py` or similar)
- How are adapters structured? (check existing direct/mcp adapter files)
- What's in `requirements.in` or `requirements.txt`?
- How does the app access Supabase? (check existing DB access patterns — direct client, ORM, etc.)

Follow existing patterns exactly. Do not invent new patterns.

---

## Task 1: Skill Data Store (Persistence Layer)

A generic persistence service for structured, queryable, per-user data that skills produce. The `skill_data_store` table already exists in Supabase.

### Service to create:

Follow the same singleton/service pattern as the existing MemoryService (`app/services/memory/`). Place this alongside it — likely `app/services/persistence/` or wherever makes sense given the existing structure.

```python
class SkillDataStore:
    """Generic persistence layer for skill-produced data.
    No skill-specific logic lives here. Namespace isolation between skills."""

    async def write(self, user_id: str, namespace: str, record_type: str,
                    payload: dict, recorded_at: datetime, metadata: dict | None = None) -> str:
        """Insert a record into skill_data_store. Returns the record ID."""

    async def query(self, user_id: str, namespace: str, record_type: str,
                    since: datetime | None = None, until: datetime | None = None,
                    payload_filter: dict | None = None,
                    limit: int | None = None, order: str = "desc") -> list[dict]:
        """Query records. payload_filter uses JSONB containment (@>).
        Order by recorded_at. Returns list of record dicts."""

    async def query_latest(self, user_id: str, namespace: str,
                           record_type: str) -> dict | None:
        """Get most recent record of this type."""

    async def delete(self, user_id: str, namespace: str,
                     record_type: str | None = None, before: datetime | None = None) -> int:
        """Delete records. Returns count deleted."""

    async def count(self, user_id: str, namespace: str,
                    record_type: str, since: datetime | None = None) -> int:
        """Count records matching criteria."""
```

Use the Supabase Python client (already in the project) for database operations. Follow the existing DB access pattern in the codebase.

Create a `get_skill_data_store()` singleton function following the same pattern as `get_memory_service()`.

### Pydantic models:

```python
class SkillDataRecord(BaseModel):
    id: str
    user_id: str
    namespace: str
    record_type: str
    payload: dict
    recorded_at: datetime
    created_at: datetime
    metadata: dict | None = None
```

---

## Task 2: Scheduled Job Executor

A scheduler service that manages cron jobs and a background worker that executes them. The `scheduled_jobs` table already exists in Supabase.

### Service to create:

Place alongside other services, following existing patterns.

```python
class SchedulerService:
    """Manages scheduled jobs — cron-like recurring task execution."""

    async def register_job(self, user_id: str, job_type: str,
                           cron_expression: str, config: dict) -> str:
        """Register a new scheduled job. Computes initial next_run from cron_expression. Returns job ID."""

    async def get_user_jobs(self, user_id: str) -> list[dict]:
        """List all jobs for a user."""

    async def update_job(self, job_id: str, enabled: bool | None = None,
                         cron_expression: str | None = None,
                         config: dict | None = None) -> None:
        """Update job settings. Recompute next_run if cron_expression changes."""

    async def delete_job(self, job_id: str) -> None:
        """Remove a scheduled job."""

    async def get_due_jobs(self) -> list[dict]:
        """Get all enabled jobs where next_run <= now(). Called by the background worker."""

    async def mark_executed(self, job_id: str, next_run: datetime) -> None:
        """Update last_run to now() and set next_run. Called after job execution."""
```

### Background worker:

Create a lightweight background task runner that:
1. Runs as a FastAPI background task (check if APScheduler or similar is already in the project; if not, use a simple asyncio loop with a polling interval)
2. Every 60 seconds: calls `get_due_jobs()`
3. For each due job:
   - `job_type: "skill_invocation"` → call the skill adapter directly with the config params (skill_id and input from `config` JSONB)
   - `job_type: "graph_execution"` → trigger the orchestration pipeline (however the existing codebase triggers a plan execution from a user message — replicate that pattern but with the job's config as the input)
4. After execution: call `mark_executed()` with the next computed run time
5. Wrap each job execution in try/except — a failing job must not crash the worker or block other jobs

For computing `next_run` from a cron expression, use the `croniter` library. Add `croniter` to requirements.

Hook the background worker into FastAPI's lifespan (startup/shutdown) following whatever pattern the app already uses (check for existing lifespan hooks, startup events, etc.).

---

## Task 3: Push Notification Skill

A Direct adapter skill that sends push notifications via Firebase Cloud Messaging (FCM).

### What to build:

1. **Handler function** — create a new handler file in the direct adapter directory (follow the pattern of existing handlers like whatsapp_send or similar):

```python
async def push_notification_send(input: dict, context: Any) -> dict:
    """
    Send a push notification to a user.

    Input schema:
      - user_id: str (required)
      - title: str (required)
      - body: str (required)
      - data: dict (optional — custom payload for the mobile app to handle)

    Implementation:
      - Query skill_data_store for the user's push token:
        namespace="user_profile", record_type="push_token"
      - Use firebase-admin SDK to send the notification
      - Return: {success: bool, message_id: str}
    """
```

2. **Skill definition** — create a skill definitions file for notifications (see Task 6 for file structure):

```python
NOTIFICATION_SKILLS = [
    SkillDefinition(
        id="notification.push.send",
        name="Push Notification",
        kind="tool",
        adapter_type="direct",
        capabilities=["notification.push.send"],
        description="Send a push notification to the user's mobile device. Used for delivering daily briefings, commute alerts, and digest summaries.",
        input_schema={...},  # title, body, data
        output_schema={...},  # success, message_id
        side_effect=True,
        requires_consent=True,
        consent_risk_level="low",
    )
]
```

3. **Config** — add to app config:
```python
FIREBASE_CREDENTIALS_PATH: str | None = None  # path to Firebase service account JSON
```

4. Add `firebase-admin` to requirements.

**Note:** If firebase-admin setup is complex or credentials aren't available, stub the handler with a TODO and a mock response that returns `{success: True, message_id: "mock_id"}`. The interface and skill registration are what matter for the parallel sprint.

---

## Task 4: Artifact System (Backend Models)

Define the backend data models for structured artifacts. The mobile rendering side will be built separately.

### What to build:

Add to the shared models directory (wherever `PlanSpec`, `SkillResult`, etc. are defined):

```python
class StructuredArtifact(BaseModel):
    """Rich result card sent to mobile for rendering.
    Mobile app has a renderer registry mapping artifact_type → React Native component."""
    artifact_type: str          # e.g., "briefing_card", "digest_card", "itinerary_card"
    payload: dict               # type-specific data — mobile renderer knows the shape
    title: str | None = None
    created_at: datetime = Field(default_factory=datetime.utcnow)

# Pre-defined artifact types as constants:
ARTIFACT_TYPES = {
    "briefing_card",
    "digest_card",
    "itinerary_card",
    "comparison_table",
    "commute_alert",
    "consent_card",
}
```

Ensure that `SkillResult` (or whatever the executor's node output model is) can carry a `StructuredArtifact` alongside its regular output. Check the existing `SkillResult` model and add an optional `artifact: StructuredArtifact | None = None` field if it doesn't already support rich outputs.

---

## Task 5: Parallel Fan-Out Fix in Executor

The current executor may serialize nodes even when they have no dependencies between them. Fix it to run independent nodes in parallel.

### What to do:

1. Find the graph compilation code (likely `plan_builder.py` or `compiler_graph.py` — explore first)
2. The current implementation probably does topological sort and adds nodes/edges sequentially
3. Modify to detect independent nodes (nodes whose `depends_on` lists don't reference each other) and execute them in parallel using LangGraph's `Send` API or parallel state branches
4. Independent nodes = nodes at the same depth in the dependency graph with no edges between them

This is the riskiest foundation task because it modifies existing executor code. Be conservative:
- Keep the existing sequential path as a fallback
- Add a feature flag: `PARALLEL_FANOUT_ENABLED: bool = True` in config
- If the existing code already handles this correctly (check!), just verify and move on — don't fix what isn't broken
- Test with a simple 2-branch parallel plan before declaring done

---

## Task 6: Per-Use-Case Skill Registration Structure

Set up the file structure that allows 3 parallel Claude Code instances to add skills without merge conflicts.

### What to do:

1. Find the current skill registration file (likely `seed_registry.py` or similar)
2. Create separate skill definition files for each use case — place them alongside the existing registration file:

```
skills_commute.py          # UC1 — will be populated by parallel Instance 1
skills_briefing.py         # UC4 — will be populated by parallel Instance 1
skills_digest.py           # UC5 — will be populated by parallel Instance 2
skills_travel.py           # UC2 — will be populated by parallel Instance 3
skills_purchase.py         # UC3 — will be populated by parallel Instance 3
skills_notifications.py    # Push notification — built in Task 3 above
```

3. Each file exports a list: `COMMUTE_SKILLS = [...]`, `BRIEFING_SKILLS = [...]`, etc.
4. The main registry file imports and includes all of them:

```python
from .skills_commute import COMMUTE_SKILLS
from .skills_briefing import BRIEFING_SKILLS
from .skills_digest import DIGEST_SKILLS
from .skills_travel import TRAVEL_SKILLS
from .skills_purchase import PURCHASE_SKILLS
from .skills_notifications import NOTIFICATION_SKILLS

ALL_USE_CASE_SKILLS = [
    *COMMUTE_SKILLS,
    *BRIEFING_SKILLS,
    *DIGEST_SKILLS,
    *TRAVEL_SKILLS,
    *PURCHASE_SKILLS,
    *NOTIFICATION_SKILLS,
]
```

Make sure `ALL_USE_CASE_SKILLS` is included in whatever the existing function returns that registers skills (e.g., `seed_skill_definitions()` or similar — check the codebase).

5. Start each file (except `skills_notifications.py`) with an empty list and a docstring:

```python
"""UC1: Smart Commute Optimizer — skill definitions.
Will be populated by a parallel Claude Code instance.
Do not edit this file from other instances."""

COMMUTE_SKILLS: list = []
```

---

## Execution Order

```
Task 1 (Skill Data Store)  ─── no dependency, start first
Task 2 (Scheduler)         ─── no dependency, can parallel with Task 1
Task 4 (Artifact Models)   ─── no dependency, can parallel with Task 1
Task 6 (Skill File Setup)  ─── no dependency, quick, do anytime
Task 3 (Push Notification) ─── depends on Task 1 (queries push token from data store) + Task 6 (writes skill def)
Task 5 (Parallel Fan-Out)  ─── no dependency on Tasks 1-4, but riskiest — do last
```

---

## Requirements to add:

```
croniter          # cron expression parsing for scheduler
firebase-admin    # FCM push notifications
```

---

## Verification checklist:

- [ ] `SkillDataStore.write()` and `query()` work against Supabase (the table already exists)
- [ ] `SchedulerService.register_job()` creates a job, `get_due_jobs()` returns it when due
- [ ] Background worker starts with the app and polls for due jobs
- [ ] Push notification handler sends via FCM (or stub with mock response is in place)
- [ ] `StructuredArtifact` model exists and `SkillResult` can carry one
- [ ] Independent executor nodes run in parallel when feature flag is on
- [ ] 6 skill definition files exist (5 empty + 1 for notifications) and all are imported by the main registry
- [ ] App starts and runs without errors with all new modules
- [ ] No existing tests broken

---

## What NOT to build in this session:

- **Database migrations** — tables already exist, managed by UI repo
- Individual use case skills (Maps, Weather, RSS, Flights, etc.) — parallel sprint
- Mobile UI components — artifact renderers are mobile-side work
- GID route definitions for new use cases — parallel instances will add these
- Any use-case-specific business logic
