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
1. Estimation      — QPS, storage, bandwidth. Derive constraints first.
2. Requirements    — functional + non-functional. Clarify scope.
3. High-level      — components + data flow. Skeleton only.
4. Foundation      — new concepts needed for this system. Explain before deep dive.
5. Deep dive       — bottlenecks, failure cases, tradeoffs.
6. Why reasoning   — why this decision over alternatives? What breaks if you chose differently?
7. Concepts        — theory behind the decisions made.
8. Interview angle — how to present in 45 min.
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

# Core

## Module 0: Core Mental Models

Quick primers — cover these before diving into systems.

| # | Topic | Why it matters |
|---|-------|---------------|
| 01 | Latency numbers every engineer should know | [x] |
| 02 | Scaling reads vs writes | [x] |
| 03 | Stateless vs stateful services | [x] |

## Module 1: Foundations

Concepts that appear in almost every system design.

| # | Topic | Status |
|---|-------|--------|
| 01 | Load balancing | [x] |
| 02 | Caching strategies | [x] |
| 03 | Database replication | [x] |
| 04 | Database sharding | [x] |
| 05 | CAP theorem in practice | [x] |
| 06 | Consistent hashing | [x] |
| 07 | Message queues | [x] |
| 08 | Rate limiting | [x] |
| 09 | Idempotency | [x] |

---

# Classic Interview Systems

## Module 2: Systems — Level 1

Single hard problem per system. Good for warming up.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | URL Shortener | ID generation at scale, read-heavy | [x] | [ ] |
| 02 | Rate Limiter | Distributed counter consistency | [x] | [ ] |
| 03 | Pastebin | Storage, expiry, access control | [x] | [ ] |
| 04 | Web Crawler | BFS at scale, deduplication | [x] | [ ] |
| 05 | Notification System | Fan-out at scale, delivery guarantees | [x] | [ ] |
| 06 | Key-Value Store | Replication, consistency, partitioning | [x] | [ ] |
| 07 | Unique ID Generator | Distributed uniqueness, ordering, clock skew | [ ] | [ ] |

## Module 3: Systems — Level 2

Multiple interacting hard problems per system.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | Twitter Feed | Fan-out, celebrity problem | [ ] | [ ] |
| 02 | Chat System | Message ordering, presence | [ ] | [ ] |
| 03 | Ride-sharing (Uber) | Location updates, matching, ETA | [ ] | [ ] |
| 04 | Typeahead Search | Low latency prefix search | [ ] | [ ] |
| 05 | Ticket Booking | Concurrency, seat locking | [ ] | [ ] |
| 06 | Proximity Service (Yelp) | Geospatial indexing, quadtree/geohash | [ ] | [ ] |
| 07 | Ad Click Aggregation | Real-time counting, dedup, late events | [ ] | [ ] |
| 08 | Metrics Monitoring | Time-series DB, downsampling, alerting | [ ] | [ ] |

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
| 08 | Stock Exchange | Order matching, low latency, fairness | [ ] | [ ] |
| 09 | File Storage (S3) | Metadata vs data separation, replication, multipart upload | [ ] | [ ] |

---

# Real Systems Teardown

## Module 5: How Real Systems Work

Not the marketing — the internals.

| # | System | What we dissect | Status |
|---|--------|----------------|--------|
| 01 | Kafka | Sequential writes, zero-copy, consumer groups | [x] |
| 02 | Redis | Single-threaded event loop, persistence tradeoffs | [x] |
| 03 | PostgreSQL | WAL, replication, MVCC | [x] |
| 04 | Cloudflare | Anycast routing, DDoS, edge caching | [x] |
| 05 | Elasticsearch | Inverted index, sharding, near real-time search | [x] |
| 06 | Nginx | Event-driven architecture, reverse proxy, load balancing | [x] |
| 07 | RabbitMQ | AMQP protocol, exchanges, queues, acknowledgments | [x] |
| 08 | MongoDB | Document model, WiredTiger engine, replica sets | [x] |
| 09 | Kubernetes | Pod scheduling, etcd, service discovery | [x] |
| 10 | Docker | Namespaces, cgroups, overlay filesystem | [x] |
| 11 | Git | Content-addressable storage, DAG, merge strategies | [x] |
| 12 | Linux kernel (networking) | TCP stack, epoll, io_uring | [x] |

---

