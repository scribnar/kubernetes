# Kubernetes Scheduler Architecture Documentation

**Version**: 1.0
**Status**: Comprehensive Architecture Analysis
**Last Updated**: 2025-10-20

---

## Overview

This directory contains comprehensive architecture documentation for the Kubernetes Scheduler, covering all aspects from high-level design to low-level implementation details. The documentation is organized into 20 detailed documents that progressively explore the scheduler's architecture, implementation, and operational characteristics.

## Documentation Structure

### Part I: High-Level Architecture (Documents 1-3)

#### ✅ [01 - Overview and Introduction](./01-overview-and-introduction.md)
**Status**: Complete

Executive summary of the Kubernetes Scheduler including:
- System overview and key statistics (208 Go files, 6 major subsystems, 25+ plugins)
- Core responsibilities (pod placement, priority/preemption, resource management)
- System context and role in Kubernetes architecture
- Architectural principles (separation of concerns, optimistic concurrency, extensibility)
- Technology stack and key dependencies
- Design decisions and trade-offs

**Key Diagrams**:
- High-level scheduling loop flowchart
- Pod queue state machine
- System context diagram (C4 model)

---

#### ✅ [02 - System Architecture](./02-system-architecture.md)
**Status**: Complete

Detailed component architecture including:
- Six major subsystems with detailed breakdowns
- Component interaction patterns
- Data flow architecture
- Layered architecture view
- Thread and concurrency model with goroutine structure

**Key Components Covered**:
1. **Scheduler Core** - Orchestration and coordination
2. **Queue Management** - Three-tier queue system (activeQ, backoffQ, unschedulableQ)
3. **Cache System** - Assume/bind mechanism and snapshot generation
4. **Plugin Framework** - Extension points and plugin execution
5. **Event Handlers** - Cluster state change processing
6. **Metrics & Observability** - Monitoring and debugging

**Key Diagrams**:
- Component architecture (C4 Component diagram)
- Queue state transitions
- Cache class diagram
- Plugin framework structure
- Complete scheduling sequence diagram
- Data flow diagram
- Layered architecture
- Goroutine structure

---

#### ✅ [03 - Deployment and Runtime](./03-deployment-and-runtime.md)
**Status**: Complete

Deployment patterns and runtime initialization including:
- Standard and high-availability deployment architectures
- Leader election mechanism (Lease-based, 15s duration)
- Complete bootstrap sequence (9-step initialization)
- Configuration management and precedence
- Health checks and readiness probes
- Graceful shutdown sequence

**Key Diagrams**:
- HA deployment architecture
- Leader election flow
- Complete bootstrap sequence (23-step sequence diagram)
- Configuration loading priority
- Health check architecture
- Shutdown sequence

---

### Part II: Functional Specifications (Documents 4-8)

#### 📋 04 - Functional Requirements

**Planned Content**:
- Core scheduling functions and use cases
- Pod selection and filtering requirements
- Node scoring and ranking
- Priority and preemption functional requirements
- Resource allocation policies
- Failure handling and retry logic
- Edge cases and error scenarios

**Diagrams to Include**:
- Use case diagram for scheduling scenarios
- Sequence diagrams for each major use case
- Activity diagram for priority handling
- State diagram for pod scheduling states

---

#### 📋 05 - Scheduling Workflow

**Planned Content**:
- Complete scheduling cycle breakdown
- ScheduleOne() detailed workflow
- Scheduling cycle vs binding cycle separation
- Pod lifecycle through the scheduler
- Queue movement decision tree
- Event-driven requeuing logic
- Backoff and retry mechanisms

**Diagrams to Include**:
- Complete scheduling workflow (activity diagram)
- ScheduleOne() flow (detailed sequence)
- Pod lifecycle state machine
- Queue transition logic (state diagram)
- Filter/score parallel execution
- Binding cycle async flow

---

#### 📋 06 - Plugin Framework Specification

**Planned Content**:
- 12 extension points detailed specifications
- Plugin lifecycle and execution order
- Plugin configuration and registration
- Profile-based multi-scheduler support
- Plugin dependencies and interactions
- Error handling in plugin execution
- Plugin performance considerations

**Extension Points**:
1. PreEnqueue - Gate pods before queue entry
2. QueueSort - Determine scheduling order
3. PreFilter - Pre-processing and node reduction
4. Filter - Feasibility checking (predicates)
5. PostFilter - Preemption and failure handling
6. PreScore - Score preparation
7. Score - Node ranking
8. Reserve - Resource reservation
9. Permit - Final approval with wait capability
10. PreBind - Pre-binding operations
11. Bind - Actual pod binding
12. PostBind - Post-binding cleanup

**Diagrams to Include**:
- Extension point sequence
- Plugin execution flow
- Plugin lifecycle diagram
- Profile configuration structure
- Error propagation in plugin chain

