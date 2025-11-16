# Watch Implementation in Kubernetes-etcd Integration

**Version**: 1.0
**Last Updated**: 2025-11-05
**Related Documents**: [../high-level/04-watch-mechanism.md](../high-level/04-watch-mechanism.md), [../high-level/03-data-model.md](../high-level/03-data-model.md), [01-storage-backend.md](01-storage-backend.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Table of Contents**

- [Introduction](#introduction)
- [Data Structures](#data-structures)
- [Core Components](#core-components)
- [Component Interactions](#component-interactions)
- [Synchronization Patterns](#synchronization-patterns)
- [Aspect-Oriented Concerns](#aspect-oriented-concerns)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Summary](#summary)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Introduction**

This document provides a comprehensive examination of the watch implementation in Kubernetes' etcd integration. The watch mechanism is one of Kubernetes' most critical features, enabling real-time event notification when resources change in the cluster.

### **Why Watch Matters**

Watch is fundamental to Kubernetes' control loop architecture:

```mermaid
graph LR
    A[Controllers] -->|Watch| B[API Server]
    B -->|Watch| C[Watch Cache]
    C -->|Watch| D[etcd]

    D -->|Events| C
    C -->|Events| B
    B -->|Events| A

    style A fill:#FFE5B4
    style B fill:#E1F5FF
    style C fill:#D4E6F1
    style D fill:#90EE90
```

**Key Benefits**:
- **Real-time updates**: Controllers react immediately to changes
- **Reduced load**: No polling required
- **Efficiency**: Only changed objects are transmitted
- **Ordering guarantees**: Events arrive in order
- **Resume capability**: Watch can resume from a ResourceVersion

### **Document Scope**

This document covers:

1. **Data structures** for watch events, watchers, and caches
2. **Core components**: etcd3 watcher, watch cache (cacher), Reflector
3. **Component interactions** in the full watch pipeline
4. **Synchronization patterns** for watch resume, event transformation, concurrency
5. **Aspect-oriented concerns**: observability, error handling, performance, reliability

**Code Locations**:
- `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go` - etcd3 watch client
- `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go` - Watch cache
- `staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go` - Cache implementation
- `staging/src/k8s.io/client-go/tools/cache/reflector.go` - Client-side reflector

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Data Structures**

### **etcd3 Internal Event**

The internal event structure used by the etcd3 watcher:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/event.go:25
type event struct {
    key              string   // etcd key: "/registry/pods/default/mypod"
    value            []byte   // Current encoded object value
    prevValue        []byte   // Previous encoded object value (for updates)
    rev              int64    // etcd ModRevision
    isDeleted        bool     // True for DELETE events
    isCreated        bool     // True for CREATE events
    isProgressNotify bool     // True for bookmark events
    isInitialEventsEndBookmark bool // Special bookmark after initial list
}
```

**Field Mapping**:

| Field | etcd Source | Purpose | Example |
|-------|-------------|---------|---------|
| `key` | `Event.Kv.Key` | Object key | `/registry/pods/default/nginx` |
| `value` | `Event.Kv.Value` | Current object | Protobuf-encoded Pod |
| `prevValue` | `Event.PrevKv.Value` | Previous object | For detecting changes |
| `rev` | `Event.Kv.ModRevision` | Revision number | `12345` |
| `isDeleted` | `Event.Type == EventTypeDelete` | Deletion indicator | `true/false` |
| `isCreated` | `Event.IsCreate()` | Creation indicator | `true/false` |

**Event Creation from etcd**:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/event.go:58
func parseEvent(e *clientv3.Event) (*event, error) {
    if !e.IsCreate() && e.PrevKv == nil {
        // Previous value required for proper change detection
        return nil, fmt.Errorf("etcd event received with PrevKv=nil (key=%q, modRevision=%d, type=%s)",
            string(e.Kv.Key), e.Kv.ModRevision, e.Type.String())
    }

    ret := &event{
        key:       string(e.Kv.Key),
        value:     e.Kv.Value,
        rev:       e.Kv.ModRevision,  // Maps to ResourceVersion
        isDeleted: e.Type == clientv3.EventTypeDelete,
        isCreated: e.IsCreate(),
    }

    if e.PrevKv != nil {
        ret.prevValue = e.PrevKv.Value
    }

    return ret, nil
}
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/event.go:58`

### **etcd3 watcher Structure**

The watcher manages watch operations to etcd:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:71
type watcher struct {
    client                   *clientv3.Client          // etcd client connection
    codec                    runtime.Codec             // Object encoder/decoder
    newFunc                  func() runtime.Object     // Factory for new objects
    objectType               string                    // Type name for logging
    groupResource            schema.GroupResource      // Resource being watched
    versioner                storage.Versioner         // ResourceVersion manager
    transformer              value.Transformer         // Encryption transformer
    getCurrentStorageRV      func(context.Context) (uint64, error)  // Get current RV
    getResourceSizeEstimator func() *resourceSizeEstimator
}
```

**Key Responsibilities**:
- Create watch channels to etcd
- Transform etcd events to Kubernetes watch.Event
- Handle watch lifecycle (creation, cancellation, errors)
- Manage ResourceVersion semantics

### **watchChan Structure**

Individual watch channel representing a single watch request:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:84
type watchChan struct {
    watcher                  *watcher                  // Parent watcher
    key                      string                    // Watch key or prefix
    initialRev               int64                     // Starting revision
    recursive                bool                      // Watch children too?
    progressNotify           bool                      // Send bookmarks?
    internalPred             storage.SelectionPredicate // Filter predicate
    ctx                      context.Context           // Cancellation context
    cancel                   context.CancelFunc        // Cancel function
    incomingEventChan        chan *event               // Events from etcd
    resultChan               chan watch.Event          // Events to client
    getResourceSizeEstimator func() *resourceSizeEstimator
}
```

**Buffer Sizes**:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:47
const (
    incomingBufSize         = 100   // Buffer for etcd events
    outgoingBufSize         = 100   // Buffer for client events
    processEventConcurrency = 10    // Parallel event processing
)
```

**Watch Flow Diagram**:

```mermaid
graph TB
    A[etcd Watch Stream] -->|etcd Event| B[incomingEventChan]
    B -->|Process & Transform| C[transform thread]
    C -->|Kubernetes Event| D[resultChan]
    D -->|deliver| E[Client]

    F[context.Context] -.->|cancel| C

    style A fill:#90EE90
    style B fill:#FFE5B4
    style C fill:#E1F5FF
    style D fill:#FFE5B4
    style E fill:#FFD700
```

### **Kubernetes watch.Event**

The final event structure delivered to clients:

```go
// k8s.io/apimachinery/pkg/watch/watch.go
type Event struct {
    Type   EventType      // Added, Modified, Deleted, Bookmark, Error
    Object runtime.Object // The Kubernetes object
}

type EventType string

const (
    Added    EventType = "ADDED"      // Object created
    Modified EventType = "MODIFIED"   // Object updated
    Deleted  EventType = "DELETED"    // Object deleted
    Bookmark EventType = "BOOKMARK"   // Progress notification
    Error    EventType = "ERROR"      // Watch error occurred
)
```

**Event Type Mapping**:

| etcd Event | isCreated | isDeleted | Kubernetes EventType |
|------------|-----------|-----------|---------------------|
| PUT (new) | true | false | `Added` |
| PUT (update) | false | false | `Modified` |
| DELETE | false | true | `Deleted` |
| Progress Notify | N/A | N/A | `Bookmark` |

### **watchCacheEvent Structure**

Enhanced event for the watch cache layer:

```go
// staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go:66
type watchCacheEvent struct {
    Type            watch.EventType   // Event type
    Object          runtime.Object    // Current object
    ObjLabels       labels.Set        // Current object labels
    ObjFields       fields.Set        // Current object fields
    PrevObject      runtime.Object    // Previous object state
    PrevObjLabels   labels.Set        // Previous labels
    PrevObjFields   fields.Set        // Previous fields
    Key             string            // Object key
    ResourceVersion uint64            // Kubernetes ResourceVersion
    RecordTime      time.Time         // Event timestamp
}
```

**Why PrevObject?**

The watch cache stores both current and previous objects to enable:
- **Label/field selector filtering** on transitions
- **Detecting what changed** without decoding
- **Synthetic event generation** when filters match differently

**Example**:
```go
// Pod updated: labels changed from {app: v1} to {app: v2}
// Watch with labelSelector "app=v2"
// PrevObject: {labels: {app: v1}}  -> doesn't match -> ignore
// Object:     {labels: {app: v2}}  -> matches -> send ADDED event
```

### **cacheWatcher Structure**

Represents a single watcher connected to the watch cache:

```go
// staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go (inferred from code)
type cacheWatcher struct {
    sync.Mutex
    input     chan *watchCacheEvent   // Events from cache
    result    chan watch.Event        // Events to client
    done      chan struct{}           // Signals watcher stopped
    filter    filterWithAttrsFunc     // Label/field filter
    stopped   bool                    // Stop flag
    forget    func()                  // Cleanup callback
    versioner storage.Versioner       // ResourceVersion handler

    // Bookmark support
    deadline            time.Time     // Next bookmark time
    allowWatchBookmarks bool          // Client accepts bookmarks?
}
```

**Watcher Lifecycle**:

```mermaid
stateDiagram-v2
    [*] --> Created: Cacher.Watch()
    Created --> Running: Add to watchers map
    Running --> Buffering: Events arrive
    Buffering --> Delivering: nonblockingAdd()
    Delivering --> Running: Event sent
    Running --> Stopped: Stop() or timeout
    Stopped --> [*]: Remove from map

    Running --> Blocked: Client slow
    Blocked --> Stopped: Timeout
```

### **watchCache Structure**

The sliding window cache storing recent events:

```go
// staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go:88
type watchCache struct {
    sync.RWMutex

    // Sliding window buffer
    cache      []*watchCacheEvent  // Circular buffer
    startIndex int                 // Start of valid events
    endIndex   int                 // End of valid events (exclusive)
    capacity   int                 // Current buffer size

    // State
    store           storeIndexer     // Current object state
    resourceVersion uint64           // Current RV
    listResourceVersion uint64       // RV of last full list

    // Configuration
    keyFunc          func(runtime.Object) (string, error)
    getAttrsFunc     func(runtime.Object) (labels.Set, fields.Set, error)
    versioner        storage.Versioner
    eventFreshDuration time.Duration  // How long to keep events

    // Coordination
    cond              *sync.Cond      // Wait for RV updates
    eventHandler      func(*watchCacheEvent)
    onReplace         func()

    // Metrics
    groupResource     schema.GroupResource
    clock             clock.Clock
}
```

**Circular Buffer Visualization**:

```
Initial state (capacity=8):
[nil, nil, nil, nil, nil, nil, nil, nil]
 ^startIndex=0, endIndex=0

After 3 events:
[E1, E2, E3, nil, nil, nil, nil, nil]
 ^startIndex=0      ^endIndex=3

After 10 events (wrapped):
[E9, E10, E3, E4, E5, E6, E7, E8]
          ^startIndex=2      ^endIndex=10%8=2

Current events: E3, E4, E5, E6, E7, E8, E9, E10
```

**Capacity Management**:

```go
// staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go:58
const (
    defaultLowerBoundCapacity = 100       // Minimum events
    defaultUpperBoundCapacity = 100 * 1024 // Maximum events (100K)
)

// Capacity grows with event freshness duration
func capacityUpperBound(eventFreshDuration time.Duration) int {
    if eventFreshDuration <= DefaultEventFreshDuration {
        return defaultUpperBoundCapacity
    }
    // Scale capacity for longer retention periods
    exponent := int(math.Ceil(math.Log2(
        eventFreshDuration.Seconds() / DefaultEventFreshDuration.Seconds()
    )))
    return defaultUpperBoundCapacity << exponent
}
```

### **Reflector Structure**

Client-side component implementing the List+Watch pattern:

```go
// staging/src/k8s.io/client-go/tools/cache/reflector.go:87
type Reflector struct {
    name            string                    // Reflector name
    typeDescription string                    // Type being watched
    expectedType    reflect.Type              // Expected object type
    store           ReflectorStore            // Local cache (e.g., DeltaFIFO)
    listerWatcher   ListerWatcherWithContext  // API operations

    // ResourceVersion tracking
    lastSyncResourceVersion string            // Last observed RV
    isLastSyncResourceVersionUnavailable bool // RV expired?
    lastSyncResourceVersionMutex sync.RWMutex

    // Configuration
    resyncPeriod    time.Duration             // Resync interval
    minWatchTimeout time.Duration             // Watch timeout (default 5m)
    clock           clock.Clock               // Time source
    backoffManager  wait.BackoffManager       // Retry backoff

    // Callbacks
    watchErrorHandler WatchErrorHandlerWithContext
    ShouldResync      func() bool

    // WatchList optimization
    useWatchList       bool                   // Use streaming watch
    WatchListPageSize  int64                  // Chunk size
}
```

**Reflector Flow**:

```mermaid
graph TB
    A[Start] --> B[List]
    B -->|Success| C[Store initial objects]
    C --> D[Start Watch from RV]
    D --> E{Watch Event?}

    E -->|Added| F[Store.Add]
    E -->|Modified| G[Store.Update]
    E -->|Deleted| H[Store.Delete]
    E -->|Bookmark| I[Update RV]
    E -->|Error| J{Recoverable?}

    F --> E
    G --> E
    H --> E
    I --> E

    J -->|Yes| K[Backoff]
    J -->|No| L[Fatal Error]
    K --> B

    style B fill:#E1F5FF
    style D fill:#90EE90
    style J fill:#FFB6C1
```

**Code Reference**: `staging/src/k8s.io/client-go/tools/cache/reflector.go:87`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚙️ Core Components**

### **etcd3 Watch Client**

The etcd3 watcher provides the foundation for watch operations.

#### **Watch Creation**

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:105
func (w *watcher) Watch(ctx context.Context, key string, rev int64, opts storage.ListOptions) (watch.Interface, error) {
    // Validate recursive watch has trailing slash
    if opts.Recursive && !strings.HasSuffix(key, "/") {
        return nil, fmt.Errorf(`recursive key needs to end with "/"`)
    }

    // Validate progress notify support
    if opts.ProgressNotify && w.newFunc == nil {
        return nil, apierrors.NewInternalError(
            errors.New("progressNotify for watch is unsupported by the etcd storage because no newFunc was provided"))
    }

    // Determine starting ResourceVersion
    startWatchRV, err := w.getStartWatchResourceVersion(ctx, rev, opts)
    if err != nil {
        return nil, err
    }

    // Create watch channel
    wc := w.createWatchChan(ctx, key, startWatchRV, opts.Recursive, opts.ProgressNotify, opts.Predicate)

    // Start watching in background
    go wc.run(isInitialEventsEndBookmarkRequired(opts), areInitialEventsRequired(rev, opts))

    // Signal watch initialized
    utilflowcontrol.WatchInitialized(ctx)

    return wc, nil
}
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:105`

**Starting ResourceVersion Logic**:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:154
func (w *watcher) getStartWatchResourceVersion(ctx context.Context, resourceVersion int64, opts storage.ListOptions) (int64, error) {
    if resourceVersion > 0 {
        // Explicit RV: start from that point
        return resourceVersion, nil
    }

    if !utilfeature.DefaultFeatureGate.Enabled(features.WatchList) {
        // Legacy: start from beginning
        return 0, nil
    }

    if opts.SendInitialEvents == nil || *opts.SendInitialEvents {
        // Need initial events: do list first
        return 0, nil
    }

    // Start from "now" - get current storage RV
    currentStorageRV, err := w.getCurrentStorageRV(ctx)
    if err != nil {
        return 0, err
    }

    return int64(currentStorageRV), nil
}
```

**Watch Semantics**:

| ResourceVersion | SendInitialEvents | Behavior |
|-----------------|-------------------|----------|
| `0` or `""` | `nil` or `true` | List current objects, then watch |
| `0` or `""` | `false` | Watch from "now" (current RV) |
| `>0` | Any | Watch from specified RV |

#### **Watch Execution**

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:230
func (wc *watchChan) run(initialEventsEndBookmarkRequired, forceInitialEvents bool) {
    watchClosedCh := make(chan struct{})
    var resultChanWG sync.WaitGroup

    // Start watching etcd
    resultChanWG.Add(1)
    go func() {
        defer resultChanWG.Done()
        wc.startWatching(watchClosedCh, initialEventsEndBookmarkRequired, forceInitialEvents)
    }()

    // Process and transform events
    wc.processEvents(&resultChanWG)

    // Wait for watch to close
    select {
    case <-watchClosedCh:
    case <-wc.ctx.Done():
    }

    // Cleanup
    wc.cancel()
    resultChanWG.Wait()
    close(wc.resultChan)
}
```

**Two-Phase Processing**:

```mermaid
sequenceDiagram
    participant Run
    participant StartWatch
    participant ProcessEvents
    participant etcd
    participant Client

    Run->>StartWatch: go startWatching()
    Run->>ProcessEvents: processEvents()

    par Watch Phase
        StartWatch->>etcd: Watch(key, revision)
        loop Event Loop
            etcd->>StartWatch: WatchResponse
            StartWatch->>StartWatch: parseEvent()
            StartWatch->>ProcessEvents: queueEvent()
        end
    and Process Phase
        loop Event Loop
            ProcessEvents->>ProcessEvents: <-incomingEventChan
            ProcessEvents->>ProcessEvents: transform()
            ProcessEvents->>Client: resultChan<-event
        end
    end
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:230`

#### **Initial Sync (List Before Watch)**

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:270
func (wc *watchChan) sync() error {
    opts := []clientv3.OpOption{}

    if wc.recursive {
        opts = append(opts, clientv3.WithLimit(defaultWatcherMaxLimit))
        rangeEnd := clientv3.GetPrefixRangeEnd(wc.key)
        opts = append(opts, clientv3.WithRange(rangeEnd))
    }

    var lastKey []byte
    var withRev int64

    for {
        startTime := time.Now()
        getResp, err := wc.watcher.client.KV.Get(wc.ctx, wc.key, opts...)
        metrics.RecordEtcdRequest("list", wc.watcher.groupResource, err, startTime)

        if err != nil {
            return interpretListError(err, true, wc.key, wc.key)
        }

        // Send all items as "created" events
        for i, kv := range getResp.Kvs {
            lastKey = kv.Key
            wc.queueEvent(parseKV(kv))
            getResp.Kvs[i] = nil  // Free memory early
        }

        // Set initial revision from list
        if withRev == 0 {
            wc.initialRev = getResp.Header.Revision
        }

        // Done if no more results
        if !getResp.More {
            return nil
        }

        // Continue paginated list
        wc.key = string(lastKey) + "\x00"
        if withRev == 0 {
            withRev = getResp.Header.Revision
            opts = append(opts, clientv3.WithRev(withRev))
        }
    }
}
```

**Pagination Strategy**:

```
Page 1: List keys from "a" to "z" at revision=100
  [a1, a2, a3, ..., m5] (limit reached, More=true)

Page 2: List keys from "m5\x00" at revision=100
  [m6, m7, ..., z9] (More=false)

All pages use same revision (100) for consistency
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:270`

#### **Event Streaming**

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:354
func (wc *watchChan) startWatching(watchClosedCh chan struct{}, initialEventsEndBookmarkRequired, forceInitialEvents bool) {
    // Sync (list) if needed
    if forceInitialEvents {
        if err := wc.sync(); err != nil {
            wc.sendError(err)
            return
        }
    }

    // Send initial events end bookmark if needed
    if initialEventsEndBookmarkRequired {
        wc.queueEvent(func() *event {
            e := progressNotifyEvent(wc.initialRev)
            e.isInitialEventsEndBookmark = true
            return e
        }())
    }

    // Start etcd watch
    opts := []clientv3.OpOption{
        clientv3.WithRev(wc.initialRev + 1),  // Watch from next revision
        clientv3.WithPrevKV(),                 // Include previous values
    }
    if wc.recursive {
        opts = append(opts, clientv3.WithPrefix())
    }
    if wc.progressNotify {
        opts = append(opts, clientv3.WithProgressNotify())
    }

    wch := wc.watcher.client.Watch(wc.ctx, wc.key, opts...)
    estimator := wc.getResourceSizeEstimator()

    // Process watch responses
    for wres := range wch {
        if wres.Err() != nil {
            logWatchChannelErr(wres.Err())
            wc.sendError(wres.Err())
            return
        }

        if wres.IsProgressNotify() {
            // Bookmark event
            wc.queueEvent(progressNotifyEvent(wres.Header.GetRevision()))
            metrics.RecordEtcdBookmark(wc.watcher.groupResource)
            continue
        }

        // Regular events
        for _, e := range wres.Events {
            if estimator != nil {
                switch e.Type {
                case clientv3.EventTypePut:
                    estimator.UpdateKey(e.Kv)
                case clientv3.EventTypeDelete:
                    estimator.DeleteKey(e.Kv)
                }
            }

            metrics.RecordEtcdEvent(wc.watcher.groupResource)

            parsedEvent, err := parseEvent(e)
            if err != nil {
                logWatchChannelErr(err)
                wc.sendError(err)
                return
            }

            wc.queueEvent(parsedEvent)
        }
    }

    close(watchClosedCh)
}
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:354`

#### **Event Transformation**

The core transformation from etcd events to Kubernetes events:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:571
func (wc *watchChan) transform(e *event) (res *watch.Event, err error) {
    curObj, oldObj, err := wc.prepareObjs(e)
    if err != nil {
        return nil, err
    }

    switch {
    case e.isProgressNotify:
        // Bookmark event
        object := wc.watcher.newFunc()
        if err := wc.watcher.versioner.UpdateObject(object, uint64(e.rev)); err != nil {
            return nil, fmt.Errorf("failed to propagate object resource version: %w", err)
        }
        if e.isInitialEventsEndBookmark {
            if err := storage.AnnotateInitialEventsEndBookmark(object); err != nil {
                return nil, fmt.Errorf("error while accessing object's metadata: %w", err)
            }
        }
        res = &watch.Event{
            Type:   watch.Bookmark,
            Object: object,
        }

    case e.isDeleted:
        if !wc.filter(oldObj) {
            return nil, nil  // Filtered out
        }
        res = &watch.Event{
            Type:   watch.Deleted,
            Object: oldObj,
        }

    case e.isCreated:
        if !wc.filter(curObj) {
            return nil, nil  // Filtered out
        }
        res = &watch.Event{
            Type:   watch.Added,
            Object: curObj,
        }

    default:
        // Update: check if filtering changed
        if wc.acceptAll() {
            res = &watch.Event{
                Type:   watch.Modified,
                Object: curObj,
            }
            return res, nil
        }

        curObjPasses := wc.filter(curObj)
        oldObjPasses := wc.filter(oldObj)

        switch {
        case curObjPasses && oldObjPasses:
            // Still matches: Modified
            res = &watch.Event{
                Type:   watch.Modified,
                Object: curObj,
            }
        case curObjPasses && !oldObjPasses:
            // Now matches: Added
            res = &watch.Event{
                Type:   watch.Added,
                Object: curObj,
            }
        case !curObjPasses && oldObjPasses:
            // No longer matches: Deleted
            res = &watch.Event{
                Type:   watch.Deleted,
                Object: oldObj,
            }
        // case !curObjPasses && !oldObjPasses:
        //     Neither matches: filtered out (res=nil)
        }
    }

    return res, nil
}
```

**Filter Transition Example**:

```yaml
# Watch with selector: status.phase=Running

# Event 1: Pod created with phase=Pending
etcd: PUT, isCreated=true, value={phase: Pending}
filter(curObj): false
result: nil (filtered out)

# Event 2: Pod updated to phase=Running
etcd: PUT, isCreated=false, value={phase: Running}, prevValue={phase: Pending}
filter(oldObj): false
filter(curObj): true
result: watch.Event{Type: Added, Object: {phase: Running}}

# Event 3: Pod updated, still Running
etcd: PUT, isCreated=false, value={phase: Running, ...}, prevValue={phase: Running, ...}
filter(oldObj): true
filter(curObj): true
result: watch.Event{Type: Modified, Object: {phase: Running}}

# Event 4: Pod completed
etcd: PUT, isCreated=false, value={phase: Succeeded}, prevValue={phase: Running}
filter(oldObj): true
filter(curObj): false
result: watch.Event{Type: Deleted, Object: {phase: Running}}
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:571`

### **Watch Cache (Cacher)**

The watch cache is a critical optimization that:
1. Reduces load on etcd by serving watches from memory
2. Maintains a sliding window of recent events
3. Supports multiple concurrent watchers efficiently

#### **Cacher Architecture**

```mermaid
graph TB
    subgraph "Watch Cache Layer"
        A[Cacher] --> B[watchCache]
        A --> C[Reflector]
        A --> D[cacheWatcher 1]
        A --> E[cacheWatcher 2]
        A --> F[cacheWatcher N]

        C -->|LIST+WATCH| G[etcd3 Storage]
        C -->|Updates| B

        B -->|Events| H[dispatchEvent]
        H --> D
        H --> E
        H --> F
    end

    style A fill:#E1F5FF
    style B fill:#D4E6F1
    style C fill:#FFE5B4
    style G fill:#90EE90
```

#### **Event Dispatching**

```go
// staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go:952
func (c *Cacher) dispatchEvent(event *watchCacheEvent) {
    c.startDispatching(event)
    defer c.finishDispatching()

    if event.Type == watch.Bookmark {
        // Bookmarks can be dropped, send non-blocking
        for _, watcher := range c.watchersBuffer {
            watcher.nonblockingAdd(event)
        }
    } else {
        // Make shallow copy for caching serializations
        wcEvent := *event
        setCachingObjects(&wcEvent, c.versioner)
        event = &wcEvent

        c.blockedWatchers = c.blockedWatchers[:0]

        // Try non-blocking send first
        for _, watcher := range c.watchersBuffer {
            if !watcher.nonblockingAdd(event) {
                c.blockedWatchers = append(c.blockedWatchers, watcher)
            }
        }

        // Handle blocked watchers with timeout
        if len(c.blockedWatchers) > 0 {
            timeout := c.dispatchTimeoutBudget.takeAvailable()
            c.timer.Reset(timeout)

            for _, watcher := range c.blockedWatchers {
                if !watcher.add(event, c.timer) {
                    // Timer fired, watcher too slow - close it
                }
            }

            if !c.timer.Stop() {
                <-c.timer.C  // Drain timer
            }
        }
    }
}
```

**Dispatch Flow**:

```mermaid
graph TB
    A[Event arrives] --> B{Bookmark?}
    B -->|Yes| C[nonblockingAdd to all]
    B -->|No| D[Try nonblockingAdd]

    D --> E{All accepted?}
    E -->|Yes| F[Done]
    E -->|No| G[Start timeout timer]

    G --> H[Blocking add to slow watchers]
    H --> I{Timer expired?}
    I -->|Yes| J[Close slow watcher]
    I -->|No| K[Event delivered]

    J --> F
    K --> F

    style J fill:#FFB6C1
    style F fill:#90EE90
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go:952`

### **Reflector (Client-Side)**

The Reflector implements the List+Watch pattern for client-side caching.

#### **ListAndWatch Implementation**

```go
// staging/src/k8s.io/client-go/tools/cache/reflector.go:397 (conceptual)
func (r *Reflector) ListAndWatchWithContext(ctx context.Context) error {
    var resourceVersion string

    // Phase 1: List
    options := metav1.ListOptions{ResourceVersion: r.relistResourceVersion()}

    list, err := r.listerWatcher.List(options)
    if err != nil {
        return fmt.Errorf("failed to list %v: %v", r.typeDescription, err)
    }

    // Extract resource version from list
    listMetaInterface, err := meta.ListAccessor(list)
    if err != nil {
        return fmt.Errorf("unable to understand list result %#v: %v", list, err)
    }
    resourceVersion = listMetaInterface.GetResourceVersion()

    // Sync items to store
    items, err := meta.ExtractList(list)
    if err != nil {
        return fmt.Errorf("unable to understand list result %#v (%v)", list, err)
    }

    if err := r.syncWith(items, resourceVersion); err != nil {
        return fmt.Errorf("unable to sync list result: %v", err)
    }

    r.setLastSyncResourceVersion(resourceVersion)

    // Phase 2: Watch
    for {
        options = metav1.ListOptions{
            ResourceVersion: resourceVersion,
            AllowWatchBookmarks: true,
        }

        w, err := r.listerWatcher.Watch(options)
        if err != nil {
            return err
        }

        if err := r.watchHandler(ctx, w, &resourceVersion); err != nil {
            if err != errorResyncRequested && err != errorStopRequested {
                return err
            }
        }

        if err == errorResyncRequested {
            // Resync: go back to List phase
            continue
        }

        // Stop requested
        return nil
    }
}
```

The Reflector periodically resyncs (re-lists) to:
- Detect missed events due to watch errors
- Verify cache consistency
- Trigger reconciliation in controllers

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Component Interactions**

### **Full Watch Pipeline**

The complete flow from API request to event delivery:

```mermaid
sequenceDiagram
    participant Client
    participant APIServer
    participant StorageInterface
    participant Cacher
    participant WatchCache
    participant Reflector
    participant etcd3Watcher
    participant etcdCluster

    Note over Client,etcdCluster: Phase 1: Watch Initiation

    Client->>APIServer: GET /api/v1/pods?watch=true&resourceVersion=1000
    APIServer->>StorageInterface: Watch(ctx, "/registry/pods/", rv=1000)
    StorageInterface->>Cacher: Watch(ctx, key, opts)

    Cacher->>WatchCache: EventsAfter(1000)
    WatchCache-->>Cacher: Event iterator

    Cacher->>Cacher: Create cacheWatcher
    Cacher-->>APIServer: watch.Interface
    APIServer-->>Client: HTTP stream opened

    Note over Client,etcdCluster: Phase 2: Historical Events

    loop Buffered events (1001-1050)
        Cacher->>Client: Watch event
    end

    Note over Client,etcdCluster: Phase 3: Real-time Events

    Reflector->>etcd3Watcher: Watch(key, rv=1051)
    etcd3Watcher->>etcdCluster: clientv3.Watch()

    loop Ongoing
        etcdCluster->>etcd3Watcher: WatchResponse
        etcd3Watcher->>etcd3Watcher: parseEvent()
        etcd3Watcher->>etcd3Watcher: transform()
        etcd3Watcher->>Reflector: watch.Event
        Reflector->>WatchCache: Add/Update/Delete
        WatchCache->>Cacher: eventHandler()
        Cacher->>Cacher: dispatchEvent()
        Cacher->>Client: Watch event
    end
```

### **Watch Establishment from Scratch**

Starting a watch with no ResourceVersion (or RV="0"):

```mermaid
sequenceDiagram
    participant Client
    participant Watcher
    participant etcd

    Client->>Watcher: Watch(key="/registry/pods/", rv=0)

    Note over Watcher: Need to send current state first

    Watcher->>etcd: Get(key, WithPrefix(), WithLimit())
    etcd-->>Watcher: Response (pods 1-10000, More=true, Revision=5000)

    loop Send current objects
        Watcher->>Client: Event{Type: Added, Object: pod}
    end

    Watcher->>etcd: Get(key=lastKey+"\x00", WithRev(5000))
    etcd-->>Watcher: Response (pods 10001-15000, More=false)

    loop Send remaining objects
        Watcher->>Client: Event{Type: Added, Object: pod}
    end

    Note over Watcher: All current objects sent, now watch changes

    Watcher->>etcd: Watch(key, WithRev(5001), WithPrefix(), WithPrevKV())

    loop Stream events
        etcd->>Watcher: WatchResponse{Events: [...]}
        Watcher->>Client: Watch events
    end
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:270` (sync method)

### **Watch Resume from ResourceVersion**

Resuming a watch from a specific ResourceVersion:

```mermaid
sequenceDiagram
    participant Client
    participant Cacher
    participant WatchCache
    participant etcd3Watcher
    participant etcd

    Client->>Cacher: Watch(rv=1234)

    Cacher->>WatchCache: EventsAfter(1234)

    alt Events in cache
        WatchCache-->>Cacher: Events 1235-1250
        loop Buffered events
            Cacher->>Client: Cached event
        end
        Note over Cacher: Now watching from rv=1250
    else Events too old (compacted)
        WatchCache-->>Cacher: Error: resource version too old
        Cacher-->>Client: Error (410 Gone)
        Note over Client: Must re-list and start over
    end

    opt Watch continues
        etcd3Watcher->>etcd: Watch(WithRev(1251))
        loop New events
            etcd->>etcd3Watcher: Event
            etcd3Watcher->>WatchCache: Update cache
            WatchCache->>Cacher: Dispatch
            Cacher->>Client: Event
        end
    end
```

### **Watch with Bookmarks**

Bookmark events keep ResourceVersion current without data changes:

```mermaid
sequenceDiagram
    participant Client
    participant Watcher
    participant etcd

    Client->>Watcher: Watch(allowWatchBookmarks=true)
    Watcher->>etcd: Watch(WithProgressNotify())

    loop Watch stream
        alt Data event
            etcd->>Watcher: WatchResponse{Events: [PUT pod-1]}
            Watcher->>Client: Event{Type: Modified, RV: 1001}
        else Progress notification
            etcd->>Watcher: WatchResponse{IsProgressNotify: true, Revision: 1005}
            Watcher->>Client: Event{Type: Bookmark, RV: 1005}
            Note over Client: No data change, but RV updated
        end
    end

    Note over Client,etcd: Client can now resume from latest RV<br/>even if no data changed
```

**Why Bookmarks Matter**:

```
Without bookmarks:
- Last event: RV=1000 (Pod created)
- Current etcd RV: 2000 (but no Pod changes)
- Client disconnects and reconnects
- Resume from RV=1000, replay 1000 events from other resources

With bookmarks:
- Last event: RV=1000 (Pod created)
- Bookmark: RV=2000 (progress notification)
- Client disconnects and reconnects
- Resume from RV=2000, no replay needed
```

### **Watch Failure and Reconnection**

Handling watch stream errors:

```mermaid
sequenceDiagram
    participant Client
    participant Watcher
    participant etcd

    Watcher->>etcd: Watch(WithRev(1000))

    loop Normal operation
        etcd->>Watcher: Events
        Watcher->>Client: Events (RV up to 1500)
    end

    etcd--XWatcher: Connection lost

    Note over Watcher: Error detected, close watch
    Watcher->>Client: Event{Type: Error, Reason: "connection lost"}
    Client->>Client: Note last RV=1500

    Note over Client: Wait with exponential backoff
    Client->>Client: Sleep 1s

    Client->>Watcher: Watch(rv=1500)
    Watcher->>etcd: Watch(WithRev(1501))

    alt Resume successful
        etcd-->>Watcher: Stream opened
        loop Continue
            etcd->>Watcher: Events
            Watcher->>Client: Events
        end
    else ResourceVersion too old
        etcd--XWatcher: Error: required revision has been compacted
        Watcher->>Client: Error (410 Gone)
        Note over Client: Must re-list from RV=0
        Client->>Watcher: Watch(rv=0)
    end
```

**Error Handling Code**:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:327
func logWatchChannelErr(err error) {
    switch {
    case strings.Contains(err.Error(), "mvcc: required revision has been compacted"):
        // Compaction - warn level
        klog.Warningf("watch chan error: %v", err)
    case isCancelError(err):
        // Expected cancellation - don't log
    default:
        // Unexpected error - error level
        klog.Errorf("watch chan error: %v", err)
    }
}
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:327`

### **Event Fan-out from Watch Cache**

How one etcd event reaches multiple watchers:

```mermaid
graph TB
    A[etcd Event: Pod Updated] --> B[Reflector]
    B --> C[WatchCache.Update]

    C --> D[Store updated]
    C --> E[Event added to buffer]
    C --> F[eventHandler called]

    F --> G[Cacher.processEvent]
    G --> H[incoming channel]
    H --> I[Cacher.dispatchEvents]

    I --> J{Select watchers}

    J -->|ns=default| K[Watcher 1]
    J -->|ns=default| L[Watcher 2]
    J -->|ns=kube-system| M[Watcher 3 - filtered out]
    J -->|All namespaces| N[Watcher 4]

    K --> O[Client 1]
    L --> P[Client 2]
    N --> Q[Client 4]

    style A fill:#90EE90
    style I fill:#E1F5FF
    style M fill:#FFB6C1
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔁 Synchronization Patterns**

### **Watch Resume with Bookmarks**

#### **Bookmark Generation**

The watch cache periodically generates bookmark events:

```go
// staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go:860
func (c *Cacher) dispatchEvents() {
    bookmarkTimer := c.clock.NewTimer(wait.Jitter(time.Second, 0.25))
    defer bookmarkTimer.Stop()

    // Wait for initial resource version
    lastProcessedResourceVersion := uint64(0)
    if err := wait.PollUntilContextCancel(wait.ContextForChannel(c.stopCh), 10*time.Millisecond, true,
        func(_ context.Context) (bool, error) {
            if rv := c.watchCache.getListResourceVersion(); rv != 0 {
                lastProcessedResourceVersion = rv
                return true, nil
            }
            return false, nil
        }); err != nil {
        return
    }

    for {
        select {
        case event, ok := <-c.incoming:
            if !ok {
                return
            }

            // Don't propagate etcd bookmarks (too frequent)
            if event.Type != watch.Bookmark {
                c.dispatchEvent(&event)
            }
            lastProcessedResourceVersion = event.ResourceVersion

        case <-bookmarkTimer.C():
            // Send periodic bookmark
            bookmarkTimer.Reset(wait.Jitter(time.Second, 0.25))

            bookmarkEvent := &watchCacheEvent{
                Type:            watch.Bookmark,
                Object:          c.newFunc(),
                ResourceVersion: lastProcessedResourceVersion,
            }

            if err := c.versioner.UpdateObject(bookmarkEvent.Object, bookmarkEvent.ResourceVersion); err != nil {
                klog.Errorf("failure to set resourceVersion to %d on bookmark event %+v",
                    bookmarkEvent.ResourceVersion, bookmarkEvent.Object)
                continue
            }

            c.dispatchEvent(bookmarkEvent)

        case <-c.stopCh:
            return
        }
    }
}
```

**Bookmark Frequency**:

```go
// staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go:74
const (
    defaultBookmarkFrequency = time.Minute   // Base frequency
)

// Actual frequency has jitter: 750ms - 1.25s around 1 second
```

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go:860`

### **ResourceVersion Tracking**

#### **ModRevision to ResourceVersion Mapping**

```go
// etcd ModRevision is a 64-bit counter
etcdRevision := int64(12345)

// Kubernetes ResourceVersion is the string representation
kubernetesResourceVersion := strconv.FormatInt(etcdRevision, 10)  // "12345"
```

**Version Update on Objects**:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:758
func decodeObj(codec runtime.Codec, versioner storage.Versioner, data []byte, rev int64) (_ runtime.Object, err error) {
    // Decode object from storage
    obj, err := runtime.Decode(codec, []byte(data))
    if err != nil {
        return nil, err
    }

    // Set ResourceVersion from etcd revision
    if err := versioner.UpdateObject(obj, uint64(rev)); err != nil {
        return nil, fmt.Errorf("failure to version api object (%d) %#v: %v", rev, obj, err)
    }

    return obj, nil
}
```

### **Event Transformation Pipeline**

#### **etcd Event → Kubernetes Event**

```mermaid
graph LR
    A[etcd WatchResponse] --> B[clientv3.Event]
    B --> C[Internal event struct]
    C --> D[prepareObjs: Decrypt & Decode]
    D --> E[transform: Apply filters]
    E --> F[watch.Event]

    G[PrevKv handling] --> D
    H[ModRevision] --> I[ResourceVersion]

    style A fill:#90EE90
    style F fill:#FFD700
```

**Transformation Steps**:

1. **Parse etcd event** (`parseEvent`)
2. **Prepare objects** (`prepareObjs`): Decrypt and decode
3. **Transform with filtering** (`transform`): Apply predicates and generate appropriate event type

### **Concurrency and Threading**

#### **Watch Channel Threading Model**

```mermaid
graph TB
    subgraph "watchChan goroutines"
        A[run goroutine]
        B[startWatching goroutine]
        C[processEvents goroutine]
        D[transform worker 1]
        E[transform worker 2]
        F[transform worker N]
    end

    A -->|spawn| B
    A -->|spawn| C
    C -->|spawn if concurrent| D
    C -->|spawn if concurrent| E
    C -->|spawn if concurrent| F

    G[incomingEventChan] --> C
    B -->|events| G

    H[resultChan] --> I[Client]
    C -->|events| H

    style A fill:#E1F5FF
    style B fill:#FFE5B4
    style C fill:#D4E6F1
```

**Concurrent Event Processing**:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:471
func (wc *watchChan) concurrentProcessEvents(wg *sync.WaitGroup) {
    p := concurrentOrderedEventProcessing{
        wc:              wc,
        processingQueue: make(chan chan *processingResult, processEventConcurrency-1),
        objectType:      wc.watcher.objectType,
        groupResource:   wc.watcher.groupResource,
    }

    // Schedule events for processing
    wg.Add(1)
    go func() {
        defer wg.Done()
        p.scheduleEventProcessing(wc.ctx, wg)
    }()

    // Collect processed events in order
    wg.Add(1)
    go func() {
        defer wg.Done()
        p.collectEventProcessing(wc.ctx)
    }()
}
```

**Ordering Guarantee**: Even with concurrent processing, events are delivered in order.

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:471`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Aspect-Oriented Concerns**

### **Observability**

#### **Watch Metrics**

**etcd Request Metrics**:

```go
// staging/src/k8s.io/apiserver/pkg/storage/etcd3/metrics/metrics.go
var (
    etcdRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "etcd_request_duration_seconds",
            Help:    "Etcd request latency in seconds",
            Buckets: prometheus.ExponentialBuckets(0.001, 2, 13),
        },
        []string{"operation", "type"},
    )

    etcdEventsReceived = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "etcd_events_received_total",
            Help: "Total number of events received from etcd",
        },
        []string{"resource"},
    )

    etcdBookmarkCounts = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "etcd_bookmark_counts",
            Help: "Total number of bookmark events",
        },
        []string{"resource"},
    )
)
```

**Watch Cache Metrics**:

```go
var (
    watchCacheCapacity = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "watch_cache_capacity",
            Help: "Current capacity of watch cache",
        },
        []string{"resource"},
    )

    watcherCount = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "watch_cache_watchers",
            Help: "Number of active watchers",
        },
        []string{"resource"},
    )
)
```

**Example Queries**:

```promql
# Watch request rate by resource
rate(etcd_request_duration_seconds_count{operation="watch"}[5m])

