# System Design Playbook — 16 Core Patterns

**Source:** System Design Playbook PDF — Neo Kim (newsletter.systemdesign.one) © 2025
**Saved:** 2026-06-17
**Tags:** technology, fundamentals, infrastructure, javascript, design

> ⚠️ Copyright: This PDF is for educational use only (Neo Kim, © 2025). Notes are paraphrased summaries — no verbatim reproduction.

---

## TL;DR
16 short system design explainers covering real production architectures: password hashing, micro frontends, object storage durability, serverless, payments, geospatial indexing, caching, DNS, and JWT. Each pattern is drawn from production systems at scale (Amazon, Uber, YouTube, Apple, Stripe, Meta, Tinder).

---

## 1. How Databases Store Passwords Securely

The core approach: never store the password itself, only a one-way hash (the "fingerprint"). At login, re-hash the input and compare.

**The rainbow table problem**: Pre-computed hash→password mappings make simple hashing reversible. **The fix**: add a unique random string (salt) to each password before hashing. The database stores both the hash and the salt. At login, the server combines the stored salt with the entered password and re-hashes to compare.

Key: every user gets a different salt, so identical passwords produce different hashes. Rainbow tables become useless.

---

## 2. How Micro Frontends Work

Micro frontends extend microservices to the UI layer — slicing a site's frontend into self-contained, domain-driven micro apps.

Key properties: technology-agnostic (React, Angular, Vue can coexist), boundaries set by business domain not technical layer, autonomous teams own a feature from backend to frontend, shared libraries maintain visual consistency, communication between micro apps happens via events/messages. Suited for large organizations with technical maturity.

**The backend-for-frontend (BFF) pattern** is commonly paired: each micro frontend has its own dedicated API aggregator rather than hitting shared services directly.

---

## 3. How Amazon S3 Achieves 99.999999999% Durability (11 Nines)

The durability stack:
- **Metadata and content separated**: metadata in fast key-value store, file content in mechanical storage
- **Erasure coding** instead of simple replication — reduces storage overhead while tolerating disk failures
- **Checksums at multiple layers**: on raw data, in HTTP trailers, on erasure-coded chunks
- **Bracketing**: confirms erasure-coded data is durable before returning success to caller
- **Physical isolation**: data centers with separate networking and power
- **Disk health monitoring**: continuously measures failure rates to scale the repair service
- **Versioning and backups**: protect against user error (the most common cause of data loss)
- **Durability review gate**: changes to storage systems require review before deployment

---

## 4. How Amazon S3 Works (Architecture)

S3 is an object store for unstructured data (logs, media, backups). Built on microservices. Key design decisions:

- Metadata (file info, permissions) stored in key-value database with caching layer for availability
- File content stored on mechanical hard disks (cost optimization)
- **ShardStore** (an LSM-tree variant) organizes data on disk efficiently
- Data replicated across many disks; parallel reads for throughput
- Erasure coding reduces the storage overhead of that replication

---

## 5. How Uber Computes ETA

ETA = shortest path in a directed weighted graph (roads = edges, intersections = nodes, travel time = edge weight).

**Why not Dijkstra?** O(n log n) doesn't scale to road networks with millions of nodes.

**The solution**: partition the graph into regions, precompute the best paths within each partition. Reduces complexity from O(n²) to O(n). Edge weights are updated continuously with live traffic data.

**Map matching**: GPS coordinates rarely fall exactly on roads. The Kalman filter and Viterbi algorithm match noisy GPS signals to the most likely road position before computing ETA.

---

## 6. How YouTube Supports 2.49 Billion Users with MySQL

YouTube built Vitess — an abstraction layer on top of MySQL that makes it horizontally scalable.

Key components: a sidecar server per MySQL instance (manages the instance + caches hot data to prevent thundering herd), a proxy server for query routing to the right shard and connection pooling, a distributed key-value database for shard metadata, automatic rewriting of expensive queries (adding LIMIT), an HTTP server for metadata updates. Written in Go, open-sourced. MySQL's ACID guarantees remain intact throughout.

---

## 7. How AWS Lambda Works

Lambda runs functions on shared workers across many tenants. The key challenge: isolation without the overhead of full VMs.

**MicroVMs (Firecracker)**: lightweight virtual machines that provide tenant-level isolation with much lower overhead than traditional VMs. Each Lambda execution gets its own microVM.

Cold start reduction techniques: microVM snapshots (restores to a pre-initialized state, 90% cold start reduction), lazy-loading container images, deduplication of shared layers between images. Worker metadata persisted in a journal log for fault tolerance.

---

## 8. How Apple Pay Works

The core insight: credit card numbers never touch Apple's servers or the phone's regular storage.

**DAN (Device Account Number)**: when a card is added, the payment network creates a unique number representing the card+device pair. Stored in the Secure Element (a specialized chip on the phone).

**Transaction flow**: Phone communicates with card reader via NFC → reader creates transaction record → phone generates a cryptogram (single-use token) from DAN + transaction details → only the cryptogram and transaction details are transmitted, never the card number → payment network validates by re-generating the cryptogram using its copy of the DAN → bidirectional validation (both sides verify each other).

---

## 9. How Figma Scaled Postgres to 4 Million Users

A textbook database scaling progression:

1. Vertical scaling (bigger machine)
2. Read replicas (scale reads, improve fault tolerance)
3. Caching layer (reduce database load on hot data)
4. Vertical partitioning by domain (move tables into separate databases per business domain)
5. Connection pooler (prevent connection starvation at scale)
6. Column-level partitioning (move high-traffic columns to separate databases)
7. Horizontal sharding (split tables at row level across many databases)

