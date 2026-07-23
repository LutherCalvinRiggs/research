# OpenAI Presence — Enterprise AI Agent Platform

**Source:** https://openai.com/index/introducing-openai-presence/ (OpenAI, July 22, 2026)
**Secondary:** https://venturebeat.com/orchestration/openai-unveils-presence-a-new-platform-that-lets-enterprises-launch-and-manage-realtime-voice-agents-and-chatbots
**Saved:** 2026-07-23
**Tags:** ai, tools, infrastructure, orchestration, agentic-ai, technology

> **Availability:** Limited general availability, July 22, 2026. Not self-serve — requires OpenAI account team contact. Deployments led by OpenAI Forward Deployed Engineers (FDEs) and select global systems integrators. Pricing not disclosed.

---

## TL;DR
OpenAI Presence is a managed enterprise deployment platform for production AI agents — not raw model access, but a fully packaged system including policies, guardrails, approved actions, simulations, evaluation tools, and a Codex-powered improvement loop. The central thesis: the challenge is no longer proving agents work, it's making them reliable enough for high-value production use and keeping them accurate as conditions change. Available today for voice and chat. Not self-serve.

---

## The Problem Presence Solves

The challenge for enterprises has shifted from "can AI agents work?" to "can we make them reliable enough for high-value production work?" Two specific problems:

1. **Reliability in production** — agent behavior must be correct even as products, policies, and user behavior change over time
2. **Control without starting over** — teams need to update agent behavior (when policies change, when products change) without rebuilding the deployment from scratch

Presence packages the answer to both: systems for deploying agents, evaluating them in production, and updating them in a controlled way.

---

## What Presence Is (The Six Components)

OpenAI describes Presence as bringing together:

| Component | What it does |
|-----------|-------------|
| **Policies and SOPs** | What the agent can do, what it must avoid, how it should handle edge cases |
| **Guardrails** | Intervene when an interaction moves outside company boundaries |
| **Approved actions** | The specific system actions the agent is permitted to take |
| **Simulations** | Pre-launch testing against common requests, edge cases, and high-risk scenarios |
| **Evaluation tools** | Graders that check outcome, policy compliance, tool use correctness, escalation appropriateness |
| **Codex-powered improvement loop** | Production signals (sessions, escalations, quality metrics) → Codex investigates and proposes updates → team tests against current version → controlled rollout |

