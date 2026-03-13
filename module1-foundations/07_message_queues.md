# Message Queues

## The Pain
Synchronous request chains: if one downstream service is slow/down, entire request fails and user waits. Services are tightly coupled — producer must know every consumer.

## The Solution: Async via Queue
Producer writes message to queue → returns immediately. Consumers process at their own pace. If consumer is down, messages wait — no data loss.

```
[SYNC]  User → Order Service → Payment → "Order confirmed!" (200ms)
[ASYNC] Queue → Restaurant notification, Email, Analytics (background)
```

## Two Patterns

### Point-to-Point (Queue)
1 message → 1 consumer. Work is distributed.
- Use: task distribution (process order, resize image)

### Pub/Sub (Topic)
1 message → ALL subscribers get a copy.
- Use: event broadcasting (order placed → email + analytics both need it)

### When to Use Which
- **Queue** when you want ONE worker to handle each task (restaurant notification — don't send 5 alerts)
- **Topic** when MULTIPLE services need the same event (email + analytics each do different things)
- **Same system often needs both** — some work distributed, some broadcast

### Kafka Consumer Groups (Best of Both)
```
Topic: "order-events"
  → Consumer Group A (email, 3 instances): queue behavior within group
  → Consumer Group B (analytics, 2 instances): queue behavior within group
  Between groups: topic behavior — both get every message
```

## The 4 Powers of Queues

| Power | What it gives |
|-------|--------------|
| Decoupling | Producer doesn't know/care who consumes. Add consumers without changing producer |
| Buffering | Absorbs traffic spikes. 10K/sec spike → queue holds, consumers process at steady 1K/sec |
| Resilience | Consumer dies → messages wait. No data loss. Restart picks up where left off |
| Ordering | Messages processed in order (within a partition) |

## Delivery Guarantees

| Guarantee | Meaning | Tradeoff |
|-----------|---------|----------|
| At-most-once | Might lose, never duplicate | Fire-and-forget. Fast but unsafe for payments |
| At-least-once | Never lose, might duplicate | Retry until ACK. Consumer must be idempotent |
| Exactly-once | Never lose, never duplicate | Expensive. Kafka does it with idempotent producers + transactions |

**Standard: at-least-once + idempotent consumer** = effectively exactly-once.

```
Consumer reads → processes → ACKs
If crash between process and ACK → queue redelivers → consumer sees duplicate
→ Consumer must handle duplicates (idempotency)
```

## Back-Pressure
Producer faster than consumer → queue grows. Options:
1. Grow unbounded → OOM crash
2. Drop old messages → data loss (ok for metrics, not orders)
3. Block producer → back-pressure (slow down, I can't keep up)
4. Scale consumers → add more instances

## Real Systems

| System | Type | Strength |
|--------|------|----------|
| RabbitMQ | Traditional broker | Flexible routing, good for task queues |
| Kafka | Distributed log | Massive throughput, replay, ordering per partition |
| SQS | Managed queue | Zero ops, auto-scale, dead letter queues |
| SNS | Managed pub/sub | Fan-out to SQS/Lambda/HTTP |
| Redis Streams | Lightweight | Low latency, simple use cases |

### Kafka vs RabbitMQ

| | Kafka | RabbitMQ |
|---|-------|----------|
| Model | Append-only log | Traditional queue |
| Replay | Yes — re-read old messages | No — gone after ACK |
| Throughput | Millions msg/sec | Tens of thousands msg/sec |
| Ordering | Per partition | Per queue |
| Use case | Event streaming, analytics | Task distribution, RPC |

## Design Example: Food Delivery Order

```
User places order:
  [SYNC] Payment → charge card → if fail, stop → return "Order confirmed!"
  [QUEUE - high priority] Restaurant notification (guaranteed, retried)
  [TOPIC - low priority] → Email consumer (confirmation)
                         → Analytics consumer (dashboard)
```

Why queue for restaurant (not topic)? Only ONE notification needed. Why topic for email+analytics? Both need the same event independently.

## Interview Angle
1. Explain sync vs async — "critical path sync, everything else queued"
2. Queue vs topic — know when to use each
3. Delivery guarantees — at-least-once + idempotency is the standard
4. Name real systems — Kafka for streaming, RabbitMQ for tasks, SQS for managed
5. Back-pressure — show you think about what happens when things go wrong