# Domain Specializations

Pick based on interest.

## Module 6: E-Commerce

Every critical subsystem in an e-commerce platform.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | Product Catalog & Search | Faceted search, inventory sync, SKU modeling | [ ] | [ ] |
| 02 | Shopping Cart & Checkout | Cart persistence, price consistency, race conditions | [ ] | [ ] |
| 03 | Order Processing Pipeline | State machine, saga pattern, partial failures | [ ] | [ ] |
| 04 | Payment System | Exactly-once, idempotency, refunds, reconciliation | [ ] | [ ] |
| 05 | Inventory Management | Distributed stock, overselling prevention, warehouse sync | [ ] | [ ] |
| 06 | Pricing & Promotions | Coupon stacking, flash sales, price calculation engine | [ ] | [ ] |
| 07 | Recommendation Engine | Collaborative filtering, real-time personalization | [ ] | [ ] |
| 08 | Notification & Email | Transactional vs marketing, delivery guarantees, templating | [ ] | [ ] |

## Module 7: Fintech & Payments

Money moves differently from data — every failure mode costs real dollars.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | Digital Wallet | Ledger design, double-entry, balance consistency | [ ] | [ ] |
| 02 | Payment Gateway | Provider routing, retry logic, idempotency, PCI compliance | [ ] | [ ] |
| 03 | Fraud Detection | Real-time scoring, feature engineering, false positives | [ ] | [ ] |
| 04 | Reconciliation Engine | Cross-system matching, discrepancy detection, settlement | [ ] | [ ] |
| 05 | Transfer System (P2P) | Atomicity across accounts, currency conversion, AML checks | [ ] | [ ] |
| 06 | Lending Platform | Credit scoring, loan lifecycle, interest calculation, collections | [ ] | [ ] |
| 07 | Trading Platform | Order book, matching engine, market data streaming | [ ] | [ ] |

## Module 8: AI/ML Systems

Design systems that serve, train, and orchestrate ML models at scale.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | RAG Pipeline | Chunking, embedding, retrieval, hallucination reduction | [ ] | [ ] |
| 02 | ML Feature Store | Online vs offline features, consistency, freshness | [ ] | [ ] |
| 03 | Model Serving Platform | Latency, batching, A/B testing, canary deployment | [ ] | [ ] |
| 04 | AI Agent Orchestration | Tool routing, memory, context management, guardrails | [ ] | [ ] |

## Module 9: Real-Time Systems

Low latency, high throughput, eventual consistency is not an option.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | Live Dashboard | WebSocket fan-out, aggregation, backpressure | [ ] | [ ] |
| 02 | Collaborative Editor | CRDT vs OT, conflict resolution, cursor sync | [ ] | [ ] |
| 03 | Multiplayer Game Server | State sync, lag compensation, client prediction | [ ] | [ ] |

## Module 10: Auth & Identity

Security architecture — the system everyone depends on, nobody wants to own.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | SSO & OAuth Provider | Token lifecycle, consent, refresh rotation | [ ] | [ ] |
| 02 | RBAC/ABAC Authorization | Policy engine, permission inheritance, multi-tenant | [ ] | [ ] |
| 03 | Multi-Tenant Platform | Data isolation, routing, tenant-aware caching | [ ] | [ ] |

## Module 11: Observability & DevOps

The systems that watch your systems.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | Logging Pipeline | Ingestion at scale, structured logs, retention, search | [ ] | [ ] |
| 02 | Distributed Tracing | Context propagation, sampling, trace assembly | [ ] | [ ] |
| 03 | Alerting System | Threshold vs anomaly, alert fatigue, escalation | [ ] | [ ] |
| 04 | Feature Flag Platform | Gradual rollout, targeting rules, kill switch | [ ] | [ ] |
| 05 | CI/CD at Scale | Pipeline orchestration, artifact management, blue-green | [ ] | [ ] |

## Module 12: Data Platform

From raw events to business insights.

| # | System | Hardest Part | Mentor | Interview |
|---|--------|-------------|--------|-----------|
| 01 | ETL Pipeline | Exactly-once processing, schema evolution, backfill | [ ] | [ ] |
| 02 | Data Lake / Warehouse | Partitioning, columnar storage, query optimization | [ ] | [ ] |
| 03 | Real-Time Analytics | Stream processing, windowing, late-arriving data | [ ] | [ ] |