The Codex improvement loop is the most distinctive element — a continuous improvement mechanism where Codex (OpenAI's code-capable model) autonomously investigates where the agent is underperforming and proposes specific updates that humans then test and approve before rollout.

---

## The Deployment Model

**Each deployment starts with a specific job:**
- Resolving billing disputes
- Processing insurance claims
- Handling internal IT service requests

The agent receives only the knowledge and system access required for that specific job — not broad access. Company sets policies: what it can do, when it needs approval, when a human takes over.

**OpenAI's involvement is hands-on:**
- OpenAI works alongside each customer to identify high-value workflows
- Connect knowledge and systems
- Establish permissions and policies
- Test the agent
- Bring it into production

This is not a self-serve API product. It's closer to a managed service with embedded deployment engineers.

---

## Claimed Performance Data

**OpenAI's own phone support line** (1-888-GPT-0090):
- Handles open-ended requests, caller verification, account context, approved actions
- Met or exceeded human support benchmarks within weeks of launch
- Resolves **75% of inbound issues** without human assistance
- Codex-powered improvement loop **reduced human handoffs by 15 percentage points in 10 days**

**Enterprise customers at launch:**
- **BBVA** — exploring AI-powered voice support for banking in Mexico
- **SoftBank** — testing natural Japanese-language customer conversations; frontline teams rated quality highly
- **IAG (Insurance Australia Group)** — exploring support during high-demand events (severe weather, natural disasters)

---

## The Market Context: Services-Led Enterprise AI

Presence is part of a broader industry shift from "sell model access" to "sell managed deployment":

- **OpenAI Deployment Company** — launched May 2026, enterprise AI consulting and integration firm, investment from Bain & Company. Also offers model customization and fine-tuning programs.
- **Anthropic Ode** — consulting organization with forward-deployed engineers helping companies integrate Claude into complex workflows, launched approximately one week before Presence.

Both represent the same recognition: enterprises need more than model access. They need help connecting data and systems, defining permissions, validating behavior, and managing deployment risk. The question is whether this is a transitional service (companies eventually learn to do this themselves) or a permanent business model (complexity is high enough that managed deployment stays valuable).

---

## What Makes Presence Distinct vs. Raw API Access

| Raw API | Presence |
|---------|---------|
| You build everything | OpenAI provides the deployment infrastructure |
| You define evaluation | Graders and simulations built in |
| You manage updates | Codex investigates and proposes; you approve |
| Self-serve | Not self-serve; requires OpenAI FDEs |
| Any use case | Specific task-scoped deployments |
| You manage agent behavior changes | Continuous improvement loop with controlled rollout |

The VentureBeat framing: "If your business has been interested in using AI agents, but you aren't sure how to stitch together OpenAI's models, APIs, internal systems, security controls and evaluation tools into something reliable, Presence is designed to simplify that process."

---

## Unanswered Questions (from the announcement)

- **Pricing** — not disclosed
- **Geographic limits** — not disclosed
- **Contractual terms** — not disclosed
- **Cost of the FDE engineering work** — not disclosed
- **Whether Presence can use non-OpenAI models** — not addressed (including open-weights alternatives)
- **Whether this works on existing OpenAI API voice customers** — OpenAI states it will "continue supporting voice customers with access to frontier models through the OpenAI API," suggesting Presence and API access are separate tracks

---

## Questions & Gaps
- The Codex improvement loop ("investigates signals and proposes updates") is the most interesting technical claim. What does "proposes updates" mean concretely — changes to prompts, changes to policies, changes to tool definitions, or changes to the evaluation criteria? The mechanism matters significantly for whether this is genuinely self-improving or structured prompt iteration.
- The 75% resolution rate and 15-point handoff reduction in 10 days are compelling but self-reported by OpenAI. Independent verification would significantly strengthen these claims.
- "Agent receives only the knowledge and system access required for that job" — this is the minimum-privilege principle from the security notes. How is this enforced technically? API-key scoping, RAG retrieval scoping, or something else?
- Presence vs. the OpenAI API: companies that have already built on the API are effectively on a different track. Is there a migration path, or are these permanently separate product lines?
- The BBVA/SoftBank/IAG deployments are described as "exploring" — not yet live at scale. The 75% resolution rate comes from OpenAI's own support line. Real enterprise results at production scale haven't been disclosed yet.

## Related Notes
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md) — Presence is a commercial implementation of many of the 20 concepts: policies (concept 5 — config files), guardrails (concept 14 — sandboxing/permissions), simulations (concept 18 — pre-commit gates), evaluation tools (concept 19 — tracing), and the Codex improvement loop (concept 20 — outcome metrics → improvement). The managed service model removes the need for customers to implement concepts 14–20 themselves.
- [AI Biggest Winners Have the Lowest Margins](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/ai-biggest-winners-lowest-margins.md) — Presence is the product-layer implementation of the "AI as infrastructure" thesis from that article: not a tool employees adopt, but a system that runs inside the existing workflow. The BBVA banking voice support and IAG insurance claims deployments are exactly the low-margin, labour-heavy industries the article describes.
- [Meta's AI Bot Instagram Account Takeover](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/metas-ai-bot-instagram-account-takeover.md) — Presence's policy + guardrails + approved actions + escalation architecture is the engineered version of what Meta's bot lacked. The confused deputy attack succeeded because the bot had modification authority without verification gates. Presence explicitly packages those gates.
- [Claude Fable 5: Self-Improving Agent System](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-fable-5-self-improving-agent-system.md) — Presence's Codex improvement loop is a commercial product version of the same self-improvement pattern: production signals → propose updates → human approval → controlled rollout. The 5-stage memory progression (Investigate → Verify → Distill → Consult) maps onto how Codex is described as investigating signals and proposing changes.
- [Own the Outer Loop — Agentic Accountability](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/own-the-outer-loop-agentic-accountability.md) — Osmani's Verdict concept (the human production decision before work enters the dependent system) is exactly what Presence's "teams can test each proposed change against the version in production, then approve a controlled rollout" implements commercially. Presence sells the outer loop infrastructure.
