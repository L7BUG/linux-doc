# K8s + Istio + Spring Boot 实战教程

> 🎯 目标：在 K8s 上用 Istio 管理 Spring Boot 微服务——流量管理、安全、可观测性
> 🖥️ 环境：Arch Linux + minikube + istioctl
> 📝 前置：已完成 K8s 基础 5 章学习

---

## 1. Istio 是什么

**一句话：Istio 是 K8s 微服务的"交通警察"。**

你的 Spring Boot 微服务部署到 K8s 后，服务之间互相调用（A → B → C）。Istio 在**每个 Pod 旁边塞一个 Envoy 代理**（sidecar），拦截所有网络流量，然后你可以：

| 能力 | 说明 |
|---|---|
| **流量管理** | 金丝雀发布、A/B 测试、故障注入、超时重试 |
| **安全** | 服务间自动 mTLS 加密、细粒度访问控制 |
| **可观测性** | 自动收集 metrics/logs/traces，接 Grafana/Kiali |

**你不需要改一行 Spring Boot 代码**——Istio 全在网络层搞定。

## 2. 架构（30秒搞懂）

```
┌───────────── Pod A ─────────────┐    ┌───────────── Pod B ─────────────┐
│  Spring Boot App                │    │  Spring Boot App                │
│  localhost:8080                 │    │  localhost:8080                 │
│                                 │    │                                 │
│  ┌──── Envoy Sidecar ───────┐  │    │  ┌──── Envoy Sidecar ───────┐  │
│  │ 拦截所有进出流量          │──┼────┼──│ 拦截所有进出流量          │  │
│  │ 执行 Istio 策略          │  │    │  │ 执行 Istio 策略          │  │
│  └──────────────────────────┘  │    │  └──────────────────────────┘  │
└─────────────────────────────────┘    └─────────────────────────────────┘
         │                                   │
         └──────── istiod（控制平面）────────┘
                   ↙        ↓        ↘
              流量规则    安全策略    可观测性
```

- **istiod**：控制平面（大脑），下发配置给所有 Envoy
- **Envoy**：数据平面（每个 Pod 一个 sidecar），执行策略
- **你的应用**：完全无感知，正常跑

---

## 3. 安装 Istio（15分钟）

### 3.1 前提

```bash
# 确保 minikube 在跑
minikube status
kubectl get nodes

# minikube 需要至少 4G 内存（Istio 有额外开销）
minikube stop
minikube start --memory=4096 --cpus=2
```

### 3.2 安装 istioctl

```bash
# Arch AUR
yay -S istioctl

# 验证
istioctl version
```

### 3.3 安装 Istio

```bash
# 用 demo 配置（包含所有组件，适合学习）
istioctl install --set profile=demo -y

# 验证安装
istioctl verify-install

# 等待 istiod 就绪
kubectl wait --for=condition=Ready pod -l app=istiod -n istio-system --timeout=120s

# 查看状态
kubectl get pods -n istio-system
# 应该看到: istiod-xxx   Running
#           istio-ingressgateway-xxx  Running
```

### 3.4 启用 Sidecar 自动注入

```bash
# 给 default 命名空间开启自动注入（新 Pod 自动带 Envoy sidecar）
kubectl label namespace default istio-injection=enabled

# 验证
kubectl get namespace default --show-labels
# istio-injection=enabled
```

---

## 4. Spring Boot 微服务示例

### 4.1 创建 2 个 Spring Boot 服务

**服务架构：**
```
用户 → Gateway(8080) → OrderService(8081) → ProductService(8082)
```

#### 创建 Spring Boot 项目（用 Spring Initializr）

**OrderService（订单服务）：**

```bash
# 用 curl 下载 Spring Initializr 生成的项目
mkdir -p ~/k8s-istio-demo && cd ~/k8s-istio-demo

# OrderService
curl -s "https://start.spring.io/starter.zip?type=maven-project&language=java&bootVersion=3.5.4&baseDir=order-service&groupId=com.example&artifactId=order-service&name=order-service&dependencies=web" -o order-service.zip
unzip order-service.zip && rm order-service.zip

# ProductService（产品服务）
curl -s "https://start.spring.io/starter.zip?type=maven-project&language=java&bootVersion=3.5.4&baseDir=product-service&groupId=com.example&artifactId=product-service&name=product-service&dependencies=web" -o product-service.zip
unzip product-service.zip && rm product-service.zip
```

#### OrderService 代码

