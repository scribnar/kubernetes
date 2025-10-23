# Kube-Controller-Manager: Shared Infrastructure

**Document Version**: 1.0
**Last Updated**: 2025-10-22
**Status**: Draft

---

## 1. Overview

This document provides deep technical analysis of the shared infrastructure components used by all controllers in kube-controller-manager. These components enable efficient, scalable, and reliable cluster state reconciliation.

## 2. Shared Informer Framework

### 2.1 Architecture Overview

```mermaid
graph TB
    subgraph "API Server"
        API[kube-apiserver<br/>Watch Endpoints]
    end

    subgraph "SharedInformerFactory"
        FACTORY[Factory Manager]

        subgraph "Per-Resource Informers"
            direction TB
            INF_POD[Pod Informer]
            INF_DEP[Deployment Informer]
            INF_SVC[Service Informer]
            INF_MORE[... 20+ more]
        end
    end

    subgraph "Single Informer Components"
        direction LR
        REF[Reflector<br/>List & Watch]
        DELTA[DeltaFIFO<br/>Event Queue]
        STORE[Indexer/Store<br/>Thread-safe Cache]
        PROC[Process Loop]
        DIST[Event Distribution]
    end

    subgraph "Multiple Event Handlers"
        direction TB
        H1[Deployment Controller<br/>OnAdd/OnUpdate/OnDelete]
        H2[ReplicaSet Controller<br/>OnAdd/OnUpdate/OnDelete]
        H3[DaemonSet Controller<br/>OnAdd/OnUpdate/OnDelete]
    end

    API -->|Single Watch| REF
    REF --> DELTA
    DELTA --> STORE
    DELTA --> PROC
    PROC --> DIST
    DIST --> H1
    DIST --> H2
    DIST --> H3

    FACTORY --> INF_POD
    FACTORY --> INF_DEP
    FACTORY --> INF_SVC
    FACTORY --> INF_MORE

    INF_POD -.-> REF
    H1 -.->|Get from cache| STORE
    H2 -.->|Get from cache| STORE
    H3 -.->|Get from cache| STORE

    style STORE fill:#9f9,stroke:#333,stroke-width:2px
    style DIST fill:#99f,stroke:#333,stroke-width:2px
```

### 2.2 Reflector Component

**Purpose**: Maintains a watch on the API server and feeds changes to the local cache.

**Source**: `staging/src/k8s.io/client-go/tools/cache/reflector.go`

```mermaid
sequenceDiagram
    participant Reflector
    participant API as kube-apiserver
    participant Store as Local Store
    participant Queue as DeltaFIFO

    Note over Reflector: Initial List
    Reflector->>API: LIST /api/v1/pods
    API-->>Reflector: All pods + resourceVersion
    Reflector->>Queue: Sync(allPods)
    Queue->>Store: Replace(allPods)

    Note over Reflector: Continuous Watch
    Reflector->>API: WATCH /api/v1/pods?resourceVersion=X

    loop Watch Events
        alt ADDED
            API-->>Reflector: ADDED pod-new
            Reflector->>Queue: Add(Added, pod-new)
        else MODIFIED
            API-->>Reflector: MODIFIED pod-x
            Reflector->>Queue: Add(Updated, pod-x)
        else DELETED
            API-->>Reflector: DELETED pod-y
            Reflector->>Queue: Add(Deleted, pod-y)
        else ERROR
            API-->>Reflector: Watch closed/error
            Reflector->>Reflector: Exponential backoff
            Reflector->>API: Re-LIST (full resync)
        end
    end
```

**Key Features**:

1. **List-Watch Pattern**:
   ```go
   // Initial LIST
   list, err := r.listerWatcher.List(options)
   resourceVersion := listMetaInterface.GetResourceVersion()

   // Then WATCH from that version
   watcher, err := r.listerWatcher.Watch(options)
   ```

2. **Error Recovery**:
   ```go
   // On watch error, exponential backoff then full relist
   wait.BackoffUntil(func() {
       if err := r.ListAndWatch(stopCh); err != nil {
           utilruntime.HandleError(err)
       }
   }, r.backoffManager, true, stopCh)
   ```

