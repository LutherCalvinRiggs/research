# Obsidian Masterclass: Empty Vault to AI-Powered Second Brain

**Source:** https://x.com/cyrilXBT/status/2063992615721435154 (pasted content by @cyrilXBT)
**Saved:** 2026-06-10
**Tags:** ai, productivity, tools, learning, prompting, research

---

## TL;DR
A complete 15-part system for building an Obsidian vault that compounds over time — from the philosophy that makes it worth learning (permanent local files, connected ideas, AI intelligence) through the architecture, naming conventions, CLAUDE.md configuration, linking discipline, and five automated Claude workflows that transform the vault from passive storage into an active thinking partner. The three-month build plan takes it from empty to genuinely intelligent.

## Key Concepts & Terms
- **Retrieval-first principle**: Every organizational decision optimized for "when I need this in the future, what will I know about it that I can use to find it?" — not "what category does this belong in right now?" The filing cabinet question optimizes capture; the retrieval question optimizes all future access.
- **Atomic note**: One idea per note, not one topic per note. A reusable asset that can be linked from many contexts. Atomic notes compound; topic dumps don't. Written in your own words from your own understanding, not a collection of what others said.
- **Connection principle**: The value of a note is not in what it contains — it's in what it connects to. A rough note connected to ten others is worth more than a polished note connected to none.
- **Output principle**: A vault that never produces output is a hobby. The measure of a second brain is how many decisions it improves, how many pieces of writing it fuels, how many insights it surfaces before you need them.
- **CLAUDE.md**: The most important file in the vault. Read at the start of every Claude interaction. Contains identity, vault purpose, intellectual interests, active projects, current theses, decision-making context, and intelligence layer instructions. The quality of everything Claude produces is determined by its specificity.
- **Map of Content (MoC)**: A note whose primary purpose is linking to other notes on a topic and adding commentary on how they relate. Create one when a topic accumulates 15+ permanent notes. Contains the core question, foundation notes, complications, applications, synthesis links, and open questions.
- **08-COMPOUND folder**: Did not exist in traditional Obsidian setups. The Claude-generated output layer — connections discovered between notes, patterns identified across captures, syntheses generated from accumulated permanent notes. The intelligence layer made visible.
- **Active Theses**: Positions you currently hold that new information might support or contradict. The most underused section of CLAUDE.md. The thesis monitor that checks new notes against your positions is what makes the intelligence layer genuinely useful.
- **Two-link rule**: No permanent note gets filed without at least two explicit links to existing notes. The first connection is usually obvious. The second requires genuine thought. The harder thinking produces better connections.
- **Comprehension debt (vault version)**: A vault full of literature notes from sources is a collection of what others said, not what you understand. Processing into permanent notes in your own words is the conversion that creates genuine second-brain value.

## The Vault Architecture

```
VAULT/
    00-INBOX/        ← capture buffer, empties nightly
    01-PERMANENT/    ← atomic notes in your words, the heart of the vault
    02-LITERATURE/   ← source notes (books, articles, podcasts)
    03-MAPS/         ← Maps of Content, navigational layer
    04-PROJECTS/     ← one subfolder per active project
    05-DAILY/        ← one file per day
    06-RESOURCES/    ← reference material
    07-OUTPUTS/      ← finished work produced from the vault
    08-COMPOUND/     ← Claude-generated outputs (NEW — the intelligence layer)
    09-ARCHIVE/      ← completed material (never delete, archive)
    10-SYSTEM/       ← CLAUDE.md, templates, skill files
```

**Five architecture principles:**
1. Every note in exactly one folder — no duplicates, nothing at root
2. Folders represent types of content, not topics (topics are handled by tags and links)
3. Naming convention navigable without opening folders
4. Designed for 10,000 notes, not 100
5. Claude can navigate it without a map

## File Naming Convention

```
YYYY-MM-DD-[TYPE]-[TOPIC].md

Type identifiers:
  daily   → daily notes
  perm    → permanent notes
  lit     → literature notes
  map     → maps of content
  proj    → project overview
  meet    → meeting notes
  dec     → decision notes
  ref     → reference notes

Examples:
  2026-06-09-daily-monday.md
  2026-06-08-perm-compounding-knowledge.md
  2026-06-07-lit-thinking-fast-and-slow.md
  2026-06-05-map-productivity.md
  2026-06-01-dec-switch-to-obsidian.md
```

