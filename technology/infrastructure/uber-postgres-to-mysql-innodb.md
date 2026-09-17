# Why Uber Engineering Switched from Postgres to MySQL

**Source:** https://www.uber.com/in/en/blog/postgres-to-mysql-migration/
**Author:** Evan Klitzke, Staff Software Engineer, Uber Engineering Core Infrastructure
**Published:** July 26, 2016
**Saved:** 2026-09-09
**Tags:** technology, infrastructure, fundamentals, technology

> Widely-cited engineering post. Based on Postgres 9.2 experience. Many of the specific bugs mentioned have been fixed in later Postgres versions, but the architectural differences (ctid-based indexes, WAL physical replication, process-per-connection) remain accurate descriptions of Postgres's design. This post is frequently cited and equally frequently misread — it describes Uber's specific situation at scale, not a universal recommendation.

---

## TL;DR
Uber migrated from Postgres to MySQL (via their Schemaless sharding layer built on InnoDB) due to five problems encountered at scale: write amplification from immutable tuple design, replication amplification from physical WAL, data corruption from a replication bug, inadequate replica MVCC, and painful major version upgrades requiring full downtime. MySQL's InnoDB avoids these via secondary indexes pointing to primary keys (not disk offsets), logical replication stream, true replica MVCC, and online version upgrades. The core architectural difference: Postgres uses physical (disk-offset) pointers everywhere; InnoDB uses logical (primary key) pointers for secondary indexes.

---

## The Core Architectural Difference

### Postgres: Physical Pointers (ctid)

Every row in Postgres is an immutable **tuple** identified by a **ctid** (physical disk offset). All indexes — primary AND secondary — point directly to ctids.

```
Row update in Postgres:
  Original row at ctid D (Muhammad, al-Khwārizmī, birth_year=780)
      ↓ update birth_year to 770
  NEW tuple written at ctid I (Muhammad, al-Khwārizmī, birth_year=770)
  Old tuple at D marked inactive (pending autovacuum)

  Result: ALL indexes must add an entry for ctid I:
    - Primary key index: id=4 → add entry for I
    - (first, last) index: Muhammad al-Khwārizmī → add entry for I
    - birth_year index: add 770→I, old 780→D stays until vacuum
```

**Secondary indexes point to ctids.** When a row moves (any update), EVERY index must be updated with the new ctid — even indexes over fields that didn't change.

### MySQL InnoDB: Logical Pointers (Primary Key)

InnoDB secondary indexes point to **primary key values**, not disk offsets. The primary key index holds the actual disk location.

```
Row update in InnoDB:
  Row with id=4 (Muhammad, al-Khwārizmī, birth_year=780)
      ↓ update birth_year to 770
  birth_year field updated IN PLACE (or old row copied to rollback segment)

  Result: ONLY the birth_year index needs updating
    - Primary key index: unchanged (id=4 still points to same row)
    - (first, last) index: unchanged (still points to primary key 4)
    - birth_year index: updated (new value → primary key 4)
```

**Secondary index lookup requires two hops** (secondary index → primary key → row). This is a slight read disadvantage vs. Postgres's one-hop. But writes are dramatically cheaper.

---

## Five Problems with Postgres at Uber's Scale

### Problem 1: Write Amplification

Updating a single field in Postgres requires writing:
1. New tuple to tablespace
2. Updated primary key index entry (new ctid)
3. Updated secondary index 1 entry (new ctid)
4. Updated secondary index N entry (new ctid)
5. All of the above written to WAL (for crash recovery)

A table with 12 secondary indexes requires 13 index updates per row update — even if only one field changed. In InnoDB: only the indexes covering changed fields get updated.

### Problem 2: Replication Amplification

Postgres replication is **physical** — the WAL contains actual disk operations ("at offset 8,382,491, write bytes XYZ"). Every physical write from Problem 1 propagates over the wire.

MySQL replication is **logical** — the binary log contains row-level changes ("change birth_year for row id=4 from 780 to 770"). Replicas infer and apply index changes locally.

For Uber's multi-datacenter setup (West Coast master → East Coast replicas):
- Postgres WAL bandwidth: enormous (write amplification × network)
- MySQL binlog bandwidth: compact (logical changes only)

The WAL archival bandwidth problem was particularly acute — during peak traffic, bandwidth to the storage service couldn't keep up with WAL write rate.

### Problem 3: Data Corruption

A Postgres 9.2 bug during master promotion caused replicas to misapply WAL records during timeline switches. Some rows that should have been marked inactive by the MVCC versioning mechanism were not, causing duplicate rows to be returned:

```sql
SELECT * FROM users WHERE id = 4;
-- Returns TWO rows: the old and new versions of al-Khwārizmī
```

Root causes of the severity:
- **Difficult to scope:** No easy way to count affected rows across all instances
- **Asymmetric corruption:** Different replicas had different rows corrupted (replica A: row X bad, row Y good; replica B: row X good, row Y bad)
- **WAL physical replication = total reset:** Fix required wiping and resyncing ALL replicas from a fresh master snapshot — extremely laborious
- **Systemic risk:** B-tree rebalancing at the physical level can corrupt large sections of an index if underlying data is wrong

In MySQL: replication happens at the logical layer. A bug can cause a statement to be skipped or applied twice (missing or duplicate data), but cannot corrupt index structure. B-tree rebalancing on replicas is an independent local operation.

### Problem 4: Replica MVCC

Postgres replicas apply WAL in a single thread. If an open read transaction on a replica conflicts with a WAL update that needs to apply to the same rows:
- Postgres **pauses WAL application** until the transaction closes
- If the transaction takes too long, Postgres **kills the transaction** (configurable timeout)
- Result: replicas routinely lag behind master; application queries are killed unexpectedly

