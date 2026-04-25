# Agentic Voice Calling — Low-Latency Call Flow & Multilingual Agent Response

## Context

The platform currently handles voice calls through three telephony providers — **Twilio** (international PSTN), **Knowlarity** (India +91), and **LiveKit** (browser/app WebRTC). Each provider has its own streaming service (`agent_stream_service.py`, `knowlarity_stream_service.py`, `livekit_stream_service.py`), its own audio format, and its own WebSocket protocol. The call flow works, but it is **not agentic** — the agent reacts to individual utterances rather than managing the call as a structured conversation with goals, state, and tool execution.

This component defines **how the voice agent manages the entire call lifecycle agentically** — from greeting through conversation, tool execution, language adaptation, and wrap-up — with a target of **p50 < 800ms, p95 < 1200ms** end-to-end response latency (down from the current p90 < 3.5s target).

### What Already Exists

| What | Where | Status |
|---|---|---|
| Twilio bidirectional streaming | `agent_stream_service.py` — WebSocket, μ-law 8kHz, 20ms frames | Working |
| Knowlarity streaming | `knowlarity_stream_service.py` — PCM 16-bit 16kHz, 200ms batches | Working |
| LiveKit WebRTC sessions | `livekit_stream_service.py` — 48kHz PCM, resampled to 8kHz | Working |
| TurnManager | `agent_stream_service.py` lines 69-288 — debounce-based turn detection | Working — but too slow (1.2s debounce + 2.5s EOS quiet) |
| VAD with barge-in | `livekit_stream_service.py` lines 293-354 — RMS energy detection | Working — but thresholds are conservative |
| Cartesia ASR | `asr_cartesia.py` — WebSocket streaming, batched frames | Working |
| ElevenLabs ASR | `asr_elevenlabs.py` — streaming transcription | Working |
| Conversation service | `conversation_service.py` — intro, intent detection, response generation | Working — but stateless (no call flow state machine) |
| Voice service | `voice_service.py` — orchestrates ASR/LLM/TTS | Working |
| Call creation + status | `call_service.py`, `call_status_service.py` | Working |
| Provider selection | `telephony_factory.py` — +91 → Knowlarity, else Twilio | Working |
| LLM integration | OpenAI (gpt-4.1), Google Gemini, Ollama | Working |
| TTS providers | ElevenLabs, Cartesia, CosyVoice (150ms ultra-low latency) | Working |
| Cartesia ASR constants | `constants/cartesia.py` — all timing/threshold parameters | Working — but values too conservative |

### What's Missing

| Capability | Gap |
|---|---|
| **Call flow state machine** | No structured call states (greeting → listen → intent → action → confirm → wrap-up). Agent reacts per-utterance without session-level goals. |
| **Semantic turn detection** | Relies on silence duration (2.5s EOS quiet) instead of semantic end-of-utterance prediction. Adds 1-2s unnecessary latency. |
| **Language detection + adaptation** | No language detection on incoming speech. Agent always responds in the language of the system prompt, not the caller's language. |
| **Speculative tool calling** | Tool calls block the audio pipeline. No filler speech while tools execute. |
| **Pre-synthesized phrases** | Every response synthesized from scratch, including greetings, acknowledgments, and common phrases. |
| **Dynamic model routing** | Same LLM used for simple acknowledgments and complex reasoning. No fast-path for trivial turns. |
| **Latency-optimized thresholds** | Current constants tuned for accuracy over speed: ASR_FINAL_DEBOUNCE_SECONDS=1.2s, CARTESIA_EOS_MIN_QUIET_MS=2500ms, etc. |

### Tech Stack (Voice-Specific)

| Layer | Technology |
|---|---|
| **Telephony** | Twilio (PSTN international), Knowlarity (India), LiveKit (WebRTC) |
| **ASR** | Cartesia (Ink Whisper), ElevenLabs Scribe |
| **LLM** | OpenAI gpt-4.1 (complex), Gemini Flash 2.5 (fast), GPT-4.1 Mini (balanced) |
| **TTS** | ElevenLabs (multilingual v2), Cartesia Sonic-2, CosyVoice (ultra-low latency) |
| **Agent orchestration** | LangGraph |
| **Transport** | WebSocket (Twilio/Knowlarity), WebRTC (LiveKit) |

