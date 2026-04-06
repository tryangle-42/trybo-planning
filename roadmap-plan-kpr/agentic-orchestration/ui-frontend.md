# UI / Frontend — Baseline Definition

> Owner: AKN
> Scope: Component 10 — 7 subcomponents (10.1–10.7)
> Module mapping: UI / Frontend
> Current completion: ~63% — 48 of 95 capabilities exist, 12 partial, 35 missing
> Source: SUBCOMPONENT_CAPABILITY_MAP.md

---

## What This Is

The face of the system. Zero intelligence of its own, total fidelity to what the other components are doing. The UI renders agent execution in real time, manages voice calls, provides contact and task management, handles settings, notifications, and consent flows, and wraps everything in an authenticated app shell.

The core screens work. The gaps are in polish (recording playback, error recovery, artifact rendering), operational features (analytics, notification inbox), and platform expansion (dark mode, i18n, accessibility).

## Current State

| What Exists | What's Missing |
|---|---|
| Agentic chat with SSE streaming, plans, graphs, approvals | Complete artifact rendering by type |
| Call logs, dialer, in-call controls, transcripts | Call recording playback |
| Contact list with search, sync, quick actions | Unified contact detail (all channels) |
| Task creation, history, detail, scheduling, restart | (task dashboard baseline met) |
| Settings: profile, DND, voice, Google, scheduling | (settings baseline met) |
| Push + toast + OTP + consent overlays | Complete notification type registry |
| Auth, onboarding, tabs, permissions, theme, offline | (app shell baseline met) |

## Baseline Definition

**The line is:** Every major user workflow (chat with agent, make calls, manage contacts, create tasks, configure settings, receive notifications) can be completed end-to-end without hitting a dead end — including error recovery when things fail and playback of recordings that are already being stored. Anything beyond this is refinement.

---

## Baseline Components

### 10.1 Agentic Chat Interface — ~4.5 dev days

**What this is:** The primary interaction surface. User sends a natural language message and watches the agent execute in real time via SSE streaming. The backend emits typed events and this component reduces them into visual state: execution graphs, plan steps, approval cards, clarification prompts, and rendered artifacts.

#### Baseline deliverables

**1. Complete artifact rendering by type**
Extend `FormattedText.tsx` to an artifact type registry: `{comparison_table: TableRenderer, summary: SummaryCard, booking: BookingCard}` with markdown fallback. Why baseline: the backend already emits typed artifacts, but the UI only renders markdown. A comparison table in raw markdown is unreadable. The agent's output quality is limited by what the UI can display.

**2. Error recovery UI**
On `step.failed` event → error card with `{error_message}` + action buttons: Retry / Skip / Abort. Why baseline: without error recovery, a failed step means the entire run is stuck. The user can only abandon and restart from scratch — the most common source of user frustration.

| Refinement (deferred) | Why |
|---|---|
| Streaming partial results | Results arrive on step completion; incremental is perceived-speed |
| Cost/token indicator | Power-user interest, not blocking |
| Inline feedback collection | Valuable for model improvement, not for task completion |
| Input templates/shortcuts | Convenience; typing works |
| Offline input queuing | Edge case; most users online |
| Cross-platform voice input | iOS works; Android is a gap |

**Dependencies:** Backend SSE events (exists), artifact type definitions (backend)

---

### 10.2 Voice Call Console — ~2 dev days

**What this is:** The call experience: initiating outbound calls, handling inbound (including LiveKit transfers), in-call controls, call logs with filtering, conversation detail with transcripts, and post-call review.

#### Baseline deliverables

**1. Call recording playback**
Audio player: load recording URL from call record → `expo-av` playback → timeline slider → play/pause → speed control (1x, 1.5x, 2x). Why baseline: recordings are already stored (Voice Pipeline 7.2). But users can't listen to them. The recording feature is useless without playback.

| Refinement (deferred) | Why |
|---|---|
| Real-time transcript during live call | Full transcript available post-call |
| Visual voicemail | Backend doesn't support voicemail yet |
| Post-call action prompt | Users can navigate to follow-up manually |

**Dependencies:** Call recordings with signed URLs (7.2 Voice Pipeline, exists)

---

### 10.3 Contact Manager — ~2 dev days

**What this is:** UI for browsing, searching, and managing the user's contact graph. Handles device sync, contact detail with interaction history, quick actions (call, message, task), and search across contacts + transcripts.

#### Baseline deliverables

**1. Complete contact detail with unified timeline**
Extend `ConversationDetailScreen` to show calls + WhatsApp + tasks chronologically per contact (not just calls). Requires the cross-channel `interactions` table from 8.4. Why baseline: a contact detail screen that only shows calls gives an incomplete picture.

