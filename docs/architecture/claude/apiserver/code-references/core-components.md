# Core Components Code Reference

> **Complete code reference for kube-apiserver core components with file locations and line numbers**

---

## Entry Points

### main() - Application Entry
**File**: `cmd/kube-apiserver/apiserver.go`
```go
// Line 37
func main() {
    command := app.NewAPIServerCommand()
    code := cli.Run(command)
    os.Exit(code)
}
```

### NewAPIServerCommand() - Command Setup
**File**: `cmd/kube-apiserver/app/server.go:70-145`
```go
func NewAPIServerCommand() *cobra.Command {
    s := options.NewServerRunOptions()
    cmd := &cobra.Command{
        Use: "kube-apiserver",
        RunE: func(cmd *cobra.Command, args []string) error {
            completedOptions, err := s.Complete(ctx)
            // Validate and Run
            return Run(ctx, completedOptions)
        },
    }
    return cmd
}
```

### Run() - Main Execution
**File**: `cmd/kube-apiserver/app/server.go:148-173`
```go
func Run(ctx context.Context, opts options.CompletedOptions) error {
    config, err := NewConfig(opts)
    completed, err := config.Complete()
    server, err := CreateServerChain(completed)
    prepared, err := server.PrepareRun()
    return prepared.Run(ctx)
}
```

---

## Server Chain Creation

### CreateServerChain() - Build Delegation Chain
**File**: `cmd/kube-apiserver/app/server.go:176-197`
```go
func CreateServerChain(config CompletedConfig) (*aggregatorapiserver.APIAggregator, error) {
    // 1. NotFound handler
    notFoundHandler := notfoundhandler.New(...)

    // 2. API Extensions Server (CRDs)
    apiExtensionsServer, err := config.ApiExtensions.New(
        genericapiserver.NewEmptyDelegateWithCustomHandler(notFoundHandler))

    // 3. Kube API Server (built-in APIs)
    kubeAPIServer, err := config.KubeAPIs.New(apiExtensionsServer.GenericAPIServer)

    // 4. Aggregator Server (top of chain)
    aggregatorServer, err := controlplaneapiserver.CreateAggregatorServer(
        config.Aggregator,
        kubeAPIServer.ControlPlane.GenericAPIServer,
        ...)

    return aggregatorServer, nil
}
```

---

## Configuration

### BuildGenericConfig() - Generic Configuration
**File**: `pkg/controlplane/apiserver/config.go:113-243`
```go
func BuildGenericConfig(opts options.CompletedOptions, ...) (
    genericConfig *genericapiserver.Config,
    versionedInformers clientgoinformers.SharedInformerFactory,
    storageFactory *serverstorage.DefaultStorageFactory,
    err error,
) {
    genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)

    // Apply options
    opts.GenericServerRunOptions.ApplyTo(genericConfig)
    opts.SecureServing.ApplyTo(&genericConfig.SecureServing, ...)
    opts.Authentication.ApplyTo(ctx, &genericConfig.Authentication, ...)

    // Build authorization
    genericConfig.Authorization.Authorizer, _, _, err = BuildAuthorizer(ctx, opts, ...)

    // Setup admission
    opts.Admission.ApplyTo(genericConfig, ...)

    return
}
```

### Options Completion
**File**: `cmd/kube-apiserver/app/options/completion.go:47-92`
```go
func (s *ServerRunOptions) Complete(ctx context.Context) (CompletedOptions, error) {
    // Parse service IP ranges
    apiServerServiceIP, primaryRange, secondaryRange, err :=
        getServiceIPAndRanges(s.ServiceClusterIPRanges)

    // Complete controlplane options
    controlplane, err := s.Options.Complete(ctx, ...)

    // Set watch cache sizes
    if completed.Etcd != nil && completed.Etcd.EnableWatchCache {
        sizes := kubeapiserver.DefaultWatchCacheSizes()
        // Merge with user-specified sizes
    }

    return CompletedOptions{...}, nil
}
```

