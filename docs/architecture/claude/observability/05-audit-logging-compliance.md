# **Audit Logging and Compliance in Kubernetes**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Document Overview**

**Purpose**: Comprehensive guide to Kubernetes audit logging for security, compliance, and forensic analysis

**Target Audience**:
- Security engineers implementing audit controls
- Compliance teams meeting regulatory requirements
- Platform engineers managing multi-tenant clusters
- Incident responders investigating security events

**Scope**: Audit architecture, policy configuration, backend implementations, compliance patterns, and production best practices

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Why Audit Logging Matters**

### **The Audit Logging Imperative**

Kubernetes audit logs provide a **security-relevant chronological set of records** documenting:
- **Who** performed actions (user, service account, system component)
- **What** actions were performed (create, delete, update, exec)
- **When** actions occurred (microsecond timestamps)
- **Where** actions originated (source IP, user agent)
- **On What** resources actions were performed (pods, secrets, configmaps)
- **What Happened** as a result (success, failure, status codes)

### **Compliance Requirements**

| **Standard** | **Audit Requirements** | **Kubernetes Implementation** |
|--------------|------------------------|-------------------------------|
| **SOC 2** | Audit trail of all system changes | RequestResponse logging for mutations |
| **PCI DSS 10.x** | Access logging for cardholder data | Metadata logging for secret access |
| **HIPAA** | PHI access audit trails | Resource-specific audit policies |
| **GDPR Article 30** | Processing activity records | User-level audit tracking |
| **ISO 27001** | Security event logging | Comprehensive event capture |
| **FedRAMP** | Government audit requirements | Full request/response logging |

### **Security Use Cases**

**Real-World Incident**: Unauthorized Secret Access

```
Timeline:
15:23:42 - User "contractor-a" lists secrets in prod namespace
15:23:45 - User "contractor-a" reads secret "database-credentials"
15:24:10 - New connection from unknown IP to production database
15:25:00 - Database shows unauthorized queries

Audit Evidence:
{
  "user": {"username": "contractor-a"},
  "verb": "get",
  "objectRef": {"resource": "secrets", "name": "database-credentials"},
  "sourceIPs": ["192.168.100.45"],
  "userAgent": "kubectl/v1.28.0",
  "responseStatus": {"code": 200}
}

Resolution:
- Revoked contractor's access
- Rotated compromised credentials
- Implemented secret-specific RBAC
- Added real-time alerting on secret access
- Initiated compliance investigation
```

### **Audit Logging ROI**

| **Area** | **Without Audit Logs** | **With Audit Logs** |
|----------|------------------------|---------------------|
| **Incident Response** | "We think this happened..." | Precise timeline with evidence |
| **Compliance Audit** | Manual evidence gathering | Automated compliance reports |
| **Security Investigation** | Limited visibility | Complete activity history |
| **Forensic Analysis** | Incomplete reconstruction | Full request/response capture |
| **Anomaly Detection** | Manual review | Automated ML-based detection |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Audit Event Architecture**

### **Event Structure**

**File**: `staging/src/k8s.io/apiserver/pkg/apis/audit/v1/types.go`

```go
type Event struct {
    // Event classification
    Level   Level      // None, Metadata, Request, RequestResponse
    AuditID types.UID  // Unique identifier for this audit event
    Stage   Stage      // RequestReceived, ResponseStarted, ResponseComplete, Panic

    // Request identification
    RequestURI string            // Full request URI
    Verb       string            // get, list, create, update, delete, patch, etc.
    User       authnv1.UserInfo  // Authenticated user information
    ImpersonatedUser *authnv1.UserInfo  // If request uses impersonation

    // Request context
    SourceIPs []string  // Client IP addresses (including X-Forwarded-For)
    UserAgent string    // Client user agent string

    // Object reference
    ObjectRef *ObjectReference {
        Resource        string  // pods, deployments, secrets, etc.
        Namespace       string  // Target namespace
        Name            string  // Resource name
        UID             types.UID
        APIGroup        string  // "", "apps", "batch", etc.
        APIVersion      string  // v1, v1beta1, etc.
        ResourceVersion string
        Subresource     string  // exec, logs, status, scale, etc.
    }

    // Timing information
    RequestReceivedTimestamp metav1.MicroTime  // Request arrival time
    StageTimestamp          metav1.MicroTime   // Current stage time

    // Request/Response data (based on Level)
    RequestObject  *runtime.Unknown  // Included at Level >= Request
    ResponseObject *runtime.Unknown  // Included at Level >= RequestResponse
    ResponseStatus *metav1.Status    // HTTP status and reason

    // Additional context
    Annotations map[string]string  // Plugin-provided metadata
}
```

### **Audit Levels**

| **Level** | **Records** | **Use Cases** | **Storage Impact** |
|-----------|-------------|---------------|-------------------|
| **None** | Nothing | Health checks, system noise | 0% |
| **Metadata** | Request metadata only | General activity tracking | 1-5% baseline |
| **Request** | Metadata + request body | Mutation tracking | 5-20% increase |
| **RequestResponse** | Metadata + request + response | Forensic analysis | 20-100% increase |

**Example Progression**:

```yaml
# Level: None
# (No audit event generated)

# Level: Metadata
{
  "verb": "create",
  "objectRef": {"resource": "pods", "name": "nginx", "namespace": "default"},
  "user": {"username": "admin"},
  "responseStatus": {"code": 201}
}

# Level: Request
{
  "verb": "create",
  "objectRef": {...},
  "user": {...},
  "requestObject": {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {"name": "nginx"},
    "spec": {"containers": [...]}
  }
}

# Level: RequestResponse
{
  "verb": "create",
  "objectRef": {...},
  "user": {...},
  "requestObject": {...},
  "responseObject": {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {"uid": "abc-123", "resourceVersion": "12345"},
    "status": {"phase": "Pending"}
  }
}
```

### **Audit Stages**