| Refinement (deferred) | Why |
|---|---|
| Groups and tags | Batch operations are product features |
| Relationship metadata editing | Backend (8.3) needed first; read-only sufficient |
| Strength visualization | Interesting but not actionable |
| Per-contact preferences | Backend (8.3) needed first |
| Import/export | Power-user feature |
| Shared org contacts | B2B feature |

**Dependencies:** Cross-channel interactions table (8.4), Relationship metadata (8.3)

---

### 10.4 Task Dashboard — 0 dev days (BASELINE MET)

**What this is:** Task creation (natural language with @mentions), task history with 3-tab filtering, task detail with WhatsApp threads, scheduling, and restart. The operational control plane for delegated work.

All 5 core capabilities exist. Search/filtering is partial but functional.

| Refinement (deferred) | Why |
|---|---|
| Task templates | Convenience; typing works |
| Batch management | Low volume; individual management acceptable |
| Recurring tasks | One-shot covers launch |
| Analytics | Product metrics |
| Progress indicator | Status-based tracking sufficient |

---

### 10.5 Settings & Account — 0 dev days (BASELINE MET)

**What this is:** Centralizes all user configuration: profile, DND, voice cloning, integrations, task scheduling, and account management. Preferences set here propagate to all backend components.

All 8 core capabilities exist.

| Refinement (deferred) | Why |
|---|---|
| Notification preferences per category | Ships with Notification Service (1.10) |
| Dark mode | Visual preference, not functional |
| Language/locale | English-only at launch |
| Approval policies | Manual approval works |
| Data export, connected accounts | Polish features |

---

### 10.6 Notification & Consent Layer — ~1 dev day

**What this is:** Cross-cutting overlay for push notifications, in-app toasts, consent/approval overlays (OTP, data access consent), and notification lifecycle. Sits above all other screens.

#### Baseline deliverables

**1. Complete extensible notification type registry**
Refactor 4 hardcoded types to a registry: `notificationHandlers[type] = {screen, icon, sound, actions}`. New types registered declaratively. Why baseline: adding a new notification type currently requires modifying a switch/case in multiple files.

| Refinement (deferred) | Why |
|---|---|
| Notification inbox | OS notification center works |
| Action buttons on push | Opening the app works |
| Grouping/batching | Low volume at launch |
| Badge count | Functional without it |
| Per-category preferences | Ships with Settings + Notification Service |

---

### 10.7 App Shell & Navigation — 0 dev days (BASELINE MET)

**What this is:** The container: authentication flow, onboarding sequence, bottom tab navigation, design system, offline caching, feature flags, and analytics. Zero business logic.

All 8 core shell capabilities exist.

| Refinement (deferred) | Why |
|---|---|
| Dark mode | Visual preference; light works |
| Accessibility | Important for compliance, not blocking launch |
| i18n / multi-language | English-only at launch |
| Crash reporting (Sentry) | Firebase Crashlytics covers basics |
| Multi-platform (web) | Mobile-first; web is separate |

---

## What Counts as Refinement (explicitly deferred)

| Category | Examples | Why deferred |
|---|---|---|
| Chat polish | Partial results, cost indicator, feedback, templates, offline | Core chat workflow works |
| Call enhancements | Live transcript, voicemail, post-call prompts | Post-call review works |
| Contact depth | Groups, tags, metadata editing, import/export | Basic CRUD + search works |
| Task features | Templates, batch, recurring, analytics, progress | Core task workflow works |
| Platform | Dark mode, accessibility, i18n, web, crash reporting | Mobile-first English launch |
| Notifications | Inbox, action buttons, grouping, badges | OS notifications + toast work |

## Dependencies

| Needs | From | Why |
|---|---|---|
| Cross-channel interactions table | User Memory 8.4 | Unified contact timeline |
| Relationship metadata | User Memory 8.3 | Role/type display on contacts |
| Backend SSE events | Exists | Drives all chat UI state |
| Call recordings | Skills 7.2 | Playback source |

## Effort Estimate

| Module | Effort |
|---|---|
| 10.1 Agentic Chat Interface | 4.5 days |
| 10.2 Voice Call Console | 2 days |
| 10.3 Contact Manager | 2 days |
| 10.4 Task Dashboard | 0 days (baseline met) |
| 10.5 Settings & Account | 0 days (baseline met) |
| 10.6 Notification & Consent Layer | 1 day |
| 10.7 App Shell & Navigation | 0 days (baseline met) |
| **Total** | **~9.5 dev days** |

