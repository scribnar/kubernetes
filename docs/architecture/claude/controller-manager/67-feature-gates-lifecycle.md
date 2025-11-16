# Feature Gates Lifecycle Management

**Document**: 67-feature-gates-lifecycle.md
**Status**: Course Module - Feature Development
**Audience**: Contributors, Platform Engineers
**Prerequisites**: Kubernetes development, API versioning

---

## **Overview**

Feature gates enable gradual rollout of new features through Alpha → Beta → GA stages, allowing safe experimentation and rollback.

### **Feature Gate Lifecycle**

```mermaid
stateDiagram-v2
    [*] --> Alpha
    Alpha --> Beta: Feedback positive
    Alpha --> Removed: Feedback negative
    Beta --> GA: Stable
    Beta --> Deprecated: Issues found
    GA --> Deprecated: Better alternative
    Deprecated --> Removed
    Removed --> [*]

    note right of Alpha: Default: disabled<br/>May be buggy<br/>Can change
    note right of Beta: Default: enabled<br/>Well tested<br/>Mostly stable
    note right of GA: Always on<br/>Cannot disable<br/>Stable
```

---

## **1. Feature Gate Stages**

### **1.1 Alpha**

**Characteristics**:
- Disabled by default
- May be buggy
- No guarantees of backward compatibility
- May be removed without notice
- Not recommended for production

**Example**:
```go
// pkg/features/kube_features.go
const (
    // Alpha: Kubernetes v1.25
    DynamicResourceAllocation featuregate.Feature = "DynamicResourceAllocation"
)

var defaultKubernetesFeatureGates = map[featuregate.Feature]featuregate.FeatureSpec{
    DynamicResourceAllocation: {Default: false, PreRelease: featuregate.Alpha},
}
```

**Enable**:
```bash
kube-controller-manager \
  --feature-gates=DynamicResourceAllocation=true
```

### **1.2 Beta**

**Characteristics**:
- Enabled by default
- Well tested
- Safe for production (with caution)
- Support for entire beta period
- May still have minor changes

**Example**:
```go
const (
    // Beta: Kubernetes v1.26
    DynamicResourceAllocation featuregate.Feature = "DynamicResourceAllocation"
)

var defaultKubernetesFeatureGates = map[featuregate.Feature]featuregate.FeatureSpec{
    DynamicResourceAllocation: {Default: true, PreRelease: featuregate.Beta},
}
```

### **1.3 GA (Generally Available)**

**Characteristics**:
- Always enabled
- Cannot be disabled
- Fully stable
- Feature gate removed (eventually)

**Example**:
```go
const (
    // GA: Kubernetes v1.27 (feature gate removed v1.29)
    // DynamicResourceAllocation - feature gate removed, always on
)
```

---

## **2. Implementing Feature-Gated Code**

### **2.1 Controller with Feature Gate**

```go
import (
    utilfeature "k8s.io/apiserver/pkg/util/feature"
    "k8s.io/kubernetes/pkg/features"
)

func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Check if feature is enabled
    if utilfeature.DefaultFeatureGate.Enabled(features.MyNewFeature) {
        return r.reconcileWithNewFeature(ctx, req)
    }

    // Fall back to old behavior
    return r.reconcileOldWay(ctx, req)
}

func (r *MyReconciler) reconcileWithNewFeature(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // New feature implementation
    log.Info("Using new feature")
    // ...
    return ctrl.Result{}, nil
}
```

### **2.2 Conditional Field in CRD**

```go
type MyResourceSpec struct {
    // Always present
    Image string `json:"image"`

    // Feature-gated field (alpha)
    // +optional
    // +featureGate=MyNewFeature
    AdvancedConfig *AdvancedConfig `json:"advancedConfig,omitempty"`
}

// Validation with feature gate
func (r *MyResource) ValidateCreate() error {
    if r.Spec.AdvancedConfig != nil {
        if !utilfeature.DefaultFeatureGate.Enabled(features.MyNewFeature) {
            return fmt.Errorf("advancedConfig requires MyNewFeature feature gate")
        }
    }
    return nil
}
```

---

## **3. Feature Gate Best Practices**

### **3.1 Graduation Criteria**

**Alpha → Beta**:
- [ ] E2E tests passing
- [ ] Unit test coverage > 80%
- [ ] No major bugs reported
- [ ] API review approved
- [ ] Documentation complete
- [ ] At least 2 releases in alpha

**Beta → GA**:
- [ ] In beta for at least 2 releases
- [ ] No major bugs in last release
- [ ] Performance benchmarks acceptable
- [ ] Conformance tests added
- [ ] Production usage validated
- [ ] Upgrade/downgrade tested

### **3.2 Feature Gate Checklist**

```go
// When adding new feature:

// 1. Add feature gate constant
const MyNewFeature featuregate.Feature = "MyNewFeature"

// 2. Register with default value
var defaultKubernetesFeatureGates = map[featuregate.Feature]featuregate.FeatureSpec{
    MyNewFeature: {Default: false, PreRelease: featuregate.Alpha},
}

// 3. Check feature gate before using
if !utilfeature.DefaultFeatureGate.Enabled(features.MyNewFeature) {
    return nil // Feature disabled
}

// 4. Add tests for both enabled and disabled
func TestWithFeatureEnabled(t *testing.T) {
    defer featuregatetesting.SetFeatureGateDuringTest(t, utilfeature.DefaultFeatureGate, features.MyNewFeature, true)()
    // Test with feature enabled
}

func TestWithFeatureDisabled(t *testing.T) {
    defer featuregatetesting.SetFeatureGateDuringTest(t, utilfeature.DefaultFeatureGate, features.MyNewFeature, false)()
    // Test with feature disabled
}
```

