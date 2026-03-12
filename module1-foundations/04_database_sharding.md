# Database Sharding

## The Pain
Replication solves read scaling + availability. But when writes bottleneck or data outgrows a single node, replication can't help — all writes still go to one primary. Sharding splits data across multiple DB instances.

## What Is Sharding
Horizontal partitioning across multiple database instances. Each shard holds a subset; together they hold the complete dataset.

```
Single DB:  [100M users, 500M orders] — one machine

Sharded:
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Shard 0  │ │ Shard 1  │ │ Shard 2  │ │ Shard 3  │
│ 25M each │ │ 25M each │ │ 25M each │ │ 25M each │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

## Shard Key Selection (Most Critical Decision)

| Strategy | How | Good when | Bad when |
|----------|-----|-----------|----------|
| Range-based | `user_id 1-1M → shard 0` | Sequential scans needed | Hot shard on latest range |
| Hash-based | `hash(key) % N` | Even distribution | Range queries → scatter-gather |
| Directory-based | Lookup table maps key → shard | Flexible rebalancing | Lookup table = SPOF |

### E-commerce Example: Shard Key for `orders`
- **`order_id`** — sequential IDs hot-shard the latest range; user queries scatter across all shards
- **`user_id`** — best choice: "my orders" hits single shard (hot path); merchant queries scatter but that's cold path (acceptable)
- **`merchant_id`** — top sellers create hot shards (Amazon-scale merchants crush one shard)

**Rule: Optimize shard key for the hot-path query. Accept scatter-gather on cold paths.**

## Routing Layer

```
Client → App Server → Routing Layer → Shard N
                          │
                    ├── Application-level (code knows formula)
                    ├── Proxy layer (Vitess, ProxySQL)
                    └── DB-native (Citus for Postgres)
```

Hash routing: `shard = hash(user_id) % num_shards`

## The Hard Problems

| Problem | Pain |
|---------|------|
| Cross-shard queries | Can't JOIN across shards at DB level |
| Cross-shard transactions | No simple BEGIN/COMMIT across shards |
| Resharding | `hash % 4` → `hash % 5` moves almost every row. Consistent hashing minimizes this |
| Global ordering | `ORDER BY created_at LIMIT 10` → scatter-gather all shards, merge in app |
| Pagination | `OFFSET 10000` — each shard returns 10,010 rows. Brutal at scale |
| Schema changes | ALTER TABLE on every shard, coordinated |

## Scatter-Gather

```
"Top 10 recent orders globally":
  → Shard 0: LIMIT 10
  → Shard 1: LIMIT 10
  → Shard 2: LIMIT 10
  → Shard 3: LIMIT 10
App merges 40 rows → returns top 10
```

4 shards = fine. 400 shards = 400 parallel queries, latency = slowest shard.

### Avoiding Scatter-Gather: Precompute / CDC

```
Shard 0 ──┐
Shard 1 ──┤── CDC stream ──→ "recent_orders" (unsharded table or Redis sorted set)
Shard 2 ──┤                   keeps only last N orders
Shard 3 ──┘
```

Tradeoff: eventual consistency (1-2s delay). Fine for dashboards, not for trading.

## When NOT to Shard (Exhaust These First)

| Alternative | Why |
|------------|-----|
| Vertical scaling | 64 cores, 512GB RAM handles more than people think |
| Read replicas | If reads are the bottleneck, 10x simpler |
| Table partitioning | Postgres native — same DB, data split by range/hash |
| Archiving old data | Move 2+ year old data to cold storage |
| Caching | Redis eliminates most read load |

**Rule of thumb:** Don't shard until millions of writes/day and single machine genuinely can't keep up.

## Interview Angle (45 min)
1. Don't jump to sharding — show you know simpler alternatives
2. State the bottleneck clearly — "writes exceed single primary capacity"
3. Pick shard key with reasoning — explain the hot-path query
4. Acknowledge tradeoffs — cross-shard joins, resharding
5. Mention consistent hashing — shows you know the resharding solution
