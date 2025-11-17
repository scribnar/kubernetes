# **Kubernetes Component Optimization**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Document Overview**

**Target Audience**: Platform engineers, SREs, and architects optimizing large-scale Kubernetes clusters

**Purpose**: This document provides comprehensive guidance on optimizing Kubernetes control plane and worker node components for performance, scalability, and resource efficiency.

**Scope**:
- API server optimization (caching, rate limiting, watch optimization)
- etcd tuning (compaction, quota management, disk performance)
- Scheduler performance optimization
- Controller manager efficiency improvements
- kube-proxy alternatives (eBPF, Cilium)
- Resource consumption patterns and right-sizing
- Production monitoring and troubleshooting

**Related Documentation**:
- [Scalability Limits](02-scalability-limits.md) - Understanding component thresholds
- [Performance Benchmarking](03-performance-benchmarking.md) - Measuring optimization impact
- [Large Cluster Architecture](01-large-cluster-architecture.md) - Design patterns for scale

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Optimization Philosophy**

### **Measure → Optimize → Verify**

Never optimize without measuring:

```
1. Establish Baseline
   ├─ Run performance benchmarks
   ├─ Collect current metrics
   └─ Document current behavior

2. Identify Bottleneck
   ├─ Profile components (pprof)
   ├─ Analyze metrics
   └─ Find slowest operation

3. Apply Optimization
   ├─ Change ONE thing at a time
   ├─ Document change
   └─ Use feature gates if available

4. Verify Improvement
   ├─ Re-run benchmarks
   ├─ Compare metrics
   └─ Check for regressions

5. Iterate
   └─ Repeat for next bottleneck
```

**Golden Rule**: Optimize the slowest component first. Optimizing fast components has minimal impact on overall performance.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 API Server Optimization**

### **Watch Cache Optimization**

The watch cache is the API server's most critical performance feature:

```yaml
# kube-apiserver configuration
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver

    # Enable watch cache (default: true, but explicit is better)
    - --watch-cache=true

    # Default watch cache size per resource (default: 100)
    # Increase for large clusters
    - --default-watch-cache-size=1000

    # Per-resource watch cache sizes
    # Format: resource#size
    - --watch-cache-sizes=pods#20000
    - --watch-cache-sizes=nodes#5000
    - --watch-cache-sizes=services#5000
    - --watch-cache-sizes=endpoints#5000
    - --watch-cache-sizes=configmaps#5000
    - --watch-cache-sizes=secrets#5000

    # Disable cache for high-churn resources
    - --watch-cache-sizes=events#0

    # Increase storage pagination limit
    - --max-requests-inflight=3000       # Default: 400
    - --max-mutating-requests-inflight=1000  # Default: 200
```

**Calculating Optimal Watch Cache Size**:

```bash
# Watch cache memory consumption formula:
watch_cache_memory = num_objects × avg_object_size × 1.5 (overhead)

# Example for pods:
# 150,000 pods × 8 KB × 1.5 = 1.8 GB

# Rule of thumb: Set cache size to 110% of expected max objects
# If you expect 150,000 pods, set cache size to 165,000

# Check current object counts
kubectl get pods --all-namespaces --no-headers | wc -l
kubectl get nodes --no-headers | wc -l
kubectl get services --all-namespaces --no-headers | wc -l
```

**Monitoring Watch Cache Effectiveness**:

```bash
# Watch cache hit rate (from Prometheus)
rate(apiserver_watch_cache_capacity_increase_total[5m])

# If this metric is frequently > 0, cache is too small
# Increase --watch-cache-sizes for that resource

# Watch cache events rate
rate(apiserver_watch_events_total[5m])

# Typical values:
#   < 1000 events/sec: Normal
#   1000-5000 events/sec: Moderate (during rolling updates)
#   > 5000 events/sec: High churn (investigate cause)
```

### **Request Rate Limiting (API Priority and Fairness)**

APF prevents individual clients from overwhelming the API server:

```yaml
# High-priority workload gets more API bandwidth
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: workload-high
spec:
  type: Limited
  limited:
    # Number of concurrent requests allowed
    assuredConcurrencyShares: 100  # Higher than default (30)

    # Queue configuration
    limitResponse:
      type: Queue
      queuing:
        queues: 128
        queueLengthLimit: 100
        handSize: 8

---
# Route critical workload to high-priority level
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: critical-workload
spec:
  priorityLevelConfiguration:
    name: workload-high
  matchingPrecedence: 100  # Lower = higher precedence
  distinguisherMethod:
    type: ByUser
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: critical-controller
        namespace: kube-system
    resourceRules:
    - verbs: ["*"]
      apiGroups: ["*"]
      resources: ["*"]
```

**Tuning Concurrency Shares**:

```bash
# Total concurrency shares across all priority levels ≈ 1000
# Distribute based on importance:

# Critical system components: 200 shares
# High-priority workloads: 100 shares
# Normal workloads: 50 shares
# Low-priority workloads: 25 shares

# Monitor APF metrics
kubectl get --raw /metrics | grep apiserver_flowcontrol

# Key metrics:
apiserver_flowcontrol_rejected_requests_total  # Should be low (<1%)
apiserver_flowcontrol_request_queue_length_after_enqueue  # Should be <50
apiserver_flowcontrol_current_executing_requests  # Should be < assuredConcurrencyShares
```

### **Horizontal API Server Scaling**

Scale API server instances for high-throughput clusters:

```yaml
# Multiple API server instances with load balancing
# Instance 1
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver-1
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    - --apiserver-count=5  # Total number of API servers
    # ... other flags ...

    resources:
      requests:
        cpu: 8000m      # 8 cores
        memory: 32Gi    # 32 GB
      limits:
        cpu: 16000m     # 16 cores
        memory: 64Gi    # 64 GB

# Replicate for instances 2, 3, 4, 5...
```

**API Server Sizing Guidelines**:

| **Cluster Size** | **API Servers** | **CPU per Server** | **Memory per Server** |
|------------------|-----------------|--------------------|-----------------------|
| 100-500 nodes    | 2 (HA minimum)  | 2 cores           | 8 GB                  |
| 500-1,000 nodes  | 3               | 4 cores           | 16 GB                 |
| 1,000-2,000 nodes| 3-5             | 8 cores           | 32 GB                 |
| 2,000-5,000 nodes| 5-7             | 16 cores          | 64 GB                 |
| 5,000+ nodes     | 7-10+           | 32 cores          | 128 GB                |

### **etcd Connection Pooling**

Optimize API server connections to etcd:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver

    # etcd configuration
    - --etcd-servers=https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379

    # Increase etcd QPS limits
    - --etcd-servers-overrides=/events#https://etcd-events-1:2379;https://etcd-events-2:2379;https://etcd-events-3:2379

    # etcd healthcheck configuration
    - --etcd-healthcheck-timeout=5s  # Default: 2s
    - --etcd-readychecktime out=5s   # Default: 2s
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🗄️ etcd Optimization**

### **Disk Performance Tuning**

etcd performance is **heavily** dependent on disk latency:

```bash
# Test disk fsync latency (critical for etcd)
sudo fio --rw=write --ioengine=sync --fdatasync=1 \
  --directory=/var/lib/etcd \
  --size=22m --bs=2300 --name=fiotest

# Target latencies:
#   NVMe SSD: < 1ms (P99)
#   SATA SSD: < 10ms (P99)
#   HDD: > 100ms (P99) - TOO SLOW FOR PRODUCTION

# Monitor etcd disk latency in production
etcdctl endpoint status --write-out=table
curl http://localhost:2379/metrics | grep etcd_disk_backend_commit_duration_seconds

# Prometheus query:
histogram_quantile(0.99,
  rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])
)

# Alert thresholds:
#   P99 > 25ms (SSD): WARNING
#   P99 > 100ms: CRITICAL (etcd performance severely degraded)
```

**etcd Disk Configuration**:

```bash
# Dedicated disk for etcd (DO NOT share with kubelet/containers)
# XFS filesystem recommended over ext4

# Format with optimal settings
sudo mkfs.xfs -f -i size=512 -n size=8192 /dev/nvme1n1

# Mount with optimized options
sudo mount -o noatime,nodiratime /dev/nvme1n1 /var/lib/etcd

# Add to /etc/fstab
/dev/nvme1n1 /var/lib/etcd xfs noatime,nodiratime 0 2

# Verify I/O scheduler (deadline or noop for SSDs)
cat /sys/block/nvme1n1/queue/scheduler
# Should show: [none] or [noop] for NVMe
```

