# Istio 服务网格详解

> K8s 微服务的"智能管家"——流量管理、熔断、安全、可观测性

---

## 1. Istio 是什么

**一句话：Istio = 服务间通信的"中间件"。**

你的微服务部署在 K8s 后，Istio 帮你处理服务之间的所有通信问题，业务代码不用改。

### 没有 Istio

```
User Service ←→ Order Service（直连）
  - 你自己写重试逻辑
  - 你自己写熔断逻辑
  - 你自己写超时逻辑
  - 你自己处理加密
  - 你自己收集监控指标
```

### 有 Istio

```
User Service ←→ Envoy Sidecar ←→ Envoy Sidecar ←→ Order Service
                       ↑                    ↑
                    自动重试            自动熔断
                    自动超时            自动负载均衡
                    自动加密(mTLS)      自动收集指标
                    自动故障注入        自动金丝雀发布
```

---

## 2. 架构

### 2.1 控制平面（istiod）

```
┌─────────────────────────────────────────────────────────┐
│                    istiod（控制平面）                     │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Pilot   │  │ Citadel  │  │  Galley  │  │ Mixer  │ │
│  │ 配置下发  │  │ 证书管理  │  │ 配置验证  │  │指标收集│ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
│                                                         │
│  职责：                                                  │
│  - 接收用户的 VirtualService/DestinationRule 配置        │
│  - 翻译成 Envoy 能理解的 xDS 配置                        │
│  - 下发给所有 Envoy Sidecar                              │
│  - 管理 mTLS 证书                                        │
│  - 收集遥测数据（metrics/logs/traces）                   │
└─────────────────────────────────────────────────────────┘
```

### 2.2 数据平面（Envoy Sidecar）

```
┌─────────────────────────────────────────────────────────┐
│  Pod                                                     │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │  你的业务代码  │←→│ Envoy Sidecar│ ←── istiod        │
│  │ (User Service)│  │ (流量代理)    │    配置下发       │
│  │   端口: 8081  │  │  端口: 15006 │                    │
│  └──────────────┘  └──────────────┘                    │
│         ↑                  ↑                            │
│    业务流量              拦截所有入站/出站流量            │
└─────────────────────────────────────────────────────────┘
```

**Envoy Sidecar 的工作：**

1. **拦截流量**：iptables 规则自动拦截所有入站/出站流量
2. **处理流量**：重试、超时、熔断、负载均衡、路由
3. **加密通信**：mTLS（服务间自动加密）
4. **收集指标**：请求数、延迟、错误率
5. **注入 Header**：可以自动添加/修改请求头

### 2.3 完整架构图

```
                        ┌──────────────────────────────────┐
                        │         Istio 控制平面            │
                        │           (istiod)               │
                        │    Pilot + Citadel + Galley      │
                        └───────────┬──────────────────────┘
                                    │ xDS 配置下发
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
        ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
        │  Envoy Sidecar   │ │  Envoy Sidecar   │ │  Envoy Sidecar   │
        │  (User Service)  │ │  (Order Service) │ │  (Auth Service)  │
        ├──────────────────┤ ├──────────────────┤ ├──────────────────┤
        │   User Service   │ │  Order Service   │ │  Auth Service    │
        │     (8081)       │ │     (8082)       │ │     (8080)       │
        └──────────────────┘ └──────────────────┘ └──────────────────┘
```

---

## 3. 核心概念

### 3.1 概念速查表

| 概念 | 作用 | 类比 |
|---|---|---|
| **Envoy Sidecar** | 拦截并代理所有流量 | 门卫 |
| **istiod** | 管理所有 Envoy 的配置 | 大脑 |
| **VirtualService** | 定义路由规则（流量怎么分） | 路牌 |
| **DestinationRule** | 定义负载均衡/熔断/超时策略 | 交通规则 |
| **Gateway** | 入口网关（替代 Ingress） | 大门 |
| **PeerAuthentication** | mTLS 服务间加密 | 保密通道 |
| **AuthorizationPolicy** | 访问控制（谁能调谁） | 门禁卡 |

### 3.2 VirtualService（路由规则）

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service           # 目标服务
  http:
  - route:
    - destination:
        host: order-service
        subset: v1          # 路由到 v1 版本
      weight: 90            # 90% 流量
    - destination:
        host: order-service
        subset: v2          # 路由到 v2 版本
      weight: 10            # 10% 流量
