# Harness Design for Long-Running Application Development

**Source:** https://www.anthropic.com/engineering/harness-design-long-running-apps
**Saved:** 2026-06-03
**Tags:** ai, tools, agentic-coding, multi-agent, llm, claude-code

---

## TL;DR
Anthropic engineer Prithvi Rajasekaran details how a GAN-inspired generator-evaluator multi-agent architecture dramatically improved Claude's output quality on both frontend design and full-stack app development — at the cost of longer runtimes and higher token spend.

## Key Concepts & Terms
- **Harness**: The scaffolding around a model — prompts, agent structure, context management — that shapes how it performs on complex tasks.
- **Generator-Evaluator loop**: A two-agent structure inspired by GANs where one agent produces output and a separate agent critiques it, driving iterative improvement.
- **Context anxiety**: A behavior where models begin wrapping up work prematurely as they approach their perceived context limit — distinct from actually running out of tokens.
- **Context reset**: Clearing the context window entirely and starting a fresh agent with a structured handoff artifact, rather than summarizing in place (compaction). Gives a clean slate; addresses context anxiety.
- **Sprint contract**: A pre-sprint agreement between generator and evaluator agents on exactly what "done" looks like before any code is written — bridges the gap between high-level spec and testable implementation.
- **Sprint**: A discrete chunk of work the generator completes before handing off to the evaluator for QA.

## Main Arguments & Takeaways
- **Self-evaluation is unreliable**: Agents asked to grade their own work consistently over-praise it, even when quality is obviously poor. Separating generator from evaluator is a strong fix — tuning a standalone evaluator to be skeptical is far more tractable than making a generator self-critical.
- **Subjective quality can be made gradable**: Instead of asking "is this beautiful?", the harness asked "does this meet these four design criteria?" — covering design quality, originality, craft, and functionality. Explicit criteria steered the model away from generic "AI slop" patterns.
- **The evaluator used Playwright to actually interact with the app**, not just read code or screenshots. This caught real bugs (broken route matching, missing mouse event handlers, wrong state conditions) that static review would miss.
- **Context resets were essential for Opus 4.5** but became unnecessary with Opus 4.6, which handles long autonomous sessions natively. This is a key example of harness components going stale as models improve.
- **The planner agent was load-bearing**: Without it, the generator under-scoped from the raw prompt. The planner expanded a one-sentence prompt into a 16-feature spec with a visual design language — and was explicitly prompted to weave in AI features.
- **Sprint decomposition was also removable with 4.6**: The updated harness dropped per-sprint evaluations in favor of a single QA pass at the end, cutting complexity without degrading output.
- **Cost/quality tradeoff is real**: Solo agent — 20 min, $9. Full harness — 6 hrs, $200. The harness produced a working game; the solo run produced a broken one.
- **Core lesson**: Every harness component encodes an assumption about what the model can't do alone. When a new model ships, strip the harness back and re-test each assumption — some will no longer be load-bearing.

## Notable Quotes
> "Find the simplest solution possible, and only increase complexity when needed." — Building Effective Agents (referenced)

> "The space of interesting harness combinations doesn't shrink as models improve. Instead, it moves, and the interesting work for AI engineers is to keep finding the next novel combination."

## Questions & Gaps
- The evaluator still missed subtle bugs in "deeply nested features" — what would it take to get evaluator coverage truly comprehensive?
- How transferable is this harness to domains outside software (e.g. writing, research, data analysis) where correctness is harder to verify programmatically?
- The 4-hour, $200 runs are expensive for individuals — at what price point does this become practical outside of Anthropic's own research?
- The author notes wording choices like "museum quality" steered outputs in unexpected ways — how much prompt sensitivity exists in the evaluator criteria, and how stable is it across models?
- Claude can't hear, which limited QA effectiveness on the DAW's audio features — how should harnesses handle modalities the model can't directly evaluate?

## Related Notes
- [I Stopped Hitting Claude's Usage Limits — 10 Things I Changed](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/stopped-hitting-claude-usage-limits-10-things.md) — Both deal with Claude's context window: this article goes deep on architectural solutions (resets, compaction, multi-agent handoffs), while the other covers practical user habits for reducing token burn.
