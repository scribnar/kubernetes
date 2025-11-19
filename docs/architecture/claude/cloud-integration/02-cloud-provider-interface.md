# **Cloud Provider Interface Architecture**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Comprehensive guide to implementing the Kubernetes Cloud Provider Interface

**Target Audience**:
- Cloud provider developers building new integrations
- Platform engineers customizing cloud provider behavior
- SREs understanding cloud provider internals
- Contributors to cloud-provider ecosystem

**Scope**: Interface definitions, implementation patterns, helper utilities, testing, and best practices

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Understanding the Cloud Provider Interface**

### **Design Philosophy**

The Cloud Provider Interface is a **plugin architecture** that allows Kubernetes to integrate with any cloud platform without requiring changes to core components.

**Key Principles**:

1. **Abstraction** - Hide cloud-specific details behind common interfaces
2. **Opt-in Support** - Providers implement only the interfaces they support
3. **Graceful Degradation** - Controllers skip unsupported features
4. **Immutability** - Node objects and services passed to providers are read-only
5. **Context Propagation** - All methods accept context for cancellation/timeout

### **Interface Hierarchy**

```
cloudprovider.Interface
├── Initialize(clientBuilder, stop)
├── ProviderName() string
├── HasClusterID() bool
│
├── LoadBalancer() (LoadBalancer, bool)
│   ├── GetLoadBalancer()
│   ├── GetLoadBalancerName()
│   ├── EnsureLoadBalancer()
│   ├── UpdateLoadBalancer()
│   └── EnsureLoadBalancerDeleted()
│
├── InstancesV2() (InstancesV2, bool)  ← RECOMMENDED
│   ├── InstanceExists()
│   ├── InstanceShutdown()
│   └── InstanceMetadata()
│
├── Instances() (Instances, bool)  ← DEPRECATED
│   ├── NodeAddresses()
│   ├── NodeAddressesByProviderID()
│   ├── InstanceID()
│   ├── InstanceType()
│   └── ...
│
├── Routes() (Routes, bool)
│   ├── ListRoutes()
│   ├── CreateRoute()
│   └── DeleteRoute()
│
├── Zones() (Zones, bool)  ← DEPRECATED
│   ├── GetZone()
│   ├── GetZoneByProviderID()
│   └── GetZoneByNodeName()
│
└── Clusters() (Clusters, bool)  ← OPTIONAL
    ├── ListClusters()
    └── Master()
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Core Interface Definition**

### **Main Interface**

**File**: `staging/src/k8s.io/cloud-provider/cloud.go`

```go
// Interface is an abstract, pluggable interface for cloud providers
type Interface interface {
    // Initialize provides the cloud provider with a kubernetes client builder
    // and may spawn goroutines to perform housekeeping or run custom controllers
    // specific to the cloud provider.
    //
    // Any tasks started here should be cleaned up when the stop channel closes.
    Initialize(clientBuilder ControllerClientBuilder, stop <-chan struct{})

    // LoadBalancer returns a balancer interface if supported.
    // Also returns true if the interface is supported, false otherwise.
    LoadBalancer() (LoadBalancer, bool)

    // Instances returns an instances interface if supported.
    // DEPRECATED: Use InstancesV2
    Instances() (Instances, bool)

    // InstancesV2 is an implementation for instances and should only be
    // implemented by external cloud providers.
    //
    // Implementing InstancesV2 is behaviorally identical to Instances but is
    // optimized to significantly reduce API calls to the cloud provider when
    // registering and syncing nodes.
    //
    // Implementation of this interface will disable calls to the Zones interface.
    InstancesV2() (InstancesV2, bool)

    // Zones returns a zones interface if supported.
    // DEPRECATED: Use InstancesV2.InstanceMetadata for zone/region information.
    Zones() (Zones, bool)

    // Clusters returns a clusters interface if supported.
    Clusters() (Clusters, bool)

    // Routes returns a routes interface if supported.
    Routes() (Routes, bool)

    // ProviderName returns the cloud provider ID.
    ProviderName() string

    // HasClusterID returns true if a ClusterID is required and set.
    HasClusterID() bool
}
```

### **Interface Support Pattern**

All accessor methods return `(SpecificInterface, bool)`:

```go
// Implementation pattern
func (c *MyCloudProvider) LoadBalancer() (cloudprovider.LoadBalancer, bool) {
    // Supported
    return c, true  // Provider implements the interface

    // Not supported
    return nil, false
}

