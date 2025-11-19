# **API Server Scalability and Performance**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Deep architectural analysis of Kubernetes API Server scalability features

**Target Audience**:
- Platform engineers managing 5000+ node clusters
- SREs optimizing API server performance
- Architects designing high-availability control planes
- Contributors extending the API server

**Scope**: API Priority and Fairness, request handling, watch cache, storage optimization, and performance tuning

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 API Server Scalability Architecture**

### **Request Processing Pipeline**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API Server Request Processing Pipeline                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Client Request                                                          │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────┐                                                         │
│  │ TLS/Auth    │ ─────▶ Authentication (tokens, certs, webhooks)        │
│  └──────┬──────┘                                                         │
│         │                                                                │
│         ▼                                                                │
│  ┌─────────────┐                                                         │
│  │ Authorization│ ─────▶ RBAC, Webhook, ABAC                            │
│  └──────┬──────┘                                                         │
│         │                                                                │
│         ▼                                                                │
│  ┌─────────────┐        ┌─────────────────────────────────────┐         │
│  │   APF       │ ─────▶ │  Priority Levels & Flow Schemas    │         │
│  │  Queuing    │        │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐  │         │
│  └──────┬──────┘        │  │ L1  │ │ L2  │ │ L3  │ │ L4  │  │         │
│         │               │  │     │ │     │ │     │ │     │  │         │
│         │               │  └─────┘ └─────┘ └─────┘ └─────┘  │         │
│         │               └─────────────────────────────────────┘         │
│         ▼                                                                │
│  ┌─────────────┐                                                         │
│  │  Admission  │ ─────▶ Mutating Webhooks → Validating Webhooks         │
│  └──────┬──────┘                                                         │
│         │                                                                │
│         ▼                                                                │
│  ┌─────────────┐        ┌─────────────────────────────────────┐         │
│  │Watch Cache  │ ─────▶ │  In-Memory Cache (LIST/GET/WATCH)  │         │
│  │   Layer     │        └─────────────────────────────────────┘         │
│  └──────┬──────┘                                                         │
│         │ (cache miss)                                                   │
│         ▼                                                                │
│  ┌─────────────┐                                                         │
│  │   etcd      │ ─────▶ Distributed key-value store                     │
│  └─────────────┘                                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### **Scalability Bottlenecks**

| Component | Bottleneck | Impact | Scale Limit |
|-----------|-----------|--------|-------------|
| **etcd** | Write throughput | All mutations | ~10K writes/sec |
| **Watch cache** | Memory | All watches | Node count × objects |
| **APF** | Queue depth | Request latency | Concurrent requests |
| **Admission** | Webhook latency | Write latency | Webhook response time |
| **Serialization** | CPU | All requests | Object size × count |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ API Priority and Fairness (APF)**

### **APF Architecture**

API Priority and Fairness protects the API server from overload by:
1. Classifying requests into priority levels
2. Queuing requests within flow schemas
3. Dispatching based on fair sharing

**File**: `staging/src/k8s.io/apiserver/pkg/util/flowcontrol/apf_controller.go:61-89`

```go
const (
    // Seat borrowing adjustment period
    borrowingAdjustmentPeriod = 10 * time.Second

    // Demand smoothing half-life of 5 minutes
    seatDemandSmoothingCoefficient = 0.977

    // Maximum seats a priority level can claim
    priorityLevelMaxSeatsPercent = 0.15
)
```

### **Priority Level Configuration**

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: workload-high
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 100  # Relative weight
    lendablePercent: 50            # Can lend 50% of seats
    borrowingLimitPercent: 0       # Cannot borrow from others
    limitResponse:
      type: Queue
      queuing:
        queues: 64                 # Number of queues
        handSize: 6                # Random selection spread
        queueLengthLimit: 50       # Max requests per queue
```

### **Flow Schema Configuration**

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: workload-leader-election
spec:
  priorityLevelConfiguration:
    name: leader-election
  matchingPrecedence: 100  # Lower = higher priority
  distinguisherMethod:
    type: ByUser
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: "*"
        namespace: kube-system
    resourceRules:
    - verbs: ["get", "create", "update"]
      apiGroups: ["coordination.k8s.io"]
      resources: ["leases"]
```

### **Request Work Estimation**

**File**: `staging/src/k8s.io/apiserver/pkg/util/flowcontrol/request/`

APF estimates "work" for each request to fairly share API server capacity:

| Request Type | Work Estimation | Factors |
|--------------|----------------|---------|
| **GET** | 1 seat × duration | Object size |
| **LIST** | Seats × duration | Object count, limit |
| **WATCH** | 1 seat (initial) + bookmarks | Long-running |
| **CREATE/UPDATE** | 1 seat × duration | Object size |
| **DELETE** | 1 seat × duration | Cascade depth |

**List Work Estimator** (`list_work_estimator.go`):

```go
// Estimates work based on expected object count
func (e *listWorkEstimator) Estimate(r *http.Request) WorkEstimate {
    // Get object count from storage tracker
    objectCount := e.countTracker.Get(resource)

    // Apply limit if specified
    if limit > 0 && limit < objectCount {
        objectCount = limit
    }

    // Calculate seats (1 seat per 1000 objects, minimum 1)
    seats := max(1, objectCount/1000)

    return WorkEstimate{
        InitialSeats: seats,
    }
}
```

### **APF Metrics**

**File**: `staging/src/k8s.io/apiserver/pkg/util/flowcontrol/metrics/metrics.go`

Key metrics for monitoring APF:

```go
// Requests rejected by APF
apiserverRejectedRequestsTotal = NewCounterVec(
    "apiserver_flowcontrol_rejected_requests_total",
    []string{"flow_schema", "priority_level", "reason"},
)

// Seat utilization per priority level
PriorityLevelExecutionSeatsGaugeVec = NewHistogramVec(
    "apiserver_flowcontrol_priority_level_seat_utilization",
    []string{"priority_level"},
    []float64{0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1},
)

// Queue length distribution
queueLengthBuckets = []float64{0, 10, 25, 50, 100, 250, 500, 1000}
```

### **APF Tuning for Large Clusters**

**5000+ Node Cluster Configuration**:

