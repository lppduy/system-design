# Rate Limiting

## Why
Without it: one user/bot can take down your API for everyone. Protects against abuse, brute-force, buggy client retry loops.

Response: `429 Too Many Requests`

## Where to Apply
Most common: API Gateway level (Nginx, Kong, AWS API Gateway) — before app code runs.

## The 4 Algorithms

### 1. Fixed Window Counter
Count requests per time window. Reset at window boundary.
- Simple, low memory (1 counter)
- **Problem:** boundary burst — 100 req at 00:59 + 100 at 01:00 = 200 in 2 seconds

### 2. Sliding Window Log
Store every request timestamp. Count requests in last N seconds.
- Perfect accuracy
- **Problem:** high memory — stores every timestamp

### 3. Sliding Window Counter (Hybrid) ← Production favorite
Approximate sliding window using 2 fixed window counters.

```
Formula: estimated = prev_count × (1 - elapsed%) + current_count

Example at 01:15 (25% into current window):
  prev window (00:00-00:59): 80 requests
  curr window (01:00-01:59): 40 requests so far
  estimated = 80 × 0.75 + 40 = 100

Why 0.75? The sliding window 00:15→01:15 overlaps 75% with prev window.
We estimate 75% of prev's requests fell in that overlap.
```

Previous window's influence smoothly fades as you move through current window. No boundary burst. ~99.7% accurate. Only 2 counters in memory.

### 4. Token Bucket
Bucket holds tokens (capacity = burst size). Refills at steady rate. Each request takes a token.
- Allows controlled bursts up to bucket capacity
- Enforces sustained average rate
- Good for APIs where short bursts are acceptable (payment callbacks, SDK clients)

### Comparison

| Algorithm | Memory | Accuracy | Bursts |
|-----------|--------|----------|--------|
| Fixed window | 1 counter | Boundary issue | No |
| Sliding log | All timestamps | Perfect | No |
| Sliding counter | 2 counters | ~99.7% | No |
| Token bucket | 2 values | Good | Yes (controlled) |

## Distributed Rate Limiting

Single server: in-memory counter. Multiple servers: need shared state.

**Problem:** 100 req spread across 3 servers → each sees ~33 → all under limit → 100 total allowed when limit is 50.

**Solution:** Redis INCR (atomic, ~0.1ms). All servers increment same key. TTL = window size.

```
Server A: INCR user:123:rate → 1 → allow
Server B: INCR user:123:rate → 2 → allow
...
Any server: INCR → 51 → REJECT
```

## Fail Open vs Fail Closed (When Redis Dies)

| Strategy | Behavior | Use when |
|----------|----------|----------|
| Fail open | Allow all requests | Non-critical APIs (browsing, search) |
| Fail closed | Reject all requests | Critical APIs (payment, auth) — safety > availability |

Payment API = fail closed (CP reasoning — money must be correct).

## Interview Angle
1. Know all 4 algorithms with tradeoffs
2. Sliding window counter = go-to answer (best balance)
3. Token bucket when bursts acceptable
4. Distributed = Redis INCR, mention atomicity
5. Fail open vs closed — depends on what you protect
