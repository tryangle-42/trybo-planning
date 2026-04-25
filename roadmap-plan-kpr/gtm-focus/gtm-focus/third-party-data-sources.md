# Third-Party Data Sources — Multi-Provider Integration via Composio

## Context

The platform currently accesses two categories of user data:

1. **Device sources** — calendar, contacts, OTP from the user's phone (native OS permissions)
2. **Google sources** — Google Calendar, Google Contacts (integrated via `GoogleService` on the backend and `GoogleAccountManager` / `GoogleAuthService` on the React Native frontend)

Both are handled through the **Data Source Registry** defined in the Consent Flow Cross-Channel Enhancement component. The registry maps each source key to a category, grant method, and fetch method. The consent UI and chat UI use this registry to decide what it can auto-fetch, what needs permission/connection, and what it can't handle.

This component **extends the Data Source Registry to support third-party data sources** — Gmail, Outlook, Slack, Notion, Google Drive, etc. — using **Composio** as the unified integration layer.

### What Already Exists

Composio is **already partially integrated** in the codebase:

| What | Where | Status |
|---|---|---|
| Composio HTTP adapter | `trybot-api/app/agentic_mvp/runtime/adapters/marketplace_composio.py` | Working — executes Slack, Notion, Linear, GitHub tools via raw `httpx` |
| ExecutionContext fields | `contracts.py` — `composio_base_url`, `composio_api_key`, `composio_user_id`, `composio_connected_account_id` | Working — used by agent orchestration |
| Config fallback | `contracts.py` — reads from env vars `COMPOSIO_BASE_URL`, `COMPOSIO_API_KEY`, etc. | Working |
| Google OAuth (backend) | `google_service.py`, `google_api_client.py`, `google_user_profile_service.py` — custom token encrypt/store/refresh | Working — custom implementation, not via Composio |
| Google OAuth (frontend) | `trybot-ui/lib/GoogleAuthService.ts`, `GoogleAccountManager.ts` | Working — React Native Google Sign-In |
| Consent request processing | `information_request_processing_service.py` | Working |

### Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | Python 3 + FastAPI (at `trybot-api/`) |
| **Frontend** | React Native + Expo (at `trybot-ui/`) |
| **Database** | Supabase (PostgreSQL + Auth + Storage + RLS) |
| **Agent Orchestration** | LangGraph (at `trybot-api/app/agentic_mvp/`) |
| **LLM** | OpenAI (gpt-4.1), Google Gemini, Ollama |
| **Composio** | Migrating from raw `httpx` to official Python SDK (`composio-core`) |

### Why Composio

Composio is the only unified API platform purpose-built for AI agents. It provides:
- **982 toolkits, 11,000+ tools** — Gmail, Google Calendar, Outlook, Slack, Notion, Dropbox, Twitter, Instagram, LinkedIn, HubSpot, and more
- **Managed OAuth with per-user isolation** — each user connects their own accounts, Composio handles token storage, refresh, and rotation
- **AI agent native** — designed for LLM tool calling, MCP support, works with LangGraph (which we already use)
- **Affordable** — free tier (20K calls/mo), $29/mo for 200K calls

### The Brokered Credentials Model

Composio operates a brokered credentials architecture:

```
YOUR DB                    COMPOSIO'S VAULT              THIRD-PARTY API
───────                    ────────────────              ───────────────
user_id: "usr_123"    ──>  connected_account: "ca_xyz"
                           ├─ access_token (encrypted)
                           ├─ refresh_token (encrypted)
                           ├─ auto-refresh: YES
                           └─ status: ACTIVE
                                     │
                                     ├──> Gmail API
                                     ├──> Outlook API
                                     └──> Slack API
```

- **Composio stores** encrypted OAuth tokens, handles auto-refresh, manages the full lifecycle
- **Your DB stores** only the `connected_account_id` (e.g., `ca_xyz789`) mapped to your `user_id`
- **Tool execution** routes through Composio — they inject the token, call the third-party API, return only the result
- **Your app never sees raw tokens** (unless you disable credential masking for fallback scenarios)

### Branding: "Trybo" on Consent Screens

For each provider we support, **custom OAuth credentials are registered** (e.g., Google Cloud Console for Gmail, Microsoft Azure for Outlook, Slack API for Slack). These credentials are configured in Composio's dashboard so that when a user connects a data source, the OAuth consent screen shows **"Trybo"**, not "Composio".

**Only providers with custom OAuth configured are available as data sources.** We do not fall back to Composio's managed auth for other providers. The backend maintains a simple list (`ENABLED_DATA_SOURCES`) of toolkit slugs for which custom OAuth is configured. KPR updates this list as new providers are configured.

### How the OAuth Flow Works

