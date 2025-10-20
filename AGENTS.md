# Repository Guidelines

## Project Structure & Module Organization
- Source: `cmd/` (binaries like `cmd/kubelet`, `cmd/kubectl`), libraries in `pkg/`.
- Staged modules: `staging/src/k8s.io/*` (mirrors published submodules).
- Tests: unit and integration under `pkg/**/_test.go`, `test/integration/**`, and e2e helpers in `test/e2e/**`.
- Tooling & scripts: `hack/` (e.g., `hack/verify-*.sh`, `hack/update-*.sh`).
- Build and release assets: `build/`, cross‑compilation via Make targets.
- Cluster and local dev: `cluster/`, `hack/local-up-cluster.sh`.

## Build, Test, and Development Commands
- `make help`: list common targets.
- `make all WHAT=cmd/kubectl GOFLAGS=-v`: build a specific binary.
- `make test`: run unit tests.
- `make test-integration WHAT=./test/integration/... KUBE_COVER=y`: run integration tests with coverage.
- `make verify`: run linters/format/typechecks (`hack/verify-*.sh`).
- `make update`: regenerate code/docs/openapi as needed (`hack/update-*.sh`).
- `make cross` / `make quick-release`: cross-build or create release artifacts.

## Coding Style & Naming Conventions
- Language: Go `1.25` (see `go.mod`). Use `gofmt`/`goimports`—run via `make verify` and fix with `hack/update-gofmt.sh`.
- Packages: lower-case, no underscores; files lower-case with underscores permitted; exported identifiers use Go conventions (CamelCase).
- Errors: prefer wrapped errors with context; avoid panics outside `main`/tests.

## Testing Guidelines
- Frameworks: standard `testing` for unit; Ginkgo/Gomega for many e2e suites.
- Names: files `*_test.go`; tests `TestXxx(t *testing.T)`; table-driven where appropriate.
- Run: `make test` (unit), `make test-integration WHAT=...`, e2e via sig-owned tooling under `test/e2e`.
- Coverage: enable with `GOFLAGS='-cover'` or `KUBE_COVER=y` where supported.

## Commit & Pull Request Guidelines
- Commits: concise, imperative subject; include rationale in body. Squash fixups before merge.
- Link issues: use `Fixes #<issue>`/`Refs #<issue>` where applicable.
- PRs: clear description, tests added/updated, docs or generated files refreshed (`make update`), and pass `make verify`.
- Release notes: include a `release-note` block in PR body (or `release-note-none` if not user-facing).
- Contributor requirement: ensure CNCF CLA is signed; add appropriate SIG labels/owners as needed.

## Security & Configuration Tips
- Do not commit secrets or credentials. Use local clusters via `hack/local-up-cluster.sh` for development.
- Keep dependencies pinned with `hack/pin-dependency.sh` and update via `hack/update-vendor.sh`.
