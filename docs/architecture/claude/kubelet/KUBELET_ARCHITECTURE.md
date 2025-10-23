# Kubelet Architecture Study

## Overview

The kubelet is the primary "node agent" that runs on each Kubernetes node. It is responsible for:
- Syncing pod specifications with running containers
- Managing pod lifecycle and container runtime integration
- Enforcing resource management and QoS policies
- Collecting metrics and managing node status
- Implementing container networking and volume management

## Main Entry Points

### Binary Entry Point: `/cmd/kubelet/kubelet.go`
```
cmd/kubelet/kubelet.go (40 lines)
  └─> main()
      └─> app.NewKubeletCommand(ctx)
          └─> app.Run() -> cmd/kubelet/app/server.go
```

**Key Flow:**
1. `NewKubeletCommand()` - Creates cobra CLI command
2. Parses flags and configuration files
3. `UnsecuredDependencies()` - Builds kubelet dependencies
4. `Run()` - Starts kubelet server and main loop
5. `syncLoop()` - Main event-driven loop

### Server Setup: `/cmd/kubelet/app/server.go`

**Key Functions:**
- `NewKubeletCommand()` - CLI command setup
- `loadConfigFile()` - Loads kubelet config from file
- `UnsecuredDependencies()` - Creates Dependencies struct
- `Run()` - Initializes and runs kubelet
- `PreInitRuntimeService()` - Initializes CRI connection

**Dependencies Struct** (`pkg/kubelet/kubelet.go:303-334`):
- CAdvisor interface for metrics
- Container manager for resource management
- Remote runtime/image services
- Volume plugins
- Pod configuration sources
- Health checker, tracer provider, etc.

## Core Kubelet Struct

**Location:** `/pkg/kubelet/kubelet.go:1107`

### Main Fields:

**Pod Management:**
- `podManager` - Desired set of admitted pods (from API, files, HTTP)
- `podWorkers` - Executes pod lifecycle state machine
- `podCache` - Cached pod status from container runtime

**Container Runtime Integration:**
- `containerRuntime` - CRI-based container runtime
- `runtimeService` - gRPC connection to runtime
- `runtimeCache` - Caches runtime pod/container info
- `pleg` - Pod Lifecycle Event Generator
- `eventedPleg` - Event-driven PLEG (when enabled)

**Resource Management:**
- `containerManager` - Manages cgroups, CPU, memory, device allocation
- `volumeManager` - Handles volume attach/mount/unmount/detach
- `evictionManager` - Responds to resource pressure
- `allocationManager` - Manages pod resource allocation

**Status & Probing:**
- `statusManager` - Updates pod status to API
- `probeManager` - Runs liveness/readiness/startup probes
- `livenessManager`, `readinessManager`, `startupManager` - Probe result managers

**Other Managers:**
- `imageManager` - Image garbage collection
- `secretManager` - Caches pod secrets
- `configMapManager` - Caches pod config maps
- `pluginManager` - Device plugin manager
- `serverCertificateManager` - Certificate rotation

**Runtime State:**
- `runtimeState` - Tracks runtime health
- `nodeLeaseController` - Node lease renewal
- `oomWatcher` - Out-of-memory watcher
- `cadvisor` - Metrics collection

## Architecture Patterns

### 1. Main Sync Loop (`kubelet.go:2454`)

```
syncLoop()
  ├─> Initialization: initializeModules(), start managers
  ├─> Start PLEG (Pod Lifecycle Event Generator)
  ├─> Start status manager and components
  └─> syncLoopIteration() - Main event loop
      ├─> Config channel: HandlePodAdditions/Updates/Removes
      ├─> PLEG channel: Update runtime cache, sync pods
      ├─> Periodic sync: Sync all pods waiting for sync
      ├─> Housekeeping: Cleanup pods and resources
      └─> Health manager: Sync pods with failed probes
```

**Event Sources:**
1. Config updates (API, files, HTTP)
2. PLEG events (container state changes)
3. Periodic sync tick (every 1 second check)
4. Housekeeping tick (every 2 seconds)
5. Probe updates (liveness, readiness, startup)