| **Stage** | **When Logged** | **Purpose** |
|-----------|-----------------|-------------|
| **RequestReceived** | Request arrives at API server | Track all incoming requests |
| **ResponseStarted** | Response headers sent (before body) | Long-running requests (watch, exec) |
| **ResponseComplete** | Full response sent | Complete request lifecycle |
| **Panic** | Handler panic recovery | Error scenarios |

**Example Multi-Stage Event** (watch request):

```json
// Stage 1: RequestReceived
{
  "stage": "RequestReceived",
  "auditID": "abc-123",
  "verb": "watch",
  "requestReceivedTimestamp": "2024-01-15T10:30:00.000000Z"
}

// Stage 2: ResponseStarted (connection established)
{
  "stage": "ResponseStarted",
  "auditID": "abc-123",
  "stageTimestamp": "2024-01-15T10:30:00.050000Z"
}

// Stage 3: ResponseComplete (watch closed)
{
  "stage": "ResponseComplete",
  "auditID": "abc-123",
  "stageTimestamp": "2024-01-15T10:35:00.000000Z"
}
```

### **User Information Capture**

```go
type UserInfo struct {
    Username string         // "admin", "system:serviceaccount:default:myapp"
    UID      string         // Unique user identifier
    Groups   []string       // ["system:masters", "system:authenticated"]
    Extra    map[string]ExtraValue  // Additional authentication data
}
```

**Example User Formats**:

```json
// Human user
{
  "username": "jane.doe@company.com",
  "groups": ["developers", "system:authenticated"]
}

// Service account
{
  "username": "system:serviceaccount:kube-system:deployment-controller",
  "uid": "abc-123-def-456",
  "groups": ["system:serviceaccounts", "system:serviceaccounts:kube-system"]
}

// Impersonated user
{
  "username": "admin",
  "groups": ["system:masters"],
  "impersonatedUser": {
    "username": "developer",
    "groups": ["developers"]
  }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📜 Audit Policy Configuration**

### **Policy Structure**

**File**: `staging/src/k8s.io/apiserver/pkg/apis/audit/v1/types.go`

```go
type Policy struct {
    // Ordered list of audit rules - FIRST MATCH WINS
    Rules []PolicyRule

    // Global stage omissions
    OmitStages []Stage  // e.g., ["RequestReceived"]

    // Global field omissions
    OmitManagedFields bool  // Strip .metadata.managedFields
}

type PolicyRule struct {
    // Audit level for matching requests
    Level Level

    // User matching (OR logic within each list)
    Users      []string  // ["admin", "system:*"]
    UserGroups []string  // ["system:masters"]

    // Request matching
    Verbs      []string  // ["create", "update", "delete"]
    Namespaces []string  // ["default", "kube-system"]

    // Resource matching (supports wildcards)
    Resources []GroupResources {
        Group         string    // "", "apps", "batch"
        Resources     []string  // ["pods", "pods/*"]
        ResourceNames []string  // ["critical-pod"]
    }

    // Non-resource URL matching
    NonResourceURLs []string  // ["/api", "/healthz*"]

    // Rule-specific omissions
    OmitStages        []Stage
    OmitManagedFields *bool  // Override global setting
}
```

### **Policy Evaluation Logic**

**File**: `staging/src/k8s.io/apiserver/pkg/audit/policy/checker.go`

```go
func (p *policyRuleEvaluator) EvaluatePolicyRule(attrs authorizer.Attributes) {
    // Iterate rules in order
    for i, rule := range p.Rules {
        if ruleMatches(&rule, attrs) {
            return RequestAuditConfig{
                Level:             rule.Level,
                OmitStages:        combineOmitStages(p.OmitStages, rule.OmitStages),
                OmitManagedFields: resolveOmitManagedFields(&rule, p.OmitManagedFields),
            }
        }
    }

    // No match - default to Level: None
    return RequestAuditConfig{Level: auditinternal.LevelNone}
}

func ruleMatches(rule *auditinternal.PolicyRule, attrs authorizer.Attributes) bool {
    // All conditions must match (AND logic between types)
    return checkUsers(rule, attrs) &&
           checkUserGroups(rule, attrs) &&
           checkVerbs(rule, attrs) &&
           checkResources(rule, attrs) &&
           checkNonResourceURLs(rule, attrs) &&
           checkNamespaces(rule, attrs)
}
```

### **Production Audit Policy Examples**

#### **Minimal Production Policy**

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# Reduce log volume by omitting early stages
omitStages:
  - RequestReceived
  - ResponseStarted

rules:
  # 1. Don't log health checks
  - level: None
    nonResourceURLs:
      - /healthz*
      - /readyz*
      - /livez*

  # 2. Don't log system component read-only operations
  - level: None
    users:
      - system:kube-proxy
      - system:kube-scheduler
      - system:kube-controller-manager
    verbs: ["get", "list", "watch"]

  # 3. Metadata-level audit for secret/configmap access
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
    omitStages:
      - RequestReceived

  # 4. Request-level for secret mutations (but not body)
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["secrets"]

  # 5. Full audit for privileged operations
  - level: RequestResponse
    resources:
      - group: ""
        resources:
          - pods/exec
          - pods/attach
          - pods/portforward
          - pods/proxy

  # 6. Request-level for resource mutations
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    omitManagedFields: true

  # 7. Metadata for everything else
  - level: Metadata
```

#### **Compliance-Focused Policy**

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived

rules:
  # Health checks - no audit
  - level: None
    nonResourceURLs: ["/healthz*", "/readyz*", "/livez*"]

  # SOC 2: Full audit trail for production namespace
  - level: RequestResponse
    namespaces: ["production"]
    verbs: ["create", "update", "patch", "delete"]

  # PCI DSS 10.2: Secrets access tracking
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
    annotations:
      compliance: "PCI-DSS-10.2.1"

  # HIPAA: PHI-labeled resource access
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["configmaps"]
        resourceNames: ["phi-*"]  # Naming convention

  # Privileged operations - full audit
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
    annotations:
      compliance: "privileged-access"

  # Service account token creation
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["serviceaccounts/token"]

  # RBAC changes - full audit
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]

  # Admission webhook configuration changes
  - level: RequestResponse
    resources:
      - group: "admissionregistration.k8s.io"

  # Default metadata level
  - level: Metadata
    omitManagedFields: true