```

### 3.3 DestinationRule（流量策略）

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  subsets:                  # 定义版本子集
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  trafficPolicy:            # 流量策略
    connectionPool:         # 连接池配置
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:       # 异常检测（熔断）
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

---

## 4. 核心功能详解

### 4.1 流量管理

#### 金丝雀发布（Canary Release）

```yaml
# 90% 流量到 v1，10% 到 v2
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
      weight: 90
    - destination:
        host: order-service
        subset: v2
      weight: 10
```

#### A/B 测试（按 Header 路由）

```yaml
# 带 X-User-Type: beta 的用户走 v2
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        X-User-Type:
          exact: beta
    route:
    - destination:
        host: order-service
        subset: v2
  - route:
    - destination:
        host: order-service
        subset: v1
```

#### 超时 + 重试

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - timeout: 5s                  # 超时 5 秒
    retries:
      attempts: 3                # 重试 3 次
      perTryTimeout: 2s          # 每次重试超时 2 秒
      retryOn: 5xx               # 5xx 错误时重试
    route:
    - destination:
        host: order-service
```

### 4.2 熔断（Circuit Breaker）

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100       # 最大 TCP 连接数
      http:
        http1MaxPendingRequests: 100  # 最大等待请求数
        http2MaxRequests: 1000        # 最大并发请求数
    outlierDetection:             # 异常检测
      consecutive5xxErrors: 5     # 连续 5 次 5xx → 熔断
      interval: 30s               # 每 30 秒检测一次
      baseEjectionTime: 30s       # 熔断持续 30 秒
      maxEjectionPercent: 100     # 最大熔断比例（100% = 全部熔断）
```

**熔断状态：**

```
关闭状态（正常）
    ↓ 连续失败 ≥ 5 次
打开状态（熔断，拒绝所有请求）
    ↓ 等待 30 秒
半开状态（放行 1 个请求试探）
    ↓ 成功 → 关闭状态
    ↓ 失败 → 打开状态
```

### 4.3 故障注入（测试用）

```yaml
# 模拟 Order Service 延迟 + 返回 500
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - fault:
      delay:
        percentage:
          value: 10               # 10% 的请求延迟
        fixedDelay: 5s            # 延迟 5 秒
      abort:
        percentage:
          value: 5                # 5% 的请求返回 500
        httpStatus: 500
    route:
    - destination:
        host: order-service
```

### 4.4 安全（mTLS）

```yaml
# 全自动 mTLS：服务间通信自动加密
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT                  # 强制 mTLS
```

**mTLS 工作原理：**

```
User Service                    Order Service
     │                                │
     │ ① 证书握手（自动）              │
     ├───────────────────────────────→│
     │                                │
     │ ② 加密通信                     │
     ├═══════════════════════════════→│
     │                                │
     │ ③ 双向验证                     │
     ├───────────────────────────────→│
```

**无需改代码**：Envoy Sidecar 自动处理证书、加密、验证。

### 4.5 访问控制（AuthorizationPolicy）

```yaml
# 只允许 User Service 调用 Order Service
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/user-service"]
```

---

## 5. 可观测性

### 5.1 自动收集的指标

| 指标 | 说明 | 来源 |
|---|---|---|
| `istio_requests_total` | 请求总数 | Envoy |
| `istio_request_duration_milliseconds` | 请求延迟 | Envoy |
| `istio_request_size` | 请求大小 | Envoy |
| `istio_response_size` | 响应大小 | Envoy |
| `istio_tcp_connections_opened` | TCP 连接数 | Envoy |

### 5.2 分布式追踪（Jaeger）

```
用户请求
    ↓
Ingress Gateway
    ↓ trace-id: abc123
User Service (2ms)
    ↓ trace-id: abc123
Order Service (15ms)
    ↓ trace-id: abc123
Auth Service (3ms)
    ↓
