# **Kubernetes Cluster Backup and Restore**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Document Overview**

**Target Audience**: Platform engineers, SREs, and operators responsible for disaster recovery and business continuity

**Purpose**: This document provides comprehensive guidance on backing up and restoring Kubernetes clusters, including etcd backup strategies, resource manifests backup, disaster recovery procedures, and compliance requirements.

**Scope**:
- Complete cluster backup strategy
- etcd backup automation and verification
- Resource manifests backup (GitOps, Velero)
- Persistent volume backup and restore
- Disaster recovery procedures
- Point-in-time recovery
- Compliance and retention policies
- Production troubleshooting and testing

**Related Documentation**:
- [High Availability Cluster Setup](04-high-availability-cluster-setup.md) - HA principles
- [Node Maintenance Operations](05-node-maintenance-operations.md) - Safe operations
- [Kubeadm Upgrade Strategies](02-kubeadm-upgrade-strategies.md) - Control plane management

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Backup Strategy Overview**

### **What to Back Up**

Kubernetes clusters have multiple backup targets:

```
┌─────────────────────────────────────────────────────┐
│              Kubernetes Cluster                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. ┌──────────────────┐  Cluster State            │
│     │  etcd Database   │  (All K8s objects)         │
│     └──────────────────┘  Critical: MUST backup     │
│                                                     │
│  2. ┌──────────────────┐  Application Definitions  │
│     │  Resource        │  (Deployments, Services)   │
│     │  Manifests       │  Important: Should backup  │
│     └──────────────────┘                            │
│                                                     │
│  3. ┌──────────────────┐  Application Data          │
│     │  Persistent      │  (Databases, files)        │
│     │  Volumes         │  Critical: MUST backup     │
│     └──────────────────┘                            │
│                                                     │
│  4. ┌──────────────────┐  Cluster Configuration    │
│     │  Certificates    │  (TLS certs, kubeconfigs)  │
│     │  & Secrets       │  Important: Should backup  │
│     └──────────────────┘                            │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Backup Priorities**:

| **Component** | **Priority** | **Frequency** | **RPO** | **RTO** |
|---------------|--------------|---------------|---------|---------|
| **etcd** | Critical | Every 5-15 min | 15 min | 30 min |
| **Persistent Volumes** | Critical | Daily | 24 hours | 2 hours |
| **Resource Manifests** | Important | On change (GitOps) | Real-time | 1 hour |
| **Certificates** | Important | Weekly | 7 days | 4 hours |

**RPO (Recovery Point Objective)**: Maximum acceptable data loss
**RTO (Recovery Time Objective)**: Maximum acceptable downtime

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🗄️ etcd Backup**

### **Why etcd Backup is Critical**

etcd stores ALL Kubernetes cluster state:
- All API objects (Pods, Services, Deployments, etc.)
- Secrets and ConfigMaps
- RBAC policies
- Custom Resource Definitions
- Everything visible via `kubectl get all --all-namespaces`

**Without etcd backup**: Complete cluster rebuild required (days of work)
**With etcd backup**: Cluster restored in minutes

### **Manual etcd Backup**

```bash
#!/bin/bash
# manual-etcd-backup.sh

BACKUP_DIR="/var/backups/etcd"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/etcd-snapshot-$TIMESTAMP.db"

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup etcd using etcdctl
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save $BACKUP_FILE

# Verify backup integrity
ETCDCTL_API=3 etcdctl \
  --write-out=table \
  snapshot status $BACKUP_FILE

# Output:
# +---------+----------+------------+------------+
# |  HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +---------+----------+------------+------------+
# | a1b2c3d | 12345678 |     150000 |     2.3 GB |
# +---------+----------+------------+------------+

# Compress backup (optional but recommended)
gzip $BACKUP_FILE

# Upload to remote storage
aws s3 cp ${BACKUP_FILE}.gz \
  s3://my-cluster-backups/etcd/$(basename ${BACKUP_FILE}.gz)

# Retention: Delete backups older than 30 days
find $BACKUP_DIR -name "etcd-snapshot-*.db.gz" -mtime +30 -delete

