# **Etcd Scalability Deep Dive**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Deep architectural analysis of Kubernetes etcd storage layer scalability

**Target Audience**:
- Platform engineers managing 5000+ node clusters
- SREs optimizing etcd performance
- Architects designing high-availability control planes
- Operations teams troubleshooting storage issues

**Scope**: Client configuration, storage backend, watches, compaction, pagination, leases, encryption, and performance tuning

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Etcd Architecture in Kubernetes**

### **Storage Layer Overview**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Etcd Storage Architecture                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                │
│  │ API Server 1│     │ API Server 2│     │ API Server 3│                │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘                │
│         │                   │                   │                        │
│         └───────────────────┼───────────────────┘                        │
│                             │                                            │
│                             ▼                                            │
│              ┌─────────────────────────────┐                             │
│              │    Storage Interface        │                             │
│              │  (Versioned CRUD + Watch)   │                             │
│              └──────────────┬──────────────┘                             │
│                             │                                            │
│              ┌──────────────┼──────────────┐                             │
│              ▼              ▼              ▼                             │
│       ┌──────────┐   ┌──────────┐   ┌──────────┐                        │
│       │ etcd     │   │ etcd     │   │ etcd     │                        │
│       │ Node 1   │◀─▶│ Node 2   │◀─▶│ Node 3   │                        │
│       │ (Leader) │   │(Follower)│   │(Follower)│                        │
│       └──────────┘   └──────────┘   └──────────┘                        │
│              │              │              │                             │
│              ▼              ▼              ▼                             │
│       ┌──────────┐   ┌──────────┐   ┌──────────┐                        │
│       │   Disk   │   │   Disk   │   │   Disk   │                        │
│       │  (WAL)   │   │  (WAL)   │   │  (WAL)   │                        │
│       └──────────┘   └──────────┘   └──────────┘                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### **Data Flow**

| Operation | Path | Consistency | Performance |
|-----------|------|-------------|-------------|
| **Write** | API Server → Leader → Followers → ACK | Linearizable | ~10ms |
| **Read** | API Server → Any Node | Serializable | ~1ms |
| **Watch** | API Server → Any Node | Serializable | Streaming |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 Client Configuration**

### **Connection Parameters**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/storagebackend/factory/etcd3.go:60-72`

```go
const (
    // Keep-alive to detect dead connections aggressively
    keepaliveTime    = 30 * time.Second
    keepaliveTimeout = 10 * time.Second

    // Connection establishment timeout
    // Set to 20 seconds due to ARM64 TLS performance issues (#64649)
    dialTimeout = 20 * time.Second

    // Jitter for metrics polling
    dbMetricsMonitorJitter = 0.5
)
```

**Design Rationale**:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `keepaliveTime` | 30s | Send TCP keep-alive probes every 30s |
| `keepaliveTimeout` | 10s | Mark connection dead after 10s without response |
| `dialTimeout` | 20s | Higher timeout for ARM64 TLS handshake |

### **Backend Configuration**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/storagebackend/config.go:60-96`

```go
type Config struct {
    Type   string                          // "etcd3"
    Prefix string                          // "/registry"

    // Connection settings
    Transport TransportConfig              // Endpoints, TLS

    // Serialization
    Codec           runtime.Codec          // JSON or Protobuf
    EncodeVersioner runtime.GroupVersioner
    Transformer     value.Transformer      // Encryption

    // Performance tuning
    CompactionInterval    time.Duration    // Default: 5 minutes
    CountMetricPollPeriod time.Duration    // Object count metrics
    DBMetricPollInterval  time.Duration    // Default: 30 seconds
    EventsHistoryWindow   time.Duration    // Default: 75 seconds
    HealthcheckTimeout    time.Duration    // Default: 2 seconds
    ReadycheckTimeout     time.Duration    // Default: 2 seconds

    // Lease management
    LeaseManagerConfig LeaseManagerConfig

    // Object tracking
    StorageObjectCountTracker flowcontrolrequest.StorageObjectCountTracker
}
```

### **Transport Configuration**

```go
type TransportConfig struct {
    // Etcd endpoints
    ServerList []string

    // TLS credentials
    KeyFile       string
    CertFile      string
    TrustedCAFile string

    // Egress selector for konnectivity proxy
    EgressLookup egressselector.Lookup

    // OpenTelemetry tracing
    TracerProvider oteltrace.TracerProvider
}
```

### **Default Values**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/storagebackend/config.go:38-43`

```go
const (
    DefaultCompactInterval      = 5 * time.Minute
    DefaultDBMetricPollInterval = 30 * time.Second
    DefaultEventsHistoryWindow  = 75 * time.Second
    DefaultHealthcheckTimeout   = 2 * time.Second
    DefaultReadinessTimeout     = 2 * time.Second
)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💾 Storage Backend Implementation**

### **Store Structure**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:80-98`

