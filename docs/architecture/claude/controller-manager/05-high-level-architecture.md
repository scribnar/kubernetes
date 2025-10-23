# Kube-Controller-Manager: High-Level Architecture

**Document Version**: 1.0
**Last Updated**: 2025-10-21
**Status**: Draft

---

## 1. System Context

### 1.1 Kubernetes Control Plane Context

```mermaid
C4Context
    title System Context - Kube-Controller-Manager in Kubernetes

    Person(admin, "Cluster Administrator", "Manages cluster resources")
    Person(developer, "Application Developer", "Deploys applications")

    System_Boundary(controlplane, "Kubernetes Control Plane") {
        System(kcm, "kube-controller-manager", "Reconciles cluster state")
        System(apiserver, "kube-apiserver", "API gateway & etcd frontend")
        System(scheduler, "kube-scheduler", "Pod placement")
    }

    System_Ext(etcd, "etcd", "Persistent state store")
    System_Boundary(nodes, "Worker Nodes") {
        System(kubelet, "kubelet", "Node agent")
        System(pods, "Pods", "Running workloads")
    }

    Rel(admin, apiserver, "Manages via kubectl")
    Rel(developer, apiserver, "Deploys apps")
    Rel(kcm, apiserver, "Watches & Updates resources")
    Rel(scheduler, apiserver, "Watches unscheduled pods")
    Rel(apiserver, etcd, "Persists state")
    Rel(kubelet, apiserver, "Reports status")
    Rel(kubelet, pods, "Manages")
```

### 1.2 External Dependencies

```mermaid
graph TB
    subgraph "kube-controller-manager"
        KCM[Controller Manager Process]
    end

    subgraph "Required Dependencies"
        API[kube-apiserver<br/>Must be reachable]
        ETCD[(etcd<br/>via API server)]
    end

    subgraph "Optional Dependencies"
        METRICS[metrics-server<br/>For HPA]
        CLOUD[Cloud Provider APIs<br/>Deprecated in v1.31]
    end

    subgraph "Configuration Sources"
        KUBECONFIG[kubeconfig file<br/>Authentication]
        FLAGS[CLI flags<br/>Configuration]
        SAKEYS[SA signing keys<br/>Token generation]
        CACERTS[CA certificates<br/>For CSR signing]
    end

    KCM -->|Watch/Update| API
    API --> ETCD
    KCM -.->|Optional| METRICS
    KCM -.->|Deprecated| CLOUD
    KCM --> KUBECONFIG
    KCM --> FLAGS
    KCM --> SAKEYS
    KCM --> CACERTS

    style API fill:#f96,stroke:#333,stroke-width:2px
    style KCM fill:#9f6,stroke:#333,stroke-width:4px
```

---

## 2. High-Level Component Architecture

### 2.1 Layered Architecture

```mermaid
graph TB
    subgraph "Layer 1: Entry & Configuration"
        MAIN[Main Entry Point<br/>controller-manager.go]
        OPTS[Options Parser<br/>KubeControllerManagerOptions]
        CONFIG[Configuration Builder<br/>CompletedConfig]
    end

    subgraph "Layer 2: Infrastructure Services"
        LEADER[Leader Election<br/>Lease-based]
        HTTP[HTTP Server<br/>Metrics, Health, Debug]
        EVENTS[Event Broadcasting<br/>Event Recording]
    end

    subgraph "Layer 3: Shared Resources"
        CLIENTB[Client Builders<br/>Per-controller auth]
        INFORMER[Shared Informer Factory<br/>Watch & Cache]
        META[Metadata Informer<br/>Generic controllers]
        MAPPER[REST Mapper<br/>Dynamic discovery]
        GRAPH[Graph Builder<br/>GC dependencies]
    end

    subgraph "Layer 4: Controller Context"
        CTX[ControllerContext<br/>Shared state for controllers]
    end

    subgraph "Layer 5: Controllers (50)"
        C1[Deployment]
        C2[ReplicaSet]
        C3[StatefulSet]
        CDOTS[... 47 more ...]
    end

    MAIN --> OPTS
    OPTS --> CONFIG
    CONFIG --> LEADER
    CONFIG --> HTTP
    CONFIG --> EVENTS
    LEADER -->|On elected| CLIENTB
    CLIENTB --> INFORMER
    CLIENTB --> META
    CLIENTB --> MAPPER
    CLIENTB --> GRAPH
    INFORMER --> CTX
    META --> CTX
    MAPPER --> CTX
    GRAPH --> CTX
    CLIENTB --> CTX
    CTX --> C1
    CTX --> C2
    CTX --> C3
    CTX --> CDOTS

    style CTX fill:#ff9,stroke:#333,stroke-width:2px
    style INFORMER fill:#9ff,stroke:#333,stroke-width:2px
```

