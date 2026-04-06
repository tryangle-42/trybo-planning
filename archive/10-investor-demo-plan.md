# Investor Demo Sprint — Jira-Ready Plan -> Created and Completed remaining things mentioned in 13-platform-completion-status.md

## Context

8 demo use cases for Atul Sir. 3 developers (Krishna, Avinash, Rishav). The agentic executor skeleton is complete (foundation sprint, 30 skills, memory Phase 0-1).

The key architectural decision from our discussion: **batch/task is the execution primitive**. The graph creates batches → interrupts → tasks execute via existing infra (calls, WhatsApp) → batch completes → graph resumes with results. No call-specific or channel-specific resume logic.

Task output uses **hybrid schema**: fixed base fields (status, raw_content, summary) + flexible `extracted_data` JSONB (schema decided by planner at planning time).

---

## File References (Critical Paths)

| Purpose | File |
|---|---|
| Batch progress + completion trigger | `app/services/batch_service.py` → `update_batch_progress()` line 111 |
| Batch completion reporter (insert resume hook here) | `app/services/batch_completion_reporter.py` → `process_batch_completion()` line 23 |
| Bot task status updates | `app/services/bot_task_service.py` → `update_bot_task_status()` line 112 |
| Graph creates batches | `app/agentic_mvp/runtime/langgraph/compiler_graph.py` → `create_tracking()` line 1031 |
| Graph resume handler | `app/routes/agent_runs.py` → `_resume_run()` line 330 |
| Call → task propagation | `app/services/call_service.py` → `update_bot_task_from_call()` line 396 |
| WhatsApp task completion | `app/services/whatsapp_service.py` → `_mark_bot_task_completed()` line 1512 |
| Skill registry | `app/agentic_mvp/runtime/seed_registry.py` |
| Direct adapter dispatch | `app/agentic_mvp/runtime/adapters/direct.py` |
| Tavily search adapter | `app/agentic_mvp/runtime/adapters/mcp_tavily.py` |
| LLM planner | `app/agentic_mvp/runtime/langgraph/llm_planner.py` |
| Config/env vars | `app/config/__init__.py` |
| DB schema reference | `entire-schema.md` |

---

## Tickets

### INFRA-1: Batch Completion → Graph Resume [Krishna]
**Type:** Infrastructure | **Priority:** P0-Blocker | **Estimate:** 1 day
**Blocks:** UC2, UC3, UC6 (any multi-step flow that uses call/WhatsApp results)

**Problem:** `channel.call.initiate` and `channel.whatsapp.send` fire and forget. The graph has no way to wait for task completion and use the results in subsequent nodes. `create_tracking()` creates batches but the graph continues immediately without waiting.

**Solution:** After batch completion, auto-resume the suspended graph with batch results.

**Implementation:**
1. In `compiler_graph.py`: after the `execute` node runs channel skills, add `interrupt()` call if the plan contains channel nodes. The graph suspends, waiting for batch completion.
2. In `batch_completion_reporter.py` → `process_batch_completion()`: after generating the summary, check if this batch has a linked `run_id` (stored in `batches.metadata.run_id` — already set by `create_tracking()`). If yes, call `_resume_run(run_id, resume_value=batch_results_payload)`.
3. `batch_results_payload` = collect all bot_tasks for this batch + their outputs (see INFRA-2) + batch summary.
4. In `agent_runs.py`: new internal function `resume_from_batch_completion(run_id, batch_id)` that builds the payload and calls `_resume_run()`.

**Files to modify:**
- `app/agentic_mvp/runtime/langgraph/compiler_graph.py` — interrupt after channel execution
- `app/services/batch_completion_reporter.py` — add graph resume trigger
- `app/routes/agent_runs.py` — add `resume_from_batch_completion()` helper

**Acceptance criteria:**
- Graph with a call node suspends after task creation
- When call completes + transcript ready → batch completes → graph resumes
- Next node in graph receives batch results via `$memory` reference
- Works for WhatsApp tasks too (batch with WhatsApp tasks → response received → complete → resume)

---

### INFRA-2: Structured Task Output (Hybrid Schema) [Krishna]
**Type:** Infrastructure | **Priority:** P0-Blocker | **Estimate:** Half day
**Blocks:** INFRA-1 (resume payload needs structured output), UC2, UC6