```java
// order-service/src/main/java/com/example/orderservice/OrderController.java
package com.example.orderservice;

import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import java.util.Map;
import java.util.List;

@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/{id}")
    public Map<String, Object> getOrder(@PathVariable String id) {
        // 调用 ProductService
        @SuppressWarnings("unchecked")
        Map<String, Object> product = restTemplate.getForObject(
            "http://product-service:8082/products/P001", Map.class);

        return Map.of(
            "orderId", id,
            "product", product != null ? product : "N/A",
            "status", "CONFIRMED",
            "instance", getInstanceId()
        );
    }

    @GetMapping
    public List<Map<String, String>> listOrders() {
        return List.of(
            Map.of("orderId", "ORD-001", "product", "iPhone 15"),
            Map.of("orderId", "ORD-002", "product", "MacBook Pro")
        );
    }

    private String getInstanceId() {
        return System.getenv().getOrDefault("HOSTNAME", "unknown");
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```yaml
# order-service/src/main/resources/application.yml
server:
  port: 8081
spring:
  application:
    name: order-service
```

#### ProductService 代码

```java
// product-service/src/main/java/com/example/productservice/ProductController.java
package com.example.productservice;

import org.springframework.web.bind.annotation.*;
import java.util.Map;
import java.util.List;

@RestController
@RequestMapping("/products")
public class ProductController {

    @GetMapping("/{id}")
    public Map<String, Object> getProduct(@PathVariable String id) {
        return Map.of(
            "productId", id,
            "name", "iPhone 15 Pro Max",
            "price", 9999,
            "instance", getInstanceId()
        );
    }

    @GetMapping
    public List<Map<String, Object>> listProducts() {
        return List.of(
            Map.of("productId", "P001", "name", "iPhone 15 Pro Max", "price", 9999),
            Map.of("productId", "P002", "name", "MacBook Pro", "price", 14999)
        );
    }

    private String getInstanceId() {
        return System.getenv().getOrDefault("HOSTNAME", "unknown");
    }
}
```

```yaml
# product-service/src/main/resources/application.yml
server:
  port: 8082
spring:
  application:
    name: product-service
```

### 4.2 Dockerfile（两个服务通用）

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 4.3 构建镜像

```bash
# minikube Docker 环境
eval $(minikube docker-env)

# 构建两个镜像
cd ~/k8s-istio-demo
docker build -t order-service:latest ./order-service
docker build -t product-service:latest ./product-service
```

---

## 5. K8s 部署 + Istio Sidecar 注入

### 5.1 Deployment YAML

```yaml
# k8s-deployments.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  labels:
    app: product-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
      - name: product-service
        image: product-service:latest
        ports:
        - containerPort: 8082
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  selector:
    app: product-service
  ports:
  - port: 8082
    targetPort: 8082
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        ports:
        - containerPort: 8081
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 8081
    targetPort: 8081
```

### 5.2 部署

```bash
kubectl apply -f k8s-deployments.yaml

# 等待 Pod 就绪
kubectl get pods -w   # -w 持续监听

# 验证 Sidecar 注入（每个 Pod 应该有 2/2 READY）
kubectl get pods
# NAME                                READY   STATUS    RESTARTS
# order-service-xxx-yyy              2/2     Running   0         ← 2/2 = sidecar 注入成功
# product-service-xxx-zzz            2/2     Running   0
```

**关键：`2/2 READY` 表示 Envoy sidecar 已注入成功。**

### 5.3 验证服务调用

```bash
# 直接访问 ProductService
kubectl exec -it deploy/order-service -c order-service -- \
  curl -s http://product-service:8082/products/P001
# {"productId":"P001","name":"iPhone 15 Pro Max","price":9999,"instance":"order-service-xxx"}

# 通过 OrderService 调用链
kubectl exec -it deploy/order-service -c order-service -- \
  curl -s http://localhost:8081/orders/ORD-001
# {"orderId":"ORD-001","product":{...},"status":"CONFIRMED","instance":"order-service-xxx"}
```

---

## 6. Istio 流量管理

### 6.1 VirtualService（流量路由）

```yaml
# istio/traffic-management.yaml
---
# OrderService 路由（全部流量到 v1）
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 100
---
# OrderService 目标规则（定义 v1 子集）
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  subsets:
  - name: v1
    labels:
      version: v1
---
# ProductService 路由
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
  - product-service
  http:
  - route:
    - destination:
        host: product-service
        subset: v1
      weight: 100
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: product-service
spec:
  host: product-service
  subsets:
  - name: v1
    labels:
      version: v1
```

```bash
kubectl apply -f istio/traffic-management.yaml
```

### 6.2 金丝雀发布（90/10 流量分割）

**场景：ProductService 发布 v2 版本，先给 10% 用户试用**

```bash
# 1. 部署 v2（加 version=v2 标签）
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service-v2
  labels:
    app: product-service
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-service
      version: v2
  template:
    metadata:
      labels:
        app: product-service
        version: v2
    spec:
      containers:
      - name: product-service
        image: product-service:latest
        ports:
        - containerPort: 8082
        env:
        - name: PRODUCT_NAME
          value: "iPhone 16 Pro Max（新版本）"
EOF

# 2. 修改 VirtualService：90% v1，10% v2
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
  - product-service
  http:
  - route:
    - destination:
        host: product-service
        subset: v1
      weight: 90
    - destination:
        host: product-service
        subset: v2
      weight: 10
