# **Secrets Rotation - Deep Architectural Analysis**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Target Audience**: Platform engineers, Kubernetes architects, security engineers, SREs managing production clusters, compliance teams

**Scope**: Deep architectural analysis of secret rotation strategies, certificate renewal automation, zero-downtime rotation patterns, and compliance-driven rotation policies. This document examines rotation mechanisms at the source code level to help platform engineers implement secure, automated secret lifecycle management.

**Prerequisites**:
- Understanding of [Secrets and Encryption](./03-secrets-and-encryption.md)
- Familiarity with [Certificate Management](../controller-manager/17-certificate-controllers.md)
- Knowledge of [RBAC patterns](./05-rbac-patterns-troubleshooting.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Design Philosophy**

### **Why Rotation Matters**

Secret rotation is critical for:
- **Compliance**: PCI-DSS, HIPAA, SOC 2 require periodic credential rotation
- **Security**: Limit blast radius of compromised credentials
- **Zero trust**: Assume credentials may be compromised, rotate proactively
- **Audit trail**: Track when and why secrets changed

**Rotation Challenges**:
1. **Distributed systems**: Multiple pods using same secret
2. **Zero downtime**: Applications must continue running
3. **Two-phase rotation**: Old and new credentials must coexist
4. **Coordination**: Database, application, and Kubernetes must align
5. **Rollback**: Ability to revert if rotation fails

### **Core Rotation Principles**

```
┌──────────────────────────────────────────────────────────────┐
│  SECRETS ROTATION DESIGN PRINCIPLES                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. ZERO-DOWNTIME ROTATION                                   │
│     └─ Applications remain available during rotation        │
│                                                               │
│  2. TWO-PHASE COMMIT                                         │
│     └─ New secret deployed before old secret revoked        │
│                                                               │
│  3. AUTOMATED ROTATION                                       │
│     └─ Manual rotation error-prone, automate when possible  │
│                                                               │
│  4. GRADUAL ROLLOUT                                          │
│     └─ Canary rotation before full deployment               │
│                                                               │
│  5. AUDIT AND VERIFICATION                                   │
│     └─ Log all rotations, verify success                    │
│                                                               │
│  6. ROLLBACK CAPABILITY                                      │
│     └─ Ability to revert to previous secret                 │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Certificate Rotation**

### **Automatic Certificate Renewal**

**Kubernetes certificates have 1-year validity by default**

```bash
# Check certificate expiration
kubeadm certs check-expiration

# Output:
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jan 15, 2025 10:00 UTC   364d            ca                      no
apiserver                  Jan 15, 2025 10:00 UTC   364d            ca                      no
apiserver-kubelet-client   Jan 15, 2025 10:00 UTC   364d            ca                      no
```

### **Automated Renewal with kubeadm**

**kubelet Auto-Renews Its Own Certificate**:

```go
// pkg/kubelet/certificate/bootstrap/bootstrap.go

func (m *Manager) Start() {
    go wait.Until(func() {
        // Check certificate expiration
        if m.shouldRotate() {
            // Request new certificate
            newCert, err := m.rotateCertificate()
            if err != nil {
                klog.Errorf("Failed to rotate certificate: %v", err)
                return
            }

            // Write new certificate
            m.updateCertificate(newCert)
        }
    }, m.rotationCheckInterval, m.stopCh)
}

func (m *Manager) shouldRotate() bool {
    cert, err := m.getCurrentCertificate()
    if err != nil {
        return true  // No valid cert, request new one
    }

    // Rotate when 80% of certificate lifetime has passed
    threshold := cert.NotAfter.Sub(cert.NotBefore) * 80 / 100
    elapsed := time.Since(cert.NotBefore)

    return elapsed > threshold
}
```

**Automatic Rotation Timeline**:
- Certificate issued: Jan 1, 2024 (valid 1 year)
- 80% lifetime: ~292 days (Oct 20, 2024)
- **kubelet automatically requests renewal**: Oct 20, 2024
- New certificate issued: Oct 20, 2024 (valid 1 year until Oct 20, 2025)
- **No manual intervention needed**

### **Manual Certificate Renewal**

**Renew All Certificates**:
```bash
# Renew all kubeadm-managed certificates
sudo kubeadm certs renew all

# Output:
certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for serving the Kubernetes API renewed
certificate the apiserver uses to access etcd renewed
certificate for the API server to connect to kubelet renewed
certificate embedded in the kubeconfig file for the controller manager to use renewed
certificate for liveness probes to healthcheck etcd renewed
certificate for etcd nodes to communicate with each other renewed
certificate for serving etcd renewed
certificate for the front proxy client renewed
certificate embedded in the kubeconfig file for the scheduler manager to use renewed
```

**Restart Control Plane Components**:
```bash
# kubelet automatically restarts static pods when manifests change
# Force restart by moving and restoring manifests
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 10
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Or restart kubelet (will restart all static pods)
sudo systemctl restart kubelet
```

### **Certificate Rotation for Applications**

**Scenario**: Application using TLS certificate stored in Secret

**Two-Phase Rotation**:

```yaml
# Phase 1: Create new secret with new certificate
apiVersion: v1
kind: Secret
metadata:
  name: app-tls-v2  # New secret name
type: kubernetes.io/tls
data:
  tls.crt: <new-cert-base64>
  tls.key: <new-key-base64>
---
# Phase 2: Update Deployment to use new secret (rolling update)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      volumes:
      - name: tls
        secret:
          secretName: app-tls-v2  # Changed from app-tls-v1
```

**Zero-Downtime Process**:
1. Generate new certificate
2. Create new Secret (app-tls-v2)
3. Update Deployment to reference new Secret
4. Rolling update replaces pods one at a time
5. Verify all pods using new certificate
6. Delete old Secret (app-tls-v1)

**Alternative: In-Place Update** (requires pod restart):
```bash
# Update existing secret
kubectl create secret tls app-tls \
  --cert=new.crt --key=new.key \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart pods to pick up new secret
kubectl rollout restart deployment/myapp
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔑 Database Credentials Rotation**

### **The Dual-Password Pattern**

**Challenge**: Database accepts only one password at a time

**Solution**: Two-phase rotation with temporary dual-password support

```
┌──────────────────────────────────────────────────────────────┐
│  DUAL-PASSWORD ROTATION TIMELINE                              │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  T+0: Initial State                                          │
│       Database: password1                                     │
│       Pods: password1                                         │
│                                                               │
│  T+1: Add new password to database                           │
│       Database: password1, password2 (both valid)            │
│       Pods: password1 (still using old)                       │
│                                                               │
│  T+2: Update Kubernetes Secret                               │
│       Secret: password2                                       │
│                                                               │
│  T+3: Rolling pod restart                                    │
│       Pods: password1 → password2 (gradual transition)       │
│       Database: password1, password2 (both work)             │
│                                                               │
│  T+4: Verify all pods using password2                        │
│       Pods: password2                                         │
│       Database: password1 (unused), password2                │
│                                                               │
│  T+5: Remove old password from database                      │
│       Database: password2 only                                │
│       Pods: password2                                         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### **Implementation: PostgreSQL Rotation**

**Step 1: Generate New Password**:
```bash
# Generate strong password
NEW_PASSWORD=$(openssl rand -base64 32)
```

**Step 2: Add New Password to Database**:
```sql
-- Connect with current admin credentials
-- Add new password for application user (dual-password capability)

-- Option A: Create second user (PostgreSQL)
CREATE USER myapp_v2 WITH PASSWORD 'new-password-here';
GRANT ALL PRIVILEGES ON DATABASE mydb TO myapp_v2;

-- Option B: Use ALTER USER (some databases support multiple passwords)
-- This is database-specific
```

**Step 3: Update Kubernetes Secret**:
```bash
# Create new secret version
kubectl create secret generic db-credentials \
  --from-literal=username=myapp_v2 \
  --from-literal=password=$NEW_PASSWORD \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Step 4: Rolling Restart**:
```bash
# Deployment will automatically pick up new secret on restart
kubectl rollout restart deployment/myapp

# Monitor rollout
kubectl rollout status deployment/myapp
```

**Step 5: Verify and Cleanup**:
```bash
# Verify all pods using new credentials
kubectl logs -l app=myapp | grep "database connection"

# Cleanup old user
# SQL:
DROP USER myapp;  -- Old user
ALTER USER myapp_v2 RENAME TO myapp;  -- Rename new user to original
```

### **Automated Rotation with Reloader**

**Reloader**: Automatically restarts pods when Secrets/ConfigMaps change

**Installation**:
```bash
kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml
```

**Usage**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    reloader.stakater.com/auto: "true"  # Auto-restart on any Secret/ConfigMap change
    # Or specific:
    # reloader.stakater.com/search: "true"
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

**Workflow**:
1. Update Secret: `kubectl apply -f db-credentials.yaml`
2. Reloader detects change
3. Reloader triggers rolling restart: `kubectl rollout restart deployment/myapp`
4. Pods pick up new credentials

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔐 External Secrets Operator Rotation**

### **Automatic Sync from External Source**

**External Secrets Operator** automatically syncs secrets from external providers

**Configuration**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h  # Check for updates every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
    deletionPolicy: Retain
  data:
  - secretKey: password
    remoteRef:
      key: /production/database/password
      version: latest  # Always fetch latest version
```

**Rotation Workflow**:
1. **Rotate in external system** (AWS Secrets Manager, Vault, etc.)
2. **External Secrets Operator detects change** (next sync interval)
3. **Kubernetes Secret updated automatically**
4. **Reloader restarts pods** (if configured)
5. **Pods pick up new credentials**

**Advantage**: Centralized rotation, single source of truth

### **AWS Secrets Manager Automatic Rotation**

**Enable Rotation**:
```bash
aws secretsmanager rotate-secret \
  --secret-id /production/database/password \
  --rotation-lambda-arn arn:aws:lambda:us-west-2:123456789012:function:SecretsManagerRotation \
  --rotation-rules AutomaticallyAfterDays=30
```

**Rotation Lambda** (simplified):
```python
# Lambda function for RDS password rotation
def lambda_handler(event, context):
    token = event['Token']
    step = event['Step']

    if step == "createSecret":
        # Generate new password
        new_password = generate_password()
        # Store as pending secret
        secretsmanager.put_secret_value(
            SecretId=secret_arn,
            SecretString=new_password,
            VersionStages=['AWSPENDING']
        )

    elif step == "setSecret":
        # Update database with new password
        pending_password = get_pending_password()
        rds.modify_db_instance(
            DBInstanceIdentifier=db_instance,
            MasterUserPassword=pending_password
        )

    elif step == "testSecret":
        # Test connection with new password
        test_database_connection(pending_password)

    elif step == "finishSecret":
        # Mark new password as current
        secretsmanager.update_secret_version_stage(
            SecretId=secret_arn,
            VersionStage='AWSCURRENT',
            MoveToVersionId=token
        )
```

**External Secrets Operator Integration**:
- Operator polls AWS Secrets Manager every `refreshInterval`
- Detects new `AWSCURRENT` version
- Updates Kubernetes Secret
- Pods restart automatically (via Reloader)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 API Key Rotation Patterns**

### **Pattern 1: Dual-Key Rotation**

**Scenario**: External API supports multiple active keys

```yaml
# Phase 1: Add new key
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
stringData:
  primary-key: old-key-12345    # Still valid
  secondary-key: new-key-67890  # Newly created

# Application uses primary-key
```

**Application Code** (Go example):
```go
// Support graceful key rotation
func getAPIKey() string {
    // Try primary key first
    if key := os.Getenv("PRIMARY_API_KEY"); key != "" {
        return key
    }

    // Fallback to secondary key
    if key := os.Getenv("SECONDARY_API_KEY"); key != "" {
        return key
    }

    return ""
}
```

**Rotation Steps**:
1. Generate new key in external system
2. Add as `secondary-key` to Secret
3. Update application to use `secondary-key` (rolling update)
4. Verify all pods using new key
5. Revoke old key in external system
6. Remove `primary-key` from Secret (cleanup)

### **Pattern 2: Service Account Key Rotation**

**GCP Service Account Keys**:

```bash
# List existing keys
gcloud iam service-accounts keys list \
  --iam-account=myapp@project.iam.gserviceaccount.com

# Create new key
gcloud iam service-accounts keys create new-key.json \
  --iam-account=myapp@project.iam.gserviceaccount.com

# Update Kubernetes secret
kubectl create secret generic gcp-credentials \
  --from-file=key.json=new-key.json \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart pods
kubectl rollout restart deployment/myapp

# Verify, then delete old key
gcloud iam service-accounts keys delete OLD_KEY_ID \
  --iam-account=myapp@project.iam.gserviceaccount.com
```

**Automation with CronJob**:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rotate-gcp-keys
spec:
  schedule: "0 0 1 * *"  # Monthly
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: key-rotator
          containers:
          - name: rotator
            image: google/cloud-sdk:latest
            command:
            - /bin/bash
            - -c
            - |
              # Generate new key
              gcloud iam service-accounts keys create /tmp/new-key.json \
                --iam-account=myapp@project.iam.gserviceaccount.com

              # Update Kubernetes secret
              kubectl create secret generic gcp-credentials \
                --from-file=key.json=/tmp/new-key.json \
                --dry-run=client -o yaml | kubectl apply -f -

              # Trigger rolling restart
              kubectl rollout restart deployment/myapp

              # Wait for rollout to complete
              kubectl rollout status deployment/myapp

              # Delete old key (keep most recent 2)
              OLD_KEY=$(gcloud iam service-accounts keys list --iam-account=myapp@project.iam.gserviceaccount.com --format="value(name)" --sort-by=~validAfterTime | tail -n +3 | head -n 1)
              if [ -n "$OLD_KEY" ]; then
                gcloud iam service-accounts keys delete $OLD_KEY --iam-account=myapp@project.iam.gserviceaccount.com --quiet
              fi
          restartPolicy: OnFailure
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚙️ Secret Versioning**

### **Versioned Secret Names**

**Pattern**: Create new Secret for each rotation

```yaml
# Version 1
apiVersion: v1
kind: Secret
metadata:
  name: db-creds-v1
  labels:
    version: "1"
    active: "false"
stringData:
  password: old-password
---
# Version 2 (current)
apiVersion: v1
kind: Secret
metadata:
  name: db-creds-v2
  labels:
    version: "2"
    active: "true"
stringData:
  password: new-password
```

**Deployment References Current Version**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-creds-v2  # Explicitly reference version
              key: password
```

**Advantages**:
- ✅ Full audit trail (all versions preserved)
- ✅ Easy rollback (change to db-creds-v1)
- ✅ No in-place updates (immutable secrets)

**Disadvantages**:
- ❌ Manual version tracking
- ❌ Orphaned secrets accumulate
- ❌ Deployment must be updated explicitly

**Cleanup Strategy**:
```bash
# Delete secrets older than 30 days
kubectl get secrets -l active=false -o json | \
  jq -r '.items[] | select(.metadata.creationTimestamp < (now - 30*86400 | todate)) | .metadata.name' | \
  xargs -I {} kubectl delete secret {}
```

### **Secret Annotations for Tracking**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
  annotations:
    rotation-date: "2024-01-15"
    rotated-by: "platform-team"
    next-rotation: "2024-02-15"
    rotation-reason: "scheduled-monthly-rotation"
stringData:
  api-key: rotated-key-12345
```

**Query Secrets Due for Rotation**:
```bash
# Find secrets with next-rotation < today
kubectl get secrets -A -o json | \
  jq -r '.items[] | select(.metadata.annotations["next-rotation"] != null and (.metadata.annotations["next-rotation"] < (now | strftime("%Y-%m-%d")))) | "\(.metadata.namespace)/\(.metadata.name)"'
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Monitoring and Alerting**

### **Certificate Expiration Monitoring**

**Prometheus Exporter**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: certificate-exporter-config
data:
  config.yaml: |
    certificates:
      - name: api-server
        path: /etc/kubernetes/pki/apiserver.crt
      - name: kubelet-client
        path: /etc/kubernetes/pki/apiserver-kubelet-client.crt
```

**Prometheus Alert**:
```yaml
groups:
- name: certificates
  rules:
  - alert: CertificateExpiringSoon
    expr: (x509_cert_not_after - time()) / 86400 < 30
    for: 24h
    annotations:
      summary: "Certificate {{ $labels.name }} expiring in {{ $value }} days"
      description: "Certificate will expire soon, rotation required"

  - alert: CertificateExpired
    expr: (x509_cert_not_after - time()) < 0
    for: 1m
    annotations:
      summary: "Certificate {{ $labels.name }} has EXPIRED"
```

### **Secret Age Monitoring**

**Custom Metric**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: secret-age-exporter
data:
  script.sh: |
    #!/bin/bash
    # Export secret age as Prometheus metric
    kubectl get secrets -A -o json | jq -r '
      .items[] |
      select(.metadata.annotations["rotation-date"] != null) |
      {
        namespace: .metadata.namespace,
        name: .metadata.name,
        rotation_date: .metadata.annotations["rotation-date"],
        age: ((now - (.metadata.annotations["rotation-date"] | fromdateiso8601)) / 86400 | floor)
      } |
      "secret_age_days{namespace=\"\(.namespace)\",name=\"\(.name)\"} \(.age)"
    '
```

**Alert on Stale Secrets**:
```yaml
groups:
- name: secret-rotation
  rules:
  - alert: SecretNotRotated
    expr: secret_age_days > 90
    annotations:
      summary: "Secret {{ $labels.namespace }}/{{ $labels.name }} not rotated in 90 days"
```

### **Audit Log for Secret Access**

**Enable Audit Logging**:
```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log secret access
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

**Query Secret Access**:
```bash
# Find who accessed which secrets
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource == "secrets") | {user: .user.username, secret: .objectRef.name, verb: .verb, time: .requestReceivedTimestamp}'
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting**

