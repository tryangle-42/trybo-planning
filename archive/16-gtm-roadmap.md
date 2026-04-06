# GTM Development Roadmap

> Maps every GTM gap to a module, assigns ownership, sequences into phases, and identifies dependencies.
>
> Source: `GTM-Gap-Analysis.pdf` (36 items), `15-module-ownership-map.md` (11 modules), team discussion notes.
>
> GTM scope: Communication (calls + WhatsApp) + Web Research + BYO Data, with agentic orchestration.
> Split: 60% GTM-focused, 40% long-term platform.

---

## Master Gap → Module Map

Every gap item from the analysis, mapped to modules and tagged. Ownership phrased by module domain — self-assign based on who owns which modules.

### GTM-Critical (launch blockers)

| # | Gap | Modules | Ownership Domain | Depends On | Notes |
|---|-----|---------|-----------------|------------|-------|
| 1 | All call flows through agentic orchestration | M5, M10, M3 | Orchestration + Comms | Planner must handle call intents; M10 handlers must be invokable from executor | Currently calls bypass the orchestration engine. This is the biggest architectural shift. |
| 3 | Outbound calls: consent request capability | M10, M11, M5 | Comms + Security | #1 (call flows must be agentic first) | Consent gate via `interrupt()` on outbound call skill |
| 4 | Consent request screen inside chat UI | M9, M11 | Frontend + Security | Backend consent API exists | Approval cards rendered inline in chat, not separate screen. Also covers outbound pending list. |
| 5 | Agent response latency < 2.5s during calls | M10, M3 | Comms + Orchestration | #1, #31 (fast path) | STT→LLM→TTS pipeline. Fastest win: fast path for simple call intents. |
| 10 | WhatsApp consent request + agentic brain | M10, M11, M5 | Comms + Orchestration | #1 (same agentic brain pattern) | Re-use orchestration for WhatsApp interactions. Filler injection for slow consent. |
| 14 | Task History: batch + task summary redesign | M9 | Frontend | M5 APIs for run/task data | Complete UI redesign. Paginated. Show session summary + planned tasks. |
| 22 | Call Logs: remove duplicates, generic consent list | M9 | Frontend | #4 (consent in chat) | Merge inbound+outbound consent into one list. Full UX revamp. |
| 27 | Fix call summaries ("Your AI Assistant Called...") | M10, M5 | Comms | — | Quick fix. Summary generation uses wrong template. |
| 28 | Episodic memory + semantic memory | M7 | Memory | — | Foundation for cross-session intelligence. Episodes table + mem0 wiring into planner. |
| 30 | Execution resilience: single node failure handling | M5 | Orchestration | — | Retry with backoff, fallback skills, proactive failure message in chat, restart from chat. |
| 31 | Planning quality: reliability, fast path, clarification, failure messaging | M3 | Orchestration | #28 (memory wiring improves plan quality) | Fast path for trivial intents. Benchmark plan success rate. Clear error UX. |

### GTM-Important (degrades experience if missing)