```

#### **High-Volume Cluster Policy**

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# Aggressively reduce volume
omitStages:
  - RequestReceived
  - ResponseStarted
omitManagedFields: true

rules:
  # No audit for read-only system components
  - level: None
    users: ["system:*"]
    verbs: ["get", "list", "watch"]

  # No audit for health/metrics endpoints
  - level: None
    nonResourceURLs: ["/*"]
    verbs: ["get"]

  # Metadata only for secrets (reduce storage)
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]

  # Critical mutations only
  - level: Request
    verbs: ["create", "update", "delete"]
    resources:
      - group: ""
        resources: ["pods", "services", "persistentvolumeclaims"]
      - group: "apps"
        resources: ["deployments", "statefulsets"]

  # Privileged operations
  - level: Metadata  # Not RequestResponse to save space
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]

  # Everything else - None
  - level: None
```

### **Policy Testing**

```bash
# Test policy evaluation
kubectl create -f audit-policy.yaml --dry-run=server -v=8

# Enable audit logging on test cluster
kube-apiserver \
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \
  --audit-log-path=/var/log/kubernetes/audit.log \
  --audit-log-maxage=30

# Perform test operations
kubectl get pods  # Should match a rule
kubectl create secret generic test --from-literal=key=value  # Check secret rule

# Verify audit events
cat /var/log/kubernetes/audit.log | jq '.verb, .objectRef.resource, .level'
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔌 Audit Backend Implementations**

### **1. Log Backend**

**File**: `staging/src/k8s.io/apiserver/plugin/pkg/audit/log/backend.go`

#### **Configuration**

```go
const (
    FormatLegacy = "legacy"  // One-line text format
    FormatJson   = "json"    // Structured JSON
)

// API server flags
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-format=json
--audit-log-maxage=30      // Days to retain
--audit-log-maxbackup=10   // Number of old files
--audit-log-maxsize=100    // MB per file
--audit-log-compress=true  // Compress rotated files
```

#### **Implementation**

```go
type backend struct {
    out     io.Writer        // Output destination
    format  string           // json or legacy
    encoder runtime.Encoder  // JSON encoder
}

func (b *backend) ProcessEvents(ev ...*auditinternal.Event) bool {
    for _, e := range ev {
        if b.format == FormatLegacy {
            b.logLegacyEvent(e)
        } else {
            b.encoder.Encode(e, b.out)
            b.out.Write([]byte("\n"))
        }
    }
    return true  // Success
}
```

#### **Output Formats**

**JSON Format**:
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "a1b2c3d4-e5f6-g7h8-i9j0-k1l2m3n4o5p6",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/pods",
  "verb": "list",
  "user": {
    "username": "admin",
    "groups": ["system:masters", "system:authenticated"]
  },
  "sourceIPs": ["10.0.0.1"],
  "userAgent": "kubectl/v1.28.0 (darwin/amd64) kubernetes/1234abc",
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "metadata": {},
    "code": 200
  },
  "requestReceivedTimestamp": "2024-01-15T10:30:45.123456Z",
  "stageTimestamp": "2024-01-15T10:30:45.234567Z",
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": "RBAC: allowed by ClusterRoleBinding"
  }
}
```

**Legacy Format**:
```
2024-01-15T10:30:45.234567Z AUDIT: id="a1b2c3d4" stage="ResponseComplete" ip="10.0.0.1" method="list" user="admin" groups="system:masters" as="<self>" asgroups="<lookup>" namespace="default" uri="/api/v1/namespaces/default/pods" response="200"
```

#### **Log Rotation**

Uses **lumberjack** library:

```go
import "gopkg.in/natefinch/lumberjack.v2"

logger := &lumberjack.Logger{
    Filename:   "/var/log/kubernetes/audit.log",
    MaxSize:    100,  // MB
    MaxBackups: 10,   // Files
    MaxAge:     30,   // Days
    Compress:   true, // Gzip old files
}
```

**Result**:
```bash
/var/log/kubernetes/
├── audit.log                    # Current
├── audit.log.2024-01-15.gz     # Rotated
├── audit.log.2024-01-14.gz
└── audit.log.2024-01-13.gz
```

### **2. Webhook Backend**

**File**: `staging/src/k8s.io/apiserver/plugin/pkg/audit/webhook/webhook.go`

#### **Configuration**

```bash
# API server flags
--audit-webhook-config-file=/etc/kubernetes/audit-webhook-config.yaml
--audit-webhook-initial-backoff=10s
--audit-webhook-batch-buffer-size=10000
--audit-webhook-batch-max-size=400
--audit-webhook-batch-max-wait=30s
--audit-webhook-batch-throttle-qps=10
--audit-webhook-batch-throttle-burst=15
```

**Webhook Config File**:

```yaml
# /etc/kubernetes/audit-webhook-config.yaml
apiVersion: v1
kind: Config
clusters:
- name: audit-receiver
  cluster:
    server: https://audit-collector.example.com:8443/events
    certificate-authority: /etc/kubernetes/pki/audit-ca.crt
users:
- name: audit-sender
  user:
    client-certificate: /etc/kubernetes/pki/audit-client.crt
    client-key: /etc/kubernetes/pki/audit-client.key
current-context: audit-webhook
contexts:
- name: audit-webhook
  context:
    cluster: audit-receiver
    user: audit-sender
```

#### **Implementation**