```yaml
# High-priority system workloads
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: system-critical
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 200  # 2x default
    lendablePercent: 25            # Reserve more for self
    limitResponse:
      type: Queue
      queuing:
        queues: 128                # More queues for isolation
        handSize: 8
        queueLengthLimit: 100      # Deeper queues

---
# Exempt critical controllers from APF
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: exempt-system-controllers
spec:
  priorityLevelConfiguration:
    name: exempt
  matchingPrecedence: 1
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: kube-controller-manager
        namespace: kube-system
    nonResourceRules:
    - verbs: ["*"]
      nonResourceURLs: ["*"]
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Request Handling and Throttling**

### **Max In-Flight Limits**

**File**: `staging/src/k8s.io/apiserver/pkg/server/options/server_run_options.go:54-56`

```go
type ServerRunOptions struct {
    // MaxRequestsInFlight is the maximum number of parallel non-mutating requests
    MaxRequestsInFlight int  // Default: 400

    // MaxMutatingRequestsInFlight is the maximum number of parallel mutating requests
    MaxMutatingRequestsInFlight int  // Default: 200

    // RequestTimeout is the timeout for each request
    RequestTimeout time.Duration  // Default: 1m
}
```

### **In-Flight Filter Implementation**

**File**: `staging/src/k8s.io/apiserver/pkg/server/filters/maxinflight.go:126-149`

```go
func WithMaxInFlightLimit(handler http.Handler, nonMutatingLimit int, mutatingLimit int,
    longRunningRequestCheck apirequest.LongRunningRequestCheck) http.Handler {

    // Create buffered channels as semaphores
    nonMutatingChan := make(chan bool, nonMutatingLimit)
    mutatingChan := make(chan bool, mutatingLimit)

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Long-running requests (watches) bypass limits
        if longRunningRequestCheck(r, ...) {
            handler.ServeHTTP(w, r)
            return
        }

        // Select appropriate channel
        var c chan bool
        if isMutatingRequest(r) {
            c = mutatingChan
        } else {
            c = nonMutatingChan
        }

        // Try to acquire slot
        select {
        case c <- true:
            defer func() { <-c }()
            handler.ServeHTTP(w, r)
        default:
            // Too many requests - return 429
            metrics.RecordDroppedRequest(r, ...)
            tooManyRequests(r, w)
        }
    })
}
```

### **Request Categories**

| Category | Examples | Limit Applied | Long-Running |
|----------|----------|--------------|--------------|
| **Non-mutating** | GET, LIST | MaxRequestsInFlight | No |
| **Mutating** | CREATE, UPDATE, DELETE | MaxMutatingRequestsInFlight | No |
| **Watch** | WATCH | None (bypasses) | Yes |
| **Proxy** | /proxy/* | None (bypasses) | Yes |
| **Exec/Attach** | /exec, /attach | None (bypasses) | Yes |

### **Timeout Configuration**

**File**: `staging/src/k8s.io/apiserver/pkg/server/filters/timeout.go`

```go
// Request timeout wraps all handlers
func WithTimeoutForNonLongRunningRequests(handler http.Handler, timeout time.Duration) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), timeout)
        defer cancel()

        r = r.WithContext(ctx)
        handler.ServeHTTP(w, r)
    })
}
```

### **Graceful Shutdown**

**File**: `staging/src/k8s.io/apiserver/pkg/server/options/server_run_options.go:71-95`

```go
type ServerRunOptions struct {
    // ShutdownDelayDuration is the time to wait before starting shutdown
    ShutdownDelayDuration time.Duration  // Default: 0

    // ShutdownSendRetryAfter sends 429 with Retry-After during shutdown
    ShutdownSendRetryAfter bool  // Default: false

    // ShutdownWatchTerminationGracePeriod is max time to wait for watches to drain
    ShutdownWatchTerminationGracePeriod time.Duration  // Default: 0
}
```

**Shutdown Sequence**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API Server Graceful Shutdown                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. SIGTERM received                                                     │
│     └─▶ Start ShutdownDelayDuration (allow LB to drain)                 │
│                                                                          │
│  2. Stop accepting new connections                                       │
│     └─▶ Return 429 with Retry-After if ShutdownSendRetryAfter=true      │
│                                                                          │
│  3. Wait for in-flight requests to complete                              │
│     └─▶ RequestTimeout for each request                                  │
│                                                                          │
│  4. Terminate watches                                                    │
│     └─▶ Wait up to ShutdownWatchTerminationGracePeriod                  │
│                                                                          │
│  5. Close etcd connections                                               │
│                                                                          │
│  6. Exit process                                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📦 Watch Cache**

### **Watch Cache Architecture**

The watch cache provides:
1. **In-memory serving** for GET/LIST/WATCH
2. **Event history** for resumable watches
3. **Filtered serving** without etcd round-trips

**File**: `staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go:47-64`

```go
const (
    // Time to wait for resource version propagation
    blockTimeout = 3 * time.Second

    // Retry delay for resource version too high
    resourceVersionTooHighRetrySeconds = 1

    // Cache capacity bounds
    defaultLowerBoundCapacity = 100
    defaultUpperBoundCapacity = 100 * 1024  // 100K events

    // Chunk size for initial watch list
    storageWatchListPageSize = 10000

    // Bookmark frequency
    defaultBookmarkFrequency = 1 * time.Minute
)
```

### **Watch Cache Structure**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go:88-99`

```go
type watchCache struct {
    sync.RWMutex

    // Condition variable for list synchronization
    cond *sync.Cond

    // Capacity controls
    capacity           int
    upperBoundCapacity int

    // Event ring buffer (sliding window)
    cache      []*watchCacheEvent
    startIndex int
    endIndex   int

    // Current store state
    store cache.Indexer

    // Resource version tracking
    resourceVersion uint64

    // Event handlers
    eventHandler func(*watchCacheEvent)
}
```

### **Watch Cache Event**

```go
type watchCacheEvent struct {
    Type            watch.EventType
    Object          runtime.Object
    ObjLabels       labels.Set
    ObjFields       fields.Set
    PrevObject      runtime.Object
    PrevObjLabels   labels.Set
    PrevObjFields   fields.Set
    Key             string
    ResourceVersion uint64
    RecordTime      time.Time
}
```

### **Cache Configuration**

**File**: `staging/src/k8s.io/apiserver/pkg/server/options/etcd.go:62-67`

```go
type EtcdOptions struct {
    // EnableWatchCache enables the watch cache (default: true)
    EnableWatchCache bool

    // DefaultWatchCacheSize is the default cache size per resource
    DefaultWatchCacheSize int  // Default: 100

    // WatchCacheSizes overrides cache size for specific resources
    // Format: "resource#size" e.g., "pods#1000"
    WatchCacheSizes []string
}
```

### **Cache Sizing for Large Clusters**

**Recommended Configuration**:

```bash
kube-apiserver \
  --watch-cache-sizes=pods#10000,nodes#5000,events#10000,secrets#5000 \
  --default-watch-cache-size=1000
```

**Memory Calculation**:

```
Cache Memory = Σ (resource_cache_size × average_object_size)

Example for 5000-node cluster:
- Pods (50,000 × 5KB cache entry) = 250 MB
- Nodes (5,000 × 10KB) = 50 MB
- Events (100,000 × 2KB) = 200 MB
- Total cache memory ≈ 500 MB - 1 GB per API server
```

### **BTree Store Implementation**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/cacher/store_btree.go`

```go
const btreeDegree = 16  // Optimal B-tree degree from benchmarks
```

**Store Element** (`store.go:93-98`):

```go
type storeElement struct {
    Key    string
    Object runtime.Object
    Labels labels.Set
    Fields fields.Set
}
```

Pre-computing labels and fields avoids redundant parsing for each LIST/WATCH filter.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💾 Storage Layer Optimization**

### **Etcd Connection Configuration**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/storagebackend/factory/etcd3.go:61-72`

```go
const (
    // Keep-alive for connection health detection
    keepaliveTime    = 30 * time.Second
    keepaliveTimeout = 10 * time.Second

    // Connection establishment timeout (high for ARM64 TLS)
    dialTimeout = 20 * time.Second

    // Jitter for database metrics polling
    dbMetricsMonitorJitter = 0.5
)
```

### **Storage Media Types**

**File**: `staging/src/k8s.io/apiserver/pkg/server/options/etcd.go:91-95`

| Media Type | Description | Performance |
|------------|-------------|-------------|
| `application/json` | Human-readable (default) | Larger size, slower |
| `application/vnd.kubernetes.protobuf` | Binary format | 30-50% smaller, faster |
| `application/yaml` | Text format | Larger size, slower |

**Recommendation**: Use protobuf for production clusters:

```bash
kube-apiserver --storage-media-type=application/vnd.kubernetes.protobuf
```

### **Compaction Configuration**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/storagebackend/config.go:38-43`

```go
const (
    DefaultCompactInterval      = 5 * time.Minute
    DefaultDBMetricPollInterval = 30 * time.Second
    DefaultHealthcheckTimeout   = 2 * time.Second
)
```

### **Pagination**

**Watch List Page Size** (`cacher.go:64`):

```go
const storageWatchListPageSize = 10000
```

This prevents loading entire datasets during initial synchronization.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔌 Connection Management**

### **HTTP/2 Configuration**

**File**: `staging/src/k8s.io/apiserver/pkg/server/options/serving.go:40-81`

```go
type SecureServingOptions struct {
    // DisableHTTP2Serving disables HTTP/2 support
    DisableHTTP2Serving bool

    // HTTP2MaxStreamsPerConnection limits concurrent streams per connection
    HTTP2MaxStreamsPerConnection int  // Default: 0 (library default)

    // PermitPortSharing enables SO_REUSEPORT
    PermitPortSharing bool

    // PermitAddressSharing enables SO_REUSEADDR
    PermitAddressSharing bool
}
```

### **Connection Pooling Benefits**

HTTP/2 multiplexing advantages:
- Single TCP connection per client
- Concurrent streams (default ~100)
- Header compression
- Server push (not used by Kubernetes)

**Tuning**:

```bash
kube-apiserver \
  --http2-max-streams-per-connection=1000  # For heavy clients
```

### **Watch Termination**

**File**: `staging/src/k8s.io/apiserver/pkg/server/filters/watch_termination.go`

Handles graceful termination of watch connections during shutdown:

```go
func WithWatchTerminationDuringShutdown(handler http.Handler, termination <-chan struct{},
    wg *sync.WaitGroup) http.Handler {

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        select {
        case <-termination:
            // Server shutting down - reject new watches
            http.Error(w, "server is shutting down", http.StatusServiceUnavailable)
        default:
            wg.Add(1)
            defer wg.Done()
            handler.ServeHTTP(w, r)
        }
    })
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Admission Controllers**

### **Admission Framework**

**File**: `staging/src/k8s.io/apiserver/pkg/admission/interfaces.go:29-77`

```go
type Attributes interface {
    // Request identification
    GetName() string
    GetNamespace() string
    GetResource() schema.GroupVersionResource
    GetSubresource() string
    GetOperation() Operation

    // Object data
    GetObject() runtime.Object
    GetOldObject() runtime.Object

    // User information
    GetUserInfo() user.Info

    // Audit annotations
    AddAnnotation(key, value string) error
    AddAnnotationWithLevel(key, value string, level auditinternal.Level) error
}
```

### **Admission Chain**

**File**: `staging/src/k8s.io/apiserver/pkg/admission/chain.go`

```go
type chainAdmissionHandler []Interface