| # | Gap | Modules | Ownership Domain | Depends On | Notes |
|---|-----|---------|-----------------|------------|-------|
| 2 | STT/TTS model exploration (Sarvam, latest models) | M10 | Comms | — | Improve conversation coherence. English + Hindi/Hinglish baseline. |
| 6 | Reduce interaction latency (click, scroll, swipe) | M9 | Frontend | — | Animations, transitions, perceived performance. |
| 7 | Noise cancellation for WebRTC calls | M10 | Comms | — | Audio processing layer. Krisp SDK or similar. |
| 8 | Hold feature: template voice at intervals + early resume | M10 | Comms | — | Extend existing hold. "Please hold" template voice every N seconds. |
| 9 | Auto-WebRTC for Trybo-to-Trybo calls | M10 | Comms | — | Detect if callee is on Trybo → route via WebRTC instead of PSTN. Can also fetch info without calling. |
| 11 | Delete/recreate voice sample | M10, M9 | Comms (backend) + Frontend (UI) | — | Backend: delete voice profile API. Frontend: UI flow. |
| 12 | Voice input on Android | M9 | Frontend | — | Currently iOS only. Android speech-to-text integration. |
| 13 | Data sources + permissions: test seamlessness | M9, M1 | Frontend + Adapters | #17 (Google Integration) | End-to-end test of connecting Google, etc. |
| 17 | Fix Google Integration (Composio) | M1 | Adapters | — | Composio SDK for Google Workspace. Data gathering for calls/research. |
| 18 | File attachment in chat prompt (WA/email sharing) | M9, M10 | Frontend + Comms | — | Attach files → share via WhatsApp/email. Includes drafting message + attaching. |
| 20 | Voice Connect Card redesign | M9 | Frontend | — | Profile screen UI update. |
| 21 | Remove old create Task screen | M9 | Frontend | — | Cleanup. Chat-first approach. |
| 23 | Chat: rename/delete + copy messages | M9 | Frontend | — | Standard chat management features. |
| 26 | Task scheduling from chat + scheduled task UI | M3, M5, M9 | Orchestration (backend) + Frontend (UI) | — | "Remind me every morning" in chat → creates scheduled job. UI to browse/manage scheduled tasks. |
| 29 | Channel auto-selection ("contact John" → best channel) | M8 | Intelligence | #28 (needs user preferences from memory) | Rules engine: urgency, time of day, contact reachability, past success. |

### GTM-Deferred (retain or hold for now)

| # | Gap | Modules | Ownership Domain | Decision |
|---|-----|---------|-----------------|----------|
| 15 | DND handling for outbound | M5, M8 | Orchestration + Intelligence | Tasks continue normally. Consent shown in pending screen. Low likelihood for Trybo users. |
| 16 | Profile section task scheduling (old UI) | M9 | Frontend | **Hold.** Retain old UI alongside chat for beta. Learn from usage. Revisit post-beta. |
| 19 | Read voice attachments for call context | M7, M10 | Memory + Comms | Discuss if too complex for GTM. Audio→text→context pipeline. |
| 24 | Permission granting within consent flow | M9, M11 | Frontend + Security | Show popup for missing permissions only. Simple implementation. |
| 25 | Search within and across chats | M9 | Frontend | Nice-to-have. Can ship without for GTM launch. |

### Low Priority (post-GTM)

| # | Gap | Modules | Ownership Domain | Notes |
|---|-----|---------|-----------------|-------|
| 32 | Rate limiting for authorized users | M11 | Security | Prevent misuse (1000 calls via CSV). |
| 33 | Cost visibility | M11 | Security | Per-user cost tracking. Needed before pricing. |
| 34 | Bigger file upload + RAG (pgvector) | M7, M1 | Memory + Adapters | Move docs to pgvector for retrieval. |
| 35 | LiveKit SIP trunking for Twilio calls | M10 | Comms | Better calling screen UI. Needs investigation. |
| 36 | Live transcript + interactive chat control | M9, M10 | Frontend + Comms | Complex UX. Evaluate gain vs complexity. |

---

## Phase Plan

### Phase 0: Foundation Wiring (Week 1)

Goal: Connect the memory and orchestration plumbing that everything else depends on.

| Task | Domain | Module | What specifically |
|------|--------|--------|-------------------|
| Wire episodic memory | Memory | M7 | Create `execution_episodes` table. Generate episodes in finalize node. Query before planning. |
| Wire semantic memory (mem0) into planner | Memory | M7 | Inject mem0 results into planner prompt. Extract facts after execution. |
| Wire conversation buffer properly | Memory | M7 | Configurable N, include system messages, summarize older messages. |
| Agentic call flow: architecture spike | Comms + Orchestration | M5, M10 | Design how inbound/outbound calls route through GID → Planner → Executor. Identify what changes in call handlers. |
| Consent-in-chat: design + API contract | Frontend + Security | M9, M11 | Design approval cards for chat. Define API contract with backend (what data, what actions). |

