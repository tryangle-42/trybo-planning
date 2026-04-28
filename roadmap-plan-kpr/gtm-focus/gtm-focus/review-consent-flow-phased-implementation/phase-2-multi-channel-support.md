# Phase 2 — Multi-Channel Support

## Goal

Make outbound calls and WhatsApp eligible to trigger the consent request component. The reusable `ConsentRequestProcessor` from Phase 1 is in place; this phase wires the new channel agents into it and adds the channel-specific response delivery handlers.

The existing `InformationConsentScreen` and `ConsentRequestDetailScreen` UIs are still used. They get small updates to display the channel correctly (e.g., "Outgoing Call" vs "Incoming Call" badge), but no new UI components are introduced.

## What Ships at the End of Phase 2

- All three channels can trigger consent requests:
  - **Inbound calls** (Phase 1, unchanged)
  - **Outbound calls** (Twilio / Knowlarity) — new
  - **WhatsApp tasks** — new
- `consent_action_service.handle_consent_action` has all three channel branches implemented
- The owner's response is delivered correctly for each channel:
  - Inbound call → unhold + play/stream response (existing behavior, Phase 1)
  - Outbound call → session injection (`pending_consent_response` flag picked up on next agent turn)
  - WhatsApp → `send_outbound_text` to the counterpart
- Existing screens display the correct channel context (badge + label)
- All three channels regression-tested end-to-end

## What Is NOT in Scope (Deferred to Phase 3)

- **Streaming SSE endpoint** → Phase 3
- **Agentic consent screen UI** (activity panel, step rows, two-button input, conversational refinement) → Phase 3
- **Frontend source classification helpers** → Phase 3
- **`AgentActivityPanel`, `DraftArea`, refinement loop** → Phase 3
- **Token-by-token streaming** → Phase 3

The screens stay as they are today (one-shot draft, basic edit-and-send) — they just now receive consent requests from outbound calls and WhatsApp too.

---

## Implementation Tasks

### 2.1 — Backend: Outbound Call Integration
*Dependencies: Phase 1 complete*

**Files**:
- `trybot-api/app/services/voice_service.py` — extend `process_outbound_speech_with_disconnect`
- `trybot-api/app/services/conversation_service.py` — reuse existing `detect_owner_info_intent`

| Task | Hours |
|------|-------|
| Add intent detection to `process_outbound_speech_with_disconnect` (after the AI response generation, run `detect_owner_info_intent` on the user speech) | 1 |
| New helper `_handle_outbound_info_request_intent`: build `request_payload` from session history snapshot, call `ConsentRequestProcessor.process_consent_request(channel="outbound_call", call_id=..., request_payload=...)`, return a stalling message ("Let me check on that for you, one moment") | 1.5 |
| Session pickup: at the start of `process_outbound_speech_with_disconnect`, check `session.get("pending_consent_response")`. If present, inject it into the conversation history so the AI's next response naturally weaves it in, and clear the flag | 0.5 |
| Build `conversation_context` JSONB from outbound session history (turns + data_sources + missing_data_sources from analyzer) | 0.5 |
| Integration test: mock outbound call with consent trigger, verify FCM sent with `channel="outbound_call"` and call_id linked | 0.5 |
| **Subtotal** | **4** |

### 2.2 — Backend: WhatsApp Integration
*Dependencies: Phase 1 complete*

**File**: `trybot-api/app/services/whatsapp_service.py` — extend `_maybe_auto_reply`

| Task | Hours |
|------|-------|
| Add intent detection to `_maybe_auto_reply` after `_fetch_thread_messages` and before generating the auto-reply (run `detect_owner_info_intent` on the inbound message) | 1 |
| If intent detected: call `InformationRetrievalAnalyzer.analyze_information_request` with the WhatsApp thread as conversation history; if `should_initiate_consent_request` is true, build `request_payload` and call `ConsentRequestProcessor.process_consent_request(channel="whatsapp", task_id=..., whatsapp_conversation_id=..., request_payload=...)` | 1 |
| Send a holding message via `send_outbound_text`: "Let me check with [owner] and get back to you" — then return early so the normal auto-reply doesn't fire | 0.5 |
| Integration test: mock WhatsApp inbound with consent trigger, verify FCM sent + holding message delivered | 1 |
| **Subtotal** | **3.5** |

### 2.3 — Backend: Channel-Aware Response Routing (Fill the Phase 1 Stubs)
*Dependencies: 2.1, 2.2*

**File**: `trybot-api/app/services/consent_action_service.py`

Phase 1 left `_handle_outbound_call_consent` and `_handle_whatsapp_consent` as `NotImplementedError` stubs. Phase 2 fills them in.

