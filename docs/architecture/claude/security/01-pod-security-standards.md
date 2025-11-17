# **Pod Security Standards - Deep Architectural Analysis**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Target Audience**: Platform engineers, Kubernetes architects, security engineers, SREs managing production clusters, open-source contributors

**Scope**: Deep architectural analysis of Pod Security Standards (PSS), the built-in admission plugin that replaces deprecated PodSecurityPolicy. This document examines PSS implementation at the source code level, explains the three security profiles, enforcement modes, and provides migration strategies for platform engineers building multi-tenant Kubernetes platforms.

**Prerequisites**:
- Understanding of [API server admission plugins](../apiserver/middle-level/07-admission-control.md)
- Familiarity with [security context and capabilities](./02-security-context-capabilities.md)
- Knowledge of [RBAC patterns](./05-rbac-patterns-troubleshooting.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Design Philosophy**

### **Why Pod Security Standards Exist**

Kubernetes workloads can request dangerous privileges that compromise cluster security:
- **Host namespaces**: Pods sharing host network, PID, or IPC namespaces
- **Privileged containers**: Containers with full root access to the host
- **Host path volumes**: Direct access to host filesystem
- **Privilege escalation**: Processes gaining more privileges than parent

**Historical Context**:
1. **PodSecurityPolicy (PSP)**: Original solution (Kubernetes v1.3-v1.25)
   - Complex RBAC integration
   - Difficult to understand which PSP applies to a pod
   - Deprecated in v1.21, removed in v1.25
2. **Pod Security Admission (PSA)**: Replacement (Kubernetes v1.22+)
   - Built-in admission plugin
   - Simple namespace-level labels
   - Three predefined security profiles

### **Core Design Principles**

```
┌──────────────────────────────────────────────────────────────┐
│  POD SECURITY STANDARDS DESIGN PRINCIPLES                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. SIMPLICITY OVER FLEXIBILITY                              │
│     └─ Predefined profiles instead of custom policies       │
│                                                               │
│  2. DEFENSE IN DEPTH                                         │
│     └─ Layered security: admission + RBAC + NetworkPolicy   │
│                                                               │
│  3. FAIL CLOSED                                              │
│     └─ Violations block pod creation (enforce mode)         │
│                                                               │
│  4. NAMESPACE-SCOPED                                         │
│     └─ Security posture defined per namespace               │
│                                                               │
│  5. AUDIT TRAIL                                              │
│     └─ Violations logged for security monitoring            │
│                                                               │
│  6. GRADUAL ADOPTION                                         │
│     └─ Warn/audit modes before enforcement                  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏗️ Architecture Overview**

### **Pod Security Admission Plugin**

```go
// staging/src/k8s.io/pod-security-admission/admission/admission.go

type Admission struct {
    // Configuration
    Configuration *api.PodSecurityConfiguration

    // Evaluator applies security profiles to pods
    Evaluator Evaluator

    // Metrics
    Metrics *admissionmetrics.Metrics
}

func (a *Admission) Validate(ctx context.Context, attrs admission.Attributes, o admission.ObjectInterfaces) error {
    // Only evaluate Pods
    if attrs.GetResource().GroupResource() != api.Resource("pods") {
        return nil
    }

    pod := attrs.GetObject().(*corev1.Pod)
    namespace := attrs.GetNamespace()

    // Get namespace security configuration from labels
    nsPolicy, err := a.getNamespacePolicy(namespace)

    // Evaluate pod against namespace policy
    result := a.Evaluator.EvaluatePod(nsPolicy, pod)

    // Handle violations based on mode
    return a.handleViolations(result, attrs)
}
```

**Admission Flow**:
```
User submits pod
      ↓
API Server receives request
      ↓
Authentication (who are you?)
      ↓
Authorization (RBAC - are you allowed?)
      ↓
Mutating Admission Plugins
      ↓
Pod Security Admission  ← WE ARE HERE
      │
      ├─ Read namespace labels
      ├─ Determine security profile(s)
      ├─ Evaluate pod spec
      ├─ Enforce / Audit / Warn
      ↓
Validating Admission Plugins
      ↓
Persist to etcd
      ↓
Pod created
```

### **Namespace-Level Configuration**

Security policies are defined via namespace labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production-workloads
  labels:
    # Enforce restricted profile for latest version
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.30

    # Audit against baseline profile (for monitoring)
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/audit-version: v1.30

    # Warn users about restricted violations
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.30
```

**Label Format**:
```
pod-security.kubernetes.io/<MODE>: <PROFILE>
pod-security.kubernetes.io/<MODE>-version: <VERSION>
```

**Modes**:
- `enforce`: Reject pod creation if it violates the profile
- `audit`: Allow pod creation but log violation to audit log
- `warn`: Allow pod creation but return warning to user

**Profiles**:
- `privileged`: Unrestricted (no restrictions)
- `baseline`: Minimally restrictive (prevents known privilege escalations)
- `restricted`: Heavily restricted (current hardening best practices)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔒 Security Profiles**

### **Profile 1: Privileged**

**Definition**: Unrestricted policy, allows all pod configurations

**Use Cases**:
- `kube-system` namespace (control plane components)
- CNI plugin pods (require host networking)
- CSI driver pods (require privileged containers)
- Node monitoring/management tools

**Restrictions**: None (equivalent to no Pod Security Admission)

**Example Allowed Pod**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-debug-pod
  namespace: kube-system
spec:
  hostNetwork: true  # ✓ Allowed
  hostPID: true      # ✓ Allowed
  hostIPC: true      # ✓ Allowed
  containers:
  - name: debug
    image: ubuntu:22.04
    securityContext:
      privileged: true  # ✓ Allowed
      capabilities:
        add: ["ALL"]    # ✓ Allowed
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /  # ✓ Allowed - full host filesystem access
```

### **Profile 2: Baseline**

**Definition**: Minimally restrictive policy that prevents known privilege escalations while allowing common application needs

**Design Goal**: Reasonable default for most non-security-critical applications

**Restrictions**:

| **Category** | **Restriction** | **Rationale** |
|--------------|----------------|---------------|
| **Host Namespaces** | ❌ `hostNetwork: true` forbidden | Pods could sniff network traffic, bind to host ports |
|  | ❌ `hostPID: true` forbidden | Pods could see all host processes, send signals |
|  | ❌ `hostIPC: true` forbidden | Pods could access host IPC resources |
| **Privileged Containers** | ❌ `privileged: true` forbidden | Container has full root access to host |
| **Capabilities** | ❌ Adding dangerous capabilities forbidden | `SYS_ADMIN`, `NET_ADMIN`, etc. could escape container |
|  | ✓ Dropping all capabilities allowed | Recommended best practice |
| **Host Path Volumes** | ❌ `hostPath` volumes forbidden | Direct host filesystem access |
| **Host Ports** | ❌ `hostPort` forbidden | Could conflict with host services |
| **AppArmor** | Must use allowed profiles or undefined | Restrict system calls |
| **SELinux** | Cannot use custom SELinux types | Prevent policy bypass |
| **Proc Mount** | ❌ `Unmasked` proc mount forbidden | Could expose sensitive host info |
| **Seccomp** | Must use `RuntimeDefault`, `Localhost`, or undefined | Restrict syscalls |
| **Sysctls** | Only "safe" sysctls allowed | Unsafe sysctls can affect host kernel |

**Example Allowed Pod**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-app
  namespace: dev-workloads
spec:
  # ✓ No host namespaces
  securityContext:
    # ✓ Pod-level security context allowed
    fsGroup: 2000
    runAsNonRoot: true  # ✓ Recommended
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      # ✓ Run as non-root user
      runAsUser: 1000
      runAsNonRoot: true

      # ✓ Drop all capabilities (best practice)
      capabilities:
        drop: ["ALL"]

      # ✓ Read-only root filesystem (best practice)
      readOnlyRootFilesystem: true

      # ❌ NOT allowed: privileged: true
      # ❌ NOT allowed: capabilities.add: ["SYS_ADMIN"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}  # ✓ Allowed
    # ❌ NOT allowed: hostPath volume
```

**Baseline Violations**:
```yaml
# This pod would be REJECTED by baseline profile

apiVersion: v1
kind: Pod
metadata:
  name: violating-pod
spec:
  hostNetwork: true  # ❌ VIOLATION: host namespace
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      privileged: true  # ❌ VIOLATION: privileged container
```

### **Profile 3: Restricted**

**Definition**: Heavily restricted policy following current hardening best practices

**Design Goal**: Defense-in-depth for security-critical workloads

**All Baseline restrictions PLUS**:

| **Category** | **Additional Restriction** | **Rationale** |
|--------------|---------------------------|---------------|
| **Running as root** | ✅ MUST set `runAsNonRoot: true` | Prevent root execution |
|  | ❌ MUST NOT set `runAsUser: 0` | Explicitly disallow UID 0 |
| **Capabilities** | ✅ MUST drop ALL capabilities | Minimize container privileges |
|  | ❌ MUST NOT add any capabilities | Even "safe" capabilities forbidden |
| **Seccomp** | ✅ MUST set `seccompProfile.type: RuntimeDefault` or `Localhost` | Cannot be undefined |
| **Volume Types** | Only allowed: `configMap`, `downwardAPI`, `emptyDir`, `persistentVolumeClaim`, `projected`, `secret` | Prevents access to host resources |
|  | ❌ All other volume types forbidden | Including `hostPath`, `nfs`, `csi`, etc. |
| **Privilege Escalation** | ✅ MUST set `allowPrivilegeEscalation: false` | Prevent gaining additional privileges |

**Example Compliant Pod**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-app
  namespace: production-sensitive
spec:
  securityContext:
    # ✅ REQUIRED: Pod must run as non-root
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    # ✅ REQUIRED: Seccomp profile
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      # ✅ REQUIRED: Container must run as non-root
      runAsNonRoot: true
      runAsUser: 1000

      # ✅ REQUIRED: Drop all capabilities
      capabilities:
        drop: ["ALL"]

      # ✅ REQUIRED: Prevent privilege escalation
      allowPrivilegeEscalation: false

      # ✅ REQUIRED: Read-only root filesystem (best practice)
      readOnlyRootFilesystem: true

      # ✅ REQUIRED: Seccomp profile (inherited from pod)
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config
    configMap:  # ✓ Allowed volume type
      name: app-config
  # ❌ NOT allowed: emptyDir with medium: Memory (disallowed in some restricted implementations)
```

**Restricted Violations**:
```yaml
# This pod would be REJECTED by restricted profile

apiVersion: v1
kind: Pod
metadata:
  name: violating-pod
spec:
  securityContext:
    runAsUser: 0  # ❌ VIOLATION: running as root
  containers:
  - name: app
    image: myapp:1.0
    # ❌ VIOLATION: missing runAsNonRoot
    # ❌ VIOLATION: missing capabilities.drop
    # ❌ VIOLATION: missing allowPrivilegeEscalation: false
    # ❌ VIOLATION: missing seccompProfile
```

### **Profile Comparison Matrix**

| **Control** | **Privileged** | **Baseline** | **Restricted** |
|-------------|---------------|--------------|----------------|
| Host Namespaces | ✓ Allowed | ❌ Forbidden | ❌ Forbidden |
| Privileged Containers | ✓ Allowed | ❌ Forbidden | ❌ Forbidden |
| Capabilities | ✓ Any | ⚠️ Limited | ❌ Must drop ALL |
| hostPath Volumes | ✓ Allowed | ❌ Forbidden | ❌ Forbidden |
| Host Ports | ✓ Allowed | ❌ Forbidden | ❌ Forbidden |
| runAsNonRoot | ⚠️ Optional | ⚠️ Optional | ✅ Required |
| Privilege Escalation | ⚠️ Optional | ⚠️ Optional | ❌ Forbidden |
| Seccomp | ⚠️ Optional | ⚠️ Optional | ✅ Required |
| Volume Types | ✓ Any | ⚠️ Most | ⚠️ Limited |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚙️ Implementation Details**

### **Admission Plugin Initialization**

```go
// staging/src/k8s.io/pod-security-admission/admission/admission.go

func NewPlugin() *Admission {
    return &Admission{
        Handler: admission.NewHandler(
            admission.Create,
            admission.Update,
        ),
    }
}

func (a *Admission) ValidateInitialization() error {
    // Ensure evaluator is configured
    if a.Evaluator == nil {
        a.Evaluator = policy.DefaultEvaluator()
    }

    // Load exemptions configuration
    if a.Configuration != nil && a.Configuration.Exemptions != nil {
        a.exemptions = a.Configuration.Exemptions
    }

    return nil
}
```

**Enabling Pod Security Admission**:

Since Kubernetes v1.25, Pod Security Admission is **enabled by default**. No API server flags needed.

**Configuration** (optional):
```yaml
# /etc/kubernetes/pod-security-config.yaml
apiVersion: pod-security.admission.config.k8s.io/v1
kind: PodSecurityConfiguration
defaults:
  enforce: "baseline"
  enforce-version: "v1.30"
  audit: "restricted"
  audit-version: "v1.30"
  warn: "restricted"
  warn-version: "v1.30"
exemptions:
  # Exemptions from enforcement
  usernames: []
  runtimeClassNames: []
  namespaces:
    - kube-system
    - kube-public
    - kube-node-lease
```

**API Server Flag** (if using custom configuration):
```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --admission-control-config-file=/etc/kubernetes/admission-config.yaml
```

### **Evaluation Logic**

```go
// staging/src/k8s.io/pod-security-admission/policy/check.go

func (e *Evaluator) EvaluatePod(nsPolicy NamespacePolicy, pod *corev1.Pod) Result {
    var result Result

    // Evaluate against each mode
    for _, mode := range []api.Mode{api.ModeEnforce, api.ModeAudit, api.ModeWarn} {
        level := nsPolicy.GetLevel(mode)
        if level == api.LevelPrivileged {
            continue  // Privileged profile = no checks
        }

        // Get checks for this profile level
        checks := e.getChecksForLevel(level)

        // Run each check
        for _, check := range checks {
            violations := check.Evaluate(pod)
            if len(violations) > 0 {
                result.AddViolations(mode, violations)
            }
        }
    }

    return result
}

// Example check: HostNamespaces
func CheckHostNamespaces(pod *corev1.Pod) []Violation {
    var violations []Violation

    if pod.Spec.HostNetwork {
        violations = append(violations, Violation{
            Field:   "spec.hostNetwork",
            Detail:  "hostNetwork=true is forbidden",
        })
    }

    if pod.Spec.HostPID {
        violations = append(violations, Violation{
            Field:   "spec.hostPID",
            Detail:  "hostPID=true is forbidden",
        })
    }

    if pod.Spec.HostIPC {
        violations = append(violations, Violation{
            Field:   "spec.hostIPC",
            Detail:  "hostIPC=true is forbidden",
        })
    }

    return violations
}
```

### **Enforcement Modes**

#### **Mode 1: Enforce**

```go
func (a *Admission) handleEnforce(result Result, attrs admission.Attributes) error {
    violations := result.GetViolations(api.ModeEnforce)
    if len(violations) == 0 {
        return nil  // No violations, allow pod
    }

    // Build error message
    var messages []string
    for _, v := range violations {
        messages = append(messages, fmt.Sprintf("%s: %s", v.Field, v.Detail))
    }

    // Reject pod creation
    return admission.NewForbidden(attrs, fmt.Errorf(
        "pod violates PodSecurity %q: %s",
        result.EnforceLevel,
        strings.Join(messages, "; "),
    ))
}
```

**User Experience**:
```bash
$ kubectl apply -f pod.yaml
Error from server (Forbidden): error when creating "pod.yaml": pods "my-pod" is forbidden:
violates PodSecurity "restricted:v1.30":
allowPrivilegeEscalation != false (container "app" must set securityContext.allowPrivilegeEscalation=false),
unrestricted capabilities (container "app" must set securityContext.capabilities.drop=["ALL"]),
runAsNonRoot != true (pod or container "app" must set securityContext.runAsNonRoot=true),
seccompProfile (pod or container "app" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

#### **Mode 2: Audit**

```go
func (a *Admission) handleAudit(result Result, attrs admission.Attributes) {
    violations := result.GetViolations(api.ModeAudit)
    if len(violations) == 0 {
        return
    }

    // Log to audit log (if audit logging enabled)
    audit.LogAnnotation(attrs.GetUserInfo(), attrs.GetResource(), map[string]string{
        "pod-security.kubernetes.io/audit-violations": serializeViolations(violations),
    })

    // Increment metrics
    a.Metrics.RecordAuditViolation(result.AuditLevel)
}
```

**Audit Log Entry**:
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Request",
  "auditID": "abc123-def456",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/pods",
  "verb": "create",
  "user": {
    "username": "alice",
    "groups": ["developers"]
  },
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "my-pod"
  },
  "annotations": {
    "pod-security.kubernetes.io/audit-violations": "allowPrivilegeEscalation != false (container \"app\" must set securityContext.allowPrivilegeEscalation=false)"
  }
}
```

**Querying Audit Violations**:
```bash
# If using audit log backend
cat /var/log/kubernetes/audit.log | jq 'select(.annotations["pod-security.kubernetes.io/audit-violations"])'
```

#### **Mode 3: Warn**

```go
func (a *Admission) handleWarn(result Result, attrs admission.Attributes) {
    violations := result.GetViolations(api.ModeWarn)
    if len(violations) == 0 {
        return
    }

    // Build warning message
    warning := fmt.Sprintf(
        "would violate PodSecurity %q: %s",
        result.WarnLevel,
        serializeViolations(violations),
    )

    // Add warning header to response
    // Client will display this to user
    attrs.AddWarning(warning)
}
```

**User Experience**:
```bash
$ kubectl apply -f pod.yaml
Warning: would violate PodSecurity "restricted:v1.30": runAsNonRoot != true (pod or container "app" must set securityContext.runAsNonRoot=true)
pod/my-pod created
```

**kubectl Output**:
- Warnings appear as **yellow text** in terminal
- Pod is still created (non-blocking)
- Warning is also returned in API response headers (`Warning` HTTP header)

### **Multiple Modes Combination**

**Recommended Pattern**: Gradual Tightening

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: application-team
  labels:
    # Enforce baseline now (prevent major issues)
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.30

    # Audit against restricted (monitoring for compliance)
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.30

    # Warn users about restricted violations (educate)
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.30
```

