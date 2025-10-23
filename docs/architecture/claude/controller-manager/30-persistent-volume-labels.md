# Persistent Volume Labels (Deprecated)

## Overview

**Status**: DEPRECATED - Superseded by CSI Topology Features

The PVLabeler interface was a mechanism for cloud providers to automatically add topology labels (zone, region) to PersistentVolumes. This functionality has been **deprecated in favor of CSI topology features** and is maintained only for backward compatibility with legacy in-tree volume plugins.

**Primary Location**: `staging/src/k8s.io/cloud-provider/cloud.go:292-296`

**Deprecation Notice**: This interface is deprecated. New deployments should use CSI drivers with topology support instead.

## Historical Context

### What PVLabeler Did

The PVLabeler interface allowed cloud providers to implement automatic labeling of PersistentVolumes with topology information:

```go
// Location: staging/src/k8s.io/cloud-provider/cloud.go:292-296

// PVLabeler is an abstract, pluggable interface for fetching labels for volumes
// DEPRECATED: PVLabeler is deprecated in favor of CSI topology feature.
type PVLabeler interface {
    GetLabelsForVolume(ctx context.Context, pv *v1.PersistentVolume) (map[string]string, error)
}
```

### Labels Applied

Cloud providers implementing this interface would return labels such as:

```yaml
labels:
  failure-domain.beta.kubernetes.io/zone: us-east-1a        # DEPRECATED
  failure-domain.beta.kubernetes.io/region: us-east-1      # DEPRECATED
  topology.kubernetes.io/zone: us-east-1a                  # Modern equivalent (CSI)
  topology.kubernetes.io/region: us-east-1                 # Modern equivalent (CSI)
```

## Why It Was Deprecated

### Problems with PVLabeler

1. **In-Tree Plugin Dependency**: Only worked with in-tree cloud provider volume plugins
2. **Cloud Provider Lock-in**: Tightly coupled to specific cloud provider implementations
3. **Limited Flexibility**: Could not express complex topology requirements
4. **Maintenance Burden**: Required updates to core Kubernetes for cloud-specific logic

### CSI Topology Advantages

CSI (Container Storage Interface) topology provides:

1. **Declarative Topology**: StorageClass with `allowedTopologies`
2. **Dynamic Provisioning**: Topology-aware volume provisioning
3. **Out-of-Tree**: Cloud-specific logic in external CSI drivers
4. **Rich Constraints**: Complex topology expressions via node selectors

## Migration Path

### From PVLabeler to CSI Topology

**Old Approach (PVLabeler)**:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-aws
  labels:
    failure-domain.beta.kubernetes.io/zone: us-east-1a  # Auto-added by PVLabeler
    failure-domain.beta.kubernetes.io/region: us-east-1
spec:
  capacity:
    storage: 100Gi
  awsElasticBlockStore:
    volumeID: vol-12345
```

**New Approach (CSI Topology)**:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-east-1a
    - us-east-1b
volumeBindingMode: WaitForFirstConsumer  # Topology-aware scheduling
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-csi
spec:
  capacity:
    storage: 100Gi
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-12345
  nodeAffinity:  # CSI driver sets this automatically
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - us-east-1a
```

## CSI Topology Features

### Key Improvements

**1. Topology-Aware Provisioning**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
volumeBindingMode: WaitForFirstConsumer  # Wait for pod scheduling
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-central1-a
    - us-central1-b
```

**2. Pod Scheduling Integration**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: data
      mountPath: /data
  # Scheduler ensures pod is placed in same zone as volume
```

