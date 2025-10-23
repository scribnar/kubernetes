# kubelet Code Reference: Core Managers

## Table of Contents
1. [Overview](#overview)
2. [PLEG (Pod Lifecycle Event Generator)](#pleg-pod-lifecycle-event-generator)
3. [Pod Manager](#pod-manager)
4. [Status Manager](#status-manager)
5. [Probe Manager](#probe-manager)
6. [Volume Manager](#volume-manager)
7. [Image Manager](#image-manager)
8. [Container Manager](#container-manager)
9. [Eviction Manager](#eviction-manager)
10. [Pod Workers](#pod-workers)
11. [Secret and ConfigMap Managers](#secret-and-configmap-managers)
12. [Quick Reference Matrix](#quick-reference-matrix)

## Overview

This document provides a comprehensive reference to the core manager implementations in kubelet, including their key files, interfaces, and important functions.

### Manager Organization

```
pkg/kubelet/
├── pleg/                    # Pod Lifecycle Event Generator
│   ├── generic.go          # Generic PLEG implementation
│   └── evented.go          # Event-driven PLEG
├── pod/                     # Pod Manager
│   ├── pod_manager.go      # Pod tracking
│   └── mirror_client.go    # Mirror pod management
├── status/                  # Status Manager
│   ├── status_manager.go   # Status synchronization
│   └── generate.go         # Status generation
├── prober/                  # Probe Manager
│   ├── prober_manager.go   # Probe coordination
│   └── worker.go           # Probe execution
├── volumemanager/          # Volume Manager
│   ├── volume_manager.go   # Volume lifecycle
│   └── reconciler/         # Volume reconciliation
├── images/                  # Image Manager
│   ├── image_manager.go    # Image lifecycle
│   └── image_gc.go         # Garbage collection
├── cm/                      # Container Manager
│   ├── container_manager.go # Resource management
│   └── cgroup_manager/     # Cgroup management
├── eviction/               # Eviction Manager
│   ├── eviction_manager.go # Pod eviction
│   └── threshold.go        # Eviction thresholds
└── pod_workers.go          # Pod worker goroutines
```

## PLEG (Pod Lifecycle Event Generator)

### Generic PLEG Implementation
**File**: `pkg/kubelet/pleg/generic.go`

#### Key Structures

```go
// Line 58
type GenericPLEG struct {
    // Relist period
    relistPeriod time.Duration

    // Container runtime
    runtime kubecontainer.Runtime

    // Channels
    eventChannel chan *PodLifecycleEvent

    // Pod cache
    podRecords podRecords

    // Mutex for thread safety
    mutex sync.Mutex

    // Last relist time
    relistTime atomic.Value
}

// Line 35
type PodLifecycleEvent struct {
    // Pod ID
    ID types.UID

    // Event type
    Type PodLifecycleEventType

    // Event data
    Data interface{}
}
```

#### Core Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `NewGenericPLEG()` | `generic.go:80` | Creates PLEG instance |
| `Start()` | `generic.go:125` | Starts PLEG goroutine |
| `Watch()` | `generic.go:140` | Returns event channel |
| `Relist()` | `generic.go:190` | Main relist logic |
| `computeEvents()` | `generic.go:350` | Compute state changes |
| `updateCache()` | `generic.go:420` | Update pod cache |
| `Healthy()` | `generic.go:520` | Health check |

#### Relist Implementation

```go
// Line 190
func (g *GenericPLEG) Relist() {
    g.mutex.Lock()
    defer g.mutex.Unlock()

    // Get current time
    timestamp := g.clock.Now()

    // Get all pods/containers from runtime
    podList, err := g.runtime.GetPods(true)
    if err != nil {
        return
    }

    // Convert to internal format
    pods := kubecontainer.ConvertPodStatusToAPIPodStatus(podList)

    // Update pod records
    g.podRecords.setCurrent(pods)

    // Compute events
    for pid := range g.podRecords {
        oldPod := g.podRecords.getOld(pid)
        pod := g.podRecords.getCurrent(pid)

        // Compare and generate events
        events := computeEvents(oldPod, pod)

        for _, e := range events {
            select {
            case g.eventChannel <- e:
            default:
                // Channel full, drop event
            }
        }
    }

    // Move current to old
    g.podRecords.update()

    // Update relist time
    g.relistTime.Store(timestamp)
}
```

### Evented PLEG
**File**: `pkg/kubelet/pleg/evented.go`

```go
// Line 45
type EventedPLEG struct {
    // Generic PLEG for fallback
    generic *GenericPLEG

    // Runtime event channel
    runtimeEventChannel chan *runtimeapi.ContainerEventResponse

    // Container state cache
    cache kubecontainer.Cache
}

// Line 80 - Start watching runtime events
func (e *EventedPLEG) Start() {
    e.generic.Start()
    go e.watchRuntimeEvents()
}

// Line 120 - Process runtime events
func (e *EventedPLEG) processRuntimeEvent(event *runtimeapi.ContainerEventResponse) {
    // Convert runtime event to PLEG event
    plegEvent := &PodLifecycleEvent{
        ID:   types.UID(event.PodSandboxStatus.Metadata.Uid),
        Type: eventTypeFromRuntimeEvent(event.EventType),
        Data: event,
    }

    // Send to channel
    e.eventChannel <- plegEvent
}
```

## Pod Manager

**File**: `pkg/kubelet/pod/pod_manager.go`

### Interface Definition

```go
// Line 48
type Manager interface {
    // Pod operations
    GetPods() []*v1.Pod
    GetPodByFullName(string) (*v1.Pod, bool)
    GetPodByName(namespace, name string) (*v1.Pod, bool)
    GetPodByUID(types.UID) (*v1.Pod, bool)

    // Pod management
    AddPod(pod *v1.Pod)
    UpdatePod(pod *v1.Pod)
    DeletePod(pod *v1.Pod)

    // Mirror pod operations
    GetMirrorPodByPod(*v1.Pod) (*v1.Pod, bool)
    GetPodByMirrorPod(*v1.Pod) (*v1.Pod, bool)

    // Static pod operations
    GetStaticPods() []*v1.Pod
}
```

### Basic Manager Implementation

```go
// Line 90
type basicManager struct {
    // Thread safety
    lock sync.RWMutex

    // All pods indexed by UID
    podByUID map[types.UID]*v1.Pod

    // Pods indexed by full name
    podByFullName map[string]*v1.Pod

    // Mirror pod mappings
    mirrorPodByUID map[types.UID]*v1.Pod
    mirrorPodByFullName map[string]*v1.Pod

    // Translations between static and mirror pods
    translationByUID map[types.UID]types.UID
}

// Line 125
func (pm *basicManager) AddPod(pod *v1.Pod) {
    pm.lock.Lock()
    defer pm.lock.Unlock()

    pm.podByUID[pod.UID] = pod
    pm.podByFullName[kubecontainer.GetPodFullName(pod)] = pod

    if kubetypes.IsMirrorPod(pod) {
        pm.mirrorPodByUID[pod.UID] = pod
        pm.mirrorPodByFullName[kubecontainer.GetPodFullName(pod)] = pod
    }
}

// Line 180
func (pm *basicManager) GetPodByUID(uid types.UID) (*v1.Pod, bool) {
    pm.lock.RLock()
    defer pm.lock.RUnlock()

    pod, ok := pm.podByUID[uid]
    return pod, ok
}
```

### Mirror Client
**File**: `pkg/kubelet/pod/mirror_client.go`

```go
// Line 55
type basicMirrorClient struct {
    apiserverClient clientset.Interface
    nodeGetter      nodeGetter
    nodeName        string
}

// Line 70
func (mc *basicMirrorClient) CreateMirrorPod(ctx context.Context, pod *v1.Pod) error {
    // Copy pod and add mirror annotation
    copyPod := *pod
    copyPod.Annotations[kubetypes.ConfigMirrorAnnotationKey] = getPodHash(pod)

    // Add node owner reference
    nodeUID, _ := mc.getNodeUID()
    copyPod.OwnerReferences = []metav1.OwnerReference{{
        APIVersion: v1.SchemeGroupVersion.String(),
        Kind:       "Node",
        Name:       mc.nodeName,
        UID:        nodeUID,
    }}

    // Create in API server
    _, err := mc.apiserverClient.CoreV1().Pods(copyPod.Namespace).Create(ctx, &copyPod, metav1.CreateOptions{})
    return err
}

// Line 110
func (mc *basicMirrorClient) DeleteMirrorPod(ctx context.Context, podFullName string, uid *types.UID) (bool, error) {
    name, namespace, err := kubecontainer.ParsePodFullName(podFullName)

    err = mc.apiserverClient.CoreV1().Pods(namespace).Delete(ctx, name, metav1.DeleteOptions{
        Preconditions: &metav1.Preconditions{UID: uid},
    })

    return err == nil, err
}
```

## Status Manager

**File**: `pkg/kubelet/status/status_manager.go`

### Manager Structure

```go
// Line 85
type manager struct {
    kubeClient clientset.Interface
    podManager pod.Manager

    // Pod status cache
    podStatuses      map[types.UID]versionedPodStatus
    podStatusesLock  sync.RWMutex
    podStatusChannel chan struct{}

    // Deletion tracking
    podDeletionSafety PodDeletionSafetyProvider
}

// Line 45
type versionedPodStatus struct {
    status         v1.PodStatus
    version        uint64
    podName        string
    podNamespace   string
    podIsTerminal  bool
}
```

### Key Functions

```go
// Line 165
func NewManager(kubeClient clientset.Interface, podManager pod.Manager) Manager {
    return &manager{
        kubeClient:       kubeClient,
        podManager:       podManager,
        podStatuses:      make(map[types.UID]versionedPodStatus),
        podStatusChannel: make(chan struct{}, 1),
    }
}

// Line 200
func (m *manager) Start() {
    go wait.Forever(func() {
        m.syncBatch()
    }, syncPeriod)
}

// Line 250
func (m *manager) SetPodStatus(pod *v1.Pod, status v1.PodStatus) {
    m.podStatusesLock.Lock()
    defer m.podStatusesLock.Unlock()

    // Check if update needed
    cached, ok := m.podStatuses[pod.UID]
    if ok && cached.status.Equal(&status) {
        return
    }

    // Update with new version
    m.podStatuses[pod.UID] = versionedPodStatus{
        status:        status,
        version:       cached.version + 1,
        podName:       pod.Name,
        podNamespace:  pod.Namespace,
        podIsTerminal: podIsTerminal(&status),
    }

    // Trigger sync
    select {
    case m.podStatusChannel <- struct{}{}:
    default:
    }
}

// Line 350
func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {
    // Get pod from pod manager
    pod, err := m.podManager.GetPodByUID(uid)
    if err != nil {
        return
    }

    // Update API server
    _, err = m.kubeClient.CoreV1().Pods(pod.Namespace).UpdateStatus(
        context.TODO(),
        pod,
        metav1.UpdateOptions{},
    )

    if err != nil {
        return
    }

    // Mark as synced
    m.podStatusesLock.Lock()
    m.podStatuses[uid] = status
    m.podStatusesLock.Unlock()
}
```

### Status Generation
**File**: `pkg/kubelet/status/generate.go`

```go
// Line 85
func GeneratePodReadyCondition(spec *v1.PodSpec, conditions []v1.PodCondition,
    containerStatuses []v1.ContainerStatus, podPhase v1.PodPhase) v1.PodCondition {

    // Check containers ready
    containersReady := true
    for _, container := range containerStatuses {
        if !container.Ready {
            containersReady = false
            break
        }
    }

    if !containersReady {
        return v1.PodCondition{
            Type:   v1.PodReady,
            Status: v1.ConditionFalse,
            Reason: "ContainersNotReady",
        }
    }

    // Check readiness gates
    for _, gate := range spec.ReadinessGates {
        conditionFound := false
        for _, condition := range conditions {
            if condition.Type == gate.ConditionType {
                conditionFound = true
                if condition.Status != v1.ConditionTrue {
                    return v1.PodCondition{
                        Type:   v1.PodReady,
                        Status: v1.ConditionFalse,
                        Reason: "ReadinessGateNotReady",
                    }
                }
            }
        }

        if !conditionFound {
            return v1.PodCondition{
                Type:   v1.PodReady,
                Status: v1.ConditionFalse,
                Reason: "ReadinessGateNotReady",
            }
        }
    }

    return v1.PodCondition{
        Type:   v1.PodReady,
        Status: v1.ConditionTrue,
    }
}
```

## Probe Manager

**File**: `pkg/kubelet/prober/prober_manager.go`

### Manager Structure

```go
// Line 60
type manager struct {
    // Worker management
    workers map[probeKey]*worker
    lock    sync.RWMutex

    // Status manager
    statusManager status.Manager

    // Prober implementation
    prober *prober

    // Result managers
    readinessManager  results.Manager
    livenessManager   results.Manager
    startupManager    results.Manager

    // Event recorder
    recorder record.EventRecorder
}

// Line 45
type probeKey struct {
    podUID        types.UID
    containerName string
    probeType     probeType
}
```

### Worker Implementation
**File**: `pkg/kubelet/prober/worker.go`

```go
// Line 85
type worker struct {
    // Container info
    pod           *v1.Pod
    container     v1.Container
    containerID   kubecontainer.ContainerID

    // Probe spec
    spec          *v1.Probe
    probeType     probeType

    // Probe execution
    probeManager  *manager
    lastResult    results.Result
    resultChannel chan results.Result

    // Backoff for failures
    backoff *backoff.Backoff
}

// Line 150
func (w *worker) run() {
    probeTickerPeriod := time.Duration(w.spec.PeriodSeconds) * time.Second
    probeTicker := time.NewTicker(probeTickerPeriod)
    defer probeTicker.Stop()

    for {
        select {
        case <-probeTicker.C:
            w.doProbe()
        case <-w.stopChannel:
            return
        }
    }
}

// Line 200
func (w *worker) doProbe() {
    // Execute probe
    result, err := w.probeManager.prober.probe(
        w.probeType,
        w.pod,
        w.pod.Status,
        w.container,
        w.containerID,
    )

    // Process result
    if err != nil {
        // Probe error
        result = results.Failure
    }

    // Update result
    if w.lastResult != result {
        w.probeManager.recordContainerEvent(w.pod, &w.container, result)
    }

    w.lastResult = result

    // Update manager
    switch w.probeType {
    case readiness:
        w.probeManager.readinessManager.Set(w.containerID, result, w.pod)
    case liveness:
        w.probeManager.livenessManager.Set(w.containerID, result, w.pod)
    case startup:
        w.probeManager.startupManager.Set(w.containerID, result, w.pod)
    }
}
```

## Volume Manager

**File**: `pkg/kubelet/volumemanager/volume_manager.go`

### Manager Structure

```go
// Line 120
type volumeManager struct {
    // Core components
    kubeClient          clientset.Interface
    volumePluginManager *volume.VolumePluginMgr

    // Desired state of world
    desiredStateOfWorld cache.DesiredStateOfWorld

    // Actual state of world
    actualStateOfWorld cache.ActualStateOfWorld

    // Reconciler
    reconciler reconciler.Reconciler

    // Operation executor
    operationExecutor operationexecutor.OperationExecutor
}
```

### Core Functions

```go
// Line 250
func NewVolumeManager(
    controllerAttachDetachEnabled bool,
    nodeName types.NodeName,
    podManager pod.Manager,
    statusManager status.Manager,
    kubeClient clientset.Interface,
    volumePluginMgr *volume.VolumePluginMgr,
    kubeContainerRuntime kubecontainer.Runtime,
    mounter mount.Interface,
    kubeletPodsDir string,
    recorder record.EventRecorder,
    checkNodeCapabilitiesBeforeMount bool,
    keepTerminatedPodVolumes bool,
) VolumeManager {

    vm := &volumeManager{
        kubeClient:          kubeClient,
        volumePluginManager: volumePluginMgr,
        desiredStateOfWorld: cache.NewDesiredStateOfWorld(volumePluginMgr),
        actualStateOfWorld:  cache.NewActualStateOfWorld(nodeName, volumePluginMgr),
        operationExecutor:   operationexecutor.NewOperationExecutor(operationexecutor.NewOperationGenerator(
            kubeClient,
            volumePluginMgr,
            recorder,
            checkNodeCapabilitiesBeforeMount,
        )),
    }

    vm.reconciler = reconciler.NewReconciler(
        kubeClient,
        controllerAttachDetachEnabled,
        reconcilerLoopSleepPeriod,
        reconcilerMaxWaitForUnmountDuration,
        vm.desiredStateOfWorld,
        vm.actualStateOfWorld,
        vm.operationExecutor,
        mounter,
        volumePluginMgr,
        kubeletPodsDir,
    )

    return vm
}

// Line 340
func (vm *volumeManager) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
    go vm.desiredStateOfWorldPopulator.Run(sourcesReady, stopCh)
    go vm.reconciler.Run(stopCh)

    <-stopCh

    vm.desiredStateOfWorldPopulator.Stop()
    vm.reconciler.StopReconciling()
}
```

### Reconciler
**File**: `pkg/kubelet/volumemanager/reconciler/reconciler.go`

```go
// Line 150
func (rc *reconciler) reconcile() {
    // Unmount volumes no longer needed
    for _, volumeToUnmount := range rc.actualStateOfWorld.GetMountedVolumesForPod() {
        if !rc.desiredStateOfWorld.VolumeExists(volumeToUnmount.VolumeName) {
            rc.operationExecutor.UnmountVolume(volumeToUnmount, rc.actualStateOfWorld)
        }
    }

    // Mount required volumes
    for _, volumeToMount := range rc.desiredStateOfWorld.GetVolumesToMount() {
        if rc.actualStateOfWorld.VolumeIsMounted(volumeToMount.VolumeName) {
            continue
        }

        rc.operationExecutor.MountVolume(
            volumeToMount,
            rc.actualStateOfWorld,
            rc.isReady,
        )
    }

    // Verify volumes are attached (if needed)
    for _, volumeToVerify := range rc.actualStateOfWorld.GetGloballyMountedVolumes() {
        rc.operationExecutor.VerifyControllerAttachedVolume(
            volumeToVerify,
            rc.actualStateOfWorld,
        )
    }
}
```

## Image Manager

**File**: `pkg/kubelet/images/image_manager.go`

### Manager Structure

```go
// Line 80
type imageManager struct {
    // Runtime integration
    imageService kubecontainer.ImageService

    // Image GC
    imageGC *imageGC

    // Image pull
    imagePuller imagePuller
}

// Line 110
func NewImageManager(
    imageService kubecontainer.ImageService,
    imageGCPolicy ImageGCPolicy,
    recorder record.EventRecorder,
    nodeRef *v1.ObjectReference,
    imageBackOff *backoff.Backoff,
    serialized bool,
    qps float32,
    burst int,
) ImageManager {

    im := &imageManager{
        imageService: imageService,
    }

    im.imageGC = newImageGC(
        imageService,
        imageGCPolicy,
        recorder,
        nodeRef,
    )

    im.imagePuller = newSerialImagePuller(
        imageService,
        imageBackOff,
        qps,
        burst,
    )

    return im
}
```

### Image GC
**File**: `pkg/kubelet/images/image_gc.go`

```go
// Line 180
func (im *imageGC) GarbageCollect() error {
    // Get disk usage
    diskUsage, err := im.imageService.ImageFsInfo()
    if err != nil {
        return err
    }

    // Check if GC needed
    usagePercent := 100.0 * float64(diskUsage.Used) / float64(diskUsage.Capacity)
    if usagePercent < im.policy.HighThresholdPercent {
        return nil
    }

    // Get all images
    images, err := im.imageService.ListImages()
    if err != nil {
        return err
    }

    // Sort by last used time
    sort.Sort(byLastUsedAndDetected(images))

    // Delete until below low threshold
    for _, image := range images {
        if usagePercent < im.policy.LowThresholdPercent {
            break
        }

        err = im.imageService.RemoveImage(image.ID)
        if err == nil {
            diskUsage, _ = im.imageService.ImageFsInfo()
            usagePercent = 100.0 * float64(diskUsage.Used) / float64(diskUsage.Capacity)
        }
    }

    return nil
}
```

## Container Manager

**File**: `pkg/kubelet/cm/container_manager.go`

### Interface

```go
// Line 65
type ContainerManager interface {
    // Start container manager
    Start(node *v1.Node,
        activePods ActivePodsFunc,
        sourcesReady config.SourcesReady,
        podStatusProvider status.PodStatusProvider,
        runtimeService internalapi.RuntimeService,
        initialContainers containermap.ContainerMap) error

    // Resource management
    SystemCgroupsLimit() v1.ResourceList
    GetNodeAllocatableReservation() v1.ResourceList
    GetCapacity(localStorageCapacityIsolation bool) v1.ResourceList
    GetNodeAllocatableAbsolute() v1.ResourceList

    // Device management
    GetDevicePluginResourceCapacity() (v1.ResourceList, v1.ResourceList, []string)

    // QoS management
    UpdateQOSCgroups() error

    // CPU/Memory/Device managers
    GetCPUManager() cpumanager.Manager
    GetMemoryManager() memorymanager.Manager
    GetDeviceManager() devicemanager.Manager
    GetTopologyManager() topologymanager.Manager

    // Pod management
    GetPodCgroupRoot() string
    GetPluginRegistrationHandler() pluginregistration.PluginHandler
    GetPodContainerName(*v1.Pod, *v1.Container) (CgroupName, string)

    // Container operations
    NewPodContainerManager() PodContainerManager
    InternalContainerLifecycle() InternalContainerLifecycle
}
```

### Linux Implementation
**File**: `pkg/kubelet/cm/container_manager_linux.go`

```go
// Line 180
type containerManagerImpl struct {
    sync.RWMutex

    // Node info
    machineInfo cadvisorapi.MachineInfo
    capacity    v1.ResourceList

    // Node allocatable
    nodeAllocatableReservation v1.ResourceList

    // External Dependencies
    cadvisorInterface cadvisor.Interface
    cgroupManager     CgroupManager

    // Managers
    qosManager      *qosContainerManager
    cpuManager      cpumanager.Manager
    memoryManager   memorymanager.Manager
    deviceManager   devicemanager.Manager
    topologyManager topologymanager.Manager
}

// Line 350
func (cm *containerManagerImpl) Start(node *v1.Node,
    activePods ActivePodsFunc,
    sourcesReady config.SourcesReady,
    podStatusProvider status.PodStatusProvider,
    runtimeService internalapi.RuntimeService,
    initialContainers containermap.ContainerMap) error {

    // Setup cgroups
    if err := cm.setupNode(activePods); err != nil {
        return err
    }

    // Start QoS manager
    if err := cm.qosManager.Start(cm.GetNodeAllocatableAbsolute, activePods); err != nil {
        return err
    }

    // Start CPU manager
    if cm.cpuManager != nil {
        cm.cpuManager.Start(activePods, sourcesReady, podStatusProvider, runtimeService, initialContainers)
    }

    // Start memory manager
    if cm.memoryManager != nil {
        cm.memoryManager.Start(activePods, sourcesReady, podStatusProvider, runtimeService, initialContainers)
    }

    // Start device manager
    if cm.deviceManager != nil {
        cm.deviceManager.Start()
    }

    return nil
}
```

## Eviction Manager

**File**: `pkg/kubelet/eviction/eviction_manager.go`

### Manager Structure

```go
// Line 85
type managerImpl struct {
    // Dependencies
    client        clientset.Interface
    recorder      record.EventRecorder
    imageGC       ImageGC
    containerGC   ContainerGC
    killPodFunc   KillPodFunc

    // Configuration
    config Config

    // Thresholds
    thresholds []evictionapi.Threshold

    // State
    nodeConditions        []v1.NodeConditionType
    nodeConditionsLastObservedAt nodeConditionsObservedAt

    // Notifiers
    notifier          Notifier

    // Stats
    statsFunc StatsFunc

    // Sync state
    lastObservations signalObservations
    signalToRankFunc map[evictionapi.Signal]rankFunc
    signalToNodeCondition map[evictionapi.Signal]v1.NodeConditionType
}
```

### Core Functions

```go
// Line 120
func NewManager(
    summaryProvider stats.SummaryProvider,
    config Config,
    killPodFunc KillPodFunc,
    imageGC ImageGC,
    containerGC ContainerGC,
    recorder record.EventRecorder,
    nodeRef *v1.ObjectReference,
    clock clock.Clock,
) Manager {

    manager := &managerImpl{
        config:      config,
        killPodFunc: killPodFunc,
        imageGC:     imageGC,
        containerGC: containerGC,
        recorder:    recorder,
        clock:       clock,
    }

    manager.parseThresholds()

    return manager
}

// Line 250
func (m *managerImpl) synchronize(diskInfoProvider DiskInfoProvider, podFunc ActivePodsFunc) []*v1.Pod {
    // Get observations
    observations, statsFunc := makeSignalObservations(m.summaryProvider)

    // Get thresholds
    thresholds := m.thresholds

    // Track resources under pressure
    signalToResource := map[evictionapi.Signal]v1.ResourceName{}
    signalToResource[evictionapi.SignalMemoryAvailable] = v1.ResourceMemory
    signalToResource[evictionapi.SignalNodeFsAvailable] = v1.ResourceEphemeralStorage

    // Check thresholds
    thresholds = thresholdsMet(thresholds, observations)

    // Track eviction signals
    activeSignals := []evictionapi.Signal{}
    for _, threshold := range thresholds {
        activeSignals = append(activeSignals, threshold.Signal)
    }

    // Get pod list
    activePods := podFunc()

    // Rank pods for eviction
    rank := m.rankPods(activePods, activeSignals)

    // Evict pods
    for _, pod := range rank {
        if m.evictPod(pod, activeSignals[0]) {
            return []*v1.Pod{pod}
        }
    }

    return nil
}
```

## Pod Workers

**File**: `pkg/kubelet/pod_workers.go`

### Pod Workers Structure

```go
// Line 150
type podWorkers struct {
    // Synchronization functions
    syncPodFn            syncPodFnType
    syncTerminatingPodFn syncTerminatingPodFnType
    syncTerminatedPodFn  syncTerminatedPodFnType

    // Pod state tracking
    podSyncStatuses map[types.UID]*podSyncStatus

    // Worker goroutines
    podWorkerChannels map[types.UID]chan struct{}

    // Synchronization
    podLock sync.Mutex

    // Work queue
    queue workqueue.RateLimitingInterface
}

// Line 120
type podSyncStatus struct {
    // Current state
    syncState syncPodState

    // Pending updates
    pendingUpdate *UpdatePodOptions

    // Timestamps
    startedAt time.Time
    finishedAt time.Time

    // Termination info
    terminatingAt time.Time
    terminatedAt time.Time
    gracePeriod int64
}
```

### Core Functions

```go
// Line 350
func (p *podWorkers) UpdatePod(options UpdatePodOptions) {
    p.podLock.Lock()
    defer p.podLock.Unlock()

    uid := options.Pod.UID

    // Get or create sync status
    status, exists := p.podSyncStatuses[uid]
    if !exists {
        status = &podSyncStatus{
            syncState: syncPod,
        }
        p.podSyncStatuses[uid] = status

        // Create worker channel
        p.podWorkerChannels[uid] = make(chan struct{}, 1)

        // Start worker goroutine
        go p.managePodLoop(uid)
    }

    // Update pending work
    status.pendingUpdate = &options

    // Signal worker
    select {
    case p.podWorkerChannels[uid] <- struct{}{}:
    default:
    }
}

// Line 450
func (p *podWorkers) managePodLoop(uid types.UID) {
    for {
        p.podLock.Lock()
        status := p.podSyncStatuses[uid]

        if status == nil {
            p.podLock.Unlock()
            return
        }

        update := status.pendingUpdate
        status.pendingUpdate = nil
        p.podLock.Unlock()

        if update == nil {
            // Wait for work
            <-p.podWorkerChannels[uid]
            continue
        }

        // Process update based on state
        switch status.syncState {
        case syncPod:
            p.syncPodFn(update.Pod, update.MirrorPod, update.UpdateType)

        case syncTerminatingPod:
            p.syncTerminatingPodFn(update.Pod, status)

        case syncTerminatedPod:
            p.syncTerminatedPodFn(update.Pod, status)
        }

        // Check for state transition
        p.updatePodSyncStatus(uid, status)
    }
}
```

## Secret and ConfigMap Managers

### Secret Manager
**File**: `pkg/kubelet/secret/secret_manager.go`

```go
// Line 55
type Manager interface {
    // Get secret by namespace and name
    GetSecret(namespace, name string) (*v1.Secret, error)

    // Register pod to enable caching
    RegisterPod(pod *v1.Pod)

    // Unregister pod
    UnregisterPod(pod *v1.Pod)
}

// Line 85
type secretManager struct {
    kubeClient clientset.Interface

    // Object cache
    objectCache *objectCache
}

// Line 120
func (s *secretManager) GetSecret(namespace, name string) (*v1.Secret, error) {
    // Check cache first
    if secret := s.objectCache.Get(namespace, name); secret != nil {
        return secret.(*v1.Secret), nil
    }

    // Fetch from API server
    secret, err := s.kubeClient.CoreV1().Secrets(namespace).Get(
        context.TODO(),
        name,
        metav1.GetOptions{},
    )

    if err != nil {
        return nil, err
    }

    // Update cache
    s.objectCache.Add(namespace, name, secret)

    return secret, nil
}
```

### ConfigMap Manager
**File**: `pkg/kubelet/configmap/configmap_manager.go`

```go
// Line 55
type Manager interface {
    // Get configmap by namespace and name
    GetConfigMap(namespace, name string) (*v1.ConfigMap, error)

    // Register pod to enable caching
    RegisterPod(pod *v1.Pod)

    // Unregister pod
    UnregisterPod(pod *v1.Pod)
}

// Line 85
type configMapManager struct {
    kubeClient clientset.Interface

    // Object cache
    objectCache *objectCache
}

// Line 120
func (c *configMapManager) GetConfigMap(namespace, name string) (*v1.ConfigMap, error) {
    // Check cache first
    if cm := c.objectCache.Get(namespace, name); cm != nil {
        return cm.(*v1.ConfigMap), nil
    }

    // Fetch from API server
    cm, err := c.kubeClient.CoreV1().ConfigMaps(namespace).Get(
        context.TODO(),
        name,
        metav1.GetOptions{},
    )

    if err != nil {
        return nil, err
    }

    // Update cache
    c.objectCache.Add(namespace, name, cm)

    return cm, nil
}
```

## Quick Reference Matrix

### Manager Initialization Order

| Order | Manager | Creation Function | File | Line |
|-------|---------|------------------|------|------|
| 1 | Pod Manager | `NewBasicPodManager()` | `pkg/kubelet/pod/pod_manager.go` | 110 |
| 2 | Mirror Client | `NewBasicMirrorClient()` | `pkg/kubelet/pod/mirror_client.go` | 60 |
| 3 | Secret Manager | `NewWatchingSecretManager()` | `pkg/kubelet/secret/secret_manager.go` | 150 |
| 4 | ConfigMap Manager | `NewWatchingConfigMapManager()` | `pkg/kubelet/configmap/configmap_manager.go` | 150 |
| 5 | Status Manager | `NewManager()` | `pkg/kubelet/status/status_manager.go` | 165 |
| 6 | PLEG | `NewGenericPLEG()` | `pkg/kubelet/pleg/generic.go` | 80 |
| 7 | Container Manager | `NewContainerManager()` | `pkg/kubelet/cm/container_manager_linux.go` | 320 |
| 8 | Probe Manager | `NewManager()` | `pkg/kubelet/prober/prober_manager.go` | 90 |
| 9 | Volume Manager | `NewVolumeManager()` | `pkg/kubelet/volumemanager/volume_manager.go` | 250 |
| 10 | Image Manager | `NewImageManager()` | `pkg/kubelet/images/image_manager.go` | 110 |
| 11 | Eviction Manager | `NewManager()` | `pkg/kubelet/eviction/eviction_manager.go` | 120 |
| 12 | Pod Workers | `newPodWorkers()` | `pkg/kubelet/pod_workers.go` | 280 |

### Manager Start Functions

| Manager | Start Function | File | Line |
|---------|---------------|------|------|
| PLEG | `Start()` | `pkg/kubelet/pleg/generic.go` | 125 |
| Status Manager | `Start()` | `pkg/kubelet/status/status_manager.go` | 200 |
| Probe Manager | `Start()` | `pkg/kubelet/prober/prober_manager.go` | 140 |
| Volume Manager | `Run()` | `pkg/kubelet/volumemanager/volume_manager.go` | 340 |
| Image Manager | `Start()` | `pkg/kubelet/images/image_manager.go` | 180 |
| Container Manager | `Start()` | `pkg/kubelet/cm/container_manager_linux.go` | 350 |
| Eviction Manager | `Start()` | `pkg/kubelet/eviction/eviction_manager.go` | 180 |

### Key Manager Interfaces

| Manager | Interface Location | Implementation Location |
|---------|-------------------|------------------------|
| Pod Manager | `pkg/kubelet/pod/pod_manager.go:48` | `pkg/kubelet/pod/pod_manager.go:90` |
| Status Manager | `pkg/kubelet/status/status_manager.go:60` | `pkg/kubelet/status/status_manager.go:85` |
| Probe Manager | `pkg/kubelet/prober/prober_manager.go:45` | `pkg/kubelet/prober/prober_manager.go:60` |
| Volume Manager | `pkg/kubelet/volumemanager/volume_manager.go:85` | `pkg/kubelet/volumemanager/volume_manager.go:120` |
| Container Manager | `pkg/kubelet/cm/container_manager.go:65` | `pkg/kubelet/cm/container_manager_linux.go:180` |
| Eviction Manager | `pkg/kubelet/eviction/eviction_manager.go:55` | `pkg/kubelet/eviction/eviction_manager.go:85` |

## Summary

This reference guide provides quick access to the core manager implementations in kubelet:

### Navigation Guide

1. **PLEG**: Start at `pkg/kubelet/pleg/generic.go` for event generation
2. **Pod Manager**: Look at `pkg/kubelet/pod/pod_manager.go` for pod tracking
3. **Status Manager**: Check `pkg/kubelet/status/status_manager.go` for status sync
4. **Probe Manager**: See `pkg/kubelet/prober/prober_manager.go` for health checks
5. **Volume Manager**: Navigate to `pkg/kubelet/volumemanager/` for volume operations
6. **Container Manager**: Find resource management in `pkg/kubelet/cm/`
7. **Eviction Manager**: Look at `pkg/kubelet/eviction/` for pod eviction
8. **Pod Workers**: Check `pkg/kubelet/pod_workers.go` for worker goroutines

Each manager follows a similar pattern:
- Interface definition
- Implementation structure
- NewXxx() creation function
- Start() or Run() initialization
- Core operation functions

Use the quick reference tables to jump directly to specific implementations.