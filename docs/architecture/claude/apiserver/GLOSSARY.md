# Kube-APIServer Glossary

**Last Updated**: 2025-10-21
**Purpose**: Comprehensive glossary of terms, concepts, and acronyms used in kube-apiserver architecture

---

## A

### Admission Controller
A plugin that intercepts requests to the API server after authentication and authorization but before object persistence. Can modify (mutating) or reject (validating) requests.

**Types**: Built-in plugins (NamespaceLifecycle, ResourceQuota) and webhooks (MutatingAdmissionWebhook, ValidatingAdmissionWebhook)

**See**: [middle-level/06-admission-control.md](./middle-level/06-admission-control.md)

### Admission Webhook
External HTTP callbacks that receive admission requests and can validate or mutate Kubernetes objects.

**Types**:
- **Mutating**: Modifies objects before persistence
- **Validating**: Validates objects, can reject

**See**: [middle-level/06-admission-control.md](./middle-level/06-admission-control.md)

### Aggregation Layer
A mechanism that allows extending the Kubernetes API with custom API servers while maintaining a single API endpoint.

**Components**: APIService resources, request proxy, certificate-based authentication

**See**: [middle-level/11-aggregation-layer.md](./middle-level/11-aggregation-layer.md)

### API Group
A collection of related API resources organized under a common namespace (e.g., `apps`, `batch`, `networking.k8s.io`).

**Core Group**: Empty string "" (e.g., pods, services)
**Named Groups**: Non-empty strings (e.g., `apps/v1`)

**See**: [middle-level/03-api-groups-registration.md](./middle-level/03-api-groups-registration.md)

### API Priority and Fairness (APF)
A system for managing request concurrency and preventing API server overload through priority levels and fair queuing.

**Components**: FlowSchema, PriorityLevelConfiguration

**See**: [middle-level/08-api-priority-fairness.md](./middle-level/08-api-priority-fairness.md)

### APIServer
The core component of Kubernetes that exposes the Kubernetes API, processes REST operations, and stores state in etcd.

**Types**: kube-apiserver (built-in), extension API servers (custom)

**See**: [high-level/01-system-overview.md](./high-level/01-system-overview.md)

### APIService
A Kubernetes resource that registers an extension API server with the aggregation layer.

**Fields**: `group`, `version`, `service` (backend endpoint)

**See**: [middle-level/11-aggregation-layer.md](./middle-level/11-aggregation-layer.md)

### Attributes
Context information about a request used for authorization and admission decisions.

**Types**:
- **Authorization Attributes**: User, verb, resource, namespace
- **Admission Attributes**: Object, old object, operation, user info

**See**: [low-level/11-data-structures.md](./low-level/11-data-structures.md)

### Audit
Chronological record of security-relevant events in the cluster.

**Levels**: None, Metadata, Request, RequestResponse
**Backends**: Log, Webhook, Dynamic

**See**: [middle-level/09-audit-logging.md](./middle-level/09-audit-logging.md)

### Authentication
Process of verifying the identity of a client making a request.

**Strategies**: X.509 certificates, bearer tokens, OIDC, service account tokens, bootstrap tokens

**See**: [middle-level/04-authentication.md](./middle-level/04-authentication.md)

### Authorization
Process of determining whether an authenticated user has permission to perform a requested operation.

**Modes**: RBAC, Node, Webhook, ABAC

**See**: [middle-level/05-authorization.md](./middle-level/05-authorization.md)

---

## B

### Bearer Token
An authentication credential presented in HTTP `Authorization` header as `Bearer <token>`.

**Types**: Service account tokens, OIDC tokens, bootstrap tokens

**See**: [middle-level/04-authentication.md](./middle-level/04-authentication.md)

### Binding
A subresource for Pods that atomically assigns a pod to a node.

**Endpoint**: `POST /api/v1/namespaces/{namespace}/pods/{name}/binding`

**See**: [low-level/09-subresources.md](./low-level/09-subresources.md)

### Bookmark
A special watch event that communicates the current resource version without indicating a change to any object.

**Purpose**: Progress notification, watch-list consistency