MySQL replicas have **true MVCC** because replication is logical. Replica read transactions don't block WAL application. Long-running reads on replicas don't cause lag or killed queries.

### Problem 5: Major Version Upgrades

Postgres WAL is physical → incompatible between major versions. Cannot replicate from 9.2 master to 9.3 replica or vice versa.

**Upgrade procedure:**
1. Shut down master → no traffic served
2. Run pg_upgrade on master → many hours for large database
3. Restart master
4. Create full snapshot of master → more hours
5. Wipe each replica, restore from snapshot → stagger carefully
6. Bring each replica back, wait for replication catch-up

Uber upgraded from 9.1 → 9.2 successfully but took so long they never upgraded again. Legacy Postgres instances still on 9.2 by the time of this post (9.5 was current).

**MySQL upgrade procedure:**
- Apply update to one replica at a time
- When all replicas updated, promote one to new master
- Near-zero downtime, fully online

---

## InnoDB's Additional Advantages

### Buffer Pool

Postgres caches data via the Linux **page cache** — kernel-managed, requires system calls (lseek + read) for each access — context switches.

InnoDB implements its own **LRU buffer pool** in userspace:
- Custom LRU: can detect and prevent pathological access patterns from blowing out cache
- No context switches: worst case is a TLB miss (cheap)
- Result: more predictable, lower-latency cache behavior

Uber's largest Postgres replicas: 768 GB RAM, only ~25 GB actually used by Postgres (rest goes to page cache with the expensive access pattern).

### Connection Handling

| | Postgres | MySQL |
|--|---------|-------|
| Model | Process per connection (fork) | Thread per connection |
| Memory overhead | Higher (full process) | Lower (thread stack) |
| IPC mechanism | System V IPC (slower, context switch) | Futexes (userspace, fast) |
| Practical scale | ~100–200 active connections (pgbouncer required beyond this) | 10,000+ connections |

Uber used pgbouncer for Postgres connection pooling. Application bugs causing excess "idle in transaction" connections caused extended Postgres downtime on multiple occasions.

---

## The Schemaless Layer

Uber's production solution: a novel database sharding layer called **Schemaless** built on top of MySQL/InnoDB. Schemaless provides:
- Automatic horizontal sharding across MySQL instances
- Schema flexibility (document/column model) on top of MySQL's relational engine
- Consistent hashing for shard routing

The migration was from Postgres monolith → microservices + Schemaless on MySQL — not a simple Postgres → MySQL swap.

---

## What This Doesn't Mean

**This post is frequently misread as "Postgres is bad; use MySQL."** More accurately:

1. The specific Postgres bugs (9.2 replication bug) have been fixed
2. Postgres has improved its upgrade story with pglogical (9.4+) and logical replication built-in (10+)
3. The architectural differences (ctid-based write amplification, physical WAL) remain real — they matter more at very high write throughput with many secondary indexes
4. For most applications at normal scale, Postgres is excellent. Uber hit these limits at global scale with specific workload characteristics (monorepo data models with many secondary indexes, multi-datacenter replication across slow inter-DC links)
5. The connection handling limitation is real and pgbouncer is widely used to address it

The relevant question is: at what scale and with what workload do these architectural differences become the limiting factor?

---

## Questions & Gaps
- The 2016 publication date means logical replication built into Postgres (v10+) wasn't available. Would logical replication have addressed the WAL verbosity and upgrade path problems? Probably yes for both.
- The ctid write amplification problem — is this still a significant factor in Postgres 14/15/16, or have any architectural improvements reduced it?
- Schemaless details are in separate Uber posts — the sharding layer adds significant complexity. At what scale is a sharding layer worth the operational overhead vs. a single large Postgres instance with good connection pooling?
- The buffer pool advantage (InnoDB userspace LRU vs. Postgres page cache) — has Postgres addressed this with improvements to its shared buffer behavior in recent releases?

## Related Notes
- [Cursor Git at Scale — Continuity](https://github.com/LutherCalvinRiggs/research/blob/main/technology/infrastructure/cursor-git-at-scale-continuity.md) — Continuity's WAL-based design explicitly avoids the same write amplification problem by storing logical entries in S3 rather than physical disk operations. The architectural lesson is the same: physical pointers create write amplification; logical pointers don't.
- [How Complex Systems Fail — Cook 1998](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/how-complex-systems-fail-cook-1998.md) — The Postgres 9.2 replication bug is a textbook example of Cook's propositions 3 (catastrophe requires multiple failures — the bug + the timeline switch + the specific recovery procedure) and 14 (change introduces new failure modes — replication is the added mechanism that turned a local bug into a fleet-wide corruption problem).
- [System Design Playbook](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/system-design-playbook-neo-kim.md) — the ctid vs. primary-key pointer distinction is a concrete example of the "single source of truth" vs. "denormalized pointers everywhere" tradeoff in data model design. InnoDB's two-hop secondary index is more normalized; Postgres's ctid is denormalized (faster reads, expensive writes).
- [GitHub Outage Aug 2026](https://github.com/LutherCalvinRiggs/research/blob/main/technology/infrastructure/github-outage-aug-2026-cascade-failure.md) — GitHub's storage layer (Spokes) is built on top of Git repos on NVMe, not relational databases — but the same write amplification dynamic applies. Every Git push requires updating all indexes in the packfile, not just the ones that changed. Continuity's WAL addresses this at the Git layer just as MySQL's logical replication addresses it at the database layer.
