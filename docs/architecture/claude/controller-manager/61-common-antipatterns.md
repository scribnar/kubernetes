# Common Antipatterns in Kubernetes Controllers

**Document**: 61-common-antipatterns.md
**Status**: Course Module - Best Practices & Code Quality
**Audience**: Software Engineers, Code Reviewers, Controller Developers
**Prerequisites**: Controller patterns, Go programming, Kubernetes basics

---

## **Overview**

Learning what NOT to do is as important as learning best practices. This document catalogs common anti-patterns in controller development, explains why they're problematic, and provides correct implementations.

### **Learning Objectives**

After studying this document, you will understand:
1. Common controller anti-patterns and their consequences
2. Why certain approaches fail at scale
3. How to refactor anti-patterns into proper patterns
4. Code review red flags to watch for
5. Production pitfalls to avoid
6. Real examples from Kubernetes issues and PRs

---

## **1. Workqueue Anti-Patterns**

### **❌ Anti-Pattern 1.1: Storing Objects Instead of Keys**

```go
// BAD: Storing entire objects in workqueue
type BadController struct {
    queue workqueue.RateLimitingInterface // Queue of *v1.Pod
}

func (c *BadController) enqueueObject(pod *v1.Pod) {
    c.queue.Add(pod) // Storing entire object!
}

func (c *BadController) worker() {
    for {
        obj, shutdown := c.queue.Get()
        if shutdown {
            return
        }

        pod := obj.(*v1.Pod)  // Object might be stale!
        c.syncPod(pod)
        c.queue.Done(obj)
    }
}
```

**Problems**:
1. **Stale data**: Object in queue may be outdated
2. **Memory waste**: Storing full objects vs. small strings
3. **Race conditions**: Object might be modified while in queue
4. **Inconsistent state**: Object doesn't reflect latest API server state

**✅ GOOD: Store Keys, Fetch Fresh Objects**

```go
// GOOD: Store keys, fetch fresh objects
type GoodController struct {
    queue      workqueue.RateLimitingInterface // Queue of string keys
    podLister  corelisters.PodLister
}

func (c *GoodController) enqueueObject(pod *v1.Pod) {
    key, err := cache.MetaNamespaceKeyFunc(pod)
    if err != nil {
        utilruntime.HandleError(err)
        return
    }
    c.queue.Add(key) // Store only the key!
}

func (c *GoodController) worker() {
    for {
        obj, shutdown := c.queue.Get()
        if shutdown {
            return
        }

        key := obj.(string)

        // Fetch fresh object from cache
        namespace, name, err := cache.SplitMetaNamespaceKey(key)
        if err != nil {
            c.queue.Done(obj)
            continue
        }

        pod, err := c.podLister.Pods(namespace).Get(name)
        if err != nil {
            if !apierrors.IsNotFound(err) {
                c.queue.AddRateLimited(key)
            }
            c.queue.Done(obj)
            continue
        }

        c.syncPod(pod) // Always fresh from cache
        c.queue.Done(obj)
    }
}
```

---

### **❌ Anti-Pattern 1.2: Not Calling queue.Done()**

```go
// BAD: Forgetting to call Done()
func (c *BadController) worker() {
    for {
        obj, shutdown := c.queue.Get()
        if shutdown {
            return
        }

        c.processItem(obj)

        // Missing: c.queue.Done(obj)
        // Queue thinks item is still being processed!
    }
}
```

**Problems**:
1. **Queue parallelism broken**: Queue won't process other items
2. **Memory leak**: Items accumulate in "in-progress" state
3. **Metrics incorrect**: Queue depth metrics are wrong

**✅ GOOD: Always Call Done() with Defer**

```go
// GOOD: Use defer to ensure Done() is called
func (c *GoodController) worker() {
    for {
        obj, shutdown := c.queue.Get()
        if shutdown {
            return
        }

        func() {
            defer c.queue.Done(obj) // Always called, even on panic

            if err := c.processItem(obj); err != nil {
                c.queue.AddRateLimited(obj)
            } else {
                c.queue.Forget(obj)
            }
        }()
    }
}
```

---

## **2. Concurrency Anti-Patterns**

### **❌ Anti-Pattern 2.1: Direct API Calls Instead of Informers**

