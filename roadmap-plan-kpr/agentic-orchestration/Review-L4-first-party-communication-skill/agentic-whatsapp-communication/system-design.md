# WhatsApp Communication Intelligence — System Design

> Owner: KPR
> Date: April 2026
> Related: `whatsapp-intelligence-phases.md` (WCI 4-layer implementation plan)

---

## Part 1: Current Architecture

### How WhatsApp Communication Works Today

```
┌────────────────────────────────────────────────────────────────────────┐
│                      PROVIDER LAYER                                    │
│                                                                        │
│  WHATSAPP_PROVIDER routes by config:                                   │
│    "meta"     → WhatsAppMetaClient (Meta Graph API)                   │
│    "pinnacle" → WhatsAppBSPClient (Pinnacle/Pinbot BSP API)           │
│                                                                        │
│  Both support: send_text(), send_template()                            │
│  24h session window: template required to open new session             │
└──────────────────────┬─────────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  WHATSAPP SERVICE (whatsapp_service.py ~4600 lines)    │
│                                                                        │
│  OUTBOUND:                                                             │
│   send_outbound_text() → sanitize → LLM refine (optional)             │
│   → 24h session check → template opener if stale                      │
│   → send text with retries & fallbacks                                │
│   → deferred text dispatch (waits for template delivery webhook)      │
│                                                                        │
│  INBOUND (webhook-driven):                                             │
│   process_webhook() → _handle_inbound_messages()                      │
│   → resolve parent message (context linking)                          │
│   → resolve topic_id (link to bot_task)                               │
│   → insert message → trigger auto-reply if active task                │
│                                                                        │
│  AUTO-REPLY:                                                           │
│   _maybe_auto_reply() → debounce (8s) → max turns (2)                │
│   → closure detection → build transcript (last 50 msgs)              │
│   → LLM: {"reply": str, "done": bool}                                │
│   → send reply → if done: complete task + summarize                   │
│                                                                        │
│  INTELLIGENCE LEVEL: SHALLOW                                           │
│   - No cross-channel awareness (doesn't know about prior calls)       │
│   - Binary task completion (done: true/false)                         │
│   - No contact behavioral patterns                                    │
│   - No sentiment/urgency detection                                    │
│   - Dumb SLA escalation (timeout → new task, no context carry)        │
│   - No proactive owner notifications during conversation              │
└────────────────────────────────────────────────────────────────────────┘
```

### Current Use Cases

**Outbound scenarios:**
1. Task-driven outbound — agentic chat → bot_task → send WhatsApp
2. Direct outbound — user sends via app/API
3. Template opener — 24h session expired → send template first → then free text
4. Deferred text dispatch — Pinnacle: wait for template delivery webhook → then send text
5. Batch WhatsApp — multiple contacts via batch_id

**Inbound scenarios:**
6. Inbound reply to active task — linked via parent_message_id or reference token
7. Inbound new message (no active task) — clarification flow
8. Inbound with multiple active tasks — ask "which message are you responding to?"
9. Inbound reaction (emoji) — converted to synthetic text, triggers auto-reply
10. Inbound interactive button/list reply — payload ID as body

**Auto-reply scenarios:**
11. Auto-reply to active task — LLM generates follow-up question or closure
12. Auto-reply max turns reached — "I need owner's support to proceed further"
13. Closure detection — short message matching closure tokens (thanks, ok, done)
14. Information retrieval trigger — keyword detection (schedule, calendar, otp) → consent flow

**Task lifecycle:**
15. Task completion — auto-reply detects done → summarize → update bot_task
16. Task finisher heuristic — binary COMPLETE/CONTINUE string check
17. SLA timeout escalation — hard timeout → new task on different channel (no context)

**Status & delivery:**
18. Message status updates — sent → delivered → read → failed (webhook)
19. Send failure fallback — template fallback on SESSION_REQUIRED error
20. Deferred text on delivery — Pinnacle: free text waits for opener delivery confirmation

**Voice modes:**
21. TryBo first-person — "I'm checking your availability..."
22. Legacy third-person — "John has asked me to check your availability..."

---

## Part 2: Proposed Architecture

### Intelligence upgrade across 5 phases — everything else stays