---

## Component Responsibility

```
┌──────────────────────────────────────────────────────────────────┐
│     THIS COMPONENT'S RESPONSIBILITY                              │
│                                                                  │
│  1. CALL FLOW STATE MACHINE                                      │
│     Define call states, transitions, per-state agent behavior    │
│     Maintain session-level goals and context                     │
│     Track what has been said, confirmed, and acted upon          │
│                                                                  │
│  2. LATENCY-OPTIMIZED TURN DETECTION                             │
│     Tune silence/debounce thresholds for speed                   │
│     Add semantic end-of-utterance prediction (Phase 2)           │
│     Sub-second turn commitment for common cases                  │
│                                                                  │
│  3. LANGUAGE DETECTION & ADAPTATION                              │
│     Detect caller's language from first utterance                │
│     Lock session language (allow override on sustained switch)   │
│     Route to multilingual TTS voice                              │
│     Instruct LLM to respond in detected language                 │
│                                                                  │
│  4. SPECULATIVE TOOL CALLING                                     │
│     Predict likely tool calls from conversation context          │
│     Execute read-only tools in parallel with filler speech       │
│     Never speculatively execute state-changing operations        │
│                                                                  │
│  5. PRE-SYNTHESIZED PHRASE CACHE                                 │
│     Cache TTS audio for greetings, acknowledgments, fillers      │
│     Zero-latency playback for cached phrases                     │
│     Per-language cache (populated on first use per language)      │
│                                                                  │
│  6. DYNAMIC MODEL ROUTING                                        │
│     Route simple turns (acks, FAQs) to fast LLM (Gemini Flash)  │
│     Route complex turns (reasoning, multi-step) to capable LLM   │
│     Intent-based routing, not random                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│     NOT THIS COMPONENT'S RESPONSIBILITY                          │
│                                                                  │
│  - Telephony provider integration (Twilio/Knowlarity/LiveKit)    │
│    → already handled by existing stream services                 │
│                                                                  │
│  - Audio codec encoding/decoding (μ-law, PCM, resampling)        │
│    → already handled by existing stream services                 │
│                                                                  │
│  - ASR provider integration (Cartesia, ElevenLabs)               │
│    → already handled by asr_cartesia.py, asr_elevenlabs.py       │
│                                                                  │
│  - TTS provider integration (ElevenLabs, Cartesia, CosyVoice)   │
│    → already handled by voice_service.py                         │
│                                                                  │
│  - Call recording and transcript persistence                     │
│    → already handled by call_recording_service.py,               │
│      transcript_service.py                                       │
│                                                                  │
│  - Chat-based agent orchestration (non-voice)                    │
│    → separate LangGraph flow, not this component                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## How It Should Work

### 1. Call Flow State Machine

Every call is managed by a state machine. Each state defines: what the agent's goal is, what tools are available, what constitutes a valid transition, and what happens on timeout or barge-in.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CALL FLOW STATE MACHINE                         │
│                                                                     │
│  ┌───────────┐                                                      │
│  │ GREETING  │  Pre-synthesized audio plays immediately             │
│  │           │  Load caller context in parallel (contact, history)  │
│  │           │  Detect language from first caller utterance          │
│  └─────┬─────┘                                                      │
│        │ caller responds / silence timeout (3s)                     │
│        ▼                                                            │
│  ┌───────────┐                                                      │
│  │  LISTEN   │  ASR active, VAD monitoring                          │
│  │           │  Semantic turn detection (is caller done speaking?)  │
│  │           │  Buffer partial transcripts                          │
│  └─────┬─────┘                                                      │
│        │ turn committed (EOS detected)                              │
│        ▼                                                            │
│  ┌───────────┐                                                      │
│  │  CLASSIFY │  Determine turn complexity:                          │
│  │           │    SIMPLE → fast LLM (Gemini Flash 2.5)             │
│  │           │    COMPLEX → capable LLM (GPT-4.1)                  │
│  │           │    TOOL_NEEDED → speculative execution + filler      │
│  │           │    TRANSFER → initiate human handoff                 │
│  │           │    END_CALL → wrap-up state                          │
│  └─────┬─────┘                                                      │
│        │                                                            │
│   ┌────┼────┬──────────┬──────────┐                                 │
│   ▼    ▼    ▼          ▼          ▼                                  │
│ ┌────┐┌──────┐┌──────────┐┌──────────┐┌─────────┐                  │
│ │RESP││TOOL  ││ TRANSFER ││ CONFIRM  ││ WRAP-UP │                  │
│ │    ││EXEC  ││ TO HUMAN ││          ││         │                  │
│ │ LLM││      ││          ││ Repeat   ││Summarize│                  │
│ │ +  ││Filler││Pre-load  ││ critical ││actions  │                  │
│ │TTS ││plays ││context   ││ info back││taken    │                  │
│ │    ││while ││for human ││ to caller││         │                  │
│ │    ││tool  ││          ││          ││         │                  │
│ │    ││runs  ││          ││          ││         │                  │
│ └──┬─┘└──┬───┘└──────────┘└────┬─────┘└────┬────┘                  │
│    │     │                     │            │                       │
│    └──┬──┘                     │            │                       │
│       ▼                        ▼            ▼                       │
│  ┌───────────┐           ┌───────────┐  ┌──────┐                   │
│  │  LISTEN   │ ◄─────────│  LISTEN   │  │ END  │                   │
│  │  (loop)   │           │  (loop)   │  │      │                   │
│  └───────────┘           └───────────┘  └──────┘                   │
│                                                                     │
│  BARGE-IN (any state except GREETING first 500ms):                  │
│    Detected user speech during agent audio playback                 │
│    → Cancel TTS stream immediately (< 200ms)                       │
│    → Clear audio output buffer                                      │
│    → Cancel pending LLM stream if still generating                  │
│    → Transition to LISTEN state                                     │
│                                                                     │
│  SILENCE TIMEOUT (configurable per state):                          │
│    LISTEN: 8s silence → agent prompts "Are you still there?"       │
│    LISTEN: 15s silence after prompt → end call                      │
│    CONFIRM: 5s silence → repeat confirmation request                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**State machine implementation lives in a new file: `call_state_machine.py`**

Each state is a dataclass:

```python
@dataclass
class CallState:
    name: str                          # "greeting", "listen", "classify", etc.
    available_tools: list[str]         # tools the agent can invoke in this state
    timeout_seconds: float             # silence timeout before transition
    timeout_action: str                # "prompt" | "repeat" | "end_call"
    barge_in_enabled: bool             # whether to handle interruptions
    llm_model: str | None              # override model for this state (or None for dynamic)
