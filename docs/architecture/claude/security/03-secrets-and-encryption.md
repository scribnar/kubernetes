# **Secrets and Encryption - Deep Architectural Analysis**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Target Audience**: Platform engineers, Kubernetes architects, security engineers, SREs managing production clusters, compliance teams

**Scope**: Deep architectural analysis of Kubernetes Secrets, encryption at rest, Key Management Service (KMS) integration, envelope encryption, and external secret management patterns. This document examines how Kubernetes protects sensitive data at the source code level to help platform engineers build secure, compliant multi-tenant platforms.

**Prerequisites**:
- Understanding of [Pod Security Standards](./01-pod-security-standards.md)
- Familiarity with [RBAC patterns](./05-rbac-patterns-troubleshooting.md)
- Knowledge of [etcd architecture](../etcd/high-level/01-etcd-architecture.md)
- Basic cryptography concepts (symmetric/asymmetric encryption, envelope encryption)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Design Philosophy**

### **Secrets Management Challenges**

Kubernetes must protect sensitive data:
- **Database credentials**: Connection strings, passwords
- **API keys**: External service authentication
- **TLS certificates**: mTLS for service-to-service communication
- **SSH keys**: Git repository access
- **OAuth tokens**: User authentication

**Security Requirements**:
1. **Encryption at rest**: Secrets encrypted in etcd
2. **Encryption in transit**: TLS for API communication
3. **Access control**: RBAC limits who can read secrets
4. **Audit trail**: Who accessed which secret when
5. **Rotation**: Secrets can be updated without downtime
6. **External integration**: Support enterprise KMS (AWS KMS, Vault, etc.)

### **Core Design Principles**

```
┌──────────────────────────────────────────────────────────────┐
│  SECRETS SECURITY DESIGN PRINCIPLES                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. DEFENSE IN DEPTH                                         │
│     └─ Multiple layers: RBAC + encryption + audit           │
│                                                               │
│  2. LEAST PRIVILEGE ACCESS                                   │
│     └─ Secrets only mounted to pods that need them          │
│                                                               │
│  3. ENCRYPTION AT REST                                       │
│     └─ Secrets encrypted in etcd storage                    │
│                                                               │
│  4. ENVELOPE ENCRYPTION                                      │
│     └─ Data Encryption Keys (DEKs) encrypted by KEK         │
│                                                               │
│  5. SEPARATION OF DUTIES                                     │
│     └─ Different teams manage keys vs. secrets              │
│                                                               │
│  6. AUDIT EVERYTHING                                         │
│     └─ All secret access logged                             │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔐 Kubernetes Secrets API**

### **Secret Resource Structure**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque  # Generic secret type
data:
  # Base64-encoded values
  username: YWRtaW4=        # "admin"
  password: c3VwZXJzZWNyZXQ=  # "supersecret"
stringData:
  # Plain text (API server base64-encodes before storing)
  host: postgres.example.com
```

**Secret Types**:

| **Type** | **Purpose** | **Data Fields** | **Validation** |
|----------|------------|----------------|----------------|
| `Opaque` | Generic key-value pairs | Any | None |
| `kubernetes.io/service-account-token` | Service account tokens | `token`, `ca.crt`, `namespace` | Automatic by SA controller |
| `kubernetes.io/dockercfg` | Docker registry auth (legacy) | `.dockercfg` | JSON validation |
| `kubernetes.io/dockerconfigjson` | Docker registry auth | `.dockerconfigjson` | JSON validation |
| `kubernetes.io/basic-auth` | Basic HTTP authentication | `username`, `password` | Required fields |
| `kubernetes.io/ssh-auth` | SSH authentication | `ssh-privatekey` | SSH key format |
| `kubernetes.io/tls` | TLS certificates | `tls.crt`, `tls.key` | X.509 validation |
| `bootstrap.kubernetes.io/token` | Bootstrap tokens | `token-id`, `token-secret` | Token format |

