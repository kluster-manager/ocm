# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

Go module `open-cluster-management.io/ocm` — the **core implementation** of Open Cluster Management (OCM), a CNCF sandbox project. OCM uses a hub/agent model: a central "hub" cluster runs the control plane; on each managed cluster a klusterlet agent registers with the hub and executes hub-issued work. This repo provides the five OCM core binaries plus the operator that installs them on a Kubernetes cluster.

Binaries (`cmd/*/main.go`):
- `registration` — registration agent + hub controller (handles `ManagedCluster` lifecycle and CSR auto-approval).
- `work` — `ManifestWork` controller pair: hub side queues work; spoke side applies it.
- `placement` — runs the placement engine that distributes `ManifestWork` across managed clusters based on `Placement` decisions.
- `addon` — addon manager (the upstream side of `addon-framework` integrations).
- `registration-operator` — the operator that installs/upgrades the above components as the `ClusterManager` (hub) and `Klusterlet` (spoke) operators.

Fork is mirrored to `kluster-manager/ocm`; **upstream is `open-cluster-management-io/ocm`** and this repo tracks it. Note: the default branch on origin here is **`ac-0.14.0`** (an AppsCode-pinned release branch), not `main`.

## Architecture

- `cmd/` — one entry point per binary: `addon`, `placement`, `registration`, `registration-operator`, `work`.
- `pkg/` — the corresponding implementation packages:
  - `pkg/registration/` — `hub/`, `spoke/`, `webhook/`, `helpers/`, `clientcert/`. Drives `ManagedCluster` registration via CSR-signed bootstrap kubeconfigs.
  - `pkg/work/` — `hub/`, `spoke/`, `webhook/`, `helper/`. `ManifestWork` apply controllers.
  - `pkg/placement/` — `controllers/`, `plugins/` (pluggable placement strategies), `debugger/`, `helpers/`.
  - `pkg/addon/` — `controllers/`, `manager.go`, `templateagent/`. Implements OCM's side of the `ClusterManagementAddOn` / `ManagedClusterAddOn` contract.
  - `pkg/operator/` — `operators/` (cluster-manager + klusterlet operators), `certrotation/`, `helpers/`. The bits installed by the `registration-operator` binary.
  - `pkg/singleton/spoke/` — combined registration+work agent used by `multicluster-controlplane` for lightweight deploys.
  - `pkg/common/`, `pkg/features/`, `pkg/version/` — shared.
- `manifests/` — Go package that embeds the YAML payloads:
  - `manifests/cluster-manager/` — hub deployment manifests.
  - `manifests/klusterlet/`, `manifests/klusterletkube111/` — agent deployment manifests (separate set for older k8s 1.11-compatible kubelets).
  - `fs.go`, `config.go` — `go:embed` plumbing.
- `deploy/cluster-manager/`, `deploy/klusterlet/` — install-time kustomize bundles.
- `build/` — one Dockerfile per binary (`Dockerfile.addon`, `Dockerfile.placement`, `Dockerfile.registration`, `Dockerfile.registration-operator`, `Dockerfile.work`).
- `assets/` — README diagrams and the OCM logo.
- `solutions/` — runnable example setups (dev environment, integrations).
- `test/` — integration & e2e suites.
- `troubleshooting/`, `docs-like content` — operational runbooks.
- `dependencymagnet/` — Go-only imports forcing vendoring of side-effect deps.
- `Makefile` — pulls in OpenShift's `build-machinery-go`; `vendor/` is **load-bearing** because `Makefile` `include`s `.mk` files from it.

## Common commands

The Makefile inherits most "standard" targets (`build`, `test`, `images`, `update`, `verify`) from OpenShift's `build-machinery-go`. Locally-defined targets focus on codegen and CSV (Operator Lifecycle Manager) bundles.

Standard development (from `golang.mk` / `images.mk`):

- `make build` (alias `make all`) — Go build of every `cmd/*` binary.
- `make test` — Go tests.
- `make images` — build every Dockerfile under `build/` (one per binary).

CRD + CSV updates:

- `make copy-crd` — sync CRDs from `open-cluster-management.io/api` vendored copies into `manifests/`.
- `make update` — `copy-crd update-csv`.
- `make update-csv` — regenerate Operator Lifecycle Manager bundles using `operator-sdk` (auto-installed at `OPERATOR_SDK_VERSION = v1.32.0` into `bin/`). Released CSV version is pinned at `RELEASED_CSV_VERSION = 0.13.3`.
- `make ensure-operator-sdk` — install the pinned operator-sdk.

Verification (run locally before opening a PR):

- `make verify` — `verify-fmt-imports verify-crds verify-gocilint`.
- `make verify-crds` — assert `make copy-crd` left the tree clean.
- `make verify-gocilint` — golangci-lint.
- `make verify-fmt-imports` — `golang-gci` import grouping.
- `make fmt-imports` / `make install-golang-gci` — apply the same formatting.

Run a single Go test:

```
go test ./pkg/registration/spoke/... -run TestName -v
```

The `test/` suites have their own helper scripts; consult `test/integration/` and `test/e2e/` for the entry points.

## Conventions

- Module path is `open-cluster-management.io/ocm` (**upstream**); imports must use that.
- **Upstream-tracking** fork (mirrored as `kluster-manager/ocm`). Prefer rebasing onto upstream over diverging. The default branch on origin is the AppsCode release branch (`ac-0.14.0`-style); do not force-push it without a coordinated release.
- License: Apache-2.0 (`LICENSE`, `OWNERS`, `CODE_OF_CONDUCT.md`, `SECURITY.md`). Source files carry `// Copyright Contributors to the Open Cluster Management project`.
- Sign off commits (`git commit -s`); contributions follow the DCO (`DCO`, `CONTRIBUTING.md`).
- Vendor directory is checked in **and load-bearing**: the Makefile `include`s `.mk` files from `vendor/github.com/openshift/build-machinery-go/make/`. Do not break that with a careless `go mod vendor`.
- One binary per concern. Do not merge `registration` and `work` — `pkg/singleton/spoke/` exists for that combined-agent use case, called from `multicluster-controlplane`.
- Adding a new binary: drop a new `cmd/<name>/main.go`, implementation under `pkg/<name>/`, manifests under `manifests/`, install kustomize under `deploy/`, and a `build/Dockerfile.<name>`. The images Makefile expands over `build/Dockerfile.*` automatically.
- CSV regeneration is non-trivial — bump `RELEASED_CSV_VERSION` only at a real release boundary, and re-run `make update`.