### **Issue 1: Pods Not Picking Up Rotated Secret**

**Symptom**: Secret updated but pods still using old credentials

**Cause**: Secrets mounted as volumes are **eventually consistent** (kubelet sync period: 1 minute by default)

**Verification**:
```bash
# Check secret in Kubernetes
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d

# Check what pod sees
kubectl exec -it myapp-pod -- cat /var/run/secrets/db-credentials/password
# May show old value for up to 1 minute
```

**Solutions**:

**Option 1**: Wait for kubelet sync (up to 1 minute)

**Option 2**: Restart pods immediately
```bash
kubectl rollout restart deployment/myapp
```

**Option 3**: Use environment variables instead of mounted volumes
```yaml
# Environment variables are set at pod creation, require restart
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

**Option 4**: Use projected volumes with `subPath` for faster sync
```yaml
volumes:
- name: secret-volume
  projected:
    sources:
    - secret:
        name: db-credentials
```

### **Issue 2: Database Connection Failures During Rotation**

**Symptom**: Application errors during credential rotation

**Cause**: Race condition - pods trying new password before database updated

**Investigation**:
```bash
# Check application logs
kubectl logs -l app=myapp | grep "authentication failed"

# Check database logs for failed login attempts
```

**Prevention**: **Always update database before Kubernetes Secret**

```bash
# CORRECT ORDER:
# 1. Add new password to database
psql -c "CREATE USER myapp_v2 WITH PASSWORD 'new-password';"

