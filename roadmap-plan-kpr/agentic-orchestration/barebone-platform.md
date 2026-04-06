# Barebone Platform — Baseline Definition

> Owner: AKN
> Scope: Component 1 — 12 subcomponents (1.1–1.12)
> Module mapping: Barebone Platform
> Current completion: ~48% — 52 of 129 capabilities exist, 9 partial, 68 missing
> Source: SUBCOMPONENT_CAPABILITY_MAP.md

---

## What This Is

The Platform is the ground everything else runs on — no intelligence, no opinion. It provides the infrastructure services that every other component depends on: lifecycle management, event distribution, state persistence, authentication, configuration, protocol translation, background processing, notifications, AI generation, and outbound HTTP.

The foundational happy-path for most subcomponents already works. The gap is primarily operational robustness — health reporting, distributed safety, and centralized HTTP management.

## Current State

| What Exists | What's Missing |
|---|---|
| Ordered startup/shutdown, graceful drain | Readiness + per-subsystem health endpoints |
| Event pub/sub with persistence + ordering | Fan-out completion, replay, dead letter queue |
| Checkpoint save/restore (LangGraph) | Distributed locking, durable idempotency |
| Session hot/cold storage with TTL | Distributed session sharing (hot fields in-process) |
| JWT + HMAC auth, RLS, profile caching | Rate limiting, complete token refresh |
| Full config loading, validation, typed API | (baseline met — nothing missing) |
| REST, SSE, WS, webhook normalization | Webhook dedup, consistent validation, security headers |
| Safe CRUD, RLS, files, vectors, migrations | Transaction support (multi-table atomic) |
| Job queue with retry, DLQ, scheduling | (baseline met — nothing missing) |
| Push notifications via FCM | Notification logging, user preferences |
| Provider-agnostic LLM generation | Multi-provider structured output/streaming, fallback, routing |
| (nothing centralized) | Entire HTTP Client module is new |

## Baseline Definition

**The line is:** Every platform service can start reliably, report its health, authenticate requests with rate limiting, persist data atomically, distribute events with fan-out and replay, manage sessions across instances, process background jobs, send notifications with logging, route LLM calls across providers with fallback, and make outbound HTTP calls through a centralized client with retry and circuit breakers. Anything beyond this is refinement.

---

## Baseline Components

### 1.1 System Lifecycle — ~2.5 dev days

**What this is:** The boot/shutdown/health orchestrator for all internal subsystems (DB, Redis, queue, scheduler). It starts everything in the right order, tells Cloud Run when the system is ready, shuts down gracefully on SIGTERM, and recovers orphaned work after a crash.

#### Baseline deliverables

**1. Readiness endpoint (`/ready`)**
Each subsystem registers a health-check callable during init. The `/ready` endpoint returns 200 only when all checks pass. Cloud Run startup probe points here. This is the difference between "container is running" and "container can actually serve traffic."

**2. Per-subsystem health endpoint (`/health`)**
Returns structured JSON: `{db: "ok", redis: "degraded", queue: "ok", scheduler: "ok"}`. Each subsystem has its own check function. Cloud Run and operators use this to understand partial failures rather than just "up or down."

**3. Complete startup/shutdown hook registry**
Extend existing shutdown hooks with symmetric startup hooks. `register_startup_hook(fn)` / `register_shutdown_hook(fn)`. This lets any component plug into the lifecycle without modifying the core lifespan manager.

| Refinement (deferred) | Why |
|---|---|
| Hot-reload config without restart | Config changes are infrequent; restart is acceptable |
| Self-healing with reconnect + backoff | Cloud Run already restarts unhealthy instances |

**Dependencies:** FastAPI lifespan manager (exists), Cloud Run probe configuration (infra)

---

### 1.2 Event Bus — ~3.5 dev days

**What this is:** The internal pub/sub backbone. Components publish events to named channels, and any number of subscribers receive them independently. Events are persisted for audit and replay. This decoupling is what lets components be developed and deployed independently.

#### Baseline deliverables

**1. Complete fan-out to N in-process subscribers**
Extend current SSE-per-consumer to a proper subscriber set per channel. On publish, iterate all registered subscribers and deliver. Critical for components like UI streaming + history + analytics all consuming the same events.

