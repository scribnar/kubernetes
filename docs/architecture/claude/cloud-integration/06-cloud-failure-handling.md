# **Cloud Failure Handling and Resilience**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Comprehensive guide to handling cloud provider failures in Kubernetes

**Target Audience**:
- SREs managing production Kubernetes clusters
- Platform engineers designing resilient systems
- Cloud provider developers implementing error handling
- Operations teams troubleshooting cloud issues

**Scope**: Failure modes, resilience patterns, error handling, recovery procedures

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Cloud Failure Categories**

### **Failure Taxonomy**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Cloud Failure Categories                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │   Transient     │  │   Partial       │  │   Complete      │     │
│  │   Failures      │  │   Failures      │  │   Failures      │     │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘     │
│           │                    │                    │               │
│  • API timeouts       • Zone outage        • Region outage          │
│  • Rate limiting      • Service degraded   • Provider outage        │
│  • Network blips      • Partial capacity   • Account suspension     │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │   Expected      │  │   Expected      │  │   Unexpected    │     │
│  │   Recovery:     │  │   Recovery:     │  │   Recovery:     │     │
│  │   Seconds       │  │   Minutes-Hours │  │   Hours-Days    │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### **Common Failure Scenarios**

| **Failure Type** | **Symptoms** | **Impact** | **Recovery** |
|------------------|--------------|------------|--------------|
| **API Rate Limiting** | 429 errors, throttling | Slow operations | Exponential backoff |
| **Network Timeout** | Context deadline exceeded | Failed API calls | Retry with timeout |
| **Zone Outage** | Instance unavailable | Pod disruption | Reschedule to other zones |
| **Service Degradation** | Slow responses, partial failures | Delayed operations | Wait and retry |
| **Credential Expiry** | 401/403 errors | All operations fail | Rotate credentials |
| **Instance Termination** | Node NotReady | Pod eviction | Node replacement |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 Built-in Resilience Mechanisms**

### **1. Retry with Exponential Backoff**

**File**: `staging/src/k8s.io/cloud-provider/controllers/service/controller.go`

```go
// Service controller uses rate-limited workqueue
workqueue.NewNamedRateLimitingQueue(
    workqueue.DefaultControllerRateLimiter(),
    "service-controller",
)

// DefaultControllerRateLimiter uses exponential backoff
// Initial: 5ms, Max: 1000s
// Retry sequence: 5ms, 10ms, 20ms, 40ms, ... up to 1000s
```

### **2. Context Timeouts**

```go
// Cloud provider methods accept context for timeout/cancellation
func (m *MyCloudProvider) EnsureLoadBalancer(
    ctx context.Context,  // Pass timeout via context
    clusterName string,
    service *v1.Service,
    nodes []*v1.Node,
) (*v1.LoadBalancerStatus, error) {
    // Create timeout for cloud API call
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    return m.client.CreateLoadBalancer(ctx, spec)
}
```

### **3. Node Taint for Uninitialized Nodes**

```yaml
# Kubelet registers with taint
spec:
  taints:
  - key: node.cloudprovider.kubernetes.io/uninitialized
    value: "true"
    effect: NoSchedule
```

This prevents scheduling until CCM initializes the node.

### **4. Graceful Node Shutdown**

**File**: `staging/src/k8s.io/cloud-provider/controllers/nodelifecycle/node_lifecycle_controller.go`

```go
func (c *CloudNodeLifecycleController) MonitorNodes(ctx context.Context) error {
    // Check instance shutdown status
    shutdown, err := instancesV2.InstanceShutdown(ctx, node)
    if err != nil {
        return err
    }

    if shutdown {
        // Add shutdown taint instead of immediate deletion
        c.addShutdownTaint(ctx, node)
    }
}
```

### **5. Finalizers for Cleanup**

```yaml
# LoadBalancer service has cleanup finalizer
metadata:
  finalizers:
  - service.kubernetes.io/load-balancer-cleanup
```

Ensures cloud resources are cleaned up before service deletion.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ Rate Limiting and Throttling**

### **Cloud Provider Rate Limits**

| **Provider** | **Service** | **Typical Limit** | **Burst** |
|--------------|-------------|-------------------|-----------|
| **AWS** | EC2 API | 100 req/sec | 1000 |
| **AWS** | ELB API | 20 req/sec | 100 |
| **GCE** | Compute API | 20 req/sec | 100 |
| **Azure** | ARM API | 12000 req/hour | - |

### **Handling Rate Limits**

**File**: `staging/src/k8s.io/cloud-provider/api/retry_error.go`

