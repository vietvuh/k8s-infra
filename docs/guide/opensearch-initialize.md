# Installing OpenSearch

This guide installs [OpenSearch](https://opensearch.org/) `v3.7.0` — the
cluster itself plus OpenSearch Dashboards — into the `opensearch` namespace,
pinned to the dedicated OpenSearch node-pool from
[k3d-initialize.md](k3d-initialize.md), and exposes both through Traefik
(bundled with k3s) as:

- `http://api.opensearch.local:8080` — the OpenSearch REST API
- `http://dashboard.opensearch.local:8080` — OpenSearch Dashboards

This is **two separate Helm charts/releases** (`opensearch/opensearch` and
`opensearch/opensearch-dashboards`), so this repo has **two values files**:
[deployment/opensearch-values.yaml](../../deployment/opensearch-values.yaml)
and
[deployment/opensearch-dashboards-values.yaml](../../deployment/opensearch-dashboards-values.yaml).
They can't be merged into one file even though the charts share some key
names (e.g. `ingress`) — the two charts expect different shapes under that
key (the `opensearch` chart wants `ingress.hosts` as a flat list of
hostnames; `opensearch-dashboards` wants a list of `{host, paths}` objects),
so a single shared block would misconfigure whichever chart reads it second.

## Prerequisites

- The `k8s-infra-cluster` from [k3d-initialize.md](k3d-initialize.md) is up,
  including the `-p "8080:80@loadbalancer"` port mapping (add it with `k3d
  cluster edit` if the cluster predates that change — see that guide).
- [Istio Ambient mode](istio-initialize.md) installed (not a hard
  requirement for OpenSearch itself, but this repo's `opensearch` namespace
  is already labeled `istio.io/dataplane-mode=ambient`, so OpenSearch pods
  will pick up mTLS automatically).
- [Helm](https://helm.sh/) v3
- The `opensearch` namespace already exists (created in
  [k3d-initialize.md](k3d-initialize.md)).

## 1. Add the OpenSearch Helm repo

```bash
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm repo update opensearch
```

## 2. Create the admin credentials Secret

OpenSearch 2.12+/3.x refuses to bootstrap its bundled security-plugin demo
config without a custom admin password — and that password must **not** be
committed to git as plaintext in a values file. Create it as a Secret
instead, imperatively:

```bash
kubectl create secret generic opensearch-admin-credentials \
  -n opensearch \
  --from-literal=password='<pick-a-strong-password>'
```

Both values files reference this Secret (`OPENSEARCH_INITIAL_ADMIN_PASSWORD`
for the cluster, `OPENSEARCH_PASSWORD` for Dashboards) rather than embedding
the value directly.

## 3. Install the OpenSearch cluster

Values: [deployment/opensearch-values.yaml](../../deployment/opensearch-values.yaml).
Key points:

- `replicas: 3`, `nodeSelector: {node-pool: opensearch}`, the matching
  `tolerations` entry, and `antiAffinity: "hard"` together put exactly one
  OpenSearch pod on each of the 3 dedicated nodes — the same
  taint/toleration/label pattern used for `istio-cni`/`ztunnel` in
  [istio-initialize.md](istio-initialize.md).
- `roles: [master, data, ingest, remote_cluster_client]` — all nodes are
  combined master+data+ingest, appropriate for a 3-node dev cluster (no
  separate dedicated master nodes).
- `persistence` uses the cluster's default StorageClass (`local-path`,
  bundled with k3d/k3s) — fine for local dev, not multi-node-redundant
  storage.
- `service.annotations` + the `ServersTransport` in `extraObjects` are what
  let Traefik proxy plain HTTP (browser-facing) into the HTTPS-only REST API
  (security plugin + self-signed demo certs) — see "Verify ingress" below
  for how to confirm this actually works, since it's the newest/least
  battle-tested part of this setup.

```bash
helm install opensearch opensearch/opensearch \
  -n opensearch \
  --version 3.7.0 \
  -f deployment/opensearch-values.yaml \
  --wait
```

This can take a few minutes — each of the 3 pods needs its own
`local-path` PersistentVolume provisioned and the cluster needs to form
before `--wait` is satisfied.

Verify:

```bash
kubectl get pods -n opensearch -o wide
```

You should see 3 pods (`opensearch-cluster-master-0/1/2`), one per dedicated
node, all `Running`.

## 4. Install Dashboards

Values: [deployment/opensearch-dashboards-values.yaml](../../deployment/opensearch-dashboards-values.yaml).
Deliberately **not** pinned to the OpenSearch node-pool — Dashboards is
stateless with no data-locality need, so it's left to land on the
general-purpose nodes instead of competing with OpenSearch itself for the 3
dedicated nodes.

```bash
helm install opensearch-dashboards opensearch/opensearch-dashboards \
  -n opensearch \
  --version 3.7.0 \
  -f deployment/opensearch-dashboards-values.yaml \
  --wait
```

## 5. Point your hosts file at the cluster

`/etc/hosts` only maps a hostname to an IP — it doesn't carry a port. The
`:8080` belongs in the URL you browse to, not in the hosts entry:

```
127.0.0.1 api.opensearch.local
127.0.0.1 dashboard.opensearch.local
```

If you're driving this from Windows against WSL2 (as covered when we set up
kubeconfig access earlier), add the same two lines to
`C:\Windows\System32\drivers\etc\hosts` — WSL2's default NAT networking
forwards `localhost:8080` from Windows into WSL2 automatically, same as it
did for the kubeconfig access case.

## 6. Verify

**API** (basic auth required — `admin` / the password from step 2):

```bash
curl -u admin:'<your-password>' http://api.opensearch.local:8080/_cluster/health?pretty
```

Expect a JSON body with `"status": "green"` (or `"yellow"` briefly while
shards initialize).

**Dashboards**:

```bash
curl -I http://dashboard.opensearch.local:8080/app/home
```

Expect `HTTP/1.1 200 OK`, or open the URL in a browser and log in with the
same `admin` credentials.

### If the API route doesn't work

The Traefik `ServersTransport`/backend-HTTPS setup in
`opensearch-values.yaml` is the documented mechanism
([Traefik Kubernetes Service annotations reference](https://doc.traefik.io/traefik-hub/api-gateway/reference/routing/kubernetes/http/services/ref-svc-annotations))
but hasn't been exercised end-to-end against this specific cluster yet.
If `curl` against `api.opensearch.local:8080` hangs or 502s:

```bash
kubectl get serverstransport -n opensearch
kubectl describe ingress opensearch-cluster-master -n opensearch
kubectl logs -n kube-system deploy/traefik | tail -50
```

A 502 with a TLS-handshake complaint in the Traefik logs usually means the
`ServersTransport` reference name doesn't match — double check it's
`opensearch-insecure-backend-tls@kubernetescrd` (namespace `opensearch` +
object name `insecure-backend-tls`, joined with a hyphen — Traefik always
requires the namespace prefix in this particular annotation, even for a
same-namespace reference).

## Uninstalling

```bash
helm uninstall opensearch-dashboards opensearch -n opensearch
kubectl delete secret opensearch-admin-credentials -n opensearch
```

(PersistentVolumeClaims created by the StatefulSet aren't deleted by `helm
uninstall` — `kubectl delete pvc -n opensearch -l app.kubernetes.io/instance=opensearch`
if you want to reclaim that disk space too.)

## Next steps

- Feed logs and traces into OpenSearch via OTEL — see the (upcoming) OTEL
  collector guide for how the Istio ambient mesh's trace data and workload
  logs actually get here.
