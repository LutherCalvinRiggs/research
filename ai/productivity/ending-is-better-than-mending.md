# Ending is Better than Mending

**Source:** https://jedarden.com/notes/ending-is-better-than-mending/
**Saved:** 2026-06-03
**Tags:** ai, productivity, planning, prompting, agentic-ai, learning

---

## TL;DR
The economics of code flipped: writing is cheap (tokens), understanding is expensive (human attention). When the agent produces wrong output, debugging it costs your scarce attention to load someone else's wrong reasoning into your head. `git reset --hard` + sharpen three sentences + re-dispatch is almost always cheaper — but only if the value lives in the plan, not the code.

## Key Concepts & Terms
- **The cost inversion**: Historically, code = accumulated human labor = expensive to throw away. Now, code = tokens = cheap to regenerate. What didn't get cheaper: human time, taste, judgment, attention. The optimal recovery move inverts with the cost structure.
- **The comprehension tax**: To fix wrong code you must first load the agent's reasoning into your head — including the wrong parts — before you can fix any of it. You pay full price to build a mental model of something you'll partially discard. Ending skips the tax entirely.
- **End the artifact, mend the plan**: The slogan in one line. Be ruthless with the disposable thing (code); be precious with the durable one (the plan). Regenerating against the same vague plan just produces differently-wrong output — thrashing, not ending.
- **The condition that makes ending safe**: Value has moved out of the artifact and into something durable. If the plan is the negative and the code is a print, burning the print costs nothing. If real knowledge lives only in the code (a subtle concurrency fix, edge cases discovered by running it), mend once — then move that knowledge into the plan.

## Main Arguments & Takeaways
- **Mending is a tax on the wrong input**: The human's job now is taste and judgment, not typing. Mending spends the expensive scarce input (comprehension) to rescue the cheap abundant one (code). The ratio is wrong.
- **Mend once, then end forever**: If knowledge lives only in the code, mend long enough to harvest that knowledge into the plan — then never mend that artifact again. The plan knows what the code learned; the next regeneration will include it.
- **Regenerating is not reckless if the floor is solid**: `git reset --hard` is safe on top of atomic commits, a clean plan, and a task graph that survives deletion. Knock out the floor — uncommitted work, no plan, no checkpoint — and ending becomes the reckless thing it sounds like.
- **Where Huxley's dystopia applies and where it doesn't**: Mindless disposability is corrosive when the attention saved by not salvaging never gets reinvested upstream. "End the artifact, mend the plan" is the opposite of the dystopia — judgment moves to the highest leverage point (the spec), where one improvement propagates into every future artifact.

## Notable Quotes
> "The plan is the negative; the code is just a print. You can burn the print."

> "End the artifact, mend the plan. Be ruthless with the disposable thing and precious with the durable one."

## Questions & Gaps
- How do you build the instinct for when knowledge lives only in the code? Are there reliable signals that a mend is worth making before ending?
- At what output size / complexity does the "comprehension tax" become real enough to make ending clearly cheaper? Is there a useful heuristic?
- Version-controlled plans — what does good plan version control look like in practice alongside code version control?

## Related Notes
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — why the plan is the durable artifact this post argues for.
- ['No' is Not an Instruction](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/no-is-not-an-instruction.md) — "regenerating against the same vague plan = bare 'no' wearing a git reset costume."
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the same disposability logic applied to the agent rather than the output.