```

The state machine is instantiated per call session and persisted in the session store alongside conversation history.

### 2. Latency-Optimized Turn Detection

The biggest latency win. Current thresholds are tuned for accuracy over speed. This component tunes them for conversational responsiveness.

**Phase 1 — Threshold Tuning (immediate, no new code):**

| Parameter | Current | New | Rationale |
|---|---|---|---|
| `ASR_FINAL_DEBOUNCE_SECONDS` | 1.2s | **0.5s** | Merge finals within 500ms — sufficient for natural speech pauses |
| `CARTESIA_EOS_MIN_QUIET_MS` | 2500ms | **1000ms** | 1s silence = caller is done for most utterances |
| `CARTESIA_EOS_ONE_WORD_QUIET_MS` | 3000ms | **1200ms** | Single words ("yes", "no", "hello") don't need 3s |
| `CARTESIA_EOS_INCOMPLETE_QUIET_MS` | 6000ms | **2500ms** | Still generous for mid-thought pauses |
| `CARTESIA_ASR_VAD_HANGOVER_MS` | 500ms | **250ms** | Tighter speech boundary detection |
| `ASR_SILENCE_TAIL_S` | 0.6s | **0.3s** | Less silence suppression going to ASR |

**Expected impact:** 1-2 seconds saved per turn. Moves p50 from ~3s to ~1.5s immediately.

**Phase 2 — Semantic End-of-Utterance Model:**

Instead of waiting for silence, predict whether the caller is done speaking based on the semantic content of the partial transcript.

```
Current (silence-based):
  "I'd like to reschedule my appointment" → [1.0s silence] → committed
  "I'd like to reschedule my..." → [2.5s silence] → committed (incomplete)

With semantic EOU:
  "I'd like to reschedule my appointment" → EOU model: 0.95 confidence → committed (0ms wait)
  "I'd like to reschedule my..." → EOU model: 0.15 confidence → wait for more speech