### **⚠️ Critical Misconception: Base64 is NOT Encryption**

```go
// staging/src/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go

// Secret data is base64-encoded for safe transport, NOT for security
func (e *Encoder) Encode(obj runtime.Object, w io.Writer) error {
    // ...
    for k, v := range secret.Data {
        // v is already base64-encoded by client
        data[k] = v
    }
    // ...
}
```

**Why Base64?**
- **Binary safety**: Secrets may contain binary data (certificates, keys)
- **JSON compatibility**: JSON requires UTF-8 strings
- **NOT for security**: Anyone with `kubectl get secret` access can decode

**Proof**:
```bash
# Create secret
kubectl create secret generic my-secret --from-literal=password=supersecret

# Retrieve and decode
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
# Output: supersecret
```

**Implication**: **RBAC is critical** - secret access = plaintext access

### **Secret Storage in etcd**

**Without Encryption at Rest**:
```bash
# Read secret directly from etcd
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/apiserver-etcd-client.crt \
  --key=/etc/kubernetes/pki/apiserver-etcd-client.key \
  --print-value-only

# Output: Base64-encoded secret (easily decoded)
# k8s...password...YWRtaW4=...
```

**Security Risk**:
- etcd backup tapes contain plaintext secrets (base64 ≈ plaintext)
- Compromised etcd node = all secrets leaked
- Stolen etcd snapshot = all secrets accessible

**Solution**: **Encryption at Rest**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔒 Encryption at Rest**

### **EncryptionConfiguration**

**API server flag**:
```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
    volumeMounts:
    - name: encryption-config
      mountPath: /etc/kubernetes/encryption-config.yaml
      readOnly: true
  volumes:
  - name: encryption-config
    hostPath:
      path: /etc/kubernetes/encryption-config.yaml
      type: File
```

**EncryptionConfiguration File**:
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      # Provider 1: AES-CBC encryption (first provider = write encryption)
      - aescbc:
          keys:
            - name: key1
              secret: 32-byte-base64-encoded-key-here==

      # Provider 2: Identity (no encryption - for migration/fallback)
      - identity: {}
```

**Provider Types**:

| **Provider** | **Encryption** | **Performance** | **Security** | **Key Management** | **Recommended** |
|--------------|---------------|----------------|--------------|-------------------|----------------|
| `identity` | None (plaintext) | Fastest | ❌ None | N/A | No (migration only) |
| `aescbc` | AES-CBC | Fast | ✅ Good | Manual (in config file) | Testing only |
| `aesgcm` | AES-GCM (AEAD) | Fast | ✅ Better (authenticated) | Manual | Testing only |
| `secretbox` | XSalsa20-Poly1305 | Fast | ✅ Better (authenticated) | Manual | Testing only |
| `kms` | Envelope encryption | Slower (external call) | ✅ Best (HSM-backed) | External KMS | **Production** |

### **How Encryption Works**

```go
// staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/envelope/envelope.go

func (t *envelopeTransformer) TransformToStorage(
    ctx context.Context,
    data []byte,
    dataCtx value.Context,
) ([]byte, error) {

    // 1. Get first provider (write provider)
    provider := t.providers[0]

    // 2. Encrypt data using provider
    encrypted, err := provider.Encrypt(data)

    // 3. Store encrypted data in etcd
    return encrypted, nil
}

func (t *envelopeTransformer) TransformFromStorage(
    ctx context.Context,
    data []byte,
    dataCtx value.Context,
) ([]byte, error) {

    // Try each provider in order until one succeeds
    for _, provider := range t.providers {
        decrypted, err := provider.Decrypt(data)
        if err == nil {
            return decrypted, nil  // Success
        }
    }

    return nil, fmt.Errorf("no provider could decrypt data")
}
```

**Write Path**:
```
kubectl create secret → API server → Encrypt with provider[0] → Store in etcd
```

**Read Path**:
```
kubectl get secret → API server → Read from etcd → Try decrypt with each provider → Return plaintext to authorized user
```

### **AES-CBC Provider (Manual Key)**

**Generate Encryption Key**:
```bash
# Generate 32-byte random key
head -c 32 /dev/urandom | base64

