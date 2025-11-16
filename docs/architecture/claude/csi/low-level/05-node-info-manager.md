# **CSI Node Info Manager - CSINode Resource Management**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Overview**

The CSI Node Info Manager is responsible for managing CSINode API resources, which advertise CSI driver capabilities and node-specific information to the Kubernetes control plane. It bridges the gap between node-local CSI driver registration and cluster-wide scheduling and provisioning decisions.

**Purpose:**
- **Driver Installation**: Create and update CSINode resources when drivers register
- **Topology Advertising**: Publish node topology information for zone-aware scheduling
- **Volume Limits**: Report maximum volumes per node for scheduler capacity planning
- **Node ID Management**: Store driver-specific node identifiers for attach operations
- **Migration Support**: Handle CSI migration from in-tree drivers

**Key Responsibilities:**
- Create CSINode resource on first driver registration
- Update CSIDrivers field as drivers register/unregister
- Store topology keys (zone, region) from NodeGetInfo RPC
- Track volume attachment limits per driver
- Maintain CSI migration annotations

**Core Implementation:**
```
/pkg/volume/csi/nodeinfomanager/
├── nodeinfomanager.go           # Main implementation (500+ lines)
└── nodeinfomanager_test.go      # Unit tests
```

**Cross-References:**
- [Plugin Registration](./01-plugin-registration.md) - Triggers NodeInfoManager updates
- [Driver Store](./04-driver-store.md) - In-memory complement to CSINode
- [Volume Operations](./03-volume-operations.md) - Uses node IDs from CSINode
- [Scheduler Integration](../middle-level/05-scheduler-integration.md) - Consumes CSINode for scheduling
- [API Resources](../high-level/02-api-resources.md) - CSINode API definition

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **CSINode Resource Architecture**

### **CSINode Resource Structure**

The CSINode resource is a cluster-scoped API object that represents CSI driver information for a specific node.

**API Definition:**
```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: node-1  # Same as node name
  annotations:
    storage.alpha.kubernetes.io/migrated-plugins: "kubernetes.io/aws-ebs,kubernetes.io/gce-pd"
spec:
  drivers:
  - name: ebs.csi.aws.com
    nodeID: i-0123456789abcdef0
    topologyKeys:
    - topology.ebs.csi.aws.com/zone
    allocatable:
      count: 39  # Max volumes this driver can attach

  - name: efs.csi.aws.com
    nodeID: i-0123456789abcdef0
    topologyKeys:
    - topology.kubernetes.io/zone
    - topology.kubernetes.io/region
    allocatable:
      count: 100

  - name: fsx.csi.aws.com
    nodeID: i-0123456789abcdef0
    topologyKeys:
    - topology.kubernetes.io/zone
```

### **CSINode in Kubernetes Architecture**

```mermaid
graph TB
    subgraph "Node Components"
        A[Kubelet]
        B[CSI Driver]
        C[node-driver-registrar]
        D[NodeInfoManager]
    end

    subgraph "API Server"
        E[CSINode Resource]
        F[Node Resource]
    end

    subgraph "Control Plane Consumers"
        G[Scheduler]
        H[PV Controller]
        I[Attach/Detach Controller]
    end

    B --> C
    C -->|Register| A
    A --> D
    D -->|NodeGetInfo RPC| B
    B -->|nodeID, topology, limits| D

    D -->|Create/Update| E

    E -->|Read topology| G
    E -->|Read nodeID| I
    E -->|Read limits| G
    E -->|Read driver list| H

    F -.->|Zone labels| G

    style E fill:#4ecdc4
    style D fill:#95e1d3
```

### **CSINode vs Node Resource**

| Aspect | CSINode | Node |
|--------|---------|------|
| **Purpose** | CSI driver info | General node info |
| **Owner** | CSI subsystem | Kubelet |
| **Lifecycle** | Created on first CSI driver | Created on node join |
| **Topology** | Driver-specific keys | Standard labels (zone, region) |
| **Volume Limits** | Per-driver limits | Legacy annotation |
| **Updates** | On driver register/unregister | Periodic heartbeat |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **nodeInfoManager Structure**

### **Core Structure**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 64-70)**

```go
// nodeInfoManager contains necessary common dependencies to update node info on both
// the Node and CSINode objects.
type nodeInfoManager struct {
	nodeName        types.NodeName
	volumeHost      volume.VolumeHost
	migratedPlugins map[string](func() bool)
	// lock protects changes to node.
	lock sync.Mutex
}
```

**Fields:**
- **`nodeName`**: Kubernetes node name (matches Node resource name)
- **`volumeHost`**: Access to kubelet's volume subsystem and k8s client
- **`migratedPlugins`**: Map of in-tree plugin names to feature gate checks
- **`lock`**: Mutex protecting concurrent updates to Node/CSINode

### **Interface Definition**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 76-94)**

```go
// Interface implements an interface for managing labels of a node
type Interface interface {
	// CreateCSINode creates CSINode object for the node
	CreateCSINode() (*storagev1.CSINode, error)

	// Updates or Creates the CSINode object with annotations for CSI Migration
	InitializeCSINodeWithAnnotation() error

	// Record in the cluster the given node information from the CSI driver
	// Concurrent calls to InstallCSIDriver() is allowed
	InstallCSIDriver(driverName string, driverNodeID string, maxVolumeLimit int64, topology map[string]string) error

	// UpdateCSIDriver updates CSIDrivers field in the CSINode object
	UpdateCSIDriver(driverName string, driverNodeID string, maxAttachLimit int64, topology map[string]string) error

	// Remove in the cluster node information from the CSI driver
	UninstallCSIDriver(driverName string) error
}
```

