# kiro-config — Luther's Personal AI-Assisted Development Workflow

**Source:** https://github.com/LutherCalvinRiggs/kiro-config
**Saved:** 2026-06-07
**Tags:** ai, tools, productivity, orchestration, prompting, infrastructure

> Personal note: This is my own Kiro configuration — the optimized pet model I currently use to ship MAC.BID (online auction platform) solo. Reading these notes in the context of gstack and NEEDLE is the point.

---

## TL;DR
A complete AI-assisted development workflow for a solo developer managing 6 repos (API, nextwebsite, nextpist, mobile-app, pist, infra) for MAC.BID. Built on Kiro. Uses a file-based memory system, specialist agent roles, worktree-per-feature parallelism, and structured skill definitions for every workflow phase. The developer makes all architectural decisions; agents plan, execute, track state, and hand off.

## What MAC.BID Is
Online auction platform for discounted overstock products (mac.bid / macdiscount.com). Six repos, solo developer. Key systems: auctions, bidding, payments (Stripe), shipping, AI customer service, fraud detection, marketing attribution, ad platform conversions, Firebase, WebSockets.

**Tech stack:**
- API: Node.js + Serverless Framework + AWS Lambda + MySQL (RDS) + SQS + DynamoDB
- nextwebsite: Next.js (Pages Router) + Vercel + Stripe
- nextpist: Next.js (Pages Router) + Vercel
- mobile-app: React Native + Expo SDK 52+ + Expo Router + EAS
- pist: PHP 7+ CodeIgniter (legacy admin) + Heroku
- infra: Terraform + AWS us-east-1

## System Architecture

### Three Layers
```
Steering files      →  permanent project constitution (tech, product, git, code standards)
Agent configs       →  specialist roles (orchestrator, api-dev, qa, mobile-dev, infra-dev...)
Skills (SKILL.md)   →  workflow procedures (orchestrate, pickup-memory, handoff, worktree...)
```

**Steering** (`~/.kiro/steering/`) is always-on context injected into every session. It's the constitution — it doesn't change per task.

**Agents** (`agents/*.json`) define specialist roles: what model, what tools, what repos, what prompt. The orchestrator spawns subagents; subagents implement and call `/handoff` when done.

**Skills** (`skills/*/SKILL.md`) are invokable procedures — the "how" for each workflow phase.

### The Orchestrator Pattern
```
Orchestrator (never writes code)
  ├── Plans feature, creates task decomposition
  ├── Spawns api-dev subagent for backend work
  ├── Spawns mobile-dev subagent for app work
  ├── Spawns qa subagent after dev completes
  ├── Acts as approval gate for irreversible actions
  └── Coordinates handoffs between agents
```

The orchestrator's constraint — "You never implement code yourself" — is the key architectural decision. It keeps the planner separate from the executor, solving self-preferential bias before it starts.

## The File-Based Memory System

All state in `~/.kiro/memory/`:

```
~/.kiro/memory/
├── features/
│   └── <feature-name>/
│       ├── active-work.md         ← Current state (overwritten on each update, max 20 lines)
│       ├── plan.md                ← Implementation checklist with completion status
│       ├── decisions.md           ← Technical decisions (append-only)
│       └── implementation-notes.md ← Gotchas, patterns discovered (append-only)
├── archive/
│   └── <date>-<feature-name>.md  ← Completed features
├── cross-repo-active-work.md      ← Multi-repo feature tracking
└── feature-work-log.md            ← Chronological log (append-only)
```

**Design principles:**
- `active-work.md` is overwritten (current state, not history) — max 20 lines
- `decisions.md` and `implementation-notes.md` are append-only (institutional memory)
- `feature-work-log.md` is append-only (audit trail)
- Memory is the source of truth. Always read before starting; always write after finishing.

### active-work.md Format
```markdown
**Feature:** <name>
**Status:** In Progress | Complete | Blocked
**Last Updated:** <timestamp>
**Repo(s):** <repos touched>
**Worktree(s):** <full paths — the working directory for all commands>
**Branch(es):** <current branch per worktree>

## What Was Done
<bullets>

## Next Steps
<what to do next>

## Blockers
<or "None">
```

## The Skills (15 active)

### Workflow Control
- **`/orchestrate <feature>`** — Full feature lifecycle: Planning → Building → Completion. One task at a time. Explicit phase gates ("Plan ready. 'Approved' to start"). Reads/writes memory throughout.
- **`/pickup-memory <feature>`** — Read feature state and orient. Uses worktree path from memory as working directory. Does NOT start work — orients and waits.
- **`/update-memory <feature>`** — Save progress to memory files after completing work or before ending session.
- **`/handoff <feature>`** — Called by agents when they finish a task. Writes structured state to memory for the next agent or session to pick up.
- **`/status`** — Cross-repo status report. Read-only. Reads all memory files and presents summary table.

### Dev & Infrastructure
- **`/api-dev`** — API development reference. Points to four reference files: business rules, core safety patterns, dev standards, Lambda invocation guide.
- **`/infra-dev`** — Infrastructure (Terraform) reference and patterns.
- **`/mobile-dev`** — React Native + Expo development reference.
- **`/worktree`** — Git worktree management across all 6 repos simultaneously.
- **`/migration`** — Database migration creation with rollback scripts.
- **`/deploy`** — Guided deployment with pre-deploy checks and explicit confirmation gates.

