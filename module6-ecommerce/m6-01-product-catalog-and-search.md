# M6-01: Product Catalog & Search

## Estimation (Shopee scale)

- DAU: 30M (300M MAU × 10%)
- QPS avg: ~20K | Peak (11.11): ~200K (×10)
- Products: 900M × 2KB = 1.8TB (PostgreSQL)
- ES index: ×3 = ~5TB
- Images: 900M × 5 × 500KB = ~2PB (S3)

**Key insight:** Peak = 10× average → design for burst, not steady state.

## Architecture

```
Browser → CDN → Load Balancer → API Gateway
                                    ├── Search Svc → Elasticsearch
                                    └── Catalog Svc → PostgreSQL
                                              ↕ cache-aside
                                           Redis

PostgreSQL → Debezium (reads WAL) → Kafka "product.updated"
                                         → Index Sync Svc → Elasticsearch
                                                          → Redis (invalidate)
```

**Write path:** Seller → API GW → Catalog Svc → PostgreSQL → CDC → Kafka → Index Sync → ES

## Search vs Browse

| | Search | Browse |
|--|--|--|
| Input | Free keyword | Category path |
| Engine | Elasticsearch (full-text) | PostgreSQL + Redis (structured) |
| Use case | "giày nike size 42" | Click "Giày & Dép" category |

## Foundation Concepts

### Inverted Index (Elasticsearch)
```
"nike" → [Doc1, Doc3]
"giày" → [Doc1, Doc2]
```
O(1) lookup. Ranking via BM25 (term frequency + rarity).

### CDC via WAL (Debezium)
- PostgreSQL writes to WAL before applying to data files
- Debezium reads WAL as a replica, tracks LSN (Log Sequence Number)
- Crash-safe: restart → resume from last LSN, zero event loss
- Emit events: { op: "u/i/d", product_id, changed_fields }

### Index Sync Service
- Consume Kafka → update ES → invalidate Redis → commit offset (in this order)
- Commit offset last = idempotent retry safety

## Bottlenecks

### 1. ES overload at peak (200K QPS)
- **Cache search results in Redis** (TTL 60s) — top 1K queries = 80% traffic
- **ES sharding:** 10 shards × 3 replicas → 3× read throughput
- **Pre-warm cache** 1h before flash sale

### 2. Stale search results after price update
- Problem: Redis caches full search result (including price) for 60s
- Can't invalidate by product_id — cache key is query string

**Solutions:**
- **Option A:** Short TTL (5s) — lower hit rate, higher ES load
- **Option B:** Separate price from ES index — ES returns product_ids only, Catalog Svc batch-fetches price from Redis/DB
  - Trade-off: extra round trip (+5ms), N+1 risk → must use `WHERE id IN (...)` batch query

**Amazon/Shopee use Option B.**

### 3. Cache Stampede (Thundering Herd)
Cache expire → 10K concurrent misses → DB dies.

- **Mutex lock:** `SET lock:key EX 5 NX` — only 1 thread queries DB, others wait
- **Probabilistic Early Expiration:** refresh cache before TTL expires
- **Jitter TTL:** `TTL = 60 + random(0, 10)` — stagger expiry across keys

## Why These Decisions

| Decision | Why | Why not alternative |
|--|--|--|
| Elasticsearch | Scale (900M), typo tolerance, BM25 | PG FTS: no typo/synonym, hard to shard |
| Kafka (not direct ES write) | Decoupled, retry on ES failure, replay | Direct write: tight coupling, data loss on failure |
| PostgreSQL + JSONB | ACID + flexible schema via JSONB column | MongoDB: complex ops, weaker transactions |

## Key Patterns

- **CQRS:** Write → PostgreSQL, Read → ES + Redis. Async sync via CDC.
- **Cache-Aside:** App manages cache manually (read → miss → DB → cache).
- **Fan-out on Read:** Browse category queries at request time (no precompute).
- **Cache Stampede:** Mutex + Jitter TTL + Pre-warm.

## Interview Angle (45 min)

```
0-5   Clarify: search + browse + detail? Scale?
5-10  Estimation: QPS, storage, peak vs avg
10-20 High-level diagram (above)
20-35 Deep dive: "How does search work?" → inverted index, sharding, cache
35-42 Bottlenecks: thundering herd, stale price → proactively raise
42-45 Wrap up: "Next I'd cover ranking + personalized search"
```

**Key phrase:**
> "Search and Browse are separate services — CQRS pattern. Different access patterns: full-text vs structured query."
