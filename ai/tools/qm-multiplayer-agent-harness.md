# QM — Multiplayer Agent Harness for Work

**Source:** https://github.com/yc-software/qm
**Author:** yc-software (YC-backed startup)
**Saved:** 2026-07-31
**Tags:** ai, tools, infrastructure, orchestration, agentic-ai, technology

> MIT licensed. Early stage (11 stars, 19 commits). TypeScript / Node.js. Deploy target: Fly.io or AWS. Vendor-agnostic — Pi, OpenCode, Codex, and Claude Code all drive the same core.

---

## TL;DR
QM is an open-source multiplayer agent harness designed for startups. Unlike personal assistant agents, QM gives each employee their own isolated workspace (scoped memory, files, keychain, permissions, crons, sandbox) while enabling shared collaboration in Slack channels and projects. Vendor-agnostic at the harness level — swap Pi, OpenCode, Codex, or Claude Code without changing the deployment. The core distinguishing design: per-scope isolation with shared infrastructure, deployed into the operator's own cloud account.

---

## The Core Design Insight

> "Most agents are designed like personal assistants. You can make one work for a whole company, but it quickly gets complex. QM is designed for startups."

The problem with "one agent for the whole company": context, memory, files, permissions, and credentials all end up shared — creating noise, security risk, and conflicts between different employees' work.

QM's answer: **per-scope isolation** at the workspace level, with shared infrastructure underneath.

| Scope | Isolation |
|-------|-----------|
| Each person | Own memory, files, keychain view, permissions, crons, web apps, durable sandbox |
| Each room/channel | Own scoped memory and shared context |
| Org level | Admin config, security posture, harness/model policy |

---

## Architecture

```
Postgres DB
  sessions · memory · queue
       ↕
  HEADLESS CORE
    API · identity · policy · scheduler
    Agent loop (Pi / OpenCode / Claude Code / Codex)
       ↕
  Per-scope sandbox
    files · tools · logged-in services
```

**Every turn** runs through a central core that uses a variety of models and harnesses. Postgres holds user data, session history, and durable state.

**The `execute` tool** — the core has a small, fixed tool surface. One of those tools is `execute`, which runs commands in the scope's own isolated sandbox — its durable computer, where installed tools stay installed across sessions.

**Plugins over the core's HTTP API:**
- Web UI (Vite + Lit)
- Admin panel
- Public portal
- Slack (Bolt, in-process plugin)

**The deployment directory contract:** Everything company-specific (org config, custom tools and skills, sandbox image, infrastructure) lives in a deployment directory. The `qm` CLI validates and deploys it. Every substrate (harness, session store, sandbox, memory) sits behind an interface — production implementations swap in via one wiring file.

---

## Vendor Agnosticism (Key Design Choice)

QM's harness-model interface is explicitly decoupled:

> "Pick your own harness and model and switch between them — Pi, OpenCode, Codex, and Claude Code all drive the same core, so a deployment isn't tied to any single vendor."

This is the same adapter contract that NEEDLE implements (`cd {workspace} && <agent-cli> --model {model} < {prompt_file}`). QM implements the same principle at the product layer — company choice of agent backend without re-architecting the deployment.

---

## Security Posture (Three Tiers)

QM defines three security postures at the org level, which individual scopes can only tighten, not loosen:

| Posture | Behavior |
|---------|---------|
| **Strict** | Every harness tool call pauses for human approval (except two no-effect turn enders) |
| **Auto** (default) | Classifier screens provenance-labelled external data and tool results before they reach the model |
| **Dangerous** | No content screening, no pauses between tool calls |

**Predeclared command policy** applies in every posture including Dangerous: approval rules and hard denials for things like recursive deletes or destructive SQL.

**The provenance-labelling approach in Auto mode** is notable — external data is labelled by source before it reaches the model, enabling the screening classifier to apply different trust levels to different data sources. This is a practical implementation of prompt injection defense (30 core concepts, concept 17).

---

## Skills Architecture

Skills are scope-owned and shareable by grant:
- Personal skills (individual scope)
- Shared skills (granted to specific people or rooms)
- Org-wide promotion (admin-gated)
- Skill packs imported from git repositories (e.g., `qm init` pulls from the deployment repo)

Two meta-skills maintain the private fork boundary:
- **`update-qm`** — merges upstream qm into the private fork, opens sync PR
- **`upstream-pr`** — sends org-agnostic fixes back to qm, checks outgoing diff for org identifiers before pushing

