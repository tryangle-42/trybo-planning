Parallel Workstreams                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                    
  Workstream 1: Memory Layer (mem0) — feature/mem0-memory                                                                                                                                                                                             
                                                                                                                                                                                                                                                      
  Conflict risk: LOW — entirely new files + one integration point

  What to build now:
  - app/orchestration/memory/mem0_service.py — MemoryService class wrapping mem0ai with Supabase/pgvector backend
  - app/orchestration/memory/config.py — mem0 config (vector store provider, embedding model, collection name)
  - FastAPI lifespan integration (init mem0 at startup)
  - Two methods that the orchestration layer will call:
  async def retrieve_context(user_id: str, query: str) -> list[dict]
  async def store_facts(user_id: str, messages: list[dict]) -> None

  Why it's safe in parallel: This is a standalone service with no imports from agentic_mvp/. After his branch merges, you wire it into ExecutionContext and the planner prompt — a 10-line integration.

  Prep work you can do today:
  1. Add mem0ai to requirements.in
  2. Set up pgvector in your Supabase instance (enable extension, create memories table)
  3. Build and test the MemoryService against your local Supabase
  4. Write a simple test endpoint to verify retrieval/storage works

  ---
  Workstream 2: GID (Global Intent Detector) — feature/gid-semantic-router

  Conflict risk: LOW — entirely new module, sits upstream of the planner

  What to build now:
  - app/orchestration/gid/router.py — wraps semantic-router with Trybo's route definitions
  - app/orchestration/gid/routes.py — route definitions with sample utterances per intent category
  - app/orchestration/gid/service.py — GIDService returning {routing_decision, intent_type, entities, confidence_score}
  - Entity extraction helper (LLM call, only triggered on PLANNER/CLARIFY paths)

  The three routing outcomes:
  FAST_PATH  → single skill, no side-effect, confidence > 0.85
  PLANNER    → multi-step, side-effects, or ambiguous
  CLARIFY    → missing required entity

  Why it's safe in parallel: GID sits upstream of the planner. It's a new entry point that decides whether to invoke the full planner pipeline or shortcut directly to a single skill execution. After his branch merges, you insert GID between the
  chat endpoint and compiler_graph.py.

  Prep work you can do today:
  1. Add semantic-router to requirements.in
  2. Define route utterances for your 11 v1 skills (10-20 examples per route)
  3. Build and benchmark the router — verify sub-10ms routing latency
  4. Test entity extraction for common patterns ("Send X to Y", "Call Z about W")

  ---
  Workstream 3: Missing P0 Skills — feature/v1-skill-definitions

  Conflict risk: LOW — new files that register alongside his existing skills

  What to build now:
  - Skill definition files (same pattern as his seed_registry.py) for:
    - channel.voice.call_place — thin wrapper around existing call_service.py
    - contacts.lookup — thin wrapper around existing contacts_service.py
    - channel.chat.reply — thin wrapper around existing conversation_service.py
  - Direct adapter handlers for each (same pattern as his _whatsapp_send in direct.py)
  - Input/output JSON schemas for each skill

  Why it's safe in parallel: You're writing new SkillDefinition entries and new handler functions. These get added to seed_skill_definitions() after merge — it's an append, not a modification.

  Don't build yet (wait for his adapter refactor or yours):
  - Gmail/Calendar (needs MCP adapter generalization)
  - Slack search (needs MCP adapter generalization)

  Prep work you can do today:
  1. Write the SkillDefinition Pydantic objects with full JSON schemas for all 3 direct skills
  2. Write the handler functions that call existing services
  3. Test them standalone against your local environment

  ---
  Workstream 4: MCP Adapter Generalization — feature/mcp-multi-server

  Conflict risk: MEDIUM — replaces his mcp_tavily.py, but yours is a new file

  What to build now:
  - app/orchestration/adapters/mcp_adapter.py — generic MCP adapter using langchain-mcp-adapters MultiServerMCPClient
  - MCP server configuration (which servers to connect to, transport type per server)
  - This single adapter handles ALL MCP skills (Tavily, Gmail, Calendar, Slack) rather than one file per MCP tool

  # Config-driven, not code-driven
  MCP_SERVERS = {
      "tavily": {"url": "https://mcp.tavily.com/mcp/...", "transport": "http"},
      "google-workspace": {"command": "npx", "args": ["-y", "@anthropic/google-workspace-mcp"], "transport": "stdio"},
      "slack": {"url": "https://mcp.slack.com/mcp", "transport": "http"},
  }

  Why it's medium risk: His branch has mcp_tavily.py (Tavily-specific). Your adapter replaces it with a generic one. But since yours is a new file with a new name, the merge is clean — you just swap the import in plan_builder.py and delete his
  Tavily-specific file.

  Prep work you can do today:
  1. Add langchain-mcp-adapters, mcp to requirements.in
  2. Test MultiServerMCPClient with Tavily's remote MCP URL
  3. Test with the official Slack MCP server
  4. Test with a Google Workspace MCP server (if you have OAuth set up)
  5. Verify tools are correctly converted to LangChain-compatible format

  ---
  Workstream 5: Composio Adapter — feature/composio-langgraph

  Conflict risk: MEDIUM — same pattern as Workstream 4, replaces his marketplace_composio.py

  What to build now:
  - app/orchestration/adapters/composio_adapter.py — uses composio-langgraph SDK instead of raw REST
  - OAuth connection management for Slack (and future apps)
  - This handles the authentication lifecycle Composio manages for you (token refresh, multi-app)

  Prep work you can do today:
  1. Add composio, composio-langgraph to requirements.in
  2. Set up a Composio account, create an auth config for Slack
  3. Test composio.tools.get(user_id=..., tools=["SLACK_CHAT_POST_MESSAGE"]) flow
  4. Verify it returns LangChain-compatible tools

  ---
  What to NOT start yet (depends on his P0 fixes)

  ┌─────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────┐
  │                    Work                     │                                          Why wait                                           │
  ├─────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Wiring mem0 into ExecutionContext / planner │ Needs his Postgres checkpointing to land first — mem0 context needs to survive restarts too │
  ├─────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Wiring GID into the chat endpoint           │ Needs his compiler_graph.py to stabilize after interrupt() refactor                         │
  ├─────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Parallel fan-out in plan execution          │ Needs his plan_builder.py to land — you'd be modifying the same file                        │
  ├─────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Supabase event persistence                  │ Needs his event_store.py to stabilize — you'd add write-through to his code                 │
  └─────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────┘

  ---
  Recommended Execution Order

  Week 1 (start now, in parallel):
    ├── You: Workstream 1 (mem0 service, standalone) -------> Doing this right now
    ├── You: Workstream 2 (GID, standalone)
    ├── You: Workstream 3 (skill definitions for call/contacts/reply)
    └── Co-worker: Fixes P0s (rebase, auth, checkpoint-postgres, interrupt())

  Week 2:
    ├── You: Workstream 4 (MCP multi-server adapter)
    ├── You: Workstream 5 (Composio SDK adapter)
    └── Co-worker: Fixes P1s (parallel fan-out, event persistence)

  After his branch merges:
    ├── Wire mem0 into ExecutionContext + planner prompt (10 lines)
    ├── Insert GID upstream of compiler_graph (20 lines)
    ├── Append new skill definitions to seed_registry (append-only)
    ├── Swap MCP adapter import in plan_builder (1 line change)
    └── Swap Composio adapter import in plan_builder (1 line change)

  The integration work after his merge is deliberately minimal — each of your workstreams produces a self-contained module with a clean interface, and the wiring is a few lines each.