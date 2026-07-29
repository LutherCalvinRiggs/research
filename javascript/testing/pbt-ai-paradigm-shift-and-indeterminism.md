# Property-Based Testing in the Age of AI: Paradigm Shift & Defeating Indeterminism

**Source 1:** https://medium.com/@VectorWorksAcademy/property-based-testing-in-the-age-of-ai-coding-a-paradigm-shift-97b83051f8b8 (VectorWorks Academy, April 2026)
**Source 2:** https://sumeetmore.medium.com/can-we-defeat-indeterminism-in-ai-agents-with-property-based-testing-94ac015fa3dc (Sumeet More, March 2026)
**Saved:** 2026-07-23
**Tags:** ai, testing, fundamentals, agentic-ai, javascript, prompting

---

## TL;DR
Two complementary framings of why PBT matters now more than ever: VectorWorks frames it as a paradigm shift from example-driven to specification-driven engineering (AI generates code, humans define invariants, machines explore possibilities). Sumeet More frames it as the answer to AI agent indeterminism — not eliminating non-determinism, but constraining it within behavioral boundaries using semantic generators and property checkers. Together: PBT is the verification layer that scales with AI-generated code volume, and for AI agents specifically, semantic generators (not random strings) are required to explore meaningful failure modes.

---

## The Paradigm Shift (VectorWorks)

### The Old World → The New World

| Before | After |
|--------|-------|
| Developers write code AND test cases | AI generates code; humans define correctness properties |
| Example-driven: "does this work for these cases?" | Specification-driven: "what must always be true?" |
| Human imagination limits test coverage | Machines explore vast input spaces automatically |
| Tests encode implementation bias | Properties express invariants independent of implementation |

**The key reframe:**
> Instead of asking "does this work for these cases?", ask "what must always be true, no matter the case?"

### Why AI Accelerates PBT Adoption

**AI as Generator, PBT as Verifier.** AI systems are exceptionally good at generating plausible implementations. They do not inherently guarantee correctness across all scenarios. PBT acts as a scalable verification layer — you define properties, the framework searches aggressively for counterexamples.

> If AI can generate many possible implementations, testing must scale to evaluate them just as broadly.

**Alignment with generative thinking.** AI models operate over distributions rather than fixed inputs. PBT evaluates behavior across a distribution of inputs. Both operate in a generalization-first paradigm — they're philosophically aligned in a way that example-based testing is not.

**Reducing the human bottleneck.** In the AI era, the bottleneck shifts from writing code to defining correctness. Humans are better at defining intent; machines are better at exploring possibilities. PBT operationalizes this division of labor.

### The Five Property Patterns (VectorWorks)

| Pattern | Description | Example |
|---------|-------------|---------|
| **Idempotence** | Applying twice = applying once | `normalize(normalize(x)) == normalize(x)` |
| **Roundtrip** | Encode → Decode = original | `decode(encode(x)) == x` |
| **Monotonicity** | Increasing input shouldn't decrease output | `bid(higher) >= bid(lower)` |
| **Invariant** | Universal rule that always holds | Sort output length == input length |
| **No exception** | Weakest — function doesn't crash | Valid input never throws |

### The Strength Hierarchy (Trail of Bits formalization)
Weakest → Strongest: **No Exception → Type Preservation → Invariant → Idempotence → Roundtrip**

Always push for the strongest applicable property. "No crash" is not a test — it's the floor.

### Domain Use Cases

**Data pipelines / ETL:** Transformations must not unintentionally drop or duplicate records. PBT validates these guarantees across many input variations.

**APIs and distributed systems:** Failures often arise from unexpected sequences of events. PBT can simulate retries, partial failures, serialization round-trips — ensuring invariants hold under varied conditions.

**Ads/optimization systems:** Many guarantees are monotonic or conservation-based. Increasing a bid should not reduce expected exposure; total spend should remain within budget. PBT validates these across simulated scenarios.

**ML systems:** PBT helps test the "shape" of model behavior — small input perturbations shouldn't cause disproportionately large output changes; ranking functions should behave consistently under scaling.

### What Goes Wrong

**Weak or vague properties** — "output looks reasonable" provides no assurance. Properties must be both general and precise.

**Poor generators** — generators that don't reflect real-world inputs miss critical scenarios. Generator quality is as important as property quality.

**Not a replacement** — PBT complements, not replaces, unit tests, integration tests, and E2E tests.

---

## Defeating Indeterminism in AI Agents (Sumeet More)

### The Problem: AI Agents Aren't Deterministic

Traditional testing assumes: same input → same output. AI agents violate this. A tax assistant supposed to answer only tax questions might occasionally respond to a weather query. These failures emerge from the probabilistic nature of LLMs — not obvious code bugs.

Example-based testing provides false confidence: correct for your specific prompts, fragile to anything outside them. Language is effectively infinite.

### The Answer: Constrain, Don't Eliminate

> Property-based testing does not eliminate the indeterminism inherent in AI systems. Language models will continue to produce varied responses, and that variability is often what makes them useful. Instead, PBT provides a way to **constrain this indeterminism within well-defined behavioral boundaries.**

The goal shifts from "predict exact outputs" to "verify consistent behavior within defined rules."

### The Four PBT Components for AI Agents

**1. The Property** — a rule that must always hold:
```
If a query is unrelated to taxation, the agent must respond with "not supported."
```

**2. The Semantic Generator** — the critical innovation. Standard PBT generators (random strings, regex patterns) produce noise, not meaningful inputs. For AI agents, you need a generator that operates at the level of **meaning, not syntax**:

```fsharp
type Intent =
    | TaxQuestion of string
    | NonTaxQuestion of string

let semanticGenerator () =
    let nonTaxExamples = [ "What's the weather today?"; "Who won the match?"; "Recommend a movie" ]
    let taxExamples = [ "How do I file income tax?"; "What deductions are allowed?" ]
    if Random.bool() then TaxQuestion (Random.pick taxExamples)
    else NonTaxQuestion (Random.pick nonTaxExamples)
```

Three approaches to semantic generation:
- **Structured intent generator** — domain model of possible user intents (deterministic, reproducible)
- **Embedding-based exploration** — semantic perturbations around known examples (mathematical)
- **LLM-powered generator** — another LLM generates diverse, realistic prompts (realistic but adds more non-determinism)

**3. The Test Runner** — executes the agent with generated inputs, collects responses (hundreds or thousands of runs)

**4. The Property Checker** — verifies the defined property holds after each response:
```fsharp
let property (intent: Intent) =
    let response = runAgent intent
    match intent with
    | NonTaxQuestion _ -> response = "not supported"
    | TaxQuestion _ -> response <> "not supported"
```

### The Key Shift: Syntactic → Semantic Generation

This is the innovation that makes PBT applicable to AI systems. Without semantic generators, you run thousands of noise queries that never explore the meaningful failure space. With semantic generators, you systematically probe the behavioral boundaries where real failures occur.

For an auction agent: instead of random strings, generate semantically meaningful queries — valid bids, invalid bids, cross-lot requests, post-deadline bids, bids from banned users — and define what behavioral rules must hold for each class.

---

## Practical Synthesis: PBT for Your Agent Workflows

**For code generated by agents (fast-check approach):**
- Write property tests against the code the agent produces
- Generator: fast-check arbitraries (typed, domain-specific)
- Properties: invariants, roundtrip, idempotence, monotonicity

**For behavior of AI agents themselves (Sumeet More approach):**
- Write property tests against the agent's output
- Generator: semantic (domain-modeled intents, LLM-generated prompts, embedding perturbations)
- Properties: scope constraints ("agent never answers out-of-domain"), safety properties ("agent never reveals system prompt"), consistency properties ("agent response to same-intent queries is categorically consistent")

---

## Questions & Gaps
- The semantic generator approach requires a domain model of intents. How do you build that model systematically for complex domains (e.g., a full auction platform with dozens of agent capabilities)?
- LLM-powered generators introduce additional non-determinism into the testing process. How do you prevent the generator itself from being the source of test flakiness?
- For auction agents where behavior should be deterministic (same bid amount, same lot, same user → same decision), is the semantic PBT approach necessary, or is standard PBT sufficient?
- The property checker must classify agent outputs into pass/fail. For nuanced natural language responses, how do you write a checker that's reliable enough to be trusted?

## Related Notes
- [fast-check: Property-Based Testing for JavaScript/TypeScript](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/testing/fast-check-property-based-testing.md) — the fast-check implementation for testing agent-generated code. VectorWorks' paradigm shift framing explains why fast-check matters now more than before.
- [Anthropic Agentic PBT: Finding Bugs Across the Python Ecosystem](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/anthropic-agentic-pbt-python-ecosystem.md) — Anthropic's implementation of Claude-as-PBT-agent. Sumeet More's semantic generator concept maps to Anthropic's "understand the input domain by looking at calling functions" step.
- [Uncle Bob: Extreme Constraints for AI Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/uncle-bob-extreme-constraints-ai-agents.md) — PBT is the constraint Uncle Bob names. This note explains the "why now" for both the code and the agent.
- [Kiro: Correctness with Property-Based Testing](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/testing/kiro-correctness-property-based-testing.md) — Kiro's built-in PBT uses EARS requirements as the property source. VectorWorks' "specification-driven engineering" framing is exactly what Kiro is implementing.