# Watch cache hit rate
rate(watch_cache_events[5m]) / rate(etcd_events_received_total[5m])

# Number of active watches
sum(watch_cache_watchers) by (resource)
```

### **Error Handling**

#### **Watch Errors**

**Common Error Types**:

| Error | Cause | Recovery |
|-------|-------|----------|
| `required revision has been compacted` | ResourceVersion too old | Re-list from RV=0 |
| `connection lost` | Network failure | Retry with backoff |
| `context canceled` | Client closed watch | Clean shutdown |
| `too large resource version` | RV in future | Wait and retry |
| `resource version too old` | Cache doesn't have history | Re-list from RV=0 |

#### **Slow Watcher Detection**

```go
// staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go (conceptual)
func (wc *cacheWatcher) nonblockingAdd(event *watchCacheEvent) bool {
    select {
    case wc.input <- event:
        return true
    default:
        // Channel full, watcher is too slow
        return false
    }
}

func (wc *cacheWatcher) add(event *watchCacheEvent, timer *time.Timer) bool {
    if timer == nil {
        wc.input <- event
        return true
    }

    select {
    case wc.input <- event:
        return true
    case <-timer.C:
        // Timeout: close slow watcher
        klog.Warningf("Terminating slow watcher for %v", wc.groupResource)
        wc.stop()
        return false
    }
}
```

### **Performance**

#### **Watch Cache Benefits**

**Load Reduction**:

```
Without watch cache:
- 100 controllers watching Pods
- Total: 100 etcd watch streams

