# **Network Policy Security Architecture**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Deep architectural analysis of Kubernetes NetworkPolicy implementation

**Target Audience**:
- Platform engineers designing network security for Kubernetes
- Architects implementing zero-trust network architectures
- SREs troubleshooting network policy issues
- CNI plugin developers building policy enforcement

**Scope**: API design, validation logic, policy evaluation model, and enforcement architecture

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 NetworkPolicy Architecture Overview**

### **Design Philosophy**

NetworkPolicy follows a **whitelist-only additive model**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NetworkPolicy Design Principles                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │
│  │   Declarative   │  │   Composable    │  │   Enforcement   │          │
│  │   Only          │  │   Rules         │  │   Delegated     │          │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘          │
│           │                    │                    │                    │
│  • No imperative       • Multiple policies   • Kubernetes is             │
│    commands              can select same       just the API               │
│  • Pure API objects      pod (additive)     • CNI plugins do             │
│  • Immutable rules     • OR logic between     enforcement                │
│                          different policies                              │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │
│  │   No Explicit   │  │   Empty Field   │  │   Namespace     │          │
│  │   Deny Rules    │  │   Semantics     │  │   Scoped        │          │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘          │
│           │                    │                    │                    │
│  • All traffic        • Empty selector =   • Policies apply to          │
│    allowed by default   match ALL            pods in namespace          │
│  • Deny via absence   • Empty rules =      • Cannot span                │
│    of allow rules       deny ALL             namespaces                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### **Component Architecture**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 NetworkPolicy Component Architecture                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                  │
│  │   kubectl   │───▶│  API Server │───▶│    etcd     │                  │
│  │   (client)  │    │ (validation)│    │  (storage)  │                  │
│  └─────────────┘    └──────┬──────┘    └─────────────┘                  │
│                            │                                             │
│                            ▼                                             │
│                    ┌─────────────┐                                       │
│                    │    List/    │                                       │
│                    │    Watch    │                                       │
│                    └──────┬──────┘                                       │
│                           │                                              │
│         ┌─────────────────┼─────────────────┐                            │
│         ▼                 ▼                 ▼                            │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                    │
│  │   Calico    │   │   Cilium    │   │   Antrea    │                    │
│  │   Agent     │   │   Agent     │   │   Agent     │                    │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘                    │
│         │                 │                 │                            │
│         ▼                 ▼                 ▼                            │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                    │
│  │  iptables/  │   │    eBPF     │   │    OVS      │                    │
│  │   ipsets    │   │   datapath  │   │   flows     │                    │
│  └─────────────┘   └─────────────┘   └─────────────┘                    │
│                                                                          │
│         ▼                 ▼                 ▼                            │
│  ┌───────────────────────────────────────────────────────┐              │
│  │              Kernel Network Stack                      │              │
│  │          (packet filtering & forwarding)               │              │
│  └───────────────────────────────────────────────────────┘              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📦 Core API Types**

### **NetworkPolicy Resource**

**File**: `staging/src/k8s.io/api/networking/v1/types.go:30-46`

