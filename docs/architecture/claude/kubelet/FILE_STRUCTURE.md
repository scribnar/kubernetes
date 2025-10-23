# Kubelet Source File Structure and Organization

## Directory Structure

```
kubernetes/
├── cmd/kubelet/                          # Binary entry point
│   ├── kubelet.go                        # Main entry (40 lines)
│   └── app/
│       ├── server.go                     # Server initialization (56KB)
│       ├── options/                      # Kubelet flags and options
│       └── plugins.go                    # Plugin management
│
└── pkg/kubelet/                          # Main kubelet package (87 directories)
    ├── kubelet.go                        # Core Kubelet struct (137KB)
    ├── kubelet_pods.go                   # Pod sync logic (109KB)
    ├── kubelet_node_status.go            # Node status updates (30KB)
    ├── pod_workers.go                    # Pod worker implementation (77KB)
    │
    ├── container/                        # Container runtime abstractions
    │   ├── runtime.go                    # Runtime interface (30KB)
    │   ├── runtime_cache.go              # Runtime state caching
    │   ├── sync_result.go                # Pod sync results
    │   └── helpers.go                    # Container utilities
    │
    ├── kuberuntime/                      # CRI (Container Runtime Interface)
    │   ├── kuberuntime_manager.go        # Main CRI manager (82KB)
    │   ├── kuberuntime_container.go      # Container operations (60KB)
    │   ├── kuberuntime_sandbox.go        # Pod sandbox management (13KB)
    │   ├── kuberuntime_image.go          # Image operations
    │   ├── kuberuntime_gc.go             # Container garbage collection (15KB)
    │   ├── labels.go                     # Container labels
    │   ├── helpers.go                    # Helper functions
    │   ├── security_context.go           # Security enforcement
    │   └── instrumented_services.go      # CRI instrumentation
    │
    ├── pleg/                             # Pod Lifecycle Event Generator
    │   ├── pleg.go                       # PLEG interfaces (1KB)
    │   ├── generic.go                    # GenericPLEG implementation (22KB)
    │   ├── evented.go                    # EventedPLEG implementation (16KB)
    │   ├── generic_test.go               # GenericPLEG tests (33KB)
    │   └── evented_test.go               # EventedPLEG tests
    │
    ├── cm/                               # Container Manager (resource management)
    │   ├── container_manager.go          # CM interface
    │   ├── container_manager_linux.go    # Linux CM implementation (38KB)
    │   ├── container_manager_windows.go  # Windows CM implementation
    │   ├── cgroup_manager_linux.go       # Cgroup operations (15KB)
    │   ├── cgroup_v1_manager_linux.go    # Cgroup v1
    │   ├── cgroup_v2_manager_linux.go    # Cgroup v2
    │   │
    │   ├── cpumanager/                   # CPU allocation manager
    │   │   ├── cpu_manager.go            # CPU manager (19KB)
    │   │   ├── policy.go                 # Policy interface
    │   │   ├── policy_none.go            # No-op policy
    │   │   ├── policy_static.go          # Static CPU pinning (34KB)
    │   │   ├── policy_options.go         # Policy options
    │   │   ├── topology/                 # CPU topology awareness
    │   │   └── state/                    # CPU state tracking
    │   │
    │   ├── memorymanager/                # Memory allocation manager
    │   │   ├── memory_manager.go         # Memory manager interface
    │   │   ├── policy/                   # Allocation policies
    │   │   └── state/                    # Memory state
    │   │
    │   ├── devicemanager/                # Device plugin manager
    │   │   ├── manager.go                # Device manager (49KB)
    │   │   ├── pod_devices.go            # Pod device tracking (15KB)
    │   │   ├── endpoint.go               # Device plugin endpoint
    │   │   ├── plugin/                   # Device plugin interface
    │   │   ├── checkpoint/               # Device state checkpoint
    │   │   └── topology_hints.go         # Device topology hints
    │   │
    │   ├── topologymanager/              # Topology coordination
    │   │   ├── topology_manager.go       # Topology manager
    │   │   ├── socket_admitter.go        # Socket-based admission
    │   │   ├── numa_based_manager.go     # NUMA policies
    │   │   └── policies/                 # Topology policies
    │   │
    │   ├── dra/                          # Dynamic Resource Allocation
    │   │   ├── manager.go                # DRA manager
    │   │   └── kubelet_resource_manager.go
    │   │
    │   ├── admission/                    # Container admission
    │   ├── containermap/                 # Container tracking
    │   ├── resourceupdates/              # Resource update handling
    │   ├── helpers_linux.go              # Linux helpers
    │   └── util/                         # Utilities
    │
    ├── volumemanager/                    # Volume management
    │   ├── volume_manager.go             # Volume manager interface (24KB)
    │   ├── volume_manager_test.go        # Volume manager tests
    │   ├── populator/                    # Desired state populator
    │   │   ├── desired_state_of_world_populator.go
    │   │   └── *_test.go
    │   ├── reconciler/                   # Actual state reconciler
    │   │   ├── reconciler.go             # Volume reconciliation
    │   │   ├── reconstruct_common_linux.go
    │   │   └── *_test.go
    │   ├── cache/                        # Volume state cache
    │   │   ├── actual_state_of_world.go  # Actual volumes
    │   │   ├── desired_state_of_world.go # Desired volumes
    │   │   ├── operation_executor.go     # Volume operations
    │   │   └── *_test.go
    │   └── metrics/                      # Volume metrics
    │
    ├── status/                           # Pod status management
    │   ├── status_manager.go             # Status manager (53KB)
    │   ├── generate.go                   # Status generation (12KB)
    │   ├── generate_test.go              # Generation tests
    │   └── testing/                      # Test utilities
    │
    ├── prober/                           # Health probe management
    │   ├── prober_manager.go             # Probe manager (11KB)
    │   ├── prober.go                     # Prober implementation (10KB)
    │   ├── worker.go                     # Probe worker (11KB)
    │   ├── common_test.go                # Common tests
    │   ├── results/                      # Probe result tracking
    │   └── testing/                      # Test utilities
    │
    ├── eviction/                         # Resource eviction management
    │   ├── eviction_manager.go           # Eviction manager (25KB)
    │   ├── helpers.go                    # Eviction helpers (56KB)
    │   ├── helpers_test.go               # Helper tests (127KB)
    │   ├── types.go                      # Eviction types
    │   ├── api/                          # Eviction API types
    │   ├── threshold_notifier_linux.go   # Threshold monitoring
    │   ├── memory_threshold_notifier.go  # Memory threshold
    │   └── defaults_*.go                 # Platform defaults
    │
    ├── images/                           # Image management
    │   ├── image_gc_manager.go           # Image GC
    │   ├── image_gc_manager_test.go      # Image GC tests
    │   ├── types.go                      # Image types
    │   └── policy/                       # Image policy
    │
    ├── lifecycle/                        # Pod lifecycle
    │   ├── handlers.go                   # Lifecycle handlers (8KB)
    │   ├── predicate.go                  # Pod admission predicates (15KB)
    │   ├── predicates_test.go            # Predicate tests
    │   └── features_*.go                 # Platform-specific features
    │
    ├── config/                           # Pod configuration
    │   ├── pod_config.go                 # Pod config aggregation
    │   ├── sources.go                    # Config sources
    │   ├── file.go                       # File-based config
    │   ├── http.go                       # HTTP-based config
    │   └── apiserver.go                  # API server config
    │
    ├── server/                           # HTTP server
    │   ├── server.go                     # Main HTTP server
    │   ├── routes.go                     # HTTP routes
    │   ├── auth.go                       # Authentication
    │   ├── stats/                        # Stats endpoints
    │   └── metrics/                      # Metrics endpoints
    │
    ├── metrics/                          # Kubelet metrics
    │   ├── metrics.go                    # Metrics registration
    │   ├── collectors/                   # Prometheus collectors
    │   └── server_metrics.go             # Server metrics
    │
    ├── stats/                            # Pod/node statistics
    │   ├── cadvisor_stats_provider.go    # cAdvisor-based stats
    │   ├── cri_stats_provider.go         # CRI-based stats
    │   ├── host_stats_provider.go        # Host stats
    │   ├── resource_analyzer.go          # Resource analysis
    │   ├── pidlimit/                     # PID limit tracking
    │   └── server/                       # Stats server
    │
    ├── types/                            # Kubelet types
    │   ├── types.go                      # Core types
    │   ├── operation_executor_types.go   # Operation executor types
    │   └── pod_update_types.go           # Pod update types
    │
    ├── util/                             # Utilities
    │   ├── cache.go                      # Caching utilities
    │   ├── manager/                      # Manager utilities
    │   ├── queue/                        # Queue implementations
    │   ├── sliceutils/                   # Slice utilities
    │   ├── ioutils/                      # I/O utilities
    │   ├── handlers.go                   # HTTP handlers
    │   └── pod_startup_latency_tracker.go # Latency tracking
    │
    ├── qos/                              # QoS management
    │   └── manager_linux.go              # QoS manager
    │
    ├── userns/                           # User namespace support
    │   └── manager.go                    # User namespace manager
    │
    ├── nodeshutdown/                     # Node shutdown handling
    │   ├── manager.go                    # Shutdown manager
    │   ├── types.go                      # Shutdown types
    │   └── handlers.go                   # Shutdown handlers
    │
    ├── preemption/                       # Pod preemption
    │   └── preemption.go                 # Preemption logic
    │
    ├── watchdog/                         # Systemd watchdog
    │   └── types.go                      # Watchdog types
    │
    ├── oom/                              # Out-of-memory handling
    │   ├── oom_watcher.go                # OOM monitoring
    │   └── oom_watcher_linux.go          # Linux OOM
    │
    ├── pluginmanager/                    # Device plugin manager
    │   ├── pluginmanager.go              # Plugin manager
    │   └── cache/                        # Plugin cache
    │
    ├── secret/                           # Secret management
    │   ├── secret_manager.go             # Secret manager
    │   └── provider.go                   # Secret provider
    │
    ├── configmap/                        # ConfigMap management
    │   ├── configmap_manager.go          # ConfigMap manager
    │   └── provider.go                   # ConfigMap provider
    │
    ├── clustertrustbundle/               # ClusterTrustBundle support
    │   ├── manager.go                    # Manager interface
    │   └── lazy_informer_manager.go      # Informer-based manager
    │
    ├── token/                            # Service account token
    │   ├── manager.go                    # Token manager
    │   └── cache.go                      # Token cache
    │
    ├── podcertificate/                   # Pod certificate requests
    │   ├── manager.go                    # Certificate manager
    │   └── issuing_manager.go            # Issuing manager
    │
    ├── runtimeclass/                     # RuntimeClass support
    │   └── runtimeclass_manager.go       # RuntimeClass manager
    │
    ├── certificate/                      # Certificate management
    │   ├── certificate_manager.go        # Cert manager interface
    │   ├── bootstrap/                    # Bootstrap certificates
    │   └── manager.go                    # Cert file manager
    │
    ├── cadvisor/                         # cAdvisor integration
    │   ├── cadvisor.go                   # cAdvisor interface
    │   ├── cadvisor_linux.go             # Linux implementation
    │   └── types.go                      # cAdvisor types
    │
    ├── allocation/                       # Resource allocation
    │   ├── manager.go                    # Allocation manager
    │   ├── handler.go                    # Allocation handler
    │   └── state/                        # Allocation state
    │
    ├── network/                          # Network management
    │   ├── dns/                          # DNS configuration
    │   │   ├── dns.go                    # DNS manager
    │   │   └── config_reader.go          # DNS config reading
    │   └── hostutil/                     # Host networking
    │
    ├── logs/                             # Container log management
    │   ├── container_log_manager.go      # Log manager
    │   ├── log_rotation.go               # Log rotation
    │   └── filesystem_handler.go         # Filesystem handling
    │
    ├── nodestatus/                       # Node status
    │   └── status.go                     # Node status builder
    │
    ├── sysctl/                           # Sysctl management
    │   ├── sysctl.go                     # Sysctl interface
    │   └── types.go                      # Sysctl types
    │
    ├── winstats/                         # Windows statistics
    │   └── winstats.go                   # Windows stats
    │
    ├── checkpointmanager/                # Checkpoint management
    │   └── checkpoint_manager.go         # Checkpoint manager
    │
    ├── apis/                             # Kubelet API types
    │   └── config/                       # Kubelet configuration
    │       ├── v1beta1/                  # Configuration v1beta1
    │       └── validation/               # Configuration validation
    │
    └── test files (*_test.go)            # Extensive test coverage

```

