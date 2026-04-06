# Sprint Plan: April–September 2026

> Beta release: May 26. All baselines complete: September 30.
> Split: 60% GTM (beta features), 40% core architecture.
> Weekly sprints. Source: `Prosumer-GTM.md`, `roadmap-plan-akn/`, `roadmap-plan-rkp/`, `roadmap-plan-kpr/`

---

## Team & Capacity

| Month | Team | Headcount | Working days | Total hours (8h/day) |
|-------|------|-----------|-------------|---------------------|
| April | AKN, RKP, KPR | 3 | 66 | 528h |
| May | + GenAI hire | 4 | 88 | 704h |
| June | + Frontend Lead (Product Owner) | 5 | 110 | 880h |
| Jul–Sept | + Security Eng + Backend Eng + QA Eng | 8 | 528 | 4,224h |
| **Total** | | | **~792 days** | **~6,336 hours** |

| Person | Strengths | Domain |
|--------|-----------|--------|
| AKN | Full-stack architect, system design | Skill ecosystem, observability, dev tooling, integrations |
| RKP | Backend, orchestration, built WhatsApp + comms | Orchestration, voice/call pipeline, security, personas |
| KPR | Full-stack/frontend, WebRTC + voice features | Memory, frontend, 1P skills, platform infra |
| GenAI hire (May) | Generative AI / agentic systems | Skill creation, agent intelligence (under AKN) |
| Frontend Lead (June) | **Product Owner**. React Native. | Owns M9 independently |
| Security Eng (July) | Threat models, compliance | PII protection, RBAC, audit |
| Backend Eng (July) | Payments, infra | Payment gateway, platform |
| QA Eng (July) | Testing, CI | E2E tests, regression |

---

## GTM Audience Progression

| Phase | Timeline | Audience | What they need |
|-------|----------|----------|---------------|
| **Beta** | May 15 | **Executives** (EA/Secretary persona) | Smooth call/WhatsApp/email delegation with file sharing, calendar, web research, recurring automations |
| **Post-beta** | Jun–Jul | **Middle management** | Multi-step delegation, persona switching, Slack, meeting transcripts, channel auto-selection |
| **Scale** | Aug+ | **ICs** (Sales, Support, Analysts, PMs) | Specialized personas, deep integrations, Hindi, complex workflows |

---

## GTM Use Cases (UC1–UC10)

Each sprint delivers one or more of these end-to-end. Referenced throughout the plan.

| # | Use Case | Example prompt | What the user expects |
|---|----------|---------------|----------------------|
| **UC1** | **Agent-handled outbound call** | "Call Sharma about tomorrow's meeting" | Agent pulls meeting context from calendar, shows draft talking points for approval, calls Sharma, returns summary in chat. |
| **UC2** | **Agent-handled inbound call** | Phone rings from Sharma | Agent answers with context: "Hi Sharma, I believe you're calling about the meeting reschedule?" Handles the conversation, returns summary. |
| **UC3** | **Agent-handled WhatsApp** | "Send the meeting agenda to the team on WhatsApp" | Agent drafts message using calendar data, shows consent card, sends on approval. Supports file attachment. |
| **UC4** | **Information retrieval + analysis** | "Research our top 3 competitors and email me a summary" | Agent creates a research batch, synthesizes findings, drafts email with attachment, waits for approval before sending. |
| **UC5** | **Multi-step delegation** | "Check in with project leads about Q2 status, consolidate into a summary" | Agent identifies contacts, reaches out via WhatsApp/call, collects responses, synthesizes into executive summary. Multiple batches. |
| **UC6** | **Recurring automation** | "Every morning at 7am, check weather and summarize my calendar" | Agent saves as automation, runs daily via APScheduler, delivers briefing via WhatsApp. |
| **UC7** | **Memory-informed follow-up** | "Change the hotel in my Goa trip to something cheaper" | Agent recalls the trip plan from 3 days ago, knows "cheaper" = your budget preference, modifies just the hotel. |
| **UC8** | **Smart channel selection** | "Contact John" (no channel specified) | Agent evaluates: John is on WhatsApp + has phone. It's 3pm on a weekday. Last interactions were WhatsApp. Agent picks WhatsApp. |
| **UC9** | **Persona-specific interaction** | "Switch to research mode" → "Compare AWS vs GCP pricing" | Agent switches to Research Assistant persona (different voice, concise style, web search skills only). |
| **UC10** | **Hindi/Hinglish interaction** | "कल 3 बजे Sharma जी को call कर देना" | Agent understands mixed Hindi-English, schedules call, uses Hindi-capable voice during the call. |

