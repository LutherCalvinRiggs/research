# NEEDLE — Build-in-Public Learnings from `docs/notes/`

**Source:** https://github.com/jedarden/NEEDLE/tree/main/docs/notes
**Saved:** 2026-06-03
**Tags:** ai, tools, orchestration, agentic-ai, infrastructure, postmortem

---

## TL;DR
NEEDLE-deprecated was a ~45,000-line Bash predecessor to the current Rust rewrite. The `docs/notes/` directory contains 10 post-mortems and lesson files extracted from that system's production failures. These are among the most valuable documents in the repo — they document the hard-won constraints the Rust rewrite was designed around.

## 1. The Bash-at-Scale Collapse

The original NEEDLE was written in Bash. It worked until it didn't, and the scale at which it stopped working was lower than expected.

**The six fatal Bash problems at scale:**

- **No module system**: 50+ files with `source` dependencies in undefined order. Circular deps, initialization bugs, no static analysis. Adding a new module required tracing the full transitive dependency graph by hand.
- **stdout/stderr confusion**: Bash returns values via stdout. Debug logging also writes to stdout by default. Any function captured in a subshell (`$(func)`) has debug output corrupt its return value. This caused multiple worker starvation incidents — `_needle_debug` was writing to stdout inside `_needle_get_claimable_beads`, corrupting the JSON output, causing `jq` to return zero candidates. Fix is just `>&2`, but **there is no compiler that catches this bug class**. Every function that logs and is ever called in a subshell is vulnerable forever.
- **Unbound variables**: `set -u` is required for safety but produces crash-inducing typos. A typo in `NEEDLE_GLOBAL_CONFIG_LOADED_AT` vs `NEEDLE_CONFIG_LOADED_AT` caused the worker loop to exit silently after ~15 iterations.
- **Silent external command failures**: No exceptions, no error propagation. `br update --blocked-by` was not a valid `br` flag — the call silently failed with non-zero exit code, the code didn't check, and the result was an infinite mitosis-parent re-claim loop.
- **Fragile JSON parsing**: All JSON via `jq`. Every `br` CLI output format change broke multiple jq expressions across multiple files. No schema validation.
- **Primitive concurrency**: flock + /dev/shm + PID files + `trap`. Background heartbeat processes weren't always cleaned up. Race windows between read and write couldn't be made atomic.

**The rewrite lesson**: Rust gave them exhaustive match (no `_` wildcards on outcome enums as a compiler error), proper types, `Result<T>` propagation, and actual atomicity via SQLite transactions. The architecture didn't change — the language eliminated entire bug classes.

## 2. Claim Race Conditions (Five Distinct Races)

Running 20 workers against the same SQLite queue produced five distinct race conditions before the system was stable.

**Race 1 — Thundering herd**: All workers query `br ready` simultaneously, get the same top-priority bead, all attempt `br update --claim`. Only one succeeds; the rest waste a full iteration. Fix: per-workspace `/dev/shm` flock serializing all bead mutation operations.

**Race 2 — TOCTOU on closed beads**: Worker A claims bead X. Worker B's earlier query still shows X as claimable. B's claim succeeds because the close operation wasn't atomic — A cleared `claimed_by` before setting `status = closed`, creating a window. Fix: same workspace flock.

**Race 3 — Mitosis split storm**: Two workers simultaneously evaluate the same bead for task decomposition. Both pass the skip-label checks (no labels exist yet), both invoke the LLM (seconds-long operation), both create a full set of child beads — doubling the work. Fix: write `mitosis-pending` label *before* the LLM call, narrowing the race window from the full LLM duration to milliseconds.

**Race 4 — Mitosis-parent re-claim loop**: After splitting, `br update --blocked-by` was supposed to block the parent on its children. `--blocked-by` is not a valid `br` flag — the call silently failed, parent returned to `open` status, workers re-claimed and re-split it every 3 seconds. Fix: pre-claim filtering — query for `mitosis-parent` labels before the claim attempt and exclude those beads.

**Race 5 — Parent labeled after children**: `mitosis-parent` label was applied after all children were created. A concurrent worker could claim and re-split the parent in the window between first child creation and the late label write. Fix: label the parent before creating any children; roll back the label if no children are created.

