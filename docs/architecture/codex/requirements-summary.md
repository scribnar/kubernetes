# Quick Requirements Summary

- Placement correctness: obey filters, priorities, and quotas; one bind per Pod.
- Throughput: assume-before-bind, parallel Filter/Score, adaptive node sampling.
- Resilience: informer-driven cache; safe on relist; queue backoff and hints.
- Extensibility: framework plugins, extenders, multiple profiles.
- Operability: leader election, health/ready endpoints, metrics, profiling.

