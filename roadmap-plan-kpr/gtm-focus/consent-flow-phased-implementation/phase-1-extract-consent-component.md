# Phase 1 — Extract a Reusable Consent Request Component

## Goal

Take the existing inbound-call consent functionality and refactor it into a clean, channel-agnostic component. **No user-visible behavior change.** Existing inbound-call consent continues to work end-to-end exactly as it does today.

This phase is purely a refactor: separating the consent request logic from the inbound-call specifics so it can be reused by other channels in Phase 2.

## What Ships at the End of Phase 1

- Existing inbound-call consent flow works end-to-end (regression-tested)
- Backend has a clean `ConsentRequestProcessor` with a channel-agnostic API (`process_consent_request(channel, ...)`)
- DB schema has the new shape (`channel`, `request_payload` JSONB, `whatsapp_conversation_id` columns)
- `consent_action_service.handle_consent_action` has channel-dispatch scaffolding (only the inbound-call branch is implemented; other branches throw `NotImplementedError`)
- Frontend FCM handler, Redux slice, and consent screens read from the new payload shape
- The existing `InformationConsentScreen` and `ConsentRequestDetailScreen` UIs are visually identical to before
- All inbound-call regression tests pass

## What Is NOT in Scope (Deferred to Later Phases)

- **Outbound call integration** → Phase 2
- **WhatsApp integration** → Phase 2
- **`_handle_outbound_call_consent` and `_handle_whatsapp_consent`** dispatch handlers → Phase 2
- **Streaming SSE endpoint** → Phase 3
- **Agentic consent screen UI** (activity panel, step rows, two-button input, conversational refinement) → Phase 3
- **Frontend source classification helpers** → Phase 3
- **Agent activity panel + draft area + refinement loop** → Phase 3

---

## Implementation Tasks

### 1.1 — DB Schema Migration
*Dependencies: None*

Add the new columns and migrate existing data into the JSONB shape.

**File**: `trybot-ui/supabase/migrations/20260408000000_consent_requests_request_payload.sql`