echo "Backup completed: ${BACKUP_FILE}.gz"
```

### **Automated etcd Backup CronJob**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  # Run every 5 minutes
  schedule: "*/5 * * * *"

  # Keep last 3 completed jobs
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1

  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: etcd-backup
        spec:
          # Run on master node where etcd lives
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""

          # Use host network to access etcd
          hostNetwork: true

          # Don't restart on failure
          restartPolicy: Never

          containers:
          - name: backup
            image: k8s.gcr.io/etcd:3.5.9-0
            command:
            - /bin/sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="/backup/etcd-snapshot-$TIMESTAMP.db"

              # Create backup
              etcdctl \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key \
                snapshot save $BACKUP_FILE

              # Verify backup
              etcdctl snapshot status $BACKUP_FILE

              # Compress
              gzip $BACKUP_FILE

              # Upload to S3 (requires AWS credentials)
              aws s3 cp ${BACKUP_FILE}.gz \
                s3://my-cluster-backups/etcd/$(basename ${BACKUP_FILE}.gz)

              # Cleanup old local backups (keep last 24 hours)
              find /backup -name "etcd-snapshot-*.db.gz" -mtime +1 -delete

              echo "Backup completed successfully"

            volumeMounts:
            # Mount etcd certificates
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true

            # Mount backup storage
            - name: backup-storage
              mountPath: /backup

            env:
            # AWS credentials for S3 upload
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: etcd-backup-aws-creds
                  key: access-key-id
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: etcd-backup-aws-creds
                  key: secret-access-key
            - name: AWS_DEFAULT_REGION
              value: "us-east-1"

          volumes:
          # etcd certificates from host
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
              type: Directory

          # Local backup storage
          - name: backup-storage
            hostPath:
              path: /var/backups/etcd
              type: DirectoryOrCreate

---
# AWS credentials secret
apiVersion: v1
kind: Secret
metadata:
  name: etcd-backup-aws-creds
  namespace: kube-system
type: Opaque
data:
  access-key-id: <base64-encoded-access-key>
  secret-access-key: <base64-encoded-secret-key>
```

### **etcd Backup Verification**

Always verify backups can be restored:

```bash
#!/bin/bash
# verify-etcd-backup.sh

BACKUP_FILE="/var/backups/etcd/etcd-snapshot-20241117-143000.db"

# 1. Check backup file integrity
etcdctl snapshot status $BACKUP_FILE --write-out=table

# 2. Restore to temporary directory (doesn't affect running cluster)
TEMP_DIR="/tmp/etcd-restore-test-$$"
mkdir -p $TEMP_DIR

etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=$TEMP_DIR \
  --name=test-etcd \
  --initial-cluster=test-etcd=http://localhost:2380 \
  --initial-advertise-peer-urls=http://localhost:2380

# 3. Verify restored data directory
if [ -d "$TEMP_DIR/member" ]; then
  echo "✓ Backup is valid and restorable"
  du -sh $TEMP_DIR
else
  echo "✗ Backup restore failed"
  exit 1
fi

# 4. Cleanup
rm -rf $TEMP_DIR

echo "Backup verification complete"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **♻️ etcd Restore**

### **Complete Cluster Restore Procedure**

**Scenario**: Complete cluster failure, need to restore from etcd backup

```bash
#!/bin/bash
# restore-cluster-from-etcd.sh

# WARNING: This stops all control plane components
# Only use during disaster recovery

BACKUP_FILE="/var/backups/etcd/etcd-snapshot-20241117-143000.db"
RESTORE_DIR="/var/lib/etcd-restore"

echo "=== KUBERNETES CLUSTER RESTORE ==="
echo "Backup file: $BACKUP_FILE"
echo "WARNING: This will stop the cluster temporarily"
read -p "Continue? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
  echo "Aborted"
  exit 1
fi

# Step 1: Stop control plane components
echo "Step 1: Stopping control plane..."
systemctl stop kube-apiserver
systemctl stop kube-controller-manager
systemctl stop kube-scheduler

# Step 2: Stop etcd
systemctl stop etcd

# Step 3: Backup current etcd data (just in case)
mv /var/lib/etcd /var/lib/etcd-backup-$(date +%Y%m%d-%H%M%S)

# Step 4: Restore from backup
echo "Step 2: Restoring from backup..."

# For single-node etcd
etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=$RESTORE_DIR \
  --name=etcd-1 \
  --initial-cluster=etcd-1=https://10.0.1.10:2380 \
  --initial-advertise-peer-urls=https://10.0.1.10:2380 \
  --initial-cluster-token=etcd-cluster-1

# Move restored data to etcd data directory
mv $RESTORE_DIR /var/lib/etcd

# Fix permissions
chown -R etcd:etcd /var/lib/etcd

# Step 5: Start etcd
echo "Step 3: Starting etcd..."
systemctl start etcd

