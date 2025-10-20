# Kubernetes Scheduler Architecture: The Scheduling Framework

This document describes the scheduling framework, a pluggable architecture for the Kubernetes scheduler. The framework is defined in `pkg/scheduler/framework/interface.go`.

## 1. Plugin-Based Architecture

The scheduling framework is designed to be highly extensible. It provides a set of extension points where custom logic can be injected in the form of plugins. This allows for fine-grained control over the scheduling process without modifying the core scheduler code.

## 2. Extension Points

The framework defines the following extension points, each corresponding to a specific phase of the scheduling cycle:

| Extension Point      | Description                                                                                                                            | Interface                                    |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| **PreEnqueue**       | Called before a pod is added to the scheduling queue. Can be used to preemptively reject a pod.                                        | `PreEnqueuePlugin`                           |
| **QueueSort**        | Sorts the pods in the scheduling queue. Only one QueueSort plugin can be active at a time.                                             | `QueueSortPlugin`                            |
| **PreFilter**        | Called at the beginning of the scheduling cycle. Can be used to filter out nodes before the main filtering logic.                      | `PreFilterPlugin`                            |
| **Filter**           | Filters out nodes that cannot run the pod.                                                                                             | `FilterPlugin`                               |
| **PostFilter**       | Called after the filtering phase if the pod is determined to be unschedulable. Can be used for preemption or other recovery mechanisms. | `PostFilterPlugin`                           |
| **PreScore**         | Called before the scoring phase. Can be used for pre-processing.                                                                       | `PreScorePlugin`                             |
| **Score**            | Ranks the feasible nodes by assigning a score to each one.                                                                             | `ScorePlugin`                                |
| **Reserve**          | An informational plugin that can update the scheduler's state before a pod is bound.                                                   | `ReservePlugin`                              |
| **Permit**           | Can prevent or delay the binding of a pod.                                                                                             | `PermitPlugin`                               |
| **PreBind**          | Called before a pod is bound. Can be used for any necessary setup.                                                                     | `PreBindPlugin`                              |
| **Bind**             | Binds the pod to a node.                                                                                                               | `BindPlugin`                                 |
| **PostBind**         | Called after a pod has been successfully bound. Can be used for cleanup.                                                               | `PostBindPlugin`                             |

### 2.1. EnqueueExtensions

In addition to the main extension points, the framework also provides `EnqueueExtensions`. This optional interface can be implemented by plugins to provide hints to the scheduling queue about when to re-enqueue a pod that has been deemed unschedulable.

## 3. Plugin Registration

Plugins are registered with the scheduler through the scheduler configuration. Each profile can have its own set of enabled plugins, allowing for different scheduling policies to be used for different workloads.

This flexible, plugin-based architecture is what makes the Kubernetes scheduler so powerful and adaptable to a wide range of scheduling needs.
