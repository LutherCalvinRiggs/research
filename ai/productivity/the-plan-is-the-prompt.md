# The Plan is the Prompt

**Source:** https://jedarden.com/notes/plan-is-the-prompt/
**Saved:** 2026-06-03
**Tags:** ai, productivity, prompting, planning, orchestration, agentic-ai

---

## TL;DR
In a cattle fleet, the detailed plan document is the most important artifact you will write — not the code. A pet session accumulates task context through dialogue; cattle workers start cold with only what you wrote down. Plan gaps cost O(N × execution budget) across a fleet, not O(1) like in a chat.

## Key Concepts & Terms
- **Genesis bead**: Root task in NEEDLE's hierarchy. References the plan document. Ties all phases together.
- **Phase beads**: One coherent unit of work per phase — a vertical slice, capability, or milestone — with its own acceptance criteria and child tasks. Phase boundaries defined as conditions, not dates.
- **Task beads**: Atomic units a worker executes in one run. Worker reads task bead → parent phase bead → genesis bead's plan reference (smallest to largest scope). Only works if the plan is coherent enough to anchor the hierarchy.
- **Plan vs. specification**: A plan records decisions without recording deliberations. It says "this tradeoff resolved this direction" not "here are both sides." Compression is the point — workers need foreground, not background.
- **Plan-review gate**: `/plan-review` skill that checks 80+ structural patterns before any worker touches code. Developed from Jeffrey Emanuel's methodology for plans that survive contact with implementation. Outputs a scorecard (PRESENT/PARTIAL/MISSING). Available at jedarden/jeds-curated-skills.

## Main Arguments & Takeaways
- **The cost asymmetry**: In a pet session, a plan gap costs one correction — O(1). In a cattle fleet with N workers, the same gap costs N failed executions before you notice the pattern — O(N × execution budget). At 20 workers × 100K token budget × 2 failed iterations = 4M tokens wasted on a single plan gap.
- **Six things every plan needs**: Scope lock (precise enough for in/out-of-scope decisions); testable acceptance criteria (not aspirations — required for the orchestrator to classify outcomes); phase boundaries as conditions not calendar dates; known unknowns stated explicitly; constraint inventory (the fixed points that eliminate solution space); rollback plan.
- **The plan is the negative; the code is a print**: Code is reproducible from the plan in hours by a fleet. If the plan is wrong, fix the plan, delete the code, regenerate. The plan is the expensive artifact.
- **Test for plan completeness**: "Could a worker who has never spoken to me, reading only this document and the task bead, produce something I would accept on the first try?" Both parts matter — "never spoken to me" rules out context-dependent plans; "on the first try" rules out plans that need iteration to clarify.
- **Most common failure patterns in plans**: No acceptance criteria (workers can't self-evaluate); no phase gates (workers don't know when a phase is complete); no rollback plan (failures have no recovery path); no constraint inventory (workers make incompatible design decisions).

## Notable Quotes
> "The workers are ready. The question is whether the inputs are."

## Questions & Gaps
- The plan-review skill checks 80+ patterns — what are the highest-signal ones that catch the most failures?
- How do you handle plan drift (plan written at start of project becomes stale after 6 weeks)? Are there good conventions for marking sections as superseded?
- For smaller tasks that don't warrant a full genesis/phase/task hierarchy — is there a lightweight version of this practice?

## Related Notes
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the context for why plan quality has a fleet-level cost multiplier.
- [Ending is Better than Mending](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/ending-is-better-than-mending.md) — if the plan was wrong, fix the plan and regenerate the code — don't mend the output.
- ['No' is Not an Instruction](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/no-is-not-an-instruction.md) — the same principle at prompt level: unspecified rejections = plan gaps at micro scale.
