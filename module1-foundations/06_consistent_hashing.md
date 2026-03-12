# Consistent Hashing

## The Problem
`hash(key) % N` — when N changes (add/remove server), ~100% of keys remap. Moving data = downtime, network load, cache misses.

**Goal:** When adding/removing a node, move only `K/N` keys (K=total keys, N=nodes). 100 nodes + 1 new → only ~1% moves.

## How It Works: The Hash Ring

1. Hash each server onto a ring (0 to 2^32)
2. Hash each key onto the ring
3. Walk clockwise → first server you hit owns that key

```
Ring (0-360 simplified):
  Server A → 90,  Server B → 210,  Server C → 330

Key "user_1" → hashes to 120 → clockwise → Server B (210)
Key "user_2" → hashes to 350 → clockwise → wraps → Server A (90)
```

### Adding a Server
Add Server D at position 150:
- Before: keys 91-210 → Server B
- After: keys 91-150 → Server D (only these move), keys 151-210 → Server B (unchanged)
- Everything else stays put. Only one ring segment affected.

### Removing a Server
Remove Server B (210): its keys move to next clockwise neighbor. Everyone else untouched.

## Virtual Nodes (vnodes)

3 servers on a ring → uneven distribution (one might own 60%). Solution: each server gets multiple positions.

### How to Compute Vnode Positions
Hash the server name + index:
```
position_0 = hash("ServerA-0") % 2^32 → 847291
position_1 = hash("ServerA-1") % 2^32 → 2391847
position_2 = hash("ServerA-2") % 2^32 → 3918274
```

Hash function scatters them uniformly. No manual placement needed. Deterministic — every client computes same positions from same server list.

### Implementation (Java pseudocode)
```java
TreeMap<Integer, String> ring = new TreeMap<>();
for (Server s : servers)
    for (int i = 0; i < NUM_VNODES; i++)
        ring.put(hash(s.name + "-" + i), s.name);

// Lookup: O(log N) via ceiling/first entry
int keyHash = hash("user_123");
Entry entry = ring.ceilingEntry(keyHash);
if (entry == null) entry = ring.firstEntry(); // wrap
String owner = entry.getValue();
```

**In practice:** 100-200 vnodes per server. Cassandra uses 256 by default.

**Bonus:** More powerful server → more vnodes → proportionally more load. Heterogeneous hardware handled naturally.

## Node Failure

5-node Memcached cluster, Node 3 dies:
- **Node 3's keys:** distributed across multiple surviving nodes (each vnode's next-clockwise neighbor differs) — no single node gets slammed
- **Other nodes' keys:** completely unchanged
- **Cache hit rate:** drops ~20% (Node 3's share), recovers as new owners cache on first miss
- **Danger:** thundering herd if DB can't handle the spike

## Replication on the Ring

Store each key on N consecutive clockwise nodes (typically N=3):
```
Key hashes to 120, N=3:
  → Node B (210) primary
  → Node C (330) replica 1
  → Node A (90)  replica 2
Node B dies → Node C already has data. Zero cache miss.
```

Tunable consistency: W=2, R=2 → strong (W+R > N). W=1, R=1 → fast, eventual.

## Where It's Used

| System | What it hashes |
|--------|---------------|
| DynamoDB | Partition keys → storage nodes |
| Cassandra | Row keys → ring nodes (murmur3, 256 vnodes) |
| Memcached (Ketama) | Cache keys → servers (MD5, 150 vnodes) |
| CDNs (Akamai) | URLs → edge servers |
| Load balancers | Requests → backends (sticky sessions) |

## Interview Angle
1. Start with the modulo problem — show why it's needed
2. Draw the ring — hash servers, hash keys, walk clockwise
3. Mention vnodes — solves uneven distribution
4. Connect to replication — N consecutive nodes for fault tolerance
5. Name real systems — Cassandra, DynamoDB, CDNs
