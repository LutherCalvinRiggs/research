# NEEDLE Implementation Guide

> A self-contained brief for an agent setting up a headless multi-worker bead fleet from scratch.
> Read this in full before writing any code or running any install commands.

---

## Critical Context

NEEDLE has a well-documented production history including catastrophic failures: 5,741 duplicate beads from a mitosis explosion, fleet-wide crashes from bad hot-reloads, and a week of 100% false-positive starvation alerts. Most are fixed in the current Rust rewrite. The lessons are embedded throughout this guide — do not skip the failure modes section.

---

## What NEEDLE Is

NEEDLE (**N**avigates **E**very **E**nqueued **D**eliverable, **L**ogs **E**ffort) is a Rust binary that wraps any headless coding CLI agent (Claude Code, OpenCode, Codex, Aider) in a deterministic state machine. It processes a shared SQLite-backed task queue, dispatches work to agents, and routes every possible outcome through an explicit, predefined handler.

**Core principle:** "If an outcome can happen, it has a handler. If it doesn't have a handler, it cannot happen." There are no wildcards, no swallowed errors, no undefined paths.

Multiple NEEDLE workers run in parallel against the same queue with no central orchestrator. Coordination happens entirely through SQLite atomic transactions. Adding a worker means running one command in a new tmux session.

---

## The Six-Step Loop

Every NEEDLE worker executes this loop indefinitely until it times out or is killed:

| Step | Name | What Happens |
|------|------|-------------|
| 1 | SELECT | Query the bead queue for the next claimable bead in deterministic priority order. Ties broken by creation time (oldest first). Same queue state = same ordering on every worker. |
| 2 | CLAIM | Atomic SQLite transaction. Exactly one worker wins. The loser excludes that candidate and retries SELECT immediately. |
| 3 | BUILD | Construct the agent prompt from the bead: title, body, workspace path, relevant files, dependency context. Deterministic — same bead always produces the same prompt. |
| 4 | DISPATCH | Load the agent adapter YAML, render the invoke template, execute via `bash -c`. Agent runs headless with no interactive communication. |
| 5 | EXECUTE | Wait. The only return values are exit code and what was written to disk. Nothing else. |
| 6 | OUTCOME | Classify the exit code. Route to the explicit handler. Loop. |

### Outcome Table

Every outcome has a handler. This is the complete set:

| Outcome | Exit Code | Handler |
|---------|-----------|---------|
| ✅ Success | `0` | Validate output → close bead → log effort → loop |
| ❌ Failure | `1` | Log failure reason → release bead → increment retry count → loop |
| ⏰ Timeout | `124` | Release bead → mark deferred → loop |
| 💀 Crash | `>128` (SIGKILL, SIGSEGV, etc.) | Release bead → create alert bead → loop |
| 🏁 Race lost | `4` | Exclude this candidate → retry SELECT immediately |
| 🫙 Queue empty | — | Enter strand escalation waterfall |

### Strand Escalation

When the primary workspace has no claimable beads, NEEDLE evaluates seven strands in waterfall order. The first that yields a bead wins:

| # | Strand | Always On? | Purpose |
|---|--------|-----------|---------|
| 1 | Pluck | Yes | Process beads from the assigned workspace. Primary work. |
| 2 | Explore | Yes | Search other configured workspaces for claimable beads. |
| 3 | Mend | Yes | Cleanup: orphaned claims, stale locks, health checks. |
| 4 | Weave | Opt-in (24h cooldown) | Create new beads from documentation gaps. |
| 5 | Unravel | Opt-in (7-day cooldown) | Propose alternatives for HUMAN-blocked beads. |
| 6 | Pulse | Opt-in (48h cooldown) | Codebase health scans, auto-generate beads. |
| 7 | Knot | Yes | All strands exhausted. Alert human. Wait. |

> ⚠️ **Fleet gotcha:** Weave, Pulse, and Unravel cooldowns are per-workspace and shared across ALL workers. A fleet of 20 workers can reach a state where all strand gates are closed simultaneously — every worker idles even though work exists in other workspaces. Monitor with: `needle status --idle-strands`

**For a first implementation, start with Pluck → Explore → Mend → Knot only.** Weave, Pulse, and Unravel are opt-in and have historically caused the most operational problems. Enable them only after the core loop is stable for at least a week.

