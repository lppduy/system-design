# M6-02: Shopping Cart & Checkout

## Estimation
- 10M DAU, 4 sessions/user, 5 actions/session → 200M actions/day
- Cart RPS: ~2,300 total (read ~1,400, write ~900)
- Checkout: 10M × 3% = 300k/day = ~3.5 RPS avg, ~175 RPS peak (flash sale 50x)
- Cart size: ~1KB/cart (JSON encoded) → 10M × 1KB = 10GB total

## Requirements

### Functional
- Cart: add/remove/update qty, view, clear, merge guest→user on login
- Checkout: select address + payment, apply coupon, confirm order
- Checkout flow: re-validate price → reserve inventory → process payment → create order

### Non-functional
- Cart: <50ms p99, eventual consistency OK, 99.9% availability
- Checkout/Inventory: strong consistency (no oversell), idempotency (no duplicate orders)
- Payment: exactly-once semantics, graceful timeout/retry handling
- Orders: durable (survive crash)

## High-Level Design

```
User → LB → API Gateway
  ├── Cart Service     → Redis (cart data, TTL 30d)
  └── Checkout Service → orchestrates:
        ├── Pricing Service    (re-validate prices)
        ├── Inventory Service  → Inventory DB (reserve stock)
        ├── Order Service      → Order DB (create order PENDING)
        └── Payment Service    → Payment DB (charge, idempotency key = order_id)
```

Services communicate via Kafka for async events (order.confirmed, inventory.released).

## Deep Dive

### 1. Race Conditions — Overselling Prevention

**Two-layer approach:**

```
Layer 1: Redis DECRBY (atomic gate)
  DECRBY stock:sku-101 1
  result < 0 → INCRBY (rollback), reject "Out of stock"  ← 99% of flash sale traffic
  result ≥ 0 → proceed to DB

Layer 2: DB Optimistic Lock (source of truth)
  UPDATE inventory SET stock = stock-1, version = v+1
  WHERE sku_id = 101 AND version = :v AND stock > 0
  affected = 0 → INCRBY Redis, retry or reject
```

Redis acts as gate → DB sees low contention → optimistic lock is safe.

### 2. Price Consistency

Re-validate at checkout moment (not page load):
- `current < snapshot` → charge current (user benefits, no confirmation needed)
- `current > snapshot` → BLOCK, show diff, require user confirmation

Store `final_price` per item in order — immutable after order created.

### 3. Cart Persistence

**Guest cart:** keyed by `cart:guest:{session_id}` (UUID from cookie), TTL 7 days

**Merge on login:**
```
for item in guest_cart:
    if sku in user_cart: user_cart[sku].qty += item.qty
    else: user_cart.add(item)
delete guest_cart
```

**Cross-device:** single Redis key `cart:user:{user_id}` → auto-synced

**Durability:** RDB snapshot every 60s (acceptable — cart loss = annoyance, not data loss)

### 4. Saga Pattern — Checkout State Machine

```
CREATED → INVENTORY_RESERVED → ORDER_CREATED → PAYMENT_PENDING
                                                    ↓           ↓
                                               COMPLETED    PAYMENT_FAILED
                                                              ↓
                                                    ORDER_CANCELLED → INVENTORY_RELEASED → FAILED
```

State persisted in DB → crash-safe resume. Checkout Svc = orchestrator (not choreography).

**Why orchestration:** compensating transactions centralized, easier to trace, money flow controlled.

### 5. Idempotency

`order_id` = idempotency key for Payment Service.
Double-click checkout → same `order_id` → Payment Service deduplicates → 1 charge only.

## Key Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Cart storage | Redis | <1ms latency, TTL built-in, 10GB fits in RAM |
| Checkout pattern | Orchestration | Centralized compensation, easier crash recovery |
| Inventory order | Before payment | Fail fast, fail cheap — no refund needed |
| Stock gate | Redis DECRBY | Atomic, rejects 99% at cache layer |
| DB concurrency | Optimistic lock | Low contention after Redis gate |
| Cart durability | RDB (60s) | Cart loss acceptable, AOF overkill for ephemeral data |

## Interview Angle (45 min)

```
0-5   Clarify scope + estimation
5-10  Requirements
10-18 High-level diagram
18-35 Deep dive: checkout race condition + saga state machine
35-42 Tradeoffs + why decisions
42-45 Scaling, bottlenecks, wrap up
```

**Opening:** "Before I design, let me clarify scope..."
**Key signal:** "The hardest part isn't the cart — it's checkout. Money movement requires exactly-once semantics, and distributed systems don't give you that for free."
