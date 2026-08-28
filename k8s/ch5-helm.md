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
# ├── Chart.yaml          # Chart 元信息（版本、描述）
# ├── values.yaml         # 默认配置值（你在这里改参数）
# ├── templates/          # YAML 模板（用 Go template 语法）
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── ingress.yaml
# │   ├── _helpers.tpl    # 模板函数（helm create 自动生成，不用管）
# │   └── ...
# └── README.md
```

### 2.2 三个文件分别怎么写

---

#### 📄 Chart.yaml（Chart 的身份证）

```yaml
apiVersion: v2              # Helm 3 固定 v2
name: my-java-app           # Chart 名字
description: My Java application on K8s
type: application           # application 或 library
version: 0.1.0              # Chart 版本（你自己的）
appVersion: "1.0.0"         # 应用版本
```

> 这个文件基本不用改，`helm create` 已经生成好了。

---

#### 📄 values.yaml（你的配置面板）

**这是你最常改的文件。** 所有可配置的参数都在这里定义默认值。

```yaml
# 副本数
replicaCount: 2

# 镜像配置
image:
  repository: yourname/my-java-app
  tag: "latest"
  pullPolicy: IfNotPresent

# 容器端口
containerPort: 8080

# 环境变量
env:
  profiles: "production"

# Service 配置
service:
  type: NodePort
  port: 80
  nodePort: 30080

# 资源限制
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Ingress 配置
ingress:
  enabled: true
  host: myapp.local
```

**怎么用？** 两种方式覆盖默认值：

```bash
# 方式1：命令行 --set
helm install my-app ./my-java-app --set replicaCount=5 --set image.tag=v2.0

# 方式2：写自己的 values 文件
helm install my-app ./my-java-app -f my-values.yaml
```

---

#### 📄 templates/ 目录（YAML 模板语法）

**这是 Helm 的核心——用 Go template 语法让 YAML 动态化。**

### 2.3 Go Template 语法速查

| 语法 | 含义 | 例子 |
|---|---|---|
| `{{ .Values.xxx }}` | 读 values.yaml 的值 | `{{ .Values.replicaCount }}` → `2` |
| `{{ .Chart.Name }}` | 读 Chart.yaml 的值 | `{{ .Chart.Name }}` → `my-java-app` |
| `{{ .Release.Name }}` | 当前 Release 名 | `helm install my-app` → `my-app` |
| `{{ include "xxx" . }}` | 调用 _helpers.tpl 里的模板 | `{{ include "my-java-app.fullname" . }}` |
| `{{- if .Values.xxx }}` | 条件判断 | 条件为 true 才渲染下面内容 |
| `{{- range .Values.xxx }}` | 循环遍历列表 | 遍历环境变量列表 |
| `\| nindent N` | 缩进 N 个空格 | YAML 必须缩进对齐 |
| `\| toYaml` | 把对象转成 YAML | 渲染 resources 整块 |

### 2.4 从零写 templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}                    # Release 名作为 Deployment 名
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}          # 读 values.yaml 的 replicaCount
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}                # Chart 名作为容器名
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.containerPort }}
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: {{ .Values.env.profiles }}
        resources:                              # 直接渲染整块 resources
          {{- toYaml .Values.resources | nindent 10 }}
```

### 2.5 从零写 templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-svc
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.containerPort }}
    {{- if eq .Values.service.type "NodePort" }}
    nodePort: {{ .Values.service.nodePort }}   # 只有 NodePort 类型才渲染这行
    {{- end }}
```

### 2.6 条件渲染（if/else）

```yaml
{{- if .Values.ingress.enabled }}              # 只有 ingress.enabled=true 才渲染
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ .Release.Name }}-svc
            port:
              number: {{ .Values.service.port }}
{{- end }}
```

### 2.7 循环渲染（range）

```yaml
# values.yaml 里：
env:
  DB_HOST: "192.168.1.100"
  DB_PORT: "5432"

# templates 里：
spec:
  containers:
  - env:
    {{- range $key, $value := .Values.env }}
    - name: {{ $key }}
      value: "{{ $value }}"
    {{- end }}
```

渲染结果：
```yaml
    - name: DB_HOST
      value: "192.168.1.100"
    - name: DB_PORT
      value: "5432"
```

---

### 2.8 修改 values.yaml（你的配置）

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
  nodePort: 30080

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

ingress:
  enabled: true
  host: myapp.local
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
