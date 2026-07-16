# React 16: A Look Inside an API-Compatible Rewrite of Our Frontend UI Library

**Source:** https://engineering.fb.com/2017/09/26/web/react-16-a-look-inside-an-api-compatible-rewrite-of-our-frontend-ui-library/
**Saved:** 2026-07-15
**Tags:** javascript, technology, fundamentals, infrastructure, design

> Author: Sophie Alpert, Facebook Engineering. Published September 26, 2017. The engineering story behind React Fiber — a complete internal rewrite of React while keeping the public API unchanged.

---

## TL;DR
Facebook completely rewrote React's internals (as "React Fiber") while keeping the public API essentially unchanged — like swapping out the engine of a running car. The rewrite was developed alongside the old code using a feature flag, validated by making the existing Jest test suite pass, tracked with a public progress site (isfiberreadyyet.com), and dogfooded on facebook.com and messenger.com before public release. Key engineering lessons: feature flags over long-lived branches, test-driven parity tracking, early dogfooding, incremental rollout starting with smaller surface area.

---

## Why a Complete Rewrite

At time of writing, Facebook had 30,000+ React components in their main web repo. The old implementation had architectural limits that prevented adding frequently-requested features:

- Couldn't support **asynchronous rendering** (processing large component trees without blocking the main thread)
- Couldn't implement **error boundaries** (catching exceptions in component trees)
- Couldn't return **multiple components from render** (returning arrays/fragments)

The new architecture (Fiber) was designed ground-up for async rendering — though React 16 shipped in synchronous mode compatible with existing code. The async capabilities were unlocked incrementally in later versions.

---

## The Engineering Process: Five Key Decisions

### 1. Feature Flags Over Long-Lived Branches

Facebook's standard practice: no long-lived branches (merge conflicts compound). Instead, all new code committed to the same repository, switched with a boolean `useFiber` feature flag at runtime.

Benefits:
- Bug fixes to the old implementation continued uninterrupted
- No merge conflict debt accumulated
- The switch was a boolean value change, not a codebase merge
- New code could be enabled/disabled at deployment time without a code change

### 2. Test-Driven Parity (The Jest Suite as the Contract)

First long-term goal: make the existing Jest test suite pass against the new code.

The new renderer started as a minimal skeleton supporting only a subset of React's APIs, then expanded to pass more tests over time. Most tests already used React's public APIs, so they ran against the new implementation without modification. Tests that relied on implementation details of the old codebase were rewritten as part of the project.

**The daily workflow:**
1. Turn on the `useFiber` feature flag
2. Run the React unit tests
3. Pick a failing test
4. Change the renderer to make it pass
5. (Often) discover the fix also fixed several other tests

Starting state: ~700 of 1,500 unit tests passing. Final state: ~2,000 tests (500 new tests added during the project), all passing.

### 3. The "Ratchet" System (No Regressions)

A text file in the repo listed every test currently passing. Any PR fixing a test added that test's name to the list. CI verified the list was always up-to-date.

Effect: you could not accidentally break a previously passing test and have it go unnoticed. The first day the system was in place, it caught three accidental regressions introduced by engineers on the team.

Plus: `isfiberreadyyet.com` — a public website displaying the pass percentage, updated with each commit. Made progress visible to the team and external contributors.

### 4. Early Dogfooding Before Feature Parity

Rather than waiting for 100% test parity, they enabled Fiber on facebook.com for a few team members when about half the tests were passing.

First version didn't even support most DOM properties — many UIs looked wrong. Once `className` support was added, most UIs looked correct.

**Why dogfood early:**
- Reveals bugs that unit tests don't surface (integration issues, lifecycle ordering)
- Distinguishes "defects in the new implementation" from "brittle components relying on undocumented behavior"
- Provides confidence that a complete swap is achievable before the full investment is made

Some regressions were bugs to fix. Some were components relying on undocumented behavior — in those cases, the team decided case-by-case: make the new implementation conform to old behavior, or update the component and document the breaking change.