```go
func NewBackend(
    kubeConfigFile string,
    groupVersion schema.GroupVersion,
    retryBackoff wait.Backoff,
) (audit.Backend, error) {
    // Load webhook config
    config, err := loadConfig(kubeConfigFile)

    // Create REST client
    restClient, err := rest.RESTClientFor(config)

    // Create webhook with retry logic
    webhook := &backend{
        w: webhook.NewWithRetry(
            restClient,
            retryBackoff,
        ),
    }

    return webhook, nil
}

func (b *backend) ProcessEvents(ev ...*auditinternal.Event) bool {
    // Batch events into EventList
    var list auditinternal.EventList
    for _, e := range ev {
        list.Items = append(list.Items, *e)
    }

    // Send with exponential backoff
    err := b.w.WithExponentialBackoff(context.Background(), func() error {
        return b.restClient.Post().
            Body(&list).
            Do(context.Background()).
            Error()
    })

    return err == nil
}
```

#### **Retry Configuration**

```go
retryBackoff := wait.Backoff{
    Duration: 10 * time.Second,  // Initial delay
    Factor:   1.5,                // Backoff multiplier
    Jitter:   0.2,                // Random jitter
    Steps:    5,                  // Max retry attempts
    Cap:      1 * time.Minute,    // Max delay
}

// Retry sequence:
// Attempt 1: 10s
// Attempt 2: 15s (10 * 1.5)
// Attempt 3: 22.5s (15 * 1.5)
// Attempt 4: 33.75s (22.5 * 1.5)
// Attempt 5: 50.6s (33.75 * 1.5)
// After 5 attempts: Event dropped
```

#### **Webhook Receiver Example**

```go
type AuditWebhookReceiver struct {
    storage AuditStorage
}

func (r *AuditWebhookReceiver) HandleEvents(w http.ResponseWriter, req *http.Request) {
    var eventList auditv1.EventList

    // Decode request
    if err := json.NewDecoder(req.Body).Decode(&eventList); err != nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    // Process events
    for _, event := range eventList.Items {
        if err := r.storage.Store(event); err != nil {
            log.Printf("Failed to store event %s: %v", event.AuditID, err)
            http.Error(w, "Storage error", http.StatusInternalServerError)
            return
        }
    }

    // Success
    w.WriteHeader(http.StatusOK)
}
```

### **3. Buffered Backend Wrapper**

**File**: `staging/src/k8s.io/apiserver/plugin/pkg/audit/buffered/buffered.go`

#### **Purpose**

Wraps log or webhook backends to provide:
- **Asynchronous processing** - Non-blocking event collection
- **Batching** - Reduce overhead by grouping events
- **Rate limiting** - Prevent overwhelming downstream systems
- **Buffer overflow protection** - Drop events gracefully under load

#### **Configuration**

```go
type BatchConfig struct {
    BufferSize    int           // Queue size (default: 10000)
    MaxBatchSize  int           // Events per batch (default: 400)
    MaxBatchWait  time.Duration // Max time between batches (default: 30s)

    ThrottleEnable bool    // Enable rate limiting
    ThrottleQPS   float32  // Events/sec (default: 10)
    ThrottleBurst int      // Burst capacity (default: 15)

    AsyncDelegate bool  // Process batches asynchronously
}
```

#### **Implementation**

```go
type bufferedBackend struct {
    delegateBackend audit.Backend
    buffer          chan *auditinternal.Event
    maxBatchSize    int
    maxBatchWait    time.Duration
    throttle        flowcontrol.RateLimiter
}

func (b *bufferedBackend) ProcessEvents(ev ...*auditinternal.Event) bool {
    // Non-blocking send to buffer
    for _, e := range ev {
        select {
        case b.buffer <- e:
            // Buffered successfully
        default:
            // Buffer full - drop event
            metrics.DroppedCounter.Inc()
            return false
        }
    }
    return true
}

func (b *bufferedBackend) Run(stopCh <-chan struct{}) {
    timer := time.NewTimer(b.maxBatchWait)
    defer timer.Stop()

    var batch []*auditinternal.Event

    for {
        select {
        case ev := <-b.buffer:
            batch = append(batch, ev)

            // Send batch if full
            if len(batch) >= b.maxBatchSize {
                b.sendBatch(batch)
                batch = nil
                timer.Reset(b.maxBatchWait)
            }

        case <-timer.C:
            // Send partial batch on timeout
            if len(batch) > 0 {
                b.sendBatch(batch)
                batch = nil
            }
            timer.Reset(b.maxBatchWait)

        case <-stopCh:
            // Flush remaining events
            if len(batch) > 0 {
                b.sendBatch(batch)
            }
            return
        }
    }
}

func (b *bufferedBackend) sendBatch(events []*auditinternal.Event) {
    // Rate limiting
    if b.throttle != nil {
        b.throttle.Accept()
    }

    // Async or sync processing
    if b.asyncDelegate {
        go b.delegateBackend.ProcessEvents(events...)
    } else {
        b.delegateBackend.ProcessEvents(events...)
    }
}
```

### **4. Truncate Backend Wrapper**

**File**: `staging/src/k8s.io/apiserver/plugin/pkg/audit/truncate/truncate.go`

#### **Purpose**

Prevents oversized audit events from overwhelming storage:

```go
type Config struct {
    MaxEventSize int64  // Per-event limit (default: 100KB)
    MaxBatchSize int64  // Batch size limit (default: 10MB)
}

func (b *backend) ProcessEvents(ev ...*auditinternal.Event) bool {
    var truncated []*auditinternal.Event

    for _, e := range ev {
        size := estimateSize(e)

        if size > b.maxEventSize {
            // Truncate event
            eTruncated := e.DeepCopy()

            // Remove request/response bodies
            if eTruncated.Level >= auditinternal.LevelRequest {
                eTruncated.RequestObject = nil
            }
            if eTruncated.Level >= auditinternal.LevelRequestResponse {
                eTruncated.ResponseObject = nil
            }

            // Add truncation annotation
            if eTruncated.Annotations == nil {
                eTruncated.Annotations = make(map[string]string)
            }
            eTruncated.Annotations[auditinternal.TruncatedAnnotation] = "true"

            truncated = append(truncated, eTruncated)
            metrics.TruncatedCounter.Inc()
        } else {
            truncated = append(truncated, e)
        }
    }

    return b.delegateBackend.ProcessEvents(truncated...)
}
```

