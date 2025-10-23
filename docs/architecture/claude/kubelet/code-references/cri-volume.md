# kubelet Code Reference: CRI and Volume Plugins

## Table of Contents
1. [Overview](#overview)
2. [CRI Client](#cri-client)
3. [Runtime Manager](#runtime-manager)
4. [Image Service](#image-service)
5. [Volume Plugin Framework](#volume-plugin-framework)
6. [CSI Plugin](#csi-plugin)
7. [Built-in Volume Plugins](#built-in-volume-plugins)
8. [Volume Operations](#volume-operations)
9. [Mount Utilities](#mount-utilities)
10. [Quick Reference Tables](#quick-reference-tables)

## Overview

This document provides comprehensive code references for CRI (Container Runtime Interface) integration and volume plugin implementations in kubelet.

### CRI and Volume Structure

```
pkg/kubelet/
├── cri/                          # CRI integration
│   ├── remote/                   # Remote runtime client
│   │   ├── remote_runtime.go    # RuntimeService client
│   │   └── remote_image.go      # ImageService client
│   └── streaming/                # Streaming server
│       └── server.go            # Exec/Attach/PortForward
├── kuberuntime/                  # Generic runtime manager
│   ├── kuberuntime_manager.go   # Main runtime manager
│   ├── kuberuntime_container.go # Container operations
│   ├── kuberuntime_sandbox.go   # Pod sandbox operations
│   └── kuberuntime_image.go     # Image operations
└── volume/                       # Volume plugins
    ├── volume.go                 # Plugin interfaces
    ├── plugins.go                # Plugin registry
    ├── csi/                      # CSI plugin
    ├── emptydir/                 # EmptyDir plugin
    ├── hostpath/                 # HostPath plugin
    ├── configmap/                # ConfigMap plugin
    └── secret/                   # Secret plugin
```

## CRI Client

### Remote Runtime Client
**File**: `pkg/kubelet/cri/remote/remote_runtime.go`

```go
// Line 85
type remoteRuntimeService struct {
    client           runtimeapi.RuntimeServiceClient
    conn             *grpc.ClientConn
    timeout          time.Duration
    logReduction     *logreduction.LogReducer
    instrumentedClient runtimeapi.RuntimeServiceClient
}

// Line 120 - Create CRI runtime client
func NewRemoteRuntimeService(endpoint string, connectionTimeout time.Duration) (RuntimeService, error) {
    conn, err := createConnection(endpoint, connectionTimeout)
    if err != nil {
        return nil, err
    }

    service := &remoteRuntimeService{
        client:  runtimeapi.NewRuntimeServiceClient(conn),
        conn:    conn,
        timeout: connectionTimeout,
    }

    // Wrap with instrumentation
    service.instrumentedClient = newInstrumentedRuntimeClient(service.client)

    return service, nil
}

// Line 180 - Version negotiation
func (r *remoteRuntimeService) Version(ctx context.Context) (*runtimeapi.VersionResponse, error) {
    ctx, cancel := context.WithTimeout(ctx, r.timeout)
    defer cancel()

    return r.client.Version(ctx, &runtimeapi.VersionRequest{})
}

// Line 200 - Create pod sandbox
func (r *remoteRuntimeService) RunPodSandbox(ctx context.Context, config *runtimeapi.PodSandboxConfig,
    runtimeHandler string) (string, error) {

    ctx, cancel := context.WithTimeout(ctx, r.timeout)
    defer cancel()

    req := &runtimeapi.RunPodSandboxRequest{
        Config:         config,
        RuntimeHandler: runtimeHandler,
    }

    resp, err := r.client.RunPodSandbox(ctx, req)
    if err != nil {
        return "", err
    }

    return resp.PodSandboxId, nil
}

// Line 250 - Create container
func (r *remoteRuntimeService) CreateContainer(ctx context.Context,
    podSandboxID string,
    config *runtimeapi.ContainerConfig,
    sandboxConfig *runtimeapi.PodSandboxConfig) (string, error) {

    ctx, cancel := context.WithTimeout(ctx, r.timeout)
    defer cancel()

    req := &runtimeapi.CreateContainerRequest{
        PodSandboxId:  podSandboxID,
        Config:        config,
        SandboxConfig: sandboxConfig,
    }

    resp, err := r.client.CreateContainer(ctx, req)
    if err != nil {
        return "", err
    }

    return resp.ContainerId, nil
}

// Line 300 - Start container
func (r *remoteRuntimeService) StartContainer(ctx context.Context, containerID string) error {
    ctx, cancel := context.WithTimeout(ctx, r.timeout)
    defer cancel()

    _, err := r.client.StartContainer(ctx, &runtimeapi.StartContainerRequest{
        ContainerId: containerID,
    })

    return err
}

// Line 350 - Stop container
func (r *remoteRuntimeService) StopContainer(ctx context.Context, containerID string, timeout int64) error {
    ctx, cancel := context.WithTimeout(ctx, r.timeout)
    defer cancel()

    _, err := r.client.StopContainer(ctx, &runtimeapi.StopContainerRequest{
        ContainerId: containerID,
        Timeout:     timeout,
    })

    return err
}
```

### Remote Image Service
**File**: `pkg/kubelet/cri/remote/remote_image.go`

```go
// Line 55
type remoteImageService struct {
    client  runtimeapi.ImageServiceClient
    conn    *grpc.ClientConn
    timeout time.Duration
}

// Line 85 - Create image service client
func NewRemoteImageService(endpoint string, connectionTimeout time.Duration) (ImageService, error) {
    conn, err := createConnection(endpoint, connectionTimeout)
    if err != nil {
        return nil, err
    }

    return &remoteImageService{
        client:  runtimeapi.NewImageServiceClient(conn),
        conn:    conn,
        timeout: connectionTimeout,
    }, nil
}

// Line 120 - Pull image
func (r *remoteImageService) PullImage(ctx context.Context,
    image *runtimeapi.ImageSpec,
    auth *runtimeapi.AuthConfig,
    podSandboxConfig *runtimeapi.PodSandboxConfig) (string, error) {

    ctx, cancel := context.WithTimeout(ctx, r.timeout)
    defer cancel()

    req := &runtimeapi.PullImageRequest{
        Image:         image,
        Auth:          auth,
        SandboxConfig: podSandboxConfig,
    }

    resp, err := r.client.PullImage(ctx, req)
    if err != nil {
        return "", err
    }

    return resp.ImageRef, nil
}

// Line 180 - List images
func (r *remoteImageService) ListImages(ctx context.Context,
    filter *runtimeapi.ImageFilter) ([]*runtimeapi.Image, error) {

    ctx, cancel := context.WithTimeout(ctx, r.timeout)
    defer cancel()

    resp, err := r.client.ListImages(ctx, &runtimeapi.ListImagesRequest{
        Filter: filter,
    })

    if err != nil {
        return nil, err
    }

    return resp.Images, nil
}

// Line 220 - Remove image
func (r *remoteImageService) RemoveImage(ctx context.Context, image *runtimeapi.ImageSpec) error {
    ctx, cancel := context.WithTimeout(ctx, r.timeout)
    defer cancel()

    _, err := r.client.RemoveImage(ctx, &runtimeapi.RemoveImageRequest{
        Image: image,
    })

    return err
}
```

## Runtime Manager

### KubeGenericRuntimeManager
**File**: `pkg/kubelet/kuberuntime/kuberuntime_manager.go`

```go
// Line 120
type kubeGenericRuntimeManager struct {
    runtimeName         string
    runtimeService      internalapi.RuntimeService
    imageService        internalapi.ImageService
    internalLifecycle   cm.InternalContainerLifecycle
    runner              kubecontainer.CommandRunner

    // Pod lifecycle
    containerRefManager *kubecontainer.RefManager

    // Image management
    imagePuller         images.ImageManager
    imageBackOff        *flowcontrol.Backoff

    // OSInterface for file operations
    osInterface kubecontainer.OSInterface

    // Pod logs
    logManager logs.ContainerLogManager

    // Runtime cache
    runtimeCache kubecontainer.RuntimeCache

    // Pod state provider
    podStateProvider podStateProvider
}

// Line 200 - Create runtime manager
func NewKubeGenericRuntimeManager(
    recorder record.EventRecorder,
    livenessManager proberesults.Manager,
    readinessManager proberesults.Manager,
    startupManager proberesults.Manager,
    rootDirectory string,
    machineInfo *cadvisorapi.MachineInfo,
    podStateProvider podStateProvider,
    osInterface kubecontainer.OSInterface,
    runtimeService internalapi.RuntimeService,
    imageService internalapi.ImageService,
    imageBackOff *flowcontrol.Backoff,
    serializeImagePulls bool,
    maxParallelImagePulls int32,
    imagePullQPS float32,
    imagePullBurst int,
    cpuCFSQuota bool,
    cpuCFSQuotaPeriod metav1.Duration,
    runtimeClassManager *runtimeclass.Manager,
) (kubecontainer.Runtime, error) {

    kubeRuntimeManager := &kubeGenericRuntimeManager{
        runtimeName:       runtimeName,
        runtimeService:    runtimeService,
        imageService:      imageService,
        osInterface:       osInterface,
        runtimeHelper:     runtimeHelper,
        imageBackOff:      imageBackOff,
        podStateProvider:  podStateProvider,
        recorder:          recorder,
    }

    kubeRuntimeManager.runtimeCache = kubecontainer.NewRuntimeCache(kubeRuntimeManager, cachePeriod)

    return kubeRuntimeManager, nil
}

// Line 350 - Sync pod
func (m *kubeGenericRuntimeManager) SyncPod(ctx context.Context,
    pod *v1.Pod,
    podStatus *kubecontainer.PodStatus,
    pullSecrets []v1.Secret,
    backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {

    // Step 1: Compute sandbox and container changes
    podContainerChanges := m.computePodActions(ctx, pod, podStatus)

    // Step 2: Kill pod sandbox if necessary
    if podContainerChanges.KillPod {
        m.killPodWithSyncResult(ctx, pod, kubecontainer.ConvertPodStatusToRunningPod(m.runtimeName, podStatus), nil)
    } else {
        // Step 3: Kill containers in the pod
        for containerID, containerInfo := range podContainerChanges.ContainersToKill {
            m.killContainer(ctx, pod, containerID, containerInfo.name, containerInfo.message, containerInfo.reason, nil)
        }
    }

    // Step 4: Create sandbox if necessary
    podSandboxID := podContainerChanges.SandboxID
    if podContainerChanges.CreateSandbox {
        podSandboxID, msg, err = m.createPodSandbox(ctx, pod, podContainerChanges.Attempt)
    }

    // Step 5: Start ephemeral containers
    if utilfeature.DefaultFeatureGate.Enabled(features.EphemeralContainers) {
        for _, idx := range podContainerChanges.EphemeralContainersToStart {
            container := pod.Spec.EphemeralContainers[idx]
            m.startContainer(ctx, podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs)
        }
    }

    // Step 6: Start init containers
    if container := podContainerChanges.NextInitContainerToStart; container != nil {
        msg, err := m.startContainer(ctx, podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs)
    }

    // Step 7: Start containers in podContainerChanges.ContainersToStart
    for _, idx := range podContainerChanges.ContainersToStart {
        container := pod.Spec.Containers[idx]
        m.startContainer(ctx, podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs)
    }

    return result
}
```

### Container Operations
**File**: `pkg/kubelet/kuberuntime/kuberuntime_container.go`

```go
// Line 150 - Start container
func (m *kubeGenericRuntimeManager) startContainer(ctx context.Context,
    podSandboxID string,
    podSandboxConfig *runtimeapi.PodSandboxConfig,
    spec *startSpec,
    pod *v1.Pod,
    podStatus *kubecontainer.PodStatus,
    pullSecrets []v1.Secret,
    podIP string,
    podIPs []string) (string, error) {

    // Step 1: Pull the image
    imageRef, msg, err := m.imagePuller.EnsureImageExists(ctx, pod, container, pullSecrets, podSandboxConfig)

    // Step 2: Create the container
    containerConfig, cleanupAction, err := m.generateContainerConfig(ctx, container, pod, restartCount, podIP, imageRef, podIPs, target)

    containerID, err := m.runtimeService.CreateContainer(ctx, podSandboxID, containerConfig, podSandboxConfig)

    // Step 3: Start the container
    err = m.runtimeService.StartContainer(ctx, containerID)

    // Step 4: Execute post-start lifecycle hooks
    if container.Lifecycle != nil && container.Lifecycle.PostStart != nil {
        m.runner.Run(ctx, kubecontainer.ContainerID{Type: m.runtimeName, ID: containerID}, pod, container, container.Lifecycle.PostStart)
    }

    return "", nil
}

// Line 350 - Generate container config
func (m *kubeGenericRuntimeManager) generateContainerConfig(ctx context.Context,
    container *v1.Container,
    pod *v1.Pod,
    restartCount int,
    podIP string,
    imageRef string,
    podIPs []string,
    nsTarget *kubecontainer.ContainerID) (*runtimeapi.ContainerConfig, func(), error) {

    config := &runtimeapi.ContainerConfig{
        Metadata: &runtimeapi.ContainerMetadata{
            Name:    container.Name,
            Attempt: uint32(restartCount),
        },
        Image:       &runtimeapi.ImageSpec{Image: imageRef},
        Command:     command,
        Args:        args,
        WorkingDir:  container.WorkingDir,
        Labels:      newContainerLabels(container, pod),
        Annotations: newContainerAnnotations(container, pod, restartCount, opts),
        Devices:     makeDevices(opts),
        CDIDevices:  makeCDIDevices(opts),
        Mounts:      m.makeMounts(opts, container),
        LogPath:     containerLogsPath,
        Stdin:       container.Stdin,
        StdinOnce:   container.StdinOnce,
        Tty:         container.TTY,
    }

    // Set Linux-specific options
    if runtime.GOOS == "linux" {
        config.Linux = m.generateLinuxContainerConfig(container, pod, uid, username, nsTarget)
    }

    // Set Windows-specific options
    if runtime.GOOS == "windows" {
        config.Windows = m.generateWindowsContainerConfig(container, pod, uid, username, nsTarget)
    }

    return config, cleanupAction, nil
}
```

### Pod Sandbox Operations
**File**: `pkg/kubelet/kuberuntime/kuberuntime_sandbox.go`

```go
// Line 85 - Create pod sandbox
func (m *kubeGenericRuntimeManager) createPodSandbox(ctx context.Context,
    pod *v1.Pod, attempt uint32) (string, string, error) {

    podSandboxConfig, err := m.generatePodSandboxConfig(pod, attempt)

    // Create pod logs directory
    err = m.osInterface.MkdirAll(podSandboxConfig.LogDirectory, 0755)

    // Call CRI to create sandbox
    podSandboxID, err := m.runtimeService.RunPodSandbox(ctx, podSandboxConfig, runtimeHandler)

    return podSandboxID, "", nil
}

// Line 150 - Generate sandbox config
func (m *kubeGenericRuntimeManager) generatePodSandboxConfig(pod *v1.Pod, attempt uint32) (*runtimeapi.PodSandboxConfig, error) {
    podUID := string(pod.UID)
    podSandboxConfig := &runtimeapi.PodSandboxConfig{
        Metadata: &runtimeapi.PodSandboxMetadata{
            Name:      pod.Name,
            Namespace: pod.Namespace,
            Uid:       podUID,
            Attempt:   attempt,
        },
        Labels:      newPodLabels(pod),
        Annotations: newPodAnnotations(pod),
    }

    // Set DNS config
    dnsConfig, err := m.runtimeHelper.GetPodDNS(pod)
    podSandboxConfig.DnsConfig = dnsConfig

    // Set port mappings
    if !kubecontainer.IsHostNetworkPod(pod) {
        podPortMappings := kubecontainer.MakePortMappings(pod)
        podSandboxConfig.PortMappings = podPortMappings
    }

    // Set Linux-specific config
    if runtime.GOOS == "linux" {
        podSandboxConfig.Linux = m.generatePodSandboxLinuxConfig(pod)
    }

    // Set Windows-specific config
    if runtime.GOOS == "windows" {
        podSandboxConfig.Windows = m.generatePodSandboxWindowsConfig(pod)
    }

    return podSandboxConfig, nil
}
```

## Image Service

### Image Management
**File**: `pkg/kubelet/kuberuntime/kuberuntime_image.go`

```go
// Line 55 - Pull image
func (m *kubeGenericRuntimeManager) PullImage(ctx context.Context,
    image kubecontainer.ImageSpec,
    pullSecrets []v1.Secret,
    podSandboxConfig *runtimeapi.PodSandboxConfig) (string, error) {

    img := image.Image
    authConfig := credentialprovider.LazyProvide(pullSecrets, img)

    imageSpec := &runtimeapi.ImageSpec{
        Image:       img,
        Annotations: image.Annotations,
    }

    imageRef, err := m.imageService.PullImage(ctx, imageSpec, authConfig, podSandboxConfig)

    return imageRef, err
}

// Line 100 - Get image status
func (m *kubeGenericRuntimeManager) GetImageRef(ctx context.Context, image kubecontainer.ImageSpec) (string, error) {
    imageSpec := &runtimeapi.ImageSpec{
        Image:       image.Image,
        Annotations: image.Annotations,
    }

    status, err := m.imageService.ImageStatus(ctx, imageSpec, false)
    if err != nil {
        return "", err
    }

    if status == nil || status.Image == nil {
        return "", fmt.Errorf("image %q not found", image.Image)
    }

    return status.Image.Id, nil
}

// Line 140 - List images
func (m *kubeGenericRuntimeManager) ListImages(ctx context.Context) ([]kubecontainer.Image, error) {
    images, err := m.imageService.ListImages(ctx, nil)
    if err != nil {
        return nil, err
    }

    result := make([]kubecontainer.Image, 0, len(images))
    for _, img := range images {
        result = append(result, kubecontainer.Image{
            ID:       img.Id,
            Size:     int64(img.Size_),
            RepoTags: img.RepoTags,
        })
    }

    return result, nil
}

// Line 180 - Remove image
func (m *kubeGenericRuntimeManager) RemoveImage(ctx context.Context, image kubecontainer.ImageSpec) error {
    imageSpec := &runtimeapi.ImageSpec{
        Image:       image.Image,
        Annotations: image.Annotations,
    }

    return m.imageService.RemoveImage(ctx, imageSpec)
}
```

## Volume Plugin Framework

### Plugin Interface
**File**: `pkg/volume/volume.go`

```go
// Line 85
type VolumePlugin interface {
    Init(host VolumeHost) error
    GetPluginName() string
    GetVolumeName(spec *Spec) (string, error)
    CanSupport(spec *Spec) bool
    RequiresRemount() bool
    NewMounter(spec *Spec, podRef *v1.Pod, opts VolumeOptions) (Mounter, error)
    NewUnmounter(name string, podUID types.UID) (Unmounter, error)
    ConstructVolumeSpec(volumeName, volumePath string) (*Spec, error)
    GetVolumeLimits() (map[string]int, error)
    VolumeLimitAffectsResource(name v1.ResourceName) bool
}

// Line 135 - Mounter interface
type Mounter interface {
    Volume
    SetUp(mounterArgs MounterArgs) error
    SetUpAt(dir string, mounterArgs MounterArgs) error
    GetAttributes() Attributes
}

// Line 155 - Unmounter interface
type Unmounter interface {
    Volume
    TearDown() error
    TearDownAt(dir string) error
}

// Line 175 - Attacher interface
type Attacher interface {
    DeviceMounter
    Attach(spec *Spec, nodeName types.NodeName) (string, error)
    VolumesAreAttached(specs []*Spec, nodeName types.NodeName) (map[*Spec]bool, error)
    WaitForAttach(spec *Spec, devicePath string, pod *v1.Pod, timeout time.Duration) (string, error)
    GetDeviceMountPath(spec *Spec) (string, error)
    MountDevice(spec *Spec, devicePath string, deviceMountPath string) error
}

// Line 195 - Detacher interface
type Detacher interface {
    DeviceUnmounter
    Detach(volumeName string, nodeName types.NodeName) error
    UnmountDevice(deviceMountPath string) error
}
```

### Plugin Manager
**File**: `pkg/volume/plugins.go`

```go
// Line 120
type VolumePluginMgr struct {
    mutex                     sync.RWMutex
    plugins                   map[string]VolumePlugin
    prober                    DynamicPluginProber
    probedPlugins             map[string]VolumePlugin
    host                      VolumeHost
}

// Line 145 - Initialize plugin manager
func NewInitializedVolumePluginMgr(host VolumeHost,
    plugins []VolumePlugin,
    prober DynamicPluginProber) (*VolumePluginMgr, error) {

    vpm := &VolumePluginMgr{
        plugins:       make(map[string]VolumePlugin),
        probedPlugins: make(map[string]VolumePlugin),
        host:          host,
        prober:        prober,
    }

    if err := vpm.InitPlugins(plugins, nil, host); err != nil {
        return nil, err
    }

    return vpm, nil
}

// Line 180 - Find plugin by spec
func (vpm *VolumePluginMgr) FindPluginBySpec(spec *Spec) (VolumePlugin, error) {
    vpm.mutex.Lock()
    defer vpm.mutex.Unlock()

    // Check if spec specifies a plugin
    if spec.Volume != nil {
        for _, plugin := range vpm.plugins {
            if plugin.CanSupport(spec) {
                return plugin, nil
            }
        }
    }

    if spec.PersistentVolume != nil {
        for _, plugin := range vpm.plugins {
            if plugin.CanSupport(spec) {
                return plugin, nil
            }
        }
    }

    return nil, fmt.Errorf("no volume plugin matched")
}

// Line 250 - Find attachable plugin by spec
func (vpm *VolumePluginMgr) FindAttachablePluginBySpec(spec *Spec) (AttachableVolumePlugin, error) {
    volumePlugin, err := vpm.FindPluginBySpec(spec)
    if err != nil {
        return nil, err
    }

    attachableVolumePlugin, ok := volumePlugin.(AttachableVolumePlugin)
    if !ok {
        return nil, fmt.Errorf("plugin %s is not attachable", volumePlugin.GetPluginName())
    }

    return attachableVolumePlugin, nil
}
```

## CSI Plugin

### CSI Plugin Implementation
**File**: `pkg/volume/csi/csi_plugin.go`

```go
// Line 85
type csiPlugin struct {
    host                      volume.VolumeHost
    csiDriverLister           storagelistersv1.CSIDriverLister
    serviceAccountTokenGetter func(namespace, name string) (*v1.Secret, error)
    volumeAttachmentLister    storagelistersv1.VolumeAttachmentLister
}

// Line 120 - Initialize CSI plugin
func (p *csiPlugin) Init(host volume.VolumeHost) error {
    p.host = host

    csiClient := host.GetKubeClient()
    if csiClient == nil {
        return errors.New("csi plugin requires a KubeClient")
    }

    p.csiDriverLister = host.GetCSIDriverLister()
    if p.csiDriverLister == nil {
        return errors.New("csi plugin requires a CSIDriverLister")
    }

    p.volumeAttachmentLister = host.GetVolumeAttachmentLister()

    return nil
}

// Line 180 - Create CSI mounter
func (p *csiPlugin) NewMounter(spec *volume.Spec,
    pod *v1.Pod,
    opts volume.VolumeOptions) (volume.Mounter, error) {

    mounter := &csiMountMgr{
        plugin:       p,
        k8s:          p.host.GetKubeClient(),
        spec:         spec,
        pod:          pod,
        podUID:       pod.UID,
        driverName:   csiDriverName(spec),
        volumeID:     getVolumeID(spec),
        readOnly:     readOnly,
        kubeVolHost:  p.host,
    }

    return mounter, nil
}

// Line 250 - Create CSI unmounter
func (p *csiPlugin) NewUnmounter(volName string, podUID types.UID) (volume.Unmounter, error) {
    return &csiMountMgr{
        plugin:      p,
        podUID:      podUID,
        volName:     volName,
        kubeVolHost: p.host,
    }, nil
}

// Line 280 - Create CSI attacher
func (p *csiPlugin) NewAttacher() (volume.Attacher, error) {
    return &csiAttacher{
        plugin:       p,
        k8s:          p.host.GetKubeClient(),
        waitTimeout:  csiTimeout,
    }, nil
}

// Line 300 - Create CSI detacher
func (p *csiPlugin) NewDetacher() (volume.Detacher, error) {
    return &csiAttacher{
        plugin:       p,
        k8s:          p.host.GetKubeClient(),
        waitTimeout:  csiTimeout,
    }, nil
}
```

### CSI Mount Manager
**File**: `pkg/volume/csi/csi_mounter.go`

```go
// Line 95
type csiMountMgr struct {
    plugin          *csiPlugin
    k8s             clientset.Interface
    spec            *volume.Spec
    pod             *v1.Pod
    podUID          types.UID
    driverName      string
    volumeID        string
    readOnly        bool
    kubeVolHost     volume.KubeletVolumeHost
}

// Line 140 - SetUp CSI volume
func (c *csiMountMgr) SetUp(mounterArgs volume.MounterArgs) error {
    return c.SetUpAt(c.GetPath(), mounterArgs)
}

// Line 150 - SetUpAt CSI volume
func (c *csiMountMgr) SetUpAt(dir string, mounterArgs volume.MounterArgs) error {
    ctx, cancel := context.WithTimeout(context.Background(), csiTimeout)
    defer cancel()

    // Get CSI client for driver
    csi, err := c.csiClientGetter.Get()
    if err != nil {
        return err
    }

    // Node stage volume if required
    err = csi.NodeStageVolume(ctx,
        c.volumeID,
        publishContext,
        stagingTargetPath,
        fsType,
        accessMode,
        nodeStageSecrets,
        volumeContext,
        mountOptions,
    )

    // Node publish volume
    err = csi.NodePublishVolume(ctx,
        c.volumeID,
        readOnly,
        stagingTargetPath,
        targetPath,
        accessMode,
        publishContext,
        volumeContext,
        nodePublishSecrets,
        fsType,
        mountOptions,
    )

    return nil
}

// Line 250 - TearDown CSI volume
func (c *csiMountMgr) TearDown() error {
    return c.TearDownAt(c.GetPath())
}

// Line 260 - TearDownAt CSI volume
func (c *csiMountMgr) TearDownAt(dir string) error {
    ctx, cancel := context.WithTimeout(context.Background(), csiTimeout)
    defer cancel()

    csi, err := c.csiClientGetter.Get()
    if err != nil {
        return err
    }

    // Node unpublish volume
    if err := csi.NodeUnpublishVolume(ctx, c.volumeID, dir); err != nil {
        return err
    }

    // Node unstage volume if required
    if err := csi.NodeUnstageVolume(ctx, c.volumeID, stagingTargetPath); err != nil {
        return err
    }

    return nil
}
```

## Built-in Volume Plugins

### EmptyDir Plugin
**File**: `pkg/volume/emptydir/empty_dir.go`

```go
// Line 85
type emptyDirPlugin struct {
    host volume.VolumeHost
}

// Line 100
func (plugin *emptyDirPlugin) NewMounter(spec *volume.Spec,
    pod *v1.Pod,
    opts volume.VolumeOptions) (volume.Mounter, error) {

    return &emptyDir{
        pod:             pod,
        volName:         spec.Name(),
        medium:          spec.Volume.EmptyDir.Medium,
        sizeLimit:       spec.Volume.EmptyDir.SizeLimit,
        mounter:         plugin.host.GetMounter(plugin.GetPluginName()),
        mountDetector:   mountDetector,
        plugin:          plugin,
        MetricsProvider: volume.NewMetricsStatFS(GetPath(pod.UID, spec.Name(), plugin.host)),
    }, nil
}

// Line 150 - SetUp empty directory
func (ed *emptyDir) SetUp(mounterArgs volume.MounterArgs) error {
    return ed.SetUpAt(ed.GetPath(), mounterArgs)
}

// Line 160
func (ed *emptyDir) SetUpAt(dir string, mounterArgs volume.MounterArgs) error {
    if ed.medium == v1.StorageMediumMemory {
        return ed.setupTmpfs(dir)
    }

    return ed.setupDir(dir)
}

// Line 180
func (ed *emptyDir) setupDir(dir string) error {
    if err := os.MkdirAll(dir, perm); err != nil {
        return err
    }

    // Set ownership
    volume.SetVolumeOwnership(ed, dir, mounterArgs.FsGroup, mounterArgs.FSGroupChangePolicy, completeFunc(logger, err))

    // Set quota if size limit specified
    if ed.sizeLimit != nil && ed.sizeLimit.Value() > 0 {
        err := ed.plugin.host.GetQuotaApplier().SetQuota(dir, ed.sizeLimit.Value())
    }

    return nil
}
```

### ConfigMap Plugin
**File**: `pkg/volume/configmap/configmap.go`

```go
// Line 85
type configMapPlugin struct {
    host volume.VolumeHost
}

// Line 100
func (plugin *configMapPlugin) NewMounter(spec *volume.Spec,
    pod *v1.Pod,
    opts volume.VolumeOptions) (volume.Mounter, error) {

    return &configMapVolume{
        configMapVolume: &configMapVolume{
            spec.Name(),
            pod.UID,
            plugin,
            plugin.host.GetMounter(plugin.GetPluginName()),
            volume.NewMetricsStatFS(getPath(pod.UID, spec.Name(), plugin.host)),
            spec.Volume.ConfigMap,
        },
        pod:         pod,
        getConfigMap: plugin.host.GetConfigMapFunc(),
    }, nil
}

// Line 150 - SetUp ConfigMap volume
func (c *configMapVolume) SetUp(mounterArgs volume.MounterArgs) error {
    return c.SetUpAt(c.GetPath(), mounterArgs)
}

// Line 160
func (c *configMapVolume) SetUpAt(dir string, mounterArgs volume.MounterArgs) error {
    // Get ConfigMap from API server
    configMap, err := c.getConfigMap(c.pod.Namespace, c.source.Name)
    if err != nil {
        return err
    }

    // Create volume directory
    if err := os.MkdirAll(dir, 0755); err != nil {
        return err
    }

    // Write ConfigMap data to files
    payload, err := makePayload(c.source.Items, configMap, c.source.DefaultMode, optional)
    if err != nil {
        return err
    }

    err = writer.Write(payload, dir)
    return err
}
```

### Secret Plugin
**File**: `pkg/volume/secret/secret.go`

```go
// Line 85
type secretPlugin struct {
    host volume.VolumeHost
}

// Line 100
func (plugin *secretPlugin) NewMounter(spec *volume.Spec,
    podRef *v1.Pod,
    opts volume.VolumeOptions) (volume.Mounter, error) {

    return &secretVolume{
        secretVolume: &secretVolume{
            spec.Name(),
            podRef.UID,
            plugin,
            plugin.host.GetMounter(plugin.GetPluginName()),
            volume.NewMetricsStatFS(getPath(podRef.UID, spec.Name(), plugin.host)),
            spec.Volume.Secret,
        },
        pod:       podRef,
        getSecret: plugin.host.GetSecretFunc(),
    }, nil
}

// Line 150 - SetUp Secret volume
func (s *secretVolume) SetUp(mounterArgs volume.MounterArgs) error {
    return s.SetUpAt(s.GetPath(), mounterArgs)
}

// Line 160
func (s *secretVolume) SetUpAt(dir string, mounterArgs volume.MounterArgs) error {
    // Get Secret from API server
    secret, err := s.getSecret(s.pod.Namespace, s.source.SecretName)
    if err != nil {
        return err
    }

    // Create volume directory
    if err := os.MkdirAll(dir, 0755); err != nil {
        return err
    }

    // Write Secret data to files
    payload, err := makePayload(s.source.Items, secret, s.source.DefaultMode, optional)
    if err != nil {
        return err
    }

    err = writer.Write(payload, dir)
    return err
}
```

## Volume Operations

### Volume Manager Operations
**File**: `pkg/kubelet/volumemanager/volume_manager.go`

```go
// Line 280 - Wait for volumes to attach
func (vm *volumeManager) WaitForAttachAndMount(pod *v1.Pod) error {
    expectedVolumes := getExpectedVolumes(pod)

    err := wait.PollImmediate(
        podAttachAndMountRetryInterval,
        podAttachAndMountTimeout,
        func() (bool, error) {
            mounted := vm.actualStateOfWorld.GetMountedVolumesForPod(UniquePodName(pod))

            for _, expectedVolume := range expectedVolumes {
                if _, ok := mounted[expectedVolume]; !ok {
                    // Volume not mounted yet
                    return false, nil
                }
            }

            // All volumes mounted
            return true, nil
        })

    return err
}

// Line 350 - Get mounted volumes for pod
func (vm *volumeManager) GetMountedVolumesForPod(podName types.UniquePodName) container.VolumeMap {
    volumeMap := make(container.VolumeMap)

    for _, mountedVolume := range vm.actualStateOfWorld.GetMountedVolumesForPod(podName) {
        volumeMap[mountedVolume.OuterVolumeSpecName] = container.VolumeInfo{
            Mounter:           mountedVolume.Mounter,
            BlockVolumeMapper: mountedVolume.BlockVolumeMapper,
            ReadOnly:          mountedVolume.VolumeSpec.ReadOnly,
            InnerVolumeSpecName: mountedVolume.InnerVolumeSpecName,
        }
    }

    return volumeMap
}
```

### Operation Executor
**File**: `pkg/volume/util/operationexecutor/operation_executor.go`

```go
// Line 180 - Mount volume operation
func (oe *operationExecutor) MountVolume(
    waitForAttachTimeout time.Duration,
    volumeToMount VolumeToMount,
    actualStateOfWorld ActualStateOfWorldMounterUpdater,
    isRemount bool) error {

    generatedFunc := func() (interface{}, error) {
        // Get mounter plugin
        volumePlugin, err := og.volumePluginMgr.FindPluginBySpec(volumeToMount.VolumeSpec)

        // Create volume mounter
        volumeMounter, err := volumePlugin.NewMounter(
            volumeToMount.VolumeSpec,
            volumeToMount.Pod,
            volume.VolumeOptions{})

        // Execute mount
        mountErr := volumeMounter.SetUp(volume.MounterArgs{
            FsUser:              fsUser,
            FsGroup:             fsGroup,
            FSGroupChangePolicy: fsGroupChangePolicy,
            DesiredSize:         volumeToMount.DesiredSizeLimit,
        })

        // Update actual state of world
        if mountErr == nil {
            actualStateOfWorld.MarkVolumeAsMounted(
                volumeToMount.PodName,
                volumeToMount.VolumeName,
                volumeMounter,
                volumeToMount.OuterVolumeSpecName,
                volumeToMount.VolumeSpec,
                volumeToMount.DesiredSizeLimit)
        }

        return nil, mountErr
    }

    return oe.pendingOperations.Run(
        volumeToMount.VolumeName, podName, generatedFunc)
}

// Line 280 - Unmount volume operation
func (oe *operationExecutor) UnmountVolume(
    volumeToUnmount MountedVolume,
    actualStateOfWorld ActualStateOfWorldMounterUpdater,
    podsDir string) error {

    generatedFunc := func() (interface{}, error) {
        // Get unmounter plugin
        volumePlugin, err := og.volumePluginMgr.FindPluginByName(volumeToUnmount.PluginName)

        // Create volume unmounter
        volumeUnmounter, err := volumePlugin.NewUnmounter(
            volumeToUnmount.InnerVolumeSpecName,
            volumeToUnmount.PodUID)

        // Execute unmount
        unmountErr := volumeUnmounter.TearDown()

        // Update actual state of world
        if unmountErr == nil {
            actualStateOfWorld.MarkVolumeAsUnmounted(
                volumeToUnmount.PodName,
                volumeToUnmount.VolumeName)
        }

        return nil, unmountErr
    }

    return oe.pendingOperations.Run(
        volumeToUnmount.VolumeName, "", generatedFunc)
}
```

## Mount Utilities

### Mount Interface
**File**: `pkg/util/mount/mount.go`

```go
// Line 55
type Interface interface {
    Mount(source string, target string, fstype string, options []string) error
    MountSensitive(source string, target string, fstype string, options []string, sensitiveOptions []string) error
    Unmount(target string) error
    List() ([]MountPoint, error)
    IsLikelyNotMountPoint(file string) (bool, error)
    GetMountRefs(pathname string) ([]string, error)
}

// Line 85
type MountPoint struct {
    Device string
    Path   string
    Type   string
    Opts   []string
    Freq   int
    Pass   int
}
```

### Mount Implementation
**File**: `pkg/util/mount/mount_linux.go`

```go
// Line 120 - Linux mount implementation
func (mounter *Mounter) Mount(source string, target string, fstype string, options []string) error {
    bind, bindOpts, bindRemountOpts, bindRemountOptsSensitive := MakeBindOpts(options, sensitiveOptions)

    if bind {
        err := mounter.doMount(mounterPath, defaultMountCommand, source, target, fstype, bindOpts, bindRemountOptsSensitive, nil /* mountFlags */, true)
        if err != nil {
            return err
        }

        if len(bindRemountOpts) > 0 {
            err = mounter.doMount(mounterPath, defaultMountCommand, source, target, fstype, bindRemountOpts, bindRemountOptsSensitive, nil /* mountFlags */, true)
        }

        return err
    }

    return mounter.doMount(mounterPath, defaultMountCommand, source, target, fstype, options, sensitiveOptions, nil /* mountFlags */, true)
}

// Line 180 - Execute mount syscall
func (mounter *Mounter) doMount(mounterPath string, mountCmd string, source string, target string, fstype string, options []string, sensitiveOptions []string, mountFlags []string, systemdMountRequired bool) error {
    if systemdMountRequired {
        return mounter.mountWithSystemd(source, target, fstype, options, sensitiveOptions)
    }

    mountArgs, mountArgsLogStr := MakeMountArgs(source, target, fstype, options, sensitiveOptions)
    command := exec.Command(mountCmd, mountArgs...)

    output, err := command.CombinedOutput()
    if err != nil {
        return fmt.Errorf("mount failed: %v\nMounting command: %s\nOutput: %s", err, mountArgsLogStr, string(output))
    }

    return nil
}

// Line 250 - Unmount
func (mounter *Mounter) Unmount(target string) error {
    command := exec.Command("umount", target)
    output, err := command.CombinedOutput()
    if err != nil {
        return fmt.Errorf("unmount failed: %v\nOutput: %s", err, string(output))
    }

    return nil
}
```

## Quick Reference Tables

### CRI Client Functions

| Function | File | Line | Purpose |
|----------|------|------|---------|
| `NewRemoteRuntimeService()` | `cri/remote/remote_runtime.go` | 120 | Create runtime client |
| `RunPodSandbox()` | `cri/remote/remote_runtime.go` | 200 | Create pod sandbox |
| `CreateContainer()` | `cri/remote/remote_runtime.go` | 250 | Create container |
| `StartContainer()` | `cri/remote/remote_runtime.go` | 300 | Start container |
| `StopContainer()` | `cri/remote/remote_runtime.go` | 350 | Stop container |
| `RemoveContainer()` | `cri/remote/remote_runtime.go` | 400 | Remove container |
| `PullImage()` | `cri/remote/remote_image.go` | 120 | Pull image |
| `ListImages()` | `cri/remote/remote_image.go` | 180 | List images |

### Runtime Manager Functions

| Function | File | Line | Purpose |
|----------|------|------|---------|
| `NewKubeGenericRuntimeManager()` | `kuberuntime/kuberuntime_manager.go` | 200 | Create runtime manager |
| `SyncPod()` | `kuberuntime/kuberuntime_manager.go` | 350 | Synchronize pod state |
| `startContainer()` | `kuberuntime/kuberuntime_container.go` | 150 | Start container |
| `generateContainerConfig()` | `kuberuntime/kuberuntime_container.go` | 350 | Generate container config |
| `createPodSandbox()` | `kuberuntime/kuberuntime_sandbox.go` | 85 | Create pod sandbox |
| `PullImage()` | `kuberuntime/kuberuntime_image.go` | 55 | Pull image |

### Volume Plugin Types

| Plugin | File | Interface | Purpose |
|--------|------|-----------|---------|
| CSI | `volume/csi/csi_plugin.go` | `VolumePlugin` | Container Storage Interface |
| EmptyDir | `volume/emptydir/empty_dir.go` | `VolumePlugin` | Temporary directory |
| HostPath | `volume/hostpath/host_path.go` | `VolumePlugin` | Host filesystem |
| ConfigMap | `volume/configmap/configmap.go` | `VolumePlugin` | ConfigMap volumes |
| Secret | `volume/secret/secret.go` | `VolumePlugin` | Secret volumes |
| PVC | `volume/persistentvolumeclaim/pvc.go` | `VolumePlugin` | Persistent volumes |

### Volume Operations

| Operation | File | Function | Line |
|-----------|------|----------|------|
| Mount | `util/operationexecutor/operation_executor.go` | `MountVolume()` | 180 |
| Unmount | `util/operationexecutor/operation_executor.go` | `UnmountVolume()` | 280 |
| Attach | `util/operationexecutor/operation_executor.go` | `AttachVolume()` | 380 |
| Detach | `util/operationexecutor/operation_executor.go` | `DetachVolume()` | 450 |
| Expand | `util/operationexecutor/operation_executor.go` | `ExpandVolume()` | 520 |

### CRI gRPC Services

| Service | Proto File | Methods |
|---------|------------|---------|
| RuntimeService | `staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto` | RunPodSandbox, CreateContainer, StartContainer, StopContainer, RemoveContainer |
| ImageService | `staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto` | PullImage, ListImages, ImageStatus, RemoveImage |

## Summary

This reference provides comprehensive navigation for CRI and volume implementations:

### Navigation Guide

1. **CRI Client**: Start at `pkg/kubelet/cri/remote/` for container runtime integration
2. **Runtime Manager**: Look at `pkg/kubelet/kuberuntime/` for pod lifecycle management
3. **Volume Framework**: Check `pkg/volume/` for plugin interfaces
4. **CSI Implementation**: Navigate to `pkg/volume/csi/` for CSI support
5. **Built-in Plugins**: Find plugins in `pkg/volume/*/` directories

### Key Design Patterns

- **gRPC Communication**: CRI uses gRPC for runtime communication
- **Plugin Architecture**: Volume system supports multiple plugin types
- **Operation Executor**: Centralized execution of volume operations
- **State Management**: Actual vs desired state reconciliation
- **Mount Utilities**: Platform-specific mount implementations

Use the quick reference tables to jump directly to specific implementations and functions.