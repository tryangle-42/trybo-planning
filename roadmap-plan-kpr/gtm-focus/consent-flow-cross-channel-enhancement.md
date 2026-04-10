# Cross-Channel Consent Flow - System Design & Flow Logic

## What This Document Covers

This is not an implementation plan. This document describes:
- How any channel intelligence (voice agent, WhatsApp agent) communicates the need for a consent request to the Consent Request Component
- Who prepares what data (the channel agent vs the consent component)
- How the Consent Request Component prepares and sends a channel-agnostic notification
- How the UI renders differently based on conditions (OTP, text, missing permissions, undefined data sources)
- How the response flows back from the owner through the consent component to the originating channel

---

## 1. The Core Question: Who Prepares What?

### Decision: The Calling Channel Agent Prepares Everything

When any channel's intelligence (inbound voice agent, outbound voice agent, WhatsApp agent) decides that a consent request is needed, **the channel agent does the analysis work and hands the consent component a ready-to-use package**. The consent component does NOT re-read transcripts or re-analyze the conversation.

**Why this decision:**
- The channel agent already has the conversation in memory (session history, WhatsApp thread). Re-reading it would be wasteful duplication.
- The channel agent's LLM call (`InformationRetrievalAnalyzer`) already determines the summary, data sources, and missing data sources. This analysis is context-dependent - it needs to understand the tone, intent, and flow of the conversation. The consent component is a delivery mechanism, not an intelligence layer.
- The consent component should be a thin, channel-agnostic pipe: receive a structured request, create a DB record, send a notification, and later handle the owner's response. If we make it re-analyze, it becomes coupled to each channel's conversation format.

### What the Channel Agent Provides to the Consent Component

When any channel decides "I need the owner's consent to share information", it calls the Consent Request Component with two things:

**1. Relational identifiers (stored as separate DB columns - need FKs, indexes, RLS):**
```
channel:                    "inbound_call" | "outbound_call" | "whatsapp"
user_id:                    the owner's ID
call_id:                    (nullable) FK to calls table
task_id:                    (nullable) FK to bot_tasks table
whatsapp_conversation_id:   (nullable) FK to whatsapp_conversations table
```

**2. The request payload (stored as a single JSONB column - opaque blob, written once, read as a whole):**
```json
{
  "summary":                      "John is asking for your schedule this Thursday",
  "information_retrieval_prompt":  "Retrieve calendar events for Thursday...",
  "requester_name":               "John Smith",
  "requester_number":             "+1234567890",
  "contact_id":                   "uuid-or-null",
  "conversation_context": {
    "transcript_id":              "uuid-of-existing-transcript",
    "transcript_snapshot_seq":    12,
    "data_sources":               ["device_calendar", "device_contact"],
    "missing_data_sources":       ["device_calendar"]
  }
}
```

`conversation_context` bundles the analysis output from the channel agent: which data sources are needed, which are missing, and a **reference** to the conversation transcript (not a copy of it). The transcript already lives in `transcript_segments` (for calls) or `whatsapp_messages` (for WhatsApp) — storing it again inside `request_payload` would be redundant storage.

- `transcript_id` — points to the existing transcript record in the `transcripts` table. The UI fetches segments on demand when the owner expands the transcript section (collapsed by default).
- `transcript_snapshot_seq` — the last segment sequence number at the moment the consent request was created. When fetching, the UI queries `transcript_segments WHERE transcript_id = X AND seq <= snapshot_seq` so the owner sees exactly what was said up to the consent request — not turns that happened after (e.g., hold messages, or continued outbound agent conversation). For WhatsApp, the equivalent is filtering `whatsapp_messages` by `created_at <= consent_request.created_at`.

### Why This Split?

The consent_requests table follows a clear principle:

- **Separate columns** for things Postgres needs to enforce or query: FKs (call_id, task_id, whatsapp_conversation_id with referential integrity), channel discriminator (indexed, used in dispatch), lifecycle state (status, timestamps - queried and updated independently)
- **Single JSONB (`request_payload`)** for the opaque request data the channel agent assembles and the UI consumes as a whole. This blob is written once at creation, never queried against (you never search "find all requests where missing_data_sources contains X"), and always read in full when the UI opens.