### 2. Pod Worker Pattern (`pod_workers.go`)

Each pod has a dedicated goroutine (pod worker) that manages its lifecycle:

**Pod Worker States:**
- `SyncPod` - Pod should be running
- `TerminatingPod` - Pod is being torn down
- `TerminatedPod` - Pod cleanup complete

**PodWorkers Interface** (`pod_workers.go:157`):
```go
UpdatePod(options UpdatePodOptions)              // Queue pod for sync
SyncKnownPods(desiredPods []*v1.Pod)             // Reconcile desired vs running
ShouldPodContainersBeTerminating(uid)            // Check if terminating
ShouldPodRuntimeBeRemoved(uid)                   // Check if runtime should be cleaned
ShouldPodContentBeRemoved(uid)                   // Check if all content should be removed
```

**Work Queue:**
- FIFO queue per pod UID
- Backoff for failed syncs
- Periodic resync intervals

### 3. Pod Lifecycle Event Generator (PLEG)

**Location:** `/pkg/kubelet/pleg/`

**Purpose:** Detect container state changes and generate events

**Two Implementations:**

1. **GenericPLEG** (`generic.go`):
   - Periodic relisting of containers from runtime
   - Default relist period: 1 second
   - Threshold: 3 minutes
   - Compares pod state snapshots to detect changes

2. **EventedPLEG** (`evented.go`) - When EventedPLEG feature gate enabled:
   - Uses CRI event stream for real-time updates
   - Falls back to GenericPLEG on stream errors
   - Lower latency, higher resource efficiency
   - Relist period: 300 seconds (fallback)

**Events Generated:**
- `ContainerStarted` - Container is running
- `ContainerDied` - Container exited
- `ContainerRemoved` - Container garbage collected
- `PodSync` - General pod resync needed
- `ContainerChanged` - Container state unknown
- `ConditionMet` - Watch condition satisfied

### 4. Container Runtime Integration (CRI)

**Location:** `/pkg/kubelet/container/runtime.go`

**Runtime Interface:**
```go
type Runtime interface {
    SyncPod(ctx, pod, podStatus, secrets, backOff)    // Create/update pod
    KillPod(ctx, pod, runningPod, gracePeriod)        // Stop pod
    GetPodStatus(ctx, uid, name, namespace)           // Query pod status
    GetPods(ctx, all bool)                            // List all pods
    GarbageCollect(ctx, policy, allReady, evictAll)   // Cleanup dead containers
    PullImage(ctx, image, credentials, config)        // Pull container image
    GetContainerLogs(ctx, pod, containerID, options)  // Get container logs
    // ... more methods
}
```

**Implementation:** `kuberuntime/` package (CRI-based runtime manager)

**Key Components:**
- `KubeGenericRuntimeManager` - Main CRI runtime implementation
- Container creation/stopping/restarting logic
- Sandbox (pod infrastructure container) management
- Security context enforcement
- Device and volume mounting

### 5. Resource Management (`pkg/kubelet/cm/`)

**Container Manager Interface** (`cm/container_manager.go`):
- Cgroup creation and management
- Resource enforcement (CPU, memory)
- Device plugin support
- QoS class management

**Sub-managers:**

1. **CPU Manager** (`cpumanager/`):
   - Static policy: Pin containers to specific CPUs
   - None policy: Default (no pinning)
   - Supports CPU topology awareness
   - Maintains CPU allocation state

2. **Memory Manager** (`memorymanager/`):
   - Static allocation of memory
   - NUMA-aware memory assignment
   - Memory QoS enforcement

3. **Device Manager** (`devicemanager/`):
   - Device plugin discovery and management
   - Device allocation per pod
   - Topology hints for device placement
   - Checkpoint/restore for device state

4. **Topology Manager** (`topologymanager/`):
   - Coordinates CPU, memory, device allocation
   - Ensures alignment-based allocation
   - Policies: none, best-effort, restricted, single-numa-node