```go
// Cloud provider can return RetryError for rate limiting
func (m *MyCloudProvider) EnsureLoadBalancer(...) (*v1.LoadBalancerStatus, error) {
    err := m.client.CreateLoadBalancer(ctx, spec)
    if err != nil {
        if isRateLimitError(err) {
            // Return RetryError with fixed backoff
            return nil, cloudproviderapi.NewRetryError(
                "rate limited by cloud provider",
                30*time.Second,
            )
        }
        return nil, err
    }
    return status, nil
}
```

### **Client-Side Rate Limiting**

```go
import "golang.org/x/time/rate"

type RateLimitedClient struct {
    client  *CloudClient
    limiter *rate.Limiter
}

func NewRateLimitedClient(client *CloudClient) *RateLimitedClient {
    return &RateLimitedClient{
        client:  client,
        limiter: rate.NewLimiter(rate.Limit(10), 20), // 10/sec, burst 20
    }
}

func (r *RateLimitedClient) CreateLoadBalancer(ctx context.Context, spec *LBSpec) error {
    // Wait for rate limiter
    if err := r.limiter.Wait(ctx); err != nil {
        return err
    }
    return r.client.CreateLoadBalancer(ctx, spec)
}
```

### **Monitoring Rate Limits**

```promql
# Cloud API rate limit errors
rate(cloudprovider_api_errors_total{error_type="rate_limited"}[5m])

# API call rate
rate(cloudprovider_api_requests_total[5m])

# Retry queue depth
workqueue_depth{name="service-controller"}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌍 Zone and Region Failures**

### **Zone Failure Impact**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Multi-Zone Cluster                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Zone A (Healthy)    Zone B (Healthy)    Zone C (Failed)           │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │  Nodes: 3   │    │  Nodes: 3   │    │  Nodes: 3   │ ← Unreachable
│  │  Pods: 10   │    │  Pods: 10   │    │  Pods: 10   │             │
│  └─────────────┘    └─────────────┘    └─────────────┘             │
│                                                                     │
│  After 5 min: Nodes in Zone C marked NotReady                       │
│  After ~5 min: Pods evicted and rescheduled to Zone A/B             │
│                                                                     │
│  Zone A             Zone B             Zone C                       │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │  Nodes: 3   │    │  Nodes: 3   │    │  Nodes: 0   │             │
│  │  Pods: 15   │    │  Pods: 15   │    │  (deleted)  │             │
│  └─────────────┘    └─────────────┘    └─────────────┘             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### **Node Controller Behavior**

**File**: `staging/src/k8s.io/cloud-provider/controllers/nodelifecycle/node_lifecycle_controller.go`

```go
// Node lifecycle controller monitors instance status
func (c *CloudNodeLifecycleController) MonitorNodes(ctx context.Context) error {
    nodes, _ := c.nodeLister.List(labels.Everything())

    for _, node := range nodes {
        // Check if instance exists in cloud
        exists, err := instancesV2.InstanceExists(ctx, node)
        if err != nil {
            continue // Transient error, retry later
        }

        if !exists {
            // Instance terminated - delete node
            klog.Infof("Deleting node %s because instance no longer exists", node.Name)
            c.kubeClient.CoreV1().Nodes().Delete(ctx, node.Name, metav1.DeleteOptions{})
        }
    }
}
```

### **Pod Topology Spread**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-app
```

### **Regional Storage**

```yaml
# GCE Regional Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-pd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd  # Replicated across zones
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔐 Credential and Authentication Failures**

### **Common Authentication Issues**

| **Issue** | **Symptoms** | **Resolution** |
|-----------|--------------|----------------|
| **Expired credentials** | 401 Unauthorized | Rotate credentials |
| **Invalid permissions** | 403 Forbidden | Update IAM policy |
| **Wrong region** | 404 Not Found | Fix configuration |
| **Token refresh failure** | Intermittent 401 | Check token endpoint |

### **AWS IAM Roles for Service Accounts (IRSA)**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloud-controller-manager
  namespace: kube-system
  annotations:
    # Associate with IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/ccm-role
```

### **GKE Workload Identity**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloud-controller-manager
  namespace: kube-system
  annotations:
    # Bind to GCP service account
    iam.gke.io/gcp-service-account: ccm@project.iam.gserviceaccount.com
```

### **Azure Managed Identity**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloud-controller-manager
  namespace: kube-system
  annotations:
    # Use managed identity
    azure.workload.identity/client-id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### **Credential Rotation**

```bash
# AWS - Update secret
kubectl create secret generic cloud-credentials \
  -n kube-system \
  --from-file=credentials=~/.aws/credentials \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart CCM to pick up new credentials
kubectl rollout restart deployment/cloud-controller-manager -n kube-system
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚨 Failure Scenarios and Recovery**

