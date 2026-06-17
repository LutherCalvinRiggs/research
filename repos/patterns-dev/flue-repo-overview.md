# Flue — The TypeScript Agent Harness Framework

**Source:** https://github.com/withastro/flue
**Saved:** 2026-06-17
**Tags:** ai, tools, javascript, infrastructure, orchestration, agentic-ai

> From the team that built Astro. 1.0 Beta released June 16, 2026. MIT licensed.

---

## TL;DR
Flue is a TypeScript framework for building autonomous AI agents and deterministic AI workflows with zero lock-in. Central premise: an agent is a model running inside a harness — the harness provides sessions, tools, skills, filesystem access, and a secure sandbox. Flue is the first framework to make that harness-driven architecture (used by Claude Code and Codex) available to any developer for any model on any deployment target.

## Key Concepts & Terms

- **Harness**: The environment the LLM uses to navigate the world. Without a harness, a model is a brain in a jar — no memory, no tools, no ability to act. The harness provides: filesystem, tools, sandbox (for safe action), context (for long tasks), and subagents (for parallelism). Every Flue agent is configured through its harness.
- **Agent** (`src/agents/<name>.ts`): A stateful, autonomous entity that maintains context across conversations and events. Defined via `createAgent()`. Exposed over HTTP at `/agents/<name>/:id`. Solves open-ended problems on its own.
- **Workflow** (`src/workflows/<name>.ts`): A finite, deterministic automation where your code drives the model from input to result. Defined via `export async function run(...)`. Exposed at `POST /workflows/<name>`. Best for background jobs, document transformation, code review, CI tasks.
- **Agent vs. Workflow**: Agents solve open-ended problems autonomously; workflows run the exact steps you define. Both share the same foundational core and can be mixed in the same repo.
- **Session**: One conversation thread within an agent harness. `harness.session()` opens it. History is persisted so later operations can continue from where earlier work ended.
- **Skill**: Reusable instructions and supporting resources imported as `SKILL.md` files. Agents load them on demand for specialized work. Bundled with the application; also discovered from `<cwd>/.agents/skills/` in the sandbox at runtime.
- **Channel**: Inbound HTTP event integration (Slack, GitHub, Discord, Linear, Stripe, Telegram, WhatsApp, Zendesk, Teams, Intercom, etc.). A channel verifies the provider request, parses it to typed data, and calls your handler. Channels are inbound only — use the provider SDK for outbound calls.
- **Sandbox**: A secure environment where agents act and run code. Built-in virtual sandbox (`local()`), or connect to remote providers (E2B, Daytona, Modal, Cloudflare, Vercel, etc.).
- **Durable execution**: On Cloudflare (Durable Objects backed) and Node.js (with persistence adapter), Flue recovers interrupted work conservatively — never re-runs tool calls that completed, surfaces interrupted calls as unknown outcome, resumes interrupted responses from durably stored partial output.
- **Blueprint**: A Markdown implementation guide returned by `flue add`. Used by coding agents to scaffold channel, sandbox, database, or tooling integrations automatically: `flue add channel slack --print | codex`.

## Architecture

```
Agent module (agents/<name>.ts)
  └─ AgentInstance (URL: <id>)
     └─ Harness (initialized runtime environment)
        └─ Session (one conversation thread)
           └─ Operation (prompt / skill / task / shell call)
              └─ Turn (one LLM round-trip)

Workflow (workflows/<name>.ts)
  └─ Workflow run (unique runId per invocation)
```

## Creating an Agent

```typescript
// src/agents/triage.ts
import { createAgent, type AgentRouteHandler } from '@flue/runtime';
import { local } from '@flue/runtime/node';
import triage from '../skills/triage/SKILL.md' with { type: 'skill' };
import verify from '../skills/verify/SKILL.md' with { type: 'skill' };
import * as githubTools from '../tools/github.ts';

// Protect your agent over HTTP
export const route: AgentRouteHandler = async (_c, next) => next();

// Compose the harness
export default createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  tools: [...githubTools],
  skills: [triage, verify],
  sandbox: local(),
  instructions: `Triage a bug report end-to-end: reproduce the bug,
    diagnose the root cause, verify whether intentional, attempt a fix.`,
}));
```

## Creating a Workflow

```typescript
// src/workflows/summarize.ts
import { createAgent, type FlueContext } from '@flue/runtime';

const summarizer = createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Summarize the supplied document clearly and concisely.',
}));

export async function run({ init, payload }: FlueContext<{ text: string }>) {
  const harness = await init(summarizer);
  const session = await harness.session();
  const response = await session.prompt(payload.text);
  return { summary: response.text };
}
```

Run locally: `flue run summarize --target node --payload '{"text":"..."}'`

## Channel Integration

