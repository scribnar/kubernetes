# Controller Patterns and Best Practices Reference

## Overview

This document consolidates common controller design patterns, best practices, error handling strategies, rate limiting, performance optimization, testing approaches, and debugging techniques used throughout kube-controller-manager.

## Common Controller Patterns

### 1. Controller Loop Pattern

**The Standard Reconciliation Loop:**

```go
func (c *Controller) Run(ctx context.Context, workers int) {
    defer utilruntime.HandleCrash()
    defer c.queue.ShutDown()

    // Start informers
    if !cache.WaitForCacheSync(ctx.Done(), c.synced) {
        return
    }

    // Start workers
    for i := 0; i < workers; i++ {
        go wait.UntilWithContext(ctx, c.worker, time.Second)
    }

    <-ctx.Done()
}

func (c *Controller) worker(ctx context.Context) {
    for c.processNextWorkItem(ctx) {
    }
}

func (c *Controller) processNextWorkItem(ctx context.Context) bool {
    key, shutdown := c.queue.Get()
    if shutdown {
        return false
    }
    defer c.queue.Done(key)

    err := c.syncHandler(ctx, key.(string))
    c.handleErr(err, key)
    return true
}
```

**Key Elements:**
- Workqueue for decoupling watch from processing
- Multiple workers for parallelism
- Graceful shutdown handling
- Error handling and requeuing

### 2. Informer Pattern

**Efficient Watching and Caching:**

```go
func (c *Controller) setupInformers(
    informerFactory informers.SharedInformerFactory,
) {
    // Pod informer
    podInformer := informerFactory.Core().V1().Pods()
    c.podLister = podInformer.Lister()
    c.podSynced = podInformer.Informer().HasSynced

    // Event handlers
    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: c.addPod,
        UpdateFunc: c.updatePod,
        DeleteFunc: c.deletePod,
    })
}

func (c *Controller) addPod(obj interface{}) {
    pod := obj.(*v1.Pod)
    c.queue.Add(keyFunc(pod))
}
```

**Benefits:**
- Reduces API server load
- Local cache for reads
- Efficient updates via watches

### 3. Owner References Pattern

**Automatic Cleanup:**

```go
func setOwnerReference(child, owner metav1.Object) {
    ownerRef := metav1.NewControllerRef(
        owner,
        schema.GroupVersionKind{
            Group:   "apps",
            Version: "v1",
            Kind:    "ReplicaSet",
        },
    )

    child.SetOwnerReferences([]metav1.OwnerReference{*ownerRef})
}
```

**Use Cases:**
- Automatic garbage collection
- Relationship tracking
- Orphan prevention

### 4. Expectations Pattern

**Avoiding Race Conditions:**

```go
type ControllerExpectations interface {
    ExpectCreations(key string, adds int)
    ExpectDeletions(key string, dels int)
    CreationObserved(key string)
    DeletionObserved(key string)
    SatisfiedExpectations(key string) bool
}

// Before creating pods
c.expectations.ExpectCreations(rsKey, diff)

// Create pods...

// In update handler
if c.expectations.SatisfiedExpectations(rsKey) {
    // Process normally
}
```

**Purpose:**
- Prevent processing stale data
- Coordinate create/delete operations
- Reduce unnecessary reconciliation

### 5. Finalizer Pattern

**Controlled Deletion:**

```go
func (c *Controller) addFinalizer(obj metav1.Object) error {
    finalizers := obj.GetFinalizers()
    for _, f := range finalizers {
        if f == myFinalizer {
            return nil // Already has finalizer
        }
    }

    obj.SetFinalizers(append(finalizers, myFinalizer))
    return c.client.Update(ctx, obj)
}

func (c *Controller) handleDeletion(obj metav1.Object) error {
    if obj.GetDeletionTimestamp() == nil {
        return nil // Not being deleted
    }

    // Perform cleanup
    if err := c.cleanup(obj); err != nil {
        return err
    }

    // Remove finalizer
    return c.removeFinalizer(obj)
}
```

**Examples:**
- PV/PVC protection
- Job tracking
- Namespace cleanup

## Error Handling Strategies

