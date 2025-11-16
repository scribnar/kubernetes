# **Code References: Entry Points**

**Part of**: [kube-proxy Architecture Documentation](../00-README.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Overview**

Complete code navigation guide for kube-proxy entry points and initialization flow, with exact file locations and line numbers.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **1. Main Entry Point**

### **1.1 Main Function**

```go
// cmd/kube-proxy/proxy.go:30-50
package main

import (
    "k8s.io/component-base/cli"
    "k8s.io/kubernetes/cmd/kube-proxy/app"
)

func main() {
    command := app.NewProxyCommand()
    code := cli.Run(command)
    os.Exit(code)
}
```

**Call Chain**:
```
main() 
  → app.NewProxyCommand() [cmd/kube-proxy/app/server.go:150]
  → cmd.Execute()
  → Run() [cmd/kube-proxy/app/server.go:500]
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **2. Command Setup**

### **2.1 NewProxyCommand**

```go
// cmd/kube-proxy/app/server.go:150-250
func NewProxyCommand() *cobra.Command {
    opts := NewOptions()
    
    cmd := &cobra.Command{
        Use: "kube-proxy",
        RunE: func(cmd *cobra.Command, args []string) error {
            return opts.Run()
        },
    }
    
    opts.AddFlags(cmd.Flags())
    return cmd
}
```

### **2.2 Options Structure**

```go
// cmd/kube-proxy/app/server.go:100-130
type Options struct {
    ConfigFile      string
    config          *kubeproxyconfig.KubeProxyConfiguration
    master          string
    InitAndExit     bool
    CleanupAndExit  bool
    // ... more fields
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **3. Initialization Flow**

### **3.1 Run Method**

```go
// cmd/kube-proxy/app/server.go:500-700
func (o *Options) Run() error {
    // 1. Load configuration
    config, err := o.loadConfig()
    
    // 2. Create clients
    client, eventClient, err := createClients(config)
    
    // 3. Detect proxy mode
    proxyMode := detectProxyMode(config)
    
    // 4. Create proxier
    proxier, err := createProxier(config, proxyMode, client)
    
    // 5. Setup informers
    informers := setupInformers(client, proxier)
    
    // 6. Start sync loop
    go proxier.SyncLoop()
    
    // 7. Start informers
    informers.Start(stopCh)
    
    // 8. Wait for shutdown
    <-stopCh
    return nil
}
```

### **3.2 Proxier Factory**

```go
// cmd/kube-proxy/app/server_linux.go:100-200
func newProxyServer(...) (*ProxyServer, error) {
    switch proxyMode {
    case "iptables":
        return newIPTablesProxier(...)
    case "ipvs":
        return newIPVSProxier(...)
    case "nftables":
        return newNFTablesProxier(...)
    default:
        return nil, fmt.Errorf("unknown proxy mode: %s", proxyMode)
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **4. Proxier Creation**

### **4.1 iptables Proxier**

```go
// pkg/proxy/iptables/proxier.go:300-500
func NewProxier(
    ipt utiliptables.Interface,
    sysctl utilsysctl.Interface,
    exec utilexec.Interface,
    syncPeriod time.Duration,
    minSyncPeriod time.Duration,
    // ... more parameters
) (*Proxier, error) {
    proxier := &Proxier{
        svcPortMap:          make(ServicePortMap),
        endpointsMap:        make(EndpointsMap),
        iptables:            ipt,
        masqueradeMark:      masqueradeMark,
        // ... initialize fields
    }
    
    burstSyncs := 2
    proxier.syncRunner = async.NewBoundedFrequencyRunner(
        "sync-runner",
        proxier.syncProxyRules,
        minSyncPeriod,
        syncPeriod,
        burstSyncs,
    )
    
    return proxier, nil
}
```

### **4.2 IPVS Proxier**

```go
// pkg/proxy/ipvs/proxier.go:400-600
func NewProxier(
    ipt utiliptables.Interface,
    ipvs utilipvs.Interface,
    ipset utilipset.Interface,
    sysctl utilsysctl.Interface,
    exec utilexec.Interface,
    syncPeriod time.Duration,
    minSyncPeriod time.Duration,
    // ... more parameters
) (*Proxier, error) {
    proxier := &Proxier{
        svcPortMap:          make(ServicePortMap),
        endpointsMap:        make(EndpointsMap),
        ipvs:                ipvs,
        ipset:               ipset,
        iptables:            ipt,
        ipvsScheduler:       scheduler,
        // ... initialize fields
    }
    
    proxier.syncRunner = async.NewBoundedFrequencyRunner(
        "sync-runner",
        proxier.syncProxyRules,
        minSyncPeriod,
        syncPeriod,
        burstSyncs,
    )
    
    proxier.gracefuldeleteManager = NewGracefulTerminationManager(ipvs)
    
    return proxier, nil
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **5. Informer Setup**

### **5.1 Service Informer**

```go
// pkg/proxy/config/config.go:80-150
func NewServiceConfig(serviceInformer coreinformers.ServiceInformer, resyncPeriod time.Duration) *ServiceConfig {
    c := &ServiceConfig{
        listerSynced: serviceInformer.Informer().HasSynced,
    }
    
    serviceInformer.Informer().AddEventHandlerWithResyncPeriod(
        cache.ResourceEventHandlerFuncs{
            AddFunc:    c.handleAddService,
            UpdateFunc: c.handleUpdateService,
            DeleteFunc: c.handleDeleteService,
        },
        resyncPeriod,
    )
    
    return c
}
```

### **5.2 EndpointSlice Informer**

```go
// pkg/proxy/config/config.go:200-270
func NewEndpointSliceConfig(endpointSliceInformer discoveryinformers.EndpointSliceInformer, resyncPeriod time.Duration) *EndpointSliceConfig {
    c := &EndpointSliceConfig{
        listerSynced: endpointSliceInformer.Informer().HasSynced,
    }
    
    endpointSliceInformer.Informer().AddEventHandlerWithResyncPeriod(
        cache.ResourceEventHandlerFuncs{
            AddFunc:    c.handleAddEndpointSlice,
            UpdateFunc: c.handleUpdateEndpointSlice,
            DeleteFunc: c.handleDeleteEndpointSlice,
        },
        resyncPeriod,
    )
    
    return c
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **6. Sync Loop**

### **6.1 SyncLoop Start**

```go
// pkg/proxy/iptables/proxier.go:700-750
func (proxier *Proxier) SyncLoop() {
    proxier.syncRunner.Loop(wait.NeverStop)
}
```

### **6.2 BoundedFrequencyRunner**

```go
// pkg/util/async/bounded_frequency_runner.go:100-200
func (bfr *BoundedFrequencyRunner) Loop(stop <-chan struct{}) {
    go func() {
        bfr.timer.Reset(bfr.maxInterval)
        for {
            select {
            case <-stop:
                return
            case <-bfr.run:
                bfr.tryRun()
            case <-bfr.timer.C:
                bfr.tryRun()
                bfr.timer.Reset(bfr.maxInterval)
            }
        }
    }()
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **7. Key File Locations**

### **7.1 Core Files**

| File | Purpose | Lines |
|------|---------|-------|
| `cmd/kube-proxy/proxy.go` | Main entry point | 50 |
| `cmd/kube-proxy/app/server.go` | Command and initialization | 2,000 |
| `cmd/kube-proxy/app/server_linux.go` | Linux-specific setup | 500 |
| `pkg/proxy/iptables/proxier.go` | iptables implementation | 2,500 |
| `pkg/proxy/ipvs/proxier.go` | IPVS implementation | 2,800 |
| `pkg/proxy/config/config.go` | Informer configuration | 500 |

### **7.2 Supporting Files**

| File | Purpose |
|------|---------|
| `pkg/proxy/types.go` | Core types and interfaces |
| `pkg/proxy/service.go` | ServiceChangeTracker |
| `pkg/proxy/endpoints.go` | EndpointsChangeTracker |
| `pkg/proxy/endpointslicecache.go` | EndpointSlice caching |
| `pkg/proxy/healthcheck/` | Health check server |
| `pkg/proxy/metrics/` | Prometheus metrics |
| `pkg/util/iptables/` | iptables interface |
| `pkg/util/ipvs/` | IPVS interface |
| `pkg/util/ipset/` | ipset interface |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **8. Quick Navigation**

### **8.1 By Feature**

**Service Discovery**:
- Service watching: `pkg/proxy/config/config.go:80-150`
- EndpointSlice watching: `pkg/proxy/config/config.go:200-270`
- Change tracking: `pkg/proxy/service.go`, `pkg/proxy/endpoints.go`

**Rule Generation**:
- iptables: `pkg/proxy/iptables/proxier.go:1000-2000`
- IPVS: `pkg/proxy/ipvs/proxier.go:1200-2000`

**Sync Loop**:
- Trigger mechanism: `pkg/util/async/bounded_frequency_runner.go`
- syncProxyRules (iptables): `pkg/proxy/iptables/proxier.go:900-1000`
- syncProxyRules (IPVS): `pkg/proxy/ipvs/proxier.go:900-1100`

**Health Checks**:
- Server: `pkg/proxy/healthcheck/healthcheck.go`
- NodePort: `pkg/proxy/healthcheck/proxier_health.go`

---

*Last Updated*: Session 13  
*Status*: ✅ Complete  
*Part of*: [kube-proxy Architecture Documentation](../00-README.md)