The provider layer, message delivery, template handling, webhook processing, and data model remain unchanged. The intelligence upgrade happens in how the system **understands context**, **detects outcomes**, **adapts to contacts**, **escalates smartly**, and **alerts the owner proactively**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                      PROVIDER LAYER (unchanged)                        │
│                                                                        │
│  Meta Graph API / Pinnacle BSP API                                     │
│  Template handling, 24h session window, delivery webhooks              │
└──────────────────────┬─────────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────────┐
│               WHATSAPP SERVICE (enhanced)                              │
│                                                                        │
│  Outbound + Inbound + Status handling: UNCHANGED                       │
│                                                                        │
│  Auto-reply: ENHANCED with intelligence layers ↓                       │
└──────────────────────┬─────────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────────┐
│            WHATSAPP COMMUNICATION INTELLIGENCE                         │
│            (5 phases — shared services with Voice Calling)             │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ P0: CROSS-CHANNEL CONTEXT                                       │  │
│  │                                                                  │  │
│  │  contact_interaction_service.py (shared with calling)            │  │
│  │  → "As discussed on the call yesterday..."                       │  │
│  │  → WhatsApp auto-reply knows about prior calls                  │  │
│  │  → Outbound messages reference prior interactions               │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ P1: CONTACT INTELLIGENCE                                        │  │
│  │                                                                  │  │
│  │  contact_intelligence_service.py (shared with calling)           │  │
│  │  → Response time patterns, completion rates, best times          │  │
│  │  → Dynamic SLA deadlines based on contact behavior              │  │
│  │  → Planner uses WhatsApp intelligence for channel selection     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ P2: SEMANTIC OUTCOME DETECTION                                   │  │
│  │                                                                  │  │
│  │  outcome_detection_service.py (shared with calling)              │  │
│  │  → Replaces binary done/not-done with:                          │  │
│  │    completed | partially_completed | blocked | deferred | failed │  │
│  │  → Partial progress tracking (what_achieved, what_remains)      │  │
│  │  → Blocker detection → owner notification                       │  │
│  │  → Deferral detection → scheduled retry                         │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ P3: INTELLIGENT ESCALATION                                       │  │
│  │                                                                  │  │
│  │  smart_escalation_service.py (shared with calling)               │  │
│  │  → Contact-aware: use call if high call answer rate              │  │
│  │  → Time-aware: wait until responsive hours, don't escalate blind│  │
│  │  → Context-carrying: WhatsApp→Call carries full conversation     │  │
│  │  → Retry-aware: retry WhatsApp with progress context            │  │
│  │  → Channel recommend skill for planner                          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ P4: REAL-TIME INTELLIGENCE                                       │  │
│  │                                                                  │  │
│  │  sentiment_service.py (shared with calling)                      │  │
│  │  proactive_notification_service.py (shared with calling)         │  │
│  │  → Urgency detection: "URGENT: need this TODAY" → owner push    │  │
│  │  → Frustration detection → adjust auto-reply tone + alert owner │  │
│  │  → Key info detection → "Good news: contact confirmed X"        │  │
│  │  → Stuck detection → "Task seems stuck after 3 turns"           │  │
│  │  → Dynamic SLA tightening on urgency                            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Every Use Case in the Proposed Architecture

### Outbound Scenarios

#### UC-1: Task-Driven Outbound with Cross-Channel Context

```
TODAY:
  Agentic chat: "WhatsApp Neha about vendor pricing"
  → Bot task created → send_outbound_text()
  → "Reaching out on behalf of Krishna: Could you share the vendor pricing?"
  → No awareness of yesterday's call with Neha about the same topic

PROPOSED (P0):
  Same trigger
  → contact_interaction_service.get_contact_interaction_summary()
  → Finds: "Called Neha yesterday. She confirmed ABC Corp ₹4.2L 
            and Delta ₹3.8L. 4 vendors still pending from PO system."
  → Outbound message: "Following up on our call yesterday — 
     you mentioned 4 vendors were still pending from the PO system.
     Were you able to pull those?"
  → Contact doesn't repeat herself. Conversation continues where it left off.
```

---

#### UC-2: Task-Driven Outbound with Contact Intelligence

```
TODAY:
  WhatsApp task created for Neha at 11pm
  → Sent immediately → Neha doesn't respond (she's asleep)
  → SLA timeout at 2am → escalated to call at 2am → no answer

PROPOSED (P1):
  contact_intelligence_service.get_contact_profile(neha)
  → avg_response_time: 45 min
  → best_messaging_times: 9am-6pm IST
  → whatsapp_completion_rate: 85%
  
  Planner sees: "It's 11pm, Neha responds 9am-6pm IST"
  → Schedules WhatsApp for 9am tomorrow
  → SLA deadline: 9am + 10x avg (45m × 10 = ~8h) = 5pm
  → No midnight escalation. Contact gets message at her responsive time.
```