```go
// BAD: Calling API server directly in hot path
func (c *BadController) syncDeployment(key string) error {
    namespace, name, _ := cache.SplitMetaNamespaceKey(key)

    // API call on every sync - kills API server at scale!
    deployment, err := c.client.AppsV1().Deployments(namespace).Get(
        context.TODO(),
        name,
        metav1.GetOptions{},
    )
    if err != nil {
        return err
    }

    // Another API call
    rsList, err := c.client.AppsV1().ReplicaSets(namespace).List(
        context.TODO(),
        metav1.ListOptions{},
    )
    if err != nil {
        return err
    }

    return c.sync(deployment, rsList.Items)
}
```

**Problems**:
1. **API server overload**: Thousands of unnecessary API calls
2. **High latency**: Network round-trips for every sync
3. **No caching**: Defeats purpose of informer architecture
4. **Scalability failure**: Doesn't work with large clusters

**✅ GOOD: Use Informer Cache**

```go
// GOOD: Read from local cache
func (c *GoodController) syncDeployment(key string) error {
    namespace, name, _ := cache.SplitMetaNamespaceKey(key)

    // Read from cache - no API call!
    deployment, err := c.deploymentLister.Deployments(namespace).Get(name)
    if err != nil {
        if apierrors.IsNotFound(err) {
            // Object deleted - no need to sync
            return nil
        }
        return err
    }

    // List from cache with selector
    selector, _ := metav1.LabelSelectorAsSelector(deployment.Spec.Selector)
    rsList, err := c.replicaSetLister.ReplicaSets(namespace).List(selector)
    if err != nil {
        return err
    }

    return c.sync(deployment, rsList)
}
```

---

### **❌ Anti-Pattern 2.2: Not Using Worker Pool**

```go
// BAD: Single-threaded processing
func (c *BadController) Run(stopCh <-chan struct{}) {
    defer c.queue.ShutDown()

    klog.Info("Starting controller")

    // Single worker - bottleneck!
    go wait.Until(c.worker, time.Second, stopCh)

    <-stopCh
}
```

**Problems**:
1. **Low throughput**: Processes one item at a time
2. **Can't scale**: More resources don't help
3. **Wasted CPU**: Single goroutine on multi-core machine

**✅ GOOD: Use Worker Pool**

```go
// GOOD: Multiple concurrent workers
func (c *GoodController) Run(workers int, stopCh <-chan struct{}) {
    defer c.queue.ShutDown()

    klog.Info("Starting controller")

    // Start multiple workers
    for i := 0; i < workers; i++ {
        go wait.Until(c.worker, time.Second, stopCh)
    }

    <-stopCh
    klog.Info("Shutting down controller")
}

// Typical usage: Run(5, stopCh) for 5 parallel workers
```

---

## **3. Resource Management Anti-Patterns**

### **❌ Anti-Pattern 3.1: Unnecessary Deep Copies**

```go
// BAD: Copying objects unnecessarily
func (c *BadController) processObject(obj interface{}) error {
    // Deep copy for read-only operation - wasteful!
    pod := obj.(*v1.Pod).DeepCopy()

    klog.Infof("Processing pod: %s", pod.Name)

    // Another unnecessary copy
    podCopy2 := pod.DeepCopy()

    // Yet another copy
    podCopy3 := podCopy2.DeepCopy()

    return c.validatePod(podCopy3)
}
```

**Problems**:
1. **CPU waste**: DeepCopy is expensive
2. **Memory waste**: Multiple copies of same object
3. **GC pressure**: Frequent allocations trigger GC

**Benchmark**:
```
BenchmarkWithCopies:    500 ns/op  800 B/op  15 allocs/op
BenchmarkWithoutCopies: 2 ns/op    0 B/op    0 allocs/op
```

**✅ GOOD: Copy Only When Modifying**

```go
// GOOD: Copy only when necessary
func (c *GoodController) processObject(obj interface{}) error {
    // Direct reference for read-only
    pod := obj.(*v1.Pod)

    klog.Infof("Processing pod: %s", pod.Name)

    // Only copy if we're going to modify
    if c.needsUpdate(pod) {
        podCopy := pod.DeepCopy()
        podCopy.Labels["processed"] = "true"

        _, err := c.client.CoreV1().Pods(pod.Namespace).Update(
            context.TODO(),
            podCopy,
            metav1.UpdateOptions{},
        )
        return err
    }

    return nil
}
```

