# PatternsDev/skills — Agent Skills for JavaScript, React, and Vue

**Source:** https://github.com/PatternsDev/skills
**Saved:** 2026-06-05
**Tags:** javascript, tools, react, prompting, frontend, oss

> Personal note: This repo is directly installable into Claude Code, Cursor, and Codex as agent skills. 58 skills covering design patterns, performance, rendering strategies, and Vue. Framework-agnostic React emphasis — Vite-first, not Next.js-first.

---

## TL;DR
58 agent-optimized skills from [patterns.dev](https://patterns.dev), packaged for Claude Code / Cursor / Codex. Covers JavaScript design patterns (Singleton, Observer, Proxy, etc.), performance patterns (bundle splitting, tree shaking, virtual lists), React rendering strategies (CSR, SSR, SSG, ISR, streaming, RSC), modern React 2026 stack guidance, AI UI patterns, and 11 Vue skills. Distinguished from Vercel's skills by being framework-agnostic, Vite-first, and educationally deep rather than prescriptively rule-heavy.

## Key Concepts & Terms
- **Agent Skill**: A `SKILL.md` file in a `skill-name/` folder that an AI coding agent loads as context when working on tasks matching the skill's `paths` or description. Tells the agent when to use a pattern and how.
- **`context: fork`**: The skill runs in an isolated git checkout (worktree). The agent can read but changes are staged before applying to the main workspace.
- **`allowed-tools`**: What the agent can do when this skill is active. Most skills are `Read, Grep, Glob` — read-only exploration before suggesting changes.
- **`related_skills`**: Skills that commonly apply together. Used by skill managers to suggest complementary installs.
- **Barrel file**: An `index.ts` that re-exports from many modules. Identified as the #1 bundle size issue in React — forces bundlers to load the entire module graph even when only one export is used. Can add 200–800ms to startup.

## Installation

```bash
# Install an entire technology stack
npx skills add PatternsDev/skills/javascript
npx skills add PatternsDev/skills/react
npx skills add PatternsDev/skills/vue

# Install individual skills by folder name
npx skills add PatternsDev/skills --skill hooks-pattern
npx skills add PatternsDev/skills --skill ai-ui-patterns
npx skills add PatternsDev/skills --skill vite-bundle-optimization

# Manual install — copy to agent skills directory
cp -r react/hooks-pattern ~/.cursor/skills/
cp -r javascript/singleton-pattern ~/.claude/skills/
```

Works with `.cursor/skills/`, `.claude/skills/`, or `.codex/skills/`.

## Skill Inventory (58 total)

### JavaScript — Design Patterns (11)
`singleton-pattern`, `observer-pattern`, `proxy-pattern`, `prototype-pattern`, `module-pattern`, `mixin-pattern`, `mediator-pattern`, `flyweight-pattern`, `factory-pattern`, `command-pattern`, `provider-pattern`

### JavaScript — Performance (14)
`loading-sequence`, `static-import`, `dynamic-import`, `import-on-visibility`, `import-on-interaction`, `route-based`, `bundle-splitting`, `prpl`, `tree-shaking`, `preload`, `prefetch`, `third-party`, `virtual-lists`, `compression`

### JavaScript — Rendering (2)
`islands-architecture`, `view-transitions`

### JavaScript — Build (2)
`js-performance-patterns`, `vite-bundle-optimization`

### React — Design Patterns (7)
`hooks-pattern`, `hoc-pattern`, `compound-pattern`, `render-props-pattern`, `presentational-container-pattern`, `ai-ui-patterns`, `react-2026`

### React — Rendering (8)
`client-side-rendering`, `server-side-rendering`, `static-rendering`, `incremental-static-rendering`, `streaming-ssr`, `progressive-hydration`, `react-server-components`, `react-selective-hydration`

### React — Optimization (3)
`react-render-optimization`, `react-data-fetching`, `react-composition-2026`

### Vue (11)
`components`, `composables`, `script-setup`, `state-management`, `provide-inject`, `dynamic-components`, `async-components`, `render-functions`, `renderless-components`, `container-presentational`, `data-provider`

## Most Distinctive Skills Worth Installing

### `react-2026` — Modern React Stack Guide
Opinionated recommendations for mid-to-senior developers choosing a React stack in 2026. Key stances:
- CRA deprecated in early 2025 — use a framework or Vite
- Full-stack apps needing SSR/SEO: Next.js or Remix
- SPAs, internal tools, dashboards: Vite + React Router or TanStack Router
- Server state: TanStack Query. Complex global state: Zustand or Redux
- Use React 19 APIs: `ref` as prop (no `forwardRef`), `use()`, `useActionState`, `useOptimistic`, React Compiler for auto-memoization
- React Server Components: early reports showed 20%+ bundle reduction

### `ai-ui-patterns` — AI Interface Design
Patterns for building streaming chat UIs, intelligent assistants, and AI-driven interactions in React. Covers:
- Vercel AI SDK `useChat` hook for conversation state and streaming
- API keys stay on the server — Next.js route handlers or separate backend
- Enable streaming for real-time output; disable input during streaming
- Reusable `ChatMessage` / `InputBox` components decoupled from data-fetching
- Works with OpenAI, Anthropic, or Gemini via unified SDK interface

### `vite-bundle-optimization` — Critical Performance
The most practically impactful skill for Vite users. Ordered by impact:
1. **Avoid barrel files** (CRITICAL — 200–800ms startup penalty): import directly from source files, not index re-exports. Use `vite-plugin-barrel` to auto-transform.
2. **`manualChunks` for vendor splitting**: separate React, UI libraries, and utilities into separate chunks with long-term cache headers
3. **`optimizeDeps` configuration**: pre-bundle only what you use
4. **Compression**: Gzip/Brotli via `vite-plugin-compression`
5. **Dead code elimination**: `import.meta.env` conditions stripped at build time
- Diagnosis: `npx vite-bundle-visualizer` — run this before any optimization pass

### `react-composition-2026` — Composition Patterns
Core principle: composition over configuration. Key patterns:
- **Replace boolean props with composition**: 4 booleans = 16 states, most untested. Compose children instead.
- **Polymorphic `as` prop**: flexible element rendering without wrapping
- **Slot pattern**: layout components that accept named regions
- **Headless components**: hooks-based behavior without markup (renderless)
- **Context interface pattern**: swappable state implementations

## How SKILL.md Files Are Structured

Every skill follows this schema:
```yaml
---
name: skill-name
description: One sentence — when to activate this skill
context: fork              # or: none, remote
allowed-tools: Read, Grep, Glob
paths:
  - "**/*.tsx"
  - "**/*.jsx"
license: MIT
metadata:
  author: patterns.dev
  version: "1.1"
related_skills:
  - "complementary-skill"
---
# Pattern Title

## When to Use
## Instructions   ← what the agent should do
## Details        ← educational depth, code examples, avoid/prefer pairs
## Source
```

The agent reads `description` to decide relevance, `instructions` to know what to do, and `details` for the educational depth and code examples.

## PatternsDev vs. Vercel Skills — Key Differences

| Dimension | Vercel Skills | PatternsDev Skills |
|-----------|--------------|-------------------|
| Framework | Next.js-first | Framework-agnostic, Vite-emphasis |
| Structure | 3 monolithic skills (57 rules in one) | 58 individual focused skills |
| Vue | None | 11 skills |
| AI UI | None | `ai-ui-patterns` skill |
| React 19 | Partial (`forwardRef` removal) | Full coverage (`use()`, Actions, Compiler) |
| Bundle optimization | Next.js-specific APIs | Vite-first (`vite-plugin-barrel`, `manualChunks`) |
| Teaching style | Prescriptive rules with CRITICAL/HIGH ratings | Educational with "when to use" context |
| State management | Favors SWR (their own library) | TanStack Query as primary, SWR as alternative |

## Questions & Gaps
- Are these skills compatible with NEEDLE's bead-level orchestration — could a NEEDLE worker activate a skill and have it inform the agent's behavior per-bead?
- `context: fork` runs in an isolated worktree — how does this interact with the NEEDLE Production Safety Guide's filesystem scope controls?
- The `ai-ui-patterns` skill teaches Vercel AI SDK patterns — does it cover Anthropic's native streaming API or only via the SDK abstraction?
- Version 1.1 — how actively is this maintained? React 19 / RSC is covered but no mention of React 20 direction.
- No TypeScript-specific skill — type-safety patterns for React are spread across several skills rather than consolidated.

## Related Notes
- [Claude Code Dynamic Workflows](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-code-dynamic-workflows-6-patterns-14-steps.md) — the `ai-ui-patterns` skill teaches how to build the front-end for AI interfaces; Dynamic Workflows teaches how to orchestrate the back-end agent logic that powers them.
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — the `context: fork` skill isolation model is the same worktree pattern Anthropic's own harness research describes for per-agent isolation.
