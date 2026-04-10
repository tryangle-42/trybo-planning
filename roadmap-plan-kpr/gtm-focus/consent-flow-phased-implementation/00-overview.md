# Cross-Channel Consent Flow — Phased Implementation Roadmap

## Context

The consent flow currently only works for **incoming calls**. The trigger lives in `voice_service.py:process_inbound_speech()` which calls `generate_inbound_response_with_intent_detection()` and routes to `_handle_owner_info_request_intent()`. Outbound calls and WhatsApp have no consent integration. The consent screen is a one-shot draft generator with a basic edit-and-send UI.

This is the **implementation plan** that delivers the design described in [`../consent-flow-cross-channel-enhancement.md`](../consent-flow-cross-channel-enhancement.md). The design doc is the spec; this folder is the rollout plan.

The design is delivered through:
1. A reusable, channel-agnostic consent request component
2. Cross-channel support (incoming calls, outbound calls, WhatsApp)
3. A truly agentic consent screen with token-by-token streaming and conversational refinement

The work is broken into **three independently shippable phases**. At every phase boundary, the consent functionality is fully working — nothing is half-built. Later phases may modify earlier-phase code; that's expected and considered part of the later phase's scope.

---

## The Three Phases

### Phase 1 — Extract a Reusable Consent Request Component
**Goal**: Take the existing inbound-call consent functionality and refactor it into a clean, channel-agnostic component. **No behavior change visible to the user.** Existing inbound-call consent continues to work end-to-end.

**Why this phase first**: We need a clean foundation before we can plug new channels into it. The current code is intertwined with inbound-call specifics (TwiML, hold/unhold, FCM payloads with hardcoded `"channel": "call"`, etc.). Phase 1 extracts that into a reusable shape so Phase 2 can plug other channels in cleanly.

See: [phase-1-extract-consent-component.md](./phase-1-extract-consent-component.md)

### Phase 2 — Multi-Channel Support
**Goal**: Make outbound calls and WhatsApp able to trigger the consent request component. The current screens still display the request, just now from more sources.

**Why this phase second**: The reusable component from Phase 1 is in place; Phase 2 wires the new channel agents into it and adds the channel-specific response delivery handlers. Existing screens get a small update to display the channel correctly, but the core UI is unchanged.

See: [phase-2-multi-channel-support.md](./phase-2-multi-channel-support.md)

### Phase 3 — Agentic UI with Token Streaming
**Goal**: Replace the existing consent screens with a true agentic experience. Token-by-token streaming, an agent activity panel with step rows, "Ask agent" conversational refinement, and inline permission grant flows.

**Why this phase last**: This is the largest single chunk of work. Phases 1 and 2 give us a stable, multi-channel consent system to build the new UI on top of. None of the previous phases need to be undone — they're upgraded in place.

See: [phase-3-agentic-ui-with-streaming.md](./phase-3-agentic-ui-with-streaming.md)

---

## Phasing Principles

1. **Always shipped, always working**: At the end of each phase, the consent functionality is fully working end-to-end. No phase leaves the system in a broken or half-finished state.
2. **Backward-compatible refactoring is OK within a phase**: Phase 1's job is specifically to refactor without behavior change. The DB migration backfills existing rows so the existing screen reads them correctly via the new code paths.
3. **Later phases can change earlier phases**: If Phase 3 needs Phase 1's `consent_action_service` to do something differently, that change happens in Phase 3 and is part of Phase 3's scope. We don't try to over-engineer Phase 1 to anticipate Phase 3.
4. **Each phase has its own verification**: Every phase ends with a regression-and-validation pass that proves the consent flow still works (and the new functionality of that phase works too).

---

## Cross-Cutting Design (Stable Across All Phases)

These design principles apply across all three phases. Each phase implements them progressively.

### 1. Who Prepares What?

When any channel decides "I need the owner's consent to share information", **the channel agent does the analysis work and hands the consent component a ready-to-use package**. The consent component does NOT re-read transcripts or re-analyze the conversation.

**Why**:
- The channel agent already has the conversation in memory. Re-reading is wasteful.
- The channel agent's LLM call (`InformationRetrievalAnalyzer`) already determines the summary, data sources, missing data sources. This is context-dependent intelligence.
- The consent component should be a thin pipe: receive a structured request, persist it, send a notification, route the response back.

### 2. The Channel Agent's Input Contract

When any channel calls the consent component, it passes two things:

**(a) Relational identifiers** (separate DB columns — need FKs, indexes, RLS):
```
channel:                    "inbound_call" | "outbound_call" | "whatsapp"
user_id:                    the owner's ID
call_id:                    (nullable) FK to calls
task_id:                    (nullable) FK to bot_tasks
whatsapp_conversation_id:   (nullable) FK to whatsapp_conversations
```

**(b) The request payload** (single JSONB column — opaque blob, written once, read as a whole):
```json
{
  "summary": "John is asking for your schedule this Thursday",
  "information_retrieval_prompt": "Retrieve calendar events for Thursday...",
  "requester_name": "John Smith",
  "requester_number": "+1234567890",
  "contact_id": "uuid-or-null",
  "conversation_context": {
    "turns": [{"role": "caller", "text": "..."}, ...],
    "data_sources": ["device_calendar", "device_contact"],
    "missing_data_sources": ["device_calendar"]
  }
}
```

