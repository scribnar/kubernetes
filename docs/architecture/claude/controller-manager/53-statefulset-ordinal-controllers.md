# StatefulSet Ordinal Controllers

## Overview

The StatefulSet Ordinal Controllers manage ordered, sequential pod creation and deletion for StatefulSets. Each pod receives a unique, stable ordinal index (0, 1, 2, ..., N-1) that persists across rescheduling, providing predictable identity and ordering guarantees essential for stateful applications.

**Key Features:**
- **Ordered Deployment**: Pods created sequentially (0→1→2→...)
- **Ordered Termination**: Pods deleted in reverse order (N-1→N-2→...→0)
- **Stable Identity**: Pod names include ordinal (web-0, web-1, web-2)
- **Persistent Storage**: PVCs bound to specific ordinals
- **Parallel Scaling**: Optional parallel pod management

**Introduced in:** Kubernetes 1.5 (Beta), 1.9 (GA)

## Architecture

### StatefulSet Ordinal System

```mermaid
graph TB
    subgraph "StatefulSet Spec"
        SS[StatefulSet<br/>replicas: 5<br/>podManagementPolicy]
        TMPL[Pod Template]
        VCT[VolumeClaimTemplate]
    end

    subgraph "Ordinal Management"
        OM[Ordinal Manager]
        OA[Ordinal Allocator]
        OT[Ordinal Tracker]
    end

    subgraph "Pod Creation (Ordered)"
        P0[Pod: web-0<br/>ordinal: 0]
        P1[Pod: web-1<br/>ordinal: 1]
        P2[Pod: web-2<br/>ordinal: 2]
        P3[Pod: web-3<br/>ordinal: 3]
        P4[Pod: web-4<br/>ordinal: 4]
    end

    subgraph "Storage Binding"
        PVC0[PVC: data-web-0]
        PVC1[PVC: data-web-1]
        PVC2[PVC: data-web-2]
        PVC3[PVC: data-web-3]
        PVC4[PVC: data-web-4]
    end

    SS -->|Manage| OM
    OM -->|Allocate| OA
    OA -->|Track| OT

    OT -->|Create Sequential| P0
    P0 -->|Ready, then create| P1
    P1 -->|Ready, then create| P2
    P2 -->|Ready, then create| P3
    P3 -->|Ready, then create| P4

    P0 -.->|Bind| PVC0
    P1 -.->|Bind| PVC1
    P2 -.->|Bind| PVC2
    P3 -.->|Bind| PVC3
    P4 -.->|Bind| PVC4

    style SS fill:#326CE5,color:#fff
    style OM fill:#FF6B6B,color:#fff
    style P0 fill:#4ECDC4,color:#fff
    style PVC0 fill:#FFE66D,color:#000
```

### Pod Lifecycle with Ordinals

```mermaid
stateDiagram-v2
    [*] --> PendingCreation: StatefulSet Created

    PendingCreation --> CreateOrdinal0: Start with ordinal 0
    CreateOrdinal0 --> WaitReady0: Pod web-0 Creating

    WaitReady0 --> Ready0: Pod Running & Ready
    Ready0 --> CreateOrdinal1: Create next ordinal

    CreateOrdinal1 --> WaitReady1: Pod web-1 Creating
    WaitReady1 --> Ready1: Pod Running & Ready
    Ready1 --> CreateOrdinal2: Create next ordinal

    CreateOrdinal2 --> WaitReady2: Pod web-2 Creating
    WaitReady2 --> Ready2: Pod Running & Ready
    Ready2 --> AllCreated: All replicas created

    AllCreated --> ScaleDown: Decrease replicas
    ScaleDown --> DeleteHighest: Delete web-2 first

    DeleteHighest --> WaitDeleted2: Wait for termination
    WaitDeleted2 --> Deleted2: Pod removed
    Deleted2 --> DeleteNext: Delete web-1

    DeleteNext --> WaitDeleted1: Wait for termination
    WaitDeleted1 --> Deleted1: Pod removed
    Deleted1 --> FinalState: Scaled down

    FinalState --> [*]: StatefulSet stable

    note right of CreateOrdinal0
        OrderedReady policy:
        - Wait for pod N to be Ready
        - Before creating pod N+1
    end note

    note right of DeleteHighest
        Termination order:
        - Always delete highest ordinal first
        - Wait for complete termination
        - Respect terminationGracePeriod
    end note
```

