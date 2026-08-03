# ECC Plan 命令完全指南

## 1. 概述

ECC（Everything Claude Code）提供了一整个 **plan（规划）相关命令家族**，分两条工作流：**轻量流程**（`/plan-prd` → `/plan`）和 **PRP 深度流程**（`/prp-prd` → `/prp-plan` → `/prp-implement` → `/prp-commit`），外加桥接命令 `/plan-orchestrate`。

> 前提：ECC 插件已安装（2.0.0），命令带 `ecc:` 前缀使用，如 `/ecc:plan`。

## 2. 命令家族总览

| 命令 | 阶段 | 输入 | 产物 | 一句话定位 |
|---|---|---|---|---|
| `/plan-prd` | 需求 | 功能想法（可空） | `.claude/prds/{名}.prd.md` | 生成**精简 PRD**，只写"什么/为什么"，不写"怎么做" |
| `/plan` | 规划 | 需求文本 / PRD 路径 | inline 计划 或 `.claude/plans/{名}.plan.md` | 重述需求 → 识别风险 → 分阶段计划 → **等确认** |
| `/prp-prd` | 需求 | 功能想法（可空） | PRD | 交互式、假设驱动的**深度 PRD**，多轮问答 |
| `/prp-plan` | 规划 | 功能描述 / PRD 路径 | `.claude/PRPs/plans/...` | 自包含实施计划，**一次性捕获所有代码库模式** |
| `/prp-implement` | 执行 | 计划文件路径 | 代码 | 逐步执行 + **每步立即验证**，失败先修再继续 |
| `/prp-commit` / `/prp-pr` | 收尾 | 自然语言 | commit / PR | 提交和开 PR |
| `/plan-orchestrate` | 桥接 | 计划文档路径 | 命令列表 | 把计划拆成 `/orchestrate` agent 链，**纯生成不执行** |

## 3. 两条工作流怎么选

```
┌── 轻量流程（日常快速开发）─────────────────────────────┐
│  /plan-prd "功能想法"   →   /plan 路径/xx.prd.md        │
│     生成 PRD                读取 PRD，规划并等确认        │
└────────────────────────────────────────────────────────┘
┌── PRP 深度流程（复杂/多模块功能）──────────────────────┐
│  /prp-prd → /prp-plan → /prp-implement → /prp-commit   │
│  问答式PRD   捕获全部模式   逐步执行+验证     自然语言提交 │
└────────────────────────────────────────────────────────┘
```

**关键区别**：`/plan` 的计划是给人看、等确认后由当前会话执行的；`/prp-plan` 的计划是"自包含执行手册"——要求所有信息（代码模式、约定、坑）都写进计划里，让实现者（可能是另一个会话/agent）**不再需要搜索代码库就能一次跑完**。

## 4. 命令详解

### 4.1 `/plan` — 通用计划命令

```text
/plan 需求文本                     # 对话模式：inline 计划
/plan 路径/xx.prd.md               # PRD 模式：写 .claude/plans/xx.plan.md
/plan                              # 空输入：反问你要规划什么
```

特点：

- **必须等确认才动代码**（"yes/proceed" 或 "modify: ..."）
- PRD 模式下自动找下一个 pending 的 Delivery Milestone，并把 PRD 的状态行更新为 in-progress
- 计划前会做 **Pattern Grounding**：搜索仓库里的命名/错误处理/测试等约定，让实现"镜像现有模式而非重新发明"
- 规划完成后可接 `/code-review`、`/pr`、`/build-fix`

### 4.2 `/plan-prd` — 精简 PRD 生成器

```text
/plan-prd                          # 空：从"你想构建什么？"开始提问
/plan-prd "多租户数据隔离功能"      # 直接给想法
```

特点：

- **问题优先**：先问 4 个框架问题——谁有这个问题、可观察的痛点是什么、为什么现有方案解决不了、**为什么是现在**
- **四阶段门控**：FRAME（框架提问）→ GROUND（要证据）→ RESEARCH（研究）→ 生成，每阶段等你的回答才继续
- **Anti-fluff 铁律**：信息缺失写 `TBD — needs validation via {方法}`，禁止编造需求
- **边界清晰**：只写需求和成功标准，一旦写出实现细节就切掉——那是 `/plan` 的事

### 4.3 `/prp-prd` — 交互式深度 PRD

```text
/prp-prd "AI 简历筛选工具"          # 或空着直接开始
```

与 `/plan-prd` 的区别：**假设驱动 + 三轮问答**（`QUESTION SET 1 → GROUNDING → SET 2 → RESEARCH → SET 3 → GENERATE`），把你当"产品经理访谈对象"，每个问题集都建立在前一轮答案上，Grounding 阶段验证假设。适合功能边界模糊、需要想清楚的产品型需求。