**Problem:** `bot_tasks` has no `output` or `result` field. When a call/WhatsApp task completes, only the status is stored. The graph needs structured results (transcript text, extracted data) to feed into synthesis nodes.

**Solution:** Add structured output fields to bot_tasks + post-completion extraction skill.

**Implementation:**
1. DB migration: add to `bot_tasks`:
   - `output_raw TEXT` — raw content (transcript text for calls, message thread for WhatsApp)
   - `output_summary TEXT` — LLM-generated 1-2 sentence summary
   - `output_extracted JSONB` — structured data extracted per planner's schema
   - `extraction_schema JSONB` — the schema the planner defined (set at task creation time)
2. In `call_service.py` → `update_bot_task_from_call()`: when task completes, populate `output_raw` from transcript `full_text`.
3. In `whatsapp_service.py` → `_mark_bot_task_completed()`: populate `output_raw` from `whatsapp_messages` WHERE `topic_id = task_id`.
4. New skill: `task.output.extract` — takes `bot_task_id` + `extraction_schema` → reads `output_raw` → LLM extracts structured data → writes to `output_extracted`.
5. In `create_tracking()`: when creating bot_tasks from plan nodes, copy `output_extraction_schema` from the PlanSpec node into `extraction_schema` on the bot_task.
6. In `batch_completion_reporter.py`: before resuming graph (INFRA-1), run extraction for all tasks in batch that have an `extraction_schema`.

**PlanSpec addition:** Each channel node can optionally have:
```json
{
  "output_extraction_schema": {
    "company": "string",
    "premium": "number",
    "coverage": "string"
  }
}
```
The planner decides this at planning time based on the user's goal.

**Files to modify:**
- DB migration (new columns on `bot_tasks`)
- `app/services/bot_task_service.py` — populate output fields on completion
- `app/services/call_service.py` — write transcript to `output_raw`
- `app/services/whatsapp_service.py` — write message thread to `output_raw`
- `app/agentic_mvp/runtime/langgraph/compiler_graph.py` → `create_tracking()` — pass extraction schema
- `app/agentic_mvp/runtime/contracts.py` — add `output_extraction_schema` to PlanSpec node model
- `app/services/batch_completion_reporter.py` — run extraction before graph resume

---

### INFRA-3: Perplexity Sonar Search Integration [Avinash]
**Type:** Infrastructure | **Priority:** P0 | **Estimate:** Half day
**Blocks:** UC2, UC6, UC8

**Problem:** Tavily returns raw web snippets. For entity discovery ("insurance companies in Delhi with phone numbers"), structured, synthesized answers are needed.

**Implementation:**
1. New handler `_search_perplexity()` in `adapters/direct.py` (or new file `adapters/search_perplexity.py`)
2. Perplexity Sonar API: `POST https://api.perplexity.ai/chat/completions` with `model: "sonar"`
3. Skill definition `web.search.perplexity` in seed_registry
4. Input: `query`, `search_focus` (web/academic), `return_citations`
5. Output: `answer` (synthesized text), `citations[]`, `sources[]`
6. Config: `PERPLEXITY_API_KEY` in `app/config/__init__.py`
7. Fallback: if Perplexity fails → fall back to Tavily
8. Update planner prompts: prefer `web.search.perplexity` over `web.research.tavily` for entity discovery

**Files to modify:**
- `app/agentic_mvp/runtime/adapters/direct.py` — new handler
- `app/agentic_mvp/runtime/seed_registry.py` — skill definition
- `app/config/__init__.py` — `PERPLEXITY_API_KEY`
- `requirements.in` — httpx already present, no new deps

---

### INFRA-4: Document Upload + RAG Context [Rishav]
**Type:** Infrastructure | **Priority:** P1 | **Estimate:** 1 day
**Blocks:** UC3

**Problem:** UC3 (sales pitch) needs the AI call agent to reference uploaded pitch docs, FAQs, and past call material during a live call.

