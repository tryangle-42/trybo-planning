Current State vs. Roadmap Gap Analysis
  What Already Exists (solid foundation)
  ┌───────────────────────────────────────────────┬────────┬─────────────────────────────────────────────────────┐
  │                  Capability                   │ Status │                      Key Files                      │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ FastAPI backend with middleware stack         │ Done   │ app/main.py                                         │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ Chat system (Supabase realtime, last 20 msgs) │ Done   │ app/services/chat_service.py                        │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ Batch task execution model                    │ Done   │ orchestrator_service.py, batch_service.py           │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ WhatsApp send/receive (GreenAPI + Meta)       │ Done   │ whatsapp_service.py                                 │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ Voice calls (Twilio + Knowlarity)             │ Done   │ call_service.py, telephony providers                │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ Intent detection (rule-based + LLM)           │ Done   │ intent_service.py                                   │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ Mini planner (multi-contact decomposition)    │ Done   │ planner_mini.py                                     │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ Consent request lifecycle                     │ Done   │ consent_request_store.py, consent_action_service.py │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ LLM abstraction (OpenAI/Gemini/Ollama)        │ Done   │ llm_service.py                                      │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ Fast response pattern (<500ms)                │ Done   │ orchestrator_service.py                             │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ JWT auth + Supabase RLS                       │ Done   │ app/deps.py                                         │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ Firebase push notifications                   │ Done   │ firebase_service.py                                 │
  ├───────────────────────────────────────────────┼────────┼─────────────────────────────────────────────────────┤
  │ Google OAuth + contacts sync                  │ Done   │ google_service.py                                   │
  └───────────────────────────────────────────────┴────────┴─────────────────────────────────────────────────────┘

  What the Roadmap Requires (new orchestration layer)
┌────────────────────────────────────────┬──────────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────────┐
  │               Component                │                             Exists?                              │                                             Gap                                              │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Global Intent Detector (GID)           │ Partial — intent_service.py only decides call vs WhatsApp        │ Needs full routing: FAST_PATH | PLANNER | CLARIFY with confidence scoring, entity extraction │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Skill/Agent Registry                   │ No                                                               │ Entirely new — unified registry with schema, health checks, adapter types                    │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Planner (full)                         │ Partial — planner_mini.py only decomposes contacts               │ Needs full PlanSpec generation: task graph with dependencies, skill selection, validation    │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ LangGraph Executor                     │ No                                                               │ Entirely new — StateGraph builder from PlanSpec, node types, checkpoint persistence          │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Adapter layer (unified)                │ No                                                               │ Direct/MCP/Marketplace adapters behind a single invoke() interface                           │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Consent gate (in-graph)                │ Partial — consent exists but not integrated into graph execution │ Needs interrupt() / resume flow within LangGraph                                             │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Memory layer (mem0)                    │ No                                                               │ Long-term memory via mem0 MCP is new                                                         │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ TaskContext shared state               │ No                                                               │ New working memory model for graph execution                                                 │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Audit log (per-node)                   │ No                                                               │ New task_audit_log table + writer                                                            │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Registry health monitor                │ No                                                               │ Canary checks, passive telemetry, auto-degrade                                               │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ WebRTC room + SMS link                 │ No                                                               │ New direct adapter skills                                                                    │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ MCP adapters (Tavily, Gmail, Calendar) │ No                                                               │ New adapter type + integrations                                                              │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Marketplace adapter (Composio)         │ No                                                               │ New adapter type                                                                             │
  ├────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Bidirectional execution                │ Partial — inbound WhatsApp/calls exist                           │ Needs unified entrypoint through the new orchestration pipeline                              │

  Key Dependencies to Add

  The project currently has no LangGraph, no MCP client libraries, no mem0, and no Composio SDK in requirements.txt.
  ---
  Architecture Summary

  CURRENT:  User → Chat → Orchestrator → PlannerMini → Batch → [WhatsApp | Call]
                                                                      ↑ flat, no graph
  ROADMAP:  User → Chat → GID ──→ FastPath (single skill, no side-effect)
                           └──→ Planner → PlanSpec → LangGraph Executor
                                                        ├─ Tool nodes
                                                        ├─ Agent nodes (sub-LLM)
                                                        ├─ Consent nodes (interrupt/resume)
                                                        └─ Condition nodes (routing)
                                                     :arrow_up_down:
                                                Skill Registry
                                             (Direct | MCP | Marketplace)

