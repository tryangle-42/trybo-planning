# Phase 3 — Agentic UI with Token Streaming

## Goal

Replace the existing consent screens with a true **agentic experience**. The owner sees an agent working on their behalf in real time — fetching data, drafting a response token-by-token, and accepting conversational refinements.

After Phase 1 (extract reusable component) and Phase 2 (multi-channel support), the consent functionality is fully working across all three channels with the existing one-shot draft UI. Phase 3 upgrades that UI without changing the underlying consent component or channel dispatch — both stay intact.

## What Ships at the End of Phase 3

- The consent screen behaves like a conversational agent the owner is collaborating with
- An **Agent Activity Panel** shows step rows for everything the agent does (fetching data, asking for permissions, drafting, refining)
- **Token-by-token streaming** of the draft response into a draft area via SSE
- A **two-button input model**: one input box, "Ask agent" (refine the draft), "Send to caller" (send the draft as final response)
- **Conversational refinement loop**: the owner can type instructions like "make it shorter" and the agent re-streams the draft
- **Inline permission grant flow**: missing device permissions appear as `?` step rows with `[Grant Permission]` buttons inside the activity panel
- **Mid-stream cancel**: typing in the input or tapping the draft area cancels the active stream
- All three channels (inbound, outbound, WhatsApp) use the new agentic screen
- Special cases handled: OTP-only bypasses the agentic flow; all-unknown sources show a blank input

## What Is NOT in Scope

- **Third-party data sources** (Gmail, Outlook, Slack via Composio) — handled by the future Third-Party Data Sources component
- **Backend orchestration of the agent loop** — the orchestration lives entirely in the frontend; the backend just streams LLM tokens

---

## The Agentic Consent Screen

### Visual Layout

```
┌──────────────────────────────────────────────────────────┐
│  ← Information Consent                                  │
│  [Channel badge] John Smith • +1234567890                │
│  John is asking for your schedule this Thursday          │
│  ▸ View conversation                                     │
├──────────────────────────────────────────────────────────┤
│  AGENT ACTIVITY                                          │
│  ✓ Reading your contacts                                 │
│  ✓ Granted calendar access                               │
│  ✓ Reading your device calendar                          │
│  ⟳ Drafting response on your behalf                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Hi John, I have a meeting at 3pm Thursday and I'm      │
│  free after 5. Want to grab coffee then?▊               │
│                                                          │
│  [tokens streaming live with blinking cursor]            │
│                                                          │
├──────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────┐               │
│  │ Type to refine, or edit the draft... │               │
│  └──────────────────────────────────────┘               │
│  [Ask agent]              [Send to caller] [Decline]    │
└──────────────────────────────────────────────────────────┘
```

**Five zones, top to bottom:**
1. **Header** — channel badge + requester identity + the original information request summary + collapsible transcript
2. **Agent Activity panel** — step rows showing what the agent is doing
3. **Draft area** — the streaming draft response (becomes editable when streaming finishes)
4. **Input box** — for typing instructions to the agent
5. **Action buttons** — "Ask agent" / "Send to caller" / "Decline"

### Step Rows in the Agent Activity Panel

| Icon | Meaning | Example labels |
|---|---|---|
| `⟳` | In progress | "Reading your device calendar..." |
| `✓` | Completed | "Read your contacts" |
| `⚠` | Failed (non-blocking) | "Couldn't read your contacts" |
| `⊘` | Cancelled / skipped | "Drafting cancelled by you" |
| `?` | Needs the owner's action | "Need calendar access [Grant Permission]" |

A `?` row has an inline action button right in the row. Tapping it triggers `permissionService.requestPermission(source)`. On success, the row updates to `✓ Granted [source] access` and the agent continues — adds new step rows for fetching, then drafting.

If the OS reports permanent denial (`NEVER_ASK_AGAIN` / `BLOCKED`), the row updates to: `? [Source] denied. Open Settings to enable. [Open Settings]` — tapping deep-links into app settings.

If the owner ignores the `?` row, the agent waits a few seconds, then proceeds without that source: `⊘ Skipped [source] (no permission)` and continues drafting with whatever data IS available. The owner is never blocked.

