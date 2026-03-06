# System Design

Scenario-driven, mentor-guided. Understand deeply first, then practice under interview conditions.

---

## Philosophy

> Don't memorize patterns. Understand the pain that created them.
> Every design decision is a tradeoff — know what you're trading and why it's worth it.

---

## Modes

```
/mentor    → I guide, explain, challenge your reasoning
/interview → you drive, I only ask questions, feedback at the end
```

Default is mentor until you've done a full pass on a system. Switch to interview when you want to test yourself.

---

## Session Format

```
1. Estimation      — QPS, storage, bandwidth. Derive the constraints first.
2. Requirements    — functional + non-functional. Scope kills interviews.
3. High-level      — components + data flow. No depth yet, just the skeleton.
4. Deep dive       — bottlenecks, failure cases, tradeoffs. The real work.
5. Why reasoning   — why this decision over alternatives? What breaks if you chose differently?
6. Concepts        — theory behind the decisions (CAP, consistent hashing, etc.)
7. Interview angle — how to present this in 45 min without getting lost
```

---

## Progress Tracking

```
[ ]  not started
[~]  in progress
[x]  mentor pass done
[✓]  interview pass done — can present confidently in 45 min
```

---

## Module 0: Core Mental Models

Quick primers — cover these before diving into systems.

| # | Topic | Why it matters |
|---|-------|---------------|
| 01 | Latency numbers every engineer should know | Without this, estimation is guesswork |
| 02 | Scaling reads vs writes | Almost every design splits on this axis |
| 03 | Stateless vs stateful services | Determines how you scale horizontally |

## Module 1: Foundations

Concepts that appear in almost every system design.

| # | Topic | Status |
|---|-------|--------|
| 01 | Load balancing | [ ] |
| 02 | Caching strategies | [ ] |
| 03 | Database replication | [ ] |
| 04 | Database sharding | [ ] |
| 05 | CAP theorem in practice | [ ] |
| 06 | Consistent hashing | [ ] |
| 07 | Message queues | [ ] |
| 08 | Rate limiting | [ ] |
| 09 | Idempotency | [ ] |

## Module 2: Systems — Level 1

Single hard problem per system. Good for warming up.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | URL Shortener | ID generation at scale, read-heavy | [ ] | [ ] |
| 02 | Rate Limiter | Distributed counter consistency | [ ] | [ ] |
| 03 | Pastebin | Storage, expiry, access control | [ ] | [ ] |
| 04 | Web Crawler | BFS at scale, deduplication | [ ] | [ ] |

## Module 3: Systems — Level 2

Multiple interacting hard problems per system.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | Twitter Feed | Fan-out, celebrity problem | [ ] | [ ] |
| 02 | Chat System | Message ordering, presence | [ ] | [ ] |
| 03 | Notification System | Fan-out at scale, delivery guarantees | [ ] | [ ] |
| 04 | Ride-sharing (Uber) | Location updates, matching, ETA | [ ] | [ ] |
| 05 | Typeahead Search | Low latency prefix search | [ ] | [ ] |
| 06 | Ticket Booking | Concurrency, seat locking | [ ] | [ ] |

## Module 4: Systems — Level 3

Ambiguous scope + production-scale complexity.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | Payment System | Exactly-once, partial failures | [ ] | [ ] |
| 02 | Google Drive | Chunked upload, sync, conflict | [ ] | [ ] |
| 03 | YouTube / Netflix | Ingestion, streaming, CDN | [ ] | [ ] |
| 04 | News Feed (Facebook scale) | Ranking, personalization, scale | [ ] | [ ] |
| 05 | Distributed Cache (design Redis) | Eviction, persistence, clustering | [ ] | [ ] |
| 06 | Live Streaming | Ingest, transcode, low latency | [ ] | [ ] |
| 07 | Search Engine | Crawl, index, rank | [ ] | [ ] |

## Module 5: Real Systems Teardown

How famous systems actually work — not the marketing, the internals.

| # | System | What we dissect | Status |
|---|--------|----------------|--------|
| 01 | Kafka | Sequential writes, zero-copy, consumer groups | [ ] |
| 02 | Redis | Single-threaded event loop, persistence tradeoffs | [ ] |
| 03 | PostgreSQL | WAL, replication, MVCC | [ ] |
| 04 | Cloudflare | Anycast routing, DDoS, edge caching | [ ] |
