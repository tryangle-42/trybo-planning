# Baseline Components → User-Facing Impact Map

> For each baseline component planned through September, what can a user DO on the mobile app that they couldn't before?
>
> Target persona: Urban professional in India (25-35), smartphone-first, manages work + personal life, makes 5-10 calls/day, uses WhatsApp heavily, wants to delegate routine communication + research.

---

## How to Read This

Components are grouped by the **user capability they unlock**, not by internal module. Each row maps an engineering baseline to a concrete thing the user experiences in the chat app.

**Three categories:**
- **New capability** — something the user literally cannot do today
- **Friction reduction** — something the user can do but it's slow, broken, or manual
- **Trust & safety** — things that happen invisibly but determine whether the user trusts the product

---

## 1. Talk to the Agent and Get Things Done

*The core loop: user types in chat → agent understands → plans → executes → returns result*

| Baseline Component | User Impact | Example Prompt | Before vs After |
|-------------------|------------|---------------|-----------------|
| **GID Semantic Router** (RKP) | Agent responds to simple requests in <500ms instead of 2-3 seconds | "Call John" | Before: 2-3s (LLM classifies intent). After: <500ms (embedding match → fast path → call initiated). User feels instant response. |
| **Fast Path Execution** (RKP) | Simple one-step tasks skip the planner entirely. Feels like the agent "just does it." | "What's the weather in Delhi?" / "Call mom" / "Set a reminder for 3pm" | Before: Every request goes through full planning pipeline (2-3s overhead). After: Single-skill intents execute directly. ~80% of daily requests are fast-path eligible. |
| **LLM Planner + Clarification** (RKP) | Complex multi-step requests are handled as a plan. Agent asks for missing info before acting. | "Research our competitors and call our top 3 partners about the findings" | Before: Agent either fails or guesses. After: Agent says "Which competitors? And which partners?" → user answers → agent creates a 7-step plan and executes it. |
| **Plan Validation** (RKP) | Agent never attempts impossible plans (referencing non-existent skills, circular dependencies) | Any complex request | Before: Plan fails mid-execution with cryptic error. After: Invalid plans caught before execution. User sees "I can't do X because Y" instead of a crash. |
| **Skill Retrieval API** (AKN) | Agent finds the right tool for any request from 500+ capabilities, without being overwhelmed | "Send the meeting notes to the team on Slack" | Before: All 70 skills injected into planner prompt (slow, hits token limit at scale). After: Only the 5 most relevant skills shown to planner. Faster, more accurate. |
| **Skill Health Metrics** (AKN) | Broken tools are automatically excluded. Agent never tries to use a service that's down. | Any request using an external service | Before: If Tavily search is down, agent tries it, fails, shows error. After: Agent transparently uses Perplexity instead. User never sees the failure. |
| **Context Assembly** (KPR) | Agent remembers what you said earlier — in this conversation and past ones | "Change the hotel in my Goa trip to something cheaper" (referencing a trip planned yesterday) | Before: Agent has no memory of yesterday's plan. After: Episodic memory recalls the Goa trip plan, semantic memory knows your budget preference. Agent modifies the right thing. |
| **Conversation Buffer Fix** (KPR) | Agent doesn't lose context mid-conversation. Older messages summarized, recent ones verbatim. | Long conversations (20+ messages) | Before: After 15 messages, planner loses early context. After: Hybrid buffer (last 5 verbatim + summary of earlier messages) keeps full context within token budget. |

**Frequency:** The core chat loop is used 10-30 times/day per active user. Fast path alone affects ~80% of those interactions.

---

## 2. Make and Receive Calls Through the Agent

*The biggest GTM capability: voice calls flow through the agent's brain*