This makes the consent component a true pass-through: the channel agent builds one JSON blob, the component stores it as-is, the notification carries it, the UI unpacks it. Nobody in between inspects or transforms the payload's internal structure.

**The resulting table shape:**
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

  -- Lifecycle state (separate columns - queried, indexed, updated independently)
  status                      text (pending|approved|declined|expired|canceled|delivered)
  expires_at                  timestamptz
  responded_at                timestamptz
  delivered_response          text
  delivered_at                timestamptz
  created_at                  timestamptz
  updated_at                  timestamptz
```

Note: The existing `information_prompt`, `suggested_data_sources`, and any future context fields all live inside `request_payload`. The `conversation_context` object within the payload holds the transcript, data sources, and missing data sources together as one analysis output.

### What the Consent Component Does NOT Do

- Does NOT read or analyze the transcript/conversation
- Does NOT determine which data sources are needed
- Does NOT generate the summary
- Does NOT decide whether a consent request should be triggered (that's the channel agent's job)

The consent component is a **stateless processor**: it takes the package, persists it, sends a notification, and later routes the owner's response back to the correct channel.

---

## 2. How the Consent Request Component Processes the Request

The Consent Request Component receives the package and does exactly three things:

### Step 1: Persist to Database

Create a `consent_requests` row:
- Relational columns: `channel`, `user_id`, `call_id`, `task_id`, `whatsapp_conversation_id`
- `request_payload`: the entire JSONB blob from the channel agent (summary, data sources, missing sources, conversation context, requester identity - all in one field)
- `status: "pending"`

This record is the single source of truth for the consent request lifecycle.

### Step 2: Send Push Notification (FCM)

Send a push notification to the owner's device. The FCM payload is a lightweight projection of `request_payload` plus the `request_id`:
- `request_id` (so the UI can fetch the full record - including `conversation_context` with turns, data_sources, missing_data_sources - from DB)
- `channel` (so the UI knows what kind of request this is)
- `summary` (extracted from request_payload, for the notification text)
- `requester_name` / `requester_number` (extracted from request_payload, for display)

The notification is channel-agnostic. The FCM payload carries enough for the notification banner. When the UI opens, it fetches the full `request_payload` from DB using `request_id` to get the complete `conversation_context` (transcript, data sources, missing sources).

### Step 3: Return the Request ID

Return the `consent_request_id` to the calling channel agent, so the agent can track it in its session (e.g., `active_consent_request_id` in the call session).

---

## 3. Data Sources: How the Consent Component Knows What's Available

This component works only with the **device sources currently implemented** in the codebase. It defers to the existing `permissionService.ts` for permission state and `informationConsentService.ts` for fetching.

A future "Third-Party Data Sources" component (Composio-based: Gmail, Outlook, Slack, etc.) is planned separately and is **out of scope for this iteration**. This consent flow is built so that adding more source categories later requires only extending the classification step — none of the agentic UI or backend streaming needs to change.

### 3.1 The Existing Device Sources

| Source Key | What It Is | How Permission Works | Fetch Method |
|---|---|---|---|
| `device_calendar` | Calendar events from the phone's built-in calendar app | Native OS permission dialog (Android `READ_CALENDAR`, iOS `CALENDARS`) | `informationConsentService.fetchDeviceCalendar()` (existing) |
| `device_contact` | Contacts stored on the phone | Native OS permission dialog (Android `READ_CONTACTS`, iOS `CONTACTS`) | `informationConsentService.fetchContacts()` (existing) |
| `device_otp` | One-time password from SMS | User provides via keyboard auto-fill (no permission dialog needed) | User input — no auto-fetch |

These three are the **only** sources the consent component will recognize. Any other source key the LLM suggests is treated as unknown.

### 3.2 What the Channel Agent Sends

The channel agent's `InformationRetrievalAnalyzer` outputs source keys in `conversation_context.data_sources` and `conversation_context.missing_data_sources`. The consent screen classifies each one at open time:

| Classification | Meaning | How decided |
|---|---|---|
| **Available** | Device source AND permission already granted | `permissionService.checkPermission(source)` returns granted |
| **Missing — connectable** | Device source AND permission denied but can be re-requested | `permissionService.checkPermission(source)` returns denied (and not permanently blocked) |
| **Missing — blocked** | Device source AND permission permanently denied (user must go to Settings) | `permissionService` reports `NEVER_ASK_AGAIN` / `BLOCKED` state |
| **Missing — unknown** | Source key is NOT one of the three known device sources | Source is not in `{device_calendar, device_contact, device_otp}` |

### 3.3 Why No Local Registry in This Component

We do NOT define a separate registry. The set of three device source keys is small enough to inline as a constant, and `permissionService.ts` already owns the permission semantics for each one. Adding a registry would create drift.

When the future Third-Party Data Sources component lands, it will introduce its own classification source (e.g., a connection-check API). At that point, the classification function in this component will be extended with one extra branch — the rest of the consent screen (activity panel, step rows, draft area, refinement loop) will not change.

### 3.4 What Happens When a Source Is "Missing — Unknown"

If the LLM suggests `"bank_records"`, `"gmail"`, `"google_calendar"`, `"slack_messages"`, or any key that isn't in the three known device sources:

1. The consent component stores it as-is in `request_payload` (no validation)
2. At screen open, classification returns **missing — unknown**
3. The agent does NOT emit a step row for it (no action available)
4. The agent does NOT attempt auto-fetch
5. The agentic drafting is skipped IF this is the only source — the draft area is shown empty for manual writing (Section 4.11)
6. If there are also recognized device sources, the agent drafts using those and the owner manually adds info for the unknown source

**The principle**: We only auto-prepare when we can actually access the data. Anything else, defer to the human.

---

## 4. How the UI Renders Based on Conditions

When the owner opens the consent screen (either from the push notification or from call logs), the UI has all the metadata it needs from the notification payload + DB record. Here's the rendering logic:

### 4.1 Input Mode: Determined by Data Sources + Registry

The UI checks the `data_sources` array against the registry to decide what input to show:

```
IF data_sources contains ONLY "device_otp":
  -> Show OTP-specific input (centered, number-pad, large font, auto-fill enabled)
  -> Bypass the agentic flow entirely (Section 4.10)
  -> User just enters the OTP code

