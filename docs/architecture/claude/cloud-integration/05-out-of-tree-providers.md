# **Out-of-Tree Cloud Provider Development**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Guide to developing and deploying external (out-of-tree) cloud providers

**Target Audience**:
- Cloud provider developers building Kubernetes integrations
- Platform engineers migrating from in-tree providers
- DevOps teams deploying external cloud providers
- Contributors to cloud-provider ecosystem

**Scope**: Out-of-tree architecture, development patterns, migration, and deployment

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Why Out-of-Tree Providers**

### **The In-Tree Problem**

Historically, cloud provider code lived **inside kubernetes/kubernetes**:

```
kubernetes/kubernetes/
├── pkg/
│   └── cloudprovider/
│       ├── providers/
│       │   ├── aws/
│       │   ├── azure/
│       │   ├── gce/
│       │   ├── openstack/
│       │   └── vsphere/
│       └── cloud.go
```

**Problems**:

1. **Release Coupling**: Cloud fixes waited for Kubernetes releases
2. **Testing Burden**: All providers tested in core CI
3. **Binary Size**: ~200MB binaries with all providers
4. **Security**: Cloud credentials in core components
5. **Development Speed**: Long PR review cycles

### **The Out-of-Tree Solution**

External providers live in **separate repositories**:

```
github.com/
├── kubernetes/cloud-provider-aws/
├── kubernetes/cloud-provider-gcp/
├── kubernetes-sigs/cloud-provider-azure/
├── kubernetes/cloud-provider-openstack/
└── kubernetes/cloud-provider-vsphere/
```

**Benefits**:

| **Aspect** | **In-Tree** | **Out-of-Tree** |
|------------|-------------|-----------------|
| **Release Cycle** | Kubernetes release (4 months) | Independent (weekly+) |
| **Development** | kubernetes/kubernetes PR | Separate repo PR |
| **Binary Size** | 200MB+ | 50MB per provider |
| **Testing** | Full Kubernetes CI | Provider-specific CI |
| **Security** | Shared credentials | Isolated credentials |
| **Customization** | Fork Kubernetes | Implement interface |

### **Migration Timeline**

| **Version** | **Status** |
|-------------|------------|
| **v1.20** | In-tree deprecated |
| **v1.21-v1.25** | Migration period |
| **v1.26+** | In-tree removed for new providers |
| **v1.29+** | All in-tree code removed |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Architecture**

### **Component Separation**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    In-Tree Architecture                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  kube-controller-manager              kubelet                       │
│  ┌─────────────────────┐             ┌─────────────────────┐       │
│  │ Node Controller     │             │ Cloud Provider      │       │
│  │ Route Controller    │             │ (node registration) │       │
│  │ Service Controller  │             │                     │       │
│  │ + AWS/GCE/Azure     │             │ + AWS/GCE/Azure     │       │
│  └─────────────────────┘             └─────────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                   Out-of-Tree Architecture                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  kube-controller-manager     cloud-controller-manager    kubelet    │
│  ┌──────────────────┐       ┌──────────────────┐       ┌────────┐  │
│  │ Node Controller  │       │ Node Controller  │       │ No     │  │
│  │ (no cloud)       │       │ Route Controller │       │ cloud  │  │
│  │                  │       │ Service Ctrl     │       │ code   │  │
│  │ --cloud-provider │       │ + Cloud Provider │       │        │  │
│  │   =external      │       │                  │       │ --cloud│  │
│  └──────────────────┘       └──────────────────┘       │ -prov= │  │
│                                                         │ extern │  │
│                                                         └────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### **External Provider Components**

1. **cloud-controller-manager**: Separate binary/deployment
2. **CSI Driver**: Storage operations
3. **Node driver registrar**: Node-specific cloud initialization
4. **Custom controllers**: Provider-specific functionality

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 Building an Out-of-Tree Provider**

### **Project Structure**

```
cloud-provider-mycloud/
├── cmd/
│   └── mycloud-cloud-controller-manager/
│       └── main.go
├── pkg/
│   └── providers/
│       └── mycloud/
│           ├── mycloud.go         # Provider implementation
│           ├── instances.go       # InstancesV2 interface
│           ├── loadbalancer.go    # LoadBalancer interface
│           ├── routes.go          # Routes interface
│           └── config.go          # Configuration
├── deploy/
│   └── kubernetes/
│       ├── cloud-controller-manager.yaml
│       ├── rbac.yaml
│       └── kustomization.yaml
├── Dockerfile
├── Makefile
├── go.mod
└── README.md
```

### **Main Entry Point**