```go
type store struct {
    client      *kubernetes.Client       // Etcd client
    codec       runtime.Codec            // Serialization
    versioner   storage.Versioner        // ResourceVersion tracking
    transformer value.Transformer        // Encryption/compression
    pathPrefix  string                   // Key prefix ("/registry")
    groupResource schema.GroupResource   // Resource type
    watcher     *watcher                 // Watch implementation
    leaseManager *leaseManager           // Lease reuse
    decoder     Decoder                  // Custom decoder

    resourcePrefix string                // Resource-specific prefix
    newListFunc    func() runtime.Object // List factory
    compactor      Compactor             // Compaction handler
}
```

### **Key Prefix Structure**

**Default Prefix** (`pkg/kubeapiserver/options/options.go:33`):

```go
const DefaultEtcdPathPrefix = "/registry"
```

**Key Organization**:

```
/registry/
├── pods/
│   ├── default/
│   │   ├── pod-1
│   │   └── pod-2
│   └── kube-system/
│       └── coredns-xxx
├── services/
│   └── default/
│       └── kubernetes
├── deployments/
├── nodes/
│   ├── node-1
│   └── node-2
├── events/
├── secrets/
└── configmaps/
```

### **CRUD Operations**

#### **Get Operation** (`store.go:238-271`)

```go
func (s *store) Get(ctx context.Context, key string, opts storage.GetOptions, out runtime.Object) error {
    startTime := time.Now()

    // Prepare key with prefix
    preparedKey, err := s.prepareKey(key, false)

    // Get from etcd
    getResp, err := s.client.Kubernetes.Get(ctx, preparedKey, kubernetes.GetOptions{})
    metrics.RecordEtcdRequest("get", s.groupResource, err, startTime)

    // Decrypt if encrypted
    data, _, err := s.transformer.TransformFromStorage(ctx, getResp.KV.Value, authenticatedDataString(preparedKey))

    // Decode into output object
    return s.decoder.Decode(data, out, getResp.KV.ModRevision)
}
```

#### **Create Operation** (`store.go:274-339`)

```go
func (s *store) Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error {
    // 1. Serialize object
    data, err := runtime.Encode(s.codec, obj)

    // 2. Get/reuse lease for TTL (critical for performance)
    var lease clientv3.LeaseID
    if ttl != 0 {
        lease, err = s.leaseManager.GetLease(ctx, int64(ttl))
    }

    // 3. Encrypt data
    newData, err := s.transformer.TransformToStorage(ctx, data, authenticatedDataString(preparedKey))

    // 4. Optimistic put with version check
    txnResp, err := s.client.Kubernetes.OptimisticPut(ctx, preparedKey, newData, 0, kubernetes.PutOptions{
        LeaseID: lease,
    })

    return nil
}
```

#### **GuaranteedUpdate Operation** (`store.go:463-628`)

This is the most complex operation, implementing optimistic concurrency:

```go
func (s *store) GuaranteedUpdate(ctx context.Context, key string, destination runtime.Object,
    ignoreNotFound bool, preconditions *storage.Preconditions,
    tryUpdate storage.UpdateFunc, cachedExistingObject runtime.Object) error {

    for {
        // 1. Get current state
        origState, err := getCurrentState()

        // 2. Validate preconditions (UID, ResourceVersion)
        if preconditions != nil {
            if err := preconditions.Check(key, origState.obj); err != nil {
                // Retry with fresh fetch if condition fails
            }
        }

        // 3. Apply update function
        ret, ttl, err := s.updateState(origState, tryUpdate)

        // 4. Serialize and encrypt
        data, err := runtime.Encode(s.codec, ret)
        newData, err := s.transformer.TransformToStorage(ctx, data, transformContext)

        // 5. Optimistic CAS transaction
        txnResp, err := s.client.Kubernetes.OptimisticPut(ctx, preparedKey, newData, origState.rev, kubernetes.PutOptions{
            GetOnFailure: true,  // Return current value on conflict
            LeaseID:      lease,
        })

        // 6. Handle conflict with retry
        if !txnResp.Succeeded {
            // Get new state from response, retry
            origState, err = s.getState(ctx, txnResp.KV, ...)
            continue
        }

        return nil
    }
}
```

**Performance Note**: `GetOnFailure: true` avoids extra RPC on conflict.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **👁️ Watch Implementation**

### **Watch Architecture**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:47-52`

```go
const (
    incomingBufSize         = 100   // Etcd → processing buffer
    outgoingBufSize         = 100   // Processing → client buffer
    processEventConcurrency = 10    // Parallel event processors
)
```

### **Watch Channel Structure**

**File**: `watcher.go:84-96`

