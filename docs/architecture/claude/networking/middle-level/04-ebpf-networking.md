# **eBPF Networking in Kubernetes**

**Deep Dive into eBPF-based CNI Implementations and Performance**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Table of Contents**

1. [Overview](#overview)
2. [eBPF Fundamentals](#ebpf-fundamentals)
3. [eBPF vs iptables](#ebpf-vs-iptables)
4. [Cilium eBPF Implementation](#cilium-ebpf-implementation)
5. [Calico eBPF Mode](#calico-ebpf-mode)
6. [XDP (eXpress Data Path)](#xdp-express-data-path)
7. [BPF Maps and Programs](#bpf-maps-and-programs)
8. [kube-proxy Replacement](#kube-proxy-replacement)
9. [Performance Analysis](#performance-analysis)
10. [Troubleshooting](#troubleshooting)
11. [Development Guide](#development-guide)
12. [Best Practices](#best-practices)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Overview**

### **What is eBPF?**

eBPF (extended Berkeley Packet Filter) is a revolutionary Linux kernel technology that allows running sandboxed programs in kernel space without changing kernel source code or loading kernel modules.

**Key Benefits for Networking:**
- **Performance**: Packet processing at wire speed with minimal overhead
- **Programmability**: Custom logic without kernel modules
- **Safety**: Verified programs cannot crash the kernel
- **Observability**: Deep network visibility
- **Flexibility**: Dynamic updates without downtime

### **eBPF in Kubernetes Networking**

```
Traditional iptables Path:
┌─────┐     ┌──────────┐     ┌─────────┐     ┌──────┐
│ NIC │────►│ iptables │────►│ Routing │────►│ App  │
└─────┘     │ (1000s   │     │         │     └──────┘
            │  rules)  │     └─────────┘
            └──────────┘
            O(n) complexity

eBPF Path:
┌─────┐     ┌──────────┐     ┌──────┐
│ NIC │────►│   eBPF   │────►│ App  │
└─────┘     │ (hash map│     └──────┘
            │  lookup) │
            └──────────┘
            O(1) complexity
```

### **eBPF Hook Points**

```
Linux Network Stack:
┌────────────────────────────────────────────────┐
│ Network Interface Card (NIC)                   │
└──────────────┬─────────────────────────────────┘
               │
               ▼
      ┌────────────────┐
      │  XDP (eBPF)    │ ◄─── Earliest hook, DMA buffer
      └────────┬───────┘
               │
               ▼
      ┌────────────────┐
      │  TC ingress    │ ◄─── After initial processing
      │    (eBPF)      │
      └────────┬───────┘
               │
               ▼
      ┌────────────────┐
      │ Netfilter/     │
      │ iptables       │
      └────────┬───────┘
               │
               ▼
      ┌────────────────┐
      │   Routing      │
      └────────┬───────┘
               │
               ▼
      ┌────────────────┐
      │  TC egress     │ ◄─── Before transmission
      │    (eBPF)      │
      └────────┬───────┘
               │
               ▼
      ┌────────────────┐
      │  Socket (eBPF) │ ◄─── sockmap/sockhash
      └────────┬───────┘
               │
               ▼
      ┌────────────────┐
      │    NIC TX      │
      └────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 eBPF Fundamentals**

### **eBPF Program Structure**

```c
// Example eBPF program for packet filtering
// Location: bpf/filter.c

#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <bpf/bpf_helpers.h>

// BPF map for storing allow list
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10000);
    __type(key, __u32);      // IP address
    __type(value, __u8);     // 1 = allow, 0 = deny
} ip_allowlist SEC(".maps");

SEC("xdp")
int xdp_filter(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;

    // Parse Ethernet header
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    // Only process IPv4
    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_PASS;

    // Parse IP header
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    // Lookup source IP in allowlist
    __u32 src_ip = ip->saddr;
    __u8 *allowed = bpf_map_lookup_elem(&ip_allowlist, &src_ip);

    if (allowed && *allowed == 1)
        return XDP_PASS;  // Allow packet
    else
        return XDP_DROP;  // Drop packet
}

char _license[] SEC("license") = "GPL";
```

### **BPF Map Types**

```c
// Hash map for key-value lookups
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 65536);
    __type(key, struct flow_key);
    __type(value, struct flow_stats);
} flow_map SEC(".maps");

// Array map for indexed access
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 256);
    __type(key, __u32);
    __type(value, __u64);
} counter_map SEC(".maps");

// LRU hash map (auto-eviction)
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, 10000);
    __type(key, __u32);
    __type(value, struct conn_info);
} conn_map SEC(".maps");

// Per-CPU array (no locking)
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, 256);
    __type(key, __u32);
    __type(value, __u64);
} percpu_stats SEC(".maps");

// Sockmap for socket redirection
struct {
    __uint(type, BPF_MAP_TYPE_SOCKMAP);
    __uint(max_entries, 65536);
    __type(key, struct sock_key);
    __type(value, __u32);
} sock_map SEC(".maps");
```

### **Loading eBPF Programs**

```go
// Go code to load and attach eBPF program
// Location: pkg/ebpf/loader.go

import (
    "github.com/cilium/ebpf"
    "github.com/cilium/ebpf/link"
)

func LoadXDPProgram(ifaceName string, progPath string) error {
    // Load compiled eBPF object
    spec, err := ebpf.LoadCollectionSpec(progPath)
    if err != nil {
        return err
    }

    // Create collection
    coll, err := ebpf.NewCollection(spec)
    if err != nil {
        return err
    }
    defer coll.Close()

    // Get XDP program
    prog := coll.Programs["xdp_filter"]
    if prog == nil {
        return fmt.Errorf("program xdp_filter not found")
    }

    // Get network interface
    iface, err := net.InterfaceByName(ifaceName)
    if err != nil {
        return err
    }

    // Attach XDP program to interface
    l, err := link.AttachXDP(link.XDPOptions{
        Program:   prog,
        Interface: iface.Index,
        Flags:     link.XDPGenericMode,  // or XDPDriverMode, XDPOffloadMode
    })
    if err != nil {
        return err
    }
    defer l.Close()

    return nil
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚖️ eBPF vs iptables**

### **Performance Comparison**

```
Rule Matching Performance:
┌─────────────────────────────────────────────────────────────┐
│  100 rules                                                  │
│  iptables:  5 µs   [▓▓▓▓▓░░░░░░░░░░░░░░░]                │
│  eBPF:      0.5 µs [▓░░░░░░░░░░░░░░░░░░░░]                │
├─────────────────────────────────────────────────────────────┤
│  1,000 rules                                                │
│  iptables:  50 µs  [▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░]                │
│  eBPF:      0.5 µs [▓░░░░░░░░░░░░░░░░░░░░]                │
├─────────────────────────────────────────────────────────────┤
│  10,000 rules                                               │
│  iptables:  500 µs [▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓]  (packet drop)  │
│  eBPF:      0.5 µs [▓░░░░░░░░░░░░░░░░░░░░]  (constant)    │
└─────────────────────────────────────────────────────────────┘
```

### **Scalability**

```
iptables:
- Rules checked sequentially: O(n)
- NAT table traversal adds overhead
- Connection tracking global lock contention
- Rule updates require full rebuild

eBPF:
- Hash map lookups: O(1)
- Per-CPU data structures (no locks)
- Atomic map updates
- No packet drops during updates
```

### **Feature Comparison**

| **Feature** | **iptables** | **eBPF** |
|------------|-------------|----------|
| **Performance** | O(n) rule traversal | O(1) hash lookup |
| **Scalability** | Degrades with rules | Constant performance |
| **Update Speed** | Slow (rebuild) | Fast (atomic) |
| **Packet Loss on Update** | Yes | No |
| **CPU Usage** | High with many rules | Low |
| **Memory** | Rules in kernel | Maps in kernel |
| **Observability** | Limited (counters) | Extensive (maps) |
| **Programmability** | Static chains | Full programming |
| **NAT** | Connection tracking | Stateless or stateful |
| **Load Balancing** | Limited | Advanced algorithms |
| **IPv4/IPv6** | Separate tables | Unified |
| **Kernel Version** | Ancient (2.4+) | Modern (4.8+) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🐝 Cilium eBPF Implementation**

### **Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                  Cilium Agent                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Policy Engine                                         │ │
│  │  - NetworkPolicy → eBPF rules                         │ │
│  │  - Identity management                                 │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  eBPF Compiler & Loader                               │ │
│  │  - Compile C to eBPF bytecode                         │ │
│  │  - Load into kernel                                    │ │
│  │  - Update BPF maps                                     │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
        ┌──────────────────────────────────┐
        │      Linux Kernel                 │
        │  ┌────────────────────────────┐  │
        │  │  eBPF Runtime              │  │
        │  │  - JIT compiler            │  │
        │  │  - Verifier                │  │
        │  │  - BPF maps                │  │
        │  └────────────────────────────┘  │
        │  ┌────────────────────────────┐  │
        │  │  Attached Programs         │  │
        │  │  - XDP                     │  │
        │  │  - TC (traffic control)    │  │
        │  │  - Sockops                 │  │
        │  │  - Cgroup                  │  │
        │  └────────────────────────────┘  │
        └──────────────────────────────────┘
```

### **Pod Network Setup**

```bash
# When pod is created, Cilium:
# 1. Creates veth pair
# 2. Attaches TC eBPF programs to veth
# 3. Allocates identity
# 4. Updates BPF maps

# Example: View attached programs
$ tc filter show dev lxc1234567890ab egress
filter protocol all pref 1 bpf chain 0
filter protocol all pref 1 bpf chain 0 handle 0x1 bpf_lxc.o:[from-container] direct-action not_in_hw id 123

$ tc filter show dev lxc1234567890ab ingress
filter protocol all pref 1 bpf chain 0 handle 0x1 bpf_lxc.o:[to-container] direct-action not_in_hw id 124
```

### **BPF Maps**

```c
// Cilium BPF maps
// Location: bpf/lib/maps.h

// Endpoint to identity mapping
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 65536);
    __type(key, struct endpoint_key);
    __type(value, struct endpoint_info);
    __uint(pinning, LIBBPF_PIN_BY_NAME);
} cilium_lxc SEC(".maps");

// Policy map (identity-based)
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 16384);
    __type(key, struct policy_key);
    __type(value, struct policy_entry);
} cilium_policy SEC(".maps");

