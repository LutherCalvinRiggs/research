# Benchmarks Measure a Model You Are Not Running

**Source:** https://jedarden.com/notes/benchmarks-measure-a-model-you-arent-running/
**Saved:** 2026-06-03
**Tags:** ai, research, prompting, benchmarks, context, agentic-ai

---

## TL;DR
Every major coding benchmark evaluates a model with its context window essentially empty (median 16–456 tokens across 7 benchmarks). Pet sessions run the opposite — buried in accumulated back-and-forth. Benchmark scores are ceiling measurements under near-optimal conditions; cattle workers with clean contexts and scoped tasks are the only deployment mode that actually approaches those conditions in production.

## Key Concepts & Terms
- **Benchmark operating condition**: Nearly empty context, no prior turns. HumanEval median: 117 tokens. SWE-bench Verified median: 294 tokens. MBPP median: 16 tokens. All 9,755 problems across 7 benchmarks fit under 8,000 tokens — the largest is 12% of Claude's 200K context window.
- **Lost in the middle**: Documented phenomenon where models attend less reliably to information in the middle of a long context than at the start or end. Pet sessions bury the original task requirements under layers of accumulated back-and-forth — making the critical content the least attended.
- **Prior bias from wrong attempts**: A long session contains the model's earlier wrong attempts. Those attempts create a prior that shapes subsequent outputs. The benchmark model has no such prior — it sees the problem cold.
- **Clean context cattle**: A stateless cattle worker receives: project instructions (CLAUDE.md), task body from the bead, explicit reference material. Structurally close to benchmark conditions — small deliberate input, no prior turns, no dead ends.

## Main Arguments & Takeaways
- **Benchmark scores describe the ceiling; pet sessions fall below it and drift further as sessions age**: The degradation is systematic, not random. Longer sessions → more accumulated noise → larger gap from benchmark conditions.
- **You are paying for capability you are not fully using in pet sessions**: A more capable model in a long pet session is better than a less capable model in a long pet session — but both operate below benchmark-measured ceiling. The gap grows with session length.
- **Cattle workers are the deployment mode that closes the delta**: Stateless dispatch into a clean context with a scoped task is structurally the same as a benchmark evaluation. The score you bought is roughly the score you run.
- **The empirical measurement**: 9,755 problems across HumanEval, MBPP, SWE-bench, SWE-bench Verified, LiveCodeBench, BigCodeBench, APPS. Every problem fits under 8K tokens. The median is under 500 tokens. Raw data published at jedarden.com/data/benchmark-tokens.json.
- **The diagnostic gap in current benchmarks**: None measure performance at turn 30 vs. turn 1, or with 80K prior context vs. 500 tokens, or how much capability recovers by compressing/distilling context vs. starting fresh. Those measurements would be more useful for production decisions.

## Notable Quotes
> "When you read that a model scores X% on SWE-bench Verified, the deployment model that will actually realize that capability is a stateless worker with a clean context and a well-scoped task — not an ongoing chat session where the model has been talking to you for two hours."

## Questions & Gaps
- Has anyone actually measured model performance degradation curves at different context lengths in real production sessions?
- For tasks where accumulated session context genuinely helps (e.g. long debugging sessions), what's the best practice for preserving the useful signal while discarding the noise?
- The "lost in the middle" research — is this confirmed across all frontier models or primarily an older finding?

## Related Notes
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the deployment model this post argues for empirically.
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — how clean-context cattle works: the plan document substitutes for accumulated conversation.
- [I Stopped Hitting Claude's Usage Limits](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/stopped-hitting-claude-usage-limits-10-things.md) — context window management from the user's perspective; the summarize-and-restart technique relates directly to this post's findings.