```go
type watchChan struct {
    watcher       *watcher
    key           string                        // Watch key prefix
    initialRev    int64                         // Start revision
    recursive     bool                          // Watch subtree
    progressNotify bool                         // Send bookmarks
    internalPred  storage.SelectionPredicate    // Label/field filters

    ctx           context.Context
    cancel        context.CancelFunc

    incomingEventChan chan *event               // From etcd
    resultChan        chan watch.Event          // To client
}
```

### **Watch Initialization**

**File**: `watcher.go:105-127`

```go
func (w *watcher) Watch(ctx context.Context, key string, rev int64, opts storage.ListOptions) (watch.Interface, error) {
    // Validate recursive key format
    if opts.Recursive && !strings.HasSuffix(key, "/") {
        return nil, fmt.Errorf(`recursive key needs to end with "/"`)
    }

    // Get starting revision
    startWatchRV, err := w.getStartWatchResourceVersion(ctx, rev, opts)

    // Create watch channel
    wc := w.createWatchChan(ctx, key, startWatchRV, opts.Recursive, opts.ProgressNotify, opts.Predicate)

    // Start event processing loop
    go wc.run(isInitialEventsEndBookmarkRequired(opts), areInitialEventsRequired(rev, opts))

    // Signal watch initialized (for APF)
    utilflowcontrol.WatchInitialized(ctx)

    return wc, nil
}
```

### **Buffer Sizing Considerations**

**For 5000+ node clusters with high churn**:

```
Event Rate Analysis:
- If etcd sends 1000 events/sec
- Buffer size = 100
- Buffer fills in 0.1 seconds
- Risk: Event drops if processing slower than ingestion

Mitigation:
- Increase processEventConcurrency (requires code change)
- Ensure fast event processing
- Monitor watch reconnection rate
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Compaction and Defragmentation**

### **Distributed Compaction Architecture**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/compact.go`

Kubernetes uses a **lease-based distributed compaction** algorithm to prevent duplicate compactions across API servers.

### **Compaction Algorithm**

**File**: `compact.go:153-214`

```go
func (c *compactor) runCompactLoop(stopCh chan struct{}) {
    // Uses special key: "compact_rev_key"
    // - Value: revision number to compact
    // - Version: logical clock for leader election

    var compactTime int64  // Version of compact_rev_key
    var rev int64          // Global revision

    for {
        select {
        case <-c.clock.After(c.interval):  // Default: 5 minutes
        case <-ctx.Done():
            return
        }

        // Perform compaction attempt
        compactTime, rev, compactRev, err = Compact(ctx, c.client, compactTime, rev)
    }
}
```

### **Compact Function**

**File**: `compact.go:219-252`

```go
func Compact(ctx context.Context, client *clientv3.Client, expectVersion, rev int64)
    (currentVersion, currentRev, compactRev int64, err error) {

    // Atomic CAS transaction for leader election
    resp, err := client.KV.Txn(ctx).If(
        clientv3.Compare(clientv3.Version(compactRevKey), "=", expectVersion),
    ).Then(
        // Won leadership - update compact revision
        clientv3.OpPut(compactRevKey, strconv.FormatInt(rev, 10)),
    ).Else(
        // Lost leadership - read current state
        clientv3.OpGet(compactRevKey),
    ).Commit()

    if !resp.Succeeded {
        // Another compactor won - adopt their revision
        currentVersion = resp.Responses[0].GetResponseRange().Kvs[0].Version
        return
    }

    // Perform actual compaction
    if rev != 0 {
        _, err = client.Compact(ctx, rev)
    }
}
```

### **Compaction Behavior at Scale**

**With 3 API servers and 5-minute interval**:

```
Timeline:
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ 0   │ 5   │ 10  │ 15  │ 20  │ 25  │ 30  │ 35  │ 40  │ 45  │ (minutes)
├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
│ A1  │     │ A2  │     │ A3  │     │ A1  │     │ A2  │     │ (compaction)
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘

A1, A2, A3 = API Server 1, 2, 3 winning leadership
Only one compaction per interval (distributed consensus)
```

### **Compaction Impact**

| Cluster Size | Compaction Duration | Recommendation |
|--------------|--------------------|--------------  |
| <500 nodes | 1-5 seconds | 5-minute interval |
| 500-5000 nodes | 5-30 seconds | 10-15 minute interval |
| >5000 nodes | 30-60 seconds | 30-minute interval |

**Risk**: Compaction blocks other etcd operations briefly. Longer intervals reduce frequency but allow more history accumulation.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📄 Pagination and Continuation Tokens**

### **Continuation Token Format**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/continue.go:35-43`

