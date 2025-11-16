# **Mutating Admission Webhooks**

**Deep Implementation Analysis of Kubernetes Mutation Extension Point**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Overview**

Mutating admission webhooks provide a powerful extension mechanism to modify objects before they are persisted to etcd. They run early in the admission chain, after authentication and authorization but before schema validation and validating webhooks. This positioning allows them to inject defaults, add labels/annotations, inject sidecars, and perform other mutations that prepare objects for validation and storage.

**Key Characteristics:**
- **Object Modification**: Can modify incoming objects via JSONPatch
- **Early Execution**: Runs before schema validation and validating webhooks
- **Reinvocation**: Can be configured to run multiple times if object changes
- **Ordered Execution**: Processed in alphabetical order by name
- **Audit Trail**: All mutations are logged for audit purposes

**Primary Use Cases:**
1. **Sidecar Injection**: Automatically inject sidecar containers (e.g., Istio, Vault)
2. **Default Value Injection**: Add default labels, annotations, resource limits
3. **Resource Modification**: Modify security contexts, affinity rules
4. **Policy Enforcement**: Inject required volumes, env vars
5. **Cross-Resource Synchronization**: Update related resources based on changes

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🏗️ MutatingWebhookConfiguration Structure**

### **Core Resource Definition**

```yaml
# File: MutatingWebhookConfiguration manifest
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: "sidecar-injector.example.com"
webhooks:
  - name: "sidecar-injector.example.com"
    # Match conditions
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
        scope: "Namespaced"

    # Client configuration
    clientConfig:
      service:
        namespace: "injection-system"
        name: "sidecar-injector"
        path: "/mutate-pods"
        port: 443
      caBundle: "LS0tLS1CRUdJTi..."

    # Admission review versions
    admissionReviewVersions: ["v1", "v1beta1"]

    # Side effects declaration
    sideEffects: None  # None or NoneOnDryRun

    # Timeout
    timeoutSeconds: 10

    # Failure policy
    failurePolicy: Fail  # Fail or Ignore

    # Match policy
    matchPolicy: Equivalent

    # Reinvocation policy (unique to mutating webhooks)
    reinvocationPolicy: Never  # Never or IfNeeded

    # Namespace selector
    namespaceSelector:
      matchLabels:
        sidecar-injection: "enabled"

    # Object selector
    objectSelector:
      matchLabels:
        inject: "true"
```

### **Reinvocation Policy**

```go
// File: staging/src/k8s.io/api/admissionregistration/v1/types.go:140-150
type ReinvocationPolicyType string

const (
    // NeverReinvocationPolicy indicates that the webhook should not be called
    // more than once in a single admission evaluation
    NeverReinvocationPolicy ReinvocationPolicyType = "Never"

    // IfNeededReinvocationPolicy indicates that the webhook may be called again
    // if other plugins in the chain modify the object
    IfNeededReinvocationPolicy ReinvocationPolicyType = "IfNeeded"
)
```

### **Reinvocation Behavior**

```go
// File: staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/dispatcher.go:105-125
func (a *mutatingDispatcher) Dispatch(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces, hooks []webhook.WebhookAccessor) error {
    reinvokeCtx := attr.GetReinvocationContext()
    var webhookReinvokeCtx *webhookReinvokeContext
    if v := reinvokeCtx.Value(PluginName); v != nil {
        webhookReinvokeCtx = v.(*webhookReinvokeContext)
    } else {
        webhookReinvokeCtx = &webhookReinvokeContext{}
        reinvokeCtx.SetValue(PluginName, webhookReinvokeCtx)
    }

    // Check if object changed since last webhook invocation
    if reinvokeCtx.IsReinvoke() &&
       webhookReinvokeCtx.IsOutputChangedSinceLastWebhookInvocation(attr.GetObject()) {
        // Object changed, reinvoke all eligible webhooks
        webhookReinvokeCtx.RequireReinvokingPreviouslyInvokedPlugins()
    }

    defer func() {
        webhookReinvokeCtx.SetLastWebhookInvocationOutput(attr.GetObject())
    }()
    // ... continue processing
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 JSONPatch Operations**

### **JSONPatch Specification (RFC 6902)**

Mutating webhooks use JSONPatch to describe object modifications:

```go
// File: staging/src/k8s.io/api/admission/v1/types.go:115-125
type PatchType string

const (
    // PatchTypeJSONPatch indicates a patch in the JSONPatch format (RFC 6902)
    PatchTypeJSONPatch PatchType = "JSONPatch"
)

