# Kubernetes Static Pods Implementation Analysis

## Overview
Static pods are pods that are defined locally on a node (via files or HTTP URLs) and are managed directly by the kubelet, rather than through the Kubernetes API server. This document provides a comprehensive analysis of how static pods are implemented in Kubernetes.

---

## 1. Static Pod Sources

### 1.1 File-Based Static Pods (--pod-manifest-path)

**Configuration:**
- Defined in: `/pkg/kubelet/apis/config/types.go`
- Config field: `StaticPodPath` (string)
- Check frequency: `FileCheckFrequency` (default: 20 seconds)

**Implementation:** `/pkg/kubelet/config/file.go`

The file source supports:
- **Single file**: Path to a single pod manifest file (YAML/JSON)
- **Directory**: Path to a directory containing multiple pod manifest files

Key characteristics:
- Files starting with dots (.) are ignored
- Non-recursive directory traversal
- Files sorted alphabetically
- Supports maximum file size: 10MB (maxConfigLength)

**File watching on Linux:** `/pkg/kubelet/config/file_linux.go`
- Uses `fsnotify` (inotify on Linux) for real-time file change detection
- Monitors events: Create, Write, Chmod, Remove, Rename
- Implements exponential backoff retry (1s to 20s) if watching fails
- Poll-based fallback if watch fails

**Watch event handling:**
```go
// Watch event types
type watchEvent struct {
    fileName  string
    eventType podEventType  // podAdd, podModify, podDelete
}

// On file changes:
// - Create/Write/Chmod → Extract pod from file, add/update in store
// - Remove/Rename → Delete pod from store
```

### 1.2 HTTP-Based Static Pods (--manifest-url)

**Configuration:**
- Config field: `StaticPodURL` (string)
- Headers: `StaticPodURLHeader` (map[string][]string)
- Check frequency: `HTTPCheckFrequency` (default: 20 seconds)

**Implementation:** `/pkg/kubelet/config/http.go`

Characteristics:
- HTTP GET request with custom headers
- HTTP timeout: 10 seconds
- Supports both single pod and pod list formats
- Change detection via byte-level comparison
- Exponential failure logging (first 3 failures logged, then degraded to V(4))

Processing flow:
1. Fetch content from URL
2. Compare with previously fetched data
3. Parse as single pod, then as pod list
4. Apply defaults and send updates

---

## 2. Static Pod Loading and Parsing

### 2.1 Pod Manifest Parsing

**Common parsing function:** `/pkg/kubelet/config/common.go`

```go
func tryDecodeSinglePod(logger, data []byte, defaultFn) (parsed bool, pod *v1.Pod, err error)
func tryDecodePodList(logger, data []byte, defaultFn) (parsed bool, pods v1.PodList, err error)
```

**Supported formats:**
- YAML (converted to JSON internally)
- JSON
- Lists of pods (PodList)

**Validation steps:**
1. Convert YAML to JSON if needed
2. Decode using universal decoder
3. Validate pod has a name
4. Apply default values (see 2.2)
5. Validate pod with ValidatePodCreate
6. Check for API object references (feature gate: PreventStaticPodAPIReferences)
7. Check for ClusterTrustBundle and ResourceClaims (not allowed in static pods)
8. Convert internal Pod to v1.Pod

**Errors handled:**
- `ErrStaticPodTriedToUseClusterTrustBundle`
- `ErrStaticPodTriedToUseResourceClaims`

### 2.2 Default Value Application

**Function:** `applyDefaults()` in `/pkg/kubelet/config/common.go`

Applied to all static pods:

