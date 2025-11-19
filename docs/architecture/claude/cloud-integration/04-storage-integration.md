# **Cloud Storage Integration**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Comprehensive guide to Kubernetes cloud storage integration patterns

**Target Audience**:
- Platform engineers deploying persistent storage
- Storage administrators managing cloud volumes
- SREs troubleshooting storage issues
- Cloud provider developers implementing CSI drivers

**Scope**: CSI architecture, volume provisioning, cloud provider storage, and production patterns

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Cloud Storage Architecture**

### **Storage Integration Overview**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Storage Stack                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │     Pod     │  │     Pod     │  │     Pod     │                 │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │
│         │                │                │                         │
│         └────────────────┼────────────────┘                         │
│                          │                                          │
│                  ┌───────▼───────┐                                  │
│                  │      PVC      │                                  │
│                  └───────┬───────┘                                  │
│                          │                                          │
│            ┌─────────────┼─────────────┐                            │
│            ▼             ▼             ▼                            │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│     │    PV    │  │    PV    │  │    PV    │                       │
│     └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
│          │             │             │                              │
│          └─────────────┼─────────────┘                              │
│                        │                                            │
│              ┌─────────▼─────────┐                                  │
│              │   StorageClass    │                                  │
│              │   + Provisioner   │                                  │
│              └─────────┬─────────┘                                  │
│                        │                                            │
├────────────────────────┼────────────────────────────────────────────┤
│                        │                                            │
│              ┌─────────▼─────────┐                                  │
│              │    CSI Driver     │                                  │
│              │   (Controller +   │                                  │
│              │    Node Plugin)   │                                  │
│              └─────────┬─────────┘                                  │
│                        │                                            │
│              ┌─────────▼─────────┐                                  │
│              │   Cloud Storage   │                                  │
│              │   (EBS/GCE PD/    │                                  │
│              │    Azure Disk)    │                                  │
│              └───────────────────┘                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### **CSI (Container Storage Interface) Architecture**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CSI Architecture                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Control Plane                           Node                       │
│  ┌───────────────────────────┐          ┌─────────────────────┐    │
│  │   external-provisioner    │          │   node-driver-      │    │
│  │   external-attacher       │          │   registrar         │    │
│  │   external-resizer        │          │                     │    │
│  │   external-snapshotter    │          │   CSI Node Plugin   │    │
│  │                           │          │   (DaemonSet)       │    │
│  │   CSI Controller Plugin   │          │                     │    │
│  │   (Deployment)            │          │                     │    │
│  └──────────┬────────────────┘          └──────────┬──────────┘    │
│             │                                      │                │
│             │     gRPC                    gRPC     │                │
│             │                                      │                │
│             └──────────────────┬───────────────────┘                │
│                                │                                    │
│                      ┌─────────▼─────────┐                         │
│                      │  Cloud Storage    │                         │
│                      │  API              │                         │
│                      └───────────────────┘                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### **CSI Components**

| **Component** | **Location** | **Purpose** |
|---------------|--------------|-------------|
| **CSI Controller** | Deployment (control plane) | Create/delete/snapshot volumes |
| **CSI Node** | DaemonSet (every node) | Mount/unmount volumes |
| **external-provisioner** | Sidecar to controller | Watch PVCs, call CreateVolume |
| **external-attacher** | Sidecar to controller | Watch VolumeAttachments, call ControllerPublish |
| **external-resizer** | Sidecar to controller | Watch PVCs, call ControllerExpand |
| **external-snapshotter** | Sidecar to controller | Watch VolumeSnapshots, call CreateSnapshot |
| **node-driver-registrar** | Sidecar to node | Register driver with kubelet |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Volume Provisioning**

### **Dynamic Provisioning Flow**

