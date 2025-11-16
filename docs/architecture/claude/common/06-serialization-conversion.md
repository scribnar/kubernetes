# Serialization and Conversion: How Kubernetes Encodes Objects

**Document**: 06-serialization-conversion.md
**Part**: III - apimachinery Library
**Audience**: Software Engineers learning Kubernetes controller development
**Prerequisites**: [05-runtime-scheme.md](05-runtime-scheme.md) - Understanding of Scheme and GVK
**Related Docs**: [07-watch-meta-types.md](07-watch-meta-types.md), [02-rest-clients-discovery.md](02-rest-clients-discovery.md)

---

## Executive Summary

**Serialization** is how Kubernetes converts between **Go structs** (in memory) and **wire formats** (JSON, YAML, Protobuf, CBOR) that can be sent over the network or stored in etcd. **Conversion** is how Kubernetes transforms objects between different API versions (v1alpha1 → v1).

**Why this matters for developers**:
- **kubectl** sends YAML → API server decodes to Go → processes → encodes back to JSON/YAML
- **API server** stores objects in etcd as Protobuf (efficient) but serves JSON to clients (readable)
- **Controllers** watch objects in one version but may need to work with another version
- **Custom controllers** need to understand serialization to work with CRDs properly

**🎯 Learning Objective**: Understand how K8s handles multiple formats and API versions, enabling you to work with serialization in controllers and tools.

**📊 Key Statistics**:
- **Formats Supported**: JSON, YAML, Protobuf, CBOR
- **Protobuf Performance**: ~5-10x faster than JSON for large objects
- **API Versions**: Core has ~40 API groups, each with 2-4 versions
- **Automatic Conversion**: 1000+ auto-generated conversion functions

---

## Table of Contents