返回用户
```

**无需改代码**：Envoy 自动注入 trace-id，自动上报到 Jaeger。

### 5.3 日志（Kiali）

```
┌─────────────────────────────────────────┐
│         Kiali 拓扑图                     │
│                                         │
│  User Service ──→ Order Service         │
│       │              │                  │
│       ↓              ↓                  │
│  Auth Service   Auth Service            │
│                                         │
│  颜色：绿色=正常  黄色=警告  红色=错误   │
└─────────────────────────────────────────┘
```

---

## 6. 安装 Istio

### 6.1 安装 istioctl

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
```

### 6.2 安装 Istio

```bash
# Demo 配置（学习用，资源占用小）
istioctl install --set profile=demo -y

# 生产配置（完整功能）
istioctl install --set profile=default -y
```

### 6.3 启用 Sidecar 自动注入

```bash
# 给 default namespace 启用自动注入
kubectl label namespace default istio-injection=enabled

# 验证
kubectl get namespace -L istio-injection
```

### 6.4 部署应用

```bash
# 普通部署（Istio 会自动注入 Envoy Sidecar）
kubectl apply -f deployment.yaml

# 验证 Sidecar 注入
kubectl get pods
# 每个 Pod 应该有 2/2 READY（业务容器 + Envoy Sidecar）
```

---

## 7. 常用命令

```bash
# 查看 Istio 状态
istioctl version
istioctl analyze

# 查看 Envoy 配置
istioctl proxy-config routes <pod-name>
istioctl proxy-config clusters <pod-name>

# 查看代理状态
istioctl proxy-status

# 手动注入 Sidecar（不修改 deployment.yaml）
istioctl kube-inject -f deployment.yaml | kubectl apply -f -

# 卸载 Istio
istioctl uninstall --purge -y
kubectl delete namespace istio-system
```

---

## 8. Istio vs Resilience4j 对比

| 功能 | Istio（网络层） | Resilience4j（代码层） |
|---|---|---|
| 熔断 | ✅ DestinationRule | ✅ @CircuitBreaker |
| 重试 | ✅ VirtualService | ✅ @Retry |
| 超时 | ✅ VirtualService | ✅ @TimeLimiter |
| 负载均衡 | ✅ 自动 | ❌ 需要自己实现 |
| mTLS | ✅ 自动 | ❌ 需要自己实现 |
| 金丝雀发布 | ✅ VirtualService + weight | ❌ 需要自己实现 |
| 故障注入 | ✅ VirtualService fault | ❌ 不支持 |
| 可观测性 | ✅ 自动 metrics/logs/traces | ❌ 需要自己集成 |
| 代码侵入性 | ❌ 零侵入（YAML 配置） | ✅ 需要加注解 |
| 运维复杂度 | ⭐⭐⭐ 高（需要 istiod） | ⭐ 低（加依赖即可） |
| 适合场景 | 大规模微服务 | 小中规模微服务 |

---

## 9. 什么时候用 Istio

### 适合用 Istio

| 场景 | 原因 |
|---|---|
| 微服务数量 > 20 | 统一管理流量，不用每个服务写重试逻辑 |
| 需要金丝雀发布 | VirtualService weight 配置即可 |
| 需要 mTLS | 自动加密，零代码改动 |
| 需要分布式追踪 | 自动注入 trace-id |
| 团队有 K8s 运维经验 | Istio 运维复杂度高 |

### 不适合用 Istio

| 场景 | 原因 |
|---|---|
| 微服务数量 < 5 | 杀鸡用牛刀，Resilience4j 够用 |
| 没有 K8s 运维经验 | Istio 运维复杂，出问题难排查 |
| 性能要求极高 | Sidecar 有额外延迟（约 1-2ms） |
| 单体应用 | 不需要服务网格 |

---

## 10. 总结

```
Istio 的核心价值：
  1. 业务代码零改动（通过 YAML 配置实现所有功能）
  2. 统一管理流量（重试/超时/熔断/负载均衡）
  3. 自动安全（mTLS 加密，无需改代码）
  4. 完整可观测性（metrics/logs/traces 自动收集）

Istio 的代价：
  1. 运维复杂度高（多一个控制平面 istiod）
  2. 资源占用（每个 Pod 多一个 Envoy Sidecar）
  3. 学习成本（需要理解 VirtualService/DestinationRule 等概念）

建议：
  - 小项目 → Resilience4j（代码层）
  - 大项目 → Istio（网络层）
  - 混合方案 → Resilience4j + Istio（代码层 + 网络层）
```
