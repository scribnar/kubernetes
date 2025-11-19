# **Cloud Controller Manager Architecture**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Comprehensive guide to Kubernetes Cloud Controller Manager (CCM) architecture and implementation

**Target Audience**:
- Cloud provider engineers building CCM implementations
- Platform engineers deploying Kubernetes on cloud infrastructure
- SREs managing cloud-integrated Kubernetes clusters
- Contributors to cloud-provider ecosystem

**Scope**: CCM architecture, controller implementations, cloud provider interface, configuration, and production patterns

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Why Cloud Controller Manager Exists**

### **The Cloud Integration Challenge**

Kubernetes was originally designed with **cloud-specific code embedded in core components** (kube-controller-manager, kubelet). This created problems:

1. **Tight coupling** - Cloud provider changes required Kubernetes core releases
2. **Release cycles** - Cloud providers couldn't iterate independently
3. **Testing burden** - All cloud providers tested in core CI
4. **Binary bloat** - Single binary contained all provider code
5. **Security concerns** - Cloud credentials in core components

### **The Solution: Externalization**

The Cloud Controller Manager (CCM) provides:

```
┌─────────────────────────────────────────────────────────────┐
│                    Before CCM (v1.5-)                       │
├─────────────────────────────────────────────────────────────┤
│  kube-controller-manager                                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │  AWS    │ │  GCE    │ │ Azure   │ │ vSphere │  ...     │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │
│  ┌─────────────────────────────────────────────┐           │
│  │        Core Kubernetes Controllers          │           │
│  └─────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    After CCM (v1.6+)                        │
├─────────────────────────────────────────────────────────────┤
│  kube-controller-manager          cloud-controller-manager  │
│  ┌───────────────────────┐       ┌─────────────────────┐   │
│  │ Core Controllers      │       │ Cloud Controllers   │   │
│  │ (cloud-agnostic)      │       │ + Cloud Provider    │   │
│  └───────────────────────┘       └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### **Benefits**

| **Aspect** | **Before CCM** | **After CCM** |
|------------|----------------|---------------|
| **Release Cycle** | Coupled to Kubernetes releases | Independent provider releases |
| **Development** | In kubernetes/kubernetes | Out-of-tree repositories |
| **Testing** | Central CI/CD | Provider-specific testing |
| **Binary Size** | ~200MB with all providers | ~50MB per provider |
| **Credentials** | In kube-controller-manager | Isolated to CCM |
| **Customization** | Fork Kubernetes | Implement interface |

### **Controllers Moved to CCM**

| **Controller** | **Purpose** | **Cloud Operations** |
|----------------|-------------|---------------------|
| **Cloud Node Controller** | Sync node metadata | Instance IPs, zones, instance types |
| **Cloud Node Lifecycle** | Handle instance termination | Delete nodes when instances terminate |
| **Service Controller** | Manage LoadBalancers | Create/update cloud load balancers |
| **Route Controller** | Manage pod routes | Configure cloud network routes |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 CCM Architecture**

### **High-Level Architecture**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Cloud Controller Manager                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │  Cloud Node     │  │  Service LB     │  │  Route          │     │
│  │  Controller     │  │  Controller     │  │  Controller     │     │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘     │
│           │                    │                    │               │
│           └────────────────────┼────────────────────┘               │
│                                │                                    │
│                    ┌───────────▼───────────┐                       │
│                    │   Cloud Provider      │                       │
│                    │   Interface           │                       │
│                    └───────────┬───────────┘                       │
│                                │                                    │
├────────────────────────────────┼────────────────────────────────────┤
│                                │                                    │
│                    ┌───────────▼───────────┐                       │
│                    │  Cloud Provider       │                       │
│                    │  Implementation       │                       │
│                    │  (AWS/GCE/Azure/...)  │                       │
│                    └───────────┬───────────┘                       │
│                                │                                    │
└────────────────────────────────┼────────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    Cloud APIs          │
                    │    (EC2, GCE, ARM)     │
                    └─────────────────────────┘
```

### **Component Interactions**

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Nodes   │     │ Services │     │  Routes  │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     │  Watch         │  Watch         │  Watch
     ▼                ▼                ▼
┌──────────────────────────────────────────────┐
│           Shared Informer Factory            │
└─────────────────────┬────────────────────────┘
                      │
     ┌────────────────┼────────────────┐
     │                │                │
     ▼                ▼                ▼
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Cloud   │    │ Service │    │ Route   │
│ Node    │    │ LB      │    │         │
│ Ctrl    │    │ Ctrl    │    │ Ctrl    │
└────┬────┘    └────┬────┘    └────┬────┘
     │              │              │
     └──────────────┼──────────────┘
                    │
                    ▼
          ┌─────────────────┐
          │ Cloud Provider  │
          │ Interface       │
          └────────┬────────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Instance│ │ LB      │ │ Route   │
   │ API     │ │ API     │ │ API     │
   └─────────┘ └─────────┘ └─────────┘