---

## The Beads Ecosystem

NEEDLE sits on top of a three-generation lineage of compatible issue-tracking tools. Understanding the genealogy matters because NEEDLE references `br` by name throughout its code, docs, and CLAUDE.md.

| CLI | Repo | Language | Role |
|-----|------|----------|------|
| `bd` | github.com/steveyegge/beads | Go | The original. Git-backed JSONL issue tracker designed as agent memory. Defines the `.beads/` format everyone else follows. No concurrency story. |
| `br` | github.com/dicklesworthstone/beads_rust | Rust | 10–100× faster port of `bd`. Adds sync, doctor, TOON output. Same single-writer SQLite — no atomic claiming. |
| `bf` | github.com/jedarden/bead-forge | Rust | Drop-in superset of `br`. Adds atomic claiming via `BEGIN IMMEDIATE` transactions, critical-path scoring, velocity tracking, crash-safe mitosis. Built for NEEDLE fleets. |

> ✅ **Use bead-forge (`bf`) as your `br`.** Symlink `br → bf` and all of NEEDLE's scripts, prompts, and CLAUDE.md instructions continue to work unchanged. No code changes to NEEDLE required.

### Why bead-forge Instead of beads_rust

`beads_rust` claiming is a client-side read-then-write race. With 11+ workers:

```
Worker A: br list → sees bead-123 (open, highest priority)
Worker B: br list → sees bead-123 (same bead)
Worker A: br update bead-123 --status in_progress → SUCCESS
Worker B: br update bead-123 --status in_progress → FAILS (already claimed)
# 10 out of 20 workers waste their full iteration on failed claims.
# Observed in production: 4 workers simultaneously claiming the same bead.
```

bead-forge wraps the entire read-score-pick-update sequence in a single `BEGIN IMMEDIATE` SQLite transaction. The second worker to arrive blocks for ~1ms then picks the next available bead. 20 workers calling `bf claim` simultaneously each get a distinct bead with zero phantom claims.

---

## Full Dependency List

### Hard Dependencies — NEEDLE Will Not Run Without These

#### 1. Rust 1.75+

Required to build both NEEDLE and bead-forge from source.

```bash
# Install via rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# Verify
rustc --version    # must be 1.75+
cargo --version
```

#### 2. bead-forge (the `br` CLI)

No prebuilt release binary exists yet. Must build from source.

```bash
# Build and install
cargo install --git https://github.com/jedarden/bead-forge

# Symlink so NEEDLE's scripts find "br"
ln -sf $(which bf) ~/.local/bin/br

# Verify
bf --version
br --version    # should show same output via symlink
```

> ⚠️ **Known issue:** In the original `beads_rust`, `br show --json` never included labels in its output. This caused the 5,741-bead mitosis explosion — every label-based guard silently returned empty. Always read labels with `bf label list BEAD_ID`, never `bf show --json`. Verify bead-forge's `show --json` actually includes labels before relying on it.

#### 3. NEEDLE Binary

```bash
# Prebuilt install (when release assets exist)
curl -fsSL https://github.com/jedarden/NEEDLE/releases/latest/download/install.sh | bash

# Or build from source
cargo install --git https://github.com/jedarden/NEEDLE

# Verify
needle --version
```

#### 4. tmux

Each NEEDLE worker runs in its own tmux session. Required for parallel workers and session persistence.

```bash
sudo apt install tmux    # Debian/Ubuntu
brew install tmux        # macOS
```

#### 5. An Agent CLI (pick one)

NEEDLE is agent-agnostic. Any CLI that accepts a prompt and exits works via a YAML adapter file.

| Agent | CLI | Input Method | Notes |
|-------|-----|-------------|-------|
| Claude Code (subscription) | claude-interactive plugin | stdin via PTY | Recommended. Uses subscription billing, not API credits. Requires Python 3.10+ and `pip install pyte`. |
| Claude Code (API) | `claude --print` | stdin | Uses API billing. Simpler setup but costs per-token. |
| OpenCode | `opencode` | file | — |
| Codex CLI | `codex` | args | — |
| Aider | `aider --message` | args | — |

#### 6. claude-interactive Plugin (if using subscription billing)

