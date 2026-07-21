# Secrets Management

本集群所有手工创建的 Secret 已通过 **Sealed Secrets** 纳入 GitOps 管理。

- Controller: `kube-system/sealed-secrets-controller` (ArgoCD app: `sealed-secrets`)
- 密文清单: `infra/secrets/manifests/<namespace>/<name>.yaml` (ArgoCD app: `secrets`)
- Operator 生成的 Secret (ECK / Strimzi / Istio / NGF / Longhorn / MetalLB /
  prometheus-operator / ArgoCD 自身) 不入 Git,由各自 controller 管理

## 新增 Secret 标准流程

```bash
kubectl -n <ns> create secret generic <name> --from-literal=key=value \
  --dry-run=client -o yaml \
  | kubeseal --format yaml > infra/secrets/manifests/<ns>/<name>.yaml
# commit + push; ArgoCD syncs, controller decrypts into a real Secret
```

注意:
- ArgoCD 仓库凭据必须保留 label:
  `argocd.argoproj.io/secret-type: repository` (repo-*) 或 `repo-creds` (creds-*)
- 修改已有 Secret = 重新 kubeseal 覆盖同一文件后提交,不要 kubectl edit

## 灾备恢复

集群重建时,唯一需要手工恢复的秘密是 sealing key(离线保存在 Mac,不在本仓库):

```bash
# 1. restore the sealing key BEFORE the controller starts using a new one
kubectl apply -f sealed-secrets-master-key.yaml

# 2. install sealed-secrets controller (via ArgoCD bootstrap or helm)
# 3. restart controller to pick up the restored key
kubectl -n kube-system rollout restart deploy sealed-secrets-controller

# 4. ArgoCD syncs the secrets app — all Secrets decrypt automatically
```

## Sealing Key 轮转与备份

- Controller 默认每 30 天生成新 key(旧 key 保留用于解密存量密文)
- 每次轮转后需重新备份全部 key:
  `kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-master-key.yaml`