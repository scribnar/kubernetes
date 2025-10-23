# Kube-APIServer Functional Specification

> **Functional Design and Behavior of the Kubernetes API Server**

---

## Table of Contents

- [Executive Summary](#executive-summary)
- [System Overview](#system-overview)
- [Functional Architecture](#functional-architecture)
- [Component Specifications](#component-specifications)
- [Data Flow Specifications](#data-flow-specifications)
- [Interface Specifications](#interface-specifications)
- [Behavior Specifications](#behavior-specifications)

---

## Executive Summary

The Kubernetes API Server (kube-apiserver) implements a **layered, plugin-based architecture** that processes API requests through a **24-layer handler chain**, validates and mutates them through **admission control**, and persists them to **etcd** while providing **real-time watch streams** to clients.

### Key Design Principles

1. **Separation of Concerns**: Authentication, authorization, admission, and storage are separate, composable layers
2. **Plugin Architecture**: Extensible through plugins (auth, authz, admission, storage)
3. **Delegation Pattern**: Three-server chain (Aggregator → Kube → Extensions) for modular API serving
4. **Optimistic Concurrency**: Resource versioning prevents conflicting updates
5. **Watch-Based Architecture**: Efficient real-time notifications via watch cache
6. **Type Safety**: Strong typing with automatic conversion between API versions

---

## System Overview

### Architectural Layers

```mermaid
graph TB
    subgraph "Layer 1: Transport"
        TLS[TLS Termination]
        HTTP[HTTP/2 Server]
    end

    subgraph "Layer 2: Server Chain"
        Aggregator[Aggregator Server]
        KubeAPI[Kube API Server]
        Extensions[API Extensions Server]
    end

    subgraph "Layer 3: Handler Chain"
        Auth[Authentication]
        Authz[Authorization]
        APF[Priority & Fairness]
        Admission[Admission Control]
    end

    subgraph "Layer 4: Business Logic"
        Registry[Generic Registry]
        Strategy[Resource Strategies]
        Validation[Validation]
    end

    subgraph "Layer 5: Storage"
        Cache[Watch Cache]
        Storage[Storage Layer]
        Transform[Transformers]
    end

    subgraph "Layer 6: Persistence"
        etcd[(etcd)]
    end

    TLS --> HTTP
    HTTP --> Aggregator
    Aggregator --> KubeAPI
    KubeAPI --> Extensions
    Extensions --> Auth
    Auth --> Authz
    Authz --> APF
    APF --> Admission
    Admission --> Registry
    Registry --> Strategy
    Strategy --> Validation
    Validation --> Cache
    Cache --> Storage
    Storage --> Transform
    Transform --> etcd

    style Aggregator fill:#ffcccc
    style KubeAPI fill:#ccffcc
    style Extensions fill:#ccccff
    style etcd fill:#ffffcc
```

### Core Subsystems

| Subsystem | Responsibility | Implementation |
|-----------|----------------|----------------|
| **Server Chain** | Request routing | Delegation pattern across 3 servers |
| **Handler Chain** | Request processing | 24-layer filter pipeline |
| **Authentication** | Identity verification | Pluggable authenticators (certs, tokens, OIDC) |
| **Authorization** | Permission checking | Pluggable authorizers (RBAC, Node, Webhook) |
| **Admission** | Request validation/mutation | Built-in plugins + webhooks |
| **Registry** | CRUD operations | Generic registry + resource strategies |
| **Storage** | Persistence abstraction | etcd3 + watch cache |
| **Watch** | Real-time notifications | Watch cache + bookmark events |
| **Discovery** | API enumeration | OpenAPI schema generation |
| **Aggregation** | API extension | Proxy to extension API servers |

---

## Functional Architecture

### Server Chain Architecture

The kube-apiserver uses a **delegation pattern** with three layered servers:

```mermaid
graph LR
    Client[Client] --> Aggregator[Aggregator Server<br/>APIService routing]
    Aggregator -->|delegates| KubeAPI[Kube API Server<br/>Built-in APIs]
    KubeAPI -->|delegates| Extensions[API Extensions Server<br/>CustomResourceDefinitions]
    Extensions -->|delegates| NotFound[404 Handler]

    Aggregator -.->|proxies| ExtAPI1[Extension API 1]
    Aggregator -.->|proxies| ExtAPI2[Extension API 2]

    style Aggregator fill:#ff9999
    style KubeAPI fill:#99ff99
    style Extensions fill:#9999ff
```

**Delegation Flow:**
1. **Aggregator Server** handles requests first
   - If matches an APIService → proxies to extension API server
   - Otherwise → delegates to Kube API Server

2. **Kube API Server** handles built-in APIs
   - If matches built-in API group → serves from registry
   - Otherwise → delegates to API Extensions Server

3. **API Extensions Server** handles CRDs
   - If matches a CRD → serves custom resource
   - Otherwise → delegates to 404 handler

**Implementation:**
- `cmd/kube-apiserver/app/server.go:176-197` - CreateServerChain()
- Each server wraps the next as `DelegationTarget`

---

## Component Specifications

### 1. Handler Chain Specification

**Purpose**: Process every API request through a series of filters.

**Architecture**: Chain of Responsibility pattern with 24 filters.

```mermaid
graph TB
    Request[Incoming Request] --> F1[1. Audit Init]
    F1 --> F2[2. Panic Recovery]
    F2 --> F3[3. Request Info]
    F3 --> F4[4. Wait Group]
    F4 --> F5[5. Timeout]
    F5 --> F6[6. CORS]
    F6 --> F7[7. Authentication]
    F7 --> F8[8. Authorization]
    F8 --> F9[9. Audit]
    F9 --> F10[10. Admission]
    F10 --> F11[11. Priority & Fairness]
    F11 --> Handler[API Handler]
    Handler --> Storage[(Storage)]

    style F7 fill:#ffcccc
    style F8 fill:#ffcccc
    style F9 fill:#ccffcc
    style F10 fill:#ccffcc
    style F11 fill:#ccccff
```

**Filter Responsibilities:**

| Order | Filter | Purpose | Can Short-Circuit |
|-------|--------|---------|-------------------|
| 1 | Audit Init | Initialize audit context | No |
| 2 | Panic Recovery | Catch panics, return 500 | No |
| 3 | Request Info | Extract verb, resource, user | No |
| 4 | Mux Discovery Complete | Ensure routes registered | Yes |
| 5 | Request Timestamp | Track request arrival time | No |
| 6 | HTTP Logging | Log all HTTP requests | No |
| 7 | HSTS Header | Add security headers | No |
| 8 | Wait Group | Track in-flight requests | No |
| 9 | Request Deadline | Apply request timeout | No |
| 10 | Warning Recorder | Record deprecation warnings | No |
| 11 | CORS | Handle CORS preflight | Yes |
| 12 | **Authentication** | Verify identity | **Yes** |
| 13 | Impersonation | Support user impersonation | No |
| 14 | **Authorization** | Check permissions | **Yes** |
| 15 | Audit | Log operation | No |
| 16 | **Priority & Fairness** | Rate limiting | **Yes** |
| 17 | **Admission** | Validate/mutate request | **Yes** |
| ... | ... | ... | ... |

**Implementation:**
- `staging/src/k8s.io/apiserver/pkg/server/config.go:1014-1091` - DefaultBuildHandlerChain()

---

### 2. Authentication Specification

**Purpose**: Verify the identity of the requester.

**Design**: Chain of authenticators, first success wins.

```mermaid
graph TB
    Request[HTTP Request] --> Chain{Authenticator Chain}

    Chain --> Cert[X.509 Client Cert]
    Chain --> Token[Bearer Token]
    Chain --> Basic[HTTP Basic]
    Chain --> Anon[Anonymous]

    Cert --> CertOK{Valid?}
    Token --> TokenType{Type?}
    TokenType --> SA[Service Account]
    TokenType --> OIDC[OIDC]
    TokenType --> Webhook[Webhook]

    CertOK -->|Yes| User[user.Info]
    SA -->|Valid| User
    OIDC -->|Valid| User
    Webhook -->|Valid| User

    CertOK -->|No| Chain
    SA -->|Invalid| Chain
    OIDC -->|Invalid| Chain
    Webhook -->|Invalid| Chain

    Anon --> AnonEnabled{Enabled?}
    AnonEnabled -->|Yes| User
    AnonEnabled -->|No| Fail[401 Unauthorized]

    User --> Next[Next Filter]

    style Cert fill:#ffcccc
    style Token fill:#ccffcc
    style User fill:#ccccff
```

**Authenticator Types:**

| Authenticator | Input | Output | Use Case |
|---------------|-------|--------|----------|
| **X.509 Client Cert** | TLS client certificate | CN=username, O=groups | kubectl, kubelets |
| **Service Account** | Bearer token (JWT) | sa name, namespace | Pods, controllers |
| **OIDC** | Bearer token (OIDC) | email, groups from IdP | Human users via SSO |
| **Webhook** | Bearer token | POST to webhook | Custom auth systems |
| **Bootstrap Token** | Bearer token | For node join | New node registration |
| **Anonymous** | No credentials | system:anonymous | Public endpoints |

**User Information Structure:**
```go
type Info interface {
    GetName() string          // Username
    GetUID() string           // Unique identifier
    GetGroups() []string      // Group memberships
    GetExtra() map[string][]string  // Additional attributes
}
```

**Implementation:**
- `staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go`
- `staging/src/k8s.io/apiserver/pkg/endpoints/filters/authentication.go`

---

### 3. Authorization Specification

**Purpose**: Determine if the authenticated user has permission for the requested operation.

**Design**: Chain of authorizers, first allow wins.

```mermaid
graph TB
    User[Authenticated User] --> Attrs[Build Attributes]
    Attrs --> Chain{Authorizer Chain}

    Chain --> RBAC[RBAC Authorizer]
    Chain --> Node[Node Authorizer]
    Chain --> Webhook[Webhook Authorizer]
    Chain --> ABAC[ABAC Authorizer]

    RBAC --> RBACDecision{Decision}
    Node --> NodeDecision{Decision}
    Webhook --> WebhookDecision{Decision}
    ABAC --> ABACDecision{Decision}

    RBACDecision -->|Allow| Allowed[Request Allowed]
    RBACDecision -->|NoOpinion| Chain
    RBACDecision -->|Deny| Denied

    NodeDecision -->|Allow| Allowed
    NodeDecision -->|NoOpinion| Chain
    NodeDecision -->|Deny| Denied

    WebhookDecision -->|Allow| Allowed
    WebhookDecision -->|NoOpinion| Chain
    WebhookDecision -->|Deny| Denied

    ABACDecision -->|Allow| Allowed
    ABACDecision -->|Deny| Denied
    ABACDecision -->|NoOpinion| Denied[Request Denied<br/>403 Forbidden]

    style Allowed fill:#ccffcc
    style Denied fill:#ffcccc
```

**Authorization Attributes:**
```go
type Attributes interface {
    GetUser() user.Info        // Who
    GetVerb() string           // What operation (get, list, create, etc.)
    GetNamespace() string      // Where (namespace or cluster-scoped)
    GetResource() string       // Which resource type
    GetSubresource() string    // Which subresource (status, scale, etc.)
    GetName() string           // Which specific resource instance
    GetAPIGroup() string       // Which API group
    GetAPIVersion() string     // Which API version
    GetResourceRequest() bool  // Is this a resource request or non-resource?
    GetPath() string           // For non-resource requests (/healthz, /metrics)
}
```

**Authorization Modes:**

| Mode | Logic | Configuration | Use Case |
|------|-------|---------------|----------|
| **RBAC** | Role bindings grant permissions | Roles, RoleBindings, ClusterRoles, ClusterRoleBindings | General purpose, recommended |
| **Node** | Kubelets can only access own resources | Automatic for kubelets | Kubelet security |
| **Webhook** | External authorization service | Webhook endpoint URL | Custom policy engines |
| **ABAC** | Policy file with JSON rules | Static policy file | Legacy, deprecated |

**RBAC Model:**
```mermaid
graph LR
    User[User/ServiceAccount] --> Binding[RoleBinding]
    Binding --> Role[Role]
    Role --> Rules[Rules]
    Rules --> Verbs[Verbs:<br/>get, list, create,<br/>update, delete, watch]
    Rules --> Resources[Resources:<br/>pods, services,<br/>deployments, etc.]
    Rules --> APIGroups[API Groups:<br/>core, apps, batch, etc.]

    style User fill:#ccffff
    style Role fill:#ffcccc
    style Rules fill:#ccffcc
```

**Implementation:**
- `staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go`
- `staging/src/k8s.io/apiserver/pkg/endpoints/filters/authorization.go`
- `pkg/registry/rbac/` - RBAC storage

---

### 4. Admission Control Specification

**Purpose**: Validate and mutate API requests after authentication and authorization.

**Design**: Sequential execution of admission plugins, mutating before validating.

```mermaid
graph TB
    Request[Authorized Request] --> Mutating[Mutating Admission]

    Mutating --> Plugin1[Plugin 1: NamespaceLifecycle]
    Plugin1 --> Plugin2[Plugin 2: LimitRanger]
    Plugin2 --> Plugin3[Plugin 3: ServiceAccount]
    Plugin3 --> Plugin4[Plugin 4: DefaultStorageClass]
    Plugin4 --> MutatingWebhooks[Mutating Webhooks]

    MutatingWebhooks --> Reinvoke{Need<br/>Reinvoke?}
    Reinvoke -->|Yes| Validating
    Reinvoke -->|No| Validating[Validating Admission]

    Validating --> Plugin5[Plugin 5: PodSecurity]
    Plugin5 --> Plugin6[Plugin 6: ResourceQuota]
    Plugin6 --> ValidatingWebhooks[Validating Webhooks]
    ValidatingWebhooks --> ValidatingPolicy[Validating Admission Policy<br/>CEL-based]

    ValidatingPolicy --> Decision{All<br/>Passed?}
    Decision -->|Yes| Handler[API Handler]
    Decision -->|No| Reject[Reject Request<br/>400/403]

    style Mutating fill:#ffcccc
    style Validating fill:#ccffcc
    style Handler fill:#ccccff
    style Reject fill:#ff9999
```

**Admission Plugin Types:**

| Type | Can Modify? | Can Reject? | Example |
|------|-------------|-------------|---------|
| **Mutating** | ✅ Yes | ✅ Yes | DefaultStorageClass sets default SC |
| **Validating** | ❌ No | ✅ Yes | ResourceQuota enforces limits |
| **Both** | ✅ Yes | ✅ Yes | NamespaceLifecycle mutates+validates |

**Key Built-in Admission Plugins:**

| Plugin | Type | Purpose |
|--------|------|---------|
| NamespaceLifecycle | Both | Prevents operations in terminating namespaces |
| LimitRanger | Validating | Enforces LimitRange constraints |
| ServiceAccount | Mutating | Injects service account tokens |
| DefaultStorageClass | Mutating | Sets default storage class for PVCs |
| ResourceQuota | Validating | Enforces resource quotas |
| PodSecurity | Validating | Enforces Pod Security Standards |
| PersistentVolumeLabel | Mutating | Adds labels to PVs (cloud provider) |

**Webhook Admission:**

```mermaid
graph LR
    API[API Server] -->|HTTPS POST| Webhook[Admission Webhook]
    Webhook -->|AdmissionReview Response| API

    subgraph "AdmissionReview Request"
        Request[Object:<br/>Pod spec]
        User[User Info]
        Operation[Operation:<br/>CREATE]
    end

    subgraph "AdmissionReview Response"
        Allowed[Allowed: true/false]
        Patches[JSON Patches]
        Reason[Denial Reason]
    end

    style Webhook fill:#ccffcc
```

**Webhook Configuration:**
- **MutatingWebhookConfiguration**: Defines mutating webhooks
- **ValidatingWebhookConfiguration**: Defines validating webhooks
- **Failure Policy**: Fail (reject on error) or Ignore (allow on error)
- **Timeout**: Max 30 seconds
- **Reinvocation Policy**: Never, IfNeeded

**Implementation:**
- `staging/src/k8s.io/apiserver/pkg/admission/`
- `pkg/kubeapiserver/admission/` - Kubernetes-specific plugins

---

### 5. Storage Layer Specification

**Purpose**: Persist API resources to etcd with caching for efficiency.

**Architecture**: Layered storage with transformers and caching.

```mermaid
graph TB
    Handler[API Handler] --> Registry[Generic Registry]
    Registry --> Strategy[Resource Strategy]
    Strategy --> Store[Generic Store]

    Store --> Decorator{Has<br/>Decorator?}
    Decorator -->|Yes| Cacher[Watch Cacher]
    Decorator -->|No| Storage[Storage Interface]

    Cacher --> CacheHit{Cache<br/>Hit?}
    CacheHit -->|Yes| CacheReturn[Return from Cache]
    CacheHit -->|No| Storage

    Storage --> Transform[Storage Transformer]
    Transform --> Encrypt[Encryption]
    Encrypt --> Codec[Codec<br/>Protobuf/JSON]
    Codec --> etcd[(etcd)]

    etcd --> Watch[Watch Events]
    Watch --> Cacher
    Cacher --> Clients[Watch Clients]

    style Registry fill:#ffcccc
    style Cacher fill:#ccffcc
    style etcd fill:#ffffcc
```

**Storage Layers:**

| Layer | Responsibility | Implementation |
|-------|----------------|----------------|
| **Registry** | CRUD operations | `pkg/registry/generic/registry/store.go` |
| **Strategy** | Resource-specific logic | `pkg/registry/{group}/{resource}/strategy.go` |
| **Store** | Generic storage operations | `genericregistry.Store` |
| **Cacher** | Watch cache | `staging/src/k8s.io/apiserver/pkg/storage/cacher/` |
| **Storage Interface** | etcd abstraction | `staging/src/k8s.io/apiserver/pkg/storage/interfaces.go` |
| **Transformer** | Encryption, compression | `staging/src/k8s.io/apiserver/pkg/storage/value/` |
| **etcd3** | Persistence | `staging/src/k8s.io/apiserver/pkg/storage/etcd3/` |

**Storage Operations:**

```go
type Interface interface {
    // Create inserts a new object
    Create(ctx, key string, obj, out runtime.Object, ttl uint64) error

    // Get retrieves an object
    Get(ctx, key string, opts storage.GetOptions, out runtime.Object) error

    // List retrieves objects matching the criteria
    List(ctx, key string, opts storage.ListOptions, listObj runtime.Object) error

    // GuaranteedUpdate atomically updates an object
    GuaranteedUpdate(ctx, key string, out runtime.Object, ignoreNotFound bool,
        preconditions *Preconditions, updater UpdateFunc) error

    // Delete removes an object
    Delete(ctx, key string, out runtime.Object, preconditions *Preconditions,
        validateDeletion ValidateObjectFunc, out runtime.Object) error

    // Watch streams changes
    Watch(ctx, key string, opts storage.ListOptions) (watch.Interface, error)
}
```

**Resource Versioning:**
- **resourceVersion**: Opaque string representing object version
- **Mapped to**: etcd mod_revision (monotonically increasing)
- **Used for**: Optimistic concurrency control, watch resumption
- **Preconditions**: UID and resourceVersion checked before update

**Implementation:**
- `staging/src/k8s.io/apiserver/pkg/storage/interfaces.go`
- `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go`
- `pkg/registry/generic/registry/store.go`

---

### 6. Watch Mechanism Specification

**Purpose**: Provide real-time notifications of resource changes.

**Design**: Watch cache serves most watches, falls through to etcd on miss.

```mermaid
sequenceDiagram
    participant Client
    participant APIServer
    participant Cacher
    participant etcd

    Note over Cacher: Initial State Sync
    Cacher->>etcd: LIST from rv=0
    etcd->>Cacher: All objects + rv=1000
    Cacher->>etcd: WATCH from rv=1000

    Client->>APIServer: WATCH pods?resourceVersion=900
    APIServer->>Cacher: Can serve from cache?

    alt Cache has history
        Cacher->>Client: Event stream from rv=900
        loop Ongoing events
            etcd->>Cacher: Event (rv=1001)
            Cacher->>Client: Event (rv=1001)
        end
    else Cache miss
        APIServer->>etcd: WATCH from rv=900
        etcd->>Client: Event stream (fall-through)
    end

    Note over Cacher: Bookmark Events
    loop Every 1 minute
        Cacher->>Client: Bookmark (rv=1050)
    end
```

**Watch Cache Architecture:**

**Components:**
1. **Reflector**: Syncs etcd state to in-memory cache
2. **Store**: In-memory storage of objects
3. **Event Buffer**: Ring buffer of recent events
4. **Bookmark Timer**: Periodic bookmark event generator

**Cache Properties:**
- **Buffer Size**: Last N events (configurable, default based on resource type)
- **Bookmark Frequency**: Every 1 minute
- **Event Window**: 75 seconds (bookmark frequency + 15s)
- **Fall-through**: Serves from etcd if resourceVersion too old

**Watch Event Types:**
```go
type EventType string
const (
    Added    EventType = "ADDED"
    Modified EventType = "MODIFIED"
    Deleted  EventType = "DELETED"
    Bookmark EventType = "BOOKMARK"  // Keeps watch current
    Error    EventType = "ERROR"
)
```

**Bookmark Events:**
- **Purpose**: Update client's resourceVersion without actual changes
- **Frequency**: Every 1 minute
- **Benefit**: Prevents expensive relist when watch restarts

**Implementation:**
- `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go`
- `staging/src/k8s.io/client-go/tools/cache/reflector.go`

---

### 7. API Priority and Fairness Specification

**Purpose**: Prevent API server overload through fair queuing and prioritization.

**Design**: Classify requests into FlowSchemas, apply PriorityLevels.

```mermaid
graph TB
    Request[Incoming Request] --> Classify[Classify Request]

    Classify --> FS1{FlowSchema 1<br/>system-leader-election}
    Classify --> FS2{FlowSchema 2<br/>workload-high}
    Classify --> FS3{FlowSchema 3<br/>workload-low}
    Classify --> FS4{FlowSchema 4<br/>catch-all}

    FS1 --> PL1[PriorityLevel: system-leader<br/>Concurrency: 100]
    FS2 --> PL2[PriorityLevel: workload-high<br/>Concurrency: 40]
    FS3 --> PL3[PriorityLevel: workload-low<br/>Concurrency: 20]
    FS4 --> PL4[PriorityLevel: catch-all<br/>Concurrency: 10]

    PL1 --> Seats1{Seats<br/>Available?}
    PL2 --> Seats2{Seats<br/>Available?}
    PL3 --> Seats3{Seats<br/>Available?}
    PL4 --> Seats4{Seats<br/>Available?}

    Seats1 -->|Yes| Execute1[Execute]
    Seats1 -->|No| Queue1[Queue]

    Seats2 -->|Yes| Execute2[Execute]
    Seats2 -->|No| Queue2[Queue]

    style Request fill:#ccffff
    style PL1 fill:#ffcccc
    style PL2 fill:#ffddcc
    style PL3 fill:#ffeecc
    style PL4 fill:#ffffcc
    style Execute1 fill:#ccffcc
```

**FlowSchema**: Matches requests based on:
- User/ServiceAccount
- Namespace
- API group/resource
- Verb

**PriorityLevel**: Defines:
- **Concurrency**: Max simultaneous requests
- **Queues**: Number of queues (for fairness)
- **Queue Length**: Max queued requests per queue
- **Hand Size**: Requests dispatched together

**Work Estimation:**
- **List requests**: Estimated based on result size
- **Watch requests**: Fixed cost
- **Mutations**: Based on object size
- **Reads**: Minimal cost

**Default Priority Levels:**

| Priority Level | Concurrency | Use Case |
|----------------|-------------|----------|
| system | 30 | System components |
| leader-election | 10 | Leader election |
| workload-high | 40 | Important workloads |
| workload-low | 100 | Regular workloads |
| global-default | 20 | Catch-all |
| exempt | Unlimited | Health checks, debugging |

**Implementation:**
- `staging/src/k8s.io/apiserver/pkg/util/flowcontrol/`
- `pkg/apis/flowcontrol/` - API types

---

## Data Flow Specifications

### Request Flow: CREATE Operation

```mermaid
sequenceDiagram
    participant Client
    participant Handler Chain
    participant Admission
    participant Validation
    participant Storage
    participant etcd

    Client->>Handler Chain: POST /api/v1/namespaces/default/pods
    Handler Chain->>Handler Chain: 1. Authentication
    Handler Chain->>Handler Chain: 2. Authorization
    Handler Chain->>Handler Chain: 3. Priority & Fairness

    Handler Chain->>Admission: 4. Mutating Admission
    Admission->>Admission: - Set defaults
    Admission->>Admission: - Inject SA token
    Admission->>Admission: - Call mutating webhooks

    Admission->>Validation: 5. Validation
    Validation->>Validation: - Schema validation
    Validation->>Validation: - Business logic validation

    Validation->>Admission: 6. Validating Admission
    Admission->>Admission: - Check ResourceQuota
    Admission->>Admission: - Call validating webhooks

    Admission->>Storage: 7. Storage.Create()
    Storage->>Storage: - Generate UID
    Storage->>Storage: - Set creationTimestamp
    Storage->>Storage: - Set resourceVersion=0

    Storage->>etcd: 8. etcd Create
    etcd->>Storage: rv=123, created
    Storage->>Client: 9. Return created object

    Note over etcd: Watch events generated
    etcd->>Storage: Watch event: ADDED
    Storage->>Watchers: Broadcast to watch clients
```

### Request Flow: UPDATE Operation

```mermaid
sequenceDiagram
    participant Client
    participant Storage
    participant etcd

    Client->>Storage: PUT /api/v1/namespaces/default/pods/mypod
    Note over Client: Includes resourceVersion=123

    Storage->>Storage: Prepare preconditions
    Note over Storage: Preconditions:<br/>- UID matches<br/>- resourceVersion=123

    Storage->>etcd: GuaranteedUpdate
    Note over etcd: Atomic read-modify-write

    etcd->>etcd: Read current object
    etcd->>etcd: Check preconditions

    alt Preconditions pass
        etcd->>etcd: Apply update
        etcd->>etcd: Increment mod_revision
        etcd->>Storage: Success, rv=124
        Storage->>Client: Updated object (rv=124)
    else Conflict
        etcd->>Storage: Conflict error
        Storage->>Client: 409 Conflict
        Note over Client: Client must retry<br/>with updated rv
    end
```

### Watch Flow

```mermaid
sequenceDiagram
    participant Client
    participant Cacher
    participant etcd

    Client->>Cacher: WATCH pods?resourceVersion=100

    Cacher->>Cacher: Check if rv=100 in event buffer

    alt In buffer
        Cacher->>Client: Stream events from rv=100
        Note over Cacher,Client: Events: 101, 102, 103...
    else Too old
        Cacher->>Client: 410 Gone - too old
        Note over Client: Client must relist
    end

    loop Ongoing
        etcd->>Cacher: New event (rv=125)
        Cacher->>Client: MODIFIED pod/mypod (rv=125)
    end

    loop Every 60 seconds
        Cacher->>Client: BOOKMARK (rv=130)
        Note over Client: Update last-seen rv
    end
```

---

## Interface Specifications

### RESTful API Endpoints

**Core API (/api/v1):**
```
GET    /api/v1/pods                          # List all pods (all namespaces)
GET    /api/v1/namespaces/{ns}/pods          # List pods in namespace
GET    /api/v1/namespaces/{ns}/pods/{name}   # Get specific pod
POST   /api/v1/namespaces/{ns}/pods          # Create pod
PUT    /api/v1/namespaces/{ns}/pods/{name}   # Replace pod
PATCH  /api/v1/namespaces/{ns}/pods/{name}   # Patch pod
DELETE /api/v1/namespaces/{ns}/pods/{name}   # Delete pod
```

**Named API Groups (/apis/{group}/{version}):**
```
GET    /apis/apps/v1/deployments
GET    /apis/apps/v1/namespaces/{ns}/deployments
GET    /apis/apps/v1/namespaces/{ns}/deployments/{name}
...
```

**Subresources:**
```
GET    /api/v1/namespaces/{ns}/pods/{name}/status     # Get pod status
PUT    /api/v1/namespaces/{ns}/pods/{name}/status     # Update pod status
GET    /api/v1/namespaces/{ns}/pods/{name}/log        # Get pod logs
GET    /api/v1/namespaces/{ns}/pods/{name}/exec       # Exec into pod
POST   /api/v1/namespaces/{ns}/pods/{name}/eviction   # Evict pod
```

**Discovery:**
```
GET    /api                    # Core API discovery
GET    /apis                   # Named API groups discovery
GET    /apis/{group}/{version} # Resource discovery
GET    /openapi/v2             # OpenAPI v2 spec
GET    /openapi/v3             # OpenAPI v3 spec
```

**Health & Debug:**
```
GET    /healthz                # Liveness check
GET    /readyz                 # Readiness check
GET    /livez                  # Component liveness
GET    /metrics                # Prometheus metrics
GET    /debug/pprof/*          # Profiling (if enabled)
```

### Query Parameters

**Common Parameters:**
- `?labelSelector=app=nginx` - Filter by labels
- `?fieldSelector=status.phase=Running` - Filter by fields
- `?resourceVersion=123` - Watch from specific version
- `?watch=true` - Enable watch mode
- `?limit=100&continue=token` - Pagination
- `?dryRun=All` - Dry run mode
- `?fieldManager=kubectl` - Server-side apply field manager

---

## Behavior Specifications

### Optimistic Concurrency Control

```
1. Client GETs resource (receives resourceVersion=123)
2. Client modifies resource locally
3. Client PUTs resource with resourceVersion=123
4. Server checks: current resourceVersion == 123?
   - YES: Update succeeds, new resourceVersion=124
   - NO: 409 Conflict, client must retry
```

### Graceful Deletion

```
1. DELETE request with gracePeriodSeconds=30
2. Object's deletionTimestamp set to now+30s
3. Object remains visible with deletionTimestamp
4. Finalizers run to cleanup dependencies
5. When finalizers complete, object removed from etcd
6. Clients watching see DELETE event
```

### Namespace Deletion

```
1. Namespace marked for deletion (deletionTimestamp set)
2. Namespace controller deletes all objects in namespace
3. When all objects deleted, namespace finalizers run
4. When finalizers complete, namespace deleted
5. New objects cannot be created in deleting namespace
```

---

## Summary

The kube-apiserver implements a sophisticated, layered architecture:

1. **Server Chain**: 3-layer delegation for modular API serving
2. **Handler Chain**: 24-layer filter pipeline for request processing
3. **Authentication**: Multi-strategy identity verification
4. **Authorization**: Pluggable permission checking (RBAC, etc.)
5. **Admission**: Extensible validation and mutation
6. **Storage**: Efficient etcd integration with watch caching
7. **Watch**: Real-time event streaming with bookmark optimization
8. **APF**: Fair queuing and prioritization for stability
9. **Type System**: Strong typing with automatic conversion
10. **Extensibility**: CRDs, aggregation, webhooks

This design achieves **high throughput**, **low latency**, **strong consistency**, **real-time notifications**, and **extensibility** while maintaining **security** and **reliability**.

---

**Next**: Dive into detailed component architecture in:
- [High-Level Architecture](high-level/)
- [Middle-Level Architecture](middle-level/)
- [Low-Level Technical Specs](low-level/)
