# ai-harness Repo Overview — Claude Code Harness Template

**Source:** https://github.com/LutherCalvinRiggs/ai-harness
**Owner:** LutherCalvinRiggs
**Status:** Active primary harness (as of July 2026). Used across two projects. NEEDLE placed on shelf.
**Saved:** 2026-07-31
**Tags:** ai, tools, infrastructure, orchestration, agentic-ai, productivity

> This is the canonical harness repo for all current AI-assisted development. Migrated from a Kiro user-scoped config (`~/.kiro/`) to a per-project Claude Code template on 2026-07-11. GitHub template repo — bootstrapped into new projects via `/new-project`.

---

## TL;DR
A per-project Claude Code harness template with 19 skills, 7 subagent definitions, 6 steering docs, and a file-based memory system. The orchestrator/subagent split is the core architecture: the orchestrator (interactive parent session) plans, gates, and handles irreversible operations; subagents (spawned in feature worktrees) implement code with forbidden access to git, network, deploys, and databases. `/eval` closes the loop from bug discovery back to updated steering docs.

---

## Repo Structure

```
ai-harness/
├── CLAUDE.md                    ← Root context file; imports all steering docs
├── .mcp.json                    ← Pinned MCP servers (playwright, context7, aws-docs)
├── README.md
├── .claude/
│   ├── steering/                ← Always-resident context (L1)
│   │   ├── code-standards.md
│   │   ├── git-workflow.md
│   │   ├── documentation.md
│   │   ├── subagent-guardrails.md
│   │   ├── product.md           ← BLANK: fill in per project
│   │   └── tech.md              ← BLANK: fill in per project
│   ├── skills/                  ← 19 skills (invoked as /commands)
│   ├── agents/                  ← 7 subagent definitions
│   ├── memory/                  ← Persistent work state
│   │   ├── workflow/            ← Harness-level decisions
│   │   ├── features/            ← Per-feature memory
│   │   ├── archive/             ← Completed features
│   │   └── cross-repo-active-work.md
│   └── research/                ← Data files, CSVs for /analyze-csv
└── docs/                        ← Project docs (not auto-loaded into context)
```

---

## The 19 Skills

| Skill | Purpose |
|-------|---------|
| `/orchestrate` | Full feature lifecycle: Plan → Billing Estimate → Build → Complete |
| `/handoff` | Record completed work to memory for session continuity |
| `/pickup-memory` | Orient on current work state at session start |
| `/status` | Current feature status summary |
| `/debug` | Analyze and fix a specific bug (reads clipboard via pbpaste) |
| `/eval` | Post-debug: determine what steering/skill to update to prevent recurrence |
| `/create-pr` | PR creation workflow |
| `/deploy` | Deployment workflow (project-specific commands) |
| `/run-tests` | Test runner (project-specific commands) |
| `/worktree` | Manage git worktrees for parallel feature development |
| `/new-project` | Bootstrap a new project from this template |
| `/archive` | Move completed feature memory to archive |
| `/update-memory` | Manual memory update |
| `/estimate-billing` | Estimate time for human vs. AI-assisted development |
| `/whiteboard` | Architecture diagram generation |
| `/analyze-csv` | CSV analysis skill |
| `/search-cloudwatch` | CloudWatch log search |
| `/api-dev` | API development with references to core-safety and dev-standards |
| `/infra-dev` | Infrastructure development |
| `/pdf` | PDF form filling with scripts and references |

---

## The 7 Subagents

| Agent | Role | Key Constraint |
|-------|------|----------------|
| `orchestrator` | Cross-repo coordinator, spawns subagents, manages approval gates, never writes code | Reads guardrails; enforces them on all spawned agents |
| `backend-dev` | Server-side APIs, business logic, database queries, tests | In feature worktree only |
| `frontend-dev` | UI implementation | In feature worktree only |
| `mobile-dev` | Mobile app implementation | In feature worktree only |
| `infra-dev` | Infrastructure work | In feature worktree only |
| `migrations` | Database migrations | In feature worktree only |
| `qa` | Test writing and execution | In feature worktree only; spawned by orchestrator after dev work |

---

## The Orchestrator/Subagent Split (Core Architecture)

**Parent (orchestrator, interactive session):**
- Plans features, manages phase gates
- Handles all git operations (add, commit, push, PR, merge)
- Handles deploys, dependency installs, network requests
- Approval gate for irreversible actions
- Never implements code

**Subagents (spawned, unsupervised in feature worktrees):**
- Implement code within their assigned worktree
- Read/write `.claude/memory/`, `docs/`
- Read other worktrees for reference (patterns, shared code)
- Call `/handoff` when complete

**Subagent forbidden actions (hard list):**
- All git operations
- All network requests
- Package installs or manifest modifications
- Cloud CLI create/modify/delete/deploy
- Database connections or SQL
- Infrastructure config changes
- CI/CD workflow modifications
- Dev server start commands
- Writing outside the assigned worktree

**When blocked:** Stop → document what's needed → report to parent. Never improvise.

---

## The Memory System

All state in `.claude/memory/`:

```
features/<feature-name>/
  active-work.md          ← Current status (overwritten each update, <20 lines)
  plan.md                 ← Implementation checklist + billing estimate
  decisions.md            ← Technical decisions (append-only)
  implementation-notes.md ← Gotchas, patterns

workflow/
  active-work.md          ← Harness-level current state
  decisions.md            ← Harness-level decisions
  implementation-notes.md ← Harness-level notes

archive/                  ← Completed features (timestamped)
cross-repo-active-work.md ← Cross-repo feature state
feature-work-log.md       ← Chronological log (append-only table)
```