```go
type continueToken struct {
    APIVersion      string `json:"v"`      // "meta.k8s.io/v1"
    ResourceVersion int64  `json:"rv"`     // Etcd revision for consistency
    StartKey        string `json:"start"`  // Last key seen (exclusive)
}

// Encoded: base64(json.Marshal(token))
// Example: "eyJ2IjoibWV0YS5rOHMuaW8vdjEiLCJydiI6MTAwLCJzdGFydCI6InBvZC00MjMifQ=="
```

### **GetList Implementation**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:736-899`

```go
func (s *store) GetList(ctx context.Context, key string, opts storage.ListOptions, listObj runtime.Object) error {
    limit := opts.Predicate.Limit
    paging := limit > 0

    // Parse continue token
    withRev, continueKey, err := storage.ValidateListOptions(keyPrefix, s.versioner, opts)

    var lastKey []byte
    var hasMore bool

    for {
        // Fetch from etcd with limit
        getResp, err := s.getList(ctx, keyPrefix, opts.Recursive, kubernetes.ListOptions{
            Revision: withRev,
            Limit:    limit,
            Continue: continueKey,
        })

        hasMore = int64(len(getResp.Kvs)) < getResp.Count

        // Process items
        for i, kv := range getResp.Kvs {
            if paging && int64(v.Len()) >= opts.Predicate.Limit {
                hasMore = true
                break
            }

            lastKey = kv.Key

            // Transform and decode
            data, _, err := s.transformer.TransformFromStorage(ctx, kv.Value, ...)
            obj, err := s.decoder.DecodeListItem(ctx, data, uint64(kv.ModRevision), newItemFunc)

            // Apply predicates
            if matched, err := opts.Predicate.Matches(obj); err == nil && matched {
                v.Set(reflect.Append(v, reflect.ValueOf(obj).Elem()))
            }

            getResp.Kvs[i] = nil  // Free memory immediately
        }

        continueKey = string(lastKey) + "\x00"  // Next page starts after this

        if !hasMore || !paging || int64(v.Len()) >= opts.Predicate.Limit {
            break
        }

        // Adaptive limit growth for filtering
        if limit < maxLimit {
            limit *= 2
            if limit > maxLimit {
                limit = maxLimit  // Hard limit: 10,000
            }
        }
    }

    // Prepare continuation token
    continueValue, remainingItemCount, err := storage.PrepareContinueToken(...)
    return s.versioner.UpdateList(listObj, uint64(withRev), continueValue, remainingItemCount)
}
```

### **Pagination Constants**

```go
const maxLimit = 10000  // Maximum items per etcd fetch
```

### **Pagination Performance**

**Scenario: List 50,000 pods in 5000-node cluster**

```
Assumptions:
- Average pod size: 5 KB
- Network bandwidth: 100 MB/s
- maxLimit: 10,000

Calculation:
- Pages needed: 50,000 / 10,000 = 5 requests
- Data transfer: 50,000 × 5 KB = 250 MB
- Transfer time: 250 MB / 100 MB/s = 2.5 seconds

With filtering (e.g., by namespace):
- Much faster if filtering reduces result set
- Adaptive limit growth reduces round-trips
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔑 Lease Management**

### **Lease Manager Configuration**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/lease_manager.go:33-46`

```go
type LeaseManagerConfig struct {
    ReuseDurationSeconds int64  // How long to reuse (default: 60)
    MaxObjectCount       int64  // Max objects per lease (default: 1000)
}

func NewDefaultLeaseManagerConfig() LeaseManagerConfig {
    return LeaseManagerConfig{
        ReuseDurationSeconds: 60,   // 60 seconds
        MaxObjectCount:       1000, // 1000 objects
    }
}
```

### **Lease Manager Implementation**

**File**: `lease_manager.go:54-66`

```go
type leaseManager struct {
    client                  *clientv3.Client
    leaseMu                 sync.Mutex

    prevLeaseID             clientv3.LeaseID     // Cached lease
    prevLeaseExpirationTime time.Time            // Expiration

    leaseReuseDurationSeconds   int64            // Reuse window
    leaseReuseDurationPercent   float64          // 5% of TTL
    leaseMaxAttachedObjectCount int64            // Max objects
    leaseAttachedObjectCount    int64            // Current count
}
```

### **Lease Reuse Algorithm**

**File**: `lease_manager.go:90-120`

