# **RBAC Patterns and Troubleshooting - Deep Architectural Analysis**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Target Audience**: Platform engineers, Kubernetes architects, security engineers, SREs managing production clusters, multi-tenant platform teams

**Scope**: Deep architectural analysis of Kubernetes Role-Based Access Control (RBAC), common permission patterns, multi-tenancy isolation strategies, service account management, and RBAC troubleshooting. This document examines RBAC implementation at the source code level to help platform engineers design secure, scalable authorization systems.

**Prerequisites**:
- Understanding of [Pod Security Standards](./01-pod-security-standards.md)
- Familiarity with [Secrets and Encryption](./03-secrets-and-encryption.md)
- Knowledge of [API server authentication](../apiserver/middle-level/04-authentication.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Design Philosophy**

### **RBAC Fundamentals**

**Authorization Flow**:
```
User/Service Account → Authentication → RBAC Authorization → Admission → Resource Access
                                              ↓
                                    Is action permitted?
                                    (Role + RoleBinding)
```

**RBAC Components**:
1. **Role/ClusterRole**: Define permissions (verbs on resources)
2. **RoleBinding/ClusterRoleBinding**: Grant permissions to subjects (users, groups, service accounts)
3. **Subjects**: Who gets permissions (User, Group, ServiceAccount)

### **Core Design Principles**

```
┌──────────────────────────────────────────────────────────────┐
│  RBAC DESIGN PRINCIPLES                                       │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. LEAST PRIVILEGE                                          │
│     └─ Grant minimal permissions needed for task            │
│                                                               │
│  2. DENY BY DEFAULT                                          │
│     └─ No permissions unless explicitly granted             │
│                                                               │
│  3. NAMESPACE ISOLATION                                      │
│     └─ Roles scoped to namespace when possible              │
│                                                               │
│  4. SEPARATION OF DUTIES                                     │
│     └─ Different roles for different responsibilities       │
│                                                               │
│  5. AUDIT TRAIL                                              │
│     └─ All authorization decisions logged                   │
│                                                               │
│  6. GROUP-BASED PERMISSIONS                                  │
│     └─ Grant to groups, not individual users                │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏗️ RBAC Architecture**

### **Role vs ClusterRole**

| **Aspect** | **Role** | **ClusterRole** |
|------------|---------|----------------|
| **Scope** | Single namespace | Cluster-wide |
| **Resources** | Namespaced resources (Pods, Services, etc.) | All resources (including cluster-scoped) |
| **Use Case** | Team/project permissions | Admin, cluster services |
| **Example** | Developer read access to `dev` namespace | Cluster admin, node management |

### **RoleBinding vs ClusterRoleBinding**

| **Aspect** | **RoleBinding** | **ClusterRoleBinding** |
|------------|----------------|----------------------|
| **Scope** | Single namespace | Cluster-wide |
| **Binds To** | Role or ClusterRole | ClusterRole only |
| **Grants** | Permissions in one namespace | Permissions across all namespaces (or cluster) |
| **Pattern** | Namespace-scoped delegation | Cluster-wide delegation |

**Key Pattern**: ClusterRole + RoleBinding
```yaml
# Define permissions once (ClusterRole)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# Grant in specific namespace (RoleBinding)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-read-pods
  namespace: production
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole  # References ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Advantage**: Define role once, reuse across namespaces

### **RBAC Authorization Flow**

```go
// plugin/pkg/auth/authorizer/rbac/rbac.go

func (r *RBACAuthorizer) Authorize(ctx context.Context, attr authorizer.Attributes) (authorizer.Decision, string, error) {
    // 1. Get user info from context
    user := attr.GetUser()

    // 2. Build list of RoleBindings and ClusterRoleBindings for this user
    ruleResolver := r.ruleResolver

    // 3. Get all rules that apply to this user
    rules, err := ruleResolver.RulesFor(user, attr.GetNamespace())

    // 4. Check if any rule allows this action
    for _, rule := range rules {
        if RuleAllows(attr, rule) {
            return authorizer.DecisionAllow, "", nil
        }
    }

    // 5. No matching rule = deny
    return authorizer.DecisionNoOpinion, "RBAC: no matching rule", nil
}
```

**Authorization Steps**:
1. Extract user, namespace, resource, verb from request
2. Find all RoleBindings/ClusterRoleBindings for user (direct, group, service account)
3. Collect all Roles/ClusterRoles referenced by bindings
4. Check if any rule matches `(apiGroup, resource, verb, resourceName)`
5. **Allow if match found**, otherwise **deny**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **👥 Default Roles and System Accounts**

### **Built-in ClusterRoles**

| **ClusterRole** | **Permissions** | **Use Case** |
|----------------|----------------|--------------|
| `cluster-admin` | Full cluster access (`*` on `*`) | Break-glass admin access |
| `admin` | Full namespace access (read/write) | Namespace owner |
| `edit` | Read/write resources (no RBAC) | Developer |
| `view` | Read-only access | Auditor, monitoring |
| `system:node` | Node kubelet permissions | kubelet on nodes |
| `system:kube-controller-manager` | Controller manager permissions | Controller manager |
| `system:kube-scheduler` | Scheduler permissions | Scheduler |

**Example: view ClusterRole**:
```bash
kubectl describe clusterrole view

# Output (simplified):
Resources:
  pods, pods/log, pods/status: get, list, watch
  services, endpoints: get, list, watch
  configmaps, secrets: DENIED (not in view role)
```

### **System Service Accounts**

| **Service Account** | **Namespace** | **Purpose** | **Permissions** |
|--------------------|--------------|-------------|----------------|
| `system:kube-controller-manager` | kube-system | Controller manager | Nearly full (manages resources) |
| `system:kube-scheduler` | kube-system | Scheduler | Read pods, nodes; bind pods to nodes |
| `system:node:<nodename>` | N/A | kubelet | Node-specific resources |
| `default` | Every namespace | Default pod SA | **None** (no permissions by default) |

**Critical**: `default` service account has **NO permissions**

```bash
# Pods using default SA cannot access API server
kubectl run test --image=nginx
kubectl exec test -- kubectl get pods
# Error: Forbidden (default SA has no permissions)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔐 Common RBAC Patterns**

### **Pattern 1: Namespace Admin**

**Use Case**: Grant full admin access to a namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-lead-admin
  namespace: production
subjects:
- kind: User
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin  # Built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

**Permissions Granted**:
- ✅ Create/update/delete Pods, Deployments, Services, ConfigMaps, Secrets
- ✅ Manage RBAC (Roles, RoleBindings) within namespace
- ❌ **Cannot** create namespaces
- ❌ **Cannot** access other namespaces
- ❌ **Cannot** manage cluster-scoped resources (Nodes, PVs)

### **Pattern 2: Read-Only Access**

**Use Case**: Monitoring, auditing, debugging

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-view
  namespace: production
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

**Permissions**:
- ✅ Read Pods, Services, ConfigMaps (but NOT Secrets!)
- ✅ View logs: `kubectl logs`
- ❌ **Cannot** read Secrets
- ❌ **Cannot** modify any resources

### **Pattern 3: CI/CD Service Account**

**Use Case**: Deploy applications from CI/CD pipeline

```yaml
# Create service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: production
---
# Define deployment permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]  # Read-only for debugging
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]  # Read logs
---
# Grant to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: cicd-deployer
  namespace: production
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