---

### **❌ Anti-Pattern 3.2: Ignoring Resource Versions**

```go
// BAD: Not checking resource version
func (c *BadController) updatePodStatus(pod *v1.Pod) error {
    pod.Status.Phase = v1.PodRunning

    // Blind update - will conflict if pod was modified
    _, err := c.client.CoreV1().Pods(pod.Namespace).UpdateStatus(
        context.TODO(),
        pod,
        metav1.UpdateOptions{},
    )

    return err // Conflict error!
}
```

**Problems**:
1. **Update conflicts**: Fails if resource was modified
2. **Lost updates**: Overwrites other controller's changes
3. **Retry storms**: Repeated conflicts cause retries

**✅ GOOD: Retry on Conflict with Fresh Fetch**

```go
// GOOD: Retry with optimistic locking
func (c *GoodController) updatePodStatus(pod *v1.Pod) error {
    return retry.RetryOnConflict(retry.DefaultBackoff, func() error {
        // Fetch latest version
        latest, err := c.client.CoreV1().Pods(pod.Namespace).Get(
            context.TODO(),
            pod.Name,
            metav1.GetOptions{},
        )
        if err != nil {
            return err
        }

        // Apply status change to latest version
        latest.Status.Phase = v1.PodRunning

        // Update with correct resource version
        _, err = c.client.CoreV1().Pods(pod.Namespace).UpdateStatus(
            context.TODO(),
            latest,
            metav1.UpdateOptions{},
        )

        return err
    })
}
```

---

## **4. Error Handling Anti-Patterns**

### **❌ Anti-Pattern 4.1: Swallowing Errors**

```go
// BAD: Silently ignoring errors
func (c *BadController) syncPod(key string) error {
    pod, err := c.getPod(key)
    if err != nil {
        // Silently ignored - no logging, no return!
        return nil
    }

    err = c.processPod(pod)
    if err != nil {
        // Also ignored!
        return nil
    }

    return nil
}
```

**Problems**:
1. **Hidden failures**: Errors go unnoticed
2. **No debugging info**: Can't troubleshoot
3. **Data loss**: Failed operations not retried
4. **Incorrect metrics**: Success rate appears 100%

**✅ GOOD: Log and Propagate Errors**

```go
// GOOD: Proper error handling
func (c *GoodController) syncPod(key string) error {
    pod, err := c.getPod(key)
    if err != nil {
        if apierrors.IsNotFound(err) {
            // Expected error - log and return nil
            klog.V(4).InfoS("Pod not found, assuming deleted", "key", key)
            return nil
        }
        // Unexpected error - log and return for retry
        klog.ErrorS(err, "Failed to get pod", "key", key)
        return fmt.Errorf("failed to get pod %s: %w", key, err)
    }

    if err := c.processPod(pod); err != nil {
        klog.ErrorS(err, "Failed to process pod",
            "pod", pod.Name,
            "namespace", pod.Namespace)
        return fmt.Errorf("failed to process pod: %w", err)
    }

    return nil
}
```

---

### **❌ Anti-Pattern 4.2: Retrying Permanent Errors**

```go
// BAD: Retrying errors that will never succeed
func (c *BadController) handleError(err error, key interface{}) {
    if err != nil {
        // Always retry - even for permanent errors!
        c.queue.AddRateLimited(key)

        klog.Errorf("Error processing %v: %v", key, err)
    } else {
        c.queue.Forget(key)
    }
}
```

**Problems**:
1. **Wasted resources**: Retrying impossible operations
2. **Queue pollution**: Permanent failures never leave queue
3. **Memory leak**: Retry count and backoff state accumulate
4. **Misleading metrics**: High retry counts

**✅ GOOD: Classify Errors, Don't Retry Permanent Ones**