### 1. Exponential Backoff

```go
// Rate-limited queue
queue := workqueue.NewRateLimitingQueue(
    workqueue.NewItemExponentialFailureRateLimiter(
        5*time.Millisecond,  // Base delay
        1000*time.Second,    // Max delay
    ),
)

func (c *Controller) handleErr(err error, key interface{}) {
    if err == nil {
        c.queue.Forget(key)
        return
    }

    if c.queue.NumRequeues(key) < 5 {
        klog.Errorf("Error syncing %v: %v", key, err)
        c.queue.AddRateLimited(key)
        return
    }

    c.queue.Forget(key)
    klog.Errorf("Dropping %v out of queue: %v", key, err)
}
```

### 2. Categorized Error Handling

```go
func (c *Controller) sync(ctx context.Context, key string) error {
    obj, err := c.lister.Get(key)
    if err != nil {
        if errors.IsNotFound(err) {
            // Object deleted, no retry needed
            return nil
        }
        // Transient error, will retry
        return err
    }

    if err := c.process(obj); err != nil {
        if isConflict(err) {
            // Optimistic lock failure, immediate retry
            return err
        }
        if isPermanentError(err) {
            // Log and don't retry
            klog.Errorf("Permanent error: %v", err)
            return nil
        }
        // Transient error, retry with backoff
        return err
    }

    return nil
}
```

### 3. Circuit Breaker Pattern

```go
type CircuitBreaker struct {
    failures    int
    lastFailure time.Time
    threshold   int
    timeout     time.Duration
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    if cb.isOpen() {
        return fmt.Errorf("circuit breaker open")
    }

    err := fn()
    if err != nil {
        cb.recordFailure()
    } else {
        cb.reset()
    }

    return err
}

func (cb *CircuitBreaker) isOpen() bool {
    if cb.failures < cb.threshold {
        return false
    }
    return time.Since(cb.lastFailure) < cb.timeout
}
```

## Rate Limiting Patterns

### 1. Token Bucket Rate Limiter

```go
import "golang.org/x/time/rate"

// Create rate limiter: 10 QPS, burst of 20
limiter := rate.NewLimiter(10, 20)

func (c *Controller) processItem(item interface{}) error {
    // Wait for token
    if err := limiter.Wait(context.Background()); err != nil {
        return err
    }

    return c.doWork(item)
}
```

### 2. Per-Item Rate Limiting

```go
type ItemRateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.Mutex
}

func (irl *ItemRateLimiter) Wait(key string) error {
    irl.mu.Lock()
    limiter, exists := irl.limiters[key]
    if !exists {
        limiter = rate.NewLimiter(1, 5) // 1 QPS, burst 5
        irl.limiters[key] = limiter
    }
    irl.mu.Unlock()

    return limiter.Wait(context.Background())
}
```

### 3. Workqueue Rate Limiting

```go
// Max exponential rate limiter
queue := workqueue.NewRateLimitingQueue(
    workqueue.NewMaxOfRateLimiter(
        // Exponential backoff
        workqueue.NewItemExponentialFailureRateLimiter(
            5*time.Millisecond,
            1000*time.Second,
        ),
        // Overall rate limit: 10 QPS, burst 100
        &workqueue.BucketRateLimiter{
            Limiter: rate.NewLimiter(rate.Limit(10), 100),
        },
    ),
)
```

## Performance Optimization

### 1. Efficient List/Watch

```go
// Use field selectors to reduce data
listOptions := metav1.ListOptions{
    FieldSelector: fields.OneTermEqualSelector(
        "spec.nodeName",
        nodeName,
    ).String(),
}

// Use label selectors
listOptions := metav1.ListOptions{
    LabelSelector: labels.SelectorFromSet(labels.Set{
        "app": "nginx",
    }).String(),
}
```

### 2. Batching Operations

```go
func (c *Controller) batchUpdate(items []Item) error {
    const batchSize = 100

    for i := 0; i < len(items); i += batchSize {
        end := i + batchSize
        if end > len(items) {
            end = len(items)
        }

        batch := items[i:end]
        if err := c.updateBatch(batch); err != nil {
            return err
        }
    }

    return nil
}
```

