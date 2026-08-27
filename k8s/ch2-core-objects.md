# 第2章：核心对象详解（2小时）

> 🎯 目标：会写 Deployment/Service/ConfigMap 的 YAML，会用 kubectl 管理应用

---

## 1. Pod —— 最小运行单位

**Pod = 一个/多个容器的壳。** 你的应用容器跑在 Pod 里。

### 1.1 Pod 长什么样

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app        # 标签，后面 Service 用它来找 Pod
spec:
  containers:
  - name: my-app
    image: nginx:latest
    ports:
    - containerPort: 80
```

### 1.2 实际操作：创建/查看/删除

```bash
# 创建 Pod
kubectl apply -f my-pod.yaml

# 查看 Pod 状态
kubectl get pods
# NAME     READY   STATUS    RESTARTS   AGE
# my-app   1/1     Running   0          10s

# 查看详情
kubectl describe pod my-app

# 进入 Pod 内部（调试用）
kubectl exec -it my-app -- bash

# 查看日志
kubectl logs my-app

# 删除
kubectl delete pod my-app
```

### 1.3 重要：为什么不直接用 Pod？

**Pod 是"短命"的**——挂了、被删了不会自己复活。你需要 **Deployment** 来管理 Pod。

---

## 2. Deployment —— Pod 的"管家"（最常用）

**Deployment = 你要几个 Pod，挂了自动重启。**

### 2.1 Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app              # Deployment 名字
  labels:
    app: my-app
spec:
  replicas: 3               # 要 3 个副本（Pod）
  selector:
    matchLabels:
      app: my-app           # 管理哪些 Pod
  template:
    metadata:
      labels:
        app: my-app         # Pod 的标签（必须和 matchLabels 一致）
    spec:
      containers:
      - name: my-app
        image: nginx:latest
        ports:
        - containerPort: 80
```

**关键字段解释：**

| 字段 | 含义 |
|---|---|
| `replicas: 3` | 保持 3 个 Pod 运行（挂一个自动补一个） |
| `selector.matchLabels` | 管理带 `app: my-app` 标签的 Pod |
| `template` | 创建 Pod 的模板（下面的内容就是 Pod 的 spec） |

### 2.2 Deployment 操作

```bash
# 创建
kubectl apply -f deployment.yaml

# 查看
kubectl get deploy
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# my-app   3/3     3            3           2m

# 查看 Pod（自动带了 Deployment 的标签）
kubectl get pods -l app=my-app

# 扩容到 5 个 Pod
kubectl scale deployment my-app --replicas=5

# 缩容到 2 个
kubectl scale deployment my-app --replicas=2

# 滚动更新（换镜像版本）
kubectl set image deployment/my-app my-app=nginx:1.25

# 回滚（更新出问题了）
kubectl rollout undo deployment/my-app

# 查看更新历史
kubectl rollout history deployment/my-app

# 删除
kubectl delete deploy my-app
```

---

## 3. Service —— 把 Pod 暴露出去

**Pod 的 IP 会变（重启就变），Service 提供一个稳定的访问入口。**

### 3.1 Service 类型

| 类型 | 用途 | 场景 |
|---|---|---|
| **ClusterIP**（默认） | 集群内访问 | 微服务之间调用 |
| **NodePort** | 通过节点端口访问 | 本地测试/开发 |
| **LoadBalancer** | 云厂商负载均衡 | 生产环境（需要云支持） |

### 3.2 Service YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: NodePort           # 本地测试用 NodePort
  selector:
    app: my-app            # 找带 app=my-app 标签的 Pod
  ports:
  - port: 80               # Service 端口（集群内访问）
    targetPort: 80         # Pod 的端口
    nodePort: 30080        # 节点暴露的端口（30000-32767）
```

### 3.3 Service 操作

```bash
# 创建
kubectl apply -f service.yaml

# 查看
kubectl get svc
# NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# my-app-service    NodePort    10.107.x.x     <none>        80:30080/TCP   5s

# 访问（minikube 需要用 tunnel 或 service url）
minikube service my-app-service --url
# 输出: http://127.0.0.1:30080

# curl 测试
curl $(minikube service my-app-service --url)
# 看到 Nginx 欢迎页 ✅
```

---

## 4. Namespace —— 隔离环境

**Namespace = 逻辑隔离（不同项目/环境用不同命名空间）**

```bash
# 创建命名空间
kubectl create namespace dev

# 查看
kubectl get namespaces

# 在指定命名空间操作
kubectl apply -f deployment.yaml -n dev
kubectl get pods -n dev

# 设置默认命名空间（省得每次 -n）
kubectl config set-context --current --namespace=dev
```

---

## 5. 实战：部署你自己的 Node.js/Java 应用

### 5.1 假设你有一个 Docker 镜像

```bash
# 先确保镜像存在（Docker Hub 或本地）
docker images | grep my-java-app
# my-java-app   latest   ...
```

### 5.2 完整 YAML：Deployment + Service

```yaml
# app.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-java-app
  labels:
    app: my-java-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-java-app
  template:
    metadata:
      labels:
        app: my-java-app
    spec:
      containers:
      - name: my-java-app
        image: my-java-app:latest      # 你的镜像
        ports:
        - containerPort: 8080
        resources:                      # 限制资源（后面第4章详细讲）
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:                            # 环境变量
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
---
apiVersion: v1
kind: Service
metadata:
  name: my-java-app-service
spec:
  type: NodePort
  selector:
    app: my-java-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

```bash
# 部署
kubectl apply -f app.yaml

# 等待 Pod Ready
kubectl get pods -w   # -w 持续监听状态变化

# 访问
minikube service my-java-app-service --url
```

### 5.3 本地镜像问题

**minikube 默认不认本地 Docker 镜像！** 需要：

```bash
# 方案1：用 minikube 的 Docker 环境
eval $(minikube docker-env)
docker build -t my-java-app:latest .
# 然后 apply（不再需要推到 registry）

# 方案2：推到 Docker Hub
docker login
docker tag my-java-app:latest yourname/my-java-app:latest
docker push yourname/my-java-app:latest
# Deployment 里 image 改成 yourname/my-java-app:latest
```

---

## 6. 本章小结

| 对象 | 作用 | 记住这个命令 |
|---|---|---|
| Pod | 最小单位 | `kubectl get pods` |
| Deployment | 管理 Pod 副本 | `kubectl scale deploy` |
| Service | 稳定访问入口 | `kubectl get svc` |
| Namespace | 逻辑隔离 | `kubectl create ns` |

**最常用的模式：Deployment + Service 绑定（一个 YAML 里写两个资源）**

## 7. 常见问题

**Q: Pod 一直 `Pending`？**
→ `kubectl describe pod <pod-name>` 看 Events：镜像拉不下来？资源不够？

**Q: Pod `CrashLoopBackOff`？**
→ 看日志：`kubectl logs <pod-name>`——通常是应用启动报错。

**Q: Service 访问不通？**
→ 检查 selector 标签是否和 Pod 一致：`kubectl get pods --show-labels`

---

> 📝 下一步：[第3章：网络与配置](ch3-networking.md) —— Ingress 域名路由 + ConfigMap/Secret 配置注入
