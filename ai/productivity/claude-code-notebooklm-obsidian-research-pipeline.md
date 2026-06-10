# Claude Code + NotebookLM + Obsidian: Self-Improving Research Pipeline

**Source:** https://x.com/monokern/status/2061044198418031017
**Saved:** 2026-06-09
**Tags:** ai, productivity, tools, research, learning, prompting

---

## TL;DR
A four-tool research stack (Claude Code + Skill Creator + NotebookLM + Obsidian) that executes full research pipelines on a single command, offloads heavy analysis to Google's infrastructure, saves everything to a local knowledge base, and learns your preferences over time. Setup under 30 minutes. The key insight: NotebookLM's processing runs on Google's compute — not your Claude tokens.

## Key Concepts & Terms
- **Skill Creator**: A Claude Code plugin that generates and installs reusable skills from plain-language descriptions. No programming required — you describe what you want, it writes and installs the code.
- **notebooklm-py**: An open-source Python library (`github.com/teng-lin/notebooklm-py`) that provides programmatic access to NotebookLM via browser automation. NotebookLM has no public API — this library fills the gap using authenticated Google sessions.
- **Pipeline skill**: A Claude Code skill that chains multiple other skills in sequence. One command triggers YouTube search → NotebookLM notebook creation → source loading → analysis → deliverable generation → Obsidian file save.
- **yt-dlp**: CLI tool for downloading/querying YouTube metadata. The YouTube search skill is built on top of it. Returns titles, channels, subscriber counts, view counts, upload dates, URLs, and engagement ratios (views-to-subscribers).
- **Vault**: Obsidian's local file system — a directory of Markdown files linked together. Every pipeline output lands here as a structured `.md` file, creating a growing corpus of past research.
- **claude.md** (in the Obsidian vault): A configuration file inside the vault that tells Claude Code how to work with you — your conventions, output preferences, analysis style. Updated by asking Claude to reflect on recent sessions and update its understanding.
- **Engagement ratio**: Views ÷ subscribers per video. Used by the `/yt-search` skill as a signal for content that resonated beyond the creator's base audience — often the most useful outlier signal.

## The Four-Layer Stack

| Tool | Role | What it contributes |
|------|------|---------------------|
| Claude Code | Execution engine | Runs commands, orchestrates pipeline, reads/writes files, calls skills |
| Skill Creator | Customization layer | Builds new skills from plain language — no programming required |
| NotebookLM | Analysis engine | Deep analysis, summaries, infographics, mindmaps, flashcards, audio — on Google's compute, not your tokens |
| Obsidian | Memory layer | Stores all output as local Markdown; Claude Code reads it back across sessions |

## The Five-Step Setup

```bash
# Step 1: Install Skill Creator (inside Claude Code, from Obsidian vault folder)
/plugin → search "skill-creator" → install → restart Claude Code

# Step 2: Create YouTube search skill
/skill-creator I want to create a skill that searches YouTube and returns
structured video results using yt-dlp — top 20 results, metadata including
title, channel, subscriber count, view count, duration, upload date, URL,
and views-to-subscribers engagement ratio. Filter to last 6 months by
default with --months flag override. Human-readable formatting.
# → installs /yt-search

# Step 3: Install notebooklm-py (separate terminal, not inside Claude Code)
pip install notebooklm-py
notebooklm login   # opens browser, authenticate with Google

# Step 4: Create NotebookLM skill
/skill-creator create a skill for notebooklm-py that can: create new
notebooks, add sources (YouTube URLs, text, files — up to 50 per notebook),
run analysis on sources, and generate deliverables including audio overview,
mindmap, flashcards, and infographic.
# → installs /notebooklm

# Step 5: Create the pipeline skill that chains everything
/skill-creator I want a YouTube research pipeline skill that: takes a
research topic, calls /yt-search for 10 relevant videos, creates a new
NotebookLM notebook via /notebooklm, adds the video URLs as sources, runs
analysis on the topic, asks if I want a deliverable (flashcards, infographic,
mindmap, audio — default none), then saves the full output as a Markdown file
to the vault and displays it in chat. Include all YouTube metadata in output.
# → installs /yt-pipeline
```