---

#### UC-3: Template Opener + Deferred Text

No change to template/session handling — this is provider-layer plumbing, not intelligence.

---

### Inbound Scenarios

#### UC-4: Inbound Reply to Active Task — With Semantic Outcome

```
TODAY:
  Neha replies: "I'll check and get back to you tomorrow"
  → Auto-reply LLM: {"reply": "Thanks!", "done": false}
  → Task stays in-progress. No tracking of the deferral.
  → SLA timeout eventually escalates to call.

PROPOSED (P2):
  Neha replies: "I'll check and get back to you tomorrow"
  → outcome_detection_service.detect_outcome()
  → status: "deferred"
  → deferred_until: "tomorrow"
  → what_remains: "4 vendor pricing from PO system"
  
  → Auto-reply: "No problem! I'll check back with you tomorrow."
  → Task metadata updated: deferred_until = tomorrow
  → Retry scheduled for tomorrow morning (not SLA timeout)
  → No escalation — contact said they'd respond, we respect that.
```

---

#### UC-5: Inbound with Partial Completion

```
TODAY:
  Task: "Get vendor pricing for 6 vendors"
  Neha replies: "ABC Corp is ₹4.2L and Delta is ₹3.8L"
  → Auto-reply LLM: {"done": false} → asks about all 6 again

PROPOSED (P2):
  outcome_detection_service.detect_outcome()
  → status: "partially_completed"
  → completion_pct: 33
  → what_was_achieved: "ABC Corp ₹4.2L, Delta ₹3.8L"
  → what_remains: "4 vendors still pending"
  
  → Auto-reply: "Got it, thanks! ABC Corp ₹4.2L and Delta ₹3.8L noted.
     Could you also share pricing for the remaining 4 vendors?"
  → Only asks for what's missing, not all 6 again.
```

---

#### UC-6: Inbound with Blocker

```
TODAY:
  Neha: "I need to check with the PO system but it's down right now"
  → Auto-reply: "Could you share the vendor pricing?" (repeats)
  → Owner doesn't know there's a blocker.

PROPOSED (P2 + P4):
  outcome_detection_service: status = "blocked"
  → blocker: "PO system is down"
  
  → Auto-reply: "I understand. No rush — let me know once the PO system
     is back up and you can pull the numbers."
  → Owner push notification: "Neha says PO system is down.
     Vendor pricing task is blocked."
  → Task metadata: blocked, blocker = "PO system down"
  → No SLA escalation until blocker clears.
```

---

#### UC-7: Inbound Urgent Message

```
TODAY:
  Neha: "URGENT: The board meeting is in 2 hours, I need Krishna's approval NOW"
  → Auto-reply treats it same as any message.
  → Owner doesn't know until they manually check.

PROPOSED (P4):
  sentiment_service.analyze_turn()
  → urgency: "critical"
  
  → Immediate FCM push to owner: "URGENT from Neha: Board meeting in 2 hours,
     needs your approval immediately."
  → Auto-reply: "I understand this is urgent. I'm flagging this for Krishna 
     right away."
  → SLA dynamically tightened: deadline moved to 30 min from now.
```

---

#### UC-8: Inbound from Known Contact — Cross-Channel Awareness

```
TODAY:
  Called Neha yesterday about the report.
  Neha WhatsApps today: "like I told you on the call"
  → Auto-reply has no call context → can't engage.

PROPOSED (P0):
  contact_interaction_service loaded into auto-reply context
  → Knows about yesterday's call: "Discussed school visit report.
     Neha said mostly done, needs photos."
  
  → Auto-reply: "Right, you mentioned on the call that the photos
     were still pending. Were you able to add those?"
  → Cross-channel continuity. Contact doesn't repeat herself.
```

---

### Auto-Reply Scenarios

#### UC-9: Auto-Reply with Frustration Detection

```
TODAY:
  Turn 1: Neha: "I already told you this"
  Turn 2: Neha: "Why do I keep getting asked the same thing?"
  → Auto-reply: generic follow-up question. No tone change.

PROPOSED (P4):
  sentiment_service: frustration = HIGH (2 consecutive frustrated messages)
  
  → Auto-reply tone shifts: "I apologize for the repeated follow-up.
     I've noted everything you've shared. I'll inform Krishna directly."
  → Owner push: "Neha seems frustrated in your WhatsApp conversation.
     Consider reaching out personally."
  → Task may auto-complete to prevent further irritation.
```