### **Scenario 1: Cloud API Outage**

**Symptoms**:
```bash
# CCM logs showing errors
kubectl logs -n kube-system -l component=cloud-controller-manager

# Error syncing load balancer: Post "https://api.cloud.com": dial tcp: i/o timeout
# Error getting instance metadata: context deadline exceeded
```

**Impact**:
- New LoadBalancer services stuck in Pending
- New nodes not initialized (taint not removed)
- Node metadata not updated

**Recovery**:

1. **Verify cloud status**: Check cloud provider status page
2. **Monitor CCM queue depth**: Items will queue up
3. **Wait for recovery**: CCM will retry automatically
4. **Manual intervention** (if needed):
```bash
# Force service reconciliation
kubectl annotate svc <service> force-sync=$(date +%s)

# Force node reconciliation
kubectl annotate node <node> force-sync=$(date +%s)
```

### **Scenario 2: Instance Termination**

**Symptoms**:
```bash
kubectl get nodes
# NAME       STATUS     ROLES    AGE    VERSION
# node-1     NotReady   <none>   10d    v1.28.0

kubectl describe node node-1 | grep Conditions -A10
# Conditions:
#   Type             Status  LastHeartbeatTime
#   ----             ------  -----------------
#   Ready            False   5m ago
```

**Impact**:
- Pods on terminated node are evicted
- PVs may be stuck in attaching state

**Recovery**:

1. **CCM deletes node** (after detecting termination)
2. **Pods rescheduled** to other nodes
3. **Volumes detached** and reattached to new nodes

**Manual Recovery** (if CCM not working):
```bash
# Delete the node
kubectl delete node node-1

# Force detach volumes
kubectl delete volumeattachment csi-xxx

# Trigger reschedule
kubectl delete pod <stuck-pod>
```

### **Scenario 3: LoadBalancer Creation Failure**

**Symptoms**:
```bash
kubectl describe svc my-service
# Events:
#   Warning  SyncLoadBalancerFailed  1m  service-controller
#     Error syncing load balancer: failed to ensure load balancer:
#     LimitExceeded: You have exceeded your load balancer limit
```

**Impact**:
- Service stuck in Pending state
- No external access to application

**Recovery**:

1. **Check cloud quotas**:
```bash
# AWS
aws service-quotas list-service-quotas --service-code elasticloadbalancing

# Request increase
aws service-quotas request-service-quota-increase ...
```

2. **Clean up unused LBs**:
```bash
# Find orphaned load balancers
kubectl get svc --all-namespaces -o json | jq '.items[] | select(.spec.type=="LoadBalancer") | .metadata.name'

# Compare with cloud provider list
aws elbv2 describe-load-balancers
```

3. **Consider alternatives**:
- Use Ingress controller (shared LB)
- Use NodePort services
- Consolidate services

### **Scenario 4: Volume Attachment Failure**

**Symptoms**:
```bash
kubectl describe pod my-pod
# Warning  FailedAttachVolume  2m  attachdetach-controller
#   AttachVolume.Attach failed for volume "pvc-xxx":
#   Could not attach volume "vol-xxx" to node "i-yyy":
#   VolumeInUse: vol-xxx is already attached to an instance
```

**Impact**:
- Pod stuck in ContainerCreating
- Data unavailable

**Recovery**:

1. **Find where volume is attached**:
```bash
# AWS
aws ec2 describe-volumes --volume-ids vol-xxx

# Check which node
kubectl get volumeattachment
```

2. **Force detach** (if previous node is gone):
```bash
# Delete the VolumeAttachment
kubectl delete volumeattachment csi-xxx

# Force detach in cloud (last resort)
aws ec2 detach-volume --volume-id vol-xxx --force
```

3. **Delete stuck pod**:
```bash
kubectl delete pod my-pod --force --grace-period=0
```

### **Scenario 5: CCM Leader Election Failure**

**Symptoms**:
```bash
kubectl logs -n kube-system -l component=cloud-controller-manager
# failed to renew leader lease kube-system/cloud-controller-manager: timed out waiting for leader lease
```

**Impact**:
- CCM not processing any resources
- Cloud resources not being created/updated

**Recovery**:

1. **Check CCM pods**:
```bash
kubectl get pods -n kube-system -l component=cloud-controller-manager
```

2. **Check API server connectivity**:
```bash
kubectl exec -n kube-system <ccm-pod> -- wget -O- https://kubernetes.default.svc/healthz
```

3. **Delete lease to force re-election**:
```bash
kubectl delete lease -n kube-system cloud-controller-manager
```