**2. Event replay from sequence number**
`bus.replay(channel="call.*", from_seq=30)` returns events #30 through current. Essential for recovery — a component that crashes and restarts needs to catch up from where it left off.

**3. Dead letter queue**
Events that fail processing after max retries go to `event_dlq` table instead of being silently dropped. The DLQ makes failures inspectable and retryable.

| Refinement (deferred) | Why |
|---|---|
| Schema validation at publish time | Adds friction; trust the publisher at baseline |
| Back-pressure management | Single-instance with modest load doesn't need it |
| Cross-service event delivery | Multi-instance fan-out only matters at scale |
| Event metrics | Observability, not blocking |

**Dependencies:** Postgres event_log table (exists), System Lifecycle hooks (1.1)

---

### 1.3 Checkpoint Store — ~2.5 dev days

**What this is:** Tasks that produce real-world side effects (phone calls, payments) cannot be safely re-run from scratch after a crash. The Checkpoint Store saves in-progress state at defined points so the system can resume from the last successful step. Also tracks idempotency to prevent duplicate actions.

#### Baseline deliverables

**1. Distributed locking on resume**
Redis `SETNX` lock on `checkpoint:{run_id}` with TTL. Without it, two Cloud Run instances can resume the same run simultaneously — duplicate phone calls or double-bookings. This is a correctness issue.

**2. Durable idempotency tracking**
Migrate from in-memory set to Redis `SISMEMBER`. The current in-memory set loses all idempotency tracking on process restart.

**3. Checkpoint metadata**
Add `saved_at`, `schema_version`, `component`, `run_id` columns. Minimum needed for debuggability.

| Refinement (deferred) | Why |
|---|---|
| Auto-expire old checkpoints | Storage is cheap; manual cleanup acceptable |
| Checkpoint versioning/migration | Schema changes are rare |
| Partial checkpoints, compression, audit | Optimizations, not correctness |

**Dependencies:** Redis (1.1), LangGraph checkpoint backend (exists)

---

### 1.4 Session Manager — ~3 dev days

**What this is:** Real-time interactions (voice calls, live chat) generate ephemeral state at high frequency. The Session Manager provides a two-tier store: in-memory for sub-millisecond reads, Redis for durability across process boundaries. Sessions auto-expire after TTL.

#### Baseline deliverables

**1. Migrate hot fields to Redis**
The 26 "hot fields" currently live in a Python dict — invisible to other instances. Migrate to Redis hash fields. Trades microsecond reads for millisecond reads (still fast enough for voice). Without this, sticky routing is required, which breaks Cloud Run's load balancing.

**2. Session lifecycle events**
Publish `session.created`, `session.updated`, `session.expired` to Event Bus. Redis keyspace notifications catch TTL expiry. Other components need to know when sessions start/end without polling.

| Refinement (deferred) | Why |
|---|---|
| Session size limits with overflow | Sessions bounded by call duration |
| Optimistic locking | Race conditions rare with sequential turns |
| Session migration between servers | Redis-backed sessions are already portable |

**Dependencies:** Redis (1.1), Event Bus (1.2)

---

### 1.5 Auth Gateway — ~2.5 dev days

**What this is:** Every inbound request must be authenticated. The Auth Gateway extracts identity from JWT tokens, resolves it to an internal user profile (with caching), and provides a Row-Level Security scoped database client.

#### Baseline deliverables

**1. Rate limiting per user**
Redis sliding window counter. Return 429 when exceeded. Without rate limiting, a single user or bot can exhaust backend resources for everyone.

**2. Complete token refresh cycle**
Extend existing Google OAuth refresh to a generic pattern: detect 401 → attempt refresh → retry. The current partial implementation breaks long-running sessions.

| Refinement (deferred) | Why |
|---|---|
| API key authentication | No external API consumers at baseline |
| RBAC | Single-role (user) sufficient |
| Auth audit trail | Not blocking function |
| Session revocation | Edge case; manual expiry covers it |

**Dependencies:** Redis (1.1), Config Registry (1.6)

---

### 1.6 Config Registry — 0 dev days (BASELINE MET)

