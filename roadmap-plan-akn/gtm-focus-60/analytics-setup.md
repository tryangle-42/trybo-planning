# Analytics Setup — Baseline Definition

> Owner: AKN
> Module mapping: M5 (Execution Analytics), M2 (Skill Health Metrics), M1 (Observability)
> Scope: Dev/ops-facing only. Internal dashboards and metrics for operating the system.

---

## What This Is

Operational analytics that answer: "Is the system working well? Where is it failing? What's it costing us?" This is the metrics layer that lets the team operate the product — detect degradation, measure improvement, and prioritize engineering work based on data rather than gut feel.

Not user-facing analytics ("you made 15 calls this week"). Not business analytics ("DAU, retention"). Purely: can the team see how the agentic engine is performing?

## Current State

| What Exists | What's Missing |
|-------------|----------------|
| `agent_run_events` table with 24 event types | No aggregation — raw events only, no rollups |
| `bot_tasks` with SLA tracking columns | No success/failure rate computation |
| `agent_runs` with status tracking | No latency percentile computation |
| structlog request timing | No cost tracking per run or per skill |
| Feature flag for parallel fanout | No A/B comparison metrics |
| SSE events streamed to mobile | No event volume or throughput metrics |

---

## Baseline Definition

**The line:** The team can answer these 10 questions from a single query or dashboard page, updated within the last hour. Anything beyond this is refinement.

### The 10 Baseline Questions

| # | Question | Metric | Source |
|---|----------|--------|--------|
| 1 | What % of plans succeed end-to-end? | `plan_success_rate` = completed_runs / total_runs | `agent_runs.status` |
| 2 | Where do plans fail? | `failure_distribution` by error_category | `agent_run_events` + error categorization (from debug-logs baseline) |
| 3 | How fast are plans? | `plan_latency_p50`, `plan_latency_p95` (end-to-end, GID to finalize) | span data or `agent_run_events` timestamps |
| 4 | Which skills fail most? | `skill_failure_rate` per skill_id | `executor.run_node` spans |
| 5 | Which skills are slowest? | `skill_latency_p50`, `skill_latency_p95` per skill_id | `executor.run_node` spans |
| 6 | How much are LLM calls costing? | `llm_cost_per_run`, `llm_cost_daily` | `llm.call` spans (model + tokens → cost lookup) |
| 7 | How often do users hit consent? | `consent_rate` = consent_requests / total_runs | `consent_requests` table |
| 8 | What's the consent approval rate? | `consent_approval_rate` = approved / total_consent | `consent_requests.status` |
| 9 | How many runs per day/hour? | `run_volume` time series | `agent_runs.created_at` |
| 10 | What's the GID routing distribution? | `gid_distribution` = count by routing_decision (FAST_PATH / PLANNER / CLARIFY) | `gid.route` spans |

### Baseline Components

#### 1. Metrics aggregation table

An `analytics_hourly` table stores pre-computed metric rollups — one row per (hour, metric, dimension). Each row captures count, sum, min, max, and percentiles (p50, p95). The dimension field lets you slice the same metric different ways: "global plan success rate" vs "success rate for `google.calendar` specifically."

**Why hourly batch, not real-time?** Real-time streaming (sub-minute updates) requires infrastructure like Kafka or Redis Streams. For a 3-5 person team, running a cron job every hour that aggregates from raw events is simpler, cheaper, and gives you 90% of the operational insight. Upgrade to real-time when the system handles enough traffic that an hour-old metric is too stale.

#### 2. Aggregation job

A scheduled job (Python script or Postgres function) that runs every hour:

**Plan metrics:**
- Count `agent_runs` by status (completed, failed, cancelled) per hour
- Compute success rate = completed / total
- Compute latency percentiles from run start→end timestamps

**Skill metrics:**
- From `executor.run_node` spans: group by skill_id
- Compute per-skill: invocation count, success rate, latency p50/p95
- Flag skills with >10% failure rate or p95 > 5s

**LLM cost metrics:**
- From `llm.call` spans: model + prompt_tokens + completion_tokens
- Apply cost lookup table (model → cost_per_1k_input_tokens, cost_per_1k_output_tokens)
- Aggregate: cost per run, cost per hour, cost per day

**Consent metrics:**
- From `consent_requests` table: count by status per hour
- Approval rate, average time-to-decision

**GID metrics:**
- From `gid.route` spans: count by routing_decision per hour

#### 3. LLM cost lookup table

A static config file mapping each model to its input/output token cost (per 1M tokens). The aggregation job multiplies token counts from span data by these rates to compute cost per run and daily totals.

Updated manually when new models are added or pricing changes. Refinement would auto-fetch pricing from provider APIs.

#### 4. Analytics API endpoints

Dev-only (admin-gated):

```
GET /analytics/summary?period=24h
```
Returns: plan success rate, total runs, top failing skills, total LLM cost, consent rate — for the given period.

```
GET /analytics/skills?period=7d&sort=failure_rate
```
Returns: per-skill breakdown — invocation count, success rate, latency p50/p95, cost.

```
GET /analytics/cost?period=30d&group_by=model
```
Returns: LLM cost breakdown by model, by day.

```
GET /analytics/trends?metric=plan_success_rate&period=30d&granularity=daily
```
Returns: time series for any metric.

#### 5. Health check summary

A single endpoint that returns system health at a glance:

```
GET /analytics/health
```

Returns a single JSON object: overall status (healthy/degraded/unhealthy), plan success rate, average latency, list of failing skills, LLM cost today, consent approval rate, total runs, and reasons for degradation if any.

**Thresholds for status classification:**
| Metric | Healthy | Degraded | Unhealthy |
|--------|---------|----------|-----------|
| Plan success rate | > 85% | 70-85% | < 70% |
| Any skill failure rate | < 10% | 10-25% | > 25% |
| Plan latency p95 | < 10s | 10-20s | > 20s |

---

## What Counts as Refinement (explicitly deferred)

| Refinement | Why deferred |
|------------|-------------|
| Real-time streaming metrics (sub-minute) | Hourly batch covers operational needs for a small team |
| Grafana/Datadog dashboards | API endpoints + CLI queries are sufficient at baseline |
| Automated alerting (Slack/PagerDuty on degradation) | Requires SLO definitions. Baseline is data availability. |
| User-level analytics (per-user usage, engagement) | That's product analytics, not ops analytics |
| Business metrics (DAU, retention, conversion) | Separate concern entirely |
| A/B testing metrics infrastructure | Need AgentRank first (M4). |
| Cost forecasting / budget alerts | Refinement on top of cost tracking |
| Anomaly detection (ML-based) | Statistical thresholds are sufficient at baseline |
| Custom metric definitions (user-defined) | Predefined 10 metrics cover operational needs |
| Historical data archival / retention | Data accumulates for now. Clean up post-scale. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| Span data from debug-logs baseline | Debug Logs (AKN) | Skill latency, LLM costs, GID routing sourced from spans |
| `agent_runs` + `agent_run_events` tables | Execution Engine (M5) | Primary data source for plan metrics |
| `consent_requests` table | Security (M11) | Consent metrics |
| Admin auth middleware | Security (M11) | Analytics endpoints gated |
| Scheduled job infrastructure | Execution Engine (M5) | Hourly aggregation job |

## Effort Estimate

- `analytics_hourly` table + aggregation job: 2-3 days
- LLM cost lookup + cost computation: 1 day
- Analytics API endpoints (5 endpoints): 2 days
- Health check endpoint with thresholds: 1 day

**Total: ~6-8 dev days to reach baseline.** Depends on debug-logs baseline being done first (needs span data).
