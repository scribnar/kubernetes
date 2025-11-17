# **Network Policies in Kubernetes**

**Comprehensive Guide to NetworkPolicy Objects, Patterns, and Implementation**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Table of Contents**

1. [Overview](#overview)
2. [NetworkPolicy Fundamentals](#networkpolicy-fundamentals)
3. [Policy Specification Deep Dive](#policy-specification-deep-dive)
4. [Common Policy Patterns](#common-policy-patterns)
5. [Advanced Policy Scenarios](#advanced-policy-scenarios)
6. [Multi-tenancy Policies](#multi-tenancy-policies)
7. [Security Best Practices](#security-best-practices)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Performance Considerations](#performance-considerations)
10. [Real-World Examples](#real-world-examples)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Overview**

### **Purpose**

NetworkPolicy is a Kubernetes resource that provides declarative network security controls at Layer 3/4. It allows you to specify how groups of pods are allowed to communicate with each other and other network endpoints.

### **Key Capabilities**

- **Pod-level segmentation**: Control traffic at pod granularity
- **Namespace isolation**: Implement multi-tenancy boundaries
- **Direction control**: Separate ingress and egress rules
- **Label-based selection**: Dynamic policy application
- **CIDR-based rules**: External network access control
- **Port-level filtering**: Protocol and port restrictions

### **Code References**

```go
// NetworkPolicy API definition
// Location: pkg/apis/networking/types.go

type NetworkPolicy struct {
    metav1.TypeMeta
    metav1.ObjectMeta

    // Specification of the desired behavior for this NetworkPolicy
    Spec NetworkPolicySpec

    // Status is the current state of the NetworkPolicy
    Status NetworkPolicyStatus
}

type NetworkPolicySpec struct {
    // Selects the pods to which this NetworkPolicy object applies
    PodSelector metav1.LabelSelector

    // List of ingress rules to be applied to the selected pods
    Ingress []NetworkPolicyIngressRule

    // List of egress rules to be applied to the selected pods
    Egress []NetworkPolicyEgressRule

    // List of rule types that the NetworkPolicy relates to
    PolicyTypes []PolicyType
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 NetworkPolicy Fundamentals**

### **Basic Structure**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: logging
    ports:
    - protocol: TCP
      port: 5000
```

### **Default Behavior**

**Without NetworkPolicy:**
```
┌─────────────────────────────────────────┐
│  Namespace: default                     │
│                                         │
│  ┌──────┐      ┌──────┐      ┌──────┐ │
│  │ Pod1 │◄────►│ Pod2 │◄────►│ Pod3 │ │
│  └──────┘      └──────┘      └──────┘ │
│     ▲             ▲             ▲      │
│     └─────────────┴─────────────┘      │
│          All traffic allowed           │
└─────────────────────────────────────────┘
```

**With Default Deny:**
```
┌─────────────────────────────────────────┐
│  Namespace: default                     │
│                                         │
│  ┌──────┐      ┌──────┐      ┌──────┐ │
│  │ Pod1 │  ✗   │ Pod2 │  ✗   │ Pod3 │ │
│  └──────┘      └──────┘      └──────┘ │
│                                         │
│      All traffic blocked by default    │
└─────────────────────────────────────────┘
```

### **Policy Types**

```go
// Location: pkg/apis/networking/types.go

type PolicyType string

const (
    // PolicyTypeIngress indicates that this NetworkPolicy includes ingress rules
    PolicyTypeIngress PolicyType = "Ingress"

    // PolicyTypeEgress indicates that this NetworkPolicy includes egress rules
    PolicyTypeEgress PolicyType = "Egress"
)
```

#### **Ingress Only Policy**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-only
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
  # No egress rules - all egress traffic allowed
```

#### **Egress Only Policy**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-only
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  # No ingress rules - all ingress traffic allowed
```

### **Selector Mechanics**

#### **Pod Selector**

```yaml
spec:
  podSelector:
    matchLabels:
      app: myapp
      tier: backend
    matchExpressions:
    - key: environment
      operator: In
      values:
      - production
      - staging
```

```go
// Pod selector implementation
// Location: pkg/apis/networking/validation/validation.go

func ValidateNetworkPolicySpec(spec *networking.NetworkPolicySpec, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // Validate pod selector
    allErrs = append(allErrs, unversionedvalidation.ValidateLabelSelector(
        &spec.PodSelector, fldPath.Child("podSelector"))...)

    // Validate ingress rules
    for i, ingress := range spec.Ingress {
        ingressPath := fldPath.Child("ingress").Index(i)
        allErrs = append(allErrs, validateNetworkPolicyIngressRule(&ingress, ingressPath)...)
    }

    // Validate egress rules
    for i, egress := range spec.Egress {
        egressPath := fldPath.Child("egress").Index(i)
        allErrs = append(allErrs, validateNetworkPolicyEgressRule(&egress, egressPath)...)
    }

    return allErrs
}
```

#### **Namespace Selector**

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
        team: platform
```

#### **Combined Selectors**

```yaml
# AND logic - both conditions must match
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
    podSelector:
      matchLabels:
        app: frontend

# OR logic - either condition can match
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
  - podSelector:
      matchLabels:
        app: frontend
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Policy Specification Deep Dive**

### **Ingress Rules**

```go
// Location: pkg/apis/networking/types.go

type NetworkPolicyIngressRule struct {
    // List of ports which should be made accessible on the pods selected
    Ports []NetworkPolicyPort

    // List of sources which should be able to access the pods selected
    From []NetworkPolicyPeer
}

type NetworkPolicyPeer struct {
    // Selects Pods in this namespace
    PodSelector *metav1.LabelSelector

    // Selects Namespaces using cluster-scoped labels
    NamespaceSelector *metav1.LabelSelector

    // IPBlock defines policy on a particular IPBlock
    IPBlock *IPBlock
}
```

#### **Port Specifications**

```yaml
# TCP port with number
ingress:
- ports:
  - protocol: TCP
    port: 80

# UDP port
ingress:
- ports:
  - protocol: UDP
    port: 53

# Named port
ingress:
- ports:
  - protocol: TCP
    port: http  # References port name in pod spec

# Port range (Kubernetes 1.25+)
ingress:
- ports:
  - protocol: TCP
    port: 8000
    endPort: 8999

# Multiple ports
ingress:
- ports:
  - protocol: TCP
    port: 80
  - protocol: TCP
    port: 443
  - protocol: TCP
    port: 8080
```

#### **Source Specifications**

**Pod Selector:**
```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
        version: v2
      matchExpressions:
      - key: tier
        operator: In
        values:
        - web
        - api
```

**Namespace Selector:**
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        project: myproject
        environment: production
```

**IP Block:**
```yaml
ingress:
- from:
  - ipBlock:
      cidr: 172.16.0.0/16
      except:
      - 172.16.1.0/24
      - 172.16.2.0/24
```

### **Egress Rules**

```go
type NetworkPolicyEgressRule struct {
    // List of destination ports for outgoing traffic
    Ports []NetworkPolicyPort

    // List of destinations for outgoing traffic
    To []NetworkPolicyPeer
}
```

#### **Destination Specifications**

```yaml
egress:
# Database access
- to:
  - podSelector:
      matchLabels:
        app: postgres
  ports:
  - protocol: TCP
    port: 5432

# External API access
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 169.254.169.254/32  # Block metadata service
  ports:
  - protocol: TCP
    port: 443

# DNS resolution
- to:
  - namespaceSelector:
      matchLabels:
        name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
  ports:
  - protocol: UDP
    port: 53
  - protocol: TCP
    port: 53
```

### **IPBlock Details**

```go
// Location: pkg/apis/networking/types.go

type IPBlock struct {
    // CIDR is a string representing the IP Block
    CIDR string

    // Except is a slice of CIDRs that should not be included within an IP Block
    Except []string
}
```

```yaml
# Allow specific subnet
egress:
- to:
  - ipBlock:
      cidr: 10.0.0.0/8

# Allow all except specific ranges
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 10.0.0.0/8       # Private network
      - 172.16.0.0/12    # Private network
      - 192.168.0.0/16   # Private network
      - 169.254.169.254/32  # AWS metadata
```

### **Port Range Support**

```yaml
# Kubernetes 1.25+ feature
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: port-range-example
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: monitoring
    ports:
    - protocol: TCP
      port: 9000
      endPort: 9100  # Allows ports 9000-9100
```

```go
// Port range validation
// Location: pkg/apis/networking/validation/validation.go

func validateNetworkPolicyPort(port *networking.NetworkPolicyPort, portPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    if port.Port != nil && port.EndPort != nil {
        portNum := port.Port.IntVal
        endPortNum := *port.EndPort

        if endPortNum <= portNum {
            allErrs = append(allErrs, field.Invalid(
                portPath.Child("endPort"),
                endPortNum,
                "endPort must be greater than port"))
        }

        if endPortNum > 65535 {
            allErrs = append(allErrs, field.Invalid(
                portPath.Child("endPort"),
                endPortNum,
                "endPort must be between 1 and 65535"))
        }
    }

    return allErrs
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎨 Common Policy Patterns**

### **Pattern 1: Default Deny All Traffic**

```yaml
# Deny all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}  # Applies to all pods
  policyTypes:
  - Ingress

---
# Deny all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress

---
# Deny all traffic (both directions)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### **Pattern 2: Allow All Traffic**

```yaml
# Allow all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}  # Empty rule allows all

---
# Allow all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - {}
```

### **Pattern 3: Three-Tier Application**

```yaml
# Frontend policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from internet
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
  egress:
  # Allow to backend
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# Backend policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from frontend only
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Allow to database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# Database policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from backend only
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  # Allow DNS only
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

**Flow Diagram:**
```
┌──────────────────────────────────────────────────────────┐
│  Internet (0.0.0.0/0)                                    │
└────────────────────┬─────────────────────────────────────┘
                     │ TCP 80/443
                     ▼
         ┌───────────────────────┐
         │   Frontend Tier       │
         │  (tier=frontend)      │
         └───────────┬───────────┘
                     │ TCP 8080
                     ▼
         ┌───────────────────────┐
         │   Backend Tier        │
         │  (tier=backend)       │
         └───────────┬───────────┘
                     │ TCP 5432
                     ▼
         ┌───────────────────────┐
         │   Database Tier       │
         │  (tier=database)      │
         └───────────────────────┘
```

### **Pattern 4: Namespace Isolation**

```yaml
# Isolate namespace from other namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from same namespace only
  - from:
    - podSelector: {}
  egress:
  # Allow to same namespace
  - to:
    - podSelector: {}
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Allow to kube-apiserver
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          component: kube-apiserver
    ports:
    - protocol: TCP
      port: 443
```

### **Pattern 5: Microservices Communication**

```yaml
# Service A policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: service-a-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: service-a
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from API Gateway
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Allow to Service B
  - to:
    - podSelector:
        matchLabels:
          app: service-b
    ports:
    - protocol: TCP
      port: 8080
  # Allow to Service C
  - to:
    - podSelector:
        matchLabels:
          app: service-c
    ports:
    - protocol: TCP
      port: 8080
  # Allow to shared cache
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### **Pattern 6: External Database Access**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: external-db-access
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: backend
      database-client: "true"
  policyTypes:
  - Egress
  egress:
  # Allow to external PostgreSQL
  - to:
    - ipBlock:
        cidr: 10.100.50.0/24  # Database subnet
    ports:
    - protocol: TCP
      port: 5432
  # Allow to external MySQL
  - to:
    - ipBlock:
        cidr: 10.100.51.0/24
    ports:
    - protocol: TCP
      port: 3306
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### **Pattern 7: Monitoring and Observability**

```yaml
# Allow monitoring scraping
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      monitoring: "true"
  policyTypes:
  - Ingress
  ingress:
  # Allow Prometheus scraping
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    podSelector:
      matchLabels:
        app: prometheus
    ports:
    - protocol: TCP
      port: 9090
  # Allow from Grafana
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    podSelector:
      matchLabels:
        app: grafana
    ports:
    - protocol: TCP
      port: 3000
```

### **Pattern 8: CI/CD Access**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cicd-access
  namespace: staging
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  # Allow from CI/CD namespace
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: cicd
    ports:
    - protocol: TCP
      port: 8080
  # Allow from kubectl/operators
  - from:
    - ipBlock:
        cidr: 10.200.0.0/16  # Admin VPN range
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚀 Advanced Policy Scenarios**

### **Scenario 1: Multi-Environment Deployment**

```yaml
# Production environment policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prod-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Block from staging/dev
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          environment: production
  egress:
  # Allow only to production resources
  - to:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          environment: production
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # External production APIs
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8      # Block internal dev networks
    ports:
    - protocol: TCP
      port: 443

---
# Staging can access dev but not production
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: staging-policy
  namespace: staging
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow to staging and dev
  - to:
    - namespaceSelector:
        matchExpressions:
        - key: environment
          operator: In
          values:
          - staging
          - development
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### **Scenario 2: PCI-DSS Compliance Zone**

```yaml
# PCI cardholder data environment
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pci-cde-policy
  namespace: payment-processing
  labels:
    compliance: pci-dss
spec:
  podSelector:
    matchLabels:
      pci-scope: in-scope
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only from approved applications
  - from:
    - podSelector:
        matchLabels:
          pci-approved: "true"
    ports:
    - protocol: TCP
      port: 8443  # TLS only
  # Allow from payment gateway (external)
  - from:
    - ipBlock:
        cidr: 203.0.113.0/24  # Payment processor IPs
    ports:
    - protocol: TCP
      port: 8443
  egress:
  # Database access (encrypted)
  - to:
    - podSelector:
        matchLabels:
          app: encrypted-db
          pci-scope: in-scope
    ports:
    - protocol: TCP
      port: 5432
  # External payment API
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
  # Logging (audit trail)
  - to:
    - namespaceSelector:
        matchLabels:
          name: logging
    podSelector:
      matchLabels:
        app: audit-logs
    ports:
    - protocol: TCP
      port: 9200
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### **Scenario 3: Service Mesh Integration**

```yaml
# Istio sidecar communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: istio-mesh-policy
  namespace: istio-apps
spec:
  podSelector:
    matchLabels:
      istio-injection: enabled
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from Istio ingress gateway
  - from:
    - namespaceSelector:
        matchLabels:
          name: istio-system
    podSelector:
      matchLabels:
        app: istio-ingressgateway
  # Allow from other mesh pods
  - from:
    - podSelector:
        matchLabels:
          istio-injection: enabled
  egress:
  # Allow to mesh pods
  - to:
    - podSelector:
        matchLabels:
          istio-injection: enabled
  # Allow to Istio control plane
  - to:
    - namespaceSelector:
        matchLabels:
          name: istio-system
    ports:
    - protocol: TCP
      port: 15012  # xDS
    - protocol: TCP
      port: 15014  # Telemetry
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # External HTTPS
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
```

### **Scenario 4: Database Sharding**

```yaml
# Shard 1 policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-shard-1-policy
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: postgres
      shard: "1"
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Application tier
  - from:
    - namespaceSelector:
        matchLabels:
          name: myapp
    podSelector:
      matchLabels:
        db-shard: "1"
    ports:
    - protocol: TCP
      port: 5432
  # Replication from master
  - from:
    - podSelector:
        matchLabels:
          app: postgres
          role: master
          shard: "1"
    ports:
    - protocol: TCP
      port: 5432
  # Monitoring
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    podSelector:
      matchLabels:
        app: postgres-exporter
    ports:
    - protocol: TCP
      port: 9187
  egress:
  # Replication to replicas
  - to:
    - podSelector:
        matchLabels:
          app: postgres
          role: replica
          shard: "1"
    ports:
    - protocol: TCP
      port: 5432
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### **Scenario 5: Zero Trust Architecture**

```yaml
# Zero trust policy - explicit allow only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: zero-trust-app
  namespace: zero-trust
  annotations:
    security-model: zero-trust
spec:
  podSelector:
    matchLabels:
      app: secure-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only authenticated gateway
  - from:
    - podSelector:
        matchLabels:
          app: auth-gateway
          mtls-enabled: "true"
    ports:
    - protocol: TCP
      port: 8443
  egress:
  # Explicit service dependencies
  - to:
    - podSelector:
        matchLabels:
          app: auth-service
          mtls-enabled: "true"
    ports:
    - protocol: TCP
      port: 8443
  - to:
    - podSelector:
        matchLabels:
          app: data-service
          mtls-enabled: "true"
    ports:
    - protocol: TCP
      port: 8443
  # Logging (non-blocking)
  - to:
    - namespaceSelector:
        matchLabels:
          name: logging
    podSelector:
      matchLabels:
        app: log-aggregator
    ports:
    - protocol: TCP
      port: 9200
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # No other egress allowed - no internet access
```

### **Scenario 6: Canary Deployments**

```yaml
# Stable version policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: stable-version-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
      version: stable
  policyTypes:
  - Ingress
  ingress:
  # 90% of traffic from load balancer
  - from:
    - podSelector:
        matchLabels:
          app: load-balancer
    ports:
    - protocol: TCP
      port: 8080

---
# Canary version policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: canary-version-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
      version: canary
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # 10% of traffic from load balancer
  - from:
    - podSelector:
        matchLabels:
          app: load-balancer
    ports:
    - protocol: TCP
      port: 8080
  # Allow from testing tools
  - from:
    - namespaceSelector:
        matchLabels:
          name: testing
    podSelector:
      matchLabels:
        app: integration-tests
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Shadow traffic to stable for comparison
  - to:
    - podSelector:
        matchLabels:
          app: myapp
          version: stable
    ports:
    - protocol: TCP
      port: 8080
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏢 Multi-tenancy Policies**

### **Tenant Isolation Model**

```yaml
# Tenant A namespace isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-a-isolation
  namespace: tenant-a
  labels:
    tenant: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only from same tenant
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: tenant-a
  # Allow from shared ingress
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    podSelector:
      matchLabels:
        app: ingress-nginx
  egress:
  # Only to same tenant
  - to:
    - namespaceSelector:
        matchLabels:
          tenant: tenant-a
  # Shared services
  - to:
    - namespaceSelector:
        matchLabels:
          shared-service: "true"
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Internet access (if allowed)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8  # Block internal cluster networks
    ports:
    - protocol: TCP
      port: 443

---
# Shared service accessible to all tenants
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shared-service-policy
  namespace: shared-services
  labels:
    shared-service: "true"
spec:
  podSelector:
    matchLabels:
      app: shared-cache
  policyTypes:
  - Ingress
  ingress:
  # Allow from all tenant namespaces
  - from:
    - namespaceSelector:
        matchExpressions:
        - key: tenant
          operator: Exists
    ports:
    - protocol: TCP
      port: 6379
```

### **Resource Quota Integration**

```yaml
# Tenant with network policy and resource quotas
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-b
  labels:
    tenant: tenant-b
    network-policy-enforced: "true"

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-b-quota
  namespace: tenant-b
spec:
  hard:
    pods: "10"
    services: "5"
    networkpolicies.networking.k8s.io: "10"

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-b-default-deny
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-b-allow-dns
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### **Hierarchical Tenant Structure**

```yaml
# Parent tenant namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: parent-tenant-policy
  namespace: tenant-enterprise
  labels:
    tenant-type: parent
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from child tenants
  - from:
    - namespaceSelector:
        matchLabels:
          parent-tenant: tenant-enterprise
  egress:
  # Allow to child tenants
  - to:
    - namespaceSelector:
        matchLabels:
          parent-tenant: tenant-enterprise
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# Child tenant namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: child-tenant-policy
  namespace: tenant-enterprise-dev
  labels:
    tenant-type: child
    parent-tenant: tenant-enterprise
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from parent
  - from:
    - namespaceSelector:
        matchLabels:
          tenant-type: parent
          tenant: tenant-enterprise
  # Allow from same child namespace
  - from:
    - podSelector: {}
  egress:
  # Allow to parent
  - to:
    - namespaceSelector:
        matchLabels:
          tenant-type: parent
          tenant: tenant-enterprise
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔒 Security Best Practices**

### **Defense in Depth**

```yaml
# Layer 1: Default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: 01-default-deny
  namespace: secure-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Layer 2: Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: 02-allow-dns
  namespace: secure-app
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53

---
# Layer 3: Allow specific application flows
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: 03-app-communication
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: load-balancer
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
```

### **Principle of Least Privilege**

```yaml
# Minimal permissions for web tier
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-minimal-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      tier: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only necessary ports
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080  # Application port only
  egress:
  # Only to required services
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 8080
  # DNS only
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # No internet access, no other ports
```

### **Egress Filtering**

```yaml
# Strict egress control
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: strict-egress
  namespace: restricted
spec:
  podSelector:
    matchLabels:
      security: high
  policyTypes:
  - Egress
  egress:
  # Whitelist specific external APIs
  - to:
    - ipBlock:
        cidr: 198.51.100.0/24  # Specific API subnet
    ports:
    - protocol: TCP
      port: 443
  # Block cloud metadata services
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32  # AWS
        - 169.254.169.253/32  # AWS DNS
        - 169.254.169.123/32  # Azure
        - 127.0.0.1/32
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### **Audit Logging Integration**

```yaml
# Policy with audit annotations
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: audited-policy
  namespace: compliance
  annotations:
    audit.k8s.io/level: "RequestResponse"
    compliance.company.com/framework: "PCI-DSS"
    compliance.company.com/control: "1.2.1"
    description: "Restrict network access to cardholder data environment"
spec:
  podSelector:
    matchLabels:
      pci-scope: in-scope
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          pci-approved: "true"
    ports:
    - protocol: TCP
      port: 8443
  egress:
  # All egress logged
  - to:
    - podSelector:
        matchLabels:
          pci-scope: in-scope
```

### **Encryption Enforcement**

```yaml
# TLS-only policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tls-only
  namespace: secure
spec:
  podSelector:
    matchLabels:
      encryption: required
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only HTTPS
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 8443
    - protocol: TCP
      port: 443
  egress:
  # Only HTTPS outbound
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 8443
    - protocol: TCP
      port: 443
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting Guide**

### **Common Issues**

#### **Issue 1: Policy Not Applied**

```bash
# Check if CNI supports NetworkPolicy
kubectl get pods -n kube-system | grep -E "calico|cilium|weave"

# Verify policy exists
kubectl get networkpolicy -n myapp

# Check policy details
kubectl describe networkpolicy my-policy -n myapp

# Verify pod labels match
kubectl get pods -n myapp --show-labels

# Check if policy selects pods
kubectl get pods -n myapp -l app=myapp
```

**Validation Script:**
```bash
#!/bin/bash
NAMESPACE=$1
POLICY=$2

echo "=== Checking NetworkPolicy: $POLICY in $NAMESPACE ==="

# Get policy
kubectl get networkpolicy $POLICY -n $NAMESPACE -o yaml

# Get pod selector
POD_SELECTOR=$(kubectl get networkpolicy $POLICY -n $NAMESPACE -o jsonpath='{.spec.podSelector}')
echo -e "\nPod Selector: $POD_SELECTOR"

# Find matching pods
echo -e "\nMatching Pods:"
kubectl get pods -n $NAMESPACE --selector=$(echo $POD_SELECTOR | jq -r '.matchLabels | to_entries | map("\(.key)=\(.value)") | join(",")')

# Check CNI
echo -e "\nCNI Plugins:"
kubectl get pods -n kube-system -o wide | grep -E "calico|cilium|weave|flannel"
```

#### **Issue 2: DNS Not Working**

```yaml
# Missing DNS policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: myapp
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # CoreDNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # Alternative: allow to kube-dns service IP
  - to:
    - ipBlock:
        cidr: 10.96.0.10/32  # kube-dns service IP
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

**DNS Test:**
```bash
# Test DNS from pod
kubectl run -it --rm debug --image=busybox --restart=Never -n myapp -- nslookup kubernetes.default

# Check DNS policy
kubectl get networkpolicy -n myapp -o yaml | grep -A 20 "egress"

# Verify kube-dns labels
kubectl get pods -n kube-system -l k8s-app=kube-dns --show-labels
```

#### **Issue 3: Service Communication Blocked**

```bash
# Test connectivity
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -n myapp -- \
  curl -v http://backend-service:8080

# Check service endpoints
kubectl get endpoints backend-service -n myapp

# Verify network policy
kubectl get networkpolicy -n myapp -o yaml

# Check if egress is allowed
kubectl describe networkpolicy frontend-policy -n myapp | grep -A 10 "Egress"
```

**Connection Test Script:**
```bash
#!/bin/bash
SOURCE_NS=$1
SOURCE_POD=$2
DEST_SERVICE=$3
DEST_PORT=$4

echo "Testing connectivity from $SOURCE_NS/$SOURCE_POD to $DEST_SERVICE:$DEST_PORT"

# Test TCP connection
kubectl exec -n $SOURCE_NS $SOURCE_POD -- nc -zv $DEST_SERVICE $DEST_PORT

# Check network policies
echo -e "\n=== Source Pod Policies ==="
kubectl get networkpolicy -n $SOURCE_NS --field-selector metadata.name=$SOURCE_POD

echo -e "\n=== Destination Service Endpoints ==="
kubectl get endpoints $DEST_SERVICE -n $SOURCE_NS
```

#### **Issue 4: External Traffic Blocked**

```yaml
# Allow specific external IPs
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  # Allow to specific external API
  - to:
    - ipBlock:
        cidr: 198.51.100.0/24
    ports:
    - protocol: TCP
      port: 443
  # Allow all HTTPS (be careful!)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### **Debugging Commands**

```bash
# List all network policies
kubectl get networkpolicies --all-namespaces

# Describe specific policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# Get policy YAML
kubectl get networkpolicy <policy-name> -n <namespace> -o yaml

# Check pod labels
kubectl get pods -n <namespace> --show-labels

# Test pod connectivity
kubectl exec -n <namespace> <pod-name> -- curl -v http://target:port

# Check CNI logs (Calico)
kubectl logs -n kube-system -l k8s-app=calico-node --tail=100

# Check CNI logs (Cilium)
kubectl logs -n kube-system -l k8s-app=cilium --tail=100

# View iptables rules (if accessible)
kubectl exec -n kube-system <calico-node-pod> -- iptables-save | grep <pod-ip>
```

### **Policy Validation Tools**

```bash
# Using kubectl network policy plugin
kubectl np install

# Check connectivity
kubectl np check <source-pod> <dest-pod> -n <namespace>

# Analyze policies
kubectl np analyze -n <namespace>

# Using Cilium CLI
cilium connectivity test

# Check policy enforcement
cilium policy get

# Using Calico CLI
calicoctl get networkpolicy -n <namespace>

# Check policy effectiveness
calicoctl get workloadendpoint -n <namespace>
```

### **Monitoring Network Policy Events**

```bash
# Watch network policy events
kubectl get events -n <namespace> --field-selector involvedObject.kind=NetworkPolicy -w

# Audit logs (if enabled)
kubectl logs -n kube-system kube-apiserver-* | grep NetworkPolicy

# CNI-specific monitoring
# Calico
kubectl logs -n kube-system -l k8s-app=calico-node | grep -i "policy"

# Cilium
kubectl logs -n kube-system -l k8s-app=cilium | grep -i "policy"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ Performance Considerations**

### **Policy Optimization**

#### **Minimize Policy Count**

```yaml
# BAD: Too many policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend

---
# GOOD: Combined policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-tiers
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
  - from:
    - podSelector:
        matchLabels:
          tier: backend
```

#### **Efficient Selectors**

```yaml
# BAD: Complex expressions
spec:
  podSelector:
    matchExpressions:
    - key: app
      operator: In
      values: [app1, app2, app3, app4, app5]
    - key: env
      operator: In
      values: [prod, staging]
    - key: version
      operator: In
      values: [v1, v2]

# GOOD: Simple labels
spec:
  podSelector:
    matchLabels:
      policy-group: backend-services
      environment: production
```

### **Scale Considerations**

```go
// CNI policy reconciliation loop
// Location: pkg/controller/networkpolicy/networkpolicy_controller.go

func (c *Controller) syncNetworkPolicy(key string) error {
    startTime := time.Now()
    defer func() {
        klog.V(4).Infof("Finished syncing network policy %q (%v)", key, time.Since(startTime))
    }()

    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }

    policy, err := c.policyLister.NetworkPolicies(namespace).Get(name)
    if errors.IsNotFound(err) {
        // Policy deleted
        return c.handlePolicyDelete(namespace, name)
    }
    if err != nil {
        return err
    }

    // Get all pods matching the selector
    pods, err := c.getPodsForPolicy(policy)
    if err != nil {
        return err
    }

    // Update rules for each pod
    for _, pod := range pods {
        if err := c.updatePodRules(pod, policy); err != nil {
            return err
        }
    }

    return nil
}
```

### **Resource Limits**

```yaml
# NetworkPolicy controller resources
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calico-kube-controllers
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: calico-kube-controllers
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        env:
        - name: DATASTORE_TYPE
          value: kubernetes
        # Performance tuning
        - name: RECONCILER_PERIOD
          value: "5m"
        - name: MAX_INFLIGHT_RECONCILES
          value: "10"
```

### **Caching Strategy**

```go
// Policy cache implementation
// Location: pkg/controller/networkpolicy/network_policy_cache.go

type PolicyCache struct {
    mu           sync.RWMutex
    policies     map[string]*networking.NetworkPolicy
    podPolicies  map[string][]*networking.NetworkPolicy  // pod UID -> policies
    indexedPods  map[string][]string                     // policy key -> pod UIDs
}

func (c *PolicyCache) GetPoliciesForPod(pod *v1.Pod) []*networking.NetworkPolicy {
    c.mu.RLock()
    defer c.mu.RUnlock()

    podKey := string(pod.UID)
    return c.podPolicies[podKey]
}

func (c *PolicyCache) UpdatePolicy(policy *networking.NetworkPolicy) {
    c.mu.Lock()
    defer c.mu.Unlock()

    key := getPolicyKey(policy)
    c.policies[key] = policy

    // Reindex affected pods
    c.reindexPods(policy)
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌟 Real-World Examples**

### **Example 1: E-commerce Platform**

```yaml
# Frontend (Web UI)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ecommerce-frontend
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      tier: frontend
      app: webui
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Public internet access
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
  egress:
  # API Gateway
  - to:
    - podSelector:
        matchLabels:
          tier: api
          app: gateway
    ports:
    - protocol: TCP
      port: 8080
  # CDN for static assets
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# API Gateway
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ecommerce-api-gateway
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      tier: api
      app: gateway
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # From frontend
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  # From mobile app (external)
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 8443
  egress:
  # Product service
  - to:
    - podSelector:
        matchLabels:
          service: product
    ports:
    - protocol: TCP
      port: 8080
  # Order service
  - to:
    - podSelector:
        matchLabels:
          service: order
    ports:
    - protocol: TCP
      port: 8080
  # User service
  - to:
    - podSelector:
        matchLabels:
          service: user
    ports:
    - protocol: TCP
      port: 8080
  # Payment service
  - to:
    - podSelector:
        matchLabels:
          service: payment
    ports:
    - protocol: TCP
      port: 8080
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# Payment Service (PCI compliance)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ecommerce-payment
  namespace: ecommerce
  labels:
    pci-compliant: "true"
spec:
  podSelector:
    matchLabels:
      service: payment
      pci-scope: in-scope
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only from API gateway
  - from:
    - podSelector:
        matchLabels:
          tier: api
          app: gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Payment processor (external)
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24  # Stripe/PayPal IPs
    ports:
    - protocol: TCP
      port: 443
  # Database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
          db: payment
    ports:
    - protocol: TCP
      port: 5432
  # Audit logging
  - to:
    - namespaceSelector:
        matchLabels:
          name: logging
    podSelector:
      matchLabels:
        app: audit-logs
    ports:
    - protocol: TCP
      port: 9200
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# Order Service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ecommerce-order
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      service: order
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # From API gateway
  - from:
    - podSelector:
        matchLabels:
          tier: api
          app: gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Product service (check inventory)
  - to:
    - podSelector:
        matchLabels:
          service: product
    ports:
    - protocol: TCP
      port: 8080
  # Payment service
  - to:
    - podSelector:
        matchLabels:
          service: payment
    ports:
    - protocol: TCP
      port: 8080
  # Notification service
  - to:
    - podSelector:
        matchLabels:
          service: notification
    ports:
    - protocol: TCP
      port: 8080
  # Database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
          db: order
    ports:
    - protocol: TCP
      port: 5432
  # Cache
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### **Example 2: Multi-Cluster Federation**

```yaml
# Cluster A - Primary
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: federation-primary
  namespace: federated-app
  annotations:
    cluster: primary
    region: us-east-1
spec:
  podSelector:
    matchLabels:
      app: federated-service
      cluster-role: primary
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # From local clients
  - from:
    - podSelector:
        matchLabels:
          access: federated-service
    ports:
    - protocol: TCP
      port: 8080
  # From secondary cluster (via VPN)
  - from:
    - ipBlock:
        cidr: 10.100.0.0/16  # Cluster B CIDR
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # To local database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  # To secondary cluster
  - to:
    - ipBlock:
        cidr: 10.100.0.0/16
    ports:
    - protocol: TCP
      port: 8080
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### **Example 3: Development Environment**

```yaml
# Development namespace with relaxed policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dev-environment
  namespace: development
  labels:
    environment: dev
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from developers
  - from:
    - ipBlock:
        cidr: 10.200.0.0/16  # VPN range
  # Allow from CI/CD
  - from:
    - namespaceSelector:
        matchLabels:
          name: cicd
  egress:
  # Allow to everything in dev
  - to:
    - podSelector: {}
  # Allow to shared services
  - to:
    - namespaceSelector:
        matchLabels:
          shared: "true"
  # Allow internet (for package downloads)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Summary**

### **Key Takeaways**

1. **Default Behavior**: Start with default deny and explicitly allow required traffic
2. **Selectors**: Use pod and namespace selectors effectively for dynamic policy application
3. **Layering**: Apply multiple policies for defense in depth
4. **DNS**: Always remember to allow DNS traffic in egress policies
5. **Testing**: Thoroughly test policies in non-production before applying to production
6. **Documentation**: Document policy intent and dependencies
7. **Monitoring**: Implement monitoring and alerting for policy violations
8. **Performance**: Optimize policies to minimize CNI overhead

### **Code References**

```plaintext
NetworkPolicy API:
  pkg/apis/networking/types.go
  pkg/apis/networking/validation/validation.go

Controller Implementation:
  pkg/controller/networkpolicy/

API Server Integration:
  staging/src/k8s.io/apiserver/pkg/admission/plugin/policy/

Validation:
  pkg/registry/networking/networkpolicy/strategy.go
```

### **Next Steps**

- Explore CNI-specific implementations in [02-policy-controllers.md](./02-policy-controllers.md)
- Learn about service mesh integration in [03-service-mesh-integration.md](./03-service-mesh-integration.md)
- Study eBPF networking in [04-ebpf-networking.md](./04-ebpf-networking.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