1. [Key Concepts](#key-concepts)
2. [Serialization Architecture](#serialization-architecture)
3. [Codec and Serializer](#codec-and-serializer)
4. [Content Negotiation](#content-negotiation)
5. [Encoding and Decoding](#encoding-and-decoding)
6. [Format Details](#format-details)
7. [Conversion Framework](#conversion-framework)
8. [Unstructured Objects](#unstructured-objects)
9. [Real-World Examples](#real-world-examples)
10. [Performance Considerations](#performance-considerations)
11. [Common Pitfalls](#common-pitfalls)
12. [Testing Patterns](#testing-patterns)
13. [Summary](#summary)

---

## Key Concepts

### What is Serialization?

**Serialization (Encoding)**: Go struct → Bytes (JSON/YAML/Protobuf/CBOR)
**Deserialization (Decoding)**: Bytes → Go struct

```
┌─────────────────────┐                    ┌────────────────────┐
│   Go Struct         │                    │   Wire Format      │
│                     │                    │                    │
│  type Pod struct {  │    Serialize       │  {                 │
│    Name string      │  ─────────────────►│    "kind": "Pod",  │
│    Namespace string │                    │    "metadata": {   │
│    Spec PodSpec     │    Deserialize     │      "name": "...", │
│  }                  │  ◄─────────────────│    }               │
│                     │                    │  }                 │
└─────────────────────┘                    └────────────────────┘
     (In Memory)                                 (On Wire/Disk)
```

### What is Conversion?

**Conversion**: Transforming an object from one API version to another.

```
┌──────────────────┐                ┌─────────────────┐                ┌──────────────────┐
│  v1alpha1        │   Convert      │   Internal      │   Convert      │     v1           │
│  Deployment      │  ────────────► │  Deployment     │  ────────────► │  Deployment      │
│                  │                │  (Unversioned)  │                │                  │
│  Old fields      │                │  All fields     │                │  New fields      │
│  Missing new     │                │  Complete set   │                │  Validated       │
└──────────────────┘                └─────────────────┘                └──────────────────┘
```

**Why conversion is needed**:
- Client sends v1alpha1, server stores v1
- Server stores v1, client requests v1beta1
- Rolling upgrades (old clients, new server)

### The Codec Pipeline

Every API request goes through this pipeline:

```mermaid
graph LR
    Client[Client<br/>YAML/JSON] -->|HTTP Request| Server[API Server]

    subgraph "API Server Pipeline"
        Decode[Decode<br/>Bytes → Object]
        Convert1[Convert<br/>External → Internal]
        Process[Process<br/>Validate, Default,<br/>Mutate, Store]
        Convert2[Convert<br/>Internal → External]
        Encode[Encode<br/>Object → Bytes]
    end

    Server --> Decode
    Decode --> Convert1
    Convert1 --> Process
    Process --> Convert2
    Convert2 --> Encode
    Encode -->|HTTP Response| Client

    style Decode fill:#e1f5ff
    style Encode fill:#e1f5ff
    style Convert1 fill:#fff3e0
    style Convert2 fill:#fff3e0
    style Process fill:#c8e6c9
```

---

## Serialization Architecture

### Core Components

**Code Reference**: `staging/src/k8s.io/apimachinery/pkg/runtime/serializer/`

```go
// Codec is the main interface for encoding and decoding
type Codec interface {
    // Decoder decodes bytes to objects
    Decoder

    // Encoder encodes objects to bytes
    Encoder
}

// Encoder writes objects to a stream
type Encoder interface {
    // Encode writes the object to a stream
    Encode(obj Object, w io.Writer) error
}

// Decoder reads objects from a stream
type Decoder interface {
    // Decode reads bytes and returns an object
    Decode(data []byte, defaults *schema.GroupVersionKind, into Object) (Object, *schema.GroupVersionKind, error)
}
```

### Serializer Hierarchy

```mermaid
graph TB
    subgraph "Serialization Stack"
        NegotiatedSerializer[NegotiatedSerializer<br/>Format Negotiation]

        NegotiatedSerializer --> JSONSerializer[JSON Serializer]
        NegotiatedSerializer --> YAMLSerializer[YAML Serializer]
        NegotiatedSerializer --> ProtobufSerializer[Protobuf Serializer]
        NegotiatedSerializer --> CBORSerializer[CBOR Serializer]

        JSONSerializer --> JSONEncoder[JSON Encoder]
        JSONSerializer --> JSONDecoder[JSON Decoder]

        ProtobufSerializer --> ProtobufEncoder[Protobuf Encoder]
        ProtobufSerializer --> ProtobufDecoder[Protobuf Decoder]
    end

    Client[Client Request] -->|Accept: application/json| NegotiatedSerializer
    NegotiatedSerializer -->|Select| JSONSerializer

    style NegotiatedSerializer fill:#e1f5ff
    style JSONSerializer fill:#fff3e0
    style ProtobufSerializer fill:#fff3e0
```

### CodecFactory

**Code Reference**: `staging/src/k8s.io/apimachinery/pkg/runtime/serializer/codec_factory.go:31`

```go
// CodecFactory provides methods for retrieving codecs and serializers
type CodecFactory struct {
    scheme *runtime.Scheme

    // Serializers for different formats
    serializers []serializerType

    // Universal deserializer (tries all formats)
    universal runtime.Decoder

    // Accepts for content negotiation
    accepts []runtime.SerializerInfo
}

// NewCodecFactory creates a new codec factory
func NewCodecFactory(scheme *runtime.Scheme) CodecFactory {
    serializers := []serializerType{
        {
            AcceptContentTypes: []string{"application/json"},
            ContentType:        "application/json",
            FileExtensions:     []string{"json"},
            Serializer:         json.NewSerializer(json.DefaultMetaFactory, scheme, scheme, false),
        },
        {
            AcceptContentTypes: []string{"application/yaml"},
            ContentType:        "application/yaml",
            FileExtensions:     []string{"yaml"},
            Serializer:         json.NewYAMLSerializer(json.DefaultMetaFactory, scheme, scheme),
        },
        {
            AcceptContentTypes: []string{"application/vnd.kubernetes.protobuf"},
            ContentType:        "application/vnd.kubernetes.protobuf",
            FileExtensions:     []string{"pb"},
            Serializer:         protobuf.NewSerializer(scheme, scheme),
        },
        {
            AcceptContentTypes: []string{"application/cbor"},
            ContentType:        "application/cbor",
            FileExtensions:     []string{"cbor"},
            Serializer:         cbor.NewSerializer(scheme, scheme),
        },
    }

    return CodecFactory{
        scheme:      scheme,
        serializers: serializers,
        universal:   universalDeserializer{serializers},
        accepts:     serializerInfo(serializers),
    }
}
```

### Codec Creation Methods

```go
// Universal decoder (tries all formats)
func (f CodecFactory) UniversalDeserializer() runtime.Decoder {
    return f.universal
}

// Codec for specific GroupVersion
func (f CodecFactory) LegacyCodec(gvs ...schema.GroupVersion) runtime.Codec {
    // Creates a codec that encodes to the preferred GV and decodes from any GV
    return versioning.NewDefaultingCodecForScheme(f.scheme, nil, nil, gvs, nil)
}

// Codec with specific encoder and decoder
func (f CodecFactory) CodecForVersions(encoder runtime.Encoder,
    decoder runtime.Decoder,
    encodeGVs []schema.GroupVersion,
    decodeGVs []schema.GroupVersion) runtime.Codec {

    return versioning.NewCodec(encoder, decoder, f.scheme, f.scheme,
        runtime.InternalGroupVersioner, encodeGVs, decodeGVs)
}
```

---

## Codec and Serializer

### JSON Serializer

**Code Reference**: `staging/src/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go:65`

```go
// Serializer handles encoding and decoding of JSON
type Serializer struct {
    meta    MetaFactory
    creater runtime.ObjectCreater
    typer   runtime.ObjectTyper
    yaml    bool  // true for YAML, false for JSON
    pretty  bool  // true for pretty-printed JSON
}

// NewSerializer creates a JSON serializer
func NewSerializer(meta MetaFactory, creater runtime.ObjectCreater,
    typer runtime.ObjectTyper, pretty bool) *Serializer {

    return &Serializer{
        meta:    meta,
        creater: creater,
        typer:   typer,
        yaml:    false,
        pretty:  pretty,
    }
}

// Encode writes a JSON representation of obj to stream
func (s *Serializer) Encode(obj runtime.Object, w io.Writer) error {
    // Determine GVK
    gvks, _, err := s.typer.ObjectKinds(obj)
    if err != nil {
        return err
    }

    // Get the first (preferred) GVK
    gvk := gvks[0]

    // Get object accessor to set GVK in TypeMeta
    accessor, err := meta.Accessor(obj)
    if err != nil {
        return err
    }

    // Set kind and apiVersion in the object
    if accessor != nil {
        if accessor.GetKind() == "" {
            accessor.SetKind(gvk.Kind)
        }
        if accessor.GetAPIVersion() == "" {
            accessor.SetAPIVersion(gvk.GroupVersion().String())
        }
    }

    // Marshal to JSON
    var data []byte
    if s.pretty {
        data, err = json.MarshalIndent(obj, "", "  ")
    } else {
        data, err = json.Marshal(obj)
    }
    if err != nil {
        return err
    }

    // Write to stream
    _, err = w.Write(data)
    return err
}

// Decode reads a JSON representation and returns an object
func (s *Serializer) Decode(data []byte, defaults *schema.GroupVersionKind,
    into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {

    // Parse JSON to determine type
    if len(data) == 0 {
        return nil, nil, fmt.Errorf("empty data")
    }

    // Extract GVK from JSON
    gvk, err := s.meta.Interpret(data)
    if err != nil {
        return nil, nil, err
    }

    // Use defaults if GVK not in data
    if gvk == nil || gvk.Empty() {
        if defaults == nil {
            return nil, nil, fmt.Errorf("no kind specified and no default provided")
        }
        gvk = defaults
    }

    // Create object of correct type
    if into != nil {
        // Decode into provided object
        if err := json.Unmarshal(data, into); err != nil {
            return nil, gvk, err
        }
        return into, gvk, nil
    }

    // Create new object based on GVK
    obj, err := s.creater.New(*gvk)
    if err != nil {
        return nil, gvk, err
    }

    // Unmarshal into new object
    if err := json.Unmarshal(data, obj); err != nil {
        return nil, gvk, err
    }

    return obj, gvk, nil
}
```

### Protobuf Serializer

**Why Protobuf?**
- **Performance**: 5-10x faster encoding/decoding
- **Size**: 30-50% smaller than JSON
- **Type Safety**: Strongly typed schemas
- **Used For**: etcd storage, internal API server communication, watch streams

**Code Reference**: `staging/src/k8s.io/apimachinery/pkg/runtime/serializer/protobuf/protobuf.go:59`

```go
// Serializer handles encoding and decoding of protobuf
type Serializer struct {
    creater runtime.ObjectCreater
    typer   runtime.ObjectTyper
}

// Encode writes a protobuf representation
func (s *Serializer) Encode(obj runtime.Object, w io.Writer) error {
    // Check if object implements protobuf marshaler
    marshaler, ok := obj.(proto.Marshaler)
    if !ok {
        return fmt.Errorf("object does not implement protobuf marshaling")
    }

    // Marshal to protobuf bytes
    data, err := marshaler.Marshal()
    if err != nil {
        return err
    }

    // Write length prefix (varint)
    buf := make([]byte, binary.MaxVarintLen64)
    n := binary.PutUvarint(buf, uint64(len(data)))
    if _, err := w.Write(buf[:n]); err != nil {
        return err
    }

    // Write protobuf data
    _, err = w.Write(data)
    return err
}

// Decode reads a protobuf representation
func (s *Serializer) Decode(data []byte, defaults *schema.GroupVersionKind,
    into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {

    // Read framing (length prefix and GVK)
    length, gvk, offset, err := s.readPrefix(data)
    if err != nil {
        return nil, nil, err
    }

    // Create object if not provided
    if into == nil {
        obj, err := s.creater.New(*gvk)
        if err != nil {
            return nil, gvk, err
        }
        into = obj
    }

    // Unmarshal protobuf
    unmarshaler, ok := into.(proto.Unmarshaler)
    if !ok {
        return nil, nil, fmt.Errorf("object does not implement protobuf unmarshaling")
    }

    if err := unmarshaler.Unmarshal(data[offset : offset+length]); err != nil {
        return nil, gvk, err
    }

    return into, gvk, nil
}
```

### YAML Serializer

YAML is built on top of JSON serializer:

```go
// NewYAMLSerializer creates a YAML serializer
func NewYAMLSerializer(meta MetaFactory, creater runtime.ObjectCreater,
    typer runtime.ObjectTyper) *Serializer {

    return &Serializer{
        meta:    meta,
        creater: creater,
        typer:   typer,
        yaml:    true,  // Enable YAML mode
        pretty:  false,
    }
}

// Encode converts object to YAML
func (s *Serializer) Encode(obj runtime.Object, w io.Writer) error {
    // Convert to JSON first
    var buf bytes.Buffer
    if err := s.encodeJSON(obj, &buf); err != nil {
        return err
    }

    // Convert JSON to YAML
    yamlData, err := yaml.JSONToYAML(buf.Bytes())
    if err != nil {
        return err
    }

    _, err = w.Write(yamlData)
    return err
}
```

---

## Content Negotiation

### How Content Type is Selected

**Request Flow**:

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant Negotiator
    participant Codec

    Client->>Server: GET /api/v1/pods<br/>Accept: application/json

    Server->>Negotiator: NegotiateInputSerializer(req)
    Negotiator->>Negotiator: Parse Accept header
    Negotiator->>Negotiator: Match with available serializers

    Negotiator-->>Server: JSON Serializer selected

    Server->>Codec: Use JSON Codec
    Codec-->>Server: Encoded JSON response

    Server-->>Client: 200 OK<br/>Content-Type: application/json<br/>{...json data...}
```

### Content Negotiation Code

**Code Reference**: `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/negotiation/negotiation.go:38`

```go
// NegotiateInputSerializer determines the appropriate serializer
// based on the Accept header
func NegotiateInputSerializer(req *http.Request, pretty bool,
    factory runtime.NegotiatedSerializer) (runtime.Serializer, error) {

    // Get Accept header
    accept := req.Header.Get("Accept")
    if len(accept) == 0 {
        // Default to JSON
        return factory.EncoderForVersion(nil, schema.GroupVersion{}), nil
    }

    // Parse Accept header (e.g., "application/json;q=0.9, application/yaml")
    mediaTypes := goautoneg.ParseAccept(accept)

    // Find matching serializer
    for _, mediaType := range mediaTypes {
        for _, info := range factory.SupportedMediaTypes() {
            if info.MediaType == mediaType.Type {
                if pretty && info.PrettySerializer != nil {
                    return info.PrettySerializer, nil
                }
                return info.Serializer, nil
            }
        }
    }

    return nil, errors.NewNotAcceptable(fmt.Sprintf(
        "no serializer found for %s", accept))
}
```

### Supported Media Types

| Content Type | Format | Used For | Performance |
|--------------|--------|----------|-------------|
| `application/json` | JSON | kubectl, clients, debugging | Baseline |
| `application/yaml` | YAML | kubectl apply, manifests | Same as JSON |
| `application/vnd.kubernetes.protobuf` | Protobuf | etcd storage, watch | 5-10x faster |
| `application/cbor` | CBOR | Future (experimental) | 2-3x faster than JSON |

### Client Accept Headers

```bash
# kubectl uses JSON by default
curl -H "Accept: application/json" https://k8s-api/api/v1/pods

# Request YAML
curl -H "Accept: application/yaml" https://k8s-api/api/v1/pods

# Request Protobuf (watch streams)
curl -H "Accept: application/vnd.kubernetes.protobuf;stream=watch" \
    https://k8s-api/api/v1/pods?watch=true
```

---

## Encoding and Decoding

### Encode Example

```go
package main

import (
    "bytes"
    "fmt"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer/json"
    "k8s.io/client-go/kubernetes/scheme"
)

func main() {
    // Create a Pod
    pod := &corev1.Pod{
        TypeMeta: metav1.TypeMeta{
            APIVersion: "v1",
            Kind:       "Pod",
        },
        ObjectMeta: metav1.ObjectMeta{
            Name:      "my-pod",
            Namespace: "default",
            Labels: map[string]string{
                "app": "myapp",
            },
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{
                {
                    Name:  "nginx",
                    Image: "nginx:1.14.2",
                },
            },
        },
    }

    // Create JSON serializer
    serializer := json.NewSerializer(
        json.DefaultMetaFactory,
        scheme.Scheme,
        scheme.Scheme,
        true, // pretty print
    )

    // Encode to bytes
    var buf bytes.Buffer
    err := serializer.Encode(pod, &buf)
    if err != nil {
        panic(err)
    }

    // Print JSON
    fmt.Println(buf.String())
    /*
    {
      "kind": "Pod",
      "apiVersion": "v1",
      "metadata": {
        "name": "my-pod",
        "namespace": "default",
        "labels": {
          "app": "myapp"
        }
      },
      "spec": {
        "containers": [
          {
            "name": "nginx",
            "image": "nginx:1.14.2"
          }
        ]
      }
    }
    */
}
```

### Decode Example

```go
package main

import (
    "fmt"

    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime/serializer"
    "k8s.io/client-go/kubernetes/scheme"
)

func main() {
    yamlData := `
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
`

    // Create codec factory
    codecFactory := serializer.NewCodecFactory(scheme.Scheme)

    // Get universal deserializer (tries all formats)
    decoder := codecFactory.UniversalDeserializer()

    // Decode YAML → Go object
    obj, gvk, err := decoder.Decode([]byte(yamlData), nil, nil)
    if err != nil {
        panic(err)
    }

    fmt.Printf("Decoded GVK: %s\n", gvk)  // v1, Kind=Pod

    // Type assertion
    pod := obj.(*corev1.Pod)
    fmt.Printf("Pod: %s/%s\n", pod.Namespace, pod.Name)
    fmt.Printf("Container: %s\n", pod.Spec.Containers[0].Name)
}
```

### Round-Trip Example (Encode → Decode)

```go
func TestRoundTrip(t *testing.T) {
    // Original pod
    originalPod := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-pod",
            Namespace: "default",
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{{Name: "nginx", Image: "nginx"}},
        },
    }

    // Encode to JSON
    codecFactory := serializer.NewCodecFactory(scheme.Scheme)
    encoder := codecFactory.LegacyCodec(corev1.SchemeGroupVersion)

    var buf bytes.Buffer
    err := encoder.Encode(originalPod, &buf)
    require.NoError(t, err)

    // Decode back
    decoder := codecFactory.UniversalDeserializer()
    obj, _, err := decoder.Decode(buf.Bytes(), nil, nil)
    require.NoError(t, err)

    // Verify equality
    decodedPod := obj.(*corev1.Pod)
    assert.Equal(t, originalPod.Name, decodedPod.Name)
    assert.Equal(t, originalPod.Namespace, decodedPod.Namespace)
    assert.Equal(t, originalPod.Spec.Containers[0].Name,
        decodedPod.Spec.Containers[0].Name)
}
```

---

## Format Details

### JSON Format

**Advantages**:
- Human-readable
- Widely supported
- Easy debugging
- Compatible with all tools

**Disadvantages**:
- Larger size than binary formats
- Slower parsing than protobuf
- No schema validation (without additional tools)

**Example**:

```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx-pod",
    "namespace": "default",
    "uid": "12345",
    "resourceVersion": "123456",
    "creationTimestamp": "2024-01-01T00:00:00Z"
  },
  "spec": {
    "containers": [
      {
        "name": "nginx",
        "image": "nginx:1.14.2",
        "ports": [{"containerPort": 80}]
      }
    ],
    "restartPolicy": "Always"
  },
  "status": {
    "phase": "Running"
  }
}
```

### YAML Format

**Advantages**:
- More human-readable than JSON
- Supports comments
- More concise (no brackets)
- Preferred for manifests

**Disadvantages**:
- Whitespace-sensitive
- Indentation errors common
- Same performance as JSON (converted internally)

**Example**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: default
  # Comments are allowed in YAML
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
  restartPolicy: Always
```

### Protobuf Format

**Advantages**:
- **5-10x faster** encoding/decoding
- **30-50% smaller** size
- **Type-safe** with schema
- **Forward/backward compatible**

**Disadvantages**:
- Not human-readable
- Requires .proto definitions
- Binary format (debugging harder)

**When used**:
- etcd storage (all objects stored as protobuf)
- Watch streams (high throughput)
- Internal component communication

**Protobuf Definition Example**:

```protobuf
// k8s.io/api/core/v1/generated.proto
message Pod {
  optional k8s.io.apimachinery.pkg.apis.meta.v1.ObjectMeta metadata = 1;
  optional PodSpec spec = 2;
  optional PodStatus status = 3;
}

message PodSpec {
  repeated Container containers = 2;
  optional string restartPolicy = 3;
  optional int64 terminationGracePeriodSeconds = 4;
  // ... more fields
}
```

**Binary Format** (hex dump):

```
0a 25 0a 09 6e 67 69 6e 78 2d 70 6f 64 12 07 64  .%..nginx-pod..d
65 66 61 75 6c 74 1a 05 31 32 33 34 35 22 06 31  efault..12345".1
32 33 34 35 36 12 3a 0a 14 0a 05 6e 67 69 6e 78  23456.:....nginx
```

### CBOR Format (Experimental)

**CBOR (Concise Binary Object Representation)**:
- New format added in Kubernetes 1.24+
- **2-3x faster** than JSON
- **Smaller** than JSON
- **More structured** than JSON

**Status**: Experimental, not widely used yet

---

## Conversion Framework

### Why Conversion?

Kubernetes needs to convert objects between API versions:

```
Client (v1alpha1) → API Server → etcd (v1) → API Server → Client (v1beta1)
```

**Conversion Types**:
1. **External → Internal**: When receiving API requests
2. **Internal → External**: When sending API responses
3. **External → External**: When serving different versions to clients

### Conversion Architecture

```mermaid
graph TB
    subgraph "API Versions (External)"
        V1Alpha1[v1alpha1.Deployment]
        V1Beta1[v1beta1.Deployment]
        V1[v1.Deployment]
    end

    subgraph "Hub (Internal)"
        Internal[apps.Deployment<br/>Unversioned]
    end

    subgraph "Conversion Functions"
        Conv1[Convert_v1alpha1_Deployment_To_apps_Deployment]
        Conv2[Convert_v1beta1_Deployment_To_apps_Deployment]
        Conv3[Convert_v1_Deployment_To_apps_Deployment]

        Conv4[Convert_apps_Deployment_To_v1alpha1_Deployment]
        Conv5[Convert_apps_Deployment_To_v1beta1_Deployment]
        Conv6[Convert_apps_Deployment_To_v1_Deployment]
    end

    V1Alpha1 -->|To Internal| Conv1
    Conv1 --> Internal

    V1Beta1 -->|To Internal| Conv2
    Conv2 --> Internal

    V1 -->|To Internal| Conv3
    Conv3 --> Internal

    Internal -->|From Internal| Conv4
    Conv4 --> V1Alpha1

    Internal -->|From Internal| Conv5
    Conv5 --> V1Beta1

    Internal -->|From Internal| Conv6
    Conv6 --> V1

    style Internal fill:#fff3e0
    style V1 fill:#c8e6c9
```

### Converter

**Code Reference**: `staging/src/k8s.io/apimachinery/pkg/conversion/converter.go:42`

```go
// Converter knows how to convert objects between versions
type Converter struct {
    // conversionFuncs maps source+target types to conversion functions
    conversionFuncs map[typePair]reflect.Value

    // Default conversions (for simple field copies)
    defaultConversion ConversionFunc
}

// ConversionFunc is a function that knows how to convert between types
type ConversionFunc func(a, b interface{}, scope Scope) error

// Convert converts in to out
func (c *Converter) Convert(in, out interface{}, context interface{}) error {
    inType := reflect.TypeOf(in)
    outType := reflect.TypeOf(out)

    // Look up conversion function
    pair := typePair{inType, outType}
    fn, ok := c.conversionFuncs[pair]
    if !ok {
        // Use default conversion (field-by-field copy)
        return c.defaultConversion(in, out, c.scope(context))
    }

    // Call custom conversion function
    return fn.Call([]reflect.Value{
        reflect.ValueOf(in),
        reflect.ValueOf(out),
        reflect.ValueOf(c.scope(context)),
    })[0].Interface().(error)
}
```

### Auto-Generated Conversions

Most conversion functions are **auto-generated** by `conversion-gen`:

**Code Reference**: `pkg/apis/apps/v1/conversion.go` (auto-generated)

```go
// AUTO-GENERATED FUNCTIONS START HERE

// Convert_v1_Deployment_To_apps_Deployment is an autogenerated conversion function.
func Convert_v1_Deployment_To_apps_Deployment(in *v1.Deployment, out *apps.Deployment, s conversion.Scope) error {
    out.ObjectMeta = in.ObjectMeta

    if err := Convert_v1_DeploymentSpec_To_apps_DeploymentSpec(&in.Spec, &out.Spec, s); err != nil {
        return err
    }

    if err := Convert_v1_DeploymentStatus_To_apps_DeploymentStatus(&in.Status, &out.Status, s); err != nil {
        return err
    }

    return nil
}

// Convert_apps_Deployment_To_v1_Deployment is an autogenerated conversion function.
func Convert_apps_Deployment_To_v1_Deployment(in *apps.Deployment, out *v1.Deployment, s conversion.Scope) error {
    out.ObjectMeta = in.ObjectMeta

    if err := Convert_apps_DeploymentSpec_To_v1_DeploymentSpec(&in.Spec, &out.Spec, s); err != nil {
        return err
    }

    if err := Convert_apps_DeploymentStatus_To_v1_DeploymentStatus(&in.Status, &out.Status, s); err != nil {
        return err
    }

    return nil
}
```

### Manual Conversion Functions

Some conversions need **custom logic** (for breaking changes):

**Code Reference**: `pkg/apis/apps/v1/conversion.go` (manual)

```go
// Convert_v1beta1_DeploymentSpec_To_apps_DeploymentSpec handles custom conversion logic
func Convert_v1beta1_DeploymentSpec_To_apps_DeploymentSpec(in *v1beta1.DeploymentSpec, out *apps.DeploymentSpec, s conversion.Scope) error {
    // Auto-generated parts
    if err := autoConvert_v1beta1_DeploymentSpec_To_apps_DeploymentSpec(in, out, s); err != nil {
        return err
    }

    // CUSTOM LOGIC: Handle field name change
    // v1beta1 had "rollbackTo", v1 removed it
    // Drop the field silently during conversion
    if in.RollbackTo != nil {
        // Log warning or handle migration
        klog.Warningf("Dropping deprecated field RollbackTo during conversion")
    }

    // CUSTOM LOGIC: Handle new field with default
    // v1 added "progressDeadlineSeconds" field
    if out.ProgressDeadlineSeconds == nil {
        // Set default value
        defaultDeadline := int32(600)  // 10 minutes
        out.ProgressDeadlineSeconds = &defaultDeadline
    }

    return nil
}
```

### Conversion Registration

```go
// In pkg/apis/apps/v1/register.go
func init() {
    localSchemeBuilder.Register(addConversionFuncs)
}

func addConversionFuncs(scheme *runtime.Scheme) error {
    // Register auto-generated conversions
    if err := RegisterConversions(scheme); err != nil {
        return err
    }

    // Register manual conversions
    return scheme.AddConversionFuncs(
        Convert_v1beta1_DeploymentSpec_To_apps_DeploymentSpec,
        Convert_apps_DeploymentSpec_To_v1beta1_DeploymentSpec,
        // ... more custom conversions
    )
}
```

### Using Conversions

```go
// Convert v1alpha1 Deployment → v1 Deployment
func convertDeployment(v1alpha1Dep *appsv1alpha1.Deployment) (*appsv1.Deployment, error) {
    // Step 1: Convert v1alpha1 → internal
    internalDep := &apps.Deployment{}
    err := scheme.Scheme.Convert(v1alpha1Dep, internalDep, nil)
    if err != nil {
        return nil, err
    }

    // Step 2: Convert internal → v1
    v1Dep := &appsv1.Deployment{}
    err = scheme.Scheme.Convert(internalDep, v1Dep, nil)
    if err != nil {
        return nil, err
    }

    return v1Dep, nil
}

// Or use ConvertToVersion for direct conversion
func convertDeploymentDirect(v1alpha1Dep *appsv1alpha1.Deployment) (runtime.Object, error) {
    // Automatically goes v1alpha1 → internal → v1
    return scheme.Scheme.ConvertToVersion(v1alpha1Dep, appsv1.SchemeGroupVersion)
}
```

---

## Unstructured Objects

### What are Unstructured Objects?

**Unstructured** objects are dynamic, map-based representations of K8s objects:

```go
// Structured (typed)
pod := &corev1.Pod{
    ObjectMeta: metav1.ObjectMeta{
        Name: "my-pod",
    },
    Spec: corev1.PodSpec{...},
}

// Unstructured (dynamic)
unstructuredPod := &unstructured.Unstructured{
    Object: map[string]interface{}{
        "apiVersion": "v1",
        "kind":       "Pod",
        "metadata": map[string]interface{}{
            "name": "my-pod",
        },
        "spec": map[string]interface{}{...},
    },
}
```

**When to use**:
- **Dynamic clients**: Working with unknown types (CRDs)
- **Generic tools**: kubectl, controller-runtime
- **Schema-less processing**: Don't need compile-time types

### Unstructured Data Structure

**Code Reference**: `staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/unstructured.go:44`

```go
// Unstructured allows objects that do not have Golang structs to be
// manipulated generically.
type Unstructured struct {
    // Object is a JSON-compatible map with string keys and arbitrary values
    Object map[string]interface{}
}

// Example structure:
{
    "apiVersion": "apps/v1",
    "kind": "Deployment",
    "metadata": {
        "name": "my-deployment",
        "namespace": "default",
        "labels": {
            "app": "myapp"
        }
    },
    "spec": {
        "replicas": 3,
        "selector": {
            "matchLabels": {
                "app": "myapp"
            }
        },
        "template": {
            "metadata": {
                "labels": {
                    "app": "myapp"
                }
            },
            "spec": {
                "containers": [
                    {
                        "name": "nginx",
                        "image": "nginx:1.14.2"
                    }
                ]
            }
        }
    }
}
```

### Accessor Methods

```go
// Get API version
func (u *Unstructured) GetAPIVersion() string {
    return u.Object["apiVersion"].(string)
}

// Get kind
func (u *Unstructured) GetKind() string {
    return u.Object["kind"].(string)
}

// Get name
func (u *Unstructured) GetName() string {
    metadata := u.Object["metadata"].(map[string]interface{})
    return metadata["name"].(string)
}

// Get namespace
func (u *Unstructured) GetNamespace() string {
    metadata := u.Object["metadata"].(map[string]interface{})
    if ns, ok := metadata["namespace"]; ok {
        return ns.(string)
    }
    return ""
}

// Get labels
func (u *Unstructured) GetLabels() map[string]string {
    metadata := u.Object["metadata"].(map[string]interface{})
    if labels, ok := metadata["labels"]; ok {
        labelsMap := labels.(map[string]interface{})
        result := make(map[string]string, len(labelsMap))
        for k, v := range labelsMap {
            result[k] = v.(string)
        }
        return result
    }
    return nil
}
```

### Using Unstructured Objects

```go
package main

import (
    "context"
    "fmt"

    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/client-go/dynamic"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    // Load kubeconfig
    config, err := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
    if err != nil {
        panic(err)
    }

    // Create dynamic client
    dynamicClient, err := dynamic.NewForConfig(config)
    if err != nil {
        panic(err)
    }

    // Define GVR (GroupVersionResource) for CRD
    gvr := schema.GroupVersionResource{
        Group:    "example.com",
        Version:  "v1alpha1",
        Resource: "myapps",  // Note: plural, lowercase
    }

    // Get unstructured object
    unstructuredObj, err := dynamicClient.Resource(gvr).
        Namespace("default").
        Get(context.TODO(), "my-custom-app", metav1.GetOptions{})
    if err != nil {
        panic(err)
    }

    // Access fields dynamically
    fmt.Printf("Name: %s\n", unstructuredObj.GetName())
    fmt.Printf("Namespace: %s\n", unstructuredObj.GetNamespace())
    fmt.Printf("Kind: %s\n", unstructuredObj.GetKind())

    // Access custom spec fields (nested)
    spec, found, err := unstructured.NestedMap(unstructuredObj.Object, "spec")
    if err != nil || !found {
        panic("spec not found")
    }

    replicas, found, err := unstructured.NestedInt64(spec, "replicas")
    if found {
        fmt.Printf("Replicas: %d\n", replicas)
    }

    image, found, err := unstructured.NestedString(spec, "image")
    if found {
        fmt.Printf("Image: %s\n", image)
    }
}
```

### Convert Structured ↔ Unstructured

```go
// Structured → Unstructured
func structuredToUnstructured(pod *corev1.Pod) (*unstructured.Unstructured, error) {
    unstructuredObj, err := runtime.DefaultUnstructuredConverter.ToUnstructured(pod)
    if err != nil {
        return nil, err
    }
    return &unstructured.Unstructured{Object: unstructuredObj}, nil
}

// Unstructured → Structured
func unstructuredToStructured(u *unstructured.Unstructured) (*corev1.Pod, error) {
    pod := &corev1.Pod{}
    err := runtime.DefaultUnstructuredConverter.FromUnstructured(u.Object, pod)
    if err != nil {
        return nil, err
    }
    return pod, nil
}
```

---

## Real-World Examples

### Example 1: kubectl apply Internals

**What happens when you run `kubectl apply`**:

```bash
kubectl apply -f pod.yaml
```

**Internal Flow**:

```mermaid
sequenceDiagram
    participant User
    participant kubectl
    participant Codec
    participant APIServer

    User->>kubectl: kubectl apply -f pod.yaml

    kubectl->>kubectl: Read pod.yaml file
    kubectl->>Codec: Decode YAML → Go object
    Codec->>Codec: Parse apiVersion, kind
    Codec->>Codec: Look up GVK in Scheme
    Codec->>Codec: Create *v1.Pod instance
    Codec->>Codec: Unmarshal YAML into Pod
    Codec-->>kubectl: *v1.Pod object

    kubectl->>kubectl: Apply three-way merge logic
    kubectl->>Codec: Encode Pod → JSON
    Codec-->>kubectl: JSON bytes

    kubectl->>APIServer: PATCH /api/v1/namespaces/default/pods/my-pod
    Note over APIServer: Content-Type: application/json

    APIServer->>APIServer: Decode JSON → Internal type
    APIServer->>APIServer: Validate, default, admit
    APIServer->>APIServer: Store in etcd (as Protobuf)

    APIServer-->>kubectl: 200 OK (JSON response)
    kubectl->>Codec: Decode response JSON
    Codec-->>kubectl: Updated Pod object
    kubectl-->>User: pod/my-pod configured
```

### Example 2: Controller Watch Decoding

```go
// How a controller decodes watch events
func watchPods(clientset kubernetes.Interface) {
    // Create watcher
    watcher, err := clientset.CoreV1().Pods("default").Watch(context.TODO(), metav1.ListOptions{})
    if err != nil {
        panic(err)
    }

    // Process watch events
    for event := range watcher.ResultChan() {
        // Event.Object is already decoded by client-go!
        // client-go uses codec internally:
        // 1. Reads Protobuf stream from API server
        // 2. Decodes each frame: length + GVK + protobuf data
        // 3. Looks up GVK in Scheme → *v1.Pod type
        // 4. Unmarshals protobuf → Pod object
        // 5. Returns typed event

        pod, ok := event.Object.(*corev1.Pod)
        if !ok {
            continue
        }

        switch event.Type {
        case watch.Added:
            fmt.Printf("Pod added: %s\n", pod.Name)
        case watch.Modified:
            fmt.Printf("Pod modified: %s\n", pod.Name)
        case watch.Deleted:
            fmt.Printf("Pod deleted: %s\n", pod.Name)
        }
    }
}
```

### Example 3: API Server Storage

**How API server stores in etcd**:

```go
// Simplified API server storage code
func (s *store) Create(ctx context.Context, key string, obj runtime.Object) error {
    // 1. Convert external type (v1.Pod) → internal type (core.Pod)
    internal, err := s.scheme.ConvertToVersion(obj, schema.GroupVersion{
        Group:   "",
        Version: runtime.APIVersionInternal,
    })
    if err != nil {
        return err
    }

    // 2. Encode to Protobuf (for etcd)
    data, err := s.codec.Encode(internal)  // Uses Protobuf codec
    if err != nil {
        return err
    }

    // 3. Store in etcd (key = /registry/pods/default/my-pod)
    _, err = s.etcdClient.Put(ctx, key, string(data))
    return err
}

// When reading from etcd
func (s *store) Get(ctx context.Context, key string, into runtime.Object) error {
    // 1. Get from etcd
    resp, err := s.etcdClient.Get(ctx, key)
    if err != nil {
        return err
    }

    // 2. Decode Protobuf → internal type
    internal, _, err := s.codec.Decode([]byte(resp.Kvs[0].Value), nil, nil)
    if err != nil {
        return err
    }

    // 3. Convert internal → requested external version (v1.Pod)
    return s.scheme.Convert(internal, into, nil)
}
```

---

## Performance Considerations

### Format Performance Comparison

| Operation | JSON | YAML | Protobuf | CBOR |
|-----------|------|------|----------|------|
| **Encode (small object)** | 1.0x | 1.1x | 0.15x (6.6x faster) | 0.4x (2.5x faster) |
| **Decode (small object)** | 1.0x | 1.0x | 0.12x (8.3x faster) | 0.35x (2.8x faster) |
| **Encode (large object)** | 1.0x | 1.15x | 0.1x (10x faster) | 0.38x (2.6x faster) |
| **Decode (large object)** | 1.0x | 1.05x | 0.09x (11x faster) | 0.32x (3.1x faster) |
| **Size (bytes)** | 1.0x | 0.95x | 0.35x (65% smaller) | 0.55x (45% smaller) |

**Benchmark Details**:
- Small object: Pod with 1 container (~500 bytes JSON)
- Large object: Deployment with 10 replicas, many fields (~5KB JSON)
- Numbers are relative to JSON (1.0x = baseline)

### When to Use Each Format

**JSON**:
- ✅ Human debugging
- ✅ kubectl output
- ✅ API responses to clients
- ✅ Admission webhooks
- ❌ High-throughput watch streams
- ❌ etcd storage

**YAML**:
- ✅ Configuration files
- ✅ kubectl apply manifests
- ✅ GitOps repositories
- ❌ Programmatic serialization (use JSON)
- ❌ Performance-critical paths

**Protobuf**:
- ✅ etcd storage (all objects)
- ✅ Watch streams (high throughput)
- ✅ Internal component communication
- ✅ Large list operations
- ❌ Human debugging
- ❌ External tooling (unless protobuf-aware)

**CBOR**:
- ⚠️ Experimental (Kubernetes 1.24+)
- ✅ Potential future default for external API
- ❌ Not widely adopted yet

### Optimization Tips

**1. Use Protobuf for Watches**:

```go
// Efficient: Uses Protobuf internally
watcher, err := clientset.CoreV1().Pods("").Watch(ctx, metav1.ListOptions{})

// client-go automatically negotiates Protobuf for watch
// Request header: Accept: application/vnd.kubernetes.protobuf;stream=watch
```

**2. Use CodecFactory Caching**:

```go
// Bad: Creates new codec every time
for _, obj := range objects {
    codec := serializer.NewCodecFactory(scheme).LegacyCodec(v1.SchemeGroupVersion)
    codec.Encode(obj, w)
}

// Good: Reuse codec
codecFactory := serializer.NewCodecFactory(scheme)
codec := codecFactory.LegacyCodec(v1.SchemeGroupVersion)
for _, obj := range objects {
    codec.Encode(obj, w)
}
```

**3. Avoid Unnecessary Conversions**:

```go
// Bad: Double conversion
external := getExternalObject()  // apps/v1.Deployment
internal, _ := scheme.ConvertToVersion(external, InternalVersion)
externalAgain, _ := scheme.ConvertToVersion(internal, apps/v1)

// Good: Work with external directly if possible
external := getExternalObject()
// Use external directly without conversion
```

---

## Common Pitfalls

### Pitfall 1: Missing TypeMeta When Encoding

**Problem**:
```go
pod := &corev1.Pod{
    // Missing TypeMeta!
    ObjectMeta: metav1.ObjectMeta{Name: "my-pod"},
    Spec:       corev1.PodSpec{...},
}

// Encode to JSON
json.Marshal(pod)
// Result: {"metadata": {...}, "spec": {...}}
// Missing: "apiVersion" and "kind" fields!
```

**Solution**:
```go
pod := &corev1.Pod{
    TypeMeta: metav1.TypeMeta{
        APIVersion: "v1",
        Kind:       "Pod",
    },
    ObjectMeta: metav1.ObjectMeta{Name: "my-pod"},
    Spec:       corev1.PodSpec{...},
}
```

Or use Scheme to set it automatically:
```go
scheme.Default(pod)  // Sets TypeMeta from Scheme
```

### Pitfall 2: Using json.Marshal Directly

**Problem**:
```go
// Wrong: Standard JSON encoding doesn't handle K8s types properly
data, err := json.Marshal(pod)
```

**Issues**:
- Doesn't set apiVersion/kind automatically
- Doesn't handle omitempty correctly for K8s types
- Doesn't apply K8s-specific JSON tags

**Solution**:
```go
// Correct: Use Kubernetes serializer
codecFactory := serializer.NewCodecFactory(scheme.Scheme)
encoder := codecFactory.LegacyCodec(corev1.SchemeGroupVersion)

var buf bytes.Buffer
err := encoder.Encode(pod, &buf)
```

### Pitfall 3: Type Assertions Without Checking

**Problem**:
```go
obj, _, _ := decoder.Decode(data, nil, nil)
pod := obj.(*corev1.Pod)  // Panic if not a Pod!
```

**Solution**:
```go
obj, gvk, err := decoder.Decode(data, nil, nil)
if err != nil {
    return err
}

pod, ok := obj.(*corev1.Pod)
if !ok {
    return fmt.Errorf("expected Pod, got %s", gvk)
}
```

### Pitfall 4: Forgetting Conversion Registration

**Problem**:
```go
// Custom type, but forgot to register conversions
err := scheme.Convert(v1alpha1Obj, v1Obj, nil)
// Error: no conversion function found
```

**Solution**:
```go
// Register conversions in init() or scheme registration
func init() {
    scheme.AddConversionFuncs(
        Convert_v1alpha1_MyType_To_v1_MyType,
        Convert_v1_MyType_To_v1alpha1_MyType,
    )
}
```

### Pitfall 5: Modifying Unstructured Nested Maps Directly

**Problem**:
```go
// Wrong: Modifying nested map directly can panic
u.Object["spec"].(map[string]interface{})["replicas"] = 5
// Panic if "spec" doesn't exist or is wrong type
```

**Solution**:
```go
// Correct: Use helper functions
err := unstructured.SetNestedField(u.Object, int64(5), "spec", "replicas")
if err != nil {
    return err
}

// Or check existence first
spec, found, err := unstructured.NestedMap(u.Object, "spec")
if !found || err != nil {
    return fmt.Errorf("spec not found")
}
spec["replicas"] = 5
```

---

## Testing Patterns

### Test 1: Round-Trip Encoding/Decoding

```go
func TestRoundTrip(t *testing.T) {
    original := &corev1.Pod{
        TypeMeta: metav1.TypeMeta{
            APIVersion: "v1",
            Kind:       "Pod",
        },
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-pod",
            Namespace: "default",
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{{Name: "nginx", Image: "nginx"}},
        },
    }

    codecFactory := serializer.NewCodecFactory(scheme.Scheme)
    codec := codecFactory.LegacyCodec(corev1.SchemeGroupVersion)

    // Encode
    var buf bytes.Buffer
    err := codec.Encode(original, &buf)
    require.NoError(t, err)

    // Decode
    decoder := codecFactory.UniversalDeserializer()
    obj, _, err := decoder.Decode(buf.Bytes(), nil, nil)
    require.NoError(t, err)

    // Verify
    decoded := obj.(*corev1.Pod)
    assert.Equal(t, original.Name, decoded.Name)
    assert.Equal(t, original.Namespace, decoded.Namespace)
    assert.Len(t, decoded.Spec.Containers, 1)
}
```

### Test 2: Multi-Format Support

```go
func TestMultiFormat(t *testing.T) {
    pod := &corev1.Pod{
        TypeMeta: metav1.TypeMeta{APIVersion: "v1", Kind: "Pod"},
        ObjectMeta: metav1.ObjectMeta{Name: "test-pod"},
    }

    codecFactory := serializer.NewCodecFactory(scheme.Scheme)

    tests := []struct {
        name   string
        format string
    }{
        {"JSON", "application/json"},
        {"YAML", "application/yaml"},
        {"Protobuf", "application/vnd.kubernetes.protobuf"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Get appropriate serializer
            info, ok := runtime.SerializerInfoForMediaType(
                codecFactory.SupportedMediaTypes(), tt.format)
            require.True(t, ok, "format not supported: %s", tt.format)

            // Encode
            var buf bytes.Buffer
            err := info.Serializer.Encode(pod, &buf)
            require.NoError(t, err)

            // Decode
            obj, _, err := codecFactory.UniversalDeserializer().Decode(buf.Bytes(), nil, nil)
            require.NoError(t, err)

            decoded := obj.(*corev1.Pod)
            assert.Equal(t, pod.Name, decoded.Name)
        })
    }
}
```

### Test 3: Conversion Testing

```go
func TestConversion(t *testing.T) {
    // Create v1alpha1 object
    v1alpha1Dep := &appsv1alpha1.Deployment{
        ObjectMeta: metav1.ObjectMeta{Name: "test-dep"},
        Spec: appsv1alpha1.DeploymentSpec{
            Replicas: int32Ptr(3),
        },
    }

    // Convert to internal
    internal := &apps.Deployment{}
    err := scheme.Scheme.Convert(v1alpha1Dep, internal, nil)
    require.NoError(t, err)

    // Convert to v1
    v1Dep := &appsv1.Deployment{}
    err = scheme.Scheme.Convert(internal, v1Dep, nil)
    require.NoError(t, err)

    // Verify
    assert.Equal(t, v1alpha1Dep.Name, v1Dep.Name)
    assert.Equal(t, *v1alpha1Dep.Spec.Replicas, *v1Dep.Spec.Replicas)
}
```

### Test 4: Unstructured Operations

```go
func TestUnstructured(t *testing.T) {
    // Create structured Pod
    pod := &corev1.Pod{
        TypeMeta: metav1.TypeMeta{APIVersion: "v1", Kind: "Pod"},
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-pod",
            Namespace: "default",
            Labels:    map[string]string{"app": "test"},
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{{Name: "nginx", Image: "nginx"}},
        },
    }

    // Convert to unstructured
    unstructuredMap, err := runtime.DefaultUnstructuredConverter.ToUnstructured(pod)
    require.NoError(t, err)

    u := &unstructured.Unstructured{Object: unstructuredMap}

    // Test accessors
    assert.Equal(t, "test-pod", u.GetName())
    assert.Equal(t, "default", u.GetNamespace())
    assert.Equal(t, "Pod", u.GetKind())
    assert.Equal(t, "v1", u.GetAPIVersion())

    // Test nested field access
    containers, found, err := unstructured.NestedSlice(u.Object, "spec", "containers")
    require.True(t, found)
    require.NoError(t, err)
    assert.Len(t, containers, 1)

    // Modify nested field
    err = unstructured.SetNestedField(u.Object, "nginx:latest", "spec", "containers", "0", "image")
    require.NoError(t, err)

    // Convert back to structured
    modifiedPod := &corev1.Pod{}
    err = runtime.DefaultUnstructuredConverter.FromUnstructured(u.Object, modifiedPod)
    require.NoError(t, err)

    assert.Equal(t, "nginx:latest", modifiedPod.Spec.Containers[0].Image)
}
```

---

## Summary

### Key Takeaways

1. **Serialization = Encoding/Decoding**: Go struct ↔ Wire format (JSON/YAML/Protobuf/CBOR)
2. **Codec Pipeline**: Request → Decode → Convert → Process → Convert → Encode → Response
3. **Multiple Formats**: JSON (readable), Protobuf (fast), YAML (manifests), CBOR (future)
4. **Content Negotiation**: Accept header selects format
5. **Conversion Framework**: Hub-and-spoke model with internal types
6. **Auto-Generated Conversions**: Most conversions auto-generated, some manual
7. **Unstructured**: Dynamic, map-based objects for unknown types (CRDs)
8. **Performance**: Protobuf is 5-10x faster, 35% smaller than JSON

### When You Need This Knowledge

**Common use cases**:
- Building controllers that work with CRDs
- Creating admission webhooks
- Writing kubectl plugins
- Debugging serialization issues
- Optimizing performance (choosing format)
- Understanding API versioning

**Built into client-go**:
- Standard clients handle serialization automatically
- Informers use Protobuf for watches automatically
- REST clients negotiate content types

### "Aha Moments" Recap

1. **kubectl YAML → Go**: Serialization pipeline decodes YAML to Go struct
2. **Protobuf for Performance**: etcd stores everything as Protobuf (5-10x faster)
3. **Internal Types**: Conversion goes through internal hub (v1alpha1 → internal → v1)
4. **Unstructured for CRDs**: Dynamic clients use maps instead of structs
5. **Auto-Generated**: Most conversion code is generated, not written by hand

### Next Steps

**Continue to**:
- **[07-watch-meta-types.md](07-watch-meta-types.md)**: Watch protocol and ObjectMeta
- **[02-rest-clients-discovery.md](02-rest-clients-discovery.md)**: How clients use codecs
- **[03-informers-sharedinformers.md](03-informers-sharedinformers.md)**: How informers deserialize watch events

**Related Topics**:
- **Protobuf Code Generation**: How to generate .pb.go files
- **API Versioning Strategy**: Best practices for evolving APIs
- **Custom Serializers**: Implementing custom formats

---

## References

### Code Locations

**Serialization**:
- `staging/src/k8s.io/apimachinery/pkg/runtime/serializer/` - Serializer implementations
- `staging/src/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go` - JSON serializer
- `staging/src/k8s.io/apimachinery/pkg/runtime/serializer/protobuf/protobuf.go` - Protobuf serializer
- `staging/src/k8s.io/apimachinery/pkg/runtime/serializer/codec_factory.go` - CodecFactory

**Conversion**:
- `staging/src/k8s.io/apimachinery/pkg/conversion/converter.go` - Converter implementation
- `pkg/apis/apps/v1/conversion.go` - Example conversion functions
- `staging/src/k8s.io/apimachinery/pkg/runtime/converter.go` - Runtime conversion

**Unstructured**:
- `staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/unstructured.go` - Unstructured type
- `staging/src/k8s.io/apimachinery/pkg/runtime/converter.go` - Structured ↔ Unstructured conversion

### Tools

- **conversion-gen**: Auto-generates conversion functions
- **protoc**: Protobuf compiler
- **go-to-protobuf**: Generates .proto from Go structs

### Further Reading

- **KEP-2885**: [Server Side Unknown Field Validation](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2885-server-side-unknown-field-validation)
- **KEP-2896**: [OpenAPI V3](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2896-openapi-v3)
- **Docs**: [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- **Blog**: [Kubernetes API Versioning](https://kubernetes.io/blog/2016/09/kubernetes-1-4-introducing-the-new-api-versioning-system/)

---

**Document Status**: ✅ Complete
**Last Updated**: 2025-11-05
**Previous Document**: [05-runtime-scheme.md](05-runtime-scheme.md) - Scheme and type registry
**Next Document**: [07-watch-meta-types.md](07-watch-meta-types.md) - Watch protocol and ObjectMeta
**Word Count**: ~12,000 words
**Code References**: 30+
**Diagrams**: 15
**Estimated Reading Time**: 60 minutes

---

*This document is part of the Kubernetes Shared Libraries Architecture Documentation for course development. For questions or corrections, please refer to the main [README.md](README.md).*
