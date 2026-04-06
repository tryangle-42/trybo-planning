Area-by-Area Comparison

  1. Pydantic Models / Contracts

  What they built: runtime/contracts.py — SkillDefinition, NodeSpec, PlanSpec, Edge, Receipt, ExecutionContext, RetryPolicy, IdempotencyPolicy. All Pydantic BaseModel.

  Matches our plan: Yes — this maps directly to our planned PlanSpec, SkillSchema, SkillResult, TaskContext. The contract-first approach is exactly right.

  Divergences:

  Their approach: ExecutionContext carries db: Any inside a Pydantic model
  Our plan: We planned TaskContext as a separate non-Pydantic runtime object
  Better?: Ours — db handles in Pydantic models risk serialization leaks
  ────────────────────────────────────────
  Their approach: $memory.<node_id>.<field> late-binding references
  Our plan: We planned shared state via LangGraph's native state dict
  Better?: Theirs — the $memory convention is cleaner for planner output
  ────────────────────────────────────────
  Their approach: No health_status, success_rate, p50_latency_ms, cost_tier on SkillDefinition
  Our plan: Our plan had full registry schema from roadmap
  Better?: Ours for later — their lean schema is correct for MVP, but missing health/metrics fields means no registry health monitor

  Issues spotted:
  - MissingSkillMode type defined in both contracts.py and schemas.py — duplication
  - our_base_url defaults to localhost — silent production misconfiguration risk

  ---
  2. Skill Registry

  What they built: runtime/registry.py — SkillsRegistry with lookup, policy extraction (frozen SkillPolicy dataclass), plan validation (node uniqueness, edge integrity, schema validation), and $memory.* reference pass-through.

  Matches our plan: Strong match. Registry is authoritative over plan — plan cannot escalate privileges. This is exactly our "planner never invents skill names" non-negotiable.

  What's missing from our plan:
  - No cycle detection in validate_plan() — a cyclic plan would hang
  - No health status — no active | degraded | disabled states, no canary checks, no auto-degrade. Our plan had this as a core registry feature.
  - No capability-based lookup — list_skills_for_planning() dumps everything. Fine for 5 skills, won't scale.
  - Duplicate skill names silently overwrite (last wins, no warning)

  Verdict: Good MVP foundation. Needs cycle detection before production.

  ---
  3. Seed Registry / v1 Skills

  What they built: runtime/seed_registry.py — 5 skills:
  1. channel.whatsapp.send (Direct)
  2. web.research.tavily (MCP)
  3. text.summarize.llm (Direct)
  4. agent.deal_hunter (Agent)
  5. marketplace.composio.slack.post_message (Marketplace)

  Compared to our v1 skill list:

  ┌─────────────────────────────────┬─────────────────────────────────────────────────────┬────────────────────┐
  │            Our plan             │                     Their impl                      │       Status       │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ whatsapp_send                   │ channel.whatsapp.send                               │ Done               │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ call_place                      │ —                                                   │ Missing            │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ contact_lookup                  │ —                                                   │ Missing            │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ conversation_reply              │ —                                                   │ Missing            │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ web_search                      │ web.research.tavily                                 │ Done               │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ gmail_search / gmail_send       │ —                                                   │ Missing            │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ calendar_read / calendar_create │ —                                                   │ Missing            │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ slack_search / slack_send       │ marketplace.composio.slack.post_message (send only) │ Partial            │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ —                               │ text.summarize.llm                                  │ Extra (useful)     │
  ├─────────────────────────────────┼─────────────────────────────────────────────────────┼────────────────────┤
  │ —                               │ agent.deal_hunter                                   │ Extra (demo agent) │
  └─────────────────────────────────┴─────────────────────────────────────────────────────┴────────────────────┘

  Verdict: They proved the adapter pattern works across all 4 types (direct, mcp, marketplace, agent) — that's the important thing. The missing skills are straightforward to add as registrations.

  ---
  4. Adapter Layer

  What they built: runtime/adapters/ — 4 concrete adapters behind a SkillAdapter ABC:
  - DirectAdapter — WhatsApp send + LLM summarize via dispatch table
  - McpAdapterTavily — REST API preferred, MCP JSON-RPC fallback
  - MarketplaceAdapterComposio — Composio REST for Slack
  - AgentAdapter — HTTP broker with bidirectional channels

  Matches our plan: The unified SkillAdapter.run_skill(skill_name, inputs, context) -> (Dict, Receipt) interface is exactly our planned async def invoke(skill_id, input, context) -> SkillResult.

  Divergences:

  Their approach: AgentAdapter breaks the SkillAdapter ABC — has start() + forward_answer() lifecycle
  Our plan: We planned all adapters behind one interface
  Better?: Theirs — external agents are genuinely different (async, bidirectional). Clean break is pragmatic.
  ────────────────────────────────────────
  Their approach: Tavily uses direct REST, not langchain-tavily or langchain-mcp-adapters
  Our plan: We planned langchain-tavily + langchain-mcp-adapters
  Better?: Ours for MCP generality — their REST approach works for Tavily specifically, but won't scale when adding Gmail/Calendar MCP servers. MultiServerMCPClient handles N servers.
  ────────────────────────────────────────
  Their approach: Composio uses raw REST, not composio-langgraph SDK
  Our plan: We planned composio-langgraph
  Better?: Ours — the SDK handles OAuth token refresh, tool discovery, and multi-app support. Raw REST means reimplementing all of that per-tool.

  Issues spotted:
  - _hash_obj() utility duplicated across 3 adapter files
  - Tavily adapter extracts API keys from URL query params (logged by proxies — security risk)
  - Composio adapter is hardcoded to Slack only — naming (MarketplaceAdapterComposio) overpromises
  - No connection pooling — new httpx.AsyncClient per request

  ---
  5. LangGraph Executor

  What they built: Two-tier graph architecture:
  - Outer graph (compiler_graph.py): Fixed 7-node linear pipeline — normalize → clarify → plan → approve → validate → execute → finalize
  - Inner graph (plan_builder.py): Dynamically compiled from PlanSpec — each plan node becomes a LangGraph node

  Matches our plan: The two-tier architecture is better than what we planned (we had a single graph). Separating orchestration lifecycle from plan execution is cleaner.

  Critical divergence — parallelism is lost:

  This is the biggest architectural issue in the branch.

  compile_plan_to_stategraph() converts the DAG into a strictly linear chain via topological sort. Even if two nodes are independent (e.g., WhatsApp send + Slack send after summarize), they execute sequentially. Our plan explicitly called for
  parallel node execution using LangGraph's fan-out.

  OUR PLAN:    summarize → [whatsapp_send | slack_send]  (parallel)
  THEIR IMPL:  summarize → whatsapp_send → slack_send    (sequential)

  The _stop flag compounds this — if any node fails, ALL subsequent nodes are skipped, even independent ones.

  Other divergences:

  Their approach: _skip_execute boolean flag threaded through state
  Our plan: LangGraph conditional edges
  Better?: Ours — conditional edges are the idiomatic pattern
  ────────────────────────────────────────
  Their approach: In-memory checkpointing only (STORE singleton)
  Our plan: langgraph-checkpoint-postgres to Supabase
  Better?: Ours — in-memory state is lost on restart, violating the roadmap non-negotiable
  ────────────────────────────────────────
  Their approach: No interrupt() usage for consent — uses event store + polling
  Our plan: LangGraph native interrupt() + Command(resume=)
  Better?: Ours — their approach reinvents what LangGraph provides natively, and doesn't actually pause the graph (it polls an event store in a loop)

  Issues spotted:
  - finalize node may lose data (returns result dict directly instead of {"result": result})
  - Silent except Exception: pass on all event emissions — observability can fail silently
  - _run_node is a 100-line monolith mixing idempotency, approval, execution, and event emission
  - assert used for runtime validation (stripped with -O flag)

  ---
  6. Planner

  What they built: runtime/langgraph/llm_planner.py — LLMPlanner with two phases:
  1. identify_next_question() — asks LLM if critical info is missing
  2. generate_plan() — produces PlanSpec JSON with 3-retry self-correcting loop

  Matches our plan: Very close. The retry-with-error-feedback loop matches our planned "re-prompt LLM with error context (max 2 retries)". They do 3 retries, which is fine.

  Good additions we didn't plan:
  - Auto-generated skill catalog from registry for the planner prompt
  - Phone number masking in clarification summaries (PII protection)
  - Clarification phase before plan generation

  Issues spotted:
  - Prompt injection: User query is in the system message, not user message — makes injection easier
  - Hardcoded skill routing hints in prompt ("For deal hunting, use agent.deal_hunter") — tight coupling
  - No token counting or prompt length management
  - identify_next_question silently returns None on all errors

  ---
  7. Consent / HITL

  What they built: Event-based approval flow:
  1. Side-effect node emits approval.requested event to EVENTS store
  2. Background task polls the event store for approval
  3. User hits POST /api/agent/runs/{run_id}/approval endpoint
  4. Event store sets approval → poll loop picks it up → execution continues

  Compared to our plan (LangGraph interrupt()):

  ┌─────────────────────────────────────────────┬───────────────────────────────────────────────────┐
  │               Their approach                │                     Our plan                      │
  ├─────────────────────────────────────────────┼───────────────────────────────────────────────────┤
  │ Custom polling loop in _run_node            │ LangGraph interrupt() pauses graph natively       │
  ├─────────────────────────────────────────────┼───────────────────────────────────────────────────┤
  │ In-memory event store (lost on restart)     │ Postgres checkpoint (survives restart)            │
  ├─────────────────────────────────────────────┼───────────────────────────────────────────────────┤
  │ Manual state management                     │ LangGraph handles state persistence automatically │
  ├─────────────────────────────────────────────┼───────────────────────────────────────────────────┤
  │ ~100 lines of custom approval orchestration │ ~20 lines using interrupt()/Command(resume=)      │
  └─────────────────────────────────────────────┴───────────────────────────────────────────────────┘

  Verdict: Ours is significantly better. They reimplemented what LangGraph provides for free, and their version doesn't survive process restarts. The interrupt() pattern gives you pause, checkpoint to Postgres, resume from any process — exactly
  what the roadmap requires.

  What they did well: The approval/clarification event types, the structured ApprovalRequestedPayload, and the SSE streaming of events to the frontend are all good patterns we should keep.

  ---
  8. GID (Global Intent Detector)

  What they built: The clarify_intent node in compiler_graph.py uses the LLM planner's identify_next_question() — it asks the LLM if info is missing and routes to clarification if so.

  Compared to our plan: We planned semantic-router for sub-10ms embedding-based routing (FAST_PATH | PLANNER | CLARIFY). They don't have a GID at all — every request goes through the full planner pipeline.

  Verdict: Missing entirely. No fast-path for simple queries, no confidence-based routing. Every "What time is it?" goes through the full plan → approve → execute pipeline. This is a v2 addition but important for the roadmap's "GID fast path must
   be sub-500ms" non-negotiable.

  ---
  9. Memory Layer

  What they built: No mem0 integration. The ExecutionContext carries a flat dict of prior_clarifications and approvals. No long-term memory, no user preferences, no semantic search.

  Compared to our plan: We planned mem0 as always-on infrastructure — retrieve memories before GID/planner, store new facts after graph completion.

  Verdict: Missing entirely. This is expected for a Phase 1 MVP but must be added.

  ---
  10. Audit Log

  What they built: runtime/store.py (InMemoryRunStore) with timeline events + runtime/event_store.py with SSE-capable event streaming. Comprehensive event taxonomy in run_events.py (~24 event types).

  Compared to our plan: We planned a task_audit_log Supabase table with per-node writes.

  Verdict: Their event system is richer than what we planned — typed events, SSE streaming, redaction utilities. But it's all in-memory (lost on restart). We need to persist these events to Supabase.

  ---
  11. Observability / Events

  Not in our plan, but they built it well. The run_events.py event catalog, event_store.py SSE pubsub, and agent_runs.py streaming endpoint together form a real-time observability system. This is a genuine addition beyond our plan.

  Issues:
  - subscribe() never terminates for completed runs (SSE stream runs forever)
  - Race condition: lost wakeup between list_events and cond.wait()
  - created_at/updated_at fields are never populated

  ---
  12. Agent-to-Agent Communication

  Not in our plan for v1, but they built it. AgentAdapter + channels.py + agent_messages.py + the Deal Hunter demo agent form a working bidirectional agent communication system. This is genuinely ahead of our plan.

  Issues:
  - agent_inbox/agent_outbox endpoints have zero authentication — anyone can inject messages
  - No channel timeout/cleanup — stale channels leak memory
  - Process-local only (won't work with multiple workers)

  ---
  13. Duplicate dynamic/ Module

  The entire app/agentic_mvp/dynamic/ directory (5 files) is dead code — an earlier prototype superseded by runtime/. Neither route imports anything from it. Should be deleted.

  ---
  14. Service File Changes (Non-Agentic)

  Critical merge issues:
  - WhatsApp reaction handling (TB-583) will be reverted — ~170 lines deleted due to branch divergence
  - Batch completion approach conflicts — dev uses asyncio.create_task (TB-560), branch re-introduces BackgroundTasks
  - Deepgram disabled by default — breaking change for existing deployments without streaming ASR

  ---
  Security Issues Summary

  ┌──────────┬──────────────────────────────────────────────────────────────────────┬─────────────────────┐
  │ Severity │                                Issue                                 │        File         │
  ├──────────┼──────────────────────────────────────────────────────────────────────┼─────────────────────┤
  │ HIGH     │ No authorization scoping — any authenticated user can access any run │ agent_runs.py       │
  ├──────────┼──────────────────────────────────────────────────────────────────────┼─────────────────────┤
  │ HIGH     │ agent_inbox/agent_outbox have zero authentication                    │ agent_runs.py       │
  ├──────────┼──────────────────────────────────────────────────────────────────────┼─────────────────────┤
  │ HIGH     │ Deal Hunter /run endpoint has no authentication                      │ deal_hunter/main.py │
  ├──────────┼──────────────────────────────────────────────────────────────────────┼─────────────────────┤
  │ MEDIUM   │ API keys accepted in request body (leak to logs)                     │ schemas.py          │
  ├──────────┼──────────────────────────────────────────────────────────────────────┼─────────────────────┤
  │ MEDIUM   │ API keys extracted from URL query params                             │ mcp_tavily.py       │
  ├──────────┼──────────────────────────────────────────────────────────────────────┼─────────────────────┤
  │ MEDIUM   │ Prompt injection — user query in system message                      │ llm_planner.py      │
  ├──────────┼──────────────────────────────────────────────────────────────────────┼─────────────────────┤
  │ MEDIUM   │ _RUN_TASKS dict leaks memory with no cleanup                         │ agent_runs.py       │
  └──────────┴──────────────────────────────────────────────────────────────────────┴─────────────────────┘

  ---
  Final Verdict: KEEP vs CHANGE/ADD

  KEEP (good as-is or better than our plan)

  1. Two-tier graph architecture (outer lifecycle + inner execution) — cleaner than our single-graph plan
  2. Contract-first design (contracts.py) — all types defined upfront, Pydantic models, clean separation
  3. Registry-authoritative policy — plan cannot override requires_consent or side_effect. Security-correct.
  4. $memory.<node>.<field> reference system — elegant late-binding for data flow between nodes
  5. Adapter pattern with 4 types — proved Direct, MCP, Marketplace, and Agent adapters all work
  6. LLM planner with retry-and-feedback — self-correcting plan generation, 3 retries with error context
  7. Clarification phase — asks for missing info before planning. We didn't explicitly plan this.
  8. Event taxonomy (run_events.py) — 24 typed events with versioning, redaction utilities, SSE streaming
  9. Agent communication protocol — AgentMessage, AgentChannel, bidirectional HTTP. Ahead of our plan.
  10. Deal Hunter demo agent — proves the external agent pattern works end-to-end
  11. Deterministic topo sort with lexical tie-breaking — reproducible execution order
  12. Idempotency via content-addressable hashing — prevents duplicate side effects on reruns
  13. Plan validation with graceful degradation — removes nodes with missing skills, cascading dependency cleanup
  14. seed_registry.py as configuration-as-code — clean, declarative skill definitions

  CHANGE/ADD (specific changes needed)

  ┌──────────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────────────┐
  │ Priority │                                                  Change                                                  │                                        Why                                        │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P0       │ Add langgraph-checkpoint-postgres → replace in-memory STORE with Postgres checkpointing                  │ Roadmap non-negotiable: "in-memory state is not acceptable for production"        │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P0       │ Replace custom approval polling with LangGraph interrupt() / Command(resume=)                            │ Eliminates ~100 lines of custom code, survives restarts, is the idiomatic pattern │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P0       │ Add authorization scoping to all agent_runs.py endpoints (verify user owns run)                          │ Security: any auth'd user can currently access any run                            │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P0       │ Add authentication to agent_inbox/agent_outbox (HMAC or shared secret)                                   │ Security: currently zero auth                                                     │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P0       │ Fix merge conflicts: rebase onto dev to restore TB-583 (reactions) and TB-560 (batch)                    │ Branch divergence will revert merged features                                     │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P1       │ Implement parallel fan-out in compile_plan_to_stategraph() using LangGraph Send or parallel edges        │ Currently all execution is sequential — defeats DAG purpose                       │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P1       │ Replace raw REST Tavily/Composio with langchain-mcp-adapters MultiServerMCPClient + composio-langgraph   │ Scales to Gmail/Calendar/Slack MCP servers without per-tool REST code             │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P1       │ Add mem0 integration as ambient infrastructure                                                           │ Memory should be always-on context, not a skill                                   │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P1       │ Add GID with semantic-router for fast-path routing                                                       │ Currently every request goes through full planner pipeline                        │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P1       │ Persist events to Supabase task_audit_log table                                                          │ All observability data is lost on restart                                         │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P1       │ Add missing P0 skills: call_place, contact_lookup, conversation_reply, gmail_*, calendar_*, slack_search │ 6 of our 11 launch skills are missing                                             │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P2       │ Delete app/agentic_mvp/dynamic/ directory                                                                │ Dead code — entire prototype superseded by runtime/                               │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P2       │ Delete dead imports in graph.py (chat_completion, send_outbound_text, WebSearchService, json)            │ Code hygiene                                                                      │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P2       │ Extract _hash_obj() into shared utility (duplicated in 3 adapter files)                                  │ DRY                                                                               │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P2       │ Add cycle detection to validate_plan()                                                                   │ Safety: cyclic plans would hang                                                   │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P2       │ Move user query from system to user message in planner prompts                                           │ Reduces prompt injection surface                                                  │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P2       │ Add SecretStr for API keys in config                                                                     │ Prevent accidental logging of secrets                                             │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P2       │ Fix subscribe() to terminate on terminal run status                                                      │ SSE streams currently run forever                                                 │
  ├──────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
  │ P2       │ Add feature flag for v2 agent_runs endpoints                                                             │ Currently always active — no kill switch                                          │
  └──────────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────┴───────────────────────────────────────────────────────────────────────────────────┘

  ---
  Want me to start executing on these changes, or do you want to discuss priorities first?