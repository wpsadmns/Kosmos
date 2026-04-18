# Kosmos K8s 部署 Playbook

本文档提供 Kosmos AI Scientist 在 Kubernetes 上的完整部署步骤。

## 目录

- [前置条件](#前置条件)
- [架构概览](#架构概览)
- [Step 1: 准备 K8s 集群](#step-1-准备-k8s-集群)
- [Step 2: 构建并推送容器镜像](#step-2-构建并推送容器镜像)
- [Step 3: 创建 Secrets](#step-3-创建-secrets)
- [Step 4: 部署基础设施（数据库层）](#step-4-部署基础设施数据库层)
- [Step 5: 验证基础设施就绪](#step-5-验证基础设施就绪)
- [Step 6: 部署 Kosmos 应用](#step-6-部署-kosmos-应用)
- [Step 7: 运行数据库迁移](#step-7-运行数据库迁移)
- [Step 8: 配置 Ingress 和外部访问](#step-8-配置-ingress-和外部访问)
- [Step 9: 验证部署](#step-9-验证部署)
- [Step 10: 运行首个研究任务](#step-10-运行首个研究任务)
- [Kustomize Overlay 使用](#kustomize-overlay-使用)
- [监控与运维](#监控与运维)
- [故障排查](#故障排查)
- [回滚流程](#回滚流程)
- [清理资源](#清理资源)

---

## 前置条件

| 工具 | 最低版本 | 用途 |
|------|---------|------|
| kubectl | 1.27+ | K8s 命令行 |
| kustomize | 5.0+ | K8s 资源组织（或使用 `kubectl apply -k`） |
| docker | 24+ | 镜像构建 |
| git | 2.x | 源码管理 |
| Python | 3.11+ | 本地工具 |
| Helm (可选) | 3.12+ | Ingress Controller / cert-manager 安装 |

所需权限：K8s 集群的 admin 或 namespace 级别 create/delete 权限，容器镜像仓库的 push 权限。

---

## 架构概览

```
                    ┌─────────────┐
                    │   Ingress   │ (nginx + TLS)
                    │  :443→:8000 │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌─────────� ┌─────────┐ ┌─────────┐
         │ kosmos  │ │ kosmos  │ │ kosmos  │  ← HPA: 2-10 replicas
         │  pod-0  │ │  pod-1  │ │  pod-2  │
         └────┬────┘ └────┬────┘ └────┬────┘
              │            │            │
     ┌────────┼────────────┼────────────┼────────┐
     │        ▼            ▼            ▼        │
     │   ┌─────────┐ ┌────────┐ ┌──────────┐    │
     │   │ PostgreSQL│ │ Redis  │ │  Neo4j   │    │  ← namespace: kosmos
     │   │ (Stateful)│ │(Deploy)│ │(Stateful)│    │
     │   │  :5432   │ │ :6379  │ │ :7687    │    │
     │   └─────────┘ └────────┘ └──────────┘    │
     │                                            │
     │  + NetworkPolicy: 仅 kosmos pod 可访问数据库  │
     │  + PDB: 保证最少 1 个 pod 可用               │
     └────────────────────────────────────────────┘
```

**K8s 资源清单：**

| 资源 | 文件 | 说明 |
|------|------|------|
| Namespace | `namespace.yaml` | `kosmos` 命名空间 |
| ConfigMap | `configmap.yaml` | 非敏感配置 |
| Secret | `secrets.yaml.template` | API Key、数据库密码 |
| PostgreSQL | `postgres-statefulset.yaml` | StatefulSet + Headless Service + PVC |
| Redis | `redis-deployment.yaml` | Deployment + Service + PVC |
| Neo4j | `neo4j-statefulset.yaml` | StatefulSet + Headless Service + PVC |
| Kosmos App | `kosmos-deployment.yaml` | Deployment + PVC (results) |
| Service | `kosmos-service.yaml` | ClusterIP :8000 |
| HPA | `hpa.yaml` | CPU 70% / Memory 80% 触发扩缩容 |
| Ingress | `ingress.yaml` | nginx + cert-manager TLS |
| NetworkPolicy | `networkpolicy.yaml` | 最小权限网络策略 |
| PDB | `pdb.yaml` | 保证 minAvailable=1 |

---

## Step 1: 准备 K8s 集群

### 1.1 确认集群可用

```bash
kubectl cluster-info
kubectl get nodes
```

确认所有节点状态为 `Ready`。

### 1.2 安装 Ingress Controller（如未安装）

```bash
# 使用 Helm 安装 nginx ingress
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

### 1.3 安装 cert-manager（如需 TLS）

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# 创建 ClusterIssuer
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

### 1.4 创建 StorageClass（如需自定义）

默认配置使用 `standard` StorageClass。如你的集群使用不同名称：

```bash
# 查看可用 StorageClass
kubectl get storageclass

# 如果需要，修改 k8s/ 下所有 PVC 文件中的 storageClassName
```

---

## Step 2: 构建并推送容器镜像

### 2.1 设置镜像仓库变量

```bash
# 替换为你的镜像仓库地址
export REGISTRY=your-registry.io/kosmos
export TAG=v0.2.0
```

### 2.2 构建镜像

```bash
cd /path/to/Kosmos

docker build -t ${REGISTRY}:${TAG} .
docker tag ${REGISTRY}:${TAG} ${REGISTRY}:latest
```

### 2.3 推送镜像

```bash
docker push ${REGISTRY}:${TAG}
docker push ${REGISTRY}:latest
```

### 2.4 更新 Deployment 中的镜像引用

编辑 `k8s/kosmos-deployment.yaml`，将 `image: kosmos:latest` 替换：

```yaml
image: your-registry.io/kosmos:v0.2.0
```

或使用 kustomize 的 `images` 字段（见 [Kustomize Overlay 使用](#kustomize-overlay-使用)）。

---

## Step 3: 创建 Secrets

### 3.1 准备 Secret 值

```bash
# Base64 编码你的实际值
echo -n "sk-ant-your-actual-api-key" | base64
# 输出: c2stYW50LXlvdXItYWN0dWFsLWFwaS1rZXk=

echo -n "your-postgres-password" | base64
# 输出: eW91ci1wb3N0Z3Jlcy1wYXNzd29yZA==

echo -n "your-neo4j-password" | base64
# 输出: eW91ci1uZW80ai1wYXNzd29yZA==
```

### 3.2 创建实际 Secret 文件

```bash
cat <<EOF > k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: kosmos-secrets
  namespace: kosmos
type: Opaque
data:
  anthropic-api-key: $(echo -n "sk-ant-your-actual-api-key" | base64)
  postgres-password: $(echo -n "your-postgres-password" | base64)
  neo4j-password: $(echo -n "your-neo4j-password" | base64)
EOF
```

**重要：** `secrets.yaml` 包含敏感信息，确保已加入 `.gitignore`。

### 3.3 验证 .gitignore

```bash
grep -q "k8s/secrets.yaml" .gitignore || echo "k8s/secrets.yaml" >> .gitignore
```

---

## Step 4: 部署基础设施（数据库层）

按顺序部署，先数据库后应用。

### 4.1 创建 Namespace

```bash
kubectl apply -f k8s/namespace.yaml
```

### 4.2 部署 ConfigMap 和 Secrets

```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml
```

### 4.3 部署 PostgreSQL

```bash
kubectl apply -f k8s/postgres-statefulset.yaml
```

### 4.4 部署 Redis

```bash
kubectl apply -f k8s/redis-deployment.yaml
```

### 4.5 部署 Neo4j

```bash
kubectl apply -f k8s/neo4j-statefulset.yaml
```

### 4.6 部署 NetworkPolicy

```bash
kubectl apply -f k8s/networkpolicy.yaml
```

---

## Step 5: 验证基础设施就绪

### 5.1 等待 Pod 就绪

```bash
# 等待所有基础设施 Pod Running
kubectl wait --for=condition=ready pod -l app=postgres -n kosmos --timeout=120s
kubectl wait --for=condition=ready pod -l app=redis -n kosmos --timeout=120s
kubectl wait --for=condition=ready pod -l app=neo4j -n kosmos --timeout=180s
```

### 5.2 确认所有 Pod 状态

```bash
kubectl get pods -n kosmos -w
```

预期输出：

```
NAME                       READY   STATUS    RESTARTS   AGE
postgres-0                 1/1     Running   0          2m
redis-xxxxxxxxxx-xxxxx     1/1     Running   0          2m
neo4j-0                    1/1     Running   0          2m
```

### 5.3 验证服务可达性（从集群内部）

```bash
# 临时调试 Pod
kubectl run debug --image=postgres:15-alpine -n kosmos --rm -it -- \
  pg_isready -h postgres.kosmos.svc.cluster.local -p 5432

kubectl run debug-redis --image=redis:7-alpine -n kosmos --rm -it -- \
  redis-cli -h redis.kosmos.svc.cluster.local ping
```

---

## Step 6: 部署 Kosmos 应用

### 6.1 部署应用 Deployment + Service

```bash
kubectl apply -f k8s/kosmos-deployment.yaml
kubectl apply -f k8s/kosmos-service.yaml
```

### 6.2 部署 HPA 和 PDB

```bash
kubectl apply -f k8s/hpa.yaml
kubectl apply -f k8s/pdb.yaml
```

### 6.3 等待应用 Pod 就绪

```bash
kubectl wait --for=condition=ready pod -l app=kosmos -n kosmos --timeout=180s
```

### 6.4 确认应用状态

```bash
kubectl get pods -n kosmos
kubectl get hpa -n kosmos
```

---

## Step 7: 运行数据库迁移

### 7.1 执行 Alembic 迁移

```bash
kubectl exec -it deployment/kosmos -n kosmos -- \
  alembic upgrade head
```

### 7.2 验证迁移

```bash
kubectl exec -it deployment/kosmos -n kosmos -- \
  python -c "from kosmos.db import get_session; s = get_session(); print('DB OK')"
```

---

## Step 8: 配置 Ingress 和外部访问

### 8.1 更新域名

编辑 `k8s/ingress.yaml`，将 `kosmos.yourdomain.com` 替换为你的实际域名。

### 8.2 部署 Ingress

```bash
kubectl apply -f k8s/ingress.yaml
```

### 8.3 获取外部 IP

```bash
kubectl get ingress -n kosmos
```

### 8.4 配置 DNS

将域名 A 记录或 CNAME 指向 Ingress 的 EXTERNAL-IP。

### 8.5 验证 TLS 证书签发

```bash
kubectl get certificate -n kosmos
# 等待 READY=True
kubectl describe certificate kosmos-tls -n kosmos
```

---

## Step 9: 验证部署

### 9.1 完整状态检查

```bash
# 所有资源一览
kubectl get all -n kosmos

# 各组件详细状态
kubectl get pods,svc,ingress,hpa,pvc,configmap,secrets -n kosmos
```

### 9.2 应用健康检查

```bash
# Pod 内部健康检查
kubectl exec -it deployment/kosmos -n kosmos -- \
  python -c "import kosmos; print('healthy')"

# 查看应用日志
kubectl logs -f deployment/kosmos -n kosmos --tail=50
```

### 9.3 端口转发测试（跳过 Ingress 直接测试）

```bash
kubectl port-forward svc/kosmos 8000:8000 -n kosmos &
curl http://localhost:8000/health
```

### 9.4 通过 Ingress 测试（如已配置域名）

```bash
curl -s https://kosmos.yourdomain.com/health
```

---

## Step 10: 运行首个研究任务

### 10.1 通过 CLI 提交研究任务

```bash
kubectl exec -it deployment/kosmos -n kosmos -- \
  kosmos run "What metabolic pathways differ between cancer and normal cells?" \
  --domain biology --budget 5
```

### 10.2 查看研究日志

```bash
kubectl logs -f deployment/kosmos -n kosmos
```

### 10.3 查看研究结果

```bash
kubectl exec -it deployment/kosmos -n kosmos -- \
  ls -la /app/results/
```

---

## Kustomize Overlay 使用

项目提供两个 overlay 环境：

### 开发环境部署

```bash
# 使用 dev overlay（1 replica, debug 日志, 小存储）
kustomize build k8s/overlays/dev | kubectl apply -f -

# 或使用 kubectl 内置 kustomize
kubectl apply -k k8s/overlays/dev/
```

dev overlay 特点：
- 1 个 kosmos replica（非 HPA 最小 2）
- DEBUG 日志级别
- PostgreSQL PVC 10Gi
- 无 TLS
- 域名: `kosmos-dev.local`

### 生产环境部署

```bash
kustomize build k8s/overlays/prod | kubectl apply -f -

# 或
kubectl apply -k k8s/overlays/prod/
```

prod overlay 特点：
- 3 个 kosmos replica，HPA 可扩至 20
- 大幅增加 CPU/内存配额
- 并发参数调高（experiments=8, hypotheses=5, LLM calls=10）
- PostgreSQL PVC 100Gi，结果 PVC 500Gi

### 自定义镜像

在 overlay 的 `kustomization.yaml` 中添加：

```yaml
images:
  - name: kosmos
    newName: your-registry.io/kosmos
    newTag: v0.2.0
```

---

## 监控与运维

### 查看日志

```bash
# 实时日志
kubectl logs -f deployment/kosmos -n kosmos

# 最近 100 行
kubectl logs deployment/kosmos -n kosmos --tail=100

# 特定 Pod
kubectl logs kosmos-xxxxxxxxxx-xxxxx -n kosmos
```

### HPA 状态

```bash
kubectl get hpa kosmos-hpa -n kosmos
kubectl describe hpa kosmos-hpa -n kosmos
```

### 资源使用

```bash
kubectl top pods -n kosmos
kubectl top nodes
```

### 数据库连接

```bash
# PostgreSQL
kubectl exec -it postgres-0 -n kosmos -- \
  psql -U kosmos -d kosmos -c "\dt"

# Redis
kubectl exec -it deployment/redis -n kosmos -- \
  redis-cli -h redis info

# Neo4j
kubectl exec -it neo4j-0 -n kosmos -- \
  cypher-shell -u neo4j -p $(kubectl get secret kosmos-secrets -n kosmos -o jsonpath='{.data.neo4j-password}' | base64 -d) \
  "MATCH (n) RETURN count(n)"
```

### 备份数据

```bash
# PostgreSQL
kubectl exec postgres-0 -n kosmos -- \
  pg_dump -U kosmos kosmos > kosmos_backup_$(date +%Y%m%d).sql

# Neo4j
kubectl exec neo4j-0 -n kosmos -- \
  neo4j-admin database dump neo4j --to-path=/tmp/
kubectl cp kosmos/neo4j-0:/tmp/neo4j.dump ./neo4j_backup_$(date +%Y%m%d).dump
```

---

## 故障排查

### Pod 启动失败

```bash
kubectl describe pod <pod-name> -n kosmos
kubectl logs <pod-name> -n kosmos
```

常见原因：
- 镜像拉取失败：检查 `imagePullPolicy` 和镜像仓库访问权限
- 资源不足：检查节点资源 `kubectl describe node <node>`
- Secret 缺失：确认 `kosmos-secrets` 已创建

### 数据库连接失败

```bash
# 检查 PostgreSQL 状态
kubectl get pods -l app=postgres -n kosmos
kubectl logs postgres-0 -n kosmos

# 检查 Service DNS 解析
kubectl run dnsutils --image=tutum/dnsutils -n kosmos --rm -it -- \
  nslookup postgres.kosmos.svc.cluster.local

# 检查 NetworkPolicy 是否阻断
kubectl get networkpolicy -n kosmos
```

### OOMKilled

```bash
# 查看资源限制
kubectl describe pod <pod-name> -n kosmos | grep -A5 Limits

# 增加 limits
kubectl edit deployment kosmos -n kosmos
# 修改 resources.limits.memory
```

### HPA 无法扩容

```bash
kubectl describe hpa kosmos-hpa -n kosmos
```

常见原因：节点资源不足、PodDisruptionBudget 冲突、指标服务器未安装。

---

## 回滚流程

### 回滚应用 Deployment

```bash
# 查看历史版本
kubectl rollout history deployment/kosmos -n kosmos

# 回滚到上一版本
kubectl rollout undo deployment/kosmos -n kosmos

# 回滚到指定版本
kubectl rollout undo deployment/kosmos -n kosmos --to-revision=2
```

### 回滚到指定镜像版本

```bash
kubectl set image deployment/kosmos kosmos=your-registry.io/kosmos:v0.1.0 -n kosmos
kubectl rollout status deployment/kosmos -n kosmos
```

---

## 清理资源

### 删除整个部署

```bash
kubectl delete -k k8s/
```

### 仅删除命名空间（级联删除所有资源）

```bash
kubectl delete namespace kosmos
```

**注意：** 这将删除所有 PVC 中的数据（数据库、研究结果），不可恢复。

### 保留数据仅删除应用

```bash
kubectl delete deployment kosmos -n kosmos
kubectl delete hpa kosmos-hpa -n kosmos
kubectl delete ingress kosmos-ingress -n kosmos
# 保留 postgres、redis、neo4j 和 PVC
```