With watch cache:
- 100 controllers watching Pods
- Total: 1 etcd watch stream
- Watch cache fans out to 100 in-memory watchers
- Fan-out ratio: 100:1
```

#### **Memory Usage**

**Watch Cache Memory**:

```
Single event size: ~3-25 KB per event
Watch cache capacity: 100 - 100,000 events
Default duration: 75 seconds

Memory usage estimate:
- 100K events × 5 KB average = 500 MB per resource
- With 10 resources = 5 GB total

Tuning:
- --watch-cache-sizes: Override per-resource capacity
- Reduce event-fresh-duration for lower memory
```

### **Reliability**

#### **At-Least-Once Delivery**

Kubernetes watch provides at-least-once delivery guarantees - clients may receive duplicate events but will not miss events (unless ResourceVersion is too old).

#### **Event Ordering**

Watch events for a single object are ordered by ResourceVersion, but global ordering across all objects is not guaranteed.

#### **Bookmark Guarantees**

Bookmark events guarantee that the ResourceVersion in the bookmark is at least as fresh as the last data event, allowing clients to resume without data loss.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Real-World Examples**

### **Example 1: Watching Pods with kubectl**

```bash
# Watch all pods in default namespace
kubectl get pods --watch

# Behind the scenes:
# 1. kubectl sends: GET /api/v1/namespaces/default/pods?watch=true
# 2. API server: Cacher.Watch()
# 3. Cacher: Create cacheWatcher, send events from current RV
# 4. kubectl: Display events as they arrive
```

**HTTP Watch Request**:

```http
GET /api/v1/namespaces/default/pods?watch=true&resourceVersion=12345 HTTP/1.1
Host: kube-apiserver:6443
Authorization: Bearer <token>
```

**HTTP Watch Response** (streaming):

```http
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked

{"type":"ADDED","object":{"kind":"Pod","metadata":{"name":"nginx","namespace":"default","resourceVersion":"12346"},...}}
{"type":"MODIFIED","object":{"kind":"Pod","metadata":{"name":"nginx","namespace":"default","resourceVersion":"12350"},...}}
{"type":"DELETED","object":{"kind":"Pod","metadata":{"name":"nginx","namespace":"default","resourceVersion":"12355"},...}}
```

### **Example 2: Controller Watching Resources**

```go
// Sample controller using client-go
func RunController(clientset kubernetes.Interface, stopCh <-chan struct{}) {
    // Create informer (uses Reflector internally)
    podInformer := informers.NewSharedInformerFactory(clientset, 30*time.Second).Core().V1().Pods()

    // Add event handlers
    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            fmt.Printf("Pod Added: %s/%s\n", pod.Namespace, pod.Name)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            pod := newObj.(*corev1.Pod)
            fmt.Printf("Pod Updated: %s/%s\n", pod.Namespace, pod.Name)
        },
        DeleteFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            fmt.Printf("Pod Deleted: %s/%s\n", pod.Namespace, pod.Name)
        },
    })

    // Start informer
    go podInformer.Informer().Run(stopCh)

    // Wait for cache sync
    if !cache.WaitForCacheSync(stopCh, podInformer.Informer().HasSynced) {
        fmt.Println("Timed out waiting for caches to sync")
        return
    }

    fmt.Println("Controller started and cache synced")
    <-stopCh
}
```

### **Example 3: etcdctl Watch**

Watch etcd directly (for debugging):

```bash
# Watch all Pods
etcdctl watch --prefix /registry/pods/

