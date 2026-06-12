# Claude Fable 5: Building Self-Improving Agent Systems

**Source:** https://x.com/0xCodez/status/2063992615721435154 (pasted content by @0xCodez)
**Saved:** 2026-06-11
**Tags:** ai, tools, orchestration, agentic-ai, prompting, infrastructure

> **Model context verified:** Claude Fable 5 launched June 9, 2026 — first publicly available Mythos-class model. Free on Pro/Max/Team/Enterprise through June 22, 2026; credits-only after June 23. $10/million input, $50/million output tokens. 1M context, 128k max output. 30-day mandatory data retention. Safety classifiers block cyber/bio/chem/distillation requests and fall back to Opus 4.8. Mythos 5 (same capability, no safety classifiers) remains Glasswing-only.

---

## TL;DR
Fable 5 is not a faster chat tool — it's the substrate for a self-improving system you build around it. The model is stateless; the system around it compounds. Four-layer compound stack: primitives → orchestration (/goal, Dynamic Workflows, Routines) → memory (state files, Skills, knowledge bases) → self-improvement (vision verification, eval loops, rule distillation). Each run leaves the next run smarter. The measurable Anthropic experiment: Fable 5 with a verifier sub-agent achieved 73% verification coverage vs. Opus 4.7's 17% on the Continual Learning Bench.

## What "Self-Improving" Actually Means

**Self-learning (not real, not today):** The agent updates its own weights. No publicly available model does this in production. Recursive self-improvement is the long-term direction Anthropic warned about in May 2026, not what's shipping now.

**Self-improving (real, buildable today):** The *system around the agent* compounds. Sessions write lessons to memory. Skills accumulate edge cases. State files grow verified facts. Eval loops refine prompts. The model stays the same; the environment it runs in gets sharper.

Anthropic's own framing: "Rather than directly prompting and steering Fable 5, it's often better to design loops that let the model self-correct in response to environment feedback (e.g., /goal or Outcomes) and manage its own context (e.g., via memory)."

## The Four-Layer Compound Stack

```
Layer 4: Self-improvement
  Vision self-checks, eval loops, rule distillation
  Agent grades output → refines Skill → writes lesson to memory
  The loop that closes

Layer 3: Memory
  State files, Skills, Knowledge Bases
  What makes tomorrow's session resume instead of restart

Layer 2: Orchestration
  /goal and Outcomes for self-correcting loops
  Dynamic Workflows for multi-step orchestration
  Routines for laptop-off cloud runs

Layer 1: Primitives
  Fable 5 itself, sub-agents, worktrees, tools
  Raw capability — what most people use today
```

Every output from Layer 1 flows up through Layer 4, gets graded and distilled, and writes back to Layer 3. Tomorrow's run at Layer 1 inherits the sharpened environment. The model is stateless; the system isn't.

## Model Routing by Task (Cost-Capability Matrix)

| Model | Role | When to use |
|-------|------|-------------|
| Fable 5 | Orchestrator | Planning across days, delegating, vision self-check, rule distillation. Use where "days at a time" earns its pricing. |
| Opus 4.8 | Hard-but-bounded subtasks | Architecture decisions, complex debugging, deep code reviews. Also: automatic fallback for Fable 5 classifier blocks. |
| Sonnet 4.6 | High-volume workers | Lint passes, simple refactors, test scaffolding, doc updates. Bulk of fan-out work. |
| Haiku 4.5 | Grader sub-agents | Independent verifiers, cheap classifiers. Low cost, own context window. |

**Production pattern:** Orchestrator on Fable 5 → workers on Sonnet 4.6 → graders on Haiku 4.5 → fallback to Opus 4.8 on classifier blocks.

Fable 5 costs ~5× what Opus 4.8 does per token. Route by complexity, not by default.

## The Two Goal-Loop Primitives

Both share the same shape: independent grader checks work, not-met verdict starts next iteration, loop exits when grader passes.

**`/goal` (Claude Code, in-session):**
- Plain text goal, model grader, in-terminal feedback
- Use when work happens locally and you want a quick measurable end state
- Best for: hands-on coding, debugging flaky tests, refining a single file

**Outcomes (CMA, cloud-hosted):**
- File-based rubric with gradable criteria, sub-agent grader, hard `max_iterations` bound
- Use when work needs to run for hours/days on Anthropic-hosted infrastructure
- Best for: ML training, long-running migrations, multi-day research

**The structural key to both:** the agent that wrote the code is not the agent that grades it.

## Verifier Sub-Agent vs. Self-Critique (Measured)

From Anthropic's Parameter Golf experiment, Fable 5 with an independent verifier vs. self-critique:

- Fable 5 with verifier: larger structural changes (architecture-level, not constant tweaks), pushed through negative intermediate results to continued investigation
- Opus 4.7 without verifier: first experiment produced a small win, nearly everything that followed used the same template (adjust a scalar, measure, keep if positive) — safer, not better
- Fable 5 with verifier achieved ~6× more improvement than Opus 4.7

Mechanism: a model evaluating its own output sees its own reasoning trail and prefers conclusions consistent with what it already wrote. A separate model sees only the artifact and the rubric — no skin in the game.

## The 5-Stage Memory Progression (Continual Learning Bench)

From Anthropic's CLB 1.0 experiment — each stage is a structural move:

1. **Fail** — document failure with enough detail to be useful later
2. **Investigate** — figure out why the failure happened before moving on
3. **Verify** — turn diagnosis into a checked fact, not a guess
4. **Distill** — turn verification into a general rule that applies beyond the specific case
5. **Consult** — on the next task, read the rule instead of re-deriving

