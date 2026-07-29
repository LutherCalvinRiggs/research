# Agentic Property-Based Testing — Anthropic Frontier Red Team

**Source:** https://www.anthropic.com/research/property-based-testing
**Paper:** arxiv.org/pdf/2510.09907 (NeurIPS 2025 Deep Learning for Code Workshop)
**Saved:** 2026-07-23
**Tags:** ai, research, testing, agentic-ai, tools, fundamentals

> **Primary source — Anthropic.** Written by Anthropic's Frontier Red Team. The agent is implemented as a Claude Code custom command (Markdown prompt file). Python/Hypothesis implementation; the approach is portable to other languages and agents.

---

## TL;DR
Anthropic built a Claude Code agent that autonomously writes property-based tests for existing Python code, discovered hundreds of bugs across 100+ popular PyPI packages including NumPy, SciPy, and Pandas, and got patches merged into NumPy and AWS Lambda Powertools. The agent follows a six-step process: analyze target → understand it → propose properties → write Hypothesis tests → execute → self-reflect. The self-reflection loop (noticing and removing try-catch blocks that swallow failures) was critical to reducing false alarms. Of the top-scoring bug reports, 86% were valid and 81% were worth filing.

---

## The Agent's Six-Step Process

Built as a Claude Code custom command (Markdown prompt passed to Claude Code):

```
1. Analyze the target
   → Determine if it's a module, function, or file

2. Understand the target
   → Read function signatures, docstrings, source code, existing tests
   → Look at functions that call the target (understand the input domain)

3. Propose properties
   → Identify invariants, round-trip properties, metamorphic properties
   → Ground them in documentation and actual usage patterns

4. Write tests
   → Translate properties into Hypothesis PBTs
   → Use to-do list to track multi-step progress

5. Execute and triage
   → Run with pytest
   → Classify: bug, expected behavior, false alarm

6. Self-reflect
   → Check for test anti-patterns (try-catch swallowing errors, etc.)
   → Revise and re-run if needed
```

**The self-reflection loop is critical.** In one documented case: the agent wrote a property, it passed, then during self-reflection noticed it had wrapped the whole test in a try-catch block. After removing that, the test failed, exposing a real bug. <cite index="73-1">Notable improvement in self-reflection was observed with Opus 4.1 and Sonnet 4.5 compared to Sonnet 4.</cite>

---

## Real Bugs Found and Fixed

### NumPy — `numpy.random.wald` returns negative numbers
The Wald distribution should only produce positive numbers. <cite index="73-1">Claude knew this as a property of the Wald distribution and wrote a straightforward PBT to check if all samples are positive.</cite> The bug traced to catastrophic cancellation in the numerical calculation. Anthropic's fix achieved nearly ten orders of magnitude lower relative error.

**Patch merged:** https://github.com/numpy/numpy/pull/29609

### AWS Lambda Powertools — `slice_dictionary()` returns first chunk repeatedly
The bug: not incrementing the iterator. Caught by the property: slicing then reconstructing should return the original dictionary (roundtrip invariant).

**Patch merged:** https://github.com/aws-powertools/powertools-lambda-python/pull/7246

### CloudFormation CLI — `item_hash()` returns the same hash for all lists
The bug: using in-place `.sort()` which returns `None`, so all lists hash to `hash(None)`. Caught by the property: different inputs should produce different hashes.

**Patch submitted:** https://github.com/aws-cloudformation/cloudformation-cli/pull/1106

### HuggingFace Tokenizers — missing closing parenthesis in CSS color output
Caught by the property: output should match the regex for a valid HSL color code.

**Patch merged:** https://github.com/huggingface/tokenizers/pull/1853

---

## Evaluation Results

Tested against 100+ popular PyPI packages (NumPy, SciPy, Pandas, Requests, and many more).

| Evaluation stage | Valid bug rate |
|-----------------|---------------|
| Raw bug reports (984 total) | 56% valid of 50 manually reviewed |
| After LLM scoring rubric (top-ranked) | 86% valid |
| Worth filing with maintainer | 81% |

The ranking step using Opus 4.1 against a 15-point rubric was "considerably effective" — top-scored reports were significantly more likely to be real, reportable bugs.

---

## Three Property Types That Found the Most Bugs

<cite index="80-1">The agent is given examples of high-value properties: invariants, round-trip, metamorphic.</cite>

**Invariants:** Properties that must always hold.
```python
# Wald distribution always returns positive numbers
@given(st.floats(min_value=0.001, max_value=1000))
def test_wald_positive(mean):
    samples = np.random.wald(mean, mean, size=100)
    assert all(s > 0 for s in samples)
```

