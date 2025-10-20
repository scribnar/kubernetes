# Scheduler Requirements Spec

Scope and goals
- Assign unscheduled Pods to Nodes based on constraints and scoring; write Bindings to the API server.
- Support multiple profiles (`pkg/scheduler/profile`), out-of-tree/extender integrations, and feature-gated behavior.
- Remain eventually consistent with cluster state using shared informers and an internal cache.

Correctness guarantees
- No placement on Nodes that fail mandatory filters (Node affinity/taints, resource fits, port conflicts).
- Single binding per Pod; use assume+bind to preserve throughput while tolerating API latency.
- Preemption may nominate a Node when necessary to satisfy priority and resource constraints.

Performance characteristics
- Parallel filtering/scoring (`WithParallelism`) and adaptive node sampling (`percentageOfNodesToScore`).
- Priority-aware queueing with backoff and queueing hints to reduce needless retries.

Reliability and operability
- Leader election, healthz/livez/readyz, metrics and profiling (`cmd/kube-scheduler/app/server.go`).
- Safe fallback on informer resyncs; cache snapshot guards against partial views.

Out of scope
- Node admission/runtime (kubelet), eviction policies (handled by controllers), or storage/CSI placement decisions outside scheduler interfaces.

