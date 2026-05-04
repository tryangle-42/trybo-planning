# Prompt Decomposition & Task Planning — L2 Agent Runtime Brain (C3)

> Module mapping: C3 (Prompt Decomposition & Task Planning) — supersedes the older M3 baseline framing in `task-decomposition.md`
> Status: canonical reference for the planning control plane
> Current completion: ~25% (binary intent router + single big-LLM planner + validation done; cascaded planner, capability-aware classifier, plan templates, smart clarify, recovery hints all missing)

---

## What This Is

The brain layer that turns a free-form user message into a **validated, executable PlanSpec** — using the **cheapest planner that can correctly handle the request**.

Owns five responsibilities, each a distinct service:

1. **Request Classifier** — produces a small JSON decision (intent_type, complexity, ambiguity, risk, plan_shape, needs_big_planner). Cheap.
2. **Planning Strategy Router** — deterministic dispatch from the classifier's decision to one of N planner tiers / templates.
3. **Smart Clarify** — planning-aware: distinguishes *critical missing* (block) from *derivable missing* (plan a lookup step) from *optional missing* (skip).
4. **Tiered Planner** — T0 deterministic templates → T1 small LLM → T2 big LLM. The right tier is chosen, never escalated by default.
5. **Plan Validator** — schema, skill existence, late-binding refs, recovery-hint completeness. Last gate before M7 takes over.

The planner **never sees the full skill catalog**. It receives a compressed **Capability Catalog** view from Skill Management.

---

## What This Module Does NOT Own

- Skill registry / resolution / health / governance → M2 (Skill Management). C3 only consumes the lightweight Capability Catalog.
- Runtime retry / fallback / replan-on-failure / blocked dependents → **M7 (Execution Intelligence)**. C3 produces the *hypothesis* and *recovery hints*; M7 owns *reality*.
- Memory writes / episodic recall pipeline → C5 (Memory).
- Approval gateway, consent UI rendering → M11 (Security) + mobile UI.

This module exposes:

```text
classify(user_message, context) -> RequestDecision
route(decision)                  -> PlannerStrategy
clarify(decision, missing)       -> ClarifyAction
plan(strategy, decision, ctx)    -> PlanSpec
validate(plan_spec)              -> ValidatedPlan | ValidationError
```

---

## Why "One Big LLM Planner For Everything" Is Not Enough

A single big-model planner works at MVP scale. It collapses the moment any of these become real:

| Real-world condition | Why one-big-planner fails |
|---|---|
| **Mobile latency budget** | Big-LLM planner adds 1–3s on the critical path even for "send WhatsApp to Rahul"; user perceives the assistant as sluggish |
| **Cost per turn** | Every "open Instagram" pays full GPT-class tokens; bills scale with usage, not value |
| **Repeatability** | The same simple request produces slightly different plans each time → mobile UX feels non-deterministic |
| **Confidence in decomposition** | The planner *might* decompose well; we have no way to assert it for known-shape requests |
| **Over-planning simple actions** | Six-node DAGs for "set a timer" — wasted reasoning, longer time-to-first-action |
| **Clarification thrash** | Planner asks redundant questions when the missing data could be derived from existing skills (contact lookup, calendar query) |
| **No skill-shape awareness** | Planner has to re-discover from scratch that "send a message" is a single-skill workflow vs. a fan-out |
| **Replan duplication** | Without recovery hints in the PlanSpec, M7 has to guess what "this step failed" means → retries that don't help |

The cascaded design addresses each as a first-class concern. The planning effort scales with the request's actual complexity.

---

## Current State

