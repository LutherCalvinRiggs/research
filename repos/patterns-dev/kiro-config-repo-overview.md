# kiro-config — Luther's Personal AI-Assisted Development Workflow

**Source:** https://github.com/LutherCalvinRiggs/kiro-config
**Saved:** 2026-06-07
**Updated:** 2026-06-18
**Tags:** ai, tools, productivity, orchestration, prompting, infrastructure

> Personal note: This is my own Kiro configuration — the optimized pet model I currently use to ship MAC.BID (online auction platform) solo. Reading these notes in the context of gstack and NEEDLE is the point.

---

## TL;DR
A complete AI-assisted development workflow for a solo developer managing 6 repos (API, nextwebsite, nextpist, mobile-app, pist, infra) for MAC.BID. Built on Kiro. Uses a file-based memory system, specialist agent roles, worktree-per-feature parallelism, and structured skill definitions for every workflow phase. The developer makes all architectural decisions; agents plan, execute, track state, and hand off. As of June 2026: now includes subagent guardrails (two-tier permission model) and a billing estimator integrated into the orchestrate flow.

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
  ├── Runs billing estimate after plan approval
  ├── Spawns api-dev subagent for backend work
  ├── Spawns mobile-dev subagent for app work
  ├── Spawns qa subagent after dev completes
  ├── Acts as approval gate for irreversible actions
  └── Coordinates handoffs between agents
```

The orchestrator's constraint — "You never implement code yourself" — is the key architectural decision. It keeps the planner separate from the executor, solving self-preferential bias before it starts.

## Two-Tier Agent Permission Model (Added June 2026)

As of June 17, a `subagent-guardrails.md` steering file establishes a strict permission split between the parent (interactive) session and any spawned subagents.

**Parent agent** — full access. Handles git, deploys, PRs, network requests, package installs, AWS writes. No restrictions.

**Subagents** (unsupervised workers in feature worktrees) — heavily restricted:

| Category | Allowed | Forbidden |
|----------|---------|-----------|
| Files | Read/write inside assigned feature worktree; read other worktrees for reference | Write outside assigned worktree |
| Shell | `npm test`, `npm run lint`, read-only utils (`grep`, `find`, `cat`, `diff`) | Any flag with `-rf`, `--force`, `--delete`, `--hard` |
| Git | None | Everything — `add`, `commit`, `push`, `reset`, `rebase` |
| Network | None | `curl`, `wget`, any outbound request |
| Dependencies | None | `npm install`, `yarn add`, any package modification |
| AWS | CloudWatch log reads only | Any create/modify/delete/deploy |
| Database | None | Any SQL, any connection |
| Infrastructure | None | `.tf` files, `.github/workflows/` |

**When a subagent hits a forbidden action it needs:** stop, document what's required in its output (e.g. `"Requires: npm install uuid"`), continue with remaining work. Parent reviews and performs the blocked actions after subagent execution completes.

This is the same human-gate principle as NEEDLE's Production Safety Guide — applied at the subagent/parent boundary rather than the worker/infrastructure boundary.

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

## The Skills (updated June 2026)

### Workflow Control
- **`/orchestrate <feature>`** — Full feature lifecycle: Planning → Billing Estimate → Building → Completion. One task at a time. Explicit phase gates. Reads/writes memory throughout. Now includes Phase 1.5 (billing estimate) automatically after plan approval.
- **`/pickup-memory <feature>`** — Read feature state and orient. Uses worktree path from memory as working directory. Does NOT start work — orients and waits.
- **`/update-memory <feature>`** — Save progress to memory files after completing work or before ending session.
- **`/handoff <feature>`** — Called by agents when they finish a task. Writes structured state to memory for the next agent or session to pick up.
- **`/status`** — Cross-repo status report. Read-only. Reads all memory files and presents summary table.
- **`/estimate-billing <feature|description>`** *(new June 2026)* — Estimates work in two tracks: human (no AI) hours with per-category multipliers, and AI-assisted hours with reduction factors by task type. Integrates into `/orchestrate` automatically after plan approval; callable standalone for proposals or sprint planning.

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

### External Skills (symlinked from PatternsDev)
- `agent-browser`, `building-native-ui`, `context7`, `frontend-design`, `mcp-builder`, `skill-creator`, `stripe-best-practices`, `theme-factory`, `upgrading-expo`, `vercel-react-best-practices`

## The /estimate-billing Skill (Detail)

**Step 1 — Decompose:** Break work into tasks by category (Planning, Backend, Frontend, Mobile, Infra, Migration, Testing, Deployment), complexity (Trivial → Very Complex), dependencies, and repos touched.

**Step 2 — Human estimate:** Senior dev, no AI. Includes docs reading, debugging, boilerplate, context switching. Multipliers: +20% per additional repo, +30% for new external integrations, +1–2h for migrations, +2–4h for infra.

**Step 3 — AI-assisted estimate:** Typical reduction factors:
- Boilerplate/CRUD: 70–80% reduction
- Business logic with clear spec: 50–60% reduction
- Complex debugging: 20–30% reduction
- Architecture decisions: 10–20% reduction
- Manual testing/verification: 0% reduction
- Deployment/infrastructure: 10–20% reduction

**Output format:**
```
📊 Estimate: [Feature Title]
┌─────────────────────────────────────────────────┐
│ 🧑‍💻 Human (no AI):     X–Y hours                │
│ 🤖 AI-Assisted:        X–Y hours                │
│ ⚡ Time Saved:          ~N% (X–Y hours)          │
└─────────────────────────────────────────────────┘
+ per-task table, assumptions, risks
```

Integrated into `/orchestrate` as Phase 1.5 — runs after plan approval, appends to `plan.md` under `## Billing Estimate`, then prompts to start building.

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
- After creation: run `npm install` and copy `.env` files from main worktree

