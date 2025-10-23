# Priority and Preemption in Kubernetes

## Overview

The Priority and Preemption system in Kubernetes enables pod scheduling based on relative importance, allowing higher-priority pods to preempt (evict) lower-priority pods when resources are scarce. This is critical for multi-tenant clusters and ensuring critical workloads get resources.

**Key Components:**
- **PriorityClass**: API object defining priority values
- **Scheduler Priority Queue**: Priority-based scheduling queue
- **Preemption Algorithm**: Logic for selecting victims for eviction
- **Pod Priority Admission**: Validates and sets pod priority

## Architecture

### Component Interaction

```mermaid
graph TB
    subgraph "API Server"
        PC[PriorityClass API]
        Pod[Pod API]
    end

    subgraph "Scheduler"
        PQ[Priority Queue]
        PA[Preemption Algorithm]
        SB[Scheduling Binding]
    end

    subgraph "Priority System"
        PP[Priority Plugin]
        VE[Victim Evaluator]
        NF[Node Fitness]
    end

    subgraph "Kubelet"
        PE[Pod Eviction]
        PS[Pod Status]
    end

    PC -->|Define Priorities| Pod
    Pod -->|Enqueue by Priority| PQ
    PQ -->|Pop Highest Priority| PP
    PP -->|Try Schedule| NF
    NF -->|No Resources| PA
    PA -->|Find Victims| VE
    VE -->|Evict Pods| PE
    PE -->|Update Status| PS
    PS -->|Retry Scheduling| PQ
    PA -->|Preempt & Bind| SB

    style PC fill:#326CE5,color:#fff
    style PQ fill:#FF6B6B,color:#fff
    style PA fill:#4ECDC4,color:#fff
    style VE fill:#FFE66D,color:#000
```

### Priority Queue Architecture

```mermaid
graph TB
    subgraph "Priority Queue Structure"
        AQ[Active Queue<br/>Sorted by Priority]
        BQ[Backoff Queue<br/>Temporary Failures]
        UQ[Unschedulable Queue<br/>Permanent Failures]
    end

    subgraph "Queue Operations"
        ADD[Add Pod]
        POP[Pop Pod]
        MOVE[Move Between Queues]
        UPDATE[Update Pod]
    end

    subgraph "Priority Calculation"
        PC[Pod Priority]
        TS[Timestamp]
        AT[Attempts]
        SCORE[Queue Score]
    end

    ADD -->|Insert by Priority| AQ
    AQ -->|Get Highest| POP
    POP -->|Schedule Failed| MOVE
    MOVE -->|Backoff| BQ
    MOVE -->|Unschedulable| UQ
    BQ -->|Retry Time| AQ
    UQ -->|Cluster Change| AQ

    PC -->|Primary Sort| SCORE
    TS -->|Tie Breaker| SCORE
    AT -->|Backoff Factor| SCORE
    SCORE -->|Position in Queue| AQ

    style AQ fill:#326CE5,color:#fff
    style BQ fill:#FF6B6B,color:#fff
    style UQ fill:#FFE66D,color:#000
```

## Preemption Algorithm

### Preemption Flow

```mermaid
stateDiagram-v2
    [*] --> CheckPriority: Pod Unschedulable

    CheckPriority --> NoPreemption: Low Priority
    CheckPriority --> FindVictims: High Priority

    FindVictims --> EvaluateNodes: Scan All Nodes
    EvaluateNodes --> SelectVictims: For Each Node

    SelectVictims --> CheckFeasibility: Find Minimum Set
    CheckFeasibility --> ValidatePreemption: Feasible
    CheckFeasibility --> NextNode: Not Feasible

    NextNode --> SelectVictims: More Nodes
    NextNode --> NoPreemption: No More Nodes

    ValidatePreemption --> CheckPDB: Verify Victims
    CheckPDB --> ExecutePreemption: PDB OK
    CheckPDB --> NextNode: PDB Violated

    ExecutePreemption --> NominateNode: Mark for Preemption
    NominateNode --> EvictVictims: Add NominatedNodeName
    EvictVictims --> WaitForEviction: Delete Pods

    WaitForEviction --> SchedulePod: Victims Evicted
    WaitForEviction --> Timeout: Wait Expired

    Timeout --> RetryScheduling: Re-queue
    SchedulePod --> [*]: Success
    NoPreemption --> [*]: Failed

    note right of FindVictims
        Consider:
        - Pod priority
        - PodDisruptionBudget
        - Pod affinity
        - Data locality
    end note
```

