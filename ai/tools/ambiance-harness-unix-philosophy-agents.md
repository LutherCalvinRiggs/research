# Towards a Harness That Can Do Anything — Ambiance

**Source:** https://eardatasci.github.io/c/ambiance/index.html (Arda Tasci) — via https://news.ycombinator.com/item?id=48921077
**Saved:** 2026-07-15
**Tags:** ai, tools, infrastructure, orchestration, agentic-ai, fundamentals

> Project: Ambiance (github.com/whitematterlabs/ambiance) — available at whitematterlabs.ai or via curl install. Early stage / work in progress.

---

## TL;DR
A harness design essay arguing that the Unix/Linux environment is the natural substrate for agentic AI — because LLMs already know it from training data. The core insight: the model's priors are the cheapest resource you have. A harness built from things the model already knows (files, users, logs, docs, FHS directory structure) will always beat one it has to be taught. Ambiance implements this as a filesystem-watching event bus ("Kernel") that invokes LLM instances on file change rather than polling on a fixed heartbeat.

---

## The Four Properties of a Good Harness

1. **Naturally intuitive to the Agent** — built from primitives the model already understands
2. **Fully transparent** — everything auditable by agent or human, self-healing possible, post-mortem possible
3. **As lean as possible** — minimal cognitive load (measured in tokens); the core prompt should be tiny; the agent loads skills at runtime
4. **Resilient** — error survival, update survival, no memory corruption or degradation over time

> "With the advancing intelligence of LLMs, harnesses will eventually be *reliable*. The real matter at hand is reducing how much cognitive load (measured in tokens) you are putting on your bot."

---

## The Three Preliminary Truths

1. **Determinism as much as possible.** The LLM should choose which goal to pursue, but the deliberation toward that goal should be well-defined (or a collection of well-defined steps). Non-determinism should be reserved for the judgment calls that require it.

2. **The core prompt should be as small as possible.** The LLM should choose which skills to load into context at runtime — not receive everything upfront. This is the deferred-tool-loading / L2-on-demand pattern from the vertical agent context hierarchy note.

3. **LLMs start going crazy as you approach context limits.** Context management is not optional; it's the primary engineering constraint.

---

## Don't Play the Odds, Play the Bot

The central design principle:

> A good harness MUST make use of the LLM's a priori coding knowledge. Coding and systems administration are heavily overrepresented in LLM training data — give the agent an environment it's already comfortable in. Wrangling it through a novel environment ultimately wastes tokens.

Implications:
- Don't invent new formats when standard ones exist (JSON → flat text files; curl wrangling → pre-processed data; novel APIs → familiar Unix primitives)
- The harness should feel light to the LLM but do a lot in the background (logging, sanity checks, failsafes, sanitizations)
- Precious context should not be wasted on file discovery, traversal, or navigating unfamiliar structures

---

## The Unix Philosophy Mapped to Harness Design

**Unix creed (Ritchie & Thompson):**
1. Write programs that do one thing and do it well
2. Write programs to work together
3. Write programs to handle text streams — it's a universal interface

**Harness equivalent:**
1. Write modular, transparent tools that do one thing and do it well; ensure they fail loudly
2. Write tools, skills, and connectors that work together — skills dictate workflows, tools execute them, connectors are the data the agent manipulates
3. Text streams are the universal interface — LLMs have home-court advantage; everything should be a flat text file

The critique of current harnesses: they're overly complicated, attempting to do too many things, with ill-defined agent trajectories. Tools are directly loaded into context; system prompts are pre-filled with warnings and rules that fade turn-over-turn.

---

## Everything Is a File

The key design decision: pre-process external data before it reaches the LLM.

> "Have you ever manually done JSON wrangling? What about constructing heavy curl commands? Complicated regex? An LLM might have an easier time than you, but they prefer plaintext too. Whenever dealing with external data sources, your harness should perform whatever manipulation is necessary to clean it up before it reaches your LLM."

**The filesystem as the data layer:**
- Categorizing everything into directories saves wasted tokens on discovery
- LLMs are expert at navigating Linux FS — `grep`, `find`, `which`, `rg`, `fzf` are all native
- The harness turns the messy outside world into a clean, navigable faux-VFS

### The FHS Mapping

| Harness concept | Unix equivalent | FHS location |
|----------------|-----------------|--------------|
| Agents | Users | `/home/...` |
| External data | Drivers | `/sys/` |
| Tools | Binaries | `/bin/` |
| Logs | Logs | `/var/` |
| Self-healing tools | System binaries | `/sbin/` & `/recovery` |
| Skills | Documentation | `/usr/share/doc` |

---

## The "Kernel" — Event Bus vs. Heartbeat

**The industry standard (OpenClaw et al.):** event-driven message handling paired with a heartbeat — a full agent turn on a fixed interval (30 minutes by default) to check whether anything needs attention.