```
User creates PVC
        │
        ▼
┌───────────────────┐
│  API Server       │
│  (PVC created)    │
└────────┬──────────┘
         │
         │ Watch
         ▼
┌───────────────────┐
│  PV Controller    │
│  (kube-cm)        │
└────────┬──────────┘
         │ Find StorageClass
         ▼
┌───────────────────┐
│  external-        │     CreateVolumeRequest
│  provisioner      │  ───────────────────────►
└───────────────────┘                          │
                                               │
                                               ▼
                                    ┌──────────────────┐
                                    │  CSI Driver      │
                                    │  (Controller)    │
                                    └────────┬─────────┘
                                             │
                                             │ Cloud API
                                             ▼
                                    ┌──────────────────┐
                                    │  Cloud Storage   │
                                    │  (Create Volume) │
                                    └────────┬─────────┘
                                             │
                    CreateVolumeResponse     │
         ◄───────────────────────────────────┘
         │
         ▼
┌───────────────────┐
│  Create PV        │
│  (API Server)     │
└───────────────────┘
         │
         │ Bind PVC to PV
         ▼
┌───────────────────┐
│  PVC Bound        │
└───────────────────┘
```

### **StorageClass Configuration**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    # Default storage class
    storageclass.kubernetes.io/is-default-class: "true"

# CSI driver name
provisioner: ebs.csi.aws.com

# Volume parameters (driver-specific)
parameters:
  # AWS EBS
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:..."

  # Topology constraints
  csi.storage.k8s.io/fstype: ext4

# Reclaim policy
reclaimPolicy: Delete  # Retain, Delete

# Volume binding mode
volumeBindingMode: WaitForFirstConsumer  # Immediate

# Allow volume expansion
allowVolumeExpansion: true

# Mount options
mountOptions:
  - noatime
  - nodiratime

# Allowed topologies
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-east-1a
    - us-east-1b
```

### **PersistentVolumeClaim Configuration**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  # Access modes
  accessModes:
    - ReadWriteOnce  # RWO, ROX, RWX, RWOP

  # Storage class
  storageClassName: fast-ssd

  # Resource requirements
  resources:
    requests:
      storage: 100Gi

  # Volume mode
  volumeMode: Filesystem  # Block

  # Data source (for cloning/snapshots)
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io

  # Data source ref (cross-namespace)
  dataSourceRef:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
    namespace: backup-namespace
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **☁️ Cloud Provider Storage Classes**

### **AWS EBS CSI**

```yaml
# GP3 SSD (default for most workloads)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
---
# IO2 for high-performance workloads
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-io2
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "64000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
---
# SC1 for cold storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc1
provisioner: ebs.csi.aws.com
parameters:
  type: sc1
volumeBindingMode: WaitForFirstConsumer
```

### **GCE Persistent Disk CSI**

```yaml
# SSD Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  disk-encryption-kms-key: projects/.../cryptoKeys/...
volumeBindingMode: WaitForFirstConsumer
---
# Balanced Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-balanced
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-balanced
volumeBindingMode: WaitForFirstConsumer
---
# Regional SSD (multi-zone replication)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd-regional
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
volumeBindingMode: WaitForFirstConsumer
```

### **Azure Disk CSI**

```yaml
# Premium SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS  # Standard_LRS, StandardSSD_LRS, Premium_LRS, UltraSSD_LRS
  cachingMode: ReadOnly
  diskEncryptionSetID: /subscriptions/.../diskEncryptionSets/...
volumeBindingMode: WaitForFirstConsumer
---
# Ultra SSD for high-performance
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-ultra
provisioner: disk.csi.azure.com
parameters:
  skuName: UltraSSD_LRS
  DiskIOPSReadWrite: "64000"
  DiskMBpsReadWrite: "2000"
  cachingMode: None
volumeBindingMode: WaitForFirstConsumer
---
# Zone-Redundant Storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-zrs
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_ZRS
volumeBindingMode: WaitForFirstConsumer
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Volume Operations**

### **Volume Attachment Flow**