**See**: [middle-level/07-watch-mechanism.md](./middle-level/07-watch-mechanism.md), [low-level/04-cacher-architecture.md](./low-level/04-cacher-architecture.md)

---

## C

### Cacher
The watch cache component that maintains an in-memory sliding window of recent resource changes to reduce etcd load.

**Components**: watchCache (circular buffer), Reflector, event dispatcher

**See**: [low-level/04-cacher-architecture.md](./low-level/04-cacher-architecture.md)

### CAS (Compare-And-Swap)
An atomic operation that updates a value only if it hasn't changed since it was read, using etcd's ModRevision for comparison.

**Used In**: GuaranteedUpdate, OptimisticPut/Delete

**See**: [low-level/03-storage-interface.md](./low-level/03-storage-interface.md)

### CEL (Common Expression Language)
A language for expressing validation rules in ValidatingAdmissionPolicy.

**Syntax**: C-like expressions (e.g., `object.spec.replicas >= 0`)

**See**: [middle-level/06-admission-control.md](./middle-level/06-admission-control.md)

### ClusterRole
A non-namespaced RBAC role that can grant permissions across the entire cluster.

**Scope**: Cluster-scoped resources and all namespaces

**See**: [middle-level/05-authorization.md](./middle-level/05-authorization.md)

### ClusterRoleBinding
Binds a ClusterRole to users, groups, or service accounts cluster-wide.

**See**: [middle-level/05-authorization.md](./middle-level/05-authorization.md)

### Codec
An encoder/decoder that serializes objects to/from bytes.

**Formats**: JSON, YAML, Protobuf
**Types**: Universal codec, versioning codec

**See**: [low-level/05-type-system.md](./low-level/05-type-system.md)

### Conversion
The process of transforming an object from one API version to another.

**Pattern**: Hub-and-spoke (all conversions through internal version)
**Types**: Auto-generated, manual (override)

**See**: [low-level/06-conversion-framework.md](./low-level/06-conversion-framework.md)

### Context (Go context.Context)
A Go standard library type that carries deadlines, cancellation signals, and request-scoped values across API boundaries.

**Uses**: Request timeouts, cancellation propagation, trace IDs

**See**: [low-level/01-handler-chain-construction.md](./low-level/01-handler-chain-construction.md)

### CRUD
Create, Read, Update, Delete - the fundamental operations supported by the REST API.

**HTTP Verbs**: POST (create), GET (read), PUT/PATCH (update), DELETE

**See**: [low-level/03-storage-interface.md](./low-level/03-storage-interface.md)

---

## D

### Decorator
A function applied to objects on read that transforms them without modifying stored representation.

**Uses**: Normalizing ClusterIPs, defaulting fields, compatibility transformations

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

### DefaulterFunc
A function that sets default values on an object.

**Generated**: Auto-generated by defaulter-gen tool
**Pattern**: `SetDefaults_<Type>(obj *Type)`

**See**: [low-level/05-type-system.md](./low-level/05-type-system.md)

### Discovery
The API that exposes available resources, versions, and operations.

**Endpoints**: `/api`, `/apis`, `/openapi/v2`, `/openapi/v3`

**See**: [middle-level/10-openapi-discovery.md](./middle-level/10-openapi-discovery.md)

### DryRun
A mode where the API server validates and processes a request without persisting changes.

**Parameter**: `dryRun=All` in query or options
**Levels**: Server-side (default), client-side (deprecated)

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

---

## E

### ErrorList
A collection of field validation errors with paths identifying invalid fields.

**Type**: `field.ErrorList`
**Methods**: `ToAggregate()`, `Filter()`, `WithOrigin()`

**See**: [low-level/07-validation-framework.md](./low-level/07-validation-framework.md)

### etcd
A distributed key-value store that serves as Kubernetes' backing store for all cluster data.

**Version**: etcd v3 (using gRPC)
**Features**: MVCC, watch, transactions, leases

**See**: [middle-level/02-storage-layer.md](./middle-level/02-storage-layer.md)

### Eviction
A subresource for Pods that implements graceful pod eviction with PodDisruptionBudget support.