### 3. Concurrent Processing

```go
func (c *Controller) processParallel(items []Item) error {
    errCh := make(chan error, len(items))
    sem := make(chan struct{}, 10) // Limit concurrency

    for _, item := range items {
        item := item // Capture loop variable
        go func() {
            sem <- struct{}{}        // Acquire
            defer func() { <-sem }() // Release

            errCh <- c.process(item)
        }()
    }

    // Collect errors
    var errs []error
    for range items {
        if err := <-errCh; err != nil {
            errs = append(errs, err)
        }
    }

    return utilerrors.NewAggregate(errs)
}
```

### 4. Caching Strategies

```go
type CachedController struct {
    cache *lru.Cache
    ttl   time.Duration
}

func (cc *CachedController) Get(key string) (interface{}, error) {
    // Check cache
    if val, ok := cc.cache.Get(key); ok {
        return val, nil
    }

    // Fetch from API
    val, err := cc.fetchFromAPI(key)
    if err != nil {
        return nil, err
    }

    // Store in cache
    cc.cache.Add(key, val)
    return val, nil
}
```

## Testing Strategies

### 1. Unit Testing Controllers

```go
func TestControllerSync(t *testing.T) {
    // Create fake client
    client := fake.NewSimpleClientset()

    // Create informers
    informerFactory := informers.NewSharedInformerFactory(client, 0)

    // Create controller
    controller := NewController(client, informerFactory)

    // Add test objects
    rs := &appsv1.ReplicaSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-rs",
            Namespace: "default",
        },
        Spec: appsv1.ReplicaSetSpec{
            Replicas: pointer.Int32(3),
        },
    }
    client.AppsV1().ReplicaSets("default").Create(
        context.TODO(),
        rs,
        metav1.CreateOptions{},
    )

    // Start informers
    stopCh := make(chan struct{})
    defer close(stopCh)
    informerFactory.Start(stopCh)
    informerFactory.WaitForCacheSync(stopCh)

    // Test sync
    err := controller.syncHandler(context.TODO(), "default/test-rs")
    if err != nil {
        t.Errorf("sync failed: %v", err)
    }

    // Verify results
    pods, _ := client.CoreV1().Pods("default").List(
        context.TODO(),
        metav1.ListOptions{},
    )
    if len(pods.Items) != 3 {
        t.Errorf("expected 3 pods, got %d", len(pods.Items))
    }
}
```

### 2. Integration Testing

```go
func TestControllerIntegration(t *testing.T) {
    // Start test API server
    server := kubeapiservertesting.StartTestServerOrDie(
        t,
        nil,
        []string{"--disable-admission-plugins=ServiceAccount"},
        framework.SharedEtcd(),
    )
    defer server.TearDownFn()

    // Create client
    client := kubernetes.NewForConfigOrDie(server.ClientConfig)

    // Run test
    // ...
}
```

### 3. Table-Driven Tests

```go
func TestPodSync(t *testing.T) {
    tests := []struct {
        name        string
        pod         *v1.Pod
        expectError bool
        expectPhase v1.PodPhase
    }{
        {
            name: "running pod",
            pod: &v1.Pod{
                Status: v1.PodStatus{Phase: v1.PodRunning},
            },
            expectError: false,
            expectPhase: v1.PodRunning,
        },
        {
            name: "failed pod",
            pod: &v1.Pod{
                Status: v1.PodStatus{Phase: v1.PodFailed},
            },
            expectError: false,
            expectPhase: v1.PodFailed,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := syncPod(tt.pod)
            if (err != nil) != tt.expectError {
                t.Errorf("unexpected error: %v", err)
            }
            if tt.pod.Status.Phase != tt.expectPhase {
                t.Errorf("expected phase %v, got %v",
                    tt.expectPhase, tt.pod.Status.Phase)
            }
        })
    }
}
```

## Debugging Techniques

### 1. Structured Logging

