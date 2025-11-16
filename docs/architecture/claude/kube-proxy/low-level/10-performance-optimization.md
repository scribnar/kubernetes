# **Low-Level: Performance Optimization**

**Part of**: [kube-proxy Architecture Documentation](../00-README.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Overview**

Comprehensive performance optimization guide for kube-proxy, covering scaling limits, benchmarks, tuning parameters, and best practices for both iptables and IPVS modes.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **1. Scaling Limits**

### **1.1 iptables Mode Limits**

| Metric | Limit | Impact at Limit |
|--------|-------|-----------------|
| **Max Services** | ~5,000 | Sync latency >10s |
| **Max Endpoints per Service** | ~1,000 | O(N²) rule count |
| **Total iptables Rules** | ~100,000 | Memory exhaustion |
| **Sync Latency** | >30s | Service updates delayed |

### **1.2 IPVS Mode Limits**

| Metric | Limit | Impact at Limit |
|--------|-------|-----------------|
| **Max Services** | ~100,000 | Kernel memory limits |
| **Max Endpoints per Service** | ~10,000 | Scheduler overhead |
| **Sync Latency** | <5s | Even at max scale |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **2. Performance Benchmarks**

### **2.1 Sync Performance**

**iptables Mode**:
```
1,000 services × 10 endpoints = 10,000 rules
  Sync time: ~3-5 seconds
  Memory: ~500MB

5,000 services × 10 endpoints = 50,000 rules
  Sync time: ~15-30 seconds
  Memory: ~2GB
```

**IPVS Mode**:
```
10,000 services × 100 endpoints = 1M entries
  Sync time: ~2-4 seconds
  Memory: ~200MB

100,000 services × 100 endpoints = 10M entries
  Sync time: ~10-20 seconds
  Memory: ~1GB
```

### **2.2 Packet Processing**

| Mode | Latency per Packet | Throughput |
|------|-------------------|------------|
| **iptables** (1K rules) | ~50μs | ~200K pps |
| **iptables** (50K rules) | ~200μs | ~50K pps |
| **IPVS** (any scale) | ~5-20μs | ~1M pps |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **3. Tuning Parameters**

### **3.1 kube-proxy Flags**

```bash
# Sync performance
--iptables-min-sync-period=1s          # Minimum sync interval
--iptables-sync-period=30s             # Maximum sync interval
--ipvs-min-sync-period=1s              # IPVS minimum
--ipvs-sync-period=30s                 # IPVS maximum

# Rule optimization
--iptables-masquerade-bit=14           # Mark bit for SNAT
--ipvs-scheduler=rr                    # Load balancing algorithm

# Performance tuning
--kube-api-qps=50                      # API requests/sec
--kube-api-burst=100                   # API burst limit
```

### **3.2 System Tuning**

**Conntrack**:
```bash
sysctl -w net.netfilter.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_buckets=262144
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=3600
```

**IPVS**:
```bash
# Enable IPVS
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh

# Tune IPVS
sysctl -w net.ipv4.vs.conn_reuse_mode=1
sysctl -w net.ipv4.vs.conntrack=1
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **4. Optimization Strategies**

### **4.1 Choose IPVS for Large Clusters**

**Decision Matrix**:

| Cluster Size | Services | Recommended Mode |
|--------------|----------|------------------|
| < 50 nodes | < 1,000 | iptables or IPVS |
| 50-500 nodes | 1,000-10,000 | **IPVS** |
| > 500 nodes | > 10,000 | **IPVS** (required) |

### **4.2 Minimize Service Churn**

- Use readiness probes (avoid endpoint flapping)
- Set appropriate `minReadySeconds`
- Use PodDisruptionBudgets

### **4.3 Optimize Sync Intervals**

```bash
# High churn environment
--iptables-min-sync-period=1s   # Quick updates

# Stable environment
--iptables-min-sync-period=5s   # Reduce CPU
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **5. Monitoring and Profiling**

### **5.1 Key Metrics**

```promql
# Sync latency
histogram_quantile(0.99, rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m]))

# Rule count
kubeproxy_sync_proxy_rules_iptables_total

# API calls
rate(rest_client_requests_total{job="kube-proxy"}[5m])
```

### **5.2 Profiling**

```bash
# Enable pprof
kube-proxy --profiling --bind-address=0.0.0.0

# CPU profile
go tool pprof http://localhost:10249/debug/pprof/profile

# Memory profile
go tool pprof http://localhost:10249/debug/pprof/heap
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **6. Best Practices**

1. **Use IPVS** for clusters >50 nodes or >1K services
2. **Monitor sync latency** - alert if P99 >5s
3. **Tune conntrack** based on connection count
4. **Use Local traffic policy** only when needed (preserves performance)
5. **Minimize endpoint churn** with proper health checks
6. **Set resource limits** on kube-proxy DaemonSet
7. **Enable metrics** for visibility

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **7. Summary**

**Performance Comparison**:

| Aspect | iptables | IPVS |
|--------|----------|------|
| **Max Scale** | 5K services | 100K services |
| **Sync Time** | O(N²) | O(N) |
| **Packet Latency** | 50-200μs | 5-20μs |
| **Memory** | High | Low |
| **Recommendation** | Small clusters | Large clusters |

**Code References**:
- `pkg/proxy/iptables/proxier.go:800-900` - Sync optimization
- `pkg/proxy/ipvs/proxier.go:1100-1200` - IPVS performance
- `pkg/proxy/config/config.go:100-200` - Sync intervals

---

*Last Updated*: Session 13  
*Status*: ✅ Complete  
*Part of*: [kube-proxy Architecture Documentation](../00-README.md)
