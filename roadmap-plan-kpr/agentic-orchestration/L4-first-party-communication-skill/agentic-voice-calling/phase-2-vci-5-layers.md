# Phase 2: VCI 4 Layers — Calling Agent Intelligence

> Owner: KPR
> Source: Industry analysis + codebase audit — April 2026
> Depends on: Phase 1 (Pipecat Voice AI Pipeline) — infrastructure must be in place first
> Related: `phase-1-pipecat-voice-ai-pipeline.md` (pipeline foundation), `vci-platform-decision.md` (platform rationale)
> Latency ceiling: < 2.5s end-to-end per turn

> **Layer numbering convention:** Layers are numbered in the **chronological order they run during a turn**, not by importance. L1 runs first (context, before the LLM); L4 is the LLM call itself.

> **Why 4 layers (not 5):** An earlier draft included a "Voice & Timing" layer for TTS execution, turn-taking, barge-in, and per-turn speech directives (rate/energy/pause). On review, this layer is not VCI's concern: (a) Pipecat handles VAD, end-of-utterance, barge-in, and streaming TTS natively as audio infrastructure; (b) per-turn speech directives are infeasible alongside LLM streaming — TTS would need the directive before audio starts, but with streaming the response text is still being generated. Modern TTS reads emotion from text alone, so empathetic delivery is achieved through the LLM's word choice and phrasing, not through TTS parameter directives. Audio delivery is therefore Pipecat + TTS-provider infrastructure, not a numbered intelligence layer.

---

## Part 1: What Makes a Voice Conversation Feel Human?

A phone conversation feels natural when 4 layers of intelligence work together with audio infrastructure underneath. These layers are not features to add — they are the **fundamental requirements** of natural speech.

```
┌──────────────────────────────────────────────────────────────┐
│  Chronological order — top runs first, bottom runs last      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Layer 1: Contextual Intelligence                           │
│   ─ knows WHY this call is happening, the backstory          │
│   ─ runs at call start (load briefing) and per turn          │
│     (on-demand retrieval as a tool)                          │
│                                                              │
│   Layer 2: Conversation Architecture                         │
│   ─ HOW the conversation flows as a whole                    │
│   ─ phases, topic management, agenda flexibility             │
│                                                              │
│   Layer 3: Recovery & Repair                                 │
│   ─ handles misunderstandings, objections, tangents          │
│   ─ stuck-loop detection, graceful abort, escalation         │
│                                                              │
│   Layer 4: Response Design & Conversation Intelligence       │
│   ─ WHAT the agent says, how it structures responses         │
│   ─ pragmatic: understands MEANING, not just words           │
│   ─ emotional: reads frustration, urgency, energy            │
│   ─ THIS is the LLM call itself                              │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  Audio Infrastructure (Pipecat + TTS provider — not VCI)     │
│   ─ STT, VAD, EOU, barge-in, streaming TTS, back-channels    │
│   ─ TTS auto-detects language from response text             │
└──────────────────────────────────────────────────────────────┘
```

---

### Layer 1: Contextual Intelligence — KNOWING the Situation

This is what the agent knows before and during the conversation. A human calling on behalf of Krishna would know WHY Krishna wants this, WHAT the relationship with this contact has been, and WHAT recent interactions happened. The context sources differ between outbound and inbound calls.

**Outbound calls need THREE context sources:**

1. **Chat history** (from Application Agent) — the owner's conversation in the chat UI that triggered this call. This tells the agent WHY this call is happening. "Krishna said 'Call John urgently, board meeting tomorrow'" is fundamentally different from bare `task_text = "ask about report"`. The chat history carries urgency, deadlines, owner's tone preferences, and background reasoning.

2. **Cross-channel briefing summary** — the most recent 1-2 interactions with this contact (calls + WhatsApp), summarized at call start into a compact briefing (~150 tokens). This gives RELATIONSHIP CONTINUITY. Even if the last interaction was about a different topic, it establishes "we've been in touch" — the agent isn't a cold caller each time. Specific details are not in the briefing — they're recovered on-demand via tool retrieval when the caller references them.

3. **Behavioral profile** — how this contact communicates (response patterns, deferral tendencies, best times).

**Inbound calls need TWO context sources:**

1. **Cross-channel briefing summary** — even MORE critical here because the agent doesn't know why they're calling. Same mechanism as outbound. If John was WhatsApp'd yesterday about a report and now he's calling back — the briefing carries that context.

2. **Behavioral profile** — same as outbound.

(No chat history for inbound — there was no chat UI trigger.)

**What this gives the conversation:**
- **Relationship awareness:** "We've been in touch — last time we spoke about X" instead of treating every call as a first-ever interaction
- **Reference handling:** When the contact says "like I told you last time" or "I already sent that on WhatsApp," the agent can engage instead of going blank (briefing for general continuity, on-demand retrieval for specifics)
- **Continuity across channels:** WhatsApp thread → call or call → WhatsApp follow-up feels like ONE continuous relationship
- **In-call memory:** Referencing things said earlier in the same call

**Industry approach:**
- All platforms pass structured context to the LLM (system prompt, function schemas)
- Some platforms (Bland.ai, Retell) support "knowledge bases" — background documents the agent can reference
- Google Duplex: tightly scoped context (just the task), but Duplex was single-purpose (book appointments)
- Enterprise voice agents (Nuance, Five9): CRM integration provides full customer interaction history