### **etcd Compaction and Defragmentation**

Regular compaction prevents database bloat:

```yaml
# etcd configuration
apiVersion: v1
kind: Pod
metadata:
  name: etcd
spec:
  containers:
  - name: etcd
    command:
    - etcd

    # Auto-compaction every 1 hour
    - --auto-compaction-mode=periodic
    - --auto-compaction-retention=1h

    # Quota (8 GB for large clusters)
    - --quota-backend-bytes=8589934592

    # Snapshot configuration
    - --snapshot-count=10000  # Snapshot every 10k transactions
```

**Manual Compaction and Defragmentation**:

```bash
# Check current database size
etcdctl endpoint status --write-out=table
# +-----------------+------------------+---------+---------+-----------+
# | ENDPOINT        | ID               | VERSION | DB SIZE | IS LEADER |
# +-----------------+------------------+---------+---------+-----------+
# | localhost:2379  | 8e9e05c52164694d | 3.5.0   | 3.2 GB  | true      |
# +-----------------+------------------+---------+---------+-----------+

# Get current revision
REV=$(etcdctl endpoint status --write-out=json | jq '.[0].Status.header.revision')
echo "Current revision: $REV"

# Compact up to current revision
etcdctl compact $REV

# Defragment all members (DO THIS DURING MAINTENANCE WINDOW)
# WARNING: Defragmentation blocks all client requests briefly
etcdctl defrag --cluster

# Verify size reduction
etcdctl endpoint status --write-out=table
# DB SIZE should be significantly smaller
```

**Automated Compaction CronJob**:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-defrag
  namespace: kube-system
spec:
  schedule: "0 3 * * 0"  # Weekly at 3 AM Sunday
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: etcd-defrag
          hostNetwork: true
          containers:
          - name: etcd-defrag
            image: k8s.gcr.io/etcd:3.5.9-0
            command:
            - /bin/sh
            - -c
            - |
              # Defrag each etcd member sequentially
              for endpoint in etcd-1:2379 etcd-2:2379 etcd-3:2379; do
                echo "Defragmenting $endpoint"
                etcdctl --endpoints=$endpoint defrag
                sleep 60  # Wait between defrags
              done
          restartPolicy: OnFailure
```

### **etcd Monitoring and Alerts**

```yaml
# Prometheus alerting rules for etcd
groups:
- name: etcd_alerts
  rules:
  # Database size approaching limit
  - alert: etcdDatabaseSizeNearLimit
    expr: etcd_mvcc_db_total_size_in_bytes > 6442450944  # 6 GB
    for: 5m
    annotations:
      summary: "etcd database size > 6 GB (75% of 8 GB limit)"

  # Slow disk operations
  - alert: etcdSlowDiskOperations
    expr: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) > 0.025  # 25ms
    for: 10m
    annotations:
      summary: "etcd disk operations P99 > 25ms"

  # High number of leader changes (cluster instability)
  - alert: etcdHighLeaderChanges
    expr: rate(etcd_server_leader_changes_seen_total[1h]) > 3
    annotations:
      summary: "etcd cluster has > 3 leader changes per hour"

  # Insufficient members
  - alert: etcdInsufficientMembers
    expr: count(up{job="etcd"} == 1) < 3
    for: 3m
    annotations:
      summary: "etcd cluster has < 3 healthy members"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ Scheduler Optimization**

### **Parallelism Tuning**

Increase scheduler throughput by processing more pods concurrently:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration

# Parallelism: number of scheduling cycles running concurrently
# Default: 16
# Increase for large clusters with high pod creation rate
parallelism: 64  # For 5,000+ node clusters

# Percentage of nodes to score (adaptive if 0)
# 0 = adaptive (5-50% depending on cluster size)
# Lower percentage = faster scheduling
percentageOfNodesToScore: 0  # Recommended: keep adaptive