## Running the Pipeline

```
/yt-pipeline I want to research AI agent frameworks in 2026. Which frameworks
are developers actually adopting — LangGraph, CrewAI, AutoGen, Agno, or
something else? I want to understand what's driving views, where there's
disagreement in the community, what the outliers are, and what angles haven't
been covered well yet. Find 10 relevant sources, push them to a new
NotebookLM notebook, run a full analysis, and generate an infographic showing
the landscape.
```

**What happens:** Claude Code calls `/yt-search`, finds 10 videos, passes URLs to NotebookLM, creates a notebook, runs analysis, requests the infographic. Total time: ~6 minutes — most of it NotebookLM processing on Google's servers. Output: full analysis + infographic + Markdown file saved to Obsidian vault.

## The Obsidian Compounding Effect

Without Obsidian, this is a one-time research task. With it, it's a system that compounds:

- Every output lands in the vault as a structured Markdown file
- Claude Code reads the vault across sessions — it sees your linked notes, recurring topics, preferred formats
- `claude.md` in the vault is explicitly updated to reflect your work style:

```
Can we update claude.md so it better reflects my work style, analysis
approach, and output preferences based on our latest conversations?
```

Run this weekly. After a month the outputs match what you want without heavy prompting. After a year you have a trained assistant rather than a blank tool.

## The Modular Principle (The Point Nobody Mentions)

YouTube is not the point. The pipeline structure is the point. Any Claude Code-accessible data source can be the input:

| Source | Use case |
|--------|---------|
| YouTube (`yt-dlp`) | Emerging tech, community sentiment, content gaps |
| PDFs | Academic papers, industry reports, whitepapers |
| Public web pages | News, documentation, blog posts |
| Local files | Your own notes, transcripts, exported data |
| Google Drive | Existing documents and spreadsheets |
| Crypto whitepapers | Ecosystem analysis |

The pipeline skill, the NotebookLM analysis layer, and the Obsidian memory system stay identical regardless of source. Swap the first skill; keep everything else.

## Questions & Gaps
- `notebooklm-py` uses browser automation, not a real API — how fragile is this to Google UI changes? What's the fallback when it breaks?
- The `claude.md` self-update pattern ("update claude.md based on our latest conversations") is the same principle as kiro-config's steering files. How do you prevent `claude.md` from accumulating contradictions or becoming too long to be useful?
- NotebookLM's 50-source limit — what happens to research sessions that need more than 50 sources?
- The article claims "6 minutes total" — is this with a cold NotebookLM notebook or a warm one? Processing time likely varies significantly with source count and deliverable type.
- No mention of cost — `/yt-pipeline` on a complex topic could be expensive in Claude tokens. What's the practical token cost per pipeline run?
- How does this compare to the research workflow already in this repo? Our `/take-notes` skill does a similar job — fetch, analyze, save to structured Markdown — but without the NotebookLM layer or YouTube-specific tooling.

## Related Notes
- [Claude Projects 6-Part Blueprint](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/claude-projects-6-part-blueprint.md) — the `claude.md` self-update pattern is the same as the Knowledge Files + Onboarding Message pattern in the blueprint, just implemented inside the Obsidian vault rather than a Claude Project.
- [Claude Code Dynamic Workflows](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-code-dynamic-workflows-6-patterns-14-steps.md) — the `/yt-pipeline` skill is a manual static workflow; Dynamic Workflows could generate the same pipeline on the fly and adapt it per topic rather than using a fixed 10-video, fixed-analysis template.
- [gstack Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/gstack-repo-overview.md) — gstack's `/skillify` command is the same concept as Skill Creator — converting a workflow into a reusable skill. The two are convergent implementations of the same idea on different platforms.
- [Anthropic Recursive Self-Improvement](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/anthropic-recursive-self-improvement.md) — the "gets smarter every time you use it" claim is a consumer-scale version of the compounding learning loops described in the RSI piece. The mechanism (weekly claude.md updates from session history) is a manual approximation of what NEEDLE's Reflect strand does automatically.
