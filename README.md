# gitops-lab

GitOps control repository for a self-hosted, 3-node Kubernetes homelab, managed
end-to-end with ArgoCD using the **App-of-Apps** pattern. This repo is the single
source of truth for platform components, application delivery — and secrets.
**No `kubectl apply` in day-2 operations.**

> **Source of truth:** self-hosted Gitea (`linux03.local:3000`). The GitHub copy
> is a **push mirror** for portfolio/backup only — ArgoCD pulls from Gitea, not
> GitHub.

---

## Architecture at a glance

| Layer | Choice |
|---|---|
| Cluster | kubeadm v1.36.0, containerd, Flannel CNI (vxlan, pod CIDR `10.244.0.0/16`) |
| Nodes | 3 × arm64 Rocky Linux 10.1 (1 control-plane + 2 workers, on Parallels/M4) |
| GitOps | ArgoCD (App-of-Apps + ApplicationSet), Image Updater (CR-driven, `method: git`) |
| CI | Jenkins (JCasC controller + dynamic Kaniko/Python agents), Gitea webhooks |
| Registry | Harbor (self-signed, `linux02.local:443`) |
| Service mesh | Istio (sidecar mode), Argo Rollouts (progressive delivery) |
| Ingress | Dual-entry: Istio ingress gateway for app traffic, NGF for platform tools |
| Observability | kube-prometheus-stack; EFK (Fluent Bit → Elasticsearch → Kibana, ECK) |
| Messaging | Strimzi-operated Kafka (KRaft mode, 3-node combined pool) |
| Storage | Longhorn (2-replica default + single-replica class for app-level replicated workloads) |
| Secrets | Sealed Secrets — every hand-created secret lives encrypted in this repo |

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

### Ingress model (dual entry)

- **Istio ingress gateway** — VIP `10.211.55.201`, for apps needing precise
  traffic control (canary). TLS via `local-tls` in `istio-ingress` namespace
  (`credentialName` requires the secret in the same namespace).
- **NGF main-gateway** — VIP `10.211.55.200`, for platform tools (Grafana,
  Kibana, Jenkins, ArgoCD, Prometheus). TLS terminated at the gateway.

---

## Repository layout

```
.
├── root-app.yaml              # ArgoCD root Application (App-of-Apps entrypoint)
├── bootstrap/                 # Pre-ArgoCD layer, applied once at cluster birth
│   ├── kubeadm-config.yaml
│   ├── flannel/               #   pristine upstream manifest + kustomize overlay
│   ├── metrics-server/        #   same pattern (upstream/ + kustomization.yaml)
│   └── argocd/                #   vendored Helm chart (.tgz) + values.yaml
├── infra/                     # Platform layer — one ArgoCD Application per dir
│   ├── metallb/               #   <name>-app.yaml + <name>-values.yaml (Helm apps)
│   ├── nginx-gateway-fabric/
│   ├── istio/                 #   istio-base / istiod / ingressgateway apps
│   │   └── manifests/         #     shared-gateway.yaml
│   ├── networking/manifests/  #   Gateway API routes/, MetalLB IP pool
│   ├── monitoring/            #   kube-prometheus-stack
│   ├── logging/               #   eck-operator / elastic-stack / fluent-bit /
│   │   └── manifests/         #     es-exporter apps; elasticsearch.yaml, kibana.yaml
│   ├── kafka/                 #   strimzi-operator app + kafka app
│   │   └── manifests/         #     Kafka CR, metrics/dashboards CMs, PodMonitor
│   ├── longhorn/manifests/    #   extra StorageClass, VolumeSnapshotClass
│   ├── jenkins/               #   Helm app + JCasC values
│   ├── argo-rollouts/  argocd-image-updater/  redis/  snapshot-controller/
│   ├── sealed-secrets/        #   Sealed Secrets controller (Helm app)
│   └── secrets/               #   SealedSecret manifests, grouped by namespace
│       └── manifests/<ns>/<name>.yaml
└── apps/                      # Business apps via ApplicationSet
    ├── apps-appset.yaml       #   Git file generator: apps/*/config.yaml
    └── <app>/config.yaml      #   per-app parameters consumed by the generator
```

