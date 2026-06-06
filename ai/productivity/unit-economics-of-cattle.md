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


## Code Examples

### The Three Execution Bounds (.needle.yaml)
```yaml
# These three bounds convert "worker decides when to stop"
# to "orchestrator decides when to stop"
worker:
  max_execution_minutes: 30      # wall-clock timeout per bead
  max_output_tokens: 100_000     # output token cap per bead
  max_model_calls: 50            # model invocation cap per bead
  # Any bound tripped → kill worker → release bead → retry
  max_retries: 3                 # before marking bead permanently failed
```

### Claude-Governor Config (spend cap proxy)
```yaml
# governor.yaml — sits between workers and api.anthropic.com
api_key: "sk-ant-..."           # real key, only governor sees it
proxy_port: 8080

limits:
  daily_usd: 50                 # hard daily spend cap
  weekly_usd: 200               # hard weekly spend cap
  concurrent_requests: 20       # semaphore — max simultaneous in-flight calls

quota:
  # Anthropic exposes weekly limit headers — governor reads them
  anthropic_weekly_tokens: 10_000_000
  # Z.AI exposes nothing — governor must track independently
  zai_weekly_tokens_ledger: true  # count outflow at proxy layer

# Workers point at proxy, not Anthropic directly:
# ANTHROPIC_BASE_URL=http://localhost:8080
```

### Cost Per Closed Outcome (the right metric)
```python
# Wrong metric: cost per token
cost_per_token = total_spend / total_tokens

# Right metric: cost per closed bead
cost_per_outcome = total_spend / beads_closed

# A worker burning 800K tokens to close 5 beads:
worker_a = 800_000 * 0.000015 / 5  # = $2.40 per closed bead

# A worker burning 200K tokens to close 1 bead:
worker_b = 200_000 * 0.000015 / 1  # = $3.00 per closed bead

# Worker A is 4x cheaper per outcome even though it burns 4x more tokens.
# Optimizing for token count would have made you choose Worker B.
```

### Detecting a Runaway (what to instrument)
```python
# In your governor or monitoring layer — alert before the bill arrives
def check_worker_health(worker_id: str, bead_start_time: float, tokens_this_bead: int, model_calls_this_bead: int):
    elapsed_minutes = (time.time() - bead_start_time) / 60

    if elapsed_minutes > 30:
        alert(f"Worker {worker_id}: time bound exceeded ({elapsed_minutes:.0f}min)")
        kill_and_release(worker_id)

    if tokens_this_bead > 100_000:
        alert(f"Worker {worker_id}: token bound exceeded ({tokens_this_bead:,} tokens)")
        kill_and_release(worker_id)

    if model_calls_this_bead > 50:
        alert(f"Worker {worker_id}: iteration bound exceeded ({model_calls_this_bead} calls)")
        kill_and_release(worker_id)
```
## Questions & Gaps
- What's a reasonable starting budget for time/token/iteration bounds on a coding task? Any empirical data from NEEDLE production runs?
- The three-bound kill mechanism — does this interact badly with tasks that are legitimately long (e.g. large refactors)?
- Z.AI's lack of programmatic quota exposure — is this a known issue with other providers? How do you handle a provider that gives no signal before the wall?

## Related Notes
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the framing this cost post extends.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the outcome classification system that makes cost-per-outcome computable.
- [I Stopped Hitting Claude's Usage Limits](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/stopped-hitting-claude-usage-limits-10-things.md) — user-level token optimization; complementary to this fleet-level governance approach.
