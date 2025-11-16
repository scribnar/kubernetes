# **Leader Election Patterns in Kubernetes**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **Overview**

Leader election is a fundamental distributed systems pattern that ensures only one instance of a replicated service performs certain operations at any given time. In Kubernetes, leader election is critical for controller-manager, scheduler, and other control plane components that run with multiple replicas for high availability.

### **Key Concepts**

- **Leader**: The single active instance that performs write operations
- **Candidate**: Instances attempting to become the leader
- **Lease**: A time-bound lock that represents leadership
- **ResourceLock**: Kubernetes object used to store lease information
- **Clock Skew Tolerance**: Handling time differences between nodes

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **1. Theory: Lease-Based Leader Election**

### **1.1 Distributed Leader Election Fundamentals**

Leader election solves the problem of coordination in distributed systems where multiple instances need to agree on which instance should perform certain actions. The lease-based approach provides:

1. **Mutual Exclusion**: Only one leader at a time
2. **Liveness**: A new leader is elected if the current leader fails
3. **Safety**: No split-brain scenarios where multiple leaders exist

### **1.2 Lease Mechanics**

A lease is a time-limited lock with three critical properties:

```
┌─────────────────────────────────────────────┐
│           Lease Timeline                     │
├─────────────────────────────────────────────┤
│                                              │
│  Acquire ──────> Renew ──────> Renew ───>  │
│    t0             t1            t2           │
│                                              │
│  LeaseDuration: 15s                         │
│  RenewDeadline: 10s                         │
│  RetryPeriod: 2s                            │
└─────────────────────────────────────────────┘
```

**LeaseDuration**: How long a lease is valid without renewal
**RenewDeadline**: Maximum time leader has to renew before giving up
**RetryPeriod**: How often to attempt acquisition/renewal

### **1.3 Clock Skew Tolerance**

Kubernetes leader election is tolerant to arbitrary clock skew but not arbitrary clock skew *rate*. This is achieved by:

1. Using local timestamps only for decision-making
2. Observing changes in remote timestamps (not their absolute values)
3. Configuring LeaseDuration/RenewDeadline ratio appropriately

**Clock Skew Tolerance Formula**:
```
Tolerance Ratio ≈ LeaseDuration / RenewDeadline

Example:
- LeaseDuration = 60s
- RenewDeadline = 30s
- Tolerates nodes running 2x faster/slower than each other
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **2. Kubernetes Implementation**

### **2.1 Architecture Overview**

```mermaid
graph TB
    subgraph "Leader Election Components"
        LE[LeaderElector]
        RL[ResourceLock Interface]
        LL[LeaseLock]
        API[API Server]
    end

    subgraph "Control Plane Components"
        CM1[Controller Manager 1]
        CM2[Controller Manager 2]
        CM3[Controller Manager 3]
        S1[Scheduler 1]
        S2[Scheduler 2]
    end

    CM1 --> LE
    CM2 --> LE
    CM3 --> LE
    S1 --> LE
    S2 --> LE

    LE --> RL
    RL --> LL
    LL --> API

    API --> etcd[(etcd)]

    style CM1 fill:#90EE90
    style CM2 fill:#FFB6C1
    style CM3 fill:#FFB6C1
    style S1 fill:#90EE90
    style S2 fill:#FFB6C1
```

### **2.2 Core Implementation Location**

Primary implementation:
```
/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go
```

Key components:
- `LeaderElector`: Main election logic
- `LeaderElectionConfig`: Configuration structure
- `LeaderCallbacks`: Lifecycle hooks
- `ResourceLock`: Storage abstraction

### **2.3 LeaderElector Structure**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:187-206`

```go
type LeaderElector struct {
    config LeaderElectionConfig

    // internal bookkeeping
    observedRecord    rl.LeaderElectionRecord
    observedRawRecord []byte
    observedTime      time.Time

    // used to implement OnNewLeader(), may lag slightly from the
    // value observedRecord.HolderIdentity if the transition has
    // not yet been reported.
    reportedLeader string

    // clock is wrapper around time to allow for less flaky testing
    clock clock.Clock

    // used to lock the observedRecord
    observedRecordLock sync.Mutex

    metrics leaderMetricsAdapter
}
```

**Key Fields**:
- `observedRecord`: Last seen lease state
- `observedTime`: Local timestamp when record was observed
- `observedRecordLock`: Protects concurrent access to observed state
- `clock`: Abstraction for testing and clock skew handling

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **3. ResourceLock Types**

### **3.1 Evolution of ResourceLocks**

```mermaid
graph LR
    A[Endpoints Lock<br/>v1.0-1.13] --> B[ConfigMap Lock<br/>v1.5-1.13]
    B --> C[Lease Lock<br/>v1.14+]

    style A fill:#FFB6C1,stroke:#FF0000
    style B fill:#FFB6C1,stroke:#FF0000
    style C fill:#90EE90,stroke:#00FF00
```

**Historical ResourceLocks** (REMOVED):
- `endpoints`: Used Endpoints objects (removed in v1.27)
- `configmaps`: Used ConfigMap objects (removed in v1.27)
- `endpointsleases`: Migration lock (removed)
- `configmapsleases`: Migration lock (removed)

**Current ResourceLock** (v1.14+):
- `leases`: Uses Lease objects from coordination.k8s.io/v1

### **3.2 LeaseLock Implementation**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/resourcelock/leaselock.go`

```go
type LeaseLock struct {
    // LeaseMeta holds the object metadata for the Lease
    LeaseMeta metav1.ObjectMeta

    // Client is the coordination client
    Client coordinationv1.CoordinationV1Interface

    // LockConfig holds the lock configuration
    LockConfig ResourceLockConfig

    // Labels to apply to the Lease
    Labels map[string]string
}
```

