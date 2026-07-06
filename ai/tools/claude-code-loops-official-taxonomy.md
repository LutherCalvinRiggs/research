# Getting Started with Loops — Official Claude Code Team Guide

**Source:** https://x.com/ClaudeDevs/status/2063992615721435154 (pasted content by @ClaudeDevs / @delba_oliveira, Anthropic)
**Saved:** 2026-06-27
**Tags:** ai, tools, prompting, orchestration, agentic-ai, claude-code

> **Primary source.** Written by the Claude Code team (@delba_oliveira). This is Anthropic's own official taxonomy of loop types, not a third-party synthesis. Read alongside the Osmani and 0xCodez loop engineering notes for the full picture.

---

## TL;DR
The Claude Code team's official definition and taxonomy of loops: agents repeating cycles of work until a stop condition is met. Four loop types categorized by trigger and stop mechanism — turn-based, goal-based, time-based, and proactive. Key principle: not all tasks require complex loops; start with the simplest solution and add complexity only when needed.

## The Official Four-Loop Taxonomy

| Loop type | Triggered by | Stopped by | Best for | Primitive |
|-----------|-------------|-----------|---------|-----------|
| **Turn-based** | User prompt | Claude judges task done | Exploration, short tasks, deciding | Custom verification skills |
| **Goal-based** | Manual prompt | Goal achieved OR max turns | Tasks with verifiable exit criteria | `/goal` |
| **Time-based** | Time interval | You cancel it, or work completes | Recurring work, external system polling | `/loop`, `/schedule` |
| **Proactive** | Event or schedule (no human present) | Each task exits when goal met; routine runs until turned off | Recurring streams of well-defined work | All of the above + dynamic workflows |

---

## Loop 1: Turn-Based Loop

**What it is**: Every prompt you send starts a manual loop. Claude gathers context, acts, checks its work, repeats if needed, responds. You direct each turn — this is the agentic loop.

**Improving the verification step**: Encode manual verification steps as a `SKILL.md` so Claude can check more of its own work end-to-end. The skill should include tools and connectors that let Claude see, measure, or interact with the result. The more quantitative the checks, the easier self-verification is.

**Example skill (UI verification):**
```markdown
---
name: verify-frontend-change
description: Verify any UI change end-to-end before declaring it done.
---

# Verifying frontend changes
Never report a UI change as complete based on a successful edit alone.

1. Start the dev server and open the edited page in the browser.
2. Interact with the change directly — click it, confirm expected state change, screenshot before/after.
3. Check browser console: zero new errors or warnings.
4. Use Chrome Devtools MCP, run performance trace and audit Core Web Vitals.

If any step fails, fix the issue and rerun from step 1.
Do not hand back partially verified work.
```

---

## Loop 2: Goal-Based Loop (`/goal`)

**What it is**: You define what "done" looks like. Claude iterates until the goal is met or a turn cap is reached. An evaluator model checks the goal condition — Claude doesn't self-judge "good enough."

**Why it works**: When "done" is defined externally (not by the agent that did the work), the agent can't prematurely close the loop by declaring partial completion sufficient.

**Deterministic criteria work best**: number of tests passed, Lighthouse score threshold, lint errors = 0. These can be checked mechanically.

```bash
/goal get the homepage Lighthouse score to 90 or above, stop after 5 tries.
```

**Token management**: Set explicit turn caps ("stop after 5 tries") to prevent runaway token usage on hard-to-achieve goals.

---

## Loop 3: Time-Based Loop (`/loop` and `/schedule`)

**What it is**: A prompt that re-runs on an interval. Useful for recurring work or polling external systems.

```bash
/loop 5m check my PR, address review comments, and fix failing CI
```

**`/loop`** runs on your machine — stops when you close the laptop.
**`/schedule`** moves the loop to cloud infrastructure — survives machine off.

**Token management**: Match the interval to how often the thing you're watching actually changes. A 5-minute loop on something that changes hourly wastes 11x the tokens needed.

---

## Loop 4: Proactive Loop (Routines)

**What it is**: Fully autonomous recurring work with no human in real time. Combines `/schedule`, `/goal`, dynamic workflows, and auto mode.

**Full example** (feedback triage routine):
```bash
/schedule every hour: check the project-feedback channel for bug reports.
/goal: don't stop until every report found this run is triaged, actioned, and responded to.
When fixing a bug, use a workflow to explore three solutions in parallel worktrees
and have a judge adversarially review them.
```

