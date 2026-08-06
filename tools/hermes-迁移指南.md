# Hermes 跨机器迁移指南

> **背景**：需要把 Hermes（配置、技能、会话、记忆）从一台机器搬到另一台机器时，不需要手动拷贝散落的文件。Hermes 内置了官方的一键备份/恢复命令，本指南覆盖完整流程、传输方式与验证步骤。

---

## 1. 迁移前了解备份范围

`hermes backup` 会打包整个 Hermes 主目录（`~/.hermes`），包含：

| 内容 | 说明 |
|------|------|
| `config.yaml` | 主配置（模型路由、工具集、显示等） |
| `.env` / `auth.json` | **API 密钥与 OAuth 凭据** |
| `state.db` + `sessions/` | 全部历史会话（SQLite + 转录） |
| `skills/` | 已安装技能（含自定义/学习到的技能） |
| `memories/` | 跨会话记忆 |
| `cron/` | 定时任务 |
| `hooks/`、`plugins/` | 钩子脚本与插件 |

**自动排除**：`hermes-agent` 源码目录（新机器重新安装代码即可，无需打包搬运）。

> **注意**：备份 zip 内含 API 密钥，传输务必走加密通道（SSH/SCP），不要放到公开网盘。

## 2. 旧机器：创建备份

### 2.1 完整备份（推荐）

```bash
hermes backup -o ~/hermes-backup.zip
```

### 2.2 快速备份

只需关键状态文件（配置、会话库、密钥、cron）时：

```bash
hermes backup -q -l "迁移前快照"
```

| 参数 | 说明 |
|------|------|
| `-o /path/to/输出.zip` | 指定输出路径（默认 `~/hermes-backup-<时间戳>.zip`） |
| `-q, --quick` | 快速快照：仅打包 config、state.db、.env、auth、cron |
| `-l <标签>` | 快照标签（仅配合 `--quick` 使用） |

> **提示**：备份时 gateway 正在运行也没关系，官方命令支持在线备份；追求一致性可在低峰期操作。

## 3. 传输到新机器

```bash
# 非标准 SSH 端口需要显式指定端口
scp -P <端口> ~/hermes-backup.zip <新机器用户>@<新机器IP>:~/
```

## 4. 新机器：安装并恢复

### 4.1 安装 Hermes

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

### 4.2 恢复备份

```bash
hermes import ~/hermes-backup.zip
```

目标已有配置想直接覆盖时：

```bash
hermes import --force ~/hermes-backup.zip
```

## 5. 验证迁移结果

```bash
hermes doctor      # 检查依赖与配置完整性
hermes status      # 各组件状态
hermes sessions    # 确认历史会话都在
hermes memory      # 确认记忆已恢复
```

## 6. 恢复 Telegram 网关

import 完成后，在新机器上启动 gateway，Telegram 消息才会路由到新机器：

```bash
# tmux 常驻方案
tmux new-session -d -s h 'hermes --tui'
# 或后台 gateway
hermes gateway start
```

## 7. 多机器场景：技能单独同步

若多台机器**同时运行**而非搬家，可用官方技能同步（Skill Sync）：

```bash
hermes sync status            # 查看同步状态
hermes sync enable <技能名>   # 将某技能加入跨设备同步
hermes sync now               # 立即 pull + push
```

`hermes backup` 负责整机迁移，`hermes sync` 负责技能持续同步，两者配合使用。

## 8. 常见问题

### Q: 备份文件有多大？

取决于会话与技能规模。`~/.hermes` 目录中 `hermes-agent` 源码占大头，backup 已自动排除；实际备份通常只有几十到几百 MB。

### Q: 迁移后 API 密钥需要重新配置吗？

**不需要。** `.env` 和 `auth.json` 都在备份范围内，import 后凭据原样恢复。

### Q: 迁移后 Telegram 收不到消息？

检查 gateway 是否已在新机器启动（见第 6 节），并确认 `hermes status` 中 gateway 状态正常。

### Q: 新机器上 import 前需要先跑 setup 向导吗？

**不需要。** 直接安装后 import 即可，备份中的配置会覆盖默认配置。

### Q: 只想要会话记录，不要凭据，怎么办？

手动只拷贝 `state.db` 和 `sessions/` 目录，或先备份后在新机器上选择性恢复。

## 参考

- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs/)
- [Hermes Agent GitHub 仓库](https://github.com/NousResearch/hermes-agent)
