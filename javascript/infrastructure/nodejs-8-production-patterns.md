# 8 Node.js Patterns for Production Scale

**Source:** Screenshot of Medium article (paywalled — full article unavailable)
**Fetched from:** Public sources — dev.to, 1xapi.com, apiscout.dev, stacklesson.com
**Saved:** 2026-06-05
**Tags:** javascript, technology, infrastructure, security, backend, node

> ⚠️ Paywalled — notes based on image content plus public coverage of the same patterns. The original Medium article's specific code examples and commentary are not included.

---

## TL;DR
Eight battle-tested Node.js patterns that separate services handling 10 users from services handling a million: Circuit Breaker and Bulkhead prevent cascading failures; Cache-Aside and Connection Pool reduce DB exhaustion; Async Job and Event-Driven decouple slow work from the request path; Rate Limiting protects from abuse; Graceful Shutdown prevents dropped requests during deploys.

## Key Concepts & Terms
- **Cascading failure**: A single slow or failing downstream service causes all upstream callers to pile up, exhausting connection pools and memory until the healthy service becomes unhealthy too. The most common cause of total outages in microservice architectures.
- **Circuit Breaker**: A state machine (Closed → Open → Half-Open) that sits in front of any external call. When failure rate exceeds a threshold, the circuit opens and fails fast without attempting the call — giving the downstream service time to recover. Standard Node.js library: `opossum` (v9.0+ requires Node 20+).
- **Bulkhead**: Named after watertight compartments in ships. Limits the concurrency of calls to each downstream service independently, so a surge to the payment API can't exhaust concurrency for inventory and notification services. Standard Node.js library: `p-limit`.
- **Cache-Aside (Lazy Loading)**: Application checks cache first; on miss, fetches from DB, writes to cache, returns result. Reduces DB load without requiring the cache to be pre-populated. Risk: cache stampede (thundering herd) on cold start or expiry — mitigated with the single-flight pattern.
- **Async Job Pattern**: Moves time-consuming work (email, image processing, report generation) off the synchronous request path. API responds immediately with a job ID; a worker processes asynchronously. Keeps p99 latency low regardless of background work volume.
- **Event-Driven Pattern**: Services communicate by emitting and consuming events (via a message broker like Redis, Kafka, RabbitMQ) rather than direct HTTP calls. Services become loosely coupled — the emitter doesn't know or care who consumes. Natural fit for workflows that fan out to multiple consumers.
- **Graceful Shutdown**: On SIGTERM (sent by Kubernetes, PM2, or Docker when replacing a process), the service stops accepting new connections, drains in-flight requests, closes DB/cache connections cleanly, then exits. Without it, deployments drop in-flight requests and leave DB connections orphaned.
- **Rate Limiting**: Caps requests per client per time window. Protects against abuse, runaway clients, and accidental DDoS. Implemented at the application layer (`express-rate-limit`) or infrastructure layer (nginx, API gateway). Redis-backed for multi-instance deployments.
- **Connection Pool**: Reuses a pre-established set of DB connections rather than opening/closing one per query. Prevents connection exhaustion under load. Every major Node.js DB library (`pg`, `mysql2`, `mongoose`) has built-in pooling — the key is tuning pool size to your actual concurrency.

## The Eight Patterns in Detail

### 1. Circuit Breaker — Resilience · Fault Tolerance
Three states:
- **Closed** (normal): requests pass through, failures counted
- **Open** (failing): fast-fail immediately without calling downstream; starts a timer
- **Half-Open** (recovering): allows one probe request; if it succeeds, circuit closes; if it fails, reopens

```javascript
const CircuitBreaker = require('opossum');

const breaker = new CircuitBreaker(callPaymentAPI, {
  timeout: 3000,           // fail if call takes >3s
  errorThresholdPercentage: 50,  // open if >50% of calls fail
  resetTimeout: 30000,     // try half-open after 30s
});

breaker.fallback(() => ({ status: 'degraded', cached: true }));
breaker.on('open', () => metrics.increment('circuit.open'));
```

**Critical:** integrate circuit state into your health check endpoint. A load balancer that doesn't know a circuit is open will keep routing traffic to a node that's fast-failing everything.

### 2. Bulkhead — Isolation · Stability
Limits concurrency per downstream service independently. Without bulkheads, a surge to one service consumes all event loop concurrency.

```javascript
const pLimit = require('p-limit');
const paymentLimit = pLimit(10);   // max 10 concurrent payment calls
const inventoryLimit = pLimit(50); // independent limit for inventory
```

**Pair with circuit breaker:** bulkhead prevents resource exhaustion; circuit breaker prevents repeated calls to a failing service. Together they're the full resilience primitive.

### 3. Cache-Aside — Performance · Scalability
Read path: check cache → on hit, return; on miss, read DB → write cache → return.
Write path: write DB → invalidate cache (not update — invalidate avoids stale data races).

```javascript
async function getUser(id) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await db.query('SELECT * FROM users WHERE id = $1', [id]);
  await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 300); // 5min TTL
  return user;
}
```

**Cache stampede risk:** on TTL expiry, N concurrent requests all miss and hit the DB simultaneously. Mitigate with: probabilistic early expiry, a distributed lock (Redis `SET NX`), or the single-flight pattern (deduplicate in-flight cache misses at the application layer).