```

**Implementation:** Use a lightweight classification model (LiveKit's 135M EOU model or a fine-tuned distilbert) that reads partial transcripts and predicts turn completion. The model runs on every ASR partial/final and outputs a confidence score. If confidence > threshold (0.85), commit the turn immediately without waiting for silence.

**Fallback:** If the EOU model is unavailable or confidence is ambiguous (0.4-0.85), fall back to the silence-based thresholds from Phase 1.

### 3. Language Detection & Adaptation

The agent must respond in the caller's language without being told to switch. This is critical for India (Hindi, Tamil, Telugu, Marathi, Bengali, Kannada, etc.) and international calls.

**Detection flow:**

```
Call starts → GREETING state plays in default language (configurable per task)
                    │
Caller responds → First ASR result arrives
                    │
                    ▼
         ┌─────────────────────────┐
         │  LANGUAGE DETECTION     │
         │                         │
         │  Source 1: ASR provider  │
         │    ElevenLabs Scribe:   │
         │    language_code=null    │
         │    → auto-detect        │
         │    → returns detected   │
         │      language code      │
         │                         │
         │  Source 2: LLM fallback │
         │    If ASR doesn't       │
         │    return language:      │
         │    lightweight LLM call │
         │    "What language is    │
         │     this text in?"      │
         └────────┬────────────────┘
                  │
                  ▼
         ┌─────────────────────────┐
         │  SESSION LANGUAGE LOCK  │
         │                         │
         │  session.language =     │
         │    detected_language    │
         │                         │
         │  Lock rules:            │
         │  - Set on first         │
         │    utterance            │
         │  - Override only if     │
         │    2+ consecutive turns │
         │    detected in a        │
         │    different language   │
         │  - Hindi-English code-  │
         │    switching: detect    │
         │    dominant language,   │
         │    respond in that      │
         │  - Confidence < 0.5:    │
         │    keep current lock    │
         └────────┬────────────────┘
                  │
                  ▼
         ┌─────────────────────────┐
         │  RESPONSE ADAPTATION    │
         │                         │
         │  1. LLM system prompt:  │
         │     "Respond entirely   │
         │      in {language}.     │
         │      Match dialect and  │
         │      formality."        │
         │                         │
         │  2. TTS voice selection:│
         │     ElevenLabs multi-   │
         │     lingual v2 voice    │
         │     (same voice ID,     │
         │     supports 29 langs)  │
         │                         │
         │  3. Pre-synth cache:    │
         │     Load language-      │
         │     specific cached     │
         │     phrases             │
         └─────────────────────────┘
```

**Hindi-English code-switching (critical for India):**

Indian callers frequently mix Hindi and English within a single sentence ("Mera appointment Thursday ko reschedule kar do please"). The system must:
1. Detect the **dominant** language (Hindi in this case — the grammar structure is Hindi)
2. Respond in the dominant language, preserving English loan words that are natural in Hindi ("appointment", "reschedule", "Thursday")
3. NOT flip-flop between languages on every turn

**For Knowlarity calls (India +91):** Default detection bias to Hindi with English fallback. This matches the most common pattern for Indian business calls.

### 4. Speculative Tool Calling

When the conversation context makes a tool call predictable, execute it **before** the LLM explicitly requests it. This hides 1.5-2 seconds of tool latency behind natural filler speech.

```
Example: Caller says "Can you check my appointment for Thursday?"

WITHOUT speculative calling:
  ASR final → LLM generates tool call → tool executes (1.5s) → LLM reads result → TTS
  Total: ~3.5s silence

WITH speculative calling:
  ASR final → Two parallel tracks:
    Track A: TTS plays filler "Let me check that for you..."
    Track B: Predict tool = check_calendar(day="Thursday") → execute immediately
  Tool result arrives → LLM incorporates result → TTS plays actual answer
  Perceived latency: ~0.5s (filler starts immediately)
