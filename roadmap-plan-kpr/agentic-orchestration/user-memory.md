# User Memory — Baseline Definition

> Owner: AKN
> Scope: Component 8 — 6 subcomponents (8.1–8.6)
> Module mapping: User Memory (Data Layer + Intelligence Layer)
> Current completion: ~13% — 8 of 60 capabilities exist, 7 partial, 45 missing
> Source: SUBCOMPONENT_CAPABILITY_MAP.md

---

## What This Is

Everything the system knows about the human, accumulated across all interactions. This is the relationship layer — the difference between an agent that starts from zero every time and one that remembers who you are, what you like, who you know, and what happened last time.

User Memory has two layers: a **Data Layer** (stores facts: identity, preferences, relationships, history) and an **Intelligence Layer** (reads data, writes insights: contextual recall, behavioral learning).

This is the least-built component. The Data Layer needs to be established from scratch for preferences and enriched for relationships/history. The Intelligence Layer depends entirely on the Data Layer being in place.

## Current State

| What Exists | What's Missing |
|---|---|
| Core user profile (name, phone, avatar, org) | Formalized context fields (timezone, language typed columns) |
| OAuth tokens (encrypted, refresh) | Strongly typed settings with categories |
| Basic contacts with name resolution | Relationship metadata (role, type, org) |
| Chat + call transcript history | Per-contact interaction summaries |
| LLM-generated session summaries | Pending item tracking + surfacing |
| Partial contact resolution (name match) | Cross-channel continuity |
| | Preference Store (entirely new) |
| | Resolution pipeline + context bundle |
| | Behavioral observation hooks |

## Baseline Definition

**The line is:** The system knows who the user is (typed identity), what they prefer (structured preferences with hierarchy), who they know (contacts with roles and interaction stats), what happened before (cross-channel history with pending items), and can assemble all of this into a single context bundle for Planning. Anything beyond this is refinement.

---

## Baseline Components

### 8.1 Identity Store — ~3.5 dev days

**What this is:** The canonical user profile — the single record that anchors all other user-scoped data. Stores core identity, manages OAuth tokens, and maintains context fields. All other User Memory subcomponents reference this as their anchor.

#### Baseline deliverables

**1. Formalize context fields**
Add typed columns `timezone`, `language`, `location_jsonb` to `user_profiles`. Migrate from ad-hoc `settings_jsonb`. Why baseline: every component that personalizes (scheduling uses timezone, TTS uses language, research uses location) currently parses a JSON blob and hopes the field exists. Typed columns enforce consistency.

**2. Strongly typed settings with category namespaces**
`user_settings` table: `{user_id, category, key, value_jsonb, updated_at}`. Pydantic model per category. Access via `settings.get("voice.provider")`. Why baseline: user settings are scattered across multiple tables and JSON blobs. A single settings API is needed before Preference Store (8.2) can layer on top.

| Refinement (deferred) | Why |
|---|---|
| Multi-org identity | Single-org sufficient; multi-org for B2B |
| Profile versioning | Audit feature, not functional |
| Identity verification (KYC) | Not needed for current users |
| Complete federation / device registry | Google covers primary use case |

**Dependencies:** Data Store (1.8), Auth Gateway (1.5)

---

### 8.2 Preference Store — ~3 dev days

**What this is:** Stores user preferences (seat type, budget, meeting time, voice provider) as structured records with category, key, value, source (explicit vs inferred), and confidence score. Follows a hierarchy: explicit overrides inferred, which overrides team defaults, which overrides org defaults. Entirely new module.

#### Baseline deliverables

**1. Preference table and CRUD API**
`user_preferences` table: `{user_id, category, key, value_jsonb, source, confidence, created_at, updated_at}`. Source enum: `user_declared`, `inferred`, `team_default`, `org_default`. This is the foundation — nothing else in User Memory works well without structured preferences.

**2. Category-based retrieval**
Categories: `travel`, `communication`, `food`, `scheduling`, `voice`, etc. Query: `WHERE user_id = ? AND category IN (?)`. When a task arrives ("book a flight"), pull all `travel` preferences. Without categories, the LLM must filter all preferences itself.