```
Pod scheduled to node
         │
         ▼
┌───────────────────┐
│  AD Controller    │
│  (kube-cm)        │
└────────┬──────────┘
         │ Create VolumeAttachment
         ▼
┌───────────────────┐
│  external-        │     ControllerPublishVolume
│  attacher         │  ─────────────────────────►
└───────────────────┘                            │
                                                 │
                                                 ▼
                                      ┌──────────────────┐
                                      │  CSI Controller  │
                                      │  (Attach to node)│
                                      └────────┬─────────┘
                                               │
                                               │ Cloud API
                                               ▼
                                      ┌──────────────────┐
                                      │  Cloud Storage   │
                                      │  (Attach Volume) │
                                      └────────┬─────────┘
                                               │
         ControllerPublishVolumeResponse       │
         ◄─────────────────────────────────────┘
         │
         ▼
┌───────────────────┐
│  VolumeAttachment │
│  status.attached  │
└───────────────────┘
         │
         │ Kubelet sees attachment ready
         ▼
┌───────────────────┐
│  CSI Node Plugin  │     NodeStageVolume
│                   │  ─────────────────►
└───────────────────┘                    │
                                         │
                            NodePublishVolume
                        ─────────────────►
                                         │
                                         ▼
                              ┌────────────────┐
                              │  Mount to pod  │
                              └────────────────┘
```

### **Volume Expansion**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  resources:
    requests:
      # Increase this value to expand
      storage: 200Gi  # was 100Gi
```

**Expansion Flow**:

```
PVC storage increased
         │
         ▼
┌───────────────────┐
│  external-        │     ControllerExpandVolume
│  resizer          │  ─────────────────────────►
└───────────────────┘                            │
                                                 ▼
                                      ┌──────────────────┐
                                      │  CSI Controller  │
                                      │  (Expand Volume) │
                                      └────────┬─────────┘
                                               │
                                               │ Cloud API
                                               ▼
                                      ┌──────────────────┐
                                      │  Expand cloud    │
                                      │  volume          │
                                      └────────┬─────────┘
                                               │
         ◄─────────────────────────────────────┘
         │
         ▼
┌───────────────────┐
│  Update PV        │
│  capacity         │
└───────────────────┘
         │
         │ If filesystem expansion needed
         ▼
┌───────────────────┐
│  CSI Node Plugin  │     NodeExpandVolume
│                   │  ─────────────────►
└───────────────────┘                    │
                                         │
                              ┌────────────────┐
                              │  Expand FS     │
                              │  (resize2fs)   │
                              └────────────────┘
```

### **Volume Snapshots**

```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  # Driver-specific parameters
---
# VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    # Snapshot from existing PVC
    persistentVolumeClaimName: my-pvc
---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  storageClassName: ebs-gp3
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌍 Topology-Aware Provisioning**

### **Zone-Aware Storage**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: topology-aware
provisioner: ebs.csi.aws.com
parameters:
  type: gp3

# Wait for pod scheduling to know which zone
volumeBindingMode: WaitForFirstConsumer

# Restrict to specific zones
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-east-1a
    - us-east-1b
```

### **How Topology Works**

```
Pod created with PVC
         │
         ▼
┌───────────────────┐
│  Scheduler        │
│  (find node)      │
└────────┬──────────┘
         │
         │ Select node based on:
         │ - resource requirements
         │ - affinity/anti-affinity
         │ - topology constraints
         ▼
┌───────────────────┐
│  Node selected    │
│  (zone: us-east-1a)
└────────┬──────────┘
         │
         │ PV Controller sees binding
         ▼
┌───────────────────┐
│  Provision volume │
│  in us-east-1a    │
└───────────────────┘
```

### **Cross-Zone Considerations**

| **Scenario** | **Behavior** | **Recommendation** |
|--------------|--------------|-------------------|
| **Pod moves to different zone** | Cannot attach volume | Use regional PD or replicated storage |
| **Multi-zone deployment** | One volume per zone | Use StatefulSet with volumeClaimTemplates |
| **Zone failure** | Volume inaccessible | Use multi-zone replication |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚨 Production Troubleshooting**

### **Scenario 1: PVC Stuck in Pending**

**Symptoms**:
```bash
kubectl get pvc
# NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc    Pending                                      fast-ssd       10m
```

**Investigation**:

```bash
# 1. Check PVC events
kubectl describe pvc my-pvc

