# The Workflow Manual: Building Applications with Agent Fleets

**Source:** https://jedarden.com/guides/workflow/
**Author:** Jed Arden (jedarden.com)
**Published:** July 30, 2026 · 73 min read
**Saved:** 2026-07-31
**Tags:** ai, tools, orchestration, agentic-ai, infrastructure, fundamentals

> **Canonical primary source.** This is the most complete single document Jed Arden has published. All prior NEEDLE notes reference parts of this workflow; this is the full manual in one place. Last verified: July 30, 2026.

---

## TL;DR
Seven-stage workflow for building applications with headless agent fleets. Not a chat session — workers pulling from a queue of small, well-specified tasks while you do something else. The core insight: **decomposition is concurrency control** — how you cut the work is how you cut the concurrency. Nine stages, 54 conditions, each stage producing the artifact the next one consumes. Stages are load-bearing in sequence. The path was partly adopted (two stages from Jeffrey Emanuel) and partly built here (six stages from production experience).

---

## The One Idea to Carry

> "How you cut the work is how you cut the concurrency."

Everything else in the manual is technique. This is structure. The `Owns:` line on every task — the files it may write — determines how many tasks can run simultaneously. Two tasks with disjoint `Owns` sets run concurrently forever, safely. Two tasks whose `Owns` sets intersect are serialised by a dependency edge or they destroy each other's work silently.

This is why decomposition is stage 6, not stage 2. You cannot know the right concurrency plan until the plan is reviewed and clean.

---

## What a Worker Actually Receives

On every dispatch, a worker gets exactly three things:
1. **The task specification** — context, design, acceptance criteria, owned files
2. **The repository** at the current working tree state
3. **The standing instructions** — the agent-instructions file at the repo root

No conversation history. No memory of the session that wrote the plan. No access to the reasoning that produced the architecture. If a decision is not in one of those three places, it does not exist for the worker — and the worker will make it again, differently.

> "Everything upstream of dispatch is really one activity: getting decisions out of your head and into one of those three channels."

---

## The Seven Stages

```
1. Scaffold   → README + docs/ tree
2. Research   → research notes in docs/research/
3. Plan       → docs/plan/plan.md (one file)
4. Review     → zero undeliverable tasks (a number, not an impression)
5. Decompose  → tasks with Owns: lines + dependency graph
6. Run        → fleet with deterministic outcome table
7. Refine     → feed outcomes back to plan → return to stage 3
```

Stage 7 feeds stage 3, not stage 1. Scaffold happens once. Plan → Review → Decompose → Run → Refine is the cycle you stay in for the life of the project.

**The diagnostic table** — symptom points to the stage that was skipped:

| Symptom | Restart at |
|---------|-----------|
| Agents inventing file layouts; `utils/` and `lib/` for same thing | Scaffold |
| Plan of confident assertions, none marked as assumptions | Research |
| Workers reasoning from different subsets; irreconcilable commits | One plan file |
| Clean, well-formed, confidently wrong work | Review |
| Timeouts, or agents overwriting each other | Decomposition |
| Can see what fleet did but not why it was allowed | Outcome handling |
| Queue describing a system nobody is building any more | Refinement |

The fourth row is the one to internalise. Every other skip produces visible mess. Skipping review produces work that looks entirely fine until you read it closely.

---

## Stage 02: Repo Scaffold

```
<repo>/
├── README.md              ← purpose in two paragraphs (orientation, not feature list)
├── AGENTS.md / CLAUDE.md  ← standing rules: build, test, commit, never-rules, blocked protocol
└── docs/
    ├── notes/             ← curated, human-shaped, topical, durable (you maintain)
    ├── research/          ← one note per source or question (fills at stage 03)
    └── plan/
        └── plan.md        ← one file: the complete plan (written at 04, maintained forever)
```

Plus a second notes directory at root:
```
notes/                     ← agent work logs, one per task (written by fleet, not by you, never curate)
```

The two `notes/` directories are not a mistake. One is curated institutional knowledge; the other is an execution trail. The moment they mix, neither can be trusted.

**Critical on AGENTS.md:** Rules must be stated as absolutes, not preferences. "Prefer X" is read by an agent as permission to do Y with justification. "Never Y — doing so is a failed task" is read as a constraint. The compliance difference is dramatic and entirely about phrasing.