# Output: wLmNzqLpQz+VhJxK2B5ePqXyZkN8RmTaQ1WvGfDjCxI=
```

**EncryptionConfiguration**:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: wLmNzqLpQz+VhJxK2B5ePqXyZkN8RmTaQ1WvGfDjCxI=
      - identity: {}  # Fallback for unencrypted secrets
```

**Apply Configuration**:
```bash
# 1. Update encryption config file on all control plane nodes
scp encryption-config.yaml control-plane-1:/etc/kubernetes/

# 2. Restart API server (static pod - edit manifest to trigger restart)
kubectl delete pod kube-apiserver-control-plane-1 -n kube-system
# kubelet automatically restarts static pod

# 3. Verify API server started successfully
kubectl get pods -n kube-system | grep kube-apiserver
```

**⚠️ Security Issues with Manual Keys**:
- Key stored in plaintext config file
- No key rotation mechanism
- No audit trail for key usage
- Manual distribution to all API server nodes
- **Not recommended for production**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔑 KMS Provider (Envelope Encryption)**

### **Envelope Encryption Architecture**

```
┌──────────────────────────────────────────────────────────────┐
│  ENVELOPE ENCRYPTION FLOW                                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. API Server generates Data Encryption Key (DEK)           │
│     └─ Unique AES-256 key per secret                         │
│                                                               │
│  2. Encrypt secret data with DEK                             │
│     Secret Data + DEK → Encrypted Secret Data                │
│                                                               │
│  3. Encrypt DEK with KMS Key Encryption Key (KEK)            │
│     KMS Encrypt(DEK) → Encrypted DEK                         │
│                                                               │
│  4. Store both in etcd                                       │
│     etcd stores: [Encrypted Secret Data, Encrypted DEK]      │
│                                                               │
│  5. On read: Decrypt DEK with KMS, then decrypt data         │
│     KMS Decrypt(Encrypted DEK) → DEK                         │
│     DEK decrypt → Secret Data                                │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**Why Envelope Encryption?**
- **Performance**: Encrypt large secrets with fast symmetric DEK
- **KMS efficiency**: Only small DEK sent to KMS (not entire secret)
- **Key rotation**: Rotate KEK without re-encrypting all secrets
- **Compliance**: KEK in HSM, meets regulatory requirements

### **KMS Plugin Architecture**

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  kube-apiserver                                     │     │
│  │                                                      │     │
│  │  ┌────────────────────────────────────────────┐    │     │
│  │  │ Encryption/Decryption Logic                │    │     │
│  │  └────────────────────────────────────────────┘    │     │
│  │              │                                      │     │
│  │              ▼                                      │     │
│  │  ┌────────────────────────────────────────────┐    │     │
│  │  │ KMS Plugin (gRPC client)                   │    │     │
│  │  └────────────────────────────────────────────┘    │     │
│  └──────────────────────┬───────────────────────────────     │
│                         │ Unix socket                         │
│                         │ /var/run/kmsplugin/socket          │
│                         ▼                                     │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ KMS Plugin (gRPC server)                            │     │
│  │                                                      │     │
│  │  ┌────────────────────────────────────────────┐    │     │
│  │  │ Encrypt/Decrypt RPC handlers               │    │     │
│  │  └────────────────────────────────────────────┘    │     │
│  │              │                                      │     │
│  │              ▼                                      │     │
│  │  ┌────────────────────────────────────────────┐    │     │
│  │  │ KMS Client (AWS SDK, GCP SDK, etc.)        │    │     │
│  │  └────────────────────────────────────────────┘    │     │
│  └──────────────────────┬───────────────────────────────     │
│                         │ HTTPS                               │
│                         ▼                                     │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ External KMS (AWS KMS, GCP Cloud KMS, Vault, etc.) │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### **KMS v2 Configuration (Kubernetes v1.25+)**

**EncryptionConfiguration**:
```yaml
apiVersion: apiserver.config.k8s.io/v2
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      # KMS v2 provider
      - kms:
          apiVersion: v2
          name: aws-kms-provider
          endpoint: unix:///var/run/kmsplugin/socket.sock
          cachesize: 1000  # Cache DEKs (performance optimization)
          timeout: 3s      # KMS call timeout

      # Fallback to identity for migration
      - identity: {}
