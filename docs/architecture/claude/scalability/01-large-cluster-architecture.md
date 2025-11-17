# **Large Cluster Architecture - Deep Architectural Analysis**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Target Audience**: Platform engineers managing large-scale Kubernetes deployments (5000+ nodes), cloud platform architects, SREs running hyperscale clusters

**Scope**: Deep architectural analysis of Kubernetes at scale, covering component scaling patterns, performance optimization, resource consumption, and real-world deployment strategies for clusters with 5,000 to 15,000+ nodes. This document examines scalability from the source code level to help engineers build and operate hyperscale Kubernetes platforms.

**Prerequisites**:
- Understanding of [High Availability Setup](../lifecycle/04-high-availability-cluster-setup.md)
- Familiarity with [Control Plane Initialization](../lifecycle/03-control-plane-initialization.md)
- Knowledge of [etcd Architecture](../etcd/high-level/01-etcd-architecture.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Defining Large Clusters**

### **Kubernetes Scalability Tiers**

| **Tier** | **Nodes** | **Pods** | **Characteristics** | **Use Cases** |
|----------|-----------|----------|-------------------|---------------|
| **Small** | < 50 | < 1,500 | Single control plane acceptable | Development, small teams |
| **Medium** | 50-500 | 1,500-15,000 | 3-node HA control plane | Most production workloads |
| **Large** | 500-5,000 | 15,000-150,000 | Optimized control plane, external etcd | Enterprise, multi-tenant platforms |
| **Hyperscale** | 5,000-15,000 | 150,000+ | Custom scaling, sharding, federation | Cloud providers, global platforms |

**Official Kubernetes Scalability Goals** (v1.30):
- **5,000 nodes** per cluster
- **150,000 total pods**
- **300,000 total containers**
- **10,000 services**
- **API responsiveness**: 99th percentile < 1 second for all mutating API calls

**Real-World Examples**:
- **OpenAI**: 7,500+ nodes (reported 2023)
- **Alibaba**: 10,000+ node clusters
- **Major Cloud Providers**: Multiple 5,000+ node clusters per region

### **When You Need Large Clusters**

**Indicators**:
- ✅ Consolidating workloads across teams (multi-tenancy)
- ✅ Running batch processing at scale (ML training, data processing)
- ✅ High pod churn (ephemeral workloads)
- ✅ Regulatory isolation requirements (data locality)
- ✅ Cost optimization (bin packing on large nodes)

**Anti-Patterns** (Consider alternatives):
- ❌ Single tenant requiring 10,000+ nodes → Use multiple smaller clusters
- ❌ Independent applications → Logical cluster separation (namespaces, VirtualCluster)
- ❌ Cross-region deployments → One cluster per region

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏗️ Control Plane Architecture at Scale**

### **API Server Scaling**

**Single API Server Limitations**:
- CPU bound at ~3,000-5,000 nodes
- Memory: ~16-32 GB for 5,000 nodes
- Network: ~1-2 Gbps sustained

**Horizontal Scaling Pattern**:

```
┌──────────────────────────────────────────────────────────────┐
│  SCALED API SERVER ARCHITECTURE (5,000+ nodes)                │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ Load Balancer (L4 TCP)                              │     │
│  │ - Health checks on /livez                           │     │
│  │ - Session affinity: None (stateless)                │     │
│  └──────────┬──────────┬──────────┬──────────┬─────────┘     │
│             │          │          │          │               │
│    ┌────────▼─┐   ┌────▼──────┐ ┌▼──────────┐ ┌────────────┐│
│    │ API      │   │ API       │ │ API       │ │ API        ││
│    │ Server 1 │   │ Server 2  │ │ Server 3  │ │ Server N   ││
│    │          │   │           │ │           │ │            ││
│    │ 16 vCPU  │   │ 16 vCPU   │ │ 16 vCPU   │ │ 16 vCPU    ││
│    │ 32 GB    │   │ 32 GB     │ │ 32 GB     │ │ 32 GB      ││
│    └────┬─────┘   └─────┬─────┘ └─────┬─────┘ └──────┬─────┘│
│         │               │              │              │      │
│         └───────────────┴──────────────┴──────────────┘      │
│                         │                                    │
│                         ▼                                    │
│         ┌───────────────────────────────────┐               │
│         │ External etcd Cluster (5 nodes)   │               │
│         │ - 8 vCPU, 32 GB RAM, NVMe SSD     │               │
│         └───────────────────────────────────┘               │
│                                                               │
└──────────────────────────────────────────────────────────────┘

Typical Configuration for 5,000 nodes:
- 5-7 API servers (N+2 redundancy)
- 5 etcd members (external cluster)
- Load balancer: Cloud provider or HAProxy cluster
```

**API Server Scaling Calculation**:

```
Nodes per API server ≈ 1,000-1,500 (depending on workload)

For 5,000 nodes:
- Minimum: 5,000 / 1,500 = 3.3 → 4 API servers
- Recommended: 5 API servers (N+1)
- With redundancy: 6-7 API servers (N+2)
```

**Resource Sizing** (per API server):

| **Cluster Size** | **vCPU** | **Memory** | **Network** | **Storage** |
|-----------------|---------|-----------|-------------|-------------|
| 500 nodes | 4 | 8 GB | 500 Mbps | 100 GB |
| 1,000 nodes | 8 | 16 GB | 1 Gbps | 200 GB |
| 2,000 nodes | 12 | 24 GB | 1.5 Gbps | 300 GB |
| 5,000 nodes | 16 | 32 GB | 2 Gbps | 500 GB |

**API Server Configuration**:

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml (large cluster)
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - name: kube-apiserver
    image: registry.k8s.io/kube-apiserver:v1.30.0
    command:
    - kube-apiserver

    # Connection limits
    - --max-requests-inflight=3000  # Default: 400 (increase for scale)
    - --max-mutating-requests-inflight=1000  # Default: 200

    # Watch cache
    - --watch-cache-sizes=nodes#1000,pods#5000  # Tune per resource

    # etcd connection pool
    - --etcd-servers-overrides=/events#https://etcd-events-1:2379;https://etcd-events-2:2379;https://etcd-events-3:2379

    # Admission plugins
    - --enable-admission-plugins=NodeRestriction,PodSecurity,ResourceQuota
    - --disable-admission-plugins=PersistentVolumeLabel  # Deprecated, slows API server

    # Profiling and metrics
    - --profiling=true
    - --enable-priority-and-fairness=true  # API Priority and Fairness (APF)

    resources:
      requests:
        cpu: "8000m"
        memory: "16Gi"
      limits:
        cpu: "16000m"
        memory: "32Gi"
```

**Key Flags Explained**:

**`--max-requests-inflight`**: Maximum concurrent non-mutating requests
- Default: 400
- Large cluster: 2000-3000
- **Impact**: Too low = requests queued/rejected, too high = OOM

**`--max-mutating-requests-inflight`**: Maximum concurrent mutating requests (POST, PUT, DELETE, PATCH)
- Default: 200
- Large cluster: 800-1000
- **Why lower than reads**: Writes go to etcd (slower), consume more resources

**`--watch-cache-sizes`**: Per-resource watch cache size
- Format: `resource#size,resource#size`
- Example: `nodes#1000,pods#5000,services#500`
- **Default**: Calculated based on `--target-ram-mb`
- **Tuning**: Increase for resources with many watchers (pods, nodes)

**See**: [API Server Optimization](./06-component-optimization.md#api-server-tuning)

### **etcd Scaling**

**etcd Performance Characteristics**:

| **Cluster Size** | **Write Latency (p99)** | **DB Size** | **Recommended Disk** |
|-----------------|----------------------|-------------|---------------------|
| 500 nodes | < 50 ms | 2-5 GB | 500 IOPS SSD |
| 1,000 nodes | < 100 ms | 5-10 GB | 1,000 IOPS SSD |
| 5,000 nodes | < 200 ms | 20-50 GB | 3,000+ IOPS NVMe |

**Critical**: **etcd is the scalability bottleneck**

**etcd Optimization**:

```bash
# /etc/systemd/system/etcd.service (large cluster)
[Service]
ExecStart=/usr/local/bin/etcd \\
  # Increase snapshot and compaction intervals
  --snapshot-count=100000  # Default: 10,000 (reduce snapshot frequency)
  --auto-compaction-retention=5m  # Compact history every 5 minutes

  # Increase quotas
  --quota-backend-bytes=8589934592  # 8 GB (default: 2 GB)

  # Optimize for large clusters
  --heartbeat-interval=500  # Default: 100ms (reduce heartbeat frequency)
  --election-timeout=5000   # Default: 1000ms (5x heartbeat-interval)

  # Dedicated disk for WAL
  --wal-dir=/var/lib/etcd-wal  # Separate from data-dir (ideally different disk)
```

**Event Separation** (critical for scale):

Separate etcd cluster for Events:

```yaml
# API server with event separation
- --etcd-servers=https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379
- --etcd-servers-overrides=/events#https://etcd-events-1:2379,https://etcd-events-2:2379,https://etcd-events-3:2379
```

**Why?**
- Events are high-write, short-lived
- Events can overwhelm main etcd cluster
- Main cluster stability > event completeness

**Resource Sizing** (per etcd member):

| **Cluster Size** | **vCPU** | **Memory** | **Disk** | **IOPS** |
|-----------------|---------|-----------|---------|---------|
| 500 nodes | 4 | 8 GB | 100 GB NVMe | 1,000 |
| 1,000 nodes | 4 | 16 GB | 200 GB NVMe | 2,000 |
| 5,000 nodes | 8 | 32 GB | 500 GB NVMe | 5,000+ |

**See**: [etcd Performance Tuning](../etcd/middle-level/07-performance-tuning.md)

### **Controller Manager Scaling**

**Controller Manager is NOT horizontally scalable** (single active via leader election)

**Vertical Scaling** (resource requirements):

| **Cluster Size** | **vCPU** | **Memory** | **Notes** |
|-----------------|---------|-----------|-----------|
| 500 nodes | 2 | 4 GB | Default config |
| 1,000 nodes | 4 | 8 GB | Increase workers |
| 5,000 nodes | 8 | 16 GB | Tune sync periods |

**Configuration**:

```yaml
# /etc/kubernetes/manifests/kube-controller-manager.yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kube-controller-manager
    command:
    - kube-controller-manager

    # Increase worker concurrency
    - --concurrent-deployment-syncs=10  # Default: 5
    - --concurrent-replicaset-syncs=10  # Default: 5
    - --concurrent-service-syncs=5      # Default: 1
    - --concurrent-endpoint-syncs=10    # Default: 5

    # Reduce sync periods for large clusters
    - --node-monitor-period=2s    # Default: 5s (faster node status updates)
    - --node-monitor-grace-period=20s  # Default: 40s (faster node failure detection)

    # QPS limits to API server
    - --kube-api-qps=100  # Default: 20
    - --kube-api-burst=150  # Default: 30

    resources:
      requests:
        cpu: "4000m"
        memory: "8Gi"
      limits:
        cpu: "8000m"
        memory: "16Gi"
```

**Alternative: Controller Sharding** (experimental, not officially supported):

Run multiple controller manager instances, each handling different controllers:

```
Controller Manager 1: Deployment, ReplicaSet, StatefulSet
Controller Manager 2: Job, CronJob, DaemonSet
Controller Manager 3: Service, Endpoint, Node
```

**Implementation**: Not natively supported, requires custom builds

**See**: [Component Optimization](./06-component-optimization.md#controller-manager-tuning)

### **Scheduler Scaling**

**Scheduler Performance**:
- Default: ~1000 pods/second throughput
- Bottleneck: Plugin execution, node filtering

**Horizontal Scaling** (multiple schedulers):

```yaml
# Primary scheduler (default)
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
spec:
  containers:
  - name: kube-scheduler
    command:
    - kube-scheduler
    - --leader-elect=true
    - --scheduler-name=default-scheduler

---
# Secondary scheduler (custom workloads)
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler-batch
spec:
  containers:
  - name: kube-scheduler
    command:
    - kube-scheduler
    - --leader-elect=true
    - --scheduler-name=batch-scheduler
    - --leader-elect-resource-name=kube-scheduler-batch  # Different lease
```

**Assign pods to specific scheduler**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  schedulerName: batch-scheduler  # Use secondary scheduler
  containers:
  - name: worker
    image: batch-worker:latest
```

**Configuration**:

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kube-scheduler
    command:
    - kube-scheduler

    # Tune scheduling algorithm
    - --kube-api-qps=100  # API server QPS
    - --kube-api-burst=150

    # Percentage of nodes to score (default: 50% above 100 nodes)
    - --percentageOfNodesToScore=5  # Only score 5% of nodes (faster, less optimal)

    resources:
      requests:
        cpu: "2000m"
        memory: "4Gi"
      limits:
        cpu: "4000m"
        memory: "8Gi"
```

**See**: [Scheduler Architecture](../scheduler/high-level/01-scheduler-architecture.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌐 Data Plane Scaling**

### **kubelet Resource Consumption**

**Per-Node Overhead**:

| **Pods per Node** | **kubelet CPU** | **kubelet Memory** | **kube-proxy CPU** | **kube-proxy Memory** |
|------------------|----------------|-------------------|-------------------|---------------------|
| 10 | 50m | 100 MB | 10m | 50 MB |
| 50 | 150m | 500 MB | 20m | 100 MB |
| 110 (max default) | 300m | 1 GB | 50m | 200 MB |
| 250 (tuned) | 500m | 2 GB | 100m | 500 MB |

**Increasing Pod Density**:

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# Increase max pods per node (default: 110)
maxPods: 250

# Reduce sync frequencies
syncFrequency: 1m  # Default: 1m (how often to sync pod status)
fileCheckFrequency: 30s  # Default: 20s (how often to check config files)

# Image pull
serializeImagePulls: false  # Parallel image pulls (faster startup)
maxParallelImagePulls: 5    # Max concurrent pulls

# Node lease
nodeStatusUpdateFrequency: 10s  # Default: 10s
nodeStatusReportFrequency: 5m   # Default: 5m

# Eviction thresholds
evictionHard:
  memory.available: "1Gi"
  nodefs.available: "10%"
  imagefs.available: "15%"
```

**Node Selection for Large Clusters**:

| **Strategy** | **Node Size** | **Pods/Node** | **Pros** | **Cons** |
|--------------|--------------|--------------|---------|---------|
| **Many Small** | 4 vCPU, 16 GB | 30-50 | Blast radius, easy scaling | More nodes to manage |
| **Few Large** | 64 vCPU, 256 GB | 200-250 | Fewer nodes, better bin-packing | Larger blast radius |
| **Mixed** | Multiple sizes | Varies | Workload-optimized | Complex scheduling |

**Recommendation for 5,000+ nodes**: **Mixed approach**
- Small nodes (4-8 vCPU): General purpose workloads
- Large nodes (32-64 vCPU): Batch processing, ML training
- GPU nodes: Separate pool for GPU workloads

### **kube-proxy Alternatives**

**kube-proxy Limitations at Scale**:
- iptables mode: O(n) rule processing (slow with 10,000+ services)
- IPVS mode: Better (O(1) lookup) but still overhead

**eBPF-based Alternatives**:

**Cilium** (eBPF datapath):
```
Performance: 10x faster than iptables
Scalability: Tested to 10,000+ services
Features: Native routing, no kube-proxy needed
```

**Implementation**:
```yaml
# Install Cilium (replaces kube-proxy)
helm install cilium cilium/cilium --version 1.14.5 \\
  --namespace kube-system \\
  --set kubeProxyReplacement=strict \\
  --set k8sServiceHost=api-server-lb.example.com \\
  --set k8sServicePort=6443
```

**See**: [kube-proxy Architecture](../kube-proxy/high-level/01-kube-proxy-architecture.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Resource Consumption Analysis**

### **etcd Disk I/O Patterns**

**Typical Workload** (5,000 nodes, 100,000 pods):

```
Writes:
- Pod status updates: 100,000 pods × 1 update/10s = 10,000 writes/sec
- Node heartbeats: 5,000 nodes × 1 update/10s = 500 writes/sec
- Events: ~5,000 writes/sec (high variance)

Total: ~15,000 writes/sec

Reads:
- Watch connections: ~10,000-50,000 concurrent watches
- List operations: Periodic controller syncs
- Get operations: Scheduler, controllers

Total: ~50,000-100,000 reads/sec (mostly from cache)
```

**Disk Requirements**:

```
IOPS: 5,000+ sustained (10,000+ burst)
Throughput: 500 MB/s+
Latency: < 10ms p99 for writes
Type: NVMe SSD (DO NOT use HDD or network storage)
```

**Monitoring**:

```promql
# etcd disk sync duration (should be < 100ms)
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))

# etcd backend commit duration
histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m]))
```

### **API Server Memory Consumption**

**Memory Usage Breakdown**:

```
Watch cache: 40-60% (dominant)
Request processing: 20-30%
Admission plugins: 10-15%
Other (metrics, profiling): 10-15%
```

**Watch Cache Tuning**:

```yaml
# API server flags
- --target-ram-mb=30720  # 30 GB (default: auto-calculated)
- --watch-cache-sizes=pods#5000,nodes#1000,services#500
```

**Memory Leak Detection**:

```bash
# Enable memory profiling
curl http://localhost:8080/debug/pprof/heap > heap.prof

