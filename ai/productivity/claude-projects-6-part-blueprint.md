# How to Actually Set Up Claude Projects — The 6-Part Blueprint

**Source:** https://x.com/0xcodez/status/2062127385923776831 (pasted content by @eng_khairallah1)
**Saved:** 2026-06-07
**Tags:** ai, productivity, prompting, tools, claude, learning

---

## TL;DR
Most people use Claude Projects as labeled chatboxes. High performers build them as customized AI employees using a 6-part blueprint: Identity Block, Rules Block, Process Block, Output Format Block, Knowledge Files, and an Onboarding Message. One-time 45-minute setup. Claimed 90% reduction in editing time per task.

## Key Concepts & Terms
- **Identity Block**: The opening of the system prompt. Gives Claude a specific role, a relationship framing ("you've worked with me for two years"), and a concrete audience. Creates confident, specific output instead of generic safe output.
- **Rules Block**: ALWAYS/NEVER constraints. The most important part of the system prompt. Every time you correct Claude's output manually, that correction should become a rule. Rules prevent Claude from defaulting to its generic patterns.
- **Process Block**: How to think, not what to produce. Defines the workflow Claude follows before and during output generation — plan first, outline, draft, review against rules, quality check.
- **Output Format Block**: The exact shape of the finished product. Title formula, word count range, section order, heading style, closing convention. Eliminates guessing and ensures proportional effort across sections.
- **Knowledge Files**: Persistent reference documents uploaded to the project — style guide, audience profile, competitor analysis, performance data, template library. These are what separate a project from a labeled chat.
- **Onboarding Message**: The first message in every new conversation within the project. Acts as a daily briefing — activates the knowledge files, sets context, forces strategic thinking before execution.

## The Three Failure Modes of Weak Projects
1. **Vague system prompt** — "You are a helpful writing assistant" wastes the system prompt. Claude already knows how to be helpful; it doesn't know your voice, your standards, or your fifteen pet hates.
2. **No knowledge files** — A project without files is a labeled conversation, not a project. Persistent reference documents are the entire point.
3. **Scope creep** — One project called "Work Stuff" that handles emails, content, strategy, and code produces mediocre results across all of them. Each project should do one thing extremely well.

## The 6-Part Blueprint

### Part 1: Identity Block
```
You are my [ROLE]. You have worked with me for two years.
You know my voice, my audience, and my standards.

Your job in this project is to [PRIMARY FUNCTION]
for [SPECIFIC AUDIENCE DESCRIPTION].

You are [COMMUNICATION STYLE].
You never [ANTI-PATTERNS].
```
The "two years" framing makes Claude default to confident, specific output rather than hedged generic output. The specific audience description ("250,000 followers learning AI tools for the first time") gives Claude a concrete person to write for.

### Part 2: Rules Block
```
ALWAYS:
- [Specific positive constraint]
- [Specific positive constraint]

NEVER:
- [Banned word or phrase]
- [Banned opening pattern]
- [Banned behavior]
```
Rules are constraints. Constraints force Claude into a specific lane. Build this list incrementally — every manual correction you make to Claude's output is a rule waiting to be written. After a month, the rules list is a precision instrument.

### Part 3: Process Block
```
PROCESS:
1. Think about what the reader already believes. Your opening must challenge it.
2. Outline the complete structure. Share for approval before writing.
3. Write the first draft in one pass without self-editing.
4. Review against the Rules. Fix violations.
5. Check every claim for a specific supporting number or example.
6. Read as a busy person scrolling on their phone. Cut anything that would lose them.
```
Without a process block Claude jumps to generating output. With one it follows a structured workflow. Step 6 ("read as a busy person scrolling") reportedly improved output quality more than any other single line.

### Part 4: Output Format Block
```
OUTPUT FORMAT:
- Title: [formula]
- Length: [word count range]
- Structure: [section order]
- Subheadings: [style spec]
- Closing: [exact convention]
```
When Claude knows the exact format before starting, it distributes effort correctly. Without this, it front-loads detail in early sections and rushes the ending.

### Part 5: Knowledge Files (Priority Order)

