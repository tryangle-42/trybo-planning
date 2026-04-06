# Parallel Sprint — 3 Copy-Pastable Prompts -> Complete

3 terminals, 3 branches, simultaneous. Merge at end (order: A → B → C).

---

## PROMPT A -> Commute

```
Trybo FastAPI backend. Agentic orchestration engine — decomposes user goals into skill-based task graphs via LangGraph. Skills are registered in a skill registry, executed by adapters (Direct, MCP, Marketplace).

Foundation sprint already built:
- `app/services/skill_data/` — SkillDataStore (generic persistence, namespace isolation)
- `app/services/job_scheduler/` — SchedulerService + background worker
- `app/agentic_mvp/runtime/skills_notifications.py` — `notification.push.send` skill
- `app/agentic_mvp/runtime/contracts.py` — StructuredArtifact, ARTIFACT_TYPES, artifact on Receipt
- Empty domain skill files: `skills_maps.py`, `skills_calendar.py` (with exported lists MAPS_SKILLS, CALENDAR_SKILLS)
- `seed_registry.py` imports and merges all domain skill lists

Read existing skills in `seed_registry.py` and handlers in `adapters/direct.py` to understand the patterns. Follow them exactly.

Create branch `feature/commute-briefing-skills` from `feat/hero-usecases`.

COMMUTE OPTIMIZER — STAGE 1 (critical context):
Stage 1 is "manual automation": when user asks "when should I leave to reach X by 9 AM?", fire a burst of ~16 Google Maps queries across a time window (every 15 min, 6 AM–10 AM), compare estimated durations, pick the best departure slot. Uses Google's future-departure-time estimates — inaccurate but useful for relative comparison across slots.

NO background data collection. NO scheduled cron jobs for measurements. NO historical profiles. NO statistical modeling. Those are Stage 2/3, deferred.

SKILLS TO BUILD:

In `skills_maps.py` → MAPS_SKILLS:

1. `maps.directions.google` — Direct adapter. Query Google Maps Routes/Directions API. Accepts both departure_time=now and future timestamps. Returns duration_seconds, distance_meters, summary. Add GOOGLE_MAPS_API_KEY to config settings.

2. `maps.geocode.google` — Direct adapter. Resolve address to lat/lng via Geocoding API. Same API key.

3. `commute.optimize.departure` — Direct adapter. Core Stage 1 skill. Takes origin, destination, date, window_start/end (HH:MM), interval_minutes, optional target_arrival_time. Internally loops over the time window calling maps.directions.google for each slot. Returns: all slots with durations, best_departure (shortest), latest_safe_departure (if target set — latest where arrival ≤ target), peak_window (worst block). Produces StructuredArtifact with artifact_type "commute_alert" (add to ARTIFACT_TYPES). ~16 API calls per invocation — parallelize with asyncio.gather. Support dry_run with realistic mock data (short at 6 AM, peak 8-8:30 AM, declining after 9).

In `skills_calendar.py` → CALENDAR_SKILLS:

4. `weather.forecast.openmeteo` — Direct adapter. Open-Meteo API (free, no key). Takes lat/lng, forecast_days. Returns current conditions + daily summary. Parse WMO weather codes to human text.

5. `briefing.synthesize` — Direct adapter. Takes weather, calendar_events, commute_data, email_digest (all optional, from parallel fan-out via $memory refs). Combines into morning briefing. Produces StructuredArtifact "briefing_card".

6. `briefing.preference.save` — Direct adapter, side_effect=True. Saves preferences to SkillDataStore (namespace="briefing", record_type="preference"). Registers daily scheduled job via SchedulerService (job_type="graph_execution" at user's briefing_time).

Registry + config:
7. Verify seed_registry.py picks up new skills from MAPS_SKILLS and CALENDAR_SKILLS.
8. Add GOOGLE_MAPS_API_KEY to app/config/__init__.py.

CONSTRAINTS:
- Follow existing code patterns exactly — read direct.py and seed_registry.py first
- All persistent data through SkillDataStore — no new tables, no new migrations
- Commute optimizer is stateless one-shot — does NOT use SchedulerService (briefing does)
- Do NOT build data collection, historical profiles, or measurement cron jobs — Stage 2/3
- Do NOT touch: skills_feeds.py, skills_booking.py, skills_ecommerce.py
- Do NOT create new adapter classes — add handlers to existing DirectAdapter
- Every handler must support dry_run mode
- Add any new artifact types to ARTIFACT_TYPES in contracts.py
```

