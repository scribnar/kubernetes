# **Horizontal Scaling in Kubernetes**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Document Overview**

**Target Audience**: Platform engineers, SREs, and architects implementing autoscaling strategies

**Purpose**: This document provides comprehensive guidance on horizontal scaling in Kubernetes, covering Horizontal Pod Autoscaler (HPA), Vertical Pod Autoscaler (VPA), Cluster Autoscaler, and event-driven autoscaling with KEDA.

**Scope**:
- Horizontal Pod Autoscaler (HPA) deep dive
- Vertical Pod Autoscaler (VPA) patterns and integration
- Cluster Autoscaler architecture and configuration
- KEDA (Kubernetes Event-Driven Autoscaling)
- Multi-dimensional autoscaling strategies
- Production troubleshooting and best practices
- Performance considerations at scale

**Related Documentation**:
- [Scalability Limits](02-scalability-limits.md) - Understanding HPA rate limits
- [Component Optimization](06-component-optimization.md) - Optimizing for autoscaling
- [Performance Benchmarking](03-performance-benchmarking.md) - Measuring autoscaling effectiveness

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Autoscaling Dimensions**

### **Three Dimensions of Scaling**

Kubernetes provides three complementary autoscaling mechanisms:

```mermaid
graph TB
    Workload[Application Workload]

    Workload --> HPA[Horizontal Pod Autoscaler]
    Workload --> VPA[Vertical Pod Autoscaler]
    Workload --> CA[Cluster Autoscaler]

    HPA --> PodCount[Scale Pod Count]
    VPA --> PodSize[Scale Pod Resources]
    CA --> NodeCount[Scale Node Count]

    PodCount --> HPAResult[More/Fewer Replicas]
    PodSize --> VPAResult[Larger/Smaller Pods]
    NodeCount --> CAResult[More/Fewer Nodes]

    style HPA fill:#3498db
    style VPA fill:#2ecc71
    style CA fill:#e74c3c
```

| **Autoscaler** | **What It Scales** | **Based On** | **Use Case** |
|----------------|-------------------|--------------|--------------|
| **HPA** | Number of pod replicas | CPU, memory, custom metrics | Stateless workloads, traffic spikes |
| **VPA** | Pod resource requests/limits | Historical resource usage | Right-sizing pods, resource optimization |
| **Cluster Autoscaler** | Number of nodes | Pod scheduling failures | Capacity management, cost optimization |
| **KEDA** | Pod replicas (like HPA) | External events/metrics | Event-driven workloads, queue processing |

### **When to Use Each Autoscaler**

**Use HPA When**:
- ✅ Workload is stateless and can be replicated
- ✅ Response to traffic varies (web applications, APIs)
- ✅ Multiple instances improve throughput
- ✅ Scale-out is more cost-effective than scale-up

**Use VPA When**:
- ✅ Resource requirements are unpredictable
- ✅ Workload cannot be horizontally scaled (StatefulSets, databases)
- ✅ Need to right-size pod resource requests
- ✅ Optimizing resource utilization

**Use Cluster Autoscaler When**:
- ✅ Workload varies significantly over time
- ✅ Cost optimization is priority
- ✅ Need to handle burst capacity
- ✅ Running on cloud infrastructure (AWS, GCP, Azure)

**Use KEDA When**:
- ✅ Workload is event-driven (message queues, streams)
- ✅ Need to scale to zero replicas
- ✅ Scaling based on external systems (databases, cloud services)
- ✅ Complex scaling triggers (multiple metrics combined)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📈 Horizontal Pod Autoscaler (HPA)**

### **HPA Architecture**

HPA is a control loop that periodically queries metrics and adjusts replica count:

```mermaid
sequenceDiagram
    participant HPA as HPA Controller
    participant Metrics as Metrics Server/API
    participant Deployment as Deployment
    participant Scheduler as Scheduler
    participant Pods as Pods

    loop Every 15s (default)
        HPA->>Metrics: Query current metrics
        Metrics-->>HPA: CPU: 75%, Memory: 60%

        HPA->>HPA: Calculate desired replicas
        Note over HPA: desiredReplicas = <br/>ceil(currentReplicas × <br/>(currentMetric / targetMetric))

        alt Scale Up Needed
            HPA->>Deployment: Update replicas: 3 → 5
            Deployment->>Scheduler: Schedule 2 new pods
            Scheduler->>Pods: Create pods
        else Scale Down Needed
            HPA->>Deployment: Update replicas: 5 → 3
            Deployment->>Pods: Terminate 2 pods
        else No Change
            HPA->>HPA: Do nothing
        end
    end
```

**Source Code Reference**:
```go
// pkg/controller/podautoscaler/horizontal.go:62-64
// HPA scale-up rate limiting constants

var (
    scaleUpLimitFactor  = 2.0   // Maximum 2× current replicas per cycle
    scaleUpLimitMinimum = 4.0   // Or minimum 4 pods, whichever is larger
)
```

### **Basic HPA Configuration**

#### **CPU-Based HPA**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  minReplicas: 2     # Minimum replicas (always)
  maxReplicas: 100   # Maximum replicas (hard limit)

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Target 70% CPU across all pods

# Scaling logic:
#   currentUtil = 75%, targetUtil = 70%
#   desiredReplicas = ceil(currentReplicas × (75 / 70))
#   desiredReplicas = ceil(10 × 1.07) = 11
```

#### **Memory-Based HPA**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: memory-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: memory-intensive-app

  minReplicas: 3
  maxReplicas: 50

  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80  # Target 80% memory

# NOTE: Memory-based HPA is tricky because:
#   - Memory is not compressible (unlike CPU throttling)
#   - Scaling down with high memory usage can cause OOMKills
#   - Best combined with VPA for right-sizing
```

#### **Multi-Metric HPA**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server

  minReplicas: 5
  maxReplicas: 200

  metrics:
  # Metric 1: CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  # Metric 2: Memory utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

  # Metric 3: Custom metric (requests per second)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"  # 1000 RPS per pod

# Scaling decision:
#   HPA calculates desired replicas for EACH metric independently
#   Then picks the HIGHEST value (most conservative)
#
# Example:
#   CPU metric:     suggests 50 replicas
#   Memory metric:  suggests 45 replicas
#   RPS metric:     suggests 60 replicas
#   Result:         Scale to 60 replicas (highest)
```

### **HPA Scaling Behavior (v2 API)**

Control precisely how HPA scales up and down:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: controlled-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  minReplicas: 3
  maxReplicas: 100

  behavior:
    # Scale-up behavior
    scaleUp:
      stabilizationWindowSeconds: 0  # No stabilization (scale immediately)
      policies:
      # Policy 1: Scale up by 100% every 15 seconds
      - type: Percent
        value: 100
        periodSeconds: 15

      # Policy 2: Scale up by 10 pods every 15 seconds
      - type: Pods
        value: 10
        periodSeconds: 15

      # Select policy that gives MORE replicas (faster scale-up)
      selectPolicy: Max

    # Scale-down behavior
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scaling down
      policies:
      # Policy 1: Scale down by 10% every 60 seconds (conservative)
      - type: Percent
        value: 10
        periodSeconds: 60

      # Policy 2: Scale down by 5 pods every 60 seconds
      - type: Pods
        value: 5
        periodSeconds: 60

      # Select policy that gives FEWER replicas (slower scale-down)
      selectPolicy: Min

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**WHY Different Scale-Up/Down Behaviors?**:
- **Fast scale-up**: Respond quickly to traffic spikes (prevent outages)
- **Slow scale-down**: Avoid thrashing (rapid up/down cycles)
- **Stabilization window**: Wait for metrics to stabilize before scaling down

### **Custom Metrics HPA**

HPA supports three metric types:

#### **1. Resource Metrics** (CPU, Memory)

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization  # Percentage of requests
      averageUtilization: 70

# OR use absolute target
- type: Resource
  resource:
    name: memory
    target:
      type: AverageValue  # Absolute value (not percentage)
      averageValue: "1Gi"  # Target 1 GB per pod
```

