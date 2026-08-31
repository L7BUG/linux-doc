# 微服务联调测试 —— 实施计划

> 🎯 3 个轻量 Spring Boot 服务，纯内存数据，零中间件，验证 Ingress auth-url + @HttpExchange 全链路
> 📝 基于需求文档（UserContext + AOP权限 + @HttpExchange + 白名单）
> ✅ 代码已实现：https://github.com/L7-BUG-AI/microservice-demo

---

## 一、架构图

```
用户 → Nginx Ingress(80/443)
                    │
                    ├── /api/** ──────→ User Service(8081)
                    │                     │ ① Interceptor 检查 X-User-* 头
                    │                     │ ② 白名单 /api/**/public/** 直接放行
                    │                     │ ③ AOP @RequirePermission 校验权限
                    │                     │
                    │                     ├── @HttpExchange → Order Service(8082)
                    │                     │   （自动透传 Token + 用户头）
                    │                     │
                    │                     └── Order Service → @HttpExchange → User Service
                    │                         （双向调用）
                    │
                    └── auth-url ──→ Auth Service(8080)
                                      ① 验证 Token（硬编码）
                                      ② 返回 X-User-* 响应头
                                      ③ Ingress 透传给下游
```

## 二、服务清单

| 服务 | 容器端口 | Service端口 | 用途 | 镜像 |
|---|---|---|---|---|
| **auth-service** | 8080 | 80 | JWT验证 + 透传用户头 | `l7bug/auth-service:latest` |
| **user-service** | 8081 | 80 | 用户业务 + AOP权限 + 调Order | `l7bug/user-service:latest` |
| **order-service** | 8082 | 80 | 订单业务（被User调） + 调User | `l7bug/order-service:latest` |

## 三、技术栈

| 项 | 选型 |
|---|---|
| 框架 | Spring Boot 3.4.1 + Java 21 |
| HTTP客户端 | RestClient + @HttpExchange（**不用 OpenFeign**） |
| 权限校验 | 自定义注解 @RequirePermission + AOP |
| 白名单 | UserContextInterceptor 白名单路径直接放行 |
| 依赖 | spring-boot-starter-web + spring-boot-starter-aop |
| 构建 | Maven multi-module（父POM + 3个子模块） |
| Docker | 多阶段构建（maven:3.9.9-jdk21 + eclipse-temurin:21-jre-alpine） |
| Helm | 3个独立 Chart，Service port:80，resources.limits.memory:512Mi |

## 四、项目结构

```
microservice-demo/
├── pom.xml                          ← 父 POM
├── Dockerfile                       ← 通用多阶段构建
├── auth-service/
│   ├── pom.xml
│   └── src/main/java/.../auth/
│       ├── AuthApplication.java
│       ├── controller/AuthController.java    ← GET /auth/validate
│       └── service/JwtService.java           ← 硬编码Token验证
├── user-service/
│   ├── pom.xml
│   └── src/main/java/.../user/
│       ├── UserApplication.java
│       ├── context/UserContext.java          ← ThreadLocal
│       ├── context/UserContextInterceptor.java ← 拦截器 + 白名单
│       ├── context/WebConfig.java
│       ├── annotation/RequirePermission.java
│       ├── aspect/PermissionAspect.java      ← AOP（403）
│       ├── http/AuthClientHttpRequestInterceptor.java ← 透传Token
│       ├── http/exchange/OrderClient.java    ← @HttpExchange调Order
│       ├── config/HttpClientConfig.java      ← 配置化地址
│       ├── controller/UserController.java
│       └── model/UserDto.java
├── order-service/
│   ├── pom.xml
│   └── src/main/java/.../order/
│       ├── OrderApplication.java
│       ├── context/UserContext.java          ← ThreadLocal
│       ├── context/UserContextInterceptor.java ← 拦截器 + 白名单
│       ├── context/WebConfig.java
│       ├── http/AuthClientHttpRequestInterceptor.java ← 透传Token
│       ├── http/exchange/UserClient.java     ← @HttpExchange调User（双向）
│       ├── config/HttpClientConfig.java      ← 配置化地址
│       ├── controller/OrderController.java
│       └── model/OrderDto.java
└── helm/
    ├── auth-service/    ← Deployment + Service
    ├── order-service/   ← Deployment + Service
    └── user-service/    ← Deployment + Service + 2个Ingress
```

## 五、核心实现

### 5.1 Auth Service（硬编码验证）

```java
// JwtService —— 硬编码2个Token用户，不连中间件
private static final Map<String, Map<String, Object>> USERS = Map.of(
    "token-admin-001", Map.of("id","1001","name","张三","roles","ROLE_ADMIN,ROLE_USER","permissions","user:read,user:write,order:read,order:write"),
    "token-user-002",  Map.of("id","1002","name","李四","roles","ROLE_USER","permissions","user:read,order:read")
);
```