3. **Periodic Resync**:
   ```go
   // Default: every 12 hours
   // Ensures cache consistency even if events were missed
   resyncPeriod := 12 * time.Hour
   ```

### 2.3 DeltaFIFO Queue

**Purpose**: Queues resource deltas (changes) for processing.

**Structure**:
```go
type DeltaFIFO struct {
    items map[string]Deltas  // key -> list of deltas
    queue []string            // ordered keys
    populated bool
    initialPopulationCount int
}

type Delta struct {
    Type   DeltaType  // Added, Updated, Deleted, Sync
    Object interface{}
}
```

**Processing Flow**:

```mermaid
flowchart TD
    EVENT[Reflector Event]
    KEY[Calculate Key<br/>namespace/name]
    CHECK{Key exists<br/>in map?}
    APPEND[Append delta to existing]
    NEW[Create new delta list]
    ENQUEUE[Add key to queue<br/>if not present]
    POP[Pop() from queue]
    PROCESS[Process all deltas for key]
    UPDATE[Update cache]
    NOTIFY[Notify handlers]
    DELETE[Delete from map]

    EVENT --> KEY
    KEY --> CHECK
    CHECK -->|Yes| APPEND
    CHECK -->|No| NEW
    APPEND --> ENQUEUE
    NEW --> ENQUEUE

    POP --> PROCESS
    PROCESS --> UPDATE
    UPDATE --> NOTIFY
    NOTIFY --> DELETE
```

**Deduplication**:
```go
// Multiple events for same resource are compressed
deltas := []Delta{
    {Type: Added, Object: pod-v1},
    {Type: Updated, Object: pod-v2},
    {Type: Updated, Object: pod-v3},
}
// All processed together, handlers get latest state
```

### 2.4 Indexer (Local Cache)

**Purpose**: Thread-safe in-memory cache with indexing capabilities.

**Interface**:
```go
type Indexer interface {
    Store  // Add, Update, Delete, Get, List
    Index(indexName string, obj interface{}) ([]interface{}, error)
    IndexKeys(indexName string, indexKey string) ([]string, error)
    ListIndexFuncValues(indexName string) []string
    ByIndex(indexName string, indexKey string) ([]interface{}, error)
    GetIndexers() Indexers
    AddIndexers(newIndexers Indexers) error
}
```

**Default Index**: By namespace
```go
// MetaNamespaceIndexFunc indexes by namespace
func MetaNamespaceIndexFunc(obj interface{}) ([]string, error) {
    meta, err := meta.Accessor(obj)
    if err != nil {
        return nil, err
    }
    return []string{meta.GetNamespace()}, nil
}

// Usage: Get all pods in namespace "default"
pods, err := informer.GetIndexer().ByIndex("namespace", "default")
```

**Memory Optimization - Transform Function**:
```go
// Trim ManagedFields to reduce memory ~30-50%
trim := func(obj interface{}) (interface{}, error) {
    if accessor, err := meta.Accessor(obj); err == nil {
        accessor.SetManagedFields(nil)
    }
    return obj, nil
}

informers.NewSharedInformerFactoryWithOptions(
    client,
    resyncPeriod,
    informers.WithTransform(trim),
)
```

### 2.5 Event Handler Registration

```mermaid
sequenceDiagram
    participant Controller
    participant Informer
    participant SharedProcessor
    participant ProcessListener

    Controller->>Informer: AddEventHandler(handler)
    Informer->>SharedProcessor: addListener(listener)
    SharedProcessor->>ProcessListener: Create new listener
    ProcessListener->>ProcessListener: Start goroutines (2)

    Note over ProcessListener: Goroutine 1: pop()
    loop Continuous
        ProcessListener->>ProcessListener: Receive notification
        ProcessListener->>ProcessListener: Add to buffer
    end

    Note over ProcessListener: Goroutine 2: run()
    loop Continuous
        ProcessListener->>ProcessListener: Get from buffer
        ProcessListener->>Controller: handler.OnAdd/Update/Delete(obj)
    end
```

