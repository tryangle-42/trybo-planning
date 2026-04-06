# GTM Milestone Plan (60% Effort)

> 9 milestones toward beta. Each = binary, demo-able capability.
> One owner per milestone. Hyper-focused: one milestone per dev per week.
> KISS: integrations first, intelligence later.
> Consent is always in-app between Trybo and user. Channels (call/WhatsApp/email/SMS) are how Trybo communicates with others.

---

## Weekly Time Budget

| Activity | Days/week |
|----------|-----------|
| Productive coding | 3 |
| Testing + demo | 1 |
| Sprint planning + arch review (Ritvik) | 1 |

Of the 3 coding days, the split between GTM and architecture determines the beta date:

| Split | GTM days/person/week | Arch days/person/week | Beta (optimistic + 25% buffer) | Rationale |
|-------|---------------------|----------------------|-------------------------------|-----------|
| **80/20** (recommended) | ~2.5 | ~0.5 | **End of June** | Arch gets solutioning docs + Ritvik reviews (which IS the 20%). No heavy arch coding pre-beta — solutioning and review happen before implementation. Heavy arch implementation starts post-beta. Earlier beta = more user learning time before September. |
| 60/40 | ~2 | ~1 | **Mid-July** | Clean 60/40 from day 1. More arch progress pre-beta. But beta is 3 weeks later — 3 fewer weeks of real user feedback before September. |

**Total GTM work needed:** 75-120 person-days across 3 devs (+ Sanskar from W5 as contributor).

---

## The 9 Milestones

### M1: Data Source Connections (Owner: KPR)

**Demo:** User connects Google → "What's on my calendar today?" shows real events. "Summarize recent emails" shows real data. SMS sends. Notification arrives. Location query works. Camera photo available in chat. Zoom meeting transcript imported and readable.

| Item | Est. days |
|------|-----------|
| Google Calendar + Gmail (Composio or OAuth fix) | 3-5 |
| SMS (Twilio skill + consent) | 1 |
| Notifications (Firebase → agent events) | 1 |
| Location (mobile API → planner context) | 1-2 |
| Camera (photo capture → attachment) | 1-2 |
| **Beta total** | **7-11** |

**Depends on:** Nothing. Google is the critical item.

---

### M2: Outbound/Inbound Consent Flow (Owner: KPR)

**Demo:** "Send Sharma the agenda on WhatsApp" → consent card inline in chat (recipient, draft message, approve/edit/deny) → user edits → approves → sent. Same for calls and email. Pending screen shows all waiting approvals.

> Consent is always in-app. Trybo pre-fills what it can (draft, talking points, recipient). User approves/edits/denies. Channels are for communicating with *others*.

| Item | Est. days |
|------|-----------|
| Consent card UI (React Native component) | 3-4 |
| Outbound consent: WhatsApp | 2-3 |
| Outbound consent: calls | 2-3 |
| Outbound consent: email | 1-2 |
| Unified pending list (inbound + outbound) | 2-3 |
| **Beta total** | **10-15** |

**Depends on:** Nothing. Foundational — unblocks M4, M5.

---

### M3: Data/File Formats (Owner: KPR)

**Demo:** User attaches CSV → agent summarizes data. Attaches .docx → agent references content. Attaches audio → agent transcribes and uses as context.

> PDF attach + read already works. pgvector RAG is post-beta.

| Item | Est. days |
|------|-----------|
| CSV/XLS read (tabular parser → planner context) | 2-3 |
| .docx read (Kreuzberg or python-docx) | 1-2 |
| Audio transcription (Whisper batch → context) | 2-3 |
| **Beta total** | **4-8** |

**Depends on:** Nothing.

---

### M4: File Attachment for Outbound Comms (Owner: KPR)

**Demo:** "Send this report to Sharma on WhatsApp" → file attached → consent card shows attachment preview → approve → sent. Same for email.

| Item | Est. days |
|------|-----------|
| Attach file to WhatsApp outbound | 2-3 |
| Attach file to email outbound | 1-2 |
| Attachment preview in consent card | 1-2 |
| **Beta total** | **4-7** |

**Depends on:** M2 (consent cards), M3 (file read for context).

---

### M5: Voice Intelligence (Owner: RKP)

**Demo (outbound):** "Call Sharma about tomorrow's meeting" → consent card with talking points → approve → call → agent responds <2.5s → filler during pauses → summary in chat.

**Demo (inbound):** Someone calls → agent picks up with context → caller asks for info → consent card pushed to user's chat → user provides → agent relays.

**Critical decision (W1-2): Build vs Buy.** Third-party (Vapi/Retell) for beta recommended — ships in 2-3 weeks vs 5-7 for custom. Consent management is the SAME system regardless.

