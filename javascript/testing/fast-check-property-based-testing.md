# fast-check — Property-Based Testing for JavaScript and TypeScript

**Source:** https://fast-check.dev/docs/introduction/ (official documentation, primary source)
**Also:** https://github.com/dubzzz/fast-check · https://www.pkgpulse.com/guides/property-based-testing-fast-check-javascript-2026
**Saved:** 2026-07-23
**Tags:** javascript, testing, fundamentals, tools, technology

> Author: Nicolas Dubien. 10M+ weekly npm downloads. Used by Jest, Jasmine, fp-ts, io-ts, Ramda, js-yaml. Works with Vitest, Jest, Mocha, Jasmine, and Node.js native test runner. Compatible with TypeScript. MIT licensed.

---

## TL;DR
fast-check generates random inputs for you, runs your assertion over hundreds of cases, and shrinks any failure down to the smallest input that still reproduces it. Instead of you choosing specific values to test, you describe properties that should hold for all valid inputs. Where example-based tests catch the bugs you thought of, property-based tests catch the bugs you didn't. The two approaches are complementary, not competing — add fast-check to existing test suites without rewriting anything.

---

## Examples vs. Properties: The Core Distinction

**Example-based test (what everyone writes today):**
```javascript
test('sort returns sorted array', () => {
  expect(sort([3, 1, 2])).toEqual([1, 2, 3]);
  expect(sort([5])).toEqual([5]);
  expect(sort([])).toEqual([]);
});
```
You chose these specific inputs. The test only runs on what you thought to test.

**Property-based test:**
```javascript
import fc from 'fast-check';

test('sort: output is always sorted', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const result = sort(arr);
      // Property: every adjacent pair must be in order
      for (let i = 0; i < result.length - 1; i++) {
        expect(result[i]).toBeLessThanOrEqual(result[i + 1]);
      }
    })
  );
});

test('sort: output has same elements as input', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const result = sort(arr);
      expect(result).toHaveLength(arr.length);
      expect(result.sort()).toEqual([...arr].sort());
    })
  );
});
```
fast-check generates 100 random arrays per test run, finds any input that breaks the property, and shrinks it to the simplest failing case.

---

## The Three Mechanisms

### 1. Arbitraries (Input Generators)

Arbitraries generate random values of a specific type. They compose into complex types.

**Primitive arbitraries:**
```javascript
fc.integer()           // random integers
fc.float()             // random floats
fc.string()            // random strings
fc.boolean()           // true/false
fc.bigInt()            // BigInt values
fc.date()              // Date objects
fc.uuid()              // UUID strings
```

**Composed arbitraries:**
```javascript
fc.array(fc.integer())              // arrays of integers
fc.record({                          // objects with typed fields
  id: fc.uuid(),
  name: fc.string({ minLength: 1 }),
  amount: fc.float({ min: 0, max: 10000 }),
})
fc.tuple(fc.string(), fc.integer()) // fixed-length tuples
fc.option(fc.integer())             // null or integer
fc.oneof(fc.integer(), fc.string()) // either type
```

**Domain-specific arbitraries:**
```javascript
fc.emailAddress()       // valid email strings
fc.domain()             // valid domain names
fc.ipV4()               // valid IPv4 addresses
fc.webUrl()             // valid URLs
fc.lorem({ maxCount: 5 }) // lorem ipsum words
```

**Custom arbitraries (for your domain models):**
```javascript
const bidArbitrary = fc.record({
  bidId: fc.uuid(),
  lotId: fc.uuid(),
  customerId: fc.uuid(),
  amount: fc.float({ min: 0.01, max: 999999.99 }),
  timestamp: fc.date({ min: new Date('2020-01-01') }),
});

fc.assert(
  fc.property(bidArbitrary, (bid) => {
    // property that should hold for all valid bids
    const result = processBid(bid);
    expect(result.status).toBeOneOf(['accepted', 'rejected', 'pending']);
  })
);
```

### 2. Multiple Runs (100 by default)

Each `fc.assert` call runs the property with 100 randomly-generated inputs. The number is configurable globally and per-test.

fast-check's generators are seeded — you can get deterministic behavior by setting a seed:

```javascript
// Global seed (makes all tests deterministic)
fc.configureGlobal({ seed: 42 });

// Per-test seed
fc.assert(fc.property(...), { seed: 42 });
```

fast-check also biases its generators to cover edge cases: arrays include both empty and very long arrays; integers include 0, 1, -1, MAX_SAFE_INTEGER, MIN_SAFE_INTEGER; strings include empty strings and very long strings. It's not naive random — it's designed to find bugs.

### 3. Counterexample Shrinking (The Differentiator)

When fast-check finds a failing input, it automatically reduces it to the smallest possible failing case. A failure found with a 200-element array gets shrunk to the 2-element array that still fails. A failing string of 300 characters gets reduced to the 3 characters that expose the bug.

```
Failure found at input: [847, -293, 0, 1, 99384, -7, 42, -1, 8273, ...]  (50 elements)

Shrinking...
Shrinking...
Counterexample: [0, -1]   ← the actual minimal failing case
```

This makes property-based failures dramatically easier to debug than naive fuzzing, where you're staring at a huge random input trying to understand which part of it caused the failure.

---

## What Properties to Write

The hardest part of property-based testing is knowing what to assert. Four patterns:

### Pattern 1: Invariants (Universal Rules)
Things that should always be true regardless of input.