```typescript
// src/channels/slack.ts
import { createSlackChannel } from '@flue/slack';
import { WebClient } from '@slack/web-api';

export const client = new WebClient(process.env.SLACK_BOT_TOKEN);

export const channel = createSlackChannel({
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
  async events({ payload }) {
    if (payload.type !== 'event_callback') return;
    // dispatch to agent or application code
  },
});
```

Scaffolded by: `flue add channel slack --print | codex`

## Ecosystem (1,246 files)

**17 first-party channels**: Slack, Discord, GitHub, Linear, Teams, Telegram, WhatsApp, Zendesk, Intercom, Stripe, Shopify, Notion, Messenger, Google Chat, Resend, Twilio, Salesforce Marketing Cloud

**10 sandbox integrations**: E2B, Daytona, Modal, Cloudflare, Vercel, Boxd, Cloudflare Shell, exedev, islo, Mirage

**8 database adapters**: Postgres, MySQL, libSQL, MongoDB, Redis, Valkey, Supabase, Turso

**3 observability integrations**: OpenTelemetry, Braintrust, Sentry

**Deploy targets**: Node.js, Cloudflare Workers, GitHub Actions, GitLab CI/CD, Railway, Render, Fly, SST, AWS, Docker

## Package Map

| Package | Purpose |
|---------|---------|
| `@flue/runtime` | Core: harness, sessions, tools, sandbox |
| `@flue/cli` | CLI and build/dev tooling (`flue` binary) |
| `@flue/sdk` | Client SDK for consuming deployed agents/workflows |
| `@flue/react` | Frontend React hooks (`useAgent`, `useWorkflow`) |
| `@flue/opentelemetry` | OTel tracing adapter |
| `@flue/postgres` | Postgres persistence adapter |

## Design Principles

1. **Harness-first**: Fill the harness with context (instructions, tools, skills, sessions, files, MCP servers); point a model at it; let it solve the task. No scripting required.
2. **Open by default**: Open models (any LLM provider), open sandboxes (any provider), open deploys (any target). No lock-in at any layer.
3. **AI-first**: Designed to be used alongside a coding agent. Setup, scaffolding, and several workflows are designed around handing a prompt to Claude Code or Codex and letting it do the work.

## Durable Execution

Cloudflare deployment gets strongest durability: Durable Object-backed SQLite, per-instance queues, `dispatchId` for idempotency. Key recovery guarantee: tool calls that completed are never re-run. Interrupted tool calls surface as unknown outcome — never assumed complete.

Node.js: same conservative reconciliation with a persistence adapter. Without one, state is in-memory and lost on restart.

## Questions & Gaps
- Flue is built by the Astro team — how does the web-framework design philosophy translate to agent frameworks? Are there assumptions baked in that favor web-developer workflows over, e.g., data engineering or ML workflows?
- The channel abstraction is deliberately inbound-only. For bidirectional workflows (agent reads from Slack AND posts back), how is the outbound side composed alongside the inbound channel?
- Skills are bundled at build time via static imports. How does this interact with dynamically discovered skills in the sandbox at runtime (`.agents/skills/`)? Which takes precedence?
- The `flue add channel slack --print | codex` pattern (blueprint piped to coding agent) is genuinely novel — how reliable is this across the 17 channels?
- No explicit mention of multi-agent orchestration patterns like fan-out-and-synthesize. Is this handled through subagents and Dynamic Workflows style patterns, or is there a different abstraction?

## Related Notes
- [Claude Code Dynamic Workflows — Anthropic](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/dynamic-workflows-anthropic-primary-source.md) — Dynamic Workflows write ad-hoc JS harnesses on the fly; Flue is a structured TypeScript harness framework. They're complementary — Flue gives you the foundation; Dynamic Workflows gives the agent autonomy to orchestrate within that foundation.
- [gstack Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/gstack-repo-overview.md) — gstack's skills (SKILL.md format) are directly compatible with Flue's skill import system. The `with { type: 'skill' }` import attribute is the standardized mechanism gstack skills also use.
- [Building a Good Vertical Agent](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/building-good-vertical-agent-context-hierarchy.md) — Flue's harness composition (L1 instructions + skills as L2 + sandbox access as L3) maps exactly onto the L1/L2/L3 context hierarchy framework. Flue provides the infrastructure; that essay provides the discipline for what to put in it.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — NEEDLE is a Rust fleet orchestrator for headless workers via bead queue. Flue is a TypeScript harness framework for building the agents themselves. They operate at different layers and could be complementary.
- [PatternsDev/skills Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/patterns-dev-skills-overview.md) — PatternsDev skills use the same SKILL.md format. They'd be importable directly into Flue agents via the `with { type: 'skill' }` import attribute.