### QA & Review
- **`/run-tests`** — Run tests for current repo.
- **`/debug`** — Scoped bug fix. Surgical: fix only the reported issue, nothing else. Includes post-debug evaluation.
- **`/eval`** — Post-debug evaluation: determine if standards, skills, or tests need updating based on what went wrong.
- **`/create-pr`** — Structured PR creation targeting staging. Always requires confirmation before pushing.
- **`/search-cloudwatch`** — Search Lambda CloudWatch logs for debugging.

### Memory Lifecycle
- **`/archive <feature>`** — Archive completed feature: create summary, integrate docs, remove feature directory, update work log.

## The Worktree System

One of the most sophisticated parts of the config. Worktrees enable parallel feature development without branch switching.

```
~/Mac_Code/
├── API/                ← main worktree (stays on staging, no feature work here)
├── API--lato/          ← feature worktree (feature persists here for its full lifecycle)
├── API--google-ads/    ← feature worktree
├── infra/
├── infra--lato/
├── nextwebsite/
├── nextwebsite--lato/
...
```

**Key conventions:**
- Naming: `<repo>--<feature>` (double-dash separator)
- Created/removed across ALL 6 repos together unless told otherwise
- One feature per worktree; multiple branches may cycle through it
- The worktree path in `active-work.md` is the working directory for all file operations
- Never auto-remove — only on explicit user command
- Cross-repo features use the same suffix across all repos

```bash
# Create across all repos for a feature
git -C ~/Mac_Code/API worktree add ~/Mac_Code/API--feature luther/feature
npm install  # in each new worktree

# The worktree path is tracked in memory — agents always work in the worktree, never main
```

## The Feature Lifecycle

```
pickup-memory → orient → wait

orchestrate → Plan → [human approves] → Build (one task) → Test → [human: Continue/Pause]
                                                                    ↓
                                                             Completion → handoff → create-pr
                                                                              ↓
                                                                    [human approves] → deploy
```

Every phase gate requires explicit human approval. The agent presents and waits; it never decides to proceed autonomously.

## Steering Files (The Constitution)

Five always-on steering documents injected into every session:

| File | Contents |
|------|---------|
| `tech.md` | Full stack spec: runtimes, frameworks, ports, infra patterns, CRITICAL notes |
| `product.md` | What MAC.BID is, all repos and their roles, key systems, team (solo) |
| `code-standards.md` | DB access patterns, handler logging requirements, validation rules, security rules, testing philosophy, mock strategy |
| `git-workflow.md` | Branch naming, commit discipline, worktree conventions, all 6 repo paths |
| `documentation.md` | When/how to create docs, file naming, content structure |

These are the Rules Block and Context Block from the Claude Projects blueprint — but at the project level, not the session level. They're always available without being pasted into each prompt.

## Key Design Rules (13)

1. Read before writing — always read existing code before implementing
2. One task at a time — work through plan sequentially
3. Strict scope — don't fix unrelated issues, don't add unrequested features
4. Developer decides — present plans, wait for approval, never deploy without confirmation
5. Memory is truth — always update after completing, always read before starting
6. Worktree discipline — never do feature work in main worktree
7. Test immediately — write and run tests right after implementing, not as afterthought
8. Minimal fixes — debug fixes are surgical; note other issues but don't fix them
9. Confirm destructive actions — force pushes, production deploys, data deletion always require confirmation
10. Track everything — every session leaves a trail in memory the next session can pick up
11. Permissive validation — skip invalid optional fields rather than rejecting the entire request
12. Server-side computation — priority, sentiment, classification computed from data, never set by caller
13. Post-debug evaluation — after every bug fix, determine if standards or skills need updating

## Current Active Work (as of last commit)

From `memory/cross-repo-active-work.md`:
- **Google Ads Data Manager API Migration** — migrated from deprecated UploadClickConversions to Data Manager API. Status: Implementation Complete, ready for deployment. Deadline: June 15, 2026.

## Questions & Gaps
- How does this system interact with NEEDLE? The orchestrator pattern here (plan → spawn subagents → collect handoffs) is architecturally similar to NEEDLE's state machine but implemented at the session level rather than the queue level.
- The memory system uses flat files — at what scale does this become unwieldy? gstack's gbrain uses a database (Supabase/PGLite) for the same purpose.
- The `/eval` skill (post-debug evaluation to update standards) is a self-improving loop — does this ever create conflicting rules or standards drift over time?
- The orchestrator "never writes code" rule — what happens when a task crosses the orchestrator/subagent boundary in unclear ways?
- No spend governance layer mentioned — with Kiro, is API cost managed at the IDE level or separately?

## Related Notes
- [gstack Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/gstack-repo-overview.md) — gstack's skills are similar in structure but domain-agnostic; kiro-config is project-specific. gstack adds browser automation, design tools, and security auditing. kiro-config adds deep product context, business rules, and multi-repo worktree coordination.
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — NEEDLE's bead queue is the cattle-model equivalent of this workflow's orchestrate/handoff cycle. The memory system here maps to `.beads/learnings.md` + bead history.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — The CLAUDE.md in this config is what NEEDLE's implementation guide calls the most important document you'll write.
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — `plan.md` in the memory system is exactly the "plan as the negative" Jed describes. The feature checklist is the bead hierarchy before beads.
- [Claude Projects 6-Part Blueprint](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/claude-projects-6-part-blueprint.md) — The steering files here are the engineered version of the Rules Block + Output Format Block. Skills are the Process Block. Agent configs are the Identity Block.
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — The orchestrator/subagent separation implements this principle architecturally: the orchestrator reviews what subagents produce without having written it.
