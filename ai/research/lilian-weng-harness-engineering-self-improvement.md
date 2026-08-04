# Harness Engineering for Self-Improvement

**Source:** https://lilianweng.github.io/posts/2026-07-04-harness/
**Author:** Lilian Weng (Head of Safety Research, OpenAI) — Lil'Log
**Published:** July 4, 2026 · 31 min read
**Saved:** 2026-07-31
**Tags:** ai, research, infrastructure, orchestration, agentic-ai, fundamentals

> **Primary source — Lilian Weng.** The most comprehensive academic survey of harness engineering as of mid-2026. Synthesizes ~40 papers into a unified framework. The author's framing: the harness layer between raw model and real-world context is as important as the model's raw intelligence.

---

## TL;DR
Harness engineering is the emerging discipline of designing and automatically optimizing the system surrounding a base model — how it thinks and plans, calls tools, manages context, stores artifacts, and evaluates results. Modern harnesses go beyond prompt templates into runtime and software system design. The central thesis: recursive self-improvement (RSI) in the near term will happen through harness evolution, not weight rewriting, and code is the universal language for defining and optimizing harnesses. Seven open challenges remain before full RSI is achievable.

---

## What a Harness Is (The Formal Definition)

> "A **harness** is the system surrounding a base model that orchestrates execution and decides how the model thinks and plans, calls tools and acts, perceives and manages context, stores artifacts, and evaluates results."

Compared with early agent frameworks ("agent = LLM + memory + tools + planning + action"), harness engineering additionally includes:
- Workflow design (loop engineering)
- Evaluation
- Permission controls
- Persistent state management

The design should be deliberately simple and generic — with reference to existing software engineering practices to benefit from pretraining knowledge. Strong analogy to operating systems: a harness should encapsulate complicated logic while keeping the interface simple.

**The progression of what gets optimized:**
```
instruction prompts → structured context → workflow → harness code → optimizer code
```
As models become more intelligent, the optimization target moves toward more complex and generic mechanisms.

---

## Part 1: Harness Design Patterns

### Pattern 1: Workflow Automation

A goal-oriented loop: plan → execute → observe/test → improve → execute again *until* the goal is achieved. The workflow graph emphasizes the model analyzing its own trajectories and failure cases and iterating through an "agent runtime" rather than a static prompt template.

Karpathy's autoresearch repo (github.com/karpathy/autoresearch) is cited as a clean example.

### Pattern 2: File System as Persistent Memory

> "A harness should not carry the entire workflow and all logs in context; instead, it should keep durable state in files."

In long-horizon agentic rollouts, artifacts (experiment logs, code diffs, paper summaries, error traces, rollout trajectories) grow much longer than context windows. File system management is a foundation skill that naturally benefits from core model capability improvements.

This is the key design principle: **store durable state in files, not in context**. Directly validates Jed Arden's "everything is a file" and Ambiance's filesystem-as-data-layer approaches.

### Pattern 3: Sub-agent and Backend Jobs

A harness can spawn multiple subagents to execute in parallel and monitor backend jobs — for searching multiple hypotheses, running experiments concurrently, or delegating isolated subtasks without polluting the main context.

**Critical design choice:** make parallelism explicit and inspectable. If subagent outputs only live in transient chat context, they quickly become obsolete and hidden. If stored as files, logs, and status records, the model can recover after interruptions and reason over its own execution history.

### The Standard Coding Agent Tool Set

The core tool interface has stabilized across Claude Code, Codex, OpenCode, and Cursor:

| Group | Tools |
|-------|-------|
| File system | `glob`, `grep`, `ls`, `read`, `read_many`, `write`, `edit`, `multi_edit`, `apply_patch` |
| Shell | `bash`, `PowerShell` |
| IO | `lsp`, `git_status`, `git_diff`, `git_commit` |
| External | MCP tools, Skills |
| Web | `web_search`, `web_fetch`, browser tools |
| Backend | `CronCreate`, `CronDelete`, `CronList` |
| Agent delegation | `spawn_agent`, `resume_agent`, `wait_agent`, `list_agents`, `close_agent`, `interrupt_agent` |