**Implementation:**
1. New route: `POST /api/documents/upload` — accepts multipart file (PDF, DOCX, TXT) or URL
2. New table: `user_documents(id, user_id, filename, doc_type, content_text TEXT, chunks JSONB, status, created_at)`
3. Parsing: `pymupdf` for PDF, `python-docx` for DOCX, plain read for TXT
4. Chunking: split content_text into ~500 token chunks with overlap
5. Embedding: use same pgvector infra as mem0 (`text-embedding-3-small`)
6. New skill: `document.context.retrieve` — given a query + user_id → semantic search over chunks → return top-K relevant chunks
7. Integration point: before `channel.call.initiate`, if plan includes doc context → retrieve chunks → inject into call agent system prompt as "Reference material:" section
8. `GET /api/documents` — list user's uploaded documents
9. `DELETE /api/documents/{id}` — remove document + chunks

**Files to create:**
- `app/routes/documents.py` — upload/list/delete endpoints
- `app/services/document_service.py` — parse, chunk, embed, retrieve
- DB migration for `user_documents` table + vector index

---

### INFRA-5: Task SLA / Deadline System [Rishav]
**Type:** Infrastructure | **Priority:** P1 | **Estimate:** 1 day
**Blocks:** UC4, UC5, UC7

**Problem:** Tasks have no deadline concept. Can't track overdue items or auto-escalate.

**Implementation:**
1. DB migration: add to `bot_tasks`:
   - `deadline_at TIMESTAMPTZ` — when this task should be done by
   - `sla_status TEXT DEFAULT 'on_track'` — `on_track | approaching | overdue | completed`
   - `follow_up_config JSONB` — `{"channels": ["whatsapp","call"], "escalation_interval_hours": 4, "max_attempts": 3, "attempt": 0}`
2. New scheduled job type `sla_checker` (in `app/services/job_scheduler/`):
   - Runs every 10 min
   - Queries `bot_tasks WHERE deadline_at IS NOT NULL AND status NOT IN ('completed','failed','canceled')`
   - For each: if `deadline_at - now() < escalation_interval` → set `sla_status = 'approaching'`
   - If `now() > deadline_at` → set `sla_status = 'overdue'` → trigger follow-up (see INFRA-6)
3. New skill: `tasks.due.list` — returns all tasks with approaching/overdue SLA for a user
4. Planner integration: when user says "send WhatsApp to Rahul, needs response by Friday" → planner sets `deadline_at` in the channel node config

**Files to modify:**
- DB migration (new columns on `bot_tasks`)
- `app/services/job_scheduler/worker.py` — add `sla_checker` job type
- `app/agentic_mvp/runtime/seed_registry.py` — `tasks.due.list` skill
- `app/agentic_mvp/runtime/adapters/direct.py` — handler for `tasks.due.list`

---

### INFRA-6: Channel Escalation Logic [Rishav]
**Type:** Infrastructure | **Priority:** P1 | **Estimate:** Half day
**Blocks:** UC4, UC5, UC7
**Depends on:** INFRA-5

**Problem:** "Send WhatsApp, if no response in 4 hours, call." No time-based channel escalation exists.