### The Three Phases of an Agentic Drafting Run

**Phase A — Discovery and access requests**

When the screen opens, the agent classifies each source from `request_payload.conversation_context.data_sources`:
- **Available** (device permission granted) → emit `⟳ Reading your [source]`, fetch, update to `✓`
- **Missing — connectable** (denied but can re-request) → emit `? Need [source] permission [Grant Permission]`
- **Missing — blocked** (permanently denied) → emit `? [Source] denied. Open Settings to enable. [Open Settings]`
- **Missing — unknown** (not in `{device_calendar, device_contact, device_otp}`) → silent skip

**Phase B — Drafting (token streaming)**

Once available sources are fetched and the owner has either acted on or ignored the `?` rows for a few seconds, the agent emits `⟳ Drafting response on your behalf` and starts the SSE stream. Tokens stream into the draft area. When done: `✓ Drafted response`.

**Phase C — Conversational refinement**

After the initial draft completes, the owner can:
- **Tap "Send to caller"** → the current draft area text is sent as the final response
- **Tap "Decline"** → the consent request is declined
- **Edit the draft directly** → tap into the draft area, edit
- **Type an instruction in the input + tap "Ask agent"** → the agent treats the input as a refinement instruction

When refining, a new step row appears: `⟳ Refining: "make it shorter"`. The agent re-streams the draft, replacing the previous text. When done: `✓ Refined response`. The owner can refine multiple times — the activity panel becomes a transparent log of how the response evolved.

### The Two-Button Input Model

This is the key UX decision. With one text input, the owner needs to do two completely different things:
1. **Send the response to the requester** (the final action)
2. **Talk to the agent** to refine the draft

Solved with **explicit action buttons**, not auto-detection:

```
┌──────────────────────────────────────┐
│ Type to refine, or edit the draft... │
└──────────────────────────────────────┘
[Ask agent]              [Send to caller]
```

**Rules:**

- **"Send to caller"**: ALWAYS sends the **draft area** text. It does NOT use what's in the input box. The button label adapts to the channel: "Send to caller" / "Send via WhatsApp".
- **"Ask agent"**: takes the **input box** text as an instruction, sends it to the backend, and the agent re-drafts. After streaming, the input box clears.

**Why this design:**
- The owner ALWAYS knows what each tap does. No magical intent detection that gets it wrong.
- The draft area is "the thing being sent". The input box is "the thing being told to the agent". Two visually distinct zones, two clear actions.
- If the owner just wants to write the entire response themselves: tap into the draft area, edit, tap "Send to caller". Input box stays empty.
- If the owner wants the agent to handle everything: type "make it shorter", "Ask agent", review, "Send to caller".

**Button states:**

| State | "Ask agent" | "Send to caller" |
|---|---|---|
| Streaming in progress | Disabled | Disabled |
| Stream complete, draft non-empty, input empty | Disabled | Enabled |
| Stream complete, draft non-empty, input has text | Enabled | Enabled |
| Stream complete, draft empty | Disabled | Disabled (owner must type into draft area first) |

### Mid-Stream Interruption

The owner can interrupt at any point during streaming via three equivalent actions:
- **Tap "Take over"** (visible only during streaming) → cancels the SSE
- **Tap directly into the draft area** → same as "Take over" (typing IS taking over)
- **Type in the input box** → also cancels (the owner is moving to "talk to agent" mode)

All three:
1. Frontend closes the SSE connection
2. Frontend sends `DELETE /consent-request/{id}/draft-stream`
3. Backend aborts the LLM call cleanly
4. The current step row updates to `⊘ Drafting cancelled by you`
5. The draft area becomes editable, buttons re-enable

The DB record stays in `pending` — cancelling the draft doesn't cancel the consent request.

### Special Cases

**OTP-only** (`data_sources == ["device_otp"]`): bypass the agentic UI entirely. Show the existing OTP-specific input (centered, number-pad, auto-fill enabled). Two buttons: "Send to caller" and "Decline". This preserves the existing OTP flow.