| Field | Value | Purpose |
|-------|-------|---------|
| `UID` | MD5 hash of pod + source + nodeName | Unique identifier |
| `Name` | `{podName}-{nodeName}` | Ensures uniqueness across nodes |
| `Namespace` | `default` | Default namespace if not specified |
| `Spec.NodeName` | Current node name | Binds pod to node |
| `Status.Phase` | `Pending` | Initial status |
| `Annotations` | | Local kubelet annotations |
| → `config.hash` | Pod UID | MD5 hash for change detection |
| → `config.source` | Source type | File or HTTP |
| Tolerations | `NoExecute:Exists` (files only) | Not evicted on node problems |

**UID Generation (MD5 based):**
```
hash = MD5(pod content)
// For file source:
hash.Write("host:" + nodeName)
hash.Write("file:" + source)
// For URL source:
hash.Write("url:" + source)
```

---

## 3. Mirror Pod Creation in API Server

### 3.1 Mirror Pod Concept

Mirror pods are shadow pods created in the API server to represent static pods, allowing:
- Kubelet to report static pod status via normal status update mechanism
- Users to see static pod status through `kubectl get pods`
- API server to track static pod lifecycle

**Key relationship:** Static pod and mirror pod have the same name/namespace but different UIDs.

### 3.2 Mirror Pod Creation

**Files involved:**
- `/pkg/kubelet/pod/mirror_client.go` - API interactions
- `/pkg/kubelet/kubelet.go` - Creation orchestration

**Implementation: `CreateMirrorPod()`**

```go
func (mc *basicMirrorClient) CreateMirrorPod(ctx context.Context, pod *v1.Pod) error {
    // 1. Copy static pod
    copyPod := *pod
    copyPod.Annotations = make(map[string]string)
    for k, v := range pod.Annotations {
        copyPod.Annotations[k] = v
    }
    
    // 2. Add mirror annotation with hash
    hash := getPodHash(pod)  // Get config.hash annotation
    copyPod.Annotations[kubetypes.ConfigMirrorAnnotationKey] = hash
    
    // 3. Set owner reference to Node (MirrorPodNodeRestriction feature)
    nodeUID, err := mc.getNodeUID()
    controller := true
    copyPod.OwnerReferences = []metav1.OwnerReference{{
        APIVersion: v1.SchemeGroupVersion.String(),
        Kind:       "Node",
        Name:       mc.nodeName,
        UID:        nodeUID,
        Controller: &controller,
    }}
    
    // 4. Create in API server
    apiPod, err := mc.apiserverClient.CoreV1().Pods(copyPod.Namespace).Create(ctx, &copyPod, metav1.CreateOptions{})
    
    // 5. If already exists with same hash, skip
    if err != nil && apierrors.IsAlreadyExists(err) {
        if h, ok := apiPod.Annotations[kubetypes.ConfigMirrorAnnotationKey]; ok && h == hash {
            return nil
        }
    }
    return err
}
```

**When called:**
- In `SyncPod()` after preparing pod (line 2075 in kubelet.go)
- In `fastStaticPodsRegistration()` - waits for node sync, then reconciles all static pods (line 3271)

### 3.3 Reconciliation: `tryReconcileMirrorPods()`

Located in `/pkg/kubelet/kubelet.go` (line 3221)

Handles synchronization when static and mirror pods diverge:

```go
func (kl *Kubelet) tryReconcileMirrorPods(ctx context.Context, staticPod, mirrorPod *v1.Pod) {
    if !kubetypes.IsStaticPod(staticPod) {
        return
    }
    
    deleted := false
    if mirrorPod != nil {
        // Delete mirror if:
        // 1. Mirror has DeletionTimestamp set, OR
        // 2. Mirror pod hash doesn't match static pod hash
        if mirrorPod.DeletionTimestamp != nil || !kubepod.IsMirrorPodOf(mirrorPod, staticPod) {
            kl.mirrorPodClient.DeleteMirrorPod(ctx, podFullName, &mirrorPod.UID)
            deleted = ok
        }
    }
    
    if mirrorPod == nil || deleted {
        // Create new mirror pod if:
        // 1. No mirror pod exists, OR
        // 2. Just deleted the old one
        node, err := kl.GetNode()
        if err == nil && node.DeletionTimestamp == nil {
            kl.mirrorPodClient.CreateMirrorPod(ctx, staticPod)
        }
    }
}
```

