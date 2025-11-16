# Controller Migration Strategies

**Document**: 66-migration-strategies.md
**Status**: Course Module - Production Operations
**Audience**: SRE Teams, Platform Engineers
**Prerequisites**: Controller patterns, Kubernetes versioning

---

## **Overview**

Migrating controllers and CRDs requires careful planning to avoid downtime and data loss. This document covers proven migration patterns.

### **Migration Scenarios**

1. **Version upgrades** (v1alpha1 → v1beta1 → v1)
2. **Breaking API changes**
3. **Controller logic updates**
4. **Storage version migrations**
5. **Multi-version support**

---

## **1. CRD Version Migration**

### **1.1 Multi-Version CRD**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.example.com
spec:
  group: example.com
  names:
    kind: MyResource
    plural: myresources
  scope: Namespaced
  versions:
    # Old version - deprecated
    - name: v1alpha1
      served: true
      storage: false
      deprecated: true
      deprecationWarning: "v1alpha1 is deprecated, use v1beta1"
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                oldField: {type: string}

    # New version - current
    - name: v1beta1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                newField: {type: string}
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          name: conversion-webhook
          namespace: default
          path: /convert
```

### **1.2 Conversion Webhook**

```go
import (
    "encoding/json"
    apiextensionsv1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
)

func (h *ConversionWebhook) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    var review apiextensionsv1.ConversionReview
    json.NewDecoder(r.Body).Decode(&review)

    response := &apiextensionsv1.ConversionResponse{
        UID:              review.Request.UID,
        ConvertedObjects: []runtime.RawExtension{},
        Result:           metav1.Status{Status: "Success"},
    }

    for _, obj := range review.Request.Objects {
        converted, err := h.convert(obj, review.Request.DesiredAPIVersion)
        if err != nil {
            response.Result = metav1.Status{
                Status:  "Failure",
                Message: err.Error(),
            }
            break
        }
        response.ConvertedObjects = append(response.ConvertedObjects, converted)
    }

    review.Response = response
    json.NewEncoder(w).Encode(review)
}

func (h *ConversionWebhook) convert(obj runtime.RawExtension, targetVersion string) (runtime.RawExtension, error) {
    switch targetVersion {
    case "example.com/v1beta1":
        return h.convertToV1Beta1(obj)
    case "example.com/v1alpha1":
        return h.convertToV1Alpha1(obj)
    default:
        return obj, nil
    }
}

func (h *ConversionWebhook) convertToV1Beta1(obj runtime.RawExtension) (runtime.RawExtension, error) {
    var v1alpha1Obj MyResourceV1Alpha1
    json.Unmarshal(obj.Raw, &v1alpha1Obj)

    // Field mapping: oldField → newField
    v1beta1Obj := MyResourceV1Beta1{
        Spec: MyResourceV1Beta1Spec{
            NewField: v1alpha1Obj.Spec.OldField,
        },
    }

    converted, _ := json.Marshal(v1beta1Obj)
    return runtime.RawExtension{Raw: converted}, nil
}
```

---

## **2. Storage Migration**

### **2.1 Change Storage Version**

```bash
# Step 1: Add new version as non-storage
kubectl apply -f crd-with-v1beta1-non-storage.yaml

# Step 2: Trigger storage migration
kubectl get myresources --all-namespaces -o json | \
  kubectl replace -f -

# Step 3: Update CRD to make v1beta1 storage version
kubectl apply -f crd-with-v1beta1-storage.yaml

# Step 4: Verify migration
kubectl get crd myresources.example.com -o yaml | grep storedVersions
```

### **2.2 Automated Migration Job**

```go
func migrateStorageVersion(ctx context.Context, client client.Client) error {
    // List all resources
    list := &MyResourceV1Alpha1List{}
    if err := client.List(ctx, list); err != nil {
        return err
    }

    for _, item := range list.Items {
        // Trigger re-write by updating
        item.Annotations["migration-timestamp"] = time.Now().Format(time.RFC3339)

        if err := client.Update(ctx, &item); err != nil {
            log.Printf("Failed to migrate %s: %v", item.Name, err)
            continue
        }
    }

    return nil
}
```

---

## **3. Controller Update Strategies**

### **3.1 Blue-Green Deployment**

```yaml
# Blue deployment (old version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-controller-blue
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-controller
      version: blue
  template:
    metadata:
      labels:
        app: my-controller
        version: blue
    spec:
      containers:
        - name: controller
          image: my-controller:v1.0.0
          env:
            - name: LEADER_ELECTION_ID
              value: my-controller-lock

---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-controller-green
spec:
  replicas: 0  # Start with 0, scale up after validation
  selector:
    matchLabels:
      app: my-controller
      version: green
  template:
    metadata:
      labels:
        app: my-controller
        version: green
    spec:
      containers:
        - name: controller
          image: my-controller:v2.0.0
          env:
            - name: LEADER_ELECTION_ID
              value: my-controller-lock  # Same lock!
