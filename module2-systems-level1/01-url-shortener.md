# M2-01: URL Shortener

## Estimation

- 100M URLs created/day, 1 day ~ 10^5 seconds -> **1K writes/s**
- Read:write = 100:1 -> **100K reads/s**
- 5 years: 100M x 365 x 5 = **180B URLs**
- Each record ~250 bytes -> **45 TB** total storage
- Short code: 7 chars base62 (62^7 = 3.5T possible) -> enough headroom

**Key insight:** Numbers tell us this is read-heavy. Bottleneck is serving 100K redirects/s, not storage or writes.

## Requirements

**Functional:**
1. `POST /shorten` -> returns short URL
2. `GET /{code}` -> 302 redirect to original
3. Analytics (click count per link)

**Non-functional:**
- Redirect latency: <10ms p99
- High availability
- 100K reads/s, 1K writes/s

**302 over 301** — need every click to hit our server for analytics. 301 = browser caches, you lose click data.

**No deduplication** — each request gets unique short code. Simpler, isolates analytics per user. Storage cost is negligible.

## High-Level Design

```
Write: User -> LB -> App Server -> generate code (from KGS batch) -> DB
Read:  User -> LB -> App Server -> Cache (hit?) -> DB (miss) -> populate cache -> 302

         ┌──────────┐
         │    LB    │
         └────┬─────┘
              │
      ┌───────┴───────┐
      │  App Servers   │  (stateless, hold KGS key batch in memory)
      └───┬───────┬────┘
          │       │
     ┌────▼──┐ ┌──▼────┐
     │ Cache  │ │  DB   │
     │(Redis) │ │(shard)│
     └────────┘ └───────┘

     ┌──────────┐
     │   KGS    │  (pre-generates keys offline)
     └──────────┘
```

## Deep Dive: ID Generation

Three approaches:

| Approach | Pros | Cons |
|----------|------|------|
| **Hash (MD5/SHA -> base62)** | Simple | Collisions (birthday paradox at ~2M URLs), URL normalization headaches |
| **Auto-increment counter** | Zero collisions, ordered | Single point of failure, coordination needed |
| **KGS (pre-generated keys)** | Zero collisions, no contention, fast | Extra component, lose unused keys on crash |

**Winner: KGS.** Each app server grabs a batch of 1000 keys from KGS, hands them out locally. No coordination between servers. Lost keys on crash = negligible out of 3.5T.

## Caching Strategy

- **Cache-aside** for reads: miss -> DB -> populate cache
- **Write directly to DB** for creates (only 1K/s, DB handles it fine)
- **Local in-memory cache** on app servers for viral/hot links (avoid Redis hot key saturation)

## Database & Sharding

```sql
short_code  VARCHAR(7)   PRIMARY KEY
long_url    VARCHAR(2048)
created_at  TIMESTAMP
click_count BIGINT
```

- **Shard key: `short_code`** — every read query uses it, one query hits exactly one shard
- ~50 shards (45TB / ~1TB per instance)
- Consistent hashing for shard distribution (add nodes without full reshuffle)

## Click Counting (Analytics)

Problem: 100K redirects/s = 100K writes/s for click counts. DB can't handle that.

**Solution — Redis INCR + periodic flush (simple):**
```
Click -> INCR redis:clicks:aB3kQ9x        (O(1), in-memory)
Cron every 10s -> GETSET to 0 -> batch UPDATE to DB
```

**Alternative — Kafka (rich analytics):**
```
Click -> produce event {code, timestamp, ip, referrer} to topic
Consumer -> aggregate per code per window -> batch UPDATE
```

Redis INCR for basic counts. Kafka when you need click-by-geo, time series, A/B testing.

**Principle:** Separate hot path (redirect) from analytics path (eventual consistency is fine).

## Concepts Applied

| Concept | Application |
|---------|------------|
| Back-of-envelope estimation | Derived read-heavy -> caching is critical |
| Cache-aside | Read path optimization |
| Pre-generated keys (KGS) | Eliminated collision + contention |
| Consistent hashing | Shard distribution |
| Write batching | Click counting via Redis INCR or Kafka |
| Separation of concerns | Redirect path (fast) vs analytics (eventual) |
| Hot key mitigation | Local in-memory cache for viral links |

## Interview Presentation (45 min)

```
 0-5 min   Clarify requirements, estimate QPS/storage
 5-10 min  High-level: LB -> App -> Cache -> DB
10-20 min  Deep dive: ID generation (hash vs counter vs KGS)
20-30 min  Caching strategy, hot key problem
30-40 min  Sharding, analytics (click batching)
40-45 min  Tradeoffs, what you'd improve with more time
```