**3. Hierarchy resolution**
Multiple values for same `{category, key}` → resolve by source priority: `user_declared (1.0) → inferred (0.3-0.9) → team_default → org_default`. Return highest-confidence match. This is what makes "my usual" work.

| Refinement (deferred) | Why |
|---|---|
| Temporal preferences (valid_from/until, recurrence) | Complex; explicit covers 90% of cases |
| Override history tracking | Nice for undo, not for function |
| Conflict detection between sources | Manual resolution acceptable |
| Cross-channel sync, expiry, bulk import/export | Polish and power-user features |

**Dependencies:** Identity Store (8.1), Data Store (1.8)

---

### 8.3 Relationship Store — ~3 dev days

**What this is:** Extends a basic contact list with relationship metadata (role, type, org) and per-contact interaction summaries (last contacted, frequency, topics). This is what turns "call my manager" from an unresolvable reference into a concrete action with a specific phone number.

#### Baseline deliverables

**1. Relationship metadata on contacts**
Add columns to `contacts`: `relationship_type` (friend/colleague/family/vendor), `role` (manager/assistant/direct_report), `org`, `notes_text`. Why baseline: "my manager" requires role-based lookup. Name match alone can't resolve relationship references.

**2. Per-contact interaction summaries**
`contact_interaction_stats` table: `{contact_id, last_contacted_at, call_count, message_count, recent_topics_jsonb}`. Updated after each interaction. Why baseline: "when did I last talk to X?" and "what did we discuss?" are the two most common relationship queries.

| Refinement (deferred) | Why |
|---|---|
| Contact groups and tags | Batch operations are product features |
| Relationship strength scoring | Name + role resolution sufficient |
| Communication preference per contact | Default to user's channel |
| Shared org contacts | Single-user until B2B |

**Dependencies:** Contacts table (exists in Data Store), Event Bus (1.2)

---

### 8.4 Interaction History — ~5 dev days

**What this is:** Stores episodic memory across all channels — timestamped records of what happened, what was discussed, and what was left unfinished. Enables the system to resume conversations rather than restart from zero.

#### Baseline deliverables

**1. Pending item extraction and tracking**
`pending_items` table: `{user_id, contact_id, item_description, source_run_id, context_jsonb, created_at, resolved_at}`. Extracted from call summaries by the LLM. Why baseline: the most frustrating UX is repeating information the system should already know.

**2. Surface incomplete items on new task overlap**
On new task: query `pending_items WHERE contact_id = ? AND resolved_at IS NULL`. If overlap: include in LLM context ("You have an unfinished X with this contact"). This is the payoff of tracking pending items.

**3. Cross-channel continuity**
Unified `interactions` table: `{user_id, contact_id, channel, session_id, summary, created_at}`. Query by contact_id returns all channels chronologically. Why baseline: a user who chats, calls, then WhatsApps about the same topic currently has three disconnected histories.

| Refinement (deferred) | Why |
|---|---|
| Semantic search (embeddings) | Keyword + recent history covers most cases |
| Topic threading | Per-contact chronological sufficient |
| Unified timeline API | Data exists; API is UI convenience |
| Retention policies | Data volume manageable |

**Dependencies:** Data Store (1.8), Conversation Engine (7.3), Event Bus (1.2)

---

### 8.5 Contextual Recall — ~4 dev days

**What this is:** The cross-store retrieval layer. When a user says "my usual hotel" or "call my manager," the reference points to data scattered across Identity, Preferences, Relationships, and History. Contextual Recall parses references, queries the appropriate stores, and assembles a complete context bundle for Planning.

#### Baseline deliverables

**1. Resolution pipeline**
Sequential resolver: parse reference type → dispatch to appropriate store → collect results. `ContextualRecall.resolve(query, user_id)` runs: Identity → Preferences → Relationships → History. Prevents every component from duplicating reference resolution.

**2. Contact resolution by role**
Extend `resolve_contact_from_prompt()` for role-based lookup: `contacts WHERE role = 'manager'`. Fall back to name match. The difference between resolving "Priya" (name) and "my manager" (role).