**Endpoint**: `POST /api/v1/namespaces/{namespace}/pods/{name}/eviction`

**See**: [low-level/09-subresources.md](./low-level/09-subresources.md)

### External Type
A versioned API type exposed to clients (e.g., `v1.Pod`), as opposed to the internal type.

**Location**: `staging/src/k8s.io/api/<group>/<version>/`
**Suffix**: Usually has version in package name (e.g., `corev1`)

**See**: [low-level/05-type-system.md](./low-level/05-type-system.md)

---

## F

### Field Selector
A query parameter that filters resources based on field values.

**Syntax**: `field=value`, `field!=value`
**Examples**: `spec.nodeName=node1`, `status.phase!=Running`

**See**: [middle-level/02-storage-layer.md](./middle-level/02-storage-layer.md)

### FieldPath
A dot-notation string identifying a specific field in an object.

**Examples**: `spec.containers[0].name`, `metadata.labels`
**Type**: `field.Path`

**See**: [low-level/07-validation-framework.md](./low-level/07-validation-framework.md)

### Filter
A handler in the request processing chain that performs a specific operation (authentication, authorization, etc.).

**Count**: 24 filters in standard chain
**Categories**: Infrastructure, Security, Flow Control, Observability

**See**: [low-level/01-handler-chain-construction.md](./low-level/01-handler-chain-construction.md)

### Finalizer
A string key in `metadata.finalizers` that prevents deletion until removed, allowing cleanup operations.

**Pattern**: Reverse DNS format (e.g., `kubernetes.io/pv-protection`)
**Mechanism**: Object enters "deleting" state until finalizers cleared

**See**: [middle-level/02-storage-layer.md](./middle-level/02-storage-layer.md)

### FlowSchema
An APF resource that classifies requests into priority levels based on matching rules.

**Fields**: `matchingPrecedence`, `distinguisherMethod`, `priorityLevelConfiguration`

**See**: [middle-level/08-api-priority-fairness.md](./middle-level/08-api-priority-fairness.md)

---

## G

### Generation
A monotonically increasing number in `metadata.generation` that increments when `spec` changes.

**Use**: Controllers track processed generations in `status.observedGeneration`

**See**: [low-level/05-type-system.md](./low-level/05-type-system.md)

### Generic Store
The reusable storage implementation (`genericregistry.Store`) used by most resources.

**Features**: CRUD operations, strategies, hooks, decorators
**Location**: `staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go`

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

### GracefulDelete
A deletion mode that respects a grace period before forcefully removing an object.

**Field**: `metadata.deletionGracePeriodSeconds`
**Phases**: Graceful delete → finalizer cleanup → actual deletion

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

### GroupVersionKind (GVK)
A tuple uniquely identifying a Kubernetes API type.

**Components**:
- **Group**: API group (e.g., "apps")
- **Version**: API version (e.g., "v1")
- **Kind**: Type name (e.g., "Deployment")

**Example**: `{Group: "apps", Version: "v1", Kind: "Deployment"}`

**See**: [low-level/05-type-system.md](./low-level/05-type-system.md)

### GroupVersionResource (GVR)
A tuple identifying a REST resource endpoint.

**Components**: Group, Version, Resource (plural name)
**Example**: `{Group: "apps", Version: "v1", Resource: "deployments"}`

**See**: [middle-level/03-api-groups-registration.md](./middle-level/03-api-groups-registration.md)

### GuaranteedUpdate
A storage operation that atomically updates an object using optimistic locking with automatic retry on conflict.

**Signature**: `GuaranteedUpdate(ctx, key, dest, ignoreNotFound, preconditions, tryUpdate, cached)`
**Mechanism**: Read → Modify → CAS with retry loop

**See**: [low-level/03-storage-interface.md](./low-level/03-storage-interface.md)

---

## H

### Handler Chain
The sequence of 24 filters through which all API requests pass.

**Order**: Panic Recovery → Request Info → ... → Impersonation → ... → Authorization → ... → APF → ... → Dispatcher

