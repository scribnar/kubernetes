# **Custom Controller Observability in Kubernetes**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Comprehensive guide to implementing observability in custom Kubernetes controllers

**Target Audience**:
- Platform engineers building custom controllers
- Operators developing domain-specific controllers
- SREs managing controller-based systems
- Contributors to controller-runtime ecosystem

**Scope**: Metrics, health checks, logging, debugging, and production observability patterns

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Why Controller Observability Matters**

### **The Controller Observability Challenge**

Controllers are **autonomous reconciliation loops** that continuously watch resources and take actions to achieve desired state. Unlike request-driven services, controllers:

1. **Run continuously** - No clear request/response boundaries
2. **Process events asynchronously** - Queue-based processing with retries
3. **Maintain state** - Caches, workqueues, leader election status
4. **Have complex failure modes** - Retry storms, stuck queues, cache inconsistencies
5. **Impact cluster stability** - Bad controllers can overwhelm API servers

### **Production Failures Without Observability**

**Real-World Incident**: Stuck Controller Workqueue

```
Symptoms:
- Custom resources stuck in "Pending" status
- No error messages in CR status
- Controller pod running with normal CPU/memory

Root Cause (discovered after 3 hours):
- Workqueue had 50,000 items
- Controller was dequeuing but immediately re-queueing
- No metrics exposed retry count or queue depth
- Logs showed successful processing but missed the re-queue

Fix Required:
- Added workqueue depth metrics
- Exposed retry count per item
- Implemented rate limit metrics
- Added debug logging for re-queue reasons
```

### **Observability ROI**

| **Area** | **Without Observability** | **With Observability** |
|----------|---------------------------|------------------------|
| **Incident Detection** | User reports after 30+ min | Alert within 1 minute |
| **Root Cause Analysis** | 2-4 hours of debugging | 5-15 minutes with metrics |
| **Performance Tuning** | Guesswork, trial and error | Data-driven optimization |
| **Capacity Planning** | Over-provision by 3-5x | Right-size based on metrics |
| **Regression Detection** | Production incidents | CI/CD dashboards |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Controller Metrics Architecture**

### **Metrics Framework in Kubernetes**

Kubernetes uses **Prometheus client library** for metrics collection:

**File**: `staging/src/k8s.io/component-base/metrics/`

```go
// Metrics interface abstraction
type Metric interface {
    Desc() *prometheus.Desc
    Describe(chan<- *prometheus.Desc)
    Collect(chan<- prometheus.Metric)
}

// Core metric types
type Counter interface { Inc() }
type Gauge interface { Set(float64), Inc(), Dec() }
type Histogram interface { Observe(float64) }
type Summary interface { Observe(float64) }
```

### **Metric Registration Pattern**

**File**: `pkg/controller/job/metrics/metrics.go`

```go
package metrics

import (
    "sync"
    "k8s.io/component-base/metrics"
    "k8s.io/component-base/metrics/legacyregistry"
)

const JobControllerSubsystem = "job_controller"

var (
    // Ensure metrics are registered only once
    registerMetrics sync.Once

    // Histogram for sync duration
    JobSyncDurationSeconds = metrics.NewHistogramVec(
        &metrics.HistogramOpts{
            Subsystem:      JobControllerSubsystem,
            Name:           "job_sync_duration_seconds",
            Help:           "The time it took to sync a job",
            StabilityLevel: metrics.STABLE,
            Buckets:        metrics.ExponentialBuckets(0.004, 2, 15),
        },
        []string{"completion_mode", "result", "action"},
    )

    // Counter for sync operations
    JobSyncNum = metrics.NewCounterVec(
        &metrics.CounterOpts{
            Subsystem:      JobControllerSubsystem,
            Name:           "job_syncs_total",
            Help:           "The number of job syncs",
            StabilityLevel: metrics.STABLE,
        },
        []string{"completion_mode", "result", "action"},
    )

    // Counter for finished jobs
    JobFinishedNum = metrics.NewCounterVec(
        &metrics.CounterOpts{
            Subsystem:      JobControllerSubsystem,
            Name:           "jobs_finished_total",
            Help:           "The number of finished jobs",
            StabilityLevel: metrics.ALPHA,
        },
        []string{"completion_mode", "result", "reason"},
    )
)

// Register metrics with global registry
func Register() {
    registerMetrics.Do(func() {
        legacyregistry.MustRegister(JobSyncDurationSeconds)
        legacyregistry.MustRegister(JobSyncNum)
        legacyregistry.MustRegister(JobFinishedNum)
    })
}
```

### **Metric Instrumentation in Controller**

**File**: `pkg/controller/job/job_controller.go`

```go
func (jm *Controller) syncJob(ctx context.Context, key string) error {
    startTime := time.Now()
    defer func() {
        // Record sync duration regardless of outcome
        completionMode := getCompletionMode(job)
        metrics.JobSyncDurationSeconds.WithLabelValues(
            completionMode,
            resultSuccess,
            "sync",
        ).Observe(time.Since(startTime).Seconds())
    }()

    // Increment sync counter
    metrics.JobSyncNum.WithLabelValues(
        completionMode,
        resultSuccess,
        "sync",
    ).Inc()

    // ... reconciliation logic ...

    if jobSucceeded {
        metrics.JobFinishedNum.WithLabelValues(
            completionMode,
            "succeeded",
            "JobComplete",
        ).Inc()
    }

    return nil
}
```

### **Exposing Metrics Endpoint**

**File**: `staging/src/k8s.io/controller-manager/app/serve.go`

```go
func NewBaseHandler(
    c *componentbaseconfig.DebuggingConfiguration,
    healthzHandler http.Handler,
) *mux.PathRecorderMux {
    mux := mux.NewPathRecorderMux("controller-manager")

    // Health check endpoint
    mux.Handle("/healthz", healthzHandler)

    // Prometheus metrics endpoint
    mux.Handle("/metrics", legacyregistry.Handler())

    // Profiling endpoints (if enabled)
    if c.EnableProfiling {
        routes.Profiling{}.Install(mux)
        if c.EnableContentionProfiling {
            goruntime.SetBlockProfileRate(1)
        }
    }

    // Configuration endpoint
    configz.InstallHandler(mux)

    return mux
}
```

### **Metric Types and When to Use Them**

