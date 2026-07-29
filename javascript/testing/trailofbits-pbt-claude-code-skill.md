# Trail of Bits: Property-Based Testing Claude Code Skill

**Source:** https://agenticskills.io/skills/property-based-testing → https://github.com/trailofbits/skills/tree/main/plugins/property-based-testing
**Saved:** 2026-07-23
**Tags:** ai, tools, testing, security, javascript, agentic-ai

> Author: Trail of Bits (premier security research firm). Multi-language (Python/Hypothesis, JavaScript/TypeScript/fast-check, Rust/proptest, Solidity/echidna, Vyper). Installable as Claude Code plugin or Kiro/Codex skill. MIT licensed.

---

## TL;DR
Trail of Bits' PBT skill for Claude Code auto-detects high-value testing patterns in your code (serialization pairs, normalizers, parsers, validators, pure functions, smart contract invariants) and proactively suggests or writes property-based tests. Covers the full workflow: test design, generation, review, failure interpretation, and refactoring for testability. The strength hierarchy (No Exception → Type Preservation → Invariant → Idempotence → Roundtrip) is the key decision framework. Installation: one command.

---

## Installation

**Claude Code (plugin):**
```bash
/plugin install trailofbits/skills/plugins/property-based-testing
```

**Codex:**
```bash
codex plugin marketplace add trailofbits/skills
codex plugin add property-based-testing@trailofbits
```

**Manual (any agent with SKILL.md support):**
```bash
curl -o SKILL.md https://github.com/trailofbits/skills/tree/main/plugins/property-based-testing/skills/property-based-testing/raw/main/SKILL.md
```

**Kiro:** Place the SKILL.md in your kiro-config skills folder, or reference the GitHub URL in your skill definitions.

---

## What the Skill Detects (Auto-Triggers)

The skill activates automatically when Claude detects these patterns in code:

| Pattern | Examples |
|---------|---------|
| **Serialization pairs** | encode/decode, serialize/deserialize, toJSON/fromJSON |
| **Parsers** | URL parsing, config parsing, protocol parsing |
| **Normalizers** | normalize, sanitize, clean, canonicalize |
| **Validators** | validate, verify, check, assert |
| **Pure functions** | No side effects, deterministic output |
| **Smart contract invariants** | State consistency, access control |

Also invokable explicitly: "use property-based testing on this function."

---

## The Strength Hierarchy

The most important decision framework in the skill:

```
Weakest ──────────────────────────────────────── Strongest

No Exception → Type Preservation → Invariant → Idempotence → Roundtrip
```

**No Exception**: Function doesn't crash on valid input. Weakest — this is the floor, not the goal.
```javascript
fc.assert(fc.property(fc.string(), (s) => {
  parse(s); // just shouldn't throw
}));
```

**Type Preservation**: Output type matches expected type. Slightly stronger.
```javascript
fc.assert(fc.property(fc.string(), (s) => {
  expect(typeof normalize(s)).toBe('string');
}));
```

**Invariant**: Universal rule about the output.
```javascript
fc.assert(fc.property(fc.array(fc.integer()), (arr) => {
  const sorted = sort(arr);
  expect(sorted.length).toBe(arr.length); // length preserved
}));
```

**Idempotence**: Applying twice = applying once.
```javascript
fc.assert(fc.property(fc.string(), (s) => {
  expect(normalize(normalize(s))).toEqual(normalize(s));
}));
```

**Roundtrip**: Encode → Decode = original. Strongest and most valuable.
```javascript
fc.assert(fc.property(fc.anything(), (value) => {
  expect(deserialize(serialize(value))).toEqual(value);
}));
```

**Key antipattern the skill warns against:** "The test failed, so it's a bug." Failures require validation — the property might be wrong, not the code.

---

## The Skill's Six Reference Files

The SKILL.md routes to six reference documents based on the current task:

| Task | Reference file | Content |
|------|---------------|---------|
| Writing new tests | `generating.md` | Test generation patterns and examples by language |
| Complex input generation | `strategies.md` | Arbitrary design, combining generators, domain constraints |
| Designing a new feature | `design.md` | Property-Driven Development — define properties before implementation |
| Code hard to test (mixed I/O) | `refactoring.md` | Refactoring patterns that enable stronger property testing |
| Reviewing existing PBT tests | `reviewing.md` | Identifying tautological properties, vacuous tests, weak assertions |
| Language/library reference | `libraries.md` | Hypothesis (Python), fast-check (JS/TS), proptest (Rust), echidna (Solidity) |