5. **DRA (Dynamic Resource Allocation)** (`dra/`):
   - Manages dynamic resource allocation
   - Per-container resource requests

### 6. Volume Manager (`pkg/kubelet/volumemanager/`)

**Purpose:** Attach, mount, and manage pod volumes

**Key Components:**

1. **VolumeManager Interface** (`volume_manager.go:98`):
   - `Run()` - Start volume loops
   - `WaitForAttachAndMount()` - Block until volumes ready
   - `WaitForUnmount()` - Block until volumes unmounted
   - `GetMountedVolumesForPod()` - Query mounted volumes

2. **DesiredStateOfWorld Populator** (`populator/`):
   - Watches pods for volume requirements
   - Updates desired state

3. **ActualStateOfWorld Reconciler** (`reconciler/`):
   - Attaches/detaches volumes
   - Mounts/unmounts volumes
   - Cleanup orphaned volumes

4. **Volume Plugin Architecture**:
   - Supports in-tree (built-in) plugins
   - CSI (Container Storage Interface) plugins
   - Volume migration (in-tree to CSI)

### 7. Status Management (`pkg/kubelet/status/`)

**StatusManager** - Updates pod status in API

**Key Responsibilities:**
- Updates pod phase (Pending, Running, Succeeded, Failed)
- Updates container statuses
- Tracks pod ready condition
- Handles status conflicts during pod updates
- Periodic batch updates to reduce API load

### 8. Pod Probing (`pkg/kubelet/prober/`)

**ProbeManager** - Executes health checks

**Probe Types:**
- Liveness probe - Is container still running?
- Readiness probe - Is container ready for traffic?
- Startup probe - Has container finished starting?

**Probe Handlers:**
- ExecAction - Execute command
- HTTPGetAction - HTTP GET request
- TCPSocketAction - TCP port check

**Result Managers:**
- `livenessManager` - Liveness probe results
- `readinessManager` - Readiness probe results  
- `startupManager` - Startup probe results

### 9. Eviction Management (`pkg/kubelet/eviction/`)

**EvictionManager** - Responds to resource pressure

**Monitored Signals:**
- Memory usage
- Disk usage (filesystem, image)
- PID availability
- Custom metrics

**Eviction Thresholds:**
- Hard thresholds - Immediate action
- Soft thresholds with grace period

**Pod Selection for Eviction:**
- Evicts pods with lowest QoS first
- Within QoS class, evicts highest resource usage
- Respects pod disruption budgets

### 10. Node Status (`pkg/kubelet/kubelet_node_status.go`)

**Periodic Updates:**
- Node conditions (Ready, MemoryPressure, DiskPressure, PIDPressure)
- Node capacity and allocatable resources
- Images on node
- Running pods

**Update Frequency:**
- Every `nodeStatusUpdateFrequency` (10 seconds default)
- Or via node lease (more frequent in recent versions)
- Fast initial update after startup

## Initialization Sequence

### 1. Main Startup (`server.go:178-299`)
```
NewKubeletCommand()
  ├─> Parse flags and config
  ├─> Validate configuration
  ├─> Load kubelet config file (if provided)
  ├─> Merge dropin configs (if provided)
  ├─> Apply flag precedence
  └─> Return cobra.Command
```

### 2. Dependencies Creation (`server.go:270`)
```
UnsecuredDependencies()
  ├─> Create KubeClient connection
  ├─> Initialize cAdvisor
  ├─> Setup OOM adjuster
  ├─> Create event recorder
  ├─> Build volume plugins
  ├─> Create pod config from sources
  ├─> Setup authentication
  ├─> Initialize cloud provider
  └─> Return Dependencies struct
```

### 3. Kubelet Object Creation (`kubelet.go:416`)
```
NewMainKubelet()
  ├─> Setup pod manager
  ├─> Initialize PLEG (Generic and/or Evented)
  ├─> Create container runtime manager
  ├─> Setup resource managers (CPU, memory, device, topology)
  ├─> Initialize volume manager
  ├─> Create status manager
  ├─> Setup probe manager
  ├─> Initialize image manager
  ├─> Create eviction manager
  ├─> Setup certificate manager
  ├─> Create pod workers
  ├─> Initialize node status components
  └─> Return Kubelet instance
```