| Baseline Component | User Impact | Example Prompt | Before vs After |
|-------------------|------------|---------------|-----------------|
| **Inbound Calls Through Orchestration** (RKP) | When someone calls you, the agent handles the conversation with full context about who's calling and why | Phone rings from Sharma | Before: Twilio handler answers with generic script. After: Agent loads Sharma's contact history, recent conversations, pending items. Says "Hi Sharma, I believe you're calling about the meeting reschedule?" |
| **Outbound Calls Through Orchestration** (RKP) | "Call X about Y" works end-to-end from chat | "Call Sharma about tomorrow's 3pm meeting" | Before: Not possible through chat. User must manually call. After: Agent assembles meeting context from calendar, shows consent card with draft talking points, user approves, call happens, summary returned in chat. |
| **Outbound Call Consent** (RKP) | User sees what the agent will say before the call happens. Can edit the draft. | "Call my dentist and reschedule to next week" | Before: No consent mechanism. After: Agent shows: "I'll call Dr. Patel's clinic and say: 'I'd like to reschedule Avinash's appointment from Thursday to next week. What times are available?'" User can edit before approving. |
| **WhatsApp Through Orchestration** (RKP) | Same agentic flow for WhatsApp messages | "Send Sharma the meeting agenda on WhatsApp" | Before: WhatsApp handler sends directly (no planning, no memory context). After: Agent drafts message using meeting details from calendar, shows consent, sends on approval. |
| **Voice Latency <800ms** (RKP) | Conversations during calls feel natural, not robotic | During any agent-handled call | Before: 1.5-2.5s delay between caller speaking and agent responding (feels broken). After: <800ms voice-to-voice (feels like talking to a thoughtful person). |
| **Hindi/Hinglish ASR** (RKP) | Agent understands Hindi and mixed Hindi-English during calls | Caller says "meeting kal 3 baje hai, Sharma ji ko call kar dena" | Before: Western ASR drops Hindi words or hallucinates English substitutes. After: Sarvam ASR handles code-switching natively. The agent actually understands what was said. |
| **Filler Injection** (RKP) | Agent says "Let me check that..." while processing, instead of dead silence | Agent needs 2-3 seconds to look something up mid-call | Before: 3 seconds of silence (caller thinks connection dropped). After: "One moment, let me pull that up..." → result arrives → conversation continues naturally. |
| **Interruption Handling** (RKP) | If the user speaks while the agent is talking, agent stops immediately | User interrupts mid-sentence | Before: Agent keeps talking over the user. After: Agent stops, listens to the user's new input, responds to that instead. |
| **Hold Behavior** (RKP) | When waiting for consent or long tasks, agent plays periodic reassurance | Agent waiting for user to approve consent on another device | Before: Silence on the call. Caller hangs up. After: "Please hold, I'm checking that for you..." every 8-10 seconds. Caller stays on the line. |

**Frequency:** Target persona makes 5-10 calls/day. If even 2-3 of those can be delegated to the agent, that's 15-30 minutes/day saved (including the prep/context-gathering the user would do manually).

---

## 3. Control What the Agent Does

*Consent, approvals, and trust — the user is always in control*

| Baseline Component | User Impact | Example | Before vs After |
|-------------------|------------|---------|-----------------|
| **Consent Cards in Chat** (KPR) | Every side-effecting action shows an approve/edit/deny card in the chat thread | Agent wants to send an email | Before: Consent shown in a separate screen (disconnected from chat flow). After: Inline card in the chat thread — user taps approve, chat continues. Natural flow, no context switch. |
| **Risk-Tiered Approval** (RKP) | Read-only actions happen silently. Calls/emails need approval. Payments need extra confirmation. | "What's on my calendar?" vs "Call Sharma" vs "Pay the invoice" | Before: Every action requires approval (approval fatigue → user auto-approves without reading). After: Calendar check = silent. Call = one-tap approve. Payment = explicit confirmation + audit. User attention directed to decisions that matter. |
| **Unified Consent Pending List** (KPR) | One place to see all pending approvals (inbound + outbound, calls + WhatsApp) | User opens the app after being away for an hour | Before: Separate lists for inbound responses and outbound approvals. Confusing. After: One list: "3 items need your approval" — call to Sharma, WhatsApp to vendor, email draft to team. |
| **Run/Batch/Task Visibility** (RKP) | User sees granular progress: which batch is running, which tasks completed, which failed | Agent researching 3 competitors and calling 3 partners | Before: Spinner with "Working on it..." After: "Researching competitors (2/3 done) → Calling partners (waiting for your approval on each) → Will summarize results." User sees exactly where things stand. |
| **Task-Level Retry** (RKP) | When one step fails, user can retry just that step without restarting everything | Call to partner #2 failed (busy line), but research and other calls succeeded | Before: Entire plan fails. User must repeat the whole request. After: "Call to Mehta failed (busy). Retry?" User taps retry. Only that call is reattempted. Other results preserved. |
| **Run Cancellation** (RKP) | User can cancel mid-execution at any level (single task, batch, or entire run) | User realizes they gave wrong info after agent already started | Before: No cancel button. Agent runs to completion even if wrong. After: User taps cancel. In-flight tasks complete safely (no orphaned side effects), pending tasks skipped. |