**Migration Path**:
```
Phase 1: Audit restricted, warn restricted, no enforcement
         └─ Identify all violations, fix workloads

Phase 2: Enforce baseline, audit restricted, warn restricted
         └─ Block major security issues, continue fixing for restricted

Phase 3: Enforce restricted, audit restricted, warn restricted
         └─ Full compliance
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Migration from PodSecurityPolicy**

### **PodSecurityPolicy (PSP) Deprecated**

**Timeline**:
- Kubernetes v1.21: PSP deprecated
- Kubernetes v1.25: PSP removed

**Why PSP Failed**:
1. **Complex RBAC integration**: Which PSP applies to a pod depends on RBAC permissions
2. **Unpredictable behavior**: Pod creation could fail with cryptic error "unable to validate against any pod security policy"
3. **Namespace pollution**: PSPs are cluster-scoped but used namespace-scoped
4. **Maintenance burden**: Each organization maintains custom PSPs

### **PSP to PSS Migration Strategy**

#### **Step 1: Analyze Current PSP Usage**

```bash
# List all PodSecurityPolicies
kubectl get psp

# For each PSP, list pods using it
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.annotations.kubernetes\.io/psp}{"\n"}{end}' | grep restricted-psp
```

#### **Step 2: Map PSPs to Pod Security Profiles**

| **PSP Attribute** | **Equivalent in PSS** | **Profile** |
|-------------------|----------------------|-------------|
| `allowPrivilegeEscalation: false` | Required | `restricted` |
| `requiredDropCapabilities: [ALL]` | Required | `restricted` |
| `runAsUser.rule: MustRunAsNonRoot` | Required | `restricted` |
| `seLinux` | Custom profile forbidden | `baseline` |
| `allowedHostPaths: []` | hostPath forbidden | `baseline` |
| `hostNetwork: false` | Forbidden | `baseline` |
| `hostPID: false` | Forbidden | `baseline` |
| `hostIPC: false` | Forbidden | `baseline` |
| `privileged: false` | Forbidden | `baseline` |

**Example PSP**:
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
```