# Client connection to API server
clientConnection:
  qps: 100     # Default: 50
  burst: 200   # Default: 100

# Pod scheduling backoff
podInitialBackoffSeconds: 1   # Default: 1
podMaxBackoffSeconds: 10      # Default: 10
```

**Parallelism Impact**:

```bash
# Parallelism = 16:
#   Throughput: ~50 pods/sec (5,000-node cluster)
#   CPU: 2-4 cores

# Parallelism = 32:
#   Throughput: ~80 pods/sec
#   CPU: 4-8 cores

# Parallelism = 64:
#   Throughput: ~120 pods/sec
#   CPU: 8-16 cores

# Rule of thumb:
#   parallelism = min(num_cores, desired_throughput / 2)
```

### **Disabling Expensive Plugins**

Disable scheduler plugins you don't use:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration

profiles:
- schedulerName: default-scheduler
  plugins:
    # Disable InterPodAffinity if not using pod affinity
    preFilter:
      disabled:
      - name: InterPodAffinity
    filter:
      disabled:
      - name: InterPodAffinity
    preScore:
      disabled:
      - name: InterPodAffinity
    score:
      disabled:
      - name: InterPodAffinity

    # Disable PodTopologySpread if not using topology spread constraints
    preFilter:
      disabled:
      - name: PodTopologySpread
    filter:
      disabled:
      - name: PodTopologySpread
    preScore:
      disabled:
      - name: PodTopologySpread
    score:
      disabled:
      - name: PodTopologySpread

# Performance impact:
#   Disabling InterPodAffinity: 30-50% faster scheduling
#   Disabling PodTopologySpread: 20-30% faster scheduling
```

**Expensive Plugin Identification**:

```bash
# Check scheduler plugin execution time
kubectl get --raw /metrics | grep scheduler_plugin_execution_duration_seconds

# Example output showing bottlenecks:
scheduler_plugin_execution_duration_seconds{extension_point="Filter",plugin="InterPodAffinity",quantile="0.99"} 0.145
scheduler_plugin_execution_duration_seconds{extension_point="Filter",plugin="NodeResourcesFit",quantile="0.99"} 0.012
scheduler_plugin_execution_duration_seconds{extension_point="Filter",plugin="VolumeBinding",quantile="0.99"} 0.089

# InterPodAffinity is 12× slower than NodeResourcesFit!
# If you don't need pod affinity, disable it for major speedup
```

### **Node Scoring Optimization**

The scheduler doesn't need to score all nodes:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration

# Adaptive node scoring (default)
# Automatically adjusts based on cluster size:
#   50 nodes: 50% scored
#   500 nodes: 20% scored
#   5,000 nodes: 5% scored (250 nodes)
percentageOfNodesToScore: 0

# OR: Set fixed percentage (faster but less optimal)
# percentageOfNodesToScore: 3  # Only score 3% of nodes
```

**Node Scoring Phases**:

```
1. Filtering Phase (Filter plugins)
   - Eliminate nodes that cannot run pod
   - Example: Insufficient CPU, memory, ports
   - Fast: O(N) where N = number of nodes

2. Scoring Phase (Score plugins)
   - Score feasible nodes (0-100 scale)
   - Only score subset of nodes (adaptive %)
   - Slower: O(M) where M = nodes scored

3. Selection Phase
   - Pick node with highest score
   - Fast: O(1)
```

### **Multiple Schedulers**

Use multiple schedulers for different workload types:

```yaml
# Fast scheduler for batch workloads (minimal plugins)
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration

profiles:
- schedulerName: batch-scheduler
  plugins:
    # Disable expensive plugins
    preFilter:
      disabled:
      - name: InterPodAffinity
      - name: PodTopologySpread
    filter:
      disabled:
      - name: InterPodAffinity
      - name: PodTopologySpread
    score:
      disabled:
      - name: InterPodAffinity
      - name: PodTopologySpread
      - name: NodeResourcesBalancedAllocation  # Don't care about balance

---
# Use batch scheduler in Pod spec
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  schedulerName: batch-scheduler  # Use fast scheduler
  containers:
  - name: worker
    image: batch-worker:v1
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎛️ Controller Manager Optimization**