### 2.2 Process Architecture

```mermaid
graph LR
    subgraph "Single OS Process"
        subgraph "Main Thread"
            MAIN[Main Goroutine<br/>Leader Election Loop]
        end

        subgraph "Infrastructure Goroutines"
            HTTP[HTTP Server<br/>~5 goroutines]
            REF1[Reflector: Deployments]
            REF2[Reflector: ReplicaSets]
            REF3[Reflector: Pods]
            REFN[Reflector: ... more]
            DISC[Discovery Refresh<br/>30s interval]
        end

        subgraph "Controller Goroutines"
            DC_MAIN[Deployment Main]
            DC_W1[Deployment Worker 1]
            DC_W2[Deployment Worker 2]
            DC_WN[Deployment Worker N]

            RS_MAIN[ReplicaSet Main]
            RS_W1[RS Worker 1]
            RS_WN[RS Worker N]

            MORE[... 48 more controllers]
        end
    end

    MAIN --> HTTP
    MAIN --> REF1
    MAIN --> REF2
    MAIN --> REF3
    MAIN --> REFN
    MAIN --> DISC
    MAIN --> DC_MAIN
    MAIN --> RS_MAIN
    MAIN --> MORE

    DC_MAIN --> DC_W1
    DC_MAIN --> DC_W2
    DC_MAIN --> DC_WN
    RS_MAIN --> RS_W1
    RS_MAIN --> RS_WN
```

**Typical Goroutine Count**: 150-250 depending on configuration and number of resource types watched.

---

## 3. Major Components

### 3.1 Controller Context (Shared State)

The ControllerContext is the central data structure passed to all controllers:

```mermaid
classDiagram
    class ControllerContext {
        +ClientBuilder clientbuilder.ControllerClientBuilder
        +InformerFactory informers.SharedInformerFactory
        +ObjectOrMetadataInformerFactory informerfactory.InformerFactory
        +ComponentConfig KubeControllerManagerConfiguration
        +RESTMapper *restmapper.DeferredDiscoveryRESTMapper
        +InformersStarted chan struct{}
        +ResyncPeriod func() time.Duration
        +ControllerManagerMetrics *ControllerManagerMetrics
        +GraphBuilder *garbagecollector.GraphBuilder
        +IsControllerEnabled(descriptor) bool
        +NewClient(name) clientset.Interface
        +NewClientConfig(name) *rest.Config
    }

    class SharedInformerFactory {
        +Core() CoreInformers
        +Apps() AppsInformers
        +Batch() BatchInformers
        +Storage() StorageInformers
        +Start(stopCh)
        +WaitForCacheSync()
    }

    class ClientBuilder {
        +Client(name) clientset.Interface
        +Config(name) *rest.Config
        +DiscoveryClient(name) discovery.Interface
    }

    class RESTMapper {
        +RESTMapping(gk) *RESTMapping
        +Reset()
    }

    class GraphBuilder {
        +monitors map[schema.GroupVersionResource]*monitor
        +dependencyGraphBuilder
    }

    ControllerContext --> SharedInformerFactory
    ControllerContext --> ClientBuilder
    ControllerContext --> RESTMapper
    ControllerContext --> GraphBuilder
```

**Source**: `cmd/kube-controller-manager/app/controllermanager.go:406`