---

## PROMPT B - Digest

```
Trybo FastAPI backend. Agentic orchestration engine — decomposes user goals into skill-based task graphs via LangGraph. Skills are registered in a skill registry, executed by adapters (Direct, MCP, Marketplace).

Foundation sprint already built:
- `app/services/skill_data/` — SkillDataStore (generic persistence, namespace isolation)
- `app/services/job_scheduler/` — SchedulerService + background worker
- `app/agentic_mvp/runtime/skills_notifications.py` — `notification.push.send`
- `app/agentic_mvp/runtime/contracts.py` — StructuredArtifact, ARTIFACT_TYPES, artifact on Receipt
- Empty `skills_feeds.py` with exported FEEDS_SKILLS list
- `seed_registry.py` imports and merges all domain skill lists
- Existing Composio/Marketplace adapter for external service integrations

Read existing skills in `seed_registry.py` and handlers in `adapters/direct.py` to understand the patterns. Follow them exactly. Also read the marketplace adapter to understand how Composio skills work.

Create branch `feature/digest-skills` from `feat/hero-usecases`.

USE CASE: Daily personalized digest — fetch tech news from 6 sources in parallel, deduplicate, rank by relevance to user's role/topics, categorize, summarize, extract product ideas using user's product context from mem0. Write insights to Notion, product ideas to Linear/GitHub (with consent). Push notification when ready.

SKILLS TO BUILD:

Content fetcher skills in `skills_feeds.py` → FEEDS_SKILLS (all Direct, no side effects):

1. `content.feed.fetch` — Fetch + parse RSS/Atom feeds via feedparser + httpx. Takes list of feed URLs, since_hours filter. Returns entries with title/link/published/summary. Handle failures gracefully (partial results + error list). Add feedparser to requirements.in.

2. `content.hn.top` — HN Algolia API (hn.algolia.com/api/v1/search_by_date). Free, no auth. Filter by points threshold, time range, optional keyword query.

3. `content.arxiv.search` — ArXiv API (export.arxiv.org/api/query). Free, no auth. Search by categories (cs.AI, cs.CL, cs.LG) + keywords. Respect 1 req/3s rate limit.

4. `content.producthunt.today` — Product Hunt API or Tavily fallback. Today's top launches. Add optional PRODUCT_HUNT_API_TOKEN to config.

5. `content.reddit.top` — Reddit JSON API (reddit.com/r/{sub}/top.json). No auth for public read. Set proper User-Agent. Takes list of subreddits, merges + sorts by score.

Synthesis + preference skills in `skills_feeds.py` → FEEDS_SKILLS:

6. `digest.synthesize` — Direct adapter. Takes all content arrays from parallel branches + user preferences + mem0 context. Deduplicates, LLM-ranks by relevance, categorizes into sections (must_read, research, launches, industry, community), generates per-item summaries, extracts product ideas. Produces StructuredArtifact "digest_card". LLM-heavy (~6 calls) — use existing LLM calling pattern.

7. `digest.preference.save` — Direct, side_effect=True. Saves to SkillDataStore (namespace="digest", record_type="preference") with role, topics, sources, output_targets, digest_time. Registers daily scheduled job via SchedulerService.

External write adapters in `skills_feeds.py` → FEEDS_SKILLS (all Marketplace via Composio, side_effect=True, approval_required=True):

8. `knowledge.note.write.notion` — Create/append Notion page. Uses existing Composio/Marketplace adapter. No new handler code needed.

9. `tasks.issue.create.linear` — Create Linear issue. Per-item consent. Uses Composio.

10. `tasks.issue.create.github` — Create GitHub Issue. Alternative to Linear. Uses Composio.

Registry + config:
11. Verify seed_registry.py picks up FEEDS_SKILLS.
12. Add PRODUCT_HUNT_API_TOKEN to config (optional).

CONSTRAINTS:
- Follow existing code patterns exactly — read direct.py, seed_registry.py, and marketplace adapter first
- All persistent data through SkillDataStore — no new tables, no new migrations
- Marketplace skills (Notion, Linear, GitHub) use EXISTING Composio adapter — no new adapter code
- Do NOT touch: skills_maps.py, skills_calendar.py, skills_booking.py, skills_ecommerce.py
- Do NOT create new adapter classes — add handlers to existing DirectAdapter
- Every Direct handler must support dry_run mode
- Add any new artifact types to ARTIFACT_TYPES in contracts.py
```

