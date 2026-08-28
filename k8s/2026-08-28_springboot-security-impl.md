# 微服务联调测试 —— 简化计划

> 🎯 3 个轻量 Spring Boot 服务，纯内存数据，零中间件，验证 Ingress auth-url + @HttpExchange 全链路
> 📝 基于你之前的需求文档

---

## 一、架构图

```
用户 → Nginx Ingress ──auth-url──→ Auth Service (8080)
                    │                     │ 验证 Token → 返回 X-User-* 头
                    │                     ↓
                    ├──/api/public/**──→ 直接放行（不走 Auth）
                    └──/api/**──────→ User Service (8081)
                                      └── @HttpExchange ──→ Order Service (8082)
                                            （自动透传用户信息）
```

## 二、服务清单

| 服务 | 端口 | 用途 | 数据来源 |
|---|---|---|---|
| **auth-service** | 8080 | JWT 验证 + 透传用户头 | 硬编码（内存模拟） |
| **user-service** | 8081 | 用户业务 + AOP 权限 + OrderClient 调用 | 硬编码（内存模拟） |
| **order-service** | 8082 | 订单业务（被 User 调用） | 硬编码（内存模拟） |

**三个服务都不连数据库、不连 Redis、不连 MQ。**

## 三、技术栈

| 项 | 选型 |
|---|---|
| 框架 | Spring Boot 3.3.x + Java 17+ |
| HTTP 客户端 | RestClient + @HttpExchange（**不用 OpenFeign**） |
| 权限校验 | 自定义注解 + AOP |
| 依赖 | spring-boot-starter-web + spring-boot-starter-aop |
| 构建 | Maven multi-module（父 POM + 3 个子模块） |

## 四、项目结构

```
microservice-demo/
├── pom.xml                          ← 父 POM（统一版本管理）
├── auth-service/
│   ├── pom.xml
│   └── src/main/java/.../auth/
│       ├── controller/AuthController.java    ← GET /auth/validate
│       └── service/JwtService.java           ← 硬编码 Token 验证
├── user-service/
│   ├── pom.xml
│   └── src/main/java/.../user/
│       ├── context/UserContext.java          ← ThreadLocal
│       ├── context/UserContextInterceptor.java ← 拦截器
│       ├── context/WebConfig.java            ← 拦截器注册
│       ├── annotation/RequirePermission.java ← 权限注解
│       ├── aspect/PermissionAspect.java      ← AOP 切面
│       ├── http/AuthClientHttpRequestInterceptor.java ← RestClient 拦截器（透传 Token）
│       ├── http/exchange/OrderClient.java    ← @HttpExchange 接口
│       ├── config/HttpClientConfig.java      ← RestClient + Proxy 生成
│       ├── controller/UserController.java    ← 业务 Controller
│       └── model/UserDto.java               ← 用户 DTO
├── order-service/
│   ├── pom.xml
│   └── src/main/java/.../order/
│       ├── controller/OrderController.java   ← 订单 Controller
│       └── model/OrderDto.java               ← 订单 DTO
```

## 五、核心实现细节

### 5.1 Auth Service（8080）

```java
// 硬编码 Token 验证，不连任何中间件
@Service
public class JwtService {
    // 硬编码用户数据库
    private static final Map<String, Map<String, Object>> USERS = Map.of(
        "token-admin-001", Map.of("id", "1001", "name", "张三", "roles", "ROLE_ADMIN,ROLE_USER", "permissions", "user:read,user:write,order:read,order:write"),
        "token-user-002",  Map.of("id", "1002", "name", "李四", "roles", "ROLE_USER",        "permissions", "user:read,order:read")
    );

    public Map<String, String> validate(String token) {
        if (token == null || token.isBlank()) return null;
        Map<String, Object> user = USERS.get(token);
        if (user == null) return null;
        // 返回用户信息（Ingress 透传给下游）
        return Map.of(
            "X-User-Id", (String) user.get("id"),
            "X-User-Name", (String) user.get("name"),
            "X-User-Roles", (String) user.get("roles"),
            "X-User-Permissions", (String) user.get("permissions")
        );
    }
}
```