```go
// NetworkPolicy describes what network traffic is allowed for a set of Pods
type NetworkPolicy struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

    // spec represents the specification of the desired behavior for this NetworkPolicy.
    Spec NetworkPolicySpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

    // Status intentionally does not have a status yet
    // Status NetworkPolicyStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

**Design Decision**: No status subresource. The status field is reserved (protobuf tag 3) for future implementation without breaking compatibility.

**Rationale**: NetworkPolicy is purely declarative. Status would require:
- CNI plugins to report enforcement state
- Reconciliation loop for status updates
- Complex multi-plugin aggregation

### **NetworkPolicySpec**

**File**: `staging/src/k8s.io/api/networking/v1/types.go:61-106`

```go
type NetworkPolicySpec struct {
    // podSelector selects the pods to which this NetworkPolicy object applies
    // Empty podSelector matches all pods in the namespace
    PodSelector metav1.LabelSelector `json:"podSelector" protobuf:"bytes,1,opt,name=podSelector"`

    // ingress is a list of ingress rules to be applied to the selected pods
    // Empty list denies all ingress traffic
    Ingress []NetworkPolicyIngressRule `json:"ingress,omitempty" protobuf:"bytes,2,rep,name=ingress"`

    // egress is a list of egress rules to be applied to the selected pods
    // Empty list denies all egress traffic
    Egress []NetworkPolicyEgressRule `json:"egress,omitempty" protobuf:"bytes,3,rep,name=egress"`

    // policyTypes is a list of rule types that the NetworkPolicy relates to
    // Valid options: "Ingress", "Egress", or "Ingress,Egress"
    PolicyTypes []PolicyType `json:"policyTypes,omitempty" protobuf:"bytes,4,rep,name=policyTypes,casttype=PolicyType"`
}
```

### **Policy Type Inference**

**File**: `pkg/apis/networking/v1/defaults.go:38-46`

```go
func SetDefaults_NetworkPolicy(obj *networkingv1.NetworkPolicy) {
    if len(obj.Spec.PolicyTypes) == 0 {
        // Ingress is always inferred
        obj.Spec.PolicyTypes = []networkingv1.PolicyType{networkingv1.PolicyTypeIngress}

        // Egress only inferred if egress rules exist
        if len(obj.Spec.Egress) != 0 {
            obj.Spec.PolicyTypes = append(obj.Spec.PolicyTypes, networkingv1.PolicyTypeEgress)
        }
    }
}
```

**Critical Implications**:

| Spec Configuration | Inferred PolicyTypes | Behavior |
|--------------------|---------------------|----------|
| Empty `ingress`, empty `egress`, empty `policyTypes` | `[Ingress]` | Deny all ingress, allow all egress |
| Non-empty `egress`, empty `policyTypes` | `[Ingress, Egress]` | Both directions controlled |
| Explicit `policyTypes: [Egress]` | `[Egress]` | Only egress controlled |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔒 Rule Definitions**

### **Ingress and Egress Rules**

**File**: `staging/src/k8s.io/api/networking/v1/types.go:110-151`

```go
type NetworkPolicyIngressRule struct {
    // ports is a list of ports which should be made accessible
    // Empty or missing matches all ports
    Ports []NetworkPolicyPort `json:"ports,omitempty" protobuf:"bytes,1,rep,name=ports"`

    // from is a list of sources which should be able to access the pods
    // Empty or missing matches all sources
    From []NetworkPolicyPeer `json:"from,omitempty" protobuf:"bytes,2,rep,name=from"`
}

