# Git at Any Scale — Cursor's Continuity Storage System

**Source:** https://cursor.com/blog/git-at-any-scale
**Author:** Vicent Martí (Cursor / Anysphere) · Aug 18, 2026 · 27 min read
**Saved:** 2026-08-19
**Tags:** technology, infrastructure, fundamentals, security, tools

> Author context: Vicent Martí previously worked at GitHub on the Spokes system (the distributed Git storage system described in this post) and at other VCS teams. This is both a technical post and an announcement of Cursor's "Origin" Git hosting platform, built on their new "Continuity" storage engine. Includes interactive simulators for the concepts described.

---

## TL;DR
Hosting Git at scale is hard because Git's packfile-based design requires random reads across large binary files on local disk — a pattern that defeats distributed filesystems, makes distributed key-value stores prohibitively expensive due to DAG traversal round-trips, and creates fundamental ceiling/floor scalability problems in consensus-based systems (Spokes/3PC). Cursor's Continuity fixes this with a WAL stored in S3 as the source of truth: stateless nodes, unlimited replicas, no routing tables, consistency verified via ETag conditional GETs rather than consensus protocols. Performance: 120 pushes/s on S3 Standard, 300+ on S3 Express One Zone.

---

## Why Git Hosting Is Hard (The Fundamental Problem)

Git is a **directed acyclic graph (DAG)** content-addressed store. Every object (blob, tree, commit) is keyed by SHA-1. Git was designed for decentralized workflows — Linus Torvalds' use case was the Linux Kernel — but the average company uses a centralized host with hundreds or thousands of developers. The tension: a distributed VCS that everyone uses in a centralized way.

**The packfile problem:** Git stores and transmits data as packfiles — binary files where objects are stored as deltas on top of each other, placed randomly throughout the file. Accessing any object requires:
1. Following logical hops through the DAG (commit → tree → blobs → parent commit)
2. Following physical hops through the packfile (each object is a delta on another, stored at a random offset)

At every step of the DAG walk, you don't know the next pointer until you fetch the previous one. This makes every round-trip expensive in a distributed setting.

**Three approaches to distributing Git (in increasing complexity):**
1. Distribute the filesystem
2. Distribute the packfiles
3. Distribute Git itself

All three have been tried. None are simple.

---

## Approach 1: Distribute Git Objects (Key-Value Store) — Doesn't Work

Git is content-addressed → intuitive fit for a distributed KV store (SHA-1 = key, object = value).

**Why it fails:** To list recent changes in a repo, you must walk the DAG. To walk the DAG, you need each pointer's value before you can fetch the next one. In a distributed KV store, each fetch = a round-trip. 54 objects = 54 round-trips. Prohibitively expensive.

Google tried this (Shawn Pearce, JGit + DHT). Results were acceptable for normal Git operations but `git clone` performance was bad enough to abandon the design. The Git protocol still requires packfiles over the network regardless of how you store objects server-side.

---

## Approach 2: Distribute the Filesystem — Doesn't Work Either

GitHub's early approach: distribute the filesystem, keep everything else unchanged.

**NFS:** Too slow and buggy. Git makes aggressive assumptions about filesystem semantics (locking, tearing, syncing) that work on a local disk but break over a networked filesystem.

**Block-level replication (GFS, DRBD):** Terrible to operate, no performance payoff. The fundamental issue: the random read patterns across packfiles don't cache well when you have hundreds of thousands of repositories on the same filesystem.

GitHub eventually gave up on distributed filesystems and built an RPC system — repositories on dedicated fileservers, Rails app doing all operations remotely. Provided horizontal scalability but not availability (still one server per repo).

---

## Approach 3: Spokes — Application-Level Replication (The Industry Standard Since 2013)

Spokes was developed at GitHub around 2013 and has become the de facto standard for Git hosting at scale. Most major Git hosting services use a Spokes-like design.

**Spokes' three correct choices:**
1. Don't distribute Git itself — work at the packfile level
2. Store all data as actual Git repositories on local NVMe disks
3. Keep all copies consistently in sync (not eventually consistent)

