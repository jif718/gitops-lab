# metrics-server v0.8.1 (bootstrap layer, applied manually at cluster rebuild)

- Upstream: https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.1/components.yaml (pristine)
- Local delta: NONE (verified via kubectl diff, 2026-07-16)
- Image: registry.k8s.io (not Harbor-mirrored) -> rebuild needs working egress
- Rebuild usage: kubectl apply -k bootstrap/metrics-server/