| Item | Est. days (third-party) |
|------|-----------|
| Evaluate + decide (Vapi/Retell/custom) | 3-5 |
| Outbound call flow (context → consent → platform → transcript) | 3-5 |
| Inbound call flow + mid-call consent webhook | 3-4 |
| Latency optimization (<2.5s), filler injection, endpointing | 2-3 |
| Multi-language eval + call summary generation | 2-3 |
| **Beta total** | **13-20** |

**Depends on:** M2 (consent for outbound calls + mid-call webhooks).

---

### M6: Task Visibility (Owner: AKN)

**Demo:** Task history shows sessions → batches → subtasks with statuses. User taps failed task → sees error → taps "retry." Search across chats finds last week's conversation.

| Item | Est. days |
|------|-----------|
| Task history UI (batch + task summary, paginated) | 3-5 |
| Subtask/batch status rendering (SSE + DB) | 2-3 |
| Search within and across chats (tsvector) | 2-3 |
| Retry UI on failed tasks | 1-2 |
| **Beta total** | **8-13** |

**Depends on:** M7 (execution engine must populate run→batch→task correctly).

---

### M7: Execution Intelligence (Owner: RKP)

**Demo:** 3-step plan running. Step 2 fails (API timeout). Chat shows: "Step 2 failed: timeout. Steps 1 and 3 done. Retry step 2?" User taps retry → only step 2 re-runs → succeeds. Also: "Every morning at 7am, brief me" → saved → runs next morning.

> **This milestone goes FIRST for RKP.** Execution resilience is the foundation everything else runs on. Building voice/consent on a fragile execution layer means flaky demos and wasted debug time.