**Why eventual consistency is wrong for Git:** Git clients don't tolerate it. Push a commit, fetch it immediately — it must be there. Run CI across 100 runners — all must find the same commit. Eventual consistency has sharp edges that make Git clients behave strangely.

**How Spokes achieves consistency — 3PC (Three-Phase Commit):**

A Git push has two parts:
- **Packfile** — the objects (blobs, trees, commits)
- **Reference transaction** — updates the branch reference to point to the new commit

Key insight: a pushed commit is not visible until its reference is updated. So Spokes fans out packfiles to all hosts simultaneously (no sync needed — packfiles aren't visible yet) and then does 3PC only for the reference transaction (much smaller, faster to sync).

```
Push lifecycle:
1. Client sends packfile
2. Spokes fans out packfile to all replicas (async, no sync)
3. Coordinator starts 3PC for reference transaction:
   VOTING → PRE-COMMIT → DO COMMIT
4. Push accepted only if majority of nodes acknowledge
```

**Spokes' fundamental flaws (visible in 2026, not in 2013):**

**Ceiling problem (monorepos):** 3PC latency is bounded by the slowest node. Adding replicas makes push throughput worse. In 2026, enterprise monorepos need many more than 3 replicas for CI load — but adding replicas degrades push performance.

**Floor problem (agent-created repos):** Agents create millions of small, throwaway repositories. Spokes still requires 3 replicas per repo minimum (quorum). Three mostly idle replicas for repos barely touched = massive waste.

**Operational burden:** Repositories on disk ARE the source of truth. Every copy is critical. Must maintain routing tables mapping every repo to every server. Must checksum continuously. Corruption must be detected and repaired immediately. Treat repositories as pets, not cattle.

---

## Continuity — Cursor's WAL-Based System

**Core insight:** Put the source of truth in S3, not on disk. Disk is a warm cache.

**The WAL design:**
- Every push stored as a WAL entry in S3
- Local NVMe disk holds a regular Git repo (same as Spokes — this is still right)
- WAL index file in S3 tracks which entries have been published
- Push only acknowledged after persisted to S3 AND reference transaction prepared locally

**The three guarantees:**
1. Never acknowledge a push until fully persisted to WAL
2. All pushes are linearizable (atomic CAS on S3 WAL index enforces this)
3. Every view of every repository is fully consistent

### Consensus — None Needed

No routing tables. No primary election. No consensus protocol.

Any node can be the primary for any repository — all updates to the WAL use atomic compare-and-swap (CAS) on S3. If two nodes race to push to the same repo, CAS conflict → one wins, the other refetches and retries. In practice, rendezvous hashing routes repos to consistent nodes for efficiency, but correctness doesn't depend on it.

If a node is missing the repository: materialize it from the WAL. The system is stateless.

### Replication — UDP Gossip + ETag Verification

Optimistic replication via UDP gossip packets — each packet carries metadata for replicas to catch up from S3. UDP is unreliable. That's fine.

**The key mechanism:** Each replica tracks the ETag of the last WAL index version it's synced to. On a read request:
- Conditional GET to S3 with expected ETag
- **304 (no body, <10ms)** → replica is up to date → serve immediately
- **200 (new WAL index)** → catch up from S3 → serve

UDP gossip lost? Replica topology shifted? Doesn't matter. Every read verifies against S3. Always correct when degraded, always fast when healthy.

### Compaction — Primary Only

The WAL cannot grow unbounded. Git repos also need periodic repacking (each push creates a new packfile; lookup across 100 packfiles = 100 index lookups).

Continuity's approach: only the primary does compaction. Compaction result applies to both the on-disk repo and the WAL. Replicas follow the WAL → they follow the compaction. Replicas don't repack — they download the already-compacted packs from S3 (trading bandwidth for CPU).

### Performance Numbers

| Backend | Push throughput | Bottleneck |
|---------|----------------|-----------|
| S3 Standard | ~120 pushes/s | S3 PUT latency |
| S3 Express One Zone | 300+ pushes/s | Git compaction speed |

Linear read scaling verified up to 100 replicas in synthetic stress tests.

Both large monorepos (deploy hundreds of replicas for CI load) and millions of tiny agent-created repos (one replica or zero — materialize on demand) are served efficiently.

---

## The Comparison Table

| Property | Spokes (GitHub) | Continuity (Cursor) |
|----------|----------------|---------------------|
| Source of truth | Replicas on disk | WAL in S3 |
| Consensus mechanism | 3PC per push | CAS on S3 WAL index |
| Routing | Routing table in external DB | Rendezvous hashing, stateless |
| Replicas | Fixed (typically 3) | Any number |
| Scaling ceiling | Push latency degrades with replicas | Linear read scaling to 100+ |
| Scaling floor | Always 3 replicas minimum | 0–1 for idle repos |
| Operational model | Repositories as pets | Repositories as cattle |
| Consistency model | Strong (3PC) | Strong (ETag verification) |
| Disk role | Source of truth | Warm cache |
| Repair on corruption | Must repair immediately (is source of truth) | Materialize from WAL |

---

## The Agent-Era Motivation

> "Agents have fundamentally changed the way we work with software, and in many ways they've made this situation worse. More code, more PRs, more CI runs."

Two agent-specific challenges that motivated Continuity:

**Monorepo load:** Agents working in parallel create CI bursts that 3-replica Spokes clusters can't serve. Linear read scaling with unlimited replicas solves this.

**Throwaway repos:** Agents create millions of small repos. Requiring 3 replicas each wastes enormous resources. Continuity's floor scales to 0 (materialize on demand).

---

## What Was Right in Spokes (Continuity Kept)

1. **NVMe on-disk storage for Git repos** — the random read patterns require local fast disk; this will always be true
2. **Strong consistency** — the Git client cannot tolerate eventual consistency; any correct Git hosting system must be strongly consistent
3. **Work at the packfile level** — don't distribute Git itself; don't replace packfiles on-disk

---

## Questions & Gaps
- The WAL in S3 is the source of truth — what is the recovery procedure when S3 itself is unavailable? Is there a local WAL cache or does the system simply stop accepting pushes?
- Rendezvous hashing for routing is "all the state we require." But the set of healthy nodes must come from somewhere. How is this cluster membership tracked, and what happens during rolling deploys when membership is changing?
- The 120 pushes/s on S3 Standard — is this per-repo, per-cluster, or total? For a hot monorepo with many CI systems pushing simultaneously, what's the practical ceiling?
- Azure DevOps uses blob storage + SQL Server for references — Cursor uses blob storage + WAL for everything. What's the trade-off for large reference transactions (repos with hundreds of thousands of branches)?
- "Origin" is announced as a product offering — is this available as a GitHub/GitLab migration target today, or still early access?

## Related Notes
- [System Design Playbook — Neo Kim](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/system-design-playbook-neo-kim.md) — Continuity's WAL-first design is a clean application of the system design principles described there: single source of truth, operations log for recovery, stateless compute nodes, blob storage as durable foundation.
- [React 16 Fiber Rewrite Engineering Story](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/technology/react-16-fiber-rewrite-engineering-story.md) — Both articles describe large-scale infrastructure rewrites made possible by clear API contracts at the boundary. React kept the public API; Continuity kept the Git packfile protocol. The constraint (don't change what clients see) is what forces correct architecture.
- [NEEDLE Production Safety Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Production-Safety-Guide.md) — NEEDLE's bead queue uses SQLite `BEGIN IMMEDIATE` for atomic claims (the same "atomic compare-and-swap" principle that Continuity uses on S3 for WAL updates). The pattern: atomic operations at the coordination layer, everything else stateless.
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — Spokes' flaw is treating repositories as pets (each copy individually important, must track location, must repair immediately). Continuity treats repos as cattle (any replica is disposable; materialize from WAL on demand). The same cattle/pets mental model from Jed Arden's agent design applies to distributed storage systems.