## Controller Implementation

### StatefulSet Controller with Ordinal Management

**File:** `pkg/controller/statefulset/stateful_set.go`

```go
// StatefulSetController manages StatefulSets
type StatefulSetController struct {
    kubeClient clientset.Interface
    control    StatefulSetControlInterface

    // Informers
    setLister  appslisters.StatefulSetLister
    podLister  corelisters.PodLister
    pvcLister  corelisters.PersistentVolumeClaimLister

    // Queue for StatefulSets
    queue workqueue.RateLimitingInterface
}

// sync processes a StatefulSet
func (ssc *StatefulSetController) sync(
    ctx context.Context,
    key string,
) error {
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }

    // Get StatefulSet
    set, err := ssc.setLister.StatefulSets(namespace).Get(name)
    if err != nil {
        if errors.IsNotFound(err) {
            return nil
        }
        return err
    }

    // Get selector
    selector, err := metav1.LabelSelectorAsSelector(set.Spec.Selector)
    if err != nil {
        return err
    }

    // Get pods
    pods, err := ssc.podLister.Pods(set.Namespace).List(selector)
    if err != nil {
        return err
    }

    // Get PVCs
    pvcs, err := ssc.pvcLister.PersistentVolumeClaims(set.Namespace).
        List(selector)
    if err != nil {
        return err
    }

    // Update StatefulSet
    return ssc.control.UpdateStatefulSet(ctx, set, pods, pvcs)
}
```

**Location:** `pkg/controller/statefulset/stateful_set.go:100-200`

### Ordinal Pod Management

**File:** `pkg/controller/statefulset/stateful_set_control.go`

