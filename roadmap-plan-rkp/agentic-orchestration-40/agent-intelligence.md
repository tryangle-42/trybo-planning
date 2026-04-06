# Agent Intelligence & Decisioning — Baseline Definition

> Owner: RKP
> Module mapping: M8 (Agent Intelligence)
> Current completion: ~16% — Consent gates + channel escalation done. Everything else missing.

---

## What This Is

Makes the agent smarter over time. Four concerns:

1. **Feedback & Reinforcement** — collect execution outcomes, track what works, evolve prompts
2. **Decisioning Authority** — consent management, when to ask vs when to act
3. **Strategy Adaptation & Error Classification** — learn from failures, classify errors for better handling
4. **Plan Quality Scoring** — measure if plans are good and getting better

This module sits *above* the execution engine — it doesn't run plans, it evaluates and improves them.

## Current State

| Built | Not Built |
|-------|-----------|
| Consent gates (LangGraph `interrupt()`) | Feedback collection (no outcome tracking) |
| Channel escalation (WhatsApp → call on SLA breach) | Prompt evolution (planner prompt never improves) |
| | Error classification (errors are untyped) |
| | Plan quality scoring (no measurement) |
| | Autonomy levels (everything requires explicit approval) |
| | Strategy persistence (no learning from outcomes) |

---

## Baseline Definition

**The line:** Every plan execution outcome is logged with structured feedback signals. Errors are classified into actionable categories. Plan quality is scored (rule-based). Consent management handles risk-tiered approval. This data foundation exists so that prompt evolution and strategy adaptation can be built on top. Anything beyond this is refinement.

The reasoning: you can't evolve prompts without data. Step 1 (baseline) is collecting the data. Step 2 (refinement) is acting on it.

---

## Baseline Components

### 1. Feedback Collection

Every execution produces structured outcome data. This is the raw material for all intelligence.

**Implicit feedback (no user action required):**

| Signal | What it means | Captured from |
|--------|--------------|--------------|
| Plan completed successfully | Positive — the plan worked | `agent_runs.status = 'completed'` |
| Plan failed | Negative — something broke | `agent_runs.status = 'failed'` + error category |
| User started new request on same topic | Weak negative — plan was insufficient | Chat message analysis (same intent within 5 min) |
| User manually did what agent was supposed to | Strong negative — agent was useless | Hard to detect at baseline; defer |
| Consent edits (user modified draft before approving) | The delta is feedback — agent's draft was close but not right | Diff between original draft and user-edited version |
| Time-to-approval | Long wait = user was unsure | Timestamp diff: interrupt → resume |

**Explicit feedback (user action):**

| Signal | What it means | UI element |
|--------|--------------|-----------|
| Thumbs up on result | Positive | Post-execution rating |
| Thumbs down on result | Negative | Post-execution rating |
| "This was wrong" correction | Specific negative with context | Text input on result card |

**Storage:** A `feedback_signals` table (or records in `skill_data_store` with namespace `feedback`) that links `run_id` → signal type → value → timestamp. This is append-only log data.

**At baseline, we collect only. No automated action on feedback.** The data sits in the table. Weekly manual review identifies patterns. Prompt evolution (acting on feedback) is refinement.

### 2. Error Classification System

Every failure gets a structured category that determines how the system responds. This feeds into M5's retry logic and into the feedback loop.

**Error taxonomy:**

| Category | Meaning | Executor behavior | Example |
|----------|---------|------------------|---------|
| `PLANNING_FAILED` | LLM produced an invalid plan | Retry planning (max 3) | Unparseable JSON, missing skill_id |
| `SKILL_NOT_FOUND` | Plan references nonexistent skill | Remove node + cascade | Hallucinated skill_id |
| `VALIDATION_FAILED` | PlanSpec failed structural checks | Retry with error context | Cycle detected, schema mismatch |
| `ADAPTER_TIMEOUT` | External service didn't respond | Retry with backoff | Twilio timeout |
| `ADAPTER_ERROR` | External service returned error | Retry if 5xx, skip if 4xx | API rate limit, auth expired |
| `CONSENT_DENIED` | User explicitly denied | Respect decision, skip node | User said "no, don't call them" |
| `CONSENT_TIMEOUT` | User didn't respond in time | Mark as abandoned | Approval expired after 72h |
| `LLM_RATE_LIMITED` | LLM provider throttled | Retry with backoff | OpenAI 429 |
| `LLM_CONTEXT_OVERFLOW` | Prompt too large | Reduce context, retry | Too many skills + memories in prompt |
| `USER_FIXABLE` | Requires user action | Interrupt, ask user | OAuth expired, missing permission |

**Why this matters:** Without classification, every error is treated the same — either retry everything (wasteful) or fail everything (brittle). With classification, the executor makes smart decisions: retry transients, skip permanents, ask user for fixables.

### 3. Decisioning Authority Framework

**Consent management (already built, needs extension):**

The existing `interrupt()` consent gates work. Baseline extends them with risk-tiered approval:

| Risk Level | Action type | Approval behavior |
|------------|-----------|-------------------|
| `none` | Read-only, no side effects | Auto-approve. No interrupt. |
| `low` | Low-stakes write (add reminder, draft email) | Auto-approve if user has approved similar before. Otherwise ask. |
| `medium` | Meaningful side effect (send email, make call) | Always ask. Show draft for editing. |
| `high` | Irreversible or costly (payment, delete data) | Always ask + require explicit confirmation + audit log. |

