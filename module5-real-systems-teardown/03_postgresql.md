# PostgreSQL Internals

## MVCC (Multi-Version Concurrency Control)

Readers never block writers, writers never block readers. Each transaction sees a snapshot.

**How it works:**
- UPDATE doesn't modify in place → marks old row dead (`xmax`), inserts new version (`xmin`)
- Each row has hidden columns: `xmin` (creating txid), `xmax` (deleting txid)
- Transaction sees only rows where `xmin < my_txid` and `xmax = 0 or xmax > my_txid`

**Consequence:** Dead tuples pile up → need garbage collection.

## VACUUM

Cleans up dead tuples left by MVCC.

| Type | What it does | Locks? |
|------|-------------|--------|
| VACUUM | Marks dead tuple space as reusable (doesn't shrink file) | No |
| VACUUM FULL | Rewrites entire table, shrinks file | Exclusive lock (blocks all) |
| autovacuum | Background process runs regular VACUUM automatically | No |

**Library analogy:** UPDATE = put new copy on shelf, sticker old one "do not read." VACUUM = librarian removes stickered books, marks shelf space available. VACUUM FULL = moves all books to new shelves, closes library during move.

**Critical risk:** Transaction ID wraparound. PostgreSQL has ~2 billion txid counter. VACUUM must reset it. If autovacuum can't keep up → PostgreSQL **shuts down** to prevent corruption. Most dangerous PG failure mode.

**Table bloat signs:** `pg_stat_user_tables.n_dead_tup` growing, queries slowing, table size much larger than data size.

## WAL (Write-Ahead Log)

Every change written to WAL first, then acknowledged, then lazily flushed to data files.

```
Client: UPDATE → WAL write (sequential, fast) → ACK to client → checkpoint (later)
Crash? → Replay WAL on restart → data recovered
```

**Why sequential:** Same principle as Kafka — sequential I/O is fast (no seeking).

**Checkpoint:** Periodically flushes dirty pages from memory to disk. Controlled by `checkpoint_timeout` and `max_wal_size`.

## Replication

Primary streams WAL records to replicas:

```
Primary → WAL records → network → Replica (replays WAL)
```

| Mode | Behavior | Data loss risk |
|------|----------|---------------|
| Async (default) | Primary doesn't wait for replica ACK | Small window |
| Sync | Primary waits for replica ACK before commit | Zero |

Same tradeoff as Kafka `acks=1` vs `acks=all`.

**Physical replication:** Raw WAL bytes. Exact binary copy. Can't replicate selectively.
**Logical replication:** Decoded changes (INSERT/UPDATE/DELETE). Can replicate specific tables, cross-version, feed CDC (Debezium → Kafka).

## Indexes

### B-Tree (default)
- Balanced tree → O(log N) lookup
- Leaf nodes point to heap tuples (rows on disk)
- Supports equality, range, sorting
- **Index-only scan:** If all needed columns in index, skip table read entirely

### Composite Index Column Order
**Rule:** Equality columns first, range columns after.

```sql
-- Query: WHERE status = 'active' AND created_at > '2026-01-01'

-- Good: status (equality) first, then created_at (range)
CREATE INDEX idx_status_created ON users(status, created_at);

-- Bad: created_at first → finds millions of rows, then filters status
CREATE INDEX idx_created_status ON users(created_at, status);
```

**Leftmost prefix rule:** Index on `(A, B, C)` can serve queries on `(A)`, `(A, B)`, `(A, B, C)` — but NOT `(B)` or `(C)` alone.

### Other Index Types
| Type | Use case |
|------|----------|
| Hash | Equality only (rarely used) |
| GIN | Full-text search, arrays, JSONB |
| GiST | Geometry, range types |
| BRIN | Large naturally-ordered tables (time-series) |

## Connection Model

PostgreSQL forks a **new process per connection** (not threads like MySQL).

- Heavy: each process ~5-10MB RAM
- Too many connections = memory exhaustion + context switching
- **Solution:** Connection pooler (PgBouncer, PgPool) — sits between app and PG, reuses connections
- Typical: app opens 100 connections to PgBouncer, PgBouncer uses 20 actual PG connections

## EXPLAIN ANALYZE

The most important debugging tool:

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@test.com';
```

Shows: Seq Scan vs Index Scan, rows estimated vs actual, execution time, buffer hits vs disk reads.

**Key things to look for:**
- `Seq Scan` on large table = missing index
- `Rows Removed by Filter` high = index not selective enough
- `actual rows` >> `estimated rows` = stale statistics → run `ANALYZE`

## Interview Angles

- "MVCC tradeoff?" → No read locks (fast), but dead tuples accumulate (need VACUUM)
- "WAL purpose?" → Crash recovery + replication foundation. Sequential writes = fast.
- "Index not helping?" → Check column order (equality first, range after), check selectivity, run EXPLAIN ANALYZE
- "Connection scaling?" → Process-per-connection is heavy. Use PgBouncer.
- "Sync vs async replication?" → Same as Kafka acks. Sync = safe + slow. Async = fast + small loss window.