`claude` detects non-TTY context (subprocess pipe) and switches to API billing. This plugin creates an internal PTY so Claude sees a real terminal, keeping it in interactive/subscription mode.

```bash
# Requirements
python3 --version    # must be 3.10+
pip install pyte

# Download plugin from NEEDLE release assets
gh release download v0.2.6 --repo jedarden/NEEDLE --pattern 'claude-interactive*'
chmod +x claude-interactive-install.sh
./claude-interactive-install.sh

# Verify
which claude-interactive
```

---

### Soft Dependencies — Needed for Production, Not First Run

#### 7. claude-governor (Spend Cap Proxy) — Strongly Recommended

With headless workers running unattended, there is no human watching the Anthropic dashboard. **Without a spend cap, a retry loop bug can produce a five-figure invoice overnight.** claude-governor is a Rust proxy that sits between workers and `api.anthropic.com`, enforcing hard daily/weekly spend caps.

```bash
cargo install --git https://github.com/jedarden/claude-governor

# Workers point at the proxy instead of api.anthropic.com
# The proxy holds the real API key; workers never see it
```

> ⚠️ **Do not skip this.** The pet model's hidden cost control is your attention — you are the rate limiter. Cattle removes that. Something explicit must replace it before workers go headless.

#### 8. ccdash (Fleet Health TUI — Optional)

Terminal dashboard for monitoring worker sessions, token usage, and queue health.

```bash
cargo install --git https://github.com/jedarden/ccdash
ccdash    # launch in a separate terminal
```

#### 9. beads_viewer / bv (Queue Visualizer — Optional)

Keyboard-driven TUI for browsing the bead queue visually. Kanban board, dependency graph, critical path analytics. Also has robot mode for agents.

```bash
brew install dicklesworthstone/tap/bv    # macOS/Linux

# In agent context, always use --robot flags:
bv --robot-triage    # structured JSON triage output
bv --robot-next      # top pick + exact claim command
# ⚠️ Never run bare "bv" — it launches an interactive TUI and blocks the process
```

#### 10. OTLP Backend (Observability — Optional but Recommended)

NEEDLE emits structured OpenTelemetry telemetry for every state transition, following `gen_ai.*` semantic conventions. Works with Jaeger, Tempo, Grafana, Honeycomb, Datadog, or any OTLP-compatible backend. A working docker-compose example (OTel Collector + Jaeger + Prometheus + Loki + Grafana) is at `docs/examples/otel-collector/` in the NEEDLE repo.

---

## Workspace Setup

A "workspace" is any directory with a `.beads/` directory. All workers pointing at the same workspace share the same queue.

### Step 1: Initialize the Bead Queue

```bash
cd /path/to/your/project
bf init

# This creates:
#   .beads/issues.jsonl      ← append-only source of truth (commit this)
#   .beads/beads.db          ← SQLite cache (DO NOT commit — add to .gitignore)
#   .beads/config.yaml       ← workspace config
#   .beads/metadata.json
```

### Step 2: Configure `.beads/config.yaml`

bead-forge reads NEEDLE-specific keys that the upstream `br` ignores. A minimal config for a NEEDLE fleet:

```yaml
issue_prefixes:
  - nd      # prefix for your project's beads (e.g. nd-a3f8)
default_priority: 2
default_type: task
scoring:
  priority_weight: 0.4
  blockers_weight: 0.3    # beads blocking more work score higher
  age_weight: 0.2         # older beads score higher
  labels_weight: 0.1
  max_age_hours: 20
  max_blockers: 3
claim_ttl_minutes: 30     # stale claims auto-released by mend strand
rotate:
  rotate_age_days: 30
  rotate_max_size_mb: 100
  rotate_max_archives: 10
secret_protection:
  enabled: true
```

### Step 3: Create `.needle.yaml`

```yaml
# Agent to use by default
agent: claude-interactive    # or: claude, opencode, codex, aider

# Worker settings
worker:
  idle_timeout_seconds: 300  # exit after 5min of no claimable beads
  max_retries: 3             # max retry attempts before marking a bead failed
  max_execution_minutes: 30  # kill worker if single bead takes longer
  max_output_tokens: 100000  # kill worker if bead emits more than this
  max_model_calls: 50        # kill worker if bead triggers more model calls

# Strand configuration — start with only the core three
strands:
  pluck:   { enabled: true }
  explore: { enabled: true, workspaces: [] }   # add other workspace paths here
  mend:    { enabled: true }
  weave:   { enabled: false }   # enable only after core is stable for 1+ week
  unravel: { enabled: false }
  pulse:   { enabled: false }

# Telemetry (optional)
telemetry:
  otlp_sink:
    enabled: false
    endpoint: "http://localhost:4317"
    protocol: "grpc"
```