**See**: [low-level/01-handler-chain-construction.md](./low-level/01-handler-chain-construction.md)

### Hook
A callback function invoked at specific points in an operation's lifecycle.

**Types**: BeginCreate, AfterCreate, BeginUpdate, AfterUpdate, AfterDelete
**Use**: Resource allocation, cleanup, side effects

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

### Hub-and-Spoke
The conversion architecture where all API versions convert through a central internal version (hub).

**Benefit**: O(N) conversions instead of O(N²)
**Hub**: Internal version (unversioned)

**See**: [low-level/06-conversion-framework.md](./low-level/06-conversion-framework.md)

---

## I

### Impersonation
Acting as another user/group/extra for debugging or privileged operations.

**Headers**: `Impersonate-User`, `Impersonate-Group`, `Impersonate-Extra-*`
**Permission**: Requires `impersonate` verb on `users`/`groups`/`serviceaccounts`

**See**: [middle-level/05-authorization.md](./middle-level/05-authorization.md)

### Indexer
A data structure that maintains additional indexes on cached data for efficient lookup.

**Examples**: Pod index by `spec.nodeName`, Service index by ClusterIP
**Type**: `cache.Indexer`

**See**: [low-level/04-cacher-architecture.md](./low-level/04-cacher-architecture.md)

### Informer
A client-side construct that watches resources, maintains a local cache, and triggers handlers on changes.

**Components**: Reflector, Store, Controller
**Package**: `k8s.io/client-go/tools/cache`

**See**: [middle-level/07-watch-mechanism.md](./middle-level/07-watch-mechanism.md)

### Internal Type
The unversioned, canonical representation of an API type used for storage and business logic.

**Location**: `pkg/apis/<group>/types.go`
**Version**: `__internal` (empty string)

**See**: [low-level/05-type-system.md](./low-level/05-type-system.md)

---

## J

### JSON Patch
A PATCH format that specifies a sequence of operations to apply.

**Content-Type**: `application/json-patch+json`
**Operations**: add, remove, replace, move, copy, test

**See**: [middle-level/01-request-pipeline.md](./middle-level/01-request-pipeline.md)

### JWT (JSON Web Token)
A compact token format used for OIDC authentication.

**Structure**: Header.Payload.Signature
**Claims**: Standard (iss, sub, exp) and custom

**See**: [middle-level/04-authentication.md](./middle-level/04-authentication.md)

---

## K

### KeyFunc
A function that generates a storage key from an object or name.

**Namespaced**: `/registry/<resource>/<namespace>/<name>`
**Cluster**: `/registry/<resource>/<name>`

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

### Kind
The type name of a Kubernetes object (e.g., "Pod", "Service").

**Convention**: UpperCamelCase, singular
**Related**: Resource (plural, lowercase)

**See**: [low-level/05-type-system.md](./low-level/05-type-system.md)

---

## L

### Label Selector
A query mechanism for filtering resources based on labels.

**Equality**: `app=nginx`, `tier!=frontend`
**Set**: `environment in (prod,staging)`, `tier notin (cache,db)`

**See**: [low-level/07-validation-framework.md](./low-level/07-validation-framework.md)

### Lease
An etcd mechanism for automatic expiration of keys, used for TTL on objects.

**Use**: Event retention, garbage collection

**See**: [middle-level/02-storage-layer.md](./middle-level/02-storage-layer.md)

### List
An operation that retrieves multiple resources matching selection criteria.

**Features**: Pagination (limit, continue), filtering (label/field selectors), resource version constraints

**See**: [low-level/03-storage-interface.md](./low-level/03-storage-interface.md)

---

## M

### Managed Fields
Metadata tracking which managers (users/controllers) own which fields for Server-Side Apply.

**Field**: `metadata.managedFields`
**Format**: FieldSet entries with manager, operation, time

**See**: [middle-level/01-request-pipeline.md](./middle-level/01-request-pipeline.md)

### Merge Patch
A PATCH format that recursively merges provided fields.

**Content-Type**: `application/merge-patch+json`
**Limitation**: Cannot remove array elements or set fields to null explicitly

**See**: [middle-level/01-request-pipeline.md](./middle-level/01-request-pipeline.md)

### ModRevision
etcd's version number for a key, incremented on every modification.

**Use**: Kubernetes resource version, optimistic locking
**Type**: int64, monotonically increasing

**See**: [low-level/03-storage-interface.md](./low-level/03-storage-interface.md)

### MVCC (Multi-Version Concurrency Control)
etcd's versioning system that maintains multiple versions of each key.

**Features**: Point-in-time reads, watch from specific revision, compaction

**See**: [middle-level/02-storage-layer.md](./middle-level/02-storage-layer.md)

---

## N

### Namespace
A logical partition for resources, providing scope and isolation.

**Scoped Resources**: Pods, Services, ConfigMaps
**Cluster-Scoped**: Nodes, PersistentVolumes, Namespaces

**See**: [high-level/01-system-overview.md](./high-level/01-system-overview.md)

### Node Authorizer
An authorization mode that grants kubelet permissions to access resources for pods running on its node.

**Permissions**: Read pods, services, endpoints; write node status, events

**See**: [middle-level/05-authorization.md](./middle-level/05-authorization.md)

---

## O

### ObjectMeta
Common metadata present in all Kubernetes resources.

**Fields**: name, namespace, labels, annotations, resourceVersion, generation, uid, creationTimestamp, deletionTimestamp, finalizers

**See**: [low-level/11-data-structures.md](./low-level/11-data-structures.md)

### OIDC (OpenID Connect)
An authentication protocol built on OAuth 2.0 for identity verification.

**Tokens**: JWT with standard claims
**Provider**: External IdP (e.g., Google, Okta, Dex)

**See**: [middle-level/04-authentication.md](./middle-level/04-authentication.md)

### OpenAPI
A specification format describing the Kubernetes REST API.

**Versions**: OpenAPI v2 (Swagger), OpenAPI v3
**Endpoints**: `/openapi/v2`, `/openapi/v3`

**See**: [middle-level/10-openapi-discovery.md](./middle-level/10-openapi-discovery.md)

### Optimistic Locking
A concurrency control strategy that detects conflicts by comparing resource versions.

**Mechanism**: Read (get RV) → Modify → Write (if RV unchanged)
**On Conflict**: Retry with fresh data

**See**: [low-level/03-storage-interface.md](./low-level/03-storage-interface.md)

### OwnerReference
A link in `metadata.ownerReferences` establishing parent-child relationships for garbage collection.

**Fields**: `apiVersion`, `kind`, `name`, `uid`, `controller` (bool), `blockOwnerDeletion` (bool)

**See**: [low-level/11-data-structures.md](./low-level/11-data-structures.md)

---

## P

### Preconditions
Constraints that must be satisfied for an operation to proceed.

**Types**: UID match, ResourceVersion match
**Use**: Delete protection, optimistic updates

**See**: [low-level/03-storage-interface.md](./low-level/03-storage-interface.md)

### Predicate
A filtering function that tests whether an object matches selection criteria.

**Components**: Label selector, field selector, GetAttrs function
**Type**: `storage.SelectionPredicate`

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

### PriorityLevelConfiguration
An APF resource defining concurrency limits and queuing behavior for a priority level.

**Fields**: `assuredConcurrencyShares`, `limitResponse` (queue/reject)

**See**: [middle-level/08-api-priority-fairness.md](./middle-level/08-api-priority-fairness.md)

### Protobuf
A binary serialization format used for efficient API communication.

**Advantage**: 3-10x smaller than JSON, faster encode/decode
**Content-Type**: `application/vnd.kubernetes.protobuf`

**See**: [middle-level/01-request-pipeline.md](./middle-level/01-request-pipeline.md)

---

## Q

### QPS (Queries Per Second)
Rate limiting configuration for client requests.

**Client-Side**: `--qps` and `--burst` flags
**Server-Side**: APF FlowSchema configuration

**See**: [middle-level/08-api-priority-fairness.md](./middle-level/08-api-priority-fairness.md)

---

## R

### RBAC (Role-Based Access Control)
An authorization mechanism using roles and role bindings.

**Components**: Role, ClusterRole, RoleBinding, ClusterRoleBinding
**Permissions**: Verbs (get, list, create, etc.) on resources

**See**: [middle-level/05-authorization.md](./middle-level/05-authorization.md)

### Reflector
A client-go component that lists resources and watches for changes, keeping a local store synchronized.

**Operations**: List (initial sync) → Watch (ongoing updates)
**Package**: `k8s.io/client-go/tools/cache`

**See**: [low-level/04-cacher-architecture.md](./low-level/04-cacher-architecture.md)

### Registry
The component that implements REST storage for a resource type.

**Pattern**: Interface-based (Getter, Lister, Creater, Updater, etc.)
**Implementation**: Usually wraps `genericregistry.Store`

**See**: [low-level/02-registry-pattern.md](./low-level/02-registry-pattern.md)

### RequestInfo
Parsed metadata about an HTTP request including resource, verb, namespace, and name.

**Fields**: Verb, APIGroup, APIVersion, Resource, Subresource, Namespace, Name, Parts

**See**: [low-level/11-data-structures.md](./low-level/11-data-structures.md)

### Resource
The plural, lowercase name used in REST URLs (e.g., "pods", "services").

**Mapping**: Kind → Resource (usually add 's', handle irregulars)
**Example**: Deployment → deployments, Endpoints → endpoints

**See**: [middle-level/03-api-groups-registration.md](./middle-level/03-api-groups-registration.md)

### ResourceVersion
A string representing the version of a resource, derived from etcd's ModRevision.

**Format**: Base-10 string (e.g., "12345")
**Use**: Optimistic locking, watch starting point, list consistency

**See**: [low-level/03-storage-interface.md](./low-level/03-storage-interface.md)

### REST Storage
The storage backend that implements REST operations for a resource.

**Interface**: `rest.Storage` and sub-interfaces (Getter, Lister, etc.)

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

### RESTMapper
A component that maps between GVK (GroupVersionKind) and GVR (GroupVersionResource).

**Use**: Converting client requests to storage keys

**See**: [middle-level/03-api-groups-registration.md](./middle-level/03-api-groups-registration.md)

---

## S

### Scale
A subresource providing a uniform interface for updating replica count.

**Endpoint**: `PUT /apis/apps/v1/namespaces/{ns}/deployments/{name}/scale`
**Type**: `autoscaling/v1.Scale`

**See**: [low-level/09-subresources.md](./low-level/09-subresources.md)

### Scheme
A registry mapping between GVK and Go types, plus conversion/default functions.

**Type**: `runtime.Scheme`
**Operations**: Type registration, version conversion, defaulting

**See**: [low-level/05-type-system.md](./low-level/05-type-system.md)

### SelectionPredicate
Filtering criteria combining label selector, field selector, and attribute extractor.

**Type**: `storage.SelectionPredicate`
**Use**: List and Watch operations

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

### Server-Side Apply (SSA)
A PATCH mechanism where the server merges changes while tracking field ownership.

**Content-Type**: `application/apply-patch+yaml`
**Parameter**: `?fieldManager=<name>`

**See**: [middle-level/01-request-pipeline.md](./middle-level/01-request-pipeline.md)

### Service Account
A Kubernetes identity used by pods to authenticate to the API server.

**Token**: JWT mounted as volume or projected
**Default**: `default` service account in each namespace

**See**: [middle-level/04-authentication.md](./middle-level/04-authentication.md)

### Status Subresource
A special subresource for updating only the `status` field of a resource, separate from `spec`.

**Endpoint**: `PUT /apis/apps/v1/namespaces/{ns}/deployments/{name}/status`
**Benefit**: Allows different RBAC permissions for spec vs status

**See**: [low-level/09-subresources.md](./low-level/09-subresources.md)

### Storage Version
The API version used to persist objects in etcd.

**Selection**: Usually the first version registered (often internal version)
**Interface**: `runtime.GroupVersioner`

**See**: [low-level/06-conversion-framework.md](./low-level/06-conversion-framework.md)

