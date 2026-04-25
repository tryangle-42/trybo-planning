# VCI System Design — Current Architecture to Proposed Architecture

> Owner: KPR
> Date: April 2026
> Related: `phase-1-pipecat-voice-ai-pipeline.md`, `phase-2-vci-5-layers.md`, `vci-platform-decision.md`

> **Layer numbering convention:** Layers L1–L4 are numbered in the **chronological order they run during a turn**. L1 (Context) runs first; L4 (Response Intelligence) is the LLM call. Audio delivery (Pipecat + TTS provider) is infrastructure, not a numbered layer. See `phase-2-vci-5-layers.md` for the full rationale.

---

## Part 1: Current Architecture

### How Voice Calling Works Today

Three transport paths, one custom pipeline, multiple call scenarios:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        TELEPHONY LAYER                                 │
│                                                                        │
│  telephony_factory.py routes by destination:                           │
│    +91 (India)   → KnowlarityTelephonyProvider (fallback: Twilio)     │
│    International → TwilioTelephonyProvider                             │
│                                                                        │
│  Inbound calls → Twilio webhook /incoming-voice                        │
│  LiveKit WebRTC → /livekit/session endpoint                            │
└──────────┬──────────────────┬──────────────────┬──────────────────────┘
           │                  │                  │
     Twilio WebSocket   Knowlarity WebSocket   LiveKit RTC
     (µ-law 8kHz JSON)  (PCM 16kHz binary)     (PCM 48kHz tracks)
           │                  │                  │
           └──────────────────┼──────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  CUSTOM VOICE AI PIPELINE                              │
│                  (~3830 lines across 4 services)                       │
│                                                                        │
│  agent_stream_service.py (1130L)     — WebSocket handling, ASR mgmt,  │
│                                        turn management, barge-in,      │
│                                        provider routing                │
│  media_stream_service.py (700L)      — Turn orchestration:             │
│                                        ASR → LLM → TTS → send audio   │
│  knowlarity_stream_service.py (900L) — Knowlarity-specific WebSocket  │
│                                        handling, PCM↔µ-law conversion  │
│  livekit_stream_service.py (900L)    — LiveKit room connection,        │
│                                        48kHz↔8kHz resampling           │
│                                                                        │
│  VAD: Custom RMS-based energy detection                                │
│  Turn detection: Silence-based (1000-2500ms quiet gaps)                │
│  ASR: Cartesia or ElevenLabs Scribe (streaming)                        │
│  LLM: Conversation service (sequential, not streaming)                 │
│  TTS: ElevenLabs / Cartesia / CosyVoice (voice cloned)                │
│                                                                        │
│  p90 latency: ~3.5 seconds per turn                                   │
└────────────────────────────────────────────────────────────────────────┘
```

### Current Use Cases

The system handles these scenarios today:

**Outbound calls:**
1. Task-driven outbound (from agentic chat → bot_task → call)
2. Manual outbound (user initiates from mobile app)
3. Scheduled outbound (future timestamp → APScheduler → call)
4. Batch/multi-contact outbound (batch_id → parallel calls)
5. Knowlarity fallback to Twilio (if Knowlarity API fails for +91)

**Inbound calls:**
6. Inbound from unknown caller → AI agent answers
7. Inbound from known contact → personalized greeting + AI agent
8. Inbound from another app user → direct device transfer (no AI)
9. Inbound during DND → DND message + AI agent

**During-call scenarios:**
10. Barge-in — caller interrupts agent TTS
11. Owner transfer via LiveKit — FCM push → owner joins WebRTC room
12. Owner transfer via Twilio — Dial to device client
13. Information request — agent needs owner's data (calendar, etc.) → FCM → owner approves → agent resumes
14. OTP delivery — owner enters OTP on mobile → agent reads to caller
15. Call hold — while information request is processing
16. Live transcription — per-turn persistence to transcripts table
17. Voice cloning — agent speaks in owner's cloned voice

**Post-call:**
18. Transcript summary generation
19. Bot task completion (output_raw + output_summary)
20. Call record finalization (status, duration, outcome)
21. Recording transcription (for device-transferred calls via Deepgram)

**WebRTC calls:**
22. Browser/app call via LiveKit — same AI pipeline, WebRTC transport
23. Owner transfer timeout — 35s timeout → agent resumes

**Multilingual (proposed):**
24. Language detection — STT detects caller's language
25. Auto-language response — LLM + TTS respond in caller's language
26. Code-switching (Hinglish) — mixed Hindi+English handled natively

---

## Part 2: Proposed Architecture

### Pipecat replaces the custom pipeline. Everything else stays.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        TELEPHONY LAYER (unchanged)                     │
│                                                                        │
│  telephony_factory.py routes by destination:                           │
│    +91 (India)   → KnowlarityTelephonyProvider (fallback: Twilio)     │
│    International → TwilioTelephonyProvider                             │
│                                                                        │
│  Call initiation: call_service.py (unchanged)                          │
│  Inbound handling: incoming_voice.py (unchanged)                       │
│  LiveKit sessions: livekit.py routes (unchanged)                       │
└──────────┬──────────────────┬──────────────────┬──────────────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ Twilio Transport │ │Knowlarity Transp.│ │ LiveKit Transport │
│                  │ │                  │ │                  │
│ FastAPIWebsocket │ │ FastAPIWebsocket │ │ LiveKitTransport │
│ Transport +      │ │ Transport +      │ │ (built-in)       │
│ TwilioFrame      │ │ KnowlarityFrame  │ │                  │
│ Serializer       │ │ Serializer       │ │                  │
│ (built-in)       │ │ (custom ~80L)    │ │                  │
└────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
         │                    │                     │
         └────────────────────┼─────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     PIPECAT PIPELINE                                   │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ AUDIO INFRASTRUCTURE (Pipecat + TTS provider — not VCI)         │   │
│  │                                                                 │   │
│  │  Silero VAD ──→ Smart Turn v3 (EOU, 12ms) ──→ Barge-in mgmt   │   │
│  │  ElevenLabs Scribe STT (streaming, auto language detection)     │   │
│  │  ElevenLabs TTS multilingual v2 (auto-detects language          │   │
│  │    from response text — no explicit directive needed)           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                         │
│                              ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ VCI 4 LAYERS — Conversation Intelligence (Phase 2 — we build)  │   │
│  │                                                                 │   │
│  │  ┌──── Parallel Pipeline ────────────────────────────┐          │   │
│  │  │  L1 ContextAssembler — chat memory, briefing       │          │   │
│  │  │     summary, authority rules                       │          │   │
│  │  │  L2 PhaseTracker — opening / middle / closing      │          │   │
│  │  │  L3 RecoveryMonitor — stuck-loop, repeat detection │          │   │
│  │  │  SignalDetector — frustration, hedge, consent      │          │   │
│  │  └───────────────────────────────────────────────────┘          │   │
│  │                              │                                  │   │
│  │  Orchestrator — phase transitions, consent, escalation          │   │
│  │                              │                                  │   │
│  │                              ▼                                  │   │
│  │  L4 LLM — Claude with conversation-focused prompt               │   │
│  │    + pragmatic intelligence (hedge/commitment detection)        │   │
│  │    + emotional adaptation (tone via word choice)                │   │
│  │    + recovery patterns (L3) embedded in prompt                  │   │
│  │    + auto-language matching (respond in caller's language)      │   │
│  │    + ZERO tool awareness — only knows `task_handoff` to         │   │
│  │      delegate anything it can't do itself to the main agent     │   │
│  │                              │                                  │   │
│  │  TaskOutcomeDetector — committed / deferred / refused           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                         │
│                              ▼                                         │
│  Response text → ElevenLabs TTS (auto-detects language from text)      │
│                              │                                         │
└──────────────────────────────┼─────────────────────────────────────────┘
                               │
                               ▼
                         Back to caller
                    (via same transport it came from)


OWNER TRANSFER (unchanged — outside pipeline):

  During call → intent detected → FCM push to owner
    ├── LiveKit path: Owner joins same WebRTC room, Pipecat agent disconnects
    └── Twilio path: Dial to device client
```

