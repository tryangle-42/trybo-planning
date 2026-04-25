# WCI — WhatsApp Communication Intelligence

> Owner: KPR
> Source: Codebase analysis — April 2026
> Related: `agentic-voice-calling/` (voice calling intelligence plan)

---

## What This Is

The WCI upgrades WhatsApp from "template-and-reply executor" to "intelligent agentic communicator" with 4 intelligence layers, running as a LangGraph agent that can execute skills from the main agentic orchestration's skill registry.

**WCI 4 Layers:**

| Layer | What It Does |
|---|---|
| **WCI L1: Conversation Architecture** | Phase-aware async messaging — opening, middle, closing, follow-up, retry |
| **WCI L2: Response Intelligence** | Hedge detection, structured outcomes, sentiment/urgency, skill calling decisions |
| **WCI L3: Contextual Intelligence** | Cross-channel transcripts, chat memory, contact profiles, authority rules, skill capabilities |
| **WCI L4: Recovery & Repair** | Stuck detection, smart escalation, blocker handling, owner alerts |

**WCI runs as a LangGraph agent:**

```
Inbound WhatsApp message
    ↓
LANGGRAPH WCI AGENT
    ├── WCI Node (LLM — single call per message)
    │     Reads: context, phase, skill capabilities, conversation history
    │     Outputs: { reply, tool_requests, task_outcome, phase }
    │
    ├── Tool Executor (parallel, no LLM)
    │     Executes skills from seed_registry:
    │     calendar.check, contact.lookup, web.research, consent.request
    │     Writes results to state → WCI reads on next message
    │
    └── Compaction (parallel, no LLM)
          Summarizes old messages when conversation is long
    ↓
Outbound WhatsApp reply (via whatsapp_service.send_outbound_text)
```

**This is NOT about:** 24h window enforcement, message retry, interactive messages, rich media, rate limiting, new channels.

---

## Current State

| Area | What Exists | Intelligence Level |
|---|---|---|
| Auto-reply | LLM-powered with debounce (8s), max turns (2), closure detection | Shallow |
| Task completion | Binary LLM guess: `{"reply": str, "done": bool}` | Shallow |
| Cross-channel awareness | NONE — auto-reply doesn't know about prior calls | Zero |
| Contact intelligence | Name + phone only. No response patterns, best times | Zero |
| Escalation | Hard SLA timeout → new task, no context carry-forward | Dumb |
| Sentiment/urgency | NONE — "URGENT" treated same as "ok thanks" | Zero |
| Proactive owner alerts | NONE during conversation | Zero |
| Skill execution | Keyword trigger (schedule, calendar, otp) → consent flow | Mechanical |

### Key Files

- `whatsapp_service.py` (4600L) — core orchestration
- `whatsapp_meta_client.py` (321L) — Meta/Pinnacle API client
- `whatsapp_repo.py` (246L) — data access
- `whatsapp_prompts.py` — LLM prompts
- `whatsapp_utils.py` (79L) — closure tokens, debounce
- `routes/whatsapp.py` (65L) — webhook handler
- `information_retrieval_analyzer.py` — info request detection
- `bot_task_service.py` — task lifecycle + SLA
- `sla_checker.py` — escalation
- `seed_registry.py` — skill definitions

### Current Auto-Reply Flow

```
Inbound → _handle_inbound_messages() → insert with topic_id
  → _maybe_auto_reply()
    → debounce (8s) → max turns (2) → closure detection
    → info retrieval keyword scan → consent if triggered
    → fetch thread (last 50 msgs) → build transcript
    → LLM: {"reply": str, "done": bool}
    → anti-hallucination guard
    → send reply → if done: complete task + summarize
```

---

## WCI L1: Conversation Architecture 
**What this is:** Phase-aware async messaging. Today every turn is treated the same — no concept of opening, middle, closing, follow-up, or retry.

### Phases for Async Messaging