```

**API Server Flags**:
```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
    volumeMounts:
    - name: kmsplugin
      mountPath: /var/run/kmsplugin
  volumes:
  - name: kmsplugin
    hostPath:
      path: /var/run/kmsplugin
      type: DirectoryOrCreate
```

### **AWS KMS Integration**

**Install AWS KMS Plugin**:
```yaml
# /etc/systemd/system/aws-kms-plugin.service
[Unit]
Description=AWS KMS Plugin for Kubernetes
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/aws-encryption-provider \
  --key=arn:aws:kms:us-west-2:123456789012:key/12345678-1234-1234-1234-123456789012 \
  --region=us-west-2 \
  --listen=/var/run/kmsplugin/socket.sock
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Start KMS plugin
sudo systemctl enable aws-kms-plugin
sudo systemctl start aws-kms-plugin

# Verify socket exists
ls -la /var/run/kmsplugin/socket.sock
```

**IAM Policy for KMS Plugin**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-west-2:123456789012:key/12345678-1234-1234-1234-123456789012"
    }
  ]
}
```

**Attach to EC2 Instance Role** (if using EC2):
```bash
aws iam attach-role-policy \
  --role-name k8s-control-plane-role \
  --policy-arn arn:aws:iam::123456789012:policy/KMSEncryptionPolicy
```

### **GCP Cloud KMS Integration**

**Install GCP KMS Plugin**:
```bash
# Download plugin
curl -LO https://github.com/GoogleCloudPlatform/k8s-cloudkms-plugin/releases/download/v0.3.0/k8s-cloudkms-plugin

# Install
sudo mv k8s-cloudkms-plugin /usr/local/bin/
sudo chmod +x /usr/local/bin/k8s-cloudkms-plugin
```

**Systemd Service**:
```ini
[Unit]
Description=GCP Cloud KMS Plugin
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/k8s-cloudkms-plugin \
  --project-id=my-project \
  --location=us-central1 \
  --key-ring=kubernetes-keyring \
  --key-name=kubernetes-key \
  --path-to-unix-socket=/var/run/kmsplugin/socket.sock
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**GCP IAM Permissions**:
```bash
# Grant Cloud KMS CryptoKey Encrypter/Decrypter role
gcloud kms keys add-iam-policy-binding kubernetes-key \
  --location=us-central1 \
  --keyring=kubernetes-keyring \
  --member=serviceAccount:k8s-control-plane@my-project.iam.gserviceaccount.com \
  --role=roles/cloudkms.cryptoKeyEncrypterDecrypter
```

### **HashiCorp Vault Integration**

**Install Vault KMS Plugin**:
```bash
# Using https://github.com/oracle/kubernetes-vault-kms-plugin
git clone https://github.com/oracle/kubernetes-vault-kms-plugin
cd kubernetes-vault-kms-plugin
make build
sudo cp bin/kubernetes-vault-kms-plugin /usr/local/bin/
```

**Configuration**:
```yaml
# /etc/kubernetes-vault-kms-plugin/config.yaml
vault:
  addr: https://vault.example.com:8200
  token: s.abcdefg12345
  transit_key_name: kubernetes
  tls_ca_cert: /etc/vault/ca.crt
```

**Systemd Service**:
```ini
[Service]
ExecStart=/usr/local/bin/kubernetes-vault-kms-plugin \
  --config=/etc/kubernetes-vault-kms-plugin/config.yaml \
  --socket-path=/var/run/kmsplugin/socket.sock