```

**Rules for speculative execution:**
1. **Only read-only operations**: calendar lookups, contact searches, account checks, information retrieval
2. **Never state-changing operations**: no creating appointments, sending messages, making payments, or modifying records
3. **Confidence threshold**: only execute if the predicted tool matches the conversation pattern with > 0.8 confidence
4. **Discard on mismatch**: if the LLM's actual tool call differs from the speculated one, discard the speculative result and execute the correct call

**Implementation:** A `SpeculativeToolPredictor` that maintains a mapping of conversation patterns → likely tool calls. Initially rule-based (keyword matching), evolves to ML-based in Phase 3.

### 5. Pre-Synthesized Phrase Cache

Certain phrases appear in nearly every call. Synthesizing them from scratch every time wastes 75-100ms per phrase. Pre-synthesize and cache them.

**Cache structure:**

```
phrase_cache/
  ├── en/
  │   ├── greeting_inbound.wav        "Hi, thank you for calling. How can I help you?"
  │   ├── greeting_outbound.wav       "Hi, this is calling on behalf of {owner_name}."
  │   ├── ack_yes.wav                 "Got it."
  │   ├── ack_understood.wav          "I understand."
  │   ├── filler_checking.wav         "Let me check that for you."
  │   ├── filler_moment.wav           "One moment please."
  │   ├── confirm_repeat.wav          "Just to confirm..."
  │   ├── transfer_hold.wav           "I'm connecting you now, please hold."
  │   └── goodbye.wav                 "Thank you, goodbye."
  ├── hi/                             (Hindi)
  │   ├── greeting_inbound.wav        "Namaste, aapka call attend kar raha hoon..."
  │   ├── ack_yes.wav                 "Samajh gaya."
  │   ├── filler_checking.wav         "Ek second, check karta hoon."
  │   └── ...
  ├── ta/                             (Tamil)
  ├── te/                             (Telugu)
  └── {language_code}/                (populated on first detection)
```

**Cache population:**
- **Phase 1:** Pre-generate English and Hindi caches at deployment time
- **Phase 2:** Generate other language caches on first detection (background task, first call in new language uses live TTS, subsequent calls use cache)
- **Personalized greetings:** Outbound calls include `{owner_name}` — these are synthesized once per owner + language combination and cached

**Cache storage:** Local filesystem on the API server (phrases are small — ~50KB each at 8kHz μ-law). No need for external storage. Invalidate on TTS voice change.

### 6. Dynamic Model Routing

Not every turn needs the most capable (and slowest) LLM. Route based on turn complexity.

```
Turn committed → Classify complexity:

  SIMPLE (fast path — Gemini Flash 2.5, ~370ms TTFT):
    - Acknowledgments: "yes", "no", "okay", "sure"
    - Greetings and goodbyes
    - Single-intent questions with direct answers
    - Repeating/confirming information back
    - Detection: < 10 words, or matches common pattern

  BALANCED (medium path — GPT-4.1 Mini, ~420ms TTFT):
    - Questions requiring context lookup
    - Turns that need tool calling
    - Multi-part responses
    - Detection: 10-50 words, or contains a question with context

  COMPLEX (full path — GPT-4.1, ~660ms TTFT):
    - Multi-step reasoning
    - Ambiguous intent requiring clarification
    - Negotiation or persuasion
    - Detection: > 50 words, or multiple intents, or requires reasoning

  All models receive the same conversation history and system prompt.
  The routing decision is made by a lightweight classifier (rule-based Phase 1,
  fine-tuned classifier Phase 3).