**The problem with heartbeat:** anything that isn't a pushed message (file changes, external state) only gets noticed at heartbeat granularity. Tighten the interval → burn a full LLM turn on every empty check. Loosen it → agent is up to an hour behind the world.

**Ambiance's solution:** an event bus that watches the filesystem for changes with cursors on text files, then invokes the LLM accordingly (with coalescence strategies for high throughput). The agent never misses a notification. Different "users" (LLM instances) can be wired to respond to different events.

The Kernel acts as the middle layer between the LLM and the outside world — checks that everything the LLM does is safe and not obviously harmful. Analogous to how a real OS kernel mediates between software and hardware.

---

## The Three Default Users (LLM Instances)

1. **`root`** — handles all system-level tasks: coding new drivers, writing new binaries, fixing existing ones
2. **`pai`** — the human-facing LLM that actually interacts with the outside world
3. **`librarian`** — journals what `pai` is good at, what it's bad at, and what the system did for the day (the self-improvement / Reflect strand equivalent)

All three communicate continuously via the event bus and a `send-message` binary.

---

## HN Discussion — Key Signal

The comment thread surfaced an important convergence point that the article doesn't make explicit:

**From `brainless`:** "I totally believe in a deterministic scaffold but I really think an agent should be as deterministic as possible — the more code, the better. What if we had an agent that had context of your codebase, deterministically ran test suite, linter, hooks, etc. The 'English' prompt would become a code loop with the LLM only brought in to decide if a test has failed because of feature change."

**From `vishvananda`:** "All of my success for long running tasks has been around wrapping the agentic harness in a deterministic workflow with deterministic gates. We need a name for this outer layer. In my mind that is the true harness because it constrains the agent's failure mode. I think 'flow engineering' has been proposed. Maybe it's the 'agentic exoskeleton'?"

**From `alexpotato`:** References the ACM article "Manual Work is a Bug" — the argument that any manual process should be automated into code. Applied here: you and the LLM look at what has to be done, build the scripts/tools to make it happen, then tie those tools into a system. "The more I use the above the more it makes sense and the worse the whole 'just commit the prompt' seems like nonsense."

**Emerging terminology:** "flow engineering" and "agentic exoskeleton" as candidate names for the deterministic outer layer that wraps and constrains the non-deterministic inner agent. Related to NEEDLE's state machine and kiro-config's orchestrator pattern.

---

## What This Adds to the Research Library

**The Unix/FHS framing is novel.** The insight that LLMs have home-court advantage in Unix environments because that's what dominated their training data is an underexplored design principle. Most harness discussions focus on context, tools, and memory — not on leveraging the model's existing environmental priors.

**The event bus vs. heartbeat distinction** is a concrete architectural choice with real tradeoffs that isn't documented elsewhere in the library. Heartbeat is simpler; event-driven is more responsive but requires coalescence logic.

**The `librarian` user** (journaling what the agent is good at, bad at, and what it did today) is a third implementation of the Reflect strand / session feedback pattern (alongside NEEDLE's Reflect strand and Jamon's item 8). Three independent convergences on the same pattern is strong validation.

---

## Questions & Gaps
- The FHS mapping is a design analogy — how strictly does Ambiance actually implement it? Does `/bin/` literally contain agent tools as executables, or is this metaphorical?
- The coalescence strategies for high-throughput file events — what are they? If 50 files change simultaneously (e.g., a large refactor), how does the event bus decide what to invoke and when?
- The Kernel "checks that everything the LLM does is safe" — what does this mean technically? Is this a content filter, a permission model, or something else?
- macOS only currently (noted in HN thread). FUSE-based approach for Linux was considered but not implemented.
- "Manual Work is a Bug" (ACM, queue.acm.org/detail.cfm?id=3197520) — referenced as a relevant prior art piece. Worth saving separately.

## Related Notes
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — Anthropic's harness post describes the macro multi-agent architecture. This essay goes deeper on the substrate philosophy (Unix environment as natural harness) and the event-vs-heartbeat distinction.
- [Building a Good Vertical Agent — Context Hierarchy](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/building-good-vertical-agent-context-hierarchy.md) — the L1/L2/L3 context hierarchy (always-resident, on-demand, escape hatch) maps directly onto this essay's "core prompt small, skills loaded at runtime" principle. Same insight, different framing.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the HN commenter's "agentic exoskeleton" concept and the "determinism as much as possible" principle are exactly what NEEDLE's exhaustive outcome table implements.
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — NEEDLE is the production implementation of the "deterministic outer layer wrapping non-deterministic agent" pattern. The HN thread naming exercise ("flow engineering," "agentic exoskeleton") is searching for what NEEDLE already is.
- [Jamon Holmgren 18-Point Agentic Setup](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/jamon-holmgren-18-point-agentic-setup.md) — Jamon's bin/tools folder (item 9) and AGENTS.md router (item 0) are the same instincts as this essay's `/bin/` tools and filesystem-as-data-layer. Independent convergence on Unix primitives as the natural agentic substrate.
