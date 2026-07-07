# gitops-lab

GitOps control repository for a self-hosted, 3-node Kubernetes homelab, managed
end-to-end with ArgoCD using the **App-of-Apps** pattern. This repo is the single
source of truth for platform components and application delivery.

> **Source of truth:** self-hosted Gitea (`linux03.local:3000`). The GitHub copy
> is a **push mirror** for portfolio/backup only — ArgoCD pulls from Gitea, not
> GitHub.

---

## Architecture at a glance

| Layer | Choice |
|---|---|
| Cluster | kubeadm v1.36.0, containerd 2.2.3, Flannel CNI (vxlan, pod CIDR `10.244.0.0/16`) |
| Nodes | 3 × arm64 Rocky Linux 10.1 (1 control-plane + 2 workers, on Parallels/M4) |
| GitOps | ArgoCD (App-of-Apps), ArgoCD Image Updater (CR-driven, `method: git`) |
| CI | Jenkins (controller + dynamic Kaniko/Python agents), Gitea webhooks |
| Registry | Harbor (self-signed, `linux02.local:443`) |
| Service mesh | Istio (sidecar mode), Argo Rollouts (progressive delivery) |
| Ingress | Dual-entry: Istio ingress gateway for app traffic, NGF for platform tools |
| Observability | kube-prometheus-stack; EFK (Fluent Bit → Elasticsearch → Kibana, ECK) |
| Storage | Static Local PVs, `manual` StorageClass, `Retain` policy |

### GitOps delivery flow

```
Git push (Gitea) ─▶ Jenkins CI ─▶ Kaniko build ─▶ Harbor
                                                     │
                        ArgoCD Image Updater ◀───────┘ (detects new tag)
                                 │ git write-back to chart values.yaml
                                 ▼
                        ArgoCD sync ─▶ cluster ─▶ Argo Rollout + Istio VS/DR
                                                   (canary weight split)
```

### Ingress model (Method B — dual entry)

- **Istio ingress gateway** — VIP `10.211.55.201`, for apps needing precise
  traffic control (canary). TLS via `local-tls` secret in the `istio-ingress`
  namespace (`credentialName` requires the secret in the same namespace).
- **NGF main-gateway** — VIP `10.211.55.200`, for platform tools (Grafana,
  Kibana, Jenkins, ArgoCD, Prometheus). TLS terminated at the gateway.

---

## Repository layout

```
.
├── root-app.yaml          # ArgoCD root Application (App-of-Apps entrypoint)
├── platform/              # Application defs for platform components
├── apps/                  # Application defs for business apps (per-app dirs)
│   └── <app>/
│       ├── application.yaml
│       └── image-updater.yaml
├── manifests/             # Actual K8s payloads for platform components
│   ├── istio/  logging/  networking/  redis/  storage/
├── helm-values/           # Helm values per platform chart
├── bootstrap/             # Cluster bootstrap material (kubeadm, Flannel)
├── secrets/               # Secret handling notes (Sealed Secrets planned)
└── README.md
```

### How the App-of-Apps wiring works

- **`root-app.yaml`** is applied manually (bootstrap). Its source is the repo
  root with `directory.recurse: true`, excluding non-Application directories:

  ```yaml
  source:
    path: .
    directory:
      recurse: true
      exclude: '{manifests/**,helm-values/**,bootstrap/**,secrets/**}'
  ```

  So `root` only renders Application definitions under `platform/` and `apps/`;
  it never double-manages the payloads in `manifests/`, which are owned by their
  own Applications.

- **`platform/*-app.yaml`** — each is an ArgoCD Application whose `source.path`
  points at the matching `manifests/<component>` (or a Helm chart + values).
- **`apps/<app>/application.yaml`** — points at the app's chart repo in Gitea;
  `image-updater.yaml` is the `ImageUpdater` CR driving tag write-back.

---

## Platform components (`platform/`)

Managed as individual ArgoCD Applications: `istio-base`, `istiod`,
`istio-ingressgateway`, `shared-gateway`, `nginx-gateway-fabric`, `metallb`,
`kube-prometheus-stack`, `fluent-bit`, `eck-operator`, `logging`, `redis`,
`storage`, `argo-rollouts`, `networking`.

## Applications (`apps/`)

| App | Type | Delivery | Status |
|---|---|---|---|
| `flask-demo-1` | Argo Rollout | Istio VS/DR canary weight split | validated end-to-end |
| `flask-demo-2` | Deployment | rolling update | deployed |
| `html5demos` | Deployment | rolling update | deployed |
| `spring-petclinic` | Deployment | rolling update | deployed |

---

## Operations

### First-time bootstrap

```bash
# root is applied by hand; it then manages everything else
kubectl apply -f root-app.yaml
argocd app sync root --grpc-web
```

### Add a new application

```bash
mkdir -p apps/<app>
# add application.yaml (points at the app's chart repo) + image-updater.yaml
git add apps/<app> && git commit -m "add <app>" && git push
# root picks it up via recurse on next sync (or webhook-triggered sync)
argocd app sync root --grpc-web
```

### Change a platform component

Never `kubectl apply` on ArgoCD-managed resources — edit Git, then sync.

```bash
# edit manifests/<component>/... or helm-values/<chart>/values.yaml
git commit -am "update <component>" && git push
argocd app sync <component> --grpc-web
```

### Verify cluster state

```bash
# all Applications should be Synced/Healthy
kubectl -n argocd get applications
kubectl -n argocd get applications | grep -v Synced   # ideally only the header
```

---

## Conventions & guardrails

- **GitOps discipline:** all changes go through Git commits. Direct
  `kubectl apply` on managed resources gets reverted by sync.
- **`prune: false` on root and CRD-owning Applications** — prevents cascading
  deletion of child Applications / CRD instances.
- **CRD-owning apps** (e.g. `kube-prometheus-stack`) use manual sync with
  `ServerSideApply=true`, never global prune.
- **Istio traffic split:** DestinationRule `host` points at the root service
  (not stable); external traffic must enter via the Istio ingress gateway to
  respect VirtualService weights (NGF is not a mesh member); keep `mesh` in the
  VS `gateways` so east-west calls honor weights too.
- **ArgoCD CLI** always uses `--grpc-web`.

---

## Known gaps / roadmap

- `elasticsearch-exporter-creds` (`logging` ns) is still hand-applied — the last
  GitOps gap; Sealed Secrets planned to close it.
- Remove the legacy "Update Helm Chart Repo" stage from the Jenkinsfile (avoids
  double-write conflict with Image Updater).
- Confirm ServiceMonitor → Grafana scrape pipeline end-to-end.