---

### 3.2 Controller Descriptor System

```mermaid
classDiagram
    class ControllerDescriptor {
        -name string
        -constructor ControllerConstructor
        -requiredFeatureGates []featuregate.Feature
        -aliases []string
        -isDisabledByDefault bool
        -isCloudProviderController bool
        -requiresSpecialHandling bool
        +Name() string
        +GetControllerConstructor() ControllerConstructor
        +GetRequiredFeatureGates() []Feature
        +GetAliases() []string
        +IsDisabledByDefault() bool
        +IsCloudProviderController() bool
        +RequiresSpecialHandling() bool
        +BuildController(ctx, controllerCtx) Controller
    }

    class Controller {
        <<interface>>
        +Name() string
        +Run(ctx context.Context)
    }

    class ControllerConstructor {
        <<function type>>
        func(ctx, controllerContext, name) (Controller, error)
    }

    class Debuggable {
        <<interface>>
        +DebuggingHandler() http.Handler
    }

    class HealthCheckable {
        <<interface>>
        +HealthChecker() healthz.UnnamedHealthChecker
    }

    ControllerDescriptor --> ControllerConstructor
    ControllerConstructor ..> Controller : creates
    Controller <|.. Debuggable : optional
    Controller <|.. HealthCheckable : optional
```

**Source**: `cmd/kube-controller-manager/app/controller_descriptor.go:50`

---

### 3.3 Shared Informer Architecture

```mermaid
graph TB
    subgraph "API Server"
        API[kube-apiserver<br/>Watch Endpoint]
    end

    subgraph "Shared Informer Factory"
        FACTORY[SharedInformerFactory]

        subgraph "Pod Informer"
            POD_REF[Reflector<br/>List & Watch]
            POD_FIFO[DeltaFIFO<br/>Event Queue]
            POD_CACHE[Indexer<br/>Local Cache]
            POD_DIST[Event Distributor]
        end

        subgraph "Deployment Informer"
            DEP_REF[Reflector]
            DEP_FIFO[DeltaFIFO]
            DEP_CACHE[Indexer]
            DEP_DIST[Event Distributor]
        end
    end

    subgraph "Controllers"
        DC[Deployment Controller<br/>EventHandler]
        RSC[ReplicaSet Controller<br/>EventHandler]
        DSC[DaemonSet Controller<br/>EventHandler]
    end

    API -->|Single Watch| POD_REF
    API -->|Single Watch| DEP_REF

    POD_REF --> POD_FIFO
    POD_FIFO --> POD_CACHE
    POD_FIFO --> POD_DIST
    POD_DIST --> DC
    POD_DIST --> RSC
    POD_DIST --> DSC

    DEP_REF --> DEP_FIFO
    DEP_FIFO --> DEP_CACHE
    DEP_FIFO --> DEP_DIST
    DEP_DIST --> DC

    DC -.->|Read from cache| POD_CACHE
    DC -.->|Read from cache| DEP_CACHE
    RSC -.->|Read from cache| POD_CACHE
    DSC -.->|Read from cache| POD_CACHE

    style POD_CACHE fill:#9f9,stroke:#333,stroke-width:2px
    style DEP_CACHE fill:#9f9,stroke:#333,stroke-width:2px
```

**Key Benefits**:
- **Single watch per resource type** (not per controller)
- **Shared cache** reduces memory
- **Consistent view** across controllers
- **Reduced API server load**

---

### 3.4 Work Queue Pattern

