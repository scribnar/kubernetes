# **LoadBalancer Service Integration**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Deep dive into how Kubernetes LoadBalancer services integrate with cloud providers

**Target Audience**:
- Platform engineers deploying LoadBalancer services
- Cloud provider developers implementing LoadBalancer support
- SREs troubleshooting load balancer issues
- Network engineers understanding cloud networking

**Scope**: Service controller operation, LoadBalancer lifecycle, annotations, health checks, and production patterns

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 LoadBalancer Service Architecture**

### **How LoadBalancer Services Work**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      LoadBalancer Service Flow                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  User → kubectl create service           1. Create Service Type=LB      │
│                    │                                                    │
│                    ▼                                                    │
│  ┌─────────────────────────┐                                           │
│  │    API Server           │             2. Service object created      │
│  └───────────┬─────────────┘                                           │
│              │                                                          │
│              │ Watch                                                    │
│              ▼                                                          │
│  ┌─────────────────────────┐                                           │
│  │  Service Controller     │             3. Detect new LB service       │
│  │  (Cloud Controller Mgr) │                                           │
│  └───────────┬─────────────┘                                           │
│              │                                                          │
│              │ EnsureLoadBalancer()                                     │
│              ▼                                                          │
│  ┌─────────────────────────┐                                           │
│  │   Cloud Provider API    │             4. Create cloud load balancer  │
│  │   (AWS/GCE/Azure/...)   │                                           │
│  └───────────┬─────────────┘                                           │
│              │                                                          │
│              │ Return IP/Hostname                                       │
│              ▼                                                          │
│  ┌─────────────────────────┐                                           │
│  │    Update Service       │             5. Update service.status       │
│  │    Status.LoadBalancer  │                with external IP           │
│  └─────────────────────────┘                                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### **Service Controller Overview**

**File**: `staging/src/k8s.io/cloud-provider/controllers/service/controller.go`

