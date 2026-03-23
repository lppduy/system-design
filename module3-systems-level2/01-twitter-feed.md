# M3-01: Twitter Feed

## Estimation

| Metric | Value |
|--------|-------|
| Total users | 500M |
| DAU | 200M |
| Avg follows | 200/user |
| Tweets/day | 400M (~5K/sec, peak 15K) |
| Read QPS | 23K (peak 70K) |
| Read:Write | ~5:1 (read-heavy) |
| Tweet size | ~500B (text + metadata) |
| Storage/year | ~73TB |
| Fan-out (normal) | 5K tweets/sec × 500 followers = 2.5M cache writes/sec |
| Fan-out (celeb) | 1 tweet × 10M followers = 10M writes ← unacceptable |
| Feed cache (Redis) | 200M × 500 IDs × 8B = 800GB → ~15 Redis nodes |

## Estimation → Architecture Decisions

| Metric | Threshold | Decision |
|--------|-----------|----------|
| Read:Write >3:1 | Read-heavy | Cache heavily, pre-compute (push model) |
| Write QPS >100K | 2.5M fan-out writes/sec | Must async — use Kafka |
| Storage >10TB/year | 73TB | Sharded DB or NoSQL (Cassandra) |
| Peak QPS >10K | 70K reads | Horizontal scaling + LB |
| Fan-out >5K-10K | Celebs have millions | Switch from push to pull |
| Cache >50GB | 800GB | Redis Cluster (~15 nodes) |

## Requirements
**Functional:** Post tweet, view home feed (from followed users), reverse chronological.
**Non-functional:** Feed load <200ms, high availability, eventual consistency OK.

## High-level Design

```
Client → API Gateway → Tweet Service → Tweet DB
                     → Feed Service  → Feed Cache (Redis)
                     → Kafka → Fan-out Workers
```

## Core Problem: Fan-out Strategy

### Push (Fan-out on Write) — normal users
- User tweets → fan-out service writes tweet ID into every follower's feed cache.
- Follower opens app → feed already in cache → O(1) read.
- Works for users with <5K-10K followers.

### Pull (Fan-out on Read) — celebrities
- Celeb tweets → only stored in Tweet DB. No fan-out.
- Follower opens feed → get pre-built feed (normal tweets) + query celeb tweets on demand → merge + sort.
- Necessary because pushing to 10M+ followers per tweet is too expensive.

### Hybrid (Twitter's actual approach) ★
- Normal users: push.
- Celebrities (>5K-10K followers): pull.
- On read: merge cached feed + fresh celeb tweets.

## Deep Dive

### Feed Cache Structure
- Redis list per user: `feed:user_123 → [tweet_id_99, tweet_id_98, ...]`
- Store **tweet IDs only**, not content. Content fetched separately from Tweet Cache.
- Why: if tweet edited, update once in Tweet Cache. Not millions of feed caches.

### Fan-out Storm
- 1000 users tweet/sec × 500 followers = 500K cache writes/sec.
- Solution: Kafka buffers → fan-out worker pool consumes at own pace.

### Stale Feed (Unfollow)
- Unfollow → old tweets still in cache.
- Lazy cleanup: filter on read + background job cleans gradually.

### Cold Start (New User)
- New user follows 200 people → empty feed cache.
- Trigger backfill: pull recent tweets from all followed users, build cache.

## Why Reasoning

| Decision | Why | If different? |
|----------|-----|---------------|
| Hybrid push+pull | Pure push fails for celebs (150M writes). Pure pull slow for normal reads. | Either extreme breaks at scale |
| Tweet IDs in feed, not content | 1 edit = 1 update | Full content: 1 edit = millions updates |
| Kafka buffer | Absorb spikes, enable retry | Without: fan-out service overwhelmed |
| 5K-10K threshold | Balance write cost vs read latency | Too low: too many pull queries. Too high: fan-out too heavy |

## Concepts
- **Fan-out on Write:** Pre-compute at write time. Trade write cost for read speed.
- **Fan-out on Read:** Compute on demand. Trade read latency for write simplicity.
- **Celebrity Problem:** One entity affects disproportionate users → needs special path.
- **Hybrid Strategy:** No single approach fits all. Classify entities, apply different strategies.

## Interview Flow (45 min)
1. Requirements (3 min): post, read feed, follow/unfollow
2. Estimation (3 min): 200M DAU, 5K tweets/sec, read-heavy → cache + pre-compute
3. High-level (5 min): Tweet Service, Feed Service, Feed Cache
4. Fan-out (10 min): push vs pull → derive hybrid. Key scoring point.
5. Deep dive (10 min): cache structure (IDs only), Kafka fan-out, celebrity threshold
6. Tradeoffs (5 min): threshold tuning, eventual consistency, stale feed

**Bonus points:**
- "Twitter uses hybrid — push for normal, pull for celebrities"
- "Feed cache stores tweet IDs, not content — so edits update once"
- "Kafka buffers fan-out to handle traffic spikes"
- Replication/sharding: mention when asked about availability, not proactively
