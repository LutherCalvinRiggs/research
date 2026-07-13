# The 18-Point Agentic Setup — Jamon Holmgren (Infinite Red)

**Source:** https://x.com/jamonholmgren/status/2076001786700394610
**Saved:** 2026-07-11
**Tags:** ai, tools, orchestration, agentic-ai, prompting, infrastructure

> Author context: Jamon Holmgren, Founder & CEO of Infinite Red (React Native consultancy, 20+ years, distributed team). Shared because "too many people missing giant chunks of this and it's hurting them."

---

## TL;DR
A complete 18-component production agentic coding setup covering routing, documentation, testing, cross-agent review, trace worksheets, feedback loops, tooling, and validation. The frame: without this infrastructure, you're "dry prompting and manually guiding it toward the right thing every time." With it, the agentic experience is qualitatively different. The most distinctive elements: agent trace worksheets committed with the work, cross-agent review with personas, self-healing documentation, and an end-of-shift full validation sweep.

---

## The 18 Components

### 0. AGENTS.md — The Router

The entry point for every agent session. AGENTS.md is not a monolithic instruction file — it's a **router**: it sends the agent to the right skills, docs, and tools for the task at hand.

Key design: seven-line summaries at the top of every system doc, documented in AGENTS.md, so any doc is greppable and the agent can find what it needs without reading everything.

---

### 1. Standard Workflow Doc / Skill (AGENT_WORKFLOW.md)

A customised workflow skill the agent follows. Tagged with `@/AGENT_WORKFLOW.md` in most sessions to pull it into context.

Recommended starting point: Matt Pocock's skills (see `ai/productivity/claude-code-trim-system-prompt-token-optimization.md` for context on how workflow skills interact with system prompt size).

---

### 2. Self-Healing Documentation

Every system has its own doc. **Agents are instructed to keep them updated as they work.**

Design principle: the docs stay current because the agents that change the system also update the docs describing it. Documentation debt doesn't accumulate because updating is baked into the workflow, not treated as a separate step.

Additional convention: first 7 lines of every doc are a detailed summary, making the doc easily greppable. AGENTS.md documents this convention so agents know how to search.

---

### 3. Agents Always Run the App

Agents must actually run the application and test their work as they go — especially when running autonomously or asynchronously.

The discipline: the agent doesn't declare a task done based on reading its own code. It runs the thing, observes the result, and fixes issues before moving on. This is the in-loop verification step that prevents the "looks right but doesn't work" failure mode.

---

### 4. End-to-End Tests with Agent Maintenance

Four layers:
- Instructions to write targeted tests during implementation
- Instructions to keep tests up to date (tests committed with the work, not separately)
- A doc on how to write tests and what to avoid
- A **test inventory doc** listing all tests and what each one tests

Agents write and run targeted tests during implementation, improve them, and commit everything together. Tests are not a post-implementation step.

---

### 5. Custom Linters at Pre-Commit Hooks

**The key design choice: not just flagging, actually fixing.**

Two modes:
- Problems with automatic fixes → `--fix` applied automatically
- Problems without automatic fixes → shell out to a cheaper LLM (Cursor Composer 2.5, Sonnet) to fix them

The cheaper LLM shells out specifically for the fix, not for general work. Cost-optimized: use the expensive model for implementation, the cheap model for lint remediation.

Pre-commit hooks enforce this at every commit — agents can't accidentally skip it.

---

### 6. Cross-Agent Review at Each Major Point

**The most sophisticated element in the setup.**

Review happens at four stages: research, plan, implementation, and wrap-up.

Critical rule: **not the same model reviewing the same code it wrote.**

Implementation:
- Use different agents (Codex, Claude, Cursor, etc.) for review than for implementation
- Specific docs for what review agents should look for and how to approach review
- **Personas**: each review is done from a specific lens

Persona examples:
- Maintainability reviewer
- Code quality reviewer
- Security reviewer
- Performance reviewer
- "AI smells" reviewer (patterns that look like AI-generated slop)
- Domain expert ("financial services expert", "healthcare data expert", etc.)

Each persona also **owns a set of system docs** and is responsible for keeping them updated. Domain knowledge is distributed across reviewers, not centralised.

---

### 7. Agent Trace Worksheets

A worksheet that tracks what the agent is doing each session — a running log of decisions, actions, current state, and next steps.