```java
// GET /auth/validate —— Ingress auth-url 调用的接口
@RestController
public class AuthController {
    @Autowired private JwtService jwtService;

    @GetMapping("/auth/validate")
    public ResponseEntity<Void> validate(
            @RequestHeader(value = "Authorization", required = false) String auth,
            HttpServletResponse response) {
        if (auth == null || !auth.startsWith("Bearer ")) {
            return ResponseEntity.status(401).build();
        }
        String token = auth.substring(7);
        Map<String, String> headers = jwtService.validate(token);
        if (headers == null) {
            return ResponseEntity.status(401).build();
        }
        // 把用户信息写入响应头（Ingress 会透传给下游）
        headers.forEach(response::setHeader);
        return ResponseEntity.ok().build();
    }
}
```

### 5.2 User Service（8081）—— 核心链路

```java
// UserContext —— ThreadLocal 存储当前用户
public class UserContext {
    private static final ThreadLocal<UserInfo> HOLDER = new ThreadLocal<>();
    public static void set(UserInfo info) { HOLDER.set(info); }
    public static UserInfo get() { return HOLDER.get(); }
    public static void clear() { HOLDER.remove(); }

    public record UserInfo(String userId, String userName, List<String> roles, List<String> permissions, String token) {}
}
```

```java
// 拦截器 —— 从 Ingress 透传的 X-User-* 头提取用户信息
@Component
public class UserContextInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        String userId = req.getHeader("X-User-Id");
        if (userId != null) {
            UserContext.set(new UserContext.UserInfo(
                userId,
                req.getHeader("X-User-Name"),
                List.of(req.getHeader("X-User-Roles").split(",")),
                List.of(req.getHeader("X-User-Permissions").split(",")),
                req.getHeader("Authorization")
            ));
        }
        return true;
    }
    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex) {
        UserContext.clear();
    }
}
```

```java
// @RequirePermission 注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission { String value(); }
```

```java
// AOP 切面 —— 校验权限
@Aspect @Component
public class PermissionAspect {
    @Around("@annotation(requirePermission)")
    public Object check(ProceedingJoinPoint joinPoint, RequirePermission requirePermission) throws Throwable {
        UserContext.UserInfo user = UserContext.get();
        if (user == null || user.permissions() == null || !user.permissions().contains(requirePermission.value())) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN, "无权限: " + requirePermission.value());
        }
        return joinPoint.proceed();
    }
}
```

```java
// @HttpExchange —— 声明式 Order 服务客户端
@HttpExchange
public interface OrderClient {
    @GetExchange("/api/orders/{id}")
    OrderDto getOrder(@PathVariable String id);

    @GetExchange("/api/orders/user/{userId}")
    List<OrderDto> getOrdersByUser(@PathVariable String userId);
}
```

```java
// RestClient 拦截器 —— 自动透传用户信息到下游
@Component
public class AuthClientHttpRequestInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest req, byte[] body, ClientHttpRequestExecution exec) {
        UserContext.UserInfo user = UserContext.get();
        if (user != null) {
            req.getHeaders().set("Authorization", user.token());
            req.getHeaders().set("X-User-Id", user.userId());
            req.getHeaders().set("X-User-Name", user.userName());
            req.getHeaders().set("X-User-Roles", String.join(",", user.roles()));
            req.getHeaders().set("X-User-Permissions", String.join(",", user.permissions()));
        }
        return exec.execute(req, body);
    }
}
```

```java
// UserController —— 业务示例
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired private OrderClient orderClient;

    // 公开接口（不需认证）
    @GetMapping("/public/health")
    public Map<String, String> health() { return Map.of("status", "ok"); }

    // 需要 user:read 权限
    @GetMapping("/{id}")
    @RequirePermission("user:read")
    public UserDto getUser(@PathVariable String id) {
        return new UserDto(id, "用户" + id, "active");
    }

    // 需要 user:read 权限 + 调用 Order 服务
    @GetMapping("/{id}/orders")
    @RequirePermission("user:read")
    public List<OrderDto> getUserOrders(@PathVariable String id) {
        return orderClient.getOrdersByUser(id);  // 自动透传 Token
    }

    // 需要 user:write 权限
    @PostMapping
    @RequirePermission("user:write")
    public UserDto createUser(@RequestBody UserDto user) {
        return user;
    }
}
```