```go
// UpdateStatefulSet performs update logic for StatefulSet
func (ssc *defaultStatefulSetControl) UpdateStatefulSet(
    ctx context.Context,
    set *apps.StatefulSet,
    pods []*v1.Pod,
    pvcs []*v1.PersistentVolumeClaim,
) error {
    // Get current and update revisions
    currentRevision, updateRevision, err := ssc.getStatefulSetRevisions(
        set,
        pods,
    )
    if err != nil {
        return err
    }

    // Perform update based on strategy
    status, err := ssc.updateStatefulSet(
        ctx,
        set,
        currentRevision,
        updateRevision,
        pods,
        pvcs,
    )
    if err != nil {
        return err
    }

    // Update status
    return ssc.updateStatefulSetStatus(ctx, set, status)
}

// updateStatefulSet manages pod ordinals
func (ssc *defaultStatefulSetControl) updateStatefulSet(
    ctx context.Context,
    set *apps.StatefulSet,
    currentRevision *apps.ControllerRevision,
    updateRevision *apps.ControllerRevision,
    pods []*v1.Pod,
    pvcs []*v1.PersistentVolumeClaim,
) (*apps.StatefulSetStatus, error) {

    // Get ordinal range
    replicaCount := int(*set.Spec.Replicas)

    // Sort pods by ordinal
    sort.Sort(statefulPodsByOrdinal(pods))

    // Build ordinal → pod map
    replicas := make([]*v1.Pod, replicaCount)
    for _, pod := range pods {
        ordinal := getOrdinal(pod)
        if ordinal < 0 || ordinal >= replicaCount {
            // Pod outside current range, will be deleted
            continue
        }
        replicas[ordinal] = pod
    }

    // Determine pods to create, update, or delete
    for i := 0; i < replicaCount; i++ {
        // Check if pod exists for this ordinal
        if replicas[i] == nil {
            // Need to create pod
            if err := ssc.createPodForOrdinal(
                ctx,
                set,
                i,
                currentRevision,
                updateRevision,
            ); err != nil {
                return nil, err
            }

            // In OrderedReady mode, wait for this pod to be ready
            if isOrderedReady(set) {
                break
            }
        } else {
            // Pod exists, check if it needs update
            if !isHealthy(replicas[i]) {
                // Pod not healthy, wait before proceeding
                if isOrderedReady(set) {
                    break
                }
            }

            // Check if pod needs update
            if !isUpdated(replicas[i], updateRevision) {
                if err := ssc.updatePod(
                    ctx,
                    set,
                    replicas[i],
                    updateRevision,
                ); err != nil {
                    return nil, err
                }

                // In OrderedReady mode, wait for update to complete
                if isOrderedReady(set) {
                    break
                }
            }
        }
    }

    // Delete pods with ordinals >= replicaCount
    for i := len(pods) - 1; i >= replicaCount; i-- {
        if pods[i] == nil {
            continue
        }

        ordinal := getOrdinal(pods[i])
        if ordinal >= replicaCount {
            if err := ssc.deletePod(ctx, set, pods[i]); err != nil {
                return nil, err
            }

            // In OrderedReady mode, wait for deletion to complete
            if isOrderedReady(set) {
                break
            }
        }
    }

    // Calculate status
    status := ssc.calculateStatus(set, pods, currentRevision, updateRevision)
    return status, nil
}

// createPodForOrdinal creates a pod with specific ordinal
func (ssc *defaultStatefulSetControl) createPodForOrdinal(
    ctx context.Context,
    set *apps.StatefulSet,
    ordinal int,
    currentRevision *apps.ControllerRevision,
    updateRevision *apps.ControllerRevision,
) error {
    // Determine which revision to use
    revision := currentRevision
    if ordinal >= int(*set.Status.CurrentReplicas) {
        revision = updateRevision
    }

    // Create PVCs first
    if err := ssc.createPVCsForOrdinal(ctx, set, ordinal); err != nil {
        return err
    }

    // Build pod
    pod := newPodForOrdinal(set, ordinal, revision)

    // Create pod
    _, err := ssc.kubeClient.CoreV1().Pods(set.Namespace).Create(
        ctx,
        pod,
        metav1.CreateOptions{},
    )

    if err != nil && !errors.IsAlreadyExists(err) {
        return err
    }

    klog.V(4).Infof(
        "Created pod %s/%s for StatefulSet %s/%s with ordinal %d",
        pod.Namespace,
        pod.Name,
        set.Namespace,
        set.Name,
        ordinal,
    )

    return nil
}

// newPodForOrdinal creates pod object with ordinal
func newPodForOrdinal(
    set *apps.StatefulSet,
    ordinal int,
    revision *apps.ControllerRevision,
) *v1.Pod {
    // Pod name includes ordinal
    podName := fmt.Sprintf("%s-%d", set.Name, ordinal)

    pod := &v1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      podName,
            Namespace: set.Namespace,
            Labels:    set.Spec.Template.Labels,
            Annotations: map[string]string{
                apps.StatefulSetRevisionAnnotation: revision.Name,
            },
        },
        Spec: *set.Spec.Template.Spec.DeepCopy(),
    }

    // Add ordinal label
    if pod.Labels == nil {
        pod.Labels = make(map[string]string)
    }
    pod.Labels[apps.StatefulSetPodNameLabel] = podName
    pod.Labels[apps.ControllerRevisionHashLabelKey] = revision.Name

    // Set hostname to pod name (for stable network identity)
    pod.Spec.Hostname = podName

    // Set subdomain for headless service
    if set.Spec.ServiceName != "" {
        pod.Spec.Subdomain = set.Spec.ServiceName
    }

    // Update volume claim templates
    for i := range pod.Spec.Volumes {
        if pvc := pod.Spec.Volumes[i].PersistentVolumeClaim; pvc != nil {
            // Replace claim name with ordinal-specific name
            claimName := getPVCNameForOrdinal(
                set.Spec.VolumeClaimTemplates[i].Name,
                set.Name,
                ordinal,
            )
            pvc.ClaimName = claimName
        }
    }

    // Set owner reference
    pod.OwnerReferences = []metav1.OwnerReference{
        *metav1.NewControllerRef(
            set,
            apps.SchemeGroupVersion.WithKind("StatefulSet"),
        ),
    }

    return pod
}

// getOrdinal extracts ordinal from pod name
func getOrdinal(pod *v1.Pod) int {
    // Pod name format: <statefulset-name>-<ordinal>
    parts := strings.Split(pod.Name, "-")
    if len(parts) < 2 {
        return -1
    }

    ordinal, err := strconv.Atoi(parts[len(parts)-1])
    if err != nil {
        return -1
    }

    return ordinal
}

// isOrderedReady checks if StatefulSet uses OrderedReady policy
func isOrderedReady(set *apps.StatefulSet) bool {
    if set.Spec.PodManagementPolicy == nil {
        return true
    }
    return *set.Spec.PodManagementPolicy == apps.OrderedReadyPodManagement
}

// isHealthy checks if pod is running and ready
func isHealthy(pod *v1.Pod) bool {
    if pod.Status.Phase != v1.PodRunning {
        return false
    }

    for _, condition := range pod.Status.Conditions {
        if condition.Type == v1.PodReady {
            return condition.Status == v1.ConditionTrue
        }
    }

    return false
}

// getPVCNameForOrdinal generates PVC name for ordinal
func getPVCNameForOrdinal(
    claimName string,
    setName string,
    ordinal int,
) string {
    return fmt.Sprintf("%s-%s-%d", claimName, setName, ordinal)
}
```

**Location:** `pkg/controller/statefulset/stateful_set_control.go:100-500`

### PVC Management for Ordinals

**File:** `pkg/controller/statefulset/stateful_set_utils.go`

```go
// createPVCsForOrdinal creates PVCs for a specific ordinal
func (ssc *defaultStatefulSetControl) createPVCsForOrdinal(
    ctx context.Context,
    set *apps.StatefulSet,
    ordinal int,
) error {
    for _, template := range set.Spec.VolumeClaimTemplates {
        pvc := newPVCForOrdinal(set, &template, ordinal)

        // Check if PVC already exists
        _, err := ssc.kubeClient.CoreV1().
            PersistentVolumeClaims(set.Namespace).
            Get(ctx, pvc.Name, metav1.GetOptions{})

        if err == nil {
            // PVC exists
            continue
        }

        if !errors.IsNotFound(err) {
            return err
        }

        // Create PVC
        _, err = ssc.kubeClient.CoreV1().
            PersistentVolumeClaims(set.Namespace).
            Create(ctx, pvc, metav1.CreateOptions{})

        if err != nil && !errors.IsAlreadyExists(err) {
            return err
        }

        klog.V(4).Infof(
            "Created PVC %s/%s for StatefulSet %s with ordinal %d",
            pvc.Namespace,
            pvc.Name,
            set.Name,
            ordinal,
        )
    }

    return nil
}

// newPVCForOrdinal creates PVC for specific ordinal
func newPVCForOrdinal(
    set *apps.StatefulSet,
    template *v1.PersistentVolumeClaim,
    ordinal int,
) *v1.PersistentVolumeClaim {
    pvc := template.DeepCopy()

    // Set name with ordinal
    pvc.Name = getPVCNameForOrdinal(
        template.Name,
        set.Name,
        ordinal,
    )
    pvc.Namespace = set.Namespace

    // Set labels
    if pvc.Labels == nil {
        pvc.Labels = make(map[string]string)
    }
    for k, v := range set.Spec.Template.Labels {
        pvc.Labels[k] = v
    }
    pvc.Labels[apps.StatefulSetPodNameLabel] = fmt.Sprintf(
        "%s-%d",
        set.Name,
        ordinal,
    )

    // Set owner reference
    pvc.OwnerReferences = []metav1.OwnerReference{
        *metav1.NewControllerRef(
            set,
            apps.SchemeGroupVersion.WithKind("StatefulSet"),
        ),
    }

    return pvc
}
```

