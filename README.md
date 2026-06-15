# jif-lab-platform

GitOps repo for the homelab Kubernetes cluster (3 nodes, kubeadm, v1.36.0).
Scope: platform/infra only. Business apps live in `jif-lab-apps`.

## Layout
- `bootstrap/`   kubeadm + flannel. Applied manually, owned by kubeadm. NOT ArgoCD.
- `helm-values/` inputs for manual `helm upgrade -f`. NOT ArgoCD-synced.
- `storage/`     manual PVs (StorageClass `manual`, no-provisioner). Applied manually.
- `apps/`        ArgoCD Application definitions (App-of-Apps).
- `manifests/`   declarative resources ArgoCD auto-syncs (routes, metallb pool, ECK CRs...).
- `secrets/`     encrypted only (Sealed Secrets / SOPS). No plaintext.

## What ArgoCD manages vs not
ArgoCD-synced:   manifests/**
Manual (helm):   helm-values/** , bootstrap/** , storage/**
Reason: drift on declarative manifests is an error; helm/kubeadm/PV own their own lifecycle.

## Bootstrap order (rebuild)
kubeadm init -> flannel -> metallb -> nginx-gateway-fabric -> argocd -> apply apps/root-app.yaml