**Measured model differences on SQL exploration:**
- Sonnet 4.6: exits at step 1. Memory is a list of failure notes and open guesses. Rarely consults prior notes.
- Opus 4.7: exits at step 3. Creates schema reference with uncertainty flagged. Verification coverage: median ~17%.
- Fable 5: tends to complete the full progression. Verification coverage reached 73% (22/30). Distills general rules that help with future tasks.

## The State File Structure

```markdown
# Project memory · [project name]

## Verified facts   ← stage 3 output — stop guessing about these
- [fact]: [how it was verified] [date]

## General rules    ← stage 4 — consult before re-deriving
- [rule]: [when it applies]

## Open failures (investigate next session)   ← stages 1-2 in progress
- [date]: [failure description]. Hypothesis: [hypothesis].
  Reproduction steps in [file].

## Lessons learned  ← stage 4 distillations from post-mortems
- [lesson]: [context]

## Last session     ← stage 5 — resume pointer
[date] · [what was done] · Next: [specific next action]
```

**Two operational rules:**
1. **Write before walking away.** Every session ends by updating STATE.md. No write = next session restarts from zero.
2. **Read at session start.** Every session begins by reading STATE.md and relevant Skills. Without this, Sonnet-class memory behavior shows up even in Fable 5.

## Skills That Compound

STATE.md is project memory. Skills are procedural memory — the "how to do this kind of thing" that applies across projects.

A Skill that has been compounding for two weeks looks different from a fresh one. New sections appear: known failure modes, rules from post-mortems, anti-patterns from production. The Skill is no longer static instructions — it's an accumulating record of what was actually learned.

**The compounding contract:** every confirmed lesson goes into a Skill (not just STATE.md). STATE.md is project-scoped. Skills live in `~/.claude/skills/` and travel with you across projects.

## Routines — Laptop-Off Cloud Runs

Launched April 14, 2026 (research preview). Saved Claude Code configurations that run on Anthropic-managed cloud infrastructure on a trigger. Your laptop can be off.

Three trigger types for self-improvement patterns:
- **Schedule** — morning briefing pattern. Daily at 7AM: re-run eval suite, distill new failure modes into Skills, write digest to Slack.
- **API** — "fire on event." CI fails → investigate Routine. Sentry alert → triage Routine.
- **GitHub event** — "learn from real work." On PR open → evaluate against latest Skills. On merge → write new patterns back to Skill.

## Vision Self-Verification

One of Fable 5's headline capabilities: checks its own UI against the goal using vision.

Pattern:
1. Maker sub-agent writes UI code, renders to screenshot
2. Verifier sub-agent reads screenshot with vision, compares against goal description + design tokens in Skill + previous screenshot in STATE.md
3. Verdict: match → mark complete. Mismatch → describe gap, hand back to maker with structured diff

## The Mythos Safety Boundary

Fable 5's classifiers block responses in: **cybersecurity, biology, chemistry, model distillation**. Falls back to Opus 4.8 automatically. Documented. Not a bug.

Design implications for autonomous systems:
- If your system touches security tooling (SAST, exploit research, pentest logic, some code review), architect for the fallback explicitly
- A loop that silently fails on a classifier block looks identical to a loop that fails on a real error
- Skills should document which task types may hit the classifier
- Audit the 319-page system card before production deployment
- 30-day mandatory data retention for safety monitoring (both Fable 5 and Mythos 5)

**Design principle:** treat the safety boundary as a known fallback, not a failure mode.

## Availability (as of June 2026)

- **Through June 22:** included in Pro, Max, Team, and seat-based Enterprise plans at no extra cost
- **From June 23:** usage credits required until subscription availability is restored
- API, AWS Bedrock, Vertex AI, Microsoft Foundry: generally available
- `claude-fable-5` API model string
- Mythos 5 (no classifiers): Glasswing-only

## Questions & Gaps
- The 73% verification coverage figure from CLB — what was the task distribution? How does it generalize beyond SQL exploration?
- The Parameter Golf ~6× improvement claim — what was the baseline and over what number of experiments?
- Routines are in "research preview" as of April 2026 — what's the production GA timeline?
- The 30-day mandatory data retention for safety classifiers — what are the actual retention policies for different deployment tiers (API vs. Enterprise vs. subscription)?
- How does Fable 5's context management interact with NEEDLE's six-step loop? If a worker's context grows across a long bead, does Fable 5's "manages its own context" capability reduce the need for context resets?

## Related Notes
- [Anthropic Recursive Self-Improvement](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/anthropic-recursive-self-improvement.md) — the RSI piece from May 2026 is the Anthropic warning this article references. Fable 5 is the "safe enough for general release" version of what that paper worried about. Reading both together gives the full context.
- [Loop Engineering — Addy Osmani](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-addy-osmani.md) — the five building blocks Osmani describes (automations, worktrees, skills, connectors, sub-agents) map directly onto the four-layer compound stack. This article is essentially "loop engineering with Fable 5 as the orchestrator."
- [Loop Engineering: 14-Step Roadmap](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-14-step-roadmap.md) — the prior @0xCodez synthesis. This note is the same author's Fable 5-specific update to that framework. The core architecture is identical; the new additions are the 5-stage memory progression, vision verification, and Routines.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — NEEDLE implements the same four-layer stack at fleet scale. NEEDLE's `.beads/learnings.md` = STATE.md. NEEDLE's CLAUDE.md = Skills. NEEDLE's Reflect strand = the self-improvement loop. The Fable 5 compound stack and NEEDLE are convergent implementations of the same architecture.
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — the verifier sub-agent pattern here is the same principle. The Continual Learning Bench data is the empirical confirmation: 73% vs. 17% verification coverage is the measured cost of letting the maker grade its own work.
