# Spring Boot 微服务安全架构实现文档

> 🎯 Nginx Ingress auth-url 认证 + ThreadLocal 用户上下文 + AOP 权限校验 + @HttpExchange 服务间调用
> 📝 Spring Boot 3.3.x + Java 17+ | 无 OpenFeign 依赖

---

## 架构总览

```
用户请求 → Nginx Ingress → auth-url（JWT 校验）→ 返回 X-User-* 头
                                                          ↓
Spring Boot 服务 → 拦截器提取 X-User-* → UserContext(ThreadLocal)
                                                ↓
                                    Controller / AOP 权限校验
                                                ↓
                                    @HttpExchange → 下游服务（自动透传 Token）
```

---

## 1. pom.xml 关键依赖

```xml
<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.3.5</spring-boot.version>
</properties>

<dependencies>
    <!-- Spring Boot Web（含 RestClient + @HttpExchange） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring AOP（方法级权限校验） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>

    <!-- Lombok（可选，减少样板代码） -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>

<!-- ⚠️ 严禁引入以下依赖：
     <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-starter-openfeign</artifactId>
     </dependency>
     Spring Boot 3.2+ 自带 RestClient + @HttpExchange，不需要 OpenFeign。
-->
```

---

## 2. UserContext.java（ThreadLocal 用户上下文）

```java
package com.example.auth.context;

import java.util.Collections;
import java.util.List;

/**
 * 基于 ThreadLocal 的用户上下文。
 * Ingress 透传的 X-User-* 请求头 → 拦截器提取 → 存入此上下文 → 全链路可读。
 *
 * ⚠️ 请求结束后必须调用 clear() 防止内存泄漏（在拦截器 afterCompletion 中）。
 */
public final class UserContext {

    private UserContext() {}

    private static final ThreadLocal<UserInfo> HOLDER = new ThreadLocal<>();

    /** 设置当前用户信息（拦截器 preHandle 中调用） */
    public static void set(UserInfo userInfo) {
        HOLDER.set(userInfo);
    }

    /** 获取当前用户信息（Controller / Service / AOP 中调用） */
    public static UserInfo get() {
        UserInfo info = HOLDER.get();
        if (info == null) {
            throw new IllegalStateException(
                "UserContext 未初始化。请确保 Ingress 透传了 X-User-Id 头，" +
                "且 UserContextInterceptor 已注册。");
        }
        return info;
    }

    /** 安全获取（不抛异常，返回 null） */
    public static UserInfo getOrNull() {
        return HOLDER.get();
    }

    /** 清理（afterCompletion 中必须调用） */
    public static void clear() {
        HOLDER.remove();
    }

    // ===== 便捷静态方法（常用场景） =====

    public static String getUserId() {
        return get().userId();
    }

    public static String getUserName() {
        return get().userName();
    }

    public static List<String> getRoles() {
        return get().roles();
    }

    public static List<String> getPermissions() {
        return get().permissions();
    }

    public static String getToken() {
        return get().token();
    }

    /**
     * 不可变用户信息 record（线程安全，无 setter）。
     */
    public record UserInfo(
        String userId,
        String userName,
        List<String> roles,
        List<String> permissions,
        String token
    ) {
        /** 空值安全的权限检查 */
        public boolean hasPermission(String permission) {
            return permissions != null && permissions.contains(permission);
        }

        /** 空值安全的角色检查 */
        public boolean hasRole(String role) {
            return roles != null && roles.contains(role);
        }
    }
}
```

---

## 3. UserContextInterceptor.java（拦截器提取请求头）

