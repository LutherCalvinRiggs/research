# When AI Builds Itself — Anthropic Institute on Recursive Self-Improvement

**Source:** https://www.anthropic.com/institute/recursive-self-improvement
**Saved:** 2026-06-09
**Tags:** ai, research, ethics, infrastructure, fundamentals, agentic-ai

---

## TL;DR
The Anthropic Institute presents internal data showing AI is already meaningfully accelerating its own development: Claude authors >80% of Anthropic's merged code, engineers ship 8× as much code per quarter as in 2024, Claude Mythos Preview achieves 52× speedups on optimization tasks vs ~4× for a skilled human, and Claude now beats human researchers 64% of the time when choosing the next step in an open-ended investigation. The piece maps three possible futures (stall, compounding efficiency gains, full recursive self-improvement) and argues the window for global coordination is here — but the verification infrastructure needed for a credible pause doesn't yet exist.

## Key Concepts & Terms
- **Recursive self-improvement (RSI)**: An AI system capable of fully autonomously designing and developing its own successor, creating a compounding acceleration loop. The piece argues this is not inevitable but could arrive sooner than institutions are prepared for.
- **Task horizon**: How long an AI can reliably complete a task unsupervised. Has been doubling every ~4 months (up from every 7 months). March 2024: Claude Opus 3 managed 4-minute tasks. April 2025: Claude Sonnet 3.7 managed 90-minute tasks. April 2026: Claude Opus 4.6 managed 12-hour tasks.
- **Amdahl's Law (applied to orgs)**: Speeding up one part of a process shifts the bottleneck elsewhere. Overall pace is capped by the parts that haven't sped up. Already observed at Anthropic: accelerated code generation made human code review the new bottleneck.
- **Research taste**: The capability to decide which problems are worth working on, which results to trust, and when an approach is a dead end. The piece identifies this as the current primary area of human comparative advantage — and the narrowing gap that would close the loop toward full RSI.
- **Claude Mythos Preview**: Anthropic's most capable internal model (not publicly available). Achieved 16+ hour task horizons and is "at the upper end of what METR can measure without new tasks." Used for the internal capability data in this piece.
- **Project Glasswing**: Mythos Preview's early deployment found >10,000 high- and critical-severity software vulnerabilities across the world's most important systems. The bottleneck in cyber defense has already shifted from finding vulnerabilities to patching them fast enough.
- **Automated W2S researcher**: Anthropic's April 2026 demonstration of Claude agents running an open-ended AI safety research project end-to-end — proposing hypotheses, testing them, sharing findings with parallel agents, iterating. Recovered 97% of the performance gap vs 23% for two human researchers in ~a week, using ~800 cumulative agent-hours and $18,000 in compute.

## The Internal Data (Unprecedented Specificity)

### Code production
- **>80%** of code merged to Anthropic's codebase as of May 2026 is authored by Claude (vs low single digits pre-Claude Code, Feb 2025)
- **8×** more code per engineer per day in Q2 2026 vs 2024
- Two inflection points: (1) Claude began running code rather than just suggesting it (2025); (2) models began working autonomously over longer time horizons (2026)
- Caveat: lines of code is quantity not quality — 8× is likely an overstatement of true productivity gain

### Code quality
- Rate at which staff correct, redirect, or take over mid-task from Claude: falling steadily for a year
- Open-ended task success rate (most difficult tier): **76%** in May 2026, up 50 percentage points in 6 months
- Claude-written code: worse than human-written in late 2025, roughly at parity today, expected to be strictly better within the year
- Automated Claude reviewer retrospective: would have caught ~⅓ of the bugs behind past claude.ai incidents before they reached production

### Research capability
- Code optimization benchmark: Claude Opus 4 achieved ~3× speedup in May 2025 → Claude Mythos Preview achieved ~52× in April 2026. A skilled human researcher achieves ~4× in 4–8 hours.
- Next-step judgment (129 sessions where human choice had room for improvement): Claude Opus 4.5 beat the human choice 51% of the time (Nov 2025) → Mythos Preview beat it **64%** of the time (April 2026)
- Subjective productivity: median Anthropic employee estimated 4× output with Mythos Preview vs no AI (March 2026, n=130)

### April 2026 incident (scale of autonomous work)
Claude shipped **800+ fixes** reducing a class of API errors by a factor of **1,000**. Estimated 4 human years of work completed by Claude. Enabled by Claude's ability to hold unfamiliar context that humans struggle to maintain.

## The Three Futures

### 1. Trend stalls, capabilities diffuse (least likely per Anthropic)
S-curve rather than exponential. Research taste proves irreducible to scaling. Binding constraints are physical (energy, compute, chip supply) rather than intelligence. Even at today's frozen capabilities, major disruption occurs — 100-person companies doing 1,000-person work. Most time for societal adaptation. Anthropic says they don't believe this is likely.