---

## PROMPT C - trip-purchase

```
Trybo FastAPI backend. Agentic orchestration engine — decomposes user goals into skill-based task graphs via LangGraph. Skills are registered in a skill registry, executed by adapters (Direct, MCP, Marketplace).

Foundation sprint already built:
- `app/services/skill_data/` — SkillDataStore (generic persistence, namespace isolation)
- `app/services/job_scheduler/` — SchedulerService + background worker
- `app/agentic_mvp/runtime/skills_notifications.py` — `notification.push.send`
- `app/agentic_mvp/runtime/contracts.py` — StructuredArtifact, ARTIFACT_TYPES, artifact on Receipt
- Parallel fan-out in compiler_graph.py (feature-gated)
- Empty `skills_booking.py` and `skills_ecommerce.py` with exported lists BOOKING_SKILLS, ECOMMERCE_SKILLS
- `seed_registry.py` imports and merges all domain skill lists

Read existing skills in `seed_registry.py` and handlers in `adapters/direct.py` to understand the patterns. Follow them exactly.

Create branch `feature/trip-purchase-skills` from `feat/hero-usecases`.

SKILLS TO BUILD:

Trip Planner skills in `skills_booking.py` → BOOKING_SKILLS:

1. `booking.flights.search` — Direct adapter. Amadeus Self-Service API (Python SDK `amadeus`). Takes origin/destination IATA codes, dates, cabin class. Returns flights with prices, times, stops. Add AMADEUS_API_KEY, AMADEUS_API_SECRET, AMADEUS_ENV (default "test") to config. Fallback: Tavily web search. Support dry_run.

2. `booking.hotels.search` — Direct adapter. Serpapi Google Hotels API. Takes location, dates, budget filters. Returns hotels with prices, ratings, amenities. Add SERPAPI_API_KEY to config. Fallback: Tavily. Support dry_run.

3. `trip.itinerary.synthesize` — Direct adapter. Takes flights, hotel, weather, attractions, logistics data from parallel fan-out branches. LLM generates day-wise itinerary + budget breakdown. Produces StructuredArtifact "itinerary_card".

4. `trip.plan.save` — Direct, side_effect=True, approval_required=True. Saves finalized trip to SkillDataStore (namespace="travel", record_type="trip_plan"). Optionally registers reminder scheduled job.

Purchase Advisor skills in `skills_ecommerce.py` → ECOMMERCE_SKILLS:

5. `web.scrape.structured` — Direct adapter. Extract structured data from a URL. Primary: Tavily Extract. Fallback: httpx fetch + LLM extraction. Takes URL + extract_fields hints.

6. `ecommerce.price.compare` — Direct adapter. Compare product prices across platforms (Amazon.in, Flipkart, Croma). For each platform: Tavily search → extract pricing → layer bank offers based on user's cards → compute effective prices. LLM matches bank-platform partnerships. Returns comparisons array + recommendation + StructuredArtifact "comparison_table". Extract to separate module if handler exceeds ~100 lines.

7. `user.financial.save` — Direct, side_effect=True. Saves user's bank cards/wallets to SkillDataStore (namespace="user_profile", record_type="financial_instruments"). Also store as semantic fact in mem0 if available.

Registry + config:
8. Verify seed_registry.py picks up BOOKING_SKILLS and ECOMMERCE_SKILLS.
9. Add config: AMADEUS_API_KEY, AMADEUS_API_SECRET, AMADEUS_ENV, SERPAPI_API_KEY.

CONSTRAINTS:
- Follow existing code patterns exactly — read direct.py and seed_registry.py first
- All persistent data through SkillDataStore — no new tables, no new migrations
- Do NOT touch: skills_maps.py, skills_calendar.py, skills_feeds.py
- Do NOT create new adapter classes — add handlers to existing DirectAdapter
- Every handler must support dry_run mode
- Add any new artifact types to ARTIFACT_TYPES in contracts.py
- Extract complex handler logic to separate modules if >100 lines
```

---

## Integration

Merge order: A → B → C. Conflicts only in `seed_registry.py` (imports), `direct.py` (handler branches), `config/__init__.py` (env vars), `requirements.in` (packages) — all additive, trivial merges.

After merge: verify all skills register via `load_seed_registry()`, smoke test one plan per use case.
