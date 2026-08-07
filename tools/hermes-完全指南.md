# Hermes 完全指南

> **背景**：Hermes Agent 是一套开源 AI 智能体框架，运行在你的终端、桌面应用和消息平台（如 Telegram）上，可以自主调用工具、执行任务、跨会话学习。本文从"它是什么"到"怎么用"完整介绍，涵盖安装、配置、日常使用与维护。

---

## 1. Hermes 是什么

Hermes Agent 是由 **Nous Research** 开发的开源 AI Agent 框架，与 Claude Code（Anthropic）、Codex（OpenAI）同类——都是通过**工具调用**（tool calling）在真实系统上执行任务的自主智能体。

一句话：**把 LLM 从"聊天机器人"升级成"能动手干活的助手"**——它能读写文件、执行命令、操作浏览器、调用 API、定时跑任务，并把经验沉淀为技能跨会话复用。

### 1.1 和普通 ChatGPT 的区别

| | ChatGPT 网页版 | Hermes Agent |
|---|---|---|
| 能力 | 对话、生成文本 | 对话 + **执行命令 + 操作文件 + 调用工具** |
| 记忆 | 单会话 | **跨会话记忆 + 技能学习** |
| 自主性 | 等用户输入 | 可定时任务、后台运行、自主多步执行 |
| 接入方式 | 网页 | CLI / TUI / 桌面应用 / Telegram 等 20+ 平台 |

### 1.2 核心特性

- **自我改进（Skills）**：把可复用的流程沉淀为技能（SKILL.md），未来会话自动加载，越用越强
- **跨会话记忆（Memory）**：记住用户偏好、环境信息、工具怪癖
- **多平台网关（Gateway）**：同一个 agent 核心同时服务 Telegram、Discord、Slack、WhatsApp 等消息平台，全工具访问
- **模型无关（Provider-agnostic）**：支持 OpenRouter、Anthropic、OpenAI、DeepSeek、Google、本地模型等 20+ 供应商，可随时切换
- **多界面（Surfaces）**：CLI、Ink TUI、桌面应用（Electron）、Web Dashboard、IDE 插件（ACP）
- **多 Profile**：多个独立实例，配置/会话/技能/记忆互相隔离

## 2. 架构概览

```
Telegram / CLI / TUI / 桌面应用 / IDE
              ↓
         Gateway（消息网关）
              ↓
        Agent 核心（对话循环 + 工具调用）
        ├── 工具：终端、文件、浏览器、网页搜索...
        ├── 技能库（~/.hermes/skills/）
        ├── 记忆（~/.hermes/memories/）
        └── 会话库（~/.hermes/state.db）
              ↓
    真实系统：文件、进程、网络、API、定时任务
```

### 2.1 关键目录（默认 `~/.hermes/`）

| 路径 | 内容 |
|------|------|
| `config.yaml` | 主配置（模型、工具集、显示等，不含密钥） |
| `.env` | **API 密钥**（仅密钥，绝不进配置文件） |
| `skills/` | 技能库（按分类存放 SKILL.md） |
| `memories/` | 跨会话记忆（MEMORY.md / USER.md） |
| `state.db` | 会话数据库（SQLite + 全文检索） |
| `cron/` | 定时任务 |
| `plugins/` | 插件 |

## 3. 安装

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

安装完成后：

```bash
hermes setup        # 交互式配置向导（选模型、配供应商）
hermes doctor       # 健康检查（依赖、配置）
hermes              # 进入交互式聊天（CLI）
```

> **注意**：密钥放在 `~/.hermes/.env`，配置放 `config.yaml`——两者分离，密钥永不进配置文件。

## 4. 模型配置

### 4.1 选择模型

```bash
hermes model        # 交互式选择默认模型和供应商
hermes chat -q "你好"   # 单次查询
```

### 4.2 会话内切换

```bash
/model sonnet       # 切换模型（会话级）
/model gpt --global # 持久化为默认
```

内置模型别名：`sonnet`、`opus`、`gpt`、`gemini`、`deepseek`、`grok`、`mimo`、`qwen`、`kimi` 等。

### 4.3 故障转移（Fallback）

主模型失败/限流时自动切换备用：

```bash
hermes fallback add <provider>/<model>
hermes fallback list
```

### 4.4 视觉模型

主模型不支持视觉时（如纯文本模型），图片会自动交给辅助视觉模型转成文字描述再进入对话——无需额外配置，自动降级。

## 5. 使用方式

### 5.1 CLI 模式

```bash
hermes                      # 交互式聊天
hermes chat -q "写个脚本"    # 单次问答（脚本友好）
hermes --continue           # 恢复最近会话
hermes --resume <会话ID>     # 精准恢复指定会话
```

### 5.2 TUI 模式（推荐常驻）

```bash
hermes --tui
```

适合在 tmux 里常驻，或配合 alias：