**File**: `cmd/mycloud-cloud-controller-manager/main.go`

```go
package main

import (
    "os"

    "k8s.io/apimachinery/pkg/util/wait"
    cloudprovider "k8s.io/cloud-provider"
    "k8s.io/cloud-provider/app"
    "k8s.io/cloud-provider/app/config"
    "k8s.io/cloud-provider/names"
    "k8s.io/cloud-provider/options"
    "k8s.io/component-base/cli"
    cliflag "k8s.io/component-base/cli/flag"
    _ "k8s.io/component-base/metrics/prometheus/clientgo"

    // Import your provider
    _ "github.com/mycompany/cloud-provider-mycloud/pkg/providers/mycloud"
)

func main() {
    // Create CCM options
    ccmOptions, err := options.NewCloudControllerManagerOptions()
    if err != nil {
        os.Exit(1)
    }

    // Cloud initializer
    cloudInitializer := func(config *config.CompletedConfig) cloudprovider.Interface {
        cloudConfig := config.ComponentConfig.KubeCloudShared.CloudProvider

        cloud, err := cloudprovider.InitCloudProvider(
            cloudConfig.Name,
            cloudConfig.CloudConfigFile,
        )
        if err != nil {
            panic(err)
        }
        if cloud == nil {
            panic("cloud provider is nil")
        }

        return cloud
    }

    // Default controllers
    controllerInitializers := app.DefaultInitFuncConstructors
    controllerAliases := names.CCMControllerAliases()

    // Create command
    fss := cliflag.NamedFlagSets{}
    command := app.NewCloudControllerManagerCommand(
        ccmOptions,
        cloudInitializer,
        controllerInitializers,
        controllerAliases,
        fss,
        wait.NeverStop,
    )

    // Run
    code := cli.Run(command)
    os.Exit(code)
}
```

### **Provider Registration**

**File**: `pkg/providers/mycloud/mycloud.go`

```go
package mycloud

import (
    "io"

    cloudprovider "k8s.io/cloud-provider"
)

const (
    // ProviderName is the name of this cloud provider
    ProviderName = "mycloud"
)

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
    client, err := NewCloudClient(cfg)
    if err != nil {
        return nil, err
    }

    return &MyCloudProvider{
        client:    client,
        config:    cfg,
        clusterID: cfg.ClusterID,
    }, nil
}

// MyCloudProvider implements cloudprovider.Interface
type MyCloudProvider struct {
    client    *CloudClient
    config    *Config
    clusterID string

    // Kubernetes client (set in Initialize)
    kubeClient kubernetes.Interface
}

// Initialize is called after cloud provider construction
func (m *MyCloudProvider) Initialize(
    clientBuilder cloudprovider.ControllerClientBuilder,
    stop <-chan struct{},
) {
    m.kubeClient = clientBuilder.ClientOrDie("mycloud-cloud-provider")

    // Start any provider-specific goroutines
    go m.watchCloudEvents(stop)
}

// ProviderName returns the cloud provider ID
func (m *MyCloudProvider) ProviderName() string {
    return ProviderName
}

// HasClusterID returns true if a ClusterID is required and set
func (m *MyCloudProvider) HasClusterID() bool {
    return m.clusterID != ""
}

// LoadBalancer returns a LoadBalancer interface
func (m *MyCloudProvider) LoadBalancer() (cloudprovider.LoadBalancer, bool) {
    return m, true
}

// InstancesV2 returns an InstancesV2 interface
func (m *MyCloudProvider) InstancesV2() (cloudprovider.InstancesV2, bool) {
    return m, true
}

// Instances returns an Instances interface (deprecated)
func (m *MyCloudProvider) Instances() (cloudprovider.Instances, bool) {
    return nil, false
}

// Routes returns a Routes interface
func (m *MyCloudProvider) Routes() (cloudprovider.Routes, bool) {
    return m, true
}

// Zones returns a Zones interface (deprecated)
func (m *MyCloudProvider) Zones() (cloudprovider.Zones, bool) {
    return nil, false
}

// Clusters returns a Clusters interface
func (m *MyCloudProvider) Clusters() (cloudprovider.Clusters, bool) {
    return nil, false
}
```

### **Configuration**

**File**: `pkg/providers/mycloud/config.go`