// Connection tracking
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 524288);
    __type(key, struct ct_key);
    __type(value, struct ct_entry);
} cilium_ct_tcp4 SEC(".maps");

// Service load balancer
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 65536);
    __type(key, struct lb4_key);
    __type(value, struct lb4_service);
} cilium_lb4_services SEC(".maps");

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 262144);
    __type(key, struct lb4_backend_key);
    __type(value, struct lb4_backend);
} cilium_lb4_backends SEC(".maps");
```

### **Packet Processing Flow**

```c
// Simplified packet processing in Cilium
// Location: bpf/bpf_lxc.c

SEC("tc")
int from_container(struct __sk_buff *skb) {
    struct ct_state ct_state = {};
    struct endpoint_info *ep;
    __u32 identity = 0;
    int ret;

    // 1. Extract packet metadata
    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    // 2. Parse L2/L3/L4 headers
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return TC_ACT_OK;

    // 3. Lookup source endpoint
    ep = lookup_endpoint(skb);
    if (!ep)
        return DROP_INVALID_SRC;

    identity = ep->sec_label;  // Security identity

    // 4. Connection tracking
    ret = ct_lookup4(&ct_state, skb, CT_EGRESS);
    if (ret < 0)
        return ret;

    // 5. Policy enforcement
    ret = policy_can_egress(identity, &ct_state, skb);
    if (ret != POLICY_ACT_ALLOW)
        return ret;

    // 6. NAT (if service)
    ret = lb4_local(skb, &ct_state);
    if (ret != NAT_NOT_NEEDED && ret != NAT_46X64_RECIRC)
        return ret;

    // 7. Routing
    return redirect_to_destination(skb, ep);
}

