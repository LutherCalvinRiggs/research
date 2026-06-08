# NEEDLE Implementation Guide

> A self-contained brief for an agent setting up a headless multi-worker bead fleet from scratch.
> **v2 — Updated to reflect the Kiro + NEEDLE integrated workflow.**
> Read this in full before writing any code or running any install commands.

---

## Critical Context

NEEDLE has a well-documented production history including catastrophic failures: 5,741 duplicate beads from a mitosis explosion, fleet-wide crashes from bad hot-reloads, and a week of 100% false-positive starvation alerts. Most are fixed in the current Rust rewrite. The lessons are embedded throughout this guide — do not skip the failure modes section.

**This guide assumes the following context:**
- You are a solo developer managing a multi-repo product (MAC.BID) using Kiro as your primary AI coding assistant
- You have an existing kiro-config workflow (steering files, skills, memory system, worktrees)
- You want to add NEEDLE as the execution layer for parallel, headless work
- gstack provides the product thinking and quality verification layer

The three systems form a stack. Each handles a distinct phase:

```
gstack          →  Think & Validate  (office-hours, plan reviews, security audits)
kiro-config     →  Plan & Decide     (orchestrate, memory, approval gates)
NEEDLE          →  Execute at Scale  (parallel workers, bead queue, headless)
```

---

## What NEEDLE Is

NEEDLE (**N**avigates **E**very **E**nqueued **D**eliverable, **L**ogs **E**ffort) is a Rust binary that wraps any headless coding CLI agent (Claude Code, OpenCode, Codex, Aider) in a deterministic state machine. It processes a shared SQLite-backed task queue, dispatches work to agents, and routes every possible outcome through an explicit, predefined handler.

**Core principle:** "If an outcome can happen, it has a handler. If it doesn't have a handler, it cannot happen." There are no wildcards, no swallowed errors, no undefined paths.

Multiple NEEDLE workers run in parallel against the same queue with no central orchestrator. Coordination happens entirely through SQLite atomic transactions. Adding a worker requires running one command in a new tmux session.

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

| Outcome | Exit Code | Handler |
|---------|-----------|---------|
| ✅ Success | `0` | Validate output → close bead → log effort → loop |
| ❌ Failure | `1` | Log failure reason → release bead → increment retry count → loop |
| ⏰ Timeout | `124` | Release bead → mark deferred → loop |
| 💀 Crash | `>128` | Release bead → create alert bead → loop |
| 🏁 Race lost | `4` | Exclude this candidate → retry SELECT immediately |
| 🫙 Queue empty | — | Enter strand escalation waterfall |

### Strand Escalation

When the primary workspace has no claimable beads, NEEDLE evaluates seven strands in waterfall order:

| # | Strand | Always On? | Purpose |
|---|--------|-----------|---------|
| 1 | Pluck | Yes | Process beads from the assigned workspace. Primary work. |
| 2 | Explore | Yes | Search other configured workspaces for claimable beads. |
| 3 | Mend | Yes | Cleanup: orphaned claims, stale locks, health checks. |
| 4 | Weave | Opt-in (24h cooldown) | Create new beads from documentation gaps. |
| 5 | Unravel | Opt-in (7-day cooldown) | Propose alternatives for HUMAN-blocked beads. |
| 6 | Pulse | Opt-in (48h cooldown) | Codebase health scans, auto-generate beads. |
| 7 | Knot | Yes | All strands exhausted. Alert human. Wait. |

> ⚠️ **For a first implementation, start with Pluck → Explore → Mend → Knot only.** Weave, Pulse, and Unravel have historically caused the most operational problems. Enable them only after the core loop is stable for at least a week.

---

## The Beads Ecosystem

NEEDLE calls `br` throughout its code and docs. The correct implementation for multi-worker fleets is `bead-forge` (`bf`), symlinked as `br`.