---

#### UC-10: Auto-Reply Max Turns with Intelligence

```
TODAY:
  After 2 turns: "I need Krishna's support to proceed further."
  → Blunt. No context about what was accomplished.

PROPOSED (P2):
  outcome_detection detects partial completion after 2 turns
  → what_achieved: "Confirmed availability for Tuesday afternoon"
  → what_remains: "Exact time not confirmed"
  
  → Final message: "Thanks Neha! I've noted you're available Tuesday
     afternoon. I'll have Krishna confirm the exact time with you."
  → Task completed with: completion_pct=80, what_remains="exact time"
  → Owner sees rich summary, not just "done/not done."
```

---

### Escalation Scenarios

#### UC-11: WhatsApp → Call Escalation with Context

```
TODAY:
  WhatsApp task fails (no response) → SLA timeout
  → New call task created with original prompt only
  → Agent on call: "Hi, calling about vendor pricing"
  → Contact: "I already replied to your WhatsApp!"
  → Agent has no WhatsApp context → awkward.

PROPOSED (P3):
  smart_escalation_service.decide_escalation()
  → Contact has high call answer rate (90%)
  → Decision: escalate to call
  → Carry-forward context: "WhatsApp conversation summary:
     ABC Corp ₹4.2L and Delta ₹3.8L confirmed.
     4 vendors still pending from PO system."
  
  → Call agent intro: "Hi Neha, following up on our WhatsApp conversation.
     You shared ABC Corp and Delta pricing — thanks for that.
     Were you able to pull the remaining 4 from the PO system?"
  → Contact doesn't repeat herself. Seamless channel transition.
```

---

#### UC-12: Smart Retry Instead of Escalation

```
TODAY:
  WhatsApp no response → SLA timeout → escalate to call immediately

PROPOSED (P3 + P1):
  smart_escalation_service evaluates:
  → Contact profile: responds to WhatsApp 90% of the time, avg 2h
  → Current time: 10am (within responsive hours)
  → Time since send: 1h (contact avg is 2h)
  → Decision: "wait_longer" — don't escalate yet
  
  After 4h (2x avg):
  → Decision: retry WhatsApp with follow-up message
  → "Hi Neha, just checking in — were you able to look into the 
     vendor pricing? Let me know if you need anything from our end."
  
  After retry + 4h no response:
  → Decision: escalate to call with full context
```

---

#### UC-13: Deferred Task — No Escalation

```
TODAY:
  Neha: "I'll send it tomorrow"
  → Task stays in-progress → SLA timeout → escalates anyway

PROPOSED (P2 + P3):
  outcome_detection: status = "deferred", deferred_until = "tomorrow"
  → smart_escalation: "deferred — schedule retry for tomorrow, don't escalate"
  → Retry message tomorrow: "Hi Neha, just following up — you mentioned
     you'd send the vendor pricing today. Were you able to?"
  → Respectful. No premature escalation.
```

---

### Post-Conversation Scenarios

#### UC-14: Rich Task Summary

```
TODAY:
  Task summary: "Discussed vendor pricing. Task completed."

PROPOSED (P2):
  outcome_detection provides structured data:
  → completion_pct: 100
  → what_was_achieved: "All 6 vendor prices collected:
     ABC Corp ₹4.2L, Delta ₹3.8L, Sigma ₹5.1L, ..."
  → blockers: none
  
  → Rich summary in bot_task: structured outcome + raw transcript
  → Owner sees exactly what was collected, not just "completed."
```

---

#### UC-15: Proactive Owner Notification on Key Info

```
TODAY:
  Neha confirms: "The school visit report is ready, I'll email it."
  → Owner doesn't know until they check the app.

PROPOSED (P4):
  proactive_notification_service detects key info:
  → "Contact provided answer to the objective"
  → Push to owner: "Good news: Neha confirmed the school visit report 
     is ready and will email it."
  → Owner knows instantly without checking.
```

---

## Part 4: What Changes, What Stays

### Changes (by phase)

