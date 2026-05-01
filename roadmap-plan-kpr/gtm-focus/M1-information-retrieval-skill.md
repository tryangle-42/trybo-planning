# M1 — Information Retrieval Skill

> **Milestone:** M1 — Information Retrieval Skill (Owner: KPR). See `gtm-milestone-plan.md` for the milestone-level demo and beta total.
>
> **Scope of this document:** the **third-party data source layer** of the Information Retrieval Skill — toolkit enablement, OAuth/permission flow via Composio, connection lifecycle, and the React Native surface that lets the user see and manage connected sources. The skill's data-fetch execution layer and the device-source path are tracked as follow-on phases of the same M1 effort and are explicitly deferred from this document — see §3.10 "What M1 Explicitly Defers".
>
> **Governing principle:** Every third-party integration goes through Composio. We do not build provider-specific OAuth, token storage, or API clients in our repo for any third-party data source going forward.

---

## 1. Context

### 1.1 The Problem This Module Solves

Trybo's agents — both the **consent-request agent** that drafts replies for inbound consent requests, and the **main chat agent** that talks to the owner directly — frequently need to look up information about the user to answer a request. That information lives in three physically different places:

| Category | Examples | Where the data lives |
|---|---|---|
| **Third-party providers** the user has authorized | Gmail, Google Calendar, Google Contacts (later: Outlook, Slack, Notion, Drive…) | External provider APIs, behind OAuth |
| **The user's own device** | Device calendar, device contacts, incoming SMS (OTP), location, photos | Locked behind native OS permissions, only readable from the mobile app |
| **Trybo's own database** (future) | Past call transcripts, WhatsApp/cross-channel summaries, agent-run history | Existing Supabase tables |

Each category has different auth, different storage, and a different execution surface. Without a unifying skill, every caller (each agent, each consent draft) re-implements its own retrieval logic, drifts over time, and accrues per-provider integration debt.

The **Information Retrieval Skill** is the unified capability that lets an agent ask "give me the user's X" without having to know which category X belongs to. Both agents — consent-request and main chat — invoke the same skill. M1 lays down the third-party layer of this skill.

### 1.2 Why a Single Skill, Two Callers

Both the consent-request agent and the main chat agent need this exact capability:
- **Consent-request agent** uses it to pre-fill drafts when an inbound consent arrives ("based on the owner's emails, here's what John was asking about").
- **Main chat agent** uses it during free-form chat ("what's on my calendar Thursday?", "summarise recent emails from Sharma").

Building it once means one place to evolve the planner prompt, one place to add new sources, and identical behaviour across both agents.

### 1.3 Why Composio for the Third-Party Layer

Custom OAuth integrations are expensive in maintenance — each provider has its own consent screen, token-refresh quirks, and API client to keep current. Composio is a brokered-credentials platform purpose-built for AI agents:
- **982 toolkits, 11,000+ tools** (Gmail, Outlook, Slack, Notion, Drive, and many more — all reachable through one SDK).
- **Managed OAuth with per-user isolation.** Composio holds the encrypted tokens in its vault, auto-refreshes them, and exposes a uniform tool-execution API. Our backend never sees raw tokens.
- **Agent-native.** Designed for LLM tool calling, integrates cleanly with LangGraph (which we already run).
- **Affordable.** Free tier 20K calls/month, Starter $29/month for 200K.

Adopting Composio as the **only** path for third-party data access lets us add new providers by enabling a slug in a config list — no new code, no new token store, no new API client.

### 1.4 The Brokered Credentials Model

```
TRYBO DB                   COMPOSIO'S VAULT              THIRD-PARTY API
────────                   ────────────────              ───────────────
user_id: "usr_123"   ───>  connected_account: "ca_xyz"
                           ├─ access_token (encrypted)
                           ├─ refresh_token (encrypted)
                           ├─ auto-refresh: YES
                           └─ status: ACTIVE
                                     │
                                     ├──> Gmail API
                                     ├──> Outlook API
                                     └──> ...
```

- **Composio stores** encrypted OAuth tokens, performs auto-refresh, and manages the full lifecycle.
- **Trybo's DB stores** only the opaque `connected_account_id` mapped to the Trybo `user_id`.
- **Tool execution** routes through Composio — they inject the token, call the third-party API, return only the result.
- **Trybo never handles raw tokens.**