type NetworkPolicyEgressRule struct {
    // ports is a list of destination ports for outgoing traffic
    // Empty or missing matches all ports
    Ports []NetworkPolicyPort `json:"ports,omitempty" protobuf:"bytes,1,rep,name=ports"`

    // to is a list of destinations for outgoing traffic
    // Empty or missing matches all destinations
    To []NetworkPolicyPeer `json:"to,omitempty" protobuf:"bytes,2,rep,name=to"`
}
```

### **NetworkPolicyPort**

**File**: `staging/src/k8s.io/api/networking/v1/types.go:154-173`

```go
type NetworkPolicyPort struct {
    // protocol represents the protocol (TCP, UDP, or SCTP) which traffic must match
    // Default is TCP
    Protocol *v1.Protocol `json:"protocol,omitempty" protobuf:"bytes,1,opt,name=protocol,casttype=k8s.io/api/core/v1.Protocol"`

    // port represents the port on the given protocol
    // Can be numeric (0-65535) or named port
    Port *intstr.IntOrString `json:"port,omitempty" protobuf:"bytes,2,opt,name=port"`

    // endPort indicates that the range of ports from port to endPort (inclusive)
    // Must be defined only when port is numeric (not named)
    EndPort *int32 `json:"endPort,omitempty" protobuf:"varint,3,opt,name=endPort"`
}
```

**Protocol Default** (from `defaults.go:30-36`):

```go
func SetDefaults_NetworkPolicyPort(obj *networkingv1.NetworkPolicyPort) {
    // TCP is the default protocol, NOT "all protocols"
    if obj.Protocol == nil {
        proto := v1.ProtocolTCP
        obj.Protocol = &proto
    }
}
```

### **NetworkPolicyPeer**

**File**: `staging/src/k8s.io/api/networking/v1/types.go:193-216`

```go
type NetworkPolicyPeer struct {
    // podSelector selects pods within this namespace
    // Cannot be set when IPBlock is set
    PodSelector *metav1.LabelSelector `json:"podSelector,omitempty" protobuf:"bytes,1,opt,name=podSelector"`

    // namespaceSelector selects namespaces
    // All pods in selected namespaces are matched
    // Cannot be set when IPBlock is set
    NamespaceSelector *metav1.LabelSelector `json:"namespaceSelector,omitempty" protobuf:"bytes,2,opt,name=namespaceSelector"`

    // ipBlock defines CIDR and exceptions for external traffic
    // Cannot be set when PodSelector or NamespaceSelector are set
    IPBlock *IPBlock `json:"ipBlock,omitempty" protobuf:"bytes,3,rep,name=ipBlock"`
}
```

### **Peer Selection Logic**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NetworkPolicyPeer Selection Logic                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Configuration              │  Matches                                   │
│  ──────────────────────────┼──────────────────────────────────────────  │
│  PodSelector only           │  Pods in SAME namespace matching labels   │
│  ──────────────────────────┼──────────────────────────────────────────  │
│  NamespaceSelector only     │  ALL pods in matched namespaces           │
│  ──────────────────────────┼──────────────────────────────────────────  │
│  Both PodSelector AND       │  Pods matching PodSelector labels IN      │
│  NamespaceSelector          │  namespaces matching NamespaceSelector    │
│  ──────────────────────────┼──────────────────────────────────────────  │
│  IPBlock only               │  Traffic from/to specified CIDR ranges    │
│                             │  (cannot combine with label selectors)    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### **IPBlock Definition**

**File**: `staging/src/k8s.io/api/networking/v1/types.go:178-189`

```go
type IPBlock struct {
    // cidr is a string representing the IPBlock
    // Valid examples: "192.168.1.0/24", "2001:db8::/64"
    CIDR string `json:"cidr" protobuf:"bytes,1,name=cidr"`

    // except is a slice of CIDRs that should not be included within the IPBlock
    // Values must be strict subsets of the CIDR
    Except []string `json:"except,omitempty" protobuf:"bytes,2,rep,name=except"`
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **✅ Validation Layer**

### **NetworkPolicy Validation**

**File**: `pkg/apis/networking/validation/validation.go:188-207`

```go
func ValidateNetworkPolicy(np *networking.NetworkPolicy) field.ErrorList {
    allErrs := apivalidation.ValidateObjectMeta(&np.ObjectMeta, true, ValidateNetworkPolicyName, field.NewPath("metadata"))

    specPath := field.NewPath("spec")
    allErrs = append(allErrs, ValidateNetworkPolicySpec(&np.Spec, specPath)...)

    return allErrs
}
```

### **Port Validation**

**File**: `pkg/apis/networking/validation/validation.go:71-104`

```go
func ValidateNetworkPolicyPort(port *networking.NetworkPolicyPort, portPath *field.Path) field.ErrorList {
    var allErrs field.ErrorList

    // Protocol validation
    if port.Protocol != nil && !supportedProtocols.Has(*port.Protocol) {
        allErrs = append(allErrs, field.NotSupported(portPath.Child("protocol"),
            *port.Protocol, sets.List(supportedProtocols)))
    }

    // Port validation
    if port.Port != nil {
        if port.Port.Type == intstr.Int {
            // Numeric port: 0-65535
            for _, msg := range validation.IsValidPortNum(int(port.Port.IntVal)) {
                allErrs = append(allErrs, field.Invalid(portPath.Child("port"), port.Port.IntVal, msg))
            }
        } else {
            // Named port: validate as DNS label
            for _, msg := range validation.IsValidPortName(port.Port.StrVal) {
                allErrs = append(allErrs, field.Invalid(portPath.Child("port"), port.Port.StrVal, msg))
            }
        }
    }

    // EndPort validation
    if port.EndPort != nil {
        if port.Port == nil {
            allErrs = append(allErrs, field.Invalid(portPath.Child("endPort"),
                *port.EndPort, "endPort requires port to be defined"))
        } else if port.Port.Type == intstr.String {
            allErrs = append(allErrs, field.Invalid(portPath.Child("endPort"),
                *port.EndPort, "endPort cannot be used with named port"))
        } else if *port.EndPort < port.Port.IntVal {
            allErrs = append(allErrs, field.Invalid(portPath.Child("endPort"),
                *port.EndPort, "endPort must be >= port"))
        }
    }

    return allErrs
}
```

### **Peer Validation**

**File**: `pkg/apis/networking/validation/validation.go:107-134`

```go
func ValidateNetworkPolicyPeer(peer *networking.NetworkPolicyPeer, peerPath *field.Path) field.ErrorList {
    var allErrs field.ErrorList
    numPeers := 0

    if peer.PodSelector != nil {
        numPeers++
        allErrs = append(allErrs, unversionedvalidation.ValidateLabelSelector(
            peer.PodSelector, unversionedvalidation.LabelSelectorValidationOptions{},
            peerPath.Child("podSelector"))...)
    }

    if peer.NamespaceSelector != nil {
        numPeers++
        allErrs = append(allErrs, unversionedvalidation.ValidateLabelSelector(
            peer.NamespaceSelector, unversionedvalidation.LabelSelectorValidationOptions{},
            peerPath.Child("namespaceSelector"))...)
    }

    if peer.IPBlock != nil {
        numPeers++
        allErrs = append(allErrs, ValidateIPBlock(peer.IPBlock, peerPath.Child("ipBlock"))...)
    }

    // Must have at least one selector
    if numPeers == 0 {
        allErrs = append(allErrs, field.Required(peerPath, "must have at least one peer type"))
    }

    // IPBlock cannot combine with label selectors
    if peer.IPBlock != nil && (peer.PodSelector != nil || peer.NamespaceSelector != nil) {
        allErrs = append(allErrs, field.Forbidden(peerPath.Child("ipBlock"),
            "ipBlock cannot be set when podSelector or namespaceSelector is also set"))
    }

    return allErrs
}
```

### **IPBlock Validation**

**File**: `pkg/apis/networking/validation/validation.go:246-275`

```go
func ValidateIPBlock(ipBlock *networking.IPBlock, fldPath *field.Path) field.ErrorList {
    var allErrs field.ErrorList

    // Validate CIDR
    if _, _, err := netutils.ParseCIDRSloppy(ipBlock.CIDR); err != nil {
        allErrs = append(allErrs, field.Invalid(fldPath.Child("cidr"), ipBlock.CIDR,
            "invalid CIDR notation"))
        return allErrs
    }

    // Validate Except entries
    for i, except := range ipBlock.Except {
        exceptPath := fldPath.Child("except").Index(i)

        _, exceptNet, err := netutils.ParseCIDRSloppy(except)
        if err != nil {
            allErrs = append(allErrs, field.Invalid(exceptPath, except,
                "invalid CIDR notation"))
            continue
        }

        // Except must be strict subset of CIDR
        _, cidrNet, _ := netutils.ParseCIDRSloppy(ipBlock.CIDR)
        if !cidrNet.Contains(exceptNet.IP) {
            allErrs = append(allErrs, field.Invalid(exceptPath, except,
                "must be a strict subset of the CIDR"))
        }
    }

    return allErrs
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💾 Storage & Registry**

### **NetworkPolicy Strategy**

**File**: `pkg/registry/networking/networkpolicy/strategy.go:32-99`

```go
type networkPolicyStrategy struct {
    runtime.ObjectTyper
    names.NameGenerator
}

var Strategy = networkPolicyStrategy{legacyscheme.Scheme, names.SimpleNameGenerator}

func (networkPolicyStrategy) NamespaceScoped() bool {
    return true  // NetworkPolicies are namespace-scoped
}

func (networkPolicyStrategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
    networkPolicy := obj.(*networking.NetworkPolicy)
    networkPolicy.Generation = 1  // Initial generation
    networkPolicy.Status = networking.NetworkPolicyStatus{}
}

func (networkPolicyStrategy) PrepareForUpdate(ctx context.Context, obj, old runtime.Object) {
    newNetworkPolicy := obj.(*networking.NetworkPolicy)
    oldNetworkPolicy := old.(*networking.NetworkPolicy)

    // Increment generation on spec change
    if !reflect.DeepEqual(oldNetworkPolicy.Spec, newNetworkPolicy.Spec) {
        newNetworkPolicy.Generation = oldNetworkPolicy.Generation + 1
    }

    newNetworkPolicy.Status = oldNetworkPolicy.Status  // Preserve status
}

func (networkPolicyStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
    networkPolicy := obj.(*networking.NetworkPolicy)
    return validation.ValidateNetworkPolicy(networkPolicy)
}

func (networkPolicyStrategy) AllowCreateOnUpdate() bool {
    return false  // Must use POST for creation
}

func (networkPolicyStrategy) AllowUnconditionalUpdate() bool {
    return true  // No conditional updates required
}
```

### **REST Storage**

**File**: `pkg/registry/networking/networkpolicy/storage/storage.go:37-65`

```go
type REST struct {
    *genericregistry.Store
}

func NewREST(optsGetter generic.RESTOptionsGetter) (*REST, error) {
    store := &genericregistry.Store{
        NewFunc:                   func() runtime.Object { return &networking.NetworkPolicy{} },
        NewListFunc:               func() runtime.Object { return &networking.NetworkPolicyList{} },
        DefaultQualifiedResource:  networking.Resource("networkpolicies"),
        SingularQualifiedResource: networking.Resource("networkpolicy"),

        CreateStrategy: networkpolicy.Strategy,
        UpdateStrategy: networkpolicy.Strategy,
        DeleteStrategy: networkpolicy.Strategy,

        TableConvertor: printerstorage.TableConvertor{
            TableGenerator: printers.NewTableGenerator().With(
                printersinternal.AddHandlers,
            ),
        },
    }

    options := &generic.StoreOptions{RESTOptions: optsGetter}
    if err := store.CompleteWithOptions(options); err != nil {
        return nil, err
    }

    return &REST{store}, nil
}

// ShortNames returns the short names for NetworkPolicy (netpol)
func (r *REST) ShortNames() []string {
    return []string{"netpol"}
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Policy Evaluation Model**

### **Additive Rule Semantics**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NetworkPolicy Additive Model                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Pod A selected by:                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                      │
│  │  Policy 1   │  │  Policy 2   │  │  Policy 3   │                      │
│  │ Allow: 80   │  │ Allow: 443  │  │ Allow: 8080 │                      │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                      │
│         │                │                │                              │
│         └────────────────┼────────────────┘                              │
│                          │                                               │
│                          ▼                                               │
│              ┌───────────────────────┐                                   │
│              │  Combined Allow Set:  │                                   │
│              │   80 OR 443 OR 8080   │                                   │
│              └───────────────────────┘                                   │
│                                                                          │
│  Rules are COMBINED (OR logic), not intersected (AND logic)              │
│  This enables policy composition from different teams/systems            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### **Empty Field Semantics**

| Field | Empty Meaning | Example Effect |
|-------|---------------|----------------|
| `spec.podSelector: {}` | Select ALL pods in namespace | Policy applies to entire namespace |
| `spec.ingress: []` | Deny all ingress | No incoming traffic allowed |
| `spec.egress: []` | Deny all egress | No outgoing traffic allowed |
| `ingress[].from: []` | Allow from anywhere | All sources can reach selected pods |
| `ingress[].ports: []` | Allow all ports | All port/protocol combinations allowed |

### **Default Deny Patterns**

**Default Deny All Ingress**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress
```

**Default Deny All Egress**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny all egress
```

**Default Deny All Traffic**:

```yaml
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
  # Empty rules for both = deny everything
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌐 CNI Plugin Enforcement**

### **Enforcement Architecture**

Kubernetes does NOT enforce NetworkPolicy. CNI plugins implement enforcement:

| CNI Plugin | Enforcement Method | Performance Characteristics |
|------------|-------------------|---------------------------|
| **Calico** | iptables/ipsets, eBPF | High performance, complex rule management |
| **Cilium** | eBPF datapath | Kernel-level filtering, excellent performance |
| **Antrea** | Open vSwitch flows | Declarative flow rules, good for visualization |
| **Weave Net** | iptables | Simple but limited scalability |
| **Flannel** | None | Does not support NetworkPolicy |

### **Plugin Watch Pattern**

```go
// Example CNI plugin watching NetworkPolicies
func (c *PolicyController) Run(stopCh <-chan struct{}) {
    // Watch all NetworkPolicies in cluster
    policyInformer := informers.NewSharedInformerFactory(
        clientset, time.Second*30,
    ).Networking().V1().NetworkPolicies()

    policyInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            c.syncPolicy(obj.(*v1.NetworkPolicy))
        },
        UpdateFunc: func(old, new interface{}) {
            c.syncPolicy(new.(*v1.NetworkPolicy))
        },
        DeleteFunc: func(obj interface{}) {
            c.deletePolicy(obj.(*v1.NetworkPolicy))
        },
    })

    // Also watch Pods for label changes
    podInformer := ...

    // Also watch Namespaces for label changes
    namespaceInformer := ...
}
```

### **Label Resolution Complexity**

CNI plugins must:

1. **Watch all NetworkPolicies** in the cluster
2. **Watch all Pods** for label changes
3. **Watch all Namespaces** for label changes
4. **Compute affected policies** when any change occurs
5. **Update datapath rules** for affected pods

**Complexity**: O(policies × pods × namespaces) in worst case

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚧 Production Patterns**

### **Micro-segmentation Architecture**

```yaml
# 1. Default deny all in namespace
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