### **Backend Comparison**

| **Backend** | **Latency** | **Reliability** | **Use Case** |
|-------------|-------------|-----------------|--------------|
| **Log** | Low (synchronous) | High (local disk) | Simple deployments, file-based collection |
| **Webhook** | Medium-High (network) | Medium (depends on receiver) | Centralized collection, real-time processing |
| **Buffered** | Very Low (async) | Medium (can drop on overflow) | High-volume environments |
| **Truncate** | Low (pass-through) | High | Large request/response bodies |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔒 Compliance Patterns**

### **SOC 2 Compliance**

**Requirement**: Audit trail of all system changes

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # TSC CC6.3: System operations are logged
  - level: Request
    verbs: ["create", "update", "patch", "delete"]

  # TSC CC6.1: Logical access is logged
  - level: Metadata
    resources:
      - group: "rbac.authorization.k8s.io"

  # TSC CC6.2: Prior to issuing credentials, user identity is authenticated
  - level: Metadata
    resources:
      - group: ""
        resources: ["serviceaccounts/token"]
```

**Retention**: 1 year minimum

### **PCI DSS Compliance**

**Requirement 10**: Track and monitor all access to network resources and cardholder data

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # 10.2.1: All individual user accesses to cardholder data
  - level: Metadata
    namespaces: ["pci-scope"]  # Cardholder data environment
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
    annotations:
      pci-requirement: "10.2.1"

  # 10.2.2: All actions by privileged users
  - level: RequestResponse
    userGroups: ["system:masters", "pci-admins"]
    annotations:
      pci-requirement: "10.2.2"

  # 10.2.3: Access to all audit trails
  # (Implement via RBAC on audit logs themselves)

  # 10.2.4: Invalid logical access attempts
  # (Captured via responseStatus.code != 200)

  # 10.2.5: Changes to identification and authentication
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["serviceaccounts"]
      - group: "rbac.authorization.k8s.io"
    annotations:
      pci-requirement: "10.2.5"

  # 10.2.7: Creation and deletion of system-level objects
  - level: Request
    verbs: ["create", "delete"]
    resources:
      - group: ""
        resources: ["namespaces", "persistentvolumes"]
    annotations:
      pci-requirement: "10.2.7"
```

**Retention**: 1 year online, 3 years total

### **HIPAA Compliance**

**Requirement**: Access logs for PHI (Protected Health Information)

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # 45 CFR § 164.312(b): Audit controls
  # Log all access to PHI-labeled resources
  - level: Metadata
    namespaces: ["healthcare-prod"]
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
        # Convention: PHI resources prefixed with "phi-"
    annotations:
      hipaa-requirement: "164.312(b)"

  # Log privileged operations in PHI namespace
  - level: RequestResponse
    namespaces: ["healthcare-prod"]
    verbs: ["create", "update", "delete"]
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
    annotations:
      hipaa-requirement: "164.312(b)-privileged"
```

**Retention**: 6 years minimum

### **GDPR Compliance**

**Article 30**: Records of processing activities

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Track data subject access (users accessing personal data)
  - level: Metadata
    namespaces: ["gdpr-scope"]
    verbs: ["get", "list"]
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
    annotations:
      gdpr-article: "30"
      data-category: "personal-data"

  # Track data processing operations
  - level: Request
    namespaces: ["gdpr-scope"]
    verbs: ["create", "update", "patch", "delete"]
    annotations:
      gdpr-article: "30"
      processing-activity: "data-modification"

  # Track data controller changes (RBAC)
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
    annotations:
      gdpr-article: "30"
      purpose: "access-control-changes"
```

**Retention**: Duration of processing + statute of limitations

### **Compliance Audit Queries**

**SOC 2 - List all system changes by user**:
```bash
cat audit.log | jq -r 'select(.verb | IN("create", "update", "patch", "delete")) | [.stageTimestamp, .user.username, .verb, .objectRef.resource, .objectRef.name] | @csv'
```

**PCI DSS - Privileged user actions**:
```bash
cat audit.log | jq 'select(.user.groups[] | contains("system:masters"))'
```

**HIPAA - PHI access audit**:
```bash
cat audit.log | jq 'select(.objectRef.namespace == "healthcare-prod" and .objectRef.resource == "secrets")'
```

**GDPR - Personal data access by data subject**:
```bash
cat audit.log | jq 'select(.annotations["gdpr-article"] == "30" and .verb == "get")'
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ Performance Optimization**

### **Audit Volume Estimation**

**Formula**:
```
Events/Second = (API Requests/Second) × (Stages per Request)
Storage/Day = Events/Second × 86400 × Avg Event Size
```

**Example Calculation** (1000-node cluster):

```
Component               | Requests/sec | Events/sec | Size/Event | Storage/Day
------------------------|--------------|------------|------------|-------------
Kubelet (watch)         | 1000         | 1000       | 500 B      | 43 GB
Controller Manager      | 500          | 500        | 600 B      | 26 GB
Scheduler               | 300          | 300        | 550 B      | 14 GB
User Operations         | 100          | 100        | 1 KB       | 8 GB
System Components       | 300          | 300        | 400 B      | 10 GB
------------------------|--------------|------------|------------|-------------
TOTAL                   | 2200         | 2200       | -          | 101 GB/day
```

**With Optimizations**:

```yaml
# Omit RequestReceived stage: -33% volume
omitStages: [RequestReceived]

# Omit managed fields: -20% size per event
omitManagedFields: true

# Level: Metadata instead of Request: -60% size

