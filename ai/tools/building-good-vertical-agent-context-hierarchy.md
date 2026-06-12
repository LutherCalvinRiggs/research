# Building a Good Vertical Agent — The Context Hierarchy Framework

**Source:** https://x.com/BrainsAndTennis/status/2063992615721435154 (pasted content by Peter Wang, builder of Shortcut spreadsheet agent)
**Saved:** 2026-06-11
**Tags:** ai, tools, prompting, fundamentals, agentic-ai, infrastructure

> Author context: Peter Wang built the Shortcut agent — widely considered the most accurate spreadsheet agent, deployed in three of the four largest multi-strategy hedge funds. One year of production building. "Where being wrong is expensive and nobody grades on a curve."

---

## TL;DR
A good vertical agent is a faithful compression of its task distribution. With the model fixed, accuracy is entirely a function of context quality. The framework: build context as a three-tier memory hierarchy (L1 always-resident / L2 on-demand curated specs / L3 raw escape hatch), minimize context cost averaged over the task distribution, and use a single code-execution tool instead of thirty specialized ones. Accuracy is not linear — a task that scores 99% is worth 10× more than one that scores 95%.

## Key Concepts & Terms

- **Faithful compression of task distribution**: The central thesis. The agent's context is not "everything relevant" — it's the optimal allocation of context budget across the frequency distribution of actual tasks your users bring. The 80% case lives in L1, the occasional 15% in L2, the long tail in L3.
- **Context as layered cache (L1/L2/L3)**: A direct analogy to CPU memory hierarchy. L1 = always resident, instant, tiny (system prompt). L2 = on-demand, one discovery step, curated English specs. L3 = raw substrate on disk, reached via grep/search skill, only paid on rare tasks.
- **Cache miss**: When what the agent needs isn't in the current tier and it must fetch from a slower one. One tool call for an L2 miss. 3–6 grep calls for an L3 miss. The cost is paid only when needed — never on tasks that don't require it.
- **Task distribution**: The frequency curve of what users actually ask. Long-tailed: bread-and-butter tasks dominate (80%), crucial-but-occasional tasks cluster in the middle (~15%), then a long tail of rare tasks each individually uncommon but collectively significant.
- **Single tool over thirty tools**: Model accuracy degrades as you add tools. Every tool is more schema in the prompt, more surface to confuse, more ways to pick wrong. One `execute_code` tool collapses all capability decisions into one (write code) and lets the model compose via a programming language instead of rigid tool calls.
- **Consequence-reporting wrapper**: The L1 read/write operations don't just execute — they report back structured diffs that surface what changed AND what looks wrong (invalid formulas, #REF! errors, implausible values) at the top of the response. Built-in linter on the agent's own edits.
- **Formula aliasing**: Compressing 500 near-identical formulas into one alias (`F1 → =RC[-2]*RC[-1]`) via R1C1 normalization. Big token savings, zero information loss. Applied to any repeated pattern.
- **Gotcha-aware spec**: L2 curated docs that encode not just the API signature but the canonical recipe and the things you only learn by failing — `suspendLayout()/resumeLayout()` must bracket batch pivot operations, the aggregation enum doesn't exist at runtime so pass the raw integer. Knowledge that no type signature will ever give you.
- **Deferred tool / meta-tool wall**: L2 pattern for tools whose schemas don't sit in the prompt. A `get_tool_info("web_search")` call fetches the schema and marks it as "fetched"; `execute_tool` refuses unless already fetched. Session-scoped tool cache. Prompt stays small; capability is resident only when loaded.

## The Three-Tier Framework

```
L1 — ALWAYS RESIDENT (system prompt, forever)
  The 80%. Bread-and-butter operations.
  Token-efficient, fast, consequence-reporting.
  Disproportionate engineering effort here.
  Cost: paid on EVERY task, whether used or not.

        ↓ one cheap discovery call on miss

L2 — ON DEMAND (curated English specs, loaded per task)
  The crucial ~15%. Important but occasional capabilities.
  Fetched by a single console.log() or get_tool_info() call.
  Hand-written prose: canonical recipe + gotchas + constraints.
  Cost: zero until needed. One tool call on miss.

        ↓ 3-6 grep calls on miss

L3 — ESCAPE HATCH (raw substrate on disk, never in prompt)
  The long tail. Complete raw API/reference, all of it.
  Accompanied by a short skill that teaches grep recipes.
  The agent must never be truly stuck.
  Cost: bounded but real. Only paid by rare tasks.
```