```

**Vault Transit Engine Setup**:
```bash
# Enable transit engine
vault secrets enable transit

# Create encryption key
vault write -f transit/keys/kubernetes

# Create policy
vault policy write kubernetes-kms - <<EOF
path "transit/encrypt/kubernetes" {
  capabilities = ["update"]
}
path "transit/decrypt/kubernetes" {
  capabilities = ["update"]
}
EOF

# Create token
vault token create -policy=kubernetes-kms
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Encrypting Existing Secrets**

### **The Problem**

Enabling encryption does NOT automatically encrypt existing secrets:

```bash
# Before enabling encryption
kubectl create secret generic old-secret --from-literal=key=value

# Enable encryption (restart API server)

# Old secret is STILL UNENCRYPTED in etcd
# Only NEW secrets are encrypted
```

### **Solution: Re-write All Secrets**

**Strategy**: Force API server to read and re-write every secret

```bash
# Re-encrypt all secrets in all namespaces
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -
```

**How It Works**:
1. `kubectl get secrets -o json`: Read all secrets (plaintext from API)
2. `kubectl replace -f -`: Write secrets back
3. API server writes with NEW encryption provider (provider[0])
4. etcd now contains encrypted secrets

**For Large Clusters** (thousands of secrets):
```bash
#!/bin/bash
# encrypt-all-secrets.sh

NAMESPACES=$(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}')

for NS in $NAMESPACES; do
    echo "Encrypting secrets in namespace: $NS"

    SECRETS=$(kubectl get secrets -n $NS -o name)
    for SECRET in $SECRETS; do
        kubectl get $SECRET -n $NS -o json | kubectl replace -f -
    done
done
```

**⚠️ Important Notes**:
- During re-encryption, secrets are briefly in memory (plaintext)
- Large clusters: Process may take hours
- Monitor API server memory usage
- Consider batch processing (100 secrets at a time)

### **Verification**

**Check if Secret is Encrypted in etcd**:
```bash
# Read secret from etcd
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/apiserver-etcd-client.crt \
  --key=/etc/kubernetes/pki/apiserver-etcd-client.key \
  --print-value-only | hexdump -C

# ENCRYPTED: Output starts with "k8s:enc:kms:v2:..."
# PLAINTEXT: Output contains readable text "password", "username", etc.
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔐 RBAC for Secrets**

### **Default Permissions**

**Who Can Access Secrets?**

| **Subject** | **Default Access** | **Rationale** |
|-------------|-------------------|---------------|
| `system:masters` group | ✅ Full access | Cluster admins |
| Namespace admins | ✅ Full access in namespace | `admin` ClusterRole binding |
| Pod service accounts | ❌ No access (unless granted) | Least privilege |
| Regular users | ❌ No access (unless granted) | Least privilege |

### **Granting Secret Access**

**Namespace-scoped** (recommended):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-read-secrets
  namespace: production
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**Cluster-scoped** (use sparingly):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-admin
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-team-secret-admin
subjects:
- kind: Group
  name: platform-team
roleRef:
  kind: ClusterRole
  name: secret-admin
```

### **Restricting Secret Access**

**Anti-Pattern**: Broad secret access
```yaml
# DON'T DO THIS
rules:
- apiGroups: [""]
  resources: ["*"]  # Includes secrets!
  verbs: ["*"]
```

**Best Practice**: Explicit secret access
```yaml
# Explicitly grant access to specific secrets
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-credentials", "api-keys"]  # Only these secrets
  verbs: ["get"]
```

### **Service Account Secret Access**

