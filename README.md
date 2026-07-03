# jif-lab-platform

GitOps repository for a self-hosted Kubernetes platform, built as a hands-on lab for
platform engineering / SRE practices. Every platform component in the cluster is
declared here and continuously reconciled by Argo CD — no `helm install`, no
`kubectl apply` against managed resources.

> Application workloads (Helm charts for demo apps) live in a separate repo
> following the platform/app repo split: [`jif-lab-apps`](../jif-lab-apps).

## Cluster at a Glance

| | |
|---|---|
| Topology | 3-node kubeadm cluster (1 control-plane + 2 workers) |
| Version | Kubernetes v1.36, containerd, Flannel CNI |
| OS / Arch | Rocky Linux 10, arm64 (Parallels VMs on Apple Silicon) |
| GitOps | Argo CD (App-of-Apps) + Argo CD Image Updater (CRD mode) |
| Ingress | NGF (Gateway API) for platform tools, Istio ingress gateway for mesh apps |
| Progressive delivery | Argo Rollouts + Istio VirtualService weight-based canary |
| Observability | kube-prometheus-stack, EFK (Fluent Bit → ECK-managed Elasticsearch → Kibana) |
| CI | Jenkins with dynamic K8s agents, Kaniko image builds → Harbor registry |
| Load balancing | MetalLB (L2), VIPs `10.211.55.200` (NGF) / `10.211.55.201` (Istio) |

## Repository Layout

```
.
├── apps/                  # Argo CD Application CRs only (App-of-Apps root scans this dir)
│   ├── root-app.yaml      # root Application, prune disabled to avoid cascading deletes
│   └── *-app.yaml         # one Application per platform component
├── helm-values/           # values files, one dir per Helm chart
│   └── <chart-name>/values.yaml
├── manifests/             # raw manifests, one dir per Application
│   ├── istio/             # shared Istio Gateway CR (wildcard *.local, HTTP+HTTPS)
│   ├── logging/           # ECK Elasticsearch/Kibana CRs + elasticsearch-exporter
│   ├── networking/        # NGF Gateway, HTTPRoutes, MetalLB IP pool
│   ├── redis/             # Redis StatefulSet + exporter + ServiceMonitor
│   └── storage/           # StorageClass + static Local PVs
├── bootstrap/             # pre-GitOps cluster init (kubeadm config, CNI) — reference only
└── secrets/               # placeholder; Sealed Secrets migration planned (see Roadmap)
```

Conventions:

- `apps/` contains **only** Application definitions; all real manifests live under `manifests/`.
- Helm-based Applications use the multi-source pattern: chart from upstream repo,
  values referenced via `$values/helm-values/<chart>/values.yaml` from this repo.
- Sync waves order dependent components (istio-base → istiod → ingress gateway → gateway CR).

## GitOps Principles Applied

**Argo CD is the sole deployment path.** Direct `kubectl edit` / `helm upgrade` on
managed resources is prohibited; drift is corrected by selfHeal or surfaced as OutOfSync.

**Safe adoption of pre-existing Helm releases.** Components originally installed by
hand were adopted without downtime:

1. Extract live values with `helm get values`.
2. Create the Application **without** syncPolicy; verify the diff shows only
   tracking-annotation additions (no changes, no deletions).
3. Enable auto-sync, then remove the Helm release secret
   (`kubectl delete secret -l owner=helm,name=<release>`) — never `helm uninstall`.

**Controller-owned fields are excluded from reconciliation.** `ignoreDifferences`
covers runtime-written fields, e.g. Argo Rollouts updating VirtualService weights
during a canary, ECK writing orchestration hints, and Local PV `claimRef` metadata.
CRD-owning Applications run with `prune: false`.

**Server-Side Apply where CRDs exceed annotation limits.** kube-prometheus-stack
syncs with `ServerSideApply=true`; diffs are verified with
`argocd app diff --server-side-generate --hard-refresh`.

## Traffic Architecture

Two ingress tiers, chosen per workload by traffic-governance needs:

- **NGF / Gateway API** (`10.211.55.200`) — platform tools (Argo CD, Grafana, Kibana,
  Jenkins, Prometheus) and apps without canary requirements. TLS terminates at the
  gateway with a wildcard `*.local` cert (self-managed CA).
- **Istio ingress gateway** (`10.211.55.201`) — workloads needing precise
  weight-based canary. A shared Istio `Gateway` CR fronts them; VirtualServices
  bind to both `istio-ingress/shared-gateway` and `mesh` to keep east-west routing.

A practical lesson encoded in this design: VirtualService weights only apply on the
outbound side of a mesh-member sidecar. Traffic entering through a non-mesh ingress
(NGF) bypasses weights and degrades to endpoint round-robin — hence the dedicated
Istio ingress tier for canary workloads.

## CI/CD Flow

```
push → Gitea webhook → Jenkins (dynamic K8s agents)
     → Kaniko builds image → Harbor registry
     → Argo CD Image Updater detects new tag (allowTags regexp on build format)
     → git write-back to app chart values.yaml (method: git)
     → Gitea webhook → Argo CD sync → Argo Rollouts canary via Istio
```

CI and CD are strictly separated: Jenkins never touches the cluster; Image Updater
owns the write-back; Argo CD owns deployment.

## Roadmap

- **Sealed Secrets** — move hand-applied Secrets (registry creds, exporter creds,
  TLS) into Git as `SealedSecret` resources; last remaining gap for full
  cluster-rebuild-from-Git.
- **ES Local PV nodeAffinity** — switch from role-label to hostname pinning before
  scaling Elasticsearch beyond one node.
- **Cilium evaluation** — on a separate test cluster, as a Flannel replacement.

## Related

- [`jif-lab-apps`](../jif-lab-apps) — application Helm charts (Argo Rollouts canary demo, etc.)
- AWS EKS migration project: [`jif718/aws-migration`](https://github.com/jif718/aws-migration)