**3. Node Affinity (Automatic)**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: csi-pv
spec:
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - us-east-1a
```

## Implementation Reference (Historical)

### Cloud Provider Implementation Example

**AWS (Legacy In-Tree Plugin)**:
```go
// Historical reference - DO NOT USE in new code
func (aws *Cloud) GetLabelsForVolume(ctx context.Context, pv *v1.PersistentVolume) (map[string]string, error) {
    // Extract volume ID from PV spec
    volumeID := pv.Spec.AWSElasticBlockStore.VolumeID

    // Query AWS API for volume details
    volume, err := aws.getVolumeByID(volumeID)
    if err != nil {
        return nil, err
    }

    // Return topology labels
    return map[string]string{
        "failure-domain.beta.kubernetes.io/zone":   volume.AvailabilityZone,
        "failure-domain.beta.kubernetes.io/region": aws.region,
    }, nil
}
```

### When Labels Were Applied

Labels were typically applied:
1. During PV creation (static provisioning)
2. After dynamic provisioning by in-tree provisioner
3. On PV controller reconciliation loops

## Current Status

### Still Supported For

- **Legacy in-tree volume plugins** (AWS EBS, GCE PD, Azure Disk)
- **Existing PVs** with old label format
- **Backward compatibility** during migration to CSI

### Not Supported For

- **New CSI drivers** (use CSI topology instead)
- **Out-of-tree cloud providers** (KEP-2395)
- **New clusters** (should use CSI from the start)

## Recommendations

### For New Deployments

✅ **DO**:
- Use CSI drivers with topology support
- Configure `volumeBindingMode: WaitForFirstConsumer`
- Use `allowedTopologies` in StorageClasses
- Rely on CSI driver to set node affinity

❌ **DON'T**:
- Implement new PVLabeler interfaces
- Rely on deprecated zone/region labels
- Use in-tree cloud provider volume plugins

### For Existing Deployments

**Migration Strategy**:

1. **Phase 1**: Deploy CSI drivers alongside in-tree plugins
```yaml
# Enable CSI migration feature gates
--feature-gates=CSIMigration=true,CSIMigrationAWS=true
```

2. **Phase 2**: Migrate StorageClasses to CSI
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
provisioner: ebs.csi.aws.com  # Changed from kubernetes.io/aws-ebs
parameters:
  type: gp2
volumeBindingMode: WaitForFirstConsumer
```

3. **Phase 3**: Update workloads to use new StorageClasses

4. **Phase 4**: Disable in-tree plugins
```yaml
--feature-gates=InTreePluginAWSUnregister=true
```

## Related Documentation

### CSI Topology Documentation
- CSI Specification: https://github.com/container-storage-interface/spec/blob/master/spec.md#topology
- Kubernetes CSI Documentation: https://kubernetes-csi.github.io/docs/topology.html
- KEP-1487: CSI Topology Support

### Migration Guides
- KEP-625: CSI Migration
- Cloud Provider Extraction: KEP-2395
- In-Tree Storage Plugin to CSI Migration: Official Kubernetes docs

### Replacement Features
- **CSI Topology**: Automatic topology-aware scheduling
- **Volume Binding Mode**: `WaitForFirstConsumer` for delayed binding
- **Node Affinity**: CSI drivers set this automatically
- **Allowed Topologies**: StorageClass-level topology constraints

## Summary

The PVLabeler interface was a stopgap solution for adding topology labels to PersistentVolumes in the era of in-tree cloud provider plugins. It has been superseded by the more powerful and flexible CSI topology features.

**Key Points**:
1. **DEPRECATED**: Do not use for new deployments
2. **CSI Topology**: Modern replacement with better capabilities
3. **Migration Path**: Well-defined migration from in-tree to CSI
4. **Backward Compatibility**: Still works for legacy plugins during transition
5. **Removal Timeline**: Will be removed when in-tree plugins are fully deprecated

**For Modern Kubernetes Deployments**: Use CSI drivers with topology support instead of relying on PVLabeler. CSI provides superior topology awareness, better integration with the scheduler, and follows the out-of-tree plugin model.

## Additional Resources

- CSI Drivers: https://kubernetes-csi.github.io/docs/drivers.html
- AWS EBS CSI Driver: https://github.com/kubernetes-sigs/aws-ebs-csi-driver
- GCE PD CSI Driver: https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver
- Azure Disk CSI Driver: https://github.com/kubernetes-sigs/azuredisk-csi-driver
- Volume Topology: https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode
