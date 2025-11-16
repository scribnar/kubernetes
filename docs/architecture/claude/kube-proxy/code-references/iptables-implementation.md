# **Code References: iptables Implementation**

**Part of**: [kube-proxy Architecture Documentation](../00-README.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Overview**

Comprehensive code navigation for iptables mode implementation in kube-proxy.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **1. Proxier Structure**

```go
// pkg/proxy/iptables/proxier.go:200-400
type Proxier struct {
    // Core state
    svcPortMap          ServicePortMap
    endpointsMap        EndpointsMap
    
    // iptables interface
    iptables            utiliptables.Interface
    masqueradeMark      string
    masqueradeAll       bool
    
    // Sync control
    syncRunner          *async.BoundedFrequencyRunner
    syncPeriod          time.Duration
    minSyncPeriod       time.Duration
    
    // Metrics
    syncProxyRulesLatency metrics.HistogramVec
    
    // ... 200+ fields total
}
```

**File**: `pkg/proxy/iptables/proxier.go`
**Lines**: ~2,500 total

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **2. Core Functions**

### **2.1 syncProxyRules**

**Purpose**: Main reconciliation function, generates and applies all iptables rules.

```go
// pkg/proxy/iptables/proxier.go:900-2000
func (proxier *Proxier) syncProxyRules() {
    // Phase 1: Lock and defer unlock
    proxier.mu.Lock()
    defer proxier.mu.Unlock()
    
    // Phase 2: Get service/endpoint changes
    serviceUpdateResult := proxier.svcPortMap.Update(proxier.serviceChanges)
    endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)
    
    // Phase 3: Build rule chains
    natChains := make(map[Chain][]byte)
    natRules := make(map[Chain][]byte)
    
    // Phase 4: Write base chains
    proxier.writeBaseChains(natChains, natRules)
    
    // Phase 5: Write service chains
    for svcName, svcInfo := range proxier.svcPortMap {
        proxier.writeServiceChain(svcName, svcInfo, natChains, natRules)
    }
    
    // Phase 6: Apply rules via iptables-restore
    proxier.iptables.RestoreAll(natRules, utiliptables.NoFlushTables, utiliptables.RestoreCounters)
}
```

**Key Line Numbers**:
- Lock acquisition: Line ~910
- Change tracking: Lines ~920-950
- Base chains: Lines ~1000-1100
- Service chains: Lines ~1200-1800
- iptables-restore: Lines ~1900-2000

### **2.2 writeServiceChain**

**Purpose**: Generate rules for one service (KUBE-SVC-* chain).

```go
// pkg/proxy/iptables/proxier.go:1200-1400
func (proxier *Proxier) writeServiceChain(svcName ServicePortName, svcInfo *serviceInfo, chains, rules map[Chain][]byte) {
    svcChain := servicePortChainName(svcName, protocol)
    
    // Create service chain
    chains[svcChain] = []byte{}
    
    // Add jump rules from KUBE-SERVICES
    proxier.writeJumpRule(kubeServicesChain, svcChain, svcInfo.ClusterIP(), svcInfo.Port())
    
    // If NodePort, add KUBE-NODEPORTS rules
    if svcInfo.NodePort() != 0 {
        proxier.writeNodePortRules(svcChain, svcInfo)
    }
    
    // Write endpoint selection rules (probability-based)
    proxier.writeEndpointRules(svcChain, svcInfo, endpoints)
}
```

### **2.3 writeEndpointRules**

**Purpose**: Generate probability-based load balancing rules.

```go
// pkg/proxy/iptables/proxier.go:1600-1800
func (proxier *Proxier) writeEndpointRules(svcChain Chain, svcInfo *serviceInfo, endpoints []Endpoint) {
    numEndpoints := len(endpoints)
    
    for i, endpoint := range endpoints {
        epChain := servicePortEndpointChainName(svcName, protocol, endpoint)
        
        // Calculate probability: 1/(n-i)
        if i < numEndpoints-1 {
            probability := 1.0 / float64(numEndpoints-i)
            
            // Jump to endpoint with probability
            writeLine(natRules, []string{
                "-A", string(svcChain),
                "-m", "statistic",
                "--mode", "random",
                "--probability", fmt.Sprintf("%.8f", probability),
                "-j", string(epChain),
            }...)
        } else {
            // Last endpoint: unconditional jump
            writeLine(natRules, []string{
                "-A", string(svcChain),
                "-j", string(epChain),
            }...)
        }
        
        // Write endpoint chain (DNAT)
        proxier.writeEndpointChain(epChain, endpoint)
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **3. Key Constants and Chain Names**

```go
// pkg/proxy/iptables/proxier.go:56-100
const (
    // Base chains
    kubeServicesChain        Chain = "KUBE-SERVICES"
    kubeNodePortsChain       Chain = "KUBE-NODEPORTS"
    kubePostroutingChain     Chain = "KUBE-POSTROUTING"
    KubeMarkMasqChain        Chain = "KUBE-MARK-MASQ"
    KubeMarkDropChain        Chain = "KUBE-MARK-DROP"
    kubeForwardChain         Chain = "KUBE-FORWARD"
    kubeProxyFirewallChain   Chain = "KUBE-PROXY-FIREWALL"
)

// Masquerade mark
const (
    KubeMarkMasqBit = 14
    KubeMarkMasq    = 1 << uint(KubeMarkMasqBit)  // 0x4000
)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **4. Event Handlers**

### **4.1 Service Events**

```go
// pkg/proxy/iptables/proxier.go:600-650
func (proxier *Proxier) OnServiceAdd(service *v1.Service) {
    proxier.OnServiceUpdate(nil, service)
}

func (proxier *Proxier) OnServiceUpdate(oldService, service *v1.Service) {
    if proxier.serviceChanges.Update(oldService, service) && proxier.isInitialized() {
        proxier.Sync()
    }
}

func (proxier *Proxier) OnServiceDelete(service *v1.Service) {
    proxier.OnServiceUpdate(service, nil)
}
```

### **4.2 Endpoint Events**

```go
// pkg/proxy/iptables/proxier.go:680-730
func (proxier *Proxier) OnEndpointSliceAdd(endpointSlice *discovery.EndpointSlice) {
    if proxier.endpointsChanges.EndpointSliceUpdate(endpointSlice, false) && proxier.isInitialized() {
        proxier.Sync()
    }
}

func (proxier *Proxier) OnEndpointSliceUpdate(oldEndpointSlice, endpointSlice *discovery.EndpointSlice) {
    if proxier.endpointsChanges.EndpointSliceUpdate(endpointSlice, false) && proxier.isInitialized() {
        proxier.Sync()
    }
}

func (proxier *Proxier) OnEndpointSliceDelete(endpointSlice *discovery.EndpointSlice) {
    if proxier.endpointsChanges.EndpointSliceUpdate(endpointSlice, true) && proxier.isInitialized() {
        proxier.Sync()
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **5. Utilities**

### **5.1 iptables Interface**

```go
// pkg/util/iptables/iptables.go:50-200
type Interface interface {
    GetVersion() (string, error)
    EnsureChain(table Table, chain Chain) (bool, error)
    FlushChain(table Table, chain Chain) error
    DeleteChain(table Table, chain Chain) error
    EnsureRule(position RulePosition, table Table, chain Chain, args ...string) (bool, error)
    DeleteRule(table Table, chain Chain, args ...string) error
    IsIPv6() bool
    SaveInto(table Table, buffer *bytes.Buffer) error
    Restore(table Table, data []byte, flush FlushFlag, counters RestoreCountersFlag) error
    RestoreAll(data []byte, flush FlushFlag, counters RestoreCountersFlag) error
}
```

**Implementation**: `pkg/util/iptables/iptables_linux.go`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **6. File Organization**

```
pkg/proxy/iptables/
├── proxier.go           # Main implementation (2,500 lines)
│   ├── Proxier struct
│   ├── syncProxyRules()
│   ├── Event handlers
│   └── Rule generation
│
├── proxier_test.go      # Unit tests (1,500 lines)
│
└── [deprecated files]   # Legacy code
```

**Related Files**:
```
pkg/proxy/
├── types.go             # Shared types
├── service.go           # ServiceChangeTracker
├── endpoints.go         # EndpointsChangeTracker
└── util.go              # Common utilities

pkg/util/iptables/
├── iptables.go          # Interface definition
└── iptables_linux.go    # Linux implementation
```

---

*Last Updated*: Session 13  
*Status*: ✅ Complete  
*Part of*: [kube-proxy Architecture Documentation](../00-README.md)
