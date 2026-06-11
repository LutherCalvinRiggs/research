# Dynamic Workflows in Claude Code — Anthropic Primary Source

**Source:** https://x.com/trq212/status/2063992615721435154 (Thariq Shihipar, Anthropic technical staff)
**Saved:** 2026-06-11
**Tags:** ai, tools, orchestration, agentic-ai, prompting, claude-code

> **Primary source.** Written by Thariq Shihipar and Sid Bidasaria, Anthropic technical staff who built the feature. A synthesis of this material was previously saved at `ai/tools/claude-code-dynamic-workflows-6-patterns-14-steps.md`. Read this for authoritative framing, the Bun/Zig-to-Rust example, and Anthropic's own "when not to use" guidance.

---

## TL;DR
Dynamic workflows let Claude write its own JavaScript harness on the fly — spawning subagents, choosing models per agent, isolating in worktrees, and resuming if interrupted. They solve three specific failure modes of single-context work (agentic laziness, self-preferential bias, goal drift) that arise on long-running, massively parallel, or adversarial tasks. Written by the Anthropic engineers who built the feature. "Best practices are still developing" — this is an honest early-days caveat from the authors.

## What Makes This Source Distinct

The @0xCodez synthesis is a well-structured 14-step roadmap. This source adds:

- **The authorial caveat**: "Dynamic workflows often use more tokens, so think carefully about when and how to use them." From the people who built it — more credible than a third-party saying the same.
- **The Bun rewrite example**: Bun was rewritten from Zig to Rust using dynamic workflows. The key: break into steps (callsites, failing tests, modules), spin a subagent per fix in a worktree, adversarially review, merge. Explicitly: "tell the agent not to use resource-intensive commands so you can maximally parallelize without running out of resources."
- **The deep-research skill**: Anthropic published `/deep-research` inside Claude Code using dynamic workflows — fans-out web searches, fetches sources, adversarially verifies claims, synthesizes a cited report. This is a reference implementation of the fan-out-and-synthesize pattern.
- **Memory and rule mining (reverse direction)**: "Mine your recent sessions and code review comments for corrections you keep making, cluster with parallel agents, adversarially verify each candidate (would this rule have prevented a real mistake?), and distill the survivors back into a CLAUDE.md." This reverse direction — sessions → CLAUDE.md rules — is not in the synthesis.
- **Model and intelligence routing**: A classifier agent that researches the codebase before deciding which model to route to. "The best model for 'explain how the auth module works' depends on how many files are in the auth module." Dynamic model selection based on actual task complexity.
- **The quarantine pattern (named explicitly)**: "Barring the agents that read untrusted public content from taking high-privilege actions, which are instead done by the agents in charge of acting on the information." Critical for triage workflows processing support tickets, bug reports, or any user-submitted content.

## The Three Failure Modes (Anthropic's Official Framing)

These are the named failure modes from Anthropic's own engineering docs:

1. **Agentic laziness**: Claude stops before finishing a complex multi-part task and declares it done after partial progress. Example: addressing 20 of 50 items in a security review and calling the rest "handled."

2. **Self-preferential bias**: Claude's tendency to prefer its own results or findings when asked to verify or judge them against a rubric. A verifier with skin in the game can't be fair.

3. **Goal drift**: Gradual loss of fidelity to the original objective across many turns, especially after compaction. Each summarization step is lossy. "Don't do X" constraints disappear at turn 47.

## How Dynamic Workflows Work (Technical)

A dynamic workflow executes a JavaScript file with special functions for spawning and coordinating subagents, plus standard JS (JSON, Math, Array) for processing data.

**Key capabilities:**
- Each subagent gets its own context window
- The workflow chooses which model each subagent uses
- Subagents can run in their own isolated worktrees
- Workflows resume from interruption — session resume picks up where it left off

**Trigger:** Ask Claude to "make a workflow" or use `ultracode` as a trigger word.

## The Six Patterns (Anthropic's Canonical List)