### **3.3 ResourceLock Interface**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/resourcelock/interface.go:81-100`

```go
type Interface interface {
    // Get returns the LeaderElectionRecord
    Get(ctx context.Context) (*LeaderElectionRecord, []byte, error)

    // Create attempts to create a LeaderElectionRecord
    Create(ctx context.Context, ler LeaderElectionRecord) error

    // Update will update and existing LeaderElectionRecord
    Update(ctx context.Context, ler LeaderElectionRecord) error

    // RecordEvent is used to record events
    RecordEvent(string)

    // Identity will return the locks Identity
    Identity() string

    // Describe is used to convert details on current resource lock
    // into a string
    Describe() string
}
```

### **3.4 LeaderElectionRecord Structure**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/resourcelock/interface.go:46-59`

```go
type LeaderElectionRecord struct {
    // HolderIdentity is the ID that owns the lease
    HolderIdentity       string                      `json:"holderIdentity"`
    LeaseDurationSeconds int                         `json:"leaseDurationSeconds"`
    AcquireTime          metav1.Time                 `json:"acquireTime"`
    RenewTime            metav1.Time                 `json:"renewTime"`
    LeaderTransitions    int                         `json:"leaderTransitions"`
    Strategy             v1.CoordinatedLeaseStrategy `json:"strategy"`
    PreferredHolder      string                      `json:"preferredHolder"`
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **4. Leader Election Lifecycle**

### **4.1 Complete Election Flow**

```mermaid
stateDiagram-v2
    [*] --> Candidate

    Candidate --> Acquiring: Start Election
    Acquiring --> Acquiring: Retry on Failure
    Acquiring --> Leader: Acquired Lease
    Acquiring --> Candidate: Context Cancelled

    Leader --> Renewing: Start Renewal Loop
    Renewing --> Renewing: Successful Renewal
    Renewing --> Leader: Continue Leading
    Renewing --> Releasing: Renewal Failed
    Renewing --> Releasing: Context Cancelled

    Releasing --> Candidate: Released Lease
    Releasing --> [*]: Exit

    Leader --> [*]: Fatal Error
```

### **4.2 Acquire Phase**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:252-276`

```go
func (le *LeaderElector) acquire(ctx context.Context) bool {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    succeeded := false
    desc := le.config.Lock.Describe()
    logger := klog.FromContext(ctx)
    logger.Info("Attempting to acquire leader lease...", "lock", desc)

    wait.JitterUntil(func() {
        if !le.config.Coordinated {
            succeeded = le.tryAcquireOrRenew(ctx)
        } else {
            succeeded = le.tryCoordinatedRenew(ctx)
        }
        le.maybeReportTransition()
        if !succeeded {
            logger.V(4).Info("Failed to acquire lease", "lock", desc)
            return
        }
        le.config.Lock.RecordEvent("became leader")
        le.metrics.leaderOn(le.config.Name)
        logger.Info("Successfully acquired lease", "lock", desc)
        cancel()
    }, le.config.RetryPeriod, JitterFactor, true, ctx.Done())

    return succeeded
}
```

**Key Points**:
1. Uses `wait.JitterUntil` for retries with jitter
2. Jitter factor of 1.2 prevents thundering herd
3. Calls `tryAcquireOrRenew` repeatedly until success
4. Records event and metrics on successful acquisition
5. Cancels context to exit retry loop

### **4.3 TryAcquireOrRenew Logic**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:423-494`

```go
func (le *LeaderElector) tryAcquireOrRenew(ctx context.Context) bool {
    logger := klog.FromContext(ctx)
    now := metav1.NewTime(le.clock.Now())
    leaderElectionRecord := rl.LeaderElectionRecord{
        HolderIdentity:       le.config.Lock.Identity(),
        LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
        RenewTime:            now,
        AcquireTime:          now,
    }

    // 1. FAST PATH: optimistic update for current leader
    if le.IsLeader() && le.isLeaseValid(now.Time) {
        oldObservedRecord := le.getObservedRecord()
        leaderElectionRecord.AcquireTime = oldObservedRecord.AcquireTime
        leaderElectionRecord.LeaderTransitions = oldObservedRecord.LeaderTransitions

        err := le.config.Lock.Update(ctx, leaderElectionRecord)
        if err == nil {
            le.setObservedRecord(&leaderElectionRecord)
            return true
        }
        logger.Error(err, "Failed to update lease optimistically")
    }

    // 2. SLOW PATH: fetch current record
    oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get(ctx)
    if err != nil {
        if !errors.IsNotFound(err) {
            logger.Error(err, "Error retrieving lease lock")
            return false
        }
        // Lease doesn't exist, create it
        if err = le.config.Lock.Create(ctx, leaderElectionRecord); err != nil {
            logger.Error(err, "Error initially creating lease lock")
            return false
        }
        le.setObservedRecord(&leaderElectionRecord)
        return true
    }

    // 3. Update observed record if changed
    if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
        le.setObservedRecord(oldLeaderElectionRecord)
        le.observedRawRecord = oldLeaderElectionRawRecord
    }

    // 4. Check if lease is held by someone else and still valid
    if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
       le.isLeaseValid(now.Time) && !le.IsLeader() {
        logger.V(4).Info("Lease is held by and has not yet expired",
            "holder", oldLeaderElectionRecord.HolderIdentity)
        return false
    }

    // 5. Attempt to acquire or renew
    if le.IsLeader() {
        leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
        leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
        le.metrics.slowpathExercised(le.config.Name)
    } else {
        leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
    }

    if err = le.config.Lock.Update(ctx, leaderElectionRecord); err != nil {
        logger.Error(err, "Failed to update lease")
        return false
    }

    le.setObservedRecord(&leaderElectionRecord)
    return true
}
```