```go
// GOOD: Check if error is retryable
func (c *GoodController) handleError(err error, key interface{}) {
    if err == nil {
        c.queue.Forget(key)
        return
    }

    // Don't retry permanent errors
    if apierrors.IsNotFound(err) {
        klog.V(4).InfoS("Resource not found, dropping", "key", key)
        c.queue.Forget(key)
        return
    }

    if apierrors.IsInvalid(err) {
        klog.ErrorS(err, "Invalid resource, dropping", "key", key)
        c.queue.Forget(key)
        utilruntime.HandleError(err)
        return
    }

    if apierrors.IsForbidden(err) {
        klog.ErrorS(err, "Forbidden, dropping", "key", key)
        c.queue.Forget(key)
        utilruntime.HandleError(err)
        return
    }

    // Transient errors - retry with backoff
    if c.queue.NumRequeues(key) < 15 {
        klog.V(2).InfoS("Error syncing, retrying",
            "key", key,
            "error", err,
            "retries", c.queue.NumRequeues(key))
        c.queue.AddRateLimited(key)
        return
    }

    // Max retries exceeded
    klog.ErrorS(err, "Dropping after max retries",
        "key", key,
        "retries", c.queue.NumRequeues(key))
    c.queue.Forget(key)
    utilruntime.HandleError(err)
}
```

---

## **5. Reconciliation Loop Anti-Patterns**

### **❌ Anti-Pattern 5.1: Status Updates Trigger Reconciliation**

```go
// BAD: Updating spec and status together
func (c *BadController) syncDeployment(deployment *apps.Deployment) error {
    // Modify both spec and status
    deployment.Spec.Replicas = pointer.Int32(10)
    deployment.Status.Replicas = 10 // This triggers another sync!

    _, err := c.client.AppsV1().Deployments(deployment.Namespace).Update(
        context.TODO(),
        deployment,
        metav1.UpdateOptions{},
    )

    return err
}
```

**Problems**:
1. **Infinite loops**: Status update triggers new sync
2. **High CPU usage**: Continuous reconciliation
3. **API server load**: Excessive watch events
4. **Race conditions**: Spec and status out of sync

**✅ GOOD: Separate Spec and Status Updates**

```go
// GOOD: Update spec and status separately
func (c *GoodController) syncDeployment(deployment *apps.Deployment) error {
    // Update spec only if needed
    if *deployment.Spec.Replicas != 10 {
        deploymentCopy := deployment.DeepCopy()
        deploymentCopy.Spec.Replicas = pointer.Int32(10)

        _, err := c.client.AppsV1().Deployments(deployment.Namespace).Update(
            context.TODO(),
            deploymentCopy,
            metav1.UpdateOptions{},
        )
        if err != nil {
            return err
        }
    }

    // Update status separately, check generation to prevent loops
    if deployment.Status.ObservedGeneration < deployment.Generation ||
       deployment.Status.Replicas != 10 {

        deploymentCopy := deployment.DeepCopy()
        deploymentCopy.Status.Replicas = 10
        deploymentCopy.Status.ObservedGeneration = deployment.Generation

        _, err := c.client.AppsV1().Deployments(deployment.Namespace).UpdateStatus(
            context.TODO(),
            deploymentCopy,
            metav1.UpdateOptions{},
        )
        if err != nil {
            return err
        }
    }

    return nil
}
```

---

### **❌ Anti-Pattern 5.2: Not Using Expectations**

```go
// BAD: Creating pods without expectation tracking
func (c *BadController) scalePods(rs *apps.ReplicaSet, diff int) error {
    for i := 0; i < diff; i++ {
        pod := newPod(rs, i)
        _, err := c.client.CoreV1().Pods(rs.Namespace).Create(
            context.TODO(),
            pod,
            metav1.CreateOptions{},
        )
        if err != nil {
            return err
        }
    }

    // Problem: Next sync might see pods not yet in cache
    // and create duplicates!

    return nil
}
```

**Problems**:
1. **Duplicate creates**: Creates pods that are already being created
2. **Race conditions**: Cache lag causes over-provisioning
3. **Wasted resources**: Extra pods created then deleted

**✅ GOOD: Use Controller Expectations**