**Problem**: Service account can read secrets in same namespace by default? **NO**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: production
---
# ServiceAccount has NO default secret permissions
# Must explicitly grant via Role + RoleBinding
```

**Granting Service Account Access**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["myapp-db-creds"]  # Specific secret
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-read-db-secret
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp
  namespace: production
roleRef:
  kind: Role
  name: myapp-secret-reader
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔌 External Secret Management**

### **Why External Secret Management?**

**Limitations of Native Secrets**:
- Secret rotation requires pod restart
- No built-in secret versioning
- Limited audit capabilities
- No centralized secret management across clusters

**External Solutions**:
- **HashiCorp Vault**: Enterprise secret management
- **AWS Secrets Manager**: AWS-native solution
- **GCP Secret Manager**: GCP-native solution
- **Azure Key Vault**: Azure-native solution
- **Sealed Secrets**: Encrypted secrets in Git

### **External Secrets Operator**

**Architecture**:
```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ External Secrets Operator                           │     │
│  │                                                      │     │
│  │  Watches: ExternalSecret CRD                        │     │
│  │  ↓                                                   │     │
│  │  Fetches secret from external provider              │     │
│  │  ↓                                                   │     │
│  │  Creates/Updates Kubernetes Secret                  │     │
│  └──────────────────────────────────────────────────────     │
│              │                                                │
│              ▼                                                │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ External Provider (Vault, AWS SM, GCP SM, etc.)     │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**Installation**:
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace
```

**Example: AWS Secrets Manager**

**1. Create SecretStore** (connection to AWS):
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

**2. Create ExternalSecret** (reference to external secret):
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h  # Sync every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-credentials  # Name of K8s Secret to create
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: /production/database  # AWS Secrets Manager path
      property: username
  - secretKey: password
    remoteRef:
      key: /production/database
      property: password
```

**Result**: Kubernetes Secret automatically created/updated from AWS Secrets Manager

### **Sealed Secrets (GitOps-Friendly)**

**Problem**: Can't commit Secrets to Git (plaintext)

**Solution**: Encrypted secrets that only cluster can decrypt

**Architecture**:
```
Developer                     Cluster
    │                            │
    ├─ Create Secret            │
    ├─ Encrypt with kubeseal    │
    │  (public key)              │
    ├─ Commit SealedSecret      │
    │  to Git                    │
    │                            │
    └─ GitOps sync ─────────────►│
                                 ├─ SealedSecret Controller
                                 ├─ Decrypt with private key
                                 └─ Create K8s Secret
```

**Installation**:
```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml
```

**Usage**:
```bash
# Install kubeseal CLI
brew install kubeseal

# Create secret (don't apply!)
kubectl create secret generic my-secret \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml > secret.yaml

# Seal secret (encrypt)
kubeseal -f secret.yaml -w sealed-secret.yaml

# Commit to Git
git add sealed-secret.yaml
git commit -m "Add sealed secret"
```

**SealedSecret**:
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: default
spec:
  encryptedData:
    password: AgA8... # Encrypted, safe to commit
```

**Apply**:
```bash
kubectl apply -f sealed-secret.yaml

# Controller decrypts and creates Secret
kubectl get secret my-secret
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting**

### **Issue 1: KMS Plugin Not Responding**

**Symptom**:
```bash
$ kubectl create secret generic test --from-literal=key=value
Error from server (InternalError): Internal error occurred: failed to encrypt data: connection error
```

**Investigation**:
```bash
# Check KMS plugin process
systemctl status aws-kms-plugin

# Check socket exists
ls -la /var/run/kmsplugin/socket.sock

# Check API server logs
kubectl logs -n kube-system kube-apiserver-control-plane-1 | grep -i kms

# Test KMS plugin manually (if plugin exposes health endpoint)
curl http://localhost:8080/healthz
```

**Common Causes**:
- KMS plugin crashed
- Socket file missing/wrong permissions
- Network connectivity to external KMS
- IAM/credentials issue (AWS, GCP)

**Resolution**:
```bash
# Restart KMS plugin
sudo systemctl restart aws-kms-plugin

# Verify API server can connect
# Check API server logs for successful KMS connection
```