func (admissionHandler chainAdmissionHandler) Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error {
    for _, handler := range admissionHandler {
        if !handler.Handles(a.GetOperation()) {
            continue
        }
        err := handler.Admit(ctx, a, o)
        if err != nil {
            return err
        }
    }
    return nil
}
```

### **Webhook Admission**

**Mutating Webhooks** (`plugin/webhook/mutating/`):
- Called before persistence
- Can modify request objects
- Sequential execution (order matters)

**Validating Webhooks** (`plugin/webhook/validating/`):
- Called after persistence decision
- Cannot modify objects
- Parallel execution possible

### **Webhook Timeout**

**File**: `staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/dispatcher.go:275`

```go
// Per-webhook timeout from configuration
ctx, cancel := context.WithTimeout(ctx, time.Duration(*hook.TimeoutSeconds)*time.Second)
defer cancel()
```

### **Admission Performance Impact**

| Admission Type | Latency Impact | Scaling Concern |
|----------------|----------------|-----------------|
| **Built-in** | <1ms | Negligible |
| **ResourceQuota** | 1-10ms | Large namespace quotas |
| **Mutating Webhook** | 10-100ms | Network latency |
| **Validating Webhook** | 10-100ms | Can parallelize |
| **Policy Webhook** | 10-100ms | Complex policies |

### **Admission Metrics**

**File**: `staging/src/k8s.io/apiserver/pkg/admission/metrics/`

```go
// Track webhook call latencies
webhookLatency = NewHistogramVec(
    "apiserver_admission_webhook_admission_duration_seconds",
    []string{"name", "type", "operation", "rejected"},
    []float64{0.005, 0.025, 0.1, 0.5, 1, 2.5},
)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Aggregated API Servers**

### **Aggregator Architecture**

**File**: `staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go:78-99`

```go
type ExtraConfig struct {
    // ProxyClientCert is used to authenticate to backend API servers
    ProxyClientCertFile string
    ProxyClientKeyFile  string

    // ProxyTransport for custom connection handling
    ProxyTransport *http.Transport
}
```

### **Proxy Handler**

**File**: `staging/src/k8s.io/kube-aggregator/pkg/apiserver/handler_proxy.go:48-80`

```go
type proxyHandler struct {
    // Local delegate for local APIServices
    localDelegate http.Handler

    // Transport for proxying
    proxyTransportDial *transport.DialHolder

    // Service resolution
    serviceResolver ServiceResolver

    // OpenTelemetry tracing
    tracerProvider oteltrace.TracerProvider
}
```

### **APIService Controller**

**File**: `staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiservice_controller.go`

Manages APIService objects and maintains routing information:

```go
// APIService defines an aggregated API
type APIService struct {
    Spec APIServiceSpec
    Status APIServiceStatus
}

type APIServiceSpec struct {
    // Service reference for backend
    Service *ServiceReference

    // GroupVersion this APIService provides
    Group   string
    Version string

    // InsecureSkipTLSVerify skips TLS verification
    InsecureSkipTLSVerify bool

    // CABundle for TLS verification
    CABundle []byte
}
```

### **OpenAPI Aggregation**

**File**: `staging/src/k8s.io/kube-aggregator/pkg/controllers/openapi/aggregator/`

- `aggregator.go` - Combines OpenAPI specs from all APIServices
- `priority.go` - Prioritization for conflicting specs
- `downloader.go` - Fetches specs from aggregated servers

### **Aggregation Performance**

**Considerations**:
- Connection pooling to backend APIServices
- DNS-based load balancing via Service resolution
- Keepalive for persistent connections

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Metrics and Monitoring**

### **Request Metrics**

**File**: `staging/src/k8s.io/apiserver/pkg/endpoints/metrics/`

```go
// Request latency by verb, resource
apiserver_request_duration_seconds{verb, resource, subresource, scope, code}

// Request count by status
apiserver_request_total{verb, resource, subresource, scope, code}

// Request size
apiserver_request_body_size_bytes{verb, resource, subresource}

// Response size
apiserver_response_size_bytes{verb, resource, subresource}
```

