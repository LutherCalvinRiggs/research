# gstack — Garry Tan's Open Source AI Software Factory

**Source:** https://github.com/garrytan/gstack
**Saved:** 2026-06-07
**Tags:** ai, tools, productivity, orchestration, agentic-ai, oss

> Personal note: Shared by @noisyb0y1. This is Garry Tan's (YC President/CEO) personal Claude Code skill stack, used in production to ship 3 production services and 40+ features in 60 days, part-time, while running YC full-time.

---

## TL;DR
gstack turns Claude Code into a virtual engineering team by giving it 23 specialist roles and 8 power tools as slash commands. A persistent headless Chromium browser daemon, a "gbrain" memory layer, multi-agent review pipelines, and an ETHOS system that injects builder principles into every skill. Works on 10 AI coding agents (Claude Code, Codex, OpenCode, Cursor, Factory, Slate, Kiro, OpenClaw, Hermes, gbrain). MIT licensed, free, used daily by its author.

## Key Metrics
- **810×** Garry Tan's 2026 logical code velocity vs 2013 baseline
- **240×** the entire 2013 year's output, in YTD 2026 alone (through April 18)
- **3 production services, 40+ shipped features** in 60 days, part-time while running YC full-time
- **1,237 GitHub contributions** Jan–Apr 2026 vs 772 in all of 2013

## Key Concepts & Terms
- **Skill**: A Markdown file (`SKILL.md`) that gives Claude Code a named role with a defined workflow, allowed tools, triggers, and preamble bash script. Invoked as `/skill-name`.
- **gbrain**: A persistent memory layer (Supabase or local PGLite) that stores design docs, learnings, builder profile, office-hours sessions, and decision logs across projects. The brain that remembers what you've built and decided.
- **Preamble**: Bash code that runs at the start of every skill invocation — checks for updates, loads session state, injects learnings, sets repo mode, logs telemetry.
- **ETHOS**: A set of builder principles (Boil the Ocean, Search Before Building, User Sovereignty) injected automatically into every workflow skill's preamble. The philosophy layer.
- **Browse daemon**: A long-lived Chromium instance that gstack's CLI talks to over localhost HTTP. Sub-second response after first start (~3s cold start, 100–200ms warm). Persists cookies, tabs, login sessions across commands.
- **Repo mode**: gstack detects what kind of repo it's in (web app, iOS app, library, etc.) and adjusts skill behavior accordingly.
- **Slop scan**: A quality scanner that detects "AI slop" — generic boilerplate, low-effort prose, over-hedged language — in generated code or docs.
- **OpenClaw**: A separate AI agent system (247K GitHub stars) that can dispatch Claude Code sessions with gstack skills via ACP.

## The 23 Skills (Specialist Roles)

### Product & Strategy
- `/office-hours` — YC-style interrogation: six forcing questions (demand reality, status quo, desperate specificity, narrowest wedge, observation, future-fit). Two modes: startup validation and builder brainstorming.
- `/plan-ceo-review` — Strategic challenge on a feature plan: scope, trade-offs, go/no-go.
- `/autoplan` — Full auto-review pipeline: runs CEO, design, eng, and DX reviews sequentially with auto-decisions using 6 decision principles.
- `/retro` — Weekly engineering retrospective.
- `/spec` — Specification generation.

### Engineering Review
- `/plan-eng-review` — Engineering review of a plan: architecture, risk, implementation quality.
- `/review` — Pre-landing PR review: SQL safety, LLM trust boundary violations, conditional side effects, structural issues.
- `/investigate` — Root cause debugging methodology.
- `/land-and-deploy` — Automated landing + deployment pipeline.
- `/ship` — Full PR preparation: plan completion check, test coverage, review army, changelog, PR body.

### Design
- `/plan-design-review` — Design review of a feature plan.
- `/design-consultation` — Design thinking for a feature.
- `/design-shotgun` — Rapid design variant generation.
- `/design-html` — Design-to-code: generates HTML prototypes.
- `/design-review` — Evaluates design quality.
- `/office-hours` (builder mode) — Design brainstorming for side projects.

### QA & Security
- `/qa` — Opens a real browser session, runs full QA against a staging URL.
- `/qa-only` — Report-only QA (no fixes).
- `/cso` — Chief Security Officer mode: OWASP Top 10, STRIDE threat modeling, dependency supply chain, CI/CD pipeline security, LLM/AI security. Two modes: daily (8/10 confidence gate) and comprehensive monthly.

### Developer Experience
- `/plan-devex-review` — Developer experience review of a plan.
- `/devex-review` — DX review of existing code.
- `/benchmark` — Model benchmarking.
- `/canary` — Canary deployment management.

### Power Tools (8)
- `/browse` + `/connect-chrome` — Headless browser with persistent state.
- `/setup-browser-cookies` — Import cookies from a real browser session.
- `/setup-deploy` — Deploy configuration.
- `/setup-gbrain` — Memory layer setup.
- `/document-release` / `/document-generate` — Documentation generation.
- `/codex` — OpenAI Codex CLI integration.
- `/learn` — Learning capture.
- `/skillify` — Convert a workflow into a new skill.

## Architecture

### Browse Daemon (the hard part)
```
Claude Code → CLI binary (Bun compiled, ~58MB) → HTTP POST → Bun.serve() → CDP → Chromium
```
- First call: ~3s cold start. Every subsequent call: ~100–200ms.
- Chromium persists tabs, cookies, localStorage, login sessions across commands.
- 30-minute idle timeout with auto-shutdown.
- State written to `.gstack/browse.json` (pid, port, auth token, binary version).
- Port: random 10000–60000, preventing conflicts in multi-workspace setups.
- Security: bound to `127.0.0.1` only. Dual-listener architecture for the pair-agent tunnel (local listener + restricted tunnel listener on separate TCP ports — physical socket separation, not header filtering).

