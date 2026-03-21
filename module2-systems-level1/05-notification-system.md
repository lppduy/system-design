# M2-05 Notification System

## Mode: Mentor

## Estimation
- 500M users, 10 notifications/day avg → 5B notifications/day
- QPS: 5B / 86400 ≈ 58K/s, peak 3x ≈ 170K/s
- Avg notification size: 1KB (payload + metadata)
- Bandwidth: 58K × 1KB = 58 MB/s
- Storage (logs, 1 year): 5B × 1KB × 365 = 1.8 PB
- Channels: push (APNs/FCM), email (SMTP), SMS

## Requirements
**Functional:** Send notifications via push/email/SMS, user preference management, template support, notification history.
**Non-functional:** Reliability (at-least-once delivery), low latency for critical (OTP < 5s), deduplication, multi-channel, rate limiting.

## High-Level Design
```
Client/Service → Notification API → Kafka (per priority topic) → Workers → Third-party providers
                                                                    ↓
                                                            Notification Log DB

Supporting services:
- Template Service — render notification content
- User Preference Service — channel preferences, opt-out
- Device Token Store — APNs/FCM tokens per user
```

## Deep Dive

### 1. Delivery Reliability
- **Retry strategy**: exponential backoff + jitter, max 3 attempts
  - Attempt 1 → fail → wait 1s + jitter
  - Attempt 2 → fail → wait 4s + jitter
  - Attempt 3 → fail → move to DLQ
- **Jitter**: randomize retry delay to avoid thundering herd when provider recovers
- **Dead Letter Queue (DLQ)**: manual review by oncall
  - Classify: transient (provider outage → bulk replay) vs permanent (invalid token → discard + update DB)
  - Log failure reason in DLQ message metadata for quick triage

### 2. Deduplication
- Problem: Kafka at-least-once delivery → worker crash after send but before offset commit → duplicate
- **Solution: Idempotency key + manual Kafka offset commit**
  - Key: `hash(user_id + event_type + event_id)` — use event_id not timestamp (timestamp can differ on redeliver)
  - Redis: `SET notification:{key} EX 86400 NX`
    - OK = not sent yet → proceed
    - nil = already sent → skip
  - SET NX is atomic — no race condition between check and set
- **Kafka config**: `enable.auto.commit=false`, call `consumer.commitSync()` after processing
- Two layers: manual commit reduces duplicates, idempotency key catches the rest

### 3. Priority & Rate Limiting
- **Separate Kafka topics per priority** (not single topic with tag — Kafka has no native priority, reads by offset order)
  ```
  notifications.critical → dedicated 10 workers (OTP, security alerts)
  notifications.high     → 20 workers (order updates)
  notifications.low      → 5 workers (marketing)
  ```
- Why separate topics > single topic: isolation, independent scaling, no starvation. Also: consumer count <= partition count, single topic limits total consumers.
- **Rate limiting per user**: max 5 push/hr, 3 email/day
- **Critical (P0) bypasses rate limit** — OTP blocked = user can't login = broken UX
- Abuse protection for critical at API layer (rate limit OTP requests, not notifications)

### 4. Fan-out Problem
- Celebrity with 10M followers posts → 10M push notifications
- **Batch fan-out**: split followers into batches of 1000 → 10K batch messages in Kafka → workers process in parallel
- **Progressive rollout**: send to 1% first → check error rate → 10% → 100%. Prevents sending 10M with broken template.
- **Fan-out-on-write** (not on-read) — push notification is server-initiated, can't wait for user to open app

### 5. Monitoring & Feedback Loops
- **Business metrics**: delivery rate per channel (sent/delivered/failed), E2E latency P50/P95/P99, bounce rate, DLQ size
- **Infra metrics**: Kafka consumer lag, queue size, worker CPU/RAM
- **Alert rules**: delivery rate < 95% warning, < 80% page oncall, DLQ > 10K warning, consumer lag > 1M page
- **Feedback loops** (critical — ignoring = provider blacklists you):
  - APNs returns InvalidDeviceToken → remove token from DB
  - Email 550 bounce → mark email invalid
  - User unsubscribe → update preference service

## Why Reasoning
- **Why Kafka?** Buffer spikes, decouple API from workers (fire-and-forget → fast response), persistence (worker crash → replay), scale workers independently
- **Why separate topics?** Kafka reads by offset (no priority), consumer count limited by partition count, need independent scaling per priority
- **Why Redis for dedup?** In-memory ~0.1ms vs PostgreSQL ~2-5ms (disk). At 10K-100K msg/s, DB becomes bottleneck. Trade-off: Redis loses data on restart → acceptable (worst case = 1 duplicate)

## Concepts
| Concept | Usage |
|---|---|
| At-least-once delivery | Kafka default — may deliver message multiple times |
| Idempotency | Redis SET NX — ensure exactly-once send |
| Exponential backoff + jitter | Retry strategy — avoid thundering herd |
| Dead Letter Queue | Failed messages after max retries — manual review |
| Fan-out-on-write | Create notification per user at event time (push is server-initiated) |
| Priority queue via separate topics | Kafka has no native priority → separate topics |
| Rate limiting | Per user per channel — critical bypasses |
| Feedback loop | Provider status → update DB (remove invalid tokens) |

## Interview Angle (45 min)
- **0-5 min**: Estimation + requirements (channels, QPS, reliability)
- **5-15 min**: High-level design — API → Kafka → Workers → providers, draw diagram
- **15-35 min**: Deep dive 2-3 points — reliability (retry + DLQ), dedup (idempotency key), priority (separate topics)
- **35-40 min**: Fan-out — batching, progressive rollout, fan-out-on-write
- **40-45 min**: Monitoring + tradeoffs — proactively state tradeoffs before asked
- **Key interview tips**: Lead with retry + DLQ when asked about failure. Explain at-least-once vs exactly-once when asked about duplicates.

## Foundation Concepts
- **Redis SET NX**: `SET key value EX ttl NX` — set only if key not exists. Atomic. Returns OK (new) or nil (exists). Used for distributed locks and dedup.
- **Kafka manual commit**: Disable auto-commit (`enable.auto.commit=false`), call `commitSync()` after processing. Prevents message loss (auto-commit before processing done) at cost of possible redelivery (crash before commit).
