# M3-02: Chat System

## Estimation

| Metric | Value |
|--------|-------|
| MAU | 500M |
| DAU | 200M (40%) |
| Msgs/user/day | 40 |
| Total msgs/day | 8B |
| QPS | ~93K msg/s (peak ~280K) |
| Avg msg size | 150B (100B text + 50B metadata) |
| Text storage/day | ~1.2 TB |
| Media (20% msgs, avg 200KB) | ~320 TB/day |
| Concurrent WS connections | 60M (30% of DAU) |
| Chat servers needed | ~120 (500K conn/server) |
| RAM per server (connections only) | ~5GB |

## Estimation → Architecture Decisions

| Number | Decision |
|--------|----------|
| 60M concurrent connections | Cluster of chat servers, not 1 monolith |
| 93K msg/s writes | Write-optimized DB (Cassandra LSM-tree, not PostgreSQL B-tree) |
| 320 TB/day media | Separate object storage (S3), DB only for metadata |
| 500K conn/server × 10KB RAM | ~120 servers, need session map (Redis) to route messages |

## Requirements

### Functional
- 1:1 messaging — send/receive text real-time
- Group chat — up to 500 members
- Online/offline presence
- Message history — sync across devices
- Sent/delivered/read receipts
- Media sharing (out of scope for deep dive)

### Non-functional
- Low latency: < 200ms delivery (same region)
- Ordering: messages in a conversation must be strictly ordered
- Durability: no message loss (at-least-once delivery)
- Availability: 99.99% (chat is core, downtime = user churn)
- Consistency: eventual OK for presence, strong for message ordering/durability

## High-level Architecture

```
Client A ──WS──► Chat Server 1 ──persist──► Cassandra (Message DB)
                      │
                      ▼
                    Kafka
                      │
                      ▼
                 Chat Server 2 ──WS──► Client B

Session Map: Redis (user_id → chat_server_id)
Presence: Redis (user_id → online/last_heartbeat)
Media: S3 (object storage)
```

