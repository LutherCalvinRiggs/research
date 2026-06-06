# Claude Code Dynamic Workflows: 6 Patterns and 14 Steps

**Source:** https://x.com/0xcodez/status/2062127385923776831
**Saved:** 2026-06-05
**Tags:** ai, tools, prompting, orchestration, agentic-ai, claude-code

---

## TL;DR
Dynamic Workflows (shipped May 28, 2026) let Claude write its own orchestration harness on the fly — a JavaScript file that spawns and coordinates subagents, picks their models, and controls their isolation level. Six composable patterns (classify-and-act, fan-out-and-synthesize, adversarial verification, generate-and-filter, tournament, loop-until-done) replace fifty chained prompts and structurally prevent the three main failure modes of single-context work.

## Key Concepts & Terms
- **Dynamic Workflow**: Claude writing a custom JavaScript harness for a specific task. Contains special functions (`agent()`, `parallel()`, `pipeline()`) plus standard JS to process data between subagents. Triggered by saying "make a workflow that…" or the word `ultracode`.
- **Static workflow**: A generic harness written once to cover all edge cases (Claude Agent SDK, `claude -p`). Conservative by necessity — it can't know your specific codebase.
- **`agent(prompt, { model, schema })`**: Spawns a single subagent with a focused goal in its own isolated context window.
- **`parallel(fns)`**: Barrier — fans out all agents simultaneously, waits for every result before returning. Use when you need all outputs before the next step.
- **`pipeline(items, stages)`**: Streaming — each item flows through all stages independently without waiting for others. Cheaper and faster when results don't depend on each other.
- **Per-agent model choice**: The workflow assigns models per subagent — Opus for hard reasoning, Haiku for cheap exploration, Sonnet for the middle. Cost is controlled at task-type granularity, not session granularity.
- **Worktree isolation**: Subagent gets its own git checkout. Remote isolation: no checkout. The workflow decides which each agent needs.
- **`/goal`**: Sets a hard completion requirement on a loop pattern. Without it, the workflow stops at the first soft completion point.
- **`/loop`**: Runs the entire workflow on a recurring schedule. Used for continuous triage, weekly research updates, ongoing verification.
- **Quarantine pattern**: Agents that read untrusted external content (user tickets, scraped data, third-party APIs) are barred from high-privilege actions. Separate actor agents — with no exposure to raw content — perform actions. Eliminates prompt injection as an attack surface.

## The Three Failure Modes Workflows Solve

These are named directly in Anthropic's launch writing:

- **Agentic laziness**: Claude stops after partial progress and declares done. Addresses 20 of 50 security items and calls the rest "handled." Fixed by per-agent isolated goals — each agent has one definition of done.
- **Self-preferential bias**: Claude favors its own results when asked to verify them. A verifier with skin in the game can't be a fair verifier. Fixed by adversarial verification — the verifier never sees who produced the work.
- **Goal drift**: Gradual loss of fidelity to the original objective across many turns, especially after context compaction. "Don't do X" constraints quietly disappear at turn 47. Fixed by fan-out — each agent gets the original objective fresh, not a summarized version.

## The Six Patterns

### 1. Classify-and-Act
A classifier agent reads the task first, determines type/complexity, then routes to different agents or models. Use when: task is heterogeneous; you want to spend Opus only where complexity warrants it; the decomposition itself is non-trivial.

### 2. Fan-Out-and-Synthesize
Split into enumerable work items → run one agent per item in parallel → synthesize results at a barrier. Use when: clearly enumerable list (50 files, 200 endpoints); items are independent; you want one consolidated answer. This is the structural fix for goal drift.

```javascript
const reviews = await parallel(
  files.map(file => () => agent(
    `Review ${file} for security issues`,
    { model: "haiku", schema: IssueList }
  ))
)
const report = await agent(
  `Merge these reviews into one prioritized report:\n${JSON.stringify(reviews)}`,
  { model: "opus" }
)
```

### 3. Adversarial Verification
For each producer agent, spawn a separate verifier agent that checks its output against a rubric without knowing who produced it. The verifier knows only the rubric and the artifact. Use for: claim-checking, code review, quality gates before shipping. This is the structural fix for self-preferential bias.

### 4. Generate-and-Filter
Generate N options → filter by rubric → dedupe → return only the highest-quality survivors. Makes Claude commit late, after every option has been challenged. Use for brainstorming (30 names → 3), hypothesis generation, solution design.

