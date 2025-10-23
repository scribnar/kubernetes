# Kubelet Architecture - Quick Reference

## Essential Paths

### Entry Points
- **Binary**: `/cmd/kubelet/kubelet.go` → `app.NewKubeletCommand(ctx)`
- **Server Init**: `/cmd/kubelet/app/server.go` → `server.Run(ctx, kubeletServer, kubeletDeps)`

### Core Components (pkg/kubelet/)
```
kubelet.go          - Kubelet struct (137 KB) - Main orchestrator
kubelet_pods.go     - Pod sync logic (109 KB) - syncPod, syncTerminating*
pod_workers.go      - Pod worker state machine (77 KB)
pod_container_deletor.go - Container cleanup
```

### Container Runtime (CRI)
```
container/runtime.go                    - Runtime interface
kuberuntime/kuberuntime_manager.go      - CRI implementation (82 KB)
kuberuntime/kuberuntime_container.go    - Container operations (60 KB)
kuberuntime/kuberuntime_sandbox.go      - Sandbox (pod infra)
```

### Pod Lifecycle Events
```
pleg/pleg.go        - PLEG interface
pleg/generic.go     - Poll-based detection (22 KB)
pleg/evented.go     - Event-driven detection (16 KB)
```

### Resource Management
```
cm/container_manager_linux.go           - Cgroup enforcement
cm/cpumanager/cpu_manager.go            - CPU pinning
cm/memorymanager/                       - Memory allocation
cm/devicemanager/manager.go             - Device plugins (49 KB)
cm/topologymanager/                     - Topology coordination
```

### Volume Management
```
volumemanager/volume_manager.go         - Volume interface
volumemanager/populator/                - Desired volumes
volumemanager/reconciler/               - Actual volumes
volumemanager/cache/                    - State caching
```

### Status & Health
```
status/status_manager.go                - Pod status updates (53 KB)
prober/prober_manager.go                - Health probes (11 KB)
prober/worker.go                        - Probe execution
```

### Resource Pressure
```
eviction/eviction_manager.go            - Eviction policy (25 KB)
eviction/helpers.go                     - Eviction logic (56 KB)
```

## Key Structs

### Kubelet Main
```go
type Kubelet struct {
    podManager          kubepod.Manager         // Desired pods
    podWorkers         PodWorkers              // Running pods
    containerRuntime   kubecontainer.Runtime   // CRI
    pleg               pleg.PodLifecycleEventGenerator
    volumeManager      volumemanager.VolumeManager
    statusManager      status.Manager
    probeManager       prober.Manager
    evictionManager    eviction.Manager
    containerManager   cm.ContainerManager
    // ... 30+ more fields
}
```

### Pod Worker States
```go
const (
    SyncPod       PodWorkerState = iota  // Running
    TerminatingPod                       // Stopping
    TerminatedPod                        // Cleanup
)
```

### UpdatePodOptions
```go
type UpdatePodOptions struct {
    UpdateType kubetypes.SyncPodType  // Add/Update/Kill/Sync
    Pod        *v1.Pod
    MirrorPod  *v1.Pod
    RunningPod *kubecontainer.Pod
    KillPodOptions *KillPodOptions
}
```

## Main Sync Loop Flow

```
Run(ctx, updates)
  ├─ Start PLEG (generic + optional evented)
  ├─ Start managers (volume, status, eviction, etc)
  └─ syncLoop(ctx, updates)
     └─ select on 5 channels:
        1. Config changes → HandlePodAdditions/Updates/Removes
        2. PLEG events → Update runtime cache, sync pods
        3. Periodic sync → Sync waiting pods
        4. Probe updates → Handle health changes
        5. Housekeeping → Cleanup

syncLoopIteration()
  - Runs every ~1 second
  - Reads from one channel
  - Dispatches to handler
```

## Pod State Machine

```
Desired Pod (API)
        ↓
podManager.AddPod()
        ↓
podWorkers.UpdatePod(SyncPod)
        ↓
syncPod() - Create/update containers
        ↓
Running Containers (Happy path ends here)
        ↓
Pod deletion requested
        ↓
podWorkers.UpdatePod(TerminatingPod)
        ↓
syncTerminatingPod() - Graceful stop
        ↓
Containers stopped, grace period done
        ↓
podWorkers.UpdatePod(TerminatedPod)
        ↓
syncTerminatedPod() - Cleanup resources
        ↓
Fully removed from kubelet
```

## Key Interfaces

### Runtime Interface
```go
type Runtime interface {
    SyncPod(ctx, pod, podStatus, secrets, backOff)
    KillPod(ctx, pod, runningPod, gracePeriod)
    GetPodStatus(ctx, uid, name, namespace)
    GetPods(ctx, all bool)
    GarbageCollect(ctx, policy, allReady, evictAll)
    PullImage(ctx, image, credentials, config)
    GetContainerLogs(ctx, pod, containerID, options)
}
```

### StatusManager Interface
```go
type Manager interface {
    Start(ctx context.Context)
    SetPodStatus(pod *v1.Pod, status v1.PodStatus)
    SetContainerReadiness(podUID, containerID, ready)
    TerminatePod(pod *v1.Pod)
    RemoveOrphanedStatuses(pods []*v1.Pod)
}
```

### VolumeManager Interface
```go
type VolumeManager interface {
    Run(ctx, sourcesReady)
    WaitForAttachAndMount(ctx, pod) error
    WaitForUnmount(ctx, pod) error
    GetMountedVolumesForPod(podName) VolumeMap
    GetVolumesInUse() []UniqueVolumeName
}
```