---
# 2. Allow DNS egress for all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
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
# 3. Allow specific service communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### **Multi-Tenant Isolation**

```yaml
# Namespace-level isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: a  # Only from same tenant's namespaces
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tenant: a
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system  # Allow DNS
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### **External Traffic Control**

```yaml
# Allow ingress from specific external CIDRs
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-lb
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8  # Internal load balancer network
        except:
        - 10.0.1.0/24     # Except management subnet
    ports:
    - protocol: TCP
      port: 443
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting Guide**

### **Common Issues**

| Issue | Symptom | Root Cause | Solution |
|-------|---------|------------|----------|
| **Policy not applied** | Traffic allowed/denied unexpectedly | CNI plugin not supporting NetworkPolicy | Verify CNI supports NetworkPolicy |
| **DNS blocked** | Pods cannot resolve names | Egress to kube-dns not allowed | Add DNS egress rule |
| **Cross-namespace blocked** | Service calls fail | Missing namespaceSelector | Add proper namespace selection |
| **Port range not working** | Only first port works | Old CNI version | Upgrade CNI for endPort support |
| **Named port mismatch** | Connection refused | Port name doesn't match container | Verify pod spec port names |

### **Diagnostic Commands**

```bash
# List all network policies in namespace
kubectl get networkpolicy -n production

# Describe policy to see computed rules
kubectl describe networkpolicy frontend-policy -n production

# Check if policy selects specific pod
kubectl get pod web-server -n production --show-labels
kubectl get networkpolicy -n production -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.podSelector.matchLabels}{"\n"}{end}'

# Test connectivity from specific pod
kubectl exec -it debug-pod -n production -- nc -vz backend-service 8080

# Check CNI plugin logs (Calico example)
kubectl logs -n kube-system -l k8s-app=calico-node --tail=100

# Inspect iptables rules (Calico)
kubectl exec -it -n kube-system calico-node-xxx -- iptables -L -n -v | grep -A5 "cali"

# Inspect eBPF maps (Cilium)
kubectl exec -it -n kube-system cilium-xxx -- cilium bpf policy get
```