**Equivalent PSS**: `restricted` profile

#### **Step 3: Add PSS Labels to Namespaces**

```bash
#!/bin/bash
# migrate-to-pss.sh

# For each namespace currently using restricted-psp
for NS in $(kubectl get ns -o name | cut -d/ -f2); do
    # Check which PSP is used by pods in this namespace
    PSP=$(kubectl get pods -n $NS -o jsonpath='{.items[0].metadata.annotations.kubernetes\.io/psp}' 2>/dev/null)

    if [ "$PSP" = "restricted-psp" ]; then
        # Start with warn mode
        kubectl label ns $NS \
            pod-security.kubernetes.io/warn=restricted \
            pod-security.kubernetes.io/warn-version=v1.30 \
            pod-security.kubernetes.io/audit=restricted \
            pod-security.kubernetes.io/audit-version=v1.30

        echo "Namespace $NS: Added warn/audit for restricted profile"
    fi
done
```

#### **Step 4: Validate and Fix Violations**

```bash
# Test pod creation in each namespace
kubectl run test-pod --image=nginx -n application-team --dry-run=server

# If warnings appear, fix pod specs
# Common fixes:
# - Add securityContext.runAsNonRoot: true
# - Add securityContext.allowPrivilegeEscalation: false
# - Add securityContext.capabilities.drop: ["ALL"]
# - Add securityContext.seccompProfile.type: RuntimeDefault
```