| **Metric Type** | **Use Case** | **Example** |
|-----------------|--------------|-------------|
| **Counter** | Events that only increase | `job_syncs_total`, `errors_total` |
| **Gauge** | Values that go up/down | `workqueue_depth`, `active_workers` |
| **Histogram** | Distribution of values | `sync_duration_seconds`, `request_size_bytes` |
| **Summary** | Percentiles without buckets | `reconcile_latency` (use Histogram instead) |

### **Histogram Bucket Configuration**

```go
// Default buckets (NOT recommended for most controllers)
prometheus.DefBuckets
// [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]

// Exponential buckets for controller sync operations
metrics.ExponentialBuckets(0.004, 2, 15)
// [0.004, 0.008, 0.016, 0.032, 0.064, 0.128, 0.256, 0.512, 1.024,
//  2.048, 4.096, 8.192, 16.384, 32.768, 65.536]

// Linear buckets for queue depth
metrics.LinearBuckets(0, 100, 20)
// [0, 100, 200, 300, ..., 1900]

// Custom buckets for specific SLOs
[]float64{0.01, 0.05, 0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💚 Health Checks and Readiness Probes**

### **Health Check Framework**

**File**: `staging/src/k8s.io/controller-manager/pkg/healthz/healthz.go`

```go
package healthz

import (
    "net/http"
    "k8s.io/apiserver/pkg/server/healthz"
)

// UnnamedHealthChecker interface
type UnnamedHealthChecker interface {
    Check(req *http.Request) error
}

// Create named health checker
func NamedHealthChecker(name string, check UnnamedHealthChecker) healthz.HealthChecker {
    return healthz.NamedCheck(name, check.Check)
}

// Multi-handler that combines multiple checks
type Handler interface {
    http.Handler
    AddHealthChecker(healthz.HealthChecker) error
    AddReadyzChecker(healthz.HealthChecker) error
    AddLivezChecker(healthz.HealthChecker) error
}
```

### **Implementing Health Checks in Controllers**

**File**: `cmd/kube-controller-manager/app/controllermanager.go`

```go
func Run(ctx context.Context, c *config.CompletedConfig) error {
    var checks []healthz.HealthChecker
    var electionChecker *leaderelection.HealthzAdaptor

    // 1. Leader election health check
    if c.ComponentConfig.Generic.LeaderElection.LeaderElect {
        electionChecker = leaderelection.NewLeaderHealthzAdaptor(
            time.Second * 20,  // Timeout for leader election
        )
        checks = append(checks, electionChecker)
    }

    // 2. Create mutable health handler
    healthzHandler := controllerhealthz.NewMutableHealthzHandler(checks...)

    // 3. Add informer sync health check (coordinated leader election)
    if utilfeature.DefaultFeatureGate.Enabled(kubefeatures.CoordinatedLeaderElection) {
        leaseCandidate, waitForSync, err := leaderelection.NewCandidate(
            c.Client, "kube-system", id, kubeControllerManager,
            binaryVersion, emulationVersion, coordinationv1.OldestEmulationVersion,
        )
        if err != nil {
            return err
        }

        // Add informer sync check
        healthzHandler.AddHealthChecker(
            healthz.NewInformerSyncHealthz(waitForSync),
        )
    }

    // 4. Setup HTTP handlers
    handler := NewBaseHandler(&c.ComponentConfig.Generic.Debugging, healthzHandler)

    // 5. Start server
    if err := c.SecureServing.Serve(handler, 0, ctx.Done()); err != nil {
        return err
    }

    return nil
}
```

### **Leader Election Health Adaptor**

**File**: `staging/src/k8s.io/client-go/tools/leaderelection/healthzadaptor.go`

```go
type HealthzAdaptor struct {
    timeout  time.Duration
    pointerLock sync.Mutex
    le          *LeaderElector
}

func (l *HealthzAdaptor) Check(req *http.Request) error {
    l.pointerLock.Lock()
    defer l.pointerLock.Unlock()

    if l.le == nil {
        return fmt.Errorf("leader election not started")
    }

    return l.le.Check(l.timeout)
}

func (l *HealthzAdaptor) SetLeaderElection(le *LeaderElector) {
    l.pointerLock.Lock()
    defer l.pointerLock.Unlock()
    l.le = le
}

func (le *LeaderElector) Check(maxTolerableDuration time.Duration) error {
    le.observedRecordLock.Lock()
    defer le.observedRecordLock.Unlock()

    if le.observedRecord.RenewTime.IsZero() {
        return fmt.Errorf("leader election has not yet started")
    }

    if le.clock.Since(le.observedTime) > maxTolerableDuration {
        return fmt.Errorf("leader election is not healthy: renewal is overdue")
    }

    return nil
}
```

### **Kubernetes Pod Probe Configuration**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-controller
spec:
  template:
    spec:
      containers:
      - name: controller
        image: custom-controller:latest
        ports:
        - name: metrics
          containerPort: 8080
          protocol: TCP
        - name: health
          containerPort: 8081
          protocol: TCP

        # Liveness probe - restart if fails
        livenessProbe:
          httpGet:
            path: /healthz
            port: health
            scheme: HTTP
          initialDelaySeconds: 15
          periodSeconds: 20
          timeoutSeconds: 3
          failureThreshold: 3

        # Readiness probe - remove from service if fails
        readinessProbe:
          httpGet:
            path: /readyz
            port: health
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3

        # Startup probe - for slow-starting controllers
        startupProbe:
          httpGet:
            path: /healthz
            port: health
            scheme: HTTP
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 30  # 5 minutes to start
```

### **Custom Health Checks**

```go
// Cache sync health check
type CacheSyncChecker struct {
    informers []cache.InformerSynced
}

func (c *CacheSyncChecker) Check(req *http.Request) error {
    for _, synced := range c.informers {
        if !synced() {
            return fmt.Errorf("cache not synced")
        }
    }
    return nil
}

// Workqueue health check
type WorkqueueHealthChecker struct {
    queue workqueue.TypedRateLimitingInterface[string]
    maxDepth int
}

func (w *WorkqueueHealthChecker) Check(req *http.Request) error {
    depth := w.queue.Len()
    if depth > w.maxDepth {
        return fmt.Errorf("workqueue depth %d exceeds threshold %d",
            depth, w.maxDepth)
    }
    return nil
}

// API server connectivity check
type APIServerConnectivityChecker struct {
    client kubernetes.Interface
}

func (a *APIServerConnectivityChecker) Check(req *http.Request) error {
    ctx, cancel := context.WithTimeout(req.Context(), 2*time.Second)
    defer cancel()

    _, err := a.client.Discovery().ServerVersion()
    if err != nil {
        return fmt.Errorf("failed to connect to API server: %v", err)
    }
    return nil
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📝 Structured Logging Patterns**

### **Context-Based Logging**

**File**: `staging/src/k8s.io/sample-controller/controller.go`

```go
package controller

