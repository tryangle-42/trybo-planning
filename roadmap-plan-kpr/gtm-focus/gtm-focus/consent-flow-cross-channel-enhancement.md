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

> **Decision reverted (2026-04-27):** an earlier iteration of this design tried to move payload preparation INTO the consent component (the consent component would load the transcript + run the analyzer itself, given only a reference from the channel intelligence). That iteration was reverted after implementation testing — the channel agent already has the conversation in memory and the analyzer's output, so making the consent component re-read everything was wasteful duplication. The original "channel agent prepares everything" model is restored.

**Why this decision:**
- The channel agent already has the conversation in memory (session history, WhatsApp thread). Re-reading it would be wasteful duplication.
- The channel agent's LLM call (`InformationRetrievalAnalyzer`) already determines the summary, data sources, and missing data sources. This analysis is context-dependent — it needs to understand the tone, intent, and flow of the conversation. The consent component is a delivery mechanism, not an intelligence layer.
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

**2. The request payload (built by the channel agent, stored as a single JSONB column - opaque blob, written once, read as a whole):**
```json
{
  "summary":                      "John is asking for your schedule this Thursday",
  "information_retrieval_prompt":  "Retrieve calendar events for Thursday...",
  "requester_name":               "John Smith",
  "requester_number":             "+1234567890",
  "contact_id":                   "uuid-or-null",
  "conversation_context": {
    // call channels only
    "transcript_id":              "uuid-of-existing-transcript",
    "transcript_snapshot_seq":    12,
    // whatsapp channel only
    "last_whatsapp_message_id":   "uuid-of-cutoff-message"
  }
}
```

`conversation_context` carries a **reference** to the conversation transcript (not a copy). The transcript already lives in `transcript_segments` (for calls) or `whatsapp_messages` (for WhatsApp) — storing it again inside `request_payload` would be redundant.

- `transcript_id` + `transcript_snapshot_seq` (call channels) — the transcript record + segment cutoff at trigger time, so the consent screen's transcript view is frozen to what the conversation looked like when consent was triggered.
- `last_whatsapp_message_id` (WhatsApp) — message-id cutoff for the WhatsApp thread, same purpose.
- **No `data_sources` / `missing_data_sources`** — the response drafting agent (Section 4) decides at runtime which sources to call based on the `summary` + `information_retrieval_prompt` and its knowledge of the Data Source Registry. (Per Refinement 2, kept intact through this revert.)

### Why This Split?

The consent_requests table follows a clear principle:

- **Separate columns** for things Postgres needs to enforce or query: FKs (call_id, task_id, whatsapp_conversation_id with referential integrity), channel discriminator (indexed, used in dispatch), lifecycle state (status, timestamps - queried and updated independently)
- **Single JSONB (`request_payload`)** for the opaque request data the channel agent assembles and the UI consumes as a whole. This blob is written once at creation, never queried against, and always read in full when the UI opens.

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

Note: All payload fields (summary, information_retrieval_prompt, requester identity, conversation_context) live inside the `request_payload` JSONB built by the channel agent.

### What the Consent Component DOES (and DOES NOT) Do

**The consent component DOES:**
- Persist the row using the payload built by the channel agent
- Send the owner-facing notification
- Return the request id to the channel agent
- Run the agentic drafting loop on demand
- **Render the conversation context on demand** — given a consent request's channel-specific reference (transcript id + seq for calls, whatsapp_conversation_id + last_message_id for WhatsApp), return the conversation in a render-ready shape when the owner opens the transcript dropdown. The consent component owns this loader; the consent screen calls into it lazily on expand.
- Collect the owner's response and route it back to the originating channel

**The consent component does NOT:**
- Decide whether a consent request should be triggered (that's the channel intelligence's job — only it knows when in the conversation a consent is warranted)
- Pre-classify data sources up front (the response drafting agent decides at runtime which tools to call)
- Fetch data from the owner's device (the UI's tool executors do that on the agent's behalf)
- Speak to the caller or WhatsApp counterpart, ever — every requester-facing message belongs to the channel intelligence

### Responsibility Split (Locked)

The boundary between channel intelligence and consent component is fixed:

| Responsibility | Owner |
|---|---|
| All caller- or counterpart-facing messages and audio (on-hold messages, stalls, failure fallbacks, the final delivered response wording for that channel) | Channel intelligence (voice / WhatsApp / future channels) |
| Build the consent request — run the analyzer on the in-memory conversation, assemble the payload | Channel intelligence |
| Send the consent request to the owner (notification + screen) | Consent component |
| Collect the owner's response | Consent component |
| Route the collected response back to the originating channel | Consent component |
| Take the routed response and deliver it through the channel's transport | Channel intelligence |

The consent component's surface is **owner-facing only**. Anything the requester or counterpart hears or reads is the channel intelligence's job, end to end.

---

## 2. How the Consent Request Component Processes the Request

The Consent Request Component receives the channel agent's pre-built package and does three things:

### Step 1: Persist to Database

Create a `consent_requests` row:
- Relational columns: `channel`, `user_id`, `call_id`, `task_id`, `whatsapp_conversation_id`
- `request_payload`: the JSONB blob built and passed in by the channel agent
- `status: "pending"`

This record is the single source of truth for the consent request lifecycle.

### Step 2: Send Push Notification (FCM)

Send a push notification to the owner's device. The FCM payload is a lightweight projection of `request_payload` plus the `request_id`:
- `request_id` (so the UI can fetch the full record from DB)
- `channel` (so the UI knows what kind of request this is)
- `summary` (extracted from request_payload, for the notification text)
- `requester_name` / `requester_number` (extracted from request_payload, for display)

The notification is channel-agnostic. The FCM payload carries enough for the notification banner. When the UI opens, it fetches the full `request_payload` from DB using `request_id` to get the `conversation_context` (transcript reference for rendering the prior conversation in context).

### Step 3: Return the Trigger Outcome

The trigger is a round trip, not fire-and-forget. The consent component returns one of two outcomes to the channel intelligence:

- **Success — consent created and notified**. The consent row was created and the owner notification was sent. The result includes the `consent_request_id` so the channel intelligence can track it in session state.
- **Failure**. The consent component could not persist the row or could not send the notification. A single flat "failed" verdict — no failure category. The channel intelligence applies the same fallback regardless of which step failed.

**Two-stage caller-facing messaging — no silence on the caller side.**

The channel intelligence does not wait silently while the consent component works. It speaks to the requester twice:

- **Stage 1 — fired the moment the trigger is sent (parallel with the consent component's work)**: an immediate filler such as *"I'm asking the owner about that"*. Channel intelligence owns the wording; the purpose is to break the silence during the consent component's round trip.
- **Stage 2 — fired after the consent component returns its outcome**:
  - On **success**: *"I've sent the request — putting you on hold while we wait for the response"*. The on-hold state begins here, not at Stage 1.
  - On **failure**: *"I'm not able to answer that right now"* or similar fallback the channel intelligence already knows.

There is no caller-side waiting period: Stage 1 covers the consent component's preparation time; Stage 2 reflects the actual outcome.

The consent component never speaks to the caller / WhatsApp counterpart directly. Both Stage 1 and Stage 2 are owned by the channel intelligence. The consent component only prepares, persists, notifies, and reports back.

**Hard rule: the analyzer reads the full transcript from the channel agent's session, not a pre-computed summary.** The channel agent already has the conversation in memory; it passes the full conversation_history into the analyzer when building the payload. No abbreviated summary is fed in.

**Hard rule: no internal retries on the trigger path.** The consent component fails fast on the first error from any step (DB write, FCM send). The requester is on hold during the trigger round trip, so internal retries — even one — extend that wait unacceptably. The channel intelligence's failure fallback is the only retry surface available; the consent component does not absorb flakiness.

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

### 3.2 No Pre-Classification — the Agent Decides at Runtime

The `request_payload` carries NO pre-computed list of data sources. The response drafting agent (Pydantic AI consent agent) reads the Data Source Registry — which already declares the three device sources, their tools, and their permission keys — and decides autonomously which tools to call based on the `summary` and `information_retrieval_prompt`.

Permission state is checked **inline by the tool executor** at the moment the agent calls a tool, not pre-classified up front:

| Outcome | Cause | What the agent sees |
|---|---|---|
| **Tool runs and returns data** | Permission already granted (or just granted via the in-flow OS dialog) | Normal tool result |
| **Tool returns `permission_denied`** | Permission denied OR permanently blocked (`NEVER_ASK_AGAIN`/`BLOCKED`) | Agent drafts around the missing data, may surface a permission grant step row in the UI |
| **Tool not called** | Agent decided this source isn't relevant given the request | No step row, no permission dialog |

This collapses the old upfront classification table — the agent + tool executors do it lazily.

### 3.3 Why No Local Registry in This Component

The Data Source Registry lives in its own module (`information_retrieval_generation/data_source_registry.py`) and is consumed by the consent agent — but the registry itself is consent-agnostic. Other agents (future main-agent in Phase 4, future automation skills) plug into the same registry.

When the future Third-Party Data Sources component lands (Composio: Gmail, Outlook, Slack), it registers into the same registry. The consent agent picks up the new tools automatically — no changes to the consent screen, the SSE endpoints, the prompt's structure, or the channel handlers.

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

### 4.1 Input Mode: The Agent Decides

The UI does NOT branch on a `data_sources` array — that field is no longer in the payload. The agentic consent screen opens by default, and the agent itself decides what to do based on the `summary` + `information_retrieval_prompt`:

```
DEFAULT (every consent request):
  -> Open the AGENTIC CONSENT SCREEN (Sections 4.2–4.8)
  -> Agent reads summary + information_retrieval_prompt
  -> Agent picks tools from the Data Source Registry, executes them on-device
     via the tool bridge (permissions handled inline by the executors)
  -> Agent streams a draft
  -> Owner approves, refines, or declines

SPECIAL CASE — OTP request:
  -> The agent recognizes the request as an OTP fetch from the prompt
  -> The agent calls a `request_otp` tool (same shape as device tools
     like `get_calendar_events`, just with the value provided by the
     owner instead of by a device API)
  -> The frontend renders the tool call as a step row with an inline
     input bubble (numeric keypad + native auto-fill: iOS oneTimeCode,
     Android SMS retrieval) and a Done button
  -> Owner enters the OTP (autofilled or manual) → taps Done → the
     value POSTs back as the tool result → agent resumes
  -> The agent then drafts a normal response embedding the OTP
     (e.g., "Your OTP is XXXXXX") — this is the draft bubble
  -> Owner approves the draft → response is delivered to the requester
     through the normal channel-handler path
  -> If the owner refuses to enter the OTP, the agent drafts a
     "I can't provide this" response which the owner approves and sends
     (see Section 4.10)

SPECIAL CASE — agent has no usable tool for the request:
  -> Agent calls no tools, drafts a "needs your input" message, or surfaces
     a chat message asking the owner to write manually (Section 4.11)
```

The OTP and "no usable source" paths used to be UI-side branches on the (now removed) `data_sources` array. They are now the agent's runtime decisions.

### 4.2 The Agentic Consent Screen

The consent screen behaves like a **chat with an agent** — similar to how you interact with ChatGPT. The owner and the agent have a conversation thread; the agent's goal is to produce a draft response the owner approves. The interaction model:

1. Screen opens → agent starts working (tool calls appear as step rows in the chat)
2. Agent produces a **proposed draft** as a special chat bubble with an **[Approve]** button
3. If the owner approves → the draft is sent to the requester. Done.
4. If the owner wants changes → they type in the input box. The agent re-works the draft and proposes a new one with another [Approve] button.
5. If the owner **asks the agent a question** (not a draft change) → the agent answers the owner in a regular chat message AND keeps the current draft card with [Approve] visible. The draft only changes when the owner explicitly instructs a change.
6. The input box is **disabled while the agent is working** — the owner can't type until the agent finishes its current turn. This keeps the flow clean and prevents race conditions.
7. **Decline** is a separate action in the top-right header — it cancels the entire consent request, not a specific draft.

### 4.2a Dual-Output Interaction Model

The agent has two output modes per turn, determined by the owner's intent:

**Mode A — Draft instruction** (owner wants to change the response):
- Owner says: "also inform I'm available between 8-10 PM"
- Agent output: a NEW draft card with [Approve] (replaces the previous draft)
- Previous draft card loses its [Approve] button (becomes history)

**Mode B — Question to the agent** (owner wants information, not a draft change):
- Owner says: "can you check if I'm available between 8 and 10 PM?"
- Agent output: a regular chat message answering the owner ("You're free between 8-10 PM, no meetings scheduled") + the current draft card with [Approve] **stays as-is**
- The draft is NOT replaced — it stays from the last Mode A turn
- If the owner then wants to incorporate the answer into the draft, they send a Mode A instruction: "add that I'm also available between 8-10"

**How the agent detects the mode:**
The backend agent classifies each owner instruction before acting:
- Contains directive language ("add", "change", "remove", "make it", "also inform", "include") → Mode A (draft change)
- Contains question language ("can you check", "is there", "am I available", "let me know", "what about") → Mode B (question to agent)
- Ambiguous → default to Mode A (safer — always produces a draft)

**SSE event differences:**
- Mode A: emits `tool_call` → `token` (draft text) → `done` with `final_text` (same as today)
- Mode B: emits `tool_call` → `agent_message` (chat text to the owner) → keeps previous `final_text` active

The `agent_message` event is new — the frontend renders it as a regular agent chat bubble (no [Approve] card). The previous draft card stays visible and approvable.

```
┌──────────────────────────────────────────────────────────┐
│  ← Information Consent                        [Decline]  │
│  [Channel badge] John Smith • +1234567890                │
│  John is asking if you're available at 10 PM tomorrow    │
│  ▸ View conversation                                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  CHAT THREAD (scrollable)                                │
│                                                          │
│  ✓ Checking your calendar                                │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ I'm not available at 10 PM tomorrow — I have a     │ │
│  │ meeting scheduled at that time.                     │ │
│  │                                   [Approve & Send]  │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│                   ┌─ You ───────────────────────────┐    │
│                   │ can you check if I'm free       │    │
│                   │ between 8 and 10 PM?            │    │
│                   └─────────────────────────────────┘    │
│                                                          │
│  ✓ Checking your calendar                                │
│  You're free between 8-10 PM, no meetings scheduled.     │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ I'm not available at 10 PM tomorrow — I have a     │ │
│  │ meeting scheduled at that time.                     │ │
│  │                                   [Approve & Send]  │ │
│  └─────────────────────────────────────────────────────┘ │
│  ↑ Draft unchanged — owner asked a question, not a change│
│                                                          │
│                   ┌─ You ───────────────────────────┐    │
│                   │ add that I'm available between   │    │
│                   │ 8-10 PM as an alternative        │    │
│                   └─────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ I'm not available at 10 PM tomorrow due to a       │ │
│  │ meeting. However, I'm free between 8-10 PM if      │ │
│  │ that works for you!                                 │ │
│  │                                   [Approve & Send]  │ │
│  └─────────────────────────────────────────────────────┘ │
│  ↑ Draft updated — owner gave a draft instruction        │
│                                                          │
├──────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────┐  [▶]  │
│  │ Type your instruction to the agent...        │       │
│  └──────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────┘
```

**The screen has these zones, top to bottom:**

1. **Header**: channel badge + requester identity + the original request summary + collapsible transcript. **[Decline]** button in the top-right corner.
2. **Chat thread** (scrollable): alternating agent bubbles and owner bubbles. Agent bubbles contain step rows (tool activity) + the proposed draft text + an **[Approve]** button. Owner bubbles contain the refinement instructions the owner typed.
3. **Input box** (bottom, pinned): single text input + send button. Disabled while the agent is working. Sends the typed text as a refinement instruction. No "Ask agent" / "Send to caller" distinction — the input always talks to the agent; the [Approve] button on the draft bubble is the only way to send the response.

**Key UX rules:**

- The [Approve] button is only on the **latest** agent draft bubble. Previous drafts show the text but no button (they're history).
- Tapping [Approve] sends the draft text in that bubble as the final response to the requester.
- The input box is disabled (grayed out, non-focusable) while the agent is streaming a response. It re-enables when the agent finishes and the [Approve] button appears.
- There is no separate "draft area" — the draft IS a chat bubble. The owner cannot edit the draft text directly; they can only instruct the agent to change it via the input box.
- The [Decline] button in the header declines the entire consent request, not just the current draft. It's always available (even while the agent is working).
- Tool activity step rows (✓ Reading contacts, ✓ Checking calendar, etc.) appear inside the agent's chat bubble, above the draft text, giving the owner visibility into what the agent did.

### 4.3 Step-by-Step Streaming with Results (ChatGPT-style)

Every step the agent takes is visible in the chat as it happens, and every step shows its **result streamed as tokens** — just like ChatGPT renders its thinking and output.

**How it renders:**

```
✓ Checking your calendar
  You have a meeting "Project Review" at 10:00 PM tomorrow.    ← step result, streamed

✓ Drafting response
  ┌─────────────────────────────────────────────────────────┐
  │ I'm not available at 10 PM tomorrow — I have a meeting │
  │ at that time.                        [Approve & Send]   │
  └─────────────────────────────────────────────────────────┘  ← draft output in card
```

**Rules:**
- Each step has a label (`✓ Checking your calendar`) and a **result** (streamed tokens below it)
- Tool steps show what the tool found: "Found 3 contacts matching 'Rishabh'", "You have a meeting at 10 PM", etc.
- The "Drafting response" step's result is the draft text inside the [Approve] card
- If a step fails: `⚠ Couldn't read calendar — permission denied`
- On refinement: a new "Drafting response" step appears with a new [Approve] card

**Multi-step example:**

```
✓ Reading your contacts
  Found: Rishabh Personal (+91-9876543210)

✓ Checking your calendar
  You have a meeting "Team Standup" at 10:00 AM and
  "Project Review" at 10:00 PM tomorrow.

✓ Drafting response
  ┌─────────────────────────────────────────────────────────┐
  │ Hi! Rishabh's number is +91-9876543210. Also, I'm not  │
  │ available at 10 PM tomorrow due to a meeting.           │
  │                                      [Approve & Send]   │
  └─────────────────────────────────────────────────────────┘
```

**SSE events for step results (new):**

Each tool result produces a `step_result` SSE event with streamed tokens:
```
event: step
data: {"step_id": "tc_abc", "status": "completed", "label": "Checking your calendar"}

event: step_result
data: {"step_id": "tc_abc", "token": "You have a meeting"}

event: step_result
data: {"step_id": "tc_abc", "token": " \"Project Review\" at 10 PM tomorrow."}
```

The `step_result` tokens render below the step label, indented, in muted text. This gives the owner visibility into what the agent found before the draft appears.

**Permission handling is inline, not pre-classified**: The agent decides autonomously which tools to call. When a tool needs OS permission, the executor triggers the dialog on the spot. The owner grants or denies; the step result reflects the outcome.

### 4.4 How the Agent Run Works

**No pre-classification phase.** The frontend does NOT read the `data_sources` or `missing_data_sources` arrays from the request payload. These arrays exist for the channel agent's internal use; the consent agent ignores them entirely.

When the screen opens, the agent starts immediately. It reads the `information_retrieval_prompt` and decides autonomously which tools to call. Permissions are handled inline by the tool executors:

- Agent calls `get_calendar_events` → tool executor checks OS permission → if not granted, triggers the OS permission dialog → if owner grants, tool runs → if owner denies, returns `permission_denied` and agent drafts without calendar data
- Same for `search_contacts`, `get_current_location`, etc.

This means the agent works correctly regardless of what `data_sources` contains — even if it's empty, even if the voice service filtered it incorrectly. The agent is self-sufficient.

**Phase B — Agent loop (backend-orchestrated tool use + draft streaming)**

Once classification is done, the frontend opens an SSE stream to the backend agent endpoint. The backend runs a Pydantic AI agent loop:

1. Our code instantiates a Pydantic AI `Agent` with the turn-specific drafting-only system prompt, the device tools (`ExternalToolset`), and the in-process `draft_response` tool
2. We call `agent.iter(user_prompt, message_history=conversation_history)`; Pydantic AI drives the underlying LLM and surfaces `DeferredToolRequests` when the model calls a device tool
3. Backend yields `event: tool_call` over SSE → frontend executes the tool on-device → frontend POSTs the result to `/tool-result`
4. Our code resumes the agent via `agent.iter(message_history=..., deferred_tool_results=DeferredToolResults(calls={id: result}))` so Pydantic AI sees the tool output
5. Repeat 2–4 until the model calls `draft_response` (bounded to `MAX_ITERATIONS = 5`)
6. The `draft_response` return value streams back as Pydantic AI `PartDeltaEvent`s → backend yields `event: token` per delta → draft area fills token-by-token
7. Backend yields `event: done` with `final_text`

Throughout the loop, each `tool_call` / `tool_result` pair produces a step row in the activity panel (`⟳ Reading contacts...` → `✓ Read contacts`). These rows are a faithful projection of the agent's work, not a frontend animation.

**Phase C — Conversational refinement (re-enters the loop)**

After the initial draft completes, the owner can:
- **Tap "Send to caller"** → the current draft text is sent as the final response (Section 5)
- **Tap "Decline"** → the consent request is declined
- **Edit the draft directly** → the input area becomes the draft itself, owner can type/edit anywhere
- **Type an instruction in the input + tap "Ask agent"** → the agent treats the input text as a refinement instruction, NOT as the response

When the owner asks the agent for a refinement, the frontend opens a **new SSE stream** to the same agent endpoint, passing the session-scoped `conversation_history` + the new instruction. The backend re-enters the same tool-use loop — **the agent can call tools again**, now with the corrected context. This is the critical difference from a text-rewrite loop: if the initial search for "John" returned empty and the owner clarifies "the name is Jon", the agent actually re-runs `search_contacts(query="Jon", fuzzy=true)` and finds the contact, instead of just re-phrasing the previous "I don't have that contact" response.

A new step row appears at the start of the refinement:

```
✓ Drafted response
⟳ Refining: "the name is Jon, check again"
  ⟳ Reading contacts (re-querying)...
  ✓ Read contacts
```

The agent streams a new draft into the same draft area, replacing the previous text token-by-token. When done: `✓ Refined response`.

The owner can keep refining as many times as they want. Each refinement becomes a step row in the activity panel — the panel becomes a transparent log of how the response evolved, including any re-queries the agent made along the way.

### 4.5 Chat-Style Input: One Box, One Purpose

The input box at the bottom of the screen has one purpose: **talk to the agent**. There is no "Send to caller" button next to the input. The separation is:

- **Input box** → always sends instructions to the agent (refinement, corrections, new context)
- **[Approve] button** (inside the agent's draft bubble) → sends the draft to the requester
- **[Decline] button** (in the header) → declines the entire consent request

```
┌──────────────────────────────────────────────┐  [▶]
│ Type your instruction to the agent...        │
└──────────────────────────────────────────────┘
```

**Input box states:**

| State | Input box | [▶] send button |
|---|---|---|
| Agent working (tool calls or drafting in progress) | **Disabled** (grayed, non-focusable) | Disabled |
| Agent finished (draft bubble with [Approve] visible) | **Enabled** | Enabled (if text present) |
| Owner typed text + tapped send | Text sent as refinement, input clears, agent starts new turn, input disables again | — |

**Why this design (replacing the two-button model):**
- The previous design had "Ask agent" + "Send to caller" side by side, which created confusion about what each button does. The new design is unambiguous: type = talk to agent; [Approve] on the bubble = send the response.
- Blocking the input while the agent works prevents race conditions (owner typing while the agent is mid-tool-call) and creates a clear turn-taking rhythm — like a real conversation.
- The [Approve] button lives ON the draft, visually tied to the text it will send. The owner sees exactly what they're approving.

### 4.6 Mid-Stream Behavior

Since the input is disabled while the agent is working, there is no "Take over" or "tap to cancel" during streaming. The owner waits for the agent to finish its turn.

If the agent encounters an error or times out (TURN_TOTAL_TIMEOUT=60s), the chat shows an error message and the input re-enables so the owner can retry or type a manual response.

**Decline during agent work**: The [Decline] button in the header is always active, even while the agent is working. Tapping it:
1. Cancels the active SSE + any pending tool calls (same as before)
2. Declines the entire consent request
3. Closes the screen

The DB record is marked `declined`.

### 4.6a What Happens When the Owner Doesn't Approve

If the owner reads the agent's proposed draft and doesn't like it, they have two options:

1. **Refine**: type in the input box ("make it shorter", "the name is Jon not John", "also add my availability tomorrow"). The agent re-enters the loop, potentially re-queries tools, and produces a new draft bubble with a new [Approve] button. The previous draft bubble loses its [Approve] button (becomes history).

2. **Decline the whole request**: tap [Decline] in the header. This doesn't reject just the draft — it declines the consent request entirely.

There is no "reject this draft but keep trying" button. If the owner wants a different draft, they tell the agent what to change. This keeps the interaction model simple: the agent proposes, the owner instructs or approves.

### 4.7 The Backend's Role in the Agentic Flow

**The backend runs the agent.** Moving orchestration to the backend is what makes the agent actually intelligent — it can plan, re-query data, and self-correct across iterations. The frontend's role is reduced to two things: (1) executing on-device tools when the agent requests them, and (2) rendering the step trace + draft stream.

**The agent is implemented using Pydantic AI.** Pydantic AI is chosen for one decisive reason: it ships a first-class primitive (`ExternalToolset` + `DeferredToolRequests` + `DeferredToolResults`) for the exact architecture consent needs — tools declared on the backend but executed off-process (in our case, on the mobile device). The agent pauses when it wants to call a tool, the backend emits the call over SSE, the mobile client runs it, the result gets POSTed back, the agent resumes. Pydantic AI also gives us a provider-agnostic model abstraction (Claude, OpenAI, Gemini, local), Pydantic-validated tool argument schemas (zero hand-rolled validators), message-history rehydration across refinement turns, and token streaming via `PartDeltaEvent` — all of which we would otherwise have to implement ourselves. Our own code is limited to: the SSE bridge, the `tool_call_id` correlation primitive, the MAX_ITERATIONS counter, the drafting-only system prompt, the FastAPI endpoints, and the mobile tool executors.

**The agent is drafting-only.** This is a hard scope boundary. The consent agent's single objective is to produce a response the owner can send to the requester. It uses tools (contacts, calendar, etc.) ONLY when needed to ground the draft in real data, and it always terminates by calling the in-process `draft_response` tool. If the owner's instruction is off-topic ("what's the weather today"), the agent briefly acknowledges and returns to drafting — it does not answer. It does not explain its plan, does not ask clarifying questions, does not produce meta-commentary. This constraint is enforced via the system prompt passed to the Pydantic AI `Agent` on every turn, combined with the explicit `draft_response` terminal tool and the `MAX_ITERATIONS=5` cap in our wiring.

**Why not frontend-orchestrated text rewrite (the rejected design)**

An earlier version of this spec had the frontend classify sources, fetch all available data up-front, then call a single `/draft-stream` endpoint that only re-worded text based on a frozen `fetched_data` blob. That approach cannot satisfy the refinement intelligence requirement: if the initial search for "John" returned empty and the owner clarifies "the name is Jon", a text-rewrite loop can only re-phrase "I don't have that contact", because the data is frozen. A Pydantic AI agent lets the model re-call `search_contacts` with the corrected query and actually find the contact. Accepting the frontend-orchestrated design would have shipped a "dumb agent dressed in streaming UX" — visible streaming on top of a LLM that cannot recover from its own empty results.

**Architecture in layers**

The system is 8 independent layers (each testable in isolation):

| # | Layer | Responsibility |
|---|---|---|
| L1 | Wire / transport | SSE downstream (`tool_call`, `token`, `done`); POST upstream (`tool_result`, cancel) |
| L2 | Agent orchestration (Pydantic AI) | Pydantic AI `Agent` with `ExternalToolset` for deferred device tools + in-process `draft_response`. Our glue forwards `DeferredToolRequests` / `PartDeltaEvent` to SSE and resumes via `DeferredToolResults` |
| L3 | Tool declarations | Pydantic `BaseModel` arg schemas + `ExternalToolset` + `TOOL_LABELS` map |
| L4 | Device-bridge executors | Frontend functions that call native APIs on the agent's behalf |
| L5 | State (ephemeral) | Backend per-request `loop_state`; frontend per-session `conversation_history` |
| L6 | Scope constraint | Drafting-only system prompt passed to the Pydantic AI `Agent` on every turn |
| L7 | UI projection | Step rows + streaming draft + two-button input |
| L8 | Surface integration | Replaces existing consent screens; plumbs final "Send to caller" into Phase 2 handlers |

Dependency chain: L1 → L2 → L3 → (L4 + L6) → L7 → L8. L5 is embedded in L2 and L7. Pydantic AI's provider abstraction removes the need for a separate LLM-provider-abstraction layer. See the phased implementation doc for layer-aligned task breakdown and effort.

**Backend endpoints:**

| Endpoint | Purpose |
|---|---|
| `GET /consent-request/{id}/details` | Loads the full `request_payload` when the screen opens |
| `GET /consent-request/{id}/transcript` | Returns transcript segments up to `transcript_snapshot_seq` for calls, or WhatsApp messages up to `created_at`. Called on demand when the owner expands the transcript section. |
| `POST /consent-request/{id}/agent-turn` | The agent-loop SSE endpoint. Body: `{ instruction?, conversation_history? }`. Streams `tool_call`, `tool_result_ack`, `step`, `token`, `done`, `error` events. `instruction` is absent on the initial draft and present on each refinement. |
| `POST /consent-request/{id}/tool-result` | Side-channel endpoint the frontend POSTs to after executing an on-device tool. Body: `{ tool_call_id, status: "ok" \| "error" \| "permission_denied", result? }`. The backend loop is blocked on an asyncio primitive keyed by `tool_call_id`; this POST unblocks it and feeds the result into the next Claude turn. |
| `DELETE /consent-request/{id}/agent-turn` | Explicit cancel signal to abort an in-progress agent loop |

Permission state for device sources is checked entirely on the device via the existing `permissionService.ts` before the loop starts and on each tool dispatch — no backend endpoint needed for that.

**The tool set (Pydantic AI `ExternalToolset` for deferred tools + one in-process `@agent.tool`):**

| Tool | Arguments | Execution | Returns |
|---|---|---|---|
| `search_contacts` | `{ query: string, fuzzy?: boolean }` | Deferred (on-device) | `[{ name, phone, email, relation? }]` |
| `get_calendar_events` | `{ date_from: ISO, date_to: ISO }` | Deferred (on-device) | `[{ title, start, end, location? }]` |
| `get_messages` | `{ contact_query: string, limit?: number }` | Deferred (on-device) | `[{ direction, text, timestamp }]` |
| `get_current_location` | `{}` | Deferred (on-device) | `{ lat, lon, accuracy_m, label? }` |
| `get_time` | `{ timezone?: string }` | Deferred (on-device) | `{ iso, tz }` |
| `draft_response` | `{ text: string }` | In-process (backend) | terminal — returned string streams back as tokens |

All deferred tools are declared as Pydantic `BaseModel` arg schemas. The 5 deferred tools are routed through `ExternalToolset`; Pydantic AI yields `DeferredToolRequests` for them, which we forward to the frontend. `draft_response` is an in-process Pydantic AI tool — its returned text is what streams back as `PartDeltaEvent` tokens. The set is fixed in Phase 3. New tools land in later phases as the product grows.

**Loop bounds and timeouts:**
- `MAX_ITERATIONS = 5` — after the 5th tool call, Claude is forced to produce a draft with whatever data it has collected (no more `tool_call` events will be honored that turn)
- `TOOL_RESULT_TIMEOUT = 10s` — if the frontend doesn't POST a result for an emitted `tool_call` within 10 seconds, the backend treats it as `{ status: "timeout" }` and continues the loop
- `TURN_TOTAL_TIMEOUT = 60s` — hard ceiling on a single `/agent-turn` request; exceeding it yields `event: error` and ends the stream

**SSE event types from `/agent-turn`:**

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

The `tool_call` event carries the user-facing `label` the frontend should render as the `⟳` step row. The paired `✓` update is emitted as a `step` event once the matching `tool_result` POST comes in.

**Per-request in-memory state:**

The backend maintains a per-request dict keyed by `consent_request_id`:

```python
loop_state = {
    "pending_tool_calls": dict[tool_call_id, asyncio.Event],  # one Event per pending deferred tool
    "tool_results":       dict[tool_call_id, Any],            # attached before Event is set
    "message_history":    list[ModelMessage],                 # Pydantic AI ModelRequest/ModelResponse parts
    "iterations":         int,
    "cancelled":          bool,
}
```

This lives only for the duration of a single `/agent-turn` SSE stream. Nothing persists to Postgres — the session-scoped `conversation_history` that the frontend passes on each refinement is the only across-turn memory, and it stays on the device.

**Freshness guarantee (non-negotiable):** every time the consent screen opens, the agent starts from zero. The backend `loop_state` is created when the SSE opens and released when it closes. The frontend `conversation_history` lives in a `useRef` and is cleared on unmount. Closing and reopening the screen gives a blank agent with no memory of the previous session. This constraint is intentional — the consent component is a trust boundary, not a chat history store. Learning owner preferences across sessions is a future Component 7 (User Memory) concern, out of scope here.

### 4.8 Frontend Orchestration Flow

With the backend running the agent loop, the frontend is a **tool-executor + renderer**. Its state machine is:

```
Screen opens
  │
  ├── Fetch full request_payload via GET /consent-request/{id}/details
  │
  ├── Phase A — Permission classification (no data fetched here):
  │     For each source in conversation_context.data_sources:
  │       - If source ∈ {device_calendar, device_contact, device_otp}:
  │           check permissionService.checkPermission(source)
  │             granted  → mark tool as available (no row yet)
  │             denied   → emit "? Need [source] permission" row
  │             blocked  → emit "? [Source] blocked — open Settings" row
  │       - Otherwise: silent skip
  │
  ├── Phase B — Open agent loop:
  │     POST /consent-request/{id}/agent-turn  (SSE, body: { instruction: null })
  │     │
  │     ├── Receive "event: tool_call" { tool_call_id, name, input, label }:
  │     │     → Add "⟳ <label>" step row
  │     │     → If tool is blocked (permission denied or source unknown):
  │     │         POST /tool-result { tool_call_id, status: "permission_denied" }
  │     │         → step row becomes "⊘ Skipped <label>"
  │     │     → Else: execute on-device
  │     │         - search_contacts → Contacts API
  │     │         - get_calendar_events → Calendar API
  │     │         - get_messages → SMS / WhatsApp local store
  │     │         - get_current_location → Location API
  │     │         - get_time → local Date.now() + IANA tz
  │     │       On success:  POST /tool-result { tool_call_id, status: "ok", result }
  │     │       On failure:  POST /tool-result { tool_call_id, status: "error", error }
  │     │
  │     ├── Receive "event: step" { step_id: tool_call_id, status: "completed" }:
  │     │     → Update row to "✓ <label>"
  │     │
  │     ├── Receive "event: token" { token }:
  │     │     → Append to draftPartsRef, update draft area state
  │     │
  │     └── Receive "event: done" { final_text }:
  │           → Set draft area to final_text
  │           → Enable "Send to caller", "Ask agent" buttons
  │           → Push assistant turn into session-scoped conversation_history
  │
  ├── Owner interacts:
  │     - Edits draft directly → input is just for refinements now
  │     - Types in input + Ask agent:
  │         → Push user turn into conversation_history
  │         → Open NEW SSE: POST /agent-turn
  │             body: { instruction: "<input text>", conversation_history }
  │         → Same tool_call / token / done cycle runs; tokens REPLACE draft area
  │         → Clear input box on done
  │     - Owner grants a previously-denied permission:
  │         → Update "? " row to "✓ Granted <source> access"
  │         → Tool becomes available for the next agent turn
  │           (no automatic re-draft; owner asks the agent to re-try if desired)
  │
  ├── Mid-stream cancel (Take over / tap draft area / type in input):
  │     → Frontend closes SSE
  │     → Frontend sends DELETE /agent-turn
  │     → Any pending tool_call: ignore its result
  │
  └── Owner taps "Send to caller":
        → POST /consent-response with draft area text + action="approved"
        → Component routes to channel-specific delivery (Section 5)
```

The critical simplification compared to the earlier design: the frontend no longer decides when to fetch or when to draft. It only answers Claude's tool calls and renders what the backend emits.

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

### 4.10 Special Case — OTP Request

OTP requests stay inside the same agentic chat-thread layout — no separate screen variant. The agent recognizes the OTP intent from the request prompt and treats it as a **data-extraction step**, parallel to its existing device tools (`get_calendar_events`, `search_contacts`, `get_time`). The only difference is who supplies the value: the owner instead of a device API.

**Flow:**

1. Agent calls a `request_otp` tool. The frontend renders the tool call as a step row with an inline OTP input bubble — numeric keypad + native auto-fill (iOS `oneTimeCode` auto-fill, Android SMS retrieval) + a Done button.
2. Owner provides the OTP — autofilled from SMS by the OS, or typed manually — and taps Done.
3. The OTP value POSTs back as the tool result via the existing deferred-tool bridge. The step row updates to show the captured digits so the owner can verify what was submitted.
4. The agent resumes and drafts a natural response that embeds the OTP (e.g., *"Your OTP is XXXXXX"*). This is the standard drafting step that follows any data extraction.
5. The owner approves the draft. The response is delivered to the requester through the channel-handler path — same as any other consent response.

**If the owner refuses to enter the OTP** (dismisses the input, doesn't autofill, gives no value), the agent treats it as missing data and drafts a polite refusal (*"I can't provide this"* or similar). The owner approves that draft, and the refusal is delivered to the requester.

**Persistence:** the OTP value lives only in the in-memory agent loop for the duration of the screen session — never in DB or logs. The freshness guarantee covers this. The final approved draft (containing the OTP wrapped in natural language) is stored as `delivered_response`, identical to every other consent response.

Selection of the OTP path vs another data extraction path is the agent's runtime decision, not a UI-side branch on any payload field.

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
  |     |    -> Take the call off hold and proactively deliver the response
  |     |    -> (Same model as inbound — no waiting for the caller to speak first)
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
| **What happens on approve** | Unhold call + play/stream response to caller | Unhold call + proactively deliver response (same model as inbound) | Send WhatsApp message |
| **What happens on decline** | Unhold call + play decline message | Inject decline into session | Optionally send decline message |
| **Can the session expire?** | Yes (caller hangs up) | Yes (call ends) | No (WhatsApp is async) |
| **Response delivery mechanism** | TwiML / duplex stream / LiveKit stream | Same as inbound — TwiML / duplex stream / LiveKit stream (proactive on approve) | WhatsApp API `send_outbound_text` |
| **Timing pressure** | High (caller on hold) | High (caller on hold — same as inbound) | Low (async) |

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

### 6.3 Device Source Needs Permission — Inline Grant Flow with Resume

**Scenario**: A tool the agent needs (e.g., calendar) returns `permission_denied` because the source's OS permission was not granted.

**Behavior**:
- The agent does NOT silently draft around the missing data. The drafting step is paused until the permission situation is resolved.
- The step row in the chat thread shows the failure label and a source-specific button — e.g., *[Grant Permission — Calendar]*.
- Tapping the button triggers the per-source / per-OS grant flow. The mechanics vary:
  - **iOS, previously denied**: the OS does not allow the app to re-prompt natively. An app-level popup explains that the owner must go to device Settings and enable the permission manually for that source. When the owner returns to the app, the button has transformed to *[Resume]*.
  - **iOS, first-time**: the standard OS permission dialog runs.
  - **Android**: the standard OS dialog runs; if `BLOCKED` / `NEVER_ASK_AGAIN`, an app-level popup with a deep-link to settings replaces the dialog. Same *[Resume]* affordance applies on return.
  - **Future sources** (e.g., Composio-backed third-parties): bring their own grant flows; the same *[Grant Permission — <source>] → [Resume]* affordance pattern is reused.
- Tapping *[Resume]* re-checks the permission server-side / via permission service:
  - If now granted → the failed tool is re-run and the agent resumes its flow toward drafting
  - If still not granted → the button reverts to *[Grant Permission — <source>]* and the owner can try again or take the manual-response path (see 6.3a)
- The agent never silently drafts around a permission denial — the owner is always offered the explicit chance to grant and get a better answer.

### 6.3a Owner Refuses to Grant — Manual Response Escape Hatch

**Scenario**: The owner does not want to grant the requested permission and wants to write the response themselves.

**Behavior**: a *[Respond manually]* affordance is available in the agentic chat thread (it is universally available — for any reason, at any point in the consent flow). Tapping it opens an editor popup; the owner types or edits their response and confirms (Done). The submitted text becomes a draft bubble in the chat thread — visually and behaviorally identical to a "drafted response" step that the agent would have produced. The standard *[Approve & Send]* sits on this manual draft bubble, so the owner approves it the same way they would an agent draft. Manual response is the universal fallback whenever the agent cannot or should not produce the response itself.

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

### 6.5a Terminal-Status Handling (Two Checkpoints)

**Scenario**: A consent request can reach a terminal status (`expired`, `canceled`, `approved`, `declined`, `delivered`) while the owner has not yet acted on it — most commonly because a live call ended before the owner finished reviewing the draft.

**Behavior**: status is checked at two distinct moments, with different responses at each.

**Checkpoint 1 — at notification tap (before the consent screen renders).**
- If status is `pending`: navigate to the consent screen as designed.
- If status is terminal: do **not** render the consent screen. The owner does not see it. (A lightweight system toast indicating "this request is no longer active" is acceptable, but no full screen.)

The reasoning: if the request is dead by the time the owner taps the notification, there is nothing to act on — opening the screen would mislead them into thinking they could still respond.

**Checkpoint 2 — at Approve & Send tap (after the screen has been open).**
- The drafted-response API re-verifies status server-side on every submission. It does not trust the screen's last-known status.
- If still `pending`: deliver normally through the channel-specific handler.
- If now terminal: do not deliver. The API returns an "expired" outcome and the consent screen renders an **expired bubble inside the chat thread** (same style as agent / owner / draft bubbles), keeping the owner's drafted text visible while disabling action buttons.

The two checkpoints cover two distinct timing windows: cold-open of a long-dead request (Checkpoint 1) and an in-flight expiration during active drafting (Checkpoint 2). One alone is insufficient.

**Status lifecycle is owner-driven, not channel-driven.** Status stays `pending` until the owner acts. The channel intelligence does NOT proactively cancel pending consents when a call ends; there is no TTL sweep; no automatic expiration by elapsed time. The expired transition is computed lazily at Checkpoint 2 — when the owner taps Approve & Send and the consent component finds the originating channel is no longer live, status flips to `expired` at that moment and the screen shows the expired bubble. Implication: under this lazy model, Checkpoint 1 will rarely catch the "channel-ended" case (status is still pending), so cold-open of a stale consent will open the full agentic screen, the agent will draft, and Checkpoint 2 will catch it on Approve. Accepted trade-off: a little wasted compute on rare stale-tap cases, in exchange for no background timers or cancel hooks.

### 6.6b Multi-Device Contention (Not a Concern)

**Scenario**: Owner has the app installed on multiple devices and the consent notification fires on all of them.

**Behavior**: This case does not arise. The app enforces single-device login per user — only one device can be active at a time. The consent screen inherits this guarantee, so there is never more than one open consent screen for the same owner. No claim/release lock or first-write/last-write resolution is needed in the consent flow itself.

### 6.6c Outbound Calls Adopt the Inbound Hold Model

**Decision**: outbound calls drop the existing soft-hold mechanism (the session-flag suppression + reactive injection on the next caller utterance) and adopt the **same hold semantics as inbound calls**:

- **During the wait — transport-specific**: the call is placed on a literal hold at the transport layer. The behavior depends on which transport the outbound call uses:
  - **LiveKit-based outbound** → mirror inbound exactly (literal hold + hold music + full unhold-with-response on approve)
  - **Knowlarity-based outbound** → silent hold is sufficient (the current session-flag-based soft-hold suffices; no transport-level media manipulation needed)
- **On owner approve**: the call is taken off hold and the response is **proactively delivered** by the agent — no waiting for the caller to speak first.
- **The outbound flow is no longer a shape distinct from inbound** at the consent boundary. Both behave identically end-to-end during a consent cycle.

**Why**: the previous reactive-pickup model created a delivery race — if the caller hung up after Approve but before speaking again, the response sat unconsumed and was never delivered, even though the system marked it as delivered. The unification eliminates the race entirely (delivery happens at approve time, not at next-utterance time) and reduces the consent flow to one mental model across all call channels.

### 6.6a Hold-Period Discard (Calls and WhatsApp)

**Scenario**: A consent request is pending. The conversation is on hold (call literally on hold, or WhatsApp conversation in suppressed-input state). The requester keeps speaking on the call, or the counterpart keeps sending WhatsApp messages.

**Behavior**: every channel applies the same hard discard rule during hold.

- **Requester input is dropped on the floor**. Audio / speech from a caller on hold is not transcribed into the transcript record. Inbound WhatsApp messages during the suppressed-input window are not persisted to the database, not added to the conversation history, and not stored in any memory layer. Hold-period input from the requester side is non-existent from the system's point of view.
- **The agent does not respond during hold**. The channel intelligence stays silent on every channel. No filler, no acknowledgment, no clarifying question. The on-hold message that started the hold is the only message the requester receives until the consent resolves.
- **The hold ends** the moment the consent request reaches a terminal status (approved + delivered, declined, expired, canceled). From the next requester input that arrives after the hold ends, normal behavior resumes — but anything from the hold window itself is gone, not retroactively persisted or considered.

**Why this strict discard model**:
- The on-hold state is a clean break: the requester's actions during the wait must not leak into anything afterward (transcript, draft, memory, future agent decisions)
- The owner's consent decision is made against the conversation as it was when consent was triggered — adding hold-period content would change the context the owner approved against
- Eliminates a class of bugs where the WhatsApp intelligence drafts a reply addressing things the counterpart said while waiting, or where the call agent's transcript later contains "are you there?" / "hello?" noise from the hold period
- Symmetric across channels — calls and WhatsApp follow the same rule, so future channels inherit the same semantics

### 6.7 Owner Asks Agent to Refine the Draft (Tone / Style)

**Scenario**: Initial draft says "I have a meeting at 3pm Thursday." Owner types "make it more apologetic" in the input box and taps "Ask agent".

**Behavior**:
- Frontend cancels any active stream (if one is running)
- Frontend pushes `{ role: "user", content: "make it more apologetic" }` onto `conversation_history`
- POST `/consent-request/{id}/agent-turn` with `{ instruction: "make it more apologetic", conversation_history }`
- Agent Activity panel adds: `⟳ Refining: "make it more apologetic"`
- The backend agent loop runs. For a pure-tone refinement, Claude typically skips tool calls and goes straight to `draft_response` — no new `⟳ Reading ...` rows appear.
- Tokens stream into the draft area, replacing the previous draft
- New draft might read: "Sorry, I have a conflict on Thursday at 3pm — can we try another time?"
- Step row updates to: `✓ Refined response`
- Input box clears, ready for the next instruction
- "Send to caller" remains enabled, owner can either send or ask for another refinement

### 6.6d Owner Correction Is Always Authoritative

**Scenario**: the owner refines with input that contradicts the original summary, the original retrieval prompt, or what the agent has just said. (E.g., original summary says "John is asking about Thursday", owner types "actually they meant Tuesday".)

**Behavior**: the agent always defers to the owner's correction. It does not push back, does not negotiate, does not try to reconcile with the original framing, and does not re-run the analyzer. The correction is treated as the new ground truth for the rest of the screen session — the agent re-queries tools and re-drafts based on the corrected context.

The original `request_payload.summary` and `information_retrieval_prompt` in the DB stay unchanged (per the freshness guarantee, no agent session state ever writes back to the consent row). The corrected context lives only in the in-memory conversation history of the current screen session and disappears when the screen unmounts.

The principle: when the agent and the owner disagree, the owner wins. Always.

### 6.7b Owner Asks Agent to Refine Based on Correction (Re-Query)

**Scenario**: Initial request was "give John's number". First draft says "I don't have a contact named John." Owner types "the name is Jon, short for Jonathan" and taps "Ask agent".

**Behavior**:
- Frontend cancels any active stream
- Frontend pushes the instruction onto `conversation_history` and opens a new `/agent-turn` SSE
- The backend agent loop runs. This time Claude recognizes the correction and issues a new `tool_call`: `search_contacts(query="Jon", fuzzy=true)` (or a variant like "Jonathan").
- Activity panel shows:
  ```
  ✓ Drafted response
  ⟳ Refining: "the name is Jon, short for Jonathan"
    ⟳ Reading contacts (re-querying)...
    ✓ Read contacts
  ```
- Frontend receives the `tool_call`, executes on-device, POSTs the result back
- Claude now has the correct contact and calls `draft_response` with a proper reply
- Tokens stream into the draft area replacing "I don't have a contact named John" with e.g. "Jon's number is +1 555-0123"

This is the behavior a text-rewrite refinement loop cannot produce and is the core reason the backend runs the agent.

### 6.8 Owner Asks Agent Multiple Refinements in a Row

**Scenario**: Owner taps "Ask agent" three times in a row with different instructions.

**Behavior**:
- Each instruction creates a new step row in the activity panel
- Each refinement opens a fresh `/agent-turn` SSE and replaces the draft area's content via streaming
- Some refinements re-call tools; others go straight to drafting — the activity panel reflects both kinds:
  ```
  ✓ Read your device calendar
  ✓ Drafted response
  ✓ Refined: "make it more apologetic"
  ✓ Refined: "the name is Jon, short for Jonathan"
    ✓ Read contacts (re-queried)
  ✓ Refined: "include that I'll call back tonight"
  ```
- The owner can scroll the activity panel to see the history but can't "go back" to a previous version (the draft area always shows the most recent)
- Owner taps "Send to caller" when satisfied

---

## 7. Summary: The Consent Component's Contract

The Consent Request Component is a **channel-agnostic, owner-facing intermediary** between channel intelligences and the device owner:

**Input contract (from any channel intelligence):**
- A pre-built package. The channel intelligence runs the analyzer on its in-memory conversation, assembles the `request_payload` JSONB (summary, information_retrieval_prompt, requester identity, conversation_context with the channel-specific transcript reference), and hands the package to the consent component along with the relational identifiers (channel, user, call/task/whatsapp_conversation).
- The consent component does NOT run the analyzer and does NOT assemble the payload — that is the channel intelligence's job.

**Processing:**
- Persist the row using the channel agent's pre-built payload
- Send the owner-facing notification
- Return the trigger outcome (success with request id, or failure) to the channel intelligence as a synchronous round trip
- Run a Pydantic AI agent on demand via SSE (`/consent-request/{id}/agent-turn`) — the agent calls on-device tools through a deferred-tool side-channel POST (`/tool-result`) and streams the final draft back token-by-token

**Output contract (to the UI):**
- Notification carries enough data to render the screen header (channel, summary, requester); full payload fetched on open via `GET /consent-request/{id}/details`
- UI classifies each source against existing systems:
  - Device sources → checked via `permissionService.ts`
  - Unknown sources → silently skipped
  - Classification only determines which tools the agent is allowed to call; no data is pre-fetched
- UI runs the agentic consent screen as a tool-executor + renderer:
  - For each missing-but-connectable source: a `?` step row with an inline `[Grant Permission]` button (frontend-emitted, pre-loop)
  - Backend opens the agent loop; every `tool_call` emitted by Claude produces a step row; frontend executes the tool on-device and POSTs the result
  - Final draft streams token-by-token via `event: token` SSE frames
  - Owner interacts via single input + two buttons: "Ask agent" (refine) vs "Send to caller" (final)
  - Refinement loop: each "Ask agent" call opens a fresh agent-turn SSE with the session-scoped `conversation_history` + new instruction; the agent can re-call tools as needed (e.g., re-query contacts with a corrected name)
  - Mid-stream cancel: typing in input or tapping draft area cancels the SSE + sends `DELETE /agent-turn`
- Permission grants make new tools available to subsequent agent turns; the owner can ask the agent to re-try to pick them up (no automatic re-draft)

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

For the **phased implementation plan** that ships this design in four independently deliverable phases (each keeping the consent flow fully working), see:

[`consent-flow-phased-implementation/`](./consent-flow-phased-implementation/) — folder containing:
- `00-overview.md` — phasing principles and roadmap
- `phase-1-extract-consent-component.md` — extract the existing inbound flow into a reusable component (~18h)
- `phase-2-multi-channel-support.md` — add outbound call + WhatsApp triggers (~20h)
- `phase-3-agentic-ui-with-streaming.md` — agentic consent agent (Pydantic AI) + Data Source Registry (~90h)
- `phase-4-main-agent-drafting.md` — shift response drafting to the main application agent (~26h)