```
1. User taps "Connect Gmail" (in chat UI or consent screen)
       │
2. React Native app calls YOUR FastAPI backend:
       │   POST /api/data-sources/connect  { source_key: "gmail" }
       │
3. FastAPI checks: is "gmail" in ENABLED_DATA_SOURCES? → YES
       │
4. FastAPI calls Composio Python SDK:
       │   composio_client.connected_accounts.initiate(
       │       user_id=user_id,
       │       toolkit_slug="gmail",
       │       config={"callback_url": "https://api.trybo.com/api/data-sources/callback"}
       │   )
       │   → Composio uses YOUR custom OAuth credentials (registered in dashboard)
       │   → Returns redirect_url (Google's consent page, branded as "Trybo")
       │
5. FastAPI returns redirect_url to React Native app
       │
6. React Native opens URL via Linking.openURL(redirect_url)
       │   → User sees Google's "Allow Trybo to access your Gmail?" screen
       │   → User grants permission
       │
7. Google redirects to Composio's callback
       │   → Composio exchanges code for tokens, encrypts & stores them
       │   → Composio redirects to YOUR callback:
       │      https://api.trybo.com/api/data-sources/callback
       │        ?connected_account_id=ca_xyz789&status=success
       │
8. FastAPI callback handler saves ca_xyz789 in Supabase
       │   → Redirects to deep link: trybo://data-sources/connected?source_key=gmail
       │
9. React Native receives deep link → re-renders UI
       │
10. Done. Agent executes tools via Composio Python SDK:
        composio_client.tools.execute(
            action="GMAIL_FETCH_EMAILS",
            params={...},
            entity_id=user_id,
            connected_account_id="ca_xyz789"
        )
```

### Two Surfaces Where "Connect" Appears

This component serves two UI surfaces — both trigger the same Composio OAuth flow:

| Surface | When | What Renders |
|---|---|---|
| **Chat UI** | Agent needs data from a source the user hasn't connected | Inline "Connect Gmail" / "Connect Outlook" button in the chat |
| **Consent Screen** | `missing_data_sources` includes an unconnected third-party source | "Connect [Provider]" banner (Section 4.2 of consent flow doc) |

### Goals

1. **Thin enabled-sources list** — backend maintains only a list of Composio toolkit slugs for which custom OAuth is configured. Labels, icons, available actions come from Composio SDK dynamically.
2. **Unified connection management** — one pattern for all third-party providers: initiate OAuth via Composio SDK, store `connected_account_id`, check connection status, fetch data via tool execution
3. **Two-surface rendering** — "Connect [Provider]" buttons in both chat UI and consent screen, same backend flow
4. **Agent data access** — agent fetches user data from connected sources via Composio SDK during orchestration
5. **Connection lifecycle** — connect, check status, refresh, disconnect, re-connect
6. **Trybo branding** — all consent screens show "Trybo", not "Composio"

---

## The Enabled Data Sources List

Instead of a large static registry defining labels, icons, actions, and auth configs for every source, the backend maintains a **simple list of toolkit slugs**. Everything else is queried from Composio SDK at runtime.

```python
# trybot-api/app/constants/data_source_registry.py

# Updated by KPR as custom OAuth configurations are added in Composio's dashboard.
# ONLY add a slug here after custom OAuth credentials have been registered with
# the provider (Google, Microsoft, Slack, etc.) AND configured in Composio dashboard.
# This ensures the OAuth consent screen shows "Trybo" branding, not "Composio".

ENABLED_DATA_SOURCES: list[str] = [
    # "gmail",
    # "googlecalendar",
    # "googlecontacts",
    # "googledrive",
    # "outlook",
    # "outlookcalendar",
    # "slack",
    # "notion",
    # "dropbox",
]
```

**Device sources** (`device_calendar`, `device_contact`, `device_otp`) remain in the existing consent flow registry — they are NOT Composio-backed and use native OS permissions. This component only handles third-party sources.

### How the System Uses This List

```
LLM determines: "I need the user's Gmail to answer this question"
  → LLM outputs: data_source = "gmail"
  │
  ├── Backend checks: is "gmail" in ENABLED_DATA_SOURCES?
  │
  ├── IF NO → this source is not available
  │     → Same behavior as "unknown source" in consent flow doc (Section 6.1)
  │     → No auto-fetch, no connect button
  │     → Blank text input, user answers manually
  │
  ├── IF YES → check if user has connected "gmail"
  │     → Query Supabase: user_data_source_connections WHERE user_id AND source_key = "gmail"
  │     │
  │     ├── NOT CONNECTED → render "Connect Gmail" button (chat UI or consent screen)
  │     │     → On tap: Composio OAuth flow → user connects → data becomes available
  │     │
  │     └── CONNECTED (status: active) → fetch data via Composio SDK
  │           → Agent uses the data to answer the question
  │
  └── Dynamic metadata from Composio SDK (not hardcoded):
        → Label/display name: queried from Composio toolkit metadata
        → Available actions: queried from Composio tools list for this toolkit
        → Icon: provided by Composio or mapped locally
```

### Why This Design

| Approach | Pros | Cons |
|---|---|---|
| **Large static registry** (previous plan) | All metadata in one place, no runtime queries | Must maintain labels/actions/icons for every source. Falls out of sync when Composio updates. Hardcodes 200+ lines of config. |
| **Thin enabled list + dynamic metadata** (this plan) | Simple to maintain. KPR just adds a slug when ready. Metadata always fresh from Composio. | One extra Composio SDK call at runtime for toolkit metadata. Cached, so negligible. |

---

## Component Responsibility

