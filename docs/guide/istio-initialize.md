# Installing Istio in Ambient Mode

This guide installs [Istio](https://istio.io/) using the **Ambient**
data-plane mode (no sidecars — a per-node `ztunnel` proxy plus optional
per-namespace/service `waypoint` proxies) onto the cluster created in
[k3d-initialize.md](k3d-initialize.md).

## Why Ambient (not sidecars)

Ambient mode moves the mesh data plane out of application pods:

- **`istio-cni`** — a per-node DaemonSet that transparently redirects pod
  traffic into the mesh (no init-container injection needed).
- **`ztunnel`** — a per-node DaemonSet providing mTLS and L4 routing for
  every pod on that node ("secure overlay").
- **`waypoint`** — optional, per-namespace/service-account proxies for L7
  features (retries, HTTP routing, authorization policy on HTTP semantics).

Because `istio-cni` and `ztunnel` are node-level DaemonSets, **they must run
on every node** — including the 3 tainted OpenSearch nodes from the k3d
guide — or pods scheduled there won't get mesh redirection/mTLS. This is
called out explicitly below.

## Prerequisites

- The 7-node `k8s-infra-cluster` from [k3d-initialize.md](k3d-initialize.md)
  is up and `kubectl config current-context` points at it.
- [Helm](https://helm.sh/) v3
- (Optional, recommended for verification/debugging) [`istioctl`](https://istio.io/latest/docs/setup/getting-started/#download):

  ```bash
  curl -L https://istio.io/downloadIstio | sh -
  export PATH="$PWD/istio-<version>/bin:$PATH"
  ```

## 1. Add the Istio Helm repo

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

Pick a version to pin (avoid installing an untested `latest`):

```bash
helm search repo istio/base --versions | head -5
```

Export it for reuse below:

```bash
export ISTIO_VERSION=<x.y.z>   # e.g. 1.23.2
```

## 2. Install `istio-base` (CRDs + cluster roles)

```bash
kubectl create namespace istio-system

helm install istio-base istio/base \
  -n istio-system \
  --version "$ISTIO_VERSION" \
  --set defaultRevision=default \
  --wait
```

## 3. Install `istiod` (control plane)

```bash
helm install istiod istio/istiod \
  -n istio-system \
  --version "$ISTIO_VERSION" \
  --set profile=ambient \
  --wait
```

## 4. Install `istio-cni`

`istio-cni` needs to drop its plugin binary wherever the node's container
runtime actually looks for CNI plugins, and needs its config dropped
wherever that runtime reads CNI conflists from. **Don't rely on
`--set global.platform=k3s`** to get these right — on k3d/k3s (confirmed
against k3s `v1.31.5+k3s1`, chart `cni-1.30.3`) that preset resolves to
`/var/lib/rancher/k3s/data/cni`, but containerd's actual configured
`bin_dir` is `/bin` (check a node's
`/var/lib/rancher/k3s/agent/etc/containerd/config.toml` if unsure). A
mismatch here doesn't fail loudly at install time — it fails silently,
later, for every pod that tries to start on that node. Set both paths
explicitly instead:

```bash
helm install istio-cni istio/cni \
  -n istio-system \
  --version "$ISTIO_VERSION" \
  --set profile=ambient \
  --set cniBinDir=/bin \
  --set cniConfDir=/var/lib/rancher/k3s/agent/etc/cni/net.d \
  --set-json 'tolerations=[{"key":"dedicated","operator":"Equal","value":"opensearch","effect":"NoSchedule"}]' \
  --wait
```

The `tolerations` entry lets the DaemonSet schedule onto the tainted
OpenSearch node-pool.

**Verify the binary actually landed before continuing.** This is the step
that would have caught the path mismatch immediately instead of after many
minutes of pods silently failing to start. The `install-cni` container image
is minimal (no shell, no `test`/`ls`), so `kubectl exec` into it won't work
— check from the k3d node container itself instead, which does have a
regular shell:

```bash
for node in $(docker ps --filter "label=k3d.cluster=k8s-infra-cluster" --format '{{.Names}}' | grep -v serverlb); do
  echo -n "$node: "
  docker exec "$node" test -f /bin/istio-cni && echo OK || echo "MISSING"
done
```

Do not move on to step 5 until this passes on every node. If it doesn't,
double check `cniBinDir` against what's actually in the node's containerd
config rather than retrying the same install.

## 5. Install `ztunnel`

Same toleration is needed here for the same reason — `ztunnel` must run on
the OpenSearch nodes to secure/route their traffic.

```bash
helm install ztunnel istio/ztunnel \
  -n istio-system \
  --version "$ISTIO_VERSION" \
  --set-json 'tolerations=[{"key":"dedicated","operator":"Equal","value":"opensearch","effect":"NoSchedule"}]' \
  --wait
```

## 6. Verify the control plane and DaemonSets

```bash
kubectl get pods -n istio-system -o wide
kubectl get pods -n istio-system -l app=ztunnel -w
```

`istiod` should be `Running` (1+ replicas), and `istio-cni-node` /
`ztunnel` should each have **7/7** pods — one per node, confirming they
landed on the `server` node, the 3 general `agent` nodes, and all 3 tainted
`agent` (OpenSearch) nodes:

```bash
kubectl get daemonset -n istio-system
```

```
NAME             DESIRED   CURRENT   READY
istio-cni-node   7         7         7
ztunnel          7         7         7
```

If either shows fewer than 7, check `kubectl describe daemonset <name> -n
istio-system` for a `FailedScheduling` event — almost always a missing
toleration for the `dedicated=opensearch` taint.

## 7. Enable ambient for the `opensearch` namespace

Namespaces opt in to the ambient data plane with a label — nothing is
redirected into the mesh until this is set:

```bash
kubectl label namespace opensearch istio.io/dataplane-mode=ambient
```

Repeat for any other namespace you want meshed (e.g. an OTEL collector
namespace, once it exists).

## 8. Verify ambient redirection

With `istioctl` installed:

```bash
istioctl ztunnel-config workloads
```

Pods in a labeled namespace should show up with an assigned `ztunnel` once
they're running. Without `istioctl`, an indirect check is that new pods in
the `opensearch` namespace get an `ambient.istio.io/redirection: enabled`
annotation from the CNI:

```bash
kubectl get pod <pod-name> -n opensearch -o jsonpath='{.metadata.annotations}'
```

## Uninstalling

```bash
helm uninstall ztunnel istio-cni istiod istio-base -n istio-system
kubectl delete namespace istio-system
```

## Next steps

- (Optional) Install a `waypoint` proxy for the `opensearch` namespace if
  you need L7 traffic policy (HTTP retries/routing, L7 authorization) beyond
  the mTLS/L4 that `ztunnel` already provides.
- Install OpenSearch into the `opensearch` namespace and OTEL collectors to
  feed it — see the OpenSearch install guide.
