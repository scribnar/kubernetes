# Request Pipeline

> **Middle-Level Architecture: Complete request processing flow from client to storage**

---

## Table of Contents

- [Overview](#overview)
- [Complete Pipeline](#complete-pipeline)
- [Pipeline Stages](#pipeline-stages)
- [Request Flow Examples](#request-flow-examples)
- [Error Handling](#error-handling)
- [Performance Considerations](#performance-considerations)

---

## Overview

Every API request flows through a **carefully orchestrated pipeline** with multiple stages, each responsible for a specific aspect of request processing.

### Pipeline Stages

```mermaid
graph TB
    Client[Client Request] --> Stage1[1. TLS Termination]
    Stage1 --> Stage2[2. HTTP/2 Handler]
    Stage2 --> Stage3[3. Handler Chain 24 Filters]
    Stage3 --> Stage4[4. API Handler]
    Stage4 --> Stage5[5. Strategy & Validation]
    Stage5 --> Stage6[6. Storage Layer]
    Stage6 --> Stage7[7. etcd]
    Stage7 --> Response[Response to Client]

    style Stage1 fill:#e1f5ff
    style Stage3 fill:#ffcccc
    style Stage4 fill:#ccffcc
    style Stage6 fill:#ccccff
    style Stage7 fill:#ffffcc
```

**Processing Time** (typical):
- Authentication: <1ms
- Authorization: <5ms
- Admission webhooks: 10-100ms
- Storage: 10-50ms
- **Total**: 20-200ms (p99)

---

## Complete Pipeline

### Full Request Flow Diagram

```mermaid
sequenceDiagram
    participant Client
    participant TLS
    participant Handler Chain
    participant Auth
    participant Authz
    participant APF
    participant Admission
    participant Registry
    participant Strategy
    participant Storage
    participant etcd

    Client->>TLS: HTTPS Request
    TLS->>Handler Chain: Decrypted Request

    Note over Handler Chain: 24 Sequential Filters

    Handler Chain->>Handler Chain: 1. Panic Recovery
    Handler Chain->>Handler Chain: 2. Request Info Extract
    Handler Chain->>Handler Chain: 3. Wait Group Track
    Handler Chain->>Handler Chain: 4. Timeout Set

    Handler Chain->>Auth: 5. Authenticate
    Auth-->>Handler Chain: user.Info

    Handler Chain->>Handler Chain: 6. Impersonation Check

    Handler Chain->>Authz: 7. Authorize
    Authz-->>Handler Chain: Allow/Deny

    Handler Chain->>Handler Chain: 8. Audit Log

    Handler Chain->>APF: 9. Priority & Fairness
    APF-->>Handler Chain: Acquire Seat

    Handler Chain->>Admission: 10. Admission Control

    Note over Admission: Mutating Admission
    Admission->>Admission: Built-in Mutators
    Admission->>Admission: Mutating Webhooks

    Note over Admission: Validating Admission
    Admission->>Admission: Built-in Validators
    Admission->>Admission: Validating Webhooks
    Admission-->>Handler Chain: Admitted

    Handler Chain->>Registry: 11. API Handler

    Registry->>Strategy: Prepare & Validate
    Strategy->>Strategy: PrepareForCreate/Update
    Strategy->>Strategy: Validate

    Strategy->>Storage: Store
    Storage->>etcd: Write
    etcd-->>Storage: Success, rv=123
    Storage-->>Registry: Object Created
    Registry-->>Client: 201 Created

    Note over Storage: Watch Events
    etcd->>Storage: Watch Event
    Storage->>Client: Stream to Watchers
```

---

## Pipeline Stages

### Stage 1: TLS Termination

**Purpose**: Decrypt HTTPS traffic.

**Components**:
- TLS listener
- Certificate validation (for client cert authentication)
- HTTP/2 upgrade

**Configuration**:
```go
tlsConfig := &tls.Config{
    Certificates: []tls.Certificate{serverCert},
    ClientAuth:   tls.RequestClientCert,  // For X.509 auth
    MinVersion:   tls.VersionTLS12,
    CipherSuites: secureCipherSuites,
}
```

**File**: `staging/src/k8s.io/apiserver/pkg/server/secure_serving.go`

---

### Stage 2: HTTP/2 Handler

**Purpose**: Route request to appropriate handler.

**Director Pattern**:
```go
func (d director) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    path := req.URL.Path

    // API requests → go-restful
    if strings.HasPrefix(path, "/apis/") ||
       strings.HasPrefix(path, "/api/") {
        d.goRestfulContainer.ServeHTTP(w, req)
        return
    }

    // Metrics, health, debug → direct mux
    d.nonGoRestfulMux.ServeHTTP(w, req)
}
```

**Routes**:
- `/api/*` → Core API handler
- `/apis/*` → Named API groups
- `/healthz` → Health check
- `/metrics` → Prometheus metrics
- `/openapi/*` → OpenAPI specs

---

### Stage 3: Handler Chain (24 Filters)

**Purpose**: Process request through security, observability, and control layers.

**Complete Filter List**:

```mermaid
graph TB
    R[Request] --> F1[1. Audit Init]
    F1 --> F2[2. Panic Recovery]
    F2 --> F3[3. Request Info]
    F3 --> F4[4. Mux Discovery Complete]
    F4 --> F5[5. Request Timestamp]
    F5 --> F6[6. HTTP Logging]
    F6 --> F7[7. HSTS Header]
    F7 --> F8[8. Wait Group]
    F8 --> F9[9. Request Deadline]
    F9 --> F10[10. Timeout Handler]
    F10 --> F11[11. Warning Recorder]
    F11 --> F12[12. CORS]
    F12 --> F13[13. Authentication]
    F13 --> F14[14. Impersonation]
    F14 --> F15[15. Audit]
    F15 --> F16[16. Authorization]
    F16 --> F17[17. Priority & Fairness]
    F17 --> F18[18. Admission]
    F18 --> H[API Handler]

    style F13 fill:#ffcccc
    style F16 fill:#ffcccc
    style F17 fill:#ccccff
    style F18 fill:#ccffcc
```

**Filter Details**:

| # | Filter | Can Short-Circuit | Purpose |
|---|--------|-------------------|---------|
| 1 | Audit Init | No | Initialize audit context |
| 2 | Panic Recovery | No | Catch panics, return 500 |
| 3 | Request Info | No | Extract verb, resource, namespace |
| 4 | Mux Discovery Complete | Yes | Ensure all paths registered |
| 5 | Request Timestamp | No | Track arrival time |
| 6 | HTTP Logging | No | Log all HTTP requests |
| 7 | HSTS Header | No | Add Strict-Transport-Security |
| 8 | Wait Group | No | Track in-flight requests |
| 9 | Request Deadline | No | Apply request timeout |
| 10 | Timeout Handler | No | Wrap with timeout context |
| 11 | Warning Recorder | No | Record deprecation warnings |
| 12 | CORS | Yes | CORS preflight handling |
| 13 | **Authentication** | **Yes** | **Verify identity** |
| 14 | Impersonation | No | Support user impersonation |
| 15 | Audit | No | Log operation |
| 16 | **Authorization** | **Yes** | **Check permissions** |
| 17 | **Priority & Fairness** | **Yes** | **Rate limiting** |
| 18 | **Admission** | **Yes** | **Validate/mutate** |

**File**: `staging/src/k8s.io/apiserver/pkg/server/config.go:1014-1091`

---

### Stage 4: Authentication

**Purpose**: Verify the identity of the requester.

```mermaid
graph TB
    Request[HTTP Request] --> Extract[Extract Credentials]

    Extract --> Chain{Authenticator Chain}

    Chain --> Cert[X.509 Certificate]
    Chain --> Bearer[Bearer Token]
    Chain --> Basic[HTTP Basic]
    Chain --> Anon[Anonymous]

    Cert -->|CN, O| UserInfo1[user.Info]
    Bearer --> TokenType{Token Type?}
    TokenType --> SA[Service Account]
    TokenType --> OIDC[OIDC]
    TokenType --> Webhook[Webhook]

    SA -->|Validate JWT| UserInfo2[user.Info]
    OIDC -->|Validate with IdP| UserInfo3[user.Info]
    Webhook -->|POST to webhook| UserInfo4[user.Info]

    Basic -->|Username/Password| UserInfo5[user.Info]
    Anon -->|Enabled?| UserInfo6[system:anonymous]

    UserInfo1 --> Next[Next Filter]
    UserInfo2 --> Next
    UserInfo3 --> Next
    UserInfo4 --> Next
    UserInfo5 --> Next
    UserInfo6 --> Next

    Chain -->|All fail| Reject[401 Unauthorized]

    style Chain fill:#ffcccc
    style Reject fill:#ff9999
```

**Authentication Flow**:
1. Try each authenticator in sequence
2. First successful authenticator wins
3. Attach `user.Info` to request context
4. Remove sensitive headers (Authorization, Impersonate-*)
5. Continue to next filter

**User Information**:
```go
type Info interface {
    GetName() string          // "alice" or "system:serviceaccount:default:myapp"
    GetUID() string           // Unique ID
    GetGroups() []string      // ["system:authenticated", "developers"]
    GetExtra() map[string][]string  // Additional claims
}
```

**File**: `staging/src/k8s.io/apiserver/pkg/endpoints/filters/authentication.go`

**→ See**: [Authentication Details](04-authentication.md)

---

### Stage 5: Authorization

**Purpose**: Determine if the user has permission for the operation.

```mermaid
graph TB
    UserInfo[Authenticated User] --> BuildAttrs[Build Authorization Attributes]

    BuildAttrs --> Attrs[Attributes:<br/>User: alice<br/>Verb: create<br/>Resource: pods<br/>Namespace: default]

    Attrs --> Chain{Authorizer Chain}

    Chain --> RBAC[RBAC Authorizer]
    Chain --> Node[Node Authorizer]
    Chain --> Webhook[Webhook Authorizer]
    Chain --> ABAC[ABAC Authorizer]

    RBAC -->|Check Roles| D1{Decision}
    Node -->|Check Kubelet| D2{Decision}
    Webhook -->|External Call| D3{Decision}
    ABAC -->|Check Policy| D4{Decision}

    D1 -->|Allow| Allowed[✓ Allowed]
    D1 -->|NoOpinion| Chain
    D1 -->|Deny| Denied[✗ Denied]

    D2 -->|Allow| Allowed
    D2 -->|NoOpinion| Chain

    D3 -->|Allow| Allowed
    D3 -->|NoOpinion| Chain

    D4 -->|Allow| Allowed
    D4 -->|NoOpinion/Deny| Denied

    Allowed --> AuditAllow[Audit: Allowed]
    Denied --> AuditDeny[Audit: Denied]
    AuditDeny --> Reject[403 Forbidden]

    style Allowed fill:#ccffcc
    style Denied fill:#ffcccc
```

**Authorization Attributes**:
```go
type Attributes interface {
    GetUser() user.Info
    GetVerb() string              // get, list, create, update, delete, watch
    GetNamespace() string
    GetResource() string          // pods, services, deployments
    GetSubresource() string       // status, scale, log, exec
    GetName() string              // Specific resource name
    GetAPIGroup() string
    GetAPIVersion() string
}
```

**File**: `staging/src/k8s.io/apiserver/pkg/endpoints/filters/authorization.go`

**→ See**: [Authorization Details](05-authorization.md)

---

### Stage 6: Priority & Fairness (APF)

**Purpose**: Prevent API server overload through fair queuing.

```mermaid
graph LR
    Request[Authorized Request] --> Classify[Classify Request]

    Classify --> FS{FlowSchema<br/>Match}
    FS -->|system| PL1[PriorityLevel: system<br/>Concurrency: 30]
    FS -->|workload-high| PL2[PriorityLevel: workload-high<br/>Concurrency: 40]
    FS -->|workload-low| PL3[PriorityLevel: workload-low<br/>Concurrency: 100]

    PL1 --> Seats1{Seats<br/>Available?}
    PL2 --> Seats2{Seats<br/>Available?}
    PL3 --> Seats3{Seats<br/>Available?}

    Seats1 -->|Yes| Exec1[Execute Request]
    Seats1 -->|No| Q1[Queue<br/>Wait for seat]
    Q1 -.->|Timeout| Reject1[429 Too Many Requests]
    Q1 -.->|Dequeue| Exec1

    Seats2 -->|Yes| Exec2[Execute Request]
    Seats2 -->|No| Q2[Queue]

    Seats3 -->|Yes| Exec3[Execute Request]
    Seats3 -->|No| Q3[Queue]

    style PL1 fill:#ffcccc
    style PL2 fill:#ffddcc
    style PL3 fill:#ffeecc
    style Reject1 fill:#ff9999
```

**Work Estimation**:
- **List**: Estimated by result size
- **Watch**: Fixed cost per watch
- **Create/Update**: Based on object size
- **Get**: Minimal cost

**File**: `staging/src/k8s.io/apiserver/pkg/util/flowcontrol/`

**→ See**: [APF Details](08-api-priority-fairness.md)

---

### Stage 7: Admission Control

**Purpose**: Validate and mutate requests before persistence.

```mermaid
graph TB
    Request[Request with Seat] --> Phase1[Phase 1: Mutating Admission]

    Phase1 --> M1[NamespaceLifecycle]
    M1 --> M2[LimitRanger]
    M2 --> M3[ServiceAccount]
    M3 --> M4[DefaultStorageClass]
    M4 --> MW[Mutating Webhooks]

    MW --> Reinvoke{Need<br/>Reinvoke?}
    Reinvoke -->|Yes| Phase2
    Reinvoke -->|No| Phase2[Phase 2: Validating Admission]

    Phase2 --> V1[ResourceQuota]
    V1 --> V2[PodSecurity]
    V2 --> VW[Validating Webhooks]
    VW --> VP[Validating Admission Policy CEL]

    VP --> AllPass{All<br/>Passed?}
    AllPass -->|Yes| NextStage[To API Handler]
    AllPass -->|No| Reject[Reject Request<br/>400/403]

    style Phase1 fill:#ffcccc
    style Phase2 fill:#ccffcc
    style Reject fill:#ff9999
```

**Mutating Plugins** (Examples):
- **NamespaceLifecycle**: Prevent creating objects in terminating namespaces
- **ServiceAccount**: Inject service account token volumes
- **DefaultStorageClass**: Set default storage class for PVCs
- **MutatingAdmissionWebhook**: Call external mutation webhooks

**Validating Plugins** (Examples):
- **ResourceQuota**: Enforce resource quotas
- **PodSecurity**: Enforce Pod Security Standards
- **ValidatingAdmissionWebhook**: Call external validation webhooks
- **ValidatingAdmissionPolicy**: CEL-based validation

**File**: `staging/src/k8s.io/apiserver/pkg/admission/chain.go`

**→ See**: [Admission Control Details](06-admission-control.md)

---

### Stage 8: API Handler & Registry

**Purpose**: Execute the API operation (GET, CREATE, UPDATE, DELETE, etc.).

```mermaid
graph TB
    Request[Admitted Request] --> Router[API Router]

    Router -->|POST| Create[Create Handler]
    Router -->|GET| Get[Get Handler]
    Router -->|PUT| Update[Update Handler]
    Router -->|PATCH| Patch[Patch Handler]
    Router -->|DELETE| Delete[Delete Handler]
    Router -->|GET?watch=true| Watch[Watch Handler]

    Create --> Registry[Generic Registry]
    Get --> Registry
    Update --> Registry
    Patch --> Registry
    Delete --> Registry
    Watch --> Registry

    Registry --> Strategy[Resource Strategy]
    Strategy --> PrepareCreate[PrepareForCreate]
    Strategy --> Validate[Validate]
    Strategy --> Canonicalize[Canonicalize]

    Validate --> Storage[Storage Layer]

    style Registry fill:#ccffcc
    style Strategy fill:#ffddcc
```

**Strategy Pattern**:
```go
type RESTCreateStrategy interface {
    NamespaceScoped() bool
    PrepareForCreate(ctx context.Context, obj runtime.Object)
    Validate(ctx context.Context, obj runtime.Object) field.ErrorList
    Canonicalize(obj runtime.Object)
}
```

**Example - Pod Creation**:
```go
func (podStrategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
    pod := obj.(*api.Pod)
    pod.Status = api.PodStatus{Phase: api.PodPending}
    pod.Generation = 1
    podutil.DropDisabledPodFields(pod, nil)
}

func (podStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
    pod := obj.(*api.Pod)
    return validation.ValidatePod(pod)
}
```

**Files**:
- Generic registry: `pkg/registry/generic/registry/store.go`
- Pod strategy: `pkg/registry/core/pod/strategy.go`

---

### Stage 9: Storage Layer

**Purpose**: Persist to etcd with caching.

```mermaid
graph TB
    Registry[Registry] --> Store[Generic Store]
    Store --> Decorator{Has<br/>Decorator?}

    Decorator -->|Yes| Cacher[Watch Cacher]
    Decorator -->|No| Storage[Storage Interface]

    Cacher -->|Write-through| Storage

    Storage --> Transform[Storage Transformer]
    Transform --> Encrypt[Encryption at Rest]
    Encrypt --> Codec[Codec Protobuf/JSON]
    Codec --> etcd3[etcd3 Client]
    etcd3 --> etcd[(etcd)]

    etcd --> WatchEvent[Watch Events]
    WatchEvent --> Cacher
    Cacher --> WatchClients[Watch Clients]

    style Cacher fill:#ccccff
    style etcd fill:#ffffcc
```

**Storage Operations**:
```go
// Create
err := storage.Create(ctx, key, obj, out, ttl)

// Get
err := storage.Get(ctx, key, opts, out)

// Update (with optimistic locking)
err := storage.GuaranteedUpdate(ctx, key, out, ignoreNotFound,
    &storage.Preconditions{
        UID: &obj.UID,
        ResourceVersion: &obj.ResourceVersion,
    },
    func(existing runtime.Object) (runtime.Object, error) {
        // Update logic
        return updated, nil
    })

// Delete
err := storage.Delete(ctx, key, out, preconditions, validateDeletion)

// Watch
watcher, err := storage.Watch(ctx, key, opts)
```

**File**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go`

**→ See**: [Storage Layer Details](02-storage-layer.md)

---

## Request Flow Examples

### Example 1: Create Pod (Success)

```mermaid
sequenceDiagram
    participant kubectl
    participant API
    participant Auth
    participant RBAC
    participant Admission
    participant Registry
    participant etcd

    kubectl->>API: POST /api/v1/namespaces/default/pods
    Note over kubectl: Body: Pod spec

    API->>Auth: Authenticate
    Note over Auth: X.509 cert: CN=alice, O=developers
    Auth-->>API: user.Info{name: alice, groups: [developers]}

    API->>RBAC: Authorize create pods in default
    Note over RBAC: Check RoleBinding for alice
    RBAC-->>API: Allowed

    API->>Admission: Mutating Admission
    Note over Admission: Inject SA token volume
    Admission-->>API: Pod mutated

    API->>Admission: Validating Admission
    Note over Admission: Check ResourceQuota
    Admission-->>API: Admitted

    API->>Registry: Create Pod
    Registry->>Registry: Validate Pod spec
    Registry->>etcd: Create /registry/pods/default/mypod
    etcd-->>Registry: Success, rv=12345
    Registry-->>kubectl: 201 Created
    Note over kubectl: Pod created with rv=12345
```

### Example 2: Update Pod (Conflict)

```mermaid
sequenceDiagram
    participant Client1
    participant Client2
    participant API
    participant etcd

    API->>Client1: GET pod (rv=100)
    API->>Client2: GET pod (rv=100)

    Client1->>API: PUT pod (rv=100, spec.image=v2)
    API->>etcd: Update with precondition rv=100
    etcd-->>API: Success, rv=101
    API-->>Client1: 200 OK (rv=101)

    Client2->>API: PUT pod (rv=100, spec.replicas=3)
    API->>etcd: Update with precondition rv=100
    Note over etcd: rv=100 != current rv=101
    etcd-->>API: Precondition failed
    API-->>Client2: 409 Conflict
    Note over Client2: Must GET again and retry
```

### Example 3: Watch Pods

```mermaid
sequenceDiagram
    participant kubectl
    participant API
    participant Cacher
    participant etcd

    kubectl->>API: GET /api/v1/pods?watch=true
    API->>Cacher: Start watch from rv=0

    Note over Cacher: Initial snapshot
    Cacher->>kubectl: ADDED pod-1 (rv=100)
    Cacher->>kubectl: ADDED pod-2 (rv=101)

    Note over Cacher: Ongoing events
    etcd->>Cacher: Event: pod-3 created (rv=102)
    Cacher->>kubectl: ADDED pod-3 (rv=102)

    etcd->>Cacher: Event: pod-1 modified (rv=103)
    Cacher->>kubectl: MODIFIED pod-1 (rv=103)

    Note over Cacher: Bookmark event
    Cacher->>kubectl: BOOKMARK (rv=103)

    etcd->>Cacher: Event: pod-2 deleted (rv=104)
    Cacher->>kubectl: DELETED pod-2 (rv=104)
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | When Used |
|------|---------|-----------|
| 200 | OK | Successful GET, UPDATE, DELETE |
| 201 | Created | Successful CREATE |
| 400 | Bad Request | Invalid request body, schema validation failed |
| 401 | Unauthorized | Authentication failed |
| 403 | Forbidden | Authorization failed, admission denied |
| 404 | Not Found | Resource not found |
| 409 | Conflict | ResourceVersion mismatch, already exists |
| 410 | Gone | Watch too old, client must relist |
| 422 | Unprocessable Entity | Validation failed |
| 429 | Too Many Requests | Rate limited by APF |
| 500 | Internal Server Error | Unexpected error |
| 503 | Service Unavailable | Server shutting down |

### Error Response Format

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods \"mypod\" already exists",
  "reason": "AlreadyExists",
  "details": {
    "name": "mypod",
    "kind": "pods"
  },
  "code": 409
}
```

---

## Performance Considerations

### Optimization Techniques

**1. Watch Cache**
- Serves most LIST and WATCH requests from memory
- Reduces etcd load by 90%+
- 60-second bookmark events prevent expensive relists

**2. Protobuf**
- 3-5x smaller than JSON
- Faster serialization/deserialization
- Used for internal communication

**3. Priority & Fairness**
- Prevents overload
- Fair queuing ensures low-priority requests still processed
- Work estimation prevents expensive requests from monopolizing resources

**4. Request Coalescing**
- Multiple identical LIST requests can be coalesced
- Reduces duplicate work

**5. Indexers**
- Field selectors use indexes where possible
- `spec.nodeName` indexed for pods
- Speeds up filtered LIST operations

### Latency Breakdown (Typical)

```
Total Request Latency: 50ms (p99)
├─ TLS Handshake: 1ms (first request only)
├─ Authentication: <1ms (cached public keys)
├─ Authorization: 2ms (RBAC informer lookup)
├─ APF Queue: 0ms (no queue)
├─ Admission: 10ms (includes webhooks)
├─ Handler: 2ms
└─ Storage: 35ms (etcd round-trip)
```

---

## Summary

The kube-apiserver request pipeline is a **sophisticated, multi-stage** system:

**24 Handler Filters** ensure:
- ✅ Security (authentication, authorization)
- ✅ Observability (audit, metrics, tracing)
- ✅ Stability (rate limiting, timeouts)
- ✅ Validation (admission control)

**Key Characteristics**:
- **Layered**: Clear separation of concerns
- **Extensible**: Pluggable auth, authz, admission
- **Observable**: Every stage logged and metered
- **Performant**: Watch cache, protobuf, APF
- **Reliable**: Error handling at every stage

**Next**:
- [Storage Layer](02-storage-layer.md) - Deep dive into etcd integration
- [Authentication](04-authentication.md) - Auth strategies in detail
- [Authorization](05-authorization.md) - RBAC and other modes
- [Admission Control](06-admission-control.md) - Plugins and webhooks

---

**Code References**:
- Handler chain: `staging/src/k8s.io/apiserver/pkg/server/config.go:1014-1091`
- Authentication: `staging/src/k8s.io/apiserver/pkg/endpoints/filters/authentication.go`
- Authorization: `staging/src/k8s.io/apiserver/pkg/endpoints/filters/authorization.go`
- Admission: `staging/src/k8s.io/apiserver/pkg/admission/chain.go`
- Registry: `pkg/registry/generic/registry/store.go`