## 六、测试场景

### 场景 1：公开接口（不走 Auth）
```bash
curl http://localhost:8081/api/users/public/health
# → {"status": "ok"}  ✅ 不需要认证
```

### 场景 2：需要认证 + 权限校验通过
```bash
# 先模拟 Ingress 的行为：调 Auth 验证 Token，拿到用户头
curl -v http://localhost:8080/auth/validate -H "Authorization: Bearer token-admin-001"
# → 返回 X-User-Id: 1001, X-User-Roles: ROLE_ADMIN ...

# 用这个 Token 访问 User 服务（需要 user:read 权限）
curl http://localhost:8081/api/users/1001 -H "Authorization: Bearer token-admin-001"
# → {"id":"1001","name":"用户1001","status":"active"}  ✅
```

### 场景 3：权限不足（403）
```bash
# user-002 只有 user:read,order:read，没有 user:write
curl -X POST http://localhost:8081/api/users -H "Authorization: Bearer token-user-002" -d '{"id":"1003","name":"new"}'
# → 403 Forbidden: 无权限: user:write  ✅
```

### 场景 4：服务间调用（自动透传）
```bash
# user:1001 的订单（User → Order）
curl http://localhost:8081/api/users/1001/orders -H "Authorization: Bearer token-admin-001"
# → [{"orderId":"ORD-001","userId":"1001","product":"组件","amount":100}]
# Order 服务收到的请求头里有 X-User-Id: 1001  ✅
```

### 场景 5：Minikube + Ingress 完整链路
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: user-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "http://auth-service:8080/auth/validate"
    nginx.ingress.kubernetes.io/auth-response-headers: "X-User-Id, X-User-Name, X-User-Roles, X-User-Permissions"
spec:
  ingressClassName: nginx
  rules:
  - host: test.local
    http:
      paths:
      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8081
```

```bash
# 直接访问 Ingress（不手动传 Token 头）
curl -H "Authorization: Bearer token-admin-001" http://test.local/api/users/1001
# → Auth Service 验证 → 透传 X-User-* → User Service → 403 ❌ 因为没有通过 Ingress
```

## 七、实施步骤

| 步骤 | 内容 | 时间 |
|---|---|---|
| 1 | 创建父 POM + 3 个子模块骨架 | 10min |
| 2 | Auth Service：AuthController + JwtService | 15min |
| 3 | Order Service：OrderController（硬编码数据） | 10min |
| 4 | User Service：UserContext + 拦截器 + WebConfig | 15min |
| 5 | User Service：@RequirePermission + PermissionAspect | 10min |
| 6 | User Service：@HttpExchange + RestClient 拦截器 + HttpClientConfig | 20min |
| 7 | User Service：UserController（完整业务示例） | 15min |
| 8 | 测试验证（curl 5 个场景） | 15min |
| 9 | Minikube 部署测试（kubectl apply + Ingress） | 30min |

**总计约 2.5 小时**

## 八、文件清单（17 个文件）

```
microservice-demo/
├── pom.xml
├── auth-service/
│   ├── pom.xml
│   └── src/main/java/.../
│       ├── AuthApplication.java
│       ├── controller/AuthController.java
│       └── service/JwtService.java
├── user-service/
│   ├── pom.xml
│   └── src/main/java/.../
│       ├── UserApplication.java
│       ├── context/UserContext.java
│       ├── context/UserContextInterceptor.java
│       ├── context/WebConfig.java
│       ├── annotation/RequirePermission.java
│       ├── aspect/PermissionAspect.java
│       ├── http/AuthClientHttpRequestInterceptor.java
│       ├── http/exchange/OrderClient.java
│       ├── config/HttpClientConfig.java
│       ├── controller/UserController.java
│       └── model/UserDto.java
├── order-service/
│   ├── pom.xml
│   └── src/main/java/.../
│       ├── OrderApplication.java
│       ├── controller/OrderController.java
│       └── model/OrderDto.java
```