| CLI | Repo | Role |
|-----|------|------|
| `bd` | github.com/steveyegge/beads | Original. Defines the `.beads/` JSONL format. No concurrency story. |
| `br` | github.com/dicklesworthstone/beads_rust | Fast Rust port. Single-writer SQLite — race conditions at 11+ workers. |
| `bf` | github.com/jedarden/bead-forge | Drop-in superset of `br`. Atomic `BEGIN IMMEDIATE` transactions. Built for NEEDLE fleets. |

> ✅ **Use bead-forge (`bf`) as your `br`.** Symlink `br → bf` and all of NEEDLE's scripts work unchanged.

---

## How kiro-config and NEEDLE Work Together

This is the core of the v2 guide. The two systems are not alternatives — they cover different phases of the same workflow.

### The Division of Labor

**kiro-config handles judgment-intensive phases** where you need to stay in the loop:
- Product thinking (`/office-hours` via gstack)
- Strategic plan review (`/plan-ceo-review` via gstack)
- Architecture decisions (orchestrator + steering files)
- Planning phase of `/orchestrate` → produces `plan.md`
- Human approval gates on irreversible actions
- PR creation and deploy decisions

**NEEDLE handles execution-intensive phases** where parallelism matters and the spec is complete:
- Building phase — N workers execute the `plan.md` checklist in parallel
- Bulk repetitive work (add logging to 47 handlers, write tests for every untested endpoint)
- Cross-repo implementation tasks that are clearly specifiable
- Long-running autonomous work sessions

### The Handoff Point: plan.md → Beads

Your `/orchestrate` skill already produces a `plan.md` checklist. This is the natural handoff point. After you approve the plan in kiro, instead of saying "Continue" to start the interactive building phase, you create beads:

```bash
# Your /orchestrate produces plan.md with items like:
# - [ ] Add POST /auth/refresh handler in src/handlers/auth.js
# - [ ] Update authorizer allow-list in serverless.yml
# - [ ] Write unit tests for new handler

# Convert each item to a bead:
bf create "Add POST /auth/refresh handler" -p 0 --type task \
  --body "## Scope
Add handler in src/handlers/auth.js matching existing handler patterns.

## Acceptance Criteria
- Returns 200 with new JWT on valid refresh token
- Returns 401 on invalid/expired token
- Follows dbReader/dbWriter pattern from code-standards
- console.log at top with customer_id traceable identifier
- Unit tests pass

## Files
src/handlers/auth.js
tests/handlers/auth.test.js

## Close
bf close BEAD_ID --body 'summary of what was done'"
```

The key insight: **your steering files become your CLAUDE.md**. The business rules, code standards, and git workflow that kiro-config injects into every session get written into the NEEDLE workspace CLAUDE.md once — then every worker reads them cold on every bead.

### Mapping kiro-config Concepts to NEEDLE

| kiro-config | NEEDLE equivalent |
|-------------|------------------|
| `/orchestrate` planning phase | Writing the genesis bead + `plan.md` |
| `/orchestrate` building phase | Worker executing task beads |
| `plan.md` checklist item | Task bead body |
| `/handoff` | `bf close BEAD_ID --body "summary"` |
| `/update-memory` → `active-work.md` | Bead status in the queue |
| `decisions.md` (append-only) | `.beads/learnings.md` |
| Worktree per feature | NEEDLE worker worktree isolation |
| `/run-tests` | Acceptance criteria in the bead body |
| Orchestrator approval gate | HUMAN bead that blocks next phase |
| `/create-pr` | Land-and-deploy bead (still requires your confirmation) |
| `feature-work-log.md` | Bead history + NEEDLE telemetry |

### The Integrated Workflow (Full Picture)

