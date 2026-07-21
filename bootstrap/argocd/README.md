# ArgoCD v3.4.3 (helm chart argo-cd-9.5.20, bootstrap layer)

Installed manually via Helm -- NOT ArgoCD-managed (chicken-and-egg).
Day-2 upgrades:
  helm upgrade argocd argo/argo-cd --version 9.5.20 -n argocd -f values.yaml
  (always explicit -f, never --reuse-values)

## Rebuild sequence (after kubeadm + bootstrap/flannel; Gitea & Harbor must be up)
1. helm install argocd argo/argo-cd --version 9.5.20 -n argocd --create-namespace -f values.yaml
2. Manual secrets (see ../../secrets/README.md): gitea repo creds (argocd repo add,
   --grpc-web), harbor-creds (dockerconfigjson, used by image-updater), harbor-ca
3. kubectl apply -f ../../root-app.yaml   # App-of-Apps seed; everything else syncs from Git

## Notes
- Chart tgz vendored: node egress to github/quay is unreliable
- Images pull from quay.io/ghcr.io/ecr-public (not Harbor-mirrored) -- rebuild needs egress