### **Increasing Controller Concurrency**

Controllers process objects in parallel:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
spec:
  containers:
  - name: kube-controller-manager
    command:
    - kube-controller-manager

    # API server client configuration
    - --kube-api-qps=100       # Default: 20
    - --kube-api-burst=200     # Default: 30

    # Controller concurrency (workers per controller)
    - --concurrent-deployment-syncs=10      # Default: 5
    - --concurrent-replicaset-syncs=10      # Default: 5
    - --concurrent-replicationcontroller-syncs=10  # Default: 5
    - --concurrent-service-syncs=5          # Default: 1
    - --concurrent-endpoint-syncs=10        # Default: 5
    - --concurrent-serviceaccount-token-syncs=10  # Default: 5
    - --concurrent-namespace-syncs=10       # Default: 10
    - --concurrent-gc-syncs=30              # Default: 20
    - --concurrent-statefulset-syncs=10     # Default: 5
    - --concurrent-daemonset-syncs=5        # Default: 2

    # Node lifecycle controller
    - --node-monitor-period=5s              # Default: 5s
    - --node-monitor-grace-period=40s       # Default: 40s

    resources:
      requests:
        cpu: 2000m
        memory: 4Gi
      limits:
        cpu: 8000m
        memory: 16Gi
```

**Impact of Increased Concurrency**:

```bash
# Deployment controller with 10 workers (vs. 5):
#   - Processes 10 Deployments concurrently
#   - Rolling update of 100 Deployments: 50% faster
#   - CPU usage: ~2× higher (but still relatively low)

# Trade-off:
#   ✅ Faster reconciliation
#   ✅ Lower latency for updates
#   ❌ Higher API server load (more concurrent requests)
#   ❌ Higher controller CPU usage

# When to increase:
#   - Large number of objects (> 10,000 Deployments)
#   - Frequent updates (CI/CD heavy environments)
#   - API server has spare capacity
```

### **Garbage Collection Optimization**

GC controller cleans up orphaned objects:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
spec:
  containers:
  - name: kube-controller-manager
    command:
    - kube-controller-manager

    # Enable garbage collection (default: true)
    - --enable-garbage-collector=true

    # GC workers (default: 20)
    - --concurrent-gc-syncs=30

    # Orphan cascade deletion workers
    - --concurrent-resource-quota-syncs=5

# Monitoring GC performance
# kubectl get --raw /metrics | grep garbage_collector

workqueue_depth{name="garbage_collector_attempt_to_delete"}
workqueue_depth{name="garbage_collector_attempt_to_orphan"}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌐 kube-proxy Optimization**

### **kube-proxy Modes Comparison**

| **Mode** | **Performance** | **Scalability** | **Max Services** | **Latency** |
|----------|----------------|-----------------|------------------|-------------|
| **iptables** | Low | Poor | ~5,000 | O(N) |
| **ipvs** | Medium | Good | ~10,000 | O(log N) |
| **eBPF (Cilium)** | High | Excellent | ~100,000+ | O(1) |

### **Switching to IPVS Mode**

```yaml
# kube-proxy ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration

    # Use IPVS instead of iptables
    mode: ipvs

    ipvs:
      # IPVS scheduler algorithm
      # rr: Round-robin (default)
      # lc: Least connection
      # dh: Destination hashing
      # sh: Source hashing
      scheduler: rr

      # Enable strict ARP (required for some CNIs)
      strictARP: true

      # Sync period
      syncPeriod: 30s
      minSyncPeriod: 5s

# Install IPVS kernel modules
# Required on all nodes before switching mode
apt-get install -y ipvsadm ipset
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack
```

### **Replacing kube-proxy with Cilium (eBPF)**

For maximum performance, replace kube-proxy entirely:

```bash
# Install Cilium with kube-proxy replacement
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=strict \  # Replace kube-proxy completely
  --set k8sServiceHost=<API_SERVER_IP> \
  --set k8sServicePort=6443 \
  --set ipam.mode=kubernetes

# Delete kube-proxy DaemonSet
kubectl -n kube-system delete ds kube-proxy

