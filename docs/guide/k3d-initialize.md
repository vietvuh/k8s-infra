# Initializing the K3d Cluster

This guide walks through bootstrapping the local **K3d** cluster this repo
targets: 7 nodes total, with 3 of them dedicated to a labeled/tainted
`opensearch` node-pool. Istio installation and OpenSearch installation are
covered in separate guides — this document only covers cluster bring-up.

## Cluster topology

| Role                      | Count | Purpose                                                                                      |
| -------------------------- | ----- | --------------------------------------------------------------------------------------------- |
| `server` (control-plane)   | 1     | k3s control-plane (single node, no HA — this is a local dev cluster)                          |
| `agent` (worker)           | 3     | Dedicated **OpenSearch node-pool** — labeled `node-pool=opensearch` and tainted `dedicated=opensearch:NoSchedule` so only OpenSearch pods (with a matching toleration) are scheduled here |
| `agent` (worker)           | 3     | General-purpose workers — Istio control plane, OTEL collectors, and everything else            |

Total: **7 nodes**. Each OpenSearch worker instance is later scheduled
one-per-node across the 3 dedicated agents via pod anti-affinity (see the
OpenSearch install guide).

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) running locally
- [k3d](https://k3d.io/) — this guide was validated with `v5.8.3`
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)

```bash
docker version
k3d version
kubectl version --client
```

## 1. Create the cluster

```bash
k3d cluster create k8s-infra-cluster \
  --servers 1 \
  --agents 6 \
  --k3s-node-label "node-pool=opensearch@agent:0,1,2" \
  --k3s-arg "--node-taint=dedicated=opensearch:NoSchedule@agent:0,1,2" \
  --wait
```

- `--servers 1` / `--agents 6` gives the 7-node topology above.
- `--k3s-node-label` labels agents `0,1,2` (the first 3 agents k3d creates)
  with `node-pool=opensearch`, so workloads can target them with a
  `nodeSelector` / `nodeAffinity`. Agents `3,4,5` are left unlabeled/general.
- `--k3s-arg "--node-taint=..."` applies a matching `NoSchedule` taint at
  kubelet registration time, so only pods with an explicit toleration for
  `dedicated=opensearch` land on agents `0,1,2`.

## 2. Point kubectl at the cluster

`k3d cluster create` updates your default kubeconfig and switches context
automatically. Confirm it:

```bash
kubectl config current-context
# k3d-k8s-infra-cluster
```

## 3. Verify nodes, labels, and taints

```bash
kubectl get nodes -o wide
```

```bash
kubectl get nodes -l node-pool=opensearch
```

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

You should see 1 `server` node and agents `3,4,5` without the taint, and
agents `0,1,2` carrying:

```
Taints:      dedicated=opensearch:NoSchedule
Labels:      node-pool=opensearch
```

## 4. Create the `opensearch` namespace

```bash
kubectl create namespace opensearch
```

## 5. Sanity-check scheduling isolation

A quick throwaway pod without a toleration should **not** be schedulable on
the OpenSearch node-pool:

```bash
kubectl run test-noschedule --image=busybox --restart=Never \
  --overrides='{"spec":{"nodeSelector":{"node-pool":"opensearch"}}}' \
  -- sleep 3600

kubectl get pod test-noschedule
# STATUS should be Pending (unschedulable: node(s) had untolerated taint)

kubectl delete pod test-noschedule
```

## Tearing down

```bash
k3d cluster delete k8s-infra-cluster
```

## Next steps

- Install Istio in Ambient mode across the cluster.
- Install OpenSearch into the `opensearch` namespace, scheduling one worker
  per dedicated node via `nodeSelector: {node-pool: opensearch}`, a
  toleration for `dedicated=opensearch:NoSchedule`, and pod anti-affinity to
  keep exactly one instance per node.