### 1.5 What Already Exists

| Capability | Status today |
|---|---|
| Composio HTTP adapter (raw `httpx`) for Slack/Notion/Linear/GitHub tool execution | Working in the agent runtime, single shared `composio_user_id` / `connected_account_id` |
| Composio config fields on `ExecutionContext` (`composio_base_url`, `composio_api_key`, `composio_user_id`, `composio_connected_account_id`) | Working, populated from env or request |
| Custom Google OAuth (backend) — token encrypt / store / refresh / Calendar / Contacts | Working, custom implementation, **slated for removal in this milestone** |
| Custom Google integration (frontend) — Google Sign-In SDK, account-manager helper, profile card | Working, **data-access portions slated for removal in this milestone** |
| Device permission service (calendar, contacts, OTP) | Working, **untouched in this milestone** |
| Consent-request processing service | Working — it'll consume this skill |
| `Data Source Registry` concept (defined in the Consent Flow component) | Categories: device, google. **This milestone extends it with a third category: composio-backed third-party.** |

### 1.6 Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3 + FastAPI |
| Frontend | React Native (bare, **not** Expo) + React Navigation 7 |
| Database | Supabase (PostgreSQL + Auth + Storage + RLS) |
| Agent Orchestration | LangGraph |
| LLM | OpenAI (gpt-4.1), Google Gemini, Ollama |
| Composio | Official Python SDK (`composio-core`) for connection management. (Migration of the existing raw-`httpx` adapter is deferred to the Data Fetch phase.) |
| Mobile OAuth | `react-native-inappbrowser-reborn` (in-app SFSafariViewController on iOS, Custom Tabs on Android) |

---

## 2. Proposed Flow

This section describes the conceptual flow and actor responsibilities. Concrete schemas, endpoint surfaces, and file-level changes live in §3.

### 2.1 The Three Categories at Runtime

Each category has its own physical execution surface, decided by where the data lives:

