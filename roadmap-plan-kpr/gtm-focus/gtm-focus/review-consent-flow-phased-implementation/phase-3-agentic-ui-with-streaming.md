# Phase 3 — Agentic UI with Token Streaming

## Implementation Status (updated 2026-04-22)

Phase 3 is **substantially complete**. The backend Pydantic AI agent, SSE transport, device-tool bridge, and the chat-style agentic consent screen are all live. Below are the key deviations and additions vs. the original plan:

### Architectural Deviations

- **Chat-thread UI model**: The plan described separate `AgentActivityPanel.tsx` + `DraftArea.tsx` components. Implementation uses a unified **chat-thread model** via `AgenticConsentLayout.tsx` + `ChatBubble.tsx` — agent steps, draft cards, owner messages, and Mode B answers all render as bubbles in a scrollable thread (closer to ChatGPT UX). `AgentStepRow.tsx` is kept as-is inside bubbles.
- **Input model**: Plan originally described "Ask agent" + "Send to caller" two-button split. Implementation uses a **single input box** (bottom-pinned) + **[Approve & Send]** on the draft card. Simpler, less ambiguous.
- **Mid-stream cancel**: Plan described cancel-on-type / cancel-on-tap-draft / Take-over button. Implementation **disables the input during streaming** instead — no mid-stream interruption. Owner waits for agent to finish, then types a refinement. [Decline] in the header is always available.
- **No pre-classification phase**: Plan described a Phase A permission classification before the agent loop. Implementation skips this — **the agent decides autonomously** which tools to call, and the tool executors handle permissions inline when called (as described in the plan's alternative path). `data_sources` / `missing_data_sources` from the request payload are not read by the frontend.

### Additions (not in original plan)

- **Step deduplication by label**: `addStepIfNew()` in `useConsentAgent.ts` deduplicates steps by both `id` AND `label`. Prevents the same tool (e.g., `get_time`) from showing twice when the LLM calls it multiple times in one turn.
- **Mode B freeze-on-next-turn pattern**: Agent messages (Mode B — question answers) are rendered inline from hook state via `agentMessageSnapshot`, NOT frozen into `chatMessages` via `useEffect`. Freezing into chat history happens only when the **next turn starts** (inside `handleSendInstruction`), eliminating a React state timing issue where both a frozen message and a live bubble would show the same steps simultaneously.
- **Frozen agent_message carries stepResults**: When agent_messages are frozen (on next turn), they include `stepResults` alongside `steps`, so the frozen bubble shows full tool result text (e.g., found contacts list).
- **`agent_message` streaming events**: Backend streams Mode B answers token-by-token via `agent_message` SSE events (accumulated in hook state), finalized by `agent_message_done`. Plan mentioned `agent_message` but didn't detail the token-streaming variant.
- **`get_time` tool**: Backend consent agent has a `get_time` tool (current time + timezone). Listed in the plan's tool set but worth noting it's live and working.

### Key Implementation Files (actual, not planned)

| Layer | File | Notes |
|---|---|---|
| L1 Wire | `lib/sseClient.ts` (kept, not replaced by react-native-sse) | XHR-based SSE works reliably on RN |
| L2 Agent | `app/services/consent_agent_service.py` | Pydantic AI agent loop, SSE event emission |
| L3a Registry | Not yet extracted into `data_source_registry.py` | Tool set is inline in `consent_agent_tools.py` |
| L3b Tools | `app/services/consent_agent_tools.py` | 5 device tools + draft_response |
| L4 Executors | `lib/consentAgentTools.ts` | Device tool executors with permission gating |
| L6 Prompt | `app/services/consent_agent_prompt.py` | Drafting-only system prompt |
| L7 UI | `components/consent/AgenticConsentLayout.tsx`, `ChatBubble.tsx`, `AgentStepRow.tsx` | Chat-thread model |
| L7 Hook | `hooks/useConsentAgent.ts` | SSE handler, step state, conversation memory |
| L8 Surface | `components/InformationConsentScreen.tsx` | Wraps AgenticConsentLayout |

### 3.9 — Information Retrieval Module (independent from consent)

The Information Retrieval module is a **completely independent component** — it is NOT part of the consent request component or the agentic orchestration layer. It is its own module with its own growth path.

**What it is:**
- A standalone skill that retrieves data from connected data sources
- Has no knowledge of consent requests, calls, WhatsApp, or any specific consumer
- Any agent or component can use it — the consent agent is just one consumer among many
- Will grow significantly in M1 when more data sources are connected (Gmail, Slack, Outlook, etc.)
- Its future shape and capabilities cannot be predicted now — only the base is built here

**Relationship to consent:**
- The consent request agent USES this skill when it needs device data for drafting
- The Information Retrieval module does not know or care that it's being used by a consent agent
- If the consent agent is removed tomorrow, the Information Retrieval module continues to exist and serve other consumers

**Base structure:**
```
app/services/information_retrieval/
├── __init__.py
├── analyzer.py            — LLM-based analysis (determines what data is needed)
├── data_source_registry.py — Extensible source definitions
└── executors/             — Per-source data extraction (grows in M1)
```

**M1 will extend this module** with new data sources, new executors, and new capabilities. The base created here provides the structure for that growth without prescribing its direction.

### Remaining Work

- **L3a Data Source Registry**: Extract `DataSourceDefinition` entries from inline tool declarations into a standalone `data_source_registry.py` module (extensibility for future sources)
- **Inline permission grant in step rows**: Plan describes `?` step rows with [Grant Permission] buttons. Currently permission errors are returned to the agent which drafts around missing data. Full inline grant flow is partially implemented (see `AgentStepRow.tsx` `actionLabel`/`onAction` props) but not wired to the consent agent loop.
- **Late permission grant**: Granting a previously-denied permission and re-querying on the next turn — not yet wired
- **All-unknown special case**: Agent loop still runs; should bypass and show blank input
- **iOS + Android on-device E2E verification** (3.7): Partial — tested on iOS, Android pending

### Future Work (Deferred — Not in Scope for Phase 3 Refinements)

- **Server-side agent loop cleanup on dropped SSE connection**: The agent loop runs on the backend while SSE streams to the consent screen. If the connection drops without an explicit cancel (app backgrounded, network glitch, force-close), the server-side loop's lifecycle is currently undefined — it may run into a void, hold in-memory state, or only get cleaned up by the existing per-turn timeout. Deliberately deferred from this cycle. Will be addressed when **agentic chat memory** is developed in a future cycle, where session lifecycle and server-side loop ownership become first-class design concerns. Implementation of the current Phase 3 refinements should not introduce special handling for this case.

---

## Architectural Refinements (Pending — to be applied on top of shipped Phase 3)

Three flow-level changes that tighten the consent component's ownership boundary, simplify what the payload carries, and fix how the conversation context is referenced for WhatsApp. No user-visible behavior change.

### Refinement 1 — The Consent Component Prepares Its Own Payload  ❌ REVERTED (2026-04-27)

This refinement was implemented and then reverted after testing. The original "channel intelligence prepares the full payload (analyzer + assembly) and hands the finished package to the consent component" model is restored.

**Why reverted:**
- The channel intelligence already has the conversation in memory (session history, WhatsApp thread). Re-loading it inside the consent component was wasteful duplication.
- Moving the analyzer call into the consent component added ~1 LLM-call worth of latency between trigger and FCM notification — most painful on inbound calls where the owner expects the alert during the live conversation.
- The consent component is a thin, channel-agnostic pipe — keeping intelligence in each channel agent (where the conversation already lives) is cleaner than centralizing it.

**Restored model:**
The channel intelligence (voice or WhatsApp) detects an information request, runs the analyzer on its own conversation history, assembles the full request payload, and hands the finished payload to the consent component. The consent component is a pass-through — it persists what it was given, sends the FCM notification, and returns the trigger outcome.

The other refinements in this document (2 — drop data-source arrays from the analyzer output, 3 — channel-specific transcript reference, 4 — round-trip + hold-period discard, 5 — terminal-status checkpoints, 6 — agent prompt rules + `request_otp` tool) are NOT affected by this revert and remain in force.

### Refinement 2 — Drop the Data-Source Lists From the Payload

**Today's flow:**
The analyzer outputs two arrays — the data sources it thinks are relevant and the data sources it thinks are missing — and they are persisted inside the payload's conversation context. The response drafting agent and the UI both have access to them, but in practice neither uses them: the agent makes its own decisions from the registry, and the UI doesn't read these arrays at all.

**New flow:**
The analyzer no longer produces these arrays. The payload no longer carries them. The response drafting agent decides at runtime which sources to query, based on the summary, the retrieval prompt, and its knowledge of the Data Source Registry. Permission outcomes are observed inline when a tool runs, not pre-classified.

**Why:**
- Removes dead fields nobody reads
- Stops duplicating intelligence the agent already has
- Lets the agent freely re-decide on each turn (e.g., a refinement might call a different source than the original draft) without being constrained by an upfront classification

### Refinement 3 — Conversation Reference Is Channel-Specific

**Today's flow:**
The payload's `conversation_context` carries a single `transcript_id` field. This works for calls (one transcript record per call) but not for WhatsApp (a conversation is many separate message records). The WhatsApp path either passes a null/fake reference or loses the conversation context on the consent screen.

**New flow:**
The conversation reference shape depends on the channel:

| Channel | What's referenced | Cutoff |
|---|---|---|
| Inbound call | The call transcript record | Snapshot sequence number — segments up to this point |
| Outbound call | The call transcript record (same shape as inbound) | Snapshot sequence number — segments up to this point |
| WhatsApp | The WhatsApp conversation thread (already identified by a top-level relational column) | The id of the last message that existed when consent was triggered |

**Precondition for outbound calls (decided)**: outbound calls must persist their conversation as a transcript record (with sequenced segments) by the time consent is triggered — same way inbound calls do. If today's outbound flow keeps conversation only as in-memory session history, that persistence step is a prerequisite to applying Refinement 1 on the outbound channel.

The cutoff exists so the consent screen renders a **frozen view** of the conversation as it was at trigger time. Without the cutoff, messages arriving on the WhatsApp side after the consent was created would leak into the conversation context the owner sees.

**Why:**
- Honest representation of how each channel stores conversations (call = one record + segments; WhatsApp = many message records)
- Removes the fake/null transcript reference WhatsApp consents currently carry
- Gives the consent screen an unambiguous cutoff so the conversation view doesn't drift while the owner is reading it

### Verification (Flow-Level)

1. ✅ A new consent request can be created without the channel intelligence ever calling the analyzer
2. ✅ The consent component, given only a conversation reference and the requester info, produces the same quality of summary + retrieval prompt as the channel agent did before
3. ✅ Persisted consent requests no longer carry data-source classification arrays inside the payload
4. ✅ A WhatsApp consent request renders the WhatsApp thread up to its trigger-time cutoff, even if newer messages have arrived since
5. ✅ A call consent request renders the call transcript up to its snapshot point, exactly as before
6. ✅ The response drafting agent still produces sensible drafts using only the summary, the retrieval prompt, and the registry — no quality regression
7. ✅ All three channel flows (inbound call, outbound call, WhatsApp) still ship end-to-end

(Implementation guardrails for Refinements 1–5 are consolidated in the section below, after Refinement 5.)

### Refinement 4 — Trigger Round Trip and Channel "On Hold" State

The two open flow gaps above are answered together because they describe one mechanism.

**The trigger is a round trip, not a fire-and-forget.**

The channel intelligence triggers the consent component and **waits for the outcome**. The consent component does its preparation (load the full conversation transcript, run the analyzer to ASSEMBLE the payload, persist the row, send the FCM notification) and returns one of two results to the channel intelligence:

- **Success — consent created and notified**. The consent row was created and the owner notification was sent. The result includes the request id so the channel intelligence can track it in its session state.
- **Failure**. The consent component could not complete preparation or notification (DB error, FCM error, analyzer error, conversation load error, etc.). The outcome is a single flat "failed" verdict with no category — the channel intelligence falls back the same way regardless of which step failed.

**The analyzer does not gate consent.** Its job is purely to ASSEMBLE the consent request payload (summary + retrieval prompt) from the transcript. It does not have a "no consent needed" verdict. The decision that consent is needed was already made by the channel intelligence when it chose to trigger; the consent component does not second-guess that decision. There is no third "no consent needed" outcome — that path does not exist.

This avoids a redundant LLM call (analyzer deciding + voice intelligence re-generating its own reply) and keeps the consent component's intelligence narrow: build the payload, never gate.

**Two-stage caller-facing messaging — eliminates the silence.**

The channel intelligence does not wait silently for the consent component's outcome. It speaks to the requester twice:

- **Stage 1 — fired the moment the trigger is sent (parallel with the consent component's work)**: an immediate filler along the lines of *"I'm asking the owner about that"*. Wording is owned by the channel intelligence; the goal is to break the silence so the requester is never sitting in dead air while the consent component prepares the request.
- **Stage 2 — fired after the consent component's outcome arrives**:
  - On **success**: a follow-up such as *"I've sent the request — putting you on hold while we wait for the response"*. Stage 2 is the moment the conversation actually transitions to the on-hold state.
  - On **failure**: a fallback such as *"I'm not able to answer that right now"*. No on-hold state.

This two-stage pattern means there is no caller-side waiting period — Stage 1 fills the time the consent component spends preparing, and Stage 2 reflects the real outcome.

The consent component never speaks to the caller / WhatsApp counterpart directly. Both Stage 1 and Stage 2 are channel-intelligence responsibilities. The consent component only prepares, persists, notifies, and reports back.

**Why on-hold state is gated on the success-with-consent outcome (Stage 2, not Stage 1)**: an actual hold (caller cannot interact with the agent until owner responds) requires that a real consent request exists and a real notification has gone out. Stage 1 is a conversational filler, not the hold itself. The hold begins when Stage 2 confirms it.

**Responsibility split (locked):**

| Responsibility | Owner |
|---|---|
| All caller-facing or counterpart-facing messages and audio (on-hold messages, stall replies, failure fallbacks, the final delivered response wording for that channel) | Channel intelligence (voice / WhatsApp / future channels) |
| Build the consent request (load conversation, run analyzer, assemble payload) | Consent component |
| Send the consent request to the owner (notification) | Consent component |
| Collect the owner's response | Consent component |
| Route the collected response back to the originating channel | Consent component |
| Take the routed response and deliver it through the channel's transport | Channel intelligence |

The consent component's surface is owner-facing only (notification + the consent screen). Anything the requester or counterpart hears or reads is the channel intelligence's job, end to end.

**The conversation goes "on hold" the moment a consent request is pending — for every channel.**

For calls this is already the case (the call is literally placed on hold). For WhatsApp, the same semantics now apply at the conversation level:

- After the channel intelligence sends the on-hold message ("we are working on it, please hold on") in response to the triggering message, the WhatsApp conversation enters a **suppressed-input state**.
- Any further messages from the WhatsApp counterpart while the consent request is pending are **dropped on the floor — not persisted to the database, not added to the conversation history, not stored in any memory layer, not surfaced anywhere.** The WhatsApp intelligence stays silent and the inbound is discarded entirely.
- The suppressed-input state ends when the consent request reaches a terminal status — the owner approves and the response is delivered, the owner declines, or the request expires / is canceled.
- Once the state ends, normal auto-reply behavior resumes for the next inbound message **that arrives after the state ended**. The discarded suppressed messages are gone — they are not retroactively persisted or considered.

**Symmetric rule for call channels.** The same hold-period discard applies to live calls. While a call is on hold pending owner consent, any audio / speech from the caller during the hold is **not transcribed into the transcript record, not stored in the DB, and not held in any memory layer**. Hold-period input from the requester side — across every channel — is non-existent from the system's point of view.

**Outbound calls adopt the inbound hold model — no more soft suppression.** Today, outbound calls use a soft-hold mechanism: a session flag suppresses the agent's speech turns and the owner's response is picked up reactively when the caller next utters something. This is being replaced with the inbound model:

- **Literal hold during the wait — transport-specific**: when an outbound consent is triggered and the round-trip succeeds, the call is placed on a literal hold at the transport layer. The behavior depends on which transport the outbound call uses:
  - **LiveKit-based outbound** → mirror inbound exactly: literal hold, hold music, full unhold-with-response on approve. The caller experience is identical to an inbound call placed on hold.
  - **Knowlarity-based outbound** → silent hold is sufficient. The current session-flag-based soft-hold (no agent response, no transcription) is acceptable; no hold music or media manipulation needed at the transport layer.
- **Proactive delivery on approve**: when the owner approves, the consent component routes the response back through the same path inbound uses — the call is taken off hold and the response is delivered (spoken out) immediately, without waiting for the caller to speak first.
- **Reactive pickup is removed**: the previous outbound-only soft-hold flag and the reactive injection-on-next-turn model are deprecated. Outbound and inbound consent behavior become the same shape end-to-end.

This unification eliminates the outbound-specific delivery race (caller hangs up before next utterance, response sits unconsumed) by removing the reactive pickup entirely. Delivery now happens at approve time, not at next-utterance time.

**No agent response during hold, on any channel.** The channel intelligence does not respond to anything the requester says or sends during the hold period — no acknowledgment, no clarifying question, no filler. WhatsApp stays silent on the conversation; the call agent generates no speech. The only message the requester gets during hold is the on-hold message that started the hold; everything after that is deliberate silence until the consent resolves and the owner's response (or a fallback) is delivered.

**Why this WhatsApp behavior:**
- Mirrors how calls already behave (caller is on hold, can't have a parallel conversation with the agent until the consent resolves)
- Prevents the WhatsApp intelligence from drafting replies based on a partial picture while the owner is still reviewing the request
- Keeps the WhatsApp counterpart's view consistent with what the owner is being asked to approve (otherwise the agent could say something contradicting the owner's pending decision)

### Refinement 5 — Terminal-Status Handling (Two Checkpoints)

The consent screen and the response-delivery path must both check status — at two different moments — and respond differently when the request is no longer actionable.

**Checkpoint 1 — at notification tap (before the screen renders)**

When the owner taps the FCM notification, status is checked **before navigating to the consent screen**.

- **If status is `pending`**: navigate to the consent screen as designed. Agent loop starts, transcript context loads, action buttons are live.
- **If status is terminal** (`expired`, `canceled`, `approved`, `declined`, `delivered`): the owner is **not shown the consent screen at all**. The notification handler short-circuits — no screen render, no agent loop, no transcript fetch. (A lightweight signal such as a system toast may indicate "this request is no longer active", but that is a UI affordance, not a full screen.)

The intent is: if the request is already dead by the time the owner taps, there is nothing for them to act on, so the screen should not appear and waste their attention.

**Checkpoint 2 — at Approve & Send tap (after the screen has been open)**

The owner can have the consent screen open while the request is `pending`, then have it expire mid-review (clearest example: the live call ends while they're refining the draft). Before the consent component delivers the drafted response to the channel, it **re-verifies the request is still in `pending` status server-side**.

The drafted-response API performs this check on every submission. It does not trust the screen's last-known status.

- **If still pending**: deliver the response normally through the channel-specific handler.
- **If now terminal**: do NOT deliver. The API returns an "expired" outcome to the screen, and the screen renders an **expired bubble inside the existing chat thread** (same visual style as agent / owner / draft bubbles) telling the owner the request has expired. The owner's drafted text stays visible so they can see what they tried to send; action buttons disable.

**Why two checkpoints, not one:**
- Checkpoint 1 protects the cold-open case — the owner taps the notification on a request whose status has already flipped to terminal (e.g., the owner declined earlier on a different notification surface, or any future path that explicitly cancels). Showing the screen at all would mislead them into thinking they can still act.
- Checkpoint 2 protects the warm-edit case — the owner has the screen open and is actively drafting when the channel ends. The recheck prevents delivering a response into a dead conversation, and the bubble closes the loop honestly so the owner knows their draft was ready but never sent.

**Status lifecycle is owner-driven, not channel-driven.**

Status stays `pending` until the owner acts. There is no proactive cancellation by the channel intelligence when a call ends, no TTL sweep, no automatic expiration by elapsed time. A consent request can sit in `pending` indefinitely if the owner never touches it.

The expired transition is computed lazily at Checkpoint 2: when the owner taps Approve & Send, the consent component checks whether the originating channel is still live. If it isn't, the request flips to `expired` at that moment and the screen shows the expired bubble. That is the only path by which "the channel has ended" causes a status change.

Implication: under this lazy model, Checkpoint 1 will rarely fire for the "channel-ended" case — because status will still be `pending` when the owner taps the notification. The cold-open of a stale consent therefore opens the full agentic screen, the agent drafts, the owner refines and hits Approve — and Checkpoint 2 catches it. Acceptable trade-off: a little wasted drafting compute on rare stale-tap cases, in exchange for not introducing background timers or cancel hooks.

### Implementation Guardrails (to prevent the obvious wrong turns)

- **Boundary direction**: the consent component depends on the analyzer module, never the reverse. The analyzer must remain consent-unaware so other components (e.g., the future main agent in Phase 4) can use it.
- **WhatsApp cutoff is a message id, not a timestamp**: timestamps in a chat thread can collide and have lower precision; an explicit message id matches a single row unambiguously.
- **No DB schema migration is required**: the dropped data-source arrays and the new WhatsApp cutoff field both live inside the existing JSONB payload column. New rows have the new shape. Do not write a destructive migration.
- **No backward-compatibility work for old rows**: this refactor is treated as a fresh implementation. Do not write legacy rendering paths, do not backfill old rows, do not run a one-time migration to rewrite old payloads. Existing pending consents from before the refactor are out of scope — implementation does not need to make them render correctly. Effort spent on backward-compat would be wasted.
- **Analyzer output must shrink, not just the payload**: the data-source arrays should be removed at the analyzer's output too. Otherwise the analyzer keeps producing dead fields that simply never get persisted.
- **Frontend dead-plumbing must be removed in the same scope as Refinement 2**: when the data-source arrays leave the payload, every frontend touchpoint that referenced them — the FCM notification handler, the Redux slice, the consent request interface, any source-classification helpers, any conditional rendering branching on these arrays — must be cleaned up in the same change set. No deprecation period, no follow-up cleanup pass. Leaving dead plumbing means future readers will assume those fields still mean something.
- **The analyzer call belongs in the consent component, not in the screen layer**: the response drafting agent (which lives next to the screen) does its own runtime source decisions; it must not duplicate the analyzer's work. The analyzer runs once, at consent-creation time, on the server.
- **Channel intelligence loses the analyzer dependency entirely**: after Refinement 1, voice and WhatsApp intelligences should not import or invoke the analyzer anywhere. If they still do, the boundary has been drawn wrong.
- **Trigger round trip is synchronous from the channel's perspective**: the channel intelligence must know the trigger outcome before it sends any caller-facing message. Do not implement this as a fire-and-forget.
- **The on-hold caller-facing message remains the channel intelligence's responsibility**: the consent component must not deliver caller / counterpart messages — it only reports the trigger outcome upstream.
- **The analyzer reads the full transcript, not a pre-computed summary**: the analyzer's verdict on "is consent needed?" must be made against the actual conversation, not against any earlier shortened version. The consent component loads the transcript via the channel-aware loader and hands the full content to the analyzer.
- **The trigger has exactly two outcomes — success and failure**: there is no third "no consent needed" outcome. The analyzer assembles the payload; it does not decide whether consent is needed. If the channel intelligence triggered, a consent request is created (success) or it is not (failure). The on-hold message fires on success, the fallback fires on failure — no other branches exist.
- **The analyzer assembles, it does not gate**: the analyzer's only job is to produce the summary and retrieval prompt from the transcript. It does not output a "needs_consent" boolean and it must not be wired up that way. The decision that consent is needed already happened upstream in the channel intelligence; revisiting it inside the consent component would mean two LLM passes for the same decision and would create a path where the channel says "I'm asking the owner" then has to backtrack.
- **No internal retries on the trigger path**: the consent component must fail fast on the first error from any step (analyzer LLM call, DB write, FCM send, conversation load). Internal retries — even one — extend the trigger latency, and on a live call the requester is waiting in silence the whole time. The channel intelligence's failure fallback is the only retry surface; it is not the consent component's job to hide flakiness.
- **WhatsApp suppressed-reply state is keyed off pending consent for the conversation**: the WhatsApp intelligence checks "is there a pending consent request for this `whatsapp_conversation_id`?" before deciding whether to auto-reply. The check must be on consent status, not on a separate flag, so the state always matches the actual lifecycle.
- **Multi-device contention is not a concern**: the app already enforces single-device login (only one active device per user at a time). The consent flow inherits this guarantee — there can never be two consent screens open for the same owner on different devices, so no claim/release/lock mechanism is needed. The freshness guarantee remains correct as written.
- **Owner correction is always authoritative**: when the owner refines with input that contradicts the original summary, the original retrieval prompt, or what the agent previously said, the agent must defer to the owner's correction without negotiation. It does not try to reconcile, does not re-run the analyzer, does not preserve the original framing. It treats the correction as the new ground truth for the rest of the screen session and re-queries tools / re-drafts as needed. The original `request_payload.summary` and `information_retrieval_prompt` in the DB stay unchanged (the freshness guarantee — agent session state never writes back to the consent row); the corrected context lives only in the in-memory conversation history of the current screen session.
- **OTP is a data-extraction step, not a terminal step**: when the agent recognizes an OTP fetch intent, it calls a `request_otp` tool — same shape as the device tools (`get_calendar_events`, `search_contacts`, `get_time`), just with the value supplied by the owner instead of by a device API. The frontend renders the tool call as a step row with an inline input bubble (numeric keypad + native OTP auto-fill: iOS `oneTimeCode` auto-fill, Android SMS retrieval) plus a **Done** button and a **Can't provide** button.
  - **The agent loop is FROZEN at the OTP step until the owner explicitly acts.** No token streaming. No auto-proceed to drafting. No fallback after N seconds. The agent stays paused on the deferred `request_otp` tool indefinitely until either Done or Can't provide is tapped.
  - **General principle (all human-intervention steps): the agent loop waits indefinitely on ANY tool call that requires human intervention until the frontend posts the actual outcome.** This applies to `request_otp`, to any tool that returns `permission_denied` and surfaces an inline grant bubble (the frontend holds the result while the owner is in Settings — agent must keep waiting), and to any future tool that requires owner action. The defensive per-turn `TOOL_RESULT_TIMEOUT_S` does NOT fire on these — it would auto-fall-through to drafting and break the contract. Cancelation is via SSE disconnect (owner closes the screen), never via timeout.
  - **Done has two paths**:
    - **Autofill path (auto-Done)**: when the OS supplies the full OTP via native auto-fill (iOS `oneTimeCode` suggestion tapped, or Android SMS Retriever fires), the bubble treats the autofill as the owner's explicit Done and submits immediately — no extra tap required. The autofill action itself is the explicit gesture.
    - **Manual path (explicit Done tap)**: when the owner types the OTP digit-by-digit, the Done button must be tapped to submit. No auto-submit on character-by-character input.
  - On **Done** (either path): the value POSTs back as the tool result, the agent resumes and drafts the response normally (e.g., *"Your OTP is XXXXXX"*), the owner approves the draft, and the response is delivered through the channel-handler path.
  - On **Can't provide**: a refusal signal POSTs back as the tool result, the agent resumes and drafts a polite refusal (*"I can't provide this"* or similar), the owner approves that draft, and the refusal is delivered.
  - **The OTP TextInput is rendered INLINE in the bubble** (explicit exception to the universal popup rule). Reason: native OTP auto-fill needs the input focused immediately on bubble appearance to attach iOS `oneTimeCode` suggestion bar / Android SMS Retriever — wrapping in a tap-to-open popup adds a click before autofill can fire and degrades the OTP UX. The bubble shows: inline OTP TextInput (numeric keypad + `autoFocus` + autofill props) + outlined "Can't provide" (left) + filled-green "Done" (right). Autofill drop-in auto-Done fires from the inline input.
  - **OTP visibility**: the digits show in the step row after submission so the owner can verify what got captured before approving the draft. Per the freshness guarantee, the OTP value lives only in the in-memory agent loop for the screen session — it never persists to DB or logs. The final approved draft (which contains the OTP wrapped in natural language) IS stored as `delivered_response`, the same as any other consent response.
- **Conversation context is lazy-loaded, not eager**: the consent screen does NOT pre-fetch the transcript or WhatsApp thread on open. The transcript view is collapsed by default; the payload only carries a reference (transcript id + snapshot seq for calls, whatsapp_conversation_id + last_message_id for WhatsApp). The actual fetch happens only when the owner taps to expand the transcript section. Eager loading the conversation on every screen open is wasteful — most consents never need the transcript opened.
- **Transcript rendering is a capability owned by the consent component**: the on-demand transcript fetch + render is part of the consent component's owner-facing surface, not a separate generic loader. Given the channel-specific reference inside the payload, the consent component returns the conversation in a render-ready shape (call segments for calls, WhatsApp messages for WhatsApp) when the owner opens the dropdown. This keeps the channel-aware loader logic inside the same component that prepared the payload — one place owns "how to read a consent request's conversation context."

---

## Goal

Give the consent-request component **true agentic intelligence**. Phase 3 adds a backend agent that plans, calls on-device tools, observes results, and drafts a response token-by-token — implemented using **Pydantic AI** as the orchestration framework.

Pydantic AI is chosen specifically because it ships a first-class primitive (`ExternalToolset` + `DeferredToolRequests`) for the exact architecture consent needs: **tools defined on the backend but executed on the mobile device**. The agent pauses when it wants to call a tool, the backend emits the call over SSE, the mobile client runs the tool, the result gets POSTed back, and the agent resumes. This pattern is provider-agnostic, Pydantic-native (fits our existing stack), and keeps the consent component lightweight — the framework handles the LLM conversation, tool-arg validation, and streaming; our code handles the device bridge, the UI, and the drafting-only scope constraint.

The owner sees the agent working on their behalf in real time — issuing tool calls against the device (contacts, calendar, messages, location, time), drafting a response token-by-token, and accepting conversational refinements that can re-query data when the owner corrects context.

After Phase 1 (extract reusable component) and Phase 2 (multi-channel support), the consent functionality is fully working across all three channels with the existing one-shot draft UI. Phase 3 upgrades that UI AND introduces the backend agent — both are required for the agent to actually be intelligent (re-plan on clarification, re-query on correction). The existing consent component contract and channel dispatch stay intact.

## What Ships at the End of Phase 3

- The consent screen behaves like a conversational tool-using agent the owner is collaborating with
- A **Data Source Registry** (`data_source_registry.py`) that declares all available data sources (contacts, calendar, OTP) with their types, permissions, tool schemas, and UI labels — extensible foundation for future sources
- A backend **Pydantic AI agent** exposed as `POST /consent-request/{id}/agent-turn` (SSE) that plans, dispatches device-tool calls (derived from the registry), and streams the final draft
- A frontend **device-tool bridge** that executes agent-issued tool calls on-device and POSTs results back via `/tool-result`, re-entering the agent via Pydantic AI's `DeferredToolResults`
- An **Agent Activity Panel** that renders a live projection of the agent's tool trace (fetching data, asking for permissions, drafting, refining)
- **Token-by-token streaming** of the draft response into a draft area via SSE
- A **two-button input model**: one input box, "Ask agent" (refine the draft), "Send to caller" (send the draft as final response)
- **Conversational refinement loop** that re-enters the agent loop: the owner can type "the name is Jon" and the agent actually re-queries contacts with the corrected name — not just re-words the previous empty result
- **Inline permission grant flow**: missing device permissions appear as `?` step rows with `[Grant Permission]` buttons inside the activity panel
- **Mid-stream cancel**: typing in the input or tapping the draft area cancels the active stream (aborts any pending tool call)
- All three channels (inbound, outbound, WhatsApp) use the new agentic screen
- Special cases handled: OTP-only bypasses the agentic flow; all-unknown sources show a blank input

## Data Source Registry (Phase 3 addition)

Phase 3 introduces a **Data Source Registry** — a structured module that declares all available data sources the consent agent can use. The consent agent's tools are derived FROM the registry, not hardcoded.

**Why now (not later):** Building the registry with the 3 existing sources creates the extensible foundation. When third-party sources (Gmail, Outlook, Slack) land in a future module, they register into the same registry — no refactoring of the consent agent needed.

**Initial registry (3 sources):**

| Source | Type | Location | Permission | Tool name |
|---|---|---|---|---|
| `device_contact` | Device + DB | Contacts on phone + `contacts` table in Supabase | OS `contacts` permission | `search_contacts` |
| `device_calendar` | Device | Calendar events on phone | OS `calendar` permission | `get_calendar_events` |
| `device_otp` | Device | SMS auto-fill on phone | None (user input) | — (no tool; handled by UI) |

**Registry structure:**

```python
# app/services/data_source_registry.py

@dataclass
class DataSourceDefinition:
    key: str                          # e.g. "device_contact"
    name: str                         # e.g. "Contacts"
    description: str                  # For LLM context
    source_type: str                  # "device" | "db" | "api" | "device+db"
    permission_key: str | None        # OS permission key, or None
    tool_name: str | None             # Agent tool name, or None (OTP has no tool)
    tool_args_schema: type[BaseModel] | None  # Pydantic schema for tool args
    ui_label: str                     # "Reading your contacts" for step rows
    enabled: bool = True

DATA_SOURCE_REGISTRY: dict[str, DataSourceDefinition] = {
    "device_contact": DataSourceDefinition(...),
    "device_calendar": DataSourceDefinition(...),
    "device_otp": DataSourceDefinition(...),
}
```

**How the consent agent uses it:**
- `consent_agent_tools.py` reads `DATA_SOURCE_REGISTRY` to build the `ExternalToolset` dynamically — one tool per source that has a `tool_name`
- `TOOL_LABELS` map is derived from `ui_label` fields
- `consentAgentTools.ts` (frontend) has a matching executor per `tool_name`
- Adding a new source = register it + add a frontend executor. No changes to the agent loop, prompt, or endpoints.

**What the registry does NOT do (deferred to the future data source module):**
- No dynamic discovery (sources are statically registered)
- No per-user availability checks (the agent + tool executors handle permissions inline)
- No third-party OAuth flows (Composio integration is out of scope)

## What Is NOT in Scope

- **Third-party data sources** (Gmail, Outlook, Slack via Composio) — they will register into the Data Source Registry when built, but are not implemented in Phase 3
- **LangGraph / checkpointed orchestration** — the consent agent is in-memory and ephemeral. Pydantic AI is intentionally used without a persistent message-history store; the main product's LangGraph orchestration is a separate concern. Phase 4 will shift drafting to the main agent.
- **Cross-turn server-side memory** — each `/agent-turn` request is stateless on the server; the `conversation_history` passed in by the frontend (session-scoped, in-memory) is the only between-turn state
- **General-purpose chat** — the agent is drafting-only. It exists to produce a response the owner can send to the requester; it does not answer off-topic questions or act as a general assistant

---

## Freshness Guarantee (Non-Negotiable)

**Every time the consent request screen opens, the agent starts from zero.** No prior conversation, no prior drafts, no prior tool results.

- Nothing about the agent is persisted to Postgres or any other DB. The `consent_requests` table holds only the immutable request payload from the channel agent (Phase 1); the agent's turn state and refinement dialog are never written to it.
- The backend agent's `loop_state` exists only for the duration of a single `/agent-turn` SSE stream. It is created when the stream opens and released when the stream closes (normal end, error, cancel, or client disconnect).
- The frontend `conversation_history` lives in the `useConsentAgent` hook's `useRef` and is cleared on component unmount. Closing the consent screen and reopening it gives a blank agent with no memory of the previous session.
- No telemetry or debug path captures the conversation_history. Logging is limited to tool-call names, outcomes, and timings — never the message content.

This is tested in verification criterion 7b.

---

## The Agentic Consent Screen

### Visual Layout

```
┌──────────────────────────────────────────────────────────┐
│  ← Information Consent                        [Decline]  │
│  [Channel badge] John Smith • +1234567890                │
│  John is asking for your schedule this Thursday          │
│  ▸ View conversation                                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  CHAT THREAD (scrollable)                                │
│                                                          │
│  ┌─ Agent ──────────────────────────────────────────┐   │
│  │ ✓ Reading your contacts                          │   │
│  │ ✓ Checking your calendar                         │   │
│  │                                                   │   │
│  │ Hi John, I have a meeting at 3pm Thursday and    │   │
│  │ I'm free after 5. Want to grab coffee then?      │   │
│  │                                        [Approve] │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
├──────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────┐  [▶]  │
│  │ Type your instruction to the agent...        │       │
│  └──────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────┘
```

**Three zones, top to bottom:**
1. **Header** — channel badge + requester identity + the original request summary + collapsible transcript. **[Decline]** button in the top-right corner.
2. **Chat thread** (scrollable) — steps with streamed results, draft cards with [Approve], owner messages. Everything streams token-by-token like ChatGPT.
3. **Input box** (bottom, pinned) — single text input + send button. Disabled while agent is working.

### Step-by-Step Streaming with Results

Every step the agent takes shows its label AND its result streamed as tokens — the owner sees what the agent found in real time:

```
✓ Checking your calendar
  You have a meeting "Project Review" at 10:00 PM tomorrow.

✓ Drafting response
  ┌─────────────────────────────────────────────────────────┐
  │ I'm not available at 10 PM tomorrow — I have a meeting │
  │ at that time.                        [Approve & Send]   │
  └─────────────────────────────────────────────────────────┘
```

- Tool steps: label + result text (muted, indented, streamed)
- Draft step: label + [Approve] card (streamed into the card)
- Failed steps: label + error text
- On refinement: new "Drafting response" step with new [Approve] card; previous card loses button

**SSE events for step results:**

```
event: step
data: {"step_id": "tc_abc", "status": "completed", "label": "Checking your calendar"}

event: step_result
data: {"step_id": "tc_abc", "token": "You have a meeting"}

event: step_result
data: {"step_id": "tc_abc", "token": " \"Project Review\" at 10 PM."}
```

The backend generates step result tokens by asking the LLM to summarize the tool output in one sentence before proceeding to the next step. This is a lightweight LLM call per tool result — not a full draft.

### Step Row Icons

| Icon | Meaning | Example labels |
|---|---|---|
| `⟳` | In progress | "Reading your device calendar..." |
| `✓` | Completed | "Read your contacts" |
| `⚠` | Failed (non-blocking) | "Couldn't read your contacts" |
| `⊘` | Cancelled / skipped | "Drafting cancelled by you" |
| `?` | Needs the owner's action | "Need calendar access [Grant Permission]" |

Permissions are handled inline by the tool executors — not pre-classified. When the agent calls a tool that needs OS permission, the executor checks + requests permission on the spot. If granted, the tool runs. If denied, the tool returns `permission_denied` and the agent drafts without that data. On iOS with blocked permissions, the executor opens the device Settings page.

### How the Agent Run Works

**No pre-classification phase.** The frontend does NOT read the `data_sources` or `missing_data_sources` arrays from the request payload. The agent decides autonomously which tools to call based on the `information_retrieval_prompt`. This makes the agent self-sufficient — it works correctly regardless of what the voice service puts in `data_sources`, even if it's empty or filtered incorrectly.

**When the screen opens**, the agent starts immediately:

The frontend opens an SSE stream: `POST /consent-request/{id}/agent-turn`. The backend runs a Pydantic AI agent configured with:
- A drafting-only system prompt (L6)
- An `ExternalToolset` declaring the 5 device tools (`search_contacts`, `get_calendar_events`, `get_messages`, `get_current_location`, `get_time`) — marked as deferred (executed off-process)
- An in-process tool `draft_response(text)` — the terminal action

Per-turn flow:

1. Backend calls `agent.iter(user_prompt, message_history=conversation_history)` inside an async generator
2. Pydantic AI streams the model's response. When the model emits a deferred tool call, `iter()` yields a `DeferredToolRequests` → our code emits `event: tool_call` over SSE with the `tool_call_id` + validated args + user-facing label
3. Frontend executes the tool on-device and POSTs `{tool_call_id, status, result}` to `/tool-result`
4. Backend resolves the corresponding `asyncio.Event`; our code resumes the agent via `agent.iter(message_history=..., deferred_tool_results=DeferredToolResults(calls={id: result}))`
5. Pydantic AI re-enters with the tool result folded into the conversation; the model plans the next step
6. When the model calls `draft_response`, its tokens are streamed as `PartDeltaEvent`s → we forward each delta as `event: token` SSE frames → final `event: done`
7. Whole turn bounded by `MAX_ITERATIONS = 5` (our counter wrapped around `agent.iter()`)

Each `tool_call` produces a step row: `⟳ Reading contacts...` → `✓ Read contacts`. These rows are a live projection of the agent's real dispatch trace. Pydantic AI handles: LLM provider abstraction, tool-arg validation via Pydantic schemas, message-history rehydration across refinements, token streaming. Our code handles: the SSE bridge, the `tool_call_id` correlation dict with asyncio.Event, the MAX_ITERATIONS counter, the drafting-only system prompt, the FastAPI endpoints.

**Phase C — Owner interaction (dual-output model)**

After the initial draft completes, the owner types in the input box. The backend classifies the owner's intent before acting:

**Mode A — Draft instruction** (owner wants to change the response):
- Detected by: directive language ("add", "change", "remove", "also inform", "include")
- Agent re-enters the loop, may call tools, produces a NEW draft card with [Approve]
- Previous draft card loses [Approve] (becomes history)
- SSE events: `tool_call` → `token` → `done` with `final_text`

**Mode B — Question to the agent** (owner wants information, not a draft change):
- Detected by: question language ("can you check", "is there", "am I available", "let me know")
- Agent calls tools if needed, then answers the owner in a regular chat message
- The current draft card with [Approve] **stays unchanged**
- SSE events: `tool_call` → `agent_message` (new event type — chat text to the owner)
- The owner can then decide to incorporate the answer into the draft via a Mode A instruction

**`agent_message` SSE event (new):**
```
event: agent_message
data: {"text": "You're free between 8-10 PM, no meetings scheduled."}
```
The frontend renders this as a regular agent chat bubble (no [Approve] card). The previous draft card remains visible and approvable.

**Ambiguous input:** defaults to Mode A (safer — always produces a draft).

This dual-output model lets the owner ask the agent questions ("am I free between 8-10?"), get answers in the chat, and then decide whether to change the draft — without losing the current draft every time they interact.

### Human–Agent Interaction Lifecycle (complete sequence)

The screen is a chat between the owner and the agent. The agent proposes drafts and answers questions; the owner approves when ready.

```
Screen opens (fresh agent, zero memory)
  │
  ├── Agent turn 1 — initial draft (Mode A):
  │     ✓ Checking your calendar
  │     ┌─────────────────────────────────────────────────┐
  │     │ I'm not available at 10 PM tomorrow — I have   │
  │     │ a meeting at that time.          [Approve]      │
  │     └─────────────────────────────────────────────────┘
  │
  │     OWNER DECIDES:
  │
  ├── [Approve] → sends draft to requester. Done.
  ├── [Decline] → declines consent request. Done.
  │
  ├── Owner asks a question (Mode B):
  │     Owner types: "can you check if I'm free between 8-10 PM?"
  │     │
  │     ├── Agent answers the owner (regular chat text):
  │     │     ✓ Checking your calendar
  │     │     "You're free between 8-10 PM, no meetings scheduled."
  │     │
  │     │     Draft card STAYS from turn 1 — still has [Approve]:
  │     │     ┌─────────────────────────────────────────────────┐
  │     │     │ I'm not available at 10 PM tomorrow — I have   │
  │     │     │ a meeting at that time.          [Approve]      │
  │     │     └─────────────────────────────────────────────────┘
  │     │
  │     │     OWNER DECIDES AGAIN
  │
  ├── Owner gives a draft instruction (Mode A):
  │     Owner types: "add that I'm available between 8-10 PM instead"
  │     │
  │     ├── Agent produces NEW draft (replaces the old one):
  │     │     ┌─────────────────────────────────────────────────┐
  │     │     │ I'm not available at 10 PM tomorrow due to a   │
  │     │     │ meeting. However, I'm free between 8-10 PM     │
  │     │     │ if that works for you!          [Approve]       │
  │     │     └─────────────────────────────────────────────────┘
  │     │
  │     │     Previous draft card loses [Approve] (history)
  │
  └── At any point: [Decline] in header is always available
```

**Key guarantees:**
- The agent NEVER sends a response to the requester on its own. Only the [Approve] button does that.
- Only the **latest** draft card has an [Approve] button. Previous drafts become history.
- Mode B (question) never changes the draft — it only adds a chat message to the owner.
- Mode A (instruction) always produces a complete replacement draft that addresses the main request + all prior instructions.
- The draft card text is editable — the owner can tap into it and modify before approving.
- The input box is disabled while the agent is working.
- Every turn carries the main request context so the agent never loses sight of what the caller asked.

### Chat-Style Input Model

The input box at the bottom has ONE purpose: send instructions to the agent. There is no "Send to caller" button next to the input.

- **Input box** → sends refinement instructions to the agent
- **[Approve]** (on the agent's draft bubble) → sends the draft to the requester
- **[Decline]** (in the header) → declines the entire consent request

**Input box states:**

| State | Input box | Send [▶] |
|---|---|---|
| Agent working (tool calls / drafting) | **Disabled** | Disabled |
| Agent finished (draft bubble with [Approve] visible) | **Enabled** | Enabled if text present |
| After owner sends instruction | Clears, disables (agent starts new turn) | — |

**Why this replaces the two-button model:** The previous "Ask agent" / "Send to caller" split confused users about what each button does. The new model is unambiguous: type = talk to agent; [Approve] on the bubble = send the response. Blocking input during agent work creates clean turn-taking.

### Mid-Turn Behavior

The input is disabled while the agent works — no mid-stream interruption is needed.

- On agent error or timeout (60s): the chat shows an error message and the input re-enables.
- **[Decline]** is always active in the header, even during agent work. Tapping it cancels the SSE, declines the request, and closes the screen.
- There is no "Take over" button. If the owner wants to change course, they wait for the agent to finish, then type a new instruction.

### Special Cases

**OTP-only** (`data_sources == ["device_otp"]`): bypass the agentic UI entirely. Show the existing OTP-specific input (centered, number-pad, auto-fill enabled). Two buttons: "Send to caller" and "Decline". This preserves the existing OTP flow.

**All-unknown sources** (every source in `data_sources` is not in `{device_calendar, device_contact, device_otp}`): show a brief activity row `⊘ I don't have access to any of the requested data — please write your response manually`. Draft area is empty and editable from the start. "Ask agent" is disabled.

---

## Backend Architecture

### The Agent Loop Endpoints

The backend runs a **Pydantic AI agent** configured for remote (on-device) tool execution. The frontend is a tool-executor + renderer.

**Why Pydantic AI**: It ships a first-class primitive for our exact pattern — tools defined on the backend but executed outside the agent process. An `ExternalToolset` pre-declares the device tools; when the model calls one, the agent's `iter()` yields a `DeferredToolRequests` object carrying the `tool_call_id` + validated args; we emit that over SSE; the frontend executes on-device and POSTs the result back; we resume the agent by re-invoking `iter()` with `deferred_tool_results=DeferredToolResults(calls={id: result})`. This is provider-agnostic (Claude, OpenAI, Gemini, local models via a one-line model-string change), Pydantic-native (fits our existing stack), and production-proven (used by OpenBB and others).

**Why on the backend and not the frontend**: Planning requires seeing tool results across turns and feeding them back into the next model call. That is a multi-step server-side conversation. The frontend is still involved because the tools run on-device — hence the side-channel POST pattern below.

#### Endpoint 1: `POST /consent-request/{id}/agent-turn` (SSE)

Main agent-loop endpoint. Opens an SSE stream; emits `tool_call`, `tool_result_ack`, `step`, `token`, `done`, `error` events.

Request body:
```json
{
  "instruction": "make it shorter",          // optional; absent on initial draft
  "conversation_history": [                  // optional; in-memory session log
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
  ]
}
```

SSE event types:

```
event: tool_call
data: {"tool_call_id": "tc_abc123", "name": "search_contacts", "input": {"query": "John", "fuzzy": false}, "label": "Reading your contacts"}

event: tool_result_ack
data: {"tool_call_id": "tc_abc123", "status": "ok"}

event: step
data: {"step_id": "tc_abc123", "status": "completed", "label": "Read your contacts"}

event: token
data: {"token": "Hi"}

event: token
data: {"token": " John"}

event: step
data: {"step_id": "draft", "status": "completed"}

event: done
data: {"final_text": "Hi John, I have a meeting at 3pm Thursday..."}
```

For a refinement, an additional step row fires first: `{"step_id": "refine", "status": "started", "label": "Refining: \"<instruction>\""}`. Any re-queried tools nest under it.

#### Endpoint 2: `POST /consent-request/{id}/tool-result` (JSON side channel)

The frontend POSTs here after executing a tool on-device. The backend agent loop is blocked on an asyncio primitive keyed by `tool_call_id`; this POST unblocks it.

Request body:
```json
{
  "tool_call_id": "tc_abc123",
  "status": "ok",                            // ok | error | permission_denied | timeout
  "result": [{"name": "John Doe", "phone": "+1555..."}]
}
```

Returns 200 with empty body. The actual result feeds into the next Claude turn via the still-open SSE.

#### Endpoint 3: `DELETE /consent-request/{id}/agent-turn`

Explicit cancel signal from the frontend when the owner interrupts.

### The Tool Set (Pydantic AI `ExternalToolset`)

Declared once as Pydantic BaseModels + an `ExternalToolset` in our code. Pydantic AI handles arg validation, JSON-schema generation for the model, and the tool-call emission.

```python
from pydantic import BaseModel
from pydantic_ai.toolsets import ExternalToolset

class SearchContactsArgs(BaseModel):
    query: str
    fuzzy: bool = False

class GetCalendarEventsArgs(BaseModel):
    date_from: datetime
    date_to: datetime

class GetMessagesArgs(BaseModel):
    contact_query: str
    limit: int = 20

class GetCurrentLocationArgs(BaseModel):
    pass

class GetTimeArgs(BaseModel):
    timezone: str | None = None

device_tools = ExternalToolset(
    tools=[
        {"name": "search_contacts",      "args_model": SearchContactsArgs,
         "description": "Search the owner's device contacts by name."},
        {"name": "get_calendar_events",  "args_model": GetCalendarEventsArgs,
         "description": "Read calendar events within a date range."},
        {"name": "get_messages",         "args_model": GetMessagesArgs,
         "description": "Recent messages with a given contact."},
        {"name": "get_current_location", "args_model": GetCurrentLocationArgs,
         "description": "Owner's current location."},
        {"name": "get_time",             "args_model": GetTimeArgs,
         "description": "Current time, optionally in a specific timezone."},
    ]
)

TOOL_LABELS = {
    "search_contacts":      "Reading your contacts",
    "get_calendar_events":  "Checking your calendar",
    "get_messages":         "Reading recent messages",
    "get_current_location": "Getting your location",
    "get_time":             "Checking the time",
}
```

`draft_response` is implemented as an in-process Pydantic AI tool (not deferred) — it returns a string the agent then streams back. When the model calls it, Pydantic AI yields `PartDeltaEvent`s with token deltas that we forward as SSE `event: token` frames.

### The Agent (Pydantic AI `Agent`)

```python
from pydantic_ai import Agent

consent_agent = Agent(
    model='anthropic:claude-sonnet-4-6',  # swap via env / config
    toolsets=[device_tools],
    system_prompt=build_drafting_only_system_prompt(request_payload),
    # Pydantic AI ships: provider abstraction, arg validation,
    # message-history rehydration across turns, token streaming.
)

@consent_agent.tool
async def draft_response(ctx, text: str) -> str:
    """Terminal action: produce the response text the owner will send."""
    return text
```

The agent is re-instantiated per turn with the turn-specific system prompt and the session's `conversation_history` passed as `message_history` to `agent.iter()`.

### Loop Bounds and Timeouts

| Bound | Value | Behavior on hit |
|---|---|---|
| `MAX_ITERATIONS` | 5 | After the 5th tool call, Claude is forced to produce a draft with whatever data it has |
| `TOOL_RESULT_TIMEOUT` | 10s | If the frontend doesn't POST a result, treat as `{status: "timeout"}` and continue |
| `TURN_TOTAL_TIMEOUT` | 60s | Hard ceiling on a single `/agent-turn`; yields `event: error` and ends the stream |

### Per-Request In-Memory State

Lives only for the duration of a single `/agent-turn` stream:

```python
loop_state = {
    "pending_tool_calls": dict[tool_call_id, asyncio.Future],
    "conversation": list[Message],   # Claude messages for this turn
    "iterations": int,
    "cancelled": bool,
}
```

Nothing persists to Postgres. The session-scoped `conversation_history` passed in from the frontend is the only across-turn memory, and it stays on the device.

### Frontend State Machine (tool-executor + renderer)

```
Screen opens
  │
  ├── Fetch full request_payload via GET /consent-request/{id}/details
  │
  ├── Phase A — Permission classification (NO data fetching):
  │     For each source in conversation_context.data_sources:
  │       - If source ∈ {device_calendar, device_contact, device_otp}:
  │           check permissionService.checkPermission(source)
  │             granted → tool enabled
  │             denied  → emit "? Need [source] permission"
  │             blocked → emit "? [Source] blocked — Open Settings"
  │       - Otherwise: silent skip
  │
  ├── Phase B — Open agent loop:
  │     POST /agent-turn  (SSE, body: { instruction: null })
  │     │
  │     ├── event: tool_call { tool_call_id, name, input, label }
  │     │     → add "⟳ <label>" step row
  │     │     → if tool blocked (permission denied / unknown):
  │     │         POST /tool-result { tool_call_id, status: "permission_denied" }
  │     │     → else execute on-device:
  │     │         search_contacts      → Contacts API
  │     │         get_calendar_events  → Calendar API
  │     │         get_messages         → SMS / local store
  │     │         get_current_location → Location API
  │     │         get_time             → local Date + IANA tz
  │     │       on success: POST /tool-result { tool_call_id, status: "ok", result }
  │     │       on error:   POST /tool-result { tool_call_id, status: "error", error }
  │     │
  │     ├── event: step { step_id: tool_call_id, status: "completed" }
  │     │     → update row to "✓ <label>"
  │     │
  │     ├── event: token { token }
  │     │     → append to draftPartsRef; update draft area state
  │     │
  │     └── event: done { final_text }
  │           → set draft area to final_text
  │           → enable "Send to caller" / "Ask agent" buttons
  │           → push assistant turn into session-scoped conversation_history
  │
  ├── Owner interacts:
  │     - Edits draft directly → input is now just for refinement
  │     - Types in input + Ask agent:
  │         → push {role: "user", content: instruction} onto conversation_history
  │         → open NEW SSE: POST /agent-turn
  │             body: { instruction, conversation_history }
  │         → same tool_call / token / done cycle; tokens REPLACE draft area
  │         → clear input box on done
  │     - Owner grants previously-denied permission:
  │         → update "?" row to "✓ Granted <source> access"
  │         → tool becomes available for the next agent turn
  │           (no automatic re-draft; owner asks the agent to re-try if desired)
  │
  ├── Mid-stream cancel (Take over / tap draft / type in input):
  │     → close SSE
  │     → POST DELETE /agent-turn
  │     → ignore any pending tool_result
  │
  └── Owner taps "Send to caller":
        → POST /consent-response with draft area text + action="approved"
        → routed to channel-specific delivery (Phase 2 handlers)
```

---

## Layered Architecture

The consent-drafting agent is built as 8 layers. Each is independently testable and implementation tasks are aligned to them.

| # | Layer | What it does | Primary files |
|---|---|---|---|
| L1 | **Wire / transport** | SSE downstream (`tool_call`, `token`, `done`, `error`); plain POST upstream (`tool_result`, cancel). Correlation via `tool_call_id` UUIDs | `react-native-sse` (new dep), `app/routes/consent_request.py` |
| L2 | **Agent orchestration (Pydantic AI)** | Pydantic AI `Agent` with `ExternalToolset` for deferred device tools + in-process `draft_response`. Our glue: iterate `agent.iter()`, forward `DeferredToolRequests` → SSE, forward `PartDeltaEvent` tokens → SSE, resume with `DeferredToolResults` on POST, bound with `MAX_ITERATIONS=5` | `app/services/consent_agent_service.py` (new, ~150L glue over Pydantic AI) |
| L3a | **Data Source Registry** | Central registry of all data sources (`DataSourceDefinition` entries). Declares source type, permission key, tool name, arg schema, UI label. The consent agent's tools are derived from this registry, not hardcoded | `app/services/data_source_registry.py` (new) |
| L3b | **Tool declarations (derived from registry)** | Reads `DATA_SOURCE_REGISTRY` to dynamically build Pydantic AI `ExternalToolset` + `TOOL_LABELS`. Adding a new source = register it in L3a; L3b auto-picks it up | `app/services/consent_agent_tools.py` (new, reads from L3a) |
| L4 | **Device-bridge executors** | Frontend functions that run tools on-device using native APIs. Permission-gated before execution. Canonical return shape `{status, result \| error}` | `trybot-ui/lib/consentAgentTools.ts` (new) |
| L5 | **State (ephemeral)** | Backend per-request `loop_state` dict (pending tool-call Events, `conversation_history`, iteration counter, cancelled flag); frontend per-session `conversationHistoryRef`. Neither persists to DB | Lives inside L2 + L7 |
| L6 | **Scope constraint / system prompt** | Drafting-only system prompt passed to the Pydantic AI `Agent` on every turn. Interpolates request context, requester name, channel. Keeps agent on-task via prompt discipline | `app/services/consent_agent_prompt.py` (new) |
| L7 | **UI projection** | Renders step rows + streaming draft; owns the two-button input | `trybot-ui/hooks/useConsentAgent.ts` + `components/AgentActivityPanel.tsx`, `AgentStepRow.tsx`, `DraftArea.tsx` |
| L8 | **Surface integration** | Replaces existing consent screens; FCM → screen routing stays; "Send to caller" plumbs into Phase 2 handlers | `trybot-ui/components/InformationConsentScreen.tsx`, `ConsentRequestDetailScreen.tsx`, `store/informationConsentSlice.ts` |

**Dependency chain**: L1 → L2 → L3 → (L4 + L6) → L7 → L8. L5 is embedded in L2 + L7. Tests land alongside each layer. Pydantic AI's provider abstraction removes the need for a separate "LLM provider abstraction" layer — it is the abstraction.

**The drafting-only contract** (L6): The agent's sole objective is to produce a response the owner can send to the requester. It uses other tools only when needed to ground the draft in real data; it redirects off-topic instructions back to drafting; it never explains its plan or asks clarifying questions. This is enforced by the system prompt passed to the Pydantic AI `Agent` on every turn. Combined with the explicit `draft_response` terminal tool and the `MAX_ITERATIONS=5` cap in our glue, the agent stays bounded and on-task.

---

## Implementation Tasks

### 3.1 — Backend: Device-Bridge Spike (de-risk the hard part first)
*Dependencies: Phase 1 + Phase 2 complete*

The SSE `tool_call` → frontend-executes → POST-result → SSE-resumes round-trip is the real engineering risk in Phase 3. Before the full build, prove the Pydantic AI `ExternalToolset` + `DeferredToolRequests` + `DeferredToolResults` round-trip works end-to-end with **one toy tool** (`get_time`). This is a throwaway spike; the code lands on a spike branch and is discarded.

**File**: `trybot-api/app/routes/consent_request.py` (spike) + `trybot-ui/hooks/useConsentAgent.ts` (spike)

| Task | Hours |
|------|-------|
| Install `pydantic-ai` + `react-native-sse`; verify both build | 0.5 |
| Minimal Pydantic AI agent with one-tool `ExternalToolset` (`get_time`) and `draft_response` | 1 |
| `/agent-turn` SSE endpoint wrapping `agent.iter()`: forward `DeferredToolRequests` → SSE `tool_call`, forward `PartDeltaEvent` → SSE `token` | 1 |
| `/tool-result` POST endpoint + asyncio.Event correlation keyed by `tool_call_id`; re-invoke `agent.iter(deferred_tool_results=...)` on POST | 1 |
| Prototype frontend hook: open SSE, receive `tool_call`, execute `get_time`, POST result, await next event, render tokens | 1 |
| Verify end-to-end over ngrok + device build (not simulator) | 1 |
| Document findings: wire timings, Pydantic AI gotchas, any surprises to inform 3.2 | 0.5 |
| Throw away spike code after sign-off | 0 |
| **Subtotal** | **6** |

### 3.2 — Backend: Pydantic AI Agent + Tool Set + Prompt + FastAPI Wiring
*Dependencies: 3.1 (spike signed off)*

Production implementation of L2 + L3 + L6. Pydantic AI handles the agent loop, tool-arg validation, message-history threading, and token streaming. Our code wires it to FastAPI SSE, the device-bridge POST endpoint, and the drafting-only system prompt.

**Agent scope boundary — drafting-only**

The consent agent's single objective is to **produce a response the owner can send to the requester**. Every tool call, every reply, every refinement must serve that objective. The drafting-only system prompt (L6) — passed to the Pydantic AI `Agent` on every turn — constrains the agent to:

- Always terminate by calling `draft_response`
- Use other tools ONLY when needed to ground the draft in real data; minimize tool calls
- If the owner's instruction is off-topic (general question, chit-chat, request for advice), briefly acknowledge in the draft and return to drafting: "I'm focused on drafting your reply to {requester}. Here's a revised draft based on what you said…"
- Never produce meta-commentary, never explain its plan, never ask clarifying questions — if context is ambiguous, make a reasonable drafting choice and let the owner correct via refinement

**Files**: `trybot-api/app/services/consent_agent_service.py` (new, ~150L glue) + `consent_agent_tools.py` (new) + `consent_agent_prompt.py` (new) + `app/routes/consent_request.py`

| Task | Hours |
|------|-------|
| **L3** — Pydantic `BaseModel` arg schemas for the 5 device tools + `ExternalToolset` declaration + `TOOL_LABELS` map | 1 |
| **L3** — `draft_response` as an in-process Pydantic AI tool (`@agent.tool`) that returns a string (the final text); this is what streams as `PartDeltaEvent` tokens | 0.5 |
| **L6** — drafting-only system prompt template with request context, requester name, channel, current instruction interpolation | 1.5 |
| **L2** — `ConsentAgentService` — instantiate Pydantic AI `Agent` per turn with turn-specific prompt; iterate `agent.iter(message_history, deferred_tool_results?)` | 1.5 |
| **L2** — translate Pydantic AI events to SSE: `DeferredToolRequests` → `event: tool_call` (with `TOOL_LABELS` lookup); `PartDeltaEvent` → `event: token`; final → `event: done` | 1.5 |
| **L5** — per-request in-memory `loop_state` dict: pending `asyncio.Event` per tool_call_id, iteration counter, cancelled flag, accumulated `message_history` | 0.5 |
| **L1** — `/tool-result` POST endpoint: look up Event by `tool_call_id`, attach result, set Event, return 200 | 0.5 |
| **L2** — bounded iteration: `MAX_ITERATIONS=5` counter around `agent.iter()`; on hit, force a final `draft_response` call with accumulated context | 0.5 |
| **L2** — timeouts: `TOOL_RESULT_TIMEOUT=10s` via `asyncio.wait_for` on the Event; `TURN_TOTAL_TIMEOUT=60s` on the whole turn | 0.5 |
| **L1 / L2** — cancel handling: detect client disconnect + `DELETE /agent-turn`; cancel pending Events; release state | 1 |
| **L2** — refinement support: accept `conversation_history` in the POST body; thread it into `agent.iter(message_history=...)` so Pydantic AI sees the prior dialog | 0.5 |
| **L3** — tool-result normalization: `{status: error \| permission_denied \| timeout}` → structured Pydantic AI `ToolReturnPart` so the agent can reason about missing data | 0.5 |
| **L1** — `GET /consent-request/{id}/transcript` — transcript segments up to `transcript_snapshot_seq` (calls) or WhatsApp messages up to `created_at` | 1 |
| Unit tests: mock Pydantic AI agent iteration, deferred-tool resume, timeout path, cancel path, MAX_ITER cap, drafting-only prompt redirect behavior | 3 |
| **Subtotal** | **13.5** |

### 3.2a — Backend: Analyzer Awareness Refinement (legacy from prior plan)
*Dependencies: 3.2*

The Phase 1 analyzer already emits `data_sources`. Phase 3 ensures `missing_data_sources` is reliably populated based on per-user device permission state so the frontend classification step has accurate data.

| Task | Hours |
|------|-------|
| Update `InformationRetrievalAnalyzer` LLM prompt to be explicit about the three device sources | 0.5 |
| Wire analyzer to emit `missing_data_sources` from per-user device permission state | 1 |
| Unit tests | 0.5 |
| **Subtotal** | **2** |

### 3.2 — Backend: Analyzer Awareness Refinement
*Dependencies: 3.1*

The Phase 1 analyzer already emits `data_sources`. Phase 3 ensures `missing_data_sources` is reliably populated based on per-user device permission state, so the agentic UI's classification step has accurate data.

| Task | Hours |
|------|-------|
| Update `InformationRetrievalAnalyzer` LLM prompt to be explicit about the three device sources | 0.5 |
| Wire analyzer to emit `missing_data_sources` from per-user device permission state (extend existing `get_user_data_source_availability` helper if present) | 1 |
| Unit tests | 0.5 |
| **Subtotal** | **2** |

### 3.3 — Frontend: Source Classification Helpers
*Dependencies: 3.2a*

| Task | Hours |
|------|-------|
| `classifySources(data_sources)` helper: returns `available` / `missing_connectable` / `missing_blocked` / `unknown` for each source key | 0.5 |
| Update `consentUtils.ts` with classification helpers + label/description map for the three device sources | 0.5 |
| Unit tests | 0.5 |
| **Subtotal** | **1.5** |

### 3.4 — Frontend: Header Sub-Components
*Dependencies: 3.3*

| Task | Hours |
|------|-------|
| `ConsentRequestSummary.tsx` (channel badge, requester, structured prompt) | 1.5 |
| `ConsentTranscriptSection.tsx` (collapsible, fetches transcript on demand via `GET /consent-request/{id}/transcript` using the `transcript_id` + `transcript_snapshot_seq` from request_payload — no inline transcript data stored) | 2 |
| Unit tests | 0.5 |
| **Subtotal** | **4** |

### 3.5 — Frontend: Agentic Consent Screen (The Big One)
*Dependencies: 3.2 (backend agent loop), 3.3 (classification)*

The most substantial frontend phase. Builds the tool-executor + renderer for the agent loop.

| Task | Hours |
|------|-------|
| Install + configure `react-native-sse` (replaces the homegrown XHR-based `lib/sseClient.ts` — the XHR approach doesn't reliably deliver `onprogress` chunk-by-chunk on RN) | 0.75 |
| `useConsentAgent` hook: opens `/agent-turn` SSE, dispatches `tool_call` / `tool_result_ack` / `step` / `token` / `done` / `error` events (forwarded from the backend Pydantic AI agent) to state, handles cancel (replaces the old `useAgenticDraftStream`) | 3 |
| Device tool executors (`lib/consentAgentTools.ts`) — 5 thin wrappers over native APIs: `search_contacts` (Contacts), `get_calendar_events` (Calendar), `get_messages` (SMS/local), `get_current_location` (Location), `get_time` (local Date + IANA tz) | 2.5 |
| Tool-result POST logic: after each on-device execution, POST to `/consent-request/{id}/tool-result` with `{ tool_call_id, status, result/error }` and continue awaiting the next SSE event | 1 |
| Tool blocked/permission-denied handling: if the requested tool's source is classified `blocked` or `unknown`, POST `status: "permission_denied"` without executing | 0.5 |
| `AgentActivityPanel.tsx` — the step rows panel with all status states; driven entirely by backend `tool_call` / `step` events (no frontend-fabricated labels) | 2 |
| `AgentStepRow.tsx` — single row with inline action button (Grant Permission / Open Settings) | 1.5 |
| Native permission grant integration via existing `permissionService.ts` — granted / denied / blocked / settings deep-link | 1.5 |
| `DraftArea.tsx` — streaming draft display with blinking cursor and direct edit support | 2 |
| Two-button input row: input box + "Ask agent" + "Send to caller" + "Decline" | 1 |
| Conversational refinement loop: typing in input → "Ask agent" → open NEW `/agent-turn` SSE with `instruction` + `conversation_history` → same tool_call/token/done cycle → input clears | 2 |
| Mid-stream cancel logic (Take over button + tap-to-edit + type-in-input all cancel) — closes SSE, sends `DELETE /agent-turn`, ignores any pending tool_result | 1 |
| Frontend state machine wiring: classify → open agent-turn → handle tool_call dispatch → render tokens → handle refinement turns | 2.5 |
| Late permission grant handling: update `?` row to `✓`, enable the tool for the next turn (no automatic re-draft — owner asks the agent to re-try) | 0.75 |
| OTP-only special case rendering (bypass agent loop entirely, preserve existing flow) | 0.5 |
| All-unknown special case rendering | 0.5 |
| Unit tests: mock SSE emitter, tool_call dispatch, tool_result POST, cancel during pending tool_call, refinement, classification, step state transitions | 2.5 |
| **Subtotal** | **25.5** |

### 3.5a — Session-Scoped Conversation Memory (across-turn dialog)
*Dependencies: 3.2 (backend loop), 3.5 (agentic UI hook)*

The refinement loop from 3.5 passes `conversation_history` on every `/agent-turn` call so the agent sees the full dialog across turns within one screen session. 3.5a is the discipline of accumulating and threading that history correctly — not a separate capability layer, but the cross-turn memory wiring.

**Scope boundary (L5):** In-memory only. Conversation history lives in the `useConsentAgent` hook's `conversationHistoryRef` and is discarded on unmount. NOT persisted to Postgres — the consent component stays a thin pipe; chat artifacts don't belong in `consent_requests` rows.

**Experience the owner gets:**
- Tap "Ask agent" with "include my schedule" → agent calls `get_calendar_events` then drafts
- Tap "Ask agent" with "make it shorter" → agent re-drafts (no tool call), still remembers the schedule context from turn 1
- Tap "Ask agent" with "the name is Jon, check again" → agent re-calls `search_contacts(query="Jon", fuzzy=true)`, finds the contact, drafts correctly — because it saw the previous failed "I don't have that contact" turn in the history
- Close the screen → history is gone; next open starts fresh

**Backend changes:**

| Task | Hours |
|------|-------|
| Add `ConversationTurn` Pydantic model (`role: "user" \| "assistant"`, `content: str`) and optional `conversation_history: List[ConversationTurn]` field to the `/agent-turn` request body in `routes/consent_request.py` | 0.25 |
| In `ConsentAgentService`, map `conversation_history` into Pydantic AI's `message_history` list (`ModelRequest`/`ModelResponse` parts) and pass to `agent.iter(message_history=...)`. Pydantic AI handles rehydration — we just shape the list | 0.75 |
| Preserve initial-draft path (no history = fresh agent run) and OTP fast path (no agent at all) | 0.25 |
| Thread `conversation_history` through from route to service (serialize via `model_dump`) | 0.25 |
| Unit tests: initial draft (no history), refinement with 1 turn, refinement with multiple turns, refinement that triggers a tool re-query | 0.75 |
| **Subtotal** | **2.25** |

**Frontend changes:**

| Task | Hours |
|------|-------|
| Add `conversationHistoryRef` (`useRef<Array<{role, content}>>([])`) to `useConsentAgent`. Reset on unmount | 0.5 |
| After each completed stream (`done` event), push `{role: "assistant", content: final_text}` | 0.25 |
| In `refine()`, before opening the stream: push `{role: "user", content: instruction}`, then POST history alongside the instruction | 0.5 |
| Optional polish: render conversation turns as a collapsible history strip in the activity panel. Defer if time-boxed | 1 |
| Unit tests: verify history accumulates across refinements and resets between consent requests | 0.5 |
| **Subtotal** | **2.75** |

**Why no DB persistence:**
- The consent component is a trust boundary and a thin pipe (see `00-overview.md`). Adding chat history to `consent_requests` rows would balloon row size for a signal that has no downstream consumer.
- The owner's instructions are ephemeral UI interactions, not first-class business data.
- Future requirement for persisted chat history (e.g. learning owner preferences across sessions) is a Component 7 (User Memory) concern.

**What 3.5a does NOT touch:**
- `consent_requests` table schema (unchanged)
- DB migrations (none)
- `ConsentRequestProcessor` (unchanged)
- Permission orchestration from 3.5 (unchanged — runs pre-loop)

### 3.5a Total Effort

| Area | Hours |
|---|---|
| Backend | 2.25 |
| Frontend | 2.75 |
| **3.5a Total** | **5 hours (~0.6 day)** |

---

### 3.6 — Frontend: Integrate Agentic Screen Into Existing Surfaces (L8)
*Dependencies: 3.4, 3.5*

| Task | Hours |
|------|-------|
| Replace contents of `InformationConsentScreen.tsx` (modal during call) with the new agentic layout (header → activity panel → draft area → two-button input) wired to `useConsentAgent` | 2 |
| Replace contents of `ConsentRequestDetailScreen.tsx` (full screen from call logs) with the same layout | 2 |
| Update Redux `informationConsentSlice` (add agent activity state, classification state, conversation-history ref is hook-local not Redux) | 0.5 |
| Update FCM handler `informationConsentHandler.ts` (no major change since payload shape from Phase 1 stays the same) | 0.5 |
| Remove the old `useAgenticDraftStream` hook, old `lib/sseClient.ts`, and old `/draft-stream` client code — they are fully replaced by L1 + L7 | 0.5 |
| Unit tests | 1 |
| **Subtotal** | **6.5** |

### 3.7 — Phase 3 End-to-End Integration Testing
*After all of Phase 3 complete*

| Task | Hours |
|------|-------|
| Inbound call regression test (Phase 1 + 2 unhold paths still work) | 1 |
| Outbound call full flow with new agentic UI (intent → consent → agent loop → "Send to caller" → session injection → next turn) | 1 |
| WhatsApp full flow with new agentic UI (inbound → consent → agent loop → "Send to caller" → outbound message delivery) | 1 |
| **Tool-use happy path**: agent emits `search_contacts`, frontend executes, result feeds back, agent drafts using the contact data | 1 |
| **Tool-use re-query (the John/Jon case)**: initial draft says "I don't have that contact" for "John"; owner clarifies "the name is Jon"; agent re-calls `search_contacts(query="Jon", fuzzy=true)`, finds the contact, drafts correctly. Activity panel shows both queries | 1 |
| **Drafting-only scope**: owner types off-topic instruction ("what's the weather"); agent briefly acknowledges + redirects to drafting without leaving the loop | 0.5 |
| Conversational refinement: 2-3 "Ask agent" rounds, verify draft replaces correctly + activity log accumulates across turns + conversation_history threads correctly | 1 |
| Mid-stream cancel via Take over button + tap-into-draft + type-in-input; verify pending `tool_call` is abandoned cleanly (no orphan state in backend) | 1 |
| **Tool-result timeout**: force frontend to not POST `/tool-result` for 10s; verify backend treats as timeout and continues the loop without hanging | 1 |
| **MAX_ITERATIONS cap**: force the agent (via crafted prompt) to want more than 5 tool calls; verify it's forced to draft on the 5th | 0.5 |
| Missing device permission on Android: ? row → grant → fetch (on next agent turn) | 1 |
| Missing device permission on iOS (same) | 1 |
| Permanently denied permission (NEVER_ASK_AGAIN / BLOCKED): ? row shows "Open Settings", deep-link verified. Backend receives `status: "permission_denied"`, agent drafts around the missing data | 1 |
| Mixed available + denied tools: agent calls available ones, gets denied for blocked ones, drafts with partial data | 0.5 |
| All-unknown special case: blank draft, "Ask agent" disabled (agent loop never runs) | 0.5 |
| OTP-only special case (regression - preserve existing flow, no agent loop) | 0.5 |
| Android build + on-device test across all 3 channels + full agentic flow | 2 |
| iOS build + on-device test across all 3 channels + full agentic flow | 2 |
| Bug fixes from integration testing | 3 |
| **Subtotal** | **20** |

---

## Phase 3 Total Effort

| Sub-phase | Hours | Layers touched |
|---|---|---|
| 3.1 — Backend device-bridge spike (Pydantic AI round-trip) | 6 | L1, L2-minimal, L4-minimal |
| 3.2 — Backend Pydantic AI agent + tools + prompt + wiring | 13.5 | L2, L3, L5-backend, L6 |
| 3.2a — Backend analyzer awareness refinement | 2 | (pre-existing) |
| 3.3 — Frontend source classification helpers | 1.5 | (pre-loop L7) |
| 3.4 — Frontend header sub-components | 4 | L7 |
| 3.5 — Frontend agentic consent screen (hook + executors + UI) | 25.5 | L1, L4, L5-frontend, L7 |
| 3.5a — Session-scoped conversation memory | 5 | L5 |
| 3.6 — Frontend integrate agentic screen into surfaces | 6.5 | L8 |
| 3.7 — Phase 3 E2E integration testing | 20 | all |
| Buffer | 6 | — |
| **Phase 3 Total** | **90 hours (~11 days)** | |

Using Pydantic AI saves ~3.5h on the backend agent implementation (13.5h vs ~17h first-party) because the framework ships deferred-tool semantics, message-history rehydration, tool-arg validation, and the provider abstraction. Those savings are absorbed by the slightly larger spike (Pydantic AI is a new dep; need to verify the primitives on-device before committing). Total lands near the same envelope but with significantly less first-party orchestration code to maintain.

---

## Verification Criteria for Phase 3

Phase 3 is complete and shippable when ALL of the following are true:

1. ✅ Phase 1 + Phase 2 regression: all three channels still trigger consent and route the response correctly
2. ✅ Opening a consent screen (any channel) shows the agentic layout: header + activity panel + draft area + two-button input
3. ✅ The agent activity panel populates step rows in real time as the backend emits `tool_call` / `step` events — NOT from frontend-fabricated labels
4. ✅ Tokens stream into the draft area (visible character-by-character or chunk-by-chunk) via `react-native-sse` — the old homegrown XHR-based SSE client is removed
5. ✅ Tapping "Send to caller" sends the draft area text as the final response (not the input box)
6. ✅ Tapping "Ask agent" with an instruction triggers a new `/agent-turn` SSE that re-streams into the draft area
7. ✅ Refinement can be done multiple times in a row, each adding a row to the activity panel
7a. ✅ **Conversational memory**: three consecutive refinements ("include my schedule" → "make it shorter" → "add that I can reschedule") produce a final draft reflecting ALL three instructions, proving the agent remembers the dialog within the session
7b. ✅ **Session isolation**: closing and reopening the consent screen starts a fresh conversation (no history leaks across sessions, nothing in the DB)
7c. ✅ **Tool-use re-query on clarification (the John/Jon case)**: initial draft says "I don't have that contact" for "John"; owner says "the name is Jon"; agent re-calls `search_contacts` with the corrected name, finds the contact, drafts correctly. The activity panel shows both searches
7d. ✅ **Drafting-only scope**: owner types an off-topic instruction ("what's the weather today"); agent briefly acknowledges and redirects to drafting without leaving the loop or producing a general answer
8. ✅ Mid-stream cancel works via Take over button, tap-into-draft, and type-in-input — including when a `tool_call` is pending (no orphan state in backend)
9. ✅ Missing device permission shows a `?` row with [Grant Permission] button that triggers the native dialog
10. ✅ Permanently denied permission shows [Open Settings] and deep-links correctly
11. ✅ Late permission grant: row updates to `✓`, the next agent turn can use the newly-available tool
12. ✅ OTP-only special case still works (agent loop never runs, existing flow preserved)
13. ✅ All-unknown special case shows blank input with "Ask agent" disabled (agent loop never runs)
14. ✅ **Tool-result timeout**: backend handles a frontend that doesn't POST `/tool-result` within 10s; continues the loop with `{status: "timeout"}` without hanging
15. ✅ **MAX_ITERATIONS cap**: agent cannot exceed 5 tool calls per turn; forced to `draft_response` on the 5th
16. ✅ All three channels deliver the response correctly through Phase 2's handlers
17. ✅ Android device build + on-device test passes for all flows
18. ✅ iOS device build + on-device test passes for all flows

---

## Risk Notes

**Primary risk**:

**Device-bridge correlation protocol (L1 + L4)** — the SSE `tool_call` → frontend-executes → POST `/tool-result` → agent-resumes round-trip is not a standard pattern in most RN apps, even though Pydantic AI's `DeferredToolRequests` / `DeferredToolResults` makes the backend side clean. Race conditions to watch for:
- Owner taps "Take over" while a `tool_call` is in-flight — frontend must cancel SSE AND not POST the stale result
- Frontend dies / network blips mid-tool — backend's `TOOL_RESULT_TIMEOUT=10s` must trigger cleanly and release the Event
- Multiple concurrent consent screens (shouldn't happen but test it) — `tool_call_id` UUIDs keep them isolated
- The 3.1 spike exists specifically to de-risk this before the full 3.2 build

**Secondary risks**:

- **Pydantic AI version pinning** — pin to a specific `pydantic-ai` version (e.g., `^1.82`) to avoid upstream API shifts mid-project. Budget 0.5h per minor version upgrade during maintenance.
- **P3.5 is the dominant frontend risk** at 25.5 hours — where most of the new agentic UX + device tool executors live. If the team is unfamiliar with RN native-module permission flows or the `react-native-sse` API, add 3-5h buffer.
- **Device builds**: Android and iOS builds are split across two days to avoid back-to-back wall-clock time.
- **Backend cancel handling**: cancelling mid-tool needs to release the pending asyncio.Event, halt the agent iteration, and drop per-request state cleanly. Test the cancel-then-retry path explicitly.
- **Drafting-only scope enforcement** depends on a strong system prompt (L6). If testing shows the agent wanders off-topic, the fix is prompt iteration, not code changes. Budget 1-2h of prompt tuning during 3.7.
- **ngrok buffering during dev**: if developing over ngrok, SSE streams may buffer. Test against a LAN IP before assuming production streaming is broken.

---

## What This Phase Does NOT Touch in Earlier Phases

**Phase 1 code stays intact:**
- DB schema (Phase 1 added the `request_payload` JSONB; Phase 3 uses it as-is)
- `ConsentRequestProcessor` (Phase 1's channel-agnostic API is unchanged)
- `consent_action_service.handle_consent_action` channel dispatch — stays as-is

**Phase 2 code stays intact:**
- `_handle_outbound_call_consent`, `_handle_whatsapp_consent` — these only run on the *final* "Send to caller" / "Decline" action, not during the agentic drafting flow. Phase 3's drafting is purely a UI + streaming concern; the channel-specific delivery happens after.
- Channel detection in `voice_service.process_outbound_speech_with_disconnect` and `whatsapp_service._maybe_auto_reply` — unchanged

---

## 3.8 — Agent Response Behavior Specification

### Response Behavior Rules

The consent agent's response behavior follows these principles:

**1. The agent executes requests through a series of steps — drafting is just one of them.**
Every request is fulfilled through multiple steps: checking time, reading calendar, searching contacts, answering the owner, drafting a response, etc. Drafting the response is one step among many, not the primary focus. Each step executes in sequence, shows its label in real time, and streams its output as it completes — the owner sees the agent's work as it happens.

**2. Mode decision is LLM-based, never keyword-based.**
The agent determines the appropriate steps (including whether to draft, answer, or both) by understanding the owner's intent from conversational context. No signal word lists, no pattern matching — the LLM reasons about what the owner wants and plans the steps accordingly.

**3. Every draft addresses the original request first.**
The draft's primary purpose is always to address what the requester asked. Owner refinements are additive — they layer onto the original answer, never replace it. A complete draft = original request answer + all prior additions + the latest refinement.

**4. Answering the owner is a separate step from drafting.**
When the owner wants to learn something for themselves, the agent executes an "Answering your question" step and responds directly. This does not affect the existing draft — the draft card stays intact and approvable.

**5. Subsequent drafts are redrafts, never drafts from scratch.**
After the initial draft exists, any further drafting is a **redraft** — the agent refines, adds to, or adjusts the existing draft. It never starts from scratch. The agent determines from conversational context whether the owner's instruction implies a redraft. Not every owner message triggers one — only when the owner's intent signals a change, addition, or revision to outgoing content. When the agent identifies redraft intent, it adds a "Redrafting response" step to the execution.

**6. The agent sees the full drafting history, not just the latest draft.**
All previous drafted responses are available to the agent — not only the most recent one. This allows the agent to understand how the draft evolved, what was added or changed across turns, and make informed decisions about what to carry forward vs. what the owner intentionally dropped. The agent uses this history to produce a draft that reflects the owner's cumulative intent, not just the last snapshot.

**7. Conversation history carries mode context.**
Each prior turn is annotated with its step type (DRAFT sent to requester / ANSWER told to owner) so the agent can reason about the conversation flow and what content belongs in the draft.

**8. The agent requests missing permissions inline during the conversation.**
When the agent needs a data source (calendar, contacts, etc.) and the OS permission has not been granted, the agent surfaces a permission request to the owner within the real-time chat flow. The owner can grant the permission directly from the conversation — the agent does not silently skip the data source or fail. Once granted, the agent proceeds to fetch the data and incorporate it into the response.

**9. Every step streams its output in real time.**
Each step the agent executes shows its label and streams its result as it completes. The owner sees tool results (found contacts, calendar events), answers, and draft text appearing in real time beneath each step row. Nothing is hidden or batched.

**10. The draft card is an approval control, separate from the drafting step.**
The drafting step ("Drafting response") streams its output like any other step — text appears beneath the step row in real time. The draft card with [Approve & Send] is a separate UI element that presents the final output for the owner to review, edit, and send. The step shows the agent's work; the card gives the owner control over the final action.

**11. When intent is unclear, the agent asks the owner.**
Rather than guessing, the agent can ask the owner to clarify what they want. This is preferable to silently making the wrong choice — whether that's altering outgoing content the owner didn't intend to change, or answering when the owner wanted a redraft. If asking isn't practical, default to ANSWER — safer to inform than to change what gets sent.

**12. The first-turn draft is mandatory when data is sufficient.**
On the first turn, the agent MUST produce a draft response if it has enough data from the available sources to do so. It does not pause to ask the owner for direction first, does not surface a "what do you want to say?" prompt — it just drafts. Drafting is the default first-turn outcome whenever the data supports it. (If data is insufficient, see Rule 11 — the agent asks the owner.)

**13. Conversation history is append-only — the agent never rewrites its own past messages.**
When the owner corrects something the agent said earlier, the contradicted prior agent message stays in the in-memory conversation history as-is. The history is not edited, struck through, or mutated. The LLM has full visibility into the entire conversation including its own past mistakes, and weights newer corrections over older statements when reasoning. Transparency wins over a "clean" rewritten history.

**14. Owner correction is authoritative; on ambiguity between sources and owner, the agent asks.**
When the owner provides input that contradicts what data sources returned (or what the agent previously said), the owner's correction wins by default — the agent re-queries with the corrected context and re-drafts. When the conflict is genuinely ambiguous (it's not clear whether the owner is correcting the data, or asking about a different scenario, or providing supplementary info), the agent does not silently pick a side — it surfaces a clarifying question to the owner via Mode B (ANSWER) and waits for the next turn. Interactive resolution is preferred over guessing.

**15. The drafting step always emits on the first turn — placeholder when data is insufficient.**
The "drafting response" step is mandatory on the first response routing, regardless of whether the agent has enough data. If the agent has enough data, it streams the actual draft tokens into the draft bubble (existing behavior). If the agent does NOT have enough data to draft, the draft bubble still renders on the first turn but shows a professional placeholder along the lines of *"I don't have enough information to draft a response — please reply with your own message below."* The placeholder informs the owner that manual input is needed; the owner then writes their response through the manual-response escape hatch (Rule 17). The draft bubble is never silently skipped on the first turn. (Exception: when the agent is paused waiting on a permission grant — see Rule 16 — the drafting step is delayed until the permission situation is resolved.)

**16. Permission denial pauses drafting; the grant flow runs inline in the chat thread, with a Resume affordance.**
When a tool returns `permission_denied`, the agent does NOT draft around the missing data on that turn. Instead:
- The step row in the chat thread shows the failure label and a source-specific button — e.g., *[Grant Permission — Calendar]*.
- The agent's drafting step is paused; the agent waits for the permission situation to resolve before continuing.
- Tapping the button triggers the per-source / per-OS grant flow. The actual mechanics vary: on iOS, where a previously-denied OS permission cannot be re-prompted natively, an app-level popup explains that the owner needs to go to the device Settings and enable the permission manually. On Android, the standard OS dialog or settings deep-link applies. Future sources (e.g., Composio-backed third-parties) bring their own grant flows.
- While the owner is away handling the grant, the button transforms to *[Resume]*.
- On *[Resume]* tap, the consent screen re-checks the permission. If now granted, the failed tool is re-run and the agent resumes its flow (continues toward drafting). If still not granted, the button reverts to *[Grant Permission — <source>]* and the owner can try again or take the manual-response path (Rule 17).
- The agent never silently drafts around a permission denial — the owner is always given the explicit chance to grant and get a better answer.

**17. Manual response is always an available escape hatch.**
The owner can elect to write the response themselves — for any reason, at any point in the consent flow (no data, refused permission, simply prefer to write it). A *[Respond manually]* affordance is available in the agentic chat thread. Tapping it opens an editor popup; the owner types or edits their response and confirms (e.g., a Done action). The submitted text becomes a draft bubble in the chat thread — treated visually and behaviorally as a "drafted response" step, even though the agent didn't produce it. The standard *[Approve & Send]* action sits on this manual draft bubble, so the owner approves it the same way they would approve an agent draft. Manual response is the universal fallback: it is what makes Rule 15's insufficient-data placeholder actionable, and what makes Rule 16's "owner refuses to grant" path resolvable.


### Change 1: Pass current draft text to the backend on refinement

The frontend already holds the current draft in the `draftText` state variable. Pass it to the backend so the agent can build on it instead of starting from scratch.

| File | What changes |
|------|-------------|
| `app/routes/consent_request.py` | Add optional `current_draft_text: str` field to `AgentTurnRequest` |
| `app/services/consent_agent_service.py` | Accept `current_draft_text` param in `run_agent_turn()` and `_run_loop()`. When present, inject into user prompt as `CURRENT DRAFT:` block and tell the LLM to build on it |
| `trybot-ui/hooks/useConsentAgent.ts` | Add `currentDraftText` param to `refine()` and `startAgentTurn()`, include in request body |
| `trybot-ui/components/consent/AgenticConsentLayout.tsx` | Pass `draftText` to `refine()` in `handleSendInstruction` |

### Change 2: Annotate conversation history with mode labels

So the agent knows which prior turns were drafts vs answers.

| File | What changes |
|------|-------------|
| `trybot-ui/hooks/useConsentAgent.ts` | Add `mode?: 'draft' \| 'answer'` to `ConversationTurn` type. Set `mode: 'answer'` on `agent_message_done`, `mode: 'draft'` on `done` |
| `app/routes/consent_request.py` | Add optional `mode` field to `ConversationTurn` model |
| `app/services/consent_agent_service.py` | Format history with mode labels: `Agent [DRAFT sent to requester]: ...` vs `Agent [ANSWER to owner]: ...` |

### Change 3: Strengthen system prompt (principle-based, no examples)

Replace the mode decision section with ordered tests. Add a draft continuity rule.

**File:** `app/services/consent_agent_prompt.py`

Mode decision — ordered tests:
1. Is the owner directing you to produce/modify/refine content to send to the requester? → DRAFT
2. Is the owner seeking information for their own understanding? → ANSWER
3. No owner instruction yet (initial turn)? → DRAFT
4. Still ambiguous? → ANSWER (safer)

New rule: When prior history contains DRAFT entries, any new draft must incorporate all content from the most recent draft plus the requested changes. Never produce a draft that covers less ground than the previous one.

### Change 4: Split refinement prompt into draft/no-draft paths

**File:** `app/services/consent_agent_service.py` (user prompt construction)

When `current_draft_text` is present:
- Include it as `CURRENT DRAFT:` block
- Explicitly tell the LLM to produce an updated DRAFT that preserves all existing content and incorporates the owner's instruction
- Use `DRAFT:` prefix directive (no ambiguity)

When `current_draft_text` is absent (owner is on an ANSWER turn or no draft exists yet):
- Keep current prompt structure
- Let the LLM decide mode autonomously

### Implementation order

1. Backend models (`consent_request.py`) — add optional fields, backward-compatible
2. System prompt (`consent_agent_prompt.py`) — principle-based rewrite
3. Service layer (`consent_agent_service.py`) — accept new params, annotate history, split prompt paths
4. Frontend hook (`useConsentAgent.ts`) — add mode annotations, pass draft text
5. Frontend UI (`AgenticConsentLayout.tsx`) — pass `draftText` to `refine()`

All new fields are optional with defaults → backend and frontend can be deployed independently without breaking.

### Verification

1. **Initial turn**: Agent reads calendar, produces DRAFT addressing original request
2. **Mode B question**: Owner asks a question for their own knowledge → agent shows "Answering your question" step, responds with ANSWER (not "Drafting response")
3. **Refinement after ANSWER**: Owner says "also inform X" → agent produces DRAFT that includes BOTH the original request answer AND the new addition
4. **Multiple refinements**: Each refinement builds on the previous draft, never drops earlier content
5. **Pure ANSWER stays ANSWER**: Owner asks a question purely for their own knowledge → no draft change, existing draft card stays intact

### Effort

| Task | Hours |
|------|-------|
| Backend models + prompt + service changes | 3 |
| Frontend hook + UI changes | 2 |
| Testing + prompt tuning | 2 |
| **Total** | **7** |