```mermaid
graph TB
    subgraph "Event Flow"
        INFORMER[Informer Event<br/>OnAdd/OnUpdate/OnDelete]
        HANDLER[Event Handler<br/>Fast, non-blocking]
        KEY[Extract Key<br/>namespace/name]
    end

    subgraph "Rate-Limiting Work Queue"
        QUEUE[Queue Interface]
        WAITING[Waiting Area<br/>Deduplication]
        LIMITER[Rate Limiter<br/>Exponential Backoff]
        READY[Ready Queue<br/>FIFO]
    end

    subgraph "Worker Pool"
        W1[Worker 1]
        W2[Worker 2]
        WN[Worker N]
    end

    subgraph "Processing"
        GET[Get item]
        SYNC[Sync Handler<br/>Reconcile]
        RESULT{Success?}
        DONE[Done]
        REQUEUE[Requeue with<br/>backoff]
    end

    INFORMER --> HANDLER
    HANDLER --> KEY
    KEY --> QUEUE
    QUEUE --> WAITING
    WAITING --> LIMITER
    LIMITER --> READY

    READY --> W1
    READY --> W2
    READY --> WN

    W1 --> GET
    W2 --> GET
    WN --> GET

    GET --> SYNC
    SYNC --> RESULT
    RESULT -->|Yes| DONE
    RESULT -->|No| REQUEUE
    REQUEUE --> LIMITER

    style WAITING fill:#ff9,stroke:#333,stroke-width:2px
    style LIMITER fill:#f99,stroke:#333,stroke-width:2px
```

---

## 4. Data Flow Patterns

### 4.1 Watch-Based Event Flow

```mermaid
sequenceDiagram
    participant API as kube-apiserver
    participant Reflector
    participant Cache as Local Cache
    participant Handler as Event Handler
    participant Queue as Work Queue
    participant Worker
    participant Sync as Sync Handler

    Note over Reflector: Initialization
    Reflector->>API: LIST /api/v1/pods
    API-->>Reflector: All pods (rv: 1000)
    Reflector->>Cache: Update cache
    Reflector->>Handler: OnAdd() for each pod
    Handler->>Queue: Enqueue keys

    Note over Reflector: Continuous watching
    Reflector->>API: WATCH /api/v1/pods?rv=1000

    loop Event stream
        API-->>Reflector: ADDED pod-x (rv: 1001)
        Reflector->>Cache: Add pod-x
        Reflector->>Handler: OnAdd(pod-x)
        Handler->>Queue: Enqueue "ns/pod-x"

        Worker->>Queue: Get()
        Queue-->>Worker: "ns/pod-x"
        Worker->>Cache: Get("ns/pod-x")
        Cache-->>Worker: pod-x object
        Worker->>Sync: syncPod(pod-x)

        alt Success
            Sync-->>Worker: nil
            Worker->>Queue: Done("ns/pod-x")
        else Error
            Sync-->>Worker: error
            Worker->>Queue: Requeue("ns/pod-x")
        end
    end
```

### 4.2 Controller Coordination Example

**Scenario**: User creates Deployment

```mermaid
sequenceDiagram
    participant User
    participant API
    participant DC as Deployment Controller
    participant RSC as ReplicaSet Controller
    participant Scheduler
    participant Kubelet

    User->>API: POST Deployment (replicas=3)
    API->>API: Store in etcd
    API-->>DC: WATCH: ADDED Deployment

    DC->>DC: syncDeployment()
    DC->>API: LIST ReplicaSets (none found)
    DC->>API: POST ReplicaSet (replicas=3)
    API->>API: Store in etcd
    API-->>RSC: WATCH: ADDED ReplicaSet

    RSC->>RSC: syncReplicaSet()
    RSC->>API: LIST Pods (none found)
    RSC->>API: POST Pod-1
    RSC->>API: POST Pod-2
    RSC->>API: POST Pod-3

    API-->>Scheduler: WATCH: ADDED Pod-1,2,3
    Scheduler->>API: BIND Pod-1 -> node-1
    Scheduler->>API: BIND Pod-2 -> node-2
    Scheduler->>API: BIND Pod-3 -> node-3

    API-->>Kubelet: WATCH: Pod bound to this node
    Kubelet->>Kubelet: Start containers
    Kubelet->>API: UPDATE Pod status: Running

    API-->>RSC: WATCH: Pod status updated
    RSC->>RSC: Update ReplicaSet status
    RSC->>API: UPDATE ReplicaSet (availableReplicas=3)

    API-->>DC: WATCH: ReplicaSet updated
    DC->>DC: Update Deployment status
    DC->>API: UPDATE Deployment (availableReplicas=3)
```

