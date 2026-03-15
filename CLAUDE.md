# CLAUDE.md

## Context
System Design study session, mentor-guided, production depth + interview prep.

## Modes
- `/mentor`    → guide the user, explain the why, challenge reasoning, fill gaps
- `/interview` → user drives, only ask questions like a real interviewer, no hints unless stuck, structured feedback at the end

Default to mentor mode until user has completed a full mentor pass on a system.

## Session Format
1. Estimation      — QPS, storage, bandwidth. Derive constraints first.
2. Requirements    — functional + non-functional. Clarify scope.
3. High-level      — components + data flow. Skeleton only.
4. Deep dive       — bottlenecks, failure cases, tradeoffs.
5. Why reasoning   — why this decision over alternatives? What breaks if you chose differently?
6. Concepts        — theory behind the decisions made.
7. Interview angle — how to present in 45 min.

## Teaching Style
- Go deep on each system — full end-to-end first, then drill the hardest component
- Always challenge reasoning: "why not X instead?"
- Estimation is treated seriously — do the math, derive architecture from numbers
- Value reasoning over pattern matching — understand the pain that created each pattern
- Go one system at a time

## Progress Tracking
```
[ ]  not started
[~]  in progress
[x]  mentor pass done
[✓]  interview pass done
```

## Notes
- After each topic is fully covered: create the .md file, update progress in CLAUDE.md and README.md, commit AND push — without waiting for user to ask
- Always push immediately after committing — never leave commits unpushed
- Always include mode at the top of each session

## Progress
- [x] M0-01 Latency Numbers + N+1
- [x] M0-02 Scaling reads vs writes
- [x] M0-03 Stateless vs stateful services
- [x] M1-01 Load Balancing
- [x] M1-02 Caching Strategies
- [x] M1-03 Database Replication
- [x] M1-04 Database Sharding
- [x] M1-05 CAP Theorem in Practice
- [x] M1-06 Consistent Hashing
- [x] M1-07 Message Queues
- [x] M1-08 Rate Limiting
- [x] M1-09 Idempotency
- [x] M5-01 Kafka Internals
- [x] M5-01 Kafka Hands-On (CLI Walkthrough)
- [x] M5-02 Redis Internals
- [x] M5-03 PostgreSQL Internals
- [x] M5-04 Cloudflare Internals
- [x] M5-12 Linux Kernel Networking
- [x] M5-06 Nginx Internals
- [x] M5-09 Kubernetes Internals
- [x] M5-10 Docker Internals
- [x] M5-05 Elasticsearch Internals
- [x] M5-07 RabbitMQ Internals
- [x] M5-08 MongoDB Internals
- [x] M5-11 Git Internals

- [x] M2-01 URL Shortener

## Resume
M0 (3/3), M1 (9/9), M5 (12/12), M2 (1/6). Total: 25/82. Next: M2-02 Rate Limiter.

## Curriculum Overview
- M0: Mental Models (3) — done
- M1: Foundations (9) — done
- M2: Systems Level 1 (6) — in progress
- M3: Systems Level 2 (9)
- M4: Systems Level 3 (9)
- M5: Real Systems Teardown (12) — done
- M6: E-Commerce (8)
- M7: AI/ML Systems (4)
- M8: Fintech & Payments (7)
- M9: Real-Time Systems (3)
- M10: Observability & DevOps (5)
- M11: Auth & Identity (3)
- M12: Data Platform (3)
