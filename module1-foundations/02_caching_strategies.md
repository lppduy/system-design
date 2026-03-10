# M1-02 Caching Strategies

## The Pain

Read-heavy workload (e.g. 10k req/sec, same 100 products). DB query = 5ms each.
Without cache: 10,000 × 5ms = 50 CPU-seconds/sec on one machine → falls over.
Cache cuts DB load to a fraction of that.

---

## Core Terms

| Term | Definition |
|------|-----------|
| Cache hit | Data found in cache — fast, DB not touched |
| Cache miss | Data not in cache — fall back to DB, then store in cache |
| Hit rate | % of requests served from cache. 99% = only 1% hit DB |
| TTL | Time To Live — how long an entry lives before expiry |
| Eviction | Removing entries when cache is full. LRU (least recently used) is most common |
| Cache invalidation | Explicitly removing/updating stale cache entries |

---

## Read Strategies

### Cache-aside (lazy loading) — most common
```
1. Check cache
2. Miss → query DB
3. Store result in cache with TTL
4. Return data

Next request → cache hit, DB never touched
```

**Problem:** Cold start / cache stampede — on startup, all requests miss at once and slam DB.

**Mitigation:**
- Cache warming — pre-populate before traffic hits
- TTL jitter — TTL = base ± random(0, N) so entries don't expire simultaneously
- Single-flight / locking — only one request fetches from DB, rest wait

---

## Write Strategies

| Strategy | Flow | Tradeoff |
|----------|------|----------|
| Write-through | Write DB + write cache | Always fresh. Double write latency. May cache unread data |
| Write-around | Write DB + delete cache key | One miss after write. Avoids concurrency bugs. Most common |
| Write-back | Write cache only, async flush to DB | Fast writes. Risk: data loss if cache crashes |

**Why delete instead of update on write?**
Concurrent writes can race — one overwrites the other in cache with a stale value.
Delete forces next read to fetch correct value from DB.

**Choose based on workload:**
- Read-heavy → write-through or write-around
- Write-heavy + loss tolerable → write-back

---

## Failure Modes

### Cache stampede
Many requests for the **same key** at the same time (e.g. TTL expires on popular key).
Fix: TTL jitter, single-flight pattern.

### Thundering herd
Variant of stampede — mass expiry causes burst of DB requests.
Fix: staggered TTLs.

### Cache avalanche
Entire cache goes down (node crash, mass expiry). All traffic hits DB simultaneously.
Fix: Redis Sentinel/Cluster, circuit breaker, serve stale data.

---

## Distributing Cache (Redis Cluster)

**Naive approach — modulo hashing:**
```
node = hash(key) % N
```
Problem: if N changes (node added/removed), almost all keys remap → instant avalanche.

**Consistent hashing:**
- Place nodes and keys on a hash ring (0 → 2^32)
- Key → stored on first node clockwise
- Node removed → only that node's keys move to next node (~1/N of keys, not all)
- Node added → only keys between new node and predecessor move

**Virtual nodes:** each physical node occupies multiple positions on the ring → even distribution, prevents hotspots.

Used by: Redis Cluster, Cassandra, DynamoDB.

---

## Decision Framework

| Decision | Depends on |
|----------|-----------|
| Read strategy | Almost always cache-aside |
| Write strategy | Read/write ratio + data loss tolerance |
| Failure mitigation | Traffic pattern + SLA |
| Distribution | Number of nodes + rebalancing cost |

**Always derive from numbers, not instinct.**