**Usage in CI/CD**:
```bash
# Get service account token
SA_TOKEN=$(kubectl get secret -n production \
  $(kubectl get sa cicd-deployer -n production -o jsonpath='{.secrets[0].name}') \
  -o jsonpath='{.data.token}' | base64 -d)

# Use token for kubectl
kubectl --token=$SA_TOKEN --namespace=production apply -f deployment.yaml
```

### **Pattern 4: Multi-Namespace Access**

**Use Case**: User needs access to multiple namespaces

**Anti-Pattern**: Multiple RoleBindings
```yaml
# DON'T: Duplicate role per namespace
kind: RoleBinding in dev
kind: RoleBinding in staging
kind: RoleBinding in production
```

**Best Practice**: ClusterRole + Multiple RoleBindings
```yaml
# Define once
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update"]
---
# Grant in dev
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: dev
subjects:
- kind: User
  name: alice@example.com
roleRef:
  kind: ClusterRole
  name: developer
---
# Grant in staging
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: staging
subjects:
- kind: User
  name: alice@example.com
roleRef:
  kind: ClusterRole
  name: developer
```

### **Pattern 5: Specific Resource Names**

**Use Case**: Grant access to specific resources only

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: specific-secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-credentials", "api-keys"]  # Only these secrets
  verbs: ["get"]
