# Phase 4 — Shift Response Drafting to Main Application Agent

## Goal

Move the response-drafting intelligence from the consent agent (Pydantic AI, ephemeral, runs per-screen-session) to the **main application agent** (LangGraph orchestrator, persistent, runs per-task). The consent component becomes a UI-only surface that receives drafts from the main agent, presents them for approval, and routes the approved response back.

After Phase 3, the consent agent runs its own Pydantic AI loop on the backend, calls device tools via the SSE/POST bridge, and streams drafts to the mobile screen. Phase 4 replaces that self-contained loop with a skill inside the main agent's orchestration — the consent screen keeps its chat-based UI but no longer drives its own LLM calls.

## Why This Phase

1. **Unified orchestration.** The main agent already handles multi-step tasks, batch execution, retries, and re-planning. Response drafting is just another skill in that framework — keeping it in a separate Pydantic AI loop creates a parallel system to maintain.

2. **Shared context.** The main agent has access to the full conversation history, working memory, and semantic memory layers. The consent agent only sees the `request_payload` snapshot. Drafting quality improves when the drafter sees everything the main agent sees.

3. **Data Source Registry integration.** Phase 3 built the registry with 3 device sources. Future sources (Gmail, Outlook, Slack via Composio) will be server-side, accessible only through the main agent's adapter layer (`DirectAdapter` / `MCPAdapter` / `MarketplaceAdapter`). Running those through the consent agent's device-bridge SSE pattern doesn't make sense — they should run server-side inside the main agent.

4. **Skill reuse.** The main agent's skill registry (`seed_registry.py`) already has 70+ skills. `draft_consent_response` becomes skill #71. The planner can compose it with other skills (e.g., "look up the contact in CRM, then draft the response" — a multi-step plan the consent agent can't do alone).

## What Changes

### Before (Phase 3): Consent agent owns drafting

```
Consent screen opens
  → POST /consent-request/{id}/agent-turn (SSE)
  → Pydantic AI agent runs on backend
  → Agent calls device tools via SSE/POST bridge
  → Agent drafts response
  → Owner approves/refines
  → POST /consent-response
```

### After (Phase 4): Main agent owns drafting, consent screen is a viewer

```
Voice/WhatsApp agent detects information request
  → Creates consent request (same as before)
  → Main agent adds "draft_consent_response" to its plan
  → Main agent calls data sources (device + server-side) via its adapter layer
  → Main agent produces a draft
  → Draft is pushed to the consent screen via Supabase Realtime or FCM
  → Owner sees the draft in the chat UI, approves/refines
  → Refinement instructions go back to the main agent (not a separate Pydantic AI loop)
  → Main agent re-drafts
  → Owner approves → POST /consent-response (same as before)
```

### What stays the same

- The consent request DB schema (Phase 1)
- The channel dispatch for response routing (Phase 2)
- The chat-based consent screen UI (Phase 3) — still shows agent bubbles with [Approve], owner instructions, step rows
- The Data Source Registry (Phase 3) — sources register once, used by both the consent agent (Phase 3 fallback) and the main agent (Phase 4)
- The `/consent-response` endpoint and channel-specific handlers
- FCM notification to the owner's device
- Freshness guarantee: no cross-session memory on the consent screen

### What changes

| Component | Phase 3 | Phase 4 |
|---|---|---|
| **Who runs the LLM** | Consent agent (Pydantic AI, ephemeral) | Main agent (LangGraph, persistent) |
| **Where tools execute** | Device-only via SSE/POST bridge | Device (via bridge) + server-side (via adapters) |
| **Data source access** | 3 device sources from registry | All registered sources (device + DB + API) |
| **Conversation context** | `request_payload` snapshot + session `conversation_history` | Full main-agent conversation buffer + working memory |
| **Refinement path** | New `/agent-turn` SSE per refinement | Main agent re-enters its plan with the refinement instruction |
| **Draft delivery to screen** | SSE from `/agent-turn` endpoint | Supabase Realtime subscription or push event |
| **Consent agent** | Active (runs the loop) | Fallback only (used when main agent is unavailable) |

## Architecture

### New skill: `draft_consent_response`

Registered in `seed_registry.py`:

```python
SkillDefinition(
    name="draft_consent_response",
    description="Draft a response to a consent request using the Data Source Registry",
    approval_required=True,  # Owner must approve before sending
    adapter_type="direct",
    handler="handle_draft_consent_response",
    input_schema={
        "consent_request_id": "string",
        "instruction": "string (optional refinement)",
    },
    output_schema={
        "draft_text": "string",
        "sources_used": "list[string]",
    },
)
```

### Draft delivery: consent screen as a Realtime subscriber

The consent screen subscribes to `consent_requests` row changes via Supabase Realtime. When the main agent produces a draft, it writes it to a new `draft_text` column (or a separate `consent_drafts` table). The screen picks it up and renders it as an agent bubble.

### Refinement: owner → main agent → re-draft

When the owner types a refinement instruction:
1. Frontend POSTs the instruction to a new endpoint (or reuses `/agent-turn`)
2. Backend injects the instruction as a new user message into the main agent's conversation
3. Main agent re-enters its plan, re-queries sources if needed, produces a new draft
4. Draft is pushed to the screen via Realtime
5. Owner approves or refines again

### Device tools in the main agent context

The SSE/POST device-bridge pattern from Phase 3 is reused for device-only sources. The main agent's skill handler emits `tool_call` events the same way the Pydantic AI agent did. Server-side sources (DB contacts, future Gmail, etc.) are called directly via the main agent's adapter layer — no device bridge needed.

### Fallback: consent agent still available

If the main agent is unavailable (e.g., the channel agent didn't create a bot_task, or the call ended before the main agent could plan), the Phase 3 consent agent runs as a fallback. The consent screen checks: is there a main-agent draft coming via Realtime? If not after N seconds, fall back to `/agent-turn` (Phase 3 Pydantic AI loop).

## What Needs to Be Built

| # | Task | Effort | Description |
|---|---|---|---|
| 1 | `draft_consent_response` skill definition + handler | 4h | Register in `seed_registry.py`, implement `DirectAdapter` handler that reads `request_payload`, queries Data Source Registry, calls sources, drafts |
| 2 | Main agent planner awareness | 2h | Update planner prompt to recognize consent requests and plan `draft_consent_response` skill |
| 3 | Draft delivery via Realtime | 3h | Write draft to DB, consent screen subscribes to Realtime for that row |
| 4 | Refinement endpoint | 2h | Accept owner instruction, inject into main agent conversation, trigger re-plan |
| 5 | Server-side data source execution | 3h | Extend Data Source Registry with server-side executors for DB-backed sources (contacts from Supabase) |
| 6 | Fallback logic in consent screen | 2h | Timer-based: if no Realtime draft within 5s, fall back to Phase 3 Pydantic AI loop |
| 7 | Data Source Registry extensions | 3h | Add `executor_type: "device" \| "server" \| "api"` field, server-side executor interface |
| 8 | Integration testing | 4h | Full flow: main agent drafts → screen shows → refine → approve → channel delivery |
| 9 | Buffer | 3h | — |
| **Total** | | **~26h (~3.5 days)** | |

## Dependencies

- Phase 3 must be complete (consent screen UI, Data Source Registry, device bridge)
- Main agent orchestration (M7 execution intelligence) must be stable
- Supabase Realtime must be wired in the mobile app (already used for other features)

## What This Phase Does NOT Touch

- The consent request creation flow (channel agents still create them the same way)
- The `/consent-response` endpoint and channel-specific handlers (Phase 2)
- The DB schema for `consent_requests` (Phase 1)
- The chat-based consent screen layout (Phase 3) — it gains a new data source (Realtime) but the UI stays the same
- The Data Source Registry structure (Phase 3) — Phase 4 extends it with server-side executors, doesn't restructure

## Risk Notes

- **Main agent availability timing:** The main agent runs asynchronously. If the owner opens the consent screen before the main agent has produced a draft, they see a loading state. The 5s fallback to Phase 3 handles this, but tuning the timeout matters for UX.
- **Dual-path complexity:** Having both Phase 3 (Pydantic AI fallback) and Phase 4 (main agent primary) active creates two code paths for the same feature. Plan to deprecate Phase 3's consent agent once Phase 4 is stable — but keep it for at least one release cycle as fallback.
- **Realtime subscription management:** The consent screen must subscribe on mount and unsubscribe on unmount. Stale subscriptions or missed events would leave the screen blank. Use Supabase's built-in retry + the fallback timer as safety nets.