**The critical property:** if the agent fails partway through, another agent can pick up the worksheet and finish the job.

Committed with the work (not separate). Git tags correspond to specific worksheet names for easy retrieval.

> "You will reference these later!!" — Jamon's emphasis. The traces are institutional memory, not debugging artifacts.

This is the NEEDLE `active-work.md` pattern applied at the session level, with the addition of git tags as retrieval anchors.

---

### 8. Automatic Agent Feedback to You

At the end of every session, the agent writes feedback about the session to a doc — what went well, what was difficult, what patterns worked, what didn't.

**The compounding loop:**
- Feedback is committed with the work
- Periodically ingested into an interactive session
- Used to improve the workflows themselves

This is the Reflect strand at the session level: lessons flow from work back into the system that produces work.

---

### 9. Tools / Bin Folder

A `tools/` or `bin/` folder containing Python or bash scripts that agents have skills to create.

Example: `agent_review` bash script — lets the agent kick off cross-agent reviews via CLI without knowing each agent's particular CLI incantations. The script abstracts the agent-specific syntax.

Supporting infrastructure:
- Docs on how to make scripts effectively
- Instructions to **constantly build these out** — the tool folder grows as the agent discovers friction

The agent both uses the tools and creates new ones. Self-expanding toolset.

---

### 10. Periodic Commit Sweeps

Agents periodically sweep through recent commits from a higher level — looking for problems and gotchas that span multiple commits but aren't visible in any individual diff.

Cross-commit perspective catches: architectural drift, inconsistent patterns introduced gradually, compounding technical debt, patterns that looked fine in isolation but create problems in aggregate.

---

### 11. Coding Conventions Doc

A doc specifically for coding conventions — distinct from linter rules because not everything is lintable.

Heavy use by review agents. But the principle: anything that *can* be in a linter *should* be in a linter. The conventions doc covers the judgment-intensive cases that automation can't handle.

---

### 12. Agent Loop / Night Shift Skill

A skill specifically for autonomous/asynchronous work. Lays out the orchestration approach for unattended sessions: how to handle uncertainty, when to pause vs. continue, what constitutes a blocking issue, how to hand off to the next session.

This is the skill that makes headless operation safe — the agent knows how to behave when you're not there.

---

### 13. Task Queue Accessible to Agent

The task queue the agent works from. Can be:
- Simple: `TODOS.md` file in the repo
- Integrated: Linear, GitHub Issues, etc. via CLI fetched through API

Key property: the agent has programmatic access to the queue, not just human-readable access.

---

### 14. False-Confidence Test Audit

A periodic skill that audits the test suite for tests that are **passing but not actually testing what you think they're testing**.

These are worse than missing tests — they create false confidence that coverage exists. The skill both identifies them and fixes them.

---

### 15. Visual Regression Tests

Screenshots taken, compared via tool and visual agent review, committed with the work.

Infrastructure: git-lfs recommended for screenshot storage (binary files in git without LFS create repo size problems). At minimum, pushed into PRs for visual review.

---

### 16. Automatic Performance Benchmark Tests

Benchmarks that run automatically and notice when performance degrades.

Not manual profiling — automated signals that surface regressions before they reach production.

---

### 17. Performance Profiling Tools (Agent-Accessible)

Dedicated profiling tools agents can use for:
- Targeted benchmarking
- Trying new techniques
- Comparing outputs
- Comparing profiles between implementations

The agent doesn't just run code — it measures the code and compares alternatives. Profiling is part of the implementation loop, not a separate post-implementation step.

---

### 18. End-of-Shift Full Validation

The final step before the agent stops. Full sweep including:
- All tests
- Performance benchmarks
- Agent reviews
- Periodic sweeps
- Everything

Goal: when you return, the codebase is "as pristine as it can be." The end-of-shift validation is what makes asynchronous/overnight work trustworthy — you don't return to unknown state, you return to a known-good state plus a worksheet explaining what happened.

---

## The System Map

```
ROUTING            KNOWLEDGE           QUALITY GATES
  AGENTS.md    →   Self-healing docs   Pre-commit linters (+ auto-fix)
  AGENT_WORKFLOW   Test inventory      Cross-agent review (personas)
  Task queue       Conventions doc     False-confidence test audit
                   Bin/tools folder    Visual regression tests
                                       Performance benchmarks

EXECUTION          MEMORY              END-OF-SHIFT
  Run the app      Trace worksheets    Full validation sweep
  Write tests      Agent feedback doc  All tests + reviews
  Night shift skill  Git tags           Return to known-good state
```