### **Issue 2: Secrets Still Unencrypted After Enabling Encryption**

**Symptom**: etcd contains plaintext secrets even after configuring encryption

**Cause**: Existing secrets not re-encrypted

**Verification**:
```bash
# Check secret in etcd
ETCDCTL_API=3 etcdctl get /registry/secrets/default/old-secret --print-value-only | hexdump -C

# If readable text visible: NOT encrypted
```

**Resolution**:
```bash
# Re-encrypt all secrets
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

### **Issue 3: Performance Degradation After Enabling KMS**

**Symptom**: Secret operations significantly slower

**Cause**: External KMS call latency

**Investigation**:
```promql
# KMS operation latency
histogram_quantile(0.99, rate(apiserver_storage_transformation_duration_seconds_bucket{transformation_type="from_storage"}[5m]))
```

**Mitigation**:
```yaml
# Enable DEK caching
- kms:
    cachesize: 1000  # Cache up to 1000 DEKs
    timeout: 3s       # Reasonable timeout
```

**Explanation**: Caching DEKs reduces KMS calls (only call KMS for new DEKs)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Related Documentation**

### **Security Context**
- **[Pod Security Standards](./01-pod-security-standards.md)** - Container security policies
- **[RBAC Patterns](./05-rbac-patterns-troubleshooting.md)** - Access control
- **[Secret Rotation](./04-secrets-rotation.md)** - Automated secret updates

### **Storage and Persistence**
- **[etcd Architecture](../etcd/high-level/01-etcd-architecture.md)** - Where secrets are stored
- **[etcd Backup and Restore](../etcd/middle-level/06-backup-restore.md)** - Protecting secret backups

### **Audit and Compliance**
- **[Audit Logging](../observability/02-logging-and-analysis.md)** - Secret access audit trail
- **[Compliance Patterns](../common/13-common-patterns-integration.md)** - Meeting regulatory requirements

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Key Takeaways**

### **For Platform Engineers**

1. **Base64 is NOT Encryption**:
   - Secrets are base64-encoded for transport, NOT security
   - Anyone with RBAC access can decode secrets
   - MUST enable encryption at rest for production

2. **Use KMS for Production**:
   - Manual keys (aescbc, aesgcm) NOT recommended
   - KMS provides envelope encryption with HSM-backed keys
   - Supports key rotation without re-encrypting secrets
   - Meets compliance requirements (PCI-DSS, HIPAA, etc.)

3. **Re-encrypt Existing Secrets**:
   - Enabling encryption does NOT encrypt existing secrets
   - Must explicitly re-write all secrets
   - Use `kubectl get secrets -o json | kubectl replace -f -`

4. **RBAC is Critical**:
   - Secret access = plaintext access (even with encryption at rest)
   - Grant minimal necessary permissions
   - Use resourceNames to restrict to specific secrets

5. **Consider External Secret Management**:
   - External Secrets Operator for AWS/GCP/Azure integration
   - Sealed Secrets for GitOps workflows
   - Centralized secret management across clusters

### **For Kubernetes Contributors**

1. **Implementation Locations**:
   - Encryption at rest: `staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/`
   - KMS provider: `staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/envelope/`
   - Secret API: `pkg/registry/core/secret/`

2. **Encryption Flow**:
   - Write: Generate DEK → Encrypt data → Encrypt DEK with KMS → Store both
   - Read: Fetch encrypted data → Decrypt DEK with KMS → Decrypt data → Return

3. **KMS Plugin Protocol**:
   - gRPC-based (v2 API)
   - Unix socket communication
   - Encrypt/Decrypt RPCs

4. **Testing Encryption**:
   - Unit tests: `staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/envelope/envelope_test.go`
   - Integration tests: `test/integration/apiserver/encryption/`
   - E2E tests: Read from etcd, verify encrypted

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Kubernetes Version**: v1.30
**Last Updated**: 2024-01-15
**Maintained By**: Kubernetes Architecture Study Group