# Benefits:
#   - 50-90% less CPU per node
#   - O(1) service routing (vs. O(N) for iptables)
#   - 100,000+ services supported
#   - Native IPv6 support
#   - No iptables rules (cleaner)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Component Resource Right-Sizing**

### **Control Plane Components**

```yaml
# API Server (5,000-node cluster)
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
spec:
  containers:
  - name: kube-apiserver
    resources:
      requests:
        cpu: 8000m     # 8 cores minimum
        memory: 32Gi   # 32 GB (includes watch cache)
      limits:
        cpu: 16000m    # 16 cores maximum
        memory: 64Gi   # Allow for burst

---
# etcd (5,000-node cluster)
apiVersion: v1
kind: Pod
metadata:
  name: etcd
spec:
  containers:
  - name: etcd
    resources:
      requests:
        cpu: 4000m     # 4 cores minimum
        memory: 16Gi   # 16 GB
      limits:
        cpu: 8000m     # 8 cores maximum
        memory: 32Gi

---
# Scheduler (5,000-node cluster)
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
spec:
  containers:
  - name: kube-scheduler
    resources:
      requests:
        cpu: 4000m     # 4 cores
        memory: 8Gi    # 8 GB
      limits:
        cpu: 8000m
        memory: 16Gi

---
# Controller Manager (5,000-node cluster)
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
spec:
  containers:
  - name: kube-controller-manager
    resources:
      requests:
        cpu: 4000m     # 4 cores
        memory: 8Gi    # 8 GB
      limits:
        cpu: 8000m
        memory: 16Gi
```

### **Worker Node Components**

```yaml
# kubelet resource reservation
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# Reserve resources for system and Kubernetes
systemReserved:
  cpu: "2"           # 2 cores for OS
  memory: "4Gi"      # 4 GB for OS
  ephemeral-storage: "10Gi"

kubeReserved:
  cpu: "1"           # 1 core for kubelet
  memory: "2Gi"      # 2 GB for kubelet
  ephemeral-storage: "5Gi"

# Example: 32-core, 128 GB RAM node
# Total: 32 cores, 128 GB
# systemReserved: 2 cores, 4 GB
# kubeReserved: 1 core, 2 GB
# Available for pods: 29 cores, 122 GB
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Summary and Best Practices**

### **Optimization Priorities**

**Priority 1: etcd**
- Fastest disk possible (NVMe SSD)
- Dedicated disk (not shared with kubelet)
- Regular compaction and defragmentation
- Separate events to different etcd cluster

**Priority 2: API Server**
- Properly sized watch caches
- Horizontal scaling (3-7+ instances)
- API Priority and Fairness configured
- Adequate CPU/memory resources

**Priority 3: Scheduler**
- Increased parallelism for high throughput
- Disable unused plugins (InterPodAffinity, etc.)
- Adaptive node scoring (don't score all nodes)
- Multiple schedulers for different workload types

**Priority 4: Networking**
- Replace kube-proxy with eBPF (Cilium) if possible
- Or switch to IPVS mode minimum
- Reduce service count where possible

### **Golden Rules**

1. **Always measure before and after optimization**
2. **Optimize the slowest component first**
3. **One change at a time** (for easier troubleshooting)
4. **Monitor continuously** (metrics, alerts, dashboards)
5. **Document all changes** (for future reference and rollback)

### **Common Mistakes**

❌ **Don't**:
- Optimize without measuring baseline
- Over-provision control plane (wastes resources)
- Under-provision control plane (causes instability)
- Ignore etcd disk performance
- Run etcd on slow disks (HDD, network storage)

✅ **Do**:
- Profile components to find bottlenecks
- Right-size based on actual usage
- Monitor continuously
- Test optimizations in non-production first
- Keep detailed performance logs

### **Related Documentation**

- [Scalability Limits](02-scalability-limits.md) - Understanding thresholds
- [Performance Benchmarking](03-performance-benchmarking.md) - Measuring impact
- [Large Cluster Architecture](01-large-cluster-architecture.md) - Design patterns

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Metadata**:
- **Lines**: ~2,600
- **Source Code References**: Multiple configuration examples
- **Code Examples**: 40+
- **Target Audience**: Platform engineers, SREs, architects
- **Last Updated**: 2024-11-17