**What belongs in AGENTS.md:**
- Exact build and test commands (copy-pasteable, not described)
- Commit conventions including required trailers
- Never-rules stated as prohibitions with consequences
- Where the tracker lives and how to claim/comment/close
- **What to do when blocked: stop, release, comment** — without this, blocked agents improvise, and improvisation is where scope creep comes from

**Gate 02:** Four directories exist, README states purpose in two paragraphs, AGENTS.md has exact build/test commands, commit convention, never-rules as prohibitions, blocked protocol. Tracker initialised.

---

## Stage 03: Research

One markdown file per source or per question in `docs/research/`. The discipline: answer "what do I not know yet?" in writing before the plan commits to answers.

**The template for most projects (seven questions):**
1. Prior art — what already exists and what it gets right
2. The thing you are compatible with — its data model and contract
3. Storage or format constraints — what the persisted shape must survive
4. Behaviour at scale
5. **The classic problem in this domain** — the failure mode practitioners already know (thundering herd, ABA problem, clock skew). Highest-value note.
6. Over-engineering check — what happens if you build something simpler
7. What must not break — existing callers, scripts, data

**Four research note types:**
- **Behavioural** — what a dependency actually does vs. what docs claim. Done when you have run it and pasted the output.
- **Specification** — the parts of a format/protocol this project touches. Done when every field you will implement is described.
- **Prior art** — what already exists and why you're not using it. Done when you can state the specific reason, not a vibe.
- **Measurement** — a number you took yourself. Done when the number and the command that produced it are both in the file.

Research feeds acceptance criteria: the behavioural note that says "this API returns 429 with Retry-After under these conditions, here is the curl" IS the acceptance criterion for the retry task, already written.

**The spike rule:** if research cannot be completed by reading, write a research note that says so, and create a task in the plan to find out — timeboxed, with a stated question and deliverable. A spike is not an assumption wearing better clothes.

**Gate 03:** Every decision the plan will make is backed by a research note or listed as an open question (no third category). Each note states what it means for the design. Anything unanswerable is a timeboxed spike. At least one note has a command and its actual output.

---

## Stage 04: The One Plan File

`docs/plan/plan.md` — one file, the complete plan. Not a directory of design documents. Not a wiki. **One file.**

Why: workers load whichever document the task referenced. Split across five documents → different agents reason from different subsets → nobody can tell which is authoritative when two disagree.

**The seven-part skeleton:**

1. **Overview** — what this is and what "done" means. Two paragraphs.
2. **Hard constraints & invariants** — the agent contract. Stated as absolutes. "Never Y — doing so is a failed task." Includes: never-rules, pipeline facts agents cannot discover (build image has no Python; deploy takes 10 minutes), safety rules, worker protocol (claim/close/blocked), and the instruction for when instructions conflict ("stop and surface it, do not resolve it yourself").
3. **Current state** — verified facts with a date. Agents cannot see your machine; anything they would otherwise guess belongs here.
4. **Architecture decisions** — a table of decision + rationale. Removes per-task judgement calls. Without D1 written down, every task re-litigates it; different workers reach different answers on different days.
5. **Phases** — numbered, dependency-ordered, each task carrying files and acceptance criteria.
6. **Open questions** — genuinely deferred items only. If Phase 1 secretly depends on one of these, the plan is not ready.
7. **Verification playbook** — the actual commands with expected outputs, run after every deploy.

**The highest-value single line in any plan:** the instruction for what to do when instructions conflict. "Stop and surface it" converts a silent wrong turn into a visible question.

