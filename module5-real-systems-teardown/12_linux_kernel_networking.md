# Linux Kernel Networking

## What It Is
The networking stack inside the Linux kernel. Every network request (HTTP, DB query, Kafka message) goes through this stack. Understanding it explains why Nginx, Redis, and Node.js are fast.

## Life of a Packet

```
Wire → NIC → DMA → Ring Buffer → Interrupt/NAPI → L2 → L3 → L4 → Socket Buffer → app read()
```

## NIC and DMA

NIC (Network Interface Card) receives bits from the wire. Can't write to app memory directly.

**DMA (Direct Memory Access):** NIC writes packet data directly to kernel memory without involving CPU.
- Without DMA: NIC → CPU → RAM (CPU wastes cycles copying)
- With DMA: NIC → RAM directly, CPU notified after

## Ring Buffer

Pre-allocated circular array in kernel memory. NIC writes packets here via DMA, kernel drains them.

- 1 per NIC, shared across ALL connections
- Contains raw unsorted packets from every connection
- If kernel is slow and buffer fills → packets dropped silently (`ethtool -S eth0 | grep rx_dropped`)
- Size: 256-4096 slots, tunable with `ethtool -G eth0 rx 4096`

## Interrupts and NAPI

After DMA, NIC fires hardware interrupt to notify CPU.

**Problem:** At high traffic (1M pkt/sec), interrupt storms — CPU spends all time handling interrupts.

**NAPI (New API):** Hybrid approach.
- Low traffic: interrupt-driven (responsive)
- High traffic: switches to polling (CPU checks ring buffer periodically)
- When buffer empties → re-enable interrupts

Analogy: doorbell (interrupt) vs. checking mailbox every minute (polling).

## Network Stack Processing

Kernel processes raw packet through layers:
1. **L2 (Ethernet):** Strip MAC header, check if frame is for this machine
2. **L3 (IP):** Strip IP header, routing, iptables/Netfilter rules execute here
3. **L4 (TCP):** Strip TCP header, sequence check, reorder, flow control, send ACK
4. **Socket buffer:** Append payload to the correct connection's buffer

## Socket Buffer

Per-connection buffer holding reassembled, in-order TCP byte stream. App calls `read(fd)` to consume.

- 1 per TCP connection
- Tunable: `sysctl net.ipv4.tcp_rmem` (default: 4KB min, 128KB default, 6MB max)
- If app reads slowly and buffer fills → TCP tells sender to slow down (window = 0) → **back-pressure**

### Ring Buffer vs Socket Buffer

| | Ring Buffer | Socket Buffer |
|---|---|---|
| Count | 1 per NIC | 1 per TCP connection |
| Contents | Raw packets, all connections mixed | Clean byte stream, single connection |
| Full = ? | Packets dropped (silent) | TCP back-pressure (sender slows) |
| Tuned by | `ethtool -G` | `sysctl tcp_rmem` |

## Userspace vs Kernel Space

Memory split into two zones, enforced by CPU:
- **Userspace:** Your apps (Redis, Nginx). Restricted — can't touch hardware.
- **Kernel space:** OS, drivers, TCP stack. Full hardware access.

**Syscall** = the only way to cross the wall. Each crossing costs ~100-200ns (save registers, switch CPU mode, switch back).

## I/O Multiplexing Evolution

### select (1983)
Pass all fds to kernel, kernel scans ALL of them. O(n) per call. Hard limit of 1024 fds.

### poll (1986)
Same as select, no 1024 limit. Still O(n) scan every call.

### epoll (2002)
Callback-based. Register fds once, kernel notifies only when ready.

**How it works (pub/sub inside kernel):**
- `epoll_ctl(ADD, fd)` = subscribe. Registers callback on socket's wait queue.
- Data arrives → callback fires → fd added to ready list.
- `epoll_wait()` = consume events. Returns only ready fds. No scanning.

**Event loop pattern (Redis, Nginx, Node.js):**
```
epoll = epoll_create()
epoll_ctl(ADD, server_socket)    // server socket = listening for new connections

while (true) {
    ready = epoll_wait(epoll)    // sleep until something happens
    for fd in ready {
        if fd == server_socket → accept() new client, add to epoll
        else → read data, process, write response
    }
}
```

Server socket vs client socket:
- **Server socket (listening):** "ready" = new connection waiting → `accept()` produces a client fd
- **Client socket:** "ready" = data available → `read()` to get the data

### io_uring (2019)
Two shared memory ring buffers between userspace and kernel — eliminates syscalls entirely.

- **Submission Queue (SQ):** App writes requests (just memory writes, no syscall)
- **Completion Queue (CQ):** Kernel writes results (just memory writes)
- Both mapped via `mmap` — shared memory visible to both sides

```
epoll + read():  5 ready fds = 5+ syscalls = 5+ wall crossings
io_uring:        5 ready fds = 0 syscalls = write SQ, read CQ (shared memory)
```

Originally designed for disk I/O, expanded to networking. Becoming Linux's universal async I/O interface.

## Evolution Summary

| Year | API | Mechanism | Performance |
|------|-----|-----------|-------------|
| 1983 | select | scan all fds | O(n) per call, max 1024 |
| 1986 | poll | scan all fds | O(n) per call, no limit |
| 2002 | epoll | callbacks, ready list | O(ready), 1 syscall per I/O |
| 2019 | io_uring | shared memory rings | O(ready), 0 syscalls |

Each solves the bottleneck the previous exposed.

## Zero-Copy / sendfile

### The Problem
Sending a file to a network client (e.g., Kafka sending log segments, Nginx serving static files).

**Normal way (read + write):** 4 copies, 2 wall crossings.
```
Disk →DMA→ [Kernel page cache] →CPU→ [App buffer] →CPU→ [Kernel socket buf] →DMA→ NIC
                                  ↑                  ↑
                             wall crossing #1    wall crossing #2
```
App never inspects the data. Round trip through userspace is pointless.

**sendfile:** 2 copies, 0 wall crossings.
```c
sendfile(socket_fd, file_fd, offset, length)
// "kernel, send this file directly to this socket. Leave me out of it."
```
```
Disk →DMA→ [Kernel page cache] →DMA→ NIC
           stays in kernel, never enters userspace
```

### Who Uses It
- **Kafka:** consumers read log segments via sendfile → why consumers are fast
- **Nginx:** `sendfile on;` for static file serving
- **Java NIO:** `FileChannel.transferTo()` calls sendfile under the hood

## Interview Angle
"Every optimization in high-performance servers (Redis, Nginx, Kafka) traces back to the kernel networking stack: epoll avoids scanning, event loops avoid thread-per-connection, sendfile avoids unnecessary copies, and io_uring avoids syscall overhead. Understanding this stack explains WHY these systems are fast, not just HOW to use them."
