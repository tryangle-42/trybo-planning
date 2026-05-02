# C1 — Barebone Platform: Long-Term Architecture Plan

> **Jira:** TB-705 (parent: TB-704 Arch Reviews)
> **Owner:** AKN
> **Layer:** L1 — Platform Foundation
> **Sibling component:** C2 — Execution Tools (TB-706)
> **Status:** Solutioning — pre-implementation pre-review
> **Last updated:** 2026-04-25

---

## 0. Purpose of this document

This is the **architecture solutioning doc** for L1 / C1 (Barebone Platform) in the format the April 5 meeting laid out: responsibilities, data flows, internal sub-components, key contracts, integration points, dependencies, current vs. target state, phased roadmap, open questions. **Not a full LLD.** Enough to expose dependencies, allow ticket creation, and give rough estimates.

The lens is **long-term maturity** — what this surface looks like at the end-state, not what ships for beta. Baseline-vs-mature deltas are noted per surface so the team can derive a near-term cut without losing the long-term frame.

This supersedes the framing in `roadmap-plan-kpr/agentic-orchestration/barebone-platform.md` (which was scoped as "production baseline + deferred refinements"). Some surfaces from that draft moved to **C2 / Execution Tools** (LLM Gateway, Notification Service) — see §3.

---

## 1. The C1 / C2 dividing line, in one sentence

**C1 (Barebone Platform)** = anything you'd need to run a backend at all, even with zero agents and zero skills. Inward-facing infrastructure primitives.

**C2 (Execution Tools)** = every uniform-contract interface to something *outside the agent's brain* — channels, third-party APIs, MCPs, sub-agents, and LLMs-as-tools.

The clean test for C1: *would this exist even if Trybo had no LLMs, no skills, no planner?* If yes → C1.

---

## 2. Mission

Provide the **shared, domain-agnostic substrate** that every other layer depends on, so that L2/L3/L4/L5 components — and every skill — can assume a stable, user-scoped, observable, recoverable runtime without each one re-implementing core infrastructure.

Two non-negotiables that flow from this:

1. **Nothing domain-specific lives in C1.** If a table, API, or event type exists for one feature, it belongs to that feature's layer, not here.
2. **Every C1 surface is a primitive, not a policy.** C1 owns the *machinery*; the *rules* (consent, retention, autonomy) live in their respective layers.

---

## 3. What moved out of C1 (vs. KPR's earlier draft)

| Surface | Where in this plan | Why |
|---|---|---|
| **LLM Gateway** | C2 (Execution Tools) | Calling an LLM is an outbound action against an external system, not a generic primitive. Structurally identical to calling Tavily or Twilio. |
| **Notification Service** (FCM/APNs send) | C2 (Execution Tools) — channel adapter | "Send a push to user X" is a channel action. C1 keeps the *push-token registry* (data) and the *outbound HTTP transport* (primitive). The send-action moves to C2. |
| **Specific provider channels** (Twilio, Meta, LiveKit, etc.) | C2 entirely | Were never in C1; calling out for clarity. |

What stays in C1: the *enabling primitives* those things sit on (HTTP client, config, secrets, lifecycle, persistence, observability, etc.).

---

## 4. The 12 surfaces of C1

Each surface is a long-term architectural commitment. Each gets a sub-section in §6 with full scope.

| # | Surface | One-liner |
|---|---------|-----------|
| **C1.1** | Process & Service Lifecycle | Boot, drain, health, deployment safety |
| **C1.2** | Persistence Substrate | RLS-safe DB, transactions, migrations, files, vectors, archival, DR |
| **C1.3** | Event Bus & Streaming Spine | Pub/sub, fan-out, replay, DLQ, SSE/WS distribution |
| **C1.4** | State & Checkpoint Store | Distributed locks, durable idempotency, checkpoint versioning |
| **C1.5** | Session / Working-State Layer | Two-tier (in-mem + Redis) ephemeral state for live interactions |
| **C1.6** | Identity & Access | Auth, user-scoped session handling, rate limits, cost ceilings (tenancy / RBAC deferred — see C1.6 open questions) |
| **C1.7** | Configuration & Feature Flags | Typed config, secrets, hot-reload, %/cohort flags |
| **C1.8** | Outbound HTTP Substrate | Centralized client, pooling, retries, breakers, signing |
| **C1.9** | Inbound Protocol Bridge | REST + SSE + WS + webhook normalization, dedup, sigverify |
| **C1.10** | Background Work & Scheduling | Job queue, cron, retries, DLQ, cancellation |
| **C1.11** | Observability & Tracing Spine | Logs, OTel traces, metrics, generic audit log |
| **C1.12** | Compliance & Data-Lifecycle Primitives | Retention, GDPR/DPDP rights pipelines, encryption helpers, PII redaction utility |

---

## 5. How C1 connects to the rest of the system

```
                          ┌──────────────────────────────────────┐
                          │  L5 Product (UI/UX)                  │
                          └─────────────┬────────────────────────┘
                                        │ SSE/WS over C1.3 + C1.9
                          ┌─────────────▼────────────────────────┐
                          │  L4 Intelligence & Memory             │
                          └─────────────┬────────────────────────┘
                                        │ C1.2 (vector + RLS)
                                        │ C1.3 (memory.* events)
                          ┌─────────────▼────────────────────────┐
                          │  L3 Capability System (Skill Map)     │
                          └─────────────┬────────────────────────┘
                                        │ C1.2 (registry tables)
                                        │ C1.7 (skill config flags)
                          ┌─────────────▼────────────────────────┐
                          │  L2 Agent Runtime Brain (Planner +    │
                          │  Execution Engine)                    │
                          └─────────────┬────────────────────────┘
                                        │ C1.4 (checkpoints)
                                        │ C1.3 (run.* events)
                                        │ C1.5 (live session state)
                                        │ C1.10 (scheduled triggers)
        ┌───────────────────────────────▼──────────────────────────────┐
        │                C2 — Execution Tools                          │
        │ uses: C1.8 (outbound HTTP), C1.9 (inbound webhook            │
        │ normalization), C1.11 (per-tool spans)                       │
        └───────────────────────────────┬──────────────────────────────┘
                                        │
        ┌───────────────────────────────▼──────────────────────────────┐
        │  C1 — Barebone Platform (this doc)                           │
        │  Lifecycle | Persistence | Event Bus | Checkpoints | Sessions│
        │  Identity  | Config      | HTTP Out  | Protocol    | Jobs   │
        │  Observability           | Compliance Primitives             │
        └──────────────────────────────────────────────────────────────┘
                                        │
                          ┌─────────────▼────────────────────────┐
                          │  Infrastructure                      │
                          │  Cloud Run · Supabase · Redis · OTel │
                          │  Collector · KMS · Pub/Sub           │
                          └──────────────────────────────────────┘
```

**Rule of thumb for routing tickets:** if it's a *primitive* needed by ≥3 layers above, it lives in C1. If it's a *capability* invocable as a tool, it lives in C2. If it's a *policy* about who/when/why, it lives in the relevant L (e.g. autonomy in L4, consent rules in M11/Governance).

---

## 6. The 12 surfaces in detail

For each surface: **Mission** → **Sub-components** → **Key contracts (DB / API / events)** → **Integration points** → **Current state** → **Long-term target** → **Phased roadmap (rough)** → **Open questions**.