**Two-Path Strategy**:

1. **Fast Path** (lines 435-446):
   - For current leader with valid lease
   - Optimistic update without fetching
   - Falls back to slow path on conflict

2. **Slow Path** (lines 448-493):
   - Fetch current lease state
   - Check if lease exists, create if not
   - Check if lease is valid
   - Update lease with new timestamp

### **4.4 Renew Phase**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:278-307`

```go
func (le *LeaderElector) renew(ctx context.Context) {
    defer le.config.Lock.RecordEvent("stopped leading")
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    logger := klog.FromContext(ctx)

    wait.Until(func() {
        err := wait.PollUntilContextTimeout(ctx,
            le.config.RetryPeriod,
            le.config.RenewDeadline,
            true,
            func(ctx context.Context) (done bool, err error) {
                if !le.config.Coordinated {
                    return le.tryAcquireOrRenew(ctx), nil
                } else {
                    return le.tryCoordinatedRenew(ctx), nil
                }
            })

        le.maybeReportTransition()
        desc := le.config.Lock.Describe()

        if err == nil {
            logger.V(5).Info("Successfully renewed lease", "lock", desc)
            return
        }

        le.metrics.leaderOff(le.config.Name)
        logger.Info("Failed to renew lease", "lock", desc, "err", err)
        cancel()
    }, le.config.RetryPeriod, ctx.Done())

    // Release lease if configured
    if le.config.ReleaseOnCancel {
        le.release(logger)
    }
}
```

**Renewal Logic**:
1. Polls every `RetryPeriod`
2. Must succeed within `RenewDeadline`
3. Uses `wait.PollUntilContextTimeout` for bounded retries
4. Cancels on failure (leader steps down)
5. Optionally releases lease on exit

### **4.5 Release Phase**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:309-342`

```go
func (le *LeaderElector) release(logger klog.Logger) bool {
    ctx := context.Background()
    timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
    defer timeoutCancel()

    // Fetch current lease
    oldLeaderElectionRecord, _, err := le.config.Lock.Get(timeoutCtx)
    if err != nil {
        if !errors.IsNotFound(err) {
            logger.Error(err, "error retrieving resource lock")
            return false
        }
        logger.Info("lease lock not found")
        return false
    }

    // Verify we're still the leader
    if !le.IsLeader() {
        return true
    }

    // Release by setting LeaseDurationSeconds to 1
    now := metav1.NewTime(le.clock.Now())
    leaderElectionRecord := rl.LeaderElectionRecord{
        LeaderTransitions:    oldLeaderElectionRecord.LeaderTransitions,
        LeaseDurationSeconds: 1,  // Expire immediately
        RenewTime:            now,
        AcquireTime:          now,
    }

    if err := le.config.Lock.Update(timeoutCtx, leaderElectionRecord); err != nil {
        logger.Error(err, "Failed to release lease")
        return false
    }

    le.setObservedRecord(&leaderElectionRecord)
    return true
}
```

**Release Mechanism**:
- Sets `LeaseDurationSeconds` to 1 (expires immediately)
- Allows new leader to acquire quickly
- Only releases if still the leader
- Uses RenewDeadline as timeout

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **5. Configuration and Timing Parameters**

### **5.1 LeaderElectionConfig Structure**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:116-166`

```go
type LeaderElectionConfig struct {
    // Lock is the resource that will be used for locking
    Lock rl.Interface

    // LeaseDuration is the duration that non-leader candidates will
    // wait to force acquire leadership. This is measured against time of
    // last observed ack.
    //
    // Core clients default this value to 15 seconds.
    LeaseDuration time.Duration

    // RenewDeadline is the duration that the acting master will retry
    // refreshing leadership before giving up.
    //
    // Core clients default this value to 10 seconds.
    RenewDeadline time.Duration

    // RetryPeriod is the duration the LeaderElector clients should wait
    // between tries of actions.
    //
    // Core clients default this value to 2 seconds.
    RetryPeriod time.Duration

    // Callbacks are callbacks that are triggered during certain lifecycle
    // events of the LeaderElector
    Callbacks LeaderCallbacks

    // WatchDog is the associated health checker
    WatchDog *HealthzAdaptor

    // ReleaseOnCancel should be set true if the lock should be released
    // when the run context is cancelled.
    ReleaseOnCancel bool

    // Name is the name of the resource lock for debugging
    Name string

    // Coordinated will use the Coordinated Leader Election feature
    // WARNING: Coordinated leader election is ALPHA.
    Coordinated bool
}
```

### **5.2 Default Values in Core Components**

**Controller Manager Defaults**:
```go
// Default values used in kube-controller-manager
LeaseDuration: 15 * time.Second
RenewDeadline: 10 * time.Second
RetryPeriod:   2 * time.Second
```

**Scheduler Defaults**:
```go
// Default values used in kube-scheduler
LeaseDuration: 15 * time.Second
RenewDeadline: 10 * time.Second
RetryPeriod:   2 * time.Second
```

### **5.3 Validation Constraints**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:76-98`

