# bead-forge (`bf`) — Atomic Bead Claiming for NEEDLE Fleets

**Source:** https://github.com/jedarden/bead-forge
**Saved:** 2026-06-03
**Tags:** ai, tools, orchestration, agentic-ai, infrastructure, oss

> **This is the `br` replacement NEEDLE needs.** bead-forge is a drop-in superset of `beads_rust` (`br`) with atomic claiming via SQLite `BEGIN IMMEDIATE` transactions. Symlink `br → bf` and NEEDLE works with no other changes.

---

## TL;DR
bead-forge (`bf`) is a Rust CLI that manages the `.beads/` issue database for NEEDLE multi-worker fleets. It solves the thundering-herd race condition in the original `beads_rust` (`br`) by wrapping the entire read-score-pick-update sequence in a single `BEGIN IMMEDIATE` SQLite transaction — 20 workers can call `bf claim` simultaneously and each gets a distinct bead in ~1ms. 100% compatible with the existing `.beads/` JSONL format and `br` CLI surface.

## Key Concepts & Terms
- **JSONL source of truth**: Issues are stored as an append-only `.beads/issues.jsonl` log. SQLite is a read cache on top of it, not the primary store. This is inherited from the upstream beads ecosystem.
- **`BEGIN IMMEDIATE` transaction**: SQLite's write lock is acquired at transaction start, not at first write. Serializes the entire read-score-pick-update sequence. The second worker to arrive blocks for ~0.5–2ms then picks the next bead. No central coordinator required — the database file is the coordination primitive.
- **Drop-in compatibility**: Every `br` command works identically on a bead-forge–managed workspace. The symlink `br → bf` is the entire migration. Scripts, NEEDLE workers, and CLAUDE.md instructions that reference `br` continue to work unchanged.
- **Critical-path float**: Among equal-priority beads, `bf claim` picks the one on the longest blocking chain — finishing it unblocks the most downstream work. Computed via recursive CTE on the dependency DAG.
- **Velocity affinity**: Optional routing — beads can be routed to workers whose historical p50 close time for that issue type is fastest.
- **Mitosis**: `bf mitosis` splits an oversized bead into children atomically inside a `BEGIN IMMEDIATE` transaction. No race window between reading the parent state and writing children.

## The Problem bead-forge Solves

`br` (beads_rust) claiming is a client-side read-then-write race:
```
Worker A: br list → sees bead-123 (open)
Worker B: br list → sees bead-123 (same bead)
Worker A: br update bead-123 --status in_progress → SUCCESS
Worker B: br update bead-123 --status in_progress → FAILS (already claimed)
```

With 11+ workers, 10 workers waste their full iteration on failed claims. Observed in production: 4 workers simultaneously claiming the same bead; pervasive phantom claims with 20-worker fleets.

`br`'s `busy_timeout` prevents "database is busy" crashes but doesn't prevent thundering herd. bead-forge fixes it with:

```sql
BEGIN IMMEDIATE;
  SELECT id, priority, critical_path_float FROM issues
   WHERE status = 'open'
     AND NOT EXISTS (
       SELECT 1 FROM dependencies d
        WHERE d.blocks_id = issues.id
          AND EXISTS (SELECT 1 FROM issues b
                       WHERE b.id = d.blocked_by_id
                         AND b.status != 'closed'))
   ORDER BY priority ASC, critical_path_float DESC
   LIMIT 1;
UPDATE issues SET status = 'in_progress', assignee = ?, updated_at = ? WHERE id = ?;
COMMIT;
```

## Architecture

```
.beads/
  issues.jsonl    ← append-only audit log (source of truth)
  beads.db        ← SQLite query cache + bf-only tables
  config.yaml     ← workspace config (bf fields are additive)
```

**bf-only SQLite tables** (additive — `br` ignores them, so `br` and `bf` are interchangeable on the same workspace):

| Table | Purpose |
|-------|---------|
| `bead_annotations` | Arbitrary key/value metadata per bead |
| `worker_sessions` | Active worker registry (name, model, started_at) |
| `velocity_stats` | p50/p90 close times per model + harness + issue_type |
| `critical_path_cache` | Pre-computed DAG float scores, invalidated on dep changes |
| `operation_log` | Full event history per bead |

## NEEDLE Integration

**Old (race condition with 11+ workers):**
```bash
BEAD=$(br list --format json | jq -r '.[0].id')
br update $BEAD --status in_progress --assignee $WORKER
```

**New (atomic, no race):**
```bash
bf claim --assignee $WORKER --format json
```

NEEDLE's `pluck` strand calls `bf claim` as a single atomic operation.

## Bead Genealogy (Understanding the Ecosystem)

1. **[beads](https://github.com/steveyegge/beads)** (Steve Yegge, Python/Go) — the original. Git-backed JSONL issue tracker designed as "memory for coding agents." DAG of `blocks/blocked_by` deps. No concurrency story.
2. **[beads_rust](https://github.com/dicklesworthstone/beads_rust)** (`br`, Jeffrey Emanuel, Rust) — 10–100× faster Rust port. Adds sync, doctor, TOON output. Single-writer SQLite, same no-concurrency-story as original.
3. **bead-forge** (`bf`, this repo, jedarden, Rust) — superset of `br`. Adds atomic claiming, critical-path scoring, velocity tracking, crash-safe mitosis. Built specifically for NEEDLE fleets.

## Installation
```bash
# Build from source (no release assets yet)
cargo install --git https://github.com/jedarden/bead-forge

# Symlink to replace br
ln -sf $(which bf) ~/.local/bin/br
```

Rust dependencies: `clap`, `rusqlite` (bundled SQLite), `serde_json`, `chrono`, `anyhow`.

## Key Commands for NEEDLE
```bash
bf claim --assignee alpha --format json   # Atomic claim (replaces br list + br update)
bf mitosis BEAD_ID --children '[...]'     # Crash-safe task splitting
bf list --status open                     # Same as br list
bf close BEAD_ID --body "Summary"         # Same as br close
bf label list BEAD_ID                     # Correct label read (not br show --json)
bf update BEAD_ID --status blocked        # Same as br update
```

## Questions & Gaps for My Implementation
- No release binary yet — must build from source. Check if Cargo.toml v0.1.0 is stable enough.
- The `br show --json` label bug (that caused the NEEDLE mitosis explosion) — does bead-forge fix this? The README mentions `br label list` as the correct path; worth verifying `bf show --json` includes labels.
- How does bead-forge handle the `NOT_INITIALIZED` error that NEEDLE-deprecated hit when workers ran `br` in the wrong directory? Is workspace context passed explicitly or still via `$PWD`?
- Config: bead-forge's `.beads/config.yaml` has NEEDLE-specific keys (`claim_ttl_minutes`, `rotate_*`) that `br` ignores — this is how they coexist on the same workspace.

## Related Notes
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — the orchestrator bead-forge was built for
- [NEEDLE Build-in-Public Learnings](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-build-in-public-learnings.md) — the race condition and mitosis explosion incidents that motivated bead-forge
- [beads (`bd`) Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/beads/beads-repo-overview.md) — the upstream ecosystem bead-forge is compatible with