#### **Step 5: Enable Enforcement**

```bash
# After all violations fixed, enable enforcement
kubectl label ns application-team \
    pod-security.kubernetes.io/enforce=restricted \
    pod-security.kubernetes.io/enforce-version=v1.30 \
    --overwrite
```

#### **Step 6: Remove PSP (Kubernetes v1.25+)**

```bash
# Delete PodSecurityPolicies (no longer functional)
kubectl delete psp --all

# Delete PSP-related RBAC
kubectl delete clusterrole psp:restricted
kubectl delete clusterrolebinding psp:restricted
```

### **Migration Challenges**

| **Challenge** | **PSP Behavior** | **PSS Behavior** | **Solution** |
|---------------|-----------------|------------------|-------------|
| Default allow | PSP grants permissions | PSS blocks by default | Explicitly label all namespaces |
| Service accounts | PSP bound via RBAC | PSS namespace-scoped | No change needed |
| Privileged namespaces | PSP allows via bindings | PSS `privileged` profile | Label kube-system as `privileged` |
| Custom policies | PSP supports custom rules | PSS only 3 profiles | Use external admission controller (OPA, Kyverno) for custom policies |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🛠️ Operational Patterns**

### **Pattern 1: Progressive Enforcement**

**Use Case**: Gradually tighten security in existing clusters without breaking workloads