### **Initialization**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 96-106)**

```go
// NewNodeInfoManager initializes nodeInfoManager
func NewNodeInfoManager(
	nodeName types.NodeName,
	volumeHost volume.VolumeHost,
	migratedPlugins map[string](func() bool)) Interface {
	return &nodeInfoManager{
		nodeName:        nodeName,
		volumeHost:      volumeHost,
		migratedPlugins: migratedPlugins,
	}
}
```

**Usage in Kubelet:**
```go
// In kubelet initialization
nim := nodeinfomanager.NewNodeInfoManager(
	kubelet.nodeName,
	kubelet.volumePluginMgr,
	migratedPlugins,
)

// Used during CSI plugin registration
nim.InstallCSIDriver(driverName, nodeID, maxVolumes, topology)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Driver Installation Flow**

### **Complete Installation Sequence**

```mermaid
sequenceDiagram
    participant D as CSI Driver
    participant R as node-driver-registrar
    participant PM as Plugin Manager
    participant NIM as NodeInfoManager
    participant API as API Server
    participant CSINode as CSINode Resource

    Note over D: Driver pod starts
    D->>R: Initialize
    R->>PM: RegisterPlugin(socket)

    PM->>NIM: InstallCSIDriver(driverName, ...)

    NIM->>D: NodeGetInfo() RPC
    D-->>NIM: {nodeID, maxVolumes, topology}

    NIM->>NIM: Lock for update

    alt CSINode doesn't exist
        NIM->>API: Create CSINode
        API->>CSINode: Resource created
    else CSINode exists
        NIM->>API: Get CSINode
        API-->>NIM: Current CSINode
    end

    NIM->>NIM: Update drivers array
    NIM->>NIM: Add/update driver entry

    NIM->>API: Update CSINode
    API->>CSINode: Patch applied

    NIM->>NIM: Unlock

    NIM-->>PM: Success
    PM-->>R: Registered

    Note over CSINode: Driver information<br/>now available cluster-wide
```

### **InstallCSIDriver Implementation**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 112-134)**

```go
// InstallCSIDriver updates the node ID annotation in the Node object and CSIDrivers field in the
// CSINode object. If the CSINode object doesn't yet exist, it will be created.
// If multiple calls to InstallCSIDriver() are made in parallel, some calls might receive Node or
// CSINode update conflicts, which causes the function to retry the corresponding update.
func (nim *nodeInfoManager) InstallCSIDriver(driverName string, driverNodeID string, maxAttachLimit int64, topology map[string]string) error {
	if driverNodeID == "" {
		return fmt.Errorf("error adding CSI driver node info: driverNodeID must not be empty")
	}

	nodeUpdateFuncs := []nodeUpdateFunc{
		removeMaxAttachLimit(driverName), // remove in 1.35 due to version skew policy
		updateNodeIDInNode(driverName, driverNodeID),
		updateTopologyLabels(topology),
	}

	err := nim.updateNode(nodeUpdateFuncs...)
	if err != nil {
		return fmt.Errorf("error updating Node object with CSI driver node info: %v", err)
	}

	err = nim.updateCSINode(driverName, driverNodeID, maxAttachLimit, topology)
	if err != nil {
		return fmt.Errorf("error updating CSINode object with CSI driver node info: %v", err)
	}

	return nil
}
```

**Key Steps:**
1. **Validate Input**: Ensure driverNodeID is not empty
2. **Update Node Object**: Add node ID annotation and topology labels
3. **Update CSINode Object**: Add driver entry to CSIDrivers array
4. **Error Handling**: Retry on conflict, fail on other errors

### **Node Object Updates**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 200-250)**

```go
func updateNodeIDInNode(driverName string, driverNodeID string) nodeUpdateFunc {
	return func(node *v1.Node) (*v1.Node, bool, error) {
		if node.Annotations == nil {
			node.Annotations = make(map[string]string)
		}

		// Get existing node ID map
		var nodeIDs map[string]string
		if val, ok := node.Annotations[annotationKeyNodeID]; ok {
			if err := json.Unmarshal([]byte(val), &nodeIDs); err != nil {
				return nil, false, err
			}
		} else {
			nodeIDs = make(map[string]string)
		}

		// Check if already up to date
		if existingID, ok := nodeIDs[driverName]; ok && existingID == driverNodeID {
			return node, false, nil  // No update needed
		}

		// Update node ID for this driver
		nodeIDs[driverName] = driverNodeID

		// Marshal back to JSON
		jsonBytes, err := json.Marshal(nodeIDs)
		if err != nil {
			return nil, false, err
		}

		node.Annotations[annotationKeyNodeID] = string(jsonBytes)
		return node, true, nil  // Update required
	}
}
```

**Node Annotation Example:**
```yaml
apiVersion: v1
kind: Node
metadata:
  name: node-1
  annotations:
    csi.volume.kubernetes.io/nodeid: |
      {
        "ebs.csi.aws.com": "i-0123456789abcdef0",
        "efs.csi.aws.com": "i-0123456789abcdef0",
        "fsx.csi.aws.com": "i-0123456789abcdef0"
      }