### LangGraph Inside the Pipecat Pipeline

The VCI intelligence (Layers 1-4) runs as a **LangGraph agent** inside Pipecat's LLM FrameProcessor. LangGraph provides state management, parallel tool execution, and context compaction — while Pipecat handles the real-time audio pipeline.

**Three LangGraph nodes:**

```
┌────────────────────────────────────────────────────────────────┐
│                       LANGGRAPH AGENT                          │
│                    (inside Pipecat pipeline)                    │
│                                                                │
│  L4 / VCI NODE (the brain — single LLM call per turn)          │
│    - Reads conversation state + context each turn              │
│    - Generates response text + task_handoffs                   │
│    - Decides when it needs help from the main agent            │
│    - HAS ZERO TOOL AWARENESS — never knows skill names         │
│    - Sees task_handoffs running/completed (by description)     │
│    - Keeps talking while task_handoffs run in background       │
│    - Delivers task results naturally when they arrive          │
│    - ALL intelligence lives here — one LLM, one call           │
│                                                                │
│  TASK DISPATCHER NODE (dumb worker — no LLM)                   │
│    - Runs in PARALLEL with L4 node                             │
│    - Picks up task_handoffs from shared state                  │
│    - Forwards each natural-language description to the         │
│      main Application Agent                                    │
│    - Application Agent routes to the right skill in Trybo's    │
│      full registry: web research, transcript search, calendar, │
│      consent request, WhatsApp send, payment, file extraction, │
│      etc. — VCI never sees these names                         │
│    - Writes results back to shared state as completed_tasks    │
│    - L4 reads results on next turn (by description)            │
│    - Multiple task_handoffs can run concurrently               │
│                                                                │
│  COMPACTION NODE (dumb worker — no LLM)                        │
│    - Runs in PARALLEL when context exceeds threshold           │
│    - Summarizes older conversation turns (~50 tokens)          │
│    - Keeps recent 5 turns verbatim                             │
│    - Drops delivered tool results, used OTPs                   │
│    - Writes compacted context back to state                    │
│    - Keeps per-turn latency stable over long calls             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**VCI node's shared state (what the LLM reads each turn):**

```python
class VCIState(TypedDict):
    # Conversation
    messages: list              # conversation history (compacted + recent)
    phase: str                  # opening / middle / closing
    caller_language: str        # auto-detected from STT (used to instruct
                                # the LLM to mirror; NOT a TTS directive)

    # Context (L1)
    chat_memory: str            # why this call was triggered
    briefing_summary: str       # ~150-tok summary of recent cross-channel
                                # interactions, generated at call start
    authority_rules: dict       # what needs consent vs autonomous

    # Task delegation (VCI has ZERO tool awareness — only task_handoff)
    running_tasks: list         # task_handoffs currently being handled
                                # by main Application Agent (by description)
    completed_tasks: list       # results ready, not yet delivered to caller
    delivered_tasks: list       # results already communicated

    # Outcomes
    task_outcome: str           # committed / deferred / hedged / refused
