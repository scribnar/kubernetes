# **KUBERNETES NETWORKING ARCHITECTURE**

**Network Policies, Service Mesh, and Container Networking**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Purpose**

This documentation series explains Kubernetes networking architecture, focusing on network policies, service mesh integration, and how container networking is implemented. Understanding these components is essential for securing and optimizing Kubernetes workloads.

**What You'll Learn**:
- ✅ How NetworkPolicy resources enforce network segmentation
- ✅ How network policy controllers (Calico, Cilium, etc.) implement policies
- ✅ How service mesh (Istio, Linkerd) integrates with Kubernetes
- ✅ How eBPF enables advanced networking features
- ✅ How CNI plugins provide pod networking

**What Makes This Different**:
- Real implementation details from kubernetes/kubernetes and popular CNI plugins
- Complete coverage of network policy enforcement mechanisms
- Service mesh architecture and integration patterns
- eBPF-based networking (Cilium) vs iptables approaches
- Practical network segmentation strategies

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Documentation Structure**

### **High-Level Architecture** (2 documents)

| Document | Topic | Lines | Diagrams | Priority |
|----------|-------|------:|----------|----------|
| **[01-networking-overview.md](high-level/01-networking-overview.md)** | Kubernetes networking model | ~2,000 | 12 | 🚨 Critical |
| **[02-network-policy-architecture.md](high-level/02-network-policy-architecture.md)** | NetworkPolicy resource architecture | ~1,800 | 10 | 🚨 Critical |

### **Middle-Level Implementation** (5 documents)

| Document | Topic | Lines | Diagrams | Priority |
|----------|-------|------:|----------|----------|
| **[01-network-policies.md](middle-level/01-network-policies.md)** | NetworkPolicy implementation | ~2,500 | 15 | 🚨 Critical |
| **[02-policy-controllers.md](middle-level/02-policy-controllers.md)** | Calico, Cilium, Weave | ~2,800 | 16 | 🚨 Critical |
| **[03-service-mesh-integration.md](middle-level/03-service-mesh-integration.md)** | Istio, Linkerd integration | ~3,000 | 18 | ⚠️ High |
| **[04-ebpf-networking.md](middle-level/04-ebpf-networking.md)** | eBPF-based networking | ~2,600 | 14 | ⚠️ High |
| **[05-network-segmentation.md](middle-level/05-network-segmentation.md)** | Multi-tenancy patterns | ~2,200 | 12 | ⚠️ High |

### **Low-Level Implementation** (4 documents)

| Document | Topic | Lines | Diagrams | Priority |
|----------|-------|------:|----------|----------|
| **[01-policy-enforcement.md](low-level/01-policy-enforcement.md)** | iptables/eBPF enforcement | ~2,400 | 12 | ⚠️ High |
| **[02-pod-networking.md](low-level/02-pod-networking.md)** | Pod network setup | ~2,200 | 10 | ⚠️ High |
| **[03-service-implementation.md](low-level/03-service-implementation.md)** | Service proxy details | ~2,600 | 14 | 📘 Medium |
| **[04-dns-resolution.md](low-level/04-dns-resolution.md)** | CoreDNS integration | ~2,000 | 10 | 📘 Medium |

**Total**: 11 documents, ~26,100 lines, ~133 diagrams

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🗺️ Learning Path**

### **Beginner Path** (Start Here - 4-6 hours)
1. **[01-networking-overview.md](high-level/01-networking-overview.md)** - Kubernetes networking fundamentals
2. **[02-network-policy-architecture.md](high-level/02-network-policy-architecture.md)** - NetworkPolicy basics
3. **[01-network-policies.md](middle-level/01-network-policies.md)** - Writing network policies
4. **[05-network-segmentation.md](middle-level/05-network-segmentation.md)** - Practical segmentation

### **Intermediate Path** (8-10 hours)
1. Complete Beginner Path
2. **[02-policy-controllers.md](middle-level/02-policy-controllers.md)** - CNI plugin comparison
3. **[03-service-mesh-integration.md](middle-level/03-service-mesh-integration.md)** - Service mesh patterns
4. **[01-policy-enforcement.md](low-level/01-policy-enforcement.md)** - Enforcement mechanisms

### **Advanced Path** (12-14 hours)
1. Complete Intermediate Path
2. **[04-ebpf-networking.md](middle-level/04-ebpf-networking.md)** - eBPF deep dive
3. **[02-pod-networking.md](low-level/02-pod-networking.md)** - Pod networking internals
4. **[03-service-implementation.md](low-level/03-service-implementation.md)** - Service proxy details

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Quick Navigation**

### **I Want To...**

#### **Implement Network Policies**
→ [01-network-policies.md](middle-level/01-network-policies.md) - Complete NetworkPolicy guide
→ [02-network-policy-architecture.md](high-level/02-network-policy-architecture.md) - Architecture
→ [05-network-segmentation.md](middle-level/05-network-segmentation.md) - Multi-tenancy patterns

