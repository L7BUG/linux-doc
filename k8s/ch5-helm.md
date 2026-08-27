# 第5章：Helm + 综合实战（2小时）

> 🎯 目标：Helm 包管理 + 把你的 Java 应用一键部署到 K8s

---

## 1. Helm —— K8s 的"apt/brew"

**Helm = K8s 的包管理器，一个 Chart = 一组可复用的 YAML 模板。**

```bash
# Arch 安装
sudo pacman -S helm

# 验证
helm version
```

### 1.1 Helm 核心概念

| 概念 | 类比 | 说明 |
|---|---|---|
| **Chart** | 安装包 | 一组 YAML 模板（Deployment/Service/ConfigMap…） |
| **Repository** | 应用商店 | 存放 Chart 的仓库 |
| **Release** | 已安装的实例 | 一个 Chart 部署一次就是一个 Release |
| **values.yaml** | 配置文件 | 覆盖 Chart 的默认参数 |

### 1.2 常用命令

```bash
# 添加官方仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 搜索可用 Chart
helm search repo nginx

# 安装（一个命令部署 Nginx）
helm install my-nginx bitnami/nginx

# 查看
helm list
helm status my-nginx

# 升级配置
helm upgrade my-nginx bitnami/nginx --set service.type=NodePort

# 卸载
helm uninstall my-nginx
```

### 1.3 用 Helm 部署 MySQL（5分钟）

```bash
# 部署 MySQL
helm install my-mysql bitnami/mysql \
  --set auth.rootPassword=my-secret-pw \
  --set primary.persistence.size=1Gi

# 查看密码
kubectl get secret --namespace default my-mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d

# 连接测试
kubectl run mysql-client --rm -it --image=mysql -- mysql -h my-mysql -p
```

---

## 2. 自己写一个 Helm Chart

### 2.1 创建 Chart 骨架

```bash
helm create my-java-app
tree my-java-app/
# my-java-app/
# ├── Chart.yaml          # Chart 元信息
# ├── values.yaml         # 默认配置值
# ├── templates/          # YAML 模板
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── ingress.yaml
# │   └── ...
# └── README.md
```

### 2.2 修改 templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-java-app.fullname" . }}
  labels:
    {{- include "my-java-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-java-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-java-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.containerPort }}
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: {{ .Values.env.profiles }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

### 2.3 修改 values.yaml

```yaml
replicaCount: 2

image:
  repository: yourname/my-java-app
  tag: "latest"

containerPort: 8080

env:
  profiles: "production"

service:
  type: NodePort
  port: 80

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 2.4 部署自己的 Chart

```bash
# 检查模板渲染（不实际部署）
helm template my-app ./my-java-app

# 部署
helm install my-app ./my-java-app

# 查看状态
helm status my-app

# 升级（改了 values.yaml 后）
helm upgrade my-app ./my-java-app

# 卸载
helm uninstall my-app
```

---

## 3. 综合实战：把 email-parent 部署到 K8s

### 3.1 前提

```bash
# 确保你有 Docker 镜像（email-parent 打包好的 jar）
docker build -t email-parent:latest .
docker images | grep email-parent
```

### 3.2 完整部署 YAML

```yaml
# email-parent-deploy.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: email-app-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres:5432/email"
---
apiVersion: v1
kind: Secret
metadata:
  name: email-app-secrets
type: Opaque
data:
  SPRING_DATASOURCE_USERNAME: ZW1haWw=          # base64: email
  SPRING_DATASOURCE_PASSWORD: cGFzc3dvcmQ=     # base64: password
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: email-parent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: email-parent
  template:
    metadata:
      labels:
        app: email-parent
    spec:
      containers:
      - name: email-parent
        image: email-parent:latest
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: email-app-config
        - secretRef:
            name: email-app-secrets
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /api/accounts
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /api/accounts
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: email-parent-service
spec:
  type: NodePort
  selector:
    app: email-parent
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: email-parent-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: email.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: email-parent-service
            port:
              number: 80
```

### 3.3 部署步骤

```bash
# 1. 确保 Ingress 已启用
minikube addons enable ingress

# 2. 部署
kubectl apply -f email-parent-deploy.yaml

# 3. 等待 Pod Ready
kubectl get pods -w

# 4. 添加 hosts 映射
MINIKUBE_IP=$(minikube ip)
echo "$MINIKUBE_IP email.local" | sudo tee -a /etc/hosts

# 5. 验证
curl http://email.local/api/accounts
# 看到账号列表 JSON ✅
```

---

## 4. 调试技巧

```bash
# 查看 Pod 日志
kubectl logs -f <pod-name>

# 进入 Pod 调试
kubectl exec -it <pod-name> -- bash

# 查看所有资源
kubectl get all

# 查看事件（排错用）
kubectl get events --sort-by='.lastTimestamp'

# 检查 YAML 语法
kubectl apply -f xxx.yaml --dry-run=client
```

---

## 5. 本章小结

| 概念 | 作用 | 命令 |
|---|---|---|
| Helm | K8s 包管理器 | `helm install` |
| Chart | 可复用模板 | `helm create` |
| values.yaml | 配置覆盖 | `helm upgrade --set` |
| 调试 | 日志/事件/exec | `kubectl logs/events/exec` |

---

## 6. 学习路线继续

学完这 5 章，你已经掌握了 **K8s 开发者 80% 的日常技能**：

| 能力 | 状态 |
|---|---|
| 部署应用到 K8s | ✅ |
| 配置管理（ConfigMap/Secret） | ✅ |
| 健康检查 + 自动扩缩 | ✅ |
| 持久化存储 | ✅ |
| Helm 包管理 | ✅ |
| 域名路由（Ingress） | ✅ |

**进阶方向（需要时再学）：**

| 方向 | 说明 |
|---|---|
| CI/CD 集成 | GitHub Actions / Jenkins 自动部署到 K8s |
| 生产环境 | 多节点集群、负载均衡、TLS 证书 |
| 可观测性 | Prometheus + Grafana 监控 |
| 日志收集 | EFK/ELK Stack |
| 服务网格 | Istio / Linkerd（微服务通信） |

**实践建议：** 把你的 email-parent 先在本地 K8s 跑通，再考虑上云（阿里云/AWS 都有 K8s 托管服务）。