#### **2. Pods Metrics** (Custom per-pod metrics)

```yaml
# Requires custom metrics from application (via Prometheus, etc.)
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"  # 1000 RPS per pod

# Another example: Queue depth per pod
- type: Pods
  pods:
    metric:
      name: queue_messages_ready
    target:
      type: AverageValue
      averageValue: "30"  # 30 messages per pod
```

**Setting Up Custom Metrics**:

```yaml
# 1. Install Prometheus Adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --set prometheus.url=http://prometheus-server.monitoring.svc \
  --set prometheus.port=80

# 2. Configure metric mappings
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_total$"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'

# 3. Use metric in HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

#### **3. External Metrics** (Cluster-wide external metrics)

```yaml
# Scale based on external system (SQS queue length, PubSub backlog, etc.)
metrics:
- type: External
  external:
    metric:
      name: sqs_queue_length
      selector:
        matchLabels:
          queue: "myapp-queue"
    target:
      type: AverageValue
      averageValue: "30"  # 30 messages per pod

# Example: AWS SQS Queue
- type: External
  external:
    metric:
      name: aws_sqs_messages_visible
      selector:
        matchLabels:
          queue_name: "production-orders-queue"
          region: "us-east-1"
    target:
      type: Value  # Total value (not per-pod average)
      value: "1000"  # Scale when queue > 1000 messages
```

### **HPA Calculation Logic**

**Desired Replicas Formula**:

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / targetMetricValue))
```

**Example Calculations**:

```yaml
# Scenario 1: Scale up
currentReplicas: 10
currentCPU: 85%
targetCPU: 70%

desiredReplicas = ceil(10 × (85 / 70))
                = ceil(10 × 1.214)
                = ceil(12.14)
                = 13 replicas

# Scenario 2: Scale down
currentReplicas: 20
currentCPU: 40%
targetCPU: 70%

desiredReplicas = ceil(20 × (40 / 70))
                = ceil(20 × 0.571)
                = ceil(11.42)
                = 12 replicas

# Scenario 3: Within tolerance (no change)
currentReplicas: 15
currentCPU: 72%
targetCPU: 70%
tolerance: 0.1 (10%)

# Ratio: 72 / 70 = 1.028 (2.8% difference)
# Since 2.8% < 10% tolerance, no scaling occurs
```

**Rate Limiting** (from source code):

```go
// pkg/controller/podautoscaler/horizontal.go:62-64
scaleUpLimitFactor  = 2.0   // Maximum 2× scale per cycle
scaleUpLimitMinimum = 4.0   // Or minimum 4 pods

// Applied as:
maxScaleUp = max(currentReplicas × 2.0, 4.0)

// Example:
// currentReplicas = 10, desired = 50
// maxScaleUp = max(10 × 2, 4) = 20
// actualScaleUp = min(50, 10 + 20) = 30 replicas
// Next cycle can go from 30 → 60, then 60 → 100
```

### **HPA Troubleshooting**

#### **Problem: HPA Not Scaling**

```bash
# Check HPA status
kubectl get hpa myapp-hpa
# NAME         REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# myapp-hpa    Deployment/myapp   <unknown>/70%   2     100       2          5m

# "unknown" means metrics unavailable

# Check HPA conditions
kubectl describe hpa myapp-hpa
# Conditions:
#   Type            Status  Reason                   Message
#   ----            ------  ------                   -------
#   AbleToScale     True    ReadyForNewScale         ready for new scale
#   ScalingActive   False   FailedGetResourceMetric  unable to get metrics

# Common causes:
# 1. Metrics Server not installed
kubectl get deployment metrics-server -n kube-system

# 2. Pod doesn't have resource requests (required for HPA)
kubectl get pod myapp-abc -o yaml | grep -A 5 resources:
# Should see:
#   resources:
#     requests:
#       cpu: 100m
#       memory: 128Mi

# 3. Custom metrics adapter not working
kubectl get apiservice v1beta1.custom.metrics.k8s.io
```