---

## Generic API Server

### GenericAPIServer Structure
**File**: `staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go`
```go
// Line ~100
type GenericAPIServer struct {
    discoveryAddresses discovery.Addresses
    LoopbackClientConfig *restclient.Config
    Handler *APIServerHandler
    delegationTarget DelegationTarget
    admissionControl admission.Interface
    AuditBackend audit.Backend
    Authorizer authorizer.Authorizer
    Serializer runtime.NegotiatedSerializer
    StorageFactory serverstorage.StorageFactory
    postStartHooks map[string]postStartHookEntry
    preShutdownHooks map[string]preShutdownHookEntry
    // ... 50+ more fields
}

// Line ~400
func (s *GenericAPIServer) PrepareRun() preparedGenericAPIServer {
    // Install OpenAPI
    // Install health checks
    // Install metrics
    // Install profiling
    return preparedGenericAPIServer{s}
}

// Line ~500
func (s preparedGenericAPIServer) Run(ctx context.Context) error {
    s.RunPostStartHooks(ctx.Done())
    s.NonBlockingRun(ctx.Done(), ...)
    <-ctx.Done()
    s.RunPreShutdownHooks()
    return nil
}
```

### APIServerHandler
**File**: `staging/src/k8s.io/apiserver/pkg/server/handler.go`
```go
type APIServerHandler struct {
    FullHandlerChain http.Handler
    GoRestfulContainer *restful.Container
    NonGoRestfulMux *mux.PathRecorderMux
    Director http.Handler
}

func (d director) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    path := req.URL.Path
    if strings.HasPrefix(path, "/apis/") || strings.HasPrefix(path, "/api/") {
        d.goRestfulContainer.ServeHTTP(w, req)
        return
    }
    d.nonGoRestfulMux.ServeHTTP(w, req)
}
```

---

## Handler Chain

### DefaultBuildHandlerChain() - Build Filter Pipeline
**File**: `staging/src/k8s.io/apiserver/pkg/server/config.go:1014-1091`
```go
func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
    handler := apiHandler

    // Layer 18: Admission
    handler = withAdmission(handler, c.AdmissionControl, ...)

    // Layer 17: Priority & Fairness
    handler = withPriorityAndFairness(handler, c.FlowControl, ...)

    // Layer 16: Authorization
    handler = withAuthorization(handler, c.Authorization.Authorizer, ...)

    // Layer 15: Audit
    handler = withAudit(handler, c.AuditBackend, ...)

    // Layer 14: Impersonation
    handler = withImpersonation(handler, c.Authorization.Authorizer, ...)

    // Layer 13: Authentication
    handler = withAuthentication(handler, c.Authentication.Authenticator, ...)

    // Layers 12-1: Infrastructure filters
    handler = withCORS(handler, c.CorsAllowedOriginList, ...)
    handler = withWarningRecorder(handler)
    handler = withTimeout(handler, c.RequestTimeout)
    handler = withWaitGroup(handler, c.longRunningFunc, ...)
    handler = withRequestInfo(handler, ...)
    handler = withPanicRecovery(handler)

    return handler
}
```

---

## Authentication

### Authenticator Interface
**File**: `staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go`
```go
type Request interface {
    AuthenticateRequest(req *http.Request) (*Response, bool, error)
}

type Response struct {
    User user.Info
    Audiences Audiences
}
```

### WithAuthentication Filter
**File**: `staging/src/k8s.io/apiserver/pkg/endpoints/filters/authentication.go`
```go
func WithAuthentication(handler http.Handler, auth authenticator.Request, ...) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
        // Authenticate request
        resp, ok, err := auth.AuthenticateRequest(req)

        if !ok {
            // Authentication failed
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Add user to context
        req = req.WithContext(request.WithUser(req.Context(), resp.User))

        // Remove sensitive headers
        req.Header.Del("Authorization")

        // Continue to next handler
        handler.ServeHTTP(w, req)
    })
}
```