### 5. Tournament
Spawn N agents on the same task using different approaches → judge pairwise until one wins. Comparative judgment is more reliable than absolute scoring, especially for taste-based work. Use for sorting 1,000+ items (tournament beats sort-by-score — each comparison is fast, fair, isolated, and the bracket lives in deterministic code not in context).

### 6. Loop Until Done
Loop spawning agents until a stop condition is met — zero new findings, zero errors, theory verified — instead of running a fixed number of passes. Pair with `/goal` to set the hard completion requirement. Use for: flaky test debugging, bug hunting, pattern mining.

## Composing Patterns for Real Use Cases

Patterns rarely appear alone. Real workflows compose 2–4:

| Use Case | Patterns |
|----------|---------|
| Migration / refactor | Fan-out (one agent per callsite in a worktree) → adversarial verification → loop until done |
| Deep research | Fan-out (parallel web searches) → adversarial verification (each claim) → synthesize |
| Sorting 1,000+ items | Tournament (pairwise, never absolute scores) |
| Root-cause investigation | Fan-out (different agents read logs, files, data) → panel of verifiers/refuters per theory → loop until one survives |
| Triage at scale | Classify-and-act → dedupe → fix or escalate → `/loop` for continuous operation |
| Exploration / taste | Generate-and-filter → tournament with rubric |
| Lightweight evals | Run candidate in worktree → comparison agents grade against rubric → refine and re-grade |

Mapping failure modes to patterns: Drift → fan-out. Self-preference → adversarial verification. Open-ended work → loop until done. Hard-to-score → tournament.

## Cost Controls

Workflows can balloon to 5–10× expected tokens without explicit controls:

- **Token budget in the prompt**: "Use 10k tokens." Sets a cap on the run.
- **`/goal`**: Forces hard completion rather than stopping at the first soft completion point.
- **`/loop`**: Recurring schedule. Don't rebuild the same workflow weekly.
- **Model assignment per agent**: Haiku for cheap exploration, Opus only where complexity demands it.

```
> ultracode quick adversarial review of this assumption:
  "moving to Postgres eliminates our shard rebalancing."
  Use 5k tokens. /goal don't stop until you have either
  a counterexample or three independent confirmations.
```

## Saving and Shipping Workflows

- Press `s` in the workflow menu to save. Goes to `~/.claude/workflows`.
- Bundle the JS file in a Skill folder, reference in `SKILL.md` → anyone who installs the Skill runs the same workflow.
- When packaging as a Skill, tell Claude to treat the workflow as a template, not a script — leaves room to adapt the shape per task while keeping the structure intact.

## Common Mistakes

- Reaching for a workflow when a 5-minute Claude Code session would do
- No token budget — workflows balloon without an explicit cap
- One agent doing both the work and verification — self-preferential bias
- Treating `parallel()` and `pipeline()` as interchangeable — the barrier matters
- Skipping `/goal` on loop patterns — stops at first soft completion
- Letting untrusted content reach actor agents — quarantine isn't optional
- Sorting with absolute scores instead of tournament/pairwise
- Not saving working workflows — re-prompting the same shape every week

## Questions & Gaps
- How does Dynamic Workflow state persistence work across terminal restarts? The article says resuming picks up where it left off — what's the storage mechanism?
- What are the token cost characteristics of `parallel()` vs `pipeline()` at scale? The article frames pipeline as "cheaper overall" but doesn't quantify.
- The quarantine pattern — how strictly is the separation enforced at the API level? Can a quarantined agent escalate to actor-level permissions through a prompt?
- Skills packaging: does "treat as template" require a specific prompt pattern or is it a SKILL.md directive?
- No mention of how workflows interact with NEEDLE's bead-level orchestration — could a NEEDLE worker spawn a Dynamic Workflow as part of a bead execution?

## Related Notes
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — Anthropic's own generator-evaluator multi-agent architecture is the direct precursor to adversarial verification pattern; Dynamic Workflows formalize and democratize what that post described building by hand.
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — the adversarial verification pattern is this principle implemented architecturally rather than as a workflow discipline.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — NEEDLE's outcome table and Dynamic Workflows' pattern library are solving the same underlying problem from different angles: how do you make non-deterministic agents produce reliable, structured results?
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — Dynamic Workflows is Claude Code's native cattle mode: stateless subagents, parallel execution, orchestrator controls the fleet.
- [Agent Platform Security Checklist](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/agent-platform-security-checklist.md) — the quarantine pattern directly maps to the checklist's "separate thinking environment from acting environment" principle.
- [I Stopped Hitting Claude's Usage Limits](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/stopped-hitting-claude-usage-limits-10-things.md) — token budget controls in workflows are the fleet-scale version of the individual session habits described here.