| Phase | What Happens | Current State | What Changes |
|---|---|---|---|
| **Opening** | First outbound message — sets tone, states objective | Generic template, no cross-channel awareness | Context-aware intro referencing prior interactions |
| **Middle** | Auto-reply turns — pursue objective, collect info | Exists but binary done/not-done, no partial progress | Phase-aware prompt directives, partial progress tracking |
| **Closing** | Detect completion → summarize → complete task | Fragile closure tokens + anti-hallucination guard | Structured completion via outcome detection |
| **Follow-up** | Contact replies after task closed | "Thanks for your message, will let them know" | Intelligent: reconnect to task if relevant, or route to owner |
| **Retry** | Same channel retry after partial/deferred | Doesn't exist — only SLA timeout | Progress-aware follow-up: "You mentioned ABC Corp, could you also..." |

### Deliverables

1. **Phase tracking in WCI state** — `phase` field in LangGraph state: `opening` → `middle` → `closing` → `completed`. Phase changes the WCI Node's prompt directive per message.

2. **Opening phase: context-aware first message** — When sending first outbound, WCI L3 (Context) provides cross-channel history. Opening prompt generates: "Following up on our call yesterday — you mentioned the report needs photos. Were you able to add those?" instead of generic "Could you share the report?"

3. **Closing phase: structured completion** — Replace fragile closure token matching + anti-hallucination guard with WCI L2 (Response Intelligence) structured outcome detection. `status: completed` → summarize + complete task. `status: deferred` → schedule retry.

4. **Follow-up phase: intelligent routing** — When contact replies after task closed, check if reply is related to the closed task (new info, correction) vs new topic. Route accordingly instead of generic "will let them know."

5. **Retry phase: progress-aware follow-up** — When retrying after `partially_completed`, inject prior progress: "You shared ABC Corp ₹4.2L and Delta ₹3.8L. Could you also share the remaining 4?"

### What Changes

- `whatsapp_service.py` — `_maybe_auto_reply()` reads phase from state, selects phase-specific prompt
- `whatsapp_prompts.py` — phase-aware prompt sections (opening/middle/closing/retry directives)
- `task_initiation_service.py` — opening phase uses cross-channel context for first message

---

## WCI L2: Response Intelligence 
**What this is:** Replaces binary `done: true/false` with structured outcome detection, adds sentiment/urgency awareness, and enables skill execution from the agent orchestration's skill registry.

### Capabilities

- **Hedge vs commitment detection** — "I'll try" ≠ "I will send by 5pm"
- **Structured outcome model** — committed / partially_completed / blocked / deferred / refused
- **Partial progress tracking** — what_achieved, what_remains, completion_pct
- **Sentiment/urgency detection** — lightweight LLM classification per message
- **Tool/skill calling decisions** — when to execute skills from registry

### What's Removed (voice-specific)

- Speech directives (rate/energy/pause) — text has no audio delivery
- Filler generation — async messaging, no silence to fill
- Real-time emotional tone adaptation — text responses don't need TTS parameter mapping

### What's Customized (WhatsApp-specific)

- **Response length** — WhatsApp: 20-40 words (per existing prompts). Voice: flexible.
- **Closure detection** — WhatsApp-specific: WHATSAPP_CLOSURE_TOKENS (thanks, ok, done, 🙏, 👍)
- **Emoji interpretation** — 👍/✅/💯 = yes, 👎/❌ = no (existing anti-hallucination guard, formalized)
- **Tool execution timing** — Fast tool (<2s): wait and include in same reply. Slow tool (>2s): send "Let me check" first, follow up with result.

### Deliverables

**1. Structured Outcome Detection** (shared service)
New file: `outcome_detection_service.py` (~200L)

```
TaskOutcome:
  status: completed | partially_completed | blocked | deferred | failed | no_response
  completion_pct: 0-100
  what_was_achieved: str
  what_remains: str
  blockers: List[str]
  next_action: retry_same_channel | escalate | wait | owner_action_needed
```

**2. Replace binary done check in auto-reply**
Modify `whatsapp_service.py`:

Current: `LLM → {"reply": str, "done": bool}` → if done: close task
New: `LLM → {"reply": str}` + `outcome_detection_service.detect_outcome()` → structured handling:
- `completed` → close task (existing path)
- `partially_completed` → continue, record progress, ask only for remaining items
- `blocked` → notify owner: "Contact says PO system is down"
- `deferred` → record, schedule retry: "Contact said they'll respond tomorrow"

**3. Sentiment & Urgency Detection** (shared service)
New file: `sentiment_service.py` (~150L)

Integrated into `_maybe_auto_reply()` — runs in parallel with reply generation:
- `urgency == "critical"` → immediate owner push + auto-reply acknowledges urgency
- `sentiment == "frustrated"` → adjust tone + owner notification
- Contact provides key answer → "Good news" push to owner

**4. Skill execution from WCI Node**
WCI Node can request tools via `tool_requests` in structured output. Tool Executor picks up from LangGraph state, executes skill from `seed_registry`, writes result back. WCI Node reads result on next message.

**5. Effort estimation**
Before WhatsApp dispatch, estimate effort from contact profile (WCI L3):
- Contact responds in 3m, 90% completion → "low, ~10m"
- Contact responds in 2h, 50% completion → "high, ~6h"
- Stored on `bot_tasks.metadata.effort_estimate`

### What Changes

- `whatsapp_service.py` — replace `_generate_auto_reply_with_llm()` binary check, replace `_maybe_llm_complete_task()` string parsing, add sentiment analysis
- `whatsapp_prompts.py` — partial completion awareness, blocker awareness
- `bot_task_service.py` — progress tracking in metadata, urgency adjustment
- `task_initiation_service.py` — effort estimation

### Verification

- Task with 3 questions → contact answers 1 → verify `completion_pct=33`, auto-reply asks only remaining 2
- "I'll check and get back tomorrow" → verify `deferred`, retry scheduled
- "I need to ask my manager" → verify `blocked`, owner push notification
- "URGENT: need this TODAY" → verify owner immediate push, SLA tightened

---

## WCI L3: Contextual Intelligence 
**What this is:** Everything the WCI agent knows — cross-channel history, contact behavioral patterns, authority rules, and skill capabilities. Assembled from existing database tables before each LLM call.

### Deliverables

**1. Contact Interaction History Service** (shared with voice calling)
New file: `contact_interaction_service.py` (~200L)

Queries EXISTING tables:
- `bot_tasks` — task history with status, prompt, channel_type
- `transcripts` (via calls) — call summaries
- `whatsapp_conversations` — WhatsApp thread summaries

Returns: total interactions, last channel, recent summaries, open tasks, pre-formatted LLM context.

**2. Contact Intelligence Service** (shared with voice calling)
New file: `contact_intelligence_service.py` (~250L)

Uses existing `skill_data_store` with `namespace="contact_intel"`. Per-interaction metrics:
- `response_time_seconds`, `turns_to_completion`, `time_of_day_utc`, `outcome`

Aggregated profile:
- `avg_response_time_s.whatsapp`, `whatsapp_completion_rate`, `best_messaging_times`, `avg_turns_to_completion`

**3. Inject cross-channel context into auto-reply and outbound**
- `_maybe_auto_reply()` — call `get_contact_interaction_summary()`, inject into LLM prompt
- `process_whatsapp_outbound()` — fetch context, pass to opening message
- Prompts get `{prior_interactions}` section

**4. Dynamic SLA deadline from contact patterns**
- Contact avg response 3m → deadline 30m (10x buffer)
- Contact avg response 2h → deadline 8h
- No history → global default

**5. Surface WhatsApp intelligence to planner**
- `llm_planner.py` `_build_contacts_catalog()` — append: "Intel: WhatsApp response 3m avg, 90% completion. Best: 9am-6pm IST."

**6. Tool capabilities and authority rules**
WCI Node knows what skills it can execute and what needs owner consent — injected into prompt alongside authority rules.