Each skill in the registry is tagged with a risk level. The executor checks: `if risk_level > user's autonomy threshold → interrupt()`.

**At baseline, autonomy threshold is fixed at `low`** — everything above low-risk requires explicit approval. User-configurable autonomy levels (per-domain, per-action) are refinement.

**Multi-plan generation:** When confidence is low, the planner generates 2-3 alternative plans and presents them to the user: "I can do this two ways: (A) search online then call, or (B) call directly and ask. Which do you prefer?" At baseline, this is a planner prompt instruction ("when unsure, generate alternatives"), not a separate system. Structured multi-plan UI is refinement.

### 4. Plan Quality Scoring

Measure whether plans are good and getting better over time. Baseline uses rule-based scoring (no LLM-as-judge yet).

**Rule-based score (computed after every execution):**

| Metric | Weight | How it's measured |
|--------|--------|------------------|
| **Task completion rate** | 0.35 | completed_tasks / total_tasks |
| **No retries needed** | 0.20 | 1.0 if zero retries, 0.5 if 1-2, 0.0 if 3+ |
| **No user corrections** | 0.15 | 1.0 if no consent edits, 0.5 if minor edits, 0.0 if major edits |
| **Execution time vs estimate** | 0.15 | 1.0 if under estimate, scales down proportionally |
| **No clarifications needed** | 0.15 | 1.0 if planner had enough info, 0.0 if 2+ clarification rounds |

**Composite plan quality score:** Weighted sum → 0.0 to 1.0. Stored per run in `agent_runs` or a dedicated `plan_scores` table.

**What you do with it at baseline:** Track the score over time. If average plan quality drops below 0.6 for a week, flag for manual investigation. This is the signal that tells you "the planner is getting worse" — which triggers the refinement work (prompt evolution).

---

## What Counts as Refinement

| Refinement | Why deferred | Trigger to build |
|------------|-------------|-----------------|
| **Prompt evolution** (GEPA/OpenAI self-evolving pattern) | Needs feedback data first. Baseline collects data. | When you have 500+ execution outcomes to analyze patterns. |
| **LLM-as-judge plan quality** | Adds latency + cost per execution. Rule-based scoring sufficient at baseline. | When rule-based scoring fails to differentiate good from bad plans. |
| **Autonomy levels** (user-configurable per domain) | Fixed threshold works at baseline. | When users complain about too many approval prompts. |
| **Strategy persistence** ("calling Sharma at 10am works better") | Requires cross-request outcome tracking. | When episodic memory (M7) is mature. |
| **Proactive intelligence** (agent initiates actions unprompted) | High risk. Agent should be reactive until trust is established. | Post-beta, when user trust is proven. |
| **Preference learning** (infer from behavior, not just explicit statements) | Needs significant interaction volume. | 1000+ user interactions per user. |
| **Goal inference** ("meeting at 9 in Gurgaon" → check traffic) | Requires deep context understanding. | When memory layers 3-5 are wired. |
| **Explainability** ("why did you choose IndiGo?") | Nice UX, not functionally blocking. | When users ask "why did you do that?" frequently. |

---

## Dependencies

| Needs | From | Why |
|-------|------|-----|
| Execution outcomes (run status, task results, errors) | Execution Engine (M5, RKP) | Raw data for feedback signals |
| Error categories on all failures | Execution Engine (M5, RKP) | Classification feeds into feedback |
| Risk-level tags on skills | Skill Management (M2, AKN) | Consent tiers reference skill risk level |
| Consent UI (approval cards in chat) | Frontend (M9, KPR) | Users see and respond to approval requests |
| `skill_data_store` for feedback storage | Memory (M7, KPR) | Feedback signals stored as structured data |

## Effort Estimate

| Component | Effort |
|-----------|--------|
| Feedback signal collection (implicit + explicit) | 2-3 days |
| `feedback_signals` table + write hooks in M5 finalize | 1-2 days |
| Error classification taxonomy + integration with M5 retry | 2-3 days |
| Risk-tiered consent (extend existing interrupt system) | 2 days |
| Plan quality scoring (rule-based, per-run) | 2 days |
| Multi-plan generation (planner prompt instruction) | 1 day |

**Total: ~10-14 dev days to reach baseline.**

---

## Key Concepts Explained

**Why collect feedback before acting on it?**
Prompt evolution (automatically improving the planner's system prompt based on failure patterns) sounds great. But it requires enough data to identify patterns. With 50 executions, you can't tell if a failure pattern is systematic or random. With 500+, you can. Baseline builds the data pipeline. Refinement builds the intelligence on top.

**What is prompt evolution?**
Instead of manually tweaking the planner prompt when plans go wrong, an automated system: (1) samples recent execution traces, (2) identifies failure patterns ("planner keeps generating 7-step plans for simple intents"), (3) rewrites the system prompt to address the pattern, (4) A/B tests the new prompt vs the old one. This is the OpenAI Self-Evolving Agents pattern and DSPy's GEPA optimizer. It's powerful but needs data volume to work.

**Why risk-tiered consent instead of "ask for everything"?**
Asking for approval on every action (including read-only searches) creates "approval fatigue" — the user starts auto-approving without reading. Risk tiers mean: searches happen silently, email drafts need a glance, calls need explicit approval, payments need confirmation. The user's attention is directed to decisions that actually matter.