**Key functions:**
- `IsMirrorPodOf(mirrorPod, pod)` - Checks name, namespace, and hash match
- `getPodHash(pod)` - Returns ConfigHashAnnotationKey annotation
- `getHashFromMirrorPod(pod)` - Returns ConfigMirrorAnnotationKey annotation

---

## 4. Static Pod Update Detection and Handling

### 4.1 Change Detection Mechanism

**File source:**
- inotify events on Linux (fsnotify library)
- Changes: Create, Write, Chmod, Remove, Rename
- Polling as fallback (FileCheckFrequency)

**HTTP source:**
- Byte-level comparison `bytes.Equal(newData, oldData)`
- Polled at HTTPCheckFrequency intervals

### 4.2 Update Processing Flow

**Architecture: Multi-source Mux Pattern**

```
File Source ──┐
              ├─→ [Mux] ──→ podStorage ──→ Updates Channel
HTTP Source ──┤
              └─→ Config merges and delivers change notifications
API Source ───┘
```

**Files:**
- `/pkg/kubelet/config/mux.go` - Channel mux
- `/pkg/kubelet/config/config.go` - Pod storage and merging

**Update operations:**

| Operation | Meaning | Usage |
|-----------|---------|-------|
| SET | Full state replacement | Initial load, periodic polling |
| ADD | New pod from source | Static pod creation |
| UPDATE | Existing pod modified | Static pod manifest changed |
| DELETE | Graceful deletion | Pod scheduled to delete |
| REMOVE | Hard removal | Pod removed from source |
| RECONCILE | Status reconciliation | Pod status changed |

### 4.3 Merging and Change Detection

**Key logic in `/pkg/kubelet/config/config.go` - `merge()` function:**

```go
// For each pod in update:
// 1. Filter invalid pods (check for duplicates per source)
// 2. Annotate with source (config.source)
// 3. Check if pod already exists in current source
//    - If exists: check if semantically different
//      - If different: ADD to updatePods
//      - If same: skip (unless status changed → RECONCILE)
//    - If new: ADD to addPods
// 4. Detect removed pods from previous state (REMOVE)
```

**Semantic difference check:**
- Spec
- Labels  
- DeletionTimestamp
- DeletionGracePeriodSeconds
- Annotations (excluding local kubelet annotations)

Local annotations preserved:
- `kubernetes.io/config.source`
- `kubernetes.io/config.hash`
- `kubernetes.io/config.seen`

---

## 5. Static Pod Deletion Process

### 5.1 Deletion Flow

**When static pod file/URL is removed:**

```
File deleted → produceWatchEvent(DELETE) 
             → consumeWatchEvent() 
             → store.Delete()
             → PodUpdate{Op: DELETE, Pods: [pod]}
             → Sent to kubelet pod workers
```

### 5.2 Graceful Deletion

**File:** `/pkg/kubelet/kubelet_pods.go` (line 1315-1324)

Orphaned mirror pod cleanup:

```go
// In HandlePodCleanups():
for _, podFullname := range orphanedMirrorPodFullnames {
    if !kl.podWorkers.IsPodForMirrorPodTerminatingByFullName(podFullname) {
        _, err := kl.mirrorPodClient.DeleteMirrorPod(ctx, podFullname, nil)
        // Log results
    }
}
```

**Process:**
1. Pod manager detects static pod removed
2. `GetPodsAndMirrorPods()` identifies orphaned mirror pods (mirror without static)
3. Mirror pods deleted from API server
4. All container cleanup performed by pod worker

### 5.3 Mirror Pod Deletion

**File:** `/pkg/kubelet/pod/mirror_client.go` (line 116)

