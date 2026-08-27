# 第3章：网络与配置（1.5小时）

> 🎯 目标：Ingress 域名路由 + ConfigMap/Secret 配置注入 + Namespace 隔离

---

## 1. Ingress —— 域名路由（把 Service 暴露到域名）

**Service 只能用 IP:端口 访问，Ingress 让你用域名访问（类似 Nginx 反向代理）。**

### 1.1 架构

```
外部请求 → Ingress（域名匹配） → Service → Pod
            ↓
   api.example.com  →  my-api-service:80  →  Pod
   web.example.com  →  my-web-service:80  →  Pod
```

### 1.2 启用 Ingress（minikube 默认没开）

```bash
# 启用 Ingress 插件
minikube addons enable ingress

# 验证
kubectl get pods -n ingress-nginx
# 应该看到 ingress-nginx-controller 在 Running
```

### 1.3 Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.local          # 域名（本地用 /etc/hosts 伪造）
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: my-java-app-service    # 对应 Service 名
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-web-service
            port:
              number: 80
```

### 1.4 配置本地域名

```bash
# 查看 Ingress 的 IP
kubectl get ingress my-ingress
# HOSTS        ADDRESS        PORTS   AGE
# myapp.local  192.168.x.x   80      30s

# 添加 hosts 映射（IP 用上面的）
echo "192.168.49.2 myapp.local" | sudo tee -a /etc/hosts

# 验证
curl http://myapp.local/api
curl http://myapp.local/
```

---

## 2. ConfigMap —— 配置注入

**把配置文件/环境变量从镜像里拆出来，用 ConfigMap 注入。**

### 2.1 创建 ConfigMap

```bash
# 方式1：从命令行创建
kubectl create configmap app-config \
  --from-literal=DB_HOST=192.168.1.100 \
  --from-literal=DB_PORT=5432 \
  --from-literal=APP_ENV=production

# 方式2：从文件创建
kubectl create configmap nginx-config \
  --from-file=nginx.conf
```

### 2.2 ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "192.168.1.100"
  DB_PORT: "5432"
  APP_ENV: "production"
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      location / {
        proxy_pass http://127.0.0.1:8080;
      }
    }
```

### 2.3 在 Pod 里使用

```yaml
# 注入为环境变量
spec:
  containers:
  - name: my-app
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST

# 注入为文件挂载
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
```

---

## 3. Secret —— 敏感配置

**跟 ConfigMap 一样，但值是 base64 编码（注意：不是加密！只是编码）。**

### 3.1 创建 Secret

```bash
# 从命令行创建
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=S3cret!

# 从文件创建（证书等）
kubectl create secret tls my-tls \
  --cert=cert.pem \
  --key=key.pem
```

### 3.2 Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=          # echo -n "admin" | base64
  password: UzNjcmV0IQ==     # echo -n "S3cret!" | base64
```

### 3.3 在 Pod 里使用

```yaml
spec:
  containers:
  - name: my-app
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

---

## 4. 实战：部署带配置的应用

### 4.1 完整 YAML

```yaml
# deployment-with-config.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  APP_NAME: "MyApp"
  LOG_LEVEL: "INFO"
---
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=    # base64 编码
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-java-app:latest
        ports:
        - containerPort: 8080
        env:
        # 从 ConfigMap 注入
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: my-app-config
              key: APP_NAME
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: my-app-config
              key: LOG_LEVEL
        # 从 Secret 注入
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-app-secrets
              key: DB_PASSWORD
```

---

## 5. Namespace 实战：环境隔离

```bash
# 创建开发/生产命名空间
kubectl create namespace dev
kubectl create namespace prod

# 开发环境部署（副本少，不用 Ingress）
kubectl apply -f deployment.yaml -n dev

# 生产环境部署（副本多 + Ingress）
kubectl apply -f deployment.yaml -n prod

# 查看所有命名空间的资源
kubectl get pods --all-namespaces

# 按命名空间筛选
kubectl get pods -n dev
kubectl get pods -n prod
```

---

## 6. 本章小结

| 对象 | 用途 | 记住 |
|---|---|---|
| Ingress | 域名路由 | `minikube addons enable ingress` |
| ConfigMap | 非敏感配置 | `kubectl create configmap` |
| Secret | 敏感配置 | `kubectl create secret generic` |
| Namespace | 环境隔离 | `kubectl create namespace` |

**最佳实践：** ConfigMap/Secret 不要在 YAML 里硬编码，用 `kubectl create configmap` 从文件创建。

## 7. 常见问题

**Q: Ingress 访问 404？**
→ 检查 path 是否正确、Service 名和端口是否匹配

**Q: Secret 值读出来是乱码？**
→ Secret 是 base64，读出来要解码：`echo xxx | base64 -d`

**Q: ConfigMap 改了，Pod 要重启才生效？**
→ 环境变量不会自动更新（需要重启 Pod）；文件挂载会自动更新（有延迟）

---

> 📝 下一步：[第4章：存储与运维](ch4-storage.md) —— 持久化存储 + 健康检查 + 自动扩缩