SEC("tc")
int to_container(struct __sk_buff *skb) {
    struct endpoint_info *ep;
    __u32 src_identity = 0;
    int ret;

    // 1. Lookup destination endpoint
    ep = lookup_endpoint(skb);
    if (!ep)
        return DROP_INVALID_DST;

    // 2. Extract source identity
    src_identity = skb->mark;  // Set by previous program

    // 3. Policy enforcement (ingress)
    ret = policy_can_ingress(src_identity, ep->sec_label, skb);
    if (ret != POLICY_ACT_ALLOW)
        return ret;

    // 4. Deliver to container
    return TC_ACT_OK;
}
```

### **Identity-Based Security**

```c
// Identity assignment
struct endpoint_info {
    __u32 sec_label;     // Security identity
    __u32 lxc_id;        // Container ID
    __u32 ifindex;       // Network interface
    __u16 flags;
};

// Policy lookup
static __always_inline int
policy_can_egress(__u32 src_id, struct ct_state *ct, struct __sk_buff *skb) {
    struct policy_key key = {
        .sec_label = src_id,
        .dport = ct->dport,
        .protocol = ct->proto,
        .egress = 1,
    };

    struct policy_entry *policy = map_lookup_elem(&cilium_policy, &key);
    if (!policy)
        return DROP_POLICY;

    // Check if destination identity is allowed
    __u32 dst_id = derive_dst_identity(skb);
    if (!(policy->allowed_ids & (1 << dst_id)))
        return DROP_POLICY;

    // L7 policy check (if needed)
    if (policy->l7_proxy) {
        return proxy_redirect(skb, policy->proxy_port);
    }

    return POLICY_ACT_ALLOW;
}
```

### **Service Load Balancing**

```c
// Service load balancing implementation
static __always_inline int
lb4_local(struct __sk_buff *skb, struct ct_state *ct) {
    struct lb4_key key = {
        .address = ct->addr,
        .dport = ct->dport,
        .backend_slot = 0,
        .proto = ct->proto,
    };

    // Lookup service
    struct lb4_service *svc = map_lookup_elem(&cilium_lb4_services, &key);
    if (!svc)
        return NAT_NOT_NEEDED;

    // Select backend
    __u32 backend_id = select_backend(svc, skb);

    key.backend_slot = backend_id;
    struct lb4_backend *backend = map_lookup_elem(&cilium_lb4_backends, &key);
    if (!backend)
        return DROP_NO_SERVICE;

    // Perform DNAT
    return lb4_xlate(skb, &backend->address, backend->port, ct);
}