```go
import "k8s.io/klog/v2"

func (c *Controller) sync(ctx context.Context, key string) error {
    klog.V(4).InfoS(
        "Starting sync",
        "controller", "replicaset",
        "key", key,
    )

    obj, err := c.lister.Get(key)
    if err != nil {
        klog.ErrorS(err, "Failed to get object", "key", key)
        return err
    }

    klog.V(5).InfoS(
        "Processing object",
        "name", obj.Name,
        "namespace", obj.Namespace,
        "replicas", *obj.Spec.Replicas,
    )

    return nil
}
```

**Log Levels:**
- Level 0: Errors only
- Level 2: Useful steady state
- Level 4: Debug-level verbosity
- Level 6: Display requested resources
- Level 8: Trace-level verbosity

### 2. Metrics for Debugging

```go
var (
    syncDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "controller_sync_duration_seconds",
            Help: "Time taken to sync",
        },
        []string{"controller", "result"},
    )
)

func (c *Controller) sync(ctx context.Context, key string) error {
    start := time.Now()
    defer func() {
        result := "success"
        if err != nil {
            result = "error"
        }
        syncDuration.WithLabelValues(
            "replicaset",
            result,
        ).Observe(time.Since(start).Seconds())
    }()

    // Sync logic
    return c.doSync(ctx, key)
}
```

### 3. Debug Endpoints

```go
// Add debug handler
mux.HandleFunc("/debug/queue", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "depth":    queue.Len(),
        "requeues": getRequeueStats(),
    })
})
```

### 4. Event Recording for Debugging

```go
func (c *Controller) sync(ctx context.Context, key string) error {
    obj, err := c.lister.Get(key)
    if err != nil {
        return err
    }

    // Record event for debugging
    c.recorder.Eventf(
        obj,
        v1.EventTypeNormal,
        "Syncing",
        "Started sync at %v",
        time.Now(),
    )

    defer func() {
        c.recorder.Event(
            obj,
            v1.EventTypeNormal,
            "SyncComplete",
            "Sync completed successfully",
        )
    }()

    return c.doSync(obj)
}
```

## Common Anti-Patterns to Avoid

### 1. Don't Use List Instead of Watch

```go
// BAD: Polling
func (c *Controller) pollList() {
    for {
        list, _ := c.client.CoreV1().Pods("").List(ctx, metav1.ListOptions{})
        c.process(list)
        time.Sleep(10 * time.Second)
    }
}

// GOOD: Watch with informers
func (c *Controller) watch() {
    informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    c.addPod,
        UpdateFunc: c.updatePod,
        DeleteFunc: c.deletePod,
    })
}
```

### 2. Don't Ignore Errors

```go
// BAD
_ = c.updateStatus(obj)

// GOOD
if err := c.updateStatus(obj); err != nil {
    klog.ErrorS(err, "Failed to update status")
    return err
}
```

### 3. Don't Block in Event Handlers

```go
// BAD: Blocking operation in handler
func (c *Controller) addPod(obj interface{}) {
    pod := obj.(*v1.Pod)
    c.expensiveOperation(pod) // Blocks informer
}

// GOOD: Enqueue for async processing
func (c *Controller) addPod(obj interface{}) {
    pod := obj.(*v1.Pod)
    key, _ := cache.MetaNamespaceKeyFunc(pod)
    c.queue.Add(key)
}
```

### 4. Don't Modify Cached Objects

```go
// BAD: Mutating cached object
func (c *Controller) sync(key string) error {
    pod, _ := c.podLister.Pods("default").Get("my-pod")
    pod.Labels["foo"] = "bar" // Modifies cache!
    return c.client.Update(ctx, pod)
}

// GOOD: Deep copy first
func (c *Controller) sync(key string) error {
    pod, _ := c.podLister.Pods("default").Get("my-pod")
    pod = pod.DeepCopy()
    pod.Labels["foo"] = "bar"
    return c.client.Update(ctx, pod)
}
```

## References

- **client-go**: `k8s.io/client-go`
- **Workqueue**: `k8s.io/client-go/util/workqueue`
- **Controller Patterns**: `k8s.io/kubernetes/pkg/controller`
- **Testing**: `k8s.io/client-go/testing`
- **Metrics**: `k8s.io/component-base/metrics`