Result: ~30 GB/day (70% reduction)
```

### **High-Performance Configuration**

```bash
# API Server Configuration for Large Clusters
kube-apiserver \
  # Audit policy
  --audit-policy-file=/etc/kubernetes/audit-policy-optimized.yaml \

  # Log backend with rotation
  --audit-log-path=/var/log/kubernetes/audit.log \
  --audit-log-maxsize=500 \
  --audit-log-maxbackup=3 \
  --audit-log-maxage=7 \
  --audit-log-compress=true \

  # Webhook backend (async)
  --audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml \
  --audit-webhook-mode=batch \

  # Buffering (critical for performance)
  --audit-webhook-batch-buffer-size=50000 \
  --audit-webhook-batch-max-size=1000 \
  --audit-webhook-batch-max-wait=10s \
  --audit-webhook-batch-throttle-qps=50 \
  --audit-webhook-batch-throttle-burst=100 \

  # Truncation
  --audit-webhook-truncate-enabled=true \
  --audit-webhook-truncate-max-event-size=102400 \
  --audit-webhook-truncate-max-batch-size=10485760
```

### **Monitoring Audit Performance**

```promql
# Audit events generated per second
rate(apiserver_audit_event_total[5m])

# Audit errors
rate(apiserver_audit_error_total[5m])

# Webhook latency
histogram_quantile(0.99,
  rate(apiserver_audit_requests_duration_seconds_bucket[5m]))

# Buffer saturation
apiserver_audit_dropped_events_total

# Event size distribution
histogram_quantile(0.95,
  rate(apiserver_audit_event_size_bytes_bucket[5m]))
```

**Grafana Alerts**:

```yaml
groups:
- name: audit-alerts
  rules:
  - alert: AuditDroppedEvents
    expr: rate(apiserver_audit_dropped_events_total[5m]) > 0
    annotations:
      summary: "Audit events being dropped - increase buffer size"

  - alert: AuditWebhookSlow
    expr: |
      histogram_quantile(0.99,
        rate(apiserver_audit_requests_duration_seconds_bucket[5m])) > 5
    annotations:
      summary: "Audit webhook latency high - check receiver"

  - alert: AuditErrorRateHigh
    expr: rate(apiserver_audit_error_total[5m]) > 1
    annotations:
      summary: "Audit errors occurring - check backend health"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔍 Audit Analysis Patterns**

### **Security Investigation**

**Scenario**: Investigate unauthorized secret access

```bash
# 1. Find all secret access events
cat audit.log | jq 'select(.objectRef.resource == "secrets")' > secret-access.json

# 2. Group by user
cat secret-access.json | jq -r '.user.username' | sort | uniq -c | sort -rn

# 3. Find unusual access patterns
cat secret-access.json | jq -r '
  select(.user.username == "suspicious-user") |
  [.stageTimestamp, .verb, .objectRef.name, .sourceIPs[]] |
  @csv'

# 4. Check for successful vs failed access
cat secret-access.json | jq '
  group_by(.responseStatus.code) |
  map({code: .[0].responseStatus.code, count: length})'

# Result:
# [{"code": 200, "count": 145}, {"code": 403, "count": 28}]
```

### **Anomaly Detection**

```bash
# Detect unusual exec/attach operations
cat audit.log | jq -r '
  select(.objectRef.resource == "pods/exec" or .objectRef.resource == "pods/attach") |
  [.stageTimestamp, .user.username, .objectRef.namespace, .objectRef.name, .sourceIPs[]] |
  @csv' | \
awk -F, '{count[$2]++} END {for (user in count) print user, count[user]}' | \
sort -k2 -rn

# Detect after-hours activity
cat audit.log | jq -r '
  select(.stageTimestamp | fromdateiso8601 | strftime("%H") | tonumber | . < 6 or . > 20) |
  [.stageTimestamp, .user.username, .verb, .objectRef.resource] |
  @csv'

# Detect privilege escalation attempts
cat audit.log | jq 'select(
  .objectRef.resource == "clusterrolebindings" and
  .verb == "create" and
  .requestObject.roleRef.name == "cluster-admin"
)'
```

### **Compliance Reporting**

```bash
# SOC 2 - System change report
cat audit.log | jq -r '
  select(.verb | IN("create", "update", "patch", "delete")) |
  select(.level != "None") |
  [.stageTimestamp, .user.username, .verb, .objectRef.resource,
   .objectRef.namespace, .objectRef.name, .responseStatus.code] |
  @csv' > soc2-changes-report.csv

# PCI DSS - Privileged access report
cat audit.log | jq -r '
  select(.user.groups[]? | contains("system:masters")) |
  [.stageTimestamp, .user.username, .verb, .objectRef.resource,
   .sourceIPs[], .userAgent] |
  @csv' > pci-privileged-access.csv

# HIPAA - PHI access audit
cat audit.log | jq -r '
  select(.objectRef.namespace == "healthcare-prod") |
  select(.objectRef.resource == "secrets") |
  [.stageTimestamp, .user.username, .verb, .objectRef.name,
   .responseStatus.code] |
  @csv' > hipaa-phi-access.csv
```

### **Forensic Timeline**

```bash
# Create timeline for specific incident
INCIDENT_START="2024-01-15T10:00:00Z"
INCIDENT_END="2024-01-15T11:00:00Z"

cat audit.log | jq -r --arg start "$INCIDENT_START" --arg end "$INCIDENT_END" '
  select(.stageTimestamp >= $start and .stageTimestamp <= $end) |
  select(.user.username == "compromised-user") |
  [.stageTimestamp, .verb, .objectRef.resource, .objectRef.name,
   .sourceIPs[], .responseStatus.code] |
  @csv' | sort
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚨 Production Troubleshooting**

### **Scenario 1: Audit Events Being Dropped**

**Symptoms**:
```promql
apiserver_audit_dropped_events_total > 0
```

**Investigation**:

```bash
# Check metrics
curl http://localhost:8080/metrics | grep audit_dropped

# Check API server logs
kubectl logs -n kube-system kube-apiserver-master-1 | grep "audit buffer full"