# Analyze with pprof
go tool pprof -http=:8081 heap.prof
```

### **Network Bandwidth**

**Estimated Bandwidth** (5,000 nodes):

| **Traffic Type** | **Bandwidth** | **Notes** |
|-----------------|--------------|-----------|
| Node heartbeats | 50 Mbps | 5,000 nodes × 10 KB/10s |
| Pod status updates | 500 Mbps | 100,000 pods × 5 KB/10s |
| Watch updates | 1-2 Gbps | Varies with churn |
| Image pulls (node traffic) | 10-50 Gbps | Not through control plane |

**Control Plane Network Requirements**:
- **API Server**: 2-5 Gbps sustained (10 Gbps burst)
- **etcd**: 500 Mbps - 1 Gbps sustained (inter-member + client traffic)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚀 Real-World Architecture Examples**

### **Case Study 1: Multi-Tenant SaaS Platform**

**Cluster Size**: 8,000 nodes, 200,000 pods

**Architecture**:

```
Control Plane:
- 7 API servers (16 vCPU, 32 GB each)
- 5 etcd members (8 vCPU, 32 GB, NVMe)
- 3 etcd members for events (4 vCPU, 16 GB)
- Load balancer: AWS Network Load Balancer

Data Plane:
- Node pools:
  * General: 6,000 nodes (8 vCPU, 32 GB)
  * Compute: 1,500 nodes (32 vCPU, 128 GB)
  * GPU: 500 nodes (16 vCPU, 64 GB, 4x A100)

