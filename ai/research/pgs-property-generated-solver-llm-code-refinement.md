# Property-Generated Solver (PGS): Property-Oriented Feedback for LLM Code Refinement

**Source:** https://arxiv.org/html/2506.18315v2 (ICML 2026 submission)
**Saved:** 2026-07-23
**Tags:** ai, research, testing, agentic-ai, fundamentals, tools

> Research paper submitted to ICML 2026. Introduces PGS (Property-Generated Solver), a two-agent framework (Generator + Tester) that achieves 1.4–1.6× higher bug fix rate than state-of-the-art TDD methods. Key empirical finding: property-oriented, structurally minimal feedback dramatically outperforms standard I/O feedback for LLM code refinement.

---

## TL;DR
PGS proves that the quality of feedback given to an LLM during code refinement matters far more than the quantity of tests. The key insight: effective feedback must be both (1) property-oriented (verifying semantic invariants rather than input/output pairs) and (2) structurally minimal (using the simplest counterexample that exposes the bug). A complex I/O failure creates cognitive load that leads to repair failure. A short, property-violating counterexample with a clear assertion error enables successful repair. PGS achieves 74.5% pass rate vs. the TDD baseline's 64.4%.

---

## The Two Principles of Effective Feedback

**Principle 1: Property-Oriented.** Standard I/O feedback gives the LLM a specific failing input and output, which shares the same logical biases as the buggy code. The LLM patches the code to pass that specific case rather than fixing the root cause. Property-oriented feedback expresses a semantic invariant ("the product of prime numbers is 2! = 4") that is logically simpler to verify than to generate. This breaks the self-deception cycle.

**Principle 2: Structurally Minimal.** The simplest counterexample that violates the property. Not the raw failing input, but the shrunk minimal case. Complex counterexamples create cognitive load; minimal counterexamples isolate the error's root cause.

**The empirical proof (Figure 1 from the paper):**

| Feedback type | Repair result |
|--------------|---------------|
| Complex I/O counterexample (7,782 tokens of thinking) | ❌ Repair failed |
| Property-oriented + structurally minimal (327 tokens) | ✅ Repair successful |

Same bug. Different feedback. 24× fewer tokens thinking. Successful repair.

---

## The Asymmetry of Verification

A core theoretical insight: **verifying a solution is logically simpler than generating it.**

Empirical confirmation (Table 2, DeepSeek-R1-32B):

| Difficulty | Generation accuracy | Verification accuracy | Verification with filtering |
|-----------|--------------------|-----------------------|-----------------------------|
| Easy (N=32) | 90.6% | 93.8% | 96.9% |
| Medium (N=34) | 67.6% | 91.2% | 94.1% |
| Hard (N=34) | 32.4% | 76.5% | 88.2% |
| **Overall** | **63.0%** | **87.0%** | **93.0%** |

The model is dramatically better at verifying correctness than generating correct solutions — especially on hard problems. This is why a Tester agent (verification) can reliably guide a Generator agent (synthesis) even when the Tester uses a smaller model.

**Key implication:** Using a weaker model as the Tester is fine. Qwen2.5 as Tester still significantly boosts the more powerful DeepSeek-R1-32B as Generator. Verification is cheaper to compute than generation.

---

## The PGS Framework: Five-Step Loop

```
Generator                    Tester
    |                            |
    | 0. Generate initial code   | 1. Define properties from problem desc
    |                            |    (invariants, roundtrip, metamorphic)
    |                            |
    | ←─── "Start Iteration" ──→ | 2. Instantiate properties
    |                            |    (generate diverse input pool)
    |                            |
    | 3. Execute code against    |
    |    test cases (Executer)   |
    |    → Pass / Wrong Answer / |
    |      RuntimeError / TLE    |
    |                            |
    | 4. Feedback Formulation    |
    |    Rank counterexamples    |
    |    by MINIMALITY           |
    |    (token count as proxy)  |
    |                            |
    | 5. Code Refinement         |
    |    Generator uses targeted |
    |    feedback to repair      |
    |                            |
    [repeat steps 1-5]
```

**The Tester's property definition step** reads the problem description and identifies high-level invariants — not just I/O relationships, but semantic properties: "the product of prime factors must equal N," "the output must be non-negative," "adjacent elements must satisfy ordering constraints."

---

## The Minimality Finding

**Token count as the optimal complexity proxy** (Table 1):

| Feedback selection strategy | Avg. improvement | Avg. tokens |
|----------------------------|-----------------|-------------|
| Line Coverage Max | +5.3% | 5.12k |
| Runtime Min | +5.6% | 4.76k |
| **Token Count Min** | **+6.0%** | **4.72k** |

