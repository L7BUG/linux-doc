# ECC 插件 Java 开发完全指南

## 1. 概述

ECC（Everything Claude Code）是一个 Claude Code 插件体系，提供 **agents（委托代理）、skills（技能工作流）、hooks（钩子）、rules（代码规则）** 四层能力。对 Java 开发者来说，它内置了完整的 Spring Boot / Quarkus / JPA / Kotlin 支持，可覆盖"编码 → 构建 → 审查 → 测试 → 安全加固"全流程。

> 你的环境已完成安装（插件版本 2.0.0）。源码仓库：[github.com/affaan-m/ECC](https://github.com/affaan-m/ECC)

## 2. Java 相关资源总览

### 2.1 专属 Agents（2 个）

| Agent | 用途 | 何时触发 |
|-------|------|---------|
| `java-reviewer` | Spring Boot / Quarkus 代码审查专家 | 审查 Java 改动时 |
| `java-build-resolver` | Maven/Gradle 构建、编译、依赖错误修复 | 构建失败时 |

两个 agent 都会**自动检测框架**：读取 `pom.xml` / `build.gradle(.kts)`，含 `spring-boot` → 应用 SPRING 规则，含 `quarkus` → 应用 QUARKUS 规则，都没有 → 仅用通用 Java 规则。

### 2.2 技能 Skills（15 个）

| 技能 | 用途 |
|------|------|
| `java-coding-standards` | Java 17+ 编码规范：命名、不可变性、Optional、streams、异常、泛型、CDI、项目布局 |
| `springboot-patterns` | Spring Boot 架构模式：分层、REST API、JPA、缓存、异步、日志 |
| `springboot-security` | Spring Boot 安全加固 |
| `springboot-tdd` | Spring Boot 测试驱动开发流程 |
| `springboot-verification` | Spring Boot 验证循环 |
| `quarkus-patterns` / `quarkus-security` / `quarkus-tdd` / `quarkus-verification` | Quarkus 对应四件套 |
| `jpa-patterns` | JPA 数据访问模式 |
| `kotlin-patterns` 等 5 个 | Kotlin / Coroutines / Exposed / Ktor / 测试（KMP 与 Android 开发） |

### 2.3 规则 Rules（`rules/java/`）

| 规则文件 | 内容 |
|---------|------|
| `coding-style.md` | Java 编码风格 |
| `patterns.md` | Repository、Service Layer 等架构模式（自动应用于 `**/*.java`） |
| `security.md` | Java 安全规范 |
| `testing.md` | 测试规范 |
| `hooks.md` | Java 项目钩子建议 |

### 2.4 命令 Commands

| 命令 | 说明 |
|------|------|
| `/code-review` | 通用代码审查（本地未提交改动或 PR 模式），Java 项目会自动调用 java-reviewer |
| `/build-fix` | 通用构建修复，自动派发 java-build-resolver |
| `/gradle-build` | Gradle 构建修复（Android / KMP 项目） |
| `/kotlin-build` / `/kotlin-review` / `/kotlin-test` | Kotlin 构建 / 审查 / TDD |
| `/plan` `/feature-dev` `/pr` `/review-pr` `/harness-audit` 等 | 通用工作流命令 |

## 3. 日常使用工作流

### 3.1 编码时（自动生效）

在 Java 项目中写代码时，`java-coding-standards` 和对应框架技能会被自动激活：

- 命名：类 `PascalCase`、方法 `camelCase`、常量 `UPPER_SNAKE_CASE`
- 不可变优先，`Optional` 正确使用（禁 `.get()`，用 `.orElseThrow()`）
- 构造器注入而非字段 `@Autowired`
- Spring 用 `@RestControllerAdvice`、Quarkus 用 `ExceptionMapper` 集中处理异常

### 3.2 构建失败时

```
遇到 Maven/Gradle 构建错误时，直接说"修复这个构建错误"
```

会调用 `java-build-resolver`：检测框架 → 分组错误（配置错误优先）→ 最小化修复 → 重新构建验证。修改超过 3 次仍失败或需要新增依赖时会停下来询问你。

### 3.3 代码审查时

```bash
# 审查本地未提交改动
/code-review

# 审查 GitHub PR（PR 号码或 URL）
/code-review 42
/code-review https://github.com/owner/repo/pull/42
```

`java-reviewer` 会按 **CRITICAL → HIGH → MEDIUM** 分级报告，重点关注：

- **安全（CRITICAL）**：SQL 注入（拼接查询）、硬编码密钥、路径穿越、`@RequestBody` 无 `@Valid`、日志泄露 token
- **错误处理（CRITICAL）**：吞异常的空 catch、Optional `.get()`、无集中异常处理、HTTP 状态码错误
- **架构（HIGH）**：字段注入、`findById().get()`、N+1 查询

发现 CRITICAL 安全问题会直接**升级到 `security-reviewer`**。

### 3.4 测试驱动开发（TDD）

```
/springboot-tdd   # Spring Boot 项目
/quarkus-tdd      # Quarkus 项目
```

强制先写测试（80%+ 覆盖率目标）再实现，用 `./mvnw verify` 或 `./gradlew check` 验证。

### 3.5 安全加固

```
/springboot-security   # Spring Boot 安全审查与加固
/quarkus-security      # Quarkus 安全审查与加固
```

### 3.6 全自动功能开发

```bash
/feature-dev          # 基于代码库理解的引导式功能开发
/orch-add-feature     # 端到端：研究→规划→TDD→审查→门控提交
/orch-fix-defect      # 复现 bug → 回归测试 → 修复 → 审查
/orch-refine-code     # 保持行为不变的重构
```

## 4. 框架自动检测机制

ECC 的 Java 工具链核心设计是**从构建文件自动识别框架**，无需手动配置：

```bash
cat pom.xml 2>/dev/null || cat build.gradle 2>/dev/null || cat build.gradle.kts 2>/dev/null
```

| 构建文件内容 | 应用的规则集 |
|-------------|------------|
| 含 `spring-boot` | SPRING 规则（`@RestControllerAdvice`、`@Valid`、`@Query` 绑定参数等） |
| 含 `quarkus` | QUARKUS 规则（`*Resource` 命名、CDI 作用域、Panache 实体、`quarkus-csrf-reactive` 等） |
| 两者都有 | 标记为问题并同时应用两套规则 |
| 都没有 | 仅通用 Java 规则 |

## 5. 常见问题

**Q1：ECC 会修改我的代码吗？**
`java-reviewer` 只报告问题不重写代码；`java-build-resolver` 只做最小化修复构建错误。

**Q2：Java 项目没有独立的 `/java-build` 命令？**
对，ECC 只有 Kotlin 系列独立命令（`/kotlin-build` 等）。Java 项目使用通用 `/build-fix` 和 `/code-review`，它们会自动派发到 Java 专属 agent。

**Q3：rules 目录需要手动安装吗？**
是的。插件路径下 rules 不会自动分发，需要手动拷贝到 `~/.claude/rules/ecc/`（例如 `cp -R rules/java ~/.claude/rules/ecc/`），否则 `**/*.java` 规则不会生效。

**Q4：如何检查 ECC 安装健康度？**
运行 `/harness-audit` 获得就绪度评分。
