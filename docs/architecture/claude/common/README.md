# Kubernetes Shared Libraries Architecture Documentation

**Version**: 1.0
**Status**: Comprehensive Architecture Analysis
**Last Updated**: 2025-10-20

---

## Overview

This directory contains comprehensive architecture documentation for the shared libraries used across all Kubernetes components (kube-apiserver, kube-scheduler, kube-controller-manager, kubelet, kube-proxy, etc.). These shared libraries provide the foundation for building Kubernetes components and custom controllers.

## Shared Libraries Analyzed

### Core Libraries (staging/src/k8s.io/)

1. **client-go** (~2,310 Go files)
   - REST clients for Kubernetes API
   - SharedInformers for watching resources
   - Workqueue for reliable task processing
   - Leader election framework
   - Listers for local caching
   - Client tools and utilities

2. **apimachinery** (~508 Go files)
   - Runtime schema and type system
   - Serialization and conversion framework
   - Meta types (ObjectMeta, TypeMeta)
   - Watch mechanism
   - Label and field selectors

3. **component-base** (~195 Go files)
   - Metrics framework (Prometheus)
   - Configuration management
   - Feature gates
   - Logging infrastructure
   - Version information
   - Tracing support

4. **apiserver** (~1,028 Go files)
   - Generic API server framework
   - Storage layer abstraction
   - Admission control
   - Authentication and authorization
   - API endpoints and discovery
   - Request filtering and handler chains

---

## Documentation Structure

### Part I: High-Level Overview (Document 1)

#### ✅ [01 - Overview and Introduction](./01-overview-and-introduction.md)
**Status**: To be created

**Content**:
- Executive summary of shared libraries
- Why these libraries exist and their purpose
- High-level statistics (file counts, usage patterns)
- Component dependency map
- Usage across Kubernetes components
- Key design principles
- Technology stack

**Key Diagrams**:
- Shared library dependency graph
- Component usage matrix (which components use which libraries)
- High-level architecture (C4 system context)
- Library interaction patterns

---

### Part II: client-go Library (Documents 2-4)

#### 📋 02 - client-go: REST Clients and Discovery

**Planned Content**:
- REST client architecture
- Request building and execution
- Content negotiation (JSON, Protobuf, CBOR)
- Rate limiting and backoff
- Client authentication
- Discovery client for API resources
- Dynamic client for unstructured resources

**Key Data Structures**:
- `RESTClient` - Generic REST client
- `Request` - Request builder
- `ClientContentConfig` - Content negotiation
- `RateLimiter` - Client-side rate limiting

**Diagrams to Include**:
- REST client class diagram
- Request execution flow
- Content negotiation sequence
- Rate limiting flowchart
- Discovery API flow

---

#### 📋 03 - client-go: Informers and SharedInformers

**Planned Content**:
- SharedInformer architecture
- SharedInformerFactory pattern
- Informer lifecycle (List, Watch, Resync)
- Event handler registration
- Reflector implementation
- DeltaFIFO queue
- Local cache (Store/Indexer)
- Resource event handlers
- Resync mechanism

**Key Components**:
1. **SharedInformer** - Watches and caches resources
2. **SharedInformerFactory** - Factory for creating informers
3. **Reflector** - List and watch resources
4. **DeltaFIFO** - Queue for resource changes
5. **Store/Indexer** - Local cache with indexing
6. **ResourceEventHandler** - Add/Update/Delete callbacks

**Diagrams to Include**:
- SharedInformer architecture diagram
- Informer lifecycle sequence
- List/Watch protocol flow
- DeltaFIFO operation
- Event handler distribution
- Resync mechanism flowchart
- Processor listener architecture

---

#### 📋 04 - client-go: Workqueue and Leader Election

**Planned Content**:

**Workqueue**:
- Workqueue interface and implementation
- Rate limiting queue
- Delay queue
- Metrics integration
- Processing tracking (dirty, processing, queue)
- Shutdown mechanisms

**Leader Election**:
- Leader election algorithm
- Lease-based implementation
- Acquire and renew flow
- Configuration (LeaseDuration, RenewDeadline, RetryPeriod)
- Callbacks (OnStartedLeading, OnStoppedLeading, OnNewLeader)
- Clock skew tolerance

**Diagrams to Include**:
- Workqueue class diagram
- Workqueue state machine (dirty, processing, queue)
- Rate limiting queue flowchart
- Leader election architecture
- Lease acquisition sequence
- Leader renewal flow
- Leader failover scenario

---

### Part III: apimachinery Library (Documents 5-7)

#### 📋 05 - apimachinery: Runtime and Scheme

**Planned Content**:
- Scheme architecture
- Type registration (GVK ↔ Go Type)
- Object kind determination
- Version priorities
- Unversioned types
- Defaulting functions
- Validation functions
- Object creation from GVK

