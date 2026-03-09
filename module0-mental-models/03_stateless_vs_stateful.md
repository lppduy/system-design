# M0-03: Stateless vs Stateful Services

## Why it matters

Determines how freely you can scale horizontally. Stateless = cattle. Stateful = pets.

---

## The Problem

User logs in. Server needs to remember the session.

**Stateful (Option A):** Store session in server memory.

```
Load Balancer
  ├── Server 1 (holds User A's session)
  ├── Server 2
  └── Server 3
```

Next request routed to Server 2 → no session → user logged out.

**Fix attempt: Sticky sessions** — always route same user to same server.
- Problem 1: Heavy users concentrate on one server → hotspot
- Problem 2: Server crashes → all its sessions lost instantly

---

## Stateless (Option B): Push State Out

Store sessions in shared Redis:

```
Server 1 ─┐
Server 2 ──→ Redis (sessions)
Server 3 ─┘
```

Any server handles any request. Server crashes → sessions survive in Redis.

---

## What Stateless Enables

**Horizontal autoscaling:**
- Traffic spikes → spin up new instances in seconds
- Traffic drops → kill them, save cost
- Server crashes → replace it, no data lost

Only possible because all instances are interchangeable — none holds anything unique.

---

## Cattle vs Pets

> **Pets:** named, unique, you care if one dies.
> **Cattle:** identical, disposable, replace any one instantly.

Stateless servers = cattle. Stateful servers = pets.

Design goal: make app servers as stateless as possible. Push all state to dedicated stores (Redis, DB, object storage).

---

## When Stateful is Unavoidable

**WebSocket connections** — persistent connections held in server memory. Can't move to Redis.

```
User A → Server 1 (holds open socket)
User B → Server 2 (holds open socket)
```

User A messages User B → arrives at Server 1 → but B's socket is on Server 2.

**Fix:** Message broker (Redis Pub/Sub, Kafka) as bridge between servers.
The socket stays stateful, but messaging is decoupled.

→ Will cover in depth in Chat System (Module 3).

---

## Summary

| | Stateful | Stateless |
|---|---|---|
| Session storage | Server memory | Redis / DB |
| Load balancing | Sticky (hotspots) | Any server |
| Crash impact | Sessions lost | No impact |
| Autoscaling | Hard | Easy |
| Example | WebSocket server | REST API server |

**Rule:** Make servers as stateless as possible. Where state is unavoidable, isolate it and design around it explicitly.