### Message Flow (A → B)
1. A sends via WebSocket → Chat Server 1
2. Chat Server 1 persists to Cassandra → ack "sent" to A
3. Publish to Kafka
4. Chat Server 2 (holding B's connection) consumes from Kafka
5. Push to B via WebSocket
6. B offline → push notification via APNs/FCM, B syncs on reconnect

## Foundation Concepts

### WebSocket vs Long Polling vs SSE
| | WebSocket | Long Polling | SSE |
|---|---|---|---|
| Direction | Bi-directional | Client→Server (fake push) | Server→Client only |
| Connection | 1 persistent TCP | Repeated HTTP requests | 1 persistent HTTP |
| Overhead | Low (2-6 bytes/frame) | High (HTTP headers each req) | Medium |
| Use case | **Chat, gaming** | Legacy fallback | News feed, notifications |

### Message Ordering
- Timestamps unreliable (clock skew between servers)
- Solution: **sequence number per conversation** (atomic increment)
- Redis INCR for per-conversation counter — single-threaded = naturally atomic
- Snowflake ID for global uniqueness + approximate time ordering
- Combo: Snowflake ID (global) + per-conversation sequence (ordering)

### Presence Detection
- Heartbeat every 30s → update TTL in Redis (`EXPIRE key 90`)
- Miss 3 heartbeats → TTL expires → offline event
- Only **broadcast status changes**, not every heartbeat
- Group presence: pub/sub per group, only for active conversations

## Deep Dive

### 1. Hot Partition — Time Bucketing

**Problem:** `PRIMARY KEY (conversation_id, message_id)` — long-lived group = millions of msgs in one partition. Cassandra recommends < 100MB/partition.

**Solution:** Add time bucket to partition key:
```sql
PRIMARY KEY ((conversation_id, bucket), message_id)
-- bucket = 'YYYY-MM'
```

Query recent 20 msgs: read current bucket first, fallback to previous if < 20.

**Trade-off:** Cross-bucket query needs 2 reads. But 99% cases, 1 bucket has > 20 msgs.

### 2. Message Routing — Kafka vs Direct

| | Direct (gRPC) | Kafka |
|---|---|---|
| Latency | ~1-2ms | ~5-10ms |
| Durability | Message lost if receiver crashes | Message persisted in Kafka |
| Coupling | Tight (server-to-server) | Decoupled |
| Offline handling | Must build retry queue | Natural (consume on reconnect) |

**Winner:** Kafka — durability is non-negotiable for chat. 5ms extra latency is acceptable (target < 200ms).

### 3. Group Chat Fan-out

**Problem:** Chat server holding WS connections shouldn't also fan-out to 10K members (blocking).

**Solution:** Dedicated **Group Fan-out Worker** (stateless):
1. Chat Server publishes 1 msg to Kafka topic `group_messages`
2. Fan-out Worker consumes, reads member list, batch Redis lookup (`MGET`)
3. Groups members by chat server, publishes per-server Kafka topics
4. Offline members → flag in DB + push notification

**Why:** Chat server stays responsive. Fan-out workers scale horizontally.

### 4. Offline Message Sync

```
Client: SYNC { last_seen_sequences: { conv_1: 450, conv_2: 890 } }
Server: SELECT * FROM messages WHERE conv_id = X AND seq > 450
```

Optimizations: pagination (20 msgs first), priority (recent convos first), gzip compression.

### 5. Read Receipts

```
sent      → server persisted (ack to sender)
delivered → recipient's device received (recipient's server ack)
read      → recipient opened conversation (explicit client event)
```

Group: per-member read status stored as `message_id → {user_1: read_at, user_2: null}`.

## Server Failure Recovery

1. Chat server dies → 500K WS connections drop
2. Client detects via heartbeat timeout (~30s)
3. Client reconnects with **exponential backoff + jitter** → LB assigns new server
4. Redis session map updated to new server
5. Client syncs missed messages via `last_seen_message_id` from DB
6. **No message loss** — messages already persisted before ack

## Why Reasoning

| Decision | Why | What breaks otherwise |
|----------|-----|----------------------|
| WebSocket over Long Polling | Bi-directional, 2-6 bytes/frame vs HTTP headers. 60M connections × header overhead = massive waste | High latency, bandwidth waste |
| Cassandra over PostgreSQL | LSM-tree sequential writes handle 93K/s. Horizontal scaling. PostgreSQL B-tree caps at ~10-20K writes/node | Write bottleneck, can't scale |
| Kafka over direct gRPC | Durability (persist before route), decoupling, natural offline handling | Message loss on server crash, must build retry queue = reinventing MQ |
| Redis for session + presence | Sub-ms latency for 93K lookups/s + 2M heartbeat ops/s | DB can't handle this throughput |
| Time-bucketing | Keep partition < 100MB, avoid compaction/repair slowdowns | Hot partitions, degraded read/write perf |
| Fan-out worker for groups | Chat server stays responsive, workers scale independently | Chat server blocked during fan-out, latency spike for all users on that server |

## Key Concepts

| Concept | Role in system |
|---------|---------------|
| WebSocket | Persistent bi-directional client↔server connection |
| LSM-tree | Cassandra write path — sequential, write-optimized |
| Time-bucketing | Partition strategy to avoid hot partitions |
| Fan-out | Group message delivery via dedicated stateless workers |
| Pub/Sub | Presence status broadcast (only status changes) |
| Heartbeat + TTL | Detect online/offline without continuous polling |
| Snowflake ID | Globally unique, time-ordered message IDs |
| Exponential backoff + jitter | Client reconnect strategy to avoid thundering herd |
| At-least-once delivery | Persist first, ack later — zero message loss |

## Interview Presentation (45 min)

| Time | Section | Key points |
|------|---------|------------|
| 0-5 | Clarify & Estimate | 500M MAU, 93K msg/s, 60M concurrent WS, 1.2TB/day |
| 5-10 | Requirements | 1:1 + group, ordering, durability, presence |
| 10-20 | High-level | Client→WS→Chat Server→Kafka→Chat Server→Client. Redis session. Cassandra storage |
| 20-35 | Deep dive (pick 2) | (1) Group fan-out worker (2) Time-bucketing schema (3) Presence optimization |
| 35-45 | Tradeoffs | Why Cassandra > PG, Why Kafka > direct, Server failure recovery |