**Round-trip:** Encode → decode (or split → reconstruct) should return the original.
```python
# Slicing and reconstructing a dict should return the original
@given(st.dictionaries(st.text(), st.integers()))
def test_slice_roundtrip(d):
    chunks = slice_dictionary(d, chunk_size=3)
    reconstructed = {k: v for chunk in chunks for k, v in chunk.items()}
    assert reconstructed == d
```

**Metamorphic:** If you change the input in a specific way, the output should change in a predictable way.
```python
# Two different inputs should produce different hashes
@given(st.lists(st.integers()), st.lists(st.integers()))
def test_distinct_inputs_distinct_hashes(a, b):
    assume(sorted(a) != sorted(b))  # genuinely different lists
    assert item_hash(a) != item_hash(b)
```

---

## Applying This to Your Own Codebase (with Claude + fast-check)

The Anthropic agent uses Python/Hypothesis. The same approach applies in JavaScript/TypeScript with fast-check:

**Step 1: Point Claude at a function and ask it to propose properties**

```
You are a property-based testing expert. Analyze this function:
[paste function + docstring]

Read the function signature, implementation, and how it's called.
Identify:
1. Invariants that should always hold for all valid inputs
2. Round-trip properties (if applicable)
3. Metamorphic relationships between inputs and outputs
4. Edge cases the implementation might handle incorrectly

Output a list of 3-5 properties before writing any tests.
```

**Step 2: Write the fast-check tests**

```
Translate these properties into fast-check property-based tests.
Use fc.property() with appropriate arbitraries.
Do NOT wrap the assertion in try-catch.
After writing, check: would these tests actually fail if the implementation had the bug?
```

**Step 3: Self-reflect (the critical step)**

```
Review each test you just wrote. Check for:
- try-catch blocks that swallow failures
- Assertions that always pass regardless of the implementation
- Properties so broad they don't actually constrain behavior
- Missing edge cases in the arbitraries (empty arrays, null values, boundary numbers)

Revise any tests that have these issues before running.
```

---

## The Key Insight for Kiro Integration

<cite index="80-1">The agent is implemented as a natural language prompt stored in a Markdown file. As it is a prompt, the agent is easily portable to other agent implementations.</cite>

This means Anthropic's PBT agent is essentially a **Kiro skill** or **CLAUDE.md instruction** — not a separate tool. To replicate it in your workflow:

1. Create a `property-testing` skill file (SKILL.md) with the six-step process
2. Invoke it on any new function/module after implementation
3. Run the generated tests as part of the CI gate

The skill can be as simple as:

```markdown
# Property-Based Testing Skill

When asked to write property-based tests for a function:
1. Read the function signature, docstring, and source code
2. Look at other functions that call this one (understand real usage)
3. Propose 3-5 properties before writing any code
4. Write fast-check tests for each property
5. Review for try-catch blocks or trivially-passing assertions
6. Run the tests and report results
```

---

## Limitations (Important)

<cite index="73-1">Deriving properties from code with subtle or complex semantics remains difficult. If the code makes an implicit assumption, only the library maintainers can decide what the correct property to test is.</cite>

The python-dateutil case illustrates this: the agent flagged `easter()` returning a non-Sunday date as a bug, but this was intentional behavior due to differing calendar system semantics. The agent can't know about domain conventions that aren't documented.

---

## Questions & Gaps
- The Anthropic agent uses Hypothesis (Python). A JavaScript equivalent using fast-check would need different arbitrary patterns. Has anyone published a systematic comparison of Hypothesis vs. fast-check for the same test problems?
- The self-reflection step improved bug validity significantly — what specific prompting patterns trigger the best self-reflection? The paper mentions the prompt is in Appendix A of the arXiv paper.
- The evaluation used 100+ packages but focused on Python. For TypeScript/Node.js codebases, are the same property types (invariants, roundtrip, metamorphic) equally productive?

## Related Notes
- [fast-check: Property-Based Testing for JavaScript/TypeScript](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/testing/fast-check-property-based-testing.md) — the fast-check equivalent of Hypothesis. Implement this agent's approach using fast-check instead.
- [Uncle Bob: Extreme Constraints for AI Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/uncle-bob-extreme-constraints-ai-agents.md) — property-based testing is one of the six constraint types Uncle Bob names. This paper shows what it looks like when Claude itself writes the PBT constraints.
- [Anthropic AI-Native SDLC Security](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/anthropic-ai-native-sdlc-security.md) — the CLAUDE.md closed loop pattern from that article applies here: when the PBT agent finds a bug class, update CLAUDE.md to prevent that class from being generated.