**What this is:** Loads, validates (Pydantic), and exposes all configuration through a single typed API. Separates secrets from non-secret config. Selects providers per capability.

All 6 baseline capabilities already exist: load + validate, environment defaults, feature flags, provider selection, secret separation, typed API.

| Refinement (deferred) | Why |
|---|---|
| Hot-reload without restart | Config changes are infrequent |
| Feature flag targeting (per user/org/%) | Boolean flags sufficient |
| Config versioning and rollback | Versioned via git |
| Secret rotation without downtime | Brief downtime acceptable |

---

### 1.7 Protocol Bridge — ~3.5 dev days

**What this is:** Translates between external protocol/payload formats (Twilio, Meta, REST, SSE, WebSocket) and internal representations at the system boundary.

#### Baseline deliverables

**1. Webhook replay protection**
Redis `SETNX webhook:{provider}:{webhook_id}` with TTL. Without dedup, retried webhooks process the same event multiple times — duplicate calls or messages.

**2. Consistent request validation on all routes**
Apply `@validate_request` with Pydantic models to all route handlers. Currently only some routes validate.

**3. Complete security headers**
`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`. Standard security hygiene.

| Refinement (deferred) | Why |
|---|---|
| API versioning (/v1, /v2) | No external API consumers yet |
| Gateway-level rate limiting | Auth Gateway handles per-user |
| Protocol negotiation | Clients know which transport to use |

**Dependencies:** Redis (1.1)

---

### 1.8 Data Store — ~2 dev days

**What this is:** The single persistence layer. Safe CRUD with RLS-enforced user isolation, service-role bypass, connection pooling, file/object storage, vector search, and schema migrations.

#### Baseline deliverables

**1. Transaction support for multi-table atomic operations**
Supabase `rpc()` calling Postgres functions with `BEGIN/COMMIT/ROLLBACK`. Several workflows must succeed or fail atomically. Without transactions, partial failures leave inconsistent states.

| Refinement (deferred) | Why |
|---|---|
| Query performance monitoring | Supabase dashboard covers basics |
| Soft delete | Hard delete acceptable |
| Data archival | Tables small enough at current scale |
| Enhanced backup | Supabase daily backups suffice |

**Dependencies:** Supabase infrastructure (exists)

---

### 1.9 Job Queue — 0 dev days (BASELINE MET)

**What this is:** Accepts work immediately, processes asynchronously via background workers with retry and dead-letter handling. Jobs are typed, scheduled, and processed in parallel.

All 7 baseline capabilities already exist: enqueue, worker with at-least-once delivery, retry with backoff, DLQ, parallel processing, scheduling, handler registry.

| Refinement (deferred) | Why |
|---|---|
| Priority levels | FIFO sufficient at current volume |
| Job cancellation | Uncommon; let job run and ignore |
| Progress tracking | UI convenience, not backend correctness |
| Rate limiting per type | Provider-level handled by HTTP Client |
| Job dependencies | LangGraph handles sequencing |

---

### 1.10 Notification Service — ~2.5 dev days

**What this is:** Handles backend-to-device communication — reaching the user's phone when the app is in background or closed. Sends typed push notifications via FCM with per-device token management.

#### Baseline deliverables

**1. Notification history logging**
`notification_log` table. Write on every send. Without logs, you cannot debug "I didn't get a notification" — the most common push notification support issue.

**2. User notification preferences per category**
`notification_preferences` table with per-category toggles. Without preferences, users who find notifications annoying disable them entirely at the OS level.

| Refinement (deferred) | Why |
|---|---|
| Multi-channel (email, SMS, in-app) | Push is primary; others need providers |
| Templating | Hardcoded strings acceptable at current volume |
| Delivery tracking, batch, scheduling, throttling | Low volume at launch |

**Dependencies:** Firebase Admin SDK (exists), Data Store (1.8)

---

### 1.11 LLM Gateway — ~5.5 dev days

**What this is:** Single provider-agnostic interface for all AI generation. Provider selection via config. Changing LLM providers is a config change, not a code change across 15 services.

#### Baseline deliverables