### **APF Metrics**

```go
// Rejected requests
apiserver_flowcontrol_rejected_requests_total{flow_schema, priority_level, reason}

// Dispatched requests
apiserver_flowcontrol_dispatched_requests_total{flow_schema, priority_level}

// Seat utilization
apiserver_flowcontrol_priority_level_seat_utilization{priority_level}

// Request utilization
apiserver_flowcontrol_priority_level_request_utilization{priority_level}

// Read vs write ratio
apiserver_flowcontrol_read_vs_write_current_requests{priority_level, phase}
```

### **Storage Metrics**

```go
// Etcd request latency
etcd_request_duration_seconds{operation, type}

// Database size
apiserver_storage_db_total_size_in_bytes{endpoint}

// List metrics
apiserver_storage_list_total{resource}
apiserver_storage_list_fetched_objects_total{resource}
apiserver_storage_list_returned_objects_total{resource}
```

### **Key Alerts for Large Clusters**

```yaml
# APF rejecting requests
- alert: APIServerAPFRejections
  expr: rate(apiserver_flowcontrol_rejected_requests_total[5m]) > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "API server rejecting requests due to APF"

# High request latency
- alert: APIServerHighLatency
  expr: histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m])) > 2
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "API server p99 latency exceeds 2 seconds"

# Watch cache not serving
- alert: APIServerWatchCacheMiss
  expr: rate(apiserver_cache_list_total{resourceVersion=""}[5m]) / rate(apiserver_cache_list_total[5m]) > 0.1
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Watch cache miss rate exceeds 10%"

# etcd slow
- alert: EtcdHighLatency
  expr: histogram_quantile(0.99, rate(etcd_request_duration_seconds_bucket[5m])) > 0.5
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "etcd p99 latency exceeds 500ms"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 Production Tuning**

### **Tuning Parameters for 5000+ Node Clusters**

| Parameter | Default | Recommended | Rationale |
|-----------|---------|-------------|-----------|
| `--max-requests-inflight` | 400 | 800-2000 | More concurrent reads |
| `--max-mutating-requests-inflight` | 200 | 400-1000 | More concurrent writes |
| `--request-timeout` | 1m | 1-2m | Larger operations |
| `--default-watch-cache-size` | 100 | 1000-5000 | Better cache coverage |
| `--storage-media-type` | json | protobuf | 30-50% smaller |
| `--etcd-compaction-interval` | 5m | 15-30m | Less etcd overhead |
| `--shutdown-delay-duration` | 0 | 5-10s | Graceful LB drain |

### **Complete API Server Configuration**

```bash
kube-apiserver \
  # Request throttling
  --max-requests-inflight=1200 \
  --max-mutating-requests-inflight=600 \
  --request-timeout=2m \
  --min-request-timeout=300 \

  # Watch cache
  --watch-cache-sizes=pods#10000,nodes#5000,events#10000,secrets#5000,configmaps#5000 \
  --default-watch-cache-size=1000 \

  # Storage
  --storage-media-type=application/vnd.kubernetes.protobuf \
  --etcd-compaction-interval=15m \
  --etcd-count-metric-poll-period=5m \

  # Connection management
  --http2-max-streams-per-connection=1000 \

  # Graceful shutdown
  --shutdown-delay-duration=10s \
  --shutdown-send-retry-after=true \
  --shutdown-watch-termination-grace-period=10s \

  # Audit (batch for performance)
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \
  --audit-log-path=/var/log/kubernetes/audit.log \
  --audit-log-maxage=7 \
  --audit-log-maxbackup=10 \
  --audit-log-maxsize=100
```

### **APF Configuration for High Load**

```yaml
# Priority for system controllers
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: system-high
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 200
    lendablePercent: 30
    limitResponse:
      type: Queue
      queuing:
        queues: 128
        handSize: 8
        queueLengthLimit: 100

---
# Flow schema for controller-manager
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: system-controller-manager
spec:
  priorityLevelConfiguration:
    name: system-high
  matchingPrecedence: 100
  distinguisherMethod:
    type: ByUser
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: kube-controller-manager
        namespace: kube-system
    resourceRules:
    - verbs: ["*"]
      apiGroups: ["*"]
      resources: ["*"]
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting Guide**

### **Common Issues**