### Victim Selection Algorithm

```mermaid
graph TB
    subgraph "Node Evaluation"
        N1[Select Node]
        N2[Get Running Pods]
        N3[Sort by Priority]
    end

    subgraph "Victim Selection"
        V1[Start with Lowest Priority]
        V2[Add to Victim Set]
        V3[Check Resource Fit]
        V4[More Victims Needed?]
    end

    subgraph "Validation"
        C1[Check PDB]
        C2[Check Anti-Affinity]
        C3[Check QoS Class]
        C4[Calculate Cost]
    end

    subgraph "Decision"
        D1[Compare All Nodes]
        D2[Select Best Node]
        D3[Execute Preemption]
    end

    N1 --> N2
    N2 --> N3
    N3 --> V1
    V1 --> V2
    V2 --> V3
    V3 --> V4
    V4 -->|Yes| V1
    V4 -->|No| C1

    C1 --> C2
    C2 --> C3
    C3 --> C4
    C4 --> D1
    D1 --> D2
    D2 --> D3

    style V2 fill:#FF6B6B,color:#fff
    style C1 fill:#FFE66D,color:#000
    style D3 fill:#4ECDC4,color:#fff
```

## Source Code References

### Priority Queue Implementation

**File:** `pkg/scheduler/internal/queue/scheduling_queue.go`

```go
// Priority queue implementation
type PriorityQueue struct {
    // activeQ holds pods being scheduled
    activeQ *heap.Heap
    // podBackoffQ holds pods in backoff
    podBackoffQ *heap.Heap
    // unschedulablePods holds permanently unschedulable pods
    unschedulablePods *UnschedulablePods

    // nominatedPods tracks pods nominated for nodes
    nominatedPods *nominatedPodMap

    // Metrics
    metrics *queueMetrics

    lock sync.RWMutex
    cond sync.Cond
}

// Add adds a pod to the active queue
func (p *PriorityQueue) Add(pod *v1.Pod) error {
    p.lock.Lock()
    defer p.lock.Unlock()

    pInfo := p.newPodInfo(pod)
    if err := p.activeQ.Add(pInfo); err != nil {
        return err
    }

    // Update metrics
    p.metrics.addToActiveQ()
    p.cond.Broadcast()
    return nil
}

// Pop removes the highest priority pod
func (p *PriorityQueue) Pop() (*framework.QueuedPodInfo, error) {
    p.lock.Lock()
    defer p.lock.Unlock()

    for p.activeQ.Len() == 0 {
        p.cond.Wait()
    }

    obj, err := p.activeQ.Pop()
    if err != nil {
        return nil, err
    }

    pInfo := obj.(*framework.QueuedPodInfo)
    p.metrics.removeFromActiveQ()
    return pInfo, nil
}
```

**Location:** `pkg/scheduler/internal/queue/scheduling_queue.go:100-200`

### Preemption Algorithm

**File:** `pkg/scheduler/framework/plugins/defaultpreemption/default_preemption.go`