Reuse vs. Build — Component-by-Component Verdict

  1. Executor (Task Graph Runtime)

  Column 1: Use
  Column 2: langgraph v1.0.9
  ────────────────────────────────────────
  Column 1: Why
  Column 2: StateGraph builder, parallel node execution (Pregel supersteps), Send for dynamic fan-out, sub-graphs for ReAct agent
    nodes, interrupt()/Command(resume=) for HITL — all first-class, production-grade, 25k stars, weekly releases
  ────────────────────────────────────────
  Column 1: Build
  Column 2: ~50-line build_graph_from_planspec() function that translates your PlanSpec JSON into add_node/add_edge calls. LangGraph
    has no declarative JSON-to-graph parser — but the builder API is trivial to wrap.
  ────────────────────────────────────────
  Column 1: Packages
  Column 2: langgraph, langgraph-prebuilt (for pre-built ReAct agents)

  2. Checkpointing / Resumability

  Column 1: Use
  Column 2: langgraph-checkpoint-postgres v3.0.4
  ────────────────────────────────────────
  Column 1: Why
  Column 2: First-party, async, works directly with Supabase's Postgres connection string. Call .setup() once → tables created → done.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: Nothing. Zero custom code.
  ────────────────────────────────────────
  Column 1: Packages
  Column 2: langgraph-checkpoint-postgres, psycopg[binary]

  3. Consent / HITL Layer

  Column 1: Use
  Column 2: LangGraph interrupt() + Command(resume=)
  ────────────────────────────────────────
  Column 1: Why
  Column 2: Your consent gate maps directly to interrupt(). Node pauses → surfaces approval artifact to mobile UI via your existing
    Supabase realtime → user taps Approve/Edit/Deny → Command(resume={"decision": "APPROVED", "edited_content": ...}) → graph
    continues. State persisted in Postgres checkpoint between pause and resume.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: Consent artifact formatting (the payload you surface to the mobile app), approval policy logic (auto-approve low-risk
    repeat actions), and the REST endpoint that accepts the user's decision and calls graph.ainvoke(Command(resume=...)). ~150 lines
  of
     custom glue.
  ────────────────────────────────────────
  Column 1: Skip
  Column 2: HumanLayer SDK — adds Slack/Email delivery you don't need. Your delivery channel is the mobile app via Supabase realtime.

  4. Global Intent Detector (GID)

  Column 1: Use
  Column 2: semantic-router (Aurelio Labs, 3.1k stars)
  ────────────────────────────────────────
  Column 1: Why
  Column 2: Embedding-based routing, sub-10ms, no LLM call. Define routes with sample utterances + confidence thresholds. Unmatched
    queries fall through to planner. This is exactly the FAST_PATH | PLANNER | CLARIFY pattern.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: Route definitions (utterance examples per intent), entity extraction (LLM call only when needed — on PLANNER/CLARIFY
    paths), confidence threshold tuning. Wrap semantic-router in a GIDService that returns routing_decision, intent_type, entities,
    confidence_score. ~200 lines.
  ────────────────────────────────────────
  Column 1: Note
  Column 2: Your existing intent_service.py (call vs WhatsApp) becomes one downstream step after GID routes to FAST_PATH, not a
    replacement for GID.
  ────────────────────────────────────────
  Column 1: Packages
  Column 2: semantic-router

  5. Planner (PlanSpec Generation)

  Column 1: Use
  Column 2: LangGraph plan-and-execute pattern + Pydantic for schema validation
  ────────────────────────────────────────
  Column 1: Why
  Column 2: LangGraph has a plan-and-execute template. Pydantic gives you schema-validated structured output from the LLM — define
    PlanSpec as a Pydantic model, LLM generates conformant JSON, validator catches hallucinated skill IDs and circular deps.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: PlanSpec Pydantic model, the planner prompt (system prompt with registry context + user goal → structured JSON), the
    validation pass (skill_id exists in registry, input matches schema, no cycles, consent flags match), re-prompt logic on failure.
    ~300 lines. This is your core IP — the planner prompt and validation logic.
  ────────────────────────────────────────
  Column 1: Packages
  Column 2: Already have pydantic, openai. Use LLM structured output (response_format).

  6. Skill / Agent Registry

  Column 1: Use
  Column 2: Postgres (Supabase) as the store — no special framework needed
  ────────────────────────────────────────
  Column 1: Why
  Column 2: Your registry is a table with JSON schema columns. langgraph-bigtool exists for semantic tool retrieval, but your planner
    already gets filtered skills as context — you don't need embedding-based search over 500 tools. You have 5-7 skills at v1. A
    Postgres table + a Python service class is the right tool for the job.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: skills table in Supabase, RegistryService (CRUD + health status + filtering), skill schema as a Pydantic model matching
  the
    roadmap's JSON schema. Health monitor as a scheduled job (you already have APScheduler). ~250 lines.
  ────────────────────────────────────────
  Column 1: Skip
  Column 2: langgraph-bigtool (overkill for <10 skills), A2A Agent Cards (future consideration, not v1), MCP Registry (public catalog,

    not your private registry).

  7. Adapter Layer (Direct / MCP / Marketplace)

  Column 1: Use
  Column 2: langchain-mcp-adapters v0.2.1 for MCP, composio-langgraph for Marketplace
  ────────────────────────────────────────
  Column 1: Why
  Column 2: langchain-mcp-adapters provides MultiServerMCPClient — connect to multiple MCP servers (Tavily, Gmail, Calendar,
    Playwright, mem0), get LangChain-compatible tools, plug directly into LangGraph ToolNode. Composio gives you 500+ pre-built
    connectors with managed OAuth via composio-langgraph.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: UnifiedAdapter interface (async def invoke(skill_id, input, context) -> SkillResult) that dispatches to the right adapter
    type based on adapter_type in the registry. Direct adapter wraps your existing WhatsApp/Call/WebRTC services. MCP adapter wraps
    MultiServerMCPClient. Marketplace adapter wraps Composio. ~200 lines of glue.
  ────────────────────────────────────────
  Column 1: Packages
  Column 2: langchain-mcp-adapters, composio, composio-langgraph, mcp

  8. Web Search Skill

  Column 1: Use
  Column 2: langchain-tavily v0.2.17
  ────────────────────────────────────────
  Column 1: Why
  Column 2: First-party LangChain integration, plugs directly into LangGraph tool nodes. TavilySearch(max_results=5) — done. Free
  tier:
    1,000 searches/month.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: Nothing. Register it in your skill registry, wire it up.
  ────────────────────────────────────────
  Column 1: Packages
  Column 2: langchain-tavily

  9. Memory Layer (Long-term)

  Column 1: Use
  Column 2: mem0ai v1.0.4 with Supabase/pgvector backend
  ────────────────────────────────────────
  Column 1: Why
  Column 2: 44k stars, $24M funded, auto fact extraction, deduplication, user-scoped memory, semantic search. Native Supabase support
  —
    your data stays in your own Postgres. Official LangGraph integration pattern exists.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: Config (point mem0 at your Supabase), initialization in FastAPI lifespan, integration into TaskContext (retrieve relevant
    memories before planner runs, store new facts after graph completes). ~100 lines.
  ────────────────────────────────────────
  Column 1: Note
  Column 2: mem0's Memory class is synchronous — wrap in asyncio.to_thread() for FastAPI.
  ────────────────────────────────────────
  Column 1: Packages
  Column 2: mem0ai

  10. Audit Log

  Column 1: Use
  Column 2: Nothing — this is a simple Supabase insert
  ────────────────────────────────────────
  Column 1: Why
  Column 2: It's a task_audit_log table + a write_audit_entry() function called by each graph node. No framework needed for a table
    write.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: Table schema, AuditService with log_node_execution(). ~50 lines.

  11. Bidirectional Execution (Inbound Pipeline)

  Column 1: Use
  Column 2: Your existing webhook endpoints + the new orchestration pipeline
  ────────────────────────────────────────
  Column 1: Why
  Column 2: You already have WhatsApp webhooks, call status callbacks, etc. The change is routing these through GID → Planner →
    Executor instead of the current flat handler.
  ────────────────────────────────────────
  Column 1: Build
  Column 2: Refactor inbound webhook handlers to create a TaskContext with entrypoint_type=INBOUND and feed it into the same GID →
    Planner → Executor pipeline. ~150 lines of refactoring.

  ---
  Summary: What You're Actually Building

  ┌────────────────────────┬──────────────────────────────────┬─────────────────────────────────────────────────────────────┐
  │         Layer          │           Reuse (OSS)            │                       Build (Custom)                        │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Graph runtime          │ LangGraph                        │ -                                                           │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Checkpointing          │ langgraph-checkpoint-postgres    │ -                                                           │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ HITL/Consent           │ LangGraph interrupt()            │ Consent artifact format, approval policies, resume endpoint │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Intent routing         │ semantic-router                  │ Route definitions, entity extraction, GIDService wrapper    │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Plan generation        │ Pydantic + LLM structured output │ PlanSpec model, planner prompt, validation, re-prompt logic │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Skill registry         │ Supabase (Postgres)              │ skills table, RegistryService, health monitor               │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Adapters (MCP)         │ langchain-mcp-adapters           │ -                                                           │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Adapters (Marketplace) │ composio-langgraph               │ -                                                           │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Adapters (Direct)      │ Existing WhatsApp/Call services  │ UnifiedAdapter dispatch wrapper                             │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Web search             │ langchain-tavily                 │ -                                                           │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Memory                 │ mem0ai (Supabase backend)        │ Config + integration glue                                   │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Audit                  │ Supabase                         │ Table + 50-line service                                     │
  ├────────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Bidirectional          │ Existing webhooks                │ Reroute through new pipeline                                │
  └────────────────────────┴──────────────────────────────────┴─────────────────────────────────────────────────────────────┘

  Custom code estimate: ~1,000-1,200 lines of new orchestration logic. Everything else is wiring maintained OSS together.

  New dependencies to add:
  langgraph
  langgraph-prebuilt
  langgraph-checkpoint-postgres
  psycopg[binary]
  semantic-router
  langchain-mcp-adapters
  langchain-tavily
  composio
  composio-langgraph
  mem0ai
  mcp

  ---
  Ready to move to phased implementation planning when you are. Want me to break this into sprint-sized phases with dependency
  ordering?