Effort estimates are deliberately bands (S/M/L/XL = ~2 / ~5 / ~10 / ~20+ dev-days), not point estimates. They will tighten during epic decomposition.

---

### C1.1 — Process & Service Lifecycle

**Mission.** Every Cloud Run instance starts in the right order, declares when it's ready, drains gracefully on SIGTERM, recovers orphaned work on resume, and lets ops ship new versions without dropping in-flight work.

**Sub-components.**
- `LifecycleManager` — registry of startup hooks, shutdown hooks, readiness checks. Symmetric API: `register_startup_hook(fn, order)` / `register_shutdown_hook(fn, order)` / `register_health_check(name, fn)`.
- `/health` and `/ready` endpoints — `/ready` returns 200 only when all checks pass; `/health` returns per-subsystem rollup `{db, redis, queue, scheduler, mcp_pool}: ok|degraded|down`.
- **Drain protocol** — on SIGTERM, stop accepting new requests, finish in-flight work up to a deadline, persist any unflushed checkpoints, then exit. In-flight LangGraph runs that can't complete in the drain window must checkpoint cleanly so another instance can resume.
- **Orphan recovery** — at startup, scan for `agent_runs` left in `RUNNING` with no checkpoint heartbeat within last 5 min, mark them as resumable, requeue.
- **Deployment safety** — gradual rollout (Cloud Run revision % traffic split), automated rollback hooks tied to error-rate / latency SLOs, blue-green capable.

**Key contracts.**
- Internal API: `LifecycleManager.register_*`, `LifecycleManager.run_startup()`, `LifecycleManager.drain(deadline)`.
- Endpoint: `GET /ready` → 200/503; `GET /health` → JSON rollup. Schema versioned.
- Event: `system.instance.started`, `system.instance.draining`, `system.instance.exited` on the bus.

**Integration points.**
- C1.4 Checkpoint Store — drain calls flush; orphan recovery reads.
- C1.10 Background Work — drain stops accepting new jobs first.
- C1.11 Observability — every lifecycle transition emits a span + event.

**Current state.** FastAPI lifespan manager exists. Shutdown hooks exist. Symmetric startup hooks and per-subsystem health endpoint do not. Orphan recovery is implicit (LangGraph checkpoints survive, but nothing actively scans for stuck runs). No automated rollback.

**Long-term target.** Multi-instance, drain-safe, blue-green/canary by default. Per-subsystem health visible. Orphans resumed within 60s of detection. Rollback automatic when SLOs breach within first N minutes of a new revision.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Symmetric hooks, `/ready` + `/health` rollup, basic drain | M |
| P2 | Orphan-run recovery scanner; drain-deadline checkpoint flush | M |
| P3 | Cloud Run canary + auto-rollback wiring | M |
| P4 | Cross-instance lifecycle events (multi-instance coordination) | S |

**Open questions.**
- Are we committed to Cloud Run long-term, or do we expect to move to GKE/EKS at scale? (Affects how much of the lifecycle we own vs. rely on the platform for.)
- SLO definitions for auto-rollback — RKP and AKN to align before P3.

---

### C1.2 — Persistence Substrate

**Mission.** A single, RLS-safe persistence layer that every other layer reads/writes through. Owns the contract for *how* data is stored, not *what* data is stored. User-scoped, transactional, migration-versioned, and disaster-recoverable. (Multi-tenancy posture is a deferred, open decision — see open questions.)

**Sub-components.**
- **DB client** — connection pool, RLS-aware Supabase client wrapper, service-role bypass for admin paths. Today: `db.user(user_id).table("...").select(...)` — user scope is enforced at the wrapper level via Supabase RLS (`auth.uid()`). Tenant-scoping is *not* assumed; if shared-DB tenancy is ever introduced (open question), it would layer in here without rewriting call sites.
- **Transaction support** — Postgres functions invoked via `rpc()` for multi-table atomic operations (e.g., create batch + tasks + audit entries in one shot). No half-written states.
- **Migration tooling** — versioned, forward + reverse, blue-green-safe (all migrations must be backward-compatible with the previous app version). Lives in `trybot-ui/supabase/migrations/` today; long-term we want it in this repo or a shared submodule.
- **File / object storage** — Supabase Storage buckets per data class (audio, documents, images), with signed URLs and lifecycle policies (auto-expire to comply with C1.12 retention).
- **Vector storage** — pgvector with a single `embeddings` table indexed by `(user_id, namespace, owner_id)`. HNSW indexes. Used by L4 (semantic memory) and L3 (skill discovery) — but the substrate is here, the schema/policies are theirs.
- **Archival & cold storage** — old data (closed runs > 6 months, deleted users in grace period, etc.) tiers to cold storage (GCS) on a scheduled job. Read-on-demand with a slow path.
- **Backup & DR** — Supabase daily backups + a weekly export to a secondary region; documented RPO/RTO; restore-drill cadence.

**Key contracts.**
- Internal API: `DbClient`, `StorageClient`, `VectorClient`, `MigrationRunner`.
- Schema: every domain table includes `id UUID`, `user_id UUID NOT NULL` (where applicable), `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`, plus Supabase RLS policies enforcing user-scoped access via `auth.uid()`.
- Event: `data.migration.applied`, `data.archival.run`, `data.backup.completed`.

**Integration points.**
- C1.6 Identity — `user_id` flows from Supabase JWT claims into RLS via `auth.uid()`.
- C1.10 Background Work — archival, backup, retention all run as scheduled jobs.
- C1.12 Compliance — retention policies and erasure pipelines call into this layer.
- L3/L4/L7 — every skill, memory, and persistence concern consumes this.

**Current state.** Supabase client + RLS at user level (phone-OTP auth). No multi-table transactions. Migrations live in sibling repo. Storage exists. Vector tables not deployed. Archival not built. Backups via Supabase only (no secondary region).

**Long-term target.** Hardened user-scoped client primitive. Multi-table transactions for any multi-row write. Migrations co-located + blue-green-safe. pgvector deployed and shared by L3/L4. Tiered archival. Documented RPO ≤ 1h, RTO ≤ 4h with quarterly restore drills. Tenancy posture (shared-DB or DB-per-tenant) remains an open question, not part of the long-term plan.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | User-scoped DB client wrapper hardening; transaction RPC pattern; RLS policy audit on existing tables | M |
| P2 | pgvector deploy + `embeddings` table; signed-URL helper for storage | M |
| P3 | Migration tooling co-located + reverse-migration discipline; blue-green migration playbook | M |
| P4 | Archival tier + retention sweeper (consumes C1.12 retention policies) | M |
| P5 | Secondary-region backup + restore drill | M |

**Open questions.**
- **Multi-tenancy posture (deferred decision — no commitment).** Three plausible futures, all open:
  - (a) **Stay user-scoped indefinitely.** Trybo remains B2C; tenancy never lands.
  - (b) **DB-per-tenant / per-deployment** for B2B and on-prem. Same code, same migrations, separate database per customer. Common ask from enterprise / regulated buyers; preserves the current OTP auth flow and code paths; no schema pollution.
  - (c) **Shared-DB multi-tenancy** with `tenant_id` columns + RLS rewrite. Largest retrofit cost: includes auth-flow rewrite (phone OTP → tenant-scoped JWTs), invite/membership UX, RLS migration on a live and already-large DB. Only justified if many small B2B tenants on a single deployment is the actual product shape.
  - **No work is scheduled.** We will not introduce `tenant_id` columns or rewrite auth speculatively. The decision is held open until a concrete B2B use case forces it.