### X.509 Authenticator
**File**: `staging/src/k8s.io/apiserver/pkg/authentication/request/x509/x509.go`
```go
func (a *Authenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
    if req.TLS == nil || len(req.TLS.PeerCertificates) == 0 {
        return nil, false, nil
    }

    cert := req.TLS.PeerCertificates[0]

    // Extract username from CN, groups from O
    user := &user.DefaultInfo{
        Name:   cert.Subject.CommonName,
        Groups: cert.Subject.Organization,
    }

    return &authenticator.Response{User: user}, true, nil
}
```

---

## Authorization

### Authorizer Interface
**File**: `staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go`
```go
type Authorizer interface {
    Authorize(ctx context.Context, a Attributes) (Decision, string, error)
}

type Decision int
const (
    DecisionAllow Decision = iota
    DecisionDeny
    DecisionNoOpinion
)

type Attributes interface {
    GetUser() user.Info
    GetVerb() string
    GetNamespace() string
    GetResource() string
    GetSubresource() string
    GetName() string
    GetAPIGroup() string
    GetAPIVersion() string
}
```

### WithAuthorization Filter
**File**: `staging/src/k8s.io/apiserver/pkg/endpoints/filters/authorization.go`
```go
func WithAuthorization(handler http.Handler, a authorizer.Authorizer, ...) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
        // Build authorization attributes
        attrs := authorizer.AttributesRecord{
            User:            request.UserFrom(ctx),
            Verb:            requestInfo.Verb,
            Namespace:       requestInfo.Namespace,
            Resource:        requestInfo.Resource,
            // ...
        }

        // Authorize
        decision, reason, err := a.Authorize(ctx, attrs)

        if decision != authorizer.DecisionAllow {
            http.Error(w, "Forbidden", http.StatusForbidden)
            return
        }

        handler.ServeHTTP(w, req)
    })
}
```

---

## Admission Control

### Admission Interface
**File**: `staging/src/k8s.io/apiserver/pkg/admission/interfaces.go`
```go
type Interface interface {
    Handles(operation Operation) bool
}

type MutationInterface interface {
    Interface
    Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error
}

type ValidationInterface interface {
    Interface
    Validate(ctx context.Context, a Attributes, o ObjectInterfaces) error
}

type Attributes interface {
    GetName() string
    GetNamespace() string
    GetResource() schema.GroupVersionResource
    GetSubresource() string
    GetOperation() Operation
    GetObject() runtime.Object
    GetOldObject() runtime.Object
    GetUserInfo() user.Info
    IsDryRun() bool
}
```

### Chain Handler
**File**: `staging/src/k8s.io/apiserver/pkg/admission/chain.go`
```go
func (admissionHandler chainAdmissionHandler) Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error {
    for _, handler := range admissionHandler {
        if !handler.Handles(a.GetOperation()) {
            continue
        }
        if mutator, ok := handler.(MutationInterface); ok {
            err := mutator.Admit(ctx, a, o)
            if err != nil {
                return err
            }
        }
    }
    return nil
}
```

---

## Storage Layer

### Storage Interface
**File**: `staging/src/k8s.io/apiserver/pkg/storage/interfaces.go`
```go
type Interface interface {
    Versioner() Versioner
    Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error
    Delete(ctx context.Context, key string, out runtime.Object, preconditions *Preconditions, ...) error
    Watch(ctx context.Context, key string, opts ListOptions) (watch.Interface, error)
    Get(ctx context.Context, key string, opts GetOptions, objPtr runtime.Object) error
    GetList(ctx context.Context, key string, opts ListOptions, listObj runtime.Object) error
    GuaranteedUpdate(ctx context.Context, key string, destination runtime.Object, ignoreNotFound bool, preconditions *Preconditions, tryUpdate UpdateFunc, cachedExistingObject runtime.Object) error
}

type Preconditions struct {
    UID *types.UID
    ResourceVersion *string
}
```