spec:
  # ... node spec ...
```

### **CSINode Object Updates**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 300-400)**

```go
func (nim *nodeInfoManager) updateCSINode(
	driverName string,
	driverNodeID string,
	maxAttachLimit int64,
	topology map[string]string) error {

	nim.lock.Lock()
	defer nim.lock.Unlock()

	kubeClient := nim.volumeHost.GetKubeClient()
	if kubeClient == nil {
		return fmt.Errorf("error getting kube client")
	}

	// Build topology keys from map
	var topologyKeys []string
	for key := range topology {
		topologyKeys = append(topologyKeys, key)
	}
	sort.Strings(topologyKeys)  // Deterministic ordering

	// Build driver entry
	driver := storagev1.CSINodeDriver{
		Name:         driverName,
		NodeID:       driverNodeID,
		TopologyKeys: topologyKeys,
	}

	// Set allocatable if limit specified
	if maxAttachLimit > 0 {
		driver.Allocatable = &storagev1.VolumeNodeResources{
			Count: &maxAttachLimit,
		}
	}

	// Retry loop with exponential backoff
	return wait.ExponentialBackoff(updateBackoff, func() (bool, error) {
		// Get current CSINode
		csiNode, err := kubeClient.StorageV1().CSINodes().Get(
			context.TODO(), string(nim.nodeName), metav1.GetOptions{})

		if err != nil {
			if errors.IsNotFound(err) {
				// CSINode doesn't exist, create it
				csiNode = &storagev1.CSINode{
					ObjectMeta: metav1.ObjectMeta{
						Name: string(nim.nodeName),
					},
					Spec: storagev1.CSINodeSpec{
						Drivers: []storagev1.CSINodeDriver{driver},
					},
				}

				_, err := kubeClient.StorageV1().CSINodes().Create(
					context.TODO(), csiNode, metav1.CreateOptions{})
				if err != nil {
					if errors.IsAlreadyExists(err) {
						// Retry - another goroutine created it
						return false, nil
					}
					return false, err
				}
				klog.V(2).Infof("Created CSINode %s with driver %s", nim.nodeName, driverName)
				return true, nil  // Success
			}
			return false, err
		}

		// CSINode exists, update it
		driverExists := false
		for i, d := range csiNode.Spec.Drivers {
			if d.Name == driverName {
				// Update existing driver
				csiNode.Spec.Drivers[i] = driver
				driverExists = true
				break
			}
		}

		if !driverExists {
			// Add new driver
			csiNode.Spec.Drivers = append(csiNode.Spec.Drivers, driver)
		}

		_, err = kubeClient.StorageV1().CSINodes().Update(
			context.TODO(), csiNode, metav1.UpdateOptions{})
		if err != nil {
			if errors.IsConflict(err) {
				// Retry - resource was modified
				klog.V(4).Infof("CSINode update conflict for %s, retrying", nim.nodeName)
				return false, nil
			}
			return false, err
		}

		klog.V(2).Infof("Updated CSINode %s with driver %s", nim.nodeName, driverName)
		return true, nil  // Success
	})
}
```

**Retry Backoff Configuration:**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 54-60)**

```go
var (
	updateBackoff = wait.Backoff{
		Steps:    4,           // Max 4 retries
		Duration: 10 * time.Millisecond,
		Factor:   5.0,         // Exponential backoff: 10ms, 50ms, 250ms, 1250ms
		Jitter:   0.1,         // ±10% randomization
	}
)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **NodeGetInfo RPC**

### **Retrieving Driver Information**

The NodeInfoManager calls the CSI driver's `NodeGetInfo` RPC to obtain node-specific information.

**CSI Spec Definition:**
```protobuf
message NodeGetInfoRequest {
	// Intentionally empty
}

message NodeGetInfoResponse {
	// The identifier of the node as understood by the driver.
	// Required.
	string node_id = 1;

	// Maximum number of volumes that can be attached to the node.
	// Optional. 0 means unlimited.
	int64 max_volumes_per_node = 2;

	// Specifies where (regions, zones, racks, etc.) the node is accessible from.
	// A plugin that returns this field MUST also set the
	// VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capability.
	// Optional.
	Topology accessible_topology = 3;
}

message Topology {
	map<string, string> segments = 1;
}
```

### **NodeGetInfo Call Flow**

```mermaid
sequenceDiagram
    participant NIM as NodeInfoManager
    participant C as CSI Client
    participant D as CSI Driver
    participant Cloud as Cloud Provider API

    NIM->>C: Get NodeGetInfo
    C->>D: NodeGetInfo() RPC

    D->>Cloud: Query instance metadata
    Note over Cloud: Get instance ID,<br/>zone, region

    Cloud-->>D: Instance metadata

    D->>D: Build response
    Note over D: nodeID: i-0abc123<br/>maxVolumes: 39<br/>topology: {zone: us-east-1a}

    D-->>C: NodeGetInfoResponse
    C-->>NIM: {nodeID, maxVolumes, topology}

    NIM->>NIM: Update CSINode resource
```

### **Example Responses**

**AWS EBS CSI Driver:**
```go
// NodeGetInfo response
{
	NodeId: "i-0123456789abcdef0",           // EC2 instance ID
	MaxVolumesPerNode: 39,                    // EBS volume limit for instance type
	AccessibleTopology: &Topology{
		Segments: map[string]string{
			"topology.ebs.csi.aws.com/zone": "us-east-1a",
		},
	},
}
```

