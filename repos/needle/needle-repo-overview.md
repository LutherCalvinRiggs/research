# NEEDLE — Deterministic Bead Processing Orchestrator

**Source:** https://github.com/jedarden/NEEDLE
**Saved:** 2026-06-03
**Tags:** ai, tools, orchestration, agentic-ai, infrastructure, oss

> Personal note: This is a repo a friend runs. Goal is to understand it well enough to implement it.

---

## TL;DR
NEEDLE (Navigates Every Enqueued Deliverable, Logs Effort) is a Rust binary that wraps any headless coding CLI (Claude Code, OpenCode, Codex, Aider) in a deterministic state machine. It processes a shared bead queue via SQLite atomic claims, dispatches work to agents, and routes every possible outcome through an explicit handler. No wildcards. No swallowed errors. Currently v0.2.6, MIT licensed, used in production by the author.

## Key Concepts & Terms
- **Bead**: An atomic unit of work tracked in a SQLite database (`.beads/beads.db`). Has title, body, status (`open` → `in_progress` → `closed`), priority, labels, and dependency links. Managed by the `br` CLI (beads_rust, a separate project).
- **Bead queue**: The `.beads/` directory in a workspace. All workers share the same queue; SQLite transactions serialize claims.
- **Worker**: A single NEEDLE process running the six-step loop inside a tmux session. Multiple workers run concurrently against the same queue with no central orchestrator.
- **Strand**: What NEEDLE does when the primary queue is exhausted. Seven strands evaluated in waterfall order: Pluck (primary work) → Explore (other workspaces) → Mend (cleanup) → Weave (gap creation, opt-in) → Unravel (propose alternatives, opt-in) → Pulse (health scans, opt-in) → Knot (exhausted, alert human).
- **Agent adapter**: A YAML file that tells NEEDLE how to invoke a specific CLI. Adding a new agent = writing one YAML file, no code changes.
- **Genesis bead**: The root bead for a multi-phase project. References the plan document. Phase beads and task beads descend from it.
- **Mitosis**: Automatic task decomposition — a bead that's too large gets split into children by the LLM. Has been the source of the most catastrophic production incidents.
- **claude-interactive plugin**: PTY wrapper that lets NEEDLE run Claude Code in interactive mode, keeping workers on subscription billing instead of consuming API credits.
- **ccdash**: Companion TUI for monitoring worker sessions, token usage, and fleet health.
- **claude-governor**: Companion spend proxy that caps API costs and enforces weekly Anthropic quotas.

## The Six-Step Loop
Every NEEDLE worker executes this indefinitely:

1. **SELECT** — query the bead queue for the next claimable bead in deterministic priority order (same queue state = same ordering across all workers; ties broken by creation time, oldest first)
2. **CLAIM** — atomic SQLite transaction via `br update --claim`; exactly one worker wins; loser excludes that candidate and retries SELECT
3. **BUILD** — construct the prompt from the bead (title, body, workspace path, files, dependency context); deterministic — same bead = same prompt every time
4. **DISPATCH** — load the agent adapter YAML, render the invoke template, execute via `bash -c`; agent runs headless
5. **EXECUTE** — wait; only inputs back are exit code and what was written to disk; no interactive communication
6. **OUTCOME** — classify the exit code; route to the explicit handler:

| Outcome | Exit Code | Handler |
|---------|-----------|---------|
| Success | `0` | Validate → close bead → log effort → loop |
| Failure | `1` | Log reason → release bead → increment retry count → loop |
| Timeout | `124` | Release → mark deferred → loop |
| Crash | `>128` | Release → create alert bead → loop |
| Race lost | `4` | Exclude candidate → retry SELECT |
| Queue empty | — | Enter strand escalation |

Every row has a handler. There are no unhandled cases.

## Strand Escalation (Queue Empty)
Seven strands, evaluated in waterfall order. First that yields a bead wins.

1. **Pluck** — process beads from assigned workspace (always runs, no gating)
2. **Explore** — search other configured workspaces for claimable beads (always runs, no gating)
3. **Mend** — cleanup: orphaned claims, stale locks, health checks (always runs)
4. **Weave** — create beads from documentation gaps, 24h cooldown, opt-in
5. **Unravel** — propose alternatives for HUMAN-blocked beads, 7-day cooldown, opt-in
6. **Pulse** — codebase health scans, 48h cooldown, opt-in
7. **Knot** — all strands exhausted; alert human, wait