---

## Architectural Decision: Call Intelligence Separation

Call intelligence is a **separate low-latency subsystem** from the main agentic orchestration. The main agent handles chat, WhatsApp, email, scheduling through GID → Planner → Executor. The call subsystem handles real-time voice conversations through its own STT → Call LLM → TTS pipeline (<800ms).

**Why separate:** The main planner takes 1-3 seconds per plan. During a live call, every utterance needs a response in <800ms. You can't route each caller statement through full planning.

**How they connect:**

```
MAIN AGENT                              CALL SUBSYSTEM
(chat, WhatsApp, email, scheduling)     (real-time voice, <800ms)
                                        
GID → Planner → Executor               STT → Call LLM → TTS
                                        Prompt-based, memory-aware
                                        Own conversation state per call
       │                                       │
       │         SHARED LAYER                  │
       ├── Contacts + Memory ─────────────────┤
       ├── Consent Management ────────────────┤
       ├── Transcript + Summary storage ──────┤
       └── Skill Data Store ──────────────────┘

CALL INITIATION:
  Main agent plans "call Sharma about X"
    → assembles context bundle (meeting details, contact history, talking points)
    → consent card in chat → user approves
    → hands context to Call subsystem → call starts

MID-CALL EVENTS (via webhooks):
  Call subsystem needs consent for something mid-call
    → fires webhook to main agent
    → consent card pushed to user's chat
    → user approves → webhook back to call subsystem
    → call continues

CALL COMPLETION:
  Call subsystem returns transcript + outcomes
    → main agent processes: save episode, extract facts, generate summary
    → summary appears in chat
```

**Consent management is ONE system for all channels:**

| Channel | How consent works | Same system? |
|---------|------------------|-------------|
| WhatsApp (outbound) | Agent drafts message → consent card in chat → approve → send | Yes — `interrupt()` + consent card |
| Call (outbound) | Agent prepares context → consent card in chat → approve → call starts | Yes — same consent card, different action |
| Call (mid-call) | Call subsystem fires webhook → consent card in chat → approve → webhook back | Yes — same card, async via webhook |
| Email (outbound) | Agent drafts email → consent card in chat → approve → send | Yes — same system |

**Third-party call solutions:** We're evaluating providers (Vapi, Retell, or custom LiveKit pipeline) that give us:
- Multi-language support (English + Hindi) out of the box
- Webhooks for mid-call events (consent triggers, data requests)
- Transcript streaming + post-call transcript
- Low-latency voice pipeline we don't have to build from scratch

The choice between build-own vs third-party is a Week 2-3 decision. Either way, the integration pattern is the same: context in → webhooks during → transcript out.

---

## GTM Integration Requirements (Bold Items)

All bold items from `Prosumer-GTM.md` mapped to sprints:

### Beta (by May 15)

| Integration | Status today | Work needed | Sprint |
|-------------|-------------|-------------|--------|
| **Email (Gmail)** | Read-only, broken Composio | Fix Composio + add send/compose | W3-4 |
| **Calendars** | Broken Composio | Fix alongside Gmail (same Composio fix) | W3 |
| **WhatsApp** | Done but not agentic | Route through orchestration + outbound consent | W3-4 |
| **Calls/Contacts** | Done but bypass orchestration | Agentic call flows + outbound consent + call intelligence design | W2-5 |
| **SMS** | Basic send via Twilio | Add to skill registry, consent flow | W5 |
| **Web research + scraping** | Done (Tavily/Perplexity) | Already works | — |
| **User input** | Chat exists | Already works | — |
| **File attachment outbound** | Not built | Text, PDF attach to WhatsApp/email | W5-6 |
| **File read: Text, Documents, Numbers** | Partial (Kreuzberg) | Wire text/PDF/CSV/XLS read into planner context | W5-6 |
| **Audio read (domain type)** | Not built | Basic audio transcription for call context (Whisper batch) | W6 |
| **Outbound consent flow** | Inbound only | Build for outbound calls + WhatsApp + email | W3-4 |
| **Voice intelligence (coherence + latency)** | Simple prompt-based | Call subsystem design + latency optimization | W2-6 |
| **Notifications** | Firebase exists | Wire into orchestration events | W5 |
| **Location** | Not built | Mobile location API → planner context | W6 |
| **Camera** | Not built | Basic photo capture → attach to outbound | W6 |
| **Recurring automation (APScheduler)** | APScheduler implemented | Wire into chat UI ("every morning do X") | W4-5 |
| **Analytics** | Not built | Hourly rollups, health endpoint, cost tracking | W5-6 |