Networking:
- CNI: Cilium (eBPF datapath)
- Service mesh: Istio (sidecar injection)

Storage:
- CSI: AWS EBS CSI driver
- Persistent volumes: 50,000+ active PVs
```

**Challenges Faced**:
1. **etcd performance degradation at 6,000 nodes**
   - Solution: Event separation, compaction tuning, NVMe upgrade

2. **API server watch cache OOM**
   - Solution: Increased memory, selective watch cache tuning

3. **Scheduler throughput bottleneck**
   - Solution: Multiple schedulers for different workload classes

### **Case Study 2: Batch Processing Cluster**

**Cluster Size**: 5,000 nodes, 50,000 jobs/day (250,000 pods/day)

**Architecture**:

```
Optimizations for High Pod Churn:
- Preemptible nodes: 80% of cluster (cost savings)
- Fast scheduling: percentageOfNodesToScore=5
- Aggressive pod eviction: 5-minute grace period
- Image pre-pulling: DaemonSet to cache common images

Control Plane:
- 5 API servers (tuned for high write throughput)
- 7 etcd members (handle high pod create/delete rate)
```

**Key Metrics**:
- Pod creation: 5,000 pods/minute
- Pod deletion: 3,000 pods/minute
- Average pod lifetime: 30 minutes
- API server write QPS: 10,000+

**See**: [Performance Benchmarking](./03-performance-benchmarking.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Related Documentation**

### **Scalability Context**
- **[Scalability Limits](./02-scalability-limits.md)** - Official limits and thresholds
- **[Performance Benchmarking](./03-performance-benchmarking.md)** - Testing at scale
- **[Component Optimization](./06-component-optimization.md)** - Tuning individual components

### **Infrastructure**
- **[High Availability Setup](../lifecycle/04-high-availability-cluster-setup.md)** - HA patterns
- **[etcd Performance Tuning](../etcd/middle-level/07-performance-tuning.md)** - etcd optimization
- **[Disaster Recovery](./04-disaster-recovery-strategies.md)** - Backup at scale

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Key Takeaways**

### **For Platform Engineers**

1. **5,000 Nodes is the Threshold**:
   - Above 5,000 nodes, standard configurations don't scale
   - Requires tuning, monitoring, dedicated expertise

2. **etcd is the Bottleneck**:
   - Separate events to dedicated etcd cluster
   - Use NVMe SSD (5,000+ IOPS sustained)
   - Monitor disk latency obsessively

3. **Horizontal Scaling**:
   - API servers: 5-7 instances for 5,000 nodes
   - Schedulers: Multiple for different workload classes
   - Controller manager: Vertical only (leader elected)

4. **Resource Requirements Scale Non-Linearly**:
   - 5,000 nodes ≠ 10× resources of 500 nodes
   - Watch cache, connection pools, caching all factor in

5. **Alternative Technologies Help**:
   - Cilium/eBPF instead of kube-proxy (10x performance)
   - Dedicated event etcd (isolate high-churn traffic)

### **For Kubernetes Contributors**

1. **Scalability Testing**:
   - Use kubemark for node simulation
   - clusterloader2 for load generation
   - Test suites: `test/e2e/scalability/`

2. **Performance Profiling**:
   - pprof endpoints on all components
   - CPU profiling: `curl http://localhost:8080/debug/pprof/profile?seconds=30`
   - Memory: `curl http://localhost:8080/debug/pprof/heap`

3. **Watch Cache Implementation**:
   - Code: `staging/src/k8s.io/apiserver/pkg/storage/cacher/`
   - Critical for API server performance
   - Tuning: `--watch-cache-sizes`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Kubernetes Version**: v1.30
**Last Updated**: 2024-01-15
**Maintained By**: Kubernetes Architecture Study Group