# 2. Update Kubernetes Secret
kubectl apply -f db-credentials.yaml

# 3. Restart pods
kubectl rollout restart deployment/myapp

# 4. Verify all pods using new credentials
kubectl logs -l app=myapp | grep "database connected"

# 5. Remove old password from database
psql -c "DROP USER myapp;"
```

### **Issue 3: Certificate Rotation Causing TLS Errors**

**Symptom**: TLS handshake failures after certificate rotation

**Cause**: Certificate chain mismatch, wrong CA

**Debugging**:
```bash
# Verify certificate chain
openssl s_client -connect myapp.example.com:443 -showcerts

# Check certificate in secret
kubectl get secret app-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# Common issues:
# - Missing intermediate certificates
# - Wrong CA certificate
# - Certificate and key mismatch
```

**Fix**: Ensure complete certificate chain
```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: app-tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    <server certificate>
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    <intermediate CA certificate>  # Include intermediate!
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    <private key>
    -----END PRIVATE KEY-----
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Related Documentation**

### **Security Context**
- **[Secrets and Encryption](./03-secrets-and-encryption.md)** - Secret storage and encryption
- **[RBAC Patterns](./05-rbac-patterns-troubleshooting.md)** - Secret access control
- **[Certificate Controllers](../controller-manager/17-certificate-controllers.md)** - Automated cert management