---

## 5. Leader Election Architecture

### 5.1 Leader Election Flow

```mermaid
stateDiagram-v2
    [*] --> Startup: Process starts
    Startup --> Contending: Attempt to acquire lease

    Contending --> Leader: Lease acquired
    Contending --> Standby: Lease held by other

    Leader --> Running: Start controllers
    Running --> Running: Renew lease every 5s
    Running --> Lost: Lease renewal failed
    Running --> Graceful: Context cancelled

    Standby --> Standby: Retry every 2s
    Standby --> Contending: Lease expired

    Lost --> [*]: Exit process
    Graceful --> [*]: Exit process

    note right of Running
        LeaseDuration: 15s
        RenewDeadline: 10s
        RetryPeriod: 2s
    end note
```

### 5.2 Multi-Instance Deployment

```mermaid
graph TB
    subgraph "Instance 1 (Leader)"
        I1_MAIN[Main Process]
        I1_LEASE[Lease Holder<br/>holderIdentity: instance-1]
        I1_CTRL[50 Controllers<br/>ACTIVE]

        I1_MAIN --> I1_LEASE
        I1_LEASE -->|Renew every 5s| I1_LEASE
        I1_MAIN --> I1_CTRL
    end

    subgraph "Instance 2 (Standby)"
        I2_MAIN[Main Process]
        I2_WATCH[Lease Watcher<br/>Retry every 2s]
        I2_CTRL[50 Controllers<br/>IDLE]

        I2_MAIN --> I2_WATCH
        I2_WATCH -.->|Waiting| I2_CTRL
    end

    subgraph "Instance 3 (Standby)"
        I3_MAIN[Main Process]
        I3_WATCH[Lease Watcher<br/>Retry every 2s]
        I3_CTRL[50 Controllers<br/>IDLE]

        I3_MAIN --> I3_WATCH
        I3_WATCH -.->|Waiting| I3_CTRL
    end

    subgraph "kube-apiserver"
        LEASE[Lease Object<br/>kube-controller-manager<br/>kube-system namespace]
    end

    I1_LEASE <-->|Renew| LEASE
    I2_WATCH -.->|Try acquire| LEASE
    I3_WATCH -.->|Try acquire| LEASE

    style I1_CTRL fill:#9f9,stroke:#333,stroke-width:2px
    style I2_CTRL fill:#ccc,stroke:#333,stroke-width:1px
    style I3_CTRL fill:#ccc,stroke:#333,stroke-width:1px
```

---

## 6. HTTP Server & Observability

### 6.1 HTTP Endpoints

```mermaid
graph LR
    subgraph "HTTP Server :10257"
        HANDLER[Handler Chain]

        subgraph "Endpoints"
            HEALTH[/healthz<br/>Health Check]
            LIVE[/livez<br/>Liveness]
            READY[/readyz<br/>Readiness]
            METRICS[/metrics<br/>Prometheus]
            CONFIGZ[/configz<br/>Configuration]
            DEBUG[/debug/controllers/<br/>Per-controller debug]
            PROF[/debug/pprof/*<br/>Profiling]
            FLAGZ[/flagz<br/>Flags]
        end
    end

    subgraph "Middleware"
        AUTH[Authentication]
        AUTHZ[Authorization]
        LOG[HTTP Logging]
        PANIC[Panic Recovery]
    end

    CLIENT[Monitoring Client] --> AUTH
    AUTH --> AUTHZ
    AUTHZ --> LOG
    LOG --> PANIC
    PANIC --> HANDLER
    HANDLER --> HEALTH
    HANDLER --> METRICS
    HANDLER --> CONFIGZ
    HANDLER --> DEBUG
    HANDLER --> PROF
    HANDLER --> FLAGZ
```

**Source**: `staging/src/k8s.io/controller-manager/app/serve.go:57`

### 6.2 Metrics Architecture

