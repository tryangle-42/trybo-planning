# First Party Skill Creation — Baseline Definition

> Owner: AKN
> Scope: Component 7 — 5 subcomponents (7.1–7.5)
> Module mapping: First Party Skills (Communication + Research)
> Current completion: ~66% — 30 of 56 capabilities exist, 7 partial, 19 missing
> Source: SUBCOMPONENT_CAPABILITY_MAP.md

---

## What This Is

These are the skills Trybot owns and operates — calling people, messaging on WhatsApp, researching the web. Each skill is a complete capability from trigger to outcome. They are the "hands" of the system: the components that actually do things in the real world.

The core happy-path for each skill works. The gaps are in edge-case handling (WhatsApp 24-hour window), operational enforcement (retry policies), and cost optimization (research caching).

## Current State

| What Exists | What's Missing |
|---|---|
| Outbound/inbound calls with provider failover | Complete scheduling retry policies |
| Full voice pipeline (ASR → LLM → TTS → playback) | (voice pipeline baseline met) |
| Conversation engine (intros, context, intent, summaries) | (conversation engine baseline met) |
| WhatsApp send/receive/track/auto-reply | 24-hour window enforcement, message retry |
| Web research with provider routing + fallback | Result caching |

## Baseline Definition

**The line is:** Each skill can execute its core function end-to-end reliably — calls are placed/received with retry on scheduled failures, WhatsApp complies with Business API rules, and web research doesn't pay twice for the same query. Anything beyond this is refinement.

---

## Baseline Components

### 7.1 Call Lifecycle — ~1.5 dev days

**What this is:** Manages the full phone call state machine: outbound creation across providers (Twilio, Knowlarity), inbound routing with DND/availability checks, status tracking via webhooks, multi-party membership, and provider failover. Every call state change produces events consumed by UI, history, and billing.

#### Baseline deliverables

**1. Complete scheduling retry policy**
The scheduling mechanism exists (APScheduler DateTrigger), but retry policy fields (`max_attempts`, `backoff_seconds`, `retry_window_start`, `retry_window_end`) need formalization on the call record. Without retry policies, a scheduled call that fails at the scheduled time is simply lost — the user doesn't know it failed, and no retry happens.

| Refinement (deferred) | Why |
|---|---|
| Queue rate limiting per provider/user | Providers enforce their own limits; volume low |
| Warm transfer with context | Cold transfer works; warm requires context summary + handoff protocol |
| Detailed cost tracking | Duration tracked; cost is a lookup table |
| Full policy enforcement | Basic DND/availability exist; advanced policies are edge cases |

**Dependencies:** Job Queue (1.9), Event Bus (1.2)

---

### 7.2 Voice Pipeline — 0 dev days (BASELINE MET)

**What this is:** Handles real-time bidirectional audio: ingest from telephony providers, streaming ASR with provider abstraction, TTS with voice cloning, agent audio playback, multi-track recording, and transcript persistence. Latency is critical — parallel TTS, audio buffering, and VAD keep responses conversational.

9 of 12 capabilities exist. The 2 missing (noise cancellation, latency monitoring) and 1 partial (word-level timestamps) are all refinements.

| Refinement (deferred) | Why |
|---|---|
| Audio preprocessing (noise cancel, echo removal) | Audio quality acceptable; adds latency |
| Streaming latency monitoring | Useful for optimization, not function |
| Word-level timestamps | Speaker-level labels exist; word-level is UI enhancement |

---

### 7.3 Conversation Engine — 0 dev days (BASELINE MET)

**What this is:** The conversational intelligence layer for voice calls. Determines what the agent says, how it responds, and when it escalates. Generates personalized intros, context-aware responses, classifies intent per turn, manages turn-taking, and produces structured summaries with action items.

All 6 core capabilities exist. Escalation to human is partial (cold transfer works via LiveKit) but the core conversation loop is complete.