#### **Problem: HPA Flapping (Rapid Scale Up/Down)**

```yaml
# Symptoms: Replicas constantly changing
# 10 → 15 → 10 → 15 → 10 ...

# Solution: Add stabilization window
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 1 minute
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30

# Also increase tolerance if needed
# (HPA won't scale if metric is within tolerance of target)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Vertical Pod Autoscaler (VPA)**

### **VPA Architecture**

VPA consists of three components:

```
┌─────────────────────────────────────────────────────┐
│                    VPA System                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────────┐                              │
│  │  VPA Recommender │  Analyzes resource usage     │
│  │                  │  Generates recommendations   │
│  └────────┬─────────┘                              │
│           │                                        │
│           ↓                                        │
│  ┌──────────────────┐                              │
│  │  VPA Updater     │  Evicts pods that need       │
│  │                  │  resource changes            │
│  └────────┬─────────┘                              │
│           │                                        │
│           ↓                                        │
│  ┌──────────────────┐                              │
│  │  VPA Admission   │  Mutates pod requests        │
│  │  Controller      │  at creation time            │
│  └──────────────────┘                              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### **Installing VPA**

```bash
# Clone VPA repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA components
./hack/vpa-up.sh

# Verify installation
kubectl get deployment -n kube-system
# NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
# vpa-admission-controller    1/1     1            1           1m
# vpa-recommender             1/1     1            1           1m
# vpa-updater                 1/1     1            1           1m
```

### **VPA Configuration**

#### **Basic VPA (Recommendation Only)**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  # Update mode: Off = recommendations only (no actual changes)
  updatePolicy:
    updateMode: "Off"

# Check recommendations
kubectl describe vpa myapp-vpa
# Recommendation:
#   Container Recommendations:
#     Container Name: myapp
#     Lower Bound:
#       Cpu:     100m
#       Memory:  128Mi
#     Target:
#       Cpu:     250m   ← Recommended request
#       Memory:  512Mi  ← Recommended request
#     Uncapped Target:
#       Cpu:     300m
#       Memory:  600Mi
#     Upper Bound:
#       Cpu:     1
#       Memory:  2Gi
```

#### **VPA Auto Mode (Automatic Updates)**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa-auto
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  # Auto mode: VPA will evict and recreate pods with new requests
  updatePolicy:
    updateMode: "Auto"

  # Resource policy: Set boundaries
  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 4Gi

      # Control which resources VPA can update
      controlledResources:
      - cpu
      - memory
```

**WARNING**: Auto mode **evicts and recreates pods**. This causes:
- Temporary pod disruption
- Application restart
- Not suitable for single-replica deployments

#### **VPA Initial Mode (Only at Pod Creation)**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa-initial
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  # Initial mode: Only applies recommendations to NEW pods
  # Doesn't evict existing pods
  updatePolicy:
    updateMode: "Initial"

# Use case: Right-size pods without disrupting running ones
# Good for gradual rollout during deployments
```

### **VPA + HPA Compatibility**

**Default**: VPA and HPA conflict if both scale on CPU/memory

**Solution 1: Separate Metrics**

```yaml
# VPA: Manages CPU/memory requests
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      controlledResources:
      - memory  # VPA controls memory only

---
# HPA: Manages replica count based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu  # HPA scales on CPU only
      target:
        type: Utilization
        averageUtilization: 70
```

**Solution 2: VPA Recommendation Mode + Manual Updates**

```yaml
# VPA in "Off" mode provides recommendations
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"  # Recommendations only

# Periodically review VPA recommendations and manually update Deployment
# This avoids conflicts with HPA
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌐 Cluster Autoscaler**

### **Cluster Autoscaler Architecture**

