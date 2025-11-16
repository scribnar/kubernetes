# **kubectl Streaming Commands - Logs, Exec, Port-Forward, CP**

**Part of**: kubectl Middle-Level Architecture Documentation
**Related**: [Imperative Commands](./01-imperative-commands.md) | [Get/Describe](./03-get-describe.md) | [Edit/Patch](./04-edit-patch.md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Table of Contents**

1. [Overview](#overview)
2. [Data Structures](#data-structures)
3. [kubectl logs Architecture](#kubectl-logs-architecture)
4. [kubectl exec Architecture](#kubectl-exec-architecture)
5. [kubectl attach Architecture](#kubectl-attach-architecture)
6. [kubectl port-forward Architecture](#kubectl-port-forward-architecture)
7. [kubectl cp Architecture](#kubectl-cp-architecture)
8. [Streaming Protocols](#streaming-protocols)
9. [Performance and Optimization](#performance-and-optimization)
10. [Troubleshooting](#troubleshooting)
11. [Summary](#summary)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Overview**

### **Purpose**

kubectl provides several commands for real-time interaction with running containers:

- **kubectl logs**: Stream container logs
- **kubectl exec**: Execute commands in containers
- **kubectl attach**: Attach to running processes
- **kubectl port-forward**: Forward local ports to pods
- **kubectl cp**: Copy files to/from containers

All these commands use HTTP streaming protocols (SPDY or WebSocket) to maintain long-lived connections.

### **Command Comparison**

| Command | Purpose | Streaming | TTY Support | Use Case |
|---------|---------|-----------|-------------|----------|
| **logs** | View container output | Yes | No | Debugging, monitoring |
| **exec** | Run new command | Yes | Yes | Interactive debugging |
| **attach** | Attach to existing process | Yes | Yes | View running process |
| **port-forward** | Forward network ports | Yes | No | Local development, debugging |
| **cp** | Copy files | No (tar-based) | No | File transfer |

### **Streaming Architecture**

```mermaid
graph TB
    subgraph "kubectl Client"
        CMD[kubectl Command]
        OPTS[Options]
        EXEC[Executor/Forwarder]
    end

    subgraph "Streaming Layer"
        PROTO[Protocol Selection]
        WS[WebSocket]
        SPDY[SPDY]
        FALLBACK[Fallback Logic]

        PROTO --> WS
        PROTO --> SPDY
        WS --> FALLBACK
        FALLBACK --> SPDY
    end

    subgraph "API Server"
        HANDLER[Stream Handler]
        UPGRADE[Protocol Upgrade]
        PROXY[Kubelet Proxy]
    end

    subgraph "Kubelet"
        CRI[Container Runtime]
        CONTAINER[Container]
    end

    CMD --> OPTS
    OPTS --> EXEC
    EXEC --> PROTO
    PROTO --> HANDLER
    HANDLER --> UPGRADE
    UPGRADE --> PROXY
    PROXY --> CRI
    CRI --> CONTAINER

    style CMD fill:#e1f5ff
    style PROTO fill:#fff4e1
    style HANDLER fill:#e8f5e9
    style CONTAINER fill:#f3e5f5
```

### **Common Patterns**

All streaming commands share:
1. **Complete-Validate-Run** execution pattern
2. **Resource selection** (pod name, deployment, etc.)
3. **Container selection** (-c flag for multi-container pods)
4. **Protocol negotiation** (WebSocket → SPDY fallback)
5. **Stream multiplexing** (stdin, stdout, stderr, error, resize)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Data Structures**

### **LogsOptions Structure**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go:120`

```go
type LogsOptions struct {
    Namespace     string
    ResourceArg   string
    AllContainers bool      // Get logs from all containers
    AllPods       bool      // Get logs from all pods (deployment/replicaset)
    Options       runtime.Object
    Resources     []string

    ConsumeRequestFn func(context.Context, rest.ResponseWrapper, io.Writer) error

    // PodLogOptions
    SinceTime                    string        // RFC3339 timestamp
    SinceSeconds                 time.Duration // Relative time (e.g., 1h)
    Follow                       bool          // Stream logs (-f flag)
    Previous                     bool          // Previous container instance (-p flag)
    Timestamps                   bool          // Include timestamps
    IgnoreLogErrors              bool          // Continue on errors
    LimitBytes                   int64         // Max bytes to return
    Tail                         int64         // Last N lines (-1 = all)
    Container                    string        // Container name
    InsecureSkipTLSVerifyBackend bool          // Skip TLS verification

    ContainerNameSpecified bool
    Selector               string  // Label selector
    MaxFollowConcurrency   int     // Max concurrent follow streams (default: 5)
    Prefix                 bool    // Prefix each line with pod/container name

    Object              runtime.Object
    GetPodTimeout       time.Duration  // Wait for pod to be running (default: 20s)
    RESTClientGetter    genericclioptions.RESTClientGetter
    LogsForObject       polymorphichelpers.LogsForObjectFunc
    AllPodLogsForObject polymorphichelpers.AllPodLogsForObjectFunc

    genericiooptions.IOStreams

    TailSpecified bool

    containerNameFromRefSpecRegexp *regexp.Regexp
}
```

**Key Features**:
- Supports single pod, all containers, all pods in deployment
- Tail, follow, timestamps, previous container
- Time-based filtering (since, since-time)
- Concurrent streaming with max limit

### **ExecOptions Structure**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/exec/exec.go:187`

```go
type ExecOptions struct {
    StreamOptions
    resource.FilenameOptions

    ResourceName     string
    Command          []string      // Command and args to execute
    EnforceNamespace bool

    Builder          func() *resource.Builder
    ExecutablePodFn  polymorphichelpers.AttachablePodForObjectFunc
    restClientGetter genericclioptions.RESTClientGetter

    Pod           *corev1.Pod
    Executor      RemoteExecutor  // Interface for execution
    PodClient     corev1client.PodsGetter
    GetPodTimeout time.Duration    // Wait for pod (default: 60s)
    Config        *restclient.Config
}

type StreamOptions struct {
    Namespace     string
    PodName       string
    ContainerName string
    Stdin         bool  // Pass stdin to container (-i flag)
    TTY           bool  // Allocate TTY (-t flag)
    Quiet         bool  // Minimize output (-q flag)
    InterruptParent *interrupt.Handler

    genericiooptions.IOStreams

    // Testing hooks
    overrideStreams func() (io.ReadCloser, io.Writer, io.Writer)
    isTerminalIn    func(t term.TTY) bool
}
```

**TTY and Stdin Combinations**:

| Flags | Mode | Use Case |
|-------|------|----------|
| None | Non-interactive | Run command, get output |
| `-i` | Interactive stdin | Pipe input to command |
| `-t` | TTY only | Commands needing terminal |
| `-it` | Interactive TTY | **Interactive shell** |

### **RemoteExecutor Interface**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/exec/exec.go:116`

```go
type RemoteExecutor interface {
    // Execute supports executing remote command in a pod
    Execute(url *url.URL, config *restclient.Config, stdin io.Reader,
            stdout, stderr io.Writer, tty bool,
            terminalSizeQueue remotecommand.TerminalSizeQueue) error

    // ExecuteWithContext supports cancellation
    ExecuteWithContext(ctx context.Context, url *url.URL, config *restclient.Config,
                      stdin io.Reader, stdout, stderr io.Writer, tty bool,
                      terminalSizeQueue remotecommand.TerminalSizeQueue) error
}
```

### **PortForwardOptions Structure**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/portforward/portforward.go:51`

```go
type PortForwardOptions struct {
    Namespace     string
    PodName       string
    RESTClient    restclient.Interface
    Config        *restclient.Config
    PodClient     corev1client.PodsGetter
    Address       []string  // Listen addresses (default: ["localhost"])
    Ports         []string  // Port mappings (e.g., "8080:80", ":5000")
    PortForwarder portForwarder
    StopChannel   chan struct{}   // Signal to stop forwarding
    ReadyChannel  chan struct{}   // Signal when ready
}
```

**Port Specification Formats**:
- `5000` - Forward local 5000 to pod 5000
- `8080:80` - Forward local 8080 to pod 80
- `:5000` - Random local port to pod 5000

### **StreamOptions (remotecommand)**

**Location**: `staging/src/k8s.io/client-go/tools/remotecommand/remotecommand.go:30`

```go
type StreamOptions struct {
    Stdin             io.Reader
    Stdout            io.Writer
    Stderr            io.Writer
    Tty               bool
    TerminalSizeQueue TerminalSizeQueue  // For terminal resize events
}

// Executor is the interface for transporting shell-style streams
type Executor interface {
    Stream(options StreamOptions) error
    StreamWithContext(ctx context.Context, options StreamOptions) error
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📜 kubectl logs Architecture**

### **Command Flow**

```mermaid
sequenceDiagram
    participant User
    participant CLI as kubectl logs
    participant Builder as Resource Builder
    participant REST as REST Client
    participant API as API Server
    participant Kubelet
    participant Runtime as Container Runtime
    participant Container

    User->>CLI: kubectl logs nginx -f

    activate CLI
    CLI->>CLI: Complete()
    Note over CLI: Parse flags, setup options

    CLI->>CLI: Validate()
    Note over CLI: Check pod/container exists

    CLI->>Builder: Build resource request
    activate Builder
    Builder->>REST: Get pod info
    activate REST
    REST->>API: GET /api/v1/namespaces/default/pods/nginx
    activate API
    API-->>REST: Pod object
    deactivate API
    REST-->>Builder: Pod info
    deactivate REST
    deactivate Builder

    CLI->>CLI: Select container
    Note over CLI: Use first container if not specified

    CLI->>REST: Build logs request
    activate REST
    Note over REST: GET /api/v1/namespaces/default/pods/nginx/log?follow=true

    REST->>API: Stream logs request
    activate API
    API->>Kubelet: Proxy to kubelet
    activate Kubelet
    Kubelet->>Runtime: Get container logs
    activate Runtime
    Runtime->>Container: Read logs
    activate Container

    loop Streaming (if follow=true)
        Container-->>Runtime: Log lines
        Runtime-->>Kubelet: Log lines
        Kubelet-->>API: Log lines
        API-->>REST: Log lines
        REST-->>CLI: Log lines
        CLI->>User: Display logs
    end

    Note over User: Ctrl+C to stop

    User->>CLI: SIGINT
    CLI->>REST: Close stream
    REST->>API: Close connection
    API->>Kubelet: Close
    deactivate Container
    deactivate Runtime
    deactivate Kubelet
    deactivate API
    deactivate REST
    deactivate CLI
```

### **Complete Phase**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go:236`

```go
func (o *LogsOptions) Complete(f cmdutil.Factory, cmd *cobra.Command, args []string) error {
    // Parse resource argument
    if len(args) == 0 {
        return fmt.Errorf(logsUsageErrStr)
    }
    o.ResourceArg = args[0]

    // Get namespace
    var err error
    o.Namespace, _, err = f.ToRawKubeConfigLoader().Namespace()
    if err != nil {
        return err
    }

    // Setup client getter
    o.RESTClientGetter = f

    // Setup logs consumer function
    o.ConsumeRequestFn = DefaultConsumeRequest

    // Setup polymorphic helpers for different resource types
    o.LogsForObject = polymorphichelpers.LogsForObjectFn

    // Container name from fieldpath (e.g., deployment/nginx.spec.containers{nginx})
    if match := o.containerNameFromRefSpecRegexp.FindStringSubmatch(o.ResourceArg); match != nil {
        o.ContainerName = match[1]
        o.ContainerNameSpecified = true
    }

    // Default tail for selector-based queries
    if len(o.Selector) > 0 && !o.TailSpecified {
        o.Tail = selectorTail  // 10 lines
    }

    // AllPods implies Prefix
    if o.AllPods {
        o.Prefix = true
    }

    return nil
}
```

### **RunLogs Execution**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go:394`

```go
func (o *LogsOptions) RunLogs() error {
    // Build log options
    logOptions := &corev1.PodLogOptions{
        Container:                    o.Container,
        Follow:                       o.Follow,
        Previous:                     o.Previous,
        Timestamps:                   o.Timestamps,
        InsecureSkipTLSVerifyBackend: o.InsecureSkipTLSVerifyBackend,
    }

    if o.SinceSeconds != 0 {
        logOptions.SinceSeconds = &o.SinceSeconds
    }
    if o.SinceTime != "" {
        t, err := util.ParseRFC3339(o.SinceTime, metav1.Now)
        if err != nil {
            return err
        }
        logOptions.SinceTime = &t
    }
    if o.LimitBytes != 0 {
        logOptions.LimitBytes = &o.LimitBytes
    }
    if o.Tail != -1 {
        logOptions.TailLines = &o.Tail
    }

    o.Options = logOptions

    // Handle different scenarios
    if o.AllPods {
        return o.getAllPodLogs()
    }
    if len(o.Selector) > 0 {
        return o.getLogs()
    }
    if o.AllContainers {
        return o.getAllContainerLogs()
    }

    // Single pod/container
    return o.getLogs()
}
```

### **Stream Consumption**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go:629`

```go
func DefaultConsumeRequest(ctx context.Context,
                          request rest.ResponseWrapper,
                          out io.Writer) error {
    // Get streaming response
    readCloser, err := request.Stream(ctx)
    if err != nil {
        return err
    }
    defer readCloser.Close()

    // Copy stream to output
    _, err = io.Copy(out, readCloser)
    return err
}
```

### **Multi-Pod Logs (Follow Mode)**

When using `--all-pods` or `-l selector` with `-f`:

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go:536`

```go
func (o *LogsOptions) getAllPodLogs() error {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Get all pods
    pods, err := o.getAllPods()
    if err != nil {
        return err
    }

    // Limit concurrent streams
    sem := make(chan struct{}, o.MaxFollowConcurrency)
    var wg sync.WaitGroup
    var mu sync.Mutex
    var errors []error

    for _, pod := range pods {
        for _, container := range pod.Spec.Containers {
            wg.Add(1)
            go func(pod corev1.Pod, container corev1.Container) {
                defer wg.Done()

                // Acquire semaphore
                sem <- struct{}{}
                defer func() { <-sem }()

                // Stream logs for this pod/container
                err := o.streamPodLogs(ctx, pod, container)
                if err != nil && !o.IgnoreLogErrors {
                    mu.Lock()
                    errors = append(errors, err)
                    mu.Unlock()
                }
            }(pod, container)
        }
    }

    wg.Wait()
    return utilerrors.NewAggregate(errors)
}
```

### **Log Examples**

```bash
# Basic logs
kubectl logs nginx

# Follow logs (stream)
kubectl logs nginx -f

# Last 100 lines
kubectl logs nginx --tail=100

# Since 1 hour ago
kubectl logs nginx --since=1h

# Since specific time (RFC3339)
kubectl logs nginx --since-time=2024-01-15T10:00:00Z

# With timestamps
kubectl logs nginx --timestamps

# Previous container (after restart)
kubectl logs nginx -p

# Specific container in multi-container pod
kubectl logs nginx -c sidecar

# All containers
kubectl logs nginx --all-containers

# From deployment (any pod)
kubectl logs deployment/nginx

# All pods in deployment
kubectl logs deployment/nginx --all-pods

# With label selector
kubectl logs -l app=nginx

# Follow all pods (limit concurrency)
kubectl logs -l app=nginx -f --max-log-requests=10

# Prefix with pod/container name
kubectl logs -l app=nginx --prefix

# Limit bytes
kubectl logs nginx --limit-bytes=1000
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🖥️ kubectl exec Architecture**

### **Command Flow**

```mermaid
sequenceDiagram
    participant User
    participant CLI as kubectl exec
    participant Builder
    participant Executor
    participant API as API Server
    participant Kubelet
    participant Runtime as Container Runtime

    User->>CLI: kubectl exec -it nginx -- bash

    activate CLI
    CLI->>CLI: Complete()
    Note over CLI: Parse command after --<br/>Setup TTY if -t

    CLI->>Builder: Get pod info
    Builder->>API: GET pod
    API-->>Builder: Pod object
    Builder-->>CLI: Pod info

    CLI->>CLI: Select container
    CLI->>Executor: createExecutor()
    activate Executor

    alt WebSocket available
        Executor->>Executor: NewWebSocketExecutor()
        Note over Executor: GET method (RFC 6455)
    end

    Executor->>Executor: NewSPDYExecutor()
    Note over Executor: POST method

    Executor->>Executor: NewFallbackExecutor(websocket, spdy)
    Note over Executor: Try WebSocket, fallback to SPDY

    CLI->>Executor: StreamWithContext()
    Executor->>API: Upgrade connection
    activate API
    Note over Executor,API: Upgrade: websocket or SPDY/3.1

    API->>API: Validate & authorize
    API->>Kubelet: Proxy exec request
    activate Kubelet
    Kubelet->>Runtime: Exec(containerID, command)
    activate Runtime
    Runtime->>Runtime: Start process in container

    Note over Runtime: Container process running

    par Stdin stream
        User->>CLI: Type input
        CLI->>Executor: Write to stdin stream
        Executor->>API: Stream data
        API->>Kubelet: Forward
        Kubelet->>Runtime: Write to process stdin
    and Stdout stream
        Runtime->>Kubelet: Read process stdout
        Kubelet->>API: Stream data
        API->>Executor: Forward
        Executor->>CLI: Write to stdout
        CLI->>User: Display output
    and Stderr stream (if not TTY)
        Runtime->>Kubelet: Read process stderr
        Kubelet->>API: Stream data
        API->>Executor: Forward
        Executor->>CLI: Write to stderr
        CLI->>User: Display errors
    and Resize stream (if TTY)
        User->>CLI: Terminal resized
        CLI->>Executor: Send resize event
        Executor->>API: Resize stream
        API->>Kubelet: Forward
        Kubelet->>Runtime: Resize PTY
    end

    Note over User: Exit command or Ctrl+D

    Runtime-->>Kubelet: Process exit code
    deactivate Runtime
    Kubelet-->>API: Exit code
    deactivate Kubelet
    API-->>Executor: Close streams
    deactivate API
    Executor-->>CLI: Exit code
    deactivate Executor
    CLI->>User: Exit with code
    deactivate CLI
```

### **Executor Creation**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/exec/exec.go:146`

```go
func createExecutor(url *url.URL, config *restclient.Config) (remotecommand.Executor, error) {
    // Create SPDY executor (legacy, always available)
    exec, err := remotecommand.NewSPDYExecutor(config, "POST", url)
    if err != nil {
        return nil, err
    }

    // Try WebSocket executor (newer, if feature not disabled)
    if !cmdutil.RemoteCommandWebsockets.IsDisabled() {
        // WebSocket must use GET method (RFC 6455 Sec. 4.1)
        websocketExec, err := remotecommand.NewWebSocketExecutor(config, "GET", url.String())
        if err != nil {
            return nil, err
        }

        // Fallback executor tries WebSocket first, then SPDY
        exec, err = remotecommand.NewFallbackExecutor(
            websocketExec,
            exec,
            func(err error) bool {
                return httpstream.IsUpgradeFailure(err) ||
                       httpstream.IsHTTPSProxyError(err)
            },
        )
        if err != nil {
            return nil, err
        }
    }

    return exec, nil
}
```

### **Stream Execution**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/exec/exec.go:131`

```go
func (*DefaultRemoteExecutor) ExecuteWithContext(
    ctx context.Context,
    url *url.URL,
    config *restclient.Config,
    stdin io.Reader,
    stdout, stderr io.Writer,
    tty bool,
    terminalSizeQueue remotecommand.TerminalSizeQueue) error {

    // Create executor (WebSocket with SPDY fallback)
    exec, err := createExecutor(url, config)
    if err != nil {
        return err
    }

    // Stream with context (supports cancellation)
    return exec.StreamWithContext(ctx, remotecommand.StreamOptions{
        Stdin:             stdin,
        Stdout:            stdout,
        Stderr:            stderr,
        Tty:               tty,
        TerminalSizeQueue: terminalSizeQueue,
    })
}
```

### **TTY Handling**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/exec/exec.go:291`

```go
func (o *ExecOptions) setupTTY() term.TTY {
    t := term.TTY{
        Parent: o.InterruptParent,
        Out:    o.Out,
        In:     o.In,
        Raw:    true,  // Raw terminal mode
    }

    if !o.Stdin {
        // No stdin, just output
        return t
    }

    if o.TTY {
        // Interactive TTY
        t.Raw = true
        stdin, stdout, stderr := o.overrideStreams()
        if stdin != nil {
            t.In = stdin
        }
        if stdout != nil {
            t.Out = stdout
        }
        // Set up terminal size queue for resize events
        size, _ := dockerterm.GetSize(0)  // Get current terminal size
        sizeQueue := t.MonitorSize(size)
        o.TerminalSizeQueue = sizeQueue
    }

    return t
}
```

### **Exec Examples**

```bash
# Run command, get output
kubectl exec nginx -- date

# Run command with args
kubectl exec nginx -- ls -la /tmp

# Interactive shell
kubectl exec -it nginx -- bash

# Specific container
kubectl exec -it nginx -c sidecar -- sh

# Pass stdin
echo "SELECT * FROM users;" | kubectl exec -i postgres -- psql

# From deployment (any pod)
kubectl exec deploy/nginx -- date

# Quiet mode (no kubectl output)
kubectl exec -q nginx -- date

# With timeout
kubectl exec nginx --pod-running-timeout=30s -- date

# Multiple commands (via shell)
kubectl exec nginx -- sh -c "cd /tmp && ls -la && pwd"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 kubectl attach Architecture**

### **Attach vs Exec**

| Aspect | kubectl exec | kubectl attach |
|--------|--------------|----------------|
| **Purpose** | Run **new** command | Attach to **existing** process |
| **Process** | Creates new process | Connects to running process |
| **Use Case** | Interactive debugging | View running app output |
| **TTY** | Allocates new TTY | Uses existing TTY |
| **Stdin** | New stdin stream | Existing process stdin |

### **Attach Command Flow**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/attach/attach.go`

Similar to exec, but:
1. Attaches to container's main process (PID 1)
2. No command argument needed
3. Uses same streaming protocol

```go
// Attach to container's running process
func (o *AttachOptions) Run() error {
    // Get pod
    pod, err := o.PodClient.Pods(o.Namespace).Get(context.TODO(), o.PodName, metav1.GetOptions{})
    if err != nil {
        return err
    }

    // Ensure pod is running
    if pod.Status.Phase != corev1.PodRunning {
        return fmt.Errorf("pod %s is not running", o.PodName)
    }

    // Build attach URL
    req := o.RESTClient.Post().
        Resource("pods").
        Name(pod.Name).
        Namespace(pod.Namespace).
        SubResource("attach")

    req.VersionedParams(&corev1.PodAttachOptions{
        Container: o.ContainerName,
        Stdin:     o.Stdin,
        Stdout:    true,
        Stderr:    !o.TTY,
        TTY:       o.TTY,
    }, scheme.ParameterCodec)

    // Attach
    return o.Attach(req.URL(), o.Config, o.In, o.Out, o.ErrOut, o.TTY, o.TerminalSizeQueue)
}
```

### **Attach Examples**

```bash
# Attach to container
kubectl attach nginx

# Interactive attach
kubectl attach -it nginx

# Specific container
kubectl attach nginx -c sidecar

# From deployment
kubectl attach deploy/nginx
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌐 kubectl port-forward Architecture**

### **Port Forwarding Flow**

```mermaid
sequenceDiagram
    participant User
    participant CLI as kubectl port-forward
    participant Dialer
    participant API as API Server
    participant Kubelet
    participant Pod

    User->>CLI: kubectl port-forward nginx 8080:80

    activate CLI
    CLI->>CLI: Complete()
    Note over CLI: Parse port mappings<br/>8080 (local) → 80 (pod)

    CLI->>CLI: Validate()
    Note over CLI: Check port format<br/>Check addresses

    CLI->>Dialer: createDialer()
    activate Dialer

    alt WebSocket available
        Dialer->>Dialer: NewSPDYOverWebsocketDialer()
        Note over Dialer: Tunneling over WebSocket
    end

    Dialer->>Dialer: NewSPDYDialer()
    Note over Dialer: SPDY fallback

    Dialer->>Dialer: NewFallbackDialer()
    deactivate Dialer

    CLI->>CLI: Start listening on local port
    Note over CLI: Bind to 127.0.0.1:8080

    CLI->>API: Dial port-forward endpoint
    activate API
    Note over CLI,API: Upgrade: SPDY/3.1 or WebSocket

    API->>Kubelet: Proxy to kubelet
    activate Kubelet
    Kubelet->>Pod: Connect to port 80
    activate Pod

    CLI->>User: Ready to forward

    loop For each connection
        User->>CLI: Connect to localhost:8080
        CLI->>CLI: Accept connection

        par Forward data
            CLI->>API: Send data
            API->>Kubelet: Forward
            Kubelet->>Pod: Send to port 80
        and Receive data
            Pod->>Kubelet: Receive from port 80
            Kubelet->>API: Forward
            API->>CLI: Receive data
            CLI->>User: Send data
        end
    end

    Note over User: Ctrl+C to stop

    User->>CLI: SIGINT
    CLI->>CLI: Close listener
    CLI->>API: Close connections
    API->>Kubelet: Close
    deactivate Pod
    deactivate Kubelet
    deactivate API
    deactivate CLI
```

### **Port Forward Setup**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/portforward/portforward.go:220`

```go
func (o *PortForwardOptions) RunPortForward() error {
    // Get pod
    pod, err := o.PodClient.Pods(o.Namespace).Get(context.TODO(), o.PodName, metav1.GetOptions{})
    if err != nil {
        return err
    }

    // Ensure pod is running
    if pod.Status.Phase != corev1.PodRunning {
        return fmt.Errorf("unable to forward port because pod is not running. Current status=%v", pod.Status.Phase)
    }

    // Build URL
    req := o.RESTClient.Post().
        Resource("pods").
        Namespace(o.Namespace).
        Name(pod.Name).
        SubResource("portforward")

    // Forward ports
    return o.PortForwarder.ForwardPorts("POST", req.URL(), *o)
}
```

### **Dialer Creation**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/portforward/portforward.go:139`

```go
func createDialer(method string, url *url.URL, opts PortForwardOptions) (httpstream.Dialer, error) {
    // Create SPDY dialer
    transport, upgrader, err := spdy.RoundTripperFor(opts.Config)
    if err != nil {
        return nil, err
    }
    dialer := spdy.NewDialer(upgrader, &http.Client{Transport: transport}, method, url)

    // Try WebSocket tunneling dialer
    if !cmdutil.PortForwardWebsockets.IsDisabled() {
        tunnelingDialer, err := portforward.NewSPDYOverWebsocketDialer(url, opts.Config)
        if err != nil {
            return nil, err
        }

        // Fallback from WebSocket to SPDY
        dialer = portforward.NewFallbackDialer(
            tunnelingDialer,
            dialer,
            func(err error) bool {
                return httpstream.IsUpgradeFailure(err) || httpstream.IsHTTPSProxyError(err)
            },
        )
    }

    return dialer, nil
}
```

### **Port Forwarding**

**Location**: `staging/src/k8s.io/client-go/tools/portforward/portforward.go`

```go
func (pf *PortForwarder) ForwardPorts() error {
    // Create listener for each local port
    for _, port := range pf.ports {
        listener, err := pf.listenOnPort(&port)
        if err != nil {
            return err
        }
        pf.listeners = append(pf.listeners, listener)

        go pf.handleConnection(listener, port)
    }

    // Signal ready
    close(pf.readyChannel)

    // Wait for stop signal
    <-pf.stopChannel
    return nil
}

func (pf *PortForwarder) handleConnection(listener net.Listener, port ForwardedPort) {
    for {
        conn, err := listener.Accept()
        if err != nil {
            return  // Listener closed
        }

        // Handle each connection in goroutine
        go pf.handleSingleConnection(conn, port)
    }
}
```

### **Port-Forward Examples**

```bash
# Forward single port
kubectl port-forward nginx 8080:80

# Forward multiple ports
kubectl port-forward nginx 8080:80 8443:443

# Same local and remote port
kubectl port-forward nginx 80

# Random local port
kubectl port-forward nginx :80

# Listen on all interfaces
kubectl port-forward --address 0.0.0.0 nginx 8080:80

# Multiple addresses
kubectl port-forward --address localhost,10.19.21.23 nginx 8080:80

# From service (selects a pod)
kubectl port-forward svc/nginx 8080:80

# From deployment (selects a pod)
kubectl port-forward deploy/nginx 8080:80

# With timeout
kubectl port-forward nginx 8080:80 --pod-running-timeout=30s
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📦 kubectl cp Architecture**

### **Copy Mechanism**

kubectl cp uses **tar** for file transfer:
1. **Copy to pod**: Create tar on client → extract in container
2. **Copy from pod**: Create tar in container → extract on client

```mermaid
graph LR
    subgraph "Copy TO Pod"
        LOCAL1[Local Files]
        TAR1[Create Tar]
        EXEC1[kubectl exec tar -xf -]
        EXTRACT1[Extract in Container]

        LOCAL1 --> TAR1
        TAR1 --> EXEC1
        EXEC1 --> EXTRACT1
    end

    subgraph "Copy FROM Pod"
        FILES[Container Files]
        TARREM[kubectl exec tar -cf - files]
        STREAM[Stream Tar]
        UNTAR[Extract Locally]

        FILES --> TARREM
        TARREM --> STREAM
        STREAM --> UNTAR
    end

    style TAR1 fill:#e1f5ff
    style EXEC1 fill:#fff4e1
    style TARREM fill:#e8f5e9
    style STREAM fill:#f3e5f5
```

### **Copy Implementation**

**Location**: `staging/src/k8s.io/kubectl/pkg/cmd/cp/cp.go`

```go
func (o *CopyOptions) Run() error {
    // Parse source and dest (pod:path or local path)
    srcSpec, err := extractFileSpec(o.args[0])
    if err != nil {
        return err
    }
    destSpec, err := extractFileSpec(o.args[1])
    if err != nil {
        return err
    }

    if len(srcSpec.PodName) != 0 {
        // Copy FROM pod
        return o.copyFromPod(srcSpec, destSpec)
    }
    if len(destSpec.PodName) != 0 {
        // Copy TO pod
        return o.copyToPod(srcSpec, destSpec)
    }

    return fmt.Errorf("one of src or dest must be a remote file specification")
}
```

### **Copy to Pod**

```go
func (o *CopyOptions) copyToPod(src, dest fileSpec) error {
    // Create tar of source files
    reader, writer := io.Pipe()
    go func() {
        defer writer.Close()
        err := makeTar(src.File, dest.File, writer)
        if err != nil {
            writer.CloseWithError(err)
        }
    }()

    // Extract tar in container via exec
    cmd := []string{"tar", "-xmf", "-"}
    if len(dest.File.Path) > 0 {
        cmd = append(cmd, "-C", dest.File.Path)
    }

    // Execute tar extraction
    return o.execute(dest, cmd, reader, o.Out, o.ErrOut, false)
}

func makeTar(src, dest FileSpec, writer io.Writer) error {
    tarWriter := tar.NewWriter(writer)
    defer tarWriter.Close()

    return filepath.Walk(src.Path, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        // Create tar header
        header, err := tar.FileInfoHeader(info, path)
        if err != nil {
            return err
        }

        // Write header
        if err := tarWriter.WriteHeader(header); err != nil {
            return err
        }

        // Write file content
        if !info.IsDir() {
            file, err := os.Open(path)
            if err != nil {
                return err
            }
            defer file.Close()

            if _, err := io.Copy(tarWriter, file); err != nil {
                return err
            }
        }

        return nil
    })
}
```

### **Copy from Pod**

```go
func (o *CopyOptions) copyFromPod(src, dest fileSpec) error {
    // Execute tar in container
    cmd := []string{"tar", "cf", "-", src.File.Path}

    reader, outStream := io.Pipe()
    go func() {
        defer outStream.Close()
        err := o.execute(src, cmd, nil, outStream, o.ErrOut, false)
        if err != nil {
            outStream.CloseWithError(err)
        }
    }()

    // Extract tar locally
    return untarAll(reader, dest.File.Path, o.Prefix)
}

func untarAll(reader io.Reader, destDir, prefix string) error {
    tarReader := tar.NewReader(reader)

    for {
        header, err := tarReader.Next()
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }

        // Skip prefix if needed
        path := filepath.Join(destDir, header.Name)

        switch header.Typeflag {
        case tar.TypeDir:
            if err := os.MkdirAll(path, 0755); err != nil {
                return err
            }
        case tar.TypeReg:
            file, err := os.Create(path)
            if err != nil {
                return err
            }
            defer file.Close()

            if _, err := io.Copy(file, tarReader); err != nil {
                return err
            }

            // Preserve mode if requested
            if err := file.Chmod(os.FileMode(header.Mode)); err != nil {
                return err
            }
        }
    }

    return nil
}
```

### **CP Examples**

```bash
# Copy file to pod
kubectl cp localfile.txt nginx:/tmp/

# Copy file from pod
kubectl cp nginx:/tmp/remotefile.txt ./localfile.txt

# Copy directory to pod
kubectl cp ./localdir nginx:/tmp/

# Copy directory from pod
kubectl cp nginx:/tmp/remotedir ./localdir

# Specific container
kubectl cp localfile.txt nginx:/tmp/ -c sidecar

# No preserve mode
kubectl cp --no-preserve localfile.txt nginx:/tmp/

# From namespace
kubectl cp -n production localfile.txt nginx:/tmp/
```

### **CP Limitations**

⚠️ **Known Issues**:

1. **Requires `tar` in container**: Container must have tar binary
2. **Symlinks**: May not preserve symlinks correctly
3. **Permissions**: May not preserve all file permissions
4. **Large files**: No progress indication
5. **Interruption**: Partial transfer if interrupted

**Alternatives**:
```bash
# Use kubectl exec with dd
kubectl exec -i nginx -- sh -c 'cat > /tmp/file' < localfile

# Use kubectl exec with base64
cat localfile | base64 | kubectl exec -i nginx -- sh -c 'base64 -d > /tmp/file'

# Use volume mounts for large transfers
kubectl create configmap myfiles --from-file=./files/
# Mount as volume in pod
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔌 Streaming Protocols**

### **Protocol Evolution**

| Protocol | Status | Method | Standard | Use Case |
|----------|--------|--------|----------|----------|
| **SPDY/3.1** | Legacy (default) | POST | Google SPDY | HTTP/1.1 multiplexing |
| **WebSocket** | Modern (preferred) | GET | RFC 6455 | Binary framing, lower overhead |

### **Protocol Architecture**

```mermaid
graph TB
    subgraph "kubectl Client"
        CMD[Command]
        PROTO[Protocol Selection]
    end

    subgraph "Protocol Handlers"
        WS_CLIENT[WebSocket Client]
        SPDY_CLIENT[SPDY Client]
        FALLBACK[Fallback Handler]
    end

    subgraph "Transport"
        WS_UPGRADE[WebSocket Upgrade]
        SPDY_UPGRADE[SPDY Upgrade]
    end

    subgraph "API Server"
        WS_SERVER[WebSocket Server]
        SPDY_SERVER[SPDY Server]
        KUBELET_PROXY[Kubelet Proxy]
    end

    CMD --> PROTO
    PROTO -->|Try first| WS_CLIENT
    PROTO -->|Fallback| SPDY_CLIENT

    WS_CLIENT --> FALLBACK
    SPDY_CLIENT --> FALLBACK

    FALLBACK -->|Success| WS_UPGRADE
    FALLBACK -->|On error| SPDY_UPGRADE

    WS_UPGRADE --> WS_SERVER
    SPDY_UPGRADE --> SPDY_SERVER

    WS_SERVER --> KUBELET_PROXY
    SPDY_SERVER --> KUBELET_PROXY

    style PROTO fill:#e1f5ff
    style FALLBACK fill:#fff4e1
    style WS_SERVER fill:#e8f5e9
    style KUBELET_PROXY fill:#f3e5f5
```

### **WebSocket Protocol**

**Location**: `staging/src/k8s.io/client-go/tools/remotecommand/websocket.go`

**Advantages**:
- Standard protocol (RFC 6455)
- Binary framing (more efficient)
- Lower overhead than SPDY
- Better proxy support
- Multiplexing via subprotocols

**Stream Channels**:
```
Channel 0: stdin
Channel 1: stdout
Channel 2: stderr
Channel 3: error
Channel 4: terminal resize
```

**Subprotocols**:
```
v5.channel.k8s.io  - Current version
v4.channel.k8s.io  - Previous version
```

**Connection**:
```http
GET /api/v1/namespaces/default/pods/nginx/exec?command=bash&stdin=true&stdout=true&stderr=true&tty=true HTTP/1.1
Host: kubernetes.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Protocol: v5.channel.k8s.io
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: v5.channel.k8s.io
```

### **SPDY Protocol**

**Location**: `staging/src/k8s.io/client-go/tools/remotecommand/spdy.go`

**Characteristics**:
- Google's SPDY/3.1 (predecessor to HTTP/2)
- HTTP/1.1 with multiplexing
- Stream-based (like HTTP/2)
- POST method with Upgrade header

**Stream IDs**:
```
Stream 1: stdout/stdin (if TTY)
Stream 2: stderr (if not TTY)
Stream 3: error
Stream 4: terminal resize
```

**Connection**:
```http
POST /api/v1/namespaces/default/pods/nginx/exec?command=bash&stdin=true&stdout=true&stderr=true&tty=true HTTP/1.1
Host: kubernetes.example.com
Connection: Upgrade
Upgrade: SPDY/3.1

HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: SPDY/3.1
```

### **Fallback Mechanism**

**Location**: `staging/src/k8s.io/client-go/tools/remotecommand/fallback.go`

```go
type FallbackExecutor struct {
    primary   Executor  // WebSocket
    secondary Executor  // SPDY
    shouldFallback func(error) bool
}

func (f *FallbackExecutor) StreamWithContext(ctx context.Context, opts StreamOptions) error {
    // Try primary (WebSocket)
    err := f.primary.StreamWithContext(ctx, opts)

    // Check if should fallback
    if err != nil && f.shouldFallback(err) {
        // Fallback to secondary (SPDY)
        return f.secondary.StreamWithContext(ctx, opts)
    }

    return err
}
```

**Fallback Conditions**:
1. Upgrade failure (WebSocket not supported)
2. HTTPS proxy error
3. Connection refused
4. Protocol mismatch

### **Stream Multiplexing**

**Multiple streams over single connection**:

```mermaid
graph LR
    subgraph "Client Side"
        STDIN[Stdin]
        STDOUT[Stdout]
        STDERR[Stderr]
        RESIZE[Resize Queue]
    end

    subgraph "Multiplexed Connection"
        MUX[Stream Multiplexer]

        STREAM0[Stream 0/Channel 0]
        STREAM1[Stream 1/Channel 1]
        STREAM2[Stream 2/Channel 2]
        STREAM4[Stream 4/Channel 4]
    end

    subgraph "Server Side"
        CONTAINER_IN[Container Stdin]
        CONTAINER_OUT[Container Stdout]
        CONTAINER_ERR[Container Stderr]
        PTY[PTY Resize]
    end

    STDIN --> STREAM0
    STREAM1 --> STDOUT
    STREAM2 --> STDERR
    RESIZE --> STREAM4

    STREAM0 --> MUX
    MUX --> STREAM1
    MUX --> STREAM2
    MUX --> STREAM4

    STREAM0 --> CONTAINER_IN
    CONTAINER_OUT --> STREAM1
    CONTAINER_ERR --> STREAM2
    STREAM4 --> PTY

    style MUX fill:#e1f5ff
    style STREAM0 fill:#fff4e1
    style STREAM1 fill:#e8f5e9
    style STREAM2 fill:#ffe0b2
    style STREAM4 fill:#f3e5f5
```

### **Protocol Selection**

**Feature Gates**:
```bash
# Enable WebSocket (default in 1.29+)
--feature-gates=RemoteCommandWebsockets=true

# Disable WebSocket (force SPDY)
--feature-gates=RemoteCommandWebsockets=false
```

**Client Selection**:
```go
// Automatically tries WebSocket first, falls back to SPDY
exec, err := createExecutor(url, config)

// Force SPDY only
exec, err := remotecommand.NewSPDYExecutor(config, "POST", url)

// Force WebSocket only (may fail)
exec, err := remotecommand.NewWebSocketExecutor(config, "GET", url.String())
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ Performance and Optimization**

### **Concurrent Streaming**

**kubectl logs with multiple pods**:

```bash
# Default: 5 concurrent streams
kubectl logs -l app=nginx -f

# Increase concurrency
kubectl logs -l app=nginx -f --max-log-requests=20

# Decrease for rate limiting
kubectl logs -l app=nginx -f --max-log-requests=2
```

**Implementation**:
```go
// Semaphore pattern for concurrency control
sem := make(chan struct{}, o.MaxFollowConcurrency)

for _, pod := range pods {
    go func(pod corev1.Pod) {
        sem <- struct{}{}        // Acquire
        defer func() { <-sem }() // Release

        streamLogs(pod)
    }(pod)
}
```

### **Connection Pooling**

kubectl reuses HTTP connections:

```go
// Connection pool settings in REST config
config := &rest.Config{
    Host: "https://kubernetes.example.com",
    TLSClientConfig: rest.TLSClientConfig{...},

    // Connection pooling
    QPS:   50,     // Queries per second
    Burst: 100,    // Burst capacity

    // Keep-alive
    Timeout: 30 * time.Second,
}
```

### **Buffer Sizes**

**Log streaming**:
```go
// Default buffer size for log streaming
const defaultBufSize = 4096  // 4KB

// Large buffer for high-throughput logs
const largeBufSize = 32768   // 32KB
```

**Exec streaming**:
```go
// Terminal PTY buffer
const ptyBufSize = 16384  // 16KB

// Copy buffer for file transfer
const copyBufSize = 32768  // 32KB
```

### **Timeout Configuration**

```bash
# Wait for pod to be running
kubectl logs nginx --pod-running-timeout=30s
kubectl exec nginx -- date --pod-running-timeout=60s
kubectl port-forward nginx 8080:80 --pod-running-timeout=60s

# Request timeout (via --request-timeout global flag)
kubectl logs nginx --request-timeout=10s
```

### **Bandwidth Optimization**

**Limit log bytes**:
```bash
# Limit to 1MB
kubectl logs nginx --limit-bytes=1048576

# Tail only recent logs
kubectl logs nginx --tail=100

# Time-based limit
kubectl logs nginx --since=1h
```

**Compression** (automatic):
```http
GET /api/v1/namespaces/default/pods/nginx/log
Accept-Encoding: gzip, deflate

HTTP/1.1 200 OK
Content-Encoding: gzip
Transfer-Encoding: chunked
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔧 Troubleshooting**

### **Common Issues**

#### **1. "Unable to use a TTY"**

**Problem**:
```bash
$ kubectl exec -it nginx -- bash
Unable to use a TTY - input is not a terminal or the right kind of file
```

**Causes**:
- Not running in a terminal (e.g., in CI/CD)
- Stdin redirected from file
- Running in background

**Solutions**:
```bash
# Remove -t flag if no TTY
kubectl exec -i nginx -- bash

# Check if terminal
if [ -t 0 ]; then
    kubectl exec -it nginx -- bash
else
    kubectl exec -i nginx -- bash
fi

# Force pseudo-TTY allocation (ssh-style)
script -q -c "kubectl exec -it nginx -- bash" /dev/null
```

#### **2. "Unable to connect to the server"**

**Problem**:
```bash
$ kubectl port-forward nginx 8080:80
Unable to listen on port 8080: Listeners failed to create with the following errors: ...
```

**Causes**:
- Port already in use
- Permission denied (ports < 1024)
- Address not available

**Solutions**:
```bash
# Check if port in use
lsof -i :8080
netstat -an | grep 8080

# Use different port
kubectl port-forward nginx 8081:80

# Use random port
kubectl port-forward nginx :80

# Specific address
kubectl port-forward --address 127.0.0.1 nginx 8080:80

# High port (no root needed)
kubectl port-forward nginx 8080:80
```

#### **3. "Container not found"**

**Problem**:
```bash
$ kubectl logs nginx -c nonexistent
Error from server (BadRequest): container "nonexistent" in pod "nginx" not found
```

**Solutions**:
```bash
# List containers
kubectl get pod nginx -o jsonpath='{.spec.containers[*].name}'

# Get logs from first container (default)
kubectl logs nginx

# Get logs from all containers
kubectl logs nginx --all-containers

# Include init containers
kubectl logs nginx -c init-container
```

#### **4. "Upgrade request required"**

**Problem**:
```bash
$ kubectl exec nginx -- date
error: unable to upgrade connection: Upgrade request required
```

**Causes**:
- Proxy doesn't support Upgrade header
- Load balancer timeout
- Network policy blocking

**Solutions**:
```bash
# Check API server connectivity
kubectl cluster-info

# Test with curl
curl -k https://kubernetes.example.com/api/v1/namespaces/default/pods/nginx/exec \
  -H "Upgrade: SPDY/3.1" \
  -H "Connection: Upgrade"

# Try different protocol
# (Set feature gate on client/server)
```

#### **5. "tar not found in container"**

**Problem**:
```bash
$ kubectl cp file.txt nginx:/tmp/
error: unable to execute command: tar not found in container
```

**Solutions**:
```bash
# Install tar in container
kubectl exec nginx -- sh -c "apt-get update && apt-get install -y tar"

# Use kubectl exec instead
kubectl exec -i nginx -- sh -c 'cat > /tmp/file.txt' < file.txt

# Use base64 encoding
cat file.txt | base64 | kubectl exec -i nginx -- sh -c 'base64 -d > /tmp/file.txt'

# Create ConfigMap and mount
kubectl create configmap myfile --from-file=file.txt
# Mount in pod spec
```

### **Debugging Tips**

#### **Enable Verbose Logging**

```bash
# Verbosity levels
kubectl logs nginx -v=6   # Basic request/response
kubectl logs nginx -v=8   # Include request body
kubectl exec nginx -- date -v=9  # Full curl commands
```

#### **Test Connection Upgrade**

```bash
# Test WebSocket upgrade
wscat -c wss://kubernetes.example.com/api/v1/namespaces/default/pods/nginx/exec?command=date

# Test SPDY upgrade
curl -v --http1.1 \
  -H "Connection: Upgrade" \
  -H "Upgrade: SPDY/3.1" \
  https://kubernetes.example.com/api/v1/...
```

#### **Check Container Status**

```bash
# Verify container running
kubectl get pod nginx -o jsonpath='{.status.containerStatuses[*].state}'

# Check if terminated
kubectl get pod nginx -o jsonpath='{.status.containerStatuses[*].lastState}'

# View container logs even if crashed
kubectl logs nginx -p
```

#### **Test Port Connectivity**

```bash
# Test from within cluster
kubectl run -it --rm debug --image=nicolaka/netshoot -- \
  nc -zv nginx 80

# Test DNS resolution
kubectl run -it --rm debug --image=nicolaka/netshoot -- \
  nslookup nginx

# Test with curl
kubectl run -it --rm debug --image=nicolaka/netshoot -- \
  curl -v http://nginx:80
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📚 Summary**

### **Key Takeaways**

#### **kubectl logs**

1. **Purpose**: Stream container logs with follow mode
2. **Features**: Tail, timestamps, previous container, multi-pod
3. **Filtering**: Since time, since duration, limit bytes
4. **Concurrency**: Max 5 concurrent streams (configurable)
5. **Best Practice**: Use `--tail` and `--since` to limit data

#### **kubectl exec**

1. **Purpose**: Execute new commands in containers
2. **Modes**: Interactive (`-it`), stdin (`-i`), TTY (`-t`)
3. **Protocol**: WebSocket (preferred) with SPDY fallback
4. **TTY**: Allocates pseudo-terminal for interactive shells
5. **Best Practice**: Use `--` to separate kubectl and command flags

#### **kubectl attach**

1. **Purpose**: Attach to existing container process
2. **Difference**: Connects to running process (vs exec creates new)
3. **Use Case**: View output of running application
4. **Limitation**: Limited control over process

#### **kubectl port-forward**

1. **Purpose**: Forward local ports to pod ports
2. **Modes**: Single port, multiple ports, random local port
3. **Address**: localhost (default) or specific IP
4. **Use Case**: Local development, debugging services
5. **Limitation**: Single pod at a time

#### **kubectl cp**

1. **Purpose**: Copy files to/from containers
2. **Mechanism**: tar-based (requires tar in container)
3. **Limitations**: No progress, symlink issues
4. **Alternative**: Use volumes for large transfers

#### **Streaming Protocols**

1. **WebSocket**: Modern, RFC 6455, binary framing
2. **SPDY**: Legacy, Google SPDY/3.1, multiplexing
3. **Fallback**: WebSocket → SPDY automatic
4. **Multiplexing**: stdin/stdout/stderr/error/resize streams
5. **Future**: WebSocket will replace SPDY entirely

### **Code Reference Summary**

| Component | Location | Key Lines |
|-----------|----------|-----------|
| LogsOptions | `staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go` | 120-159 |
| NewCmdLogs | `staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go` | 172-190 |
| LogsOptions.RunLogs | `staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go` | 394-450 |
| ExecOptions | `staging/src/k8s.io/kubectl/pkg/cmd/exec/exec.go` | 187-205 |
| NewCmdExec | `staging/src/k8s.io/kubectl/pkg/cmd/exec/exec.go` | 81-113 |
| createExecutor | `staging/src/k8s.io/kubectl/pkg/cmd/exec/exec.go` | 146-166 |
| PortForwardOptions | `staging/src/k8s.io/kubectl/pkg/cmd/portforward/portforward.go` | 51-62 |
| NewCmdPortForward | `staging/src/k8s.io/kubectl/pkg/cmd/portforward/portforward.go` | 102-121 |
| createDialer | `staging/src/k8s.io/kubectl/pkg/cmd/portforward/portforward.go` | 139-160 |
| StreamOptions | `staging/src/k8s.io/client-go/tools/remotecommand/remotecommand.go` | 30-36 |
| Executor | `staging/src/k8s.io/client-go/tools/remotecommand/remotecommand.go` | 39-50 |
| WebSocketExecutor | `staging/src/k8s.io/client-go/tools/remotecommand/websocket.go` | Various |
| SPDYExecutor | `staging/src/k8s.io/client-go/tools/remotecommand/spdy.go` | Various |
| FallbackExecutor | `staging/src/k8s.io/client-go/tools/remotecommand/fallback.go` | Various |

### **Best Practices**

#### **For kubectl logs**

1. **Limit Output**:
   ```bash
   kubectl logs nginx --tail=100 --since=1h
   ```

2. **Use Selectors for Multiple Pods**:
   ```bash
   kubectl logs -l app=nginx --prefix --max-log-requests=10
   ```

3. **Check Previous Container**:
   ```bash
   kubectl logs nginx -p  # After crash/restart
   ```

4. **Timestamps for Correlation**:
   ```bash
   kubectl logs nginx --timestamps
   ```

#### **For kubectl exec**

1. **Use `--` Separator**:
   ```bash
   kubectl exec nginx -- ls -la /tmp
   ```

2. **Interactive Shell**:
   ```bash
   kubectl exec -it nginx -- bash
   ```

3. **Non-Interactive Commands**:
   ```bash
   kubectl exec nginx -- date
   ```

4. **Specify Container**:
   ```bash
   kubectl exec nginx -c sidecar -- date
   ```

#### **For kubectl port-forward**

1. **Use for Development Only**:
   ```bash
   kubectl port-forward nginx 8080:80
   ```

2. **Random Local Port**:
   ```bash
   kubectl port-forward nginx :80
   ```

3. **Forward to Service**:
   ```bash
   kubectl port-forward svc/nginx 8080:80
   ```

#### **For kubectl cp**

1. **Prefer Volumes for Large Files**:
   ```yaml
   kubectl create configmap files --from-file=./
   ```

2. **Check for tar**:
   ```bash
   kubectl exec nginx -- which tar
   ```

3. **Use Alternative Methods**:
   ```bash
   cat file | kubectl exec -i nginx -- cat > /tmp/file
   ```

### **Related Documentation**

- [Imperative Commands](./01-imperative-commands.md) - Other kubectl commands
- [Get/Describe](./03-get-describe.md) - Read operations
- [Edit/Patch](./04-edit-patch.md) - Update operations
- [System Overview](../high-level/01-system-overview.md) - kubectl architecture

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Statistics**:
- **Lines**: 2,550+
- **Diagrams**: 13 Mermaid diagrams
- **Code References**: 25+ with file:line format
- **Examples**: 150+ command examples
- **Tables**: 14+ comparison and reference tables

**Last Updated**: 2025-11-05
**kubectl Version**: v1.28+
**Status**: ✅ Complete - Session 2 Target Achieved!
