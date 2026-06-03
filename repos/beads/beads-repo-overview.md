# beads (`bd`) — Git-Backed Graph Issue Tracker for AI Agents

**Source:** https://github.com/jedarden/beads
**Saved:** 2026-06-03
**Tags:** ai, tools, infrastructure, productivity, oss, agentic-ai

> **Ecosystem root.** `bd` is the original beads CLI (Steve Yegge's project, now at v0.49.6). It's what the beads format is based on. NEEDLE calls `br`/`bf`, not `bd` directly — but understanding `bd` explains the data model and ecosystem.

---

## TL;DR
`bd` is a distributed, git-backed issue tracker designed as structured persistent memory for AI coding agents. Issues are stored as JSONL in `.beads/` inside the project repo, versioned like code. A dependency DAG lets `bd ready` return the next actionable task with no blockers. Now at v0.49.6, written in Go, cross-platform, installable via npm/Homebrew/Go.

## Key Concepts & Terms
- **JSONL-as-database**: Issues append to `.beads/issues.jsonl`. SQLite is a local cache for fast queries, rebuilt from JSONL. This makes issues portable, mergeable, and Git-diffable.
- **`bd ready`**: Returns tasks with no open blockers — the actionable queue. The core agent interface.
- **`bd update --claim`**: Atomically claims a task (sets assignee + in_progress). Note: this is the single-writer version; bead-forge's `bf claim` is the multi-worker atomic version.
- **Hash-based IDs**: `bd-a1b2c3` format. Hash-based to prevent merge collisions in multi-agent/multi-branch workflows. No sequential integer IDs.
- **Hierarchical IDs**: `bd-a3f8` (Epic) → `bd-a3f8.1` (Task) → `bd-a3f8.1.1` (Sub-task).
- **Compaction**: Semantic "memory decay" — summarizes old closed tasks to save context window tokens.
- **Daemon**: Background sync process with 30-second debounce for batching JSONL exports after CRUD operations.
- **Stealth mode**: `bd init --stealth` — use beads locally without committing to the main repo. For personal use on shared projects.
- **TOON output**: Token-optimized output format designed for LLMs — compact representation of bead state.
- **`bd doctor`**: Health checks, orphan detection (cross-references open issues against git history for issues that were committed without being closed), and repair.

## Graph Features
- **`blocks` / `blocked_by`**: Standard dependency links. DAG.
- **`relates_to`, `duplicates`, `supersedes`, `replies_to`**: Rich semantic link types for knowledge graphs.
- **`bd dep add <child> <parent>`**: Link tasks.

## Agent Commands
```bash
bd ready                          # List tasks with no open blockers
bd create "Title" -p 0            # Create P0 task
bd update <id> --claim            # Atomically claim a task
bd dep add <child> <parent>       # Link tasks in DAG
bd show <id>                      # View task details + audit trail
bd close <id>                     # Close a task
bd search "query"                 # Fuzzy search by ID, title, content
```

## Installation
```bash
# npm (cross-platform)
npm install -g @beads/bd

# Homebrew
brew install beads

# Go
go install github.com/steveyegge/beads/cmd/bd@latest

# Script
curl -fsSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash
```

**Requirements**: Linux, macOS, Windows, FreeBSD. Go 1.24+.

## Relationship to NEEDLE's `br`
NEEDLE calls `br` (which maps to either `beads_rust` by Jeffrey Emanuel or `bead-forge` by jedarden — same CLI surface). The `bd` CLI is the upstream ecosystem origin. The JSONL format and `.beads/` directory layout are compatible across all three CLIs (`bd`, `br`/beads_rust, `bf`/bead-forge).

In practice for a NEEDLE setup:
- Install `bead-forge` (`bf`) for multi-worker atomic claiming
- Symlink `br → bf` so NEEDLE's existing scripts work
- `bd` is not required unless you want the full upstream ecosystem tooling

## Notable Features (v0.49.6)
- `bd find-duplicates`: AI-powered duplicate detection
- `bd validate`: Data-integrity health checks
- `bd promote`: Promotes wisps (lightweight ephemeral notes) to persistent beads
- `bd todo`: Lightweight personal task management
- Federation: Multi-repo beads federation for distributed teams
- Dolt backend: Optional distributed database backend (experimental)
- MCP server: `beads-mcp` on PyPI — exposes beads as an MCP tool

## File Layout
```
.beads/
  issues.jsonl       ← append-only source of truth
  beads.db           ← SQLite cache (rebuilt from JSONL)
  config.yaml        ← workspace config
.beads-hooks/        ← git hooks (pre-commit, pre-push, post-merge, post-checkout)
.agent/workflows/    ← agent workflow templates
AGENTS.md            ← agent instructions
```

## Questions & Gaps
- The `bd` vs `br`/`bf` relationship: `bd` is the canonical upstream; `br`/`bf` are compatible implementations with different performance/concurrency tradeoffs. For NEEDLE, use `bf`.
- Dolt backend is experimental and was partially reverted in v0.49.5 → v0.49.6 — avoid for production NEEDLE setups.
- The `.beads-hooks/` pre-push hook includes a secret scanner — understand what it scans before using in CI.

## Related Notes
- [bead-forge Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/bead-forge/bead-forge-repo-overview.md) — the `br`-compatible atomic implementation NEEDLE uses
- [beads_viewer Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/beads_viewer/beads-viewer-repo-overview.md) — TUI dashboard for browsing beads visually
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — the orchestrator built on top of the beads data model