**Key memory rules:**
- `active-work.md` is always overwritten (current state, not history)
- `feature-work-log.md` and `decisions.md` are always appended
- `/handoff` writes; `/pickup-memory` reads; `/archive` moves completed work

---

## The `/orchestrate` Feature Lifecycle

```
Phase 1: Planning
  Analyze requirement → read existing code patterns → create plan.md checklist
  → architecture whiteboard (if new endpoints/services/patterns)
  → flag risks → "Plan ready. Approved to start?"

Phase 1.5: Billing Estimate
  Run /estimate-billing against plan.md tasks
  → append to plan.md under ## Billing Estimate
  → "Estimate recorded. Continue to start building?"

Phase 2: Building
  Read next unchecked task from plan.md
  → read reference files → implement → write tests immediately
  → run tests → check off task → update active-work.md
  → "Task complete. N tests passing. Continue for next?"

Phase 3: Completion
  Call /handoff with full summary
  → present files changed, test results, suggested commit
  → "Ready to commit?"
```

---

## The `/eval` Closed Loop

The most architecturally significant skill. After a bug is fixed:

1. Analyze: what was the bug, what category, was it preventable with existing standards?
2. Determine target: code-standards.md, CLAUDE.md, run-tests skill, orchestrate skill, or implementation-notes
3. Draft the update (directive, not narrative; 1–3 lines)
4. Present for approval → apply

This is the **CLAUDE.md closed loop** from the Anthropic AI-Native SDLC Security note: when a bug class is discovered, update the steering file so the bug class doesn't get generated again. `/eval` is the operationalization of that principle.

---

## The Billing Estimate Skill

Generates two estimates per feature:
- **Human (no AI):** Senior dev working solo, includes context-switching, docs-reading, boilerplate, testing, deployment
- **AI-assisted:** The orchestration workflow with subagents

Multipliers: multi-repo (+20% per additional repo), new external integration (+30%), DB migration (+1–2h), infra changes (+2–4h).

Integrated into `/orchestrate` Phase 1.5 — runs automatically after plan approval before building starts.

---

## The `/eval`-to-Steering Update Pipeline

| Bug category | Update target |
|-------------|--------------|
| Pattern violation that should be global | `.claude/steering/code-standards.md` |
| Repo-specific gotcha | `<repo>/CLAUDE.md` |
| Missing test coverage pattern | `/run-tests` skill or repo steering |
| Process gap (missed step) | Relevant skill (orchestrate, deploy, etc.) |
| Detectable automatically | Propose a hook (PreToolUse/PostToolUse) |
| Feature-specific gotcha | `.claude/memory/features/<name>/implementation-notes.md` |

---

## Key Architecture Decisions (from archive)

**Template model over hub-repo or user-scope:** Three options considered — (1) hub repo as global cross-repo orchestrator (like ~/.kiro/); (2) user-scoped install at ~/.claude/; (3) per-project template. Chose (3): each project self-contained, no risk of harness files mixing with source code.

**MCP version pinning:** `.mcp.json` pins exact versions for playwright, context7, aws-docs. Never @latest — supply-chain risk. Bump deliberately.

**Memory and research under `.claude/`:** Not at repo root — keeps harness state out of the project's file namespace.

**NEEDLE deferred:** Active-work.md notes: "Revisit NEEDLE if hitting persistence/scale/retry ceiling." Shelf decision is conditional, not permanent.

---

## Questions & Gaps

- The worktree skill has a blank repo table: "update this table to match your project's repos." In practice, is this filled in per-project or left for the orchestrator to infer?
- Cross-repo features use `cross-repo-active-work.md` — but the orchestrator also tracks per-repo state in individual feature directories. Is there a risk of state drift between the two?
- The QA agent is spawned after dev work completes (per orchestrator.md) — but tests are also written immediately during Phase 2 Building (per orchestrate/SKILL.md). Are there two testing layers or is one the canonical?
- The subagent guardrails forbid database connections — but some features require verifying schema or querying test data. How does this get handled? (Presumably via the parent orchestrator session.)

## Related Notes
- [kiro-config Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/kiro-config-repo-overview.md) — the predecessor. ai-harness is the migration of kiro-config from Kiro user-scope to per-project Claude Code native format.
- [Jed Arden Workflow Manual](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/jed-arden-workflow-manual.md) — the workflow that ai-harness partially implements. Stages 02 (scaffold), 04 (plan file), 05 (review), and the memory system all have direct equivalents.
- [Lilian Weng: Harness Engineering for Self-Improvement](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/lilian-weng-harness-engineering-self-improvement.md) — the academic framing. ai-harness implements Weng's Pattern 1 (workflow automation), Pattern 2 (file system as persistent memory), and Pattern 3 (sub-agent and backend jobs).
- [Anthropic AI-Native SDLC Security](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/anthropic-ai-native-sdlc-security.md) — the `/eval` → steering-update loop is exactly the "CLAUDE.md closed loop" described there.
- [Jamon Holmgren 18-Point Agentic Setup](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/jamon-holmgren-18-point-agentic-setup.md) — the 18-point scorecard reference.
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — shelved for now; revisit if hitting persistence/scale/retry ceiling.