ELSE IF data_sources contains ANY known device source
        (device_calendar or device_contact):
  -> Show the AGENTIC CONSENT SCREEN (Sections 4.2–4.8)
  -> Agent classifies sources, requests missing permissions via step rows,
     fetches data, streams a draft, and supports refinement via "Ask agent"
  -> Owner can:
       - Edit the draft directly
       - Tap "Ask agent" with an instruction to refine the draft
       - Tap "Send to caller" to deliver the draft as the final response
       - Tap "Decline" to refuse

ELSE IF every source is missing — unknown
        (LLM asked for sources we don't support yet, e.g., gmail, slack):
  -> Show blank draft area, "Ask agent" disabled
  -> Owner writes the response manually (Section 4.11)
```

### 4.2 The Agentic Consent Screen

The consent screen behaves like a **conversational agent** the owner is collaborating with. The agent handles three jobs in real time:

1. **Asks for missing access** when it can't reach the data it needs (and waits for the owner to grant it as a step in the flow)
2. **Drafts the response on the owner's behalf** with token-by-token streaming once it has the data
3. **Refines the draft conversationally** when the owner gives instructions like "make it shorter" or "add that I'll be late"

There is **one input box** at the bottom of the screen and **two action buttons** next to it:

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

**The screen has these zones, top to bottom:**

1. **Header**: channel badge + requester identity + the original information request summary + collapsible transcript
2. **Agent Activity panel**: a stack of step rows showing what the agent is doing (or has done, or needs help with)
3. **Draft area**: the agent's current response, streamed in token-by-token, becomes editable when streaming finishes
4. **Input + actions**: one text input, three buttons — "Ask agent", "Send to caller", "Decline"

### 4.3 The Agent Activity Panel — Steps Are First-Class

The Agent Activity panel is what makes the flow feel agentic. Every action the agent takes (or wants to take) appears as a row:

| Step icon | Meaning | Example labels |
|---|---|---|
| `⟳` | In progress | "Reading your Google Calendar..." |
| `✓` | Completed | "Read your contacts" |
| `⚠` | Failed (non-blocking) | "Couldn't reach your Outlook calendar" |
| `⊘` | Cancelled / skipped | "Drafting cancelled by you" |
| `?` | Needs the owner's action | "Need access to your Gmail to read recent emails" |

A `?` step row is special: it has an inline action button right inside the row. The agent literally renders a step that says "I need this — here's the button to grant it":

```
? Need calendar access to read your schedule
  [Grant Permission]
```

When the owner taps the button, the native OS permission flow runs via `permissionService.ts`. On success, that step row updates: `✓ Granted calendar access`. The agent then continues — adds new step rows for fetching, then drafting, then streaming into the input.

If the OS reports that permission was permanently denied (Android `NEVER_ASK_AGAIN`, iOS `BLOCKED`), the row updates to: `? Calendar denied. Open Settings to enable it. [Open Settings]` — tapping deep-links into the app's settings page.

If the owner ignores the `?` step (doesn't tap the button), the agent waits a few seconds, then proceeds without that source: it adds a `⊘ Skipped device calendar (no permission)` step and continues drafting with whatever data IS available. The owner is never blocked.

### 4.4 The Three Phases of an Agentic Drafting Run

**Phase A — Discovery and access requests**

When the screen opens, the agent classifies each source from `request_payload.conversation_context.data_sources` (Section 3.2). For each:

- **Available** → emit `⟳ Reading your [source]` step, fetch the data, update to `✓`
- **Missing — connectable** → emit `? Need [source] permission` step with `[Grant Permission]` button
- **Missing — blocked** → emit `? [Source] permission denied — open Settings to enable` step with `[Open Settings]` button
- **Missing — unknown** → don't emit any step (silent skip; the source isn't actionable)

The owner sees several rows appear quickly. Some are checkmarks (already done), some are spinners (in progress), some are `?` rows asking for permission.

**Phase B — Drafting (token streaming)**

Once all available sources are fetched (and the owner has either acted on or ignored the `?` rows for a few seconds), the agent emits a `⟳ Drafting response on your behalf` step and starts the SSE stream. Tokens stream into the draft area above the input. When done, the step row becomes `✓ Drafted response`.

**Phase C — Conversational refinement**

After the initial draft completes, the owner can:
- **Tap "Send to caller"** → the current draft text is sent as the final response (Section 5)
- **Tap "Decline"** → the consent request is declined
- **Edit the draft directly** → the input area becomes the draft itself, owner can type/edit anywhere
- **Type an instruction in the input + tap "Ask agent"** → the agent treats the input text as a refinement instruction, NOT as the response

When the owner asks the agent for a refinement, a new step row appears:

```
✓ Drafted response
⟳ Refining: "make it shorter and more casual"
```

The agent re-streams the draft into the same draft area, replacing the previous text token-by-token. When done: `✓ Refined response`.

The owner can keep refining as many times as they want. Each refinement becomes a step row in the activity panel — the panel becomes a transparent log of how the response evolved.

### 4.5 Two-Button Input: How One Input Box Serves Both Modes

This is the key UX decision. With one text input, the owner needs to be able to do two completely different things:
1. **Send the response to the requester** (the final action)
2. **Talk to the agent** to refine the draft

We solve this with **explicit action buttons**, not auto-detection:

```
┌──────────────────────────────────────┐
│ Type to refine, or edit the draft... │
└──────────────────────────────────────┘
[Ask agent]              [Send to caller]
```

**Rules for the two buttons:**

- **"Send to caller"**: ALWAYS sends the **draft area text** (the area above the input — the current best draft) as the final response. It does NOT use what's in the input box. This is the green CTA. The button label includes the channel context: "Send to caller" / "Send via WhatsApp" depending on `channel`.

- **"Ask agent"**: takes the **input box text** as an instruction, sends it to the backend, and the agent re-drafts based on that instruction. After the new draft streams in, the input box clears and waits for the next instruction (if any).

**Why this design:**
- The owner ALWAYS knows what tapping each button will do. No magic intent detection that gets it wrong.
- The draft area is the "thing being sent". The input box is the "thing being told to the agent". Two visually distinct zones with two clear actions.
- If the owner just wants to write the entire response themselves, they can: tap into the draft area, edit (or replace) the text, then tap "Send to caller". The input box can stay empty.
- If the owner just wants the agent to handle everything, they can: type "make it shorter, and friendlier" in the input, tap "Ask agent", review, then tap "Send to caller".

**State of the buttons:**

| State | "Ask agent" | "Send to caller" |
|---|---|---|
| Streaming in progress | Disabled | Disabled |
| Stream complete, draft non-empty, input empty | Disabled | Enabled |
| Stream complete, draft non-empty, input has text | Enabled | Enabled |
| Stream complete, draft empty | Disabled | Disabled (owner must type something into the draft area first) |

### 4.6 Mid-Stream Interruption

The owner can interrupt at any point during streaming:

- **Tap "Take over"** (visible only during streaming) → cancels the SSE, unlocks the draft area for direct editing, draft contains whatever was streamed so far
- **Tap directly into the draft area** → same as "Take over" (typing IS taking over)
- **Type in the input box** → also cancels the current stream (the owner is moving to "talk to agent" mode), keeping whatever was streamed so far in the draft

In all three cases:
1. Frontend closes the SSE connection
2. Frontend sends `DELETE /consent-request/{id}/draft-stream` (explicit cancel signal)
3. Backend aborts the LLM call cleanly
4. The current step row updates to `⊘ Drafting cancelled by you`
5. The draft area becomes editable, the buttons re-enable

The DB record stays in `pending` state — cancelling the draft doesn't cancel the consent request itself.

### 4.7 The Backend's Role in the Agentic Flow

The backend exposes the streaming endpoint(s) but doesn't orchestrate the agent loop. The orchestration (which sources to fetch, when to draft, when to refine) happens on the **frontend**, because:
- Device sources can only be read on-device
- The owner's interactions (button taps, text input) are local to the UI
- Only the LLM streaming call needs server-side compute

**Backend endpoints:**

| Endpoint | Purpose |
|---|---|
| `GET /consent-request/{id}/details` | Loads the full `request_payload` when the screen opens |
| `GET /consent-request/{id}/transcript` | Returns transcript segments up to `transcript_snapshot_seq` for calls (from `transcript_segments`), or WhatsApp messages up to `created_at` (from `whatsapp_messages`). Called on demand when the owner expands the transcript section. |
| `POST /consent-request/{id}/draft-stream` | The streaming draft endpoint. Body: `{ fetched_data, instruction }`. Returns SSE stream of step + token + done events. `instruction` is optional — present when the owner is asking for a refinement, absent on the initial draft. |
| `DELETE /consent-request/{id}/draft-stream` | Explicit cancel signal to abort an in-progress draft stream |

Permission state for device sources is checked entirely on the device via the existing `permissionService.ts` — no backend endpoint needed for that.

**SSE event types from `/draft-stream`:**

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

For a refinement request, `step_id` is `refine` and `label` is `"Refining: \"<owner's instruction>\""`.

### 4.8 Frontend Orchestration Flow

The frontend orchestrates the entire agentic flow as a state machine:

```
Screen opens
  │
  ├── Fetch full request_payload via GET /consent-request/{id}/details
  │
  ├── Classify each source in conversation_context.data_sources:
  │     - If source ∈ {device_calendar, device_contact, device_otp}:
  │         check permissionService.checkPermission(source)
  │           granted   → Available
  │           denied    → Missing — connectable
  │           blocked   → Missing — blocked (must open Settings)
  │     - Otherwise: Missing — unknown (silent skip)
  │
  ├── Render Agent Activity panel with initial step rows:
  │     - "✓ Read your contacts" (already permitted, will fetch in next step)
  │     - "? Need calendar access [Grant Permission]"
  │     - device_otp doesn't appear here (it's user-input, not a permission)
  │
  ├── Start fetching data from available device sources in parallel
  │     - device_calendar → fetchDeviceCalendar()
  │     - device_contact → fetchContacts()
  │     - Each fetch updates its row from ⟳ to ✓ (or ⚠ on failure)
  │
  ├── Wait briefly (e.g., 3-5 seconds) for owner to act on ? rows
  │     - If owner taps [Grant Permission]:
  │         → run permissionService.requestPermission(source)
  │         → on success: row updates to ✓, fetch starts, new ⟳ row added
  │         → on permanent denial: row updates to ? with [Open Settings]
  │         → on cancel: row updates to ⊘
  │     - If owner ignores: continue without that source
  │
  ├── Once all fetches resolved (success, fail, or skip):
  │     - Open SSE: POST /consent-request/{id}/draft-stream with fetched_data
  │     - Add ⟳ Drafting response step
  │     - Stream tokens into draft area
  │     - On done: step becomes ✓
  │
  ├── Owner interacts:
  │     - Edits draft directly → input is just for refinements now
  │     - Types in input + Ask agent:
  │         → cancels any active stream
  │         → POST /draft-stream with {fetched_data, instruction: "<input text>"}
  │         → Add ⟳ Refining: "<instruction>" step
  │         → Stream new tokens REPLACING the draft area
  │         → On done: step becomes ✓
  │         → Clear input box
  │     - Owner can grant a missing permission even after the initial draft:
  │         → On success: re-fetch that source, automatically trigger a new draft stream
  │           (or ask "New data available — re-draft?" non-destructively if owner has edits)
  │
  └── Owner taps "Send to caller":
        → POST /consent-response with draft area text + action="approved"
        → Component routes to channel-specific delivery (Section 5)
