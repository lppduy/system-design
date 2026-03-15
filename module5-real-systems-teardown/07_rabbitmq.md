# M5-07: RabbitMQ Internals

## Why RabbitMQ (vs Kafka)

| | Kafka | RabbitMQ |
|---|---|---|
| Model | Distributed log (append-only) | Message broker (smart broker, dumb consumer) |
| Delivery | Consumer pulls from log | Broker pushes to consumer |
| Message lifetime | Retained (replay possible) | Deleted after ack |
| Ordering | Per-partition | Per-queue |
| Best for | Event streaming, log aggregation, high throughput | Task distribution, request/reply, complex routing |
| Protocol | Custom (Kafka protocol) | AMQP 0-9-1 (open standard) |

**Pain solved:** smart broker that routes messages based on rules, supports request/reply, guarantees each message processed by one worker.

---

## Core Architecture

```
Producer → Exchange → (binding + routing key) → Queue → Consumer → ack → deleted
```

Three primitives:
1. **Exchange** — receives messages, routes them (never stores)
2. **Binding** — rule connecting exchange to queue (with routing key/pattern)
3. **Queue** — stores messages until consumer acks

Producers never send directly to queues. Decoupling is the point.

---

## Exchange Types

### Direct
Exact routing key match. Use: task dispatch to known destination.

### Topic
Pattern matching: `*` = one word, `#` = zero or more.
- `order.*.created` matches `order.us.created`
- `order.#` matches `order.us.created` and `order`

Use: event systems with category subscriptions.

### Fanout
Broadcasts to all bound queues. Ignores routing key.
Use: cache invalidation, logging — everyone gets everything.

### Headers
Routes on message headers. Rarely used.

---

## AMQP 0-9-1 Protocol

### Connection Model
- **Connection** = TCP socket (expensive)
- **Channel** = virtual connection inside TCP (cheap, multiplexed)
- One TCP connection carries many channels
- Channel error doesn't kill other channels on same connection
- Like HTTP/2 multiplexing

---

## Message Lifecycle

```
Producer → publish(exchange, routing_key)
  → Exchange routes via bindings → enqueue
  → Broker pushes (basic.deliver) → Consumer
  → Consumer processes → basic.ack
  → Message removed from queue (gone forever, no replay)
```

---

## Acknowledgments & Delivery Guarantees

| Mode | Behavior | Risk |
|------|----------|------|
| Auto-ack | Removed on delivery | Message lost if consumer crashes mid-processing |
| Manual ack | Consumer sends explicit ack | Message redelivered if consumer dies before ack |
| Publisher confirms | Broker confirms receipt to producer | Without it, broker crash = message lost |

**At-least-once delivery requires ALL THREE:**
- Publisher confirms (producer side)
- Manual ack (consumer side)
- Durable queue + persistent messages (broker side)

### Duplicate Processing Problem
Consumer processes → writes to DB → crashes before ack → message redelivered → processed again.
**Fix:** Idempotency keys. Check message_id before processing. Ideally in same DB transaction as business logic.

---

## Queue Properties

| Property | Meaning |
|----------|---------|
| Durable | Survives broker restart (metadata on disk) |
| Exclusive | One connection only; auto-deleted on disconnect |
| Auto-delete | Deleted when last consumer disconnects |
| TTL | Messages expire after N ms |
| Max-length | Overflow: drop-head, reject-publish, or dead-letter |

### Persistence
- Durable queue + persistent message (delivery_mode=2) = survives crash
- Persistent messages require fsync → 10-100x slower than transient
- RabbitMQ batches fsyncs to mitigate

---

## Dead Letter Exchange (DLX)

Messages dead-lettered when:
1. Consumer nacks/rejects with `requeue=false`
2. Message TTL expires
3. Queue max-length exceeded

### Retry Pattern (no external scheduler)
```
Main Queue →(nack)→ Retry Exchange → Retry Queue (TTL=30s)
     ↑                                      |
     └──────── (TTL expires, DLX back) ─────┘
After N retries → Parking Lot Queue (manual inspection)
```

---

## Clustering & HA

### Mirrored Queues (deprecated 3.13+)
Replicate queue across nodes synchronously. Slow, doesn't scale.

### Quorum Queues (modern, Raft-based)
- Leader + followers, write requires majority consensus
- 3 nodes: tolerates 1 failure. 5 nodes: tolerates 2.
- Lose majority → queue unavailable (CP tradeoff)
- No split-brain — Raft leader election handles it
- Tradeoff: higher per-message latency for stronger safety

---

## Prefetch (Consumer Flow Control)

`basic.qos(prefetch_count=N)` — broker sends N unacked messages max.

- `prefetch=1` → fairest, low throughput
- `prefetch=10-50` → good balance for most workloads
- `prefetch=unlimited` → unfair, fast consumer idles while slow one drowns

Natural load balancing: fast consumers ack quickly → get more messages.

---

## Interview Angle

1. **Why** — decouple producers/consumers, handle spikes, retry failures
2. **Exchange + bindings** — smart routing, not just "put on queue"
3. **Delivery guarantees** — publisher confirms + manual ack + durable = at-least-once. Idempotency for exactly-once semantics.
4. **DLX for retry** — production maturity
5. **Quorum queues for HA** — Raft, majority consensus
6. **When NOT RabbitMQ** — event streaming/replay (Kafka), high-throughput log aggregation (Kafka), simple pub/sub (Redis)

---

## Key Concepts

| Concept | RabbitMQ Implementation |
|---------|------------------------|
| Smart routing | Exchange types (direct, topic, fanout, headers) |
| Delivery guarantee | Publisher confirms + manual ack + persistence |
| Retry/DLQ | Dead Letter Exchange + TTL chaining |
| HA/Fault tolerance | Quorum queues (Raft consensus) |
| Flow control | Prefetch (basic.qos) |
| Protocol | AMQP 0-9-1 (connection → channels) |
| Message lifecycle | Publish → exchange → route → queue → deliver → ack → delete |