```mermaid
graph TB
    subgraph "Controllers"
        C1[Deployment Controller]
        C2[ReplicaSet Controller]
        CN[... more ...]
    end

    subgraph "Metrics Collection"
        WQ_METRICS[WorkQueue Metrics<br/>depth, duration, retries]
        CTRL_METRICS[Controller Metrics<br/>started, stopped]
        CLIENT_METRICS[REST Client Metrics<br/>requests, latency]
        CUSTOM[Controller-specific<br/>Custom metrics]
    end

    subgraph "Prometheus Registry"
        REGISTRY[Legacy Registry<br/>Global metrics]
    end

    subgraph "Exposition"
        HTTP_METRICS[/metrics endpoint<br/>Prometheus format]
    end

    C1 --> WQ_METRICS
    C1 --> CTRL_METRICS
    C1 --> CLIENT_METRICS
    C1 --> CUSTOM

    C2 --> WQ_METRICS
    C2 --> CTRL_METRICS
    C2 --> CLIENT_METRICS

    CN --> WQ_METRICS

    WQ_METRICS --> REGISTRY
    CTRL_METRICS --> REGISTRY
    CLIENT_METRICS --> REGISTRY
    CUSTOM --> REGISTRY

    REGISTRY --> HTTP_METRICS

    PROMETHEUS[Prometheus Server] -.->|Scrape| HTTP_METRICS
```

---

## 7. Security Architecture

### 7.1 Authentication & Authorization

```mermaid
graph TB
    subgraph "kube-controller-manager"
        PROCESS[Controller Manager Process]
        ROOT_CLIENT[Root Client<br/>Full permissions]
        SA_CLIENTS[Per-Controller Clients<br/>Limited permissions]
    end

    subgraph "Authentication"
        KUBECONFIG[kubeconfig<br/>Initial credentials]
        SA_TOKENS[ServiceAccount Tokens<br/>Dynamic]
    end

    subgraph "kube-apiserver"
        AUTHN[Authentication]
        AUTHZ[Authorization<br/>RBAC]
    end

    subgraph "Permissions"
        ROOT_RBAC[ClusterRole:<br/>system:kube-controller-manager]
        DC_RBAC[ClusterRole:<br/>system:controller:deployment-controller]
        RSC_RBAC[ClusterRole:<br/>system:controller:replicaset-controller]
        MORE_RBAC[... per-controller roles ...]
    end

    PROCESS --> ROOT_CLIENT
    PROCESS --> SA_CLIENTS

    ROOT_CLIENT --> KUBECONFIG
    SA_CLIENTS --> SA_TOKENS

    KUBECONFIG --> AUTHN
    SA_TOKENS --> AUTHN

    AUTHN --> AUTHZ

    AUTHZ --> ROOT_RBAC
    AUTHZ --> DC_RBAC
    AUTHZ --> RSC_RBAC
    AUTHZ --> MORE_RBAC

    style ROOT_CLIENT fill:#f99,stroke:#333,stroke-width:2px
    style SA_CLIENTS fill:#9f9,stroke:#333,stroke-width:2px
```

**Best Practice**: Use `--use-service-account-credentials=true` for per-controller authentication.

### 7.2 TLS & Secure Serving

```mermaid
graph LR
    subgraph "Secure Serving"
        CERT[Self-signed Certificate<br/>OR<br/>Provided Certificate]
        TLS[TLS Server<br/>:10257]
    end

    subgraph "HTTP Handler Chain"
        AUTHN_MW[Authentication<br/>Middleware]
        AUTHZ_MW[Authorization<br/>Middleware]
        HANDLERS[Endpoint Handlers]
    end

    CLIENT[Prometheus/Monitoring] -->|HTTPS| TLS
    TLS --> CERT
    TLS --> AUTHN_MW
    AUTHN_MW --> AUTHZ_MW
    AUTHZ_MW --> HANDLERS
```

---

## 8. Configuration Management

### 8.1 Configuration Sources