```go
func (mc *basicMirrorClient) DeleteMirrorPod(ctx, podFullName string, uid *types.UID) (bool, error) {
    name, namespace, err := kubecontainer.ParsePodFullName(podFullName)
    
    // Delete with preconditions:
    // - GracePeriodSeconds: 0 (immediate)
    // - UID precondition if provided (prevents race conditions)
    err := mc.apiserverClient.CoreV1().Pods(namespace).Delete(
        ctx, name, 
        metav1.DeleteOptions{
            GracePeriodSeconds: &GracePeriodSeconds,
            Preconditions: &metav1.Preconditions{UID: uid}
        }
    )
    
    // Not found or conflict (UID mismatch) = success (not an error)
    if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
        return false, nil
    }
    
    if err != nil {
        logger.Error(err, "Failed deleting a mirror pod")
        return false, nil  // Don't propagate error
    }
    return true, nil
}
```

---

## 6. Configuration Options

### 6.1 Static Pod Configuration

**Kubelet Configuration File** (typically `/etc/kubernetes/kubelet/kubelet-config.yaml`):

```yaml
staticPodPath: /etc/kubernetes/manifests          # File or directory path
fileCheckFrequency: 20s                           # How often to check files
staticPodURL: http://example.com:8080/manifests   # HTTP URL for pods
httpCheckFrequency: 20s                           # How often to check HTTP
staticPodURLHeader:                               # Custom HTTP headers
  X-Custom-Header:
    - value1
    - value2
```

**Command-line flags** (deprecated but still supported):
- `--pod-manifest-path=/etc/kubernetes/manifests`
- `--manifest-url=http://example.com:8080/manifests`

**Where used:**
- `/pkg/kubelet/apis/config/types.go` - Config struct definition
- `/cmd/kubelet/app/options/options.go` - Command-line flag binding
- `/pkg/kubelet/kubelet.go` lines 376-385 - Configuration initialization

### 6.2 Polling Frequencies

Both file and HTTP sources are watched at configurable intervals:
- **FileCheckFrequency** (default: 20s) - Poll cycle for file system changes
- **HTTPCheckFrequency** (default: 20s) - Poll cycle for HTTP changes

These work alongside event-based detection on Linux (inotify).

---

## 7. Static Pod vs Regular Pod Handling Differences

### 7.1 Identification

**Functions:** `/pkg/kubelet/types/pod_update.go`

```go
// IsMirrorPod checks for ConfigMirrorAnnotationKey annotation
func IsMirrorPod(pod *v1.Pod) bool {
    if pod.Annotations == nil {
        return false
    }
    _, ok := pod.Annotations[v1.MirrorPodAnnotationKey]  // "kubelet.kubernetes.io/config.mirror"
    return ok
}

// IsStaticPod checks if source is NOT from API server
func IsStaticPod(pod *v1.Pod) bool {
    source, err := GetPodSource(pod)
    return err == nil && source != ApiserverSource
}
```

**Sources:**
- `FileSource = "file"`
- `HTTPSource = "http"`
- `ApiserverSource = "api"`

### 7.2 Key Differences

| Aspect | Static Pod | Regular Pod |
|--------|-----------|-----------|
| **Source** | File/HTTP | API Server |
| **Management** | Kubelet | API Server/Controllers |
| **Mirror Pod** | Yes, created automatically | No |
| **UID Generation** | MD5 hash of content + source | Assigned by API |
| **Name Mangling** | `{name}-{nodeName}` | As specified |
| **Critical Status** | Always critical | Depends on priority |
| **Tolerations** | NoExecute:Exists (files) | Specified in spec |
| **Eviction** | Won't be evicted (files) | Can be evicted |
| **Status Reporting** | Via mirror pod | Direct |
| **Scheduler** | None (kubelet direct) | Normal scheduling |
| **Deletion** | Remove from source | Delete via API |

### 7.3 Critical Pod Status

**File:** `/pkg/kubelet/types/pod_update.go` (line 159)