**Handler Interface**:
```go
type ResourceEventHandler interface {
    OnAdd(obj interface{})
    OnUpdate(oldObj, newObj interface{})
    OnDelete(obj interface{})
}

// Usage in controller
informer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        controller.handleAdd(obj)
    },
    UpdateFunc: func(oldObj, newObj interface{}) {
        controller.handleUpdate(oldObj, newObj)
    },
    DeleteFunc: func(obj interface{}) {
        controller.handleDelete(obj)
    },
})
```

---

## 3. Work Queue System

### 3.1 Work Queue Architecture

```mermaid
graph TB
    subgraph "RateLimitingQueue"
        INTERFACE[Queue Interface]

        subgraph "Delay Queue"
            WAITING[Waiting Heap<br/>Sorted by ready time]
            READY[Ready Channel]
        end

        subgraph "Rate Limiter"
            LIMITER[Rate Limiter<br/>Exponential Backoff]
            BUCKET[Token Bucket<br/>Overall limit]
        end

        subgraph "Processing"
            DIRTY[Dirty Set<br/>Items needing process]
            PROCESSING[Processing Set<br/>Currently being processed]
        end
    end

    ADD[Add/AddAfter]
    GET[Get]
    DONE[Done]
    REQUEUE[AddRateLimited]

    ADD --> LIMITER
    LIMITER --> WAITING
    WAITING -->|Time elapsed| READY
    READY --> DIRTY
    DIRTY --> GET
    GET --> PROCESSING
    PROCESSING --> DONE
    PROCESSING --> REQUEUE
    REQUEUE --> LIMITER

    style DIRTY fill:#ff9,stroke:#333,stroke-width:2px
    style PROCESSING fill:#f99,stroke:#333,stroke-width:2px
```

### 3.2 Rate Limiting Strategies

**Default Controller Rate Limiter**:
```go
func DefaultControllerRateLimiter() RateLimiter {
    return NewMaxOfRateLimiter(
        // Exponential backoff: 5ms, 10ms, 20ms, ... up to 1000s
        NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),

        // Overall rate limit: 10 qps, burst 100
        &BucketRateLimiter{
            Limiter: rate.NewLimiter(rate.Limit(10), 100),
        },
    )
}
```

**Backoff Calculation**:
```mermaid
graph LR
    ATTEMPT1[Attempt 1<br/>5ms] --> ATTEMPT2[Attempt 2<br/>10ms]
    ATTEMPT2 --> ATTEMPT3[Attempt 3<br/>20ms]
    ATTEMPT3 --> ATTEMPT4[Attempt 4<br/>40ms]
    ATTEMPT4 --> ATTEMPT5[Attempt 5<br/>80ms]
    ATTEMPT5 --> MORE[...]
    MORE --> ATTEMPT15[Attempt 15<br/>~82s]
    ATTEMPT15 --> MAX[Max attempts<br/>Drop item]

    style ATTEMPT1 fill:#9f9
    style ATTEMPT5 fill:#ff9
    style ATTEMPT15 fill:#f99
    style MAX fill:#f66
```

### 3.3 Work Queue Operations

**Adding Items**:
```go
// Simple add
queue.Add("default/my-deployment")

// Add after delay
queue.AddAfter("default/my-deployment", 5*time.Second)

// Add with rate limiting (on error)
queue.AddRateLimited("default/my-deployment")
```

**Processing Pattern**:
```go
func (c *Controller) worker(ctx context.Context) {
    for c.processNextWorkItem(ctx) {
    }
}

func (c *Controller) processNextWorkItem(ctx context.Context) bool {
    key, quit := c.queue.Get()
    if quit {
        return false
    }
    defer c.queue.Done(key)

    err := c.syncHandler(ctx, key.(string))
    if err != nil {
        // Requeue with backoff
        c.queue.AddRateLimited(key)
        return true
    }

    // Success: remove from rate limiter
    c.queue.Forget(key)
    return true
}
```

### 3.4 Queue Metrics

```go
// Workqueue metrics (Prometheus)
workqueue_depth{name="deployment"}                        // Current items
workqueue_adds_total{name="deployment"}                   // Total adds
workqueue_queue_duration_seconds{name="deployment"}       // Time in queue
workqueue_work_duration_seconds{name="deployment"}        // Processing time
workqueue_retries_total{name="deployment"}                // Total retries
workqueue_longest_running_processor_seconds{name="deployment"}
workqueue_unfinished_work_seconds{name="deployment"}
```