---

## Mapping to the 20-Concept Framework

| Concept | Jamon's Implementation |
|---------|----------------------|
| 1. Agent (vs chatbot) | Night shift skill, agent loop |
| 2. Execution loop | Run the app (item 3), end-of-shift validation |
| 5. Config files | AGENTS.md (router), AGENT_WORKFLOW.md |
| 6. Workflow files | Skills per phase, persona docs |
| 8. Context rot | 7-line summaries, AGENTS.md routing (load only what's needed) |
| 11. Persistent memory | Self-healing docs, trace worksheets, feedback doc |
| 12. Subagents | Cross-agent review (different model = different agent) |
| 15. Permissions | Pre-commit hooks as enforcement gates |
| 16. Hooks | Pre-commit hooks with auto-fix, cheaper LLM fallback |
| 17. Prompt injection | Agent review with "AI smells" persona |
| 18. Pre-commit gates | Items 5, 15, 16: linters, visual tests, performance benchmarks |
| 19. Tracing | Trace worksheets committed with work + git tags |
| 20. Metrics | End-of-shift validation, benchmark tests, performance profiling |

Items 5–10 (config, workflow files, prompt caching, context rot) and items 14–20 (sandboxing, permissions, hooks, injection, gates, tracing, metrics) are all present. The setup checks all 20.

---

## What's Most Distinctive vs. Other Frameworks

**Agent trace worksheets with git tags** — the most novel element. Not just logging, but structured hand-off artifacts that another agent can pick up. The git tag retrieval anchor is simple and practical.

**Self-healing documentation** — agents are required to update docs as part of doing work. Documentation is not a separate task; it's a completion criterion.

**Cross-agent review with personas** — the "AI smells" persona and domain-expert personas are uncommon. Most setups stop at "review for bugs"; this adds structured perspective diversity.

**Bin/tools folder as a self-expanding agent toolset** — the agent creates new tools for itself as it discovers friction. The toolset compounds.

**Cheaper LLM for lint remediation** — explicitly routing auto-fix tasks to a cheaper model is a concrete cost-optimization pattern not commonly documented.

---

## Questions & Gaps
- The cross-agent review requires different agents (Codex, Claude, Cursor) — what's the workflow for coordinating their outputs? Is there a synthesis step, or does each persona review independently with the human reading all of them?
- The `agent_review` bash script abstracts CLI incantations — how often does this break when agent CLI interfaces update? Maintenance burden for the tooling layer?
- Self-healing docs require the agent to know which docs to update for any given change — is this governed by AGENTS.md routing, or does the agent infer it? Missing an update is silent failure.
- The false-confidence test audit (item 14) — what does this look like in practice? Does the agent read test assertions and verify they test what the name implies, or does it run mutation testing, or something else?
- Performance profiling tools (item 17) are agent-accessible — what profiling tools translate well to agent use? Most profilers produce output that requires human interpretation.

## Related Notes
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md) — the 18-point setup is a production implementation of the full 20-concept taxonomy. The mapping table above shows the correspondence.
- [kiro-config Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/kiro-config-repo-overview.md) — kiro-config implements many of the same concepts (steering files = AGENTS.md, skills = workflow docs, memory system = trace worksheets, orchestrator = cross-agent review pattern). This note fills in several gaps: self-healing docs, personas, bin folder, false-confidence audit.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — the night shift skill (item 12) and task queue (item 13) are exactly what NEEDLE implements at fleet scale. The trace worksheet (item 7) maps to the bead body and learnings.md.
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — the cross-agent review principle (item 6) is the operationalization of this. Different model, different context, different perspective — not the same agent reviewing its own work.
- [Own the Outer Loop — Agentic Accountability](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/own-the-outer-loop-agentic-accountability.md) — Osmani's outer loop (constraints, sampling, audit, ownership) maps directly onto this setup. Items 5, 6, 7, 10, 18 are the audit and sampling loops. Items 0, 1, 2, 12 are the constraints loop.
- [gstack Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/gstack-repo-overview.md) — gstack's `/cso` and `/qa` skills implement items 6 (cross-agent review) and 15 (visual QA). The bin/tools folder pattern (item 9) is what gstack's `/skillify` command produces.