### **Policy Debugging Flow**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NetworkPolicy Debugging Flow                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Verify CNI supports NetworkPolicy                                    │
│     └─▶ Check CNI documentation / kubectl get pods -n kube-system        │
│                                                                          │
│  2. Verify policy exists and selects pod                                 │
│     └─▶ kubectl get netpol -n <ns> -o wide                               │
│     └─▶ Compare labels with podSelector                                  │
│                                                                          │
│  3. Check policy rules cover traffic                                     │
│     └─▶ Verify from/to peers match source/destination                    │
│     └─▶ Verify ports match required protocol/port                        │
│                                                                          │
│  4. Check default deny is intentional                                    │
│     └─▶ Empty rules = deny all of that type                              │
│     └─▶ Missing policyTypes may not control egress                       │
│                                                                          │
│  5. Inspect CNI plugin state                                             │
│     └─▶ Check plugin logs for errors                                     │
│     └─▶ Inspect datapath rules (iptables/eBPF/OVS)                       │
│                                                                          │
│  6. Test with tcpdump/packet capture                                     │
│     └─▶ Capture on both source and destination pods                      │
│     └─▶ Identify where packets are dropped                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚠️ Limitations and Trade-offs**

### **Design Limitations**

| Limitation | Impact | Workaround |
|------------|--------|-----------|
| **No explicit deny rules** | Cannot deny specific traffic while allowing rest | Use deny-all + specific allows |
| **No rule precedence** | All rules have equal weight | Design policies to avoid conflicts |
| **No status feedback** | Cannot see if policy is enforced | Use CNI-specific observability |
| **ICMP not supported** | Cannot allow/deny ping | Use diagnostic namespaces |
| **No logging built-in** | Cannot audit blocked traffic | Use CNI logging features |
| **Named ports complexity** | Must match container port names exactly | Use numeric ports when possible |