### etcd3 Store
**File**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go`
```go
type store struct {
    client *clientv3.Client
    codec runtime.Codec
    versioner storage.Versioner
    transformer value.Transformer
    pathPrefix string
    // ...
}

func (s *store) Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error {
    // Encode object
    data, err := runtime.Encode(s.codec, obj)

    // Transform (encrypt)
    data, err = s.transformer.TransformToStorage(data, ...)

    // Create in etcd
    txnResp, err := s.client.KV.Txn(ctx).If(
        notFound(key),
    ).Then(
        clientv3.OpPut(key, string(data), clientv3.WithLease(...)),
    ).Commit()

    return decode(txnResp, out)
}
```

---

## Watch Cache (Cacher)

### Cacher Structure
**File**: `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go`
```go
type Cacher struct {
    storage storage.Interface
    versioner storage.Versioner
    objectType reflect.Type
    watchCache *watchCache
    reflector *cache.Reflector
    // ...
}

func (c *Cacher) Watch(ctx context.Context, key string, opts storage.ListOptions) (watch.Interface, error) {
    // Check if resourceVersion in cache
    if c.watchCache.canWatch(opts.ResourceVersion) {
        return c.watchCache.Watch(ctx, key, opts)
    }

    // Fall through to underlying storage
    return c.storage.Watch(ctx, key, opts)
}
```

### Watch Cache
**File**: `staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go`
```go
type watchCache struct {
    sync.RWMutex
    clock clock.Clock
    store cache.Indexer
    resourceVersion uint64
    eventBuffer *eventBuffer
    bookmarkTimer *time.Timer
    // ...
}

func (w *watchCache) Add(obj interface{}) error {
    event := watchCacheEvent{
        Type: watch.Added,
        Object: obj,
        ResourceVersion: getResourceVersion(obj),
    }
    w.eventBuffer.Add(event)
    w.broadcastEvent(event)
    return nil
}
```

---

## Generic Registry

### Store Structure
**File**: `pkg/registry/generic/registry/store.go`
```go
type Store struct {
    NewFunc func() runtime.Object
    NewListFunc func() runtime.Object
    CreateStrategy rest.RESTCreateStrategy
    UpdateStrategy rest.RESTUpdateStrategy
    DeleteStrategy rest.RESTDeleteStrategy
    DefaultQualifiedResource schema.GroupResource
    Storage storage.Interface
    Decorator ObjectFunc
    // ...
}