```go
package mycloud

import (
    "io"
    "os"

    "gopkg.in/gcfg.v1"
)

// Config holds the configuration for MyCloud provider
type Config struct {
    Global GlobalConfig
}

// GlobalConfig is the global configuration section
type GlobalConfig struct {
    // ClusterID is the unique identifier for this cluster
    ClusterID string `gcfg:"cluster-id"`

    // Region is the cloud region
    Region string `gcfg:"region"`

    // VPC is the VPC ID
    VPC string `gcfg:"vpc-id"`

    // APIEndpoint for the cloud API
    APIEndpoint string `gcfg:"api-endpoint"`

    // Authentication
    APIKey    string `gcfg:"api-key"`
    APISecret string `gcfg:"api-secret"`
}

func parseConfig(config io.Reader) (*Config, error) {
    cfg := &Config{}

    if config != nil {
        if err := gcfg.ReadInto(cfg, config); err != nil {
            return nil, err
        }
    }

    // Override with environment variables
    if clusterID := os.Getenv("MYCLOUD_CLUSTER_ID"); clusterID != "" {
        cfg.Global.ClusterID = clusterID
    }
    if region := os.Getenv("MYCLOUD_REGION"); region != "" {
        cfg.Global.Region = region
    }
    if apiKey := os.Getenv("MYCLOUD_API_KEY"); apiKey != "" {
        cfg.Global.APIKey = apiKey
    }
    if apiSecret := os.Getenv("MYCLOUD_API_SECRET"); apiSecret != "" {
        cfg.Global.APISecret = apiSecret
    }

    return cfg, nil
}
```

### **InstancesV2 Implementation**

**File**: `pkg/providers/mycloud/instances.go`

```go
package mycloud

import (
    "context"
    "fmt"

    v1 "k8s.io/api/core/v1"
    cloudprovider "k8s.io/cloud-provider"
)

// InstanceExists returns true if the instance exists
func (m *MyCloudProvider) InstanceExists(ctx context.Context, node *v1.Node) (bool, error) {
    instanceID := m.getInstanceID(node)

    instance, err := m.client.GetInstance(ctx, instanceID)
    if err != nil {
        if isNotFoundError(err) {
            return false, nil
        }
        return false, err
    }

    return instance != nil, nil
}

// InstanceShutdown returns true if the instance is shutdown
func (m *MyCloudProvider) InstanceShutdown(ctx context.Context, node *v1.Node) (bool, error) {
    instanceID := m.getInstanceID(node)

    instance, err := m.client.GetInstance(ctx, instanceID)
    if err != nil {
        return false, err
    }

    return instance.State == "stopped" || instance.State == "stopping", nil
}

// InstanceMetadata returns the instance's metadata
func (m *MyCloudProvider) InstanceMetadata(ctx context.Context, node *v1.Node) (*cloudprovider.InstanceMetadata, error) {
    instanceID := m.getInstanceID(node)

    instance, err := m.client.GetInstance(ctx, instanceID)
    if err != nil {
        if isNotFoundError(err) {
            return nil, cloudprovider.InstanceNotFound
        }
        return nil, err
    }

    // Build node addresses
    addresses := []v1.NodeAddress{
        {Type: v1.NodeInternalIP, Address: instance.PrivateIP},
        {Type: v1.NodeHostName, Address: instance.Hostname},
    }
    if instance.PublicIP != "" {
        addresses = append(addresses, v1.NodeAddress{
            Type:    v1.NodeExternalIP,
            Address: instance.PublicIP,
        })
    }

    return &cloudprovider.InstanceMetadata{
        ProviderID:   fmt.Sprintf("mycloud:///%s/%s", instance.Zone, instance.ID),
        InstanceType: instance.Type,
        NodeAddresses: addresses,
        Zone:         instance.Zone,
        Region:       instance.Region,
        AdditionalLabels: map[string]string{
            "mycloud.com/instance-family": instance.Family,
        },
    }, nil
}

// getInstanceID extracts instance ID from node
func (m *MyCloudProvider) getInstanceID(node *v1.Node) string {
    // Prefer provider ID
    if node.Spec.ProviderID != "" {
        return parseProviderID(node.Spec.ProviderID)
    }
    // Fallback to node name
    return node.Name
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚀 Deployment**

### **Kubernetes Manifest**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mycloud-cloud-controller-manager
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: cloud-controller-manager
  template:
    metadata:
      labels:
        component: cloud-controller-manager
    spec:
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
      - key: node.cloudprovider.kubernetes.io/uninitialized
        value: "true"
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      serviceAccountName: cloud-controller-manager
      containers:
      - name: cloud-controller-manager
        image: mycompany/mycloud-cloud-controller-manager:v1.0.0
        command:
        - /mycloud-cloud-controller-manager
        args:
        - --cloud-provider=mycloud
        - --cloud-config=/etc/cloud/cloud-config
        - --cluster-name=$(CLUSTER_NAME)
        - --controllers=*
        - --leader-elect=true
        - --use-service-account-credentials=true
        - --v=2
        env:
        - name: CLUSTER_NAME
          valueFrom:
            configMapKeyRef:
              name: cluster-info
              key: cluster-name
        - name: MYCLOUD_API_KEY
          valueFrom:
            secretKeyRef:
              name: mycloud-credentials
              key: api-key
        - name: MYCLOUD_API_SECRET
          valueFrom:
            secretKeyRef:
              name: mycloud-credentials
              key: api-secret
        volumeMounts:
        - name: cloud-config
          mountPath: /etc/cloud
          readOnly: true
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10258
            scheme: HTTPS
          initialDelaySeconds: 15
        ports:
        - containerPort: 10258
      volumes:
      - name: cloud-config
        configMap:
          name: cloud-config
      priorityClassName: system-node-critical
```