# Output:
# PUT
# /registry/pods/default/nginx
# <protobuf-encoded-pod-data>
#
# PUT
# /registry/pods/default/nginx
# <updated-protobuf-data>
#
# DELETE
# /registry/pods/default/nginx
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **✅ Best Practices**

### **1. Use Appropriate ResourceVersion Semantics**

```go
// ✓ Good: Watch from most recent
options := metav1.ListOptions{
    ResourceVersion: "",  // or "0"
}

// ✗ Bad: Hardcoded old ResourceVersion
options := metav1.ListOptions{
    ResourceVersion: "1000",  // Likely to be compacted
}
```

### **2. Handle Watch Errors Gracefully**

```go
func watchWithRetry(ctx context.Context, client kubernetes.Interface) {
    backoff := wait.NewExponentialBackoffManager(800*time.Millisecond, 30*time.Second,
        2*time.Minute, 2.0, 1.0, clock.RealClock{})

    for {
        err := doWatch(ctx, client)

        if err == nil || ctx.Err() != nil {
            return
        }

        // Check error type
        if errors.IsResourceExpired(err) {
            // ResourceVersion too old, re-list
            resourceVersion = ""
        }

        // Backoff and retry
        select {
        case <-ctx.Done():
            return
        case <-backoff.Backoff().C():
        }
    }
}
```