```

---

## What Exists Today vs. What's New

| Capability | Exists Today | What's New |
|---|---|---|
| Call flow management | Per-utterance reactive (`conversation_service.py`) | State machine with defined states, transitions, timeouts |
| Turn detection | Silence-based (TurnManager in `agent_stream_service.py`) | Tuned thresholds (Phase 1) + semantic EOU model (Phase 2) |
| Language detection | Not implemented | ASR-based detection, session lock, LLM prompt adaptation |
| Multilingual TTS | ElevenLabs multilingual v2 voices exist but not language-routed | Dynamic voice/language routing based on detected language |
| Speculative tool calling | Not implemented | Parallel filler + tool execution for read-only operations |
| Pre-synthesized phrases | Not implemented | Cached common phrases per language, zero-latency playback |
| Dynamic model routing | Single LLM per call | Per-turn routing: Gemini Flash / GPT-4.1 Mini / GPT-4.1 |
| Barge-in handling | Basic RMS detection (`VOICE_BARGE_IN_RMS_THRESHOLD`) | < 200ms cancel of TTS + LLM stream + buffer clear |
| Latency thresholds | Conservative (2.5s EOS, 1.2s debounce) | Aggressive (1.0s EOS, 0.5s debounce) |

---

## Key Design Decisions

### 1. State Machine Lives Alongside Existing Stream Services, Not Inside Them

**Decision:** The call state machine is a **new service** (`call_state_machine.py`) that the existing stream services (`agent_stream_service.py`, `livekit_stream_service.py`, `knowlarity_stream_service.py`) call into. The stream services continue to own audio I/O. The state machine owns conversation flow logic.

**Why:** The three stream services already work and handle provider-specific audio formats. Embedding call flow logic into each one would triple the maintenance burden. A single state machine service means call flow changes apply to all providers simultaneously.

### 2. Threshold Tuning Before Semantic EOU

**Decision:** Phase 1 tunes existing numerical thresholds. Phase 2 adds the semantic model. Don't wait for Phase 2 to ship latency improvements.

**Why:** Threshold tuning is a config change — zero new code, zero new dependencies, immediately testable. The semantic EOU model requires model integration, inference infrastructure, and tuning. The thresholds alone deliver 1-2s improvement. Ship the easy win first.

### 3. Language Lock on First Utterance, Not Per-Turn Detection

**Decision:** Detect language once on the first substantive utterance and lock it for the session. Only override on 2+ consecutive turns in a different language.

**Why:** Per-turn detection causes language flip-flopping, especially with Hindi-English code-switching. A caller who says "appointment Thursday ko reschedule karo" is speaking Hindi with English loan words — not switching to English. Locking prevents the agent from responding in English to a Hindi speaker just because they used an English word.

### 4. Speculative Tool Calling Limited to Read-Only Operations

**Decision:** Only speculatively execute operations that read data (calendar lookups, contact searches, account checks). Never speculatively create, update, or delete.

**Why:** A speculative read that gets discarded wastes compute but causes no harm. A speculative write that gets discarded may have already created/modified data that the caller didn't intend. The rule is simple: if the operation is idempotent and has no side effects, it's safe to speculate.

### 5. Gemini Flash 2.5 for Simple Turns, Not a Smaller Self-Hosted Model

**Decision:** Use Gemini Flash 2.5 (hosted, ~370ms TTFT, $0.30/1M tokens) for simple turns rather than self-hosting a small model.

**Why:** Self-hosting adds infrastructure complexity (GPU provisioning, model serving, autoscaling). Gemini Flash is fast enough (370ms), cheap enough ($0.30/1M), and eliminates operational burden. The latency difference between a self-hosted 7B model on Groq (~100ms) and Gemini Flash (~370ms) is 270ms — real but not worth the infrastructure cost at current scale.

### 6. Pre-Synthesized Phrases on Local Filesystem, Not External Cache

**Decision:** Store cached TTS audio on the API server's local filesystem, not Redis or Supabase Storage.

**Why:** Each phrase is ~50KB at 8kHz. Total cache for 10 phrases × 5 languages = ~2.5MB. Reading from local disk is < 1ms. Reading from Redis adds ~1-3ms network. Reading from Supabase Storage adds ~50-100ms. The cache is small enough that every API server instance can hold a full copy. Invalidation is simple: delete the cache directory on TTS voice configuration change.

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Aggressive EOS thresholds cause premature turn commits (agent responds before caller finishes) | Agent interrupts mid-sentence — poor experience | Gradual rollout: tune one parameter at a time, measure false-positive rate. Target < 5% false triggers. Phase 2 semantic EOU reduces risk further. |
| Language detection misidentifies language on short utterances ("hello", "yes") | Agent responds in wrong language | Require minimum 3 words for language detection. For < 3 words, keep default language. Lock only changes on first substantive utterance (> 3 words). |
| Hindi-English code-switching detected as English | Hindi speakers get English responses | For Knowlarity (+91) calls, bias detection toward Hindi. Require high confidence (> 0.8) to classify as English when the call originated from an Indian number. |
| Speculative tool call returns stale or wrong data | Agent speaks incorrect information before LLM validates | Speculative results are always validated by the LLM before being spoken. The filler plays while both the tool AND the LLM run. The LLM sees the tool result and decides what to say. If the speculative result is wrong, the LLM ignores it and the correct tool is called. |
| Pre-synthesized greetings sound robotic compared to contextual greetings | Less natural call opening | Use high-quality TTS voice for synthesis. Include slight prosody variation. Phase 2: generate context-aware greetings (with caller name) and cache per-caller. |
| Model routing misclassifies a complex turn as simple | Gemini Flash gives shallow response to a nuanced question | Conservative classification: when in doubt, route to capable model. Monitor response quality scores per model and adjust classification thresholds. |

---

## Implementation Priority & Time Estimates

All implementation is done using Claude. Claude writes all code and unit tests. Human time is for device testing, verification, and bug triage.

**Working day = 8 hours**

### Phase 1: Latency Threshold Tuning + Pre-Synthesized Phrases
*Dependencies: None (config changes + new cache module)*

| Task | Hours |
|------|-------|
| Tune `ASR_FINAL_DEBOUNCE_SECONDS`, `CARTESIA_EOS_MIN_QUIET_MS`, `CARTESIA_EOS_ONE_WORD_QUIET_MS`, `CARTESIA_EOS_INCOMPLETE_QUIET_MS`, `CARTESIA_ASR_VAD_HANGOVER_MS`, `ASR_SILENCE_TAIL_S` to new values | 0.5 |
| Add configurable override via environment variables (so thresholds can be A/B tested without code deploy) | 1 |
| Implement `phrase_cache.py` — TTS synthesis of common phrases, local filesystem storage, per-language directories | 2 |
| Pre-generate English and Hindi phrase caches (greeting_inbound, greeting_outbound, ack_yes, ack_understood, filler_checking, filler_moment, confirm_repeat, transfer_hold, goodbye) | 1 |
| Wire phrase cache into `voice_service.py` — check cache before calling TTS provider, fall through to live TTS on miss | 1.5 |
| Improve barge-in: cancel TTS + LLM stream + clear buffer within 200ms (tighten existing barge-in in all three stream services) | 2 |
| Integration test: measure end-to-end latency with new thresholds on test calls | 1 |
| **Phase 1 Total** | **9 hours (~1.1 days)** |

### Phase 2: Language Detection & Call State Machine
*Dependencies: Phase 1 (thresholds must be tuned before state machine can rely on fast turn commits)*

| Task | Hours |
|------|-------|
| Implement `language_detector.py` — extract language from ASR result, LLM fallback for ASR providers that don't return language, session lock logic, code-switching handling | 3 |
| Wire language detection into ASR pipeline: on first substantive utterance (> 3 words), detect and lock | 1.5 |
| Modify `conversation_service.py` system prompt generation to include language instruction | 1 |
| Implement TTS voice routing: select multilingual voice + language parameter based on session language | 1.5 |
| Wire phrase cache to load language-specific phrases on language detection | 0.5 |
| Implement `call_state_machine.py` — CallState dataclass, state transitions, per-state configuration (tools, timeout, barge-in, model) | 4 |
| Define state configurations for: GREETING, LISTEN, CLASSIFY, RESPOND, TOOL_EXEC, CONFIRM, TRANSFER, WRAP_UP, END | 2 |
| Wire state machine into `voice_service.py` — state machine drives conversation flow, stream services provide audio I/O | 3 |
| Silence timeout handling per state (LISTEN: 8s prompt, 15s end; CONFIRM: 5s repeat) | 1 |
| Unit tests: language detection (Hindi, English, Tamil, Telugu, code-switching, short utterances) | 1.5 |
| Unit tests: state machine transitions, timeout behavior, barge-in at each state | 2 |
| Integration test: full call flow with language switching on test calls | 1.5 |
| **Phase 2 Total** | **23 hours (~2.9 days)** |

### Phase 3: Dynamic Model Routing + Speculative Tool Calling
*Dependencies: Phase 2 (state machine must exist for routing and speculation to plug into)*

| Task | Hours |
|------|-------|
| Implement `turn_classifier.py` — rule-based complexity classification (SIMPLE/BALANCED/COMPLEX) based on word count, pattern matching, intent signals | 2 |
| Configure LLM routing: Gemini Flash 2.5 for SIMPLE, GPT-4.1 Mini for BALANCED, GPT-4.1 for COMPLEX | 1.5 |
| Wire routing into state machine CLASSIFY state | 1 |
| Implement `speculative_tool_predictor.py` — conversation pattern → predicted tool call mapping (rule-based) | 3 |
| Implement parallel execution: filler TTS on Track A, speculative tool on Track B, LLM incorporation of result | 3 |
| Safety gate: whitelist of read-only tools eligible for speculation, reject all others | 1 |
| Wire speculative calling into TOOL_EXEC state in the state machine | 1.5 |
| Unit tests: turn classification across 50+ sample utterances | 1.5 |
| Unit tests: speculative tool predictor (correct predictions, mismatch handling, safety gate) | 1.5 |
| Integration test: measure latency improvement with speculation on real tool calls | 1.5 |
| **Phase 3 Total** | **18 hours (~2.25 days)** |

### Phase 4: Semantic End-of-Utterance Model
*Dependencies: Phase 1 thresholds as fallback, Phase 2 state machine for integration*

| Task | Hours |
|------|-------|
| Evaluate EOU models: LiveKit 135M EOU model, distilbert fine-tuned for turn prediction | 2 |
| Integrate selected model: load on startup, run inference on every ASR partial/final | 3 |
| Implement confidence-based commit: > 0.85 → immediate commit, 0.4-0.85 → fall back to silence thresholds, < 0.4 → wait for more speech | 2 |
| Tune confidence threshold on recorded call transcripts (replay historical calls through the model) | 2 |
| Fallback handling: if model inference fails or times out (> 50ms), fall back to silence-based detection | 1 |
| A/B test framework: route 50% of calls through semantic EOU, 50% through silence-based, compare latency + false positive rate | 2 |
| Unit tests: model predictions on sample transcripts (complete sentences, incomplete, questions, single words) | 1.5 |
| Integration test: measure turn commit latency with semantic EOU on test calls | 1.5 |
| **Phase 4 Total** | **15 hours (~1.9 days)** |

---

## Total Effort

| Phase | Scope | Hours | Days |
|---|---|---|---|
| Phase 1 | Threshold tuning + phrase cache | 9 | 1.1 |
| Phase 2 | Language detection + state machine | 23 | 2.9 |
| Phase 3 | Model routing + speculative tools | 18 | 2.25 |
| Phase 4 | Semantic EOU model | 15 | 1.9 |
| **Total** | | **65** | **~8.2 days** |

---

## Expected Latency Impact

| Metric | Current | After Phase 1 | After Phase 2 | After Phase 3+4 |
|---|---|---|---|---|
| **p50 response** | ~3.0s | ~1.5s | ~1.2s | **< 800ms** |
| **p95 response** | ~5.0s | ~2.5s | ~2.0s | **< 1.2s** |
| **Turn commit** | 2.5-6s | 1.0-2.5s | 0.8-2.0s | **0.3-1.0s** |
| **Tool call turns** | ~4.5s | ~3.0s | ~2.5s | **~1.0s** (with speculation) |
| **Greeting** | ~500ms (live TTS) | **~5ms** (cached) | ~5ms | ~5ms |

---

## Provider-Specific Notes

### Twilio (International PSTN)
- PSTN adds 150-700ms of inherent transport latency — this component cannot reduce it
- The 800ms p50 target accounts for PSTN overhead on WebRTC/LiveKit calls
- For PSTN calls via Twilio, realistic target is p50 < 1.2s (800ms agent + 400ms PSTN)
- μ-law 8kHz audio quality limits ASR accuracy — speech-to-speech models lose accuracy here, confirming the cascading pipeline is the right architecture

### Knowlarity (India +91)
- Default language detection bias: Hindi
- Knowlarity's 16kHz audio quality is better than Twilio's 8kHz for ASR
- 200ms batch outbound: consider reducing to 100ms batches for lower latency (trade-off: more WebSocket messages)

### LiveKit (Browser/App WebRTC)
- Best latency path: 48kHz audio, no PSTN overhead, 60-120ms transport
- This is where the p50 < 800ms target is most achievable
- LiveKit's built-in AEC/AGC/noise suppression reduces VAD false positives
- The EOU model from LiveKit's Agents framework (Phase 4) is designed for this integration