```
┌──────────────────────────────────────────────────────────────────┐
│     THIS COMPONENT'S RESPONSIBILITY                              │
│                                                                  │
│  Connection Management:                                          │
│  1. Maintain ENABLED_DATA_SOURCES list (updated by KPR)          │
│  2. Initiate OAuth flows via Composio Python SDK                 │
│  3. Store connected_account_id in Supabase (mapped to user_id)   │
│  4. Check connection status (active/expired/missing)             │
│  5. Handle token refresh failures (re-prompt user)               │
│  6. Disconnect / revoke connections                              │
│                                                                  │
│  Data Fetching:                                                  │
│  7. Execute Composio tools to fetch user data from connected     │
│     sources (called by agent during orchestration or by          │
│     consent UI during draft generation)                          │
│  8. Query available actions dynamically from Composio SDK        │
│                                                                  │
│  UI Integration:                                                 │
│  9. Provide connection status to chat UI (for inline connect     │
│     buttons) and consent UI (for missing_data_sources banners)   │
│  10. Handle OAuth callback from Composio → update Supabase       │
│      → deep link back to React Native app                       │
│                                                                  │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ↓
┌──────────────────────────────────────────────────────────────────┐
│     NOT THIS COMPONENT'S RESPONSIBILITY                          │
│                                                                  │
│  - Device source permissions (calendar, contacts, OTP)           │
│    → handled by permissionService.ts (existing)                  │
│                                                                  │
│  - Deciding WHEN to request a data source connection             │
│    → channel agent's InformationRetrievalAnalyzer decides        │
│    → or the agent during chat orchestration decides              │
│                                                                  │
│  - Consent request lifecycle (persist, notify, route response)   │
│    → handled by Consent Flow component                           │
│                                                                  │
│  - What the agent does with fetched data                         │
│    → agent's own judgment based on user request                  │
│                                                                  │
│  - Registering OAuth credentials with providers (Google, etc.)   │
│    → done by KPR in provider developer consoles + Composio       │
│      dashboard. Then KPR adds the slug to ENABLED_DATA_SOURCES.  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## How It Should Work

### Connection Flow (Same for All Providers)

```
User taps "Connect [Provider]" (chat UI or consent screen)
  │
  ├── React Native calls FastAPI backend:
  │     POST /api/data-sources/connect  { source_key: "gmail" }
  │
  ├── FastAPI checks: is "gmail" in ENABLED_DATA_SOURCES? → YES
  │
  ├── FastAPI calls Composio Python SDK:
  │     composio_client.connected_accounts.initiate(
  │         user_id=str(user_id),
  │         toolkit_slug="gmail",
  │         config={"callback_url": f"{settings.API_BASE_URL}/api/data-sources/callback"}
  │     )
  │
  ├── Composio returns redirect_url → provider's OAuth page (branded "Trybo")
  │
  ├── FastAPI returns { redirect_url } to React Native
  │
  ├── React Native opens URL via Linking.openURL(redirect_url)
  │     → User sees provider's consent screen: "Allow Trybo to access your Gmail?"
  │     → User grants permission
  │
  ├── Provider redirects to Composio's callback
  │     → Composio exchanges code for tokens, encrypts & stores
  │     → Composio redirects to YOUR callback URL:
  │        https://api.trybo.com/api/data-sources/callback
  │          ?connected_account_id=ca_xyz789&status=success
  │
  ├── FastAPI callback handler:
  │     → Saves to Supabase: user_data_source_connections table
  │       { user_id, source_key: "gmail", composio_account_id: "ca_xyz789",
  │         status: "active", connected_at: now() }
  │     → Redirects to deep link: trybo://data-sources/connected?source_key=gmail
  │
  └── React Native receives deep link → re-renders:
        → Chat UI: removes "Connect Gmail" button, shows "Gmail connected"
        → Consent screen: removes banner, fetches data, regenerates draft
```

### Connection Status Checking

Before fetching data or rendering UI, the system checks connection status:

```
Backend checks connection for a user + source_key:

  1. Is source_key in ENABLED_DATA_SOURCES? → if NO, source unavailable

  2. Look up user_data_source_connections table:
       SELECT * FROM user_data_source_connections
       WHERE user_id = $1 AND source_key = $2

  3. IF no row exists → source NOT connected → render "Connect [Provider]"

  4. IF row exists with composio_account_id:
       → Call Composio SDK: composio_client.connected_accounts.get(composio_account_id)
       → Check status:
           ACTIVE    → source is ready, can fetch data
           EXPIRED   → token refresh failed → show "Reconnect [Provider]"
           FAILED    → auth failed → show "Reconnect [Provider]"
           INACTIVE  → disabled → show "Reconnect [Provider]"

  5. Cache the status check result briefly (e.g., 5 minutes) to avoid
     hitting Composio on every render
```

### Data Fetching via Composio SDK

When the agent (during orchestration) or the consent UI (during draft generation) needs data from a connected source:

```
Caller requests data: fetch_from_source(user_id, source_key, action, params)
  │
  ├── Check: source_key in ENABLED_DATA_SOURCES? → YES
  │
  ├── Look up user's connection in Supabase:
  │     → composio_account_id: "ca_xyz789"
  │
  ├── Execute via Composio Python SDK:
  │     composio_client.tools.execute(
  │         action="GMAIL_FETCH_EMAILS",
  │         params={"query": "from:john after:2026-04-01", "max_results": 10},
  │         entity_id=user_id,
  │         connected_account_id="ca_xyz789"
  │     )
  │
  ├── Composio:
  │     → Retrieves encrypted token from vault
  │     → Injects token into Gmail API call
  │     → Executes request against Gmail API
  │     → Returns result (emails, events, contacts, etc.)
  │
  └── Result returned to caller (agent or consent UI)
```

### How It Integrates with the Consent Flow

The consent flow doc (Section 4.2) already defines the rendering logic for `missing_data_sources`. This component changes **what happens behind the scenes** when the user taps "Connect [Provider]":

```
BEFORE (current — Google only, custom OAuth):
  missing_data_sources includes "google_calendar"
  → UI shows: "Connect your Google account"
  → On tap: GoogleAuthService.signInToGoogle()
  → On success: GoogleAccountManager.linkAccount()
  → Fetch: GoogleAccountManager.getCalendarEvents()