### Step 4: Create `CLAUDE.md`

`CLAUDE.md` is the project-level instruction file every NEEDLE worker reads at the start of each task. It is the most important document you will write for the fleet. Workers start cold — they have only this file and the bead body. Anything not in `CLAUDE.md` or the bead body is unknown to the worker.

**Minimum required content:**

```markdown
# CLAUDE.md — [project name]

## Overview
[What this project does, in one paragraph.]

## Bead Workflow
When work is complete:
  bf close BEAD_ID --body "Summary of what was done"

If work cannot be completed:
  bf update BEAD_ID --status blocked --body "Reason"

## Code Style
[Language version. Linting command. Formatter command.]
[Do not add dependencies that require a newer version without updating X.]

## Testing
[How to run tests. Where tests live.]
[IMPORTANT: Do not run [expensive command] locally — push and let CI handle it.]

## Commit Convention
feat(bead-id): short description
fix(bead-id): short description

## Architecture
[Module dependency graph.]
[Key constraints and non-negotiables.]

## Known Constraints
[Fixed points that eliminate solution space.]
[External interfaces you cannot change.]
[Performance or security budgets.]
```

---

## Bead Structure and Task Decomposition

A bead is the atomic unit of work a NEEDLE worker processes in one session. Getting granularity right is critical — oversized beads are the primary cause of timeouts. ~20% of all bead closures in production required timeout escalations before sizing was tuned.

### Bead Sizing Guidelines

**Split the bead if any of:**
- Body length exceeds 4,000 characters
- More than 10 acceptance criteria
- Mixes concerns (e.g. CLI + execution + recovery)
- Contains sequential phases ("first X, then Y, then Z")

**Keep together if all of:**
- Single file or module
- Under 3,000 characters
- Fewer than 8 acceptance criteria

**Common split pattern for oversized beads:**
1. Setup / infrastructure / framework (P0)
2. Core implementation / execution (P0)
3. Advanced features / cleanup / recovery (P1 or P2)

### Genesis → Phase → Task Hierarchy

For multi-phase projects, use a three-level hierarchy:

| Level | Bead Type | Content |
|-------|-----------|---------|
| Root | Genesis bead | References the plan document. Ties all phases. Closed when all phases close. |
| Mid | Phase bead | One coherent unit of work. Own acceptance criteria. Own child tasks. Phase boundary = a condition, not a date. |
| Leaf | Task bead | Atomic work a single worker can complete in one session. One file, one function, one test suite. |

### Writing a Good Bead Body

Workers cannot ask clarifying questions. The bead body is the complete spec.

**Test:** "Could a stateless worker who has never spoken to me produce acceptable output on the first try from this bead alone?" If no, the bead needs more detail.

**Required sections:**
- **Scope:** What to do, stated precisely enough to determine in-scope vs out-of-scope
- **Acceptance criteria:** Testable, not aspirational. "p99 latency under 200ms" not "should be fast"
- **Files to touch:** Which files are in scope
- **Known constraints:** Interfaces you cannot change, dependencies you cannot add
- **Close command:** The literal `bf close` command at the bottom

**Example well-formed bead body:**

```
## Scope
Add rate limiting to the /api/auth endpoint. Limit: 10 requests per minute per IP.
Do not touch any other endpoints.

## Acceptance Criteria
- The /api/auth endpoint returns HTTP 429 when limit is exceeded
- The Retry-After header is present on 429 responses
- Existing tests continue to pass
- A test for the rate-limit path exists

## Files
src/api/auth.rs — add rate limiting middleware
tests/api/auth_test.rs — add rate limit test

## Constraints
Use the existing RateLimiter struct in src/middleware/rate_limit.rs.
Do not add new dependencies.

## Close
bf close nd-XXXX --body "Added rate limiting to /api/auth via RateLimiter middleware"
```

---

## Running Workers

### Starting a Single Worker