### What Changes

- `whatsapp_service.py` — context injection in auto-reply, outcome recording in `_handle_auto_reply_completion()`
- `whatsapp_prompts.py` — `{prior_interactions}` section
- `task_initiation_service.py` — context fetch, smart SLA defaults
- `bot_task_service.py` — outcome recording on terminal status
- `llm_planner.py` — contact intel in planner catalog

### Verification

- Call contact X, complete with summary → WhatsApp to X references the call
- Exchange 5+ conversations → verify avg response time, completion rate, best times
- New task without channel → verify planner prefers WhatsApp for high-WhatsApp contacts
- Verify SLA deadline reflects contact's response time

---

## WCI L4: Recovery & Repair 
**What this is:** Smart handling when conversations get stuck, fail, or need escalation — stuck detection, blocker escalation, context-carrying channel transitions, and proactive owner alerts.

### Deliverables

**1. Smart Escalation Service** (shared with voice calling)
New file: `smart_escalation_service.py` (~250L)

WhatsApp-specific escalation logic:
- `no_response` + high call answer rate → escalate to call WITH WhatsApp context summary
- `no_response` + sent outside responsive hours → `wait_longer` until best time, retry WhatsApp
- `partially_completed` → retry WhatsApp with progress: "You mentioned {achieved}. Could you also confirm {remaining}?"
- `deferred` with `deferred_until` → schedule retry at that time, don't escalate
- `blocked` → notify owner, don't escalate (contact can't proceed without owner action)
- Max retries exceeded → escalate to call with full conversation summary

**2. Context-carrying WhatsApp→Call escalation**
Modify `sla_checker.py`:
- Build `carry_forward_context`: WhatsApp thread summary + what_achieved + what_remains + blocker
- Pass as enriched `task_text` to call: "Following up on WhatsApp: {summary}. Remaining: {what_remains}."

**3. Context-carrying WhatsApp→WhatsApp retry**
- Don't re-send original message on retry
- Send follow-up acknowledging progress: "Hi {name}, following up — you mentioned {achieved}. Could you let me know about {remaining}?"

**4. Proactive Owner Notifications** (shared with voice calling)
New file: `proactive_notification_service.py` (~100L)

WhatsApp triggers:
- Urgency detected → immediate push
- Frustrated 2+ messages → push + conversation preview
- Key info provided → "Good news: {contact} confirmed {info}"
- Stuck 3+ turns → "Your WhatsApp task seems stuck"
- Contact asks for owner → immediate push

**5. Channel recommendation skill**
Register `contact.channel.recommend` in `seed_registry.py` — recommends WhatsApp vs call based on contact profile, time of day, urgency, prior failures.

### What Changes

- `sla_checker.py` — smart escalation with context carry-forward
- `whatsapp_service.py` — retry with progress context
- `seed_registry.py` — channel recommend skill

### Verification

- No response + high call answer rate → verify call escalation with WhatsApp context
- Sent outside responsive hours → verify `wait_longer` until best time
- `partially_completed` → verify retry acknowledges prior answers
- `deferred` → verify retry scheduled at `deferred_until`, not immediate escalation
- Stuck 3+ turns → verify owner push notification

---

## LangGraph Architecture

```
CURRENT:
  Inbound message → _maybe_auto_reply() → LLM → {"reply", "done"} → send

PROPOSED:
  Inbound message → LangGraph WCI Agent
    │
    ├── WCI L3 (Context): load cross-channel, contact profile, open tasks, skills
    │
    ├── WCI L1 (Architecture): determine phase (opening/middle/closing/retry)
    │
    ├── WCI Node (LLM — single call):
    │     System prompt includes: phase directive + context + skill awareness
    │     + WCI L2 (Response): hedge detection, outcomes, sentiment
    │     + WCI L4 (Recovery): stuck detection, abort conditions
    │     Output: { reply, tool_requests, task_outcome, phase }
    │
    ├── Tool Executor (parallel, no LLM):
    │     Executes skills from seed_registry
    │     Fast (<2s): result included in same reply
    │     Slow (>2s): "Let me check" → follow-up with result
    │
    └── Compaction (if conversation spans many messages):
          Summarize older messages, keep recent verbatim
    │
    ▼
  send_outbound_text() → Meta/Pinnacle API → contact receives reply
```

