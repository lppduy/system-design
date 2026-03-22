# M2-06 Key-Value Store

## Estimation
- 100M DAU, 50 ops/user/day → 58K QPS (peak 175K)
- 70% read, 30% write
- Storage: 2 TB (6 TB with 3x replication)
- Bandwidth: read 40 MB/s, write 18 MB/s
- Constraint: distributed system needed — single node can't handle peak QPS

## Requirements
**Functional:** put(key, value), get(key), delete(key)
**Non-functional:** High availability, scalability, low latency (single-digit ms), tunable consistency, automatic partitioning

## High-Level Architecture
```
Client → Coordinator (any node) → Partition Map (consistent hashing) → Storage Nodes (N replicas)
```
- Coordinator: receives request, determines which node owns key via hash ring
- Storage Nodes: each holds a partition + replicas of other partitions
- Decentralized: every node can be coordinator (no single point of failure)

## 3 Core Problems

### 1. Data Placement → Consistent Hashing
- hash(key) → position on ring → next node clockwise owns it
- Virtual nodes for load balancing
- Add/remove node only affects neighbors (not full reshuffle)

### 2. Fault Tolerance → Replication + Quorum
- Each key replicated on N consecutive nodes on ring (typically N=3)
- **Quorum parameters:**
  - N = total replicas
  - W = nodes that must ACK a write
  - R = nodes that must respond to a read
- **Rule:** W + R > N → strong consistency (guaranteed overlap between write set and read set)

| Config | W | R | Characteristic |
|--------|---|---|----------------|
| Strong | 2 | 2 | Slow but always correct |
| Fast read | 1 | 1 | Fast but may read stale |
| Fast write | 1 | 2 | Write fast, read correct |

### 3. Concurrent Writes → Conflict Resolution
**Last-Write-Wins (LWW):** Timestamp-based, simple, but loses data. Used by DynamoDB.
**Vector Clock:** Per-node counter, detects true conflicts, returns both versions to client for merge. Used by Dynamo (shopping cart — never lose items).

## Write Path (LSM Tree)
```
put(key, value)
  → ① WAL (disk) — crash recovery, append-only
  → ② MemTable (RAM) — sorted in-memory tree
  → ③ SSTable (disk) — flushed when MemTable full (~64MB), sorted file
```
- Why not write directly to disk? Random I/O slow. LSM converts random → sequential writes (100x faster).
- **Compaction:** Merge overlapping SSTables, discard old versions. Size-tiered (write-optimized) vs Leveled (read-optimized).

## Read Path
```
get(key)
  → ① Check MemTable (RAM) → found? return
  → ② Bloom Filter per SSTable → skip SSTables that definitely don't have key
  → ③ Read matching SSTable from disk → return newest version
```
- Bloom Filter: probabilistic, O(1), ~10x less RAM than HashSet. False positive OK (extra disk read), zero false negative.

## Failure Handling

| Scenario | Solution |
|----------|----------|
| Temporary failure | **Hinted handoff** — neighbor holds data temporarily, sends back when node recovers |
| Permanent failure | **Merkle tree** — hash tree comparison, sync only differing data |
| Detecting failures | **Gossip protocol** — random peer heartbeat exchange, decentralized, no single point of failure |

## Why These Decisions?

| Decision | Why | If different? |
|----------|-----|---------------|
| Consistent hashing | Add/remove node affects only neighbors | Mod hashing: reshuffle everything |
| Quorum W+R>N | Tunable consistency per request | Fixed strong: slow. Fixed eventual: incorrect |
| LSM Tree | Fast sequential writes | B-Tree: fast reads but slow writes |
| Gossip | No SPOF for failure detection | Central monitor: dies = blind |
| Vector clock | No data loss on conflicts | LWW: simple but loses writes |

## Concepts
- **Consistent Hashing** — key → node mapping on ring, virtual nodes for balance
- **Quorum (N/W/R)** — tunable consistency: W+R>N = strong
- **Vector Clock** — per-node counters, detect concurrent writes without data loss
- **LSM Tree** — WAL → MemTable → SSTable, sequential write optimization
- **Bloom Filter** — probabilistic set membership, skip unnecessary disk reads
- **Gossip Protocol** — decentralized failure detection via random peer exchange
- **Hinted Handoff** — neighbor temporarily holds data for downed node
- **Merkle Tree** — hash tree for efficient data sync between replicas

## Interview Angle (45 min)
```
Min 1-5:   Requirements + Estimation
Min 5-10:  High-level: client → coordinator → nodes (consistent hashing)
Min 10-15: Partitioning: consistent hashing + virtual nodes
Min 15-25: Replication + Quorum (N/W/R) — CORE section
Min 25-35: Write/Read path (LSM tree, bloom filter)
Min 35-40: Failure handling (gossip, hinted handoff, merkle tree)
Min 40-45: Conflict resolution (vector clock vs LWW), tradeoffs
```
Interviewers drill into: quorum tradeoffs, conflict resolution.
