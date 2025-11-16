# Building Custom Controllers: A Practical Guide

**Document**: 63-custom-controllers-guide.md
**Status**: Course Module - Hands-On Development
**Audience**: Software Engineers, Platform Developers
**Prerequisites**: Go programming, Kubernetes basics, controller patterns

---

## **Overview**

This hands-on guide walks through building a production-ready custom controller from scratch. You'll learn the complete development workflow using real-world examples.

### **Learning Objectives**

1. Scaffold a custom controller project
2. Define Custom Resource Definitions (CRDs)
3. Implement controller reconciliation logic
4. Add validation and defaulting webhooks
5. Write comprehensive tests
6. Deploy to production

---

## **1. Project Setup**

### **1.1 Using Kubebuilder**

```bash
# Install kubebuilder
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/

# Create project
mkdir myapp-controller
cd myapp-controller
kubebuilder init --domain example.com --repo github.com/myorg/myapp-controller

# Create API (CRD + Controller)
kubebuilder create api \
  --group apps \
  --version v1alpha1 \
  --kind MyApp \
  --resource \
  --controller

# Project structure created:
# myapp-controller/
# ├── api/v1alpha1/
# │   ├── myapp_types.go          # CRD definition
# │   └── groupversion_info.go
# ├── controllers/
# │   └── myapp_controller.go     # Controller logic
# ├── config/
# │   ├── crd/                    # CRD manifests
# │   ├── manager/                # Deployment manifests
# │   └── rbac/                   # RBAC rules
# ├── Dockerfile
# └── Makefile
```

---

## **2. Define Custom Resource**

### **2.1 CRD Specification**

```go
// api/v1alpha1/myapp_types.go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// MyAppSpec defines the desired state
type MyAppSpec struct {
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinLength=1
    Image string `json:"image"`

    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    // +kubebuilder:default=3
    Replicas int32 `json:"replicas,omitempty"`

    // +kubebuilder:validation:Optional
    Resources ResourceRequirements `json:"resources,omitempty"`
}

type ResourceRequirements struct {
    CPU    string `json:"cpu,omitempty"`
    Memory string `json:"memory,omitempty"`
}

// MyAppStatus defines observed state
type MyAppStatus struct {
    // Current number of running replicas
    ReadyReplicas int32 `json:"readyReplicas"`

    // Conditions track the status
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Image",type=string,JSONPath=`.spec.image`
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`

type MyApp struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   MyAppSpec   `json:"spec,omitempty"`
    Status MyAppStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

type MyAppList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []MyApp `json:"items"`
}

func init() {
    SchemeBuilder.Register(&MyApp{}, &MyAppList{})
}
```

---

## **3. Controller Implementation**

### **3.1 Reconciliation Loop**

```go
// controllers/myapp_controller.go
package controllers

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"

    appsv1alpha1 "github.com/myorg/myapp-controller/api/v1alpha1"
)

type MyAppReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=apps.example.com,resources=myapps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps.example.com,resources=myapps/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete

func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // Fetch the MyApp instance
    myApp := &appsv1alpha1.MyApp{}
    err := r.Get(ctx, req.NamespacedName, myApp)
    if err != nil {
        if errors.IsNotFound(err) {
            log.Info("MyApp resource not found. Ignoring since object must be deleted")
            return ctrl.Result{}, nil
        }
        log.Error(err, "Failed to get MyApp")
        return ctrl.Result{}, err
    }

    // Reconcile Deployment
    if err := r.reconcileDeployment(ctx, myApp); err != nil {
        return ctrl.Result{}, err
    }

    // Reconcile Service
    if err := r.reconcileService(ctx, myApp); err != nil {
        return ctrl.Result{}, err
    }

    // Update status
    if err := r.updateStatus(ctx, myApp); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}

func (r *MyAppReconciler) reconcileDeployment(ctx context.Context, myApp *appsv1alpha1.MyApp) error {
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, client.ObjectKey{
        Namespace: myApp.Namespace,
        Name:      myApp.Name,
    }, deployment)

    if err != nil && errors.IsNotFound(err) {
        // Create deployment
        dep := r.deploymentForMyApp(myApp)
        if err := controllerutil.SetControllerReference(myApp, dep, r.Scheme); err != nil {
            return err
        }
        return r.Create(ctx, dep)
    } else if err != nil {
        return err
    }

    // Update if needed
    if deployment.Spec.Replicas == nil || *deployment.Spec.Replicas != myApp.Spec.Replicas {
        deployment.Spec.Replicas = &myApp.Spec.Replicas
        if err := r.Update(ctx, deployment); err != nil {
            return err
        }
    }

    return nil
}