# Check current buffer usage (if exposed)
curl http://localhost:8080/metrics | grep apiserver_audit_buffer
```

**Root Causes**:

1. **Buffer too small**:
```bash
# Increase buffer size
--audit-webhook-batch-buffer-size=100000  # Up from 10000
```

2. **Webhook receiver too slow**:
```bash
# Check webhook latency
curl http://localhost:8080/metrics | grep apiserver_audit_requests_duration

# Increase batch size to reduce request frequency
--audit-webhook-batch-max-size=2000  # Up from 400
--audit-webhook-batch-max-wait=60s   # Up from 30s
```

3. **Rate limiting too aggressive**:
```bash
# Increase throttle limits
--audit-webhook-batch-throttle-qps=100    # Up from 10
--audit-webhook-batch-throttle-burst=200  # Up from 15
```

### **Scenario 2: Excessive Storage Growth**

**Symptoms**:
```bash
# Audit logs consuming 1TB/week
df -h /var/log/kubernetes/
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1       2.0T  1.5T  500G  75% /var
```

**Investigation**:

```bash
# Check audit log size growth
du -h /var/log/kubernetes/audit.log*
# 150G  audit.log
# 140G  audit.log.2024-01-15.gz
# 138G  audit.log.2024-01-14.gz

# Identify largest event types
cat audit.log | jq -r '.objectRef.resource' | sort | uniq -c | sort -rn | head -20

# Check average event size
cat audit.log | jq -r 'tostring | length' | \
  awk '{sum+=$1; count++} END {print sum/count " bytes"}'
```

**Optimizations**:

```yaml
# 1. Reduce audit level
- level: Metadata  # Instead of RequestResponse
  resources:
    - group: ""
      resources: ["pods", "services"]

# 2. Omit stages
omitStages:
  - RequestReceived
  - ResponseStarted

# 3. Omit managed fields
omitManagedFields: true

# 4. Exclude noisy resources
- level: None
  resources:
    - group: ""
      resources: ["events"]  # Very high volume
    - group: "coordination.k8s.io"
      resources: ["leases"]  # Leader election noise
```

### **Scenario 3: Webhook Backend Failures**

**Symptoms**:
```bash
# High error rate
rate(apiserver_audit_error_total[5m]) > 10

# API server logs
"Failed to send audit event" error="context deadline exceeded"
```

**Investigation**:

```bash
# Check webhook receiver health
kubectl get pods -n audit-system
kubectl logs -n audit-system audit-receiver-xxx

# Test webhook endpoint
curl -k -v https://audit-collector.example.com:8443/events

# Check network connectivity
kubectl run test --rm -it --image=curlimages/curl -- \
  curl -v https://audit-collector.example.com:8443/events
```

**Fixes**:

1. **Enable retry with backoff**:
```bash
--audit-webhook-initial-backoff=10s  # Retry after 10s
```

2. **Use async mode**:
```bash
--audit-webhook-mode=batch  # Non-blocking
```

3. **Add fallback to log backend**:
```go
// Configure multiple backends
backends := []audit.Backend{
    webhookBackend,   // Primary
    logBackend,       // Fallback
}
multiBackend := audit.Union(backends...)
```

### **Scenario 4: Performance Impact**

**Symptoms**:
```bash
# API server latency increased
histogram_quantile(0.99,
  rate(apiserver_request_duration_seconds_bucket[5m])) > 2

# Audit webhook blocking requests
kubectl get pods --v=8
# Shows slow responses
```

**Investigation**:

```bash
# Check audit mode
ps aux | grep kube-apiserver | grep audit-webhook-mode

# Check if blocking
# blocking-strict = API request fails if audit fails
# blocking = API request waits for audit
# batch = API request continues immediately
```

**Fix**:

```bash
# Switch to batch mode (async)
--audit-webhook-mode=batch

# Or reduce audit level
# Change from RequestResponse to Metadata
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💡 Best Practices**

### **Policy Design**

1. **Start restrictive, expand gradually**
   ```yaml
   # Phase 1: Minimal (Week 1)
   - level: Metadata

   # Phase 2: Add mutations (Week 2-3)
   - level: Request
     verbs: ["create", "update", "delete"]

   # Phase 3: Add sensitive resources (Week 4+)
   - level: RequestResponse
     resources:
       - group: ""
         resources: ["secrets"]
   ```

2. **Use omitStages aggressively**
   ```yaml
   # Reduce volume by 50-66%
   omitStages:
     - RequestReceived
     - ResponseStarted
   ```

3. **Omit managed fields**
   ```yaml
   # Reduce event size by ~20%
   omitManagedFields: true
   ```

4. **Order rules by specificity**
   ```yaml
   # Most specific first
   - level: None
     nonResourceURLs: ["/healthz"]

   - level: RequestResponse
     resources: ["secrets"]

   - level: Request
     verbs: ["create", "update", "delete"]

   - level: Metadata  # Default catch-all
   ```

### **Backend Selection**

| **Scenario** | **Recommended Backend** | **Configuration** |
|--------------|-------------------------|-------------------|
| **Small cluster (<100 nodes)** | Log + log aggregator | MaxSize: 100MB, Retention: 30 days |
| **Medium cluster (100-1000)** | Webhook + buffering | BufferSize: 50000, Async: true |
| **Large cluster (1000+)** | Webhook + buffering + truncation | MaxEventSize: 50KB, MaxBatchSize: 5MB |
| **Compliance-focused** | Dual (log + webhook) | Separate immutable storage |
| **High security** | Webhook to SIEM | TLS mutual auth, encryption |

### **Performance Tuning**

```bash
# Small cluster (< 100 nodes)
--audit-webhook-batch-buffer-size=10000
--audit-webhook-batch-max-size=400
--audit-webhook-batch-max-wait=30s

# Medium cluster (100-1000 nodes)
--audit-webhook-batch-buffer-size=50000
--audit-webhook-batch-max-size=1000
--audit-webhook-batch-max-wait=15s
--audit-webhook-batch-throttle-qps=50

# Large cluster (1000+ nodes)
--audit-webhook-batch-buffer-size=100000
--audit-webhook-batch-max-size=2000
--audit-webhook-batch-max-wait=10s
--audit-webhook-batch-throttle-qps=100
--audit-webhook-truncate-enabled=true
```

