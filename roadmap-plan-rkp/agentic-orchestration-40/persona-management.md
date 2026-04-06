# Persona Management — Baseline Definition

> Owner: RKP
> Module mapping: NEW — not in current 11-module architecture
> Current completion: 0% — nothing built

---

## What This Is

A persona is a specialized agent configuration that determines *who the agent is* for a given interaction. Not just voice and tone — a persona constrains what the agent can do, how it communicates, and what it remembers.

**Target personas (10-20, starting with these 5):**

| Persona | Specialization | Example interaction |
|---------|---------------|-------------------|
| **Chief of Staff / EA** | Calendar, email, calls, task management. Plans thoroughly. Proactive. | "Clear my afternoon and reschedule the 3pm with Sharma to Friday" |
| **Research Assistant** | Web search, document analysis, data synthesis. Fast, concise. | "Compare pricing of AWS vs GCP for our workload" |
| **Personal Assistant** | Reminders, shopping, travel, daily logistics. Warm, conversational. | "Book a cab to the airport for my 6pm flight" |
| **Customer Support Agent** | Ticket resolution, FAQ, escalation. Professional, empathetic. | "My order hasn't arrived. Order #4521" |
| **Sales Assistant** | CRM, outreach, follow-ups, pitch prep. Assertive, data-driven. | "Draft follow-up emails for everyone I met at the conference" |

## Why This Is a Separate Module

Personas touch multiple modules but aren't owned by any of them:

| What a persona controls | Which module is affected |
|------------------------|------------------------|
| System prompt (personality, tone, boundaries) | M3 (Planner) — persona prompt injected into planning context |
| Allowed skills (EA can schedule, but can't do research) | M2 (Skill Management) — skill catalog filtered per persona |
| Voice (specific TTS voice, speed, expressiveness) | M10/Voice Intelligence — voice config per persona |
| Communication style (concise vs thorough, formal vs casual) | M3 (Planner) + M10 (Voice) — affects both text and voice output |
| Memory namespace (EA's context separate from Research Assistant's) | M7 (Memory) — episodic + semantic memory partitioned |
| Approval thresholds (EA auto-approves scheduling, Support doesn't) | M8 (Intelligence) — risk tiers configurable per persona |

No existing module owns all of these. Persona Management is the configuration layer that binds them.

---

## Baseline Definition

**The line:** Each persona is a config object that the planner, executor, and voice pipeline consume. Switching personas changes the agent's behavior, voice, skill access, and memory context. A user has a default persona that can be switched via chat or automatically based on the interaction type. Anything beyond this is refinement.

---

## Baseline Components

### 1. Persona Config Object

Each persona is defined as a structured configuration:

```
Persona:
  id: "chief-of-staff"
  display_name: "Ava — Chief of Staff"

  llm:
    system_prompt: "You are a Chief of Staff / Executive Assistant. You manage
                    the user's calendar, email, and calls. Plan thoroughly —
                    check for conflicts before scheduling, draft emails before
                    sending. Be proactive: suggest prep for upcoming meetings."
    temperature: 0.4        # lower = more predictable for scheduling
    max_response_length: 5  # sentences — concise for action-oriented tasks

  skills:
    allowed: ["calendar.*", "email.*", "call.*", "reminder.*", "contact.*"]
    denied: ["web_search.*", "document.*"]
    # Chief of Staff can schedule and communicate, but doesn't do research

  voice:
    provider: "elevenlabs"
    voice_id: "ava_professional"
    settings: { stability: 0.7, speed: 1.0, similarity_boost: 0.8 }

  behavior:
    interruption_sensitivity: "medium"  # doesn't interrupt aggressively
    filler_words: false                 # professional, no "umm"
    approval_threshold: "low"           # auto-approves scheduling, asks for calls
    proactive_suggestions: true         # "You have a meeting in 30min, need prep?"

  memory:
    namespace: "chief-of-staff"         # separate from other personas
    shared_profile_access: true         # can read user's global profile
```

This is stored as a YAML/JSON config file or in `app_configurations` table. Not hardcoded — new personas can be added without code changes.

### 2. Persona Context in Planning

The planner receives the active persona's config as input alongside the user message, skill catalog, and memory context. This changes planning behavior:

**What the persona constrains in the planner:**

| Persona field | How it affects planning |
|--------------|----------------------|
| `system_prompt` | Sets the agent's personality, priorities, and boundaries |
| `skills.allowed` | Planner only sees skills in the allowed list — can't plan with denied skills |
| `temperature` | Affects creativity vs predictability in plan generation |
| `max_response_length` | Synthesis step respects this limit |
| `approval_threshold` | Overrides the default risk-tier consent behavior (see agent-intelligence.md) |

**Example:** User says "find me a restaurant and book a table."
- **Personal Assistant** persona: Plans `web_search.restaurants` → `call.initiate` (to make reservation). Has access to both.
- **Chief of Staff** persona: Only has call/calendar skills. Plans `call.initiate` (to the user's assistant) to delegate the restaurant search. Different plan, same user request.

The persona doesn't change the planning *engine* — it changes the *inputs* to the engine.

### 3. Voice Persona

Each persona has its own voice configuration — a bundle of TTS provider, voice ID, and speaking parameters. When the persona switches, the voice changes.

**Why this matters for the product:** A Chief of Staff sounds professional and measured. A Personal Assistant sounds warm and conversational. A Sales Assistant sounds energetic and confident. The voice is part of the persona's identity.

**At baseline:** The voice config is a static mapping per persona. The voice pipeline reads the active persona's `voice` config and uses it for all TTS in that session. No dynamic voice switching mid-conversation.

### 4. Skill Constraints Per Persona

The M2 skill retrieval API (`find_skills`) accepts a persona filter. When the planner calls `find_skills("schedule a meeting")`, the retrieval filters by the persona's allowed skill namespaces before returning results.

**How this works:**
- Persona config has `skills.allowed: ["calendar.*", "email.*"]`
- `find_skills` adds a filter: `WHERE namespace IN ('calendar', 'email')`
- The planner never sees research or document skills — it can't hallucinate a plan using them

This is a simple namespace filter on the existing retrieval query. No new infrastructure needed.

### 5. Persona Memory Separation

Each persona has its own memory namespace. The Chief of Staff's episodic memory (past scheduling decisions, meeting preps) is separate from the Research Assistant's (past searches, report drafts).

**How it works:**
- Episodic memory (M7 Layer 3) queries include `WHERE namespace = persona.memory.namespace`
- Semantic memory (M7 Layer 4, mem0) searches are scoped to the persona's namespace
- The user's global profile (name, preferences, contacts) is shared across all personas (`shared_profile_access: true`)

**Why separate:** If the EA persona remembers "user prefers morning meetings," that shouldn't pollute the Research Assistant's context with irrelevant scheduling data. Each persona builds its own specialized knowledge.

### 6. Persona Selection

**How the system picks a persona:**

| Method | How | When |
|--------|-----|------|
| **Explicit switch** | User says "switch to research mode" or taps persona in UI | User wants a different agent behavior |
| **Default persona** | Configured per user in settings | Most interactions use this |
| **Auto-detection (refinement)** | GID classifies intent → maps to best persona | Future — requires confidence in classification |

**At baseline:** Default persona + explicit switching. Auto-detection is refinement because misclassifying the persona is worse than using a generic one.

---

## What Counts as Refinement

| Refinement | Why deferred |
|------------|-------------|
| **Auto-persona selection** (GID detects intent → picks persona) | Misclassification is worse than using default. Need high confidence. |
| **Persona-specific prompt evolution** (each persona's prompt improves independently) | Needs feedback data per persona first. |
| **Per-persona autonomy levels** (EA auto-approves scheduling, Support always asks) | Fixed approval thresholds per persona work at baseline. User-tunable is refinement. |
| **Persona handoff** (mid-conversation: "actually, look this up for me" → switch to Research) | Explicit switching works. Seamless handoff needs context passing logic. |
| **Custom user-created personas** | 5 pre-built personas cover the launch. User-created is post-beta. |
| **Persona marketplace** (share/install personas) | Way future. |
| **Overlay pattern** (model-level persona injection via token probability modification) | Requires fine-tuning infrastructure. System prompt approach works. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| Skill retrieval with namespace filter | Skill Management (M2, AKN) | Persona constrains available skills |
| System prompt injection in planner | Task Decomposition (M3, RKP) | Persona prompt fed to planner |
| Voice config consumption | Voice Intelligence (RKP) | TTS pipeline reads persona's voice settings |
| Memory namespace scoping | Memory (M7, KPR) | Episodic + semantic memory partitioned per persona |
| Persona selector in chat UI | Frontend (M9, KPR) | User switches personas |

## Effort Estimate

| Component | Effort |
|-----------|--------|
| Persona config schema + storage (5 initial persona configs) | 2 days |
| Planner integration (inject persona system prompt + skill filter) | 2 days |
| Voice pipeline integration (read persona voice config) | 1 day |
| Memory namespace scoping (filter queries by persona) | 1-2 days |
| Persona switching (explicit, via chat command or UI) | 1 day |
| Testing across all 5 personas | 2 days |

**Total: ~9-12 dev days to reach baseline.**

---

## Key Concepts Explained

**Persona vs system prompt — what's the difference?**
A system prompt is one piece of a persona. A persona also includes: which skills the agent can use, what voice it speaks in, what memory it accesses, how aggressive it is with approvals, and how long its responses are. Changing the system prompt changes personality. Changing the persona changes capability.

**Why not just one "smart" agent that adapts?**
A general-purpose agent that tries to be everything is mediocre at everything. A Chief of Staff that "also does research" will plan research tasks with scheduling-oriented prompts and the wrong skill access. Personas let each configuration be *specialized* — better prompts, better skill access, better voice — without building separate agents. It's one engine, multiple configurations.

**Why memory separation?**
If the Sales Assistant persona learns "user closes deals better with case studies," that insight is valuable for sales but useless (and confusing) for the Personal Assistant planning a dinner reservation. Namespace separation keeps each persona's learned knowledge relevant. The shared user profile (name, contacts, preferences) is still accessible to all.