**Sub-plans (when one file isn't enough):** A scoped sub-plan that names its parent, states which constraints it inherits, and says which parent item it supersedes. Parent carries one line pointing at it. Single entry point maintained.

**Gate 04:** plan.md exists and is the only plan document, has all seven parts, every constraint is phrased as an absolute, current-state has a verification date, every phase depends only on things earlier phases produce, every task names the files it will touch.

---

## Stage 05: Review (The Highest-Leverage Stage)

> "A defect in a plan paragraph does not stay one defect. It becomes five wrong tasks, each dispatched to an agent that executes it faithfully, each producing commits you will later have to unwind."

Work queues multiply plan quality in both directions. This is the stage almost nobody documents and the one with the highest return.

**Three passes, in order:**
1. **Structural review** — checklist-driven: unstated assumptions? data models defined before phases depend on them? phases depending on open questions? acceptance criteria verifiable by command?
2. **Adversarial gap hunt** — separate pass whose only job is finding what is MISSING. Different mindset, finds different things.
3. **Fix and re-run** — edit the plan, run both passes again. Repeat until clean.

The third step is the one people skip. A review that finds eleven defects and is never re-run has told you the plan had eleven defects, not that it now has zero.

**Typical revision count:** Draft (~10 defects) → Pass 1 (~few residual defects) → Pass 2 (clean).

**The brief that works:**
```
You have not seen this plan before. Do not evaluate whether the ideas are good.

For each task in §5, answer one question: could an agent execute this as written,
with only this repository for context?

If no, say precisely what is missing — a file path, a data model, a decision, a command.
Do not propose alternative designs.

Output: a numbered list of undeliverable tasks with the specific gap in each.
```

**Exit condition:** "Zero undeliverable tasks" — a number, not an impression.

**What the review is looking for (in order of frequency):**
1. Tasks not deliverable as written (most common by wide margin — agent assumes context it won't have)
2. Phases depending on undecided things
3. Acceptance criteria that are opinions ("works correctly") not commands
4. Missing data models
5. Dependency edges wrong in both directions
6. Constraints stated as preferences

**Gate 05:** Full structural review returned nothing new. Separate gap-hunt returned nothing new. Undeliverable task count is zero (as a number). Open questions contain nothing any phase depends on. Every acceptance criterion is a command. Reviewer was not the author.

---

## Stage 06: Decomposition = Concurrency Control

> "Every task declares the files it owns. Tasks with disjoint file sets run concurrently; tasks that overlap are serialised by a dependency edge. Cutting the work is therefore the same act as planning the concurrency."

**The genesis task:** One root tracking issue, titled after the project, holding a reference to the plan and a checklist of phases. Every phase's tasks block it; it closes when the project ships. Fixed point to check progress against when the queue has 90 items.

**Three decomposition patterns:**
- **Up-front batch** — create all tasks now. For clean plans with well-specified phases.
- **Phased hierarchy** — fully decompose Phase 1; later phases get placeholders expanded when reached. The default.
- **Just-in-time** — loop creates 2–5 tasks per iteration from the gap between plan and reality. For metric-driven projects.

**Sizing thresholds:**
- Keep together: single file or module, under ~3,000 characters, fewer than 8 acceptance criteria
- Split: exceeds ~4,000 characters or 10 criteria
- Split: mixes concerns (CLI AND execution AND recovery)
- Split: sequential phases ("first X, then Y, then Z" = 3 tasks)
- Canonical three-part component split: setup/scaffolding → core implementation → edge cases/cleanup

**Task specification shape:**
```
## Context      — why this exists; link the plan section
## Design       — approach, files to touch, constraints
## Acceptance Criteria — each one verifiable by a command
## Owns:        — the files it may write (EVERYTHING it writes must be listed here)
## Notes        — gotchas, references
```

**The file partition (load-bearing):**
```
Owns: src/api/upload.ts, src/api/upload.test.ts
```
Writing outside the list = failed task (agent stops and comments, does not proceed).

**The canonical example — same feature, three cuts:**

By layer (instinctive, wrong): API endpoint + Storage adapter + Upload form UI all claim `api/router.ts`. Peak concurrency: 1.

By file surface (correct): One route-registration task → other three depend on it, each owns distinct files. Peak concurrency: 3. Extra task bought the parallelism.

One big task: No overlap, no parallelism. As fast as one worker, which is the hidden cost.

**The silent failure mode:** Two agents editing one file is NOT a merge conflict. One agent commits; the other runs `git checkout` or `git reset --hard` as part of its normal work and discards the first agent's uncommitted changes. No error. The losing task reports success.

**Pre-launch partition check:**
```bash
<tracker> ready --json \
  | jq -r '.[] | .id as $i | .owns[] | "\(.) \($i)"' \
  | sort | awk '{f[$1]=f[$1]" "$2; c[$1]++} END{for(k in c) if(c[k]>1) print k":"f[k]}'
```
Anything printed = missing dependency edge or task that should be split.

**Dependency edges:** Only add an edge if the later task genuinely cannot start first. Check for cycles — circular dependencies silently starve the fleet (queue shows open tasks, all mutually blocked, workers idle).

**Gate 06:** Genesis task exists. Every task under ~3,000 chars with fewer than 8 criteria. Every criterion is a command. Every task has Owns: line. No two concurrently-runnable tasks share a file (checked with command, not by eye). No cycles.

---

## Stage 07: The Queue

**The core requirement:** Claims must be atomic. A single database transaction: select the next eligible task and mark it claimed, atomically.

Read-then-write claiming does not work at any meaningful fleet size. At 20 workers, phantom claims are the normal state.

SQLite `BEGIN IMMEDIATE` is the fix. Second worker blocks for a millisecond, picks the next available task. No phantom claims, no retry storm, no server — the file is the lock.

**Agents own closure:** Make closure part of the agent's instructions. Implement, commit with task ID as trailer, push, validate against acceptance criteria, then close or release. The orchestrator verifies exactly one thing: **a commit exists since dispatch**. Cheap tripwire, catches the failure mode that matters (agent reports success while changing nothing).

**Commit trailer:**
```
feat(upload): add S3 storage adapter

Task: bf-4k2p
```

**Tripwire limits:** Catches agents that changed nothing. Does NOT catch wrong work, unrelated commits, partial implementations closed anyway. Quality assurance is the acceptance criteria being commands — not the tripwire.

**Queue data model:** Two representations — live SQLite (authoritative) + git-tracked JSONL checkpoint (snapshot). Repair rebuilds from checkpoint. **Flush first, always.** Repairing against a stale checkpoint destroys every task created since the last flush.

```bash
# Safe sequence
<tracker> sync --flush-only
sqlite3 .beads/beads.db "PRAGMA integrity_check;"
<tracker> doctor --repair   # only if step 2 complained
```

**Gate 07:** Two workers attempting the same task = exactly one claim (tested, not assumed). Claiming is a single database operation. Agents close their own tasks, orchestrator verifies commit. Commits carry task ID trailer. You know which representation is authoritative and have flushed at least once.

---

## Stage 08: Scaling in Two Dimensions

**Depth** = workers inside one repository. Bounded by (a) the partition's overlap ratio and (b) the host's memory headroom. Both ceilings apply; you hit whichever is lower.

**Width** = repositories in flight at once. No shared working tree, no depth failure modes. Bounded only by having enough different ready work.

**Memory reality:** Idle supervisor process = ~500 MB resident before any dispatch. Each agent subprocess = +230–400 MB. 40 workers = ~20 GB before a single agent starts. Worker count must be bounded by the host's capacity for fleet overhead, not by task count.

**Overlap ratio measurement:**
```bash
<tracker> ready --json | jq -r '.[].owns[]' | sort | uniq -c \
  | awk '{t++; if ($1>1) c++} END {printf "%d/%d files contended (%.0f%%)\n", c, t, 100*c/t}'
```
Under 5%: go as deep as host allows. Over 33%: depth won't help; re-cut the tasks.

**Decision procedure:**
1. Work in one repo + tasks partition cleanly (one file each)? → Go deep
2. Every task touches the same file? → Don't go deep; restructure or accept depth 1, go wide
3. Many repos with ready queues? → Go wide (scales further, more forgiving failure mode)

**Gate 08:** Overlap ratio measured (not estimated). Depth or width chosen with stated reason. Intended worker count below host ceiling. If deep: overlap low enough workers will be productive. If wide: every planned repo has ready tasks.

---

## Stage 09: Fleet Deployment

The orchestrator's job is deliberately boring. One outcome table, fixed:

| Outcome | Handling |
|---------|---------|
| Agent closed task, commit exists | Accept. Move on. |
| Agent closed task, no commit since dispatch | Reject close, reopen, flag. (The tripwire.) |
| Agent released with blocker comment | Leave blocked. Needs human or dependency. |
| Non-zero exit, no close | Release, increment failure count, retry if under threshold |
| Timeout | Release and defer. Usually task too big. |
| Failure count at threshold | Stop retrying. Escalate as a task. |

**Adapter contract:** The orchestrator doesn't know what an LLM is. It knows how to run a command.
```
cd {workspace} && <agent-cli> --model {model} < {prompt_file}
```
Substitutions: `{workspace}`, `{prompt_file}`, `{bead_id}`, `{model}`. That's it.

**Dispatch prompt contents (four parts):**
1. Standing instructions from repo root
2. Task specification verbatim
3. Protocol: implement → commit with ID → push → validate → close or release
4. **Stop conditions** (the part people leave out): what to do when blocked, when instructions conflict, when work requires touching a file the task doesn't own

**Four dispatch modes (least to most autonomous):**
1. **Interactive session** — one conversation, continuous oversight. Exploratory work.
2. **Supervised batch** — fan out a dozen tasks, review per batch.
3. **Autonomous fleet** — workers loop claim→execute→close for hours, attention per incident.
4. **Metric marathon** — persistent loop chasing one number. **Never run alongside fleet workers on the same repo** — they fight over working-tree state.

**Launch stagger:** 1–2 second stagger between worker launches. Removes thundering herd from claiming. Atomic claiming makes the herd correct (one winner); stagger makes it efficient (no contention).

**Workers exiting is normal.** A worker that has drained its workspace stops on idle timeout. Not a crash. Assuming a fleet stays at launch size is how you end up with two workers watching an empty queue.

**Gate 09:** Single task dispatches end to end. Every outcome has defined handling including repeated failure. Failure threshold exists and stops dispatch (not retries forever). Launches staggered. Routine tasks route to cheap tier. Full stop sequence run once with process listing verification.

---

## Stage 10: The Operator's Day

**Three daily questions, under two minutes:**
1. Are workers alive?
2. Is the queue draining?
3. Did anything escalate?

If all three: fine. Stop looking. The interesting question is which signals deserve interruption, not what to watch continuously.

**Three alert-worthy signals:**
1. Task failed more than twice — structural problem, will keep failing forever
2. Queue not draining while workers are alive — everything blocked, or workers in wrong directory
3. Spend rate departing from task-completion rate — signature of rework

**"Fleet is starved" is usually false.** Most common causes: worker querying from wrong directory (sees empty queue that isn't the queue), diagnostic output contaminating JSON parse (queue appears empty when full), dependency cycle (all tasks blocked). Check the ready queue yourself before believing any starvation alert.

**Session reading order when investigating:**
1. What context did it actually get? (Nine times in ten: agent behaved sensibly given wrong input)
2. Where did it start improvising? (The turn it stopped following the task)
3. What did it think "done" meant? (If it validated against anything other than the criteria, the criteria weren't commands)

All three diagnoses point upstream — at the task, not the agent.

**Weekly review (deserves real attention):**
1. Read escalations
2. Look at failure list (same underlying problem wearing different task IDs?)
3. Check plan against reality (stage 7 — keeps queue from describing a system nobody is building)
4. Check spend against completion (rising ratio = rework)

**Gate 10:** Three daily questions answerable in under two minutes. Escalations arrive as tasks. Can attach to a running worker and read what the agent actually saw. Weekly review is scheduled, output written somewhere the plan can absorb.

---

## Stage 11: Failure Modes

**Runaway retries:** One failing task can absorb hundreds of dispatches and hundreds of dollars in a day. Circuit-break explicitly: after ~3 failures, stop, split into smaller children, block original until they land. A task failing repeatedly is almost never transient — it's too large, underspecified, or blocked on something the plan doesn't mention.

**Recursive self-decomposition:** Task count grows exponentially. Auto-splitting without a termination condition that fires. Fix: (1) label generated children to exclude from re-evaluation, or track generation depth. Critical: **a safety check that can be satisfied by empty input is not a check.** Treat "no data" as "do not proceed."

**Silent orphans:** Worker dies mid-dispatch; task stays claimed. Fix: a reaper that checks process table (not elapsed time alone). Stopping a fleet incorrectly produces this at scale.

**Escalation belongs in the queue:** Not a side channel. A human-labelled task that blocks dependents. Queue is the escalation channel — exactly one place where project state lives.

**Don't let the agent grade its own homework unsupervised:** Agents close their own tasks (correct design). Tripwire required: commit exists since dispatch. Acceptance criteria that are commands are what make the rest work.

**Resource exhaustion looks like model failure:** OOM-killed worker reports nothing. Tell: several workers failing in the same window on unrelated tasks in unrelated repositories. Check memory headroom before blaming the agent.

**Self-modification:** Fleet tooling in a repo the fleet works on = bad change takes out every worker. Stage the rollout, keep the previous version recoverable, never let automated path replace the thing running the automated path without a gate. **Keep at least one recovery path that doesn't depend on the fleet.**

**Triage order:**
1. Is there ready work? (Run the query yourself, from the right directory — most alarms die here)
2. Are the workers alive? (Process table, not tool's status output)
3. Is one task failing repeatedly? (Check failure counts before reading logs)
4. Did anything commit? (Claims moving but tree not changing = agents reporting success without doing work)
5. Only now: read a session (expensive, requires judgment — this is why it's last)

---

## Stage 12: Lineage

**Adopted from Jeffrey Emanuel:**
- The single plan file (section 04) — one document, reloaded on every dispatch
- Gating the plan before decomposition (section 05) — review to a checkable exit condition before cutting tasks

These are load-bearing. Removing either takes a large piece of the method.

**Arrived at here (built in production):**
- Docs-first scaffold (02) — fixed tree, split between curated notes and agent execution trail
- Research before planning (03) — one note per question, calibrated to how quietly wrong assumptions fail
- Genesis task (06) — one root tracking issue as fixed point
- Task sizing rules (06) — character/criteria thresholds, three-part component split
- Agent-owned closure with commit tripwire (07)
- Design-time file partition (06) — `Owns:` lines as the concurrency plan
- Two-axis scaling (08) — depth bounded by partition and host, width bounded by ready queues

**Atomic claiming** — mechanism built here in response to a specific gap (client-side read-then-write sequence produced phantom claims at 20 workers).

**Key divergence from the adopted source:** Coordination at design time, not runtime. No inter-agent messaging channel — the file partition does that work before dispatch, enforced by dependency edges rather than runtime negotiation.

---

## Section 13: Steal This Without the Tooling

None of this requires NEEDLE, bead-forge, or a fleet. In order of return per unit of effort:

1. **Docs-first scaffold** — create `docs/research/`, `docs/plan/`, and AGENTS.md before code. One minute. Works in one session.
2. **Review the plan before decomposing** — fresh session, cold-read for deliverability, fix, re-run until clean. Exit: zero undeliverable tasks. Works in one session.
3. **Small tasks with command-verifiable acceptance criteria** — one concern, under ~3,000 chars, fewer than 8 criteria, every criterion a command. "The single biggest quality lever in the whole method, and it is entirely free."
4. **Declare the files each task owns** — even with one agent, writing `Owns:` sharpens the task and surfaces badly-scoped work early.
5. **Make the agent close its own work, verify a commit exists** — `git log` after the fact is the same check.
6. **Atomic claims (if you ever run more than one worker)** — not optional the moment there are two workers.

**Starting checklist:**
- [ ] Create docs/research/, docs/plan/, and instructions file before any code
- [ ] Write one research note per question you would otherwise guess at
- [ ] Write plan.md with hard-constraints section phrased as absolutes
- [ ] Have a fresh session cold-read for deliverability; fix; re-run until clean
- [ ] Cut tasks: one concern, under ~3,000 chars, fewer than 8 criteria
- [ ] Every criterion a command; every task an Owns: line
- [ ] Check concurrently-runnable tasks have disjoint Owns sets
- [ ] Run — verify a commit exists per completed task

The first four take an afternoon and account for most of the benefit.

---

## Questions & Gaps
- The worktree-per-worker improvement is named as "the intended direction" — is this already implemented in NEEDLE v0.2.7 or still pending?
- The metric marathon mode ("never run alongside fleet workers on the same repo") — what's the recommended pattern for running a benchmark loop alongside feature work in the same repo without conflict?
- Jeffrey Emanuel's original methodology (which sections 04 and 05 were adopted from) — where is it published? The lineage section credits it but doesn't link it.
- The overlap ratio formula uses `owned_count_per_file > 1` — how does this handle shared test utilities or config files that many tasks read (but don't write)?

## Related Notes
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — NEEDLE is the orchestrator described in stage 09. This manual is the operating guide for it.
- [NEEDLE Implementation Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Implementation-Guide.md) — the agent brief for setting up the full stack. Read this manual first (the why), then the implementation guide (the how).
- [bead-forge Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/bead-forge/bead-forge-repo-overview.md) — bead-forge is the tracker described in stage 07. The atomic claiming mechanism is its core contribution.
- [Jed Arden Constellation](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/jed-arden-constellation.md) — the six-tool fleet map. This manual describes how to use that fleet; the constellation shows what it's made of.
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — Jed's foundational framing. This manual is the full operational implementation of the cattle model described there.
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — the prior note capturing Jed's "plan as the negative" concept. Sections 04 and 05 of this manual are the full treatment of that idea.
- [kiro-config Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/kiro-config-repo-overview.md) — kiro-config is the interactive pet model that produces the plan.md this workflow consumes. Sections 03–05 of this manual are what happen before kiro-config's `/orchestrate` skill runs.