The sequence matters — each step is triggered by a specific bottleneck, and they're done incrementally, not all at once.

---

## 10. How Apple AirTag Works

Location tracking without GPS, WiFi, or a battery-draining radio.

**The mechanism**: AirTag broadcasts its public key via Bluetooth Low Energy every 2 seconds. Any nearby iPhone receives the broadcast, encrypts its own GPS location using that public key, and silently uploads the encrypted location to Apple's servers. Only the AirTag owner holds the private key and can decrypt the location. Apple's servers can't read the location data — they only store encrypted blobs.

The owner's iPhone decrypts the locations from all nearby iPhones that logged a sighting. The network of every iPhone in the world becomes the tracking infrastructure, with cryptographic privacy guarantees.

---

## 11. How Stripe Prevents Double Payments

The core tool: idempotency keys — a UUID sent with every payment request in HTTP headers.

Flow: client generates UUID → sends with request → server stores UUID in database → on success, UUID marked as processed → on retry, server checks UUID, finds prior result, returns it without re-processing. UUID expires after 24 hours.

When payload changes (different amount, different card), a new UUID is generated — this distinguishes intentional new charges from accidental retries.

**Thundering herd**: exponential backoff with jitter prevents all retrying clients from hitting the server simultaneously. ACID database transactions ensure clean rollback on server errors.

---

## 12. How Uber Finds Nearby Drivers

Uber built H3 — a hexagonal hierarchical geospatial indexing system, open-sourced.

**Why hexagons?** Uniform distance between cell centers (unlike squares, where corner cells are further than edge cells). Each hexagon subdivides into exactly 7 smaller hexagons, enabling multi-resolution indexing. Bitwise operations switch between resolution levels in constant time.

Can index areas as small as 1 square meter. Consistent hash ring distributes cells across servers for scalability. Finding nearby drivers = find the driver's H3 cell + adjacent cells at the appropriate resolution.

---

## 13. How Meta Achieves 99.99999999% Cache Consistency

The problem: race conditions between database writes and cache invalidation cause temporary inconsistency. Even a tiny window where cache and database disagree produces inconsistent reads.

**Polaris**: a separate monitoring service that receives cache invalidation events and independently queries cache servers to detect inconsistency. Queries the database at 1/5/10 minute intervals (not continuously) for efficiency.

**Tracing**: a library embedded in each cache server that logs data changes occurring during race condition time windows. Polaris detects that inconsistency occurred; the trace reveals why. Separate invalidation event stream between client and Polaris for reliability.

---

## 14. How Tinder Scaled to 1.6 Billion Swipes/Day

Key patterns: Google S2 library for geospatial indexing (similar to Uber's H3), key-value database for user profiles, change data capture for propagating data changes, 500 microservices on a service mesh, WebSockets for real-time match notifications.

**Hot shard problem** (caused by time zone clustering): co-locate multiple shards on the same server to distribute load; caching layer for reads; rate limiting for writes; exponential backoff with jitter on failures.

Swipe stream → separate matching server → match result pushed via WebSocket. Disliked profiles stored in object storage to feed the recommendation algorithm.

---

## 15. How DNS Works

DNS translates domain names to IP addresses through a hierarchy of servers:

```
Browser cache → OS cache → Resolver server cache
  → Root server (finds the TLD server for .com, .org, etc.)
  → TLD server (finds the authoritative name server for the domain)
  → Authoritative name server (holds the actual IP address)
  → Returns IP to resolver → Returns to browser → Browser connects to web server
```

Each layer caches results according to TTL (Time To Live). The authoritative name server is the ground truth; everything above it is a cached approximation.

---

## 16. How JWT Works

JWT (JSON Web Token) enables stateless authentication — the server doesn't need to store session information.

**Structure**: three base64-encoded parts separated by dots: header (algorithm), payload (claims including user ID, roles, expiry), signature (HMAC of header+payload using server's secret key).

**Flow**: user authenticates → server creates JWT and signs it → client stores JWT → client sends JWT with every request → server validates by re-computing the signature.

**Security considerations**: JWT should only travel over HTTPS. Set minimum role scope and short expiry times to limit damage if stolen. Maintain a server-side denial list (blocklist) for known-stolen JWTs. The secret key must remain server-only — anyone with the secret can forge valid tokens.

---

## Cross-Cutting Patterns in This Playbook

Several techniques appear across multiple systems:

| Pattern | Where it appears |
|---------|-----------------|
| Metadata/data separation | S3, Lambda |
| Erasure coding | S3 durability |
| Idempotency keys | Stripe double-payment prevention |
| Exponential backoff + jitter | Stripe, Tinder |
| Hexagonal/hierarchical geospatial index | Uber drivers (H3), Tinder (S2) |
| Thundering herd prevention | YouTube (sidecar cache), Tinder (rate limiting + caching) |
| Cryptographic public/private key separation | AirTag, Apple Pay |
| Checksums for data integrity | S3 |
| MicroVMs for isolation | Lambda (Firecracker) |
| Sidecar pattern | YouTube (Vitess), Lambda (worker management) |
| Connection pooling | Figma, YouTube (Vitess proxy) |
| Change data capture | Tinder |

## Related Notes
- [API Versioning Strategies](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/api-versioning-strategies.md) — same author (Neo Kim). Complements this playbook's API and infrastructure patterns.
- [Node.js 8 Production Patterns](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/nodejs-8-production-patterns.md) — Node-level implementations of several patterns here (circuit breaker, connection pool, rate limiting, exponential backoff).
- [Agent Platform Security Checklist](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/agent-platform-security-checklist.md) — the JWT security model (minimum scope, short expiry, denial list) maps directly to the checklist's credential rotation and least-privilege principles.