**The invariant**: every capability placed at the tier that minimizes total cost averaged over the task distribution. Nothing more in L1 than the 80% case demands. Nothing less in L3 than "reachable in a bounded number of steps."

## Why One Tool, Not Thirty

```typescript
// This is the entire tool surface for a production spreadsheet agent:
async function execute() {
  const data = await sheet.getCellRange("Sheet1!A1:D200");
  // ...read, compute, write...
}
```

No `read_range` tool. No `write_range` tool. No `make_chart` tool. One `execute_code` tool. The API lives inside the code.

**Why it wins:**
- Every additional tool adds schema to the prompt, adds confusion surface, adds wrong-selection risk
- Tool overlap (two tools with adjacent responsibilities) is especially harmful
- A single execute tool collapses all capability decisions to one: "write code"
- The model composes capabilities with the full power of a programming language instead of stitching rigid tool calls

Codex and Claude Code ship ~30 tools each. Pi ships 7. The 4× disagreement on the most basic design question signals there's no agreed principle. This essay argues for the fewest tools possible — ideally one.

## What L1 Actually Looks Like (Spreadsheet Example)

The `getCellRange()` function compresses a 200-row table into a fraction of the raw tokens:

1. **Formula aliasing**: 500 formulas `=A2*B2, =A3*B3...` → normalized to R1C1 → patterns appearing 10+ times → alias `F1 = =RC[-2]*RC[-1]`. One legend line instead of 500 formulas.

2. **Free context injection**: Reading `C5:E20` automatically scans leftward for row labels and upward for the header row (by voting on which nearby row has the most text cells). The model gets `Region | Q1 | Q2` and `North America | …` without asking.

3. **Style compression**: Full cell style per cell would swamp the values. Instead: group by identical style, collapse to connected range, one line per group: `D2:D201: 200 cells → numberFormat:#,##0.00, font.color:#1A7F37`.

The write diff (`setCell`) returns:
- Changed cells: grouped by sheet/row, sampled deterministically, totals shown
- `Cells that need review`: pulled up automatically — `#REF!` errors labeled MUST FIX at the top

"The feedback loop isn't 'here's what changed,' it's 'here's what changed, and here's the part you probably got wrong' — a built-in linter on the agent's own edits."

## What L2 Actually Looks Like

A curated pivot-table spec doesn't list type signatures. It teaches the recipe:

```typescript
const pt = sheet.originalSheet.pivotTables.add("SalesPivot", "SalesData", 0, 0, ...);
pt.suspendLayout();                           // ← must bracket all changes
pt.add("Region",  "Region",  rowField);
pt.add("Quarter", "Quarter", columnField);
pt.add("Amount",  "Sum of Amount", valueField, 8);  // ← 8 = sum (enum doesn't exist at runtime)
pt.resumeLayout();
```

The two gotchas (suspend/resume, raw integer for aggregation) are "the actual shape of doing pivots correctly, written down once by someone who already paid for it."

Called from inside the code:
```typescript
console.log(general.getPivotTableInfo());  // one line, zero cost until needed
```

## What L3 Actually Looks Like

70,000 lines of raw API reference. Never pasted into prompt. Accompanied by a ~100-line skill that teaches grep:

```bash
grep -n '"charts.add"' api-reference.json -A 5     # find a method
grep -n '"pivots\.' api-reference.json | head       # list a namespace
grep -n '"ChartConfig"' api-reference.json -A 10    # resolve a type
grep -n '"isEnum": true' api-reference.json -B2 -A10  # enumerate enums
```