```bash
cd /path/to/your/project

needle run --agent claude-interactive --identity alpha

# The worker will:
#   1. Select the highest-priority claimable bead
#   2. Claim it atomically (BEGIN IMMEDIATE transaction)
#   3. Build the prompt from bead body + CLAUDE.md
#   4. Dispatch to the agent CLI
#   5. Wait for exit code
#   6. Route the outcome, loop
```

### Starting a Fleet

```bash
# Start 4 workers with staggered launches
# IMPORTANT: stagger by 1-2 seconds to avoid thundering herd on first claim
for identity in alpha bravo charlie delta; do
  tmux new-session -d -s "needle-$identity" \
    "needle run --agent claude-interactive --identity $identity"
  sleep 2
done

# Check worker status
tmux list-sessions | grep needle

# Attach to a worker session to observe
tmux attach -t needle-alpha

# Detach without killing: Ctrl+B, D
```

### Fleet Size Limits

Worker count must be bounded by the server's capacity for overhead operations — not by bead count.

- On a 20-core Hetzner EX44 (production reference): max ~20 stable workers
- At 40+ workers: the Explore strand's filesystem scans drove CPU load to 35+, degrading all workers
- General guideline: workers ≤ CPU cores

### Worker Lifecycle

Workers exit normally when their workspace has no claimable beads and `idle_timeout_seconds` elapses. This is expected behavior, not a crash. Either relaunch periodically or configure the Explore strand to find work in other workspaces automatically.

---

## Known Failure Modes and How to Avoid Them

These are extracted from `docs/notes/` in the NEEDLE repo — ten post-mortems from the Bash predecessor. The Rust rewrite was designed around these failures.

### 1. Mitosis Explosion (5,741 Duplicate Beads)

The most catastrophic production incident. A chain of four silent failures produced 5,741 duplicate beads across five workspaces.

**Root cause chain:**
- `br show --json` never included labels (upstream bug). Every label-based guard silently returned empty.
- Depth guard saw depth=0 always → `mitosis-parent` check never fired → `mitosis-pending` lock never worked.
- No workspace context on `br update --blocked-by` → parents never blocked by children → re-claimed and re-split every 3 seconds.
- No session-level loop guard, no hard limit on children → 5^6 = 15,625 potential beads from one parent.

**How to avoid:**
- Always read labels with `bf label list BEAD_ID`, never `bf show --json`.
- Do not enable mitosis until the core loop has been stable for at least one week.
- When enabling mitosis, verify the depth guard is actually receiving label data before running at scale.
- Build cleanup tooling (`bf batch-close --label mitosis-child`) before enabling mitosis. Cleanup must exist before the feature that can create messes.

### 2. Worker Starvation False Positives (100% False Alarm Rate)

Over one week, 16 starvation alert beads were created. Every single one was a false positive — the database typically showed 20–40 ready beads at alert time.

**Root causes:**
- Broken dependency filtering: `dependency_count == 0` excluded beads whose dependencies were all closed.
- Debug logging to stdout corrupted JSON output in subshells — `jq` returned zero candidates.
- Workers ran `br list` without `cd "$workspace"` first — queried the wrong database.
- One `br` wrapper had a hardcoded path to a different workspace's database.

**How to avoid:**
- Never alert without independent verification via a different code path.
- Distinguish "no work exists" from "all work is claimed" from "I cannot see the work" — three different responses.
- Before trusting any starvation signal, verify with `bf list --status open`.

### 3. Claim Race Conditions (Five Distinct Races)

This is why bead-forge exists. All five races are solved by `bf claim` (atomic `BEGIN IMMEDIATE` transaction). The key races to understand:

- **Thundering herd:** All workers attempt to claim the same top bead simultaneously. → Solved by `bf claim`.
- **TOCTOU on closed beads:** Worker B re-claims a bead Worker A already closed. → Solved by atomic transactions.
- **Mitosis split storm:** Two workers simultaneously analyze the same bead for splitting. → Solved by writing `mitosis-pending` label before the LLM call, not after.
- **Mitosis-parent re-claim loop:** Parent returns to ready queue every 3 seconds because `--blocked-by` call silently failed. → Solved by pre-claim label filtering.
- **Parent labeled after children:** Race window between child creation and parent labeling. → Solved by labeling parent before creating any children.

