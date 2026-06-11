# Loop Engineering — Addy Osmani (Primary Source)

**Source:** https://x.com/addyosmani/status/2063992615721435154 (pasted content by @addyosmani)
**Saved:** 2026-06-10
**Tags:** ai, productivity, tools, orchestration, agentic-ai, prompting

> **Note:** This is the primary source essay. A synthesis of this material into a 14-step roadmap was saved separately as `ai/productivity/loop-engineering-14-step-roadmap.md`. Read this for Osmani's voice, caveats, and framing; read the roadmap for structured implementation steps and code examples.

---

## TL;DR
Loop engineering is replacing yourself as the person who prompts the agent — building a system that does the prompting instead. Five building blocks (automations, worktrees, skills, connectors, sub-agents) plus persistent memory. Both Claude Code and Codex now ship all five. The leverage point moved from writing better prompts to designing better loops — but the loop doesn't delete you from the work, it changes the nature of what you do.

## Key Quotes That Anchor the Piece

> "@steipete: You shouldn't be prompting coding agents anymore. You should be designing loops that prompt your agents."

> "@bcherny (Head of Claude Code at Anthropic): I don't prompt Claude anymore. I have loops running that prompt Claude and figuring out what to do. My job is to write loops."

> "The agent forgets, the repo doesn't."

> "Build the loop. Stay the engineer."

> "Designing the loop is the cure when you do it with judgment and the accelerant when you do it to avoid thinking. Same action, opposite result."

## What's Distinct in This Version vs. the 14-Step Synthesis

The @0xCodez roadmap is a structured breakdown of this essay's content with added code examples and a decision framework. This original has things the synthesis doesn't:

**Osmani's genuine skepticism:** "I'm skeptical and you absolutely have to be careful about token costs." He warns "slop concerns are valid." This hesitation is absent in the synthesis, which reads more bullish.

**The comprehension debt framing is personal:** "The faster the loop ships code you did not write, the bigger the gap between what exists and what you actually get. That's comprehension debt and a smooth loop just makes it grow faster unless you read what the loop made." He frames this as a risk that gets *sharper* as the loop improves.

**Cognitive surrender:** "When the loop runs itself it's very tempting to stop having an opinion and just take whatever it gives back." Named explicitly as the danger of the optimized state.

**The closing caveat about different outcomes:** "Two people can build the exact same loop and get completely opposite results. One uses it to move faster on work they understand deeply. The other uses it to avoid understanding the work at all. The loop doesn't know the difference. You do."

**The "orchestration tax" and "intent debt" references:** Osmani references his prior essays — the orchestration tax (your review bandwidth decides how many parallel agents you can run, not the tool) and intent debt (every session starts cold; skills are intent written down so it doesn't cost you again on every run). These are distinct concepts worth following up.

## The Five Building Blocks (Osmani's Framing)

### 1. Automations — The Heartbeat
Discovery and triage by themselves on a schedule. Claude Code: `/loop`, cron scheduling, hooks, GitHub Actions. Codex: Automations tab with Triage inbox for findings.

**The two in-session primitives:**
- `/loop` — re-runs on cadence regardless of state
- `/goal` — keeps going until a verifiable condition is actually true, checked by a *separate small model* (the maker/checker split applied to the stop condition)

### 2. Worktrees — Parallel Without Collision
Separate working directory per agent on its own branch, same repo history. "The worktrees take away the mechanical collision but YOU are still the ceiling. Your review bandwidth decides how many you can actually run, not the tool."

### 3. Skills — Intent Written Down Once
"An agent starts every session cold and will fill any hole in your intent with a confident guess. A skill is that intent written down on the outside." Without skills, the loop re-derives project context from zero every cycle. With skills, intent compounds.

Key distinction: the skill is the authoring format; the plugin is how you ship it across repos.

### 4. Plugins and Connectors (MCP)
"The difference between an agent that says 'here is the fix' and a loop that opens the PR, links the Linear ticket, and pings the channel once CI is green by itself."

### 5. Sub-Agents — Maker Away from Checker
"The model that wrote the code is way too nice grading its own homework." The loop runs while you are not watching — a verifier you actually trust is the only reason you can walk away.

Burn tokens on sub-agents only where a second opinion is worth paying for.

## One Complete Loop Shape (Osmani's Own)

```
Morning automation fires on repo
  → Triage skill reads: CI failures, open issues, recent commits
  → Writes findings to markdown/Linear board
  
For each actionable finding:
  → Opens isolated worktree
  → Sub-agent A: drafts the fix
  → Sub-agent B: reviews against project skills + existing tests
  
Connectors: open PR, update ticket
Anything the loop can't handle → lands in triage inbox for human

State file: remembers what was tried, what passed, what's still open
Tomorrow's run picks up where today stopped
```

"You designed it one time. You did not prompt any of those steps."

## What the Loop Still Doesn't Do

Three problems that get *sharper* as the loop gets better:

1. **Verification is still on you.** "Done" is a claim, not a proof. Even a verifier sub-agent doesn't change that — it makes the claim more credible, not certain. "Your job is to ship code you confirmed works."

2. **Comprehension debt grows faster with a smooth loop.** The gap between what the repo contains and what you understand. A smooth loop just accelerates the accumulation unless you actively read what it made.

3. **Cognitive surrender is the comfortable posture.** And comfort is the risk. "The loop doesn't know the difference between someone using it to move faster on work they understand deeply and someone using it to avoid understanding the work at all."

## Osmani's Honest Closing Position

"If I weren't reviewing the code myself or if I relied entirely on automated loops to fix it, my product's quality would suffer. I'd likely end up stuck in a downward spiral, continuously digging myself into a deeper hole. That said, go ahead and set up your loops, but don't forget that prompting your agents directly is still effective. It's all about finding the right balance."

This is notably more cautious than the @0xCodez synthesis, which reads more prescriptively.

## Related Notes
- [Loop Engineering: 14-Step Roadmap](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-14-step-roadmap.md) — structured synthesis of this essay with implementation steps, decision framework, and code examples. Read that for the "how"; read this for the "why" and the caveats.
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — Osmani's maker/checker split and `/goal` with independent checker are the same principle at the architectural level.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — Jed Arden's state machine is the production implementation of the loop Osmani describes. Exhaustive outcome table = the verifier sub-agent pattern codified into a compiler-enforced contract.
- [Agentic Coding Ladder](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/agentic-coding-ladder.md) — Osmani's "build the loop, stay the engineer" is Jed's rung 8/9 framing: the ladder isn't about ascending as high as possible, it's about choosing the right rung per task.
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — Osmani references "agent harness engineering" as a cousin to loop engineering. That Anthropic post is the harness piece he means.