**Implementation**:
```yaml
# Week 1: Visibility only
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline

# Week 3: After reviewing audit logs
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    pod-security.kubernetes.io/enforce: baseline  # Added
    pod-security.kubernetes.io/audit: restricted  # Tightened
    pod-security.kubernetes.io/warn: restricted   # Tightened

# Week 6: After fixing all restricted violations
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    pod-security.kubernetes.io/enforce: restricted  # Tightened
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### **Pattern 2: Tiered Security Model**

**Use Case**: Different security postures for different workload classes

**Implementation**:
```yaml
# Tier 1: Privileged infrastructure
apiVersion: v1
kind: Namespace
metadata:
  name: infrastructure
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
# Tier 2: Standard applications
apiVersion: v1
kind: Namespace
metadata:
  name: applications
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
---
# Tier 3: Sensitive data processing
apiVersion: v1
kind: Namespace
metadata:
  name: pci-compliant-apps
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

### **Pattern 3: Exemptions for Specific Workloads**

**Problem**: Some pods legitimately need elevated privileges (monitoring, CNI)

**Solution 1**: Dedicated privileged namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring-privileged
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

**Solution 2**: Configuration-based exemptions
```yaml
# /etc/kubernetes/pod-security-config.yaml
apiVersion: pod-security.admission.config.k8s.io/v1
kind: PodSecurityConfiguration
exemptions:
  namespaces:
    - kube-system
    - monitoring-privileged
  runtimeClassNames:
    - trusted-runtime
  usernames:
    - system:serviceaccount:monitoring:privileged-sa
```