### **Operational Patterns**
- **[Upgrade Strategies](../lifecycle/02-kubeadm-upgrade-strategies.md)** - Certificate renewal during upgrades
- **[Disaster Recovery](../scalability/04-disaster-recovery-strategies.md)** - Secret backup strategies
- **[Audit Logging](../observability/02-logging-and-analysis.md)** - Secret access auditing

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Key Takeaways**

### **For Platform Engineers**

1. **Certificate Auto-Renewal Works**:
   - kubelet automatically renews its own certificate at 80% lifetime
   - kubeadm certificates can be renewed with `kubeadm certs renew all`
   - Check expiration: `kubeadm certs check-expiration`

2. **Database Credentials Need Two-Phase Rotation**:
   - Add new credentials to database first
   - Update Kubernetes Secret second
   - Restart pods to pick up new credentials
   - Remove old credentials last

3. **External Secrets Operator Simplifies Rotation**:
   - Centralized secret management
   - Automatic sync from external providers (AWS SM, Vault, GCP SM)
   - Pair with Reloader for automatic pod restarts

4. **Monitoring is Critical**:
   - Alert on certificate expiration (< 30 days)
   - Track secret age with annotations
   - Audit secret access in audit logs

5. **Versioned Secrets Enable Rollback**:
   - Create new Secret for each rotation (db-creds-v1, db-creds-v2)
   - Keep 2-3 versions for rollback
   - Cleanup old versions regularly

### **For Kubernetes Contributors**

1. **Certificate Renewal Implementation**:
   - kubelet: `pkg/kubelet/certificate/bootstrap/`
   - kubeadm: `cmd/kubeadm/app/cmd/certs/`
   - Auto-renewal threshold: 80% of certificate lifetime

2. **Secret Update Propagation**:
   - Mounted secrets: Eventually consistent (kubelet sync period)
   - Environment variables: Set at pod creation (require restart)
   - Projected volumes: Faster sync

3. **Extension Points**:
   - Custom rotation controllers using client-go
   - CRDs for rotation policies
   - Admission webhooks for rotation enforcement

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Kubernetes Version**: v1.30
**Last Updated**: 2024-01-15
**Maintained By**: Kubernetes Architecture Study Group