import (
    "context"
    "k8s.io/klog/v2"
)

func NewController(
    ctx context.Context,
    kubeclientset kubernetes.Interface,
    sampleclientset clientset.Interface,
    deploymentInformer appsinformers.DeploymentInformer,
    fooInformer informers.FooInformer,
) *Controller {
    logger := klog.FromContext(ctx)

    logger.V(4).Info("Creating event broadcaster")
    eventBroadcaster := record.NewBroadcaster(record.WithContext(ctx))

    logger.Info("Setting up event handlers")

    controller := &Controller{
        // ... fields ...
    }

    return controller
}
```

### **Structured Logging with Key-Value Pairs**

```go
func (c *Controller) syncHandler(ctx context.Context, objectRef cache.ObjectName) error {
    // Create logger with contextual information
    logger := klog.LoggerWithValues(
        klog.FromContext(ctx),
        "objectRef", objectRef,
        "controller", "foo-controller",
    )

    namespace, name, err := cache.SplitMetaNamespaceKey(objectRef.String())
    if err != nil {
        logger.Error(err, "Invalid resource key")
        return nil
    }

    logger = klog.LoggerWithValues(logger,
        "namespace", namespace,
        "name", name,
    )

    // Get resource from lister
    foo, err := c.foosLister.Foos(namespace).Get(name)
    if err != nil {
        if errors.IsNotFound(err) {
            logger.Info("Resource has been deleted")
            return nil
        }
        logger.Error(err, "Failed to get resource from lister")
        return err
    }

    logger.V(4).Info("Processing resource",
        "generation", foo.Generation,
        "resourceVersion", foo.ResourceVersion,
    )

    // Business logic with logging
    deploymentName := foo.Spec.DeploymentName
    deployment, err := c.deploymentsLister.Deployments(foo.Namespace).Get(deploymentName)

    if errors.IsNotFound(err) {
        logger.Info("Creating new deployment",
            "deploymentName", deploymentName,
            "replicas", *foo.Spec.Replicas,
        )
        deployment, err = c.kubeclientset.AppsV1().Deployments(foo.Namespace).Create(
            ctx, newDeployment(foo), metav1.CreateOptions{},
        )
    }

    if err != nil {
        logger.Error(err, "Failed to create deployment",
            "deploymentName", deploymentName,
        )
        return err
    }

    // Update if needed
    if foo.Spec.Replicas != nil && *foo.Spec.Replicas != *deployment.Spec.Replicas {
        logger.V(4).Info("Update deployment resource",
            "currentReplicas", *deployment.Spec.Replicas,
            "desiredReplicas", *foo.Spec.Replicas,
        )
        deployment, err = c.kubeclientset.AppsV1().Deployments(foo.Namespace).Update(
            ctx, newDeployment(foo), metav1.UpdateOptions{},
        )
    }

    if err != nil {
        logger.Error(err, "Failed to update deployment",
            "deploymentName", deploymentName,
        )
        return err
    }

    logger.Info("Successfully synced resource")
    return nil
}
```

### **Verbosity Levels**

| **Level** | **Purpose** | **Example** |
|-----------|-------------|-------------|
| **0** | Always shown | Startup, shutdown, critical errors |
| **1** | Important info | Controller started, leader elected |
| **2** | Useful info | Reconciliation completed, resource created |
| **3** | Extended info | Resource updates, API calls |
| **4** | Debug info | Cache operations, queue events |
| **5+** | Trace info | Every function call, detailed state |

### **Rate-Limited Error Logging**

**File**: `staging/src/k8s.io/sample-controller/controller.go`

```go
func (c *Controller) processNextWorkItem(ctx context.Context) bool {
    logger := klog.FromContext(ctx)

    objRef, shutdown := c.workqueue.Get()
    if shutdown {
        return false
    }
    defer c.workqueue.Done(objRef)

    err := c.syncHandler(ctx, objRef)
    if err == nil {
        // Success - forget rate limiting
        c.workqueue.Forget(objRef)
        logger.Info("Successfully synced", "objectName", objRef)
        return true
    }

    // Error - apply rate limiting
    utilruntime.HandleErrorWithContext(ctx, err,
        "Error syncing; requeuing for later retry",
        "objectReference", objRef,
    )
    c.workqueue.AddRateLimited(objRef)

    return true
}
```

### **Logging Best Practices**

```go
// ✅ GOOD: Structured logging with context
logger.Info("Reconciling resource",
    "namespace", obj.Namespace,
    "name", obj.Name,
    "generation", obj.Generation,
)

// ❌ BAD: String concatenation
logger.Info(fmt.Sprintf("Reconciling %s/%s generation %d",
    obj.Namespace, obj.Name, obj.Generation))

// ✅ GOOD: Appropriate verbosity
logger.V(4).Info("Cache updated", "key", key)

// ❌ BAD: Too verbose at level 0
logger.Info("Cache updated", "key", key)

// ✅ GOOD: Error with context
logger.Error(err, "Failed to update resource",
    "namespace", obj.Namespace,
    "name", obj.Name,
)

// ❌ BAD: Error without context
logger.Error(err, "Update failed")

// ✅ GOOD: Conditional debug logging
if logger.V(5).Enabled() {
    logger.V(5).Info("Full object state", "object", obj)
}

// ❌ BAD: Always serialize expensive data
logger.V(5).Info("Full object state", "object", obj)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎛️ Workqueue Observability**

### **Workqueue Metrics**

**File**: `staging/src/k8s.io/client-go/util/workqueue/metrics.go`