| Phase | What Changes | Where |
|---|---|---|
| **P0** | Cross-channel context injected into auto-reply + outbound | `whatsapp_service.py`, `whatsapp_prompts.py`, `task_initiation_service.py` |
| **P1** | Contact behavioral patterns recorded + used for SLA + planner | `whatsapp_service.py`, `bot_task_service.py`, `llm_planner.py` |
| **P2** | Binary done/not-done → structured outcome detection | `whatsapp_service.py`, `whatsapp_prompts.py`, `bot_task_service.py` |
| **P3** | Dumb SLA escalation → smart context-carrying escalation | `sla_checker.py`, `whatsapp_service.py`, `seed_registry.py` |
| **P4** | No sentiment awareness → urgency detection + owner alerts | `whatsapp_service.py`, `bot_task_service.py` |

### Stays Unchanged

| Component | Why |
|---|---|
| `whatsapp_meta_client.py` | Provider API integration — no intelligence change |
| `whatsapp_repo.py` | Data access layer — queries stay the same |
| `whatsapp_utils.py` | Utilities, closure tokens — unchanged |
| `routes/whatsapp.py` | Webhook endpoint — unchanged |
| Template handling | 24h window, session openers — provider-layer plumbing |
| Deferred text dispatch | Pinnacle optimization — provider-layer plumbing |
| Message status handling | Delivery webhooks — provider-layer plumbing |
| Conversation/message data model | Tables unchanged — intelligence uses metadata JSONB |
| Reference token tracking | Reply linking — unchanged |
| Voice modes (TryBo/legacy) | Tone selection — unchanged |

### New Services (ALL shared with Voice Calling)

| Service | Phase | Purpose | ~Lines |
|---|---|---|---|
| `contact_interaction_service.py` | P0 | Cross-channel history | 200 |
| `contact_intelligence_service.py` | P1 | Behavioral patterns | 250 |
| `outcome_detection_service.py` | P2 | Structured outcomes | 200 |
| `smart_escalation_service.py` | P3 | Intelligent escalation | 250 |
| `sentiment_service.py` | P4 | Sentiment detection | 150 |
| `proactive_notification_service.py` | P4 | Owner alerts | 100 |

**These 6 services are built ONCE and used by both WhatsApp and Voice Calling.** The voice calling VCI layers (Phase 2 of voice plan) and WhatsApp intelligence phases share the same underlying services.

---

## Part 5: Relationship to Voice Calling Plan

```
VOICE CALLING PLAN:                    WHATSAPP PLAN:

  Phase 1: Pipecat pipeline              (no pipeline change needed —
           (infrastructure)               WhatsApp is text, not audio)
           
  Phase 2: VCI 6 Layers                  P0: Cross-channel context
           (conversation intelligence)    P1: Contact intelligence
                                          P2: Semantic outcomes
                                          P3: Smart escalation
                                          P4: Real-time intelligence

SHARED SERVICES (built once):

  contact_interaction_service  ← used by both voice L5 and WhatsApp P0
  contact_intelligence_service ← used by both voice L5 and WhatsApp P1
  outcome_detection_service    ← used by both voice L4 and WhatsApp P2
  smart_escalation_service     ← used by both voice orchestrator and WhatsApp P3
  sentiment_service            ← used by both voice L4 and WhatsApp P4
  proactive_notification_service ← used by both voice and WhatsApp P4
```

**Key difference: WhatsApp doesn't need a pipeline change.** Voice calling needed Pipecat (Phase 1) because the audio pipeline was custom and needed replacement. WhatsApp is text-based — the delivery infrastructure (Meta API, templates, webhooks) works fine. The upgrade is purely in intelligence — how the system understands, responds, and adapts.

---

## Part 6: Effort Summary

| Phase | What | Effort | Dependencies |
|---|---|---|---|
| **P0** | Cross-channel context | ~2 days | None |
| **P1** | Contact intelligence | ~3 days | P0 |
| **P2** | Semantic outcomes | ~4 days | P1 (for effort estimation) |
| **P3** | Smart escalation | ~3 days | P1 + P2 |
| **P4** | Real-time intelligence | ~2 days | Independent (parallel with P2/P3) |
| **Total** | | **~14 days** | |

**Combined with Voice Calling:** ~20-26 days total (shared services built once, not twice).

---

## Part 7: DB Schema Changes

**None.** All intelligence data stored in:
- `bot_tasks.metadata` (JSONB) — outcome history, completion_pct, blockers, urgency
- `skill_data_store` (existing) — contact behavioral patterns (namespace="contact_intel")
- `whatsapp_conversations.summary_metadata` (existing JSONB) — task summaries

No new tables. No migrations.