**All-unknown sources** (every source in `data_sources` is not in `{device_calendar, device_contact, device_otp}`): show a brief activity row `⊘ I don't have access to any of the requested data — please write your response manually`. Draft area is empty and editable from the start. "Ask agent" is disabled.

---

## Backend Architecture

### Streaming Endpoint

The frontend orchestrates the agentic flow as a state machine. The backend just exposes the LLM streaming endpoint and the existing consent action endpoint. The orchestration (which sources to fetch, when to draft, when to refine) lives entirely on the frontend because:
- Device sources can only be read on-device
- The owner's interactions (button taps, text input) are local to the UI
- Only the LLM streaming call needs server-side compute

**New endpoint**: `POST /consent-request/{id}/draft-stream`

Request body:
```json
{
  "fetched_data": { "device_calendar": [...], "device_contact": [...] },
  "instruction": "make it shorter"   // optional; absent on initial draft
}
```

Response: SSE stream of events:
```
event: step
data: {"step_id": "draft", "status": "started", "label": "Drafting response on your behalf"}

event: token
data: {"token": "Hi"}

event: token
data: {"token": " John"}

...

event: step
data: {"step_id": "draft", "status": "completed"}

event: done
data: {"final_text": "Hi John, I have a meeting at 3pm Thursday..."}
```

For a refinement, `step_id` is `refine` and `label` is `"Refining: \"<owner instruction>\""`.

**Cancel endpoint**: `DELETE /consent-request/{id}/draft-stream` — explicit cancel signal.

### Frontend Orchestration State Machine

```
Screen opens
  │
  ├── Fetch full request_payload via GET /consent-request/{id}/details
  │
  ├── Classify each source in conversation_context.data_sources:
  │     - If source ∈ {device_calendar, device_contact, device_otp}:
  │         check permissionService.checkPermission(source)
  │           granted → Available
  │           denied  → Missing — connectable
  │           blocked → Missing — blocked
  │     - Otherwise: Missing — unknown (silent skip)
  │
  ├── Render Agent Activity panel with initial step rows
  │
  ├── Start fetching data from available device sources in parallel
  │     Each fetch updates its row from ⟳ to ✓ (or ⚠ on failure)
  │
  ├── Wait briefly (3-5 seconds) for owner to act on ? rows
  │     - Tap [Grant Permission] → permissionService.requestPermission()
  │       → granted: row ✓, fetch, new ⟳
  │       → blocked: row updates to [Open Settings]
  │       → cancelled: row ⊘
  │
  ├── Once all fetches resolved (success/fail/skip):
  │     - Open SSE: POST /consent-request/{id}/draft-stream with fetched_data
  │     - Add ⟳ Drafting response step
  │     - Stream tokens into draft area
  │     - On done: step ✓
  │
  ├── Owner interacts:
  │     - Edits draft directly → input is now just for refinement
  │     - Types in input + Ask agent:
  │         → cancels any active stream
  │         → POST /draft-stream with {fetched_data, instruction}
  │         → Add ⟳ Refining step
  │         → Stream replaces draft area content
  │         → On done: step ✓
  │         → Clear input box
  │     - Late permission grant after initial draft:
  │         → Re-fetch, automatically restream draft
  │         → If owner has manual edits: show non-destructive prompt
  │
  └── Owner taps "Send to caller":
        → POST /consent-response with draft area text + action="approved"
        → Routed to channel-specific delivery (Phase 2 handlers)
```

---

## Implementation Tasks

### 3.1 — Backend: Streaming Draft Endpoint
*Dependencies: Phase 1 + Phase 2 complete*

**File**: `trybot-api/app/services/consent_response_service.py` + `trybot-api/app/routes/consent_request.py`