---

## 4. Client Builder System

### 4.1 Client Builder Types

```mermaid
classDiagram
    class ControllerClientBuilder {
        <<interface>>
        +Config(name) *rest.Config
        +ConfigOrDie(name) *rest.Config
        +Client(name) clientset.Interface
        +ClientOrDie(name) clientset.Interface
        +DiscoveryClient(name) discovery.Interface
    }

    class SimpleControllerClientBuilder {
        -ClientConfig *rest.Config
        +Config(name) *rest.Config
        +Client(name) clientset.Interface
        +DiscoveryClient(name) discovery.Interface
    }

    class DynamicClientBuilder {
        -ClientConfig *rest.Config
        -CoreClient corev1client.CoreV1Interface
        -Namespace string
    }

    ControllerClientBuilder <|.. SimpleControllerClientBuilder
    ControllerClientBuilder <|.. DynamicClientBuilder
```

**SimpleControllerClientBuilder**:
```go
// All controllers use same credentials from kubeconfig
type SimpleControllerClientBuilder struct {
    ClientConfig *rest.Config
}

func (b SimpleControllerClientBuilder) Client(name string) (clientset.Interface, error) {
    // Just clone the config and create client
    return clientset.NewForConfig(rest.CopyConfig(b.ClientConfig))
}
```

**DynamicClientBuilder**:
```go
// Each controller gets its own ServiceAccount token
type DynamicClientBuilder struct {
    ClientConfig *rest.Config
    CoreClient   corev1client.CoreV1Interface
    Namespace    string
}

func (b DynamicClientBuilder) Client(name string) (clientset.Interface, error) {
    // 1. Get ServiceAccount for this controller
    sa, err := b.CoreClient.ServiceAccounts(b.Namespace).Get(context.TODO(), name, metav1.GetOptions{})

    // 2. Get token from ServiceAccount
    token, err := b.getServiceAccountToken(sa)

    // 3. Create client config with token
    clientConfig := rest.AnonymousClientConfig(b.ClientConfig)
    clientConfig.BearerToken = token

    // 4. Create client
    return clientset.NewForConfig(clientConfig)
}
```

### 4.2 Per-Controller Authentication Flow

```mermaid
sequenceDiagram
    participant Controller
    participant ClientBuilder
    participant API as kube-apiserver
    participant SA as ServiceAccount

    Controller->>ClientBuilder: Client("deployment-controller")
    alt DynamicClientBuilder
        ClientBuilder->>API: GET ServiceAccount "deployment-controller"
        API-->>ClientBuilder: ServiceAccount object
        ClientBuilder->>API: GET Secret (token)
        API-->>ClientBuilder: Token
        ClientBuilder->>ClientBuilder: Create config with token
    else SimpleControllerClientBuilder
        ClientBuilder->>ClientBuilder: Use root kubeconfig
    end
    ClientBuilder-->>Controller: Authenticated client

    Controller->>API: GET Deployments (with token)
    API->>API: Authenticate token
    API->>API: Authorize (RBAC)
    alt Authorized
        API-->>Controller: Deployment list
    else Unauthorized
        API-->>Controller: 403 Forbidden
    end
```

### 4.3 RBAC Configuration Example

```yaml
# ServiceAccount for deployment controller
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployment-controller
  namespace: kube-system

---
# ClusterRole with specific permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:controller:deployment-controller
rules:
# Deployments: full access
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments/status"]
  verbs: ["update", "patch"]
# ReplicaSets: create, manage
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["*"]
# Pods: read-only (for scaling decisions)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
# Events: create
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch", "update"]

---
# Binding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:controller:deployment-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:controller:deployment-controller
subjects:
- kind: ServiceAccount
  name: deployment-controller
  namespace: kube-system
```

---

## 5. Event Broadcasting

### 5.1 Event Broadcasting Architecture

