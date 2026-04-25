# Phase 1: Pipecat Voice AI Pipeline Integration

> Owner: KPR
> Date: April 2026
> Status: Planned
> Related: `vci-platform-decision.md` (why Pipecat), `phase-2-vci-6-layers.md` (conversation intelligence built on top)

---

## What This Phase Does

Phase 1 replaces the current custom Agent Stream Service with **Pipecat** as the voice AI pipeline framework — giving us production-grade real-time audio infrastructure. Phase 2 then builds the 6 layers of voice calling intelligence (conversation architecture, response design, contextual intelligence, etc.) as Pipecat FrameProcessors on top of this pipeline.

---

## Why Pipecat (Summary)

Full rationale in `vci-platform-decision.md`. The core reason:

Trybo uses **Twilio** (international) and **Knowlarity** (India +91) for outbound calls, both via bidirectional WebSocket streams. Pipecat is the only voice AI framework that supports both without requiring SIP trunking — through its **serializer pattern** (`TwilioFrameSerializer` built-in, `KnowlarityFrameSerializer` we write).

---

## What Pipecat Replaces

Today, the voice AI pipeline is a custom implementation spread across multiple services:

| Current Service | What It Does | Pipecat Replacement |
|---|---|---|
| `agent_stream_service.py` (~1130 lines) | WebSocket handling, ASR session management, turn management, barge-in detection, audio frame routing between providers | Pipecat `FastAPIWebsocketTransport` + serializers + built-in pipeline orchestration |
| `media_stream_service.py` (~700 lines) | Turn orchestration: ASR final → LLM → TTS → send audio, agent reply streaming | Pipecat pipeline: STT → LLM → TTS FrameProcessors with built-in streaming |
| `knowlarity_stream_service.py` (~900 lines) | Knowlarity WebSocket handling, PCM↔µ-law conversion, 16kHz→8kHz resampling, outbound audio batching | Pipecat `FastAPIWebsocketTransport` + `KnowlarityFrameSerializer` |
| `livekit_stream_service.py` (~900 lines) | LiveKit room connection, audio track subscription, 48kHz↔8kHz resampling, RTC frame publishing | Pipecat `LiveKitTransport` |
| Custom VAD in `agent_stream_service.py` | RMS-based voice activity detection, barge-in thresholds | Pipecat built-in **Silero VAD** (industry standard) |
| Custom `TurnManager` in `agent_stream_service.py` | End-of-utterance detection via silence gaps (1000-2500ms) + Cartesia EOS gating | Pipecat **Smart Turn v3** (12ms CPU inference, 94% accuracy, 23 languages) |

**What Pipecat does NOT replace:**
- `telephony_factory.py` — telephony provider routing stays as-is
- `twilio_service.py` — call initiation via Twilio REST API stays as-is
- `knowlarity_service.py` — call initiation via Knowlarity API stays as-is
- `conversation_service.py` — LLM response generation becomes a Pipecat FrameProcessor
- `enhanced_call_voice_service.py` — TTS generation becomes a Pipecat FrameProcessor
- `voice_service.py` — business logic (transfer, disconnect, consent) stays, called from within Pipecat pipeline

---

## How Pipecat Gives Intelligence to Voice Calling

### The Pipeline Architecture

Pipecat processes audio as **Frames** flowing through a chain of **FrameProcessors**. Each processor does one job — and they compose into a pipeline:

```
CURRENT (custom, tightly coupled):

  WebSocket audio → custom ASR wiring → custom turn manager → custom LLM call
  → custom TTS wiring → custom audio frame sender
  
  All in one monolithic service. Hard to modify, hard to test.


WITH PIPECAT (modular, composable):

  Transport (Twilio/Knowlarity/LiveKit)
       ↓
  Serializer (format conversion)
       ↓
  ┌─────────────────────────────────────────────────────────┐
  │                   PIPECAT PIPELINE                       │
  │                                                          │
  │  Silero VAD                                              │
  │    ↓                                                     │
  │  Smart Turn v3 (end-of-utterance, 12ms)                  │
  │    ↓                                                     │
  │  STT FrameProcessor (ElevenLabs Scribe)                  │
  │    ↓                                                     │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │  VCI Intelligence FrameProcessors (Phase 2)     │     │
  │  │                                                  │     │
  │  │  L3: PhaseTracker — opening/middle/closing       │     │
  │  │  L5: ContextAssembler — cross-channel transcripts│     │
  │  │  Orchestrator — consent triggers, phase shifts   │     │
  │  └─────────────────────────────────────────────────┘     │
  │    ↓                                                     │
  │  LLM FrameProcessor (Claude)                             │
  │    ↓ (streams tokens as they generate)                   │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │  Post-LLM Processors                            │     │
  │  │                                                  │     │
  │  │  SpeechDirectiveExtractor — parse rate/energy    │     │
  │  │  TaskOutcomeDetector — committed/deferred/refused│     │
  │  └─────────────────────────────────────────────────┘     │
  │    ↓                                                     │
  │  TTS FrameProcessor (ElevenLabs, with speech directives) │
  │    ↓                                                     │
  │  Audio frames out                                        │
  │                                                          │
  └─────────────────────────────────────────────────────────┘
       ↓
  Serializer (format conversion)
       ↓
  Transport (back to caller)
```