**Solution 3**: External admission controller (for complex policies)
```yaml
# Use OPA Gatekeeper or Kyverno for fine-grained exemptions
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: allow-privileged-for-monitoring
spec:
  rules:
  - name: allow-node-exporter
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - monitoring
          names:
          - node-exporter-*
    exclude:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "node-exporter pods are exempt from Pod Security Standards"
      pattern:
        spec:
          =(hostNetwork): true
```

### **Pattern 4: Audit-Driven Compliance**

**Use Case**: Monitor compliance without blocking workloads

**Implementation**:
```yaml
# Enforce baseline, but audit against restricted
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
```

**Compliance Monitoring**:
```bash
# Query audit logs for violations
cat /var/log/kubernetes/audit.log | \
  jq 'select(.annotations["pod-security.kubernetes.io/audit-violations"] != null) |
      {namespace: .objectRef.namespace, pod: .objectRef.name, violations: .annotations["pod-security.kubernetes.io/audit-violations"]}'

# Output:
# {
#   "namespace": "production",
#   "pod": "legacy-app",
#   "violations": "runAsNonRoot != true"
# }
```

**Metrics**:
```promql
# Prometheus metrics (if exposed)
pod_security_evaluations_total{decision="deny", mode="enforce", policy_level="baseline"}
pod_security_evaluations_total{decision="deny", mode="audit", policy_level="restricted"}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Troubleshooting**

### **Issue 1: Pod Rejected by Baseline Profile**

**Symptom**:
```bash
$ kubectl apply -f app.yaml
Error from server (Forbidden): error when creating "app.yaml": pods "my-app" is forbidden:
violates PodSecurity "baseline:v1.30": hostNetwork (spec.hostNetwork=true is forbidden)
```

**Root Cause**: Pod requires `hostNetwork: true` but baseline forbids host namespaces

**Resolution Options**:

**Option 1**: Fix pod spec (recommended)
```yaml
spec:
  # hostNetwork: true  # Remove this
  containers:
  - name: app
    # Use service discovery instead of host networking