```go
type Controller struct {
    cloud         cloudprovider.Interface
    kubeClient    clientset.Interface
    clusterName   string

    serviceLister corelisters.ServiceLister
    nodeLister    corelisters.NodeLister

    workqueue     workqueue.RateLimitingInterface
    recorder      record.EventRecorder
}

func (s *Controller) Run(ctx context.Context, workers int) {
    defer s.workqueue.ShutDown()

    // Wait for cache sync
    if !cache.WaitForNamedCacheSync("service", ctx.Done(),
        s.serviceListerSynced, s.nodeListerSynced) {
        return
    }

    // Start workers
    for i := 0; i < workers; i++ {
        go wait.UntilWithContext(ctx, s.worker, time.Second)
    }

    <-ctx.Done()
}

func (s *Controller) worker(ctx context.Context) {
    for s.processNextItem(ctx) {
    }
}

func (s *Controller) processNextItem(ctx context.Context) bool {
    key, quit := s.workqueue.Get()
    if quit {
        return false
    }
    defer s.workqueue.Done(key)

    err := s.syncLoadBalancerIfNeeded(ctx, key.(string))
    if err == nil {
        s.workqueue.Forget(key)
        return true
    }

    s.workqueue.AddRateLimited(key)
    return true
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 LoadBalancer Lifecycle**

### **Creation Flow**

```go
func (s *Controller) syncLoadBalancerIfNeeded(ctx context.Context, key string) error {
    namespace, name, _ := cache.SplitMetaNamespaceKey(key)

    // Get service
    service, err := s.serviceLister.Services(namespace).Get(name)
    if err != nil {
        if errors.IsNotFound(err) {
            return nil  // Service deleted
        }
        return err
    }

    // Only handle LoadBalancer services
    if service.Spec.Type != v1.ServiceTypeLoadBalancer {
        return nil
    }

    // Check finalizer for cleanup
    if service.DeletionTimestamp != nil {
        return s.processServiceDeletion(ctx, service)
    }

    // Get cloud LoadBalancer interface
    lb, supported := s.cloud.LoadBalancer()
    if !supported {
        return errors.New("cloud provider does not support load balancers")
    }

    // Get nodes for backend pool
    nodes, err := s.nodeLister.ListWithPredicate(func(node *v1.Node) bool {
        return nodeReadyForLoadBalancer(node)
    })
    if err != nil {
        return err
    }

    // Ensure load balancer exists
    status, err := lb.EnsureLoadBalancer(ctx, s.clusterName, service, nodes)
    if err != nil {
        s.recorder.Eventf(service, v1.EventTypeWarning, "SyncLoadBalancerFailed",
            "Error syncing load balancer: %v", err)
        return err
    }

    // Update service status
    return s.updateServiceStatus(ctx, service, status)
}
```

### **Node Selection for Backends**

```go
func nodeReadyForLoadBalancer(node *v1.Node) bool {
    // Check if node is ready
    for _, condition := range node.Status.Conditions {
        if condition.Type == v1.NodeReady {
            if condition.Status != v1.ConditionTrue {
                return false
            }
            break
        }
    }

    // Check for taints that exclude from load balancer
    for _, taint := range node.Spec.Taints {
        // Nodes with NoSchedule taint for ToBeDeletedByClusterAutoscaler
        if taint.Key == v1.TaintNodeUnschedulable {
            return false
        }
        // Nodes being deleted
        if taint.Key == "ToBeDeletedByClusterAutoscaler" {
            return false
        }
    }

    return true
}
```

### **Status Update**

```go
func (s *Controller) updateServiceStatus(ctx context.Context, service *v1.Service,
    status *v1.LoadBalancerStatus) error {

    // Check if update needed
    if loadBalancerStatusEqual(&service.Status.LoadBalancer, status) {
        return nil
    }

    // Create service copy
    serviceCopy := service.DeepCopy()
    serviceCopy.Status.LoadBalancer = *status

    // Update status
    _, err := s.kubeClient.CoreV1().Services(service.Namespace).UpdateStatus(
        ctx, serviceCopy, metav1.UpdateOptions{})
    if err != nil {
        return err
    }

    s.recorder.Eventf(service, v1.EventTypeNormal, "EnsuredLoadBalancer",
        "Ensured load balancer")

    return nil
}
```

### **Deletion Flow**

```go
func (s *Controller) processServiceDeletion(ctx context.Context,
    service *v1.Service) error {

    // Check if finalizer present
    if !hasLBFinalizer(service) {
        return nil
    }

    // Get cloud LoadBalancer interface
    lb, supported := s.cloud.LoadBalancer()
    if !supported {
        return s.removeFinalizer(ctx, service)
    }

    // Delete load balancer
    err := lb.EnsureLoadBalancerDeleted(ctx, s.clusterName, service)
    if err != nil {
        s.recorder.Eventf(service, v1.EventTypeWarning, "DeleteLoadBalancerFailed",
            "Error deleting load balancer: %v", err)
        return err
    }

    s.recorder.Eventf(service, v1.EventTypeNormal, "DeletedLoadBalancer",
        "Deleted load balancer")

    // Remove finalizer
    return s.removeFinalizer(ctx, service)
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚙️ Service Configuration**

### **Basic LoadBalancer Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
```

### **LoadBalancer Source Ranges**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: restricted-service
spec:
  type: LoadBalancer
  # Restrict access to specific CIDR ranges
  loadBalancerSourceRanges:
    - 10.0.0.0/8
    - 192.168.0.0/16
    - 203.0.113.0/24
  ports:
  - port: 80
```

### **Internal LoadBalancer**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-service
  annotations:
    # AWS
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    # GCE
    networking.gke.io/load-balancer-type: "Internal"
    # Azure
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  ports:
  - port: 80
```

### **External Traffic Policy**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: local-traffic-service
spec:
  type: LoadBalancer

  # Cluster (default): Distribute to any pod
  # Local: Preserve client IP, only local pods
  externalTrafficPolicy: Local

  # Health check port for Local traffic policy
  healthCheckNodePort: 30000

  ports:
  - port: 80
```

### **Load Balancer IP Mode**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: proxy-mode-service
spec:
  type: LoadBalancer

  # Allocate specific IP (cloud provider dependent)
  loadBalancerIP: "192.0.2.100"

  ports:
  - port: 80
status:
  loadBalancer:
    ingress:
    - ip: "192.0.2.100"
      # VIP = direct routing to nodes
      # Proxy = traffic goes through proxy/gateway
      ipMode: VIP
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **☁️ Cloud Provider Annotations**

### **AWS ELB/NLB Annotations**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aws-nlb-service
  annotations:
    # Load balancer type
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # nlb, nlb-ip, external

    # Internal vs external
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"

    # Subnet selection
    service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-abc,subnet-def"

    # Security groups
    service.beta.kubernetes.io/aws-load-balancer-extra-security-groups: "sg-12345"

    # Cross-zone load balancing
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

    # SSL/TLS
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:..."
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"

    # Proxy protocol (NLB)
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"

    # Health check
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "TCP"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "traffic-port"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/healthz"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"

    # Target type (NLB)
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"  # instance, ip

    # Access logs
    service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "my-bucket"

spec:
  type: LoadBalancer
  ports:
  - port: 443
```

### **GCE Annotations**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gce-service
  annotations:
    # Internal load balancer
    networking.gke.io/load-balancer-type: "Internal"

    # Backend service configuration
    cloud.google.com/backend-config: '{"default": "my-backend-config"}'

    # Network tier
    cloud.google.com/network-tier: "Premium"  # Standard

    # Allow global access (internal LB)
    networking.gke.io/internal-load-balancer-allow-global-access: "true"

    # Subnet selection (internal LB)
    networking.gke.io/internal-load-balancer-subnet: "my-subnet"

spec:
  type: LoadBalancer
  ports:
  - port: 80
```

### **Azure Annotations**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: azure-service
  annotations:
    # Internal load balancer
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"

    # Subnet for internal LB
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "my-subnet"

    # Health probe
    service.beta.kubernetes.io/azure-load-balancer-health-probe-protocol: "Http"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/healthz"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-interval: "5"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-num-of-probe: "2"

    # Disable floating IP
    service.beta.kubernetes.io/azure-disable-load-balancer-floating-ip: "true"

    # Resource group
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "my-rg"

    # DNS label
    service.beta.kubernetes.io/azure-dns-label-name: "my-service"

    # Idle timeout
    service.beta.kubernetes.io/azure-load-balancer-tcp-idle-timeout: "4"

spec:
  type: LoadBalancer
  ports:
  - port: 80
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏥 Health Checks**

### **Local Traffic Policy Health Checks**

When using `externalTrafficPolicy: Local`, the cloud provider needs health checks to determine which nodes have pods:

**File**: `staging/src/k8s.io/cloud-provider/service/helpers/helper.go`

```go
// NeedsHealthCheck checks if service needs health check
func NeedsHealthCheck(service *v1.Service) bool {
    if service.Spec.Type != v1.ServiceTypeLoadBalancer {
        return false
    }
    return RequestsOnlyLocalTraffic(service)
}

// GetServiceHealthCheckPathPort returns health check configuration
func GetServiceHealthCheckPathPort(service *v1.Service) (string, int32) {
    if !NeedsHealthCheck(service) {
        return "", 0
    }
    return "/healthz", service.Spec.HealthCheckNodePort
}
```

### **Health Check Endpoint**

kube-proxy exposes a health check endpoint on each node:

```
http://<node-ip>:<healthCheckNodePort>/healthz

Response:
- 200 OK: Node has at least one pod for this service
- 503 Service Unavailable: No pods on this node
```

### **Cloud Provider Health Check Implementation**

```go
func (m *MyCloudProvider) EnsureLoadBalancer(ctx context.Context,
    clusterName string, service *v1.Service, nodes []*v1.Node) (*v1.LoadBalancerStatus, error) {

    // Check if health check is needed
    healthCheckPath, healthCheckPort := helper.GetServiceHealthCheckPathPort(service)

    var healthCheck *HealthCheckConfig
    if healthCheckPort != 0 {
        healthCheck = &HealthCheckConfig{
            Protocol: "HTTP",
            Port:     healthCheckPort,
            Path:     healthCheckPath,
            Interval: 10 * time.Second,
            Timeout:  5 * time.Second,
            Healthy:  2,
            Unhealthy: 2,
        }
    } else {
        // Default TCP health check
        healthCheck = &HealthCheckConfig{
            Protocol: "TCP",
            Port:     service.Spec.Ports[0].NodePort,
        }
    }

    // Create/update load balancer with health check
    return m.createLoadBalancer(ctx, service, nodes, healthCheck)
}
```

### **Traffic Flow Comparison**

**Cluster Traffic Policy (default)**:

```
Client → LB → Any Node → kube-proxy → Any Pod

Pros:
- Even distribution across all nodes
- Works even if node has no pods

Cons:
- Extra network hop (SNAT)
- Client IP not preserved
```

**Local Traffic Policy**:

```
Client → LB → Node with Pod → Pod

Pros:
- Client IP preserved
- Lower latency (no extra hop)

Cons:
- Uneven distribution if pods unbalanced
- Node without pods gets no traffic
- Requires health checks
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Node Updates**

### **Triggering Backend Updates**

The service controller watches nodes and updates load balancer backends:

```go
func NewController(...) (*Controller, error) {
    // ...

    // Watch services
    serviceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            s.enqueueService(obj)
        },
        UpdateFunc: func(old, cur interface{}) {
            s.enqueueService(cur)
        },
        DeleteFunc: func(obj interface{}) {
            s.enqueueService(obj)
        },
    })

    // Watch nodes for backend updates
    nodeInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            s.nodeSyncLoop()
        },
        UpdateFunc: func(old, cur interface{}) {
            oldNode := old.(*v1.Node)
            curNode := cur.(*v1.Node)
            // Only trigger if readiness changed
            if nodeReady(oldNode) != nodeReady(curNode) {
                s.nodeSyncLoop()
            }
        },
        DeleteFunc: func(obj interface{}) {
            s.nodeSyncLoop()
        },
    })

    return s, nil
}