AFTER (with this component — any enabled provider via Composio):
  missing_data_sources includes "gmail" or "outlook" or "slack" or ...
  → Backend checks: is source_key in ENABLED_DATA_SOURCES?
  → YES → render "Connect [Provider]" button
  → On tap: POST /api/data-sources/connect { source_key }
  → FastAPI initiates Composio OAuth → returns redirect URL
  → React Native opens URL → user grants permission → callback → saved
  → UI re-fetches, data is now available, draft regenerated
```

### How It Integrates with the Chat UI (Agent Flow)

During agent orchestration, when the agent needs data from a third-party source:

```
Agent orchestration starts
  │
  ├── Agent determines: "I need the user's recent emails to answer this question"
  │
  ├── Agent checks: is "gmail" enabled AND connected for this user?
  │     → Calls: check_connection(user_id, "gmail")
  │     → Returns: { connected: True, status: "active" }
  │                 OR { connected: False }
  │                 OR { enabled: False }  (not in ENABLED_DATA_SOURCES)
  │
  ├── IF connected:
  │     → Agent calls: fetch_from_source(user_id, "gmail", "GMAIL_FETCH_EMAILS",
  │                      { "query": "from:john subject:meeting" })
  │     → Gets email data → uses it to answer user's question
  │
  ├── IF NOT connected (but enabled):
  │     → Agent responds to user with an inline connect prompt:
  │       "I need access to your Gmail to find that email.
  │        [Connect Gmail]"
  │     → User taps "Connect Gmail" → same OAuth flow as above
  │     → After connection, agent retries the data fetch
  │
  ├── IF NOT enabled:
  │     → Agent responds: "I don't have access to that data source.
  │        Please provide the information manually."
  │
  └── Agent continues orchestration with fetched data
```

### Example Flows

**Consent Screen — Missing Gmail:**
```
Caller asks owner about an email from last week.
Channel agent analyzes: data_sources: ["gmail"], missing_data_sources: ["gmail"]
  → Consent request sent to owner
  → Owner opens consent screen
  → Backend checks: "gmail" in ENABLED_DATA_SOURCES? YES. Connected? NO.
  → Banner: "Connect your Gmail to auto-prepare your response" [Connect Gmail]
  → Owner taps "Connect Gmail"
  → OAuth flow opens in browser → "Allow Trybo to access your Gmail?" → grants
  → UI removes banner, fetches recent emails via Composio SDK
  → Draft generated: "Based on your emails, here's what John sent last week..."
  → Owner reviews, edits, sends
```

**Chat UI — Agent Needs Calendar:**
```
User asks in chat: "What's on my Outlook calendar this Thursday?"
  → Agent checks: "outlookcalendar" in ENABLED_DATA_SOURCES? YES. Connected? YES.
  → Agent calls: fetch_from_source(user_id, "outlookcalendar",
                   "OUTLOOKCALENDAR_LIST_EVENTS",
                   { "timeMin": "2026-04-09", "timeMax": "2026-04-10" })
  → Composio fetches Outlook calendar events
  → Agent responds: "You have 3 events on Thursday: ..."
```

**Chat UI — Agent Needs Slack, Not Connected:**
```
User asks: "Find the message John sent in Slack about the deployment"
  → Agent checks: "slack" in ENABLED_DATA_SOURCES? YES. Connected? NO.
  → Agent responds: "I need access to your Slack to search messages.
                      [Connect Slack]"
  → User taps → OAuth flow → "Allow Trybo to access Slack?" → grants
  → Agent retries: fetch_from_source(user_id, "slack",
                     "SLACK_SEARCH_MESSAGES",
                     { "query": "from:john deployment" })
  → Agent responds with the found message
```

**Agent Needs a Source Not in Enabled List:**
```
User asks: "Check my Trello board for overdue tasks"
  → Agent checks: "trello" in ENABLED_DATA_SOURCES? NO.
  → Agent responds: "I don't have access to Trello currently.
     Could you share the overdue tasks manually?"
```

---

## Design Phases

### Phase 1: Composio Python SDK Integration + Database

Set up the Composio Python SDK on the FastAPI backend. Create the connection tracking table in Supabase.

**Why the SDK, not raw HTTP:**
The existing `marketplace_composio.py` uses raw `httpx` calls for tool execution. For this component, the Composio Python SDK (`composio-core`) is the better choice:
- **Connection management**: OAuth initiation, status polling (`wait_for_connection()`), token refresh, disconnect — the SDK handles redirect flows, polling loops, and error states
- **Tool execution**: The SDK provides typed responses, built-in retries, and automatic token injection
- **Future-proofing**: As Composio's API evolves (v3 → v4), the SDK abstracts breaking changes
- The existing `marketplace_composio.py` should be **migrated to the SDK** as part of this work

**Backend Setup:**
- Add `composio-core` to `trybot-api/requirements.txt`
- Create `trybot-api/app/services/data_source_connection_service.py` — wraps SDK calls for initiating connections, checking status, disconnecting, refreshing, executing tools
- Create `trybot-api/app/constants/data_source_registry.py` — the `ENABLED_DATA_SOURCES` list
- Extend `ExecutionContext` in `contracts.py` — add `connected_data_sources` dict

**Database — Supabase migration:**
```sql
CREATE TABLE user_data_source_connections (
    id                      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id                 uuid NOT NULL REFERENCES user_profiles(id) ON DELETE CASCADE,
    source_key              text NOT NULL,  -- Composio toolkit slug, e.g. "gmail"
    composio_account_id     text NOT NULL,  -- e.g. "ca_xyz789"
    status                  text NOT NULL DEFAULT 'active'
                            CHECK (status IN ('active', 'expired', 'disconnected')),
    connected_at            timestamptz NOT NULL DEFAULT now(),
    disconnected_at         timestamptz,
    created_at              timestamptz NOT NULL DEFAULT now(),
    updated_at              timestamptz NOT NULL DEFAULT now(),

    UNIQUE(user_id, source_key)
);

