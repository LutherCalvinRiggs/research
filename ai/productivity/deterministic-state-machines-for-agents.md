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


## Code Examples

### The Exhaustive Outcome Enum (Rust)
```rust
// Every possible outcome is a variant. The compiler enforces exhaustiveness.
// No `_ => continue` wildcard allowed — that's the entire discipline.
#[derive(Debug)]
enum WorkerOutcome {
    Success,           // exit 0 — validate, close bead, loop
    Failure,           // exit 1 — release bead, increment retry, loop
    Timeout,           // exit 124 — release bead, mark deferred, loop
    Crash(i32),        // exit >128 — release, create alert bead, loop
    RaceLost,          // exit 4 — exclude candidate, retry SELECT
    QueueEmpty,        // no beads — enter strand escalation
}

fn handle_outcome(outcome: WorkerOutcome, bead_id: &str, db: &Db) {
    match outcome {
        WorkerOutcome::Success       => { validate_and_close(bead_id, db); }
        WorkerOutcome::Failure       => { release_and_retry(bead_id, db); }
        WorkerOutcome::Timeout       => { release_and_defer(bead_id, db); }
        WorkerOutcome::Crash(code)   => { release_and_alert(bead_id, code, db); }
        WorkerOutcome::RaceLost      => { /* retry SELECT immediately */ }
        WorkerOutcome::QueueEmpty    => { run_strand_escalation(db); }
        // No wildcard. Adding a new outcome requires handling it here.
        // The compiler will refuse to compile until every arm is covered.
    }
}
```

### Classifying Exit Code → Outcome
```rust
fn classify_exit(code: i32) -> WorkerOutcome {
    match code {
        0        => WorkerOutcome::Success,
        1        => WorkerOutcome::Failure,
        4        => WorkerOutcome::RaceLost,
        124      => WorkerOutcome::Timeout,
        c if c > 128 => WorkerOutcome::Crash(c),
        _        => WorkerOutcome::Failure,  // unexpected non-zero → treat as failure
    }
}
```

### The Six-Step Loop (pseudocode)
```rust
loop {
    // Step 1: SELECT
    let Some(bead) = db.select_next_claimable() else {
        handle_outcome(WorkerOutcome::QueueEmpty, "", &db);
        continue;
    };

    // Step 2: CLAIM (atomic BEGIN IMMEDIATE transaction)
    if !db.claim(&bead.id, &worker_identity) {
        handle_outcome(WorkerOutcome::RaceLost, &bead.id, &db);
        continue;
    }

    // Step 3: BUILD prompt deterministically from bead + CLAUDE.md
    let prompt = build_prompt(&bead, &workspace_config);

    // Step 4 + 5: DISPATCH and EXECUTE
    let exit_code = dispatch_agent(&prompt, &agent_adapter, timeout_secs);

    // Step 6: OUTCOME — classify and route, no wildcards
    let outcome = classify_exit(exit_code);
    handle_outcome(outcome, &bead.id, &db);
}
```

### Schema-First New Outcome (how to add a new outcome safely)
```rust
// Step 1: Add the variant to the enum
enum WorkerOutcome {
    // ... existing variants ...
    RateLimited,  // new: API returned 429
}

// Step 2: The compiler now refuses to compile — every match arm must handle it.
// It will point you to every place that needs updating. Fix them all.

// Step 3: Add the classifier
fn classify_exit(code: i32) -> WorkerOutcome {
    match code {
        // ...
        42 => WorkerOutcome::RateLimited,  // custom exit code from agent adapter
        // ...
    }
}

// Step 4: Add the handler
WorkerOutcome::RateLimited => { backoff_and_release(&bead.id, &db); }
```
## Questions & Gaps
- How do you handle outcomes that require human judgment — e.g. "agent completed but output is ambiguous"? Is there a pattern for routing those to async human review?
- The article suggests exit codes are too coarse. What would a structured outcome envelope look like in practice, and what format would agent adapters use?
- At 20 workers running the same six-step loop, how does SQLite perform under concurrent claim contention?

## Related Notes
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the mental model this state machine architecture serves.
- [Unit Economics of Running Cattle](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/unit-economics-of-cattle.md) — the cost governance layer that wraps the same system.