- Do we move migrations into this repo or keep them in `trybot-ui`? (Currently a coordination headache.)
- Is `pgvector` in Supabase Pro tier sufficient for L4 long-term, or do we need a dedicated vector DB at scale?

---

### C1.3 — Event Bus & Streaming Spine

**Mission.** The internal nervous system. Components publish *facts that already happened* to named channels; N subscribers receive independently. Events are persisted (ordered, replayable, DLQ-backed) and streamable to clients (SSE/WS) without each component reimplementing the wheel.

> **Not the same as C1.10 Background Work.** The bus carries *notifications* (1 publish → N subscribers, fire-and-forget, no "owner" of the work). The job queue carries *instructions* (1 enqueue → 1 worker, retryable, cancellable, schedulable, with a typed handler that owns the work). They cooperate: a job completion *publishes an event*; an event subscriber may *enqueue a job*. See §7 cross-cutting note for the full distinction.

**Sub-components.**
- **Pub/sub core** — channel-keyed subscriber registry, in-process fan-out, ordered delivery per channel.
- **Persistence** — every event written to an append-only log table (`event_log`) with `(user_id, channel, seq, event_type, payload, created_at)`. Seq is per-channel monotonic.
- **Replay** — `bus.replay(channel, from_seq)` returns events from a given offset. Critical for crashed consumers and for late-joining clients.
- **Dead-letter queue** — `event_dlq` for events whose subscribers fail beyond max retries. Inspectable, requeueable.
- **Cross-instance fan-out** — at multi-instance scale, in-process fan-out is insufficient. Long-term: bus messages also published to GCP Pub/Sub or Redis Streams so subscribers on other instances receive them.
- **SSE / WS distribution** — client-facing event delivery sits *on top of* the internal bus. Mobile clients subscribe to `agent_run:{run_id}`, `chat:{chat_id}` etc.; the gateway translates to SSE/WS frames.
- **Schema versioning** — event payloads versioned (`event_type@v2`); subscribers tolerant of unknown fields, breaking changes go through a coexistence period.

**Key contracts.**
- Internal API: `bus.publish(channel, event_type, payload, user_id)`, `bus.subscribe(channel_pattern, handler)`, `bus.replay(channel, from_seq)`, `bus.dead_letter(reason)`.
- Channel naming convention: `<domain>.<entity>.<action>` (e.g., `run.node.completed`, `consent.request.created`, `system.instance.started`).
- Event taxonomy is versioned and lives in a shared module — additions are non-breaking, removals require a deprecation window.

**Integration points.**
- C1.2 Persistence — `event_log` and `event_dlq` tables.
- C1.9 Inbound Protocol Bridge — SSE/WS endpoints subscribe to the bus on the user's behalf.
- L2 Execution Engine — emits `run.*` and `node.*` events; the typed event taxonomy is co-owned.
- L4 Memory — subscribes to `run.completed` to trigger episode/fact writes.
- L5 UI — consumes everything via SSE.

**Current state.** `agent_run_events` table persists run events. SSE streaming works for individual runs. No general-purpose pub/sub spine. No replay-by-seq API. No DLQ. No cross-instance fan-out.

**Long-term target.** Generic bus that any component can publish/subscribe to. Replayable from any seq. DLQ inspectable from admin UI. Cross-instance delivery so we can scale Cloud Run to >1 instance without duplicate or lost events.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Generalize `agent_run_events` into a generic `event_log` with channel; in-process fan-out abstraction | M |
| P2 | `bus.replay()` API; DLQ table + retry tooling | M |
| P3 | Schema versioning convention + event taxonomy module | S |
| P4 | Cross-instance fan-out via Pub/Sub or Redis Streams | L |
| P5 | Admin UI for DLQ inspection / requeue | S |

**Open questions.**
- GCP Pub/Sub vs. Redis Streams for cross-instance — Pub/Sub is more durable and managed; Streams is lower-latency and we already run Redis. Decision deferred until we're actually multi-instance.
- Do we want event-sourcing semantics (rebuild state from events) for any domain? Probably not — too much cost, current snapshot-based model is fine.

---

### C1.4 — State & Checkpoint Store

**Mission.** Tasks that produce real-world side effects (calls, payments, sends) must be safely resumable after crashes. C1.4 provides the locking, idempotency, and checkpoint persistence that make resume safe.

**Sub-components.**
- **LangGraph checkpoint backend** — `langgraph-checkpoint-postgres` against Supabase. Already deployed (`public.checkpoints`, `checkpoint_blobs`, `checkpoint_writes`).
- **Distributed locking** — Redis `SETNX` lock on `checkpoint:{run_id}` with TTL on resume, so two instances can't resume the same run. Lock TTL ≥ max single-step duration; refresh on long-running steps.
- **Durable idempotency** — content-addressable `(skill_id, input_hash)` set in Redis (and persisted), not in-memory. Survives restarts.
- **Checkpoint metadata** — every checkpoint tagged with `saved_at, schema_version, component, run_id, user_id` for debuggability and migration.
- **Checkpoint versioning** — when state shape changes, old checkpoints either migrate forward (best) or are marked unresumable with a clear error.
- **GC & retention** — completed-run checkpoints aged out per retention policy (tied to C1.12). Default: 30 days.

**Key contracts.**
- Internal API: `lock_manager.acquire(key, ttl)`, `lock_manager.refresh(key)`, `idempotency.has(key)`, `idempotency.mark(key, result_hash)`.
- Tables: existing LangGraph tables; new `idempotency_keys (user_id, key, marked_at, result_ref)`.
- Event: `checkpoint.saved`, `checkpoint.resumed`, `lock.contention.detected`.

**Integration points.**
- C1.1 Lifecycle — drain calls `flush_checkpoints()`.
- C1.5 Sessions — for live calls, session state is checkpointed too (different cadence).
- L2 Execution Engine — every node-completion is a checkpoint boundary.

**Current state.** LangGraph checkpoints deployed. Idempotency uses an `InMemoryRunStore` (process-local — loses state on restart). No distributed locking — single-instance only. No GC.

**Long-term target.** Resume is safe across instances. Idempotency persists across restarts. Old checkpoints garbage-collected on schedule. Checkpoint schema versions can roll forward.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Redis-backed distributed lock for resume; lock TTL refresh on long steps | M |
| P2 | Migrate `InMemoryRunStore` to Redis (idempotency_keys table as backup) | S |
| P3 | Checkpoint metadata columns + schema_version | S |
| P4 | Scheduled GC sweeper + retention integration | S |
| P5 | Forward-migration of checkpoint schemas (only when needed) | M |

**Open questions.**
- Lock TTL — what's a safe upper bound? Voice calls can run 30+ min. Recommend 60s TTL with active refresh from the running node.
- Idempotency key shape — input hash today is naive. Should it normalize timestamps / nondeterministic fields? Defer — case-by-case.

---

### C1.5 — Session / Working-State Layer

**Mission.** Real-time interactions (voice calls, live chat) generate ephemeral state at high frequency — too hot for the DB, too valuable to lose if a process restarts mid-call. Two-tier store: in-memory for sub-millisecond reads, Redis for durability and cross-instance visibility.

