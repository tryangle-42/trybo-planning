# Skill Management — Registry + Indexing Baseline

> Owner: AKN
> Module mapping: M2 (Skill Management)
> Current completion: ~17% (registry + contracts done; hierarchy, discovery, health, versioning, testing all missing)

---

## What This Is

The system's catalog of everything it can do. Owns three things:

1. **Registry** — the source of truth for all skills (seed-registered, composed, auto-generated, third-party). A skill doesn't exist unless it's here.
2. **Index** — embedding-based search so the planner finds the right skill in <50ms from 500+ skills, without the full catalog in its prompt.
3. **Health** — live per-skill success rate and latency. Broken skills are auto-excluded.

**What this module does NOT own:** discovering new skills, generating skills from API specs, or validating untrusted skills. That's New Skill Creation (M6). This module provides the `register()` endpoint that M6 calls after a skill passes all quality gates.

## Current State

| What Exists | What's Missing |
|-------------|----------------|
| `seed_registry.py` — 70+ skill definitions with full metadata | Flat list, no hierarchy/taxonomy |
| `SkillsRegistry` class with lookup by ID and capabilities | No embedding-based search |
| `SkillDefinition` schema (contracts, schemas, side_effect flags) | No health metrics per skill |
| Planner receives full skill catalog in prompt | Won't scale past ~100 skills |
| No dynamic registration — restart required for new skills | |

---

## Baseline Definition

**The line:** The planner finds the right skill from 500+ skills in <100ms without the full catalog in its prompt. Every skill has live health metrics. New skills can be registered without restart. Anything beyond this is refinement.

---

## Baseline Components

### 1. Embedding Index (pgvector)

Every skill gets converted into a numerical "fingerprint" (an embedding) and stored in Postgres. When a user says "Call John about the meeting," the system converts that sentence into the same kind of fingerprint and finds the skills whose fingerprints are closest — semantically most similar.

A `skill_index` table stores each skill's metadata (name, description, namespace, health status, success rate, latency) alongside its embedding vector.

**Why this approach:**
- Trybo already uses Supabase (Postgres). pgvector adds vector search to the same DB — no new infrastructure.
- Hybrid queries combine "find similar skills" with "but only healthy ones in the communication category" in a single query.
- The HNSW index makes lookups O(log n). At 500 skills: <50ms.

**Why `full_text` matters:** Research (SkillRouter, 2026) found that indexing just the skill name and description gives mediocre retrieval accuracy. Including the handler's docstring and parameter names improves accuracy by 29-44%.

### 2. Skill Hierarchy (Capability Tree)

Skills organized into a taxonomy so the planner reasons at the capability level:

```
root
├── communication
│   ├── voice (call.initiate, call.answer, call.transfer)
│   ├── messaging (whatsapp.send, sms.send, slack.post)
│   ├── email (email.send, email.search, email.summarize)
│   └── notifications (push.send, reminder.set)
├── information
│   ├── web_search (search.web, search.news)
│   ├── document (doc.parse, doc.summarize)
│   └── data_lookup (weather.get, finance.stock)
├── scheduling
│   ├── calendar (calendar.create, calendar.query)
│   └── tasks (task.create, task.update)
├── contacts
│   └── lookup (contact.find, contact.details)
└── system
    ├── memory (memory.search, memory.store)
    └── preferences (preference.get, preference.set)
```

Implemented as a `namespace` field on each skill. GID uses top-level categories for intent classification. The planner's catalog is dynamically assembled from the relevant namespace(s).

**This hierarchy also doubles as the provider grouping.** When multiple skills exist under the same leaf (e.g., `twilio.call.initiate` and `knowlarity.call.initiate` both under `voice`), retrieval returns them sorted by health metrics — the healthiest provider comes first. No separate ranking module needed at baseline (see `agent-rank-and-allot.md` for why).

### 3. Health Metrics (Live)

Every skill invocation updates rolling health using an Exponential Moving Average (EMA).

**Why EMA instead of a simple average?** A simple average treats a failure from last month the same as one from 5 minutes ago. EMA with `alpha = 0.1` means the last ~10 invocations dominate. If Tavily starts failing *right now*, its score drops within minutes.

**Auto-degradation thresholds:**

| Success Rate | Status | Effect |
|-------------|--------|--------|
| > 50% | `active` | Normal — appears in planner catalog |
| 20–50% | `degraded` | Available as fallback, never primary |
| < 20% | `disabled` | Excluded from catalog entirely |
| Returns above 80% after degraded | `active` | Auto-recovered |

Retrieval always filters by `health_status = 'active'`. Broken skills never reach the planner.

### 4. Retrieval API