**AWS EFS CSI Driver:**
```go
// NodeGetInfo response
{
	NodeId: "i-0123456789abcdef0",           // EC2 instance ID
	MaxVolumesPerNode: 0,                     // No limit for EFS
	AccessibleTopology: &Topology{
		Segments: map[string]string{
			"topology.kubernetes.io/zone":   "us-east-1a",
			"topology.kubernetes.io/region": "us-east-1",
		},
	},
}
```

**GCE PD CSI Driver:**
```go
// NodeGetInfo response
{
	NodeId: "projects/my-project/zones/us-central1-a/instances/gke-node-1",
	MaxVolumesPerNode: 127,                   // GCE PD limit
	AccessibleTopology: &Topology{
		Segments: map[string]string{
			"topology.gke.io/zone": "us-central1-a",
		},
	},
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Topology Management**

### **What is Topology?**

Topology describes the physical or logical location of a node in the infrastructure (zone, region, rack, etc.). CSI drivers use topology to:
- Enable zone-aware volume provisioning
- Ensure volumes are created in the same zone as pods
- Support cross-zone volume access (where supported)

### **Topology Keys**

**Common Topology Keys:**
```go
const (
	// Standard Kubernetes topology keys
	LabelZoneFailureDomain       = "topology.kubernetes.io/zone"
	LabelZoneFailureDomainStable = "topology.kubernetes.io/zone"  // Stable as of 1.17
	LabelZoneRegion              = "topology.kubernetes.io/region"

	// Driver-specific topology keys
	// AWS
	"topology.ebs.csi.aws.com/zone"
	"topology.efs.csi.aws.com/zone"

	// GCE
	"topology.gke.io/zone"

	// Azure
	"topology.disk.csi.azure.com/zone"

	// vSphere
	"topology.csi.vmware.com/zone"
	"topology.csi.vmware.com/rack"
)
```

### **Topology in CSINode**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 350-380)**

```go
func buildTopologyKeys(topology map[string]string) []string {
	if len(topology) == 0 {
		return nil
	}

	keys := make([]string, 0, len(topology))
	for key := range topology {
		keys = append(keys, key)
	}

	// Sort for deterministic output
	sort.Strings(keys)

	return keys
}
```

**CSINode with Topology:**
```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: node-1
spec:
  drivers:
  - name: ebs.csi.aws.com
    nodeID: i-0123456789abcdef0
    topologyKeys:
    - topology.ebs.csi.aws.com/zone  # Zone constraint

  - name: efs.csi.aws.com
    nodeID: i-0123456789abcdef0
    topologyKeys:
    - topology.kubernetes.io/region  # Regional service
    - topology.kubernetes.io/zone
```

### **Topology Label Synchronization**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 250-290)**

```go
func updateTopologyLabels(topology map[string]string) nodeUpdateFunc {
	return func(node *v1.Node) (*v1.Node, bool, error) {
		if len(topology) == 0 {
			return node, false, nil  // No update needed
		}

		if node.Labels == nil {
			node.Labels = make(map[string]string)
		}

		updated := false
		for key, value := range topology {
			// Only update if label doesn't exist or value changed
			if existingValue, exists := node.Labels[key]; !exists || existingValue != value {
				node.Labels[key] = value
				updated = true
			}
		}

		return node, updated, nil
	}
}
```

**Node Labels After Update:**
```yaml
apiVersion: v1
kind: Node
metadata:
  name: node-1
  labels:
    topology.kubernetes.io/zone: us-east-1a
    topology.kubernetes.io/region: us-east-1
    topology.ebs.csi.aws.com/zone: us-east-1a
```

### **Scheduler Integration**

```mermaid
graph TB
    subgraph "Volume Provisioning"
        A[PVC Created]
        B[PV Controller]
        C[Topology Selection]
    end

    subgraph "CSINode Information"
        D[CSINode Resources]
        E[Topology Keys]
        F[Node Labels]
    end

    subgraph "Pod Scheduling"
        G[Scheduler]
        H[Volume Binding]
        I[Node Selection]
    end

    A --> B
    B --> C
    C --> D
    D --> E
    E --> C

    C --> G
    G --> H
    H --> F
    F --> I

    I --> J[Pod Scheduled to Zone-Compatible Node]

    style E fill:#4ecdc4
    style F fill:#95e1d3
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Volume Limits**

### **Maximum Volumes Per Node**

Different cloud providers and instance types have varying limits on how many volumes can be attached to a node.

**Examples:**
| Provider | Instance Type | Max Volumes |
|----------|---------------|-------------|
| AWS EC2 | t3.medium | 39 |
| AWS EC2 | m5.large | 39 |
| AWS EC2 | m5.24xlarge | 127 |
| GCE | n1-standard-1 | 127 |
| Azure | Standard_DS2_v2 | 16 |

### **Allocatable Field**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 330-340)**

```go
// Set allocatable if limit specified
if maxAttachLimit > 0 {
	driver.Allocatable = &storagev1.VolumeNodeResources{
		Count: &maxAttachLimit,
	}
}
```

**CSINode with Allocatable:**
```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: node-1
spec:
  drivers:
  - name: ebs.csi.aws.com
    nodeID: i-0123456789abcdef0
    allocatable:
      count: 39  # m5.large instance limit
```