This composes: scheduled trigger → goal-based inner loop → dynamic workflows for parallel exploration → adversarial review → auto mode to avoid permission interrupts.

**Model routing for proactive loops**: route recurring/commodity tasks to smaller faster models; use the most capable model only for judgment calls. This is the same cost-optimization principle as NEEDLE's model routing (Fable 5 for orchestration, Sonnet for workers, Haiku for graders).

---

## Maintaining Code Quality in Loops

**Keep the codebase clean**: Claude follows existing patterns and conventions. Messy codebases produce messier loop output.

**Give Claude a way to verify its own work**: Encode "what good looks like" in skills. Without this, Claude can only judge by its own reasoning (self-preferential bias).

**Make docs reachable**: Current library and framework documentation prevents hallucinated API calls.

**Use a second agent for code review**: A reviewer with fresh context is less biased and not influenced by the main agent's reasoning. Built-in `/code-review` skill or Code Review for GitHub.

**Encode lessons, not just fixes**: When an individual result fails, don't only fix it — encode the lesson to improve the system for all future iterations.

---

## Managing Token Usage

**Observability commands** (official, from the Claude Code team):
- `/usage` — breaks down recent usage by skills, subagents, and MCPs
- `/goal` with no arguments — shows number of turns and token usage so far
- `/workflows` — shows each agent's token usage; allows stopping an agent mid-run

**Budget discipline**:
- Smaller tasks don't need multiple agents or loops
- Some tasks can use cheaper/faster models
- Pilot dynamic workflows on a small slice before a large run — workflows can spawn hundreds of agents
- Use scripts for deterministic work — running a script is cheaper than reasoning through steps ("a PDF skill can ship a form-filling script that Claude runs each time, instead of re-deriving the code")
- Don't run routines more often than you need to

---

## The "Hand Off" Mental Model

To choose the right loop type, ask: **what are you handing off?**

| What you hand off | Use |
|---|---|
| The check (you define the verification, Claude does the work) | Turn-based + custom verification skill |
| The stop condition (you know what done looks like) | Goal-based `/goal` |
| The trigger (work happens on a schedule or event) | Time-based `/loop` or `/schedule` |
| The prompt (work is recurring and well-defined) | Proactive routine |

---

## Questions & Gaps
- `/schedule` is described as "research preview" — what's the GA timeline and are there execution count limits comparable to N8N Cloud?
- The evaluator model for `/goal` — is this explicitly configurable (choose Haiku as the evaluator, Sonnet as the maker), or does Claude Code pick the evaluator model automatically?
- "Pilot before a large run" is good advice but there's no guidance on how to instrument a pilot — what metrics to collect, what constitutes a passing pilot before scaling up?
- The article doesn't address what happens when a proactive loop produces bad output that gets merged before human review. What's the recommended rollback/circuit-breaker pattern for proactive loops in production codebases?

## Related Notes
- [Loop Engineering — Addy Osmani (Primary Source)](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-addy-osmani.md) — Osmani's essay describes the same five building blocks from the outside. This note is the Claude Code team's inside view with official primitive names and token management commands.
- [Loop Engineering: 14-Step Roadmap](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-14-step-roadmap.md) — the @0xCodez synthesis maps to this taxonomy: turn-based = basic agentic loop; goal-based = `/goal`; proactive = the full Routines + dynamic workflows pattern.
- [Dynamic Workflows — Anthropic Primary Source](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/dynamic-workflows-anthropic-primary-source.md) — dynamic workflows are the orchestration layer for proactive loops. That note covers the fan-out-and-synthesize, adversarial verification, and tournament patterns that the proactive loop example uses.
- [Claude Fable 5: Self-Improving Agent System](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-fable-5-self-improving-agent-system.md) — Fable 5's four-layer compound stack maps onto this taxonomy: Layer 2 (Orchestration) is the `/goal` + Routines primitives; Layer 3 (Memory) is the state files that persist between loop runs; Layer 4 (Self-improvement) is the "encode lessons, not just fixes" discipline.
- [kiro-config Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/kiro-config-repo-overview.md) — kiro's orchestrator pattern is a manual turn-based loop with the human directing each turn. `/goal` and proactive routines are the natural next step once the task is well enough defined to automate the check and trigger.