### 4. Self-Modification Risks

NEEDLE workers write code. When workers are assigned beads to implement NEEDLE itself, feedback loops emerge.

**The four incidents:**
- Workers modified `build.sh`, committed a broken binary, hot-reload deployed it to the entire fleet simultaneously.
- A bad commit propagated to all workers within seconds via hot-reload + `needle upgrade`.
- Weave strand with no bead-creation cap ran on the NEEDLE workspace, creating an infinite improvement loop.
- Workers split their own mitosis feature — producing thousands of task-splitting tasks.

**Rules:**
- Never auto-deploy changes to the NEEDLE binary without human approval.
- If using NEEDLE to develop NEEDLE: pin workers to a specific version during work sessions.
- Weave strand must have a hard bead-creation bound. Start with max 10 beads per weave run.
- Weave on the NEEDLE workspace itself should be disabled entirely until fully stable.

---

## Observability

A silent worker is a broken worker.

### Exported Signals

- **Traces:** `worker.session`, `bead.lifecycle`, `bead.claim`, `agent.dispatch`, `strand.evaluated`, `outcome.handled`
- **Metrics:** `needle.beads.completed`, `needle.beads.duration`, `needle.agent.tokens.input`, `needle.cost.usd`
- **Logs:** All events not represented as spans, with severity mapping

### Enable Telemetry

In `.needle.yaml`:

```yaml
telemetry:
  otlp_sink:
    enabled: true
    endpoint: "http://localhost:4317"    # gRPC
    protocol: "grpc"                     # or "http" for :4318
```

### Fleet Health Commands

```bash
needle status                           # worker status and queue depth
needle status --idle-strands            # strand gate status across all workspaces
needle status --idle-strands --format json   # for scripting

bf list --status open | head -20        # inspect queue directly
bf stats                                # queue statistics
```

### Cost Governance

**Three execution bounds every worker should have** (configure in `.needle.yaml` under `worker:`):

- **Time bound** (`max_execution_minutes`): Kill any worker running on the same bead longer than N minutes.
- **Token bound** (`max_output_tokens`): Kill any worker that has emitted more than N output tokens on a single bead.
- **Iteration bound** (`max_model_calls`): Kill any worker that has invoked the model more than N times on a single bead.

These convert "worker decides when to stop" to "orchestrator decides when to stop." A killed worker releases the bead back to the queue and retries. The cost of a wasted bounded run is one bounded run. The cost of an unwatched runaway is unbounded.

---

## MVP Implementation Checklist

Follow this sequence. Do not skip steps or reorder them.

### Phase 1: Install Prerequisites

1. Install Rust 1.75+
2. Build and install bead-forge: `cargo install --git https://github.com/jedarden/bead-forge`
3. Symlink: `ln -sf $(which bf) ~/.local/bin/br`
4. Install tmux
5. Install Claude Code CLI and verify: `which claude`
6. Install claude-interactive plugin (if using subscription billing). Verify Python 3.10+, install `pyte`.
7. Build and install NEEDLE: `cargo install --git https://github.com/jedarden/NEEDLE`
8. Verify all three: `needle --version`, `bf --version`, `br --version`

### Phase 2: Initialize Workspace

1. Run `bf init` in the project root
2. Edit `.beads/config.yaml` — set `issue_prefixes` for your project
3. Create `.needle.yaml` — set agent, idle_timeout, disable Weave/Pulse/Unravel
4. Create `CLAUDE.md` — overview, bead workflow, code style, testing, commit conventions, architecture
5. Add `.beads/beads.db` to `.gitignore`

### Phase 3: Create First Beads

1. Create a genesis bead: `bf create "Project name" -p 0 --type epic`
2. Create 3–5 task beads with well-formed bodies (scope, acceptance criteria, files, constraints, close command)
3. Verify beads are visible: `bf list --status open`
4. Verify priority ordering: `bf ready`

### Phase 4: Run First Worker

1. Start a single worker: `needle run --agent claude-interactive --identity alpha`
2. Watch it claim and process the first bead
3. Verify the bead closes correctly: `bf show BEAD_ID`
4. Verify the learnings file updates: `cat .beads/learnings.md`

### Phase 5: Scale to Fleet