## Key File Purposes

### Entry Points (40-60 lines)
- `cmd/kubelet/kubelet.go` - Binary main function
- `cmd/kubelet/app/server.go` - Server initialization (56KB comprehensive)

### Core Kubelet (137-241 KB)
- `pkg/kubelet/kubelet.go` - Main Kubelet struct, initialization
- `pkg/kubelet/kubelet_pods.go` - Pod sync, creation, deletion logic
- `pkg/kubelet/pod_workers.go` - Pod worker state machine

### Container Runtime (140+ KB)
- `pkg/kubelet/container/runtime.go` - Runtime interface definition
- `pkg/kubelet/kuberuntime/kuberuntime_manager.go` - CRI implementation
- `pkg/kubelet/kuberuntime/kuberuntime_container.go` - Container operations

### Pod Lifecycle (38+ KB)
- `pkg/kubelet/pleg/pleg.go` - PLEG interface
- `pkg/kubelet/pleg/generic.go` - Poll-based PLEG
- `pkg/kubelet/pleg/evented.go` - Event-driven PLEG

### Resource Management (200+ KB)
- `pkg/kubelet/cm/container_manager_linux.go` - Cgroup management
- `pkg/kubelet/cm/cpumanager/cpu_manager.go` - CPU pinning
- `pkg/kubelet/cm/devicemanager/manager.go` - Device plugins