```go
// Default metrics provider
type defaultMetricsProvider struct{}

func (defaultMetricsProvider) NewDepthMetric(name string) GaugeMetric {
    return &noopMetric{}
}

func (defaultMetricsProvider) NewAddsMetric(name string) CounterMetric {
    return &noopMetric{}
}

func (defaultMetricsProvider) NewLatencyMetric(name string) HistogramMetric {
    return &noopMetric{}
}

func (defaultMetricsProvider) NewWorkDurationMetric(name string) HistogramMetric {
    return &noopMetric{}
}

func (defaultMetricsProvider) NewUnfinishedWorkSecondsMetric(name string) SettableGaugeMetric {
    return &noopMetric{}
}

func (defaultMetricsProvider) NewLongestRunningProcessorSecondsMetric(name string) SettableGaugeMetric {
    return &noopMetric{}
}

func (defaultMetricsProvider) NewRetriesMetric(name string) CounterMetric {
    return &noopMetric{}
}
```

### **Prometheus Metrics Provider**

**File**: `staging/src/k8s.io/component-base/metrics/prometheus/workqueue/metrics.go`

```go
package workqueue

import (
    "k8s.io/client-go/util/workqueue"
    compbasemetrics "k8s.io/component-base/metrics"
)

var (
    depth = compbasemetrics.NewGaugeVec(&compbasemetrics.GaugeOpts{
        Subsystem:      WorkQueueSubsystem,
        Name:           "depth",
        Help:           "Current depth of workqueue",
        StabilityLevel: compbasemetrics.ALPHA,
    }, []string{"name"})

    adds = compbasemetrics.NewCounterVec(&compbasemetrics.CounterOpts{
        Subsystem:      WorkQueueSubsystem,
        Name:           "adds_total",
        Help:           "Total number of adds handled by workqueue",
        StabilityLevel: compbasemetrics.ALPHA,
    }, []string{"name"})

    latency = compbasemetrics.NewHistogramVec(&compbasemetrics.HistogramOpts{
        Subsystem:      WorkQueueSubsystem,
        Name:           "queue_duration_seconds",
        Help:           "How long in seconds an item stays in workqueue before being requested",
        Buckets:        compbasemetrics.ExponentialBuckets(10e-9, 10, 12),
        StabilityLevel: compbasemetrics.ALPHA,
    }, []string{"name"})

    workDuration = compbasemetrics.NewHistogramVec(&compbasemetrics.HistogramOpts{
        Subsystem:      WorkQueueSubsystem,
        Name:           "work_duration_seconds",
        Help:           "How long in seconds processing an item from workqueue takes",
        Buckets:        compbasemetrics.ExponentialBuckets(10e-9, 10, 12),
        StabilityLevel: compbasemetrics.ALPHA,
    }, []string{"name"})

    retries = compbasemetrics.NewCounterVec(&compbasemetrics.CounterOpts{
        Subsystem:      WorkQueueSubsystem,
        Name:           "retries_total",
        Help:           "Total number of retries handled by workqueue",
        StabilityLevel: compbasemetrics.ALPHA,
    }, []string{"name"})
)
```

### **Workqueue Creation with Metrics**

**File**: `pkg/controller/job/job_controller.go`

```go
func newControllerWithClock(
    ctx context.Context,
    kubeClient clientset.Interface,
    podControl Controller,
    clock clock.WithTicker,
) (*Controller, error) {
    logger := klog.FromContext(ctx)

    // Create rate-limited workqueue with metrics
    jm := &Controller{
        kubeClient: kubeClient,
        podControl: podControl,

        // Workqueue with exponential backoff
        queue: workqueue.NewTypedRateLimitingQueueWithConfig(
            workqueue.NewTypedItemExponentialFailureRateLimiter[string](
                DefaultJobApiBackOff,  // 5ms initial delay
                MaxJobApiBackOff,      // 1000s max delay
            ),
            workqueue.TypedRateLimitingQueueConfig[string]{
                Name:  "job",  // Appears in metrics as workqueue_depth{name="job"}
                Clock: clock,
            },
        ),

        // Other fields...
    }

    // Register controller-specific metrics
    metrics.Register()

    logger.Info("Created job controller with workqueue",
        "name", "job",
        "initialBackoff", DefaultJobApiBackOff,
        "maxBackoff", MaxJobApiBackOff,
    )

    return jm, nil
}
```

### **Monitoring Workqueue Metrics**

**Prometheus Queries**:

```promql
# Queue depth (current number of items)
workqueue_depth{name="job"}

# Items added per second
rate(workqueue_adds_total{name="job"}[5m])

# Queue latency (time in queue before processing)
histogram_quantile(0.99,
    rate(workqueue_queue_duration_seconds_bucket{name="job"}[5m]))

# Work duration (processing time)
histogram_quantile(0.99,
    rate(workqueue_work_duration_seconds_bucket{name="job"}[5m]))

# Retry rate
rate(workqueue_retries_total{name="job"}[5m])

# Unfinished work (items currently being processed)
workqueue_unfinished_work_seconds{name="job"}
```

**Grafana Dashboard Example**:

```json
{
  "dashboard": {
    "panels": [
      {
        "title": "Workqueue Depth",
        "targets": [{
          "expr": "workqueue_depth{name=~\"$controller\"}"
        }]
      },
      {
        "title": "Queue Latency P99",
        "targets": [{
          "expr": "histogram_quantile(0.99, rate(workqueue_queue_duration_seconds_bucket{name=~\"$controller\"}[5m]))"
        }]
      },
      {
        "title": "Retry Rate",
        "targets": [{
          "expr": "rate(workqueue_retries_total{name=~\"$controller\"}[5m])"
        }]
      }
    ]
  }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Event Recording and Auditing**

### **Event Broadcaster Setup**

**File**: `staging/src/k8s.io/sample-controller/controller.go`

```go
func NewController(ctx context.Context, ...) *Controller {
    logger := klog.FromContext(ctx)

    // Create event broadcaster
    logger.V(4).Info("Creating event broadcaster")
    eventBroadcaster := record.NewBroadcaster(record.WithContext(ctx))

    // Log events to structured logging
    eventBroadcaster.StartStructuredLogging(0)

    // Send events to API server
    eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{
        Interface: kubeclientset.CoreV1().Events(""),
    })

    // Create event recorder
    recorder := eventBroadcaster.NewRecorder(
        scheme.Scheme,
        corev1.EventSource{Component: controllerAgentName},
    )

    controller := &Controller{
        recorder: recorder,
        // ... other fields ...
    }

    return controller
}
```

### **Recording Events**

```go
const (
    // Event reasons
    SuccessSynced         = "Synced"
    ErrResourceExists     = "ErrResourceExists"
    MessageResourceSynced = "Foo synced successfully"
)

