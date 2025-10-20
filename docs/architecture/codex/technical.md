# Scheduler Technical Spec

Key components and data structures
- `Scheduler` (`pkg/scheduler/scheduler.go`): holds `Cache`, `SchedulingQueue`, profiles (`framework.Framework`), extenders, and snapshot.
- Cache (`pkg/scheduler/backend/cache`): node/pod state with assume/finish/forget lifecycle; exposes `UpdateSnapshot` into `Snapshot`/`NodeInfo` for fast iteration.
- Queue (`pkg/scheduler/backend/queue`): `PriorityQueue` with `activeQ`, `backoffQ`, `unschedulablePods`. Core types: `QueuedPodInfo`, `nominator`, pre-enqueue checks, queueing hints.
- Framework (`pkg/scheduler/framework`): `Framework` orchestrates phases; `CycleState`, `PodsToActivate`, `APICacher/Dispatcher` for async API calls.
- Profiles (`pkg/scheduler/profile`): per-scheduler-name plugin stacks and queue sort func.
 - Plugins: in-tree at `pkg/scheduler/framework/plugins/*` (see `plugins-and-profiles.md`, `plugins-catalog.md`).

Scheduling pipeline details
- Snapshot: `Cache.UpdateSnapshot` → `nodeInfoSnapshot` for the cycle.
- Filtering: parallelized across nodes (`framework/parallelize`); short-circuits on first failure per node; optional nominated-node fast path.
- Scoring: PreScore/Score/NormalizeScore; merges extender scores; `selectHost` picks highest.
- Assume/Reserve/Permit: set Pod.NodeName in cache, run reserve hooks, optionally wait on Permit.
- Binding: PreBindPreFlight/PreBind/Bind (may update status/nominating info through APIDispatcher); PostBind for cleanup/signals.

Concurrency and synchronization
- Goroutines: main `sched.Run` loop calling `ScheduleOne`; async binding goroutine; queue background goroutines (backoff flusher, leftover sync);
  informer handlers run concurrently (add/update/delete Nodes/Pods) and push into cache/queue.
- Locks: `PriorityQueue.lock` protects queue state; follow lock order: `lock > activeQueue.lock > backoffQueue.lock > nominator.nLock`.
- Snapshots: immutable view per cycle avoids holding locks during heavy Filter/Score work.
- Wait/permit: coordination via Framework’s Permit/WaitOnPermit; cancellation with context on cycle end.

Key files to read
- Entrypoint/server: `cmd/kube-scheduler/app/server.go` (leader election, health, metrics), `cmd/kube-scheduler/scheduler.go`.
- Core loop: `pkg/scheduler/schedule_one.go`, `pkg/scheduler/scheduler.go`.
- Queue internals: `pkg/scheduler/backend/queue/scheduling_queue.go`.
- Cache internals: `pkg/scheduler/backend/cache/cache.go`, `snapshot.go`.
- Framework contracts: `pkg/scheduler/framework/interface.go`, `types.go`, `runtime/`.
 - Plugins registry and names: `pkg/scheduler/framework/plugins/registry.go`, `pkg/scheduler/framework/plugins/names/`.
 - Defaults and args: `pkg/scheduler/apis/config/v1/defaults.go`, `.../testing/defaults/defaults.go`, `pkg/scheduler/apis/config/types.go`.

Observability
- Metrics: scheduler algorithm latency, goroutine gauges, queue metrics; resource metrics under `/metrics/resources`.
- Debug: cache debugger (`backend/cache/debugger`), profiling endpoints if enabled, zpages when feature-gated.

Diagrams: see `diagrams/concurrency.mmd` and `diagrams/scheduling-sequence.mmd`.

Embedded concurrency overview
```mermaid
flowchart TB
  subgraph Informers
    IN[SharedInformerFactory] --> EH[Event Handlers]
  end
  subgraph Queue
    AQ[activeQ]
    BQ[backoffQ]
    UQ[unschedulable]
  end
  subgraph Scheduler
    SL[Schedule loop goroutine]
    BG[Binding goroutine]
  end
  subgraph Cache
    CC[Assume/Finish/Forget]
    SS[Snapshot]
  end

  EH -->|Add/Upd/Del| AQ
  EH -->|MoveAllToActiveOrBackoffQueue| AQ
  SL -->|Pop| AQ
  AQ -->|Flush| SL
  BQ -->|Timer| AQ
  SL --> CC
  CC --> SS
  SL --> BG

  note right of Scheduler
    Context cancellation and Permit/Wait
    coordinate end-of-cycle
  end note
```