| Task | Hours |
|------|-------|
| Write migration: add `channel`, `request_payload` JSONB, `whatsapp_conversation_id` columns. Backfill existing rows: `channel='inbound_call'`, `request_payload` built from old `information_prompt` + `suggested_data_sources` (with `conversation_context.transcript_id` left null for backfilled rows — they didn't store transcript references). Drop the old columns. | 1.5 |
| Indexes: `idx_consent_requests_channel`, `idx_consent_requests_wa_conversation` | 0.5 |
| RLS policy review | 0.5 |
| Local supabase verification with existing rows | 0.5 |
| **Subtotal** | **3** |

### 1.2 — Backend: Channel-Agnostic Consent Request Component
*Dependencies: 1.1*

**File**: `trybot-api/app/services/information_request_processing_service.py` → renamed to `consent_request_processor.py` (or keep filename, rename class)

| Task | Hours |
|------|-------|
| Refactor `InformationRequestProcessor` → `ConsentRequestProcessor` with new method `process_consent_request(channel, user_id, call_id?, task_id?, whatsapp_conversation_id?, request_payload, db)` | 2 |
| Update `_create_consent_request` to write `channel` + `request_payload` JSONB instead of the old columns | 1 |
| Existing `_handle_owner_info_request_intent` in `voice_service.py` builds the `request_payload` dict (summary, conversation_context with `transcript_id` + `transcript_snapshot_seq` + data_sources + missing_data_sources — no full transcript copy, just a reference) and calls `process_consent_request(channel="inbound_call", ...)` | 1 |
| Update `ConsentRequestStore` (`consent_request_store.py`) to read/write `request_payload` JSONB | 1 |
| Update FCM payload builder (`firebase_service.send_information_request_notification`): extract summary/channel/requester from `request_payload`, replace hardcoded `"channel": "call"` with the dynamic value | 1 |
| Unit tests | 1 |
| **Subtotal** | **7** |

### 1.3 — Backend: Channel-Dispatch Scaffolding (Inbound Only)
*Dependencies: 1.2*

**File**: `trybot-api/app/services/consent_action_service.py`

Refactor `handle_consent_action` to dispatch by channel, with only the inbound branch implemented. Other branches throw `NotImplementedError("Channel <X> handler arrives in Phase 2")`.

| Task | Hours |
|------|-------|
| Refactor `handle_consent_action` with channel dispatch | 0.5 |
| Extract existing inbound logic into `_handle_inbound_call_consent` (no behavior change) — TwiML / duplex / LiveKit unhold paths preserved verbatim | 1 |
| Add `_handle_outbound_call_consent` and `_handle_whatsapp_consent` stubs that raise `NotImplementedError` | 0.25 |
| Add `GET /consent-request/{id}/details` endpoint that returns the full row including `request_payload` JSONB (the UI will need this in Phase 3, but adding it now lets the existing screens fetch the new shape consistently) | 0.5 |
| Unit tests for inbound dispatch | 0.75 |
| **Subtotal** | **3** |

### 1.4 — Frontend: Read from New Payload Shape
*Dependencies: 1.1, 1.2*

The existing UI is unchanged, but the data plumbing reads from the new structure.

| Task | Hours |
|------|-------|
| Update `lib/informationConsentHandler.ts` to parse the new FCM payload shape (channel field, payload extraction) — backward compatible with the old shape during the migration window | 0.5 |
| Update `services/consentRequestService.ts` `ConsentRequest` interface: add `channel`, `request_payload` JSONB; remove old `information_prompt` and `suggested_data_sources` direct fields (those now live inside `request_payload`) | 0.5 |
| Update `store/informationConsentSlice.ts`: pull `informationPrompt` and `dataSources` out of `request_payload` when populating Redux state | 0.5 |
| Update `components/InformationConsentScreen.tsx` and `components/ConsentRequestDetailScreen.tsx` to receive the new shape — minimal changes since the props they consume can be derived from `request_payload` upstream | 1 |
| Add a fetch on screen open: call `GET /consent-request/{id}/details` to load the full record (so the screens always have the freshest data, which becomes important in Phase 3) | 0.5 |
| **Subtotal** | **3** |

### 1.5 — Phase 1 Verification
*Dependencies: all of Phase 1*

| Task | Hours |
|------|-------|
| Inbound call regression test: trigger consent request from a real inbound call, verify FCM lands, verify screen renders with all the same fields, verify approve/decline still unholds the call correctly through TwiML / duplex / LiveKit paths | 1 |
| Verify existing pre-migration consent_request rows still display correctly (backfill check) | 0.5 |
| Bug fixes from regression | 0.5 |
| **Subtotal** | **2** |

---

## Phase 1 Total Effort

| Sub-phase | Hours |
|---|---|
| 1.1 — DB schema migration | 3 |
| 1.2 — Backend channel-agnostic component | 7 |
| 1.3 — Backend channel-dispatch scaffolding (inbound only) | 3 |
| 1.4 — Frontend read from new payload | 3 |
| 1.5 — Phase 1 verification | 2 |
| **Phase 1 Total** | **18 hours (~2.5 days)** |

---

## Verification Criteria for Phase 1

Phase 1 is complete and shippable when ALL of the following are true:

1. ✅ The DB migration runs cleanly on a copy of production with no data loss; existing consent_requests rows are correctly backfilled
2. ✅ An inbound call that triggers `_handle_owner_info_request_intent` produces a new consent_requests row with `channel='inbound_call'` and a populated `request_payload` JSONB
3. ✅ The FCM notification lands on the owner's device with the new payload shape
4. ✅ Opening the existing `InformationConsentScreen` (modal during call) shows the same UI as before, with prompt + data sources + draft response
5. ✅ Opening the existing `ConsentRequestDetailScreen` (from call logs) shows the same UI as before
6. ✅ Approving the consent unholds the call correctly (TwiML / duplex / LiveKit — all three paths verified)
7. ✅ Declining the consent unholds with the decline message correctly
8. ✅ Existing call-ended-before-response flow still marks the request as expired
9. ✅ No regressions in the OTP-only special case
10. ✅ `_handle_outbound_call_consent` and `_handle_whatsapp_consent` exist as stubs that raise `NotImplementedError` (Phase 2 will implement them)

---

## What Phase 2 / Phase 3 Will Change in Phase 1's Code

It's expected that future phases will modify code introduced in Phase 1. This is intentional — we don't try to over-engineer Phase 1 to anticipate later needs. Anticipated changes:

**Phase 2 will modify:**
- `consent_action_service.py` — fill in `_handle_outbound_call_consent` and `_handle_whatsapp_consent` stubs
- `voice_service.py` — outbound call paths will start calling `process_consent_request(channel="outbound_call", ...)`
- `whatsapp_service.py` — `_maybe_auto_reply` will start calling `process_consent_request(channel="whatsapp", ...)`
- Frontend Redux + handler may update slightly to differentiate channel display (e.g., showing "WhatsApp Message" vs "Incoming Call")

**Phase 3 will modify:**
- Frontend consent screens (`InformationConsentScreen.tsx`, `ConsentRequestDetailScreen.tsx`) — replaced with the agentic UI components. The data plumbing from Phase 1 remains intact; only the rendering changes.
- `consent_response_service.py` — converted to a streaming SSE endpoint
- `useConsentScreenLoading.ts` hook — superseded by `useAgenticDraftStream`
- A new `process_consent_request` argument may be added for the agentic context, but the existing channel-agnostic API stays intact