**Key Data Structures**:
- `Scheme` - Type registry and conversion
- `GroupVersionKind` - API group, version, kind tuple
- `Object` - Base interface for all Kubernetes objects
- `TypeMeta` - Kind and APIVersion fields
- `ObjectMeta` - Metadata fields

**Diagrams to Include**:
- Scheme class diagram
- Type registration flow
- GVK resolution sequence
- Version priority mechanism
- Object creation from GVK

---

#### 📋 06 - apimachinery: Serialization and Conversion

**Planned Content**:
- Codec architecture
- Serializer chain
- Encoding formats (JSON, YAML, Protobuf, CBOR)
- Framer for streaming
- Conversion framework
- Converter architecture
- Auto-generated conversions
- Manual conversion functions
- Unstructured objects
- DefaultUnstructuredConverter

**Key Components**:
- `Codec` - Encoder and Decoder interface
- `NegotiatedSerializer` - Format negotiation
- `Converter` - Type conversion
- `Unstructured` - Dynamic objects

**Diagrams to Include**:
- Codec architecture
- Serialization pipeline
- Format negotiation flow
- Conversion sequence (version A → version B)
- Unstructured conversion flow

---

#### 📋 07 - apimachinery: Watch and Meta Types

**Planned Content**:

**Watch Mechanism**:
- Watch interface
- Watch event types (Added, Modified, Deleted, Error)
- Bookmark events
- Resource version tracking
- Watch stream framing

**Meta Types**:
- ObjectMeta fields and usage
- TypeMeta for kind identification
- ListMeta for list responses
- Label selectors
- Field selectors
- Owner references
- Finalizers

**Diagrams to Include**:
- Watch architecture
- Watch event flow
- Resource version progression
- ObjectMeta class diagram
- Label selector evaluation
- Owner reference chain

---

### Part IV: component-base Library (Documents 8-9)

#### 📋 08 - component-base: Metrics and Observability

**Planned Content**:
- Metrics architecture
- KubeRegistry vs Prometheus registry
- Metric stability levels (ALPHA, BETA, STABLE)
- Metric deprecation lifecycle
- Counter, Gauge, Histogram, Summary
- Metric labeling and cardinality
- Hidden metrics
- Disabled metrics
- SLI metrics
- Custom collectors

**Key Components**:
- `KubeRegistry` - Metric registration
- `Counter`, `Gauge`, `Histogram` - Metric types
- `MetricsProvider` - Metric factory
- `StabilityLevel` - ALPHA, BETA, STABLE

**Diagrams to Include**:
- Metrics architecture
- KubeRegistry class diagram
- Metric lifecycle (registration, collection, deprecation)
- Stability level transitions
- Metric scraping flow

---

#### 📋 09 - component-base: Config, Logs, and Feature Gates

**Planned Content**:

**Configuration**:
- Config loading patterns
- Flag binding
- Config file formats
- Environment variables

**Logging**:
- klog integration
- Structured logging
- Log levels
- Contextual logging
- Log output formats (text, JSON)

**Feature Gates**:
- Feature gate architecture
- Feature lifecycle (Alpha → Beta → GA)
- Runtime feature gate checks
- Feature gate configuration

**Version Information**:
- Version struct
- Git commit and tree state
- Build date and platform

**Diagrams to Include**:
- Config loading sequence
- Feature gate state machine
- Log flow architecture
- Version information structure

---

### Part V: apiserver Library (Documents 10-12)

#### 📋 10 - apiserver: Server Framework and Config

**Planned Content**:
- GenericAPIServer architecture
- Server configuration
- Secure serving setup
- Handler chain building
- Delegated authentication
- Request context
- Post-start hooks
- Pre-shutdown hooks
- Graceful shutdown
- Health checks (healthz, livez, readyz)

**Key Components**:
- `Config` - Server configuration
- `GenericAPIServer` - Server implementation
- `SecureServingInfo` - HTTPS configuration
- `AuthenticationInfo` - Authentication config
- `AuthorizationInfo` - Authorization config

**Diagrams to Include**:
- GenericAPIServer architecture
- Server initialization sequence
- Handler chain construction
- Request flow through filters
- Post-start hook execution
- Graceful shutdown sequence

---

#### 📋 11 - apiserver: Storage and Registry

**Planned Content**:
- Storage interface abstraction
- etcd3 storage implementation
- Cacher for watch caching
- Store interface
- Generic registry pattern
- RESTStorage implementation
- List and watch from storage
- Resource versioning
- Consistency guarantees

**Key Components**:
- `Interface` - Storage abstraction
- `Cacher` - Watch cache
- `Store` - REST CRUD operations
- `RESTStorage` - REST handlers for resources