With the skill, "tens of thousands of lines I can't read" → "the 3–6 greps that surface exactly the signature I need." The system prompt makes the escape hatch explicit: "Raw API: use when the wrapped API doesn't cover your need… don't compromise."

## How the Budget Actually Splits

| Tier | Prompt lines | What's resident |
|------|-------------|-----------------|
| L1 | ~hundreds | Core operations, execute contract, key types, guidelines |
| L2 | ~50 | Allowlist of blessed methods + pointers to getXInfo() specs |
| L3 | ~5 | Skill name/description. 70K raw lines stay on disk. |

"The budget mirrors the frequency curve: most of the prompt is spent on the 80% case, a little on signposting the 15%, and almost nothing on the long tail — which is exactly the allocation the cache-hierarchy framing predicts."

## The Three Questions for Any Domain

1. **What goes in L1?** The operations on the steep part of the frequency curve. Make them token-efficient, fast, and consequence-reporting. Spend disproportionate effort here — the agent pays this cost on every task.

2. **What goes in L2?** The important-but-occasional capabilities. Write gotcha-aware English specs reachable in one discovery step. Encode the canonical recipe and the constraints, not just the signatures.

3. **What is your L3 escape hatch?** The raw, complete substrate plus a skill that teaches the agent to mine it. It doesn't need to be ergonomic — it needs to be reachable, complete, and findable in a bounded number of steps.

## The Drift Over Time

"What counts as L1 is not fixed; it drifts with model strength."

- Early, weak models needed tiny single-purpose tools and everything spelled out
- Today's models can absorb larger L2 specs and reason over more raw L3 detail
- Yesterday's L3 becomes tomorrow's L2; yesterday's L2 collapses into L1
- The tiers slide down as models improve; the agent's scope expands outward

But the hierarchy never disappears: "context will always be scarce relative to everything you could put in it, and noise will always cost you accuracy. There is no model so large that 'put the right thing in front of it at the right time' stops mattering."

## Questions & Gaps
- The "one tool" thesis is argued from accuracy experiments but no numbers are given. What was the measured accuracy delta between 1 tool and 30 tools on the same task distribution?
- The L2 specs are "hand-written prose." At what vault size does maintaining these specs become a bottleneck? Is there a spec-generation workflow?
- The L3 grep skill assumes the raw reference is structured JSON. How does the approach change for unstructured documentation (e.g., natural language API docs)?
- "Accuracy is not linear — 99% is worth 10× more than 95%." This is asserted without a model. In what domains is this specifically true (high-stakes financial tasks) vs. less true (draft generation)?
- The consequence-reporting diff pattern (MUST FIX labels, categories) — how is this implemented? Is it rule-based classification or model-assisted?

## Related Notes
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — Anthropic's harness post describes the macro-level multi-agent architecture; this note goes one level deeper into the micro-level context architecture within a single agent. The two are complementary layers of the same stack.
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — Jed's "everything not encoded in the task definition is gone" is the same principle as L1: the agent starts cold, and what's in context is the entire world it sees. This essay is the engineering framework for what to put there.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — NEEDLE's CLAUDE.md is the L1 layer for workers. The steering files (tech.md, code-standards.md) are L2 curated specs. The escape hatch for workers is the project codebase itself. The three-tier framework maps directly.
- [kiro-config Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/kiro-config-repo-overview.md) — kiro's steering files are already structured as a three-tier hierarchy: always-on steering (L1), per-skill reference files (L2), and the full codebase (L3). This framework provides the vocabulary to describe what was built intuitively.
- [Dynamic Workflows — Anthropic Primary Source](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/dynamic-workflows-anthropic-primary-source.md) — dynamic workflows solve the multi-agent orchestration problem; this essay solves the single-agent context optimization problem. They operate at different levels. A well-architected L1/L2/L3 context makes each subagent in a workflow more accurate.
- [PatternsDev/skills Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/patterns-dev-skills-overview.md) — PatternsDev skills are L2 artifacts: curated, domain-specific, loaded on demand. The SKILL.md format is exactly the "curated English spec reachable in one discovery step" this essay describes.