✻ Churned for 3m 30s




  ---
  1. Is GID extremely simple right now?

  Yes, intentionally. At v1 with 5-7 skills, semantic-router does embedding-based matching against sample utterances — no LLM
  call, sub-10ms. It returns one of three routes: FAST_PATH, PLANNER, CLARIFY.

  But it's fully extensible. The design scales to Agent-of-Agents OS because:
  - Adding a new skill = adding new route utterances to the router. No code change.
  - As skills grow to 50+, you swap in a hybrid approach: semantic-router for fast coarse routing → LLM for fine-grained
  disambiguation. The GIDService interface stays the same.
  - Entity extraction (who, what, when) already runs as a separate step on PLANNER/CLARIFY paths — it's decoupled from
  routing.
  - Confidence thresholds are tunable per route, so you can tighten/loosen as you learn from production traffic.

  The GID is a thin, hot-swappable routing layer — simple now, but the abstraction boundary is in the right place.

  ---
  2. By Skills do you mean Agents/Tools/MCPs all?

  Yes. "Skill" is the unified abstraction in the registry. Every capability — regardless of how it's implemented — registers
  as a skill with the same schema:

  ┌─────────────────────────────────┬───────────────┬──────────────┬───────────────────────────────┐
  │    Underlying Implementation    │     kind      │ adapter_type │            Example            │
  ├─────────────────────────────────┼───────────────┼──────────────┼───────────────────────────────┤
  │ Your own code (WhatsApp, Calls) │ tool          │ direct       │ whatsapp_send, call_place     │
  ├─────────────────────────────────┼───────────────┼──────────────┼───────────────────────────────┤
  │ MCP server (Tavily, Gmail)      │ tool          │ mcp          │ web_search, gmail_send        │
  ├─────────────────────────────────┼───────────────┼──────────────┼───────────────────────────────┤
  │ Sub-LLM with tool access        │ agent         │ direct       │ research_vendors (ReAct loop) │
  ├─────────────────────────────────┼───────────────┼──────────────┼───────────────────────────────┤
  │ External platform (Composio)    │ tool or agent │ marketplace  │ slack_post, notion_create     │
  └─────────────────────────────────┴───────────────┴──────────────┴───────────────────────────────┘

  The planner doesn't know or care whether a skill is a direct function call, an MCP server, or a Composio connector. It sees:
   skill_id, capabilities, input_schema, side_effect, requires_consent. The UnifiedAdapter dispatches to the right adapter
  type at runtime.

  ---
  3. What to build first vs. later?

  This is the core question. The answer is: build the skeleton first, then add skills incrementally. Here's the dependency
  order:

  Phase 1 — The Spine (must exist before any skill works)
  - Pydantic models (PlanSpec, SkillResult, TaskContext, SkillSchema)
  - Skill Registry (Supabase table + RegistryService)
  - Unified Adapter interface (dispatch layer)
  - Executor (PlanSpec → LangGraph StateGraph builder + runner)
  - Checkpointing (Postgres)
  - Audit log

  Phase 2 — Routing & Planning (makes it intelligent)
  - GID (semantic-router + GIDService wrapper)
  - Planner (LLM structured output → PlanSpec + validator)
  - Consent gate (interrupt/resume + approval endpoint)
  - Wire into existing chat endpoint (replace current orchestrator)

  Phase 3 — Skills (one by one, each is independent)
  - Register existing WhatsApp + Calls as Direct adapter skills
  - Add Tavily web search (MCP/LangChain)
  - Add mem0 memory layer
  - Add Gmail / Calendar (MCP)
  - Add Composio marketplace connector

  Phase 4 — Polish & Bidirectional
  - Inbound pipeline through GID → Planner → Executor
  - Registry health monitor (APScheduler canary checks)
  - Batch approval cards for multi-recipient actions

  ---
  4. Should we list the first set of skills before building?

  Absolutely yes. The skill list drives everything — the registry schema, the adapter types you need, the GID routes, and the
  planner's context window. You should nail this down first.

  Here's my proposed v1 launch skill set based on the roadmap + what already exists in the codebase:

  ┌─────┬────────────────────┬──────┬─────────────┬─────────────┬─────────────────────┬──────────┐
  │  #  │       Skill        │ Kind │   Adapter   │ Side Effect │    Exists Today?    │ Priority │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 1   │ whatsapp_send      │ tool │ direct      │ Yes         │ Yes (wrap existing) │ P0       │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 2   │ call_place         │ tool │ direct      │ Yes         │ Yes (wrap existing) │ P0       │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 3   │ web_search         │ tool │ mcp         │ No          │ No (add Tavily)     │ P0       │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 4   │ contact_lookup     │ tool │ direct      │ No          │ Yes (wrap existing) │ P0       │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 5   │ conversation_reply │ tool │ direct      │ No          │ Yes (wrap existing) │ P0       │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 6   │ gmail_send         │ tool │ mcp         │ Yes         │ No (add MCP)        │ P1       │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 7   │ gmail_search       │ tool │ mcp         │ No          │ No (add MCP)        │ P1       │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 8   │ calendar_read      │ tool │ mcp         │ No          │ No (add MCP)        │ P1       │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 9   │ calendar_create    │ tool │ mcp         │ Yes         │ No (add MCP)        │ P1       │
  ├─────┼────────────────────┼──────┼─────────────┼─────────────┼─────────────────────┼──────────┤
  │ 10  │ slack_post         │ tool │ marketplace │ Yes         │ No (add Composio)   │ P1       │
  └─────┴────────────────────┴──────┴─────────────┴─────────────┴─────────────────────┴──────────┘

  P0 = March 13 launch. P1 = fast follow.

  Skills 1, 2, 4, 5 are wrapping existing code — no new integration, just registering them with the unified schema and writing
   a thin Direct adapter. Skill 3 (web search) is the only net-new P0 integration.

  ---
  My recommendation for next steps:

  1. You confirm/adjust the v1 skill list above
  2. I build Phase 1 (the spine) — models, registry, adapter interface, executor
  3. We register the existing skills (WhatsApp, Calls, Contacts) as proof the spine works
  4. Then Phase 2 (GID + Planner + Consent) on top

  Want to finalize the skill list and kick off Phase 1?


  ---
  Full Analysis: Trybo v1 Skill/Integration Strategy

  ---
  1. BhindiAI Ecosystem Analysis

  Bhindi (bhindi.io) is the closest direct competitor — same "text-to-action" thesis, same India+US early adopter target. Key intel:

  - 200+ app integrations, 300+ agents, $4M pre-seed at $28M valuation, 5K active users
  - Deliberately rejected MCP — uses proprietary agent spec (GET /tools, POST /tools/:toolName)
  - Covers: Gmail, Calendar, Docs, Sheets, Slack, Telegram, GitHub, Linear, Trello, Groww, CoinGecko, Reddit, YouTube, image/audio
  generation
  - Weaknesses: No voice/calling, no WhatsApp execution, no HITL consent gates, no bidirectional (inbound) handling, text-only

  Trybo's moat vs Bhindi: Voice calls, WhatsApp execution, consent gates, bidirectional inbound/outbound, India-first real-world task
  execution (not just SaaS tool chaining).

  ---
  2. Categorized Integration Landscape

  A. High Daily Frequency (Personal) — drives daily opens

  ┌───────────────────────┬─────────────────────────┬─────────────┬───────────────────────────────────┐
  │      Integration      │     API Available?      │ MCP Server? │            Stickiness             │
  ├───────────────────────┼─────────────────────────┼─────────────┼───────────────────────────────────┤
  │ WhatsApp send/receive │ Yes (Meta/GreenAPI)     │ Community   │ VERY HIGH — you already have this │
  ├───────────────────────┼─────────────────────────┼─────────────┼───────────────────────────────────┤
  │ Voice calls           │ Yes (Twilio/Knowlarity) │ No          │ VERY HIGH — you already have this │
  ├───────────────────────┼─────────────────────────┼─────────────┼───────────────────────────────────┤
  │ Zomato ordering       │ Official MCP            │ Yes         │ HIGH — daily use India            │
  ├───────────────────────┼─────────────────────────┼─────────────┼───────────────────────────────────┤
  │ Swiggy ordering       │ Official MCP (Jan 2026) │ Yes         │ HIGH — daily use India            │
  ├───────────────────────┼─────────────────────────┼─────────────┼───────────────────────────────────┤
  │ Google Calendar       │ MCP servers exist       │ Yes         │ HIGH — daily scheduling           │
  ├───────────────────────┼─────────────────────────┼─────────────┼───────────────────────────────────┤
  │ Gmail                 │ MCP servers exist       │ Yes         │ HIGH — daily comms                │
  ├───────────────────────┼─────────────────────────┼─────────────┼───────────────────────────────────┤
  │ Reminders / To-dos    │ Build on top of mem0    │ No          │ MEDIUM                            │
  └───────────────────────┴─────────────────────────┴─────────────┴───────────────────────────────────┘

  B. Prosumer / Productivity

  ┌────────────────────┬─────────────────────────┬─────────────┬────────────┐
  │    Integration     │     API Available?      │ MCP Server? │ Stickiness │
  ├────────────────────┼─────────────────────────┼─────────────┼────────────┤
  │ Slack              │ Official MCP (Feb 2026) │ Yes         │ HIGH       │
  ├────────────────────┼─────────────────────────┼─────────────┼────────────┤
  │ Notion             │ Via Composio            │ Yes         │ MEDIUM     │
  ├────────────────────┼─────────────────────────┼─────────────┼────────────┤
  │ Google Docs/Sheets │ Via Composio/MCP        │ Yes         │ MEDIUM     │
  ├────────────────────┼─────────────────────────┼─────────────┼────────────┤
  │ Linear/Trello      │ Via Composio            │ No          │ LOW-MEDIUM │
  ├────────────────────┼─────────────────────────┼─────────────┼────────────┤
  │ GitHub             │ Via Composio            │ Yes         │ MEDIUM     │
  └────────────────────┴─────────────────────────┴─────────────┴────────────┘

  C. Finance / Trading — high-value prosumer

  ┌────────────────────┬───────────────────────────┬─────────────────┬─────────────────┐
  │    Integration     │      API Available?       │   MCP Server?   │   Stickiness    │
  ├────────────────────┼───────────────────────────┼─────────────────┼─────────────────┤
  │ Zerodha Kite       │ Full REST API (Rs 500/mo) │ Yes (read-only) │ VERY HIGH       │
  ├────────────────────┼───────────────────────────┼─────────────────┼─────────────────┤
  │ Groww              │ REST API (Rs 499/mo)      │ No              │ HIGH            │
  ├────────────────────┼───────────────────────────┼─────────────────┼─────────────────┤
  │ Angel One SmartAPI │ Free REST API             │ No              │ HIGH            │
  ├────────────────────┼───────────────────────────┼─────────────────┼─────────────────┤
  │ Dhan               │ Free REST API (<50ms)     │ No              │ HIGH            │
  ├────────────────────┼───────────────────────────┼─────────────────┼─────────────────┤
  │ Razorpay payments  │ Official MCP              │ Yes             │ HIGH (merchant) │
  ├────────────────────┼───────────────────────────┼─────────────────┼─────────────────┤
  │ CoinGecko          │ Free API                  │ Community       │ MEDIUM          │
  └────────────────────┴───────────────────────────┴─────────────────┴─────────────────┘

  D. Commerce / Delivery

  ┌─────────────┬──────────────────────────┬───────────────────────┐
  │ Integration │      API Available?      │      Feasibility      │
  ├─────────────┼──────────────────────────┼───────────────────────┤
  │ Zomato      │ Official MCP             │ HIGH (payment manual) │
  ├─────────────┼──────────────────────────┼───────────────────────┤
  │ Swiggy      │ Official MCP             │ HIGH (COD only)       │
  ├─────────────┼──────────────────────────┼───────────────────────┤
  │ Zepto       │ Unofficial only          │ LOW                   │
  ├─────────────┼──────────────────────────┼───────────────────────┤
  │ Blinkit     │ Unofficial only          │ LOW                   │
  ├─────────────┼──────────────────────────┼───────────────────────┤
  │ Uber        │ REST API                 │ MEDIUM                │
  ├─────────────┼──────────────────────────┼───────────────────────┤
  │ Ola         │ REST API (restricted)    │ LOW-MEDIUM            │
  ├─────────────┼──────────────────────────┼───────────────────────┤
  │ IRCTC       │ Restricted (partnership) │ LOW                   │
  └─────────────┴──────────────────────────┴───────────────────────┘

  E. Creative / Content

  ┌─────────────────────────┬────────────────┬────────────────────────────────┐
  │       Integration       │ API Available? │             Notes              │
  ├─────────────────────────┼────────────────┼────────────────────────────────┤
  │ YouTube search          │ Free API       │ Read-only, useful for research │
  ├─────────────────────────┼────────────────┼────────────────────────────────┤
  │ Web search (Tavily)     │ SDK + MCP      │ Foundation skill               │
  ├─────────────────────────┼────────────────┼────────────────────────────────┤
  │ Reddit                  │ Free API       │ Research/sentiment             │
  ├─────────────────────────┼────────────────┼────────────────────────────────┤
  │ Image gen (Flux/DALL-E) │ APIs available │ Differentiation play           │
  └─────────────────────────┴────────────────┴────────────────────────────────┘

  F. Infra / Dev Tools

  ┌───────────────┬────────────────────┬─────────────────────────────────────┐
  │  Integration  │   API Available?   │                Notes                │
  ├───────────────┼────────────────────┼─────────────────────────────────────┤
  │ GitHub        │ Via Composio       │ Low priority for v1 target audience │
  ├───────────────┼────────────────────┼─────────────────────────────────────┤
  │ Supabase      │ You already use it │ Internal only                       │
  ├───────────────┼────────────────────┼─────────────────────────────────────┤
  │ MongoDB/MySQL │ Via Composio       │ Enterprise, not v1                  │
  └───────────────┴────────────────────┴─────────────────────────────────────┘

  ---
  3. MUST-HAVE vs Differentiator vs Low-Priority

  MUST-HAVE for v1 (without these, the product feels incomplete)

  ┌─────┬─────────────────────────────┬───────────────────────────────────────────────┬───────────────────────────┐
  │  #  │            Skill            │                      Why                      │          Source           │
  ├─────┼─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────┤
  │ 1   │ WhatsApp send/receive       │ Core action — already built                   │ Direct adapter (existing) │
  ├─────┼─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────┤
  │ 2   │ Voice calls                 │ Core action — already built                   │ Direct adapter (existing) │
  ├─────┼─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────┤
  │ 3   │ Web search                  │ Every assistant needs this                    │ Tavily (langchain-tavily) │
  ├─────┼─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────┤
  │ 4   │ Contact lookup              │ Prerequisite for #1 and #2                    │ Direct adapter (existing) │
  ├─────┼─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────┤
  │ 5   │ Conversation/chat reply     │ Basic interaction                             │ Direct adapter (existing) │
  ├─────┼─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────┤
  │ 6   │ Google Calendar read/create │ "What's on my schedule?" is top-3 daily query │ MCP (Google Workspace)    │
  ├─────┼─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────┤
  │ 7   │ Gmail search/send           │ "Check my email" / "Send an email to..."      │ MCP (Google Workspace)    │
  ├─────┼─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────┤
  │ 8   │ Memory (long-term)          │ "Remember that I prefer..." — personalization │ mem0ai                    │
  └─────┴─────────────────────────────┴───────────────────────────────────────────────┴───────────────────────────┘

  DIFFERENTIATORS (what Bhindi/OpenClaw can't do)

  ┌─────┬────────────────────────┬───────────────────────────────────────────────────────────────────────────────────────┐
  │  #  │         Skill          │                                    Why it's a moat                                    │
  ├─────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
  │ 9   │ Zomato ordering        │ Real-world commerce via official MCP — nobody else does this with consent gates       │
  ├─────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
  │ 10  │ Swiggy ordering        │ Same — India daily-use, official MCP                                                  │
  ├─────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
  │ 11  │ Zerodha/stock trading  │ High-value prosumer. "Buy 10 shares of Reliance" with approval gate = killer feature  │
  ├─────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
  │ 12  │ Razorpay payment links │ "Send a payment request to X" — real financial action                                 │
  ├─────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
  │ 13  │ Bidirectional handling │ Inbound WhatsApp/call → auto-draft response → consent → reply. Nobody does this well. │
  └─────┴────────────────────────┴───────────────────────────────────────────────────────────────────────────────────────┘

  LOW-PRIORITY (post-launch, nice-to-have)

  ┌─────┬──────────────────────┬─────────────────────────────────────────────────────────────────────────┐
  │  #  │        Skill         │                                Why wait                                 │
  ├─────┼──────────────────────┼─────────────────────────────────────────────────────────────────────────┤
  │ 14  │ Slack                │ Good for prosumers but not daily-life sticky for India early adopters   │
  ├─────┼──────────────────────┼─────────────────────────────────────────────────────────────────────────┤
  │ 15  │ Notion/Linear/Trello │ Productivity tools — add via Composio when marketplace adapter is ready │
  ├─────┼──────────────────────┼─────────────────────────────────────────────────────────────────────────┤
  │ 16  │ GitHub               │ Dev-focused, narrow audience                                            │
  ├─────┼──────────────────────┼─────────────────────────────────────────────────────────────────────────┤
  │ 17  │ Image generation     │ Cool but not task execution                                             │
  ├─────┼──────────────────────┼─────────────────────────────────────────────────────────────────────────┤
  │ 18  │ Uber/Ola             │ APIs restricted or flaky                                                │
  ├─────┼──────────────────────┼─────────────────────────────────────────────────────────────────────────┤
  │ 19  │ IRCTC                │ Requires partnership — explore post-launch                              │
  └─────┴──────────────────────┴─────────────────────────────────────────────────────────────────────────┘

  ---
  4. 10-15 Additional High-Stickiness Integrations (Especially India-Relevant)

  #: 1
  Integration: Zerodha Kite (trading)
  Category: Finance
  Why High Stickiness: "Buy/sell stocks" with consent gate. Official MCP exists. Rs 500/mo API. India's #1 broker.
  Feasibility: HIGH
  ────────────────────────────────────────
  #: 2
  Integration: Razorpay (payments)
  Category: Finance
  Why High Stickiness: Official MCP. Create payment links, send invoices, process refunds. Partnered with NPCI for agentic UPI.
  Feasibility: HIGH
  ────────────────────────────────────────
  #: 3
  Integration: Zomato (food)
  Category: Commerce
  Why High Stickiness: Official MCP. Order food via chat. Daily use.
  Feasibility: HIGH
  ────────────────────────────────────────
  #: 4
  Integration: Swiggy (food + grocery)
  Category: Commerce
  Why High Stickiness: Official MCP (Jan 2026). Food + Instamart + Dineout.
  Feasibility: HIGH
  ────────────────────────────────────────
  #: 5
  Integration: Angel One SmartAPI (trading)
  Category: Finance
  Why High Stickiness: Free API. Alternative broker for users not on Zerodha.
  Feasibility: HIGH
  ────────────────────────────────────────
  #: 6
  Integration: Groww (investing)
  Category: Finance
  Why High Stickiness: Rs 499/mo API. Stocks + MF. Popular with younger Indians.
  Feasibility: HIGH
  ────────────────────────────────────────
  #: 7
  Integration: Google Maps / Places
  Category: Daily life
  Why High Stickiness: "Find the nearest pharmacy" / "Directions to airport". Free API.
  Feasibility: HIGH
  ────────────────────────────────────────
  #: 8
  Integration: Expense tracking (custom)
  Category: Finance
  Why High Stickiness: "I spent 500 on dinner" → auto-log to a personal ledger. Build on top of Supabase. No external API needed.
  Feasibility: HIGH
  ────────────────────────────────────────
  #: 9
  Integration: Weather (AccuWeather/OpenWeather)
  Category: Daily life
  Why High Stickiness: "Will it rain today?" — simple, high-frequency, free API.
  Feasibility: VERY HIGH
  ────────────────────────────────────────
  #: 10
  Integration: DigiLocker (govt docs)
  Category: Utility
  Why High Stickiness: KYC verification, document retrieval. Via authorized providers.
  Feasibility: MEDIUM
  ────────────────────────────────────────
  #: 11
  Integration: ONDC (open commerce)
  Category: Commerce
  Why High Stickiness: Decentralized commerce protocol. Huge India potential but significant integration effort.
  Feasibility: LOW (v2)
  ────────────────────────────────────────
  #: 12
  Integration: Dhan (trading)
  Category: Finance
  Why High Stickiness: Free API, <50ms latency. Modern alternative to Zerodha.
  Feasibility: HIGH
  ────────────────────────────────────────
  #: 13
  Integration: Google Flights
  Category: Travel
  Why High Stickiness: "Find cheapest flights to Goa next weekend" — high intent, free API.
  Feasibility: MEDIUM
  ────────────────────────────────────────
  #: 14
  Integration: Cashfree (payments)
  Category: Finance
  Why High Stickiness: MCP server available. Alternative to Razorpay.
  Feasibility: MEDIUM
  ────────────────────────────────────────
  #: 15
  Integration: YouTube search/summarize
  Category: Research
  Why High Stickiness: "Summarize this video" — daily use, free API.
  Feasibility: MEDIUM

  ---
  5. Slack Deep Evaluation

  Can we search/read/send messages? Yes, all three:

  ┌──────────────────────┬────────┬────────────────────────────────────────────────────────────────────────────────────────┐
  │      Operation       │ Works? │                                       Constraint                                       │
  ├──────────────────────┼────────┼────────────────────────────────────────────────────────────────────────────────────────┤
  │ Search messages      │ Yes    │ Requires user token (not bot token). search:read scope.                                │
  ├──────────────────────┼────────┼────────────────────────────────────────────────────────────────────────────────────────┤
  │ Read channel history │ Yes    │ Bot token works. But non-Marketplace apps get 1 req/min, 15 msgs max after March 2026. │
  ├──────────────────────┼────────┼────────────────────────────────────────────────────────────────────────────────────────┤
  │ Send messages        │ Yes    │ Bot token, chat:write, 1 msg/sec/channel limit.                                        │
  ├──────────────────────┼────────┼────────────────────────────────────────────────────────────────────────────────────────┤
  │ Reply to threads     │ Yes    │ chat.postMessage with thread_ts.                                                       │
  ├──────────────────────┼────────┼────────────────────────────────────────────────────────────────────────────────────────┤
  │ Reactions            │ Yes    │ reactions:write scope.                                                                 │
  └──────────────────────┴────────┴────────────────────────────────────────────────────────────────────────────────────────┘

  Lock-in workflows beyond info retrieval:

  Workflow: Slack Connect (cross-org channels)
  Lock-in Level: VERY HIGH
  Why: External business relationships anchored in Slack. Migration = losing partner communication history.
  ────────────────────────────────────────
  Workflow: Canvas/docs management
  Lock-in Level: HIGH
  Why: Knowledge base locked in Slack. Create/edit/export canvases via API.
  ────────────────────────────────────────
  Workflow: Channel + User Group structure
  Lock-in Level: HIGH
  Why: Org hierarchy encoded in channels. An agent that auto-manages team channels creates dependency.
  ────────────────────────────────────────
  Workflow: Custom workflow steps
  Lock-in Level: HIGH
  Why: Automated business processes tied to Slack via Bolt framework.
  ────────────────────────────────────────
  Workflow: Threaded conversations as CRM
  Lock-in Level: MEDIUM
  Why: Using threads + reactions as lightweight ticket tracking.
  ────────────────────────────────────────
  Workflow: Scheduled messages
  Lock-in Level: MEDIUM
  Why: Recurring engagement patterns.
  ────────────────────────────────────────
  Workflow: Huddles/Calls
  Lock-in Level: LOW
  Why: Minimal API — can start/end calls but no recording/transcript access.

  Verdict on Slack for v1: It's a P1, not P0. Your early adopters are prosumers in India — they use WhatsApp 10x more than Slack. Add
  Slack via Composio (89 bot-compatible actions) or the official Slack MCP server (Feb 2026) as a fast-follow after launch.

  ---
  6. Comparison with OpenClaw

  Dimension: Architecture
  Trybo: Agent-of-Agents OS (dynamic graph routing)
  OpenClaw: Local-first personal assistant (static config routing)
  Bhindi: Proprietary agent platform (dynamic prompting)
  ────────────────────────────────────────
  Dimension: Routing
  Trybo: GID → semantic routing + confidence-based FAST_PATH/PLANNER/CLARIFY
  OpenClaw: Static YAML bindings (channel/user/chatType)
  Bhindi: Dynamic agent switching (proprietary)
  ────────────────────────────────────────
  Dimension: HITL / Consent
  Trybo: Mandatory consent gates with approval artifacts
  OpenClaw: None built-in (agent runs unchecked — caused inbox deletion incident)
  Bhindi: None mentioned
  ────────────────────────────────────────
  Dimension: Voice / Calls
  Trybo: Yes (core feature)
  OpenClaw: No
  Bhindi: No
  ────────────────────────────────────────
  Dimension: WhatsApp execution
  Trybo: Yes (send + receive + inbound handling)
  OpenClaw: WhatsApp channel (receive only, via community plugin)
  Bhindi: No execution
  ────────────────────────────────────────
  Dimension: India commerce
  Trybo: Zomato, Swiggy, Zerodha, Razorpay (via MCP)
  OpenClaw: Not India-focused
  Bhindi: Groww, CoinGecko only
  ────────────────────────────────────────
  Dimension: Security
  Trybo: JWT + RLS + consent gates
  OpenClaw: Critical security crisis — RCE vulns, 20% of plugin store was malicious
  Bhindi: OAuth-based, private beta
  ────────────────────────────────────────
  Dimension: Bidirectional
  Trybo: Yes (inbound events → draft → consent → reply)
  OpenClaw: Inbound only (no outbound orchestration engine)
  Bhindi: Outbound only
  ────────────────────────────────────────
  Dimension: Memory
  Trybo: mem0 (semantic, user-scoped, Supabase-backed)
  OpenClaw: Local file memory + optional vector DB
  Bhindi: Not disclosed
  ────────────────────────────────────────
  Dimension: Checkpointing
  Trybo: Supabase Postgres (survives app disconnect)
  OpenClaw: Local SQLite
  Bhindi: Not disclosed
  ────────────────────────────────────────
  Dimension: Multi-channel
  Trybo: WhatsApp + Voice + Chat + Email + Slack
  OpenClaw: 12+ messaging channels
  Bhindi: Text-only web/app

  Trybo's unique position: The only platform combining voice + WhatsApp execution + India commerce MCP + mandatory consent gates +
  bidirectional handling. OpenClaw has breadth but no depth (and massive security issues). Bhindi has breadth but no real-world
  execution (no calls, no WhatsApp delivery, no consent).

  ---
  Recommended v1 Skill List (Final)

  Launch set (P0 — 8 skills):
  1. whatsapp_send — Direct (existing)
  2. call_place — Direct (existing)
  3. contact_lookup — Direct (existing)
  4. conversation_reply — Direct (existing)
  5. web_search — Tavily MCP/LangChain
  6. gmail_search + gmail_send — Google Workspace MCP
  7. calendar_read + calendar_create — Google Workspace MCP
  8. memory_recall — mem0ai

  Fast-follow (P1 — weeks after launch):
  9. zomato_order — Zomato MCP
  10. swiggy_order — Swiggy MCP
  11. stock_trade — Zerodha Kite API or Angel One SmartAPI
  12. payment_link_create — Razorpay MCP

  P2 (month after launch):
  13. slack_read + slack_send — Slack MCP / Composio
  14. expense_log — Custom (Supabase)
  15. weather_check — OpenWeather API


  How information flows between nodes today (shared state)                                                                            
                                                                                                                                      
  LangGraph's TaskContext (shared state) is the communication bus. Every node reads from it, every node writes to it:                 

  Node A (contact_lookup) → writes result to state.task_results["task_1"]
      ↓
  Node B (gmail_send) → reads state.task_results["task_1"] to get email address
      ↓
  Node C (whatsapp_send) → reads state.task_results["task_1"] to get phone number

  The planner declares depends_on in the PlanSpec. The executor builds edges from those dependencies. By the time Node B runs, Node
  A's result is already in shared state. No callback needed — it's a DAG, not a conversation.

  ---
  Three scenarios and how each is handled

  Scenario 1: Node needs output from a prior node

  Solution: depends_on + shared state. Already solved by graph topology.

  gmail_search (task_1) → stock_trade (task_2, depends_on: [task_1])
                           ↑ reads task_1 result from TaskContext

  Scenario 2: Node discovers mid-execution it needs something unexpected

  Example: Composio's Slack agent is posting a message but realizes it needs the user's Slack handle, which wasn't in the original
  input.

  Solution: LangGraph's Command API — a node can dynamically reroute the graph:

  def slack_send_node(state):
      if "slack_handle" not in state["task_results"]:
          # Route to a lookup node, then come back
          return Command(
              update={"needed": "slack_handle"},
              goto="contact_lookup"  # dynamic reroute
          )
      # ... proceed with send

  This is intra-graph routing, not A2A protocol. The graph handles it.

  Scenario 3: External agent genuinely needs a multi-turn conversation

  Example: A marketplace agent asks "Which of these 3 vendors do you want?" — it needs a back-and-forth.

  Solution: This is what agent nodes (ReAct sub-graphs) are for. The sub-graph has its own LLM loop and tool access. It can reason,
  make multiple tool calls, and resolve ambiguity internally before returning a single result to the parent graph.

  Parent Graph
    └─ Agent Node (ReAct sub-graph)
         ├─ LLM: "I need to compare 3 vendors"
         ├─ Tool call: web_search("vendor A pricing")
         ├─ Tool call: web_search("vendor B pricing")
         ├─ Tool call: web_search("vendor C pricing")
         ├─ LLM: "Vendor B is cheapest"
         └─ Returns: { "recommendation": "Vendor B", "reason": "..." }

  If the sub-graph needs info from the user (not another agent), it uses interrupt() — which surfaces the question to the mobile app
  and waits.

  ---
  What about true A2A (agent-to-agent protocol)?

  The A2A protocol (Google, Linux Foundation) is designed for cross-organization agent communication — Agent X at Company A negotiates
   with Agent Y at Company B. Think: "My travel agent talks to the airline's booking agent."

  You don't need this for v1. Here's why:

  ┌──────────────────────────────┬─────────────────────────────────────────────────────┐
  │         A2A Use Case         │                   Your v1 Reality                   │
  ├──────────────────────────────┼─────────────────────────────────────────────────────┤
  │ Agent discovery across orgs  │ You have a fixed registry of 11 skills              │
  ├──────────────────────────────┼─────────────────────────────────────────────────────┤
  │ Negotiation between agents   │ Your graph is plan-driven, not negotiation-driven   │
  ├──────────────────────────────┼─────────────────────────────────────────────────────┤
  │ Cross-org auth/trust         │ All your adapters use your own credentials          │
  ├──────────────────────────────┼─────────────────────────────────────────────────────┤
  │ Long-running agent dialogues │ Your nodes are request/response, not conversational │
  └──────────────────────────────┴─────────────────────────────────────────────────────┘

  When you will need it (post-v1): When you add marketplace agents that are truly autonomous — e.g., an external "insurance comparison
   agent" that needs to ask your orchestrator clarifying questions. At that point, the marketplace adapter would implement the A2A
  tasks/sendSubscribe pattern for streaming back-and-forth. But that's a Phase 4+ concern.

  ---
  Summary

  ┌───────────────────────────────────────────────────┬───────────────────────────────────┬────────────────────────────┐
  │                      Pattern                      │             Mechanism             │         Build now?         │
  ├───────────────────────────────────────────────────┼───────────────────────────────────┼────────────────────────────┤
  │ Node reads prior node's output                    │ Shared state via depends_on       │ Yes (core of executor)     │
  ├───────────────────────────────────────────────────┼───────────────────────────────────┼────────────────────────────┤
  │ Node needs unexpected data                        │ Command(goto=...) dynamic reroute │ Yes (built into LangGraph) │
  ├───────────────────────────────────────────────────┼───────────────────────────────────┼────────────────────────────┤
  │ Complex subtask with multi-step reasoning         │ Agent node (ReAct sub-graph)      │ Yes (LangGraph prebuilt)   │
  ├───────────────────────────────────────────────────┼───────────────────────────────────┼────────────────────────────┤
  │ Need user input mid-graph                         │ interrupt() / consent gate        │ Yes (Phase 2)              │
  ├───────────────────────────────────────────────────┼───────────────────────────────────┼────────────────────────────┤
  │ External agent wants a dialogue with orchestrator │ A2A protocol                      │ No (post-v1)               │
  └───────────────────────────────────────────────────┴───────────────────────────────────┴────────────────────────────┘

  The short answer: LangGraph's shared state + Command API + sub-graphs + interrupt() covers every v1 scenario. No custom inter-agent
  protocol needed. A2A is a post-launch concern for when you open the marketplace to truly autonomous external agents.

  ---
  Ready to start Phase 1?

✻ Cogitated for 44s

❯ what about interruptable flows? asking user permissions before execution/asking more info from the user? have you planned this      
  using  LangGraph's TaskContext/interrupts or anything others?                                                                      
                                                                                                                                      
⏺ Yes — this is one of the most critical parts of the design. Let me lay out the complete interrupt strategy. There are four distinct 
  interrupt types, all built on LangGraph's interrupt() but with different UX and resumption patterns.
                                                                                                                                      
  ---                                                                                                                                 
  The Four Interrupt Types                                                                                                            
                                                                                                                                      
  1. Consent Gate — "Should I do this?"                                                                                               

  Side-effect node needs approval before executing.

  Graph running → hits whatsapp_send node → node sees requires_consent=true
    → interrupt({ type: "consent", artifact: { ... } })
    → graph PAUSES → state saved to Postgres checkpoint
    → mobile app receives consent artifact via Supabase realtime
    → user sees card: [Draft message] [To: Rahul] [Edit | Approve | Deny]
    → user taps Approve
    → backend calls graph.ainvoke(Command(resume={"decision": "APPROVED"}))
    → graph RESUMES → executes whatsapp_send → continues to next node

  2. Clarification — "I need more info from you"

  GID or Planner can't proceed without missing information.

  User: "Send a message to Raj"
    → GID detects: missing entity (which Raj? what message?)
    → interrupt({ type: "clarification", questions: [
         { field: "recipient", prompt: "Which Raj?", options: ["Raj Patel", "Raj Kumar"] },
         { field: "content", prompt: "What should the message say?" }
       ]})
    → graph PAUSES
    → mobile app shows clarification card with input fields
    → user replies: "Raj Patel, tell him I'm running late"
    → Command(resume={"recipient": "Raj Patel", "content": "I'm running late"})
    → graph RESUMES → planner now has complete info → builds plan

  3. Edit-and-Approve — "Here's a draft, refine it"

  The system generates content that the user might want to edit before sending.

  Graph running → agent node drafts an email → hits consent gate
    → interrupt({ type: "edit_approve", artifact: {
         action: "gmail_send",
         draft: { subject: "Meeting tomorrow", body: "Hi team, ..." },
         recipient: "team@company.com",
         editable_fields: ["subject", "body"]
       }})
    → mobile app shows EDITABLE draft card
    → user modifies the body, taps Send
    → Command(resume={
         "decision": "APPROVED",
         "edited_content": { "body": "Hi team, updated text..." }
       })
    → graph RESUMES with edited content → executes gmail_send with user's version

  4. Batch Approval — "Approve all at once"

  Multiple side-effect actions for different recipients.

  User: "Send meeting invite to all 5 team members"
    → Planner creates 5 calendar_create tasks
    → Executor collects all consent artifacts BEFORE executing any
    → interrupt({ type: "batch_consent", artifacts: [
         { recipient: "Alice", action: "calendar_create", details: {...} },
         { recipient: "Bob", action: "calendar_create", details: {...} },
         ... (5 items)
       ]})
    → mobile app shows SINGLE batch card: "Send to 5 people" [Review All | Approve All | Deny]
    → user taps Approve All (or deselects specific recipients)
    → Command(resume={ "approved": ["alice","bob","carol"], "denied": ["dave"] })
    → graph RESUMES → executes only approved actions in parallel

  ---
  How It Works Under the Hood

  ┌─────────────────────────────────────────────────────────────────┐
  │                        LangGraph Executor                       │
  │                                                                 │
  │  ┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌───────┐ │
  │  │ Tool Node │───→│ Consent  │───→│ Side-Effect  │───→│ Done  │ │
  │  │ (lookup)  │    │  Gate    │    │   Node       │    │       │ │
  │  └──────────┘    └────┬─────┘    └──────────────┘    └───────┘ │
  │                       │                                         │
  │                  interrupt()                                    │
  │                       │                                         │
  │              ┌────────▼────────┐                                │
  │              │  Postgres       │  ← full state checkpointed    │
  │              │  Checkpoint     │                                │
  │              └────────┬────────┘                                │
  └───────────────────────┼─────────────────────────────────────────┘
                          │
              ┌───────────▼───────────┐
              │  Supabase Realtime    │  ← push notification to app
              │  (consent_artifacts)  │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐
              │  Mobile App           │
              │  - Shows card         │
              │  - User acts          │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐
              │  POST /consent/{id}   │  ← REST endpoint
              │  /respond             │
              └───────────┬───────────┘
                          │
                  Command(resume={...})
                          │
                    Graph RESUMES

  Key implementation details:

  # Inside a consent-aware node wrapper
  async def consent_gate_wrapper(state: TaskContext, skill_id: str, input: dict):
      skill = registry.get(skill_id)

      if skill.requires_consent:
          artifact = build_consent_artifact(skill, input, state)

          # Persist artifact to Supabase for mobile app to display
          await save_consent_artifact(artifact, state.plan_id, state.user_id)

          # Push notification to mobile
          await notify_user(state.user_id, artifact)

          # PAUSE graph — state saved to Postgres checkpoint
          # This line blocks until Command(resume=...) is called
          decision = interrupt(artifact.to_dict())

          if decision["decision"] == "DENIED":
              return SkillResult(status="skipped", reason="user_denied")

          if decision["decision"] == "EDIT_REQUIRED":
              input = apply_edits(input, decision["edited_content"])

          # Fall through to execute with (possibly edited) input

      return await adapter.invoke(skill_id, input, state)

  ---
  Edge Cases We Need to Handle

  Scenario: User never responds (app closed, forgot)
  Solution: Timeout policy: configurable per risk level. Low=24h auto-expire. High=never auto-approve. Graph stays paused in
    checkpoint.
  ────────────────────────────────────────
  Scenario: User responds hours/days later
  Solution: Works automatically — checkpoint in Postgres survives indefinitely. Command(resume=...) picks up exactly where it left
  off.
  ────────────────────────────────────────
  Scenario: Multiple interrupts in one graph
  Solution: Sequential — LangGraph processes them in node execution order. Each interrupt pauses independently.
  ────────────────────────────────────────
  Scenario: Parallel nodes both need consent
  Solution: LangGraph's Send fan-out collects all interrupts in the same superstep. We batch them into a single batch approval card.
  ────────────────────────────────────────
  Scenario: User edits break the input schema
  Solution: Validate edited content against input_schema from registry before resuming. If invalid, re-surface with error.
  ────────────────────────────────────────
  Scenario: Consent denied mid-graph
  Solution: Check fallback_skill_id in the PlanSpec task. If exists, try fallback. If no fallback, mark task failed, continue rest of
    graph if possible.
  ────────────────────────────────────────
  Scenario: App disconnect during graph execution
  Solution: Checkpoint already saved. On reconnect, load checkpoint, check for pending interrupts, re-surface any unanswered consent
    cards.

  ---
  What We're Using vs Building

  Component: Pause/resume mechanism
  Using (OSS): LangGraph interrupt() + Command(resume=)
  Building (Custom): Nothing — it's built-in
  ────────────────────────────────────────
  Component: State persistence during pause
  Using (OSS): langgraph-checkpoint-postgres
  Building (Custom): Nothing — point at Supabase Postgres
  ────────────────────────────────────────
  Component: Consent artifact schema
  Using (OSS): —
  Building (Custom): Pydantic model (~40 lines)
  ────────────────────────────────────────
  Component: Consent artifact rendering
  Using (OSS): —
  Building (Custom): Mobile app card component (UI repo)
  ────────────────────────────────────────
  Component: Approval policy engine
  Using (OSS): —
  Building (Custom): should_auto_approve() function (~60 lines) — checks risk level, repeat action, same recipient
  ────────────────────────────────────────
  Component: REST endpoint for user response
  Using (OSS): —
  Building (Custom): POST /consent/{artifact_id}/respond → calls Command(resume=...) (~50 lines)
  ────────────────────────────────────────
  Component: Push notification to mobile
  Using (OSS): Existing Firebase service
  Building (Custom): Wire to consent artifact creation (~20 lines)
  ────────────────────────────────────────
  Component: Batch approval aggregation
  Using (OSS): —
  Building (Custom): Collect parallel interrupts into one card (~40 lines)
  ────────────────────────────────────────
  Component: Timeout/expiry
  Using (OSS): —
  Building (Custom): APScheduler job to expire old artifacts (~30 lines)

  Total custom code for interrupt system: ~240 lines. The heavy lifting (pause, checkpoint, resume, state management) is all
  LangGraph.

  ---
  This is fully planned. The consent system you already have in consent_request_store.py and consent_action_service.py maps directly
  to this — we're essentially upgrading it from a standalone flow to one that's embedded inside graph execution via interrupt().

  Ready to start Phase 1?

✻ Brewed for 58s                                                                                                                      
                                                                                           
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
❯ yes let's start, save our research and plan to memory first                                                                         
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  PR #185                                                                                                                             
                                                                                                                                      
