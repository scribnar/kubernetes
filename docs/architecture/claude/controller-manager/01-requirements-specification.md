# Kube-Controller-Manager Requirements Specification

**Document Version**: 1.0
**Last Updated**: 2025-10-21
**Status**: Draft

---

## 1. Executive Overview

The kube-controller-manager is a critical control plane component in Kubernetes responsible for running 50+ controller processes that regulate the state of the cluster. It embeds the core control loops that watch the shared state of the cluster through the API server and make changes to move the current state toward the desired state.

## 2. System Purpose

### 2.1 Primary Objectives

1. **State Reconciliation**: Continuously reconcile actual cluster state with desired state
2. **Resource Lifecycle Management**: Manage creation, update, and deletion of Kubernetes resources
3. **High Availability**: Support leader election for active-passive HA deployments
4. **Scalability**: Handle large clusters with thousands of nodes and hundreds of thousands of resources
5. **Extensibility**: Support feature gates and progressive rollout of new functionality

### 2.2 Design Goals

- **Decoupled Architecture**: Each controller operates independently with minimal dependencies
- **Event-Driven**: Use informers and work queues for efficient, reactive processing
- **Fault Tolerance**: Gracefully handle API server unavailability and network partitions
- **Performance**: Minimize API server load through shared informers and caching
- **Observability**: Expose metrics, health checks, and debugging endpoints

## 3. Functional Requirements

### 3.1 Core Capabilities

#### FR-1: Controller Management
- **FR-1.1**: Support registration and lifecycle management of 50+ controllers
- **FR-1.2**: Enable/disable controllers via configuration flags
- **FR-1.3**: Support controller-specific configuration options
- **FR-1.4**: Implement graceful startup with jittered initialization
- **FR-1.5**: Support graceful shutdown with configurable timeout

#### FR-2: Workload Management
- **FR-2.1**: Manage Deployments with rolling updates and rollback capabilities
- **FR-2.2**: Maintain ReplicaSets at desired replica counts
- **FR-2.3**: Ensure StatefulSets maintain ordered, unique pod identities
- **FR-2.4**: Manage DaemonSets to ensure one pod per node
- **FR-2.5**: Execute Jobs to completion with parallelism control
- **FR-2.6**: Schedule and execute CronJobs based on cron expressions
- **FR-2.7**: Maintain ReplicationControllers (legacy) for backward compatibility

#### FR-3: Node Management
- **FR-3.1**: Monitor node health and lifecycle states
- **FR-3.2**: Allocate pod CIDRs to nodes (IPAM)
- **FR-3.3**: Evict pods from unhealthy or tainted nodes
- **FR-3.4**: Handle device-specific taints for dynamic resource allocation
- **FR-3.5**: Manage node zones and regions

#### FR-4: Service Discovery
- **FR-4.1**: Maintain Endpoints for Services
- **FR-4.2**: Create and update EndpointSlices for scalable service discovery
- **FR-4.3**: Mirror Endpoints to EndpointSlices for compatibility
- **FR-4.4**: Manage ServiceCIDR allocations

#### FR-5: Storage Management
- **FR-5.1**: Bind PersistentVolumeClaims to PersistentVolumes
- **FR-5.2**: Attach and detach volumes from nodes
- **FR-5.3**: Expand PersistentVolumeClaims when requested
- **FR-5.4**: Manage ephemeral volumes for pods
- **FR-5.5**: Protect PVCs and PVs from premature deletion
- **FR-5.6**: Handle VolumeAttributesClass protection
- **FR-5.7**: Warn about SELinux policy conflicts
- **FR-5.8**: Manage ResourceClaims for dynamic resource allocation

#### FR-6: Resource Lifecycle
- **FR-6.1**: Clean up terminated namespaces
- **FR-6.2**: Garbage collect orphaned resources using owner references
- **FR-6.3**: Delete terminated pods based on threshold
- **FR-6.4**: Clean up expired nodes (TTL)
- **FR-6.5**: Delete finished Jobs after TTL expiration
- **FR-6.6**: Garbage collect old StorageVersions

