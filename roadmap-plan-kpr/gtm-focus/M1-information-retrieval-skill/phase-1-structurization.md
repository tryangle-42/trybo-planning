# M1 — Information Retrieval Skill — Phase 1: Structurization

> **Milestone:** M1 — Information Retrieval Skill (Owner: KPR). See `gtm-milestone-plan.md` for the milestone-level demo and beta total.
>
> **This is Phase 1 of two.** Phase 1 *defines the contract* by which any agent in the system communicates with the Information Retrieval Skill — the category registry, the capability bundle, the uniform response envelope, the unified permission flow, the per-source adapter pattern, and the runtime coordination. The actual data-source integrations (Composio third-party + device-level OS permissions + the OAuth/deep-link/RN management surface) are built against this contract in Phase 2 (`phase-2-integration.md`). Defining the contract first means each integration is a mechanical fill against a stable interface, not a bespoke build that has to be retrofitted later. Adding new sources after Phase 2 (Outlook, Slack, Notion, …) is a registry-only change.
>
> **Scope of this document:** the agent ↔ skill communication contract, the per-source adapter pattern, the runtime coordination (LangGraph for the main agent), the device-source `interrupt()` bridge for the main agent, and the **plumbing-reroute plan** for the existing Pydantic AI consent flow (executed in Phase 2 — its agent architecture is preserved, only the data-fetch step is rerouted through skill adapters). Real adapters are stubbed in Phase 1 (canned envelopes for runtime-loop validation); they get filled in with real Composio / OS-permission code in Phase 2. The **only callers** of this skill, now and in the future, are the **Pydantic AI consent-request agent** (drafts replies for inbound consent) and the **LangGraph main chat agent** (responds to the owner directly). Communication / channel agents (call, WhatsApp, etc.) do **not** call this skill directly — when they need user data, they go through one of these two agents.
>
> **Governing principle:** the brain reasons, the skill executes, the runtime coordinates. The skill is a thin pipe — no LLM inside. Every retrieval source — third-party, device, or future internal — exposes the same contract so the brain has one decision tree for every fetch.

---

## 1. Context

### 1.1 The Problem This Phase Solves

Reaching the data sources (the integration work in Phase 2) is necessary but not sufficient. Without a contract for **how an agent talks to them**, every agent ends up re-implementing its own fetch logic, drifts over time, and accrues per-source coupling that makes adding a new agent or a new source painful. Phase 1 (this doc) is the contract that prevents that — defined first so Phase 2's integrations are mechanical fills against a stable shape, not bespoke builds that need to be retrofitted later.

Today's consent flow demonstrates this exact debt:
- Server-side, `information_retrieval_analyzer.py` runs a one-shot LLM call to decide which data sources are needed for a consent.
- The consent screen on RN then **fetches each source itself** with hardcoded code paths — `fetchDeviceCalendar` reads from `react-native-calendar-events`, `fetchGoogleCalendar` calls a backend Google route, `fetchContacts` fuzzy-matches device contacts via Fuse.js.
- A backend `/draft-consent-response` endpoint receives the prompt + collected data and returns a draft.

This is **procedural, not agentic**. There's no agent in the LangGraph sense — no brain↔runtime↔skill loop, no iterative reasoning, no envelope contract. Adding a new source means adding a new hardcoded fetcher in RN, a new backend route, and a new bypass of the analyzer's decisions. Adding a new agent that wants the same data means re-implementing the fetch logic from scratch.

Phase 2 replaces all of that with a **single contract** every agent uses identically.

### 1.2 What "Structurization" Means Here

Five discrete contracts:

1. **Category layer** — a first-class taxonomy of *information categories* (EMAIL, CALENDAR, CONTACTS, OTP, LOCATION, TRANSCRIPTS) that groups multiple data sources serving the same semantic purpose. The brain reasons in categories first ("I need calendar data"), then picks a source within the category based on availability and quality.
2. **Capability bundle** — what every agent loads at run start to know what categories exist, which sources implement each category, and how to ask for them.
3. **Uniform response envelope** — what every retrieval tool returns, regardless of source category, so the brain has one decision tree for handling responses.
4. **Unified permission flow** — a structured `PermissionFlow` shape that captures *both* third-party OAuth (out-of-band: initiate URL + callback URL + return deep link) *and* device-level OS permission (in-band: interrupt + permission key + resume endpoint) under one type-discriminated contract. The agent always sees the same fields shape; the runtime handles the in-band vs out-of-band mechanics.
5. **Per-source adapter pattern** — what every data source implements internally, so adding a source is mechanical and adding an agent is free.

Plus the **runtime coordination**: how LangGraph carries the brain↔skill loop, dispatches tool calls to the right adapter, persists state across `interrupt()` boundaries for device sources, and streams output to each agent's channel.

### 1.3 Two Different Axes — "Origin" vs "Category"

Phase 1 introduces a deliberate separation between two orthogonal axes that previous design conversations conflated:

**Axis A — Origin (where the data physically lives + how we reach it):**

| Origin | Where | Execution surface |
|---|---|---|
| **Third-party** | Provider APIs (Gmail, Google Calendar, Google Contacts) reached via Composio | Server-side: backend → Composio → provider |
| **Device** | The user's phone (OS-permissioned APIs: calendar, contacts, SMS, location) | Client-side: backend pauses run via `interrupt()`, RN executes, run resumes |
| **Internal** | Trybo's own DB (transcripts, summaries) | Server-side: direct Supabase query — *future, post vector DB* |

**Axis B — Information Category (what semantic kind of information the data represents):**

| Category | What it represents | Implementations |
|---|---|---|
| `EMAIL` | Email messages, threads, senders | gmail (third-party) |
| `CALENDAR` | Scheduled events, availability | googlecalendar (third-party), device_calendar (device) |
| `CONTACTS` | People's phone / email / name | googlecontacts (third-party), device_contacts (device) |
| `OTP` | One-time SMS verification codes | device_otp (device) |
| `LOCATION` | The user's current physical location | device_location (device) |
| `TRANSCRIPTS` | Past Trybo conversation history | internal — *future* |

**Why both axes matter:**
- The **agent's brain reasons in categories first** ("the user is asking about a meeting → I need CALENDAR data"), then picks a source within the category based on availability and data quality.
- The **runtime + skill execute by origin** — third-party origins go through Composio, device origins go through `interrupt()`, internal origins go through Supabase. Origin determines mechanics; category determines selection.
- A category like `CALENDAR` spans both third-party (googlecalendar) and device (device_calendar) origins. The brain needs to know "these two sources are interchangeable for calendar information," and that knowledge belongs in the category layer, not buried in adapter prose.

The structural contract treats all origins identically from the brain's perspective at the envelope and permission-flow layer; the implementation differs only inside each per-source adapter.

### 1.4 Two Callers, Two Different Runtimes — Skill Is Runtime-Agnostic

The Information Retrieval Skill has exactly two callers, and they use **different agent frameworks**. The skill is designed runtime-agnostic so each consumes it natively without forcing a framework migration.

| Agent | Framework | How it calls the skill |
|---|---|---|
| **Main chat agent** | LangGraph | Brain emits structured tool calls; LangGraph dispatches via `ToolNode`; envelopes return through the agentic loop; device sources use LangGraph's `interrupt()`. |
| **Consent-request agent** | Pydantic AI | Existing analyzer LLM call decides which data sources are needed; the consent service calls the skill's adapter functions **directly as plain Python**, gets envelopes back, branches on `status`; the existing draft-compose LLM call produces the reply. **No LangGraph in the consent path** in this milestone. |

Both consume the same envelope, the same `PermissionFlow`, the same `CATEGORY_REGISTRY`, the same per-source adapters. The skill's adapters are **plain Python functions returning typed envelopes** — they're framework-agnostic. LangGraph wraps them with `@tool` for the main agent's tool-calling loop. The Pydantic AI consent-request agent calls the same functions directly when its analyzer outputs a list of needed data sources. Adding a third runtime later (or migrating consent-request to LangGraph in a future phase) wouldn't require changes inside the skill.