func (c *Controller) syncHandler(ctx context.Context, objectRef cache.ObjectName) error {
    logger := klog.LoggerWithValues(klog.FromContext(ctx), "objectRef", objectRef)

    // Get the resource
    namespace, name, _ := cache.SplitMetaNamespaceKey(objectRef.String())
    foo, err := c.foosLister.Foos(namespace).Get(name)

    // Check if deployment exists
    deploymentName := foo.Spec.DeploymentName
    deployment, err := c.deploymentsLister.Deployments(foo.Namespace).Get(deploymentName)

    if errors.IsNotFound(err) {
        // Create deployment
        deployment, err = c.kubeclientset.AppsV1().Deployments(foo.Namespace).Create(
            ctx, newDeployment(foo), metav1.CreateOptions{},
        )
        if err != nil {
            // Record error event
            c.recorder.Event(foo, corev1.EventTypeWarning, ErrResourceExists,
                fmt.Sprintf("Failed to create deployment %s: %v", deploymentName, err))
            return err
        }

        // Record success event
        c.recorder.Event(foo, corev1.EventTypeNormal, SuccessSynced,
            fmt.Sprintf("Created deployment %s", deploymentName))
    }

    // Update resource status
    err = c.updateFooStatus(foo, deployment)
    if err != nil {
        return err
    }

    // Record final success event
    c.recorder.Event(foo, corev1.EventTypeNormal, SuccessSynced, MessageResourceSynced)

    return nil
}
```

### **Event Types and Patterns**

```go
// Event type constants
const (
    EventTypeNormal  = "Normal"
    EventTypeWarning = "Warning"
)

// Event reasons (should be CamelCase)
const (
    // Success reasons
    ReasonCreated     = "Created"
    ReasonUpdated     = "Updated"
    ReasonDeleted     = "Deleted"
    ReasonSynced      = "Synced"

    // Warning reasons
    ReasonFailed      = "Failed"
    ReasonBackOff     = "BackOff"
    ReasonConflict    = "Conflict"
    ReasonInvalid     = "Invalid"
)

// Recording patterns
func (c *Controller) recordEvent(
    obj runtime.Object,
    eventType, reason, message string,
    args ...interface{},
) {
    if len(args) > 0 {
        message = fmt.Sprintf(message, args...)
    }
    c.recorder.Event(obj, eventType, reason, message)
}

// Examples
c.recordEvent(pod, EventTypeNormal, ReasonCreated,
    "Created pod %s/%s", pod.Namespace, pod.Name)

c.recordEvent(pod, EventTypeWarning, ReasonFailed,
    "Failed to mount volume: %v", err)
```

### **Viewing Events**

```bash
# Get events for a specific resource
kubectl describe foo my-foo

# Events:
#   Type    Reason          Age   From              Message
#   ----    ------          ----  ----              -------
#   Normal  Synced          2m    foo-controller    Foo synced successfully
#   Normal  Created         2m    foo-controller    Created deployment my-foo-deployment

# Get all events in namespace
kubectl get events -n default --sort-by='.lastTimestamp'

# Get events for debugging
kubectl get events -n default --field-selector involvedObject.name=my-foo
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔬 Debugging and Profiling**

### **Profiling Configuration**

**File**: `staging/src/k8s.io/controller-manager/options/debugging.go`

```go
package options

import (
    "github.com/spf13/pflag"
    componentbaseconfig "k8s.io/component-base/config"
)

type DebuggingOptions struct {
    *componentbaseconfig.DebuggingConfiguration
}

func RecommendedDebuggingOptions() *DebuggingOptions {
    return &DebuggingOptions{
        DebuggingConfiguration: &componentbaseconfig.DebuggingConfiguration{
            EnableProfiling:           true,
            EnableContentionProfiling: false,
        },
    }
}

func (o *DebuggingOptions) AddFlags(fs *pflag.FlagSet) {
    if o == nil {
        return
    }

    fs.BoolVar(&o.EnableProfiling, "profiling", o.EnableProfiling,
        "Enable profiling via web interface host:port/debug/pprof/")

    fs.BoolVar(&o.EnableContentionProfiling, "contention-profiling",
        o.EnableContentionProfiling,
        "Enable block profiling, if profiling is enabled")
}
```

### **Enabling Profiling Endpoints**

**File**: `staging/src/k8s.io/controller-manager/app/serve.go`

```go
import (
    "net/http"
    goruntime "runtime"
    "k8s.io/apiserver/pkg/server/routes"
)

func NewBaseHandler(
    c *componentbaseconfig.DebuggingConfiguration,
    healthzHandler http.Handler,
) *mux.PathRecorderMux {
    mux := mux.NewPathRecorderMux("controller-manager")

    mux.Handle("/healthz", healthzHandler)
    mux.Handle("/metrics", legacyregistry.Handler())

    // Install profiling endpoints
    if c.EnableProfiling {
        routes.Profiling{}.Install(mux)

        // Enable contention profiling
        if c.EnableContentionProfiling {
            goruntime.SetBlockProfileRate(1)
            goruntime.SetMutexProfileFraction(1)
        }
    }

    // Install configz endpoint
    configz.InstallHandler(mux)

    return mux
}
```

### **Available Profiling Endpoints**

| **Endpoint** | **Purpose** | **Usage** |
|--------------|-------------|-----------|
| `/debug/pprof/` | Profile index page | List available profiles |
| `/debug/pprof/profile` | CPU profile | `curl -o cpu.prof http://localhost:8080/debug/pprof/profile?seconds=30` |
| `/debug/pprof/heap` | Memory heap profile | `curl -o heap.prof http://localhost:8080/debug/pprof/heap` |
| `/debug/pprof/goroutine` | Goroutine dump | `curl http://localhost:8080/debug/pprof/goroutine?debug=2` |
| `/debug/pprof/block` | Blocking profile | `curl -o block.prof http://localhost:8080/debug/pprof/block` |
| `/debug/pprof/mutex` | Mutex contention | `curl -o mutex.prof http://localhost:8prof/mutex` |
| `/debug/pprof/threadcreate` | Thread creation | `curl http://localhost:8080/debug/pprof/threadcreate?debug=1` |
| `/debug/pprof/trace` | Execution trace | `curl -o trace.out http://localhost:8080/debug/pprof/trace?seconds=5` |