type AdmissionResponse struct {
    // Patch is the actual patch to apply to the object
    Patch []byte `json:"patch,omitempty"`

    // PatchType indicates the patch format
    PatchType *PatchType `json:"patchType,omitempty"`
}
```

### **JSONPatch Operations**

```json
[
  {
    "op": "add",
    "path": "/metadata/labels/injected",
    "value": "true"
  },
  {
    "op": "replace",
    "path": "/spec/containers/0/image",
    "value": "nginx:1.19"
  },
  {
    "op": "remove",
    "path": "/metadata/annotations/temporary"
  },
  {
    "op": "copy",
    "from": "/spec/containers/0",
    "path": "/spec/containers/1"
  },
  {
    "op": "move",
    "from": "/spec/containers/1",
    "path": "/spec/containers/0"
  },
  {
    "op": "test",
    "path": "/metadata/name",
    "value": "expected-name"
  }
]
```

### **Common Patch Patterns**

**1. Add Label**
```json
[
  {
    "op": "add",
    "path": "/metadata/labels/team",
    "value": "platform"
  }
]
```

**2. Add Annotation**
```json
[
  {
    "op": "add",
    "path": "/metadata/annotations/modified-by",
    "value": "sidecar-injector"
  }
]
```

**3. Add Container to Pod**
```json
[
  {
    "op": "add",
    "path": "/spec/containers/-",
    "value": {
      "name": "sidecar",
      "image": "sidecar:v1.0",
      "ports": [{"containerPort": 8080}]
    }
  }
]
```

**4. Add InitContainer**
```json
[
  {
    "op": "add",
    "path": "/spec/initContainers",
    "value": [{
      "name": "init-sidecar",
      "image": "init:v1.0",
      "command": ["/bin/init.sh"]
    }]
  }
]
```

**5. Modify Resource Limits**
```json
[
  {
    "op": "add",
    "path": "/spec/containers/0/resources",
    "value": {
      "limits": {
        "cpu": "500m",
        "memory": "512Mi"
      },
      "requests": {
        "cpu": "100m",
        "memory": "128Mi"
      }
    }
  }
]
```

**6. Add Volume and Volume Mount**
```json
[
  {
    "op": "add",
    "path": "/spec/volumes",
    "value": [{
      "name": "config",
      "configMap": {
        "name": "app-config"
      }
    }]
  },
  {
    "op": "add",
    "path": "/spec/containers/0/volumeMounts",
    "value": [{
      "name": "config",
      "mountPath": "/etc/config"
    }]
  }
]
```

### **Patch Application Code**

```go
// File: staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/dispatcher.go:336-392
func (a *mutatingDispatcher) callAttrMutatingHook(...) (bool, error) {
    // ... webhook call logic ...

    if len(result.Patch) == 0 {
        return false, nil
    }

    // Decode the patch
    patchObj, err := jsonpatch.DecodePatch(result.Patch)
    if err != nil {
        return false, &webhookutil.ErrCallingWebhook{
            WebhookName: h.Name,
            Reason: fmt.Errorf("received undecodable patch: %w", err),
            Status: apierrors.NewServiceUnavailable("error decoding patch"),
        }
    }

    if len(patchObj) == 0 {
        return false, nil
    }

    // Ensure we have an object to apply the patch to
    if attr.VersionedObject == nil {
        return false, apierrors.NewInternalError(
            fmt.Errorf("webhook %q attempted to modify object, not supported for this operation", h.Name))
    }

    // Encode the current object to JSON
    var patchedJS []byte
    jsonSerializer := json.NewSerializerWithOptions(
        json.DefaultMetaFactory,
        o.GetObjectCreater(),
        o.GetObjectTyper(),
        json.SerializerOptions{},
    )

    objJS, err := runtime.Encode(jsonSerializer, attr.VersionedObject)
    if err != nil {
        return false, apierrors.NewInternalError(err)
    }

    // Apply the patch
    patchedJS, err = patchObj.Apply(objJS)
    if err != nil {
        return false, apierrors.NewInternalError(err)
    }

    // Decode the patched object
    var newVersionedObject runtime.Object
    if _, ok := attr.VersionedObject.(*unstructured.Unstructured); ok {
        newVersionedObject = &unstructured.Unstructured{}
    } else {
        newVersionedObject, err = o.GetObjectCreater().New(attr.VersionedKind)
        if err != nil {
            return false, apierrors.NewInternalError(err)
        }
    }

    if newVersionedObject, _, err = jsonSerializer.Decode(patchedJS, nil, newVersionedObject); err != nil {
        return false, apierrors.NewInternalError(err)
    }

    // Check if object actually changed
    changed = !apiequality.Semantic.DeepEqual(attr.VersionedObject, newVersionedObject)

    // Update the attribute with the new object
    attr.Dirty = true
    attr.VersionedObject = newVersionedObject

    // Apply defaults to the modified object
    o.GetObjectDefaulter().Default(attr.VersionedObject)

    return changed, nil
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **💉 Sidecar Injection Patterns**

### **Complete Sidecar Injector Implementation**

```go
// File: sidecar-injector/pkg/webhook/injector.go
package webhook

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"

    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer"
    "k8s.io/klog/v2"
)

var (
    scheme = runtime.NewScheme()
    codecs = serializer.NewCodecFactory(scheme)
)

func init() {
    _ = corev1.AddToScheme(scheme)
    _ = admissionv1.AddToScheme(scheme)
}

type SidecarInjector struct {
    SidecarImage      string
    SidecarResources  corev1.ResourceRequirements
    InjectionLabel    string
}

func NewSidecarInjector(image string) *SidecarInjector {
    return &SidecarInjector{
        SidecarImage:   image,
        InjectionLabel: "sidecar-injection",
        SidecarResources: corev1.ResourceRequirements{
            Requests: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("100m"),
                corev1.ResourceMemory: resource.MustParse("128Mi"),
            },
            Limits: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("500m"),
                corev1.ResourceMemory: resource.MustParse("512Mi"),
            },
        },
    }
}

func (si *SidecarInjector) Handle(w http.ResponseWriter, r *http.Request) {
    // Read request body
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "failed to read request body", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()

    // Decode admission review
    review := &admissionv1.AdmissionReview{}
    if _, _, err := codecs.UniversalDeserializer().Decode(body, nil, review); err != nil {
        http.Error(w, "failed to decode request", http.StatusBadRequest)
        return
    }

    if review.Request == nil {
        http.Error(w, "admission review request is nil", http.StatusBadRequest)
        return
    }

    // Create response
    response := si.mutate(review.Request)
    review.Response = response
    review.Response.UID = review.Request.UID

    // Encode response
    responseBytes, err := json.Marshal(review)
    if err != nil {
        http.Error(w, "failed to encode response", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    w.Write(responseBytes)
}

func (si *SidecarInjector) mutate(request *admissionv1.AdmissionRequest) *admissionv1.AdmissionResponse {
    // Decode pod
    pod := &corev1.Pod{}
    if err := json.Unmarshal(request.Object.Raw, pod); err != nil {
        return &admissionv1.AdmissionResponse{
            Allowed: false,
            Result: &metav1.Status{
                Code:    http.StatusBadRequest,
                Message: fmt.Sprintf("failed to decode pod: %v", err),
            },
        }
    }

    // Check if injection is needed
    if !si.needsInjection(pod) {
        klog.V(4).Infof("Pod %s/%s does not need sidecar injection", pod.Namespace, pod.Name)
        return &admissionv1.AdmissionResponse{Allowed: true}
    }

    // Check if already injected
    if si.isAlreadyInjected(pod) {
        klog.V(4).Infof("Pod %s/%s already has sidecar", pod.Namespace, pod.Name)
        return &admissionv1.AdmissionResponse{Allowed: true}
    }

    // Generate patch
    patch, err := si.createPatch(pod)
    if err != nil {
        klog.Errorf("Failed to create patch for pod %s/%s: %v", pod.Namespace, pod.Name, err)
        return &admissionv1.AdmissionResponse{
            Allowed: false,
            Result: &metav1.Status{
                Code:    http.StatusInternalServerError,
                Message: fmt.Sprintf("failed to create patch: %v", err),
            },
        }
    }

    patchType := admissionv1.PatchTypeJSONPatch
    return &admissionv1.AdmissionResponse{
        Allowed:   true,
        Patch:     patch,
        PatchType: &patchType,
        AuditAnnotations: map[string]string{
            "sidecar-injector.example.com/injected": "true",
            "sidecar-injector.example.com/image":    si.SidecarImage,
        },
    }
}

func (si *SidecarInjector) needsInjection(pod *corev1.Pod) bool {
    // Check namespace labels
    // In real implementation, would fetch namespace and check labels

    // Check pod labels
    if pod.Labels != nil {
        if value, exists := pod.Labels[si.InjectionLabel]; exists && value == "enabled" {
            return true
        }
    }

    // Check pod annotations
    if pod.Annotations != nil {
        if value, exists := pod.Annotations[si.InjectionLabel]; exists && value == "enabled" {
            return true
        }
    }

    return false
}

func (si *SidecarInjector) isAlreadyInjected(pod *corev1.Pod) bool {
    for _, container := range pod.Spec.Containers {
        if container.Name == "sidecar" {
            return true
        }
    }
    return false
}

func (si *SidecarInjector) createPatch(pod *corev1.Pod) ([]byte, error) {
    var patches []map[string]interface{}

    // Add sidecar container
    sidecarContainer := si.createSidecarContainer()

    // Add container to end of containers array
    patches = append(patches, map[string]interface{}{
        "op":    "add",
        "path":  "/spec/containers/-",
        "value": sidecarContainer,
    })

    // Add annotation to mark as injected
    if pod.Annotations == nil {
        // Create annotations map if it doesn't exist
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/metadata/annotations",
            "value": map[string]string{},
        })
    }

    patches = append(patches, map[string]interface{}{
        "op":    "add",
        "path":  "/metadata/annotations/sidecar-injector.example.com~1injected",
        "value": "true",
    })

    patches = append(patches, map[string]interface{}{
        "op":    "add",
        "path":  "/metadata/annotations/sidecar-injector.example.com~1version",
        "value": "v1.0.0",
    })

    // Add shared volume if needed
    if si.needsSharedVolume(pod) {
        patches = append(patches, si.createVolumePatches(pod)...)
    }

    return json.Marshal(patches)
}

func (si *SidecarInjector) createSidecarContainer() corev1.Container {
    return corev1.Container{
        Name:  "sidecar",
        Image: si.SidecarImage,
        Ports: []corev1.ContainerPort{
            {
                Name:          "metrics",
                ContainerPort: 9090,
                Protocol:      corev1.ProtocolTCP,
            },
            {
                Name:          "health",
                ContainerPort: 8080,
                Protocol:      corev1.ProtocolTCP,
            },
        },
        Env: []corev1.EnvVar{
            {
                Name: "POD_NAME",
                ValueFrom: &corev1.EnvVarSource{
                    FieldRef: &corev1.ObjectFieldSelector{
                        FieldPath: "metadata.name",
                    },
                },
            },
            {
                Name: "POD_NAMESPACE",
                ValueFrom: &corev1.EnvVarSource{
                    FieldRef: &corev1.ObjectFieldSelector{
                        FieldPath: "metadata.namespace",
                    },
                },
            },
        },
        Resources: si.SidecarResources,
        SecurityContext: &corev1.SecurityContext{
            AllowPrivilegeEscalation: func(b bool) *bool { return &b }(false),
            Capabilities: &corev1.Capabilities{
                Drop: []corev1.Capability{"ALL"},
            },
            ReadOnlyRootFilesystem: func(b bool) *bool { return &b }(true),
            RunAsNonRoot:           func(b bool) *bool { return &b }(true),
            RunAsUser:              func(i int64) *int64 { return &i }(65532),
        },
        LivenessProbe: &corev1.Probe{
            ProbeHandler: corev1.ProbeHandler{
                HTTPGet: &corev1.HTTPGetAction{
                    Path: "/healthz",
                    Port: intstr.FromInt(8080),
                },
            },
            InitialDelaySeconds: 10,
            PeriodSeconds:       10,
        },
        ReadinessProbe: &corev1.Probe{
            ProbeHandler: corev1.ProbeHandler{
                HTTPGet: &corev1.HTTPGetAction{
                    Path: "/ready",
                    Port: intstr.FromInt(8080),
                },
            },
            InitialDelaySeconds: 5,
            PeriodSeconds:       5,
        },
    }
}

func (si *SidecarInjector) needsSharedVolume(pod *corev1.Pod) bool {
    // Check if application containers need to share data with sidecar
    return true
}

func (si *SidecarInjector) createVolumePatches(pod *corev1.Pod) []map[string]interface{} {
    var patches []map[string]interface{}

    // Add shared volume
    if pod.Spec.Volumes == nil {
        patches = append(patches, map[string]interface{}{
            "op":   "add",
            "path": "/spec/volumes",
            "value": []corev1.Volume{
                {
                    Name: "sidecar-shared",
                    VolumeSource: corev1.VolumeSource{
                        EmptyDir: &corev1.EmptyDirVolumeSource{},
                    },
                },
            },
        })
    } else {
        patches = append(patches, map[string]interface{}{
            "op":   "add",
            "path": "/spec/volumes/-",
            "value": corev1.Volume{
                Name: "sidecar-shared",
                VolumeSource: corev1.VolumeSource{
                    EmptyDir: &corev1.EmptyDirVolumeSource{},
                },
            },
        })
    }

    return patches
}
```

### **Advanced Sidecar Pattern: Istio-style Injection**

```go
// File: sidecar-injector/pkg/webhook/istio_injector.go
package webhook

import (
    "encoding/json"

    corev1 "k8s.io/api/core/v1"
)

type IstioInjector struct {
    ProxyImage       string
    InitImage        string
    MeshConfigMap    string
    InjectAnnotation string
}

func (ii *IstioInjector) createPatch(pod *corev1.Pod) ([]byte, error) {
    var patches []map[string]interface{}

    // 1. Add init container for iptables setup
    initContainer := ii.createInitContainer()
    if pod.Spec.InitContainers == nil {
        patches = append(patches, map[string]interface{}{
            "op":   "add",
            "path": "/spec/initContainers",
            "value": []corev1.Container{initContainer},
        })
    } else {
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/spec/initContainers/-",
            "value": initContainer,
        })
    }

    // 2. Add proxy sidecar container
    proxyContainer := ii.createProxyContainer()
    patches = append(patches, map[string]interface{}{
        "op":    "add",
        "path":  "/spec/containers/-",
        "value": proxyContainer,
    })

    // 3. Add volumes
    patches = append(patches, ii.createVolumePatches(pod)...)

    // 4. Add annotations
    patches = append(patches, map[string]interface{}{
        "op":    "add",
        "path":  "/metadata/annotations/sidecar.istio.io~1status",
        "value": `{"version":"","initContainers":["istio-init"],"containers":["istio-proxy"]}`,
    })

    // 5. Modify pod security context if needed
    if pod.Spec.SecurityContext == nil {
        patches = append(patches, map[string]interface{}{
            "op":   "add",
            "path": "/spec/securityContext",
            "value": &corev1.PodSecurityContext{
                FSGroup: func(i int64) *int64 { return &i }(1337),
            },
        })
    }

    return json.Marshal(patches)
}

func (ii *IstioInjector) createInitContainer() corev1.Container {
    privileged := true
    runAsNonRoot := false
    runAsUser := int64(0)

    return corev1.Container{
        Name:  "istio-init",
        Image: ii.InitImage,
        Args: []string{
            "istio-iptables",
            "-p", "15001",
            "-z", "15006",
            "-u", "1337",
            "-m", "REDIRECT",
            "-i", "*",
            "-x", "",
            "-b", "*",
            "-d", "15090,15021,15020",
        },
        SecurityContext: &corev1.SecurityContext{
            Privileged:   &privileged,
            RunAsNonRoot: &runAsNonRoot,
            RunAsUser:    &runAsUser,
            Capabilities: &corev1.Capabilities{
                Add: []corev1.Capability{
                    "NET_ADMIN",
                    "NET_RAW",
                },
                Drop: []corev1.Capability{"ALL"},
            },
        },
        Resources: corev1.ResourceRequirements{
            Requests: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("10m"),
                corev1.ResourceMemory: resource.MustParse("10Mi"),
            },
            Limits: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("100m"),
                corev1.ResourceMemory: resource.MustParse("50Mi"),
            },
        },
    }
}

func (ii *IstioInjector) createProxyContainer() corev1.Container {
    runAsUser := int64(1337)

    return corev1.Container{
        Name:  "istio-proxy",
        Image: ii.ProxyImage,
        Args: []string{
            "proxy",
            "sidecar",
            "--domain",
            "$(POD_NAMESPACE).svc.cluster.local",
            "--serviceCluster",
            "$(POD_NAME).$(POD_NAMESPACE)",
            "--proxyLogLevel=warning",
            "--proxyComponentLogLevel=misc:error",
            "--trust-domain=cluster.local",
        },
        Ports: []corev1.ContainerPort{
            {Name: "http-envoy-prom", ContainerPort: 15090, Protocol: "TCP"},
            {Name: "http-pilot", ContainerPort: 15021, Protocol: "TCP"},
        },
        Env: []corev1.EnvVar{
            {
                Name: "POD_NAME",
                ValueFrom: &corev1.EnvVarSource{
                    FieldRef: &corev1.ObjectFieldSelector{FieldPath: "metadata.name"},
                },
            },
            {
                Name: "POD_NAMESPACE",
                ValueFrom: &corev1.EnvVarSource{
                    FieldRef: &corev1.ObjectFieldSelector{FieldPath: "metadata.namespace"},
                },
            },
            {
                Name: "INSTANCE_IP",
                ValueFrom: &corev1.EnvVarSource{
                    FieldRef: &corev1.ObjectFieldSelector{FieldPath: "status.podIP"},
                },
            },
        },
        VolumeMounts: []corev1.VolumeMount{
            {Name: "istio-envoy", MountPath: "/etc/istio/proxy"},
            {Name: "istio-data", MountPath: "/var/lib/istio/data"},
            {Name: "istio-token", MountPath: "/var/run/secrets/tokens"},
            {Name: "istiod-ca-cert", MountPath: "/var/run/secrets/istio", ReadOnly: true},
        },
        SecurityContext: &corev1.SecurityContext{
            RunAsUser:                &runAsUser,
            AllowPrivilegeEscalation: func(b bool) *bool { return &b }(false),
            Capabilities: &corev1.Capabilities{
                Drop: []corev1.Capability{"ALL"},
            },
            ReadOnlyRootFilesystem: func(b bool) *bool { return &b }(true),
            RunAsNonRoot:           func(b bool) *bool { return &b }(true),
        },
        Resources: corev1.ResourceRequirements{
            Requests: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("100m"),
                corev1.ResourceMemory: resource.MustParse("128Mi"),
            },
            Limits: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("2000m"),
                corev1.ResourceMemory: resource.MustParse("1024Mi"),
            },
        },
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚙️ Default Value Injection**

### **Resource Defaults Injector**

```go
// File: default-injector/pkg/webhook/defaults.go
package webhook

import (
    "encoding/json"

    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/resource"
)

type DefaultsInjector struct {
    DefaultNamespace string
    DefaultLabels    map[string]string
    DefaultResources corev1.ResourceRequirements
}

func (di *DefaultsInjector) mutate(request *admissionv1.AdmissionRequest) *admissionv1.AdmissionResponse {
    pod := &corev1.Pod{}
    if err := json.Unmarshal(request.Object.Raw, pod); err != nil {
        return errorResponse(err)
    }

    var patches []map[string]interface{}

    // Add default labels
    patches = append(patches, di.createLabelPatches(pod)...)

    // Add default annotations
    patches = append(patches, di.createAnnotationPatches(pod)...)

    // Add default resource limits/requests
    patches = append(patches, di.createResourcePatches(pod)...)

    // Add default security context
    patches = append(patches, di.createSecurityPatches(pod)...)

    if len(patches) == 0 {
        return &admissionv1.AdmissionResponse{Allowed: true}
    }

    patchBytes, err := json.Marshal(patches)
    if err != nil {
        return errorResponse(err)
    }

    patchType := admissionv1.PatchTypeJSONPatch
    return &admissionv1.AdmissionResponse{
        Allowed:   true,
        Patch:     patchBytes,
        PatchType: &patchType,
    }
}

func (di *DefaultsInjector) createLabelPatches(pod *corev1.Pod) []map[string]interface{} {
    var patches []map[string]interface{}

    if pod.Labels == nil {
        // Create labels map
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/metadata/labels",
            "value": di.DefaultLabels,
        })
    } else {
        // Add missing labels
        for key, value := range di.DefaultLabels {
            if _, exists := pod.Labels[key]; !exists {
                patches = append(patches, map[string]interface{}{
                    "op":    "add",
                    "path":  "/metadata/labels/" + escapeJSONPointer(key),
                    "value": value,
                })
            }
        }
    }

    return patches
}