Important: Weave/Unravel/Pulse cooldowns are **per-workspace, shared across all workers**. A fleet of 20 workers can hit a state where all strand gates are simultaneously closed. Monitor with `needle status --idle-strands`.

## Repository Structure (Key Directories)
```
src/
  claim/         # Atomic bead claiming via SQLite
  dispatch/      # Agent invocation + YAML adapter loading
  outcome/       # Explicit handler per outcome type
  strand/        # pluck/explore/mend/weave/knot/pulse/reflect/unravel
  decision/      # Outcome classification from exit code + output
  mitosis/       # Worker spawn and task decomposition
  health/        # Liveness, stale-claim cleanup, watchdog
  learning/      # Per-agent performance tracking and weighting
  telemetry/     # OTLP exporter, gen_ai semantic conventions
  cost/          # Token + USD spend tracking per bead and worker
  rate_limit/    # Quota enforcement, backoff, weekly limit gates
  prompt/        # Deterministic prompt construction from bead
  registry/      # Agent adapter registry (YAML-loaded)
  sanitize/      # Output redaction and prompt-injection guards
  validation/    # Pre-dispatch and post-execution checks

docs/notes/      # 10 build-in-public post-mortems and lessons (gold mine)
.beads/          # Live bead queue, learnings, skills, traces
plugins/claude-interactive/  # PTY wrapper for subscription billing
```

## Observability
NEEDLE emits structured OTLP telemetry for every state transition. Follows OpenTelemetry `gen_ai.*` semantic conventions.

**Traces:** `worker.session`, `bead.lifecycle`, `bead.claim`, `agent.dispatch`, `strand.evaluated`, `outcome.handled`
**Metrics:** `needle.beads.completed`, `needle.beads.duration`, `needle.agent.tokens.input`, `needle.cost.usd`
**Logs:** All non-span events with severity mapping

Works with Jaeger, Tempo, Grafana, Honeycomb, Datadog via OTLP. Example docker-compose in `docs/examples/otel-collector/`.

## Getting Started
```bash
# Install prebuilt
curl -fsSL https://github.com/jedarden/NEEDLE/releases/latest/download/install.sh | bash

# Or build from source (requires Rust 1.75+)
cargo install --git https://github.com/jedarden/NEEDLE

# Run a worker
cd /path/to/workspace
needle run --agent claude --identity alpha

# Run with subscription billing (claude-interactive plugin)
needle run --agent claude-interactive --count 4
```

Minimal `.needle.yaml`:
```yaml
telemetry:
  otlp_sink:
    enabled: true
    endpoint: "http://localhost:4317"
    protocol: "grpc"
```

## Bead Scoring (Priority Queue)
From `.beads/config.yaml`:
- Priority weight: 0.4
- Blockers weight: 0.3 (beads blocking more things score higher)
- Age weight: 0.2 (older beads score higher)
- Labels weight: 0.1
- Max age hours: 20
- Claim TTL: 30 minutes (stale claims auto-released by mend strand)

## Related Projects
- **[claude-governor](https://github.com/jedarden/claude-governor)** — spend cap proxy
- **[ccdash](https://github.com/jedarden/ccdash)** — fleet health TUI
- **[CLASP](https://github.com/jedarden/CLASP)** — proxy letting Claude Code target any LLM backend
- **[agentists-quickstart](https://github.com/jedarden/agentists-quickstart)** — DevPod workspaces for Claude Code + NEEDLE
- **beads_rust** — the `br` CLI that manages the bead database (separate project, upstream dependency)

## Questions & Gaps for My Implementation
- The `br` CLI is a separate project — need to find/build beads_rust first. This is a hard dependency.
- The claude-interactive PTY plugin requires Python 3.10+ and `pyte` — where is the upstream source? Is it `pip`-installable?
- K8s CI via Argo Workflows is specific to Jed's infra — need to substitute a local CI approach.
- Worker count recommendation: bound by server capacity for overhead ops (explore's filesystem scans), not bead count. 20 workers was the ceiling on a 20-core Hetzner EX44.
- The Weave/Pulse/Unravel strands are opt-in and add significant complexity — start with just Pluck/Explore/Mend/Knot for MVP.

## Related Notes
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — NEEDLE is the code implementation of this mental model
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the architecture NEEDLE implements
- [Unit Economics of Running Cattle](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/unit-economics-of-cattle.md) — claude-governor is the cost governance component described here
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — genesis bead → phase beads → task beads is the practical hierarchy