### **Security Hardening**

1. **Protect audit logs**:
```bash
# Restrict file permissions
chmod 600 /var/log/kubernetes/audit.log

# Use dedicated volume
mount -o noexec,nosuid /dev/sdb1 /var/log/kubernetes

# Enable SELinux/AppArmor
semanage fcontext -a -t var_log_t "/var/log/kubernetes(/.*)?"
```

2. **Webhook TLS**:
```yaml
# Mutual TLS authentication
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/audit-ca.crt
    server: https://audit-collector:8443
users:
- user:
    client-certificate: /etc/kubernetes/pki/audit-client.crt
    client-key: /etc/kubernetes/pki/audit-client.key
```

3. **RBAC for audit logs**:
```yaml
# Restrict access to audit logs
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: audit-log-reader
rules:
- nonResourceURLs: ["/logs/audit.log"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: security-team-audit-access
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: audit-log-reader
subjects:
- kind: Group
  name: security-team
```

### **Compliance Best Practices**

1. **Implement retention policies**:
```bash
# SOC 2: 1 year
--audit-log-maxage=365

# PCI DSS: 1 year online, 3 years archived
# Use external archival system after 365 days

# HIPAA: 6 years
# Implement external long-term storage
```

2. **Regular compliance audits**:
```bash
# Monthly audit coverage check
cat audit.log | jq -r '.objectRef.resource' | sort -u > current-coverage.txt
diff required-coverage.txt current-coverage.txt

# Quarterly audit policy review
# Validate all compliance requirements are met
```

3. **Automated compliance reports**:
```bash
#!/bin/bash
# daily-compliance-report.sh

REPORT_DATE=$(date +%Y-%m-%d)

# SOC 2 report
cat /var/log/kubernetes/audit.log | \
  jq -r 'select(.verb | IN("create", "update", "patch", "delete"))' > \
  "/reports/soc2-${REPORT_DATE}.json"

# PCI DSS report
cat /var/log/kubernetes/audit.log | \
  jq -r 'select(.annotations["pci-requirement"])' > \
  "/reports/pci-${REPORT_DATE}.json"

# Archive to immutable storage
aws s3 cp "/reports/" "s3://compliance-archive/${REPORT_DATE}/" --recursive
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚧 Known Limitations and Gaps**

### **🔴 Architectural Gaps**

#### **🔻 No Built-in Log Integrity**
- No cryptographic signatures on audit events
- No tamper-evident logging mechanism
- **Mitigation**: Forward to external immutable storage (WORM, blockchain-based systems)

#### **🔻 No Native Encryption at Rest**
- Audit logs written in plaintext
- Secrets visible in RequestResponse level
- **Mitigation**: Encrypt log volumes, use Level: Metadata for secrets

#### **🔻 Limited Retention Management**
- No built-in archival system
- File rotation only (lumberjack)
- **Mitigation**: External log management systems (Elasticsearch, Splunk, S3)

#### **🔻 Performance vs. Completeness Trade-off**
- Batch mode can drop events under extreme load
- Blocking mode impacts API server latency
- **Mitigation**: Careful capacity planning, monitoring buffer saturation

#### **🔻 No Sampling Support**
- All-or-nothing per policy rule
- Can't audit "10% of GET requests"
- **Mitigation**: Custom webhook receiver with sampling logic

### **🔴 Operational Challenges**

1. **Storage Growth**
   - RequestResponse level can generate 100GB+/day in large clusters
   - Compressed logs still require significant storage

2. **Analysis Complexity**
   - Massive JSON log files difficult to query
   - Requires log aggregation infrastructure

3. **Policy Tuning**
   - Finding balance between coverage and volume
   - Compliance vs. performance trade-offs

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 References**

### **Source Code**
- **Audit Event Types**: `staging/src/k8s.io/apiserver/pkg/apis/audit/v1/types.go`
- **Log Backend**: `staging/src/k8s.io/apiserver/plugin/pkg/audit/log/backend.go`
- **Webhook Backend**: `staging/src/k8s.io/apiserver/plugin/pkg/audit/webhook/webhook.go`
- **Buffered Backend**: `staging/src/k8s.io/apiserver/plugin/pkg/audit/buffered/buffered.go`
- **Truncate Backend**: `staging/src/k8s.io/apiserver/plugin/pkg/audit/truncate/truncate.go`
- **Policy Checker**: `staging/src/k8s.io/apiserver/pkg/audit/policy/checker.go`
- **Request Filter**: `staging/src/k8s.io/apiserver/pkg/endpoints/filters/audit.go`

### **Related Documentation**
- **API Server Architecture**: `docs/architecture/claude/apiserver/`
- **Authentication & Authorization**: `docs/architecture/claude/security/`
- **Metrics & Monitoring**: `docs/architecture/claude/observability/01-metrics-and-dashboards.md`
- **Logging Architecture**: `docs/architecture/claude/observability/02-logging-and-analysis.md`

### **External Resources**
- **Kubernetes Audit Documentation**: https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/
- **Audit Policy Examples**: https://github.com/kubernetes/kubernetes/tree/master/cluster/gce/gci/configure-helper.sh
- **SOC 2 Framework**: https://www.aicpa.org/interestareas/frc/assuranceadvisoryservices/sorhome.html
- **PCI DSS Requirements**: https://www.pcisecuritystandards.org/
- **HIPAA Security Rule**: https://www.hhs.gov/hipaa/for-professionals/security/

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: Complete
**Last Updated**: 2024-11-17
**Target Audience**: Security engineers, compliance teams, platform engineers
**Scope**: Production audit logging, compliance patterns, forensic analysis