func (e *Store) Create(ctx context.Context, obj runtime.Object, createValidation rest.ValidateObjectFunc, options *metav1.CreateOptions) (runtime.Object, error) {
    // Prepare for create
    e.CreateStrategy.PrepareForCreate(ctx, obj)

    // Validate
    if errs := e.CreateStrategy.Validate(ctx, obj); len(errs) != 0 {
        return nil, errors.NewInvalid(...)
    }

    // Store in etcd
    key, err := e.KeyFunc(ctx, name)
    out := e.NewFunc()
    if err := e.Storage.Create(ctx, key, obj, out, ttl); err != nil {
        return nil, err
    }

    return out, nil
}
```

---

## API Installation

### Kube API Server New()
**File**: `pkg/controlplane/instance.go:312-384`
```go
func (c CompletedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
    // Create control plane server
    cp, err := c.ControlPlane.New(controlplaneapiserver.KubeAPIServer, delegationTarget)

    s := &Instance{ControlPlane: cp}

    // Get storage providers
    client, _ := kubernetes.NewForConfig(c.ControlPlane.Generic.LoopbackClientConfig)
    restStorageProviders, err := c.StorageProviders(client)

    // Install API groups
    if err := s.ControlPlane.InstallAPIs(restStorageProviders...); err != nil {
        return nil, err
    }

    // Setup controllers
    // PostStartHooks...

    return s, nil
}
```

### Storage Providers
**File**: `pkg/controlplane/instance.go:386-442`
```go
func (c CompletedConfig) StorageProviders(client *kubernetes.Clientset) ([]controlplaneapiserver.RESTStorageProvider, error) {
    legacyRESTStorageProvider, err := corerest.New(corerest.Config{...})

    providers := []controlplaneapiserver.RESTStorageProvider{
        legacyRESTStorageProvider,              // Core API
        apiserverinternalrest.StorageProvider{},
        authenticationrest.RESTStorageProvider{...},
        authorizationrest.RESTStorageProvider{...},
        autoscalingrest.RESTStorageProvider{},
        batchrest.RESTStorageProvider{},
        certificatesrest.RESTStorageProvider{...},
        coordinationrest.RESTStorageProvider{},
        discoveryrest.StorageProvider{},
        networkingrest.RESTStorageProvider{},
        noderest.RESTStorageProvider{},
        policyrest.RESTStorageProvider{},
        rbacrest.RESTStorageProvider{...},
        schedulingrest.RESTStorageProvider{},
        storagerest.RESTStorageProvider{},
        flowcontrolrest.RESTStorageProvider{...},
        appsrest.StorageProvider{},             // Apps API
        admissionregistrationrest.RESTStorageProvider{...},
        eventsrest.RESTStorageProvider{...},
        resourcerest.RESTStorageProvider{...},
    }

    return providers, nil
}
```

---

## Resource Strategies

### Pod Strategy
**File**: `pkg/registry/core/pod/strategy.go`
```go
type podStrategy struct {
    runtime.ObjectTyper
    names.NameGenerator
}

var Strategy = podStrategy{legacyscheme.Scheme, names.SimpleNameGenerator}

func (podStrategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
    pod := obj.(*api.Pod)
    pod.Generation = 1
    pod.Status = api.PodStatus{
        Phase: api.PodPending,
        QOSClass: qos.GetPodQOS(pod),
    }
    podutil.DropDisabledPodFields(pod, nil)
}

func (podStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
    pod := obj.(*api.Pod)
    opts := podutil.GetValidationOptionsFromPodSpecAndMeta(&pod.Spec, nil, &pod.ObjectMeta, nil)
    opts.ResourceIsPod = true
    return corevalidation.ValidatePodCreate(pod, opts)
}
```

### Service Strategy
**File**: `pkg/registry/core/service/strategy.go`
```go
type serviceStrategy struct {
    runtime.ObjectTyper
    names.NameGenerator
}

var Strategy = serviceStrategy{legacyscheme.Scheme, names.SimpleNameGenerator}

func (serviceStrategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
    service := obj.(*api.Service)
    service.Status = api.ServiceStatus{}

    // Normalize cluster IPs
    normalizeClusterIPs(...)

    // Drop disabled fields
    dropServiceDisabledFields(service, nil)
}
```

---

## Summary

This code reference provides direct links to the most important components in kube-apiserver:

**Entry Points**: main() → NewAPIServerCommand() → Run()
**Server Chain**: CreateServerChain() creates 3-server delegation
**Configuration**: BuildGenericConfig() sets up infrastructure
**Handler Chain**: DefaultBuildHandlerChain() builds 24-layer pipeline
**Authentication**: Authenticator interface + strategies
**Authorization**: Authorizer interface + modes
**Admission**: Admission interface + plugins
**Storage**: Storage interface + etcd3 + cacher
**Registry**: Generic store + resource strategies

**Total Files Referenced**: 30+ core files
**Total Functions Referenced**: 50+ key functions

For complete architecture documentation, see:
- [00-README.md](../00-README.md) - Navigation
- [High-Level Architecture](../high-level/)
- [Middle-Level Architecture](../middle-level/)
- [QUICK-REFERENCE.md](../QUICK-REFERENCE.md) - Quick reference