### **Collecting and Analyzing Profiles**

**CPU Profiling**:

```bash
# Collect 30-second CPU profile
kubectl port-forward -n kube-system deploy/custom-controller 8080:8080 &
curl -o cpu.prof http://localhost:8080/debug/pprof/profile?seconds=30

# Analyze with pprof
go tool pprof cpu.prof

# Interactive commands:
(pprof) top10        # Top 10 functions by CPU time
(pprof) list syncHandler  # Source code for function
(pprof) web          # Open browser visualization
(pprof) pdf > cpu.pdf     # Generate PDF report
```

**Memory Profiling**:

```bash
# Collect heap profile
curl -o heap.prof http://localhost:8080/debug/pprof/heap

# Analyze allocations
go tool pprof -alloc_space heap.prof

# Analyze in-use memory
go tool pprof -inuse_space heap.prof

# Compare two profiles
go tool pprof -base heap1.prof heap2.prof
```

**Goroutine Analysis**:

```bash
# Get goroutine dump
curl http://localhost:8080/debug/pprof/goroutine?debug=2 > goroutines.txt

# Example output:
# goroutine 123 [chan receive, 45 minutes]:
# k8s.io/client-go/util/workqueue.(*Type).Get(...)
#     /go/pkg/mod/k8s.io/client-go/util/workqueue/queue.go:123
# main.(*Controller).worker(...)
#     /app/controller.go:234
```

**Mutex Contention**:

```bash
# Enable mutex profiling (requires EnableContentionProfiling)
curl -o mutex.prof http://localhost:8080/debug/pprof/mutex

# Analyze
go tool pprof mutex.prof
(pprof) top10
```

### **Trace Analysis**

```bash
# Collect execution trace
curl -o trace.out http://localhost:8080/debug/pprof/trace?seconds=5

# Analyze trace
go tool trace trace.out

# Opens browser with:
# - View trace timeline
# - Goroutine analysis
# - Network blocking profile
# - Synchronization blocking profile
# - Syscall blocking profile
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Controller-Runtime Observability**

### **Manager Setup with Observability**

**File**: `docs/architecture/claude/controller-manager/65-controller-runtime-guide.md`

```go
package main

import (
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/metrics"
)

func main() {
    // Setup logger
    ctrl.SetLogger(klog.NewKlogr())

    // Create manager with observability
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,

        // Metrics endpoint
        MetricsBindAddress: ":8080",

        // Health/readiness endpoint
        HealthProbeBindAddress: ":8081",

        // Leader election
        LeaderElection:   true,
        LeaderElectionID: "my-controller-lock",

        // Increase client QPS for large clusters
        Client: client.Options{
            QPS:   50,
            Burst: 100,
        },
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }

    // Add health checks
    if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up health check")
        os.Exit(1)
    }

    if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up ready check")
        os.Exit(1)
    }

    // Setup reconciler
    if err = (&MyReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller")
        os.Exit(1)
    }

    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

### **Reconciler Metrics**

```go
package controllers

import (
    "context"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
    reconcileTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "controller_reconcile_total",
            Help: "Total number of reconciliations per controller",
        },
        []string{"controller", "result"},
    )

    reconcileDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "controller_reconcile_duration_seconds",
            Help: "Duration of reconciliations per controller",
            Buckets: prometheus.ExponentialBuckets(0.001, 2, 15),
        },
        []string{"controller"},
    )
)

func init() {
    // Register metrics with controller-runtime registry
    metrics.Registry.MustRegister(reconcileTotal, reconcileDuration)
}

type MyReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    startTime := time.Now()

    defer func() {
        // Record duration
        reconcileDuration.WithLabelValues("my-controller").Observe(
            time.Since(startTime).Seconds(),
        )
    }()

    // Fetch the resource
    var obj MyResource
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        if errors.IsNotFound(err) {
            reconcileTotal.WithLabelValues("my-controller", "not_found").Inc()
            return ctrl.Result{}, nil
        }
        reconcileTotal.WithLabelValues("my-controller", "error").Inc()
        logger.Error(err, "unable to fetch resource")
        return ctrl.Result{}, err
    }

    // Reconciliation logic...

    reconcileTotal.WithLabelValues("my-controller", "success").Inc()
    return ctrl.Result{}, nil
}
```

### **Predicates for Efficient Reconciliation**

```go
import (
    "sigs.k8s.io/controller-runtime/pkg/event"
    "sigs.k8s.io/controller-runtime/pkg/predicate"
)

// Custom predicate to reduce reconciliations
type GenerationChangedPredicate struct {
    predicate.Funcs
}

func (GenerationChangedPredicate) Update(e event.UpdateEvent) bool {
    // Only reconcile if generation changed (spec changed)
    if e.ObjectOld.GetGeneration() != e.ObjectNew.GetGeneration() {
        return true
    }

    // Also reconcile if deletion timestamp changed
    if e.ObjectOld.GetDeletionTimestamp() != e.ObjectNew.GetDeletionTimestamp() {
        return true
    }

    return false
}

// Setup with predicates
func (r *MyReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&myapiv1.MyResource{}).
        WithEventFilter(GenerationChangedPredicate{}).
        Complete(r)
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚨 Production Troubleshooting**

### **Scenario 1: High Workqueue Depth**

**Symptoms**:
```promql
# Workqueue depth growing continuously
workqueue_depth{name="my-controller"} > 1000

# High retry rate
rate(workqueue_retries_total{name="my-controller"}[5m]) > 10
```

**Investigation**:

```bash
# 1. Check workqueue metrics
kubectl port-forward -n system pod/controller-xxx 8080:8080
curl http://localhost:8080/metrics | grep workqueue

# workqueue_depth{name="my-controller"} 5234
# workqueue_retries_total{name="my-controller"} 15234

# 2. Check controller logs
kubectl logs -n system controller-xxx --tail=100 | grep -A5 "Error syncing"