func (r *MyAppReconciler) deploymentForMyApp(myApp *appsv1alpha1.MyApp) *appsv1.Deployment {
    labels := map[string]string{
        "app":        myApp.Name,
        "managed-by": "myapp-controller",
    }

    return &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      myApp.Name,
            Namespace: myApp.Namespace,
            Labels:    labels,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &myApp.Spec.Replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: labels,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: labels,
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "app",
                            Image: myApp.Spec.Image,
                            Ports: []corev1.ContainerPort{
                                {
                                    ContainerPort: 8080,
                                    Name:          "http",
                                },
                            },
                            Resources: r.getResourceRequirements(myApp),
                        },
                    },
                },
            },
        },
    }
}

func (r *MyAppReconciler) updateStatus(ctx context.Context, myApp *appsv1alpha1.MyApp) error {
    // Get deployment to check ready replicas
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, client.ObjectKey{
        Namespace: myApp.Namespace,
        Name:      myApp.Name,
    }, deployment)
    if err != nil {
        return err
    }

    // Update status
    myApp.Status.ReadyReplicas = deployment.Status.ReadyReplicas

    // Set condition
    condition := metav1.Condition{
        Type:               "Ready",
        Status:             metav1.ConditionTrue,
        Reason:             "DeploymentReady",
        Message:            fmt.Sprintf("%d/%d replicas ready", deployment.Status.ReadyReplicas, myApp.Spec.Replicas),
        LastTransitionTime: metav1.Now(),
    }

    if deployment.Status.ReadyReplicas < myApp.Spec.Replicas {
        condition.Status = metav1.ConditionFalse
        condition.Reason = "DeploymentNotReady"
    }

    // Update or append condition
    updated := false
    for i, c := range myApp.Status.Conditions {
        if c.Type == "Ready" {
            myApp.Status.Conditions[i] = condition
            updated = true
            break
        }
    }
    if !updated {
        myApp.Status.Conditions = append(myApp.Status.Conditions, condition)
    }

    return r.Status().Update(ctx, myApp)
}

func (r *MyAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appsv1alpha1.MyApp{}).
        Owns(&appsv1.Deployment{}).
        Owns(&corev1.Service{}).
        Complete(r)
}
```

---

## **4. Testing**

### **4.1 Unit Tests**

```go
// controllers/myapp_controller_test.go
package controllers

import (
    "context"
    "testing"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    "sigs.k8s.io/controller-runtime/pkg/client"

    appsv1alpha1 "github.com/myorg/myapp-controller/api/v1alpha1"
)

var _ = Describe("MyApp Controller", func() {
    Context("When creating a MyApp resource", func() {
        It("Should create a Deployment", func() {
            By("Creating a new MyApp")
            ctx := context.Background()
            myApp := &appsv1alpha1.MyApp{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-myapp",
                    Namespace: "default",
                },
                Spec: appsv1alpha1.MyAppSpec{
                    Image:    "nginx:latest",
                    Replicas: 3,
                },
            }

            Expect(k8sClient.Create(ctx, myApp)).To(Succeed())

            // Wait for Deployment to be created
            deploymentLookupKey := types.NamespacedName{
                Name:      "test-myapp",
                Namespace: "default",
            }

            Eventually(func() bool {
                deployment := &appsv1.Deployment{}
                err := k8sClient.Get(ctx, deploymentLookupKey, deployment)
                return err == nil
            }, timeout, interval).Should(BeTrue())

            // Verify deployment has correct replicas
            deployment := &appsv1.Deployment{}
            Expect(k8sClient.Get(ctx, deploymentLookupKey, deployment)).To(Succeed())
            Expect(*deployment.Spec.Replicas).To(Equal(int32(3)))
        })
    })
})
```

---

## **5. Local Development**

### **5.1 Run Locally**

```bash
# Install CRDs
make install

# Run controller locally (connects to current kubeconfig context)
make run

# In another terminal, create a test resource
kubectl apply -f config/samples/apps_v1alpha1_myapp.yaml