// Usage pattern in controllers
func startServiceController(cloud cloudprovider.Interface) {
    lb, supported := cloud.LoadBalancer()
    if !supported {
        klog.Info("LoadBalancer not supported, skipping service controller")
        return
    }
    // Use lb interface
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚀 InstancesV2 Interface (Recommended)**

### **Interface Definition**

**File**: `staging/src/k8s.io/cloud-provider/cloud.go`

```go
// InstancesV2 is an abstract, pluggable interface for cloud provider instances.
// Unlike the Instances interface, InstancesV2 returns all instance information
// through InstanceMetadata, significantly reducing API calls.
type InstancesV2 interface {
    // InstanceExists returns true if the instance for the given node exists
    // according to the cloud provider.
    //
    // Use the node.name or node.spec.providerID field to find the node
    // in the cloud provider.
    InstanceExists(ctx context.Context, node *v1.Node) (bool, error)

    // InstanceShutdown returns true if the instance is shutdown according
    // to the cloud provider.
    //
    // Use the node.name or node.spec.providerID field to find the node
    // in the cloud provider.
    InstanceShutdown(ctx context.Context, node *v1.Node) (bool, error)

    // InstanceMetadata returns the instance's metadata.
    //
    // The values returned in InstanceMetadata are translated into specific
    // fields and labels in the Node object on registration.
    //
    // Implementations should always check node.spec.providerID first when
    // trying to discover the instance for a given node. In cases where
    // node.spec.providerID is empty, implementations can use other
    // properties of the node like its name, labels and annotations.
    InstanceMetadata(ctx context.Context, node *v1.Node) (*InstanceMetadata, error)
}
```

### **InstanceMetadata Structure**

```go
// InstanceMetadata contains metadata about a specific instance
type InstanceMetadata struct {
    // ProviderID is a unique ID used to identify an instance on the cloud provider.
    // The ProviderID set here will be set on the node's spec.providerID field.
    //
    // Format: <provider-name>://<instance-id>
    // Examples:
    //   aws:///us-east-1a/i-0123456789abcdef0
    //   gce://project-id/us-central1-a/instance-name
    //   azure:///subscriptions/sub/resourceGroups/rg/providers/.../vm-name
    ProviderID string

    // InstanceType is the instance's type.
    //
    // Sets labels:
    //   - node.kubernetes.io/instance-type
    //   - beta.kubernetes.io/instance-type (DEPRECATED)
    InstanceType string

    // NodeAddresses contains information for the instance's addresses.
    //
    // The node addresses returned here will be set on the node's
    // status.addresses field.
    NodeAddresses []v1.NodeAddress

    // Zone is the zone that the instance is in.
    //
    // Sets labels:
    //   - topology.kubernetes.io/zone
    //   - failure-domain.beta.kubernetes.io/zone (DEPRECATED)
    Zone string

    // Region is the region that the instance is in.
    //
    // Sets labels:
    //   - topology.kubernetes.io/region
    //   - failure-domain.beta.kubernetes.io/region (DEPRECATED)
    Region string

    // AdditionalLabels is a map of additional labels provided by
    // the cloud provider.
    //
    // When provided, they will be applied to the node and enable
    // cloud providers to label nodes with information that may be
    // valuable to that provider.
    AdditionalLabels map[string]string
}
```

### **Node Address Types**

```go
const (
    NodeHostName    NodeAddressType = "Hostname"
    NodeInternalIP  NodeAddressType = "InternalIP"
    NodeExternalIP  NodeAddressType = "ExternalIP"
    NodeInternalDNS NodeAddressType = "InternalDNS"
    NodeExternalDNS NodeAddressType = "ExternalDNS"
)
```

### **Implementation Example**

```go
type MyCloudProvider struct {
    client *cloud.Client
}

func (m *MyCloudProvider) InstancesV2() (cloudprovider.InstancesV2, bool) {
    return m, true
}

func (m *MyCloudProvider) InstanceExists(ctx context.Context, node *v1.Node) (bool, error) {
    // Prefer ProviderID lookup
    instanceID := parseProviderID(node.Spec.ProviderID)
    if instanceID == "" {
        // Fallback to node name
        instanceID = string(node.Name)
    }

    instance, err := m.client.GetInstance(ctx, instanceID)
    if err != nil {
        if isNotFoundError(err) {
            return false, nil  // Instance doesn't exist
        }
        return false, err  // Transient error
    }

    // Instance exists (even if stopped/shutdown)
    return instance != nil, nil
}

func (m *MyCloudProvider) InstanceShutdown(ctx context.Context, node *v1.Node) (bool, error) {
    instanceID := parseProviderID(node.Spec.ProviderID)
    if instanceID == "" {
        instanceID = string(node.Name)
    }

    instance, err := m.client.GetInstance(ctx, instanceID)
    if err != nil {
        return false, err
    }

    // Check instance state
    switch instance.State {
    case "stopped", "stopping", "terminated", "terminating":
        return true, nil
    default:
        return false, nil
    }
}

func (m *MyCloudProvider) InstanceMetadata(ctx context.Context, node *v1.Node) (*cloudprovider.InstanceMetadata, error) {
    instanceID := parseProviderID(node.Spec.ProviderID)
    if instanceID == "" {
        instanceID = string(node.Name)
    }

    instance, err := m.client.GetInstance(ctx, instanceID)
    if err != nil {
        if isNotFoundError(err) {
            return nil, cloudprovider.InstanceNotFound
        }
        return nil, err
    }

    return &cloudprovider.InstanceMetadata{
        ProviderID:   fmt.Sprintf("mycloud:///%s", instance.ID),
        InstanceType: instance.Type,
        Zone:         instance.Zone,
        Region:       instance.Region,
        NodeAddresses: []v1.NodeAddress{
            {Type: v1.NodeInternalIP, Address: instance.PrivateIP},
            {Type: v1.NodeExternalIP, Address: instance.PublicIP},
            {Type: v1.NodeHostName, Address: instance.Hostname},
        },
        AdditionalLabels: map[string]string{
            "mycloud.com/instance-family": instance.Family,
        },
    }, nil
}
```

### **Why InstancesV2 Over Instances**

| **Aspect** | **Instances (Deprecated)** | **InstancesV2 (Recommended)** |
|------------|---------------------------|-------------------------------|
| **API Calls** | Multiple calls per sync | Single call for all metadata |
| **Efficiency** | 5-10 calls per node | 1 call per node |
| **Zone Information** | Requires separate Zones interface | Included in InstanceMetadata |
| **Additional Labels** | Not supported | Supported via AdditionalLabels |
| **Design** | Legacy per-attribute methods | Modern consolidated approach |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔌 LoadBalancer Interface**

### **Interface Definition**

**File**: `staging/src/k8s.io/cloud-provider/cloud.go`

```go
// LoadBalancer is an abstract, pluggable interface for load balancers
type LoadBalancer interface {
    // GetLoadBalancer returns whether the specified load balancer exists,
    // and if so, what its status is.
    //
    // Implementations must treat the *v1.Service parameter as read-only
    // and not modify it.
    //
    // Parameter 'clusterName' is the name of the cluster as presented
    // to kube-controller-manager.
    GetLoadBalancer(ctx context.Context, clusterName string, service *v1.Service) (
        status *v1.LoadBalancerStatus, exists bool, err error)

    // GetLoadBalancerName returns the name of the load balancer.
    //
    // Implementations must treat the *v1.Service parameter as read-only
    // and not modify it.
    GetLoadBalancerName(ctx context.Context, clusterName string, service *v1.Service) string

    // EnsureLoadBalancer creates a new load balancer 'name', or updates
    // the existing one.
    //
    // Returns the status of the balancer. Implementations must treat the
    // *v1.Service and *v1.Node parameters as read-only and not modify them.
    //
    // Parameter 'clusterName' is the name of the cluster as presented
    // to kube-controller-manager.
    //
    // Implementations may return a (possibly wrapped) api.RetryError to
    // enforce backing off at a fixed duration. This can be used for cases
    // like when the load balancer is not ready yet (e.g., still being
    // provisioned) and polling at a fixed rate is preferred over backing
    // off exponentially in order to minimize latency.
    EnsureLoadBalancer(ctx context.Context, clusterName string, service *v1.Service,
        nodes []*v1.Node) (*v1.LoadBalancerStatus, error)

    // UpdateLoadBalancer updates hosts under the specified load balancer.
    //
    // Implementations must treat the *v1.Service and *v1.Node parameters
    // as read-only and not modify them.
    //
    // Parameter 'clusterName' is the name of the cluster as presented
    // to kube-controller-manager.
    UpdateLoadBalancer(ctx context.Context, clusterName string, service *v1.Service,
        nodes []*v1.Node) error

    // EnsureLoadBalancerDeleted deletes the specified load balancer if it
    // exists, returning nil if the load balancer specified either didn't
    // exist or was successfully deleted.
    //
    // This construction is useful because many cloud providers' load
    // balancers have multiple underlying components, meaning a Get could
    // say that the LB doesn't exist even if some part of it is still
    // laying around.
    //
    // Implementations must treat the *v1.Service parameter as read-only
    // and not modify it.
    //
    // Parameter 'clusterName' is the name of the cluster as presented
    // to kube-controller-manager.
    EnsureLoadBalancerDeleted(ctx context.Context, clusterName string,
        service *v1.Service) error
}
```

### **LoadBalancerStatus Structure**

```go
type LoadBalancerStatus struct {
    // Ingress is a list containing ingress points for the load-balancer.
    // Traffic intended for the service should be sent to these ingress points.
    Ingress []LoadBalancerIngress
}

type LoadBalancerIngress struct {
    // IP is set for load-balancer ingress points that are IP based
    IP string

    // Hostname is set for load-balancer ingress points that are DNS based
    Hostname string

    // IPMode specifies how the load-balancer IP behaves
    // VIP = service traffic sent to this IP routes to node with Service pods
    // Proxy = service traffic sent to this IP routes through proxy to Service pods
    IPMode *LoadBalancerIPMode

    // Ports is a list of records of service ports
    Ports []PortStatus
}
```

### **Implementation Example**

```go
func (m *MyCloudProvider) LoadBalancer() (cloudprovider.LoadBalancer, bool) {
    return m, true
}

func (m *MyCloudProvider) GetLoadBalancerName(ctx context.Context,
    clusterName string, service *v1.Service) string {

    return fmt.Sprintf("%s-%s-%s", clusterName, service.Namespace, service.Name)
}

func (m *MyCloudProvider) GetLoadBalancer(ctx context.Context,
    clusterName string, service *v1.Service) (*v1.LoadBalancerStatus, bool, error) {

    name := m.GetLoadBalancerName(ctx, clusterName, service)

    lb, err := m.client.GetLoadBalancer(ctx, name)
    if err != nil {
        if isNotFoundError(err) {
            return nil, false, nil  // Doesn't exist
        }
        return nil, false, err  // Error
    }

    status := &v1.LoadBalancerStatus{
        Ingress: []v1.LoadBalancerIngress{
            {IP: lb.VIP},
        },
    }

    return status, true, nil
}

func (m *MyCloudProvider) EnsureLoadBalancer(ctx context.Context,
    clusterName string, service *v1.Service, nodes []*v1.Node) (*v1.LoadBalancerStatus, error) {

    name := m.GetLoadBalancerName(ctx, clusterName, service)

    // Build backend pool from nodes
    var backends []string
    for _, node := range nodes {
        for _, addr := range node.Status.Addresses {
            if addr.Type == v1.NodeInternalIP {
                backends = append(backends, addr.Address)
            }
        }
    }

    // Build port configuration
    var ports []PortConfig
    for _, port := range service.Spec.Ports {
        ports = append(ports, PortConfig{
            Protocol:   string(port.Protocol),
            Port:       port.Port,
            TargetPort: port.NodePort,
        })
    }

    // Create or update load balancer
    lb, err := m.client.EnsureLoadBalancer(ctx, &LoadBalancerSpec{
        Name:     name,
        Backends: backends,
        Ports:    ports,
        SourceRanges: getSourceRanges(service),
    })
    if err != nil {
        // Return RetryError for provisioning delays
        if isProvisioningError(err) {
            return nil, cloudproviderapi.NewRetryError(
                "load balancer is provisioning",
                30*time.Second,
            )
        }
        return nil, err
    }

    return &v1.LoadBalancerStatus{
        Ingress: []v1.LoadBalancerIngress{
            {IP: lb.VIP},
        },
    }, nil
}

func (m *MyCloudProvider) UpdateLoadBalancer(ctx context.Context,
    clusterName string, service *v1.Service, nodes []*v1.Node) error {

    name := m.GetLoadBalancerName(ctx, clusterName, service)

    // Build updated backend pool
    var backends []string
    for _, node := range nodes {
        for _, addr := range node.Status.Addresses {
            if addr.Type == v1.NodeInternalIP {
                backends = append(backends, addr.Address)
            }
        }
    }

    return m.client.UpdateLoadBalancerBackends(ctx, name, backends)
}

func (m *MyCloudProvider) EnsureLoadBalancerDeleted(ctx context.Context,
    clusterName string, service *v1.Service) error {

    name := m.GetLoadBalancerName(ctx, clusterName, service)

    err := m.client.DeleteLoadBalancer(ctx, name)
    if err != nil {
        if isNotFoundError(err) {
            return nil  // Already deleted
        }
        return err
    }

    return nil
}
```

### **Service Annotations for LoadBalancer**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    # Cloud provider specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/gcp-load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
    - 10.0.0.0/8
  ports:
  - port: 80
    targetPort: 8080
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🛣️ Routes Interface**

### **Interface Definition**

**File**: `staging/src/k8s.io/cloud-provider/cloud.go`

```go
// Routes is an abstract, pluggable interface for advanced routing rules
type Routes interface {
    // ListRoutes lists all managed routes that belong to the specified clusterName
    ListRoutes(ctx context.Context, clusterName string) ([]*Route, error)

    // CreateRoute creates the described managed route.
    // route.Name will be ignored, although the cloud-provider may use nameHint
    // to create a more user-meaningful name.
    CreateRoute(ctx context.Context, clusterName string, nameHint string,
        route *Route) error

    // DeleteRoute deletes the specified managed route.
    // Route should be as returned by ListRoutes.
    DeleteRoute(ctx context.Context, clusterName string, route *Route) error
}

// Route is a representation of an advanced routing rule
type Route struct {
    // Name is the name of the routing rule in the cloud-provider.
    // It will be ignored in a Create (although nameHint may influence it).
    Name string

    // TargetNode is the NodeName of the target instance.
    TargetNode types.NodeName

    // EnableNodeAddresses is a feature gate for TargetNodeAddresses.
    // If false, ignore TargetNodeAddresses.
    EnableNodeAddresses bool

    // TargetNodeAddresses are the Node IPs of the target Node.
    TargetNodeAddresses []v1.NodeAddress

    // DestinationCIDR is the CIDR format IP range that this routing rule
    // applies to.
    DestinationCIDR string

    // Blackhole is set to true if this is a blackhole route.
    // The node controller will delete the route if it is in the managed range.
    Blackhole bool
}
```

### **Implementation Example**

```go
func (m *MyCloudProvider) Routes() (cloudprovider.Routes, bool) {
    return m, true
}

func (m *MyCloudProvider) ListRoutes(ctx context.Context, clusterName string) (
    []*cloudprovider.Route, error) {

    // Get routes from cloud provider
    cloudRoutes, err := m.client.ListRoutes(ctx, m.routeTableID)
    if err != nil {
        return nil, err
    }

    var routes []*cloudprovider.Route
    for _, r := range cloudRoutes {
        // Filter routes by cluster tag
        if !hasTag(r.Tags, "kubernetes.io/cluster/"+clusterName) {
            continue
        }

        routes = append(routes, &cloudprovider.Route{
            Name:            r.ID,
            TargetNode:      types.NodeName(r.InstanceID),
            DestinationCIDR: r.DestinationCIDR,
            Blackhole:       r.State == "blackhole",
        })
    }

    return routes, nil
}

func (m *MyCloudProvider) CreateRoute(ctx context.Context, clusterName string,
    nameHint string, route *cloudprovider.Route) error {

    // Get instance ID for target node
    instanceID, err := m.getInstanceIDForNode(ctx, route.TargetNode)
    if err != nil {
        return err
    }

    // Create route in cloud provider
    return m.client.CreateRoute(ctx, &RouteSpec{
        RouteTableID:    m.routeTableID,
        DestinationCIDR: route.DestinationCIDR,
        InstanceID:      instanceID,
        Tags: map[string]string{
            "kubernetes.io/cluster/" + clusterName: "owned",
            "kubernetes.io/node":                   string(route.TargetNode),
        },
    })
}

func (m *MyCloudProvider) DeleteRoute(ctx context.Context, clusterName string,
    route *cloudprovider.Route) error {

    return m.client.DeleteRoute(ctx, m.routeTableID, route.DestinationCIDR)
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚠️ Error Handling**

### **Error Constants**

**File**: `staging/src/k8s.io/cloud-provider/cloud.go`

```go
var (
    // InstanceNotFound is used to indicate that the instance does not exist.
    // Should NOT be returned for instances that exist but are stopped/sleeping.
    InstanceNotFound = errors.New("instance not found")

    // DiskNotFound is used to indicate that a disk does not exist.
    DiskNotFound = errors.New("disk is not found")

    // NotImplemented is used to indicate that a method is not implemented
    // by the cloud provider.
    NotImplemented = errors.New("unimplemented")

    // ImplementedElsewhere is used to indicate that the cloud provider
    // manages load balancer handling separately.
    // Should NOT be returned from EnsureLoadBalancerDeleted.
    ImplementedElsewhere = errors.New("implemented by alternate to cloud provider")
)
```

### **RetryError for Service Reconciliation**

**File**: `staging/src/k8s.io/cloud-provider/api/retry_error.go`

```go
// RetryError can be returned from LoadBalancer methods to enforce
// fixed-duration retries instead of exponential backoff.
type RetryError struct {
    msg        string
    retryAfter time.Duration
}

// NewRetryError returns a RetryError.
func NewRetryError(msg string, retryAfter time.Duration) *RetryError {
    return &RetryError{
        msg:        msg,
        retryAfter: retryAfter,
    }
}

func (re *RetryError) Error() string {
    return re.msg
}

func (re *RetryError) RetryAfter() time.Duration {
    return re.retryAfter
}
```

### **Error Usage Patterns**

```go
// InstanceNotFound - Only for truly non-existent instances
func (m *MyCloudProvider) InstanceMetadata(ctx context.Context,
    node *v1.Node) (*cloudprovider.InstanceMetadata, error) {

    instance, err := m.client.GetInstance(ctx, parseProviderID(node.Spec.ProviderID))
    if err != nil {
        if isNotFoundError(err) {
            // Instance truly doesn't exist - will cause node deletion
            return nil, cloudprovider.InstanceNotFound
        }
        // Transient error - will be retried
        return nil, err
    }

    // Don't return InstanceNotFound for stopped instances!
    return &cloudprovider.InstanceMetadata{...}, nil
}

// RetryError - For provisioning delays
func (m *MyCloudProvider) EnsureLoadBalancer(ctx context.Context,
    clusterName string, service *v1.Service, nodes []*v1.Node) (*v1.LoadBalancerStatus, error) {

    lb, err := m.client.CreateLoadBalancer(ctx, spec)
    if err != nil {
        return nil, err
    }

    // LB is still provisioning
    if lb.State == "provisioning" {
        return nil, cloudproviderapi.NewRetryError(
            "load balancer is provisioning",
            30*time.Second,  // Fixed retry interval
        )
    }

    return &v1.LoadBalancerStatus{...}, nil
}

// NotImplemented - For unsupported methods
func (m *MyCloudProvider) AddSSHKeyToAllInstances(ctx context.Context,
    user string, keyData []byte) error {
    return cloudprovider.NotImplemented
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🛠️ Helper Utilities**

### **Node Address Helpers**

**File**: `staging/src/k8s.io/cloud-provider/node/helpers/address.go`

```go
// AddToNodeAddresses appends addresses without duplicates
func AddToNodeAddresses(addresses *[]v1.NodeAddress, addAddresses ...v1.NodeAddress) {
    for _, add := range addAddresses {
        exists := false
        for _, existing := range *addresses {
            if existing.Address == add.Address && existing.Type == add.Type {
                exists = true
                break
            }
        }
        if !exists {
            *addresses = append(*addresses, add)
        }
    }
}

// GetNodeAddressesFromNodeIP filters addresses based on node IP annotation
func GetNodeAddressesFromNodeIP(providedNodeIP string,
    cloudNodeAddresses []v1.NodeAddress) ([]v1.NodeAddress, error) {

    var nodeAddresses []v1.NodeAddress
    nodeIP := netutils.ParseIPSloppy(providedNodeIP)
    if nodeIP == nil {
        return nil, fmt.Errorf("failed to parse node IP: %s", providedNodeIP)
    }

    // Find matching address first
    for _, addr := range cloudNodeAddresses {
        if netutils.ParseIPSloppy(addr.Address).Equal(nodeIP) {
            nodeAddresses = append(nodeAddresses, addr)
        }
    }

    // Add remaining addresses
    for _, addr := range cloudNodeAddresses {
        if !netutils.ParseIPSloppy(addr.Address).Equal(nodeIP) {
            nodeAddresses = append(nodeAddresses, addr)
        }
    }

    return nodeAddresses, nil
}
```

### **Service Helpers**

**File**: `staging/src/k8s.io/cloud-provider/service/helpers/helper.go`

```go
// GetLoadBalancerSourceRanges parses LoadBalancerSourceRanges from service
func GetLoadBalancerSourceRanges(service *v1.Service) (utilnet.IPNetSet, error) {
    // Check annotation first (deprecated)
    if annotation := service.Annotations[v1.AnnotationLoadBalancerSourceRangesKey]; annotation != "" {
        return parseSourceRanges(annotation)
    }

    // Use spec field
    if len(service.Spec.LoadBalancerSourceRanges) > 0 {
        return parseSourceRanges(strings.Join(service.Spec.LoadBalancerSourceRanges, ","))
    }

    // Default: allow all
    return utilnet.IPNetSet{}, nil
}

// RequestsOnlyLocalTraffic checks if service requests local traffic only
func RequestsOnlyLocalTraffic(service *v1.Service) bool {
    if service.Spec.Type != v1.ServiceTypeLoadBalancer &&
        service.Spec.Type != v1.ServiceTypeNodePort {
        return false
    }
    return service.Spec.ExternalTrafficPolicy == v1.ServiceExternalTrafficPolicyTypeLocal
}

// NeedsHealthCheck checks if service needs health check
func NeedsHealthCheck(service *v1.Service) bool {
    if service.Spec.Type != v1.ServiceTypeLoadBalancer {
        return false
    }
    return RequestsOnlyLocalTraffic(service)
}

// GetServiceHealthCheckPathPort returns health check configuration
func GetServiceHealthCheckPathPort(service *v1.Service) (string, int32) {
    if !NeedsHealthCheck(service) {
        return "", 0
    }
    return "/healthz", service.Spec.HealthCheckNodePort
}

// LoadBalancerStatusEqual compares two load balancer statuses
func LoadBalancerStatusEqual(l, r *v1.LoadBalancerStatus) bool {
    return apiequality.Semantic.DeepEqual(l, r)
}
```

### **Zone Helpers**

**File**: `staging/src/k8s.io/cloud-provider/volume/helpers/zones.go`

```go
// LabelZonesToSet converts zone label to set
func LabelZonesToSet(labelZonesValue string) (sets.String, error) {
    // Parse "__" delimited zone list
    return ZonesToSet(strings.ReplaceAll(labelZonesValue, "__", ","))
}

// ZonesSetToLabelValue converts set to zone label
func ZonesSetToLabelValue(strSet sets.String) string {
    return strings.Join(strSet.List(), "__")
}

// SelectZoneForVolume selects a single zone for volume
func SelectZoneForVolume(zoneParameterPresent, zonesParameterPresent bool,
    zoneParameter string, zonesParameter, zonesWithNodes sets.String,
    node *v1.Node, allowedTopologies []v1.TopologySelectorTerm,
    pvcName string) (string, error) {

    // Priority:
    // 1. Zone from storage class parameter
    // 2. Zone from node (if PVC bound to node)
    // 3. Zone from allowed topologies
    // 4. Random zone from cluster

    // ... implementation
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔌 Provider Registration**

### **Registration Pattern**

**File**: `staging/src/k8s.io/cloud-provider/plugins.go`

```go
var (
    providersMutex sync.Mutex
    providers      = make(map[string]Factory)
)

// Factory is a function that returns a cloudprovider.Interface
type Factory func(config io.Reader) (Interface, error)

// RegisterCloudProvider registers a cloudprovider.Factory by name
func RegisterCloudProvider(name string, cloud Factory) {
    providersMutex.Lock()
    defer providersMutex.Unlock()
    if _, found := providers[name]; found {
        klog.Fatalf("Cloud provider %q was registered twice", name)
    }
    klog.V(1).Infof("Registered cloud provider %q", name)
    providers[name] = cloud
}

// GetCloudProvider creates an instance of the named cloud provider
func GetCloudProvider(name string, config io.Reader) (Interface, error) {
    providersMutex.Lock()
    defer providersMutex.Unlock()
    f, found := providers[name]
    if !found {
        return nil, fmt.Errorf("unknown cloud provider %q", name)
    }
    return f(config)
}
```

### **Provider Implementation Pattern**

```go
package mycloud

import (
    "io"
    "k8s.io/cloud-provider"
)

const ProviderName = "mycloud"

func init() {
    cloudprovider.RegisterCloudProvider(ProviderName, newMyCloudProvider)
}

func newMyCloudProvider(config io.Reader) (cloudprovider.Interface, error) {
    // Parse configuration
    cfg, err := parseConfig(config)
    if err != nil {
        return nil, err
    }

    // Create cloud client
    client, err := cloud.NewClient(cfg)
    if err != nil {
        return nil, err
    }

    return &MyCloudProvider{
        client:   client,
        region:   cfg.Region,
        vpc:      cfg.VPC,
    }, nil
}

type MyCloudProvider struct {
    client   *cloud.Client
    region   string
    vpc      string
    clusterID string
}

func (m *MyCloudProvider) Initialize(clientBuilder cloudprovider.ControllerClientBuilder,
    stop <-chan struct{}) {
    // Initialize Kubernetes client
    m.kubeClient = clientBuilder.ClientOrDie("mycloud-provider")

    // Start any provider-specific controllers
    go m.runCustomController(stop)
}

func (m *MyCloudProvider) ProviderName() string {
    return ProviderName
}

func (m *MyCloudProvider) HasClusterID() bool {
    return m.clusterID != ""
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🧪 Testing**

### **Fake Implementation**

**File**: `staging/src/k8s.io/cloud-provider/fake/fake.go`

```go
// Cloud is a test-double implementation of cloudprovider.Interface
type Cloud struct {
    // Interface support flags
    DisableInstances     bool
    DisableRoutes        bool
    DisableLoadBalancers bool
    DisableZones         bool
    DisableClusters      bool

    // InstancesV2 support
    EnableInstancesV2 bool

    // Return values
    Exists           bool
    Err              error
    NodeShutdown     bool
    MetadataErr      error
    Addresses        []v1.NodeAddress
    InstanceTypes    map[types.NodeName]string
    ProviderID       map[types.NodeName]string
    Zone             cloudprovider.Zone

    // LoadBalancer tracking
    Balancers      map[string]Balancer
    UpdateCalls    []UpdateBalancerCall
    EnsureCalls    []UpdateBalancerCall
    EnsureCallCb   func(UpdateBalancerCall)

    // Route tracking
    RouteMap map[string]*Route

    // Call tracking
    Calls []string
    Lock  sync.Mutex

    // Custom metadata override
    OverrideInstanceMetadata func(ctx context.Context, node *v1.Node) (*cloudprovider.InstanceMetadata, error)
}
```

### **Testing Patterns**

```go
func TestNodeController(t *testing.T) {
    // Create fake cloud provider
    cloud := &fake.Cloud{
        EnableInstancesV2: true,
        Addresses: []v1.NodeAddress{
            {Type: v1.NodeInternalIP, Address: "10.0.0.1"},
            {Type: v1.NodeExternalIP, Address: "1.2.3.4"},
        },
        ProviderID: map[types.NodeName]string{
            "node1": "mycloud:///i-123",
        },
        Zone: cloudprovider.Zone{
            FailureDomain: "zone-a",
            Region:        "region-1",
        },
    }

    // Create controller with fake
    controller := NewCloudNodeController(nodeInformer, kubeClient, cloud)

    // Run test
    node := &v1.Node{
        ObjectMeta: metav1.ObjectMeta{Name: "node1"},
        Spec:       v1.NodeSpec{},
    }

    err := controller.syncNode(context.Background(), node.Name)
    assert.NoError(t, err)

    // Verify results
    updatedNode, _ := kubeClient.CoreV1().Nodes().Get(context.TODO(), "node1", metav1.GetOptions{})
    assert.Equal(t, "mycloud:///i-123", updatedNode.Spec.ProviderID)
    assert.Equal(t, "zone-a", updatedNode.Labels[v1.LabelTopologyZone])
}

func TestLoadBalancer(t *testing.T) {
    cloud := &fake.Cloud{
        Balancers: make(map[string]fake.Balancer),
        EnsureCallCb: func(call fake.UpdateBalancerCall) {
            // Custom verification
            assert.Equal(t, "my-service", call.Service.Name)
        },
    }

    // ... test service controller
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💡 Best Practices**

### **Implementation Guidelines**

1. **Implement InstancesV2, Not Instances**
```go
// ✅ GOOD: InstancesV2 reduces API calls
func (m *MyCloudProvider) InstancesV2() (cloudprovider.InstancesV2, bool) {
    return m, true
}

// ❌ AVOID: Legacy Instances interface
func (m *MyCloudProvider) Instances() (cloudprovider.Instances, bool) {
    return m, true
}
```

2. **Use ProviderID for Node Lookups**
```go
// ✅ GOOD: Prefer ProviderID
func (m *MyCloudProvider) InstanceExists(ctx context.Context, node *v1.Node) (bool, error) {
    if node.Spec.ProviderID != "" {
        return m.existsByProviderID(ctx, node.Spec.ProviderID)
    }
    return m.existsByName(ctx, node.Name)
}

// ❌ BAD: Always use node name
func (m *MyCloudProvider) InstanceExists(ctx context.Context, node *v1.Node) (bool, error) {
    return m.existsByName(ctx, node.Name)
}
```

3. **Return InstanceNotFound Correctly**
```go
// ✅ GOOD: Only for truly non-existent instances
if err != nil && isNotFoundError(err) {
    return nil, cloudprovider.InstanceNotFound
}

// ❌ BAD: Return for stopped instances
if instance.State == "stopped" {
    return nil, cloudprovider.InstanceNotFound  // Will delete node!
}
```

4. **Handle EnsureLoadBalancerDeleted Properly**
```go
// ✅ GOOD: Attempt deletion, return nil for not-found
func (m *MyCloudProvider) EnsureLoadBalancerDeleted(...) error {
    err := m.client.DeleteLoadBalancer(ctx, name)
    if err != nil && isNotFoundError(err) {
        return nil  // Already deleted
    }
    return err
}

// ❌ BAD: Return ImplementedElsewhere
func (m *MyCloudProvider) EnsureLoadBalancerDeleted(...) error {
    return cloudprovider.ImplementedElsewhere  // Will prevent cleanup!
}
```

5. **Respect Context Cancellation**
```go
// ✅ GOOD: Pass context to API calls
func (m *MyCloudProvider) InstanceMetadata(ctx context.Context, ...) (..., error) {
    return m.client.GetInstance(ctx, instanceID)  // Respects cancellation
}

// ❌ BAD: Ignore context
func (m *MyCloudProvider) InstanceMetadata(ctx context.Context, ...) (..., error) {
    return m.client.GetInstance(context.Background(), instanceID)
}
```

6. **Thread-Safe Implementation**
```go
// ✅ GOOD: Use mutexes for shared state
type MyCloudProvider struct {
    cache map[string]*Instance
    mu    sync.RWMutex
}

func (m *MyCloudProvider) getFromCache(id string) *Instance {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return m.cache[id]
}
```

### **Configuration Best Practices**

1. **Use Standard Configuration Format**
```ini
# /etc/kubernetes/cloud-config.conf
[Global]
region = us-east-1
vpc-id = vpc-12345
cluster-name = production

[LoadBalancer]
subnet-id = subnet-abcde
security-group = sg-12345
```

2. **Support Environment Variables**
```go
func parseConfig(config io.Reader) (*Config, error) {
    cfg := &Config{}
    if config != nil {
        // Parse from file
    }
    // Override with environment
    if region := os.Getenv("CLOUD_REGION"); region != "" {
        cfg.Region = region
    }
    return cfg, nil
}
```

3. **Validate Configuration**
```go
func (cfg *Config) Validate() error {
    if cfg.Region == "" {
        return fmt.Errorf("region is required")
    }
    if cfg.VPC == "" {
        return fmt.Errorf("vpc-id is required")
    }
    return nil
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 References**

### **Source Code**
- **Main Interface**: `staging/src/k8s.io/cloud-provider/cloud.go`
- **Plugin Registration**: `staging/src/k8s.io/cloud-provider/plugins.go`
- **Fake Implementation**: `staging/src/k8s.io/cloud-provider/fake/fake.go`
- **Sample Implementation**: `staging/src/k8s.io/cloud-provider/sample/basic_main.go`
- **Node Helpers**: `staging/src/k8s.io/cloud-provider/node/helpers/address.go`
- **Service Helpers**: `staging/src/k8s.io/cloud-provider/service/helpers/helper.go`
- **Zone Helpers**: `staging/src/k8s.io/cloud-provider/volume/helpers/zones.go`
- **RetryError**: `staging/src/k8s.io/cloud-provider/api/retry_error.go`

### **Related Documentation**
- **Cloud Controller Manager**: `docs/architecture/claude/cloud-integration/01-cloud-controller-manager.md`
- **LoadBalancer Integration**: `docs/architecture/claude/cloud-integration/03-loadbalancer-integration.md`

### **External Resources**
- **Cloud Provider AWS**: https://github.com/kubernetes/cloud-provider-aws
- **Cloud Provider GCE**: https://github.com/kubernetes/cloud-provider-gcp
- **Cloud Provider Azure**: https://github.com/kubernetes-sigs/cloud-provider-azure
- **Cloud Provider OpenStack**: https://github.com/kubernetes/cloud-provider-openstack

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: Complete
**Last Updated**: 2024-11-17
**Target Audience**: Cloud provider developers, platform engineers
**Scope**: Interface definitions, implementation patterns, testing, best practices
