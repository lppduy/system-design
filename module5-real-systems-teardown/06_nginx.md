# Nginx Internals

## What It Is
Reverse proxy, load balancer, and web server. Designed from day one to solve C10K problem (10,000 concurrent connections). Built on top of epoll.

## Architecture: Master + Workers

```
Master Process (1):
  - Reads nginx.conf, binds to port 80/443
  - Spawns worker processes
  - Does NOT handle requests

Worker Processes (1 per CPU core):
  - Each runs a single-threaded epoll event loop
  - Each handles thousands of connections
```

**vs Apache:** 1 thread per connection → 10K connections = 10K threads = 10GB stack memory.
**Nginx:** 1 event loop per core → 10K connections on 4 cores = 4 threads total.

## Event Loop (Inside Each Worker)

Same epoll pattern as Redis:
```
epoll_wait() → get ready fds → process each → repeat
```

### State Machine Per Connection

TCP is a byte stream — HTTP request may arrive in chunks across multiple epoll events. Each connection is a state machine:

```
READING_REQUEST     → accumulate headers until \r\n\r\n
CONNECTING_UPSTREAM → non-blocking connect to backend
READING_UPSTREAM    → reading backend response
SENDING_RESPONSE    → writing to client
```

Event loop advances whichever connections are ready. No connection ever blocks the loop.

## Reverse Proxy

```
Client ──HTTPS──→ Nginx ──HTTP──→ Backend
```

Why put Nginx in front:

| Concern | Without Nginx | With Nginx |
|---------|--------------|------------|
| TLS | Every backend handles TLS | Nginx handles once |
| Slow clients | Backend blocked waiting | Nginx buffers, frees backend |
| Static files | Backend wastes CPU | Nginx uses sendfile (zero-copy) |
| Load balancing | Client must know backend IPs | Nginx distributes |

### Slow Client Buffering (Key Feature)

Backend generates 10MB in 50ms → Nginx buffers it → backend is FREE.
Nginx slowly feeds it to slow 3G client. Backend already handling next request.

Even with a single backend, Nginx in front still helps.

## Load Balancing

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

Algorithms:
- **Round Robin** (default): rotate through backends
- **Least Connections**: send to backend with fewest active connections
- **IP Hash**: same client IP → same backend (sticky sessions)
- **Weighted**: assign more traffic to stronger servers

Health checks: detects dead backends, stops sending traffic.

## Static File Serving + sendfile

```nginx
location /static/ {
    sendfile on;       # zero-copy: disk → kernel → NIC (skips Nginx process)
    tcp_nopush on;     # batch headers + body into one TCP packet
    root /var/www;
}
```

## TLS Termination

Nginx handles TLS handshake, certificate management, session resumption, HTTP/2. Backends see plain HTTP — simpler and faster.

TLS handshake cost: ~2ms RSA, ~0.5ms ECDSA. CPU-heavy — one reason Nginx needs multiple workers.

## Kernel Features Used

| Feature | Purpose |
|---------|---------|
| epoll | Event loop, monitor thousands of connections |
| sendfile | Zero-copy static file serving |
| non-blocking I/O | Never block the event loop |
| TCP_NODELAY | Disable Nagle's algorithm for low latency |
| TCP_NOPUSH | Batch headers + body into one packet |
| SO_REUSEPORT | Multiple workers bind to same port, kernel distributes evenly |

**SO_REUSEPORT:** Without it, workers compete for accept() on same socket (thundering herd). With it, kernel distributes incoming connections across workers.

## Why Nginx Needs Multiple Workers but Redis Doesn't

Redis: each op is ~1 microsecond (hash lookup). One core handles 100K+ ops/sec easily.

Nginx: each request involves TLS crypto (~0.5-2ms), 2x I/O (client + backend), HTTP parsing, header manipulation, compression. Much more CPU per request → need all cores.

## Interview Angle
"Nginx is fast because it's an optimal application of the Linux kernel's networking primitives: epoll for I/O multiplexing, sendfile for zero-copy static serving, non-blocking sockets with state machines to never block the event loop, and multi-worker architecture to use all CPU cores. Its killer feature as a reverse proxy is slow-client buffering — it absorbs slow clients so backends stay free."