// Consistent hashing for backend selection
static __always_inline __u32
select_backend(struct lb4_service *svc, struct __sk_buff *skb) {
    __u32 hash = get_packet_hash(skb);

    // Maglev hashing for consistent load balancing
    __u32 slot = hash % svc->slot_count;
    return svc->backend_ids[slot];
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔶 Calico eBPF Mode**

### **Enabling eBPF Mode**

```yaml
# ConfigMap for Calico eBPF
apiVersion: v1
kind: ConfigMap
metadata:
  name: calico-config
  namespace: kube-system
data:
  # Enable eBPF dataplane
  felix_bpf_enabled: "true"

  # Disable kube-proxy
  felix_bpf_kube_proxy_iptables_cleanup_enabled: "true"

  # BPF log level
  felix_bpf_log_level: "info"

  # External node IP
  felix_bpf_external_service_mode: "tunnel"

  # DSR mode for services
  felix_bpf_dsr_enabled: "true"

---
# Disable kube-proxy
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
```

### **BPF Program Structure**

```c
// Calico BPF program
// Location: felix/bpf-gpl/tc.c

SEC("tc")
int calico_tc_ingress(struct __sk_buff *skb) {
    struct cali_tc_ctx ctx = {
        .skb = skb,
        .fwd = {
            .res = TC_ACT_UNSPEC,
            .mark = 0,
        },
    };

    // Policy enforcement
    if (CALI_F_TO_HOST) {
        // Packet to host namespace
        return forward_to_host(&ctx);
    } else {
        // Packet to workload
        return policy_ingress(&ctx);
    }
}

static CALI_BPF_INLINE int policy_ingress(struct cali_tc_ctx *ctx) {
    // Lookup policy
    struct cali_policy_key key = {
        .ip = ctx->state->ip_dst,
        .port = ctx->state->dport,
        .proto = ctx->state->ip_proto,
    };

    struct cali_policy_result *policy = cali_v4_policy_lookup(&key);
    if (!policy || policy->action != CALI_POL_ALLOW) {
        CALI_DEBUG("Policy denied\n");
        return TC_ACT_SHOT;
    }

    return TC_ACT_UNSPEC;
}
```

### **Features**

```yaml
# Calico eBPF capabilities
Features:
  - kube-proxy replacement (Services, NodePort, LoadBalancer)
  - DSR (Direct Server Return) for performance
  - Source IP preservation
  - Topology-aware routing
  - eBPF-based policy enforcement
  - Connection tracking in eBPF
  - NAT46/64 (IPv4/IPv6 translation)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ XDP (eXpress Data Path)**

### **XDP Overview**

```
XDP Execution Flow:
┌──────────────────────────────────────────────────┐
│                  NIC                              │
│                                                   │
│  Packet arrives ───► DMA buffer ───► XDP hook    │
│                                         │         │
└─────────────────────────────────────────┼─────────┘
                                          │
                          ┌───────────────┼───────────────┐
                          │               ▼               │
                          │     ┌──────────────────┐     │
                          │     │   XDP Program    │     │
                          │     │   (eBPF)         │     │
                          │     └────────┬─────────┘     │
                          │              │               │
                          │         Decision:            │
                          │              │               │
                ┌─────────┼──────────────┼───────────────┼─────────┐
                │         │              │               │         │
                ▼         ▼              ▼               ▼         ▼
            XDP_DROP  XDP_PASS      XDP_TX         XDP_REDIRECT  XDP_ABORTED
            (discard) (continue)   (bounce back)   (redirect)    (error)
```

### **XDP Modes**

```bash
# Generic mode (no driver support needed)
ip link set dev eth0 xdpgeneric obj xdp_prog.o sec xdp

# Native mode (driver support required - fastest)
ip link set dev eth0 xdpdrv obj xdp_prog.o sec xdp

# Offload mode (SmartNIC - hardware offload)
ip link set dev eth0 xdpoffload obj xdp_prog.o sec xdp
```

### **XDP Program Example**

```c
// XDP DDoS protection
// Location: examples/xdp_ddos.c

#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

#define MAX_CPUS 128

// Per-CPU packet counter
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, 256);
    __type(key, __u32);
    __type(value, __u64);
} pkt_count SEC(".maps");