Each VCI layer from Phase 2 becomes a **FrameProcessor** — a Python class with a `process_frame()` method. They plug into the pipeline at the right point. Adding, removing, or modifying a layer doesn't affect the rest of the pipeline.

### Parallel Processing

Pipecat supports **parallel pipelines** — run multiple processors on the same audio simultaneously without blocking the main flow:

```
                     ┌── SentimentAnalyzer (flags frustration)
  Audio frames ──────┤
                     ├── KeywordDetector (consent triggers)
                     │
                     └── Main Pipeline (STT → LLM → TTS)
```

This means the rule-based orchestrator from Phase 2 (signal detection, consent triggers, frustration flags) can run **in parallel** with the main conversation — zero latency cost to the response.

---

## How Pipecat Reduces Latency

### Current Latency Problem

Today's pipeline has a p90 latency of **~3.5 seconds** end-to-end per turn. Here's where the time goes:

```
CURRENT PIPELINE LATENCY BREAKDOWN:

  End-of-utterance detection:  1000-2500ms (silence-based gaps)
  ASR finalization:            200-400ms
  LLM response:               500-1000ms
  TTS generation:              300-600ms
  Audio delivery:              100-200ms
  ─────────────────────────────────────────
  Total:                       2100-4700ms (p90 ~3500ms)
```

The biggest bottleneck is **end-of-utterance detection** — the system waits 1-2.5 seconds of silence before deciding the caller has finished speaking.

### How Pipecat Fixes Each Stage

**1. End-of-Utterance: 1000-2500ms → ~200-400ms**

Current: Silence-based detection with conservative gaps (1000-2500ms quiet required).

Pipecat: **Smart Turn v3** — a dedicated ML model (8MB, 12ms inference) that predicts turn completion from audio features. It doesn't wait for silence — it recognizes speech patterns that indicate the speaker is done. 23 languages supported including Hindi.

```
Current:  Caller stops → wait 1500ms silence → "they're done"
Pipecat:  Caller stops → Smart Turn detects completion in ~200ms → "they're done"
```

Savings: **~800-2000ms per turn.**

**2. Streaming STT→LLM→TTS: Sequential → Overlapped**

Current: ASR completes → full text sent to LLM → LLM generates full response → full text sent to TTS → TTS generates full audio → play.

Pipecat: **Streaming throughout.** STT emits partial transcripts while caller is still speaking. LLM starts generating as soon as STT emits a final. TTS starts synthesizing as soon as the first LLM tokens arrive. Audio starts playing before the LLM has finished generating.

```
Current (sequential):
  [──── STT ────][──── LLM ────][──── TTS ────][── Play ──]
                                                            Total: sum of all

Pipecat (streaming):
  [──── STT ────]
           [──── LLM ────]
                    [──── TTS ────]
                         [── Play ──]
                                     Total: ~max of any single stage
```

Savings: **~500-1000ms per turn** (overlap instead of sequential).

**3. Barge-In: Faster Detection**

Current: Custom RMS-based detection with conservative thresholds. Agent continues speaking for 200-500ms after caller starts talking.

Pipecat: Silero VAD with configurable sensitivity. Detects speech onset faster and **automatically cancels pending LLM/TTS tasks** — the pipeline framework handles cancellation, not custom code.

**4. Back-Channel: Dead Silence → Natural**

Current: Dead silence while LLM processes. No "mm-hmm" or "let me think" while the caller is speaking.

Pipecat: Pipeline supports injecting pre-synthesized audio frames at any point. Back-channel responses and filler speech ("let me check on that...") can be sent while the LLM is processing — through a parallel FrameProcessor.

### Target Latency (Realistic)

The LLM is the bottleneck we cannot remove. Honest numbers based on industry reality:

```
WITH PIPECAT (streaming — all stages overlap):

  Smart Turn v3 EOU detection:          200-400ms
  Network (audio to server):            50-100ms
  STT finalization:                     100-200ms
  LLM first token (Claude Sonnet):      500-800ms  ← the bottleneck
  TTS first audio chunk (ElevenLabs):   200-400ms
  Network (audio back to caller):       50-100ms
  ─────────────────────────────────────────────────
  Total first word (streaming):         ~1.1-2.0s

  Phase 1 targets: p50 ~1.3s, p90 ~1.8s
  Today:           p90 ~3.5s
```

From **~3.5 seconds to ~1.5-1.8 seconds** — primarily from Smart Turn v3 EOU detection (~1-2s saved) and streaming overlap (~0.5-1s saved). LLM latency is unchanged — this is the industry bottleneck that every voice platform faces (Vapi, Retell are all in the ~1.5-2.0s range).

---

## The Three Transports

Pipecat handles all three of Trybo's audio paths through one pipeline:

### Twilio (International Outbound)

```python
from pipecat.serializers.twilio import TwilioFrameSerializer
from pipecat.transports.websocket.fastapi import FastAPIWebsocketTransport

transport = FastAPIWebsocketTransport(
    websocket=websocket,
    params=FastAPIWebsocketParams(
        audio_in_enabled=True,
        audio_out_enabled=True,
        vad_analyzer=SileroVADAnalyzer(),
        serializer=TwilioFrameSerializer(stream_sid, call_sid, account_sid, auth_token),
    ),
)
```

- `TwilioFrameSerializer` is **built-in** — handles µ-law 8kHz ↔ PCM conversion, JSON media events, DTMF, buffer clearing on interruption.
- Replaces: custom Twilio WebSocket handling in `agent_stream_service.py`.

### Knowlarity (India +91 Outbound)

```python
from app.pipecat.serializers.knowlarity import KnowlarityFrameSerializer

transport = FastAPIWebsocketTransport(
    websocket=websocket,
    params=FastAPIWebsocketParams(
        audio_in_enabled=True,
        audio_out_enabled=True,
        vad_analyzer=SileroVADAnalyzer(),
        serializer=KnowlarityFrameSerializer(call_id),
    ),
)
```

- `KnowlarityFrameSerializer` is **custom-written** (~50-100 lines) following the Exotel pattern.
- Handles: binary PCM 16kHz inbound → resample → PCM; outbound PCM → 8kHz → base64 → JSON `playAudio` command.
- Replaces: `knowlarity_stream_service.py` (~900 lines).

### LiveKit WebRTC (Browser/App Calls + Owner Transfer)

```python
from pipecat.transports.livekit import LiveKitTransport, LiveKitParams

transport = LiveKitTransport(
    url=livekit_url,
    token=agent_token,
    room_name=room_name,
    params=LiveKitParams(
        audio_in_enabled=True,
        audio_out_enabled=True,
        vad_analyzer=SileroVADAnalyzer(),
    ),
)
```

- `LiveKitTransport` is **built-in** — joins a LiveKit room as a participant, subscribes to audio tracks, publishes agent audio.
- Replaces: `livekit_stream_service.py` (~900 lines).
- Owner transfer: when the owner joins the LiveKit room, the Pipecat agent disconnects (no AI needed for human-to-human conversation).

### Same Pipeline, Three Transports

```python
# The pipeline is IDENTICAL regardless of transport
pipeline = Pipeline([
    transport.input(),           # Audio in (from any transport)
    stt,                         # ElevenLabs Scribe
    phase_tracker,               # L3: conversation phase
    context_assembler,           # L5: cross-channel context
    orchestrator,                # Rule-based signal check
    llm,                         # Claude (with VCI prompt)
    speech_directive_extractor,  # Parse rate/energy from LLM output
    tts,                         # ElevenLabs (with speech directives)
    transport.output(),          # Audio out (to any transport)
])
```

One codebase. Swap the transport. The intelligence stays the same.

---

## What We Build vs What Pipecat Provides

| Component | Pipecat Provides | We Build |
|---|---|---|
| VAD (Silero) | Yes | - |
| Turn detection (Smart Turn v3) | Yes | - |
| Barge-in / interruption handling | Yes | - |
| Streaming STT→LLM→TTS pipeline | Yes | - |
| Audio codec conversion | Yes (via serializers) | `KnowlarityFrameSerializer` (~50-100 lines) |
| Twilio WebSocket handling | Yes (`TwilioFrameSerializer`) | - |
| LiveKit room handling | Yes (`LiveKitTransport`) | - |
| L3: Phase tracking | - | Custom FrameProcessor |
| L4: Response intelligence (prompt) | - | LLM system prompt design |
| L4: Speech directives | - | Custom FrameProcessor (extract from LLM output, pass to TTS) |
| L5: Context assembly | - | Custom FrameProcessor (Supabase queries) |
| L6: Recovery strategies | - | LLM prompt additions |
| Rule-based orchestrator | - | Custom FrameProcessor (parallel pipeline) |
| Task outcome detection | - | Custom FrameProcessor |
| Consent request triggering | - | Custom FrameProcessor (calls existing consent service) |

