# Docker Internals

## What It Is
Container runtime. A container is NOT a VM — it's a regular Linux process with namespaces (isolation) + cgroups (limits) + overlay filesystem (layered images).

## Containers vs VMs

```
VM:        App + Guest OS + virtual hardware per VM (heavy, seconds to start, GBs RAM)
Container: App + libraries only, shares host kernel (light, milliseconds to start, MBs RAM)
```

VMs use a **hypervisor** to emulate hardware. Containers use **kernel features** directly.

## Namespaces — Isolation (What You Can See)

Linux kernel feature (not a Docker invention). Makes process think it's alone on the machine.

| Namespace | Isolates | Example |
|-----------|----------|---------|
| PID | Process IDs | Container sees its app as PID 1 |
| NET | Network stack | Container gets own IP and ports |
| MNT | Filesystem mounts | Container sees own root `/` |
| UTS | Hostname | Container has own hostname |
| IPC | Shared memory | Separate IPC between containers |
| USER | User/group IDs | Root in container ≠ root on host |

Created via `clone()` syscall with namespace flags. Can also use `unshare` command manually.

## cgroups — Resource Limits (What You Can Use)

Linux kernel feature. Limits CPU, memory, disk I/O per process group.

```
K8s resource limits are cgroups under the hood:
  limits.cpu: 500m      → cgroup cpu.quota = 50% of one core
  limits.memory: 256Mi  → cgroup memory.limit = 256Mi (OOMKilled if exceeded)
```

Implemented as virtual filesystem at `/sys/fs/cgroup/`.

## Overlay Filesystem — How Images Work

Image = stack of read-only layers. Each Dockerfile instruction = one layer.

```
Layer 4: index.html        ← COPY (your content)
Layer 3: app.conf          ← COPY (your config)
Layer 2: nginx binaries    ← RUN apt-get install
Layer 1: ubuntu base       ← FROM ubuntu

Container adds writable layer on top (copy-on-write).
```

**Layer sharing:** 10 containers from same image = 1 copy of image layers + 10 thin writable layers.

**Copy-on-Write:** Modify a file from read-only layer → kernel copies file to writable layer first, modifies copy. Original unchanged.

## Build Best Practices

```dockerfile
# GOOD — layer caching optimized
FROM node:18
COPY package*.json ./       # rarely changes → cached
RUN npm install             # cached if package.json unchanged
COPY . .                    # code changes only rebuild from here
CMD ["node", "app.js"]
```

Put least-changing instructions at top, most-changing at bottom.

## Container Runtime Stack

```
Docker CLI → Docker Daemon → containerd → runc → Linux kernel
                                           ↑
                              creates namespaces, cgroups, overlay mount, exec() app
```

- **runc:** low-level, calls kernel syscalls
- **containerd:** lifecycle management (pull, start, stop, logs)
- **Docker:** CLI + build + developer experience
- K8s uses containerd directly (dropped Docker dependency in 1.24)

## Networking

Each container gets own NET namespace + veth pair connecting to docker0 bridge.

```
docker0 bridge (172.17.0.1) — virtual network switch
  ├── veth → Container A (172.17.0.2)
  └── veth → Container B (172.17.0.3)
```

Same bridge = can talk to each other. Host does NAT for external traffic.

## Security Implication

Containers share host kernel. Kernel exploit in one container → compromises entire host + all containers.

VM kernel exploit → only escapes to hypervisor. Other VMs safe (separate kernels).

Mitigations:
- Don't run as root inside containers
- K8s Pod Security Standards
- gVisor (Google) / Firecracker (AWS) — lightweight VM isolation with container speed

## Interview Angle
"A container is just a Linux process with namespace isolation, cgroup resource limits, and an overlay filesystem. Understanding this explains why containers start in milliseconds (no OS boot), why they're less isolated than VMs (shared kernel), and why image layers enable efficient storage sharing. Docker popularized containers, but the real innovation is in Linux kernel features that existed years before Docker."
