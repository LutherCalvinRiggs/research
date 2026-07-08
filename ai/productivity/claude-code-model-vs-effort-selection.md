# Model and Effort in Claude Code: Knowing More vs. Trying Harder

**Source:** https://x.com/ClaudeDevs/status/2063992615721435154 (pasted content by @ClaudeDevs, Anthropic)
**Saved:** 2026-06-27
**Tags:** ai, productivity, prompting, tools, fundamentals

> **Primary source.** Written by the Claude Code team (Anthropic). Official explanation of how model selection and effort levels work mechanically, and how to decide between them.

---

## TL;DR
Model selection and effort level both "make the answer better" but through completely different mechanisms. Model = which frozen set of weights handles the request (what Claude knows). Effort = how many tokens Claude generates before considering the task done (how hard it tries). When Claude gets something wrong: check context first, then ask "did it not know enough?" (model problem) or "did it not try hard enough?" (effort problem).

## The Core Distinction

| Setting | Controls | Mechanism | Analogy |
|---------|---------|-----------|---------|
| **Model** | Which frozen weights handle the request | Swaps the knowledge base permanently encoded at training time | Which person you hired |
| **Effort** | How many tokens get generated before Claude considers the task done | Sent as part of the request; model trained to behave differently at each level | How much time that person spends |

**The key insight:** effort means more than "thinking time." Effort controls how many files Claude reads, how much it verifies, and how far it pushes through a multi-step task before checking in. At higher effort, Claude takes more actions (reads files, runs tests, double-checks) before returning. At lower effort, it prefers to ask for more context rather than spending tokens figuring it out.

---

## How Model Selection Works (Mechanically)

1. Claude Code assembles: your message + system prompt + tool definitions + CLAUDE.md + conversation history + files in context → single API request
2. On the server: text is tokenized (each piece mapped to an integer from a fixed vocabulary — `const` → 1978, `await` → 4293)
3. The model predicts one token at a time by running input through frozen weight matrices → outputs probability distribution over all vocabulary tokens
4. A 200-token response = 200 separate passes through the weights

**The weights are read-only during inference.** Nothing in your prompt, CLAUDE.md, or context changes them. Everything Claude "knows" was encoded at training time.

**What changing the model does:** swaps which set of frozen weights handles the request — and also changes the per-token cost.

**What changing the model doesn't do:** doesn't change how many tokens get generated. That's effort.

**On hallucination:** when Claude confidently calls an API that doesn't exist, that's the weights producing a plausible-looking token sequence from training patterns — not a failed lookup. The API wasn't in the weights.

**On context and docs:** putting library docs in context works as steering (influences that one request) but doesn't add to the weights. Claude uses them for that request; the underlying model retains nothing.

---

## How Effort Works (Mechanically)

All of Claude's output falls into the same token loop, billed at the same rate:
- **Thinking tokens**: reasoning that streams before/between actions
- **Tool calls**: structured blocks naming a tool and its arguments
- **Text to you**: plans, progress updates, summaries

Effort level is sent as part of the request, alongside the prompt. The model was trained to understand how to behave at each effort level — that learned behavior is baked into the weights. Higher effort = higher confidence threshold before considering the task done = more tokens to reach that confidence.

**The same prompt at different effort levels can produce ~7× different token counts.**

**Plans update mid-run:** at higher effort, Claude creates a more detailed plan, but the plan isn't frozen. When step 1 of a three-hypothesis debugging plan finds the bug, Claude will typically say so explicitly and skip hypotheses 2 and 3. Higher effort makes double-checking more likely, but Claude generally won't artificially inflate usage on simple tasks. "Overthinking" is something the Claude Code team specifically watches for during model training.

---

## The Decision Framework

### When Claude gets something wrong — check context first

Before touching model or effort: is the prompt too vague? Does Claude have the right tools? Does it have the right skills? If you're increasing effort on a task that shouldn't need it, the fix is usually upstream — in context, CLAUDE.md, or how the task is scoped.

### Then ask the right question

```
Did Claude not try hard enough?    →  Effort problem
Did Claude not know enough?        →  Model problem
```

**Model problem signals:**
- Claude is confidently wrong even with good context
- Subtle bugs, unfamiliar domains, architecture decisions
- Larger models are better at handling ambiguity — smaller models do better with specific, direct instructions