**1. Complete structured output across all providers**
Extend JSON mode from OpenAI-only to Gemini and Ollama. Without this, switching providers breaks every service using structured output.

**2. Complete streaming across all providers**
Extend streaming from OpenAI-only to Gemini and Ollama. Same issue.

**3. Provider fallback chain**
Ordered list `[openai, gemini, ollama]`. On failure → try next → log failover. Provider outages are common. Without fallback, a provider outage = total system outage.

**4. Multi-model routing by task type**
`{classification: "gpt-4o-mini", planning: "gpt-4o"}`. Using the expensive model for everything wastes money; cheap model for everything produces bad plans.

| Refinement (deferred) | Why |
|---|---|
| Token budget enforcement | Costs monitored manually |
| Prompt caching | Optimization; correct before fast |
| Response validation + retry | Trust model at baseline |
| Cost tracking, provider rate limiting | Manual review sufficient |

**Dependencies:** Config Registry (1.6)

---

### 1.12 HTTP Client — ~8.5 dev days

**What this is:** 22 services make outbound HTTP calls to 15+ external APIs, each managing its own connection/timeout/retry independently. The HTTP Client centralizes this with shared connection pools, configurable timeouts, retry, and circuit breakers. Entirely new module.

#### Baseline deliverables

**1. Provider registry** — Dict from Config: `{provider_name: {base_url, auth_header, timeout, max_retries}}`. Currently hardcoded in 22 files.

**2. Connection pooling per provider** — `httpx.AsyncClient` singleton per provider. Without pooling, every call opens new TCP + TLS handshake (100-300ms overhead).

**3. Configurable timeouts** — Default from Config with per-provider overrides. Without this, slow providers block workers indefinitely.

**4. Retry with exponential backoff** — `tenacity`: retry on 5xx, stop after 3 attempts, wait 1s/2s/4s. Without retry, transient errors become user-visible failures.

**5. Circuit breaker per provider** — `pybreaker` with fail_max=5, reset_timeout=60. Without circuit breakers, a down provider causes cascading timeouts.

**6. Unified API** — `http_client.post(provider="twilio", path="/Calls", json={...})`. One interface instead of 22 patterns.

| Refinement (deferred) | Why |
|---|---|
| Request/response logging | Nice but not blocking |
| Rate limiting per provider | Providers enforce their own |
| Response caching for GETs | Most calls are POSTs |
| Metrics | Observability, not functional |

**Dependencies:** Config Registry (1.6), System Lifecycle (1.1)

---

## What Counts as Refinement (explicitly deferred)

All capabilities not listed as baseline deliverables above are refinements. The major deferred categories:

| Category | Examples | Why deferred |
|---|---|---|
| Observability | Event metrics, query monitoring, latency dashboards | Function before measurement |
| Scale | Cross-service events, back-pressure, multi-instance | Single instance sufficient at launch |
| Operational maturity | Config hot-reload, self-healing, secret rotation | Manual processes acceptable at early stage |
| Advanced access control | RBAC, API keys, audit trails | Single-role user base |

## Dependencies

| Needs | From | Why |
|---|---|---|
| Redis | Infrastructure | Locking, rate limiting, caching, sessions |
| Supabase (Postgres + Storage) | Infrastructure | All persistence |
| GCP Cloud Run | Infrastructure | Container hosting, scaling |
| Firebase | Infrastructure | Push notifications, analytics |
| Stripe, Twilio, Meta APIs | External | Provider integrations |

## Effort Estimate

| Module | Effort |
|---|---|
| 1.1 System Lifecycle | 2.5 days |
| 1.2 Event Bus | 3.5 days |
| 1.3 Checkpoint Store | 2.5 days |
| 1.4 Session Manager | 3 days |
| 1.5 Auth Gateway | 2.5 days |
| 1.6 Config Registry | 0 days (baseline met) |
| 1.7 Protocol Bridge | 3.5 days |
| 1.8 Data Store | 2 days |
| 1.9 Job Queue | 0 days (baseline met) |
| 1.10 Notification Service | 2.5 days |
| 1.11 LLM Gateway | 5.5 days |
| 1.12 HTTP Client | 8.5 days |
| **Total** | **~36 dev days** |