```go
// Preempt finds nodes with pods that can be preempted
func (pl *DefaultPreemption) Preempt(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    m framework.NodeToStatusMap,
) (*framework.PostFilterResult, *framework.Status) {

    // Get all nodes for evaluation
    allNodes, err := pl.fh.SnapshotSharedLister().NodeInfos().List()
    if err != nil {
        return nil, framework.AsStatus(err)
    }

    // Find potential nodes for preemption
    potentialNodes, err := pl.findCandidates(
        ctx, state, pod, allNodes, m,
    )
    if err != nil {
        return nil, framework.AsStatus(err)
    }

    if len(potentialNodes) == 0 {
        return nil, framework.NewStatus(
            framework.Unschedulable,
            "no nodes available for preemption",
        )
    }

    // Select best node
    node := pl.selectBestNode(ctx, state, pod, potentialNodes)

    return &framework.PostFilterResult{
        NominatedNodeName: node.Name,
    }, framework.NewStatus(framework.Success)
}

// findCandidates finds all nodes where preemption is possible
func (pl *DefaultPreemption) findCandidates(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    nodes []*framework.NodeInfo,
    m framework.NodeToStatusMap,
) ([]Candidate, error) {

    var candidates []Candidate

    for _, node := range nodes {
        // Check if pod can fit after preemption
        victims, fits := pl.selectVictimsOnNode(
            ctx, state, pod, node, m,
        )

        if !fits {
            continue
        }

        // Validate PodDisruptionBudget
        if !pl.validatePDB(victims) {
            continue
        }

        candidates = append(candidates, Candidate{
            Node:    node,
            Victims: victims,
        })
    }

    return candidates, nil
}

// selectVictimsOnNode selects pods to preempt on a node
func (pl *DefaultPreemption) selectVictimsOnNode(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    node *framework.NodeInfo,
    m framework.NodeToStatusMap,
) ([]*v1.Pod, bool) {

    podPriority := pod.Spec.Priority

    // Get all pods on node sorted by priority
    pods := node.Pods
    sort.Slice(pods, func(i, j int) bool {
        return *pods[i].Pod.Spec.Priority < *pods[j].Pod.Spec.Priority
    })

    var victims []*v1.Pod
    resourcesReleased := framework.NewResource()

    // Add victims until pod fits
    for _, p := range pods {
        // Skip higher or equal priority pods
        if *p.Pod.Spec.Priority >= *podPriority {
            continue
        }

        victims = append(victims, p.Pod)
        resourcesReleased.Add(p.Pod)

        // Check if pod fits now
        if pl.podFitsAfterPreemption(
            ctx, state, pod, node, victims, resourcesReleased,
        ) {
            return victims, true
        }
    }

    return nil, false
}
```

**Location:** `pkg/scheduler/framework/plugins/defaultpreemption/default_preemption.go:50-250`

### Priority Plugin

**File:** `pkg/scheduler/framework/plugins/podpriority/pod_priority.go`

```go
// PodPriority is a plugin that orders pods by priority
type PodPriority struct{}

// Name returns plugin name
func (pl *PodPriority) Name() string {
    return "PodPriority"
}

// Less compares two pods by priority
func (pl *PodPriority) Less(
    pInfo1, pInfo2 *framework.QueuedPodInfo,
) bool {
    p1 := corev1helpers.PodPriority(pInfo1.Pod)
    p2 := corev1helpers.PodPriority(pInfo2.Pod)

    // Higher priority first
    if p1 != p2 {
        return p1 > p2
    }

    // Same priority: older pod first
    return pInfo1.Timestamp.Before(pInfo2.Timestamp)
}
```

**Location:** `pkg/scheduler/framework/plugins/podpriority/pod_priority.go:30-60`

### PriorityClass Validation

**File:** `pkg/apis/scheduling/validation/validation.go`

```go
// ValidatePriorityClass validates a PriorityClass
func ValidatePriorityClass(pc *scheduling.PriorityClass) field.ErrorList {
    allErrs := field.ErrorList{}

    // Validate name
    allErrs = append(
        allErrs,
        apivalidation.ValidateObjectMeta(
            &pc.ObjectMeta,
            false,
            ValidatePriorityClassName,
            field.NewPath("metadata"),
        )...,
    )

    // Validate value
    if pc.Value < scheduling.LowestUserDefinablePriority ||
       pc.Value > scheduling.HighestUserDefinablePriority {
        allErrs = append(
            allErrs,
            field.Invalid(
                field.NewPath("value"),
                pc.Value,
                fmt.Sprintf(
                    "must be between %d and %d",
                    scheduling.LowestUserDefinablePriority,
                    scheduling.HighestUserDefinablePriority,
                ),
            ),
        )
    }

    return allErrs
}
```

**Location:** `pkg/apis/scheduling/validation/validation.go:40-80`

## Priority Queue State Machine