| Category | Permission flow | Extraction flow | M1 ownership |
|---|---|---|---|
| **Third-party** (Gmail, Calendar, Contacts via Composio) | **Server-side initiated.** Backend creates the OAuth URL, callback lands on backend, backend writes connection state. | **Server-side execution.** Backend calls Composio, Composio calls the provider, result returns to backend. | ✅ This document |
| **Device** (device calendar, contacts, OTP, location, photos) | **Client-side only.** Only the mobile app can ask the OS for permission. | **Client-side only.** Only the mobile app can read device data. | Untouched (existing service) |
| **Internal** (Trybo's own DB — transcripts, summaries) | None — always available to the agent | **Server-side.** Direct Supabase query. | Future |

The unified Information Retrieval Skill will, in a later phase, expose a single contract that branches internally by category. M1 establishes the third-party path so the rest can build on top of it.

### 2.2 Who Decides What — Boundary Between Agent and Skill

| Decision | Owner |
|---|---|
| Whether the user needs information from a particular source for the current turn | Agent (its planner) |
| Which source(s) to consult for that information | Agent (its planner) |
| Whether the source is available, connected, expired, or missing | Skill — by reading the Data Source Registry + connection-state table |
| What to do when a source is needed but not connected (surface a connect prompt, ask the user manually, retry later) | Agent — it owns the chat-side UX |
| Performing the OAuth handshake, persisting connection state, revoking, surfacing the deep-link return | Skill (this milestone) |
| Reading data from a connected source | Skill (deferred fetch phase) |
| Per-source error semantics (expired vs rate-limited vs outage) | Skill (deferred fetch phase) |
| Retry / fallback policy on those errors | Agent (deferred fetch phase) |

Crisp principle: **the agent is the brain, the skill is a thin pipe**. The skill never embeds an LLM of its own; the agent's planner has all the reasoning. The skill simply exposes connection state and (later) executes structured fetch calls the agent has already chosen.

### 2.3 First-Time Connect (Third-Party)

1. The agent decides — during a chat turn or while drafting a consent reply — that it needs data from a source the user hasn't connected. The agent surfaces a "Connect [Provider]" prompt as part of its response. **The agent owns this UX; M1 simply exposes the connection state the agent reads.**
2. The user taps the prompt.
3. The mobile app asks the backend to start an OAuth flow for that source.
4. The backend asks Composio to initiate a connection for this user against that toolkit. Composio returns a redirect URL to the provider's consent screen.
5. The mobile app opens that URL **in the in-app browser**.
6. The user sees the provider's consent screen and grants access. (The screen says "Trybo" or "Composio" depending on whether Trybo has registered custom OAuth credentials in the Composio dashboard for this toolkit — this is dashboard-only configuration, not a code change.)
7. The provider redirects to Composio's callback. Composio exchanges the code for tokens, encrypts and stores them in its vault.
8. Composio redirects to **Trybo's** backend callback URL with the new `connected_account_id`.
9. The backend verifies the connection with Composio, writes a row to `user_data_source_connections` (one row per `(user, source)`), then returns an HTTP 302 to the deep link `trybo://data-sources/connected?source_key=…`.
10. The in-app browser observes the `trybo://` URL and dismisses itself, returning the user to the app.
11. The mobile app's deep-link router fires, refreshes the connection list, and notifies any waiting screen.
12. The agent now sees the source as connected (via `ExecutionContext.connected_data_sources`) and can fetch from it on the next turn.

**Important:** the deep link is a **notification, not a data carrier**. All persistent state was already written to Trybo's DB by the backend before the deep link fired. The deep link's only job is "browser, close yourself; app, refresh state."

### 2.4 Connection-State Check

Whenever the skill (or any caller) needs to know "is X connected for this user right now?", it reads the row in `user_data_source_connections` and returns:

| State | Meaning |
|---|---|
| Row absent | Source is not connected. Agent will likely surface a connect prompt. |
| Row present, status=`active` | Source is connected and ready. |
| Row present, status=`expired` | Token refresh failed at some point during a fetch. UI surfaces "Reconnect". |

There is **no live Composio status check on the read path** in M1 — the row is the source of truth. It gets updated lazily when something fails. (Composio webhooks and proactive pollers are deferred.)

### 2.5 Disconnect (Full Revoke)

The user disconnects a source from the Connected Data Sources screen. The backend:
1. Asks Composio to delete (revoke) the OAuth grant for that `connected_account_id`. Composio revokes the token at the provider — the user sees Trybo disappear from their Google connected-apps page.
2. Deletes the row from `user_data_source_connections`.

There is no `disconnected` status; row absence means disconnected.

### 2.6 Reconnect (Expired or Re-auth)

The user (or the agent on the user's behalf) hits the same connect endpoint for a source that already has a row. The backend always:
1. Revokes the existing `connected_account_id` at Composio first (so no orphan in the vault).
2. Initiates a fresh OAuth as if it were a first-time connect.
3. The callback handler upserts the row with the new `connected_account_id` and `status=active`.

This makes one endpoint serve all three cases: fresh connect, expired reconnect, stale-active edge case.

### 2.7 What the Mobile App Sees

- **Profile screen:** a new **Connected Data Sources** card replaces the existing Google card. The card shows a brief summary (e.g., a count of currently-connected sources). Tapping it opens a dedicated screen.
- **Connected Data Sources screen:** lists every row from the user's `user_data_source_connections`. Each `active` row has a Disconnect button. Each `expired` row has a Reconnect button. **There is no Connect button on this screen** — new connections are initiated only by the agent in context.
- **Chat UI:** unchanged in M1. The agent's "Connect [Provider]" prompts are an agent-side UX concern handled outside this milestone.
- **Consent screen:** the existing rewire is mechanical — calls that used to fetch Google data through the frontend account-manager helper now go through the backend's data-source surface (the backend fetch endpoint is deferred, so M1 leaves these call sites stubbed until the fetch phase ships).

### 2.8 What the Agent Sees

`ExecutionContext` carries a new field `connected_data_sources: dict[str, str]` keyed by source slug → `connected_account_id`. The agent's planner reads it during run preparation and knows which sources are available without any additional lookup. This dict replaces the previous singular `composio_connected_account_id` field. The future fetch skill consumes this dict to look up the right account ID for a given fetch.

### 2.9 The Two Surfaces of Data Access (Future Tension)

Third-party flows are server-side; device flows are client-side. The fetch phase will need a single skill contract that bridges both — likely via a LangGraph interrupt pattern where device-source requests pause the run, instruct the app to perform the read, then resume on result delivery. **M1 does not solve this**; it is named here so the deferred design has a clear pointer.

---

## 3. Execution Details

This section describes exactly what gets built. File-level placements and concrete schemas live here, not in the flow section above.

### 3.1 V1 Enabled Data Sources

The backend keeps a thin list of Composio toolkit slugs at `trybot-api/app/constants/data_source_registry.py`:

```python
ENABLED_DATA_SOURCES: list[str] = [
    "gmail",            # M1 v1
    "googlecalendar",   # M1 v1
    "googlecontacts",   # M1 v1
    # --- Below: candidates for future enablement, NOT in M1 v1 ---
    # "googledrive",
    # "outlook",
    # "outlookcalendar",
    # "slack",
    # "notion",
    # "dropbox",
]
```

Adding a future toolkit is a one-line change once it's enabled in Composio's dashboard. Labels, icons, and available actions are queried from Composio's SDK at runtime — no static metadata duplicated locally.

**Device sources** (`device_calendar`, `device_contact`, `device_otp`) remain in the existing consent flow registry and use native OS permissions. M1 does not touch them.

### 3.2 Connection State (Database)

A new Supabase migration creates `user_data_source_connections`:

```sql
CREATE TABLE user_data_source_connections (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             uuid NOT NULL REFERENCES user_profiles(id) ON DELETE CASCADE,
    source_key          text NOT NULL,            -- Composio toolkit slug, e.g. "gmail"
    composio_account_id text NOT NULL,            -- e.g. "ca_xyz789"
    status              text NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'expired')),
    connected_at        timestamptz NOT NULL DEFAULT now(),
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),

    UNIQUE(user_id, source_key)
);

ALTER TABLE user_data_source_connections ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users_own_connections" ON user_data_source_connections
    USING (user_id = auth.uid());
```

The migration file lives at `trybot-ui/supabase/migrations/YYYYMMDDHHMMSS_create_user_data_source_connections.sql` (the symlinked migrations directory used by both repos).

### 3.3 Backend Endpoint Surface

A new router `trybot-api/app/routes/data_sources.py` is wired into `main.py` and exposes:

| Endpoint | Purpose |
|---|---|
| `POST /api/data-sources/connect` | Initiate OAuth. Always revokes any existing `connected_account_id` for `(user, source)` first, then asks Composio for a fresh redirect URL. Returns the redirect URL to the mobile app. |
| `GET /api/data-sources/callback` | Composio's redirect target after the user grants. Verifies via Composio SDK, upserts the row, 302s to `trybo://data-sources/connected?source_key=…`. |
| `GET /api/data-sources/connections` | Returns the authenticated user's full list of connection rows. Powers the Connected Data Sources screen. |
| `GET /api/data-sources/connections/{source_key}/status` | Returns `active` / `expired` / `not_connected` for a single source. |
| `DELETE /api/data-sources/connections/{source_key}` | Full revoke: asks Composio to delete the connection (revokes at provider), then deletes the row. |
| `GET /api/data-sources/enabled` | Returns the enabled list with metadata (label, icon) queried from Composio's SDK. |

All endpoints behind `Depends(get_authenticated_user)` (the existing JWT auth dependency). Auth-token attachment from RN matches the existing pattern.

### 3.4 Identity in Composio

`composio_user_id = str(user_profile_id)` everywhere. No mapping table, no extra column — the Trybo profile UUID is the Composio identity directly. Stable, unique, and avoids an indirection.

### 3.5 ExecutionContext Shape Change

In `trybot-api/app/agentic_mvp/runtime/contracts.py`, `ExecutionContext` is updated:

| Before (single value) | After (dict) |
|---|---|
| `composio_user_id: Optional[str]` | `composio_user_id: Optional[str]` (unchanged) |
| `composio_connected_account_id: Optional[str]` | **removed** — replaced by the dict below |
| — | `connected_data_sources: dict[str, str]` — `{source_slug: connected_account_id}` |

The existing raw-`httpx` adapter (`marketplace_composio.py`) is migrated to look up account IDs from the dict by toolkit slug **as part of the deferred fetch phase**, not M1. Until that migration ships, that adapter continues to read from the old singular field via a backwards-compat shim local to that file. (M1 keeps the old field as a deprecated alias on `ExecutionContext` until the fetch phase removes it.)

### 3.6 Backend Setup Module

A new service `trybot-api/app/services/data_source_connection_service.py` wraps the Composio SDK calls used by the route handlers: initiate, get-by-id, delete (revoke), upsert-row, list-rows. Pure adapter — no business logic embedded.

### 3.7 Frontend — Additions

| Addition | Location |
|---|---|
| `react-native-inappbrowser-reborn` dependency + iOS pod / Android linking | `trybot-ui/package.json` + native projects |
| `trybo://` scheme registration on iOS | `trybot-ui/ios/.../Info.plist` (`CFBundleURLTypes`) |
| `trybo://` scheme registration on Android | `trybot-ui/android/.../AndroidManifest.xml` (`<intent-filter android:scheme="trybo">`) |
| Central deep-link router | `trybot-ui/lib/deepLinkRouter.ts` (or inline in `App.tsx`) — single `Linking.addEventListener('url', …)`, dispatches by path. `data-sources/connected` is the first consumer; future deep links plug in alongside. |
| RN-side data-source API client | `trybot-ui/services/dataSourceService.ts` — wraps the six `/api/data-sources/*` endpoints |
| Redux slice for connection list | `trybot-ui/store/dataSourcesSlice.ts` — refreshed on screen mount and on every `data-sources/connected` deep-link callback |
| `ConnectedDataSourcesCard` (Profile screen card) | `trybot-ui/components/cards/ConnectedDataSourcesCard.tsx` — replaces the existing Google card |
| `ConnectedDataSourcesScreen` | `trybot-ui/screens/ConnectedDataSourcesScreen.tsx` — list with Disconnect on active rows and Reconnect on expired rows; **no Connect button** |

### 3.8 Frontend — Removals

| Removal | Location |
|---|---|
| Google data-access account-manager helper | `trybot-ui/lib/GoogleAccountManager.ts` — deleted |
| Existing Profile-screen Google card | `trybot-ui/components/.../GoogleIntegrationCard.tsx` — deleted (replaced by Connected Data Sources card) |
| Direct calls to `GoogleAccountManager.getCalendarEvents()` / `searchContactsByPhone()` in the consent service | `trybot-ui/services/informationConsentService.ts` — rewired to call the new backend data-source endpoints (which become live in the deferred fetch phase; M1 leaves these as stubbed paths) |

### 3.9 Frontend — Preserved (Out of Scope)

- `trybot-ui/lib/GoogleAuthService.ts` — Google Sign-In service. Not wired into the app today; preserved untouched for future login integration.
- `trybot-ui/services/permissionService.ts` — device permission service. Untouched.

### 3.10 Backend — Removals

The custom Google data-access stack is removed (no feature flag, full cutover):

| Removal | Location |
|---|---|
| Custom Google service (token storage/refresh + Calendar/Contacts orchestration) | `trybot-api/app/services/google_service.py` |
| Direct Google API HTTP client | `trybot-api/app/services/google_api_client.py` |
| Token CRUD in `user_profiles.settings_jsonb.google` | `trybot-api/app/services/google_user_profile_service.py` |
| Fernet encryption for Google tokens | `trybot-api/app/services/token_encryption.py` (Google-only paths; if shared with another feature, only Google paths are removed) |
| Custom Google routes | `trybot-api/app/routes/google.py` |

**Legacy data is not migrated.** Existing Google tokens in `user_profiles.settings_jsonb.google` are left in place. The new code path never reads them; pre-production users force-reconnect via Composio on their next interaction needing Google data.

### 3.11 What M1 Explicitly Defers

The following are part of the broader Information Retrieval Skill but **not** in M1's scope. They land in a follow-on **Data Source Fetch** component:

- **Generic backend fetch service** that the agent calls to read data from a connected source.
- **Migration of the existing raw-`httpx` `marketplace_composio.py` adapter** to the official SDK.
- **LangGraph planner-prompt updates** that teach the agent which sources are connected and how to invoke the fetch skill.
- **Skill-level error taxonomy and retry policy** (distinct error types: `connection_expired`, `composio_unavailable`, `provider_rate_limited`, etc.).
- **The writer of `status='expired'`** — that lives on the fetch auth-error path.
- **Composio webhooks** for proactive connection-status updates.
- **Background pollers** for connection health.
- **Device-source path** for the unified skill (LangGraph interrupt + app-side execution + result-resume).
- **Inline "Connect [Provider]" CTAs in chat bubbles** — agent-side UX.

### 3.12 What Trybo Owner Provides Outside Code

- A Composio account.
- `COMPOSIO_API_KEY` (and `COMPOSIO_BASE_URL` only if non-default). Both already exist as backend env vars.
- Toolkit enablement on the Composio dashboard for `gmail`, `googlecalendar`, `googlecontacts`.
- (Optional, any time post-launch) Custom OAuth credential registration in Google Cloud Console, paste into Composio's dashboard, when "Trybo"-branded consent screens are wanted instead of "Composio"-branded ones. **No code change required.**

### 3.13 Phasing & Effort

All implementation is done with Claude. Claude writes code and unit tests. Human time is for device verification, OAuth/Composio dashboard configuration, and on-device testing. Working day = 8 hours.

| Phase | In M1? | Hours | Days |
|---|---|---|---|
| **P1** — Composio Python SDK setup + DB migration + connection service wrapper | ✅ | 4.75 | 0.6 |
| **P2** — FastAPI connection endpoints (connect / callback / list / status / disconnect / enabled) + unit tests | ✅ | 6.5 | 0.8 |
| **P3** — Generic data-fetch service + migrate `marketplace_composio.py` to SDK | ❌ deferred | — | — |
| **P4** — Backend Google cleanup (remove custom-OAuth Google data-access stack) | ✅ | 4.25 | 0.5 |
| **P5** — RN integration (in-app browser dep, `trybo://` scheme, deep-link router, Profile-card replacement, Connected Data Sources screen, consent rewire stubs) | ✅ | 15 | 1.9 |
| **P6** — LangGraph agent awareness + planner-prompt updates + fetch-skill registration | ❌ deferred | — | — |
| End-to-end device testing on Android + iOS for the connect / disconnect / reconnect / deep-link / browser-return flows | ✅ | 9 | 1.1 |
| **M1 Total** | | **39.5 hours** | **~5 days** |

> Deferred items (P3 + P6) move to the **Data Source Fetch** component, which depends on M1 being live. Combined deferred budget is ~13.5 hours / ~1.7 days; that's the next ticket, not M1.

**Recommended execution order:**
- Day 1: P1 → start P2.
- Day 2: finish P2 → start P4.
- Day 3: finish P4 → start P5 (deep-link scaffolding + in-app browser dependency).
- Day 4: P5 (Profile card replacement, Connected Data Sources screen, consent rewire stubs).
- Day 5: end-to-end device testing on Android + iOS for the full connect / disconnect / reconnect / deep-link return loop.

Critical path: P1 → P2 → P5. P4 runs alongside P5 once P2 is done.

---

## 4. Rationalised Questions — Why This, Why Not That

### 4.1 Why Composio, not direct OAuth per provider?

Direct OAuth per provider means re-implementing token storage, encryption, refresh, scope handling, and an API client for each toolkit — and keeping all of that current as providers evolve. That's a permanent maintenance cost that grows linearly with the number of sources. Composio collapses it into one integration and one config: enable a slug, get the toolkit. It's also AI-agent-native (built around LLM tool calling), works cleanly with LangGraph, and per-user isolation comes for free. The trade is vendor dependency, which we accept for the MVP and revisit if scale or compliance demands it.

### 4.2 Why is the "Trybo" branding on the OAuth consent screen a dashboard config, not a code change?

Composio decouples OAuth credential configuration from runtime code. The same SDK call (`composio_client.connected_accounts.initiate(user_id, toolkit_slug)`) works whether the toolkit is set to default Composio OAuth or to custom Trybo-registered credentials. Whichever is set in the dashboard for that toolkit is what the user sees. Trybo can ship M1 with default branding and flip to custom branding once Google's app verification is complete — without a code release.

### 4.3 Why a thin enabled list instead of a rich static registry?

Composio already maintains the canonical metadata for each toolkit (label, icon, action catalog). Duplicating any of that locally guarantees drift the moment Composio ships an update. A thin slug list keeps the source of truth where it belongs (Composio) and reduces the local-config surface to a single line per source. Adding a new source post-M1 is therefore: register custom OAuth (optional), enable in Composio dashboard, add slug. No registry-config sprawl.

### 4.4 Why server-side fetch only, never RN → Composio?

Three reasons. First, trust boundary — RN never sees raw provider data, only what our backend chooses to surface. Second, single credential surface — the Composio API key lives in one environment, not bundled in a mobile app. Third, symmetry with how the agents already run — both the consent-request agent and the main chat agent already execute on the backend, so server-side fetching means one execution path for both callers, not a split between mobile and server.

### 4.5 Why no "Connect [Provider]" button in any settings screen?

Connections are most useful when they're contextual — the agent knows it needs Gmail to answer the current turn, surfaces the prompt in chat, the user connects, the agent resumes. A connect-button-in-settings creates a dead garden: a list users rarely visit and never use. The Connected Data Sources screen exists for visibility (seeing what's connected) and reversibility (disconnecting / reconnecting), not for initiation. Initiation is the agent's job, not a settings menu.

### 4.6 Why full revoke on disconnect, not just delete the row?

If we only delete the local row, the user's Google connected-apps page still shows "Trybo" as authorised, and the OAuth grant lingers at the provider — a privacy and trust hole. Disconnect should mean disconnect everywhere: tokens revoked at the provider, account removed at Composio, row deleted in our DB.

### 4.7 Why revoke-then-reinitiate on reconnect, not just initiate?

Reconnecting without first deleting the existing `connected_account_id` would leave an orphaned account in Composio's vault — paid for but unused, and visible in dashboards. Always revoking first means one uniform endpoint handles fresh connect, expired reconnect, and the stale-active edge case (e.g., duplicate tap) without special-casing.

### 4.8 Why force-reconnect existing Google users instead of silent migration?

We're pre-production. Token-import APIs at Composio (or any vendor) are imperfect and the engineering cost to plumb them is significant. Force-reconnect is one tap for a small dev cohort and trivially clears the slate. The cleanup migration to wipe the legacy JSONB tokens isn't even necessary — the new code path simply doesn't read them.

### 4.9 Why two status values (`active` / `expired`) and no `disconnected`?

Disconnect deletes the row outright. Row absence already means "not connected." A `disconnected` status would be a tombstone with no use case — we don't need an audit trail in v1, and treating row presence as "connected" makes every read simpler.

### 4.10 Why a dict on `ExecutionContext` instead of a single account ID?

A single user can have many simultaneous third-party connections (Gmail and Calendar and Contacts, each with its own `connected_account_id`). One singular field can't carry all three. The dict is the natural shape and scales to any future enabled source without further schema changes. The existing raw-`httpx` adapter migrates to read from this dict in the deferred fetch phase.

### 4.11 Why does the agent decide which source to call (dumb skill), not the skill (smart skill)?

A smart skill that embeds its own LLM call to pick sources duplicates work the agent's planner is already paying for — two LLM hops instead of one, twice the latency, twice the cost. It also blinds the agent to the choice (the agent can't recover when the skill picks the wrong source). LangGraph's strength is structured tool calls with typed parameters; the cleanest pattern is a thin skill the agent calls with a fully-formed intent. M1 sets up `connected_data_sources` so the agent's planner has the visibility it needs; the skill's contract gets defined in the deferred fetch phase, but the architectural commitment to "agent reasons, skill executes" is made here.