```java
package com.example.auth.interceptor;

import com.example.auth.context.UserContext;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.servlet.HandlerInterceptor;

import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import java.util.stream.Collectors;

/**
 * 拦截所有请求，从 Ingress 透传的请求头中提取用户信息，填充到 UserContext。
 *
 * 请求头约定（Ingress auth-response-headers 透传）：
 *   X-User-Id         → userId
 *   X-User-Name       → userName
 *   X-User-Roles      → roles（逗号分隔）
 *   X-User-Permissions → permissions（逗号分隔）
 *   Authorization     → token（Bearer xxx）
 */
@Component
public class UserContextInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(UserContextInterceptor.class);

    private static final String HEADER_USER_ID = "X-User-Id";
    private static final String HEADER_USER_NAME = "X-User-Name";
    private static final String HEADER_ROLES = "X-User-Roles";
    private static final String HEADER_PERMISSIONS = "X-User-Permissions";
    private static final String HEADER_AUTHORIZATION = "Authorization";

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        // 提取请求头（Ingress 透传的值）
        String userId = request.getHeader(HEADER_USER_ID);
        String userName = request.getHeader(HEADER_USER_NAME);
        String rolesStr = request.getHeader(HEADER_ROLES);
        String permissionsStr = request.getHeader(HEADER_PERMISSIONS);
        String token = request.getHeader(HEADER_AUTHORIZATION);

        // 未携带用户标识 → 拒绝（Ingress 配置有误或未开启 auth-url）
        if (!StringUtils.hasText(userId)) {
            log.warn("请求缺少 {} 头，路径: {}", HEADER_USER_ID, request.getRequestURI());
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false; // 阻止请求继续
        }

        // 解析逗号分隔的列表
        List<String> roles = parseList(rolesStr);
        List<String> permissions = parseList(permissionsStr);

        // 构建不可变 UserInfo，存入 ThreadLocal
        UserContext.UserInfo userInfo = new UserContext.UserInfo(
            userId,
            StringUtils.hasText(userName) ? userName : userId,
            roles,
            permissions,
            token
        );
        UserContext.set(userInfo);

        log.debug("用户上下文已设置: userId={}, roles={}, permissions={}",
            userId, roles, permissions);

        return true; // 放行
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler,
                                Exception ex) {
        // ⚠️ 必须清理，防止 ThreadLocal 内存泄漏
        UserContext.clear();
    }

    /**
     * 解析逗号分隔的字符串为列表，去除空白。
     * "admin,user" → ["admin", "user"]
     */
    private List<String> parseList(String value) {
        if (!StringUtils.hasText(value)) {
            return Collections.emptyList();
        }
        return Arrays.stream(value.split(","))
            .map(String::trim)
            .filter(StringUtils::hasText)
            .collect(Collectors.toList());
    }
}
```

---

## 4. WebConfig.java（注册拦截器）

```java
package com.example.auth.config;

import com.example.auth.interceptor.UserContextInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * 注册 UserContext 拦截器。
 */
@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final UserContextInterceptor userContextInterceptor;

    public WebConfig(UserContextInterceptor userContextInterceptor) {
        this.userContextInterceptor = userContextInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(userContextInterceptor)
            .addPathPatterns("/**")           // 拦截所有路径
            .excludePathPatterns("/health");  // 排除健康检查
    }
}
```

---

## 5. RequirePermission.java（自定义权限注解）

```java
package com.example.auth.annotation;

import java.lang.annotation.*;

/**
 * 方法级权限校验注解。
 *
 * 用法：
 *   @RequirePermission("user:save")
 *   public void saveUser(...) { ... }
 *
 *   @RequirePermission("order:query")
 *   public Order getOrder(...) { ... }
 *
 * AOP 切面会从 UserContext 获取当前用户权限列表，
 * 如果不包含指定权限，抛出 AccessDeniedException。
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequirePermission {

    /** 需要的权限标识（如 "user:save"、"order:query"） */
    String value();
}
```

---

## 6. PermissionAspect.java（AOP 权限校验切面）

