# How to Kill the Bloat in Claude Code's System Prompt

**Source:** https://mattpocock.com/blog/claude-code-system-prompt (Matt Pocock)
**Saved:** 2026-06-27
**Tags:** ai, productivity, prompting, tools, economics

---

## TL;DR
Every Claude Code request ships a hidden payload — tool definitions, skill catalogues, system instructions — billed on every turn. A six-step process to measure it with `/context`, identify the biggest offenders via a logging proxy, then trim with `disable*` flags, `permissions.deny` rules, and `skillOverrides`. Matt Pocock's configuration dropped his context by tens of thousands of tokens per turn.

## Key Concepts & Terms

- **`/context`**: Claude Code command that breaks the context window down by category (system prompt, system tools, MCP tools, memory files, messages) with token counts and percentage share. The baseline measurement tool.
- **`disableBundledSkills`**: Flag that removes all of Anthropic's bundled skills at once (dataviz, review, init catalogue) from the payload. Their slash commands remain typeable but Claude never sees the skill definitions. All-or-nothing.
- **`disableWorkflows`**: Flag that removes the multi-agent Workflow tool — described as the largest single line in the tool table.
- **`permissions.deny` with bare name**: Removes an individual tool's definition from the payload entirely. Claude never sees it. Different from a scoped rule.
- **Bare name vs. scoped rule** (critical distinction): `"NotebookEdit"` (bare) removes the tool definition from the payload — Claude never sees it. `"Bash(rm *)"` or `"Skill(dataviz)"` (scoped) blocks the matching call but leaves the definition in the payload. To shrink the request, always use bare names.
- **`skillOverrides`**: Finer-grained alternative to `disableBundledSkills` — set individual skills to `"off"` (removed from payload) or `"user-invocable-only"` (typeable but Claude doesn't see the definition).
- **Logging proxy**: A Node.js proxy that intercepts Claude Code's HTTP requests to the Anthropic API, forwards them untouched, streams the response back, and logs every request as readable Markdown. Reveals the full payload including every tool schema.

## The Six-Step Process

### Step 1: Measure with `/context`
```
/context
```
Records your baseline: system prompt tokens, system tools tokens, MCP tools tokens, memory files tokens, message tokens. Note these numbers before changing anything.

**Limitation**: `/context` shows tools as a group total, not ranked individually. Step 2 gets the breakdown.

### Step 2: Find Biggest Offenders with Logging Proxy

Run a local proxy that intercepts Claude Code's API calls:
```bash
node proxy.mjs
```

In another terminal:
```bash
ANTHROPIC_BASE_URL=http://localhost:8787 claude
```

Output — ranked table of tools by size:
```
[agent-proxy] 69 tools · 154,946 tool bytes · 65,538 real input tokens
  Workflow      21229 B  ~5307 tok
  DesignSync     8978 B  ~2245 tok
  Monitor        7767 B  ~1942 tok
```

Also writes each request to `./logs/` as full Markdown showing every tool schema, system prompt, and message history exactly as the model receives it.

### Step 3: Disable Whole Features with `disable*` Flags

For features you never use — removes the feature plus all tools and instructions it carries:

| Flag | What it removes |
|------|----------------|
| `disableBundledSkills` | All bundled Anthropic skills (dataviz, review, init) |
| `disableWorkflows` | Multi-agent Workflow tool (largest single tool) |
| `disableRemoteControl` | Remote control tools |
| `disableClaudeAiConnectors` | Claude.ai connector tools |
| `disableArtifact` | Artifact tools |

### Step 4: Remove Individual Tools with `permissions.deny`

Bare tool name = removes from payload:
```json
{
  "permissions": {
    "deny": ["NotebookEdit", "CronCreate"]
  }
}
```

**Important**: `"NotebookEdit"` removes the definition. `"Bash(rm *)"` only blocks the call — definition stays in context.

### Step 5: The Configuration (Treat as Menu, Not Prescription)

Matt Pocock's configuration — what he cut after measuring:
```json
{
  "permissions": {
    "deny": [
      "EnterPlanMode",
      "ExitPlanMode",
      "DesignSync",
      "NotebookEdit",
      "SendMessage",
      "PushNotification",
      "RemoteTrigger",
      "ReportFindings",
      "ScheduleWakeup",
      "AskUserQuestion",
      "CronCreate",
      "CronDelete",
      "CronList"
    ]
  },
  "disableBundledSkills": true,
  "disableWorkflows": true,
  "disableRemoteControl": true,
  "disableClaudeAiConnectors": true,
  "disableArtifact": true
}
```

**The caveat**: Background jobs, multi-agent runs, and worktree workflows depend on some of this "bloat" — task tools, Workflow, worktree tools. If you use loops or proactive routines, don't remove those. The right configuration is derived from your own `/context` measurement and proxy log, not copied from someone else's.

### Step 6: Re-Measure with `/context`

Restart Claude Code (forces settings reload), run `/context` again. Compare token totals to baseline. The difference is what you were paying per turn and now aren't.

## What Goes in `~/.claude/settings.json` vs `.claude/settings.json`

- `~/.claude/settings.json` — applies to every project
- `.claude/settings.json` — applies to one specific project

Aggressive trimming makes sense at the global level for interactive coding sessions. Project-level settings can add tools back for projects that need specific capabilities.

## The Context Rot Connection

This note is the practical implementation of context rot (concept 8 in the 30 Core Agentic Engineering Concepts): "every token should earn its place." The tool definitions and skill schemas in the default Claude Code payload are context rot at the infrastructure level — tokens consumed on every request for capabilities you never use, diluting the signal of what you're actually asking the model to do.

## Questions & Gaps
- The logging proxy (`proxy.mjs`) is referenced but the source isn't linked in the article — where does it come from? Is this Matt Pocock's own tool or a published package?
- `disableWorkflows` removes the multi-agent Workflow tool — but if you want to use `/goal` (a related primitive), does that also get disabled? The article doesn't clarify the relationship between `/goal`, `/loop`, and the Workflow tool.
- `skillOverrides` is mentioned but not shown — what's the exact settings.json syntax?
- The proxy approach logs every request to disk. For privacy-sensitive codebases, are the logs ephemeral or persistent? What's the cleanup story?
- After trimming, does removing tools affect Claude's ability to self-discover capabilities it needs (e.g., does removing `CronCreate` prevent `/schedule` from working)?

## Related Notes
- [Stopped Hitting Claude's Usage Limits — 10 Things](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/stopped-hitting-claude-usage-limits-10-things.md) — session-level token optimization habits. This note is the infrastructure-level equivalent: reducing the baseline cost of every request before any per-session habits apply.
- [Building a Good Vertical Agent — Context Hierarchy](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/building-good-vertical-agent-context-hierarchy.md) — the L1/L2/L3 context hierarchy framework directly applies here. Default Claude Code loads all tools into L1 (always-resident). The `disable*` flags and `deny` rules push unused tools out of L1 entirely (not even L3 — they simply don't exist in the prompt).
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md) — concept 7 (Prompt Caching) and concept 8 (Context Rot) are directly addressed here. Removing unused tools reduces context rot; caching becomes cheaper when the stable prefix is smaller.
- [Claude Code Loops — Official Taxonomy](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-code-loops-official-taxonomy.md) — the caveat from this note matters: if using proactive loops or dynamic workflows, don't remove Workflow, CronCreate, CronDelete, CronList. The right trim depends on which loop primitives you actually use.