```

### 4.9 Summary & Transcript Display

**Summary (the "what is being asked" section):**
- Currently shows raw `informationPrompt` text which is unclear
- Enhanced to show structured layout in the screen header:
  - Channel indicator (icon: phone/WhatsApp + label: "Incoming Call" / "Outgoing Call" / "WhatsApp Message")
  - Requester identity (name + number)
  - Clear statement of what information is requested

**Transcript (the "conversation context" section):**
- Expandable/collapsible section in the header, collapsed by default
- Shows the conversation that led to this consent request
- **Not stored inline** — the transcript already exists in the DB (`transcript_segments` for calls, `whatsapp_messages` for WhatsApp). The `request_payload` only stores a `transcript_id` + `transcript_snapshot_seq` reference. This avoids duplicating potentially large transcript data.
- **Fetched on demand**: when the owner expands the section, the UI calls a backend endpoint that queries `transcript_segments WHERE transcript_id = X AND seq <= snapshot_seq` (for calls) or `whatsapp_messages WHERE conversation_id = X AND created_at <= consent_request.created_at` (for WhatsApp). This returns the transcript up to the point the consent request was created — not turns that happened after.
- For calls: renders as speaker-labeled transcript turns ("Caller: ...", "Agent: ...")
- For WhatsApp: renders as message-style layout (inbound/outbound messages with timestamps)
- Helps the owner understand the full context before deciding what to do

### 4.10 Special Case — OTP-Only

When `data_sources == ["device_otp"]` (and nothing else), the agentic UI is bypassed entirely. There's no draft to generate — the owner just enters the OTP code.

Render: a centered numeric input with auto-fill enabled. Two buttons: "Send to caller" and "Decline". No agent activity panel, no streaming, no refinement.

This is the only special case that bypasses the agentic flow.

### 4.11 Special Case — All Sources Are Missing — Unknown

When every source in `data_sources` is classified as missing — unknown (not in either system's registry):

- The Agent Activity panel briefly shows: `⊘ I don't have access to any of the requested data — please write your response manually`
- The draft area is shown empty and editable from the start
- The "Ask agent" button is disabled (no agent context to ask) — only "Send to caller" and "Decline" are active
- The owner writes the response themselves and sends it