1. Only after Phase 4 is working cleanly
2. Install claude-governor if using API billing
3. Launch 3–4 workers with staggered starts (sleep 2 between each)
4. Verify no thundering herd: `bf stats` — confirm different workers claimed different beads
5. Enable the Explore strand if running multiple workspaces
6. Set up OTLP observability if desired

At the end of Phase 5 you have a working cattle fleet: multiple stateless headless workers processing a shared bead queue in deterministic order, with every outcome routed through an explicit handler, and cost governance preventing runaway spend.

---

## Quick Reference

### Key Commands

| Command | Purpose |
|---------|---------|
| `bf init` | Initialize a new bead queue in the current directory |
| `bf create "Title" -p 0 --type task` | Create a P0 task bead |
| `bf claim --assignee alpha --format json` | Atomically claim the next available bead |
| `bf list --status open` | List open beads |
| `bf ready` | List beads with no open blockers (actionable queue) |
| `bf show BEAD_ID` | View full bead details |
| `bf close BEAD_ID --body "Summary"` | Close a bead with a completion summary |
| `bf update BEAD_ID --status blocked --body "Reason"` | Mark a bead as blocked |
| `bf label list BEAD_ID` | Read labels (use this, NOT `bf show --json`) |
| `bf dep add CHILD_ID PARENT_ID` | Add a blocking dependency |
| `bf stats` | Queue statistics |
| `needle run --agent claude-interactive --identity alpha` | Start a named worker |
| `needle status` | Worker and queue health |
| `needle status --idle-strands` | Strand gate status across all workspaces |
| `bv --robot-triage` | Structured triage output for agents |
| `bv --robot-next` | Top pick + exact claim command (minimum tokens) |

### Repository URLs

| Project | URL | Required? |
|---------|-----|-----------|
| NEEDLE | https://github.com/jedarden/NEEDLE | Yes — the orchestrator |
| bead-forge (`bf`) | https://github.com/jedarden/bead-forge | Yes — the `br` replacement |
| beads (`bd`) | https://github.com/steveyegge/beads | No — upstream ecosystem reference |
| beads_viewer (`bv`) | https://github.com/jedarden/beads_viewer | Optional — TUI dashboard |
| claude-governor | https://github.com/jedarden/claude-governor | Recommended for API billing |
| ccdash | https://github.com/jedarden/ccdash | Optional — fleet monitoring TUI |
| CLASP | https://github.com/jedarden/CLASP | Optional — multi-provider proxy |
| agentists-quickstart | https://github.com/jedarden/agentists-quickstart | Optional — DevPod workspaces |

### File Locations

| File / Directory | Purpose |
|------------------|---------|
| `.beads/` | Bead queue (workspace-local) |
| `.beads/issues.jsonl` | Append-only source of truth. **Commit this.** |
| `.beads/beads.db` | SQLite cache. **Do NOT commit.** Add to `.gitignore`. |
| `.beads/config.yaml` | Workspace config including NEEDLE-specific keys |
| `.beads/learnings.md` | Auto-managed by NEEDLE. Captures lessons from completed beads. |
| `.needle.yaml` | NEEDLE worker configuration (agents, strands, timeouts) |
| `CLAUDE.md` | Project instructions every worker reads on start |
| `docs/notes/` | Post-mortems and operational lessons (NEEDLE repo — read these before writing any code) |


---

## CI/CD Without Jed's Argo Workflows

NEEDLE's own CI runs on Kubernetes via Argo Workflows — Jed's personal infra. That setup is not portable. You need to substitute your own CI approach. The good news: NEEDLE does not require CI to run. It is an orchestrator that runs locally or on a server. CI is about validating the code your workers produce, not running NEEDLE itself.

### What NEEDLE's CI Actually Does

From the repo's `.github/workflows/` and Argo config, NEEDLE's CI pipeline does three things:

1. **Build validation** — `cargo build --release` to confirm the binary compiles after worker changes
2. **Test suite** — `cargo test` across all modules
3. **Integration smoke test** — spin up a single worker against a fixture workspace with pre-seeded beads, verify it claims and closes them correctly, check exit code

You need equivalents for all three.

### Option A: GitHub Actions (Recommended Starting Point)

The simplest substitute. Free for public repos, 2,000 minutes/month for private.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: "1.75"

      - name: Cache cargo
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target/
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: Build
        run: cargo build --release

      - name: Test
        run: cargo test

      - name: Lint
        run: cargo clippy -- -D warnings