### Harness Layer vs. Core Intelligence

Lilian Weng's prediction for near-term RSI:
1. Harness engineering evolves toward **meta-methodology** — improving the machinery for getting better answers, not just improving the answers themselves. The harness becomes an optimization target with fewer heuristic rules and more general mechanisms.
2. Mature harnesses enable auto-research for model self-improvement; smarter models prevent harnesses from overengineering.

Eventually, many harness improvements will be internalized into core model behavior — but the interface with external context and tools remains. Analogy: manual prompt tricks became less central as instruction tuning improved, but the need to specify goals, constraints, context, and evaluation did not disappear.

---

## Part 2: Harness Optimization

### 2a. Context Engineering

**Agentic Context Engineering (ACE)** — treats context as an evolving playbook rather than a lengthening prompt. Three components: Generator (produces task trajectories), Reflector (distills insights from trajectories), Curator (updates structured context with itemized entries). Key design: the Curator does not rewrite a full prompt blob — it outputs structured bullets (identifier, description) merged with deterministic logic to prevent context collapse.

**Meta Context Engineering (MCE)** — separates mechanism (how to manage context) from artifact content (what is in context). Runs skill evolution at the meta-optimization level and context optimization at the base level. A bi-level optimization: inner loop finds best context given a skill; outer loop finds the optimal skill. Skills stored as files in a dedicated directory including both static (`skill.md`) and dynamic components.

**Meta-Harness** — the optimization object is the *code* that determines what information should be stored, retrieved, and presented to the model. "Meta-" means it is a harness for optimizing harnesses. Uses a Pareto frontier of harness candidates. Execution history accessible via file system; proposed harness is a dictionary containing its own source code, scores, rollout trajectories, and state updates.

### 2b. Workflow Design

**AI Scientist** (Nature, 2026) — pipeline to propose research ideas, write code, run experiments, analyze results, write manuscript, perform peer review.

**ScientistOne** — verifiability as the central constraint. Every claim must trace to an evidence source and is audited by Chain-of-Evidence checks.

**Autodata** — agentic data scientist. Challenger proposes problems; weak solver and strong solver attempt them; verifier/judge evaluates. Goal: synthesize data at "just right" difficulty (strong solver succeeds, weak solver fails). Challenger prompt updated iteratively from feedback.

**ADAS (Automated Design of Agentic Systems)** — formulates agent design as an optimization problem. Meta-agent proposes new agentic workflows in code, evaluates candidates, and adds successful ones to an archive. Iterates.

**AFlow** — represents agentic workflow as a graph (nodes = LLM-invoking actions, edges = logical operations in code). Optimization via MCTS (Monte Carlo Tree Search). Showed improvement over manually designed workflows and ADAS.

### 2c. Self-Improving Harness

> "Code is the universal language for defining programs and systems. If an LLM can optimize the code that executes agents, it can access a much larger design space than hand-written prompts."

**STOP (Self-Taught Optimizer)** — recursively improves the improver function itself. The meta-utility is the average performance of an improver over downstream tasks. STOP discovered strategies like genetic algorithms, multi-armed prompt bandits, simulated annealing, beam/tree search. **Cautionary finding: STOP improved performance with GPT-4 but degraded with weaker models (GPT-3.5, Mixtral). The base model must be capable enough to improve the mechanism.**

**Key disentanglement (Lin et al. 2026):** Two distinct capabilities:
- *Harness-updating*: capability of producing useful harness edits — surprisingly flat across model sizes (Qwen3.5-9B to Claude Opus 4.6). A 9B model can write a skill procedurally isomorphic to Opus.
- *Harness-benefit*: capability of utilizing the updated harness — non-monotonic; **middle-tier models benefit the most**.

Implication: small models can evolve harnesses, but you need a capable model to benefit from the evolved harness.

**Self-Harness** — three-stage propose-evaluate-accept loop:
1. Weakness mining: cluster failures into verifier-grounded failure patterns with rich causal information
2. Harness proposal: bounded edits based on mined patterns, distinct and diverse candidates
3. Proposal validation: regression tests on held-in (weakness resolved?) and held-out (no new issues?) data; candidates accepted only if no regression on both

