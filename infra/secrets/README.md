# Secrets Management

本集群所有手工创建的 Secret 已通过 **Sealed Secrets** 纳入 GitOps 管理。

- Controller: `kube-system/sealed-secrets-controller` (ArgoCD app: `sealed-secrets`)
- 密文清单: `infra/secrets/manifests/<namespace>/<name>.yaml` (ArgoCD app: `secrets`)
- Operator 生成的 Secret (ECK / Strimzi / Istio / NGF / Longhorn / MetalLB /
  prometheus-operator / ArgoCD 自身) 不入 Git,由各自 controller 管理

以下命令均在 linux01 的 `~/gitops-lab` 下执行(`kubectl` / `kubeseal` 可直连集群,无需导出公钥证书)。
操作前先 `git pull`。

## 新增 Secret(文件不存在时)

```bash
read -rs -p 'value: ' V; echo
kubectl -n <ns> create secret generic <name> --from-literal=<KEY>="$V" \
  --dry-run=client -o yaml \
  | kubeseal --format yaml > infra/secrets/manifests/<ns>/<name>.yaml
unset V
# commit + push; ArgoCD syncs, controller decrypts into a real Secret
```

多个 key:为每个 key 各 `read` 一次,在 `create secret` 后重复 `--from-literal=<KEY2>="$V2"`。

## 已有 Secret 追加 / 修改 key(文件已存在时)

用 `--merge-into`:只更新指定的 key,其余 key 的密文保持不变,不需要知道它们的明文。

```bash
read -rs -p 'value: ' V; echo
kubectl -n <ns> create secret generic <name> --from-literal=<KEY>="$V" \
  --dry-run=client -o yaml \
  | kubeseal --format yaml --merge-into infra/secrets/manifests/<ns>/<name>.yaml
unset V
# commit + push
```

**不要**对已有文件使用新增流程里的 `> file`:它会覆盖整个文件,其他 key 会丢失。

## 提交前后检查

```bash
# 提交前:diff 里只应出现目标 key 的行
git diff infra/secrets/manifests/<ns>/<name>.yaml | grep '^[+-]' | cut -c1-80

# 同步后:key 名齐全、值长度不为 0(不要打印值)
kubectl -n <ns> get secret <name> -o json | jq '.data | keys'
kubectl -n <ns> get secret <name> -o jsonpath='{.data.<KEY>}' | base64 -d | wc -c
```

注意:
- ArgoCD 仓库凭据必须保留 label:
  `argocd.argoproj.io/secret-type: repository` (repo-*) 或 `repo-creds` (creds-*)
- `dockerconfigjson` / TLS 等非 `generic` 类型,需要对应的 `--type` 或原有 label,不能直接套用上面的命令
- 修改已有 Secret 一律走 `--merge-into` 后提交,不要 `kubectl edit`
- 曾出现 key 存在但值为空,导致下游 401。验证凭据时用需要认证的接口
  (如 Nexus `/service/rest/v1/status/check`),匿名可读的路径对空凭据也返回 200
- 以挂载文件方式使用 Secret 的应用不一定自动重新读取(Jenkins:
  Manage Jenkins → Configuration as Code → Reload existing configuration);
  以环境变量方式使用的需要重启对应工作负载
- 密码不要贴到聊天、工单或提交信息里;已暴露的立即改密并重新加密

## Jenkins 凭据(例:`NEXUS_USERNAME` / `NEXUS_PASSWORD` → `nexus-deployer`)

Secret 里新增 key 只是第一步,Jenkins 还需在 `infra/jenkins/*-values.yaml` 引用:

1. `controller.additionalExistingSecrets` 加入 `name: jenkins-credentials` + `keyName: <KEY>`
2. `controller.JCasC.configScripts.credentials` 用 `${jenkins-credentials-<KEY>}` 定义凭据(ID 如 `nexus-deployer`)
3. 顺序:先让 Secret 里有该 key,再改 values。反过来 `jenkins-0` 会卡在 `ContainerCreating`
4. 在 UI 里手动添加的凭据会在 JCasC 重载或重启后丢失,凭据只通过这条路径维护

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