```mermaid
stateDiagram-v2
    [*] --> ActiveQueue: Pod Created

    ActiveQueue --> Scheduling: Pop from Queue
    Scheduling --> Scheduled: Success
    Scheduling --> BackoffQueue: Transient Failure
    Scheduling --> UnschedulableQueue: Permanent Failure

    BackoffQueue --> ActiveQueue: Backoff Expired
    BackoffQueue --> ActiveQueue: Cluster Event

    UnschedulableQueue --> ActiveQueue: Node Added
    UnschedulableQueue --> ActiveQueue: Pod Updated
    UnschedulableQueue --> ActiveQueue: Resource Changed

    Scheduled --> [*]: Bound to Node

    note right of ActiveQueue
        Sorted by:
        1. Priority (desc)
        2. Timestamp (asc)
    end note

    note right of BackoffQueue
        Exponential backoff:
        1s, 2s, 4s, 8s...
        Max: 10s
    end note

    note right of UnschedulableQueue
        Waiting for:
        - New nodes
        - Resource changes
        - Pod updates
    end note
```

## Sequence Diagrams

### Pod Scheduling with Priority

```mermaid
sequenceDiagram
    participant U as User
    participant API as API Server
    participant S as Scheduler
    participant PQ as Priority Queue
    participant K as Kubelet

    U->>API: Create High Priority Pod
    API->>API: Validate PriorityClass
    API->>PQ: Add to Queue
    PQ->>PQ: Insert by Priority

    S->>PQ: Pop Highest Priority
    PQ->>S: Return Pod

    S->>S: Find Node
    alt Resources Available
        S->>API: Bind Pod to Node
        API->>K: Start Pod
        K->>API: Update Status: Running
    else No Resources
        S->>S: Check for Preemption
        S->>API: Query Lower Priority Pods
        API->>S: Return Candidates
        S->>S: Select Victims
        S->>API: Delete Victim Pods
        API->>K: Evict Pods
        K->>API: Pods Terminated
        S->>API: Bind High Priority Pod
        API->>K: Start Pod
    end
```

### Preemption Sequence

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant PA as Preemption Algorithm
    participant N as Node Evaluator
    participant P as PDB Checker
    participant API as API Server
    participant K as Kubelet

    S->>PA: Pod Unschedulable
    PA->>N: Evaluate All Nodes

    loop For Each Node
        N->>N: Get Running Pods
        N->>N: Sort by Priority
        N->>N: Select Victims
        N->>P: Validate PDB

        alt PDB OK
            P->>N: Approved
            N->>PA: Add Candidate
        else PDB Violated
            P->>N: Rejected
        end
    end

    PA->>PA: Select Best Node
    PA->>API: Nominate Pod for Node
    API->>API: Set NominatedNodeName

    loop For Each Victim
        PA->>API: Delete Pod
        API->>K: Evict Pod
        K->>API: Pod Terminated
    end

    PA->>S: Retry Scheduling
    S->>API: Bind Pod to Node
```

## Configuration Examples

### PriorityClass Definition

```yaml
# High priority for critical system components
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-critical
value: 1000000000
globalDefault: false
description: "Critical system components"
---
# Medium priority for production workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-high
value: 10000
globalDefault: false
description: "High priority production workloads"
---
# Low priority for batch jobs
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 100
globalDefault: false
description: "Low priority batch processing"
preemptionPolicy: Never  # Cannot preempt other pods
---
# Default priority
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 0
globalDefault: true
description: "Default priority for pods"
```

### Pod with Priority

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
  labels:
    app: critical-service
spec:
  priorityClassName: system-critical

  containers:
  - name: app
    image: critical-service:1.0
    resources:
      requests:
        memory: "2Gi"
        cpu: "1000m"
      limits:
        memory: "4Gi"
        cpu: "2000m"

  # Prevent preemption of this pod
  preemptionPolicy: Never
```

### Deployment with Priority and PDB

```yaml
# Deployment with high priority
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      priorityClassName: production-high

      containers:
      - name: frontend
        image: frontend:1.0
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
---
# PodDisruptionBudget to limit preemption
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: frontend
```

### Scheduler Configuration with Preemption

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    # Queue sorting plugin
    queueSort:
      enabled:
      - name: PrioritySort

    # Preemption plugin
    postFilter:
      enabled:
      - name: DefaultPreemption
      disabled:
      - name: "*"

  pluginConfig:
  - name: DefaultPreemption
    args:
      # Minimum candidate nodes to consider
      minCandidateNodesPercentage: 10
      minCandidateNodesAbsolute: 100