**Agentic Harness Engineering (AHE)** — three observability pillars:
1. *Component observability*: every editable harness component has a file system representation. Seven components: system prompt, tool description, tool implementation, middleware, skill, sub-agent configuration, long-term memory.
2. *Experience observability*: raw trajectories → per-task analysis reports → benchmark overview (layered access is token-efficient)
3. *Decision observability*: every edit paired with a prediction for the next round; edits are evidence-driven with a "manifesto entry" (failure evidence, inferred root cause, targeted fix, predicted impact including at-risk regressions)

**Critical constraint in AHE:** Edits only applied to the harness workspace. The runs directory, tracer, verifier, and LLM configuration are **read-only** — disabling reward hacking (disabling the verifier, swapping the model, raising the reasoning budget). Every recorded gain is attributable to harness edits.

AHE result: transfers to SWE-bench-verified without further evolution, indicating the evolved harness encodes general engineering experience, not benchmark-specific optimization.

### 2d. Evolutionary Search

**AlphaEvolve** — coding-agent evolutionary search. Stores pool of candidate programs, prompts frozen LLMs to generate diffs for improvement. Key details: code regions for improvement explicitly marked with `# EVOLVE-BLOCK-START` / `# EVOLVE-BLOCK-END`. Meta-prompt co-evolves with instructions.

**Darwin Gödel Machine (DGM)** — explicitly targets evolution of an editable harness-code repository. Agent allowed to modify its own harness. Pool-based evolutionary selection (proportional to performance, inversely proportional to number of children). Under `Claude 3.5 Sonnet`, discovered agents comparable to or outperforming handcrafted agents on SWE-bench Verified (20% → 50%) and Polyglot (14.2% → 30.7%).

**ShinkaEvolve** improvements: performance-rank + offspring-count balanced parent sampling, code-novelty rejection (cosine similarity check), meta-scratchpad for successful patterns.

### 2e. Joint Optimization with Model Weights

**SIA (Self Improving AI)** — combines harness improvement and model-parameter updates in same loop. Meta-Agent proposes initial harness; Task-Specific Agent executes; Feedback-Agent decides whether to update harness or weights based on trajectories. Direction promising but evidence provisional; training stability and Goodhart effects remain open.

---

## Part 3: Seven Future Challenges

**1. Weak and fuzzy evaluators.** Self-improvement works best for tasks with measurable, objective evaluation metrics. Research taste, novelty, and long-term scientific value are much harder to measure. Current loops are gated by evaluator quality.

**2. Context and memory lifecycle.** Memory grows as agents become more autonomous. Context engineering will and should become a core part of intelligence, not just a software system layer. Humans maintain lifetime memory — the analogy suggests this should be internalized.

**3. Negative results.** LLMs may be bad at deciding when to abandon a hypothesis or acknowledge failure — literature bias toward successes means training data has imbalanced success/failure cases. Harnesses should make failed attempts easy to preserve.

**4. Diversity collapse.** Evolutionary and RL loops exploit known high-reward patterns. Need mechanisms to prevent population from collapsing into variants of the same solution — critical for open-ended research where the best path may initially look worse under the current evaluator.

**5. Reward hacking.** A self-improvement loop optimizes whatever signal it's given. If reward comes from unit tests → overfit to tests. From a judge model → learn judge-specific tricks. From benchmarks → exploit benchmark artifacts. The evaluator and permission control should sit *outside* the loop that evolves the harness.

**6. Long-term success.** Coding agents increase daily productivity but many optimization goals are too short-term. Standard sandbox RLVR training rarely captures maintainability, ownership boundaries, migration cost, backwards compatibility, or future debugging burden.

**7. The role of humans.** Humans should move up the stack, not be removed from the loop. Oversight at the right time and the right abstraction level. "After all, we are building the technology for better future of humanity, not other way around."

---

## Six Failure Modes in Autonomous Research (Trehan & Chopra 2026)