---

#### 📋 07 - Queue Management Specification

**Planned Content**:
- Three-tier queue architecture detailed design
- activeQ heap implementation
- backoffQ with exponential backoff
- unschedulableQ event-driven optimization
- Nominator pattern for preemption
- Queueing hints system
- In-flight pod tracking
- Lock hierarchy and synchronization

**Diagrams to Include**:
- Queue architecture diagram
- Queue transition state machine
- Queueing hint decision tree
- Backoff algorithm flowchart
- Lock ordering diagram

---

#### 📋 08 - Event Handling Specification

**Planned Content**:
- Cluster event types and taxonomy
- Event handler implementations for each resource
- Event-driven pod requeuing logic
- Informer integration and watch patterns
- PreCheck functions for optimization
- Event handling latency considerations
- Event filtering and batching

**Event Types Covered**:
- Node events (Add, Update, Delete)
- Pod events (scheduled and unscheduled)
- PVC/PV events
- Service, StorageClass events
- CSINode events
- ResourceClaim/ResourceSlice events (DRA)

**Diagrams to Include**:
- Event flow sequence diagrams
- Event handling architecture
- Informer watch pattern
- Event-to-queue movement logic

---

### Part III: Technical Specifications (Documents 9-18)

#### 📋 09 - Bootstrapping and Initialization

**Planned Content**:
- Detailed bootstrap sequence (all 23 steps)
- Command creation and flag registration
- Configuration loading and merging
- Client creation and authentication
- Informer factory initialization
- Event handler registration
- Cache and queue initialization
- Framework and plugin instantiation
- Leader election setup

**Diagrams to Include**:
- Complete initialization sequence diagram
- Configuration precedence flowchart
- Component initialization order
- Informer synchronization flow

---

#### 📋 10 - Data Structures

**Planned Content**:
Complete analysis of all major data structures:

1. **Scheduler** struct - Main orchestrator
2. **PriorityQueue** - Queue implementation
3. **cacheImpl** - Cache with doubly-linked list
4. **frameworkImpl** - Plugin framework runtime
5. **CycleState** - Thread-safe state container
6. **QueuedPodInfo** - Pod with scheduling metadata
7. **NodeInfo** - Node resource tracking
8. **NodeToStatus** - Diagnosis information
9. **ScheduleResult** - Scheduling decision
10. **podState** - Cache pod tracking
11. **nodeInfoListItem** - Linked list node
12. **Snapshot** - Cache snapshot

**Diagrams to Include**:
- Class diagrams for all structures
- Memory layout diagrams
- Relationship diagrams
- Data structure evolution through scheduling

---

#### 📋 11 - Scheduling Algorithm Technical

**Planned Content**:
- schedulePod() implementation details
- findNodesThatFitPod() filtering algorithm
- Parallel filtering with worker pools
- Nominated node fast path
- prioritizeNodes() scoring algorithm
- Plugin score normalization
- selectHost() reservoir sampling
- Percentage-based node evaluation
- Extender integration points

**Algorithms Covered**:
- Filter parallelization: `chunk_size = sqrt(numNodes)`
- Adaptive node %: `50 - (numNodes/125), min 5%`
- Reservoir sampling for fairness
- Node selection with equal scores

**Diagrams to Include**:
- schedulePod() activity diagram
- Parallel filtering worker pool
- Scoring aggregation flow
- Selection algorithm flowchart

---

#### 📋 12 - Plugin Framework Runtime

**Planned Content**:
- frameworkImpl internals
- Plugin registry and factory pattern
- Plugin initialization sequence
- Dependency injection mechanism
- Extension point execution details
- Plugin instrumentation (10% sampling)
- Waiting pods coordination (Permit)
- MultiPoint plugin expansion
- Score weight calculation

**Diagrams to Include**:
- Framework initialization sequence
- Plugin execution sequence for each extension point
- Waiting pods coordination
- Metrics recording flow

---

#### 📋 13 - Cache Implementation

**Planned Content**:
- Cache architecture and design
- Assume mechanism internals
- Bind finish tracking
- Doubly-linked list optimization
- Snapshot generation algorithm
- Generation-based updates
- Node tree structure
- Image state caching
- Pod state lifecycle
- Cache expiry goroutine

**Diagrams to Include**:
- Cache class diagram
- Assume/bind state machine
- Snapshot update algorithm
- Doubly-linked list operations
- Lock synchronization points

---

#### 📋 14 - Queue Implementation

**Planned Content**:
- PriorityQueue internals
- Heap implementation (activeQ, backoffQ)
- Unschedulable pods map
- Nominator implementation
- Queueing hints evaluation
- Event-to-plugin mapping
- Lock hierarchy: `main > active > backoff > nominator`
- In-flight pod tracking
- Pop blocking mechanism
- Queue flush goroutines