```mermaid
graph TB
    subgraph "Controllers"
        C1[Deployment Controller]
        C2[ReplicaSet Controller]
        C3[StatefulSet Controller]
    end

    subgraph "Event Broadcasting"
        BROADCASTER[EventBroadcaster]
        RECORDER[EventRecorder]

        subgraph "Sinks"
            LOG_SINK[Logging Sink<br/>Structured logging]
            API_SINK[API Sink<br/>Create Event objects]
        end
    end

    subgraph "API Server"
        EVENTS[Events API<br/>/api/v1/events]
    end

    C1 --> RECORDER
    C2 --> RECORDER
    C3 --> RECORDER

    RECORDER --> BROADCASTER
    BROADCASTER --> LOG_SINK
    BROADCASTER --> API_SINK
    API_SINK --> EVENTS

    style BROADCASTER fill:#9f9,stroke:#333,stroke-width:2px
```

### 5.2 Event Creation

**Setup**:
```go
// Create broadcaster
eventBroadcaster := record.NewBroadcaster()

// Start logging (logs 3 most recent events)
eventBroadcaster.StartStructuredLogging(3)

// Start recording to API server
eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{
    Interface: client.CoreV1().Events(""),
})

// Create recorder for this controller
eventRecorder := eventBroadcaster.NewRecorder(
    scheme.Scheme,
    v1.EventSource{Component: "deployment-controller"},
)
```

**Recording Events**:
```go
// Normal event
eventRecorder.Event(
    deployment,                        // Object
    v1.EventTypeNormal,                // Type
    "ScalingReplicaSet",               // Reason
    "Scaled up replica set nginx-5d to 3",  // Message
)

// Warning event
eventRecorder.Event(
    deployment,
    v1.EventTypeWarning,
    "FailedCreate",
    fmt.Sprintf("Error creating: %v", err),
)

// Event with annotations
eventRecorder.AnnotatedEventf(
    deployment,
    map[string]string{"reason": "user-requested"},
    v1.EventTypeNormal,
    "ScalingReplicaSet",
    "Scaled from %d to %d",
    oldReplicas, newReplicas,
)
```

### 5.3 Event Aggregation

**Problem**: Avoid creating duplicate events

**Solution**: Event aggregation
```go
// Similar events within time window are aggregated
Event 1: "Failed to create pod" (count: 1)
Event 2: "Failed to create pod" (count: 1)  // Within 10 minutes
Event 3: "Failed to create pod" (count: 1)  // Within 10 minutes

// Results in single event:
Event: "Failed to create pod" (count: 3)
       firstTimestamp: <Event 1 time>
       lastTimestamp: <Event 3 time>
```

---

## 6. REST Mapper & Discovery

### 6.1 Deferred Discovery REST Mapper

**Purpose**: Maps resource types to API groups without blocking on discovery.

```mermaid
sequenceDiagram
    participant Controller
    participant Mapper as DeferredDiscoveryRESTMapper
    participant Cache as MemCacheClient
    participant Discovery as Discovery Client
    participant API as kube-apiserver

    Controller->>Mapper: RESTMapping(GroupKind)
    Mapper->>Cache: Check cached discovery

    alt Cache valid
        Cache-->>Mapper: Cached API resources
        Mapper-->>Controller: RESTMapping
    else Cache invalid
        Mapper->>Discovery: ServerResources()
        Discovery->>API: GET /api, /apis
        API-->>Discovery: API groups & resources
        Discovery->>Cache: Update cache
        Cache-->>Mapper: Fresh API resources
        Mapper-->>Controller: RESTMapping
    end

    Note over Mapper: Periodic refresh every 30s
    loop Every 30 seconds
        Mapper->>Mapper: Reset()
        Mapper->>Cache: Invalidate()
    end
```

**Usage**:
```go
// Create cached discovery client
cachedClient := cacheddiscovery.NewMemCacheClient(discoveryClient)

// Create REST mapper with caching
restMapper := restmapper.NewDeferredDiscoveryRESTMapper(cachedClient)

// Periodic refresh to discover new CRDs
go wait.Until(func() {
    restMapper.Reset()
}, 30*time.Second, ctx.Done())

// Use mapper
mapping, err := restMapper.RESTMapping(
    schema.GroupKind{Group: "apps", Kind: "Deployment"},
    "v1",
)
// mapping.Resource = "deployments"
// mapping.GroupVersionKind = apps/v1, Kind=Deployment
```