```
THINK (gstack)
  /office-hours         → validates the problem is worth solving
  /plan-ceo-review      → strategic challenge before any code
  /plan-eng-review      → architecture decisions, edge cases, failure modes

PLAN (kiro-config)
  /orchestrate <feature>
    → reads steering files
    → analyzes existing code patterns
    → produces plan.md checklist
    → presents for approval
  [YOU: approve or revise]

DECOMPOSE (bridge step)
  For each item in plan.md:
    bf create "task title" -p 0 --body "..."
  bf dep add <child> <parent>   # enforce ordering where needed
  HUMAN gate beads for irreversible actions (migrations, deploys)

EXECUTE (NEEDLE)
  needle run --agent claude-interactive --identity alpha
  needle run --agent claude-interactive --identity bravo
  needle run --agent claude-interactive --identity charlie
  # Workers run in parallel, claim beads atomically, close when done

VERIFY (gstack + kiro-config)
  gstack /cso           → security audit before PR
  gstack /qa            → browser-based QA against staging
  /create-pr            → structured PR, requires your confirmation

SHIP
  [YOU: merge PR]
  /deploy               → guided deployment with explicit confirmation gates

REFLECT
  /archive <feature>    → consolidate learnings, update memory
  .beads/learnings.md   → NEEDLE captures worker learnings
```

### When to Use kiro-config vs NEEDLE

Use **kiro-config (interactive)** when:
- The task requires your judgment at each step
- The spec isn't complete enough to execute unattended
- You're doing the planning and architecture phase
- The work involves decisions you haven't made yet
- You want to steer mid-task

Use **NEEDLE (headless)** when:
- You can answer "yes" to: "Could a stateless worker complete this from the bead body alone?"
- The task is parallel — multiple independent items that don't need to wait for each other
- The work is repetitive across many files/endpoints/components
- The plan is complete and approved
- You want to step away while it runs

### Converting Your Worktrees

NEEDLE uses git worktrees natively — your existing `<repo>--<feature>` convention maps directly. When creating beads for a feature, specify the worktree path in CLAUDE.md or in each bead body:

```markdown
## Context
Working directory: ~/Mac_Code/API--lato
Branch: luther/lato
All commands must run from this worktree, not ~/Mac_Code/API
```

This mirrors exactly what `/pickup-memory` does — loads the worktree path so the agent works in the right place.

### HUMAN Gate Beads for Your Workflow

Your code-standards.md already has the right instincts ("confirm destructive actions"). Encode them as HUMAN beads in the bead hierarchy:

```bash
# Example: migration → human review → production deploy
bf create "Run migration on staging" -p 0 --type task --body "..."
MIGRATION_BEAD=$( bf list --status open --format json | jq -r '.[0].id' )

bf create "HUMAN: Review staging migration results before production" -p 0 --type human
HUMAN_BEAD=$( bf list --status open --format json | jq -r '.[0].id' )

bf create "Run migration on production" -p 0 --type task --body "..."
PROD_BEAD=$( bf list --status open --format json | jq -r '.[0].id' )

# Enforce ordering
bf dep add $HUMAN_BEAD $MIGRATION_BEAD   # human gate blocked by migration completing
bf dep add $PROD_BEAD $HUMAN_BEAD        # prod deploy blocked by human gate
```