**Diagrams to Include**:
- Queue class diagram
- Heap operations
- Lock acquisition order
- Queue flush timing diagram
- Queueing hint evaluation flow

---

#### 📋 15 - Concurrency and Synchronization

**Planned Content**:
- Parallel filter execution (16 workers default)
- Parallelizer implementation
- Chunk size calculation: `max(1, min(sqrt(n), n/parallelism))`
- Extender scoring with WaitGroup
- Mutex hierarchy and deadlock prevention
- Atomic operations for counters
- Channel-based coordination
- Context cancellation patterns
- Lock-free optimizations
- Race condition prevention

**Concurrency Patterns**:
1. Worker pool pattern (filtering/scoring)
2. Producer-consumer (queue)
3. Fan-out/fan-in (extenders)
4. Optimistic locking (assume/bind)

**Diagrams to Include**:
- Worker pool architecture
- Parallel execution sequence
- Lock hierarchy diagram
- Context cancellation flow
- Race condition scenarios and solutions

---

#### 📋 16 - Preemption Logic

**Planned Content**:
- Preemption architecture overview
- PostFilter extension point role
- Default preemption plugin
- Victim selection algorithm
- Priority-based selection
- PDB (PodDisruptionBudget) consideration
- Candidate node evaluation
- Nominated node tracking
- Async vs sync preemption
- PreEnqueue gate for preempting pods

**Diagrams to Include**:
- Preemption flow (activity diagram)
- Victim selection algorithm
- Candidate evaluation sequence
- Nominated node lifecycle

---

#### 📋 17 - Binding and API Integration

**Planned Content**:
- Binding cycle detailed flow
- Reserve/Unreserve plugin execution
- Permit waiting mechanism (max 15min timeout)
- PreBind plugin use cases
- Bind plugin precedence (extenders vs framework)
- PostBind cleanup
- API dispatcher for async calls
- API cacher for status updates
- Error handling and rollback
- Event recording

**Diagrams to Include**:
- Complete binding sequence
- Reserve/Unreserve flow
- Permit waiting coordination
- API call dispatch
- Error handling sequence

---

#### 📋 18 - Metrics and Observability

**Planned Content**:
- Metrics architecture
- MetricAsyncRecorder pattern
- Key metrics catalog:
  - Scheduling latency
  - Plugin execution duration
  - Queue depth
  - Event handling latency
  - Goroutine counts
- Sampling strategy (10% for plugins)
- Logging best practices
- Debug endpoints
- Cache dump functionality
- Queue inspection tools
- Profiling integration (pprof)

**Diagrams to Include**:
- Metrics collection architecture
- Metrics recording flow
- Debug endpoint structure

---

### Part IV: Appendices (Documents 19-20)

#### 📋 19 - Plugin Catalog

**Planned Content**:
Complete catalog of 25+ in-tree plugins:

**Sorting**:
- PrioritySort

**Filtering**:
- NodeName, NodePorts, NodeAffinity, NodeUnschedulable
- TaintToleration, NodeResourcesFit
- VolumeBinding, VolumeRestrictions, VolumeZone
- PodTopologySpread, InterPodAffinity
- NodeVolumeLimits (CSI, EBS, GCE, Azure)
- SchedulingGates
- DynamicResources

**Scoring**:
- NodeResourcesBalancedAllocation, NodeResourcesFit
- ImageLocality
- InterPodAffinity
- PodTopologySpread
- TaintToleration
- NodeAffinity

**Binding**:
- DefaultBinder

**Preemption**:
- DefaultPreemption

For each plugin:
- Purpose and use cases
- Extension points implemented
- Configuration options
- Performance characteristics

**Diagrams to Include**:
- Plugin interaction matrix
- Plugin dependency graph

---

#### 📋 20 - Glossary and References

**Planned Content**:

**Glossary**:
- Assume/Bind
- Backoff
- Cycle State
- Extender
- Framework
- Nominated Node
- Preemption
- Profile
- QueueingHint
- Snapshot

**File References**:
Complete mapping of concepts to source files with line numbers

**Key Functions Index**:
- ScheduleOne - schedule_one.go:66
- schedulePod - schedule_one.go:430
- AssumePod - cache/cache.go:369
- Pop - queue/scheduling_queue.go
- (and many more)

**External References**:
- KEPs (Kubernetes Enhancement Proposals)
- Design docs
- API specifications
- Related components

---

## Usage Guide

### For New Contributors
1. Start with **01-Overview** for system understanding
2. Read **02-System Architecture** for component overview
3. Deep dive into specific **Technical Specifications** as needed

### For Plugin Developers
1. **06-Plugin Framework Specification** - Required reading
2. **12-Plugin Framework Runtime** - Implementation details
3. **19-Plugin Catalog** - Examples and patterns

