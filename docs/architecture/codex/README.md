# Kubernetes Scheduler Architecture (Codex)

This folder explains the kube-scheduler in layers: requirements, functional flows, and technical internals. It is based on `cmd/kube-scheduler/` and `pkg/scheduler/` and the scheduler backend packages.

- Start with Requirements for scope and guarantees.
- Use Functional Spec to understand runtime behavior and flows.
- Dive into Technical Spec for data structures, concurrency, and key files.

Key code entry points
- `cmd/kube-scheduler/scheduler.go` and `cmd/kube-scheduler/app/server.go`
- `pkg/scheduler/scheduler.go`, `pkg/scheduler/schedule_one.go`
- Queues: `pkg/scheduler/backend/queue/`
- Cache & snapshot: `pkg/scheduler/backend/cache/`
- Framework: `pkg/scheduler/framework/` (+ plugins under `framework/plugins/`)

Diagrams
- High level: `diagrams/high-level.mmd`
- Scheduling sequence: `diagrams/scheduling-sequence.mmd`
- Queue activity: `diagrams/queue-activity.mmd`
- Preemption sequence: `diagrams/preemption-sequence.mmd`
- Concurrency overview: `diagrams/concurrency.mmd`

Deep dives
- Plugins & Profiles: `plugins-and-profiles.md` (with `diagrams/plugins-flow.mmd`)
- Plugins Catalog: `plugins-catalog.md`
- Requirements summary: `requirements-summary.md`
- Concurrency & Permit: `concurrency-permit.md` (with `diagrams/permit-sequence.mmd`)
