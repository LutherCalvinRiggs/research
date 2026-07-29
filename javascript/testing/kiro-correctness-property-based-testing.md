# Kiro: Correctness with Property-Based Testing

**Source:** https://kiro.dev/docs/specs/correctness/
**Saved:** 2026-07-23
**Tags:** ai, tools, testing, agentic-ai, javascript, fundamentals

> **Primary source — Kiro official docs.** Kiro is AWS's spec-driven agentic IDE (Amazon Kiro). This page documents how Kiro integrates PBT into the spec → design → implementation workflow.

---

## TL;DR
Kiro integrates property-based testing directly into spec-driven development. It extracts properties from EARS-format requirements ("THE System SHALL..."), generates hundreds of random test cases automatically, runs them against implementations, and when a test fails, lets you chat with Kiro to understand whether the fix should be in the implementation, the test, or the requirement itself. The spec-to-test link is explicit and traceable — every property test links back to the requirement that generated it.

---

## The Core Argument

<cite index="74-1">Property-Based Testing is a step towards a fundamental shift in how we think about correctness with AI, moving from checking individual examples to validating universal properties across entire input spaces. Traditional unit tests only check specific examples, and whoever writes them — human or AI — is limited by their own biases. By automatically translating natural language specifications into executable properties and generating comprehensive test cases, Kiro creates a powerful feedback loop that helps both AI agents and human developers build more reliable software.</cite>

The key insight: whoever writes the tests (human or AI) brings their own blindspots. PBT breaks this by letting the framework generate cases neither the human nor the AI would have thought to test.

---

## What is a Property (in Kiro's Framing)

<cite index="74-1">A property is a universal statement about how your system should behave. Properties express the invariants and contracts that should always be true in your system, regardless of the specific data involved.</cite>

Kiro maps this to EARS (Easy Approach to Requirements Syntax) requirements:

```
EARS requirement:
"For any authenticated user and any active listing, the user can view that listing."

→ PBT property:
For any user (authenticated) + any listing (active): viewListing(user, listing) succeeds
```

---

## The Car Sales Example (Kiro's Canonical Illustration)

| Approach | What gets tested |
|----------|-----------------|
| Traditional test | User adds Car #5 to favorites → Car #5 appears in their list |
| Kiro PBT | For ANY user + ANY car: when user adds car to favorites, it appears in their list |

<cite index="74-1">PBT automatically tests this with User A adding Car #1, User B adding Car #500, users with special characters in names, cars with various statuses, and hundreds more combinations — catching edge cases and verifying implementation matches intent.</cite>

The PBT probes for counterexamples through shrinking — almost like a red team trying to break the code.

---

## The Kiro PBT Workflow

```
Spec (EARS requirements)
         ↓
  Kiro extracts properties
  (which requirements can be logically tested?)
         ↓
  Properties shown with connection to original requirement + linked task
  (hover to see the link)
         ↓
  Developer runs PBT cases (optional by default — focus on implementation first)
         ↓
  Cases run against implementation
         ↓
  Failure → Kiro surfaces the specific failing scenario
         ↓
  Chat with Kiro: "Is this a bug in the implementation, the test, or the requirement?"
         ↓
  Fix the right layer
```

**Three possible responses to a PBT failure:**
1. Fix the implementation (the code has a bug)
2. Adjust the test (the property was over-constrained)
3. Refine the requirement (the spec was ambiguous)

This is the key advantage over pure unit testing: when a test fails, you don't know which of these is true. Kiro's chat interface helps you diagnose which layer needs to change.

---

## Kiro's PBT is Spec-Linked

Every property test in Kiro traces back to the specific requirement that generated it. This is the traceability advantage of spec-driven development applied to testing:

- Requirement changes automatically invalidate dependent property tests
- Property test failures point directly to the requirement they're verifying
- Audit trail: "why does this test exist?" → "because of this requirement"

This makes PBT maintainable in a way that ad-hoc property tests are not — they don't orphan when requirements change.

---

## Implementing PBT with Kiro (Practical Notes for Your Workflow)

Since you're using Kiro as your primary AI coding agent, here's how to activate and extend its PBT capabilities:

**1. Write EARS-format requirements in your spec files**

Kiro extracts properties from EARS syntax: `THE System SHALL [action] WHEN [condition]`