**Sub-components.**
- **Two-tier session store** — write-through cache: writes go to Redis; reads check in-memory first, fall back to Redis. Sticky session affinity is *not* required (Redis-backed = portable).
- **Session lifecycle** — `session.created`, `session.updated`, `session.expired` published to C1.3. Redis keyspace notifications surface TTL expiry.
- **TTL & cleanup** — sessions auto-expire (default 1h, configurable per session type). On expiry, a final flush event allows downstream consumers (transcript writers, billing) to capture the close-out state.
- **Optimistic locking** — for sequential turn-taking (e.g., voice, chat), a per-session sequence number prevents lost updates if two writes race.
- **Size limits** — max bytes per session, with overflow detection. Prevents a runaway session from blowing memory.

**Key contracts.**
- Internal API: `sessions.create(type, ttl)`, `sessions.get(id)`, `sessions.update(id, patch, expected_seq)`, `sessions.expire(id)`.
- Events on C1.3: `session.created`, `session.updated`, `session.expired`.

**Integration points.**
- C1.3 Event Bus — lifecycle events.
- L2/L3 — voice, chat, and any "live" multi-turn interaction.
- C2 / Voice adapter — call-state lives here during a call.

**Current state.** Hot fields live in a Python dict per process. Means sticky-routing is required, which fights Cloud Run's load balancing. No formal session contract.

**Long-term target.** Redis-backed sessions, multi-instance safe, lifecycle eventful, size-bounded, sequence-locked.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Migrate hot fields to Redis hash; in-memory cache as read-through | M |
| P2 | Lifecycle events to bus; keyspace notification for expiry | S |
| P3 | Optimistic locking via sequence number | S |
| P4 | Size limits + overflow alerts | S |

**Open questions.**
- Voice latency budget vs. Redis read latency — single-region Redis ~1ms, fine. If we go multi-region, may need session locality.

---

### C1.6 — Identity & Access

**Mission.** Authenticate every request, scope everything to the calling user, enforce per-user / per-tier limits, and provide the primitives for account lifecycle. (RBAC, orgs, and tenancy are deferred decisions — see open questions.)

**Sub-components.**
- **JWT auth** — Supabase JWT validation (phone OTP today); FastAPI `Depends(get_authenticated_user)` extracts identity. Token refresh handled with explicit refresh endpoint and 401 → refresh → retry on the client.
- **User scope** — every authenticated request resolves to `user_id`. Flows into the DB client via Supabase auth claims, into RLS via `auth.uid()`. (Tenant scope is *not* assumed; see open questions.)
- **Role-based access control** — *deferred*. Not built today. If shared-DB tenancy is ever adopted (open question), RBAC layers on top via a `(tenant_id, user_id, role)` membership table with roles `owner | admin | member | viewer` and a `Depends`-chain idiom. Until then, all paths are single-user; service-role token gates admin paths.
- **Rate limiting** — per-user and per-route sliding-window counters in Redis. Library: `slowapi` or `fastapi-limiter`. Returns 429 with `Retry-After`.
- **Cost ceilings** — separate from rate limits. Token-bucket per (user, plan) decremented before LLM/tool calls. Hard ceiling per billing period; soft ceiling triggers in-app warning. Independent of rate limits because expensive calls (long LLM completions) cost more than cheap ones.
- **Account lifecycle** — registration, phone OTP verification, password-less, account deletion (compliance hook in C1.12), session revocation.
- **Service-role auth** — admin/internal paths use a service-role token; never exposed to clients.

**Key contracts.**
- Internal API: `auth.get_user(req)`, `rate_limit.check(key)`, `cost_ledger.spend(user_id, amount)`. Tenancy / RBAC APIs (`auth.get_tenant`, `auth.require_role`) reserved for future, gated on the multi-tenancy decision.
- Tables: existing `user_profiles`; new `cost_ledger (user_id, period_start, spent, limit)`. `tenants` / `tenant_memberships` deferred.
- Events: `auth.session.created`, `auth.session.revoked`, `cost.ceiling.warned`, `cost.ceiling.exceeded`. Tenancy events (`tenancy.member.*`) deferred.

**Integration points.**
- C1.2 Persistence — RLS depends on user claim today; tenant claim only if shared-DB tenancy is ever adopted.
- C2 Execution Tools — every tool call passes through cost ledger spend.
- L2 Planner — checks cost ceiling before kicking off expensive plans.
- C1.12 Compliance — account deletion drives erasure pipeline.

**Current state.** JWT (phone OTP) + RLS at user level. No tenancy model. No RBAC. No rate limits. No cost ceilings. Account deletion exists.

**Long-term target.** Hardened user-scoped auth, with rate limits and cost ceilings keyed by `(user, plan)` and surfaced to clients via headers + 429s. Tenancy / RBAC remain optional layers — introduced only if a B2B path forces them, see open questions.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Rate limiting with `slowapi` + Redis backend; per-route configs | M |
| P2 | Cost ledger primitive (user-scoped); per-plan ceilings; in-app warning + 402 on hard ceiling | L |
| P3 | Service-role hardening + admin-path audit | S |
| P4 | *(Deferred — not scheduled.)* Tenancy + RBAC layers. Only triggered by a concrete B2B use case. Includes `tenants` + `tenant_memberships`, tenant claim in JWT, auth-flow rewrite from phone OTP to tenant-scoped login, RLS migration on a live DB. Costs are non-trivial; see open questions for the three-way deferred decision. | XL |

**Open questions.**
- **Multi-tenancy posture (deferred decision — open until a concrete B2B use case lands).** Three plausible futures, all open:
  - (a) **Stay user-scoped indefinitely** — Trybo remains B2C.
  - (b) **DB-per-tenant / per-deployment** — same code, same migrations, separate DB per B2B customer. Common ask from enterprise / regulated buyers; on-prem-friendly; preserves OTP auth and current code paths.
  - (c) **Shared-DB multi-tenancy** with `tenant_id` columns + RLS rewrite + auth-flow rewrite. Largest retrofit; only justified if many small B2B tenants on a single deployment is the actual product shape.
  - **No work scheduled now.** We do not pay the auth rewrite, schema retrofit, or RLS migration cost speculatively. Revisit only when a buyer / use case forces a decision.
- Cost ceilings — who defines "plan"? Billing system not built yet. Recommend a hardcoded plan map until billing exists; cost ceiling primitive lands first.

---

### C1.7 — Configuration & Feature Flags

**Mission.** A single typed config surface for the app, secrets isolated, hot-reloadable where it matters, with percent/cohort feature flags driving safe rollouts.

**Sub-components.**
- **Pydantic-settings** — typed config classes, env-var sourced, validated at startup. `get_settings()` cached with `lru_cache`.
- **Hot reload** — Cloud Run is request-driven, so SIGHUP-style daemon reloads don't apply. Pattern: a Pub/Sub topic `config.changed` triggers a `force_latest=True` invalidation + reload. Bounded set of "hot" config keys (LLM model selection, rate-limit overrides, MCP server list); the rest is restart-only.
- **Secret management** — env vars for now; long-term, GCP Secret Manager with rotation. KMS keys for application-level encryption (C1.12).
- **Feature flags** — OpenFeature Python SDK as the vendor-neutral interface. Provider: GrowthBook (richest cohort/attribute targeting, free) or self-hosted Unleash. Decision deferred until we actually need percent rollouts.
- **Per-skill / per-tool config** — flags can target by `(user_id, plan, route)`. Cohort targeting drives canary rollouts of new skills, new providers, new prompt versions.
- **Config versioning** — tracked in git; production rollouts via PR + Cloud Run revision deploy. No database-backed config admin (yet).