### Strategic Merge Patch
A PATCH format with Kubernetes-specific semantics for arrays and fields.

**Content-Type**: `application/strategic-merge-patch+json`
**Features**: `patchStrategy` tags, `patchMergeKey` for arrays

**See**: [middle-level/01-request-pipeline.md](./middle-level/01-request-pipeline.md)

### Strategy
A pluggable component that customizes validation and preparation for a resource.

**Types**: RESTCreateStrategy, RESTUpdateStrategy, RESTDeleteStrategy
**Methods**: PrepareForCreate, Validate, Canonicalize

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

### Subresource
A secondary endpoint under a resource for specialized operations.

**Examples**: status, scale, binding, log, exec, attach, portforward, proxy

**See**: [low-level/09-subresources.md](./low-level/09-subresources.md)

---

## T

### Table
A structured output format for `kubectl get` that presents resources in tabular form.

**Conversion**: `rest.TableConvertor` transforms objects to tables
**Type**: `meta/v1.Table`

**See**: [middle-level/10-openapi-discovery.md](./middle-level/10-openapi-discovery.md)

### Transformer
A component that transforms data during storage (encryption/decryption).

**Types**: Identity (no-op), AES-CBC, AES-GCM, KMS plugin
**Interface**: `value.Transformer`

**See**: [middle-level/02-storage-layer.md](./middle-level/02-storage-layer.md)

### TTL (Time To Live)
The duration after which a resource expires and is automatically deleted.

**Mechanism**: etcd leases
**Use**: Events, temporary resources

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

---

## U

### UID
A unique, immutable identifier for a resource instance.

**Format**: UUID (e.g., `550e8400-e29b-41d4-a716-446655440000`)
**Field**: `metadata.uid`
**Use**: OwnerReferences, preconditions

**See**: [low-level/11-data-structures.md](./low-level/11-data-structures.md)

### UpdatedObjectInfo
An interface that generates updated objects from old objects, used in Update operations.

**Methods**: `UpdatedObject(ctx, old) (new, error)`, `Preconditions()`
**Use**: Enables different update strategies (replace, patch)

**See**: [low-level/08-rest-storage-impl.md](./low-level/08-rest-storage-impl.md)

---

## V

### Validation
The process of checking that an object conforms to schema and business rules.

**Layers**: Schema validation, field validation, object validation, admission validation
**Type**: `field.ErrorList`

**See**: [low-level/07-validation-framework.md](./low-level/07-validation-framework.md)

### Verb
The operation being performed on a resource.

**Standard**: get, list, watch, create, update, patch, delete, deletecollection
**Special**: proxy, bind, impersonate

**See**: [middle-level/05-authorization.md](./middle-level/05-authorization.md)

### Versioner
An interface that manages resource versions on objects.

**Methods**: `UpdateObject`, `PrepareObjectForStorage`, `ObjectResourceVersion`, `ParseResourceVersion`
**Implementation**: `APIObjectVersioner`

**See**: [low-level/03-storage-interface.md](./low-level/03-storage-interface.md)

---

## W

### Watch
A streaming API that notifies clients of resource changes.

**Events**: ADDED, MODIFIED, DELETED, BOOKMARK, ERROR
**Protocol**: Chunked HTTP, WebSocket, or HTTP/2

**See**: [middle-level/07-watch-mechanism.md](./middle-level/07-watch-mechanism.md)

### Watch Cache
See **Cacher**

### WatchCache
The sliding window component within Cacher that stores recent events.

**Structure**: Circular buffer, store (current state)
**Capacity**: Dynamic (100 to 100K+ events)

**See**: [low-level/04-cacher-architecture.md](./low-level/04-cacher-architecture.md)

### Webhook
An HTTP callback for extending Kubernetes functionality.

**Types**:
- Admission webhooks (mutating, validating)
- Authorization webhooks
- CRD conversion webhooks
- Audit webhooks

**See**: [middle-level/06-admission-control.md](./middle-level/06-admission-control.md)

---

## X

### X.509 Certificate
A standard for public key certificates used for client authentication.