### **RBAC Configuration**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloud-controller-manager
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloud-controller-manager
rules:
# Nodes
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch", "delete", "patch", "update"]
- apiGroups: [""]
  resources: ["nodes/status"]
  verbs: ["patch", "update"]

# Services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: [""]
  resources: ["services/status"]
  verbs: ["patch", "update"]

# Events
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch", "update"]

# Service accounts
- apiGroups: [""]
  resources: ["serviceaccounts"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["serviceaccounts/token"]
  verbs: ["create"]

# Endpoints
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get", "list", "watch", "create", "update"]

# Secrets and ConfigMaps
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["get", "list", "watch"]

# Leader election
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "list", "watch", "create", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cloud-controller-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cloud-controller-manager
subjects:
- kind: ServiceAccount
  name: cloud-controller-manager
  namespace: kube-system
```

### **Configuring Kubernetes Components**

**kube-apiserver**:
```bash
# No changes needed
```

**kube-controller-manager**:
```bash
# Disable in-tree cloud provider
--cloud-provider=external

# Route controller runs in CCM
--configure-cloud-routes=false

# Node lifecycle handled by CCM
--cloud-node-lifecycle-controller=false
```

**kubelet**:
```bash
# Disable in-tree cloud provider
--cloud-provider=external

# Register with taint
--register-with-taints="node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Migration from In-Tree**

### **Migration Strategies**

**1. Blue-Green Migration**:
- Deploy new cluster with out-of-tree provider
- Migrate workloads
- Decommission old cluster

**2. In-Place Migration**:
- Deploy CCM alongside kube-controller-manager
- Migrate controllers one by one
- Update kubelet configuration

### **In-Place Migration Steps**

**Step 1: Deploy CCM (disabled)**

```yaml
# CCM with controllers disabled
args:
- --cloud-provider=mycloud
- --controllers=-*  # All disabled
- --leader-elect=true
```

**Step 2: Migrate Node Controller**

```yaml
# kube-controller-manager
args:
- --cloud-provider=external
- --controllers=*,-cloud-node,-cloud-node-lifecycle

# CCM
args:
- --controllers=cloud-node-controller,cloud-node-lifecycle-controller
```

**Step 3: Migrate Service Controller**

```yaml
# kube-controller-manager
args:
- --controllers=*,-cloud-node,-cloud-node-lifecycle,-service

# CCM
args:
- --controllers=cloud-node-controller,cloud-node-lifecycle-controller,service-lb-controller
```

**Step 4: Migrate Route Controller**

```yaml
# kube-controller-manager
args:
- --controllers=*,-cloud-node,-cloud-node-lifecycle,-service,-route

# CCM
args:
- --controllers=*  # All controllers
```

**Step 5: Update Kubelet**

```bash
# Rolling update of nodes
kubelet --cloud-provider=external \
  --register-with-taints="node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule"
```

### **Migration Verification**

```bash
# Check CCM is processing nodes
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.providerID}{"\n"}{end}'

# Check services are getting IPs
kubectl get svc --all-namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.loadBalancer.ingress[*].ip}{"\n"}{end}'

# Check routes are created
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# Check CCM logs
kubectl logs -n kube-system -l component=cloud-controller-manager
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🧪 Testing**

### **Unit Testing**

```go
package mycloud

import (
    "context"
    "testing"

    v1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes/fake"
)

func TestInstanceMetadata(t *testing.T) {
    // Create mock cloud client
    client := &MockCloudClient{
        instances: map[string]*Instance{
            "i-123": {
                ID:        "i-123",
                PrivateIP: "10.0.0.1",
                PublicIP:  "1.2.3.4",
                Zone:      "zone-a",
                Region:    "region-1",
                Type:      "medium",
            },
        },
    }

    provider := &MyCloudProvider{
        client: client,
    }

    node := &v1.Node{
        ObjectMeta: metav1.ObjectMeta{
            Name: "node-1",
        },
        Spec: v1.NodeSpec{
            ProviderID: "mycloud:///zone-a/i-123",
        },
    }

    meta, err := provider.InstanceMetadata(context.Background(), node)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if meta.Zone != "zone-a" {
        t.Errorf("expected zone zone-a, got %s", meta.Zone)
    }

    if len(meta.NodeAddresses) != 3 {
        t.Errorf("expected 3 addresses, got %d", len(meta.NodeAddresses))
    }
}
```

### **Integration Testing**

```go
func TestLoadBalancerIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    // Create real cloud client
    client, err := NewCloudClient(&Config{
        Global: GlobalConfig{
            Region: "test-region",
        },
    })
    if err != nil {
        t.Fatal(err)
    }

    provider := &MyCloudProvider{
        client:    client,
        clusterID: "test-cluster",
    }

    // Create test service
    service := &v1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-lb",
            Namespace: "default",
        },
        Spec: v1.ServiceSpec{
            Type: v1.ServiceTypeLoadBalancer,
            Ports: []v1.ServicePort{
                {Port: 80, NodePort: 30000},
            },
        },
    }

    // Test EnsureLoadBalancer
    status, err := provider.EnsureLoadBalancer(
        context.Background(),
        "test-cluster",
        service,
        []*v1.Node{},
    )
    if err != nil {
        t.Fatalf("EnsureLoadBalancer failed: %v", err)
    }

    if len(status.Ingress) == 0 {
        t.Error("expected load balancer IP")
    }

    // Cleanup
    defer provider.EnsureLoadBalancerDeleted(
        context.Background(),
        "test-cluster",
        service,
    )
}
```

### **E2E Testing**

```bash
# Run Kubernetes E2E tests for cloud provider
go test -v ./test/e2e/... \
  -ginkgo.focus="Cloud Provider" \
  -cloud-provider=mycloud \
  -cloud-config=/path/to/config