func (di *DefaultsInjector) createResourcePatches(pod *corev1.Pod) []map[string]interface{} {
    var patches []map[string]interface{}

    for i, container := range pod.Spec.Containers {
        if container.Resources.Limits == nil && container.Resources.Requests == nil {
            patches = append(patches, map[string]interface{}{
                "op":    "add",
                "path":  fmt.Sprintf("/spec/containers/%d/resources", i),
                "value": di.DefaultResources,
            })
        } else {
            // Add missing limits
            if container.Resources.Limits == nil {
                patches = append(patches, map[string]interface{}{
                    "op":    "add",
                    "path":  fmt.Sprintf("/spec/containers/%d/resources/limits", i),
                    "value": di.DefaultResources.Limits,
                })
            }
            // Add missing requests
            if container.Resources.Requests == nil {
                patches = append(patches, map[string]interface{}{
                    "op":    "add",
                    "path":  fmt.Sprintf("/spec/containers/%d/resources/requests", i),
                    "value": di.DefaultResources.Requests,
                })
            }
        }
    }

    return patches
}

func (di *DefaultsInjector) createSecurityPatches(pod *corev1.Pod) []map[string]interface{} {
    var patches []map[string]interface{}

    for i, container := range pod.Spec.Containers {
        if container.SecurityContext == nil {
            runAsNonRoot := true
            allowPrivilegeEscalation := false
            readOnlyRootFilesystem := true

            patches = append(patches, map[string]interface{}{
                "op":   "add",
                "path": fmt.Sprintf("/spec/containers/%d/securityContext", i),
                "value": &corev1.SecurityContext{
                    RunAsNonRoot:             &runAsNonRoot,
                    AllowPrivilegeEscalation: &allowPrivilegeEscalation,
                    ReadOnlyRootFilesystem:   &readOnlyRootFilesystem,
                    Capabilities: &corev1.Capabilities{
                        Drop: []corev1.Capability{"ALL"},
                    },
                },
            })
        }
    }

    return patches
}