```go
func (l *leaseManager) GetLease(ctx context.Context, ttl int64) (clientv3.LeaseID, error) {
    now := time.Now()
    l.leaseMu.Lock()
    defer l.leaseMu.Unlock()

    // Calculate reuse duration
    reuseDurationSeconds := l.getReuseDurationSecondsLocked(ttl)
    // reuseDuration = min(60 seconds, 5% of TTL)

    // Check if can reuse existing lease
    valid := now.Add(time.Duration(ttl) * time.Second).Before(l.prevLeaseExpirationTime)
    sufficient := now.Add(time.Duration(ttl+reuseDurationSeconds) * time.Second).After(l.prevLeaseExpirationTime)

    l.leaseAttachedObjectCount++

    // Reuse if conditions met
    if valid && sufficient && l.leaseAttachedObjectCount <= l.leaseMaxAttachedObjectCount {
        return l.prevLeaseID, nil  // Reuse!
    }

    // Request new lease
    ttl += reuseDurationSeconds
    lcr, err := l.client.Lease.Grant(ctx, ttl)

    // Cache new lease
    l.prevLeaseID = lcr.ID
    l.prevLeaseExpirationTime = now.Add(time.Duration(ttl) * time.Second)
    metrics.UpdateLeaseObjectCount(l.leaseAttachedObjectCount)
    l.leaseAttachedObjectCount = 1

    return lcr.ID, nil
}
```

### **Lease Reuse Impact**

**Example: Creating 50,000 pods with 1-hour TTL**

| Scenario | Lease.Grant Calls | Reduction |
|----------|-------------------|-----------|
| Without reuse | 50,000 | - |
| With reuse (defaults) | ~50 | 99.9% |

**Calculation**:
- Reuse window: min(60s, 5% × 3600s) = 60s
- Max objects per lease: 1000
- Creating 50,000 pods at 1000/sec = 50 seconds
- Leases needed: ceil(50,000 / 1000) = 50

### **Lease Configuration**

**File**: `staging/src/k8s.io/apiserver/pkg/server/options/etcd.go:208-209`

```bash
kube-apiserver \
  --lease-reuse-duration-seconds=60  # Default
```

**Tuning**:
- High churn (>1000 pods/sec): Increase to 120-300s
- Low churn (<100 pods/min): Can reduce to 30s
- Trade-off: Higher reuse = fewer leases but longer lifetime overhead

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔐 Encryption at Rest**

### **Envelope Encryption Architecture**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/envelope/envelope.go`

Kubernetes uses **envelope encryption** (KEK-DEK scheme):
- **KEK** (Key Encryption Key): Stored in external KMS
- **DEK** (Data Encryption Key): Generated per object, encrypted by KEK

### **Envelope Transformer**

**File**: `envelope.go:49-82`

```go
type envelopeTransformer struct {
    envelopeService     Service                               // KMS provider
    transformers        *lru.Cache                            // DEK cache
    baseTransformerFunc func(cipher.Block) value.Transformer  // AES cipher
    cacheSize           int
    cacheEnabled        bool
}
```

### **Encryption Flow**

**File**: `envelope.go:122-150`

```go
func (t *envelopeTransformer) TransformToStorage(ctx context.Context, data []byte, dataCtx value.Context) ([]byte, error) {
    // 1. Generate random 32-byte DEK
    newKey, err := generateKey(32)  // AES-256

    // 2. Encrypt DEK with external KMS (expensive RPC!)
    encKey, err := t.envelopeService.Encrypt(newKey)

    // 3. Cache the DEK for future decryption
    transformer, err := t.addTransformer(encKey, newKey)

    // 4. Encrypt data with DEK (AES-GCM)
    result, err := transformer.TransformToStorage(ctx, data, dataCtx)

    // 5. Prepend encrypted DEK to ciphertext
    // Format: [2-byte length][encrypted DEK][encrypted data]
    b := cryptobyte.NewBuilder(nil)
    b.AddUint16LengthPrefixed(func(b *cryptobyte.Builder) {
        b.AddBytes([]byte(encKey))
    })

    return append(b.BytesOrPanic(), result...), nil
}
```

### **Decryption Flow**

**File**: `envelope.go:84-119`

```go
func (t *envelopeTransformer) TransformFromStorage(ctx context.Context, data []byte, dataCtx value.Context) ([]byte, bool, error) {
    // 1. Extract encrypted DEK
    s := cryptobyte.String(data)
    s.ReadUint16LengthPrefixed(&encKey)
    encData := []byte(s)

    // 2. Check DEK cache (fast path)
    transformer := t.getTransformer(encKey)
    if transformer == nil {
        // Cache miss - decrypt DEK with KMS (slow path)
        key, err := t.envelopeService.Decrypt(encKey)
        transformer, err = t.addTransformer(encKey, key)
    }

    // 3. Decrypt data with DEK
    return transformer.TransformFromStorage(ctx, encData, dataCtx)
}
```

### **KMS v2 Improvements**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/envelope/kmsv2/envelope.go:78-106`

```go
const (
    KMSAPIVersionv2 = "v2"
    cacheTTL        = 24 * time.Hour  // DEK cache for 24 hours
)
```

**KMS v2 State** (`kmsv2/envelope.go:111-128`):

