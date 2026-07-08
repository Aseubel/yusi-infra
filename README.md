# Yusi Platform GitOps Infrastructure Manifests

此仓库托管 **Yusi (灵魂叙事)** 平台的 Kubernetes (K8s) 声明式部署清单，遵循 **GitOps** 最佳实践进行持续交付与运维管理。

---

## 📁 目录结构

*   `base/`: 基础部署模板（所有环境共享）
    *   `namespace.yaml`: 命名空间定义 (`yusi`)
    *   `configmap.yaml`: 通用非敏感配置（数据库地址、模型名称等）
    *   `secrets-template.yaml`: 敏感配置模板（API Key、密码等，实际密文不可直接提交）
    *   `backend.yaml`: 后端 Java 服务的 Deployment & Service
    *   `frontend.yaml`: 前端 Nginx 服务的 Deployment & Service
    *   `mcp.yaml`: Go MCP 辅助服务的 Deployment & Service
    *   `ingress.yaml`: 外部网关流量分发与 TLS 配置
    *   `kustomization.yaml`: Kustomize 基础定义
*   `overlays/`: 环境差异化配置
    *   `dev/`: 开发/测试环境配置差异覆盖（单实例运行、dev 域名映射）
    *   `prod/`: 生产环境配置差异覆盖（高可用副本数、prod 域名映射）

---

## 🚀 部署与同步流程

我们采用 **ArgoCD** 作为 GitOps 同步引擎。

### 1. 安装 ArgoCD
在 K8s 集群中执行安装：
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. 配置应用自动同步 (Application CRD)
在集群中应用以下声明，ArgoCD 将持续监控此仓库并在变更时自动拉取并应用配置：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: yusi-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/Aseubel/yusi-infra.git'
    targetRevision: HEAD
    path: overlays/prod
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: yusi-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 🔒 敏感信息与 Secrets 最佳实践

`base/secrets-template.yaml` 仅作为占位模板，**请勿在此仓库中提交真实的 API 密钥或密码**。

### 推荐的 Secret 管理方案：
1. **集群手动创建**：
   在部署前，直接在集群中针对目标 Namespace 手动运行命令创建真实 Secret：
   ```bash
   kubectl create secret generic yusi-secret -n yusi-prod \
     --from-literal=MYSQL_PASSWORD="YOUR_PASSWORD" \
     --from-literal=CHAT_MODEL_APIKEY="sk-xxx" \
     --from-literal=EMBEDDING_MODEL_APIKEY="sk-xxx" \
     --from-literal=YUSI_ENCRYPTION_KEY="your-encryption-key" \
     --from-literal=AMAP_JS_KEY="amap-key" \
     --from-literal=AMAP_SECURITY_CODE="amap-code"
   ```
2. **Sealed Secrets**：
   使用 Bitnami 提供的 Sealed Secrets 工具，在本地通过集群公钥加密 Secret 生成 `SealedSecret` 密文，然后再提交到此 Git 仓库。