```

## Monitoring and Metrics

### Key Metrics

```yaml
# Priority queue metrics
scheduler_queue_incoming_pods_total{queue="active"}
scheduler_queue_incoming_pods_total{queue="backoff"}
scheduler_queue_incoming_pods_total{queue="unschedulable"}

# Preemption metrics
scheduler_preemption_attempts_total
scheduler_preemption_victims_total

# Scheduling metrics by priority
scheduler_pod_scheduling_duration_seconds{priority="1000000000"}
scheduler_pod_scheduling_duration_seconds{priority="10000"}
scheduler_pod_scheduling_duration_seconds{priority="0"}

# Queue depth by priority
scheduler_pending_pods{queue="active",priority="high"}
scheduler_pending_pods{queue="active",priority="medium"}
scheduler_pending_pods{queue="active",priority="low"}
```

### Prometheus Queries

```promql
# Preemption rate
rate(scheduler_preemption_attempts_total[5m])

# Average victims per preemption
rate(scheduler_preemption_victims_total[5m]) /
rate(scheduler_preemption_attempts_total[5m])

# Queue depth by priority
sum(scheduler_pending_pods{queue="active"}) by (priority)

# Scheduling latency by priority
histogram_quantile(0.95,
  rate(scheduler_pod_scheduling_duration_seconds_bucket[5m])
) by (priority)

# Preemption impact
sum(rate(pod_evictions_total{reason="Preempted"}[5m]))
```

## Troubleshooting Guide

### Common Issues

#### Issue 1: High Priority Pod Not Preempting

**Symptoms:**
- High priority pod remains pending
- Lower priority pods running on nodes
- No preemption attempts in logs

**Diagnosis:**
```bash
# Check pod priority
kubectl get pod high-priority-pod -o jsonpath='{.spec.priority}'

# Check PriorityClass
kubectl get priorityclass

# Check scheduler logs
kubectl logs -n kube-system kube-scheduler-xxx | grep -i preempt

# Check pod events
kubectl describe pod high-priority-pod
```

**Common Causes:**
1. **PreemptionPolicy set to Never**
   ```yaml
   spec:
     preemptionPolicy: Never  # Cannot preempt
   ```

2. **PodDisruptionBudget blocking preemption**
   ```bash
   # Check PDBs
   kubectl get pdb -A
   kubectl describe pdb frontend-pdb
   ```

3. **No suitable victims**
   - All running pods have higher/equal priority
   - Pod anti-affinity prevents preemption

**Resolution:**
```bash
# Update preemption policy
kubectl patch pod high-priority-pod -p '
{
  "spec": {
    "preemptionPolicy": "PreemptLowerPriority"
  }
}'

# Adjust PDB
kubectl patch pdb frontend-pdb -p '
{
  "spec": {
    "minAvailable": 1
  }
}'
```

#### Issue 2: Excessive Preemption

**Symptoms:**
- Frequent pod evictions
- Unstable workloads
- High scheduling churn

**Diagnosis:**
```bash
# Check preemption rate
kubectl get events -A --field-selector reason=Preempted

# Check evicted pods
kubectl get pods -A --field-selector status.phase=Failed

# Analyze preemption patterns
kubectl get events -A --sort-by='.lastTimestamp' | grep Preempted
```

**Common Causes:**
1. **Too many priority classes**
2. **Insufficient cluster resources**
3. **Missing resource requests**
4. **Priority inversion**

**Resolution:**
```bash
# Consolidate priority classes
kubectl get priorityclass

# Add resource requests
kubectl patch deployment frontend -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "app",
          "resources": {
            "requests": {
              "memory": "512Mi",
              "cpu": "500m"
            }
          }
        }]
      }
    }
  }
}'

# Add PodDisruptionBudget
kubectl apply -f pdb.yaml
```

#### Issue 3: Priority Queue Backup

**Symptoms:**
- Increasing queue depth
- Slow scheduling
- Pods stuck in pending

**Diagnosis:**
```bash
# Check scheduler metrics
kubectl get --raw /metrics | grep scheduler_queue

# Check pending pods
kubectl get pods -A --field-selector status.phase=Pending

# Check scheduler health
kubectl get componentstatus scheduler
```

**Common Causes:**
1. **Scheduler overload**
2. **Too many unschedulable pods**
3. **Node capacity issues**

**Resolution:**
```bash
# Scale up cluster
kubectl scale nodes --replicas=10

