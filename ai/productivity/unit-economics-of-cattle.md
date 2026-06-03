# The Unit Economics of Running Cattle

**Source:** https://jedarden.com/notes/unit-economics-of-cattle/
**Saved:** 2026-06-03
**Tags:** ai, productivity, tools, orchestration, agentic-ai, economics

---

## TL;DR
The pet model's hidden cost control is your attention — you're the rate limiter. Cattle removes that control. You must replace it with three explicit mechanisms before workers go headless: a spend-capping proxy between workers and providers, per-task execution bounds (time/tokens/iterations), and outcome-denominated cost metrics (cost per closed task, not cost per token).

## Key Concepts & Terms
- **Cost per closed outcome**: The correct unit of measure — not tokens burned but closed beads / passing tests / merged PRs per dollar. A worker burning 800K tokens to close 5 tasks is 4x cheaper per outcome than one burning 200K tokens to close 1.
- **claude-governor**: Jed's Rust proxy sitting between workers and api.anthropic.com. Enforces: hard daily/weekly/monthly spend caps; concurrency semaphore (limits simultaneous in-flight calls); quota gating against subscription windows.
- **Subscription vs. metered API**: Subscriptions (Anthropic Max, Z.AI Max) cap your downside — fixed monthly cost, quota window, cut off if exceeded. Metered API is uncapped — flexible for bursts but a bug in retry logic can produce five-figure overnight invoices. Run a portfolio of both; subscription for steady throughput, metered for surge capacity.
- **Cheap restart reflex**: The pet operator's instinct to kill a stuck agent and start fresh. In cattle, automated via three bounds: time-bounded (wall-clock timeout), token-bounded (output token cap per task), iteration-bounded (model call count cap per task). Any of the three trips → kill the worker, release the task, loop.

## Main Arguments & Takeaways
- **The pet model's hidden cost control is your attention**: You pace token-heavy ops, you notice runaways, you self-throttle on quota. All three disappear in cattle. Something else must replace all three.
- **Tokens are the wrong optimization target**: Trimming context windows on a productive worker to reduce token cost kills the outcome rate, making the worker more expensive per task even as it appears cheaper per token.
- **Quota observability is asymmetric**: Anthropic exposes weekly limit headers on every API response. Z.AI exposes nothing programmatic — no quota endpoint, no rate-limit headers. The proxy must model quota independently, count outflow at the proxy layer, and treat its own ledger as ground truth.
- **Three execution bounds per task convert "worker decides when to stop" to "orchestrator decides when to stop"**: A worker killed early may have been about to succeed — that's fine, the task retries. The cost of a wasted bounded execution is one bounded run. The cost of an unwatched runaway is unbounded.
- **Run a portfolio**: Subscription = bounded throughput at known cost. Metered API = surge capacity with a hard governor cap. Never run metered API without the governor enforcing a ceiling.

## Notable Quotes
> "If you have not built that, you do not have a cattle system. You have a pet system with the supervision removed, which is a different thing. It looks the same right up until the bill arrives."

## Questions & Gaps
- What's a reasonable starting budget for time/token/iteration bounds on a coding task? Any empirical data from NEEDLE production runs?
- The three-bound kill mechanism — does this interact badly with tasks that are legitimately long (e.g. large refactors)?
- Z.AI's lack of programmatic quota exposure — is this a known issue with other providers? How do you handle a provider that gives no signal before the wall?

## Related Notes
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the framing this cost post extends.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the outcome classification system that makes cost-per-outcome computable.
- [I Stopped Hitting Claude's Usage Limits](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/stopped-hitting-claude-usage-limits-10-things.md) — user-level token optimization; complementary to this fleet-level governance approach.