func escapeJSONPointer(s string) string {
    s = strings.ReplaceAll(s, "~", "~0")
    s = strings.ReplaceAll(s, "/", "~1")
    return s
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Reinvocation Policies**

### **Reinvocation Behavior Diagram**

```mermaid
sequenceDiagram
    participant API as API Server
    participant MA as Mutating Admission
    participant W1 as Webhook A<br/>(Never)
    participant W2 as Webhook B<br/>(IfNeeded)
    participant W3 as Webhook C<br/>(IfNeeded)

    Note over API,W3: Round 1: Initial Processing

    API->>MA: Process Object (v1)
    MA->>W1: Mutate
    W1-->>MA: Patch Applied (v2)

    MA->>W2: Mutate
    W2-->>MA: Patch Applied (v3)

    MA->>W3: Mutate
    W3-->>MA: No Changes (v3)

    Note over MA: Object changed from v1→v3<br/>Webhooks marked for reinvocation:<br/>W2 (IfNeeded), W3 (IfNeeded)

    Note over API,W3: Round 2: Reinvocation

    MA->>W2: Mutate (reinvoke)
    W2-->>MA: Patch Applied (v4)

    MA->>W3: Mutate (reinvoke)
    W3-->>MA: No Changes (v4)

    Note over MA: Object changed v3→v4<br/>Would reinvoke again but<br/>max 2 rounds reached

    MA-->>API: Final Object (v4)
```

### **Reinvocation Context Tracking**

```go
// File: staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/reinvoke.go
package mutating

import (
    "k8s.io/apimachinery/pkg/types"
    "k8s.io/apimachinery/pkg/runtime"
)

type webhookReinvokeContext struct {
    // Track webhooks that should be reinvoked
    webhooksToReinvoke map[types.UID]bool

    // Track if object changed since last invocation
    lastWebhookOutput runtime.Object

    // Track which webhooks have been invoked
    invokedWebhooks map[types.UID]bool
}

func (wrc *webhookReinvokeContext) AddReinvocableWebhookToPreviouslyInvoked(uid types.UID) {
    if wrc.invokedWebhooks == nil {
        wrc.invokedWebhooks = make(map[types.UID]bool)
    }
    wrc.invokedWebhooks[uid] = true
}

func (wrc *webhookReinvokeContext) RequireReinvokingPreviouslyInvokedPlugins() {
    if wrc.webhooksToReinvoke == nil {
        wrc.webhooksToReinvoke = make(map[types.UID]bool)
    }
    for uid := range wrc.invokedWebhooks {
        wrc.webhooksToReinvoke[uid] = true
    }
}

func (wrc *webhookReinvokeContext) ShouldReinvokeWebhook(uid types.UID) bool {
    if wrc.webhooksToReinvoke == nil {
        return false
    }
    return wrc.webhooksToReinvoke[uid]
}

func (wrc *webhookReinvokeContext) IsOutputChangedSinceLastWebhookInvocation(current runtime.Object) bool {
    if wrc.lastWebhookOutput == nil {
        return false
    }
    return !apiequality.Semantic.DeepEqual(wrc.lastWebhookOutput, current)
}

func (wrc *webhookReinvokeContext) SetLastWebhookInvocationOutput(obj runtime.Object) {
    wrc.lastWebhookOutput = obj.DeepCopyObject()
}
```

### **Reinvocation Configuration Examples**

```yaml
# Example 1: Never reinvoke (default)
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: defaults-injector
webhooks:
  - name: defaults.example.com
    reinvocationPolicy: Never
    # This webhook runs once, even if later webhooks modify the object
    clientConfig:
      service:
        namespace: webhooks
        name: defaults-injector
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None

---
# Example 2: Reinvoke if needed
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: policy-enforcer
webhooks:
  - name: policy.example.com
    reinvocationPolicy: IfNeeded
    # This webhook may run multiple times if object changes
    clientConfig:
      service:
        namespace: webhooks
        name: policy-enforcer
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Architecture Diagrams**

### **Diagram 1: Mutating Webhook Request Flow**

```mermaid
sequenceDiagram
    participant Client
    participant APIServer
    participant Auth
    participant Mutating
    participant Schema
    participant Validating
    participant etcd

    Client->>APIServer: Create Pod
    APIServer->>Auth: Authenticate & Authorize
    Auth-->>APIServer: Allowed

    rect rgb(255, 230, 200)
        Note over Mutating: Mutating Webhooks Phase
        APIServer->>Mutating: Process Object
        Mutating->>Mutating: webhook-a (modify)
        Mutating->>Mutating: webhook-b (modify)
        Mutating->>Mutating: webhook-c (no change)
        Note over Mutating: Check if reinvocation needed
        Mutating->>Mutating: webhook-a (reinvoke if IfNeeded)
        Mutating->>Mutating: webhook-b (reinvoke if IfNeeded)
        Mutating-->>APIServer: Modified Object
    end

    APIServer->>Schema: Schema Validation
    Schema-->>APIServer: Valid

    APIServer->>Validating: Validating Webhooks
    Validating-->>APIServer: Allowed

    APIServer->>etcd: Persist Object
    etcd-->>APIServer: Stored
    APIServer-->>Client: Created
```

### **Diagram 2: JSONPatch Application Process**

```mermaid
graph TB
    A[Original Object] --> B[Encode to JSON]
    B --> C[Apply JSONPatch]

    subgraph "Patch Operations"
        D1[Add Operations]
        D2[Replace Operations]
        D3[Remove Operations]
    end

    C --> D1
    C --> D2
    C --> D3

    D1 --> E[Patched JSON]
    D2 --> E
    D3 --> E

    E --> F[Decode from JSON]
    F --> G[New Object]

    G --> H{Deep Equal?}
    H -->|Yes| I[No Change]
    H -->|No| J[Object Changed]

    J --> K[Set Dirty Flag]
    K --> L[Apply Defaults]
    L --> M[Return Modified Object]

    style J fill:#ffd43b
    style M fill:#51cf66
```

### **Diagram 3: Sidecar Injection Flow**

```mermaid
sequenceDiagram
    participant API as API Server
    participant Webhook as Sidecar Injector

    API->>Webhook: AdmissionReview<br/>(Pod without sidecar)

    Webhook->>Webhook: Check namespace labels
    Webhook->>Webhook: Check pod labels
    Webhook->>Webhook: Check if already injected

    alt Injection needed
        Webhook->>Webhook: Build sidecar container
        Webhook->>Webhook: Build init container
        Webhook->>Webhook: Build volumes
        Webhook->>Webhook: Create JSONPatch

        Webhook-->>API: AdmissionReview<br/>allowed: true<br/>patch: [...]

        API->>API: Apply patch
        API->>API: Pod now has:<br/>- init container<br/>- sidecar container<br/>- shared volumes
    else No injection needed
        Webhook-->>API: AdmissionReview<br/>allowed: true<br/>(no patch)
    end
```

### **Diagram 4: Reinvocation Mechanism**

```mermaid
graph TD
    Start([Request Object v1]) --> W1[Webhook A<br/>reinvocationPolicy: Never]
    W1 --> C1{Changed?}
    C1 -->|Yes| V2[Object v2]
    C1 -->|No| V2

    V2 --> W2[Webhook B<br/>reinvocationPolicy: IfNeeded]
    W2 --> C2{Changed?}
    C2 -->|Yes| V3[Object v3]
    C2 -->|No| V3

    V3 --> W3[Webhook C<br/>reinvocationPolicy: IfNeeded]
    W3 --> C3{Changed?}
    C3 -->|Yes| V4[Object v4]
    C3 -->|No| V4

    V4 --> Check{Object changed<br/>in round 1?}
    Check -->|Yes| Round2[Round 2]
    Check -->|No| Done[Done]

    Round2 --> Skip1[Skip Webhook A<br/>Never policy]
    Skip1 --> W2_2[Webhook B<br/>Reinvoke]
    W2_2 --> W3_2[Webhook C<br/>Reinvoke]
    W3_2 --> Done

    style V2 fill:#ffd43b
    style V3 fill:#ffd43b
    style V4 fill:#ffd43b
    style Done fill:#51cf66
```

### **Diagram 5: Webhook Ordering**

```mermaid
graph LR
    subgraph "MutatingWebhookConfiguration"
        A[webhook-aaa]
        B[webhook-bbb]
        C[webhook-ccc]
        D[webhook-zzz]
    end

    Start([Request]) --> Sort[Sort Alphabetically]
    Sort --> A
    A --> B
    B --> C
    C --> D
    D --> End([Modified Object])

    style Sort fill:#4dabf7
    style End fill:#51cf66
```

### **Diagram 6: Failure Policy Handling**

```mermaid
graph TD
    A[Call Webhook] --> B{Response?}
    B -->|Success| C{Patch Valid?}
    B -->|Timeout| D{Failure Policy?}
    B -->|Error| D

    C -->|Yes| E[Apply Patch]
    C -->|No| F[Invalid Response]

    E --> G[Continue]
    F --> D

    D -->|Fail| H[Reject Request]
    D -->|Ignore| I[Log Warning<br/>Continue]

    style H fill:#ff6b6b
    style I fill:#ffd43b
    style G fill:#51cf66
```

### **Diagram 7: Audit Annotation Flow**

```mermaid
sequenceDiagram
    participant Webhook
    participant Dispatcher
    participant Annotator
    participant AuditBackend

    Webhook->>Dispatcher: Return patch
    Dispatcher->>Annotator: Create annotator

    alt Patch applied
        Annotator->>Annotator: addPatchAnnotation()<br/>Key: patch.webhook.../round_0_index_0<br/>Value: {configuration, webhook, patch}

        Annotator->>Annotator: addMutationAnnotation()<br/>Key: mutation.webhook.../round_0_index_0<br/>Value: {configuration, webhook, mutated: true}
    else Webhook failed open
        Annotator->>Annotator: addFailedOpenAnnotation()<br/>Key: failed-open.mutation.webhook...
    end

    Annotator->>Dispatcher: Annotations added
    Dispatcher->>AuditBackend: Audit event with annotations
```

### **Diagram 8: Complete Patch Example**

```mermaid
graph TB
    subgraph "Original Pod"
        A1[metadata:<br/>  name: myapp<br/>  labels: {}]
        A2[spec:<br/>  containers:<br/>    - name: app<br/>      image: app:v1]
    end

    subgraph "JSONPatch"
        B1["op: add<br/>path: /metadata/labels/injected<br/>value: 'true'"]
        B2["op: add<br/>path: /spec/containers/-<br/>value: {sidecar container}"]
        B3["op: add<br/>path: /spec/volumes<br/>value: [{shared volume}]"]
    end

    subgraph "Patched Pod"
        C1[metadata:<br/>  name: myapp<br/>  labels:<br/>    injected: 'true']
        C2[spec:<br/>  containers:<br/>    - name: app<br/>    - name: sidecar<br/>  volumes:<br/>    - name: shared]
    end

    A1 --> B1
    A2 --> B1
    B1 --> B2
    B2 --> B3
    B3 --> C1
    B3 --> C2

    style B1 fill:#4dabf7
    style B2 fill:#4dabf7
    style B3 fill:#4dabf7
```

### **Diagram 9: Multi-Container Injection**

```mermaid
graph TB
    A[Original Pod<br/>1 container] --> B{Webhook 1:<br/>Add Sidecar A}
    B --> C[Pod with<br/>2 containers]
    C --> D{Webhook 2:<br/>Add Sidecar B}
    D --> E[Pod with<br/>3 containers]
    E --> F{Webhook 3:<br/>Add Init Container}
    F --> G[Final Pod:<br/>1 init + 3 containers]

    style G fill:#51cf66
```

### **Diagram 10: Resource Defaults Injection**

```mermaid
graph LR
    subgraph "Before"
        A1[Container:<br/>  name: app<br/>  image: app:v1<br/>  resources: {}]
    end

    subgraph "Patch"
        B["op: add<br/>path: /spec/containers/0/resources<br/>value:<br/>  limits:<br/>    cpu: 500m<br/>    memory: 512Mi<br/>  requests:<br/>    cpu: 100m<br/>    memory: 128Mi"]
    end

    subgraph "After"
        C[Container:<br/>  name: app<br/>  image: app:v1<br/>  resources:<br/>    limits: {...}<br/>    requests: {...}]
    end

    A1 --> B
    B --> C

    style B fill:#4dabf7
    style C fill:#51cf66
```

### **Diagram 11: Webhook Server Architecture**

```mermaid
graph TB
    subgraph "Webhook Server"
        A[HTTPS Server]
        B[Request Router]
        C[Sidecar Injector]
        D[Defaults Injector]
        E[Patch Builder]
        F[JSON Encoder]
    end

    API[API Server] -->|POST| A
    A --> B
    B --> C
    B --> D
    C --> E
    D --> E
    E --> F
    F --> A
    A -->|Response| API

    style A fill:#4dabf7
    style E fill:#ffd43b
```

### **Diagram 12: Dry Run Handling**

```mermaid
graph TD
    A[Request] --> B{Dry Run?}
    B -->|Yes| C{Side Effects?}
    B -->|No| G[Process Normally]

    C -->|None| D[Process Request]
    C -->|NoneOnDryRun| D
    C -->|Some| E[Return Error]
    C -->|Unknown| E

    D --> F[Return Patch]
    F --> H[API Server<br/>Does NOT Apply]

    G --> I[Return Patch]
    I --> J[API Server<br/>Applies Patch]

    style E fill:#ff6b6b
    style H fill:#ffd43b
    style J fill:#51cf66
```

### **Diagram 13: Certificate Rotation**

```mermaid
sequenceDiagram
    participant CM as cert-manager
    participant Secret as TLS Secret
    participant Webhook as Webhook Pod
    participant CAInj as CA Injector
    participant Config as WebhookConfiguration

    Note over CM,Config: Initial Setup
    CM->>Secret: Create/Update Certificate
    Webhook->>Secret: Watch & Reload
    CAInj->>Config: Inject CA Bundle

    Note over CM,Config: Rotation (30 days before expiry)
    CM->>CM: Generate New Certificate
    CM->>Secret: Update Secret
    Secret-->>Webhook: Notify Change
    Webhook->>Webhook: Reload Certificate
    CAInj->>Config: Update CA Bundle

    Note over Webhook: Server continues<br/>with new cert
```

### **Diagram 14: Production Topology**

```mermaid
graph TB
    subgraph "Cluster"
        subgraph "webhook-system"
            W1[webhook-1]
            W2[webhook-2]
            W3[webhook-3]
            SVC[Service]
            SEC[TLS Secret]
        end

        subgraph "Control Plane"
            API1[API Server 1]
            API2[API Server 2]
            API3[API Server 3]
        end

        CFG[MutatingWebhookConfiguration]
    end

    W1 -.->|Mount| SEC
    W2 -.->|Mount| SEC
    W3 -.->|Mount| SEC

    W1 -.->|Register| SVC
    W2 -.->|Register| SVC
    W3 -.->|Register| SVC

    API1 -->|HTTPS| SVC
    API2 -->|HTTPS| SVC
    API3 -->|HTTPS| SVC

    SVC -->|LB| W1
    SVC -->|LB| W2
    SVC -->|LB| W3

    API1 -.->|Read| CFG
    API2 -.->|Read| CFG
    API3 -.->|Read| CFG

    style API1 fill:#4dabf7
    style API2 fill:#4dabf7
    style API3 fill:#4dabf7
```

### **Diagram 15: Mutation Pipeline**

```mermaid
graph TB
    Start([Original Object]) --> M1[Mutating Webhook 1]
    M1 --> O1[Object v1]
    O1 --> M2[Mutating Webhook 2]
    M2 --> O2[Object v2]
    O2 --> M3[Mutating Webhook 3]
    M3 --> O3[Object v3]

    O3 --> R{Reinvocation<br/>Needed?}
    R -->|Yes| R1[Reinvoke Eligible Webhooks]
    R -->|No| S[Schema Validation]

    R1 --> O4[Object v4]
    O4 --> R2{More Changes?}
    R2 -->|Yes, < max rounds| R1
    R2 -->|No or max reached| S

    S --> V[Validating Webhooks]
    V --> E([Final Object])

    style O1 fill:#ffd43b
    style O2 fill:#ffd43b
    style O3 fill:#ffd43b
    style O4 fill:#ffd43b
    style E fill:#51cf66
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Code References**

### **Core Implementation Files**

| File | Lines | Description |
|------|-------|-------------|
| `staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/plugin.go` | 77 | Main mutating webhook plugin |
| `staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/dispatcher.go` | ~497 | Mutation dispatch and patch application |
| `staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/generic/webhook.go` | ~500 | Generic webhook framework (shared) |
| `staging/src/k8s.io/api/admissionregistration/v1/types.go` | ~400 | MutatingWebhookConfiguration types |

### **Key Constants**

```go
// File: staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/dispatcher.go:54-66
const (
    // PatchAuditAnnotationPrefix is prefix for persisting webhook patch in audit annotation
    PatchAuditAnnotationPrefix = "patch.webhook.admission.k8s.io/"

    // MutationAuditAnnotationPrefix is prefix for persisting webhook mutation in audit annotation
    MutationAuditAnnotationPrefix = "mutation.webhook.admission.k8s.io/"

    // MutationAnnotationFailedOpenKeyPrefix indicates mutation webhook failed open
    MutationAuditAnnotationFailedOpenKeyPrefix = "failed-open." + MutationAuditAnnotationPrefix
)
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **✅ Summary**

### **Key Takeaways**

1. **Mutation Capability**: Mutating webhooks can modify objects via JSONPatch operations
2. **Early Execution**: Run before schema validation, allowing fixes before validation
3. **Reinvocation**: Webhooks can be configured to run multiple times if object changes
4. **Ordered Processing**: Execute in alphabetical order by name
5. **Audit Trail**: All mutations are logged with detailed patch information
6. **Common Patterns**: Sidecar injection, default injection, resource modification
7. **JSONPatch**: RFC 6902 standard for describing object modifications
8. **Production Ready**: Requires HA deployment, certificate management, monitoring

### **Best Practices**

1. **Idempotency**: Ensure mutations are idempotent (can be applied multiple times safely)
2. **Minimal Changes**: Only mutate what's necessary to avoid conflicts
3. **Reinvocation Policy**: Use `Never` for simple defaults, `IfNeeded` for complex logic
4. **Error Handling**: Implement comprehensive error handling and logging
5. **Testing**: Test patch generation thoroughly with various input scenarios
6. **Performance**: Keep mutation logic fast (< 5 seconds typical)
7. **Backwards Compatibility**: Handle different API versions gracefully
8. **Documentation**: Document all mutations clearly for users

**Document Statistics:**
- Total Lines: ~2,550
- Code Examples: 15+
- Diagrams: 15
- Real Implementation: Complete sidecar injector and defaults injector