**Implementation:**
1. In `sla_checker` job (from INFRA-5): when a task is overdue AND has `follow_up_config`:
   - Read `follow_up_config.channels` array (ordered: WhatsApp first, then call)
   - Read `follow_up_config.attempt` (which channel attempt we're on)
   - Create a new `bot_task` with the next channel in the escalation ladder
   - Same prompt/contact as original task, different `channel_type`
   - Link new task to same `batch_id`
   - Increment `attempt` in `follow_up_config`
   - If `attempt >= max_attempts` → mark original task as `failed` with `error_message = "escalation_exhausted"`
2. Track escalation chain: `bot_tasks.metadata.escalated_from = original_task_id`

**Files to modify:**
- `app/services/job_scheduler/worker.py` — escalation logic in `sla_checker`
- `app/services/bot_task_service.py` — `create_follow_up_task()` helper
- `app/services/task_initiation_service.py` — reuse `initiate_task()` for follow-up

---

### INFRA-7: Task Restart [Rishav]
**Type:** Infrastructure | **Priority:** P2 | **Estimate:** Half day
**Blocks:** UC5

**Problem:** Failed or timed-out tasks can't be retried. User has to recreate manually.

**Implementation:**
1. New endpoint: `POST /api/tasks/{task_id}/restart`
   - Reads original `bot_tasks` record
   - Creates new task with same params (prompt, contact, channel) + fresh status
   - Increments `attempt_count` on original
   - Returns new task_id
2. For agentic runs: `POST /api/agent/runs/{run_id}/restart`
   - Reloads plan from event store
   - Re-invokes graph from last failed node (or from scratch)
3. Batch integration: if restarted task is part of a batch → adjust batch counts

**Files to modify:**
- `app/routes/tasks.py` — restart endpoint
- `app/services/bot_task_service.py` — `restart_task()` function
- `app/routes/agent_runs.py` — run restart endpoint

---

### INFRA-8: Inbound WebRTC Call Flow [Krishna]
**Type:** Infrastructure | **Priority:** P1 | **Estimate:** 1 day
**Blocks:** UC1

**Problem:** System only initiates outbound calls. UC1 needs agent to RECEIVE calls (delivery person calls the WebRTC link).

**Implementation:**
1. Twilio incoming call webhook: `POST /twilio/incoming-voice` → create LiveKit room → connect AI agent
2. AI agent joined to room with configurable system prompt (OTP handling, delivery instructions)
3. User configuration: "For deliveries, use this Twilio number" → stored in `skill_data_store` with `namespace="user_profile", record_type="inbound_routing"`
4. After call: push notification to user with call summary (use existing `notification.push.send`)
5. Inbound call stored in `calls` table with `direction: 'inbound'`

**Files to modify:**
- `app/routes/twilio.py` — incoming call webhook handler
- `app/services/call_service.py` — `handle_inbound_call()` function
- LiveKit room creation for inbound (extend existing `device_call_service.py`)
- AI agent prompt configuration for inbound calls

---

### INFRA-9: Heartbeat Push Notifications on Pending Tasks [Avinash]
**Type:** Infrastructure | **Priority:** P2 | **Estimate:** Half day

**Problem:** Tasks stuck in pending/waiting state with no user visibility.

**Implementation:**
1. New scheduled job `pending_task_notifier` (runs every 10 min):
   - Query `bot_tasks WHERE status IN ('pending','scheduled') AND created_at < now() - interval '30 min'`
   - For each unique user_id: `notification.push.send` with count + task summaries
2. Configurable threshold (default 30 min)
3. Don't re-notify for same task within 1 hour (track last notification in `bot_tasks.metadata.last_notified_at`)

**Files to modify:**
- `app/services/job_scheduler/worker.py` — new job type
- Config: `PENDING_TASK_NOTIFY_THRESHOLD_MIN` in `app/config/__init__.py`

---

### INFRA-10: Event Store Persistence [Avinash]
**Type:** Infrastructure | **Priority:** P2 | **Estimate:** Half day

**Problem:** `InMemoryAgentRunEventStore` loses all run state on server restart. Demo killer.

**Implementation:**
1. Replace `InMemoryAgentRunEventStore` with `SupabaseAgentRunEventStore`
2. New table: `agent_run_events(id, run_id, seq, event_type, payload JSONB, created_at)`
3. Same interface: `append()`, `get_events(after_seq)`, `get_record()`
4. Index on `(run_id, seq)` for fast event retrieval
5. Reaper job: clean up events for completed runs after 24h

**Files to modify:**
- `app/agentic_mvp/runtime/event_store.py` — new Supabase-backed implementation
- DB migration for `agent_run_events` table
- `app/routes/agent_runs.py` — swap `EVENTS` to use persistent store

---

### UC1: OTP / Delivery Handling via WebRTC [Krishna]
**Type:** Use Case | **Priority:** P1 | **Estimate:** 1 day
**Depends on:** INFRA-8 (inbound WebRTC)

**Demo flow:**
1. User gives a Twilio number as "delivery contact number" (e.g., paste in Amazon delivery instructions)
2. Delivery agent calls that number
3. AI agent picks up: "Hi, I'm calling on behalf of [user]. How can I help?"
4. Delivery agent asks for OTP → AI agent relays OTP question to user via push notification
5. User responds with OTP → AI agent tells delivery agent
6. Call ends → push notification to user: "Delivery call handled. OTP shared. Package arriving in 10 min."

**Skills needed:**
- Inbound call handling (from INFRA-8)
- OTP-specific agent prompt template
- Push notification relay (existing `notification.push.send`)
- New skill: `delivery.otp.relay` — handles the push → wait → relay cycle

**Files to create/modify:**
- Agent prompt templates for delivery/OTP handling
- `app/agentic_mvp/runtime/seed_registry.py` — delivery skill definitions
- `app/agentic_mvp/runtime/adapters/direct.py` — delivery handlers

---

### UC2: Insurance Vendor Comparison [Krishna]
**Type:** Use Case | **Priority:** P1 | **Estimate:** 1-2 days
**Depends on:** INFRA-1 (batch→graph resume), INFRA-2 (structured output), INFRA-3 (Perplexity)

**Demo flow:**
1. User: "Compare term insurance plans from top 5 companies for a 30-year-old, 1 crore cover"
2. Agent searches via Perplexity → finds LIC, HDFC Life, ICICI Prudential, Max Life, Tata AIA + phone numbers
3. Agent creates batch with 5 call tasks (one per company)
4. Graph interrupts (approval gate): "I'll call these 5 companies to compare plans. Approve?"
5. User approves → calls execute (can be sequential or parallel depending on demo pacing)
6. Each call: AI agent asks about premium, coverage, claim process, exclusions
7. Calls complete → transcripts extracted → structured data per planner schema
8. Graph resumes → synthesis node creates comparison report
9. Structured artifact: comparison table with recommendation

**Skills needed:**
- `web.search.perplexity` (INFRA-3)
- `search.extract.phone_numbers` — LLM extracts phone numbers from Perplexity answer
- `channel.call.initiate` (existing, enhanced with INFRA-1/2)
- `task.output.extract` (INFRA-2)
- New skill: `comparison.synthesize` — takes N extracted outputs → generates comparison table + recommendation
- Structured artifact type: `comparison_report`

**Planner flow:**
```
search_companies → extract_phones → [call_company_1 ∥ call_company_2 ∥ ... ∥ call_company_5] → synthesize_comparison
```

**Files to create/modify:**
- `app/agentic_mvp/runtime/adapters/direct.py` — `_search_extract_phones()`, `_comparison_synthesize()`
- `app/agentic_mvp/runtime/seed_registry.py` — new skill definitions
- `app/agentic_mvp/runtime/contracts.py` — `comparison_report` artifact type

---

### UC3: Salesperson / Lead Pitch with Docs [Krishna]
**Type:** Use Case | **Priority:** P1 | **Estimate:** 1 day
**Depends on:** INFRA-1 (batch→graph resume), INFRA-4 (doc upload)

**Demo flow:**
1. User uploads pitch deck + FAQ document
2. User: "Call Rahul (+91...) and pitch our product. Use the uploaded material."
3. Agent retrieves relevant doc chunks → builds call agent prompt
4. Agent calls Rahul → pitches product → answers questions using doc context
5. If agent gets stuck → puts Rahul on hold → pushes clarification to user → user answers → agent resumes
6. Call ends → summary: "Pitched to Rahul. Interested in enterprise plan. Follow-up scheduled."

**Skills needed:**
- `document.context.retrieve` (INFRA-4)
- `channel.call.initiate` (existing, enhanced with doc context injection)
- Clarification mid-call (existing interrupt mechanism, but triggered from call agent)

**Files to modify:**
- Call agent prompt construction — inject doc chunks as reference material
- `app/agentic_mvp/runtime/adapters/direct.py` — `_call_initiate()` to accept `reference_material` input

---

### UC4: Field Sales / School Visit Tracking [Rishav]
**Type:** Use Case | **Priority:** P1 | **Estimate:** 1 day
**Depends on:** INFRA-5 (SLA), INFRA-6 (escalation)

**Demo flow:**
1. User (sales manager) sets up: "Every evening at 6 PM, collect reports from my 3 sales reps"
2. Scheduled job fires at 6 PM → creates batch with 3 WhatsApp tasks
3. Each WhatsApp: "Hi [rep], please share: 1) Schools visited today 2) Schools planned tomorrow 3) Target vs achievement"
4. Reps reply via WhatsApp → responses captured
5. If rep doesn't reply by 8 PM → auto-call (escalation from INFRA-6)
6. All responses collected → synthesis: "2/3 reps reported. Rahul: 4 schools visited, 80% target. Priya: 3 schools, 60% target. Amit: no response (called, voicemail)."

**Skills needed:**
- `channel.whatsapp.send` (existing)
- `channel.call.initiate` (existing, for escalation)
- New skill: `report.collect.synthesize` — takes WhatsApp responses + call transcripts → structured report
- Scheduled job configuration skill (existing `briefing.preference.save` pattern)

**Files to create/modify:**
- `app/agentic_mvp/runtime/adapters/direct.py` — report collection handler
- `app/agentic_mvp/runtime/seed_registry.py` — skill definitions
- Planner prompt: teach it to generate evening reporting flows

---

### UC5: Follow-up & Task SLA Management [Rishav]
**Type:** Use Case | **Priority:** P1 | **Estimate:** 1 day
**Depends on:** INFRA-5 (SLA), INFRA-6 (escalation), INFRA-7 (restart)

**Demo flow:**
1. User: "Send WhatsApp to Rahul about the invoice. He needs to respond by Friday."
2. Agent creates task with `deadline_at = Friday EOD`, `follow_up_config = {channels: ["whatsapp","call"], interval: 4h}`
3. WhatsApp sent. Thursday afternoon: SLA checker detects "approaching" → WhatsApp reminder: "Reminder: pending invoice response"
4. Friday morning: still no response → auto-call Rahul
5. User asks: "What's due today?" → `tasks.due.list` returns overdue/approaching tasks
6. User: "Restart the failed call to Rahul" → task restart (INFRA-7)

**Skills needed:**
- `channel.whatsapp.send` (existing)
- `channel.call.initiate` (existing)
- `tasks.due.list` (INFRA-5)
- Task restart endpoint (INFRA-7)

**Planner integration:** Planner must understand deadline/SLA when user mentions time constraints → set `deadline_at` and `follow_up_config` in plan.

---

### UC6: Bakery / Grocery + Coordination [Avinash]
**Type:** Use Case | **Priority:** P1 | **Estimate:** 1-2 days
**Depends on:** INFRA-1 (batch→graph resume), INFRA-2 (structured output), INFRA-3 (Perplexity)

**Demo flow:**
1. User: "Find bakeries within 1 km that have a 1 kg pineapple cake. Check availability and price."
2. Agent searches via Perplexity (with user location context) → finds 3 bakeries + phone numbers
3. Agent creates batch with 3 call tasks
4. Each call: "Do you have a 1 kg pineapple cake? What's the price? How soon can it be ready?"
5. Calls complete → extracted: `{bakery, available, price, ready_in}`
6. Synthesis: "2/3 have it. Baker's Dozen: ₹850, ready in 1 hour. Theobroma: ₹1200, ready now."
7. Optional: "Call my driver and tell him to pick up from Baker's Dozen" → driver coordination call

**Skills needed:**
- `web.search.perplexity` (INFRA-3) with location context
- `search.extract.phone_numbers`
- `channel.call.initiate` (existing, enhanced)
- `task.output.extract` (INFRA-2) with availability schema
- `comparison.synthesize` (shared with UC2)

**Planner flow:**
```
search_bakeries → extract_phones → [call_bakery_1 ∥ call_bakery_2 ∥ call_bakery_3] → synthesize_availability → (optional) call_driver
```

---

### UC7: Multi-channel Communication & Escalation [Avinash]
**Type:** Use Case | **Priority:** P1 | **Estimate:** Half day
**Depends on:** INFRA-5 (SLA), INFRA-6 (escalation)

**Demo flow:**
1. User: "Contact Rahul about the Q1 review deck"
2. Agent decides channel: WhatsApp (business hours, non-urgent topic)
3. WhatsApp sent with contextual message
4. No response in 4 hours → auto-escalation to call
5. Call connects → agent discusses the review deck → gets response
6. Summary: "Contacted Rahul. WhatsApp at 2 PM (no response). Called at 6 PM — he'll send the deck by tomorrow."

**Skills needed:**
- Channel selection logic in planner (teach planner to pick channel based on urgency, time, contact preference)
- `channel.whatsapp.send` (existing)
- `channel.call.initiate` (existing)
- Escalation (INFRA-6)

**Implementation:** Mostly planner prompt engineering — teach the planner to set `follow_up_config` with escalation ladder when the instruction is generic ("contact them" vs explicit "call them").

---

### UC8: Web Search + Skill Orchestration [Avinash]
**Type:** Use Case | **Priority:** P2 | **Estimate:** Half day
**Depends on:** INFRA-3 (Perplexity), INFRA-1 (batch→graph resume)

**Demo flow:**
1. User: "Find 3 co-working spaces near Hauz Khas, call them and check if they have a 10-person meeting room available tomorrow"
2. This is the GENERIC version of UC2/UC6 — same pattern, different domain
3. Perplexity search → extract phones → multi-call → synthesis

**Skills needed:** All reused from UC2/UC6. Zero new skills.

**Implementation:** This is purely a planner generalization demo. The planner should handle arbitrary "search → extract → call → compare" flows without domain-specific skills. If UC2 and UC6 work, UC8 works automatically.

**Only work needed:** Test with diverse queries, fix planner prompt if it fails on unfamiliar domains.

---

## Team Assignment Summary

### Krishna — Voice/Call Flows
| Ticket | Priority | Estimate |
|---|---|---|
| INFRA-1: Batch completion → graph resume | P0 | 1 day |
| INFRA-2: Structured task output | P0 | half day |
| INFRA-8: Inbound WebRTC call flow | P1 | 1 day |
| UC1: OTP/Delivery handling | P1 | 1 day |
| UC2: Insurance vendor comparison | P1 | 1-2 days |
| UC3: Salesperson pitch with docs | P1 | 1 day |
| **Total** | | **~5-6 days** |

### Rishav — Task Management & Documents
| Ticket | Priority | Estimate |
|---|---|---|
| INFRA-4: Document upload + RAG | P1 | 1 day |
| INFRA-5: Task SLA / deadline system | P1 | 1 day |
| INFRA-6: Channel escalation logic | P1 | half day |
| INFRA-7: Task restart | P2 | half day |
| UC4: Field sales tracking | P1 | 1 day |
| UC5: Follow-up & SLA management | P1 | 1 day |
| **Total** | | **~5 days** |

### Avinash — Search & Orchestration
| Ticket | Priority | Estimate |
|---|---|---|
| INFRA-3: Perplexity Sonar integration | P0 | half day |
| INFRA-9: Heartbeat notifications | P2 | half day |
| INFRA-10: Event store persistence | P2 | half day |
| UC6: Bakery/grocery coordination | P1 | 1-2 days |
| UC7: Multi-channel escalation | P1 | half day |
| UC8: Generic search→act demo | P2 | half day |
| **Total** | | **~4-5 days** |

---

## Execution Sequence

```
Phase 1 — Infrastructure (parallel, ~2 days)
├── Krishna: INFRA-1 (batch→resume) + INFRA-2 (structured output)
├── Rishav: INFRA-5 (SLA) + INFRA-6 (escalation) + INFRA-4 (doc upload)
└── Avinash: INFRA-3 (Perplexity) + INFRA-9 (heartbeat) + INFRA-10 (event store)

Phase 2 — UC Flows (parallel, ~3 days)
├── Krishna: INFRA-8 (inbound WebRTC) → UC1 → UC2 → UC3
├── Rishav: INFRA-7 (restart) → UC4 → UC5
└── Avinash: UC6 → UC7 → UC8

Phase 3 — Integration + Demo Rehearsal (~1 day)
├── Cross-UC testing (UC2 needs Perplexity, UC3 needs docs)
├── Planner prompt tuning for all 8 UC patterns
├── Demo script rehearsal — run all 8 end-to-end
└── Fix broken flows
```

## Dependency Graph

```
INFRA-1 (batch→resume) ────┬──→ UC2
  └── INFRA-2 (output)     ├──→ UC3
                            └──→ UC6, UC8

INFRA-3 (Perplexity) ──────┬──→ UC2
                            ├──→ UC6
                            └──→ UC8

INFRA-4 (doc upload) ──────→ UC3

INFRA-5 (SLA) + INFRA-6 ──┬──→ UC4
                            ├──→ UC5
                            └──→ UC7

INFRA-7 (restart) ─────────→ UC5

INFRA-8 (inbound WebRTC) ──→ UC1

No cross-developer blocking: each person's infra feeds their own UCs.
Exception: UC2 needs INFRA-3 (Avinash) — Avinash delivers Perplexity on Day 1.
           UC3 needs INFRA-4 (Rishav) — Rishav delivers doc upload on Day 1.
```
