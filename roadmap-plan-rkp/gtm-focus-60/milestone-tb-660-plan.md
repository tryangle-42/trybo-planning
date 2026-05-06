# TB-660 Milestone Plan: Voice Intelligence (M5)

> Owner: RKP
> Date: 2026-05-06
> Status: Draft for review
> Jira: [TB-660](https://tryangle42-team.atlassian.net/browse/TB-660) (In Progress)
> Parent epic: TB-644 (GTM)
> Blocked by: [TB-647](https://tryangle42-team.atlassian.net/browse/TB-647) (M2 cross-channel consent — In Progress)
> Related research: `vci-platform-decision.md`, `phase-1-pipecat-voice-ai-pipeline.md`, `phase-2-vci-5-layers.md`, `system-design.md`

---

## 1. Context & Decision Tension

The Jira ticket was scoped before KPR's platform research landed. Its assumptions no longer hold:

| Original Jira assumption | Research outcome (KPR, April 2026) |
|---|---|
| Vapi/Retell ships in 2–3 weeks (recommended for beta) | **Vapi rejected** — bundles its own telephony, cannot keep Knowlarity for India +91 |
| Custom build = 5–7 weeks | **Pipecat selected** — open-source, transport-agnostic; sits between our existing telephony and our intelligence |
| "Build vs buy" decision pending in W1-2 | Decision already made: **build on Pipecat, not from scratch, not on a managed platform** |

The Jira sub-items remain valid as *capabilities* (outbound flow, inbound + mid-call consent, latency, multilingual, summary) but the path to them is different. The Jira estimate of 13–20 days assumed the Vapi path; the Pipecat path is realistically 26–38 days for the full VCI, with a sensible demo cut at ~18–26 days.

This document supersedes the Jira description's scope/estimate. The Jira ticket's *demo* (outbound consent → call → summary; inbound → mid-call consent) remains the success bar.

---

## 2. Revised Milestone Goal

Beta-ready, agentic voice agent on Pipecat, across all three transports (Twilio, Knowlarity, LiveKit), with mid-call consent, cross-channel context, and multilingual auto-routing.

**Success criteria:**
- p90 first-word latency ≤ 2.0s (today: ~3.5s)
- Outbound + inbound demos from the Jira ticket work end-to-end
- Mid-call consent webhook fires through Pipecat → Application Agent → owner FCM → caller hears result
- Hindi / English / Hinglish handled with no per-turn directives (text carries language, ElevenLabs multilingual v2 auto-detects)
- Owner transfer (LiveKit + Twilio) still works
- All current use cases UC-1…UC-23 from `system-design.md` have feature parity
- Legacy stream services (~3830 LOC) decommissioned without regression

---

## 3. Original M5 Sub-items → Pipecat/VCI Mapping

| Original Jira bullet | New work (under Pipecat path) | Sub-milestone |
|---|---|---|
| Evaluate + decide build vs buy (3-5d) | **Done** by KPR research; ratify in this doc | (this doc) |
| Outbound call flow (3-5d) | Pipecat Twilio + Knowlarity transport, thin VCI (L1+L4) | SM-A, SM-B, SM-C |
| Inbound call flow + mid-call consent webhook (3-4d) | Inbound on Pipecat, `task_handoff` → Application Agent → existing `information_request_processing_service` | SM-D |
| Latency optimization, filler, endpointing (2-3d) | Smart Turn v3 + streaming overlap + Pipecat back-channel (mostly free with Pipecat foundation) | SM-A (free), SM-E (filler) |
| Multi-language eval + call summary (2-3d) | ElevenLabs Scribe + multilingual v2 (verification, not new build); summary enrichment via TaskOutcomeDetector | SM-G |

---

## 4. Current Codebase Impact

### Replaced (decommissioned in SM-H, ~3830 LOC removed)

| File | LOC | Replaced by |
|---|---|---|
| `app/services/agent_stream_service.py` | ~1130 | Pipecat pipeline + `FastAPIWebsocketTransport` |
| `app/services/media_stream_service.py` | ~700 | Pipecat streaming pipeline |
| `app/services/knowlarity_stream_service.py` | ~900 | `KnowlarityFrameSerializer` (~80 LOC) |
| `app/services/livekit_stream_service.py` | ~900 | Pipecat `LiveKitTransport` |
| Custom VAD + `TurnManager` (in agent_stream_service) | ~200 | Silero VAD + Smart Turn v3 |

### Preserved (no changes)

- `app/services/telephony_factory.py`, `twilio_telephony.py`, `knowlarity_telephony.py` — provider routing & call initiation
- `app/services/call_service.py`, `call_status_service.py`, `call_recording_service.py` — call lifecycle, DB records
- `app/services/voice_service.py` — transfer / disconnect business logic (called from Application Agent via task_handoff)
- `app/services/incoming_call_service.py` — direct-transfer eligibility (runs before pipeline)
- `app/services/voice_cloning_service.py`, `voice_profile_service.py` — TTS configuration
- `app/services/transcript_service.py`, `transcript_persistence_service.py` — transcript storage
- `app/services/information_retrieval_*`, `information_request_processing_service.py` — consent flow (called via task_handoff)
- `app/services/call_hold_service.py`, `enhanced_call_voice_service.py` — held audio, voice utilities
- `app/services/livekit_session_service.py` — token / room management
- `app/routes/calls.py`, `incoming_voice.py`, `livekit.py`, `twilio*.py`, `voice*.py` — API surface unchanged
- All DB schema (`calls`, `transcripts*`, `bot_tasks`, `voice_profiles`, `consent_requests`)

### New (added across SMs)

```
app/pipecat/                          # SM-A, SM-B
├── pipeline_factory.py               # Per-call pipeline construction
├── serializers/
│   └── knowlarity.py                 # KnowlarityFrameSerializer
├── processors/
│   ├── langgraph_processor.py        # Wraps LangGraph VCI agent
│   ├── transcript_persister.py       # Per-turn DB writes
│   ├── language_detector.py          # Sets caller_language in state
│   └── back_channel.py               # Pre-synthesized "mm-hmm"
└── config.py

app/langgraph/                        # SM-C onward
├── vci_agent.py                      # 3-node graph (vci, dispatcher, compaction)
├── nodes/
│   ├── vci_node.py                   # L4 — single LLM call per turn
│   ├── task_dispatcher.py            # Forwards task_handoffs to App Agent
│   └── compaction_node.py            # Background context compaction
├── state.py                          # VCIState TypedDict
├── context/
│   ├── context_assembler.py          # L1
│   ├── briefing_summarizer.py        # L1 helper
│   ├── phase_tracker.py              # L2
│   └── recovery_monitor.py           # L3
└── prompts/
    └── vci_system_prompt.py          # L4 prompt
```

---

## 5. Sub-milestone & Ticket Breakdown

Each SM ends in a demo + review gate. After every gate, we re-decide: continue to next SM, pause to harden, or scope-cut.

### SM-A — Pipecat foundation on Twilio (gating)
**Goal:** one outbound Twilio call running end-to-end through Pipecat, behind a feature flag. No VCI yet — same LLM/prompt as today.

| Ticket | Title | Est. | Notes |
|---|---|---|---|
| TB-660.1 | Install Pipecat + base pipeline scaffold | 2d | `app/pipecat/pipeline_factory.py`, Silero VAD, Smart Turn v3, ElevenLabs Scribe STT, ElevenLabs TTS wired together |
| TB-660.2 | Wire `TwilioFrameSerializer` + transport behind feature flag | 2d | Flag: `voice.pipecat_enabled` (per-call). Routes through `app/routes/twilio_agent_streams.py` |
| TB-660.3 | `TranscriptPersister` FrameProcessor | 1d | Drop-in for existing per-turn `transcript_service` writes |

**SM-A estimate: 5 days. Review gate:** real outbound Twilio call works through Pipecat; latency baseline measured against today's p90.

---

### SM-B — Transport parity (Knowlarity + LiveKit)
**Goal:** all three transports on Pipecat. Same pipeline, three serializers/transports.

| Ticket | Title | Est. | Notes |
|---|---|---|---|
| TB-660.4 | `KnowlarityFrameSerializer` | 2d | ~80 LOC; pattern from Pipecat's built-in `ExotelFrameSerializer`. Handles binary PCM 16kHz in / base64 µ-law 8kHz out / `playAudio` JSON wrapping |
| TB-660.5 | Pipecat `LiveKitTransport` for WebRTC | 2d | Owner-transfer interaction: Pipecat agent disconnects when owner joins room (verify) |
| TB-660.6 | Knowlarity → Twilio fallback on Pipecat | 1d | Fallback path already exists in `telephony_factory`; verify pipeline stays identical when serializer swaps |

**SM-B estimate: 5 days. Review gate:** UC-1…UC-23 feature parity verified across all three transports.

---

### SM-C — Thin VCI (L1 briefing + L4 LLM)
**Goal:** demo-able context-aware agent. The two layers a caller actually perceives.

| Ticket | Title | Est. | Notes |
|---|---|---|---|
| TB-660.7 | LangGraph `vci_node` as Pipecat FrameProcessor | 2d | Wraps current `conversation_service` LLM call into a LangGraph node; structured output (`response`, `task_handoffs`, `task_outcome`, `phase_transition`) |
| TB-660.8 | L1 ContextAssembler + briefing summarizer | 2d | Loads chat memory + authority rules at call start; `briefing_summarizer` generates ~150-tok cross-channel summary from recent calls + WhatsApp |
| TB-660.9 | VCI system prompt v1 | 1d | Pragmatic + emotional intelligence + language-mirroring instruction + `task_handoff` capability (no tool list) |

**SM-C estimate: 5 days. Review gate:** record an outbound + inbound demo. Is it noticeably more human than today?

---

### SM-D — Mid-call consent via task_handoff (the M5 hero demo)
**Goal:** the exact Jira demo — caller asks, agent triggers consent, owner approves, agent relays.

| Ticket | Title | Est. | Notes |
|---|---|---|---|
| TB-660.10 | Task Dispatcher node + plumbing into Application Agent | 2d | `task_dispatcher` forwards natural-language descriptions to the main Application Agent, which routes to a registered skill. Results land back in `VCIState.completed_tasks` |
| TB-660.11 | Bridge `information_request_processing_service` as a task_handoff target | 1d | Application Agent can route a "get permission for X, then return X" handoff to the existing consent + retrieval pipeline |
| TB-660.12 | End-to-end mid-call consent flow on Pipecat | 2d | Depends on TB-647 (M2 mid-call webhooks) being landed. Verify FCM round-trip from Pipecat-driven call. Includes smart-hold polish: `call_hold_service` plays a template "still checking on that..." every 8–10s while consent is pending, and resumes immediately when result arrives (early resume on FCM callback) |

**SM-D estimate: 5 days. Review gate:** record the Jira demo (inbound + outbound consent). Internal-user soft launch.

---

### SM-E — Conversation architecture (L2 + L3)
**Goal:** natural openings/closings + recovery. Polish, not blocking for first demo.

| Ticket | Title | Est. | Notes |
|---|---|---|---|
| TB-660.13 | L2 PhaseTracker | 1d | Opening / middle / closing state; transitions emitted by L4 |
| TB-660.14 | L3 RecoveryMonitor + recovery patterns in prompt | 2d | Stuck-loop detection, repeat detection, graceful-abort prompt patterns. **Concrete handoff-to-human triggers:** explicit caller request ("let me talk to a person", "transfer me"), 3+ consecutive "I don't understand" / repeat-back failures, keyword detection on STT stream ("emergency", "cancel my account"). Triggers route through existing `voice_service` owner-transfer path |
| TB-660.15 | Pre-cached common-utterance audio | 1d | Pre-synthesize TTS for top-N high-frequency utterances (greetings, "one moment", "thanks for calling", "goodbye", "let me check on that"). FrameProcessor checks the cache before TTS roundtrip — saves 200–400ms on cached responses. Distinct from Pipecat's back-channel (which is mid-utterance acknowledgment); this is full-response caching |

**SM-E estimate: 4 days. Review gate:** stability under adversarial calls (interruptions, off-topic, repetition); handoff triggers verified.

---

### SM-F — Stability for long calls

| Ticket | Title | Est. | Notes |
|---|---|---|---|
| TB-660.16 | Compaction node | 2d | Background, no LLM cost during the turn. Summarizes turns older than last 5 into ~100 tokens; drops used OTPs and delivered tool results |

**SM-F estimate: 2 days. Review gate:** 20+ turn call stays under p90 target; context size flat.

---

### SM-G — Multilingual + summary polish

| Ticket | Title | Est. | Notes |
|---|---|---|---|
| TB-660.17 | Multilingual eval (Hindi / Hinglish / code-switch) + Sarvam fallback gate | 2d | Record 5 calls each in Hindi, English, Hinglish on Pipecat + ElevenLabs Scribe. Measure WER on a code-switched corpus. **Decision gate:** if Scribe WER > 25% on Hinglish, integrate Sarvam Saaras for Indian-language ASR (TTS stays on ElevenLabs multilingual v2). Sarvam integration is +1d if triggered, but kept out of base estimate. Rationale: voice-intelligence baseline (RKP) flagged Western ASR providers historically drop Hindi words on code-switched speech — this is the validation point |
| TB-660.18 | Call-summary enrichment with task outcomes | 1d | TaskOutcomeDetector emits `committed/deferred/hedged/refused/blocked`; threaded into existing `generate_call_summary` |

**SM-G estimate: 3 days (+1d if Sarvam fallback triggers). Review gate:** Hindi/Hinglish demo + summary quality check.

---

### SM-H — Decommission legacy

| Ticket | Title | Est. | Notes |
|---|---|---|---|
| TB-660.19 | Remove legacy stream services after ≥1 week at 100% Pipecat | 1d | Delete `agent_stream_service`, `media_stream_service`, `knowlarity_stream_service`, `livekit_stream_service`. Update `docs/arch/module-map.md` |

**SM-H estimate: 1 day. Review gate:** all old code removed, no regressions on dashboards.

---

## 6. Sequencing & Estimate

```
SM-A (5d) ──► SM-B (5d) ──► SM-C (5d) ──► SM-D (5d) ──► SM-E (4d) ──► SM-F (2d) ──► SM-G (3d) ──► SM-H (1d)
   ▲                                          ▲
   │ feature flag on                          │ TB-647 must be landed
   │                                          │
   └─── "real M5 beta" cut here ──────────────┘
        SM-A → SM-D = 20 days = the Jira demo bar
        SM-E onward = post-demo polish (extra 10 days for full VCI)

Total full VCI: ~30 days (~31 if Sarvam fallback triggers in SM-G)
                vs Jira's original 13-20d Vapi-path estimate
```

**Hard dependencies:**
- SM-D blocked by TB-647 (M2 cross-channel consent + mid-call webhooks)
- Everything blocked by SM-A (Pipecat must work on at least one transport before parallelism makes sense)

**What can parallelize after SM-A:**
- SM-B (Knowlarity serializer + LiveKit) and SM-C (VCI thin) are independent — can run in parallel if there are two engineers
- SM-E, SM-F, SM-G are independent of each other

---

## 7. Risks & Open Questions

| Risk | Mitigation |
|---|---|
| `KnowlarityFrameSerializer` is unproven — no Pipecat reference for Knowlarity | Use `ExotelFrameSerializer` as reference (similar Indian telephony WS protocol). Spike at start of SM-B. If serializer is harder than expected, defer Knowlarity to SM-D and ship Twilio-only beta first |
| LiveKit owner-transfer interaction with Pipecat disconnect is untested | Verify in SM-B as a discrete subtask before declaring SM-B done. Owner transfer must remain functional |
| Anthropic Claude streaming compatibility with Pipecat LLM FrameProcessor | Validate in SM-A — Pipecat's `OpenAILLMService` pattern needs adaptation for Anthropic. May need a custom `AnthropicLLMService` |
| TB-647 slips → SM-D blocked | SM-A through SM-C ship independently; SM-D parks until TB-647 lands. Consent demo deferred but rest of milestone is still demo-able |
| Latency target (p90 ≤ 2.0s) depends on Smart Turn v3 + LLM streaming both working as advertised | Measure latency at every review gate. If Smart Turn v3 underperforms on Indian-accented English, fall back to silence-based EOU with shorter gap (300-500ms) |
| Feature flag complexity — running two pipelines in parallel during rollout | Per-call flag (not per-user) so we can A/B by call. Default off until SM-D demo passes |
| ElevenLabs Scribe may underperform on Hinglish / code-switched Indian speech | RKP's voice-intelligence baseline flagged that Western ASR providers (Deepgram, Whisper) historically drop Hindi words or hallucinate English substitutes for code-mixed speech. Validate explicitly in SM-G with a Hinglish corpus. Fallback path: Sarvam Saaras for Indian-language ASR while keeping ElevenLabs multilingual v2 TTS. Adds ~1d if triggered |

---

## 8. Verification Plan

**Per SM:**
- SM-A: outbound Twilio call works; latency baseline measured
- SM-B: smoke test all 23 use cases on each transport (Twilio, Knowlarity, LiveKit)
- SM-C: side-by-side recording — current pipeline vs Pipecat+VCI on the same scenario
- SM-D: end-to-end consent demo (the exact Jira demo: inbound + outbound)
- SM-E: 10-call adversarial test set (interruptions, repetition, off-topic, ASR failures)
- SM-F: 20+ turn synthetic conversation; latency stays flat
- SM-G: 5 calls each in Hindi, English, Hinglish; summary quality reviewed
- SM-H: 1 week at 100% Pipecat with no error-rate regression

**Continuous:**
- Latency dashboards (today's p90 ~3.5s baseline, target ≤ 2.0s post-SM-C)
- Per-turn cost tracking (LLM tokens, STT seconds, TTS characters)
- Per-call error-rate monitoring (timeouts, ASR failures, transfer failures)

---

## 9. Out of Scope for TB-660

**Surface / scope deferrals:**
- Voice agent for non-call surfaces (web embed, in-app voice) — separate milestone
- Replacing `voice_cloning_service` or training new voices
- Procedural memory updates from call data (mem0 wiring) — separate milestone
- Call recording transcription pipeline (already works via Deepgram) — unchanged
- New telephony providers beyond Twilio + Knowlarity

**Audio / ML refinements (carried over from voice-intelligence baseline as deferred):**
- Noise cancellation (Krisp SDK or equivalent) — audio-quality layer, not blocking
- Speculative LLM execution on partial transcripts — saves 200–500ms but adds substantial complexity
- Sentiment / prosody detection during calls — keyword detection sufficient at baseline
- Multiple voice profiles per user — one profile per user is sufficient for launch
- Transcript topic / action-item extraction — raw transcript already available for memory module
- Auto-WebRTC routing for Trybo-to-Trybo calls (PSTN bypass) — cost optimization, not a baseline requirement
- Live transcript + interactive in-call chat control — UX-heavy, evaluate post-beta
- LiveKit SIP trunking for Twilio calls — explicitly rejected by `vci-platform-decision.md` rationale

---

## 10. Open Decisions Before Implementation Starts

1. **Feature flag strategy:** per-call (recommended) vs per-user vs global? Per-call gives the safest rollout.
2. **LLM choice for VCI L4:** stay on the same Claude model as `conversation_service` today, or evaluate a faster model for first-token latency? Recommend stay-the-same for SM-C, evaluate in SM-G.
3. **Anthropic streaming in Pipecat:** confirm Pipecat has a maintained Anthropic LLM service, or do we write a thin wrapper in SM-A? (~half-day if needed.)
4. **Beta-cut bar:** is SM-A → SM-D (20 days) the bar for "M5 ships," with SM-E onward as a follow-up milestone? Recommend yes, formalize at SM-D review gate.