```

**Option 2**: Move to privileged namespace
```bash
kubectl create namespace infrastructure
kubectl label namespace infrastructure pod-security.kubernetes.io/enforce=privileged
kubectl apply -f app.yaml -n infrastructure
```

**Option 3**: Add exemption (least recommended)
```yaml
# /etc/kubernetes/pod-security-config.yaml
exemptions:
  namespaces:
    - my-namespace
```

### **Issue 2: Pod Rejected by Restricted Profile**

**Symptom**:
```bash
$ kubectl run nginx --image=nginx -n production
Error from server (Forbidden): pods "nginx" is forbidden: violates PodSecurity "restricted:v1.30":
runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true),
unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]),
allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false),
seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

**Root Cause**: Default nginx image runs as root, doesn't meet restricted requirements

**Resolution**: Add compliant security context
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534  # nobody user
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx:1.25
    securityContext:
      runAsNonRoot: true
      runAsUser: 65534
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

**Or use a restricted-compliant image**:
```yaml
spec:
  containers:
  - name: nginx
    image: nginxinc/nginx-unprivileged:1.25  # Runs as UID 101
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

### **Issue 3: Deployment Rollout Fails**

**Symptom**:
```bash
$ kubectl rollout status deployment/myapp
Waiting for deployment "myapp" rollout to finish: 0 of 3 updated replicas are available...
```

**Investigation**:
```bash
$ kubectl get events -n production --field-selector involvedObject.name=myapp
Warning  FailedCreate  15s   replicaset-controller  Error creating: pods "myapp-xyz" is forbidden: violates PodSecurity "restricted:v1.30"
```

**Root Cause**: Deployment spec violates namespace security policy

**Resolution**: Update Deployment template
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      securityContext:  # Add pod-level security context
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: myapp:1.0
        securityContext:  # Add container-level security context
          runAsNonRoot: true
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
```