# Specific tests
go test -v ./test/e2e/... \
  -ginkgo.focus="LoadBalancer" \
  -cloud-provider=mycloud
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💡 Best Practices**

### **Development**

1. **Use InstancesV2**: More efficient than legacy Instances interface
2. **Implement All Required Methods**: Even if returning NotImplemented
3. **Use Context**: Pass context through all API calls
4. **Handle Errors Properly**: Distinguish transient vs permanent errors
5. **Add Metrics**: Expose Prometheus metrics for observability

### **Deployment**

1. **Run Multiple Replicas**: Enable leader election
2. **Set Resource Limits**: Prevent resource exhaustion
3. **Use Service Accounts**: Don't use cluster admin credentials
4. **Secure Credentials**: Use Kubernetes secrets or cloud IAM

### **Operations**

1. **Monitor CCM Health**: Alert on leader election failures
2. **Track Cloud API Errors**: Monitor rate limiting
3. **Test Upgrades**: Validate in staging before production
4. **Document Configuration**: Keep cloud config documented

### **Migration**

1. **Test in Non-Production**: Validate migration steps
2. **Migrate Incrementally**: One controller at a time
3. **Keep Rollback Plan**: Be ready to revert
4. **Monitor During Migration**: Watch for errors

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 References**

### **Source Code**
- **Cloud Provider Framework**: `staging/src/k8s.io/cloud-provider/`
- **Sample Provider**: `staging/src/k8s.io/cloud-provider/sample/`
- **Controller Manager App**: `staging/src/k8s.io/cloud-provider/app/`

### **Related Documentation**
- **Cloud Controller Manager**: `docs/architecture/claude/cloud-integration/01-cloud-controller-manager.md`
- **Cloud Provider Interface**: `docs/architecture/claude/cloud-integration/02-cloud-provider-interface.md`

### **External Resources**
- **Cloud Provider AWS**: https://github.com/kubernetes/cloud-provider-aws
- **Cloud Provider GCE**: https://github.com/kubernetes/cloud-provider-gcp
- **Cloud Provider Azure**: https://github.com/kubernetes-sigs/cloud-provider-azure
- **Cloud Provider Development Guide**: https://kubernetes.io/docs/tasks/administer-cluster/developing-cloud-controller-manager/

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: Complete
**Last Updated**: 2024-11-19
**Target Audience**: Cloud provider developers, platform engineers
**Scope**: Out-of-tree development, deployment, migration, testing