```

### **Main Entry Point**

**File**: `cmd/cloud-controller-manager/main.go`

```go
func main() {
    rand.Seed(time.Now().UnixNano())

    // 1. Create CCM options
    ccmOptions, err := options.NewCloudControllerManagerOptions()
    if err != nil {
        klog.Fatalf("unable to initialize command options: %v", err)
    }

    // 2. Define default controller initializers
    controllerInitializers := app.DefaultInitFuncConstructors

    // 3. Define controller name aliases (for backwards compatibility)
    controllerAliases := names.CCMControllerAliases()

    // 4. Cloud provider initializer function
    fss := cliflag.NamedFlagSets{}
    cloudInitializer := func(config *cloudcontrollerconfig.CompletedConfig) cloudprovider.Interface {
        cloudConfig := config.ComponentConfig.KubeCloudShared.CloudProvider

        // Initialize cloud provider
        cloud, err := cloudprovider.InitCloudProvider(
            cloudConfig.Name,
            cloudConfig.CloudConfigFile,
        )
        if err != nil {
            klog.Fatalf("Cloud provider could not be initialized: %v", err)
        }
        if cloud == nil {
            klog.Fatalf("Cloud provider is nil")
        }

        // Optionally add Node IPAM controller
        if !config.ComponentConfig.KubeCloudShared.AllocateNodeCIDRs {
            return cloud
        }

        // Add node IPAM controller
        controllerInitializers[nodeipamcontroller.ControllerName] = nodeipamconfig.ControllerInitFunc(
            cloud,
            config.ComponentConfig.KubeCloudShared,
            config.ComponentConfig.NodeIPAMController,
        )

        return cloud
    }

    // 5. Create and execute CCM command
    command := app.NewCloudControllerManagerCommand(
        ccmOptions,
        cloudInitializer,
        controllerInitializers,
        controllerAliases,
        fss,
        wait.NeverStop,
    )

    code := cli.Run(command)
    os.Exit(code)
}
```

### **Controller Manager Run Flow**

**File**: `staging/src/k8s.io/cloud-provider/app/controllermanager.go`

```go
func Run(ctx context.Context, c *cloudcontrollerconfig.CompletedConfig,
    cloud cloudprovider.Interface, controllers map[string]ControllerInitFuncConstructor) error {

    // 1. Setup event broadcasting
    eventBroadcaster := record.NewBroadcaster()
    eventBroadcaster.StartStructuredLogging(0)
    eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{
        Interface: c.Client.CoreV1().Events(""),
    })

    // 2. Create controller context
    controllerContext, err := CreateControllerContext(ctx, c, cloud)
    if err != nil {
        return err
    }

    // 3. Setup health checks
    var checks []healthz.HealthChecker
    var electionChecker *leaderelection.HealthzAdaptor

    // 4. Leader election (if enabled)
    if c.ComponentConfig.Generic.LeaderElection.LeaderElect {
        electionChecker = leaderelection.NewLeaderHealthzAdaptor(
            time.Second * 20,
        )
        checks = append(checks, electionChecker)
    }

    // 5. Setup HTTP handlers
    healthzHandler := controllerhealthz.NewMutableHealthzHandler(checks...)
    handler := buildHandlers(c, controllerContext, healthzHandler)

    // 6. Start HTTPS server
    if _, _, err := c.SecureServing.Serve(handler, 0, ctx.Done()); err != nil {
        return err
    }

    // 7. Run controllers with leader election
    run := func(ctx context.Context) {
        // Initialize cloud provider
        cloud.Initialize(controllerContext.ClientBuilder, ctx.Done())

        // Start controllers
        if err := startControllers(ctx, controllerContext, controllers, cloud); err != nil {
            klog.Fatalf("error running controllers: %v", err)
        }
    }

    if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
        run(ctx)
        panic("unreachable")
    }

    // Leader election
    leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
        Lock:          resourcelock,
        LeaseDuration: c.ComponentConfig.Generic.LeaderElection.LeaseDuration.Duration,
        RenewDeadline: c.ComponentConfig.Generic.LeaderElection.RenewDeadline.Duration,
        RetryPeriod:   c.ComponentConfig.Generic.LeaderElection.RetryPeriod.Duration,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                if electionChecker != nil {
                    electionChecker.SetLeaderElection(le)
                }
                run(ctx)
            },
            OnStoppedLeading: func() {
                klog.Fatalf("leaderelection lost")
            },
        },
    })

    panic("unreachable")
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔌 Cloud Provider Interface**

### **Core Interface Definition**

**File**: `staging/src/k8s.io/cloud-provider/cloud.go`

```go
// Interface is an abstract, pluggable interface for cloud providers
type Interface interface {
    // Initialize provides the cloud provider with a kubernetes client builder
    // and may spawn goroutines to perform housekeeping or run custom controllers
    // specific to the cloud provider. Any tasks started here should be cleaned up
    // when the stop channel closes.
    Initialize(clientBuilder ControllerClientBuilder, stop <-chan struct{})

    // LoadBalancer returns a balancer interface if supported
    LoadBalancer() (LoadBalancer, bool)

    // Instances returns an instances interface if supported
    // DEPRECATED: Use InstancesV2
    Instances() (Instances, bool)

    // InstancesV2 returns an instances interface if supported
    InstancesV2() (InstancesV2, bool)

    // Zones returns a zones interface if supported
    // DEPRECATED: Use InstancesV2
    Zones() (Zones, bool)

    // Clusters returns a clusters interface if supported
    Clusters() (Clusters, bool)

    // Routes returns a routes interface if supported
    Routes() (Routes, bool)

    // ProviderName returns the cloud provider ID
    ProviderName() string

    // HasClusterID returns true if a ClusterID is required and set
    HasClusterID() bool
}
```

### **LoadBalancer Interface**

**File**: `staging/src/k8s.io/cloud-provider/cloud.go`

```go
type LoadBalancer interface {
    // GetLoadBalancer returns whether the specified load balancer exists,
    // and if so, what its status is
    GetLoadBalancer(ctx context.Context, clusterName string, service *v1.Service) (
        status *v1.LoadBalancerStatus, exists bool, err error)

    // GetLoadBalancerName returns the name of the load balancer
    GetLoadBalancerName(ctx context.Context, clusterName string, service *v1.Service) string

    // EnsureLoadBalancer creates a new load balancer or updates existing one
    EnsureLoadBalancer(ctx context.Context, clusterName string, service *v1.Service,
        nodes []*v1.Node) (*v1.LoadBalancerStatus, error)

    // UpdateLoadBalancer updates hosts under the specified load balancer
    UpdateLoadBalancer(ctx context.Context, clusterName string, service *v1.Service,
        nodes []*v1.Node) error

    // EnsureLoadBalancerDeleted deletes the specified load balancer if it exists
    EnsureLoadBalancerDeleted(ctx context.Context, clusterName string,
        service *v1.Service) error
}
```

### **InstancesV2 Interface**

**File**: `staging/src/k8s.io/cloud-provider/cloud.go`