**WCI State:**

```python
class WCIState(TypedDict):
    messages: list              # conversation history (compacted + recent)
    phase: str                  # opening / middle / closing / completed / retry
    static_context: dict        # cross_channel, contact_profile, authority_rules, skill_capabilities
    running_tools: list         # currently executing in background
    completed_tools: list       # results ready
    task_outcome: dict          # status, what_achieved, what_remains, completion_pct
```

---

## Implementation Order

| Step | What | WCI Layer | Effort | Dependencies |
|---|---|---|---|---|
| 1 | Cross-channel context + contact profiles | WCI L3 | ~3 days | None |
| 2 | Structured outcome detection + sentiment | WCI L2 | ~4 days | Step 1 |
| 3 | Phase-aware messaging + retry logic | WCI L1 | ~2 days | Step 2 |
| 4 | Smart escalation + owner notifications | WCI L4 | ~3 days | Steps 1+2 |
| 5 | LangGraph agent + skill execution | All | ~2 days | Steps 1-4 |
| | **Total** | | **~14 days** | |

---

## New Files (shared across channels)

| File | WCI Layer | Purpose | ~Lines |
|---|---|---|---|
| `contact_interaction_service.py` | WCI L3 | Cross-channel history | 200 |
| `contact_intelligence_service.py` | WCI L3 | Behavioral patterns | 250 |
| `outcome_detection_service.py` | WCI L2 | Structured outcomes | 200 |
| `smart_escalation_service.py` | WCI L4 | Intelligent escalation | 250 |
| `sentiment_service.py` | WCI L2 | Sentiment detection | 150 |
| `proactive_notification_service.py` | WCI L4 | Owner alerts | 100 |

## Modified Files

| File | What Changes |
|---|---|
| `whatsapp_service.py` | Context injection, outcome detection, sentiment, phase-aware replies, retry context |
| `whatsapp_prompts.py` | Phase directives, prior interactions section, partial completion awareness |
| `task_initiation_service.py` | Context fetch, smart SLA, effort estimation |
| `bot_task_service.py` | Outcome recording, progress tracking, urgency adjustment |
| `sla_checker.py` | Smart escalation with context carry-forward |
| `llm_planner.py` | Contact WhatsApp intel in planner catalog |
| `seed_registry.py` | Channel recommend skill |

## DB Changes

**None.** Uses existing `bot_tasks.metadata` (JSONB), `skill_data_store`, `whatsapp_conversations.summary_metadata`.

---

## Relationship to Voice Calling Intelligence

Both WCI and voice calling intelligence need similar capabilities — context assembly, outcome detection, sentiment analysis, smart escalation. The 6 new service files are **shared across channels** (built once, used by both):

```
SHARED SERVICES (channel-agnostic):
  contact_interaction_service     — cross-channel history
  contact_intelligence_service    — behavioral patterns
  outcome_detection_service       — structured outcomes
  sentiment_service               — urgency/frustration detection
  smart_escalation_service        — intelligent channel transitions
  proactive_notification_service  — owner alerts

WHATSAPP-SPECIFIC:
  WCI L1 async phase management (opening/middle/closing/retry)
  Closure detection (emoji tokens: 👍 = yes, 🙏 = thanks)
  Response length (20-40 words)
  Template/session window handling (provider layer, not intelligence)

VOICE-SPECIFIC:
  Voice & Timing layer — audio delivery, TTS parameter mapping
  Real-time phase tracking (opening → middle → closing)
  Speech directives, filler generation, barge-in handling

Combined effort (both channels): ~20-26 days (shared services built once)
```