# Events:
#   Type     Reason                Age   Message
#   ----     ------                ----  -------
#   Normal   WaitForFirstConsumer  1m    waiting for first consumer to be created before binding

# 2. Check StorageClass exists
kubectl get sc fast-ssd

# 3. Check CSI driver pods
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl logs -n kube-system -l app=ebs-csi-controller -c csi-provisioner

# 4. Check external-provisioner logs
kubectl logs -n kube-system deployment/ebs-csi-controller -c csi-provisioner

# 5. Check cloud quotas
# AWS: EC2 volume limits
# GCE: Disk quotas
# Azure: Storage account limits
```

**Common Causes**:

1. **WaitForFirstConsumer**: PVC won't bind until pod is scheduled
2. **Storage quota exceeded**: Check cloud provider limits
3. **Invalid parameters**: Check StorageClass configuration
4. **CSI driver not running**: Check driver pods

### **Scenario 2: Volume Attachment Failures**

**Symptoms**:
```bash
kubectl describe pod my-pod
# Warning  FailedAttachVolume  1m  attachdetach-controller
#   AttachVolume.Attach failed for volume "pvc-xxx": rpc error:
#   code = Internal desc = Could not attach volume "vol-xxx" to node "i-xxx"
```

**Investigation**:

```bash
# 1. Check VolumeAttachment status
kubectl get volumeattachment
kubectl describe volumeattachment csi-xxx

# 2. Check external-attacher logs
kubectl logs -n kube-system deployment/ebs-csi-controller -c csi-attacher

# 3. Check node status
kubectl describe node <node-name> | grep -A5 "Attached"

# 4. Check cloud provider attachment
# AWS
aws ec2 describe-volumes --volume-ids vol-xxx

# 5. Check instance attachment limits
# AWS: Different instance types have different EBS limits
```

**Common Causes**:

1. **Instance attachment limit**: EC2 instances have max volume attachments
2. **Wrong availability zone**: Volume and node in different zones
3. **Volume already attached**: Detach from previous node first
4. **IAM permissions**: CSI controller needs attach permissions

### **Scenario 3: Filesystem Corruption**

**Symptoms**:
```bash
# Pod stuck in ContainerCreating
kubectl describe pod my-pod
# Warning  FailedMount  1m  kubelet
#   MountVolume.MountDevice failed for volume "pvc-xxx":
#   rpc error: code = Internal desc = Failed to mount device:
#   exit status 32

# Or in logs:
# EXT4-fs error (device nvme1n1): ext4_find_entry: bad entry in directory
```

**Investigation**:

```bash
# 1. Check node kubelet logs
kubectl logs -n kube-system -l k8s-app=kubelet --tail=100

# 2. SSH to node and check dmesg
dmesg | grep -i ext4
dmesg | grep -i nvme

# 3. Check filesystem
fsck /dev/nvme1n1

# 4. Check volume health in cloud provider
# AWS
aws ec2 describe-volume-status --volume-ids vol-xxx
```

**Recovery**:

```bash
# 1. Detach from current node
kubectl delete volumeattachment csi-xxx

# 2. Attach to recovery instance and run fsck
aws ec2 attach-volume --volume-id vol-xxx --instance-id i-recovery --device /dev/sdf
ssh recovery-instance
fsck -y /dev/xvdf

# 3. Re-attach to Kubernetes node
```

### **Scenario 4: Slow Volume Performance**

**Symptoms**:
- High disk latency
- Slow application response
- IOPS exhausted

**Investigation**:

```bash
# 1. Check cloud metrics
# AWS CloudWatch: VolumeReadOps, VolumeWriteOps, VolumeThroughputLimit
# GCE: disk/read_bytes_count, disk/write_bytes_count
# Azure: Data Disk IOPS Consumed Percentage

# 2. Check volume type
kubectl get pv <pv-name> -o yaml | grep -A10 csi