**Exit criteria:** Planner receives conversation history + semantic memories + episode summaries. Call agentic architecture agreed. Consent-in-chat API contract locked.

---

### Phase 1: Agentic Calls + Planning Quality (Weeks 2-3)

Goal: The core product works — calls flow through the agent brain, plans are reliable, failures are handled.

**Orchestration (M3, M5):**

| Task | Gap # | What specifically |
|------|-------|-------------------|
| Fast path for trivial intents | 31 | GID routes "call John" directly to skill, bypasses planner. Sub-500ms. |
| Plan failure messaging | 31 | When planning fails, user sees a clear message in chat, not a generic error. |
| Clarification UX polish | 31 | "Which John?" flow: interruptible, user can correct, planner resumes. |
| Single node failure recovery | 30 | Retry with backoff. Fallback skill. Proactive failure message in chat. Restart from chat. |
| Context assembly engine | 28 | `assemble_for_planner()` → conversation buffer + top 3 episodes + top 5 memories + procedural summary. |
| Procedural memory surfacing | 28 | Planner sees "User has: voice profile, Google connected, 3 contacts" from skill_data_store. |

**Comms (M10, M5):**

| Task | Gap # | What specifically |
|------|-------|-------------------|
| Inbound calls through orchestration | 1 | Incoming call → GID → Planner → Executor (consent gate → call handler). |
| Outbound calls through orchestration | 1, 3 | "Call John about the meeting" → plan → consent → execute call. |
| Outbound call consent capability | 3 | Consent artifact for outbound calls. User approves in chat before call initiates. |
| WhatsApp through orchestration | 10 | WhatsApp interactions routed through same agentic brain. Consent + filler injection. |
| Fix call summaries | 27 | Summary template fix. Quick win. |
| Call latency < 2.5s | 5 | Profile STT→LLM→TTS pipeline. Optimize bottlenecks. |

**Frontend (M9):**

| Task | Gap # | What specifically |
|------|-------|-------------------|
| Consent cards in chat | 4 | Inline approval/edit/deny cards in chat thread. Both inbound and outbound. |
| Consent pending list (unified) | 4, 22 | Single list for all pending consent (inbound + outbound). Replace old to-respond tab. |
| Remove old create Task screen | 21 | Chat-first. Remove the legacy flow. |
| Call Logs screen cleanup | 22 | Remove duplicate tabs. Unified consent list. |
| Interaction latency / animations | 6 | Smooth transitions, reduce perceived glitchiness. |

**Exit criteria:** User can say "call John about the meeting" in chat → plan created → consent card shown → user approves → call executes → summary returned in chat. Same flow works for WhatsApp. Single node failures show clear message + retry option.

---

### Phase 2: Intelligence + UX Polish (Weeks 4-5)

Goal: The agent feels smart and the app feels polished.

**Intelligence + Orchestration (M8, M3):**

| Task | Gap # | What specifically |
|------|-------|-------------------|
| Channel auto-selection | 29 | "Contact John" → evaluate channels → pick best based on urgency, time, preferences. |
| Plan benchmarking | 31 | Test suite for common communication intents. Measure plan success rate. Target: >90%. |
| Task scheduling from chat | 26 | "Remind me every morning to check calendar" → scheduled job. Backend integration. |
| DND handling (design only) | 15 | Define behavior. Tasks continue; consent queued. Document for beta. |

**Comms + Adapters (M10, M1):**