# Wait for etcd to be healthy
sleep 10
etcdctl endpoint health

# Step 6: Start control plane components
echo "Step 4: Starting control plane..."
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler

# Step 7: Verify cluster is healthy
echo "Step 5: Verifying cluster health..."
sleep 30

kubectl get nodes
kubectl get pods --all-namespaces

echo "=== RESTORE COMPLETE ==="
echo "Check cluster carefully before resuming production traffic"
```

### **Multi-Node etcd Cluster Restore**

For HA clusters with 3-node etcd:

```bash
#!/bin/bash
# restore-ha-etcd-cluster.sh

BACKUP_FILE="/var/backups/etcd/etcd-snapshot-20241117-143000.db"

# Execute on EACH etcd node with appropriate values

# Node 1:
etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=/var/lib/etcd-restore \
  --name=etcd-1 \
  --initial-cluster=etcd-1=https://10.0.1.10:2380,etcd-2=https://10.0.1.11:2380,etcd-3=https://10.0.1.12:2380 \
  --initial-advertise-peer-urls=https://10.0.1.10:2380 \
  --initial-cluster-token=etcd-cluster-1

# Node 2:
etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=/var/lib/etcd-restore \
  --name=etcd-2 \
  --initial-cluster=etcd-1=https://10.0.1.10:2380,etcd-2=https://10.0.1.11:2380,etcd-3=https://10.0.1.12:2380 \
  --initial-advertise-peer-urls=https://10.0.1.11:2380 \
  --initial-cluster-token=etcd-cluster-1

# Node 3:
etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=/var/lib/etcd-restore \
  --name=etcd-3 \
  --initial-cluster=etcd-1=https://10.0.1.10:2380,etcd-2=https://10.0.1.11:2380,etcd-3=https://10.0.1.12:2380 \
  --initial-advertise-peer-urls=https://10.0.1.12:2380 \
  --initial-cluster-token=etcd-cluster-1

# Important: All nodes must restore from SAME backup file
# This ensures cluster consistency
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📦 Resource Manifests Backup**

### **GitOps Approach (Recommended)**

Store all Kubernetes manifests in Git:

```bash
# Directory structure
kubernetes-manifests/
├── namespaces/
│   ├── production.yaml
│   └── staging.yaml
├── deployments/
│   ├── frontend-deployment.yaml
│   └── backend-deployment.yaml
├── services/
│   ├── frontend-service.yaml
│   └── backend-service.yaml
├── configmaps/
│   └── app-config.yaml
├── secrets/
│   └── sealed-secrets.yaml  # Use Sealed Secrets, not plain secrets!
└── helm-releases/
    ├── mysql.yaml
    └── redis.yaml

# Benefits:
#   ✅ Version control (full history)
#   ✅ Code review process
#   ✅ Automated deployment (ArgoCD, Flux)
#   ✅ Declarative configuration
#   ✅ Disaster recovery (git clone + kubectl apply)
```

**Automated Backup of Live Resources**:

```bash
#!/bin/bash
# backup-k8s-resources.sh

BACKUP_DIR="/var/backups/kubernetes/resources"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
EXPORT_DIR="$BACKUP_DIR/export-$TIMESTAMP"

mkdir -p $EXPORT_DIR

echo "Backing up Kubernetes resources..."

# Backup all namespaces
kubectl get namespaces -o yaml > $EXPORT_DIR/namespaces.yaml

# Backup resources per namespace
for NAMESPACE in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  NS_DIR="$EXPORT_DIR/namespaces/$NAMESPACE"
  mkdir -p $NS_DIR

  # Backup common resources
  for RESOURCE in deployments statefulsets daemonsets services configmaps secrets ingresses; do
    if kubectl get $RESOURCE -n $NAMESPACE &>/dev/null; then
      kubectl get $RESOURCE -n $NAMESPACE -o yaml > $NS_DIR/$RESOURCE.yaml
    fi
  done
done

# Backup cluster-wide resources
mkdir -p $EXPORT_DIR/cluster
kubectl get clusterroles -o yaml > $EXPORT_DIR/cluster/clusterroles.yaml
kubectl get clusterrolebindings -o yaml > $EXPORT_DIR/cluster/clusterrolebindings.yaml
kubectl get persistentvolumes -o yaml > $EXPORT_DIR/cluster/persistentvolumes.yaml
kubectl get storageclasses -o yaml > $EXPORT_DIR/cluster/storageclasses.yaml

# Create tarball
cd $BACKUP_DIR
tar czf export-$TIMESTAMP.tar.gz export-$TIMESTAMP/

# Upload to S3
aws s3 cp export-$TIMESTAMP.tar.gz s3://my-cluster-backups/resources/

# Cleanup
rm -rf export-$TIMESTAMP/
find $BACKUP_DIR -name "export-*.tar.gz" -mtime +7 -delete

echo "Resource backup complete: export-$TIMESTAMP.tar.gz"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💾 Persistent Volume Backup with Velero**

### **Velero Architecture**

Velero provides cluster resource and persistent volume backups:

```
┌────────────────────────────────────────┐
│         Kubernetes Cluster             │
├────────────────────────────────────────┤
│                                        │
│  ┌──────────────┐   ┌──────────────┐  │
│  │   Velero     │   │    Velero    │  │
│  │   Server     │   │   Plugin     │  │
│  └──────┬───────┘   └──────┬───────┘  │
│         │                  │           │
│         ↓                  ↓           │
│  ┌──────────────────────────────────┐  │
│  │     Resources + PV Snapshots     │  │
│  └──────────────┬───────────────────┘  │
└─────────────────┼──────────────────────┘
                  │
                  ↓
        ┌─────────────────────┐
        │  Object Storage     │
        │  (S3, GCS, Azure)   │
        └─────────────────────┘