```

**Why VCI generates response + task_handoffs in one call (no separate orchestrator for delegation):**

The LLM is the only component that understands the caller's intent. Adding a separate orchestrator LLM to decide "does this need delegation?" would mean two LLM calls per turn — doubling latency. Instead, the VCI LLM does everything in one call: understands intent, generates response, emits task_handoffs if needed.

```python
# VCI LLM returns structured output — one call does everything:
{
    "response": "Let me look that up for you. In the meantime,
                 anything else on the vendor side?",
    "task_handoffs": [
        # VCI describes what it needs in natural language;
        # the main Application Agent routes to the right skill.
        # VCI does NOT know which skill executes this.
        { "description": "Find the current Mumbai steel market rate" }
    ],
    "task_outcome": null,
    "phase_transition": null
}
```

No `tool_requests` field. No tool names, no skill schemas. The only thing VCI emits besides the response and outcomes is a list of natural-language task descriptions for the main Application Agent to route.

The response text **is** the language directive — the LLM emits the response in the caller's language and ElevenLabs multilingual v2 TTS auto-detects from the text. No `language` field in the output.

**What VCI knows about its own capabilities (zero tool awareness):**

```
L1 ContextAssembler provides:

  authority_rules:
    autonomous: collect info, schedule callbacks
    needs_consent: calendar commitments, share pricing

  task_handoff_available: true
    // VCI knows it can describe ANY task it can't do itself
    // and the main Application Agent will handle it.
    // VCI does NOT see a list of skills, schemas, or names.
    // Adding/changing skills in Trybo never touches VCI's prompt.

  not_available:
    - bank/email access, payment execution, medical/legal records
    // VCI declines honestly when caller asks for these.
```

If the caller asks for something genuinely beyond reach, VCI declines: "I'm not able to access that. I can have Krishna get back to you on that."

**How parallel task delegation works during a call:**

```
Turn 3: Caller asks about steel rate
  VCI: "Let me look that up. Anything else on vendors?"
  → task_handoff: { description: "Find current Mumbai steel rate" }
  → Task Dispatcher forwards to Application Agent
  → App Agent routes to web_research skill, runs in background
  → VCI never sees the skill name

Turn 4: Caller asks about delivery date (task still running)
  VCI: (state shows running_tasks: ["Find current Mumbai steel rate"])
       "Sharma confirmed delivery for the 25th in the last WhatsApp.
        Still pulling up the steel rate — should have it shortly."
  → VCI sees the task description and mentions it naturally

Turn 5: Caller says "that's all on vendors" (result arrives)
  VCI: (state shows completed_tasks: [{description: "...steel rate",
                                       result: "₹52,000/tonne"}])
       "Got it. Oh, and the steel rate — it's currently around
        ₹52,000 per tonne in Mumbai, down about 3% from last month."
  → VCI delivers result naturally at the right moment
```

**Context compaction over long calls:**

```
Turn 1-5:   Context ~800 tokens. No compaction.
Turn 6-10:  Context ~1500 tokens. Still okay.
Turn 11-15: Context ~2500 tokens. Threshold exceeded.
            → Compaction node runs in background
            → Turns 1-10 summarized into ~100 tokens
            → Recent turns kept verbatim
Turn 16:    Context back to ~800 tokens. LLM call stays fast.

Compaction rules:
  ALWAYS KEEP: system prompt, phase state, authority rules, briefing summary,
               active tool results
  COMPACT: conversation turns older than last 5
  DROP: used OTP values, expired tool results, duplicate info
```

### Two-Tier Prompt Composition (briefing + on-demand retrieval)

L1's static context loaded at call start is split by token cost:

| Static item | Size | Strategy |
|---|---|---|
| Chat memory (why this call) | ~150 tok | Rides every turn |
| Authority rules | ~80 tok | Rides every turn |
| Behavioral profile | ~80 tok | Rides every turn |
| `task_handoff` instruction | ~50 tok | Rides every turn (one generic capability, NOT a tool list) |
| **Cross-channel briefing summary** | **~150 tok** | **Summarized at call start; rides every turn** |
| **Raw cross-channel transcripts** | **NOT in voice pipeline** | **Held outside VCI; retrieved via `task_handoff` to Application Agent when caller references specifics** |

```
Cacheable prefix (stable across turns, prompt-cache hits):
  System prompt + chat memory + authority rules +
  behavioral profile + task_handoff instruction +
  cross-channel briefing summary
  ──────────────────────────────────────────────
  ~1,250 tokens, paid once via prompt caching

Dynamic per-turn:
  Compacted conversation history + phase state +
  module outputs + orchestrator directive (if any) +
  retrieved tool results (if any) + user message
  ──────────────────────────────────────────────
  ~700-2,000 tokens, recomposed each turn