| What Exists | What's Missing |
|---|---|
| `app/services/intent_router.py` — 2-class (chat / task) classifier with rule fast-path + LLM fallback | No multi-class decision (intent_type, complexity, ambiguity, risk, plan_shape) |
| `app/agentic_mvp/runtime/langgraph/llm_planner.py` — single big-LLM planner, runs for every "task" | No tiered planner (T0 deterministic / T1 small / T2 big) |
| `compiler_graph.py` outer lifecycle: `ingress_normalize → clarify → plan → validate → execute → finalize` | No `request_classifier` and no `planning_strategy_router` nodes between ingress and clarify |
| `clarify` node generates clarification questions via LLM | `clarify` is not planning-aware — no critical/derivable/optional taxonomy |
| `validate` node checks schema + ref integrity | No recovery-hint completeness check; M7 has to guess on failure |
| `seed_registry.py` — full skill catalog | No compressed Capability Catalog surface (planning-facing read view) |
| `PlanSpec` carries `tasks`, `args`, `$memory.*` refs | No `success_criteria`, `expected_outputs.empty_result_policy`, `failure_hints` |
| `intent_router` already routes between "skip planner" and "use planner" | Routing is binary; no `direct_skill_plan` / `template_plan` / `transactional_planner` / `research_planner` branches |
| Plan templates: zero | Templates for top mobile actions (`send_message`, `make_call`, `set_reminder`, `search_web`, `summarize_file`, `book_calendar_event`, `open_app_and_do_action`, `compare_options`) |

---

## Target Definition (Advanced Baseline)

> **The line:** for any user message, the system spends *planning effort proportional to task complexity*. A simple action like "open Instagram" routes to a deterministic template in <50ms. A complex multi-domain workflow routes to the big planner with recovery hints. The planner is **adaptive**, not generic. The clarifier asks **only when the plan cannot otherwise be correctly formed**. Execution intelligence (M7) receives a plan annotated with success criteria and failure hints so its recovery decisions are guided, not guessed.

Three tests of the baseline:

1. **No big-LLM for slam-dunk tasks** — single-action requests must hit a deterministic template path; big-LLM call rate <40% of total turns.
2. **No clarification thrash** — clarify only asks when missing data is *critical* and *not derivable*; baseline rate of "redundant" clarifications = 0 on the eval set.
3. **No M7 guessing** — every PlanSpec carries `success_criteria` + per-task `failure_hints`; M7's recovery branch coverage on the eval set = 100%.

---

## Architecture Overview

```text
                          User Message
                                ↓
                       ingress_normalize
                                ↓
                      ┌──────────────────────┐
                      │ deterministic_precheck│  (regex / @mentions / app intents / OS deeplinks)
                      └──────────────────────┘
                                ↓
                      ┌──────────────────────┐
                      │ Request Classifier   │  ← Capability Catalog (read-only)
                      │ (cheap: rules +      │
                      │  embeddings + small  │
                      │  LLM)                │
                      └──────────────────────┘
                                ↓
                       RequestDecision (JSON)
                                ↓
                      ┌──────────────────────┐
                      │ Planning Strategy    │  ← deterministic if/else
                      │ Router (no LLM)      │
                      └──────────────────────┘
                                ↓
                      ┌──────────────────────┐
                      │ Smart Clarify        │  ← classifies missing fields
                      │ (planning-aware)     │      critical / derivable / optional
                      └──────────────────────┘
                                ↓
                      ┌──────────────────────┐
                      │ Tiered Planner       │
                      │  T0: template fill   │  (deterministic, <50ms)
                      │  T1: small LLM       │  (2–4 step plans)
                      │  T2: big LLM         │  (DAG, multi-domain, high-risk)
                      └──────────────────────┘
                                ↓
                            PlanSpec
                       (+ success_criteria,
                          failure_hints,
                          expected_outputs)
                                ↓
                      ┌──────────────────────┐
                      │ Plan Validator       │
                      │ (schema, refs,       │
                      │  hint completeness)  │
                      └──────────────────────┘
                                ↓
                      ┌──────────────────────┐
                      │ Plan Critic          │  (only for high-risk / T2 plans)
                      │ (small LLM, optional)│
                      └──────────────────────┘
                                ↓
                  Execute via M7 Execution Intelligence
                  (uses success_criteria + failure_hints
                   to decide continue / retry / block /
                   replan / ask user)
```