**Diagrams to Include**:
- Storage architecture
- Storage interface hierarchy
- Cacher architecture
- Watch cache flow
- List/Watch from etcd
- Generic registry pattern

---

#### 📋 12 - apiserver: Admission, Authentication, and Authorization

**Planned Content**:

**Admission Control**:
- Admission chain architecture
- Admission plugins
- ValidatingAdmissionPolicy
- MutatingAdmissionWebhook
- ValidatingAdmissionWebhook
- Admission phase (Mutating → Validating)

**Authentication**:
- Authenticator interface
- Authentication strategies (tokens, certificates, webhooks)
- User and group extraction
- Service account tokens

**Authorization**:
- Authorizer interface
- RBAC authorizer
- Webhook authorizer
- Node authorizer
- Decision (Allow, Deny, NoOpinion)

**Diagrams to Include**:
- Admission chain sequence
- Admission webhook flow
- Authentication sequence
- Authorization decision flow
- Request filter chain

---

### Part VI: Integration and Patterns (Document 13)

#### 📋 13 - Common Patterns and Integration

**Planned Content**:
- Controller pattern using shared libraries
- Complete controller example architecture
- Informer + workqueue pattern
- Leader election for HA
- Metrics integration
- Error handling patterns
- Retry and backoff strategies
- Graceful shutdown patterns
- Testing patterns for controllers

**Example Integration**:
- Sample controller using informers, workqueue, and leader election
- Metrics and logging integration
- Configuration management

**Diagrams to Include**:
- Complete controller architecture
- Informer + Workqueue integration
- Leader election + controller pattern
- End-to-end request flow across libraries

---

## Library Statistics

| Library | Go Files | Key Packages | Used By |
|---------|----------|-------------|---------|
| **client-go** | ~2,310 | informers, workqueue, rest, tools | All components |
| **apimachinery** | ~508 | runtime, apis/meta, conversion, watch | All components |
| **component-base** | ~195 | metrics, config, featuregate, logs | All components |
| **apiserver** | ~1,028 | server, storage, admission, auth | kube-apiserver, aggregated API servers |

---

## Key Cross-Cutting Concerns

### 1. Type System (apimachinery)
- **Scheme**: Type registry mapping GVK ↔ Go types
- **Conversion**: Cross-version object conversion
- **Serialization**: JSON, YAML, Protobuf, CBOR encoding/decoding

### 2. API Communication (client-go)
- **REST Client**: HTTP client for Kubernetes API
- **Informers**: Efficient watching and caching
- **Rate Limiting**: Client-side request throttling

### 3. Controller Pattern (client-go + workqueue)
- **SharedInformers**: Watch Kubernetes resources
- **Workqueues**: Reliable task queuing with rate limiting
- **Leader Election**: High availability

### 4. Observability (component-base)
- **Metrics**: Prometheus integration
- **Logging**: Structured logging with klog
- **Tracing**: OpenTelemetry support

### 5. API Server Framework (apiserver)
- **Generic Server**: Framework for building API servers
- **Storage**: Abstract storage with etcd3 backend
- **Admission/Auth**: Pluggable admission, authentication, authorization

---

## Usage Patterns Across Components

### kube-scheduler
- Uses: `client-go` (informers, listers), `apimachinery` (scheme), `component-base` (metrics, logs), `apiserver` (not directly)
- Patterns: Informer + Workqueue, Leader Election, Metrics

### kube-controller-manager
- Uses: `client-go` (informers, workqueue, leader election), `apimachinery`, `component-base`
- Patterns: Multiple controllers with informers, leader election, metrics

### kube-apiserver
- Uses: All libraries
- Patterns: API server framework, admission, authentication, authorization, storage, metrics

### kubelet
- Uses: `client-go` (REST client, informers), `apimachinery`, `component-base`
- Patterns: REST client for API server communication, metrics

### kube-proxy
- Uses: `client-go` (informers), `apimachinery`, `component-base`
- Patterns: Informers for watching services and endpoints

### Custom Controllers/Operators
- Uses: Primarily `client-go`, `apimachinery`, `component-base`
- Patterns: Informer + Workqueue + Leader Election

---

## Design Principles

### 1. **Separation of Concerns**
- Each library has a well-defined purpose
- Clear boundaries between libraries
- Minimal circular dependencies

### 2. **Interface-Based Design**
- Heavy use of interfaces for abstraction
- Allows for testing and mocking
- Enables pluggable implementations

### 3. **Consistency Guarantees**
- Watch mechanism provides eventual consistency
- Resource versioning for optimistic concurrency
- Atomic operations where needed