### Post-Beta (June–July)

| Integration | Sprint |
|-------------|--------|
| **Note taking apps** | W9-10 |
| **Meeting apps (Zoom/Teams/Meet transcripts)** | W10-11 |
| **Slack (full: DMs, threads)** | W8-9 |
| **Device file storage** (text/PDF only) | W9 |

---

## Weekly Sprint Plan

### Week 1 (Apr 7–11): Tooling + Memory + Call Architecture

| Person | 60% GTM 🎯 | 40% Arch ⚙️ |
|--------|-----------|------------|
| **AKN** | | ⚙️ CLAUDE.md (backend + frontend). import-linter in CI. `/ship` + `/test-local` skills. START-HERE.md, RUNBOOK.md, PR template. |
| **RKP** | 🎯 Call intelligence architecture spike: design the separation (main agent vs call subsystem). Define webhook contracts. Evaluate third-party call platforms (Vapi/Retell/custom). | ⚙️ GID semantic router integration (`semantic-router`, <10ms). |
| **KPR** | 🎯 Consent-in-chat API contract design (covers WhatsApp + calls + email — one system). | ⚙️ Memory wiring: `execution_episodes` table, generate episodes in finalize node. |

> **RULE (Day 1):** Developers must manually update CLAUDE.md after any code change affecting module boundaries, skill definitions, API contracts, or architecture.

---

### Week 2 (Apr 14–18): Memory Complete + Call Design Finalized

| Person | 60% GTM 🎯 | 40% Arch ⚙️ |
|--------|-----------|------------|
| **AKN** | 🎯 `find_skills` retrieval API (pgvector + HNSW). Skill hierarchy namespace config. | ⚙️ Product planning templates (ADR, ticket structure, dev prompt). |
| **RKP** | 🎯 Call intelligence design finalized: build-own vs third-party decision. Webhook contract for mid-call events → consent system. Fast path for single-skill intents. | ⚙️ DAG compiler hardening (mixed node types, error edges). |
| **KPR** | 🎯 Wire mem0 into planner prompt. Fix conversation buffer (configurable N, include AI responses, summarize older). | ⚙️ Context assembly engine start (`assemble_for_planner()`). |

**Week 2 exit:** Memory wired (3 layers). Call architecture decided and documented. GID routes <10ms.

---

### Week 3 (Apr 21–25): Calls + WhatsApp Through Orchestration

| Person | 60% GTM 🎯 | 40% Arch ⚙️ |
|--------|-----------|------------|
| **AKN** | 🎯 **Fix Google Integration** (Composio — Calendar + Gmail read). Calendar data flows into call context. | ⚙️ Correlation ID propagation (trace_id). Lightweight span model (10 key spans). |
| **RKP** | 🎯 **Outbound calls through orchestration**: "Call Sharma" → planner → consent card → call subsystem. **Outbound consent** for calls (consent card before call initiates). | ⚙️ Run→batch→task ID management. |
| **KPR** | 🎯 **Consent cards in chat** (inline approve/edit/deny). Works for calls AND WhatsApp AND email — one card type, different actions. | ⚙️ Context assembly engine complete. |

---

### Week 4 (Apr 28 – May 2): Inbound Calls + WhatsApp Agentic + Recurring

| Person | 60% GTM 🎯 | 40% Arch ⚙️ |
|--------|-----------|------------|
| **AKN** | 🎯 **Email send/compose** via Gmail (Composio). UC4 pipeline: research → draft email → consent → send. | ⚙️ Error categorization taxonomy (12-15 categories). Debug API endpoints. |
| **RKP** | 🎯 **Inbound calls through orchestration** (call → context load → call subsystem handles with context). **WhatsApp through orchestration** (same agentic brain, consent for outbound). | ⚙️ Per-node retry with exponential backoff + error classification. |
| **KPR** | 🎯 **Recurring automations** wiring: chat prompt "every morning do X" → planner generates PlanSpec → save as automation → APScheduler cron trigger. Unified consent pending list. | ⚙️ Skill health metrics (EMA + auto-degradation). |