```bash
alias h='tmux attach -t h 2>/dev/null || tmux new -s h "hermes --tui"'
```

### 5.3 Telegram 等消息平台

通过 Gateway 接入后，直接在 Telegram 里指挥 agent：

```bash
hermes gateway setup                 # 配置消息平台
hermes gateway install --system      # 安装为 systemd 服务（开机自启）
hermes gateway status                # 查看状态
```

> **注意**：gateway 与 TUI 都常驻时，避免安装两份 systemd 服务（用户级 + 系统级会冲突，保留一份即可）。

## 6. 常用斜杠命令

| 命令 | 功能 |
|------|------|
| `/new` | 开新会话 |
| `/model <名字>` | 切换模型 |
| `/title <名字>` | 修改会话名 |
| `/status` | 会话、模型、token、上下文信息 |
| `/usage` | token 用量和速率限制 |
| `/compress` | 压缩上下文（长对话保速） |
| `/resume` | 恢复指定会话 |
| `/sessions` | 浏览历史会话 |
| `/cron` | 管理定时任务 |
| `/skills` | 搜索/管理技能 |
| `/curator` | 技能维护（清理、归档、合并） |
| `/memory` | 查看待写入的记忆 |
| `/update` | 更新 Hermes |
| `/help` | 全部命令列表 |

## 7. 核心概念

### 7.1 会话（Session）

- 每次对话是一个会话，存于 `state.db`
- 会话可命名（`/title`）、恢复（`/resume`）、压缩（`/compress`）
- **消息平台和 CLI/TUI 的会话互相隔离**（Telegram 会话 ≠ TUI 会话）

### 7.2 技能（Skills）

- 存储在 `~/.hermes/skills/`，每个技能是带元数据的 SKILL.md
- 新会话自动扫描技能索引，任务匹配时加载完整内容
- **技能索引是会话开始时的快照**：常驻会话不会自动看到新技能，可用 `/reload-skills` 手动刷新
- 后台 Curator 会跟踪技能使用、清理闲置技能、可选合并重叠技能

### 7.3 记忆（Memory）

- 每轮对话注入（小而精，注意预算）
- 存用户偏好、环境事实、工具怪癖
- 与技能的分工：**记忆存"是什么"（事实），技能存"怎么做"（流程）**

### 7.4 定时任务（Cron）

```bash
hermes cron list          # 查看任务
hermes cron add "0 9 * * *" "每日九点总结"   # 添加任务
```

支持持续时间（`30m`）、短语（`every monday 9am`）、5 段 cron 表达式、ISO 时间戳。

### 7.5 子代理委派（Delegation）

复杂任务可以拆给子代理并行执行，每个子代理有独立上下文和终端会话，结果汇总返回。

## 8. 维护

### 8.1 备份与恢复

```bash
hermes backup -o ~/hermes-backup.zip    # 完整备份（配置/技能/会话/记忆/密钥）
hermes backup -q                        # 快速备份（仅关键状态）
hermes import ~/hermes-backup.zip       # 恢复
```

> 备份自动排除 hermes-agent 源码和可再生缓存目录，并对 SQLite 数据库做一致性快照。高压缩归档可用 `tar | zstd -19 --ultra`。

### 8.2 更新

```bash
hermes update           # 从 git 拉取最新代码并重装依赖
hermes update --check   # 检查是否有更新
```

> 更新会重启 gateway，消息平台的会话会短暂中断几秒。

### 8.3 查看状态

```bash
hermes status           # 各组件状态
hermes doctor           # 配置与依赖健康检查
hermes sessions list    # 会话列表
```

## 9. 常见问题

### Q: Hermes 和 Claude Code 有什么区别？

**定位相同、生态不同。** 两者都是工具调用型 agent，但 Hermes 是模型无关的（Claude Code 绑定 Anthropic 模型），且 Hermes 更强调多平台网关、跨会话记忆与技能学习。

### Q: 换模型会丢失会话吗？

**不会。** 会话历史存在本地 `state.db`，切换模型只改变请求发给谁；首轮因缓存重建会稍慢，几轮后恢复。

### Q: 重启服务器后 Hermes 会自动启动吗？

**取决于配置。** 建议用 `hermes gateway install --system` 把网关装成 systemd 服务（开机自启 + 崩溃自愈）；TUI 可放 tmux 会话配合 alias 手动恢复。

### Q: 技能和记忆有什么区别？

**记忆是每轮注入的高信号事实（小而精）；技能是按需加载的完整流程（可长可详细）。** 大段过程性知识放技能，高频偏好放记忆。

### Q: 消息平台发文件有大小限制吗？

**Telegram Bot API 单文件上限 50MB。** 超大文件需先分卷（`split -b 45M`）再发送，接收方合并（`cat`）。

## 参考

- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs/)
- [Hermes Agent GitHub 仓库](https://github.com/NousResearch/hermes-agent)
- [Hermes 跨机器迁移指南](hermes-迁移指南.md)