4. **Restart CCM**:
```bash
kubectl rollout restart deployment/cloud-controller-manager -n kube-system
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Monitoring and Alerting**

### **Key Metrics**

```promql
# CCM leader election status
leader_election_master_status{name="cloud-controller-manager"}

# Cloud API errors
rate(cloudprovider_api_requests_total{code=~"4..|5.."}[5m])

# Service sync failures
rate(service_controller_sync_errors_total[5m])

# Node not ready count
sum(kube_node_status_condition{condition="Ready",status="false"})

# Uninitialized nodes
sum(kube_node_spec_taint{key="node.cloudprovider.kubernetes.io/uninitialized"})
```

### **Alerts**

```yaml
groups:
- name: cloud-provider-alerts
  rules:
  # CCM not running
  - alert: CloudControllerManagerDown
    expr: |
      absent(up{job="cloud-controller-manager"} == 1)
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Cloud Controller Manager is down"
      runbook_url: "https://runbooks.example.com/ccm-down"

  # CCM not leader
  - alert: CloudControllerManagerNotLeader
    expr: |
      leader_election_master_status{name="cloud-controller-manager"} == 0
    for: 5m
    labels:
      severity: critical

  # High cloud API error rate
  - alert: CloudAPIHighErrorRate
    expr: |
      rate(cloudprovider_api_requests_total{code=~"5.."}[5m]) /
      rate(cloudprovider_api_requests_total[5m]) > 0.1
    for: 10m
    labels:
      severity: warning

  # Rate limiting
  - alert: CloudAPIRateLimited
    expr: |
      rate(cloudprovider_api_requests_total{code="429"}[5m]) > 0
    for: 5m
    labels:
      severity: warning

  # Nodes stuck uninitialized
  - alert: NodesStuckUninitialized
    expr: |
      kube_node_spec_taint{key="node.cloudprovider.kubernetes.io/uninitialized"} == 1
    for: 15m
    labels:
      severity: warning

  # Services stuck pending
  - alert: LoadBalancerStuckPending
    expr: |
      kube_service_status_load_balancer_ingress == 0
      and on(namespace, service)
      kube_service_spec_type{type="LoadBalancer"} == 1
    for: 15m
    labels:
      severity: warning
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💡 Best Practices**

### **Design for Failure**

1. **Multi-zone deployments**: Spread across availability zones
2. **Pod disruption budgets**: Limit simultaneous disruptions
3. **Topology constraints**: Ensure even distribution
4. **Regional storage**: Use replicated storage for critical data

### **Configure Timeouts**

```yaml
# CCM configuration
args:
- --node-monitor-period=5s
- --node-status-update-frequency=5m
- --route-reconciliation-period=10s
```

### **Implement Circuit Breakers**

```go
type CircuitBreaker struct {
    failures    int
    lastFailure time.Time
    threshold   int
    timeout     time.Duration
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    if cb.IsOpen() {
        return errors.New("circuit breaker open")
    }

    err := fn()
    if err != nil {
        cb.RecordFailure()
        return err
    }

    cb.Reset()
    return nil
}
```

### **Monitor Cloud Health**

1. **Subscribe to cloud status**: AWS Health, GCP Status
2. **Monitor API latency**: Track cloud API response times
3. **Alert on degradation**: Don't wait for complete failure

### **Test Failure Scenarios**

```bash
# Chaos engineering
# Test zone failure
kubectl drain --selector topology.kubernetes.io/zone=us-east-1a

# Test CCM failure
kubectl scale deployment/cloud-controller-manager --replicas=0 -n kube-system

# Test node failure
aws ec2 terminate-instances --instance-ids i-xxx
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 References**

### **Source Code**
- **Node Lifecycle Controller**: `staging/src/k8s.io/cloud-provider/controllers/nodelifecycle/`
- **Service Controller**: `staging/src/k8s.io/cloud-provider/controllers/service/`
- **RetryError**: `staging/src/k8s.io/cloud-provider/api/retry_error.go`

### **Related Documentation**
- **Cloud Controller Manager**: `docs/architecture/claude/cloud-integration/01-cloud-controller-manager.md`
- **LoadBalancer Integration**: `docs/architecture/claude/cloud-integration/03-loadbalancer-integration.md`
- **Storage Integration**: `docs/architecture/claude/cloud-integration/04-storage-integration.md`

### **External Resources**
- **AWS Status**: https://status.aws.amazon.com/
- **GCP Status**: https://status.cloud.google.com/
- **Azure Status**: https://status.azure.com/
- **Kubernetes Disruption Budgets**: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: Complete
**Last Updated**: 2024-11-19
**Target Audience**: SREs, platform engineers, operations teams
**Scope**: Failure modes, resilience patterns, recovery procedures, monitoring
