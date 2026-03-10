# M1-03 Database Replication

## The Pain

Single PostgreSQL instance: 5,000 reads/sec + 500 writes/sec.
- Load: one machine handling everything
- SPOF: it dies, entire app is down

Solution: primary-replica replication.

---

## Architecture

```
Writes → Primary only
Reads  → Replicas (distributed load)
```

Primary dies → promote a replica → no SPOF.

---

## How Data Moves: WAL Streaming

WAL (Write-Ahead Log) — every change written to WAL before touching data files.
Replication = stream WAL to replicas → replicas replay it → identical data, slightly behind.

**Replication lag** = time between primary writing and replica applying. Usually milliseconds, but meaningful.

---

## Replication Lag Problem: Read-Your-Own-Writes

```
1. User updates profile picture → write to primary → success
2. Redirect to profile page → read from replica
3. Replica hasn't caught up → user sees old picture
```

**Fixes:**
- Route write + immediate next read to primary
- Always read sensitive data (payments, profile) from primary
- Synchronous replication for critical data
- Client tracks WAL position, replica waits until it catches up

---

## Async vs Sync Replication

| | Async (default) | Sync |
|--|--|--|
| Write speed | Fast | Slow (waits for replica ACK) |
| Consistency | Eventual | Strong |
| Data loss risk | Yes (if primary crashes before replication) | No |
| Availability | High | Blocked if replica is down |

**Semi-sync** — primary waits for at least 1 replica to confirm, falls back to async on timeout. Best of both worlds. Used in MySQL.

---

## Data Loss Scenario (Async)

```
t=0  Write: "$500 transfer" → primary WAL position 1000
t=1  Primary returns success to client
t=2  Streaming to replica... (at position 995)
t=3  PRIMARY CRASHES
t=4  Replica promoted — WAL position 995
     Writes at 996-1000 are lost forever
```

---

## Failover

**Manual:** ops promotes replica. Minutes of downtime. Unacceptable.

**Automatic (Patroni + etcd):**
1. Monitor detects primary failure (health check timeout ~10-30s)
2. Picks replica with highest WAL position (least data loss)
3. Promotes it, updates distributed config (etcd)
4. Proxy (PgBouncer/HAProxy) reroutes traffic to new primary

App connects via virtual IP or proxy — never directly to primary. No connection string change on failover.

---

## Split-Brain Problem

Primary becomes slow (not dead) → failover triggers → now TWO primaries accepting writes → data corruption.

**Fix: Fencing**
- **STONITH** — forcibly power off old primary before promoting replica
- **Lease-based** — primary holds a lease from etcd. Can't renew → stops writes. New primary takes lease. Only one node holds leader lock.

---

## Promotion: Which Replica?

Promote the one **furthest ahead in WAL** (least replication lag = least data loss).

```
Replica A: WAL position 998  ← promote this
Replica B: WAL position 995
```

---

## The 3 Core Tradeoffs

| Tradeoff | Problem | Fix |
|----------|---------|-----|
| Read scaling vs stale data | Replication lag → stale reads | Route sensitive reads to primary |
| Async → data loss | Unconfirmed writes lost on crash | Semi-sync replication |
| Failover complexity | Downtime + split-brain risk | Patroni + etcd + fencing + proxy layer |

---

## Why Replication Isn't Enough

Primary is still the bottleneck for writes. 500 writes/sec today, 50,000 tomorrow?
→ One primary can't scale writes vertically forever.
→ Next: **sharding** — distribute writes across multiple primaries.