**Why the consent-request agent stays on Pydantic AI in this milestone:**
- The current consent flow ships and works. Its real pain point is *what data it can fetch* (Google-only, hardcoded RN-side fetchers, no support for new third-party sources) — addressed by routing fetches through the new skill adapters. The agent's framework choice isn't the bottleneck.
- A framework swap (Pydantic AI → LangGraph) for the consent-request would be substantial rewrite cost without a current product gain.
- Pydantic AI is well-suited to the consent-request shape (structured analyzer + draft-compose). Standardizing on LangGraph just for runtime uniformity would be over-engineering.
- A future phase can migrate or re-architect the consent-request agent if iterative drill-down (search → narrow → fetch full body for drafting) becomes a real need. The skill's contract would not change.

**Device-source path on the consent side:** the skill's device adapters return `needs_device_permission` envelopes when called from the consent path; the consent service relays the request to the React Native app via the existing fetch-on-device pattern (today's `informationConsentService.ts` flow); RN performs the OS-permission read; the result is delivered back to the consent service. The LangGraph `interrupt()` mechanism is **not** used in the consent path — that primitive is LangGraph-specific and lives only in the main agent's runtime. The two paths converge on the same envelope shape; their orchestration mechanics differ.

**The skill's caller set is closed at these two.** Communication / channel agents (call agent, WhatsApp agent, and any other future channel agent) do not invoke the Information Retrieval Skill directly. When they need user data they go through the consent-request agent or the main chat agent. The skill's design boundary is owner-context retrieval driven by an agent that has the owner's full conversational state — channel agents operate with channel-bound context (a live call, a WhatsApp thread) and exposing them to retrieval would broaden the trust surface and pull channel-specific concerns into a contract that should stay channel-agnostic.

### 1.5 Hard Constraints (Inherited)

These constraints from earlier design conversations are non-negotiable for Phase 1:

- **Skill is dumb. Agent is smart.** No LLM inside the skill. The brain's planner picks sources, actions, and parameters.
- **All third-party data fetching happens server-side.** RN never calls Composio.
- **Every retrieval is consent-gated.** No agent fetches third-party / device / internal data without explicit user buy-in.
- **The skill never talks to the human.** It talks to the agent. The agent translates skill output into user-channel-appropriate communication.
- **Phase 2 (Integration) depends on Phase 1's contracts, not the other way around.** Phase 1 is contract-only and ships standalone with stub adapters. Phase 2 then provides the real connection state, the `connected_data_sources` dict population on `ExecutionContext`, the `DEVICE_DATA_SOURCES` real wiring, the OAuth + deep-link + in-app browser stack, and replaces the stub adapters with real ones.

---

## 2. Proposed Flow

### 2.1 The Three Actors

| Actor | Role | What it never does |
|---|---|---|
| **Brain** (the LLM call — OpenAI / Gemini / Ollama) | Reads context + history + prior tool results. Produces structured **tool-call intents** (name + args). Decides when to stop. | Never executes code. Never calls Composio. Never reads the DB. |
| **Runtime** (LangGraph) | Loads the capability bundle. Dispatches the brain's tool calls to the matching skill function. Validates args. Captures envelopes. Persists state. Handles `interrupt()` for device sources. Streams output. | Never reasons. Never decides what to fetch. Never embeds business logic. |
| **Skill** (the per-source adapters behind the `@tool` functions) | Executes one round-trip per call: checks availability, fetches data, wraps in envelope. | Never sees natural language. Never knows which agent called it. Never decides retry policy. |

### 2.2 What the Agent Loads at Run Start

Two compact pieces only:

1. **Capability** — what the agent can do. Two facets:
   - A **capability map** (purpose → tool / category) for selection guidance.
   - The **tool schemas** (name + Pydantic params + description) that act as the link to the runtime — when the brain emits a tool call by name, the runtime dispatches by the same name from its registry.

2. **Envelope spec** — the single response shape every tool returns. Branches on `status`.