```

**What this does not cover:** the integration smoke test (claiming and closing real beads) — that requires `bf` and a real `.beads/` fixture. Add it in Phase 2 once the basic pipeline is green.

### Option B: Integration Smoke Test (Phase 2 Addition)

Once the build/test pipeline is stable, add a smoke test that actually exercises the full claim→close loop. This catches regressions that unit tests miss (wrong workspace path, broken agent adapter YAML, silent `bf` invocation failures).

```yaml
  integration:
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: "1.75"

      - name: Build bead-forge
        run: cargo install --git https://github.com/jedarden/bead-forge

      - name: Build NEEDLE
        run: cargo install --git https://github.com/jedarden/NEEDLE

      - name: Symlink br → bf
        run: ln -sf $(which bf) ~/.local/bin/br

      - name: Initialize fixture workspace
        run: |
          mkdir -p /tmp/needle-fixture
          cd /tmp/needle-fixture
          bf init
          bf create "Smoke test bead" -p 0 --type task \
            --body "Write the string SMOKE_TEST_PASSED to /tmp/needle-fixture/result.txt"

      - name: Run worker (echo agent)
        run: |
          cd /tmp/needle-fixture
          needle run --agent echo --identity ci --once
        timeout-minutes: 2

      - name: Verify result
        run: grep SMOKE_TEST_PASSED /tmp/needle-fixture/result.txt
```

> **Note on the echo agent:** NEEDLE's agent adapter system lets you define a YAML adapter that calls any command. For CI, define an `echo` adapter that writes deterministic output and exits 0 — no LLM call required. This tests the claim/dispatch/outcome machinery without API costs or flakiness.

**Echo adapter YAML** (save as `.needle/adapters/echo.yaml` in your workspace):

```yaml
name: echo
description: Deterministic CI adapter — no LLM
invoke: echo "SMOKE_TEST_PASSED" > /tmp/needle-fixture/result.txt && bf close {bead_id} --body "ci"
```

### Option C: Self-Hosted Runner on Your NEEDLE Server

If you are running NEEDLE on a dedicated server (Hetzner, etc.), running CI there eliminates the round-trip to GitHub's runners and lets you test against real hardware limits.

```bash
# On your server — install the GitHub Actions runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.317.0/actions-runner-linux-x64-2.317.0.tar.gz
tar xzf actions-runner-linux-x64.tar.gz
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO --token YOUR_TOKEN
./run.sh
```

Then in your workflow, change `runs-on: ubuntu-latest` to `runs-on: self-hosted`.

**Tradeoff:** Faster and cheaper at scale, but your CI and your NEEDLE fleet compete for the same CPU and SQLite. Run CI jobs with `nice -n 19` to de-prioritize them relative to live workers.

### What to Validate in CI vs. What to Leave to Workers

| Concern | Where to handle it |
|---------|-------------------|
| Does NEEDLE compile? | CI |
| Do unit tests pass? | CI |
| Does the claim→close loop work mechanically? | CI (echo adapter smoke test) |
| Does the agent produce correct output? | Bead acceptance criteria + validation gate |
| Does the agent produce good code? | Human review of closed beads |
| Did workers introduce regressions? | Your existing test suite, run as a bead |

CI verifies the machinery. Workers verify the output. Keep them separate.

### Protecting the NEEDLE Binary from Worker Modification

One of the documented self-modification incidents: workers modified `build.sh`, the CI pipeline ran, produced a broken binary, and hot-reload deployed it to the entire fleet simultaneously.

Add a protection rule in CI:

```yaml
  protect-needle-binary:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Fail if orchestration files changed without approval
        run: |
          CHANGED=$(git diff --name-only HEAD~1 HEAD)
          for file in build.sh Cargo.toml .needle.yaml; do
            if echo "$CHANGED" | grep -q "^$file"; then
              echo "Protected orchestration file changed: $file"
              echo "Requires manual review before merge."
              exit 1
            fi
          done
```

This blocks automated merges of any PR that touches the orchestration layer — even if a worker opened the PR. A human must explicitly approve and remove the block.

---

*NEEDLE v0.2.6 · MIT License · github.com/jedarden/NEEDLE*