### 2. Compounding efficiency gains, human direction persists (most likely near-term)
AI handles execution; humans set direction. Each researcher/engineer steers far more work than before. 100-person companies do work of 10,000- or 100,000-person organizations. New bottlenecks emerge at the unseeded parts (human code review, decision capacity, organizational processing speed). Knowledge work and government services revolutionized. Also weaponizable: authoritarian surveillance at scale, hyper-personalized influence operations.

### 3. Full recursive self-improvement
AI systems develop sufficient research taste to design their own successors. Pace of AI progress determined entirely by compute availability and algorithmic efficiency gains. Humans shift to oversight, validation, and verification of a "virtual lab." Alignment problem becomes critical — misalignment present in today's models could compound through the self-improvement loop. Most uncertain future. Physical bottlenecks (embodiment, election cycles, clinical trial durations, human relationships) still constrain felt pace of change for most people even as the lab upstream accelerates.

## The Coordination Problem

The piece argues for the *option* to slow or pause, not a unilateral slowdown. A credible pause requires:
- Multiple well-resourced labs at or near the frontier in multiple countries agreeing to stop under the same conditions
- Verification that others have actually stopped (not just detected)
- Training runs are far easier to conceal than missile silos; inputs are general-purpose; incentive to defect quietly is enormous
- Requires specification of trigger conditions, lifting conditions, and adjudication mechanism
- Historical precedent (INF Treaty) took decades; AI doesn't have that long

Anthropic's position: a unilateral pause changes who the front-runner is but doesn't create the needed global deliberation. They will organize conversations in coming months with policymakers, researchers, civil society, and other AI companies. The window is here. People outside AI companies should be involved.

## What Is Still Distinctly Human (For Now)
- Choosing which problems matter
- Deciding which results to trust
- Recognizing when an approach is a dead end
- Research taste and judgment
- Direction-setting vs. execution

Per the piece: "The comparative advantage of humans as of right now is still in seeing the bigger picture and thinking beyond the confines of the immediate task."

## The Human Cost (Two Employee Quotes Worth Noting)
> "Work (and life) ran on a gift economy of small favors between humans. [...] each one created a little debt, a little mutual awareness. [Claude is] faster, it creates zero debt, but each of these is a lost bid for human collaboration."

> "On days where everything works well, I can't help but think nothing I do matters, everything is automated and better and faster than I ever will be. But then there are days where everything breaks and I don't understand why and I realize I have no idea what I've been up to anymore."

## Questions & Gaps
- The automated W2S researcher result "didn't transfer cleanly to production-scale models" — how significant is this limitation? If the research capability only works at small scale, how close are we really to closing the loop?
- The next-step judgment test deliberately selected sessions where the human had room for improvement — how does Claude perform on the sessions where the human's direction was already strong? (The article mentions a check: models were judged better only ~20% of the time on the strong-human set. This is a substantial caveat the piece buries in a footnote.)
- The 4× median productivity estimate from the March 2026 survey — recent METR research shows developer estimates of AI productivity uplift are often overestimated. How much does this discount the claim?
- Project Glasswing found 10,000+ critical vulnerabilities. Who is responsible for coordinating the patches? The bottleneck has shifted from finding to fixing — but fixing at that scale requires organizational capacity that doesn't obviously exist.
- The piece doesn't address what happens to alignment research itself under RSI. If models are building their successors, who is doing the alignment work? Is it automated too?

## Related Notes
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — the generator-evaluator harness Anthropic described there is now internalized as standard Claude Code operation. The 76% success rate on open-ended tasks is the harness research paying off at scale.
- [Benchmarks Measure a Model You Aren't Running](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/benchmarks-measure-a-model-you-arent-running.md) — this piece provides the internal data that public benchmarks can't: what's happening inside a frontier lab, not just on SWE-bench. The two should be read together.
- [Agentic Coding Ladder](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/agentic-coding-ladder.md) — Jed Arden's 9-rung ladder maps directly to the capability trajectory described here. "Workers exit when done" (rung 8) → "you give up being reachable" is where Anthropic engineers are living now. Rung 9 (Operator) is what full RSI looks like from the inside.
- [Claude Code Dynamic Workflows](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-code-dynamic-workflows-6-patterns-14-steps.md) — the automated W2S researcher used exactly the fan-out-and-synthesize + loop-until-done patterns described there. The $18,000 compute + 800 agent-hours is what those patterns cost at research scale.
- [Meta's AI Bot Instagram Account Takeover](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/metas-ai-bot-instagram-account-takeover.md) — Project Glasswing finding 10,000+ critical vulnerabilities is the flip side of the same coin: AI accelerating offense and defense simultaneously. The bottleneck has shifted to the patching infrastructure, not the finding.
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — Anthropic's "virtual lab" future (scenario 2) is an organization-scale cattle fleet. The 800-fix API error reduction is exactly what cattle looks like in production: tasks a human would never attempt because of context overhead, executed at scale by agents that don't accumulate debt.
