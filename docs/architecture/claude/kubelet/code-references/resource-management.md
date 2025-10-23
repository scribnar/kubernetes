# kubelet Code Reference: Resource Management

## Table of Contents
1. [Overview](#overview)
2. [CPU Manager](#cpu-manager)
3. [Memory Manager](#memory-manager)
4. [Device Manager](#device-manager)
5. [Topology Manager](#topology-manager)
6. [Container Manager](#container-manager)
7. [QoS Management](#qos-management)
8. [cgroup Management](#cgroup-management)
9. [Resource Calculation](#resource-calculation)
10. [Quick Reference Tables](#quick-reference-tables)

## Overview

This document provides comprehensive code references for resource management components in kubelet, including CPU, memory, device, and topology management.

### Resource Management Structure

```
pkg/kubelet/cm/                        # Container Manager root
├── cpumanager/                        # CPU Manager
│   ├── cpu_manager.go                # Manager implementation
│   ├── policy_static.go              # Static policy
│   ├── policy_none.go                # None policy
│   ├── cpu_assignment.go             # CPU allocation algorithm
│   ├── state/                        # State management
│   └── topology/                     # CPU topology
├── memorymanager/                     # Memory Manager
│   ├── memory_manager.go             # Manager implementation
│   ├── policy_static.go              # Static policy
│   ├── policy_none.go                # None policy
│   └── state/                        # State management
├── devicemanager/                     # Device Manager
│   ├── manager.go                    # Manager implementation
│   ├── endpoint.go                   # Device plugin endpoint
│   ├── plugin_watcher.go             # Plugin discovery
│   └── checkpoint/                   # State persistence
├── topologymanager/                   # Topology Manager
│   ├── topology_manager.go           # Manager implementation
│   ├── policy.go                     # Policy interface
│   ├── policy_best_effort.go         # Best effort policy
│   ├── policy_restricted.go          # Restricted policy
│   ├── policy_single_numa_node.go    # Single NUMA policy
│   └── scope.go                      # Container/Pod scope
└── qos_container_manager.go          # QoS cgroup management
```

## CPU Manager

### Manager Implementation
**File**: `pkg/kubelet/cm/cpumanager/cpu_manager.go`

#### Core Structure

```go
// Line 102
type manager struct {
    sync.Mutex
    policy Policy
    reconcilePeriod time.Duration

    // CPU assignment state
    state state.State
    lastUpdateState state.State

    // Runtime integration
    containerRuntime runtimeService
    activePods ActivePodsFunc
    podStatusProvider status.PodStatusProvider
    containerMap containermap.ContainerMap

    // CPU topology
    topology *topology.CPUTopology
    nodeAllocatableReservation v1.ResourceList

    // State persistence
    stateFileDirectory string

    // CPU sets
    allCPUs cpuset.CPUSet
    allocatableCPUs cpuset.CPUSet
}
```

#### Key Functions

```go
// Line 159 - Create CPU manager
func NewManager(cpuPolicyName string,
                cpuPolicyOptions map[string]string,
                reconcilePeriod time.Duration,
                machineInfo *cadvisorapi.MachineInfo,
                specificCPUs cpuset.CPUSet,
                nodeAllocatableReservation v1.ResourceList,
                stateFileDirectory string,
                affinity topologymanager.Store) (Manager, error) {

    topo, err := topology.Discover(machineInfo)

    var policy Policy
    switch policyName(cpuPolicyName) {
    case PolicyNone:
        policy, err = NewNonePolicy(cpuPolicyOptions)

    case PolicyStatic:
        reservedCPUs, ok := nodeAllocatableReservation[v1.ResourceCPU]
        numReservedCPUs := int(math.Ceil(float64(reservedCPUs.MilliValue()) / 1000))
        policy, err = NewStaticPolicy(topo, numReservedCPUs, specificCPUs, affinity, cpuPolicyOptions)
    }

    manager := &manager{
        policy:             policy,
        reconcilePeriod:    reconcilePeriod,
        topology:           topo,
        stateFileDirectory: stateFileDirectory,
        allCPUs:           topo.CPUDetails.CPUs(),
    }

    return manager, nil
}

// Line 320 - Allocate CPUs for container
func (m *manager) Allocate(pod *v1.Pod, container *v1.Container) error {
    m.Lock()
    defer m.Unlock()

    err := m.policy.Allocate(m.state, pod, container)
    if err != nil {
        return err
    }

    m.allocatableCPUs = m.policy.GetAllocatableCPUs(m.state)
    return nil
}

// Line 360 - Remove container CPU allocation
func (m *manager) RemoveContainer(containerID string) error {
    m.Lock()
    defer m.Unlock()

    err := m.policy.RemoveContainer(m.state, containerID)
    if err != nil {
        return err
    }

    m.allocatableCPUs = m.policy.GetAllocatableCPUs(m.state)
    return nil
}
```

### Static Policy
**File**: `pkg/kubelet/cm/cpumanager/policy_static.go`

```go
// Line 106
type staticPolicy struct {
    topology *topology.CPUTopology
    reservedCPUs cpuset.CPUSet
    reservedPhysicalCPUs cpuset.CPUSet
    affinity topologymanager.Store
    cpusToReuse map[string]cpuset.CPUSet
    options StaticPolicyOptions
    cpuGroupSize int
}

// Line 380 - Check if container eligible for exclusive CPUs
func (p *staticPolicy) podGuaranteedCPUs(pod *v1.Pod) int {
    if v1qos.GetPodQOS(pod) != v1.PodQOSGuaranteed {
        return 0
    }

    cpuQuantity := resource.Quantity{}
    for _, container := range pod.Spec.Containers {
        if cpu, ok := container.Resources.Requests[v1.ResourceCPU]; ok {
            cpuQuantity.Add(cpu)
        }
    }

    // Must be integer CPU request
    if cpuQuantity.Value()*1000 != cpuQuantity.MilliValue() {
        return 0
    }

    return int(cpuQuantity.Value())
}

// Line 450 - Allocate CPUs
func (p *staticPolicy) Allocate(s state.State, pod *v1.Pod, container *v1.Container) error {
    numCPUs := p.guaranteedCPUs(pod, container)
    if numCPUs == 0 {
        // Use shared pool
        return nil
    }

    // Get topology hint if available
    hint := p.affinity.GetAffinity(string(pod.UID), container.Name)

    // Allocate CPUs
    cpuset, err := p.allocateCPUs(s, numCPUs, hint)
    if err != nil {
        return err
    }

    s.SetCPUSet(string(pod.UID), container.Name, cpuset)
    s.SetDefaultCPUSet(s.GetDefaultCPUSet().Difference(cpuset))

    return nil
}
```

### CPU Assignment Algorithm
**File**: `pkg/kubelet/cm/cpumanager/cpu_assignment.go`

```go
// Line 200
type cpuAccumulator struct {
    topo          *topology.CPUTopology
    details       topology.CPUDetails
    numCPUsNeeded int
    result        cpuset.CPUSet

    // Available resources at each level
    freeCPUs      cpuset.CPUSet
    freeNUMANodes cpuset.CPUSet
    freeSockets   cpuset.CPUSet
    freeCores     cpuset.CPUSet
}

// Line 250
func takeByTopology(topo *topology.CPUTopology,
                   availableCPUs cpuset.CPUSet,
                   numCPUs int) (cpuset.CPUSet, error) {

    acc := newCPUAccumulator(topo, availableCPUs, numCPUs)

    // 1. Try whole NUMA nodes
    acc.takeFullNUMANodes()
    if acc.isSatisfied() {
        return acc.result, nil
    }

    // 2. Try whole sockets
    acc.takeFullSockets()
    if acc.isSatisfied() {
        return acc.result, nil
    }

    // 3. Try whole cores
    acc.takeFullCores()
    if acc.isSatisfied() {
        return acc.result, nil
    }

    // 4. Take individual CPUs
    acc.takeRemainingCPUs()
    return acc.result, nil
}
```

### CPU Manager State
**File**: `pkg/kubelet/cm/cpumanager/state/state.go`

```go
// Line 35
type State interface {
    Reader
    Writer
}

type Reader interface {
    GetCPUSet(podUID types.UID, containerName string) cpuset.CPUSet
    GetDefaultCPUSet() cpuset.CPUSet
    GetCPUSetOrDefault(podUID types.UID, containerName string) cpuset.CPUSet
    GetCPUAssignments() map[string]cpuset.CPUSet
}

type Writer interface {
    SetCPUSet(podUID types.UID, containerName string, cpuset cpuset.CPUSet)
    SetDefaultCPUSet(cpuset cpuset.CPUSet)
    Delete(podUID types.UID, containerName string)
    ClearState()
}
```

## Memory Manager

### Manager Implementation
**File**: `pkg/kubelet/cm/memorymanager/memory_manager.go`

```go
// Line 97
type manager struct {
    sync.Mutex
    policy Policy
    state state.State
    containerRuntime runtimeService
    activePods ActivePodsFunc
    podStatusProvider status.PodStatusProvider
    containerMap containermap.ContainerMap
    sourcesReady config.SourcesReady
    stateFileDirectory string
    allocatableMemory []state.Block
}

// Line 135 - Create memory manager
func NewManager(ctx context.Context,
               policyName string,
               machineInfo *cadvisorapi.MachineInfo,
               nodeAllocatableReservation v1.ResourceList,
               reservedMemory []kubeletconfig.MemoryReservation,
               stateFileDirectory string,
               affinity topologymanager.Store) (Manager, error) {

    var policy Policy
    switch policyType(policyName) {
    case policyTypeNone:
        policy = NewPolicyNone(ctx)

    case PolicyTypeStatic:
        systemReserved, err := getSystemReservedMemory(machineInfo, nodeAllocatableReservation, reservedMemory)
        policy, err = NewPolicyStatic(ctx, machineInfo, systemReserved, affinity)
    }

    manager := &manager{
        policy:             policy,
        stateFileDirectory: stateFileDirectory,
    }

    return manager, nil
}

// Line 270 - Allocate memory for container
func (m *manager) Allocate(pod *v1.Pod, container *v1.Container) error {
    m.Lock()
    defer m.Unlock()

    err := m.policy.Allocate(context.TODO(), m.state, pod, container)
    if err != nil {
        return err
    }

    m.allocatableMemory = m.policy.GetAllocatableMemory(context.TODO(), m.state)
    return nil
}
```

### Static Policy
**File**: `pkg/kubelet/cm/memorymanager/policy_static.go`

```go
// Line 47
type staticPolicy struct {
    machineInfo *cadvisorapi.MachineInfo
    systemReserved systemReservedMemory
    affinity topologymanager.Store
    initContainersReusableMemory reusableMemory
}

// Line 101 - Allocate memory
func (p *staticPolicy) Allocate(ctx context.Context,
                               s state.State,
                               pod *v1.Pod,
                               container *v1.Container) error {

    // Only for guaranteed pods
    qos := v1qos.GetPodQOS(pod)
    if qos != v1.PodQOSGuaranteed {
        return nil
    }

    // Check if already allocated
    if blocks := s.GetMemoryBlocks(string(pod.UID), container.Name); blocks != nil {
        return nil
    }

    // Get topology hint
    hint := p.affinity.GetAffinity(string(pod.UID), container.Name)

    // Calculate requested resources
    requestedResources, err := getRequestedResources(pod, container)

    // Allocate memory blocks
    var containerBlocks []state.Block
    maskBits := hint.NUMANodeAffinity.GetBits()

    for resourceName, requestedSize := range requestedResources {
        containerBlocks = append(containerBlocks, state.Block{
            NUMAAffinity: maskBits,
            Size:         requestedSize,
            Type:         resourceName,
        })
    }

    s.SetMemoryBlocks(string(pod.UID), container.Name, containerBlocks)
    return nil
}
```

### Memory Manager State
**File**: `pkg/kubelet/cm/memorymanager/state/state.go`

```go
// Line 80
type Block struct {
    NUMAAffinity []int
    Type         v1.ResourceName
    Size         uint64
}

// Line 88
type ContainerMemoryAssignments map[string]map[string][]Block

// Line 103
type Reader interface {
    GetMachineState() NUMANodeMap
    GetMemoryBlocks(podUID string, containerName string) []Block
    GetMemoryAssignments() ContainerMemoryAssignments
}

// Line 113
type State interface {
    Reader
    SetMachineState(memoryMap NUMANodeMap)
    SetMemoryBlocks(podUID string, containerName string, blocks []Block)
    Delete(podUID string, containerName string)
    ClearState()
}
```

## Device Manager

### Manager Implementation
**File**: `pkg/kubelet/cm/devicemanager/manager.go`

```go
// Line 95
type ManagerImpl struct {
    sync.Mutex

    // Device plugin endpoints
    endpoints map[string]endpoint
    endpointsStopped chan struct{}

    // Resource allocation
    allocatedDevices map[string]sets.Set[string]

    // Pod devices
    podDevices podDevices

    // Topology manager integration
    topologyAffinityStore topologymanager.Store

    // Plugin management
    pluginWatcher pluginwatcher.PluginWatcher

    // State checkpoint
    checkpointManager checkpointmanager.CheckpointManager
}

// Line 150 - Create device manager
func NewManagerImpl(topology []cadvisorapi.Node,
                   topologyAffinityStore topologymanager.Store) (*ManagerImpl, error) {

    manager := &ManagerImpl{
        endpoints:             make(map[string]endpoint),
        allocatedDevices:      make(map[string]sets.Set[string]),
        podDevices:           newPodDevices(),
        topologyAffinityStore: topologyAffinityStore,
    }

    manager.pluginWatcher = pluginwatcher.NewWatcher(
        devicePluginPath,
        manager,
    )

    return manager, nil
}

// Line 250 - Allocate devices
func (m *ManagerImpl) Allocate(pod *v1.Pod, container *v1.Container) error {
    m.Lock()
    defer m.Unlock()

    // Get container device requests
    requests := container.Resources.Requests

    // Allocate each requested resource
    for resource, quantity := range requests {
        if !m.isDevicePluginResource(string(resource)) {
            continue
        }

        needed := int(quantity.Value())
        endpoint := m.endpoints[string(resource)]

        // Get available devices
        available := m.getAvailableDevices(string(resource))

        // Select devices based on topology hint
        hint := m.topologyAffinityStore.GetAffinity(string(pod.UID), container.Name)
        selected := m.selectDevices(available, needed, hint)

        // Allocate via plugin
        response, err := endpoint.allocate(selected)
        if err != nil {
            return err
        }

        // Update allocated devices
        m.allocatedDevices[string(resource)].Insert(selected...)

        // Store pod device allocation
        m.podDevices.insert(pod.UID, container.Name, string(resource), selected, response)
    }

    return nil
}
```

### Device Plugin Endpoint
**File**: `pkg/kubelet/cm/devicemanager/endpoint.go`

```go
// Line 60
type endpoint interface {
    getDevices() []*pluginapi.Device
    allocate(devices []string) (*pluginapi.AllocateResponse, error)
    preStartContainer(devices []string) (*pluginapi.PreStartContainerResponse, error)
    callback(resourceName string, devices []*pluginapi.Device)
    stop()
}

// Line 85
type endpointImpl struct {
    sync.Mutex

    // Plugin connection
    client     pluginapi.DevicePluginClient
    clientConn *grpc.ClientConn

    // Resource info
    resourceName string

    // Device list
    devices map[string]*pluginapi.Device

    // Callbacks
    callback monitorCallback
}

// Line 150 - Allocate devices via plugin
func (e *endpointImpl) allocate(deviceIDs []string) (*pluginapi.AllocateResponse, error) {
    // Build allocation request
    req := &pluginapi.AllocateRequest{
        ContainerRequests: []*pluginapi.ContainerAllocateRequest{{
            DevicesIDs: deviceIDs,
        }},
    }

    // Call plugin
    resp, err := e.client.Allocate(context.Background(), req)
    if err != nil {
        return nil, err
    }

    return resp.ContainerResponses[0], nil
}
```

### Device Plugin Registration
**File**: `pkg/kubelet/cm/devicemanager/plugin_watcher.go`

```go
// Line 45
func (m *ManagerImpl) GetWatcherHandler() cache.PluginHandler {
    return cache.PluginHandler{
        RegisterPlugin:   m.registerPlugin,
        DeRegisterPlugin: m.deregisterPlugin,
    }
}

// Line 60
func (m *ManagerImpl) registerPlugin(pluginName string, endpoint string, versions []string) error {
    // Create gRPC client
    client, conn, err := dial(endpoint)
    if err != nil {
        return err
    }

    // Get resource name from plugin
    resp, err := client.GetDevicePluginOptions(context.Background(), &pluginapi.Empty{})
    resourceName := resp.ResourceName

    // Create endpoint
    ep := newEndpointImpl(resourceName, client, conn)

    // Start monitoring
    go ep.monitor()

    // Store endpoint
    m.Lock()
    m.endpoints[resourceName] = ep
    m.Unlock()

    return nil
}
```

## Topology Manager

### Manager Implementation
**File**: `pkg/kubelet/cm/topologymanager/topology_manager.go`

```go
// Line 72
type manager struct {
    scope Scope
}

// Line 57
type scope struct {
    sync.Mutex
    name string
    podTopologyHints podTopologyHints
    hintProviders []HintProvider
    policy Policy
    podMap containermap.ContainerMap
}

// Line 135 - Create topology manager
func NewManager(topology []cadvisorapi.Node,
               topologyPolicyName string,
               topologyScopeName string,
               topologyPolicyOptions map[string]string) (Manager, error) {

    if topologyPolicyName == PolicyNone {
        return &manager{scope: NewNoneScope()}, nil
    }

    numaInfo, err := NewNUMAInfo(topology, opts)

    var policy Policy
    switch topologyPolicyName {
    case PolicyBestEffort:
        policy = NewBestEffortPolicy(numaInfo, opts)
    case PolicyRestricted:
        policy = NewRestrictedPolicy(numaInfo, opts)
    case PolicySingleNumaNode:
        policy = NewSingleNumaNodePolicy(numaInfo, opts)
    }

    var scope Scope
    switch topologyScopeName {
    case containerTopologyScope:
        scope = NewContainerScope(policy)
    case podTopologyScope:
        scope = NewPodScope(policy)
    }

    return &manager{scope: scope}, nil
}

// Line 229 - Pod admission
func (m *manager) Admit(attrs *lifecycle.PodAdmitAttributes) lifecycle.PodAdmitResult {
    podAdmitResult := m.scope.Admit(context.TODO(), attrs.Pod)
    return podAdmitResult
}
```

### Topology Policies
**File**: `pkg/kubelet/cm/topologymanager/policy.go`

```go
// Line 25
type Policy interface {
    Name() string
    Merge(logger klog.Logger, providersHints []map[string][]TopologyHint) (TopologyHint, bool)
}

// Line 44 - Merge hints
func mergePermutation(defaultAffinity bitmask.BitMask, permutation []TopologyHint) TopologyHint {
    preferred := true
    var numaAffinities []bitmask.BitMask

    for _, hint := range permutation {
        if hint.NUMANodeAffinity != nil {
            numaAffinities = append(numaAffinities, hint.NUMANodeAffinity)
            if !hint.Preferred {
                preferred = false
            }
        }
    }

    // Merge using bitwise AND
    mergedAffinity := bitmask.And(defaultAffinity, numaAffinities...)

    return TopologyHint{
        NUMANodeAffinity: mergedAffinity,
        Preferred:        preferred,
    }
}
```

### Container Scope
**File**: `pkg/kubelet/cm/topologymanager/scope_container.go`

```go
// Line 45
func (s *containerScope) Admit(ctx context.Context, pod *v1.Pod) lifecycle.PodAdmitResult {
    for _, container := range append(pod.Spec.InitContainers, pod.Spec.Containers...) {
        result := s.admitContainer(ctx, pod, &container)
        if !result.Admit {
            return result
        }
    }
    return admission.GetPodAdmitResult(nil)
}

// Line 65
func (s *containerScope) admitContainer(ctx context.Context,
                                       pod *v1.Pod,
                                       container *v1.Container) lifecycle.PodAdmitResult {
    // Gather hints
    providersHints := s.gatherHints(pod, container)

    // Merge hints
    bestHint, admit := s.policy.Merge(providersHints)
    if !admit {
        return admission.GetPodAdmitResult(TopologyAffinityError{})
    }

    // Store hint
    s.setTopologyHints(string(pod.UID), container.Name, bestHint)

    // Allocate resources
    for _, provider := range s.hintProviders {
        err := provider.Allocate(pod, container)
        if err != nil {
            return admission.GetPodAdmitResult(err)
        }
    }

    return admission.GetPodAdmitResult(nil)
}
```

## Container Manager

### Main Container Manager
**File**: `pkg/kubelet/cm/container_manager_linux.go`

```go
// Line 180
type containerManagerImpl struct {
    sync.RWMutex

    machineInfo cadvisorapi.MachineInfo
    capacity v1.ResourceList
    nodeAllocatableReservation v1.ResourceList

    cadvisorInterface cadvisor.Interface
    cgroupManager CgroupManager

    // Sub-managers
    qosManager *qosContainerManager
    cpuManager cpumanager.Manager
    memoryManager memorymanager.Manager
    deviceManager devicemanager.Manager
    topologyManager topologymanager.Manager
}

// Line 320 - Create container manager
func NewContainerManager(mountUtil mount.Interface,
                        cadvisorInterface cadvisor.Interface,
                        nodeConfig NodeConfig,
                        failSwapOn bool,
                        recorder record.EventRecorder,
                        kubeClient clientset.Interface) (ContainerManager, error) {

    subsystems, err := GetCgroupSubsystems()
    cgroupManager := NewCgroupManager(subsystems, nodeConfig.CgroupDriver)

    cm := &containerManagerImpl{
        cadvisorInterface: cadvisorInterface,
        cgroupManager:     cgroupManager,
        capacity:          capacityFromMachineInfo(machineInfo),
    }

    // Create QoS manager
    cm.qosManager = NewQOSContainerManager(subsystems, cgroupRoot, nodeConfig)

    // Create CPU manager
    cm.cpuManager, err = cpumanager.NewManager(
        nodeConfig.CPUManagerPolicy,
        nodeConfig.CPUManagerPolicyOptions,
        nodeConfig.CPUManagerReconcilePeriod,
        machineInfo,
        nodeConfig.ReservedSystemCPUs,
        cm.GetNodeAllocatableReservation(),
        nodeConfig.KubeletRootDir,
        cm.topologyManager,
    )

    // Create memory manager
    cm.memoryManager, err = memorymanager.NewManager(
        nodeConfig.MemoryManagerPolicy,
        machineInfo,
        cm.GetNodeAllocatableReservation(),
        nodeConfig.ReservedMemory,
        nodeConfig.KubeletRootDir,
        cm.topologyManager,
    )

    // Create device manager
    cm.deviceManager, err = devicemanager.NewManagerImpl(
        machineInfo.Topology,
        cm.topologyManager,
    )

    // Create topology manager
    cm.topologyManager, err = topologymanager.NewManager(
        machineInfo.Topology,
        nodeConfig.TopologyManagerPolicy,
        nodeConfig.TopologyManagerScope,
        nodeConfig.TopologyManagerPolicyOptions,
    )

    return cm, nil
}
```

## QoS Management

### QoS Container Manager
**File**: `pkg/kubelet/cm/qos_container_manager.go`

```go
// Line 55
type qosContainerManager struct {
    sync.Mutex
    cgroupManager CgroupManager
    cgroupRoot string
    qosConfigs map[v1.PodQOSClass]*CgroupConfig
}

// Line 85 - Setup QoS cgroups
func (m *qosContainerManager) Start(getNodeAllocatable func() v1.ResourceList,
                                   activePods ActivePodsFunc) error {

    rootContainer := m.cgroupManager.Name(m.cgroupRoot, nil)

    // Create QoS containers
    for qosClass := range m.qosConfigs {
        cgroupName := m.cgroupManager.Name(rootContainer, &CgroupName{qosClass})

        if err := m.cgroupManager.Create(&CgroupConfig{
            Name: cgroupName,
            ResourceParameters: &ResourceConfig{
                Memory: m.qosConfigs[qosClass].ResourceParameters.Memory,
            },
        }); err != nil {
            return err
        }
    }

    // Update QoS cgroup resource limits
    m.UpdateCgroups()

    return nil
}

// Line 145 - Get pod cgroup parent
func (m *qosContainerManager) GetPodContainerName(pod *v1.Pod) (CgroupName, string) {
    qosClass := pod.Status.QOSClass
    if qosClass == "" {
        qosClass = v1qos.GetPodQOS(pod)
    }

    parentContainer := CgroupName{qosClass}
    podContainer := GetPodCgroupNameSuffix(pod.UID)

    return parentContainer, podContainer
}
```

## cgroup Management

### cgroup Manager
**File**: `pkg/kubelet/cm/cgroup_manager_linux.go`

```go
// Line 85
type cgroupManagerImpl struct {
    subsystems *CgroupSubsystems
    adapter cgroupAdapter
}

// Line 120 - Create cgroup
func (m *cgroupManagerImpl) Create(cgroupConfig *CgroupConfig) error {
    cgroupPaths := m.buildCgroupPaths(cgroupConfig.Name)

    // Create cgroup in each subsystem
    for controller, path := range cgroupPaths {
        if err := os.MkdirAll(path, 0755); err != nil {
            return err
        }
    }

    // Set resource parameters
    if cgroupConfig.ResourceParameters != nil {
        if err := m.setResourceParameters(cgroupPaths, cgroupConfig.ResourceParameters); err != nil {
            return err
        }
    }

    return nil
}

// Line 180 - Update cgroup resources
func (m *cgroupManagerImpl) Update(cgroupConfig *CgroupConfig) error {
    cgroupPaths := m.buildCgroupPaths(cgroupConfig.Name)

    if cgroupConfig.ResourceParameters.Memory != nil {
        memoryPath := filepath.Join(cgroupPaths["memory"], "memory.limit_in_bytes")
        if err := os.WriteFile(memoryPath,
            []byte(strconv.FormatInt(*cgroupConfig.ResourceParameters.Memory, 10)), 0644); err != nil {
            return err
        }
    }

    if cgroupConfig.ResourceParameters.CPUQuota != nil {
        cpuPath := filepath.Join(cgroupPaths["cpu"], "cpu.cfs_quota_us")
        if err := os.WriteFile(cpuPath,
            []byte(strconv.FormatInt(*cgroupConfig.ResourceParameters.CPUQuota, 10)), 0644); err != nil {
            return err
        }
    }

    return nil
}

// Line 250 - Destroy cgroup
func (m *cgroupManagerImpl) Destroy(cgroupName CgroupName) error {
    cgroupPaths := m.buildCgroupPaths(cgroupName)

    for _, path := range cgroupPaths {
        if err := os.RemoveAll(path); err != nil {
            return err
        }
    }

    return nil
}
```

## Resource Calculation

### Node Allocatable
**File**: `pkg/kubelet/cm/node_container_manager_linux.go`

```go
// Line 85
func (cm *containerManagerImpl) GetNodeAllocatableReservation() v1.ResourceList {
    evictionReservation := hardEvictionReservation(
        cm.nodeConfig.HardEvictionThresholds,
        cm.capacity,
    )

    result := make(v1.ResourceList)

    for k := range cm.capacity {
        value := cm.capacity[k].DeepCopy()

        // Subtract kube-reserved
        if kubeReserved, exists := cm.nodeConfig.KubeReserved[k]; exists {
            value.Sub(kubeReserved)
        }

        // Subtract system-reserved
        if systemReserved, exists := cm.nodeConfig.SystemReserved[k]; exists {
            value.Sub(systemReserved)
        }

        // Subtract eviction reservation
        if evictionValue, exists := evictionReservation[k]; exists {
            value.Sub(evictionValue)
        }

        if value.Sign() < 0 {
            value.Set(0)
        }

        result[k] = value
    }

    return result
}

// Line 150 - Enforce node allocatable
func (cm *containerManagerImpl) enforceNodeAllocatable() error {
    // Get pods cgroup
    podsCgroup := cm.cgroupManager.Name(cm.cgroupRoot, &CgroupName{"pods"})

    // Calculate allocatable resources
    allocatable := cm.GetNodeAllocatableReservation()

    // Update pods cgroup
    cgroupConfig := &CgroupConfig{
        Name: podsCgroup,
        ResourceParameters: &ResourceConfig{
            Memory: allocatable.Memory().Value(),
            CPUShares: MilliCPUToShares(allocatable.Cpu().MilliValue()),
        },
    }

    return cm.cgroupManager.Update(cgroupConfig)
}
```

## Quick Reference Tables

### Resource Manager Files

| Manager | Main File | Policy Files | State Files |
|---------|-----------|--------------|-------------|
| CPU Manager | `cm/cpumanager/cpu_manager.go` | `policy_static.go`, `policy_none.go` | `state/state.go` |
| Memory Manager | `cm/memorymanager/memory_manager.go` | `policy_static.go`, `policy_none.go` | `state/state.go` |
| Device Manager | `cm/devicemanager/manager.go` | N/A | `checkpoint/checkpoint.go` |
| Topology Manager | `cm/topologymanager/topology_manager.go` | `policy_*.go` | N/A |

### Key Functions by Manager

| Manager | Create | Allocate | Remove | Start |
|---------|--------|----------|---------|-------|
| CPU | `NewManager()` :159 | `Allocate()` :320 | `RemoveContainer()` :360 | `Start()` :220 |
| Memory | `NewManager()` :135 | `Allocate()` :270 | `RemoveContainer()` :310 | `Start()` :185 |
| Device | `NewManagerImpl()` :150 | `Allocate()` :250 | `DeallocateDevices()` :380 | `Start()` :200 |
| Topology | `NewManager()` :135 | N/A | N/A | N/A |

### Resource Interfaces

| Interface | File | Line | Purpose |
|-----------|------|------|---------|
| `cpumanager.Manager` | `cm/cpumanager/cpu_manager.go` | 55 | CPU management |
| `memorymanager.Manager` | `cm/memorymanager/memory_manager.go` | 58 | Memory management |
| `devicemanager.Manager` | `cm/devicemanager/manager.go` | 65 | Device plugin management |
| `topologymanager.Manager` | `cm/topologymanager/topology_manager.go` | 58 | NUMA topology coordination |

### Policy Implementations

| Manager | Policy | File | Line |
|---------|--------|------|------|
| CPU | None | `cm/cpumanager/policy_none.go` | 30 |
| CPU | Static | `cm/cpumanager/policy_static.go` | 106 |
| Memory | None | `cm/memorymanager/policy_none.go` | 25 |
| Memory | Static | `cm/memorymanager/policy_static.go` | 47 |
| Topology | None | `cm/topologymanager/policy_none.go` | 25 |
| Topology | BestEffort | `cm/topologymanager/policy_best_effort.go` | 35 |
| Topology | Restricted | `cm/topologymanager/policy_restricted.go` | 35 |
| Topology | SingleNumaNode | `cm/topologymanager/policy_single_numa_node.go` | 35 |

### State Management

| Manager | State Interface | Checkpoint File |
|---------|----------------|-----------------|
| CPU | `state.State` | `/var/lib/kubelet/cpu_manager_state` |
| Memory | `state.State` | `/var/lib/kubelet/memory_manager_state` |
| Device | `podDevices` | `/var/lib/kubelet/device-plugins/checkpoint` |

## Summary

This reference provides comprehensive navigation for kubelet resource management:

### Navigation Guide

1. **CPU Manager**: Start at `pkg/kubelet/cm/cpumanager/` for CPU pinning and allocation
2. **Memory Manager**: Look at `pkg/kubelet/cm/memorymanager/` for NUMA-aware memory
3. **Device Manager**: Check `pkg/kubelet/cm/devicemanager/` for device plugin support
4. **Topology Manager**: See `pkg/kubelet/cm/topologymanager/` for NUMA coordination
5. **Container Manager**: Navigate to `pkg/kubelet/cm/` for overall resource management
6. **cgroup Management**: Find cgroup operations in `pkg/kubelet/cm/cgroup_manager_linux.go`

### Key Design Patterns

- **Policy Pattern**: Each manager has pluggable policies (none, static, etc.)
- **State Management**: Checkpoint-based persistence for recovery
- **Topology Awareness**: NUMA-based resource allocation
- **Plugin Architecture**: Device manager supports external plugins
- **Hint System**: Topology manager coordinates via hints

Use the quick reference tables to jump directly to specific implementations and functions.