---

## Core Components

### 1. Deterministic Precheck

The cheapest possible filter. Runs **before** any model call.

| Match | Action |
|---|---|
| `@mention` of a known contact / app | route to single-skill template, skip classifier |
| OS deeplink intent (`tel:`, `mailto:`, `whatsapp://`) | route to direct_skill_plan, skip classifier |
| Pure greeting / acknowledgement | route to chat reply, skip planner |
| Empty / whitespace | drop |

Why: half of mobile messages are deterministically classifiable. Spending an LLM call on "ok thanks" is malpractice.

### 2. Request Classifier

Produces a small typed JSON decision. Does **not** generate a plan.

```json
{
  "intent_type":      "action | question | research | conversation",
  "capability_path":  "communication.messaging.whatsapp",
  "complexity":       "simple | medium | complex",
  "ambiguity":        "low | medium | high",
  "risk":             "L0 | L1 | L2 | L3 | L4",
  "plan_shape":       "single_step | linear | dag | batch",
  "needs_big_planner": false,
  "confidence":       0.91
}
```

**Implementation tiers (each can short-circuit):**

| Layer | Used when | Latency |
|---|---|---|
| Rules | precheck-style hard matches survive into the classifier | <5ms |
| Embedding + capability index | request maps cleanly to one capability path | <50ms |
| Small LLM (3B–7B class) | ambiguous intent, multi-capability hint | 200–400ms |
| Big LLM | confidence < 0.6 from small LLM AND deferred isn't safe | only as last resort |

**Why this matters:** Today every `task` request pays the big-LLM tax just to discover *what kind of request it is*. The classifier's only job is to label, not solve. Labeling is much cheaper than solving.

### 3. Planning Strategy Router

A pure function. No LLM. No I/O.

```python
def route(decision: RequestDecision) -> PlannerStrategy:
    if decision.intent_type == "conversation":
        return CHAT_REPLY
    if decision.complexity == "simple" and decision.plan_shape == "single_step":
        return DIRECT_SKILL_TEMPLATE
    if decision.ambiguity == "high":
        return CLARIFY_FIRST
    if decision.intent_type == "research":
        return RESEARCH_PLANNER          # T2 + reflection
    if decision.plan_shape == "batch":
        return BATCH_ACTION_PLANNER      # T2 + fan-out hint
    if decision.risk in {"L3", "L4"}:
        return TRANSACTIONAL_PLANNER     # T2 + critic + mandatory approval
    if decision.plan_shape == "dag":
        return DAG_PLANNER               # T2
    return DEFAULT_LINEAR_PLANNER        # T1
```

**Why deterministic:** routing decisions need to be inspectable, testable, and identical across runs. An LLM router would re-introduce the very non-determinism we're trying to eliminate.

### 4. Smart Clarify (Planning-Aware)

Replaces the generic "ask if anything is unclear" loop. Classifies each missing field:

| Class | Definition | Action |
|---|---|---|
| **Critical** | Plan cannot be safely constructed without it (e.g. amount in a payment) | **Block** — ask user, wait, then re-enter classifier with the answer |
| **Derivable** | Can be filled by an upstream skill (phone number → `contact.lookup`) | **Plan it** — add the lookup step, do not bother the user |
| **Optional** | Improves output but plan is valid without it | **Skip** — the planner can decide whether to use a default |
| **Ambiguous** | Multiple valid interpretations affect the plan shape (e.g. "book a flight to Delhi" — when?) | **Ask** if the alternatives diverge structurally; otherwise pick the safe default |

**One-line principle:** *clarify to make planning possible, not to perfect the plan.*

**Why this matters:** today's clarify asks too much (annoying) or too little (wasted execution cycles when the plan turns out wrong). The taxonomy + planning-awareness collapses both failure modes.

### 5. Tiered Planner

Three planners, one is chosen per request.

