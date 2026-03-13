# Idempotency

## The Pain
User clicks "Pay $100" → network timeout → retry → charged $200. Without idempotency, retries are dangerous.

**Idempotency:** same operation N times = same result as once. `f(x) = f(f(x))`

## HTTP Methods — Natural Idempotency

| Method | Idempotent? | Why |
|--------|------------|-----|
| GET | Yes | Reading doesn't change state |
| PUT | Yes | "Set value to X" — twice = same state |
| DELETE | Yes | "Remove this" — twice = still gone |
| POST | **No** | "Create new" — twice = two records |
| PATCH | Depends | "Set name=Bob" yes. "Increment counter" no |

POST is the dangerous one — used for payments, orders, transfers.

## The Idempotency Key Pattern

Client generates unique key per intended operation. Server deduplicates.

```
Request:  POST /payments  |  Idempotency-Key: abc-123  |  {amount: 100}

Server flow:
  1. Check: seen "abc-123" before?
  2. No  → process payment → store result with key → return 200
  3. Yes → return STORED result → don't process again
```

### Implementation (PostgreSQL)

```sql
-- Atomic insert-or-find using unique constraint
INSERT INTO idempotency_keys (key, status)
VALUES ('abc-123', 'started')
ON CONFLICT (key) DO NOTHING
RETURNING *;

-- Returns row    → first time, proceed with processing
-- Returns empty  → duplicate, fetch and return stored result
```

`RETURNING *` = return the affected rows after INSERT/UPDATE/DELETE (PostgreSQL-specific).

## Crash Recovery: Stale Lock Detection

Problem: server crashes mid-processing. Key stuck with status "started" forever.

Solution: timeout-based reclaim (lease-based locking).

```sql
-- Reclaim stale key (original processor assumed dead)
UPDATE idempotency_keys
SET status = 'started', created_at = NOW()
WHERE key = 'abc-123'
  AND status = 'started'
  AND created_at < NOW() - INTERVAL '5 minutes'
RETURNING *;
```

State machine: `started → complete` (happy path) or `started → (crash + timeout) → started (reclaimed) → complete`

## Making Non-Idempotent Operations Idempotent

Convert relative operations to absolute:

| Not idempotent | Idempotent version |
|---------------|-------------------|
| `counter += 1` | `SET counter = 5` |
| `INSERT INTO orders` | `INSERT ... ON CONFLICT DO NOTHING` |
| `send email` | Check "already_sent" flag first |
| `charge $100` | Idempotency key pattern |

## Connection to Message Queues

At-least-once delivery + idempotent consumer = effectively exactly-once.

```
Message: {order_id: 123, action: "send_email"}
First delivery  → send email → record "order_123_email_sent" in DB
Redelivery      → check DB → already sent → skip (no-op)
```

## Where It Matters

| System | Risk without idempotency |
|--------|------------------------|
| Payment APIs | Double charge |
| Order creation | Duplicate orders |
| Queue consumers | Process message twice |
| Webhooks | Duplicate notifications |
| DB migrations | Run twice = error |

## Interview Angle
1. Define it clearly — same operation N times = same result
2. Idempotency key pattern — client-generated key, server deduplicates
3. Atomic check — INSERT ON CONFLICT prevents race conditions
4. Crash recovery — stale lock detection with timeout
5. Queue connection — at-least-once + idempotency = exactly-once semantics