**Frequency:** Consent interactions happen 3-8 times/day for an active user. The difference between "tap approve and continue" (inline card) vs "navigate to a separate screen" saves ~5 seconds per approval × 5 approvals/day = ~25 seconds/day. Small per-interaction, significant in feel.

---

## 4. Get Personalized, Context-Aware Responses

*The agent knows who you are and gets smarter over time*

| Baseline Component | User Impact | Example | Before vs After |
|-------------------|------------|---------|-----------------|
| **Episodic Memory** (KPR) | Agent remembers what it did for you in past sessions | "Change the hotel in my Goa trip" (planned 3 days ago) | Before: "I don't have any information about a Goa trip." After: Agent recalls the full trip plan — flights, hotel, activities — and modifies just the hotel. |
| **Semantic Memory (mem0)** (KPR) | Agent remembers facts about you — preferences, relationships, context | "Book a flight to Mumbai" (user always prefers IndiGo, window seat, morning flights) | Before: Agent asks every preference every time. After: Agent knows your preferences and books accordingly: "I found an IndiGo morning flight, window seat, ₹4,200. Book it?" |
| **Procedural Memory** (KPR) | Agent knows what data it already has about you (voice profile, connected accounts, contacts) | "Set up my assistant" | Before: Agent asks "Do you have a voice profile?" even though you set one up last week. After: Agent knows: "You have a voice profile (ElevenLabs), Google connected, 47 contacts imported." Plans accordingly. |
| **Persona Switching** (RKP) | Switch between specialized agent modes — each with its own voice, skills, and memory | "Switch to research mode" or tap persona in UI | Before: One generic agent tries to do everything. After: Chief of Staff persona manages calendar + calls with a professional voice. Research Assistant digs deep into web search with a concise, data-driven style. Personal Assistant handles daily logistics warmly. |
| **Persona Skill Constraints** (RKP) | Each persona only uses relevant skills — Chief of Staff doesn't try to do research, Research Assistant doesn't make calls | User in Research mode asks "also call Sharma" | Before: Any request routed to any skill (can produce bad plans). After: Research persona says "I can't make calls — switch to Chief of Staff mode?" Or auto-switches if configured. |
| **Persona Memory Separation** (RKP) | Chief of Staff's scheduling context doesn't pollute Research Assistant's search context | Switch between personas throughout the day | Before: All context mashed together (scheduling facts show up during research). After: Each persona has its own memory namespace. Clean, focused context per mode. |

**Frequency:** Memory-informed responses improve almost every interaction (10-30/day). Persona switching happens 2-5 times/day for power users. The cumulative effect: the agent feels like it *knows* you, not like a stateless chatbot you have to re-explain things to every time.

---

## 5. Create Reusable Automations

*"Do this every morning" — user-created workflows from chat*

| Baseline Component | User Impact | Example Prompt | Before vs After |
|-------------------|------------|---------------|-----------------|
| **Skill Composition** (AKN) | User creates multi-step automations via natural conversation | "Every morning at 7am, check weather, summarize my calendar, and send me a WhatsApp briefing" | Before: Not possible. Each request is one-shot. After: Agent saves this as "Morning Briefing" automation, runs daily at 7am, results delivered via WhatsApp. User created a Zapier-like workflow by just talking. |
| **User-Defined Schedules** (AKN) | Automations can be scheduled on cron (daily, weekly, etc.) | "Every Friday at 5pm, check my unresolved tasks and send me a summary" | Before: User must remember to ask every Friday. After: Agent runs it automatically. User gets a WhatsApp summary every Friday at 5pm without asking. |
| **Composed vs Third-Party Skills** (AKN) | If a third-party tool does what your automation does but faster, agent can use it transparently | Agent finds a "daily briefing" MCP tool that's faster than chaining 5 skills | Before: Only user-created automations. After: Agent uses the fastest option automatically. User's automation still works as fallback if third-party degrades. |

