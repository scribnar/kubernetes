# **Code References: IPVS Implementation**

**Part of**: [kube-proxy Architecture Documentation](../00-README.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Overview**

Comprehensive code navigation for IPVS mode implementation in kube-proxy.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **1. Proxier Structure**

```go
// pkg/proxy/ipvs/proxier.go:250-500
type Proxier struct {
    // Core state
    svcPortMap          ServicePortMap
    endpointsMap        EndpointsMap
    
    // IPVS/ipset/iptables interfaces
    ipvs                utilipvs.Interface
    ipset               utilipset.Interface
    iptables            utiliptables.Interface
    
    // IPVS configuration
    ipvsScheduler       string
    syncPeriod          time.Duration
    minSyncPeriod       time.Duration
    
    // Graceful termination
    gracefuldeleteManager *GracefulTerminationManager
    
    // ipset lists
    ipsetList           map[string]*IPSet
    
    // Metrics
    syncProxyRulesLatency metrics.HistogramVec
    
    // ... 250+ fields total
}
```

**File**: `pkg/proxy/ipvs/proxier.go`
**Lines**: ~2,800 total

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **2. Core Functions**

### **2.1 syncProxyRules**

**Purpose**: Main reconciliation function, syncs IPVS virtual/real servers and ipsets.

```go
// pkg/proxy/ipvs/proxier.go:900-1800
func (proxier *Proxier) syncProxyRules() {
    // Phase 1: Lock
    proxier.mu.Lock()
    defer proxier.mu.Unlock()
    
    // Phase 2: Get changes
    serviceUpdateResult := proxier.svcPortMap.Update(proxier.serviceChanges)
    endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)
    
    // Phase 3: Sync IPVS virtual servers
    for svcName, svcInfo := range proxier.svcPortMap {
        proxier.syncService(svcName, svcInfo)
    }
    
    // Phase 4: Sync ipsets
    proxier.syncIPSet()
    
    // Phase 5: Sync iptables (minimal rules)
    proxier.syncIPTablesRules()
    
    // Phase 6: Cleanup stale entries
    proxier.cleanupStaleServices()
}
```

**Key Line Numbers**:
- Lock: Line ~910
- Changes: Lines ~920-960
- Virtual servers: Lines ~1000-1400
- ipsets: Lines ~1450-1600
- iptables: Lines ~1650-1750
- Cleanup: Lines ~1800-1900

### **2.2 syncService**

**Purpose**: Sync one service (create/update virtual server and real servers).

```go
// pkg/proxy/ipvs/proxier.go:1100-1400
func (proxier *Proxier) syncService(svcName ServicePortName, svcInfo *serviceInfo) {
    // Create virtual server
    vs := &utilipvs.VirtualServer{
        Address:   net.ParseIP(svcInfo.ClusterIP()),
        Port:      uint16(svcInfo.Port()),
        Protocol:  string(svcInfo.Protocol()),
        Scheduler: proxier.ipvsScheduler,  // e.g., "rr"
    }
    
    // Add or update virtual server
    if err := proxier.ipvs.AddVirtualServer(vs); err != nil {
        klog.Error(err)
        return
    }
    
    // Add real servers (endpoints)
    for _, endpoint := range svcInfo.Endpoints() {
        rs := &utilipvs.RealServer{
            Address: net.ParseIP(endpoint.IP),
            Port:    uint16(endpoint.Port),
            Weight:  100,
        }
        
        if err := proxier.ipvs.AddRealServer(vs, rs); err != nil {
            klog.Error(err)
        }
    }
    
    // Update ipsets
    proxier.updateIPSet(svcInfo)
}
```

### **2.3 syncEndpoint**

**Purpose**: Sync one endpoint (add/update/delete real server).

```go
// pkg/proxy/ipvs/proxier.go:1500-1650
func (proxier *Proxier) syncEndpoint(vs *VirtualServer, endpoint *BaseEndpointInfo) {
    rs := &RealServer{
        Address: endpoint.IP(),
        Port:    uint16(endpoint.Port()),
        Weight:  endpoint.GetIsLocal() ? 100 : 0,
    }
    
    // Check if endpoint is terminating
    if endpoint.IsTerminating() {
        // Graceful termination: set weight to 0, wait for drain
        rs.Weight = 0
        proxier.gracefuldeleteManager.MoveRSOutOfGracefulDelete(rs)
    }
    
    // Add or update real server
    if err := proxier.ipvs.UpdateRealServer(vs, rs); err != nil {
        klog.Error(err)
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **3. IPVS Interface**

```go
// pkg/util/ipvs/ipvs.go:50-150
type Interface interface {
    // Virtual server operations
    AddVirtualServer(*VirtualServer) error
    UpdateVirtualServer(*VirtualServer) error
    DeleteVirtualServer(*VirtualServer) error
    GetVirtualServer(*VirtualServer) (*VirtualServer, error)
    GetVirtualServers() ([]*VirtualServer, error)
    
    // Real server operations
    AddRealServer(*VirtualServer, *RealServer) error
    UpdateRealServer(*VirtualServer, *RealServer) error
    DeleteRealServer(*VirtualServer, *RealServer) error
    GetRealServers(*VirtualServer) ([]*RealServer, error)
    
    // Configuration
    ConfigureTimeouts(tcpTimeout, tcpFinTimeout, udpTimeout time.Duration) error
}

type VirtualServer struct {
    Address   net.IP
    Port      uint16
    Protocol  string
    Scheduler string
    Flags     uint32
    Timeout   uint32
}

type RealServer struct {
    Address net.IP
    Port    uint16
    Weight  int32
}
```

**Implementation**: `pkg/util/ipvs/ipvs_linux.go`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **4. ipset Management**

```go
// pkg/proxy/ipvs/ipset.go:50-200
const (
    // ipset names
    kubeLoopBackIPSet              = "KUBE-LOOP-BACK"
    kubeClusterIPSet               = "KUBE-CLUSTER-IP"
    kubeExternalIPSet              = "KUBE-EXTERNAL-IP"
    kubeLoadBalancerSet            = "KUBE-LOAD-BALANCER"
    kubeLoadBalancerLocalSet       = "KUBE-LOAD-BALANCER-LOCAL"
    kubeLoadBalancerFWSet          = "KUBE-LOAD-BALANCER-FW"
    kubeLoadBalancerSourceIPSet    = "KUBE-LOAD-BALANCER-SOURCE-IP"
    kubeLoadBalancerSourceCIDRSet  = "KUBE-LOAD-BALANCER-SOURCE-CIDR"
    kubeNodePortSetTCP             = "KUBE-NODE-PORT-TCP"
    kubeNodePortLocalSetTCP        = "KUBE-NODE-PORT-LOCAL-TCP"
    kubeNodePortSetUDP             = "KUBE-NODE-PORT-UDP"
    kubeNodePortLocalSetUDP        = "KUBE-NODE-PORT-LOCAL-UDP"
    kubeNodePortSetSCTP            = "KUBE-NODE-PORT-SCTP"
    kubeNodePortLocalSetSCTP       = "KUBE-NODE-PORT-LOCAL-SCTP"
    kubeHealthCheckNodePortSet     = "KUBE-HEALTH-CHECK-NODE-PORT"
)

func (proxier *Proxier) syncIPSet() {
    for name, set := range proxier.ipsetList {
        if err := proxier.ipset.EnsureIPSet(set); err != nil {
            klog.Error(err)
        }
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **5. Graceful Termination**

```go
// pkg/proxy/ipvs/graceful_termination.go:50-250
type GracefulTerminationManager struct {
    ipvs         Interface
    rsDeleteChan chan *terminatingRS
}

type terminatingRS struct {
    vs        *VirtualServer
    rs        *RealServer
    startTime time.Time
}

func (m *GracefulTerminationManager) MoveRSOutOfGracefulDelete(rs *RealServer) {
    // Set weight to 0 (stop new connections)
    rs.Weight = 0
    m.ipvs.UpdateRealServer(vs, rs)
    
    // Start monitoring for connection drain
    go m.tryDeleteRS(vs, rs)
}

func (m *GracefulTerminationManager) tryDeleteRS(vs *VirtualServer, rs *RealServer) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    
    timeout := time.After(30 * time.Second)
    
    for {
        select {
        case <-timeout:
            // Force delete after timeout
            m.ipvs.DeleteRealServer(vs, rs)
            return
            
        case <-ticker.C:
            activeConns, _ := m.ipvs.GetRealServerActiveConns(vs, rs)
            if activeConns == 0 {
                // All connections drained
                m.ipvs.DeleteRealServer(vs, rs)
                return
            }
        }
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **6. File Organization**

```
pkg/proxy/ipvs/
├── proxier.go              # Main implementation (2,800 lines)
│   ├── Proxier struct
│   ├── syncProxyRules()
│   ├── Event handlers
│   └── Virtual/Real server management
│
├── graceful_termination.go # Graceful draining (300 lines)
├── ipset.go                # ipset management (400 lines)
├── proxier_test.go         # Unit tests (2,000 lines)
└── util/                   # Utilities
    ├── ipvs_linux.go       # IPVS implementation (800 lines)
    └── ipset_linux.go      # ipset implementation (500 lines)
```

**Related Files**:
```
pkg/proxy/
├── types.go             # Shared types
├── service.go           # ServiceChangeTracker
├── endpoints.go         # EndpointsChangeTracker
└── util.go              # Common utilities

pkg/util/ipvs/
├── ipvs.go              # IPVS interface
└── ipvs_linux.go        # Linux implementation via netlink

pkg/util/ipset/
├── ipset.go             # ipset interface
└── ipset_linux.go       # Linux implementation
```

---

*Last Updated*: Session 13  
*Status*: ✅ Complete  
*Part of*: [kube-proxy Architecture Documentation](../00-README.md)