Three pieces of information (type + date + topic) make every note findable without folder navigation.

## The CLAUDE.md Template

```markdown
# Vault Intelligence System — CLAUDE.md

## Identity
Name: [YOUR NAME]
Primary domain: [YOUR WORK AND FOCUS]
Stage: [WHERE YOU ARE IN YOUR WORK]

## Vault Purpose
[One paragraph — specific domains, questions, projects this vault serves]

## Primary Intellectual Interests
[8-12 specific topics you think about regularly]

## Active Projects
[PROJECT NAME]: [One sentence status]
Current phase: [PHASE]
Next action: [SPECIFIC NEXT STEP]

## Current Active Theses
Thesis 1: [STATEMENT]
Supporting evidence: [BRIEF SUMMARY]
Counter-evidence: [WHAT CHALLENGES IT]
Last updated: [DATE]

## Decision Log Context
[How you make decisions and what you care about]

## Content Standards
[Voice, format, topics, what you never publish]

## Vault Navigation
[Folder map]

## Intelligence Layer Instructions
When finding connections: strong connection = reading one note changes how you read the other
When monitoring theses: flag contradictions, not complexity
When generating syntheses: the synthesis must say something no individual note says
When detecting patterns: requires 4+ notes from different sources

## Update Schedule
Active projects: weekly | Active theses: monthly | Primary interests: quarterly
```

## The Five Automated Claude Workflows (via Filesystem MCP)

**Setup:**
```json
{
  "mcpServers": {
    "obsidian": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/vault"]
    }
  }
}
```

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| Morning Brief | 6AM daily | Reads CLAUDE.md + active projects + yesterday's open loops + external developments → structured brief linked in daily note |
| Evening Capture Processor | 8PM daily | Files tasks to projects, creates permanent notes from ideas, stores decisions, carries open loops to tomorrow. Inbox clears itself. |
| Nightly Connection Finder | 11PM daily | Reads notes modified in last 48h, searches entire vault for strong connections not yet explicit → connection notes in 08-COMPOUND |
| Weekly Pattern Detector | Sunday 6PM | Reads all notes from the week, identifies structural patterns across multiple sources → named patterns in 08-COMPOUND |
| Monthly Synthesis Generator | 1st of month | For any topic with 10+ permanent notes, synthesizes across them → output in 08-COMPOUND requires all notes to produce |

## The Six Core Claude Prompts

**Connection Finder:**
```
Read [[NOTE NAME]] and search my entire vault for strong connections not already explicit.
Strong connection = reading both together reveals something neither alone reveals.
Skip obviously same-topic connections.
```

**Question Answerer:**
```
I'm working on: [QUESTION]
Search my vault first. Tell me what my accumulated notes already say.
Then tell me what the vault can't answer.
```

**Decision Support:**
```
I'm making this decision: [DECISION]
Read CLAUDE.md for active theses. Search vault for similar historical decisions.
Tell me: relevant history, the assumption this rests on, what would have to be true for it to go wrong, your honest recommendation.
```

**Writing Activator:**
```
I'm writing about: [TOPIC]
Search 01-PERMANENT for every relevant note.
Tell me: the strongest claim my notes support, specific evidence available, counterarguments my notes raise, what's missing.
Don't tell me what to write. Show me what my notes already know.
```

**Thinking Partner:**
```
Read my recent notes from the last 14 days across 05-DAILY and 01-PERMANENT.
Find: contradictory positions, arguments without evidence, questions referenced multiple times without being addressed.
Push on each one. Don't validate. Challenge.
```

**Vault Audit:**
```
Tell me: permanent notes with fewer than 2 connections, topics accumulating without a MoC,
active theses not updated in 90+ days, most connected notes and what that reveals about my actual interests.
```

## The Permanent Note Writing Process

The most important skill. Most people skip it and end up with a vault of what others said, not what they understand.