Workers will complete the migration bead, then stop at the HUMAN bead (it's not a worker task), and you close it manually after reviewing. Production deploy then becomes claimable.

---

## Full Dependency List

### Hard Dependencies

#### 1. Rust 1.75+
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
rustc --version    # must be 1.75+
```

#### 2. bead-forge (the `br` CLI)
```bash
cargo install --git https://github.com/jedarden/bead-forge
ln -sf $(which bf) ~/.local/bin/br
bf --version && br --version    # both should work
```

> ⚠️ Always read labels with `bf label list BEAD_ID`, never `bf show --json`. The `show --json` bug caused the 5,741-bead mitosis explosion.

#### 3. NEEDLE Binary
```bash
curl -fsSL https://github.com/jedarden/NEEDLE/releases/latest/download/install.sh | bash
# Or build from source:
cargo install --git https://github.com/jedarden/NEEDLE
needle --version
```

#### 4. tmux
```bash
sudo apt install tmux    # Debian/Ubuntu
brew install tmux        # macOS
```

#### 5. An Agent CLI
For your setup, `claude-interactive` is recommended — keeps workers on subscription billing rather than API credits.

| Agent | CLI | Notes |
|-------|-----|-------|
| Claude Code (subscription) | claude-interactive plugin | Recommended. Requires Python 3.10+, `pip install pyte`. |
| Claude Code (API) | `claude --print` | Simpler setup, costs per-token. |
| OpenCode | `opencode` | |
| Kiro | via adapter | See Kiro integration section below. |

#### 6. claude-interactive Plugin
```bash
python3 --version    # must be 3.10+
pip install pyte
gh release download v0.2.6 --repo jedarden/NEEDLE --pattern 'claude-interactive*'
./claude-interactive-install.sh
```

### Soft Dependencies (Production)

#### 7. claude-governor (Spend Cap Proxy) — Strongly Recommended
```bash
cargo install --git https://github.com/jedarden/claude-governor
# Set ANTHROPIC_BASE_URL=http://localhost:8080 in worker environment
# Governor holds the real API key; workers never see it
```

> ⚠️ Without a spend cap, a retry loop bug can produce a five-figure invoice overnight.

#### 8. ccdash — Fleet Health TUI
```bash
cargo install --git https://github.com/jedarden/ccdash
```

#### 9. beads_viewer / bv
```bash
brew install dicklesworthstone/tap/bv
bv --robot-triage    # always use --robot flags in agent context
# Never run bare "bv" — it launches an interactive TUI and blocks
```

---

## Workspace Setup

### Step 1: Initialize the Bead Queue

```bash
cd /path/to/your/project    # e.g. ~/Mac_Code/API--feature
bf init

# Creates:
#   .beads/issues.jsonl      ← append-only source of truth (commit this)
#   .beads/beads.db          ← SQLite cache (add to .gitignore)
#   .beads/config.yaml

echo ".beads/beads.db" >> .gitignore
```

### Step 2: Configure `.beads/config.yaml`

```yaml
issue_prefixes:
  - mb        # mac.bid — use a prefix meaningful to your project
default_priority: 2
default_type: task
scoring:
  priority_weight: 0.4
  blockers_weight: 0.3
  age_weight: 0.2
  labels_weight: 0.1
  max_age_hours: 20
claim_ttl_minutes: 30
secret_protection:
  enabled: true
```

### Step 3: Create `.needle.yaml`

```yaml
agent: claude-interactive

worker:
  idle_timeout_seconds: 300
  max_retries: 3
  max_execution_minutes: 30
  max_output_tokens: 100000
  max_model_calls: 50

strands:
  pluck:   { enabled: true }
  explore: { enabled: true, workspaces: [] }
  mend:    { enabled: true }
  weave:   { enabled: false }    # enable only after core is stable
  unravel: { enabled: false }
  pulse:   { enabled: false }

telemetry:
  otlp_sink:
    enabled: false
    endpoint: "http://localhost:4317"
    protocol: "grpc"
```

### Step 4: Create CLAUDE.md from Your Steering Files

This is the critical step for your setup. Your kiro-config steering files are already excellent — transplant them into CLAUDE.md.

```markdown
# CLAUDE.md — MAC.BID API

## Project Context
MAC.BID — online auction platform. Serverless Node.js backend on AWS Lambda.
See full product context: ~/kiro-config/steering/product.md

## Bead Workflow
When work is complete:
  bf close BEAD_ID --body "Summary of what was done"

If work cannot be completed:
  bf update BEAD_ID --status blocked --body "Reason"

## Working Directory
Always check your bead body for the Worktree path.
Never work in ~/Mac_Code/API — always use the feature worktree (e.g. ~/Mac_Code/API--feature).

## Code Standards
[Contents of steering/code-standards.md]

## Tech Stack
[Contents of steering/tech.md]

## Git Workflow
[Contents of steering/git-workflow.md — branch naming, commit discipline, no force pushes]

## Testing
Mock at manager module level, not SDK level.
Test business logic, not infrastructure plumbing.
Write tests immediately after implementing, not as afterthought.

## Database Access
Always use databaseConnections.js helpers.
dbReader() for SELECT, dbWriter() for INSERT/UPDATE/DELETE.
Always disconnect in a finally block.

## Handler Logging
Every handler MUST have console.log near top with:
1. Natural-language description of what the handler does
2. Traceable identifier (customer_id, ticket_ref, invoice_id, lot_id)

## Known Constraints
[Contents of any relevant reference files from skills/api-dev/references/]
```

---

## Bead Structure for MAC.BID

### Sizing Guidelines

**Split the bead if:**
- Body length exceeds 4,000 characters
- More than 10 acceptance criteria
- Mixes concerns (e.g. endpoint + migration + tests)
- Contains sequential phases

**Keep together if:**
- Single file or function
- Under 3,000 characters, fewer than 8 criteria

**MAC.BID-specific split pattern:**
1. Schema / migration (P0)
2. Handler implementation (P0)
3. Tests + validation (P0)
4. Documentation / cleanup (P1)

### Genesis → Phase → Task for a MAC.BID Feature

```
Genesis bead:  "Implement auth token refresh flow"
  Phase bead:  "Phase 1: Database schema"
    Task bead: "Create migration 202506071400.sql + rollback"
    Task bead: "HUMAN: Review migration before running on staging"
  Phase bead:  "Phase 2: API endpoint"
    Task bead: "Add POST /auth/refresh handler"
    Task bead: "Update authorizer allow-list in serverless.yml"
  Phase bead:  "Phase 3: Tests"
    Task bead: "Write unit tests for refresh handler"
    Task bead: "Write integration test for full refresh flow"
```

### Example Well-Formed Bead for Your Stack

```
## Scope
Add POST /auth/refresh handler in src/handlers/auth.js.
Do not touch any other handlers or the authorizer.

## Context
Working directory: ~/Mac_Code/API--auth-refresh
Branch: luther/auth-refresh
Existing pattern to follow: src/handlers/auth/login.js

## Acceptance Criteria
- POST /auth/refresh returns 200 with {token, expiresAt} on valid refresh token
- Returns 401 with {error: "Invalid token"} on expired/invalid refresh token
- Returns 400 on missing body fields
- Uses dbReader() for token lookup, no manual database calls
- console.log at top includes customer_id
- All existing auth tests continue to pass
- New tests cover 200, 401, and 400 paths
- No files outside src/handlers/auth.js and tests/handlers/auth.test.js modified

## Files
src/handlers/auth.js
tests/handlers/auth.test.js

## Constraints
Use databaseConnections.js helpers only — never getSecret, getParam, or new Database
Match logging pattern from src/handlers/auth/login.js exactly
Branch from staging, PR targets staging

## Close
bf close BEAD_ID --body "Added POST /auth/refresh handler with JWT validation. Tests: 8 passing."
```

---

## Running Workers

### Starting a Single Worker

```bash
cd ~/Mac_Code/API--feature   # always use the feature worktree
needle run --agent claude-interactive --identity alpha
```

### Starting a Fleet

```bash
for identity in alpha bravo charlie delta; do
  tmux new-session -d -s "needle-$identity" \
    "cd ~/Mac_Code/API--feature && needle run --agent claude-interactive --identity $identity"
  sleep 2    # stagger launches — prevents thundering herd on first claim
done

tmux list-sessions | grep needle
tmux attach -t needle-alpha    # observe; Ctrl+B D to detach
```

### Fleet Size Limits

- 20-core server: max ~20 stable workers
- At 40+: Explore strand filesystem scans drive CPU load unsustainably
- Rule: workers ≤ CPU cores

### Worker Lifecycle

Workers exit normally when their workspace has no claimable beads and `idle_timeout_seconds` elapses. This is expected, not a crash. Relaunch when new beads are created, or configure the Explore strand to find work in other workspaces.

---

## Known Failure Modes and How to Avoid Them

### 1. Mitosis Explosion (5,741 Duplicate Beads)

**Root cause chain:** `br show --json` never included labels → depth guard saw depth=0 → locks never worked → parents re-split every 3 seconds → exponential growth.

**How to avoid:**
- Always read labels with `bf label list BEAD_ID`, never `bf show --json`
- Do not enable mitosis until core loop is stable for one week
- Build `bf batch-close --label mitosis-child` cleanup tooling before enabling mitosis

### 2. Worker Starvation False Positives (100% False Alarm Rate)

**Root causes:** broken dependency filtering, debug logging to stdout, wrong workspace directory, hardcoded paths.

**How to avoid:**
- Distinguish "no work exists" from "all work is claimed" from "I cannot see the work"
- Verify with `bf list --status open` before trusting any starvation signal

### 3. Claim Race Conditions (Five Distinct Races)

All solved by `bf claim` (atomic `BEGIN IMMEDIATE` transaction). Use `bf claim --assignee alpha --format json` — never `br list` + `br update` in sequence.

### 4. Self-Modification Risks

If you use NEEDLE to work on your own tooling/config repos:
- Never auto-deploy changes to the NEEDLE binary without human approval
- Weave strand on config repos must have a hard bead-creation bound
- Pin workers to a specific version when they're working on the orchestration layer itself

---

## Observability

### Fleet Health Commands

```bash
needle status                           # worker status and queue depth
needle status --idle-strands            # check if all strand gates are closed
bf list --status open | head -20        # inspect queue directly
bf stats                                # queue statistics
```

### Enable Telemetry (in `.needle.yaml`)

```yaml
telemetry:
  otlp_sink:
    enabled: true
    endpoint: "http://localhost:4317"
    protocol: "grpc"
```

### Cost Governance

Three execution bounds (in `.needle.yaml` under `worker:`):
- `max_execution_minutes: 30` — wall-clock timeout per bead
- `max_output_tokens: 100000` — output token cap per bead
- `max_model_calls: 50` — model invocation cap per bead

Any bound tripped → worker killed → bead released → retried.

---

## MVP Implementation Checklist

Follow this sequence. Do not skip steps or reorder them.

### Phase 1: Install Prerequisites

1. Install Rust 1.75+
2. Build bead-forge: `cargo install --git https://github.com/jedarden/bead-forge`
3. Symlink: `ln -sf $(which bf) ~/.local/bin/br`
4. Install tmux
5. Install Claude Code CLI: `which claude`
6. Install claude-interactive plugin (Python 3.10+, `pip install pyte`)
7. Build NEEDLE: `cargo install --git https://github.com/jedarden/NEEDLE`
8. Verify: `needle --version`, `bf --version`, `br --version`

### Phase 2: Initialize First Workspace

1. Pick a feature worktree you're actively working on in kiro-config
2. Run `bf init` in that worktree
3. Configure `.beads/config.yaml` with your `mb` prefix
4. Create `.needle.yaml` — agent, timeouts, disable Weave/Pulse/Unravel
5. Create `CLAUDE.md` from your steering files (tech.md + code-standards.md + git-workflow.md)
6. Add `.beads/beads.db` to `.gitignore`

### Phase 3: Convert a plan.md to Beads

1. Take a feature you've already planned with `/orchestrate`
2. Open the `plan.md` from `~/.kiro/memory/features/<feature>/plan.md`
3. For each unchecked item, run `bf create` with a well-formed bead body
4. Add `bf dep add` links where ordering matters
5. Add HUMAN beads for any migrations or deploys
6. Verify: `bf list --status open`, `bf ready`

### Phase 4: Run First Worker

1. Start single worker: `needle run --agent claude-interactive --identity alpha`
2. Watch it claim and process the first bead
3. Verify bead closes: `bf show BEAD_ID`
4. Verify learnings: `cat .beads/learnings.md`
5. Review the output — does it match your code standards?

### Phase 5: Scale to Fleet

1. Only after Phase 4 output passes your quality bar
2. Install claude-governor if using API billing
3. Launch 3–4 workers with staggered starts
4. Verify no thundering herd: `bf stats` — different workers claimed different beads
5. Run gstack `/cso` on the output before creating a PR

---

## CI/CD for Your Stack

You have GitHub Actions already. Three additions needed:

### Build Validation

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [staging]
  pull_request:
    branches: [staging]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: "1.75"
      - uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            target/
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
      - run: cargo build --release
      - run: cargo test
      - run: cargo clippy -- -D warnings
```

### Integration Smoke Test

```yaml
  integration:
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - uses: actions/checkout@v4
      - run: cargo install --git https://github.com/jedarden/bead-forge
      - run: cargo install --git https://github.com/jedarden/NEEDLE
      - run: ln -sf $(which bf) ~/.local/bin/br
      - name: Fixture workspace
        run: |
          mkdir -p /tmp/needle-fixture && cd /tmp/needle-fixture
          bf init
          bf create "Smoke test" -p 0 --type task \
            --body "Write SMOKE_TEST_PASSED to /tmp/needle-fixture/result.txt"
      - name: Run worker (echo agent)
        run: cd /tmp/needle-fixture && needle run --agent echo --identity ci --once
        timeout-minutes: 2
      - run: grep SMOKE_TEST_PASSED /tmp/needle-fixture/result.txt
```

### Protect Orchestration Files

```yaml
  protect-needle-binary:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      - name: Block changes to orchestration layer without approval
        run: |
          CHANGED=$(git diff --name-only HEAD~1 HEAD)
          for file in build.sh Cargo.toml .needle.yaml; do
            if echo "$CHANGED" | grep -q "^$file"; then
              echo "Protected file changed: $file — requires manual review"
              exit 1
            fi
          done
```

---

## Quick Reference

### Key Commands

| Command | Purpose |
|---------|---------|
| `bf init` | Initialize bead queue in current directory |
| `bf create "Title" -p 0 --type task` | Create a P0 task bead |
| `bf claim --assignee alpha --format json` | Atomically claim next bead |
| `bf list --status open` | List open beads |
| `bf ready` | List beads with no open blockers |
| `bf show BEAD_ID` | View full bead details |
| `bf close BEAD_ID --body "Summary"` | Close a bead |
| `bf update BEAD_ID --status blocked --body "Reason"` | Mark blocked |
| `bf label list BEAD_ID` | Read labels (use this, NOT `bf show --json`) |
| `bf dep add CHILD_ID PARENT_ID` | Add blocking dependency |
| `bf stats` | Queue statistics |
| `needle run --agent claude-interactive --identity alpha` | Start named worker |
| `needle status` | Worker and queue health |
| `needle status --idle-strands` | Strand gate status |
| `bv --robot-triage` | Structured triage for agents |
| `bv --robot-next` | Top pick + exact claim command |

### Repository URLs

| Project | URL | Required? |
|---------|-----|-----------|
| NEEDLE | https://github.com/jedarden/NEEDLE | Yes |
| bead-forge (`bf`) | https://github.com/jedarden/bead-forge | Yes |
| beads (`bd`) | https://github.com/steveyegge/beads | No — reference only |
| beads_viewer (`bv`) | https://github.com/jedarden/beads_viewer | Optional |
| claude-governor | https://github.com/jedarden/claude-governor | Recommended |
| ccdash | https://github.com/jedarden/ccdash | Optional |
| gstack | https://github.com/garrytan/gstack | Companion — think & verify layer |
| kiro-config | https://github.com/LutherCalvinRiggs/kiro-config | Your existing pet model |

### File Locations

| File / Directory | Purpose |
|------------------|---------|
| `.beads/` | Bead queue (workspace-local) |
| `.beads/issues.jsonl` | Append-only source of truth. **Commit this.** |
| `.beads/beads.db` | SQLite cache. **Do NOT commit.** Add to `.gitignore`. |
| `.beads/config.yaml` | Workspace config |
| `.beads/learnings.md` | Auto-managed. Captures worker learnings. |
| `.needle.yaml` | NEEDLE worker configuration |
| `CLAUDE.md` | Project instructions every worker reads — build from your steering files |
| `~/.kiro/memory/features/<name>/plan.md` | Source for bead decomposition |
| `~/.kiro/steering/` | Source for CLAUDE.md content |
| `docs/notes/` | Post-mortems and lessons (NEEDLE repo — read before writing any code) |

---

*NEEDLE v0.2.6 · MIT License · github.com/jedarden/NEEDLE*
*kiro-config · github.com/LutherCalvinRiggs/kiro-config*
*gstack · github.com/garrytan/gstack*
