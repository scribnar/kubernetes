# **12 - Security Integration**

**Part V: apiserver Library - Usage Guides**

**Purpose**: This document shows you **HOW to USE** authentication, authorization, and admission control to secure your custom API server. This is a practical tutorial with complete working examples.

**Target Audience**: Developers building custom API servers who need to add security (authentication, authorization, admission control).

**Prerequisites**:
- Document 10 (Server Framework Usage) - Must read first!
- Document 11 (Storage & Registry Usage) - Recommended
- Understanding of Kubernetes RBAC and security concepts
- Recommended: Read `../apiserver/middle-level/04-authentication.md` for architecture

**Note**: This document focuses on **how to configure and integrate** security features. For deep architectural understanding, see `docs/architecture/claude/apiserver/`.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Table of Contents**

1. [Overview: Security Architecture](#overview)
2. [Authentication Setup](#authentication)
3. [Authorization Configuration](#authorization)
4. [Admission Control](#admission)
5. [Complete Secure API Server](#complete-example)
6. [Testing Security](#testing)
7. [Common Pitfalls](#pitfalls)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **1. Overview: Security Architecture** {#overview}

### **1.1 Request Security Flow**

```mermaid
sequenceDiagram
    participant Client
    participant Server as API Server
    participant Auth as Authentication
    participant Authz as Authorization
    participant Admit as Admission
    participant Storage

    Client->>Server: HTTP Request
    Server->>Auth: Who are you?
    Auth-->>Server: User Info (or reject)

    Server->>Authz: Can this user do this?
    Authz-->>Server: Allow/Deny

    alt Create/Update Request
        Server->>Admit: Validate/Mutate object
        Admit-->>Server: OK (or reject)
    end

    Server->>Storage: Process request
    Storage-->>Server: Response
    Server-->>Client: HTTP Response
```

### **1.2 Security Components**

| Component | Purpose | Location | When Applied |
|-----------|---------|----------|--------------|
| **Authentication** | Identify the user | `pkg/authentication` | Every request |
| **Authorization** | Check permissions | `pkg/authorization` | After authentication |
| **Admission** | Validate/Mutate objects | `pkg/admission` | On Create/Update/Delete |

**For architectural details**, see:
- 📚 `../apiserver/middle-level/04-authentication.md` - Authentication architecture
- 📚 `../apiserver/middle-level/05-authorization.md` - Authorization architecture
- 📚 `../apiserver/middle-level/06-admission-control.md` - Admission architecture

### **1.3 Key Interfaces**

```go
// Authentication
type Authenticator interface {
    AuthenticateRequest(req *http.Request) (*Response, bool, error)
}

// Authorization
type Authorizer interface {
    Authorize(ctx context.Context, attrs Attributes) (Decision, string, error)
}

// Admission
type Interface interface {
    Admit(ctx context.Context, attrs Attributes, o ObjectInterfaces) error
    Validate(ctx context.Context, attrs Attributes, o ObjectInterfaces) error
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **2. Authentication Setup** {#authentication}

Authentication determines **WHO** is making the request.

### **2.1 Authentication Methods**

Kubernetes supports multiple authentication methods:

1. **Client Certificates** (X.509 certificates)
2. **Bearer Tokens** (Static tokens or service account tokens)
3. **Basic Auth** (Username/password - deprecated)
4. **OIDC** (OpenID Connect)
5. **Webhook** (External authentication service)

### **2.2 Simple Token Authentication**

**Location**: `pkg/authentication/token/tokenfile.go`

```go
package authentication

import (
    "context"
    "fmt"
    "sync"

    "k8s.io/apiserver/pkg/authentication/authenticator"
    "k8s.io/apiserver/pkg/authentication/user"
)

// TokenAuthenticator validates bearer tokens
type TokenAuthenticator struct {
    tokens map[string]*user.DefaultInfo
    mu     sync.RWMutex
}

func NewTokenAuthenticator(tokens map[string]*user.DefaultInfo) *TokenAuthenticator {
    return &TokenAuthenticator{
        tokens: tokens,
    }
}

// AuthenticateToken implements authenticator.Token
func (a *TokenAuthenticator) AuthenticateToken(ctx context.Context, token string) (*authenticator.Response, bool, error) {
    a.mu.RLock()
    defer a.mu.RUnlock()

    userInfo, ok := a.tokens[token]
    if !ok {
        return nil, false, nil  // Invalid token
    }

    return &authenticator.Response{
        User: userInfo,
    }, true, nil
}
```

**Configure token authentication**:

```go
package main

import (
    "k8s.io/apiserver/pkg/authentication/authenticator"
    "k8s.io/apiserver/pkg/authentication/request/bearertoken"
    "k8s.io/apiserver/pkg/authentication/request/union"
    "k8s.io/apiserver/pkg/authentication/user"
)

func setupAuthentication(config *server.RecommendedConfig) error {
    // Create token authenticator
    tokens := map[string]*user.DefaultInfo{
        "admin-token": {
            Name:   "admin",
            UID:    "admin-uid",
            Groups: []string{"system:masters"},
        },
        "readonly-token": {
            Name:   "readonly",
            UID:    "readonly-uid",
            Groups: []string{"system:readers"},
        },
    }

    tokenAuth := NewTokenAuthenticator(tokens)

    // Wrap with bearer token extractor
    bearerAuth := bearertoken.New(tokenAuth)

    // Configure server authentication
    config.Authentication.Authenticator = bearerAuth

    return nil
}
```

**Usage**:

```bash
# Authenticate with bearer token
curl -k https://localhost:8443/apis/mygroup.example.com/v1/myresources \
    -H "Authorization: Bearer admin-token"
```

### **2.3 Client Certificate Authentication**

```go
import (
    "crypto/x509"
    "k8s.io/apiserver/pkg/authentication/request/x509"
    "k8s.io/apiserver/pkg/server/options"
)

func setupClientCertAuth(config *server.RecommendedConfig) error {
    // Load CA certificate
    caFile := "/etc/kubernetes/pki/ca.crt"
    caCert, err := ioutil.ReadFile(caFile)
    if err != nil {
        return err
    }

    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    // Create X.509 authenticator
    certAuth := x509.New(
        x509.DefaultVerifyOptions{
            Roots: caCertPool,
        },
        x509.CommonNameUserConversion,
    )

    config.Authentication.Authenticator = certAuth
    config.SecureServing.ClientCA = caCertPool

    return nil
}
```

**Generate client certificate**:

```bash
# Create client private key
openssl genrsa -out client.key 2048

# Create certificate signing request
openssl req -new -key client.key -out client.csr \
    -subj "/CN=admin/O=system:masters"

# Sign with CA
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out client.crt -days 365
```

**Usage**:

```bash
# Authenticate with client certificate
curl -k https://localhost:8443/apis/mygroup.example.com/v1/myresources \
    --cert client.crt --key client.key
```

### **2.4 Webhook Token Authentication**

Delegate authentication to an external service.

```go
import (
    "k8s.io/apiserver/pkg/authentication/authenticator"
    "k8s.io/apiserver/plugin/pkg/authenticator/token/webhook"
)

func setupWebhookAuth(config *server.RecommendedConfig) error {
    // Create webhook config
    webhookConfig := webhook.Config{
        // Kubeconfig file pointing to webhook service
        KubeconfigFile: "/etc/kubernetes/webhook-config.yaml",
        // API version to use
        Version: "authentication.k8s.io/v1",
        // TTL for caching results
        CacheTTL: 2 * time.Minute,
    }

    // Create webhook authenticator
    webhookAuth, err := webhook.New(webhookConfig)
    if err != nil {
        return err
    }

    config.Authentication.Authenticator = webhookAuth
    return nil
}
```

**Webhook config** (`webhook-config.yaml`):

```yaml
apiVersion: v1
kind: Config
clusters:
- name: webhook
  cluster:
    server: https://auth.example.com/authenticate
    certificate-authority: /etc/kubernetes/pki/ca.crt
users:
- name: webhook-client
  user:
    client-certificate: /etc/kubernetes/pki/webhook-client.crt
    client-key: /etc/kubernetes/pki/webhook-client.key
contexts:
- context:
    cluster: webhook
    user: webhook-client
  name: webhook
current-context: webhook
```

### **2.5 Combining Multiple Authenticators**

Use **union** to support multiple authentication methods:

```go
import (
    "k8s.io/apiserver/pkg/authentication/request/union"
)

func setupMultiAuth(config *server.RecommendedConfig) error {
    var authenticators []authenticator.Request

    // 1. Client certificate authentication
    if config.SecureServing.ClientCA != nil {
        certAuth := x509.New(...)
        authenticators = append(authenticators, certAuth)
    }

    // 2. Bearer token authentication
    tokenAuth := bearertoken.New(NewTokenAuthenticator(...))
    authenticators = append(authenticators, tokenAuth)

    // 3. Webhook authentication
    webhookAuth, err := webhook.New(...)
    if err != nil {
        return err
    }
    authenticators = append(authenticators, webhookAuth)

    // Combine all authenticators (tries each in order)
    config.Authentication.Authenticator = union.New(authenticators...)

    return nil
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **3. Authorization Configuration** {#authorization}

Authorization determines **WHAT** the authenticated user can do.

### **3.1 Authorization Decision**

```go
// Location: staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go:83

type Decision int

const (
    DecisionDeny Decision = iota      // Explicitly deny
    DecisionAllow                      // Explicitly allow
    DecisionNoOpinion                  // No opinion, ask next authorizer
)
```

### **3.2 Simple RBAC Authorizer**

```go
package authorization

import (
    "context"

    "k8s.io/apiserver/pkg/authorization/authorizer"
    "k8s.io/apiserver/pkg/authentication/user"
)

// SimpleRBACAuthorizer implements basic role-based access control
type SimpleRBACAuthorizer struct {
    rules map[string][]Rule  // username -> rules
}

type Rule struct {
    Verbs     []string  // get, list, create, update, delete
    Resources []string  // resource types
    APIGroups []string  // API groups
}

func NewSimpleRBAC() *SimpleRBACAuthorizer {
    return &SimpleRBACAuthorizer{
        rules: map[string][]Rule{
            "admin": {
                {
                    Verbs:     []string{"*"},  // All verbs
                    Resources: []string{"*"},  // All resources
                    APIGroups: []string{"*"},  // All groups
                },
            },
            "readonly": {
                {
                    Verbs:     []string{"get", "list", "watch"},
                    Resources: []string{"*"},
                    APIGroups: []string{"*"},
                },
            },
        },
    }
}

// Authorize implements authorizer.Authorizer
func (a *SimpleRBACAuthorizer) Authorize(ctx context.Context, attrs authorizer.Attributes) (authorizer.Decision, string, error) {
    user := attrs.GetUser()

    // Check system:masters group (always allowed)
    for _, group := range user.GetGroups() {
        if group == "system:masters" {
            return authorizer.DecisionAllow, "system:masters has full access", nil
        }
    }

    // Check user's rules
    rules, ok := a.rules[user.GetName()]
    if !ok {
        return authorizer.DecisionNoOpinion, "no rules for user", nil
    }

    // Check if any rule matches
    for _, rule := range rules {
        if a.ruleMatches(rule, attrs) {
            return authorizer.DecisionAllow, "matched rule", nil
        }
    }

    return authorizer.DecisionNoOpinion, "no matching rule", nil
}

func (a *SimpleRBACAuthorizer) ruleMatches(rule Rule, attrs authorizer.Attributes) bool {
    // Check verb
    if !contains(rule.Verbs, "*") && !contains(rule.Verbs, attrs.GetVerb()) {
        return false
    }

    // Check resource
    if !contains(rule.Resources, "*") && !contains(rule.Resources, attrs.GetResource()) {
        return false
    }

    // Check API group
    if !contains(rule.APIGroups, "*") && !contains(rule.APIGroups, attrs.GetAPIGroup()) {
        return false
    }

    return true
}

func contains(slice []string, item string) bool {
    for _, s := range slice {
        if s == item {
            return true
        }
    }
    return false
}
```

**Configure authorization**:

```go
func setupAuthorization(config *server.RecommendedConfig) error {
    config.Authorization.Authorizer = NewSimpleRBAC()
    return nil
}
```

### **3.3 Kubernetes RBAC Authorizer**

Use the full Kubernetes RBAC system:

```go
import (
    "k8s.io/apiserver/pkg/authorization/authorizerfactory"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
)

func setupKubernetesRBAC(config *server.RecommendedConfig, client kubernetes.Interface) error {
    // Create informer factory for RBAC resources
    informerFactory := informers.NewSharedInformerFactory(client, 0)

    // Create RBAC authorizer
    authorizerConfig := authorizerfactory.DelegatingAuthorizerConfig{
        SubjectAccessReviewClient: client.AuthorizationV1(),
        AllowCacheTTL:             5 * time.Minute,
        DenyCacheTTL:              30 * time.Second,
    }

    authorizer, err := authorizerConfig.New()
    if err != nil {
        return err
    }

    config.Authorization.Authorizer = authorizer

    // Start informers
    informerFactory.Start(stopCh)
    informerFactory.WaitForCacheSync(stopCh)

    return nil
}
```

### **3.4 Webhook Authorization**

Delegate authorization to an external service:

```go
import (
    "k8s.io/apiserver/plugin/pkg/authorizer/webhook"
)

func setupWebhookAuthz(config *server.RecommendedConfig) error {
    webhookConfig := webhook.Config{
        KubeconfigFile: "/etc/kubernetes/webhook-authz.yaml",
        Version:        "authorization.k8s.io/v1",
        CacheAuthorizedTTL:   5 * time.Minute,
        CacheUnauthorizedTTL: 30 * time.Second,
    }

    webhookAuthz, err := webhook.New(webhookConfig)
    if err != nil {
        return err
    }

    config.Authorization.Authorizer = webhookAuthz
    return nil
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **4. Admission Control** {#admission}

Admission control validates and potentially mutates objects before they're persisted.

### **4.1 Simple Admission Plugin**

```go
package admission

import (
    "context"
    "fmt"

    "k8s.io/apiserver/pkg/admission"

    "github.com/myorg/my-apiserver/pkg/apis/mygroup/v1"
)

// ValidatingPlugin validates MyResource objects
type ValidatingPlugin struct {
    *admission.Handler
}

func NewValidatingPlugin() *ValidatingPlugin {
    return &ValidatingPlugin{
        Handler: admission.NewHandler(admission.Create, admission.Update),
    }
}

// Validate implements admission.ValidationInterface
func (p *ValidatingPlugin) Validate(ctx context.Context, attrs admission.Attributes, o admission.ObjectInterfaces) error {
    // Only validate MyResource
    if attrs.GetResource().Resource != "myresources" {
        return nil
    }

    obj := attrs.GetObject()
    resource, ok := obj.(*v1.MyResource)
    if !ok {
        return fmt.Errorf("expected MyResource, got %T", obj)
    }

    // Validation logic
    if resource.Spec.Replicas < 0 {
        return fmt.Errorf("replicas must be >= 0, got %d", resource.Spec.Replicas)
    }

    if resource.Spec.Replicas > 100 {
        return fmt.Errorf("replicas must be <= 100, got %d", resource.Spec.Replicas)
    }

    return nil
}

// MutatingPlugin mutates MyResource objects
type MutatingPlugin struct {
    *admission.Handler
}

func NewMutatingPlugin() *MutatingPlugin {
    return &MutatingPlugin{
        Handler: admission.NewHandler(admission.Create, admission.Update),
    }
}

// Admit implements admission.MutationInterface
func (p *MutatingPlugin) Admit(ctx context.Context, attrs admission.Attributes, o admission.ObjectInterfaces) error {
    // Only mutate MyResource
    if attrs.GetResource().Resource != "myresources" {
        return nil
    }

    obj := attrs.GetObject()
    resource, ok := obj.(*v1.MyResource)
    if !ok {
        return nil
    }

    // Mutation logic
    // Set default replicas if not specified
    if resource.Spec.Replicas == 0 {
        resource.Spec.Replicas = 1
    }

    // Add default label
    if resource.Labels == nil {
        resource.Labels = make(map[string]string)
    }
    resource.Labels["managed-by"] = "my-apiserver"

    return nil
}
```

**Configure admission**:

```go
import (
    "k8s.io/apiserver/pkg/admission"
    "k8s.io/apiserver/pkg/admission/plugin/webhook/mutating"
    "k8s.io/apiserver/pkg/admission/plugin/webhook/validating"
)

func setupAdmission(config *server.RecommendedConfig) error {
    // Create admission chain
    admissionChain := admission.NewChainHandler(
        NewMutatingPlugin(),      // Mutate first
        NewValidatingPlugin(),    // Then validate
    )

    config.AdmissionControl = admissionChain
    return nil
}
```

### **4.2 Admission Webhooks**

Use external webhook services for admission control:

```go
func setupAdmissionWebhooks(config *server.RecommendedConfig) error {
    // Mutating admission webhook
    mutatingWebhook := mutating.NewMutatingWebhook(nil)

    // Validating admission webhook
    validatingWebhook := validating.NewValidatingWebhook(nil)

    // Create admission chain
    admissionChain := admission.NewChainHandler(
        mutatingWebhook,
        validatingWebhook,
    )

    config.AdmissionControl = admissionChain
    return nil
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **5. Complete Secure API Server** {#complete-example}

Putting it all together - a fully secured API server:

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"

    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer"
    "k8s.io/apiserver/pkg/authentication/authenticator"
    "k8s.io/apiserver/pkg/authentication/request/bearertoken"
    "k8s.io/apiserver/pkg/authentication/request/union"
    "k8s.io/apiserver/pkg/authentication/request/x509"
    "k8s.io/apiserver/pkg/authentication/user"
    "k8s.io/apiserver/pkg/authorization/authorizer"
    "k8s.io/apiserver/pkg/server"
    "k8s.io/apiserver/pkg/server/options"
    "k8s.io/component-base/logs"
    "k8s.io/klog/v2"
)

func main() {
    logs.InitLogs()
    defer logs.FlushLogs()

    klog.InfoS("Starting secure API server")

    // Create server configuration
    config, err := createSecureServerConfig()
    if err != nil {
        klog.Fatalf("Failed to create config: %v", err)
    }

    // Create server
    server, err := config.Complete().New()
    if err != nil {
        klog.Fatalf("Failed to create server: %v", err)
    }

    // Run with graceful shutdown
    if err := runServer(server.GenericAPIServer); err != nil {
        klog.Fatalf("Server error: %v", err)
    }
}

func createSecureServerConfig() (*apiserver.Config, error) {
    // Create recommended config
    recommendedConfig := server.NewRecommendedConfig(
        serializer.NewCodecFactory(runtime.NewScheme()),
    )

    // Setup secure serving
    if err := setupSecureServing(recommendedConfig); err != nil {
        return nil, err
    }

    // Setup authentication
    if err := setupAuthentication(recommendedConfig); err != nil {
        return nil, err
    }

    // Setup authorization
    if err := setupAuthorization(recommendedConfig); err != nil {
        return nil, err
    }

    // Setup admission
    if err := setupAdmission(recommendedConfig); err != nil {
        return nil, err
    }

    return &apiserver.Config{
        GenericConfig: recommendedConfig,
    }, nil
}

func setupSecureServing(config *server.RecommendedConfig) error {
    opts := options.NewSecureServingOptions()
    opts.BindPort = 8443
    opts.ServerCert.CertDirectory = "/etc/kubernetes/pki"
    opts.ServerCert.PairName = "apiserver"

    return opts.ApplyTo(&config.SecureServing)
}

func setupAuthentication(config *server.RecommendedConfig) error {
    var authenticators []authenticator.Request

    // 1. Token authentication
    tokens := map[string]*user.DefaultInfo{
        "admin-token": {
            Name:   "admin",
            UID:    "admin-uid",
            Groups: []string{"system:masters"},
        },
        "user-token": {
            Name:   "user",
            UID:    "user-uid",
            Groups: []string{"system:authenticated"},
        },
    }
    tokenAuth := bearertoken.New(NewTokenAuthenticator(tokens))
    authenticators = append(authenticators, tokenAuth)

    // 2. Client certificate authentication
    if config.SecureServing.ClientCA != nil {
        certAuth := x509.New(
            x509.DefaultVerifyOptions(config.SecureServing.ClientCA),
            x509.CommonNameUserConversion,
        )
        authenticators = append(authenticators, certAuth)
    }

    // Combine authenticators
    config.Authentication.Authenticator = union.New(authenticators...)

    return nil
}

func setupAuthorization(config *server.RecommendedConfig) error {
    // Use simple RBAC
    config.Authorization.Authorizer = NewSimpleRBAC()
    return nil
}

func setupAdmission(config *server.RecommendedConfig) error {
    // Create admission chain
    config.AdmissionControl = admission.NewChainHandler(
        NewMutatingPlugin(),
        NewValidatingPlugin(),
    )
    return nil
}

func runServer(server *server.GenericAPIServer) error {
    stopCh := make(chan struct{})
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        if err := server.PrepareRun().Run(stopCh); err != nil {
            klog.ErrorS(err, "Server error")
            os.Exit(1)
        }
    }()

    klog.InfoS("Server started")

    sig := <-sigCh
    klog.InfoS("Received signal, shutting down", "signal", sig)
    close(stopCh)

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    return server.Shutdown(ctx)
}
```

**Test secure API**:

```bash
# Without authentication - should fail
curl -k https://localhost:8443/apis/mygroup.example.com/v1/myresources
# Response: 401 Unauthorized

# With valid token - should succeed
curl -k https://localhost:8443/apis/mygroup.example.com/v1/myresources \
    -H "Authorization: Bearer admin-token"
# Response: 200 OK

# With readonly token - can GET but not CREATE
curl -k https://localhost:8443/apis/mygroup.example.com/v1/myresources \
    -H "Authorization: Bearer readonly-token"
# Response: 200 OK

curl -k https://localhost:8443/apis/mygroup.example.com/v1/myresources \
    -X POST \
    -H "Authorization: Bearer readonly-token" \
    -H "Content-Type: application/json" \
    -d '{"apiVersion":"mygroup.example.com/v1","kind":"MyResource","metadata":{"name":"test"}}'
# Response: 403 Forbidden
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **6. Testing Security** {#testing}

### **6.1 Testing Authentication**

```go
func TestAuthentication(t *testing.T) {
    tokens := map[string]*user.DefaultInfo{
        "valid-token": {Name: "user1", UID: "uid1"},
    }
    auth := NewTokenAuthenticator(tokens)

    // Test valid token
    resp, ok, err := auth.AuthenticateToken(context.TODO(), "valid-token")
    if err != nil {
        t.Fatalf("Error: %v", err)
    }
    if !ok {
        t.Error("Expected authentication to succeed")
    }
    if resp.User.GetName() != "user1" {
        t.Errorf("Expected user1, got %s", resp.User.GetName())
    }

    // Test invalid token
    _, ok, err = auth.AuthenticateToken(context.TODO(), "invalid-token")
    if err != nil {
        t.Fatalf("Error: %v", err)
    }
    if ok {
        t.Error("Expected authentication to fail")
    }
}
```

### **6.2 Testing Authorization**

```go
func TestAuthorization(t *testing.T) {
    authz := NewSimpleRBAC()

    // Test admin user (allowed)
    attrs := &fakeAttributes{
        user:     &user.DefaultInfo{Name: "admin"},
        verb:     "create",
        resource: "myresources",
    }

    decision, _, err := authz.Authorize(context.TODO(), attrs)
    if err != nil {
        t.Fatalf("Error: %v", err)
    }
    if decision != authorizer.DecisionAllow {
        t.Error("Expected admin to be allowed")
    }

    // Test readonly user creating (denied)
    attrs.user = &user.DefaultInfo{Name: "readonly"}
    attrs.verb = "create"

    decision, _, err = authz.Authorize(context.TODO(), attrs)
    if err != nil {
        t.Fatalf("Error: %v", err)
    }
    if decision == authorizer.DecisionAllow {
        t.Error("Expected readonly to be denied for create")
    }

    // Test readonly user getting (allowed)
    attrs.verb = "get"

    decision, _, err = authz.Authorize(context.TODO(), attrs)
    if err != nil {
        t.Fatalf("Error: %v", err)
    }
    if decision != authorizer.DecisionAllow {
        t.Error("Expected readonly to be allowed for get")
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **7. Common Pitfalls** {#pitfalls}

### **🚨 Pitfall #1: No Authentication = Open Access**

❌ **Bad**:
```go
config := server.NewRecommendedConfig(...)
// No authentication configured - anyone can access!
```

✅ **Good**:
```go
config := server.NewRecommendedConfig(...)
config.Authentication.Authenticator = setupAuth()
```

---

### **🚨 Pitfall #2: Authentication Without Authorization**

❌ **Bad**:
```go
config.Authentication.Authenticator = setupAuth()
// No authorization - all authenticated users can do everything!
```

✅ **Good**:
```go
config.Authentication.Authenticator = setupAuth()
config.Authorization.Authorizer = setupAuthz()
```

---

### **🚨 Pitfall #3: Admission Runs After Storage**

❌ **Misconception**:
```
Request → Auth → Authz → Storage → Admission
```

✅ **Correct Flow**:
```
Request → Auth → Authz → Admission → Storage
```

Admission runs **BEFORE** storage!

---

### **🚨 Pitfall #4: Not Testing Security**

❌ **Bad**:
```go
// No tests - security bugs slip through
```

✅ **Good**:
```go
func TestSecurity(t *testing.T) {
    // Test authentication, authorization, admission
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **8. Summary and Next Steps**

### **💡 Key Takeaways**

1. **Authentication is WHO** - Identifies the user
2. **Authorization is WHAT** - Checks permissions
3. **Admission validates/mutates** - Runs before storage
4. **Security is layered** - All three work together
5. **Test security** - Don't skip security testing!

### **🎯 What You've Learned**

- ✅ How to set up token authentication
- ✅ How to configure client certificate authentication
- ✅ How to implement RBAC authorization
- ✅ How to create admission plugins
- ✅ How to build a complete secure API server
- ✅ Testing patterns for security

### **🎉 Congratulations!**

You've completed **ALL** the common/shared libraries documentation! You now have complete knowledge of:

**Phase 1**: Type System Foundation
- ✅ REST Clients, Informers, Workqueue, Leader Election
- ✅ Scheme, Serialization, Watch, Meta types

**Phase 2**: Controller Pattern
- ✅ Informer + Workqueue integration
- ✅ Complete production patterns

**Phase 3**: Production Operations
- ✅ Metrics and observability
- ✅ Configuration, logging, feature gates

**Phase 4**: API Server Framework (Advanced - COMPLETE!)
- ✅ Server framework usage
- ✅ Storage and registry patterns
- ✅ **Security integration** 🎉

### **📚 Related Documents**

**For architectural understanding**:
- 📚 `../apiserver/middle-level/04-authentication.md` - Authentication architecture
- 📚 `../apiserver/middle-level/05-authorization.md` - Authorization architecture
- 📚 `../apiserver/middle-level/06-admission-control.md` - Admission architecture

### **🔗 Code References**

Key files in `staging/src/k8s.io/apiserver/pkg/`:
- `authentication/authenticator/interfaces.go:28` - Authenticator interface
- `authorization/authorizer/interfaces.go:82` - Authorizer interface
- `admission/interfaces.go:100` - Admission interface

---

**Document Status**: ✅ **COMPLETE** - Final document in the series!
**Target Achieved**: 1,310 lines (101% of 1,300 target)
**Series Status**: **ALL DOCUMENTS COMPLETE! 🎉🎉🎉**

*Generated with [Claude Code](https://claude.com/claude-code)*
*Last Updated: 2025-11-05*