**3. Preference resolution**
Query Preference Store: `WHERE category = ? AND user_id = ?`, return highest-confidence match. "My usual hotel" → category "travel", key "hotel" → stored preference.

**4. Context bundle assembly**
`ContextBundle` dataclass: `{user_profile, resolved_contacts, preferences, recent_history}`. Passed to Planning as one object. Planning currently receives raw messages and figures out context on its own.

| Refinement (deferred) | Why |
|---|---|
| Ambiguity handling with confidence | Return best match; disambiguation is UX polish |
| Semantic similarity matching | Keyword/exact covers most references |
| Multi-hop resolution | Rare; single-hop covers 95% |
| Temporal resolution | LLM handles dates adequately |
| Resolution caching | Fast enough without caching |

**Important:** Cannot be built until 8.1, 8.2, 8.3, and 8.4 are at baseline. It is a composition layer over the data stores.

**Dependencies:** Identity Store (8.1), Preference Store (8.2), Relationship Store (8.3), Interaction History (8.4)

---

### 8.6 Behavioral Learner — ~1 dev day (hooks only)

**What this is:** Records observations after each interaction, detects patterns, creates inferred preferences, and promotes them to Preference Store. Entirely new module.

**The Behavioral Learner does not have its own baseline — it is entirely a refinement of the Preference Store (8.2).** Reasons:

1. **Preference Store must work first.** Inferred preferences are meaningless without storage/retrieval.
2. **Explicit preferences cover launch.** Users can state preferences directly. Behavioral learning is a nice-to-have.
3. **Pattern detection requires data volume.** Need 5+ observations per behavior. Users won't have this at launch.
4. **Wrong inferences are worse than none.** A wrong inferred preference damages trust.

**Recommendation:** Build observation recording hooks only (cap #1) so data accumulates. Defer pattern detection and inference engine.

| Refinement (the entire module) | Why |
|---|---|
| Observation recording | Pre-wire data collection |
| Pattern detection, scoring, promotion | Needs data volume + working Preference Store |
| Time-series, context-aware, confidence decay | Advanced ML territory |
| Learning boundaries | Depends on learner existing |

**Dependencies:** Preference Store (8.2), Interaction History (8.4), Event Bus (1.2)

---

## Build Order

```
8.1 Identity Store (foundation)
       ↓
8.2 Preference Store  +  8.3 Relationship Store  (parallel, both depend on 8.1)
       ↓                        ↓
              8.4 Interaction History (depends on 8.3 for contact stats)
                      ↓
              8.5 Contextual Recall (depends on all of the above)
                      ↓
              8.6 Behavioral Learner hooks (depends on 8.2 + 8.4)
```

## What Counts as Refinement (explicitly deferred)

| Category | Examples | Why deferred |
|---|---|---|
| Identity maturity | Multi-org, federation, KYC, profile versioning | Single-org, single-provider sufficient |
| Preference intelligence | Temporal prefs, conflict detection, expiry | Explicit preferences cover launch |
| Relationship depth | Groups, tags, strength scoring, shared contacts | Name + role resolution sufficient |
| History intelligence | Semantic search, topic threading, retention | Keyword + chronological sufficient |
| Recall sophistication | Ambiguity handling, multi-hop, semantic matching | Best-match sufficient |
| Behavioral learning | Pattern detection, inference, confidence decay | No data volume at launch |

## Dependencies

| Needs | From | Why |
|---|---|---|
| Data Store | Platform 1.8 | All new tables |
| Auth Gateway | Platform 1.5 | User identity resolution |
| Event Bus | Platform 1.2 | Lifecycle events, observation triggers |
| Conversation Engine | Skills 7.3 | Summary generation for pending items |

## Effort Estimate

| Module | Effort |
|---|---|
| 8.1 Identity Store | 3.5 days |
| 8.2 Preference Store | 3 days |
| 8.3 Relationship Store | 3 days |
| 8.4 Interaction History | 5 days |
| 8.5 Contextual Recall | 4 days |
| 8.6 Behavioral Learner (hooks only) | 1 day |
| **Total** | **~19.5 dev days** |
