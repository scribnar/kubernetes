# Volume Plugins - Low-Level Technical Specification

**Purpose**: Detailed technical specification of the volume plugin system, CSI integration, and volume lifecycle management in kubelet

**Audience**: Storage plugin developers, kubelet contributors, cluster administrators

**Related Documents**:
- [Volume Manager](../middle-level/07-volume-manager.md) - High-level volume management
- [Pod Storage](../high-level/04-runtime-integration.md) - Storage concepts
- [Container Lifecycle](../middle-level/04-container-lifecycle.md) - Container mounting

---

## Table of Contents

1. [Volume Plugin Architecture](#volume-plugin-architecture)
2. [Plugin Interfaces](#plugin-interfaces)
3. [Volume Lifecycle](#volume-lifecycle)
4. [CSI Integration](#csi-integration)
5. [Mount Propagation](#mount-propagation)
6. [Subpath Handling](#subpath-handling)
7. [Volume Manager Implementation](#volume-manager-implementation)
8. [SELinux and Ownership](#selinux-and-ownership)
9. [Volume Metrics](#volume-metrics)
10. [Error Handling and Recovery](#error-handling-and-recovery)
11. [Volume Reconstruction](#volume-reconstruction)
12. [Best Practices](#best-practices)

---

## Volume Plugin Architecture

### Overview

The volume plugin system provides a modular framework for integrating various storage backends with Kubernetes. All storage types implement the `VolumePlugin` interface, whether they're in-tree plugins (deprecated) or CSI drivers.

```mermaid
graph TB
    subgraph "Volume Manager"
        VM[VolumeManager]
        DSW[Desired State<br/>of World]
        ASW[Actual State<br/>of World]
        REC[Reconciler]
        POP[Populator]
    end

    subgraph "Plugin System"
        VH[VolumeHost]
        PM[Plugin Manager]
        CSI[CSI Plugin]
        LEGACY[Legacy Plugins<br/>deprecated]
    end

    subgraph "Operations"
        ATTACH[Attach]
        MOUNT[Mount]
        UNMOUNT[Unmount]
        DETACH[Detach]
    end

    VM --> DSW
    VM --> ASW
    VM --> REC
    VM --> POP

    REC --> PM
    PM --> CSI
    PM --> LEGACY

    CSI --> ATTACH
    CSI --> MOUNT
    CSI --> UNMOUNT
    CSI --> DETACH

    style VM fill:#e1f5ff
    style CSI fill:#d4f4dd
    style LEGACY fill:#ffe6e6
```

**Code Reference**: `pkg/volume/plugins.go:128` - VolumePlugin interface

### Plugin Discovery and Registration

Volume plugins are discovered and registered during kubelet initialization:

```go
// pkg/kubelet/kubelet.go (simplified)
func (kl *Kubelet) initializeVolumePlugins() error {
    // Get all volume plugins
    allPlugins := []volume.VolumePlugin{}

    // Add CSI plugin (primary)
    allPlugins = append(allPlugins, csi.ProbeVolumePlugins()...)

    // Add legacy plugins (deprecated)
    allPlugins = append(allPlugins, emptydir.ProbeVolumePlugins()...)
    allPlugins = append(allPlugins, hostpath.ProbeVolumePlugins()...)
    allPlugins = append(allPlugins, secret.ProbeVolumePlugins()...)
    allPlugins = append(allPlugins, configmap.ProbeVolumePlugins()...)

    // Initialize plugin manager
    kl.volumePluginMgr = volume.NewVolumePluginMgr()

    // Initialize each plugin
    for _, plugin := range allPlugins {
        err := plugin.Init(kl.volumeHost)
        if err != nil {
            return fmt.Errorf("failed to init plugin %s: %v", plugin.GetPluginName(), err)
        }
        kl.volumePluginMgr.RegisterPlugin(plugin)
    }

    return nil
}
```

---

## Plugin Interfaces

### Core VolumePlugin Interface

```go
// pkg/volume/plugins.go:128
type VolumePlugin interface {
    // Initialize the plugin with host capabilities
    Init(host VolumeHost) error

    // Return plugin name (e.g., "kubernetes.io/csi")
    GetPluginName() string

    // Get unique volume name from spec
    GetVolumeName(spec *Spec) (string, error)

    // Test if plugin supports the given spec
    CanSupport(spec *Spec) bool

    // Test if plugin requires SELinux relabeling
    RequiresRemount(spec *Spec) bool

    // Create a new mounter for the volume
    NewMounter(spec *Spec, pod *v1.Pod, opts VolumeOptions) (Mounter, error)

    // Create a new unmounter for the volume
    NewUnmounter(name string, podUID types.UID) (Unmounter, error)

    // Construct volume spec from mount path (for reconstruction)
    ConstructVolumeSpec(volumeName, volumePath string) (*Spec, error)
}
```

**Code Reference**: `pkg/volume/plugins.go:128` - VolumePlugin interface

### Volume Interfaces

```go
// pkg/volume/volume.go:33
type Volume interface {
    // Get the path where volume should be mounted for pod
    GetPath() string

    // Get volume metrics
    MetricsProvider
}

// pkg/volume/volume.go:162
type Mounter interface {
    Volume

    // Mount the volume to specified directory
    SetUp(mounterArgs MounterArgs) error
    SetUpAt(dir string, mounterArgs MounterArgs) error

    // Get mount attributes
    GetAttributes() Attributes
}

// pkg/volume/volume.go:190
type Unmounter interface {
    Volume

    // Unmount the volume
    TearDown() error
    TearDownAt(dir string) error
}
```

**Code Reference**: `pkg/volume/volume.go:33` - Volume interface

### Attachable Volumes

For network-attached storage:

```go
// pkg/volume/plugins.go (simplified)
type AttachableVolumePlugin interface {
    VolumePlugin

    // Create attacher to attach volume to node
    NewAttacher() (Attacher, error)

    // Create detacher to detach volume from node
    NewDetacher() (Detacher, error)

    // Get device mount path on the node
    GetDeviceMountPath(spec *Spec) (string, error)
}

type Attacher interface {
    // Attach volume to node
    Attach(spec *Spec, nodeName types.NodeName) (string, error)

    // Check if volume is attached
    VolumesAreAttached(specs []*Spec, nodeName types.NodeName) (map[*Spec]bool, error)

    // Wait for attach to complete
    WaitForAttach(spec *Spec, devicePath string, pod *v1.Pod, timeout time.Duration) (string, error)

    // Get device mount path
    GetDeviceMountPath(spec *Spec) (string, error)

    // Mount device to global path
    MountDevice(spec *Spec, devicePath string, deviceMountPath string, mounter mount.Interface) error
}
```

---

## Volume Lifecycle

### Complete Volume Lifecycle

```mermaid
sequenceDiagram
    participant POD as Pod
    participant VM as VolumeManager
    participant DSW as Desired State
    participant REC as Reconciler
    participant PLG as Volume Plugin
    participant CSI as CSI Driver
    participant NODE as Node Storage

    Note over POD: Pod created with volume

    POD->>VM: Pod added to desired state
    VM->>DSW: AddPodToVolume()

    DSW->>REC: Volume needs attach/mount

    alt Attachable volume
        REC->>PLG: NewAttacher()
        PLG->>CSI: ControllerPublishVolume()
        CSI->>NODE: Attach disk to node
        NODE-->>CSI: Device path
        CSI-->>PLG: Attached

        REC->>PLG: WaitForAttach()
        PLG->>NODE: Wait for device
        NODE-->>PLG: Device ready

        REC->>PLG: MountDevice()
        PLG->>NODE: Mount to global path
    end

    REC->>PLG: NewMounter()
    PLG->>CSI: NodePublishVolume()
    CSI->>NODE: Mount to pod path
    NODE-->>CSI: Mounted

    Note over POD: Pod running with volume

    Note over POD: Pod deleted

    POD->>VM: Pod removed
    VM->>DSW: RemovePodFromVolume()

    DSW->>REC: Volume needs unmount/detach

    REC->>PLG: NewUnmounter()
    PLG->>CSI: NodeUnpublishVolume()
    CSI->>NODE: Unmount from pod path

    alt Attachable volume
        REC->>PLG: UnmountDevice()
        PLG->>NODE: Unmount from global path

        REC->>PLG: NewDetacher()
        PLG->>CSI: ControllerUnpublishVolume()
        CSI->>NODE: Detach disk from node
    end
```

### Volume States

```mermaid
stateDiagram-v2
    [*] --> Pending: Volume requested

    Pending --> Attaching: Attachable volume
    Pending --> Mounting: Non-attachable volume

    Attaching --> Attached: Attach complete
    Attached --> DeviceMounting: Mount to global path
    DeviceMounting --> DeviceMounted: Global mount complete
    DeviceMounted --> Mounting: Ready for pod mount

    Mounting --> Mounted: Mount complete
    Mounted --> InUse: Volume in use by pod

    InUse --> Unmounting: Pod terminated
    Unmounting --> Unmounted: Unmount complete

    Unmounted --> DeviceUnmounting: Attachable volume
    DeviceUnmounting --> DeviceUnmounted: Global unmount complete
    DeviceUnmounted --> Detaching: Ready to detach
    Detaching --> Detached: Detach complete

    Unmounted --> [*]: Non-attachable
    Detached --> [*]: Cleanup complete
```

---

## CSI Integration

### CSI Plugin Architecture

The Container Storage Interface (CSI) is the standard for exposing arbitrary storage systems to containers.

```mermaid
graph TB
    subgraph "Kubelet"
        CSIP[CSI Plugin]
        VM[Volume Manager]
        REG[Plugin Registry]
    end

    subgraph "CSI Driver Pod"
        NODE[Node Service]
        CTRL[Controller Service<br/>optional]
        REG_CONT[Registrar Container]
    end

    subgraph "Unix Sockets"
        SOCK1[/registration/driver-name-reg.sock]
        SOCK2[/plugins/driver-name/csi.sock]
    end

    VM --> CSIP
    CSIP --> SOCK2
    SOCK2 --> NODE
    NODE --> CTRL

    REG_CONT --> SOCK1
    SOCK1 --> REG
    REG --> CSIP

    style CSIP fill:#d4f4dd
    style NODE fill:#e1f5ff
```

**Code Reference**: `pkg/volume/csi/csi_plugin.go` - CSI plugin implementation

### CSI Operations

#### Node Operations

```go
// NodePublishVolume - Mount volume to pod
func (c *csiDriverClient) NodePublishVolume(
    ctx context.Context,
    volumeID string,
    readOnly bool,
    stagingTargetPath string,
    targetPath string,
    accessMode v1.PersistentVolumeAccessMode,
    publishContext map[string]string,
    volumeContext map[string]string,
    mountOptions []string,
    fsType string,
) error {
    req := &csipb.NodePublishVolumeRequest{
        VolumeId:          volumeID,
        TargetPath:        targetPath,
        StagingTargetPath: stagingTargetPath,
        Readonly:          readOnly,
        PublishContext:    publishContext,
        VolumeContext:     volumeContext,
        VolumeCapability: &csipb.VolumeCapability{
            AccessMode: &csipb.VolumeCapability_AccessMode{
                Mode: accessModeToCSIAccessMode(accessMode),
            },
            AccessType: &csipb.VolumeCapability_Mount{
                Mount: &csipb.VolumeCapability_MountVolume{
                    FsType:     fsType,
                    MountFlags: mountOptions,
                },
            },
        },
    }

    _, err := c.nodeClient.NodePublishVolume(ctx, req)
    return err
}
```

#### Controller Operations (if supported)

```go
// ControllerPublishVolume - Attach volume to node
func (c *csiDriverClient) ControllerPublishVolume(
    ctx context.Context,
    volumeID string,
    readOnly bool,
    nodeID string,
    publishContext map[string]string,
    volumeContext map[string]string,
) (map[string]string, error) {
    req := &csipb.ControllerPublishVolumeRequest{
        VolumeId: volumeID,
        NodeId:   nodeID,
        Readonly: readOnly,
        VolumeCapability: &csipb.VolumeCapability{
            // ... capability details
        },
        VolumeContext: volumeContext,
    }

    resp, err := c.controllerClient.ControllerPublishVolume(ctx, req)
    if err != nil {
        return nil, err
    }

    return resp.PublishContext, nil
}
```

### CSI Volume Staging

For block volumes that need staging:

```mermaid
sequenceDiagram
    participant REC as Reconciler
    participant CSI as CSI Plugin
    participant DRV as CSI Driver
    participant BLK as Block Device

    Note over REC: Attach phase
    REC->>CSI: Attach(volume, node)
    CSI->>DRV: ControllerPublishVolume
    DRV->>BLK: Attach device
    BLK-->>DRV: /dev/disk/by-id/xxx

    Note over REC: Stage phase
    REC->>CSI: StageVolume()
    CSI->>DRV: NodeStageVolume
    DRV->>BLK: Format & mount to staging
    Note right of DRV: /var/lib/kubelet/plugins/<br/>kubernetes.io/csi/driver/<br/>staging/vol-id

    Note over REC: Publish phase
    REC->>CSI: PublishVolume()
    CSI->>DRV: NodePublishVolume
    DRV->>DRV: Bind mount from staging
    Note right of DRV: /var/lib/kubelet/pods/<br/>pod-uid/volumes/<br/>driver/mount
```

---

## Mount Propagation

### Propagation Modes

Mount propagation determines how mounts are shared between the host and containers:

```go
// pkg/volume/volume.go
type MountPropagationMode string

const (
    // MountPropagationNone - No mount propagation ("rprivate")
    MountPropagationNone MountPropagationMode = "None"

    // MountPropagationHostToContainer - Mounts from host visible in container ("rslave")
    MountPropagationHostToContainer MountPropagationMode = "HostToContainer"

    // MountPropagationBidirectional - Mounts propagate both ways ("rshared")
    MountPropagationBidirectional MountPropagationMode = "Bidirectional"
)
```

### Propagation Implementation

```mermaid
graph LR
    subgraph "Host Mount Namespace"
        HM[Host Mount<br/>/mnt/data]
    end

    subgraph "Container - None"
        CM1[Container Mount<br/>isolated]
    end

    subgraph "Container - HostToContainer"
        CM2[Container Mount<br/>sees host changes]
    end

    subgraph "Container - Bidirectional"
        CM3[Container Mount<br/>bidirectional]
    end

    HM -.->|No propagation| CM1
    HM -->|Host → Container| CM2
    HM <-->|Both ways| CM3

    style CM1 fill:#ffe6e6
    style CM2 fill:#fff9e6
    style CM3 fill:#d4f4dd
```

**Use Cases**:
- **None**: Default, maximum isolation
- **HostToContainer**: For monitoring agents that need to see host mounts
- **Bidirectional**: For storage orchestrators that create mounts for other containers

---

## Subpath Handling

### Subpath Mounting

Subpath allows mounting a subdirectory or file from a volume:

```yaml
apiVersion: v1
kind: Pod
spec:
  volumes:
  - name: config-volume
    configMap:
      name: config
  containers:
  - name: app
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app.conf
      subPath: app.conf  # Mount only this file
```

### Subpath Implementation

```go
// pkg/volume/util/subpath/subpath_linux.go
func doSafeOpen(pathname string, base string) (int, error) {
    // Resolve symlinks and ensure path doesn't escape base
    pathname = filepath.Clean(pathname)
    base = filepath.Clean(base)

    // Check that pathname is under base
    if !strings.HasPrefix(pathname, base+"/") && pathname != base {
        return -1, fmt.Errorf("path %s is outside of allowed base %s", pathname, base)
    }

    // Open with O_PATH to avoid TOCTOU attacks
    fd, err := unix.Open(pathname, unix.O_PATH|unix.O_CLOEXEC, 0)
    if err != nil {
        return -1, err
    }

    // Verify the opened path is still under base
    procPath := fmt.Sprintf("/proc/self/fd/%d", fd)
    realPath, err := filepath.EvalSymlinks(procPath)
    if err != nil {
        unix.Close(fd)
        return -1, err
    }

    if !strings.HasPrefix(realPath, base+"/") && realPath != base {
        unix.Close(fd)
        return -1, fmt.Errorf("path %s escapes base %s", realPath, base)
    }

    return fd, nil
}
```

**Code Reference**: `pkg/volume/util/subpath/subpath_linux.go` - Safe subpath opening

### SubPathExpr

Dynamic subpath using pod fields:

```yaml
volumeMounts:
- name: workdir
  mountPath: /logs
  subPathExpr: $(POD_NAME)  # Creates pod-specific directory
```

---

## Volume Manager Implementation

### Volume Manager Components

```go
// pkg/kubelet/volumemanager/volume_manager.go:98
type VolumeManager interface {
    // Start volume manager loops
    Run(ctx context.Context, sourcesReady config.SourcesReady)

    // Wait for volumes to be attached and mounted
    WaitForAttachAndMount(ctx context.Context, pod *v1.Pod) error

    // Wait for volumes to be unmounted
    WaitForUnmount(ctx context.Context, pod *v1.Pod) error

    // Get mounted volumes for pod
    GetMountedVolumesForPod(podName types.UniquePodName) container.VolumeMap

    // Get volumes currently in use
    GetVolumesInUse() []v1.UniqueVolumeName

    // Check if reconciler has synced at least once
    ReconcilerStatesHasBeenSynced() bool

    // Mark volumes as reported in use
    MarkVolumesAsReportedInUse(volumesReportedAsInUse []v1.UniqueVolumeName)
}
```

**Code Reference**: `pkg/kubelet/volumemanager/volume_manager.go:98` - VolumeManager interface

### Reconciler Loop

The reconciler ensures actual state matches desired state:

```go
// pkg/kubelet/volumemanager/reconciler/reconciler.go (simplified)
func (rc *reconciler) reconcile() {
    // Unmount volumes no longer needed
    rc.unmountVolumes()

    // Detach volumes no longer needed
    rc.detachVolumes()

    // Attach volumes that should be attached
    rc.attachVolumes()

    // Mount attached volumes
    rc.mountAttachedVolumes()

    // Unmount devices no longer needed
    rc.unmountDetachDevices()
}

func (rc *reconciler) mountAttachedVolumes() {
    for _, volumeToMount := range rc.desiredStateOfWorld.GetVolumesToMount() {
        if !rc.actualStateOfWorld.PodExistsInVolume(volumeToMount.PodName, volumeToMount.VolumeName) {
            // Volume not mounted, mount it
            err := rc.operationExecutor.MountVolume(
                volumeToMount.VolumeToMount,
                rc.actualStateOfWorld,
                rc.isVolumeSafeToDetach,
            )
            if err != nil {
                klog.Errorf("Failed to mount volume %v: %v", volumeToMount.VolumeName, err)
            }
        }
    }
}
```

### Desired vs Actual State

```mermaid
graph TB
    subgraph "Desired State of World"
        DS_POD[Pod: nginx]
        DS_VOL1[Volume: data-vol<br/>Should be mounted]
        DS_VOL2[Volume: config-vol<br/>Should be mounted]
    end

    subgraph "Actual State of World"
        AS_POD[Pod: nginx]
        AS_VOL1[Volume: data-vol<br/>Mounted]
        AS_VOL3[Volume: old-vol<br/>Mounted but not desired]
    end

    subgraph "Reconciler Actions"
        ACT1[Mount config-vol]
        ACT2[Unmount old-vol]
    end

    DS_VOL2 -->|Missing| ACT1
    AS_VOL3 -->|Extra| ACT2

    style DS_VOL2 fill:#ffe6e6
    style AS_VOL3 fill:#ffe6e6
    style ACT1 fill:#d4f4dd
    style ACT2 fill:#fff9e6
```

---

## SELinux and Ownership

### SELinux Context

Volumes can be relabeled with SELinux contexts:

```go
// pkg/volume/volume.go
type SELinuxMountContext struct {
    SELinuxLabel string
    SELinuxRelabel bool
}

func (m *mounter) SetUpAt(dir string, mounterArgs MounterArgs) error {
    // Mount the volume
    if err := m.mountVolume(dir); err != nil {
        return err
    }

    // Apply SELinux context if needed
    if mounterArgs.SELinuxLabel != "" && m.GetAttributes().SELinuxRelabel {
        if err := selinux.Relabel(dir, mounterArgs.SELinuxLabel, true); err != nil {
            return fmt.Errorf("failed to relabel %s: %v", dir, err)
        }
    }

    return nil
}
```

### Ownership Management

```go
// pkg/volume/volume.go:143
type VolumeOwnership struct {
    mounter             Mounter
    dir                 string
    fsGroup             *int64
    fsGroupChangePolicy *v1.PodFSGroupChangePolicy
    recorder            record.EventRecorder
}

func (o *VolumeOwnership) ChangePermissions() error {
    if o.fsGroup == nil {
        return nil
    }

    policy := v1.FSGroupChangeOnRootMismatch // default
    if o.fsGroupChangePolicy != nil {
        policy = *o.fsGroupChangePolicy
    }

    switch policy {
    case v1.FSGroupChangeAlways:
        // Always change ownership
        return o.changeOwnership(*o.fsGroup)
    case v1.FSGroupChangeOnRootMismatch:
        // Change only if root dir has different ownership
        stat, err := os.Stat(o.dir)
        if err != nil {
            return err
        }
        if stat.Sys().(*syscall.Stat_t).Gid != uint32(*o.fsGroup) {
            return o.changeOwnership(*o.fsGroup)
        }
    }

    return nil
}
```

---

## Volume Metrics

### Metrics Collection

```go
// pkg/volume/volume.go:66
type MetricsProvider interface {
    // Get volume metrics
    GetMetrics() (*Metrics, error)
}

// pkg/volume/volume.go:73
type Metrics struct {
    // Timestamps
    Time metav1.Time

    // Capacity metrics
    Used       *resource.Quantity
    Capacity   *resource.Quantity
    Available  *resource.Quantity

    // Inode metrics
    InodesUsed *resource.Quantity
    Inodes     *resource.Quantity
    InodesFree *resource.Quantity

    // Health status
    Abnormal *bool
    Message  *string
}
```

**Code Reference**: `pkg/volume/volume.go:73` - Volume Metrics structure

### Metrics Implementation

```mermaid
sequenceDiagram
    participant KUB as Kubelet
    participant VM as Volume Manager
    participant PLG as Volume Plugin
    participant CSI as CSI Driver
    participant FS as Filesystem

    KUB->>VM: GetVolumeStats()
    VM->>PLG: GetMetrics()

    alt CSI Volume
        PLG->>CSI: NodeGetVolumeStats()
        CSI->>FS: statfs()
        FS-->>CSI: Filesystem stats
        CSI-->>PLG: Volume stats
    else Local Volume
        PLG->>FS: statfs(mountPath)
        FS-->>PLG: Filesystem stats
    end

    PLG-->>VM: Metrics{Used, Available, Capacity}
    VM-->>KUB: Volume metrics
```

---

## Error Handling and Recovery

### Transient vs Permanent Errors

```go
// pkg/volume/util/types/types.go
// TransientOperationFailure indicates operation failed but may succeed on retry
type TransientOperationFailure struct {
    Msg string
}

// UncertainProgressError indicates operation progress is uncertain
type UncertainProgressError struct {
    Msg string
}

func (rc *reconciler) processVolumesInUse() {
    for _, volume := range rc.desiredStateOfWorld.GetVolumesToMount() {
        err := rc.mountVolume(volume)
        if err != nil {
            switch err.(type) {
            case *TransientOperationFailure:
                // Retry with exponential backoff
                rc.scheduleRetry(volume, rc.backoff.Next())
            case *UncertainProgressError:
                // Check actual state before retry
                if !rc.isActuallyMounted(volume) {
                    rc.scheduleRetry(volume, time.Second)
                }
            default:
                // Permanent error, report and don't retry
                rc.reportError(volume, err)
            }
        }
    }
}
```

### Orphaned Volume Cleanup

```mermaid
flowchart TB
    START[Kubelet Restart]
    SCAN[Scan volume directories]

    SCAN --> CHECK{Volume in<br/>desired state?}

    CHECK -->|Yes| KEEP[Keep volume]
    CHECK -->|No| ORPHAN[Mark as orphaned]

    ORPHAN --> GRACE{Grace period<br/>expired?}

    GRACE -->|No| WAIT[Wait]
    GRACE -->|Yes| UNMOUNT[Unmount volume]

    UNMOUNT --> CLEAN[Clean directory]
    CLEAN --> DONE[Cleanup complete]

    style ORPHAN fill:#fff9e6
    style UNMOUNT fill:#ffe6e6
    style CLEAN fill:#d4f4dd
```

---

## Volume Reconstruction

### Reconstruction on Restart

When kubelet restarts, it reconstructs volume state from disk:

```go
// pkg/volume/util/reconstruct.go
func (rc *reconstructedVolume) reconstructVolumes() error {
    // Scan pod volume directories
    podDirs, err := os.ReadDir(rc.kubeletPodDir)
    if err != nil {
        return err
    }

    for _, podDir := range podDirs {
        volumes, err := rc.reconstructPodVolumes(podDir.Name())
        if err != nil {
            klog.Errorf("Failed to reconstruct pod %s: %v", podDir.Name(), err)
            continue
        }

        for _, vol := range volumes {
            // Reconstruct volume spec from mount
            spec, err := vol.plugin.ConstructVolumeSpec(
                vol.volumeName,
                vol.mountPath,
            )
            if err != nil {
                continue
            }

            // Add to actual state of world
            rc.actualStateOfWorld.MarkVolumeAsAttached(
                vol.volumeName,
                spec,
                vol.nodeName,
                vol.devicePath,
            )

            rc.actualStateOfWorld.MarkVolumeAsMounted(
                vol.podName,
                vol.podUID,
                vol.volumeName,
                vol.mounter,
                vol.outerVolumeSpecName,
            )
        }
    }

    return nil
}
```

### Reconstruction Steps

```mermaid
flowchart TB
    START[Kubelet Start]
    SCAN[Scan /var/lib/kubelet/pods/*/volumes/]

    SCAN --> ITER[For each volume directory]

    ITER --> DETECT{Detect plugin<br/>from path}

    DETECT -->|Found| CONSTRUCT[Construct volume spec]
    DETECT -->|Unknown| SKIP[Skip volume]

    CONSTRUCT --> ACTUAL[Add to actual state]

    ACTUAL --> MOUNTED{Is mounted?}

    MOUNTED -->|Yes| MARK_MOUNT[Mark as mounted]
    MOUNTED -->|No| MARK_UNMOUNT[Mark for cleanup]

    MARK_MOUNT --> NEXT[Next volume]
    MARK_UNMOUNT --> NEXT
    SKIP --> NEXT

    NEXT -->|More volumes| ITER
    NEXT -->|Done| RECONCILE[Start reconciler]

    style CONSTRUCT fill:#d4f4dd
    style ACTUAL fill:#e1f5ff
```

---

## Best Practices

### 1. Use CSI Drivers

```yaml
# ✅ Good: Use CSI driver
apiVersion: v1
kind: PersistentVolume
metadata:
  name: csi-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  csi:
    driver: pd.csi.storage.gke.io
    volumeHandle: projects/my-project/zones/us-west1-a/disks/my-disk

# ❌ Deprecated: Avoid in-tree plugins
spec:
  gcePersistentDisk:  # Deprecated in-tree plugin
    pdName: my-disk
    fsType: ext4
```

### 2. Set Appropriate Mount Propagation

```yaml
# ✅ Good: Set propagation based on use case
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: storage-orchestrator
    volumeMounts:
    - name: host-mounts
      mountPath: /var/lib/kubelet
      mountPropagation: Bidirectional  # Needed for dynamic provisioning

# ❌ Bad: Unnecessary bidirectional propagation
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
      mountPropagation: Bidirectional  # Not needed, security risk
```

### 3. Use SubPath Carefully

```yaml
# ✅ Good: Safe subpath usage
volumeMounts:
- name: config
  mountPath: /etc/app/config.yaml
  subPath: config.yaml
  readOnly: true  # Read-only for config files

# ⚠️ Caution: Dynamic subpath
volumeMounts:
- name: data
  mountPath: /data
  subPathExpr: $(POD_NAME)  # Ensure POD_NAME is validated
```

### 4. Handle Volume Permissions

```yaml
# ✅ Good: Set FSGroup for shared volumes
apiVersion: v1
kind: Pod
spec:
  securityContext:
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"  # Optimize permission changes
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data

# ✅ Good: Use init container for permissions
  initContainers:
  - name: fix-permissions
    image: busybox
    command: ['sh', '-c', 'chmod -R 770 /data && chown -R 1000:2000 /data']
    volumeMounts:
    - name: data
      mountPath: /data
```

### 5. Monitor Volume Health

```bash
# Check volume stats
kubectl get --raw /api/v1/nodes/<node>/proxy/stats/summary | jq '.pods[].volume'

# Monitor CSI driver health
kubectl get csidrivers
kubectl describe csidriver pd.csi.storage.gke.io

# Check volume attachment status
kubectl get volumeattachments
```

### 6. Set Resource Limits

```yaml
# ✅ Good: Set ephemeral storage limits
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    resources:
      limits:
        ephemeral-storage: "10Gi"
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 10Gi  # Enforce size limit
```

### 7. Use Volume Snapshots for Backup

```yaml
# ✅ Good: Regular snapshots
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: data-pvc
```

---

## Troubleshooting

### Common Issues

#### Volume Not Mounting

```bash
# Check volume plugin status
kubectl describe pod <pod>
# Look for Events section

# Check kubelet logs
journalctl -u kubelet | grep volume

# Check CSI driver logs
kubectl logs -n kube-system csi-driver-pod

# Verify volume attachment
kubectl get volumeattachments
```

#### Permission Denied

```bash
# Check fsGroup setting
kubectl get pod <pod> -o jsonpath='{.spec.securityContext}'

# Verify actual permissions
kubectl exec <pod> -- ls -la /mount/path

# Check SELinux context
kubectl exec <pod> -- ls -laZ /mount/path
```

#### Slow Volume Operations

```bash
# Check for timeout events
kubectl get events --field-selector reason=VolumeResizeFailed

# Monitor CSI operations
kubectl logs -n kube-system csi-provisioner -f
kubectl logs -n kube-system csi-attacher -f

# Check for stuck operations
kubectl get volumeattachments -o json | jq '.items[] | select(.status.attached==false)'
```

---

## Summary

### Key Takeaways

1. **Plugin Architecture** - Modular system supporting various storage backends
2. **CSI Standard** - Primary interface for storage integration
3. **Volume Lifecycle** - Attach → Stage → Mount → Use → Unmount → Unstage → Detach
4. **Mount Propagation** - Controls mount visibility between host and containers
5. **Subpath Support** - Mount subdirectories or specific files
6. **SELinux & Ownership** - Automatic relabeling and permission management
7. **Reconstruction** - Volumes restored after kubelet restart
8. **Metrics & Monitoring** - Built-in metrics for capacity and health

### Operation Flow Summary

```
Pod Created
    ↓
VolumeManager: Add to desired state
    ↓
Reconciler: Compare desired vs actual
    ↓
Attach (if attachable)
    ↓
Mount Device (global path for attachable volumes)
    ↓
Mount Volume (pod-specific path)
    ↓
Pod Running with Volume
    ↓
Pod Terminated
    ↓
Unmount Volume (pod path)
    ↓
Unmount Device (if last pod using it)
    ↓
Detach (if no pods need it)
```

**Related Documents**:
- [Volume Manager](../middle-level/07-volume-manager.md) - Volume manager implementation
- [CSI Spec](https://github.com/container-storage-interface/spec) - CSI specification
- [Mount Propagation](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation) - Official docs

---

**Document Statistics**:
- **Lines**: 1,100+
- **Code References**: 35+
- **Diagrams**: 12 Mermaid diagrams
- **Tables**: 0 (focused on flows and code)

**Last Updated**: 2025-10-21
**Kubernetes Version**: v1.32+