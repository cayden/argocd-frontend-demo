# ArgoCD GitOps 前端部署 Demo

用 ArgoCD 把一个简单静态前端通过 **GitOps** 自动部署到本地 Kubernetes（kind）集群。
改 Git 仓库里的 K8s 清单 → ArgoCD 自动检测并同步到集群，无需手动 `kubectl apply`。

## 架构

- **Git 仓库**（GitHub `cayden/argocd-frontend-demo`）= 唯一可信源
- **ArgoCD**（kind 集群 `argocd` 命名空间）= 持续对比 Git 与集群，发现偏差自动 apply
- **前端 Pod**（`default` 命名空间，nginx 服务静态页）
- **访问**：NodePort 30080 → kind 端口映射 → `localhost:8081`

## 文件结构

| 路径 | 说明 |
|------|------|
| `app/index.html` | 静态前端页面 |
| `app/Dockerfile` | 基于 `nginx:1.27-alpine` |
| `k8s/deployment.yaml` | Deployment（replicas=3）|
| `k8s/service.yaml` | Service（NodePort 30080）|
| `argocd/application.yaml` | ArgoCD Application（指向 GitHub `k8s/`，auto-sync + selfHeal）|
| `kind-config.yaml` | kind 集群配置（映射 30080→8081）|
| `argocd-install.yaml` | ArgoCD 官方安装清单（已改 imagePullPolicy: IfNotPresent）|

## 部署步骤

### 1. 准备环境
```bash
# 启动 Docker Desktop（GUI），然后：
kind create cluster --config kind-config.yaml --name argocd-demo
```

### 2. 安装 ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f argocd-install.yaml   # 已改 imagePullPolicy: IfNotPresent
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=180s
```

### 3. 访问 ArgoCD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
# 浏览器打开 https://localhost:8080 ，用户名 admin + 上面的密码
```

### 4. 构建并导入前端镜像
```bash
docker build -t frontend-demo:v1 app/
# 导入到 kind 节点（避免 multi-platform 坑，见踩坑记录）
docker save frontend-demo:v1 | docker exec -i argocd-demo-control-plane \
  ctr --namespace=k8s.io images import --snapshotter=overlayfs -
```

### 5. 推送清单到 Git
把 `k8s/` 和 `argocd/application.yaml` 推到 GitHub 仓库 `cayden/argocd-frontend-demo`。

### 6. 创建 ArgoCD Application
```bash
kubectl apply -f argocd/application.yaml
# ArgoCD 自动 clone 仓库 + sync 部署
kubectl get application -n argocd
kubectl get pods -l app=frontend-demo
```

## 访问
- **前端**：http://localhost:8081
- **ArgoCD UI**：https://localhost:8080（admin + 初始密码）

## GitOps 演示
```bash
# 1. 改 k8s/deployment.yaml: replicas 2 -> 3，push 到 GitHub
# 2. 触发 ArgoCD 立即 refresh（否则默认 3 分钟轮询）
kubectl annotate application frontend-demo -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
# 3. 观察 Pod 自动扩到 3 个
kubectl get pods -l app=frontend-demo -w
```

## 清理
```bash
kind delete cluster --name argocd-demo
```

## 踩坑记录（国内环境）

1. **镜像拉取超时**：docker.io / quay.io 国内慢，用镜像源加速
   - daocloud：`docker.m.daocloud.io/library/<img>`
   - 1ms.run：`quay.1ms.run/argoproj/<img>`、`ghcr.1ms.run/dexidp/<img>`

2. **`kind load docker-image` 失败**：multi-platform 镜像的 `--all-platforms` 找不到平台层。
   改用 `docker save | docker exec ctr images import` 导入单平台。

3. **ArgoCD `imagePullPolicy=Always` 卡住**：即使本地有镜像仍去 quay 验证拉取。
   把 `argocd-install.yaml` 里 `imagePullPolicy: Always` 全改成 `IfNotPresent`。

4. **dex/redis 是 OCI image index，导入后 CRI 不识别单平台**：
   在 kind 节点内 `ctr images pull --platform linux/amd64 <镜像源>` 拉 单平台再 `tag` 成原名。

5. **git push 走本地代理失败**：用 GitHub Contents API（`gh api --method PUT /repos/.../contents/...`）上传文件绕过。

6. **Git Bash 把 `/repos/...` 当文件路径**：加 `export MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL='*'` 阻止路径转换。