| Task | Gap # | What specifically |
|------|-------|-------------------|
| STT/TTS model exploration | 2 | Evaluate Sarvam, latest models. English + Hindi/Hinglish. Benchmark latency + quality. |
| Noise cancellation for WebRTC | 7 | Integrate noise cancellation (Krisp SDK or equivalent). |
| Hold feature: template voice | 8 | "Please hold" at intervals during consent wait. Early resume capability. |
| Auto-WebRTC for Trybo-to-Trybo | 9 | Detect Trybo user → route via WebRTC. Explore fetching info without calling. |
| Delete/recreate voice sample (backend) | 11 | API to delete voice profile + re-record flow. |
| Google Integration fix (Composio) | 17 | Fix data gathering via Composio for Google Workspace. |

**Frontend (M9):**

| Task | Gap # | What specifically |
|------|-------|-------------------|
| Task History redesign | 14 | Batch + task summary. Session overview + planned ahead. Paginated. |
| Voice Connect Card redesign | 20 | Profile screen update. |
| Chat rename/delete + copy | 23 | Standard chat management. |
| Voice input on Android | 12 | Speech-to-text integration for Android. |
| File attachment in chat | 18 | Attach files → share via WA/email with drafted message. |
| Delete/recreate voice sample (UI) | 11 | Frontend flow for voice profile management. |
| Scheduled task UI | 26 | Browse/manage scheduled tasks. Explore Openclaw/Claude-like UX. |

**Exit criteria:** "Contact John" picks the best channel automatically. Scheduled tasks work from chat. Voice quality noticeably improved. Core UI polished and consistent.

---

### Phase 3: Hardening + Beta Prep (Week 6)

Goal: Production-ready for beta users.

| Task | Domain | Gap # | What specifically |
|------|--------|-------|-------------------|
| End-to-end test suite | All | — | 10 core flows tested end-to-end: inbound call, outbound call, WhatsApp outbound, consent flows, scheduled task, multi-step plan, failure recovery, memory recall, file share, search. |
| Data source permissions testing | Frontend | 13 | Test Google, file upload, web research data flows for seamlessness. |
| Permission popup in consent flow | Frontend + Security | 24 | Show popup for missing permissions during consent. |
| Search within/across chats | Frontend | 25 | If time permits. |
| Feature flags for staged rollout | Orchestration | — | Developer-only flags for: agentic calls, channel auto-selection, memory layers. |
| Rate limiting for authorized users | Security | 32 | Basic per-user limits to prevent misuse. |
| Monitoring + alerting setup | Orchestration | — | Key metrics: plan success rate, call latency, failure rate, memory hit rate. |

---

## Dependency Chain (critical path)

```
Memory Wiring (Phase 0)
    │
    ├── Context assembly ────── Planning quality improvements
    │                               │
    │                               ├── Fast path (latency)
    │                               └── Plan benchmarking
    │
    └── Episodic memory ─────── Channel auto-selection
                                    │
                                    └── (needs preference history)

Agentic Call Architecture (Phase 0 spike)
    │
    ├── Inbound calls through orchestration ──┐
    ├── Outbound calls through orchestration ──┼── Call latency optimization
    ├── Outbound call consent ─────────────────┘
    │
    └── WhatsApp through orchestration
            │
            └── WhatsApp consent + filler injection

Consent-in-Chat API (Phase 0 design)
    │
    ├── Consent cards in chat ──── Unified consent list
    │
    └── Outbound consent UI ───── Call/WhatsApp consent flows
```

**The critical path is:** Memory wiring → Context assembly → Planning quality → Fast path → Call latency target met.

If memory wiring slips, planning quality suffers, which means call latency suffers, which means the core GTM experience is degraded.

---

## 60/40 Split Mapping

### The 60% (GTM-focused)

Work that directly serves the communication + research GTM:

| Module | GTM Work |
|--------|----------|
| M3 | Fast path, clarification UX, plan failure messaging, plan benchmarking |
| M5 | Single node failure recovery, run restart from chat, agentic call/WhatsApp routing |
| M7 | Conversation buffer improvements, episodic memory, mem0 wiring, context assembly |
| M8 | Channel auto-selection |
| M9 | Consent in chat, task history redesign, call logs cleanup, chat management, voice input Android, file attachments, scheduled task UI, interaction polish |
| M10 | Agentic call flows (inbound + outbound), WhatsApp agentic, consent for calls/WhatsApp, call summaries fix, latency optimization, STT/TTS models, noise cancellation, hold improvements, auto-WebRTC, voice sample management, Google integration |
| M11 | Basic rate limiting |

### The 40% (long-term platform, but still mapped)

| Module | Platform Work | GTM-Relevant? |
|--------|--------------|----------------|
| M2 | Skill hierarchy, versioning, discovery, health metrics, testing framework | Partially — skill health metrics help debug GTM failures |
| M3 | Algorithmic decomposition, plan templates, plan optimization | Yes — templates for common comm patterns speed up planning |
| M4 | Multi-provider ranking, A/B testing, cost routing, stats tracking | No — single provider per capability is fine for GTM |
| M5 | Map nodes, timeout management, resource limits, execution analytics | Partially — timeouts prevent hung calls |
| M6 | All of it | No — future R&D |
| M7 | Memory decay, conflict resolution, token budget management, user dashboard | Partially — conflict resolution prevents stale facts |
| M8 | Autonomy levels, proactive intelligence, goal inference, explainability, feedback loop | Partially — autonomy levels reduce approval fatigue |
| M9 | Planning view, artifact rendering, skill browser, notification management | Partially — artifact rendering makes results look better |
| M10 | Email, SMS rich, Slack, Telegram, cross-channel context, smart templates | Partially — email is Phase 2 GTM channel |
| M11 | PII detection, data retention, GDPR/DPDP compliance, RBAC, security logging, encryption | Partially — PII matters when handling real user comms |

---

## Per-Domain Phase Summary

Use this to self-assign. Each domain is a workstream that can be picked up by one person.

### Memory (M7)

| Phase | Focus | Key Deliverables |
|-------|-------|-----------------|
| 0 | Memory wiring | Episodic memory table + generation. mem0 → planner. Conversation buffer fix. |
| 1 | Context assembly | `assemble_for_planner()` wiring. Procedural memory surfacing. |
| 2 | — | (Memory work front-loaded in Phase 0-1) |
| 3 | — | — |

### Orchestration (M3, M5)

| Phase | Focus | Key Deliverables |
|-------|-------|-----------------|
| 0 | — | (Depends on memory wiring from Phase 0) |
| 1 | Planning + resilience | Fast path. Failure recovery. Clarification UX. Failure messaging. |
| 2 | Intelligence + scheduling | Channel auto-selection. Task scheduling from chat. Plan benchmarking. DND design. |
| 3 | Hardening | Feature flags. Rate limiting. Monitoring. |

### Comms + Adapters (M10, M1)

| Phase | Focus | Key Deliverables |
|-------|-------|-----------------|
| 0 | Architecture spike | Design agentic call flow. Identify handler changes. |
| 1 | Agentic calls + WhatsApp | Inbound/outbound calls through orchestration. WhatsApp agentic. Consent. Latency. Call summaries. |
| 2 | Voice quality + features | STT/TTS models. Noise cancellation. Hold improvements. Auto-WebRTC. Voice sample mgmt. Google fix. |
| 3 | Hardening | End-to-end call flow testing. |

### Frontend (M9)

| Phase | Focus | Key Deliverables |
|-------|-------|-----------------|
| 0 | Consent design | Consent-in-chat API contract. Design approval cards. |
| 1 | Consent + cleanup | Consent cards in chat. Unified consent list. Remove old task screen. Call logs cleanup. Animations. |
| 2 | Polish + features | Task history redesign. Chat management. Voice input Android. File attachments. Scheduled task UI. Voice card. |
| 3 | Hardening | Data source testing. Permissions popup. Search (if time). |