### 6.2 Discovery Caching Benefits

```
Without caching (per request):
- 50 controllers × 10 mappings/controller = 500 discovery calls
- Each discovery call = ~100ms
- Total: 50 seconds of discovery time

With caching:
- Initial discovery: ~100ms
- Subsequent lookups: <1ms from memory
- Refresh: Every 30s in background
- Total impact: negligible
```

---

## 7. Metadata-Only Informers

### 7.1 Purpose

**Problem**: Full object informers consume too much memory for generic controllers (e.g., GarbageCollector).

**Solution**: Metadata-only informers that only cache object metadata.

```mermaid
graph TB
    subgraph "Traditional Informer"
        FULL_OBJ[Full Object Cache<br/>~5-10 KB per object]
        FULL_MEM[Memory: High<br/>10000 pods = 50-100 MB]
    end

    subgraph "Metadata-Only Informer"
        META_OBJ[Metadata Cache<br/>~0.5-1 KB per object]
        META_MEM[Memory: Low<br/>10000 pods = 5-10 MB]
    end

    FULL_OBJ --> FULL_MEM
    META_OBJ --> META_MEM

    style META_MEM fill:#9f9,stroke:#333,stroke-width:2px
```

### 7.2 Metadata Structure

**PartialObjectMetadata**:
```go
type PartialObjectMetadata struct {
    TypeMeta   `json:",inline"`
    ObjectMeta `json:"metadata,omitempty"`
}

// Contains only:
// - Name, Namespace, UID
// - Labels, Annotations
// - OwnerReferences
// - ResourceVersion
// - DeletionTimestamp
// - Finalizers
//
// Does NOT contain:
// - Spec
// - Status
```

### 7.3 Usage Example

```go
// Create metadata informer factory
metadataClient, err := metadata.NewForConfig(config)
metadataInformers := metadatainformer.NewSharedInformerFactory(
    metadataClient,
    resyncPeriod,
)

// Use for generic operations
podInformer := metadataInformers.ForResource(
    schema.GroupVersionResource{
        Group:    "",
        Version:  "v1",
        Resource: "pods",
    },
)

// Handler receives PartialObjectMetadata
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        metaObj := obj.(*metav1.PartialObjectMetadata)
        // Can access: metaObj.Name, metaObj.OwnerReferences, etc.
        // Cannot access: spec, status
    },
})
```

---

## 8. GraphBuilder (GarbageCollector)

### 8.1 Dependency Graph Structure

```mermaid
graph TB
    subgraph "Dependency Graph"
        DEP[Deployment<br/>uid: 1234]
        RS1[ReplicaSet new<br/>uid: 5678<br/>owner: 1234]
        RS2[ReplicaSet old<br/>uid: 9abc<br/>owner: 1234]
        POD1[Pod-1<br/>uid: def0<br/>owner: 5678]
        POD2[Pod-2<br/>uid: 1234<br/>owner: 5678]
        POD3[Pod-3<br/>uid: 5678<br/>owner: 9abc]

        DEP -->|owns| RS1
        DEP -->|owns| RS2
        RS1 -->|owns| POD1
        RS1 -->|owns| POD2
        RS2 -->|owns| POD3
    end

    subgraph "Graph Node Structure"
        NODE[Node]
        OWNERS[Owners Set<br/>UIDs of owners]
        DEPENDENTS[Dependents Set<br/>UIDs of dependents]
        VIRTUAL{Virtual?}

        NODE --> OWNERS
        NODE --> DEPENDENTS
        NODE --> VIRTUAL
    end
```

### 8.2 GraphBuilder Monitors

```go
// GraphBuilder creates monitors for each resource type
type GraphBuilder struct {
    monitors map[schema.GroupVersionResource]*monitor
    // ...
}

type monitor struct {
    store *concurrentUIDStore  // UID -> node mapping
    controller cache.Controller // Informer controller
}

// When resource deleted, GraphBuilder:
// 1. Finds node in graph
// 2. Checks dependents
// 3. Queues dependents for deletion (if not orphan mode)
```