```javascript
// Sorting invariants
fc.assert(fc.property(fc.array(fc.integer()), (arr) => {
  const sorted = sort(arr);
  // Invariant 1: same length
  expect(sorted).toHaveLength(arr.length);
  // Invariant 2: each element is <= the next
  sorted.every((v, i) => i === 0 || sorted[i-1] <= v);
  // Invariant 3: same elements (as multisets)
  expect(sorted.sort()).toEqual([...arr].sort());
}));

// Financial invariant: after any sequence of credits/debits,
// balance = initial + sum(credits) - sum(debits)
```

### Pattern 2: Roundtrip (Encode → Decode Returns Original)
```javascript
fc.assert(fc.property(fc.string(), (str) => {
  expect(decode(encode(str))).toEqual(str);
}));

fc.assert(fc.property(bidArbitrary, (bid) => {
  expect(parseBid(serializeBid(bid))).toEqual(bid);
}));
```

### Pattern 3: Idempotence (Running Twice = Running Once)
```javascript
fc.assert(fc.property(fc.array(fc.integer()), (arr) => {
  expect(sort(sort(arr))).toEqual(sort(arr));
}));

fc.assert(fc.property(invoiceArbitrary, (invoice) => {
  const normalized = normalizeInvoice(invoice);
  expect(normalizeInvoice(normalized)).toEqual(normalized);
}));
```

### Pattern 4: Oracle (Compare Against Reference Implementation)
```javascript
fc.assert(fc.property(fc.array(fc.integer()), (arr) => {
  // My fast sort vs. naive reference sort
  expect(myFastSort(arr)).toEqual(arr.slice().sort((a, b) => a - b));
}));
```

---

## Integration with Vitest / Jest

```bash
npm install --save-dev fast-check
```

```javascript
// vitest.config.ts or jest.config.ts — no changes needed
// fast-check works inside existing test blocks

// my-feature.test.ts
import { describe, test, expect } from 'vitest'; // or jest
import fc from 'fast-check';

describe('bid processing', () => {
  test('valid bids always produce a result', () => {
    fc.assert(
      fc.property(
        fc.record({
          amount: fc.float({ min: 0.01, max: 999999 }),
          customerId: fc.uuid(),
          lotId: fc.uuid(),
        }),
        (bid) => {
          const result = processBid(bid);
          expect(result).toBeDefined();
          expect(result.status).toMatch(/^(accepted|rejected|pending)$/);
        }
      )
    );
  });
});
```

---

## Why It's Especially Valuable for AI-Generated Code

<cite index="61-1">AI-generated test suites can reach high coverage while killing far fewer mutants — coverage reports which code executed. Mutation score reports whether tests detected injected faults.</cite>

The same problem applies to AI-generated assertions: the agent writes tests that pass because it wrote them to match the code it already wrote. Property-based tests break this feedback loop — the properties are defined by *you* (what should always be true?), and fast-check generates inputs the agent never considered.

<cite index="72-1">Property-based testing can help identify edge cases and potential issues that may not have been considered with example-based tests. fast-check will ensure that tests which depend on arrays receive some arrays with duplicate values. It's common for numerical code to have edge cases around 0, 1, or -1, or when handling unexpectedly large inputs. fast-check numerical arbitraries ensure coverage of values close to the edges of their valid ranges.</cite>

For auction/bidding systems specifically: property-based tests can verify that bidding logic invariants hold across all possible combinations of bid amounts, lot states, customer states, and timing conditions — not just the happy paths the agent was prompted to handle.

---

## What fast-check Has Actually Found in Production

From the official track record: real CVEs discovered in open-source projects used by millions of developers, by fuzzing their input handling with the `__proto__` string, deeply nested objects, and boundary-value inputs. Projects like TypeScript's parser, js-yaml, and Ramda have all found real bugs through fast-check that example-based test suites had missed for years.

---

## Questions & Gaps
- Writing good properties requires understanding what invariants your domain has. For a complex auction system (lot states, bid queues, payment flows), what's the process for identifying which invariants to encode? Is there a systematic approach?
- fast-check generates 100 inputs per test by default. For long-running test suites, does increasing this number make sense or is 100 usually sufficient to find bugs?
- The `seed` determinism feature is important for CI. What's the recommended seed management approach — global seed per project, or per-file?
- For async code (Lambda handlers, WebSocket handlers), how does `fc.asyncProperty` behave differently from the sync version? Are there gotchas?
- The "AI-Powered Testing" section in the fast-check docs (noted in the sidebar) — this wasn't explored in this fetch. Likely covers using fast-check specifically with AI-generated code.

## Related Notes
- [Uncle Bob: Extreme Constraints for AI Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/uncle-bob-extreme-constraints-ai-agents.md) — fast-check is the property-based testing constraint Uncle Bob names. This note is the implementation guide for that constraint.
- [Node.js 8 Production Patterns](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/nodejs-8-production-patterns.md) — property-based tests complement the Circuit Breaker and Connection Pool patterns described there: invariants like "circuit breaker always returns to closed state after threshold success count" are ideal property-based test candidates.
- [JWT Token Refresh Architecture](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/jwt-token-refresh-architecture.md) — the roundtrip property (encode → decode → original) and timing invariants (token is always refreshed before expiry under any network delay) are ideal property-based tests for JWT handling code.
- [Jamon Holmgren 18-Point Agentic Setup](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/jamon-holmgren-18-point-agentic-setup.md) — property-based tests sit between unit tests (item 4) and the false-confidence test audit (item 14). They're the constraint that catches bugs unit tests weren't designed to find.
