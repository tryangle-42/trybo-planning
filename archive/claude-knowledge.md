***

# Agent Skills — Complete Write-Up

## The Core Insight: Stop Building Agents, Build Skills

The talk from Barry and Mahes (Anthropic) presents a fundamental shift in how to think about agents. The old model was: **one agent per domain**, each with its own scaffolding and tools. The new realization is that the agent underneath is more universal than expected — and the real gap is not capability, but **expertise**.[1]

The analogy they use is sharp: who do you want doing your taxes — a 300 IQ mathematical genius figuring it out from first principles, or an experienced tax professional? Agents today are like the genius. Brilliant, but without domain knowledge, consistent execution, or institutional memory. Skills are the answer to that gap.

***

## What Are Skills, Exactly?

**Skills are just folders.** That's the entire design philosophy.[2]

A skill is an organized collection of files that packages composable procedural knowledge for an agent. The structure of the `anthropic_brand/` example from your slides makes this concrete:

```
anthropic_brand/
├── SKILL.md          ← Entry point (YAML metadata + markdown instructions)
├── docs.md           ← Sub-skill: how to create documents
├── slide-decks.md    ← Sub-skill: slide deck instructions
└── apply_template.py ← Actual executable script as a tool
```

This simplicity is **deliberate**. Anyone — human or agent — can create and use a skill as long as they have a computer. They work with what already exists: Git for versioning, Google Drive for sharing, zip files for distribution.[2]

***

## How Skills Work at Runtime — Progressive Disclosure

This is the most important technical mechanism.[3]

Skills are **progressively disclosed** to protect the context window. The model doesn't load everything at once:

1. **At startup** — only the YAML frontmatter from `SKILL.md` is shown. This is a tiny snippet: just the `name` and `description` fields. The model simply knows the skill exists.
2. **When the agent decides to use a skill** — it reads the full `SKILL.md`, which contains the core instructions and a directory pointing to everything else in the folder.
3. **On-demand from there** — the agent reads only the sub-files it actually needs (e.g., `slide-decks.md`) and runs only the scripts relevant to the task.

This design means you can equip an agent with **hundreds or thousands of skills** without bloating the context window. Nothing is loaded until it's actually needed.

***

## Skills Can Include Scripts as Tools

[4]

Traditional tools have well-known problems: poorly written instructions, no ability for the model to fix or adapt them, and they always occupy context window space.

Scripts inside skills solve all of this:
- **Self-documenting** — code explains itself
- **Modifiable** — the agent can edit the script if needed
- **Lives on the filesystem** — loaded only when used
- **Reusable** — the agent can save a script it wrote once so its future self never rewrites it

The real example from the talk: Claude kept writing the same Python script to apply brand styling to PowerPoint slides. They simply asked Claude to save it inside the skill as a reusable tool (`apply_template.py`). The skill's `slide-decks.md` then references it with one line: *"Use the `./apply_template.py` script to update a pptx file in-place."*[4]

***

## The Skill Ecosystem — Three Categories

Since Anthropic's launch, thousands of skills formed around three types:

**1. Foundational Skills**
General or domain-specific capabilities the agent didn't have before.
- Anthropic's own document skills: create/edit professional-quality Office files
- Cadence's scientific research skills: EHR data analysis, bioinformatics Python libraries

**2. Third-Party / Partner Skills**
Skills built by ecosystem partners to make Claude better at their own products.
- **Browserbase** built skills for their Stagehand browser automation tool — Claude can now navigate the web more effectively
- **Notion** launched skills for deep research over an entire Notion workspace

**3. Enterprise / Team Skills**
The highest-traction category. Fortune 100s are using skills to:
- Teach agents organizational best practices
- Handle their internal bespoke software
- Deploy coding agents (like Claude Code) with company-specific code style guidelines to thousands of internal developers

***

## Where Skills Fit in the Emerging Agent Architecture

[1]

The full stack as described:

```
┌─────────────────────────────────────────────┐
│              AGENT LOOP                      │
│  (manages context, tokens in/out)            │
│                                              │
│  ┌──────────┐    ┌─────────────────────────┐│
│  │  Model   │◄──►│  Runtime Environment    ││
│  │(Claude)  │    │  (filesystem + bash)    ││
│  └──────────┘    └─────────────────────────┘│
└─────────────────────────────────────────────┘
        │                        │
        ▼                        ▼
  MCP Servers              Skill Library
  (external world         (procedural expertise,
  connectivity,           loaded on-demand,
  APIs, DBs)              hundreds/thousands)
```

**MCP and Skills are complementary, not competing:**
- MCP = connection to the outside world (APIs, databases, live data)
- Skills = expertise and procedural knowledge for *how to work*