---

## Code Reduction

| Current File | Lines | After Pipecat |
|---|---|---|
| `agent_stream_service.py` | ~1130 | Replaced by Pipecat pipeline config (~100-150 lines) |
| `media_stream_service.py` | ~700 | Replaced by Pipecat pipeline streaming |
| `knowlarity_stream_service.py` | ~900 | Replaced by `KnowlarityFrameSerializer` (~50-100 lines) |
| `livekit_stream_service.py` | ~900 | Replaced by Pipecat `LiveKitTransport` config |
| Custom VAD / TurnManager | ~200 | Replaced by Silero VAD + Smart Turn v3 |
| **Total custom pipeline code** | **~3830 lines** | **~200-300 lines** (serializer + pipeline config) |
| **VCI FrameProcessors (new)** | - | ~500-800 lines (Phase 2 intelligence layers) |

We go from ~3830 lines of custom real-time audio infrastructure to ~200-300 lines of Pipecat configuration + ~500-800 lines of VCI intelligence. The infrastructure becomes Pipecat's responsibility. Our code focuses entirely on conversation intelligence.

---

## Implementation Order

### Step 1: Pipecat Foundation
- Install Pipecat + dependencies
- Write `KnowlarityFrameSerializer`
- Set up basic pipeline: Silero VAD → ElevenLabs Scribe STT → Claude LLM → ElevenLabs TTS
- Wire to Twilio transport (replace `agent_stream_service.py` for Twilio calls)
- Verify: outbound Twilio call works with Pipecat pipeline

### Step 2: Knowlarity Transport
- Wire `KnowlarityFrameSerializer` to the same pipeline
- Verify: outbound Knowlarity call works with same pipeline
- Test: India +91 number dialing, audio quality, latency

### Step 3: LiveKit Transport
- Wire Pipecat `LiveKitTransport` for WebRTC calls
- Verify: browser/app calls work with same pipeline
- Verify: owner transfer still works (Pipecat agent disconnects, owner joins room)

### Step 4: VCI Intelligence Layers (Phase 2)
- Build custom FrameProcessors for L3-L6
- PhaseTracker, ContextAssembler, Orchestrator, SpeechDirectiveExtractor, TaskOutcomeDetector
- Wire into the pipeline
- This is where Phase 1 (pipeline) and Phase 2 (intelligence) converge

### Step 5: Remove Legacy Pipeline
- Decommission `agent_stream_service.py`, `media_stream_service.py`, `knowlarity_stream_service.py`, `livekit_stream_service.py`
- All calls flow through Pipecat

---

## Summary

| Aspect | Current | After Phase 1 | After Phase 2 (VCI + LangGraph) |
|---|---|---|---|
| **Voice AI pipeline** | Custom Agent Stream Service (~3830 lines) | Pipecat framework (~200-300 lines config) | Same Pipecat + LangGraph agent inside |
| **End-of-utterance** | Silence-based (1000-2500ms gaps) | Smart Turn v3 (200-400ms, ML-based) | Same |
| **STT→LLM→TTS** | Sequential | Streaming (overlapped) | Same streaming + LangGraph VCI node |
| **VAD** | Custom RMS-based | Silero VAD (industry standard) | Same |
| **Transport** | 3 separate custom implementations | 3 Pipecat transports, same pipeline | Same |
| **p90 latency** | ~3.5 seconds | ~1.8 seconds | ~2.0 seconds (richer context, ~20ms LangGraph overhead) |
| **Conversation intelligence** | None | Basic (same LLM call as today, just faster pipeline) | LangGraph VCI node with L3-L6 layers |
| **Tool calling** | Not supported | Not supported | VCI delegates to parallel Tool Executor node |
| **Context compaction** | Not supported | Not supported | Compaction node keeps context stable over long calls |
| **Multilingual** | Not supported | Auto-detect via ElevenLabs Scribe | VCI responds in caller's language |
| **Adding a new telephony provider** | Write ~900 lines custom service | Write ~50-100 line serializer | Same |

Phase 1 gives us the production-grade pipeline infrastructure. Phase 2 gives us the conversation intelligence via LangGraph — VCI as a single node handling all turns, with parallel Tool Executor and Compaction nodes running in the background. Together, they transform the voice agent from "LLM generates text → TTS speaks it" into a natural, context-aware, multilingual conversational partner that can research, fetch data, and engage the caller simultaneously.