### PodWorkers Interface
```go
type PodWorkers interface {
    UpdatePod(options UpdatePodOptions)
    SyncKnownPods(desiredPods []*v1.Pod) map[UID]PodWorkerSync
    ShouldPodContainersBeTerminating(uid) bool
    ShouldPodRuntimeBeRemoved(uid) bool
    ShouldPodContentBeRemoved(uid) bool
}
```

## Important Constants

**Timing:**
- PLEG relist period: 1 second (generic) / 300 seconds (evented)
- PLEG relist threshold: 3 minutes
- Sync loop tick: 1 second
- Housekeeping period: 2 seconds
- Node status update: 10 seconds
- Container GC: 1 minute
- Image GC: 5 minutes

**Resource Limits:**
- Max pods per node: 110 (configurable)
- Pod eviction timeout: 5 minutes
- Graceful termination period: 30 seconds (default)
- Crash loop backoff: 10s initial, 5m max

**Backoff:**
- Initial: 100ms
- Max: 5 seconds
- Factor: 2x

## Event Types

### PLEG Events
- `ContainerStarted` - Container running
- `ContainerDied` - Container exited
- `ContainerRemoved` - Container GC'd
- `PodSync` - Pod needs sync
- `ContainerChanged` - Container state unknown
- `ConditionMet` - Watch condition satisfied

### Pod Update Types
- `ADD` - New pod from config source
- `UPDATE` - Pod changed
- `REMOVE` - Pod deleted
- `RECONCILE` - Periodic reconciliation
- `DELETE` - Graceful deletion
- `SET` - Replace all pods

## Debugging Entry Points

**Monitor pod state:**
```go
// In kubelet.go
podWorkers.SyncKnownPods(desiredPods) // Pod state snapshot
podManager.GetPods() // Desired pods
```

**Check runtime state:**
```go
// In kubelet.go
containerRuntime.GetPods(ctx, all) // Running containers
pleg.Watch() // Pod lifecycle events
runtimeCache.Get(uid) // Cached pod status
```

**Check volume state:**
```go
// In volumemanager
volumeManager.GetMountedVolumesForPod(podName)
volumeManager.GetVolumesInUse()
```

**Check pod status:**
```go
// In status_manager.go
statusManager.GetPodStatus(uid) // Cached status
```

## Common Patterns

### 1. Desired vs Actual
- **Desired**: API pods + pod manager
- **Actual**: Runtime containers + pod workers
- Kubelet reconciles them continuously

### 2. Three-Phase Termination
- Sync → Terminating → Terminated
- Each phase has different allowed operations
- Status changes between phases

### 3. Event-Driven + Fallback
- PLEG detects container changes
- Periodic sync catches missed events
- Housekeeping does background cleanup

### 4. Resource Enforcement
- Container manager: CPU, memory via cgroups
- Device manager: GPUs, etc
- Topology manager: NUMA awareness
- Eviction manager: Pressure response

### 5. Plugin Architecture
- Container runtime: CRI pluggable
- Volume: In-tree + CSI plugins
- Device: External device plugins
- Probes: Different handlers

## Critical Goroutines

1. **syncLoop** - Main event dispatcher (always running)
2. **Pod workers** - 1 per running pod (lifecycle management)
3. **PLEG** - Detects container changes
4. **Volume manager** - Manages volumes
5. **Status manager** - Updates pod status
6. **Eviction manager** - Monitors resource pressure
7. **Node status** - Periodic node updates
8. **Probes** - Health check workers

## File Reading Strategy

**To Understand:**
1. Entry point: Start with `/cmd/kubelet/app/server.go`
2. Main loop: Read `kubelet.go:syncLoop()` (line 2454)
3. Pod sync: Read `kubelet_pods.go:syncPod()` function
4. Pod workers: Read `pod_workers.go` UpdatePod logic
5. Runtime: Read `kuberuntime/kuberuntime_manager.go:SyncPod()`
6. PLEG: Read `pleg/generic.go:Relist()` function
7. Status: Read `status/status_manager.go:SetPodStatus()`
8. Volumes: Read `volumemanager/reconciler/reconciler.go`
9. Eviction: Read `eviction/eviction_manager.go:synchronize()`
10. Resources: Read `cm/container_manager_linux.go`

## Version-Specific Features

**EventedPLEG** (Feature gate: EventedPLEG):
- CRI event stream instead of polling
- Lower CPU, faster detection
- Falls back to GenericPLEG on errors

**Pod Resource Resizing** (Feature gate: InPlacePodVerticalScaling):
- Resize CPU/memory without restart
- Requires runtime support

**DRA** (Dynamic Resource Allocation):
- Per-container dynamic resources
- Separate from CPU/memory

**User Namespaces** (Feature gate: UserNamespacesStatelessPodsSupport):
- Pod UID mapping isolation

**Node Shutdown** (Feature gate: GracefulNodeShutdown):
- Graceful pod termination on node shutdown

## Quick Grep Tips

Find where syncPod is called:
```bash
grep -r "syncPod(" pkg/kubelet/
```

Find PLEG event handling:
```bash
grep -r "plegCh" pkg/kubelet/
```

Find container operations:
```bash
grep -r "containerRuntime\." pkg/kubelet/
```

Find volume operations:
```bash
grep -r "volumeManager\." pkg/kubelet/
```

Find status updates:
```bash
grep -r "statusManager\." pkg/kubelet/
```