```bash
# Create across all repos for a feature
git -C ~/Mac_Code/API worktree add ~/Mac_Code/API--feature luther/feature
npm install  # in each new worktree

# The worktree path is tracked in memory — agents always work in the worktree, never main
```

## The Feature Lifecycle (Updated)

```
pickup-memory → orient → wait

orchestrate → Plan → [human approves]
                          ↓
                   Billing Estimate → [human: Continue/Revise]
                          ↓
                   Build (one task) → Test → [human: Continue/Pause]
                          ↓
                   Completion → handoff → create-pr
                                              ↓
                                    [human approves] → deploy
```

Every phase gate requires explicit human approval. The agent presents and waits; it never decides to proceed autonomously.

## Steering Files (The Constitution)

Six always-on steering documents injected into every session:

| File | Contents |
|------|---------|
| `tech.md` | Full stack spec: runtimes, frameworks, ports, infra patterns, CRITICAL notes |
| `product.md` | What MAC.BID is, all repos and their roles, key systems, team (solo) |
| `code-standards.md` | DB access patterns, handler logging requirements, validation rules, security rules, testing philosophy, mock strategy |
| `git-workflow.md` | Branch naming, commit discipline, worktree conventions, all 6 repo paths. Updated June 2026: rebase onto staging before force-pushing. |
| `documentation.md` | When/how to create docs, file naming, content structure |
| `subagent-guardrails.md` | *(new June 2026)* Two-tier permission model for spawned subagents vs. parent session |

## Setting Up on a New Machine

The canonical setup reference lives at `docs/ai-assisted-development-workflow.md` in the repo — it's AI-agnostic and covers every step. Summary:

1. **Clone all project repos** into a shared code directory
2. **Install language runtimes** required by the project
3. **Install dependencies** in each repo (`npm install` or equivalent)
4. **Configure cloud CLI tools** with appropriate credentials
5. **Install VCS CLI** (`gh` for GitHub) for PR creation
6. **Create memory directory structure:**
   ```bash
   mkdir -p ~/.agent/memory/features
   mkdir -p ~/.agent/memory/archive
   touch ~/.agent/memory/feature-work-log.md
   touch ~/.agent/memory/cross-repo-active-work.md
   mkdir -p ~/.agent/docs/
   ```
7. **Set up steering files** in `~/.kiro/steering/` — tech.md, product.md, code-standards.md, git-workflow.md, documentation.md, subagent-guardrails.md
8. **Set up skill definitions** — clone kiro-config and point Kiro at the skills directory
9. **Configure git** with developer identity and push defaults
10. **Configure agent** with filesystem read/write, shell execution, cloud CLIs, memory directory access, and steering file references

> Note: the memory path in the workflow doc uses `~/.agent/memory/` as a generic placeholder. The actual kiro-config uses `~/.kiro/memory/`. Adapt to whatever your agent framework supports.

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

## Questions & Gaps
- How does this system interact with NEEDLE? The orchestrator pattern here (plan → spawn subagents → collect handoffs) is architecturally similar to NEEDLE's state machine but implemented at the session level rather than the queue level.
- The memory system uses flat files — at what scale does this become unwieldy? gstack's gbrain uses a database (Supabase/PGLite) for the same purpose.
- The `/eval` skill (post-debug evaluation to update standards) is a self-improving loop — does this ever create conflicting rules or standards drift over time?
- The orchestrator "never writes code" rule — what happens when a task crosses the orchestrator/subagent boundary in unclear ways?
- No spend governance layer mentioned — with Kiro, is API cost managed at the IDE level or separately? (Note: `/estimate-billing` now provides pre-work cost visibility, but no runtime spend cap equivalent to claude-governor is documented.)
- The subagent guardrails forbid all network access — how do subagents that need to reference external docs or APIs handle that? Presumably parent fetches and pastes into the bead/task body.

## Related Notes
- [gstack Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/gstack-repo-overview.md) — gstack's skills are similar in structure but domain-agnostic; kiro-config is project-specific. gstack adds browser automation, design tools, and security auditing. kiro-config adds deep product context, business rules, and multi-repo worktree coordination.
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — NEEDLE's bead queue is the cattle-model equivalent of this workflow's orchestrate/handoff cycle. The memory system here maps to `.beads/learnings.md` + bead history.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — The CLAUDE.md in this config is what NEEDLE's implementation guide calls the most important document you'll write.
- [NEEDLE Production Safety Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Production-Safety-Guide.md) — The subagent-guardrails.md steering file implements the same human-gate and scope-restriction principles as NEEDLE's Layer 3–4 controls, applied at the intra-session agent boundary.
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — `plan.md` in the memory system is exactly the "plan as the negative" Jed describes. The feature checklist is the bead hierarchy before beads.
- [Claude Projects 6-Part Blueprint](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/claude-projects-6-part-blueprint.md) — The steering files here are the engineered version of the Rules Block + Output Format Block. Skills are the Process Block. Agent configs are the Identity Block.
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — The orchestrator/subagent separation implements this principle architecturally: the orchestrator reviews what subagents produce without having written it.