Tested LLMs going from research idea to paper with minimal scaffolding. Only one of four ideas was fully executed into a paper. Six recurring failure modes:

1. **Bias toward training-data defaults** — use old libraries, stale commands, standard formats, assumptions not grounded in the actual repo
2. **Implementation drift under execution pressure** — when implementation becomes complex, model moves toward common simpler solution rather than the proposed method
3. **Memory and context degradation** — long-horizon projects lose critical details unless logs are written as persistent artifacts
4. **Over-optimism** — model declares success despite noisy or failed experiments ("p-hacking and eureka-ing"); introduces "numerical duct tape" and declares victory when signals are still noise
5. **Insufficient domain intelligence** — lacks tacit craft knowledge (predicting implementation complexity, judging whether results are plausible, knowing which baselines matter)
6. **Weak scientific taste** — experiments may be executable but fail to answer the right question

---

## What This Note Adds to the Research Library

**The meta-methodology framing** — harness engineering as "improving the machinery for getting better answers, not just improving the answers themselves" is the clearest statement of where NEEDLE and kiro-config sit in the RSI landscape. They are early harness engineering; what Weng describes is where that work goes.

**The disentanglement of harness-updating vs. harness-benefit** — small models (9B) can write harness improvements that are procedurally equivalent to those written by large models. But benefit from those improvements requires a capable base model. This means harness evolution can be delegated to cheap workers; exploitation requires capable workers.

**The observability-first design principle (AHE)** — three pillars (component, experience, decision observability) are the formalization of what Jed Arden's trace worksheets and FABRIC telemetry implement in practice. The "manifesto entry" pattern (failure evidence + root cause + targeted fix + predicted impact) is precisely what a good bead comment should contain.

**Read-only verifier** — the AHE constraint (verifier sits outside the editable surface) is the formalization of why NEEDLE's acceptance criteria are set at dispatch, not evaluated by the worker. The worker cannot grade its own homework.

---

## Questions & Gaps
- The harness-benefit non-monotonicity (middle-tier models benefit most) — does this mean there's a ceiling at high capability where models become less responsive to harness improvements? Why would larger models benefit less from evolved harnesses?
- The "diversity collapse" challenge is named but no solutions are proposed beyond citing exploration literature. For NEEDLE fleet design, what mechanisms prevent all workers from converging on the same approach to a problem?
- AHE achieves transfer to SWE-bench without further evolution — but how much of that transfer is from the harness versus the base model? Is there a clean ablation?
- The "over-optimism / p-hacking and eureka-ing" failure mode — does this manifest in coding agents specifically, or is it limited to research agents? NEEDLE workers declaring success without meeting acceptance criteria would be the coding-agent version of this.

## Related Notes
- [Anthropic Recursive Self-Improvement](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/anthropic-recursive-self-improvement.md) — Anthropic's primary source on RSI. Weng's post is the academic survey of the same space; both describe AI research at frontier labs accelerating through AI-assisted research.
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — Anthropic's practical harness design post. Weng's post is the research survey above that practical post.
- [Ambiance Harness — Unix Philosophy for Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/ambiance-harness-unix-philosophy-agents.md) — Pattern 2 (file system as persistent memory) and the OS analogy are both independently present in Ambiance. Weng validates the instinct: filesystem-native, persistent-state-in-files is the right harness substrate.
- [Jed Arden Workflow Manual](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/jed-arden-workflow-manual.md) — Jed's workflow is a practical implementation of Pattern 1 (workflow automation), Pattern 2 (file system as memory), and Pattern 3 (sub-agents). AHE's three observability pillars describe what `active-work.md`, FABRIC telemetry, and the trace worksheet implement.
- [NEEDLE Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/needle-repo-overview.md) — NEEDLE is an early production implementation of the self-improving harness vision. Darwin Gödel Machine (pool-based harness evolution) is where NEEDLE points, long-term.
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md) — concepts 11–20 (memory, subagents, loops, sandboxing, permissions, hooks, injection, gates, tracing, metrics) are the practical implementation layer of what Weng surveys at the research level.
