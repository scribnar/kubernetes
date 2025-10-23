# Kubernetes Scheduler: System Architecture

**Document Version:** 1.0
**Last Updated:** 2025-10-20
**Status:** Living Document

---

## Table of Contents

1. [Component Architecture Overview](#component-architecture-overview)
2. [Major Subsystems](#major-subsystems)
3. [Component Interactions](#component-interactions)
4. [Data Flow Architecture](#data-flow-architecture)
5. [Layered Architecture View](#layered-architecture-view)
6. [Thread and Concurrency Model](#thread-and-concurrency-model)

---

## Component Architecture Overview

The Kubernetes Scheduler is composed of six major subsystems that work together to assign pods to nodes:

```mermaid
C4Component
    title Component Diagram - Kubernetes Scheduler

    Container_Boundary(scheduler, "Scheduler") {
        Component(core, "Scheduler Core", "Go", "Orchestrates scheduling workflow")
        Component(queue, "Queue Management", "PriorityQueue", "Three-tier queue system")
        Component(cache, "Cache System", "cacheImpl", "Cluster state snapshot and assume/bind tracking")
        Component(framework, "Plugin Framework", "frameworkImpl", "Extensible plugin execution engine")
        Component(events, "Event Handlers", "Go functions", "Process cluster state changes")
        Component(metrics, "Metrics Collector", "Prometheus", "Observability and monitoring")
    }

    Container_Ext(api, "API Server", "Kubernetes", "Cluster state storage and coordination")
    Container_Ext(informers, "Shared Informers", "client-go", "Watch and cache API resources")

    Rel(core, queue, "Pop pod, Activate pods", "Interface call")
    Rel(core, cache, "Snapshot, Assume, Bind", "Interface call")
    Rel(core, framework, "Execute plugins", "Interface call")
    Rel(queue, events, "Requeue on events", "Event callbacks")
    Rel(cache, events, "Update on events", "Event callbacks")
    Rel(informers, events, "Trigger handlers", "Watch callbacks")
    Rel(informers, api, "List/Watch resources", "HTTPS")
    Rel(core, api, "Bind pods", "HTTPS client")
    Rel(core, metrics, "Record metrics", "Function calls")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

---

## Major Subsystems

### 1. Scheduler Core (`pkg/scheduler/scheduler.go`)

**Responsibility**: Orchestrate the scheduling workflow and coordinate between subsystems.

**Key Components**:
- **Scheduler struct** (`scheduler.go:72`): Main scheduler object
- **ScheduleOne()** (`schedule_one.go:66`): Main scheduling loop
- **schedulingCycle()** (`schedule_one.go:150`): Synchronous scheduling phase
- **bindingCycle()** (`schedule_one.go:278`): Asynchronous binding phase

**Core Fields**:
```go
type Scheduler struct {
    Cache             internalcache.Cache           // Cluster state cache
    SchedulingQueue   internalqueue.SchedulingQueue // Priority queue
    Profiles          profile.Map                   // Framework instances per profile
    Extenders         []fwk.Extender               // External schedulers (optional)
    NextPod           func() (*framework.QueuedPodInfo, error) // Pop from queue
    SchedulePod       func(...) (ScheduleResult, error)        // Scheduling algorithm
    FailureHandler    FailureHandlerFn             // Handle scheduling failures
    StopEverything    <-chan struct{}              // Shutdown signal
    APIDispatcher     *apidispatcher.APIDispatcher // Async API calls (feature gated)
    // ...
}
```

**Source Reference**: `pkg/scheduler/scheduler.go:72-125`

---

### 2. Queue Management Subsystem (`pkg/scheduler/backend/queue/`)

**Responsibility**: Manage pods waiting to be scheduled using a three-tier queue system.

```mermaid
graph TB
    subgraph "Queue Subsystem"
        A[PriorityQueue] --> B[activeQ - Heap]
        A --> C[backoffQ - Heap with Backoff]
        A --> D[unschedulableQ - Map]
        A --> E[nominator - Nominated Pods]
    end

    F[New Pod] --> B
    G[Cluster Event] --> B
    G --> C
    B --> H[Pop for Scheduling]
    H --> I{Success?}
    I -->|Yes| J[Scheduled]
    I -->|No Transient| C
    I -->|No Permanent| D
    C -->|Backoff Expired| B
    D -->|Cluster Event Match| B
    D -->|Cluster Event Match| C

    style B fill:#d4f1d4
    style C fill:#fff4d4
    style D fill:#ffe1e1
```

**Key Components**:

1. **PriorityQueue** (`scheduling_queue.go:163`):
   - Main queue structure
   - Coordinates all three sub-queues
   - Implements `SchedulingQueue` interface

2. **activeQ** (`active_queue.go`):
   - Heap-based priority queue
   - Pods ready for immediate scheduling
   - Sorted by QueueSort plugin (default: priority/timestamp)

3. **backoffQ** (`backoff_queue.go`):
   - Heap with expiry time tracking
   - Exponential backoff: 1s (initial) to 10s (max)
   - Automatically moves to activeQ when backoff expires

4. **unschedulableQ** (`unschedulable_pods.go`):
   - Map-based storage
   - Pods waiting for cluster state changes
   - Event-driven requeuing via queueing hints

5. **nominator** (`nominator.go`):
   - Tracks nominated nodes for preempted pods
   - Used during node evaluation to consider nominated pods

**Source Reference**: `pkg/scheduler/backend/queue/scheduling_queue.go`

---

### 3. Cache System (`pkg/scheduler/backend/cache/`)

**Responsibility**: Maintain consistent, fast access to cluster state and track assumed pods.

```mermaid
classDiagram
    class Cache {
        <<interface>>
        +AssumePod(pod) error
        +ForgetPod(pod) error
        +FinishBinding(pod) error
        +AddPod(pod) error
        +UpdatePod(old, new) error
        +RemovePod(pod) error
        +AddNode(node) NodeInfo
        +UpdateNode(old, new) NodeInfo
        +RemoveNode(node) error
        +UpdateSnapshot(snapshot) error
        +Dump() Dump
    }

    class cacheImpl {
        -mu sync.RWMutex
        -assumedPods Set~string~
        -podStates map~string~*podState
        -nodes map~string~*nodeInfoListItem
        -headNode *nodeInfoListItem
        -nodeTree *nodeTree
        -imageStates map~string~*ImageStateSummary
        -apiDispatcher APIDispatcher
        +AssumePod(pod)
        +FinishBinding(pod)
        +UpdateSnapshot(snapshot)
    }

    class nodeInfoListItem {
        +info *NodeInfo
        +next *nodeInfoListItem
        +prev *nodeInfoListItem
    }

    class podState {
        +pod *Pod
        +deadline *time.Time
        +bindingFinished bool
    }

    class Snapshot {
        +nodeInfoMap map~string~*NodeInfo
        +nodeInfoList []NodeInfo
        +havePodsWithAffinityNodeInfoList []NodeInfo
        +havePodsWithRequiredAntiAffinityNodeInfoList []NodeInfo
        +usedPVCSet Set~string~
        +generation int64
    }

    Cache <|-- cacheImpl
    cacheImpl o-- nodeInfoListItem
    cacheImpl o-- podState
    cacheImpl ..> Snapshot
    nodeInfoListItem --> nodeInfoListItem : next/prev
```

**Key Features**:

1. **Assume/Bind Mechanism**:
   - `AssumePod()`: Mark pod as assumed (optimistic scheduling)
   - `FinishBinding()`: Mark binding complete (allows expiration)
   - `ForgetPod()`: Rollback on binding failure

2. **Doubly-Linked List**:
   - Most recently updated nodes at head
   - Efficient snapshot generation (stop at last snapshot's generation)
   - O(1) move to head operation

3. **Snapshot System**:
   - Consistent view of cluster state during scheduling
   - Generation-based incremental updates
   - Separate lists for affinity tracking

**Source Reference**: `pkg/scheduler/backend/cache/cache.go:61-109`

---

### 4. Plugin Framework (`pkg/scheduler/framework/`)

**Responsibility**: Execute plugins at defined extension points during scheduling.

```mermaid
graph LR
    subgraph "Plugin Framework"
        A[frameworkImpl] --> B[Plugin Registry]
        A --> C[Extension Point Executors]
        A --> D[Parallelizer]
        A --> E[Waiting Pods Map]
        A --> F[Metrics Recorder]

        C --> C1[RunPreFilterPlugins]
        C --> C2[RunFilterPlugins]
        C --> C3[RunPostFilterPlugins]
        C --> C4[RunPreScorePlugins]
        C --> C5[RunScorePlugins]
        C --> C6[RunReservePlugins]
        C --> C7[RunPermitPlugins]
        C --> C8[RunPreBindPlugins]
        C --> C9[RunBindPlugins]
        C --> C10[RunPostBindPlugins]
    end

    G[Scheduler Core] --> A
    B --> H[In-Tree Plugins]
    B --> I[Out-of-Tree Plugins]

    style A fill:#e1f5ff
```

**Key Components**:

1. **frameworkImpl** (`framework/runtime/framework.go:54`):
   ```go
   type frameworkImpl struct {
       registry             Registry                     // Plugin factory functions
       snapshotSharedLister fwk.SharedLister            // Snapshot interface
       waitingPods          *waitingPodsMap             // Permit plugin coordination
       scorePluginWeight    map[string]int              // Plugin scoring weights

       // Plugin slices for each extension point
       preEnqueuePlugins    []fwk.PreEnqueuePlugin
       preFilterPlugins     []fwk.PreFilterPlugin
       filterPlugins        []fwk.FilterPlugin
       postFilterPlugins    []fwk.PostFilterPlugin
       preScorePlugins      []fwk.PreScorePlugin
       scorePlugins         []fwk.ScorePlugin
       reservePlugins       []fwk.ReservePlugin
       permitPlugins        []fwk.PermitPlugin
       preBindPlugins       []fwk.PreBindPlugin
       bindPlugins          []fwk.BindPlugin
       postBindPlugins      []fwk.PostBindPlugin
       // ...
   }
   ```

2. **CycleState** (thread-safe state container):
   - Stores plugin-specific data during scheduling
   - Uses `sync.Map` for concurrent access
   - Passed through all plugin extension points

3. **WaitingPodsMap**:
   - Coordinates Permit plugins
   - Allows plugins to signal approval/rejection asynchronously
   - Implements timeouts (max 15 minutes)

**Source Reference**: `pkg/scheduler/framework/runtime/framework.go:54-94`

---

### 5. Event Handling Subsystem (`pkg/scheduler/eventhandlers.go`)

**Responsibility**: React to cluster state changes and trigger appropriate actions.

**Event Handlers**:

```mermaid
sequenceDiagram
    participant Informer
    participant EventHandler
    participant Cache
    participant Queue

    Note over Informer,Queue: Node Add Event
    Informer->>EventHandler: addNodeToCache()
    EventHandler->>Cache: AddNode(node)
    Cache-->>EventHandler: nodeInfo
    EventHandler->>Queue: MoveAllToActiveOrBackoffQueue(NodeAdd, node)

    Note over Informer,Queue: Pod Update Event (Unscheduled)
    Informer->>EventHandler: updatePodInSchedulingQueue()
    EventHandler->>EventHandler: Check if assumed
    EventHandler->>Queue: Update(oldPod, newPod)

    Note over Informer,Queue: Pod Add Event (Scheduled)
    Informer->>EventHandler: addPodToCache()
    EventHandler->>Cache: AddPod(pod)
    EventHandler->>Queue: MoveAllToActiveOrBackoffQueue(PodAdd, pod)
```

**Event Types Handled**:

| Resource | Event Type | Handler Function | Actions |
|----------|-----------|------------------|---------|
| Node | Add | `addNodeToCache` | Update cache, requeue pods |
| Node | Update | `updateNodeInCache` | Update cache, requeue if node more schedulable |
| Node | Delete | `deleteNodeFromCache` | Remove from cache, requeue all pods |
| Pod (unscheduled) | Add | `addPodToSchedulingQueue` | Add to queue |
| Pod (unscheduled) | Update | `updatePodInSchedulingQueue` | Update in queue |
| Pod (unscheduled) | Delete | `deletePodFromSchedulingQueue` | Remove from queue |
| Pod (scheduled) | Add | `addPodToCache` | Add to cache, requeue waiting pods |
| Pod (scheduled) | Update | `updatePodInCache` | Update cache, requeue if resources freed |
| Pod (scheduled) | Delete | `deletePodFromCache` | Remove from cache, requeue waiting pods |
| PVC | Add/Update/Delete | `addPVCToCache`, etc. | Update cache, requeue pods waiting for volumes |
| PV | Add/Update/Delete | `addPVToCache`, etc. | Update cache, requeue pods |
| StorageClass | Add | `addStorageClassToQueue` | Requeue pods waiting for storage |
| CSINode | Add/Update | `addCSINodeToCache`, etc. | Update cache, requeue pods with volume requirements |

**Source Reference**: `pkg/scheduler/eventhandlers.go`

---

### 6. Metrics and Observability (`pkg/scheduler/metrics/`)

**Responsibility**: Collect and expose metrics for monitoring and debugging.

**Key Metrics**:

1. **Scheduling Latency**:
   - `scheduler_scheduling_algorithm_latency_microseconds`: Time to schedule a pod
   - `scheduler_binding_duration_seconds`: Time to bind a pod
   - `scheduler_e2e_scheduling_duration_seconds`: End-to-end scheduling time

2. **Queue Metrics**:
   - `scheduler_queue_incoming_pods_total`: Pods added to queue
   - `scheduler_pending_pods`: Pods in queue by state (active, backoff, unschedulable)
   - `scheduler_pod_scheduling_attempts`: Histogram of scheduling attempts per pod

3. **Framework Metrics**:
   - `scheduler_plugin_execution_duration_seconds`: Plugin execution time per extension point
   - `scheduler_framework_extension_point_duration_seconds`: Extension point total duration

4. **Event Metrics**:
   - `scheduler_event_handling_duration_seconds`: Event handler execution time

5. **Goroutine Metrics**:
   - `scheduler_goroutines`: Number of goroutines by operation (binding, prioritizing)

**Source Reference**: `pkg/scheduler/metrics/`

---

## Component Interactions

### Scheduling Flow with Component Interactions

```mermaid
sequenceDiagram
    autonumber
    participant Q as Queue
    participant Core as Scheduler Core
    participant Cache as Cache
    participant FW as Framework
    participant API as API Server

    Note over Q,API: Scheduling Cycle (Synchronous)
    Core->>Q: Pop()
    Q-->>Core: podInfo

    Core->>Cache: UpdateSnapshot(snapshot)
    Cache-->>Core: snapshot updated

    Core->>FW: RunPreFilterPlugins(pod)
    FW-->>Core: preFilterResult, status

    Core->>FW: RunFilterPlugins(pod, nodes)
    Note over FW: Parallel execution across nodes
    FW-->>Core: feasibleNodes[]

    Core->>FW: RunPreScorePlugins(pod, nodes)
    FW-->>Core: status

    Core->>FW: RunScorePlugins(pod, nodes)
    Note over FW: Parallel execution across nodes
    FW-->>Core: nodeScores[]

    Core->>Core: selectHost(nodeScores)

    Core->>Cache: AssumePod(pod, selectedNode)
    Cache-->>Core: assumed

    Core->>FW: RunReservePluginsReserve(pod, node)
    FW-->>Core: status

    Core->>FW: RunPermitPlugins(pod, node)
    FW-->>Core: permitStatus

    Note over Q,API: Binding Cycle (Asynchronous Goroutine)
    Core->>Core: go bindingCycle()

    par Binding Goroutine
        Core->>FW: WaitOnPermit(pod)
        FW-->>Core: status

        Core->>Q: Done(pod.UID)

        Core->>FW: RunPreBindPlugins(pod, node)
        FW-->>Core: status

        Core->>FW: RunBindPlugins(pod, node)
        FW->>API: Create Binding
        API-->>FW: bound
        FW-->>Core: status

        Core->>Cache: FinishBinding(pod)

        Core->>FW: RunPostBindPlugins(pod, node)
    end
```

---

## Data Flow Architecture

### Data Flow Diagram

```mermaid
graph TB
    subgraph "Input"
        A1[API Server Events]
        A2[Pod Add/Update]
        A3[Node Add/Update]
        A4[Resource Events]
    end

    subgraph "Event Processing"
        B1[Shared Informers]
        B2[Event Handlers]
    end

    subgraph "Queue Management"
        C1[activeQ]
        C2[backoffQ]
        C3[unschedulableQ]
    end

    subgraph "Scheduling Engine"
        D1[Pop Pod]
        D2[Snapshot Cache]
        D3[Run Plugins]
        D4[Select Node]
    end

    subgraph "State Management"
        E1[Assume Pod]
        E2[Reserve Resources]
        E3[Permit Check]
    end

    subgraph "Output"
        F1[Bind Pod - Async]
        F2[Update API Server]
        F3[Emit Metrics]
    end

    A1 --> B1
    A2 --> B1
    A3 --> B1
    A4 --> B1

    B1 --> B2
    B2 --> C1
    B2 --> C2
    B2 --> C3

    C1 --> D1
    C2 --> D1
    C3 --> D1

    D1 --> D2
    D2 --> D3
    D3 --> D4

    D4 --> E1
    E1 --> E2
    E2 --> E3

    E3 --> F1
    F1 --> F2
    D3 --> F3

    style D3 fill:#e1f5ff
    style E1 fill:#fff4d4
    style F1 fill:#d4f1d4
```

---

## Layered Architecture View

```mermaid
graph TB
    subgraph "Layer 1: Interface & Entry Point"
        L1A[CLI - cmd/kube-scheduler]
        L1B[Configuration Loading]
        L1C[Server Setup]
    end

    subgraph "Layer 2: Orchestration"
        L2A[Scheduler Core]
        L2B[ScheduleOne Loop]
        L2C[Failure Handler]
    end

    subgraph "Layer 3: Scheduling Logic"
        L3A[Plugin Framework]
        L3B[Scheduling Algorithm]
        L3C[Preemption Logic]
    end

    subgraph "Layer 4: State Management"
        L4A[Queue Management]
        L4B[Cache System]
        L4C[Snapshot Generation]
    end

    subgraph "Layer 5: Integration"
        L5A[Event Handlers]
        L5B[API Client]
        L5C[Shared Informers]
    end

    subgraph "Layer 6: Cross-Cutting Concerns"
        L6A[Metrics Collection]
        L6B[Logging]
        L6C[Profiling]
    end

    L1A --> L1B --> L1C --> L2A
    L2A --> L2B
    L2B --> L3A
    L2B --> L3B
    L3A --> L3C

    L3A --> L4A
    L3B --> L4B
    L3B --> L4C

    L4A --> L5A
    L4B --> L5C
    L5A --> L5B
    L5C --> L5B

    L2A -.-> L6A
    L3A -.-> L6A
    L4A -.-> L6A
    L5A -.-> L6A

    L2A -.-> L6B
    L3A -.-> L6B
```

---

## Thread and Concurrency Model

### Goroutine Structure

```mermaid
graph TB
    Main[Main Goroutine]

    Main --> SQ[ScheduleOne Loop<br/>Goroutine]
    Main --> BQ[Backoff Queue<br/>Flush Goroutine]
    Main --> UQ[Unschedulable Queue<br/>Flush Goroutine]
    Main --> CP[Cache Expiry<br/>Goroutine]
    Main --> AD[API Dispatcher<br/>Workers Optional]

    SQ --> BG1[Binding Goroutine 1]
    SQ --> BG2[Binding Goroutine 2]
    SQ --> BGN[Binding Goroutine N]

    BG1 --> EP1[Extender Prioritize<br/>Goroutines]

    SQ --> FW1[Filter Workers 1-16]
    SQ --> SW1[Score Workers 1-16]

    style Main fill:#e1f5ff
    style SQ fill:#d4f1d4
    style BG1 fill:#fff4d4
    style BG2 fill:#fff4d4
    style BGN fill:#fff4d4
```

**Goroutine Breakdown**:

1. **Main Goroutine**: Initializes scheduler and starts subsystems
2. **ScheduleOne Loop** (`Run()` in `scheduler.go:524`): Continuously pops and schedules pods
3. **Backoff Queue Flush**: Moves pods from backoffQ to activeQ when backoff expires
4. **Unschedulable Queue Flush**: Periodically moves long-waiting pods back to activeQ
5. **Cache Expiry Cleanup**: Removes expired assumed pods
6. **Binding Goroutines**: One per scheduled pod (asynchronous binding)
7. **Filter/Score Workers**: Pool of workers for parallel plugin execution
8. **Extender Prioritize Goroutines**: One per extender (if configured)
9. **API Dispatcher Workers** (feature gated): Async API call workers

**Synchronization Points**:
- Queue Pop: Blocks until pod available
- Permit Wait: Blocks until plugins approve
- WaitGroup: Extender prioritization
- Context Cancellation: Graceful shutdown

---

## Next Steps

Continue to [03-deployment-and-runtime.md](./03-deployment-and-runtime.md) for deployment architecture and runtime initialization details.

---

**Document Status**: Complete
**Next Review**: Upon scheduler refactoring or major architectural changes