#### Tier 0 — Deterministic Template

For obvious one-step actions. No LLM call.

```yaml
template: send_whatsapp_message
slots:
  recipient:    derive_from(message, type=contact)
  body:         derive_from(message, type=quoted_text | trailing_text)
plan:
  - skill: contact.lookup
    args: { name: $slot.recipient }
  - skill: channel.whatsapp.send
    args: { phone: $memory.contact.lookup.phone, body: $slot.body }
```

**Coverage target:** the top ~12 mobile actions:

- `send_message` (WhatsApp / SMS / Email)
- `make_call`
- `set_reminder`
- `search_web`
- `summarize_file`
- `book_calendar_event`
- `open_app_and_do_action`
- `compare_options`
- `get_directions`
- `play_media`
- `note_capture`
- `translate_text`

If a template fits, slot-fill and execute. If slots can't be filled deterministically, fall through to T1 with the template as a hint.

#### Tier 1 — Small LLM Planner

For simple multi-step plans (2–4 tasks). 3B–7B class model. Receives:
- the request
- the strategy decision
- the relevant capability slice (not full registry)
- the closest matching template (as a hint, not a constraint)

Outputs a PlanSpec. No critic.

#### Tier 2 — Big LLM Planner

Used **only** for:
- Multi-domain workflows
- DAGs with > 4 nodes
- L3 / L4 risk tasks
- Research / synthesis
- Plans involving multiple user approvals
- Long-running batched workflows

Adds a `plan_critic` pass (small LLM) for L3+/research. Optional reflection loop after each task during execution (lives in M7, but Tier 2 plans flag which steps to reflect on).

**Why three tiers and not two:** T0 is the only way to reach human-perceptible "instant" on mobile (<300ms total). T1 saves cost without sacrificing quality on the long tail of simple multi-step. T2 earns its cost on the requests where reasoning quality is the bottleneck.

### 6. Plan Templates (the unlock)

Most mobile turns are minor variations of a small set of patterns. Templates make those patterns:

- **Repeatable** — the same input produces the same plan
- **Cheap** — no LLM required to plan
- **Inspectable** — the plan shape is a file, not an emergent property of a model
- **Safe** — every template is reviewed once; unsafe shapes never ship

Storage: YAML/JSON in repo, hot-reloadable. Each template declares:

```yaml
id: book_calendar_event
trigger:
  capability_path: scheduling.calendar
  intent_keywords: [book, schedule, set up]
  required_slots: [title, when]
  optional_slots: [attendees, location, duration]
slot_extraction:
  when: nlp_datetime_parser   # tier 0 helper, not LLM
plan:
  - skill: calendar.create_event
    args: ...
success_criteria:
  - event id returned
  - calendar shows event at requested time
failure_hints:
  if_overlap: ask_user_to_choose
  if_permission_missing: ask_for_calendar_grant
```

### 7. Skill Awareness — The Capability Catalog

C3 must be skill-aware, but **only via a compressed planning-facing view** owned by Skill Management.