#### FR-7: Security & Access Control
- **FR-7.1**: Create default ServiceAccounts in namespaces
- **FR-7.2**: Generate and mount ServiceAccount tokens
- **FR-7.3**: Sign and approve CertificateSigningRequests
- **FR-7.4**: Clean up approved/denied CSRs
- **FR-7.5**: Sign bootstrap tokens for node join
- **FR-7.6**: Clean up expired tokens
- **FR-7.7**: Publish root CA certificates to namespaces
- **FR-7.8**: Publish ClusterTrustBundles for kube-apiserver
- **FR-7.9**: Clean up legacy ServiceAccount tokens

#### FR-8: Policy Enforcement
- **FR-8.1**: Enforce ResourceQuotas in namespaces
- **FR-8.2**: Respect PodDisruptionBudgets during voluntary disruptions
- **FR-8.3**: Aggregate ClusterRoles based on labels
- **FR-8.4**: Update ValidatingAdmissionPolicy status

#### FR-9: Autoscaling
- **FR-9.1**: Scale workloads based on HorizontalPodAutoscaler metrics

### 3.2 Infrastructure Requirements

#### FR-10: Leader Election
- **FR-10.1**: Support lease-based leader election (default)
- **FR-10.2**: Support ConfigMap-based leader election (legacy)
- **FR-10.3**: Support Endpoints-based leader election (deprecated)
- **FR-10.4**: Enable leader migration between lock types
- **FR-10.5**: Participate in coordinated leader election

#### FR-11: Shared Infrastructure
- **FR-11.1**: Maintain shared informer factory for typed resources
- **FR-11.2**: Maintain metadata-only informer factory for generic controllers
- **FR-11.3**: Provide RESTMapper with periodic CRD discovery refresh
- **FR-11.4**: Support per-controller client builders
- **FR-11.5**: Broadcast and record events

#### FR-12: Observability
- **FR-12.1**: Expose Prometheus metrics for controllers
- **FR-12.2**: Provide health check endpoints (/healthz, /livez, /readyz)
- **FR-12.3**: Support per-controller debugging handlers
- **FR-12.4**: Expose configuration via /configz
- **FR-12.5**: Support profiling endpoints (when enabled)

## 4. Non-Functional Requirements

### 4.1 Performance

- **NFR-1**: Process 1000+ pod updates per second per controller
- **NFR-2**: Maintain sub-second reconciliation latency for high-priority resources
- **NFR-3**: Support clusters with 5000+ nodes
- **NFR-4**: Handle 150,000+ pods across the cluster
- **NFR-5**: Minimize API server load through efficient caching

### 4.2 Reliability

- **NFR-6**: Achieve 99.9% uptime in HA configurations
- **NFR-7**: Recover from API server unavailability within 30 seconds
- **NFR-8**: Handle network partitions gracefully without data corruption
- **NFR-9**: Support zero-downtime rolling upgrades
- **NFR-10**: Implement exponential backoff for transient errors

### 4.3 Scalability

- **NFR-11**: Support horizontal scaling via leader election (active-passive)
- **NFR-12**: Scale informer cache memory linearly with resource count
- **NFR-13**: Support configurable concurrency per controller
- **NFR-14**: Enable independent scaling of controllers (future: separate binaries)

### 4.4 Security

- **NFR-15**: Support TLS for secure serving
- **NFR-16**: Integrate with Kubernetes authentication (delegating auth)
- **NFR-17**: Integrate with Kubernetes authorization (RBAC)
- **NFR-18**: Support per-controller ServiceAccount credentials
- **NFR-19**: Never log or expose secrets in plain text

### 4.5 Maintainability

- **NFR-20**: Provide structured logging with contextual information
- **NFR-21**: Support feature gates for gradual rollout of changes
- **NFR-22**: Maintain backward compatibility for at least 2 Kubernetes minor versions
- **NFR-23**: Document all controller behaviors and edge cases
- **NFR-24**: Support runtime configuration updates where safe

### 4.6 Compatibility

- **NFR-25**: Support Kubernetes API compatibility skew (n-1, n, n+1)
- **NFR-26**: Maintain compatibility with in-tree cloud providers (deprecated but present)
- **NFR-27**: Support cloud-controller-manager migration path
- **NFR-28**: Handle CRDs discovered at runtime

## 5. Constraints

### 5.1 Technical Constraints