#### **Choose a CNI Plugin**
→ [02-policy-controllers.md](middle-level/02-policy-controllers.md) - Calico vs Cilium vs Weave
→ [04-ebpf-networking.md](middle-level/04-ebpf-networking.md) - eBPF advantages
→ [01-cni-networking-overview.md](01-cni-networking-overview.md) - CNI basics

#### **Integrate Service Mesh**
→ [03-service-mesh-integration.md](middle-level/03-service-mesh-integration.md) - Istio/Linkerd
→ [01-networking-overview.md](high-level/01-networking-overview.md) - Networking model

#### **Understand Policy Enforcement**
→ [01-policy-enforcement.md](low-level/01-policy-enforcement.md) - iptables/eBPF rules
→ [02-policy-controllers.md](middle-level/02-policy-controllers.md) - Controller implementations

#### **Debug Networking Issues**
→ [02-pod-networking.md](low-level/02-pod-networking.md) - Pod network setup
→ [04-dns-resolution.md](low-level/04-dns-resolution.md) - DNS troubleshooting
→ [03-service-implementation.md](low-level/03-service-implementation.md) - Service connectivity

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💡 Key Concepts**

### **Kubernetes Networking Model**

Kubernetes enforces the following networking requirements:
1. **All pods can communicate** with all other pods without NAT
2. **All nodes can communicate** with all pods (and vice versa) without NAT
3. **The IP a pod sees itself as** is the same IP others see it as

### **Network Policy Fundamentals**

| Feature | Description | Use Case |
|---------|-------------|----------|
| **Ingress Rules** | Control incoming traffic to pods | Restrict which pods can access a service |
| **Egress Rules** | Control outgoing traffic from pods | Prevent data exfiltration |
| **Pod Selector** | Select pods by labels | Apply policies to groups of pods |
| **Namespace Selector** | Allow traffic from specific namespaces | Multi-tenant isolation |
| **IP Block** | Allow/deny specific IP ranges | External service access control |
| **Port/Protocol** | Restrict by port and protocol | Fine-grained access control |

### **CNI Plugin Comparison**

| Plugin | Enforcement | Performance | Features |
|--------|-------------|-------------|----------|
| **Calico** | iptables/eBPF | High | BGP routing, network policies, Wireguard encryption |
| **Cilium** | eBPF | Very High | Identity-based policies, L7 visibility, multi-cluster |
| **Weave** | iptables | Medium | Simple setup, encryption, multicast |
| **Flannel** | None | High | Simple overlay, no network policies |

### **Service Mesh vs Network Policies**

| Aspect | Network Policies | Service Mesh |
|--------|------------------|--------------|
| **Layer** | L3/L4 (IP/Port) | L7 (HTTP/gRPC) |
| **Granularity** | Pod-to-pod | Request-level |
| **Features** | Firewall rules | mTLS, retries, circuit breakers, observability |
| **Overhead** | Low | Medium-High (sidecar proxy) |
| **Use Case** | Network segmentation | Service-to-service communication |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏗️ Networking Architecture Overview**

### **Component Stack**

```
┌────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│              (Pods, Services, Endpoints)                    │
└────────────────────┬───────────────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
┌─────────────────┐   ┌──────────────────────┐
│ Service Mesh    │   │ NetworkPolicy API    │
│ (Istio/Linkerd) │   │ (k8s.io/networking)  │
│ - Envoy Proxy   │   │ - Ingress/Egress     │
│ - mTLS          │   │ - Selectors          │
│ - L7 Policy     │   │ - IP Blocks          │
└─────────┬───────┘   └──────────┬───────────┘
          │                      │
          └──────────┬───────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│            CNI Plugin (Network Policy Controller)          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │   Calico     │  │   Cilium     │  │   Weave/Others   │ │
│  │ - iptables   │  │ - eBPF       │  │ - iptables       │ │
│  │ - BGP        │  │ - Identity   │  │ - Overlay        │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│                  Kernel Networking                          │
│  • iptables/nftables  • eBPF maps  • Network namespaces    │
│  • Routing tables     • veth pairs • Bridge/VXLAN          │
└────────────────────────────────────────────────────────────┘
```

### **Code Locations**

| Component | Location | Purpose |
|-----------|----------|---------|
| **NetworkPolicy API** | `/staging/src/k8s.io/api/networking/v1/types.go` | NetworkPolicy resource types |
| **Network Plugin** | `/pkg/kubelet/network/` | Kubelet network plugin interface |
| **Service Proxy** | `/pkg/proxy/` | kube-proxy implementation |
| **DNS** | `/cluster/addons/dns/` | CoreDNS manifests |
| **CNI Invocation** | `/pkg/kubelet/kubelet_network.go` | CNI plugin invocation |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📖 Document Summaries**

### **High-Level Documents**

#### **01. Networking Overview**
Complete introduction to Kubernetes networking model. Covers pod networking, service networking, network namespaces, and how containers communicate.

**Key Topics**: Networking requirements, pod-to-pod communication, service discovery, DNS