```

### **Installing Velero**

```bash
# Install Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/

# AWS S3 Setup
# Create S3 bucket for backups
aws s3 mb s3://my-cluster-velero-backups

# Create IAM user with S3 permissions
cat > velero-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": [
        "arn:aws:s3:::my-cluster-velero-backups/*",
        "arn:aws:s3:::my-cluster-velero-backups"
      ]
    }
  ]
}
EOF

aws iam create-policy --policy-name VeleroBackupPolicy --policy-document file://velero-policy.json

# Create credentials file
cat > credentials-velero <<EOF
[default]
aws_access_key_id=<ACCESS_KEY_ID>
aws_secret_access_key=<SECRET_ACCESS_KEY>
EOF

# Install Velero in cluster
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-cluster-velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero

# Verify installation
kubectl get pods -n velero
```

### **Creating Velero Backups**

```bash
# Full cluster backup
velero backup create full-backup-$(date +%Y%m%d-%H%M%S) \
  --include-namespaces '*'

# Backup specific namespace
velero backup create app-backup \
  --include-namespaces production

# Backup with persistent volumes
velero backup create pv-backup \
  --include-namespaces production \
  --snapshot-volumes

# Exclude specific resources
velero backup create selective-backup \
  --include-namespaces production \
  --exclude-resources secrets,configmaps

# Check backup status
velero backup describe full-backup-20241117-143000

# List all backups
velero backup get
```

### **Scheduled Backups**

```bash
# Create schedule for daily backups at 2 AM
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces '*'

# Create schedule for hourly production backups
velero schedule create production-hourly \
  --schedule="0 * * * *" \
  --include-namespaces production

# List schedules
velero schedule get

# Delete schedule
velero schedule delete daily-backup
```

### **Restoring from Velero Backup**

```bash
# List available backups
velero backup get

# Restore entire cluster from backup
velero restore create --from-backup full-backup-20241117-143000

# Restore specific namespace
velero restore create production-restore \
  --from-backup full-backup-20241117-143000 \
  --include-namespaces production

# Restore to different namespace
velero restore create test-restore \
  --from-backup full-backup-20241117-143000 \
  --namespace-mappings production:test

# Check restore status
velero restore describe production-restore

# Watch restore progress
velero restore logs production-restore -f
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚨 Disaster Recovery Procedures**

### **Scenario 1: Complete Cluster Loss**

**Recovery Steps**:

```bash
# 1. Provision new cluster infrastructure
#    - New nodes
#    - Network configuration
#    - Load balancers

# 2. Install Kubernetes control plane
kubeadm init --config=cluster-config.yaml

# 3. Restore etcd from backup
systemctl stop etcd
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd

# 4. Start control plane
systemctl start etcd
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler

# 5. Join worker nodes
kubeadm join <API_SERVER>:6443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH>

# 6. Verify cluster is operational
kubectl get nodes
kubectl get pods --all-namespaces

# 7. Restore persistent volumes (if needed)
velero restore create full-restore --from-backup <backup-name>

# Expected RTO: 2-4 hours for complete cluster rebuild
```

### **Scenario 2: Single etcd Member Failure**

**Recovery Steps**:

```bash
# 1. Remove failed member from cluster
etcdctl member list
# Find member ID of failed node

etcdctl member remove <MEMBER_ID>

# 2. Provision new etcd node

# 3. Add new member to cluster
etcdctl member add etcd-new \
  --peer-urls=https://10.0.1.13:2380

# 4. Start new etcd member
# It will sync data from existing members automatically

# 5. Verify cluster health
etcdctl endpoint health --cluster

# Expected RTO: 15-30 minutes
```

### **Scenario 3: Accidental Resource Deletion**

```bash
# 1. Check if deleted resources are in recent backup
velero backup get

# 2. Restore specific resources
velero restore create accidental-delete-restore \
  --from-backup recent-backup \
  --include-resources deployments,services \
  --selector app=myapp

# 3. Verify restoration
kubectl get deployments -l app=myapp

# Expected RTO: 5-15 minutes
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Compliance and Retention Policies**

### **Backup Retention Strategy**

```yaml
# Backup retention policy example
retention:
  # etcd backups
  etcd:
    frequency: "Every 5 minutes"
    retention:
      - last_24_hours: "All backups (288 backups)"
      - last_7_days: "Hourly backups (168 backups)"
      - last_30_days: "Daily backups (30 backups)"
      - last_365_days: "Weekly backups (52 backups)"

  # Velero backups
  velero:
    frequency: "Daily"
    retention:
      - last_7_days: "All daily backups"
      - last_90_days: "Weekly backups"
      - last_365_days: "Monthly backups"

  # Compliance requirements
  compliance:
    - GDPR: "30 days minimum retention"
    - SOC2: "90 days minimum retention"
    - HIPAA: "6 years minimum retention"
    - PCI-DSS: "3 months minimum retention"
```

**Implementing Retention Policy**:

```bash
#!/bin/bash
# cleanup-old-backups.sh

S3_BUCKET="s3://my-cluster-backups"

# Delete etcd backups older than 365 days
aws s3 ls $S3_BUCKET/etcd/ | while read -r line; do
  CREATE_DATE=$(echo $line | awk '{print $1" "$2}')
  CREATE_DATE_SECONDS=$(date -d "$CREATE_DATE" +%s)
  NOW_SECONDS=$(date +%s)
  AGE_DAYS=$(( ($NOW_SECONDS - $CREATE_DATE_SECONDS) / 86400 ))

  if [ $AGE_DAYS -gt 365 ]; then
    FILE=$(echo $line | awk '{print $4}')
    echo "Deleting $FILE (age: $AGE_DAYS days)"
    aws s3 rm $S3_BUCKET/etcd/$FILE
  fi
done

# Velero has built-in TTL
velero backup create my-backup --ttl 720h  # 30 days
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Summary and Best Practices**

### **Backup Best Practices**

✅ **Do**:
- Automate all backups (CronJobs, schedules)
- Test restore procedures regularly (monthly)
- Store backups in different region/availability zone
- Encrypt backups at rest and in transit
- Monitor backup jobs for failures
- Document recovery procedures
- Practice disaster recovery drills

❌ **Don't**:
- Store backups only locally (single point of failure)
- Skip backup verification
- Assume backups work without testing
- Ignore backup failures
- Store secrets in plain text in backups

### **Testing Backup/Restore**

```bash
#!/bin/bash
# monthly-dr-test.sh

echo "=== DISASTER RECOVERY TEST ==="
echo "Creating test cluster..."

# 1. Create backup of current cluster
velero backup create dr-test-$(date +%Y%m%d)

# 2. Create temporary test cluster
kind create cluster --name dr-test

# 3. Install Velero on test cluster
velero install --provider aws ...

# 4. Restore backup to test cluster
velero restore create dr-test-restore \
  --from-backup dr-test-$(date +%Y%m%d)

# 5. Verify all resources restored
kubectl get all --all-namespaces

# 6. Test application functionality
./test-apps.sh

# 7. Cleanup
kind delete cluster --name dr-test

echo "DR test complete"
```

### **Related Documentation**

- [High Availability Setup](04-high-availability-cluster-setup.md) - HA principles
- [Node Maintenance](05-node-maintenance-operations.md) - Safe operations
- [Disaster Recovery Strategies](../scalability/04-disaster-recovery-strategies.md) - DR patterns

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Metadata**:
- **Lines**: ~2,200
- **Code Examples**: 30+
- **Target Audience**: Platform engineers, SREs, operators
- **Last Updated**: 2024-11-17
