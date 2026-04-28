# Skill Creation On-the-Fly — Plan

> Owner: KPR
> Date: April 2026
> Status: Draft — needs refinement
> Dependencies: Skill Registry, Execution Engine

---

## What This Is

Two agents that expand the system's capabilities — finding and integrating new skills. One works at runtime (triggered when the main agent can't find a skill for a task), and one works with a human admin (guided integration through a dashboard).

**Growth model: organic, demand-driven.** Skills are only discovered when actually needed during real tasks. No speculative crawling. Every skill candidate in the database has a proven demand behind it.

---

## The Two Agents

### Agent 1: Skill Creation On-the-Fly Agent

**Purpose:** When the main agent encounters a task it has no skill for, this agent is triggered to find and provide a usable skill — fast, within a timeout window.

**Trigger:** Main agent already searched the skill registry and confirmed no matching skill exists.

**How it works:**

```
Main Agent: "I need a skill that gets air quality for a city"
    │
    ▼
On-the-Fly Agent activated:
    │
    ├── Step 1: Search for the capability
    │     → Search skill_candidates (pgvector semantic search) — previously discovered skills
    │     → If nothing found → web search to find a matching API/service (within timeout)
    │
    ├── Step 2: Evaluate what's found
    │     → Does it need authentication? API key? Payment?
    │     → Does it need developer setup? OAuth app? Webhook config?
    │     → Classify: "no_intervention_needed" or "needs_admin"
    │
    ├── Step 3a: No intervention needed
    │     → Test the API/service (sandbox call with sample input)
    │     → If works → provide skill details to Main Agent
    │     → Main Agent uses it for the CURRENT task only
    │     → Log in skill_candidates: status="used_at_runtime_pending_admin"
    │     → NOTE: even no-intervention skills are NOT auto-registered
    │       in the skill registry. Admin verification is always required
    │       before permanent registration. The On-the-Fly Agent provides
    │       the skill for ONE-TIME use. Permanent registration goes
    │       through the Integration Agent (admin dashboard).
    │
    └── Step 3b: Intervention needed
          → Log in skill_candidates: status="pending_admin"
          → Generate intervention_steps (from API docs)
          → Return to Main Agent: "Can't provide this now. Logged for admin."
          → Main Agent: "I can't do that right now. I'll flag it for setup."
```

**Timeout behavior:**

```
Main Agent triggers On-the-Fly Agent
    │
    ├── Sets timeout: X seconds (configurable per use case)
    │
    ├── On-the-Fly Agent searches...
    │
    ├── WITHIN timeout + no intervention needed:
    │     → Main Agent receives skill details → uses it
    │
    ├── WITHIN timeout + intervention needed:
    │     → Main Agent informed: "needs admin setup"
    │     → Main Agent tells caller: "I can't do that right now"
    │
    └── TIMEOUT expires (agent still searching):
          → Main Agent proceeds without the skill
          → On-the-Fly Agent continues in background
          → If found later → logged in skill_candidates for future
```

**What the On-the-Fly Agent returns to Main Agent (when successful):**

```
{
    "found": true,
    "needs_intervention": false,
    "skill_details": {
        "name": "Air Quality Index",
        "description": "Get real-time AQI for any city",
        "how_to_use": {
            "url": "https://api.waqi.info/feed/{city}/",
            "method": "GET",
            "params": { "token": "demo" },
            "response_path": "data.aqi"
        },
        "tested": true,
        "test_result": { "input": "mumbai", "output": { "aqi": 156 } }
    },
    "candidate_id": "uuid-xxx"  // logged in skill_candidates
}
```

---

### Agent 2: Integration Agent (Admin UI)

**Purpose:** Admin-facing agent that presents discovered skill candidates and guides the admin through integration — step by step.

**Trigger:** Admin opens the integration dashboard.

**What the admin sees:**