#### **02. Network Policy Architecture**
Architecture of NetworkPolicy resources and how they're enforced by CNI plugins.

**Key Topics**: NetworkPolicy API, policy controllers, ingress/egress rules, selectors

### **Middle-Level Documents**

#### **01. Network Policies**
Complete guide to writing and managing NetworkPolicy resources. Includes common patterns and examples.

**Key Topics**: Policy syntax, selectors, CIDR blocks, default deny, namespace isolation

#### **02. Policy Controllers**
Comparison of popular CNI plugins that implement network policies: Calico, Cilium, Weave.

**Key Topics**: Calico BGP routing, Cilium eBPF, Weave encryption, performance comparison

#### **03. Service Mesh Integration**
How service meshes (Istio, Linkerd) integrate with Kubernetes networking and complement network policies.

**Key Topics**: Sidecar injection, mTLS, L7 policies, observability, multi-cluster

#### **04. eBPF Networking**
Deep dive into eBPF-based networking with Cilium. Modern alternative to iptables.

**Key Topics**: eBPF programs, identity-based security, L7 visibility, performance benefits

#### **05. Network Segmentation**
Practical patterns for multi-tenancy and network segmentation in Kubernetes.

**Key Topics**: Namespace isolation, tenant separation, default deny policies, compliance

### **Low-Level Documents**

#### **01. Policy Enforcement**
How network policies are enforced at the kernel level using iptables or eBPF.

**Key Topics**: iptables chains, eBPF maps, packet filtering, rule generation

#### **02. Pod Networking**
How pod networking is set up: veth pairs, network namespaces, IP allocation.

**Key Topics**: CNI plugin execution, network namespace creation, IP address management

#### **03. Service Implementation**
How Services are implemented by kube-proxy using iptables, IPVS, or eBPF.

**Key Topics**: Service proxy modes, load balancing, session affinity, external traffic

#### **04. DNS Resolution**
How CoreDNS provides service discovery and DNS resolution for pods.

**Key Topics**: DNS server configuration, service DNS records, pod DNS policy

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Related Documentation**

### **Component Documentation**
How networking integrates with other components:

- **kube-proxy**: `docs/architecture/claude/kube-proxy/` - Service implementation
- **kubelet**: `docs/architecture/claude/kubelet/` - CNI plugin invocation
- **API Server**: `docs/architecture/claude/apiserver/` - NetworkPolicy API

### **Foundation Concepts**
Prerequisites for understanding networking:

- **Distributed Systems**: `docs/architecture/claude/distributed-systems/` - Eventual consistency
- **Common Patterns**: `docs/architecture/claude/common/` - Informers, workqueues

### **External Resources**
- **Kubernetes Networking Model**: https://kubernetes.io/docs/concepts/cluster-administration/networking/
- **NetworkPolicy**: https://kubernetes.io/docs/concepts/services-networking/network-policies/
- **CNI Specification**: https://github.com/containernetworking/cni/blob/main/SPEC.md
- **Cilium Documentation**: https://docs.cilium.io/
- **Calico Documentation**: https://docs.projectcalico.org/

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **✅ Why This Matters**

### **For Understanding Kubernetes**
Networking is fundamental to Kubernetes:
- Pods must communicate across nodes
- Services provide stable endpoints
- Network policies enforce security
- Service mesh adds observability and resilience

### **For Securing Kubernetes**
Network security requires:
- ✅ **Network Segmentation**: Isolate workloads by namespace/label
- ✅ **Default Deny**: Block all traffic except explicitly allowed
- ✅ **Egress Control**: Prevent data exfiltration
- ✅ **Encryption**: mTLS for service-to-service communication

### **For Operating Kubernetes**
Operational considerations:
- CNI plugin choice affects performance and features
- Network policies require CNI plugin support
- Service mesh adds overhead but provides observability
- Troubleshooting requires understanding the network stack

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Documentation Standards**

All networking documents follow these standards:

- **Dark-mode optimized**: Bold headings, long separators
- **Theory + Practice**: Networking concepts + real implementations
- **Code references**: Exact file:line from kubernetes/kubernetes
- **Real examples**: Complete NetworkPolicy YAMLs, CNI configurations
- **Visual diagrams**: Mermaid diagrams for packet flows
- **Practical focus**: How to secure and debug networks

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎓 Learning Objectives**

After completing this documentation series, you should be able to:

1. **Explain** the Kubernetes networking model and requirements
2. **Write** NetworkPolicy resources for various use cases
3. **Choose** the appropriate CNI plugin for your requirements
4. **Design** network segmentation strategies for multi-tenancy
5. **Integrate** service mesh with Kubernetes networking
6. **Debug** networking issues at all layers
7. **Optimize** network performance
8. **Secure** Kubernetes workloads with network policies

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**📅 Last Updated**: 2025-01-16
**📝 Repository Version**: kubernetes/kubernetes (master branch)
**👤 Generated By**: Claude AI (Sonnet 4.5)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