`conversation_context` bundles the transcript snapshot, the data sources needed, and the missing data sources into a single object — they all come from the same analyzer pass.

### 3. Why the Schema Split?

- **Separate columns** for things Postgres needs to enforce or query: FKs (referential integrity), `channel` (indexed dispatch), lifecycle state (status, timestamps — queried/updated independently).
- **Single JSONB (`request_payload`)** for the opaque request blob — written once, read as a whole, never queried against internally.

**The resulting `consent_requests` table shape** (after Phase 1 migration):
```
consent_requests:
  -- Identity & relationships (separate columns)
  id                          uuid PK
  user_id                     uuid FK -> user_profiles
  call_id                     uuid FK -> calls (nullable)
  task_id                     uuid FK -> bot_tasks (nullable)
  whatsapp_conversation_id    uuid FK -> whatsapp_conversations (nullable)
  channel                     text CHECK (IN 'inbound_call','outbound_call','whatsapp')

  -- The request payload (single JSONB, opaque)
  request_payload             jsonb

  -- Lifecycle state (separate columns)
  status                      text (pending|approved|declined|expired|canceled|delivered)
  expires_at                  timestamptz
  responded_at                timestamptz
  delivered_response          text
  delivered_at                timestamptz
  created_at                  timestamptz
  updated_at                  timestamptz
```

The existing `information_prompt` and `suggested_data_sources` columns are dropped during Phase 1's migration; their data is backfilled into `request_payload`.

### 4. Data Sources In Scope (Existing Only)

This component works only with the **device sources currently implemented** in the codebase:

| Source Key | What It Is | Permission |
|---|---|---|
| `device_calendar` | Calendar events from the phone's built-in calendar | Native OS permission (Android `READ_CALENDAR`, iOS `CALENDARS`) |
| `device_contact` | Contacts on the phone | Native OS permission (Android `READ_CONTACTS`, iOS `CONTACTS`) |
| `device_otp` | One-time password from SMS | User input via auto-fill — no permission needed |

A future "Third-Party Data Sources" component (Composio-based: Gmail, Outlook, Slack, etc.) is planned separately and is **out of scope** for this work. When the LLM suggests a source key that isn't one of these three (e.g., `gmail`, `google_calendar`, `slack`), it's classified as `missing — unknown` and falls through to manual response.

### 5. What the Consent Component Does NOT Do

- Does NOT analyze conversations
- Does NOT determine if consent is needed
- Does NOT figure out data sources
- Does NOT generate summaries
- Does NOT fetch data from the owner's device (that's the UI's job)
- Does NOT know or care about TwiML, LiveKit, WhatsApp APIs (those are in the channel-specific response handlers)

The consent component is a **channel-agnostic message broker** between channel agents and the device owner.

---

## Cross-Cutting: Edge Cases (Apply to All Phases)

These edge cases apply across all phases. Each phase handles them at its level of capability.

### Source Is Missing — Unknown
The LLM suggests a source key not in `{device_calendar, device_contact, device_otp}` (e.g., `gmail`, `bank_records`).
- The consent component stores it as-is in `request_payload` (no validation)
- The UI does not auto-fetch and does not show a connect/grant button
- If it's the only requested source: blank input, owner writes manually
- If there are also recognized sources: agent drafts using just the recognized ones, owner supplements manually

### Mixed Known and Unknown Sources
e.g., `data_sources: ["device_calendar", "gmail"]`
- Calendar is fetched; Gmail is silently skipped
- Draft is generated using only calendar data
- Owner supplements manually via direct edit or "Ask agent"

### Consent Request After Call/Chat Has Ended
- **For calls**: If the owner responds after the call has ended, the response is recorded in DB for audit, status marked `expired`, owner sees a message
- **For WhatsApp**: WhatsApp messages can always be sent (async), no expiration

### Multiple Consent Requests in the Same Session
Each triggers a separate consent request with its own DB record, notification, and response cycle. The channel agent tracks active request IDs in its session state.

---

## Total Effort Across All Phases

| Phase | Hours | Days (8h/day) |
|---|---|---|
| Phase 1 — Extract reusable consent component | ~18 | ~2.5 |
| Phase 2 — Multi-channel support | ~20 | ~2.5 |
| Phase 3 — Agentic UI with streaming | ~66 | ~8 |
| **Grand total** | **~104 hours** | **~13 working days** |

Each phase ships independently. The user can pause between phases without breaking anything.

---

## File Index

- [00-overview.md](./00-overview.md) — this file
- [phase-1-extract-consent-component.md](./phase-1-extract-consent-component.md) — Phase 1 detailed plan
- [phase-2-multi-channel-support.md](./phase-2-multi-channel-support.md) — Phase 2 detailed plan
- [phase-3-agentic-ui-with-streaming.md](./phase-3-agentic-ui-with-streaming.md) — Phase 3 detailed plan