**Key contracts.**
- Internal API: `settings.<key>`, `flags.is_enabled(name, ctx)`, `flags.variant(name, ctx)`.
- Events: `config.changed`, `flag.evaluated` (sampled — for analytics only).

**Integration points.**
- Every layer reads config; few write it.
- C1.3 Event Bus — `config.changed` triggers in-process invalidation.
- C1.6 — flags can target on user/plan, requires identity context.

**Current state.** Pydantic-settings exists. Env-var only. No hot reload. No feature flags (just env booleans).

**Long-term target.** OpenFeature wrapper everywhere, with %/cohort targeting for real rollout safety. Hot reload for a specific allowlist of keys. Secrets in GCP Secret Manager with documented rotation runbooks.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | OpenFeature SDK wrapper + provider choice (GrowthBook recommended); replace all env booleans | M |
| P2 | Cohort targeting on identity attributes; gradual-rollout helper for new skills | S |
| P3 | Hot-reload allowlist with Pub/Sub trigger | M |
| P4 | GCP Secret Manager migration + rotation playbook | M |

**Open questions.**
- GrowthBook vs. Unleash — both viable. GrowthBook is better for experimentation (A/B/N), Unleash is better for pure flag management. Recommendation: GrowthBook, since we'll want to A/B prompts and provider routing. Locks in by P1.
- Should `flags.is_enabled` calls themselves be sampled into spans? Yes for debug, no for production noise. Sampled at 1% by default.

---

### C1.8 — Outbound HTTP Substrate

**Mission.** Every outbound HTTP call from anywhere in the app goes through one centralized client with pooling, configurable timeouts, retries, circuit breakers, request signing, and observability. No more `httpx.AsyncClient()` scattered across 22 services.