```mermaid
graph TB
    subgraph "Configuration Inputs"
        CLI[CLI Flags<br/>--flag=value]
        FILE[Config File<br/>YAML/JSON]
        DEFAULTS[Built-in Defaults]
    end

    subgraph "Configuration Processing"
        OPTS[KubeControllerManagerOptions]
        VALIDATE[Validation]
        COMPLETE[CompletedConfig]
    end

    subgraph "Runtime Configuration"
        COMPONENT[ComponentConfig<br/>All controller settings]
        FEATURE[Feature Gates<br/>Enable/disable features]
        LEADER_CFG[Leader Election Config]
        SERVER_CFG[Server Config]
    end

    CLI --> OPTS
    FILE --> OPTS
    DEFAULTS --> OPTS

    OPTS --> VALIDATE
    VALIDATE --> COMPLETE

    COMPLETE --> COMPONENT
    COMPLETE --> FEATURE
    COMPLETE --> LEADER_CFG
    COMPLETE --> SERVER_CFG
```

### 8.2 Configuration Hierarchy

```
KubeControllerManagerConfiguration
├── Generic (applies to all controllers)
│   ├── LeaderElection
│   ├── ClientConnection
│   ├── Controllers (enable/disable list)
│   ├── MinResyncPeriod
│   └── ControllerStartInterval
├── KubeCloudShared
│   ├── CloudProvider (deprecated)
│   ├── AllocateNodeCIDRs
│   └── ClusterCIDR
├── Per-Controller Configuration
│   ├── DeploymentController
│   │   └── ConcurrentDeploymentSyncs: 5
│   ├── ReplicaSetController
│   │   └── ConcurrentRSSyncs: 5
│   ├── StatefulSetController
│   │   └── ConcurrentStatefulSetSyncs: 5
│   └── ... (30+ more controllers)
└── ServiceController (cloud LB)
```

---

## 9. Failure Domains & Resilience

### 9.1 Failure Isolation

```mermaid
graph TB
    subgraph "Independent Failure Domains"
        C1[Deployment Controller<br/>Goroutines]
        C2[ReplicaSet Controller<br/>Goroutines]
        C3[StatefulSet Controller<br/>Goroutines]

        subgraph "Shared Infrastructure (Single Point of Failure)"
            API_CLIENT[API Client Connection]
            INFORMERS[Shared Informers]
            LEADER[Leader Election]
        end
    end

    C1 -.->|Isolated failure| C1_PANIC[Panic Recovery<br/>Controller stops]
    C2 -.->|Isolated failure| C2_PANIC[Panic Recovery<br/>Controller stops]
    C3 -.->|Isolated failure| C3_PANIC[Panic Recovery<br/>Controller stops]

    API_CLIENT -.->|Connection lost| RECONNECT[Auto-reconnect<br/>Exponential backoff]
    INFORMERS -.->|Cache desync| RESYNC[Periodic resync<br/>Full relist]
    LEADER -.->|Lease lost| EXIT[Exit process<br/>Standby takes over]

    style C1_PANIC fill:#f99,stroke:#333
    style C2_PANIC fill:#f99,stroke:#333
    style C3_PANIC fill:#f99,stroke:#333
    style EXIT fill:#f66,stroke:#333,stroke-width:2px
```

### 9.2 Recovery Patterns

| Failure Type | Detection | Recovery | Impact |
|--------------|-----------|----------|--------|
| Controller panic | utilruntime.HandleCrash() | Log error, stop controller | Single controller affected |
| API connection lost | Watch error | Auto-reconnect + resync | All controllers pause temporarily |
| Informer cache desync | Periodic resync | Full relist | Temporary stale data |
| Leader election lost | Lease renewal failure | Exit process, standby takes over | 15-30s reconciliation gap |
| Work queue overflow | Backpressure | Rate limiting + retry | Slower reconciliation |

---

## 10. Performance Characteristics

### 10.1 Resource Consumption