```

**Limitation**: `resourceNames` only works with `get`, `update`, `patch`, `delete` (not `list`, `watch`, `create`)

### **Pattern 6: Aggregate Roles**

**Use Case**: Compose roles from multiple smaller roles

```yaml
# Base role 1
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
  labels:
    rbac.example.com/aggregate: "true"
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
# Base role 2
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: service-reader
  labels:
    rbac.example.com/aggregate: "true"
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
---
# Aggregated role (automatically includes both)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: resource-reader
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate: "true"
rules: []  # Automatically populated by controller
```

**How It Works**: ClusterRole controller watches for changes, aggregates rules from matching ClusterRoles

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏢 Multi-Tenancy RBAC Patterns**

### **Soft Multi-Tenancy (Namespace Isolation)**

**Strategy**: One namespace per tenant, RBAC isolates access

```yaml
# Tenant A namespace
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
---
# Tenant A admin
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-a-admin
  namespace: tenant-a
subjects:
- kind: Group
  name: tenant-a-users
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
---
# Prevent tenant-a from accessing tenant-b
# (No RoleBinding in tenant-b namespace)
```

**Enforcement**:
- ✅ RBAC prevents cross-namespace access
- ⚠️ Shared control plane (API server, etcd)
- ⚠️ Shared worker nodes (pods on same node can see each other's processes with `hostPID: true`)

**Additional Isolation**:
```yaml
# Limit tenant resource consumption
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    pods: "50"
---
# Enforce network isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}  # Only pods in same namespace
```

### **Hard Multi-Tenancy (Cluster-per-Tenant)**

**Strategy**: Separate cluster for each tenant

**Pros**:
- ✅ Complete isolation (control plane, etcd, nodes)
- ✅ Independent upgrades
- ✅ Blast radius containment

**Cons**:
- ❌ Higher cost (more clusters)
- ❌ More operational complexity
- ❌ Resource inefficiency (unused capacity per cluster)

**When to Use**: High security requirements, regulatory compliance, large tenants

### **Service Account Isolation**

**Best Practice**: One service account per application

```yaml
# Application 1 service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app1-sa
  namespace: production
---
# Grant minimal permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app1-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app1-config"]  # Only app1's ConfigMap
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app1-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app1-sa
roleRef:
  kind: Role
  name: app1-role
```

**Application 1 Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  template:
    spec:
      serviceAccountName: app1-sa  # Use dedicated SA
      containers:
      - name: app1
        image: app1:latest
```

**Prevents**:
- App1 accessing App2's ConfigMaps/Secrets
- App1 listing all pods in namespace
- App1 escalating privileges

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting RBAC**

### **Tool 1: kubectl auth can-i**

**Check if current user can perform action**:
```bash
# Can I create deployments in production?
kubectl auth can-i create deployments --namespace=production
# Output: yes or no

# Can I delete pods?
kubectl auth can-i delete pods -n production

# Can specific user access resource?
kubectl auth can-i get secrets --namespace=production --as=alice@example.com

# Can service account access resource?
kubectl auth can-i list pods --as=system:serviceaccount:production:app1-sa -n production

# Check all permissions for current user
kubectl auth can-i --list -n production
```

**Example Debugging Session**:
```bash
# User reports: "Cannot create deployment"
$ kubectl auth can-i create deployments -n production --as=bob@example.com
no

# Check what bob CAN do
$ kubectl auth can-i --list -n production --as=bob@example.com
Resources   Verbs
pods        get, list
services    get, list

# Bob only has read access, needs 'create' verb on deployments
```

### **Tool 2: Audit Logs**