# Delete unschedulable pods
kubectl delete pods --field-selector status.phase=Pending \
  --all-namespaces

# Restart scheduler
kubectl delete pod -n kube-system kube-scheduler-xxx
```

### Debug Commands

```bash
# View priority queue state
kubectl get --raw /debug/api_priority_and_fairness/dump_priority_levels

# Check pod scheduling decisions
kubectl get events --sort-by='.lastTimestamp' \
  --field-selector involvedObject.kind=Pod

# Analyze preemption decisions
kubectl logs -n kube-system kube-scheduler-xxx \
  --tail=1000 | grep -A 10 "preemption"

# Check nominated pods
kubectl get pods -A -o json | jq '.items[] |
  select(.status.nominatedNodeName != null) |
  {name: .metadata.name, node: .status.nominatedNodeName}'

# Monitor scheduling latency
kubectl get --raw /metrics | grep scheduler_pod_scheduling_duration
```

## Best Practices

### Priority Class Design

1. **Use System Priorities Sparingly**
   ```yaml
   # Reserve 1B+ for critical system components only
   apiVersion: scheduling.k8s.io/v1
   kind: PriorityClass
   metadata:
     name: system-node-critical
   value: 2000001000
   ```

2. **Create Logical Tiers**
   ```yaml
   # Critical services: 10000
   # Production: 1000
   # Staging: 100
   # Development: 10
   # Batch: 0
   ```

3. **Document Priority Rationale**
   ```yaml
   description: "Frontend services requiring <1s response time"
   ```

### Preemption Guidelines

1. **Protect Critical Workloads with PDB**
   ```yaml
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: critical-pdb
   spec:
     maxUnavailable: 1
     selector:
       matchLabels:
         tier: critical
   ```

2. **Use PreemptionPolicy Judiciously**
   ```yaml
   spec:
     priorityClassName: critical
     preemptionPolicy: Never  # For truly critical pods
   ```

3. **Set Appropriate Resource Requests**
   ```yaml
   resources:
     requests:
       memory: "1Gi"  # Realistic request
       cpu: "500m"
   ```

### Queue Management

1. **Monitor Queue Depth**
   - Alert on excessive pending pods
   - Track queue latency metrics

2. **Tune Backoff Parameters**
   ```yaml
   pluginConfig:
   - name: DefaultPreemption
     args:
       podInitialBackoffSeconds: 1
       podMaxBackoffSeconds: 10
   ```

3. **Handle Unschedulable Pods**
   - Delete permanently unschedulable pods
   - Fix cluster capacity issues

## Performance Considerations

### Preemption Cost

**Factors:**
- Number of nodes to evaluate
- Number of pods per node
- PDB validation overhead
- Pod termination time

**Optimization:**
```yaml
# Limit candidate nodes
pluginConfig:
- name: DefaultPreemption
  args:
    minCandidateNodesPercentage: 10  # Evaluate 10% of nodes
    minCandidateNodesAbsolute: 100   # At least 100 nodes
```

### Queue Performance

**Considerations:**
- Priority queue is O(log n) for insert/pop
- Lock contention with high pod churn
- Memory usage with large queue depth

**Tuning:**
```yaml
# Batch processing
--kube-api-qps=100
--kube-api-burst=200

# Scheduling rate
--pods-per-core=10
```

## Related Components

- **Scheduler** (`pkg/scheduler/`): Implements priority queue and preemption
- **PriorityClass API** (`pkg/apis/scheduling/`): Priority class definitions
- **Pod Admission** (`plugin/pkg/admission/priority/`): Priority validation
- **Resource Quota** (`pkg/quota/`): Quota enforcement with priority
- **PodDisruptionBudget** (`pkg/controller/disruption/`): Limits preemption impact

## References

- **KEP-902**: [Pod Priority and Preemption](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/902-pod-priority-preemption)
- **KEP-1845**: [PreemptionPolicy](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/1845-preemption-policy)
- **Design Doc**: [Priority and Preemption](https://github.com/kubernetes/design-proposals-archive/blob/main/scheduling/pod-priority-api.md)
- **Source Code**: `pkg/scheduler/framework/plugins/defaultpreemption/`
- **API Reference**: [PriorityClass v1](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/priority-class-v1/)