```
DASHBOARD SECTIONS:

  1. PENDING SKILLS (all skills discovered by On-the-Fly Agent)
     → Sorted by demand (gap_count — how many times this skill was needed)
     → Shows: what it does, which task triggered it, what steps are needed
     → Two types:
        - No intervention needed → admin reviews + approves → agent tests + integrates
        - Intervention needed → admin clicks → guided step-by-step flow
     → Admin clicks → guided integration flow

  2. RECENTLY INTEGRATED
     → Skills that were successfully integrated
     → Shows: when, by whom, test results

  3. REJECTED / FAILED
     → Skills that didn't pass testing or admin rejected
```

**Guided integration flow (when admin clicks a skill):**

```
Integration Agent reads the skill's intervention_steps
Presents each step to the admin:

  For "account_creation" step:
    → Shows: where to go, what to create
    → Admin confirms when done

  For "credential_input" step:
    → Shows: what credential to find, where to find it
    → Provides input field for admin to paste
    → Agent validates format (API key pattern, token format)

  For "permission_setup" step:
    → Shows: what permissions/scopes to enable
    → Admin confirms when done

  For "payment_required" step:
    → Shows: pricing info, what plan is needed
    → Admin confirms purchase completed
    → Provides input for billing-related credentials if needed

  For "auto_test" step:
    → Agent runs automatically after human steps complete
    → Tests the skill with sample inputs using provided credentials
    → Shows results: pass/fail with details
    → If fail: suggests what might be wrong

  For "auto_register" step:
    → Depends on skill registry module design
    → Could be: DB insert, code generation + branch + CI/CD, or other
    → Agent handles whatever the registry requires
```

**The Integration Agent generates steps dynamically:**

The agent doesn't have hardcoded steps for every service. When a candidate is discovered:

```
Crawling Agent finds Slack API:
  → Fetches https://api.slack.com/docs
  → LLM analyzes documentation:
      "What does a human need to do to set this up?"
  → LLM generates structured steps:
      1. account_creation: "Create Slack app at api.slack.com/apps"
      2. credential_input: "Copy Bot User OAuth Token (xoxb-...)"
      3. permission_setup: "Add scopes: chat:write, channels:read"
      4. auto_test: (agent runs this)
      5. auto_register: (depends on registry design)
  → Stores in skill_candidates.intervention_steps
```

Different APIs produce different steps — the LLM adapts based on each API's documentation.

---

## skill_candidates Table

The central tracking system for both agents:

```sql
CREATE TABLE skill_candidates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Discovery (always by On-the-Fly Agent, triggered by real demand)
    discovered_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    source_url TEXT,              -- where the skill/API was found
    triggered_by_task_id UUID,    -- bot_task that triggered the gap
    
    -- What was found
    name TEXT NOT NULL,
    description TEXT NOT NULL,
    description_embedding VECTOR(1536),  -- pgvector embedding for semantic search
    category TEXT,               -- 'weather' | 'communication' | 'calendar' | etc.
    spec JSONB,                  -- OpenAPI spec, MCP tool def, or raw API details
    
    -- Classification
    intervention_type TEXT NOT NULL,  -- 'none' | 'api_key' | 'oauth' | 'payment' | 'infrastructure'
    intervention_steps JSONB,         -- array of step objects for admin
    
    -- Status
    status TEXT NOT NULL DEFAULT 'pending_admin',
    -- 'pending_admin': discovered, waiting for admin review
    -- 'used_at_runtime': on-the-fly agent found it, main agent used it
    -- 'pending_admin': needs human intervention, waiting on dashboard
    -- 'admin_in_progress': admin started guided flow
    -- 'testing': agent running automated tests
    -- 'verified': tests passed, ready for registration
    -- 'registered': integrated into skill registry
    -- 'rejected': admin rejected or tests failed permanently
    -- 'failed': integration failed
    
    -- Admin interaction
    admin_user_id UUID,           -- which admin is working on this
    admin_started_at TIMESTAMPTZ,
    credentials JSONB,            -- encrypted, admin-provided tokens/keys
    
    -- Testing
    test_results JSONB,           -- array of test runs with pass/fail
    last_tested_at TIMESTAMPTZ,
    
    -- Registration (once integrated)
    registered_skill_id TEXT,     -- skill_id in the skill registry
    registered_at TIMESTAMPTZ,
    
    -- Priority
    priority INTEGER DEFAULT 0,   -- higher = more urgent (gap-log sourced = higher)
    gap_count INTEGER DEFAULT 0,  -- how many times main agent needed this and couldn't find it
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Priority logic:**

```
Every candidate is demand-driven (triggered by a real task):
  → gap_count increments each time the same skill is needed again
  → Most-needed skills bubble to top of admin dashboard
  → Admin sees which skills are blocking the most tasks