### **Issue 4: Unclear Which Mode is Blocking**

**Symptom**: Pod creation fails, not clear if enforce/audit/warn is blocking

**Investigation**:
```bash
# Check namespace labels
kubectl get ns production -o yaml | grep pod-security

# Output:
#   pod-security.kubernetes.io/enforce: restricted
#   pod-security.kubernetes.io/audit: restricted
#   pod-security.kubernetes.io/warn: restricted
```

**Clarification**: Only `enforce` mode blocks pod creation. `audit` and `warn` are non-blocking.

**Testing**:
```bash
# Temporarily relax enforcement to identify violations
kubectl label ns production pod-security.kubernetes.io/enforce=privileged --overwrite

# Try creating pod (should succeed now)
kubectl apply -f pod.yaml

# Check warnings (will show restricted violations)
kubectl apply -f pod.yaml
# Warning: would violate PodSecurity "restricted:v1.30": ...

# Restore enforcement
kubectl label ns production pod-security.kubernetes.io/enforce=restricted --overwrite
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 Related Documentation**

### **Security Context**
- **[Security Context and Capabilities](./02-security-context-capabilities.md)** - Container-level security configuration
- **[Secrets and Encryption](./03-secrets-and-encryption.md)** - Protecting sensitive data
- **[RBAC Patterns](./05-rbac-patterns-troubleshooting.md)** - Authorization and access control

### **Admission Control**
- **[API Server Admission Control](../apiserver/middle-level/07-admission-control.md)** - Admission plugin architecture
- **[Validating Admission Webhooks](../extension-architecture/high-level/02-admission-webhooks.md)** - External policy enforcement

### **Compliance and Governance**
- **[Audit Logging](../observability/02-logging-and-analysis.md)** - Audit trail for security events
- **[Network Policy](./06-network-policy-security.md)** - Network-level security controls

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Key Takeaways**

### **For Platform Engineers**

1. **Pod Security Standards Replace PodSecurityPolicy**:
   - PSP deprecated in v1.21, removed in v1.25
   - PSS is simpler: namespace labels instead of RBAC bindings
   - Three profiles: `privileged`, `baseline`, `restricted`

2. **Namespace-Scoped Configuration**:
   - Security posture defined per namespace via labels
   - Multiple modes: `enforce` (block), `audit` (log), `warn` (notify)
   - Can combine modes for gradual adoption

3. **Restricted Profile is Defense-in-Depth**:
   - Requires `runAsNonRoot`, drops all capabilities, enables seccomp
   - Prevents privilege escalation
   - Best practice for production workloads

4. **Migration Strategy**:
   - Phase 1: Add `warn` and `audit` labels (non-blocking)
   - Phase 2: Fix all violations
   - Phase 3: Enable `enforce` mode

5. **Exemptions for Infrastructure**:
   - `kube-system` should use `privileged` profile
   - CNI/CSI pods need host access
   - Use dedicated namespaces for privileged workloads

### **For Kubernetes Contributors**

1. **Implementation Location**:
   - Admission plugin: `staging/src/k8s.io/pod-security-admission/`
   - Policy checks: `staging/src/k8s.io/pod-security-admission/policy/`
   - Profile definitions: `staging/src/k8s.io/pod-security-admission/api/`

2. **Evaluation Architecture**:
   - Namespace labels define policy
   - Evaluator runs checks for each profile level
   - Results handled differently per mode (enforce/audit/warn)

3. **Extension Points**:
   - Custom profiles: NOT supported (use external admission controller)
   - Exemptions: Configure via PodSecurityConfiguration
   - Metrics: `pod_security_evaluations_total`

4. **Testing PSS Changes**:
   - Unit tests: `staging/src/k8s.io/pod-security-admission/test/`
   - Integration tests: `test/integration/auth/podsecuritypolicy/`
   - E2E tests: `test/e2e/auth/pod_security_admission.go`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Kubernetes Version**: v1.30
**Last Updated**: 2024-01-15
**Maintained By**: Kubernetes Architecture Study Group
