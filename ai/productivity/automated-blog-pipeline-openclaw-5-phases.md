# Automated Blog Pipeline: 5 Phases, 3 Specialized Agents with OpenClaw

**Source:** https://x.com/rethink_hub/status/2063992615721435154 (pasted content by @rethink_hub / PirateCoder)
**Saved:** 2026-06-10
**Tags:** ai, productivity, tools, orchestration, agentic-ai, writing

---

## TL;DR
An indie developer replaced 3–4 hours of manual blog work per post with a daily cron pipeline: 5 phases, 3 specialized agents (Arc Expert, Writing Expert, Publisher), OpenClaw for orchestration, real Google Analytics data for information gain, and a mandatory SEO audit gate that blocks publication if anything fails. 28+ posts published with zero manual writing. Wall time per post: 15–20 minutes.

## Key Concepts & Terms
- **OpenClaw**: The orchestration layer — spawns agents, manages pipeline state, handles failure recovery. Each agent runs with its own focused knowledge base.
- **Durable Pipeline Skill**: A stateful OpenClaw skill that tracks which phase completed successfully. If the pipeline crashes mid-cycle, it resumes from the last successful step rather than restarting from scratch.
- **Information gain**: The SEO ranking signal that's hardest to fake — real data only you have. "Arc users generate 3,300+ AI summaries per month" is uncopiable. "AI apps are popular" is not. Real first-party data is the content moat.
- **Knowledge base over prompting**: Instead of instructing an agent to "use real screenshots," give it a curated inventory of real screenshot filenames. Instead of "include accurate stats," give it the actual data. Constraints beat instructions.
- **Specialized agent**: A single-purpose agent with a focused knowledge base and narrow mandate. "The Arc Expert only knows Arc features. The Writing Expert only writes blog posts. The Publisher only publishes." Specialization beats general-purpose.
- **SEO audit gate**: A validation layer that checks title length, keyword placement, meta description, internal link count, FAQ existence, author field, CTA. If anything fails, the pipeline stops. No SEO-challenged posts go live.

## The Five-Phase Pipeline

```
Phase 1: Research
  → Search trending topics (Android problems people are actually searching for)
  → Check what's already been covered, find keyword gaps
  → Map topic to a product feature
  → Pull real Google Analytics data: monthly AI summaries, custom action runs,
    user geography, engagement metrics

Phase 2: Arc Expert Agent
  → Spawned by OpenClaw with dedicated knowledge base
  → Reads: feature docs, real screenshot filenames, use cases, capabilities
  → Returns: feature description, key capabilities, 3-4 real-world use cases,
    exact screenshot filenames (from validated inventory — no hallucinated images),
    Play Store link

Phase 3: Writing Expert Agent
  → Trained for specific blog standards:
    - SEO: title under 60 chars, meta description 150–160 chars, keyword placement
    - Voice: first-person "I/my" (indie dev, not corporate)
    - Structure: hook → problem → solution → data → FAQ → CTA
    - Mandatory: real Arc usage stats in every post

Phase 4: SEO Audit Gate
  → Validates: title length + keyword, meta description length/structure,
    internal links (min 2), FAQ section, author field, voice consistency,
    Play Store CTA
  → HARD STOP if anything fails — nothing goes live with SEO issues

Phase 5: Publish + Announce
  → Blog: sets post to published, commits to repo, pushes to site
    (static site rebuilds automatically)
  → X: writes tweet (hook + summary + link + hashtags),
    publishes via X API v2 with hero image attached
```

**Cadence:** Daily cron. Fully unattended from topic research to tweet.

## The Three Agents

| Agent | Knowledge base | Single job |
|-------|---------------|-----------|
| Arc Expert | Feature docs, screenshot inventory, use cases, Play Store link | Returns accurate product information — never guesses |
| Writing Expert | SEO guidelines, voice rules, post structure, usage stats | Writes the post to spec |
| Publisher | Blog repo access, X API credentials | Commits, pushes, tweets |

## The Stack

| Component | Tool |
|-----------|------|
| Orchestration | OpenClaw |
| Pipeline state | Durable Pipeline Skill (stateful, resumable) |
| Real usage data | Google Analytics API |
| Site publishing | Git + static site (auto-rebuild on push) |
| Social publishing | X API v2 (with image attachment) |

## The Five Learnings (Most Transferable)

**1. Validation layers are non-negotiable.**
Without the SEO audit gate, agents publish poor content. "They don't know what they don't know." The gate doesn't fix the writing — it prevents bad writing from going live. Every pipeline needs a layer that can fail work automatically.

**2. Specialized agents beat general-purpose ones.**
A single agent asked to research topics, write an SEO post, validate it, and publish it will be mediocre at all four. Three agents, each with a narrow knowledge base and one job, are reliable.

**3. Knowledge bases beat prompting.**
Instructions like "use real screenshots" fail because the agent can't verify what's real. A curated inventory of real filenames makes hallucination structurally impossible. Data beats instruction.

**4. State machines prevent duplicate work.**
Resumable pipelines that track completed phases are non-negotiable for any multi-step workflow. A crash in Phase 4 should not re-run Phases 1–3. State tracking is the difference between a pipeline and a script you have to restart manually.

**5. Real first-party data is the moat.**
Anyone can prompt an AI for a blog post. Nobody else can write "Arc users generate 3,300+ AI summaries per month" — that's proprietary data that flows from a real product. First-party data injected into AI-generated content is what makes it uncopiable and what signals genuine information gain to search engines.

## What He Still Does Manually

- Topic direction — chooses which feature areas to focus on
- Spot checks on published posts
- Strategic decisions about which features to highlight

The 90/10 split: pipeline handles execution, human handles direction.

## Questions & Gaps
- How does the SEO audit gate handle edge cases — e.g. a post legitimately not needing a FAQ section? Is it configurable or hard-coded?
- The screenshot inventory approach solves hallucination by giving the agent only real filenames. What's the maintenance burden when screenshots change as the app updates?
- Google Analytics API integration — is this a live pull on each pipeline run or a cached dataset updated periodically?
- OpenClaw spawns a fresh Arc Expert agent per pipeline run. Is this stateless (reads from docs every time) or does it have persistent memory across runs?
- The X announcement is automated — what's the quality distribution across tweets? Is there a tweet validation step, or only the blog SEO gate?

## Related Notes
- [Loop Engineering: 14-Step Roadmap](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-14-step-roadmap.md) — this pipeline is a textbook loop implementation: one automation (daily cron), skills (Arc Expert + Writing Expert knowledge bases), sub-agents (maker/checker separation), state file (Durable Pipeline Skill), and a hard gate (SEO audit). The roadmap's MVL checklist maps exactly.
- [Loop Engineering — Addy Osmani](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-addy-osmani.md) — Osmani's "build it one time, don't prompt any of the steps" is what this developer built. Also validates his "specialized agents beat general-purpose" point from the essay.
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — the SEO audit gate is the independent verifier pattern implemented as a hard stop. The Writing Expert doesn't validate its own work; a separate audit phase does.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the Durable Pipeline Skill's crash-resumption pattern is the same principle as NEEDLE's exhaustive outcome table: every possible phase state has a defined handler, including failure.
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — this pipeline passed Jed's cattle-readiness test: stateless, headless, verifiable output, automated gate. The developer is the human who sets topic direction; the pipeline is the fleet that executes it.
- [gstack Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/gstack-repo-overview.md) — gstack's `/document-release` skill (auto-updates docs after every change) is the same knowledge-base-maintenance pattern this developer uses for the Arc Expert's screenshot inventory and feature docs.
