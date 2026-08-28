# Benchmarks Measure a Model You Are Not Running

**Source:** https://jedarden.com/notes/benchmarks-measure-a-model-you-arent-running/
**Author:** Jed Arden (@jedardencodes)
**Published:** May 17, 2026 · 8 min read
**Saved:** 2026-05-17 (updated 2026-08-25 with full primary source)
**Tags:** ai, productivity, orchestration, fundamentals, agentic-ai

> Companion pieces: [Pet Agents vs. Cattle Agents](https://jedarden.com/notes/agents-pets-cattle/) and [The Plan Is the Prompt](https://jedarden.com/notes/plan-is-the-prompt/). Data: 9,755 benchmark problems measured with `cl100k_base` tokenizer. Raw data at jedarden.com/data/benchmark-tokens.json.

---

## TL;DR
Every major coding benchmark evaluates a model operating with essentially an empty context window. SWE-bench problems have a median of 282 tokens — 0.14% of Claude's 200K context window. HumanEval: 117 tokens (0.06%). MBPP: 16 tokens (0.008%). The model being scored is not the model you run in a pet session after two hours of back-and-forth. Benchmark scores are ceiling measurements under clean-context conditions. Cattle workers — stateless, dispatched with a precise task body and no accumulated conversation — are the deployment model closest to benchmark conditions. That is the case, made empirically.

---

## The Benchmark Token Data

Measured across 9,755 problems using `cl100k_base` tokenizer (GPT-4 / tiktoken). Numbers are for raw problem input only — text actually sent to the model.

| Benchmark | Problems | Median | P99 | Max |
|-----------|---------|--------|-----|-----|
| MBPP | 257 | **16 tokens** | 48 | 49 |
| HumanEval | 164 | **117 tokens** | 310 | 391 |
| BigCodeBench | 1,140 | **129 tokens** | 363 | 1,216 |
| SWE-bench Verified | 500 | **294 tokens** | 2,514 | 6,939 |
| SWE-bench | 2,294 | **282 tokens** | 2,937 | 22,483 |
| LiveCodeBench | 400 | **421 tokens** | 1,105 | 1,521 |
| APPS | 5,000 | **456 tokens** | 1,103 | 1,815 |

Every problem across all seven benchmarks fits under 8,000 tokens. The largest problem in the entire dataset (SWE-bench outlier at 22,483 tokens) is still under 12% of Claude's 200,000-token context window. At the MBPP median, the model uses 0.008% of the available context.

**The model being benchmarked is operating with its context window essentially empty.**

---

## Two Conditions That Travel Together (But Shouldn't Be Conflated)

A benchmark evaluation has two distinct properties:

**1. Few tokens of input.** The problem statement is small. A HumanEval prompt at 117 tokens is shorter than most Slack messages.

**2. No prior turns.** The context contains nothing except the problem. No corrections from earlier. No dead ends. No half-finished implementations the model is tempted to continue in the wrong direction. No long system prompt that is now only partially relevant.

> "The second condition is the one that matters more, and it is the one people talk about least."

---

## What Fills a Pet Session's Context

A pet agent session accumulates context continuously. After an hour of work, the context contains:
- The original task description (probably underspecified)
- Early corrections and clarifications
- The model's first attempt (which missed something)
- The redirect
- The model's second attempt (partially correct)
- More corrections
- Tool call outputs — file reads, test runs, error messages
- The model's current understanding, shaped by all of the above

**Two degradation mechanisms:**

**Lost in the middle:** Research shows models attend more reliably to content at the start and end of a context than in the middle. A long pet session buries the actual task requirements under layers of accumulated back-and-forth.

**Prior anchoring:** The model's wrong attempts are in the context. It has seen itself go down a path. It has a prior. That prior shapes the next attempt — not always in the right direction.

The benchmark model has none of this. It sees the problem cold. Its full capability for that problem is expressed without interference.

---

## The Cattle Worker Is the Benchmark Condition

A stateless cattle worker dispatched by an orchestrator starts each task with a context containing:
- Project instructions (CLAUDE.md) — stable, curated, written once
- Task body from the bead — a precise specification of one unit of work
- Whatever reference material the task body explicitly includes

**This is structurally identical to a benchmark evaluation.** Small, deliberate input. No accumulated conversation. No prior wrong attempts. No corrections that anchored the model toward a direction it should abandon.

> "Benchmark scores are a better predictor of cattle performance than pet performance."

The pet session is not that model. The pet session operates under conditions systematically worse than benchmark conditions — and those conditions degrade further the longer the session runs.

---

## The Score You Are Buying Is Not What You Are Running

Most people buy capability (higher benchmark score, more expensive tier) and then run it in a mode that systematically degrades that capability below what the benchmark measured.

A pet session with a more capable model is better than a pet session with a less capable model — but both operate below their benchmark ceiling. The gap between "what the benchmark measured" and "what the pet session delivers" grows as the context accumulates. By hour three with 50,000 tokens of back-and-forth, you are running a noticeably different model than the one that got the score.

**There is no equivalent degradation in the cattle model.** A stateless worker dispatched against a well-specified task is as close to benchmark conditions as production use gets. The score is what you bought. The score is roughly what you run.

**The economic implication:** If you are running pet sessions, you are paying for capability you are not fully using. If you are running cattle workers with clean contexts and scoped tasks, you are.

---

## What an Honest Benchmark Would Measure

The existing benchmarks were designed for clean evaluations, not for measuring how capability degrades across a production conversation. None of them measure:

- Performance at turn 30 vs. turn 1
- Performance with 80K tokens of prior context vs. 500
- How much capability is recovered by compressing and distilling context vs. starting fresh

Existing benchmarks tell you the ceiling. What is missing is the curve: how quickly do you fall from that ceiling as context accumulates, and how do different deployment models (cattle vs. pet) track that curve differently?

Until those measurements exist, the empirical data points in one direction: the deployment model closest to benchmark conditions is stateless dispatch into clean context. That is the cattle model.

---

## The Question to Ask When Someone Cites a Benchmark

> "Under what context conditions was that score measured, and are those the conditions we are actually running?"

If the answer is "a 300-token prompt against a blank context window" and the deployment is "an ongoing chat session running for six hours" — the score describes a model that is not the model being run.

Knowing this does not mean you stop using pet sessions. There are tasks where they are the right tool, and where accumulated context is a feature rather than a bug. It means you understand that benchmark scores are ceiling measurements, and the delta between the ceiling and what you actually get is a function of the deployment model you chose.

Cattle closes that delta. That is the case, made empirically.

---

## Questions & Gaps
- The "lost in the middle" phenomenon is cited but the specific research is not linked. What is the source, and how consistent is the effect across different model architectures and context lengths?
- The cattle model eliminates context accumulation — but introduces the problem of how to pass accumulated project knowledge to each worker. CLAUDE.md + bead body is the answer, but how well does this scale for projects where the relevant context genuinely exceeds a well-scoped task body?
- No measurement exists yet of performance vs. context fill percentage across pet vs. cattle sessions. This would be the most practically useful data to have — does anyone have it?

## Related Notes
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the foundational framing this note extends. This note provides the empirical benchmark data that makes the cattle argument concrete.
- [The Plan Is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — the mechanism that makes clean-context cattle work. If the plan encodes all necessary context, the worker starts with what it needs without a polluted history.
- [Claude Code Model vs. Effort Selection](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/claude-code-model-vs-effort-selection.md) — the Anthropic primary source on model selection. That note explains that model weights are frozen at inference; this note adds that the context you give those frozen weights also fundamentally shapes the result. Both matter.
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — NEEDLE implements the cattle deployment model. Each bead worker starts with clean context (CLAUDE.md + bead body). This is the production system that makes benchmark-condition deployment achievable.
- [Jed Arden Workflow Manual](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/jed-arden-workflow-manual.md) — sections 03–06 describe how to write task specifications (bead bodies) precise enough to deliver benchmark-condition context to each worker. "Everything upstream of dispatch is really one activity: getting decisions out of your head and into one of those three channels."
