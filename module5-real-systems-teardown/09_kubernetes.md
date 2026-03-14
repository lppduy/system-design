# Kubernetes Internals

## What It Is
A container orchestrator — you declare desired state, K8s makes reality match. Solves: "I have 50 services and 20 servers, figure out placement, healing, and scaling."

## Architecture

```
Control Plane:
  API Server    → single gateway, only thing that talks to etcd
  Scheduler     → assigns unscheduled pods to nodes (filter → score → bind)
  Controller Mgr → runs reconciliation loops (desired vs actual state)
  etcd          → distributed KV store (Raft consensus), stores ALL cluster state

Worker Nodes:
  kubelet       → agent on each node, pulls images, starts containers, health checks
  kube-proxy    → maintains iptables/IPVS rules for service routing
```

## etcd — The Brain
Stores all cluster state as key-value pairs. Uses Raft consensus (leader election, majority writes).
- Has **watch** mechanism — components subscribe to changes (no polling)
- Typically 3 or 5 nodes. Lose majority → cluster down.
- Only API Server talks to etcd directly.

## Reconciliation Loop (Core Pattern)
Every controller runs: `desired state (etcd) vs actual state → take action to converge`
- ReplicaSet controller: "need 3 pods, see 2 → create 1"
- Node controller: "no heartbeat from node-3 → mark unhealthy, reschedule pods"
- Deployment controller: "image changed → rolling update via new ReplicaSet"

## Scheduler
1. **Filter:** eliminate nodes that can't fit the pod (insufficient CPU/memory)
2. **Score:** rank remaining nodes (resource balance, affinity rules, spread)
3. **Bind:** assign pod to highest-scoring node

CPU measured in millicores: 500m = 0.5 core, 1000m = 1 core.

## Service Discovery
- **CoreDNS:** every service gets DNS entry: `service-name.namespace.svc.cluster.local`
- **kube-proxy:** iptables/IPVS rules DNAT ClusterIP to actual pod IPs, load balanced

## Workload Resources

| Resource | Use Case | Key Property |
|----------|----------|-------------|
| Deployment | Stateless apps (APIs, web) | Pods are interchangeable, random names |
| StatefulSet | Stateful apps (DBs, Kafka) | Stable names (mysql-0), own storage, ordered start |
| DaemonSet | Node agents (logging, monitoring) | One pod per node |
| Job/CronJob | Batch/scheduled tasks | Run once or on schedule |

### Deployment vs StatefulSet
- Deployment: pods are cattle — any can replace any other
- StatefulSet: pods are pets — identity matters (mysql-0 is primary, mysql-1 replicates from it)

## Resource Requests vs Limits
```
requests: minimum guaranteed, used by scheduler for placement
limits:   maximum allowed, enforced by kernel (CPU throttled, memory OOMKilled)
```

## Services — Exposing Pods

| Type | Scope | How |
|------|-------|-----|
| ClusterIP | Internal only | Stable IP, DNS resolves to it |
| NodePort | External via node IP | Node:30000-32767 → Service → Pod |
| LoadBalancer | External via cloud LB | Cloud LB → Service → Pod |
| Ingress | HTTP routing | Path/host-based routing (usually Nginx controller) |

## Probes — Health Checks
- **startupProbe:** "finished starting?" → fail = keep waiting
- **readinessProbe:** "can serve traffic?" → fail = remove from LB (don't restart)
- **livenessProbe:** "still alive?" → fail = kill and restart

## ConfigMaps & Secrets
- ConfigMap: non-sensitive config (env vars, config files)
- Secret: sensitive data (base64 encoded, NOT encrypted by default)
- Both injected into pods as env vars or mounted files

## Persistent Volumes
```
Pod → PVC (claim) → PV (actual storage) → disk
StorageClass defines disk type (SSD, HDD)
```

## CKAD Speed Tricks
```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment my-app --image=app --dry-run=client -o yaml > deploy.yaml
kubectl set image deployment/my-app my-app=my-app:v3
kubectl rollout undo deployment/my-app
```

## Why etcd Over ZooKeeper
Single interface principle: everything goes through API Server → etcd. No second coordination system to manage. Kafka made same move: ZooKeeper → KRaft.

## Interview Angle
"K8s is a declarative reconciliation system. You describe desired state, controllers continuously compare it to reality and take corrective action. etcd stores truth, API Server is the single gateway, scheduler places pods, kubelet enforces on each node. The reconciliation loop pattern is the key insight — it's what makes K8s self-healing."