```go
func NewLeaderElector(lec LeaderElectionConfig) (*LeaderElector, error) {
    // Constraint 1: LeaseDuration > RenewDeadline
    if lec.LeaseDuration <= lec.RenewDeadline {
        return nil, fmt.Errorf("leaseDuration must be greater than renewDeadline")
    }

    // Constraint 2: RenewDeadline > RetryPeriod * JitterFactor
    if lec.RenewDeadline <= time.Duration(JitterFactor*float64(lec.RetryPeriod)) {
        return nil, fmt.Errorf("renewDeadline must be greater than retryPeriod*JitterFactor")
    }

    // Constraint 3: All values must be positive
    if lec.LeaseDuration < 1 {
        return nil, fmt.Errorf("leaseDuration must be greater than zero")
    }
    if lec.RenewDeadline < 1 {
        return nil, fmt.Errorf("renewDeadline must be greater than zero")
    }
    if lec.RetryPeriod < 1 {
        return nil, fmt.Errorf("retryPeriod must be greater than zero")
    }

    // ... additional validation
}
```

### **5.4 Timing Relationships**

```mermaid
graph TD
    A[Time Parameters] --> B[LeaseDuration: 15s]
    A --> C[RenewDeadline: 10s]
    A --> D[RetryPeriod: 2s]
    A --> E[JitterFactor: 1.2]

    B --> F[Must be > RenewDeadline]
    C --> G[Must be > RetryPeriod * 1.2]
    D --> H[Controls retry frequency]
    E --> I[Prevents thundering herd]

    F --> J[Ensures candidates wait<br/>before force-acquiring]
    G --> K[Ensures leader has time<br/>for multiple retries]
    H --> L[Balance: too fast = API load<br/>too slow = slow failover]
    I --> M[Adds randomness to<br/>retry timing]
```

### **5.5 Tuning Guidelines**

**For High Availability (Fast Failover)**:
```go
LeaseDuration: 10 * time.Second   // Faster detection
RenewDeadline: 6 * time.Second    // Less time to renew
RetryPeriod:   1 * time.Second    // More frequent checks
```
**Trade-offs**: Higher API server load, more network traffic

**For Large Clusters (Stability)**:
```go
LeaseDuration: 30 * time.Second   // Longer grace period
RenewDeadline: 20 * time.Second   // More time for renewal
RetryPeriod:   5 * time.Second    // Less frequent checks
```
**Trade-offs**: Slower failover, lower API server load

**For API Latency Tolerance**:
```go
LeaseDuration: 60 * time.Second   // Very long lease
RenewDeadline: 40 * time.Second   // Lots of retry time
RetryPeriod:   10 * time.Second   // Infrequent retries
```
**Trade-offs**: Very slow failover, minimal API impact

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **6. Callbacks and Lifecycle Hooks**

### **6.1 LeaderCallbacks Structure**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:173-185`

```go
type LeaderCallbacks struct {
    // OnStartedLeading is called when a LeaderElector client starts leading
    OnStartedLeading func(context.Context)

    // OnStoppedLeading is called when a LeaderElector client stops leading.
    // This callback is always called when the LeaderElector exits, even if it did not start leading.
    OnStoppedLeading func()

    // OnNewLeader is called when the client observes a leader that is
    // not the previously observed leader. This includes the first observed
    // leader when the client starts.
    OnNewLeader func(identity string)
}
```

### **6.2 Callback Execution Flow**

```mermaid
sequenceDiagram
    participant LE as LeaderElector
    participant CB as Callbacks
    participant App as Application

    LE->>LE: acquire() succeeds
    LE->>CB: OnNewLeader(self)
    LE->>CB: OnStartedLeading(ctx)
    CB->>App: Start controller logic

    Note over App: Running as leader...

    LE->>LE: renew() fails
    LE->>CB: OnStoppedLeading()
    CB->>App: Stop controller logic
    App->>App: Cleanup and exit
```

### **6.3 Run Method with Callbacks**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:211-222`

```go
func (le *LeaderElector) Run(ctx context.Context) {
    defer runtime.HandleCrashWithContext(ctx)
    defer le.config.Callbacks.OnStoppedLeading()  // ALWAYS called

    if !le.acquire(ctx) {
        return // ctx signalled done
    }

    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    go le.config.Callbacks.OnStartedLeading(ctx)  // Called in goroutine
    le.renew(ctx)
}
```

**Important Notes**:
1. `OnStoppedLeading` is ALWAYS called via defer
2. `OnStartedLeading` runs in a separate goroutine
3. Application receives context for graceful shutdown
4. `OnNewLeader` may be called multiple times

### **6.4 Example Usage in Controller Manager**

**File**: `/pkg/controlplane/controller/leaderelection/run_with_leaderelection.go`

```go
func RunWithLeaderElection(ctx context.Context, config Config) {
    leaderElectionConfig := leaderelection.LeaderElectionConfig{
        Lock:          lock,
        LeaseDuration: config.LeaseDuration,
        RenewDeadline: config.RenewDeadline,
        RetryPeriod:   config.RetryPeriod,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                logger.Info("Started leading")
                // Start all controllers
                config.Run(ctx)
            },
            OnStoppedLeading: func() {
                logger.Info("Stopped leading")
                // This typically results in process exit
                klog.FlushAndExit(klog.ExitFlushTimeout, 0)
            },
            OnNewLeader: func(identity string) {
                if identity == config.Identity {
                    logger.Info("I am the leader", "identity", identity)
                } else {
                    logger.Info("New leader elected", "identity", identity)
                }
            },
        },
    }

    leaderElector, err := leaderelection.NewLeaderElector(leaderElectionConfig)
    if err != nil {
        logger.Error(err, "Failed to create leader elector")
        return
    }

    leaderElector.Run(ctx)
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **7. Multi-Leader Patterns (Controller Sharding)**

### **7.1 Sharding Architecture**

In some cases, multiple leaders are needed for scalability. This is achieved through sharding:

```mermaid
graph TB
    subgraph "Shard 1: Namespaces A-H"
        L1[Leader 1]
        NS1[Namespaces A-H]
    end

    subgraph "Shard 2: Namespaces I-P"
        L2[Leader 2]
        NS2[Namespaces I-P]
    end

    subgraph "Shard 3: Namespaces Q-Z"
        L3[Leader 3]
        NS3[Namespaces Q-Z]
    end

    L1 --> NS1
    L2 --> NS2
    L3 --> NS3

    L1 -.Lease.-> Lease1[(Lease: shard-1)]
    L2 -.Lease.-> Lease2[(Lease: shard-2)]
    L3 -.Lease.-> Lease3[(Lease: shard-3)]