**Week 4 exit:** UC1 (outbound call), UC2 (inbound call), UC3 (WhatsApp) all work. Recurring automations work from chat. Email send works.

---

### Week 5 (May 5–9): Voice Latency + File Sharing + SMS + Notifications

| Person | 60% GTM 🎯 | 40% Arch ⚙️ |
|--------|-----------|------------|
| **AKN** | 🎯 **Analytics baseline**: hourly rollups, health endpoint, LLM cost tracking. | ⚙️ Onboard GenAI hire (starts this week): architecture walkthrough, first scoped task. |
| **RKP** | 🎯 **Voice latency <2.5s**: profile STT→LLM→TTS pipeline. Streaming optimization. Filler injection ("let me check..."). Endpointing tuning (350ms). Fix call summaries. | ⚙️ Input content safety (OpenAI Moderation API). |
| **KPR** | 🎯 **File attachment outbound** (text, PDF to WhatsApp/email). **File read** for context (text, PDF, CSV, XLS — Kreuzberg wiring). **SMS** skill registration + consent flow. **Notifications** — wire Firebase into orchestration events. | ⚙️ Output PII scanning (Aadhaar, PAN, phone regex). |
| **GenAI hire** | ⚙️ Onboarding: read docs, set up dev env, shadow AKN. | |

---

### Week 6 (May 12–16): Beta Polish — Audio, Location, Camera, Integration Testing

| Person | 60% GTM 🎯 | 40% Arch ⚙️ |
|--------|-----------|------------|
| **AKN** | 🎯 **Audio file read** (Whisper batch transcription → planner context for call prep). End-to-end integration testing: UC1-UC4 full flows. | ⚙️ Skill hierarchy refinements. |
| **RKP** | 🎯 **Voice UX polish**: interruption handling, hold behavior (template voice every 8-10s). Call logs cleanup. Mid-call webhook → consent flow tested end-to-end. | ⚙️ Approval gates: risk-level tags on all skills. Auto-approve reads. |
| **KPR** | 🎯 **Location** (mobile API → planner context: "find restaurants near me"). **Camera** (photo capture → attach to outbound WhatsApp/email). UI polish: interaction animations, remove old task screen. 1P skill reliability (call retry, WhatsApp 24h). | ⚙️ Conversation buffer improvements. |
| **GenAI hire** | 🎯 Help with integration testing. Start exploring MCP registry. | ⚙️ First scoped task (import-linter rules or PR classification). |

---

### Week 7 (May 19–23): Beta Hardening + Integration Testing

**Goal:** Everything built in W1-6 tested end-to-end as integrated flows. Bug fixes. Edge cases. No new features — only stability.

| Person | Focus |
|--------|-------|
| **AKN** | Integration testing all flows end-to-end (UC1-UC4). Skill hierarchy refinements. Bug fixes across integrations (Composio, file read, analytics). |
| **RKP** | Mid-call webhook → consent flow tested end-to-end. Voice edge cases (interruption during consent, hold timeout, caller hangs up). Bug fixes. |
| **KPR** | Remove old task screen. Conversation buffer edge cases. Integration bug fixes (file attachment + WhatsApp, SMS delivery tracking). Beta polish. |
| **GenAI hire** | Help with integration testing. Scoped architecture task (import-linter rules or PR classification). |

**Week 7 exit:** All beta flows tested, stable, and demo-ready. No known P0 bugs.

---

### ✅ Week 8: May 26 — BETA RELEASE

**What works at beta (executive-ready):**