**Frequency:** Power users create 3-5 automations in the first week, then rely on them daily. Each automation saves 5-15 minutes of manual work per execution. A "Morning Briefing" running daily saves ~1.5 hours/week.

---

## 6. Trust the Agent with Sensitive Data

*Invisible but critical — determines whether users give the agent access to real data*

| Baseline Component | User Impact | Example | Before vs After |
|-------------------|------------|---------|-----------------|
| **Input Content Safety** (RKP) | Harmful or adversarial inputs are blocked before reaching the LLM | Someone tries to trick the agent via a forwarded message | Before: Agent follows injected instructions blindly. After: Safety classifier catches adversarial patterns. Agent ignores the attack. |
| **Output PII Scanning** (RKP) | Agent never leaks your phone number, Aadhaar, or financial data to external services | Agent uses a web search API to look something up | Before: Your personal details included in the API call to a third-party search service. After: PII redacted from all external API calls. Your data stays private. |
| **Audit Trail** (RKP) | Every action the agent took is logged — user can review what happened | User asks "what did my agent do while I was away?" | Before: No record of agent actions beyond chat messages. After: Full audit: "Sent WhatsApp to Sharma (approved), searched flights (auto-approved), drafted email to team (pending)." |
| **Rate Limiting** (RKP) | Agent can't run away — hard limits on calls, messages, API usage per hour | Bug or misconfiguration causes agent to loop | Before: Agent could theoretically make 1000 calls. After: Capped at 30 runs/hour, 100 API calls/hour. Safety net against runaway automation. |
| **RBAC (Two Roles)** (RKP) | Admin users can manage skills and system config; regular users can't accidentally break things | Two people share an account | Before: Single user role — anyone can change anything. After: Admin manages settings, regular user interacts with the agent. |
| **Field Encryption** (KPR) | API keys, OAuth tokens, and financial data encrypted at rest — even a DB breach doesn't expose them | N/A (invisible to user) | Before: API keys stored as plaintext env vars. After: Application-level encryption. Even if database is compromised, keys are unreadable. |
| **Payment Gateway** (KPR) | User can pay for the service, manage subscription, see invoices | "Upgrade my plan" / "Show my invoice" | Before: No way to pay. Free-only, unsustainable. After: Stripe-powered subscriptions, self-serve billing portal, invoicing with GST. Revenue starts flowing. |

**Frequency:** Safety measures are always-on. The user never "uses" them directly — but the moment data leaks or the agent does something unauthorized, trust is destroyed. These baselines are the difference between "I'll give it access to my Google account" and "I'll never connect real data."

---

## 7. Developer-Facing (Not User-Visible, But Enables Everything)

*These components are invisible to the user but directly impact product quality and ship speed*

| Baseline Component | What it enables | Impact on user |
|-------------------|----------------|---------------|
| **CLAUDE.md (backend + frontend)** (AKN) | AI-assisted development produces correct, convention-following code from day 1 | Fewer bugs, faster feature delivery |
| **Debug Logs + Spans** (AKN) | Developer can trace any user request end-to-end and find failures in <2 minutes | Bugs fixed faster → fewer user-visible failures |
| **Analytics (10 baseline questions)** (AKN) | Team knows: plan success rate, skill failure rate, call latency, LLM cost | Data-driven prioritization → team works on what matters most to users |
| **Error Classification** (RKP) | Every failure categorized → smart retry vs skip vs ask user | User sees "retry?" instead of cryptic error messages |
| **Plan Quality Scoring** (RKP) | Track whether plans are getting better or worse over time | Plans improve → user's requests succeed more often |
| **import-linter in CI** (AKN) | Module boundaries enforced automatically on every PR | Architecture stays clean → fewer cross-module bugs |
| **Unit Testing + CI** (KPR) | Code changes verified before deploy | Fewer regressions → stable experience |
| **Review Subagents** (RKP) | AI catches code quality, security, and architecture issues before merge | Higher code quality → fewer production bugs |
| **Self-Learning Repo** (AKN) | Architecture docs auto-update with code changes | New hires productive faster → faster team scaling → faster feature delivery |

