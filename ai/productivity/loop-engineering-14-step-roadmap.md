# Loop Engineering: The 14-Step Roadmap from Prompter to Loop Designer

**Source:** https://x.com/0xCodez/status/2063295755104747553 (pasted content by @0xCodez)
**Saved:** 2026-06-10
**Tags:** ai, productivity, tools, orchestration, agentic-ai, prompting

> Sources cited in the article: Anthropic engineering docs, Addy Osmani's long-form on loop engineering, AlphaSignal analysis, Geoffrey Huntley (Ralph Wiggum loop), Cherny.

---

## TL;DR
Loop engineering is replacing yourself as the prompter by building a small system that finds work, hands it to the agent, checks the result, records what happened, and decides next moves — on its own. The 14-step roadmap covers three tiers: determine if you actually need a loop (most developers don't yet), learn the five building blocks, then build the smallest loop that works without hurting you. The leverage point has moved from writing better prompts to designing better loops.

## Key Concepts & Terms
- **Loop engineering**: Building a system that prompts the agent rather than prompting the agent yourself. The system finds the work, dispatches it, verifies output, records state, and determines the next move — unattended.
- **Automation**: The heartbeat of a loop — what makes it recur rather than run once. Fires on schedule, event, or trigger. In Claude Code: `/loop` (session-scoped), Desktop scheduled tasks (restart-survival), Routines (cloud runs while laptop is off). In Codex: the Automations tab.
- **`/loop`**: Re-runs on a cadence regardless of state. Use when you want regular checks.
- **`/goal`**: Keeps going until a condition you wrote is actually true — verified by a separate small model, not the agent that did the work. The stop condition has its own checker.
- **Worktree**: A separate working directory on its own branch, sharing repo history. Prevents file collisions when multiple agents run in parallel. Claude Code: `--worktree` flag and `isolation: worktree` on subagents. Codex: built-in.
- **Skill (SKILL.md)**: Persistent project knowledge written once, read on every loop run. Prevents re-deriving context from zero every cycle. Same format across Codex and Claude Code: a folder with SKILL.md + optional scripts and references.
- **Connector (MCP)**: Lets the agent read your issue tracker, query a database, hit a staging API, post to Slack. Built on Model Context Protocol. Both tools speak MCP. Converts "here is the fix" into "opened the PR, linked the ticket, pinged the channel."
- **Sub-agent (maker/checker split)**: The most useful structural element in a loop. The agent that wrote the code is "too nice grading its own homework." A separate agent with different instructions — sometimes a different model — runs the verification. Defined in `.codex/agents/` (TOML) or `.claude/agents/`.
- **State file**: A markdown file, Linear board, or JSON state that lives outside any single conversation and records what's done and what's next. The agent forgets; the file does not. Tomorrow's run resumes instead of restarting.
- **Ralph Wiggum loop**: Named by Geoffrey Huntley. An agent emits a completion token before actually finishing, the loop exits on a half-done job, and keeps spending. The failure mode of soft completion conditions — "done" defined by the agent's judgment rather than a test, build, or linter.
- **Comprehension debt**: The growing gap between what the repository contains and what you understand. Accumulates faster as the loop ships code you didn't write. The bill arrives when you have to debug a system no one on the team has read.
- **Cognitive surrender**: The pull to stop forming an opinion and accept whatever the loop returns. Designing a loop with judgment = accelerant. Designing it to avoid thinking = the same action with the opposite result.
- **Cost per accepted change**: The only metric that matters. Not tokens spent, not tasks attempted, not loops scheduled. If accepted-change rate is below 50%, the loop is generating review work faster than it's reducing it.

---

## Tier 1 — The Why and the Test

### The 4-Condition Test (Run Before Building Anything)

A loop earns its cost only when all four conditions hold. Miss one and it costs more than it returns:

| Condition | What it means | Why it matters |
|-----------|--------------|----------------|
| **Task repeats** | At least weekly | A loop amortizes its setup cost across many runs. One-time job → a good prompt is faster |
| **Verification is automated** | Test suite, type checker, linter, build | Without an automated gate, you're back reading every diff — the job the loop was supposed to remove |
| **Token budget can absorb waste** | Loops re-read context, retry, explore | Scales with budget. Obvious to people with free tokens; reckless on a metered $20 plan |
| **Agent has senior engineer's tools** | Logs, reproduction environment, can run code it writes | Without these, the loop iterates blind |

**Who benefits in practice:**
- Teams with repetitive, machine-checkable work and budget to run it (CI triage, dependency bumps, lint-and-fix, issue-to-PR on codebases with strong test coverage)
- Codebases where a junior engineer could do the task from a checklist and a test suite would catch mistakes

**Who should skip it today:**
- Solo builders on consumer plans — token bill arrives before productivity gain
- Anyone with no automated verification — a loop with no real check is the agent agreeing with itself
- Teams whose constraint is review capacity, not typing speed — a loop generates more code; if review was the bottleneck, it just lengthens the queue

### The 30-Second Loop Check (Per Task)

Before converting any specific task to a loop, check all five:

```
☐ Task happens at least weekly
☐ A test, type check, build, or linter can reject bad output
☐ Agent can run the code it changes (reproduction environment exists)
☐ Loop has a hard stop (token budget, iteration count, or time limit)
☐ A human reviews before merge, deploy, or dependency changes
```

**Good first loops:** CI failure triage, dependency bump PRs, lint-and-fix passes, flaky test reproduction, issue-to-PR on code with strong tests.

**Bad first loops:** Architecture rewrites, auth or payments code, production deploys, vague product work, anything where "done" is a judgment call.

---

## Tier 2 — The Five Building Blocks

### Block 1: Automations (The Heartbeat)

```python
# Claude Code — session-scoped cadence
> /loop 30m /goal All tests in test/auth pass and lint is clean.
  Scan src/auth for new failures, propose fixes in claude/auth-fixes,
  open draft PR when goal condition holds.

# /loop = re-runs on cadence regardless of state
# /goal = runs until condition is true, verified by a SEPARATE checker model
```

The `/goal` condition being checked by a separate model is the key primitive. The agent that wrote the code is not the one deciding whether it's done.

### Block 2: Worktrees (Parallel Without Chaos)

```bash
# Claude Code — open session in isolated worktree
claude --worktree feature/auth-fix

# Subagent with auto-cleanup
{
  "isolation": "worktree",
  "cleanup": "on_complete"
}
```

Worktrees eliminate mechanical file collisions. Your review bandwidth — not the tool's parallelism limit — is the actual ceiling on how many agents you can run.

### Block 3: Skills (Project Knowledge Written Once)

```yaml
# .claude/skills/ci-triage/SKILL.md
name: ci-triage
description: >
  Classify CI failures by root cause (env, flake, real bug, dependency, infra),
  draft fixes for the easy ones, escalate the rest.
  Trigger whenever a workflow run fails or on the morning triage loop.
---

## Classification rules
- env: missing secret, wrong env var → human
- flake: passes on retry without code change → retry once, then file
- bug: deterministic failure tied to recent commit → draft fix
- dependency: failure tied to version bump → draft rollback
- infra: timeout, OOM, runner issue → escalate

## Never do
- Disable failing tests — always escalate instead
- Modify CI config without human approval
- Touch src/payments/ or src/billing/

## State
Update STATE.md after each run: files checked, classifications, PRs opened, escalations.
```

Without skills, a loop re-derives your entire project context from zero every cycle. With skills, intent compounds — conventions, build steps, tribal knowledge — written once, read every run.

### Block 4: Connectors (Loop Touches Real Tools via MCP)

Fastest ROI connectors for loop work:
1. **GitHub** — create branches, open PRs, comment on issues, react to webhook events
2. **Linear / Jira** — update tickets as loop progresses, link PRs, auto-close on verification pass
3. **Slack** — post triage results, ping humans on escalations, summarize overnight runs
4. **Sentry / error tracker** — investigate live alerts, draft fixes for high-frequency errors

### Block 5: Sub-Agents (Keep Maker Away from Checker)

```toml
# .codex/agents/security-reviewer.toml
name = "security-reviewer"
description = "Review PRs for OWASP Top 10 violations and secret exposure"
model = "o3"
reasoning_effort = "high"
instructions = """
You are a skeptical security reviewer. You have never seen the code
you are reviewing before. Your job is to find what the author missed.
Never approve unless tests pass AND no security violations found.
"""

# .codex/agents/explorer.toml  
name = "explorer"
description = "Fast read-only exploration to identify what needs fixing"
model = "haiku"
reasoning_effort = "low"
instructions = "Read-only. Identify candidates for fixing. Do not write code."
```

The maker-checker split applied to the stop condition: the model that wrote the code does not decide it's done. A separate model with no exposure to the maker's reasoning makes that call.

---

## Tier 3 — Build It Right or Don't Build It

### The Minimum Viable Loop (4 Parts)

```
1. One automation  — scheduled run with a hard stop condition
2. One skill       — SKILL.md storing project context
3. One state file  — records done / in-progress / next / lessons learned
4. One gate        — test, type check, or build that fails bad work automatically
```

**Order matters:**
1. Get one manual run reliable
2. Turn it into a skill
3. Wrap it in a loop
4. Then schedule it

Skipping ahead is how loops fail in production.

### The State File

```markdown
# Loop state · ci-triage

## Last run
2026-06-09 03:30 UTC · 7 failures classified, 3 fixes drafted, 4 escalated

## In progress
- claude/fix-auth-token-refresh — tests passing locally, awaiting CI
- claude/fix-flaky-payment-webhook — retry pattern applied, monitoring

## Completed today
- claude/bump-axios-1.7.4 → merged (CI green)
- claude/lint-fix-pass-june-9 → merged

## Escalated to humans
- src/billing/refund.ts — tests failing in 3 ways, root cause unclear
- ci/staging-runner — infra timeouts, not a code issue

## Lessons learned (write here, not in chat)
- 2026-06-08: PowerShell TLS 1.2 issue on Windows runner. Use bash.
- 2026-06-07: tests/e2e/checkout requires Stripe webhook secret in env. Skip if missing.
```

For long-running loops at risk of goal drift: pair the state file with a standing `VISION.md` or `AGENTS.md` reread each run. State tells the agent where it is; the spec tells it where to go.

### The Ralph Wiggum Loop (How Loops Fail Quietly)

Named by Geoffrey Huntley. The agent emits a completion token before actually finishing. Loop exits on a half-done job. Keeps spending.

**The three causes:**
1. No real verifier — just a second agent asked to "review" with no objective signal. Two optimists agreeing.
2. Soft completion conditions — "done" defined by agent's judgment, not a test or build.
3. No hard stops — loop continues until rate limit or you noticing, not until success is verified.

**Fix:** The gate from the MVL — something objective that can fail the work. A test that passes or fails. A build that compiles or doesn't. Not a verifier that has an opinion.

### The Three Failure Modes (Named)

| Failure mode | Cause | Mitigation |
|-------------|-------|-----------|
| **Goal drift** | Each summarization step is lossy; constraints disappear at turn 47 | Standing `VISION.md` or `AGENTS.md` reread each run |
| **Self-preferential bias** | Maker grades own homework — always "A+" | Separate verifier subagent with no exposure to maker's reasoning |
| **Agentic laziness** | Loop declares "done enough" at partial completion | `/goal` with objective stop condition checked by fresh model |

### The Security Tax

```
Threat                        Mitigation
──────────────────────────    ────────────────────────────────────────
Generated code ships          Gate must include SAST + dependency audit
  unreviewed                    + secret scanning before auto-merge
Skills as injection vectors   Audit skill sources before installing;
                                520 of 17,022 audited skills leak creds
Credentials in logs           Disable verbose logging in production loops;
                                sanitize what does get logged
Permission scope creep        Re-audit permissions every 30 days; one
                                "convenience" write permission gets forgotten
```

### The 10 Mistakes That Turn Loops Into Money Pits

1. Building without running the 4-condition test
2. No objective gate — a "reviewer" agent without a test is a second optimist
3. One agent doing both writing and verifying
4. No state file — tomorrow's run restarts from zero
5. Vague stop conditions — "done when it looks good" never holds
6. No token budget cap — ambitious loops burn 5–10× expected without one
7. Running heavy verification loops on a consumer plan
8. Auto-installing community skills without auditing (520/17,022 audited skills leak credentials)
9. Loops on judgment-call work — architecture, auth, payments, vague product
10. Not reading the diffs — comprehension debt at compound interest

## Questions & Gaps
- The article cites Anthropic's "8× more code per day" and immediately notes Anthropic itself calls it "almost certainly an overstatement." What's the actual measured productivity gain? The METR research on this question (referenced in the RSI note) would be worth cross-referencing.
- The 50% accepted-change rate threshold — where does this number come from? Is it empirically grounded or a rule of thumb?
- The "520 of 17,022 audited skills leak credentials" stat — what was the audit methodology, and over what skill registry?
- `/goal` with an independent checker model is described as the key primitive, but the article doesn't say what model is used as the checker or how to configure it. What's the actual implementation?
- Comprehension debt and cognitive surrender are named but the mitigations ("read the diffs," "spot-check the gate") are human discipline, not technical controls. Is there an automated way to flag when comprehension debt is accumulating?

## Related Notes
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — "loop engineering" is the single-developer version of the cattle model. The loop IS the orchestrator. The same four conditions Jed uses for cattle (headless execution, specifiable work, automated verification, cost governance) are the four conditions in this article's test.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the Ralph Wiggum loop failure mode is exactly what NEEDLE's exhaustive outcome table prevents. "Soft completion conditions" = missing outcome handlers. The fix is the same: an objective gate that can fail the work.
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — the maker/checker sub-agent split is this principle implemented architecturally. The article's framing ("too nice grading its own homework") quotes the same insight.
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — the state file's `VISION.md` spec reread each run is the bead body equivalent. State tells the agent where it is; the spec tells it where to go. Same discipline, different surface.
- [Claude Code Dynamic Workflows](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-code-dynamic-workflows-6-patterns-14-steps.md) — Dynamic Workflows is loop engineering with Claude writing the loop itself. The article's MVL (one automation + one skill + one state file + one gate) is the static version; Dynamic Workflows generates the loop shape per task.
- [Agentic Coding Ladder](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/agentic-coding-ladder.md) — "the leverage point moved from writing better prompts to designing the loop that prompts" is Jed's threshold between rung 7 (Bottleneck — you realize your attention doesn't scale) and rung 8 (Dispatcher — you give up being reachable mid-task).
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — NEEDLE is a production implementation of the loop engineering pattern at fleet scale. The state file → `.beads/learnings.md`. The gate → bead acceptance criteria. The skill → CLAUDE.md. The automation → `needle run`. The Ralph Wiggum fix → exhaustive outcome table.
- [gstack Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/gstack-repo-overview.md) — gstack's `/autoplan` runs CEO + design + eng + DX reviews sequentially, which is loop engineering applied to the planning phase. The ETHOS injected into every skill is the `VISION.md` this article recommends reading each run.