### For Operators/SREs
1. **03-Deployment and Runtime** - Deployment patterns
2. **18-Metrics and Observability** - Monitoring
3. **07-Queue Management** - Performance tuning

### For Architecture Reviews
1. **02-System Architecture** - Component design
2. **15-Concurrency and Synchronization** - Thread safety
3. **13-Cache** + **14-Queue** - Critical path performance

---

## Document Status

| Document | Status | Completeness |
|----------|--------|--------------|
| 01 - Overview and Introduction | ✅ Complete | 100% |
| 02 - System Architecture | ✅ Complete | 100% |
| 03 - Deployment and Runtime | ✅ Complete | 100% |
| 04 - Functional Requirements | 📋 Planned | Spec defined |
| 05 - Scheduling Workflow | 📋 Planned | Spec defined |
| 06 - Plugin Framework Spec | 📋 Planned | Spec defined |
| 07 - Queue Management Spec | 📋 Planned | Spec defined |
| 08 - Event Handling Spec | 📋 Planned | Spec defined |
| 09 - Bootstrapping | 📋 Planned | Spec defined |
| 10 - Data Structures | 📋 Planned | Spec defined |
| 11 - Scheduling Algorithm | 📋 Planned | Spec defined |
| 12 - Plugin Framework Runtime | 📋 Planned | Spec defined |
| 13 - Cache Implementation | 📋 Planned | Spec defined |
| 14 - Queue Implementation | 📋 Planned | Spec defined |
| 15 - Concurrency & Sync | 📋 Planned | Spec defined |
| 16 - Preemption Logic | 📋 Planned | Spec defined |
| 17 - Binding & API Integration | 📋 Planned | Spec defined |
| 18 - Metrics & Observability | 📋 Planned | Spec defined |
| 19 - Plugin Catalog | 📋 Planned | Spec defined |
| 20 - Glossary & References | 📋 Planned | Spec defined |

---

## Key Insights from Completed Analysis

### Architecture Highlights

1. **Sophisticated Concurrency Model**
   - Up to 16 parallel workers for filtering/scoring
   - Strict lock hierarchy prevents deadlocks
   - Optimistic concurrency via assume/bind

2. **Three-Tier Queue System**
   - activeQ: Immediate scheduling candidates
   - backoffQ: Exponential backoff (1s-10s)
   - unschedulableQ: Event-driven optimization

3. **Extensible Plugin Framework**
   - 12 well-defined extension points
   - 25+ in-tree plugins
   - Profile-based multi-scheduler support

4. **Performance Optimizations**
   - Adaptive node evaluation (50% - n/125)
   - Snapshot-based scheduling
   - Queueing hints for targeted requeuing
   - Generation-based incremental cache updates

5. **High Availability**
   - Lease-based leader election (15s duration)
   - Stateless design (can restart on any control plane node)
   - Graceful shutdown with 5s timeout

---

## Source Code Statistics

- **Total Files**: 208 Go source files
- **cmd/kube-scheduler**: 10 files
- **pkg/scheduler**: 198 files
- **Lines of Code**: ~50,000+ LOC (estimated)

### Key Directories

```
kubernetes/
├── cmd/kube-scheduler/           (10 files)
│   ├── app/
│   │   ├── options/              (5 files - CLI)
│   │   ├── config/               (1 file - config structs)
│   │   └── server.go             (main server logic)
│   └── scheduler.go              (entry point)
│
└── pkg/scheduler/                (198 files)
    ├── scheduler.go, schedule_one.go, eventhandlers.go
    ├── framework/                (interfaces and runtime)
    │   ├── interface.go
    │   ├── runtime/              (framework implementation)
    │   ├── plugins/              (25+ plugin directories)
    │   └── parallelize/
    ├── backend/
    │   ├── queue/                (12 files - queue implementation)
    │   ├── cache/                (11 files - cache implementation)
    │   └── api_dispatcher/       (async API calls)
    └── metrics/                  (metrics and observability)
```

---

## Contributing to This Documentation

This documentation is a living resource. To contribute:

1. **Corrections**: Submit PRs for any inaccuracies
2. **Additions**: Add missing diagrams or details
3. **Updates**: Keep in sync with scheduler code changes
4. **Clarity**: Improve explanations and add examples

---

## Related Resources

- [Kubernetes Scheduler Design](https://github.com/kubernetes/design-proposals-archive/blob/main/scheduling/scheduler.md)
- [Scheduler Framework KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/624-scheduling-framework)
- [Queueing Hints KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/4247-queueing-hints)
- [Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)

---

**Last Updated**: 2025-10-20
**Maintainers**: Kubernetes Scheduler Architecture Analysis Team
**Feedback**: Please file issues for corrections or improvements
