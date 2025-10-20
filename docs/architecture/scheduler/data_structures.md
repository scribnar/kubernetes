# Kubernetes Scheduler Architecture: Key Data Structures

This document describes the key data structures used by the Kubernetes scheduler to make scheduling decisions.

## 1. NodeInfo (`pkg/scheduler/framework/types.go`)

`NodeInfo` is a struct that holds aggregated information about a node. It is a key data structure used in the filtering and scoring phases of the scheduling cycle. The scheduler maintains a `NodeInfo` object for each node in the cluster, which is kept up-to-date through informers.

### Key Fields:

*   **node**: A pointer to the `v1.Node` object.
*   **Pods**: A list of `PodInfo` objects for all pods running on the node.
*   **Requested**: The total amount of resources requested by all pods on the node.
*   **Allocatable**: The total amount of allocatable resources on the node.
*   **UsedPorts**: A map of ports that are in use on the node.
*   **ImageStates**: A map of image states, which provides information about the images present on the node.

By maintaining this aggregated information, the scheduler can quickly access the necessary data to evaluate a node's fitness for a given pod.

## 2. CycleState (`pkg/scheduler/framework/cycle_state.go`)

`CycleState` is a struct that provides a mechanism for plugins to store and retrieve arbitrary data during a single scheduling cycle. It is essentially a key-value store that is passed to all plugins at each extension point.

### Purpose:

*   **State Sharing**: `CycleState` allows plugins to share data with each other. For example, a `PreFilter` plugin could perform some expensive computation and store the result in `CycleState` to be used by a `Filter` plugin later in the cycle.
*   **Per-Cycle Cache**: It acts as a cache for the duration of a single scheduling attempt, avoiding the need to recompute data at different stages of the scheduling cycle.

`CycleState` is thread-safe and is designed for a "write once, read many" pattern, which is typical for scheduling plugins.

## 3. PriorityQueue (`pkg/scheduler/internal/queue/scheduling_queue.go`)

The `PriorityQueue` is the main scheduling queue that holds all pods waiting to be scheduled. It is a complex data structure that consists of three sub-queues:

*   **activeQ**: A heap that holds pods that are ready to be scheduled. The pod with the highest priority is at the head of the heap.
*   **backoffQ**: A heap that holds pods that have failed scheduling and are in a backoff period. Pods are moved to the `activeQ` once their backoff period is complete.
*   **unschedulablePods**: A map that holds pods that have been deemed unschedulable. These pods are moved to the `activeQ` or `backoffQ` when a cluster event occurs that might make them schedulable.

This multi-queue design allows the scheduler to efficiently manage pods in different states and prioritize them for scheduling.