### 4.12 Why an in-app browser instead of `Linking.openURL` + system browser?

`Linking.openURL` sends the user to their default browser — the app fully backgrounds, the OS may delay the return on the deep link, and the transition feels jarring on a consumer app. `react-native-inappbrowser-reborn` opens SFSafariViewController on iOS / Custom Tabs on Android: the browser appears as a modal sheet over the app, dismisses cleanly when the `trybo://` redirect fires, and the user's mental model never leaves Trybo. The dependency is well-maintained and standard for this use case.

### 4.13 Why is the deep link a notification, not a data carrier?

All persistent state — the connection row in `user_data_source_connections` — is written by the backend before the deep link is issued. The deep link's only job is "browser, close yourself; app, refresh state." Carrying sensitive data through a URL would mean the OS, intermediate apps, or system logs could see it. Keeping the deep link payload minimal (`source_key`, `status=success`) means the URL itself is non-sensitive.

### 4.14 Why preserve `GoogleAuthService.ts` but delete `GoogleAccountManager.ts`?

`GoogleAuthService.ts` is the Google Sign-In login surface (identity for the user's Trybo account). Login isn't a "data source" — Composio doesn't replace login systems, and the file isn't even wired into the app today. We preserve it for future login integration. `GoogleAccountManager.ts` does Google **data access** (Calendar/Contacts fetches from RN). Composio fully replaces this, and the file goes away.

### 4.15 Why no migration to delete legacy JSONB Google tokens?

Pre-production. The new code path never reads `user_profiles.settings_jsonb.google`. The values are harmless dead bytes for a small dev cohort and migration churn for zero user impact isn't worth it. Once we hit a real user base, a one-line cleanup migration can run at any time without consequence.

### 4.16 Why defer the data-fetch service and planner-prompt updates from M1?

The connection layer is independently shippable and demoable: the user can connect, disconnect, see status. The fetch layer carries its own design questions — the smart-vs-dumb skill split, the device-source interrupt pattern, the action catalog shape, the error taxonomy — and benefits from being designed against a working OAuth flow rather than imagined. Splitting also keeps M1 to ~5 days, which fits the milestone budget.

### 4.17 Why is M1 the right milestone for this?

M1's demo is "user connects Google → real calendar events / real emails appear." That demo cannot run without the third-party connection layer landing somewhere. Putting it here is the latest possible point — any later, and M1's demo can't ship.

### 4.18 What's the failure mode if Composio goes down?

All third-party connections stop: connect, disconnect, status check, fetch (when fetch ships). The fallback for M1 is graceful: connect endpoints return an error the app surfaces as a temporary outage; existing connection rows continue to be readable from our DB so the Connected Data Sources screen still works in read-only form. Composio is SOC 2 Type 2 compliant, so the operational risk is acceptable for MVP. Long-term mitigations (raw-token fallback by disabling credential masking, self-hosted Composio enterprise) are out-of-scope.

### 4.19 What's the failure mode if a token expires silently?

In M1, the row says `active` until something tries to use it and fails. M1 doesn't fetch (fetch is deferred), so there's no path that flips a row to `expired` during M1. Once the fetch phase ships, an auth-error during fetch flips the row to `expired`, the UI reflects "Reconnect", and the agent treats the source as unavailable until the user reconnects. This is acceptable because the worst case is "one wasted fetch attempt before the UI catches up."

### 4.20 What about provider-side OAuth verification (e.g., Gmail's restricted scopes)?

Custom OAuth — when Trybo's own Google Cloud project requests Gmail read scopes — triggers Google's app verification process, which can take 4–6 weeks. Default Composio OAuth bypasses this with a "Composio"-branded screen. M1 can ship with default branding and flip to "Trybo" branding the moment Google completes verification, without any code change or release.

### 4.21 What about Composio rate limits and cost?

Free tier is 20K calls/month; Starter is $29/month for 200K. M1 itself hardly consumes budget — connections are infrequent, and fetches are deferred. Cost scaling becomes meaningful only once the fetch phase ships and agents are calling Composio per turn, which is a separate budget conversation.

### 4.22 What about multiple Google scopes (Gmail vs Calendar vs Contacts)?

Each Composio toolkit (`gmail`, `googlecalendar`, `googlecontacts`) is a separate connection with its own scopes. The user connects them individually as the agent needs each. A future "Connect Google" macro that requests all three at once is a UX improvement, not an M1 concern.

### 4.23 Where does the unified Information Retrieval Skill come together?

M1 builds the third-party connection layer. The deferred Data Source Fetch component builds the skill execution layer (third-party fetches + device-source interrupt pattern + planner-prompt updates). Together they form the Information Retrieval Skill that both the consent-request agent and the main chat agent invoke. Internal-DB sources (transcripts, summaries) plug into the same skill in a later milestone.