### How the wiring works

- **`root-app.yaml`** is the only manually-applied Application. It discovers the
  Application manifests under `infra/` (`*-app.yaml`) plus `apps/apps-appset.yaml`;
  raw payloads (`manifests/`, values files) are owned by their child Applications,
  never double-managed by root.
- **`infra/<component>/<component>-app.yaml`** — one Application per platform
  component. Helm-based ones use ArgoCD multi-source: chart from the upstream
  Helm repo + values from this repo via `$values` ref.
- **`apps/apps-appset.yaml`** — ApplicationSet with a Git file generator; each
  `apps/<app>/config.yaml` yields one Application. Adding an app = adding one file.

---

## Secrets under GitOps

All hand-created secrets (registry creds, TLS certs, ArgoCD repo credentials,
app passwords) are stored as encrypted **SealedSecret** manifests in
`infra/secrets/manifests/`, decrypted in-cluster by the controller. The sealing
key is the only secret kept outside Git (offline backup) — restoring it is the
single manual step in disaster recovery.

Operator-generated secrets (ECK, Strimzi, Istio CAs, webhook certs) are
intentionally excluded: they are derived state owned by their controllers, and
sealing them would create ownership conflicts. See `infra/secrets/README.md`.

---

## Operations

### First-time bootstrap

```bash
# 1. kubeadm init/join, then apply bootstrap/ (Flannel, metrics-server, ArgoCD)
# 2. restore the Sealed Secrets master key (the only manual secret)
# 3. hand ArgoCD the root — it reconciles everything else from Git
kubectl apply -f root-app.yaml
argocd app sync root --grpc-web
```

### Add a new application

```bash
mkdir -p apps/<app>
# add config.yaml — the ApplicationSet generator picks it up
git add apps/<app> && git commit -m "add <app>" && git push
```

### Change a platform component

Never `kubectl apply` on ArgoCD-managed resources — edit Git, then sync.

```bash
# edit infra/<component>/*-values.yaml or infra/<component>/manifests/...
git commit -am "update <component>" && git push
argocd app sync <component> --grpc-web
```

### Add a new secret

```bash
kubectl -n <ns> create secret generic <name> --from-literal=k=v \
  --dry-run=client -o yaml | kubeseal --format yaml \
  > infra/secrets/manifests/<ns>/<name>.yaml
git add infra/secrets && git commit -m "add secret <ns>/<name>" && git push
```

### Verify cluster state

```bash
kubectl -n argocd get applications
kubectl -n argocd get applications | grep -v Synced   # ideally only the header
kubectl get sealedsecrets -A                          # all Synced=True
```

---

## Conventions & guardrails

- **GitOps discipline:** all changes go through Git commits. Direct
  `kubectl apply` on managed resources gets reverted by sync.
- **`prune: false` on root and CRD-owning Applications** — prevents cascading
  deletion of child Applications / CRD instances. Strimzi CRDs additionally
  require `ServerSideApply=true` (256KB annotation limit).
- **Istio traffic split:** DestinationRule `host` points at the root service
  (not stable); external traffic must enter via the Istio ingress gateway to
  respect VirtualService weights (NGF is not a mesh member); keep `mesh` in the
  VS `gateways` so east-west calls honor weights too.
- **Controllers reconcile what they watch:** deleting a downstream Secret does
  not trigger the Sealed Secrets controller (it watches `SealedSecret`); editing
  an ECK-owned StatefulSet gets instantly reverted (it watches everything below
  the CR). Always act on the resource the controller treats as desired state.
- **Every chart change ships as a diff:** `helm template` before/after plus
  server-side dry-run precede every commit.
- **ArgoCD CLI** always uses `--grpc-web`.

## Roadmap

- Remove the legacy "Update Helm Chart Repo" stage from the Jenkinsfile (avoids
  double-write conflict with Image Updater).
- Confirm ServiceMonitor → Grafana scrape pipeline end-to-end.
- Ansible-ize cluster initialization; evaluate Argo Workflows.
