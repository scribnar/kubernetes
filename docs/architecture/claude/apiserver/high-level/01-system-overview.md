# Kube-APIServer: System Overview

> **High-Level Architecture: Understanding the role and context of kube-apiserver in Kubernetes**

---

## Table of Contents

- [Introduction](#introduction)
- [System Context](#system-context)
- [Core Responsibilities](#core-responsibilities)
- [Architectural Principles](#architectural-principles)
- [System Architecture](#system-architecture)
- [Key Characteristics](#key-characteristics)

---

## Introduction

The **Kubernetes API Server** (kube-apiserver) is the central control plane component that serves as the **unified API gateway** for the entire Kubernetes cluster. It is the **only component that directly communicates with etcd** and acts as the **front door** for all cluster operations.

### What is kube-apiserver?

```mermaid
graph TB
    subgraph "External World"
        Users[👤 Users<br/>kubectl, UI]
        Operators[🤖 Operators<br/>Custom Controllers]
        CI[🔧 CI/CD<br/>Automation]
    end

    subgraph "Control Plane"
        API[🏛️ kube-apiserver<br/>API Gateway]
        Controller[📋 Controller Manager]
        Scheduler[⏱️ Scheduler]
    end

    subgraph "Data Plane"
        Kubelet1[📦 Kubelet<br/>Node 1]
        Kubelet2[📦 Kubelet<br/>Node 2]
    end

    subgraph "Storage"
        etcd[(💾 etcd<br/>Cluster State)]
    end

    Users --> API
    Operators --> API
    CI --> API
    Controller --> API
    Scheduler --> API
    Kubelet1 --> API
    Kubelet2 --> API
    API <--> etcd

    style API fill:#ff9999,stroke:#333,stroke-width:4px
    style etcd fill:#ffffcc
```

**Key Points:**
- **Single Point of Entry**: All cluster interactions go through the API server
- **State Manager**: Manages all cluster state in etcd
- **Policy Enforcer**: Enforces authentication, authorization, and admission policies
- **Event Broadcaster**: Notifies clients of state changes via watch streams

### Why does kube-apiserver exist?

**Problems it solves:**

1. **Centralized Access Control**: Single place to enforce security policies
2. **Data Consistency**: Strong consistency guarantees via etcd
3. **API Versioning**: Supports multiple API versions simultaneously
4. **Extensibility**: Allows custom resources and APIs
5. **Resource Validation**: Ensures data integrity before persistence
6. **Event Notification**: Real-time updates to interested clients
7. **Audit Trail**: Complete audit log of all operations

---

## System Context

### Position in Kubernetes Architecture

```mermaid
graph TB
    subgraph "User Interfaces"
        kubectl[kubectl CLI]
        Dashboard[Kubernetes Dashboard]
        Helm[Helm]
    end

    subgraph "kube-apiserver"
        direction TB
        API[REST API Server]
        Auth[Authentication/Authorization]
        Admission[Admission Control]
        Storage[Storage Layer]
    end

    subgraph "Control Plane Components"
        direction LR
        CM[Controller<br/>Manager]
        Scheduler[Scheduler]
        CCM[Cloud Controller<br/>Manager]
    end

    subgraph "Nodes"
        direction LR
        Kubelet1[Kubelet]
        Kubelet2[Kubelet]
        KubeProxy[Kube-Proxy]
    end

    subgraph "Extension Points"
        direction LR
        Webhooks[Admission<br/>Webhooks]
        CRDs[Custom<br/>Resources]
        AggAPI[Aggregated<br/>APIs]
    end

    kubectl --> API
    Dashboard --> API
    Helm --> API

    API --> Auth
    Auth --> Admission
    Admission --> Storage

    CM --> API
    Scheduler --> API
    CCM --> API

    Kubelet1 --> API
    Kubelet2 --> API
    KubeProxy --> API

    Webhooks -.-> Admission
    CRDs -.-> Storage
    AggAPI -.-> API

    Storage --> etcd[(etcd)]

    style API fill:#ff9999
    style etcd fill:#ffffcc
```

### External Dependencies

| Component | Relationship | Purpose |
|-----------|--------------|---------|
| **etcd** | Storage backend | Persistent state storage |
| **Kubelet** | Client | Node agent reports status, receives pod specs |
| **Controller Manager** | Client | Reconciles cluster state |
| **Scheduler** | Client | Assigns pods to nodes |
| **Cloud Provider** | Optional integration | Cloud-specific operations |
| **External Webhooks** | Extension point | Custom admission logic |
| **Extension API Servers** | Aggregated | Custom API groups |
| **OIDC Provider** | Authentication | User identity verification |

---

## Core Responsibilities

### 1. API Gateway

**Expose RESTful APIs** for all Kubernetes resources:

```
GET    /api/v1/namespaces/default/pods
POST   /api/v1/namespaces/default/pods
PUT    /api/v1/namespaces/default/pods/mypod
DELETE /api/v1/namespaces/default/pods/mypod
WATCH  /api/v1/namespaces/default/pods?watch=true
```

**Responsibilities:**
- HTTP/HTTPS endpoint serving
- Content negotiation (JSON, Protobuf, YAML)
- API versioning (alpha, beta, stable)
- Request routing and delegation

**Implementation**: `cmd/kube-apiserver/app/server.go`

---

### 2. Authentication

**Verify the identity** of every request:

```mermaid
graph LR
    Request[HTTP Request] --> Auth{Authenticate}
    Auth -->|X.509 Cert| User1[user: alice<br/>groups: admins]
    Auth -->|Bearer Token| User2[user: system:serviceaccount:default:myapp]
    Auth -->|OIDC| User3[user: bob@example.com<br/>groups: developers]
    Auth -->|Fail| Reject[401 Unauthorized]

    style User1 fill:#ccffcc
    style User2 fill:#ccffcc
    style User3 fill:#ccffcc
    style Reject fill:#ffcccc
```

**Supported Methods:**
- X.509 client certificates
- Bearer tokens (service accounts, OIDC)
- Webhook token authentication
- Bootstrap tokens
- Anonymous (optional)

**Implementation**: `staging/src/k8s.io/apiserver/pkg/authentication/`

---

### 3. Authorization

**Determine if the authenticated user has permission**:

```mermaid
graph TB
    User[Authenticated User] --> Authz{Authorize}
    Authz --> RBAC{RBAC Check}
    RBAC -->|Has Permission| Allow[✓ Allowed]
    RBAC -->|No Permission| Node{Node Authz}
    Node -->|Is Kubelet Resource| Allow
    Node -->|Not Allowed| Webhook{Webhook Authz}
    Webhook -->|Allow| Allow
    Webhook -->|Deny| Deny[✗ 403 Forbidden]

    style Allow fill:#ccffcc
    style Deny fill:#ffcccc
```

**Authorization Modes:**
- **RBAC** (recommended): Role-based access control
- **Node**: Kubelet-specific authorization
- **Webhook**: External authorization decisions
- **ABAC**: Attribute-based (deprecated)

**Implementation**: `staging/src/k8s.io/apiserver/pkg/authorization/`

---

### 4. Admission Control

**Validate and mutate requests** before persistence:

```mermaid
graph LR
    Authorized[Authorized Request] --> Mutate[Mutating Admission]
    Mutate --> Validate[Validating Admission]
    Validate --> Decision{All Pass?}
    Decision -->|Yes| Persist[Persist to etcd]
    Decision -->|No| Reject[Reject Request]

    style Persist fill:#ccffcc
    style Reject fill:#ffcccc
```

**Admission Types:**
- **Built-in plugins**: NamespaceLifecycle, LimitRanger, ResourceQuota, etc.
- **Mutating webhooks**: External mutation logic
- **Validating webhooks**: External validation logic
- **Admission policies**: CEL-based policies

**Example**: Injecting service account tokens, enforcing resource quotas

**Implementation**: `staging/src/k8s.io/apiserver/pkg/admission/`

---

### 5. Storage Management

**Persist and retrieve cluster state** from etcd:

```mermaid
graph TB
    API[API Request] --> Cache{Watch Cache}
    Cache -->|Hit| Return[Return Cached]
    Cache -->|Miss| Storage[Storage Layer]
    Storage --> Transform[Transform<br/>Encrypt/Compress]
    Transform --> etcd[(etcd)]

    etcd --> Watch[Watch Events]
    Watch --> Cache
    Cache --> Clients[Watch Clients]

    style Cache fill:#ccffcc
    style etcd fill:#ffffcc
```

**Operations:**
- **Create**: Insert new resources
- **Get/List**: Retrieve resources
- **Update**: Modify existing resources (with optimistic locking)
- **Delete**: Remove resources (with finalizers)
- **Watch**: Stream changes in real-time

**Key Features:**
- Optimistic concurrency control (resourceVersion)
- Watch caching for performance
- Encryption at rest
- Storage versioning

**Implementation**: `staging/src/k8s.io/apiserver/pkg/storage/`

---

### 6. Watch Streams

**Provide real-time notifications** of resource changes:

```mermaid
sequenceDiagram
    participant Client
    participant Cacher
    participant etcd

    Client->>Cacher: WATCH pods
    Note over Cacher: Initial snapshot
    Cacher->>Client: ADDED pod-1
    Cacher->>Client: ADDED pod-2

    etcd->>Cacher: Event: pod-3 created
    Cacher->>Client: ADDED pod-3

    etcd->>Cacher: Event: pod-1 deleted
    Cacher->>Client: DELETED pod-1

    loop Every 60s
        Cacher->>Client: BOOKMARK (keep alive)
    end
```

**Watch Mechanism:**
- Efficient streaming via HTTP/2
- Bookmark events prevent timeouts
- Watch cache reduces etcd load
- Resume from last seen resourceVersion

**Implementation**: `staging/src/k8s.io/apiserver/pkg/storage/cacher/`

---

### 7. API Discovery

**Enable clients to discover** available APIs and resources:

```
GET /api              → Core API groups
GET /apis             → Named API groups
GET /apis/apps/v1     → Resources in apps/v1
GET /openapi/v2       → OpenAPI v2 spec
GET /openapi/v3       → OpenAPI v3 spec
```

**Provides:**
- API group enumeration
- Resource type discovery
- Field and schema information
- Supported operations per resource

**Implementation**: `staging/src/k8s.io/apiserver/pkg/endpoints/discovery/`

---

### 8. Extensibility

**Support custom resources and APIs**:

```mermaid
graph TB
    subgraph "Built-in APIs"
        Core[Core APIs<br/>pods, services, etc.]
        Apps[apps/v1<br/>deployments, etc.]
        Batch[batch/v1<br/>jobs, etc.]
    end

    subgraph "Extension Mechanisms"
        CRD[CustomResourceDefinitions<br/>User-defined resources]
        Agg[API Aggregation<br/>Extension API servers]
        Webhooks[Admission Webhooks<br/>Custom validation/mutation]
    end

    APIServer[kube-apiserver] --> Core
    APIServer --> Apps
    APIServer --> Batch
    APIServer --> CRD
    APIServer --> Agg
    APIServer --> Webhooks

    style APIServer fill:#ff9999
    style CRD fill:#ccffcc
    style Agg fill:#ccccff
    style Webhooks fill:#ffffcc
```

**Extension Points:**
1. **CustomResourceDefinitions (CRDs)**: Define new resource types
2. **API Aggregation**: Run separate API servers
3. **Admission Webhooks**: Custom validation/mutation logic
4. **Custom Authenticators/Authorizers**: Pluggable auth

**Implementation**:
- CRDs: `staging/src/k8s.io/apiextensions-apiserver/`
- Aggregation: `staging/src/k8s.io/kube-aggregator/`

---

## Architectural Principles

### 1. Single Source of Truth

```
All cluster state lives in etcd
   ↓
Only kube-apiserver writes to etcd
   ↓
All components read from kube-apiserver
   ↓
Consistent view of cluster state
```

**Benefits:**
- Strong consistency guarantees
- Centralized access control
- Simplified state management
- Audit trail of all changes

---

### 2. Declarative API

**Desired State Model:**

```mermaid
graph LR
    User[User] -->|Declares| Desired[Desired State]
    Desired -->|Stored in| API[API Server]
    API -->|Persisted to| etcd[(etcd)]
    Controller[Controllers] -->|Watch| API
    Controller -->|Reconcile| Current[Current State]
    Current -.->|Converge to| Desired

    style Desired fill:#ccffcc
    style API fill:#ff9999
```

**Characteristics:**
- Users specify **what** they want, not **how** to achieve it
- Controllers continuously reconcile current state → desired state
- Idempotent operations
- Self-healing system

---

### 3. Level-Based Triggers

**Watch-based reconciliation** (not edge-triggered):

```
Controller watches for changes
   ↓
Reads current state from API
   ↓
Compares to desired state
   ↓
Takes action to reconcile
   ↓
(Repeat)
```

**Benefits:**
- Resilient to missed events
- Self-correcting
- Works across restarts

---

### 4. Optimistic Concurrency

**Conflict detection via resourceVersion**:

```mermaid
sequenceDiagram
    participant Client1
    participant Client2
    participant APIServer

    APIServer->>Client1: GET pod (rv=100)
    APIServer->>Client2: GET pod (rv=100)

    Client1->>APIServer: PUT pod (rv=100)
    APIServer->>Client1: Success (rv=101)

    Client2->>APIServer: PUT pod (rv=100)
    APIServer->>Client2: 409 Conflict
    Note over Client2: Must retry with rv=101
```

**No distributed locks needed!**

---

### 5. Extensibility by Design

**Plugin architecture** throughout:

| Layer | Extension Mechanism |
|-------|---------------------|
| Authentication | Pluggable authenticators |
| Authorization | Pluggable authorizers |
| Admission | Pluggable admission plugins + webhooks |
| Storage | Pluggable storage backends (only etcd in practice) |
| API | CRDs, aggregation |

---

### 6. Defense in Depth

**Multiple security layers**:

```
1. Network (TLS)
   ↓
2. Authentication (Who are you?)
   ↓
3. Authorization (What can you do?)
   ↓
4. Admission (Is this request valid?)
   ↓
5. Validation (Does this meet schema?)
   ↓
6. Storage (Encrypted at rest)
```

---

## System Architecture

### Three-Server Delegation Chain

```mermaid
graph TB
    Client[Client Request] --> Agg[Aggregator Server]

    Agg -->|Extension APIs| ExtAPI[Extension API Servers]
    Agg -->|Built-in APIs| Kube[Kube API Server]

    Kube -->|CRDs| CRDServer[API Extensions Server]

    CRDServer -->|Not found| NotFound[404 Handler]

    subgraph "kube-apiserver Process"
        Agg
        Kube
        CRDServer
    end

    style Agg fill:#ffcccc
    style Kube fill:#ccffcc
    style CRDServer fill:#ccccff
```

**Delegation Order:**
1. **Aggregator**: Handles APIService routing to extension servers
2. **Kube API**: Handles built-in Kubernetes APIs
3. **API Extensions**: Handles CustomResourceDefinitions
4. **NotFound**: Returns 404 for unknown paths

**File**: `cmd/kube-apiserver/app/server.go:176-197`

---

### Request Processing Pipeline

```mermaid
graph TB
    Request[HTTP Request] --> Filter1[TLS Termination]
    Filter1 --> Filter2[Request Info]
    Filter2 --> Filter3[Authentication]
    Filter3 --> Filter4[Authorization]
    Filter4 --> Filter5[Priority & Fairness]
    Filter5 --> Filter6[Admission Control]
    Filter6 --> Handler[API Handler]
    Handler --> Registry[Registry]
    Registry --> Storage[Storage Layer]
    Storage --> etcd[(etcd)]

    style Filter3 fill:#ffcccc
    style Filter4 fill:#ffcccc
    style Filter6 fill:#ccffcc
    style etcd fill:#ffffcc
```

**24-layer handler chain** processes each request!

**File**: `staging/src/k8s.io/apiserver/pkg/server/config.go:1014-1091`

---

## Key Characteristics

### Performance

**Metrics (typical production instance):**
- **Throughput**: 1000+ req/sec
- **Latency**: <100ms p99 (cached reads)
- **Watch connections**: 5000+ concurrent
- **Memory**: 2-8 GB depending on cluster size

**Optimizations:**
- Watch cache reduces etcd load by 90%+
- Protobuf reduces bandwidth vs JSON
- Priority & Fairness prevents overload

---

### Scalability

**Horizontal scaling**:
- Multiple active API server instances
- Stateless design (all state in etcd)
- Load balancer distributes requests
- Supports 5000+ node clusters

---

### Availability

**High availability setup**:
- Active-active deployment (3+ instances)
- Watch cache provides read availability during etcd downtime
- Graceful shutdown prevents request drops
- Health checks (liveness, readiness)

---

### Security

**Multi-layered security**:
- TLS for all communication
- Multiple authentication methods
- Fine-grained RBAC
- Audit logging
- Admission webhooks for custom policies
- Encryption at rest

---

## Summary

The **kube-apiserver** is the **heart of Kubernetes**, serving as:

✅ **API Gateway**: Single entry point for all cluster operations
✅ **State Manager**: Persistent storage via etcd with strong consistency
✅ **Security Enforcer**: Authentication, authorization, admission control
✅ **Event Broadcaster**: Real-time watch streams
✅ **Extension Platform**: CRDs, aggregation, webhooks

**Key Strengths:**
- Centralized control and security
- Strong consistency guarantees
- Real-time event notifications
- Highly extensible architecture
- Battle-tested at massive scale

**Next Steps:**
- [Server Chain Architecture](02-server-chain-architecture.md) - Deep dive into delegation pattern
- [Initialization Flow](03-initialization-flow.md) - Startup sequence
- [Key Components](04-key-components.md) - Major subsystems

---

**Code References:**
- Entry point: `cmd/kube-apiserver/apiserver.go:37`
- Server creation: `cmd/kube-apiserver/app/server.go:176-197`
- Generic server: `staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go`