### **Performance Considerations**

- **Policy count**: O(n) evaluation with matching policy count
- **Label resolution**: Delegated to CNI plugins
- **Namespace traversal**: Required for cross-namespace rules
- **IPBlock matching**: Standard prefix matching with except list overhead

### **Security Considerations**

1. **CNI trust**: Enforcement depends on CNI plugin correctness
2. **Label spoofing**: Pods can set arbitrary labels (use admission control)
3. **CIDR overlap**: Carefully manage IPBlock except lists
4. **Egress to metadata**: Block cloud provider metadata services (169.254.169.254)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Related Documentation**

### **Kubernetes Source Files**

| Component | File Path |
|-----------|-----------|
| API Types | `staging/src/k8s.io/api/networking/v1/types.go` |
| Validation | `pkg/apis/networking/validation/validation.go` |
| Defaults | `pkg/apis/networking/v1/defaults.go` |
| Strategy | `pkg/registry/networking/networkpolicy/strategy.go` |
| Storage | `pkg/registry/networking/networkpolicy/storage/storage.go` |
| Installation | `pkg/apis/networking/install/install.go` |

### **Related Architecture Docs**

- `security/01-pod-security-standards.md` - Pod-level security
- `security/05-rbac-patterns-troubleshooting.md` - API authorization

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **✨ Key Takeaways**

1. **Declarative Only**: NetworkPolicy is pure API declaration; enforcement is by CNI
2. **Additive Model**: Multiple policies combine with OR logic, enabling composition
3. **Empty = Match All**: Empty selectors match everything; empty rules deny all
4. **Namespace Scoped**: Policies only affect pods in their own namespace
5. **No Status**: Cannot query enforcement state from API; use CNI observability
6. **Default TCP**: Unspecified protocol defaults to TCP, not "all protocols"
7. **Label-Based**: All selection uses Kubernetes label selectors (except IPBlock)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Last Updated**: 2024-11-19
**Kubernetes Version**: 1.31+