**Sub-components.**
- **Provider registry** — `{provider_name: {base_url, auth, timeout, max_retries, breaker_config}}` populated from config. Adding a new provider is a config entry.
- **Connection pooling** — single `httpx.AsyncClient` per provider, pool sized per concurrency expectations, reused across requests. Eliminates TCP+TLS handshake overhead (100-300ms per call).
- **Timeouts** — per-provider defaults with per-call overrides via `Timeout(connect=5, read=30, write=10, pool=5)`. No "default infinite" timeouts ever.
- **Retries** — `tenacity` with `stop_after_attempt(5)` + `wait_random_exponential(min=1, max=60)`. **Always honors `Retry-After`**. Retries only on transient/5xx, never on 4xx (except specific 429s).
- **Circuit breakers** — `aiobreaker` (asyncio fork of pybreaker), one breaker per provider with `fail_max=5`, `reset_timeout=60s`. **Critical ordering: breaker outside retry**, so a single logical call attempt is one breaker trip, not N. (Otherwise tenacity's retry-storm trips the breaker on the first failure.)
- **Request signing** — for providers requiring HMAC/OAuth/etc., signing happens in the substrate. Each provider declares its auth spec.
- **Response caching** — opt-in for idempotent GETs with TTL. Off by default.
- **Observability** — every call emits a span with `gen_ai`-style or `http`-style attributes (provider, route, status, duration, retry count, breaker state). Logs structured.

**Key contracts.**
- Internal API: `http_client.post(provider="twilio", path="/Calls", json={...})`, `http_client.get(provider="composio", path="/tools", params={...})`.
- Provider config schema: `{ base_url, auth_type, auth_config, default_timeout, retry_config, breaker_config }`.
- Events: `http.call.started/completed/failed`, `http.breaker.opened/closed`.

**Integration points.**
- **Every C2 adapter and skill** uses this. C2 owns the *what*; C1.8 owns the *how*.
- C1.7 Config — provider registry sourced here.
- C1.11 Observability — every call is a span.

**Current state.** Each service constructs its own `httpx.AsyncClient`. Some retry, some don't. No circuit breakers. No central provider registry. No standardized timeouts. ~22 places to change when an external API moves.

**Long-term target.** One client, one registry, one observability path. Adding a provider = config change + one auth spec.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Centralized client with provider registry; pooling + timeouts | L |
| P2 | Tenacity retry wrapper with `Retry-After` honoring; ordered correctly outside breaker | M |
| P3 | aiobreaker integration; per-provider configs | M |
| P4 | Migrate the 22 ad-hoc httpx callers to the substrate | L |
| P5 | Request signing helpers (HMAC, OAuth refresh, JWT) | M |
| P6 | Optional GET caching for idempotent endpoints | S |

**Open questions.**
- Migration strategy — we won't migrate all 22 in one sprint. Strangler pattern: new code uses the substrate, old code migrates as we touch it. Lock in the deadline (~Sept) for full migration.
- Do we want `pyresilience`'s composite `@resilient()` decorator (retry+breaker+timeout+bulkhead in one) instead of composing them ourselves? Tempting for ergonomics; deferred until we've used the lower-level pieces enough to know what we want.

---

### C1.9 — Inbound Protocol Bridge

**Mission.** Every inbound request — REST API call, SSE/WS subscription, webhook from a provider — gets normalized at the edge: validated, authenticated, deduplicated, signature-verified, and security-headers-stamped before any domain code runs.

**Sub-components.**
- **REST routing** — FastAPI handlers with Pydantic validation on every route. No untyped `dict` request bodies.
- **SSE / WS endpoints** — persistent connections subscribe to C1.3 channels. Reconnection-friendly: client provides `last_event_id`, server replays from there.
- **Webhook normalization** — generic adapter pattern: each provider has a normalizer that translates Twilio/Meta/Composio/etc. webhook shapes into internal events on the bus. Webhook handlers are *thin* — just validate, dedupe, publish.
- **Webhook dedup** — Redis `SETNX webhook:{provider}:{webhook_id}` with 24h TTL. Providers retry; we must process exactly once.
- **Signature verification** — every provider's HMAC/JWT signature checked at the edge. Reject unsigned/forged requests with 401 before any business logic.
- **Security headers** — `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy: ...`. Applied via middleware.
- **API versioning** — `/api/v1/...` from the start; `/v2` reserved for breaking changes; coexistence pattern documented.
- **Request ID propagation** — every request gets `X-Request-Id` (or generated); flows into `trace_id` (C1.11), into logs, into events.

**Key contracts.**
- Internal API: `webhook_handler.normalize(provider, raw_payload) → InternalEvent`, `signature.verify(provider, raw_payload, headers)`.
- Middleware: `RequestContextMiddleware` sets `request_id`, `user_id` into a `contextvar`.
- Events: `webhook.received`, `webhook.deduplicated`, `webhook.signature.failed`.

**Integration points.**
- C1.3 — webhooks publish normalized events.
- C1.6 — auth happens here.
- C1.11 — every inbound request creates a root span.

**Current state.** REST + Pydantic on most routes (some inconsistent). SSE for agent runs. Webhook handlers per-provider, no centralized normalizer. Some signature verification, no shared library. Some security headers via middleware, not all. Request IDs partial.

**Long-term target.** Every route validated. Every webhook normalized + deduped + signature-verified by a generic primitive. Security headers consistent. Versioned API.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Generic webhook normalizer + dedup; migrate 2-3 highest-volume providers | M |
| P2 | Signature verification helpers per provider auth type | S |
| P3 | Consistent security headers middleware | S |
| P4 | API versioning convention + `/v1` rebrand | S |
| P5 | SSE replay-from-last-event-id (if not already in C1.3 P2) | S |
| P6 | Migrate remaining webhook handlers | M |

**Open questions.**
- WebSocket vs. SSE long-term — SSE is sufficient for one-way streaming (which is most of what we do). WS for bidirectional voice control? Deferred.
- API versioning policy — we don't have external API consumers yet. Pre-bake `/v1` so we don't have a costly rename later.

---

### C1.10 — Background Work & Scheduling

**Mission.** A typed job queue + cron scheduler for *work that needs to be done by exactly one worker*. At-least-once delivery to *one* handler (not N subscribers), retry policies, DLQ, cancellation, priority, scheduling. Usable by any layer.

> **Not the same as C1.3 Event Bus.** The queue is `enqueue → claim → execute → ack` semantics with a single owning handler per job type. The bus is `publish → fan-out to N` semantics. If "send the morning briefing" were a bus event, every subscriber would send it (5 briefings). If "user finished a run" were a job, only one consumer would react (analytics, memory writer, UI stream all miss). Different guarantees, different shapes. They compose: jobs publish events on completion; subscribers may enqueue follow-up jobs.

**Sub-components.**
- **Job queue** — typed jobs with handler registry. At-least-once delivery, idempotent handlers required (handler responsibility, primitive provides the contract).
- **Worker pool** — async workers per job type, concurrency configurable, gracefully drain on shutdown.
- **Retry policy** — per-job-type retry config (max attempts, backoff). On exhaustion → DLQ.
- **Dead-letter queue** — `job_dlq` table, inspectable from admin UI, requeue with one click.
- **Scheduling** — cron-style recurring (daily briefings, hourly rollups), one-shot delayed (`run X at T+30s`), tied to user timezone.
- **Cancellation** — `jobs.cancel(job_id)`. In-flight work must check cancel flag at safe checkpoints.
- **Priority** — initially FIFO. Long-term: priority lanes (interactive > scheduled > maintenance).
- **Observability** — every job execution is a span; metrics on queue depth, latency, failure rate per job type.

**Key contracts.**
- Internal API: `jobs.enqueue(type, payload, schedule=None, priority=normal)`, `jobs.cancel(id)`, `jobs.register_handler(type, fn)`.
- Tables: `jobs (id, user_id, type, payload, status, attempts, scheduled_at, ...)`, `job_dlq`.
- Events: `job.enqueued`, `job.started`, `job.completed`, `job.failed`, `job.dead_lettered`.

**Integration points.**
- L2 Execution Engine — registers scheduled-graph-execution job type.
- L4 Memory — episode-write jobs, archival jobs.
- C1.12 Compliance — retention sweeper, GDPR/DPDP request fulfillment jobs.
- C1.1 Lifecycle — drain stops accepting new jobs first.

**Current state.** APScheduler-based scheduler exists for recurring tasks. Job-queue patterns ad-hoc per service. No shared DLQ. No priority. No admin UI.

**Long-term target.** One queue, typed jobs, observable, cancelable, DLQ-inspectable, priority-aware, multi-instance-safe.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Generic typed-job primitive on top of existing scheduler; handler registry | M |
| P2 | DLQ table + admin endpoint for inspect/requeue | S |
| P3 | Cancellation primitive + cooperative-cancel pattern documented | S |
| P4 | Priority lanes (interactive > scheduled > maintenance) | M |
| P5 | Multi-instance safety: leader-elect for cron, distributed work claim | M |

**Open questions.**
- Do we adopt Celery/RQ/arq, or stay on a custom-on-Postgres queue? Postgres-as-queue (using `SKIP LOCKED`) is operationally simpler and we already have the DB; recommend staying there until volume justifies a dedicated broker.
- Multi-instance cron — one scheduler instance, leader-elected, vs. each instance running cron with idempotent handlers. Recommend leader-elect.

---

### C1.11 — Observability & Tracing Spine

**Mission.** Everything that happens — every request, every node, every tool call, every bus event — is observable. Logs structured, traces propagated through async boundaries, metrics aggregated, generic audit log queryable. Standard semantic conventions.

**Sub-components.**
- **Structured logging** — `structlog`, JSON output, with `trace_id`, `user_id`, `request_id` automatically attached from contextvars.
- **Distributed tracing** — OpenTelemetry SDK with OTLP exporter. Spans for: HTTP request (root), planner LLM call, plan compilation, each node execution, each tool/adapter call, each DB query (sampled). **GenAI semantic conventions** for LLM/tool spans (`gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.tool.name`).
- **Async context propagation** — contextvars carry trace context across `asyncio.create_task` and LangGraph's `Send` parallel fanout. The fix: any spawn that crosses the active span must explicitly carry context (`opentelemetry.context.attach`/`detach` in a try/finally) — documented as a coding pattern.
- **Metrics** — RED metrics (Rate, Errors, Duration) per route + per skill + per provider. Plus token/cost metrics from LLM spans.
- **Backend** — Langfuse self-hosted (OSS, MIT, OTLP-native, free) for traces. Cloud Logging for raw logs. Prometheus + Grafana for metrics. Decision deferred between Langfuse and LangSmith managed; Langfuse default for cost/control.
- **Generic audit log** — separate from observability traces. Table-backed (`audit_log`). Every privileged operation (admin actions, consent decisions, data exports, account deletions) writes one row. Queryable, tamper-evident (append-only).
- **Sampling** — head-based for traces, 100% for errors, 100% for slow requests, ~10% baseline. Tunable via flag.

**Key contracts.**
- Internal API: `tracer.start_span()`, `logger.bind(...)`, `metrics.observe(...)`, `audit.record(actor, action, target, metadata)`.
- Span attributes follow GenAI semconv where applicable; everything else uses standard HTTP/DB semconv.
- Audit log schema: `(id, actor_id, actor_type, action, target_type, target_id, metadata, created_at)`.

**Integration points.**
- **Every** layer. This is the spine.
- C1.3 Bus events also create spans; bus-event publish is a span event, not a separate span (avoid span explosion).
- C1.12 Compliance — audit log feeds DSAR exports.

**Current state.** structlog. RequestContextMiddleware covers HTTP path/method/IP. trace_id partial (started in TB-662 / TB-669 series). No OTel spans yet. No central audit log. No metrics dashboards.

**Long-term target.** Full OTel coverage with GenAI semconv. Trace propagated through every async boundary. RED metrics dashboarded. Audit log unified across consent decisions, admin actions, data access. Backend choice locked.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | OTel SDK wired in; HTTP root spans; `trace_id` everywhere | M |
| P2 | Planner / node / tool spans with GenAI semconv | M |
| P3 | Async context propagation pattern documented + enforced; fix `Send` fanout | M |
| P4 | Backend choice (Langfuse vs LangSmith); deploy + dashboards | M |
| P5 | Generic `audit_log` + recording from M11 hooks | M |
| P6 | RED metrics per route/skill/provider | M |
| P7 | Cost metric extraction from `gen_ai.usage.*` | S |

**Open questions.**
- Langfuse self-hosted vs. LangSmith managed — Langfuse is OSS, MIT, no per-trace cost. LangSmith is tightly integrated with LangGraph and has end-to-end OTel support. Recommendation: **Langfuse** for cost control and ownership; revisit if LangSmith integration buys us specific debugging power we lack. Lock in by P4.
- Trace sampling targets — what's the cost of 100% sampling? Cheap at our volume; expensive at scale. Default 100% for now, tunable via flag.

---

### C1.12 — Compliance & Data-Lifecycle Primitives

**Mission.** Provide the *machinery* for compliance — retention, erasure, export, encryption, redaction. The *policies* (who can see what, what's PII, what consent looks like) live in M11/Governance and the relevant domain layer. C1.12 is what M11 and L7 stand on.

**Sub-components.**
- **Retention policies as data** — `retention_policies (data_class, ttl_days, action)` table; e.g., audio recordings → 90 days → delete; chat messages → 365 days → archive. C1.10 background sweeper enforces.
- **GDPR / DPDP rights pipelines** — three pipelines: **export** (DSAR — collect all user data into a downloadable bundle), **erasure** (cascade delete + soft-delete + audit), **correction** (update + audit). Each is a job type that can run idempotently and is auditable.
- **Encryption helpers** — application-level encryption for sensitive fields (financial info, voice biometrics, PII). KMS-backed envelope encryption. `encrypt_field(plaintext, key_id)` / `decrypt_field(ciphertext, key_id)`. Key rotation supported.
- **PII redaction utility** — pluggable detector (regex for Aadhaar/PAN/phone/IFSC; future: NER for names/addresses) + redactor. Returns `(redacted_text, detections)`. Used by L7 (governance) at egress, by L8 (intelligence) before LLM calls, etc. **C1.12 owns the utility; the *enforcement decisions* — when to redact, what to log — belong to M11.**
- **Consent capture data model** — generic structure for "user consented to X at time T with content C, withdrawable." Consent rules, UI, and enforcement live in M11/L11; the generic "we have a record" lives here so that DPDP Section 5/6 audit trails work.
- **DPDP Act 2023 specifics** —
  - 7-year consent record retention if a Consent Manager is registered.
  - Data principal rights: access, correction, erasure, portability, nominee.
  - Breach notification to Data Protection Board within 72 hours (and to user "without delay").
  - Grievance Officer endpoint surfaced; contact in privacy policy.
  - Cross-border: negative-list model — Supabase US/EU regions remain legal until/unless India blacklists. Sectoral overlays (RBI, telecom) apply if we ever offer those services.

**Key contracts.**
- Internal API: `retention.enforce()`, `gdpr.export(user_id)`, `gdpr.erase(user_id)`, `pii.scan(text)`, `pii.redact(text, policy)`, `crypto.encrypt(plaintext, key_id)`, `crypto.decrypt(ciphertext, key_id)`.
- Tables: `retention_policies`, `gdpr_requests (id, user_id, type, status, requested_at, completed_at)`, `encryption_keys` (KMS-backed), `consent_records`.
- Events: `retention.enforced`, `gdpr.requested`, `gdpr.completed`, `pii.detected`, `breach.detected`.

**Integration points.**
- C1.10 — retention sweeper, GDPR jobs.
- C1.11 — every privileged op writes to audit log.
- C1.6 — account deletion triggers erasure.
- M11/Governance — owns the policies, calls into these primitives.
- L7 / consent layer — consent records, withdrawal endpoints.

**Current state.** Account deletion exists. No retention policies. No DSAR/export. No application-level encryption. No PII redaction utility. Audio retention flag exists but no general framework.

**Long-term target.** Every data class has a retention policy. DSAR/erasure pipelines run as scheduled or on-demand jobs. Sensitive fields encrypted at rest. PII redaction available as a utility. DPDP compliance scaffolding in place: 72-hour breach notification, grievance officer endpoint, consent records with audit, data principal rights APIs.

**Phased roadmap.**
| Phase | Scope | Effort |
|---|---|---|
| P1 | Retention-policies table + sweeper job; first 3 data classes (audio, chat, agent_run_events) | M |
| P2 | DSAR export pipeline (`gdpr.export`) | M |
| P3 | Erasure pipeline (`gdpr.erase`) — cascade safe-deletes, audit trail | L |
| P4 | PII detection utility (regex first; NER deferred) | M |
| P5 | Encryption helpers + KMS integration; first sensitive fields encrypted | M |
| P6 | Breach detection + 72h notification workflow + Grievance Officer endpoint | M |
| P7 | Consent records primitive (drop-in for M11) | S |
| P8 | Cross-border data-flow audit + secondary-region eligibility check | S |

**Open questions.**
- Do we register as a Data Fiduciary under DPDP from launch? Probably yes (any user-facing service handling personal data qualifies). Affects 7-year consent retention requirement.
- Do we eventually classify ourselves as a Significant Data Fiduciary (SDF)? SDF designation comes from the Central Government based on volume/sensitivity criteria; we plan as if not SDF, but the architecture should be SDF-upgradable (audit, DPIA artifacts).
- KMS choice — Supabase Vault, GCP KMS, or HashiCorp Vault? Recommendation: **GCP KMS** since we're already on GCP for compute; cheapest in-cloud option.

---

## 7. Cross-cutting design principles

Pulled out so they're not buried in 12 sections.

**0. Event Bus (C1.3) and Job Queue (C1.10) are distinct primitives with different guarantees.** This is the most-asked-about boundary in this doc, so settling it up front:

| | Event Bus (C1.3) | Job Queue (C1.10) |
|---|---|---|
| Semantics | Pub/sub — 1 publish → N subscribers | Work claim — 1 enqueue → 1 worker |
| Delivery | At-least-once per subscriber | At-least-once total |
| Ordering | Per-channel sequence | None (priority lanes) |
| Retry policy | Subscriber's own (or DLQ on subscriber failure) | First-class: max attempts, backoff, DLQ |
| Cancellation | N/A (already fired) | First-class |
| Scheduling | N/A (immediate) | First-class: cron + delayed |
| Replay | By seq from any point | N/A |
| Typical payload | A fact ("X happened") | An instruction ("do X") |
| Storage | `event_log` (append-only) | `jobs` table (mutable status) |

They cooperate constantly: a job's completion publishes an event; an event subscriber may enqueue a follow-up job. Example: `run.completed` event → memory subscriber enqueues an `episode.write` job → job worker computes the episode → publishes `memory.episode.written` event. Two systems, one flow. Conflating them is the most common platform mistake.

1. **User-scoped today; tenancy is a deferred, open question.** Every primitive accepts a `user_id` and uses Supabase RLS via `auth.uid()`. Tenancy — whether shared-DB with `tenant_id` columns, or DB-per-tenant deployments — is *not* assumed and not built. The decision is held open until a concrete B2B use case lands. If B2B happens, the current default assumption is **DB-per-tenant deployment** (clean code, no schema pollution, often what enterprise/on-prem buyers ask for anyway), but no path is committed. We do not pay the auth-flow rewrite, schema retrofit, or RLS migration cost speculatively.

2. **Primitives, not policies.** C1 owns *machinery* (a retention sweeper, a PII redactor, a rate limiter). C1 does *not* own *rules* (what's PII, what's the retention, what's the rate limit value). Rules live in their domain layer. The primitive is configured.

3. **Idempotent everything.** Every job, every retry, every webhook. C1 provides idempotency primitives; consumers must use them.

4. **Async-safe context.** Trace context, user context, request context flow through `contextvars`. Spawning across an `asyncio.create_task` boundary requires explicit context handoff — codified as a coding rule.

5. **Strangler migration.** Most C1 surfaces have a current naive implementation. New code uses the substrate; old code migrates as we touch it. Set a deadline for full migration per surface (~Sept 30) so nothing rots indefinitely.

6. **Versioned everything that has a wire format.** Events, API routes, checkpoints, config schemas. Breaking changes go through coexistence windows.

7. **Observable by default.** No new C1 primitive ships without spans, metrics, and structured logs. The substrate provides the helpers; not opting in is harder than opting in.

---

## 8. Dependencies on other layers

C1 has no upstream layers — it's the foundation. But it has external dependencies:

| Dependency | What we use | Long-term commitment |
|---|---|---|
| **GCP Cloud Run** | Compute, autoscaling, deployment | Confirmed for now; revisit if we hit Cloud Run's per-instance / cold-start ceilings |
| **Supabase Postgres** | Primary datastore + RLS + Storage + Auth | Confirmed for now; pgvector at Pro tier may need replacing at scale |
| **Redis** | Locks, rate limits, sessions, idempotency, caching | Confirmed |
| **GCP Pub/Sub or Redis Streams** | Cross-instance event fan-out (when we go multi-instance) | Decision deferred |
| **GCP KMS** | Application-level encryption | Recommended; deferred until C1.12 P5 |
| **Langfuse (self-hosted) or LangSmith** | OTel trace backend | Decision pending C1.11 P4; Langfuse leaning |
| **Firebase / FCM** | Device registry only at C1 layer (the *send* moves to C2) | Confirmed |

---

## 9. Layers that depend on C1

| Layer | What they need from C1 |
|---|---|
| **C2 Execution Tools** | C1.8 (HTTP), C1.9 (webhook bridge), C1.11 (per-tool spans), C1.4 (tool idempotency) |
| **L2 Agent Runtime Brain** (Planner + Execution Engine) | C1.4 (checkpoints), C1.3 (run events), C1.5 (live session), C1.10 (scheduled triggers), C1.2 (run/task tables) |
| **L3 Capability System** | C1.2 (registry tables, vector index), C1.7 (skill rollout flags), C1.11 (skill stats from telemetry) |
| **L4 Intelligence & Memory** | C1.2 (vector storage, RLS), C1.3 (memory.* events), C1.10 (episode-write jobs) |
| **L5 Product (UI)** | C1.9 (REST/SSE/WS), C1.3 (events to stream), C1.6 (auth) |
| **M11 Governance** | C1.12 (compliance primitives), C1.11 (audit log), C1.6 (RBAC, rate limits) |

Anything that doesn't fit one of these is a candidate for a misclassified ticket — it probably belongs in a domain layer, not in C1.

---

## 10. Phased delivery (rough, to be refined post review)

C1 is too big to land as one block. Sequence by criticality and unblocking dependencies:

**Tranche A — user-scoped + observable foundation (must-have for any post-beta scaling).**
- C1.6 P1 (rate limits) + P3 (service-role hardening), C1.2 P1 (DB client hardening + RLS audit), C1.11 P1-P3 (OTel + async context), C1.1 P1-P2 (lifecycle + orphan recovery). Tenancy / RBAC explicitly out of scope here — see §11 open question 1.

**Tranche B — outbound robustness + state safety.**
- C1.8 P1-P4 (centralized HTTP), C1.4 P1-P2 (distributed locking + durable idempotency), C1.5 P1-P2 (Redis-backed sessions).

**Tranche C — operational maturity.**
- C1.3 P1-P3 (generic event spine), C1.10 P1-P3 (typed job queue + DLQ), C1.7 P1-P2 (OpenFeature + cohort flags).

**Tranche D — compliance + scale.**
- C1.12 (most phases), C1.11 P5-P7 (audit log + metrics), C1.2 P3-P5 (migration discipline + archival + DR), cross-instance (C1.3 P4, C1.10 P5).

Tranches A and B unblock most of L2/L3/L4/C2 work. Tranche C is the operational glue. Tranche D is what we need before we're "compliance-ready" and "horizontally scalable."

**Total rough effort:** ~80-110 dev-days across the surface. Allocated against the 40% architecture bucket, that's spread across most of the year. Real estimates per phase tighten when each surface decomposes into epics and tickets.

---

## 11. Open questions for review

Ranked by how much they constrain other decisions if we get them wrong:

1. **Multi-tenancy posture (deferred — no commitment now).** Three options open: (a) stay user-scoped (B2C only); (b) **DB-per-tenant / per-deployment** for B2B and on-prem — assumed default *if* B2B happens, because it preserves the current OTP auth flow and code paths and is what enterprise/regulated buyers typically ask for; (c) shared-DB multi-tenancy with `tenant_id` everywhere — explicitly *not* recommended given the current OTP auth flow, the size of the existing DB, and the likely B2B buyer profile. **No work is scheduled.** We will not build tenancy primitives until a concrete B2B use case forces a decision.
2. **Feature-flag provider** (GrowthBook vs. Unleash)? Recommended GrowthBook — better experimentation primitives.
3. **KMS choice** (GCP KMS vs. Supabase Vault vs. Vault)? Recommended GCP KMS.
4. **Cross-instance fan-out** (GCP Pub/Sub vs. Redis Streams)? Defer until we're multi-instance.
5. **DPDP Data Fiduciary registration timing** — pre-launch or post? Affects retention obligations and Grievance Officer surface.

---

## 12. Companion document

C2 Execution Tools (TB-706) sits alongside this and shares several primitives — particularly C1.8 (HTTP), C1.9 (webhook bridge), C1.11 (telemetry), C1.4 (idempotency). See `roadmap-plan-akn/agentic-orchestration-40/execution-tools.md`.

---

## 13. Source references

- April 5 meeting notes: `meetings/2026-04-05.md`
- Pre-existing draft (KPR): `roadmap-plan-kpr/agentic-orchestration/barebone-platform.md`
- GTM gap analysis: `gap-analysis/gtm-gaps.md`
- Sprint plan: `sprint-plan.md`
- Platform completion snapshot: `../../trybo-arch/platform-completion-status.md` (M1 Core Platform section)
- Trybo arch (gold standard): `../../trybo-arch/architecture.md`, `../../trybo-arch/module-ownership-map.md`
- Codebase anchors:
  - `app/main.py`, `app/agentic_mvp/runtime/contracts.py`
  - `app/services/*` (52 services — most are domain, not C1; the C1 candidates are auth, lifecycle, db client, scheduler)
  - Postgres `public.checkpoints, checkpoint_blobs, checkpoint_writes` (LangGraph)
  - `app schema: agent_runs, agent_run_events, bot_tasks, batches, ...`

External research informing this doc:
- [Supabase RLS docs](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [LangSmith end-to-end OTel](https://docs.langchain.com/langsmith/trace-with-opentelemetry)
- [OTel GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/)
- [DPDP Rules 2025 (PIB)](https://static.pib.gov.in/WriteReadData/specificdocs/documents/2025/nov/doc20251117695301.pdf)
- [DPDP cross-border requirements](https://securiti.ai/cross-border-data-transfer-requirements-under-india-dpdpa/)
- [Langfuse self-hosted OTel](https://langfuse.com/integrations/native/opentelemetry)
- [OpenFeature Python SDK](https://openfeature.dev/docs/reference/sdks/server/python/)
- [aiobreaker (asyncio circuit breaker)](https://pypi.org/project/aiobreaker/)
- [Tenacity retry](https://tessl.io/registry/tessl/pypi-tenacity/9.1.0/files/docs/async-support.md)