Increasingly, skills are orchestrating *workflows of multiple MCP tools* stitched together. MCP handles the connection; the skill handles the logic of what to do with it.

***

## Skills as the "Software Layer" of the AI Stack

The talk ends with a clean analogy to computing history:

| Computing | AI Equivalent |
|---|---|
| Processor | Model (massive investment, immense potential, not useful alone) |
| Operating System | Agent Runtime (orchestrates resources, gets tokens in/out efficiently) |
| **Applications / Software** | **Skills** (encode domain expertise, unique POVs, solve real problems) |

A few companies build processors and OSes. **Millions of developers build software.** Skills are designed to open up that applications layer to everyone — including non-technical people in finance, legal, recruiting, and accounting.

***

## Skills as Continuous Learning — The Long Game

This is the most forward-looking part of the talk. Skills are designed as a **concrete mechanism for agents to learn over time**:

- **Day 1**: Claude is generic. Skills give it your domain context.
- **During work**: Claude can *create* skills — saving scripts, instructions, and patterns it discovers for its future self. The standardized format guarantees anything Claude writes down can be efficiently used by a future version of itself.
- **Day 30**: Claude already knows your team's practices, your tools, your code style, your workflows.
- **At scale**: When one person in an org teaches the agent something via a skill, all agents in the org benefit. When someone joins the team, the agent is already trained.

This creates a **compounding, collective, evolving knowledge base** — similar to how an MCP server built by someone across the world makes your agent better, a skill built by a community member does the same.

***

## What's Coming Next for Skills

The team is actively working on treating skills like software, which means:

- **Testing & Evaluation** — tooling to verify agents load and trigger skills at the right time, and that output quality meets expectations
- **Versioning & Lineage** — clear tracking of how a skill evolves and how agent behavior changes with it
- **Explicit Dependencies** — skills that declare dependencies on other skills, MCP servers, or packages, making agent behavior more predictable across runtime environments
- **Composability** — multiple skills working together to elicit complex behavior

***

Here is the extracted and organized write-up from Shaw's video on Claude Skills.

***

# Claude Skills — Shaw's Technical Deep Dive

## The Core Problem Skills Solve

Every LLM interaction has the same friction: **the clearer your instructions, the better the results**. In practice this plays out as either writing instructions from scratch every time, or maintaining a personal library of prompts that you manually copy-paste into Claude. Both work, but both are tedious at scale. Skills automate this entirely — write instructions once, and Claude intelligently pulls them in *only when relevant*, without you doing anything.

***

## What a Skill Actually Is — The File Structure

A skill is just a **named folder** with a `skill.md` file inside it. The folder lives at:

```
~/.claude/skills/<skill-name>/
```

Example for an AI Tutor skill:

```
ai-tutor/
├── skill.md                  ← Entry point (metadata + body)
├── research-methodology.md   ← Sub-instructions, loaded on-demand
└── scripts/
    └── get_youtube_transcript.py  ← Executable tool
```

The `skill.md` file has exactly two components:

**1. Metadata (YAML frontmatter)**
- `name` — max 64 characters
- `description` — max 1,024 characters; this is what Claude reads to decide if the skill is relevant

**2. Body**
- A normal markdown-formatted prompt — your full specialized instructions
- Can be up to ~5,000 tokens
- References to sub-files and scripts are written directly in the body (e.g., *"if concept requires research, load `research-methodology.md`"*)

***

## Progressive Disclosure — The Key Technical Mechanism

This is the most important design principle. Skills use **three levels of context injection**, each loaded only when needed:

| Level | What Gets Loaded | When | Token Cost |
|---|---|---|---|
| **Level 1** | Skill metadata (name + description) | At Claude startup, always | ~100 tokens per skill |
| **Level 2** | Full `skill.md` body | When Claude decides the skill is relevant | Up to 5,000 tokens |
| **Level 3** | Sub-files, folders, scripts | When the body references them and Claude needs them | Practically unlimited |

This means you can have **hundreds or thousands of skills** loaded simultaneously and their combined startup cost is still tiny — just the metadata. A skill with 5,000 tokens of instructions only uses those tokens when it's actually invoked. Everything else is zero cost.

***

## Sub-Files and Sub-Folders — Layered Instructions

Beyond the basic `skill.md`, a skill folder can contain:

- **Additional `.md` files** — specialized sub-instructions. E.g., a SaaS validator skill might have a `btoc-validation.md` that only loads when the user has a B2C idea.
- **Sub-folders** — for organizing larger collections of instructions logically (e.g., a `how-tos/` folder with separate files for X, Y, Z).
- **Scripts (`/scripts/`)** — Python, JavaScript, or bash scripts Claude can execute directly. Claude has access to a bash shell with Python and Node.js, so it can run `python scripts/get_youtube_transcript.py <url>` directly as a tool call.