| Item | Est. days |
|------|-----------|
| Single node failure handling (don't kill the run) | 2-3 |
| Proactive failure messaging in chat | 1-2 |
| Task restart from chat ("retry" button) | 2-3 |
| Recurring automations (chat → PlanSpec → APScheduler → execute) | 2-3 |
| **Beta total** | **7-11** |

**Depends on:** M2 (consent cards for retry/failure UI). But basic error handling can start before consent is ready.

---

### M8: Traceability (Owner: AKN)

**Demo:** Developer opens `/debug/trace/{id}`. Full request journey: GID (2ms) → Planner (1.2s) → Executor (3 nodes, 2.4s). Errors categorized (ADAPTER_TIMEOUT, PLANNING_FAILED). Analytics: 87% plan success, $12.40 LLM cost today.

| Item | Est. days |
|------|-----------|
| Correlation ID (trace_id) propagation | 1-2 |
| Span model (10 key spans) | 2-3 |
| Error categorization (~12 categories) | 1-2 |
| Debug API endpoints | 1 |
| Analytics (hourly rollup, health endpoint, cost tracking) | 3-4 |
| **Beta total** | **8-12** |

**Depends on:** Nothing. Start first — helps everyone debug.

---

### M9: Compliance (Owner: AKN)

**Demo:** Outbound WhatsApp → PII scan → Aadhaar redacted from API payload. Audit log shows every agent action. Rate limit: 31st run/hour returns "slow down."

| Item | Est. days |
|------|-----------|
| PII scanning (regex: Aadhaar, PAN, phone) | 2-3 |
| Audit trail (consent + API calls + data access) | 2-3 |
| Rate limiting (per-user, 429 response) | 1-2 |
| Input content safety (moderation classifier) | 1-2 |
| **Beta total** | **6-10** |

**Depends on:** Nothing. Lower demo priority than M2, M5, M1.

---

### Meeting App Transcripts (Owner: AKN, or Sanskar when he joins W5)

**Demo:** User connects Zoom → "What was discussed in yesterday's standup?" → agent summarizes the transcript.

| Item | Est. days |
|------|-----------|
| Zoom/Teams/Meet transcript import (Composio or MCP) | 3-5 |

Scoped standalone task — ideal for Sanskar's first real contribution.

---

## Dependency Graph

```
M7 (Exec intelligence) ────── foundation for everything
   │
M2 (Consent) ─────────┐
   │                    │
   ├──▶ M4 (File attach)│
   │      ▲              │
   │    M3 (File formats)│
   │                    │
   ├──▶ M5 (Voice) ─────┤
   │                    │
M1 (Data sources) ──────┤  parallel
M6 (Task visibility) ───┤  parallel (needs M7 data)
M8 (Traceability) ──────┤  parallel, start first
M9 (Compliance) ────────┘  parallel, lower priority
Meeting transcripts ────────  standalone
```

---

## Weekly Focus (80/20 — recommended)

~2.5 GTM coding days per person per week + 0.5 arch day + 1 day testing/demo + 1 day planning/review.

| Week | AKN | RKP | KPR | Demo-ready |
|------|-----|-----|-----|------------|
| **W1 (Apr 7)** | M8: correlation IDs, spans, debug endpoints | **M7: error recovery, single node failure handling, failure messaging** | M2: consent card UI, outbound consent WhatsApp | |
| **W2 (Apr 14)** | M8: error categories, analytics, health endpoint | **M7: task restart, recurring automations (APScheduler)** | M2: outbound consent calls + email, pending list | M8 ✅ |
| **W3 (Apr 21)** | M6: task history redesign, batch/subtask status | M5: third-party eval + decision. Outbound call flow. | M1: Google Calendar + Gmail (Composio/OAuth) | M2 ✅, M7 ✅ |
| **W4 (Apr 28)** | M6: search across chats, retry UI | M5: inbound call flow, mid-call consent webhook | M1: SMS, notifications, location, camera | M6 ✅ |
| **W5 (May 5)** | M9: PII scanning, audit trail, rate limiting | M5: latency optimization, filler, endpointing, call summary | M3: CSV/XLS/docx read, audio transcription | M5 ✅, M1 ✅ |
| **W6 (May 12)** | M9: input safety, completion. Meeting transcripts start. | M5: multi-language eval, polish | M4: file attach WhatsApp + email, preview in consent | M3 ✅ |
| **W7 (May 19)** | Meeting transcripts. Integration testing. | Integration testing. Voice edge cases. | M4 completion. Integration testing. | M4 ✅, M9 ✅ |
| **W8 (May 26)** | Integration testing, bug fixes | Integration testing, bug fixes | Integration testing, bug fixes | |
| **W9 (Jun 2)** | Polish, final fixes | Polish, final fixes | Polish, final fixes | |
| **W10 (~Jun 9)** | | | | **ALL MILESTONES → BETA** |

**Sanskar (joins W5):** Onboards W5. Takes meeting app transcripts in W6-7, or helps with integration testing.

---

## Load Check (at 80/20 = ~2.5 GTM days/week)

| Person | Milestones | Optimistic days | Weeks needed | Available weeks | Fit? |
|--------|-----------|----------------|-------------|----------------|------|
| **KPR** | M2 (10-15) + M1 (7-11) + M3 (4-8) + M4 (4-7) | 25-41 | 10-16 | 9 (+Sanskar help) | ⚠️ Tight. M3+M4 are small. Google fix is the risk. |
| **RKP** | M7 (7-11) + M5 (13-20) | 20-31 | 8-12 | 9 | ✅ Comfortable with third-party voice. |
| **AKN** | M8 (8-12) + M6 (8-13) + M9 (6-10) + transcripts (3-5) | 25-40 | 10-16 | 9 (+Sanskar help) | ⚠️ Tight. Sanskar takes transcripts. |

**At 60/40 (2 GTM days/week):** Everyone needs 12-20 weeks → beta mid-July. 3 extra weeks of build, 3 fewer weeks of user feedback.

**At 80/20 (2.5 GTM days/week):** Everyone needs 8-16 weeks → beta ~end of June. The 20% arch time covers solutioning docs + Ritvik reviews. Heavy arch implementation post-beta.

---

## Demo Schedule

| Date | Milestones shown | Who |
|------|-----------------|-----|
| **Apr 18 (W2)** | M8: debug trace end-to-end. M7: task failure → retry. M2: consent cards (WhatsApp + call). | AKN, RKP, KPR |
| **May 2 (W4)** | M5: outbound call with consent → call → summary. M6: task history with batch view. M1: Google Calendar live. | RKP, AKN, KPR |
| **May 16 (W6)** | M5: voice quality (<2.5s). M3: CSV/audio read. M9: PII scanning + audit. M1: SMS + location + camera. | All |
| **May 26 (W8)** | M4: file attachment outbound. Meeting transcripts. Full integration test. | All |
| **~Jun 9 (W10)** | **BETA** — all milestones in one integrated flow | All |

---

## What Is NOT in Beta

| Capability | When |
|-----------|------|
| Persona switching | Post-beta (arch work) |
| Channel auto-selection | Post-beta (arch work) |
| Hindi/Hinglish voice | Post-beta (Sarvam + tuning) |
| Note taking apps | Post-beta |
| Payment gateway | Post-beta (July hire) |
| Full PII protection (NER, encryption) | Post-beta (July security hire) |
| Multi-step delegation (UC5) | Post-beta (needs mature memory + execution) |
| Skill composition (save automations) | Post-beta. Recurring via APScheduler IS in beta. |
| pgvector RAG | Post-beta |
| Review subagents in CI | Post-beta |
