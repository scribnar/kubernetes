# **Disaster Recovery Strategies - Deep Architectural Analysis**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Target Audience**: Platform engineers, Kubernetes architects, SREs managing production clusters, disaster recovery planning teams

**Scope**: Comprehensive disaster recovery strategies for Kubernetes clusters, covering etcd backup/restore, multi-region failover, RTO/RPO planning, and automated recovery workflows. This document examines DR implementation at the source code level to help engineers build resilient, recoverable Kubernetes platforms.

**Prerequisites**:
- Understanding of [etcd Architecture](../etcd/high-level/01-etcd-architecture.md)
- Familiarity with [High Availability Setup](../lifecycle/04-high-availability-cluster-setup.md)
- Knowledge of [Secrets and Encryption](../security/03-secrets-and-encryption.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Disaster Recovery Fundamentals**

### **Defining Disaster Scenarios**

| **Disaster Type** | **Scope** | **RTO Target** | **RPO Target** | **Recovery Method** |
|-------------------|----------|---------------|---------------|-------------------|
| **Single node failure** | Worker/control plane node | Minutes | 0 (HA handles) | Auto-recovery via HA |
| **etcd corruption** | Control plane data | Hours | Minutes | etcd snapshot restore |
| **Zone failure** | Availability zone outage | Minutes-Hours | Minutes | Multi-zone HA |
| **Region failure** | Entire cloud region | Hours-Days | Minutes-Hours | Multi-region DR |
| **Complete cluster loss** | Total infrastructure failure | Days | Hours | Rebuild from backup |
| **Ransomware/compromise** | Security breach | Days | Hours | Clean rebuild |

### **RTO and RPO Definitions**

**RTO (Recovery Time Objective)**:
- Time from disaster detection to service restoration
- Includes: Detection + Decision + Execution + Verification
- Target: < 4 hours for most production clusters

**RPO (Recovery Point Objective)**:
- Maximum acceptable data loss
- Determined by backup frequency
- Target: < 1 hour for critical workloads

**Example Timeline**:
```
Disaster occurs: T+0
Detection: T+5 min (monitoring alerts)
Decision to restore: T+15 min (assessment)
Restore execution: T+15 min to T+2 hours (depending on cluster size)
Verification: T+2 hours to T+3 hours (testing)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total RTO: 3 hours
RPO: 1 hour (hourly backups)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💾 etcd Backup Strategies**

### **Manual etcd Snapshot**

**One-Time Backup**:
```bash
# Create snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-20240115-120000.db --write-out=table

# Output:
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 1a2b3c4d |   500000 |      15000 |   8.5 GB   |
+----------+----------+------------+------------+
```

### **Automated Backup with CronJob**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  concurrencyPolicy: Forbid  # Don't overlap backups
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeName: control-plane-1  # Pin to specific node with etcd
          containers:
          - name: etcd-backup
            image: registry.k8s.io/etcd:3.5.12-0
            command:
            - /bin/sh
            - -c
            - |
              BACKUP_FILE="/backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db"

              # Create snapshot
              etcdctl snapshot save $BACKUP_FILE \\
                --endpoints=https://127.0.0.1:2379 \\
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
                --cert=/etc/kubernetes/pki/etcd/server.crt \\
                --key=/etc/kubernetes/pki/etcd/server.key

              # Verify
              etcdctl snapshot status $BACKUP_FILE --write-out=table

              # Upload to S3/GCS/Azure Blob
              aws s3 cp $BACKUP_FILE s3://my-k8s-backups/etcd/

              # Cleanup old local backups (keep last 3)
              ls -t /backup/etcd-snapshot-*.db | tail -n +4 | xargs rm -f

            env:
            - name: ETCDCTL_API
              value: "3"
            - name: AWS_REGION
              value: "us-west-2"
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup-dir
              mountPath: /backup
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup-dir
            hostPath:
              path: /var/lib/etcd-backup
              type: DirectoryOrCreate
          restartPolicy: OnFailure
```

### **Backup Retention Policy**

**3-2-1 Rule**:
- **3** copies of data
- **2** different storage media
- **1** offsite backup

**Implementation**:
```
Copy 1: On control plane node (/var/lib/etcd-backup)
Copy 2: In cloud storage (S3, GCS, Azure Blob)
Copy 3: In different region/cloud (multi-region S3, GCS)
```

**Retention Schedule**:
```
Hourly backups: Keep 24 (last 24 hours)
Daily backups: Keep 7 (last week)
Weekly backups: Keep 4 (last month)
Monthly backups: Keep 12 (last year)
```

**Script**:
```bash
#!/bin/bash
# etcd-backup-rotation.sh

BACKUP_DIR="/var/lib/etcd-backup"
S3_BUCKET="s3://my-k8s-backups/etcd"

# Create hourly backup
HOURLY_BACKUP="$BACKUP_DIR/hourly/etcd-$(date +%Y%m%d-%H%M).db"
etcdctl snapshot save $HOURLY_BACKUP ...

# If it's midnight, copy to daily
if [ $(date +%H) -eq 00 ]; then
  cp $HOURLY_BACKUP $BACKUP_DIR/daily/etcd-$(date +%Y%m%d).db
fi

# If it's Sunday midnight, copy to weekly
if [ $(date +%u) -eq 7 ] && [ $(date +%H) -eq 00 ]; then
  cp $HOURLY_BACKUP $BACKUP_DIR/weekly/etcd-$(date +%Y%V).db
fi

# If it's 1st of month, copy to monthly
if [ $(date +%d) -eq 01 ] && [ $(date +%H) -eq 00 ]; then
  cp $HOURLY_BACKUP $BACKUP_DIR/monthly/etcd-$(date +%Y%m).db
fi

# Upload to S3
aws s3 sync $BACKUP_DIR $S3_BUCKET

# Cleanup old backups
find $BACKUP_DIR/hourly -name "etcd-*.db" -mtime +1 -delete
find $BACKUP_DIR/daily -name "etcd-*.db" -mtime +7 -delete
find $BACKUP_DIR/weekly -name "etcd-*.db" -mtime +28 -delete
find $BACKUP_DIR/monthly -name "etcd-*.db" -mtime +365 -delete
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 etcd Restore Procedures**

### **Complete Cluster Restore**

**Scenario**: All control plane nodes lost, restore from backup

**Pre-requisites**:
- Latest etcd snapshot
- Access to control plane nodes (or ability to recreate them)
- Original cluster certificates (or regenerate with kubeadm)

**Procedure**:

**Step 1: Stop all etcd members**
```bash
# On all control plane nodes
systemctl stop etcd
# Or if static pod:
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
```

**Step 2: Remove old data**
```bash
# On all control plane nodes
rm -rf /var/lib/etcd/*
```

**Step 3: Restore snapshot on first node**
```bash
# On control-plane-1
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \\
  --name=control-plane-1 \\
  --initial-cluster=control-plane-1=https://192.168.1.101:2380,control-plane-2=https://192.168.1.102:2380,control-plane-3=https://192.168.1.103:2380 \\
  --initial-cluster-token=etcd-cluster-1 \\
  --initial-advertise-peer-urls=https://192.168.1.101:2380 \\
  --data-dir=/var/lib/etcd-restored
```

**Step 4: Restore on remaining nodes**
```bash
# On control-plane-2
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \\
  --name=control-plane-2 \\
  --initial-cluster=control-plane-1=https://192.168.1.101:2380,control-plane-2=https://192.168.1.102:2380,control-plane-3=https://192.168.1.103:2380 \\
  --initial-cluster-token=etcd-cluster-1 \\
  --initial-advertise-peer-urls=https://192.168.1.102:2380 \\
  --data-dir=/var/lib/etcd-restored

# Repeat for control-plane-3
```

**Step 5: Update etcd configuration**
```bash
# Update data-dir in etcd manifest
sed -i 's|/var/lib/etcd|/var/lib/etcd-restored|' /etc/kubernetes/manifests/etcd.yaml

# Or systemd service:
sed -i 's|--data-dir=/var/lib/etcd|--data-dir=/var/lib/etcd-restored|' /etc/systemd/system/etcd.service
systemctl daemon-reload
```

**Step 6: Start etcd cluster**
```bash
# On all nodes simultaneously (or static pod - restore manifest)
systemctl start etcd

# Verify cluster health
ETCDCTL_API=3 etcdctl member list --write-out=table
ETCDCTL_API=3 etcdctl endpoint health
```

**Step 7: Restart control plane components**
```bash
# API server will automatically reconnect to etcd
# For static pods:
kubectl delete pod kube-apiserver-control-plane-1 -n kube-system
# Or:
systemctl restart kubelet
```

**Step 8: Verification**
```bash
# Check cluster state
kubectl get nodes
kubectl get pods --all-namespaces

# Verify data integrity
kubectl get all --all-namespaces | wc -l  # Should match pre-disaster count

# Check etcd consistency
ETCDCTL_API=3 etcdctl endpoint status --write-out=table
```

### **Single-Member Restore**

**Scenario**: One etcd member corrupted, others healthy

**Procedure**:

**Step 1: Remove corrupted member from cluster**
```bash
# On healthy member
MEMBER_ID=$(ETCDCTL_API=3 etcdctl member list | grep control-plane-2 | cut -d',' -f1)
ETCDCTL_API=3 etcdctl member remove $MEMBER_ID
```

**Step 2: Clean corrupted member**
```bash
# On control-plane-2
systemctl stop etcd
rm -rf /var/lib/etcd/*
```

**Step 3: Add member back to cluster**
```bash
# On healthy member
ETCDCTL_API=3 etcdctl member add control-plane-2 \\
  --peer-urls=https://192.168.1.102:2380
```

**Step 4: Start member with --initial-cluster-state=existing**
```bash
# On control-plane-2
# Update etcd config
etcd \\
  --initial-cluster-state=existing \\
  --initial-cluster=control-plane-1=https://192.168.1.101:2380,control-plane-2=https://192.168.1.102:2380,control-plane-3=https://192.168.1.103:2380 \\
  ...

# Or update static pod manifest
```

**Member syncs data from existing cluster (no snapshot restore needed)**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🗂️ Application State Backup**

### **Velero (Cluster Backup Tool)**

**Installation**:
```bash
# Install Velero CLI
brew install velero

# Install Velero server in cluster
velero install \\
  --provider aws \\
  --plugins velero/velero-plugin-for-aws:v1.9.0 \\
  --bucket my-k8s-backups \\
  --backup-location-config region=us-west-2 \\
  --snapshot-location-config region=us-west-2 \\
  --secret-file ./credentials-velero
```

**Backup Entire Cluster**:
```bash
# Create backup
velero backup create full-cluster-backup \\
  --include-namespaces '*' \\
  --include-cluster-resources=true

# Check status
velero backup describe full-cluster-backup

# Schedule daily backups
velero schedule create daily-backup \\
  --schedule="0 2 * * *" \\
  --include-namespaces '*'
```

**Backup Specific Namespace**:
```bash
velero backup create production-backup \\
  --include-namespaces production \\
  --include-cluster-resources=false
```

**Restore from Backup**:
```bash
# List backups
velero backup get

# Restore entire backup
velero restore create --from-backup full-cluster-backup

# Restore specific namespace
velero restore create --from-backup full-cluster-backup \\
  --include-namespaces production

# Restore with resource mapping (e.g., change namespace)
velero restore create --from-backup full-cluster-backup \\
  --namespace-mappings production:production-restored
```

**What Velero Backs Up**:
- ✅ Kubernetes resources (Deployments, Services, ConfigMaps, Secrets, etc.)
- ✅ Persistent Volume snapshots (cloud provider snapshots)
- ❌ **NOT etcd data** (use separate etcd backup)

### **GitOps as Backup**

**Principle**: All resources defined in Git = infrastructure as code

**Advantages**:
- ✅ Version control
- ✅ Audit trail
- ✅ Easy rollback (git revert)
- ✅ Disaster recovery = git clone + kubectl apply

**Implementation**:
```yaml
# ArgoCD ApplicationSet for all applications
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-applications
spec:
  generators:
  - git:
      repoURL: https://github.com/company/k8s-manifests
      revision: HEAD
      directories:
      - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      source:
        repoURL: https://github.com/company/k8s-manifests
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**Disaster Recovery**:
```bash
# 1. Restore etcd (cluster state)
# 2. Reinstall ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Apply ApplicationSet
kubectl apply -f applicationset.yaml

# ArgoCD automatically recreates all applications from Git
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌍 Multi-Region Disaster Recovery**

### **Active-Passive DR**

**Architecture**:
```
Primary Region (us-west-2):
  - Full production cluster (5,000 nodes)
  - Active workloads
  - Continuous etcd backups to S3

DR Region (us-east-1):
  - Standby cluster (minimal or no resources)
  - Receives etcd backups from primary
  - Ready to scale up on failover
```

**Failover Procedure**:

**Step 1: Detect primary region failure**
```bash
# Monitoring detects region outage
# Alert triggers DR runbook
```

**Step 2: Restore etcd in DR cluster**
```bash
# Download latest backup from S3
aws s3 cp s3://my-k8s-backups/etcd/latest.db /backup/

# Restore etcd in DR cluster
# (Follow etcd restore procedure from above)
```

**Step 3: Update DNS**
```bash
# Point application DNS to DR region
aws route53 change-resource-record-sets \\
  --hosted-zone-id Z123456 \\
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "DR_LB_IP"}]
      }
    }]
  }'
```

**Step 4: Scale up DR cluster**
```bash
# Increase node count
eksctl scale nodegroup --cluster=dr-cluster --nodes=5000 --name=workers

# Or autoscaling group:
aws autoscaling set-desired-capacity \\
  --auto-scaling-group-name dr-cluster-workers \\
  --desired-capacity 5000
```

**Step 5: Verify applications**
```bash
# Check pod status
kubectl get pods --all-namespaces

# Run smoke tests
kubectl run test-pod --image=busybox --restart=Never -- wget -O- http://api.example.com/health
```

**RTO**: 1-4 hours (depending on cluster size and automation)
**RPO**: Last backup (e.g., 1 hour if hourly backups)

### **Active-Active Multi-Cluster**

**Architecture**:
```
Region 1 (us-west-2):
  - Cluster A (5,000 nodes)
  - Handles 50% of traffic

Region 2 (us-east-1):
  - Cluster B (5,000 nodes)
  - Handles 50% of traffic

Load Balancer:
  - Global load balancer (AWS Global Accelerator, GCP Cloud Load Balancing)
  - Health checks both clusters
  - Automatic failover if one region fails
```

**Advantages**:
- ✅ Zero RTO (automatic failover)
- ✅ Zero RPO (active-active, no data loss)
- ✅ Geographic distribution (lower latency)

**Challenges**:
- ❌ Cost (2x infrastructure)
- ❌ Data synchronization (requires application-level replication)
- ❌ Cross-region networking complexity

**Implementation**:
```yaml
# Application with active-active support
apiVersion: v1
kind: Service
metadata:
  name: api-service
  annotations:
    # Multi-cluster service
    service.kubernetes.io/topology-aware-hints: auto
spec:
  type: LoadBalancer
  selector:
    app: api
```

**Data Replication**: Application responsibility (not Kubernetes)
- Databases: Multi-region replication (RDS Multi-AZ, Cloud Spanner, CockroachDB)
- Object storage: Cross-region replication (S3, GCS)
- Caching: Distributed cache (Redis Cluster, Memcached)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🧪 DR Testing**

### **DR Drill Checklist**

**Monthly Drills** (recommended):

```markdown
## DR Drill Checklist

### Pre-Drill
- [ ] Announce drill to team (no surprises)
- [ ] Verify backups are up-to-date
- [ ] Review runbooks
- [ ] Prepare monitoring dashboards

### Drill Execution
- [ ] T+0: Simulate disaster (e.g., shut down primary cluster)
- [ ] T+5: Detect outage (verify monitoring alerts)
- [ ] T+10: Initiate DR procedure
- [ ] T+15-120: Execute restore (etcd + applications)
- [ ] T+120-180: Verify applications operational

### Post-Drill
- [ ] Document actual RTO/RPO achieved
- [ ] Identify gaps in runbooks
- [ ] Update procedures based on findings
- [ ] Schedule fixes for identified issues
```

**Chaos Engineering**:
```yaml
# Chaos Mesh experiment - kill random control plane node
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-control-plane
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - kube-system
    labelSelectors:
      component: kube-apiserver
  scheduler:
    cron: '@weekly'  # Weekly test
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Related Documentation**

### **Backup and Recovery**
- **[etcd Backup and Restore](../etcd/middle-level/06-backup-restore.md)** - Detailed etcd procedures
- **[Secrets Rotation](../security/04-secrets-rotation.md)** - Certificate backup
- **[Cluster Backup Restore](../lifecycle/06-cluster-backup-restore.md)** - Complete cluster backup

### **High Availability**
- **[High Availability Setup](../lifecycle/04-high-availability-cluster-setup.md)** - Multi-master patterns
- **[Large Cluster Architecture](./01-large-cluster-architecture.md)** - Scaling for DR
- **[Performance Tuning](../etcd/middle-level/07-performance-tuning.md)** - etcd optimization

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Key Takeaways**

### **For Platform Engineers**

1. **Backups are Mandatory**:
   - Automate etcd backups (hourly minimum)
   - Store offsite (S3, GCS, Azure Blob)
   - Test restores regularly (monthly drills)

2. **RTO/RPO Targets**:
   - Production: RTO < 4 hours, RPO < 1 hour
   - Critical: RTO < 1 hour, RPO < 15 minutes (active-active)

3. **Backup Multiple Layers**:
   - etcd (cluster state)
   - Application manifests (GitOps)
   - Persistent volume snapshots (Velero)

4. **DR is Not Just Backup**:
   - Documented procedures
   - Tested runbooks
   - Automated recovery where possible

5. **Multi-Region for Critical Workloads**:
   - Active-passive: Lower cost, higher RTO
   - Active-active: Higher cost, zero RTO

### **For Kubernetes Contributors**

1. **etcd Backup Implementation**:
   - Snapshot API: `etcdserver/api/v3snapshot/`
   - Restore logic: `etcdserver/api/snap/snapshotter.go`

2. **Testing Restore**:
   - Integration tests: `test/integration/etcd/`
   - E2E tests for backup/restore scenarios

3. **Metrics**:
   - etcd backup duration
   - Snapshot size over time
   - Restore time benchmarks

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Kubernetes Version**: v1.30
**Last Updated**: 2024-01-15
**Maintained By**: Kubernetes Architecture Study Group