---

## 5. How the Response Flows Back

When the owner approves or declines, the consent component receives:
- `request_id`
- `action` ("approved" or "declined")
- `response` (the text the owner wants to send, if approved)

The consent component then looks up the `channel` from the DB record and routes the response accordingly:

### 5.1 Response Routing by Channel

```
Consent Component receives owner's decision
  |
  |-- Look up consent_request by request_id -> get channel
  |
  |-- IF channel == "inbound_call":
  |     -> Look up call_sid from call_id
  |     -> Check if call session is still live
  |     -> IF live:
  |     |    -> Determine call type (TwiML / duplex / LiveKit)
  |     |    -> Unhold and deliver response through appropriate mechanism
  |     -> IF not live:
  |          -> Mark as expired ("The call has ended. Your response was recorded.")
  |
  |-- IF channel == "outbound_call":
  |     -> Look up call_sid from call_id
  |     -> Check if call session is still live
  |     -> IF live:
  |     |    -> Store response in session as "pending_consent_response"
  |     |    -> The outbound agent picks it up on its next speech turn
  |     |    -> Agent naturally incorporates it: "I have that information for you..."
  |     -> IF not live:
  |          -> Mark as expired
  |
  |-- IF channel == "whatsapp":
  |     -> Look up whatsapp_conversation_id
  |     -> IF approved: send the response as a WhatsApp message to the counterpart
  |     -> IF declined: optionally send a polite decline message
  |     -> No liveness check needed (WhatsApp is async, message always deliverable)
```