### 4. Runtime Service Init (`kubelet.go:394`)
```
PreInitRuntimeService()
  ├─> Connect to container runtime endpoint
  ├─> Connect to image service endpoint
  └─> Verify CRI connectivity
```

### 5. Run Phase (`kubelet.go:1743`)
```
Run(updates)
  ├─> Start PLEG.Start()
  ├─> Start EventedPLEG.Start() (if enabled)
  ├─> Start volume manager
  ├─> Start status manager
  ├─> Start eviction monitoring
  ├─> Start node status updates
  ├─> Start node lease renewal
  ├─> Start pod config sync loops
  └─> Enter syncLoop()
```

## Key Interfaces and Types

### Pod Configuration
- **PodConfig** (`config/pod_config.go`) - Aggregates pods from multiple sources
- **PodUpdate** (`types/types.go`) - Pod change event

### Container Interface
- **Runtime** - Container runtime implementation
- **Pod** - Group of containers
- **Container** - Individual container
- **PodStatus** - Container runtime pod status

### Manager Interfaces
- **ContainerManager** - Resource management
- **VolumeManager** - Volume operations
- **StatusManager** - Pod status updates
- **ProbeManager** - Health checking
- **EvictionManager** - Resource pressure response
- **ImageManager** - Image garbage collection

### State Tracking
- **RuntimeCache** - Caches pod/container info from runtime
- **PodCache** - Caches pod status from PLEG
- **PodManager** - Desired pods to run
- **PodWorkers** - Actually running pods

## Important Implementation Files

### Entry Points
- `/cmd/kubelet/kubelet.go` - Binary entry
- `/cmd/kubelet/app/server.go` - Server initialization (55KB)

### Core Kubelet
- `/pkg/kubelet/kubelet.go` - Main Kubelet struct (137KB)
- `/pkg/kubelet/kubelet_pods.go` - Pod sync logic (109KB)
- `/pkg/kubelet/pod_workers.go` - Pod worker implementation (77KB)

### Container Runtime
- `/pkg/kubelet/kuberuntime/kuberuntime_manager.go` - CRI manager (82KB)
- `/pkg/kubelet/container/runtime.go` - Runtime interface (30KB)

### Pod Lifecycle
- `/pkg/kubelet/pleg/generic.go` - GenericPLEG (22KB)
- `/pkg/kubelet/pleg/evented.go` - EventedPLEG (16KB)
- `/pkg/kubelet/pleg/pleg.go` - PLEG interfaces (1KB)

### Resource Management
- `/pkg/kubelet/cm/container_manager_linux.go` - Cgroup management (38KB)
- `/pkg/kubelet/cm/cpumanager/cpu_manager.go` - CPU allocation (19KB)
- `/pkg/kubelet/cm/devicemanager/manager.go` - Device management (49KB)

### Volume Management
- `/pkg/kubelet/volumemanager/volume_manager.go` - Volume manager (24KB)
- `/pkg/kubelet/volumemanager/reconciler/` - Volume reconciliation
- `/pkg/kubelet/volumemanager/populator/` - Desired state updates

### Status & Monitoring
- `/pkg/kubelet/status/status_manager.go` - Pod status (53KB)
- `/pkg/kubelet/kubelet_node_status.go` - Node status (30KB)
- `/pkg/kubelet/prober/prober_manager.go` - Health probes (11KB)
- `/pkg/kubelet/eviction/eviction_manager.go` - Eviction policy (25KB)

## Data Flow

### Pod Addition Flow
```
Config Source (API/file/HTTP)
  ↓
PodConfig.Channel()
  ↓
syncLoop() - configCh
  ↓
HandlePodAdditions()
  ↓
podWorkers.UpdatePod() - SyncPod state
  ↓
syncPod() - create/update containers
  ↓
containerRuntime.SyncPod()
  ↓
statusManager.SetPodStatus()
  ↓
API Server update
```