```go
func IsCriticalPod(pod *v1.Pod) bool {
    if IsStaticPod(pod) {
        return true  // All static pods are critical
    }
    if IsMirrorPod(pod) {
        return true  // All mirror pods are critical
    }
    if pod.Spec.Priority != nil && IsCriticalPodBasedOnPriority(*pod.Spec.Priority) {
        return true
    }
    return false
}
```

### 7.4 Pod Splitting

**Function:** `/pkg/kubelet/kubelet_pods.go` (line 1504)

```go
func splitPodsByStatic(pods []*v1.Pod) (regular, static []*v1.Pod) {
    for _, pod := range pods {
        if kubetypes.IsMirrorPod(pod) {
            continue  // Exclude mirrors from both lists
        }
        if kubetypes.IsStaticPod(pod) {
            static = append(static, pod)
        } else {
            regular = append(regular, pod)
        }
    }
    return
}
```

Mirror pods are excluded because they're not configuration sources.

### 7.5 Filtering Restrictions

**Static pods cannot have:**
- ClusterTrustBundle projected volume sources
- ResourceClaims (under PreventStaticPodAPIReferences feature)

Enforced in `tryDecodeSinglePod()` and `tryDecodePodList()`.

---

## 8. Use Cases for Static Pods

### 8.1 Control Plane Components

The primary use case - bootstrap and self-hosted control plane:

**Common control plane static pods:**
- `kube-apiserver`
- `kube-controller-manager`
- `kube-scheduler`
- `etcd`

**Why static pods for control plane:**
1. **Bootstrap problem**: API server must be running before kubelet can contact API
2. **Independence**: Kubelet doesn't need API to manage control plane pods
3. **Node co-location**: Each control plane node runs its own copies
4. **Fault tolerance**: Can replace individual control plane nodes easily

**File locations:**
- `/etc/kubernetes/manifests/` - Standard location
- Kubelet watches this directory continuously
- Changes to manifests trigger immediate pod updates

### 8.2 Node-Local System Components

- CNI plugins
- Log aggregators
- Monitoring agents
- Machine-specific services

### 8.3 Bootstrapping Scenarios

- Standalone kubelet nodes
- Edge nodes without API connectivity
- Development/testing without full cluster

### 8.4 HTTP Source Use Cases

Less common but enabled for:
- Dynamic manifest serving
- Configuration from external systems
- Multi-node coordinated pod definitions
- Custom deployment mechanisms

---

## 9. Pod Manager and Mirror Pod Mapping

### 9.1 Pod Manager

**File:** `/pkg/kubelet/pod/pod_manager.go`

Maintains in-memory mappings:

```go
type basicManager struct {
    lock sync.RWMutex
    
    // Regular pods indexed by UID
    podByUID map[kubetypes.ResolvedPodUID]*v1.Pod
    
    // Mirror pods indexed by UID
    mirrorPodByUID map[kubetypes.MirrorPodUID]*v1.Pod
    
    // Full names (namespace/name) for fast lookup
    podByFullName map[string]*v1.Pod
    mirrorPodByFullName map[string]*v1.Pod
    
    // UID translations: mirror UID → static UID
    translationByUID map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID
}
```

### 9.2 Key Methods

| Method | Purpose |
|--------|---------|
| `GetPodByUID(uid)` | Get static pod by UID |
| `GetMirrorPodByPod(pod)` | Get mirror pod for static pod |
| `GetPodByMirrorPod(mirrorPod)` | Get static pod from mirror |
| `GetPodsAndMirrorPods()` | Get all pods + orphaned mirrors |
| `GetStaticPodToMirrorPodMap()` | Map of static to mirror pods |
| `TranslatePodUID(uid)` | Convert mirror UID to static UID |
| `GetUIDTranslations()` | Get complete UID mapping |

### 9.3 Orphaned Mirror Pod Detection

