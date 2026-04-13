# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cilium is an eBPF-based networking, observability, and security solution for Kubernetes. The userspace code is Apache-2.0 licensed; BPF kernel programs are dual-licensed GPL-2.0 OR BSD-2-Clause.

## Build Commands

```bash
make all              # precheck -> build -> postcheck
make build            # build all components
make precheck         # formatting, log newline checks, code generation verification
make postcheck        # post-build checks
make debug            # build with NOOPT=1 NOSTRIP=1 for debugging
```

## Lint

```bash
golangci-lint run --verbose --modules-download-mode=vendor
# For Kubernetes API packages:
golangci-lint run -c tools/golangci-lint-kubeapi/golangci-lint-kubeapi.yaml ./pkg/k8s/apis/cilium.io/...
```

Key linters enabled (see `.golangci.yaml`): `err113`, `errorlint`, `forbidigo`, `goheader`, `gosec`, `govet`, `staticcheck`, `testifylint`, `unused`. Imports of `logrus`, `gopkg.in/yaml.v2`, and `gopkg.in/yaml.v3` are forbidden (use `slog` and alternatives).

## Test Commands

```bash
make tests-privileged          # all unit tests (requires elevated privileges)
make tests-privileged-only     # only tests requiring elevated privileges
make integration-tests         # non-privileged + integration tests (auto-starts etcd)
make bench                     # benchmarks
make start-kvstores            # start etcd container for integration tests
make stop-kvstores             # stop kvstore containers
```

**Key Makefile variables:**
- `TESTPKGS` — space-separated Go packages to test (default: `./...`)
- `BENCH` — benchmark filter pattern
- `JUNIT_PATH` — path for JUnit XML output
- `SKIP_KVSTORES=true` — skip etcd startup

**Running a single test:**
```bash
# Specific test function
go test ./pkg/endpoint/... -run TestFunctionName -timeout 720s -v

# Specific package with coverage
go test ./pkg/endpoint/... -timeout 720s -coverprofile=coverage.out

# Privileged tests
PRIVILEGED_TESTS=true go test ./pkg/... -run TestName -timeout 720s

# Specific packages via Makefile
make tests-privileged TESTPKGS=./pkg/endpoint
```

Integration tests in `/test/` use the Ginkgo v2 framework.

## Code Architecture

### Main Binaries

| Binary | Directory | Role |
|--------|-----------|------|
| `cilium-agent` | `daemon/` | Main control-plane daemon; manages endpoints, eBPF programs, policies |
| `cilium-operator` | `operator/` | Cluster-wide operator (variants: generic, aws, azure, alibabacloud) |
| `hubble` / `hubble-relay` | `hubble/`, `hubble-relay/` | Observability: real-time flow visibility and metrics |
| `clustermesh-apiserver` | `clustermesh-apiserver/` | Multi-cluster connectivity server |
| `cilium-health` | `cilium-health/` | Cluster health daemon |
| `cilium-dbg` | `cilium-dbg/` | BPF inspection and policy debugging tool |

### Core Package Areas (`pkg/`)

- **`pkg/datapath`** — Dataplane abstraction layer (BPF loader, TC/XDP hooks, iptables, tunnel, maps)
- **`pkg/endpoint`** — Endpoint lifecycle, policy enforcement, eBPF regeneration
- **`pkg/policy`** — Policy engine: network policy, FQDN/DNS policies, identity-based security
- **`pkg/k8s`** — Kubernetes integration: watchers, informers, resource types, label handling
- **`pkg/bpf`** — eBPF map management and program loading utilities
- **`pkg/kvstore`** — Key-value store abstraction (etcd, consul)
- **`pkg/option`** — Daemon configuration flags and feature toggles
- **`pkg/bgp`** — BGP routing support
- **`pkg/auth`** — SPIFFE/mTLS mutual authentication
- **`pkg/ciliumenvoyconfig`** — Envoy integration for L7 policy enforcement
- **`pkg/clustermesh`** — Multi-cluster service discovery and cross-cluster policies
- **`pkg/aws`, `pkg/azure`, `pkg/alibabacloud`** — Cloud provider integrations

### BPF Kernel Programs (`bpf/`)

eBPF C programs loaded by the agent into the Linux kernel for the actual dataplane. Compiled separately from Go code.

### Dependency Injection

The codebase uses the [Hive](https://github.com/cilium/hive) framework for dependency injection and lifecycle management. Components declare dependencies via `hive.Module` / `cell.Module` and are wired together in main entry points.

### Networking Modes

Cilium supports overlay (VXLAN/Geneve) and native routing modes. The datapath is selected at startup and influences which BPF programs are loaded.