### Volume Management (100+ KB)
- `pkg/kubelet/volumemanager/volume_manager.go` - Volume interface
- `pkg/kubelet/volumemanager/populator/` - Desired volumes
- `pkg/kubelet/volumemanager/reconciler/` - Volume operations

### Pod Status (65+ KB)
- `pkg/kubelet/status/status_manager.go` - Pod status updates
- `pkg/kubelet/status/generate.go` - Status generation logic

### Health Monitoring (40+ KB)
- `pkg/kubelet/prober/prober_manager.go` - Probe orchestration
- `pkg/kubelet/prober/worker.go` - Individual probe execution

### Resource Pressure (210+ KB)
- `pkg/kubelet/eviction/eviction_manager.go` - Eviction policy
- `pkg/kubelet/eviction/helpers.go` - Eviction helpers

## Component Interaction Map

```
           ┌─────────────────────────────────┐
           │    Kubelet Main (kubelet.go)    │
           │    - syncLoop()                 │
           │    - pod lifecycle              │
           └────────────┬────────────────────┘
                        │
        ┌───────────────┼───────────────┬─────────────┬─────────────┐
        │               │               │             │             │
        v               v               v             v             v
    ┌────────┐  ┌──────────────┐  ┌──────────┐  ┌───────────┐  ┌────────────┐
    │PodMgr  │  │Pod Workers   │  │StatusMgr │  │ProbeMgr   │  │EvictionMgr │
    │        │  │  - SyncPod   │  │          │  │           │  │            │
    │        │  │  - Terminating   │          │  │           │  │            │
    │        │  │  - Terminated    │          │  │           │  │            │
    └────────┘  └──────────────┘  └──────────┘  └───────────┘  └────────────┘
        │               │               │             │             │
        │        ┌──────┴─────────┐     │             │             │
        │        │                │     │             │             │
        v        v                v     v             v             v
    ┌─────────────────────────────────────────────────────────────────────┐
    │              Container Runtime Integration (CRI)                     │
    │  ┌────────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
    │  │  PLEG              │  │  RuntimeManager  │  │ImageManager    │  │
    │  │  - GenericPLEG     │  │  - SyncPod()     │  │- PullImage()   │  │
    │  │  - EventedPLEG     │  │  - KillPod()     │  │- GarbageCollect│  │
    │  └────────────────────┘  └──────────────────┘  └────────────────┘  │
    └─────────────────────────────────────────────────────────────────────┘
        │        │                │     │             │             │
        v        v                v     v             v             v
    ┌─────────────────────────────────────────────────────────────────────┐
    │               Resource Management (CM)                               │
    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
    │  │CPU Manager   │  │Memory Manager│  │Device/Topology Managers │  │
    │  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
    └─────────────────────────────────────────────────────────────────────┘
        │                                                                   │
        v                                                                   v
    ┌────────────────────────┐                          ┌──────────────────┐
    │  Volume Manager        │                          │  Node Status     │
    │  - Attach volumes      │                          │  - Ready cond.   │
    │  - Mount volumes       │                          │  - Capacity      │
    │  - Unmount volumes     │                          │  - Images        │
    └────────────────────────┘                          └──────────────────┘
        │
        v
    ┌────────────────────────┐
    │  Volume Plugins        │
    │  - In-tree (local)     │
    │  - CSI plugins         │
    └────────────────────────┘
```

