# **Service Mesh Integration with Kubernetes Networking**

**Comprehensive Guide to Istio, Linkerd, and CNI Integration**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Table of Contents**

1. [Overview](#overview)
2. [Service Mesh Fundamentals](#service-mesh-fundamentals)
3. [Istio Architecture](#istio-architecture)
4. [Linkerd Architecture](#linkerd-architecture)
5. [Cilium Service Mesh](#cilium-service-mesh)
6. [NetworkPolicy Integration](#networkpolicy-integration)
7. [mTLS and Security](#mtls-and-security)
8. [Traffic Management](#traffic-management)
9. [Observability](#observability)
10. [Performance Comparison](#performance-comparison)
11. [Migration Strategies](#migration-strategies)
12. [Best Practices](#best-practices)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Overview**

### **What is a Service Mesh?**

A service mesh is a dedicated infrastructure layer for handling service-to-service communication, providing:
- **Traffic management**: Load balancing, routing, traffic splitting
- **Security**: mTLS, authentication, authorization
- **Observability**: Metrics, traces, logs
- **Resilience**: Retries, timeouts, circuit breaking

### **Service Mesh vs NetworkPolicy**

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer Comparison                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  L7 (Application)                                           │
│  ├─ Service Mesh                                            │
│  │  ├─ HTTP routing                                         │
│  │  ├─ gRPC load balancing                                  │
│  │  ├─ Request-level policies                               │
│  │  └─ Distributed tracing                                  │
│  │                                                           │
│  L4 (Transport)                                             │
│  ├─ Service Mesh + NetworkPolicy                            │
│  │  ├─ mTLS encryption                                      │
│  │  ├─ TCP/UDP policies                                     │
│  │  └─ Port-level filtering                                 │
│  │                                                           │
│  L3 (Network)                                               │
│  └─ NetworkPolicy (CNI)                                     │
│     ├─ IP-based filtering                                   │
│     ├─ Pod isolation                                        │
│     └─ Namespace segmentation                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### **Architecture Patterns**

```
Pattern 1: Sidecar (Istio, Linkerd)
┌─────────────────────────────────┐
│          Pod                     │
│  ┌──────────┐    ┌───────────┐ │
│  │   App    │◄──►│  Sidecar  │ │
│  │Container │    │   Proxy   │ │
│  └──────────┘    └─────┬─────┘ │
└────────────────────────┼────────┘
                         │
                         ▼
              ┌──────────────────┐
              │  Control Plane   │
              │  (Istiod/Linkerd)│
              └──────────────────┘

Pattern 2: Node Proxy (Cilium Service Mesh)
┌─────────────────────────────────┐
│          Pod                     │
│  ┌──────────┐                   │
│  │   App    │                   │
│  │Container │                   │
│  └─────┬────┘                   │
└────────┼────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│   Cilium Agent (per node)       │
│   - eBPF sockmap redirection    │
│   - L7 proxy (Envoy)            │
│   - Identity-based policy       │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│   Cilium Operator               │
└─────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏗️ Service Mesh Fundamentals**

### **Core Components**

```go
// Generic service mesh components
type ServiceMesh interface {
    // Data plane - handles actual traffic
    DataPlane() DataPlaneInterface

    // Control plane - configures data plane
    ControlPlane() ControlPlaneInterface

    // Policy enforcement
    PolicyEngine() PolicyInterface
}

type DataPlaneInterface interface {
    // Proxy configuration
    ConfigureProxy(pod *v1.Pod, config ProxyConfig) error

    // Traffic interception
    InterceptTraffic(rules []TrafficRule) error

    // Load balancing
    LoadBalance(endpoints []Endpoint, algorithm LBAlgorithm) Endpoint
}

type ControlPlaneInterface interface {
    // Service discovery
    DiscoverServices() ([]Service, error)

    // Certificate management
    IssueCertificate(identity string) (*Certificate, error)

    // Configuration push
    PushConfiguration(proxies []Proxy, config Configuration) error
}
```

### **Traffic Flow**

```
Without Service Mesh:
┌──────────┐                    ┌──────────┐
│  Pod A   │───────────────────►│  Pod B   │
│  App:8080│    Direct TCP      │  App:8080│
└──────────┘                    └──────────┘

With Service Mesh (Sidecar):
┌───────────────────┐          ┌───────────────────┐
│  Pod A            │          │  Pod B            │
│ ┌────┐   ┌─────┐ │          │ ┌─────┐   ┌────┐ │
│ │App │──►│Proxy│─┼──────────┼►│Proxy│──►│App │ │
│ │:8080  │Envoy│ │  mTLS    │ │Envoy│   │:8080│
│ └────┘   └─────┘ │          │ └─────┘   └────┘ │
└───────────────────┘          └───────────────────┘
         │                              ▲
         └──────────────────────────────┘
              Telemetry, Metrics, Traces
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⛵ Istio Architecture**

### **Components**

```
┌─────────────────────────────────────────────────────────────┐
│                    Istiod (Control Plane)                    │
│  ┌────────────┐  ┌───────────┐  ┌──────────────┐          │
│  │   Pilot    │  │  Citadel  │  │    Galley    │          │
│  │(Discovery) │  │   (CA)    │  │(Configuration)│          │
│  └────────────┘  └───────────┘  └──────────────┘          │
└──────────────────────┬──────────────────────────────────────┘
                       │ xDS API
                       ▼
        ┌──────────────────────────────────┐
        │      Envoy Proxies (Data Plane)   │
        │  ┌────────┐  ┌────────┐  ┌─────┐ │
        │  │Pod A   │  │Pod B   │  │Pod C│ │
        │  │Envoy   │  │Envoy   │  │Envoy│ │
        │  └────────┘  └────────┘  └─────┘ │
        └──────────────────────────────────┘
```

### **Installation**

```bash
# Install Istio CLI
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio with default profile
istioctl install --set profile=default -y

# Verify installation
kubectl get pods -n istio-system

# Expected output:
# NAME                                   READY   STATUS
# istiod-xxxxx                          1/1     Running
# istio-ingressgateway-xxxxx            1/1     Running
```

**Installation Profiles:**
```yaml
# Minimal profile (for testing)
istioctl install --set profile=minimal

# Demo profile (with addons)
istioctl install --set profile=demo

# Production profile (HA)
istioctl install --set profile=production

# Custom configuration
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istio-controlplane
spec:
  profile: default

  # Resource allocation
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 4Gi

    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 2000m
            memory: 1Gi
        service:
          type: LoadBalancer

  # mTLS settings
  meshConfig:
    enableAutoMtls: true
    defaultConfig:
      holdApplicationUntilProxyStarts: true

  # Telemetry
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 2000m
            memory: 1Gi

      # Tracing
      tracer:
        zipkin:
          address: zipkin.istio-system:9411
```

### **Sidecar Injection**

```yaml
# Automatic injection (namespace label)
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    istio-injection: enabled

---
# Manual injection (pod annotation)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
```

**Injected Pod Structure:**
```yaml
# After injection
apiVersion: v1
kind: Pod
metadata:
  name: myapp-xxxxx
  annotations:
    sidecar.istio.io/status: '{"version":"...","initContainers":["istio-init"],"containers":["istio-proxy"]}'
spec:
  # Init container sets up iptables
  initContainers:
  - name: istio-init
    image: istio/proxyv2:1.17.0
    command:
    - istio-iptables
    args:
    - -p
    - "15001"  # Envoy inbound port
    - -u
    - "1337"   # Envoy UID
    - -m
    - REDIRECT
    - -i
    - '*'      # Capture all inbound
    - -b
    - '*'      # Capture all ports
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_RAW

  containers:
  # Application container
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080

  # Istio proxy sidecar
  - name: istio-proxy
    image: istio/proxyv2:1.17.0
    args:
    - proxy
    - sidecar
    - --domain
    - $(POD_NAMESPACE).svc.cluster.local
    - --proxyLogLevel=warning
    - --proxyComponentLogLevel=misc:error
    - --log_output_level=default:info
    env:
    - name: ISTIO_META_MESH_ID
      value: cluster.local
    - name: TRUST_DOMAIN
      value: cluster.local
    ports:
    - containerPort: 15090  # Prometheus metrics
      name: http-envoy-prom
    - containerPort: 15021  # Health check
      name: status-port
    - containerPort: 15001  # Envoy admin
      name: admin-port
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 2000m
        memory: 1Gi
    volumeMounts:
    - name: workload-socket
      mountPath: /var/run/secrets/workload-spiffe-uds
    - name: credential-socket
      mountPath: /var/run/secrets/credential-uds
    - name: istiod-ca-cert
      mountPath: /var/run/secrets/istio
    - name: istio-token
      mountPath: /var/run/secrets/tokens

  volumes:
  - name: workload-socket
    emptyDir: {}
  - name: credential-socket
    emptyDir: {}
  - name: istiod-ca-cert
    configMap:
      name: istio-ca-root-cert
  - name: istio-token
    projected:
      sources:
      - serviceAccountToken:
          path: istio-token
          expirationSeconds: 43200
          audience: istio-ca
```

### **Traffic Management**

#### **VirtualService**

```yaml
# HTTP routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-routes
  namespace: myapp
spec:
  hosts:
  - myapp.example.com
  - myapp.myapp.svc.cluster.local

  gateways:
  - myapp-gateway
  - mesh  # Internal mesh traffic

  http:
  # Canary routing (10% to v2)
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: myapp.myapp.svc.cluster.local
        subset: v2
      weight: 100

  # Default routing (90% v1, 10% v2)
  - route:
    - destination:
        host: myapp.myapp.svc.cluster.local
        subset: v1
      weight: 90
    - destination:
        host: myapp.myapp.svc.cluster.local
        subset: v2
      weight: 10

    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure

    # Timeout
    timeout: 10s

    # Fault injection (testing)
    fault:
      delay:
        percentage:
          value: 1
        fixedDelay: 5s
```

#### **DestinationRule**

```yaml
# Load balancing and circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp-destination
  namespace: myapp
spec:
  host: myapp.myapp.svc.cluster.local

  # Traffic policy
  trafficPolicy:
    # Load balancer
    loadBalancer:
      consistentHash:
        httpCookie:
          name: user
          ttl: 0s

    # Connection pool
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 2

    # Circuit breaker
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 40

  # Subsets for version routing
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN

  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: LEAST_REQUEST
```

#### **Gateway**

```yaml
# Ingress gateway
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: myapp-gateway
  namespace: myapp
spec:
  selector:
    istio: ingressgateway

  servers:
  # HTTPS
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: myapp-tls
    hosts:
    - myapp.example.com

  # HTTP (redirect to HTTPS)
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - myapp.example.com
    tls:
      httpsRedirect: true
```

### **Security Policies**

#### **PeerAuthentication**

```yaml
# Enforce mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: myapp
spec:
  mtls:
    mode: STRICT  # STRICT, PERMISSIVE, DISABLE

---
# Per-port mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: myapp-auth
  namespace: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  mtls:
    mode: STRICT
  portLevelMtls:
    8080:
      mode: STRICT
    9090:
      mode: DISABLE  # Metrics port
```

#### **AuthorizationPolicy**

```yaml
# L7 authorization
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: myapp-authz
  namespace: myapp
spec:
  selector:
    matchLabels:
      app: myapp

  action: ALLOW  # ALLOW, DENY, AUDIT, CUSTOM

  rules:
  # Allow from frontend
  - from:
    - source:
        principals:
        - cluster.local/ns/myapp/sa/frontend

    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
        ports: ["8080"]

  # Allow from monitoring
  - from:
    - source:
        namespaces: ["monitoring"]

    to:
    - operation:
        methods: ["GET"]
        paths: ["/metrics"]
        ports: ["9090"]

---
# Deny all by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: myapp
spec:
  # No rules = deny all
```

### **Integration with NetworkPolicy**

```yaml
# Layer 3/4: NetworkPolicy (CNI)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-network-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
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
  # Allow from same namespace
  - from:
    - podSelector: {}
  egress:
  # Allow to istiod (control plane)
  - to:
    - namespaceSelector:
        matchLabels:
          name: istio-system
    ports:
    - protocol: TCP
      port: 15012  # xDS

---
# Layer 7: AuthorizationPolicy (Istio)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: myapp-l7-policy
  namespace: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/myapp/sa/frontend
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/v1/*"]
    when:
    - key: request.headers[user-agent]
      values: ["my-app/*"]
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Linkerd Architecture**

### **Components**

```
┌─────────────────────────────────────────────────────────────┐
│              Linkerd Control Plane                           │
│  ┌────────────┐  ┌───────────┐  ┌──────────────┐          │
│  │ Destination│  │  Identity │  │    Proxy     │          │
│  │   (discovery)│  │   (mTLS)  │  │  Injector  │          │
│  └────────────┘  └───────────┘  └──────────────┘          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
        ┌──────────────────────────────────┐
        │   Linkerd2-proxy (Data Plane)    │
        │  ┌────────┐  ┌────────┐  ┌─────┐ │
        │  │Pod A   │  │Pod B   │  │Pod C│ │
        │  │Proxy   │  │Proxy   │  │Proxy│ │
        │  └────────┘  └────────┘  └─────┘ │
        └──────────────────────────────────┘
```

### **Installation**

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# Validate cluster
linkerd check --pre

# Install CRDs
linkerd install --crds | kubectl apply -f -

# Install control plane
linkerd install | kubectl apply -f -

# Verify
linkerd check

# Install viz extension (observability)
linkerd viz install | kubectl apply -f -
```

**Production Installation:**
```yaml
# High-availability control plane
apiVersion: v1
kind: Namespace
metadata:
  name: linkerd
  annotations:
    linkerd.io/inject: disabled

---
# Control plane configuration
apiVersion: linkerd.io/v1alpha2
kind: ControlPlane
metadata:
  name: linkerd-control-plane
  namespace: linkerd
spec:
  # HA configuration
  controllerReplicas: 3

  # Identity (mTLS)
  identityTrustDomain: cluster.local
  identityTrustAnchorsPEM: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

  # Proxy configuration
  proxy:
    resources:
      cpu:
        request: 100m
        limit: 1000m
      memory:
        request: 20Mi
        limit: 250Mi

    # Disable for system namespaces
    ignoreInboundPorts: "25,443,587,3306,5432,6379,9200,11211"
    ignoreOutboundPorts: "25,443,587,3306,5432,6379,9200,11211"

  # Destination service
  destinationResources:
    cpu:
      request: 100m
      limit: 1000m
    memory:
      request: 50Mi
      limit: 250Mi

  # Identity service
  identityResources:
    cpu:
      request: 100m
      limit: 1000m
    memory:
      request: 10Mi
      limit: 100Mi
```

### **Proxy Injection**

```yaml
# Automatic injection (namespace annotation)
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  annotations:
    linkerd.io/inject: enabled

---
# Selective injection (pod annotation)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  template:
    metadata:
      annotations:
        linkerd.io/inject: enabled
    spec:
      containers:
      - name: app
        image: myapp:latest
```

**Injected Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    linkerd.io/inject: enabled
    linkerd.io/proxy-version: stable-2.12.0
spec:
  initContainers:
  # Init container for iptables setup
  - name: linkerd-init
    image: cr.l5d.io/linkerd/proxy-init:v2.0.0
    args:
    - --incoming-proxy-port
    - "4143"
    - --outgoing-proxy-port
    - "4140"
    - --proxy-uid
    - "2102"
    - --inbound-ports-to-ignore
    - "4190,4191"
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_RAW
      privileged: false

  containers:
  - name: app
    image: myapp:latest

  # Linkerd proxy
  - name: linkerd-proxy
    image: cr.l5d.io/linkerd/proxy:stable-2.12.0
    args:
    - --log-level
    - warn,linkerd=info
    env:
    - name: LINKERD2_PROXY_LOG
      value: warn,linkerd=info
    - name: LINKERD2_PROXY_DESTINATION_SVC_ADDR
      value: linkerd-dst.linkerd.svc.cluster.local:8086
    - name: LINKERD2_PROXY_IDENTITY_DIR
      value: /var/run/linkerd/identity/end-entity
    - name: LINKERD2_PROXY_INBOUND_LISTEN_ADDR
      value: 0.0.0.0:4143
    - name: LINKERD2_PROXY_OUTBOUND_LISTEN_ADDR
      value: 127.0.0.1:4140
    - name: LINKERD2_PROXY_ADMIN_LISTEN_ADDR
      value: 0.0.0.0:4191
    - name: LINKERD2_PROXY_TAP_SVC_NAME
      value: linkerd-tap.linkerd.svc.cluster.local
    ports:
    - containerPort: 4143
      name: linkerd-proxy
    - containerPort: 4191
      name: linkerd-admin
    resources:
      limits:
        cpu: 1000m
        memory: 250Mi
      requests:
        cpu: 100m
        memory: 20Mi
    volumeMounts:
    - mountPath: /var/run/linkerd/identity/end-entity
      name: linkerd-identity-end-entity

  volumes:
  - name: linkerd-identity-end-entity
    emptyDir:
      medium: Memory
```

### **Traffic Policies**

#### **ServerAuthorization**

```yaml
# L7 authorization
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  name: myapp-authz
  namespace: myapp
spec:
  # Target server
  server:
    name: myapp-server
    selector:
      matchLabels:
        app: myapp

  # Allowed clients
  client:
    # By service account
    meshTLS:
      serviceAccounts:
      - name: frontend
        namespace: myapp

    # By identity
    meshTLS:
      identities:
      - "frontend.myapp.serviceaccount.identity.linkerd.cluster.local"

---
# Server definition
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  name: myapp-server
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
  port: 8080
  proxyProtocol: HTTP/2
```

#### **HTTPRoute**

```yaml
# HTTP routing (Gateway API)
apiVersion: policy.linkerd.io/v1alpha1
kind: HTTPRoute
metadata:
  name: myapp-routes
  namespace: myapp
spec:
  parentRefs:
  - name: myapp
    kind: Service
    group: core
    port: 8080

  rules:
  # Canary routing
  - matches:
    - headers:
      - name: x-canary
        value: "true"
    backendRefs:
    - name: myapp-v2
      port: 8080

  # Default routing
  - backendRefs:
    - name: myapp-v1
      port: 8080
      weight: 90
    - name: myapp-v2
      port: 8080
      weight: 10

    timeouts:
      request: 10s

    retry:
      codes: [500, 502, 503, 504]
      max: 3
```

### **mTLS Configuration**

```bash
# Check mTLS status
linkerd viz edges deployment -n myapp

# View certificates
linkerd viz tap deploy/myapp -n myapp -o wide

# Identity validation
linkerd identity -n myapp
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🐝 Cilium Service Mesh**

### **Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│          Cilium Operator (Control Plane)                     │
│  - Service discovery                                         │
│  - TLS certificate management                                │
│  - Ingress/Gateway API                                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
      ┌─────────────────────────────────────────┐
      │      Cilium Agent (per node)            │
      │  ┌──────────────────────────────────┐  │
      │  │    eBPF Dataplane                │  │
      │  │  - Socket-level redirection      │  │
      │  │  - L7 proxy (Envoy, optional)    │  │
      │  │  - Identity-based policy         │  │
      │  └──────────────────────────────────┘  │
      └─────────────────────────────────────────┘
```

### **Installation**

```bash
# Install with service mesh features
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=strict \
  --set l7Proxy=true \
  --set encryption.enabled=true \
  --set encryption.type=wireguard

# Enable Hubble for observability
cilium hubble enable --ui
```

**Configuration:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cilium-config
  namespace: kube-system
data:
  # Enable L7 proxy
  enable-l7-proxy: "true"

  # Proxy port
  proxy-prometheus-port: "9095"

  # Enable service mesh features
  enable-envoy-config: "true"

  # Enable ingress controller
  enable-ingress-controller: "true"

  # Encryption
  enable-wireguard: "true"

  # Identity allocation
  identity-allocation-mode: "crd"

  # kube-proxy replacement
  kube-proxy-replacement: "strict"
```

### **L7 Policy**

```yaml
# HTTP-aware policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: myapp-l7-policy
  namespace: myapp
spec:
  endpointSelector:
    matchLabels:
      app: myapp

  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend

    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP

      rules:
        http:
        # Allow GET /api/*
        - method: "GET"
          path: "/api/.*"

        # Allow POST /api/data with JSON
        - method: "POST"
          path: "/api/data"
          headers:
          - "Content-Type: application/json"

        # Deny admin endpoints
        - method: ".*"
          path: "/admin/.*"
          action: DENY

  # L7 visibility
  - toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - {}  # Log all HTTP traffic
```

### **Service Mesh without Sidecars**

```
Traditional Sidecar (Istio/Linkerd):
┌─────────────────────────────────────┐
│  Pod (2 containers, more memory)    │
│  ┌────────┐       ┌──────────────┐ │
│  │  App   │◄─────►│   Sidecar    │ │
│  │100m CPU│       │  Proxy       │ │
│  │256Mi   │       │  100m CPU    │ │
│  │        │       │  128Mi RAM   │ │
│  └────────┘       └──────────────┘ │
└─────────────────────────────────────┘
Total: 200m CPU, 384Mi RAM per pod

Cilium (eBPF):
┌─────────────────────────────────────┐
│  Pod (1 container, less memory)    │
│  ┌────────┐                         │
│  │  App   │                         │
│  │100m CPU│                         │
│  │256Mi   │                         │
│  └───┬────┘                         │
└──────┼──────────────────────────────┘
       │ eBPF sockmap redirect
       ▼
┌──────────────────────────────────────┐
│  Cilium Agent (shared per node)     │
│  - Handles all pods on node          │
│  - Optional Envoy for L7             │
└──────────────────────────────────────┘
Total: 100m CPU, 256Mi RAM per pod
       + shared Cilium agent overhead
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔐 mTLS and Security**

### **Certificate Management**

**Istio (SPIFFE/SPIRE):**
```yaml
# Certificate rotation
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
  namespace: istio-system
data:
  mesh: |
    # Certificate lifetime
    defaultConfig:
      proxyMetadata:
        ISTIO_META_CERT_VALIDITY_DURATION: 24h

# View certificates
$ istioctl proxy-config secret deployment/myapp -n myapp
RESOURCE NAME     TYPE           STATUS     VALID CERT     SERIAL NUMBER
default           Cert Chain     ACTIVE     true           329939034...
ROOTCA            CA             ACTIVE     true           165564612...
```

**Linkerd (linkerd-identity):**
```bash
# Check certificate expiry
linkerd check --proxy

# Rotate certificates
linkerd upgrade --crds | kubectl apply -f -
linkerd upgrade | kubectl apply -f -

# View identity
linkerd identity -n myapp
```

**Cilium (cert-manager integration):**
```yaml
# Use cert-manager for mTLS
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cilium-ca
  namespace: kube-system
spec:
  secretName: cilium-ca
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  commonName: "Cilium CA"
  isCA: true
```

### **Zero Trust Architecture**

```yaml
# Deny all by default (Istio)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: myapp
spec:
  {}  # No rules = deny all

---
# Explicit allow
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: myapp
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/myapp/sa/frontend
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚦 Traffic Management**

### **Progressive Delivery**

**Canary Deployments:**
```yaml
# Istio canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-canary
spec:
  hosts:
  - myapp
  http:
  - match:
    - headers:
        x-canary-user:
          exact: "true"
    route:
    - destination:
        host: myapp
        subset: v2
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 95
    - destination:
        host: myapp
        subset: v2
      weight: 5
```

**Blue/Green Deployments:**
```yaml
# Switch traffic instantly
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-bluegreen
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: green  # Switch to green
      weight: 100
```

### **Traffic Mirroring (Shadow Traffic)**

```yaml
# Istio traffic mirroring
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-mirror
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 100
    mirror:
      host: myapp
      subset: v2
    mirrorPercentage:
      value: 100  # Mirror 100% of traffic
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Observability**

### **Istio (Kiali, Jaeger, Prometheus)**

```bash
# Install addons
kubectl apply -f samples/addons/

# Access Kiali dashboard
istioctl dashboard kiali

# Jaeger tracing
istioctl dashboard jaeger

# Grafana metrics
istioctl dashboard grafana

# Prometheus
istioctl dashboard prometheus
```

**Distributed Tracing:**
```yaml
# Enable tracing
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100  # 100% sampling (reduce in production)
        zipkin:
          address: zipkin.istio-system:9411
```

### **Linkerd (Viz, Jaeger)**

```bash
# Install viz extension
linkerd viz install | kubectl apply -f -

# Dashboard
linkerd viz dashboard

# Top command
linkerd viz top deploy -n myapp

# Tap live traffic
linkerd viz tap deploy/myapp -n myapp

# Service profile
linkerd viz routes deploy/myapp -n myapp
```

### **Cilium (Hubble)**

```bash
# Enable Hubble
cilium hubble enable --ui

# Port-forward UI
cilium hubble ui

# CLI observability
hubble observe --namespace myapp

# Service map
hubble observe --namespace myapp -o json | \
  jq -r '.flow | "\(.source.namespace)/\(.source.pod_name) -> \(.destination.namespace)/\(.destination.pod_name)"' | \
  sort | uniq
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ Performance Comparison**

### **Latency Impact**

```
┌─────────────────────────────────────────────────────────────┐
│  p50 Latency (HTTP request)                                 │
├─────────────────────────────────────────────────────────────┤
│  Baseline (no mesh):      1.2ms  [▓░░░░░░░░░░]             │
│  Cilium (eBPF):           1.5ms  [▓░░░░░░░░░░]             │
│  Linkerd:                 2.8ms  [▓▓░░░░░░░░░]             │
│  Istio:                   3.5ms  [▓▓▓░░░░░░░░]             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  p99 Latency                                                │
├─────────────────────────────────────────────────────────────┤
│  Baseline:                5.0ms  [▓▓░░░░░░░░░]             │
│  Cilium:                  6.5ms  [▓▓▓░░░░░░░░]             │
│  Linkerd:                 12ms   [▓▓▓▓▓░░░░░░]             │
│  Istio:                   18ms   [▓▓▓▓▓▓▓░░░░]             │
└─────────────────────────────────────────────────────────────┘
```

### **Resource Overhead**

```
Per-Pod Overhead:
┌──────────────┬──────────┬──────────┬──────────────┐
│ Mesh         │ CPU (m)  │ Memory   │ Extra        │
│              │          │ (Mi)     │ Containers   │
├──────────────┼──────────┼──────────┼──────────────┤
│ None         │ 0        │ 0        │ 0            │
│ Cilium       │ 0*       │ 0*       │ 0 (shared)   │
│ Linkerd      │ 100      │ 20       │ 1 (proxy)    │
│ Istio        │ 100      │ 128      │ 1 (proxy)    │
└──────────────┴──────────┴──────────┴──────────────┘

* Cilium overhead is per-node, not per-pod

Node-Level Overhead:
┌──────────────┬──────────┬──────────┐
│ Mesh         │ CPU      │ Memory   │
├──────────────┼──────────┼──────────┤
│ Cilium       │ 500m     │ 1Gi      │
│ Linkerd      │ 200m     │ 500Mi    │
│ Istio        │ 300m     │ 800Mi    │
└──────────────┴──────────┴──────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Migration Strategies**

### **From NetworkPolicy to Service Mesh**

```yaml
# Phase 1: Keep NetworkPolicy, add service mesh
# NetworkPolicy (L3/L4 filtering)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-netpol
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend

# Service Mesh (L7 policies)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: myapp-authz
spec:
  selector:
    matchLabels:
      app: myapp
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/default/sa/frontend
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/*"]

# Phase 2: Gradually migrate to service mesh policies
# Phase 3: Remove NetworkPolicies if service mesh provides sufficient security
```

### **Progressive Rollout**

```bash
# Step 1: Install service mesh control plane
istioctl install --set profile=default

# Step 2: Enable injection for one namespace
kubectl label namespace myapp-staging istio-injection=enabled

# Step 3: Restart pods in staging
kubectl rollout restart deployment -n myapp-staging

# Step 4: Validate staging
istioctl analyze -n myapp-staging

# Step 5: Production rollout
kubectl label namespace myapp-prod istio-injection=enabled
kubectl rollout restart deployment -n myapp-prod
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **✅ Best Practices**

### **Security**

1. **Always enable mTLS**: Encrypt all service-to-service traffic
2. **Use AuthorizationPolicies**: Implement zero-trust with deny-all defaults
3. **Regular certificate rotation**: Automate cert lifecycle
4. **Audit policies**: Monitor and log authorization decisions
5. **Combine with NetworkPolicy**: Defense in depth

### **Performance**

1. **Resource limits**: Set appropriate proxy resource limits
2. **Connection pooling**: Configure connection pools
3. **Circuit breaking**: Implement circuit breakers
4. **Caching**: Use response caching where appropriate
5. **Monitor overhead**: Track proxy CPU/memory usage

### **Observability**

1. **Distributed tracing**: Enable sampling for critical services
2. **Service graphs**: Use service mesh dashboards
3. **Golden signals**: Monitor latency, traffic, errors, saturation
4. **Custom metrics**: Export app-specific metrics
5. **Alerting**: Set up alerts for service mesh health

### **Operations**

1. **Gradual rollouts**: Use canary deployments for service mesh
2. **Testing**: Thoroughly test in staging
3. **Documentation**: Document service dependencies
4. **Backup configs**: Version control all configurations
5. **Disaster recovery**: Plan for control plane failures

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Summary**

### **Decision Matrix**

| **Requirement** | **Istio** | **Linkerd** | **Cilium** |
|----------------|-----------|-------------|------------|
| **Simplicity** | Medium | High | Medium |
| **Performance** | Medium | High | Very High |
| **Features** | Extensive | Focused | Extensive |
| **L7 Policy** | Full | Full | Full |
| **Resource Usage** | Higher | Lower | Lowest |
| **Learning Curve** | Steep | Gentle | Medium |
| **Ecosystem** | Large | Medium | Large |
| **Production Ready** | Yes | Yes | Yes |
| **Multi-cluster** | Excellent | Good | Excellent |
| **Observability** | Excellent | Excellent | Excellent |

### **Recommendations**

- **Istio**: Choose for enterprise features, extensive ecosystem, multi-cloud
- **Linkerd**: Choose for simplicity, low overhead, fast deployment
- **Cilium**: Choose for best performance, eBPF benefits, no sidecars

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
