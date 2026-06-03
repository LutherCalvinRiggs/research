# Anchor on a Fact the Model Can't See

**Source:** https://jedarden.com/notes/anchor-on-a-fact-the-model-cant-see/
**Saved:** 2026-06-03
**Tags:** ai, prompting, productivity, agentic-ai, context

---

## TL;DR
Model fluency is uncorrelated with accuracy. An agent describes a wrong node pool with the same calm competence it uses to describe a correct one. The highest-leverage human move in any session is to hold out one ground-truth fact the agent couldn't have known — usually a number — and check its output against it. At fleet scale, this becomes the requirement to put operational facts into the plan.

## Key Concepts & Terms
- **Facts that live in the world, not the repo**: The actual node size, the real close rate, the column that's secretly nullable, the rate limit on a third-party API, the fact that this cluster is read-only. These live in dashboards, billing consoles, your own head — not in any file the agent can read.
- **Fluency without accuracy**: The model cannot tell when a fact is missing from its context. Absence of a fact doesn't feel like anything from inside a context window — so it doesn't ask. It interpolates, confidently, with the same production values it uses for verified facts.
- **Load-bearing number**: The quantitative claim a conclusion rests on. Hunt for it in every agent report — often unstated, implicit in a decision. Making the implicit number explicit is what enables the check.

## Main Arguments & Takeaways
- **You cannot audit an agent by reading for hesitation — there is none**: The model's confidence signal carries no information about correctness. You have to audit against something outside the text.
- **Numbers fail loudly; prose claims are slippery**: "This should improve latency" is hard to falsify. "The node is sized for 8GB" is either true or it isn't, and you can check by looking at the billing console.
- **Your job in the loop is to be the holder of facts the agent structurally cannot hold**: Not to write code (the agent is faster); not to generate ideas (the agent does that too). The irreplaceable human contribution is the operational context that lives outside any repository.
- **Fleet translation**: Interactively you supply the missing fact in real time. At cattle scale you supply it in advance, in writing (in the plan), and you keep auditing the output against the world — because a confident fleet report is exactly as trustworthy as a confident pet session report, which is to say not at all until checked.
- **Spot-check discipline**: Read the report, find the load-bearing number, check it against your own knowledge. Ask explicitly if the agent didn't state it: "what RAM budget did you assume here?" The moment it commits to a figure, you can compare.

## Notable Quotes
> "The model is fluent about a number it can't see. Your job is to be the one who can see it."

## Questions & Gaps
- What types of facts are most commonly missing from agent context in practice? Could these be systematized into a pre-task checklist?
- At fleet scale — auditing confident reports against dashboards manually doesn't scale either. What does automated spot-checking of fleet outputs look like?
- Is there a way to design prompts/task templates that force the agent to state its assumptions explicitly, making the load-bearing numbers visible?

## Related Notes
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — the same verification instinct: the check must come from somewhere the work didn't.
- [Benchmarks Measure a Model You Aren't Running](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/benchmarks-measure-a-model-you-arent-running.md) — on what the model's context actually contains and what it's missing.