# 3. Get goroutine dump
curl http://localhost:8080/debug/pprof/goroutine?debug=2 > goroutines.txt
grep "worker" goroutines.txt

# 4. Check API server rate limiting
kubectl logs -n system controller-xxx | grep "rate limit"
```

**Common Causes**:

1. **API Server Throttling**:
```go
// Increase client QPS
config.QPS = 50
config.Burst = 100
```

2. **Slow Reconciliation**:
```go
// Add timeout to reconciliation
ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
defer cancel()
```

3. **Retry Storm**:
```go
// Adjust rate limiter
workqueue.NewTypedItemExponentialFailureRateLimiter[string](
    100*time.Millisecond,  // Increase base delay
    1000*time.Second,      // Keep max delay
)
```

### **Scenario 2: Memory Leak**

**Symptoms**:
```promql
# Memory growing over time
process_resident_memory_bytes > 2GB

# GC unable to keep up
rate(go_gc_duration_seconds_sum[5m]) > 0.1
```

**Investigation**:

```bash
# 1. Collect heap profile
curl -o heap1.prof http://localhost:8080/debug/pprof/heap
sleep 300
curl -o heap2.prof http://localhost:8080/debug/pprof/heap

# 2. Compare profiles
go tool pprof -base heap1.prof heap2.prof
(pprof) top20
(pprof) list suspiciousFunction

# 3. Check for leaked goroutines
curl http://localhost:8080/debug/pprof/goroutine?debug=2 | grep "^goroutine" | wc -l

# 4. Analyze allocation sites
go tool pprof -alloc_space heap2.prof
(pprof) top20
```

**Common Causes**:

1. **Unbounded Cache**:
```go
// ❌ BAD: No eviction policy
cache := make(map[string]*largeObject)

// ✅ GOOD: Use LRU cache with size limit
cache := lru.New(1000)
```

2. **Goroutine Leak**:
```go
// ❌ BAD: Context never canceled
go func() {
    ticker := time.NewTicker(time.Second)
    for range ticker.C {
        // Work...
    }
}()

// ✅ GOOD: Respect context cancellation
go func(ctx context.Context) {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            // Work...
        }
    }
}(ctx)
```

3. **Large Informer Cache**:
```go
// Monitor cache size
cacheMetrics := prometheus.NewGaugeVec(
    prometheus.GaugeOpts{
        Name: "controller_cache_size",
        Help: "Size of cached objects",
    },
    []string{"resource"},
)

func (c *Controller) monitorCacheSize() {
    ticker := time.NewTicker(time.Minute)
    defer ticker.Stop()

    for range ticker.C {
        pods := c.podLister.List(labels.Everything())
        cacheMetrics.WithLabelValues("pods").Set(float64(len(pods)))
    }
}
```

### **Scenario 3: Slow Reconciliation**

**Symptoms**:
```promql
# P99 work duration > 10s
histogram_quantile(0.99,
    rate(workqueue_work_duration_seconds_bucket{name="my-controller"}[5m])) > 10

# Queue latency growing
histogram_quantile(0.99,
    rate(workqueue_queue_duration_seconds_bucket{name="my-controller"}[5m])) > 60
```

**Investigation**:

```bash
# 1. CPU profile
curl -o cpu.prof http://localhost:8080/debug/pprof/profile?seconds=30
go tool pprof cpu.prof
(pprof) top20
(pprof) web

# 2. Trace analysis
curl -o trace.out http://localhost:8080/debug/pprof/trace?seconds=5
go tool trace trace.out
# View trace timeline

# 3. Check external API latency
kubectl logs controller-xxx | grep "API call took"
```

**Optimization Patterns**:

```go
// 1. Batch API calls
var pods []*corev1.Pod
err := r.List(ctx, &corev1.PodList{},
    client.InNamespace(ns),
    client.MatchingLabels(labels),
).Items(&pods)

// 2. Use cached reads
pod, err := r.podLister.Pods(ns).Get(name)  // Cached
// vs
pod := &corev1.Pod{}
err := r.Get(ctx, types.NamespacedName{...}, pod)  // API call

// 3. Parallelize independent operations
g, ctx := errgroup.WithContext(ctx)

g.Go(func() error {
    return r.reconcilePods(ctx, obj)
})

g.Go(func() error {
    return r.reconcileServices(ctx, obj)
})

if err := g.Wait(); err != nil {
    return ctrl.Result{}, err
}

// 4. Add reconciliation timeout
ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
defer cancel()
```

### **Scenario 4: Leader Election Flapping**

**Symptoms**:
```bash
# Frequent leader changes
kubectl logs controller-xxx | grep "became leader\|stopped leading"

# I0117 10:23:15 leaderelection.go:248] successfully acquired lease kube-system/my-controller
# I0117 10:24:32 leaderelection.go:258] failed to renew lease: context deadline exceeded
# I0117 10:24:33 leaderelection.go:248] successfully acquired lease kube-system/my-controller
```

**Investigation**:

```bash
# 1. Check leader election health
curl http://localhost:8081/healthz/leaderElection

# 2. Check API server latency
kubectl top nodes  # High latency indicator

# 3. Check network latency
kubectl exec controller-xxx -- ping -c 10 kubernetes.default.svc