```

**Migration process**:
1. Deploy green with replicas=0
2. Scale green to 1 (takes over via leader election)
3. Validate green is working
4. Scale blue to 0
5. Delete blue deployment

### **3.2 Canary Rollout**

```yaml
# Split traffic with multiple controllers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-controller-v2
spec:
  replicas: 1  # Canary
  selector:
    matchLabels:
      app: my-controller
      version: v2
  template:
    spec:
      containers:
        - name: controller
          image: my-controller:v2.0.0
          env:
            - name: CONTROLLER_FILTERS
              value: "namespace=canary-ns"  # Only handle canary namespace

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-controller-v1
spec:
  replicas: 2  # Stable
  selector:
    matchLabels:
      app: my-controller
      version: v1
  template:
    spec:
      containers:
        - name: controller
          image: my-controller:v1.0.0
          env:
            - name: CONTROLLER_FILTERS
              value: "namespace!=canary-ns"  # Handle all except canary
```

---

## **4. Data Migration**

### **4.1 Finalizer-based Migration**

```go
const migrationFinalizer = "migration.example.com/v1alpha1-to-v1beta1"

func (r *Reconciler) migrate(ctx context.Context, obj *v1alpha1.MyResource) error {
    // Check if already migrated
    if !controllerutil.ContainsFinalizer(obj, migrationFinalizer) {
        return nil
    }

    // Create new version object
    newObj := &v1beta1.MyResource{
        ObjectMeta: metav1.ObjectMeta{
            Name:      obj.Name,
            Namespace: obj.Namespace,
        },
        Spec: v1beta1.MyResourceSpec{
            NewField: obj.Spec.OldField,
        },
    }

    // Create new object
    if err := r.Create(ctx, newObj); err != nil && !errors.IsAlreadyExists(err) {
        return err
    }

    // Remove finalizer from old object
    controllerutil.RemoveFinalizer(obj, migrationFinalizer)
    if err := r.Update(ctx, obj); err != nil {
        return err
    }

    // Delete old object
    return r.Delete(ctx, obj)
}
```

---

## **5. Rollback Strategies**

### **5.1 Safe Rollback**

```bash
# 1. Check current state
kubectl get myresources -A

# 2. Backup current state
kubectl get myresources -A -o yaml > backup.yaml

# 3. Rollback CRD
kubectl apply -f crd-v1alpha1.yaml

# 4. Rollback controller
kubectl rollout undo deployment/my-controller

# 5. Verify
kubectl rollout status deployment/my-controller
```

### **5.2 Emergency Rollback**

```yaml
# Keep old CRD version available
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.example.com
spec:
  versions:
    - name: v1beta1
      served: true
      storage: true
    - name: v1alpha1
      served: true  # Keep serving for rollback
      storage: false
```

---

## **6. Testing Migrations**

### **6.1 Migration Test Suite**

```go
func TestMigration(t *testing.T) {
    // Create v1alpha1 object
    oldObj := &v1alpha1.MyResource{
        ObjectMeta: metav1.ObjectMeta{Name: "test"},
        Spec: v1alpha1.MyResourceSpec{
            OldField: "value",
        },
    }

    // Apply conversion
    newObj, err := ConvertToV1Beta1(oldObj)
    if err != nil {
        t.Fatalf("Conversion failed: %v", err)
    }

    // Verify fields mapped correctly
    if newObj.Spec.NewField != "value" {
        t.Errorf("Expected newField=value, got %s", newObj.Spec.NewField)
    }

    // Round-trip test
    roundtrip, err := ConvertToV1Alpha1(newObj)
    if err != nil {
        t.Fatalf("Round-trip failed: %v", err)
    }

    if !reflect.DeepEqual(oldObj.Spec, roundtrip.Spec) {
        t.Error("Round-trip conversion lost data")
    }
}
```

---

## **7. Migration Checklist**

**Pre-Migration**:
- [ ] Backup all CRs
- [ ] Document current state
- [ ] Test conversion in staging
- [ ] Prepare rollback plan
- [ ] Communicate to users

**During Migration**:
- [ ] Apply new CRD version
- [ ] Deploy conversion webhook
- [ ] Migrate storage version
- [ ] Update controller
- [ ] Monitor error rates

**Post-Migration**:
- [ ] Verify all resources migrated
- [ ] Check controller health
- [ ] Remove deprecated versions
- [ ] Update documentation
- [ ] Clean up old resources

---

## **8. Common Pitfalls**

**❌ Pitfall 1**: Changing storage version without migration
```bash
# DON'T: Breaks existing resources
kubectl apply -f crd-with-new-storage-version.yaml
```

**✅ Solution**: Migrate data first
```bash
# DO: Migrate, then change storage
kubectl get myresources -A -o json | kubectl replace -f -
kubectl apply -f crd-with-new-storage-version.yaml
```

**❌ Pitfall 2**: Removing old version too quickly
```yaml
# DON'T: Immediately remove v1alpha1
versions:
  - name: v1beta1
    served: true
    storage: true
```

**✅ Solution**: Keep deprecated version for transition period
```yaml
# DO: Deprecate gradually
versions:
  - name: v1alpha1
    served: true
    deprecated: true
  - name: v1beta1
    served: true
    storage: true
```

---

## **Summary**

Safe migrations require:
- **Multi-version CRDs** - Support old and new simultaneously
- **Conversion webhooks** - Automatic version translation
- **Storage migration** - Rewrite etcd data
- **Gradual rollout** - Blue-green or canary deployments
- **Comprehensive testing** - Validate conversions
- **Rollback plan** - Prepare for failures

Plan carefully, test thoroughly, migrate gradually!
