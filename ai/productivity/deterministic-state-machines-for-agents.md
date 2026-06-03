# Deterministic State Machines for Non-Deterministic Agents

**Source:** https://jedarden.com/notes/deterministic-state-machines-for-agents/
**Saved:** 2026-06-03
**Tags:** ai, productivity, tools, orchestration, agentic-ai, infrastructure

---

## TL;DR
The control structure that lets twenty headless workers run unattended: a rigid, exhaustive state machine that wraps the fuzzy agent. Every outcome the agent can produce has an explicit handler — no wildcards, no implicit fallbacks. The orchestrator is deterministic; the agent is non-deterministic. Together they are a system that routes.

## Key Concepts & Terms
- **Exhaustive outcome table**: The core constraint — if an outcome can happen, it has a handler; if it doesn't have a handler, it cannot happen. Every state transition has an explicit handler. No `match _ => continue`.
- **Race lost**: When two workers both try to claim the same task and one loses. First-class outcome, not a failure or an error. The loser skips and tries the next available task.
- **Queue empty**: When a worker has nothing to do. First-class outcome that triggers strand escalation — search other workspaces, do cleanup, propose alternatives for blocked tasks. Not a non-event; it's a signal.
- **Crash vs. failure**: Failure = agent tried and produced nothing useful (exit code 1). Crash = agent died without saying anything (exit code >128). Different handlers: failure increments retry count and tries again; crash creates an alert task and may indicate something fundamentally wrong.
- **Strand escalation**: The sequence NEEDLE runs when the queue is empty in the current workspace — search other workspaces, cleanup, alert if all strands exhausted.

## Main Arguments & Takeaways
- **Six-step loop**: SELECT (next task) → CLAIM (atomic SQLite transaction) → BUILD (deterministic prompt from task definition) → DISPATCH (agent adapter) → EXECUTE (wait for exit code + disk writes) → OUTCOME (classify + route).
- **The game is entirely in step 6 (OUTCOME)**. The first five are mechanical; any orchestration framework can do them.
- **Exhaustiveness discipline costs discipline against wildcards**. Rust's exhaustive match is a compiler error; other languages tempt you toward a wildcard handler. Wildcards are how state machines silently grow undefined behavior.
- **Schema-first, not caller-first**: When a new outcome appears, add it to the type first, then let the compiler find all the call sites that need to handle it.
- **The state machine is the contract; agents are the implementation**: NEEDLE doesn't know if the worker is Claude Code, OpenCode, or Aider. It knows the worker takes a prompt and returns an exit code. Agent adapters are YAML files.
- **Two disciplines together**: Cattle says the agent must be replaceable. State machine says the orchestrator must be exhaustive. Either alone is a tarpit; together they are a system that runs unattended.

## Notable Quotes
> "If an outcome can happen, it has a handler. If it doesn't have a handler, it cannot happen."

> "You stop debugging individual workers and start debugging the outcome distribution — which row of the table is firing more often than it should?"

## Questions & Gaps
- How do you handle outcomes that require human judgment — e.g. "agent completed but output is ambiguous"? Is there a pattern for routing those to async human review?
- The article suggests exit codes are too coarse. What would a structured outcome envelope look like in practice, and what format would agent adapters use?
- At 20 workers running the same six-step loop, how does SQLite perform under concurrent claim contention?

## Related Notes
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the mental model this state machine architecture serves.
- [Unit Economics of Running Cattle](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/unit-economics-of-cattle.md) — the cost governance layer that wraps the same system.