### **3. Enable Watch Bookmarks**

```go
// ✓ Good: Enable bookmarks to keep ResourceVersion current
options := metav1.ListOptions{
    AllowWatchBookmarks: true,
}
```

### **4. Use Informers Over Raw Watches**

```go
// ✓ Good: Use informer (built-in retry, caching, deduplication)
informer := informers.NewSharedInformerFactory(client, 30*time.Second)
podInformer := informer.Core().V1().Pods().Informer()

// ✗ Bad: Raw watch (must handle all edge cases manually)
watcher, err := client.CoreV1().Pods("").Watch(ctx, metav1.ListOptions{})
```

### **5. Limit Watch Scope**

```go
// ✓ Good: Watch specific namespace
watcher, err := client.CoreV1().Pods("default").Watch(ctx, options)

// ✓ Good: Use label selector
options := metav1.ListOptions{
    LabelSelector: "app=nginx",
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 Troubleshooting**

### **Problem 1: Watch Disconnects Frequently**

**Symptoms**: Watches close unexpectedly, frequent re-list operations

**Solutions**:

1. **Increase watch timeout**:
   ```yaml
   # kube-apiserver flag
   --min-request-timeout=1800  # 30 minutes
   ```

2. **Check network stability**: Ensure low latency between API server and etcd

### **Problem 2: "ResourceVersion Too Old" Errors**

**Symptoms**: Error 410 Gone, "resource version too old"

**Solutions**:

1. **Adjust etcd compaction**:
   ```yaml
   # etcd flags
   --auto-compaction-retention=5m  # Keep 5 minutes of history
   --auto-compaction-mode=periodic
   ```

2. **Increase watch cache size**:
   ```yaml
   # kube-apiserver flags
   --watch-cache-sizes=pods#10000
   --event-fresh-duration=10m
   ```

### **Problem 3: High Watch Latency**

**Symptoms**: Events arrive delayed, high watch_cache_event_processing latency

**Solutions**:

1. **Enable concurrent event decoding**:
   ```yaml
   # kube-apiserver feature gate
   --feature-gates=ConcurrentWatchObjectDecode=true
   ```

2. **Reduce watch scope**: Use namespace-scoped watches

### **Problem 4: Memory Exhaustion from Watch Cache**

**Symptoms**: API server OOM, high memory usage

**Solutions**:

1. **Reduce event retention**:
   ```yaml
   --event-fresh-duration=1m  # Reduce to 1 minute
   ```

2. **Limit per-resource cache size**:
   ```yaml
   --watch-cache-sizes=events#100,pods#1000
   ```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📝 Summary**

### **Key Takeaways**

1. **Watch Architecture**:
   - Three-layer design: etcd3 watcher → watch cache → client watchers
   - Watch cache dramatically reduces etcd load (fan-out ratio often >100:1)
   - Reflector implements List+Watch pattern for client-side caching

2. **Data Flow**:
   - etcd events → transform → filter → dispatch → deliver
   - ResourceVersion tracks consistency (etcd ModRevision → Kubernetes RV)
   - Bookmarks keep ResourceVersion current without data changes

3. **Key Components**:
   - **etcd3 watcher**: Direct watch to etcd, event transformation
   - **Watch cache**: Sliding window buffer, event fan-out, watcher management
   - **Reflector**: Client-side List+Watch, retry logic, cache synchronization

4. **Performance Optimizations**:
   - Concurrent event decoding (10 workers)
   - Non-blocking event dispatch
   - Cached object serializations during dispatch
   - Watch cache serves from memory

5. **Reliability Features**:
   - At-least-once delivery guarantee
   - Automatic watch resume with backoff
   - Event ordering per object
   - Bookmark support for efficient resume

### **Critical Code Locations**

| Component | File | Key Functions |
|-----------|------|---------------|
| etcd3 watcher | `watcher.go:105` | `Watch()` - Create watch |
| | `watcher.go:354` | `startWatching()` - Event streaming |
| | `watcher.go:571` | `transform()` - Event transformation |
| Event parsing | `event.go:58` | `parseEvent()` - etcd to internal |
| Watch cache | `cacher.go:378` | `Watch()` - Cache watch |
| | `cacher.go:952` | `dispatchEvent()` - Fan-out |
| | `watch_cache.go:233` | `Add()` - Event storage |
| Reflector | `reflector.go:397` | `ListAndWatch()` - List+Watch loop |

### **Next Steps**

To learn more about related topics:

- **[03-compaction-defrag.md](03-compaction-defrag.md)**: etcd compaction and defragmentation
- **[04-transactions-consistency.md](04-transactions-consistency.md)**: Optimistic concurrency and consistency
- **[07-performance-tuning.md](07-performance-tuning.md)**: Watch performance optimization
- **[../high-level/04-watch-mechanism.md](../high-level/04-watch-mechanism.md)**: High-level watch overview

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: Complete
**Lines**: 2,100+
**Diagrams**: 20
**Code References**: 22+
**Last Updated**: 2025-11-05