# 3. Check if hitting limits
# GP3: 3000 base IOPS
# IO2: Up to 64000 IOPS

# 4. Monitor from node
iostat -x 1
```

**Optimization**:

```yaml
# Upgrade to higher performance tier
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-perf
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "64000"  # Max for io2
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Monitoring Storage**

### **Key Metrics**

```promql
# PVC capacity utilization
(kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) * 100

# Volume IOPS (node exporter)
rate(node_disk_reads_completed_total[5m])
rate(node_disk_writes_completed_total[5m])

# Volume latency
rate(node_disk_read_time_seconds_total[5m]) / rate(node_disk_reads_completed_total[5m])
rate(node_disk_write_time_seconds_total[5m]) / rate(node_disk_writes_completed_total[5m])

# CSI operations
csi_operations_seconds_bucket{driver_name="ebs.csi.aws.com"}

# Pending PVCs
kube_persistentvolumeclaim_status_phase{phase="Pending"}
```

### **Alerts**

```yaml
groups:
- name: storage-alerts
  rules:
  - alert: PVCPending
    expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
    for: 15m
    annotations:
      summary: "PVC stuck in pending"

  - alert: VolumeAlmostFull
    expr: |
      (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.85
    for: 5m
    annotations:
      summary: "Volume is almost full (>85%)"

  - alert: VolumeExpansionFailed
    expr: |
      kube_persistentvolumeclaim_resource_requests_storage_bytes >
      kube_persistentvolume_capacity_bytes
    for: 30m
    annotations:
      summary: "Volume expansion not completed"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💡 Best Practices**

### **StorageClass Configuration**

1. **Use WaitForFirstConsumer**:
```yaml
volumeBindingMode: WaitForFirstConsumer
# Ensures volume created in same zone as pod
```

2. **Enable Encryption**:
```yaml
parameters:
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:..."
```

3. **Allow Expansion**:
```yaml
allowVolumeExpansion: true
# Plan for growth
```

4. **Set Appropriate Reclaim Policy**:
```yaml
# Production: Retain to prevent data loss
reclaimPolicy: Retain

# Dev/test: Delete to clean up
reclaimPolicy: Delete
```

### **Capacity Planning**

1. **Right-size volumes**: Don't over-provision initially
2. **Monitor utilization**: Alert at 80% capacity
3. **Plan for expansion**: Use expandable storage classes
4. **Understand limits**: Know cloud provider attachment/IOPS limits

### **High Availability**

1. **Multi-zone for critical data**:
```yaml
# GCE Regional PD
parameters:
  replication-type: regional-pd

# Azure ZRS
parameters:
  skuName: Premium_ZRS
```

2. **Regular snapshots**:
```yaml
# Scheduled snapshots
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: daily-snapshot
```

3. **Test restore procedures**: Regularly verify backups work

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 References**

### **Source Code**
- **CSI Spec**: https://github.com/container-storage-interface/spec
- **external-provisioner**: https://github.com/kubernetes-csi/external-provisioner
- **external-attacher**: https://github.com/kubernetes-csi/external-attacher
- **external-resizer**: https://github.com/kubernetes-csi/external-resizer

### **Related Documentation**
- **CSI Architecture**: `docs/architecture/claude/csi/`
- **Cloud Controller Manager**: `docs/architecture/claude/cloud-integration/01-cloud-controller-manager.md`

### **External Resources**
- **AWS EBS CSI Driver**: https://github.com/kubernetes-sigs/aws-ebs-csi-driver
- **GCE PD CSI Driver**: https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver
- **Azure Disk CSI Driver**: https://github.com/kubernetes-sigs/azuredisk-csi-driver
- **Kubernetes Storage Concepts**: https://kubernetes.io/docs/concepts/storage/

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: Complete
**Last Updated**: 2024-11-19
**Target Audience**: Platform engineers, storage administrators, SREs
**Scope**: CSI architecture, cloud storage classes, volume operations, troubleshooting