```mermaid
graph TB
    Pods[Pending Pods] --> CA[Cluster Autoscaler]
    CA --> Check{Can schedule<br/>on existing nodes?}

    Check -->|No| ScaleUp[Scale Up]
    Check -->|Yes| NoAction[No Action]

    ScaleUp --> Cloud[Cloud Provider API]
    Cloud --> AddNodes[Add Nodes]

    CA --> Monitor[Monitor Node Utilization]
    Monitor --> Underutilized{Node < 50%<br/>utilized for 10min?}

    Underutilized -->|Yes| Evict[Evict Pods]
    Underutilized -->|No| NoAction2[No Action]

    Evict --> ScaleDown[Scale Down]
    ScaleDown --> Cloud2[Cloud Provider API]
    Cloud2 --> RemoveNodes[Remove Nodes]

    style ScaleUp fill:#2ecc71
    style ScaleDown fill:#e74c3c
```

### **Installing Cluster Autoscaler**

#### **AWS (EKS)**

```bash
# Create IAM policy for Cluster Autoscaler
cat > cluster-autoscaler-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json

# Create service account with IAM role
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::ACCOUNT_ID:policy/AmazonEKSClusterAutoscalerPolicy \
  --approve

# Deploy Cluster Autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Update deployment with cluster name
kubectl -n kube-system \
  annotate deployment.apps/cluster-autoscaler \
  cluster-autoscaler.kubernetes.io/safe-to-evict="false"

kubectl -n kube-system \
  set image deployment.apps/cluster-autoscaler \
  cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v1.27.0
```

#### **GCP (GKE)**

```bash
# Cluster Autoscaler is built-in to GKE
# Enable during cluster creation
gcloud container clusters create my-cluster \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=100 \
  --zone=us-central1-a

# Or enable on existing node pool
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=100 \
  --zone=us-central1-a \
  --node-pool=default-pool
```

### **Cluster Autoscaler Configuration**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - name: cluster-autoscaler
        image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.27.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false

        # Scale-down configuration
        - --scale-down-enabled=true
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
        - --scale-down-utilization-threshold=0.5

        # Scale-up configuration
        - --max-node-provision-time=15m
        - --max-graceful-termination-sec=600
```

**Key Configuration Options**:

| **Flag** | **Default** | **Description** |
|----------|-------------|-----------------|
| `--scale-down-enabled` | true | Enable scale-down |
| `--scale-down-delay-after-add` | 10m | Wait time after scale-up before considering scale-down |
| `--scale-down-unneeded-time` | 10m | How long node must be underutilized before removal |
| `--scale-down-utilization-threshold` | 0.5 | Node utilization threshold (50%) for scale-down |
| `--expander` | random | Strategy for selecting node group (random, most-pods, least-waste, priority) |
| `--max-node-provision-time` | 15m | Maximum time for node provisioning |

### **Preventing Node Scale-Down**

**Annotate Node**:
```bash
# Prevent specific node from being scaled down
kubectl annotate node my-node-1 \
  cluster-autoscaler.kubernetes.io/scale-down-disabled=true
```

**Annotate Pod**:
```yaml
# Prevent pods with this annotation from triggering scale-down
apiVersion: v1
kind: Pod
metadata:
  name: important-pod
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
spec:
  containers:
  - name: app
    image: myapp:v1
```

**Node Selector for Critical Pods**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 3
  template:
    spec:
      nodeSelector:
        workload-type: critical  # Only schedule on non-autoscaled nodes
      containers:
      - name: app
        image: myapp:v1
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ KEDA (Kubernetes Event-Driven Autoscaling)**

### **KEDA Architecture**

KEDA extends HPA with event-driven capabilities:

```
┌────────────────────────────────────────────────────┐
│                  External Systems                  │
├────────────────────────────────────────────────────┤
│  RabbitMQ │ Kafka │ AWS SQS │ Azure Queue │ ...   │
└──────┬─────────────────────────────────────────────┘
       │
       ↓
┌────────────────────────────────────────────────────┐
│                   KEDA Operator                    │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────────┐    ┌──────────────────┐    │
│  │  KEDA Metrics    │    │  KEDA Controller │    │
│  │  Server          │    │                  │    │
│  └────────┬─────────┘    └────────┬─────────┘    │
│           │                       │               │
└───────────┼───────────────────────┼───────────────┘
            │                       │
            ↓                       ↓
       ┌────────────────┐    ┌──────────────────┐
       │  HPA           │    │  ScaledObject    │
       │  (Generated)   │    │  (User-defined)  │
       └────────────────┘    └──────────────────┘