```

Result: relationship continuity is always present (briefing). Specific details are recoverable on demand (tool retrieval). Per-turn token cost stays bounded regardless of how rich the contact's history is.

### Streaming Pipeline — How Latency Works

The entire pipeline streams — STT, LLM, and TTS overlap. The caller doesn't wait for each stage to finish:

```
STREAMING (everything overlaps):

  [── STT streaming partial transcripts ──]
              ↓ (final transcript ready)
              [── LLM streaming tokens ──────────]
                    ↓ (first few tokens ready)
                    [── TTS streaming audio ──────────]
                          ↓ (first audio chunk ready)
                          [── Caller hears response ──]
  
  0ms      Caller stops speaking
  200ms    Smart Turn v3: "they're done"
  200ms    L4 / VCI node receives transcript + state
  700ms    LLM first tokens arrive
  900ms    TTS first audio chunk ready
  ~1000ms  Caller hears FIRST WORD of response
           (rest streams while caller listens)
```

### Realistic Latency Targets

The LLM is the bottleneck we cannot remove. Honest numbers based on industry reality:

```
REALISTIC LATENCY (end-to-end, time to first word):

  Smart Turn v3 EOU detection:          200-400ms
  Network (audio to server):            50-100ms
  STT finalization:                     100-200ms
  LangGraph state read + routing:       20-30ms
  LLM first token (Claude Sonnet):      500-800ms  ← the bottleneck
  TTS first audio chunk (ElevenLabs):   200-400ms
  Network (audio back to caller):       50-100ms
  ─────────────────────────────────────────────────
  Total first word (streaming):         ~1.1-2.0s

  TARGETS:
    Phase 1 (Pipecat, no VCI layers):   p50 ~1.3s, p90 ~1.8s
    Phase 2 (Pipecat + VCI context):    p50 ~1.5s, p90 ~2.0s
    Today:                              p90 ~3.5s

  Improvement: ~1.5-2.0 seconds faster vs. current Trybo baseline.
  Phase 2 matches industry leaders at p90; does not beat them.
  LLM latency: unchanged — this is the industry bottleneck.
```

Industry comparison for context:

| Platform | Real-World p90 |
|---|---|
| Vapi | ~1.5-2.0 seconds |
| Retell AI | ~1.2-1.8 seconds |
| Trybo today | ~3.5 seconds |
| **Trybo Phase 1 target** | **p90 1.8s** (matches Vapi, slightly behind Retell) |
| **Trybo Phase 2 target** | **p90 2.0s** (Phase 1 + ~150–200ms VCI overhead) |

---

## Part 3: Every Use Case in the Proposed Architecture

### Outbound Calls

#### UC-1: Task-Driven Outbound Call

```
Agentic chat: "Call John about the board meeting report"
    │
    ▼
Application Agent (LangGraph) → creates bot_task → call_service.create_call()
    │
    ▼
telephony_factory: +91? → Knowlarity / else → Twilio
    │
    ▼
Provider dials recipient → recipient answers → WebSocket connects
    │
    ▼
PIPECAT PIPELINE (new):
    │
    ├── Transport: TwilioFrameSerializer or KnowlarityFrameSerializer
    ├── VAD: Silero (replaces custom RMS)
    ├── EOU: Smart Turn v3 at 12ms (replaces 1000-2500ms silence gaps)
    ├── STT: ElevenLabs Scribe (detects language automatically)
    │
    ├── [Phase 2 — VCI 4 layers]:
    │   ├── L1 ContextAssembler: at call start, loads chat memory
    │   │     ("Call John urgently, board meeting tomorrow") and
    │   │     summarizes recent cross-channel transcripts (last
    │   │     WhatsApp with John) into ~150-tok briefing. Loads
    │   │     authority rules + tool capabilities (fast lane +
    │   │     task_handoff)
    │   ├── L2 PhaseTracker: phase=opening
    │   ├── L3 RecoveryMonitor: clean state at start
    │   ├── Orchestrator: "normal — proceed"
    │   │
    │   ├── L4 LLM: generates opening with full context:
    │   │     "Hi John, this is calling for Krishna. He's preparing for
    │   │      the board meeting tomorrow and needs the school visit report.
    │   │      Is now a good time?"
    │   │   (empathetic / warm tone is in the word choice — TTS reads it)
    │   │
    │   └── TTS: ElevenLabs (cloned voice, multilingual v2 auto-routes
    │            language from response text)
    │
    ▼
Conversation continues turn-by-turn through pipeline:
    │
    ├── Caller: "I'll try to send it by Thursday"
    │   → STT transcribes → L2 PhaseTracker: middle
    │   → L4 LLM detects HEDGE (pragmatic intelligence)
    │   → L4 probes: "Thursday works. Could we lock that in as a firm deadline?"
    │   → TaskOutcomeDetector: status=hedged
    │
    ├── Caller: "Yeah, Thursday for sure"
    │   → L4: "Perfect. I'll let Krishna know."
    │   → TaskOutcomeDetector: status=committed, deadline=Thursday
    │   → L2 PhaseTracker transitions → closing
    │
    └── Natural closing (L2): summary + goodbye over 1-2 turns
        → Call completes → transcript summary → bot_task updated