The minimization strategy (smallest token count) consistently outperforms median and max strategies across all benchmarks. Simpler feedback is not just cheaper — it's more effective. The minimal counterexample provides the clearest signal for where the bug is.

**Why:** Complex counterexamples require the model to filter out which parts of the execution trace are relevant. Minimal counterexamples eliminate that filtering — every token in the feedback is load-bearing.

---

## What PGS Does to Bug Visibility (Figure 6)

The mechanism in three stages:

| Stage | Wrong Answer % | Pass % | AssertionError % |
|-------|--------------|--------|-----------------|
| Initial (baseline) | 25.3% | 64.4% | — |
| After property injection | 12.6% | 64.4% | 10.3% → 10.5% |
| Final (PGS) | 6.4% | **74.5%** | — |

Stage 1→2: Injecting properties converts vague "Wrong Answer" failures into explicit "AssertionError" failures. The bug was always there — now it has a name and a location.

Stage 2→3: The Generator uses the AssertionError signals to debug and fix, resolving 74.5% of problems.

**The insight:** PBT doesn't fix bugs — it makes hidden bugs visible. Once visible, the Generator can fix them.

---

## Benchmark Results (Table 3)

PGS vs. all baselines across 6 models, 5 datasets:

| Dataset | Baseline | PGS (best) | Improvement |
|---------|---------|-----------|------------|
| HumanEval | 76.2–97.2% | 89.0–99.1% | Consistent gain |
| MBPP | 56.8–93.5% | 67.6–96.5% | Consistent gain |
| LiveCodeBench | 26.7–63.1% | 34.1–75.5% | Largest gains on hard problems |
| CodeContest | 12.5–46.8% | 20.2–60.2% | Consistent gain |
| SWE-Bench | 9.8–65.5% | 11.9–70.2% | Real-world repo tasks |

**On LiveCodeBench (hardest benchmark):** PGS with DeepSeek-R1-32B achieves 12.1% improvement over baseline — the largest gap. Simple self-debugging fails on deep logical errors; PGS does not.

**Cross-model robustness (Table 4):** Asymmetric configurations (small Tester, large Generator) still show consistent gains, validating that verification is cheaper than generation.

---

## Practical Implications for Your Workflow

**For NEEDLE workers / Claude Code agents:**

When an agent's code fails a test, the quality of the feedback sent back to the agent matters enormously. Standard test output ("AssertionError: expected 5 but got 4") is I/O feedback. PGS-style feedback would be: the property being violated ("the product of prime factors must equal N"), the minimal counterexample that triggers it, and the exact assertion that fails.

**The immediate takeaway:** when writing test feedback that agents will read (in acceptance criteria, in debug prompts, in CLAUDE.md), structure it as property violations with minimal examples — not as raw test output.

```javascript
// Standard feedback (I/O):
// "Test failed: input=[10, 7], output=9973, expected=70"

// PGS-style feedback (property + minimal example):
// "Property violated: product of prime factors must equal N
//  Minimal failing input: N=4, factors=[2,2]
//  Expected: 2*2=4, Got: function only collects 2 once"
```

---

## Questions & Gaps
- PGS uses token count as the minimality proxy — how does this interact with different tokenizers? A "minimal" example by one model's tokenizer might not be minimal by another's.
- The Tester defines properties from the problem description — for a real codebase without a formal problem description (just code), how does the Tester identify properties? This is the gap the Anthropic agentic PBT paper addresses (reading docstrings, function names, calling code).
- PGS is evaluated on competitive programming problems (clean specs, known correct outputs). How well does it transfer to messy enterprise code with implicit business rules?

## Related Notes
- [Anthropic Agentic PBT](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/anthropic-agentic-pbt-python-ecosystem.md) — Anthropic's PBT agent reads docstrings and calling code to derive properties (the real-codebase version of PGS's problem-description reading). The self-reflection loop in Anthropic's agent parallels PGS's Tester-to-Generator feedback loop.
- [fast-check: Property-Based Testing for JavaScript/TypeScript](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/testing/fast-check-property-based-testing.md) — fast-check's built-in shrinking produces the "structurally minimal counterexample" that PGS shows is the most effective feedback signal. The shrinking output IS PGS-style feedback.
- [Don't Let the Agent Grade its Own Homework](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/dont-let-the-agent-grade-its-own-homework.md) — PGS empirically validates this at scale. The asymmetry of verification (87% vs. 63% accuracy) is why the Tester agent must be separate from the Generator agent.