| Capability | Status | Detail |
|-----------|--------|--------|
| **Outbound calls** with consent | ✅ | "Call Sharma about the meeting" → context from calendar → consent card → call → summary |
| **Inbound calls** with context | ✅ | Agent answers with caller history, pending items, meeting context |
| **WhatsApp** with consent + files | ✅ | Draft → consent → send. Attach text/PDF. |
| **Email** read + send + attachments | ✅ | Gmail via Composio. Research → draft email → attach file → consent → send. |
| **Calendar** integration | ✅ | Meeting context flows into calls, scheduling works |
| **Web research** | ✅ | Tavily/Perplexity search → synthesis → summary |
| **File read** (text, PDF, CSV, XLS) | ✅ | Agent reads attached docs for context |
| **File attachment** outbound | ✅ | Send files via WhatsApp/email |
| **Audio read** (basic) | ✅ | Whisper transcription for call context prep |
| **SMS** | ✅ | Basic send via Twilio with consent |
| **Notifications** | ✅ | Firebase push for agent events |
| **Location** | ✅ | Mobile location for context ("near me" queries) |
| **Camera** | ✅ | Photo → attach to outbound message |
| **Recurring automations** | ✅ | "Every morning do X" → APScheduler → runs automatically |
| **Analytics** | ✅ | Plan success rate, skill failure rate, LLM cost, health endpoint |
| **Memory** (3 layers) | ✅ | Buffer + episodic + semantic. Agent remembers context. |
| **GID** (<10ms routing) | ✅ | Fast path for simple intents, planner for complex |
| **Consent** (all channels) | ✅ | One system: calls + WhatsApp + email. Inbound + outbound. |
| **Voice latency** <2.5s | ✅ | Streaming pipeline, filler injection, endpointing |
| **Content safety** | ✅ | Input classification + output PII scanning |
| **Error recovery** | ✅ | Retry + clear failure messages + task-level retry |

**What does NOT work at beta:**

| Capability | When |
|-----------|------|
| Persona switching | June (W9-10) |
| Multi-step delegation (UC5) | June (W9-11) |
| Channel auto-selection | July (W13+) |
| Note taking apps | June (W11-12) |
| Meeting app transcripts | June (W11-12) |
| Slack (full) | June (W9-10) |
| Hindi/Hinglish | July (W15-17) |
| Payment gateway | Jul-Aug |
| RBAC / formal PII protection | Jul-Aug |
| Review subagents in CI | Aug |

---

### Post-Beta Weekly Sprints

#### Weeks 9-10 (Jun 2 – Jun 13): UC5 + Personas + Slack

| Person | Focus |
|--------|-------|
| **AKN** | 🎯 **Skill composition Phase A** — `composed_skills` table, `ComposedAdapter`, `automation.create` handler. Users save multi-step plans as reusable automations. |
| **RKP** | 🎯 **Persona Management** — 5 persona configs (EA, Research, Personal, Support, Sales). Planner integration, voice per persona, skill constraints. |
| **KPR** | 🎯 **Slack full integration** (DMs, threads — UC5 needs cross-channel delegation). Memory completion: procedural memory surfacing. |
| **GenAI hire** | 🎯 MCP registry poller + OpenAPI-to-skill generator (UC6: agent discovers new tools). |

#### Weeks 11-12 (Jun 16 – Jun 27): Meeting Apps + Note Taking + UC5 Full

| Person | Focus |
|--------|-------|
| **AKN** | 🎯 **Meeting app integration** (Zoom/Teams transcript import via Composio/MCP). Self-learning repo CI. |
| **RKP** | 🎯 **Agent Intelligence baseline** — feedback collection, error taxonomy integration, risk-tiered consent, plan quality scoring. |
| **KPR** | 🎯 **Note taking app integration**. Memory: three-read/three-write patterns, token budget. Unit testing: pytest infra + CI gates. |
| **GenAI hire** | 🎯 Quality gates + Docker sandbox for auto-generated skills. `skill_candidates` review queue. |
| **Frontend Lead** | 🎯 **Task history redesign** (batch + task summary, paginated — UC5 visibility). Chat management (rename/delete/copy). |

#### Weeks 13-14 (Jun 30 – Jul 11): Payment Start + Device Storage + Polish

| Person | Focus |
|--------|-------|
| **AKN** | ⚙️ New Skill Creation Phase C — dynamic MCP, pattern extraction. Architecture oversight. |
| **RKP** | ⚙️ Privacy/Security baseline — RBAC (2 roles), audit log, rate limiting, encryption. |
| **KPR** | 🎯 **Payment Gateway Phase 1** start (Stripe integration, subscription setup). **Device file storage** (text/PDF access). |
| **GenAI hire** | 🎯 **Channel auto-selection** (UC8) — preference learning from feedback data. |
| **Frontend Lead** | 🎯 Scheduled task UI. Voice connect card redesign. Planning view (show plan being built). |

#### Weeks 15-19 (Jul 14 – Aug 15): Hindi + Scale + IC Features (8 devs)