The brain does **not** load:
- Live connection / permission state. That's discovered at the moment of tool invocation via the envelope's `status` field.
- The full registered tool catalog at process startup (the runtime has it; the brain only sees what's bound for the current run).

### 2.3 The Loop (Same for Every Agent)

```
Run starts.
  Runtime loads InformationRetrievalContext (capability + envelope spec)
  into the brain's prompt. Tools are bound based on Phase 2's registries.

  ┌──────────────────────────────────────────────────────────────────┐
  │  LOOP                                                            │
  │                                                                  │
  │  ▶ Runtime calls Brain with [system + history + tool results]    │
  │                                                                  │
  │  ▶ Brain emits ONE OF:                                           │
  │       (a) final answer → run ends                                │
  │       (b) one or more tool calls (name + structured args)        │
  │                                                                  │
  │  ▶ For each tool call, Runtime:                                  │
  │       1. Looks up the function by name in its registry           │
  │       2. Validates args against the Pydantic schema              │
  │       3. Calls the @tool function (which delegates to its        │
  │          per-source adapter)                                     │
  │       4. Captures the returned RetrievalEnvelope                 │
  │       5. Appends envelope to conversation state                  │
  │       6. Persists state to the checkpointer                      │
  │                                                                  │
  │  ▶ Runtime loops back to Brain with the new tool results visible │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘

Run ends when Brain emits a final answer instead of more tool calls.
Runtime streams the final answer to the agent's channel.
```

### 2.4 Per-Source Adapter — Same Shape, Different Bodies

Inside each `@tool` function, the body delegates to a uniform per-source adapter:

```
DataSourceAdapter {

  check_availability(user_id, state)
    → "available" | "needs_connection" | "needs_permission"
    | "expired" | "not_enabled"

  build_prompt_payload(reason)
    → PromptPayload telling the agent how to ask for access:
      { display_label, scopes_requested, oauth_url?,
        device_permission_key?, instruction_for_agent }

  extract(params, state)
    → executes the actual fetch:
        third-party: Composio tools.execute(action, params, account_id)
        device:      interrupt({...}) → resume → use returned data
        internal:    Supabase query

  wrap_result(raw)
    → maps raw provider data into the typed `data` field of the envelope
}
```

How each category fills these in:

| Function | Third-party (e.g., gmail) | Device (e.g., device_calendar) | Internal (future) |
|---|---|---|---|
| **check_availability** | Read `user_data_source_connections.status` | Ask permission service for OS permission state | Always "available" |
| **build_prompt_payload** | Construct `oauth_url` pointing to Phase 2's connect endpoint | Construct interrupt payload with `device_permission_key` and binding message | N/A |
| **extract** | `composio.tools.execute(action, params, connected_account_id)` | `interrupt(payload)` → wait → resume returns the data | Supabase query |
| **wrap_result** | Map Composio response shape to typed envelope data | Map RN's posted-back result to typed envelope data | Map rows to typed envelope data |

### 2.5 The Five Response Branches the Brain Handles

Every tool returns the uniform envelope. The brain has one decision tree:

```
envelope.status = "ok"
   → use envelope.data → reason on it → continue or finish

envelope.status = "needs_connection"
   → read envelope.prompt_payload
   → compose a connect message in the agent's channel:
       Consent screen → banner with Connect button
       Chat → inline bubble with Connect button
   → emit final answer (this turn ends; user connects out-of-band;
     next run finds the source available)

envelope.status = "needs_device_permission"
   → triggered automatically via interrupt(); RN gets the ask,
     user grants, run resumes inside extract() — brain rarely
     sees this as a final state; it just continues with data

envelope.status = "partial"
   → use envelope.data + envelope.partial info
   → decide: retry the failed slice, proceed with what was fetched,
     or compose a caveated response

envelope.status = "error"
   → read envelope.error
   → if retryable=true: emit the same tool call again
   → if retryable=false: compose a user-facing fallback or ask the
     user to provide info manually
```

### 2.6 Consent-Request Agent — Concrete Walk (Pydantic AI)

```
Trigger: inbound consent arrives. Owner opens consent screen.
         Consent-request flow runs on the existing Pydantic AI orchestration.

Step 1 — Analyzer (Pydantic AI agent, existing):
  Inputs: caller, request, channel, conversation context.
  Output (structured Pydantic model):
    {
      data_sources_needed: ["gmail", "googlecontacts"],
      information_retrieval_prompt: "find recent emails from Sharma
                                     about the deployment",
      ...
    }

Step 2 — Consent service iterates over data_sources_needed:
  For each source_key returned by the analyzer:
    envelope = skill.adapter(source_key).extract(params, state)
    # — plain Python call into the skill's per-source adapter
    # — same adapter that LangGraph @tool-wraps for the main agent
    branch on envelope.status:
      "ok"                          → collect envelope.data
      "needs_connection"            → surface connect banner on consent screen
                                       using envelope.permission_flow
      "needs_device_permission"     → relay to RN via existing on-device fetch
                                       pattern (no LangGraph interrupt here);
                                       RN performs OS read, posts back, service
                                       merges the result into the data set
      "error" (retryable=true)      → retry once
      "error" (retryable=false)     → drop this source from the data set; the
                                       draft will be composed without it

Step 3 — Draft-compose (Pydantic AI agent, existing):
  Inputs: information_retrieval_prompt, collected envelope data.
  Output: structured draft reply text.

Consent screen renders the draft for the owner to review, edit, approve, send.
```

**What's unchanged from today:** the analyzer agent, the draft-compose agent, the order of operations (analyze → fetch → compose), the consent screen UX (banners and modals), the RN-side device fetch pattern, the structured-output shape of the agents.

**What's new:** the per-source fetch step now calls the skill's runtime-agnostic adapters instead of hardcoded `GoogleAccountManager` / `/google/*` routes; the response shape is the uniform envelope; "needs_connection" cases use the skill's `PermissionFlow` to drive the connect banner via Phase 2's connect endpoint.

### 2.7 Main Chat Agent — Concrete Walk

```
Trigger: owner sends a chat message. Main agent run starts.

System prompt: capability map + envelope spec + chat-specific framing
               (recent history, latest message, the agent's job: respond).

Iteration 1:
  Owner asked: "What's on my calendar Thursday?"
  Brain → tool_call(calendar_list_events,
                    {time_range: {from: "2026-04-30T00:00Z",
                                  to:   "2026-05-01T00:00Z"}})
  Runtime → adapter.check_availability → "available"
  Adapter → Composio → Google Calendar → 4 events
  Envelope: {status: "ok", data: EventList[4]}

Iteration 2:
  Brain has enough.
  Brain → final_answer (streamed): "You have 4 events on Thursday: ..."

Run ends. Runtime streams the answer to the chat stream.
```

### 2.8 Edge Path — Source Not Connected (Either Agent)

```
Brain → tool_call(gmail_search_emails, {...})
Runtime → adapter.check_availability(user_id, state)
       → state.connected_data_sources has no "gmail" entry
       → returns "needs_connection"
Adapter skips extract; calls build_prompt_payload("not_connected")
   → returns {display_label: "Gmail", oauth_url: ".../connect?source=gmail",
              scopes_requested: [...], instruction_for_agent: "..."}
Envelope returned: {status: "needs_connection",
                    prompt_payload: {...as above...}}

Brain reads envelope. Composes a channel-appropriate connect message:
   Consent screen → banner: "Connect your Gmail to draft this reply"
   Chat → inline: "I need access to your Gmail. [Connect Gmail]"
Brain → final_answer = the connect message.

Run ends. User taps Connect → Phase 2's OAuth flow runs → row inserted.
On the next run (consent: auto-rerun; chat: user re-asks), the new run
finds gmail in connected_data_sources → happy path.
```

### 2.9 Edge Path — Device Source, Permission Required

```
Brain → tool_call(device_calendar_read, {time_range: {...}})
Runtime → adapter.check_availability → permission service says NOT GRANTED
Adapter.extract → calls interrupt({
    type: "device_permission_request",
    request_id: "req_uuid",
    action: "device_calendar_read",
    scope: ["calendar.read"],
    params: {time_range: {...}},
    binding_message: "Trybo wants to read your calendar to find a free slot"
})

LangGraph PAUSES the run. Checkpoints state to PostgreSQL.
FastAPI surfaces the interrupt to the agent's channel:
   Consent screen → modal popup matching the existing draft Edit popup style
   Chat → modal or inline ask, agent's choice

User grants permission. RN performs the actual OS calendar read. Posts result:
   POST /api/runs/{run_id}/resume
   { request_id: "req_uuid", status: "granted",
     data: [...calendar events...] }

Runtime resumes via Command(resume=<posted result>).
The interrupt() call inside the adapter returns the posted result.
Adapter wraps it and returns envelope: {status: "ok", data: EventList[...]}.

Brain receives the envelope on the next iteration. Continues normally.
```

### 2.10 Migration: Consent Flow Plumbing Reroute (Architecture Preserved)

The consent flow's **agent architecture is preserved**. Only the data-fetch plumbing reroutes through the new skill adapters. The Pydantic AI agents stay; the order of operations stays; the consent screen UX stays.

| What | Today | Phase 2 |
|---|---|---|
| Pydantic AI analyzer agent (decides `data_sources_needed`) | Same | **Unchanged** |
| Pydantic AI draft-compose agent (produces draft text) | Same | **Unchanged** |
| Order: analyze → fetch → compose | Same | **Unchanged** |
| Consent screen UX (banners, draft area, modals) | Same | **Unchanged** |
| Server-side third-party data fetch | `GoogleAccountManager` → `/google/calendar/{userId}` etc. (custom Google routes) | Consent service calls the skill's adapter functions directly: `gmail_search_emails(...)`, `calendar_list_events(...)`. Returns the uniform envelope. |
| RN-side device data fetch | `fetchDeviceCalendar`, `fetchContacts` etc. with ad-hoc code paths | RN-side fetch pattern unchanged in shape; consent service relays a "needs device data" step (same as today) when an adapter returns `needs_device_permission` |
| Connection state read | `user_profiles.settings_jsonb.google` | `user_data_source_connections` table (Phase 2) |
| "Connect Google" banner | Hardcoded for Google, via custom OAuth | Generic "Connect [Provider]" banner driven by `permission_flow.resolution` from any adapter's envelope |
| Adding a new data source | New RN fetcher + new backend route + analyzer prompt change | Add a new adapter + a row in `CATEGORY_REGISTRY` + the slug to the analyzer's allowed list. **No agent framework changes.** |

**This is a plumbing migration, not an architecture migration.** Smaller scope, lower risk. The framework decision (Pydantic AI vs LangGraph for the consent-request) is deferred to a future phase if and when iterative drill-down becomes a product need.

---

## 3. Execution Details

### 3.1 The Capability Bundle (What the Agent Loads at Run Start)

Two pieces, both compact:

**Piece A — Capability map**, rendered into the system prompt. Category-first, with each category listing its implementations:

```
=== INFORMATION RETRIEVAL SKILL ===

You retrieve information by CATEGORY first, then by SOURCE within the category.

Information categories → user's purpose:
  EMAIL          email messages, threads, senders
  CALENDAR       scheduled events, free time, availability
  CONTACTS       a person's phone / email / name
  OTP            one-time SMS verification code
  LOCATION       the user's current location
  TRANSCRIPTS    past Trybo conversation history (future)

Each category has one or more sources:

  EMAIL
    gmail (third-party)                          [preferred]
      gmail_search_emails(query, max_results)    → metadata only
      gmail_get_thread(thread_id)                → full body

  CALENDAR
    googlecalendar (third-party)                 [preferred when connected]
      calendar_list_events(time_range)
      calendar_get_event(event_id)
    device_calendar (device)                     [fallback / always available]
      device_calendar_read(time_range)

  CONTACTS
    googlecontacts (third-party)                 [preferred when connected]
      contacts_search(name?, phone?)
    device_contacts (device)                     [fallback / always available]
      device_contacts_search(query)

  OTP
    device_otp (device)                          [only source]
      device_request_otp(timeout_seconds)

  LOCATION
    device_location (device)                     [only source]
      device_get_location()

Decision rules:
  1. Pick the CATEGORY that matches the user's information need.
  2. Within a category, prefer third-party sources when connected — richer
     data, server-side filtering, fresher state.
  3. If a third-party tool returns needs_connection, you have two choices:
       (a) compose a connect prompt for the user (better long-term UX), or
       (b) fall back to a device source in the same category if data
           quality is acceptable for this turn.
  4. For multi-category needs (e.g., "email Sharma about Thursday's
     meeting"), emit parallel tool calls across categories in one
     assistant turn.
  5. Use the most specific tool. Don't list all events when you have
     an event_id — use calendar_get_event.
```

**Piece B — Envelope spec**, rendered into the system prompt:

```
Every retrieval tool returns this envelope:

  status         "ok" | "needs_connection" | "needs_device_permission"
                 | "partial" | "error"
  source_key     which source produced this
  request_id     for tracing
  data?          present when status="ok" — typed per tool
  prompt_payload? present when status in needs_connection /
                  needs_device_permission — fields you use to compose
                  a channel-appropriate ask
  partial?       present when status="partial"
  error?         present when status="error" — error_type, retryable,
                 user_message, developer_message

Read status first, branch accordingly.
```

### 3.2 Category Registry — First-Class Data Structure

The category layer is not just prose in the capability map; it's a typed registry the skill exposes:

```python
class InformationCategory(BaseModel):
    key:           Literal["EMAIL", "CALENDAR", "CONTACTS",
                           "OTP", "LOCATION", "TRANSCRIPTS"]
    display_label: str            # "Calendar"
    description:   str            # "Scheduled events, free time, availability"
    sources:       list[CategorySource]   # ordered: preferred first

class CategorySource(BaseModel):
    source_key:    str            # "googlecalendar", "device_calendar"
    origin:        Literal["third_party", "device", "internal"]
    preference:    Literal["preferred", "fallback", "only"]
    tool_names:    list[str]      # ["calendar_list_events", "calendar_get_event"]


CATEGORY_REGISTRY: list[InformationCategory] = [
    InformationCategory(
        key="EMAIL",
        display_label="Email",
        description="Email messages, threads, senders",
        sources=[
            CategorySource(source_key="gmail",
                           origin="third_party",
                           preference="only",
                           tool_names=["gmail_search_emails", "gmail_get_thread"]),
        ],
    ),
    InformationCategory(
        key="CALENDAR",
        display_label="Calendar",
        description="Scheduled events, free time, availability",
        sources=[
            CategorySource(source_key="googlecalendar",
                           origin="third_party",
                           preference="preferred",
                           tool_names=["calendar_list_events", "calendar_get_event"]),
            CategorySource(source_key="device_calendar",
                           origin="device",
                           preference="fallback",
                           tool_names=["device_calendar_read"]),
        ],
    ),
    InformationCategory(
        key="CONTACTS",
        display_label="Contacts",
        description="A person's phone, email, or name",
        sources=[
            CategorySource(source_key="googlecontacts",
                           origin="third_party",
                           preference="preferred",
                           tool_names=["contacts_search"]),
            CategorySource(source_key="device_contacts",
                           origin="device",
                           preference="fallback",
                           tool_names=["device_contacts_search"]),
        ],
    ),
    InformationCategory(
        key="OTP",
        display_label="SMS Verification Code",
        description="One-time SMS verification code received on the device",
        sources=[
            CategorySource(source_key="device_otp",
                           origin="device",
                           preference="only",
                           tool_names=["device_request_otp"]),
        ],
    ),
    InformationCategory(
        key="LOCATION",
        display_label="Device Location",
        description="The user's current physical location",
        sources=[
            CategorySource(source_key="device_location",
                           origin="device",
                           preference="only",
                           tool_names=["device_get_location"]),
        ],
    ),
    # InformationCategory(key="TRANSCRIPTS", ...) — future, post vector DB
]
```

This registry is the **single source of truth** for:
- The capability-map rendering injected into each agent's system prompt at run start.
- The category and origin fields populated on every `RetrievalEnvelope`.
- The fallback-source-lookup logic if the agent ever asks the skill "give me any usable source for this category" (a convenience helper, not a primary path).
- New-source onboarding: add a new toolkit by adding a `CategorySource` to the relevant category and registering its `@tool` functions.

### 3.3 The Tool Catalog (≤15 tools at v1)

Same set bound for both consent-request agent and main chat agent. Per-source typed Pydantic args. The tools are grouped by category here for readability; in the runtime they are a flat registered list.

| Category | Tool | Source | Notes |
|---|---|---|---|
| EMAIL | `gmail_search_emails(query, max_results)` | gmail | Lightweight metadata returned |
| EMAIL | `gmail_get_thread(thread_id)` | gmail | Full body |
| CALENDAR | `calendar_list_events(time_range)` | googlecalendar | Events in range |
| CALENDAR | `calendar_get_event(event_id)` | googlecalendar | Single event detail |
| CALENDAR | `device_calendar_read(time_range)` | device_calendar | Via `interrupt()` |
| CONTACTS | `contacts_search(name?, phone?)` | googlecontacts | Find by name or phone |
| CONTACTS | `device_contacts_search(query)` | device_contacts | Via `interrupt()` |
| OTP | `device_request_otp(timeout_seconds)` | device_otp | Via `interrupt()` |
| LOCATION | `device_get_location()` | device_location | Via `interrupt()` |
| TRANSCRIPTS | `internal_transcript_search(query, time_range?)` | internal | Future |

Bind all at once for v1 (Pattern X — ~700 tokens of bound schemas, comfortable). Switch to dynamic binding via `@wrap_model_call` middleware when the catalog exceeds ~30 tools (Pattern Y).

### 3.3 RetrievalEnvelope (Pydantic) — Including Unified Permission Flow

```python
class RetrievalEnvelope(BaseModel):
    status:           Literal["ok", "needs_connection",
                              "needs_device_permission", "partial", "error"]
    source_key:       str           # e.g. "gmail", "device_calendar"
    category:         str           # e.g. "EMAIL", "CALENDAR" — same as the
                                    # capability-map category for this source
    request_id:       str
    data:             Any | None = None              # typed Union per tool
    permission_flow:  PermissionFlow | None = None   # populated when status
                                                     # is needs_connection
                                                     # or needs_device_permission
    partial:          PartialInfo | None = None
    error:            ErrorDetail | None = None
```

The previous loose `PromptPayload` is now a **typed `PermissionFlow`** with a discriminated `resolution` field that captures the full mechanics of each origin's permission flow:

```python
class PermissionFlow(BaseModel):
    """Structured permission flow returned when a source can't be used yet.
    Both third-party (out-of-band OAuth) and device (in-band OS permission)
    flows live under this single typed shape. The agent reads display_label,
    scopes_requested, and instruction_for_agent to compose its channel-
    appropriate ask. The runtime / Phase 2 infrastructure mechanically
    resolves the flow based on the resolution type."""

    flow_type:             Literal["oauth", "device_permission"]
    reason:                Literal["not_connected", "expired",
                                   "permission_not_granted",
                                   "permission_denied"]
    display_label:         str         # "Gmail", "Device Calendar" — for the
                                       # agent's user-facing message
    scopes_requested:      list[str]   # ["read your emails", ...]
    instruction_for_agent: str         # short hint the agent paraphrases
                                       # in its channel
    resolution:            OAuthResolution | DeviceResolution


class OAuthResolution(BaseModel):
    """OUT-OF-BAND OAuth flow handled by Phase 2's connection layer.
    The agent surfaces a connect affordance to the user; OAuth completes
    in the in-app browser; Phase 2's callback writes the connection row;
    the in-app browser dismisses on the return deep link; the run that
    surfaced this flow has already ENDED, and the *next* run for this
    user picks up the now-connected source."""

    flow_type:           Literal["oauth"] = "oauth"

    initiate_endpoint:   str         # POST /api/data-sources/connect
    initiate_payload:    dict        # body to POST: {"source_key": "gmail"}

    # The actual OAuth handshake URL is opened by the agent's channel
    # via the in-app browser; the backend returns it from the initiate
    # call. We do NOT carry the third-party provider's URL in the
    # permission flow — that's an internal mechanic of Phase 2's connect
    # endpoint.

    return_deep_link:    str         # trybo://data-sources/connected?source_key=gmail
                                     # — the in-app browser dismisses on
                                     # this URL; deep-link router refreshes
                                     # connection state in the app.

    callback_url:        str         # the FastAPI callback Composio redirects
                                     # to — for transparency / debugging only;
                                     # the agent does not act on this directly.

    in_band:             Literal[False] = False   # NOT resumed in this run.
                                                  # The current run ends after
                                                  # the agent surfaces the
                                                  # connect prompt.


class DeviceResolution(BaseModel):
    """IN-BAND device permission flow handled by LangGraph's interrupt().
    The runtime has already paused the run and surfaced the request to the
    agent's channel. The agent renders the ask; on user grant, the React
    Native app performs the actual OS permission read and posts back to
    the resume endpoint; the run resumes inside the original adapter call."""

    flow_type:           Literal["device_permission"] = "device_permission"

    permission_key:      str         # "calendar", "contacts", "sms", "location"
                                     # — maps to Phase 2's DEVICE_DATA_SOURCES
    binding_message:     str         # human context: "Trybo wants to read your
                                     # calendar to find a free slot"
    request_id:          str         # correlation id for the resume post

    resume_endpoint:     str         # POST /api/runs/{run_id}/resume
    expires_at:          datetime    # how long the runtime will hold the
                                     # interrupt before timing out

    in_band:             Literal[True] = True    # resumed in this run via
                                                 # Command(resume=...)


class ErrorDetail(BaseModel):
    error_type:        Literal["auth_expired", "rate_limited",
                               "upstream_unavailable", "invalid_params",
                               "not_found", "permission_denied",
                               "device_timeout", "user_denied",
                               "internal_error"]
    retryable:         bool
    user_message:      str        # safe for the consent-screen / chat surface
    developer_message: str        # logs/traces only
    request_id:        str


class PartialInfo(BaseModel):
    fetched:          int
    failed:           int
    failure_reasons:  list[str]
```

**Why a discriminated `resolution` instead of optional fields:**
- The agent always sees the same outer shape (`PermissionFlow` with `display_label`, `scopes_requested`, `instruction_for_agent`).
- The mechanical fields per origin are typed, not optional-mixed-bag. An `OAuthResolution` cannot accidentally have a `permission_key`; a `DeviceResolution` cannot accidentally carry a `return_deep_link`.
- The `flow_type` discriminator lets the runtime branch cleanly on whether to resume in-band or end the run.
- Adding a future origin (say, `BiometricResolution` or `EmailMagicLinkResolution`) is a new variant of the union, not a flag soup.

### 3.4 The Per-Source Adapter Module Layout

```
trybot-api/app/agentic_mvp/skills/information_retrieval/
  ├── __init__.py
  ├── tools.py                    # the @tool functions (entry points)
  ├── envelope.py                 # RetrievalEnvelope + PromptPayload + ErrorDetail
  ├── adapters/
  │     ├── base.py               # DataSourceAdapter ABC: 4 methods
  │     ├── third_party_gmail.py
  │     ├── third_party_googlecalendar.py
  │     ├── third_party_googlecontacts.py
  │     ├── device_calendar.py
  │     ├── device_contacts.py
  │     ├── device_otp.py
  │     ├── device_location.py
  │     └── internal_transcripts.py   # future placeholder
  ├── capability_map.py           # generates the prompt rendering of map + envelope spec
  └── registration.py             # registers tools with the LangGraph runtime
```

Each `@tool` body in `tools.py` is one-liner: instantiate the right adapter, call its four methods, return the envelope. All source-specific logic lives in `adapters/<source>.py`.

### 3.5 Runtime Coordination — Where LangGraph Sits (Main Agent Only)

> **Scope of this section:** the **main chat agent's** runtime. LangGraph is the framework for the main agent. The consent-request agent runs on **Pydantic AI** (existing) and does not use LangGraph; its consumption of the skill is described in §3.7.1 below.

| LangGraph capability | What this phase uses it for |
|---|---|
| Typed state object | Carries `InformationRetrievalContext`, conversation history, tool results, `connected_data_sources` dict |
| `ToolNode` (built-in tool dispatch) | Parses brain's tool-call output, looks up by name in the registry, calls the function |
| Parallel tool execution | Brain can emit multiple tool calls in one assistant turn (e.g., Gmail + Calendar). LangGraph runs them concurrently |
| PostgreSQL checkpointer | Persists state at every node boundary so device-source `interrupt()` pauses can resume after minutes |
| `interrupt()` primitive | Skill's device adapters call `interrupt(payload)` to bridge to RN. State checkpointed; FastAPI surfaces the payload; `Command(resume=value)` returns the value to the adapter |
| Streaming | Final answers stream to chat / consent-screen; custom progress events stream during long fetches via `get_stream_writer` |
| `@wrap_model_call` middleware | Dynamic binding of tool schemas per turn — used post-v1 when the catalog grows |
| Subgraphs | The skill plugs into Trybo's existing outer 8-node lifecycle graph as an inner skill execution surface |
| LLM-agnostic | Works with the existing OpenAI / Gemini / Ollama setup |

### 3.6 ExecutionContext Reads (Phase 2 Provides the Data)

The contract requires `ExecutionContext.connected_data_sources: dict[source_key, account_id]`. Phase 1 defines the field shape and the read pattern; **Phase 2 populates it** at run start from the connection table. Phase 1's stub adapters and Phase 2's real adapters both read this dict in `check_availability` and `extract` — no DB query at the adapter level. The `DEVICE_DATA_SOURCES` registry (concretely wired in Phase 2) is read by device adapters to map `source_key → permission_key`.

### 3.7 Runtime Registration (Once at Process Startup)

```python
# In the FastAPI app's startup hook:

from app.agentic_mvp.skills.information_retrieval.tools import (
    gmail_search_emails, gmail_get_thread,
    calendar_list_events, calendar_get_event,
    contacts_search,
    device_calendar_read, device_contacts_search,
    device_request_otp, device_get_location,
    internal_transcript_search,
)

INFORMATION_RETRIEVAL_TOOLS = [
    gmail_search_emails, gmail_get_thread,
    calendar_list_events, calendar_get_event,
    contacts_search,
    device_calendar_read, device_contacts_search,
    device_request_otp, device_get_location,
    internal_transcript_search,
]

# The main chat agent's LangGraph graph binds these as @tool functions:
main_chat_agent_graph = create_react_agent(
    model=chat_model,
    tools=INFORMATION_RETRIEVAL_TOOLS,
    state_schema=...,
    checkpointer=postgres_checkpointer,
)
```

The same Python functions are imported by the consent-request flow and called directly (without `@tool` wrapping) — see §3.7.1 below.

### 3.7.1 Consent-Request Agent (Pydantic AI) — How It Consumes the Skill

The consent-request agent runs on Pydantic AI, not LangGraph. It consumes the skill by importing the same adapter functions and calling them as plain Python:

```python
# In the consent service, after the Pydantic AI analyzer returns
# data_sources_needed:

from app.skills.information_retrieval.tools import (
    gmail_search_emails, gmail_get_thread,
    calendar_list_events,
    contacts_search,
    device_calendar_read, device_contacts_search,
    device_request_otp,
)

ADAPTER_BY_SOURCE = {
    "gmail":            gmail_search_emails,
    "googlecalendar":   calendar_list_events,
    "googlecontacts":   contacts_search,
    "device_calendar":  device_calendar_read,
    "device_contacts":  device_contacts_search,
    "device_otp":       device_request_otp,
}

collected = {}
for source_key in analyzer_output.data_sources_needed:
    fetch = ADAPTER_BY_SOURCE.get(source_key)
    if not fetch:
        continue
    envelope = fetch(...params built from the analyzer's prompt..., state=...)
    if envelope.status == "ok":
        collected[source_key] = envelope.data
    elif envelope.status == "needs_connection":
        # surface connect banner on consent screen using envelope.permission_flow
        ...
    elif envelope.status == "needs_device_permission":
        # relay to RN via existing on-device fetch pattern (no LangGraph
        # interrupt here); RN performs the OS read, posts back, service
        # injects the result into `collected`
        ...
    elif envelope.status == "error":
        # branch on retryable / error_type
        ...

# Pydantic AI draft-compose agent then runs with `collected` and the
# information_retrieval_prompt to produce the draft text.
```

The skill's adapter functions are framework-agnostic. Same code, two consumers: LangGraph wraps them with `@tool` for the main agent's loop; the consent flow imports them and calls them directly.

### 3.8 Marketplace Composio Adapter Migration

The existing `marketplace_composio.py` (raw `httpx` for Slack/Notion/Linear/GitHub) migrates in Phase 2:
- Replace raw `httpx` calls with `composio_client.tools.execute(...)` via the SDK.
- Read `connected_account_id` from `ExecutionContext.connected_data_sources` dict by toolkit slug (instead of the previous singular field).
- Keep the `_SKILL_TO_COMPOSIO` mapping intact — only the execution method changes.
- Unifies all Composio interactions under one SDK and one credential-lookup convention.

### 3.9 Consent Flow Plumbing Reroute (Phase 2 Scope)

The consent flow's agent architecture is preserved. Only the data-fetch plumbing reroutes through the new skill adapters.

| Today's piece | Phase 2 |
|---|---|
| `information_retrieval_analyzer.py` (Pydantic AI analyzer agent) | **Unchanged.** Continues to output `data_sources_needed`. |
| Pydantic AI draft-compose agent / `/draft-consent-response` endpoint | **Unchanged.** Continues to receive prompt + collected data and produce draft text. |
| `GoogleAccountManager` (RN-side) → `/google/calendar/{userId}` etc. (custom Google routes, backend) | **Replaced.** Server-side consent service iterates over `data_sources_needed` and calls the skill's adapter functions directly. Custom Google routes are removed (Phase 2 backend cleanup). |
| `informationConsentService.ts` device fetches (`fetchDeviceCalendar`, `fetchContacts`) | **Pattern preserved.** When an adapter returns `needs_device_permission`, the consent service relays to RN via the existing on-device fetch flow; RN posts back; service injects result into the collected data. No LangGraph `interrupt()` in the consent path. |
| Hardcoded "Connect Google" banner | **Replaced.** Generic "Connect [Provider]" banner driven by `permission_flow.resolution` from any adapter's envelope. |

The consent screen's UI surface stays. The Pydantic AI agents stay. Only the data plumbing changes.

### 3.10 What Phase 1 Hands Off to Phase 2

Phase 2 cannot ship without Phase 1's contracts in place. Phase 1 produces the following stable surfaces that Phase 2 fills in with real implementations:

- **The `RetrievalEnvelope` shape** (with `category`, `permission_flow`, `partial`, `error`) — Phase 2's adapters return envelopes of this shape.
- **The `PermissionFlow` discriminated union** (`OAuthResolution` and `DeviceResolution`) — Phase 2 populates it with real OAuth endpoints / device permission keys.
- **The `CATEGORY_REGISTRY`** — Phase 2 ensures each `CategorySource` entry maps to a working adapter.
- **The `DataSourceAdapter` base class** with the four-method contract (`check_availability`, `build_prompt_payload`, `extract`, `wrap_result`) — Phase 2's per-source adapters subclass it.
- **The `@tool` registration mechanism** with the LangGraph runtime — Phase 2's adapters wire into this.
- **The `interrupt()` resume endpoint** (`POST /api/runs/{run_id}/resume`) — Phase 2's device adapters use it; React Native posts results to it. **Auth: JWT (the same `get_authenticated_user` dependency used elsewhere)** — the user's own RN app posts the resume payload, so JWT both proves identity and gates access to the run.
- **The capability-map renderer** — Phase 2's category additions automatically flow into the rendered system-prompt section.
- **Stub adapters** that return canned envelopes — Phase 2 replaces them with real Composio calls and device interrupts. The runtime loop is end-to-end validated against the stubs in Phase 1, so Phase 2's risk is contained to per-source integration work, not contract debugging.

### 3.11 What Phase 1 Does NOT Build

Phase 1 is contract-only. It explicitly defers to Phase 2:
- Composio SDK setup, env vars, `composio_client` initialization.
- The `user_data_source_connections` Supabase migration.
- The `ENABLED_DATA_SOURCES` list with real toolkit slugs.
- The `DEVICE_DATA_SOURCES` registry's actual permission-key wiring.
- The OAuth initiate / callback FastAPI endpoints.
- The `trybo://` deep-link scheme registration on iOS / Android.
- The in-app browser dependency (`react-native-inappbrowser-reborn`).
- The ConnectedDataSourcesCard + ConnectedDataSourcesScreen on RN.
- The backend custom-Google-OAuth removal (`google_service.py`, etc.).
- The `marketplace_composio.py` migration to Composio SDK.
- The procedural-to-agentic consent flow migration.

All of the above is Phase 2's scope. Phase 1 ships when the contract is defined, the runtime loop works against stub adapters, the capability bundle renders correctly into agent prompts, and the interrupt-resume mechanism is round-trip-tested end-to-end with a stub device adapter.

### 3.12 Effort Estimates (Phase 1 — Structurization Only)

Phase 1 is contract + runtime + stubs. Real adapters are out of scope here.

| Task | Hours |
|---|---|
| Define `RetrievalEnvelope`, `PermissionFlow`, `OAuthResolution`, `DeviceResolution`, `ErrorDetail`, `PartialInfo` (Pydantic) | 1.5 |
| Define `CATEGORY_REGISTRY` shape (`InformationCategory`, `CategorySource`) and seed with v1 categories | 1 |
| Implement `DataSourceAdapter` base class with the four-method contract | 1 |
| Implement capability-map renderer (system-prompt section generated from registry) | 1 |
| Build stub adapters for every v1 source returning canned envelopes (so the runtime loop runs end-to-end) | 2 |
| Wire `@tool` functions delegating to (stub) adapters; register with LangGraph runtime | 1 |
| FastAPI `/api/runs/{run_id}/resume` endpoint for `interrupt()` resume (JWT-authenticated via `get_authenticated_user`, same as the rest of the user-facing API); round-trip test with a stub device adapter | 2 |
| Consent-request agent: define graph + system prompt; validate run loop with stub envelopes producing a stub draft | 2 |
| Main chat agent: integrate stub tools into existing graph; validate run loop | 1.5 |
| Define plumbing-reroute plan for the existing Pydantic AI consent flow (executed in Phase 2; agent architecture preserved) | 0.5 |
| Unit tests across envelope shape, permission-flow union, category registry, adapter base, runtime registration, interrupt round-trip | 2.5 |
| **Phase 1 Total** | **~16 hours / ~2 days** |

### 3.13 Effort Estimates (Phase 2 — Integration — Reference)

For sequencing context. Detailed in `phase-2-integration.md`.

| Task | Hours |
|---|---|
| Composio SDK setup, env wiring | 0.5 |
| `user_data_source_connections` Supabase migration + `ENABLED_DATA_SOURCES` real list | 1 |
| `DEVICE_DATA_SOURCES` registry wiring with `permissionService.ts` mapping | 1 |
| FastAPI connection endpoints (connect / callback / list / status / disconnect / enabled) | 6.5 |
| Real adapters replacing Phase 1 stubs: gmail, googlecalendar, googlecontacts | 4.5 |
| Real adapters: device_calendar, device_contacts, device_otp, device_location — main agent path uses `interrupt()` → RN; consent agent path uses existing RN device-fetch flow (same envelope shape, two orchestrations) | 6 |
| `marketplace_composio.py` migration to Composio SDK + read from `connected_data_sources` dict | 1.5 |
| Backend Google cleanup (remove `google_service.py`, `google_api_client.py`, `google_user_profile_service.py`, `token_encryption.py` (Google paths), `routes/google.py`) | 4 |
| RN integration: `react-native-inappbrowser-reborn`, `trybo://` scheme on iOS/Android, deep-link router | 4 |
| RN: ConnectedDataSourcesCard + ConnectedDataSourcesScreen + Redux slice + dataSourceService.ts | 5 |
| Consent flow plumbing reroute: replace `GoogleAccountManager`/`/google/*` calls with skill adapter calls; wire `permission_flow`-driven generic connect banner; keep analyzer + draft-compose agents and RN device fetch flow intact | 2 |
| End-to-end device testing (Android + iOS) — connect / disconnect / reconnect / fetch / interrupt + consent flow with new plumbing | 9 |
| **Phase 2 Total** | **~45 hours / ~5.5 days** |

Combined milestone (Phase 1 + Phase 2): **~61 hours / ~7.5 days**.

### 3.14 What This Whole Milestone Explicitly Defers (Phase 1 + Phase 2 Combined)

- **Internal-DB sources** (transcripts, summaries) — adapter slot exists, implementation lands when the vector DB ships.
- **Action catalog scaling beyond v1's ~10 tools** — Pattern Y (dynamic binding via middleware) is the upgrade path; not needed at current scale.
- **Composio webhooks** for proactive connection-status updates — connection state stays lazy (DB row updated on auth-error during fetch).
- **Background pollers** for connection health — same reason.
- **Cross-run result caching** — every fetch is fresh; revisit when a measurable hot path emerges.
- **Multi-account-per-source** — single account per `(user, source)` is locked from Phase 1.

---

## 4. Critical Questions

### 4.1 Why is the Information Retrieval a skill, not a sub-agent?

Sub-agents pay a 15× token premium and add ~800ms latency per fetch (Anthropic's published numbers from their multi-agent research system, validated by independent benchmarks). For a high-frequency, low-individual-value operation like "fetch the user's last 5 emails," that overhead is wasted. A sub-agent with its own LLM would also re-derive in English what the calling agent's planner already produced as structured intent — duplicating work.

LangChain has converged on the same view: their current docs recommend the supervisor pattern via tool-calling, not sub-agent protocols, for use cases like this. Anthropic's own production systems (their multi-agent research system, Claude Code) use sub-agents for breadth-first parallel exploration but tools for actual data retrieval.

### 4.2 Why is the skill "dumb" — no LLM inside it?

The brain (the calling agent's planner) is already running per turn — adding another LLM inside the skill duplicates its reasoning work. A smart skill would either embed an LLM (cost, latency, prompt drift across multiple agent callers) or rely on static templates (kills channel-appropriate UX). The dumb-skill pattern keeps reasoning in one place — the brain — where it belongs, and keeps the skill boundary stable across model swaps and prompt iterations.

### 4.3 Why does the agent decide source + action + params, not the skill?

Because the brain is already producing structured intent. Asking it to translate user language into a Pydantic-typed tool call is exactly what the LLM was trained for; asking it to produce a free-form English handoff that the skill then re-parses is strictly more expensive and strictly less reliable.

The capability map and tool descriptions teach the brain the source-specific syntax (Gmail's `from:` / `subject:` / `after:`, Calendar's `time_range`). That's enough.

### 4.4 Why a uniform envelope across categories?

So the brain has **one** decision tree for response handling — five branches on `status`, regardless of which source the data came from. Without a uniform envelope, the brain would need per-tool error handling, per-tool prompt-rendering, per-tool partial-success logic. The envelope collapses all that into a single contract.

It also makes "add a new source" a self-contained operation: write the adapter, return the envelope, done. No agent code changes.

### 4.5 Why does the agent own channel rendering, not the skill?

The chat agent renders messages as markdown + button bubbles in a streaming chat. The consent-request agent renders responses as a draft on the consent screen, with banners and modal popups for permission asks. Even between just these two callers, the tone, formatting, and affordance shapes differ enough that a single static template can't serve both. The skill cannot know each agent's channel constraints — and shouldn't, because doing so requires either an LLM in the skill (rejected) or static templates (kills personalization).

The agent owning rendering means: skill returns structured fields (`prompt_payload`), agent's planner LLM composes the channel-appropriate message using those fields. The agent's LLM is running anyway; this is essentially free.

### 4.6 Why does the brain see only capability + envelope spec upfront, not live state?

Live state (which sources are connected, which device permissions are granted) goes stale within a run — the user can connect a source between turns. Pre-loading state into the system prompt also pays tokens on every run, even on turns where no source is needed. The envelope's `status` field carries fresh state at the moment of need — exactly when the brain can act on it.

### 4.7 Why bind all tools at run start in v1 instead of dynamic per-turn binding?

At v1's ~10 tools, the bound-schemas footprint is ~700 tokens — comfortable on any modern model. Dynamic binding (LangChain's `@wrap_model_call` middleware filtering tools per turn) is the right answer when the catalog exceeds ~30 tools and the planner's selection quality starts to degrade with too many options.

Phase 1's data structures are designed so the swap from Pattern X (bind all) to Pattern Y (dynamic) is configuration only, not redesign.

### 4.8 Why use LangGraph's `interrupt()` for device sources?

It's the canonical primitive for "tool needs human action, then resume." LangGraph checkpoints state, the run can pause for any duration (seconds to minutes), and `Command(resume=value)` returns the value to the original `interrupt()` call. Auth0's published OAuth-gated tool reference uses exactly this shape; Composio's docs recommend the same pattern.

The alternative — return a `needs_device_permission` envelope and have the brain compose a final answer asking the user, then expect the user to re-trigger the run — works for OAuth (the connection persists across runs) but is wrong for device data (the read needs to happen NOW, in this turn). Interrupt is the right primitive for ephemeral, in-turn device operations.

### 4.9 Why the same skill for both consent-request agent and main chat agent?

If they used different skills, they'd diverge. Two teams iterating on two skills means two evolving fetch surfaces, two sets of error semantics, two sets of source-availability checks. A bug fixed in one wouldn't be fixed in the other.

One skill, two agent configurations — same source-decision logic, channel-specific rendering only — is the cleaner pattern. The caller set is closed at these two; communication / channel agents do not invoke this skill.

### 4.10 Why does the agent runtime execute the tool functions, not the brain?

Brains can't execute code; they're LLM API calls. Even if they could, you wouldn't want them to — you'd lose validation, retries, parallelism, persistence, and structured error handling. The runtime (LangGraph) holds the function references registered at process startup, validates the brain's args against Pydantic schemas, dispatches via direct Python function calls, and persists state at every boundary. The brain's only artifact is structured JSON tool-call intents; everything mechanical is the runtime's job.

### 4.11 Why migrate the existing consent flow in Phase 2 instead of leaving it alone?

Today's consent flow is procedural: server-side analyzer → RN-side hardcoded fetchers → backend draft endpoint. It's hardcoded per source. Adding a new source means adding new RN code, new backend routes, and new analyzer prompt logic — three places to touch per source. It also can't drill down (single-shot fetch, no iterative reasoning).

Migrating to the agentic loop means: adding a new source = one adapter + one `@tool` + one capability-map row. Both agents pick it up automatically. The consent flow gains iterative drill-down (search → get_thread for relevant ones, instead of bulk-fetching everything).

The migration is real work (~5 hours in the estimate above) but pays back the moment a fourth source is added.

### 4.12 Why is the marketplace_composio.py adapter migration in Phase 2, not Phase 1?

Phase 1 is contract-only — it doesn't ship real Composio integration, just the adapter base class and stubs. The marketplace adapter migration (raw `httpx` → Composio SDK + read from `connected_data_sources` dict) requires the real Composio SDK setup to be wired in. That setup lives in Phase 2. Migrating in Phase 1 would mean either including SDK setup in Phase 1 (out of scope — Phase 1 is contract-only), or migrating against a stub SDK and rewriting in Phase 2 — wasted churn either way.

Phase 2 owns the migration in one shot, alongside the new adapters that already use the SDK and the dict. One migration, one mental model.

### 4.13 What's the failure mode if Composio goes down?

All third-party fetches fail with `status=error, error_type=upstream_unavailable, retryable=true`. The brain decides whether to retry next turn or compose a fallback message. Device-source fetches and internal-DB fetches (when shipped) are unaffected — they don't go through Composio. Composio is SOC 2 Type 2 compliant; outages are infrequent and brief.

### 4.14 What's the failure mode if a token expires silently?

The first fetch attempt against the expired source returns `status=error, error_type=auth_expired`. The adapter flips the row's `status` to `expired` in the same call. The brain reads the envelope, treats it as needs-reconnection, composes a reconnect prompt for the user. Next turn after reconnect, the source is `active` and the fetch succeeds.

In Phase 2, `expired` becomes a real state with a writer — fixing the gap noted in Phase 1's design where no code path wrote that status.

### 4.15 What if multiple agents drift on tool usage?

They share the same registered `@tool` functions and the same Pydantic schemas — the call format can't drift. The only place drift can happen is each agent's system prompt (how it interprets the capability map, how it renders failure states for its channel). That's caught by integration tests: end-to-end tests per agent that exercise the connect / disconnect / fetch / interrupt paths.

### 4.16 What about the catalog scaling past 30 tools?

Pattern Y kicks in: `@wrap_model_call` middleware reads the conversation context and binds only the tool schemas relevant to the current turn's intent. The brain sees a small subset; the runtime's registry stays full. This is an upgrade path, not a redesign — Phase 2's data structures already support it.

### 4.17 Why are the two callers (consent-request agent + main chat agent) the closed set, and why don't communication / channel agents call the skill?

The skill's design boundary is **owner-context retrieval driven by an agent that has the owner's full conversational state**. The consent-request agent (drafting on the consent screen) and the main chat agent (responding to the owner directly) both operate against that shape — they hold owner context, they reason over what the owner is asking, they produce output the owner will read or send.

Communication / channel agents (call agent, WhatsApp agent, future channel agents) operate inside a *channel-bound* context — a live phone call with millisecond TTFB budgets, a WhatsApp thread with message-length constraints, etc. Exposing them to the retrieval surface would broaden the trust surface and force the skill's contract to absorb concerns it shouldn't (voice latency budgets, SMS character limits, channel-specific tool subsets). Worse, it would invite per-channel drift in the skill's behaviour over time.

When a channel agent needs user data, it goes through the consent-request agent or the main chat agent — both of which already speak the retrieval contract. **The skill's caller set is intentionally closed at these two and remains closed.**

### 4.18 Why elevate "category" into a first-class data structure instead of leaving it as prose in the capability map?

Two reasons.

First, **fallback semantics need to be machine-readable, not prose**. When the brain decides "I need calendar info" and `googlecalendar` returns `needs_connection`, the runtime / skill / agent should be able to look up "what other sources serve the CALENDAR category?" mechanically. Burying that relationship in capability-map text means relying on the LLM to pattern-match across a paragraph — works most of the time, fails on the edge cases. A typed `CATEGORY_REGISTRY` makes "CALENDAR has [googlecalendar, device_calendar]" a fact the system knows, not a hint the model interprets.

Second, **adding a new source is registry-mechanical instead of prose-editorial**. With prose only, adding (say) `outlookcalendar` to the CALENDAR category means rewriting capability-map text in three places (the section header, the fallback notation, the decision rules). With the registry, it's one new `CategorySource` entry — and every downstream surface (system prompt rendering, fallback lookup, envelope tagging, future dynamic binding) updates automatically.

The category registry is the structural commitment that makes "CALENDAR is one thing with multiple implementations" a property of the system, not a hope.

### 4.19 Why a single `PermissionFlow` shape that covers both OAuth and device permission, instead of two separate fields?

Because the agent's job is identical in both cases: read `display_label`, `scopes_requested`, `instruction_for_agent`, and compose a channel-appropriate ask. The mechanical resolution differs (OAuth runs out-of-band in the next session; device interrupt resumes in the same session) — but the *agent* doesn't need to know that distinction at the message-composition level.

By making `resolution` a discriminated union (`OAuthResolution | DeviceResolution`):
- The agent's prompt-rendering code is one function, branching on `flow_type` only when it needs to (e.g., a chat agent that wants to disable input while a device interrupt is pending).
- The runtime's resolution logic is one function, branching on `resolution.in_band` to decide whether to end the run or keep it paused.
- Adding a future origin (e.g., `BiometricResolution` for native biometric prompts, or `EmailMagicLinkResolution` for passwordless flows) is a new union variant — not a flag soup with `oauth_url?`, `permission_key?`, `magic_link_token?` all optional.

The previous shape (`PromptPayload` with optional `oauth_url` and optional `device_permission_key`) leaked permission mechanics into a flat dict where the agent had to know which combination of optionals went together. The typed `resolution` makes that contract explicit and prevents impossible states (e.g., an OAuth flow accidentally carrying a `permission_key`).

### 4.20 Why include `category` directly on the `RetrievalEnvelope`?

So callers — including the runtime, telemetry, and the agent's own reasoning over multiple envelopes — can reason at the category level without re-resolving "which category does source X belong to?" via a registry lookup. It's denormalized for ergonomic access; the source of truth remains `CATEGORY_REGISTRY`.

### 4.21 What's the dependency graph and shipping order?

```
Phase 1 (Structurization) — defines the contract
  ├── Envelope + PermissionFlow + ErrorDetail + PartialInfo (Pydantic)
  ├── CATEGORY_REGISTRY (typed)
  ├── DataSourceAdapter base class (4-method contract)
  ├── Capability-map renderer (system-prompt section)
  ├── @tool registration with LangGraph runtime
  ├── interrupt() resume endpoint (POST /api/runs/{run_id}/resume)
  ├── Stub adapters (canned envelopes) for every v1 source
  ├── Consent-request agent graph + system prompt (validated against stubs)
  └── Main chat agent integration (validated against stubs)
              │
              ▼
Phase 2 (Integration) — fills the contract with real code
  ├── Composio SDK setup + env wiring
  ├── user_data_source_connections Supabase migration
  ├── ENABLED_DATA_SOURCES + DEVICE_DATA_SOURCES real wiring
  ├── FastAPI connection endpoints (connect / callback / list / status / disconnect / enabled)
  ├── Real adapters replacing stubs:
  │     third-party: gmail, googlecalendar, googlecontacts (Composio)
  │     device: device_calendar, device_contacts, device_otp, device_location (interrupt → RN)
  ├── marketplace_composio.py migration to Composio SDK
  ├── Backend Google cleanup (custom-OAuth Google data-access stack removal)
  ├── RN: react-native-inappbrowser-reborn + trybo:// scheme + deep-link router
  ├── RN: ConnectedDataSourcesCard + screen + dataSourceService.ts + Redux slice
  ├── Procedural-to-agentic consent flow migration (analyzer + draft endpoint + RN fetches removed)
  └── End-to-end device testing (Android + iOS)
```

**Ship Phase 1 first.** Demo: agent run loop runs end-to-end against stub adapters; system-prompt rendering shows the capability map cleanly; the interrupt-resume mechanism round-trips. The contract is real code, just not yet wired to real data sources.

**Phase 2 follows.** Demo: user connects Gmail / Calendar / Contacts via in-app browser; agent says "let me check your Gmail" → real fetches → real draft / chat answer with cited sources; device calendar fetch via `interrupt()` round-trips against a real device. Both phases together fulfil the M1 milestone demo from `gtm-milestone-plan.md`.

**Why this ordering wins:** new sources added after Phase 2 (Outlook, Slack, Notion, Drive, others) are not "integration projects" — they are registry additions plus a new adapter implementation, against a contract that's already proven. The architectural cost of adding the seventh source is the same as adding the fourth.
