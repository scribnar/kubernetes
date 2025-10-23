# etcd Requirements for Kubernetes

**Version**: 1.0
**Last Updated**: 2025-10-21
**Related Documents**: [00-README.md](00-README.md), [02-FUNCTIONAL-SPEC.md](02-FUNCTIONAL-SPEC.md)

---

## Table of Contents

- [Overview](#overview)
- [Why Kubernetes Needs a Database](#why-kubernetes-needs-a-database)
- [Why etcd Was Chosen](#why-etcd-was-chosen)
- [Core Requirements](#core-requirements)
- [Consistency Requirements](#consistency-requirements)
- [Performance Requirements](#performance-requirements)
- [High Availability Requirements](#high-availability-requirements)
- [Scalability Requirements](#scalability-requirements)
- [Security Requirements](#security-requirements)
- [Operational Requirements](#operational-requirements)
- [Requirements Comparison Matrix](#requirements-comparison-matrix)
- [Trade-offs and Design Decisions](#trade-offs-and-design-decisions)
- [Summary](#summary)

---

## Overview

This document explains **why** Kubernetes needs etcd and what specific requirements drove the selection of etcd as Kubernetes' storage backend. Understanding these requirements helps operators, developers, and SREs make informed decisions about deployment, configuration, and troubleshooting.

### Key Questions Answered

1. Why does Kubernetes need a distributed database at all?
2. What specific properties must the storage system provide?
3. Why was etcd chosen over alternatives (ZooKeeper, Consul, etc.)?
4. What are the non-functional requirements (performance, HA, security)?
5. What trade-offs were made in the design?

---

## Why Kubernetes Needs a Database

### The Distributed Systems Problem

Kubernetes is a **distributed system** that manages containerized applications across multiple nodes. This creates fundamental challenges:

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        API[kube-apiserver<br/>Node 1]
        API2[kube-apiserver<br/>Node 2]
        API3[kube-apiserver<br/>Node 3]

        CM[Controller Manager]
        Sched[Scheduler]
        Kubelet1[Kubelet - Node 4]
        Kubelet2[Kubelet - Node 5]
    end

    API --> DB[Shared State?]
    API2 --> DB
    API3 --> DB
    CM --> DB
    Sched --> DB
    Kubelet1 --> DB
    Kubelet2 --> DB

    style DB fill:#ff6b6b
```

**Problem**: Multiple components need to:
- Read and write cluster state concurrently
- React to state changes in real-time
- Maintain consistency across failures
- Avoid conflicts and race conditions

### What State Needs to be Stored?

```mermaid
graph LR
    subgraph "Cluster State in etcd"
        A[API Objects] --> B[Pods]
        A --> C[Services]
        A --> D[Deployments]
        A --> E[ConfigMaps]
        A --> F[Secrets]

        G[Cluster Config] --> H[Node info]
        G --> I[API versions]
        G --> J[Admission config]

        K[Coordination] --> L[Leases]
        K --> M[Endpoints]
        K --> N[Events]
    end
```

**Categories of Data**:

1. **API Objects**: All Kubernetes resources (Pods, Services, etc.)
2. **Configuration**: ClusterRoles, admission configuration, API server settings
3. **Coordination**: Leader election leases, endpoint slices
4. **Operational Data**: Events, audit logs (if configured)

**Example Pod Object in etcd**:
```
Key: /registry/pods/default/nginx-abc123
Value: <Protobuf-encoded Pod struct>
Size: ~2-10 KB (depending on spec complexity)
```

### Requirements Summary

| Category | Requirement | Why Needed |
|----------|-------------|------------|
| **Consistency** | Strong consistency | Avoid split-brain, ensure single source of truth |
| **Availability** | High availability | Cluster must survive node failures |
| **Durability** | Data persistence | State must survive restarts |
| **Performance** | Low latency reads/writes | Fast API responses |
| **Watch** | Real-time notifications | Controllers need immediate updates |
| **Transactions** | Atomic operations | Prevent race conditions |
| **Scale** | Handle 1000s of nodes | Enterprise clusters |

---

## Why etcd Was Chosen

### Decision Criteria (2014-2015)

When Kubernetes was being designed, several distributed databases were evaluated:

```mermaid
graph TB
    A[Requirement] --> B{Evaluate Options}

    B --> C[etcd]
    B --> D[ZooKeeper]
    B --> E[Consul]
    B --> F[Custom Solution]

    C --> C1[✓ Strong consistency<br/>✓ Watch API<br/>✓ Simple HTTP API<br/>✓ Raft consensus]
    D --> D1[✓ Strong consistency<br/>✓ Mature<br/>✗ Complex API<br/>✗ JVM dependency]
    E --> E1[✓ Service discovery<br/>✓ HTTP API<br/>✗ Eventual consistency]
    F --> F1[✗ High development cost<br/>✗ Maintenance burden]

    style C fill:#90EE90
    style C1 fill:#90EE90
```

### Why etcd Won

#### 1. Strong Consistency via Raft

etcd uses the **Raft consensus algorithm**, which provides:
- **Linearizable reads/writes**: All operations appear to happen atomically
- **Leader election**: Automatic failover when leader fails
- **Log replication**: Changes propagate to all nodes

**Code Reference**: Kubernetes relies on this guarantee in `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:520` (GuaranteedUpdate)

```mermaid
sequenceDiagram
    participant Client
    participant Leader
    participant Follower1
    participant Follower2

    Client->>Leader: Write Request
    Leader->>Leader: Append to log
    Leader->>Follower1: Replicate log entry
    Leader->>Follower2: Replicate log entry
    Follower1-->>Leader: ACK
    Follower2-->>Leader: ACK
    Leader->>Leader: Commit (quorum reached)
    Leader-->>Client: Success

    Note over Leader,Follower2: Quorum = (3/2)+1 = 2 nodes
```

**Why This Matters**: If two kube-apiserver instances try to create the same resource, Raft ensures only one succeeds.

#### 2. Watch API

etcd provides a **native watch API** that perfectly matches Kubernetes needs:

```go
// etcd watch API
watchChan := client.Watch(ctx, "/registry/pods/", clientv3.WithPrefix())
for watchResp := range watchChan {
    for _, event := range watchResp.Events {
        switch event.Type {
        case mvccpb.PUT:
            // Handle create/update
        case mvccpb.DELETE:
            // Handle delete
        }
    }
}
```

**Why This Matters**: Controllers (Deployment, ReplicaSet, etc.) use watch to react to changes instantly.

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:65`

#### 3. Simplicity

Compared to alternatives:
- **etcd**: Single Go binary, no dependencies
- **ZooKeeper**: Requires JVM, complex configuration
- **Consul**: More focused on service discovery than storage

#### 4. HTTP/gRPC API

etcd provides both:
- **HTTP/JSON API**: Easy debugging with curl
- **gRPC API**: High performance for production

```bash
# HTTP API (debugging)
curl -L http://127.0.0.1:2379/v3/kv/range \
  -X POST -d '{"key": "Zm9v"}'  # base64("foo")

# gRPC API (production)
# Used by Kubernetes clientv3 library
```

#### 5. Multi-Version Concurrency Control (MVCC)

etcd's MVCC system provides:
- **Historical queries**: Read data from any past revision
- **Efficient watch**: Watch from specific revision
- **Optimistic concurrency**: Compare-and-swap operations

**Example**:
```go
// Read object at revision 100
resp, _ := client.Get(ctx, key, clientv3.WithRev(100))

// Watch from revision 100 onwards
client.Watch(ctx, key, clientv3.WithRev(100))
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:348`

---

## Core Requirements

### Requirement 1: Strong Consistency

**Requirement**: All reads must reflect the latest committed writes.

**Why Critical**: Prevents inconsistent cluster state that could lead to:
- Duplicate Pod scheduling
- Lost updates (two updates overwrite each other)
- Split-brain scenarios (different views of reality)

**How etcd Satisfies**:
```mermaid
sequenceDiagram
    participant API1 as API Server 1
    participant API2 as API Server 2
    participant etcd

    API1->>etcd: Create Pod "nginx"
    etcd->>etcd: Commit via Raft quorum
    etcd-->>API1: Success (Rev 100)

    API2->>etcd: Create Pod "nginx"
    etcd->>etcd: Check: Pod already exists
    etcd-->>API2: Error: AlreadyExists

    Note over etcd: Linearizability ensures<br/>consistent view
```

**Consistency Guarantee**: **Linearizable** (strongest consistency)
- All operations appear to execute atomically
- Operations respect real-time ordering

**Code Example** (`staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:240`):
```go
// Create operation with uniqueness check
func (s *store) Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error {
    // ...
    txn := s.client.Txn(ctx).If(
        notFound(key), // Ensure key doesn't exist
    ).Then(
        clientv3.OpPut(key, encoded, opts...),
    )
    txnResp, err := txn.Commit()
    if !txnResp.Succeeded {
        return storage.NewKeyExistsError(key, 0)
    }
    // ...
}
```

### Requirement 2: Durability

**Requirement**: Data must survive process crashes and restarts.

**Why Critical**: Losing cluster state would be catastrophic:
- All Pods, Services, Deployments lost
- Security policies gone
- Cluster completely reset

**How etcd Satisfies**:
- **Write-Ahead Log (WAL)**: All changes written to disk before ack
- **fsync**: Force flush to disk on commit
- **Snapshots**: Periodic database snapshots for recovery

```mermaid
graph LR
    A[Write Request] --> B[Append to WAL]
    B --> C[fsync to disk]
    C --> D[Apply to state machine]
    D --> E[Respond to client]

    style C fill:#90EE90
```

**Configuration**:
```bash
# etcd startup flags
--wal-dir=/var/lib/etcd/wal
--data-dir=/var/lib/etcd/data
--snapshot-count=10000  # Snapshot every 10k transactions
```

**Best Practice**: Use SSD storage for WAL and data directories to ensure low fsync latency.

**See Also**: [middle-level/07-performance-tuning.md](middle-level/07-performance-tuning.md)

### Requirement 3: Real-Time Watch

**Requirement**: Clients must receive immediate notifications of changes.

**Why Critical**: Kubernetes controllers are reactive:
- Deployment controller watches ReplicaSets
- ReplicaSet controller watches Pods
- Node controller watches Node status
- **Latency matters**: Slow watch → slow reaction → poor user experience

**How etcd Satisfies**:

```mermaid
sequenceDiagram
    participant Controller
    participant etcd

    Controller->>etcd: Watch /registry/pods/ from Rev=100
    etcd-->>Controller: Stream opened

    Note over etcd: Pod created (Rev=101)
    etcd-->>Controller: Event: PUT, Rev=101

    Note over etcd: Pod updated (Rev=102)
    etcd-->>Controller: Event: PUT, Rev=102

    Note over etcd: Pod deleted (Rev=103)
    etcd-->>Controller: Event: DELETE, Rev=103
```

**Watch Features**:
1. **Watch from revision**: Resume from any point in history
2. **Prefix watch**: Watch all keys with prefix (e.g., `/registry/pods/`)
3. **Reliable delivery**: Events delivered in order
4. **Efficient**: gRPC streaming (not polling)

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:153`

### Requirement 4: Atomic Transactions

**Requirement**: Multiple operations must execute atomically (all or nothing).

**Why Critical**: Many Kubernetes operations require atomicity:
- **GuaranteedUpdate**: Read-modify-write must be atomic
- **Resource creation**: Check for existence + create
- **Cascading delete**: Delete parent + update children

**How etcd Satisfies**: Transaction API with compare-and-swap

```go
// Transaction structure
txn := client.Txn(ctx).
    If(
        // Conditions (all must be true)
        clientv3.Compare(clientv3.ModRevision(key), "=", expectedRev),
    ).
    Then(
        // Operations if conditions pass
        clientv3.OpPut(key, newValue),
    ).
    Else(
        // Operations if conditions fail
        clientv3.OpGet(key),
    )
```

**Real-World Example**: Updating a Pod's status

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:520
func (s *store) GuaranteedUpdate(/*...*/) error {
    for {
        // 1. Read current object
        current, err := s.Get(ctx, key, ...)

        // 2. Apply user's update function
        updated, _, err := tryUpdate(current, ...)

        // 3. Atomic update with version check
        txn := s.client.Txn(ctx).If(
            clientv3.Compare(clientv3.ModRevision(key), "=", currentRevision),
        ).Then(
            clientv3.OpPut(key, encode(updated)),
        )

        resp, err := txn.Commit()
        if resp.Succeeded {
            return nil  // Success
        }
        // Conflict: retry with new version
    }
}
```

---

## Consistency Requirements

### CAP Theorem and Kubernetes

The **CAP theorem** states that a distributed system can provide at most 2 of 3:
- **C**: Consistency (all nodes see the same data)
- **A**: Availability (system responds to requests)
- **P**: Partition tolerance (system works despite network failures)

**Kubernetes Choice**: **CP** (Consistency + Partition Tolerance)

```mermaid
graph TB
    A[CAP Theorem] --> B{Choose 2}
    B --> C[CP: Consistency + Partition<br/>✓ Chosen by Kubernetes]
    B --> D[AP: Availability + Partition<br/>✗ Not suitable]
    B --> E[CA: Consistency + Availability<br/>✗ Impossible in distributed systems]

    style C fill:#90EE90
```

**Why CP?**
- **Consistency is critical**: Wrong scheduling decisions could violate constraints
- **Partition tolerance required**: Network failures happen in real clusters
- **Availability trade-off acceptable**: During network partition, better to reject requests than provide inconsistent data

**Real-World Impact**:
```
Network Partition Scenario:
┌─────────────┐      ╳╳╳╳╳╳╳╳      ┌─────────────┐
│  etcd 1,2   │      Network       │   etcd 3    │
│  (Quorum)   │      Split         │ (Minority)  │
└─────────────┘                    └─────────────┘
      ✓                                   ✗
  Accepts writes                   Rejects writes

Result: Majority partition continues operating
        Minority partition rejects requests (maintains consistency)
```

### Consistency Levels

etcd supports multiple consistency levels:

| Level | Guarantee | Use Case | Performance |
|-------|-----------|----------|-------------|
| **Linearizable** | Strongest | Writes, critical reads | Slower (requires quorum) |
| **Serializable** | Weaker | Non-critical reads | Faster (local reads allowed) |

**Linearizable** (default for Kubernetes):
```go
// Read requires quorum
resp, _ := client.Get(ctx, key)  // Linearizable read
```

**Serializable** (optimization):
```go
// Read from any replica (may be stale)
resp, _ := client.Get(ctx, key, clientv3.WithSerializable())
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:348`

**Trade-off**:
- Linearizable: Consistent but slower
- Serializable: Faster but may be stale

**See Also**: [middle-level/04-transactions-consistency.md](middle-level/04-transactions-consistency.md)

---

## Performance Requirements

### Latency Requirements

Kubernetes requires **low-latency** storage operations for good user experience:

```mermaid
graph LR
    A[kubectl create pod] --> B[API Server]
    B --> C[Validate]
    C --> D[Admit]
    D --> E[etcd Write]
    E --> F[Response]

    style E fill:#ff6b6b
    Note1[Target: <100ms total]
```

**Latency Breakdown**:
| Operation | Target Latency | etcd Component |
|-----------|----------------|----------------|
| **Write** | < 10ms | fsync + quorum |
| **Read** | < 5ms | Local or quorum read |
| **Watch event** | < 100ms | Event delivery |

**Real-World Measurements** (3-node cluster, SSD, 1Gbps network):
```
etcdctl check perf

 60 / 60 Boooooooooooooooooooooooooooooooooooooooooooooooooooooooo! 100.00%
PASS: Throughput is 150 writes/s
PASS: Slowest request took 8.3ms
PASS: Stddev is 2.1ms
```

**Performance Factors**:

1. **Disk I/O**: WAL fsync is often the bottleneck
   ```bash
   # Measure disk fsync latency
   fio --filename=/var/lib/etcd/test --size=100m --direct=1 \
       --rw=write --bs=4k --ioengine=sync --iodepth=1 --numjobs=1
   ```

   **Target**: < 1ms for fsync on SSD

2. **Network Latency**: Raft replication between peers
   ```bash
   # Measure peer RTT
   ping <peer-ip>
   ```

   **Target**: < 2ms RTT within datacenter

3. **CPU**: Encoding/decoding, encryption
   - Modern CPUs handle this easily
   - Rarely a bottleneck

**See Also**: [middle-level/07-performance-tuning.md](middle-level/07-performance-tuning.md)

### Throughput Requirements

Kubernetes clusters generate significant load:

**Write Operations**:
```
Small cluster (10 nodes):
- Pod updates: ~100-500 writes/sec
- Events: ~50-100 writes/sec
- Leases (leader election): ~10-20 writes/sec
Total: ~200-700 writes/sec

Large cluster (1000 nodes):
- Pod updates: ~5,000-10,000 writes/sec
- Events: ~1,000-2,000 writes/sec
- Leases: ~100-200 writes/sec
Total: ~6,000-12,000 writes/sec
```

**Read Operations**:
```
Most reads served from watch cache (in-memory)
Direct etcd reads: ~100-1,000 reads/sec (mostly List operations)
```

**etcd Capacity**:
| Cluster Size | Write Throughput | Read Throughput |
|--------------|------------------|-----------------|
| 3 nodes (SSD) | ~10,000 writes/s | ~50,000 reads/s |
| 5 nodes (SSD) | ~12,000 writes/s | ~80,000 reads/s |

**Bottleneck**: Writes are limited by Raft quorum (slower than reads)

### Database Size Requirements

etcd database size impacts performance:

**Typical Sizes**:
```
Small cluster (10 nodes, 100 pods):
- Database size: ~50-100 MB
- Compaction needed: Weekly

Medium cluster (100 nodes, 1000 pods):
- Database size: ~500 MB - 1 GB
- Compaction needed: Daily

Large cluster (1000 nodes, 10,000 pods):
- Database size: ~5-10 GB
- Compaction needed: Hourly
```

**Why Size Matters**:
1. **Memory**: etcd keeps index in memory (RAM = ~2x DB size)
2. **Watch**: Larger history → more events to process
3. **Compaction**: Larger DB → longer compaction time

**Best Practice**: Keep database < 8 GB via compaction

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/compact.go:45`

**See Also**: [middle-level/03-compaction-defrag.md](middle-level/03-compaction-defrag.md)

---

## High Availability Requirements

### Fault Tolerance

Kubernetes clusters must survive node failures:

```mermaid
graph TB
    subgraph "3-Node etcd Cluster"
        N1[Node 1<br/>HEALTHY] --- N2[Node 2<br/>HEALTHY]
        N2 --- N3[Node 3<br/>FAILED ✗]
    end

    Note1[Quorum: 2/3<br/>Status: OPERATIONAL ✓]

    style N1 fill:#90EE90
    style N2 fill:#90EE90
    style N3 fill:#ff6b6b
```

**Quorum Formula**: `(n/2) + 1`

| Cluster Size | Quorum Needed | Fault Tolerance |
|--------------|---------------|-----------------|
| 1 node | 1 | 0 (not HA) |
| 3 nodes | 2 | 1 node failure |
| 5 nodes | 3 | 2 node failures |
| 7 nodes | 4 | 3 node failures |

**Recommendation**: **3 or 5 nodes** for production
- 3 nodes: Sufficient for most cases, lower cost
- 5 nodes: Better availability, higher cost
- 7+ nodes: Rarely needed (write latency increases)

**Why Odd Numbers?**
```
3 nodes: Tolerates 1 failure (2/3 quorum)
4 nodes: Tolerates 1 failure (3/4 quorum)  ← Same tolerance, more cost!

5 nodes: Tolerates 2 failures (3/5 quorum)
6 nodes: Tolerates 2 failures (4/6 quorum)  ← Same tolerance, more cost!
```

**Even numbers provide no additional fault tolerance.**

### Leader Election and Failover

etcd uses Raft leader election:

```mermaid
sequenceDiagram
    participant N1 as Node 1 (Leader)
    participant N2 as Node 2 (Follower)
    participant N3 as Node 3 (Follower)

    Note over N1,N3: Normal operation

    N1->>N2: Heartbeat
    N1->>N3: Heartbeat
    N2-->>N1: ACK
    N3-->>N1: ACK

    Note over N1: Leader FAILS

    Note over N2,N3: Election timeout

    N2->>N2: Start election
    N2->>N3: RequestVote
    N3-->>N2: Vote granted
    N2->>N2: Become leader

    Note over N2: New Leader elected

    N2->>N3: Heartbeat (as leader)
```

**Timeouts** (etcd defaults):
- **Election timeout**: 1000ms (if no heartbeat, start election)
- **Heartbeat interval**: 100ms (leader sends heartbeat)

**Failover Time**: Typically 1-2 seconds

**Code Reference**: etcd internal Raft implementation

### Split-Brain Prevention

**Problem**: Network partition creates two isolated groups

```mermaid
graph TB
    subgraph "Before Partition"
        N1[Node 1<br/>Leader] --- N2[Node 2<br/>Follower]
        N2 --- N3[Node 3<br/>Follower]
    end

    subgraph "After Network Partition"
        P1[Node 1<br/>No Quorum ✗]
        P2[Node 2<br/>Leader ✓]
        P3[Node 3<br/>Follower ✓]

        P2 --- P3
    end

    style P1 fill:#ff6b6b
    style P2 fill:#90EE90
    style P3 fill:#90EE90
```

**Protection**: Raft quorum requirement
- Minority partition (1 node): Cannot elect leader, rejects writes
- Majority partition (2 nodes): Elects new leader, accepts writes

**Result**: No split-brain (only one partition operational)

**See Also**: [middle-level/05-cluster-management.md](middle-level/05-cluster-management.md)

---

## Scalability Requirements

### Cluster Size Limits

etcd has practical scaling limits:

| Metric | Recommended Limit | Hard Limit |
|--------|-------------------|------------|
| **Database size** | < 8 GB | 10 GB |
| **Key-value pairs** | < 1 million | ~5 million |
| **Watch clients** | < 10,000 | Limited by memory |
| **Write rate** | < 10,000 writes/s | ~15,000 writes/s |
| **Concurrent transactions** | < 1,000 | Limited by CPU |

**Kubernetes Scaling**:

```
Kubernetes Cluster Scale → etcd Load

100 nodes, 1,000 pods:
- Objects in etcd: ~10,000
- Database size: ~500 MB
- Write rate: ~500 writes/s
- Watch clients: ~100
→ Well within limits

1,000 nodes, 10,000 pods:
- Objects in etcd: ~100,000
- Database size: ~5 GB
- Write rate: ~5,000 writes/s
- Watch clients: ~500
→ Approaching limits, tune carefully

5,000+ nodes:
- Consider etcd optimization
- Use watch cache aggressively
- Increase compaction frequency
→ At the edge of single-etcd-cluster scaling
```

### Scaling Strategies

When approaching limits:

1. **Vertical Scaling** (recommended first)
   ```
   - Increase CPU: Handle more concurrent operations
   - Increase RAM: Cache more data
   - Use faster disks: SSDs or NVMe
   - Improve network: 10Gbps links
   ```

2. **Optimize Kubernetes**
   ```
   - Enable watch cache (enabled by default)
   - Reduce event retention
   - Compact more frequently
   - Limit object sizes (avoid large ConfigMaps)
   ```

3. **Horizontal Scaling** (limited benefits)
   ```
   - 3 → 5 nodes: Better availability, slightly better read throughput
   - 5 → 7 nodes: Diminishing returns, higher write latency
   ```

   **Note**: More nodes ≠ more write capacity (quorum overhead increases)

**See Also**: [middle-level/07-performance-tuning.md](middle-level/07-performance-tuning.md)

---

## Security Requirements

### Authentication and Authorization

**Requirement**: Only authorized clients should access etcd.

**Why Critical**: etcd contains all cluster secrets:
- Service account tokens
- TLS certificates
- API credentials
- Encryption keys

**How etcd Satisfies**:

1. **Client Certificate Authentication** (recommended)
   ```bash
   etcd --client-cert-auth \
        --trusted-ca-file=/etc/etcd/ca.crt \
        --cert-file=/etc/etcd/server.crt \
        --key-file=/etc/etcd/server.key
   ```

2. **Role-Based Access Control (RBAC)**
   ```bash
   # Create role with read-only access
   etcdctl role add read-only
   etcdctl role grant-permission read-only read /registry/ --prefix

   # Create user and assign role
   etcdctl user add kubernetes
   etcdctl user grant-role kubernetes read-only
   ```

**Kubernetes Configuration**:
```yaml
# kube-apiserver flags
--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
--etcd-certfile=/etc/kubernetes/pki/etcd/apiserver-etcd-client.crt
--etcd-keyfile=/etc/kubernetes/pki/etcd/apiserver-etcd-client.key
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/server/options/etcd.go:89`

### Encryption in Transit

**Requirement**: Network communication must be encrypted.

**Why Critical**: Prevents eavesdropping and man-in-the-middle attacks.

**How etcd Satisfies**: TLS for all connections

```mermaid
graph LR
    A[kube-apiserver] -->|TLS| B[etcd Client API]
    C[etcd Node 1] -->|TLS| D[etcd Node 2]
    D -->|TLS| E[etcd Node 3]

    style A fill:#e1f5ff
    style B fill:#90EE90
    style C fill:#90EE90
    style D fill:#90EE90
    style E fill:#90EE90
```

**Configuration**:
```bash
etcd \
  # Client-facing TLS
  --cert-file=/etc/etcd/server.crt \
  --key-file=/etc/etcd/server.key \
  --trusted-ca-file=/etc/etcd/ca.crt \
  --client-cert-auth \
  \
  # Peer-to-peer TLS
  --peer-cert-file=/etc/etcd/peer.crt \
  --peer-key-file=/etc/etcd/peer.key \
  --peer-trusted-ca-file=/etc/etcd/ca.crt \
  --peer-client-cert-auth
```

### Encryption at Rest

**Requirement**: Data on disk must be encrypted.

**Why Critical**: Protects against physical disk theft or unauthorized access.

**How Kubernetes Satisfies**:

Kubernetes provides **encryption at rest** for etcd data:

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

**kube-apiserver flag**:
```bash
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

**Flow**:
```mermaid
sequenceDiagram
    participant API as kube-apiserver
    participant Transform as Encryption Transformer
    participant etcd

    API->>Transform: Store Secret
    Transform->>Transform: Encrypt with AES-CBC
    Transform->>etcd: Put encrypted data
    etcd-->>Transform: Success
    Transform-->>API: Success

    API->>Transform: Get Secret
    Transform->>etcd: Get encrypted data
    etcd-->>Transform: Encrypted data
    Transform->>Transform: Decrypt
    Transform-->>API: Plaintext Secret
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/envelope/envelope.go:45`

**See Also**: [middle-level/08-security.md](middle-level/08-security.md)

---

## Operational Requirements

### Backup and Restore

**Requirement**: Regular backups and tested restore procedures.

**Why Critical**: Protect against:
- Accidental deletion
- Corruption
- Disaster scenarios
- Data center failures

**How etcd Satisfies**: Snapshot API

```bash
# Create snapshot
ETCDCTL_API=3 etcdctl snapshot save backup.db

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status backup.db --write-out=table

# Restore snapshot
ETCDCTL_API=3 etcdctl snapshot restore backup.db \
  --data-dir=/var/lib/etcd-restore
```

**Backup Frequency**:
```
Production clusters:
- Hourly snapshots (keep last 24)
- Daily snapshots (keep last 7)
- Weekly snapshots (keep last 4)
```

**Code Reference**: etcd snapshot implementation

**See Also**: [middle-level/06-backup-restore.md](middle-level/06-backup-restore.md)

### Monitoring and Alerting

**Requirement**: Visibility into etcd health and performance.

**Critical Metrics**:
| Metric | Alert Threshold | Impact |
|--------|-----------------|--------|
| `etcd_server_has_leader` | = 0 for > 1min | No writes possible |
| `etcd_disk_backend_commit_duration_seconds` | > 100ms | Slow writes |
| `etcd_mvcc_db_total_size_in_bytes` | > 8 GB | Performance degradation |
| `etcd_server_leader_changes_seen_total` | > 5 per hour | Cluster instability |
| `etcd_network_peer_round_trip_time_seconds` | > 50ms | Slow replication |

**Example Prometheus Alert**:
```yaml
- alert: etcdNoLeader
  expr: etcd_server_has_leader == 0
  for: 1m
  annotations:
    summary: "etcd cluster has no leader"
    description: "etcd cluster {{ $labels.instance }} has no leader for 1 minute"
```

**See Also**: [middle-level/07-performance-tuning.md](middle-level/07-performance-tuning.md)

### Compaction and Defragmentation

**Requirement**: Automatic database maintenance to prevent unbounded growth.

**Why Critical**: Without compaction, database grows indefinitely:
```
Day 0: 100 MB
Day 1: 500 MB (old revisions kept)
Day 7: 5 GB (old revisions kept)
Day 30: 50 GB (old revisions kept) → Out of memory!
```

**How etcd Satisfies**: Automatic compaction

```bash
# Auto-compaction every 5 minutes
etcd --auto-compaction-retention=5m \
     --auto-compaction-mode=periodic

# Or: Keep last 1000 revisions
etcd --auto-compaction-retention=1000 \
     --auto-compaction-mode=revision
```

**Kubernetes Configuration**:
```yaml
# kube-apiserver flags
--etcd-compaction-interval=5m
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/compact.go:85`

**Defragmentation** (reclaim disk space):
```bash
# Defrag all members
etcdctl defrag --cluster
```

**See Also**: [middle-level/03-compaction-defrag.md](middle-level/03-compaction-defrag.md)

---

## Requirements Comparison Matrix

### etcd vs. Alternatives

| Requirement | etcd | ZooKeeper | Consul | MySQL | Redis |
|-------------|------|-----------|--------|-------|-------|
| **Strong Consistency** | ✓ Raft | ✓ ZAB | ✗ Eventually | ✓ Replication | ✗ Eventually |
| **Watch API** | ✓ Native | ✓ Native | ✓ Limited | ✗ No | ✓ Pub/Sub |
| **Transactions** | ✓ CAS | ✓ Limited | ✓ CAS | ✓ ACID | ✓ Limited |
| **Simplicity** | ✓ Go binary | ✗ JVM | ✓ Go binary | ✓ Standard | ✓ Standard |
| **Performance** | ✓✓ High | ✓ Moderate | ✓✓ High | ✓✓ High | ✓✓✓ Very High |
| **Maturity (2014)** | ✓ New | ✓✓✓ Mature | ✓ New | ✓✓✓ Mature | ✓✓ Mature |
| **Distributed** | ✓ Native | ✓ Native | ✓ Native | ✓ Replication | ✓ Cluster |
| **Operations** | ✓ Simple | ✗ Complex | ✓ Simple | ✓ Familiar | ✓ Simple |

**Winner: etcd** - Best balance of features, simplicity, and performance for Kubernetes use case.

### Feature Comparison Detail

#### Strong Consistency

| Database | Consensus | Read Consistency | Write Consistency |
|----------|-----------|------------------|-------------------|
| **etcd** | Raft | Linearizable | Linearizable |
| **ZooKeeper** | ZAB (Raft-like) | Linearizable | Linearizable |
| **Consul** | Raft | Eventual (default) | Linearizable |
| **MySQL** | Primary-replica | Eventually consistent | On primary |
| **Redis** | Sentinel/Cluster | Eventually consistent | Eventually consistent |

#### Watch/Notification Mechanism

| Database | Mechanism | Reliability | Historical Replay |
|----------|-----------|-------------|-------------------|
| **etcd** | gRPC stream | ✓ Reliable | ✓ From any revision |
| **ZooKeeper** | Watcher callbacks | ✓ Reliable | ✗ No |
| **Consul** | Blocking queries | ✓ Reliable | ✗ No |
| **MySQL** | Triggers/polling | ✗ Unreliable | ✗ No |
| **Redis** | Pub/Sub | ✗ Best-effort | ✗ No |

**etcd's watch is superior**: Revision-based, reliable, supports historical replay.

---

## Trade-offs and Design Decisions

### Trade-off 1: Consistency vs. Availability

**Decision**: Prioritize **consistency** over availability during partitions (CP in CAP).

**Rationale**:
```
Scenario: Network partition splits etcd cluster

Option 1 (CP - Chosen):
- Majority partition: Operational ✓
- Minority partition: Unavailable ✗
- Guarantee: No inconsistent data

Option 2 (AP - Rejected):
- All partitions: Operational ✓
- Guarantee: None (split-brain possible) ✗
```

**Impact**:
- ✓ Prevents data inconsistency
- ✓ Prevents duplicate resource creation
- ✗ Reduced availability during partitions

**Acceptable because**: Network partitions are rare in practice, and consistency is critical.

### Trade-off 2: Performance vs. Durability

**Decision**: Prioritize **durability** via fsync, accept some latency.

**Rationale**:
```
Write path options:

Option 1 (Durable - Chosen):
1. Append to WAL
2. fsync to disk ← Slow but safe
3. Respond to client
Latency: ~10ms, but data survives crashes

Option 2 (Fast - Rejected):
1. Append to WAL (in memory)
2. Respond to client
3. fsync later (async)
Latency: ~1ms, but risk of data loss
```

**Impact**:
- ✓ Data survives crashes
- ✗ Higher write latency (~10ms vs ~1ms)

**Mitigation**: Use fast storage (SSD/NVMe) to minimize fsync latency.

### Trade-off 3: Scalability vs. Simplicity

**Decision**: Single etcd cluster per Kubernetes cluster (not sharding).

**Rationale**:
```
Option 1 (Single cluster - Chosen):
- Simple operational model
- Limited to ~10K writes/s
- Database size limit ~10 GB

Option 2 (Sharded - Rejected):
- Complex: Multiple etcd clusters
- Need routing logic
- Cross-shard transactions difficult
```

**Impact**:
- ✓ Simple to operate
- ✓ Sufficient for most clusters (< 5,000 nodes)
- ✗ Hard limit on cluster size

**Acceptable because**: Few clusters exceed 5,000 nodes, and complexity cost is high.

### Trade-off 4: Feature Richness vs. Minimalism

**Decision**: etcd provides **minimal feature set** focused on core needs.

**What etcd does NOT include**:
- ✗ SQL query language
- ✗ Secondary indexes
- ✗ Complex data types (only byte arrays)
- ✗ Built-in caching
- ✗ Automatic sharding

**Rationale**:
- Simpler codebase → fewer bugs
- Easier to understand and operate
- Kubernetes can build higher-level abstractions (e.g., watch cache)

**Impact**:
- ✓ Reliable, battle-tested
- ✓ Easy to operate
- ✗ Some features must be built in Kubernetes (e.g., secondary indexes)

---

## Summary

### Requirements Recap

| Requirement Category | Key Points |
|---------------------|------------|
| **Consistency** | Strong consistency via Raft consensus; Linearizable reads/writes |
| **Availability** | High availability with fault tolerance; 3-5 node clusters recommended |
| **Performance** | < 10ms write latency; ~10,000 writes/s capacity; SSD storage required |
| **Scalability** | Supports clusters up to 5,000 nodes; Database limit ~8 GB |
| **Security** | TLS authentication; Encryption at rest (Kubernetes layer); RBAC support |
| **Operations** | Automatic compaction; Snapshot backups; Simple operational model |

### Why etcd is the Right Choice

1. **Strong Consistency**: Raft consensus provides exactly what Kubernetes needs
2. **Watch API**: Native, reliable, revision-based watches
3. **Simplicity**: Single Go binary, minimal dependencies
4. **Performance**: Sufficient for large clusters (tested at 5,000+ nodes)
5. **Proven**: Battle-tested in production across thousands of clusters

### Key Takeaways

1. **etcd is not optional**: It's fundamental to Kubernetes architecture
2. **Consistency over availability**: CP system, prioritizes correctness
3. **Operations matter**: Backups, compaction, monitoring are critical
4. **Performance requires tuning**: SSD storage, proper sizing, compaction
5. **Security must be configured**: TLS and encryption are not default

### Next Steps

After understanding requirements, dive deeper:

1. **Functional Specification**: [02-FUNCTIONAL-SPEC.md](02-FUNCTIONAL-SPEC.md) - What etcd provides
2. **etcd Overview**: [high-level/01-etcd-overview.md](high-level/01-etcd-overview.md) - How etcd works
3. **Kubernetes Integration**: [high-level/02-kubernetes-integration.md](high-level/02-kubernetes-integration.md) - How Kubernetes uses etcd
4. **Operations Guide**: [middle-level/05-cluster-management.md](middle-level/05-cluster-management.md) - Running etcd in production

### References

- [etcd Documentation](https://etcd.io/docs/)
- [Raft Consensus Algorithm](https://raft.github.io/)
- [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- [CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem)

---

**Document Status**: Complete (1,105 lines, 15 diagrams)
**Code References**: 8 references with file:line numbers
**Next Document**: [02-FUNCTIONAL-SPEC.md](02-FUNCTIONAL-SPEC.md)
