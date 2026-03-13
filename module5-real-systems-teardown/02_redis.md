# Redis Internals

## Why Redis Is Fast

### 1. In-Memory
All data in RAM. Reads/writes in nanoseconds, no disk I/O.

### 2. Single-Threaded Event Loop
- One thread executes ALL commands sequentially
- No locks, no context switching, no thread contention
- Each command takes microseconds of CPU → 100K+ ops/sec on one core
- Uses I/O multiplexing (epoll/kqueue) to monitor thousands of sockets with one thread
- Since Redis 6.0: I/O threads for network parsing/response, but command execution still single-threaded

**I/O multiplexing analogy:** One waiter checks all tables: "Table 3 needs to order, table 7 needs the bill." Efficient as long as each request is quick.

**Danger:** Slow commands (`KEYS *`) block everything. Use `SCAN` instead (cursor-based, non-blocking batches).

### 3. Efficient Data Structures
Each type uses compact encoding for small data, switches to full structure when data grows:

| Type | Small (< threshold) | Large |
|------|---------------------|-------|
| String | int or embstr | raw SDS |
| List | listpack | quicklist |
| Hash | listpack | hashtable |
| Set | listpack or intset | hashtable |
| Sorted Set | listpack | skiplist + hashtable |

**Listpack:** flat contiguous byte array. O(N) lookup but CPU-cache-friendly. Beats hashtable O(1) for small N because no pointer chasing.

```
hash-max-listpack-entries 128    ← switch to hashtable after 128 fields
hash-max-listpack-value 64      ← switch if any value > 64 bytes
```

## Persistence

### RDB (Snapshot)
- Point-in-time binary snapshot to `.rdb` file
- `fork()` + copy-on-write: child writes snapshot, parent keeps serving
- **COW memory spike:** 10GB data + 5GB modified during snapshot = ~15GB peak. Provision 2x RAM.
- Good for: backups, disaster recovery, fast restarts

### AOF (Append-Only File)
- Logs every write command
- `appendfsync everysec` (recommended) — lose max 1 second on crash
- `appendfsync always` — safest but slowest (fsync every command)
- AOF rewrite: compacts file (1000 INCRs → one SET)
- Good for: minimal data loss

### Production: Use Both
RDB for backups + fast restarts. AOF for minimal data loss. Redis loads AOF on restart.

## Eviction Policies

When `maxmemory` reached:

| Policy | Behavior |
|--------|----------|
| `noeviction` | Return error on writes (default) |
| `allkeys-lru` | Evict least recently used (most common) |
| `volatile-lru` | LRU only among keys with TTL |
| `allkeys-lfu` | Evict least frequently used |
| `allkeys-random` | Random eviction |
| `volatile-ttl` | Evict keys closest to expiring |

**Cache hit rate drop causes:** maxmemory too small for working set, traffic pattern change, cache stampede (popular key expires), Redis restart (cold cache).

## Replication

```
Master (read + write)
  ├── Replica 1 (read-only)
  ├── Replica 2 (read-only)
  └── Replica 3 (read-only)
```

- Asynchronous replication — small data loss window if master crashes
- **Sentinel:** monitors cluster, auto-promotes replica if master dies
- **Redis Cluster:** sharding via 16384 hash slots across multiple masters

## Common Use Cases

| Pattern | Why Redis | Key Commands |
|---------|-----------|-------------|
| Session store | Fast R/W, TTL for expiry | SET, GET, EXPIRE |
| Cache | Sub-ms reads, LRU eviction | GET, SET, DEL |
| Rate limiter | Atomic increment + expiry | INCR, EXPIRE |
| Leaderboard | Sorted Set, O(log N) | ZADD, ZRANGE, ZRANK |
| Pub/Sub | Real-time messaging | PUBLISH, SUBSCRIBE |
| Distributed lock | Atomic set-if-not-exists | SET key val NX EX 30 |
| Queue | List as queue (but Kafka better) | LPUSH, BRPOP |

## KEYS vs SCAN

- `KEYS pattern` — scans ALL keys, blocks server. Banned in production.
- `SCAN cursor MATCH pattern COUNT 100` — cursor-based, non-blocking batches.

## Interview Angles

- "Why single-threaded?" → No locks, no contention. Bottleneck is network, not CPU. Commands are microseconds.
- "What if one command is slow?" → Blocks everything. Avoid O(N) commands on large datasets.
- "RDB vs AOF?" → RDB = fast recovery, minutes of data loss. AOF = slow recovery, seconds of data loss. Use both.
- "How to scale writes?" → Redis Cluster (sharding). Single master can't scale writes.
- "COW during fork?" → Memory can double. Provision 2x RAM.