```
THE System SHALL accept a bid WHEN the bid amount exceeds the current highest bid
THE System SHALL reject a bid WHEN the lot status is not active
THE System SHALL prevent duplicate bids WHEN the same customer submits identical amounts within 100ms
```

Each of these is a PBT candidate — Kiro can generate the property from the natural language.

**2. Run PBT at task execution time**

PBTs are optional by default. In your `.kiro/settings` or workflow instructions, you can make them mandatory before a task closes — treating them as part of the acceptance criteria rather than optional verification.

**3. Use the failure chat to diagnose**

When a PBT fails, Kiro's chat lets you ask:
- "Is this a valid bug in my implementation?"
- "Is the test over-constraining the behavior?"
- "Did I write the requirement incorrectly?"

This is faster than manually reading test output and tracing through code.

---

## Kiro PBT vs. Claude Code + fast-check (Your Two Options)

Since you mentioned wanting to implement PBT with both Kiro and Claude separately:

| Dimension | Kiro PBT | Claude Code + fast-check |
|-----------|---------|--------------------------|
| Property extraction | Automatic from EARS specs | Manual prompt (or skill file) |
| Test framework | Kiro-managed | fast-check (you configure) |
| Spec traceability | Built-in | Manual |
| Failure diagnosis | Chat with Kiro | You read test output |
| Integration | Within Kiro IDE | Works anywhere (VS Code, CLI) |
| Language | Multi-language | JavaScript/TypeScript |
| Customization | Limited | Full (you write the arbitraries) |

**Recommended hybrid approach:**
- Use **Kiro PBT** for spec-driven properties (requirements → tests, traceability, automatic extraction)
- Use **Claude Code + fast-check skill** for function-level property discovery (the Anthropic agent approach — point it at a function and have it propose and write properties)

The two complement each other: Kiro tests "does this meet the spec?"; fast-check tests "what edge cases does this function have that we didn't spec?"

---

## The Indeterminism Point (from Sumeet More)

<cite index="75-1">Property-based testing does not eliminate the indeterminism inherent in AI systems. Language models will continue to produce varied responses, and that variability is often what makes them useful. Instead, property-based testing provides a way to constrain this indeterminism within well-defined behavioral boundaries. By defining properties that must always hold and exploring the input space through semantic generators, we shift the focus from predicting exact outputs to verifying consistent behavior.</cite>

This is the key framing for AI agent testing: you can't test that an LLM produces an exact output. You CAN test that its output satisfies properties — that it always returns a valid status code, that it never returns empty when given non-empty input, that its JSON is always parseable, that its recommendations always stay within the permitted range. Properties constrain without over-specifying.

---

## Questions & Gaps
- Kiro's PBT page was last updated November 2025 — what has changed in the PBT workflow since then? The feature may have evolved significantly.
- The "optional by default" setting for PBT in task execution — can this be made mandatory in a CLAUDE.md or Kiro configuration without modifying each spec individually?
- Kiro extracts properties from EARS requirements automatically — what happens with requirements that aren't in EARS format? Does it fall back to natural language parsing or skip them?
- For real-time systems (bidding, WebSocket handlers), how does Kiro handle async PBT? The test framework needs to handle async behavior for auction/bidding invariants to be meaningful.

## Related Notes
- [fast-check: Property-Based Testing for JavaScript/TypeScript](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/testing/fast-check-property-based-testing.md) — the fast-check implementation for Claude Code sessions. Use alongside Kiro's built-in PBT.
- [Anthropic Agentic PBT: Finding Bugs Across the Python Ecosystem](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/anthropic-agentic-pbt-python-ecosystem.md) — Anthropic's PBT agent research. The six-step process there (analyze → understand → propose properties → write tests → execute → self-reflect) can be implemented as a Kiro skill for function-level PBT beyond what Kiro's spec-driven PBT covers.
- [Uncle Bob: Extreme Constraints for AI Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/uncle-bob-extreme-constraints-ai-agents.md) — PBT as part of the gauntlet. Kiro's built-in PBT is the easiest path to adding the PBT constraint to the gauntlet.
- [kiro-config Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/kiro-config-repo-overview.md) — your existing kiro-config. EARS-format requirements in spec files will activate Kiro's automatic property extraction; adding a fast-check skill fills the gap for function-level properties.