```go
type InstancesV2 interface {
    // InstanceExists returns true if the instance for the given node exists
    InstanceExists(ctx context.Context, node *v1.Node) (bool, error)

    // InstanceShutdown returns true if the instance is shutdown
    InstanceShutdown(ctx context.Context, node *v1.Node) (bool, error)

    // InstanceMetadata returns the instance's metadata
    InstanceMetadata(ctx context.Context, node *v1.Node) (*InstanceMetadata, error)
}

type InstanceMetadata struct {
    // ProviderID is the provider's unique ID for the instance
    ProviderID string

    // InstanceType is the instance's type (e.g., "m5.xlarge")
    InstanceType string

    // NodeAddresses is a list of addresses for the instance
    NodeAddresses []v1.NodeAddress

    // Zone is the failure domain zone for the instance
    Zone string

    // Region is the region for the instance
    Region string
}
```

### **Routes Interface**

**File**: `staging/src/k8s.io/cloud-provider/cloud.go`

```go
type Routes interface {
    // ListRoutes lists all managed routes that belong to the specified clusterName
    ListRoutes(ctx context.Context, clusterName string) ([]*Route, error)

    // CreateRoute creates the described managed route
    CreateRoute(ctx context.Context, clusterName string, nameHint string,
        route *Route) error

    // DeleteRoute deletes the specified managed route
    DeleteRoute(ctx context.Context, clusterName string, route *Route) error
}

type Route struct {
    // Name is the name of the route
    Name string

    // TargetNode is the NodeName of the target instance
    TargetNode types.NodeName

    // DestinationCIDR is the CIDR format IP range for this routing table entry
    DestinationCIDR string

    // Blackhole is set to true if this is a blackhole route
    Blackhole bool
}
```

### **Provider Registration**

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

