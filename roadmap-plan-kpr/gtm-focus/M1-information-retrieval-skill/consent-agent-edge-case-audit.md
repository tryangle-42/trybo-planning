# Consent Agent — Edge-Case Audit (backend + UI)

**Date:** 2026-05-14
**Scope:** Cross-channel consent agent (epic TB-618, phases 1-6) — `trybot-api/app/consent_requests/*` + `trybot-ui/{hooks,lib,services,components/consent,components/{ConsentRequestDetailScreen,InformationConsentScreen}}.tsx`.
**Method:** Two parallel deep maps of the consent agent surface across both repos, followed by direct re-reading of the highest-leverage files (`service.py`, `routes.py`, `useConsentAgent.ts`, `AgenticConsentLayout`, `deepLinkRouter`, `dataSourceService`) to verify each finding at the cited line numbers.
**Testing reality:** Local API on `localhost:8000` was not running during this audit. All findings are static-code-verified only. Recipes for dynamic verification are provided where relevant.

**Severity legend:**
- **[CRITICAL]** — data leak, auth bypass, will recur in prod
- **[HIGH]** — user-visible breakage on a common flow
- **[MEDIUM]** — occasional UX cliff or silent failure
- **[LOW]** — defensive / hygiene

---

## 1. [CRITICAL] Half the consent endpoints are unauthenticated

`trybot-api/app/consent_requests/routes.py`:

| Endpoint | Line | Auth |
|---|---|---|
| `GET /consent-request/{id}/details` | 38 | none |
| `POST /consent-request/{id}/agent-turn` | 102 | none |
| `GET /consent-request/{id}/transcript` | 242 | none |
| `POST /draft-consent-response` | 261 | none |
| `POST /consent-response` | 294 | none |
| `POST /consent-request/{id}/tool-result` | 160 | `Depends(get_authenticated_user)` |
| `DELETE /consent-request/{id}/agent-turn` | 231 | authed |

Router is included in `main.py:117` with no global `dependencies=[Depends(...)]`.

**What an attacker who guesses or scrapes a `request_id` can do:**
- `GET …/details` — pull requester name, requester number, summary, `information_retrieval_prompt`, `agent_state` (PII leak).
- `POST …/agent-turn` — burn LLM tokens, fill DB snapshot rows, force device tool round-trips on the legitimate owner's app.
- `POST /consent-response` with `{request_id, action: "approved", response: "<exfiltrated text>"}` — deliver an attacker-controlled response over Twilio / WhatsApp to the original requester as if the owner approved. This is the worst one: `runtime/action_handler.py` routes the payload through the live call session or WhatsApp without re-checking ownership.

**Fix:** every consent route depends on `get_authenticated_user` plus an explicit check that `consent_request.owner_user_id == user.id`.

**Dynamic verification recipe (with API running):**
```bash
curl -i -X POST http://localhost:8000/consent-request/<any-real-id>/agent-turn \
     -H 'Content-Type: application/json' \
     -d '{"user_timezone":"UTC","current_time_utc":""}'
# Expect 401/403. 404 (consent not found) means the route is hit unauthenticated — bug confirmed.
```

---

## 2. [CRITICAL] Frontend SSE is never aborted on screen close — orphan streams keep running

`hooks/useConsentAgent.ts:582-586`:
```ts
useEffect(() => {
  return () => { conversationHistoryRef.current = []; };
}, []);
```
The entire unmount cleanup. It does not call `cancel()`, does not `connectionRef.current?.abort()`, does not DELETE `/agent-turn`.

Neither `ConsentRequestDetailScreen` (l.49-57: bare `onSend(...)`), nor `InformationConsentScreen` (l.65-94), nor `AgenticConsentLayout.handleApprove` (l.249-255: bare `onSend(text.trim())`) call `cancelAgent` before approve / decline / close.

