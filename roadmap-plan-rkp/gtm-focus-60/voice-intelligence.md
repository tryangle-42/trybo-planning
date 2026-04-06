# Voice Intelligence — Baseline Definition

> Owner: RKP
> Module mapping: M10 (Communication Skills) — voice subsystem
> Current completion: Voice infra ~44% done. Twilio, Knowlarity, WebRTC, voice cloning built. Agentic call flows, STT/TTS optimization, multilingual ASR, noise cancellation missing.

---

## What This Is

The voice pipeline that makes the agent sound human and respond in real-time during calls. Five sub-systems:

1. **Voice Synthesis & Cloning** — TTS with cloned user voice, low latency
2. **Speech Recognition & Understanding** — ASR, multilingual (English + Hindi/Hinglish), mobile voice input
3. **Conversational AI Call Intelligence** — inbound/outbound call management, intent detection during calls, handoff
4. **Real-time Voice Streaming** — the STT→LLM→TTS pipeline, latency optimization
5. **Voice UX** — hold behavior, filler injection, turn-taking, interruption handling

## Current State

| Built | Not Built |
|-------|-----------|
| Twilio outbound/inbound + AI voice agent | Agentic call flows (calls don't go through orchestration engine) |
| Knowlarity alternative telephony | STT/TTS latency optimization (no profiling, no streaming optimization) |
| WebRTC via LiveKit (browser calls) | Multilingual ASR (no Hindi/Hinglish baseline) |
| Voice cloning (ElevenLabs, CosyVoice, Cartesia, Coqui) | Noise cancellation |
| Call recording + transcription | Smart hold (template voice at intervals) |
| Media streams via WebSocket | Call intent detection separate from orchestration GID |

---

## Baseline Definition

**The line:** Voice-to-voice latency is under 800ms for English, under 1000ms for Hindi. Calls route through the agentic orchestration engine (not bypassing it). Hindi/Hinglish ASR works for conversational speech. The agent handles turn-taking, interruptions, and filler injection. Anything beyond this is refinement.

---

## Baseline Components

### 1. Voice Synthesis & Cloning

**Provider strategy:**

| Provider | Use case | Why |
|----------|---------|-----|
| **Cartesia Sonic** | Default real-time TTS during calls | Sub-100ms time-to-first-byte. Built for conversational AI. Latency king. |
| **ElevenLabs** | High-quality voice cloning, async content | Best quality. ~300ms TTFB — acceptable for non-realtime. |
| **Sarvam Bulbul** | Hindi/Hinglish TTS | Purpose-built for Indian languages. Handles code-switching natively. |
| **CosyVoice** | Self-hosted fallback | Open-source, self-hostable. Cost control at scale. |

**Voice persona integration:** Each persona (see persona-management.md) has its own voice config — provider, voice_id, and speaking parameters (speed, stability, expressiveness). The TTS pipeline reads the active persona's voice config per session.

**Baseline voice cloning flow:**
1. User records a 30-second voice sample in the app
2. ElevenLabs "Instant Voice Clone" creates a voice profile
3. Voice profile ID stored in `voice_profiles` table (already exists)
4. During calls, TTS uses this profile for outbound speech

**At baseline, one voice profile per user.** Multiple profiles (different voices for different contexts) and voice profile deletion/recreation are refinement.

### 2. Speech Recognition & Understanding

**Provider strategy:**

| Provider | Use case | Why |
|----------|---------|-----|
| **Deepgram Nova-3** | English streaming ASR during calls | ~100-200ms partial results. Best production streaming ASR. |
| **Sarvam Saaras** | Hindi/Hinglish streaming ASR | Significantly better than Whisper for code-mixed speech (Hindi-English in same sentence). Non-negotiable for India market. |
| **Whisper** (batch only) | Voicemail transcription, call recording transcription | Best accuracy for batch. Not suitable for real-time (no native streaming). |

**Language routing at baseline:**
- User profile has a `preferred_language` setting (English or Hindi)
- ASR provider selected based on this setting: English → Deepgram, Hindi → Sarvam
- No auto-detection at baseline (auto-detection adds latency and can misroute). Auto-language-detection is refinement.

**Mobile voice input:** Android speech-to-text integration for chat voice input (currently iOS only). Uses the device's built-in ASR for non-call scenarios. At baseline, this is a frontend integration (M9/KPR), not a backend concern.

**Transcript understanding:** Post-call, the full transcript (from streaming ASR) is stored in `transcripts` + `transcript_segments` (already built). At baseline, this is available for fact extraction by M7 (memory) and for display in the UI. Deeper transcript analysis (sentiment, topic extraction, action item detection) is refinement.

### 3. Conversational AI Call Intelligence

**The core gap:** Today, calls bypass the orchestration engine. A call comes in → Twilio handler manages it directly. At baseline, calls must route through the agentic brain:

**Inbound call flow (baseline):**

```
Phone rings (Twilio webhook)
    │
    ▼
Call handler creates a new agent_run
    │
    ▼
GID classifies the call's initial context → PLANNER
    │
    ▼
Planner creates a plan: "handle inbound call from [caller]"
    │
    ├─ If caller is known contact → load context from memory
    ├─ If purpose is clear → route to appropriate skill
    └─ If unclear → conversational discovery (ask "how can I help?")
    │
    ▼
Executor runs the plan through the normal pipeline
    │
    ▼
Results flow back as voice (TTS) to the caller
```

**Outbound call flow (baseline):**

```
User in chat: "Call Sharma about the meeting tomorrow"
    │
    ▼
Planner creates PlanSpec:
    task 1: assemble call context (meeting details from calendar)
    task 2: initiate call to Sharma (requires_consent)
    task 3: summarize call outcomes
    │
    ▼
Consent card shown in chat: "Call Sharma about tomorrow's 3pm meeting?"
    │
    ▼
User approves → Executor initiates call via Twilio
    │
    ▼
During call: AI voice agent uses assembled context to guide conversation
    │
    ▼
Post-call: summary generated, stored in chat, facts extracted to memory
```

**Call session management:** Each call is an `agent_run` with its own thread in LangGraph. The call state includes: caller info, conversation transcript (streaming), extracted intents, tool call results (if the agent looked something up mid-call), and call metadata (duration, recording URL).

**Intent detection during calls:** Uses the same GID routing as chat messages, but applied to each user utterance during the call. The STT stream feeds utterances → GID classifies → if a new intent emerges mid-call, the planner can create a sub-plan. Example: caller says "also, can you check my calendar?" → GID detects calendar intent → sub-plan created → agent responds with calendar info without ending the call.

**Handoff to human:** At baseline, triggered by:
- Explicit request ("let me talk to a person")
- 3+ consecutive "I don't understand" from either side
- Keyword detection on the STT stream ("emergency," "cancel my account")

### 4. Real-time Voice Streaming

The critical path: STT → LLM → TTS. Every millisecond is felt by the user.

**Latency budget (target: 600-800ms voice-to-voice):**

| Stage | Target | Current bottleneck |
|-------|--------|--------------------|
| Audio capture + network | 50-100ms | Fixed (physics) |
| STT (streaming partial results) | 100-200ms | Endpointing (when to stop listening) |
| LLM time-to-first-token | 150-400ms | Model size, prompt length |
| TTS time-to-first-byte | 80-300ms | Provider choice (Cartesia: 80ms, ElevenLabs: 300ms) |
| Audio playback start | 50-100ms | Buffer + network |
| **Total** | **430-1100ms** | |

**Optimization strategies at baseline:**

| Strategy | What it does | Impact |
|----------|-------------|--------|
| **Stream everything** | STT streams words → LLM streams tokens → TTS streams audio. No stage waits for the previous to finish. | -200-400ms. The single most important optimization. |
| **Filler injection** | While waiting for LLM response, play pre-cached filler ("Let me check that...") | Reduces *perceived* latency to near-zero for the user. |
| **Response caching** | Common utterances (greetings, "what can you do?", "goodbye") → pre-cached TTS audio | 0ms for cached responses. |
| **Endpointing tuning** | Reduce silence threshold from default 500ms to 300-400ms with VAD | -100-200ms per turn. Tradeoff: too aggressive = cuts user off. |
| **Fast LLM for calls** | Use GPT-4o-mini or Groq-hosted model during calls (faster TTFT than GPT-4o) | -100-200ms. Quality tradeoff acceptable for conversational turns. |

**Infrastructure:** LiveKit Agents framework handles the audio routing, turn detection, and pipeline orchestration. Twilio Media Streams bridges PSTN calls to the LiveKit pipeline. This is already partially built (LiveKit integration exists).

### 5. Voice UX

**Turn-taking:** The system needs to know when the user has finished speaking and when to let them continue. At baseline:
- VAD (Voice Activity Detection) detects speech start/stop
- Endpointing threshold: 350ms silence → user is done talking
- If the agent is speaking and the user starts talking → agent stops immediately (interruption handling)

**Hold behavior:** When the agent needs time (waiting for consent, long-running skill), play a template voice at intervals: "Please hold, I'm checking that for you..." every 8-10 seconds. Resume immediately when the result is ready. Already partially built — baseline hardens the interval timing and adds early resume.

**Interruption handling:** User speaks while agent is speaking → immediately stop TTS playback, start processing user's new utterance. No "please wait for me to finish" behavior — the user is always in control.

---

## What Counts as Refinement

| Refinement | Why deferred |
|------------|-------------|
| Auto-language detection (route ASR based on detected language, not user setting) | Adds latency + risk of misrouting. User setting is reliable. |
| Noise cancellation (Krisp SDK or equivalent) | Audio processing layer. Important for quality but not blocking. |
| Auto-WebRTC for Trybo-to-Trybo calls (detect if callee is on Trybo → route via WebRTC instead of PSTN) | Cost optimization. PSTN works at baseline. |
| Speculative execution (start LLM inference on partial transcript) | Advanced optimization. Saves 200-500ms but adds complexity. |
| Sentiment detection during calls | Useful for handoff triggers but keyword detection is sufficient at baseline. |
| LiveKit SIP trunking for Twilio calls | Better calling UX but needs investigation. |
| Live transcript + interactive chat control | Complex UX. Evaluate gain vs complexity post-beta. |
| Transcript analysis (topic extraction, action item detection) | Useful but not blocking. Raw transcript available for memory extraction. |
| Multiple voice profiles per user | One profile is sufficient for launch. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| Agentic orchestration pipeline | Task Decomposition (M3) + Execution Engine (M5) | Calls route through planner/executor |
| Persona voice config | Persona Management (RKP) | TTS reads persona's voice settings |
| Memory context assembly | Memory (M7, KPR) | Call agent needs user context during calls |
| Consent cards in chat UI | Frontend (M9, KPR) | Outbound call consent shown in chat |
| Twilio + LiveKit infra | Already built | Telephony + WebRTC already deployed |

## Effort Estimate

| Component | Effort |
|-----------|--------|
| Inbound call → orchestration pipeline integration | 3-4 days |
| Outbound call → plan → consent → execute flow | 3-4 days |
| Sarvam ASR integration (Hindi/Hinglish streaming) | 2-3 days |
| Cartesia TTS integration (low-latency default) | 1-2 days |
| Streaming-all-the-way pipeline (STT→LLM→TTS without blocking) | 3-4 days |
| Filler injection + response caching | 1-2 days |
| Endpointing tuning + interruption handling | 2 days |
| Call session management (call as agent_run, mid-call intent routing) | 2-3 days |
| Hold behavior hardening (interval template voice + early resume) | 1 day |

**Total: ~18-24 dev days to reach baseline.** Largest single module for RKP. Can be phased:
- Phase A (8-10 days): Agentic call flows (inbound + outbound through orchestration)
- Phase B (5-7 days): Streaming pipeline optimization (latency target)
- Phase C (5-7 days): Hindi ASR + voice UX polish

---

## Key Concepts Explained

**Why is <800ms voice-to-voice the target?**
Below 800ms, a conversation feels natural — like talking to a slightly thoughtful person. Between 800ms-1200ms, users notice the delay but tolerate it. Above 1200ms, it feels like the system is broken. The threshold is perceptual, not technical.

**Why Sarvam is non-negotiable for Hindi:**
Western ASR providers (Deepgram, Whisper) are trained primarily on English. When an Indian user says "meeting kal 3 baje hai, Sharma ji ko call kar dena" (mixed Hindi-English), these providers either drop the Hindi words or hallucinate English substitutes. Sarvam is purpose-built for code-switched Indian speech. The accuracy difference is not marginal — it's the difference between usable and unusable for Hindi-speaking users.

**Why "stream everything" is the #1 optimization:**
Without streaming, the pipeline is sequential: wait for STT to finish → wait for LLM to finish → wait for TTS to finish. Total: 800-1500ms. With streaming, each stage starts as soon as the previous stage produces its first output. STT produces partial words → LLM starts generating → TTS starts speaking the first sentence while the LLM is still generating the second. This overlapping eliminates wait time between stages and can cut 200-400ms.

**What is endpointing and why it matters:**
Endpointing = deciding when the user has finished speaking. The STT system detects silence. Default threshold is ~500ms of silence → "user is done." But 500ms of dead air after every utterance feels sluggish. Tuning to 300-350ms makes the conversation feel snappier — but too aggressive (200ms) and the system cuts the user off mid-thought when they pause to think. This is a product-feel parameter, not just a technical setting.