---

## Summary: What Changes for the User by September

### Before (today)

The user has a chat app where they can:
- Send messages, get AI responses
- Make calls (but they bypass the agent's brain)
- Send WhatsApp (but without planning or context)
- No memory across sessions
- No automations
- No consent control beyond basic approve/deny
- No persona specialization

### After (September baseline)

| Capability | How often used | Time saved per use | Monthly impact |
|-----------|---------------|-------------------|----------------|
| **Fast responses** (<500ms for simple requests) | 20-30× /day | ~2s per interaction | ~20-30 min/month (perceived speed, not just time) |
| **Agent-handled calls** (inbound + outbound through AI brain) | 2-3 calls/day delegated | 5-15 min per call (context prep + execution) | ~3-7 hours/month |
| **Agent-handled WhatsApp** (with context + consent) | 3-5 messages/day delegated | 2-5 min per message (draft + review) | ~2-4 hours/month |
| **Memory-informed responses** (knows your preferences, history) | Every interaction | Eliminates re-explanation (~30s each time) | ~2-3 hours/month |
| **Reusable automations** (morning briefing, weekly review, etc.) | 5-7 automations running daily/weekly | 5-15 min per automation per execution | ~5-10 hours/month |
| **Persona switching** (Chief of Staff, Research, Personal) | 2-5 switches/day | Better results per interaction (right tools, right voice, right style) | Quality improvement, not time savings |
| **Granular consent + progress** (see what agent is doing, approve inline) | 3-8 approvals/day | ~5s per approval (inline vs separate screen) | ~10 min/month + trust |
| **Failure recovery** (retry one task, not the whole thing) | 1-2 failures/day | Saves full re-execution (~2-5 min) | ~1-2 hours/month |

**Total estimated impact for an active user: ~13-26 hours/month saved**, plus qualitative improvements in trust, personalization, and perceived speed.

### The Prompts That Become Possible

These are things a user can type in the chat app by September that don't work today:

| Prompt | What happens | Which baselines enable it |
|--------|------------|--------------------------|
| "Call Sharma about tomorrow's meeting" | Agent pulls meeting details from calendar, shows consent card with draft talking points, user approves, call happens, summary in chat | Agentic call flows, consent cards, context assembly, voice pipeline |
| "Research our top 3 competitors and email me a summary" | Agent creates a 3-task research batch + 1 synthesis task + 1 email task. Shows progress per competitor. | Planner, batch orchestration, run→batch→task visibility, email skill |
| "Every morning at 7, check weather and summarize my calendar" | Agent saves as "Morning Briefing" automation, runs daily, delivers via WhatsApp | Skill composition, scheduled jobs, WhatsApp through orchestration |
| "Switch to research mode" | Agent changes persona — different voice, different skills (web search + docs, no calls), different response style | Persona management, skill constraints, voice persona |
| "Change the hotel in my Goa trip to something cheaper" | Agent recalls the trip plan from 3 days ago, knows "cheaper" = your budget preference, modifies just the hotel | Episodic memory, semantic memory, partial re-planning |
| "What did my agent do today?" | Shows full audit: 3 calls made, 5 WhatsApp messages sent, 2 research tasks completed, with outcomes | Audit trail, run history, task-level results |
| "Remind me every Friday to review my unresolved tasks" | Agent creates scheduled automation, runs weekly, summarizes pending tasks via WhatsApp | Skill composition, scheduled jobs, task management skills |
| "कल 3 बजे Sharma जी को call कर देना meeting के बारे में" | Agent understands Hindi-English mixed input, schedules the call for tomorrow 3pm, uses Hindi-capable voice | Hindi ASR (Sarvam), multilingual planner, voice persona |