**Location:** `pkg/controller/statefulset/stateful_set_utils.go:200-300`

## Pod Management Policies

### OrderedReady vs Parallel

```mermaid
graph TB
    subgraph "OrderedReady Policy (Default)"
        OR1[Create web-0]
        OR2[Wait for Ready]
        OR3[Create web-1]
        OR4[Wait for Ready]
        OR5[Create web-2]

        OR1 --> OR2
        OR2 --> OR3
        OR3 --> OR4
        OR4 --> OR5
    end

    subgraph "Parallel Policy"
        P1[Create web-0]
        P2[Create web-1]
        P3[Create web-2]

        P1 -.->|All at once| P2
        P2 -.->|All at once| P3
    end

    style OR1 fill:#326CE5,color:#fff
    style OR2 fill:#FF6B6B,color:#fff
    style P1 fill:#4ECDC4,color:#fff
```

## Configuration Examples

### Basic StatefulSet with Ordinals

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  # Service for stable network identity
  serviceName: "web"

  # Number of replicas (creates ordinals 0-4)
  replicas: 5

  # Pod management policy
  podManagementPolicy: OrderedReady  # Default, can be "Parallel"

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web

        # Volume mount using ordinal-specific PVC
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html

  # Volume claim templates (one PVC per ordinal)
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 1Gi

# Results in:
# Pods: web-0, web-1, web-2, web-3, web-4
# PVCs: www-web-0, www-web-1, www-web-2, www-web-3, www-web-4
```

### Headless Service for Stable Network Identity

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app: nginx
spec:
  # Headless service (clusterIP: None)
  clusterIP: None

  # Selector matches StatefulSet pods
  selector:
    app: nginx

  ports:
  - port: 80
    name: web

# DNS entries created:
# web-0.web.default.svc.cluster.local
# web-1.web.default.svc.cluster.local
# web-2.web.default.svc.cluster.local
# web-3.web.default.svc.cluster.local
# web-4.web.default.svc.cluster.local
```

### Parallel Pod Management

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-parallel
spec:
  serviceName: "web"
  replicas: 10

  # Parallel pod management (no ordering)
  podManagementPolicy: Parallel

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21

  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi

# All 10 pods created simultaneously (web-0 through web-9)
# No waiting for pod readiness between creates
```

### Rolling Update Strategy

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 5

  # Update strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      # Partition controls which pods get updated
      partition: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.22  # New version

# With partition: 3
# - Pods web-4 and web-3 will be updated to nginx:1.22
# - Pods web-2, web-1, web-0 remain on old version
# - Allows gradual rollout and canary testing
```

## Monitoring and Metrics

### StatefulSet Metrics

```yaml
# StatefulSet metrics
kube_statefulset_status_replicas{statefulset="web"}
kube_statefulset_status_replicas_ready{statefulset="web"}
kube_statefulset_status_replicas_current{statefulset="web"}
kube_statefulset_status_replicas_updated{statefulset="web"}

# Pod metrics by ordinal
kube_pod_info{pod="web-0",statefulset="web"}
kube_pod_status_phase{pod="web-0",phase="Running"}

# PVC metrics by ordinal
kube_persistentvolumeclaim_info{persistentvolumeclaim="www-web-0"}
```

### Prometheus Queries

```promql
# StatefulSet replica status
kube_statefulset_status_replicas_ready / kube_statefulset_status_replicas

# Pods not ready by ordinal
kube_pod_status_ready{condition="false"} *
on(pod) group_left(statefulset) kube_pod_labels{statefulset="web"}

# PVC bind status by ordinal
kube_persistentvolumeclaim_status_phase{phase!="Bound"} *
on(persistentvolumeclaim) group_left() kube_persistentvolumeclaim_labels{statefulset="web"}

# Ordinal-specific queries
kube_pod_info{pod=~"web-[0-4]"}
```