```java
package com.example.auth.aspect;

import com.example.auth.annotation.RequirePermission;
import com.example.auth.context.UserContext;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.stereotype.Component;

/**
 * AOP 环绕通知：拦截所有标注了 @RequirePermission 的方法。
 *
 * 执行流程：
 *   1. 从 UserContext 获取当前用户权限列表
 *   2. 判断是否包含注解指定的权限
 *   3. 包含 → 放行；不包含 → 抛出 AccessDeniedException（403）
 */
@Aspect
@Component
public class PermissionAspect {

    private static final Logger log = LoggerFactory.getLogger(PermissionAspect.class);

    /**
     * 拦截所有 @RequirePermission 注解标注的方法。
     * @Around 是最可靠的切入点（可以控制是否放行，且能拿到返回值）。
     */
    @Around("@annotation(requirePermission)")
    public Object checkPermission(ProceedingJoinPoint joinPoint,
                                  RequirePermission requirePermission) throws Throwable {
        String requiredPermission = requirePermission.value();
        String methodName = joinPoint.getSignature().toShortString();

        // 获取当前用户上下文
        UserContext.UserInfo user = UserContext.getOrNull();
        if (user == null) {
            log.warn("权限校验失败：UserContext 为空，方法={}", methodName);
            throw new AccessDeniedException("未登录：无法获取用户上下文");
        }

        // 权限检查
        if (user.hasPermission(requiredPermission)) {
            log.debug("权限校验通过：用户={}, 权限={}, 方法={}",
                user.userId(), requiredPermission, methodName);
            return joinPoint.proceed(); // 放行，执行原方法
        }

        // 权限不足
        log.warn("权限校验失败：用户={}, 需要={}, 拥有={}, 方法={}",
            user.userId(), requiredPermission, user.permissions(), methodName);
        throw new AccessDeniedException(
            String.format("权限不足：需要 [%s]，当前用户 [%s] 拥有 %s",
                requiredPermission, user.userId(), user.permissions()));
    }
}
```

> 💡 如果项目已引入 Spring Security，直接用 `@PreAuthorize` 替代自定义 AOP：
> ```java
> @PreAuthorize("hasPermission('user:save')")
> public void saveUser(...) { ... }
> ```
> 本文档按需求要求实现自定义 AOP 方案。

---

## 7. AuthClientHttpRequestInterceptor.java（RestClient 拦截器，透传身份）

```java
package com.example.auth.http;

import com.example.auth.context.UserContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.util.StringUtils;

import java.io.IOException;
import java.util.StringJoiner;

/**
 * RestClient 拦截器：服务间调用时自动透传用户身份到下游。
 *
 * 发出的请求会带上：
 *   Authorization: Bearer xxx（原始 Token）
 *   X-User-Id: xxx
 *   X-User-Roles: role1,role2
 *   X-User-Permissions: perm1,perm2
 *
 * 下游服务的 UserContextInterceptor 会提取这些头 → 实现身份透传。
 */
public class AuthClientHttpRequestInterceptor implements ClientHttpRequestInterceptor {

    private static final Logger log = LoggerFactory.getLogger(AuthClientHttpRequestInterceptor.class);

    @Override
    public ClientHttpResponse intercept(HttpRequest request,
                                        byte[] body,
                                        ClientHttpRequestExecution execution) throws IOException {
        UserContext.UserInfo user = UserContext.getOrNull();

        if (user != null) {
            // 透传 Token
            if (StringUtils.hasText(user.token())) {
                request.getHeaders().set("Authorization", user.token());
            }

            // 透传用户信息
            request.getHeaders().set("X-User-Id", user.userId());
            if (StringUtils.hasText(user.userName())) {
                request.getHeaders().set("X-User-Name", user.userName());
            }
            request.getHeaders().set("X-User-Roles", joinList(user.roles()));
            request.getHeaders().set("X-User-Permissions", joinList(user.permissions()));

            log.debug("RestClient 拦截器：透传身份到 {} {}，userId={}",
                request.getMethod(), request.getURI(), user.userId());
        } else {
            log.warn("RestClient 拦截器：UserContext 为空，无法透传身份，URI={}",
                request.getURI());
        }

        // 执行实际请求
        return execution.execute(request, body);
    }

    private String joinList(java.util.List<String> list) {
        if (list == null || list.isEmpty()) return "";
        return String.join(",", list);
    }
}
```

---

## 8. OrderClient.java（@HttpExchange 声明式客户端）

