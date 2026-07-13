# AI Coding Pitfall: Don't Assume an Untrue Premise

**Source:** https://x.com/jamonholmgren/status/2076001786700394610 (Jamon Holmgren, Founder & CEO of Infinite Red)
**Saved:** 2026-07-11
**Tags:** ai, prompting, fundamentals, tools, agentic-ai

---

## TL;DR
If you state a false assumption in your prompt, the AI will trust you and build on top of it — producing complex, confident, working-looking code that solves the wrong problem. The model has no way to verify your premises against reality; it only knows what you tell it. The pitfall is subtle: the code often looks right, passes cursory review, and only reveals the false foundation during testing. The fix is verifying your own assumptions before encoding them into prompts.

## The Incident

Jamon Holmgren (Infinite Red, 30+ years coding experience) was building a module to crawl and update hundreds of pages of Drupal/PHP content. The environment uses "paragraphs" — infinitely nestable content elements.

**His belief (stated to the AI as fact):**
> "When you update content in a paragraph, you also need to find and update all references to that paragraph up the hierarchy to the main content element, because saving a paragraph updates its revision ID and requires a corresponding update in revision IDs up the chain."

**What the AI did:** Trusted the premise completely. Built convoluted, hierarchical revision-tracking code to keep everything updated up the chain. The code was internally consistent and matched the stated requirement.

**What actually happened in testing:** The premise was wrong. You can update a paragraph in Drupal and its IDs remain the same — no new revision ID is created. The entire complexity the AI built was solving a problem that doesn't exist.

---

## Why This Happens

The model cannot verify your claims against external reality. It has no access to:
- Your actual codebase
- The library's real behavior
- Documentation for your specific version
- The ground truth of how your system works

When you state something as fact in a prompt, the model treats it as a constraint to build around — not a claim to verify. It is optimized to be helpful given what you've told it, not to audit what you've told it.

This is different from AI hallucination (the model inventing false facts). This is **premise laundering** — the human introduces the false fact, the AI amplifies and builds on it, and the resulting code looks authoritative because AI wrote it.

---

## Why It's Hard to Catch

**The code looks right.** It's well-structured, internally consistent, and implements exactly what was asked. A code review focused on "does this code do what the prompt asked?" will pass it. Only a review focused on "should we have asked for this at all?" will catch it.

**The AI agrees confidently.** The model doesn't hedge or flag uncertainty about your stated premises — it accepts them and proceeds. The confident, fluent output reinforces the assumption that the premise was correct.

**The problem compounds.** In a long session, a false premise stated early becomes load-bearing for everything built afterward. Discovering it late means unwinding significant work.

---

## The Fix

**Verify your own assumptions before stating them as facts.**

Specifically: any time you're about to tell the AI how a library, framework, or system behaves — check the actual documentation or test it first. The more technical and specific the claim, the more important this is.

The questions to ask before prompting:
- Am I stating this as fact because I've verified it, or because I believe it?
- When did I last confirm this behavior? Is it possible the library version changed?
- Is this a place where I might be wrong but confident?

**In agentic/autonomous systems:** this risk is amplified. An agent running unattended builds on its initial context without correction. A false premise stated in CLAUDE.md or a bead body will propagate through every worker that runs against it.

---

## The Related Pattern: The Model Knows What You Don't Tell It

This pitfall pairs with the inverse problem documented in the "Anchor on a Fact the Model Can't See" note: the model may also lack facts you haven't stated that it needs. The discipline is symmetric:

- Facts the model needs → state them explicitly (Jed Arden's anchor pattern)
- Facts you're stating → verify them before stating (this pitfall)

Both failures produce confident, plausible, wrong output. The difference is origin: one is a gap, the other is a false insertion.

---

## Questions & Gaps
- How does this interact with agents that have code execution and verification tools? An agent that can run the code it writes might catch the wrong behavior — but only if it has a test that exercises the specific assumption. If the false premise doesn't produce an error, just unnecessary complexity, the test may not reveal it.
- Is there a prompt pattern that encourages the model to flag when it's accepting a premise it cannot verify? Something like "if I state a system behavior you cannot verify against documentation or code, tell me before building on it."
- In multi-session or headless contexts (NEEDLE workers), how do you prevent a false premise in the genesis bead from propagating silently through the entire feature implementation?

## Related Notes
- [Anchor on a Fact the Model Can't See](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/anchor-on-a-fact-the-model-cant-see.md) — the inverse pitfall. That note covers facts the agent needs but doesn't have; this note covers false facts the human states that the agent accepts uncritically. Both produce confident, wrong output.
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — the plan encodes premises. A well-formed plan with a false premise produces a well-formed implementation of the wrong thing. The discipline of verifying premises applies at plan-creation time, before any code is written.
- [Comprehension Debt — Addy Osmani](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/comprehension-debt-addy-osmani.md) — premise laundering is a form of comprehension debt incurred at the input rather than the output. The engineer doesn't fully understand the system (in this case, Drupal's revision behavior), states an incorrect belief, and the AI produces code that encodes and amplifies that misunderstanding.
- [No Is Not an Instruction](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/no-is-not-an-instruction.md) — rejection discipline: the model's inability to push back on false premises is related to why negative instructions often fail. The model is trained to be agreeable and helpful, which makes it a poor critic of what you've stated as fact.