| Task | Hours |
|------|-------|
| Adapt `consent_response_service` to use streaming LLM API (`stream=True`) | 1.5 |
| New endpoint `POST /consent-request/{id}/draft-stream` (SSE) | 1 |
| SSE event emitter: `step`, `token`, `done`, `error` events | 1 |
| Build LLM messages from `request_payload` + frontend-supplied `fetched_data` | 0.5 |
| Refinement support: optional `instruction` in request body. When present, build messages as a follow-up turn (system prompt + previous draft + owner's instruction) | 1.5 |
| Cancel handling: detect client disconnect, abort LLM call cleanly | 1 |
| Cancel endpoint `DELETE /consent-request/{id}/draft-stream` | 0.5 |
| New endpoint `GET /consent-request/{id}/transcript` — returns transcript segments up to `transcript_snapshot_seq` (for calls) or WhatsApp messages up to `created_at` (for WhatsApp). Called on demand when the owner expands the transcript section. | 1 |
| Unit tests (mock streaming + cancel + refinement + transcript fetch) | 2 |
| **Subtotal** | **10** |

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
*Dependencies: 3.2*

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
*Dependencies: 3.1 (backend SSE), 3.3 (classification)*

The most substantial frontend phase. Builds the entire agentic interaction model.

| Task | Hours |
|------|-------|
| Install + configure `react-native-sse` (or fetch-streaming polyfill) | 0.5 |
| `useAgenticDraftStream` hook: opens SSE, dispatches step/token/done events to state, handles cancel | 2.5 |
| `AgentActivityPanel.tsx` — the step rows panel with all status states | 2 |
| `AgentStepRow.tsx` — single row with inline action button (Grant Permission / Open Settings) | 1.5 |
| Native permission grant integration via existing `permissionService.ts` — handle granted / denied / blocked / settings deep-link | 1.5 |
| `DraftArea.tsx` — streaming draft display with blinking cursor and direct edit support | 2 |
| Two-button input row: input box + "Ask agent" + "Send to caller" + "Decline" | 1 |
| Conversational refinement loop: typing in input → "Ask agent" → POST `/draft-stream` with `instruction` → re-stream replaces draft → input clears | 2 |
| Mid-stream cancel logic (Take over button + tap-to-edit + type-in-input all cancel) | 1 |
| Frontend orchestration state machine: classify → emit steps → fetch in parallel → wait briefly for ? actions → start draft stream → handle refinement turns | 2.5 |
| Re-stream after late permission grant: handle "owner has edited the draft" case with non-destructive prompt | 1 |
| OTP-only special case rendering (preserve existing flow) | 0.5 |
| All-unknown special case rendering | 0.5 |
| Unit tests (mock SSE, cancel, refinement, classification, step state transitions) | 2 |
| **Subtotal** | **20.5** |

### 3.6 — Frontend: Integrate Agentic Screen Into Existing Surfaces
*Dependencies: 3.4, 3.5*

| Task | Hours |
|------|-------|
| Replace contents of `InformationConsentScreen.tsx` (modal during call) with the new agentic layout (header → activity panel → draft area → two-button input) | 2 |
| Replace contents of `ConsentRequestDetailScreen.tsx` (full screen from call logs) with the same layout | 2 |
| Update Redux `informationConsentSlice` (add agent activity state, classification state, refinement history) | 0.5 |
| Update FCM handler `informationConsentHandler.ts` (no major change since payload shape from Phase 1 stays the same) | 0.5 |
| Unit tests | 1 |
| **Subtotal** | **6** |

### 3.7 — Phase 3 End-to-End Integration Testing
*After all of Phase 3 complete*

| Task | Hours |
|------|-------|
| Inbound call regression test (Phase 1 + 2 unhold paths still work) | 1 |
| Outbound call full flow with new agentic UI (intent → consent → agentic drafting → "Send to caller" → session injection → next turn) | 1 |
| WhatsApp full flow with new agentic UI (inbound → consent → agentic drafting → "Send to caller" → outbound message delivery) | 1 |
| Agentic flow happy path: classify → fetch → draft stream → "Send to caller" | 1 |
| Conversational refinement: 2-3 "Ask agent" rounds, verify draft replaces correctly + activity log accumulates | 1 |
| Mid-stream cancel via Take over button + tap-into-draft + type-in-input | 1 |
| Missing device permission on Android: ? row → grant → fetch → re-stream | 1 |
| Missing device permission on iOS (same) | 1 |
| Permanently denied permission (NEVER_ASK_AGAIN / BLOCKED): ? row shows "Open Settings", deep-link verified | 1 |
| Late permission grant: complete initial draft, then grant a missing permission, verify "owner edited?" prompt | 1 |
| Mixed device + unknown sources (partial draft, owner supplements manually) | 0.5 |
| All-unknown special case: blank draft, "Ask agent" disabled | 0.5 |
| OTP-only special case (regression - preserve existing flow) | 0.5 |
| Android build + on-device test across all 3 channels + full agentic flow | 2 |
| iOS build + on-device test across all 3 channels + full agentic flow | 2 |
| Bug fixes from integration testing | 2.5 |
| **Subtotal** | **17** |

---

## Phase 3 Total Effort

| Sub-phase | Hours |
|---|---|
| 3.1 — Backend streaming draft endpoint + transcript fetch endpoint | 10 |
| 3.2 — Backend analyzer awareness refinement | 2 |
| 3.3 — Frontend source classification helpers | 1.5 |
| 3.4 — Frontend header sub-components | 4 |
| 3.5 — Frontend agentic consent screen | 20.5 |
| 3.6 — Frontend integrate agentic screen | 6 |
| 3.7 — Phase 3 E2E integration testing | 17 |
| Buffer | 6 |
| **Phase 3 Total** | **67 hours (~8.5 days)** |

---

## Verification Criteria for Phase 3

Phase 3 is complete and shippable when ALL of the following are true:

1. ✅ Phase 1 + Phase 2 regression: all three channels still trigger consent and route the response correctly
2. ✅ Opening a consent screen (any channel) shows the agentic layout: header + activity panel + draft area + two-button input
3. ✅ The agent activity panel populates step rows in real time as fetches complete
4. ✅ Tokens stream into the draft area (visible character-by-character or chunk-by-chunk)
5. ✅ Tapping "Send to caller" sends the draft area text as the final response (not the input box)
6. ✅ Tapping "Ask agent" with an instruction triggers a refinement that re-streams into the draft area
7. ✅ Refinement can be done multiple times in a row, each adding a row to the activity panel
8. ✅ Mid-stream cancel works via Take over button, tap-into-draft, and type-in-input
9. ✅ Missing device permission shows a `?` row with [Grant Permission] button that triggers the native dialog
10. ✅ Permanently denied permission shows [Open Settings] and deep-links correctly
11. ✅ Late permission grant after initial draft triggers re-fetch + restream, with non-destructive prompt if owner had edited
12. ✅ OTP-only special case still works (existing flow preserved)
13. ✅ All-unknown special case shows blank input with "Ask agent" disabled
14. ✅ All three channels deliver the response correctly through Phase 2's handlers
15. ✅ Android device build + on-device test passes for all flows
16. ✅ iOS device build + on-device test passes for all flows

---

## Risk Notes

- **P3.5 is the dominant risk** at 20.5 hours. This is where most of the new agentic UX work lives. If the team is unfamiliar with React Native streaming patterns, add 3-5h buffer.
- **Device builds**: Android (Day 7) and iOS (Day 8) are split across two days to avoid back-to-back wall-clock time.
- **Backend cancel handling**: The LLM streaming abort needs to flush properly so the next request doesn't see leaked state. Test the cancel-then-retry path explicitly.
- **Re-stream after late permission grant** is a subtle case — make sure the "owner edited" detection is reliable (don't blow away their work).

---

## What This Phase Does NOT Touch in Earlier Phases

**Phase 1 code stays intact:**
- DB schema (Phase 1 added the `request_payload` JSONB; Phase 3 uses it as-is)
- `ConsentRequestProcessor` (Phase 1's channel-agnostic API is unchanged)
- `consent_action_service.handle_consent_action` channel dispatch — stays as-is

**Phase 2 code stays intact:**
- `_handle_outbound_call_consent`, `_handle_whatsapp_consent` — these only run on the *final* "Send to caller" / "Decline" action, not during the agentic drafting flow. Phase 3's drafting is purely a UI + streaming concern; the channel-specific delivery happens after.
- Channel detection in `voice_service.process_outbound_speech_with_disconnect` and `whatsapp_service._maybe_auto_reply` — unchanged