**Enable Authorization Audit**:
```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all authorization denials
- level: Request
  omitStages:
  - RequestReceived
  verbs: ["*"]
```

**Query Denials**:
```bash
# Find RBAC denials
cat /var/log/kubernetes/audit.log | \
  jq 'select(.responseStatus.code == 403) | {user: .user.username, resource: .objectRef.resource, verb: .verb, reason: .responseStatus.message}'

# Output:
# {
#   "user": "alice@example.com",
#   "resource": "secrets",
#   "verb": "get",
#   "reason": "Forbidden: User cannot get resource \"secrets\" in API group \"\" in the namespace \"production\""
# }
```

### **Tool 3: Describe RoleBindings**

**Find what permissions a user has**:
```bash
# List all RoleBindings in namespace
kubectl get rolebindings -n production

# Describe specific binding
kubectl describe rolebinding developers-binding -n production

# Output:
# Name:         developers-binding
# Namespace:    production
# Labels:       <none>
# Annotations:  <none>
# Role:
#   Kind:  ClusterRole
#   Name:  edit
# Subjects:
#   Kind   Name                    Namespace
#   ----   ----                    ---------
#   Group  developers
```

**Find all bindings for a user**:
```bash
# ClusterRoleBindings
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.subjects[]? | .name == "alice@example.com") | .metadata.name'

# RoleBindings (all namespaces)
kubectl get rolebindings -A -o json | \
  jq -r '.items[] | select(.subjects[]? | .name == "alice@example.com") | "\(.metadata.namespace)/\(.metadata.name)"'
```

### **Tool 4: rbac-lookup**

**Install**:
```bash
kubectl krew install rbac-lookup
```

**Usage**:
```bash
# Find all permissions for user
kubectl rbac-lookup alice@example.com

# Output:
# SUBJECT                   SCOPE       ROLE
# alice@example.com        production   edit
# alice@example.com        staging      view
# alice@example.com        cluster      cluster-reader

# Find who has access to a resource
kubectl rbac-lookup --kind secret --namespace production

# Output:
# SUBJECT                   SCOPE       ROLE
# system:serviceaccount:production:app1-sa  production  secret-reader
# alice@example.com        production  admin
```

### **Common RBAC Errors**

#### **Error 1: Forbidden**

**Symptom**:
```bash
$ kubectl get pods -n production
Error from server (Forbidden): pods is forbidden: User "bob@example.com" cannot list resource "pods" in API group "" in the namespace "production"
```

**Diagnosis**:
```bash
# Check if user has any permissions
kubectl auth can-i --list -n production --as=bob@example.com

# Check RoleBindings
kubectl get rolebindings -n production -o yaml | grep bob@example.com
```

**Resolution**: Grant appropriate Role/RoleBinding

#### **Error 2: ServiceAccount Cannot Access API**

**Symptom**:
```bash
$ kubectl exec pod -- kubectl get pods
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:default" cannot list resource "pods"
```

**Cause**: Default service account has NO permissions

**Resolution**:
```yaml
# Create service account with permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: app-sa
roleRef:
  kind: Role
  name: pod-reader
---
# Update pod to use service account
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: app-sa  # Use SA with permissions
  containers:
  - name: app
    image: myapp
```

#### **Error 3: ClusterRoleBinding Not Taking Effect**

**Symptom**: Created ClusterRoleBinding but user still denied

**Cause**: RBAC controller cache delay (~1 minute)

**Diagnosis**:
```bash
# Check if binding exists
kubectl get clusterrolebinding my-binding -o yaml

# Verify subjects and roleRef are correct
```

**Resolution**: Wait 1-2 minutes for RBAC controller to sync, or restart API server (not recommended in production)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚠️ RBAC Anti-Patterns**

### **Anti-Pattern 1: Granting cluster-admin**

**Bad**:
```yaml
# DON'T: Give everyone cluster-admin
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers-admin
subjects:
- kind: Group
  name: developers
roleRef:
  kind: ClusterRole
  name: cluster-admin  # Full cluster access!
```

**Why Bad**: cluster-admin can:
- Delete entire cluster
- Read all secrets (including credentials)
- Modify RBAC to escalate privileges
- **No blast radius containment**