## Troubleshooting Guide

### Common Issues

#### Issue 1: Pod Stuck in Pending (Ordinal blocked)

**Symptoms:**
- Pod web-1 not created
- web-0 exists but not Ready
- Entire StatefulSet stuck

**Diagnosis:**
```bash
# Check pod status
kubectl get pods -l app=nginx

# Check events for pod
kubectl describe pod web-0

# Check if blocking ordinal
kubectl get statefulset web -o jsonpath='{.status}'
```

**Common Causes:**
1. Lower ordinal pod not Ready (OrderedReady)
2. PVC not bound
3. Insufficient resources

**Resolution:**
```bash
# Check PVC status
kubectl get pvc -l app=nginx

# Check pod logs
kubectl logs web-0

# Switch to Parallel if ordering not needed
kubectl patch statefulset web -p '
{
  "spec": {
    "podManagementPolicy": "Parallel"
  }
}'
```

#### Issue 2: Wrong PVC Bound to Pod

**Symptoms:**
- Pod using incorrect PVC
- Data from wrong ordinal
- PVC name mismatch

**Diagnosis:**
```bash
# Check PVC bindings
kubectl get pods -l app=nginx \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.volumes[?(@.persistentVolumeClaim)].persistentVolumeClaim.claimName}{"\n"}{end}'

# Expected pattern:
# web-0 → www-web-0
# web-1 → www-web-1
```

**Resolution:**
```bash
# Delete pod to force rebind
kubectl delete pod web-1

# Check VolumeClaimTemplate
kubectl get statefulset web -o yaml | grep -A 10 volumeClaimTemplates
```

### Debug Commands

```bash
# List pods by ordinal
kubectl get pods -l app=nginx --sort-by=.metadata.name

# Check ordinal readiness
kubectl get pods -l app=nginx \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Verify PVC per ordinal
kubectl get pvc -l app=nginx --sort-by=.metadata.name

# Check StatefulSet status
kubectl get statefulset web -o jsonpath='{.status}' | jq

# Scale StatefulSet
kubectl scale statefulset web --replicas=3

# Watch ordinal creation
kubectl get pods -l app=nginx -w

# Delete specific ordinal
kubectl delete pod web-2

# Check DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  nslookup web-0.web.default.svc.cluster.local
```

## Best Practices

1. **Use Headless Service**
   ```yaml
   # Required for stable DNS
   clusterIP: None
   ```

2. **Set Appropriate Grace Period**
   ```yaml
   spec:
     template:
       spec:
         terminationGracePeriodSeconds: 30
   ```

3. **Use Partition for Canary**
   ```yaml
   updateStrategy:
     rollingUpdate:
       partition: N  # Only update pods >= N
   ```

4. **Monitor Ordinal Health**
   - Alert on pods not Ready
   - Track PVC binding
   - Monitor creation order

## Performance Considerations

### Scaling

- **OrderedReady**: O(N) time, sequential
- **Parallel**: O(1) time, concurrent
- **Recommendation**: Use Parallel if ordering not required

### Storage

- One PVC per ordinal per volume claim template
- PVCs persist even after StatefulSet deletion
- Manual PVC cleanup required

## Related Components

- **StatefulSet Controller** (`pkg/controller/statefulset/`)
- **Pod Controller** (creates/deletes pods)
- **PVC Controller** (binds volumes)
- **Service Controller** (DNS entries)

## References

- **KEP-1847**: [StatefulSet Ordinal Management](https://github.com/kubernetes/enhancements/tree/master/keps/sig-apps/1847-autodelete-statefulset-pvcs)
- **KEP-3335**: [StatefulSet Slice](https://github.com/kubernetes/enhancements/tree/master/keps/sig-apps/3335-statefulset-slice)
- **Source Code**: `pkg/controller/statefulset/`
- **API Reference**: [StatefulSet v1](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/stateful-set-v1/)