| Person | Focus |
|--------|-------|
| **AKN** | ⚙️ Skill creation completion. Meeting app transcript refinements. Architecture oversight. |
| **RKP** | 🎯 **Hindi ASR** (Sarvam integration + tuning). Noise cancellation. Voice UX polish. |
| **KPR** | 🎯 Payment Gateway Phase 2 (metering, dunning, invoicing). |
| **GenAI hire** | ⚙️ Prompt evolution (if enough feedback data). LLM-as-judge plan quality. |
| **Frontend Lead** | 🎯 Voice input Android. Search within chats. Data source connection UI. |
| **Security Eng** | ⚙️ PII Protection Phase 1+2 (field encryption, RBAC/MFA, consent management, DSAR, audit). |
| **Backend Eng** | 🎯 Payment support (metering, Stripe operational). SMS two-way. Richer file formats (.docx, .xlsx, images). |
| **QA Eng** | 🎯 E2E test suite: 10 core flows. Integration tests for all data sources. CI integration. |

#### Weeks 20-26 (Aug 18 – Sept 30): Hardening + All Baselines Complete

| Track | Owner(s) | Deliverables |
|-------|----------|-------------|
| **E2E test suite** | QA Eng + all | 10 core flows tested + regression suite |
| **Feature flags** | AKN + RKP | Staged rollout: agentic calls, personas, automations, Hindi |
| **Monitoring + alerting** | AKN | Dashboards: plan success, call latency, failure rate, cost |
| **Review subagents** | RKP | Code review + PR review agents in CI |
| **Frontend hardening** | Frontend Lead | Data source testing, permissions, UX consistency |
| **Payment hardening** | KPR + Backend Eng | Billing correctness, revenue metrics |
| **Security hardening** | RKP + Security Eng | PII completion, pen test prep, compliance |
| **IC persona refinements** | GenAI hire | Sales, Support, Analyst persona configs tuned from beta feedback |

### ✅ September 30: ALL BASELINES COMPLETE

---

## Use Case Delivery Map

```
           W1   W2   W3   W4   W5   W6   W7       W8     W9-10  W11-12  W13-14  W15-19  W20-26
           Apr ──────────────── May ─────────────────── Jun ──────────── Jul ──── Aug-Sept ──

UC1 Call    [arch] [design] [outbound] [inbound] [latency] [polish] [test] BETA  ── robust ── hardened
UC2 Call in [arch] [design]           [works]   [latency] [polish] [test] BETA  ── robust ── hardened
UC3 WhatsApp       [consent] [works]  [agentic] [+files]  [polish] [test] BETA  ── robust ── hardened
UC4 Research              [Google] [email]  [files]  [audio] [test] BETA  [+Slack] [+meet]── hardened
UC5 Delegate                                                              [works]  [full] ── hardened
UC6 Automate                      [APSched]                        BETA  [comp]          ── hardened
UC7 Memory  [wire] [mem0]  [ctx]                                   BETA  [full]          ── hardened
UC8 Channel                                                                       [works]── hardened
UC9 Persona                                                              [works] [polish]── hardened
UC10 Hindi                                                                        [works]── hardened

                                                                     ▲                        ▲
                                                               May 26: BETA             Sept 30: ALL
                                                               (Executives)             BASELINES

Target:  ──────── Executives (EA persona) ───────────────────────│──── Managers + ICs ────────
```

---

## Call Intelligence: Build vs Buy Decision (Week 2)

| Option | Pros | Cons | Cost |
|--------|------|------|------|
| **Vapi/Retell** (managed) | Multi-language, webhooks, transcripts out of box. 2-3 weeks to integrate vs 6-8 weeks to build. | Per-minute pricing ($0.05-0.10/min on top). Less control over latency. Vendor lock-in. | ~$0.10/min |
| **Custom (LiveKit + Deepgram + Cartesia)** | Full control. Lower per-minute cost at scale. Can optimize latency to <600ms. | 6-8 weeks to build the pipeline. Significant engineering investment. | ~$0.02/min at scale |
| **Hybrid** (start managed, migrate) | Ship fast with Vapi/Retell for beta. Migrate to custom when call volume justifies. | Two integration efforts. But second one is planned, not reactive. | $0.10/min → $0.02/min |

**Recommendation:** Hybrid. Use Vapi/Retell for beta (ships in 2-3 weeks). Start custom pipeline post-beta (June). Migrate when >1000 calls/month makes the cost difference meaningful.

---

## Critical Path

