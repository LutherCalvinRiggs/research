# The Constellation — How Jed Arden's Tools Fit Together

**Source:** https://jedarden.com/constellation
**Saved:** 2026-07-11
**Tags:** ai, tools, infrastructure, orchestration, agentic-ai

> The canonical map of Jed Arden's fleet tooling. Six tools, one queue, one state machine. Read alongside the individual repo overview notes for deeper detail on NEEDLE, bead-forge, and beads_viewer.

---

## TL;DR
Six tools form a complete headless agent fleet: NEEDLE orchestrates the state machine, bead-forge holds the work queue with atomic claims, FABRIC records every dispatch and outcome, claude-governor gates cost and quotas, HOOP is the fleet control plane, and ARMOR is the encrypted backup proxy. Six workers, one shared queue, no coordinator — just the SELECT→CLAIM→BUILD→DISPATCH→EXECUTE→OUTCOME loop running unattended across heterogeneous agent harnesses (Claude Code, Codex, Goose, Pi).

---

## The Six Tools

| Role | Tool | GitHub |
|------|------|--------|
| **Orchestrator** | NEEDLE | github.com/jedarden/NEEDLE |
| **Work queue** | bead-forge | github.com/jedarden/bead-forge |
| **Telemetry** | FABRIC | github.com/jedarden/FABRIC |
| **Cost gate** | claude-governor | github.com/jedarden/claude-governor |
| **Control plane** | HOOP | github.com/jedarden/HOOP |
| **Backups** | ARMOR | github.com/jedarden/ARMOR |

### NEEDLE — Orchestrator
Deterministic state machine that dispatches the whole agent fleet. The six-step loop: SELECT → CLAIM → BUILD → DISPATCH → EXECUTE → OUTCOME. No dispatcher, no coordinator — just the loop, running unattended. Workers run in parallel with no central authority; coordination happens entirely through SQLite atomic transactions in bead-forge.

### bead-forge — Work Queue
Git-native work queue with atomic claims. The fleet pulls tasks from it. Each claim is a `BEGIN IMMEDIATE` SQLite transaction — exactly one worker wins, losers move on immediately. Success closes a bead, failure re-queues it, timeout defers it.

### FABRIC — Telemetry
Fleet telemetry layer. Every dispatch and every outcome lands here. The full audit trail of what each worker did, what result it produced, and how long it took. The observability layer that makes the headless fleet debuggable.

> **New tool — not previously in our notes.** FABRIC is the answer to concept 19 (Tracing) from the 30 Core Agentic Engineering Concepts note. The existing NEEDLE and bead-forge notes mentioned OTel telemetry as optional; FABRIC is the dedicated fleet telemetry solution.

### claude-governor — Cost Gate
Cost and quota gate sitting between workers and model providers. Workers never hold real API keys — they call claude-governor, which holds the credentials and enforces spend limits. If a retry loop bug starts burning tokens, claude-governor stops it before a five-figure invoice arrives.

### HOOP — Control Plane
Fleet control plane for NEEDLE workers. New tool not previously documented. Likely handles fleet-level commands: pause/resume workers, inspect worker state, scale the fleet up or down, override strand behavior at runtime without restarting workers.

> **New tool — not previously in our notes.** HOOP fills the gap between "workers run headlessly" and "you need a way to control them without killing and restarting tmux sessions."

### ARMOR — Backup Proxy
Encrypted S3 backup proxy guarding the estate's data. New tool. The fleet's data persistence and recovery layer — bead state, learnings, audit logs.

> **New tool — not previously in our notes.** ARMOR is the production safety layer for data: if the bead queue's SQLite database is corrupted or the server is lost, ARMOR has the encrypted backup.

---

## The Engine Room (The Full Architecture)

```
┌─────────────────────────────────────────────────────┐
│                    FLEET                            │
│                                                     │
│  Worker α  Worker β  Worker γ  Worker δ  Worker ε  │
│  (Claude   (Codex)   (Goose)   (Pi)      (Claude   │
│   Code)                                   Code)    │
│      │         │         │         │         │      │
│      └────────────────────────────────────┘        │
│                        │                            │
│                   bead-forge                        │
│              (atomic claim queue)                   │
│                        │                            │
│      ┌─────────────────┼─────────────────┐          │
│      │                 │                 │          │
│   NEEDLE            FABRIC          claude-         │
│  (dispatch)        (telemetry)      governor        │
│                                    (cost gate)      │
│                        │                            │
│                      HOOP                           │
│                 (control plane)                     │
│                        │                            │
│                      ARMOR                          │
│                    (backups)                        │
└─────────────────────────────────────────────────────┘
```

**Key property:** Workers are not uniform. Each drives a different agent harness (Claude Code, Codex, Goose, Pi) through the same adapter contract: **prompt in, exit code out**. The same bead queue feeds heterogeneous agent backends — you can mix-and-match the best tool for each bead type without changing the orchestration layer.

---

## What This Map Reveals

**The constellation is a complete production system, not a demo.** The presence of ARMOR (backup proxy), FABRIC (dedicated telemetry), and HOOP (control plane) signals this is infrastructure designed for real fleet operation — not just "run some workers in tmux."

**The 20-concept gap closes further.** The outer loop accountability note identified observability (concepts 19–20) as the main gap in the NEEDLE+kiro-config stack. FABRIC is the dedicated solution for concept 19 (Tracing). HOOP likely addresses parts of concept 20 (Metrics) — fleet control plane implies fleet health visibility.

**Heterogeneous workers are intentional.** The adapter contract (prompt in, exit code out) deliberately abstracts over which agent runs the bead. This means you can route different bead types to different agents — use Claude Code for code beads, Goose for research beads, Codex for structured tasks — all from the same queue, same state machine, same telemetry.

---

## Questions & Gaps
- HOOP is described only as "fleet control plane" — what specific operations does it expose? Pause/resume workers, inspect bead state, drain the queue before maintenance?
- ARMOR is "encrypted S3 backup proxy" — what gets backed up? Just the bead queue SQLite, or also FABRIC telemetry, learnings.md, and CLAUDE.md?
- FABRIC's telemetry format — does it speak OTel, or is it a custom format? How does it integrate with ccdash (the fleet health TUI)?
- The live demo on the constellation page shows "beads shown are simulated" — is there a way to connect FABRIC to a real live display, or is that what ccdash is for?
- The heterogeneous worker model (Claude Code + Codex + Goose + Pi through the same adapter) — is the adapter contract published, or inferred from how the existing agent configs work?

## Related Notes
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — deep dive on NEEDLE itself: 9 strands, v0.2.7, the six-step loop, bead taxonomy.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — the agent brief for setting up the full stack, including claude-governor installation and the kiro-config/NEEDLE integration.
- [bead-forge Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/bead-forge/bead-forge-repo-overview.md) — detailed coverage of the atomic claim mechanism and why bead-forge replaces the standard br CLI.
- [NEEDLE Production Safety Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Production-Safety-Guide.md) — the five-layer safety model. ARMOR (backups) and claude-governor (cost gate) are the infrastructure implementations of layers 1 and 5 of that guide.
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md) — the constellation implements concepts 2 (execution loop), 3 (agent state), 12 (subagents), 13 (agent loops), 14 (sandboxing), 15 (permissions), 19 (tracing via FABRIC), and partially 20 (metrics via HOOP).