### 8.3 Garbage Collection Flow

```mermaid
sequenceDiagram
    participant User
    participant API
    participant GB as GraphBuilder
    participant GC as GarbageCollector
    participant Graph

    User->>API: DELETE Deployment
    API->>API: Set deletionTimestamp
    API->>API: Add foreground finalizer
    API-->>GB: WATCH: Deployment updated

    GB->>Graph: Find node (uid: 1234)
    GB->>Graph: Get dependents
    Graph-->>GB: [RS-5678, RS-9abc]

    GB->>GC: Enqueue RS-5678
    GB->>GC: Enqueue RS-9abc

    GC->>API: DELETE ReplicaSet-5678
    API->>API: Set deletionTimestamp
    API->>API: Add foreground finalizer
    API-->>GB: WATCH: RS updated

    GB->>Graph: Get dependents of RS-5678
    Graph-->>GB: [Pod-def0, Pod-1234]
    GB->>GC: Enqueue pods

    GC->>API: DELETE Pod-def0
    GC->>API: DELETE Pod-1234

    loop Wait for dependents
        GC->>API: Check RS-5678 dependents
        alt All pods deleted
            GC->>API: Remove finalizer from RS-5678
            API->>API: Delete RS-5678
        end
    end

    loop Wait for RS deletion
        GC->>API: Check Deployment dependents
        alt All RS deleted
            GC->>API: Remove finalizer from Deployment
            API->>API: Delete Deployment
        end
    end
```

---

## 9. Performance Optimizations

### 9.1 Memory Optimizations

| Technique | Memory Saved | Use Case |
|-----------|--------------|----------|
| Transform func (trim ManagedFields) | 30-50% | All informers |
| Metadata-only informers | 80-90% | GarbageCollector, generic controllers |
| Selector filtering | 50-90% | Namespace-scoped controllers |
| Resource-specific informers | Variable | Single-namespace controllers |

### 9.2 CPU Optimizations

| Technique | Benefit | Implementation |
|-----------|---------|----------------|
| Work queue deduplication | Avoid redundant reconciliations | Automatic in work queue |
| Parallel workers | Process multiple items concurrently | Configurable per controller |
| Rate limiting | Prevent API server overload | Exponential backoff |
| Batch updates | Reduce API calls | Event batching in endpoints controller |

### 9.3 Network Optimizations

| Technique | Benefit |
|-----------|---------|
| Shared informers | 1 watch per resource type vs N watches |
| Local cache reads | 0 API calls for cached data |
| Watch bookmarks | Efficient watch resume after disconnect |
| Compression | Reduced bandwidth (disabled by default in controller-manager) |

---

## 10. Configuration & Tuning

### 10.1 Informer Configuration

```yaml
# Resync period (default: 12h)
--min-resync-period=12h0m0s

# Affects how often cache is fully resynced
# Actual resync per informer: min-resync-period * random(1.0, 2.0)
```

### 10.2 Work Queue Configuration

```go
// Per-controller concurrency
--concurrent-deployment-syncs=5      // 5 workers
--concurrent-replicaset-syncs=5
--concurrent-statefulset-syncs=5
--concurrent-daemonset-syncs=2       // DaemonSet uses less

// Rate limits (built into queue, not configurable)
// Exponential backoff: 5ms to 1000s
// Overall rate: 10 QPS, burst 100
```

### 10.3 Client Configuration

```go
// From kubeconfig/flags
--kube-api-qps=20               // Queries per second
--kube-api-burst=30             // Burst allowance

// Specific controllers may multiply this
// e.g., GC uses 2x QPS for delete operations
```

---

## Related Documentation

- **High-Level Architecture**: `05-high-level-architecture.md`
- **Initialization Flow**: `06-initialization-lifecycle.md`
- **Controller Patterns**: `16-controller-patterns.md`
- **Concurrency**: `18-concurrency-synchronization.md`

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-10-22 | Architecture Analysis | Initial shared infrastructure documentation |