A `find_skills(query, namespace?, top_k=5)` function replaces injecting the full catalog into the planner prompt. Embeds the query, searches pgvector (filtered by namespace + health), returns top 5.

**Two approaches by scale:**

| Approach | How | Best when |
|----------|-----|-----------|
| **API call** (baseline) | Pipeline calls `find_skills` before planning. Planner receives 5 skills, not 70. | < 200 skills |
| **Meta-tool** (Composio pattern) | `find_skills` is exposed as a skill itself. Planner calls it first, gets results, then plans. | > 200 skills |

The meta-tool pattern is how Composio handles 1000+ tools: the agent never sees the full catalog. It searches, gets results, then acts.

### 5. Registration Endpoint

An admin API that accepts a validated `SkillDefinition` and adds it to the live registry + embedding index without restart. This is the interface that New Skill Creation (M6) calls after a skill passes quality gates.

Registration behaves differently by skill type:

| Skill type | What happens | Runtime cost |
|-----------|-------------|-------------|
| Direct API / Composio | Store definition + embedding. Done. | None |
| MCP | Store definition + embedding + reinitialize MCP client | Brief blip (~seconds) — all MCP connections drop and reconnect |

The MCP blip is acceptable: new MCP servers are added weekly at most. Building a proper dynamic `add_server()` is a refinement.

---

## What Counts as Refinement

| Refinement | Why deferred |
|------------|-------------|
| Cross-encoder reranking (see Key Concepts) | Bi-encoder handles up to ~500 skills accurately |
| Full capability tree traversal (AgentSkillOS-style) | Namespace filtering sufficient at baseline |
| Skill versioning (canary rollout, rollback) | Single version per skill is fine for GTM |
| Skill caching (memoize idempotent results) | Not needed at current scale |
| Skill documentation generation | Descriptions in registry are sufficient |
| Skill deprecation lifecycle | Small catalog. Manual removal works. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| pgvector extension in Supabase | DevOps | Embedding storage + HNSW index |
| `sentence-transformers` in requirements | AKN | Embedding model |
| M5 finalize hook for telemetry | Execution Engine (RKP) | Health metrics updated after each invocation |
| Admin auth middleware | Security (M11) | Registration endpoint gated |

## Effort Estimate

| Component | Effort |
|-----------|--------|
| `skill_index` table + pgvector setup | 1 day |
| Embedding pipeline (embed all 70+ skills) | 1 day |
| `find_skills` retrieval API | 1 day |
| Skill hierarchy (namespace taxonomy + config) | 1 day |
| Health metrics (EMA + auto-degradation) | 1-2 days |
| Registration endpoint (hot-add) | 1 day |

**Total: ~6-8 dev days to reach baseline.**

---

## Key Concepts Explained

**What is an embedding?**
A way to convert text into a list of numbers (a "vector") such that similar meanings produce similar numbers. "Call John" and "Phone John" produce vectors that are close together. "Send an email" produces one that's far away. This lets the system do math on meaning — "find the 5 skills closest in meaning to this user request."

**Bi-encoder vs Cross-encoder (and why we defer the cross-encoder)**

A bi-encoder pre-computes a fingerprint for every skill, then compares fingerprints:
- Embed "Call John about meeting" → vector A
- Embed "voice.call.initiate: Make phone calls" → vector B (pre-computed, stored)
- Similarity(A, B) → 0.82
- Fast — O(1) lookup via HNSW index.

A cross-encoder feeds *both texts into one model simultaneously*, catching nuances like "Call" meaning phone call vs API call:
- Score("Call John about meeting", "voice.call.initiate") → 0.94
- Score("Call John about meeting", "email.send") → 0.31
- Much more accurate, but O(n) — must run the model on every (query, skill) pair.

At baseline, the bi-encoder alone is accurate enough. Cross-encoder reranking is a refinement for 500+ skills.

**What is pgvector / HNSW?**
pgvector is a Postgres extension for storing and searching vectors. HNSW (Hierarchical Navigable Small World) is its index — a graph where similar items are connected. Instead of comparing against every skill (O(n)), it navigates the graph in O(log n). At 500 skills: <5ms lookup.

---

## Research References

| Topic | Source |
|-------|--------|
| Embedding-based skill retrieval | Semantic Tool Discovery for MCP (arXiv 2603.20313) — 97.1% hit rate at K=3 |
| Skill body as decisive signal | SkillRouter (arXiv 2603.22455) — removing body causes 29-44% degradation |
| Capability tree at scale | AgentSkillOS (arXiv 2603.02176) — tested 200 to 200K skills |
| Meta-tool pattern | Composio architecture — `COMPOSIO_SEARCH_TOOLS` |