```
Week 1              Week 2           Week 3           Week 4           Week 5-6         Week 7          Week 8
────────            ────────         ────────         ────────         ────────         ────────        ────────

KPR: Memory    ───▶ KPR: mem0  ───▶ KPR: Consent ──▶ KPR: Recurring ─▶ KPR: Files  ──▶ KPR: Polish ──▶ BETA
wiring               + buffer        cards in chat     automations       + SMS + loc      + bug fixes

RKP: Call arch ───▶ RKP: Call  ───▶ RKP: Outbound ──▶ RKP: Inbound  ──▶ RKP: Voice ──▶ RKP: Edge   ──▶ BETA
spike                design final    calls + consent   + WhatsApp        latency          cases + test

AKN: Tooling  ───▶ AKN: Skill ───▶ AKN: Google   ──▶ AKN: Email    ──▶ AKN: Audio ──▶ AKN: E2E    ──▶ BETA
baseline             retrieval       fix (Composio)    send              + Analytics      integration
```

**If memory wiring slips:** Calls have no context → core value broken. Mitigation: AKN assists, scope to buffer + mem0 only.
**If call platform decision slips past Week 2:** Voice integration delayed by 2+ weeks. Mitigation: default to Vapi (fastest to integrate).
**If Composio Google fix is harder than expected:** No calendar context in calls. Mitigation: hardcode calendar data for beta demo, fix properly post-beta.

---

## 60/40 Split Verification

| Period | Capacity | 60% GTM target | Actual GTM | 40% Arch target | Actual Arch |
|--------|----------|----------------|------------|-----------------|-------------|
| W1-2 (Apr 7-18) | 66 days | 40 | ~38 (call arch, consent, memory) | 26 | ~28 (tooling, GID, spans) |
| W3-4 (Apr 21 – May 2) | 66 days | 40 | ~42 (calls, WhatsApp, Google, email, recurring) | 26 | ~24 (error handling, batch IDs) |
| W5-6 (May 5-16) | 88 days | 53 | ~55 (voice, files, SMS, location, camera, audio, analytics) | 35 | ~33 (safety, PII, onboarding) |
| W7 (May 19-23) | 44 days | 26 | ~28 (integration testing, bug fixes, polish all beta flows) | 18 | ~16 (approval gates, buffer edge cases) |
| W9-12 (Jun 2-27) | 154 days | 92 | ~90 (personas, Slack, meeting apps, notes, skill comp) | 62 | ~64 (feedback, testing, self-learn) |
| W13-26 (Jun 30-Sept 30) | 748 days | 449 | ~445 (Hindi, payment, channel sel, file formats, e2e, hardening) | 299 | ~303 (PII, security, platform, review agents, monitoring) |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Memory wiring slips (KPR W1-2) | Calls have no context | AKN assists. Scope to buffer + mem0 if episodic at risk. |
| Call platform decision delayed | Voice integration pushes past W5 | Default to Vapi — fastest to integrate. Hybrid approach allows migration later. |
| Composio Google fix harder than expected | No calendar/email integration at beta | Hardcode calendar data for demo. Fix properly post-beta. |
| Voice latency >2.5s | Calls feel broken | Filler injection masks perceived latency. GPT-4o-mini for call turns. Cartesia TTS. |
| Too many integrations crammed into W5-6 | Quality suffers | Deprioritize camera + location if needed (nice-to-have, not executive-critical). |
| GenAI hire ramp-up slow | Skill creation delayed | Pair with AKN. Scoped first tasks. Onboarding docs ready from W1. |
| KPR overloaded | Memory + frontend + files + SMS + payment | Frontend Lead (June) absorbs all UI. Backend Eng (July) absorbs payment. |
| July hires don't close by July 1 | PII, payment, QA delayed | Start hiring April. Clearly scoped roles. |

---

## Hiring Plan

### Already planned

| Who | Joins | Owns |
|-----|-------|------|
| GenAI hire | May 5 (W5) | Skill creation, agent intelligence (under AKN) |
| Frontend Lead | June 2 (W9) | M9 independently. Product-level UI decisions. |

### Hires needed by July 1

| Priority | Role | What they own |
|----------|------|---------------|
| 1 | **Security/Compliance Eng** | PII protection (68 days). Field encryption, RBAC/MFA, DSAR, audit, retention. |
| 2 | **Backend Eng** | Payment Gateway support. SMS two-way. Richer file formats. Platform infra. |
| 3 | **QA Eng** | E2E test suite (10 flows). Regression suite. CI integration. |