```

### **7.2 Sharding Implementation Example**

```go
// Example: Shard controllers by namespace hash
func StartShardedControllers(ctx context.Context, shardID int, totalShards int) {
    // Create lease for this shard
    lock := &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("controller-shard-%d", shardID),
            Namespace: "kube-system",
        },
        Client:     coordinationClient,
        LockConfig: resourcelock.ResourceLockConfig{Identity: hostName},
    }

    leaderElectionConfig := leaderelection.LeaderElectionConfig{
        Lock:          lock,
        LeaseDuration: 15 * time.Second,
        RenewDeadline: 10 * time.Second,
        RetryPeriod:   2 * time.Second,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                // Only process namespaces assigned to this shard
                namespaceFilter := func(ns string) bool {
                    hash := hash(ns)
                    return hash % totalShards == shardID
                }

                // Start controllers with filter
                startControllers(ctx, namespaceFilter)
            },
            OnStoppedLeading: func() {
                klog.Info("Lost shard leadership", "shard", shardID)
            },
        },
    }

    le, _ := leaderelection.NewLeaderElector(leaderElectionConfig)
    le.Run(ctx)
}

func hash(s string) int {
    h := fnv.New32a()
    h.Write([]byte(s))
    return int(h.Sum32())
}
```

### **7.3 Coordinated Leader Election (ALPHA)**

**New Feature**: Coordinated leader election allows graceful leadership transitions.

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:347-418`

```go
func (le *LeaderElector) tryCoordinatedRenew(ctx context.Context) bool {
    // ... obtain the electionRecord
    oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get(ctx)

    // Check if lease is marked as "end of term"
    if le.IsLeader() && oldLeaderElectionRecord.PreferredHolder != "" {
        logger.V(4).Info("Lease is marked as 'end of term'")
        // Don't renew - allow preferred holder to take over
        return false
    }

    // ... normal renewal logic
}
```

**PreferredHolder Field**: Indicates the next preferred leader, allowing graceful transitions.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **8. Real-World Examples**

### **8.1 Controller Manager Leader Election**

**File**: `/cmd/kube-controller-manager/app/controllermanager.go`

```go
func Run(ctx context.Context, c *config.CompletedConfig) error {
    // Setup leader election
    leaderElectionConfig, err := makeLeaderElectionConfig(c)
    if err != nil {
        return err
    }

    leaderElectionConfig.Callbacks = leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) {
            logger := klog.FromContext(ctx)
            logger.Info("Starting controllers")

            // Initialize all controllers
            controllerContext, err := CreateControllerContext(c, ctx)
            if err != nil {
                logger.Error(err, "Error creating controller context")
                return
            }

            // Start all controllers
            if err := StartControllers(ctx, controllerContext); err != nil {
                logger.Error(err, "Error starting controllers")
                return
            }

            // Start informers
            controllerContext.InformerFactory.Start(ctx.Done())

            // Wait forever
            <-ctx.Done()
        },
        OnStoppedLeading: func() {
            logger.Error(nil, "Leader election lost")
            klog.FlushAndExit(klog.ExitFlushTimeout, 1)
        },
    }

    // Run leader elector
    leaderElector, err := leaderelection.NewLeaderElector(leaderElectionConfig)
    if err != nil {
        return err
    }

    leaderElector.Run(ctx)
    return nil
}
```

### **8.2 Scheduler Leader Election**

```go
func (cc *CompletedConfig) Run(ctx context.Context) error {
    // Create lease lock
    id, err := os.Hostname()
    if err != nil {
        return err
    }
    id = id + "_" + string(uuid.NewUUID())

    lock := &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{
            Name:      "kube-scheduler",
            Namespace: "kube-system",
        },
        Client:     cc.LeaderElectionClient.CoordinationV1(),
        LockConfig: resourcelock.ResourceLockConfig{Identity: id},
    }

    // Run with leader election
    leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
        Lock:          lock,
        LeaseDuration: cc.ComponentConfig.LeaderElection.LeaseDuration.Duration,
        RenewDeadline: cc.ComponentConfig.LeaderElection.RenewDeadline.Duration,
        RetryPeriod:   cc.ComponentConfig.LeaderElection.RetryPeriod.Duration,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                // Start scheduling
                sched.Run(ctx)
            },
            OnStoppedLeading: func() {
                klog.ErrorS(nil, "Leaderelection lost")
                klog.FlushAndExit(klog.ExitFlushTimeout, 1)
            },
        },
    })

    return nil
}
```

### **8.3 Inspecting Active Leases**

```bash
# List all leases in kube-system
kubectl get leases -n kube-system

# Output:
# NAME                      HOLDER                                       AGE
# kube-controller-manager   master-1_a1b2c3d4-...                        5d
# kube-scheduler            master-2_e5f6g7h8-...                        5d

# Describe a specific lease
kubectl describe lease kube-controller-manager -n kube-system
```