### 4.4 `/prp-plan` — 自包含实施计划

```text
/prp-plan "给订单服务加消息队列"       # 自由文本
/prp-plan 路径/xx.prd.md               # 读 PRD 选下一个 pending 阶段
```

核心哲学（Golden Rule）：

> **如果你在实施时需要搜索代码库，现在就把它捕获到计划里。**

计划必须包含：所有代码模式、约定、陷阱、文件路径、验证命令。它输出给"一次跑完不提问"的实现者（比如丢给 `/prp-implement` 或另一个 agent 会话）。

### 4.5 `/prp-implement` — 计划执行器

```text
/prp-implement .claude/PRPs/plans/xx.plan.md
```

特点：

- **验证循环**：每个变更后立即跑检查（typecheck/lint/test/build），失败先修复再继续，**绝不累积破坏状态**
- 自动检测包管理器（npm/pnpm/yarn/bun/cargo/go/uv…）来选验证命令
- 执行完可接 `/prp-commit`（自然语言描述提交什么）和 `/prp-pr`（开 PR）

### 4.6 `/plan-orchestrate` — 计划 → agent 链桥接

```text
/plan-orchestrate 路径/计划.md --lang=java --scope=all
/plan-orchestrate 路径/计划.md --lang=java --scope=step:3   # 只生成第 3 步
/plan-orchestrate 路径/计划.md --dry-run                     # 只看拆解不生成命令
```

特点：

- 把计划文档的每个步骤打上标签（impl/security/db/docs…），按标签表**自动组 agent 链**（如 `impl+db` → `tdd-guide,database-reviewer,java-reviewer`）
- 任务描述压缩成 200–600 字、带验收标准、开头带 `[Plan: 路径#step-N]` 定位符
- **纯生成**：只输出可粘贴的命令列表，绝不自己执行 `/orchestrate`

## 5. 选择指南（对号入座）

| 你的场景 | 用这个 |
|---|---|
| 小功能，思路清楚，想快速过一遍再动手 | `/plan`（直接给需求文本） |
| 功能有想法但需求模糊，需要先想清楚"为什么做" | `/plan-prd` 然后 `/plan` |
| 复杂产品型需求，边界不清，愿意被深度访谈 | `/prp-prd` |
| 复杂功能，需要给"一次跑完"的执行手册 | `/prp-plan` → `/prp-implement` |
| 已有计划文档，想拆成 agent 自动执行 | `/plan-orchestrate` |
| 只想要个快速计划，多模型并行产出方案 | `/multi-plan`（补充：多模型实施计划） |

## 6. 实践建议

1. **日常开发默认走轻量流程**：`/plan-prd` 只在需求真模糊时用，思路清楚直接 `/plan 需求文本`，更快
2. **PRP 流程是给"交接"场景的**：你自己写代码时不需要自包含计划；要交给另一个会话/agent 执行时才值得用 `/prp-plan` 的深度捕获
3. **计划里的 Pattern Grounding 是精华**：`/plan` 会自动扫描仓库现有代码约定，新代码镜像旧代码，比从零写省很多事
4. **`/plan-orchestrate` 和 `/plan` 是互补的**：`/plan` 产出给人看的计划 → 你觉得步骤清晰了 → `/plan-orchestrate` 把它变成 agent 链条自动跑

## 7. 常见问题

**Q1：`/plan` 和 `/prp-plan` 有什么本质区别？**
`/plan` 产出给人看、当前会话执行的计划，特点是 Pattern Grounding 和等待确认；`/prp-plan` 产出"自包含执行手册"，把所有代码库知识一次性捕获，让实现者无需再搜索即可一次跑完，通常配合 `/prp-implement` 执行。

**Q2：`/plan-prd` 和 `/prp-prd` 都是生成 PRD，用哪个？**
需求思路大致清楚用 `/plan-prd`（快、四阶段门控）；产品型复杂需求、边界模糊、需要深入访谈用 `/prp-prd`（三轮问答、假设驱动）。两者产物都是 PRD，区别在提问深度。

**Q3：`/plan-orchestrate` 会自己执行任务吗？**
不会。它是纯生成命令，只输出一条条可粘贴的 `/orchestrate custom "agents" "task"` 命令，由你自己决定何时粘贴执行。

**Q4：计划文件都存在哪里？**
- `/plan` PRD 模式 → `.claude/plans/{名}.plan.md`
- `/plan-prd` → `.claude/prds/{名}.prd.md`
- PRP 系列 → `.claude/PRPs/{prds,plans,reports,reviews}/`（旧格式，不迁移）