| Task | Hours |
|------|-------|
| `_handle_outbound_call_consent`: look up call_sid from call_id, check session existence. If live: `await sessions.set_value(call_sid, "pending_consent_response", response)` so the next outbound speech turn picks it up. If not live: mark expired | 1.5 |
| `_handle_whatsapp_consent`: look up `whatsapp_conversation_id`, on approve call `send_outbound_text(...)` to deliver the response to the WhatsApp counterpart, on decline send a polite decline message (or no message). No liveness check (WhatsApp is async) | 1.5 |
| Unit tests for both new handlers | 1 |
| **Subtotal** | **4** |

### 2.4 — Frontend: Display Channel Context
*Dependencies: 2.1, 2.2, 2.3*

The existing screens get a small update so they show the correct channel badge/label. No new components.

| Task | Hours |
|------|-------|
| Add `formatChannelLabel(channel)` and `getChannelIcon(channel)` helpers in `utils/consentUtils.ts` (e.g., "Incoming Call" / "Outgoing Call" / "WhatsApp Message" + Ionicons names) | 0.5 |
| Update `InformationConsentScreen.tsx` and `ConsentRequestDetailScreen.tsx` headers to show the channel badge — read `channel` from `request_payload` parent context | 1 |
| Update `informationConsentHandler.ts` to pass `channel` through to the screen | 0.25 |
| **Subtotal** | **1.75** |

### 2.5 — Phase 2 Verification
*Dependencies: all of Phase 2*

| Task | Hours |
|------|-------|
| **Inbound call regression** — verify Phase 1 inbound flow still works (no break from new dispatch handlers) | 0.5 |
| **Outbound call E2E** — initiate an outbound call, have the recipient ask for the owner's schedule, verify FCM lands with `channel="outbound_call"`, verify owner sees the screen, verify approval injects the response into the next agent turn naturally | 1.5 |
| **WhatsApp E2E** — start a WhatsApp conversation with an info request, verify FCM lands with `channel="whatsapp"`, verify owner sees the screen, verify approval delivers the response as a WhatsApp message to the counterpart | 1.5 |
| **Decline scenarios** — verify decline behavior for all three channels | 1 |
| **Channel badge display** — confirm the new badges render correctly for each channel | 0.5 |
| Bug fixes from regression | 1.5 |
| **Subtotal** | **6.5** |

---

## Phase 2 Total Effort

| Sub-phase | Hours |
|---|---|
| 2.1 — Backend outbound call integration | 4 |
| 2.2 — Backend WhatsApp integration | 3.5 |
| 2.3 — Backend channel-aware response routing | 4 |
| 2.4 — Frontend channel display | 1.75 |
| 2.5 — Phase 2 verification | 6.5 |
| Buffer | 0.25 |
| **Phase 2 Total** | **20 hours (~2.5 days)** |

---

## Verification Criteria for Phase 2

Phase 2 is complete and shippable when ALL of the following are true:

1. ✅ Phase 1 inbound regression still passes (no break from new code paths)
2. ✅ An outbound call where the recipient asks for the owner's information triggers a consent request with `channel='outbound_call'`
3. ✅ The owner's approval on an outbound consent injects the response into the agent's next turn (the agent naturally says "I have that information for you...")
4. ✅ A WhatsApp message asking for owner info triggers a consent request with `channel='whatsapp'`
5. ✅ The agent sends a holding message ("Let me check with [owner]...") in response
6. ✅ The owner's approval on a WhatsApp consent delivers the response as a WhatsApp message to the counterpart
7. ✅ Declining works correctly for all three channels
8. ✅ The existing screens display the correct channel badge for each channel
9. ✅ Multiple consent requests in the same session (e.g., two info requests during one call) work independently

---

## Risk Notes

- **Outbound session injection timing**: The session flag pickup at the start of `process_outbound_speech_with_disconnect` requires careful state management. If the next speech turn happens before the owner responds, the agent should not stall — it should continue normally and the response will surface on a later turn. The stalling message handles the immediate moment.
- **WhatsApp delivery delays**: WhatsApp messages can be delivered out-of-order or after long delays. The "holding message" should make this acceptable to the counterpart, but watch for double-delivery (e.g., owner approves twice, or the holding message + the actual response arrive far apart).
- **Inbound regression risk**: Adding the channel dispatch to `consent_action_service` is mechanical, but the existing TwiML / duplex / LiveKit unhold logic is intricate. The regression test in 2.5 must cover all three transport types.

---

## What Phase 3 Will Change in Phase 2's Code

**Phase 3 will modify:**
- The screens get replaced by the agentic UI. The channel badge from 2.4 carries over, but the rest of the screen layout changes significantly.
- The frontend handler / Redux slice may receive new fields (e.g., agent activity state) but the channel-display logic from 2.4 stays intact.
- Backend `consent_response_service.py` becomes a streaming SSE endpoint. The `_handle_outbound_call_consent` / `_handle_whatsapp_consent` handlers from 2.3 are unchanged — they only run on the *final* approve/decline action, not during the agentic drafting flow.
