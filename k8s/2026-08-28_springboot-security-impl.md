# Spring Boot 微服务安全架构实施计划

> 📅 2026-08-28
> 🎯 实现需求：ThreadLocal 用户上下文 + AOP 权限校验 + @HttpExchange 服务间调用
> 📝 基于 email-parent 项目现有架构（COLA 4层 + Spring Boot 4.0.6 + Java 25）

---

## 〇、认证服务（Auth Service）—— 整个架构的入口

> ⚠️ 这是独立于业务服务的**前置认证服务**，Nginx Ingress 通过 `auth-url` 调用它。验证通过后才放行请求到下游业务服务。

### 架构位置

```
用户请求 → Nginx Ingress → auth-url → Auth Service（验证 JWT）
                                         ↓ 验证通过
                                   透传 X-User-* 头 → 业务服务
                                         ↓ 验证失败
                                   Ingress 直接返回 401/403
```

### Auth Service 职责

| 职责 | 说明 |
|---|---|
| JWT 验证 | 校验签名、过期时间、Issuer |
| 用户信息提取 | 从 Token Claims 提取 userId/roles/permissions |
| 身份透传 | 将用户信息写入响应头（Ingress 自动透传给下游） |
| 快速失败 | 无效 Token 直接返回 401，Ingress 拒绝请求 |

### Ingress 配置示例

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-service-ingress
  annotations:
    # 认证服务地址（Ingress 先调它，通过才放行）
    nginx.ingress.kubernetes.io/auth-url: "http://auth-service.default.svc.cluster.local:8080/auth/validate"
    # 透传哪些响应头给下游业务服务
    nginx.ingress.kubernetes.io/auth-response-headers: "X-User-Id, X-User-Name, X-User-Roles, X-User-Permissions"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
```

### Auth Service 接口设计

```
GET /auth/validate
  请求头：Authorization: Bearer eyJhbG...
  ↓
  验证 JWT → 提取 Claims
  ↓
  成功：返回 200 + 响应头
    X-User-Id: u001
    X-User-Name: 张三
    X-User-Roles: admin,user
    X-User-Permissions: order:query,order:create
  ↓
  失败：返回 401（Ingress 拒绝请求，用户看到 401）
```

### Auth Service 技术选型

| 方案 | 推荐度 | 说明 |
|---|---|---|
| **独立 Spring Boot 服务** | ⭐⭐⭐⭐⭐ | 职责单一，可独立扩缩容 |
| 内嵌到业务服务 | ⭐⭐ | 违反微服务原则，不推荐 |

### Auth Service 实现要点

```java
@RestController
@RequestMapping("/auth")
public class AuthController {

    private final JwtDecoder jwtDecoder;  // Nimbus JOSE 或 Spring Security JWT