```java
package com.example.auth.http.exchange;

import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;

import java.util.Map;

/**
 * 声明式 HTTP 客户端（Spring Boot 3.2+ 官方推荐，替代 OpenFeign）。
 *
 * 底层由 RestClient 实现，通过 HttpServiceProxyFactory 生成代理。
 * 调用时自动携带 UserContext 中的身份信息（拦截器处理）。
 */
@HttpExchange(url = "/api/orders")
public interface OrderClient {

    /**
     * 按 ID 查询订单。
     * GET /api/orders/{id}
     */
    @GetExchange("/{id}")
    Map<String, Object> getOrderById(@RequestParam("id") String id);

    /**
     * 创建订单。
     * POST /api/orders
     */
    @PostExchange
    Map<String, Object> createOrder(@RequestBody Map<String, Object> order);
}
```

---

## 9. HttpClientConfig.java（生成代理 Bean）

```java
package com.example.auth.config;

import com.example.auth.http.AuthClientHttpRequestInterceptor;
import com.example.auth.http.exchange.OrderClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.support.RestClientAdapter;
import org.springframework.web.service.invoker.HttpServiceProxyFactory;

/**
 * 配置 @HttpExchange 代理客户端。
 *
 * 执行流程：
 *   1. RestClient.builder() 创建 RestClient（带拦截器）
 *   2. RestClientAdapter 适配 RestClient → RestOperations
 *   3. HttpServiceProxyFactory 生成接口代理
 *   4. 注入 Bean → Controller 直接使用
 */
@Configuration
public class HttpClientConfig {

    /**
     * 创建带身份透传拦截器的 RestClient。
     */
    @Bean
    public RestClient authenticatedRestClient() {
        return RestClient.builder()
            // 基础 URL（下游服务地址，K8s 内部 DNS 解析）
            .baseUrl("http://order-service:8081")
            // 注册拦截器：自动透传 UserContext 身份
            .requestInterceptor(new AuthClientHttpRequestInterceptor())
            .build();
    }

    /**
     * 生成 OrderClient 代理 Bean。
     * 调用 orderClient.getOrderById("123") 等同于：
     *   GET http://order-service:8081/api/orders/123
     *   + 自动携带 Authorization/X-User-* 头
     */
    @Bean
    public OrderClient orderClient(RestClient authenticatedRestClient) {
        RestClientAdapter adapter = RestClientAdapter.create(authenticatedRestClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory
            .builderFor(adapter)
            .build();
        return factory.createClient(OrderClient.class);
    }
}
```

---

## 10. UserController.java（示例业务）

```java
package com.example.auth.controller;

import com.example.auth.annotation.RequirePermission;
import com.example.auth.context.UserContext;
import com.example.auth.http.exchange.OrderClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/users")
public class UserController {

    private static final Logger log = LoggerFactory.getLogger(UserController.class);

    private final OrderClient orderClient;

    public UserController(OrderClient orderClient) {
        this.orderClient = orderClient;
    }

    /**
     * 查看个人信息（需要登录即可，无额外权限要求）。
     * GET /api/users/me
     */
    @GetMapping("/me")
    public Map<String, Object> getCurrentUser() {
        UserContext.UserInfo user = UserContext.get();
        return Map.of(
            "userId", user.userId(),
            "userName", user.userName(),
            "roles", user.roles(),
            "permissions", user.permissions()
        );
    }

    /**
     * 查询用户订单（需要 order:query 权限）。
     * GET /api/users/{userId}/orders
     */
    @GetMapping("/{userId}/orders")
    @RequirePermission("order:query")
    public Map<String, Object> getUserOrders(@PathVariable String userId) {
        log.info("查询用户订单: userId={}, 操作人={}", userId, UserContext.getUserId());

        // 服务间调用：OrderClient 自动透传身份到下游
        Map<String, Object> order = orderClient.getOrderById("ORD-001");

        return Map.of(
            "userId", userId,
            "orders", order,
            "queriedBy", UserContext.getUserId()
        );
    }

    /**
     * 创建订单（需要 order:create 权限）。
     * POST /api/users/{userId}/orders
     */
    @PostMapping("/{userId}/orders")
    @RequirePermission("order:create")
    public Map<String, Object> createUserOrder(
            @PathVariable String userId,
            @RequestBody Map<String, Object> order) {
        log.info("创建订单: userId={}, 操作人={}", userId, UserContext.getUserId());

        order.put("userId", userId);
        Map<String, Object> result = orderClient.createOrder(order);

        return Map.of(
            "success", true,
            "order", result,
            "createdBy", UserContext.getUserId()
        );
    }
}
```

