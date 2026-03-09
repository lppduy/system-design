# M1-01: Load Balancing

## Why it matters

Distributes traffic across servers. Without it, one server gets everything — single point of failure and bottleneck.

---

## Algorithms

| Algorithm | Based on | Weakness |
|---|---|---|
| Round-robin | Turn order | Ignores server capacity |
| Weighted round-robin | Static capacity config | Ignores runtime load |
| Least connections | Active connections | Ignores request duration |
| Least response time | Connections + latency | More complex to implement |

**Least connections** is the most practical default for HTTP services — dynamic, no static config needed.

---

## Layer 4 vs Layer 7

**Layer 4 (Transport):** Routes based on IP + TCP/UDP port only. Doesn't inspect packet contents — just forwards bytes.

```
Client → LB sees: IP 1.2.3.4, port 443 → forwards to Server
```

**Layer 7 (Application):** Understands HTTP. Routes based on URL, headers, cookies.

```
/api/*        → API servers
/images/*     → image servers
/checkout/*   → payment servers
```

### Why Layer 7 is slower

Layer 7 must **terminate the TCP connection**, read the HTTP request, decide routing, then **open a new TCP connection** to the backend:

```
Layer 4:  Client ──TCP──► LB ──TCP──► Server  (forwarded, one connection)
Layer 7:  Client ──TCP──► LB (reads request) ──new TCP──► Server (two connections)
```

Two TCP handshakes + HTTP parsing = more CPU and latency.

**Overhead** = extra work that doesn't directly contribute to the result but is required. Layer 7 overhead: terminate TCP, parse headers, read URL, make routing decision, open new connection.

### When to use each

| | Layer 4 | Layer 7 |
|---|---|---|
| Speed | Faster | Slower |
| Routing | IP + port | URL, headers, cookies |
| Use case | Databases, game servers, raw TCP | HTTP microservices, API routing |
| Example | AWS NLB | AWS ALB, Nginx, Envoy |

---

## TCP vs UDP

**TCP** — reliable, connection-based.
- 3-way handshake before data flows (SYN → SYN-ACK → ACK)
- Every packet acknowledged, lost packets retransmitted, order guaranteed
- Slower but reliable

**UDP** — fire and forget.
- No handshake, no acknowledgement, no ordering
- Packets can be lost or arrive out of order
- Faster, used for video streaming, gaming, DNS

---

## High Availability — Removing the Single Point of Failure

### Active-Passive
```
Primary LB (active)   ← all traffic
Secondary LB (passive) ← standby
```
Primary dies → Secondary detects via heartbeat → takes over. Simple but wastes resources — Secondary sits idle 100% of the time.

### Active-Active
Both LBs handle traffic simultaneously. One dies → other already serves traffic. No wasted resource.

Coordination options:
- **DNS round-robin** — simple but has caching issues and no health checking
- **Anycast** — multiple machines share the same IP globally, network routes to nearest/healthiest (used by Cloudflare)
- **Managed LB** (AWS ALB, GCP LB) — handles HA for you, practical default

---

## IP Failover / Floating IP

Normal: IP is tied to one machine. Floating IP: the IP can move between machines.

```
Virtual IP (VIP): 10.0.0.1  ← what clients connect to
Primary LB:   10.0.0.2  (owns the VIP)
Secondary LB: 10.0.0.3  (standby)
```

Primary dies → Secondary stops receiving heartbeat → claims VIP via **Gratuitous ARP** (broadcasts new IP→MAC mapping to network) → traffic flows to Secondary in 1-3 seconds.

**Why not DNS?** DNS changes are cached everywhere — can take minutes/hours to propagate. IP failover happens at network layer, nearly instant.

---

## Summary

| Concept | What it solves |
|---|---|
| Round-robin / Weighted | Distribute traffic evenly |
| Least connections / response time | Runtime-aware routing |
| Layer 4 | Fast, IP+port routing |
| Layer 7 | Content-based routing |
| Active-Passive | LB redundancy, simple failover |
| Active-Active | LB redundancy, no wasted resource |
| IP Failover | Instant failover without DNS delay |
| Anycast | Global LB at network level |