    @GetMapping("/validate")
    public ResponseEntity<Void> validate(
            @RequestHeader("Authorization") String authorization) {

        // 1. 提取 Token
        String token = authorization.replace("Bearer ", "");

        // 2. 验证 JWT（签名 + 过期 + Claims）
        try {
            Jwt jwt = jwtDecoder.decode(token);

            // 3. 提取用户信息
            String userId = jwt.getClaimAsString("sub");
            List<String> roles = jwt.getClaimAsStringList("roles");
            List<String> permissions = jwt.getClaimAsStringList("permissions");

            // 4. 写入响应头（Ingress 透传给下游）
            return ResponseEntity.ok()
                .header("X-User-Id", userId)
                .header("X-User-Name", jwt.getClaimAsString("name"))
                .header("X-User-Roles", String.join(",", roles))
                .header("X-User-Permissions", String.join(",", permissions))
                .build();

        } catch (Exception e) {
            // 5. 验证失败 → 401
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
    }
}
```

### Auth Service 的 COLA 架构分层

```
auth-service/                          ← 独立 Spring Boot 项目
├── auth-domain/
│   └── JwtValidator.java              ← JWT 验证逻辑（核心业务）
├── auth-adapter/
│   └── AuthController.java            ← /auth/validate 接口
├── auth-app/
│   └── AuthApplication.java           ← 启动类
├── auth-infrastructure/
│   └── JwtDecoderConfig.java          ← JWT 解码器配置（公钥/密钥）
└── start/
    └── pom.xml
```

### Auth Service 与业务服务的关系

```
auth-service（独立部署）
  ├── 验证 JWT Token
  ├── 提取用户信息
  └── 返回 X-User-* 头

业务服务（user-service / order-service 等）
  ├── UserContextInterceptor（从头提取 → UserContext）
  ├── AOP 权限校验（@RequirePermission）
  └── @HttpExchange 透传身份
```

> **Auth Service 是前置网关层，业务服务是后端逻辑层，职责完全分离。**

---

## 一、实施范围

用户需求文档 10 个模块 + Auth Service，在 email-parent COLA 架构下的实现分工：

| 模块 | 归属层 | 说明 |
|---|---|---|
| **0. Auth Service（独立项目）** | **独立** | **JWT 验证 + 身份透传（Ingress auth-url）** |
| 1. UserContext | email-domain（共享） | ThreadLocal 工具类 + UserInfo record |
| 2. UserContextInterceptor | email-adapter | HandlerInterceptor |
| 3. WebConfig | email-adapter | 拦截器注册 |
| 4. RequirePermission 注解 | email-domain（共享） | 自定义注解 |
| 5. PermissionAspect | email-app | AOP 切面 |
| 6. AuthClientHttpRequestInterceptor | email-domain（共享） | RestClient 拦截器 |
| 7. OrderClient | email-domain（共享） | @HttpExchange 接口 |
| 8. HttpClientConfig | email-app | 代理 Bean 配置 |
| 9. 示例 Controller | email-adapter | 演示用法 |
| 10. pom.xml 依赖 | start 模块 | spring-boot-starter-aop |

---

## 二、实施步骤（8 步）

### Step 0：Auth Service（独立项目）
**目录：** `auth-service/`（独立 COLA 项目，和 email-parent 平级）
- auth-domain：JwtValidator（JWT 验证逻辑）
- auth-adapter：AuthController（GET /auth/validate）
- auth-infrastructure：JwtDecoderConfig（JWT 解码器配置）
- auth-app：启动类
- start/pom.xml：依赖（spring-boot-starter-web + nimbus-jose-jwt）
- **测试**：AuthControllerTest（MockMvc 验证 200/401）

### Step 1：添加 AOP 依赖
**文件：** `start/pom.xml`
- 加 `spring-boot-starter-aop` 依赖
- 验证：`mvn compile` 通过

### Step 2：UserContext（ThreadLocal）
**文件（新建）：** `email-domain/src/main/java/.../auth/context/UserContext.java`
- UserInfo record（userId/userName/roles/permissions/token）
- ThreadLocal 持有 UserInfo
- set/get/getOrNull/clear/便捷静态方法
- 无外部依赖，纯 Java

### Step 3：拦截器 + WebConfig
**文件（新建）：**
- `email-adapter/src/main/java/.../auth/interceptor/UserContextInterceptor.java`
- `email-adapter/src/main/java/.../auth/config/WebConfig.java`
- 拦截所有路径（排除 /health /actuator）
- preHandle 提取 X-User-* 头 → UserContext.set()
- afterCompletion → UserContext.clear()
- **注意**：email-parent 有 `/api` 前缀（`spring.mvc.servlet.path=/api`），拦截器 addPathPatterns("/**")

### Step 4：权限注解 + AOP 切面
**文件（新建）：**
- `email-domain/src/main/java/.../auth/annotation/RequirePermission.java`
- `email-app/src/main/java/.../auth/aspect/PermissionAspect.java`
- @Around 环绕通知 + UserContext.getOrNull() + 权限检查 + AccessDeniedException

### Step 5：RestClient 拦截器 + @HttpExchange
**文件（新建）：**
- `email-domain/src/main/java/.../auth/http/AuthClientHttpRequestInterceptor.java`
- `email-domain/src/main/java/.../auth/http/exchange/OrderClient.java`（示例）
- `email-app/src/main/java/.../auth/config/HttpClientConfig.java`
- RestClient.builder().requestInterceptor(new AuthClientHttpRequestInterceptor())
- HttpServiceProxyFactory 生成代理 Bean

### Step 6：测试 + 示例 Controller
**测试（新建）：**
- `UserContextTest.java`：ThreadLocal set/get/clear
- `PermissionAspectTest.java`：权限通过/拒绝
- `UserContextInterceptorTest.java`：请求头提取
- `OrderClientTest.java`：MockServer 模拟下游
- **覆盖目标：100% 行覆盖率**（JaCoCo）

**示例 Controller（新建）：**
- `email-adapter/src/main/java/.../auth/controller/AuthDemoController.java`
- GET /api/auth/me（读取 UserContext）
- GET /api/auth/demo（@RequirePermission 演示）

---

## 三、文件清单（17 个新文件）

```
auth-service/                          ← 独立项目（COLA 架构）
├── auth-domain/src/main/java/.../jwt/
│   └── JwtValidator.java              ← JWT 验证逻辑
├── auth-adapter/src/main/java/.../auth/
│   └── AuthController.java            ← /auth/validate
├── auth-infrastructure/src/main/java/.../config/
│   └── JwtDecoderConfig.java          ← JWT 解码器配置
├── auth-app/src/main/java/.../
│   └── AuthApplication.java           ← 启动类
├── start/pom.xml
└── auth-adapter/src/test/.../
    └── AuthControllerTest.java        ← 200/401 测试

email-parent/                          ← 业务服务（现有项目）
├── email-domain/src/main/java/.../auth/
│   ├── context/UserContext.java       ← ThreadLocal
│   ├── annotation/RequirePermission.java ← 权限注解
│   └── http/
│       ├── AuthClientHttpRequestInterceptor.java ← RestClient 拦截器
│       └── exchange/OrderClient.java  ← @HttpExchange 示例
├── email-adapter/src/main/java/.../auth/
│   ├── interceptor/UserContextInterceptor.java  ← 请求拦截
│   ├── config/WebConfig.java          ← 拦截器注册
│   └── controller/AuthDemoController.java ← 示例
├── email-app/src/main/java/.../auth/
│   ├── aspect/PermissionAspect.java   ← AOP 切面
│   └── config/HttpClientConfig.java   ← 代理 Bean
├── start/pom.xml                      ← 加 aop 依赖
└── email-app/src/test/.../auth/
    ├── UserContextTest.java
    ├── PermissionAspectTest.java
    ├── UserContextInterceptorTest.java
    └── OrderClientTest.java
```

---

## 四、风险点

| 风险 | 应对 |
|---|---|
| ThreadLocal 内存泄漏 | afterCompletion 强制 clear() |
| AOP 代理失效（同类调用） | AOP 代理模式 + public 方法 |
| @HttpExchange 和现有 RestClient 冲突 | 独立 Bean 名称（authenticatedRestClient） |
| 测试 mock UserContext | 用 UserContext.set() 手动设置 |

---

## 五、预期产出

- ✅ 13 个新文件（代码 + 测试）
- ✅ 100% 覆盖率（JaCoCo）
- ✅ 0 编译警告
- ✅ 全链路可运行：Ingress 透传 → 拦截器 → UserContext → AOP → 下游透传

---

## 六、执行顺序

```
Step 1 (5min) → Step 2 (10min) → Step 3 (10min)
→ Step 4 (15min) → Step 5 (15min) → Step 6 (20min)
≈ 75 分钟总工时
```

---

## 七、确认点

1. **email-parent 里做还是新项目？** → 建议 email-parent（COLA 架构现成）
2. **OrderClient 是示例还是真实服务？** → 示例如需求文档要求
3. **Spring Security 要不要引入？** → 需求说"可以用 AccessDeniedException"，当前方案用自定义异常也行
4. **前端怎么测试？** → 手动 curl 模拟 Ingress 头，或搭简单 auth-url mock

**计划文件位置：** `~/.hermes/plans/2026-08-28_springboot-security-impl.md`