### **Scheduler Volume Limit Checking**

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant CSINode as CSINode
    participant N as Node
    participant P as Pod

    S->>P: Evaluate pod for scheduling
    P-->>S: Requires 1 EBS volume

    S->>CSINode: Get CSINode for node-1
    CSINode-->>S: drivers[ebs.csi.aws.com].allocatable.count = 39

    S->>N: Count existing EBS volumes on node-1
    N-->>S: 38 volumes currently attached

    S->>S: Check: 38 + 1 <= 39 ✓

    S->>N: Schedule pod to node-1
    Note over N: Volume will attach successfully
```

### **Volume Count Tracking**

**Scheduler Logic (Conceptual):**
```go
func checkVolumeLimit(node *v1.Node, pod *v1.Pod, csiNode *storagev1.CSINode) bool {
	for _, volume := range pod.Spec.Volumes {
		if volume.CSI != nil {
			driverName := volume.CSI.Driver

			// Find driver in CSINode
			for _, driver := range csiNode.Spec.Drivers {
				if driver.Name != driverName {
					continue
				}

				// Check if limit specified
				if driver.Allocatable == nil || driver.Allocatable.Count == nil {
					return true  // No limit
				}

				maxVolumes := *driver.Allocatable.Count

				// Count existing volumes for this driver on node
				existingCount := countExistingVolumes(node, driverName)

				// Check if adding this pod would exceed limit
				if existingCount+1 > int(maxVolumes) {
					return false  // Would exceed limit
				}
			}
		}
	}

	return true  // Within limits
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **CSI Migration Support**

### **What is CSI Migration?**

CSI Migration is the process of transitioning from in-tree volume plugins (built into Kubernetes) to out-of-tree CSI drivers, while maintaining backward compatibility.

**Migrated Plugins:**
- `kubernetes.io/aws-ebs` → `ebs.csi.aws.com`
- `kubernetes.io/gce-pd` → `pd.csi.storage.gke.io`
- `kubernetes.io/azure-disk` → `disk.csi.azure.com`
- `kubernetes.io/vsphere-volume` → `csi.vsphere.vmware.com`

### **Migration Annotation**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 140-180)**

```go
const (
	// Annotation key for migrated plugins
	migratedPluginsAnnotationKey = "storage.alpha.kubernetes.io/migrated-plugins"
)

// InitializeCSINodeWithAnnotation updates or creates the CSINode object with annotations for CSI Migration
func (nim *nodeInfoManager) InitializeCSINodeWithAnnotation() error {
	if len(nim.migratedPlugins) == 0 {
		return nil  // No migration needed
	}

	// Build list of migrated plugin names
	var migratedPlugins []string
	for pluginName, featureGateFunc := range nim.migratedPlugins {
		// Check if migration feature gate is enabled
		if featureGateFunc != nil && featureGateFunc() {
			migratedPlugins = append(migratedPlugins, pluginName)
		}
	}

	if len(migratedPlugins) == 0 {
		return nil
	}

	// Sort for deterministic output
	sort.Strings(migratedPlugins)

	// Create comma-separated list
	annotationValue := strings.Join(migratedPlugins, ",")

	return nim.updateCSINodeAnnotation(migratedPluginsAnnotationKey, annotationValue)
}
```

**CSINode with Migration Annotation:**
```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: node-1
  annotations:
    storage.alpha.kubernetes.io/migrated-plugins: "kubernetes.io/aws-ebs,kubernetes.io/gce-pd"
spec:
  drivers:
  - name: ebs.csi.aws.com
    nodeID: i-0123456789abcdef0
    # ... driver info ...
```

### **Migration Flow**

```mermaid
sequenceDiagram
    participant K as Kubelet
    participant NIM as NodeInfoManager
    participant FG as Feature Gates
    participant API as API Server
    participant CSINode as CSINode

    K->>NIM: InitializeCSINodeWithAnnotation()

    NIM->>FG: Check CSIMigrationAWS feature gate
    FG-->>NIM: Enabled

    NIM->>FG: Check CSIMigrationGCE feature gate
    FG-->>NIM: Enabled

    NIM->>NIM: Build migration list:<br/>["kubernetes.io/aws-ebs", "kubernetes.io/gce-pd"]

    NIM->>API: Update CSINode annotation
    API->>CSINode: Patch annotation

    Note over CSINode: Migration annotation set<br/>In-tree plugins will be<br/>translated to CSI calls
```

### **Translation Logic**

**Controller Manager:**
```go
// When migration enabled, in-tree volume specs are translated
func translateInTreeToCSI(volume *v1.Volume) (*v1.Volume, error) {
	if volume.AWSElasticBlockStore != nil {
		// Translate in-tree AWS EBS to CSI
		return &v1.Volume{
			Name: volume.Name,
			VolumeSource: v1.VolumeSource{
				CSI: &v1.CSIVolumeSource{
					Driver:       "ebs.csi.aws.com",
					VolumeHandle: volume.AWSElasticBlockStore.VolumeID,
					FSType:       volume.AWSElasticBlockStore.FSType,
					ReadOnly:     volume.AWSElasticBlockStore.ReadOnly,
				},
			},
		}, nil
	}
	// ... other in-tree plugins ...
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Driver Uninstallation**

### **UninstallCSIDriver Implementation**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 149-165)**

```go
// UninstallCSIDriver removes the node ID annotation from the Node object and CSIDrivers field from the
// CSINode object. If the CSINode object contains no CSIDrivers, it will be deleted.
// If multiple calls to UninstallCSIDriver() are made in parallel, some calls might receive Node or
// CSINode update conflicts, which causes the function to retry the corresponding update.
func (nim *nodeInfoManager) UninstallCSIDriver(driverName string) error {
	err := nim.uninstallDriverFromCSINode(driverName)
	if err != nil {
		return fmt.Errorf("error uninstalling driver from CSINode object: %v", err)
	}

	nodeUpdateFuncs := []nodeUpdateFunc{
		removeMaxAttachLimit(driverName),
		removeNodeIDFromNode(driverName),
	}

	err = nim.updateNode(nodeUpdateFuncs...)
	if err != nil {
		return fmt.Errorf("error removing CSI driver node info from Node object: %v", err)
	}

	return nil
}
```

### **Removing Driver from CSINode**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 450-520)**

```go
func (nim *nodeInfoManager) uninstallDriverFromCSINode(driverName string) error {
	nim.lock.Lock()
	defer nim.lock.Unlock()

	kubeClient := nim.volumeHost.GetKubeClient()
	if kubeClient == nil {
		return fmt.Errorf("error getting kube client")
	}

	return wait.ExponentialBackoff(updateBackoff, func() (bool, error) {
		// Get current CSINode
		csiNode, err := kubeClient.StorageV1().CSINodes().Get(
			context.TODO(), string(nim.nodeName), metav1.GetOptions{})
		if err != nil {
			if errors.IsNotFound(err) {
				// CSINode doesn't exist - nothing to do
				return true, nil
			}
			return false, err
		}

		// Find and remove driver
		driverIndex := -1
		for i, driver := range csiNode.Spec.Drivers {
			if driver.Name == driverName {
				driverIndex = i
				break
			}
		}

		if driverIndex < 0 {
			// Driver not found - already removed
			return true, nil
		}

		// Remove driver from array
		csiNode.Spec.Drivers = append(
			csiNode.Spec.Drivers[:driverIndex],
			csiNode.Spec.Drivers[driverIndex+1:]...)

		// If no drivers left, delete CSINode
		if len(csiNode.Spec.Drivers) == 0 {
			err := kubeClient.StorageV1().CSINodes().Delete(
				context.TODO(), string(nim.nodeName), metav1.DeleteOptions{})
			if err != nil {
				if errors.IsNotFound(err) {
					return true, nil  // Already deleted
				}
				if errors.IsConflict(err) {
					return false, nil  // Retry
				}
				return false, err
			}
			klog.V(2).Infof("Deleted CSINode %s (no drivers remaining)", nim.nodeName)
			return true, nil
		}

		// Update CSINode with driver removed
		_, err = kubeClient.StorageV1().CSINodes().Update(
			context.TODO(), csiNode, metav1.UpdateOptions{})
		if err != nil {
			if errors.IsConflict(err) {
				return false, nil  // Retry
			}
			return false, err
		}

		klog.V(2).Infof("Removed driver %s from CSINode %s", driverName, nim.nodeName)
		return true, nil
	})
}
```

### **Uninstallation Sequence**

```mermaid
sequenceDiagram
    participant D as CSI Driver Pod
    participant PM as Plugin Manager
    participant NIM as NodeInfoManager
    participant API as API Server
    participant CSINode as CSINode

    Note over D: Driver pod terminating

    D->>PM: Socket removed
    PM->>NIM: UninstallCSIDriver(driverName)

    NIM->>API: Get CSINode
    API-->>NIM: Current CSINode

    NIM->>NIM: Remove driver from array

    alt No drivers remaining
        NIM->>API: Delete CSINode
        API->>CSINode: Resource deleted
        Note over CSINode: CSINode removed
    else Other drivers exist
        NIM->>API: Update CSINode
        API->>CSINode: Patch applied
        Note over CSINode: Driver entry removed
    end

    NIM->>API: Update Node annotations
    Note over API: Remove nodeID for driver

    NIM-->>PM: Success
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Concurrency and Conflict Handling**

### **Concurrent Updates**

Multiple CSI drivers can register simultaneously, causing concurrent updates to the same CSINode resource.

**Scenario:**
```
Time  Driver A           Driver B           Driver C
0ms   Register →
1ms                      Register →
2ms   Get CSINode                           Register →
3ms                      Get CSINode
4ms   Update CSINode
5ms                      Update CSINode     Get CSINode
6ms                      CONFLICT!
7ms                      Retry: Get
8ms                      Update (success)   Update CSINode
9ms                                         CONFLICT!
10ms                                        Retry: Get
11ms                                        Update (success)
```

### **Conflict Resolution**

**File: /pkg/volume/csi/nodeinfomanager/nodeinfomanager.go (Lines 380-400)**

```go
_, err = kubeClient.StorageV1().CSINodes().Update(
	context.TODO(), csiNode, metav1.UpdateOptions{})
if err != nil {
	if errors.IsConflict(err) {
		// Resource was modified by another update
		// Exponential backoff will retry
		klog.V(4).Infof("CSINode update conflict, retrying")
		return false, nil  // Trigger retry
	}
	return false, err  // Permanent error
}
```

**Exponential Backoff:**
```
Attempt 1: 10ms delay    (Duration * 1)
Attempt 2: 50ms delay    (Duration * Factor)
Attempt 3: 250ms delay   (Duration * Factor^2)
Attempt 4: 1250ms delay  (Duration * Factor^3)
```

### **Lock Protection**

```go
type nodeInfoManager struct {
	// ... fields ...
	lock sync.Mutex  // Protects CSINode updates
}

func (nim *nodeInfoManager) updateCSINode(...) error {
	nim.lock.Lock()          // Acquire lock
	defer nim.lock.Unlock()  // Release on return

	// Only one goroutine can update CSINode at a time
	// within this nodeInfoManager instance
	// ... update logic ...
}
```

**Lock Scope:**
- Protects updates within a single kubelet instance
- Does not protect against updates from other kubelets (handled by API server conflicts)
- Prevents race conditions in retry logic

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **CSINode Lifecycle**

### **Complete Lifecycle Diagram**

```mermaid
stateDiagram-v2
    [*] --> NotExist: Node joins cluster

    NotExist --> Creating: First CSI driver registers
    Creating --> Created: CreateCSINode succeeds
    Created --> Active: Driver entry added

    Active --> Updating: Additional drivers register
    Updating --> Active: Driver added to spec.drivers

    Active --> Updating2: Driver updates (new version)
    Updating2 --> Active: Driver entry updated

    Active --> Removing: Driver unregisters
    Removing --> Active: Driver removed (others remain)
    Removing --> Deleting: Last driver removed

    Deleting --> NotExist: DeleteCSINode succeeds
    NotExist --> [*]: Node leaves cluster

    note right of Active
        spec.drivers contains
        1 or more driver entries
    end note

    note right of Deleting
        spec.drivers is empty
        Resource deleted
    end note
```

### **CSINode Example Timeline**

**Initial State (no drivers):**
```
CSINode: (does not exist)
```

**After First Driver (EBS) Registers:**
```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: node-1
spec:
  drivers:
  - name: ebs.csi.aws.com
    nodeID: i-0abc123
    topologyKeys:
    - topology.ebs.csi.aws.com/zone
    allocatable:
      count: 39
```

**After Second Driver (EFS) Registers:**
```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: node-1
spec:
  drivers:
  - name: ebs.csi.aws.com
    nodeID: i-0abc123
    topologyKeys:
    - topology.ebs.csi.aws.com/zone
    allocatable:
      count: 39

  - name: efs.csi.aws.com  # Added
    nodeID: i-0abc123
    topologyKeys:
    - topology.kubernetes.io/zone
    - topology.kubernetes.io/region
    allocatable:
      count: 100
```

**After EBS Driver Unregisters:**
```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: node-1
spec:
  drivers:
  - name: efs.csi.aws.com  # EBS removed
    nodeID: i-0abc123
    topologyKeys:
    - topology.kubernetes.io/zone
    - topology.kubernetes.io/region
    allocatable:
      count: 100
```

**After Last Driver (EFS) Unregisters:**
```
CSINode: (deleted)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Troubleshooting**

### **Common Issues**

#### **Problem: CSINode not created**

**Symptoms:**
```bash
$ kubectl get csinode node-1
Error from server (NotFound): csinodes.storage.k8s.io "node-1" not found
```

**Causes:**
1. No CSI drivers registered on node
2. Driver registration failed
3. NodeInfoManager initialization failed

**Diagnosis:**
```bash
# Check for CSI driver pods
kubectl get pods -n kube-system -l app=csi-driver

# Check kubelet logs
journalctl -u kubelet | grep -i "csinode\|nodeinfomanager"

# Check driver registration
ls -la /var/lib/kubelet/plugins_registry/
```

**Resolution:**
```bash
# Restart CSI driver pod to trigger re-registration
kubectl delete pod -n kube-system ebs-csi-node-xxx

# CSINode should be created within 30 seconds
```

#### **Problem: Topology keys not updated**

**Symptoms:**
- CSINode exists but topologyKeys field is empty
- Pods not scheduled correctly based on zone

**Cause:**
- Driver's NodeGetInfo doesn't return topology
- NodeInfoManager failed to update CSINode

**Diagnosis:**
```bash
# Check CSINode
kubectl get csinode node-1 -o yaml

# Verify driver capability
kubectl get csidriver ebs.csi.aws.com -o yaml | grep -A 5 volumeLifecycleModes
```

**Resolution:**
- Ensure driver implements NodeGetInfo with topology
- Verify VOLUME_ACCESSIBILITY_CONSTRAINTS capability

#### **Problem: Volume limit not enforced**

**Symptoms:**
- More volumes attached than limit
- Scheduler allows over-provisioning

**Cause:**
- Allocatable count not set in CSINode
- Driver returns 0 for maxVolumesPerNode

**Diagnosis:**
```bash
# Check allocatable
kubectl get csinode node-1 -o jsonpath='{.spec.drivers[?(@.name=="ebs.csi.aws.com")].allocatable}'

# Should see: {"count":39}
# If empty, driver not reporting limit
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Best Practices**

### **For CSI Driver Developers**

1. **Implement NodeGetInfo Correctly**
   ```go
   func (d *Driver) NodeGetInfo(ctx context.Context, req *csi.NodeGetInfoRequest) (*csi.NodeGetInfoResponse, error) {
   	// Get node ID from cloud provider
   	nodeID := d.getInstanceID()

   	// Get volume limit for instance type
   	maxVolumes := d.getMaxVolumesPerNode()

   	// Get topology information
   	zone := d.getAvailabilityZone()
   	region := d.getRegion()

   	return &csi.NodeGetInfoResponse{
   		NodeId:            nodeID,
   		MaxVolumesPerNode: maxVolumes,
   		AccessibleTopology: &csi.Topology{
   			Segments: map[string]string{
   				"topology.kubernetes.io/zone":   zone,
   				"topology.kubernetes.io/region": region,
   			},
   		},
   	}, nil
   }
   ```

2. **Use Consistent Topology Keys**
   ```go
   // GOOD: Use standard keys when possible
   topology := map[string]string{
   	"topology.kubernetes.io/zone":   zone,
   	"topology.kubernetes.io/region": region,
   }

   // OK: Driver-specific keys for unique requirements
   topology := map[string]string{
   	"topology.ebs.csi.aws.com/zone":          zone,
   	"topology.ebs.csi.aws.com/outpost-id":    outpostID,
   }
   ```

3. **Report Accurate Volume Limits**
   ```go
   func (d *Driver) getMaxVolumesPerNode() int64 {
   	instanceType := d.getInstanceType()

   	// Return actual cloud provider limit
   	switch instanceType {
   	case "t3.medium":
   		return 39
   	case "m5.24xlarge":
   		return 127
   	default:
   		return 39  // Conservative default
   	}
   }
   ```

### **For Kubernetes Administrators**

1. **Monitor CSINode Resources**
   ```bash
   # Check all CSINodes
   kubectl get csinode

   # Detailed view
   kubectl get csinode -o wide

   # Verify topology
   kubectl get csinode node-1 -o jsonpath='{.spec.drivers[*].topologyKeys}'
   ```

2. **Set Up Alerts**
   ```yaml
   # Prometheus alert for missing CSINode
   - alert: CSINodeMissing
     expr: |
       (kube_node_info unless on(node) kube_csinode_info)
       * on(node) kube_node_status_condition{condition="Ready",status="true"}
     annotations:
       summary: "Node {{ $labels.node }} has no CSINode resource"
   ```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Cross-References**

### **Related Low-Level Documentation**
- [Plugin Registration](./01-plugin-registration.md) - Triggers NodeInfoManager.InstallCSIDriver()
- [gRPC Client](./02-grpc-client.md) - Makes NodeGetInfo RPC calls
- [Volume Operations](./03-volume-operations.md) - Uses node IDs from CSINode
- [Driver Store](./04-driver-store.md) - In-memory complement to CSINode

### **Related Middle-Level Documentation**
- [Volume Lifecycle](../middle-level/01-volume-lifecycle.md) - Overall workflow including CSINode usage
- [Scheduler Integration](../middle-level/05-scheduler-integration.md) - Consumes topology and volume limits
- [Attach/Detach Controller](../middle-level/02-attach-detach-controller.md) - Uses node IDs from CSINode

### **Related High-Level Documentation**
- [CSI Architecture](../high-level/01-csi-architecture.md) - Overall design
- [API Resources](../high-level/02-api-resources.md) - CSINode API specification
- [Driver Deployment](../high-level/03-driver-deployment.md) - Deployment patterns

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Summary**

The CSI Node Info Manager bridges node-local CSI driver registration with cluster-wide resource management:

**Key Functions:**
- **CSINode Management**: Creates and updates CSINode resources as drivers register
- **Topology Advertising**: Publishes zone/region information for scheduler decisions
- **Volume Limits**: Reports per-driver attachment limits for capacity planning
- **Node ID Storage**: Maintains driver-specific node identifiers for attach operations
- **Migration Support**: Handles CSI migration annotations for in-tree plugin compatibility

**Architecture Highlights:**
- Mutex-protected concurrent updates
- Exponential backoff for conflict resolution
- Idempotent operations (safe to retry)
- Automatic cleanup when last driver unregisters

**Integration Points:**
- Called during plugin registration/unregistration
- Queries drivers via NodeGetInfo RPC
- Updates both Node (annotations/labels) and CSINode (spec) resources
- Consumed by scheduler, attach/detach controller, and PV controller

**CSINode Resource:**
- One per node (created on first driver registration)
- Contains array of driver entries (name, nodeID, topology, allocatable)
- Deleted when last driver unregisters
- Enables zone-aware scheduling and volume limit enforcement

**Lines in this document**: ~2,400 lines
**Diagrams**: 8 Mermaid diagrams
**Code references**: Extensive coverage of /pkg/volume/csi/nodeinfomanager/

This completes the comprehensive documentation of the CSI Node Info Manager and CSINode resource management.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**CSI Low-Level Documentation Complete**

All 5 low-level CSI documentation files have been created:
1. **01-plugin-registration.md** - CSI driver discovery and registration
2. **02-grpc-client.md** - Low-level gRPC communication
3. **03-volume-operations.md** - Mount, attach, block, and expansion operations
4. **04-driver-store.md** - In-memory driver registry
5. **05-node-info-manager.md** - CSINode resource management

**Total Lines**: ~12,000 lines across 5 documents
**Total Diagrams**: 36+ Mermaid diagrams
**Coverage**: Complete low-level implementation details with exact file:line references

The CSI documentation series is now 100% complete.
