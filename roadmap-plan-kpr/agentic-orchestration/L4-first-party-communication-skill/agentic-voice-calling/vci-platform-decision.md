# VCI Platform Decision: Pipecat as Voice AI Pipeline

> Owner: KPR
> Date: April 2026
> Status: Decision made
> Related: `agentic-voice-calling-intelligence.md` (6-layer VCI architecture)

---

## Context

The Voice Calling Intelligence (VCI) plan defines 6 layers that make voice conversations feel natural. Before building, we evaluated whether a third-party voice AI platform or framework could deliver the pipeline layers (L1-L2: voice, timing, turn-taking) while preserving our telephony-provider-agnostic architecture (Twilio for international, Knowlarity for India).

### How Trybo's outbound calling works today

Trybo has a **telephony factory** (`telephony_factory.py`) that routes outbound calls to different providers based on destination:

```
get_outbound_telephony_provider(destination):

  +91 (India)      → KnowlarityTelephonyProvider
  Everything else  → TwilioTelephonyProvider
```

When the recipient answers, the call connects via bidirectional WebSocket to our **Agent Stream Service** (`agent_stream_service.py`) — which runs the voice AI pipeline (Cartesia ASR → LLM → TTS). LiveKit is used only for owner live transfer (owner joins an active call via WebRTC from the mobile app).

### What we evaluated

| Category | Platforms | Outcome |
|---|---|---|
| **Managed voice AI** | Vapi, Retell AI, Bland.ai | Rejected — not telephony-provider-agnostic |
| **Open-source pipeline** | **Pipecat (Daily.co)** | **Selected** — transport-agnostic, has Twilio serializer, supports custom serializers |
| **Open-source infra + pipeline** | LiveKit Agents | Not usable without SIP trunking — outbound calls don't go through LiveKit |

---

## Decision

**Pipecat** — use Pipecat as the transport-agnostic voice AI pipeline framework. Build the 6 VCI intelligence layers on top of Pipecat's pipeline.

Pipecat replaces our custom Agent Stream Service for the voice AI pipeline (L1-L2), while preserving our existing telephony factory (Twilio + Knowlarity).

---

## Why Not Vapi

Vapi is the strongest managed voice AI platform — bring-your-own LLM/TTS/STT, good function calling, production-grade turn-taking, clean developer API.

**The single reason we can't use it: Vapi is not telephony-provider-agnostic — it bundles its own telephony, so we cannot use our Knowlarity integration for India outbound calls.**

Vapi lets you choose which LLM, TTS, and STT provider to use — but NOT which telephony provider. When you use Vapi, ALL calls go through Vapi's Twilio (or Vonage/Telnyx). There is no way to route India calls through Knowlarity while using Vapi's AI pipeline.

```
WHAT TRYBO NEEDS:                    WHAT VAPI OFFERS:

  Knowlarity (India +91)  ──┐         Vapi's Twilio ─── All calls
  Twilio (international)   ──┤                           (can't use
  [future providers]       ──┘                            Knowlarity)
```

If Vapi supported plugging in custom telephony providers, it would be the recommended platform.

---

## Why Not LiveKit Agents

LiveKit Agents provides a production-grade voice AI pipeline (VAD, turn-taking, barge-in, streaming STT→LLM→TTS). But it requires calls to be inside a **LiveKit room** — which means audio must enter via WebRTC or SIP.

Trybo's outbound calls use **Twilio/Knowlarity bidirectional WebSocket streams**, not LiveKit rooms. Without SIP trunking (Twilio → LiveKit SIP Bridge), the call audio never enters a LiveKit room, and LiveKit Agents' pipeline has nothing to work with.

```
WITHOUT SIP TRUNKING:

  Knowlarity/Twilio → WebSocket → Our backend
  LiveKit Agents sits idle — no audio to process.

WITH SIP TRUNKING:

  Knowlarity/Twilio → SIP → LiveKit Room → LiveKit Agents pipeline
  But: Knowlarity SIP trunking compatibility is unverified.
```

SIP trunking adds infrastructure complexity and an unverified Knowlarity integration. Pipecat achieves the same result (production-grade pipeline) by connecting directly to the existing WebSocket streams — no SIP trunking needed.

LiveKit continues to be used for its current role: **owner live transfer** (owner joins an active call via WebRTC from the mobile app).

---

## Why Pipecat

### Transport-agnostic by design

Pipecat separates the **voice AI pipeline** (VAD, turn-taking, STT, LLM, TTS) from the **transport** (how audio gets in and out). It uses a serializer pattern — different serializers handle different telephony providers' WebSocket protocols, all feeding into the same pipeline.

Pipecat has **7 built-in telephony serializers**:

| Serializer | Provider |
|---|---|
| `TwilioFrameSerializer` | **Twilio** (we use this) |
| `ExotelFrameSerializer` | **Exotel** (Indian telephony — similar to Knowlarity) |
| `TelnyxFrameSerializer` | Telnyx |
| `VonageFrameSerializer` | Vonage |
| `PlivoFrameSerializer` | Plivo |
| `GenesysFrameSerializer` | Genesys |
| `ProtobufFrameSerializer` | Generic protobuf |

And **9 transport implementations** including:

| Transport | Our Use |
|---|---|
| `FastAPIWebsocketTransport` | Twilio/Knowlarity WebSocket calls |
| `LiveKitTransport` | Owner transfer (WebRTC) |
| Others (Daily, SmallWebRTC, local, etc.) | Not needed |

### Direct plug-in to our existing telephony

No SIP trunking. No transport bridging. Pipecat connects directly to Twilio/Knowlarity bidirectional WebSocket streams:

```
Telephony Factory (existing, unchanged)
  ├── Knowlarity (India +91) → WebSocket → KnowlarityFrameSerializer ──┐
  └── Twilio (international) → WebSocket → TwilioFrameSerializer ──────┤
                                                                         │
                                                                         ▼
                                                            ┌──────────────────────┐
                                                            │   PIPECAT PIPELINE   │
                                                            │   (same for both)    │
                                                            │                      │
                                                            │  Silero VAD          │
                                                            │  Turn-taking         │
                                                            │  Barge-in            │
                                                            │  EOU detection       │
                                                            │  STT streaming       │
                                                            │  LLM orchestration   │
                                                            │  TTS streaming       │
                                                            └──────────────────────┘
```

The `TwilioFrameSerializer` already exists and is production-ready. For Knowlarity, we write a `KnowlarityFrameSerializer` following the same pattern as the `ExotelFrameSerializer` (Exotel is an Indian telephony provider with a similar WebSocket protocol). A serializer requires only two methods: `serialize()` and `deserialize()`.

### Production-grade voice AI pipeline we don't have to maintain

Pipecat provides the L1-L2 capabilities that our Agent Stream Service currently handles manually:

| Capability | Our Agent Stream Service | Pipecat |
|---|---|---|
| VAD | Custom implementation | Silero VAD (industry standard) |
| Turn-taking | Custom `TurnManager` | Built-in, configurable |
| Barge-in / interruption | Custom detection + handling | Built-in with audio buffer clearing |
| EOU detection | Silence-based (1000-2500ms gaps) | Silero VAD + configurable |
| Streaming STT→LLM→TTS | Custom wiring | Built-in pipeline with FrameProcessors |
| Back-channel ("mm-hmm") | Not implemented | Supported via pipeline injection |
| Audio codec conversion | Custom µ-law ↔ PCM | Handled by serializer automatically |
| DTMF handling | Custom | Built-in (parsed by serializer) |

By moving to Pipecat, we stop maintaining custom real-time audio infrastructure and focus on the VCI intelligence layers (L3-L6) which are our actual differentiator.

### Knowlarity serializer is a small build

Knowlarity's WebSocket transport (verified from `knowlarity_stream_service.py`) uses bidirectional WebSocket with:
- **Inbound:** Raw binary PCM at 16kHz (not JSON-wrapped like Twilio)
- **Outbound:** 8kHz PCM, base64-encoded, wrapped in JSON `{"type": "playAudio", "data": {"audioContent": "..."}}`
- **First message:** JSON metadata with `callid` and `customer_number`
- **Batching:** 200ms outbound chunks (vs. Twilio's 20ms)

Writing the `KnowlarityFrameSerializer` follows the established Pipecat pattern. The `ExotelFrameSerializer` (also an Indian telephony provider) is a reference implementation. A serializer implements two methods:

```python
class KnowlarityFrameSerializer(FrameSerializer):
    async def serialize(self, frame: Frame) -> str | bytes | None:
        # Pipeline PCM → resample to 8kHz → base64 encode
        # → wrap in {"type": "playAudio", "data": {"audioContent": "..."}}

    async def deserialize(self, data: str | bytes) -> Frame | None:
        # First message (text): parse JSON metadata (callid, customer_number)
        # Subsequent messages (binary): raw PCM 16kHz → resample to pipeline rate
        # → return InputAudioRawFrame
```

The format differences from Twilio (binary PCM vs. JSON µ-law, 16kHz vs. 8kHz inbound) are exactly the kind of variation that serializers abstract away. Pipecat's `FastAPIWebsocketTransport` already handles both text and binary WebSocket messages.

### LiveKit transport for owner transfer

Pipecat has a built-in `LiveKitTransport`. This means the owner transfer flow can also be handled through Pipecat's pipeline — potentially giving the owner a better experience (hearing the actual caller audio in the LiveKit room, not just the agent's TTS responses).

### Open-source, no vendor lock-in

Pipecat is BSD-licensed, actively maintained by Daily.co, with a large community. We control the entire stack:
- Our telephony (Twilio + Knowlarity)
- Our serializers (Twilio built-in, Knowlarity custom)
- Our STT/LLM/TTS providers (ElevenLabs, Claude, etc.)
- Our VCI intelligence layers (L3-L6 as FrameProcessors)

---

## The Architecture

```
┌──────────────────────────────────────────────────────────┐
│              TELEPHONY LAYER (existing, unchanged)        │
│                                                           │
│  telephony_factory.py routes by destination:              │
│    +91 (India) → Knowlarity                               │
│    International → Twilio                                  │
│                                                           │
│  Recipient answers → bidirectional WebSocket               │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│         PIPECAT TRANSPORT + SERIALIZER                   │
│                                                           │
│  FastAPIWebsocketTransport                                │
│    + TwilioFrameSerializer (international calls)          │
│    + KnowlarityFrameSerializer (India calls)              │
│                                                           │
│  Handles: µ-law ↔ PCM conversion, DTMF, audio buffering  │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│         PIPECAT PIPELINE (Layers 1-2)                    │
│                                                           │
│  L1: ElevenLabs TTS (cloned voice) + speech directives   │
│      (rate/energy/pauses driven by LLM output)            │
│  L2: Silero VAD → turn-taking → barge-in → EOU detection │
│      back-channel responses, thinking pauses              │
│  ASR: ElevenLabs Scribe (streaming)                       │
│                                                           │
│  All wired as Pipecat FrameProcessors                     │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│         VCI INTELLIGENCE (Layers 3-6)                    │
│         Built as custom Pipecat FrameProcessors          │
│                                                           │
│  Rule-based orchestrator (pre-LLM):                      │
│   - Phase state (L3) — opening/middle/closing             │
│   - Context assembly (L5) — cross-channel transcripts,   │
│     chat memory, authority rules                          │
│   - Signal check — consent triggers, frustration flags    │
│                                                           │
│  Single LLM call (L4) with:                              │
│   - Conversation-focused prompt (not constraints)         │
│   - Pragmatic intelligence (hedge/commitment detection)   │
│   - Emotional adaptation (tone matching)                  │
│   - Speech directives output (rate/energy for TTS)        │
│   - Recovery strategies (L6) in prompt                    │
│                                                           │
│  Post-LLM:                                               │
│   - Task outcome update (committed/deferred/refused)      │
│   - Phase transition check                                │
│   - Consent request trigger if needed                     │
└──────────────────────────────────────────────────────────┘

OWNER TRANSFER (when needed):

  Pipecat LiveKitTransport → Owner joins via WebRTC (mobile app)
```

---

## What Changes vs. Today

| Component | Today (Agent Stream Service) | With Pipecat |
|---|---|---|
| Telephony factory | Twilio + Knowlarity | **Unchanged** |
| Twilio audio handling | Custom WebSocket handler | `TwilioFrameSerializer` (built-in) |
| Knowlarity audio handling | Custom WebSocket handler | `KnowlarityFrameSerializer` (write once, ~50-100 lines) |
| VAD | Custom implementation | Silero VAD (Pipecat built-in) |
| Turn-taking / barge-in | Custom `TurnManager` | Pipecat built-in |
| STT streaming | Custom Cartesia integration | Pipecat FrameProcessor (plug in ElevenLabs Scribe) |
| TTS streaming | Custom `enhanced_call_voice_service` | Pipecat FrameProcessor (plug in ElevenLabs) |
| Audio codec conversion | Custom µ-law ↔ PCM | Handled by serializer |
| VCI Layers 3-6 | Not built yet | Built as Pipecat FrameProcessors |
| Owner transfer | Custom LiveKit bridge | Pipecat `LiveKitTransport` |

---

## Summary

| Platform | Verdict | Reason |
|---|---|---|
| **Vapi** | Rejected | Not telephony-provider-agnostic. Cannot use our Knowlarity integration for India outbound calls. Bundles telephony with AI pipeline inseparably. |
| **LiveKit Agents** | Not usable for outbound | Requires SIP trunking to get call audio into LiveKit rooms. Outbound calls use Twilio/Knowlarity WebSocket streams — no SIP trunk exists. LiveKit continues for owner transfer only. |
| **Pipecat** | **Selected** | Transport-agnostic. Has built-in `TwilioFrameSerializer`. We write a `KnowlarityFrameSerializer` (~50-100 lines) following the Exotel pattern. Same pipeline handles both providers. Gives us production-grade VAD, turn-taking, barge-in, streaming — replacing custom Agent Stream Service code we currently maintain ourselves. VCI Layers 3-6 built as Pipecat FrameProcessors. |

Pipecat gives us the voice AI pipeline (L1-L2) without forcing us to change our telephony. Our telephony factory, our Twilio integration, and our Knowlarity integration stay exactly as they are. Pipecat sits between the telephony WebSocket and our VCI intelligence — handling the real-time audio engineering so we can focus on conversation intelligence.
