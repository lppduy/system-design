# Rate Limiter

## Estimation

- 100M DAU, ~50 req/user/day → 5B req/day
- QPS avg: ~58K, peak (3x): ~175K
- Per-user counter: user_id (8B) + counter (4B) + timestamp (8B) = ~20 bytes
- Active users in 1-min window: ~5M (5% DAU) → **~100MB** — single Redis instance
- **Key insight:** Storage is trivial. The hard problem is distributed counter consistency.

## Requirements

### Functional
1. Allow or deny request based on rate limit rules
2. Return 429 + headers (`Retry-After`, `X-RateLimit-Remaining`)
3. Limit by user ID (primary) + IP (fallback for unauthenticated)
4. Support multiple tiers (free/premium) — rules configurable at runtime

### Non-Functional
1. Latency <5ms p99 (hot path of every request)
2. Availability 99.99%, **fail open** when rate limiter is down
3. Distributed — correct behavior across multiple API servers
4. Scale linearly with traffic

### Fail Open vs Fail Closed
- **Fail open:** allow all traffic when rate limiter is down. Use when service is protection, not core logic (rate limiter, WAF, feature flags).
- **Fail closed:** deny all traffic. Use when bypass causes serious harm (auth, payment authorization, permission checks).
- Rate limiter → fail open. Letting traffic through temporarily is better than blocking 100M users.

## High-Level Design

```
Client → Load Balancer → API Gateway → Backend Services
                           │     ▲
                           │     │ allow/deny
                           ▼     │
                        Rate Limiter
                           │
                           ▼
                         Redis (counters)
                           │
                      Rules DB (config)
```

### Why API Gateway (middleware)?
- Earliest server-side interception point — blocks before wasting backend resources
- Client-side: attacker bypasses easily (curl, Postman, scripts)
- Server-side: request already consumed network/LB resources

### Request Flow
1. Client sends request → API Gateway
2. Gateway extracts user_id (JWT/API key) or IP
3. Rate Limiter checks Redis (Lua script, atomic)
4. Counter ≤ limit → ALLOW → forward to backend
5. Counter > limit → DENY → return 429
6. Redis down → FAIL OPEN → forward to backend

## Algorithms

### 1. Fixed Window Counter
- Count requests in fixed time windows (0:00-1:00, 1:00-2:00)
- **Problem:** Boundary burst — 100 req at 0:59 + 100 req at 1:01 = 200 req in 2 seconds
- Simple but inaccurate at window edges

### 2. Sliding Window Log
- Store timestamp of every request, count within last N seconds
- Perfectly accurate but **memory expensive** (100 entries/user × 5M users = 500M entries)

### 3. Sliding Window Counter
- Hybrid: approximate sliding window using 2 fixed window counters
- Formula: `estimate = counter_old × overlap% + counter_new`
- Where `overlap = (window_size - elapsed_in_current) / window_size`
- <1% error (Cloudflare verified), only 2 counters per user

### 4. Token Bucket ← production choice
- Bucket with capacity N, refills at rate R tokens/sec
- Request takes 1 token. No tokens → 429.
- **Allows burst** (good UX — page loads need multiple requests)
- Memory: just 2 fields (tokens_remaining + last_refill_time)
- Used by: AWS, Stripe, Cloudflare

### 5. Leaky Bucket
- Queue of requests, processed at fixed rate
- No burst — output always smooth
- Queue full → drop
- Best for protecting downstream services (DB, internal APIs)
- Used for traffic shaping, network bandwidth

### Token Bucket vs Leaky Bucket
| | Token Bucket | Leaky Bucket |
|---|---|---|
| Stores | Tokens (permission) | Requests (queued) |
| Refill? | Yes, adds tokens over time | No — requests flow in, drain at fixed rate |
| Burst | Allowed | No — output always constant rate |
| Request delay | No — allow or deny immediately | Yes — sits in queue waiting |
| Best for | API rate limiting (user-facing) | Traffic shaping, DB protection |

## Deep Dive: Distributed Rate Limiting

### The Problem
With N servers each keeping local counters, effective limit = limit × N.
5 servers × 100 req/min = 500 req/min per user.

### Solution 1: Centralized Store (Redis) ← recommended
- All servers INCR same key in Redis
- Lua script for atomicity (no race condition between GET and SET)
- +1-2ms RTT per request (acceptable within 5ms budget)
- Fail open when Redis is down

### Solution 2: Sticky Sessions
- Hash user_id → always route to same server
- Local counters work correctly
- Problem: server death = counter reset, uneven load

### Solution 3: Local + Async Sync (Hybrid)
- Local counters, sync to Redis every N seconds
- Less network calls but inaccurate between syncs
- Used at very large scale (Cloudflare)

### Multi-Region
- Each region has own Redis (cross-region latency 100-200ms too high)
- User switching regions gets fresh counter → can exceed limit
- **Production answer:** Accept eventual consistency. Most users don't hop regions. Abuse via VPN handled by fraud detection layer, not rate limiter.

## Why Reasoning

| Decision | Why | Alternative breaks what? |
|----------|-----|--------------------------|
| API Gateway placement | Earliest server-side interception | Client = bypassable. Server = too late |
| Token Bucket | Burst-friendly UX, low memory | Fixed window = boundary bug. Sliding log = memory |
| Redis centralized | Accurate, atomic Lua, simple | Local = limit×N. Sticky = fragile |
| Fail open | Protection layer, not core logic | Fail closed = entire platform down |
| User ID primary | Accurate identification | IP only = shared NAT blocks everyone |
| 99.99% availability | Realistic + fail open compensates | 99.999% = extremely expensive, unnecessary |

## Key Concepts
- **Race condition** → solved by Redis Lua (atomic operations)
- **CAP theorem** → rate limiter chooses AP (availability + partition tolerance)
- **Single point of failure** → Redis is SPOF, mitigated by Sentinel/Cluster + fail open
- **Thundering herd** → per-user burst OK, but need global rate limit for aggregate spike
- **Backpressure** → leaky bucket = backpressure mechanism

## Interview Presentation (45 min)

```
 0-5  min: Requirements + Estimation (derive constraints from numbers)
 5-10 min: High-level (draw: Client→LB→Gateway→Backend, Redis for counters)
10-25 min: Deep dive algorithms (token bucket diagram, Lua script, compare 5 algos)
25-35 min: Distributed challenges (multi-server, multi-region, failure modes)
35-40 min: Edge cases (thundering herd, NAT, dynamic rules)
40-45 min: Q&A (headers, monitoring 429 rate, per-endpoint limits)
```