| Issue | Symptom | Root Cause | Solution |
|-------|---------|------------|----------|
| **APF rejections** | 429 errors | Priority level saturated | Tune APF config, increase concurrency |
| **Slow LIST** | Timeouts on large lists | Cache miss, etcd slow | Increase cache size, use pagination |
| **Watch disconnects** | Clients reconnecting | Resource version compacted | Increase etcd compaction interval |
| **High latency** | All requests slow | Admission webhooks | Optimize webhooks, increase timeouts |
| **Memory pressure** | OOM kills | Large watch caches | Tune cache sizes, add replicas |

### **Diagnostic Commands**

```bash
# Check APF configuration
kubectl get flowschemas
kubectl get prioritylevelconfigurations

# View APF metrics
kubectl get --raw /metrics | grep apiserver_flowcontrol

# Check request latencies
kubectl get --raw /metrics | grep apiserver_request_duration

# View watch cache sizes
kubectl get --raw /metrics | grep apiserver_cache

# Check etcd latency
kubectl get --raw /metrics | grep etcd_request_duration

# Audit APF decisions
kubectl get --raw /debug/api_priority_and_fairness/dump_priority_levels
kubectl get --raw /debug/api_priority_and_fairness/dump_queues
```

### **Performance Investigation Flow**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API Server Performance Investigation                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Check APF rejections                                                 │
│     └─▶ apiserver_flowcontrol_rejected_requests_total                   │
│     └─▶ Tune priority levels if rejecting                               │
│                                                                          │
│  2. Check request latency breakdown                                      │
│     └─▶ apiserver_request_duration_seconds by stage                     │
│     └─▶ Identify slowest stage (admission, storage, etc.)               │
│                                                                          │
│  3. Check etcd performance                                               │
│     └─▶ etcd_request_duration_seconds                                   │
│     └─▶ If slow, investigate etcd cluster health                        │
│                                                                          │
│  4. Check watch cache effectiveness                                      │
│     └─▶ apiserver_cache_list_fetched vs returned                        │
│     └─▶ If high ratio, increase cache sizes                             │
│                                                                          │
│  5. Check admission webhook latency                                      │
│     └─▶ apiserver_admission_webhook_admission_duration_seconds          │
│     └─▶ Optimize slow webhooks                                          │
│                                                                          │
│  6. Check resource utilization                                           │
│     └─▶ CPU, memory, network on API server pods                         │
│     └─▶ Scale horizontally if saturated                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Related Documentation**

### **Kubernetes Source Files**

| Component | File Path |
|-----------|-----------|
| APF Controller | `staging/src/k8s.io/apiserver/pkg/util/flowcontrol/apf_controller.go` |
| APF Filter | `staging/src/k8s.io/apiserver/pkg/server/filters/priority-and-fairness.go` |
| Max In-Flight | `staging/src/k8s.io/apiserver/pkg/server/filters/maxinflight.go` |
| Watch Cache | `staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go` |
| Cacher | `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go` |
| Server Config | `staging/src/k8s.io/apiserver/pkg/server/config.go` |
| Server Options | `staging/src/k8s.io/apiserver/pkg/server/options/server_run_options.go` |
| Etcd Options | `staging/src/k8s.io/apiserver/pkg/server/options/etcd.go` |
| Admission | `staging/src/k8s.io/apiserver/pkg/admission/interfaces.go` |
| Aggregator | `staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go` |

### **Related Architecture Docs**

- `scalability/01-large-cluster-architecture.md` - Overall scalability patterns
- `scalability/02-scalability-limits.md` - Known limits
- `scalability/03-performance-benchmarking.md` - Benchmarking methodology
- `scalability/06-component-optimization.md` - Component tuning
- `advanced-topics/03-etcd-scalability-deep-dive.md` - etcd optimization

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **✨ Key Takeaways**

1. **APF is Critical**: API Priority and Fairness protects against overload; tune it carefully
2. **Watch Cache**: Essential for performance; size it based on cluster scale
3. **Protobuf Storage**: 30-50% reduction in storage and serialization overhead
4. **Admission Webhooks**: Major latency contributor; optimize or eliminate slow ones
5. **HTTP/2**: Multiplexing reduces connection overhead; tune max streams
6. **Graceful Shutdown**: Configure properly to avoid client disruption
7. **Monitor Everything**: APF metrics, latencies, cache hits are essential

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Last Updated**: 2024-11-19
**Kubernetes Version**: 1.31+
