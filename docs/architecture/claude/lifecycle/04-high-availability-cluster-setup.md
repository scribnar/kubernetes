# **High Availability Cluster Setup - Deep Architectural Analysis**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Target Audience**: Platform engineers, Kubernetes architects, SREs managing production clusters (5000+ nodes), infrastructure teams

**Scope**: Deep architectural analysis of Kubernetes high availability (HA) patterns, multi-master control plane design, etcd cluster topology, load balancing strategies, and failure recovery mechanisms. This document examines production-grade HA setups from the source code level to help platform engineers design resilient Kubernetes platforms.

**Prerequisites**:
- Understanding of [kubeadm architecture](./01-kubeadm-architecture.md)
- Familiarity with [control plane initialization](./03-control-plane-initialization.md)
- Knowledge of [etcd cluster management](../etcd/middle-level/05-cluster-management.md)
- Understanding of [leader election](../distributed-systems/03-leader-election.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Design Philosophy**

### **Why High Availability Matters**

Production Kubernetes clusters require **continuous availability**:

- **No single point of failure**: Control plane component failure doesn't bring down cluster
- **Zero-downtime upgrades**: Rolling control plane upgrades without API downtime
- **Disaster recovery**: Survive datacenter/AZ failures
- **Performance**: Distribute load across multiple API servers
- **Geographic distribution**: Serve global users with low latency

### **HA Design Principles**

```
┌──────────────────────────────────────────────────────────────┐
│  HIGH AVAILABILITY DESIGN PRINCIPLES                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. REDUNDANCY                                               │
│     └─ Multiple replicas of every control plane component   │
│                                                               │
│  2. QUORUM-BASED CONSENSUS                                   │
│     └─ etcd requires majority for operation (2n+1)          │
│                                                               │
│  3. STATELESS API SERVERS                                    │
│     └─ Multiple API servers behind load balancer            │
│                                                               │
│  4. ACTIVE-STANDBY CONTROLLERS                               │
│     └─ Leader election ensures single active instance       │
│                                                               │
│  5. ZONE-AWARE PLACEMENT                                     │
│     └─ Distribute control plane across failure domains      │
│                                                               │
│  6. HEALTH-BASED ROUTING                                     │
│     └─ Traffic only to healthy endpoints                    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏗️ HA Topology Options**

### **Topology 1: Stacked etcd**

**Definition**: etcd runs on same nodes as control plane components

```
┌─────────────────────────────────────────────────────────────┐
│  STACKED ETCD TOPOLOGY                                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Control Plane Node 1         Control Plane Node 2          │
│  ┌──────────────────┐         ┌──────────────────┐          │
│  │  kube-apiserver  │◄────────┤  kube-apiserver  │          │
│  ├──────────────────┤         ├──────────────────┤          │
│  │  kube-scheduler  │         │  kube-scheduler  │          │
│  │  (standby)       │         │  (leader)        │          │
│  ├──────────────────┤         ├──────────────────┤          │
│  │  kube-controller │         │  kube-controller │          │
│  │  -manager        │         │  -manager        │          │
│  │  (leader)        │         │  (standby)       │          │
│  ├──────────────────┤         ├──────────────────┤          │
│  │  etcd (member 1) │◄───────►│  etcd (member 2) │          │
│  └──────────────────┘         └──────────────────┘          │
│           ▲                             ▲                    │
│           │                             │                    │
│           └──────────┬──────────────────┘                    │
│                      │                                       │
│           Control Plane Node 3                               │
│           ┌──────────────────┐                               │
│           │  kube-apiserver  │                               │
│           ├──────────────────┤                               │
│           │  kube-scheduler  │                               │
│           │  (standby)       │                               │
│           ├──────────────────┤                               │
│           │  kube-controller │                               │
│           │  -manager        │                               │
│           │  (standby)       │                               │
│           ├──────────────────┤                               │
│           │  etcd (member 3) │                               │
│           └──────────────────┘                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Characteristics**:
- **Simplicity**: Single set of nodes for entire control plane
- **Resource efficiency**: No dedicated etcd nodes needed
- **Failure domain**: Loss of control plane node = loss of etcd member
- **Minimum nodes**: 3 (for etcd quorum)
- **Recommended size**: 3-5 nodes

**Pros**:
- ✅ Simple deployment (kubeadm default)
- ✅ Lower infrastructure cost
- ✅ Easier to manage (fewer nodes)
- ✅ Good for small-medium clusters (<1000 nodes)

**Cons**:
- ❌ Coupled failure domain (control plane + etcd)
- ❌ Resource contention (API server and etcd compete for resources)
- ❌ Limited scalability (etcd performance bound by node resources)
- ❌ Cannot scale API servers independently from etcd

### **Topology 2: External etcd**

**Definition**: etcd cluster on dedicated nodes, separate from control plane

```
┌─────────────────────────────────────────────────────────────┐
│  EXTERNAL ETCD TOPOLOGY                                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Control Plane Nodes                                         │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │  kube-apiserver  │  │  kube-apiserver  │                 │
│  ├──────────────────┤  ├──────────────────┤                 │
│  │  kube-scheduler  │  │  kube-scheduler  │                 │
│  ├──────────────────┤  ├──────────────────┤                 │
│  │  kube-controller │  │  kube-controller │                 │
│  │  -manager        │  │  -manager        │                 │
│  └──────────────────┘  └──────────────────┘                 │
│           │                      │                           │
│           └──────────┬───────────┘                           │
│                      │                                       │
│                      ▼                                       │
│  ┌─────────────────────────────────────────────┐            │
│  │  Load Balancer                              │            │
│  │  (to etcd cluster)                          │            │
│  └─────────────────────────────────────────────┘            │
│                      │                                       │
│           ┌──────────┼──────────┐                            │
│           ▼          ▼          ▼                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ etcd Node 1 │ │ etcd Node 2 │ │ etcd Node 3 │           │
│  │             │ │             │ │             │           │
│  │ etcd member │◄┤ etcd member │►│ etcd member │           │
│  │     1       │ │      2      │ │      3      │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Characteristics**:
- **Separation of concerns**: etcd isolated from control plane workloads
- **Independent scaling**: Scale API servers and etcd separately
- **Dedicated resources**: etcd gets dedicated CPU/memory/disk I/O
- **Minimum nodes**: 3 etcd + 2 control plane = 5 total
- **Recommended size**: 3-5 etcd nodes, 3+ control plane nodes

**Pros**:
- ✅ Better fault isolation (control plane failure doesn't affect etcd)
- ✅ Independent scaling (add API servers without affecting etcd)
- ✅ Optimized resources (etcd nodes tuned for storage workload)
- ✅ Better for large clusters (5000+ nodes)

**Cons**:
- ❌ More complex setup
- ❌ Higher infrastructure cost (more nodes)
- ❌ Additional operational complexity (two separate clusters)
- ❌ Network dependency (control plane ↔ etcd latency)

### **Topology Comparison**

| **Criteria** | **Stacked etcd** | **External etcd** |
|--------------|-----------------|-------------------|
| **Minimum nodes** | 3 | 5 (3 etcd + 2 CP) |
| **Setup complexity** | Simple | Complex |
| **Failure domain** | Coupled | Separated |
| **Resource efficiency** | High | Medium |
| **Performance (large clusters)** | Limited | Better |
| **Operational complexity** | Low | Medium |
| **Cost** | Lower | Higher |
| **Recommended for** | <1000 nodes | 5000+ nodes |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 etcd Cluster Setup**

### **etcd Quorum Mathematics**

**Quorum**: Majority of members needed for operation

| **Cluster Size** | **Quorum** | **Tolerated Failures** | **Recommended** |
|-----------------|-----------|----------------------|----------------|
| 1 | 1 | 0 | ❌ Dev only |
| 2 | 2 | 0 | ❌ Worse than 1 |
| 3 | 2 | 1 | ✅ Good |
| 4 | 3 | 1 | ❌ Same as 3, more cost |
| 5 | 3 | 2 | ✅ Better |
| 6 | 4 | 2 | ❌ Same as 5, more cost |
| 7 | 4 | 3 | ✅ Best |

**Key Insight**: Only use **odd numbers** of etcd members

**Why?**
- 2-member cluster: Quorum = 2, tolerance = 0 (same as 1-member)
- 3-member cluster: Quorum = 2, tolerance = 1
- 4-member cluster: Quorum = 3, tolerance = 1 (same as 3-member, wastes resources)

**Production Recommendation**: **3 or 5 members**

### **Stacked etcd Setup (kubeadm)**

#### **Step 1: Initialize First Control Plane Node**

```bash
# control-plane-1
sudo kubeadm init \
  --control-plane-endpoint "k8s-api-lb.example.com:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16
```

**Key Flags**:
- `--control-plane-endpoint`: Load balancer endpoint for API server
- `--upload-certs`: Upload certificates to cluster (for joining other masters)
- `--pod-network-cidr`: Pod network CIDR (for CNI)

**Output**:
```
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s-api-lb.example.com:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:abc123... \
    --control-plane --certificate-key def456...

Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join k8s-api-lb.example.com:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:abc123...
```

#### **Step 2: Join Additional Control Plane Nodes**

```bash
# control-plane-2
sudo kubeadm join k8s-api-lb.example.com:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:abc123... \
  --control-plane \
  --certificate-key def456...

# control-plane-3
sudo kubeadm join k8s-api-lb.example.com:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:abc123... \
  --control-plane \
  --certificate-key def456...
```

**What Happens**:
1. kubeadm downloads certificates from cluster (uploaded in step 1)
2. Generates etcd member configuration
3. Joins existing etcd cluster
4. Starts static pods (API server, controller manager, scheduler)
5. Updates kubeadm-config ConfigMap with new member

#### **Step 3: Verify etcd Cluster**

```bash
# Check etcd cluster members
kubectl exec -n kube-system etcd-control-plane-1 -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Output:
# 8e9e05c52164694d, started, control-plane-1, https://192.168.1.101:2380, https://192.168.1.101:2379, false
# 91bc3c398fb3c146, started, control-plane-2, https://192.168.1.102:2380, https://192.168.1.102:2379, false
# fd422379fda50e48, started, control-plane-3, https://192.168.1.103:2380, https://192.168.1.103:2379, false
```

### **External etcd Setup**

#### **Step 1: Setup etcd Cluster (Manual)**

**On each etcd node** (etcd-1, etcd-2, etcd-3):

```bash
# etcd-1 (192.168.1.11)
ETCD_NAME="etcd-1"
ETCD_IP="192.168.1.11"

cat <<EOF > /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos/etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --listen-client-urls https://${ETCD_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${ETCD_IP}:2379 \\
  --listen-peer-urls https://${ETCD_IP}:2380 \\
  --initial-advertise-peer-urls https://${ETCD_IP}:2380 \\
  --initial-cluster etcd-1=https://192.168.1.11:2380,etcd-2=https://192.168.1.12:2380,etcd-3=https://192.168.1.13:2380 \\
  --initial-cluster-state new \\
  --initial-cluster-token etcd-cluster-1 \\
  --data-dir /var/lib/etcd \\
  --cert-file=/etc/etcd/pki/server.crt \\
  --key-file=/etc/etcd/pki/server.key \\
  --peer-cert-file=/etc/etcd/pki/peer.crt \\
  --peer-key-file=/etc/etcd/pki/peer.key \\
  --trusted-ca-file=/etc/etcd/pki/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/pki/ca.crt \\
  --client-cert-auth \\
  --peer-client-cert-auth
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

**Repeat for etcd-2, etcd-3** with appropriate IPs

#### **Step 2: Initialize First Control Plane with External etcd**

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.30.0
controlPlaneEndpoint: "k8s-api-lb.example.com:6443"
etcd:
  external:
    endpoints:
      - https://192.168.1.11:2379
      - https://192.168.1.12:2379
      - https://192.168.1.13:2379
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
networking:
  podSubnet: 10.244.0.0/16
```

```bash
# Initialize
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs
```

**Note**: API server will connect to external etcd instead of local etcd

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚖️ Load Balancing the API Server**

### **Why Load Balance?**

Multiple kube-apiserver instances need:
- **Single entry point**: Clients use single endpoint
- **Load distribution**: Distribute requests across API servers
- **Health-based routing**: Route only to healthy API servers
- **High availability**: No single point of failure

### **Load Balancer Options**

| **Option** | **Type** | **Pros** | **Cons** | **Use Case** |
|------------|---------|----------|----------|--------------|
| **HAProxy** | Software | Free, flexible, feature-rich | Requires dedicated node/HA pair | On-prem, self-managed |
| **nginx** | Software | Simple, widely used | Less feature-rich than HAProxy | On-prem, simple setups |
| **keepalived** | Virtual IP | No dedicated LB, IP failover | Limited to L3/L4 | Small on-prem setups |
| **Cloud LB** | Managed | Fully managed, HA built-in | Cloud-specific, cost | AWS, GCP, Azure |
| **kube-vip** | Kubernetes-native | Runs in cluster, no external deps | Relatively new | Modern on-prem |

### **HAProxy Configuration**

**Setup** (on dedicated load balancer nodes):

```bash
# Install HAProxy
sudo apt-get install haproxy

# Configure HAProxy
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

# Kubernetes API Server Frontend
frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

# Kubernetes API Server Backend
backend k8s-api-backend
    mode tcp
    option tcp-check
    balance roundrobin

    # Health check: TCP connect to port 6443
    tcp-check connect port 6443

    # Control plane nodes
    server control-plane-1 192.168.1.101:6443 check fall 3 rise 2
    server control-plane-2 192.168.1.102:6443 check fall 3 rise 2
    server control-plane-3 192.168.1.103:6443 check fall 3 rise 2

# Stats interface
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 30s
EOF

# Restart HAProxy
sudo systemctl restart haproxy
```

**Health Check Parameters**:
- `check`: Enable health checks
- `fall 3`: Mark unhealthy after 3 failed checks
- `rise 2`: Mark healthy after 2 successful checks
- Default check interval: 2 seconds

**HA HAProxy Setup**:
```
┌──────────────────────────────────────────────────────────────┐
│  HIGH AVAILABILITY HAPROXY                                    │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐          ┌─────────────┐                    │
│  │ HAProxy-1   │          │ HAProxy-2   │                    │
│  │ (MASTER)    │◄────────►│ (BACKUP)    │                    │
│  │             │          │             │                    │
│  │ keepalived  │  VRRP    │ keepalived  │                    │
│  └─────────────┘          └─────────────┘                    │
│         │                                                     │
│         └─────────────┬────────────────────────────          │
│                       │                                       │
│              Virtual IP: 192.168.1.100                        │
│                       │                                       │
│         ┌─────────────┼────────────┐                          │
│         ▼             ▼            ▼                          │
│  control-plane-1  control-plane-2  control-plane-3           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**keepalived Configuration** (for HA HAProxy):

```bash
# On HAProxy-1 (MASTER)
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass mypassword
    }

    virtual_ipaddress {
        192.168.1.100
    }

    track_script {
        chk_haproxy
    }
}
EOF

# On HAProxy-2 (BACKUP): Same config but priority 100
```

### **Cloud Load Balancer Integration**

#### **AWS Network Load Balancer**

```bash
# Create NLB targeting control plane nodes
aws elbv2 create-load-balancer \
  --name k8s-api-nlb \
  --type network \
  --subnets subnet-abc123 subnet-def456 subnet-ghi789 \
  --scheme internal

# Create target group
aws elbv2 create-target-group \
  --name k8s-api-targets \
  --protocol TCP \
  --port 6443 \
  --vpc-id vpc-12345678 \
  --health-check-protocol TCP \
  --health-check-port 6443

# Register control plane nodes
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-control-plane-1 Id=i-control-plane-2 Id=i-control-plane-3
```

**Advantages**:
- Fully managed
- Multi-AZ HA built-in
- Automatic health checks
- Integrated with AWS services (Route53, CloudWatch)

#### **GCP Load Balancer**

```bash
# Create instance group with control plane nodes
gcloud compute instance-groups unmanaged create k8s-control-plane-ig \
  --zone us-central1-a

gcloud compute instance-groups unmanaged add-instances k8s-control-plane-ig \
  --instances control-plane-1,control-plane-2,control-plane-3 \
  --zone us-central1-a

# Create health check
gcloud compute health-checks create tcp k8s-api-health \
  --port 6443

# Create backend service
gcloud compute backend-services create k8s-api-backend \
  --protocol TCP \
  --health-checks k8s-api-health \
  --global

# Add instance group to backend
gcloud compute backend-services add-backend k8s-api-backend \
  --instance-group k8s-control-plane-ig \
  --instance-group-zone us-central1-a \
  --global

# Create forwarding rule (TCP load balancer)
gcloud compute forwarding-rules create k8s-api-lb \
  --load-balancing-scheme INTERNAL \
  --backend-service k8s-api-backend \
  --ports 6443 \
  --network default \
  --subnet default \
  --region us-central1
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **👑 Leader Election**

### **Why Leader Election?**

**Problem**: Multiple controller-manager and scheduler instances

**Solution**: Only one instance actively processes resources (leader), others standby

```
┌──────────────────────────────────────────────────────────────┐
│  WITHOUT LEADER ELECTION          WITH LEADER ELECTION        │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Controller Manager 1              Controller Manager 1      │
│  ✓ Reconciling deployments        ✓ Reconciling (LEADER)    │
│                                                               │
│  Controller Manager 2              Controller Manager 2      │
│  ✓ Reconciling deployments        ⏸ Watching (STANDBY)      │
│                                                               │
│  Result: Conflicts!                Result: Safe!             │
│  - Duplicate pod creations         - Single active instance  │
│  - Race conditions                 - Automatic failover      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### **Leader Election Implementation**

**Mechanism**: Lease-based leader election using `coordination.k8s.io/v1` Lease resource

```go
// vendor/k8s.io/client-go/tools/leaderelection/leaderelection.go

type LeaderElector struct {
    config LeaderElectionConfig
}

type LeaderElectionConfig struct {
    // Lock to coordinate election
    Lock resourcelock.Interface

    // LeaseDuration: How long lease is valid
    LeaseDuration time.Duration  // Default: 15s

    // RenewDeadline: Leader must renew before this deadline
    RenewDeadline time.Duration  // Default: 10s

    // RetryPeriod: How often to attempt acquisition/renewal
    RetryPeriod time.Duration    // Default: 2s

    // Callbacks
    Callbacks LeaderCallbacks
}

type LeaderCallbacks struct {
    OnStartedLeading func(context.Context)
    OnStoppedLeading func()
    OnNewLeader      func(identity string)
}
```

**Lease Resource**:
```bash
kubectl get lease -n kube-system kube-controller-manager -o yaml
```

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: "control-plane-1_abc123-def456-ghi789"
  leaseDurationSeconds: 15
  acquireTime: "2024-01-15T10:00:00Z"
  renewTime: "2024-01-15T10:00:45Z"
  leaseTransitions: 3
```

**Fields**:
- `holderIdentity`: Current leader (hostname + random suffix)
- `leaseDurationSeconds`: Lease validity period
- `acquireTime`: When current leader acquired lease
- `renewTime`: Last time leader renewed lease
- `leaseTransitions`: Number of leader changes (for monitoring)

### **Failover Scenario**

```
T+0s    : control-plane-1 is leader
          Lease: holderIdentity=control-plane-1, renewTime=T+0s

T+10s   : control-plane-1 renews lease
          Lease: renewTime=T+10s

T+15s   : control-plane-1 CRASHES (network partition, node failure, etc.)

T+17s   : control-plane-2 tries to renew (fails, not leader)
          control-plane-3 tries to renew (fails, not leader)

T+25s   : Lease expires (renewTime T+10s + leaseDuration 15s = T+25s)

T+26s   : control-plane-2 acquires lease
          Lease: holderIdentity=control-plane-2, acquireTime=T+26s, renewTime=T+26s
          leaseTransitions: 4 (incremented)

T+26s   : control-plane-2 becomes leader, starts controllers

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Failover time: ~11 seconds (RenewDeadline 10s + RetryPeriod 2s)
```

**Tuning Leader Election**:

| **Parameter** | **Default** | **Lower** | **Higher** | **Recommendation** |
|---------------|------------|-----------|-----------|-------------------|
| `LeaseDuration` | 15s | Faster failover | More stable | 15s for production |
| `RenewDeadline` | 10s | Faster detection | Less flapping | 10s |
| `RetryPeriod` | 2s | Faster acquisition | Less API load | 2s |

**Production Configuration** (stable):
```yaml
LeaseDuration: 15s
RenewDeadline: 10s
RetryPeriod: 2s
# Failover time: ~12s
```

**Aggressive Configuration** (fast failover):
```yaml
LeaseDuration: 10s
RenewDeadline: 7s
RetryPeriod: 1s
# Failover time: ~8s
# Warning: More API server load, more sensitive to network blips
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌍 Multi-Zone HA**

### **Zone-Aware Control Plane Placement**

**Goal**: Survive entire availability zone (AZ) failure

```
┌──────────────────────────────────────────────────────────────┐
│  MULTI-ZONE HIGH AVAILABILITY                                 │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Zone 1              Zone 2              Zone 3              │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐      │
│  │ CP Node 1  │      │ CP Node 2  │      │ CP Node 3  │      │
│  │            │      │            │      │            │      │
│  │ API Server │      │ API Server │      │ API Server │      │
│  │ Scheduler  │      │ Scheduler  │      │ Scheduler  │      │
│  │ Controller │      │ Controller │      │ Controller │      │
│  │ etcd-1     │      │ etcd-2     │      │ etcd-3     │      │
│  └────────────┘      └────────────┘      └────────────┘      │
│         │                   │                   │             │
│         └───────────────────┴───────────────────┘             │
│                             │                                 │
│                    Load Balancer (Multi-AZ)                   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**etcd Quorum with Zone Failure**:

| **Scenario** | **Zones** | **etcd Members** | **Quorum** | **Result** |
|--------------|----------|-----------------|-----------|------------|
| Normal | 3 | 3 (1+1+1) | 2 | ✅ Operational |
| Zone 1 fails | 2 | 2 (0+1+1) | 2 | ✅ Operational (barely) |
| Zone 1 & 2 fail | 1 | 1 (0+0+1) | 2 | ❌ etcd unavailable |

**Implications**:
- **3-zone, 3-member etcd**: Survives 1 zone failure
- **3-zone, 5-member etcd** (2+2+1 distribution): Survives 1 zone failure (quorum = 3)
- **5-zone, 5-member etcd**: Survives 2 zone failures

**Recommended Distribution**:
- **3 zones**: 3 etcd members (1 per zone) or 5 etcd members (2+2+1)
- **5 zones**: 5 etcd members (1 per zone)

### **Worker Node Zone Distribution**

```yaml
# Node labels automatically added by cloud-controller-manager
apiVersion: v1
kind: Node
metadata:
  labels:
    topology.kubernetes.io/zone: us-west-2a  # or us-west-2b, us-west-2c
```

**Pod Topology Spread**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
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
            app: myapp
      containers:
      - name: app
        image: myapp:1.0
```

**Result**: 2 pods per zone (6 pods / 3 zones)

**See**: [Topology Spread Constraints](../scheduler/middle-level/06-topology-spread.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Failure Scenarios and Recovery**

### **Scenario 1: Single Control Plane Node Failure**

**Failure**: control-plane-2 crashes

**Impact**:
- ✅ API server: 2 of 3 still serving (load balancer routes to healthy instances)
- ✅ etcd: 2 of 3 members operational (quorum maintained)
- ✅ Controller manager: Leader on control-plane-1 or control-plane-3 (unaffected)
- ✅ Scheduler: Leader on control-plane-1 or control-plane-3 (unaffected)

**User Impact**: **None** (transparent failover)

**Recovery**:
```bash
# Fix and restart node
sudo reboot

# Verify node rejoins cluster
kubectl get nodes

# Verify etcd member rejoins
kubectl exec -n kube-system etcd-control-plane-1 -- etcdctl member list
```

**Automatic Recovery**: Yes (kubelet restarts static pods)

### **Scenario 2: etcd Quorum Loss**

**Failure**: 2 of 3 etcd members crash

**Impact**:
- ❌ etcd: No quorum (1 of 3 members, need 2)
- ❌ API server: Cannot write to etcd (read-only mode or unavailable)
- ❌ kubectl: Most commands fail
- ❌ New pods: Cannot be scheduled
- ✅ Existing pods: Continue running (kubelet doesn't depend on API server)

**Recovery**:

**Option 1**: Restore crashed members
```bash
# Fix etcd nodes and restart etcd
sudo systemctl restart etcd

# Verify quorum
etcdctl endpoint health --endpoints=https://192.168.1.11:2379,https://192.168.1.12:2379,https://192.168.1.13:2379
```

**Option 2**: Restore from backup (if members unrecoverable)
```bash
# Stop all etcd members
sudo systemctl stop etcd

# On all nodes, restore from snapshot
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name etcd-1 \
  --initial-cluster etcd-1=https://192.168.1.11:2380,etcd-2=https://192.168.1.12:2380,etcd-3=https://192.168.1.13:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://192.168.1.11:2380 \
  --data-dir /var/lib/etcd-restored

# Update etcd config to use restored data directory
# Restart etcd on all nodes
sudo systemctl start etcd
```

**See**: [etcd Backup and Restore](../etcd/middle-level/06-backup-restore.md)

### **Scenario 3: Network Partition**

**Failure**: Network split between control plane nodes

**Scenario**:
```
Partition 1: control-plane-1, control-plane-2
Partition 2: control-plane-3
```

**Impact**:
- ✅ etcd: Partition 1 has quorum (2 of 3), continues operation
- ❌ etcd: Partition 2 has no quorum (1 of 3), read-only
- ✅ API server: Partition 1 API servers operational
- ❌ API server: Partition 2 API server read-only/unavailable
- ⚠️ Leader election: Both partitions may try to elect leaders (but only Partition 1 succeeds)

**Worker Nodes**:
- Workers connected to Partition 1 LB endpoint: ✅ Operational
- Workers connected to Partition 2 LB endpoint: ❌ Cannot schedule pods

**Recovery**:
```bash
# Restore network connectivity
# Once partition heals:

# 1. etcd members automatically re-sync
# 2. Leader election re-stabilizes
# 3. API servers resume normal operation

# Verify cluster health
kubectl get cs  # Check component status
kubectl get nodes  # Check node status
```

### **Scenario 4: All Control Plane Nodes Down**

**Failure**: Complete control plane outage

**Impact**:
- ❌ API server: Unavailable
- ❌ kubectl: All commands fail
- ❌ Scheduler: Not running
- ❌ Controllers: Not running
- ✅ **Existing pods: Continue running** (kubelet operates autonomously)

**What Still Works**:
- Pods continue running (kubelet manages pod lifecycle)
- Services continue routing (kube-proxy iptables rules persist)
- Volumes remain attached (CSI drivers operate independently)

**What Doesn't Work**:
- Cannot create new pods
- Cannot update existing resources
- Cannot schedule pending pods
- Controllers not reconciling (e.g., crashed pods not restarted)

**Recovery**:
```bash
# Restore control plane nodes
# Priority:
# 1. etcd (restore from backup if needed)
# 2. API server
# 3. Controller manager
# 4. Scheduler

# Verify recovery
kubectl cluster-info
kubectl get cs
```

**RTO** (Recovery Time Objective): Depends on failure cause
- **Simple restart**: 2-5 minutes
- **etcd restore from backup**: 10-30 minutes
- **Full rebuild**: Hours

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Monitoring HA Health**

### **Key Metrics**

**etcd Metrics**:
```promql
# etcd leader changes (should be rare)
rate(etcd_server_leader_changes_seen_total[5m])

# etcd proposal failures
rate(etcd_server_proposals_failed_total[5m])

# etcd backend commit duration (latency)
histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m]))

# etcd member health
up{job="etcd"}
```

**API Server Metrics**:
```promql
# API server request latency
histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m]))

# API server error rate
rate(apiserver_request_total{code=~"5.."}[5m])

# API server availability
up{job="kube-apiserver"}
```

**Leader Election Metrics**:
```promql
# Leader election lease renewals
rate(leader_election_lease_renew_total[5m])

# Leader election errors
rate(leader_election_lease_renew_errors_total[5m])
```

**Load Balancer Health**:
```bash
# HAProxy stats
curl http://haproxy-lb:8404/stats

# Check backend server status
# Should show all 3 control plane nodes as "UP"
```

### **Alerting Rules**

```yaml
# prometheus-rules.yaml
groups:
- name: k8s-control-plane
  rules:
  - alert: EtcdInsufficientMembers
    expr: count(up{job="etcd"} == 1) < 3
    for: 3m
    annotations:
      summary: "etcd cluster has insufficient members"
      description: "etcd cluster has {{ $value }} members, need 3"

  - alert: EtcdNoLeader
    expr: etcd_server_has_leader{job="etcd"} == 0
    for: 1m
    annotations:
      summary: "etcd has no leader"

  - alert: APIServerDown
    expr: up{job="kube-apiserver"} == 0
    for: 1m
    annotations:
      summary: "API server is down"

  - alert: ControllerManagerNoLeader
    expr: time() - kube_controller_manager_leader_election_time > 30
    for: 5m
    annotations:
      summary: "Controller manager has no leader"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Related Documentation**

### **Core Components**
- **[kubeadm Architecture](./01-kubeadm-architecture.md)** - Cluster bootstrapping
- **[Control Plane Initialization](./03-control-plane-initialization.md)** - Component startup sequences
- **[etcd Cluster Management](../etcd/middle-level/05-cluster-management.md)** - etcd operations
- **[Leader Election](../distributed-systems/03-leader-election.md)** - Distributed consensus

### **Operational Context**
- **[Disaster Recovery](../scalability/04-disaster-recovery-strategies.md)** - Backup and restore procedures
- **[Upgrade Strategies](./02-kubeadm-upgrade-strategies.md)** - Upgrading HA clusters
- **[Performance Tuning](../etcd/middle-level/07-performance-tuning.md)** - etcd performance optimization

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Key Takeaways**

### **For Platform Engineers**

1. **Choose Topology Based on Scale**:
   - Stacked etcd: <1000 nodes, simpler operations
   - External etcd: 5000+ nodes, better isolation

2. **etcd Quorum is Critical**:
   - Use odd numbers (3, 5, 7)
   - 3 members = tolerate 1 failure
   - 5 members = tolerate 2 failures

3. **Load Balancer is Essential**:
   - Single endpoint for API server
   - Health-based routing
   - Consider cloud-managed LB for simplicity

4. **Leader Election Provides HA**:
   - Only one controller manager actively reconciles
   - Automatic failover (~12s default)
   - Tune parameters for faster failover if needed

5. **Multi-Zone for True HA**:
   - Distribute control plane across zones
   - Survive entire AZ failure
   - Careful etcd member distribution

### **For Kubernetes Contributors**

1. **Leader Election Implementation**:
   - Uses `coordination.k8s.io/v1` Lease
   - Lease-based (not ConfigMap-based anymore)
   - Code: `vendor/k8s.io/client-go/tools/leaderelection/`

2. **etcd Integration**:
   - API server connects to etcd via TLS
   - Supports multiple etcd endpoints (HA)
   - Automatic retry on connection failure

3. **kubeadm HA**:
   - `--control-plane-endpoint`: Mandatory for HA
   - `--upload-certs`: Shares certificates for joining masters
   - Join with `--control-plane` flag for master nodes

4. **Testing HA**:
   - Chaos engineering: Kill random control plane nodes
   - Network partition testing
   - Zone failure simulation

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Kubernetes Version**: v1.30
**Last Updated**: 2024-01-15
**Maintained By**: Kubernetes Architecture Study Group