### Why Bun (not Node.js)
- `bun build --compile` produces a single ~58MB executable. No `node_modules` at runtime.
- Native SQLite — reads Chromium's cookie database directly without `better-sqlite3`.
- Native TypeScript — no compilation step during development.
- Fast startup (~1ms binary vs ~100ms Node).

### Tech Stack
- **Runtime:** Bun v1.0+
- **Language:** TypeScript throughout
- **Browser:** Playwright / Chrome DevTools Protocol (CDP)
- **Memory:** Supabase (remote) or PGLite (local) via gbrain
- **Testing:** Bun test, LLM-as-judge evals, E2E via `claude -p`
- **CI:** GitHub Actions, diff-based test selection, two-tier (gate/periodic)

## Installation

```bash
# Single command — paste into Claude Code
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git \
  ~/.claude/skills/gstack && \
  cd ~/.claude/skills/gstack && ./setup
```

**Requirements:** Claude Code, Git, Bun v1.0+, Node.js (Windows only)

**Team mode** (auto-updates for all teammates):
```bash
(cd ~/.claude/skills/gstack && ./setup --team) && \
  ~/.claude/skills/gstack/bin/gstack-team-init required && \
  git add .claude/ CLAUDE.md && \
  git commit -m "require gstack for AI-assisted work"
```

**Multi-agent support:**
```bash
./setup --host codex      # → ~/.codex/skills/gstack-*/
./setup --host opencode   # → ~/.config/opencode/skills/gstack-*/
./setup --host cursor     # → ~/.cursor/skills/gstack-*/
```

## The ETHOS (injected into every skill)

Three principles that gstack bakes into every workflow:

**1. Boil the Ocean**
"Don't boil the ocean" was right when engineering time was the bottleneck. That era is over. Marginal cost of completeness is near-zero. The compression ratio table:

| Task | Human team | AI-assisted | Compression |
|------|-----------|-------------|-------------|
| Boilerplate / scaffolding | 2 days | 15 min | ~100× |
| Test writing | 1 day | 15 min | ~50× |
| Feature implementation | 1 week | 30 min | ~30× |
| Bug fix + regression test | 4 hours | 15 min | ~20× |
| Architecture / design | 2 days | 4 hours | ~5× |
| Research / exploration | 1 day | 3 hours | ~3× |

**2. Search Before Building**
Three layers of knowledge: (1) tried-and-true (standard patterns — check them), (2) new-and-popular (current trends — search but scrutinize for mania), (3) first principles (original observations — the most valuable). The goal is the Eureka Moment: understand what everyone does and why, then find the clear reason why the conventional approach is wrong.

**3. User Sovereignty**
AI models recommend. Users decide. The generation-verification loop: AI generates → user verifies and decides → AI never skips verification because it's confident. Two models agreeing is a strong signal, not a mandate.

## The Recommended Workflow
1. `/office-hours` — describe what you're building, get the YC forcing questions
2. `/plan-ceo-review` — challenge the plan strategically before any code
3. `/review` — PR review before landing
4. `/qa` — QA against staging URL before shipping
5. `/ship` — full PR preparation and release

## Related Projects
- **OpenClaw** (github.com/openclaw/openclaw) — 247K stars, dispatches Claude Code sessions via ACP. gstack skills work natively when Claude Code has gstack installed.
- **gbrain** — Memory/knowledge layer. Supabase-backed or local PGLite.

## Questions & Gaps for My Implementation
- How does gstack's `/learn` skill and gbrain memory interact with a NEEDLE bead queue? The two systems both capture project learnings but in different formats (`.jsonl` vs `.beads/learnings.md`).
- The `/cso` security audit is directly relevant to NEEDLE production safety — worth running it on any project before deploying an agent fleet.
- `/autoplan` runs CEO + design + eng + DX reviews sequentially — this is effectively a Dynamic Workflow avant la lettre. How does it compare to a Claude Code Dynamic Workflow doing the same thing?
- gbrain requires Supabase or PGLite — does this add operational complexity similar to running a separate database for the memory layer?
- The slop scanner is interesting for NEEDLE fleet output quality — could a NEEDLE bead incorporate `/review` + slop scan as a validation gate before closing?

## Related Notes
- [Claude Code Dynamic Workflows](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-code-dynamic-workflows-6-patterns-14-steps.md) — `/autoplan` is a manual sequential multi-skill pipeline; Dynamic Workflows automates the same orchestration pattern natively.
- [PatternsDev/skills Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/patterns-dev-skills-overview.md) — PatternsDev skills are front-end pattern libraries; gstack skills are workflow automation and specialist roles. Complementary.
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — gstack is the optimized pet model (one configured session); NEEDLE is the cattle model (headless fleet). The ETHOS "Boil the Ocean" principle maps to NEEDLE's philosophy of not skipping specification quality.
- [Claude Projects 6-Part Blueprint](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/claude-projects-6-part-blueprint.md) — gstack's ETHOS + skill CLAUDE.md injections are the engineered version of the blueprint's Identity Block + Process Block + Rules Block.
- [Agent Platform Security Checklist](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/agent-platform-security-checklist.md) — gstack's `/cso` skill operationalizes the security audit domain of the checklist.