```
Small Cluster (10 nodes, 100 pods):
  Memory: ~200-500 MB
  CPU: 10-50 millicores idle, 100-500m active
  Goroutines: ~100-150

Medium Cluster (100 nodes, 1000 pods):
  Memory: ~500 MB - 1 GB
  CPU: 50-200 millicores idle, 500-1000m active
  Goroutines: ~150-200

Large Cluster (1000 nodes, 10000 pods):
  Memory: ~2-4 GB
  CPU: 200-500 millicores idle, 1-2 cores active
  Goroutines: ~200-300
```

### 10.2 Scalability Limits

```mermaid
graph TB
    subgraph "Scalability Factors"
        NODES[Number of Nodes<br/>Tested: 5000]
        PODS[Number of Pods<br/>Tested: 150,000]
        CHURN[Change Rate<br/>Updates/sec]
    end

    subgraph "Bottlenecks"
        MEMORY[Informer Cache Memory<br/>~1 KB per object]
        APILOAD[API Server Load<br/>Watch connections]
        CPUSYNC[CPU for Reconciliation<br/>Work queue processing]
    end

    NODES --> MEMORY
    PODS --> MEMORY
    PODS --> APILOAD
    CHURN --> CPUSYNC

    MEMORY -.->|Mitigation| METADATA[Metadata-only informers<br/>Reduce memory 50-80%]
    APILOAD -.->|Mitigation| SHARED[Shared informers<br/>1 watch per type]
    CPUSYNC -.->|Mitigation| PARALLEL[Parallel workers<br/>Configurable concurrency]
```

---

## 11. Evolution & Extensibility

### 11.1 Adding New Controllers

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Impl as Controller Implementation
    participant Desc as Controller Descriptor
    participant Reg as Registration
    participant Build as Build System

    Dev->>Impl: Implement controller<br/>in pkg/controller/mycontroller/
    Impl->>Impl: Define struct, New(), Run()

    Dev->>Desc: Create descriptor function<br/>in cmd/.../app/
    Desc->>Desc: newMyControllerDescriptor()

    Dev->>Reg: Register in NewControllerDescriptors()
    Reg->>Reg: register(newMyControllerDescriptor())

    Dev->>Build: Add to build
    Build->>Build: Compile & test
```

### 11.2 Feature Gates Integration

```mermaid
graph LR
    subgraph "Feature Definition"
        GATE[Feature Gate<br/>MyNewFeature]
        DEFAULT[Default: false]
        ALPHA[Alpha in v1.30]
    end

    subgraph "Controller Descriptor"
        DESC[ControllerDescriptor]
        REQUIRED[requiredFeatureGates:<br/>MyNewFeature]
    end

    subgraph "Runtime Check"
        ENABLED{Feature<br/>Enabled?}
        BUILD[Build Controller]
        SKIP[Skip Controller]
    end

    GATE --> DESC
    DEFAULT --> DESC
    ALPHA --> DESC
    DESC --> REQUIRED
    REQUIRED --> ENABLED
    ENABLED -->|Yes| BUILD
    ENABLED -->|No| SKIP
```

---

## 12. Key Takeaways

1. **Monolithic Process**: All 50 controllers run in single process for efficiency
2. **Shared Infrastructure**: Informers, caches, and clients shared across controllers
3. **Event-Driven**: Watch-based architecture with local caching
4. **Leader Election**: Active-passive HA for multi-instance deployments
5. **Isolation**: Controller failures isolated via goroutine panic recovery
6. **Scalability**: Proven to 5000 nodes, 150k pods
7. **Observability**: Comprehensive metrics, health checks, and debugging
8. **Security**: Per-controller RBAC with service account credentials
9. **Extensibility**: Clean descriptor pattern for adding controllers
10. **Resilience**: Automatic reconnection and recovery mechanisms

---

## Related Documentation

- **Executive Summary**: `02-executive-summary.md` - High-level overview
- **Functional Overview**: `03-functional-overview.md` - Detailed capabilities
- **Initialization Flow**: `06-initialization-lifecycle.md` - Startup sequence
- **Shared Infrastructure**: `07-shared-infrastructure.md` - Informers, queues
- **Data Structures**: `20-data-structures.md` - Key structs and interfaces

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-10-21 | Architecture Analysis | Initial high-level architecture |
