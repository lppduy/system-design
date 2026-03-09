# M0-02: Scaling Reads vs Writes

## Why it matters

Almost every system design splits on this axis. Know the read/write ratio first — it drives your entire architecture.

---

## Estimation

Example: Twitter-like feed
- 100M DAU
- 20 reads/user/day → **Read QPS = 20,000/s**
- 2 writes/user/day → **Write QPS = 2,000/s**
- Read/write ratio = **10:1** — reads dominate

---

## Scaling Reads → Read Replicas

Single master can't serve all reads. Add read replicas — nodes that mirror the master and serve read traffic.

```
App → Master (writes)
    ↘ Replica 1 (reads)
    ↘ Replica 2 (reads)
    ↘ Replica 3 (reads)
```

Replicas stay in sync by replaying the master's **WAL (Write-Ahead Log)**.

---

## WAL (Write-Ahead Log)

Before writing data to disk, the DB first records the change in a log file. This is the WAL.

- Crash recovery: replay the log on restart
- Replication: ship the log to replicas so they apply the same changes

---

## Replication Lag

WAL shipping takes time → replicas are slightly behind master.

**Problem:** User posts a tweet, refreshes feed — doesn't see their own tweet (read went to a lagging replica).

**Fix: Read-your-own-writes consistency**
After a user writes, route their reads to master for a short window (~1-2s). Everyone else reads from replicas.

---

## Replication Fan-out Problem

10 replicas = master streams WAL to 10 destinations simultaneously. Burns master's network I/O and CPU.

**Fix: Cascading replication**
```
Master → Replica 1
              ├── Replica 2
              ├── Replica 3
              └── Replica 4
```
Master fans out to 1 node. That node takes the distribution burden.

**Tradeoff:** More replication lag (data travels two hops).

---

## Scaling Writes → Sharding

Read replicas are read-only — they don't help with write throughput.

**Sharding:** Split data across multiple masters. Each shard owns a subset of data.

```
Shard 1: user_id 0–10M   → Master A
Shard 2: user_id 10M–20M → Master B
Shard 3: user_id 20M–30M → Master C
```

No conflicts — each record lives on exactly one master.

---

## Good Shard Keys

Requirements:
1. **High cardinality** — many distinct values
2. **Uniformly accessed** — no recency bias or hotspots

| Shard Key | Good? | Why |
|-----------|-------|-----|
| user_id | ✅ | High cardinality, random access |
| created_at | ❌ | All new writes hit the latest shard |
| country | ❌ | Uneven distribution (US = 80% traffic) |

---

## Why Multi-Master is Hard

Multiple masters accepting writes on the same data → write conflicts.

**Locking across masters = network round trips:**
```
Master 1 wants to write → asks Master 2 for lock → waits → writes → releases
```
Each hop ~1ms. At 20,000 writes/s this kills throughput.

**Conflict resolution strategies:**
- **Last Write Wins (LWW)** — highest timestamp wins. Simple, but silently loses data (clocks drift)
- **CRDTs** — data structures that merge automatically (counters, sets). Used in Cassandra, Riak
- **Application-level resolution** — app decides (e.g. Google Docs merges edits)

Sharding avoids the problem entirely — two masters never own the same data.

---

## Don't Shard Prematurely

Sharding adds massive complexity:
- Cross-shard queries require scatter-gather across all shards
- Joins across shards are painful or impossible
- Re-sharding when you outgrow shard count is expensive

**Right approach:** Start with single master + read replicas. Monitor. Shard only when writes become the actual bottleneck.

---

## Summary

| Problem | Solution | Tradeoff |
|---------|----------|----------|
| Read heavy | Read replicas | Replication lag |
| Replication lag | Read-your-own-writes routing | More master load |
| Master fan-out | Cascading replication | Deeper lag |
| Write heavy | Sharding | Complex queries, hotspots |
| Hotspots | Good shard key (user_id) | May break time ordering |