EOF

# 3. 添加 v2 子集到 DestinationRule
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: product-service
spec:
  host: product-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF

# 4. 验证（多打几次看 10% 流量到 v2）
for i in $(seq 1 20); do
  kubectl exec -it deploy/order-service -c order-service -- \
    curl -s http://product-service:8082/products/P001 | grep -o '"name":"[^"]*"'
done
# 大约 2/20 次会显示 "iPhone 16 Pro Max（新版本）"
```

### 6.3 故障注入（模拟网络延迟/错误）

```yaml
# istio/fault-injection.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
  - product-service
  http:
  - fault:
      delay:
        percentage:
          value: 30          # 30% 的请求延迟 3 秒
        fixedDelay: 3s
      abort:
        percentage:
          value: 10          # 10% 的请求返回 503 错误
        httpStatus: 503
    route:
    - destination:
        host: product-service
        subset: v1
      weight: 100
```

```bash
kubectl apply -f istio/fault-injection.yaml

# 验证：部分请求变慢或报错
time kubectl exec -it deploy/order-service -c order-service -- \
  curl -s http://product-service:8082/products/P001
# 30% 概率等 3 秒，10% 概率返回错误
```

### 6.4 超时和重试

```yaml
# istio/timeout-retry.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
  - product-service
  http:
  - timeout: 5s              # 5秒超时
    retries:
      attempts: 3            # 重试3次
      perTryTimeout: 2s      # 每次重试2秒
      retryOn: 5xx,reset     # 遇到5xx或连接断开时重试
    route:
    - destination:
        host: product-service
        subset: v1
      weight: 100
```

---

## 7. Istio 安全（mTLS + 授权）

### 7.1 自动 mTLS（零代码加密）

```bash
# 全部命名空间启用 mTLS（服务间自动加密通信）
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF

# 验证：Pod 之间的通信已经是加密的
istioctl x describe pod $(kubectl get pod -l app=order-service -o jsonpath='{.items[0].metadata.name}')
# 应该显示 mTLS: STRICT
```

### 7.2 授权策略（谁能访问谁）

```yaml
# istio/authorization.yaml
---
# 只允许 order-service 访问 product-service
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: product-service-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/default"]
    to:
    - operation:
        methods: ["GET"]
---
# 禁止所有其他服务访问 product-service
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: product-service-deny-all
  namespace: default
spec:
  selector:
    matchLabels:
      app: product-service
  action: DENY
  rules:
  - to:
    - operation:
        notMethods: ["GET"]
```

```bash
kubectl apply -f istio/authorization.yaml

# 验证：只有 GET 允许，POST 被拒绝
kubectl exec -it deploy/order-service -c order-service -- \
  curl -s -X POST http://product-service:8082/products
# 403 Forbidden
```

---

## 8. 可观测性（Grafana + Kiali + Jaeger）

### 8.1 安装观测组件

```bash
# Istio demo profile 已包含 Grafana 和 Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/kiali.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/jaeger.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/prometheus.yaml

# 等待就绪
kubectl wait --for=condition=Ready pod -l app=kiali -n istio-system --timeout=120s
```

### 8.2 访问 Dashboard

```bash
# Kiali（流量拓扑图）
istioctl dashboard kiali

# Grafana（指标面板）
istioctl dashboard grafana

# Jaeger（分布式追踪）
istioctl dashboard jaeger

# 或者用 minikube service 访问
minikube service kiali -n istio-system --url
minikube service grafana -n istio-system --url
```

### 8.3 生成流量后查看

```bash
# 持续生成请求
for i in $(seq 1 100); do
  kubectl exec -it deploy/order-service -c order-service -- \
    curl -s http://product-service:8082/products/P001 > /dev/null
  sleep 0.5
done

# 打开 Kiali 看流量拓扑图
istioctl dashboard kiali
# 你会看到：order-service → product-service 的调用链，带流量百分比和延迟
```

---

## 9. 常见问题

**Q: Sidecar 没注入（1/1 READY）？**
→ 命名空间没开注入：`kubectl label namespace default istio-injection=enabled`

**Q: 服务间调不通？**
→ 检查 mTLS 模式：`istioctl x describe pod <pod-name>` 看是否 mTLS 冲突

**Q: VirtualService 不生效？**
→ 确保 DestinationRule 里的 subset 名字和 VirtualService 一致

**Q: Istio 安装后 Pod 资源不够？**
→ minikube 内存调到 4G+：`minikube start --memory=4096`

**Q: Kiali 看不到数据？**
→ 需要先有流量：curl 几次服务后再看

---

## 10. 清理

```bash
# 删除 Istio 资源
kubectl delete -f istio/
kubectl delete -f k8s-deployments.yaml

# 卸载 Istio
istioctl uninstall --purge -y
kubectl delete namespace istio-system

# 恢复 minikube
minikube stop
minikube start
```
