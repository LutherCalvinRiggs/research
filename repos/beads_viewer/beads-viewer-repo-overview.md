# beads_viewer (`bv`) — Terminal UI Dashboard for Beads

**Source:** https://github.com/jedarden/beads_viewer
**Saved:** 2026-06-03
**Tags:** ai, tools, productivity, infrastructure, oss

> **Optional tooling.** `bv` is a keyboard-driven TUI for browsing and managing beads. Not required to run NEEDLE, but useful for visual fleet monitoring, dependency graph exploration, and agent-readable triage output.

---

## TL;DR
`bv` (beads_viewer) is a Go TUI that wraps the beads issue database with Vim-style navigation, a Kanban board, dependency graph visualization, PageRank/critical-path analytics, and robot-mode JSON output for agents. Built by Dicklesworthstone (Jeffrey Emanuel), maintained separately from the `br` CLI. Latest: v0.13.0.

## Key Concepts & Terms
- **Robot mode**: `bv --robot-*` flags produce structured output (JSON/TOON) to stdout with diagnostics to stderr. **Never run bare `bv` in an agent context** — it launches the interactive TUI and blocks.
- **`bv --robot-triage`**: Single-call mega-command that returns full triage state — top picks, blockers, critical path, stats. The primary agent interface.
- **`bv --robot-next`**: Minimal output — just the top pick and the exact claim command. Lowest token cost.
- **TOON format**: Token-optimized output, more compact than JSON. Use `--format toon` or `BV_OUTPUT_FORMAT=toon`.
- **Graph WASM module**: `bv-graph-wasm/` — Rust/WebAssembly library computing graph algorithms (PageRank, betweenness centrality, critical path, k-paths, eigenvector, articulation points). Used for the insights panel.
- **Live reload**: Watches `.beads/beads.jsonl` and refreshes automatically when the file changes — no restart needed.

## Views (Interactive TUI)
| Key | View |
|-----|------|
| `j`/`k` | Navigate list (Vim-style) |
| `o`/`c`/`r` | Filter: Open / Closed / Ready (unblocked) |
| `b` | Kanban board (Open → In Progress → Blocked → Closed) |
| `g` | Dependency graph (navigate DAG visually) |
| `i` | Insights panel (PageRank, critical path, cycles, bottlenecks) |
| `h` | History view (timeline correlating git commits with bead changes) |
| `/` | Fuzzy search by ID, title, content |
| `E` | Export all issues to Markdown with Mermaid diagrams |
| `C` | Copy selected issue as formatted Markdown |
| `t`/`T` | Time-travel: compare against git revision / HEAD~5 |

## Robot Mode (Agent Interface)
```bash
# Never do this in an agent:
bv               # Launches interactive TUI — blocks

# Always use --robot-* flags:
bv --robot-triage           # Full triage state (JSON)
bv --robot-next             # Top pick + claim command only
bv --robot-triage --format toon   # Token-optimized
bv --robot-graph            # Dependency graph as JSON/DOT/Mermaid
bv --robot-help             # Full robot flag reference
```

**Output conventions**: stdout = data only, stderr = diagnostics, exit 0 = success.

## Graph Export (CLI)
```bash
bv --robot-graph                              # JSON
bv --robot-graph --graph-format=dot           # Graphviz DOT
bv --robot-graph --graph-format=mermaid       # Mermaid
bv --robot-graph --graph-root=bd-a3f8 --graph-depth=3   # Focused subgraph
```

## Installation
```bash
# Homebrew (recommended)
brew install dicklesworthstone/tap/bv

# Windows (Scoop)
scoop bucket add dicklesworthstone https://github.com/Dicklesworthstone/scoop-bucket
scoop install dicklesworthstone/bv

# Install script (Linux/macOS)
curl -fsSL "https://raw.githubusercontent.com/Dicklesworthstone/beads_viewer/main/install.sh?$(date +%s)" | bash

# Windows (PowerShell)
irm "https://raw.githubusercontent.com/Dicklesworthstone/beads_viewer/main/install.ps1" | iex
```

## Use in a NEEDLE Setup
`bv` is not required by NEEDLE but fills two useful roles:

**For operators**: Real-time visual monitoring of the bead queue — Kanban view shows which beads are in_progress (claimed by workers), blocked, or ready. Faster than running `bf list` repeatedly.

**For agents**: `bv --robot-triage` provides a richer triage output than `bf list` — includes critical path analysis, bottleneck detection, and a pre-computed pick with rationale. Some agent setups use `bv --robot-next` to let the agent see why a particular bead was selected.

## Architecture Notes
- Written in Go. The graph algorithms (PageRank, betweenness, k-paths, eigenvector centrality) are a separate Rust/WASM module in `bv-graph-wasm/`.
- Reads directly from `.beads/beads.db` (SQLite cache) and `.beads/issues.jsonl`. No daemon required.
- Live reload via filesystem watch on `issues.jsonl`.

## Questions & Gaps
- v0.13.0 — is this the version that works with bead-forge's additional SQLite tables (`bead_annotations`, `worker_sessions`, etc.)? bv reads from SQLite, so the extra tables should be invisible to it, but worth testing.
- The `--robot-triage` output format — does it expose velocity stats from bead-forge's `velocity_stats` table, or only what's in the base beads schema?

## Related Notes
- [bead-forge Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/bead-forge/bead-forge-repo-overview.md) — the `br`-compatible CLI these views read from
- [beads (`bd`) Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/beads/beads-repo-overview.md) — the upstream ecosystem and data model
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — the orchestrator whose queue `bv` visualizes