**The key insight:** Context serves two distinct purposes — task context (why this call) and relationship context (what's our history with this person). Chat history provides task context. Briefing summary provides relationship context. Both are needed for the agent to sound like it's part of a continuous relationship.

---

### Layer 2: Conversation Architecture — HOW the Conversation Flows

Every human phone call follows an invisible structure. We don't think about it, but we all do it: greeting → context-setting → core exchange → wrap-up → goodbye. Each phase has different rules.

**What natural conversation architecture requires:**
- **Opening phase:** Greeting, identify yourself, state purpose briefly, check if it's a good time. 2-3 turns of rapport before getting to business.
- **Middle phase:** Flexible information exchange. Follow the caller's direction, steer gently toward the objective. Handle topic changes naturally.
- **Closing phase:** Summarize what was discussed, confirm next steps, say goodbye naturally — over 2-3 turns, not a single sentence.
- **Topic management:** When multiple topics come up, handle them one at a time. Don't jump between topics.
- **Agenda flexibility:** The agent has a goal, but it doesn't force it. If the caller has their own agenda, the agent adapts.

**Industry approach:**
- Bland.ai: conversation "pathways" — structured but flexible dialog trees
- Google Duplex: strict task focus (book appointment) but flexible within the task
- Enterprise IVR design (Nuance, Speechmatics): conversation state machines with recovery paths

**The key insight:** Conversations have an arc. Treating every turn as an independent Q&A makes the experience feel like an interrogation, not a conversation.

---

### Layer 3: Recovery & Repair — Handling the UNEXPECTED

Real conversations are messy. People misunderstand, go off-topic, push back, get distracted, ask unexpected questions. The agent must handle all of this without breaking character.

**What recovery & repair requires:**
- **Misunderstanding repair:** "Oh sorry, I think there was a mix-up — what I meant was..." — take responsibility, reframe, move on.
- **Off-topic handling:** Engage briefly with the tangent, then bridge back naturally. NOT "Going back to our topic..."
- **Objection navigation:** Validate the objection, offer context or alternative, don't just re-ask the same question.
- **Unknown territory:** "I don't have that information with me, but I'll check with Krishna" — honest, not evasive.
- **Technical failure recovery:** If ASR misses something — "Sorry, I didn't catch that clearly. Could you say that again?"
- **Graceful abort:** If the call is going nowhere — "I think it might be easier if Krishna follows up with you directly on this. I'll let him know."

**Industry approach:**
- This is primarily prompt engineering — teaching the LLM repair strategies through instructions and examples
- Some platforms use conversation state to detect when repair is needed (e.g., 3 turns with no progress → suggest alternative)
- Google Duplex: had pre-programmed repair strategies for common failure modes

**The key insight:** Recovery is what makes a conversation feel resilient. An agent that handles the unexpected gracefully feels more human than one that handles the expected perfectly.

---

### Layer 4: Response Design & Conversation Intelligence — The Agent's Brain

This is the central intelligence layer — and it is the LLM call itself. It combines three capabilities that were originally separate but are naturally unified in a single LLM: **response craft** (how to structure what the agent says), **pragmatic intelligence** (understanding meaning beyond words), and **emotional intelligence** (reading and matching the caller's state). Modern LLMs already have strong pragmatic and emotional intelligence — the prompt just needs to unleash it rather than suppress it.

**Response craft — what natural response design requires:**
- **Acknowledgment before content:** "I see" / "Got it" / "Makes sense" before answering. Humans ALWAYS acknowledge before responding. Skipping this makes the agent feel like it wasn't listening.
- **Varied response length:** Sometimes 3 words ("Got it, thanks!"). Sometimes 30 words (explaining something). A rigid word limit produces uniformly choppy speech.
- **Natural connectors:** "So", "By the way", "Actually", "Speaking of which" — these bridge between topics and make speech flow.
- **Filler speech during processing:** "Let me think about that..." / "That's a good question..." — not empty stalling, but natural bridges that the LLM emits as the opening of a streamed response.
- **Appropriate detail level:** Don't over-explain simple things. Don't under-explain complex things. Match the complexity of the response to the complexity of the question.
- **Contractions and casual language:** "I'll" not "I will", "couldn't" not "could not", "gonna" not "going to" — formality kills naturalness.

**Pragmatic intelligence — understanding MEANING, not just words:**
- **Indirect answer detection:** "I'll see what I can do" = soft no / deferral. Don't mark task as completed.
- **Social signal reading:** "Actually, I'm in a meeting" = hang up now. "Let me check..." + long pause = they're actually checking, don't fill the silence.
- **Polite refusal detection:** "That might be difficult" = no. "I'm not the right person" = redirect.
- **Commitment vs hedge detection:** "I'll send it by 5pm" = commitment. "I'll try to send it" = hedge. The agent should probe hedges ("Would tomorrow morning work as a firm deadline?").
- **Question behind the question:** "What time does Krishna usually call?" = "I'd prefer to talk to Krishna directly, not you."

**Emotional intelligence — reading and adapting to the caller's state:**
- **Frustration detection:** Short answers, sighing, repeating themselves, saying "I already told you" — the agent should acknowledge frustration through word choice and phrasing.
- **Urgency matching:** If the caller says "this is really urgent," the agent should respond with urgency — fewer pleasantries, immediate action.
- **Energy matching:** Casual caller = casual agent. Formal caller = professional agent. Don't be cheerful with someone who's upset.
- **Empathy expression:** "I understand that's frustrating" / "I'm sorry about that" — genuine, not scripted.
- **Knowing when to escalate:** If the caller is angry or insistent, offer to connect with the owner — don't keep trying to handle it.

**Industry approach:**
- All major voice AI platforms emphasize prompt engineering for response naturalness
- Pre-synthesized common phrases (greetings, acknowledgments, fillers) eliminate TTS latency for the most frequent responses
- Few-shot examples in prompts teach the LLM the desired conversational style
- Hume AI: real-time emotion detection from voice prosody — but modern LLMs achieve pragmatic/emotional understanding from text alone when properly prompted

**The key insight:** The LLM already has pragmatic and emotional intelligence. The job of this layer is NOT to build that intelligence — it's to stop suppressing it (remove rigid constraints) and let it express itself through word choice, phrasing, response length, and tone-of-voice in text. Modern TTS providers (ElevenLabs multilingual v2) read emotion from text alone — empathetic phrasing produces empathetic delivery without per-turn TTS parameters. Response craft, pragmatic reading, and emotional adaptation are one unified capability in the LLM.

---

### Audio Infrastructure (Pipecat + TTS provider — outside the layer model)

Audio handling is not VCI's concern — it's framework + provider capability. The 4 layers above are the intelligence. The audio side is plumbing:

**Pipecat provides natively:**
- Silero VAD — voice activity detection
- Smart Turn v3 — end-of-utterance detection (12ms, 23 languages)
- Barge-in handling — cancel TTS when caller speaks
- Streaming STT and streaming TTS
- Frame-level audio routing across transports (Twilio / Knowlarity / LiveKit)

**TTS provider (ElevenLabs multilingual v2) provides natively:**
- Sub-300ms first-byte latency
- Auto-language detection from input text (no explicit language directive needed — see "How language flows" below)
- Natural prosody and emotion read from text alone
- Voice cloning across all supported languages with the same cloned voice ID

**What our code adds (small, not a layer):**
- **Back-channel pre-synthesis** — pre-synthesized "mm-hmm" / "yeah" audio clips played without interrupting the caller's turn. Triggered by Pipecat's VAD detecting long mid-utterance pauses. Context-blind by design (semantically empty), invisible to the LLM context (NOT logged into the message store).
- **TTS provider configuration per call** — voice cloning ID and multilingual model selected once at call start from `voice_cloning_service.resolve_call_voice_config()`. Not per-turn.

**How language flows (no explicit directive needed):**

```
Caller speaks Hindi
     │
     ▼
ElevenLabs Scribe STT auto-detects "hi" → transcribes in Hindi
     │
     ▼
Transcript text (Hindi characters) goes to L4 LLM
     │
     ▼
L4 system prompt: "Respond in the same language the caller is speaking"
     │
     ▼
LLM emits response in Hindi text
     │
     ▼
ElevenLabs multilingual v2 TTS auto-detects Hindi from the text
     │
     ▼
Speaks Hindi in owner's cloned voice
```

The LLM **never** emits an explicit language directive like `language: "hi"`. The TTS reads the language directly from the response text characters. Hindi text in → Hindi audio out. English text in → English audio out. Mixed Hinglish text in → mixed audio out, with natural code-switching pronunciation. The same cloned voice ID is used across all languages.

If the caller switches language mid-call, the same flow plays out automatically — no explicit signal, no extra LLM instruction, no special handling. The text carries the language; the TTS reads the text.

---

## Part 2: Runtime Architecture — The Brain Model

The 4 layers are a design framework for what the system needs to know and do. But at runtime, they don't execute as a single monolithic prompt. They operate like the human brain: **specialized modules** handle focused jobs in parallel, feeding their outputs to a **central orchestrator** (the prefrontal cortex) that has visibility into all modules and intervenes when the situation demands coordination.

### Why Not a Single LLM Call?

Stuffing all 4 layers into one prompt creates a monolithic system where everything is entangled. The brain doesn't work this way — the amygdala (emotion), hippocampus (memory), Wernicke's area (comprehension), and Broca's area (speech) all run in parallel. The prefrontal cortex doesn't do their jobs — it monitors their outputs and intervenes when coordination or override is needed.

### The Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                 ORCHESTRATOR (Prefrontal Cortex)             │
│                                                             │
│   Lightweight LLM / LangGraph supervisor node               │
│   Sees outputs from ALL specialized modules                 │
│   Usually says: "proceed normally"                          │
│   But CAN intervene:                                        │
│     → "This needs consent — pause and request"              │
│     → "Caller is frustrated — switch to empathetic mode"    │
│     → "We're stuck — trigger graceful abort"                │
│     → "Phase should shift to closing now"                   │
│     → "This crosses authority boundary — escalate"          │
│                                                             │
└────────┬──────────┬──────────┬──────────┬───────────────────┘
         │          │          │          │
    ┌────▼───┐ ┌───▼────┐ ┌──▼───┐ ┌───▼─────┐
    │ L1     │ │ L2     │ │Signal│ │ L3      │
    │Context │ │Phase   │ │Detect│ │Recovery │
    │Assembly│ │Tracker │ │      │ │Monitor  │
    └────┬───┘ └───┬────┘ └──┬───┘ └───┬─────┘
         │         │         │         │
         └─────────┴────┬────┴─────────┘
                        │
                        ▼
              ┌──────────────────┐
              │ L4 Response      │
              │ Generator (LLM)  │
              │                  │
              │ Receives:        │
              │  - directive     │
              │    from          │
              │    orchestrator  │
              │  - context (L1)  │
              │  - phase (L2)    │
              │  - signals       │
              │  - recovery (L3) │
              │                  │
              │ Generates:       │
              │  - response text │
              │  - tool requests │
              │  - task handoffs │
              │  - outcomes      │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ Audio Infra      │
              │ (Pipecat + TTS)  │
              │  - streaming TTS │
              │  - back-channels │
              │  - barge-in      │
              └──────────────────┘
```

### Module Responsibilities

| Module | Type | Job | Latency |
|---|---|---|---|
| **L1 Context Assembly** | DB queries + briefing summary at call start; tool retrieval per turn | Provide chat memory, briefing summary, authority rules; fetch specifics on demand | ~30-60ms once at call start; ~5ms per-turn read |
| **L2 Phase Tracker** | Session state read | Report current phase (opening/middle/closing), turn count, objective status | ~near instant |
| **Signal Detector** | Lightweight classifier | Detect frustration, urgency, hedges, consent-triggering requests from transcript | ~50ms (parallel) |
| **L3 Recovery Monitor** | Session state analysis | Track progress, detect stuck loops, flag repeated questions | ~near instant |
| **Orchestrator** | Lightweight LLM or rule-based | Evaluate all module outputs, decide: proceed normally OR issue directive | ~50-100ms |
| **L4 Response Generator** | Main LLM call | Generate response text + tool requests + task outcomes, following orchestrator's directive | ~500-800ms first token |
| **Audio Infrastructure** | Pipecat + TTS provider | Stream TTS, handle barge-in, back-channels, multilingual auto-routing | ~200-400ms first audio |

### How a Single Turn Flows

**Normal turn (80% of turns) — orchestrator passes through:**

```
Caller: "Yeah Thursday works for me."

  L1: briefing already loaded at call start  ← no new query needed
  L2: middle phase, turn 4                   ← instant
  Signal: neutral                            ← 50ms
  L3: progressing normally                   ← instant

  Orchestrator: "proceed normally"           ← adds ZERO effective latency
                                                (runs parallel with modules)

  L4 generates response with standard context + phase
  Audio infra streams TTS to caller
```

**Intervention turn (20% of turns) — orchestrator overrides:**

```
Caller: "Can Krishna just come to the meeting himself?"

  L1: authority rules say schedule = needs consent  ← already loaded
  L2: middle phase, turn 6                          ← instant
  Signal: request detected                          ← 50ms
  L3: normal                                        ← instant

  Orchestrator: "INTERVENE — consent required.      ← overrides default
                 Bridge the conversation while
                 requesting owner consent."

  L4 generates: "That's a good idea actually.
                 Let me check Krishna's schedule..."
  Action: consent_request() tool dispatched
  Audio infra streams TTS
```

**Emotional intervention turn:**

```
Caller: "Look, I already sent that on WhatsApp,
         I don't know why I keep getting asked about this."

  L1: briefing summary confirms WhatsApp yesterday   ← already loaded
      (no on-demand retrieval needed)
  L2: middle phase, turn 5                           ← instant
  Signal: frustration HIGH, repeat-request detected  ← 50ms
  L3: we asked for something they already provided   ← flagged

  Orchestrator: "INTERVENE — our mistake. Apologize. ← overrides default
                 Acknowledge WhatsApp. Move on."

  L4 generates: "Oh, you're right, I apologize —
                 I can see you sent that over on
                 WhatsApp yesterday. That's my mistake.
                 We have it, thank you."
  Audio infra streams TTS (ElevenLabs reads the
     empathetic word choice + phrasing as soft tone
     naturally — no per-turn TTS parameters needed)
```

### How Each Layer Contributes to Response Generation — End-to-End Trace

The 4 layers are not 4 separate LLM calls. L1, L2, L3 produce **inputs** to one LLM call. L4 **is** the LLM call. Audio infrastructure executes the result. Each layer contributes a specific piece of state, prompt content, or audio handling.

**Layer-by-layer contribution map:**

```
LAYER                         WHAT IT PRODUCES               WHEN IT RUNS
─────────────────────────────────────────────────────────────────────────
L1 Contextual Intelligence   briefing + chat memory +       PRE-LLM
                              authority + tool capabilities   (loaded at call
                              + on-demand retrievals          start; refreshed
                                                              per turn if tool
                                                              fired)

L2 Conversation Flow         current phase, turn count,     PRE-LLM
                              objective status               (parallel module)

L3 Recovery & Repair         recovery directive             PRE-LLM
                              (only when needed)             (parallel module)

Orchestrator                  final directive that wraps     PRE-LLM
                              all module outputs into a      (after parallel
                              single instruction             modules complete)

L4 Response Intelligence     the LLM call itself +           THE LLM CALL
                              conversation-focused prompt +
                              structured output

Audio Infrastructure         streaming TTS + back-channels  POST-LLM
(Pipecat + TTS provider)     + barge-in + multilingual
                              auto-routing
```

**End-to-end trace of a single turn — concrete example:**

> Caller (turn 5): "Look, I already sent that on WhatsApp yesterday."

**Step 1 — STT to LangGraph.** Pipecat finalizes the transcript. The LangGraph agent receives `user_message`, `caller_language`, `turn_number`, `call_id`.

**Step 2 — Parallel modules fire simultaneously** (total time ≈ slowest module, ~50–100ms):

```
L1 Context (briefing already loaded at call start):
  chat_memory: "Owner asked: Follow up with Neha on vendor pricing"
  briefing: "Last 24h: WhatsApp with Neha re: vendor pricing.
             She sent ABC Corp + Delta. 4 vendors still pending."
  authority_rules: { autonomous: [info gathering],
                     needs_consent: [scheduling, pricing commitments] }
  task_handoff_available: true   // L4 knows it can describe any
                                 // task it can't do itself; the
                                 // Application Agent will handle it

L2 Phase Tracker:
  phase: "middle", turn_in_phase: 4,
  objective_status: "in_progress"

Signal Detector:
  frustration: "HIGH",
  repeat_request_signal: true,    // "I already sent"

L3 Recovery Monitor:
  repeat_question_by_agent: true, // we asked something already answered
  stuck_loop: false
```

**Step 3 — Orchestrator synthesizes** (~50ms):

```
Orchestrator sees:
  - Phase: middle, objective in_progress
  - Context: briefing confirms Neha sent the pricing yesterday
  - Signals: frustration HIGH + repeat_request_signal
  - Recovery: WE asked something already answered

Orchestrator decision (NOT proceed-normally):
  directive: "INTERVENE"
  instruction: "Our mistake — caller already provided this on WhatsApp
                yesterday. Apologize sincerely, acknowledge the WhatsApp,
                move past this topic."
```

**Step 4 — L4 (VCI Node) assembles the LLM call.** All upstream layer outputs converge into one prompt + state:

```
SYSTEM PROMPT (L4 — conversation-focused prompt):
  "You are calling on behalf of Krishna. Warm, natural-sounding.
   Acknowledge before responding. Detect hedges. Match caller's energy.
   Respond in the same language the caller is speaking.
   Output structured JSON with response, tool_requests, task_handoffs,
   task_outcome, phase_transition."

  + L3 repair patterns embedded:
    "When you've made a mistake or asked something already answered,
     apologize directly and move past it."

CONVERSATION HISTORY (L1 in-call memory):
  [last 5–15 turns — kept verbatim, older turns compacted]

CONTEXT BLOCK (L1 — injected as structured section):
  ## Why this call: Owner asked: Follow up with Neha on vendor pricing
  ## Briefing: WhatsApp yesterday — Neha sent ABC + Delta, 4 pending
  ## Authority: Can gather info; needs consent for pricing/scheduling
  ## Help: If you need anything you can't do from this context,
          describe it as a task_handoff and the main agent will
          handle it. You don't need to know which skill executes it.

PHASE STATE (L2):
  "Currently in MIDDLE phase, turn 4. Objective in progress."

ORCHESTRATOR DIRECTIVE (this turn's override):
  "INTERVENE: Caller is correct — already provided on WhatsApp yesterday.
   Apologize, acknowledge the WhatsApp, move on."

USER MESSAGE:
  "Look, I already sent that on WhatsApp yesterday."
```

**Step 5 — L4 LLM call** (one Claude Sonnet call, ~500–800ms first token):

```json
{
  "response": "You're right, I apologize — I can see you sent that over
               on WhatsApp yesterday. That's my mistake, we have it.
               Was there anything else on the vendor side?",
  "tool_requests": [],
  "task_handoffs": [],
  "task_outcome": null,
  "phase_transition": null
}
```

The LLM's empathetic tone is carried by **word choice and phrasing** ("You're right", "I apologize", "my mistake") — not by per-turn TTS parameters. Modern TTS reads this naturally.

**Step 6 — Audio infrastructure executes:**
- Pipecat streams the response text to ElevenLabs multilingual v2 TTS
- TTS auto-detects language (English here) from the response text
- Audio streams in owner's cloned voice (configured at call start)
- Barge-in armed via Silero VAD — if caller starts talking, TTS cancels immediately
- Back-channels disabled while agent is speaking

Caller hears the apology.

**Step 7 — State updates for next turn:**
- L2: phase stays middle, turn counter → 5
- L1: in-call message history appends this turn
- L3: repeat_question_by_agent flag cleared (acknowledged the mistake)
- Signal Detector outputs logged for next turn's recovery monitor

**Layer scorecard for this single response:**

| Layer | Concrete contribution to this turn |
|---|---|
| L1 Contextual Intelligence | Briefing already had "WhatsApp yesterday — Neha sent pricing" — without this, LLM cannot validate caller's claim |
| L2 Conversation Flow | "We're in middle phase, turn 4" — told the LLM not to wrap up yet |
| L3 Recovery & Repair | Flagged "we asked something already answered" + repair pattern in prompt |
| Orchestrator | Combined all signals → "this is OUR mistake, apologize" directive |
| L4 Response Intelligence | The LLM + conversation-focused prompt + structured output — produced the actual words with empathetic tone via word choice |
| Audio Infrastructure | Streamed TTS audio in owner's voice, language auto-detected from text; armed barge-in |

**The two modes — normal vs. intervention:**

- **Normal turn (~80%):** Orchestrator says "proceed normally." L4 LLM gets prompt + L1 context + L2 phase + history. No override. LLM generates a response with its natural intelligence using context within current phase. Audio infrastructure executes.

- **Intervention turn (~20%):** Orchestrator detects something — frustration, consent needed, recovery flag, phase transition due. It writes a directive that overrides parts of the default behavior. L4 LLM follows the directive while still using its craft, context, and phase awareness. Audio infrastructure executes.

**Where each layer lives in code:**

```
LAYER                    PRE-LLM             DURING LLM         POST-LLM
─────────────────────────────────────────────────────────────────────────
L1 Context              Context node        In prompt          (state update)
L2 Phase                Phase node          In prompt          Phase update
L3 Recovery             Recovery node       In prompt          State update
Orchestrator            Directive node      In prompt
L4 Response                                 The LLM + prompt
(Audio Infra)                                                  TTS, back-channel,
                                                               barge-in
```

L4 **is** the LLM call. L1, L2, L3, and the orchestrator all prepare state for the LLM or contribute content to its prompt. Audio infrastructure (Pipecat + TTS provider) executes the result; not a numbered intelligence layer.

**The dynamic in one sentence:** L1/L2/L3 run as parallel pre-processors that prepare the LLM's prompt and state, the orchestrator synthesizes their outputs into a directive, L4 makes one LLM call with everything assembled, and audio infrastructure executes the audio result back to the caller.

---

### Orchestrator Design Principles

1. **Lightweight, not generative** — The orchestrator classifies and routes. It doesn't write responses. This keeps it fast (~50-100ms).
2. **Parallel execution** — Specialized modules run in parallel with each other. The orchestrator waits for all, then decides. Total module time ≈ slowest module (~100ms), not sum of all.
3. **Default is pass-through** — On 80% of turns, the orchestrator says "proceed normally" and L4 generates with standard context. No latency penalty.
4. **Intervention is explicit** — When the orchestrator intervenes, it passes a clear directive to L4: what to do, what action. L4 follows the directive.
5. **Only the orchestrator can trigger:** consent requests, phase transitions, escalation to owner, graceful abort, mode switches (empathetic, urgent, formal).

### LangGraph State Graph

This maps naturally to LangGraph's supervisor pattern:

```
              ┌──────────┐
              │  START    │
              │ (STT in) │
              └─────┬────┘
                    │
          ┌─────── │ ───────┐
          │   PARALLEL FAN  │
          │     OUT         │
     ┌────▼──┐ ┌──▼──┐ ┌──▼────┐ ┌──▼────┐
     │L1     │ │L2   │ │Signal │ │L3     │
     │Context│ │Phase│ │Detect.│ │Recov. │
     │ Node  │ │Node │ │ Node  │ │ Node  │
     └────┬──┘ └──┬──┘ └──┬────┘ └──┬────┘
          │       │       │         │
          └───────┴───┬───┴─────────┘
                      │
               ┌──────▼───────┐
               │ ORCHESTRATOR │
               │    Node      │
               └──┬────────┬──┘
                  │        │
         proceed  │        │  intervene
                  │        │
             ┌────▼──┐ ┌──▼───────┐
             │ L4    │ │ L4 with  │
             │Normal │ │Directive │
             └────┬──┘ └──┬───────┘
                  │       │
                  └───┬───┘
                      │
               ┌──────▼───────┐
               │  Audio Infra │
               │  TTS + Send  │
               └──────────────┘
```

### Consent Request Flow Through the Architecture

Consent triggering spans multiple modules but the orchestrator owns the decision:

| Step | Module | What It Does |
|---|---|---|
| Know the authority rules | **L1 Context** | Injects what needs consent vs. what's autonomous |
| Detect the moment | **Signal Detector** | Flags "schedule commitment requested" |
| Decide to intervene | **Orchestrator** | Evaluates signal + authority rules → "consent required" |
| Hold the conversation | **L4 + L2** | Natural bridging while waiting ("Let me check...") |
| Handle timeout/denial | **L3 Recovery** | Graceful fallback if consent doesn't arrive |

---

## Part 3: What Trybo Has Today — Layer by Layer

### Layer 1: Contextual Intelligence — WEAK

**Outbound context sources:**

| Requirement | Status | What Exists |
|---|---|---|
| Chat history (owner's chat UI conversation) | Not done | Calling Agent only receives bare `task_text` string ("ask about school visit report"). Doesn't receive the chat messages that triggered this call — no WHY, no urgency, no owner's reasoning |
| Cross-channel briefing summary | Not done | `search_transcripts_by_contact()` loads last 3 CALL transcripts (line ~115 in conversation_service.py). But no WhatsApp thread summaries. Only same-channel raw transcripts, no briefing summarization |
| On-demand specific retrieval | Not done | No vector store, no retrieval tool — agent cannot fetch specifics on demand |
| Behavioral profile | Not done | No contact behavioral patterns available |

**Inbound context sources:**

| Requirement | Status | What Exists |
|---|---|---|
| Cross-channel briefing summary | Not done | Same as outbound — only prior call transcripts, no WhatsApp threads, no briefing |
| On-demand specific retrieval | Not done | No retrieval mechanism |
| Behavioral profile | Not done | No contact behavioral patterns available |

**Shared (both inbound and outbound):**

| Requirement | Status | What Exists |
|---|---|---|
| Relationship awareness | Minimal | Contact name + phone available. No interaction count, no "last spoke 2 days ago", no relationship depth |
| In-call conversation memory | Weak | Only last 6 turns retained (`recent_history_max: 6`) — forgets what was said 3 exchanges ago within the same call |
| Cross-channel reference handling | Not done | When contact says "like I told you on WhatsApp" or "I already sent that" — agent goes blank because it has no WhatsApp history |

**Assessment:** The Calling Agent operates nearly context-blind in both directions. For outbound, it has a bare task string — no chat history, no urgency, no owner reasoning. For inbound, it has no cross-channel context — a contact calling back after a WhatsApp thread gets treated as a cold first-time caller. The existing `search_transcripts_by_contact()` only loads prior CALL transcripts, missing the entire WhatsApp interaction history. The agent cannot maintain relationship continuity across channels.

---

### Layer 2: Conversation Architecture — MISSING

| Requirement | Status | What Exists |
|---|---|---|
| Opening phase (greeting, rapport, timing check) | Not done | Generic intro: "Hi, calling on behalf of X about Y." No "is now a good time?" |
| Middle phase (flexible exchange) | Partial | Turn-by-turn LLM responses exist but are stateless — no phase awareness |
| Closing phase (natural wrap-up) | Not done | Keyword-matched "goodbye" → immediate disconnect. No gradual closing |
| Topic management | Not done | Agent can't track multiple topics or handle topic switches |
| Agenda flexibility | Not done | Agent rigidly pursues task_text. Can't adapt to caller's agenda |
| Phase state tracking | Not done | No `session["phase"]`. Every turn treated identically |

**Assessment:** This entire layer is missing. There is no concept of conversation phases. The call is a flat sequence of turns from start to keyword-triggered disconnect.

---

### Layer 3: Recovery & Repair — WEAK

| Requirement | Status | What Exists |
|---|---|---|
| Misunderstanding repair | Not done | No repair strategy in prompts. Agent can't say "sorry, I meant..." |
| Off-topic handling | Weak | Prompt says "acknowledge briefly, redirect to task" — robotic redirect |
| Objection navigation | Not done | No objection-handling strategies in prompts |
| Unknown territory | Partial | Prompt says "I don't have that information" — but phrasing is stiff |
| Technical failure recovery | Partial | Fallback responses exist but are templated and robotic |
| Graceful abort | Not done | No "maybe Krishna should follow up directly" pattern |

**Assessment:** Recovery exists at a basic level (fallback responses, generic "I don't know" handling) but sounds scripted. No genuine repair strategies.

---

### Layer 4: Response Design & Conversation Intelligence — WEAK

**Response craft:**

| Requirement | Status | What Exists |
|---|---|---|
| Acknowledgment before content | Not done | Agent jumps directly to response. No "I see" / "Got it" |
| Varied response length | Not done | Hard cap: 25 words outbound, 50 inbound. Every response is same length |
| Natural connectors | Not done | No "so", "by the way", "actually" — responses are isolated statements |
| Filler speech during processing | Not done | Dead silence during LLM generation. Planned in `agentic-voice-calling.md` Phase 3 |
| Pre-synthesized common phrases | Not done | Every response is live TTS. Planned in `agentic-voice-calling.md` Phase 3 |
| Appropriate detail level | Partial | LLM handles this somewhat, but word cap limits it |
| Contractions and casual language | Partial | Prompt says "use contractions" but constraint-heavy framing overrides naturalness |

**Pragmatic intelligence (LLM has it, prompt suppresses it):**

| Requirement | Status | What Exists |
|---|---|---|
| Indirect answer detection | Suppressed | LLM can detect this, but task completion is binary (done/not done) — no "deferral" or "hedge" state |
| Social signal reading | Suppressed | LLM can read these, but prompt doesn't instruct it to act on them |
| Polite refusal detection | Suppressed | Agent treats "I'll try" same as "I will" — no probe instruction |
| Commitment vs hedge detection | Suppressed | No distinction in task model. Both treated as task progress |

**Emotional intelligence (LLM has it, prompt suppresses it):**

| Requirement | Status | What Exists |
|---|---|---|
| Frustration detection + adaptation | Suppressed | LLM can detect from text, but rigid constraints prevent natural empathetic response |
| Urgency matching | Suppressed | Agent responds with same pace regardless |
| Energy matching | Suppressed | Same tone for casual and formal callers — prompt enforces uniform style |
| Empathy expression | Suppressed | Rigid word caps prevent natural empathy ("I understand that's frustrating" burns half the word budget) |
| Escalation on distress | Partial | Transfer-to-human exists (LiveKit) but only triggered by explicit request, not detected emotion |

**Assessment:** This is the biggest contributor to robotic feel AND the highest-ROI fix. The LLM already has pragmatic and emotional intelligence — the current prompt actively suppresses it with 50+ constraint lines and rigid word caps. The fix is primarily prompt redesign (unleash the LLM's natural intelligence). No new services needed.

---

### Audio Infrastructure — FUNCTIONAL, GAPS IN TIMING

**Voice (provider-handled):**

| Requirement | Status | What Exists |
|---|---|---|
| Natural voice quality | Done | ElevenLabs multilingual v2, Cartesia Sonic-2, CosyVoice |
| Voice cloning (owner identity) | Done | `voice_cloning_service.py` — multi-provider support |
| Multilingual auto-detection from text | Done | ElevenLabs multilingual v2 reads language from input text |

**Timing (Pipecat addresses in Phase 1):**

| Requirement | Status | What Exists |
|---|---|---|
| Response latency (200-800ms) | Too slow | Current p90 ~3.5s |
| End-of-utterance detection | Done (silence-based) | `TurnManager` in `agent_stream_service.py`. EOS quiet gaps: 1000-2500ms |
| Barge-in handling | Done | `VOICE_BARGE_IN_ENABLED=True`, RMS threshold 300, min 150ms |
| Back-channel responses | Not done | Zero "mm-hmm" / "yeah" while caller speaks |

**Assessment:** Voice quality is strong (provider capability — ElevenLabs reads emotion from text naturally and auto-detects language). Timing is functional but slow — silence-based EOU detection (1-2.5s gaps) is the biggest latency contributor. Pipecat (Phase 1) addresses the infrastructure (Silero VAD + Smart Turn v3 + native barge-in). Only back-channel pre-synthesis is genuinely "our work" beyond Pipecat config.

---

## Part 4: Priority Map — What to Build

Based on the gap analysis, the 4 layers ranked by **impact on naturalness** and **current gap size**:

```
HIGHEST IMPACT (build first):
  ██████████ Layer 4: Response Design & Conversation Intelligence
             — prompt suppresses LLM's natural intelligence
             — empathetic tone via word choice (TTS reads from text)
             — includes pragmatic + emotional intelligence
  ████████░░ Layer 2: Conversation Architecture — entirely missing
  ███████░░░ Layer 1: Contextual Intelligence — agent is context-blind

HIGH IMPACT (build next):
  █████░░░░░ Layer 3: Recovery & Repair — prompt engineering

INFRASTRUCTURE (Phase 1 — Pipecat, not a layer):
  ██████░░░░ Audio Infrastructure — Pipecat provides VAD, turn-taking,
             streaming, barge-in. Our code adds back-channel pre-synthesis.

CROSS-CUTTING (not a layer — it's the runtime architecture):
  LangGraph agent with VCI Node + Tool Executor + Compaction nodes
  Runs on every turn, coordinates all layers
```

---

## Part 5: Requirements for Each Layer

### Layer 1 Requirements (Contextual Intelligence)

**Outbound calls — three context sources:**

1. **Chat memory handoff** — Application Agent passes the owner's original chat message, planner reasoning, and urgency indicators to the Calling Agent. Not just `task_text` but the full intent. This is what tells the agent WHY this call is being made. Example: owner typed "Call John urgently, board meeting is tomorrow and I need his report" → the agent gets the urgency and the deadline, not just "ask about report."

2. **Cross-channel briefing summary (loaded once at call start)** — At call start, load the recent cross-channel transcripts (last 1-2 interactions across calls + WhatsApp) and **immediately summarize them into a compact briefing (~150 tokens)**. This briefing — not the raw transcripts — rides in every turn's prompt. It carries relationship continuity ("we discussed vendor pricing on WhatsApp yesterday; Neha sent ABC Corp + Delta pricing") without bloating the per-turn token budget. Raw transcripts stay in state / vector store for on-demand retrieval (see #6 below).

3. **Context-aware intro generation** — Combine chat memory + briefing summary into a natural intro. "Hi John, this is calling for Krishna. He's preparing for the board meeting tomorrow and needs the school visit report. Were you able to put that together?" — not "Hi, calling on behalf of Krishna about the school visit report."

**Inbound calls — two context sources:**

4. **Cross-channel briefing summary (loaded once at call start)** — Even more critical for inbound. Same mechanism as outbound: load recent cross-channel transcripts at call start, summarize into ~150-token briefing, ride in every turn's prompt. If John was WhatsApp'd yesterday about the report and is now calling back, the briefing carries that context: agent can immediately say "Hi John! Are you calling about the report we messaged about yesterday?" — not "How can I help you?" from scratch.

5. **Open task awareness** — Check if there's an active `bot_task` for this contact. If the contact has an in-progress WhatsApp task about "vendor pricing," the agent should proactively connect the inbound call to that task: "Hi Neha, thanks for calling. Were you able to get the remaining vendor pricing from the PO system?"

**Both inbound and outbound:**

6. **On-demand specifics via `task_handoff`** — When the caller references a specific detail not in the briefing summary ("the price you quoted Tuesday", "the file I sent last week"), L4 emits a `task_handoff` describing what it needs in natural language ("Find what Krishna and Neha discussed about steel pricing last Tuesday"). The Application Agent decides which skill to invoke (transcript search, vector retrieval, etc.) and the result lands in shared state as a completed task. L4 reads it on the next turn and weaves it into the conversation. L4 does **not** know vector stores, skill names, or retrieval mechanisms — it only describes what it needs.

7. **Expand in-call memory** — `recent_history_max` from 6 → 15-20. The agent must not forget what was said earlier in the same call.

8. **Cross-channel reference handling** — When the contact says "I already told you on WhatsApp" or "like I mentioned last time," the briefing summary already gives the agent enough context to engage. Specifics come from a `task_handoff` to the Application Agent if needed.

9. **Zero tool awareness — L4 only knows `task_handoff`.**

   L4 has **no awareness of Trybo's skill registry at all** — not even a curated voice-call-scoped subset. The L4 prompt does not list tools, capabilities, schemas, or skill names. The only action L4 knows besides generating a response is:

   ```
   task_handoff(description): describe in natural language any task you
   need done that you cannot do yourself from the context you have. The
   main Application Agent will figure out which skill to invoke.
   ```

   Examples L4 might emit:
   - `"Find the current Mumbai steel rate"`
   - `"Look up what Krishna and Neha discussed about pricing last Tuesday"`
   - `"Check Krishna's calendar for Friday afternoon"`
   - `"Get permission from Krishna to share the vendor list"`
   - `"Send Neha a WhatsApp tomorrow morning with the remaining vendor pricing"`

   L4 never knows whether these become `web_research`, `transcript_search`, `calendar_lookup`, `consent_request`, `whatsapp_send`, or any other skill. The Application Agent owns that routing entirely.

   ```
   not_available (LLM declines honestly when something is clearly
   beyond the agent's reach):
     - bank/email access, payment execution, medical/legal records
   ```

   If the caller asks for something the agent shouldn't even attempt, L4 declines: "I'm not able to access that. I can have Krishna get back to you on that."

   **Why zero tool awareness matters:**
   - L4 prompt is stable forever — adding/changing/removing skills in Trybo never touches the voice-call prompt
   - L4 prompt is much smaller — no tool schemas, no capability lists, no skill descriptions (~150 tokens saved)
   - Skill orchestration lives in one place (Application Agent), not duplicated in the voice pipeline
   - No fast/slow lane classification needed — L4 doesn't decide what kind of task it is, it just describes the need

10. **Context compaction** — As conversation grows beyond a token threshold, older turns are summarized into a compact summary (~50-100 tokens) while recent 5 turns are kept verbatim. This runs as a parallel LangGraph Compaction node — L4 keeps talking while compaction happens in the background. Keeps per-turn LLM latency stable over long calls.

11. **Two-tier prompt composition (briefing + on-demand retrieval)** — The static context loaded at call start is split by token cost:

    | Static item | Size | Strategy |
    |---|---|---|
    | Chat memory (why this call) | ~150 tok | Rides every turn (small, always relevant) |
    | Authority rules | ~80 tok | Rides every turn |
    | Behavioral profile | ~80 tok | Rides every turn |
    | `task_handoff` instruction | ~50 tok | Rides every turn (one generic capability, not a tool list) |
    | **Cross-channel briefing summary** | **~150 tok** | **Summarized at call start; rides every turn** |
    | **Raw cross-channel transcripts** | **~1,000-2,000 tok** | **NOT in prompt — held outside the voice pipeline; retrieved via `task_handoff` to Application Agent when caller references specifics** |

    The full per-turn prompt structure becomes:

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

### Layer 2 Requirements (Conversation Architecture)

1. **Phase tracking in session** — Simple state: `opening` → `middle` → `closing`. Phase changes the prompt directive for each turn.

2. **Opening phase behavior** — Always include "is now a good time?" before diving into the objective. 1-2 turns of rapport.

3. **Closing phase behavior** — When objective is met, don't immediately disconnect. Transition to closing: summarize, confirm next steps, natural goodbye over 1-2 turns.

4. **Replace keyword-based disconnect** — Current `DISCONNECT_PHRASES` list triggers immediate call end. Replace with LLM-interpreted closing intent, with gradual wind-down.

5. **Expand conversation history** — `recent_history_max` from 6 → 15-20. The agent must not forget what was said 3 exchanges ago.

### Layer 3 Requirements (Recovery & Repair)

1. **Repair strategies in prompt** — How to handle misunderstandings, off-topic tangents, objections. Not as rules ("acknowledge briefly, redirect") but as natural conversation patterns.

2. **Graceful abort path** — "Maybe it would be easier if Krishna followed up directly" — for when the conversation is going nowhere. Triggered by the orchestrator when the L3 Recovery Monitor flags 3+ turns with no progress.

3. **Escalation awareness** — The orchestrator decides when to escalate (not the response generator). L3 monitor provides the signal, orchestrator decides, L4 generates the appropriate transition language.

### Layer 4 Requirements (Response Design & Conversation Intelligence) — HIGHEST PRIORITY

**Response craft — fix the robotic feel:**

1. **Remove rigid word/token caps** — Replace `max_tokens=64` and "max 25 words" with flexible guidance. Let the LLM self-regulate length based on the moment.

2. **Add acknowledgment patterns to prompt** — Instruct the LLM to always acknowledge before responding. "I see", "Got it", "Right", "Makes sense" — these 2-word prefixes add 50ms of TTS but change the entire feel.

3. **Replace constraint-focused prompts with conversation-focused prompts** — Instead of telling the LLM what NOT to do (50+ constraint lines), tell it HOW to talk. Show the conversational style through description and examples.

4. **Remove hardcoded fallback responses** — "Thanks for taking my call" and "That's a great question about {task_text}" must go. Replace with a re-prompt to the LLM or a single generic "Sorry, I missed that."

5. **Add natural connectors to prompt guidance** — "So", "By the way", "Actually", "Speaking of which" — teach the LLM to bridge between topics instead of delivering isolated statements.

**Pragmatic intelligence — unleash what the LLM already knows:**

6. **Prompt the LLM to interpret, not just transcribe** — Instruct it to distinguish commitments from hedges, real answers from deflections, genuine interest from polite refusal.

7. **Structured outcome awareness** — Instead of binary "done/not done", the agent should recognize: committed, deferred ("I'll get back to you"), blocked ("I need to check with my boss"), refused ("I don't think I can help with that"). This is a small task model change, not a new intelligence layer.

**Emotional intelligence — let the LLM adapt naturally:**

8. **Remove constraints that prevent empathy** — Rigid word caps prevent natural empathetic responses. "I understand that's frustrating" burns half the 25-word budget. Remove the cap, and the LLM naturally expresses empathy when appropriate. The empathetic tone is delivered through **word choice and phrasing** — modern TTS reads emotion from text without needing per-turn rate/energy parameters.

9. **Owner notification on critical signals** — Frustrated 2+ turns, caller asks for owner, critical urgency. The LLM detects these through prompt instruction; the orchestrator triggers the notification.

10. **Multilingual response** — Prompt instructs the LLM to respond in the same language as the caller. The LLM emits text in that language; ElevenLabs multilingual v2 TTS auto-detects the language from the response text and speaks it. No explicit language directive in the LLM output — the text itself is the directive.

### Audio Infrastructure Requirements (not a numbered layer)

**Pipecat provides (not our code):** Silero VAD, Smart Turn v3 (EOU), barge-in handling, streaming STT/TTS, multilingual transport routing.

**Our code adds:**

1. **Back-channel pre-synthesis** — Pre-synthesized "mm-hmm" / "yeah" audio clips played without interrupting the caller's turn. Triggered by Pipecat's VAD detecting long mid-utterance pauses. Context-blind by design (semantically empty), invisible to the L1 conversation history (NOT logged into the message store sent to the LLM).

2. **TTS provider configuration per call** — voice cloning ID, multilingual model selection, streaming buffer tuning. Set once at call start from `voice_cloning_service.resolve_call_voice_config()`. Not per-turn.

### Orchestrator Requirements (Cross-cutting)

1. **Lightweight classification, not generation** — The orchestrator evaluates module outputs and issues a directive (proceed / intervene + instructions). It does NOT generate response text. Target: ~50-100ms.

2. **Authority boundary awareness** — Receives authority rules from L1 Context, receives request signals from Signal Detector. When a request crosses an authority boundary, triggers consent request flow.

3. **Phase transition authority** — Only the orchestrator can transition conversation phases (opening → middle → closing). L2 Phase Tracker reports current state; orchestrator decides transitions.

4. **Emotional override** — When Signal Detector flags high frustration or urgency, orchestrator directs L4 to adapt tone through word choice and phrasing.

5. **Default pass-through** — On ~80% of turns, orchestrator says "proceed normally." Must add near-zero latency on these turns (runs in parallel with module evaluation).

### LangGraph Runtime Architecture

The VCI layers run as a **LangGraph agent** inside Pipecat's pipeline. Three nodes:

**L4 Node (the brain — single LLM call per turn):**
- L1, L2, L3 outputs converge into the prompt — context (L1), phase awareness (L2), recovery (L3) all influence the LLM's input
- Decides when it needs help from the main Application Agent
- Generates response + task_handoffs + outcomes in ONE LLM call
- L4 has **no tool awareness** — it never knows which skill executes a task_handoff
- Sees what task_handoffs are running and completed (from shared state, by description)
- Keeps talking while task_handoffs run in background — natural openings stream from the LLM
- Delivers task results naturally when they arrive

**Task Dispatcher Node (dumb worker — no LLM):**
- Runs in PARALLEL with L4 node — caller never waits for task execution
- Picks up `task_handoffs` from shared state
- Forwards each natural-language description to the **main Application Agent**
- Application Agent routes the description to the appropriate skill in Trybo's full skill registry (web_research, transcript_search, calendar_lookup, consent_request, whatsapp_send, payment, file_extraction, etc.)
- Writes results back to shared state as `completed_tasks`
- L4 reads results on next turn (by description, not by skill name) and weaves them into conversation naturally
- Multiple task_handoffs can run concurrently

**Compaction Node (dumb worker — no LLM):**
- Runs in PARALLEL when conversation context exceeds token threshold
- Summarizes older turns into compact summary (~50-100 tokens)
- Keeps recent 5 turns verbatim, drops used tool results
- Writes compacted context back to shared state
- Keeps per-turn LLM latency stable over long calls

**The L4 LLM outputs structured data — one call does everything:**

```json
{
    "response": "Let me look that up for you. In the meantime, anything else on the vendor side?",
    "task_handoffs": [
        // L4 describes what it needs in natural language; the main
        // Application Agent decides which skill to invoke. L4 does
        // NOT know which skill executes this.
        { "description": "Find the current Mumbai steel market rate" }
    ],
    "task_outcome": null,
    "phase_transition": null
}
```

No `tool_requests` field. No tool names, no skill schemas. The only thing L4 emits besides the response and outcomes is a list of natural-language task descriptions for the Application Agent to route.

The response text **is** the language directive — the LLM emits the response in whatever language the caller is speaking, and ElevenLabs multilingual v2 TTS auto-detects from the text. No `language` field in the output.

**Latency impact: ~10-20ms** per turn for LangGraph state read/write and node routing. Negligible compared to LLM latency (500-800ms). No extra LLM calls — the Tool Executor and Compaction nodes don't use LLMs.

### Multilingual Support

Auto-language matching is built into the pipeline — no explicit language directive flows from the LLM:

1. **STT (ElevenLabs Scribe)** — auto-detects caller's language from speech, transcribes in that language
2. **L4 LLM prompt** — "Respond in the same language the caller is speaking. If they mix Hindi and English, respond in the same mix."
3. **L4 LLM output** — emits response text in the same language as the input transcript (no explicit language tag)
4. **TTS (ElevenLabs multilingual v2)** — auto-detects language from the response text characters, speaks in that language using the same cloned voice

No translation service, no language directive in the LLM's structured output. The text carries the language; STT detects it on the way in; the LLM mirrors it; TTS reads it on the way out. Hindi, English, Hinglish (mixed) all work natively. If the caller switches language mid-call, the pipeline adapts automatically because the next transcript will be in the new language and the LLM will mirror it.

---

## Part 6: What Trybo Has vs What's Needed — Summary

| Component | Industry Requirement | Trybo Status | Gap |
|---|---|---|---|
| **L1: Context** | Chat history (outbound), cross-channel briefing summary (both), open task awareness (inbound), on-demand specific retrieval, in-call memory | Weak — bare task_text string, only same-channel transcripts, 6-turn history, no WhatsApp awareness, no retrieval mechanism | Chat memory handoff + briefing-summary-at-call-start + raw transcripts in vector store + `transcript_retrieval` tool for on-demand specifics + in-call history expansion |
| **L2: Architecture** | Opening/middle/closing phases, flexible agenda | Entirely missing — flat turn sequence | Full build needed |
| **L3: Recovery** | Misunderstanding repair, objection handling, graceful abort | Weak — robotic fallbacks, rigid redirect rules | Prompt redesign |
| **L4: Response + Intelligence** | Acknowledgments, varied length, connectors, pragmatic reading, emotional adaptation through word choice, multilingual response | Wrong approach — rigid caps, constraint prompts suppress LLM's natural intelligence | Complete prompt redesign + task outcome model expansion |
| **Audio Infrastructure** (not a layer) | Turn-taking, barge-in, back-channels, streaming TTS, multilingual auto-routing | Voice strong (ElevenLabs); timing functional but slow (p90 3.5s); no back-channels | Pipecat provides VAD/EOU/barge-in/streaming (Phase 1). Our code adds back-channel pre-synthesis. |
| **LangGraph VCI Agent** | L4 node (single LLM, all intelligence), Tool Executor (parallel background tasks), Compaction (context management) | Does not exist — single monolithic prompt, no tool calling, no parallel processing | Build as LangGraph agent inside Pipecat pipeline — 3 nodes, 1 LLM call per turn, parallel tool execution |
| **Task delegation (single lane)** | L4 has zero tool awareness — describes anything it needs as natural-language `task_handoff` to the main Application Agent, which owns all skill routing | Not supported — agent goes silent or puts caller on hold; no skill registry awareness; nothing to delegate to | L4 only knows `task_handoff(description)`. Application Agent routes the description to the right skill (web_research, transcript_search, calendar_lookup, consent_request, whatsapp_send, payment, file_extraction, etc.). L4 prompt is stable across all skill-registry changes. Task Dispatcher node runs in parallel with L4 — caller never waits. |
| **Multilingual** | Auto-detect caller's language, respond in same language, handle code-switching — no explicit language directive | Not supported | STT auto-detects → L4 prompt instructs language mirroring → ElevenLabs multilingual v2 TTS auto-detects from response text |
| **Context Compaction** | Rolling summarization keeps context stable over long calls | Not supported — context grows unbounded | Compaction node runs in parallel, summarizes old turns, keeps latency flat |

### The Core Problem

The current system treats voice conversation as **"LLM generates text → TTS speaks it"** — a text generation problem with audio output. But natural phone conversation is fundamentally different from text generation. It requires:

- **Knowledge** (Layer 1) — knowing the full context across channels before speaking
- **Structure** (Layer 2) — conversations have an arc — opening, middle, closing
- **Resilience** (Layer 3) — handling the unexpected gracefully
- **Intelligence** (Layer 4) — response craft + pragmatic reading + emotional adaptation through word choice, unified in one LLM
- **Coordination** (LangGraph) — L4 node orchestrates, Tool Executor runs parallel, Compaction manages context

Audio delivery (turn-taking, barge-in, back-channels, streaming TTS, multilingual auto-routing) is Pipecat + TTS-provider infrastructure — important but not an intelligence layer.

### The Architecture Shift

The fix is not just better prompts — it's a structural change from **monolithic** to **brain-like**:

```
TODAY:                              TARGET:

  STT → one big LLM call → TTS       STT → parallel modules (L1/L2/L3) →
                                     orchestrator → L4 LLM → audio infra
  
  Single prompt tries to do           Specialized modules do focused jobs.
  everything. Prompt is 50+           Orchestrator coordinates.
  constraint lines. LLM's             L4 generates responses with clear
  natural intelligence is              directives. LLM's intelligence is
  suppressed.                          unleashed, not suppressed.
```

The biggest bang-for-effort improvements are in **Layer 4 (prompt redesign)** and **Layer 2 (conversation phases)** — these are achievable primarily through prompt engineering and simple session state. The **orchestrator** is the architectural foundation that makes all layers work together without being entangled in a single monolithic prompt.
