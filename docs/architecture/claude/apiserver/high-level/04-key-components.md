# Key Components

> **High-Level Architecture: Major subsystems and their interactions in kube-apiserver**

---

## Table of Contents

- [Overview](#overview)
- [Core Components](#core-components)
- [Component Interactions](#component-interactions)
- [Data Flow Through Components](#data-flow-through-components)
- [Component Dependencies](#component-dependencies)

---

## Overview

The kube-apiserver is composed of **modular, loosely-coupled components** that work together to provide a complete API serving solution.

### Component Architecture

```mermaid
graph TB
    subgraph "Transport Layer"
        TLS[TLS Server]
        HTTP[HTTP/2 Handler]
    end

    subgraph "Request Processing"
        HandlerChain[Handler Chain<br/>24 filters]
        Auth[Authentication]
        Authz[Authorization]
        Admission[Admission Control]
    end

    subgraph "Business Logic"
        Registry[Generic Registry]
        Strategy[Resource Strategies]
        Validation[Validation]
    end

    subgraph "Storage"
        Cacher[Watch Cache]
        Storage[Storage Interface]
        Transform[Transformers]
    end

    subgraph "Infrastructure"
        Discovery[API Discovery]
        OpenAPI[OpenAPI]
        Audit[Audit Logging]
        Metrics[Metrics]
    end

    TLS --> HTTP
    HTTP --> HandlerChain
    HandlerChain --> Auth
    HandlerChain --> Authz
    HandlerChain --> Admission
    Admission --> Registry
    Registry --> Strategy
    Strategy --> Validation
    Validation --> Cacher
    Cacher --> Storage
    Storage --> Transform
    Transform --> etcd[(etcd)]

    HandlerChain -.-> Discovery
    HandlerChain -.-> OpenAPI
    HandlerChain -.-> Audit
    HandlerChain -.-> Metrics

    style HandlerChain fill:#ffcccc
    style Registry fill:#ccffcc
    style Cacher fill:#ccccff
    style etcd fill:#ffffcc
```

---

## Core Components

### 1. GenericAPIServer

**Purpose**: Base server implementation providing common functionality.

```mermaid
classDiagram
    class GenericAPIServer {
        +Handler APIServerHandler
        +delegationTarget DelegationTarget
        +admissionControl admission.Interface
        +Authorizer authorizer.Authorizer
        +Serializer runtime.NegotiatedSerializer
        +StorageFactory StorageFactory
        +AuditBackend audit.Backend
        +LoopbackClientConfig RestConfig
        +InstallAPIGroup(APIGroupInfo) error
        +PrepareRun() PreparedGenericAPIServer
        +Run(context) error
        +AddPostStartHook(name, hook)
        +AddPreShutdownHook(name, hook)
    }

    class APIServerHandler {
        +FullHandlerChain http.Handler
        +GoRestfulContainer Container
        +NonGoRestfulMux PathRecorderMux
        +Director http.Handler
    }

    class DelegationTarget {
        <<interface>>
        +UnprotectedHandler() Handler
        +PostStartHooks() map
        +PreShutdownHooks() map
    }

    GenericAPIServer *-- APIServerHandler
    GenericAPIServer ..|> DelegationTarget
```

**Responsibilities:**
- HTTP server management
- Handler chain construction
- API group installation
- Lifecycle hook management
- Health check endpoints

**File**: `staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go` (1,086 lines)

---

### 2. Handler Chain

**Purpose**: Process every request through a pipeline of filters.

```mermaid
graph LR
    Request[Request] --> F1[Panic<br/>Recovery]
    F1 --> F2[Request<br/>Info]
    F2 --> F3[Authentication]
    F3 --> F4[Authorization]
    F4 --> F5[Priority &<br/>Fairness]
    F5 --> F6[Admission<br/>Control]
    F6 --> Handler[API<br/>Handler]

    style F3 fill:#ffcccc
    style F4 fill:#ffcccc
    style F5 fill:#ccccff
    style F6 fill:#ccffcc
```

**24 Filters Total** (abbreviated above):

| Filter | Layer | Purpose |
|--------|-------|---------|
| Panic Recovery | Infrastructure | Catch panics, return 500 |
| Request Info | Routing | Extract verb, resource, namespace |
| **Authentication** | **Security** | **Verify identity** |
| Impersonation | Security | Support user impersonation |
| **Authorization** | **Security** | **Check permissions** |
| Audit | Observability | Log operations |
| **Priority & Fairness** | **Stability** | **Rate limiting** |
| **Admission** | **Validation** | **Validate/mutate** |
| Timeout | Infrastructure | Request deadline |
| CORS | HTTP | CORS headers |

**Builder**: `staging/src/k8s.io/apiserver/pkg/server/config.go:1014-1091`

---

### 3. Authentication System

**Purpose**: Verify requester identity.

```mermaid
graph TB
    Request[Request] --> Chain{Authenticator<br/>Chain}

    Chain --> Cert[X.509<br/>Authenticator]
    Chain --> Token[Token<br/>Authenticator]
    Chain --> Anon[Anonymous<br/>Authenticator]

    Cert -->|CN + O| User1[User Info]
    Token --> SA[Service Account<br/>Token]
    Token --> OIDC[OIDC<br/>Token]
    Token --> Webhook[Webhook<br/>Token]

    SA --> User2[User Info]
    OIDC --> User3[User Info]
    Webhook --> User4[User Info]
    Anon --> User5[system:anonymous]

    User1 --> Next[Next Filter]
    User2 --> Next
    User3 --> Next
    User4 --> Next
    User5 --> Next

    style Chain fill:#ffcccc
    style Next fill:#ccffcc
```

**Components:**

| Component | Purpose | Implementation |
|-----------|---------|----------------|
| **Authenticator Interface** | Base interface | `staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go` |
| **Request Authenticator** | HTTP request auth | `staging/src/k8s.io/apiserver/pkg/authentication/request/` |
| **Token Authenticators** | Bearer token auth | `staging/src/k8s.io/apiserver/pkg/authentication/token/` |
| **Authenticator Factory** | Build chain | `staging/src/k8s.io/apiserver/pkg/authentication/authenticatorfactory/` |

**User Info Structure:**
```go
type Info interface {
    GetName() string
    GetUID() string
    GetGroups() []string
    GetExtra() map[string][]string
}
```

---

### 4. Authorization System

**Purpose**: Determine if user has permission for operation.

```mermaid
graph TB
    UserInfo[User Info] --> Attrs[Build<br/>Attributes]
    Attrs --> Chain{Authorizer<br/>Chain}

    Chain --> RBAC[RBAC<br/>Authorizer]
    Chain --> Node[Node<br/>Authorizer]
    Chain --> Webhook[Webhook<br/>Authorizer]

    RBAC -->|Check roles| Decision1{Decision}
    Node -->|Check kubelet| Decision2{Decision}
    Webhook -->|External call| Decision3{Decision}

    Decision1 -->|Allow| Allowed[Allowed]
    Decision1 -->|NoOpinion| Chain
    Decision1 -->|Deny| Denied[Denied]

    Decision2 -->|Allow| Allowed
    Decision2 -->|NoOpinion| Chain

    Decision3 -->|Allow| Allowed
    Decision3 -->|NoOpinion| Denied

    style Chain fill:#ffcccc
    style Allowed fill:#ccffcc
    style Denied fill:#ffcccc
```

**Components:**

| Component | Purpose | Implementation |
|-----------|---------|----------------|
| **Authorizer Interface** | Base interface | `staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go` |
| **RBAC Authorizer** | Role-based authz | Via informers on Roles/RoleBindings |
| **Node Authorizer** | Kubelet-specific | Restricts kubelet access |
| **Webhook Authorizer** | External authz | POST to webhook |
| **Authorizer Factory** | Build chain | `staging/src/k8s.io/apiserver/pkg/authorization/authorizerfactory/` |

**Attributes Structure:**
```go
type Attributes interface {
    GetUser() user.Info
    GetVerb() string              // get, list, create, update, delete, watch
    GetNamespace() string
    GetResource() string
    GetSubresource() string
    GetName() string
    GetAPIGroup() string
    GetAPIVersion() string
}
```

---

### 5. Admission Control System

**Purpose**: Validate and mutate requests.

```mermaid
graph LR
    Request[Authorized<br/>Request] --> Mutating[Mutating<br/>Admission]
    Mutating --> Plugins1[Built-in<br/>Plugins]
    Plugins1 --> Webhooks1[Mutating<br/>Webhooks]
    Webhooks1 --> Validating[Validating<br/>Admission]
    Validating --> Plugins2[Built-in<br/>Plugins]
    Plugins2 --> Webhooks2[Validating<br/>Webhooks]
    Webhooks2 --> Policies[Admission<br/>Policies CEL]
    Policies --> Handler[API Handler]

    style Mutating fill:#ffcccc
    style Validating fill:#ccffcc
```

**Key Built-in Plugins:**

| Plugin | Type | Purpose |
|--------|------|---------|
| NamespaceLifecycle | Both | Prevent ops in terminating namespaces |
| LimitRanger | Validating | Enforce LimitRange constraints |
| ServiceAccount | Mutating | Inject SA tokens |
| DefaultStorageClass | Mutating | Set default StorageClass |
| ResourceQuota | Validating | Enforce quotas |
| PodSecurity | Validating | Enforce Pod Security Standards |
| MutatingAdmissionWebhook | Mutating | Call external webhooks |
| ValidatingAdmissionWebhook | Validating | Call external webhooks |

**Implementation**: `staging/src/k8s.io/apiserver/pkg/admission/`

---

### 6. Generic Registry

**Purpose**: Implement CRUD operations for resources using Strategy pattern.

```mermaid
classDiagram
    class Store {
        +NewFunc() Object
        +CreateStrategy RESTCreateStrategy
        +UpdateStrategy RESTUpdateStrategy
        +DeleteStrategy RESTDeleteStrategy
        +Storage storage.Interface
        +Create(ctx, obj, validate, options) error
        +Update(ctx, name, objInfo, validate, options) error
        +Delete(ctx, name, validate, options) error
        +Get(ctx, name, options) Object
        +List(ctx, options) ObjectList
    }

    class RESTCreateStrategy {
        <<interface>>
        +NamespaceScoped() bool
        +PrepareForCreate(ctx, obj)
        +Validate(ctx, obj) ErrorList
        +Canonicalize(obj)
    }

    class RESTUpdateStrategy {
        <<interface>>
        +PrepareForUpdate(ctx, obj, old)
        +ValidateUpdate(ctx, obj, old) ErrorList
        +AllowUnconditionalUpdate() bool
    }

    class RESTDeleteStrategy {
        <<interface>>
        +CheckGracefulDelete(ctx, obj, options) bool
    }

    Store *-- RESTCreateStrategy
    Store *-- RESTUpdateStrategy
    Store *-- RESTDeleteStrategy
```

**Example - Pod Strategy:**

```go
// File: pkg/registry/core/pod/strategy.go
type podStrategy struct {
    runtime.ObjectTyper
    names.NameGenerator
}

var Strategy = podStrategy{legacyscheme.Scheme, names.SimpleNameGenerator}

func (podStrategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
    pod := obj.(*api.Pod)
    pod.Status = api.PodStatus{Phase: api.PodPending}
    pod.Generation = 1
    // ... more preparation
}

func (podStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
    pod := obj.(*api.Pod)
    return validation.ValidatePod(pod)
}
```

**File**: `pkg/registry/generic/registry/store.go`

---

### 7. Storage Layer

**Purpose**: Abstract etcd access and provide caching.

```mermaid
graph TB
    Registry[Registry] --> Store[Generic<br/>Store]
    Store --> Decorator{Decorator<br/>Set?}

    Decorator -->|Yes| Cacher[Watch<br/>Cacher]
    Decorator -->|No| Storage[Storage<br/>Interface]

    Cacher -->|Cache miss| Storage

    Storage --> Transform[Storage<br/>Transformer]
    Transform --> Codec[Codec<br/>Protobuf]
    Codec --> etcd3[etcd3<br/>Client]
    etcd3 --> etcd[(etcd)]

    etcd --> Watch[Watch<br/>Events]
    Watch --> Cacher
    Cacher --> Clients[Watch<br/>Clients]

    style Cacher fill:#ccccff
    style etcd fill:#ffffcc
```

**Components:**

| Component | Purpose | Implementation |
|-----------|---------|----------------|
| **Storage Interface** | Storage abstraction | `staging/src/k8s.io/apiserver/pkg/storage/interfaces.go` |
| **etcd3 Store** | etcd implementation | `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go` |
| **Cacher** | Watch cache | `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go` |
| **Transformer** | Encryption/compression | `staging/src/k8s.io/apiserver/pkg/storage/value/` |

**Storage Operations:**
```go
type Interface interface {
    Create(ctx, key, obj, out, ttl) error
    Get(ctx, key, opts, out) error
    List(ctx, key, opts, listObj) error
    GuaranteedUpdate(ctx, key, out, ignoreNotFound, preconditions, updater) error
    Delete(ctx, key, out, preconditions, validateDeletion) error
    Watch(ctx, key, opts) (watch.Interface, error)
}
```

---

### 8. Watch Cache (Cacher)

**Purpose**: Efficiently serve watch requests from memory.

```mermaid
sequenceDiagram
    participant etcd
    participant Reflector
    participant Store
    participant EventBuffer
    participant Client

    etcd->>Reflector: Initial LIST
    Reflector->>Store: Store objects
    Reflector->>etcd: WATCH from rv=X

    loop Continuous sync
        etcd->>Reflector: Event (rv=X+1)
        Reflector->>Store: Update object
        Reflector->>EventBuffer: Add event
        EventBuffer->>Client: Stream event
    end

    loop Every 60s
        EventBuffer->>Client: Bookmark event
    end
```

**Key Features:**
- **In-memory storage** of recent objects
- **Ring buffer** of recent events
- **Bookmark events** every 60 seconds
- **Fall-through** to etcd if cache miss

**Configuration:**
```go
type Config struct {
    Storage storage.Interface              // Underlying etcd storage
    Versioner storage.Versioner            // Resource version mapper
    ResourcePrefix string                  // etcd key prefix
    EventsHistoryWindow time.Duration      // Event retention (75s default)
}
```

**File**: `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go`

---

### 9. API Priority and Fairness (APF)

**Purpose**: Prevent API server overload through fair queuing.

```mermaid
graph TB
    Request[Request] --> Classify[Classify into<br/>FlowSchema]

    Classify --> PL1[PriorityLevel:<br/>system]
    Classify --> PL2[PriorityLevel:<br/>workload-high]
    Classify --> PL3[PriorityLevel:<br/>workload-low]

    PL1 --> Q1{Seats<br/>available?}
    PL2 --> Q2{Seats<br/>available?}
    PL3 --> Q3{Seats<br/>available?}

    Q1 -->|Yes| Exec1[Execute]
    Q1 -->|No| Queue1[Queue]
    Queue1 -.->|Dequeue| Exec1

    Q2 -->|Yes| Exec2[Execute]
    Q2 -->|No| Queue2[Queue]
    Queue2 -.->|Dequeue| Exec2

    Q3 -->|Yes| Exec3[Execute]
    Q3 -->|No| Queue3[Queue]
    Queue3 -.->|Dequeue| Exec3

    style PL1 fill:#ffcccc
    style PL2 fill:#ffddcc
    style PL3 fill:#ffeecc
```

**Components:**

| Component | Purpose |
|-----------|---------|
| **FlowSchema** | Classifies requests (user, resource, verb) |
| **PriorityLevel** | Defines concurrency limits and queuing |
| **WorkEstimator** | Estimates request cost |
| **Queue** | Fair queuing discipline |

**Default Priority Levels:**
- **exempt**: Unlimited (health checks)
- **system**: 30 seats
- **leader-election**: 10 seats
- **workload-high**: 40 seats
- **workload-low**: 100 seats
- **global-default**: 20 seats (catch-all)

**File**: `staging/src/k8s.io/apiserver/pkg/util/flowcontrol/`

---

### 10. OpenAPI & Discovery

**Purpose**: Enable API discovery and schema generation.

```mermaid
graph LR
    GoTypes[Go Types<br/>with tags] --> Gen[openapi-gen<br/>tool]
    Gen --> OpenAPIDef[GetOpenAPIDefinitions<br/>func]
    OpenAPIDef --> Builder[OpenAPI<br/>Builder]
    Builder --> OpenAPIV2[OpenAPI v2<br/>Swagger]
    Builder --> OpenAPIV3[OpenAPI v3]

    Discovery[Discovery<br/>Aggregator] --> DiscDoc[Discovery<br/>Document]

    style OpenAPIV2 fill:#ccffcc
    style OpenAPIV3 fill:#ccccff
```

**Endpoints:**

| Endpoint | Purpose | Format |
|----------|---------|--------|
| `/openapi/v2` | OpenAPI v2 spec | JSON |
| `/openapi/v3` | OpenAPI v3 spec | JSON |
| `/api` | Core API discovery | JSON |
| `/apis` | API groups discovery | JSON |
| `/apis/{group}/{version}` | Resource discovery | JSON |

**Generated from code:**
```go
// +k8s:openapi-gen=true
type PodSpec struct {
    // Containers is a list of containers belonging to the pod.
    // +listType=map
    // +listMapKey=name
    Containers []Container `json:"containers"`
}
```

**Implementation**: `staging/src/k8s.io/apiserver/pkg/endpoints/openapi/`

---

### 11. Audit System

**Purpose**: Log all API operations for security and compliance.

```mermaid
graph TB
    Request[Request] --> AuditFilter[Audit<br/>Filter]
    AuditFilter --> Policy[Audit<br/>Policy]

    Policy -->|Metadata| Metadata[Metadata<br/>Event]
    Policy -->|Request| ReqEvent[Request<br/>Event]
    Policy -->|RequestResponse| RespEvent[RequestResponse<br/>Event]
    Policy -->|None| Skip[Skip]

    Metadata --> Backend{Backend}
    ReqEvent --> Backend
    RespEvent --> Backend

    Backend --> Log[Log<br/>Backend]
    Backend --> Webhook[Webhook<br/>Backend]

    Log --> File[Log<br/>File]
    Webhook --> External[External<br/>Service]

    style Policy fill:#ffcccc
    style Backend fill:#ccffcc
```

**Event Levels:**
- **None**: Don't log
- **Metadata**: Log metadata only (user, resource, verb)
- **Request**: Log metadata + request body
- **RequestResponse**: Log metadata + request + response

**Event Stages:**
- **RequestReceived**: Request entered API server
- **ResponseStarted**: Response headers sent
- **ResponseComplete**: Response body sent
- **Panic**: Request caused panic

**Implementation**: `staging/src/k8s.io/apiserver/pkg/audit/`

---

## Component Interactions

### Request Processing Flow

```mermaid
sequenceDiagram
    participant Client
    participant HandlerChain
    participant Auth
    participant Authz
    participant Admission
    participant Registry
    participant Storage
    participant etcd

    Client->>HandlerChain: POST /api/v1/namespaces/default/pods
    HandlerChain->>Auth: Authenticate
    Auth-->>HandlerChain: user.Info
    HandlerChain->>Authz: Authorize
    Authz-->>HandlerChain: Allowed
    HandlerChain->>Admission: Admit
    Admission->>Admission: Mutate (inject SA token)
    Admission->>Admission: Validate (check quota)
    Admission-->>HandlerChain: Admitted
    HandlerChain->>Registry: Create
    Registry->>Storage: Create
    Storage->>etcd: Create
    etcd-->>Storage: Success, rv=123
    Storage-->>Registry: Pod created
    Registry-->>Client: 201 Created
```

---

## Data Flow Through Components

### Write Operation (CREATE)

```mermaid
graph TB
    Client[Client:<br/>POST pod] --> TLS[TLS<br/>Termination]
    TLS --> HC[Handler<br/>Chain]
    HC --> Auth[Authentication]
    Auth --> Authz[Authorization]
    Authz --> APF[Priority &<br/>Fairness]
    APF --> Mutate[Mutating<br/>Admission]
    Mutate --> Validate[Validating<br/>Admission]
    Validate --> Handler[Create<br/>Handler]
    Handler --> Strategy[Pod<br/>Strategy]
    Strategy --> Validation[Pod<br/>Validation]
    Validation --> Store[Generic<br/>Store]
    Store --> Storage[Storage<br/>Interface]
    Storage --> Transform[Transformer<br/>Encrypt]
    Transform --> etcd[(etcd<br/>Create)]
    etcd --> Watch[Watch<br/>Events]
    Watch --> Cacher[Cacher<br/>Update]
    Cacher --> Watchers[Watch<br/>Clients]
    etcd --> Response[Response<br/>to Client]

    style Auth fill:#ffcccc
    style Authz fill:#ffcccc
    style Mutate fill:#ccffcc
    style Validate fill:#ccffcc
    style etcd fill:#ffffcc
```

### Read Operation (GET)

```mermaid
graph TB
    Client[Client:<br/>GET pod] --> TLS[TLS<br/>Termination]
    TLS --> HC[Handler<br/>Chain]
    HC --> Auth[Authentication]
    Auth --> Authz[Authorization]
    Authz --> APF[Priority &<br/>Fairness]
    APF --> Handler[Get<br/>Handler]
    Handler --> Store[Generic<br/>Store]
    Store --> Cacher{Cacher<br/>Enabled?}
    Cacher -->|Yes| Cache{In<br/>Cache?}
    Cache -->|Hit| Return1[Return<br/>Cached]
    Cache -->|Miss| Storage[Storage<br/>Interface]
    Cacher -->|No| Storage
    Storage --> Transform[Transformer<br/>Decrypt]
    Transform --> etcd[(etcd<br/>Get)]
    etcd --> Return2[Return<br/>to Client]
    Return1 --> Client2[Response]
    Return2 --> Client2

    style Cache fill:#ccffcc
    style etcd fill:#ffffcc
```

### Watch Operation

```mermaid
graph TB
    Client[Client:<br/>WATCH pods] --> TLS[TLS<br/>Termination]
    TLS --> HC[Handler<br/>Chain]
    HC --> Auth[Authentication]
    Auth --> Authz[Authorization]
    Authz --> Handler[Watch<br/>Handler]
    Handler --> Store[Generic<br/>Store]
    Store --> Cacher[Cacher]
    Cacher --> Check{rv in<br/>buffer?}
    Check -->|Yes| Stream1[Stream from<br/>Buffer]
    Check -->|No| Relist[410 Gone<br/>Relist]
    Stream1 --> Client2[Stream<br/>Events]

    etcd[(etcd)] --> Watch[Watch<br/>Events]
    Watch --> Cacher
    Cacher --> Stream2[Stream to<br/>Client]
    Stream2 --> Client2

    Cacher --> Bookmark[Bookmark<br/>Timer]
    Bookmark -.->|Every 60s| Stream2

    style Cacher fill:#ccccff
    style etcd fill:#ffffcc
```

---

## Component Dependencies

### Dependency Graph

```mermaid
graph TB
    Main[main] --> APIServer[GenericAPIServer]
    APIServer --> HandlerChain[Handler Chain]
    APIServer --> Registry[Generic Registry]
    APIServer --> StorageFactory[Storage Factory]
    APIServer --> Discovery[Discovery]
    APIServer --> OpenAPI[OpenAPI]

    HandlerChain --> Auth[Authenticators]
    HandlerChain --> Authz[Authorizers]
    HandlerChain --> Admission[Admission]
    HandlerChain --> APF[APF]
    HandlerChain --> Audit[Audit]

    Registry --> Strategy[Strategies]
    Registry --> Validation[Validation]
    Registry --> Store[Store]

    Store --> Cacher[Cacher]
    Cacher --> Storage[Storage Interface]
    Storage --> etcd3[etcd3 Client]
    etcd3 --> etcd[(etcd)]

    StorageFactory --> Storage

    style APIServer fill:#ffcccc
    style HandlerChain fill:#ccffcc
    style Registry fill:#ccccff
    style etcd fill:#ffffcc
```

---

## Summary

The kube-apiserver is composed of **11 major subsystems**:

1. **GenericAPIServer** - Base server infrastructure
2. **Handler Chain** - 24-layer request processing pipeline
3. **Authentication** - Multi-strategy identity verification
4. **Authorization** - Permission checking (RBAC, Node, Webhook)
5. **Admission Control** - Request validation and mutation
6. **Generic Registry** - CRUD operations with Strategy pattern
7. **Storage Layer** - etcd abstraction
8. **Watch Cache** - Efficient real-time notifications
9. **API Priority & Fairness** - Fair queuing and rate limiting
10. **OpenAPI & Discovery** - API schema and discovery
11. **Audit System** - Operation logging

**Key Characteristics:**
- **Modular**: Each component has clear responsibility
- **Composable**: Components work together via interfaces
- **Extensible**: Pluggable authenticators, authorizers, admission
- **Performant**: Watch cache, protobuf, APF
- **Observable**: Audit logging, metrics, tracing

**Next Steps:**
- Dive into [Middle-Level Architecture](../middle-level/) for component details
- Explore [Low-Level Technical Specs](../low-level/) for implementation details
- View [Diagrams](../diagrams/) for visual architecture

---

**Code References:**
- GenericAPIServer: `staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go`
- Handler Chain: `staging/src/k8s.io/apiserver/pkg/server/config.go:1014-1091`
- Generic Registry: `pkg/registry/generic/registry/store.go`
- Watch Cache: `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go`
- APF: `staging/src/k8s.io/apiserver/pkg/util/flowcontrol/`