```
1. Read the capture actively (engage, don't just skim)
2. Write from memory without looking at the source
   → If you can't, you don't understand it yet. Go back.
3. Find two non-obvious connections in the vault
4. Write the note:

---
type: permanent
status: active
created: [DATE]
tags: [topics]
---
# [CONCEPT NAME]

[Your understanding in 2-4 sentences. Not what the source said. What you understand.]

## Why This Matters
[Why this concept matters for questions you're currently working on]

## The Tension
[The opposing view or complicating factor]

## Connections
- [[CONNECTION 1]] — [how they relate]
- [[CONNECTION 2]] — [how they relate]

## Source
[[LITERATURE NOTE]] — [what the source contributed to this understanding]

5. File to 01-PERMANENT
```

Done when it says something in your words that required reading and thinking to produce — not when it contains the information.

## The Seven Essential Plugins

1. **Templater** — dynamic templates with auto-filled dates
2. **Dataview** — query vault like a database (essential after 200+ notes)
3. **Calendar** — visual calendar sidebar, auto-creates daily notes
4. **Periodic Notes** — manages daily/weekly/monthly/quarterly notes
5. **Outliner** — better bullet point handling
6. **Excalidraw** — diagrams inside Obsidian
7. **Style Settings** — theme customization

Install these seven. Ignore everything else until the habits are built.

## Three-Month Build Plan

**Month 1 — Foundation**
- Wk 1: Folder structure, CLAUDE.md, plugins, templates
- Wk 2: Daily note habit (One Thing, capture, end of day)
- Wk 3: First permanent notes — 3/week, two-link rule
- Wk 4: Connect Claude via MCP, run morning brief manually
- Goal: 12+ permanent notes, daily habit established, Claude connected

**Month 2 — Intelligence**
- Wk 5-6: All five automated workflows running, review 08-COMPOUND quality, refine CLAUDE.md
- Wk 7: First Map of Content for most-accumulated topic
- Wk 8: Six manual prompts on real work (actual decisions, actual writing)
- Goal: Workflows running, first MoC built, prompts integrated into real work

**Month 3 — Compounding**
- Wk 9-10: Connection density work, vault audit, MoCs for 15+ note topics
- Wk 11: Assess pattern quality in 08-COMPOUND, refine if needed
- Wk 12: Write a manual synthesis, compare to Claude-generated version
- Goal: Vault producing genuine insights you wouldn't produce manually

## Questions & Gaps
- The Filesystem MCP gives Claude read access to the entire vault — what are the privacy implications for notes containing sensitive personal or professional information?
- The evening capture processor "creates permanent notes from ideas" automatically — how does Claude distinguish a raw idea worth turning into a permanent note from a passing thought worth discarding?
- The two-link rule forces non-obvious connections. How do you handle a genuinely new idea that doesn't yet connect to anything in the vault without violating the rule?
- The Active Theses section is described as "the most underused" — is there a template or prompting discipline that makes thesis-tracking sustainable over time rather than a one-time setup exercise?
- The 08-COMPOUND folder is Claude-generated. How do you maintain provenance and reliability — distinguishing Claude's synthesis from your own thinking when reviewing the vault months later?

## Related Notes
- [Claude Code + NotebookLM + Obsidian Pipeline](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/claude-code-notebooklm-obsidian-research-pipeline.md) — that note treats the Obsidian vault as a destination for research pipeline output; this masterclass treats it as the source of intelligence. The two together describe the full loop: research flows in via the pipeline, intelligence flows out via the six prompts.
- [Claude Projects 6-Part Blueprint](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/claude-projects-6-part-blueprint.md) — the CLAUDE.md in this vault is structurally identical to the Identity Block + Rules Block + Process Block from that blueprint, just implemented as a file rather than a project system prompt. The weekly reflection update pattern is also the same.
- [kiro-config Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/kiro-config-repo-overview.md) — kiro-config's steering files (tech.md, product.md, code-standards.md) are the same concept as CLAUDE.md applied to a development workflow rather than a personal knowledge vault. The "always-on context" pattern is identical.
- [Loop Engineering — Addy Osmani](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-addy-osmani.md) — the five automated Claude workflows here are loop engineering applied to knowledge management: automations on a schedule, state file (the vault itself), sub-agents (different workflows for different tasks), and a verifier (vault audit prompt).
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — Jed's "plan is the negative, code is a print" maps directly onto the vault philosophy here: "the permanent note is the durable artifact; outputs are prints from it." Both argue the same thing about where durable value lives.