| C3 needs | C3 must NOT touch |
|---|---|
| Capability hierarchy (tree) | Per-provider health |
| Per-leaf skill list | Resolver ranking weights |
| Side-effecting flag | Specific provider credentials |
| Approval-likely flag | Runtime allocation decisions |
| Single-vs-multi-skill hint | Schema-fit details (resolver's job) |
| Batch / template availability | Version pinning |

**Catalog shape (read-only contract from M2):**

```yaml
capability_catalog:
  communication.messaging.whatsapp:
    skills: [channel.whatsapp.send]
    side_effecting: true
    approval_likely: true
    has_template: true
    batch_pattern: true
  scheduling.calendar:
    skills: [calendar.create_event, calendar.search_events]
    side_effecting: true
    approval_likely: true
    has_template: true
  information.search:
    skills: [web.search, file.search]
    side_effecting: false
    approval_likely: false
    has_template: true
```

**Boundary rule:**
- **Classifier / Router** consume the *catalog* (capability-level view).
- **Planner** at execution-prep time consumes the *Resolver* (skill-level, governed, healthy candidates).

This keeps C3 from being coupled to skill churn while still letting it reason about what's possible.

### 8. PlanSpec Enrichment for M7

The current `PlanSpec` tells M7 *what to run*. The advanced PlanSpec also tells M7 *what success looks like* and *what to do when things go sideways*.

```yaml
PlanSpec:
  tasks:
    - node_id: search
      skill: flights.search
      args: { ... }
      expected_outputs:
        must_have: [results]
        empty_result_policy: ask_or_relax_constraints
      failure_hints:
        if_provider_failure: retry_with_backoff
        if_no_results: ask_user_to_relax
        if_timeout: fallback_skill(flights.search.alt)

    - node_id: pick
      skill: user.choose
      args: { options: $memory.search.results }
      requires_user_input: true

    - node_id: book
      skill: flights.book
      args: { flight_id: $memory.pick.choice }
      risk: L3
      requires_approval: true
      idempotency_key: $memory.pick.choice
      failure_hints:
        if_payment_declined: ask_user_to_change_card
        if_seat_taken: replan_from(search)

  success_criteria:
    - booking_confirmation_id present
    - confirmation email/SMS sent
```

**Why this matters:** today M7 has to *guess* what "search returned 0 results" means. With hints, the recovery branch is declared at plan time. Execution Intelligence remains in M7 — but its decisions are guided, not improvised.

### 9. Plan Critic (only when needed)

A tiny "is this plan reasonable?" LLM pass. Runs **only** for:
- T2 plans
- Risk ≥ L3
- Research strategies

Outputs: pass / replan / refine. Never runs for T0 / T1. Cost is bounded.

### 10. Mobile-Specific Constraints

C3 is server-side, but the mobile UX is the constraint. Specifically:

- **Time-to-first-event** — the user must see *something* (classifier decision banner, capability picked, "asking calendar…") within 200ms. The classifier's decision is the first event we stream.
- **Streaming all the way down** — every node emits SSE events; the mobile chat thread renders step rows as they happen.
- **Cancel must be cheap** — `DELETE /agent-runs/{id}` releases planner / clarify state. C3 nodes must be re-entrant after cancel.
- **Resume across network drops** — the LangGraph Postgres checkpointer already covers this; C3 must not hold non-serializable state.
- **Battery / data** — bias hard toward T0 templates so the device isn't waiting on multi-second LLM calls behind a flaky network.

---

## Execution Flow (concrete)

### Example A — Simple action (T0 path)

```text
User: "Send WhatsApp to Rahul saying I'm running late"

1. deterministic_precheck   → fits send_message template trigger
2. classifier               → SKIPPED (precheck matched)
3. router                   → DIRECT_SKILL_TEMPLATE (send_message)
4. clarify                  → recipient="Rahul" (derivable via contact.lookup),
                              body="I'm running late" (filled),
                              no critical missing → no question asked
5. T0 template              → fills slots, builds 2-task PlanSpec
6. validate                 → ok
7. execute (M7)             → contact.lookup → channel.whatsapp.send

Total LLM calls in C3: 0
Time-to-first-action:    ~80ms (classifier skipped)
```

### Example B — Ambiguous transactional (T2 path)

```text
User: "Send money to Rahul"

1. deterministic_precheck   → no match
2. classifier               → intent=action, capability=payments.send,
                              complexity=medium, ambiguity=high
                              (missing amount), risk=L3,
                              plan_shape=linear, needs_big_planner=true
3. router                   → CLARIFY_FIRST (ambiguity=high) →
                              if survives, TRANSACTIONAL_PLANNER
4. smart_clarify            → missing: amount=critical → ASK
                              missing: account=derivable → SKIP
                              → asks user: "How much do you want to send?"
   (turn pauses; resumes when user replies "₹500")

5. classifier (re-run)      → ambiguity=low now
6. router                   → TRANSACTIONAL_PLANNER (T2 + critic + approval)
7. T2 planner               → builds PlanSpec with success_criteria,
                              failure_hints, idempotency_key
8. plan_critic              → pass
9. validate                 → ok
10. execute (M7)            → account.lookup → payment.send (approval gate)

Total LLM calls in C3: 1 (classifier) + 1 (T2 planner) + 1 (critic)
                       — but only because the request actually warrants it
```

### Example C — Research (T2 + reflection)

```text
User: "Research best CRMs and make a comparison"

1. deterministic_precheck   → no match
2. classifier               → intent=research, complexity=complex,
                              ambiguity=low, risk=L0, plan_shape=dag,
                              needs_big_planner=true
3. router                   → RESEARCH_PLANNER
4. clarify                  → no critical missing → skip
5. T2 planner               → DAG: web.search × N → doc.summarize × N →
                              doc.compare → final.compose
6. critic                   → pass
7. validate                 → ok
8. execute (M7)             → with per-task reflection between search batches
```

---

## What Counts as Refinement (deferred)

| Refinement | Why deferred |
|---|---|
| Self-consistency planning (sample N plans, vote) | Critic + tiering recover most of the benefit at lower cost |
| Per-template A/B at the slot-extraction layer | Slot extractors evolve once usage data is in |
| Cross-capability conflict detection (e.g. payment + travel) | Rare in mobile; M7 catches mid-execution |
| LLM-rewritten templates from production traces | Requires telemetry maturity from M5 |
| Personalized router weights per user | Cohort-level weights are good enough at first |
| On-device classifier (privacy / offline) | Server is fine until offline becomes a real product requirement |

---

## Dependencies

| Needs | From | Why |
|---|---|---|
| Capability Catalog read endpoint | M2 (Skill Management) | Classifier + router skill-awareness |
| Resolver | M2 | Tiered planners ask only when binding skills to tasks |
| Postgres checkpointer (already in place) | LangGraph runtime | Pause / resume across clarify and approvals |
| Small LLM endpoint (3B–7B) | Infra | T1 planner + classifier fallback |
| Big LLM endpoint | Existing | T2 planner + critic |
| Embedding model | Infra | Classifier embedding layer; reuse M2's |
| Telemetry hooks | M5 (Observability) | Eval set tracking + tier-mix dashboards |
| Approval gateway | M11 (Security) | Used at execution time for L3/L4 plans |

## Effort Estimate

| Component | Effort |
|---|---|
| Multi-class request classifier (rules + embeddings + small LLM) | 2–3 days |
| Capability Catalog read endpoint (in M2) + C3 client | 1 day |
| Strategy router (deterministic) + tier dispatch | 1 day |
| Smart clarify (critical / derivable / optional taxonomy) | 2 days |
| T0 template engine + first 6 templates (send_message, make_call, set_reminder, search_web, summarize_file, book_calendar_event) | 3 days |
| T1 small-LLM planner wiring + capability slicing | 2 days |
| PlanSpec enrichment (success_criteria, failure_hints, expected_outputs) + validator updates | 2 days |
| Plan critic for T2 / L3+ | 1 day |
| Eval harness (decision-shape eval set, tier-mix targets, M7 hint coverage) | 2 days |

**Total: ~16–18 dev days to reach the advanced baseline.**

---

## Key Concepts Explained

**Why a classifier separate from the planner?**
A planner that also classifies pays the planning cost just to discover it should have been cheap. Splitting the steps lets us spend tokens on the request's actual difficulty, not on its label.

**Why deterministic routing?**
An LLM router would re-introduce the non-determinism we're moving away from. Deterministic routing is inspectable, unit-testable, and identical across runs — which is exactly what mobile UX needs.

**Why three planner tiers and not two (or five)?**
Two tiers (template / LLM) misses the "simple multi-step" middle, where a small LLM is enough but a template doesn't fit. Five tiers becomes a tuning nightmare with no clear escalation rules. Three is the smallest set that covers the latency/cost/quality space well.

**Why templates instead of "let the LLM be fast"?**
Even the fastest LLM is slower and more variable than a YAML lookup + slot fill. Templates also give us *certainty* — the same request produces the same plan, every time. That's the only way mobile UX feels trustworthy for repeated workflows like sending a daily message.

**Why is clarify "planning-aware"?**
A generic clarify either over-asks (annoying) or under-asks (wasteful execution). Planning-aware clarify knows that a missing phone number is derivable, but a missing payment amount is not — so it only interrupts when interruption is truly needed.

**Why doesn't C3 own retries / replan-on-failure?**
Because M7 already does, well, with timeout watchdogs, blocked-dependent tracking, idempotency, and structural replan. Duplicating it in C3 means two systems disagreeing about recovery. The right answer is: C3 produces a *better hypothesis* and *richer recovery hints*; M7 keeps owning the runtime.

**Why ship recovery hints in PlanSpec instead of letting M7 infer them?**
Because the planner has the *intent*; M7 only sees the *failure*. The intent (`empty_result_policy: ask_or_relax_constraints`) makes M7's recovery decision unambiguous. Without it, M7 has to choose between three plausible recoveries every time.

**Why must classifier/router be skill-aware but not skill-coupled?**
Classifier needs to know "is this even doable?" and "what kind of doable?" — that's capability-level. Coupling it to specific skills means every new provider rotation forces a planner change. The Capability Catalog is the firewall.

---

## What Becomes Possible (mental model shift)

The current planner answers: *"Given this request, what plan should I make?"*
The advanced planner answers: *"Given this request, **how much planning effort is justified**, and which planner should produce the plan?"*

That shift unlocks:

| Was not possible before | Becomes possible now |
|---|---|
| Predicting per-turn latency | Tier-mix telemetry tells us 65% T0, 25% T1, 10% T2 → median latency is bounded |
| Repeatable simple actions | Templates produce the same plan for the same request — mobile UX feels deterministic |
| Cost control as a product knob | Tier escalation rules are the dial; tighten/loosen per cohort or per LOB |
| Auditable routing decisions | Every turn has an inspectable RequestDecision JSON; QA can replay |
| Planner-as-a-policy, not planner-as-a-magic-box | The router rule-set is the policy; humans can read it, change it, and version it |
| Consent / approval prompts that are non-redundant | Smart clarify only asks for *critical* missing data; downstream approvals only for *risk* |
| Recovery that's guided, not improvised | M7 reads `failure_hints` and picks the right recovery branch the first time |
| Adaptive planning effort | Effort proportional to complexity. "Open Instagram" never pays the GPT-4 tax |

---

## Final Principle

> Planning is the **hypothesis**.
> Validation is the **contract**.
> Execution Intelligence is the **reality handler**.
>
> The planner's job is to spend the **least reasoning effort** that produces a **plan a real-world executor can succeed with** — and to hand the executor the *intent* and *recovery shape* needed to handle the messy parts.

---

## Research References

| Topic | Source |
|---|---|
| Plan-and-Execute (planner / executor split) | LangGraph plan-and-execute notebook; LLMCompiler — arXiv 2312.04511 |
| Cascaded model routing (cheap → expensive) | FrugalGPT — arXiv 2305.05176 |
| Cost-aware LLM cascades | RouteLLM — arXiv 2406.18665 |
| Plan templates & slot-filling for assistant tasks | Apple SiriKit intents; Google App Actions |
| Adaptive computation per query | "Adaptive computation time" — Graves 2016; Mixture-of-Depths |
| Hybrid retrieval as a classifier signal | LangChain Hybrid Search docs |
| Recovery hints from planner to executor | OpenAI tool-use guidelines, Anthropic action-safety |
| Mobile assistant latency budgets | Google Assistant On-device research; Apple Siri latency studies |