**Production effects:**
- Owner opens consent screen, agent starts streaming, owner taps back. The fetch-based SSE keeps reading until backend's `done` event. Native tool calls that arrive after unmount still execute (contacts read, location read), and `_deliverToolResult` POSTs the result back to a request the owner has already left. Backend continues spending LLM tokens.
- React warns "Can't perform a React state update on an unmounted component" on every `setSteps/setSurfaces/setDraftText` after unmount.
- Multiple opens of the same consent within one session leak multiple parallel SSE streams against the same `request_id` (each subsequent one triggers `_supersede_active_turn` server-side, which only flips `cancelled=True` on the in-memory state — see finding #3).

**Fix:**
```ts
useEffect(() => () => {
  connectionRef.current?.abort();
  connectionRef.current = null;
  if (activeRequestIdRef.current) _sendCancelSignal(activeRequestIdRef.current).catch(() => {});
  conversationHistoryRef.current = [];
}, []);
```

---

## 3. [HIGH] Backend `_active_turns` race — two concurrent `agent-turn`s for the same `request_id`

`agent/service.py:61`: `_active_turns: dict[str, _TurnState] = {}` — module-global, no lock, no `asyncio.Lock`.

`run_agent_turn` (l.92):
```
102  _supersede_active_turn(consent_request_id)   # marks old.cancelled = True, sets events
104  saved = agent_state_store.load_for_resume(...) # awaits I/O — yields event loop
113  state = _build_turn_state(...)
114  _active_turns[consent_request_id] = state
```

Between line 102 and line 114 there is an `await` boundary at line 104. Two requests for the same `request_id` arriving in the same uvicorn process within that gap both see the same prior state, both mark it cancelled, both create new `_TurnState` objects, and the second one's write to `_active_turns` overwrites the first. The first request's loop continues using a `_TurnState` object that is no longer in `_active_turns`:
- `receive_tool_result` (l.64) lookups route to the second state — tool results delivered to the first stream go to the wrong loop.
- Snapshot writes from both loops race on the same DB row → torn `pending_tool_calls` / `message_history`.
- The `finally` block at l.142 (`_active_turns.pop(consent_request_id, None)`) lets either turn evict the dict entry the other is using.

Worse with multiple uvicorn workers: `_active_turns` is per-process. Tool-result delivery in worker A returns `False` if the agent loop is on worker B, falls through to the DB-snapshot resume path (l.181-228), and runs a parallel second loop alongside the first. Both write to `consent_agent_state`.

**Surfaces in prod as:** flaky tool deliveries with `tool_result_no_active_turn` log lines; duplicate/repeated agent surfaces; `done` never firing because two loops are stepping on each other's `pending_events`.

**Fix:** keyed `asyncio.Lock` around supersede + state-install; long-term move state out of the process (Redis or DB row lock) if workers ever scale beyond one.

---

## 4. [HIGH] Single uvicorn worker is mandatory but undocumented

The whole agent loop assumes one event loop owns `_active_turns`. If deploy bumps to `--workers 2` or gunicorn-multi-worker, `cancel_turn` and `receive_tool_result` silently lose state (worker without the state returns `False` → falls into resume path that spawns a second loop on the same row).

Either lock workers to 1 in the deployment manifest with a comment pointing here, or move state out of the process.

---

## 5. [HIGH] Frontend `pending_data_source_connect` deep-link is dead code for cold-start OAuth completion

- `lib/deepLinkRouter.ts:18` `setupDeepLinkRouter()` is called from `App.tsx:164`.
- `services/dataSourceService.ts:13` defines `OAUTH_RETURN_URL = "trybo://data-sources/connected"`.
- **No `registerDeepLinkHandler("data-sources/connected", ...)` is registered anywhere in the repo.** `grep -rn registerDeepLinkHandler` returns only the definition.

This isn't immediately fatal because:
- **In-app browser path** (`dataSourceService.ts:158`): `react-native-inappbrowser-reborn` intercepts the redirect inside its own modal and resolves the promise with `result.type === "success"` — no deep link needed.
- **System-browser fallback** (`dataSourceService.ts:154` `Linking.openURL` when `InAppBrowser.isAvailable() === false`): user returns to the app via deep link → `_dispatch` runs (l.51) → `_handlers.get("data-sources/connected")` is `undefined` (l.64-65) → silent return. Recovery is reactive via the `AppState → active` listener (`useConsentAgent:901-941`) which polls `getStatus`.

**Edge cases this breaks:**
- App was killed while the user was in the system browser. `Linking.getInitialURL()` fires once on cold start (l.24-32) — no handler, link dropped. The `AppState` listener doesn't fire on cold start (the app is starting fresh, not "becoming active"), so the bubble in the snapshot stays stuck in "Waiting for browser…" until the user manually retries or backgrounds-then-foregrounds.
- A user whose `InAppBrowser` is unavailable (older Android customizations) is permanently on the deep-link path that has no handler.

**Fix:** register a handler in `App.tsx` after `setupDeepLinkRouter()` that fires a global event the hook subscribes to, OR check `Linking.getInitialURL()` in `useConsentAgent` mount and short-circuit a status check.

---

## 6. [HIGH] `conversation_history` size cap leaks when assistant never responds

`useConsentAgent.ts:555-558` (user push in `refine`):
```ts
conversationHistoryRef.current.push({ role: "user", content: instruction });
await startAgentTurn(...);
```
No `slice(-20)`. The 20-cap is only applied in the assistant-side branches at l.488-491 and l.511-514.

**Edge case:** user types 30 instructions while the backend is hard-erroring (each turn → `error` SSE event → no append to history). The user-side push runs every time. After 30 refines the array has 30 user turns + 0 assistant turns. Next successful turn ships a 30-entry `conversation_history` to the backend — no server-side length validation either (routes.py:79 takes `Optional[List[ConversationTurn]]`, no `max_length`). Possible token-budget exhaustion / 400 from the model provider.

**Fix:** apply the slice on the user-push branch too, and bound `conversation_history` in `AgentTurnRequest` with `Field(..., max_length=40)`.

---

## 7. [HIGH] No size cap on `conversation_history`, `current_draft_text`, or `instruction` on the wire

`routes.py:72-90` — no `max_length` on any string field. Combined with finding #1, an attacker can ship 5 MB of `current_draft_text`. It gets verbatim-inlined into the LLM user prompt (`service.py: _build_user_prompt`):
- Wasted LLM dollars.
- Possible 4xx from the provider with a generic 500 surfaced to the owner.
- DoS amplification because each request also writes to `consent_agent_state` snapshot.

**Fix:** add `Field(max_length=N)` to all three.

---

## 8. [HIGH] Tool result that arrives at 20.1s is silently dropped, and the UI never learns

`agent/service.py:33` `TOOL_WAIT_HARD_TIMEOUT_S = 20`.

When the device finally POSTs `/tool-result` after the backend has already synthesized a timeout, `receive_tool_result` finds either:
- No active turn → returns `False` → endpoint goes into resume path (l.187), which calls `run_resume_with_tool_result` and starts a fresh agent loop with the late result. May double-fire the tool's effect (e.g. a calendar query the agent already moved past) and emit a confusing second draft.
- An active turn whose `pending_events[tool_call_id]` is gone because the agent advanced past it → returns `False` (l.71) → same as above.

A slow contacts/calendar/location read (20+s, common on first cold native query) silently produces a second agent turn.

**Fix (combinable):**
- Bump `TOOL_WAIT_HARD_TIMEOUT_S` to 30s for contacts/calendar specifically.
- When `receive_tool_result` finds a stale ID and `pending_tool_calls[tcid].result` is already set, return 409 instead of falling into resume.

---

## 9. [HIGH] `consentAgentTools.ts` — `permission_denied` without `permissionKey` short-circuits the grant flow

`useConsentAgent.ts:371-383` only surfaces the grant bubble when `permission_denied && permissionKey`. If the dispatcher ever returns `{ status: "permission_denied" }` without `permissionKey` (e.g. iOS native module returns `unauthorized` for calendar but the JS layer forgets to thread through the `permissionKey`), the result flows straight to `_deliverToolResult` and the agent sees "permission denied", drafts around the missing data, and the owner is never asked to grant the permission. Exactly the regression that ships when a new tool is added without updating the structured-denial contract.

**Fix:** unit test that for every entry in `TOOL_SOURCE_LABEL` (l.999-1004), `executeDeviceTool(name, {}, undefined)` while permission is denied returns a `permissionKey`.

---

## 10. [HIGH] AppState foreground recovery clears the bubble before SSE re-emits

`useConsentAgent.ts:944-960`:
```ts
setError(null);
setPendingOtpRequest(null);
setPendingPermissionGrant(null);
startAgentTurn(reqId).catch(() => {});
```
The recovery clears `pendingOtpRequest` synchronously and assumes the backend will re-emit a `tool_call` event when the new turn replays from the snapshot. That assumption holds only if:
- The snapshot exists and has the pending tool call descriptor.
- `_is_no_op_resume` doesn't short-circuit (it doesn't, since pending tools exist).
- The new turn's `_run_loop` re-emits a `tool_call` for the same `tool_call_id`.