### 5.2 白名单（UserContextInterceptor）

```java
private static final Set<String> WHITE_LIST = Set.of(
    "/api/users/public/",   // 公开接口
    "/health",
    "/error"
);

@Override
public boolean preHandle(...) {
    if (isWhiteListed(path)) return true;        // 白名单 → 直接放行
    String userId = req.getHeader("X-User-Id");
    if (userId == null || userId.isBlank()) {     // 非白名单无头 → 401
        response.setStatus(401); return false;
    }
    UserContext.set(...);                          // 有头 → 填充Context
    return true;
}
```

### 5.3 双向调用

```java
// User → Order（UserClient）
@GetExchange("/api/orders/user/{userId}")
List<OrderDto> getOrdersByUser(@PathVariable String userId);

// Order → User（UserClient）—— 新增
@GetExchange("/api/users/{id}")
Map<String, Object> getUser(@PathVariable String id);
```

### 5.4 配置化地址

```yaml
# application.yml
order-service:
  url: http://localhost:8082   # 本地开发
# K8s 里通过环境变量 ORDER_SERVICE_URL 覆盖
# Helm 里配置：http://order-service:8082
```

## 六、测试场景

| 场景 | 命令 | 结果 |
|---|---|---|
| Auth验证Token | `GET /auth/validate -H "Authorization: Bearer token-admin-001"` | 200 + X-User-* 头 |
| 公开接口（不传头） | `GET /api/users/public/health` | 200 直接放行 |
| 需要权限（传头） | `GET /api/users/1001 -H "X-User-Id: 1001" ...` | 200 |
| 权限不足 | `POST /api/users`（user-002无user:write） | **403 Forbidden** |
| User→Order调用 | `GET /api/users/1001/orders` | 透传成功，返回2条订单 |
| Order→User反向调用 | `GET /api/orders/ORD-001/detail` | 返回订单+买家信息 |

## 七、Helm 部署

```bash
# 部署
helm install auth helm/auth-service
helm install order helm/order-service
helm install user helm/user-service

# 验证
helm template auth helm/auth-service
helm template order helm/order-service
helm template user helm/user-service
```

### Helm values 关键配置

```yaml
# 所有服务
resources:
  limits:
    cpu: 200m
    memory: 512Mi      # ← 统一512Mi
  requests:
    cpu: 100m
    memory: 256Mi

service:
  port: 80             # Service对外端口
server:
  port: 8080           # 容器内端口（Spring Boot）
```

### user-service Ingress（两个）

| Ingress | 路径 | auth-url | 用途 |
|---|---|---|---|
| `user-service` | `/api/users` | `http://auth-service:8080/auth/validate` | 业务接口（需认证） |
| `user-service-public` | `/api/users/public` | 无 | 公开接口（免认证） |

**auth-service 不需要 Ingress** —— 它是内部验证服务，被 Ingress 的 auth-url 通过 K8s Service 调用。

## 八、Docker 镜像

```bash
# 构建
docker build --build-arg SERVICE=auth-service -t l7bug/auth-service:latest .
docker build --build-arg SERVICE=order-service -t l7bug/order-service:latest .
docker build --build-arg SERVICE=user-service -t l7bug/user-service:latest .

# 推送
docker push l7bug/auth-service:latest
docker push l7bug/order-service:latest
docker push l7bug/user-service:latest
```

## 九、K8s 内服务调用

```
user-service → http://order-service:8082（Helm values配置）
order-service → http://user-service:8081（Helm values配置）
auth-service → 内部验证，不调其他服务
```

## 十、实施步骤

| 步骤 | 内容 | 状态 |
|---|---|---|
| 1 | 创建父POM + 3个子模块骨架 | ✅ |
| 2 | Auth Service：AuthController + JwtService | ✅ |
| 3 | Order Service：OrderController（硬编码数据） | ✅ |
| 4 | User Service：UserContext + 拦截器 + 白名单 | ✅ |
| 5 | User Service：@RequirePermission + PermissionAspect | ✅ |
| 6 | User Service：@HttpExchange + RestClient拦截器 | ✅ |
| 7 | Order Service：反向调用 User（UserClient） | ✅ |
| 8 | 配置化服务地址（环境变量覆盖） | ✅ |
| 9 | 测试验证（6个场景） | ✅ |
| 10 | Docker镜像构建 + 推送 | ✅ |
| 11 | Helm Charts（3个，Service port:80，512Mi） | ✅ |
| 12 | minikube部署测试 | 待做 |