**Orphaned mirrors** = mirrors without corresponding static pods

Detected in `GetPodsAndMirrorPods()`:

```go
for podFullName := range pm.mirrorPodByFullName {
    if _, ok := pm.podByFullName[podFullName]; !ok {
        orphanedMirrorPodFullnames = append(orphanedMirrorPodFullnames, podFullName)
    }
}
```

---

## 10. Lifecycle Summary

```
┌─────────────────────────────────────────────────────────┐
│ Static Pod Lifecycle                                    │
└─────────────────────────────────────────────────────────┘

1. DISCOVERY
   ├─ File source: inotify/polling of StaticPodPath
   ├─ HTTP source: polling of StaticPodURL
   └─ Both: periodic polling at configured frequency

2. PARSING & DEFAULTS
   ├─ Parse YAML/JSON manifest
   ├─ Generate UID from content hash + source + node name
   ├─ Mangle name to {name}-{nodeName}
   ├─ Set namespace to default if empty
   └─ Add tolerations (file source only)

3. POD MANAGER UPDATE
   ├─ Add to pod manager
   ├─ Identify as static pod
   └─ Tag with source annotation

4. MIRROR POD CREATION
   ├─ Create shadow pod in API server
   ├─ Set ConfigMirrorAnnotationKey with hash
   ├─ Set owner reference to Node
   └─ Establish static↔mirror mapping

5. POD SYNC
   ├─ Pod worker syncs pod to runtime
   ├─ Containers created/updated/deleted
   └─ Status written back via mirror pod

6. STATUS REPORTING
   ├─ Container status from runtime
   ├─ Mirror pod status in API server
   └─ Users see status via kubectl

7. UPDATES (file/manifest changes)
   ├─ New manifest detected
   ├─ Parsed with new defaults
   ├─ Compared to existing pod
   ├─ If different: update signal sent
   ├─ Mirror pod reconciled if hash changed
   └─ Pod synced with new config

8. DELETION
   ├─ Pod removed from source (file deleted/URL removed)
   ├─ Detected by inotify or polling
   ├─ Pod marked for deletion
   ├─ Containers gracefully terminated
   ├─ Orphaned mirror pod identified
   ├─ Mirror pod deleted from API
   └─ All resources cleaned up
```

---

## 11. Implementation Files Summary

| File | Purpose |
|------|---------|
| `pkg/kubelet/config/file.go` | File source implementation |
| `pkg/kubelet/config/file_linux.go` | inotify-based watching |
| `pkg/kubelet/config/http.go` | HTTP source implementation |
| `pkg/kubelet/config/common.go` | Pod parsing and defaults |
| `pkg/kubelet/config/config.go` | Multi-source mux and merging |
| `pkg/kubelet/config/mux.go` | Channel multiplexing |
| `pkg/kubelet/types/pod_update.go` | Pod type definitions and helpers |
| `pkg/kubelet/pod/mirror_client.go` | Mirror pod API operations |
| `pkg/kubelet/pod/pod_manager.go` | Pod and mirror pod mapping |
| `pkg/kubelet/kubelet.go` | Kubelet orchestration |
| `pkg/kubelet/kubelet_pods.go` | Pod lifecycle management |
| `pkg/kubelet/apis/config/types.go` | Configuration structs |

---

## Conclusion

Static pods provide Kubernetes with a critical bootstrap and self-hosting mechanism. The implementation uses:

1. **Multiple sources**: File system watching (inotify + polling) and HTTP polling
2. **Smart parsing**: YAML/JSON conversion, validation, and defaults
3. **Mirror pods**: Shadow pods in API server for status reporting
4. **Change detection**: Hashing and content comparison
5. **Pod manager**: Unified mapping of static and mirror pods
6. **Automatic reconciliation**: Mirror pods kept in sync with static pods

The design ensures static pods are treated as critical system pods that won't be evicted, enabling reliable control plane operation and node-local system services.