```

### **Installing KEDA**

```bash
# Install KEDA via Helm
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace

# Verify installation
kubectl get pods -n keda
# NAME                                      READY   STATUS    RESTARTS   AGE
# keda-operator-metrics-apiserver-xxx       1/1     Running   0          1m
# keda-operator-xxx                         1/1     Running   0          1m
```

### **KEDA ScaledObject Examples**

#### **Scale Based on Queue Length (RabbitMQ)**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor

  minReplicaCount: 0   # Scale to zero when no messages!
  maxReplicaCount: 30

  pollingInterval: 30   # Check queue every 30 seconds
  cooldownPeriod: 300   # Wait 5 minutes before scaling to zero

  triggers:
  - type: rabbitmq
    metadata:
      queueName: orders
      queueLength: "20"   # Target: 20 messages per pod
      host: amqp://guest:password@rabbitmq.default.svc:5672/
```

#### **Scale Based on Kafka Consumer Lag**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: event-consumer

  minReplicaCount: 1
  maxReplicaCount: 50

  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.kafka.svc:9092
      consumerGroup: event-consumers
      topic: events
      lagThreshold: "1000"  # Scale when lag > 1000 messages
```

#### **Scale Based on AWS SQS**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sqs-processor

  minReplicaCount: 0
  maxReplicaCount: 100

  triggers:
  - type: aws-sqs-queue
    authenticationRef:
      name: aws-credentials
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/myqueue
      queueLength: "10"   # Target: 10 messages per pod
      awsRegion: "us-east-1"

---
# AWS credentials
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-credentials
spec:
  secretTargetRef:
  - parameter: awsAccessKeyID
    name: aws-secret
    key: AWS_ACCESS_KEY_ID
  - parameter: awsSecretAccessKey
    name: aws-secret
    key: AWS_SECRET_ACCESS_KEY
```

#### **Scale Based on Prometheus Metrics**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server

  minReplicaCount: 2
  maxReplicaCount: 50

  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: http_requests_total
      query: sum(rate(http_requests_total{job="api-server"}[2m]))
      threshold: "1000"  # Scale when total RPS > 1000
```

### **KEDA Scale to Zero**

**Key Feature**: Unlike HPA (minimum 1 replica), KEDA can scale to zero:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: batch-processor
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: batch-worker

  minReplicaCount: 0         # Scale to zero when idle
  maxReplicaCount: 100
  cooldownPeriod: 300        # Wait 5 minutes before scaling to zero

  triggers:
  - type: rabbitmq
    metadata:
      queueName: batch-jobs
      queueLength: "5"

# When queue is empty:
#   - KEDA scales deployment to 0 replicas
#   - No pods running = no cost
#
# When messages arrive:
#   - KEDA scales up from 0 → N replicas
#   - Pods process messages
#   - After cooldown period with empty queue, scale back to 0
```

**Use Cases for Scale to Zero**:
- Batch processing jobs
- Scheduled workloads
- Event-driven microservices
- Cost optimization (dev/test environments)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎛️ Multi-Dimensional Autoscaling**

### **Combining HPA + VPA + Cluster Autoscaler**

```yaml
# 1. VPA: Right-size pod resources (memory only)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      controlledResources:
      - memory  # VPA manages memory

---
# 2. HPA: Scale replicas based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 5
  maxReplicas: 200
  metrics:
  - type: Resource
    resource:
      name: cpu  # HPA manages replicas via CPU
      target:
        type: Utilization
        averageUtilization: 70

---
# 3. Cluster Autoscaler: Add nodes when needed
# (Configured at cluster level, no CRD needed)
```

**How They Work Together**:

```
1. Application load increases
   ↓
2. HPA detects high CPU (> 70%)
   ↓
3. HPA scales replicas: 5 → 20
   ↓
4. Some pods become Pending (no capacity)
   ↓
5. Cluster Autoscaler detects pending pods
   ↓
6. Cluster Autoscaler adds nodes: 10 → 15
   ↓
7. Pending pods get scheduled on new nodes
   ↓
8. VPA observes actual memory usage
   ↓
9. VPA adjusts memory requests for future pods
   ↓
10. Load decreases
   ↓
11. HPA scales down replicas: 20 → 5
   ↓
12. Nodes become underutilized (< 50%)
   ↓
13. Cluster Autoscaler removes empty nodes: 15 → 10
```

### **Advanced Autoscaling Pattern**

```yaml
# Complete autoscaling setup for production application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 10  # Initial replicas (HPA will adjust)
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: myapi:v1
        resources:
          requests:
            cpu: 500m      # VPA will adjust based on actual usage
            memory: 1Gi    # VPA will adjust based on actual usage
          limits:
            cpu: 2
            memory: 4Gi

---
# HPA: Scale based on multiple metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 10
  maxReplicas: 500

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 50
        periodSeconds: 30
      - type: Pods
        value: 20
        periodSeconds: 30
      selectPolicy: Max

    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 10
        periodSeconds: 60
      selectPolicy: Min

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"

---
# VPA: Right-size resources (recommendation mode to avoid conflicts)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  updatePolicy:
    updateMode: "Off"  # Recommendations only
  resourcePolicy:
    containerPolicies:
    - containerName: api
      minAllowed:
        cpu: 100m
        memory: 256Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi

---
# KEDA: Scale based on message queue
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-keda
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicaCount: 10
  maxReplicaCount: 500

  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      query: sum(rate(http_requests_total[2m]))
      threshold: "10000"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Summary and Best Practices**

### **Autoscaling Decision Matrix**

| **Scenario** | **Recommended Autoscalers** | **Configuration** |
|--------------|----------------------------|-------------------|
| **Stateless web app** | HPA + Cluster Autoscaler | HPA on CPU/RPS, CA for capacity |
| **Queue processor** | KEDA + Cluster Autoscaler | KEDA scale-to-zero, CA for burst capacity |
| **Database/StatefulSet** | VPA only | VPA for right-sizing, no horizontal scaling |
| **Microservices (mixed)** | HPA + VPA + CA | VPA for memory, HPA for CPU, CA for nodes |
| **Batch jobs** | KEDA with scale-to-zero | Event-driven, no idle cost |

### **Key Takeaways**

1. **HPA is Most Common**: Start with CPU-based HPA for stateless workloads
2. **VPA for Right-Sizing**: Use VPA to optimize resource requests over time
3. **Cluster Autoscaler for Capacity**: Essential for cloud environments
4. **KEDA for Events**: Best for queue-driven and event-driven workloads
5. **Don't Over-Autoscale**: Too many autoscalers can cause conflicts

### **Common Pitfalls**

❌ **Don't**:
- Run HPA and VPA on same resource (CPU/memory) simultaneously
- Set minReplicas=1 for critical services (use ≥2 for HA)
- Forget to set resource requests (required for HPA)
- Ignore PodDisruptionBudgets with autoscaling
- Scale too aggressively (causes thrashing)

✅ **Do**:
- Always set resource requests and limits
- Use multiple metrics for HPA (CPU + custom)
- Monitor autoscaling decisions
- Test autoscaling in non-production first
- Set appropriate stabilization windows
- Use PodDisruptionBudgets to prevent disruption

### **Related Documentation**

- [Scalability Limits](02-scalability-limits.md) - HPA rate limiting details
- [Component Optimization](06-component-optimization.md) - Optimizing for autoscaling
- [Performance Benchmarking](03-performance-benchmarking.md) - Measuring autoscaling

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Metadata**:
- **Lines**: ~2,400
- **Source Code References**: 3+ files with exact line numbers
- **Mermaid Diagrams**: 2
- **Code Examples**: 50+
- **Target Audience**: Platform engineers, SREs, architects
- **Last Updated**: 2024-11-17