| Refinement (deferred) | Why |
|---|---|
| Multi-language support | English-only for initial market |
| Conversation repair | Agent already generates context-aware responses |
| Guardrails (topic drift, hallucination) | Prompt engineering handles most cases |
| Emotion detection | Agent works without it |
| Conversation scripting | Free-form covers most use cases |

---

### 7.4 WhatsApp Engine — ~2 dev days

**What this is:** Manages the complete WhatsApp message lifecycle: outbound sending (text + templates), inbound processing (text, media, reactions), conversation management, delivery status tracking, and AI auto-reply with task completion detection. Handles WhatsApp Business API constraints.

#### Baseline deliverables

**1. 24-hour window enforcement**
Track `last_user_message_at` per conversation. Before sending: check if within 24h → text OK → if expired → require template. Why baseline: violating this rule gets the phone number **banned by Meta**. This isn't a nice-to-have — it's a compliance requirement for continued operation.

**2. Message retry with exponential backoff**
On Meta API 5xx/timeout → retry with backoff (1s, 2s, 4s). On persistent failure → mark `failed` → DLQ. Without retry, messages are silently lost — the user thinks the message was sent, but it wasn't.

| Refinement (deferred) | Why |
|---|---|
| Interactive messages (buttons, lists) | Text replies work; interactive is UX enhancement |
| Rate limiting for bulk | Low volume doesn't hit Meta limits |
| Outbound rich media | Text and templates cover current use cases |
| Conversation analytics | Product metrics, not functional |

**Dependencies:** Data Store (1.8), LLM Gateway (1.11)

---

### 7.5 Web Research Engine — ~1 dev day

**What this is:** Provides the agent with access to current, real-world information — prices, availability, business details. Operates at two levels: deep synthesis (Perplexity Sonar) and fast lookup (Serper/DuckDuckGo). Provider routing is context-dependent with automatic fallback chains.

#### Baseline deliverables

**1. Result caching with TTL**
Redis cache: `search:{provider}:{query_hash}` → cached result. TTL by query type (news: 1h, facts: 24h). Dedup by URL when merging. Why baseline: every query costs money (Perplexity, Tavily). Caching identical queries avoids paying twice for the same answer. Lowest-effort, highest-impact optimization.

| Refinement (deferred) | Why |
|---|---|
| Query reformulation | Current prompts produce adequate queries |
| Multi-query decomposition | Single queries cover most use cases |
| Source credibility scoring | Provider results already ranked |
| Content extraction from URLs | Summaries suffice; full-page extraction is edge case |
| Cost optimization per provider | Manual monitoring sufficient |

**Dependencies:** Redis (1.1), LLM Gateway (1.11)

---

## What Counts as Refinement (explicitly deferred)

| Category | Examples | Why deferred |
|---|---|---|
| Advanced call features | Warm transfer, cost tracking, full policy enforcement | Core call flow works without them |
| Audio processing | Noise cancellation, echo removal, latency monitoring | Audio quality is acceptable |
| Conversation intelligence | Multi-language, guardrails, emotion, scripting | English free-form conversation works |
| WhatsApp richness | Interactive messages, rich media, analytics | Text + templates cover launch |
| Research depth | Query reformulation, decomposition, credibility | Core search + fallback works |

## Dependencies

| Needs | From | Why |
|---|---|---|
| Event Bus | Platform 1.2 | Status events for calls/messages |
| Job Queue | Platform 1.9 | Scheduled call execution |
| LLM Gateway | Platform 1.11 | AI generation for conversation + auto-reply |
| Data Store | Platform 1.8 | Persistence for all skill data |
| Redis | Platform 1.1 | Research caching |

## Effort Estimate

| Module | Effort |
|---|---|
| 7.1 Call Lifecycle | 1.5 days |
| 7.2 Voice Pipeline | 0 days (baseline met) |
| 7.3 Conversation Engine | 0 days (baseline met) |
| 7.4 WhatsApp Engine | 2 days |
| 7.5 Web Research Engine | 1 day |
| **Total** | **~4.5 dev days** |