---

## Property-Driven Development (Design First)

The `design.md` reference covers a workflow Uncle Bob and others don't typically describe — using properties to drive the design before implementation:

1. Read the requirements
2. Identify what properties the implementation must satisfy
3. Write the property tests (before writing the implementation)
4. Implement until the properties pass

This is TDD but at the property level rather than the example level. For AI agents, this means: specify the properties in the CLAUDE.md or bead body, and the agent implements until those properties pass. The properties ARE the acceptance criteria.

---

## Common Antipatterns the Skill Catches in Review

From `reviewing.md`:

**Tautological properties** — the test always passes regardless of implementation:
```javascript
// BAD: always true
fc.assert(fc.property(fc.integer(), (n) => {
  return true; // tautological
}));

// GOOD: actually constrains behavior
fc.assert(fc.property(fc.integer(), (n) => {
  expect(Math.abs(n)).toBeGreaterThanOrEqual(0);
}));
```

**Vacuous tests** — the precondition is never satisfiable:
```javascript
// BAD: fc.assume always fails, zero tests actually run
fc.assert(fc.property(fc.integer(), (n) => {
  fc.pre(n > 1000000 && n < 0); // impossible
  expect(process(n)).toBeDefined();
}));
```

**Weak assertions** — testing presence, not behavior:
```javascript
// BAD: just tests the function returns something
expect(result).toBeDefined();

// GOOD: tests behavioral invariant
expect(result).toHaveLength(input.length);
expect(result.every((v, i) => i === 0 || result[i-1] <= v)).toBe(true);
```

---

## The "No Exception is Weakest" Principle in Practice

From the skill docs: Teams often settle for "no exception" properties because they're easy to write. But this misses the point of PBT — you want to know if the code is correct, not just that it doesn't crash.

The practical sequence for writing a new property:
1. Start with "no exception" if nothing else comes to mind
2. Immediately ask: what type should the output be? → Type Preservation
3. What rule must always hold about the output? → Invariant
4. Does running this function twice change the result? → Idempotence
5. Is there an inverse function? → Roundtrip

Never stop at step 1 unless you genuinely cannot go further.

---

## Practical Integration for Your Kiro Workflow

**Adding to kiro-config:**

Option 1 — Reference in AGENTS.md:
```markdown
## Property-Based Testing
When implementing functions with serialization, normalization, or validation patterns,
use the Trail of Bits PBT skill to write property-based tests.
Skill location: [URL or path to SKILL.md]
Strength hierarchy: No Exception → Type Preservation → Invariant → Idempotence → Roundtrip
Always push for the strongest applicable property.
```

Option 2 — Add as a skill file in `skills/property-based-testing/SKILL.md`

**Practical trigger in kiro sessions:**
- After implementing a serialization function: "Write property-based tests for this using fast-check"
- After implementing a validator: "Use the PBT skill to write tests for this validator"
- When reviewing AI-generated code: "Review these tests for weak assertions or tautological properties"

---

## Questions & Gaps
- The skill covers JavaScript with fast-check — does it generate the correct fast-check v4 API (`.property()` syntax) vs. the older `fc.assert(fc.property(...))` pattern?
- Smart contract coverage (Solidity/echidna, Vyper) is mentioned but the primary focus is Python/JS/Rust. For the MAC.BID tech stack (Node.js, Next.js), fast-check is the relevant library throughout.
- Property-Driven Development (design.md) is the most valuable workflow for AI coding agents — but it requires writing properties before implementation, which means you need to define properties in the bead body or CLAUDE.md before the worker starts. Is there a template for this?

## Related Notes
- [fast-check: Property-Based Testing for JavaScript/TypeScript](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/testing/fast-check-property-based-testing.md) — this skill wraps fast-check for JavaScript. The strength hierarchy here maps directly to fast-check's property types.
- [Uncle Bob: Extreme Constraints for AI Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/uncle-bob-extreme-constraints-ai-agents.md) — this skill implements Uncle Bob's "property-based testing" constraint. Install it in Claude Code/Kiro and you have the PBT layer of the gauntlet.
- [Jamon Holmgren 18-Point Agentic Setup](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/jamon-holmgren-18-point-agentic-setup.md) — item 9 (bin/tools folder, agents build tools) is exactly what this skill represents. Trail of Bits has done the PBT skill work; install it rather than build from scratch.
- [kiro-config Repo Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/kiro-config-repo-overview.md) — adding this skill to the skills folder is a one-step addition to the existing kiro-config infrastructure.