**Effort problem signals:**
- Claude skipped a file, didn't run tests, didn't double-check its work
- Most relevant if you had selected an effort level below the model's default

---

## The Specialist / Expert / Generalist Mental Model

| Model | Role | At low effort | At high effort |
|-------|------|--------------|----------------|
| **Fable** | Specialist | Quick read but spots what no one else would — that recognition is what you're paying for | Thorough specialist on the hardest problems |
| **Opus 4.8** | Expert | 5 minutes with someone who has deep experience — brings patterns and gotchas from solving many similar problems | Expert with the whole afternoon |
| **Sonnet** | Generalist | Fast and accurate on routine work | Reads everything, runs things, double-checks — thoroughly understands your specific code |

None is universally better. Model ≈ how capable. Effort ≈ how thorough. Most real tasks need some of both.

---

## Model × Effort × Token Cost Interaction

**On routine work (same effort level):**
- Both large and small models generally get it right
- Larger model consumes more tokens with extra verification steps, at higher per-token price
- → Use smaller model for routine stretches; saves real money at no quality cost

**On hard multi-step work:**
- Smaller model grinds toward its capability limit, burning iterations
- Larger model reaches the same quality bar in fewer steps
- Total cost per task on the larger model can be *lower* — and more importantly, the larger model can finish tasks the smaller one can't at any effort level

**Fable specifically:** on long multi-step work, Fable pulls furthest ahead — finishes jobs Opus and Sonnet can't reach at any effort level. Also costs the most per token. Save it for work that genuinely needs it.

**On effort and token limits:**
- `max_tokens` is a hard ceiling that truncates mid-stream — blunt instrument for API developers
- Task budgets or "keep it brief" in the prompt are softer controls the model is trained to follow
- Effort shapes token consumption but doesn't limit it; the model wraps up as it approaches a stated limit rather than hitting a wall

---

## Practical Notes

**For most tasks:** use the model's default effort level. The default is calibrated to match what most people want to spend on a task.

**Effort as a general preference, not per-task:** reach for effort level deliberately when you have a consistent preference for thoroughness or speed based on domain and work type. Don't switch it task-by-task.

**Opus 4.8 note (from the team):** default effort on Opus 4.8 produces better results for about the same tokens as default effort on Opus 4.7 on the same task.

---

## Questions & Gaps
- The effort setting is described as "sent as part of the request" — is this a system prompt instruction, a parameter in the API call, or something else? The exact mechanism isn't shown.
- "The model was trained to understand how to behave at each effort level" — does this mean effort is simply a prompt instruction ("behave at effort level 3") or is it an actual API parameter with distinct model behavior per value?
- The 7× token count difference between effort levels — is this consistent across task types, or highly variable? Is there a published mapping of effort levels to expected token multipliers?
- The curves in the article are explicitly labeled "for illustration purposes only" and "do not represent real benchmark data." Is there published data on the actual cost-per-task crossover point between models on harder tasks?

## Related Notes
- [Benchmarks Measure a Model You Aren't Running](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/benchmarks-measure-a-model-you-arent-running.md) — this note explains *why* benchmarks don't tell you what your model will do (deployed context, prompt format, etc.). The "weights are frozen at training, context only steers" explanation here clarifies the mechanism behind that observation.
- [Stopped Hitting Claude Usage Limits](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/stopped-hitting-claude-usage-limits-10-things.md) — session-level token optimization. This note provides the mechanical foundation for why those habits work: reducing context = fewer tokens steering the model per request.
- [Trim Claude Code System Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/claude-code-trim-system-prompt-token-optimization.md) — reduces the input token cost. This note explains the output token cost (driven by effort). Together they cover both sides of the per-request token bill.
- [Claude Fable 5: Self-Improving Agent System](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/claude-fable-5-self-improving-agent-system.md) — the model routing recommendation there (Fable 5 for orchestration, Sonnet for workers, Haiku for graders) directly implements the specialist/expert/generalist framework from this note.
- [Unit Economics of Running Cattle](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/unit-economics-of-cattle.md) — Jed's cost governance framework. The "larger model can be cheaper per completed task on hard work" insight here changes the unit economics calculation: it's not always cheapest to route hard tasks to smaller models.