**Example Lease Object**:
```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: "master-1_a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  leaseDurationSeconds: 15
  acquireTime: "2025-01-15T10:00:00Z"
  renewTime: "2025-01-15T10:00:12Z"
  leaderTransitions: 3
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **9. Troubleshooting and Debugging**

### **9.1 Common Issues**

#### **Issue 1: Frequent Leader Transitions**

**Symptoms**:
```
# Check leader transitions in lease
kubectl get lease kube-controller-manager -n kube-system -o jsonpath='{.spec.leaderTransitions}'
# Output: 150  (very high!)
```

**Causes**:
1. Network instability
2. API server latency
3. LeaseDuration too short
4. RenewDeadline too short

**Solutions**:
```go
// Increase timing parameters
LeaseDuration: 30 * time.Second  // was 15s
RenewDeadline: 20 * time.Second  // was 10s
RetryPeriod:   5 * time.Second   // was 2s
```

#### **Issue 2: Split-Brain (Multiple Leaders)**

**Detection**:
```bash
# Check if multiple instances think they're leader
kubectl logs -n kube-system kube-controller-manager-master-1 | grep "leader"
kubectl logs -n kube-system kube-controller-manager-master-2 | grep "leader"

# Both showing "Started leading" - PROBLEM!
```

**This should NOT happen** due to:
1. Optimistic locking via ResourceVersion
2. Atomic API server updates
3. etcd strong consistency

**If it happens**:
- Check etcd cluster health
- Verify API server is not partitioned
- Check for clock skew exceeding tolerance

#### **Issue 3: Leader Not Renewing**

**Symptoms**:
```
Failed to renew lease default/my-lease: context deadline exceeded
```

**Causes**:
1. API server overloaded
2. Network partition
3. etcd slow/unavailable
4. Deadlock in leader code

**Debug Steps**:
```bash
# Check API server latency
kubectl get --raw /metrics | grep apiserver_request_duration

# Check etcd health
ETCDCTL_API=3 etcdctl endpoint health

# Check leader pod health
kubectl get pod -n kube-system kube-controller-manager-master-1
```

### **9.2 Metrics and Monitoring**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/metrics.go`

**Key Metrics**:
```
# Leader election on/off
leader_election_master_status{name="kube-controller-manager"}

# Slowpath exercised (optimistic update failed)
leader_election_slowpath_total{name="kube-controller-manager"}

# Leader transitions
# (from Lease object spec.leaderTransitions)
```

**Monitoring Query Examples**:
```promql
# Alert on frequent transitions
rate(leader_election_master_status{name="kube-controller-manager"}[5m]) > 0.1

# Alert on excessive slowpath usage
rate(leader_election_slowpath_total{name="kube-controller-manager"}[5m]) > 0.5

# Check current leader status
leader_election_master_status{name="kube-controller-manager"} == 1
```

### **9.3 Debugging with Increased Verbosity**

```bash
# Run controller-manager with verbose logging
kube-controller-manager \
  --leader-elect=true \
  --leader-elect-lease-duration=15s \
  --leader-elect-renew-deadline=10s \
  --leader-elect-retry-period=2s \
  --v=4  # Increased verbosity

# Relevant log levels:
# V(4): Lease acquisition failures
# V(5): Successful renewals
# V(6): Detailed lease state
```

**Example Debug Output**:
```
I0115 10:00:00.123456 leaderelection.go:258] Attempting to acquire leader lease...
V(4) Failed to acquire lease kube-system/kube-controller-manager
V(4) Lease is held by master-2_xyz and has not yet expired
I0115 10:00:15.234567 leaderelection.go:272] Successfully acquired lease kube-system/kube-controller-manager
V(5) Successfully renewed lease kube-system/kube-controller-manager
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **10. Best Practices**

### **10.1 Configuration Best Practices**

✅ **DO**:
1. **Use appropriate timing for cluster size**:
   ```go
   // Small cluster (< 100 nodes)
   LeaseDuration: 15s, RenewDeadline: 10s, RetryPeriod: 2s

   // Large cluster (> 1000 nodes)
   LeaseDuration: 30s, RenewDeadline: 20s, RetryPeriod: 5s
   ```

2. **Set unique identities**:
   ```go
   hostname, _ := os.Hostname()
   identity := hostname + "_" + string(uuid.NewUUID())
   ```

3. **Use ReleaseOnCancel for graceful shutdown**:
   ```go
   ReleaseOnCancel: true  // Allows fast failover
   ```

4. **Implement proper OnStoppedLeading**:
   ```go
   OnStoppedLeading: func() {
       logger.Info("Lost leadership, exiting")
       klog.FlushAndExit(klog.ExitFlushTimeout, 0)
   }
   ```

❌ **DON'T**:
1. **Use same identity for multiple instances**
2. **Set timing parameters that violate constraints**
3. **Ignore OnStoppedLeading callback**
4. **Block in OnStartedLeading callback** (use goroutine)

### **10.2 Healthz Integration**

**File**: `/staging/src/k8s.io/client-go/tools/leaderelection/healthzadaptor.go`

```go
// Create healthz adaptor
healthzAdaptor := leaderelection.NewLeaderHealthzAdaptor(
    time.Second * 20,  // maxTolerableExpiredLease
)

// Set in config
config := leaderelection.LeaderElectionConfig{
    // ... other config
    WatchDog: healthzAdaptor,
}

