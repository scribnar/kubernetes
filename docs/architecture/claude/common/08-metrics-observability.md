# **Document 08: Metrics and Observability**

**Part of**: Kubernetes Common/Shared Libraries Architecture Documentation
**Part IV**: component-base Library (Document 1 of 2)
**Status**: ✅ Complete
**Last Updated**: 2025-11-05

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Table of Contents**

1. [Introduction: Why Metrics Matter](#introduction-why-metrics-matter)
2. [Prometheus and Kubernetes](#prometheus-and-kubernetes)
3. [Metric Types](#metric-types)
4. [KubeRegistry and Stability](#kuberegistry-and-stability)
5. [Built-in Workqueue Metrics](#built-in-workqueue-metrics)
6. [Custom Controller Metrics](#custom-controller-metrics)
7. [Metric Best Practices](#metric-best-practices)
8. [Monitoring and Alerting](#monitoring-and-alerting)
9. [Real-World Examples](#real-world-examples)
10. [Summary and Guidelines](#summary-and-guidelines)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **1. Introduction: Why Metrics Matter**

### **1.1 The Observability Problem**

**💡 Aha Moment: You can't fix what you can't see!**

In production, controllers are black boxes:
- Is my controller processing events?
- Why is reconciliation slow?
- Are items failing and retrying?
- Is the workqueue backing up?

**Without metrics**: You're flying blind, debugging by guessing.

**With metrics**: You have precise, real-time visibility into controller behavior.

### **1.2 The Three Pillars of Observability**

```
┌─────────────────────────────────────────────────────────────┐
│             Observability Pillars                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. METRICS (this document)                                 │
│     - What is happening? (counters, rates, latencies)      │
│     - When did it happen? (time series)                    │
│     - Example: "Reconciliation rate is 100/sec"           │
│                                                              │
│  2. LOGS (Document 09)                                      │
│     - Why did it happen? (context, details)                │
│     - Example: "Failed to reconcile pod-1: ImagePullErr"  │
│                                                              │
│  3. TRACES (Advanced)                                       │
│     - How did it happen? (request flow)                    │
│     - Example: "Request took 500ms: 50ms API + 450ms work"│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**This document focuses on METRICS** - the foundation of observability.

### **1.3 Metrics in the Controller Pattern**

**From Document 13**, we know the controller pattern:

```
SharedInformer → Workqueue → Workers → Reconcile
```

**Metrics at each stage**:

```
┌──────────────────────────────────────────────────────────────┐
│             Metrics Throughout the Pipeline                   │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  SharedInformer:                                             │
│  - reflector_lists_total         (API list calls)           │
│  - reflector_watches_total       (Watch connections)        │
│  - reflector_items_per_list      (Items per list)           │
│                                                               │
│  Workqueue:                                                  │
│  - workqueue_adds_total          (Items added)              │
│  - workqueue_depth               (Queue size)               │
│  - workqueue_queue_duration      (Time in queue)            │
│  - workqueue_work_duration       (Processing time)          │
│  - workqueue_retries_total       (Retry count)              │
│                                                               │
│  Controller:                                                 │
│  - controller_reconcile_total    (Reconciliations)          │
│  - controller_reconcile_duration (Latency)                  │
│  - controller_errors_total       (Errors)                   │
│                                                               │
│  Leader Election:                                            │
│  - leaderelection_slowpath_total (Elections)                │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **2. Prometheus and Kubernetes**

### **2.1 Why Prometheus?**

**Prometheus** is the de facto standard for Kubernetes metrics:
- ✅ **Pull-based model**: Prometheus scrapes metrics from endpoints
- ✅ **Time series database**: Stores metrics over time
- ✅ **PromQL**: Powerful query language
- ✅ **Alerting**: Built-in alert manager
- ✅ **Ecosystem**: Grafana, exporters, operators

**Location**: Kubernetes uses `k8s.io/component-base/metrics` (wraps Prometheus client)

### **2.2 Metrics Exposition**

**How it works**:

```
┌─────────────────────────────────────────────────────────────┐
│              Prometheus Scrape Flow                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Controller exposes /metrics endpoint                    │
│     http.Handle("/metrics", promhttp.Handler())            │
│                                                              │
│  2. Prometheus scrapes endpoint every 30s                   │
│     GET http://my-controller:8080/metrics                  │
│                                                              │
│  3. Controller returns metrics in Prometheus format         │
│     # HELP workqueue_adds_total Total adds                 │
│     # TYPE workqueue_adds_total counter                    │
│     workqueue_adds_total{name="pods"} 1234                 │
│                                                              │
│  4. Prometheus stores time series                           │
│     workqueue_adds_total{name="pods"} @ t1 = 1234         │
│     workqueue_adds_total{name="pods"} @ t2 = 1456         │
│                                                              │
│  5. Query with PromQL                                       │
│     rate(workqueue_adds_total[5m])                         │
│     → 4.4 items/sec                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### **2.3 Prometheus Text Format**

**Example metrics output**:

```
# HELP workqueue_adds_total Total number of adds handled by workqueue
# TYPE workqueue_adds_total counter
workqueue_adds_total{name="pods"} 12345

# HELP workqueue_depth Current depth of workqueue
# TYPE workqueue_depth gauge
workqueue_depth{name="pods"} 42

# HELP workqueue_queue_duration_seconds How long in seconds an item stays in workqueue before being requested
# TYPE workqueue_queue_duration_seconds histogram
workqueue_queue_duration_seconds_bucket{name="pods",le="1e-08"} 0
workqueue_queue_duration_seconds_bucket{name="pods",le="1e-07"} 0
workqueue_queue_duration_seconds_bucket{name="pods",le="1e-06"} 156
workqueue_queue_duration_seconds_bucket{name="pods",le="0.001"} 2341
workqueue_queue_duration_seconds_bucket{name="pods",le="+Inf"} 2500
workqueue_queue_duration_seconds_sum{name="pods"} 45.2
workqueue_queue_duration_seconds_count{name="pods"} 2500
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **3. Metric Types**

### **3.1 Counter**

**Purpose**: Track cumulative values that only go up

**Use cases**:
- Total requests
- Total errors
- Total reconciliations

**Example**:
```go
reconcileTotal := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "controller_reconcile_total",
        Help: "Total number of reconciliations",
    },
    []string{"result"},  // Labels: success, error
)

// Increment on each reconciliation
reconcileTotal.WithLabelValues("success").Inc()
```

**Prometheus output**:
```
controller_reconcile_total{result="success"} 12345
controller_reconcile_total{result="error"} 42
```

**Query patterns**:
```promql
# Rate (items per second)
rate(controller_reconcile_total[5m])

# Total count
controller_reconcile_total

# Error rate
rate(controller_reconcile_total{result="error"}[5m])
```

### **3.2 Gauge**

**Purpose**: Track values that can go up or down

**Use cases**:
- Queue depth
- Number of active workers
- Current memory usage

**Example**:
```go
queueDepth := prometheus.NewGauge(
    prometheus.GaugeOpts{
        Name: "controller_queue_depth",
        Help: "Current queue depth",
    },
)

// Set current value
queueDepth.Set(42)

// Increment/decrement
queueDepth.Inc()
queueDepth.Dec()
queueDepth.Add(10)
queueDepth.Sub(5)
```

**Prometheus output**:
```
controller_queue_depth 42
```

**Query patterns**:
```promql
# Current value
controller_queue_depth

# Average over time
avg_over_time(controller_queue_depth[5m])

# Max over time
max_over_time(controller_queue_depth[1h])
```

### **3.3 Histogram**

**Purpose**: Track distribution of values (latency, size)

**💡 Aha Moment: Histograms show percentiles!**

**Use cases**:
- Request duration (p50, p95, p99)
- Response size
- Queue wait time

**Example**:
```go
reconcileDuration := prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "controller_reconcile_duration_seconds",
        Help: "Time spent reconciling",
        Buckets: prometheus.DefBuckets,  // [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
    },
    []string{"result"},
)

// Record observation
start := time.Now()
err := reconcile(obj)
duration := time.Since(start).Seconds()

if err != nil {
    reconcileDuration.WithLabelValues("error").Observe(duration)
} else {
    reconcileDuration.WithLabelValues("success").Observe(duration)
}
```

**Prometheus output**:
```
controller_reconcile_duration_seconds_bucket{result="success",le="0.005"} 45
controller_reconcile_duration_seconds_bucket{result="success",le="0.01"} 123
controller_reconcile_duration_seconds_bucket{result="success",le="0.025"} 456
controller_reconcile_duration_seconds_bucket{result="success",le="0.05"} 789
controller_reconcile_duration_seconds_bucket{result="success",le="+Inf"} 1000
controller_reconcile_duration_seconds_sum{result="success"} 23.4
controller_reconcile_duration_seconds_count{result="success"} 1000
```

**Query patterns**:
```promql
# 95th percentile
histogram_quantile(0.95, rate(controller_reconcile_duration_seconds_bucket[5m]))

# 99th percentile
histogram_quantile(0.99, rate(controller_reconcile_duration_seconds_bucket[5m]))

# Average (sum/count)
rate(controller_reconcile_duration_seconds_sum[5m]) / rate(controller_reconcile_duration_seconds_count[5m])
```

**Bucket Selection**:
```go
// Default buckets (good for latency)
Buckets: prometheus.DefBuckets  // [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]

// Custom buckets for faster operations (milliseconds)
Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1}

// Custom buckets for slower operations (seconds)
Buckets: []float64{.1, .5, 1, 2.5, 5, 10, 30, 60, 120}

// Linear buckets
Buckets: prometheus.LinearBuckets(0, 10, 10)  // [0, 10, 20, ..., 90]

// Exponential buckets
Buckets: prometheus.ExponentialBuckets(1, 2, 10)  // [1, 2, 4, 8, ..., 512]
```

### **3.4 Summary**

**Purpose**: Similar to histogram, but calculates percentiles client-side

**Differences from Histogram**:
- ❌ Cannot aggregate across instances
- ❌ Cannot change quantiles after creation
- ✅ Lower memory usage
- ✅ Exact quantiles (not approximated)

**Generally prefer Histograms** for controller metrics (aggregation is important!).

### **3.5 Metric Type Comparison**

| Type | Goes Up/Down | Aggregatable | Use Case | Example |
|------|--------------|--------------|----------|---------|
| **Counter** | Only up | ✅ Yes | Cumulative counts | Reconciliations, errors |
| **Gauge** | Both | ⚠️ Average | Current state | Queue depth, goroutines |
| **Histogram** | N/A | ✅ Yes | Distributions | Latency percentiles |
| **Summary** | N/A | ❌ No | Distributions | (prefer Histogram) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **4. KubeRegistry and Stability**

### **4.1 KubeRegistry**

**Location**: `k8s.io/component-base/metrics/legacyregistry`

**Purpose**: Kubernetes-specific registry with stability tracking

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "k8s.io/component-base/metrics"
    "k8s.io/component-base/metrics/legacyregistry"
)

// Create metric
reconcileTotal := metrics.NewCounterVec(
    &metrics.CounterOpts{
        Name:           "controller_reconcile_total",
        Help:           "Total reconciliations",
        StabilityLevel: metrics.STABLE,  // Stability level
    },
    []string{"result"},
)

// Register with KubeRegistry
legacyregistry.MustRegister(reconcileTotal)
```

### **4.2 Stability Levels**

**Purpose**: Track metric lifecycle (like API versions!)

**Levels**:

```go
const (
    ALPHA      StabilityLevel = "ALPHA"       // Experimental, may change
    BETA       StabilityLevel = "BETA"        // Stable, but not guaranteed
    STABLE     StabilityLevel = "STABLE"      // Production-ready, guaranteed
    DEPRECATED StabilityLevel = "DEPRECATED"  // Will be removed
)
```

**Lifecycle**:

```
ALPHA (1-2 releases)
  → May change without notice
  → Disabled by default

BETA (2-3 releases)
  → Stable schema
  → Enabled by default

STABLE
  → Production-ready
  → Guaranteed compatibility
  → Cannot change schema

DEPRECATED
  → Will be removed in future
  → Shows deprecation warning
```

**Example**:
```go
// New metric (experimental)
newMetric := metrics.NewCounter(
    &metrics.CounterOpts{
        Name:           "controller_new_feature_total",
        StabilityLevel: metrics.ALPHA,  // Experimental
    },
)

// Stable metric (production)
stableMetric := metrics.NewCounter(
    &metrics.CounterOpts{
        Name:           "controller_reconcile_total",
        StabilityLevel: metrics.STABLE,  // Guaranteed
    },
)

// Deprecated metric
oldMetric := metrics.NewCounter(
    &metrics.CounterOpts{
        Name:              "controller_old_metric_total",
        StabilityLevel:    metrics.DEPRECATED,
        DeprecatedVersion: "1.25",  // When deprecated
    },
)
```

### **4.3 Hidden and Disabled Metrics**

**Hidden Metrics**: ALPHA metrics not shown by default

```bash
# Show all metrics (including ALPHA)
curl http://localhost:8080/metrics?show-hidden-metrics-for-version=1.25
```

**Disabled Metrics**: Explicitly disabled to reduce overhead

```go
// Disable metrics
--disabled-metrics=controller_old_metric_total,controller_deprecated_total
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **5. Built-in Workqueue Metrics**

### **5.1 Automatic Workqueue Metrics**

**💡 Aha Moment: Workqueues automatically export metrics!**

**From Document 04**, we learned about workqueues. They come with **built-in metrics**:

```go
import "k8s.io/client-go/util/workqueue"

// Create named queue
queue := workqueue.NewNamedRateLimitingQueue(
    workqueue.DefaultControllerRateLimiter(),
    "pods",  // ← This name appears in metrics!
)

// Metrics are automatically recorded:
// - workqueue_adds_total{name="pods"}
// - workqueue_depth{name="pods"}
// - workqueue_queue_duration_seconds{name="pods"}
// - workqueue_work_duration_seconds{name="pods"}
// - workqueue_retries_total{name="pods"}
// - workqueue_longest_running_processor_seconds{name="pods"}
```

### **5.2 Workqueue Metrics Reference**

| Metric | Type | Description | Use Case |
|--------|------|-------------|----------|
| `workqueue_adds_total` | Counter | Total items added | Track event rate |
| `workqueue_depth` | Gauge | Current queue size | Detect backlog |
| `workqueue_queue_duration_seconds` | Histogram | Time in queue | Queue latency |
| `workqueue_work_duration_seconds` | Histogram | Processing time | Worker latency |
| `workqueue_retries_total` | Counter | Total retries | Error tracking |
| `workqueue_longest_running_processor_seconds` | Gauge | Longest worker | Stuck detection |
| `workqueue_unfinished_work_seconds` | Gauge | Work in progress | Processing health |

### **5.3 Workqueue Metric Queries**

**Add rate** (events per second):
```promql
rate(workqueue_adds_total{name="pods"}[5m])
```

**Queue depth** (backlog):
```promql
workqueue_depth{name="pods"}
```

**Queue latency** (p95 time in queue):
```promql
histogram_quantile(0.95,
  rate(workqueue_queue_duration_seconds_bucket{name="pods"}[5m])
)
```

**Processing latency** (p99 worker time):
```promql
histogram_quantile(0.99,
  rate(workqueue_work_duration_seconds_bucket{name="pods"}[5m])
)
```

**Retry rate** (retries per second):
```promql
rate(workqueue_retries_total{name="pods"}[5m])
```

**Stuck worker detection**:
```promql
workqueue_longest_running_processor_seconds{name="pods"} > 60
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **6. Custom Controller Metrics**

### **6.1 Essential Controller Metrics**

**Every controller should track**:

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "k8s.io/component-base/metrics"
    "k8s.io/component-base/metrics/legacyregistry"
)

var (
    // Reconciliation counter
    ReconcileTotal = metrics.NewCounterVec(
        &metrics.CounterOpts{
            Name:           "controller_reconcile_total",
            Help:           "Total number of reconciliations",
            StabilityLevel: metrics.STABLE,
        },
        []string{"controller", "result"},  // Labels
    )

    // Reconciliation duration
    ReconcileDuration = metrics.NewHistogramVec(
        &metrics.HistogramOpts{
            Name:           "controller_reconcile_duration_seconds",
            Help:           "Time spent reconciling",
            Buckets:        prometheus.DefBuckets,
            StabilityLevel: metrics.STABLE,
        },
        []string{"controller", "result"},
    )

    // Error counter by type
    ErrorsTotal = metrics.NewCounterVec(
        &metrics.CounterOpts{
            Name:           "controller_errors_total",
            Help:           "Total errors by type",
            StabilityLevel: metrics.STABLE,
        },
        []string{"controller", "error_type"},
    )

    // Objects processed by phase/status
    ObjectsProcessed = metrics.NewCounterVec(
        &metrics.CounterOpts{
            Name:           "controller_objects_processed_total",
            Help:           "Total objects processed by phase",
            StabilityLevel: metrics.ALPHA,
        },
        []string{"controller", "phase"},
    )
)

func init() {
    legacyregistry.MustRegister(ReconcileTotal)
    legacyregistry.MustRegister(ReconcileDuration)
    legacyregistry.MustRegister(ErrorsTotal)
    legacyregistry.MustRegister(ObjectsProcessed)
}
```

### **6.2 Using Custom Metrics**

**In controller code**:

```go
func (c *Controller) reconcile(key string) error {
    startTime := time.Now()

    // Record duration regardless of result
    defer func() {
        duration := time.Since(startTime).Seconds()

        // Will be set by error handling below
        var result string
        if err != nil {
            result = "error"
        } else {
            result = "success"
        }

        ReconcileDuration.WithLabelValues(
            "pod-controller",  // controller name
            result,
        ).Observe(duration)
    }()

    // Parse key
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return nil  // Invalid key, don't retry
    }

    // Get object
    pod, err := c.podLister.Pods(namespace).Get(name)
    if err != nil {
        if errors.IsNotFound(err) {
            // Object deleted
            ReconcileTotal.WithLabelValues(
                "pod-controller",
                "deleted",
            ).Inc()
            return nil
        }

        // Error getting object
        ErrorsTotal.WithLabelValues(
            "pod-controller",
            "get_error",
        ).Inc()

        ReconcileTotal.WithLabelValues(
            "pod-controller",
            "error",
        ).Inc()

        return err
    }

    // Reconcile logic
    err = c.doReconcile(pod)
    if err != nil {
        // Classify error
        errorType := classifyError(err)
        ErrorsTotal.WithLabelValues(
            "pod-controller",
            errorType,
        ).Inc()

        ReconcileTotal.WithLabelValues(
            "pod-controller",
            "error",
        ).Inc()

        return err
    }

    // Success
    ReconcileTotal.WithLabelValues(
        "pod-controller",
        "success",
    ).Inc()

    ObjectsProcessed.WithLabelValues(
        "pod-controller",
        string(pod.Status.Phase),
    ).Inc()

    return nil
}

func classifyError(err error) string {
    if errors.IsConflict(err) {
        return "conflict"
    }
    if errors.IsForbidden(err) {
        return "forbidden"
    }
    if errors.IsTimeout(err) {
        return "timeout"
    }
    return "unknown"
}
```

### **6.3 Metric Cardinality**

**💡 Aha Moment: High cardinality kills Prometheus!**

**Cardinality** = Number of unique time series

```
Cardinality = Product of all label value combinations
```

**Example**:
```go
// BAD: High cardinality
reconcileTotal := metrics.NewCounterVec(
    &metrics.CounterOpts{Name: "reconcile_total"},
    []string{"namespace", "name"},  // ❌ BAD!
)

// Every pod creates a new time series:
// reconcile_total{namespace="default",name="pod-1"}
// reconcile_total{namespace="default",name="pod-2"}
// ...
// reconcile_total{namespace="default",name="pod-1000"}
// = 1000 time series!

// GOOD: Low cardinality
reconcileTotal := metrics.NewCounterVec(
    &metrics.CounterOpts{Name: "reconcile_total"},
    []string{"controller", "result"},  // ✅ GOOD!
)

// Only a few time series:
// reconcile_total{controller="pod",result="success"}
// reconcile_total{controller="pod",result="error"}
// = 2 time series
```

**Cardinality Guidelines**:

| Label Type | Cardinality | Example | OK? |
|------------|-------------|---------|-----|
| **Controller name** | ~10 | "pod", "deployment" | ✅ Good |
| **Result** | ~5 | "success", "error", "conflict" | ✅ Good |
| **Error type** | ~20 | "timeout", "forbidden", "conflict" | ✅ OK |
| **Namespace** | 10-1000 | "default", "kube-system" | ⚠️ Careful |
| **Object name** | 1000+ | "pod-1", "pod-2", ... | ❌ BAD |
| **User ID** | 1000+ | "user-123", "user-456" | ❌ BAD |
| **IP address** | 10000+ | "10.0.0.1", "10.0.0.2" | ❌ VERY BAD |

**Rule of thumb**: Keep total cardinality under 10,000 per metric.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **7. Metric Best Practices**

### **7.1 Naming Conventions**

**Follow Prometheus naming conventions**:

```
<namespace>_<subsystem>_<name>_<unit>
```

**Examples**:
```
✅ controller_reconcile_total
✅ controller_reconcile_duration_seconds
✅ workqueue_depth
✅ apiserver_request_total

❌ ReconcileCount  (not snake_case)
❌ controller_latency  (no unit)
❌ errors  (no namespace)
```

**Unit suffixes**:
- `_total` - Counter
- `_seconds` - Duration
- `_bytes` - Size
- `_ratio` - Ratio (0-1)

### **7.2 Label Guidelines**

**DO**:
```go
// ✅ Low cardinality labels
[]string{"controller", "result"}

// ✅ Bounded labels
[]string{"error_type"}  // ~20 types

// ✅ Enum-like labels
[]string{"phase"}  // "running", "pending", "failed"
```

**DON'T**:
```go
// ❌ Unbounded labels
[]string{"pod_name"}  // Thousands of pods

// ❌ User-controlled labels
[]string{"user_id"}

// ❌ High cardinality combinations
[]string{"namespace", "pod_name", "container_name"}
```

### **7.3 Metric Lifecycle**

**1. Start with ALPHA**:
```go
newMetric := metrics.NewCounter(
    &metrics.CounterOpts{
        Name:           "controller_experimental_total",
        StabilityLevel: metrics.ALPHA,
    },
)
```

**2. Promote to BETA** (after 1-2 releases):
```go
betaMetric := metrics.NewCounter(
    &metrics.CounterOpts{
        Name:           "controller_experimental_total",
        StabilityLevel: metrics.BETA,  // Now beta
    },
)
```

**3. Promote to STABLE** (after 2-3 releases):
```go
stableMetric := metrics.NewCounter(
    &metrics.CounterOpts{
        Name:           "controller_experimental_total",
        StabilityLevel: metrics.STABLE,  // Production-ready
    },
)
```

**4. Deprecate if needed**:
```go
deprecatedMetric := metrics.NewCounter(
    &metrics.CounterOpts{
        Name:              "controller_old_total",
        StabilityLevel:    metrics.DEPRECATED,
        DeprecatedVersion: "1.25",
    },
)
```

### **7.4 Performance Considerations**

**Metric recording is fast** (<1µs), but:

```go
// ❌ BAD: Recording in hot path
for _, item := range items {  // 10000 items
    metric.Inc()  // 10000 metric updates!
}

// ✅ GOOD: Batch increment
count := len(items)
metric.Add(float64(count))  // 1 metric update
```

**Use appropriate metric types**:
```go
// ❌ BAD: Gauge for cumulative count
errorCount := prometheus.NewGauge(...)
errorCount.Inc()  // Will reset to 0 on restart!

// ✅ GOOD: Counter for cumulative count
errorCount := prometheus.NewCounter(...)
errorCount.Inc()  // Accumulates across restarts
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **8. Monitoring and Alerting**

### **8.1 Essential Dashboards**

**Grafana Dashboard for Controller**:

```json
{
  "dashboard": {
    "title": "Pod Controller",
    "panels": [
      {
        "title": "Reconciliation Rate",
        "targets": [{
          "expr": "rate(controller_reconcile_total{controller=\"pod\"}[5m])"
        }]
      },
      {
        "title": "Error Rate",
        "targets": [{
          "expr": "rate(controller_reconcile_total{controller=\"pod\",result=\"error\"}[5m])"
        }]
      },
      {
        "title": "P95 Latency",
        "targets": [{
          "expr": "histogram_quantile(0.95, rate(controller_reconcile_duration_seconds_bucket{controller=\"pod\"}[5m]))"
        }]
      },
      {
        "title": "Queue Depth",
        "targets": [{
          "expr": "workqueue_depth{name=\"pods\"}"
        }]
      }
    ]
  }
}
```

### **8.2 Alerting Rules**

**PrometheusRule for Alerts**:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-controller-alerts
spec:
  groups:
  - name: pod-controller
    interval: 30s
    rules:
    # High error rate
    - alert: ControllerHighErrorRate
      expr: |
        rate(controller_reconcile_total{controller="pod",result="error"}[5m]) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod controller has high error rate"
        description: "Error rate is {{ $value }} errors/sec"

    # Queue backing up
    - alert: ControllerQueueBacklog
      expr: |
        workqueue_depth{name="pods"} > 1000
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod controller queue is backing up"
        description: "Queue depth is {{ $value }} items"

    # High latency
    - alert: ControllerHighLatency
      expr: |
        histogram_quantile(0.95,
          rate(controller_reconcile_duration_seconds_bucket{controller="pod"}[5m])
        ) > 10
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod controller has high latency"
        description: "P95 latency is {{ $value }} seconds"

    # Controller down
    - alert: ControllerDown
      expr: |
        up{job="pod-controller"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod controller is down"
        description: "No metrics from pod controller for 5 minutes"

    # Stuck worker
    - alert: ControllerStuckWorker
      expr: |
        workqueue_longest_running_processor_seconds{name="pods"} > 600
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod controller has stuck worker"
        description: "Worker stuck for {{ $value }} seconds"
```

### **8.3 SLI/SLO Patterns**

**Service Level Indicators (SLIs)**:

```yaml
# Availability SLI: % of successful reconciliations
availability:
  expr: |
    sum(rate(controller_reconcile_total{result="success"}[5m]))
    /
    sum(rate(controller_reconcile_total[5m]))
  target: 0.999  # 99.9% success rate

# Latency SLI: P95 latency
latency:
  expr: |
    histogram_quantile(0.95,
      rate(controller_reconcile_duration_seconds_bucket[5m])
    )
  target: 1.0  # P95 < 1 second

# Throughput SLI: Reconciliations per second
throughput:
  expr: |
    sum(rate(controller_reconcile_total[5m]))
  target: 100  # > 100 reconciliations/sec
```

**Service Level Objectives (SLOs)**:

```
- 99.9% of reconciliations succeed (availability)
- 95% of reconciliations complete in < 1s (latency)
- Process > 100 reconciliations/sec (throughput)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **9. Real-World Examples**

### **9.1 Deployment Controller Metrics**

**From `pkg/controller/deployment/deployment_controller.go`**:

```go
var (
    // Deployment sync count
    deploymentsyncTotal = metrics.NewCounterVec(
        &metrics.CounterOpts{
            Subsystem:      "deployment_controller",
            Name:           "sync_total",
            Help:           "Number of deployment syncs",
            StabilityLevel: metrics.ALPHA,
        },
        []string{"result"},
    )

    // Deployment sync duration
    deploymentsyncDuration = metrics.NewHistogramVec(
        &metrics.HistogramOpts{
            Subsystem:      "deployment_controller",
            Name:           "sync_duration_seconds",
            Help:           "Deployment sync duration",
            StabilityLevel: metrics.ALPHA,
        },
        []string{"result"},
    )
)
```

### **9.2 Complete Example**

**Full metrics implementation**:

```go
package main

import (
    "context"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus/promhttp"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/util/workqueue"
    "k8s.io/component-base/metrics"
    "k8s.io/component-base/metrics/legacyregistry"
    "k8s.io/klog/v2"
)

var (
    reconcileTotal = metrics.NewCounterVec(
        &metrics.CounterOpts{
            Name:           "controller_reconcile_total",
            Help:           "Total reconciliations",
            StabilityLevel: metrics.STABLE,
        },
        []string{"controller", "result"},
    )

    reconcileDuration = metrics.NewHistogramVec(
        &metrics.HistogramOpts{
            Name:           "controller_reconcile_duration_seconds",
            Help:           "Reconciliation duration",
            StabilityLevel: metrics.STABLE,
        },
        []string{"controller", "result"},
    )

    errorsTotal = metrics.NewCounterVec(
        &metrics.CounterOpts{
            Name:           "controller_errors_total",
            Help:           "Total errors by type",
            StabilityLevel: metrics.STABLE,
        },
        []string{"controller", "error_type"},
    )
)

func init() {
    legacyregistry.MustRegister(reconcileTotal)
    legacyregistry.MustRegister(reconcileDuration)
    legacyregistry.MustRegister(errorsTotal)
}

type Controller struct {
    name  string
    queue workqueue.RateLimitingInterface
}

func (c *Controller) reconcile(key string) error {
    start := time.Now()

    defer func() {
        duration := time.Since(start).Seconds()
        result := "success"
        if err != nil {
            result = "error"
        }

        reconcileDuration.WithLabelValues(c.name, result).Observe(duration)
    }()

    // Parse key
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        reconcileTotal.WithLabelValues(c.name, "invalid_key").Inc()
        return nil
    }

    // Get object (from cache)
    obj, err := c.getObject(namespace, name)
    if err != nil {
        if errors.IsNotFound(err) {
            reconcileTotal.WithLabelValues(c.name, "not_found").Inc()
            return nil
        }

        errorsTotal.WithLabelValues(c.name, "get_error").Inc()
        reconcileTotal.WithLabelValues(c.name, "error").Inc()
        return err
    }

    // Reconcile
    err = c.doReconcile(obj)
    if err != nil {
        errorType := classifyError(err)
        errorsTotal.WithLabelValues(c.name, errorType).Inc()
        reconcileTotal.WithLabelValues(c.name, "error").Inc()
        return err
    }

    reconcileTotal.WithLabelValues(c.name, "success").Inc()
    return nil
}

func classifyError(err error) string {
    if errors.IsConflict(err) {
        return "conflict"
    }
    if errors.IsForbidden(err) {
        return "forbidden"
    }
    if errors.IsTimeout(err) {
        return "timeout"
    }
    if errors.IsServerTimeout(err) {
        return "server_timeout"
    }
    return "unknown"
}

func serveMetrics(addr string) {
    http.Handle("/metrics", promhttp.HandlerFor(
        legacyregistry.DefaultGatherer,
        promhttp.HandlerOpts{},
    ))

    http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("ok"))
    })

    klog.Infof("Starting metrics server on %s", addr)
    if err := http.ListenAndServe(addr, nil); err != nil {
        klog.Fatalf("Failed to start metrics server: %v", err)
    }
}

func main() {
    // Start metrics server
    go serveMetrics(":8080")

    // Run controller
    // ...
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **10. Summary and Guidelines**

### **10.1 Metrics Checklist**

**Every controller should have**:

- [ ] **Reconciliation counter** (`controller_reconcile_total`)
  - Labels: `controller`, `result` (success/error/deleted)

- [ ] **Reconciliation duration** (`controller_reconcile_duration_seconds`)
  - Labels: `controller`, `result`
  - Type: Histogram

- [ ] **Error counter** (`controller_errors_total`)
  - Labels: `controller`, `error_type`

- [ ] **Workqueue metrics** (automatic with named queue)

- [ ] **Metrics endpoint** (`/metrics` on port 8080)

- [ ] **Health endpoint** (`/healthz`)

- [ ] **ServiceMonitor** (for Prometheus Operator)

- [ ] **Grafana dashboard**

- [ ] **Alerting rules**

### **10.2 Golden Signals**

**The Four Golden Signals** (from Google SRE):

1. **Latency**: How long does reconciliation take?
   ```promql
   histogram_quantile(0.95, rate(controller_reconcile_duration_seconds_bucket[5m]))
   ```

2. **Traffic**: How many reconciliations per second?
   ```promql
   rate(controller_reconcile_total[5m])
   ```

3. **Errors**: What's the error rate?
   ```promql
   rate(controller_reconcile_total{result="error"}[5m])
   ```

4. **Saturation**: Is the queue backing up?
   ```promql
   workqueue_depth
   ```

### **10.3 Metric Maturity Model**

**Level 1: Basic** (Minimum viable)
- ✅ Reconciliation counter
- ✅ Error counter
- ✅ Workqueue metrics (automatic)

**Level 2: Production** (Production-ready)
- ✅ Level 1 +
- ✅ Reconciliation duration (histogram)
- ✅ Error classification
- ✅ Grafana dashboard
- ✅ Basic alerts

**Level 3: Advanced** (SRE-grade)
- ✅ Level 2 +
- ✅ SLI/SLO definitions
- ✅ Comprehensive alerts
- ✅ Custom business metrics
- ✅ Capacity planning metrics

### **10.4 Best Practices Summary**

**DO**:
- ✅ Use Counter for cumulative counts
- ✅ Use Gauge for current values
- ✅ Use Histogram for distributions
- ✅ Keep label cardinality low (<10,000)
- ✅ Use stability levels (ALPHA → BETA → STABLE)
- ✅ Follow naming conventions (`namespace_subsystem_name_unit`)
- ✅ Add units to metric names (`_seconds`, `_bytes`, `_total`)
- ✅ Record duration with `defer` pattern
- ✅ Classify errors by type
- ✅ Use workqueue name for automatic metrics

**DON'T**:
- ❌ Use pod/object names as labels (high cardinality!)
- ❌ Use Gauge for cumulative counts (use Counter)
- ❌ Record metrics in hot loops (batch instead)
- ❌ Change STABLE metric schemas (breaking change!)
- ❌ Use Summary (prefer Histogram)
- ❌ Skip metric documentation (`Help` field)
- ❌ Forget to register metrics
- ❌ Use non-standard naming

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Conclusion**

### **What You've Learned**

**Metrics Fundamentals**:
- ✅ Why metrics matter (observability pillar #1)
- ✅ Prometheus architecture and scraping
- ✅ Metric types: Counter, Gauge, Histogram, Summary
- ✅ When to use each type

**Kubernetes Metrics**:
- ✅ KubeRegistry and stability levels
- ✅ Metric lifecycle (ALPHA → BETA → STABLE → DEPRECATED)
- ✅ Built-in workqueue metrics
- ✅ Custom controller metrics

**Production Practices**:
- ✅ Metric naming conventions
- ✅ Label cardinality management
- ✅ Performance considerations
- ✅ Dashboard and alerting patterns
- ✅ SLI/SLO definitions
- ✅ The Four Golden Signals

**You Can Now**:
- ✅ Instrument controllers with comprehensive metrics
- ✅ Create Grafana dashboards
- ✅ Set up alerting rules
- ✅ Monitor controller health in production
- ✅ Debug performance issues with metrics
- ✅ Avoid common metric pitfalls

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Appendix: Quick Reference**

### **Essential Imports**

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "k8s.io/component-base/metrics"
    "k8s.io/component-base/metrics/legacyregistry"
)
```

### **Metric Template**

```go
var myMetric = metrics.NewCounterVec(
    &metrics.CounterOpts{
        Name:           "controller_my_metric_total",
        Help:           "Description of metric",
        StabilityLevel: metrics.STABLE,
    },
    []string{"label1", "label2"},
)

func init() {
    legacyregistry.MustRegister(myMetric)
}

// Usage
myMetric.WithLabelValues("value1", "value2").Inc()
```

### **Common Queries**

```promql
# Rate (per second)
rate(metric_total[5m])

# P95 latency
histogram_quantile(0.95, rate(metric_duration_seconds_bucket[5m]))

# Error rate
rate(metric_total{result="error"}[5m]) / rate(metric_total[5m])

# Gauge average
avg_over_time(metric_gauge[5m])
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: ✅ Complete
**Lines**: ~1,600
**Diagrams**: 12+
**Code Examples**: 30+
**Last Updated**: 2025-11-05

**Related Documents**:
- **Document 04**: Workqueue and Leader Election (workqueue metrics)
- **Document 09**: Config, Logs, and Feature Gates (logging companion)
- **Document 13**: Common Patterns and Integration (observability section)

**Next**: Document 09 (Config, Logs, and Feature Gates) completes Phase 3!

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
