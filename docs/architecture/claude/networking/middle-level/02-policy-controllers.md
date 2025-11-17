# **Network Policy Controllers**

**Deep Dive into CNI Plugin Implementations: Calico, Cilium, Weave Net**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Table of Contents**

1. [Overview](#overview)
2. [CNI Plugin Architecture](#cni-plugin-architecture)
3. [Calico Implementation](#calico-implementation)
4. [Cilium Implementation](#cilium-implementation)
5. [Weave Net Implementation](#weave-net-implementation)
6. [Comparison Matrix](#comparison-matrix)
7. [Policy Controller Internals](#policy-controller-internals)
8. [Performance Analysis](#performance-analysis)
9. [Troubleshooting](#troubleshooting)
10. [Migration Guide](#migration-guide)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Overview**

### **What is a Network Policy Controller?**

A Network Policy Controller is the CNI plugin component responsible for:
- Watching NetworkPolicy objects from the Kubernetes API
- Translating policies into dataplane rules (iptables, eBPF, etc.)
- Enforcing network segmentation at the pod level
- Managing network connectivity and routing

### **Architecture Overview**

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes API Server                        │
│                    (NetworkPolicy Objects)                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ Watch/List
                             │
         ┌───────────────────┴────────────────────┐
         │                                         │
         ▼                                         ▼
┌─────────────────────┐                  ┌─────────────────────┐
│  Policy Controller  │                  │  Policy Controller  │
│     (Node A)        │                  │     (Node B)        │
└──────────┬──────────┘                  └──────────┬──────────┘
           │                                         │
           │ Program                                 │ Program
           ▼                                         ▼
   ┌───────────────┐                        ┌───────────────┐
   │   Dataplane   │                        │   Dataplane   │
   │   (iptables/  │                        │   (iptables/  │
   │    eBPF/OVS)  │                        │    eBPF/OVS)  │
   └───────┬───────┘                        └───────┬───────┘
           │                                         │
    ┌──────┴──────┐                          ┌──────┴──────┐
    │    Pods     │                          │    Pods     │
    └─────────────┘                          └─────────────┘
```

### **Key Components**

```go
// Generic CNI controller interface
type NetworkPolicyController interface {
    // Watch for NetworkPolicy changes
    Run(stopCh <-chan struct{}) error

    // Handle policy creation/update
    SyncPolicy(policy *networkingv1.NetworkPolicy) error

    // Handle policy deletion
    DeletePolicy(namespace, name string) error

    // Reconcile all policies
    ReconcileAll() error
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔌 CNI Plugin Architecture**

### **CNI Specification**

```go
// CNI Plugin Interface
// Reference: https://github.com/containernetworking/cni/blob/main/pkg/types/types.go

type CNI interface {
    // AddNetworkList adds a network list to a container
    AddNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)

    // DelNetworkList deletes a network list from a container
    DelNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error

    // CheckNetworkList checks whether a container's networking conforms
    CheckNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error

    // GetNetworkListCachedResult returns the result of the last successful
    // AddNetworkList for a given NetworkConfigList and RuntimeConf
    GetNetworkListCachedResult(net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
}

// CNI Configuration
type NetworkConfig struct {
    CNIVersion string `json:"cniVersion,omitempty"`
    Name       string `json:"name,omitempty"`
    Type       string `json:"type,omitempty"`

    // NetworkPolicy support
    SupportNetworkPolicy bool `json:"supportNetworkPolicy,omitempty"`

    // Plugin-specific config
    IPAM       *IPAMConfig       `json:"ipam,omitempty"`
    DNS        *DNSConfig        `json:"dns,omitempty"`
}
```

### **Kubernetes Integration**

```go
// CRI-based pod creation flow
// Location: pkg/kubelet/dockershim/network/plugins.go

func (pm *PluginManager) SetUpPod(namespace, name string, id kubecontainer.ContainerID,
    annotations, options map[string]string) error {

    start := time.Now()
    defer func() {
        metrics.NetworkPodOperationsLatency.Observe(
            metrics.SinceInSeconds(start))
    }()

    // Get active plugin
    plugin, err := pm.getActivePlugin()
    if err != nil {
        return err
    }

    // Setup network for pod
    if err := plugin.SetUpPod(namespace, name, id, annotations, options); err != nil {
        return err
    }

    klog.V(3).Infof("SetUp pod %s/%s with id %s", namespace, name, id)
    return nil
}
```

### **Policy Watcher Implementation**

```go
// Generic policy watcher pattern
type PolicyWatcher struct {
    client       kubernetes.Interface
    informer     cache.SharedIndexInformer
    store        cache.Store
    controller   cache.Controller

    // Policy handler
    handler      PolicyEventHandler

    // Resync period
    resyncPeriod time.Duration
}

func NewPolicyWatcher(client kubernetes.Interface, handler PolicyEventHandler) *PolicyWatcher {
    lw := &cache.ListWatch{
        ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
            return client.NetworkingV1().NetworkPolicies(metav1.NamespaceAll).List(context.TODO(), options)
        },
        WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
            return client.NetworkingV1().NetworkPolicies(metav1.NamespaceAll).Watch(context.TODO(), options)
        },
    }

    informer := cache.NewSharedIndexInformer(
        lw,
        &networkingv1.NetworkPolicy{},
        30*time.Second,
        cache.Indexers{},
    )

    informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            policy := obj.(*networkingv1.NetworkPolicy)
            handler.OnPolicyAdd(policy)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            oldPolicy := oldObj.(*networkingv1.NetworkPolicy)
            newPolicy := newObj.(*networkingv1.NetworkPolicy)
            handler.OnPolicyUpdate(oldPolicy, newPolicy)
        },
        DeleteFunc: func(obj interface{}) {
            policy := obj.(*networkingv1.NetworkPolicy)
            handler.OnPolicyDelete(policy)
        },
    })

    return &PolicyWatcher{
        client:       client,
        informer:     informer,
        handler:      handler,
        resyncPeriod: 30 * time.Second,
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🐱 Calico Implementation**

### **Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes API Server                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Watch NetworkPolicy
                         ▼
            ┌────────────────────────────┐
            │  calico-kube-controllers   │
            │  - Policy controller       │
            │  - Namespace controller    │
            │  - Node controller         │
            └────────────┬───────────────┘
                         │
                         │ Write Calico resources
                         ▼
            ┌────────────────────────────┐
            │       etcd/k8s API         │
            │   (Calico datastore)       │
            └────────────┬───────────────┘
                         │
                         │ Watch Calico resources
                         ▼
      ┌──────────────────────────────────────────┐
      │            calico-node (Felix)            │
      │  - Policy agent                           │
      │  - Route programming                      │
      │  - iptables/eBPF programming             │
      └──────────────────┬───────────────────────┘
                         │
                         │ Configure
                         ▼
              ┌──────────────────┐
              │   Linux Kernel   │
              │  - iptables      │
              │  - routing       │
              │  - eBPF (optional)│
              └──────────────────┘
```

### **Calico Components**

#### **calico-kube-controllers**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calico-kube-controllers
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: calico-kube-controllers
  template:
    metadata:
      labels:
        k8s-app: calico-kube-controllers
    spec:
      serviceAccountName: calico-kube-controllers
      containers:
      - name: calico-kube-controllers
        image: calico/kube-controllers:v3.25.0
        env:
        # Datastore type
        - name: DATASTORE_TYPE
          value: "kubernetes"

        # Controllers to run
        - name: ENABLED_CONTROLLERS
          value: "policy,namespace,serviceaccount,workloadendpoint,node"

        # Reconciliation settings
        - name: RECONCILER_PERIOD
          value: "5m"

        # Log level
        - name: LOG_LEVEL
          value: "info"

        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

**Controller Code Reference:**
```go
// Policy controller implementation
// Location: github.com/projectcalico/calico/kube-controllers/pkg/controllers/policy

type PolicyController struct {
    client       kubernetes.Interface
    calicoClient client.Interface
    informer     cache.SharedIndexInformer
    queue        workqueue.RateLimitingInterface
}

func (c *PolicyController) Run(stopCh <-chan struct{}) {
    defer c.queue.ShutDown()

    go c.informer.Run(stopCh)

    if !cache.WaitForCacheSync(stopCh, c.informer.HasSynced) {
        klog.Error("Failed to sync cache")
        return
    }

    // Start workers
    for i := 0; i < 5; i++ {
        go wait.Until(c.runWorker, time.Second, stopCh)
    }

    <-stopCh
}

func (c *PolicyController) syncPolicy(key string) error {
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }

    // Get Kubernetes NetworkPolicy
    k8sPolicy, err := c.client.NetworkingV1().NetworkPolicies(namespace).Get(
        context.Background(), name, metav1.GetOptions{})
    if err != nil {
        if errors.IsNotFound(err) {
            // Delete Calico policy
            return c.deleteCalicoPolicy(namespace, name)
        }
        return err
    }

    // Convert to Calico policy
    calicoPolicy := c.convertToCalico(k8sPolicy)

    // Update Calico datastore
    _, err = c.calicoClient.NetworkPolicies().Update(
        context.Background(), calicoPolicy, options.SetOptions{})

    return err
}

func (c *PolicyController) convertToCalico(k8sPolicy *networkingv1.NetworkPolicy) *api.NetworkPolicy {
    calicoPolicy := &api.NetworkPolicy{
        ObjectMeta: metav1.ObjectMeta{
            Name:      k8sPolicy.Name,
            Namespace: k8sPolicy.Namespace,
        },
        Spec: api.NetworkPolicySpec{
            Order:    &defaultOrder,
            Selector: convertSelector(k8sPolicy.Spec.PodSelector),
            Ingress:  convertIngressRules(k8sPolicy.Spec.Ingress),
            Egress:   convertEgressRules(k8sPolicy.Spec.Egress),
            Types:    convertPolicyTypes(k8sPolicy.Spec.PolicyTypes),
        },
    }

    return calicoPolicy
}
```

#### **calico-node (Felix)**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true
      serviceAccountName: calico-node
      containers:
      - name: calico-node
        image: calico/node:v3.25.0
        env:
        # Datastore
        - name: DATASTORE_TYPE
          value: "kubernetes"

        # BGP configuration
        - name: CALICO_NETWORKING_BACKEND
          value: "bird"

        # Felix configuration
        - name: FELIX_IPTABLESREFRESHINTERVAL
          value: "60"
        - name: FELIX_IPTABLESLOCKTIMEOUTSECS
          value: "10"

        # eBPF mode (optional)
        - name: FELIX_BPFENABLED
          value: "false"

        # IP-in-IP
        - name: FELIX_IPINIPENABLED
          value: "true"

        # MTU
        - name: FELIX_IPINIPMTU
          value: "1440"

        # Logging
        - name: FELIX_LOGSEVERITYSCREEN
          value: "info"

        securityContext:
          privileged: true

        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi

        volumeMounts:
        - mountPath: /lib/modules
          name: lib-modules
          readOnly: true
        - mountPath: /var/run/calico
          name: var-run-calico
        - mountPath: /var/lib/calico
          name: var-lib-calico

      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: var-run-calico
        hostPath:
          path: /var/run/calico
      - name: var-lib-calico
        hostPath:
          path: /var/lib/calico
```

**Felix Policy Engine:**
```go
// Felix policy processing
// Location: github.com/projectcalico/calico/felix/calc

type PolicyResolver struct {
    allPolicies         map[model.PolicyKey]*model.Policy
    sortedPolicyKeys    []model.PolicyKey
    dirtyPolicies       set.Set

    // Endpoint tracking
    localEndpoints      map[proto.WorkloadEndpointID]*Endpoint

    // Active rules per endpoint
    endpointRules       map[proto.WorkloadEndpointID]*RuleSet
}

func (pr *PolicyResolver) ProcessPolicyUpdate(policy *model.Policy) {
    key := model.PolicyKey{Name: policy.Name}

    pr.allPolicies[key] = policy
    pr.dirtyPolicies.Add(key)

    // Resort policies by order
    pr.sortPolicies()

    // Recalculate affected endpoints
    pr.recalculateEndpoints()
}

func (pr *PolicyResolver) recalculateEndpoints() {
    for epID, endpoint := range pr.localEndpoints {
        rules := pr.calculateRulesForEndpoint(endpoint)
        pr.endpointRules[epID] = rules

        // Send to dataplane
        pr.sendToDataplane(epID, rules)
    }
}

func (pr *PolicyResolver) calculateRulesForEndpoint(ep *Endpoint) *RuleSet {
    rules := &RuleSet{
        InboundRules:  []Rule{},
        OutboundRules: []Rule{},
    }

    // Apply policies in order
    for _, policyKey := range pr.sortedPolicyKeys {
        policy := pr.allPolicies[policyKey]

        // Check if policy applies to endpoint
        if !pr.policyMatches(policy, ep) {
            continue
        }

        // Add ingress rules
        for _, ingressRule := range policy.InboundRules {
            rules.InboundRules = append(rules.InboundRules,
                pr.processRule(ingressRule, ep))
        }

        // Add egress rules
        for _, egressRule := range policy.OutboundRules {
            rules.OutboundRules = append(rules.OutboundRules,
                pr.processRule(egressRule, ep))
        }
    }

    return rules
}
```

### **Calico Policy Translation**

**Kubernetes NetworkPolicy:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
```

**Equivalent Calico NetworkPolicy:**
```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: myapp
spec:
  order: 1000
  selector: role == 'backend'
  types:
  - Ingress
  - Egress
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: role == 'frontend'
    destination:
      ports:
      - 8080
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: role == 'database'
      ports:
      - 5432
```

### **iptables Rules Generated**

```bash
# Calico chain structure
*filter
:cali-INPUT - [0:0]
:cali-OUTPUT - [0:0]
:cali-FORWARD - [0:0]

# Main chains
:cali-from-endpoint - [0:0]
:cali-to-endpoint - [0:0]

# Workload chains (per pod)
:cali-fw-cali1234567890 - [0:0]  # From workload
:cali-tw-cali1234567890 - [0:0]  # To workload

# Policy chains
:cali-pi-backend-policy - [0:0]  # Policy ingress
:cali-po-backend-policy - [0:0]  # Policy egress

# From workload chain
-A cali-fw-cali1234567890 -m comment --comment "Start of policies" -j MARK --set-xmark 0x0/0x20000
-A cali-fw-cali1234567890 -m mark --mark 0x0/0x20000 -j cali-po-backend-policy
-A cali-fw-cali1234567890 -m comment --comment "Return if policy accepted" -m mark --mark 0x10000/0x10000 -j RETURN
-A cali-fw-cali1234567890 -m comment --comment "Drop if no policy matched" -j DROP

# To workload chain
-A cali-tw-cali1234567890 -m comment --comment "Start of policies" -j MARK --set-xmark 0x0/0x20000
-A cali-tw-cali1234567890 -m mark --mark 0x0/0x20000 -j cali-pi-backend-policy
-A cali-tw-cali1234567890 -m comment --comment "Return if policy accepted" -m mark --mark 0x10000/0x10000 -j RETURN
-A cali-tw-cali1234567890 -m comment --comment "Drop if no policy matched" -j DROP

# Egress policy (to database)
-A cali-po-backend-policy -m comment --comment "Allow to database" -m set --match-set cali40s:role-database dst -p tcp --dport 5432 -j MARK --set-xmark 0x10000/0x10000
-A cali-po-backend-policy -m mark --mark 0x10000/0x10000 -j RETURN

# Ingress policy (from frontend)
-A cali-pi-backend-policy -m comment --comment "Allow from frontend" -m set --match-set cali40s:role-frontend src -p tcp --dport 8080 -j MARK --set-xmark 0x10000/0x10000
-A cali-pi-backend-policy -m mark --mark 0x10000/0x10000 -j RETURN

COMMIT
```

### **Calico Features**

#### **Global Network Policies**

```yaml
# Cluster-wide policy (not namespace-scoped)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny-all
spec:
  order: 2000
  selector: all()
  types:
  - Ingress
  - Egress
  # No rules = default deny
```

#### **Network Sets**

```yaml
# IP set for external resources
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: external-apis
  labels:
    type: external
spec:
  nets:
  - 198.51.100.0/24
  - 203.0.113.0/24

---
# Policy using network set
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-external-apis
  namespace: myapp
spec:
  selector: app == 'backend'
  egress:
  - action: Allow
    destination:
      selector: type == 'external'
    protocol: TCP
    ports:
    - 443
```

#### **Service Account Matching**

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: restrict-by-service-account
  namespace: myapp
spec:
  selector: all()
  ingress:
  - action: Allow
    source:
      serviceAccounts:
        names:
        - trusted-app
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🐝 Cilium Implementation**

### **Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                  Kubernetes API Server                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Watch K8s resources
                         ▼
            ┌────────────────────────────┐
            │    cilium-operator         │
            │  - CRD management          │
            │  - IP address management   │
            │  - Node management         │
            └────────────┬───────────────┘
                         │
                         │
                         ▼
      ┌──────────────────────────────────────────┐
      │           cilium-agent                    │
      │  - Policy engine                          │
      │  - Identity management                    │
      │  - eBPF compilation & loading            │
      │  - Endpoint management                    │
      └──────────────────┬───────────────────────┘
                         │
                         │ Load eBPF programs
                         ▼
              ┌──────────────────┐
              │   Linux Kernel   │
              │  - eBPF runtime  │
              │  - XDP           │
              │  - tc (traffic   │
              │    control)      │
              └──────────────────┘
```

### **Cilium Components**

#### **cilium-operator**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cilium-operator
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      name: cilium-operator
  template:
    metadata:
      labels:
        name: cilium-operator
    spec:
      serviceAccountName: cilium-operator
      containers:
      - name: cilium-operator
        image: cilium/operator:v1.13.0
        command:
        - cilium-operator
        args:
        # Identity allocation mode
        - --identity-allocation-mode=crd

        # Enable CNP status updates
        - --cnp-status-update-interval=10s

        # Identity GC
        - --identity-gc-interval=15m

        # Endpoint GC
        - --cilium-endpoint-gc-interval=5m

        env:
        - name: K8S_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName

        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

#### **cilium-agent**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cilium
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: cilium
  template:
    metadata:
      labels:
        k8s-app: cilium
    spec:
      hostNetwork: true
      serviceAccountName: cilium
      initContainers:
      - name: mount-cgroup
        image: cilium/cilium:v1.13.0
        command:
        - sh
        - -c
        - |
          # Mount cgroup v2
          mount | grep /run/cilium/cgroupv2 || {
            mkdir -p /run/cilium/cgroupv2
            mount -t cgroup2 none /run/cilium/cgroupv2
          }
        volumeMounts:
        - name: cilium-run
          mountPath: /run/cilium
        securityContext:
          privileged: true

      containers:
      - name: cilium-agent
        image: cilium/cilium:v1.13.0
        command:
        - cilium-agent
        args:
        # Datapath mode
        - --datapath-mode=veth

        # Enable BPF masquerade
        - --enable-bpf-masquerade=true

        # Enable host routing
        - --enable-host-routing=true

        # IPv4/IPv6
        - --enable-ipv4=true
        - --enable-ipv6=false

        # Identity allocation
        - --identity-allocation-mode=crd

        # Enable Hubble (observability)
        - --enable-hubble=true
        - --hubble-listen-address=:4244
        - --hubble-metrics-server=:9091
        - --hubble-metrics=dns,drop,tcp,flow,icmp,http

        # Policy enforcement
        - --enable-policy=default
        - --policy-audit-mode=false

        # Install CNI
        - --install-cni-binaries=true

        # kube-proxy replacement
        - --kube-proxy-replacement=strict

        # Encryption (optional)
        # - --enable-ipsec=true
        # - --encrypt-interface=eth0

        env:
        - name: K8S_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: CILIUM_K8S_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace

        securityContext:
          privileged: true
          capabilities:
            add:
            - NET_ADMIN
            - SYS_MODULE
            - SYS_ADMIN
            - SYS_RESOURCE

        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi

        volumeMounts:
        - name: bpf-maps
          mountPath: /sys/fs/bpf
        - name: cilium-run
          mountPath: /run/cilium
        - name: cni-path
          mountPath: /host/opt/cni/bin
        - name: etc-cni-netd
          mountPath: /host/etc/cni/net.d
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true

      volumes:
      - name: bpf-maps
        hostPath:
          path: /sys/fs/bpf
          type: DirectoryOrCreate
      - name: cilium-run
        hostPath:
          path: /run/cilium
          type: DirectoryOrCreate
      - name: cni-path
        hostPath:
          path: /opt/cni/bin
          type: DirectoryOrCreate
      - name: etc-cni-netd
        hostPath:
          path: /etc/cni/net.d
          type: DirectoryOrCreate
      - name: lib-modules
        hostPath:
          path: /lib/modules
```

### **Identity-Based Security**

```go
// Cilium identity system
// Location: github.com/cilium/cilium/pkg/identity

type Identity struct {
    // Unique identity number
    ID NumericIdentity

    // Labels associated with identity
    Labels labels.Labels

    // Reference count
    refCount uint32
}

type IdentityAllocator struct {
    // Global identity cache
    identities map[NumericIdentity]*Identity

    // Label -> Identity mapping
    labelIndex map[string]*Identity

    // Identity allocation backend (CRD/KVStore)
    backend IdentityBackend
}

func (ia *IdentityAllocator) AllocateIdentity(lbls labels.Labels) (*Identity, error) {
    // Check cache first
    if id := ia.lookupIdentity(lbls); id != nil {
        id.refCount++
        return id, nil
    }

    // Allocate new identity
    numericID, err := ia.backend.AllocateID()
    if err != nil {
        return nil, err
    }

    identity := &Identity{
        ID:       numericID,
        Labels:   lbls,
        refCount: 1,
    }

    ia.identities[numericID] = identity
    ia.labelIndex[lbls.String()] = identity

    return identity, nil
}
```

**Identity Assignment:**
```
┌──────────────────────────────────────────────────────┐
│  Pod: frontend (ns: myapp)                           │
│  Labels:                                             │
│    app=frontend                                      │
│    version=v1                                        │
│    k8s:io.kubernetes.pod.namespace=myapp            │
│                                                      │
│  Assigned Identity: 12345                           │
└──────────────────────────────────────────────────────┘
                         │
                         │ eBPF maps contain
                         │ identity -> policy rules
                         ▼
┌──────────────────────────────────────────────────────┐
│  eBPF Policy Map                                     │
│                                                      │
│  12345 -> {                                         │
│    ingress: [allow from 12346],                    │
│    egress: [allow to 12347, allow to CIDR x.x.x.x]│
│  }                                                   │
└──────────────────────────────────────────────────────┘
```

### **eBPF Policy Engine**

```c
// Simplified eBPF policy program
// Location: github.com/cilium/cilium/bpf/lib/policy.h

struct policy_entry {
    __u32 identity;      // Source/dest identity
    __u32 port;          // Destination port
    __u8  protocol;      // L4 protocol
    __u8  action;        // Allow/deny
    __u16 padding;
};

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(key_size, sizeof(__u32));
    __uint(value_size, sizeof(struct policy_entry));
    __uint(max_entries, 65536);
} cilium_policy_map SEC(".maps");

static __always_inline int
policy_can_egress(__u32 src_identity, __u32 dst_identity,
                  __u16 dport, __u8 protocol)
{
    struct policy_entry *entry;
    __u32 key = (src_identity << 16) | (dst_identity & 0xFFFF);

    entry = map_lookup_elem(&cilium_policy_map, &key);
    if (!entry)
        return DROP_POLICY;

    if (entry->port != 0 && entry->port != dport)
        return DROP_POLICY;

    if (entry->protocol != 0 && entry->protocol != protocol)
        return DROP_POLICY;

    if (entry->action == POLICY_ALLOW)
        return TC_ACT_OK;

    return DROP_POLICY;
}
```

### **CiliumNetworkPolicy CRD**

```yaml
# Extended network policy with L7 rules
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-l7-policy
  namespace: myapp
spec:
  endpointSelector:
    matchLabels:
      role: backend

  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/.*"
        - method: "POST"
          path: "/api/data"
          headers:
          - "Content-Type: application/json"

  egress:
  - toEndpoints:
    - matchLabels:
        role: database
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP

  - toFQDNs:
    - matchName: "api.example.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/v1/.*"
```

### **DNS-Based Policies**

```yaml
# Allow egress to specific DNS names
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-external-apis
  namespace: myapp
spec:
  endpointSelector:
    matchLabels:
      app: backend

  egress:
  # Allow to specific FQDNs
  - toFQDNs:
    - matchName: "api.github.com"
    - matchPattern: "*.googleapis.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP

  # DNS proxy intercepts and learns IPs
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: kube-system
        k8s-app: kube-dns
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
      rules:
        dns:
        - matchName: "api.github.com"
        - matchPattern: "*.googleapis.com"
```

**DNS Proxy Mechanism:**
```go
// DNS proxy implementation
// Location: github.com/cilium/cilium/pkg/proxy/dns

type DNSProxy struct {
    // DNS cache: FQDN -> IPs
    cache map[string][]net.IP

    // Policy rules
    allowedFQDNs map[string]bool

    // IP -> FQDN reverse mapping
    ipToFQDN map[string]string
}

func (p *DNSProxy) handleDNSRequest(req *dns.Msg) *dns.Msg {
    for _, question := range req.Question {
        fqdn := question.Name

        // Check if FQDN is allowed by policy
        if !p.isAllowed(fqdn) {
            return dnsRefused(req)
        }

        // Forward to upstream DNS
        resp := p.forwardDNS(req)

        // Cache IPs and inject policy rules
        for _, answer := range resp.Answer {
            if a, ok := answer.(*dns.A); ok {
                ip := a.A

                // Update cache
                p.cache[fqdn] = append(p.cache[fqdn], ip)
                p.ipToFQDN[ip.String()] = fqdn

                // Inject eBPF rules allowing traffic to this IP
                p.injectPolicyRule(fqdn, ip)
            }
        }

        return resp
    }

    return nil
}
```

### **Hubble Observability**

```yaml
# Enable Hubble UI
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hubble-ui
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: hubble-ui
  template:
    metadata:
      labels:
        k8s-app: hubble-ui
    spec:
      serviceAccountName: hubble-ui
      containers:
      - name: frontend
        image: quay.io/cilium/hubble-ui:v0.11.0
        ports:
        - containerPort: 8081
        env:
        - name: EVENTS_SERVER_PORT
          value: "8090"
      - name: backend
        image: quay.io/cilium/hubble-ui-backend:v0.11.0
        ports:
        - containerPort: 8090
        env:
        - name: EVENTS_SERVER_PORT
          value: "8090"
        - name: FLOWS_API_ADDR
          value: "hubble-relay:80"
```

**Flow Monitoring:**
```bash
# Watch flows in real-time
hubble observe --follow

# Filter by namespace
hubble observe --namespace myapp

# Filter by pod
hubble observe --pod frontend-12345

# Filter by verdict (dropped packets)
hubble observe --verdict DROPPED

# HTTP flows
hubble observe --protocol http

# DNS flows
hubble observe --protocol dns

# Service map
hubble observe --output json | jq '.flow | {source: .source, destination: .destination}'
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🕸️ Weave Net Implementation**

### **Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                 Kubernetes API Server                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Watch NetworkPolicy
                         ▼
      ┌──────────────────────────────────────────┐
      │            weave-net                      │
      │  - CNI plugin                             │
      │  - Network Policy Controller (weave-npc) │
      │  - Weave router                           │
      └──────────────────┬───────────────────────┘
                         │
                         │ Configure iptables
                         ▼
              ┌──────────────────┐
              │   Linux Kernel   │
              │  - iptables      │
              │  - bridge        │
              │  - veth pairs    │
              └──────────────────┘
```

### **Weave Components**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: weave-net
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: weave-net
  template:
    metadata:
      labels:
        name: weave-net
    spec:
      hostNetwork: true
      serviceAccountName: weave-net
      tolerations:
      - effect: NoSchedule
        operator: Exists

      initContainers:
      - name: weave-init
        image: weaveworks/weave-kube:2.8.1
        command:
        - /home/weave/init.sh
        volumeMounts:
        - name: cni-bin
          mountPath: /host/opt
        - name: cni-bin2
          mountPath: /host/home
        - name: cni-conf
          mountPath: /host/etc
        - name: lib-modules
          mountPath: /lib/modules
        securityContext:
          privileged: true

      containers:
      # Weave router
      - name: weave
        image: weaveworks/weave-kube:2.8.1
        command:
        - /home/weave/launch.sh
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: IPALLOC_RANGE
          value: "10.32.0.0/12"
        - name: WEAVE_MTU
          value: "1376"
        securityContext:
          privileged: true
        volumeMounts:
        - name: weavedb
          mountPath: /weavedb
        - name: dbus
          mountPath: /host/var/lib/dbus
        - name: machine-id
          mountPath: /host/etc/machine-id
          readOnly: true
        resources:
          requests:
            cpu: 50m
            memory: 128Mi

      # Network Policy Controller
      - name: weave-npc
        image: weaveworks/weave-npc:2.8.1
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
        securityContext:
          privileged: true

      volumes:
      - name: weavedb
        hostPath:
          path: /var/lib/weave
      - name: cni-bin
        hostPath:
          path: /opt
      - name: cni-bin2
        hostPath:
          path: /home
      - name: cni-conf
        hostPath:
          path: /etc
      - name: dbus
        hostPath:
          path: /var/lib/dbus
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: machine-id
        hostPath:
          path: /etc/machine-id
```

### **Weave NPC (Network Policy Controller)**

```go
// Weave NPC implementation
// Location: github.com/weaveworks/weave/npc

type NetworkPolicyController struct {
    client kubernetes.Interface

    // Informers
    podInformer      cache.SharedIndexInformer
    nsInformer       cache.SharedIndexInformer
    policyInformer   cache.SharedIndexInformer

    // iptables interface
    ipt              *iptables.IPTables

    // Policy state
    policies         map[string]*networkingv1.NetworkPolicy
    pods             map[string]*v1.Pod
    namespaces       map[string]*v1.Namespace
}

func (npc *NetworkPolicyController) Run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()

    // Start informers
    go npc.podInformer.Run(stopCh)
    go npc.nsInformer.Run(stopCh)
    go npc.policyInformer.Run(stopCh)

    // Wait for cache sync
    if !cache.WaitForCacheSync(stopCh,
        npc.podInformer.HasSynced,
        npc.nsInformer.HasSynced,
        npc.policyInformer.HasSynced) {
        return
    }

    // Initial sync
    npc.syncAll()

    // Periodic resync
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            npc.syncAll()
        case <-stopCh:
            return
        }
    }
}

func (npc *NetworkPolicyController) syncAll() error {
    // Get all policies
    policies, err := npc.client.NetworkingV1().NetworkPolicies("").List(
        context.Background(), metav1.ListOptions{})
    if err != nil {
        return err
    }

    // Rebuild iptables rules
    return npc.rebuildIPTables(policies.Items)
}

func (npc *NetworkPolicyController) rebuildIPTables(policies []networkingv1.NetworkPolicy) error {
    // Create new chain set
    chains := npc.buildChains(policies)

    // Apply to iptables
    return npc.applyIPTables(chains)
}
```

### **iptables Structure**

```bash
# Weave NPC iptables chains
*filter

# Main Weave chains
:WEAVE-NPC - [0:0]
:WEAVE-NPC-DEFAULT - [0:0]
:WEAVE-NPC-INGRESS - [0:0]
:WEAVE-NPC-EGRESS - [0:0]

# Hook into FORWARD chain
-A FORWARD -m comment --comment "Weave NPC" -j WEAVE-NPC

# Main NPC chain
-A WEAVE-NPC -m state --state RELATED,ESTABLISHED -j ACCEPT
-A WEAVE-NPC -m physdev --physdev-is-bridged --physdev-out vethwe+ -j WEAVE-NPC-EGRESS
-A WEAVE-NPC -m physdev --physdev-is-bridged --physdev-in vethwe+ -j WEAVE-NPC-INGRESS
-A WEAVE-NPC -m addrtype --dst-type LOCAL -j WEAVE-NPC-INGRESS
-A WEAVE-NPC -m addrtype --src-type LOCAL -j WEAVE-NPC-EGRESS

# Per-pod chains (created dynamically)
:WEAVE-NPC-EGRESS-10-32-0-5 - [0:0]    # Pod 10.32.0.5 egress
:WEAVE-NPC-INGRESS-10-32-0-5 - [0:0]   # Pod 10.32.0.5 ingress

# Egress rules
-A WEAVE-NPC-EGRESS -s 10.32.0.5/32 -m comment --comment "pod:myapp/frontend-abc123" -j WEAVE-NPC-EGRESS-10-32-0-5

# Default: if pod has any NetworkPolicy, default deny
-A WEAVE-NPC-EGRESS-10-32-0-5 -m state --state NEW -j WEAVE-NPC-DEFAULT
-A WEAVE-NPC-DEFAULT -m comment --comment "Default deny" -j DROP

# Policy rules (example: allow to backend)
-A WEAVE-NPC-EGRESS-10-32-0-5 -d 10.32.0.10/32 -p tcp --dport 8080 -m comment --comment "Allow to backend" -j ACCEPT

# Ingress rules
-A WEAVE-NPC-INGRESS -d 10.32.0.5/32 -m comment --comment "pod:myapp/frontend-abc123" -j WEAVE-NPC-INGRESS-10-32-0-5
-A WEAVE-NPC-INGRESS-10-32-0-5 -m state --state NEW -j WEAVE-NPC-DEFAULT
-A WEAVE-NPC-INGRESS-10-32-0-5 -s 10.32.0.1/32 -p tcp --dport 80 -m comment --comment "Allow from gateway" -j ACCEPT

COMMIT
```

### **Weave Features**

#### **DefaultAllow Annotation**

```yaml
# Namespace with default allow
apiVersion: v1
kind: Namespace
metadata:
  name: trusted-ns
  annotations:
    # Pods in this namespace are not isolated by default
    net.beta.kubernetes.io/network-policy: |
      {
        "ingress": {
          "isolation": "DefaultAllow"
        }
      }
```

#### **Multicast Support**

Weave Net supports multicast traffic, which is useful for some applications:

```yaml
# Policy allowing multicast
# Note: This is Weave-specific, not standard Kubernetes
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-multicast
  namespace: myapp
  annotations:
    # Weave-specific annotation
    weave.works/allow-multicast: "true"
spec:
  podSelector:
    matchLabels:
      app: multicast-app
  policyTypes:
  - Ingress
  - Egress
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Comparison Matrix**

### **Feature Comparison**

| **Feature** | **Calico** | **Cilium** | **Weave Net** |
|------------|-----------|-----------|--------------|
| **Dataplane** | iptables, eBPF | eBPF, iptables (legacy) | iptables |
| **Performance** | High | Very High (eBPF) | Medium |
| **NetworkPolicy Support** | Full | Full + Extensions | Full |
| **L7 Policy** | Yes (Envoy) | Yes (native eBPF) | No |
| **DNS Policy** | Limited | Full (FQDN matching) | No |
| **Encryption** | WireGuard, IPsec | IPsec, WireGuard | IPsec |
| **Service Mesh Integration** | Istio compatible | Native (Cilium Service Mesh) | Limited |
| **Observability** | Limited (felixmetrics) | Excellent (Hubble) | Basic |
| **Global Policies** | Yes | Yes | No |
| **Multi-cluster** | Yes (Calico Enterprise) | Yes (Cluster Mesh) | Limited |
| **IPv6 Support** | Yes | Yes | Yes |
| **Scalability** | Excellent (50k+ pods) | Excellent (50k+ pods) | Good (10k pods) |
| **kube-proxy Replacement** | Yes (eBPF mode) | Yes (full) | No |
| **Egress Gateway** | Yes | Yes | No |
| **FQDN Filtering** | Basic | Advanced | No |
| **HTTP/gRPC Policies** | Via Envoy | Native | No |
| **Identity-based Security** | Limited | Full | No |
| **Resource Usage (CPU)** | Low-Medium | Low (eBPF), High (compilation) | Low |
| **Resource Usage (Memory)** | Medium | Medium-High | Low |
| **Learning Curve** | Medium | High | Low |
| **Community** | Large | Large | Medium |
| **Enterprise Support** | Yes (Tigera) | Yes (Isovalent) | Limited |

### **Performance Benchmarks**

```
┌─────────────────────────────────────────────────────────────┐
│  Latency (p99) - HTTP request                               │
├─────────────────────────────────────────────────────────────┤
│  No NetworkPolicy:           5ms                            │
│  Calico (iptables):         12ms  [▓▓▓▓░░░░░░]             │
│  Calico (eBPF):              7ms  [▓▓░░░░░░░░]             │
│  Cilium (eBPF):              6ms  [▓░░░░░░░░░]             │
│  Weave:                     15ms  [▓▓▓▓▓░░░░░]             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Throughput - TCP (Gbps)                                    │
├─────────────────────────────────────────────────────────────┤
│  Baseline (no CNI):         9.5 Gbps                        │
│  Calico (iptables):         8.2 Gbps  [▓▓▓▓▓▓▓▓░░]         │
│  Calico (eBPF):             9.1 Gbps  [▓▓▓▓▓▓▓▓▓░]         │
│  Cilium (eBPF):             9.3 Gbps  [▓▓▓▓▓▓▓▓▓▓]         │
│  Weave:                     7.5 Gbps  [▓▓▓▓▓▓▓░░░]         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  CPU Usage (per 1000 pods)                                  │
├─────────────────────────────────────────────────────────────┤
│  Calico (iptables):         1.2 cores                       │
│  Calico (eBPF):             0.8 cores                       │
│  Cilium (eBPF):             1.5 cores (incl. compilation)   │
│  Weave:                     0.9 cores                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Memory Usage (per 1000 pods)                               │
├─────────────────────────────────────────────────────────────┤
│  Calico:                    800 MB                          │
│  Cilium:                    1.2 GB                          │
│  Weave:                     600 MB                          │
└─────────────────────────────────────────────────────────────┘
```

### **Use Case Recommendations**

| **Use Case** | **Recommended CNI** | **Reason** |
|-------------|-------------------|-----------|
| **High Performance** | Cilium (eBPF) | Lowest latency, highest throughput |
| **L7 Security** | Cilium | Native L7 policy enforcement |
| **Simple Setup** | Weave Net | Easy installation, minimal config |
| **Enterprise Features** | Calico | Mature, extensive enterprise support |
| **Service Mesh** | Cilium | Built-in service mesh capabilities |
| **Large Scale** | Calico or Cilium | Both scale to 50k+ pods |
| **Multi-cloud** | Calico | Strong multi-cloud support |
| **Observability** | Cilium | Hubble provides excellent visibility |
| **Windows Support** | Calico | Best Windows container support |
| **Encryption** | Calico | WireGuard performance advantage |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 Policy Controller Internals**

### **Controller Pattern**

```go
// Generic controller pattern used by all CNIs
type PolicyController struct {
    // Kubernetes client
    kubeClient kubernetes.Interface

    // Shared informers
    policyInformer cache.SharedIndexInformer
    podInformer    cache.SharedIndexInformer
    nsInformer     cache.SharedIndexInformer

    // Work queue
    workqueue workqueue.RateLimitingInterface

    // Policy store
    policyStore PolicyStore

    // Dataplane programmer
    dataplane DataplaneProgrammer
}

func (c *PolicyController) Run(workers int, stopCh <-chan struct{}) error {
    defer utilruntime.HandleCrash()
    defer c.workqueue.ShutDown()

    klog.Info("Starting policy controller")

    // Start informers
    go c.policyInformer.Run(stopCh)
    go c.podInformer.Run(stopCh)
    go c.nsInformer.Run(stopCh)

    // Wait for cache sync
    klog.Info("Waiting for informer caches to sync")
    if ok := cache.WaitForCacheSync(stopCh,
        c.policyInformer.HasSynced,
        c.podInformer.HasSynced,
        c.nsInformer.HasSynced); !ok {
        return fmt.Errorf("failed to wait for caches to sync")
    }

    klog.Info("Starting workers")
    for i := 0; i < workers; i++ {
        go wait.Until(c.runWorker, time.Second, stopCh)
    }

    klog.Info("Started workers")
    <-stopCh
    klog.Info("Shutting down workers")

    return nil
}

func (c *PolicyController) runWorker() {
    for c.processNextWorkItem() {
    }
}

func (c *PolicyController) processNextWorkItem() bool {
    obj, shutdown := c.workqueue.Get()
    if shutdown {
        return false
    }

    err := func(obj interface{}) error {
        defer c.workqueue.Done(obj)

        var key string
        var ok bool
        if key, ok = obj.(string); !ok {
            c.workqueue.Forget(obj)
            return fmt.Errorf("expected string in workqueue but got %#v", obj)
        }

        if err := c.syncHandler(key); err != nil {
            c.workqueue.AddRateLimited(key)
            return fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
        }

        c.workqueue.Forget(obj)
        klog.Infof("Successfully synced '%s'", key)
        return nil
    }(obj)

    if err != nil {
        utilruntime.HandleError(err)
        return true
    }

    return true
}

func (c *PolicyController) syncHandler(key string) error {
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }

    // Get policy from cache
    policy, err := c.policyInformer.GetStore().Get(key)
    if err != nil {
        // Policy was deleted
        return c.handlePolicyDelete(namespace, name)
    }

    // Update dataplane
    return c.dataplane.UpdatePolicy(policy.(*networkingv1.NetworkPolicy))
}
```

### **Policy Reconciliation**

```go
// Policy reconciliation loop
type PolicyReconciler struct {
    // Current state (what's programmed)
    currentState map[string]*Policy

    // Desired state (from API server)
    desiredState map[string]*Policy

    // Diff calculator
    differ PolicyDiffer

    // Dataplane
    dataplane DataplaneProgrammer
}

func (r *PolicyReconciler) Reconcile() error {
    // Calculate diff
    diff := r.differ.Calculate(r.currentState, r.desiredState)

    // Apply adds
    for _, policy := range diff.ToAdd {
        if err := r.dataplane.AddPolicy(policy); err != nil {
            return err
        }
        r.currentState[policy.Key()] = policy
    }

    // Apply updates
    for _, policy := range diff.ToUpdate {
        if err := r.dataplane.UpdatePolicy(policy); err != nil {
            return err
        }
        r.currentState[policy.Key()] = policy
    }

    // Apply deletes
    for _, key := range diff.ToDelete {
        if err := r.dataplane.DeletePolicy(key); err != nil {
            return err
        }
        delete(r.currentState, key)
    }

    return nil
}
```

### **Event Processing**

```go
// Event handlers
func (c *PolicyController) handlePolicyAdd(obj interface{}) {
    policy := obj.(*networkingv1.NetworkPolicy)
    klog.V(4).Infof("Adding policy %s/%s", policy.Namespace, policy.Name)

    key, err := cache.MetaNamespaceKeyFunc(policy)
    if err != nil {
        utilruntime.HandleError(err)
        return
    }

    c.workqueue.Add(key)
}

func (c *PolicyController) handlePolicyUpdate(oldObj, newObj interface{}) {
    oldPolicy := oldObj.(*networkingv1.NetworkPolicy)
    newPolicy := newObj.(*networkingv1.NetworkPolicy)

    // Skip if resource version is the same
    if oldPolicy.ResourceVersion == newPolicy.ResourceVersion {
        return
    }

    klog.V(4).Infof("Updating policy %s/%s", newPolicy.Namespace, newPolicy.Name)

    key, err := cache.MetaNamespaceKeyFunc(newPolicy)
    if err != nil {
        utilruntime.HandleError(err)
        return
    }

    c.workqueue.Add(key)
}

func (c *PolicyController) handlePolicyDelete(obj interface{}) {
    policy, ok := obj.(*networkingv1.NetworkPolicy)
    if !ok {
        tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
        if !ok {
            utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %#v", obj))
            return
        }
        policy, ok = tombstone.Obj.(*networkingv1.NetworkPolicy)
        if !ok {
            utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a NetworkPolicy %#v", obj))
            return
        }
    }

    klog.V(4).Infof("Deleting policy %s/%s", policy.Namespace, policy.Name)

    key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
    if err != nil {
        utilruntime.HandleError(err)
        return
    }

    c.workqueue.Add(key)
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ Performance Analysis**

### **iptables vs eBPF**

**iptables Overhead:**
```
Packet Path with iptables NetworkPolicy:
┌─────────┐
│  NIC    │ (1) Hardware interrupt
└────┬────┘
     │
     ▼
┌─────────────┐
│   Kernel    │ (2) softirq processing
│   Network   │
│   Stack     │
└────┬────────┘
     │
     ▼
┌─────────────────────────────┐
│  iptables (PREROUTING)      │ (3) Rule traversal (O(n))
│  - Raw table                │
│  - Mangle table             │
│  - NAT table                │
└────┬────────────────────────┘
     │
     ▼
┌─────────────────────────────┐
│  Routing Decision           │ (4) Lookup routing table
└────┬────────────────────────┘
     │
     ▼
┌─────────────────────────────┐
│  iptables (FORWARD)         │ (5) More rule traversal
│  - Mangle table             │
│  - Filter table (NP rules)  │ <-- NetworkPolicy enforcement
└────┬────────────────────────┘
     │
     ▼
┌─────────────────────────────┐
│  iptables (POSTROUTING)     │ (6) Even more rules
│  - Mangle table             │
│  - NAT table                │
└────┬────────────────────────┘
     │
     ▼
┌─────────┐
│  NIC    │ (7) Transmit
└─────────┘

Complexity: O(n) where n = number of rules
Typical rules per pod: 50-200 (with NetworkPolicies)
```

**eBPF Optimization:**
```
Packet Path with eBPF NetworkPolicy:
┌─────────┐
│  NIC    │ (1) Hardware interrupt
└────┬────┘
     │
     ▼
┌─────────────────────────────┐
│  XDP (eBPF program)         │ (2) Earliest possible hook
│  - Direct NIC DMA buffer    │     O(1) hash map lookup
│  - Can drop/redirect here   │
└────┬────────────────────────┘
     │
     ▼
┌─────────────┐
│   Kernel    │ (3) Minimal kernel processing
│   Network   │
│   Stack     │
└────┬────────┘
     │
     ▼
┌─────────────────────────────┐
│  tc (eBPF program)          │ (4) NetworkPolicy enforcement
│  - Identity-based lookup    │     O(1) hash map
│  - L7 parsing (optional)    │
└────┬────────────────────────┘
     │
     ▼
┌─────────┐
│  NIC    │ (5) Transmit
└─────────┘

Complexity: O(1) hash map lookups
Typical lookups per packet: 2-3
```

### **Scaling Characteristics**

```go
// iptables rules scale linearly
// Rules for N pods with M policies each: O(N * M)

// Example with 1000 pods, 5 policies each:
// Total rules: ~5000-10000 rules
// Packet processing time: 5000 * rule_eval_time

// eBPF scales with hash map size
// Memory: O(N) for identity mappings
// Lookup: O(1) regardless of pod count

// Example with 1000 pods, 5 policies each:
// Hash map entries: ~1000 identities
// Packet processing time: constant
```

### **Memory Usage**

```bash
# Calico Felix (iptables mode)
# Base: 100 MB
# Per 1000 pods: +700 MB
# Per 1000 policies: +200 MB

# Calico Felix (eBPF mode)
# Base: 120 MB
# Per 1000 pods: +500 MB (BPF maps)
# Per 1000 policies: +150 MB

# Cilium (eBPF)
# Base: 200 MB (includes compilation)
# Per 1000 pods: +600 MB (BPF maps, identities)
# Per 1000 policies: +180 MB

# Weave NPC
# Base: 50 MB
# Per 1000 pods: +550 MB
# Per 1000 policies: +100 MB
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting**

### **Calico Troubleshooting**

```bash
# Check Calico status
calicoctl node status

# Verify BGP peering
calicoctl node diags

# View policy
calicoctl get networkpolicy -n myapp -o yaml

# Check workload endpoints
calicoctl get workloadendpoints -n myapp

# Verify iptables rules
kubectl exec -n kube-system calico-node-xxx -- iptables-save | grep cali

# Check Felix logs
kubectl logs -n kube-system -l k8s-app=calico-node -c calico-node

# Dataplane troubleshooting
kubectl exec -n kube-system calico-node-xxx -- calico-node -felix-live
```

### **Cilium Troubleshooting**

```bash
# Check Cilium status
cilium status

# Verify endpoints
cilium endpoint list

# Check policy
cilium policy get

# Monitor drops
cilium monitor --type drop

# View identity
cilium identity list

# Check BPF maps
cilium bpf policy list

# Verify connectivity
cilium connectivity test

# Hubble flows
hubble observe --verdict DROPPED
```

### **Weave Troubleshooting**

```bash
# Check Weave status
kubectl exec -n kube-system weave-net-xxx -c weave -- /home/weave/weave --local status

# View NPC logs
kubectl logs -n kube-system -l name=weave-net -c weave-npc

# Check iptables
kubectl exec -n kube-system weave-net-xxx -c weave-npc -- iptables-save | grep WEAVE

# Network connectivity
kubectl exec -n kube-system weave-net-xxx -c weave -- /home/weave/weave --local status connections
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Migration Guide**

### **Migrating from Calico to Cilium**

```bash
# 1. Install Cilium in migration mode
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set cni.chainingMode=portmap \
  --set enableIPv4Masquerade=false

# 2. Gradually migrate pods
kubectl rollout restart deployment -n myapp

# 3. Verify Cilium endpoints
cilium endpoint list

# 4. Remove Calico (after all pods migrated)
kubectl delete -f calico.yaml

# 5. Reconfigure Cilium for native mode
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --set cni.chainingMode="" \
  --set enableIPv4Masquerade=true
```

### **NetworkPolicy Compatibility**

All standard Kubernetes NetworkPolicies work across CNIs. Extended features require migration:

```yaml
# Calico GlobalNetworkPolicy -> Cilium CiliumClusterwideNetworkPolicy
# BEFORE (Calico)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: global-deny
spec:
  selector: all()
  types:
  - Ingress

# AFTER (Cilium)
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: global-deny
spec:
  endpointSelector: {}
  ingress: []
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Summary**

### **Key Takeaways**

1. **Calico**: Mature, flexible, strong enterprise support
2. **Cilium**: High performance, eBPF-native, excellent observability
3. **Weave Net**: Simple, easy to use, good for smaller clusters
4. **Choose based on**: Performance needs, scale, features, operational complexity

### **Code References**

```plaintext
Calico:
  github.com/projectcalico/calico/kube-controllers/
  github.com/projectcalico/calico/felix/

Cilium:
  github.com/cilium/cilium/pkg/policy/
  github.com/cilium/cilium/pkg/datapath/
  github.com/cilium/cilium/bpf/

Weave:
  github.com/weaveworks/weave/npc/
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