func (s *Controller) nodeSyncLoop() {
    // Enqueue all LoadBalancer services for sync
    services, _ := s.serviceLister.List(labels.Everything())
    for _, service := range services {
        if service.Spec.Type == v1.ServiceTypeLoadBalancer {
            s.enqueueService(service)
        }
    }
}
```

### **UpdateLoadBalancer Implementation**

```go
func (m *MyCloudProvider) UpdateLoadBalancer(ctx context.Context,
    clusterName string, service *v1.Service, nodes []*v1.Node) error {

    name := m.GetLoadBalancerName(ctx, clusterName, service)

    // Build backend list from ready nodes
    var backends []Backend
    for _, node := range nodes {
        // Get node address
        var address string
        for _, addr := range node.Status.Addresses {
            if addr.Type == v1.NodeInternalIP {
                address = addr.Address
                break
            }
        }
        if address == "" {
            continue
        }

        // Add backend for each port
        for _, port := range service.Spec.Ports {
            backends = append(backends, Backend{
                Address: address,
                Port:    port.NodePort,
            })
        }
    }

    // Update load balancer backends
    return m.client.UpdateBackends(ctx, name, backends)
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚨 Production Troubleshooting**

### **Scenario 1: Service Stuck in Pending**

**Symptoms**:
```bash
kubectl get svc my-service
# NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# my-service   LoadBalancer   10.96.10.100   <pending>     80:30080/TCP   30m
```

**Investigation**:

```bash
# 1. Check service events
kubectl describe svc my-service

# Events:
#   Type     Reason                  Age   Message
#   ----     ------                  ----  -------
#   Warning  SyncLoadBalancerFailed  1m    Error syncing load balancer: ...

# 2. Check CCM logs
kubectl logs -n kube-system -l component=cloud-controller-manager | grep "my-service"

# 3. Check cloud provider quotas
# AWS: Service quotas for ELB/NLB
# GCE: Forwarding rules, backend services
# Azure: Load balancers per subscription

# 4. Check IAM permissions
# Ensure CCM service account has required permissions

# 5. Check network configuration
# Subnet tags, security groups, route tables
```

**Common Causes**:

1. **IAM Permission Issues**:
```json
{
  "Effect": "Allow",
  "Action": [
    "elasticloadbalancing:*",
    "ec2:DescribeSubnets",
    "ec2:DescribeSecurityGroups"
  ],
  "Resource": "*"
}
```

2. **Subnet Configuration**:
```bash
# AWS: Ensure subnets are tagged
aws ec2 create-tags --resources subnet-xxx \
  --tags Key=kubernetes.io/cluster/my-cluster,Value=shared \
         Key=kubernetes.io/role/elb,Value=1
```

3. **Quota Exceeded**:
```bash
# Check limits
aws service-quotas list-service-quotas --service-code elasticloadbalancing
```

### **Scenario 2: Load Balancer Not Updating**

**Symptoms**:
- New nodes not receiving traffic
- Deleted pods still receiving traffic
- Backend changes not reflected

**Investigation**:

```bash
# 1. Check service controller sync
kubectl logs -n kube-system -l component=cloud-controller-manager | grep "UpdateLoadBalancer"

# 2. Check node selection
kubectl get nodes --show-labels | grep Ready

# 3. Check cloud load balancer directly
# AWS
aws elbv2 describe-target-health --target-group-arn <arn>

# GCE
gcloud compute backend-services get-health <backend-service>

# Azure
az network lb show -g <rg> -n <lb-name> --query backendAddressPools
```

**Fixes**:

1. **Force Service Sync**:
```bash
# Touch service to trigger reconciliation
kubectl patch svc my-service -p '{"metadata":{"annotations":{"timestamp":"'$(date +%s)'"}}}'
```

2. **Check Node Taints**:
```bash
# Nodes with these taints are excluded
kubectl get nodes -o json | jq '.items[] | select(.spec.taints) | {name: .metadata.name, taints: .spec.taints}'
```

### **Scenario 3: Health Check Failures**

**Symptoms**:
- All backends showing unhealthy
- Intermittent 503 errors
- Traffic not reaching pods

**Investigation**:

```bash
# 1. Check health check configuration
kubectl get svc my-service -o yaml | grep -A5 healthCheck

# 2. Test health check endpoint locally
kubectl get nodes -o wide
curl http://<node-ip>:<healthCheckNodePort>/healthz

# 3. Check kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy

# 4. Check if pods exist on nodes
kubectl get pods -o wide -l app=my-app
```

**Common Causes**:

1. **Security Group Blocking Health Checks**:
```bash
# Allow health check traffic from cloud provider ranges
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 30000-32767 \
  --cidr 0.0.0.0/0
```

2. **Wrong Health Check Port**:
```yaml
spec:
  externalTrafficPolicy: Local
  # Kubernetes allocates this port automatically
  # Don't set manually unless needed
  healthCheckNodePort: 30000
```

3. **Pods Not Running on Any Node**:
```bash
kubectl get pods -l app=my-app -o wide
# Ensure at least one pod is Running
```

### **Scenario 4: Traffic Distribution Issues**

**Symptoms**:
- Uneven load across pods
- Some pods getting no traffic
- High latency for some requests

**Investigation**:

```bash
# 1. Check external traffic policy
kubectl get svc my-service -o jsonpath='{.spec.externalTrafficPolicy}'

# 2. Check pod distribution
kubectl get pods -o wide -l app=my-app

# 3. Check node health in load balancer
# (cloud provider specific)

# 4. Check connection draining
kubectl get svc my-service -o yaml | grep -i drain
```

**Optimization**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: balanced-service
  annotations:
    # AWS NLB cross-zone
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  # Use Cluster for even distribution
  externalTrafficPolicy: Cluster
  ports:
  - port: 80
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Monitoring**

### **Key Metrics**

```promql
# Service sync duration
histogram_quantile(0.99,
  rate(service_controller_sync_duration_seconds_bucket[5m]))

# Service sync errors
rate(service_controller_sync_errors_total[5m])

# Load balancer operations
rate(cloudprovider_loadbalancer_sync_total{status="success"}[5m])
rate(cloudprovider_loadbalancer_sync_total{status="error"}[5m])

# Backend health (cloud provider specific)
# AWS CloudWatch: UnHealthyHostCount
# GCE: backend_request_count, backend_response_errors
```

### **Alerts**

```yaml
groups:
- name: loadbalancer-alerts
  rules:
  - alert: LoadBalancerSyncFailing
    expr: rate(service_controller_sync_errors_total[5m]) > 0
    for: 5m
    annotations:
      summary: "LoadBalancer service sync is failing"

  - alert: LoadBalancerStuckPending
    expr: |
      kube_service_status_load_balancer_ingress == 0
      and on(namespace, service) kube_service_spec_type == "LoadBalancer"
    for: 15m
    annotations:
      summary: "LoadBalancer service stuck in pending state"

  - alert: LoadBalancerNoHealthyBackends
    expr: aws_elb_healthy_host_count == 0
    for: 5m
    annotations:
      summary: "LoadBalancer has no healthy backends"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💡 Best Practices**

### **Service Configuration**

1. **Use Appropriate Traffic Policy**:
```yaml
# Client IP preservation needed
externalTrafficPolicy: Local

# Even distribution needed
externalTrafficPolicy: Cluster
```

2. **Configure Source Ranges**:
```yaml
# Restrict access for internal services
loadBalancerSourceRanges:
  - 10.0.0.0/8
```

3. **Set Meaningful Annotations**:
```yaml
annotations:
  # Document purpose
  description: "Public API endpoint"
  owner: "platform-team"
```

### **Cloud Provider Configuration**

1. **Use Internal LBs Where Possible**:
```yaml
# Reduces attack surface
service.beta.kubernetes.io/aws-load-balancer-internal: "true"
```

2. **Enable Cross-Zone Load Balancing**:
```yaml
# Better distribution
service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
```

3. **Configure Proper Health Checks**:
```yaml
# Match application health check
service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/healthz"
service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
```

### **Operational Best Practices**

1. **Monitor Load Balancer Health**:
   - Set up alerts for unhealthy backends
   - Track sync failures
   - Monitor latency metrics

2. **Plan for Capacity**:
   - Understand cloud provider limits
   - Request quota increases proactively
   - Use multiple services if needed

3. **Test Failover**:
   - Simulate node failures
   - Test pod scaling
   - Verify health check behavior

4. **Document Configurations**:
   - Record annotation meanings
   - Document network topology
   - Keep runbooks updated

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 References**

### **Source Code**
- **Service Controller**: `staging/src/k8s.io/cloud-provider/controllers/service/controller.go`
- **Service Helpers**: `staging/src/k8s.io/cloud-provider/service/helpers/helper.go`
- **LoadBalancer Interface**: `staging/src/k8s.io/cloud-provider/cloud.go`

### **Related Documentation**
- **Cloud Controller Manager**: `docs/architecture/claude/cloud-integration/01-cloud-controller-manager.md`
- **Cloud Provider Interface**: `docs/architecture/claude/cloud-integration/02-cloud-provider-interface.md`

### **External Resources**
- **AWS Load Balancer Controller**: https://kubernetes-sigs.github.io/aws-load-balancer-controller/
- **GKE Ingress for Load Balancing**: https://cloud.google.com/kubernetes-engine/docs/concepts/ingress
- **Azure Load Balancer**: https://docs.microsoft.com/en-us/azure/aks/load-balancer-standard

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: Complete
**Last Updated**: 2024-11-17
**Target Audience**: Platform engineers, cloud provider developers, SREs
**Scope**: LoadBalancer service lifecycle, configuration, health checks, troubleshooting