If snapshot persistence failed earlier (`agent_state_store.py:81-111` silent two-attempt retry, then swallow), the snapshot won't have the pending tool, and the OTP/permission bubble is gone with no recovery. Owner sees an empty consent screen and `isStreaming=true` forever (set at l.166) — `canExit` is `false` (l.963-968) and the back button is hidden.

**Hard lock-in.** The only escape is `Decline` (intentionally always enabled) or force-quitting the app.

**Fix:** don't clear pending bubbles until the new SSE actually emits the matching `tool_call`. Or subscribe to SSE `error` and re-show the bubble.

---

## 11. [HIGH] Approve while live agent_message is streaming, then `_deliverToolResult` after unmount

Timeline:
1. Owner is on the screen, agent runs a tool, agent is mid-`agent_message`.
2. Owner taps Approve on a previous frozen draft (allowed — Approve is per-bubble at `AgenticConsentLayout.tsx:347`).
3. `onSend` runs, parent navigates away, screen unmounts.
4. SSE keeps streaming (finding #2), eventually emits `tool_call` for the next tool. `_handleEvent` (still pinned in the closure) calls `executeDeviceTool` then `_deliverToolResult`.
5. POST goes through → backend resumes → keeps generating → DB snapshot keeps updating.

The data that's POSTed (e.g. a contacts search) gets done after the owner already shipped a response. Not security-critical, but billing-relevant and confusing to debug.

Same fix as #2.

---

## 12. [MEDIUM] Backend pending-retry leak on abandoned OAuth

`agent/owner_tools.py` keeps a module-level `_PENDING_RETRY` dict mapped by `(consent_request_id, source_key)`. Populated when a server tool returns `needs_connection`. Cleared when the auto-retry executes or the snapshot is removed. If the user starts OAuth, closes the in-app browser, never retries, and abandons the consent — the entry stays in the dict for the lifetime of the process. Each connect attempt adds a new entry, none cleared.

Slow leak. With a busy server it grows.

**Fix:** TTL eviction (e.g. on each `_supersede_active_turn`, evict entries older than 1 hour).

---

## 13. [MEDIUM] Inbound call — live-session check is a TOCTOU race

`runtime/action_handler.py:331` checks `is_call_live(call_sid)` in Redis before pushing TwiML. The check and the push aren't atomic; a call can hang up between them. The TwiML push then targets a dead call. Twilio quietly drops it; the consent state still flips to `approved`, so the owner thinks it was delivered.

**Fix (for exactly-once delivery semantics):** Twilio webhook on call-completed that reconciles `approved-but-not-delivered` requests.

---

## 14. [MEDIUM] Outbound call — `pending_consent_response` injected into Redis session, may target an evicted session

`_handle_outbound_call_consent` injects into the session key `pending_consent_response` (action_handler.py:465). If the outbound caller hung up and the session was evicted (Redis TTL), the response goes into a fresh/no-op key and gets garbage-collected. Owner sees the consent flip to `approved`; receiver gets nothing.

Reconciliation same as #13.

---

## 15. [MEDIUM] WhatsApp channel — no delivery confirmation, no retry

`_handle_whatsapp_consent` (action_handler.py:535-) fires WhatsApp send-message API once and updates the DB. If the WhatsApp Business API returns 5xx or rate-limits, the consent is still marked `approved` and the response is lost. No queueing.

---

## 16. [MEDIUM] `submitOtp` swallows delivery failure — OTP disappears

`useConsentAgent.ts:720-748`:
```ts
setPendingOtpRequest(null);  // clears the bubble synchronously
await _deliverToolResult(...);  // if this throws, we already cleared the bubble
```
On network failure, the bubble disappears, the agent keeps waiting for a tool result that never comes, and the backend will synthesize a timeout 20s later. Owner has no way to retry without refreshing the whole screen.

**Fix:** reorder — deliver first, clear on success.

---

## 17. [MEDIUM] Mode-prefix detection has a 500-char silent fallback that mislabels the bubble

`service.py:712-713`: if the agent emits 500 characters without ever writing `REASONING:` / `DRAFT:` / `ANSWER:`, the parser defaults to "draft". On a long thinking preamble (rare but possible with a less-instruction-following model), the buffered 500 chars are flushed as a draft body — the owner sees the agent's chain-of-thought rendered as the outbound message to the requester. Tap-Approve-quickly and that gets sent.

**Fix:** backend test that streams a fixture without prefixes ≥ 500 chars and asserts that we error rather than coerce to draft.

---

## 18. [MEDIUM] Snapshot `sse_events` array has no size cap

`agent/runtime/agent_state_store.py` stores `sse_events` as a JSONB array that grows with every event the live stream emits. For a long, tool-heavy turn (10 device tools, 30 surfaces, ~100 tokens of draft) you can easily produce 10-30 KB per snapshot row. Snapshot retry on every iteration (l.81-111) means rewriting the entire blob N times.

**Fix:** event-count cap (e.g. keep the last 200) or move events to an append-only side table.

---

## 19. [MEDIUM] Snapshot save can silently fail and the owner can't tell

`agent_state_store.py:81-111` retries the snapshot save once with 50 ms backoff, then logs and returns `False`. The agent loop continues as if it succeeded. If the failure recurs throughout the turn, no snapshot exists at the end, and the next hydration (foreground recovery, reopen of the screen) finds nothing and triggers `startDraft()` from scratch — discarding the in-flight draft the owner was looking at.

**Fix:** on the third consecutive failure within a turn, emit an SSE `error` so the owner sees something rather than later silent state loss.

---

## 20. [MEDIUM] Reconciler can double-fire device tools on rapid re-mount

`useConsentAgent.ts:658-715` (the `pending_device_tools` reconciler) dedupes via `reconcilerInFlightRef`, but that ref is per-hook-instance. If the user navigates away and back faster than the first `executeDeviceTool` resolves, the second mount starts with an empty `reconcilerInFlightRef` and re-fires the same tool. With native contacts/calendar reads that's a duplicate read; with `request_data_source_connection` it could open a second browser.

**Fix:** hold the dedup set in a module-level singleton keyed by `tool_call_id`.

---

## 21. [LOW] `_normalizeSurfaceKind` silently coerces unknown kinds to `thinking`

`lib/consentSnapshotReplay.ts:29-37`. New surface kinds added on the backend without a UI bump will all render as italic gray "thinking" lines.

**Fix:** log / Sentry breadcrumb so this drift is visible.

---

## 22. [LOW] iOS pageSheet swipe-down — documented limitation

Already called out in `docs/context/consent-agent.md`. Worth fixing by holding the modal with `presentationStyle="fullScreen"` while `!canExit`, or by intercepting `onRequestClose` and toasting the owner. Today, swipe-down on iOS leaves an orphan SSE (combines with #2).

---

## What was not verified dynamically

Local API on `:8000` was not running during this audit, so the following remain static-only:

- **#1 (auth bypass)** — verify with the curl recipe above before acting. There's no middleware that re-asserts auth that I missed in `main.py`, but a 2-minute curl confirms.
- **#3 (race)** — needs a concurrent-load test (`hey -n 50 -c 10` against the same `request_id`).
- **#8 (20s timeout boundary)** — needs a device tool that intentionally sleeps 21s.
- **#5 (dead-code deep-link)** — needs an Android build with InAppBrowser disabled.

RN UI flows cannot be exercised from this audit at all; anything UI-state-shaped (#2, #10, #11, #16, #20, #22) is verified statically only.

---

## Suggested triage order

1. **#1 (auth)** — production-blocking. Add `dependencies=[Depends(get_authenticated_user)]` to the consent router and ownership checks in each handler.
2. **#2 (SSE leak on unmount)** — high visibility, simple fix.
3. **#3 + #4 (backend race + multi-worker assumption)** — add the lock, document the worker constraint.
4. **#5 (deep-link handler)** — register it; belt-and-braces with the AppState poll but cold-start currently breaks.
5. **#6, #7 (size caps on history / draft / instruction)** — defensive but cheap.
6. The MEDIUM list once the above are landed.
7. The LOW list as part of regular hygiene.