| Pattern | Use when |
|---------|---------|
| **Classify-and-act** | Heterogeneous task types that need different handling; expensive model only where complexity warrants |
| **Fan-out-and-synthesize** | Large number of independent steps; each benefits from clean context. Synthesize is a barrier — waits for all agents |
| **Adversarial verification** | Any output that needs checking by something without skin in the game |
| **Generate-and-filter** | Brainstorming with quality control; deduplicate; return only verified survivors |
| **Tournament** | Qualitative ranking where comparative judgment beats absolute scoring; 1000+ items that won't fit in context |
| **Loop until done** | Unknown quantity of work; loop until stop condition met (no new findings, zero errors) |

## The Prompts That Actually Work

From the article's opening examples — these are Anthropic's own demo prompts:

```
"This test fails maybe 1 in 50 runs. Set up a workflow to reproduce it,
form theories and adversarially test them in worktrees.
/goal don't stop until one theory works."

"Using a workflow, go through my last 50 sessions and mine them for
corrections I keep making and turn the recurring ones into CLAUDE.md rules."

"Use a workflow to dig through #incidents in Slack for the past six months
and find recurring root causes where nobody has filed a ticket."

"Take my business plan and run a workflow where different agents tear it
apart from an investor's, a customer's, and a competitor's perspective."

"Here's a folder of 80 resumes, use a workflow to rank them for the
backend role and double-check the top ten."

"I need a name for this CLI tool. Use a workflow to brainstorm options
and run a tournament to pick the top 3."

"Go through my blog post draft and using a workflow verify every
technical claim against the codebase."
```

## Saving and Sharing Workflows

- Press `s` in the workflow menu to save
- Saved to `~/.claude/workflows`
- Share via a Skill: put JavaScript workflow files in the skill folder, reference in `SKILL.md`
- When packaging as a Skill: "prompt Claude to think of the workflows in the skill as a template instead of a script to run verbatim" — leaves room for adaptation while preserving structure

## When NOT to Use Dynamic Workflows

From the authors directly: "Workflows are not needed for every task and may end up using significantly more tokens."

- Regular coding tasks that don't need "a panel of 5 reviewers"
- Any task where asking yourself "does this really need more compute?" gives an honest "no"
- Best practices are still developing — treat this as experimental infrastructure

## Questions & Gaps
- The Bun Zig→Rust rewrite used workflows — was this the entire rewrite or a subset? Jarred's X thread referenced but not linked. The scale of that codebase migration would be a useful data point for scoping similar work.
- The `/deep-research` skill is mentioned as available inside Claude Code — is this via a skills package or built into the core product?
- "Workflows resume from interruption" — what is the resumption mechanism? Is it session-scoped state, disk state, or something else?
- Token budget prompting ("use 10k tokens") sets a cap — how is this enforced? At the model level or the orchestration level?
- The classifier-for-model-routing pattern — does this work reliably enough in practice to save costs, or does the classifier overhead eat the savings?

## Related Notes
- [Claude Code Dynamic Workflows: 6 Patterns and 14 Steps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-code-dynamic-workflows-6-patterns-14-steps.md) — structured synthesis with code examples, decision framework, and cost controls. Read that for implementation; read this for the authoritative framing and the examples Anthropic considers canonical.
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — the prior Anthropic engineering post on custom harnesses for Research, Code Review, and agent teams. Dynamic Workflows automates what that post described building by hand.
- [Loop Engineering — Addy Osmani](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-addy-osmani.md) — `/goal` + `/loop` primitives described there are the same primitives recommended here for pairing with triage workflows.
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — NEEDLE's state machine implements the same patterns at fleet scale. Fan-out-and-synthesize = parallel worker claims. Adversarial verification = maker/checker subagent split. Loop until done = strand escalation with `/goal`-equivalent stop condition.
- [Automated Blog Pipeline](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/automated-blog-pipeline-openclaw-5-phases.md) — a production implementation of fan-out-and-synthesize + adversarial verification + loop patterns using OpenClaw instead of Claude Code workflows.