### 4. **Performance Optimization**
- Local caching (informers, listers)
- Efficient watch mechanism
- Rate limiting to protect API server
- Batch processing where possible

### 5. **Reliability**
- Retry with exponential backoff
- Workqueue with rate limiting
- Leader election for HA
- Graceful degradation

### 6. **Extensibility**
- Plugin architecture (admission, authentication, authorization)
- Custom metrics and logs
- Feature gates for gradual rollout
- Custom storage backends

---

## Common Workflows

### 1. **Watching Resources with Informers**
```
Client → SharedInformerFactory → SharedInformer → Reflector → List/Watch API
↓
DeltaFIFO → Event Handlers → Process Events
↓
Local Cache (Store) → Lister → Fast local queries
```

### 2. **Processing with Workqueue**
```
Event Handler → Workqueue.Add(key)
↓
Worker goroutine → Workqueue.Get()
↓
Process item → Handle errors → Workqueue.Done(key)
↓
Success: Remove | Failure: Requeue with backoff
```

### 3. **Leader Election for HA**
```
Multiple instances → Leader Election → One active leader
↓
Leader: OnStartedLeading → Run controller logic
↓
Leader renews lease every RetryPeriod
↓
If leader fails → New leader elected
```

### 4. **API Request Flow (apiserver)**
```
HTTP Request → Handler Chain Filters (auth, audit, etc.)
↓
Request dispatch → RESTStorage → Storage Interface
↓
etcd3 → Watch Cache → Response
```

---

## File and Package Organization

### client-go
```
k8s.io/client-go/
├── discovery/          # Discovery client
├── dynamic/            # Dynamic client for unstructured
├── informers/          # Shared informer factories
├── kubernetes/         # Typed clientsets
├── listers/            # Listers for local cache queries
├── rest/               # REST client
├── tools/
│   ├── cache/         # Informer, DeltaFIFO, Store
│   ├── clientcmd/     # Kubeconfig parsing
│   ├── leaderelection/# Leader election
│   └── record/        # Event recording
└── util/
    └── workqueue/     # Workqueue implementations
```

### apimachinery
```
k8s.io/apimachinery/pkg/
├── api/               # Resource helpers
├── apis/              # Meta types (metav1)
├── conversion/        # Conversion framework
├── fields/            # Field selectors
├── labels/            # Label selectors
├── runtime/           # Scheme, serialization
├── util/              # Utilities
├── version/           # Version parsing
└── watch/             # Watch interface
```

### component-base
```
k8s.io/component-base/
├── cli/               # CLI utilities
├── config/            # Config framework
├── featuregate/       # Feature gates
├── logs/              # Logging
├── metrics/           # Metrics framework
├── tracing/           # Tracing support
└── version/           # Version info
```

### apiserver
```
k8s.io/apiserver/pkg/
├── admission/         # Admission control
├── authentication/    # Authentication
├── authorization/     # Authorization
├── endpoints/         # API endpoints, discovery
├── registry/          # Generic registry
├── server/            # Generic API server
├── storage/           # Storage abstraction
└── util/              # Utilities
```

---

## Testing Patterns

### 1. **Fake Clients**
- Use `fake.Clientset` from `k8s.io/client-go/kubernetes/fake`
- Allows testing without real API server
- Supports tracking API calls

### 2. **Fake Informers**
- Manual event injection
- Controlled cache population
- Deterministic testing

### 3. **Fake Workqueues**
- Controlled queue operations
- Verify enqueueing logic
- Test rate limiting

### 4. **Integration Tests**
- Use `k8s.io/apiserver/pkg/server/testing`
- Start test API server
- Real etcd or memory storage

---

## Next Steps for Documentation

1. **Create Document 01**: Overview and Introduction ✅
2. **Create Documents 02-04**: client-go library
3. **Create Documents 05-07**: apimachinery library
4. **Create Documents 08-09**: component-base library
5. **Create Documents 10-12**: apiserver library
6. **Create Document 13**: Integration patterns

---

## Key References

### Official Documentation
- [client-go documentation](https://github.com/kubernetes/client-go)
- [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- [API Machinery](https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/api-machinery.md)

### Enhancement Proposals (KEPs)
- [Kubernetes Enhancement Proposal (KEP) process](https://github.com/kubernetes/enhancements)

### Source Code
- [k8s.io/client-go](https://github.com/kubernetes/client-go)
- [k8s.io/apimachinery](https://github.com/kubernetes/apimachinery)
- [k8s.io/component-base](https://github.com/kubernetes/component-base)
- [k8s.io/apiserver](https://github.com/kubernetes/apiserver)

---

**Last Updated**: 2025-10-20
**Maintainers**: Kubernetes Shared Libraries Architecture Analysis Team
**Feedback**: Please file issues for corrections or improvements