**The meta-lesson**: Labels are not locks. Using bead labels (`mitosis-pending`, `mitosis-parent`) as concurrency control is fragile. Every label-based guard has an inherent race window. Use actual locks (filesystem flock, SQLite advisory locks) for mutual exclusion.

## 3. The Mitosis Explosion (5,741 Beads)

The most catastrophic production incident. A chain of four silent failures produced 5,741 duplicate beads across five workspaces.

**The chain:**
1. `br show --json` never included labels in its output (upstream bug in beads_rust). This was invisible because the code assumed labels would be present.
2. Every label-based guard returned "no labels" → depth guard saw depth=0 always → `mitosis-parent` check never fired → `mitosis-pending` lock never worked.
3. No workspace context on `br update --blocked-by` → parents never blocked by their children → parents returned to ready queue → re-claimed and re-split.
4. No session-level loop guard → same worker could split the same bead multiple times per session.
5. No hard limit on total children → exponential growth: 5^6 = 15,625 potential beads from one parent.

**Cleanup required**: 8+ batch commits closing recursive children, 606 zombie `in_progress` beads released, multiple workspaces affected (253 duplicates in mobile-gaming, 50 in NEEDLE itself, 41 in kalshi-trading, 40 in ibkr-mcp).

**The seven defenses added (final state)**:
1. Read labels via `br label list` not `br show --json`
2. Depth guard (max_depth: 3) tracking depth via `mitosis-depth:N` labels
3. Session loop guard (`/dev/shm/needle-mitosis-guard-$$`) preventing re-splitting within one worker session
4. Label write verification — confirm label was actually written
5. Pre-claim label gate — filter mitosis-parent/pending beads before claiming
6. `mitosis-pending` lock before the LLM call
7. Parent labeled before children are created

**The meta-lesson**: "Never trust metadata from tools you don't control. Validate that expected fields exist before using them. When a field is absent, fail loudly rather than defaulting to an unsafe value." Also: cleanup tooling must exist before the feature that can create messes.

## 4. Worker Starvation: 100% False Positive Rate

Over one week, 16 starvation alert beads were created. Every single one was a false positive — the database typically showed 20-40 ready beads at the time of each alert.

**Root causes (seven distinct bugs)**:
1. **Broken dependency filtering**: Using `dependency_count == 0` to find claimable beads excluded beads whose dependencies were all closed — they looked blocked but were actually ready.
2. **Debug output to stdout**: `_needle_debug` in a subshell corrupted JSON output → `jq` returned zero candidates.
3. **Workspace directory not honored**: `br` operates on `$PWD`. The bead selection code didn't `cd "$workspace"` first, querying the wrong database.
4. **Hardcoded workspace path in br wrapper**: The `br` wrapper had a hardcoded path to a different workspace's ready-queue JSON. Workers in the NEEDLE workspace received FABRIC beads instead of NEEDLE beads.
5. **All beads legitimately claimed**: The starvation detection couldn't distinguish "no beads exist" from "all beads are claimed" — the correct response is to wait, not alert.
6. **Stuck claims**: Beads in `in_progress` with stale `claimed_by` blocked others from seeing them. Release required direct SQL due to a CHECK constraint bug.
7. **Transient contention**: Brief windows during high-concurrency claim storms.

**The false-positive feedback loop**: Worker can't find work → creates HUMAN alert bead → human investigates → finds work exists → closes alert → worker creates another alert → repeat at multiple per hour.

**The three principles this produced**:
- Never alert without independent verification using a different code path than the one that failed.
- Separate "no work exists" from "all work is claimed" from "I cannot see the work (discovery bug)" — three different responses.
- Debug output must architecturally never reach stdout, not just per-function discipline.

## 5. Self-Modification Risks

NEEDLE workers write code. When those workers were assigned beads to implement NEEDLE itself, five categories of failure emerged:

1. **Workers breaking their own build**: An LLM modified `build.sh`, didn't validate the output, committed a broken binary. Hot-reload deployed it to the entire fleet simultaneously. All workers went down.

2. **Hot-reload amplifying bad changes**: Hot-reload (workers re-exec on binary mtime change) combined with `needle upgrade` meant a broken commit could propagate to the entire fleet in seconds. A single bad change's blast radius = all running workers.