**Better**:
```yaml
# Grant minimal permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding  # Namespace-scoped
metadata:
  name: developers-edit
  namespace: dev
subjects:
- kind: Group
  name: developers
roleRef:
  kind: ClusterRole
  name: edit  # Read/write but not RBAC
```

### **Anti-Pattern 2: Using Wildcards Excessively**

**Bad**:
```yaml
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

**Why Bad**: Grants everything, including future resources

**Better**:
```yaml
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch"]
```

### **Anti-Pattern 3: Granting Secrets Access Broadly**

**Bad**:
```yaml
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # All secrets!
```

**Why Bad**: Secrets may contain sensitive data for other apps

**Better**:
```yaml
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app1-db-creds"]  # Specific secret
  verbs: ["get"]
```

### **Anti-Pattern 4: Service Account Token Mounting**

**Bad**:
```yaml
# Default behavior: SA token auto-mounted
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  # automountServiceAccountToken: true (default)
  containers:
  - name: app
    image: myapp
```

**Why Bad**: App doesn't need API access, but token is available

**Better**:
```yaml
# Disable if not needed
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  automountServiceAccountToken: false  # Don't mount token
  containers:
  - name: app
    image: myapp
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Related Documentation**

### **Security Context**
- **[Pod Security Standards](./01-pod-security-standards.md)** - Pod-level security policies
- **[Secrets and Encryption](./03-secrets-and-encryption.md)** - Secret access control
- **[Secret Rotation](./04-secrets-rotation.md)** - Rotating credentials

### **Authentication and Authorization**
- **[API Server Authentication](../apiserver/middle-level/04-authentication.md)** - User authentication
- **[Service Accounts](../controller-manager/18-service-account-controller.md)** - Service account tokens
- **[Admission Control](../apiserver/middle-level/07-admission-control.md)** - Post-authorization validation

### **Audit and Compliance**
- **[Audit Logging](../observability/02-logging-and-analysis.md)** - Authorization audit trail
- **[Network Policy](./06-network-policy-security.md)** - Network-level isolation

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Key Takeaways**

### **For Platform Engineers**

1. **Least Privilege is Critical**:
   - Grant minimal permissions needed
   - Namespace-scoped when possible (Role + RoleBinding)
   - Avoid `cluster-admin` except for break-glass

2. **Default Deny Model**:
   - No permissions unless explicitly granted
   - `default` service account has NO permissions
   - Always create dedicated service accounts for apps

3. **Group-Based Permissions**:
   - Grant to groups, not individual users
   - Easier to manage (add/remove users from group)
   - Example: `developers` group → `edit` ClusterRole

4. **Troubleshooting Tools**:
   - `kubectl auth can-i` - Check permissions
   - Audit logs - Find denial reasons
   - `kubectl rbac-lookup` - Find who has access

5. **Multi-Tenancy Patterns**:
   - Soft: Namespace + RBAC isolation
   - Hard: Separate clusters
   - Additional: ResourceQuota + NetworkPolicy

### **For Kubernetes Contributors**

1. **RBAC Implementation**:
   - Authorization: `plugin/pkg/auth/authorizer/rbac/`
   - RBAC API: `pkg/apis/rbac/`
   - Controller: `pkg/controller/rbac/`

2. **Authorization Flow**:
   - Extract user/group from authentication
   - Find RoleBindings/ClusterRoleBindings for subject
   - Collect rules from Roles/ClusterRoles
   - Match request against rules (apiGroup, resource, verb)

3. **Aggregated Roles**:
   - ClusterRole controller watches for aggregationRule
   - Automatically merges rules from matching ClusterRoles
   - Used for built-in roles (`admin`, `edit`, `view`)

4. **Testing RBAC**:
   - Unit tests: `pkg/auth/authorizer/rbac/rbac_test.go`
   - Integration tests: `test/integration/auth/rbac_test.go`
   - E2E tests: `test/e2e/auth/service_accounts.go`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Kubernetes Version**: v1.30
**Last Updated**: 2024-01-15
**Maintained By**: Kubernetes Architecture Study Group