# Watch logs
# Controller will reconcile the resource and create Deployment
```

### **5.2 Debug with Delve**

```bash
# Run with debugger
dlv debug ./main.go -- --kubeconfig=$HOME/.kube/config
```

---

## **6. Build and Deploy**

### **6.1 Build Container Image**

```bash
# Build image
make docker-build IMG=myregistry/myapp-controller:v1.0.0

# Push image
make docker-push IMG=myregistry/myapp-controller:v1.0.0

# Deploy to cluster
make deploy IMG=myregistry/myapp-controller:v1.0.0
```

### **6.2 Production Deployment**

```yaml
# config/manager/manager.yaml (kubebuilder generates this)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-controller
  namespace: myapp-system
spec:
  replicas: 1
  selector:
    matchLabels:
      control-plane: controller-manager
  template:
    metadata:
      labels:
        control-plane: controller-manager
    spec:
      containers:
        - name: manager
          image: myregistry/myapp-controller:v1.0.0
          command:
            - /manager
          args:
            - --leader-elect
          resources:
            limits:
              cpu: 500m
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 128Mi
```

---

## **7. Advanced Features**

### **7.1 Add Validation Webhook**

```go
// Implement Defaulter interface
func (r *MyApp) Default() {
    if r.Spec.Replicas == 0 {
        r.Spec.Replicas = 3
    }
}

// Implement Validator interface
func (r *MyApp) ValidateCreate() error {
    return r.validateMyApp()
}

func (r *MyApp) ValidateUpdate(old runtime.Object) error {
    return r.validateMyApp()
}

func (r *MyApp) validateMyApp() error {
    if r.Spec.Replicas > 10 {
        return fmt.Errorf("replicas cannot exceed 10")
    }
    if r.Spec.Image == "" {
        return fmt.Errorf("image is required")
    }
    return nil
}

// In main.go, enable webhooks
if err = (&appsv1alpha1.MyApp{}).SetupWebhookWithManager(mgr); err != nil {
    setupLog.Error(err, "unable to create webhook", "webhook", "MyApp")
    os.Exit(1)
}
```

---

## **8. Monitoring & Observability**

### **8.1 Add Metrics**

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
    reconcileTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "myapp_reconcile_total",
            Help: "Total number of reconciliations",
        },
        []string{"namespace", "name", "result"},
    )

    reconcileDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "myapp_reconcile_duration_seconds",
            Help: "Reconciliation duration",
        },
        []string{"namespace", "name"},
    )
)

func init() {
    metrics.Registry.MustRegister(reconcileTotal, reconcileDuration)
}

func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    start := time.Now()

    result, err := r.reconcile(ctx, req)

    duration := time.Since(start)
    reconcileDuration.WithLabelValues(req.Namespace, req.Name).Observe(duration.Seconds())

    status := "success"
    if err != nil {
        status = "error"
    }
    reconcileTotal.WithLabelValues(req.Namespace, req.Name, status).Inc()

    return result, err
}
```

---

## **9. Best Practices**

### **✅ Do's**

1. **Use owner references** for garbage collection
2. **Implement status subresource** for better updates
3. **Add validation** via webhooks or OpenAPI schema
4. **Test thoroughly** with integration tests
5. **Enable leader election** for HA
6. **Add proper RBAC** rules
7. **Version your CRDs** (v1alpha1 → v1beta1 → v1)
8. **Monitor with metrics** and alerts

### **❌ Don'ts**

1. **Don't skip finalizers** if cleanup is needed
2. **Don't ignore errors** in reconciliation
3. **Don't update status in main reconcile** - use Status().Update()
4. **Don't create too many controllers** - combine related logic
5. **Don't forget resource limits** on controller pods

---

## **10. Complete Example**

Full working example available at:
- **GitHub**: `github.com/kubernetes-sigs/kubebuilder/docs/book/src/cronjob-tutorial`
- **Kubernetes**: `sample-controller` in kubernetes/kubernetes

---

## **Summary**

Building a custom controller:
1. **Scaffold** with kubebuilder
2. **Define CRD** with proper validation
3. **Implement reconciliation** logic
4. **Add webhooks** for validation/defaulting
5. **Write tests** (unit + integration)
6. **Build & deploy** to production
7. **Monitor** with metrics

Custom controllers extend Kubernetes with domain-specific automation!