3. **Workers creating infinite work**: The weave strand (gap analysis) creates new beads when it finds gaps. Workers running weave on the NEEDLE workspace created NEEDLE beads. Implementing those beads introduced new gaps. Weave found the gaps and created more beads. No limit on beads per weave run.

4. **Mitosis explosion on own beads**: Workers split their own task-splitting feature (meta-work), creating an explosion of task-splitting tasks.

5. **Workers modifying their own prompt**: Workers improved NEEDLE's prompt templates. A poorly crafted change caused subsequent workers to produce lower-quality output, which created more beads to fix the prompt.

**The four architectural fixes for the rewrite**:
- Self-modification must be gated — never auto-deploy NEEDLE changes without human approval
- Limit blast radius — canary deployments, rollback on failure, version pinning per worker
- Gap analysis needs bounds — max beads per weave run, cooldown between runs on the same workspace
- Separate the controller from the controlled — NEEDLE binary is a stable tested artifact; workers modify application code in separate workspaces

## 6. Operational Fleet Lessons

**Worker count**: On a 20-core Hetzner EX44, the ceiling was ~20 concurrent workers. At 40+, explore's filesystem scans drove CPU load to 35+, degrading all workers. Worker count must be bounded by overhead capacity (scans, DB queries, heartbeats), not available beads.

**Staggered launches**: Launching all workers simultaneously causes thundering herd. Sleep 1-2 seconds between each worker launch.

**Bead sizing guidelines** (from bead-splitting-report.md):
- Split if >4,000 characters or >10 acceptance criteria
- Split if bead mixes concerns (CLI + execution + recovery)
- Split if sequential phases ("first X, then Y, then Z")
- Keep together if single file/module, <3,000 characters, <8 criteria
- Common pattern: split oversized beads into (1) Setup/infrastructure, (2) Core implementation, (3) Advanced features/cleanup — at roughly P0/P0/P1 priority

**~20% of closed beads had timeout escalations** — oversized beads were the primary cause. An LLM cannot complete >10 acceptance criteria within a timeout.

**Agent-owned closure**: The most important bead lifecycle lesson. NEEDLE's post-dispatch closure became a fallback, not the primary mechanism. The prompt explicitly instructs the LLM to: do the work → commit → validate against the bead spec → `br close` if valid → `br update --status blocked` with comment if not.

**Workers exit when done**: Workers completing their workspace's beads exit via `idle_timeout`. This is normal, not a crash. Requires either (a) automatic worker redistribution or (b) a supervisor that relaunches workers when work appears.

**Version management**: 30+ `chore: bump version` commits in the git log. Version management should be automated from day one with a single source of truth.

## 7. The `br label list` Lesson (Applies Universally)

Buried in the mitosis post-mortem is a principle that applies to every system that calls an external CLI: **always validate that the fields you expect are actually present before using them as guards.**

`br show --json` never included labels. This was invisible in tests (mocks returned labels correctly) and invisible in the calling code (empty array was treated as "no labels", which silently disabled all guards). The mitosis explosion was the consequence.

The explicit rule the team added: when an expected field is absent, fail loudly rather than defaulting to an unsafe value. An empty guard list means "no restrictions" is a dangerous default. An absent guard list should mean "unknown state, refuse to proceed."

## Notable Quotes
> "The features that worked well were simple, isolated modules. The features that caused problems were cross-cutting concerns that interacted with multiple subsystems."

> "Defense in depth for recursive operations. Every recursive system needs multiple independent guards: hard depth limit, session-level deduplication, global rate limiting, total descendant count limit."

> "Cleanup tooling must exist before the feature that can create messes."

## Questions & Gaps
- How mature is the Rust rewrite vs. deprecated Bash version? The docs/notes/ are lessons *from* the Bash version fed *into* the Rust rewrite — the Rust version may not have re-hit all these bugs.
- The `br show --json` label bug was upstream in beads_rust — is that fixed in the current version? This is a hard dependency.
- Hot-reload was a net negative — does the current Rust version still have hot-reload? The CHANGELOG doesn't mention removing it.
- The explore strand's filesystem scans were the CPU bottleneck at scale — what does the Rust version's explore implementation look like?

## Related Notes
- [NEEDLE — Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — architecture, quickstart, module map
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — Jed's post explaining the same architecture
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the mental model this system implements