---
---

# CROSS-COMPONENT SUMMARY

## Overall Baseline Effort (All 4 Components)

| Component | Modules at Baseline | Modules Needing Work | Total Effort |
|---|:---:|:---:|---|
| 1. Barebone Platform | 2 of 12 | 10 | ~36 dev days |
| 7. First Party Skills | 2 of 5 | 3 | ~4.5 dev days |
| 8. User Memory | 0 of 6 | 6 | ~19.5 dev days |
| 10. UI / Frontend | 3 of 7 | 4 | ~9.5 dev days |
| **Total** | **7 of 30** | **23** | **~69.5 dev days** |

## Modules Already at Baseline (no work needed)
1. **1.6 Config Registry** — all config loaded, validated, typed, environment-aware
2. **1.9 Job Queue** — enqueue, process, retry, DLQ, schedule, parallel
3. **7.2 Voice Pipeline** — full audio round-trip with recording and transcripts
4. **7.3 Conversation Engine** — converse, classify intent, summarize
5. **10.4 Task Dashboard** — create, schedule, track, review, restart tasks
6. **10.5 Settings & Account** — all core settings capabilities
7. **10.7 App Shell & Navigation** — auth, onboarding, navigation, design system

## Module That Doesn't Justify Its Own Baseline
- **8.6 Behavioral Learner** — refinement of 8.2 Preference Store. Only observation hooks (1 dev day) at baseline.

## Critical Path (build order)

```
Phase 1 — Platform Foundation (parallel tracks)
├── 1.1  System Lifecycle (readiness, health)                    2.5 days
├── 1.12 HTTP Client (new module, unblocks everything)           8.5 days
├── 1.5  Auth Gateway (rate limiting)                            2.5 days
├── 1.7  Protocol Bridge (webhook dedup, validation)             3.5 days
└── 1.8  Data Store (transactions)                               2 days

Phase 2 — Platform Services (depends on Phase 1)
├── 1.2  Event Bus (fan-out, replay, DLQ)                        3.5 days
├── 1.3  Checkpoint Store (distributed locking, idempotency)     2.5 days
├── 1.4  Session Manager (Redis migration, lifecycle events)     3 days
├── 1.10 Notification Service (logging, preferences)             2.5 days
└── 1.11 LLM Gateway (multi-provider, fallback, routing)         5.5 days

Phase 3 — Skills + Memory (depends on Phase 2)
├── 7.1  Call Lifecycle (retry policies)                         1.5 days
├── 7.4  WhatsApp Engine (24h window, retry)                     2 days
├── 7.5  Web Research Engine (caching)                           1 day
├── 8.1  Identity Store (context fields, settings)               3.5 days
├── 8.2  Preference Store (new module)                           3 days
├── 8.3  Relationship Store (metadata, interaction stats)        3 days
└── 8.4  Interaction History (pending items, cross-channel)      5 days

Phase 4 — Integration + UI (depends on Phase 3)
├── 8.5  Contextual Recall (resolution pipeline, context bundle) 4 days
├── 8.6  Behavioral Learner (hooks only)                         1 day
├── 10.1 Agentic Chat Interface (artifacts, error recovery)      4.5 days
├── 10.2 Voice Call Console (recording playback)                 2 days
├── 10.3 Contact Manager (unified timeline)                      2 days
└── 10.6 Notification & Consent Layer (type registry)            1 day
```

## Key Overlap / Ownership Boundaries

| Overlap Area | Module A Owns | Module B Owns | Interface |
|---|---|---|---|
| Event delivery | 1.2 Event Bus: backend-to-backend | 1.10 Notification: backend-to-device | Event Bus publishes; Notification subscribes and translates to push |
| Rate limiting | 1.5 Auth Gateway: per-user inbound | 1.12 HTTP Client: per-provider outbound | No overlap |
| Session vs checkpoint | 1.4 Session: hot, volatile (TTL) | 1.3 Checkpoint: cold, persisted | Sessions expire; checkpoints are explicitly saved |
| User data | 8.1 Identity: who the user IS | 8.2 Preference: what the user LIKES | Identity anchors; Preferences overlay |
| Contact data | 8.3 Relationship Store: backend model | 10.3 Contact Manager: UI | Store owns CRUD + resolution; Manager renders |
| Preferences | 8.2 Preference Store: storage | 8.5 Contextual Recall: cross-store retrieval | Store stores; Recall reads and assembles |
| Notification UX | 1.10 Notification Service: delivery | 10.6 Notification Layer: display | Backend sends; UI renders |