- **C-1**: Must run as a single process (not distributed)
- **C-2**: Requires connectivity to kube-apiserver
- **C-3**: Must use Kubernetes client-go libraries
- **C-4**: Limited to watch-based event processing (no polling)
- **C-5**: Memory constrained by informer cache size

### 5.2 Operational Constraints

- **C-6**: Typically deployed with 1-3 replicas (leader election)
- **C-7**: Should not store persistent state locally
- **C-8**: Must be stateless and recoverable from API server state
- **C-9**: Configuration changes require process restart

### 5.3 Deprecated Features

- **C-10**: In-tree cloud provider support removed in v1.31 (KEP-2395)
- **C-11**: Legacy ServiceAccount token auto-generation being phased out
- **C-12**: ConfigMap and Endpoints leader election locks deprecated

## 6. Dependencies

### 6.1 External Dependencies

- **Kube-APIServer**: All operations require API server availability
- **Etcd**: Indirectly via API server for persistent storage
- **Service Account Token Signing**: Requires valid signing keys

### 6.2 Internal Dependencies

- **Informer Framework**: client-go shared informers
- **Work Queue**: client-go rate-limiting work queues
- **Leader Election**: client-go/tools/leaderelection
- **Event Broadcasting**: client-go event recording
- **Metrics**: component-base Prometheus metrics

## 7. Controller-Specific Requirements

### 7.1 Special Handling Controllers

- **ServiceAccountToken Controller**: Must start first (FR-7.2)
- **GarbageCollector**: Requires GraphBuilder initialization before controller context

### 7.2 Feature-Gated Controllers (8 total)

1. **TaintEviction**: Requires `SeparateTaintEvictionController` feature gate
2. **DeviceTaintEviction**: Requires `DynamicResourceAllocation` + `DRADeviceTaints`
3. **ResourceClaim**: Requires `DynamicResourceAllocation`
4. **StorageVersionGC**: Requires `APIServerIdentity` + `StorageVersionAPI`
5. **VolumeAttributesClassProtection**: Requires `VolumeAttributesClass`
6. **ServiceCIDR**: Feature gated
7. **StorageVersionMigrator**: Feature gated
8. **SELinuxWarning**: Requires `SELinuxChangePolicy`, disabled by default

### 7.3 Cloud Provider Controllers (Disabled)

Since Kubernetes v1.31 (KEP-2395), these controllers are no longer functional in kube-controller-manager:

1. **Service LoadBalancer Controller**
2. **Node Route Controller**
3. **Cloud Node Lifecycle Controller**

These must be run in cloud-controller-manager instead.

## 8. Success Criteria

### 8.1 Functional Success

- All 50 controllers start successfully in default configuration
- Controllers reconcile resources within SLA latency bounds
- Failed reconciliations trigger appropriate retries with backoff
- Health checks accurately reflect controller operational status

### 8.2 Performance Success

- API server watch connections remain stable under load
- Informer cache memory usage stays within bounds
- Controller work queues drain at expected rates
- No reconciliation loops or infinite retries

### 8.3 Operational Success

- Upgrades complete without disrupting running workloads
- Leader election transitions complete within 30 seconds
- Metrics and logs provide actionable troubleshooting data
- Controllers recover from API server restarts automatically

## 9. Open Questions & Future Considerations

1. **Controller Decomposition**: Should controllers be split into separate binaries for independent scaling?
2. **Watch Scalability**: Can streaming lists (WatchList) replace traditional list-watch for better performance?
3. **Eventual Consistency**: Are there consistency guarantees that need strengthening?
4. **Cross-Controller Coordination**: Should there be explicit dependency management between controllers?
5. **Cloud Provider Migration**: What is the long-term support plan for legacy cloud provider code?

## 10. Document References

- **Source Code Location**: `/cmd/kube-controller-manager/`, `/pkg/controller/`
- **Generic Framework**: `/staging/src/k8s.io/controller-manager/`
- **Controller Names**: `/cmd/kube-controller-manager/names/controller_names.go`
- **Main Entry Point**: `/cmd/kube-controller-manager/controller-manager.go:34`
- **Orchestration Logic**: `/cmd/kube-controller-manager/app/controllermanager.go:184`

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-10-21 | Architecture Analysis | Initial requirements specification |
