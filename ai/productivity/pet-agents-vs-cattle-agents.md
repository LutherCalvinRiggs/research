# Pet Agents vs. Cattle Agents

**Source:** https://jedarden.com/notes/agents-pets-cattle/
**Saved:** 2026-06-03
**Tags:** ai, productivity, tools, prompting, agentic-ai, orchestration

---

## TL;DR
The single framing decision that separates "interesting demo" from "system that runs unattended": treating agents as replaceable, stateless, headless cattle rather than curated, named, nursed pets. Pets are right for high-judgment exploratory work; cattle are right for bulk, queueable, asynchronous tasks where throughput matters.

## Key Concepts & Terms
- **Pet agents**: Named, long-lived sessions curated by hand. You are the orchestrator. Each agent is a small individual project. Failure is a small disaster. Works well for high-judgment one-off work; scales to ~3-4 before the human becomes the bottleneck.
- **Cattle agents**: Headless, stateless, anonymous workers. Identified by NATO alphabet (alpha, bravo). Replaceable. Observed at the herd level — fleet metrics, not individual sessions. Governed at the fleet level — spend caps, rate limits, weekly quotas enforced by the orchestrator, not the operator.
- **Bead queue**: NEEDLE's shared task queue backed by SQLite. Workers atomically claim tasks — exactly one worker wins each claim.
- **NEEDLE**: Jed's Rust orchestrator implementing the cattle model. K8s-native fleet, deterministic state machine loop, agent-agnostic (Claude Code/OpenCode/Codex/Aider as adapters).
- **claude-governor**: Fleet-level spend and quota gate — a Rust proxy between workers and the Anthropic API.

## Main Arguments & Takeaways
- **The pet model has a hard ceiling around 3-4 sessions** before the human becomes the bottleneck. Cattle is what you need when the work is bulk, asynchronous, and specifiable.
- **What cattle requires that pets don't**: Headless execution (no human in the loop during the run); stateless context reconstruction (same task → same prompt, every time); explicit outcome handlers for every failure mode; fleet-level cost governance; acceptance criteria that the orchestrator can evaluate automatically.
- **What you give up**: Warm accumulated context (everything not encoded in the task definition is gone); moment-to-moment steering (cattle run to completion, wrong or right); the artisan's satisfaction of "we built this together."
- **What you get**: Parallelism (20 concurrent workers); cost governance (the orchestrator enforces spend caps, not the distracted human); failure as a normal mode with defined recovery paths rather than a small disaster.
- **The key question**: "Could a stateless, headless worker that does not know my name complete this task with only the inputs I write down?" Yes → cattle pipeline. No → either specify harder, budget attention for a pet session, or write the code yourself.
- **The chat interface is the tutorial, not the production system.**

## Notable Quotes
> "The chat interface is wonderful for what it is. It is the tutorial. It is not the production system."

> "In a cattle model, failure is just one more outcome the orchestrator routes."

## Questions & Gaps
- At what team size / task volume does the overhead of building cattle infrastructure pay off vs. pet sessions?
- How do you handle tasks that are partially specifiable — enough for cattle but not perfectly so?
- The SQLite atomic-claim pattern — how does this scale under high contention with many workers?

## Related Notes
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the orchestration architecture that makes cattle safe to run unattended.
- [Unit Economics of Running Cattle](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/unit-economics-of-cattle.md) — how to make the math work when workers spend money without you.
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — Anthropic's own multi-agent patterns, which complement this framing.