**Fields**: Subject (CN=username, O=group), SAN (alternative names)
**Use**: Kubectl client certs, kubelet certs

**See**: [middle-level/04-authentication.md](./middle-level/04-authentication.md)

---

## Acronyms

- **ABAC**: Attribute-Based Access Control
- **APF**: API Priority and Fairness
- **API**: Application Programming Interface
- **CAS**: Compare-And-Swap
- **CEL**: Common Expression Language
- **CRD**: Custom Resource Definition
- **CRUD**: Create, Read, Update, Delete
- **GC**: Garbage Collection
- **GVK**: GroupVersionKind
- **GVR**: GroupVersionResource
- **HTTP**: Hypertext Transfer Protocol
- **IdP**: Identity Provider
- **JWT**: JSON Web Token
- **KMS**: Key Management Service
- **MVCC**: Multi-Version Concurrency Control
- **OAuth**: Open Authorization
- **OIDC**: OpenID Connect
- **PDB**: PodDisruptionBudget
- **QPS**: Queries Per Second
- **RBAC**: Role-Based Access Control
- **REST**: Representational State Transfer
- **RV**: Resource Version
- **SA**: Service Account
- **SAN**: Subject Alternative Name
- **SSA**: Server-Side Apply
- **TLS**: Transport Layer Security
- **TTL**: Time To Live
- **UID**: Unique Identifier
- **UUID**: Universally Unique Identifier
- **YAML**: YAML Ain't Markup Language

---

## Common Patterns

### Optimistic Concurrency Pattern
```
1. Read object (get current ResourceVersion)
2. Modify object
3. Update with ResourceVersion precondition
4. If conflict → retry from step 1
```

### Watch Pattern
```
1. List resources (get initial state + ResourceVersion)
2. Watch from ResourceVersion + 1
3. Process ADDED/MODIFIED/DELETED events
4. On error → re-list and re-watch
```

### Strategy Pattern
```
Resources customize behavior via pluggable strategies:
- CreateStrategy: PrepareForCreate, Validate
- UpdateStrategy: PrepareForUpdate, ValidateUpdate
- DeleteStrategy: CheckGracefulDelete
```

### Hub-and-Spoke Conversion
```
v1 ↔ internal ↔ v1beta2
(not v1 ↔ v1beta2 directly)
```

### Handler Chain Pattern
```
Request → Filter1 → Filter2 → ... → Filter24 → Dispatcher
Each filter: handle() or next.ServeHTTP()
```

---

## Quick Reference

### Storage Key Format
- **Namespaced**: `/registry/<resource>/<namespace>/<name>`
- **Cluster**: `/registry/<resource>/<name>`

### HTTP Status Codes
- **200 OK**: Successful GET, PUT, PATCH
- **201 Created**: Successful POST
- **204 No Content**: Successful DELETE
- **400 Bad Request**: Invalid request
- **401 Unauthorized**: Missing/invalid authentication
- **403 Forbidden**: Authenticated but not authorized
- **404 Not Found**: Resource doesn't exist
- **409 Conflict**: Resource version conflict
- **422 Unprocessable Entity**: Validation failed
- **429 Too Many Requests**: Rate limited
- **500 Internal Server Error**: Server-side error
- **503 Service Unavailable**: Overloaded or unhealthy

### API Versions
- **alpha** (v1alpha1): Unstable, may change
- **beta** (v1beta1): Stable-ish, backwards incompatible changes possible
- **stable** (v1): Stable, backwards compatible

### Watch Event Types
- **ADDED**: New object created
- **MODIFIED**: Object updated
- **DELETED**: Object removed
- **BOOKMARK**: Progress notification (no change)
- **ERROR**: Watch error occurred

---

**Total Terms**: 150+
**Categories**: 26 (A-Z)
**Cross-References**: Extensive linking to detailed documentation

**See Also**: [QUICK-REFERENCE.md](./QUICK-REFERENCE.md) for code-level quick reference

---

*Last Updated*: 2025-10-21
*Maintained By*: Kube-APIServer Documentation Project
*Purpose*: Comprehensive terminology reference for developers and operators
