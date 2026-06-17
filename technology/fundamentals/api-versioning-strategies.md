# API Versioning Strategies

**Source:** Screenshot — infographic by Neo Kim (newsletter.systemdesign.one)
**Saved:** 2026-06-17
**Tags:** technology, infrastructure, javascript, fundamentals, design

> ⚠️ Paywalled — notes based on image content (infographic) only.

---

## TL;DR
API versioning allows clients to depend on a stable contract while the API evolves. The core rule: version only on breaking changes, not every release. Four main strategies (URI path, query parameter, header/content negotiation, message payload) each offer different tradeoffs between URL cleanliness, granularity, and client complexity.

## Key Concepts & Terms

- **Breaking change**: Any change that requires existing clients to update — removing or renaming a field, changing a field's type, changing behavior. These require a version bump.
- **Non-breaking change**: Adding optional fields. Existing clients safely ignore them. No version bump needed.
- **Version only on breaking changes**: The core discipline. Over-versioning creates unnecessary maintenance burden; under-versioning breaks clients silently.

## The Four Versioning Strategies

### 1. URI Path Versioning
```
/api/v1/users
/api/v2/users
```
- Simple and widely used
- Versions the entire API surface at once
- Visible in URLs, logs, and browser history
- Best for: public APIs with clear major version boundaries

### 2. Query Parameter Versioning
```
/api/users?version=2
```
- Per-resource versioning (can version individual endpoints independently)
- Defaults to latest when parameter is omitted
- URL path stays clean
- Best for: APIs where different resources evolve at different speeds

### 3. Header / Content Negotiation
```
Accept-Version: v2
```
- Keeps URLs clean
- Version information travels in request headers
- Ideal for long-lived clients that cache URLs
- Slightly more complex for clients to implement
- Best for: mature APIs with stable URL surfaces and sophisticated clients

### 4. Message Payload Versioning
```json
{ "version": "v1", "data": { ... } }
```
- Version embedded in the request/response body
- Best for async and queue-based systems where headers aren't available or reliable
- Works well with message brokers (Kafka, SQS, RabbitMQ)

## Breaking vs Non-Breaking Changes

| Change type | Breaking? | Version bump? |
|-------------|-----------|--------------|
| Remove a field | ✅ Yes | Required |
| Rename a field | ✅ Yes | Required |
| Change a field's type | ✅ Yes | Required |
| Add an optional field | ❌ No | Not needed |
| Add a new optional endpoint | ❌ No | Not needed |

## Best Practices

1. **Prefer additive changes** — adding optional fields never breaks clients
2. **Document and deprecate clearly** — clients need time to migrate; give them advance warning and a timeline
3. **Keep versions predictable** — v1, v2, v3 (not v1.1.3.2); treat version numbers as contracts, not release numbers

## Strategy Selection Guide

| Strategy | URL Cleanliness | Per-resource control | Async-friendly | Client complexity |
|----------|-----------------|---------------------|---------------|------------------|
| URI Path | Low | No (whole API) | Yes | Low |
| Query Param | Medium | Yes | Yes | Low |
| Header | High | Possible | No | Medium |
| Payload | High | Yes | Yes | Medium |

## Questions & Gaps
- How do versioning strategies interact with API gateways and reverse proxies? URI path versioning is easiest to route; header-based versioning requires the gateway to inspect headers.
- What's the recommended deprecation timeline? The infographic says "deprecate clearly" but doesn't specify — 6 months, 1 year, or tied to client adoption metrics?
- For message payload versioning in async systems: how do you handle schema evolution when messages from multiple versions coexist in a queue?
- GraphQL and gRPC have their own versioning philosophies (schema evolution, field deprecation). How does this framework translate?

## Related Notes
- [Node.js 8 Production Patterns](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/nodejs-8-production-patterns.md) — versioning discipline is foundational to the API-first patterns described there (circuit breakers, rate limiting, graceful shutdown all operate against versioned API surfaces).