ALTER TABLE user_data_source_connections ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users_own_connections" ON user_data_source_connections
    USING (user_id = auth.uid());
```

---

### Phase 2: Connection Endpoints (FastAPI Routes)

Create `trybot-api/app/routes/data_sources.py`. Wire into FastAPI app in `main.py`.

```python
# trybot-api/app/routes/data_sources.py

@router.post("/api/data-sources/connect")
async def connect_source(source_key: str, user=Depends(get_current_user)):
    """Initiate OAuth connection. Returns redirect_url for React Native to open."""
    # 1. Check source_key in ENABLED_DATA_SOURCES
    # 2. composio_client.connected_accounts.initiate(...)
    # 3. Return { "redirect_url": ... }

@router.get("/api/data-sources/callback")
async def oauth_callback(connected_account_id: str, status: str):
    """Handle callback from Composio after OAuth completes."""
    # 1. Verify via composio_client.connected_accounts.get(...)
    # 2. Save to Supabase
    # 3. Redirect to deep link: trybo://data-sources/connected?...

@router.get("/api/data-sources/connections")
async def list_connections(user=Depends(get_current_user)):
    """List all connected sources for the authenticated user."""

@router.get("/api/data-sources/connections/{source_key}/status")
async def check_connection(source_key: str, user=Depends(get_current_user)):
    """Check if a specific source is connected and active."""

@router.delete("/api/data-sources/connections/{source_key}")
async def disconnect_source(source_key: str, user=Depends(get_current_user)):
    """Disconnect a source. Revokes tokens via Composio SDK."""

@router.get("/api/data-sources/enabled")
async def list_enabled_sources():
    """List all enabled data sources with metadata from Composio."""
    # Returns ENABLED_DATA_SOURCES with labels/icons queried from Composio SDK
```

---

### Phase 3: Data Fetch Service + Migrate Existing Composio Adapter

Build the service that executes Composio tool calls to fetch user data. Migrate existing `marketplace_composio.py` from raw `httpx` to the SDK.

**Service:**

```python
# trybot-api/app/services/data_source_fetch_service.py

from composio import Composio

composio_client = Composio(api_key=settings.COMPOSIO_API_KEY)

async def fetch_from_source(
    user_id: str,
    source_key: str,
    action: str,         # Composio action slug, e.g. "GMAIL_FETCH_EMAILS"
    params: dict,
    db: Client
) -> dict:
    """Fetch data from a connected third-party source via Composio SDK."""
    # 1. Check source_key in ENABLED_DATA_SOURCES
    # 2. Look up user's connection in Supabase → get composio_account_id
    # 3. composio_client.tools.execute(action=action, params=params, ...)
    # 4. Return structured result

async def check_connection(user_id: str, source_key: str, db: Client) -> dict:
    """Check if a source is enabled, connected, and active."""
    # 1. Check ENABLED_DATA_SOURCES
    # 2. Check Supabase
    # 3. Verify with Composio SDK (cached 5 min)

async def get_available_sources(user_id: str, db: Client) -> list[dict]:
    """All enabled sources with connection status for this user."""
    # For each slug in ENABLED_DATA_SOURCES:
    #   → query Composio SDK for label/icon
    #   → query Supabase for connection status