| File | Priority | Contents |
|------|----------|---------|
| Style guide | REQUIRED | 3+ examples of your best work. Claude pattern-matches against these to capture your voice. |
| Audience profile | REQUIRED | One-page persona — what they know, what frustrates them, what they're trying to achieve, what depth they want. |
| Competitor analysis | RECOMMENDED | Examples of content you want to be different from. Claude uses this to avoid generic takes. |
| Performance data | RECOMMENDED | Which past pieces performed best, what titles got engagement, what topics resonated. Claude uses this for strategic decisions. |
| Template library | RECOMMENDED | Best-performing formats as reusable structures. Claude uses these as starting frameworks instead of inventing from scratch. |

### Part 6: Onboarding Message
```
Today we are working on [TASK].

Here is the context: [WHAT I NEED AND WHY]

Reference my style guide for voice. Reference my audience profile
for targeting. Reference my performance data for strategic decisions.

Before starting, tell me:
1. What angle would make this stand out from what already exists?
2. What does the reader already believe that we can challenge?
3. What is the one-sentence hook?

Once I approve the angle, proceed with a full outline.
```
This message activates all knowledge files, sets context, and forces strategic thinking before execution. Without it Claude starts generating immediately. With it, Claude plans first.

## The 5 Projects Everyone Needs

| Project | System prompt defines | Key knowledge files |
|---------|----------------------|-------------------|
| Content Production | Voice, audience, content standards | Style guide, best pieces, audience persona, content calendar |
| Research & Analysis | Structure format, trusted sources, depth of analysis | Research methodology, past analyses, evaluation criteria |
| Communication | Email/messaging style, tone per audience type | Examples of great emails, templates for common types |
| Strategy & Planning | Business context, goals, constraints, decision framework | Business plan, competitor data, market research |
| Code & Technical | Tech stack, coding conventions, architecture preferences | CLAUDE.md, architecture decision records, code examples |

## The Before/After

**Before:** Type a detailed prompt → decent response → 20 minutes editing to match voice and standards → repeat 5–10 times a day.

**After:** Type a two-sentence task → output already matches voice, follows rules, targets audience, hits quality bar → 2 minutes reviewing → repeat 5–10 times a day.

At 2 hours/day usage with 60% editing time saved: ~1 hour/day → 5 hours/week → 250 hours/year → 6 full work weeks.

## Questions & Gaps
- The author tested "every possible project configuration" and rebuilt the same project twelve times — what were the intermediate configurations that didn't work? The failures would be as instructive as the blueprint.
- Knowledge file size limits: Claude Projects has a context window limit for uploaded files. At what file size does the style guide or performance data become counterproductive (too much to attend to)?
- How does this blueprint interact with Dynamic Workflows? The Process Block describes a manual multi-step workflow — could that be replaced or augmented with a Dynamic Workflow that orchestrates the steps automatically?
- The "two years" relationship framing — is this durable across model updates, or does it need to be refreshed when a new model version is deployed to the project?
- No mention of project maintenance: when do you update the rules, prune the knowledge files, or revise the identity block as your work evolves?

## Related Notes
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — Jed Arden's fleet-scale version of the same discipline: a written specification (bead body / plan doc) substitutes for accumulated context. The Claude Projects blueprint is the single-user version of this principle.
- ['No' is Not an Instruction](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/no-is-not-an-instruction.md) — the Rules Block is the written form of the same discipline: every correction you'd type becomes a permanent constraint. Both posts argue the same thing from different angles.
- [I Stopped Hitting Claude's Usage Limits](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/stopped-hitting-claude-usage-limits-10-things.md) — knowledge files as persistent context complement the token-saving habits from that note; Projects offload repeated context to files rather than re-pasting it every session.
- [Claude Code Dynamic Workflows](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-code-dynamic-workflows-6-patterns-14-steps.md) — the Process Block here is a manual sequential workflow; Dynamic Workflows automates the same orchestration pattern in Claude Code. The two systems are solving the same problem at different scales.
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — Claude Projects is the optimized pet model: one carefully configured session with persistent context. NEEDLE is the cattle model: stateless workers, no persistent session, spec in the bead body.
