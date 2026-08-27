# 第4章：存储与运维（1.5小时）

> 🎯 目标：PVC 持久化 + 健康检查 + 资源限制 + HPA 自动扩缩

---

## 1. PersistentVolume（PV）+ PVC —— 持久化存储

**Pod 重启后数据会丢。用 PVC 让数据持久化。**

### 1.1 概念

| 概念 | 类比 | 说明 |
|---|---|---|
| **PersistentVolume（PV）** | 硬盘 | 存储资源（管理员创建或动态分配） |
| **PersistentVolumeClaim（PVC）** | 申请表 | 你告诉 K8s "我要 1G 存储" |
| **StorageClass** | 硬盘类型 | 定义存储类型（自动分配 PV） |

### 1.2 minikube 配置持久化

```bash
# minikube 默认用 hostPath（数据存在宿主机 /tmp 目录）
# 启用持久化存储插件
minikube addons enable storage-provisioner

# 创建 PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# 在 Pod 里挂载
```

### 1.3 Deployment 挂载 PVC

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
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
        image: nginx:latest
        volumeMounts:
        - name: data-volume
          mountPath: /usr/share/nginx/html   # 挂载到这个路径
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: my-pvc                  # 引用 PVC
```

```bash
# 验证：Pod 重启后数据还在
kubectl exec -it <pod-name> -- sh -c "echo 'hello' > /usr/share/nginx/html/index.html"
kubectl delete pod <pod-name>   # 删除 Pod
# Deployment 会自动重建 Pod
kubectl exec -it <new-pod> -- cat /usr/share/nginx/html/index.html
# 输出: hello ✅ 数据还在
```

---

## 2. 健康检查（Liveness / Readiness）

### 2.1 两种检查

| 类型 | 作用 | 失败后果 |
|---|---|---|
| **Liveness** | 存活检查（进程还活着吗） | 重启 Pod |
| **Readiness** | 就绪检查（能接收流量吗） | 从 Service 摘除（不转发流量） |

### 2.2 YAML 示例

```yaml
spec:
  containers:
  - name: my-app
    image: my-java-app:latest
    ports:
    - containerPort: 8080
    # 存活检查（每 10 秒检测一次）
    livenessProbe:
      httpGet:
        path: /actuator/health     # Spring Boot Actuator 健康端点
        port: 8080
      initialDelaySeconds: 30      # 启动后等 30 秒再检查（给应用启动时间）
      periodSeconds: 10
      failureThreshold: 3          # 连续失败 3 次就重启
    # 就绪检查（就绪后才接收流量）
    readinessProbe:
      httpGet:
        path: /actuator/health
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 5
```

### 2.3 TCP 检查（非 HTTP 应用）

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

---

## 3. 资源限制（requests / limits）

**防止某个 Pod 把整台机器的 CPU/内存吃光。**

```yaml
spec:
  containers:
  - name: my-app
    resources:
      requests:         # 启动时至少分配的资源
        memory: "256Mi"
        cpu: "250m"      # 0.25 核
      limits:           # 最多能用多少
        memory: "512Mi"
        cpu: "500m"      # 0.5 核
```

**单位说明：**
- CPU：`1000m` = 1 核；`250m` = 0.25 核
- 内存：`Mi` = 兆字节；`Gi` = 吉字节

**记忆口诀：** `requests` 是"保底"，`limits` 是"天花板"。超 limits 会被 OOM Kill。

---

## 4. HPA（水平自动扩缩）

**流量大了自动扩容，流量小了自动缩容。**

### 4.1 启用 HPA

```bash
# 先启用 metrics-server（minikube 插件）
minikube addons enable metrics-server

# 基于 CPU 使用率自动扩缩
kubectl autoscale deployment my-app \
  --cpu-percent=70 \
  --min=2 \
  --max=10

# 查看 HPA 状态
kubectl get hpa
# NAME     REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS
# my-app   Deployment/my-app   20%/70%   2         10        2

# 手动删除 HPA
kubectl delete hpa my-app
```

### 4.2 手动扩缩（不用 HPA）

```bash
kubectl scale deployment my-app --replicas=5
```

---

## 5. 完整实战：带健康检查 + 资源限制 + 持久化的 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-java-app
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
        image: my-java-app:latest
        ports:
        - containerPort: 8080
        # 资源限制
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        # 环境变量
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        # 存活检查
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        # 就绪检查
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
        # 持久化存储
        volumeMounts:
        - name: app-data
          mountPath: /app/data
      volumes:
      - name: app-data
        persistentVolumeClaim:
          claimName: my-pvc
```

---

## 6. 本章小结

| 概念 | 作用 | 命令 |
|---|---|---|
| PVC | 持久化存储 | `kubectl get pvc` |
| Liveness | 挂了自动重启 | 在 Deployment 里配置 |
| Readiness | 未就绪不接流量 | 在 Deployment 里配置 |
| resources | 资源隔离 | requests/limits |
| HPA | 自动扩缩 | `kubectl autoscale` |

## 7. 常见问题

**Q: Pod 一直 `OOMKilled`？**
→ 内存 limits 太低，调高 `limits.memory`

**Q: PVC 一直 `Pending`？**
→ 没有匹配的 PV/StorageClass，minikube 确保 `storage-provisioner` 已启用

**Q: HPA 不扩容？**
→ metrics-server 没装好，先 `minikube addons enable metrics-server`

---

> 📝 下一步：[第5章：Helm + 综合实战](ch5-helm.md) —— Helm 包管理 + 把你的应用一键部署到 K8s