// Rate limiting map
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, 10000);
    __type(key, __u32);      // Source IP
    __type(value, __u64);    // Packet count
} rate_limit SEC(".maps");

SEC("xdp")
int xdp_ddos_filter(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;

    // Parse Ethernet
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    // Only IPv4
    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_PASS;

    // Parse IP
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    // Rate limiting
    __u32 src_ip = ip->saddr;
    __u64 *pkt_cnt = bpf_map_lookup_elem(&rate_limit, &src_ip);

    if (pkt_cnt) {
        __sync_fetch_and_add(pkt_cnt, 1);

        // Drop if exceeds rate (e.g., 1000 pps)
        if (*pkt_cnt > 1000) {
            // Update drop counter
            __u32 key = XDP_DROP;
            __u64 *counter = bpf_map_lookup_elem(&pkt_count, &key);
            if (counter)
                __sync_fetch_and_add(counter, 1);

            return XDP_DROP;
        }
    } else {
        __u64 initial_count = 1;
        bpf_map_update_elem(&rate_limit, &src_ip, &initial_count, BPF_ANY);
    }

    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

### **XDP Performance**

```
Packet Processing Performance:
┌─────────────────────────────────────────────────────────────┐
│  Packet Rate (millions pps)                                 │
├─────────────────────────────────────────────────────────────┤
│  Linux default (no XDP):      3M   [▓▓▓░░░░░░░]            │
│  XDP Generic mode:            5M   [▓▓▓▓▓░░░░░]            │
│  XDP Native mode:            20M   [▓▓▓▓▓▓▓▓▓▓]            │
│  XDP Offload mode (SmartNIC): 100M [▓▓▓▓▓▓▓▓▓▓] (hardware) │
└─────────────────────────────────────────────────────────────┘

Latency:
┌─────────────────────────────────────────────────────────────┐
│  Packet Processing Latency                                  │
├─────────────────────────────────────────────────────────────┤
│  iptables:                   50 µs [▓▓▓▓▓▓▓▓▓▓]            │
│  XDP:                         5 µs [▓░░░░░░░░░]            │
└─────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🗺️ BPF Maps and Programs**

### **Map Operations**

```c
// Map operations in eBPF

// Lookup
void *value = bpf_map_lookup_elem(&my_map, &key);
if (!value)
    return -1;

// Update
int ret = bpf_map_update_elem(&my_map, &key, &value, BPF_ANY);
// Flags: BPF_ANY, BPF_NOEXIST, BPF_EXIST

// Delete
bpf_map_delete_elem(&my_map, &key);

// Atomic increment (per-CPU maps)
__sync_fetch_and_add(value, 1);
```

### **Map Access from Userspace**

```go
// Go code to access BPF maps
// Location: pkg/ebpf/maps.go

import (
    "github.com/cilium/ebpf"
)

func UpdatePolicyMap(policyMap *ebpf.Map, srcIdentity, dstIdentity uint32, allow bool) error {
    key := PolicyKey{
        SrcIdentity: srcIdentity,
        DstIdentity: dstIdentity,
    }

    value := PolicyValue{
        Allow: 1,
    }
    if !allow {
        value.Allow = 0
    }

    return policyMap.Put(key, value)
}

func ReadStats(statsMap *ebpf.Map) (map[uint32]uint64, error) {
    stats := make(map[uint32]uint64)

    var key uint32
    var value uint64

    iter := statsMap.Iterate()
    for iter.Next(&key, &value) {
        stats[key] = value
    }

    return stats, iter.Err()
}
```

### **Program Tail Calls**

```c
// Tail call for program chaining
struct {
    __uint(type, BPF_MAP_TYPE_PROG_ARRAY);
    __uint(max_entries, 10);
    __type(key, __u32);
    __type(value, __u32);
} jmp_table SEC(".maps");

SEC("tc")
int main_program(struct __sk_buff *skb) {
    // Do initial processing
    ...

    // Tail call to next program
    bpf_tail_call(skb, &jmp_table, 0);

    // This code is only reached if tail call fails
    return TC_ACT_OK;
}

SEC("tc")
int policy_program(struct __sk_buff *skb) {
    // Policy enforcement
    ...
    return TC_ACT_OK;
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 kube-proxy Replacement**

### **Traditional kube-proxy**

```
kube-proxy (iptables mode):
┌──────────────────────────────────────────────────┐
│  Service: 10.96.0.1:80                           │
│     Backend 1: 10.244.1.5:8080                   │
│     Backend 2: 10.244.2.3:8080                   │
└──────────────────┬───────────────────────────────┘
                   │
                   ▼
     ┌─────────────────────────────┐
     │  iptables NAT rules         │
     │  (100s-1000s of rules)      │
     │                              │
     │  -A KUBE-SVC-XXX             │
     │    -m statistic --mode random│
     │    --probability 0.5         │
     │    -j KUBE-SEP-BACKEND1      │
     │  -A KUBE-SVC-XXX             │
     │    -j KUBE-SEP-BACKEND2      │
     └─────────────────────────────┘
```

### **eBPF kube-proxy Replacement**

```
eBPF load balancer:
┌──────────────────────────────────────────────────┐
│  Service: 10.96.0.1:80                           │
│     Backend 1: 10.244.1.5:8080                   │
│     Backend 2: 10.244.2.3:8080                   │
└──────────────────┬───────────────────────────────┘
                   │
                   ▼
     ┌─────────────────────────────┐
     │  BPF Maps (O(1) lookup)     │
     │                              │
     │  Service Map:                │
     │    10.96.0.1:80 -> svc_id    │
     │                              │
     │  Backend Map:                │
     │    svc_id -> [backends]      │
     │                              │
     │  Connection Tracking:        │
     │    5-tuple -> backend_id     │
     └─────────────────────────────┘
```

### **Implementation**

```c
// Service load balancing in eBPF
// Simplified from Cilium implementation

struct lb4_key {
    __be32 address;    // Service IP
    __be16 dport;      // Service port
    __u8 proto;
    __u8 scope;
};

struct lb4_service {
    __u16 count;           // Backend count
    __u16 flags;
    __u32 backend_id;      // First backend
};

struct lb4_backend {
    __be32 address;        // Backend IP
    __be16 port;           // Backend port
    __u16 flags;
};

static __always_inline int
lb4_xlate_fwd(struct __sk_buff *skb, struct lb4_key *key) {
    // Lookup service
    struct lb4_service *svc = map_lookup_elem(&cilium_lb4_services, key);
    if (!svc)
        return DROP_NO_SERVICE;

    // Select backend (consistent hashing)
    __u32 backend_idx = lb4_select_backend(skb, svc);

    struct lb4_backend *backend = lb4_lookup_backend(svc->backend_id + backend_idx);
    if (!backend)
        return DROP_NO_SERVICE;

    // Perform DNAT
    if (skb->protocol == htons(ETH_P_IP)) {
        struct iphdr *ip4 = skb_network_header(skb);

        // Checksum update
        l3_csum_replace(skb, IP_CSUM_OFF, key->address, backend->address, 4);

        ip4->daddr = backend->address;
    }

    // Update L4 port
    if (key->proto == IPPROTO_TCP) {
        struct tcphdr *tcp = skb_transport_header(skb);
        l4_csum_replace(skb, TCP_CSUM_OFF, key->dport, backend->port, 2);
        tcp->dest = backend->port;
    }

    return TC_ACT_OK;
}
```

### **DSR (Direct Server Return)**

```
Without DSR:
Client ──────► Service ──────► Backend
               (SNAT)          │
Client ◄────── Service ◄───────┘
               (un-SNAT)

With DSR:
Client ──────► Service ──────► Backend
               (no SNAT)        │
Client ◄─────────────────────────┘
(Backend responds directly to client)

Benefits:
- Reduced latency
- Less CPU on load balancer
- Symmetric routing not required
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Performance Analysis**

### **Benchmarking**

```bash
# Packet rate test with XDP
$ ./xdp_bench

Results:
XDP_DROP:          24 Mpps
XDP_TX:            20 Mpps
XDP_REDIRECT:      18 Mpps

# Service load balancing performance
$ wrk -t12 -c400 -d30s http://service-ip

Traditional kube-proxy (iptables):
  Requests/sec:     50,000
  Latency:          8ms avg, 20ms p99

Cilium eBPF:
  Requests/sec:     120,000
  Latency:          3ms avg, 8ms p99

# CPU usage
Traditional: 40% CPU (iptables traversal)
eBPF:        15% CPU (hash lookups)
```

### **Scalability Metrics**

```
Service Scalability:
┌─────────────────────────────────────────────────────────────┐
│  # of Services    │  iptables    │  eBPF                    │
├───────────────────┼──────────────┼──────────────────────────┤
│  100              │  Good        │  Excellent               │
│  1,000            │  Degraded    │  Excellent               │
│  10,000           │  Poor        │  Excellent               │
│  50,000           │  Unusable    │  Good                    │
└─────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting**

### **Debugging eBPF Programs**

```bash
# List loaded programs
bpftool prog list

# Show specific program
bpftool prog show id 123

# Dump program instructions
bpftool prog dump xlated id 123

# View maps
bpftool map list

# Dump map contents
bpftool map dump id 456

# Cilium-specific debugging
cilium bpf policy get
cilium bpf lb list
cilium bpf endpoint list
cilium bpf ct list global

# Monitor eBPF events
cilium monitor --type drop
cilium monitor --type trace
```

### **Common Issues**

```bash
# Issue: eBPF program failed to load
# Solution: Check kernel version and BTF support
uname -r  # Should be 4.8+
ls /sys/kernel/btf/vmlinux  # BTF support

# Issue: Map update failures
# Solution: Check map size limits
sysctl kernel.bpf.max_entries

# Issue: Verifier errors
# Solution: Simplify program, add bounds checks

# Enable debug logs
echo 1 > /proc/sys/kernel/printk
dmesg -w | grep -i bpf
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💻 Development Guide**

### **Setting Up Development Environment**

```bash
# Install dependencies
sudo apt-get install -y \
    clang llvm \
    libbpf-dev \
    linux-headers-$(uname -r) \
    bpftool

# Clone Cilium (example)
git clone https://github.com/cilium/cilium.git
cd cilium

# Build BPF programs
make -C bpf build

# Run tests
make -C bpf test
```

### **Writing Custom eBPF Programs**

```c
// Example: Custom packet counter
// File: my_counter.c

#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 256);
    __type(key, __u32);
    __type(value, __u64);
} packet_counter SEC(".maps");

SEC("xdp")
int count_packets(struct xdp_md *ctx) {
    __u32 key = XDP_PASS;
    __u64 *counter = bpf_map_lookup_elem(&packet_counter, &key);

    if (counter)
        __sync_fetch_and_add(counter, 1);

    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

**Compile and Load:**
```bash
# Compile
clang -O2 -target bpf -c my_counter.c -o my_counter.o

# Load
sudo ip link set dev eth0 xdp obj my_counter.o sec xdp

# View stats
sudo bpftool map dump name packet_counter
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **✅ Best Practices**

### **Performance**

1. **Use per-CPU maps** to avoid lock contention
2. **Minimize verifier complexity** for faster loading
3. **Use tail calls** for complex programs
4. **Leverage XDP** for earliest packet processing
5. **Profile with perf** to identify bottlenecks

### **Security**

1. **Validate all packet data** to pass verifier
2. **Use BTF** for type safety
3. **Limit map sizes** to prevent memory exhaustion
4. **Regular map cleanup** for LRU maps
5. **Monitor resource usage**

### **Operations**

1. **Test in dev first** before production
2. **Monitor eBPF stats** via Prometheus
3. **Keep kernel updated** for latest features
4. **Use cilium/bpftool** for debugging
5. **Document custom programs**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Summary**

### **Key Takeaways**

1. **eBPF provides O(1) packet processing** vs O(n) for iptables
2. **XDP offers lowest latency** for packet filtering
3. **kube-proxy replacement** improves service performance
4. **Identity-based security** more scalable than IP-based
5. **Observability built-in** with map instrumentation

### **Code References**

```plaintext
Cilium eBPF:
  github.com/cilium/cilium/bpf/
  github.com/cilium/ebpf/

Calico eBPF:
  github.com/projectcalico/calico/felix/bpf-gpl/

Kernel eBPF:
  /usr/include/linux/bpf.h
  kernel/bpf/
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