// Add to healthz endpoint
mux.Handle("/healthz", healthz.NamedCheck("leaderelection", healthzAdaptor.Check))
```

**Healthz Check Logic**:
```go
func (le *LeaderElector) Check(maxTolerableExpiredLease time.Duration) error {
    if !le.IsLeader() {
        return nil  // Not leader, healthy
    }

    // If leader but lease expired beyond tolerance, unhealthy
    if le.clock.Since(le.observedTime) > le.config.LeaseDuration + maxTolerableExpiredLease {
        return fmt.Errorf("failed election to renew leadership on lease %s", le.config.Name)
    }

    return nil
}
```

### **10.3 Testing Leader Election**

```go
func TestLeaderElection(t *testing.T) {
    // Use fake clock for deterministic testing
    fakeClock := clock.NewFakeClock(time.Now())

    // Create mock resource lock
    lock := &mockResourceLock{
        identity: "test-1",
        record:   &resourcelock.LeaderElectionRecord{},
    }

    config := leaderelection.LeaderElectionConfig{
        Lock:          lock,
        LeaseDuration: 15 * time.Second,
        RenewDeadline: 10 * time.Second,
        RetryPeriod:   2 * time.Second,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                // Test started leading
                acquired <- true
            },
            OnStoppedLeading: func() {
                // Test stopped leading
                released <- true
            },
        },
    }

    le, err := leaderelection.NewLeaderElector(config)
    require.NoError(t, err)

    // Inject fake clock
    le.clock = fakeClock

    // Run election
    ctx, cancel := context.WithCancel(context.Background())
    go le.Run(ctx)

    // Wait for leadership
    select {
    case <-acquired:
        // Success
    case <-time.After(5 * time.Second):
        t.Fatal("Failed to acquire leadership")
    }

    // Advance clock to trigger renewal
    fakeClock.Step(3 * time.Second)

    // Cancel and check release
    cancel()
    select {
    case <-released:
        // Success
    case <-time.After(5 * time.Second):
        t.Fatal("Failed to release leadership")
    }
}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **11. Anti-Patterns to Avoid**

### **11.1 Anti-Pattern: Blocking in OnStartedLeading**

❌ **WRONG**:
```go
Callbacks: leaderelection.LeaderCallbacks{
    OnStartedLeading: func(ctx context.Context) {
        // BLOCKING - prevents renewal loop from running!
        runControllers(ctx)
    },
}
```

✅ **CORRECT**:
```go
Callbacks: leaderelection.LeaderCallbacks{
    OnStartedLeading: func(ctx context.Context) {
        // Run in goroutine - allows Run() to continue to renew()
        go runControllers(ctx)
    },
}
```

**Why**: The `Run()` method calls `OnStartedLeading` and then immediately calls `renew()`. If `OnStartedLeading` blocks, `renew()` never runs, and the lease expires.

### **11.2 Anti-Pattern: Not Handling OnStoppedLeading**

❌ **WRONG**:
```go
OnStoppedLeading: func() {
    logger.Info("Lost leadership")
    // Continue running - DANGEROUS!
}
```

✅ **CORRECT**:
```go
OnStoppedLeading: func() {
    logger.Info("Lost leadership, exiting")
    klog.FlushAndExit(klog.ExitFlushTimeout, 0)
}
```

**Why**: If the process continues running after losing leadership, it may perform operations that conflict with the new leader.

### **11.3 Anti-Pattern: Too Aggressive Timing**

❌ **WRONG**:
```go
LeaseDuration: 3 * time.Second   // Too short!
RenewDeadline: 2 * time.Second   // Too short!
RetryPeriod:   500 * time.Millisecond  // Too frequent!
```

**Problems**:
- Excessive API server load
- Frequent false-positive failures due to transient issues
- Doesn't meet validation constraints
- Leader thrashing

### **11.4 Anti-Pattern: Ignoring Clock Skew**

❌ **WRONG**:
```go
// Assuming all clocks are synchronized
if time.Now().After(lease.RenewTime.Add(lease.LeaseDuration)) {
    // Lease expired
}
```

✅ **CORRECT**:
```go
// Use local observed time
func (le *LeaderElector) isLeaseValid(now time.Time) bool {
    return le.observedTime.Add(
        time.Second * time.Duration(le.getObservedRecord().LeaseDurationSeconds),
    ).After(now)
}
```

**Why**: The implementation only trusts local clock readings and observes changes in remote timestamps.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **12. Performance Considerations**

### **12.1 Fast Path Optimization**

The fast path allows the current leader to renew without fetching the lease:

```mermaid
sequenceDiagram
    participant Leader
    participant API
    participant etcd

    Note over Leader: Fast Path (Optimistic)
    Leader->>API: Update(lease, expectedVersion)
    API->>etcd: Conditional Update
    etcd-->>API: Success (or Conflict)
    API-->>Leader: Success (or Error)

    Note over Leader: Slow Path (on conflict)
    Leader->>API: Get(lease)
    API->>etcd: Get
    etcd-->>API: Current Lease
    API-->>Leader: Lease Data
    Leader->>API: Update(lease, newVersion)
    API->>etcd: Conditional Update
    etcd-->>API: Success
    API-->>Leader: Success
```

**Fast Path Benefits**:
- 1 API call instead of 2
- Lower latency
- Reduced API server load
- Works in 99% of cases for stable leader

**Slow Path Triggered By**:
- ResourceVersion conflict
- Concurrent update attempt
- First acquisition attempt
- Leadership change

### **12.2 Jitter Factor Impact**

**Constant**: `JitterFactor = 1.2`

**Purpose**: Prevent thundering herd when multiple candidates compete

**Implementation**:
```go
wait.JitterUntil(func() {
    succeeded = le.tryAcquireOrRenew(ctx)
    // ...
}, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
```

