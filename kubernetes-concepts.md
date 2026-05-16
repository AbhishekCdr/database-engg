# Kubernetes — A Comprehensive Interview Guide

> A from-the-ground-up reference covering Kubernetes architecture, core objects, networking, storage, security, scheduling, observability, and real-world operational patterns. Every section pairs concepts with concrete YAML, `kubectl` commands, and practical use cases you'd actually encounter in production.

---

## Table of Contents

1. [What Is Kubernetes & Why It Exists](#1-what-is-kubernetes--why-it-exists)
2. [Containers vs VMs vs Bare Metal](#2-containers-vs-vms-vs-bare-metal)
3. [Cluster Architecture — Control Plane & Data Plane](#3-cluster-architecture--control-plane--data-plane)
4. [The API Server, etcd, and Declarative Model](#4-the-api-server-etcd-and-declarative-model)
5. [Pods — The Atomic Unit](#5-pods--the-atomic-unit)
6. [Workload Controllers — Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob](#6-workload-controllers)
7. [Services & Service Discovery](#7-services--service-discovery)
8. [Ingress, Gateway API & Load Balancing](#8-ingress-gateway-api--load-balancing)
9. [Networking Deep Dive — CNI, kube-proxy, NetworkPolicy](#9-networking-deep-dive)
10. [Storage — Volumes, PV, PVC, StorageClass, CSI](#10-storage)
11. [ConfigMaps & Secrets](#11-configmaps--secrets)
12. [Namespaces, Labels, Selectors, Annotations](#12-namespaces-labels-selectors-annotations)
13. [Scheduling — nodeSelector, Affinity, Taints, Tolerations, Topology Spread](#13-scheduling)
14. [Resource Management — Requests, Limits, QoS Classes](#14-resource-management)
15. [Autoscaling — HPA, VPA, Cluster Autoscaler, KEDA](#15-autoscaling)
16. [Health Checks — Liveness, Readiness, Startup Probes](#16-health-checks)
17. [Security — RBAC, ServiceAccounts, Pod Security, Network Policies, Admission Control](#17-security)
18. [Secrets Management & External Secret Stores](#18-secrets-management)
19. [Helm, Kustomize & Package Management](#19-helm-kustomize--package-management)
20. [Operators & Custom Resources (CRDs)](#20-operators--custom-resources)
21. [Observability — Logging, Metrics, Tracing, Events](#21-observability)
22. [Rolling Updates, Rollbacks & Deployment Strategies](#22-rolling-updates-rollbacks--deployment-strategies)
23. [StatefulSets & Running Databases on Kubernetes](#23-statefulsets--running-databases)
24. [Multi-tenancy, Multi-cluster & Federation](#24-multi-tenancy-multi-cluster--federation)
25. [Troubleshooting Playbook](#25-troubleshooting-playbook)
26. [Production Checklist & Common Pitfalls](#26-production-checklist--common-pitfalls)
27. [Real-World Scenarios & System Design](#27-real-world-scenarios--system-design)
28. [Interview Quick-Fire Q&A](#28-interview-quick-fire-qa)

---

## 1. What Is Kubernetes & Why It Exists

**Kubernetes** (often abbreviated **K8s** — eight letters between K and S) is an open-source **container orchestration platform** originally built at Google (inspired by their internal **Borg** system) and donated to the CNCF in 2015.

### The problem it solves

Before Kubernetes, deploying applications meant:
- Provisioning VMs, installing OS, patching, and binding apps to specific hosts.
- Manually scaling — start more VMs, update load balancer config, hope nothing drifts.
- Hand-rolled scripts for failover, rolling updates, secret rotation, log aggregation.
- Dev/prod parity was a fantasy.

Containers (Docker, 2013) made apps portable and lightweight, but a fleet of containers across many hosts needs a brain to:

1. **Schedule** — decide which host runs which container based on resources/constraints.
2. **Heal** — restart failed containers, replace dead nodes.
3. **Scale** — horizontally add/remove replicas as load changes.
4. **Network** — give every container an IP and let services find each other.
5. **Update** — roll out new versions safely with zero downtime.
6. **Configure** — inject env vars, secrets, files without rebuilding images.

That brain is Kubernetes.

### The declarative model — the single most important concept

You don't tell Kubernetes *how* to run your app. You tell it *what* you want ("3 replicas of nginx:1.25 exposed on port 80") in a YAML manifest. Kubernetes' **controllers** continuously reconcile the **actual state** of the cluster toward the **desired state** stored in `etcd`.

> If a pod dies, you didn't ask for that — Kubernetes spawns a replacement. If a node dies, the scheduler reschedules its pods elsewhere. You declare intent; the system maintains it.

This **reconciliation loop** is the core mental model. Every controller in Kubernetes — Deployment, ReplicaSet, Node, Endpoint, even custom ones — runs the same loop:

```
loop forever:
    desired = read from API server
    actual  = observe cluster
    diff    = desired - actual
    if diff: take action to reduce diff
```

---

## 2. Containers vs VMs vs Bare Metal

| | Bare Metal | VMs | Containers |
|---|---|---|---|
| **Isolation** | Physical | Hardware (hypervisor) | OS namespaces + cgroups |
| **Boot time** | Minutes | 30s – 2 min | < 1 second |
| **Overhead** | None | Each VM has full OS | Shares host kernel |
| **Density** | 1 app/host | 10s of VMs/host | 100s – 1000s of containers/host |
| **Portability** | Low | OS image | OCI image runs anywhere |
| **Use case** | DBs needing raw IO | Strong isolation, multi-OS | Microservices, stateless apps |

**A container is just a Linux process** with restricted views of:
- **Namespaces** (PID, NET, MNT, UTS, IPC, USER) — what the process can *see*.
- **cgroups** — what the process can *use* (CPU, memory, IO).
- **chroot/pivot_root** — its filesystem view.

There is no magical "container" syscall. Docker, containerd, CRI-O are tools that bundle these primitives.

---

## 3. Cluster Architecture — Control Plane & Data Plane

A Kubernetes cluster has two planes:

```
┌─────────────────────────  CONTROL PLANE (master nodes) ─────────────────────────┐
│                                                                                  │
│   ┌────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────┐    │
│   │ kube-api   │◄──►│     etcd     │    │ kube-scheduler  │    │ kube-    │    │
│   │  server    │    │ (KV store)   │    │                 │    │ controller│   │
│   │            │    └──────────────┘    └─────────────────┘    │ -manager │    │
│   └─────┬──────┘                                                 └──────────┘   │
│         │                                            ┌─────────────────┐         │
│         │                                            │ cloud-controller│         │
│         │                                            │   -manager      │         │
│         │                                            └─────────────────┘         │
└─────────┼────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────  DATA PLANE (worker nodes) ───────────────────────────┐
│   Node 1                    Node 2                    Node 3                    │
│   ┌─────────────┐           ┌─────────────┐           ┌─────────────┐           │
│   │  kubelet    │           │  kubelet    │           │  kubelet    │           │
│   │  kube-proxy │           │  kube-proxy │           │  kube-proxy │           │
│   │  CRI runtime│           │  CRI runtime│           │  CRI runtime│           │
│   │  ┌──┐ ┌──┐  │           │  ┌──┐ ┌──┐  │           │  ┌──┐       │           │
│   │  │P1│ │P2│  │           │  │P3│ │P4│  │           │  │P5│       │           │
│   │  └──┘ └──┘  │           │  └──┘ └──┘  │           │  └──┘       │           │
│   └─────────────┘           └─────────────┘           └─────────────┘           │
└────────────────────────────────────────────────────────────────────────────────┘
```

### Control Plane Components

| Component | Role |
|---|---|
| **kube-apiserver** | The only component that talks to `etcd`. Front door for all API requests (`kubectl`, controllers, kubelets). REST + watch streams. Stateless and horizontally scalable. |
| **etcd** | Strongly consistent distributed KV store (Raft consensus). The cluster's source of truth — every Pod, Service, Secret lives here. Lose etcd = lose the cluster. Always run 3 or 5 nodes for quorum. |
| **kube-scheduler** | Watches for unscheduled pods, picks a suitable node based on resource requests, affinity, taints, topology spread. Writes its decision back to API server. |
| **kube-controller-manager** | Bundles dozens of controllers — ReplicaSet, Deployment, Node, Job, Endpoint, ServiceAccount, etc. Each runs its own reconciliation loop. |
| **cloud-controller-manager** | Cloud-specific controllers — provisions LoadBalancers, attaches EBS volumes, manages routes. Decoupled so K8s core stays cloud-agnostic. |

### Worker Node Components

| Component | Role |
|---|---|
| **kubelet** | Node agent. Watches API server for pods assigned to its node, pulls images via CRI, starts containers, runs probes, reports status. |
| **kube-proxy** | Maintains network rules (iptables/IPVS/eBPF) so traffic to a Service IP is load-balanced to backing pods. |
| **Container runtime (CRI)** | containerd, CRI-O, or (legacy) Docker. Actually runs containers. |
| **CNI plugin** | Calico, Cilium, Flannel, AWS VPC CNI — gives each pod an IP and sets up routes. |

### Practical example — what happens when you `kubectl apply -f deployment.yaml`

1. `kubectl` POSTs the manifest to **kube-apiserver**, which authenticates (TLS cert / token), authorizes (RBAC), and runs admission controllers.
2. API server persists the Deployment object in **etcd**.
3. **Deployment controller** sees a new Deployment with no ReplicaSet → creates a ReplicaSet.
4. **ReplicaSet controller** sees a RS with `replicas: 3` and 0 pods → creates 3 Pod objects.
5. **Scheduler** sees pods with `nodeName: ""` → picks nodes, patches `nodeName`.
6. **kubelet** on each chosen node sees a pod assigned to it → pulls image via CRI → starts containers → reports status back to API server.
7. **Endpoint controller** sees the new pod IPs match a Service selector → updates Endpoints.
8. **kube-proxy** on every node sees Endpoint changes → updates iptables/IPVS rules.

Every step is a controller reading from and writing to the API server. No component talks directly to another.

---

## 4. The API Server, etcd, and Declarative Model

### The API server is the hub

Everything flows through the API server. It exposes a RESTful HTTP API at `/api/v1/...` for core resources and `/apis/<group>/<version>/...` for extensions.

```
GET  /api/v1/namespaces/default/pods          → list pods
GET  /api/v1/namespaces/default/pods/my-pod   → get one pod
POST /api/v1/namespaces/default/pods          → create
PUT  /api/v1/namespaces/default/pods/my-pod   → replace
PATCH ...                                      → partial update
DELETE ...                                     → delete
```

**Watch** — the killer feature. Clients open a long-lived HTTP connection with `?watch=true` and receive a stream of ADD/MODIFY/DELETE events. This is how every controller stays in sync without polling.

### etcd basics

- **Raft-based** distributed KV store. Leader handles writes; followers replicate.
- Every K8s object is stored under a key like `/registry/pods/default/my-pod`.
- Recommended cluster size: 3 (tolerates 1 failure) or 5 (tolerates 2). Even numbers don't help and hurt write latency.
- **Backup etcd**. Losing etcd loses the entire cluster state. Use `etcdctl snapshot save` regularly. Managed services (EKS, GKE, AKS) handle this for you.

### Declarative > imperative

```bash
# Imperative — discouraged for production
kubectl run nginx --image=nginx:1.25 --replicas=3

# Declarative — version-controlled, reviewable, reproducible
kubectl apply -f deployment.yaml
```

The declarative way means your manifests are the source of truth. Combined with GitOps tools (ArgoCD, Flux), the cluster is just a projection of your Git repo.

---

## 5. Pods — The Atomic Unit

A **Pod** is the smallest deployable unit in Kubernetes. It wraps **one or more tightly coupled containers** that share:

- **Network namespace** — same IP, same port space. Containers in a pod talk via `localhost`.
- **IPC namespace** — can use shared memory, semaphores.
- **Volumes** — declared at pod level, mounted into containers.
- **Lifecycle** — scheduled together, scaled together, terminated together.

> Rule of thumb: if two processes must run on the same machine, share a filesystem, and live/die together, they belong in one pod. Otherwise, separate pods.

### Anatomy of a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests: { cpu: "100m", memory: "128Mi" }
      limits:   { cpu: "500m", memory: "256Mi" }
    livenessProbe:
      httpGet: { path: /healthz, port: 80 }
      periodSeconds: 10
    env:
    - name: NODE_ENV
      value: production
```

### Multi-container patterns (sidecar, ambassador, adapter)

**Sidecar** — augments the main container. Examples:
- Log shipper (Fluent Bit) tailing the app's log files via shared volume.
- Service mesh proxy (Envoy/Istio) handling mTLS and traffic policy.
- Cloud SQL Proxy for DB connections.

**Ambassador** — proxies external connections so the app sees `localhost`.
**Adapter** — reformats the app's output (e.g., scrapes app's `/metrics` and re-exports in Prometheus format).

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - { name: logs, mountPath: /var/log/app }
  - name: log-shipper           # sidecar
    image: fluent/fluent-bit:2.0
    volumeMounts:
    - { name: logs, mountPath: /var/log/app, readOnly: true }
  volumes:
  - name: logs
    emptyDir: {}
```

### Init containers

Run to completion **before** main containers start. Use for migrations, waiting on dependencies, fetching configs.

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
  containers:
  - name: app
    image: myapp:1.0
```

### Pod lifecycle phases

`Pending` → `Running` → `Succeeded` / `Failed` (or `Unknown` if node lost contact).

**Container states** within: `Waiting` (pulling image, mounting volumes), `Running`, `Terminated`.

**Pods are mortal.** They get unique IPs but if a pod dies, its replacement gets a new IP. This is why you put pods behind **Services**.

---

## 6. Workload Controllers

Pods alone don't get rescheduled if they die. You almost always wrap them in a controller.

### Deployment — for stateless apps

The 90% case. Manages ReplicaSets, which manage Pods. Supports rolling updates and rollbacks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # one extra pod during update
      maxUnavailable: 0      # never drop below desired count
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
      - name: api
        image: myorg/api:v2.3.1
        ports: [{ containerPort: 8080 }]
```

### ReplicaSet

The mid-layer. You rarely create one directly — Deployments do it for you. Each new image triggers a new ReplicaSet; old ones are kept (history) for rollback.

### StatefulSet — for stateful apps

For apps needing **stable identity, ordered startup, and persistent per-pod storage**: databases, Kafka, Elasticsearch, Zookeeper.

Differences from Deployment:
- Pods are named `<sset>-0`, `<sset>-1`, ... — predictable hostnames.
- Created/deleted in order (0 first, then 1, ...).
- Each pod gets its own PersistentVolumeClaim from a `volumeClaimTemplates`.
- Requires a **headless Service** for DNS.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres   # headless service
  replicas: 3
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
      - name: postgres
        image: postgres:16
        volumeMounts:
        - { name: data, mountPath: /var/lib/postgresql/data }
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: [ReadWriteOnce]
      resources: { requests: { storage: 100Gi } }
      storageClassName: gp3
```

### DaemonSet — one pod per node

For node-level agents: log shippers, monitoring exporters, CNI plugins, CSI drivers, ingress controllers (sometimes).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector: { matchLabels: { app: node-exporter } }
  template:
    metadata: { labels: { app: node-exporter } }
    spec:
      hostNetwork: true
      containers:
      - name: exporter
        image: prom/node-exporter:v1.7
```

### Job — run to completion

For batch tasks: migrations, ML training, image processing.

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: db-migrate }
spec:
  completions: 1
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrate
        image: myorg/migrator:v1.2
        command: ["./migrate", "up"]
```

`parallelism: N` runs N pods at once; `completions: M` requires M successes.

### CronJob — scheduled jobs

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: nightly-report }
spec:
  schedule: "0 2 * * *"        # 2 AM daily
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: report
            image: myorg/reporter:v1.0
```

### Choosing the right controller

| Use case | Controller |
|---|---|
| Stateless web / API | **Deployment** |
| Database, message queue, distributed system with stable identity | **StatefulSet** |
| Per-node agent (logs, metrics, networking) | **DaemonSet** |
| One-off task | **Job** |
| Recurring task | **CronJob** |
| Single static pod (rare) | **Pod** directly |

---

## 7. Services & Service Discovery

Pods are ephemeral. **Services** give a stable virtual IP and DNS name that load-balances to a set of pods selected by labels.

### How it works

1. You define a Service with a `selector: { app: api }`.
2. The Endpoints/EndpointSlice controller watches pods, finds matches, writes their IPs.
3. `kube-proxy` on every node programs iptables/IPVS rules: traffic to `ClusterIP:port` is DNAT'd to one of the pod IPs.
4. CoreDNS gives the Service a DNS name like `api.default.svc.cluster.local`.

### Service types

| Type | What it does | Use case |
|---|---|---|
| **ClusterIP** (default) | Virtual IP reachable only inside cluster | Internal service-to-service |
| **NodePort** | Exposes on every node's IP at a port (30000–32767) | Quick external access without LB; demos |
| **LoadBalancer** | Provisions a cloud LB (ALB/NLB/GCLB) pointing to NodePorts | External traffic in cloud |
| **ExternalName** | DNS CNAME to an external host | Bridge to external services without IPs |
| **Headless** (`clusterIP: None`) | No VIP; DNS returns pod IPs directly | StatefulSets, custom load balancing |

```yaml
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  type: ClusterIP
  selector: { app: api }
  ports:
  - port: 80           # service port
    targetPort: 8080   # pod port
    protocol: TCP
```

### DNS

CoreDNS runs in `kube-system`. Every Service gets:
- `<svc>.<ns>.svc.cluster.local` → ClusterIP
- For headless: A records for each pod IP.
- For StatefulSet pods: `<pod>.<svc>.<ns>.svc.cluster.local` → pod IP.

Within the same namespace, you can just use `api`. Across namespaces: `api.production`.

### Practical example — service-to-service call

```
[frontend pod]  curl http://api.default.svc.cluster.local
                          │
                CoreDNS resolves to 10.96.45.12 (ClusterIP)
                          │
                kube-proxy iptables DNAT → 10.244.3.7 (pod IP, one of 3 backends)
                          │
                CNI routes to that pod, possibly on another node
```

---

## 8. Ingress, Gateway API & Load Balancing

`LoadBalancer` Services give you one external IP per service — expensive and inflexible. **Ingress** gives you HTTP(S) routing for many services through one entrypoint.

### Ingress

An Ingress is just a *config object*. You also need an **Ingress controller** (nginx-ingress, Traefik, AWS Load Balancer Controller, GKE Ingress) running in the cluster that watches Ingress objects and configures an actual proxy/LB.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts: [app.example.com]
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend: { service: { name: api, port: { number: 80 } } }
      - path: /
        pathType: Prefix
        backend: { service: { name: web, port: { number: 80 } } }
```

### Gateway API — the successor

Ingress is showing its age (annotation-driven, limited features). **Gateway API** (GA in 2023) splits responsibilities:

- **GatewayClass** — defined by infra provider (e.g., "AWS ALB").
- **Gateway** — instance of a class, owned by ops/platform team.
- **HTTPRoute / TCPRoute / GRPCRoute** — owned by app teams, attach to Gateway.

This separation enables multi-team workflows that Ingress couldn't.

### TLS termination

Most Ingress controllers terminate TLS. Use **cert-manager** + Let's Encrypt for automatic cert issuance and renewal.

### When to use what

| Need | Use |
|---|---|
| Internal-only HTTP routing | Service (ClusterIP) + DNS |
| Single external HTTP service | Ingress |
| Many external HTTP services on shared LB | Ingress with hostname/path rules |
| TCP/UDP, advanced routing, multi-team | Gateway API |
| L4 only, simple | Service `LoadBalancer` |

---

## 9. Networking Deep Dive

### The four networking requirements (the "Kubernetes networking model")

1. Every **Pod gets its own IP** (no NAT between pods).
2. Pods can reach all other pods on any node without NAT.
3. Nodes can reach all pods without NAT.
4. The IP a pod sees itself as is the same IP others see it as.

This flat-network model simplifies the developer's mental model but pushes complexity to the **CNI plugin**.

### CNI plugins

| CNI | Approach | Use when |
|---|---|---|
| **Flannel** | Simple overlay (VXLAN) | Learning, small clusters |
| **Calico** | BGP routing, optional overlay; strong NetworkPolicy | Performance, security-heavy |
| **Cilium** | eBPF-based; replaces kube-proxy; service mesh capability | Modern, observability, performance |
| **AWS VPC CNI** | Pods get real VPC IPs from ENIs | EKS, deep AWS integration |
| **Azure CNI / GCE** | Cloud-native | Managed cloud K8s |
| **Weave** | Mesh overlay | Legacy |

### kube-proxy modes

- **iptables** (default) — fast for small clusters, O(N) lookup at scale.
- **IPVS** — kernel L4 LB, better at scale (1000s of services).
- **eBPF** (via Cilium) — bypasses kube-proxy entirely; programmable in-kernel.

### NetworkPolicy

By default, **all pods can talk to all other pods**. NetworkPolicy is the firewall.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: api-allow }
spec:
  podSelector: { matchLabels: { app: api } }
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector: { matchLabels: { app: frontend } }
    ports:
    - { protocol: TCP, port: 8080 }
  egress:
  - to:
    - podSelector: { matchLabels: { app: postgres } }
    ports:
    - { protocol: TCP, port: 5432 }
  - to:                              # allow DNS
    - namespaceSelector: {}
      podSelector: { matchLabels: { k8s-app: kube-dns } }
    ports:
    - { protocol: UDP, port: 53 }
```

> NetworkPolicy is **enforced by the CNI**, not by Kubernetes itself. Flannel doesn't enforce; Calico/Cilium do.

### Service mesh (Istio, Linkerd, Cilium)

Sits above networking, providing:
- mTLS between every pod (zero-trust networking).
- Traffic shaping (canary, A/B, fault injection).
- Observability (golden signals per service).
- Authorization policies at L7 (HTTP method, header).

Cost: complexity, latency overhead (~1ms), resource consumption per pod (sidecar) — though sidecar-less options like Istio Ambient and Cilium Service Mesh are emerging.

---

## 10. Storage

Containers are ephemeral; their writable layer is gone when they restart. For persistence, you mount **volumes**.

### Volume types

| Type | Persistence | Use case |
|---|---|---|
| `emptyDir` | Lifetime of pod | Scratch space, sidecar log sharing |
| `hostPath` | Survives pod, tied to node | Node-level agents (use cautiously — security risk) |
| `configMap` / `secret` | Mounted from API objects | Inject config/credentials as files |
| `persistentVolumeClaim` | Survives pod | Production data |
| `csi` | Cloud / SAN backed | Anything via CSI driver |

### PV, PVC, StorageClass

This is the core abstraction:

- **PersistentVolume (PV)** — a piece of cluster storage (an EBS volume, a CephFS share). Cluster-scoped resource.
- **PersistentVolumeClaim (PVC)** — a request for storage by a pod. Namespace-scoped.
- **StorageClass** — describes a "class" of storage; with `provisioner: ebs.csi.aws.com`, the system **dynamically provisions** PVs as PVCs are created.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 50Gi } }
  storageClassName: gp3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: gp3 }
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer   # bind only after pod is scheduled
reclaimPolicy: Delete                     # or Retain
```

### Access modes

| Mode | Meaning |
|---|---|
| **ReadWriteOnce (RWO)** | One node can mount RW. Most block storage (EBS, GCE PD). |
| **ReadOnlyMany (ROX)** | Many nodes can mount RO. |
| **ReadWriteMany (RWX)** | Many nodes can mount RW. NFS, EFS, CephFS. |
| **ReadWriteOncePod (RWOP)** | Single pod (not just node) can mount RW. Newer, useful for strict ownership. |

### CSI — Container Storage Interface

Standard plugin API so storage vendors don't need K8s code changes. Every modern driver — EBS, EFS, GCE PD, Azure Disk, Portworx, Rook/Ceph, Longhorn — is CSI.

### Volume snapshots & cloning

CSI supports `VolumeSnapshot` and `VolumeSnapshotClass` for point-in-time backups, and PVC cloning (`spec.dataSource: { kind: PersistentVolumeClaim, ... }`) to start a new PVC from an existing one.

### Practical use case — Postgres on K8s

```
StatefulSet (3 replicas)
  ├── pod postgres-0 ── PVC data-postgres-0 ── PV (EBS gp3 100Gi) — AZ-a
  ├── pod postgres-1 ── PVC data-postgres-1 ── PV (EBS gp3 100Gi) — AZ-b
  └── pod postgres-2 ── PVC data-postgres-2 ── PV (EBS gp3 100Gi) — AZ-c
```

Each pod has its own dedicated volume. If a pod is rescheduled, K8s ensures the same PVC follows it (and on AWS, this only works in the same AZ — schedule with `WaitForFirstConsumer`).

---

## 11. ConfigMaps & Secrets

Decouple config from images.

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

Consume as env vars or files:

```yaml
spec:
  containers:
  - name: app
    image: myapp
    envFrom:
    - configMapRef: { name: app-config }
    volumeMounts:
    - { name: cfg, mountPath: /etc/app }
  volumes:
  - name: cfg
    configMap: { name: app-config }
```

### Secret

Same shape as ConfigMap but base64-encoded and treated as sensitive (mounted as `tmpfs` in memory).

```yaml
apiVersion: v1
kind: Secret
metadata: { name: db-creds }
type: Opaque
stringData:                # plaintext at apply time, base64 in storage
  username: admin
  password: s3cr3t
```

> **Important:** Secrets are *base64-encoded, not encrypted*, in etcd by default. Enable **encryption at rest** (`--encryption-provider-config`) using KMS. Better: use **External Secrets Operator** with AWS Secrets Manager / HashiCorp Vault.

### Hot reload caveat

When mounted as a file, ConfigMap/Secret updates propagate to pods within ~60s — but the application must **watch the file** and reload. Env vars are **NOT** updated on the running pod; you must restart pods (e.g., `kubectl rollout restart deployment/api`).

A common trick: use a **checksum annotation** on the pod template so a config change forces a rollout:

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

---

## 12. Namespaces, Labels, Selectors, Annotations

### Namespaces

Logical partitions of a cluster. Use for:
- **Multi-tenancy** (team-a, team-b).
- **Environments** in shared clusters (dev, staging — though separate clusters are safer for prod).
- **RBAC scope** — grant access to just one namespace.
- **Resource quotas** — cap a namespace's CPU/memory/object counts.

Cluster-scoped resources (Nodes, PVs, StorageClasses, ClusterRoles) live outside namespaces.

```bash
kubectl create namespace team-a
kubectl get pods -n team-a
kubectl config set-context --current --namespace=team-a   # default it
```

### Labels

Key/value pairs attached to objects. The **glue** of Kubernetes — selectors use labels to find related objects.

```yaml
metadata:
  labels:
    app: api
    tier: backend
    env: production
    version: v2.3.1
```

### Selectors

```bash
kubectl get pods -l app=api,env=production
kubectl get pods -l 'env in (production, staging)'
kubectl get pods -l '!canary'
```

Used by Services, ReplicaSets, NetworkPolicies, etc., to find their targets.

### Annotations

Same shape as labels but **not selectable**. For metadata: build SHA, owner email, prometheus scrape config, last-applied config.

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    owner: platform-team@example.com
```

### Recommended labels (Kubernetes convention)

```yaml
app.kubernetes.io/name: api
app.kubernetes.io/instance: api-prod
app.kubernetes.io/version: "2.3.1"
app.kubernetes.io/component: backend
app.kubernetes.io/part-of: shop
app.kubernetes.io/managed-by: helm
```

---

## 13. Scheduling

The scheduler picks a node for each pending pod through **filtering** (which nodes can run this?) then **scoring** (which node is best?).

### nodeSelector — simplest

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

### Affinity & Anti-affinity — expressive

**Node affinity** — prefer/require certain nodes:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: [us-east-1a, us-east-1b]
```

**Pod anti-affinity** — spread replicas across nodes/zones:

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: { app: api }
        topologyKey: kubernetes.io/hostname
```

### Taints & Tolerations — node-side filtering

A **taint** on a node *repels* pods that don't tolerate it. Inverse of affinity.

```bash
kubectl taint nodes gpu-node-1 dedicated=ml:NoSchedule
```

```yaml
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: ml
    effect: NoSchedule
```

Effects: `NoSchedule` (block), `PreferNoSchedule` (try not to), `NoExecute` (also evict existing).

### Topology Spread Constraints

Better than anti-affinity for spreading evenly:

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels: { app: api }
```

This says: don't let any zone have more than 1 extra `app=api` pod compared to the least-loaded zone.

### Practical use cases

- **GPU workloads** — taint GPU nodes, only ML pods tolerate.
- **High availability** — anti-affinity + topology spread to distribute across AZs.
- **Cost optimization** — schedule batch jobs on spot instances (taint), web on on-demand.
- **Data locality** — schedule pods near their PVs (handled automatically with `WaitForFirstConsumer`).

---

## 14. Resource Management

Every container should declare:

```yaml
resources:
  requests:                 # scheduler reserves this much
    cpu: "250m"             # 0.25 cores
    memory: "256Mi"
  limits:                   # cgroup hard cap
    cpu: "500m"
    memory: "512Mi"
```

### How requests vs limits behave

| | CPU | Memory |
|---|---|---|
| **Request** | Used by scheduler to bin-pack; guaranteed share | Used by scheduler; no enforcement |
| **Limit** | **Throttled** if exceeded (no kill) | **OOMKilled** if exceeded |

**Memory has no concept of throttling.** If your app spikes past the limit, the kernel kills it.

### QoS classes

Kubernetes assigns each pod a QoS class based on requests/limits:

| Class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | All containers have `requests == limits` for both CPU and memory | Last to be evicted |
| **Burstable** | At least one container has a request but not all match limits | Middle |
| **BestEffort** | No requests or limits anywhere | First to be evicted under pressure |

For production, prefer **Guaranteed** for critical pods.

### LimitRange & ResourceQuota

Cluster admins can enforce policy:

```yaml
apiVersion: v1
kind: LimitRange
metadata: { name: defaults, namespace: dev }
spec:
  limits:
  - type: Container
    default: { cpu: 500m, memory: 512Mi }       # if no limit set
    defaultRequest: { cpu: 100m, memory: 128Mi }
    max: { cpu: "2", memory: 2Gi }
---
apiVersion: v1
kind: ResourceQuota
metadata: { name: team-quota, namespace: dev }
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "50"
    persistentvolumeclaims: "20"
```

### The CPU throttling pitfall

A common bug: setting `cpu: limits: 500m` on a Java/Node service that does CPU spikes during GC or startup → throttling → high p99 latency. Many teams now **set CPU requests but NOT CPU limits** (or only very generous ones), letting workloads burst when nodes have headroom.

---

## 15. Autoscaling

Three dimensions:

### Horizontal Pod Autoscaler (HPA)

Scales **number of replicas** based on CPU, memory, or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 70 }
  - type: Pods
    pods:
      metric: { name: http_requests_per_second }
      target: { type: AverageValue, averageValue: "100" }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # avoid flapping
```

Requires **metrics-server** (CPU/memory) or **Prometheus Adapter / KEDA** (custom metrics).

### Vertical Pod Autoscaler (VPA)

Adjusts **requests/limits** based on observed usage. Great for right-sizing, dangerous for live traffic (in `Auto` mode it restarts pods to apply new sizes). Most teams use VPA in `Off` (recommendation-only) mode.

### Cluster Autoscaler (CA) / Karpenter

Adds/removes **nodes** when pods are unschedulable due to lack of resources. **Karpenter** (AWS, now CNCF) is more flexible and provisions exactly the right instance type per workload — increasingly the default on EKS.

### KEDA — event-driven autoscaling

Scale based on **external events**: Kafka lag, SQS depth, Redis queue size, Prometheus query. Can scale to **zero**, which HPA can't.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: consumer }
spec:
  scaleTargetRef: { name: consumer }
  minReplicaCount: 0
  maxReplicaCount: 50
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: my-group
      topic: events
      lagThreshold: "100"
```

### Order of operations

```
Traffic spikes →
  HPA: more replicas needed →
    Pods unschedulable (not enough nodes) →
      Cluster Autoscaler: add node →
        Pods schedule and serve traffic
```

---

## 16. Health Checks

Three probe types, configured per container:

| Probe | Purpose | Failure action |
|---|---|---|
| **Liveness** | "Is the container alive?" | Restart container |
| **Readiness** | "Is it ready to serve traffic?" | Remove from Service Endpoints |
| **Startup** | "Has it finished starting?" | Defer liveness/readiness |

### When to use each

```yaml
spec:
  containers:
  - name: api
    image: api:1.0
    startupProbe:                     # slow-starting Java/Spring app
      httpGet: { path: /healthz, port: 8080 }
      failureThreshold: 30
      periodSeconds: 10               # allows up to 5 minutes to start
    readinessProbe:                   # is dependency ready?
      httpGet: { path: /ready, port: 8080 }
      periodSeconds: 5
    livenessProbe:                    # is process wedged?
      httpGet: { path: /healthz, port: 8080 }
      periodSeconds: 10
      failureThreshold: 3
```

### Probe types

- `httpGet` — 200–399 = pass.
- `tcpSocket` — connect succeeds = pass.
- `exec` — command exit 0 = pass.
- `grpc` — uses standard gRPC health protocol.

### Common pitfalls

- **Liveness too aggressive** → restart loop (e.g., probe times out under load → kill → reload → probe times out again).
- **Liveness == Readiness** → if the app's struggling, you want to stop sending traffic but keep it alive to recover.
- **Liveness checks downstream deps** → cascading failures. Liveness should only check *this* process.
- **No readiness during slow startup** → traffic hits a not-ready pod → 5xx errors.

### Practical rule

- Readiness checks the app + critical deps (can it serve a real request?).
- Liveness only checks the process itself (a tiny `/livez` that returns 200 unless deadlocked).
- Startup if the boot is slow.

---

## 17. Security

Security in K8s is layered. Get any layer wrong and the others won't save you.

### RBAC (Role-Based Access Control)

Four objects:

| Object | Scope | Use |
|---|---|---|
| **Role** | Namespace | Permissions in one namespace |
| **ClusterRole** | Cluster | Cluster-wide, or template for many namespaces |
| **RoleBinding** | Namespace | Grants Role/ClusterRole to subject in a namespace |
| **ClusterRoleBinding** | Cluster | Grants ClusterRole cluster-wide |

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { namespace: dev, name: pod-reader }
rules:
- apiGroups: [""]
  resources: [pods, pods/log]
  verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { namespace: dev, name: read-pods }
subjects:
- kind: User
  name: alice@example.com
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Subjects** can be `User`, `Group`, or `ServiceAccount`. Always grant **least privilege**.

### ServiceAccounts

Identity for pods to talk to the API server. Each namespace has a `default` SA that's mounted into every pod (unless disabled). Create dedicated SAs per workload:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: api-sa, namespace: prod }
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: api-sa
      automountServiceAccountToken: false   # if pod doesn't need API access
```

On cloud, bind SAs to cloud IAM roles for **IRSA** (EKS), **Workload Identity** (GKE), or **AAD Pod Identity** (AKS) — eliminates static cloud credentials in pods.

### Pod Security Standards (PSS)

Replaced PodSecurityPolicy in K8s 1.25. Three levels enforced via namespace labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

- **privileged** — anything goes.
- **baseline** — minimal restrictions; blocks known privilege escalations.
- **restricted** — strongly hardened; non-root, drop all capabilities, no host paths, seccomp.

For policies beyond PSS, use **OPA Gatekeeper** or **Kyverno**.

### SecurityContext

Per-pod or per-container hardening:

```yaml
spec:
  securityContext:                # pod-level
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile: { type: RuntimeDefault }
  containers:
  - name: app
    securityContext:              # container-level overrides
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities: { drop: [ALL] }
```

### Image security

- **Scan** images (Trivy, Grype, Snyk) in CI.
- **Sign** images (Cosign / Sigstore) and enforce signatures with admission policy.
- **Pin** image tags to digests (`image: nginx@sha256:abc...`) — tags are mutable.
- Use minimal base images (distroless, Alpine, scratch).

### Admission control

After auth/authz, requests pass through admission controllers that can mutate or reject. Examples:

- Built-in: ResourceQuota, LimitRange, ServiceAccount, NamespaceLifecycle.
- **MutatingAdmissionWebhook** — modifies requests (e.g., Istio injecting sidecars).
- **ValidatingAdmissionWebhook** — accepts/rejects (e.g., Kyverno enforcing "no `:latest` tag").
- **ValidatingAdmissionPolicy** (CEL-based, GA in 1.30) — webhook-less validation.

### Secrets at rest

```yaml
# kube-apiserver flag
--encryption-provider-config=/etc/kubernetes/enc/enc.yaml
```

Provider config can use AES-CBC, AES-GCM, or **KMS** (best — wraps DEK with cloud KMS like AWS KMS).

---

## 18. Secrets Management

Built-in Secrets are weak. Mature setups use:

### External Secrets Operator (ESO)

Syncs from external stores (AWS Secrets Manager, Vault, GCP Secret Manager, Azure Key Vault) into K8s Secrets.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-creds }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-sm, kind: SecretStore }
  target: { name: db-creds }
  data:
  - secretKey: password
    remoteRef:
      key: prod/db
      property: password
```

### HashiCorp Vault Agent Injector

Mutating webhook that injects a sidecar fetching secrets from Vault into a shared volume — secrets never become K8s Secrets.

### Sealed Secrets (Bitnami)

Encrypts secrets in Git so manifests can be public. Cluster controller decrypts.

### SOPS + Age/PGP

Encrypt YAML files in Git, decrypted at apply time (by Helm Secrets, ArgoCD plugin, etc.).

> **Rule:** never commit unencrypted secrets to Git. Bots scan repos within minutes.

---

## 19. Helm, Kustomize & Package Management

Pure YAML doesn't scale across environments. Two dominant solutions:

### Helm — templated packaging

Helm "charts" are templated YAML with a values file:

```
mychart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

```bash
helm install my-api ./mychart --values prod-values.yaml
helm upgrade my-api ./mychart --values prod-values.yaml
helm rollback my-api 1
```

Pros: huge ecosystem (Artifact Hub), versioning, templating power. Cons: templating gets messy, debugging Go templates is painful.

### Kustomize — overlays

Built into `kubectl`. No templating; uses **base + overlays**:

```
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
overlays/
├── dev/kustomization.yaml      # patches
└── prod/kustomization.yaml     # patches
```

```yaml
# overlays/prod/kustomization.yaml
resources: [../../base]
namespace: prod
replicas:
- name: api
  count: 10
images:
- name: api
  newTag: v2.3.1
```

Pros: pure YAML, no templating language. Cons: less flexible than Helm.

### GitOps — ArgoCD & Flux

Cluster state mirrors a Git repo. Push to Git → ArgoCD/Flux detects → applies. Benefits:

- Audit log = Git log.
- Rollback = `git revert`.
- No human gets credentials to the cluster.
- Declarative end-to-end.

---

## 20. Operators & Custom Resources

You can extend Kubernetes with **CustomResourceDefinitions (CRDs)** and write **operators** (controllers) for them.

### CRD example

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: { name: backups.db.example.com }
spec:
  group: db.example.com
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    kind: Backup
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              database: { type: string }
              schedule: { type: string }
```

Now `kubectl apply -f backup.yaml` for `kind: Backup` works. But nothing happens unless an operator watches and acts.

### The operator pattern

A controller that encapsulates **operational knowledge** for a specific application: install, scale, backup, upgrade, fail over.

Famous operators:
- **postgres-operator** (Zalando, CrunchyData) — manages Postgres clusters.
- **kafka-operator** (Strimzi) — Kafka clusters.
- **prometheus-operator** — Prometheus + ServiceMonitor CRDs.
- **cert-manager** — automated TLS via Let's Encrypt.
- **cluster-api** — clusters as a CRD (manages clusters via clusters).

Build with **kubebuilder** or **operator-sdk** in Go (most common), or **Metacontroller** / **kopf** in Python.

---

## 21. Observability

Three pillars:

### Logs

- Containers write to stdout/stderr. The container runtime captures them.
- Node-level agent (Fluent Bit, Vector, Promtail, Filebeat) tails `/var/log/pods/...` and ships to a backend (Loki, Elasticsearch, Datadog, CloudWatch).
- Use **structured JSON** logging — much easier to query.

```yaml
# DaemonSet pattern for log collection
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: fluent-bit, namespace: logging }
```

### Metrics

- **Prometheus** scrapes `/metrics` endpoints exposed by your apps and exporters.
- **kube-state-metrics** exposes cluster object state as metrics.
- **node-exporter** exposes node-level OS metrics.
- **cAdvisor** (in kubelet) exposes container metrics.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor                         # from prometheus-operator
metadata: { name: api }
spec:
  selector: { matchLabels: { app: api } }
  endpoints:
  - port: metrics
    interval: 30s
```

**The four golden signals** (Google SRE book): latency, traffic, errors, saturation.

### Traces

- **OpenTelemetry** is the de facto standard.
- Apps emit spans via OTel SDK → OTel Collector (often a DaemonSet) → backend (Jaeger, Tempo, Honeycomb, Datadog).
- Use **propagation headers** (W3C TraceContext) so traces span across services.

### Events

`kubectl get events --sort-by=.lastTimestamp` — invaluable for debugging. Events are stored in etcd and expire after ~1 hour by default. Ship them to your log/metrics backend for retention.

### Dashboards

- **Grafana** — universal dashboarding (works with Prometheus, Loki, Tempo).
- **Kubernetes Dashboard** — built-in web UI (consider security carefully).
- **k9s** — terminal UI for kubectl, beloved by operators.
- **Lens** — desktop UI.

---

## 22. Rolling Updates, Rollbacks & Deployment Strategies

### RollingUpdate (default)

Replace pods incrementally. Controlled by:

- `maxSurge` — extra pods during update (e.g., `25%` or `1`).
- `maxUnavailable` — how many can be missing (e.g., `0` for zero-downtime).

```bash
kubectl set image deployment/api api=myorg/api:v2.4.0
kubectl rollout status deployment/api
kubectl rollout history deployment/api
kubectl rollout undo deployment/api --to-revision=3
kubectl rollout pause deployment/api      # mid-update pause
kubectl rollout resume deployment/api
```

### Recreate

Stop all old pods, then start new ones. Causes downtime. Use only if old and new can't coexist (e.g., DB migration not backward compatible).

### Blue/Green

Run two full environments. Flip Service selector from `version: blue` to `version: green` to cut over.

```yaml
# Service selector
selector:
  app: api
  version: green       # change to switch
```

### Canary

Send a small % of traffic to a new version. Two patterns:

1. **Pod-count canary** — 9 pods of v1, 1 pod of v2 behind same Service. Crude — traffic split = pod ratio.
2. **Service mesh canary** (Istio, Linkerd, Argo Rollouts) — exact percentages via L7 routing.

```yaml
# Argo Rollouts example
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: { duration: 5m }
      - setWeight: 50
      - pause: { duration: 10m }
      - setWeight: 100
      analysis:
        templates:
        - templateName: success-rate
```

### Feature flags

Decouples deploy from release — deploy code dark, flip flag (LaunchDarkly, GrowthBook, Unleash) to enable. Best practice for safe rollouts.

---

## 23. StatefulSets & Running Databases

**Should you run databases on Kubernetes?** — A genuine debate.

**Pros:**
- Same operational model as your apps.
- Operators (CrunchyData, Zalando, Vitess, MongoDB Operator) automate complex tasks.
- Cost — running on your own cluster vs RDS premium.

**Cons:**
- Storage is the hard part. Network-attached storage (EBS, GCE PD) has latency and failure modes.
- Failover, backups, upgrades — operators help but are still complex.
- Managed services (RDS, Cloud SQL, Aurora) are often the right answer.

### What StatefulSet provides

- Stable network identity: `mysql-0`, `mysql-1`, `mysql-2`.
- Stable storage via PVC templates.
- Ordered rolling updates and termination.

### Headless Service (required)

```yaml
apiVersion: v1
kind: Service
metadata: { name: mysql }
spec:
  clusterIP: None         # headless
  selector: { app: mysql }
  ports:
  - { port: 3306 }
```

DNS returns:
- `mysql.default.svc.cluster.local` → list of all pod IPs.
- `mysql-0.mysql.default.svc.cluster.local` → pod 0's IP.

### PodDisruptionBudget — protect availability during voluntary disruptions

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: mysql-pdb }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: mysql } }
```

Prevents `kubectl drain`, autoscaler, etc. from killing pods if it would violate the budget.

### Backup strategies

- **Volume snapshots** (CSI) for crash-consistent block-level snapshots.
- **App-aware backup** (operator-driven) — `pg_basebackup`, `mongodump`.
- Tools: **Velero** (cluster-wide backups including PVs).

---

## 24. Multi-tenancy, Multi-cluster & Federation

### Soft multi-tenancy (one cluster, multiple tenants)

- Namespace per tenant.
- ResourceQuota + LimitRange per namespace.
- NetworkPolicy isolation.
- RBAC scoped to namespace.
- Pod Security Standards.

Risk: shared kernel — a kernel exploit breaks isolation. Not for hostile multi-tenancy.

### Hard multi-tenancy

Cluster per tenant. Or use **virtual clusters** (vcluster) — full K8s API isolation without dedicated nodes.

### Multi-cluster patterns

- **Cluster per environment** (dev, stage, prod) — simple, common.
- **Cluster per region** for HA / latency.
- **Cluster per team / business unit**.

Tools:
- **Argo CD** — manages many clusters from one control plane.
- **Cluster API** — declarative cluster lifecycle.
- **Karmada / KubeFed** — workload federation across clusters.
- **Service mesh (Istio multicluster, Cilium ClusterMesh)** — cross-cluster service discovery and mTLS.

---

## 25. Troubleshooting Playbook

### Pod stuck in Pending

```bash
kubectl describe pod <name>
```

Likely causes:
- **Insufficient resources** — `0/3 nodes are available: insufficient cpu`. → Add nodes / lower requests.
- **No matching nodeSelector / affinity / taint tolerated**.
- **PVC pending** — `WaitForFirstConsumer` is normal until first pod schedules.
- **Image pull secret missing**.

### Pod CrashLoopBackOff

```bash
kubectl logs <pod> --previous          # logs from the crashed instance
kubectl describe pod <pod>             # exit codes, events
```

Common: app misconfig, missing config/secret, OOMKilled, failing migration on startup, liveness too aggressive.

### Pod stuck Terminating

- A finalizer hasn't run (e.g., PV protection finalizer waiting on a controller that's gone).
- Check `kubectl get pod <pod> -o yaml | grep finalizers`.
- Force delete (last resort): `kubectl delete pod <pod> --force --grace-period=0`.

### Service not reachable

```bash
kubectl get svc <name>
kubectl get endpoints <name>           # are there backing pods?
kubectl run -it --rm test --image=curlimages/curl -- sh
# inside pod:
nslookup <svc>
curl <svc>
```

If `Endpoints` is empty → selector doesn't match any ready pod (check labels and readiness probes).

### Image pull errors

```
ErrImagePull / ImagePullBackOff
```
- Wrong image name / tag.
- Missing imagePullSecret for private registry.
- Rate-limited (DockerHub).

### Node NotReady

```bash
kubectl describe node <node>
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
```

Common: kubelet down, network partition, disk full, Docker/containerd hang.

### High latency / errors

1. Check HPA — is it scaling?
2. Check resource saturation (CPU throttling? memory pressure?).
3. Check downstream deps (DB slow queries, network).
4. Inspect probes — readiness flapping causes endpoint churn.

### Useful commands

```bash
kubectl top pods -A --sort-by=cpu
kubectl get events -A --sort-by=.lastTimestamp
kubectl get pods -A --field-selector=status.phase!=Running
kubectl exec -it <pod> -- sh
kubectl debug -it <pod> --image=busybox --target=<container>
kubectl port-forward svc/<name> 8080:80
kubectl logs -f -l app=api --tail=100 --max-log-requests=10
```

---

## 26. Production Checklist & Common Pitfalls

### Checklist

- [ ] **Resource requests/limits** on every container. Memory limit = request (Guaranteed QoS) for critical pods.
- [ ] **Liveness, readiness, (startup) probes** correctly tuned.
- [ ] **PodDisruptionBudget** for every important workload.
- [ ] **Topology spread** across AZs.
- [ ] **HPA** configured.
- [ ] **NetworkPolicy** denying all by default, allowing only what's needed.
- [ ] **RBAC** least-privileged, no use of `cluster-admin`.
- [ ] **Pinned image digests** in production manifests.
- [ ] **Image scanning** in CI; admission policy blocks high-CVE images.
- [ ] **Secrets** sourced from external store (Vault, AWS SM, etc.).
- [ ] **Centralized logs** with retention.
- [ ] **Metrics + alerts** on golden signals.
- [ ] **Distributed tracing** for multi-service calls.
- [ ] **Backups** of etcd and PVs. Restore tested.
- [ ] **Cluster autoscaler / Karpenter** configured with node groups for different workloads (spot, on-demand, GPU).
- [ ] **Pod Security Standards** enforced (`restricted` for prod).
- [ ] **GitOps** for cluster config (no `kubectl apply` from laptops).
- [ ] **Disaster recovery runbook** documented and rehearsed.

### Common pitfalls

| Pitfall | Fix |
|---|---|
| `latest` tag images | Use immutable tags or digests |
| No resource requests | Pods evicted, scheduler can't bin-pack |
| CPU limit set, app throttled | Drop CPU limit, keep request |
| `default` ServiceAccount | Dedicated SA per workload, automount off when not needed |
| Single replica for HA-required app | Min 2-3 replicas + PDB + anti-affinity |
| No NetworkPolicy | Default deny, allow what's needed |
| `kubectl apply` from laptops | GitOps |
| Storing config in container image | ConfigMap / external config |
| Logs to file inside container | Log to stdout |
| Liveness probe hitting DB | Liveness checks process only |

---

## 27. Real-World Scenarios & System Design

### Scenario 1 — Designing a 99.95% web service on EKS

**Requirements:** 50k RPS peak, p95 < 200ms, multi-AZ, zero-downtime deploys.

**Design:**
- 3 AZs, EKS managed control plane.
- Karpenter for nodes; mix of on-demand (web) and spot (batch).
- Deployment, 20 replicas baseline, HPA up to 100 on CPU + custom RPS metric.
- Topology spread across AZs (`maxSkew: 1`, `topologyKey: zone`).
- Anti-affinity to spread across nodes.
- PDB `minAvailable: 80%`.
- AWS Load Balancer Controller → ALB → Ingress → Service → Pods.
- ACM cert on ALB; WAF in front.
- Service mesh (App Mesh or Istio) for mTLS and retries.
- Prometheus + Grafana + AlertManager (kube-prometheus-stack).
- OTel → Tempo for traces.
- Loki for logs.
- ArgoCD for GitOps; promotion via PR (dev → stage → prod overlays).

### Scenario 2 — Migrating a stateful Postgres from RDS to in-cluster

Most teams shouldn't. But if forced (cost, customization):
- Use **CloudNativePG** or **Zalando postgres-operator**.
- StatefulSet with EBS gp3 volumes (provisioned IOPS for hot data).
- 3 replicas across AZs (sync replication for primary/standby).
- Daily logical backup (`pg_dump`) + WAL archival to S3.
- PgBouncer Deployment in front for connection pooling.
- Restore drill quarterly.

### Scenario 3 — Event-driven worker that scales to zero

- Kafka topic `orders`, consumers process and write to DB.
- Deployment with `replicas: 0`.
- KEDA `ScaledObject` triggers on consumer-group lag.
- HPA-managed by KEDA: 0 to 50 replicas based on lag threshold of 100 messages.
- Saves cost during quiet periods (e.g., nights).

### Scenario 4 — Multi-tenant SaaS

- One cluster per region; tenants share clusters.
- Namespace per tenant.
- ResourceQuota: `requests.cpu: 4`, `requests.memory: 8Gi`, `pods: 50`.
- NetworkPolicy: deny all cross-namespace, allow ingress from shared `gateway` namespace.
- OPA Gatekeeper enforces required labels, no privileged pods.
- Per-tenant Ingress with `host: <tenant>.app.example.com`.

### Scenario 5 — Cost-optimized batch processing

- Karpenter provisions spot instances tainted `spot=true:NoSchedule`.
- Batch Jobs tolerate the taint; web doesn't.
- Job uses `restartPolicy: OnFailure`, `backoffLimit: 5`.
- PDB with `maxUnavailable: 100%` so spot interruptions don't block other operations.
- Batch results land in S3; if a node dies, the Job spawns a fresh pod.

---

## 28. Interview Quick-Fire Q&A

**Q: What happens when a pod is OOMKilled?**
A: The kernel OOM killer terminates the container (exit 137). The kubelet restarts it per `restartPolicy` (`Always` for Deployments). Repeated kills → CrashLoopBackOff. Fix by raising memory limit or finding the leak.

**Q: Difference between a Deployment and a StatefulSet?**
A: Deployment is for stateless apps — pods are interchangeable, get random names, no per-pod storage. StatefulSet gives stable identity (`-0`, `-1`), ordered startup/shutdown, and per-pod PVCs from a template — for databases and stateful systems.

**Q: How does Service load balancing work?**
A: kube-proxy on each node programs iptables/IPVS rules: traffic to ClusterIP:port is DNAT'd to one of the endpoint pod IPs (round-robin by default). It's L4, kernel-level — no proxy process in the data path.

**Q: What's the difference between requests and limits?**
A: Requests = scheduler reservation; guarantees the pod gets at least that. Limits = cgroup cap; CPU is throttled, memory triggers OOMKill. Pods with `requests == limits` get Guaranteed QoS class.

**Q: Why use a Service instead of pod IPs directly?**
A: Pod IPs change when pods restart/reschedule. A Service is a stable virtual IP + DNS name with built-in load balancing.

**Q: How do you do zero-downtime deploys?**
A: Rolling update with `maxUnavailable: 0`, proper readiness probes, PreStop hook + `terminationGracePeriodSeconds` to drain in-flight requests, and a PodDisruptionBudget.

**Q: What is a sidecar?**
A: A second container in the same pod that augments the main one — log shipper, mesh proxy, auth proxy. Shares network and volumes with the main container.

**Q: How do ConfigMap updates reach pods?**
A: If mounted as a volume, the file content is updated within ~60s; the app must reload. If used as env vars, only new pods see the change — existing pods need a restart.

**Q: What's the difference between Ingress and a LoadBalancer Service?**
A: LoadBalancer Service is L4 (one external LB per service). Ingress is L7 (HTTP/HTTPS) routing — one entrypoint can route by host/path to many backend services. Ingress requires a controller (nginx, ALB, Traefik).

**Q: What does kube-proxy do?**
A: Watches Services and Endpoints, programs node-level network rules (iptables/IPVS/eBPF) to forward traffic from Service IPs to backing pod IPs. It's not in the data path itself.

**Q: How does the scheduler decide where to place a pod?**
A: Filtering (which nodes can run it — resources, nodeSelector, taints, affinity) then scoring (which is best — least loaded, balanced, topology preferences). Writes the chosen node back to the pod's `spec.nodeName`.

**Q: What is a DaemonSet for?**
A: Running one pod per node — typically node-level agents like log shippers, metric exporters, CNI plugins, CSI drivers.

**Q: How do you give a pod cloud IAM permissions?**
A: Use IRSA on EKS, Workload Identity on GKE, or AAD Pod Identity on AKS. Bind a Kubernetes ServiceAccount to a cloud IAM role; the pod assumes it via projected token, no static keys.

**Q: What's a headless Service?**
A: A Service with `clusterIP: None`. DNS returns A records for each backing pod IP instead of one VIP. Used by StatefulSets for per-pod DNS and by clients doing their own load balancing.

**Q: Liveness vs readiness?**
A: Liveness: is the process alive? Failure → restart. Readiness: is the pod ready to serve? Failure → remove from Service Endpoints. Liveness should be cheap and self-only; readiness can check critical deps.

**Q: What is a PodDisruptionBudget?**
A: Limits how many pods of an app can be voluntarily disrupted (drains, autoscaler) at once. `minAvailable: 2` or `maxUnavailable: 1`. Doesn't apply to involuntary disruptions (node crash).

**Q: How does etcd fit in?**
A: It's the cluster's source of truth — every K8s object lives there. Only kube-apiserver talks to it. Run 3 or 5 nodes for Raft quorum; back it up.

**Q: What is a CRD?**
A: CustomResourceDefinition — extends the K8s API with new object types. Combined with a controller (operator), it lets you manage anything declaratively.

**Q: Difference between Helm and Kustomize?**
A: Helm = templated YAML with a values file, package manager, versioning. Kustomize = pure YAML overlays patching a base. Helm is more flexible; Kustomize is simpler and built into kubectl.

**Q: How do you secure a cluster?**
A: RBAC least-privilege, dedicated ServiceAccounts, Pod Security Standards `restricted`, NetworkPolicy default-deny, image scanning + signed images, secrets via external store with KMS encryption at rest, audit logging, regular CVE patching.

**Q: How do you debug a pod that won't start?**
A: `kubectl describe pod` for events, `kubectl logs --previous` for prior crash logs, check resource availability, image pull errors, missing ConfigMap/Secret, init container failures, probe configuration.

**Q: What is the CNI?**
A: Container Network Interface — the plugin spec for pod networking. Plugins (Calico, Cilium, Flannel, AWS VPC CNI) implement how pods get IPs and how packets flow.

**Q: How does HPA work?**
A: Reads metrics from metrics-server (CPU/memory) or custom adapters. Compares to target (e.g., 70% CPU) and computes desired replica count: `desired = ceil(current * (currentMetric / targetMetric))`. Adjusts the Deployment's replica count.

**Q: When would you NOT use Kubernetes?**
A: Single-machine apps, very small teams without ops capacity, latency-critical single-tenant workloads where overhead matters, simple static sites (use a CDN), or when a managed PaaS (Vercel, Cloud Run, App Runner) covers your needs.

---

## Appendix — Useful kubectl Commands

```bash
# Context & namespaces
kubectl config get-contexts
kubectl config use-context <ctx>
kubectl config set-context --current --namespace=<ns>

# Get / describe / explain
kubectl get pods -o wide
kubectl get all -A
kubectl describe pod <name>
kubectl explain deployment.spec.strategy

# Logs & exec
kubectl logs -f <pod> -c <container>
kubectl logs --previous <pod>
kubectl exec -it <pod> -- /bin/sh
kubectl debug -it <pod> --image=busybox --target=<ctr>

# Apply & diff
kubectl apply -f manifest.yaml
kubectl diff -f manifest.yaml
kubectl apply -k overlays/prod          # kustomize
kubectl delete -f manifest.yaml

# Rollouts
kubectl rollout status deployment/api
kubectl rollout history deployment/api
kubectl rollout undo deployment/api

# Scaling
kubectl scale deployment/api --replicas=5
kubectl autoscale deployment/api --min=3 --max=20 --cpu-percent=70

# Port-forward & proxy
kubectl port-forward svc/api 8080:80
kubectl proxy --port=8001

# Top (requires metrics-server)
kubectl top nodes
kubectl top pods -A --sort-by=memory

# Events & sorting
kubectl get events -A --sort-by=.lastTimestamp

# Field selectors
kubectl get pods --field-selector=status.phase=Running

# JSONPath / output formatting
kubectl get pods -o jsonpath='{.items[*].spec.nodeName}'
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName

# Drain & cordon (node maintenance)
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

---

> **Final advice for interviews:** When asked open-ended questions ("design a system on K8s"), think in terms of the four pillars — workload, networking, storage, security — and walk through how Kubernetes' primitives compose to satisfy each. Always mention failure modes and operational concerns; that's what separates a candidate who's deployed Kubernetes from one who's only read about it.