### 5.2 Key Behavioral Differences by Channel

| Behavior | Inbound Call | Outbound Call | WhatsApp |
|----------|-------------|---------------|----------|
| **What happens on approve** | Unhold call + play/stream response to caller | Inject into session for next agent turn | Send WhatsApp message |
| **What happens on decline** | Unhold call + play decline message | Inject decline into session | Optionally send decline message |
| **Can the session expire?** | Yes (caller hangs up) | Yes (call ends) | No (WhatsApp is async) |
| **Response delivery mechanism** | TwiML / duplex stream / LiveKit stream | Session flag pickup | WhatsApp API `send_outbound_text` |
| **Timing pressure** | High (caller on hold) | Medium (agent stalling) | Low (async) |

---

## 6. Edge Cases and Boundary Rules

### 6.1 Source Is Missing — Unknown (Not a Known Device Source)

**Scenario**: The LLM suggests `"gmail"`, `"google_calendar"`, `"slack_messages"`, or `"bank_records"`. None of these are in our three known device sources.

**Behavior**:
- The Agent Activity panel does NOT add any step for this source
- The agent has no data to draft from for this source — if it's the only source requested, the draft area is shown empty (Section 4.11)
- If other recognized device sources exist, the agent drafts using just those, and the owner manually adds information for the unknown source

