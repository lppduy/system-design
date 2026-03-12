# CAP Theorem in Practice

## The Core Problem
Network partition happens — nodes can't talk to each other. Now choose:
- **Accept writes on both sides** → data diverges → lost Consistency, kept Availability
- **Reject writes on one side** → data stays correct → kept Consistency, lost Availability

## CAP Theorem

| Letter | Means | Definition |
|--------|-------|-----------|
| C | Consistency | Every read gets the most recent write (linearizability) |
| A | Availability | Every request gets a response (no errors, no timeouts) |
| P | Partition tolerance | System works despite network failures between nodes |

**P is not optional** — networks fail. Real choice is always **CP or AP**.

## CP vs AP in Real Systems

| System | Choice | During partition |
|--------|--------|-----------------|
| PostgreSQL (primary + replicas) | CP | Minority partition stops serving |
| MongoDB (replica set) | CP | Minority becomes read-only |
| Cassandra | AP | All nodes accept writes; last-write-wins |
| DynamoDB | AP (default) | Eventually consistent; opt-in strong reads |
| ZooKeeper | CP | Minority refuses requests until quorum |
| Redis Cluster | AP | Both sides accept writes; data loss on merge |

## Common Misunderstandings
1. **"Pick 2 of 3"** — misleading. P is mandatory. You're choosing C or A.
2. **Not all-or-nothing** — same system can be CP for some ops, AP for others.
3. **Only matters during partitions** — when network is healthy, you can have all three.

## E-commerce Example: Mixed CP/AP

| Operation | Choice | Reasoning |
|-----------|--------|-----------|
| Inventory/stock | **CP** | Overselling = real money lost |
| Product catalog | **AP** | Stale price for seconds is fine; blocking catalog is not |
| Shopping cart | **AP** | Pre-purchase, no money. Merge conflicts via union of items (Amazon Dynamo approach) |
| Payment | **CP** | Money must be correct. Double-charge = disaster |
| Order status reads | **AP** | 30s stale "processing" vs blocking tracking page — stale is fine |
| Order status writes | **CP** | The moment warehouse marks "shipped" must be consistent |

**Key insight: mix CP/AP within same system, even same entity, based on read vs write.**

## Beyond CAP: PACELC

CAP only covers partition scenarios. PACELC extends to normal operation:

```
If Partition → choose A or C
Else (normal) → choose Latency or Consistency
```

| System | During Partition | Normal Operation |
|--------|-----------------|------------------|
| PostgreSQL | PC | EC (consistent, higher latency for sync replication) |
| Cassandra | PA | EL (low latency, eventual consistency) |
| MongoDB | PC | EC (consistent reads from primary) |
| DynamoDB | PA | EL (default) or EC (opt-in strong reads) |

DynamoDB per-read choice:
- `ConsistentRead: false` → fast, might be stale (EL)
- `ConsistentRead: true` → slower, guaranteed fresh (EC)

## Interview Angle
1. Don't recite "pick 2 of 3" — say "P is mandatory; real choice is C vs A during partitions"
2. Show nuance — different operations make different choices
3. Mention PACELC — shows CAP is only half the picture
4. Give concrete examples — "payment = CP, browsing = AP"
5. Key phrase: "Optimize for common case (no partition), degrade gracefully during partitions"