## Test Coverage Organization

- `*_test.go` files parallel source files
- Major test files:
  - `kubelet_test.go` (154 KB)
  - `kubelet_pods_test.go` (241 KB)
  - `pod_workers_test.go` (77 KB)
  - `kuberuntime_manager_test.go` (143 KB)
  - `eviction_manager_test.go` (120 KB)
  - `helpers_test.go` (127 KB in eviction)
  - `pleg/generic_test.go` (33 KB)

## Code Statistics

- **Total Kubelet Package**: ~4000+ Go files across 87 directories
- **Main Files**: 100+ core implementation files
- **Test Files**: 150+ test files
- **Total Lines**: ~1.5M+ lines of code (including tests)

### Largest Components by File Count
1. `cm/` - 47 files (resource management)
2. `kuberuntime/` - 45 files (CRI implementation)
3. `volumemanager/` - 30+ files (volume operations)
4. `eviction/` - 22 files (eviction management)
5. `status/` - Multiple test files

## Key Patterns in File Organization

1. **Interface Definition**: Main interface typically in short file
   - Example: `runtime.go` defines Runtime interface
   
2. **Implementation Files**: Separate per platform
   - `_linux.go`, `_windows.go`, `_unsupported.go`

3. **Test Files**: Extensive alongside source
   - `file.go` paired with `file_test.go`

4. **Sub-packages**: Logical grouping
   - `cm/cpumanager/`, `cm/devicemanager/`, etc.

5. **Helper Packages**: Utility in separate dirs
   - `util/queue/`, `util/cache/`, etc.