---

## **4. Feature Gate Metrics**

### **4.1 Track Feature Usage**

```go
var (
    featureUsageCount = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "feature_gate_usage_total",
            Help: "Number of times feature-gated code was executed",
        },
        []string{"feature", "component"},
    )
)

func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    if utilfeature.DefaultFeatureGate.Enabled(features.MyNewFeature) {
        featureUsageCount.WithLabelValues("MyNewFeature", "reconcile").Inc()
        return r.reconcileWithNewFeature(ctx, req)
    }
    return r.reconcileOldWay(ctx, req)
}
```

---

## **5. Deprecation Process**

### **5.1 Deprecation Timeline**

```go
// v1.25: Feature is GA, deprecate feature gate
var defaultKubernetesFeatureGates = map[featuregate.Feature]featuregate.FeatureSpec{
    OldFeature: {Default: true, PreRelease: featuregate.Deprecated},
}

// v1.26: Warn when feature gate is used
if utilfeature.DefaultFeatureGate.Enabled(features.OldFeature) {
    klog.Warning("OldFeature feature gate is deprecated and will be removed in v1.27")
}

// v1.27: Remove feature gate, code always on
// Remove from featuregate map
// Remove all feature gate checks from code
```

### **5.2 User Communication**

```yaml
# Release notes template for feature gates

## Feature Gates

### Graduated to Beta
- `MyNewFeature`: Now enabled by default. To disable: `--feature-gates=MyNewFeature=false`

### Graduated to GA
- `StableFeature`: Feature gate removed, always enabled

### Deprecated
- `OldFeature`: Will be removed in v1.27. Plan migration to NewFeature.

### Removed
- `VeryOldFeature`: Feature gate removed. Feature always enabled.
```

---

## **6. Testing Feature Gates**

### **6.1 Feature Gate Tests**

```go
func TestFeatureGateProgression(t *testing.T) {
    tests := []struct {
        name          string
        featureGate   featuregate.Feature
        enableFeature bool
        expectError   bool
    }{
        {
            name:          "Alpha feature disabled by default",
            featureGate:   features.AlphaFeature,
            enableFeature: false,
            expectError:   false,
        },
        {
            name:          "Alpha feature can be enabled",
            featureGate:   features.AlphaFeature,
            enableFeature: true,
            expectError:   false,
        },
        {
            name:          "Beta feature enabled by default",
            featureGate:   features.BetaFeature,
            enableFeature: true,
            expectError:   false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            defer featuregatetesting.SetFeatureGateDuringTest(
                t,
                utilfeature.DefaultFeatureGate,
                tt.featureGate,
                tt.enableFeature,
            )()

            // Test feature behavior
            enabled := utilfeature.DefaultFeatureGate.Enabled(tt.featureGate)
            if enabled != tt.enableFeature {
                t.Errorf("Expected feature %s enabled=%v, got %v",
                    tt.featureGate, tt.enableFeature, enabled)
            }
        })
    }
}
```

---

## **7. Real-World Examples**

### **7.1 Job Tracking with Finalizers (KEP-2307)**

**Timeline**:
- **v1.22**: Alpha (disabled by default)
- **v1.23**: Beta (enabled by default)
- **v1.26**: GA (always on)
- **v1.27**: Feature gate removed

**Code evolution**:
```go
// v1.22 (Alpha)
if utilfeature.DefaultFeatureGate.Enabled(features.JobTrackingWithFinalizers) {
    // Use finalizer-based tracking
} else {
    // Use legacy tracking
}

// v1.26 (GA)
// Always use finalizer-based tracking
// Feature gate check removed
```

### **7.2 Pod Security Standards (KEP-2579)**

**Timeline**:
- **v1.22**: Alpha
- **v1.23**: Beta
- **v1.25**: GA

---

## **8. Feature Gate Commands**

```bash
# List all feature gates
kube-controller-manager --feature-gates --help

# Enable specific feature
kube-controller-manager --feature-gates=MyFeature=true

# Enable multiple features
kube-controller-manager \
  --feature-gates=Feature1=true,Feature2=false,Feature3=true

# Check current feature gates
kubectl get --raw /metrics | grep feature_gate
```

---

## **Summary**

Feature gate lifecycle:
1. **Alpha**: Disabled by default, may be buggy
2. **Beta**: Enabled by default, production-ready
3. **GA**: Always on, stable
4. **Deprecated**: Preparing for removal
5. **Removed**: Feature gate gone

**Key practices**:
- Default alpha to disabled, beta to enabled
- Require at least 2 releases per stage
- Test with feature both enabled and disabled
- Document in release notes
- Communicate deprecation timeline
- Remove feature gates after GA+2 releases

Feature gates enable safe, gradual feature rollout!