```go
// GOOD: Track expectations
type Controller struct {
    expectations *controller.ControllerExpectations
    // ...
}

func (c *GoodController) scalePods(rs *apps.ReplicaSet, diff int) error {
    key, _ := controller.KeyFunc(rs)

    // Set expectation BEFORE creating
    c.expectations.ExpectCreations(key, diff)

    createErrors := []error{}
    for i := 0; i < diff; i++ {
        pod := newPod(rs, i)
        _, err := c.client.CoreV1().Pods(rs.Namespace).Create(
            context.TODO(),
            pod,
            metav1.CreateOptions{},
        )

        if err != nil {
            // Decrease expectation for failed create
            c.expectations.CreationObserved(key)
            createErrors = append(createErrors, err)
        }
    }

    return utilerrors.NewAggregate(createErrors)
}

func (c *GoodController) syncReplicaSet(rs *apps.ReplicaSet) error {
    key, _ := controller.KeyFunc(rs)

    // Check if expectations are satisfied
    if !c.expectations.SatisfiedExpectations(key) {
        klog.V(4).InfoS("Expectations not satisfied, skipping",
            "replicaset", key)
        return nil
    }

    // Proceed with sync
    return c.manageReplicas(rs)
}
```

---

## **6. Testing Anti-Patterns**

### **❌ Anti-Pattern 6.1: No Table-Driven Tests**

```go
// BAD: Repetitive test functions
func TestScaleUp(t *testing.T) {
    rs := newReplicaSet(3)
    c := newController()
    c.sync(rs)
    // assertions...
}

func TestScaleDown(t *testing.T) {
    rs := newReplicaSet(1)
    c := newController()
    c.sync(rs)
    // assertions...
}

func TestNoChange(t *testing.T) {
    rs := newReplicaSet(2)
    c := newController()
    c.sync(rs)
    // assertions...
}
// 50 more test functions...
```

**Problems**:
1. **Code duplication**: Same setup repeated
2. **Hard to maintain**: Changes require updating many functions
3. **Inconsistent**: Each test might set up differently

**✅ GOOD: Table-Driven Tests**

```go
// GOOD: Single test function with table
func TestReplicaSetSync(t *testing.T) {
    tests := []struct {
        name          string
        replicas      int32
        currentPods   int
        expectedAdds  int
        expectedDels  int
    }{
        {"scale up", 3, 0, 3, 0},
        {"scale down", 1, 5, 0, 4},
        {"no change", 2, 2, 0, 0},
        {"partial scale", 5, 2, 3, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            c := newController()
            rs := newReplicaSet(tt.replicas)
            // ... test logic ...
        })
    }
}
```

---

## **7. Code Review Checklist**

### **🚨 Red Flags to Watch For**

- [ ] Storing objects instead of keys in workqueue
- [ ] Missing `queue.Done()` calls
- [ ] Direct API calls instead of informer cache
- [ ] Single worker instead of worker pool
- [ ] Deep copying for read-only operations
- [ ] Ignoring resource versions on updates
- [ ] Swallowing errors without logging
- [ ] Retrying permanent errors
- [ ] Status updates triggering reconciliation loops
- [ ] No expectation tracking for creates/deletes
- [ ] No table-driven tests
- [ ] Missing race detector in CI (`go test -race`)
- [ ] No benchmarks for hot paths
- [ ] Hard-coded timeouts/retries
- [ ] Missing metrics/logging

---

## **8. Refactoring Guide**

### **Step-by-Step Anti-Pattern Elimination**

1. **Identify anti-pattern** using checklist
2. **Write test** demonstrating the issue
3. **Refactor** to correct pattern
4. **Verify test passes**
5. **Check performance** hasn't regressed
6. **Update documentation**

---

## **9. Source Code References**

| Anti-Pattern | Good Example File |
|--------------|-------------------|
| Workqueue usage | `pkg/controller/deployment/deployment_controller.go:400-500` |
| Informer cache | `pkg/controller/replicaset/replica_set.go:350-400` |
| Error handling | `pkg/controller/job/job_controller.go:500-600` |
| Expectations | `pkg/controller/replicaset/replica_set.go:600-700` |

---

## **Summary**

Avoid these anti-patterns:
1. **Storing objects in queues** - Store keys instead
2. **Skipping queue.Done()** - Always use defer
3. **Direct API calls** - Use informer cache
4. **Single worker** - Use worker pool
5. **Excessive copying** - Copy only when modifying
6. **Ignoring resource versions** - Use optimistic locking
7. **Swallowing errors** - Log and propagate
8. **Retrying permanent errors** - Classify errors
9. **Status update loops** - Separate spec/status, check generation
10. **No expectations** - Track creates/deletes

Following these patterns leads to scalable, maintainable, production-ready controllers.
