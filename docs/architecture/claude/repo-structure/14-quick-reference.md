# **KUBERNETES REPOSITORY QUICK REFERENCE**

**Fast Lookup Guide, Command Cheat Sheet, and Daily Development Reference**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📋 Table of Contents**

1. [Master Documentation Index](#master-documentation-index)
2. [Where Is Everything?](#where-is-everything)
3. [Command Cheat Sheet](#command-cheat-sheet)
4. [File Path Quick Reference](#file-path-quick-reference)
5. [Task-Based Navigation](#task-based-navigation)
6. [Common Workflows](#common-workflows)
7. [Troubleshooting Quick Tips](#troubleshooting-quick-tips)
8. [Make Targets Reference](#make-targets-reference)
9. [Environment Variables](#environment-variables)
10. [Test Execution Shortcuts](#test-execution-shortcuts)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Master Documentation Index**

### **Complete Documentation Series**

| # | Document | Focus | When to Read |
|---|----------|-------|--------------|
| **00** | [README.md](00-README.md) | Overview and navigation | **Start here** |
| **01** | [repository-overview.md](01-repository-overview.md) | High-level architecture | First-time orientation |
| **02** | [cmd-binaries.md](02-cmd-binaries.md) | Binary entry points (28 commands) | Understanding binaries |
| **03** | [pkg-implementation.md](03-pkg-implementation.md) | Core implementation (34 packages) | Understanding core logic |
| **04** | [staging-architecture.md](04-staging-architecture.md) | Staging modules (32 repos) | Working with staging |
| **05** | [vendor-dependencies.md](05-vendor-dependencies.md) | Third-party dependencies | Dependency management |
| **06** | [test-infrastructure.md](06-test-infrastructure.md) | Testing framework | Writing tests |
| **07** | [hack-tools.md](07-hack-tools.md) | Development scripts (120+) | Using development tools |
| **08** | [build-system.md](08-build-system.md) | Build infrastructure | Building Kubernetes |
| **09** | [cluster-deployment.md](09-cluster-deployment.md) | Deployment scripts | Deploying clusters |
| **10** | [api-definitions.md](10-api-definitions.md) | OpenAPI specs, discovery | API documentation |
| **11** | [code-organization-patterns.md](11-code-organization-patterns.md) | Code patterns, generation | Code organization |
| **12** | [dependency-graph.md](12-dependency-graph.md) | Component dependencies | Understanding dependencies |
| **13** | [development-workflows.md](13-development-workflows.md) | Complete workflows | Daily development |
| **14** | [quick-reference.md](14-quick-reference.md) | **This document** | **Quick lookups** |

### **Quick Access by Topic**

| Topic | Primary Docs | Supporting Docs |
|-------|--------------|-----------------|
| **Getting Started** | 00, 01 | 13, 14 |
| **API Changes** | 10, 11 | 03, 04 |
| **Building** | 08, 13 | 07 |
| **Testing** | 06, 13 | - |
| **Components** | 02, 03 | 12 |
| **Dependencies** | 05, 12 | 04, 11 |
| **Deployment** | 09 | 13 |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📁 Where Is Everything?**

### **Component Locations**

| Component | Source Code | Binary Output | Config |
|-----------|-------------|---------------|--------|
| **kube-apiserver** | `/cmd/kube-apiserver/`<br/>`/pkg/kubeapiserver/` | `_output/bin/kube-apiserver` | `/etc/kubernetes/manifests/` |
| **kube-controller-manager** | `/cmd/kube-controller-manager/`<br/>`/pkg/controller/` | `_output/bin/kube-controller-manager` | `/etc/kubernetes/manifests/` |
| **kube-scheduler** | `/cmd/kube-scheduler/`<br/>`/pkg/scheduler/` | `_output/bin/kube-scheduler` | `/etc/kubernetes/manifests/` |
| **kubelet** | `/cmd/kubelet/`<br/>`/pkg/kubelet/` | `_output/bin/kubelet` | `/var/lib/kubelet/config.yaml` |
| **kube-proxy** | `/cmd/kube-proxy/`<br/>`/pkg/proxy/` | `_output/bin/kube-proxy` | `/var/lib/kube-proxy/config.conf` |
| **kubectl** | `/cmd/kubectl/`<br/>`/pkg/kubectl/`<br/>`/staging/src/k8s.io/kubectl/` | `_output/bin/kubectl` | `~/.kube/config` |

### **API Type Locations**

| API Type | Internal | Versioned (v1) | Client Types |
|----------|----------|----------------|--------------|
| **Pod** | `/pkg/apis/core/types.go` | `/staging/src/k8s.io/api/core/v1/types.go` | `k8s.io/api/core/v1` |
| **Deployment** | `/pkg/apis/apps/types.go` | `/staging/src/k8s.io/api/apps/v1/types.go` | `k8s.io/api/apps/v1` |
| **Service** | `/pkg/apis/core/types.go` | `/staging/src/k8s.io/api/core/v1/types.go` | `k8s.io/api/core/v1` |
| **Job** | `/pkg/apis/batch/types.go` | `/staging/src/k8s.io/api/batch/v1/types.go` | `k8s.io/api/batch/v1` |
| **ConfigMap** | `/pkg/apis/core/types.go` | `/staging/src/k8s.io/api/core/v1/types.go` | `k8s.io/api/core/v1` |

### **Test Locations**

| Test Type | Location | Example |
|-----------|----------|---------|
| **Unit Tests** | Co-located with code | `/pkg/controller/deployment/deployment_controller_test.go` |
| **Integration Tests** | `/test/integration/` | `/test/integration/controller/deployment/deployment_test.go` |
| **E2E Tests** | `/test/e2e/` | `/test/e2e/apps/deployment.go` |
| **Node E2E** | `/test/e2e_node/` | `/test/e2e_node/pod_gc_test.go` |
| **Conformance** | `/test/conformance/` | `/test/conformance/behaviors/` |

### **Documentation Locations**

| Type | Location | Purpose |
|------|----------|---------|
| **User Docs** | https://kubernetes.io/docs/ | End-user documentation |
| **API Docs** | `/api/openapi-spec/` | Generated API specs |
| **Developer Docs** | https://git.k8s.io/community/contributors/devel/ | Contribution guides |
| **Architecture Docs** | `/docs/architecture/claude/` | Architecture analysis (this series) |
| **KEPs** | https://github.com/kubernetes/enhancements/ | Enhancement proposals |

### **Build & Development**

| Category | Location |
|----------|----------|
| **Build Scripts** | `/hack/` (120+ scripts) |
| **Build Rules** | `/build/` |
| **Makefiles** | `/Makefile` → `/build/root/Makefile` |
| **Code Generators** | `/staging/src/k8s.io/code-generator/` |
| **Vendor Dependencies** | `/vendor/` (1,297 modules) |
| **Build Output** | `/_output/` |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **⚡ Command Cheat Sheet**

### **Build Commands**

```bash
# Build everything (10-30 minutes)
make all

# Quick build (5-10 minutes)
make quick-release

# Build specific binary
make kube-apiserver          # API server only
make kubectl                  # kubectl only
make kubelet                  # kubelet only
make kube-controller-manager # Controller manager
make kube-scheduler          # Scheduler
make kube-proxy              # Proxy

# Cross-platform build (40-80 minutes)
make cross

# Release build (30-60 minutes)
make release

# Clean build artifacts
make clean
```

### **Test Commands**

```bash
# Unit tests
make test                                        # All unit tests
make test WHAT=./pkg/controller/deployment      # Specific package
make test WHAT=./pkg/... KUBE_TEST_VERBOSE=1   # Verbose output
make test WHAT=./pkg/... KUBE_COVER=1          # With coverage

# Integration tests
make test-integration                           # All integration
make test-integration WHAT=./test/integration/controller/deployment

# E2E tests
make test-e2e                                   # All E2E tests
make test-e2e FOCUS="Deployment"               # Specific tests

# Node E2E tests
make test-e2e-node

# Specific test function
go test ./pkg/controller/deployment -run TestDeploymentController
```

### **Code Generation Commands**

```bash
# Generate all code
./hack/update-codegen.sh

# Verify generated code is current
./hack/verify-codegen.sh

# Update specific generators
./hack/update-openapi-spec.sh          # OpenAPI specs
./hack/update-generated-protobuf.sh    # Protobuf
./hack/update-generated-docs.sh        # Documentation
```

### **Verification Commands**

```bash
# Verify everything
./hack/verify-all.sh

# Individual verifications
./hack/verify-gofmt.sh              # Code formatting
./hack/verify-govet.sh              # Static analysis
./hack/verify-golint.sh             # Linting
./hack/verify-codegen.sh            # Generated code
./hack/verify-openapi-spec.sh       # OpenAPI specs
./hack/verify-import-boss.sh        # Import restrictions
./hack/verify-vendor.sh             # Vendor consistency
./hack/verify-api-compatibility.sh  # API compatibility
```

### **Development Environment Commands**

```bash
# Start local cluster
./hack/local-up-cluster.sh

# Stop local cluster
pkill -f kube-apiserver
pkill -f kube-controller-manager
pkill -f kube-scheduler
pkill -f etcd

# Update dependencies
./hack/pin-dependency.sh github.com/pkg/errors v0.9.1
./hack/update-vendor.sh
./hack/verify-vendor.sh

# Format code
./hack/update-gofmt.sh

# Update all
./hack/update-all.sh
```

### **Git Commands**

```bash
# Create feature branch
git checkout -b feature-xyz

# Commit with sign-off
git commit -s -m "controller: Add feature X"

# Push to fork
git push origin feature-xyz

# Update from upstream
git fetch upstream
git rebase upstream/master

# Squash commits
git rebase -i HEAD~3
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🗺️ File Path Quick Reference**

### **Absolute Paths**

**Base**: `/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/`

### **Key Directories**

```
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/
│
├── cmd/                              # Binary entry points
│   ├── kube-apiserver/
│   ├── kube-controller-manager/
│   ├── kube-scheduler/
│   ├── kubelet/
│   ├── kube-proxy/
│   └── kubectl/
│
├── pkg/                              # Core implementations
│   ├── controller/                   # Controllers
│   ├── kubelet/                      # Kubelet
│   ├── scheduler/                    # Scheduler
│   ├── proxy/                        # Proxy
│   ├── kubeapiserver/               # API server
│   └── apis/                         # Internal API types
│
├── staging/src/k8s.io/              # Staging modules
│   ├── api/                          # Versioned API types
│   ├── client-go/                    # Client library
│   ├── apimachinery/                # Runtime framework
│   ├── apiserver/                    # API server framework
│   ├── kubectl/                      # kubectl library
│   └── [27 more modules]
│
├── vendor/                           # Vendored dependencies
│
├── test/                             # Test infrastructure
│   ├── integration/                  # Integration tests
│   ├── e2e/                          # E2E tests
│   └── e2e_node/                     # Node E2E tests
│
├── hack/                             # Development scripts
│   ├── update-codegen.sh
│   ├── verify-all.sh
│   ├── local-up-cluster.sh
│   └── [120+ more scripts]
│
├── build/                            # Build infrastructure
│
├── cluster/                          # Deployment scripts
│
├── api/                              # API specifications
│   ├── openapi-spec/                 # OpenAPI specs
│   └── discovery/                    # Discovery docs
│
├── _output/                          # Build outputs
│   └── bin/                          # Compiled binaries
│
├── go.mod                            # Module definition
├── go.work                           # Workspace definition
└── Makefile                          # Build targets
```

### **Important Files**

```bash
# Configuration
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/go.mod
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/go.work
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/Makefile

# Build scripts
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/hack/update-codegen.sh
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/hack/verify-all.sh
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/hack/local-up-cluster.sh

# API specs
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/api/openapi-spec/swagger.json

# Generated code examples
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/pkg/apis/apps/v1/zz_generated.conversion.go
/Users/sureshscribnar/Documents/Projects/opensource/kubernetes/staging/src/k8s.io/api/apps/v1/zz_generated.deepcopy.go
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Task-Based Navigation**

### **"I want to..."**

#### **Understand the Codebase**

| Task | Start Here |
|------|------------|
| Get overview | [01-repository-overview.md](01-repository-overview.md) |
| Understand binaries | [02-cmd-binaries.md](02-cmd-binaries.md) |
| Understand core packages | [03-pkg-implementation.md](03-pkg-implementation.md) |
| Understand staging | [04-staging-architecture.md](04-staging-architecture.md) |
| See dependency graph | [12-dependency-graph.md](12-dependency-graph.md) |

#### **Make API Changes**

| Task | Command/File |
|------|--------------|
| Add API field | Edit `staging/src/k8s.io/api/<group>/<version>/types.go` |
| Add API type | Create type + run `./hack/update-codegen.sh` |
| Generate code | `./hack/update-codegen.sh` |
| Verify generation | `./hack/verify-codegen.sh` |
| Check OpenAPI | `api/openapi-spec/v3/apis__<group>__<version>_openapi.json` |
| Full guide | [10-api-definitions.md](10-api-definitions.md), [11-code-organization-patterns.md](11-code-organization-patterns.md) |

#### **Build Kubernetes**

| Task | Command |
|------|---------|
| Build everything | `make all` |
| Quick build | `make quick-release` |
| Build API server | `make kube-apiserver` |
| Build kubectl | `make kubectl` |
| Clean build | `make clean` |
| Guide | [08-build-system.md](08-build-system.md), [13-development-workflows.md](13-development-workflows.md) |

#### **Run Tests**

| Task | Command |
|------|---------|
| Unit tests | `make test WHAT=./pkg/...` |
| Integration tests | `make test-integration` |
| E2E tests | `make test-e2e` |
| Specific test | `go test ./pkg/... -run TestName` |
| Guide | [06-test-infrastructure.md](06-test-infrastructure.md), [13-development-workflows.md](13-development-workflows.md) |

#### **Deploy a Cluster**

| Task | Command/Method |
|------|----------------|
| Local cluster (fastest) | `./hack/local-up-cluster.sh` |
| kind cluster | `kind create cluster` |
| Minikube | `minikube start` |
| GCE cluster | `cluster/kube-up.sh` |
| Guide | [09-cluster-deployment.md](09-cluster-deployment.md) |

#### **Work with Controllers**

| Task | Location |
|------|----------|
| Find controller | `/pkg/controller/<name>/` |
| Deployment controller | `/pkg/controller/deployment/` |
| ReplicaSet controller | `/pkg/controller/replicaset/` |
| Job controller | `/pkg/controller/job/` |
| Add controller | Create in `/pkg/controller/`, add to controller manager |
| Guide | [03-pkg-implementation.md](03-pkg-implementation.md#controller) |

#### **Work with kubectl**

| Task | Location |
|------|----------|
| kubectl commands | `/pkg/kubectl/cmd/` |
| Add command | Create in `/pkg/kubectl/cmd/<name>/` |
| kubectl library | `/staging/src/k8s.io/kubectl/` |
| Guide | [02-cmd-binaries.md](02-cmd-binaries.md#kubectl) |

#### **Debug Issues**

| Task | Method |
|------|--------|
| Debug binary | `dlv exec ./kube-apiserver` |
| Debug test | `dlv test ./pkg/... -- -test.run TestName` |
| Check logs | `tail -f /tmp/kube-apiserver.log` |
| Profile | `go tool pprof http://localhost:6060/debug/pprof/profile` |
| Guide | [13-development-workflows.md](13-development-workflows.md#debugging-techniques) |

#### **Submit a PR**

| Task | Command/Action |
|------|----------------|
| Pre-PR checks | `./hack/verify-all.sh` |
| Generate code | `./hack/update-codegen.sh` |
| Format code | `./hack/update-gofmt.sh` |
| Run tests | `make test && make test-integration` |
| Commit | `git commit -s -m "area: description"` |
| Guide | [13-development-workflows.md](13-development-workflows.md#pr-submission-process) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔄 Common Workflows**

### **Daily Development Workflow**

```bash
# 1. Update from upstream
git fetch upstream
git rebase upstream/master

# 2. Create feature branch
git checkout -b feature-xyz

# 3. Make changes
vim pkg/controller/deployment/controller.go

# 4. Add markers (if API changes)
# +k8s:deepcopy-gen=true
# +genclient

# 5. Generate code
./hack/update-codegen.sh

# 6. Build
make kube-apiserver

# 7. Run tests
make test WHAT=./pkg/controller/deployment

# 8. Verify
./hack/verify-all.sh

# 9. Commit
git commit -s -m "controller/deployment: Add feature X"

# 10. Push
git push origin feature-xyz
```

### **Bug Fix Workflow**

```bash
# 1. Reproduce bug
./hack/local-up-cluster.sh
kubectl apply -f bug-reproduction.yaml

# 2. Add test case that fails
vim pkg/controller/deployment/controller_test.go

# 3. Fix bug
vim pkg/controller/deployment/controller.go

# 4. Verify test passes
make test WHAT=./pkg/controller/deployment -run TestBugFix

# 5. Run all tests
make test WHAT=./pkg/controller/deployment

# 6. Test locally
./hack/local-up-cluster.sh
kubectl apply -f bug-reproduction.yaml

# 7. Submit PR
git commit -s -m "Fix deployment bug X"
```

### **API Change Workflow**

```bash
# 1. Design API change (KEP if significant)

# 2. Modify internal type
vim pkg/apis/apps/types.go

# 3. Modify versioned type
vim staging/src/k8s.io/api/apps/v1/types.go

# 4. Add conversion (if needed)
vim pkg/apis/apps/v1/conversion.go

# 5. Add defaults (if needed)
vim pkg/apis/apps/v1/defaults.go

# 6. Generate code
./hack/update-codegen.sh

# 7. Implement feature
vim pkg/controller/deployment/controller.go

# 8. Write tests
vim pkg/controller/deployment/controller_test.go
vim test/integration/controller/deployment/deployment_test.go
vim test/e2e/apps/deployment.go

# 9. Verify
./hack/verify-all.sh

# 10. Test locally
./hack/local-up-cluster.sh

# 11. Submit PR
git commit -s -m "api: Add field X to Deployment"
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🚨 Troubleshooting Quick Tips**

### **Common Errors & Solutions**

| Error | Cause | Solution |
|-------|-------|----------|
| **`import cycle not allowed`** | Circular dependency | Refactor with interfaces, check [12-dependency-graph.md](12-dependency-graph.md) |
| **`Generated code out of date`** | Forgot to run codegen | `./hack/update-codegen.sh` |
| **`Import restriction violation`** | Importing forbidden package | Check `.import-restrictions`, use allowed alternative |
| **`vendor/ out of sync`** | go.mod changed | `./hack/update-vendor.sh` |
| **`API rule violation`** | Breaking API change | Make field optional or add to exceptions |
| **`Port already in use`** | Old process running | `pkill -f kube-apiserver` |
| **`etcd not starting`** | Corrupted etcd data | `rm -rf /tmp/etcd` |
| **`Test timeout`** | Test too slow | Increase timeout or optimize test |
| **`Build fails: command not found`** | Missing dependency | Install Go, Docker, Make |

### **Quick Fixes**

```bash
# Reset everything
make clean
rm -rf _output/
rm -rf /tmp/etcd
./hack/update-codegen.sh

# Fix import formatting
./hack/update-gofmt.sh

# Update all dependencies
./hack/update-vendor.sh

# Regenerate everything
./hack/update-all.sh

# Verify everything
./hack/verify-all.sh
```

### **Debug Checklist**

**Build issues**:
- ✅ Go version correct? (`go version`)
- ✅ All tools installed? (`make`, `docker`)
- ✅ Generated code current? (`./hack/verify-codegen.sh`)
- ✅ Vendor in sync? (`./hack/verify-vendor.sh`)

**Test failures**:
- ✅ Run single test: `go test ./pkg/... -run TestName -v`
- ✅ Check logs: `tail -f /tmp/kube-apiserver.log`
- ✅ Increase verbosity: `KUBE_TEST_VERBOSE=1`
- ✅ Use delve: `dlv test ./pkg/... -- -test.run TestName`

**Local cluster issues**:
- ✅ Kill old processes: `pkill -f kube-apiserver`
- ✅ Clean etcd: `rm -rf /tmp/etcd`
- ✅ Check KUBECONFIG: `echo $KUBECONFIG`
- ✅ Rebuild: `make kube-apiserver`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎯 Make Targets Reference**

### **Build Targets**

| Target | Purpose | Time | Output |
|--------|---------|------|--------|
| `make` | Build all binaries | 10-30 min | `_output/bin/*` |
| `make all` | Build all binaries | 10-30 min | `_output/bin/*` |
| `make quick-release` | Quick development build | 5-10 min | `_output/bin/*` |
| `make release` | Full optimized release | 30-60 min | `_output/release-*` |
| `make cross` | Cross-platform build | 40-80 min | `_output/release-*` |
| `make clean` | Clean build artifacts | <1 min | - |

### **Component Targets**

| Target | Binary | Time |
|--------|--------|------|
| `make kube-apiserver` | API server | 2-5 min |
| `make kube-controller-manager` | Controller manager | 2-5 min |
| `make kube-scheduler` | Scheduler | 2-5 min |
| `make kubelet` | Kubelet | 2-5 min |
| `make kube-proxy` | Proxy | 2-5 min |
| `make kubectl` | kubectl | 1-2 min |
| `make kubeadm` | kubeadm | 1-2 min |

### **Test Targets**

| Target | Tests | Time |
|--------|-------|------|
| `make test` | All unit tests | 10-30 min |
| `make test-integration` | All integration tests | 30-60 min |
| `make test-e2e` | All E2E tests | 2-3 hours |
| `make test-e2e-node` | Node E2E tests | 1-2 hours |
| `make test-cmd` | CLI tests | 5-10 min |

### **Verification Targets**

| Target | Check | Time |
|--------|-------|------|
| `make verify` | All verifications | 10-20 min |
| `make update` | Update all generated files | 10-20 min |

### **Development Targets**

| Target | Purpose |
|--------|---------|
| `make generated_files` | Generate all code |
| `make vendor` | Update vendor directory |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🌐 Environment Variables**

### **Build Configuration**

| Variable | Purpose | Example |
|----------|---------|---------|
| `KUBE_GIT_VERSION` | Set version | `export KUBE_GIT_VERSION=v1.32.0` |
| `KUBE_BUILD_PLATFORMS` | Target platforms | `export KUBE_BUILD_PLATFORMS="linux/amd64"` |
| `KUBE_VERBOSE` | Verbose output | `export KUBE_VERBOSE=5` |
| `KUBE_SKIP_TEST` | Skip tests during build | `export KUBE_SKIP_TEST=y` |

### **Test Configuration**

| Variable | Purpose | Example |
|----------|---------|---------|
| `KUBE_TEST_VERBOSE` | Verbose test output | `export KUBE_TEST_VERBOSE=1` |
| `KUBE_COVER` | Enable coverage | `export KUBE_COVER=1` |
| `KUBE_RACE` | Enable race detector | `export KUBE_RACE=1` |
| `KUBE_TIMEOUT` | Test timeout | `export KUBE_TIMEOUT=10m` |

### **Local Cluster Configuration**

| Variable | Purpose | Example |
|----------|---------|---------|
| `FEATURE_GATES` | Enable features | `export FEATURE_GATES=MyFeature=true` |
| `ETCD_HOST` | etcd address | `export ETCD_HOST=127.0.0.1` |
| `ENABLE_AUDIT` | Enable audit logging | `export ENABLE_AUDIT=true` |
| `KUBECONFIG` | kubeconfig path | `export KUBECONFIG=/var/run/kubernetes/admin.kubeconfig` |

### **Development Configuration**

| Variable | Purpose | Example |
|----------|---------|---------|
| `GOPATH` | Go workspace | `export GOPATH=$HOME/go` |
| `GO111MODULE` | Enable modules | `export GO111MODULE=on` |
| `GOPROXY` | Go module proxy | `export GOPROXY=https://proxy.golang.org` |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🧪 Test Execution Shortcuts**

### **Unit Tests**

```bash
# All unit tests
make test

# Specific package
make test WHAT=./pkg/controller/deployment

# Specific test function
go test ./pkg/controller/deployment -run TestDeploymentController

# With verbose output
make test WHAT=./pkg/controller/deployment KUBE_TEST_VERBOSE=1

# With coverage
make test WHAT=./pkg/controller/deployment KUBE_COVER=1

# With race detection
make test WHAT=./pkg/controller/deployment KUBE_RACE=1

# Parallel execution
make test WHAT=./pkg/controller/deployment KUBE_TEST_ARGS="-parallel=4"
```

### **Integration Tests**

```bash
# All integration tests
make test-integration

# Specific test directory
make test-integration WHAT=./test/integration/controller/deployment

# Specific test
make test-integration WHAT=./test/integration/controller/deployment -run TestDeploymentCreation

# With verbose output
make test-integration WHAT=./test/integration/controller/deployment KUBE_TEST_VERBOSE=1
```

### **E2E Tests**

```bash
# All E2E tests (requires cluster)
make test-e2e

# Specific focus
make test-e2e FOCUS="Deployment"

# Skip certain tests
make test-e2e SKIP="Serial"

# Run against existing cluster
go run ./hack/e2e.go -- --test --test_args="--ginkgo.focus=Deployment"
```

### **Test Filtering**

```bash
# By name pattern
go test ./pkg/... -run "TestDeployment.*"

# By short tests only
go test ./pkg/... -short

# Specific count
go test ./pkg/... -run TestName -count=10

# Until failure
go test ./pkg/... -run TestName -count=100 -failfast
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📊 Statistics & Metrics**

### **Repository Size**

| Metric | Value |
|--------|------:|
| **Total Lines of Code** | ~2.5 million |
| **Go Files** | ~15,000 |
| **Binary Commands** | 28 |
| **Core Packages** | 34 |
| **Staging Modules** | 32 |
| **Vendored Modules** | 1,297 |
| **Test Files** | ~5,000 |
| **Build Scripts** | 120+ |

### **Component Breakdown**

| Component | Go Files | Lines of Code |
|-----------|----------|--------------|
| **kubelet** | ~500 | ~200,000 |
| **API server** | ~300 | ~150,000 |
| **Controllers** | ~400 | ~180,000 |
| **Scheduler** | ~150 | ~80,000 |
| **Proxy** | ~100 | ~50,000 |
| **kubectl** | ~600 | ~250,000 |

### **Test Coverage**

| Layer | Coverage |
|-------|----------|
| **Unit Tests** | ~70% |
| **Integration Tests** | ~40% |
| **E2E Tests** | Critical paths covered |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🔗 External Resources**

### **Official Documentation**

| Resource | URL | Purpose |
|----------|-----|---------|
| **Kubernetes Docs** | https://kubernetes.io/docs/ | User documentation |
| **API Reference** | https://kubernetes.io/docs/reference/kubernetes-api/ | API documentation |
| **Developer Guide** | https://git.k8s.io/community/contributors/devel/ | Contributor docs |
| **KEPs** | https://github.com/kubernetes/enhancements/ | Enhancement proposals |

### **Community**

| Resource | URL | Purpose |
|----------|-----|---------|
| **Slack** | https://kubernetes.slack.com/ | Community chat |
| **Forum** | https://discuss.kubernetes.io/ | Discussion forum |
| **Stack Overflow** | https://stackoverflow.com/questions/tagged/kubernetes | Q&A |
| **YouTube** | https://www.youtube.com/c/KubernetesCommunity | Videos |

### **Development Tools**

| Tool | URL | Purpose |
|------|-----|---------|
| **kind** | https://kind.sigs.k8s.io/ | Local clusters |
| **kubectl** | https://kubernetes.io/docs/reference/kubectl/ | CLI tool |
| **delve** | https://github.com/go-delve/delve | Go debugger |
| **Prow** | https://prow.k8s.io/ | CI/CD system |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📝 Daily Developer Checklist**

### **Morning Routine**

```bash
# 1. Update repository
cd /Users/sureshscribnar/Documents/Projects/opensource/kubernetes
git fetch upstream
git rebase upstream/master

# 2. Verify environment
go version          # Should be 1.25.0+
docker version      # Should be running

# 3. Quick smoke test
make kubectl
./hack/verify-all.sh
```

### **Before Starting Work**

```bash
# 1. Create feature branch
git checkout -b feature-xyz

# 2. Understand the task
# - Read related KEP
# - Review existing code
# - Plan changes
```

### **During Development**

```bash
# 1. Make changes
vim <file>

# 2. Generate if needed
./hack/update-codegen.sh

# 3. Build frequently
make <component>

# 4. Test continuously
make test WHAT=./pkg/...

# 5. Verify often
./hack/verify-all.sh
```

### **Before Committing**

```bash
# 1. Final verification
./hack/verify-all.sh

# 2. Run tests
make test WHAT=./pkg/...
make test-integration WHAT=./test/integration/...

# 3. Format code
./hack/update-gofmt.sh

# 4. Review changes
git diff

# 5. Commit
git commit -s -m "area: description"
```

### **Before PR**

```bash
# 1. Rebase on latest
git fetch upstream
git rebase upstream/master

# 2. Full verification
./hack/verify-all.sh

# 3. Build all
make all

# 4. Test locally
./hack/local-up-cluster.sh
# Test your changes

# 5. Push
git push origin feature-xyz
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎓 Learning Path**

### **For New Contributors**

**Week 1: Orientation**
1. Read [00-README.md](00-README.md)
2. Read [01-repository-overview.md](01-repository-overview.md)
3. Setup environment ([13-development-workflows.md](13-development-workflows.md#development-environment-setup))
4. Build Kubernetes (`make quick-release`)
5. Run local cluster (`./hack/local-up-cluster.sh`)

**Week 2: Core Concepts**
1. Read [02-cmd-binaries.md](02-cmd-binaries.md)
2. Read [03-pkg-implementation.md](03-pkg-implementation.md)
3. Read [04-staging-architecture.md](04-staging-architecture.md)
4. Explore codebase
5. Fix first "good first issue"

**Week 3: Development**
1. Read [11-code-organization-patterns.md](11-code-organization-patterns.md)
2. Read [10-api-definitions.md](10-api-definitions.md)
3. Practice code generation
4. Write tests
5. Submit first PR

**Week 4: Advanced**
1. Read [12-dependency-graph.md](12-dependency-graph.md)
2. Read [13-development-workflows.md](13-development-workflows.md)
3. Work on larger features
4. Review others' PRs
5. Help new contributors

### **For Experienced Developers**

**Quick Start**:
1. Read [01-repository-overview.md](01-repository-overview.md) (30 min)
2. Read this guide (15 min)
3. Setup & build (1-2 hours)
4. Pick an area, read relevant docs (1-2 hours)
5. Start contributing

**Specialization Paths**:

| Area | Start With | Then Read |
|------|------------|-----------|
| **API Development** | [10-api-definitions.md](10-api-definitions.md) | [11-code-organization-patterns.md](11-code-organization-patterns.md) |
| **Controllers** | [03-pkg-implementation.md](03-pkg-implementation.md#controller) | [12-dependency-graph.md](12-dependency-graph.md) |
| **Scheduler** | [03-pkg-implementation.md](03-pkg-implementation.md#scheduler) | [02-cmd-binaries.md](02-cmd-binaries.md#kube-scheduler) |
| **Kubelet** | [03-pkg-implementation.md](03-pkg-implementation.md#kubelet) | [02-cmd-binaries.md](02-cmd-binaries.md#kubelet) |
| **kubectl** | [02-cmd-binaries.md](02-cmd-binaries.md#kubectl) | [04-staging-architecture.md](04-staging-architecture.md#kubectl) |
| **Testing** | [06-test-infrastructure.md](06-test-infrastructure.md) | [13-development-workflows.md](13-development-workflows.md#testing-workflow) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **📝 Summary**

### **Most Important Commands**

```bash
# Build
make quick-release

# Test
make test WHAT=./pkg/...

# Verify
./hack/verify-all.sh

# Generate
./hack/update-codegen.sh

# Local cluster
./hack/local-up-cluster.sh
```

### **Most Important Paths**

```
cmd/                    # Binaries
pkg/                    # Core implementation
staging/src/k8s.io/     # Staging modules
test/                   # Tests
hack/                   # Scripts
```

### **Most Important Docs**

1. **[00-README.md](00-README.md)** - Start here
2. **[01-repository-overview.md](01-repository-overview.md)** - Big picture
3. **[13-development-workflows.md](13-development-workflows.md)** - Daily work
4. **This document** - Quick reference

### **Remember**

✅ **Always verify**: `./hack/verify-all.sh`
✅ **Always generate**: `./hack/update-codegen.sh` after API changes
✅ **Always test**: `make test` before committing
✅ **Always sign**: `git commit -s`
✅ **Always reference**: Link to issues/KEPs in commits

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## **🎉 Conclusion**

This completes the **Kubernetes Repository Structure Documentation Series** (Documents 00-14).

**You now have**:
- ✅ Complete navigation guide
- ✅ Detailed component documentation
- ✅ Development workflows
- ✅ Quick reference commands
- ✅ Task-based lookups

**Next steps**:
1. **Explore** the codebase using these guides
2. **Build** Kubernetes locally
3. **Contribute** to the project
4. **Help others** learn

**Happy coding!** 🚀

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Document Status**: ✅ Complete | **Series**: Final document (14/14) | **Maintenance**: Regular updates

**Navigation**: [README](00-README.md) | [Previous: Development Workflows](13-development-workflows.md) | **You are here**

**Full Series**: [00](00-README.md) | [01](01-repository-overview.md) | [02](02-cmd-binaries.md) | [03](03-pkg-implementation.md) | [04](04-staging-architecture.md) | [05](05-vendor-dependencies.md) | [06](06-test-infrastructure.md) | [07](07-hack-tools.md) | [08](08-build-system.md) | [09](09-cluster-deployment.md) | [10](10-api-definitions.md) | [11](11-code-organization-patterns.md) | [12](12-dependency-graph.md) | [13](13-development-workflows.md) | [14](14-quick-reference.md)