```go
type State struct {
    Transformer                         value.Transformer
    EncryptedObjectKeyID                string
    EncryptedObjectEncryptedDEKSource   []byte
    EncryptedObjectAnnotations          map[string][]byte
    ExpirationTimestamp                 time.Time         // 24-hour cache
    CacheKey                            []byte
    KMSProviderName                     string
}
```

### **Encryption Performance**

| Operation | Cache Hit | Cache Miss |
|-----------|-----------|------------|
| **Encrypt** | ~1ms | 50-200ms (KMS RPC) |
| **Decrypt** | ~0.1ms | 50-200ms (KMS RPC) |

**For 50,000 encrypted pods**:

| Scenario | Read Time | Write Time |
|----------|-----------|------------|
| Without KMS v2 caching | 5,000 seconds | 5,000 seconds |
| With KMS v2 caching (24h) | 0.5 seconds | 0.5 seconds |

**Critical**: KMS v2 with caching is **mandatory** for large encrypted clusters.

### **Configuration**

```bash
kube-apiserver \
  --encryption-provider-config=/etc/kubernetes/encryption-config.yaml \
  --encryption-provider-config-automatic-reload=true  # For key rotation
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏥 Health Checking**

### **Health Check Implementation**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/storagebackend/factory/etcd3.go:113-127`

```go
func newETCD3HealthCheck(c storagebackend.Config, stopCh <-chan struct{}) (func() error, error) {
    timeout := storagebackend.DefaultHealthcheckTimeout  // 2 seconds
    if c.HealthcheckTimeout != time.Duration(0) {
        timeout = c.HealthcheckTimeout
    }
    return newETCD3Check(c, timeout, stopCh)
}
```

### **Health Check Prober**

**File**: `etcd3.go:153-224`

```go
func newETCD3Check(c storagebackend.Config, timeout time.Duration, stopCh <-chan struct{}) (func() error, error) {
    // Rate limit health checks to every timeout/2
    limiter := rate.NewLimiter(rate.Every(timeout/2), 1)

    return func() error {
        if !limiter.Allow() {
            return lastError.Load()  // Return cached error
        }

        ctx, cancel := context.WithTimeout(context.Background(), timeout)
        defer cancel()

        err := prober.Probe(ctx)
        lastError.Store(err, time.Now())
        return err
    }, nil
}
```

### **Probe Operation**

**File**: `etcd3.go:257-269`

```go
func (t *etcd3ProberMonitor) Probe(ctx context.Context) error {
    // Simple GET to health key
    _, err := t.client.Get(ctx, path.Join("/", t.prefix, "health"))
    if err != nil {
        return fmt.Errorf("error getting data from etcd: %w", err)
    }
    return nil
}
```

### **Health Check Scaling**

**With default 2-second timeout**:
- Health check frequency: Every 1 second per API server
- For 5000 nodes × 3 API servers = 15,000 probes/second to etcd

**Tuning for large clusters**:

```bash
kube-apiserver \
  --etcd-healthcheck-timeout=10s \
  --etcd-readycheck-timeout=10s
```

| Timeout | Probe Frequency | Probes/sec (15k API servers) |
|---------|----------------|------------------------------|
| 2s | 1/s | 15,000 |
| 5s | 0.4/s | 6,000 |
| 10s | 0.2/s | 3,000 |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Metrics and Monitoring**

### **Core Etcd Metrics**

**File**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/metrics/metrics.go`

```go
// Request latency
etcd_request_duration_seconds{operation, type}
// Buckets: 0.005, 0.025, 0.05, 0.1, 0.2, 0.4, 0.6, 0.8, 1.0, 1.25, 1.5, 2, 3, 4, 5, 6, 8, 10, 15, 20, 30, 45, 60

// Request counts
etcd_requests_total{operation, type}

// Error counts
etcd_request_errors_total{operation, type}

// Database size
apiserver_storage_db_total_size_in_bytes{endpoint}

// Lease object counts
etcd_lease_object_counts
// Buckets: 10, 50, 100, 500, 1000, 2500, 5000

// List metrics
apiserver_storage_list_total{resource}
apiserver_storage_list_fetched_objects_total{resource}
apiserver_storage_list_returned_objects_total{resource}
```

### **Metrics Recording**

```go
// Record request latency and errors
metrics.RecordEtcdRequest("get", pods, err, startTime)

// Record list efficiency
metrics.RecordStorageListMetrics(groupResource, numFetched, numEvald, numReturn)

// Record lease reuse
metrics.UpdateLeaseObjectCount(count)