---

## Features

- **Personal and shared scopes** — employees customize their own workspace; collaborate in shared channels and projects
- **Slack and web** — same identity and configuration carries between Slack and the web app
- **Admin control** — org-level config, security posture, available harnesses and models
- **Web apps** — spin up custom internal apps, publish to the right people
- **Shared skills** — scope-owned, shareable by grant, admin-gated org promotion
- **Background work** — crons and watches run work autonomously

---

## Deployment

Initialize from npm (no source checkout required):
```bash
npm exec --yes --package=@yc-software/qm@<version> -- \
  qm init . --org <slug> --target fly-or-aws
npm install
```

Deploys into the **operator's own cloud account**. QM itself has no production deployment workflow and no CI that deploys anywhere — the org controls its own infrastructure.

**Private fork pattern** (for orgs wanting full source access):
- Clone bare → push to private repo
- Add upstream remote
- Org customizations live in `deploy/layers/<org>/` only
- Core stays byte-identical to upstream (keeps merges small)
- Never use GitHub's Fork button (creates shared object network, can't be made private)

---

## What This Represents in the Landscape

QM sits in an interesting position:

**vs. OpenAI Presence / Anthropic Ode:** Managed enterprise deployment services. QM is the self-hosted, open-source, vendor-agnostic alternative. Deploy into your own account, swap harnesses freely.

**vs. NEEDLE/bead-forge:** NEEDLE is task-queue-based headless fleet execution. QM is interactive-first with background work as a feature, not the primary mode. QM's crons are NEEDLE's scheduled beads; QM's per-scope isolation is NEEDLE's worktree-per-agent pattern applied to employees rather than tasks.

**vs. Ambiance:** Both emphasize per-scope isolation and filesystem-as-persistent-memory. Ambiance is a Unix-native single-machine harness; QM is a multi-tenant cloud deployment.

**The multiplayer angle is genuinely novel** in the open-source harness space. Most harnesses (NEEDLE, kiro-config, Ambiance) are single-developer tools. QM is the first open-source harness designed for team-scale deployment with identity, permissions, and shared infrastructure built in.

---

## Questions & Gaps
- The auto-mode classifier that screens provenance-labelled data — is this a custom model, an LLM-as-judge, or a rule-based system? The screening approach matters significantly for prompt injection defense quality.
- 11 stars and 19 commits suggests very early stage. How production-ready is the Fly.io deployment target vs. the AWS microvm-agent path?
- The `aws/microvm-agent` directory suggests AWS-based VM isolation. Is this Firecracker-based (like the sandboxing notes describe) or a different VM approach?
- Skills imported from git repositories — what format? Is it compatible with the `.claude/skills/` convention from Claude Code, or a QM-specific format?
- Background work (crons and watches) without human oversight — how does QM handle the "no human in real time" safety challenges that the outer loop accountability essay describes?

## Related Notes
- [OpenAI Presence — Enterprise Agent Platform](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/openai-presence-enterprise-agent-platform.md) — the managed-service alternative to QM. Presence is OpenAI-managed, model-locked, not self-hosted. QM is self-hosted, vendor-agnostic, open source.
- [Ambiance Harness — Unix Philosophy for Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/ambiance-harness-unix-philosophy-agents.md) — QM's per-scope durable sandbox and filesystem-as-memory pattern both align with Ambiance's design principles. QM scales this to multi-tenant cloud deployment.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — QM's crons/watches = NEEDLE's scheduled beads. QM's per-scope isolation = NEEDLE's worktree-per-agent. Different use cases (interactive team assistant vs. headless task fleet) but shared design vocabulary.
- [Lilian Weng: Harness Engineering for Self-Improvement](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/lilian-weng-harness-engineering-self-improvement.md) — QM is an early implementation of Pattern 3 (sub-agent and backend jobs) and Pattern 2 (file system as persistent memory) at the team-product level. The vendor-agnostic harness interface corresponds to the adapter contracts Weng describes.
- [Anthropic AI-Native SDLC Security](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/anthropic-ai-native-sdlc-security.md) — QM's three security postures and predeclared command policy implement the same principle as Anthropic's single-purpose agent identity with minimum permissions. The auto-mode classifier is a product-layer implementation of the SDLC's agent access boundary concept.
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md) — QM's auto-mode provenance labelling implements concept 17 (prompt injection defense). The per-scope sandbox implements concept 14 (sandboxing). The predeclared command policy implements concept 15 (permissions).