```

**Migrate `marketplace_composio.py`:**
- Replace `httpx.AsyncClient` calls with `composio_client.tools.execute()`
- Keep `_SKILL_TO_COMPOSIO` mapping — just change the execution method
- Unifies all Composio interactions under one SDK

---

### Phase 4: Google OAuth Migration

Migrate the existing Google integration from custom OAuth to Composio. This eliminates custom token management, encryption, and refresh logic.

**What gets replaced:**

| Current (custom) | Replaced by (Composio) |
|---|---|
| `google_service.py` — encrypt_and_store_tokens, refresh_tokens | Composio SDK handles token lifecycle |
| `google_api_client.py` — direct Google API calls | `composio_client.tools.execute("GOOGLECALENDAR_LIST_EVENTS")` |
| `google_user_profile_service.py` — token CRUD in Supabase | `user_data_source_connections` table (just stores composio_account_id) |
| `token_encryption.py` — AES encryption for Google tokens | Not needed — Composio encrypts tokens in their vault |
| `GoogleAuthService.ts` — React Native Google Sign-In for data access | `Linking.openURL(redirect_url)` — generic OAuth via Composio |
| `GoogleAccountManager.ts` — account linking, token refresh | Not needed — Composio manages everything |
| `routes/google.py` — custom OAuth endpoints | `routes/data_sources.py` — generic for all providers |

**Feature flag:** `USE_COMPOSIO_FOR_GOOGLE` in app config. Run both paths in parallel during testing. Once verified on Android + iOS, remove old code.

**Note:** Google Sign-In for **authentication** (login) remains unchanged — only Google API access for **data** (calendar, contacts, drive) is migrated to Composio.

---

### Phase 5: Frontend Integration (React Native — Chat UI + Consent Screen)

Wire the React Native frontend to use the new connection endpoints and render connect buttons.

**Consent Screen Changes:**
- The existing rendering logic (Section 4.2 of consent flow doc) checks `grant_method` for each missing source
- For sources in `ENABLED_DATA_SOURCES` with `grant_method == "composio_oauth"`:
  - Show banner: "Connect your [label] to auto-prepare your response"
  - Button: "Connect [label]" (label from Composio SDK metadata)
  - On tap: call `POST /api/data-sources/connect { source_key }`
  - Open redirect URL via `Linking.openURL()`
  - On deep link callback: re-render, fetch data, regenerate draft

**Chat UI Changes:**
- When agent responds with a connect prompt (source not connected):
  - Render an inline connect button with the provider icon and label
  - On tap: same OAuth flow as consent screen
  - On success: agent automatically retries the data fetch

**Data Sources Management Screen (new):**
- Accessible from app settings
- Lists all sources from `GET /api/data-sources/enabled` with connection status
- Connected sources: show "Connected on [date]" + "Disconnect" button
- Not connected: show "Connect" button
- Expired: show "Reconnect" button

---

### Phase 6: Agent Awareness + LangGraph Planner Prompt

Update the agent's planner prompt and LangGraph orchestration to be aware of connected data sources.

**Planner Prompt Update:**
- Include a section listing the user's connected data sources:
  ```
  Connected Data Sources:
  - Gmail (connected) — can search/read emails
  - Google Calendar (connected) — can list/read events
  - Slack (not connected but available — user can connect)
  ```
- Include instructions: "If you need data from a connected source, use the data_source_fetch skill. If the source is available but not connected, ask the user to connect it. If the source is not available, ask the user to provide the information manually."

**New Skill: `data_source_fetch`:**
- Registered in `seed_registry.py`
- Input: `{ source_key, action, params }`
- Adapter: calls `fetch_from_source()` from `data_source_fetch_service.py`
- Output: structured data from the source

**Orchestration Flow:**
- Before orchestration starts, the system populates `connected_data_sources` in `ExecutionContext`
- The agent sees which sources are available and decides whether to fetch data
- If the agent needs a source that isn't connected, it includes a connect prompt in its response

---

## What Exists Today vs. What's New

| Capability | Exists Today | What's New |
|---|---|---|
| Device source permissions (calendar, contacts) | Yes — `trybot-ui/services/permissionService.ts` | No change needed |
| Google OAuth (backend) | Yes — `google_service.py`, `google_api_client.py`, `google_user_profile_service.py` — custom token encrypt/store/refresh | Migrate to Composio SDK, then remove custom code |
| Google OAuth (frontend) | Yes — `GoogleAuthService.ts`, `GoogleAccountManager.ts` — React Native Google Sign-In | Replace with generic `Linking.openURL()` for Composio OAuth |
| Composio tool execution | Partial — `marketplace_composio.py` uses raw `httpx` for Slack, Notion, Linear, GitHub | Migrate to Composio Python SDK. Extend to all data source tools. |
| Enabled data sources list | No | New: `ENABLED_DATA_SOURCES` in `constants/data_source_registry.py` |
| Third-party OAuth management | No — only Google, manually managed | New: Composio SDK manages all OAuth flows, tokens, refresh |
| Connection status tracking | No | New: `user_data_source_connections` table in Supabase |
| Data fetching from third-party sources | Partial — only Google Calendar via `google_api_client.py` | New: generic `fetch_from_source()` via Composio SDK |
| Multi-provider support | No — only Google | New: any provider in ENABLED_DATA_SOURCES |
| Agent data source awareness | No | New: planner prompt includes connected sources, `data_source_fetch` skill in LangGraph |
| Data sources management screen | No | New: React Native settings screen to manage connections |
| Consent screen connect banners | Partial — "Connect Google" only | Extended: "Connect [any enabled provider]" |

---

## Implementation Priority & Time Estimates

All implementation is done using Claude. Claude writes all code and unit tests. Human time is for device testing, verification, and Composio dashboard configuration. This component spans **backend (Python/FastAPI) + frontend (React Native)**.

**Working day = 8 hours**

### Phase 1: Composio Python SDK + Database
*Dependencies: Composio account created, API key obtained*

| Task | Hours |
|------|-------|
| Add `composio-core` to `trybot-api/requirements.txt`, verify SDK import and initialization | 0.5 |
| Create `data_source_connection_service.py` — SDK wrapper (initiate, check, execute, disconnect, refresh) | 2 |
| Create `constants/data_source_registry.py` — `ENABLED_DATA_SOURCES` list (initially empty) | 0.25 |
| Create Supabase migration: `user_data_source_connections` table with RLS policies | 1 |
| Unit tests for data_source_connection_service (mock Composio SDK) | 1 |
| **Phase 1 Total** | **4.75** |

### Phase 2: FastAPI Connection Endpoints
*Dependencies: P1*

| Task | Hours |
|------|-------|
| Create `routes/data_sources.py` + wire into `main.py` | 0.5 |
| `POST /connect` — initiate OAuth via Composio SDK, return redirect URL | 1 |
| `GET /callback` — handle OAuth callback, save to Supabase, redirect to deep link | 1.5 |
| `GET /connections` — list user's connected sources from Supabase | 0.5 |
| `GET /connections/{source_key}/status` — check single source (with 5-min cache) | 0.5 |
| `DELETE /connections/{source_key}` — disconnect via SDK + update Supabase | 0.5 |
| `GET /enabled` — list enabled sources with metadata from Composio SDK | 0.5 |
| Unit tests for all endpoints | 1.5 |
| **Phase 2 Total** | **6.5** |

### Phase 3: Data Fetch Service + Migrate marketplace_composio.py
*Dependencies: P1, P2*

| Task | Hours |
|------|-------|
| Implement `data_source_fetch_service.py` — generic `fetch_from_source()` via Composio SDK | 1.5 |
| Implement `check_connection()` with caching (5-min TTL) | 0.5 |
| Implement `get_available_sources()` — enabled sources with status for a user | 0.5 |
| Migrate `marketplace_composio.py` from raw `httpx` to Composio Python SDK | 1 |
| Handle Composio errors: rate limits, expired tokens, provider API errors | 1 |
| Unit tests for fetch service + migrated adapter | 1.5 |
| **Phase 3 Total** | **6.5** |

### Phase 4: Google OAuth Migration
*Dependencies: P1, P2, P3. Requires "googlecalendar" custom OAuth configured in Composio dashboard.*

| Task | Hours |
|------|-------|
| Add `USE_COMPOSIO_FOR_GOOGLE` feature flag to app config | 0.25 |
| Wire Google Calendar fetch through Composio SDK (behind feature flag) | 1 |
| Wire Google Contacts fetch through Composio SDK (behind feature flag) | 0.5 |
| Test on device: Google Calendar via Composio vs existing (verify parity) | 1 |
| Remove old code after verification: `google_service.py`, `google_api_client.py`, `google_user_profile_service.py`, `token_encryption.py` (Google parts), `GoogleAuthService.ts`, `GoogleAccountManager.ts` | 1 |
| Unit tests for migrated Google paths | 0.5 |
| **Phase 4 Total** | **4.25** |

### Phase 5: React Native Integration (Chat UI + Consent Screen + Settings)
*Dependencies: P2, P3*

| Task | Hours |
|------|-------|
| Create `trybot-ui/services/dataSourceService.ts` — API client for `/api/data-sources/*` | 1 |
| Consent screen: render "Connect [Provider]" banner for missing Composio-backed sources | 1 |
| Consent screen: OAuth flow via `Linking.openURL()` + deep link callback handler | 2 |
| Consent screen: re-fetch data + regenerate draft after connection | 1 |
| Chat UI: inline "Connect [Provider]" button component | 1.5 |
| Chat UI: OAuth flow trigger + success callback + agent retry | 1.5 |
| Data sources management screen (settings): list, connect, disconnect | 2 |
| Android build + on-device testing (consent screen + chat UI + settings flows) | 1.5 |
| iOS build + on-device testing (same flows) | 1.5 |
| Platform-specific bug fixes (deep links, system browser OAuth) | 1.5 |
| **Phase 5 Total** | **15** |

### Phase 6: LangGraph Agent Awareness + Planner Prompt
*Dependencies: P3*

| Task | Hours |
|------|-------|
| Add `connected_data_sources` to `ExecutionContext` in `contracts.py` | 0.5 |
| Populate `connected_data_sources` before orchestration starts | 0.5 |
| Update planner prompt: list connected sources + usage instructions | 1 |
| Register `data_source_fetch` skill in `seed_registry.py` + adapter | 1.5 |
| Agent generates inline connect prompt when source is available but not connected | 1 |
| Integration test: agent discovers connected Gmail → fetches emails → uses in response | 1.5 |
| Integration test: agent detects missing source → prompts user → user connects → agent retries | 1 |
| **Phase 6 Total** | **7** |

### End-to-End Device Testing
*After all phases complete*

| Task | Hours |
|------|-------|
| Android E2E: connect Gmail → agent fetches emails → uses in response | 1 |
| Android E2E: consent screen → missing Outlook → connect → draft generated | 1 |
| Android E2E: settings screen → connect/disconnect multiple sources | 0.5 |
| iOS E2E: same test matrix as Android | 2 |
| Edge cases: expired token → reconnect, Composio down → graceful error, disconnect mid-session | 1.5 |
| Cross-flow: agent needs Gmail (chat) → user connects → consent request arrives → uses same connection | 1 |
| Bug fixes from testing (Claude fixes, human re-verifies) | 2 |
| **E2E Testing Total** | **9** |

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Composio as unified integration layer** | Purpose-built for AI agents. 982 toolkits, managed OAuth, per-user isolation. Eliminates the need to build and maintain individual OAuth integrations for each provider. |
| **Composio Python SDK, not raw HTTP** | The existing `marketplace_composio.py` uses raw `httpx` calls. The SDK provides built-in retry logic, typed responses, `wait_for_connection()` helpers, and abstracts API version changes. Migrate existing adapter + build new services on the SDK. |
| **Thin enabled list, not large static registry** | `ENABLED_DATA_SOURCES` is just a list of slugs. Labels, icons, available actions come from Composio SDK at runtime. No 200-line config dict to maintain. Adding a new provider = KPR configures custom OAuth + adds one slug. |
| **Only custom-OAuth providers are available** | Every enabled provider has custom OAuth configured so consent screens show "Trybo", not "Composio". We do not fall back to Composio's managed auth. This preserves brand trust in a consumer app. |
| **KPR owns the enabled list** | KPR registers OAuth credentials with providers (Google, Microsoft, Slack) and configures them in Composio dashboard. Then adds the slug to `ENABLED_DATA_SOURCES`. The system automatically picks it up. |
| **Brokered credentials (tokens on Composio, not our DB)** | Our backend never handles raw OAuth tokens. Composio encrypts, stores, and auto-refreshes them. Eliminates `token_encryption.py` and custom token storage. |
| **One connection per source per user** | Simplest model. A user connects one Gmail account, one Outlook account, etc. Multiple accounts per source is a future enhancement. |
| **Migrate all Google from custom OAuth to Composio** | Current Google integration spans 5+ files with custom token encryption, storage, and refresh. Composio replaces all of this. Feature flag during transition. |
| **Real-time fetch, no data syncing** | Data is fetched on demand, not pre-synced. Simpler architecture, no stale data, no storage costs. Acceptable latency for MVP. |
| **Deep link callback for mobile OAuth** | After OAuth in system browser, callback redirects to `trybo://data-sources/connected?...` which React Native intercepts. Standard mobile OAuth pattern. |
| **5-minute connection status cache** | Avoid hitting Composio on every UI render or agent check. Cache invalidated on explicit connect/disconnect. |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Composio vendor dependency** | If Composio goes down, all third-party data fetching stops | For MVP, acceptable — Composio is SOC 2 Type 2 compliant. For production: extract raw tokens (disable credential masking) as fallback. Long-term: evaluate self-hosted enterprise option. |
| **OAuth in mobile system browser** | Some providers restrict OAuth in embedded WebViews | Use system browser (`Linking.openURL()`), not in-app WebView. Standard pattern. Deep link callback returns to app. |
| **Composio rate limits** | High-volume tool execution may hit limits | Monitor usage. Free tier = 20K calls/mo, Starter = 200K. Scale plan as needed. |
| **Token expiry between connection and use** | User connects today, agent uses next week — token may expire | Composio auto-refreshes. If refresh fails, `check_connection()` detects EXPIRED status, prompts reconnect. |
| **Provider API limitations** | Some providers restrict third-party access (e.g., Instagram) | Only enable providers where Composio supports the actions we need. Agent falls back to asking user if action fails. |
| **Multiple Google scopes** | Gmail, Calendar, Contacts, Drive need separate OAuth grants | Each gets its own entry in ENABLED_DATA_SOURCES. User connects each separately. Future: combine into unified "Connect Google" with all scopes. |
| **Google OAuth migration risk** | Replacing existing integration could break current users | Feature flag `USE_COMPOSIO_FOR_GOOGLE`. Existing users re-authenticate once via Composio. Keep old code until verified. |
| **Cost scaling** | Composio charges per tool call | At $0.299/1K calls (Starter), 200K calls/mo = $29/mo. Very affordable for initial scale. Monitor and set per-user limits if needed. |

---

## Relationship to Existing Code

```
Existing (Backend — trybot-api/)          This Component Changes / Adds
────────────────────────────────          ──────────────────────────────

services/google_service.py               REMOVED after Composio migration
services/google_api_client.py            REMOVED — Composio SDK handles API calls
services/google_user_profile_service.py  REMOVED — no more custom token storage
services/token_encryption.py             REMOVED for Google tokens (Composio vault)
routes/google.py                         REPLACED by routes/data_sources.py

agentic_mvp/runtime/adapters/            MIGRATED from raw httpx to Composio
  marketplace_composio.py                Python SDK (same _SKILL_TO_COMPOSIO map)

agentic_mvp/runtime/contracts.py         EXTENDED — add connected_data_sources
  (composio_user_id already exists)      dict to ExecutionContext

                                         + services/data_source_connection_service.py (NEW)
                                         + services/data_source_fetch_service.py (NEW)
                                         + routes/data_sources.py (NEW)
                                         + constants/data_source_registry.py (NEW)
                                         + migration: user_data_source_connections (NEW)
                                         + agentic_mvp skill: data_source_fetch (NEW)

Existing (Frontend — trybot-ui/)          This Component Changes / Adds
────────────────────────────────          ──────────────────────────────

lib/GoogleAuthService.ts                 REMOVED after migration (feature flag)
lib/GoogleAccountManager.ts              REMOVED — Composio handles account linking
services/informationConsentService.ts    UPDATED — composio_oauth handling in
  (consent screen data source banners)   missing_data_sources rendering
services/permissionService.ts            UNCHANGED — device sources stay as-is

                                         + services/dataSourceService.ts (NEW)
                                         + screens/DataSourcesManagement (NEW)
                                         + components/ConnectSourceButton (NEW)
                                         + Deep link handler for OAuth callback (NEW)
```

---

## Total Effort

| Phase | Hours | Days (8h/day) |
|-------|-------|---------------|
| P1 — Composio Python SDK + database | 4.75 | 0.6 |
| P2 — FastAPI connection endpoints | 6.5 | 0.8 |
| P3 — Data fetch service + migrate marketplace_composio.py | 6.5 | 0.8 |
| P4 — Google OAuth migration | 4.25 | 0.5 |
| P5 — React Native integration (chat UI + consent screen + settings) | 15 | 1.9 |
| P6 — LangGraph agent awareness + planner prompt | 7 | 0.9 |
| End-to-end device testing (Android + iOS) | 9 | 1.1 |
| **Total** | **53 hours** | **~6.5 days** |

**Recommended order:** P1 (day 1) → P2 (day 1-2) → P3 (day 2-3) → P4 (day 3, once Google custom OAuth is configured in Composio) → P5 + P6 in parallel (day 3-5) → E2E device testing (day 5-7)

Note: P5 and P6 can overlap since React Native integration and LangGraph agent awareness touch different parts of the codebase. P4 depends on KPR having configured Google custom OAuth in Composio's dashboard. The critical path is: P1 → P2 → P3 → P5 (React Native needs FastAPI endpoints and fetch service ready).