**Effect**:
```
RetryPeriod: 2s
Actual retry interval: 2s ± 20% = 1.6s to 2.4s

With 3 candidates:
Candidate A: retries at 1.7s, 3.5s, 5.2s, ...
Candidate B: retries at 2.1s, 4.3s, 6.4s, ...
Candidate C: retries at 1.9s, 4.0s, 6.1s, ...

Without jitter:
All candidates: retry at 2s, 4s, 6s, ... (collision!)
```

### **12.3 Scaling Characteristics**

**API Server Load**:
```
Load = (Number of Replicas) × (1 / RetryPeriod)

Example with 3 replicas:
- RetryPeriod = 2s: 1.5 requests/second
- RetryPeriod = 5s: 0.6 requests/second
```

**Recommendation**:
- Small clusters (<100 nodes): RetryPeriod = 2s
- Medium clusters (100-1000 nodes): RetryPeriod = 3s
- Large clusters (>1000 nodes): RetryPeriod = 5s

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **13. Cross-References**

### **13.1 Related Documentation**

**Distributed Systems Patterns**:
- [02-consensus-algorithms.md](./02-consensus-algorithms.md) - Raft consensus in etcd
- [03-eventual-consistency.md](./03-eventual-consistency.md) - Reconciliation loops
- [04-cap-theorem-practice.md](./04-cap-theorem-practice.md) - Consistency trade-offs
- [05-failure-modes.md](./05-failure-modes.md) - Leader election failure scenarios
- [06-resilience-patterns.md](./06-resilience-patterns.md) - Retry and backoff strategies
- [07-coordination-patterns.md](./07-coordination-patterns.md) - ResourceLock patterns
- [08-synchronization-primitives.md](./08-synchronization-primitives.md) - Finalizers and OwnerReferences

**Component Documentation**:
- [../controller-manager/](../controller-manager/) - Controller manager architecture
- [../kube-scheduler/](../kube-scheduler/) - Scheduler architecture
- [../common/04-workqueue-leaderelection.md](../common/04-workqueue-leaderelection.md) - WorkQueue integration

**etcd Documentation**:
- [../etcd/low-level/01-etcd3-client.md](../etcd/low-level/01-etcd3-client.md) - etcd client details
- [../etcd/middle-level/04-transactions-consistency.md](../etcd/middle-level/04-transactions-consistency.md) - etcd consistency

### **13.2 Code References Summary**

**Primary Implementation**:
```
/staging/src/k8s.io/client-go/tools/leaderelection/
├── leaderelection.go          # Main logic (544 lines)
├── healthzadaptor.go          # Health check integration
├── metrics.go                 # Prometheus metrics
└── resourcelock/
    ├── interface.go           # ResourceLock interface
    ├── leaselock.go           # Lease-based lock implementation
    └── multilock.go           # Multi-lock for migrations
```

**Usage Examples**:
```
/cmd/kube-controller-manager/app/controllermanager.go
/cmd/kube-scheduler/app/server.go
/pkg/controlplane/controller/leaderelection/
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **14. Summary**

### **14.1 Key Takeaways**

1. **Lease-Based Election**: Kubernetes uses time-based leases for leader election, not Raft consensus
2. **Clock Skew Tolerance**: Implementation is tolerant to clock skew via local timestamp observations
3. **Fast Path Optimization**: Current leader uses optimistic updates to minimize API calls
4. **Callbacks**: Lifecycle callbacks enable application integration (OnStartedLeading, OnStoppedLeading)
5. **ResourceLock Evolution**: Migrated from Endpoints/ConfigMaps to Lease objects
6. **Jitter**: Prevents thundering herd during candidate competition
7. **Tunable Parameters**: LeaseDuration, RenewDeadline, RetryPeriod must satisfy specific constraints

### **14.2 Configuration Decision Tree**

```mermaid
graph TD
    A[Choose Leader Election Config] --> B{Cluster Size?}

    B -->|Small < 100| C[15s / 10s / 2s]
    B -->|Medium 100-1000| D[20s / 13s / 3s]
    B -->|Large > 1000| E[30s / 20s / 5s]

    C --> F{API Latency?}
    D --> F
    E --> F

    F -->|Low < 50ms| G[Use Defaults]
    F -->|Medium 50-200ms| H[Increase by 50%]
    F -->|High > 200ms| I[Increase by 100%]

    G --> J[Deploy]
    H --> J
    I --> J

    J --> K{Monitor Transitions}
    K -->|High Rate| L[Increase LeaseDuration]
    K -->|Normal| M[Done]

    L --> K
```

### **14.3 Common Commands**

```bash
# List all leases
kubectl get leases -A

# Describe controller-manager lease
kubectl describe lease kube-controller-manager -n kube-system

# Watch for leader changes
kubectl get lease kube-controller-manager -n kube-system -w

# Check leader transitions
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.leaderTransitions}'

# Get current leader identity
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.holderIdentity}'

# Monitor renewal
watch -n 1 'kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath="{.spec.renewTime}"'
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **References**

1. **Kubernetes Documentation**: https://kubernetes.io/docs/concepts/architecture/leases/
2. **Client-Go Leader Election**: https://pkg.go.dev/k8s.io/client-go/tools/leaderelection
3. **KEP-1753**: Coordinated Leader Election - https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1753-coordinated-leader-election
4. **Original Design Doc**: Leader Election Library Design
5. **etcd Lease Documentation**: https://etcd.io/docs/latest/learning/api/#lease-api

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Version**: 1.0
**Last Updated**: 2025-01-16
**Kubernetes Version**: v1.32+
**Lines**: ~2,100
**Diagrams**: 12
