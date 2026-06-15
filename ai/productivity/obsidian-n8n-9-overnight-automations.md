# 9 Things My Obsidian Vault Does While I Sleep — N8N Automation Stack

**Source:** https://x.com/DamiDefi/status/2063992615721435154 (pasted content by @DamiDefi)
**Saved:** 2026-06-11
**Tags:** ai, productivity, tools, research, infrastructure, learning

---

## TL;DR
Nine N8N automations running against an Obsidian vault: a 6am daily synthesis brief (non-obvious connections, patterns, contradictions, best capture), Telegram-to-vault instant capture, crypto morning brief (live data vs. documented theses), 11pm inbox triage, 7am thesis contradiction check, Readwise resurfacer, weekly deep connection session, source aging alert, and midnight GitHub backup. Total cost: $8–15/month. Build time: one weekend. "The vault is not where I go to find things I saved. It is a system that brings things to me."

## Key Concepts & Terms
- **N8N**: Open-source workflow automation tool (like Zapier but self-hostable). Connects APIs, reads/writes files, calls Claude, sends Telegram messages. Runs unattended on a VPS while the laptop is closed.
- **Daily synthesis brief**: Not a summary — a synthesis. Four sections: non-obvious connections between separately captured notes, a pattern forming across three or more captures, a contradiction between notes, and the single capture most worth developing. The distinction matters: summarizing what's already in the vault has zero value; synthesizing across it does.
- **Thesis contradiction check**: Reads 02-Ideas (own positions) against 01-Sources (last 30 days of captures). Looks specifically for conflict, not confirmation. "Do not look for agreement. Do not validate. Find the conflict."
- **Inbox processor**: Nightly triage of daily captures. Three categories: DEVELOP (signal worth developing), ARCHIVE (real but duplicates something existing — names the existing note), DELETE (fleeting thought that hasn't survived 6 hours). Produces recommendations, not actions — human reviews and decides.
- **Readwise resurfacer**: Pulls 20 random highlights from Readwise older than 90 days, compares them against recent vault captures, surfaces non-obvious connections between old reading and current thinking.
- **Source aging alert**: Every Monday, flags source notes older than 60 days with no reaction paragraph. A source note without a documented reaction is captured information that was never converted into owned thinking. The alert is the enforcement mechanism.
- **Weekly deep connection session**: 30-day cross-vault analysis — different from the daily brief. Specifically surfaces: emerging theses not yet explicitly named, full contradiction map across documented positions, knowledge gaps (what perspective is clearly absent), and one highest-leverage action for the week.
- **Reaction paragraph**: A section in source notes documenting the reader's own thinking about the source, typically under a "My take:" or "Reaction:" heading. Distinguishes a note that records what someone else said from a note that represents owned thinking.

## The Nine Automations

### 1. Daily Synthesis Brief — 6:00am
Reads every note modified in the last 7 days, passes to Claude, outputs four sections:
- **Connections**: Two non-obvious links between separately captured notes. If the connection is obvious it doesn't qualify.
- **Pattern**: One theme appearing across three or more notes
- **Contradiction**: Two notes where positions conflict, with quotes from both
- **Best capture**: The single note most worth developing, and why

```
System: You are reading an Obsidian vault. Synthesise, do not summarise.
Prompt: Read these vault notes from the last 7 days. Produce four sections:
Connections (two non-obvious links — name both notes, obvious connections don't qualify),
Pattern (one theme across three or more notes, one sentence),
Contradiction (two notes where positions conflict, quote both),
Best capture (single note most worth developing and why).
Do not summarise. Synthesise.
```

Output: `00-Inbox/brief-[date].md`

### 2. Telegram Capture Pipeline — Continuous
Any message sent to a Telegram bot creates a markdown note in 00-Inbox within 30 seconds. Zero friction capture from anywhere — walks, meetings, mobile.

```javascript
const msg = $input.first().json.message;
const text = msg?.text || "(no text)";
const date = new Date().toISOString().split("T")[0];
return [{ json: {
  filename: "00-Inbox/" + date + "-capture.md",
  content: "# Capture\n" + text + "\nDate: " + date
}}];
```

The mechanism: friction reduction. When capture costs nothing, everything worth keeping gets captured.

### 3. Crypto/Domain Morning Brief — 5:45am
Pulls live data from relevant APIs (price, volume, social volume, sentiment) for tracked assets/topics. Reads current thesis notes from 03-Projects. Claude compares live data against documented positions.

```
Prompt: Here is today's data for assets/topics in my research scope.
Here are my current thesis notes.
For each thesis, tell me: does today's data support, challenge,
or add new dimension to the position?
Be specific. Reference the data. Do not tell me what is already in the thesis.
```

Output: `00-Inbox/brief-[date].md` (lands 15 minutes before the synthesis brief)

### 4. Inbox Processor — 11:00pm
Reads everything added to 00-Inbox that day, reads full vault for context, produces triage recommendations:

```
Prompt: For each Inbox note, classify it:
DEVELOP (signal worth becoming a real note — explain why in one sentence),
ARCHIVE (real idea but duplicates something in vault — name the existing note),
DELETE (fleeting thought that hasn't held up).
Produce a numbered list. One line per note. Do not process. Just classify.
```

Output: `00-Inbox/triage-[date].md` — reviewed in ~2 minutes next morning.

### 5. Thesis Contradiction Check — 7:00am
Reads all notes in 02-Ideas (own positions). Reads all notes in 01-Sources modified in last 30 days. Looks specifically for conflict.

```
Prompt: Here are my current thesis notes. Here are source notes from the last 30 days.
Your only job is contradiction. For each idea, tell me whether any recent source
contradicts, complicates, or materially updates that position.
Do not look for agreement. Find the conflict.
If there is no conflict for a given idea, say so in one word: Clear.
```

Output: `00-Inbox/contradictions-[date].md`

### 6. Readwise Resurfacer — 10:00pm
Pulls 20 random highlights older than 90 days (sorted by least recently reviewed). Reads recent vault captures from last 14 days. Looks for non-obvious resonance between old reading and current thinking.

```
Prompt: Here are 20 old highlights and here are my recent vault captures.
For each highlight, does it connect to anything in my recent captures
in a way that is not obvious?
If yes, name the specific recent note and explain in one sentence.
If no connection exists, skip it. Only surface non-obvious connections.
```

Output: Telegram message (not a vault file — arrives as a nightly notification)

### 7. Weekly Deep Connection Session — Sundays 11:00pm
Reads entire vault modified in last 30 days. Four outputs:

```
Prompt: Read everything added to my vault in the last 30 days. Produce:
Emerging thesis (what position is forming that I have not explicitly named yet?),
Full contradiction map (where do my documented positions conflict with each other
  or with captured sources?),
Knowledge gaps (what perspective is clearly absent from my vault?),
One action (highest-leverage thing to research or write this week).
Be specific. Reference note titles. This session should be uncomfortable.
```

Output: `00-Inbox/weekly-deep-[date].md`

### 8. Source Aging Alert — Mondays 8:00am
Scans 01-Sources for notes older than 60 days with no reaction paragraph (no "My take:" or "Reaction:" heading). Sends list to Telegram with: *"These source notes are over 60 days old with no reaction recorded. Process or delete this week."*

The discipline this enforces: nothing stays in 01-Sources indefinitely without becoming personal thinking.

### 9. Midnight GitHub Backup — 12:00am
```bash
cd /path/to/vault && git add -A && \
git commit -m "vault-backup-$(date +%Y-%m-%d)" && \
git push origin main
```
Sends Telegram confirmation with note count. Full version history. Recoverable in minutes.

## The Five-Folder Vault Structure

```
00-Inbox/     ← raw captures, brief outputs, triage reports
01-Sources/   ← notes from external sources (books, articles, papers)
02-Ideas/     ← own positions and developing theses
03-Projects/  ← active project notes and thesis documentation
04-Claude/    ← Claude-generated synthesis and connection outputs
```

## Infrastructure Setup

**N8N hosting options:**

| Option | Cost | Constraint | Best for |
|--------|------|-----------|---------|
| Self-hosted VPS (DigitalOcean, Hetzner, Linode) | $5–7/month | Setup ~1 hour | This use case — unlimited executions, full data control |
| N8N Cloud Starter | €24/month | Execution count ceiling | Simpler setup, fewer workflows |

Self-hosted is recommended: unlimited executions, no ceiling risk, data never leaves your infrastructure.

## Real Cost Breakdown

Using Haiku 4.5 ($1/$5 per million tokens) for light automations, Sonnet 4.6 for heavy synthesis:

| Automation | Daily cost |
|-----------|-----------|
| Daily synthesis brief | ~$0.024 |
| Inbox processor | ~$0.008 |
| Thesis contradiction check | ~$0.028 |
| Domain morning brief | ~$0.008 |
| Readwise resurfacer | ~$0.005 |
| Weekly deep session | ~$0.088/week |

**Total Claude API: ~$2.50–4.00/month on Haiku. $8–12/month with Sonnet for heavy workflows.**
VPS: $5–7/month. Everything else free.

**Total stack: $8–15/month.**

## Build Order

Build these three first:
1. **Telegram capture pipeline** — simplest, highest immediate impact, no API costs, creates the raw material everything else depends on
2. **Midnight GitHub backup** — protects everything before you start filling the vault with automation outputs
3. **Daily synthesis brief** — the first time it lands before you open your laptop, you'll build the rest that same weekend

Then: inbox processor → Readwise resurfacer → domain morning brief → thesis contradiction check → weekly deep session → source aging alert (last, needs vault volume to be meaningful).

## Questions & Gaps
- The synthesis brief reads notes "modified in the last 7 days" — does this include automation outputs (briefs, triage reports) or only original captures? Including automation outputs would create a feedback loop.
- The inbox processor produces DELETE recommendations — what's the actual deletion discipline? If everything stays, the DELETE category serves no function.
- The thesis contradiction check runs at 7am but only checks 01-Sources from the last 30 days. What happens to contradictions in sources older than 30 days?
- How does the source aging alert handle notes that legitimately don't need a reaction paragraph (reference notes, templates, indexes)?
- The Readwise resurfacer sends output to Telegram rather than the vault — does this mean those connections are ephemeral unless manually captured?

## Related Notes
- [Obsidian Second Brain Masterclass](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/obsidian-second-brain-masterclass.md) — the masterclass covers vault architecture, CLAUDE.md, and manual Claude prompts. This note is the automation layer on top of that architecture: what runs while you sleep vs. what you trigger manually. The two together form the complete system.
- [Claude Code + NotebookLM + Obsidian Pipeline](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/claude-code-notebooklm-obsidian-research-pipeline.md) — that pipeline brings research into the vault. This note is what happens to that research once it's there. Complementary layers.
- [Loop Engineering — Addy Osmani](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/loop-engineering-addy-osmani.md) — the N8N automations here are loop engineering applied to knowledge management: scheduled automations, state file (the vault), sub-agents (different Claude prompts for different jobs), and a verifier (thesis contradiction check). The architecture is identical.
- [Automated Blog Pipeline](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/automated-blog-pipeline-openclaw-5-phases.md) — both are production multi-step AI pipelines with specialized agents per task, state tracking, and daily/nightly cron triggers. The blog pipeline produces external content; this system produces internal intelligence.