```

**What changes from today:**
- Intro is context-aware (chat memory + cross-channel briefing) instead of bare task_text
- Smart Turn v3 detects end-of-utterance in ~200ms instead of 1-2.5s silence
- LLM probes hedges instead of treating "I'll try" as success
- Natural phase-based closing instead of keyword-triggered disconnect

---

#### UC-2: Scheduled Outbound Call

No change to scheduling logic. When the scheduler fires:

```
APScheduler → run_scheduled_task() → call_service.create_call()
    → same Pipecat pipeline as UC-1
```

---

#### UC-3: Batch/Multi-Contact Outbound

No change to batch logic. Each contact gets a parallel call:

```
batch_service → for each contact → call_service.create_call()
    → each call runs through its own Pipecat pipeline instance
    → batch_progress tracks completion across all calls
```

---

#### UC-4: Knowlarity Fallback to Twilio

```
telephony_factory: +91 → Knowlarity
    │
    ├── Knowlarity API succeeds → KnowlarityFrameSerializer → Pipecat pipeline
    │
    └── Knowlarity API fails → fallback to Twilio
        → TwilioFrameSerializer → same Pipecat pipeline
        
Pipeline code is IDENTICAL regardless of which serializer is used.
```

---

### Inbound Calls

#### UC-5: Inbound from Unknown Caller

```
External caller dials Twilio voip_number
    │
    ▼
/incoming-voice webhook
    │
    ▼
check_direct_transfer_eligibility() → NO (unknown caller)
    │
    ▼
PIPECAT PIPELINE:
    │
    ├── L1 ContextAssembler: no prior history (unknown caller) — empty briefing
    ├── L2 PhaseTracker: phase=opening
    ├── L4 LLM: "Hi there! Thanks for calling Krishna. How can I help?"
    │
    ├── Caller: "I'm calling about the vendor pricing we discussed"
    │   → L1: no cross-channel history found (unknown contact)
    │   → L4: "I'd be happy to help. Could you tell me your name
    │           and which vendor pricing you're referring to?"
    │
    └── Conversation continues → caller identified → context loaded dynamically
        (and may dispatch transcript_retrieval if a contact match is found)
```

---

#### UC-6: Inbound from Known Contact

```
Known contact calls voip_number
    │
    ▼
/incoming-voice → contact lookup → is_known_contact=True
    │
    ▼
PIPECAT PIPELINE:
    │
    ├── L1 ContextAssembler: at call start, generates briefing summary
    │     from recent cross-channel transcripts (last WhatsApp thread +
    │     last call transcript with this contact) + open bot_tasks
    │
    ├── L4 LLM: "Hi Neha! Thanks for calling. Are you calling about
    │            the vendor pricing we messaged about yesterday?"
    │           (cross-channel awareness — briefing knows about WhatsApp)
    │
    └── If contact says "like I told you on WhatsApp..."
        → Briefing already carries the gist
        → If a specific detail is needed, L4 emits a task_handoff
          ("Find what we discussed with Neha about pricing last week")
        → Application Agent routes to the right skill; result lands
          in shared state; L4 weaves it in on the next turn
```

**What changes from today:**
- Agent greets with cross-channel awareness (briefing summary covers recent WhatsApp threads)
- Specific past details are pulled on demand via transcript_retrieval — not bulk-loaded into the prompt

---

#### UC-7: Inbound During DND

```
Caller dials → check_dnd_status() → DND active
    │
    ▼
PIPECAT PIPELINE:
    │
    ├── L1 ContextAssembler: DND context injected into briefing
    ├── L4 LLM: "Hi, you've reached Krishna's line. He's currently
    │            unavailable. Can I take a message or help with anything?"
    │
    └── Conversation continues with DND-aware behavior
```

---

#### UC-8: Direct Device Transfer (Known Contact, No AI)

**No change.** This bypasses the AI pipeline entirely:

```
Known contact calls → check_direct_transfer_eligibility() → YES
    → Twilio Dial to device client → owner's phone rings
    → No Pipecat involved
    → Recording transcribed post-call via Deepgram (unchanged)
```

---

### During-Call Scenarios

#### UC-9: Owner Transfer (LiveKit WebRTC)

```
During Pipecat pipeline conversation:
    │
    ├── Caller: "Can I speak to Krishna directly?"
    │   → L4 LLM detects transfer intent
    │   → Pipeline returns should_transfer=True
    │
    ▼
voice_service._request_livekit_owner_transfer() (unchanged):
    │
    ├── FCM push to owner's devices
    ├── Owner taps Accept → /livekit/owner-token → joins LiveKit room
    ├── Pipecat agent disconnects from pipeline
    ├── Caller + Owner talk directly in LiveKit room (no AI)
    │
    └── If timeout (35s) → agent resumes in pipeline
```

---

#### UC-10: Owner Transfer (Twilio)

```
During Pipecat pipeline conversation:
    │
    ├── Caller: "Can I speak to Krishna?"
    │   → L4 LLM detects transfer intent
    │
    ▼
Twilio Dial to device client (unchanged):
    → Owner's device rings → owner answers → voice bridge
    → Pipecat pipeline stops
```

---

#### UC-11: Information Request (Consent Gate)

```
During Pipecat pipeline conversation:
    │
    ├── Caller: "What does Krishna's calendar look like on Friday?"
    │   → L4 LLM detects information_retrieval intent
    │   → L1 authority rules: calendar access → needs consent
    │
    ▼