### Pod Deletion Flow
```
Config Source removal
  ↓
syncLoop() - configCh
  ↓
HandlePodRemoves()
  ↓
podWorkers.UpdatePod() - TerminatingPod state
  ↓
syncTerminatingPod() - graceful termination
  ↓
podWorkers.UpdatePod() - TerminatedPod state
  ↓
syncTerminatedPod() - cleanup resources
  ↓
PLEG cleanup/status update
```

### Container Failure Recovery
```
Runtime Container Exit
  ↓
PLEG.Relist() detects change
  ↓
PLEG emits ContainerDied event
  ↓
syncLoop() - plegCh
  ↓
handler.HandlePodSyncs()
  ↓
podWorkers.UpdatePod() - SyncPod state
  ↓
syncPod() - restart container
```

### Resource Pressure Response
```
EvictionManager monitoring loop
  ↓
Detects resource threshold exceeded
  ↓
Selects pods for eviction
  ↓
podWorkers.UpdatePod() - TerminatingPod state
  ↓
Pod containers terminate
  ↓
Resources freed
```

## Concurrency and Synchronization

### Goroutine Architecture
- **1 per pod worker** - Handles pod lifecycle
- **1 PLEG loop** - Detects container changes
- **1 sync loop** - Main event dispatcher
- **Multiple manager loops** - Volume, status, eviction, etc.
- **1 per node status update** - Periodic updates
- **1 per probe** - Health check execution

### Synchronization Mechanisms
- **Channel-based communication** - Between sync loop and components
- **Mutex locks** - On pod manager, runtime state
- **FIFO work queue** - Per-pod work ordering
- **Backoff** - For failed pod syncs
- **Wait.Until/wait.JitterUntil** - Periodic loops

### State Consistency
- **Pod manager** - Desired state (authoritative for admission)
- **Pod workers** - Actual running state (authoritative for runtime)
- **Pod cache** - Aggregates runtime pod status
- **Status manager** - Synthesized status for API

## Important Patterns

### 1. Desired vs Actual State
- Desired: Pod objects in API and pod manager
- Actual: Containers running in container runtime
- Kubelet continuously reconciles them

### 2. Three-Phase Pod Lifecycle
1. **Sync** - Containers should run (syncPod)
2. **Terminating** - Containers being stopped (syncTerminatingPod)
3. **Terminated** - Cleanup complete (syncTerminatedPod)

### 3. Admission and Resource Checking
- PLEG and pod sync check if pod can run
- Container manager enforces resource limits
- Eviction manager reclaims resources as needed

### 4. Event-Driven with Periodic Fallback
- PLEG detects container changes
- Periodic sync catches missed events
- Housekeeping performs background cleanup

### 5. Plugin Architecture
- Container runtime pluggable (via CRI)
- Volume plugins extensible
- Device plugins discoverable
- Probe handlers customizable

## Performance Considerations

### Optimization Points
- **PLEG relisting** - Tunable period vs detection latency
- **Evented PLEG** - Lower CPU for rapid detection
- **Status batch updates** - Reduce API load
- **Pod worker concurrency** - Parallel pod syncs
- **Volume operation executor** - Limits concurrent volume ops
- **Device plugin** - Batches device allocation

### Resource Limits
- **Max pods per node** - Configurable (default: 110)
- **Max parallel image pulls** - Configurable
- **Container GC period** - Every 1 minute
- **Image GC period** - Every 5 minutes
- **Node status update frequency** - Every 10 seconds

## Summary

The kubelet is a sophisticated, event-driven pod orchestrator that:
1. Maintains desired pod state via pod workers
2. Detects container changes via PLEG
3. Enforces resource policies via container manager
4. Manages volumes and networking
5. Reports status and responds to failures
6. Handles graceful termination and resource pressure

Its architecture emphasizes reliability through periodic reconciliation,
flexibility through pluggable components, and efficiency through
event-driven updates with fallback mechanisms.