### 4. Async Job — Responsiveness · Throughput
API receives request → validates → enqueues job → returns `202 Accepted` with job ID. Worker picks up job from queue → processes → updates job status in DB. Client polls or receives webhook on completion.

```javascript
// API handler — returns immediately
app.post('/reports', async (req, res) => {
  const job = await queue.add('generate-report', { userId: req.user.id });
  res.status(202).json({ jobId: job.id, status: 'queued' });
});

// Worker — runs separately
worker.process('generate-report', async (job) => {
  const report = await generateReport(job.data.userId);
  await db.updateJobStatus(job.id, 'complete', report);
});
```

Standard Node.js library: `BullMQ` (Redis-backed). Gives retries, delays, priorities, and job history out of the box.

### 5. Event-Driven — Decoupling · Scalability
Producer emits an event to a broker; consumers subscribe independently. Producer doesn't know or care how many consumers exist or what they do.

```javascript
// Producer — just emits
eventBus.emit('user.registered', { userId, email });

// Consumer A — sends welcome email
eventBus.on('user.registered', async ({ email }) => {
  await sendWelcomeEmail(email);
});

// Consumer B — provisions trial subscription
eventBus.on('user.registered', async ({ userId }) => {
  await createTrialSubscription(userId);
});
```

**When to use over direct HTTP:** when multiple services need to react to the same event; when the producer shouldn't wait for consumers; when you want to add consumers later without touching the producer. Message brokers: Redis Streams, Kafka, RabbitMQ, or AWS SNS/SQS.

### 6. Graceful Shutdown — Reliability · Zero Downtime
```javascript
let isShuttingDown = false;

// Health check returns 503 during shutdown — load balancer stops routing
app.get('/health', (req, res) => {
  res.status(isShuttingDown ? 503 : 200).json({ ok: !isShuttingDown });
});

process.on('SIGTERM', async () => {
  isShuttingDown = true;

  // Stop accepting new connections; drain existing ones
  server.close(async () => {
    await db.pool.end();        // close DB connections
    await redis.quit();         // close cache connections
    process.exit(0);
  });

  // Force exit if drain takes too long
  setTimeout(() => process.exit(1), 10_000);
});
```

**With PM2:** `wait_ready: true` + `kill_timeout: 10000` in `ecosystem.config.js`. With Kubernetes: set `terminationGracePeriodSeconds: 30` and ensure your readiness probe checks `isShuttingDown`.

### 7. Rate Limiting — Protection · Stability
```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

app.use('/api/', rateLimit({
  windowMs: 60 * 1000,    // 1 minute window
  max: 100,               // 100 requests per window per IP
  store: new RedisStore(), // shared across instances
  standardHeaders: true,  // return RateLimit-* headers
  message: { error: 'Too many requests', retryAfter: 60 },
}));
```

**Tiered limits:** authenticated users get higher limits than anonymous. Premium users get higher limits than free tier. Implement per-user limits via a Redis key on `req.user.id` rather than IP.

### 8. Connection Pool — Efficiency · Performance
```javascript
// pg (PostgreSQL) — configure pool at startup, reuse everywhere
const { Pool } = require('pg');
const pool = new Pool({
  max: 20,           // max connections — tune to DB server's max_connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Never open/close connections per query — always use the pool
const result = await pool.query('SELECT * FROM users WHERE id = $1', [id]);
```

**Tuning:** pool size = (core count × 2) + active disk spindles is a common starting point. Too small → connections queue up under load. Too large → DB server exhausts `max_connections` and starts rejecting. Monitor pool queue depth, not just query latency.

## Pattern Interaction Map

| Failure scenario | Primary pattern | Supporting pattern |
|-----------------|----------------|-------------------|
| Downstream API goes down | Circuit Breaker | Bulkhead |
| Traffic spike to one service | Bulkhead | Rate Limiting |
| DB overload | Cache-Aside | Connection Pool |
| Slow background task blocking API | Async Job | Event-Driven |
| Deploy drops in-flight requests | Graceful Shutdown | Health check integration |
| Abuse / runaway client | Rate Limiting | Circuit Breaker (per-client) |

## Questions & Gaps
- The original article's specific threshold recommendations (failure %, reset timeout) for circuit breakers in Node.js — not available from the image alone
- Cache-aside TTL strategy for different data types (user sessions vs. product catalog vs. analytics) — how do you decide?
- How does the Async Job pattern interact with NEEDLE's bead queue? They solve similar problems (deferring work) at different layers — worth thinking through the boundary.
- Connection pool sizing for serverless/edge deployments where instances spin up and down — traditional pool sizing rules don't apply cleanly.
- Rate limiting in a multi-region deployment: Redis-backed rate limits need low-latency Redis; how does cross-region latency affect per-request rate limit checks?

## Related Notes
- [PatternsDev/skills Overview](https://github.com/LutherCalvinRiggs/research/blob/main/repos/patterns-dev/patterns-dev-skills-overview.md) — PatternsDev covers front-end JavaScript patterns; this note covers the complementary back-end Node.js resilience patterns. Together they cover the full stack.