### 6.2 Mixed Known and Unknown Sources

**Scenario**: `data_sources: ["device_calendar", "gmail"]`. Calendar is a known device source, `gmail` is unknown (third-party not yet implemented).

**Behavior**:
- Agent activity panel emits steps only for `device_calendar` (and any required permission steps)
- `gmail` is silently skipped (no step row, no fetch, no error)
- Draft is generated using only device calendar data
- The draft may be partial (the owner can see John's question is about Thursday, agent has the calendar data but not the email context), but the owner can supplement manually via "Ask agent" or by editing the draft directly

### 6.3 Device Source Needs Permission (First Time or Previously Denied)

**Scenario**: `data_sources: ["device_calendar"]`, owner previously denied calendar permission.

**Behavior**:
- `permissionService.checkPermission("calendar")` returns denied
- Agent Activity panel emits: `? Need calendar access to read your schedule [Grant Permission]`
- Owner taps `[Grant Permission]` → native OS permission dialog opens via `permissionService.requestPermission("calendar")`
- If owner allows → row updates to `✓ Granted calendar access`, then continues with `⟳ Reading your device calendar` → `✓` → `⟳ Drafting response`
- If owner denies once → row updates to `⊘ Skipped device calendar (no permission)`
- If OS reports `NEVER_ASK_AGAIN` / `BLOCKED` (previously denied permanently): row updates to `? Calendar denied. Open Settings to enable it. [Open Settings]` — tapping deep-links to the app's settings page

### 6.4 Owner Grants a Permission Mid-Flow After Initial Draft

**Scenario**: Initial draft was generated without device contacts (owner ignored the `?` row). After the draft completes, the owner taps `[Grant Permission]` for contacts.

**Behavior**:
- Native permission dialog runs, owner grants
- Row updates: `✓ Granted contacts access`
- New rows appear: `⟳ Reading your contacts` → `✓`
- Agent automatically triggers a NEW draft stream incorporating the contact data
- New row: `⟳ Updating draft with contact data`
- The draft area's tokens are replaced as the new stream comes in
- **If the owner had already edited the draft manually**: the agent shows a non-destructive prompt: `? You've edited the draft. Update with contact data? [Yes, redraft] [Keep my edits]` — owner chooses

### 6.5 Consent Request After Call/Chat Has Ended

**For calls**: If the owner responds after the call has ended:
- The response is recorded in DB (for audit)
- Status is marked as "expired" with the owner's response preserved
- Owner sees: "The call has ended. Your response has been recorded."

**For WhatsApp**: This doesn't apply. WhatsApp messages can always be sent. Even if hours have passed, the response is delivered.

### 6.6 Multiple Consent Requests in Same Session

**Scenario**: During one call, the caller asks for two different pieces of information.

**Behavior**: Each triggers a separate consent request. Each gets its own DB record, its own notification, and its own response cycle. The consent component treats them independently. The channel agent tracks them via session state.

### 6.7 Owner Asks Agent to Refine the Draft

**Scenario**: Initial draft says "I have a meeting at 3pm Thursday." Owner types "make it more apologetic" in the input box and taps "Ask agent".

**Behavior**:
- Frontend cancels any active stream (if one is running)
- POST `/consent-request/{id}/draft-stream` with `{ fetched_data, instruction: "make it more apologetic" }`
- Agent Activity panel adds: `⟳ Refining: "make it more apologetic"`
- Tokens stream into the draft area, replacing the previous draft
- New draft might read: "Sorry, I have a conflict on Thursday at 3pm — can we try another time?"
- Step row updates to: `✓ Refined response`
- Input box clears, ready for the next instruction
- "Send to caller" remains enabled, owner can either send or ask for another refinement

### 6.8 Owner Asks Agent Multiple Refinements in a Row

**Scenario**: Owner taps "Ask agent" three times in a row with different instructions.

**Behavior**:
- Each instruction creates a new step row in the activity panel
- Each refinement replaces the draft area's content via streaming
- The activity panel becomes a transparent log of how the response evolved:
  ```
  ✓ Read your device calendar
  ✓ Drafted response
  ✓ Refined: "make it more apologetic"
  ✓ Refined: "shorter, one sentence"
  ✓ Refined: "include that I'll call back tonight"
  ```
- The owner can scroll the activity panel to see the history but can't "go back" to a previous version (the draft area always shows the most recent)
- Owner taps "Send to caller" when satisfied

---

## 7. Summary: The Consent Component's Contract

The Consent Request Component is a **channel-agnostic message broker** between channel agents and the device owner:

**Input contract (from any channel):**
- Structured package with: channel, identity, pre-analyzed summary, data sources, missing sources, conversation snapshot
- The channel agent does ALL the intelligence work before calling the component

**Processing:**
- Persist to DB
- Send FCM notification with full metadata
- Return request ID
- Stream the agentic draft response on demand via SSE (`/consent-request/{id}/draft-stream`) when the UI gathers data and asks for a draft

**Output contract (to the UI):**
- Notification carries enough data to render the screen header (channel, summary, requester); full payload fetched on open via `GET /consent-request/{id}/details`
- UI classifies each source against existing systems:
  - Device sources → checked via `permissionService.ts`
  - Unknown sources → silently skipped
- UI runs the agentic consent screen as a state machine:
  - Agent activity panel shows step rows for everything the agent does (or asks the owner to do)
  - For each missing-but-connectable source: a `?` step row with an inline `[Grant Permission]` button
  - Available sources are fetched in parallel and stream into the draft via SSE
  - Owner interacts via single input + two buttons: "Ask agent" (refine) vs "Send to caller" (final)
  - Refinement loop: each "Ask agent" call adds a new step row and re-streams the draft
  - Mid-stream cancel: typing in input or tapping draft area cancels the SSE
- Permission grants trigger automatic re-fetch and re-draft (with non-destructive prompt if owner has manually edited)

**Response routing:**
- Owner's decision comes back with request_id + action + response text
- Component looks up channel from DB
- Routes response through the correct delivery mechanism for that channel
- Channel-specific unhold/delivery/messaging handled in isolated handlers

**What the component does NOT do:**
- Does not analyze conversations
- Does not determine if consent is needed
- Does not figure out data sources
- Does not generate summaries
- Does not fetch data from the owner's device (that's the UI's job)
- Does not know or care about TwiML, LiveKit, WhatsApp APIs (those are in the channel-specific response handlers)

---

## See Also

For the **phased implementation plan** that ships this design in three independently deliverable phases (each keeping the consent flow fully working), see:

[`consent-flow-phased-implementation.md/`](./consent-flow-phased-implementation.md/) — folder containing:
- `00-overview.md` — phasing principles and roadmap
- `phase-1-extract-consent-component.md` — extract the existing inbound flow into a reusable component (~18h)
- `phase-2-multi-channel-support.md` — add outbound call + WhatsApp triggers (~20h)
- `phase-3-agentic-ui-with-streaming.md` — replace screens with the agentic UI (~66h)