The body of `skill.md` acts as the **table of contents** — it tells Claude what sub-files exist and when to load them. Claude reads them one at a time, only as needed.

***

## Real Example: AI Tutor Skill

Shaw's concrete demo had three components working together:

**`skill.md`**
- Metadata: *"Use when user asks to explain, breakdown or help understand technical concepts, AI/ML or other technical topics. Makes complex ideas accessible through plain English and narrative structure."*
- Body: Full prompt engineering instructions including a narrative structure (status quo → problem → solution → example → why it matters) and a directive to *"think hard, explore multiple narratives before responding"*

**`research-methodology.md`** (sub-file, ~200 lines)
- Triggered by: *"If a concept is unfamiliar or requires research, load research-methodology.md for detailed guidance"* in the skill body
- Not loaded for well-known concepts (neural networks, gradient descent) — only for cutting-edge or unfamiliar topics

**`scripts/get_youtube_transcript.py`** (tool)
- Referenced inside `research-methodology.md` (not `skill.md` directly)
- Takes a YouTube URL or video ID, returns the full transcript
- Uses `uv` for dependency management to avoid runtime package errors
- Claude invokes it via: `uv run scripts/get_youtube_transcript.py <video_url>`

**Demo behavior observed:**
- Asked "explain reinforcement learning" → Claude auto-detected AI Tutor skill was relevant, loaded it without being told
- Asked "explain GRPO, do research" → Claude loaded `research-methodology.md`, used built-in web search
- Asked to find and explain a YouTube video → Claude ran the transcript script successfully via terminal

***

## Skills vs. MCP — When to Use Each

Shaw addresses the confusion directly. They overlap significantly but have different sweet spots:

| Dimension | Skills | MCP |
|---|---|---|
| **What it provides** | Instructions + tools + prompts | Tools + resources + prompts |
| **Works with** | Claude only | Any LLM (open standard) |
| **Context at startup** | ~100 tokens (metadata only) | Full tool list dumped upfront (e.g., Notion MCP = ~20,000 tokens) |
| **Progressive disclosure** | ✅ Yes — 3 levels | ❌ No — everything loads at startup |
| **Best use case** | Teaching Claude *how* to use its existing tools | Giving Claude access to *complex external integrations* |
| **Build effort** | Low — markdown files and scripts | Medium/High — requires API understanding and custom tooling |

**The Notion example makes the cost concrete:** Notion's MCP server dumps ~20,000 tokens into context at startup whether you use Notion or not. A Notion skill would dump only ~100 tokens at startup. That's a **200x difference** in context cost.

**When to still use MCP:** When the integration is complex enough that building it from scratch as a skill would require deep API work. Off-the-shelf MCP servers (like Browserbase, Notion, Context7) are still worth using for their integrations — and skills can *orchestrate* those MCP tools.

***

## How Skills, MCP, and Sub-Agents Fit Together

In a full Claude Code session, these three things layer cleanly:

```
Main Claude Code Agent (context window)
├── System prompt
├── Default tools
├── Skill metadata (all skills, ~100 tokens each) ← lightweight
├── MCP server tools (full dump at startup)        ← heavier
└── User messages + Claude responses

         │
         │ spins off
         ▼

Sub-Agent (separate context window)
├── All skills (same library, metadata level)
├── ONE specific MCP server (not all of them)
└── Task result returned to main agent
```

**Sub-agents** solve a different problem: context isolation. When you want Claude to do a deep research task (e.g., understand the FastHTML library), you don't want that research polluting your main coding context. The sub-agent does the work in its own window and returns just the result. Sub-agents share the same skills library but can be given a targeted MCP server (e.g., Context7 for live docs).

***

## How to Actually Build Skills (Shaw's Workflow)

Shaw's practical advice: **have Claude write the skill for you.**

1. Open Claude in the browser
2. Describe what you want the skill to do
3. Ask it to write the `skill.md` and any sub-files
4. Give ~6 rounds of feedback until it's right
5. Drop the folder into `~/.claude/skills/`

For scripts with Python dependencies, use `uv` to manage packages inline — this prevents Claude from hitting dependency errors when it tries to execute tools at runtime.

***

## Mental Model Summary

> A skill is a folder. The folder has a `skill.md` with tiny metadata (100 tokens) and a full instruction body (up to 5,000 tokens). The metadata always lives in context. The body loads only when relevant. The body can point to sub-files and scripts that load only when *they're* relevant. Scripts are real executable tools Claude runs in its bash shell. This three-level progressive disclosure lets you have unlimited skills without context pollution — Claude always has just enough context for the next step, and nothing more.