// Record database size
metrics.UpdateEtcdDbSize(endpoint, size)
```

### **Key Alerts for Large Clusters**

```yaml
groups:
- name: etcd-scalability
  rules:
  # High etcd latency
  - alert: EtcdHighGetLatency
    expr: histogram_quantile(0.99, rate(etcd_request_duration_seconds_bucket{operation="get"}[5m])) > 0.5
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "etcd GET p99 latency exceeds 500ms"

  # etcd unavailable
  - alert: EtcdDown
    expr: up{job="etcd"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "etcd cluster is down"

  # Database growing too large
  - alert: EtcdDatabaseNearLimit
    expr: apiserver_storage_db_total_size_in_bytes / 2147483648 > 0.8
    for: 30m
    labels:
      severity: warning
    annotations:
      summary: "etcd database exceeds 80% of 2GB limit"

  # Slow LIST operations
  - alert: EtcdSlowList
    expr: histogram_quantile(0.99, rate(etcd_request_duration_seconds_bucket{operation="list"}[5m])) > 5
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "etcd LIST p99 latency exceeds 5 seconds"

  # High write error rate
  - alert: EtcdHighWriteErrors
    expr: rate(etcd_request_errors_total{operation=~"put|delete"}[5m]) > 0.01
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "etcd write error rate exceeds 1%"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 Production Tuning**

### **Tuning Parameters for 5000+ Node Clusters**

| Parameter | Default | Recommended | Rationale |
|-----------|---------|-------------|-----------|
| `--etcd-compaction-interval` | 5m | 15-30m | Reduce compaction frequency |
| `--etcd-db-metric-poll-interval` | 30s | 60s | Reduce metric polling |
| `--etcd-healthcheck-timeout` | 2s | 10s | Reduce health check traffic |
| `--etcd-count-metric-poll-period` | 1m | 5m | Reduce object count polling |
| `--lease-reuse-duration-seconds` | 60s | 120-300s | Reduce lease creation |
| `--default-watch-cache-size` | 100 | 10000+ | Reduce etcd LIST calls |
| `--storage-media-type` | json | protobuf | Smaller storage, faster I/O |

### **Complete API Server Configuration**

```bash
kube-apiserver \
  # Storage backend
  --storage-media-type=application/vnd.kubernetes.protobuf \
  --etcd-servers=https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379 \

  # Compaction and metrics
  --etcd-compaction-interval=15m \
  --etcd-db-metric-poll-interval=60s \
  --etcd-count-metric-poll-period=5m \

  # Health checks
  --etcd-healthcheck-timeout=10s \
  --etcd-readycheck-timeout=10s \

  # Lease management
  --lease-reuse-duration-seconds=180 \

  # Watch cache (reduce etcd load)
  --default-watch-cache-size=1000 \
  --watch-cache-sizes=pods#10000,nodes#5000,events#10000 \

  # Encryption (if used)
  --encryption-provider-config=/etc/kubernetes/encryption-config.yaml \
  --encryption-provider-config-automatic-reload=true
```

### **Etcd Cluster Configuration**

**For dedicated etcd cluster**:

```yaml
# /etc/etcd/etcd.conf.yaml
name: etcd-1
data-dir: /var/lib/etcd

# Performance tuning
heartbeat-interval: 100       # Leader heartbeat (ms)
election-timeout: 1000        # Election timeout (ms)

# Quotas
quota-backend-bytes: 8589934592  # 8 GB (increase from default 2 GB)
auto-compaction-mode: periodic
auto-compaction-retention: "1h"  # Keep 1 hour of history

# Snapshots
snapshot-count: 10000           # Snapshot every 10K transactions
max-snapshots: 5
max-wals: 5

# Client settings
max-request-bytes: 1572864      # 1.5 MB max request
grpc-keepalive-min-time: 5s
grpc-keepalive-timeout: 20s
```

### **Hardware Recommendations**

| Component | Minimum | Recommended (5000+ nodes) |
|-----------|---------|--------------------------|
| **Nodes** | 3 | 5 (better fault tolerance) |
| **CPU** | 4 cores | 8+ cores |
| **Memory** | 8 GB | 32+ GB |
| **Disk** | SSD | NVMe SSD |
| **IOPS** | 3000 | 8000+ |
| **Network** | 1 Gbps | 10 Gbps (dedicated) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting Guide**

### **Common Issues**

| Issue | Symptom | Root Cause | Solution |
|-------|---------|------------|----------|
| **Slow reads** | High GET latency | Disk I/O, network | Use NVMe SSD, dedicated network |
| **Watch fails** | "revision compacted" | Compaction too aggressive | Increase compaction interval |
| **Database full** | "database space exceeded" | Too much data | Increase quota, clean old data |
| **Split brain** | Multiple leaders | Network partition | Fix network, check quorum |
| **Slow writes** | High PUT latency | Slow followers | Check disk/network on followers |

### **Diagnostic Commands**

```bash
# Check etcd cluster health
etcdctl endpoint health --cluster

# Check database size
etcdctl endpoint status --cluster -w table

# Check compaction status
etcdctl alarm list

# Defragment (run on each member)
etcdctl defrag --endpoints=https://etcd-1:2379

# Check key count
etcdctl get "" --prefix --keys-only | wc -l

# Watch compaction
etcdctl watch --prefix /registry --rev=1

# Check revision
etcdctl endpoint status -w fields | grep Revision
```

### **Investigation Flow**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Etcd Performance Investigation                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Check cluster health                                                 │
│     └─▶ etcdctl endpoint health --cluster                               │
│     └─▶ All members should be healthy                                   │
│                                                                          │
│  2. Check database size                                                  │
│     └─▶ etcdctl endpoint status -w table                                │
│     └─▶ Size < quota (default 2GB, recommended 8GB for large clusters)  │
│                                                                          │
│  3. Check latency metrics                                                │
│     └─▶ etcd_disk_wal_fsync_duration_seconds                            │
│     └─▶ etcd_disk_backend_commit_duration_seconds                       │
│     └─▶ Should be <10ms for SSD                                         │
│                                                                          │
│  4. Check network between members                                        │
│     └─▶ etcd_network_peer_round_trip_time_seconds                       │
│     └─▶ Should be <10ms for same datacenter                             │
│                                                                          │
│  5. Check compaction                                                     │
│     └─▶ etcd_debugging_mvcc_db_compaction_keys_total                    │
│     └─▶ Compaction running too frequently?                              │
│                                                                          │
│  6. Check for hot keys                                                   │
│     └─▶ High write rate on specific keys                                │
│     └─▶ Leader election, ConfigMaps, etc.                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💾 Disaster Recovery**

### **Backup Strategy**

**Snapshot backup**:

```bash
# Create snapshot
etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://etcd-1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
etcdctl snapshot status /backup/etcd-*.db -w table
```

**Backup frequency**:
- Critical clusters: Every 15-30 minutes
- Production clusters: Every 1 hour
- Development clusters: Every 6-24 hours

### **Restore Process**

```bash
# Stop all API servers first!

# Restore on each etcd member
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name=etcd-1 \
  --initial-cluster=etcd-1=https://etcd-1:2380,etcd-2=https://etcd-2:2380,etcd-3=https://etcd-3:2380 \
  --initial-advertise-peer-urls=https://etcd-1:2380 \
  --data-dir=/var/lib/etcd-restored

# Update data-dir in etcd configuration
# Start etcd cluster
# Verify health
# Start API servers
```

### **Point-in-Time Recovery**

Using etcd revision:

```bash
# Find revision at specific time (from metrics/logs)
REVISION=12345

# Read data at that revision
etcdctl get "" --prefix --rev=$REVISION | ...
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Related Documentation**

### **Kubernetes Source Files**

| Component | File Path |
|-----------|-----------|
| Config | `staging/src/k8s.io/apiserver/pkg/storage/storagebackend/config.go` |
| Factory | `staging/src/k8s.io/apiserver/pkg/storage/storagebackend/factory/etcd3.go` |
| Store | `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go` |
| Watcher | `staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go` |
| Compaction | `staging/src/k8s.io/apiserver/pkg/storage/etcd3/compact.go` |
| Leases | `staging/src/k8s.io/apiserver/pkg/storage/etcd3/lease_manager.go` |
| Pagination | `staging/src/k8s.io/apiserver/pkg/storage/continue.go` |
| Encryption | `staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/envelope/envelope.go` |
| KMS v2 | `staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/envelope/kmsv2/envelope.go` |
| Options | `staging/src/k8s.io/apiserver/pkg/server/options/etcd.go` |
| Metrics | `staging/src/k8s.io/apiserver/pkg/storage/etcd3/metrics/metrics.go` |

### **Related Architecture Docs**

- `scalability/01-large-cluster-architecture.md` - Overall scalability patterns
- `scalability/04-disaster-recovery-strategies.md` - Backup and recovery
- `scalability/06-component-optimization.md` - Component tuning
- `advanced-topics/02-api-server-scalability.md` - API server optimization

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **✨ Key Takeaways**

1. **Lease Reuse**: Critical for write performance - reduces etcd RPCs by 99%+
2. **Compaction**: Distributed algorithm prevents duplicates; tune interval for scale
3. **KMS v2**: Mandatory for encrypted clusters - 24-hour DEK caching
4. **Watch Buffers**: Fixed size (100) - high churn may cause drops
5. **Health Checks**: Tune timeout to reduce etcd load at scale
6. **Protobuf Storage**: 30-50% smaller, faster I/O than JSON
7. **Pagination**: Adaptive limit growth reduces round-trips for filtered lists
8. **NVMe Required**: Disk I/O is primary bottleneck for large clusters

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Last Updated**: 2024-11-19
**Kubernetes Version**: 1.31+