---

## 11. application.yml 配置

```yaml
server:
  port: 8080

spring:
  application:
    name: user-service

# 日志配置（开发环境可开 DEBUG 看拦截器细节）
logging:
  level:
    com.example.auth: DEBUG
```

---

## 12. 全链路调用示例

```
curl -H "Authorization: Bearer eyJhbG..." https://example.com/api/users/123/orders

  ↓ Nginx Ingress（auth-url 校验 JWT → 透传头）

  → X-User-Id: u001
  → X-User-Roles: admin,user
  → X-User-Permissions: order:query,order:create
  → Authorization: Bearer eyJhbG...

  ↓ UserContextInterceptor（preHandle）

  → UserContext.set(new UserInfo("u001", ..., ["order:query"], ...))

  ↓ UserController.getUserOrders

  → @RequirePermission("order:query") → AOP 校验通过 ✅
  → orderClient.getOrderById("ORD-001")

  ↓ AuthClientHttpRequestInterceptor（自动透传）

  → OrderService: X-User-Id: u001 + Authorization: Bearer ...
  → OrderService 的 UserContextInterceptor 提取 → UserContext.set()
  → OrderService 的 Controller 可以读取 UserContext
```

---

## 13. 关键设计说明

| 设计点 | 方案 | 说明 |
|---|---|---|
| 用户上下文 | ThreadLocal + record | 不可变、线程安全、泛型友好 |
| 拦截器 | HandlerInterceptor | 比 Filter 更适合 MVC 层，可访问 handler 信息 |
| 权限校验 | AOP + 自定义注解 | 无侵入，方法上加一行注解即可 |
| 服务间调用 | @HttpExchange + RestClient | Spring Boot 3.2+ 官方方案，替代 OpenFeign |
| 身份透传 | ClientHttpRequestInterceptor | RestClient 拦截器，自动从 UserContext 取 Token |
| 内存安全 | afterCompletion + clear() | 每个请求结束清理 ThreadLocal |

---

## 14. 与 OpenFeign 的对比

| 特性 | @HttpExchange（本文） | OpenFeign |
|---|---|---|
| 来源 | Spring Boot 3.2+ 官方内置 | Spring Cloud 第三方 |
| 依赖 | 无额外依赖（spring-boot-starter-web） | 需要 spring-cloud-starter-openfeign |
| 底层实现 | RestClient（同步）/ WebClient（异步） | 代理 + HTTP 客户端 |
| 拦截器 | ClientHttpRequestInterceptor | RequestInterceptor |
| 声明式接口 | @HttpExchange | @FeignClient |
| 维护状态 | 官方持续维护 | 仍可用但不再是首选 |

> ⚠️ Spring Boot 4.x 将进一步增强 @HttpExchange，OpenFeign 逐步被官方方案替代。

---

## 15. 常见问题

**Q: UserContext.get() 抛 IllegalStateException？**
→ Ingress 没配置 `auth-url`，或 `auth-response-headers` 没透传 `X-User-Id`。

**Q: @RequirePermission 不生效？**
→ 检查切面类是否被扫描到（`@Component` + 在主包路径下）；AOP 代理模式需要是 public 方法。

**Q: 服务间调用透传失败？**
→ 下游服务也必须有 UserContextInterceptor + WebConfig；@HttpExchange 的 baseUrl 用 K8s Service DNS（如 `http://order-service:8081`）。

**Q: ThreadLocal 内存泄漏？**
→ 确保 afterCompletion 调用了 clear()；线程池复用线程时尤其危险。
