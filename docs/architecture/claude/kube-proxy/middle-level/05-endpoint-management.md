# Endpoint Management in kube-proxy

**Document Status**: ✅ Complete
**Last Updated**: 2024
**Applies to**: Kubernetes v1.21+

## Table of Contents

1. [Overview](#overview)
2. [Endpoints API (Legacy)](#endpoints-api-legacy)
3. [EndpointSlices API](#endpointslices-api)
4. [EndpointsChangeTracker](#endpointschangetracker)
5. [EndpointSliceCache](#endpointslicecache)
6. [Endpoint Selection and Filtering](#endpoint-selection-and-filtering)
7. [Endpoint Conditions](#endpoint-conditions)
8. [Topology-Aware Routing](#topology-aware-routing)
9. [Integration with syncProxyRules](#integration-with-syncproxyrules)
10. [Performance Implications](#performance-implications)
11. [Migration from Endpoints to EndpointSlices](#migration-from-endpoints-to-endpointslices)
12. [Troubleshooting](#troubleshooting)
13. [Best Practices](#best-practices)
14. [Summary](#summary)

---

## Overview

**Endpoint management** is the process by which kube-proxy tracks which pods (endpoints) back each service and programs the network rules accordingly. This is one of kube-proxy's core responsibilities.

### Evolution: Endpoints → EndpointSlices

Kubernetes has evolved from the **Endpoints API** (v1.0+) to the **EndpointSlices API** (v1.21+ stable) to address scalability challenges.

```mermaid
timeline
    title Endpoint Management Evolution
    2015 (v1.0) : Endpoints API introduced<br/>Single object per service
    2019 (v1.16) : EndpointSlices alpha<br/>Addresses scalability
    2020 (v1.17) : EndpointSlices beta<br/>Default enabled
    2021 (v1.21) : EndpointSlices GA<br/>Stable API
    2024 (v1.33) : Endpoints deprecated<br/>Use EndpointSlices
```

### Why EndpointSlices?

The Endpoints API had fundamental scalability limitations:

| Challenge | Endpoints API | EndpointSlices API |
|-----------|---------------|---------------------|
| **Max endpoints per object** | 1000 | 100 per slice, unlimited slices |
| **Update efficiency** | Update entire object | Update individual slices |
| **Watch traffic** | Full object on any change | Only changed slices |
| **etcd load** | High for large services | Distributed across slices |
| **Network traffic** | O(n) for endpoint change | O(1) for endpoint change |

### Architecture Overview

```mermaid
graph TD
    A[kube-controller-manager] -->|Watches Pods| B[EndpointSlice Controller]
    B -->|Creates/Updates| C[EndpointSlice Objects]

    C -->|Watch| D[kube-proxy EndpointSlice Informer]
    D -->|Events| E[EndpointSliceConfig]
    E -->|Dispatch| F[Proxier OnEndpointSliceUpdate]

    F -->|Track Changes| G[EndpointsChangeTracker]
    G -->|Cache| H[EndpointSliceCache]

    I[syncProxyRules] -->|Checkout Changes| G
    G -->|Apply| J[EndpointsMap]
    J -->|Categorize| K[CategorizeEndpoints]

    K -->|Cluster Endpoints| L[iptables/IPVS Rules]
    K -->|Local Endpoints| L
    K -->|Topology Filtering| L

    style A fill:#326CE5,color:#fff
    style B fill:#4CAF50,color:#fff
    style C fill:#FF9800,color:#fff
    style G fill:#9C27B0,color:#fff
    style L fill:#F44336,color:#fff
```

**High-level flow**:
1. **Controller** watches Pods, creates/updates EndpointSlices
2. **kube-proxy** watches EndpointSlices via informers
3. **Change tracker** caches endpoint changes
4. **syncProxyRules** periodically applies changes to network rules

---

## Endpoints API (Legacy)

The **Endpoints API** is the original mechanism for tracking service endpoints, introduced in Kubernetes v1.0.

### API Structure

**Location**: staging/src/k8s.io/api/core/v1/types.go:6281

```go
type Endpoints struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // Subsets contains the endpoint addresses and ports
    // The full set of endpoints is the union of all subsets
    Subsets []EndpointSubset `json:"subsets,omitempty"`
}

type EndpointSubset struct {
    // Addresses are Ready endpoints
    Addresses []EndpointAddress `json:"addresses,omitempty"`

    // NotReadyAddresses are endpoints that failed readiness checks
    NotReadyAddresses []EndpointAddress `json:"notReadyAddresses,omitempty"`

    // Ports are port numbers available on all addresses in this subset
    Ports []EndpointPort `json:"ports,omitempty"`
}

type EndpointAddress struct {
    // IP is the IP address of the endpoint
    IP string `json:"ip"`

    // Hostname (optional)
    Hostname string `json:"hostname,omitempty"`

    // NodeName indicates which node this endpoint is on
    NodeName *string `json:"nodeName,omitempty"`

    // TargetRef is a reference to the backing Pod
    TargetRef *ObjectReference `json:"targetRef,omitempty"`
}

type EndpointPort struct {
    Name     string   `json:"name,omitempty"`
    Port     int32    `json:"port"`
    Protocol Protocol `json:"protocol,omitempty"`
    AppProtocol *string `json:"appProtocol,omitempty"`
}
```

### Example Endpoints Resource

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
  namespace: default
subsets:
- addresses:
  - ip: 10.244.1.5
    nodeName: node-1
    targetRef:
      kind: Pod
      name: my-app-pod-1
      namespace: default
  - ip: 10.244.2.7
    nodeName: node-2
    targetRef:
      kind: Pod
      name: my-app-pod-2
      namespace: default
  notReadyAddresses:
  - ip: 10.244.3.9
    nodeName: node-3
    targetRef:
      kind: Pod
      name: my-app-pod-3
      namespace: default
  ports:
  - name: http
    port: 8080
    protocol: TCP
```

### Limitations

#### 1. **1000 Endpoint Limit**

**Problem**: Endpoints resources cannot exceed 1000 addresses.

**Behavior when exceeded**:
- Endpoints resource is **truncated** to first 1000
- Annotation added: `endpoints.kubernetes.io/over-capacity: "truncated"`
- Some pods won't receive traffic (unpredictable which ones)

**Code evidence**: Multiple CHANGELOG entries document this limit and truncation behavior.

#### 2. **Single Object Per Service**

**Problem**: All endpoints in one Kubernetes object.

**Impact**:
- Large etcd object size (can exceed 1MB for 1000 endpoints)
- Full object must be read/written on any change
- High etcd load for large services
- Poor performance with 100+ endpoints

#### 3. **Update Inefficiency**

**Problem**: Any endpoint change updates entire object.

**Impact**:
- **Watch traffic**: All watchers (kube-proxy on all nodes) receive full object
- **Network bandwidth**: O(n) traffic for single endpoint change
- **CPU/memory**: All watchers must process full endpoint list

**Example**:
```
Service with 1000 endpoints:
- 1 pod terminates
- Endpoints object updated (entire 1000-endpoint list)
- All 100 nodes receive ~200KB update
- Total network traffic: ~20MB for 1 endpoint change
```

#### 4. **Deprecation Timeline**

| Version | Status |
|---------|--------|
| v1.21 | EndpointSlices GA, Endpoints still supported |
| v1.25 | EndpointSlices default for all new services |
| v1.33+ | **Endpoints API deprecated**, migrate to EndpointSlices |

---

## EndpointSlices API

The **EndpointSlices API** addresses scalability by distributing endpoints across multiple Kubernetes objects.

### API Structure

**Location**: staging/src/k8s.io/api/discovery/v1/types.go:34

```go
type EndpointSlice struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // AddressType specifies the type of addresses in this slice
    // IPv4, IPv6, or FQDN
    AddressType AddressType `json:"addressType"`

    // Endpoints contains up to 1000 endpoints (typically 100)
    Endpoints []Endpoint `json:"endpoints"`

    // Ports contains up to 100 ports
    Ports []EndpointPort `json:"ports,omitempty"`
}

type Endpoint struct {
    // Addresses is a list of IP addresses for this endpoint
    // Typically contains exactly one IP
    Addresses []string `json:"addresses"`

    // Conditions contains readiness/serving/terminating state
    Conditions EndpointConditions `json:"conditions,omitempty"`

    // Hostname (optional)
    Hostname *string `json:"hostname,omitempty"`

    // TargetRef is a reference to the backing Pod
    TargetRef *v1.ObjectReference `json:"targetRef,omitempty"`

    // NodeName indicates which node this endpoint is on
    // Used for Local traffic policy and topology routing
    NodeName *string `json:"nodeName,omitempty"`

    // Zone is the availability zone for this endpoint
    // Used for topology-aware routing
    Zone *string `json:"zone,omitempty"`

    // Hints contains topology hints for routing optimization
    Hints *EndpointHints `json:"hints,omitempty"`
}

type EndpointConditions struct {
    // Ready indicates endpoint is ready to receive traffic
    // Computed as: serving && !terminating
    Ready *bool `json:"ready,omitempty"`

    // Serving indicates endpoint is able to serve traffic
    // Ignores termination state (for graceful termination)
    Serving *bool `json:"serving,omitempty"`

    // Terminating indicates endpoint has deletion timestamp
    // Allows routing to terminating pods if no Ready endpoints
    Terminating *bool `json:"terminating,omitempty"`
}

type EndpointHints struct {
    // ForZones contains zone-level topology hints (max 8)
    ForZones []ForZone `json:"forZones,omitempty"`

    // ForNodes contains node-level topology hints (max 8, alpha)
    ForNodes []ForNode `json:"forNodes,omitempty"`
}

type ForZone struct {
    Name string `json:"name"`
}
```

### Example EndpointSlice Resource

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abcd1
  namespace: default
  labels:
    kubernetes.io/service-name: my-service
    endpointslice.kubernetes.io/managed-by: endpointslice-controller.k8s.io
addressType: IPv4
endpoints:
- addresses:
  - 10.244.1.5
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: node-1
  zone: us-east-1a
  targetRef:
    kind: Pod
    name: my-app-pod-1
    namespace: default
  hints:
    forZones:
    - name: us-east-1a
- addresses:
  - 10.244.2.7
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: node-2
  zone: us-east-1b
  targetRef:
    kind: Pod
    name: my-app-pod-2
    namespace: default
  hints:
    forZones:
    - name: us-east-1b
ports:
- name: http
  port: 8080
  protocol: TCP
```

### Scalability Improvements

#### 1. **Multiple Slices Per Service**

Services can have many EndpointSlice objects:

```
Service: my-large-service (5000 endpoints)
  ├─ my-large-service-abcd1 (100 endpoints)
  ├─ my-large-service-abcd2 (100 endpoints)
  ├─ my-large-service-abcd3 (100 endpoints)
  ...
  └─ my-large-service-abcd50 (100 endpoints)
```

**Benefits**:
- No 1000 endpoint limit
- Slices grouped by label: `kubernetes.io/service-name: my-large-service`
- Parallel processing possible

#### 2. **Efficient Updates**

**Scenario**: 1 pod out of 5000 terminates

**Endpoints API**:
```
Update size: ~1MB (entire Endpoints object)
Network traffic: 1MB × 100 nodes = 100MB
```

**EndpointSlices API**:
```
Update size: ~2KB (single EndpointSlice)
Network traffic: 2KB × 100 nodes = 200KB
```

**Result**: **500x reduction** in network traffic!

#### 3. **Default Slice Size**

The EndpointSlice controller creates slices with **100 endpoints each**:

**Why 100?**
- Balance between object count and object size
- Typical slice size: 5-10KB
- Reasonable etcd object size
- Good trade-off for watch efficiency

**Configurable via**:
```bash
kube-controller-manager --max-endpoints-per-slice=100
```

#### 4. **Dual-Stack Support**

EndpointSlices explicitly support dual-stack:

```yaml
# IPv4 slice
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-ipv4-abcd1
addressType: IPv4
endpoints:
- addresses:
  - 10.244.1.5

---
# IPv6 slice
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-ipv6-abcd1
addressType: IPv6
endpoints:
- addresses:
  - 2001:db8::5
```

**kube-proxy behavior**:
- Filters slices by `addressType` matching IP family
- IPv4 proxier watches only IPv4 slices
- IPv6 proxier watches only IPv6 slices

---

## EndpointsChangeTracker

The **EndpointsChangeTracker** is the component in kube-proxy that receives EndpointSlice events and tracks changes for eventual application.

### Structure

**Location**: pkg/proxy/endpointschangetracker.go:33

```go
type EndpointsChangeTracker struct {
    // lock protects all fields
    lock sync.Mutex

    // processEndpointsMapChange is callback invoked when changes applied
    processEndpointsMapChange processEndpointsMapChangeFunc

    // addressType filters EndpointSlices (IPv4 or IPv6)
    addressType discovery.AddressType

    // endpointSliceCache stores and tracks slice changes
    endpointSliceCache *EndpointSliceCache

    // lastChangeTriggerTimes tracks when changes occurred (for metrics)
    lastChangeTriggerTimes map[types.NamespacedName][]time.Time

    // trackerStartTime is when tracker was created
    trackerStartTime time.Time
}
```

### Initialization

**Location**: pkg/proxy/endpointschangetracker.go:54

```go
func NewEndpointsChangeTracker(
    hostname string,
    makeEndpointInfo makeEndpointFunc,
    ipFamily v1.IPFamily,
    topologyLabels map[string]string,
    processEndpointsMapChange processEndpointsMapChangeFunc,
) *EndpointsChangeTracker {

    return &EndpointsChangeTracker{
        addressType: addressTypeFromIPFamily(ipFamily),
        endpointSliceCache: NewEndpointSliceCache(
            hostname,
            ipFamily,
            topologyLabels,
            makeEndpointInfo,
        ),
        lastChangeTriggerTimes:    make(map[types.NamespacedName][]time.Time),
        processEndpointsMapChange: processEndpointsMapChange,
        trackerStartTime:          time.Now(),
    }
}
```

### Key Methods

#### EndpointSliceUpdate()

**Location**: pkg/proxy/endpointschangetracker.go:81

```go
func (ect *EndpointsChangeTracker) EndpointSliceUpdate(
    endpointSlice *discovery.EndpointSlice,
    removeSlice bool,
) bool {
    ect.lock.Lock()
    defer ect.lock.Unlock()

    // Filter by address type (IPv4/IPv6)
    if endpointSlice.AddressType != ect.addressType {
        return false
    }

    // Delegate to cache
    namespacedName := types.NamespacedName{
        Namespace: endpointSlice.Namespace,
        Name:      endpointSlice.Labels[discovery.LabelServiceName],
    }

    changeNeeded := ect.endpointSliceCache.updatePending(endpointSlice, removeSlice)

    // Track change trigger time for metrics
    if changeNeeded {
        ect.lastChangeTriggerTimes[namespacedName] = append(
            ect.lastChangeTriggerTimes[namespacedName],
            time.Now(),
        )
    }

    return changeNeeded
}
```

**Returns**: `true` if syncProxyRules should be triggered.

#### checkoutChanges()

**Location**: pkg/proxy/endpointschangetracker.go:120

```go
func (ect *EndpointsChangeTracker) checkoutChanges() map[types.NamespacedName]*endpointsChange {
    ect.lock.Lock()
    defer ect.lock.Unlock()

    // Get all pending changes from cache
    changes := ect.endpointSliceCache.checkoutChanges()

    // Clear trigger times for synced changes
    for svcName := range changes {
        delete(ect.lastChangeTriggerTimes, svcName)
    }

    return changes
}
```

**Called by**: `EndpointsMap.Update()` during syncProxyRules.

**Returns**: Map of service name → endpoints change (previous/current).

### Event Flow

```mermaid
sequenceDiagram
    participant API as kube-apiserver
    participant Informer as EndpointSlice Informer
    participant Config as EndpointSliceConfig
    participant Proxier as Proxier
    participant Tracker as EndpointsChangeTracker
    participant Cache as EndpointSliceCache
    participant Sync as syncProxyRules

    API->>Informer: EndpointSlice update
    Informer->>Config: handleUpdateEndpointSlice()
    Config->>Proxier: OnEndpointSliceUpdate()

    Proxier->>Tracker: EndpointSliceUpdate(slice, false)
    Tracker->>Tracker: Filter by addressType
    Tracker->>Cache: updatePending(slice)
    Cache-->>Tracker: changeNeeded = true
    Tracker-->>Proxier: return true

    Proxier->>Proxier: Trigger syncProxyRules

    Note over Sync: Later, during sync...

    Sync->>Tracker: checkoutChanges()
    Tracker->>Cache: checkoutChanges()
    Cache-->>Tracker: map[svcName]*endpointsChange
    Tracker-->>Sync: return changes

    Sync->>Sync: Apply changes to EndpointsMap
    Sync->>Sync: Program iptables/IPVS rules
```

---

## EndpointSliceCache

The **EndpointSliceCache** stores EndpointSlice state and computes diffs for efficient updates.

### Structure

**Location**: pkg/proxy/endpointslicecache.go:34

```go
type EndpointSliceCache struct {
    // lock protects all fields
    lock sync.Mutex

    // trackerByServiceMap stores per-service slice trackers
    // Key: types.NamespacedName (service namespace/name)
    trackerByServiceMap map[types.NamespacedName]*endpointSliceTracker

    // makeEndpointInfo constructs BaseEndpointInfo from EndpointSlice
    makeEndpointInfo makeEndpointFunc

    // nodeName is this node's name (for local endpoint detection)
    nodeName string

    // ipFamily filters EndpointSlices (v1.IPv4Protocol or v1.IPv6Protocol)
    ipFamily v1.IPFamily

    // topologyLabels for this node (zone, region, etc.)
    topologyLabels map[string]string
}

type endpointSliceTracker struct {
    // applied is the currently applied state (last synced)
    applied endpointSliceDataByName

    // pending is the pending state (not yet synced)
    pending endpointSliceDataByName
}

type endpointSliceDataByName map[string]*endpointSliceData

type endpointSliceData struct {
    endpointSlice *discovery.EndpointSlice
    generation    int64
    resourceVersion string
}
```

### Key Operations

#### updatePending()

**Location**: pkg/proxy/endpointslicecache.go:95

```go
func (cache *EndpointSliceCache) updatePending(
    endpointSlice *discovery.EndpointSlice,
    remove bool,
) bool {
    cache.lock.Lock()
    defer cache.lock.Unlock()

    serviceName := types.NamespacedName{
        Namespace: endpointSlice.Namespace,
        Name:      endpointSlice.Labels[discovery.LabelServiceName],
    }

    // Get or create tracker for this service
    tracker := cache.trackerByServiceMap[serviceName]
    if tracker == nil {
        tracker = &endpointSliceTracker{
            applied: make(endpointSliceDataByName),
            pending: make(endpointSliceDataByName),
        }
        cache.trackerByServiceMap[serviceName] = tracker
    }

    sliceName := endpointSlice.Name

    if remove {
        // Mark for deletion
        if tracker.applied[sliceName] != nil || tracker.pending[sliceName] != nil {
            tracker.pending[sliceName] = nil
            return true  // Change detected
        }
        return false  // Already gone
    }

    // Check if different from applied/pending
    newData := &endpointSliceData{
        endpointSlice:   endpointSlice.DeepCopy(),
        generation:      endpointSlice.Generation,
        resourceVersion: endpointSlice.ResourceVersion,
    }

    appliedData := tracker.applied[sliceName]
    pendingData := tracker.pending[sliceName]

    // No change if identical to pending
    if pendingData != nil && pendingData.generation == newData.generation {
        return false
    }

    // No change if identical to applied and no pending
    if appliedData != nil && appliedData.generation == newData.generation && pendingData == nil {
        return false
    }

    // Store as pending
    tracker.pending[sliceName] = newData
    return true  // Change detected
}
```

**Algorithm**:
1. Find or create tracker for service
2. If removing: Mark pending as nil
3. If updating: Compare generation with applied/pending
4. Return `true` if change detected

#### checkoutChanges()

**Location**: pkg/proxy/endpointslicecache.go:122

```go
func (cache *EndpointSliceCache) checkoutChanges() map[types.NamespacedName]*endpointsChange {
    cache.lock.Lock()
    defer cache.lock.Unlock()

    changes := make(map[types.NamespacedName]*endpointsChange)

    for svcName, tracker := range cache.trackerByServiceMap {
        // Skip if no pending changes
        if len(tracker.pending) == 0 {
            continue
        }

        // Compute previous and current EndpointsMap
        previous := cache.endpointsMapFromSliceData(tracker.applied)
        current := cache.endpointsMapFromSliceData(mergeMaps(tracker.applied, tracker.pending))

        // Store change
        changes[svcName] = &endpointsChange{
            previous: previous,
            current:  current,
        }

        // Move pending → applied
        for sliceName, sliceData := range tracker.pending {
            if sliceData == nil {
                delete(tracker.applied, sliceName)
            } else {
                tracker.applied[sliceName] = sliceData
            }
        }

        // Clear pending
        tracker.pending = make(endpointSliceDataByName)

        // Clean up if no more slices
        if len(tracker.applied) == 0 {
            delete(cache.trackerByServiceMap, svcName)
        }
    }

    return changes
}
```

**Algorithm**:
1. For each service with pending changes:
   - Build "previous" EndpointsMap from applied state
   - Build "current" EndpointsMap from applied + pending
   - Create endpointsChange with before/after
2. Move pending → applied
3. Delete tracker if no slices remain

#### Endpoint Processing

**Location**: pkg/proxy/endpointslicecache.go:196-244

```go
func (cache *EndpointSliceCache) addEndpoints(
    endpointSlice *discovery.EndpointSlice,
    endpointsMap EndpointsMap,
) {
    for _, endpoint := range endpointSlice.Endpoints {
        // Extract conditions
        ready := endpoint.Conditions.Ready != nil && *endpoint.Conditions.Ready
        serving := endpoint.Conditions.Serving == nil || *endpoint.Conditions.Serving
        terminating := endpoint.Conditions.Terminating != nil && *endpoint.Conditions.Terminating

        // Determine locality
        isLocal := endpoint.NodeName != nil && *endpoint.NodeName == cache.nodeName

        // Parse topology hints
        var zoneHints, nodeHints sets.Set[string]
        if endpoint.Hints != nil {
            if len(endpoint.Hints.ForZones) > 0 {
                zoneHints = sets.New[string]()
                for _, hint := range endpoint.Hints.ForZones {
                    zoneHints.Insert(hint.Name)
                }
            }
            if len(endpoint.Hints.ForNodes) > 0 {
                nodeHints = sets.New[string]()
                for _, hint := range endpoint.Hints.ForNodes {
                    nodeHints.Insert(hint.Name)
                }
            }
        }

        // For each address (typically one IP)
        for _, address := range endpoint.Addresses {
            // For each port
            for _, port := range endpointSlice.Ports {
                if port.Port == nil || *port.Port == 0 {
                    continue
                }

                // Create BaseEndpointInfo
                endpointInfo := cache.makeEndpointInfo(
                    address,
                    int(*port.Port),
                    isLocal,
                    ready,
                    serving,
                    terminating,
                    zoneHints,
                    nodeHints,
                )

                // Add to EndpointsMap (deduplicates by endpoint string)
                endpointsMap[endpointInfo.String()] = endpointInfo
            }
        }
    }
}
```

**Per-endpoint processing**:
1. Extract conditions (ready, serving, terminating)
2. Determine locality (`nodeName == cache.nodeName`)
3. Parse topology hints (zone, node)
4. Create `BaseEndpointInfo` for each address:port combination
5. Deduplicate by endpoint string (`ip:port`)

---

## Endpoint Selection and Filtering

kube-proxy must decide which endpoints to use based on service traffic policies and endpoint readiness.

### BaseEndpointInfo Structure

**Location**: pkg/proxy/endpoint.go:56

```go
type BaseEndpointInfo struct {
    // ip is the endpoint's IP address
    ip string

    // port is the endpoint's port number
    port int

    // endpoint is cached "ip:port" string
    endpoint string

    // isLocal indicates endpoint is on same node as kube-proxy
    isLocal bool

    // ready = serving && !terminating
    ready bool

    // serving indicates endpoint can serve traffic (ignores termination)
    serving bool

    // terminating indicates endpoint has deletionTimestamp
    terminating bool

    // zoneHints are topology hints for zones
    zoneHints sets.Set[string]

    // nodeHints are topology hints for nodes (alpha feature)
    nodeHints sets.Set[string]
}

// IsReady returns true if endpoint is ready to receive traffic
func (info *BaseEndpointInfo) IsReady() bool {
    return info.ready
}

// IsServing returns true if endpoint can serve traffic
func (info *BaseEndpointInfo) IsServing() bool {
    return info.serving
}

// IsTerminating returns true if endpoint is being deleted
func (info *BaseEndpointInfo) IsTerminating() bool {
    return info.terminating
}

// IsLocal returns true if endpoint is on same node
func (info *BaseEndpointInfo) IsLocal() bool {
    return info.isLocal
}

// ZoneHints returns zone topology hints
func (info *BaseEndpointInfo) ZoneHints() sets.Set[string] {
    return info.zoneHints
}

// NodeHints returns node topology hints
func (info *BaseEndpointInfo) NodeHints() sets.Set[string] {
    return info.nodeHints
}
```

### CategorizeEndpoints Function

**Location**: pkg/proxy/topology.go:48

This is the **core endpoint selection** function used during syncProxyRules:

```go
func CategorizeEndpoints(
    allEndpoints []Endpoint,
    svcInfo ServicePort,
    nodeName string,
    topologyLabels map[string]string,
) (clusterEndpoints, localEndpoints, allReachableEndpoints []Endpoint, hasAnyEndpoints bool) {

    // Determine if service uses cluster/local endpoints
    usesClusterEndpoints := svcInfo.UsesClusterEndpoints()
    usesLocalEndpoints := svcInfo.UsesLocalEndpoints()

    // Determine topology mode from hints
    topologyMode := topologyModeFromHints(allEndpoints, topologyLabels)

    // Check endpoint availability
    hasLocalReadyEndpoints := false
    hasLocalServingTerminatingEndpoints := false
    hasClusterReadyEndpoints := false
    hasClusterServingTerminatingEndpoints := false

    for _, endpoint := range allEndpoints {
        if endpoint.IsReady() {
            hasAnyEndpoints = true
            if endpoint.IsLocal() {
                hasLocalReadyEndpoints = true
            }
            if availableForTopology(endpoint, topologyMode, nodeName, topologyLabels) {
                hasClusterReadyEndpoints = true
            }
        }
        if endpoint.IsServing() && endpoint.IsTerminating() {
            if endpoint.IsLocal() {
                hasLocalServingTerminatingEndpoints = true
            }
            if availableForTopology(endpoint, topologyMode, nodeName, topologyLabels) {
                hasClusterServingTerminatingEndpoints = true
            }
        }
    }

    // Select cluster endpoints (for Cluster traffic policy)
    if usesClusterEndpoints {
        if hasClusterReadyEndpoints {
            // Prefer Ready endpoints with topology filtering
            for _, endpoint := range allEndpoints {
                if endpoint.IsReady() && availableForTopology(endpoint, topologyMode, nodeName, topologyLabels) {
                    clusterEndpoints = append(clusterEndpoints, endpoint)
                }
            }
        } else if hasClusterServingTerminatingEndpoints {
            // Fallback: Use Serving+Terminating
            for _, endpoint := range allEndpoints {
                if endpoint.IsServing() && endpoint.IsTerminating() &&
                    availableForTopology(endpoint, topologyMode, nodeName, topologyLabels) {
                    clusterEndpoints = append(clusterEndpoints, endpoint)
                }
            }
        }
    }

    // Select local endpoints (for Local traffic policy)
    if usesLocalEndpoints {
        if hasLocalReadyEndpoints {
            // Prefer local Ready endpoints
            for _, endpoint := range allEndpoints {
                if endpoint.IsLocal() && endpoint.IsReady() {
                    localEndpoints = append(localEndpoints, endpoint)
                }
            }
        } else if hasLocalServingTerminatingEndpoints {
            // Fallback: Local Serving+Terminating
            for _, endpoint := range allEndpoints {
                if endpoint.IsLocal() && endpoint.IsServing() && endpoint.IsTerminating() {
                    localEndpoints = append(localEndpoints, endpoint)
                }
            }
        }
    }

    // Union of cluster and local
    allReachableEndpoints = append(clusterEndpoints, localEndpoints...)

    return
}
```

### Filtering Algorithm

```mermaid
flowchart TD
    Start[All Endpoints] --> CheckPolicy{Traffic Policy?}

    CheckPolicy -->|Cluster| CheckClusterReady{Has Ready<br/>Cluster Endpoints?}
    CheckPolicy -->|Local| CheckLocalReady{Has Ready<br/>Local Endpoints?}

    CheckClusterReady -->|Yes| UseClusterReady[Use Ready Endpoints<br/>+ Topology Filter]
    CheckClusterReady -->|No| CheckClusterTerm{Has Serving+Terminating<br/>Cluster Endpoints?}
    CheckClusterTerm -->|Yes| UseClusterTerm[Use Serving+Terminating]
    CheckClusterTerm -->|No| NoClusterEndpoints[No Cluster Endpoints]

    CheckLocalReady -->|Yes| UseLocalReady[Use Ready Local Endpoints]
    CheckLocalReady -->|No| CheckLocalTerm{Has Serving+Terminating<br/>Local Endpoints?}
    CheckLocalTerm -->|Yes| UseLocalTerm[Use Local Serving+Terminating]
    CheckLocalTerm -->|No| NoLocalEndpoints[No Local Endpoints<br/>Drop Traffic]

    UseClusterReady --> ClusterEndpoints[clusterEndpoints]
    UseClusterTerm --> ClusterEndpoints
    UseLocalReady --> LocalEndpoints[localEndpoints]
    UseLocalTerm --> LocalEndpoints

    ClusterEndpoints --> Merge[allReachableEndpoints]
    LocalEndpoints --> Merge
    NoClusterEndpoints --> Merge
    NoLocalEndpoints --> Merge

    style Start fill:#326CE5,color:#fff
    style ClusterEndpoints fill:#4CAF50,color:#fff
    style LocalEndpoints fill:#FF9800,color:#fff
    style NoClusterEndpoints fill:#F44336,color:#fff
    style NoLocalEndpoints fill:#F44336,color:#fff
```

### Key Selection Rules

1. **Primary**: Use **Ready** endpoints (`serving && !terminating`)
2. **Fallback**: Use **Serving+Terminating** if no Ready available
3. **Cluster policy**: Filter by topology hints
4. **Local policy**: Filter by node locality
5. **Combine**: Union of cluster and local endpoints

---

## Endpoint Conditions

EndpointSlices introduce fine-grained endpoint state tracking.

### Three Conditions

| Condition | Meaning | Set By | Used For |
|-----------|---------|--------|----------|
| **ready** | `serving && !terminating` | EndpointSlice controller | Primary load balancing |
| **serving** | Pod is ready (ignores termination) | Pod readiness probe | Graceful termination fallback |
| **terminating** | Pod has deletionTimestamp | Kubelet | Avoid terminating pods when possible |

### State Transition Diagram

```mermaid
stateDiagram-v2
    [*] --> Creating: Pod created
    Creating --> NotReady: Container starting
    NotReady --> Ready: Readiness probe passes

    Ready --> Ready: Continuous health checks
    Ready --> NotReady: Readiness probe fails
    NotReady --> Ready: Readiness probe recovers

    Ready --> Terminating: Pod deletion
    NotReady --> Terminating: Pod deletion

    Terminating --> Terminating: Graceful shutdown period
    Terminating --> [*]: Pod removed

    note right of Creating
        serving: false
        ready: false
        terminating: false
    end note

    note right of NotReady
        serving: false
        ready: false
        terminating: false
    end note

    note right of Ready
        serving: true
        ready: true
        terminating: false
    end note

    note right of Terminating
        serving: true (if was ready)
        ready: false
        terminating: true
    end note
```

### Condition Semantics

#### ready

**Definition**: `serving && !terminating`

**Use case**: **Default** for load balancing

```go
ready := (endpoint.Conditions.Ready != nil && *endpoint.Conditions.Ready)
```

**When true**:
- Pod containers running
- Readiness probe passing
- Not being deleted
- **Should receive traffic**

**When false**:
- Containers starting
- Readiness probe failing
- Pod terminating
- **Should not receive new traffic**

#### serving

**Definition**: Pod readiness status (ignores termination)

**Use case**: **Fallback** during graceful termination

```go
serving := (endpoint.Conditions.Serving == nil || *endpoint.Conditions.Serving)
```

**When true**:
- Pod was ready before termination started
- Readiness probe passing (even during termination)
- **Can still serve existing connections**

**When false**:
- Readiness probe failing
- Containers not ready
- **Cannot serve traffic**

#### terminating

**Definition**: Pod has `deletionTimestamp` set

**Use case**: Avoid terminating pods when alternatives exist

```go
terminating := (endpoint.Conditions.Terminating != nil && *endpoint.Conditions.Terminating)
```

**When true**:
- Pod is being deleted
- In graceful termination period (default 30s)
- **Prefer non-terminating endpoints**

**When false**:
- Pod is healthy
- **Preferred for new connections**

### Graceful Termination Flow

```mermaid
sequenceDiagram
    participant Client
    participant Proxy as kube-proxy
    participant Pod as Terminating Pod
    participant NewPod as Healthy Pod

    Note over Pod: kubectl delete pod
    Pod->>Pod: deletionTimestamp set
    Pod->>Pod: PreStop hook (if any)

    Note over Pod: conditions.terminating = true<br/>conditions.ready = false<br/>conditions.serving = true (if was ready)

    Pod->>Proxy: EndpointSlice update
    Proxy->>Proxy: CategorizeEndpoints()

    alt Has Ready endpoints (NewPod)
        Proxy->>Proxy: Use Ready endpoints only
        Client->>Proxy: New connection
        Proxy->>NewPod: Route to healthy pod
        Note over Pod: No new connections
    else No Ready endpoints (only terminating)
        Proxy->>Proxy: Use Serving+Terminating
        Client->>Proxy: New connection
        Proxy->>Pod: Route to terminating pod<br/>(fallback behavior)
        Note over Pod: Receives traffic until terminationGracePeriodSeconds
    end

    Note over Pod: Graceful shutdown (30s default)
    Pod->>Pod: SIGTERM sent to containers
    Pod->>Pod: Drain connections
    Pod->>Pod: SIGKILL after grace period
```

**Key insight**: kube-proxy **prefers Ready endpoints** but **falls back to Serving+Terminating** to avoid total service disruption.

---

## Topology-Aware Routing

Topology-aware routing allows kube-proxy to prefer endpoints in the **same zone** or **same node** to reduce latency and cross-AZ costs.

### Traffic Policy Determination

**Location**: pkg/proxy/serviceport.go:162-173

```go
// UsesClusterEndpoints returns true if service uses cluster-wide endpoints
func (svc *BaseServicePortInfo) UsesClusterEndpoints() bool {
    // Use cluster endpoints if:
    // - InternalTrafficPolicy is Cluster (default), OR
    // - Service is externally accessible (NodePort/LB/ExternalIP)
    return !svc.internalPolicyLocal || svc.ExternallyAccessible()
}

// UsesLocalEndpoints returns true if service uses node-local endpoints
func (svc *BaseServicePortInfo) UsesLocalEndpoints() bool {
    // Use local endpoints if:
    // - InternalTrafficPolicy is Local, OR
    // - ExternalTrafficPolicy is Local AND service is externally accessible
    return svc.internalPolicyLocal ||
           (svc.externalPolicyLocal && svc.ExternallyAccessible())
}
```

### Topology Modes

**Location**: pkg/proxy/topology.go:156

```go
const (
    // No topology constraints
    topologyModeNone = ""

    // Prefer endpoints in same zone (uses zone hints)
    topologyModePreferSameZone = v1.ServiceTrafficDistributionPreferSameZone

    // Prefer endpoints on same node (uses node hints, alpha)
    topologyModePreferSameNode = v1.ServiceTrafficDistributionPreferSameNode
)
```

#### topologyModeFromHints()

**Location**: pkg/proxy/topology.go:164

```go
func topologyModeFromHints(endpoints []Endpoint, topologyLabels map[string]string) string {
    if len(endpoints) == 0 {
        return topologyModeNone
    }

    nodeName := topologyLabels[v1.LabelHostname]
    zone := topologyLabels[v1.LabelTopologyZone]

    // Check for node-level hints (PreferSameNode - alpha)
    if nodeName != "" {
        allHaveNodeHints := true
        anyForThisNode := false

        for _, endpoint := range endpoints {
            if !endpoint.IsReady() {
                continue
            }
            if endpoint.NodeHints().Len() == 0 {
                allHaveNodeHints = false
                break
            }
            if endpoint.NodeHints().Has(nodeName) {
                anyForThisNode = true
            }
        }

        if allHaveNodeHints && anyForThisNode {
            return topologyModePreferSameNode
        }
    }

    // Check for zone-level hints (PreferSameZone)
    if zone != "" {
        allHaveZoneHints := true
        anyForThisZone := false

        for _, endpoint := range endpoints {
            if !endpoint.IsReady() {
                continue
            }
            if endpoint.ZoneHints().Len() == 0 {
                allHaveZoneHints = false
                break
            }
            if endpoint.ZoneHints().Has(zone) {
                anyForThisZone = true
            }
        }

        if allHaveZoneHints && anyForThisZone {
            return topologyModePreferSameZone
        }
    }

    // No topology constraints
    return topologyModeNone
}
```

**Algorithm**:
1. If all Ready endpoints have **node hints** AND at least one hints this node → **PreferSameNode**
2. Else if all Ready endpoints have **zone hints** AND at least one hints this zone → **PreferSameZone**
3. Else → **No topology** (use all endpoints)

#### availableForTopology()

**Location**: pkg/proxy/topology.go:222

```go
func availableForTopology(
    endpoint Endpoint,
    topologyMode string,
    nodeName string,
    topologyLabels map[string]string,
) bool {
    switch topologyMode {
    case topologyModeNone:
        return true  // All endpoints available

    case topologyModePreferSameNode:
        // Only endpoints hinted for this node
        return endpoint.NodeHints().Has(nodeName)

    case topologyModePreferSameZone:
        // Only endpoints hinted for this zone
        zone := topologyLabels[v1.LabelTopologyZone]
        return endpoint.ZoneHints().Has(zone)

    default:
        return true
    }
}
```

### Topology Hints Example

**Scenario**: Service with 6 endpoints across 3 zones

```yaml
# EndpointSlice for zone us-east-1a
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abcd1
addressType: IPv4
endpoints:
- addresses: ["10.1.1.5"]
  nodeName: node-1a
  zone: us-east-1a
  hints:
    forZones:
    - name: us-east-1a  # Should be used by proxies in 1a
- addresses: ["10.1.1.7"]
  nodeName: node-2a
  zone: us-east-1a
  hints:
    forZones:
    - name: us-east-1a

---
# EndpointSlice for zone us-east-1b
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abcd2
addressType: IPv4
endpoints:
- addresses: ["10.1.2.5"]
  nodeName: node-1b
  zone: us-east-1b
  hints:
    forZones:
    - name: us-east-1b  # Should be used by proxies in 1b
- addresses: ["10.1.2.7"]
  nodeName: node-2b
  zone: us-east-1b
  hints:
    forZones:
    - name: us-east-1b
```

**kube-proxy on node in us-east-1a**:
1. Detects topology mode: **PreferSameZone** (all endpoints have zone hints)
2. Filters endpoints: Only uses endpoints with hint `us-east-1a`
3. Result: Traffic stays in-zone (2 endpoints)

**Benefits**:
- ✅ Lower latency (no cross-AZ hops)
- ✅ Reduced cost (AWS charges for cross-AZ traffic)
- ✅ Better resource locality

**Trade-offs**:
- ⚠️ Uneven load if zones have different client counts
- ⚠️ Requires EndpointSlice controller to set hints correctly

---

## Integration with syncProxyRules

The endpoint management system integrates into the main syncProxyRules flow.

### Example: iptables Proxier

**Location**: pkg/proxy/iptables/proxier.go:937

```go
func (proxier *Proxier) syncProxyRules() {
    // ... (service iteration)

    for svcName, svcInfo := range proxier.svcPortMap {
        // Get all endpoints for this service
        allEndpoints := proxier.endpointsMap[svcName]

        // Categorize endpoints based on traffic policy
        clusterEndpoints, localEndpoints, allLocallyReachableEndpoints, hasEndpoints :=
            proxy.CategorizeEndpoints(
                allEndpoints,
                svcInfo,
                proxier.nodeName,
                proxier.topologyLabels,
            )

        // Determine which endpoint set to use
        var usableEndpoints []proxy.Endpoint
        if svcInfo.UsesClusterEndpoints() {
            usableEndpoints = clusterEndpoints
        }
        if svcInfo.UsesLocalEndpoints() {
            usableEndpoints = append(usableEndpoints, localEndpoints...)
        }

        // Skip if no endpoints
        if len(usableEndpoints) == 0 {
            klog.V(2).InfoS("Service has no endpoints", "service", svcName)
            continue
        }

        // Generate iptables rules for each endpoint
        for _, endpoint := range usableEndpoints {
            endpointChain := servicePortEndpointChainName(svcName, endpoint.String())

            // Create KUBE-SEP-XXX chain
            natChains.Write(utiliptables.MakeChainLine(endpointChain))

            // Write DNAT rule
            natRules.Write(
                "-A", string(endpointChain),
                "-p", string(svcInfo.Protocol()),
                "-j", "DNAT",
                "--to-destination", endpoint.String(),
            )
        }

        // ... (probability-based load balancing between endpoints)
    }
}
```

### Example: IPVS Proxier

**Location**: pkg/proxy/ipvs/proxier.go:1273

```go
func (proxier *Proxier) syncProxyRules() {
    // ... (service iteration)

    for svcName, svcInfo := range proxier.svcPortMap {
        // Get all endpoints for this service
        allEndpoints := proxier.endpointsMap[svcName]

        // Categorize endpoints
        clusterEndpoints, localEndpoints, _, hasEndpoints :=
            proxy.CategorizeEndpoints(
                allEndpoints,
                svcInfo,
                proxier.nodeName,
                proxier.topologyLabels,
            )

        // Create IPVS virtual server for ClusterIP
        serv := &utilipvs.VirtualServer{
            Address:   svcInfo.ClusterIP(),
            Port:      uint16(svcInfo.Port()),
            Protocol:  string(svcInfo.Protocol()),
            Scheduler: proxier.ipvsScheduler,
        }

        // Sync real servers (endpoints)
        if svcInfo.UsesClusterEndpoints() {
            proxier.syncRealServers(serv, clusterEndpoints)
        }
        if svcInfo.UsesLocalEndpoints() {
            proxier.syncRealServers(serv, localEndpoints)
        }
    }
}

func (proxier *Proxier) syncRealServers(vs *utilipvs.VirtualServer, endpoints []proxy.Endpoint) {
    for _, endpoint := range endpoints {
        rs := &utilipvs.RealServer{
            Address: net.ParseIP(endpoint.IP()),
            Port:    uint16(endpoint.Port()),
            Weight:  1,
        }

        // Add real server to virtual server
        if err := proxier.ipvs.AddRealServer(vs, rs); err != nil {
            klog.ErrorS(err, "Failed to add real server")
        }
    }
}
```

### Flow Summary

```mermaid
flowchart LR
    A[syncProxyRules] --> B[Iterate Services]
    B --> C[Get allEndpoints<br/>from endpointsMap]
    C --> D[CategorizeEndpoints]

    D --> E[clusterEndpoints]
    D --> F[localEndpoints]
    D --> G[hasEndpoints]

    E --> H{Traffic Policy?}
    F --> H

    H -->|Cluster| I[Use clusterEndpoints]
    H -->|Local| J[Use localEndpoints]
    H -->|Both| K[Use both sets]

    I --> L[Generate Rules]
    J --> L
    K --> L

    L -->|iptables| M[KUBE-SEP-XXX chains<br/>DNAT rules]
    L -->|IPVS| N[Real servers<br/>Weight = 1]

    style A fill:#326CE5,color:#fff
    style D fill:#4CAF50,color:#fff
    style L fill:#FF9800,color:#fff
```

---

## Performance Implications

Endpoint management has significant performance impact at scale.

### Endpoints API Performance

| Metric | Small (10 eps) | Medium (100 eps) | Large (1000 eps) | Huge (5000 eps) |
|--------|----------------|------------------|------------------|-----------------|
| **Object size** | ~1KB | ~10KB | ~100KB | **Truncated** |
| **Watch traffic (1 change)** | 1KB × nodes | 10KB × nodes | 100KB × nodes | N/A |
| **etcd load** | Low | Medium | High | **Excessive** |
| **CPU per sync** | <1ms | ~5ms | ~50ms | N/A |

**Problems at scale**:
- 1000 endpoint limit forces truncation
- Full object updated on any change (O(n) traffic)
- Large etcd objects cause memory pressure
- All nodes process full endpoint list

### EndpointSlices API Performance

| Metric | Small (10 eps) | Medium (100 eps) | Large (1000 eps) | Huge (5000 eps) |
|--------|----------------|------------------|------------------|-----------------|
| **Object count** | 1 slice | 1 slice | 10 slices | 50 slices |
| **Object size** | ~1KB | ~10KB | ~10KB (each) | ~10KB (each) |
| **Watch traffic (1 change)** | ~1KB × nodes | ~2KB × nodes | ~2KB × nodes | ~2KB × nodes |
| **etcd load** | Low | Low | Medium | Medium |
| **CPU per sync** | <1ms | <1ms | ~2ms | ~10ms |

**Improvements**:
- **500x reduction** in watch traffic for large services
- **Constant** object size (100 endpoints per slice)
- **Incremental** updates (only changed slices)
- **No limit** on total endpoints

### Network Traffic Comparison

**Scenario**: Service with 5000 endpoints, 1 endpoint terminates, 100 nodes in cluster

#### Endpoints API
```
Update size: ~500KB (entire Endpoints object)
Network traffic: 500KB × 100 nodes = 50MB
Processing: All 100 nodes reprocess 5000 endpoints
```

#### EndpointSlices API
```
Update size: ~2KB (single EndpointSlice)
Network traffic: 2KB × 100 nodes = 200KB
Processing: All nodes process 100-endpoint slice only
```

**Result**: **250x reduction** in network traffic!

### CPU and Memory

**EndpointsChangeTracker overhead**:
- **Memory**: O(slices) - typically <1MB per service
- **CPU (per update)**: O(1) - hash table lookup
- **CPU (per sync)**: O(endpoints) - iterate all endpoints once

**EndpointSliceCache overhead**:
- **Memory**: O(slices × 2) - stores applied + pending
- **Typical**: 10 slices × 2 × 10KB = 200KB per service
- **CPU**: O(slices) - iterate changed slices only

### Optimization Strategies

#### 1. **Reduce Resync Period**

Default resync: 30 seconds (re-syncs all endpoints)

```go
// Increase resync period to reduce CPU
endpointSliceInformer := factory.Discovery().V1().EndpointSlices()
endpointSliceInformer.Informer().SetWatchErrorHandler(...)
// Resync period set at factory level (typically 30s-60s)
```

**Trade-off**: Longer recovery time from missed watch events.

#### 2. **Enable Topology Hints**

Reduces endpoints per node:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/topology-mode: auto
```

**Effect**: Each node uses ~33% of endpoints (in 3-zone cluster).

#### 3. **Use IPVS Mode**

IPVS has O(1) endpoint lookup vs iptables O(n):

```bash
kube-proxy --proxy-mode=ipvs
```

**Benefit**: Constant-time load balancing, better for >1000 endpoints.

---

## Migration from Endpoints to EndpointSlices

### Feature Gate Timeline

| Version | Feature Gate | Status |
|---------|--------------|--------|
| v1.16 | EndpointSlice alpha | Disabled by default |
| v1.17 | EndpointSlice beta | **Enabled by default** |
| v1.21 | EndpointSlice GA | Always enabled |
| v1.33+ | Endpoints deprecated | Use EndpointSlices |

### Migration Steps

#### For Cluster Administrators

1. **Verify EndpointSlices Enabled** (v1.17+):
```bash
kubectl get endpointslices --all-namespaces
```

2. **Check kube-proxy Configuration**:
```bash
kubectl -n kube-system get cm kube-proxy -o yaml | grep -A 10 endpointSlices
```

Expected (v1.21+):
```yaml
# EndpointSlices always enabled, no configuration needed
```

3. **Monitor Endpoint Sync Performance**:
```bash
# Check sync latency metrics
curl localhost:10249/metrics | grep kubeproxy_sync_proxy_rules_duration_seconds
```

4. **Verify No Truncated Endpoints**:
```bash
# Check for over-capacity annotation
kubectl get endpoints --all-namespaces -o json | \
  jq '.items[] | select(.metadata.annotations["endpoints.kubernetes.io/over-capacity"] == "truncated")'
```

If found, services have >1000 endpoints and were truncated. EndpointSlices solves this.

#### For Application Developers

**No changes required** - EndpointSlices are transparent to applications.

Service discovery still works via:
- DNS (CoreDNS/kube-dns)
- Environment variables
- Kubernetes API (Service/Endpoints/EndpointSlices)

#### For Custom Controllers

If your controller watches Endpoints:

**Before** (Endpoints API):
```go
import corev1 "k8s.io/api/core/v1"

// Watch Endpoints
endpointsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        endpoints := obj.(*corev1.Endpoints)
        // Process endpoints
    },
})
```

**After** (EndpointSlices API):
```go
import discoveryv1 "k8s.io/api/discovery/v1"

// Watch EndpointSlices
endpointSliceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        slice := obj.(*discoveryv1.EndpointSlice)
        serviceName := slice.Labels[discoveryv1.LabelServiceName]
        // Process slice for service
    },
})
```

**Key differences**:
- Multiple slices per service (group by label)
- Conditions instead of addresses/notReadyAddresses
- Topology hints for zone/node awareness

---

## Troubleshooting

### Problem: Endpoints not updating

**Symptoms**:
- New pods not receiving traffic
- Deleted pods still receiving traffic
- `kubectl get endpoints` shows stale data

**Diagnosis**:

```bash
# Check EndpointSlice controller is running
kubectl -n kube-system get pods -l k8s-app=kube-controller-manager

# Check EndpointSlices for service
kubectl get endpointslices -n <namespace> -l kubernetes.io/service-name=<service>

# Check kube-proxy is running
kubectl -n kube-system get pods -l k8s-app=kube-proxy

# Check kube-proxy logs
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100 | grep -i endpoint
```

**Common causes**:
1. ❌ EndpointSlice controller not running
2. ❌ kube-proxy not watching EndpointSlices
3. ❌ Network partition preventing watch updates
4. ❌ Pod selector mismatch (no pods match service selector)

**Solution**:

```bash
# Restart EndpointSlice controller
kubectl -n kube-system rollout restart deployment/kube-controller-manager

# Restart kube-proxy
kubectl -n kube-system delete pods -l k8s-app=kube-proxy

# Verify pod labels match service selector
kubectl get pods -n <namespace> --show-labels
kubectl get svc <service> -n <namespace> -o jsonpath='{.spec.selector}'
```

### Problem: Endpoints truncated (>1000)

**Symptoms**:
- Service has >1000 pods
- Some pods not receiving traffic
- Annotation: `endpoints.kubernetes.io/over-capacity: "truncated"`

**Diagnosis**:

```bash
# Check for truncation
kubectl get endpoints <service> -o yaml | grep over-capacity

# Count endpoints
kubectl get endpoints <service> -o json | jq '.subsets[].addresses | length'

# Count EndpointSlices (should be multiple)
kubectl get endpointslices -l kubernetes.io/service-name=<service> --no-headers | wc -l
```

**Cause**: Using legacy Endpoints API with >1000 endpoints.

**Solution**: **Use EndpointSlices** (enabled by default v1.17+).

Verify:
```bash
# Should show multiple slices, each with ~100 endpoints
kubectl get endpointslices -l kubernetes.io/service-name=<service>
```

### Problem: High kube-proxy CPU usage

**Symptoms**:
- kube-proxy consuming >100% CPU
- High sync latency (`kubeproxy_sync_proxy_rules_duration_seconds_count`)
- Frequent sync triggers

**Diagnosis**:

```bash
# Check sync latency
kubectl -n kube-system exec <kube-proxy-pod> -- curl -s localhost:10249/metrics | \
  grep kubeproxy_sync_proxy_rules_duration_seconds

# Check sync frequency
kubectl -n kube-system logs <kube-proxy-pod> --tail=100 | grep "syncProxyRules took"

# Count endpoints
kubectl get endpointslices --all-namespaces --no-headers | wc -l
```

**Common causes**:
1. ❌ Too many endpoints (>10,000)
2. ❌ Frequent endpoint churn (pods restarting)
3. ❌ Using iptables mode with large cluster
4. ❌ Short resync period

**Solution**:

```bash
# Switch to IPVS mode for better performance
kubectl -n kube-system edit cm kube-proxy
# Set: mode: "ipvs"

# Increase resync period
# (Edit kube-proxy config)
# sync-period: "60s"  # Default: 30s

# Enable topology hints to reduce endpoints per node
kubectl annotate service <service> service.kubernetes.io/topology-mode=auto
```

### Problem: Traffic not load balanced evenly

**Symptoms**:
- Some endpoints receive more traffic than others
- Uneven CPU/memory usage across pods

**Diagnosis**:

```bash
# Check endpoint conditions
kubectl get endpointslices -l kubernetes.io/service-name=<service> -o yaml | \
  grep -A 5 conditions

# Check if topology hints are set
kubectl get endpointslices -l kubernetes.io/service-name=<service> -o yaml | \
  grep -A 5 hints

# Check traffic policy
kubectl get svc <service> -o jsonpath='{.spec.externalTrafficPolicy}'
kubectl get svc <service> -o jsonpath='{.spec.internalTrafficPolicy}'
```

**Common causes**:
1. ❌ externalTrafficPolicy=Local with uneven pod distribution
2. ❌ Topology hints causing zone-local routing
3. ❌ Some endpoints NotReady (readiness probe failing)
4. ❌ IPVS scheduler not appropriate for workload

**Solution**:

```bash
# Use Cluster traffic policy for even distribution
kubectl patch svc <service> -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'

# Check readiness probes
kubectl describe pods -l app=<app-label>

# Change IPVS scheduler
kubectl -n kube-system edit cm kube-proxy
# Set: scheduler: "rr"  # Round-robin for even distribution
```

---

## Best Practices

### 1. **Use EndpointSlices (Not Endpoints)**

✅ **Always enabled** in v1.21+
✅ **Required** for >1000 endpoints per service
✅ **Better performance** at scale

### 2. **Configure Readiness Probes**

Proper readiness probes ensure only healthy pods receive traffic:

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      successThreshold: 1
      failureThreshold: 3
```

**Effect**: Endpoints marked NotReady when probe fails.

### 3. **Set Graceful Termination Period**

Allow connections to drain during pod shutdown:

```yaml
apiVersion: v1
kind: Pod
spec:
  terminationGracePeriodSeconds: 30  # Default: 30s
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15"]
```

**Effect**: kube-proxy routes to Serving+Terminating during grace period if no Ready endpoints.

### 4. **Use Topology Hints for Multi-Zone Clusters**

Reduce cross-zone traffic and costs:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/topology-mode: auto
```

**Benefits**:
- Lower latency
- Reduced AWS/GCP cross-AZ costs
- Better resource locality

**Trade-off**: May cause uneven load if zones have different client distribution.

### 5. **Choose Traffic Policy Carefully**

| Policy | Use Case | Pros | Cons |
|--------|----------|------|------|
| **externalTrafficPolicy=Cluster** | Default | Even load distribution | Source IP lost, extra hops |
| **externalTrafficPolicy=Local** | Need source IP | Preserve client IP | Uneven load, fewer endpoints |
| **internalTrafficPolicy=Local** | Reduce latency | Fewer hops | May fail if no local endpoints |

### 6. **Monitor Endpoint Sync Performance**

```bash
# Prometheus metrics
kubeproxy_sync_proxy_rules_duration_seconds_bucket
kubeproxy_sync_proxy_rules_duration_seconds_count
kubeproxy_sync_proxy_rules_last_timestamp_seconds

# High sync latency may indicate:
# - Too many endpoints
# - Need to switch to IPVS mode
# - Need to reduce endpoint churn
```

### 7. **Use IPVS for Large Clusters**

Switch when:
- >1000 services
- >5000 endpoints total
- Sync latency >1 second

```bash
kube-proxy --proxy-mode=ipvs --ipvs-scheduler=rr
```

---

## Summary

### Key Takeaways

1. **EndpointSlices replace Endpoints**:
   - Addresses 1000 endpoint limit
   - Reduces watch traffic by 500x
   - Enabled by default v1.17+

2. **Three endpoint conditions**:
   - **ready**: Primary for load balancing (`serving && !terminating`)
   - **serving**: Fallback during termination
   - **terminating**: Avoid if alternatives exist

3. **Endpoint selection algorithm**:
   - Prefer Ready endpoints
   - Fallback to Serving+Terminating
   - Apply topology filtering
   - Respect traffic policy (Cluster/Local)

4. **Topology-aware routing**:
   - Zone hints reduce cross-AZ traffic
   - Node hints enable same-node routing (alpha)
   - Automatic fallback if hints unavailable

5. **Performance at scale**:
   - EndpointSlices: O(1) watch traffic per change
   - Endpoints: O(n) watch traffic per change
   - IPVS recommended for >1000 endpoints

### Architecture Summary

```mermaid
graph TD
    A[EndpointSlice Controller] -->|Creates| B[EndpointSlice Objects]
    B -->|Watch| C[kube-proxy Informer]
    C -->|Events| D[EndpointsChangeTracker]
    D -->|Cache| E[EndpointSliceCache]
    E -->|Changes| F[syncProxyRules]

    F -->|Checkout| D
    F -->|CategorizeEndpoints| G[Endpoint Selection]

    G -->|Ready endpoints| H[Primary]
    G -->|Serving+Terminating| I[Fallback]
    G -->|Topology filter| J[Zone/Node hints]
    G -->|Traffic policy| K[Cluster/Local]

    H --> L[iptables/IPVS Rules]
    I --> L
    J --> L
    K --> L

    style A fill:#326CE5,color:#fff
    style E fill:#4CAF50,color:#fff
    style G fill:#FF9800,color:#fff
    style L fill:#F44336,color:#fff
```

### Critical Files Reference

| Component | File | Lines |
|-----------|------|-------|
| **Endpoints API** | `staging/src/k8s.io/api/core/v1/types.go` | 6281-6297 |
| **EndpointSlice API** | `staging/src/k8s.io/api/discovery/v1/types.go` | 34-242 |
| **EndpointsChangeTracker** | `pkg/proxy/endpointschangetracker.go` | 33-292 |
| **EndpointSliceCache** | `pkg/proxy/endpointslicecache.go` | 34-336 |
| **BaseEndpointInfo** | `pkg/proxy/endpoint.go` | 56-151 |
| **CategorizeEndpoints** | `pkg/proxy/topology.go` | 48-246 |
| **Traffic Policy** | `pkg/proxy/serviceport.go` | 162-173 |
| **Event Handlers** | `pkg/proxy/config/config.go` | 38-164 |

### Next Steps

For deeper understanding:
- **Service watching**: See `01-service-watch.md`
- **Service types**: See `04-service-types.md`
- **Traffic policies**: See `07-external-traffic-policy.md` (to be written)
- **Session affinity**: See `06-session-affinity.md` (to be written)

---

**Document Complete**: This document provides comprehensive coverage of endpoint management in kube-proxy, from the legacy Endpoints API through the modern EndpointSlices API, including change tracking, selection algorithms, topology-aware routing, and performance optimization.