# 4. Check lease object
kubectl get lease -n kube-system my-controller-lock -o yaml
```

**Fixes**:

```go
// Increase lease duration and retry period
LeaderElectionConfig: leaderelection.LeaderElectionConfig{
    LeaseDuration: 30 * time.Second,  // Default: 15s
    RenewDeadline: 20 * time.Second,  // Default: 10s
    RetryPeriod:   5 * time.Second,   // Default: 2s
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Comprehensive Observability Checklist**

### **Pre-Production Checklist**

- [ ] **Metrics**
  - [ ] Controller-specific metrics registered
  - [ ] Workqueue metrics enabled
  - [ ] Histogram buckets tuned for expected latencies
  - [ ] Metric cardinality reviewed (avoid high-cardinality labels)
  - [ ] Metrics endpoint exposed on separate port

- [ ] **Health Checks**
  - [ ] Liveness probe configured
  - [ ] Readiness probe configured
  - [ ] Leader election health check added
  - [ ] Informer sync health check added
  - [ ] Appropriate timeout values set

- [ ] **Logging**
  - [ ] Structured logging with context
  - [ ] Appropriate verbosity levels
  - [ ] Error logging includes stack traces
  - [ ] Rate limiting for high-frequency logs
  - [ ] Sensitive data redacted

- [ ] **Debugging**
  - [ ] Profiling endpoints enabled
  - [ ] Debug flags supported
  - [ ] Trace support added for distributed tracing
  - [ ] Goroutine dumps available

- [ ] **Event Recording**
  - [ ] Event broadcaster configured
  - [ ] Meaningful event reasons defined
  - [ ] Events recorded for key state transitions
  - [ ] Event rate limiting in place

- [ ] **Dashboards**
  - [ ] Grafana dashboard created
  - [ ] Alerts configured for critical metrics
  - [ ] Runbooks linked from alerts
  - [ ] Dashboard reviewed by SRE team

- [ ] **Testing**
  - [ ] Load testing performed
  - [ ] Memory leak testing completed
  - [ ] Failure injection scenarios validated
  - [ ] Metrics accuracy verified

### **Production Monitoring**

**Essential Metrics**:

```yaml
# Prometheus alerts
groups:
- name: controller-alerts
  rules:
  # High workqueue depth
  - alert: ControllerWorkqueueHigh
    expr: workqueue_depth{name="my-controller"} > 1000
    for: 5m
    annotations:
      summary: "Controller workqueue depth is high"

  # High error rate
  - alert: ControllerErrorRateHigh
    expr: rate(controller_reconcile_total{result="error"}[5m]) > 0.1
    for: 5m
    annotations:
      summary: "Controller error rate is high"

  # Slow reconciliation
  - alert: ControllerReconcileSlow
    expr: |
      histogram_quantile(0.99,
        rate(controller_reconcile_duration_seconds_bucket[5m])) > 30
    for: 10m
    annotations:
      summary: "Controller reconciliation is slow"

  # Leader election unhealthy
  - alert: ControllerLeaderElectionUnhealthy
    expr: up{job="my-controller"} == 0
    for: 2m
    annotations:
      summary: "Controller leader election is unhealthy"
```

**Grafana Dashboard Panels**:

1. **Overview**
   - Active replicas
   - Leader election status
   - Health check status

2. **Workqueue**
   - Queue depth (gauge)
   - Items added/sec (counter rate)
   - Queue latency P50/P95/P99 (histogram quantiles)
   - Retry rate (counter rate)

3. **Reconciliation**
   - Reconciliations/sec by result (counter rate)
   - Reconciliation duration P50/P95/P99 (histogram quantiles)
   - Error rate (counter rate)

4. **Resources**
   - CPU usage
   - Memory usage
   - Goroutine count
   - GC pause time

5. **API Client**
   - API call rate by verb
   - API call latency
   - API error rate
   - Rate limiting events

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💡 Best Practices Summary**

### **Metrics**

1. **Use subsystems** for metric organization
2. **Define stability levels** (ALPHA, BETA, STABLE)
3. **Choose appropriate bucket sizes** for histograms
4. **Register metrics once** with sync.Once
5. **Avoid high-cardinality labels** (no UUIDs, timestamps)
6. **Use counters for events**, gauges for current state, histograms for distributions

### **Health Checks**

1. **Implement both liveness and readiness** probes
2. **Add leader election health checks** for HA controllers
3. **Include informer sync checks** to ensure cache is ready
4. **Set appropriate timeouts** (e.g., 20s for leader election)
5. **Use mutable health handlers** for dynamic checks

### **Logging**

1. **Use context-based loggers** for correlation
2. **Add structured fields** with LoggerWithValues
3. **Use appropriate verbosity levels** (0-5)
4. **Rate-limit high-frequency logs** to prevent log flooding
5. **Include resource identifiers** (namespace, name) in all logs

### **Debugging**

1. **Enable profiling endpoints** by default in non-production
2. **Support contention profiling** for lock analysis
3. **Expose configz** for runtime configuration
4. **Add debug flags** for verbose logging
5. **Collect profiles regularly** in production (with sampling)

### **Performance**

1. **Use cached reads** from listers instead of API calls
2. **Implement predicates** to reduce unnecessary reconciliations
3. **Set appropriate rate limits** on workqueues
4. **Batch API calls** where possible
5. **Add timeouts** to reconciliation contexts

### **Production Readiness**

1. **Load test** before production deployment
2. **Create dashboards** and alerts
3. **Write runbooks** for common failure scenarios
4. **Document metrics** and their meanings
5. **Test failure scenarios** (API server outage, network partition)
6. **Monitor memory** and goroutine count continuously
7. **Set up distributed tracing** for complex controllers

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 References**

### **Source Code**
- **Workqueue Metrics**: `staging/src/k8s.io/client-go/util/workqueue/metrics.go`
- **Job Controller Metrics**: `pkg/controller/job/metrics/metrics.go`
- **Sample Controller**: `staging/src/k8s.io/sample-controller/controller.go`
- **Health Check Framework**: `staging/src/k8s.io/controller-manager/pkg/healthz/healthz.go`
- **Leader Election**: `staging/src/k8s.io/client-go/tools/leaderelection/`
- **Profiling Setup**: `staging/src/k8s.io/controller-manager/app/serve.go`

### **Related Documentation**
- **Controller Manager Guide**: `docs/architecture/claude/controller-manager/`
- **Controller Runtime**: `docs/architecture/claude/controller-manager/65-controller-runtime-guide.md`
- **Metrics Architecture**: `docs/architecture/claude/observability/01-metrics-and-dashboards.md`
- **Logging Architecture**: `docs/architecture/claude/observability/02-logging-and-analysis.md`
- **Tracing**: `docs/architecture/claude/observability/03-tracing-and-profiling.md`

### **External Resources**
- **Prometheus Best Practices**: https://prometheus.io/docs/practices/naming/
- **Controller Runtime**: https://github.com/kubernetes-sigs/controller-runtime
- **Kubebuilder**: https://book.kubebuilder.io/
- **Go Profiling**: https://go.dev/blog/pprof

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: Complete
**Last Updated**: 2024-11-17
**Target Audience**: Platform engineers, controller developers, SREs
**Scope**: Production-ready controller observability patterns