L4 emits filler opening + a task_handoff in natural language:
    │
    ├── L4 response: "Let me check Krishna's schedule for you,
    │                  one moment..."
    │   task_handoff: { description: "Get permission from Krishna
    │                    to share his Friday schedule, then return it" }
    │
    ├── Application Agent receives the description, routes to the
    │   consent + calendar skills (information_request_processing_service
    │   unchanged), pushes FCM to owner, owner approves, calendar data
    │   retrieved → result lands in shared state as completed_task
    │
    ├── L4 next turn (reads completed_task by description):
    │   "So Krishna is free Friday afternoon from 2 to 5.
    │    Would that work for you?"
    │
    └── If owner doesn't respond → L3 RecoveryMonitor flags timeout →
        L4: "I don't have that info right now. I'll have Krishna
              get back to you on that."
```

---

#### UC-12: OTP Delivery

```
During Pipecat pipeline conversation:
    │
    ├── Caller: "Can you read me the OTP?"
    │   → L4 LLM detects OTP request
    │
    ├── FCM push to owner → owner enters OTP on mobile
    ├── OTP stored in session → pipeline retrieves it
    │
    └── L4: "The OTP is 1, 2, 3, 4, 5, 6."
        (clear digit pacing comes from punctuation in the response text;
         ElevenLabs reads commas as pauses naturally — no per-turn
         TTS rate directives)
```

---

#### UC-13: Barge-In

```
Agent speaking (TTS playing):
    │
    ├── Caller starts talking
    │   → Silero VAD detects speech onset (~50ms)
    │   → Pipecat AUTOMATICALLY cancels pending LLM/TTS frames
    │   → TTS stops immediately
    │   → STT begins capturing caller's speech
    │
    └── No custom code needed — Pipecat handles this natively
```

**What changes from today:**
- Silero VAD (ML-based) replaces custom RMS threshold detection
- Pipecat's pipeline framework handles TTS cancellation automatically
- No manual `clear_agent_audio_queue()` calls needed

---

#### UC-14: Call Hold

```
Information request in progress:
    │
    ├── Pipecat pipeline: agent speaks hold message
    │   "I'm checking on that for you. One moment please."
    │
    ├── Pipeline stays active (STT still listening)
    │   → If caller says "never mind" or "cancel"
    │   → L4 detects → resumes normal conversation
    │
    └── When info arrives → pipeline resumes with the data
```

---

#### UC-15: Live Transcription

```
Every turn through the Pipecat pipeline:
    │
    ├── STT produces transcript → TranscriptPersister FrameProcessor
    │   → persist_transcript_turn(call_sid, speaker, text)
    │   → Stored in transcripts_segments table (unchanged)
    │
    └── Post-call: transcript assembled + summary generated (unchanged)
```

Note: back-channel audio clips (mm-hmm/yeah played during caller's turn) are NOT persisted as transcript turns — they are audio-only events invisible to the conversation history.

---

#### UC-16: Voice Cloning

```
Pipeline initialization:
    │
    ├── resolve_call_voice_config() (unchanged):
    │   → User's voice_profile → voice_id + provider
    │
    ├── Pipecat TTS FrameProcessor configured with:
    │   → ElevenLabs voice_id = user's cloned voice
    │   → multilingual_v2 model (auto-detects language from response text)
    │
    └── Same cloned voice used across all languages automatically
```

---

### Multilingual Scenarios

#### UC-17: Caller Speaks Hindi

```
Caller speaks Hindi
    │
    ▼
STT (ElevenLabs Scribe):
    → Transcribes: "मुझे रिपोर्ट भेजनी है कल तक"
    → Detected language: "hi" (used to set caller_language in state)
    │
    ▼
L4 LLM:
    → System prompt: "Respond in the same language the caller is speaking"
    → Receives Hindi text, responds in Hindi:
      "ठीक है, मैं कृष्णा को बता दूंगा। कल तक भेज दीजिए।"
    → No explicit language directive in the structured output —
      the response text itself is the language signal
    │
    ▼
TTS (ElevenLabs multilingual v2):
    → Auto-detects language from response text characters (Hindi)
    → Speaks Hindi text in owner's cloned voice
    → Caller hears Hindi response
```

---

#### UC-18: Caller Switches Language Mid-Call

```
Turn 1: Caller speaks English → L4 responds in English text → TTS English
Turn 2: Caller speaks English → L4 responds in English text → TTS English
Turn 3: Caller switches to Hindi → STT detects → caller_language updates
    → L4 responds in Hindi text (mirrors current language)
    → TTS auto-detects Hindi from text → speaks Hindi
Turn 4: Caller speaks Hinglish → STT transcribes mixed
    → L4 responds in same Hinglish mix
    → TTS reads mixed-language text and speaks it natively
```

No explicit language directive at any layer — the text carries the language.

---

#### UC-19: Code-Switching (Hinglish)

```
Caller: "Report ready hai, but photos add karne hain"
    │
    ▼
STT: transcribes as-is (mixed Hindi + English)
L4 LLM: responds in same mix: "Okay, photos add karke Thursday tak bhej dijiye"
TTS: reads the mixed text and speaks naturally with code-switched pronunciation
```

---

### Post-Call Scenarios

#### UC-20: Transcript Summary + Task Update

No change to post-call logic:

```
Call ends → Pipecat pipeline stops
    │
    ▼
Existing post-call services (unchanged):
    ├── generate_call_summary() → LLM summarizes conversation
    ├── update_bot_task_from_call() → task status + output
    ├── update_transcript_full_text() → finalize transcript
    └── cleanup_call_session() → clear Redis session
```

**Enhancement from Phase 2:** TaskOutcomeDetector provides richer task status:
- Today: binary done / not done
- Proposed: committed / deferred / hedged / refused / blocked

Post-call task_handoffs (slow lane) emitted by L4 during the call are dispatched to the Application Agent here for execution against the appropriate skill (e.g. send a follow-up WhatsApp tomorrow morning).

---

### WebRTC Calls

#### UC-21: Browser/App Call via LiveKit

```
Browser user initiates call → /livekit/session
    │
    ▼
LiveKit room created → Pipecat agent joins via LiveKitTransport
    │
    ▼
Same Pipecat pipeline as Twilio/Knowlarity calls:
    → Silero VAD → Smart Turn v3 → ElevenLabs STT → L4 LLM → TTS
    → VCI 4 layers (Phase 2) — same intelligence, different transport
    │
    ▼
Owner transfer: owner joins same LiveKit room
    → Pipecat agent disconnects → human-to-human conversation
```

---

## Part 4: What Changes, What Stays

### Changes (Phase 1 — Pipecat Integration)

| Component | Today | Proposed |
|---|---|---|
| `agent_stream_service.py` (1130L) | Custom WebSocket + ASR + turn mgmt | **Replaced** by Pipecat pipeline config |
| `media_stream_service.py` (700L) | Custom turn orchestration | **Replaced** by Pipecat pipeline streaming |
| `knowlarity_stream_service.py` (900L) | Custom Knowlarity WebSocket | **Replaced** by `KnowlarityFrameSerializer` (~80L) |
| `livekit_stream_service.py` (900L) | Custom LiveKit RTC handling | **Replaced** by Pipecat `LiveKitTransport` |
| VAD | Custom RMS energy detection | Pipecat Silero VAD |
| Turn detection | Silence-based (1000-2500ms) | Pipecat Smart Turn v3 (12ms, ML-based) |
| ASR | Cartesia (custom integration) | ElevenLabs Scribe via Pipecat (with language detection) |
| TTS streaming | Custom frame-by-frame | Pipecat TTS FrameProcessor (multilingual v2 auto-routes language from text) |
| p90 latency | ~3.5 seconds | Target p50 1.3s / p90 1.8s (Phase 1) |

### Changes (Phase 2 — VCI 4 Layers)

| Component | Today | Proposed |
|---|---|---|
| Conversation phases | None — flat turn sequence | L2 PhaseTracker FrameProcessor |
| Response style | Rigid 25-word cap, constraint prompts | L4 conversation-focused prompt; tone via word choice |
| Pragmatic intelligence | Binary done/not-done | L4 hedge/commitment/deferral detection |
| Emotional adaptation | None | L4 LLM-driven via word choice + phrasing (TTS reads from text) |
| Cross-channel context | Bare task_text, no WhatsApp history | L1 ContextAssembler — chat memory + briefing summary at call start; raw transcripts held outside VCI |
| On-demand specifics | Not supported | `task_handoff` to Application Agent — VCI describes what it needs, App Agent routes to the right skill |
| Recovery | Robotic fallbacks | L3 RecoveryMonitor + repair patterns in prompt |
| Language matching | Not supported | LanguageDetector FrameProcessor + L4 prompt mirroring + ElevenLabs multilingual v2 (no explicit language directive) |
| Task outcomes | Binary | committed / deferred / hedged / refused / blocked |
| Task delegation | Not supported | VCI has zero tool awareness — only knows `task_handoff(description)`. Application Agent owns all skill routing across Trybo's full registry. |

### Stays Unchanged

| Component | Why |
|---|---|
| `telephony_factory.py` | Provider routing logic stays — Pipecat uses same providers |
| `call_service.py` | Call creation, DB records, status tracking — no pipeline dependency |
| `twilio_service.py` | Twilio REST API for call initiation — unchanged |
| `knowlarity_service.py` | Knowlarity API for call initiation — unchanged |
| `voice_service.py` | Business logic (transfer, consent, disconnect intent) — called from the Application Agent when it routes a task_handoff |
| `conversation_service.py` | LLM response generation — becomes a Pipecat LLM FrameProcessor |
| `incoming_call_service.py` | Direct transfer eligibility — runs before pipeline starts |
| `call_hold_service.py` | Hold message logic — triggered from within pipeline |
| `information_request_processing_service.py` | Consent flow — triggered by Application Agent when it routes a consent-related task_handoff |
| `transcript_service.py` | Transcript persistence — called from TranscriptPersister FrameProcessor |
| `voice_cloning_service.py` | Voice profile resolution — used to configure TTS FrameProcessor |
| `livekit_session_service.py` | Token generation, room management — unchanged |
| `firebase_service.py` | FCM push notifications — unchanged |
| `scheduler_service.py` | APScheduler for scheduled calls — unchanged |
| `batch_service.py` | Batch call orchestration — unchanged |
| All routes | API endpoints stay the same — internal pipeline changes are transparent |
| Redis session store | Session state management — Pipecat uses same session data |
| Database schema | calls, transcripts, bot_tasks, voice_profiles — all unchanged |

---

## Part 5: Latency Improvement by Use Case

| Scenario | Today | With Pipecat (Phase 1) | With VCI + LangGraph (Phase 2) |
|---|---|---|---|
| **End-of-utterance detection** | 1000-2500ms (silence) | ~200-400ms (Smart Turn v3) | Same |
| **First response after pickup** | ~3.5s | p50 ~1.3s, p90 ~1.8s | p50 ~1.5s, p90 ~2.0s (richer context-aware intro) |
| **Mid-conversation turn** | ~3.5s | p50 ~1.3s (streaming overlap) | p50 ~1.5s (LangGraph adds ~20ms, context delta read ~10ms) |
| **Turn with task_handoff** | Not supported | Same as mid-conversation | ~1.5s for response opening + task runs in PARALLEL via Application Agent (caller never waits) |
| **Turn when task result arrives** | Not supported | Not supported | Same ~1.5s (L4 weaves result into natural response) |
| **Barge-in response** | 200-500ms to stop TTS | ~50-100ms (Silero VAD + auto-cancel) | Same |
| **Language switch** | Not supported | ~1.5s (auto-detect + respond) | Same |
| **Information request** | Call goes on hold | Same | L4 keeps engaging caller while Application Agent handles task_handoff in background |
| **Owner transfer initiation** | Same | Same (FCM push timing unchanged) | Same |
| **Long call (20+ turns)** | Context grows → latency creeps up | Same risk | Compaction node keeps context stable → latency stays flat |

**LangGraph overhead per turn: ~10-20ms** (state read/write + node routing). Negligible compared to LLM latency (500-800ms).

---

## Part 6: Pipeline Code Structure

```
app/
├── pipecat/                          # NEW — Pipecat integration (Phase 1)
│   ├── pipeline_factory.py           # Creates pipeline for each call
│   ├── serializers/
│   │   └── knowlarity.py             # KnowlarityFrameSerializer (~80L)
│   ├── processors/                   # Custom FrameProcessors
│   │   ├── langgraph_processor.py    # Wraps LangGraph agent as a FrameProcessor
│   │   ├── transcript_persister.py   # Per-turn transcript saving
│   │   ├── language_detector.py      # Auto-language detection from STT
│   │   └── back_channel.py           # Pre-synthesized "mm-hmm" / "yeah"
│   │                                 #   played during caller's turn
│   └── config.py                     # Pipeline configuration
│
├── langgraph/                        # NEW — LangGraph VCI agent (Phase 2)
│   ├── vci_agent.py                  # LangGraph graph definition (3 nodes)
│   ├── nodes/
│   │   ├── vci_node.py               # L4 — single LLM call per turn
│   │   │                             #   conversation intelligence (L1-L4)
│   │   │                             #   task_handoff emission only
│   │   │                             #   (zero tool awareness)
│   │   │                             #   response opening + streaming
│   │   ├── task_dispatcher.py        # Task Dispatcher node — no LLM
│   │   │                             #   forwards task_handoff descriptions
│   │   │                             #   to the main Application Agent;
│   │   │                             #   writes results back to state
│   │   └── compaction_node.py        # Compaction node — no LLM
│   │                                 #   summarize old turns, drop used data
│   ├── state.py                      # VCIState TypedDict (shared state)
│   │                                 # NOTE: there is NO `tools/` directory.
│   │                                 # All skill implementations live with the
│   │                                 # main Application Agent, not VCI.
│   ├── context/
│   │   ├── context_assembler.py      # L1 — at call start: chat memory,
│   │   │                             #   briefing-summary generation,
│   │   │                             #   authority rules
│   │   ├── briefing_summarizer.py    # L1 helper — generates ~150-tok
│   │   │                             #   briefing from raw cross-channel
│   │   │                             #   transcripts at call start
│   │   ├── phase_tracker.py          # L2 — opening/middle/closing
│   │   └── recovery_monitor.py       # L3 — stuck-loop, repeat detection
│   └── prompts/
│       └── vci_system_prompt.py      # L4 — conversation-focused prompt
│                                     #   pragmatic + emotional intelligence
│                                     #   task_handoff instruction (one
│                                     #     generic capability — no tool list)
│                                     #   recovery patterns embedded (L3)
│                                     #   language mirroring instruction
│
├── services/                         # EXISTING — mostly unchanged
│   ├── call_service.py               # unchanged
│   ├── voice_service.py              # unchanged (called from Application Agent)
│   ├── conversation_service.py       # replaced by langgraph/nodes/vci_node.py
│   ├── telephony_factory.py          # unchanged
│   ├── information_request_*.py      # unchanged (called from Application Agent
│   │                                 #   when it routes a consent task_handoff)
│   ├── transcript_service.py         # unchanged (called from transcript_persister)
│   ├── agent_stream_service.py       # REMOVED (Phase 1 — replaced by Pipecat)
│   ├── media_stream_service.py       # REMOVED (Phase 1 — replaced by Pipecat)
│   ├── knowlarity_stream_service.py  # REMOVED (Phase 1 — replaced by serializer)
│   └── livekit_stream_service.py     # REMOVED (Phase 1 — replaced by LiveKitTransport)
```
