# M5-08: MongoDB Internals

## Why MongoDB (vs PostgreSQL)

| Choose MongoDB | Choose PostgreSQL |
|---|---|
| Schema varies per document | Schema well-defined and stable |
| Hierarchical/nested data | Relational data with cross-references |
| Rapid iteration, schema evolving | Strong consistency, ACID transactions |
| Horizontal scaling needed early | Complex queries, JOINs, aggregations |
| Document-per-read pattern | Multi-table transactions |

**Pain solved:** store documents as-is, no schema migrations, no JOINs for nested data. Each document can have different fields.

**Convergence:** MongoDB has multi-document ACID since 4.0. PostgreSQL has JSONB since 9.4. Each still best at what it was designed for.

---

## Document Model

### BSON (Binary JSON)
- Binary encoding of JSON — faster to parse than text JSON
- Adds native types: `Date`, `ObjectId`, `Decimal128`, `Binary`, `int32/int64`
- Length prefixes → skip fields without parsing
- Tradeoff: slightly larger on disk, much faster to traverse

### ObjectId (`_id`)
```
4 bytes timestamp + 5 bytes random + 3 bytes counter
```
- Globally unique without coordination (no central ID service)
- Roughly time-ordered (first 4 bytes = epoch seconds)

---

## WiredTiger Storage Engine

Default since MongoDB 3.2.

### Write Path
1. Write to in-memory buffer (WT cache)
2. Write to journal (WAL equivalent) → durability
3. Periodically checkpoint (flush dirty pages to disk)

### Read Path
1. Check WT cache → if hit, return
2. If miss → read from disk (B-tree pages)

### Document-Level Locking
- Optimistic concurrency control
- Two writes to different documents = no contention
- Two writes to same document = one waits
- Similar to PostgreSQL row-level locks

### Compression (on disk, decompressed in cache)
- **Snappy** (default) — fast, moderate compression
- **Zlib** — better compression, slower
- **Zstd** — best balance (added 4.2)
- Typically ~70% disk savings

### vs PostgreSQL's Storage
- PostgreSQL MVCC creates dead tuples → needs VACUUM
- WiredTiger does in-place updates (copy-on-write at page level) → no dead tuples, no vacuum
- Tradeoff: PG gives better read consistency (true MVCC snapshots), WT gives less maintenance

---

## Indexing

### Default: `_id` index on every collection (cannot remove)

### Secondary Indexes
```javascript
db.users.createIndex({ email: 1 })            // single field
db.users.createIndex({ city: 1, age: -1 })    // compound
db.users.createIndex({ bio: "text" })          // text search
db.users.createIndex({ location: "2dsphere" }) // geospatial
```

### Compound Index Rule
Leftmost prefix must match (same as PostgreSQL composite indexes).
```
Index: { city: 1, age: -1 }
Query { city: "Hanoi", age: { $gt: 20 } }  → uses index ✓
Query { age: 28 }                           → cannot use   ✗
```

### Covered Queries
All query + projection fields in the index → results from index only, zero document reads.

---

## Replica Sets

```
PRIMARY ──oplog──▶ SECONDARY
   │                    │
   └──oplog──▶ SECONDARY
```

### How it works
1. All writes go to Primary
2. Primary records writes in **oplog** (capped collection)
3. Secondaries tail oplog and replay operations
4. Primary dies → majority election → new Primary (~10-12s)

### Read Preferences

| Mode | Behavior | Risk |
|------|----------|------|
| `primary` (default) | Reads from primary | Safe, consistent |
| `primaryPreferred` | Primary if available, else secondary | Stale on failover |
| `secondary` | Always secondary | Stale reads |
| `nearest` | Lowest latency node | May be stale |

### Read-Your-Own-Writes Problem
`secondary` read preference → user updates profile → refreshes → sees old version (replication lag).
**Fix:** Use `primary` for user-facing reads after writes, or use causal consistency sessions (3.6+).

---

## Sharding

```
mongos (router, stateless) → Config Servers (metadata, 3-node RS) → Shards (each a replica set)
```

### Shard Key — Most Critical Decision

| Strategy | Pros | Cons |
|---|---|---|
| Hashed (`{ _id: "hashed" }`) | Even distribution | Range queries scatter to all shards |
| Range (`{ timestamp: 1 }`) | Range queries efficient | Hot shard (recent writes hit one shard) |
| Compound (`{ tenant_id: 1, _id: 1 }`) | Good locality + distribution | Must include shard key in queries |

**Good shard key:** high cardinality, low frequency per value, doesn't change.

### Chunks and Balancing
- Data split into chunks (~128MB default)
- Balancer migrates chunks between shards for even distribution
- Migration is expensive — brief lock during final phase

---

## Write Concern

| Setting | Meaning |
|---------|---------|
| `w: 1` | Ack after primary writes (default). Data loss possible if primary crashes |
| `w: "majority"` | Ack after majority confirms. Safe. |
| `j: true` | Wait for journal flush before ack |
| `wtimeout` | Fail if write concern not met within N ms |

`w:1, j:false` → fastest, least safe. `w:"majority", j:true` → slowest, safest.

---

## Aggregation Pipeline

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },        // WHERE
  { $group: { _id: "$city", total: { $sum: "$amount" } } },  // GROUP BY
  { $sort: { total: -1 } },                   // ORDER BY
  { $limit: 10 }                              // LIMIT
])
```

`$lookup` = LEFT JOIN equivalent. Works but slower than PG JOIN. Many `$lookup`s = probably should use PostgreSQL.

---

## Interview Angle

1. **When and why** — flexible schema, embedded documents avoid JOINs, horizontal scaling
2. **WiredTiger** — B-tree indexes, document-level locking, compression, journal for durability
3. **Replica sets** — oplog replication, majority elections, read preferences and tradeoffs
4. **Sharding** — shard key selection critical, hot shard problem, mongos routing, chunk balancing
5. **Write concern** — tunable durability (w:1 vs majority, journal)
6. **When NOT MongoDB** — heavy JOINs, complex relational integrity, strong cross-entity consistency