```

---

## How the Two Agents Interact

```
┌─────────────────────────────────────────────────────────────────┐
│ MAIN AGENT (during conversation)                                 │
│                                                                  │
│ "I need a skill for X"                                           │
│    │                                                             │
│    ▼                                                             │
│ Search skill registry → NOT FOUND                                │
│    │                                                             │
│    ▼                                                             │
│ Trigger On-the-Fly Agent (with timeout)                          │
│    │                                                             │
│    ├── Skill found + no intervention needed:                     │
│    │     → On-the-Fly Agent provides skill for ONE-TIME use      │
│    │     → Main Agent uses it for the current task               │
│    │     → Logged in skill_candidates for admin verification     │
│    │                                                             │
│    ├── Skill found + intervention needed:                        │
│    │     → Logged in skill_candidates as pending_admin           │
│    │     → Main Agent: "I can't do that right now"               │
│    │                                                             │
│    └── Timeout (nothing found):                                  │
│          → Main Agent proceeds without the skill                 │
│          → Gap logged in skill_candidates for future reference   │
│                                                                  │
└──────────────────────────────────────────┬──────────────────────┘
                                           │
                                           ▼
                              ┌──────────────────────┐
                              │  skill_candidates     │
                              │  (central tracking)   │
                              │                       │
                              │  Every skill found    │
                              │  by On-the-Fly Agent  │
                              │  is logged here.      │
                              │  All demand-driven.   │
                              └──────────────┬────────┘
                                             │
                                             ▼
                              ┌──────────────────────┐
                              │ INTEGRATION AGENT     │
                              │ (Admin UI)            │
                              │                       │
                              │ Admin sees all        │
                              │ pending skills,       │
                              │ sorted by demand.     │
                              │                       │
                              │ Guides admin through  │
                              │ setup if needed.      │
                              │ Tests + verifies.     │
                              │ Registers in skill    │
                              │ registry on approval. │
                              └──────────────────────┘
```

---

## Open Questions (to refine)

1. **How does a skill get into the skill registry?** Depends on skill registry (M2) module design — could be DB insert, code generation, or other. This plan tracks candidates and guides integration but doesn't assume the registry mechanism.

2. **What timeout should the Main Agent use?** Configurable per context. Voice call: 5-10 seconds. Chat/WhatsApp: 30-60 seconds. Background task: no timeout.

3. **How does the On-the-Fly Agent search the web?** Needs a web search + web fetch capability. Could use an existing search API skill or direct web crawling.

4. **How are credentials stored securely?** Admin-provided tokens and API keys need encryption at rest. Existing `skill_data_store` or a dedicated secrets table.

5. **No-intervention skills still need admin verification.** Even if a skill needs no auth/purchasing, it is NOT auto-registered. The On-the-Fly Agent can use it once for the current task, but permanent registration in the skill registry always requires admin review through the Integration Agent dashboard. This is a deliberate safety decision — no skill enters the permanent registry without human verification.

6. **How does the On-the-Fly Agent avoid duplicate discoveries?** Dedup by source_url + name in skill_candidates. If the same skill is needed again, increment gap_count instead of creating a new candidate.

7. **How does the Integration Agent handle code generation?** If a skill needs a custom handler (not generic HTTP), the agent would need to write code, push to a branch, trigger CI/CD. This is a complex flow — may need developer review even with agent-generated code.

8. **How does this relate to the broader skill management module?** The On-the-Fly Agent's discovery and the Integration Agent's verification + registration overlap with the skill management pipeline. Need to align — clear ownership of discover, validate, and promote steps.