// InitCloudProvider creates a cloud provider instance
func InitCloudProvider(name string, configFilePath string) (Interface, error) {
    if name == "" {
        klog.Info("No cloud provider specified")
        return nil, nil
    }

    // Check if external provider
    if IsExternal(name) {
        klog.Info("External cloud provider %q specified", name)
        return nil, nil
    }

    // Load configuration
    var config io.Reader
    if configFilePath != "" {
        var err error
        config, err = os.Open(configFilePath)
        if err != nil {
            return nil, fmt.Errorf("couldn't open cloud provider configuration %s: %v",
                configFilePath, err)
        }
        defer config.(*os.File).Close()
    }

    return GetCloudProvider(name, config)
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎮 Core Controllers Implementation**

### **Default Controllers**

**File**: `staging/src/k8s.io/cloud-provider/app/controllermanager.go`

```go
var DefaultInitFuncConstructors = map[string]ControllerInitFuncConstructor{
    // Cloud Node Controller - sync node metadata
    names.CloudNodeController: {
        InitContext: ControllerInitContext{
            ClientName: "node-controller",
        },
        Constructor: StartCloudNodeControllerWrapper,
    },

    // Cloud Node Lifecycle Controller - handle instance termination
    names.CloudNodeLifecycleController: {
        InitContext: ControllerInitContext{
            ClientName: "node-controller",
        },
        Constructor: StartCloudNodeLifecycleControllerWrapper,
    },

    // Service LB Controller - manage cloud load balancers
    names.ServiceLBController: {
        InitContext: ControllerInitContext{
            ClientName: "service-controller",
        },
        Constructor: StartServiceControllerWrapper,
    },

    // Route Controller - manage cloud network routes
    names.NodeRouteController: {
        InitContext: ControllerInitContext{
            ClientName: "route-controller",
        },
        Constructor: StartRouteControllerWrapper,
    },
}
```

### **1. Cloud Node Controller**

**File**: `staging/src/k8s.io/cloud-provider/controllers/node/node_controller.go`

**Purpose**: Synchronizes node metadata from cloud provider to Kubernetes Node objects

```go
type CloudNodeController struct {
    kubeClient       clientset.Interface
    nodeInformer     coreinformers.NodeInformer
    cloud            cloudprovider.Interface
    nodeStatusUpdate func(node *v1.Node) error
    workqueue        workqueue.RateLimitingInterface
}

func NewCloudNodeController(
    nodeInformer coreinformers.NodeInformer,
    kubeClient clientset.Interface,
    cloud cloudprovider.Interface,
    nodeStatusUpdateFrequency time.Duration,
) (*CloudNodeController, error) {

    // Get cloud provider's InstancesV2 interface
    instancesV2, ok := cloud.InstancesV2()
    if !ok {
        return nil, errors.New("cloud provider does not support InstancesV2")
    }

    cnc := &CloudNodeController{
        kubeClient:   kubeClient,
        nodeInformer: nodeInformer,
        cloud:        cloud,
        workqueue: workqueue.NewNamedRateLimitingQueue(
            workqueue.DefaultControllerRateLimiter(),
            "cloud-node-controller",
        ),
    }

    // Watch for node events
    nodeInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            node := obj.(*v1.Node)
            cnc.AddCloudNode(context.TODO(), node)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            node := newObj.(*v1.Node)
            cnc.UpdateCloudNode(context.TODO(), oldObj.(*v1.Node), node)
        },
    })

    return cnc, nil
}

func (cnc *CloudNodeController) Run(ctx context.Context, workers int) {
    defer cnc.workqueue.ShutDown()

    klog.Info("Starting cloud node controller")

    // Wait for cache sync
    if !cache.WaitForNamedCacheSync("cloud-node-controller", ctx.Done(),
        cnc.nodeInformer.Informer().HasSynced) {
        return
    }

    // Start workers
    for i := 0; i < workers; i++ {
        go wait.UntilWithContext(ctx, cnc.runWorker, time.Second)
    }

    <-ctx.Done()
}

func (cnc *CloudNodeController) syncNode(ctx context.Context, nodeName string) error {
    node, err := cnc.nodeInformer.Lister().Get(nodeName)
    if err != nil {
        if errors.IsNotFound(err) {
            return nil
        }
        return err
    }

    // Get instance metadata from cloud provider
    instancesV2, _ := cnc.cloud.InstancesV2()
    instanceMeta, err := instancesV2.InstanceMetadata(ctx, node)
    if err != nil {
        return fmt.Errorf("failed to get instance metadata: %v", err)
    }

    // Update node with cloud provider data
    nodeCopy := node.DeepCopy()

    // 1. Set Provider ID
    if nodeCopy.Spec.ProviderID == "" {
        nodeCopy.Spec.ProviderID = instanceMeta.ProviderID
    }

    // 2. Update node addresses
    nodeCopy.Status.Addresses = instanceMeta.NodeAddresses

    // 3. Update labels
    if nodeCopy.Labels == nil {
        nodeCopy.Labels = make(map[string]string)
    }
    nodeCopy.Labels[v1.LabelTopologyZone] = instanceMeta.Zone
    nodeCopy.Labels[v1.LabelTopologyRegion] = instanceMeta.Region
    nodeCopy.Labels[v1.LabelInstanceTypeStable] = instanceMeta.InstanceType

    // 4. Remove cloud-provider initialization taint
    removeTaint(nodeCopy, cloudproviderapi.TaintExternalCloudProvider)

    // Update node if changed
    if !nodeutil.NodeEqual(node, nodeCopy) {
        _, err = cnc.kubeClient.CoreV1().Nodes().Update(ctx, nodeCopy, metav1.UpdateOptions{})
        return err
    }

    return nil
}
```

### **2. Cloud Node Lifecycle Controller**

**File**: `staging/src/k8s.io/cloud-provider/controllers/nodelifecycle/node_lifecycle_controller.go`

**Purpose**: Monitors cloud instance status and handles node deletion when instances are terminated

```go
type CloudNodeLifecycleController struct {
    kubeClient   clientset.Interface
    nodeLister   corelisters.NodeLister
    cloud        cloudprovider.Interface
    recorder     record.EventRecorder

    // Interval for checking node status
    nodeMonitorPeriod time.Duration
}

func NewCloudNodeLifecycleController(
    nodeInformer coreinformers.NodeInformer,
    kubeClient clientset.Interface,
    cloud cloudprovider.Interface,
    nodeMonitorPeriod time.Duration,
) (*CloudNodeLifecycleController, error) {

    eventBroadcaster := record.NewBroadcaster()
    recorder := eventBroadcaster.NewRecorder(scheme.Scheme,
        v1.EventSource{Component: "cloud-node-lifecycle-controller"})

    c := &CloudNodeLifecycleController{
        kubeClient:        kubeClient,
        nodeLister:        nodeInformer.Lister(),
        cloud:             cloud,
        recorder:          recorder,
        nodeMonitorPeriod: nodeMonitorPeriod,
    }

    return c, nil
}

func (c *CloudNodeLifecycleController) Run(ctx context.Context) {
    defer utilruntime.HandleCrash()

    klog.Info("Starting cloud node lifecycle controller")

    // Periodically check all nodes
    wait.UntilWithContext(ctx, func(ctx context.Context) {
        if err := c.MonitorNodes(ctx); err != nil {
            klog.Errorf("error monitoring nodes: %v", err)
        }
    }, c.nodeMonitorPeriod)
}

func (c *CloudNodeLifecycleController) MonitorNodes(ctx context.Context) error {
    nodes, err := c.nodeLister.List(labels.Everything())
    if err != nil {
        return err
    }

    instancesV2, ok := c.cloud.InstancesV2()
    if !ok {
        return errors.New("cloud provider does not support InstancesV2")
    }

    for _, node := range nodes {
        // Skip nodes without provider ID
        if node.Spec.ProviderID == "" {
            continue
        }

        // Check if instance exists
        exists, err := instancesV2.InstanceExists(ctx, node)
        if err != nil {
            klog.Errorf("error checking instance existence for node %s: %v", node.Name, err)
            continue
        }

        if !exists {
            // Instance terminated - delete node
            klog.Infof("Deleting node %s because instance no longer exists", node.Name)
            c.recorder.Eventf(node, v1.EventTypeNormal, "DeletingNode",
                "Deleting Node %s because instance is gone", node.Name)

            if err := c.kubeClient.CoreV1().Nodes().Delete(ctx, node.Name,
                metav1.DeleteOptions{}); err != nil {
                klog.Errorf("error deleting node %s: %v", node.Name, err)
            }
            continue
        }

        // Check if instance is shutdown
        shutdown, err := instancesV2.InstanceShutdown(ctx, node)
        if err != nil {
            klog.Errorf("error checking instance shutdown for node %s: %v", node.Name, err)
            continue
        }

        if shutdown {
            // Add shutdown taint
            if err := c.addShutdownTaint(ctx, node); err != nil {
                klog.Errorf("error adding shutdown taint to node %s: %v", node.Name, err)
            }
        } else {
            // Remove shutdown taint
            if err := c.removeShutdownTaint(ctx, node); err != nil {
                klog.Errorf("error removing shutdown taint from node %s: %v", node.Name, err)
            }
        }
    }

    return nil
}

func (c *CloudNodeLifecycleController) addShutdownTaint(ctx context.Context,
    node *v1.Node) error {

    taint := &v1.Taint{
        Key:    cloudproviderapi.TaintNodeShutdown,
        Value:  "",
        Effect: v1.TaintEffectNoSchedule,
    }

    return controller.AddOrUpdateTaintOnNode(ctx, c.kubeClient, taint, node)
}
```

### **3. Service Controller (LoadBalancer)**

**File**: `staging/src/k8s.io/cloud-provider/controllers/service/controller.go`

**Purpose**: Creates and manages cloud load balancers for Services of type LoadBalancer

```go
type Controller struct {
    kubeClient       clientset.Interface
    serviceLister    corelisters.ServiceLister
    nodeLister       corelisters.NodeLister
    cloud            cloudprovider.Interface
    clusterName      string
    workqueue        workqueue.RateLimitingInterface
    recorder         record.EventRecorder
}

func New(
    cloud cloudprovider.Interface,
    kubeClient clientset.Interface,
    serviceInformer coreinformers.ServiceInformer,
    nodeInformer coreinformers.NodeInformer,
    clusterName string,
) (*Controller, error) {

    eventBroadcaster := record.NewBroadcaster()
    recorder := eventBroadcaster.NewRecorder(scheme.Scheme,
        v1.EventSource{Component: "service-controller"})

    s := &Controller{
        kubeClient:    kubeClient,
        cloud:         cloud,
        clusterName:   clusterName,
        serviceLister: serviceInformer.Lister(),
        nodeLister:    nodeInformer.Lister(),
        workqueue: workqueue.NewNamedRateLimitingQueue(
            workqueue.DefaultControllerRateLimiter(),
            "service-controller",
        ),
        recorder: recorder,
    }

    // Watch services
    serviceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            s.enqueueService(obj)
        },
        UpdateFunc: func(old, cur interface{}) {
            s.enqueueService(cur)
        },
        DeleteFunc: func(obj interface{}) {
            s.enqueueService(obj)
        },
    })

    // Watch nodes (for LoadBalancer backend updates)
    nodeInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            s.nodeSyncLoop()
        },
        UpdateFunc: func(old, cur interface{}) {
            oldNode := old.(*v1.Node)
            curNode := cur.(*v1.Node)
            if nodeReady(oldNode) != nodeReady(curNode) {
                s.nodeSyncLoop()
            }
        },
        DeleteFunc: func(obj interface{}) {
            s.nodeSyncLoop()
        },
    })

    return s, nil
}

func (s *Controller) syncLoadBalancerIfNeeded(ctx context.Context,
    service *v1.Service) error {

    // Only handle LoadBalancer services
    if service.Spec.Type != v1.ServiceTypeLoadBalancer {
        return nil
    }

    // Get cloud LoadBalancer interface
    lb, ok := s.cloud.LoadBalancer()
    if !ok {
        return errors.New("cloud provider does not support load balancers")
    }

    // Get nodes for load balancer backends
    nodes, err := s.nodeLister.ListWithPredicate(func(node *v1.Node) bool {
        return nodeReady(node) && !hasTaint(node, v1.TaintNodeUnschedulable)
    })
    if err != nil {
        return err
    }

    // Check if service is being deleted
    if service.DeletionTimestamp != nil {
        return s.ensureLoadBalancerDeleted(ctx, lb, service)
    }

    // Ensure load balancer exists and is configured correctly
    status, err := lb.EnsureLoadBalancer(ctx, s.clusterName, service, nodes)
    if err != nil {
        s.recorder.Eventf(service, v1.EventTypeWarning, "SyncLoadBalancerFailed",
            "Error syncing load balancer: %v", err)
        return err
    }

    // Update service status with load balancer info
    if !loadBalancerStatusEqual(&service.Status.LoadBalancer, status) {
        serviceCopy := service.DeepCopy()
        serviceCopy.Status.LoadBalancer = *status

        _, err = s.kubeClient.CoreV1().Services(service.Namespace).UpdateStatus(
            ctx, serviceCopy, metav1.UpdateOptions{})
        if err != nil {
            return err
        }

        s.recorder.Eventf(service, v1.EventTypeNormal, "EnsuredLoadBalancer",
            "Ensured load balancer")
    }

    return nil
}

func (s *Controller) ensureLoadBalancerDeleted(ctx context.Context,
    lb cloudprovider.LoadBalancer, service *v1.Service) error {

    err := lb.EnsureLoadBalancerDeleted(ctx, s.clusterName, service)
    if err != nil {
        s.recorder.Eventf(service, v1.EventTypeWarning, "DeleteLoadBalancerFailed",
            "Error deleting load balancer: %v", err)
        return err
    }

    s.recorder.Eventf(service, v1.EventTypeNormal, "DeletedLoadBalancer",
        "Deleted load balancer")
    return nil
}
```

### **4. Route Controller**

**File**: `staging/src/k8s.io/cloud-provider/controllers/route/route_controller.go`

**Purpose**: Manages cloud network routes for pod CIDR blocks

```go
type RouteController struct {
    kubeClient   clientset.Interface
    nodeLister   corelisters.NodeLister
    cloud        cloudprovider.Interface
    clusterName  string
    clusterCIDRs []*net.IPNet

    // Route reconciliation interval
    routeReconciliationPeriod time.Duration
}

func New(
    routes cloudprovider.Routes,
    kubeClient clientset.Interface,
    nodeInformer coreinformers.NodeInformer,
    clusterName string,
    clusterCIDRs []*net.IPNet,
) (*RouteController, error) {

    rc := &RouteController{
        kubeClient:                kubeClient,
        nodeLister:                nodeInformer.Lister(),
        cloud:                     routes,
        clusterName:               clusterName,
        clusterCIDRs:             clusterCIDRs,
        routeReconciliationPeriod: 10 * time.Second,
    }

    return rc, nil
}

func (rc *RouteController) Run(ctx context.Context) {
    defer utilruntime.HandleCrash()

    klog.Info("Starting route controller")

    // Periodic route reconciliation
    wait.UntilWithContext(ctx, func(ctx context.Context) {
        if err := rc.reconcile(ctx); err != nil {
            klog.Errorf("error reconciling routes: %v", err)
        }
    }, rc.routeReconciliationPeriod)
}

func (rc *RouteController) reconcile(ctx context.Context) error {
    routes, ok := rc.cloud.(cloudprovider.Routes)
    if !ok {
        return errors.New("cloud provider does not support routes")
    }

    // Get all nodes
    nodes, err := rc.nodeLister.List(labels.Everything())
    if err != nil {
        return err
    }

    // Get current routes from cloud provider
    cloudRoutes, err := routes.ListRoutes(ctx, rc.clusterName)
    if err != nil {
        return err
    }

    // Build map of desired routes
    desiredRoutes := make(map[types.NodeName]*cloudprovider.Route)
    for _, node := range nodes {
        if node.Spec.PodCIDR == "" {
            continue
        }
        desiredRoutes[types.NodeName(node.Name)] = &cloudprovider.Route{
            Name:            node.Name,
            TargetNode:      types.NodeName(node.Name),
            DestinationCIDR: node.Spec.PodCIDR,
        }
    }

    // Build map of existing routes
    existingRoutes := make(map[types.NodeName]*cloudprovider.Route)
    for _, route := range cloudRoutes {
        existingRoutes[route.TargetNode] = route
    }

    // Create missing routes
    for nodeName, desiredRoute := range desiredRoutes {
        if _, exists := existingRoutes[nodeName]; !exists {
            klog.Infof("Creating route for node %s with CIDR %s",
                nodeName, desiredRoute.DestinationCIDR)

            err := routes.CreateRoute(ctx, rc.clusterName,
                string(nodeName), desiredRoute)
            if err != nil {
                klog.Errorf("Could not create route for node %s: %v", nodeName, err)
            }
        }
    }

    // Delete orphaned routes
    for nodeName, existingRoute := range existingRoutes {
        if _, desired := desiredRoutes[nodeName]; !desired {
            klog.Infof("Deleting route for node %s with CIDR %s",
                nodeName, existingRoute.DestinationCIDR)

            err := routes.DeleteRoute(ctx, rc.clusterName, existingRoute)
            if err != nil {
                klog.Errorf("Could not delete route for node %s: %v", nodeName, err)
            }
        }
    }

    return nil
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚙️ Configuration**

### **Configuration Structure**

**File**: `staging/src/k8s.io/cloud-provider/options/options.go`

```go
type CloudControllerManagerOptions struct {
    // Generic controller manager options
    Generic *cmoptions.GenericControllerManagerConfigurationOptions

    // Cloud provider shared options
    KubeCloudShared *KubeCloudSharedOptions

    // Service controller options
    ServiceController *ServiceControllerOptions

    // Node controller options (status update frequency)
    NodeController *NodeControllerOptions

    // Secure serving options
    SecureServing *apiserveroptions.SecureServingOptionsWithLoopback

    // Authentication options
    Authentication *apiserveroptions.DelegatingAuthenticationOptions

    // Authorization options
    Authorization *apiserveroptions.DelegatingAuthorizationOptions

    // Webhook serving options
    WebhookServing *WebhookServingOptions

    // Node status update frequency
    NodeStatusUpdateFrequency metav1.Duration
}

func NewCloudControllerManagerOptions() (*CloudControllerManagerOptions, error) {
    return &CloudControllerManagerOptions{
        Generic: cmoptions.NewGenericControllerManagerConfigurationOptions(
            &cloudcontrollerconfig.CloudControllerManagerConfiguration{
                Generic: cmconfig.GenericControllerManagerConfiguration{
                    Port:            10258,
                    Address:         "0.0.0.0",
                    MinResyncPeriod: metav1.Duration{Duration: 12 * time.Hour},
                    LeaderElection: componentbaseconfig.LeaderElectionConfiguration{
                        LeaderElect:   true,
                        LeaseDuration: metav1.Duration{Duration: 15 * time.Second},
                        RenewDeadline: metav1.Duration{Duration: 10 * time.Second},
                        RetryPeriod:   metav1.Duration{Duration: 2 * time.Second},
                        ResourceLock:  "leases",
                        ResourceName:  "cloud-controller-manager",
                    },
                    Controllers: []string{"*"},
                },
            },
        ),
        KubeCloudShared: &KubeCloudSharedOptions{
            KubeCloudSharedConfiguration: &cpconfig.KubeCloudSharedConfiguration{
                RouteReconciliationPeriod:  metav1.Duration{Duration: 10 * time.Second},
                NodeMonitorPeriod:          metav1.Duration{Duration: 5 * time.Second},
                ClusterName:                "kubernetes",
                AllocateNodeCIDRs:          false,
                CIDRAllocatorType:          "",
                ConfigureCloudRoutes:       true,
            },
        },
        ServiceController: &ServiceControllerOptions{
            ConcurrentServiceSyncs: 1,
        },
        NodeStatusUpdateFrequency: metav1.Duration{Duration: 5 * time.Minute},
    }, nil
}
```

### **Common Command Line Flags**

```bash
# Basic CCM configuration
cloud-controller-manager \
  # Cloud provider
  --cloud-provider=aws \
  --cloud-config=/etc/kubernetes/cloud-config.conf \

  # Cluster identification
  --cluster-name=production \
  --cluster-cidr=10.244.0.0/16 \

  # Controllers
  --controllers=*,-cloud-node-lifecycle \
  --concurrent-service-syncs=5 \

  # Node monitoring
  --node-monitor-period=5s \
  --node-status-update-frequency=5m \

  # Route configuration
  --configure-cloud-routes=true \
  --route-reconciliation-period=10s \

  # API server connection
  --kubeconfig=/etc/kubernetes/cloud-controller-manager.kubeconfig \

  # Leader election
  --leader-elect=true \
  --leader-elect-lease-duration=15s \
  --leader-elect-renew-deadline=10s \
  --leader-elect-retry-period=2s \
  --leader-elect-resource-name=cloud-controller-manager \
  --leader-elect-resource-namespace=kube-system \

  # Observability
  --bind-address=0.0.0.0 \
  --secure-port=10258 \
  --profiling=true \
  --v=2
```

### **Controller Selection**

```bash
# Run all controllers (default)
--controllers=*

# Disable specific controller
--controllers=*,-cloud-node-lifecycle

# Run only specific controllers
--controllers=cloud-node-controller,service-lb-controller

# Common controller names:
# - cloud-node-controller
# - cloud-node-lifecycle-controller
# - service-lb-controller
# - node-route-controller
# - node-ipam-controller (optional)
```

### **Cloud Configuration File**

**AWS Example** (`/etc/kubernetes/cloud-config.conf`):

```ini
[Global]
Zone = us-east-1a
VPC = vpc-0123456789abcdef0
SubnetID = subnet-0123456789abcdef0
RouteTableID = rtb-0123456789abcdef0
RoleARN = arn:aws:iam::123456789012:role/kubernetes-cloud-controller
KubernetesClusterTag = my-cluster
KubernetesClusterID = my-cluster
DisableSecurityGroupIngress = false
ElbSecurityGroup = sg-0123456789abcdef0
```

**GCE Example**:

```ini
[Global]
project-id = my-project
network-name = my-network
subnetwork-name = my-subnetwork
node-tags = kubernetes-node
node-instance-prefix = gke-cluster

[Networking]
network-project-id = my-network-project
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚀 Deployment Patterns**

### **Deployment Manifest**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloud-controller-manager
  namespace: kube-system
  labels:
    component: cloud-controller-manager
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
      # Run on control plane nodes
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
        image: registry.k8s.io/cloud-controller-manager-aws:v1.28.0
        command:
        - /cloud-controller-manager
        args:
        - --cloud-provider=aws
        - --cloud-config=/etc/kubernetes/cloud.conf
        - --cluster-name=my-cluster
        - --cluster-cidr=10.244.0.0/16
        - --allocate-node-cidrs=true
        - --configure-cloud-routes=false
        - --leader-elect=true
        - --use-service-account-credentials=true
        - --v=2
        volumeMounts:
        - name: cloud-config
          mountPath: /etc/kubernetes
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
          timeoutSeconds: 15
        ports:
        - containerPort: 10258
          name: https
          protocol: TCP
      volumes:
      - name: cloud-config
        secret:
          secretName: cloud-config
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
# Node management
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch", "delete", "patch", "update"]
- apiGroups: [""]
  resources: ["nodes/status"]
  verbs: ["patch", "update"]

# Service management
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: [""]
  resources: ["services/status"]
  verbs: ["patch", "update"]

# Endpoint management
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get", "list", "watch", "create", "update"]

# Event creation
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch", "update"]

# Service account tokens
- apiGroups: [""]
  resources: ["serviceaccounts"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["serviceaccounts/token"]
  verbs: ["create"]

# Secrets access (for TLS)
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]

# Configmaps
- apiGroups: [""]
  resources: ["configmaps"]
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

### **Node Initialization Taint**

When using CCM, nodes must be tainted to prevent pod scheduling until CCM initializes them:

**Kubelet Configuration**:

```yaml
# /etc/kubernetes/kubelet.conf
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
registerWithTaints:
- key: "node.cloudprovider.kubernetes.io/uninitialized"
  value: "true"
  effect: "NoSchedule"
```

**Kubelet Flags**:

```bash
kubelet \
  --cloud-provider=external \
  --register-with-taints="node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚨 Production Troubleshooting**

### **Scenario 1: Nodes Not Being Initialized**

**Symptoms**:
```bash
# Nodes stuck with uninitialized taint
kubectl get nodes -o wide
# NAME      STATUS   ROLES    AGE   VERSION
# node-1    Ready    <none>   10m   v1.28.0
# (but pods not scheduling)

kubectl describe node node-1 | grep Taint
# Taints: node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
```

**Investigation**:

```bash
# 1. Check CCM logs
kubectl logs -n kube-system -l component=cloud-controller-manager

# 2. Check if CCM is leader
kubectl get lease -n kube-system cloud-controller-manager -o yaml

# 3. Check cloud provider connection
kubectl logs -n kube-system -l component=cloud-controller-manager | grep -i "error\|failed"

# 4. Verify node has provider ID
kubectl get node node-1 -o jsonpath='{.spec.providerID}'
```

**Common Causes**:

1. **Cloud credentials invalid**:
```bash
# Check cloud config secret
kubectl get secret -n kube-system cloud-config -o yaml | base64 -d
```

2. **Provider ID mismatch**:
```bash
# Manually set provider ID if needed
kubectl patch node node-1 -p '{"spec":{"providerID":"aws:///us-east-1a/i-0123456789abcdef0"}}'
```

3. **InstancesV2 not implemented**:
```bash
# Check if cloud provider supports InstancesV2
# Older providers may only support Instances()
```

### **Scenario 2: LoadBalancer Service Stuck Pending**

**Symptoms**:
```bash
kubectl get svc my-service
# NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# my-service   LoadBalancer   10.96.10.100   <pending>     80:30080/TCP   10m
```

**Investigation**:

```bash
# 1. Check service events
kubectl describe svc my-service

# Events:
#   Warning  SyncLoadBalancerFailed  2m  service-controller  Error syncing load balancer:
#            AccessDenied: User: arn:aws:... is not authorized to perform: elasticloadbalancing:CreateLoadBalancer

# 2. Check CCM service controller logs
kubectl logs -n kube-system -l component=cloud-controller-manager | grep "service-controller"

# 3. Check cloud provider quotas
# (Provider-specific - check AWS ELB limits, etc.)

# 4. Verify subnet tags
# AWS requires specific subnet tags for ELB creation
```

**Common Causes**:

1. **IAM permissions missing**:
```json
{
  "Effect": "Allow",
  "Action": [
    "elasticloadbalancing:CreateLoadBalancer",
    "elasticloadbalancing:DeleteLoadBalancer",
    "elasticloadbalancing:DescribeLoadBalancers",
    "elasticloadbalancing:ModifyLoadBalancerAttributes"
  ],
  "Resource": "*"
}
```

2. **Subnet not tagged**:
```bash
# AWS requires tags for subnet discovery
aws ec2 create-tags --resources subnet-xxx \
  --tags Key=kubernetes.io/cluster/my-cluster,Value=shared
```

3. **Security group issues**:
```bash
# Check if ELB security group allows traffic
aws ec2 describe-security-groups --group-ids sg-xxx
```

### **Scenario 3: Routes Not Created**

**Symptoms**:
```bash
# Pods can't communicate across nodes
kubectl get pods -o wide
# Pod on node-1 can't reach pod on node-2
```

**Investigation**:

```bash
# 1. Check route controller logs
kubectl logs -n kube-system -l component=cloud-controller-manager | grep "route-controller"

# 2. Check node PodCIDR
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# 3. Check cloud routes
# AWS:
aws ec2 describe-route-tables --filters "Name=tag:KubernetesCluster,Values=my-cluster"

# GCE:
gcloud compute routes list --filter="network=my-network"

# 4. Check if configure-cloud-routes is enabled
# --configure-cloud-routes=true
```

**Common Causes**:

1. **PodCIDR not allocated**:
```bash
# Ensure kube-controller-manager allocates CIDRs
# Or enable it in CCM:
--allocate-node-cidrs=true
--cluster-cidr=10.244.0.0/16
```

2. **Route table not found**:
```bash
# Verify cloud config has correct route table ID
```

3. **Route quota exceeded**:
```bash
# AWS has limits on routes per route table (100 default)
aws ec2 describe-route-tables --route-table-ids rtb-xxx
```

### **Scenario 4: CCM Leader Election Flapping**

**Symptoms**:
```bash
# Frequent leader changes
kubectl logs -n kube-system -l component=cloud-controller-manager | grep "leader"
# I0115 10:20:00.000000 acquired leader lease
# I0115 10:20:15.000000 failed to renew leader lease
# I0115 10:20:16.000000 acquired leader lease
```

**Investigation**:

```bash
# 1. Check API server connectivity
kubectl logs -n kube-system -l component=cloud-controller-manager | grep "timeout\|connection refused"

# 2. Check network latency
kubectl exec -n kube-system cloud-controller-manager-xxx -- ping kubernetes.default.svc

# 3. Check resource constraints
kubectl top pods -n kube-system -l component=cloud-controller-manager

# 4. Check lease object
kubectl get lease -n kube-system cloud-controller-manager -o yaml
```

**Fixes**:

```yaml
# Increase leader election timeouts
args:
- --leader-elect-lease-duration=30s
- --leader-elect-renew-deadline=20s
- --leader-elect-retry-period=5s
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💡 Best Practices**

### **Deployment**

1. **High Availability**:
```yaml
spec:
  replicas: 3  # Run multiple replicas
  # Leader election ensures only one is active
```

2. **Resource Requests**:
```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

3. **Priority Class**:
```yaml
priorityClassName: system-node-critical
```

4. **Health Checks**:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 10258
    scheme: HTTPS
readinessProbe:
  httpGet:
    path: /healthz
    port: 10258
    scheme: HTTPS
```

### **Configuration**

1. **Minimal Permissions**:
   - Only grant required IAM/cloud permissions
   - Use service account tokens for API server
   - Rotate credentials regularly

2. **Controller Selection**:
```bash
# Disable controllers you don't need
--controllers=*,-node-route-controller  # If using CNI that manages routes
```

3. **Rate Limiting**:
```bash
# Prevent cloud API throttling
--concurrent-service-syncs=5
--node-monitor-period=30s
```

4. **Cluster Identification**:
```bash
# Consistent cluster naming
--cluster-name=production-us-east-1
```

### **Monitoring**

1. **Key Metrics**:
```promql
# Controller work queue depth
workqueue_depth{name=~"cloud.*"}

# Cloud API latency
cloud_provider_api_duration_seconds_bucket

# Service sync errors
service_controller_sync_errors_total

# Node lifecycle events
node_lifecycle_controller_deletions_total
```

2. **Alerts**:
```yaml
- alert: CCMNotLeader
  expr: absent(leader_election_master_status{name="cloud-controller-manager"} == 1)
  for: 5m

- alert: CCMHighErrorRate
  expr: rate(service_controller_sync_errors_total[5m]) > 0.1

- alert: CCMWorkqueueHigh
  expr: workqueue_depth{name=~"cloud.*"} > 100
```

### **Security**

1. **Credentials Management**:
```yaml
# Use Kubernetes secrets
- name: cloud-config
  secret:
    secretName: cloud-config

# Or IRSA/Workload Identity
```

2. **Network Policies**:
```yaml
# Restrict CCM network access
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cloud-controller-manager
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      component: cloud-controller-manager
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - port: 443
      protocol: TCP
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 References**

### **Source Code**
- **Main Entry**: `cmd/cloud-controller-manager/main.go`
- **Controller Manager**: `staging/src/k8s.io/cloud-provider/app/controllermanager.go`
- **Cloud Interface**: `staging/src/k8s.io/cloud-provider/cloud.go`
- **Node Controller**: `staging/src/k8s.io/cloud-provider/controllers/node/node_controller.go`
- **Service Controller**: `staging/src/k8s.io/cloud-provider/controllers/service/controller.go`
- **Route Controller**: `staging/src/k8s.io/cloud-provider/controllers/route/route_controller.go`
- **Configuration**: `staging/src/k8s.io/cloud-provider/options/options.go`
- **Provider Plugins**: `staging/src/k8s.io/cloud-provider/plugins.go`

### **Related Documentation**
- **Cloud Provider Interface**: `docs/architecture/claude/cloud-integration/02-cloud-provider-interface.md`
- **LoadBalancer Integration**: `docs/architecture/claude/cloud-integration/03-loadbalancer-integration.md`
- **Controller Manager Guide**: `docs/architecture/claude/controller-manager/`

### **External Resources**
- **Cloud Controller Manager**: https://kubernetes.io/docs/concepts/architecture/cloud-controller/
- **Cloud Provider AWS**: https://github.com/kubernetes/cloud-provider-aws
- **Cloud Provider GCE**: https://github.com/kubernetes/cloud-provider-gcp
- **Cloud Provider Azure**: https://github.com/kubernetes-sigs/cloud-provider-azure

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: Complete
**Last Updated**: 2024-11-17
**Target Audience**: Cloud provider engineers, platform engineers, SREs
**Scope**: CCM architecture, controllers, configuration, and production patterns