### 5. Incremental Rollout Starting with Smaller Surface Area

Staged rollout sequence:
1. Facebook engineering team members only (on facebook.com)
2. messenger.com team (fewer components = smaller bug surface area)
3. All messenger.com employees
4. Wider employee rollout across all components
5. Small fraction of public users (A/B tested)
6. All facebook.com users
7. Mobile apps (React Native, hundreds of screens)

The A/B testing tool was standard practice: with 2+ billion users, even 0.1% rollout reaches hundreds of thousands of users. If an infrastructure change designed to be invisible causes a change in how people use the product, it usually indicates a bug.

The initial messenger.com rollout did show small regressions — tracked down to bugs in React, fixed, experiment reset. The next version performed comparably to the old one.

---

## What React Fiber Unlocked

The new core was designed for async rendering from the ground up. React 16 shipped in synchronous mode for compatibility, but the architecture enabled:
- Error boundaries (exception catching at component boundaries)
- Returning multiple elements from `render` (arrays, fragments)
- Portals
- Streaming server-side rendering
- Eventually: Concurrent Mode, Suspense, React 18's concurrent features

The async capabilities took years to fully surface (Concurrent Mode wasn't stable until React 18, 2022) — but the rewrite in 2017 is what made them possible.

---

## The Meta-Engineering Lessons

**API stability as a first-class constraint.** "Swapping out the engine of a running car" is only possible if you define the public interface as immutable before starting. The constraint — don't break the API — is what forces the architecture to be right. Without that constraint, you'd just add new APIs and leave old ones as cruft.

**The test suite as the specification.** When you can't change the public API, the test suite against that API becomes the complete specification of correct behavior. TDD here wasn't about writing tests first — it was about using the existing test suite as a formal spec the new implementation had to satisfy.

**Ratchet systems prevent invisible regression.** The "list of passing tests in git" pattern is simple and powerful. Making regression visible and code-reviewable turns it from an accidental event into a deliberate decision.

**Dogfood early, even when broken.** Seeing the new code run on real components — even when it breaks things — provides confidence that was impossible to get from unit tests alone. The willingness to run obviously-incomplete code in a controlled production environment is a cultural signal about how Facebook approaches infrastructure changes.

**Incremental rollout with A/B gates.** The staged rollout (team → product team → employees → fraction of users → all users) is a risk management pattern. Each stage has a defined rollback path. The A/B infrastructure provides a quantitative signal for "did this change anything observable?"

---

## Questions & Gaps
- The async rendering architecture was designed in but not shipped until much later — what were the hardest architectural problems that delayed Concurrent Mode from 2017 to 2022?
- The `isfiberreadyyet.com` pattern (public progress tracking during a major internal project) — did this have measurable effects on external contributor participation?
- The feature flag approach assumes a single deployment target. How does this pattern work for open-source libraries where users are running many different versions simultaneously?
- The ratchet system (passing tests in a text file) is simple but requires a shared canonical test runner. At what team size or test suite size does this approach need to be replaced with something more sophisticated?

## Related Notes
- [Node.js 8 Production Patterns](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/nodejs-8-production-patterns.md) — the graceful rollout pattern (circuit breaker, gradual traffic shifting) described there is the runtime equivalent of the staged deployment approach described here. Both manage risk by making change incremental and observable.
- [API Versioning Strategies](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/api-versioning-strategies.md) — React 16's "API-compatible rewrite" is the most extreme application of the versioning principle: the surface API doesn't change at all, only the implementation. This is harder than versioning — it requires the implementation to be fully substitutable.
- [System Design Playbook — How Amazon S3 Works](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/system-design-playbook-neo-kim.md) — S3's durability review gate ("deploy changes only through a durability review") is the same risk management philosophy as React's A/B rollout gate. Large-scale infrastructure changes get explicit review gates before going live.
