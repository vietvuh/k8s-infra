# AGENTS.md

## Purpose

This repo provisions a local Kubernetes platform for running OpenSearch as an
observability backend (logs + metrics), fronted by Istio in Ambient mode.

- **Cluster**: K3d cluster with 7 nodes total — 1 `server` (control-plane)
  and 6 `agent` (worker) nodes.
- **OpenSearch node-pool**: 3 of the 6 agent nodes are dedicated to
  OpenSearch, under a dedicated `opensearch` namespace. Each OpenSearch
  worker/data node is scheduled on its own separate node within this
  node-pool (one instance per node — no co-locating replicas on the same
  node). The remaining 3 agent nodes are general-purpose, for everything
  else (Istio, OTEL collectors, etc.).
- **Service mesh**: K8s infrastructure runs with Istio in **Ambient Mode**
  (not sidecar mode).
- **Observability pipeline**: OpenSearch is the log and metrics management
  system. Telemetry is collected via **OpenTelemetry (OTEL)** and shipped
  into OpenSearch, which is installed under the `opensearch` namespace.

When adding manifests, Helm values, or automation, keep these constraints in
mind: node-pool/taint-toleration setup must keep OpenSearch nodes isolated
from general workloads, and OpenSearch pods must anti-affine across the 3
dedicated nodes (one per node).

## Coding conventions

- **Go code**: format with `gofmt` (standard Go tooling — this means Go
  source keeps `gofmt`'s tab-indentation; the 2-space rule below does not
  apply to `.go` files).
- **Everything else** (YAML manifests, Helm charts, Kustomize, JSON, shell
  scripts, Markdown, etc.): indent with **2 spaces**, never tabs.
- **Line endings**: always **LF** (`\n`). Never CRLF or CR, in any file.
