# Linux 技术文档

以 Arch Linux 为基线的运维笔记，涵盖系统备份还原、文件系统、压缩工具等实用主题。

## 文档索引

### Arch Linux 运维

| 文档 | 简介 |
|------|------|
| [全盘还原 Arch Linux](archlinux/全盘还原archlinux.md) | 从 Live ISO（U 盘）环境出发，通过备份文件（`.tar.zst`）完整还原 Arch Linux 系统，含分区、挂载、chroot、引导修复全流程 |
| [KDE 元包详解](archlinux/kde-元包详解.md) | 详解 `plasma-meta`、`kde-applications-meta` 及 12 个子分类元包的结构、内容与选装策略 |
| [Pacman 完全指南](archlinux/pacman-完全指南.md) | Pacman 包管理器完整教程：安装/删除/升级/搜索，仓库管理，AUR 构建与 yay 助手，pacman 钩子，配置文件优化 |
| [KDE 使用技巧与快捷键指南](archlinux/kde-使用技巧与快捷键指南.md) | 窗口管理、KRunner 万能启动器、Dolphin 文件管理器、虚拟桌面与活动、KWin 窗口规则、面板自定义、Spectacle 截图、触控板手势 |

### Btrfs 文件系统

| 文档 | 简介 |
|------|------|
| [写时复制（Copy-on-Write）详解](btrfs/copy-on-write-详解.md) | 从内核 fork()、mmap、Btrfs 文件系统三重视角拆解 CoW 原理，性能权衡，日志 vs CoW 范式对比 |
| [Btrfs RAID0 与子卷实战指南](btrfs/btrfs-raid0-子卷.md) | Btrfs 组建 RAID0 提升性能，子卷布局规划，快照/回滚/还原操作，snapper 定时快照，WinBtrfs 双系统共享数据 |
| [compsize 命令详解](btrfs/btrfs-compsize-详解.md) | 精确计算 Btrfs 透明压缩的实际效果，Disk Usage / Uncompressed / Referenced 字段辨析，验证 nodatacow 是否生效 |
| [Btrfs 状态查看与健康检查](btrfs/btrfs-状态查看.md) | 文件系统用量、设备错误统计、scrub 数据校验、子卷布局、挂载与压缩确认，常用状态命令速查 |

### 工具

| 文档 | 简介 |
|------|------|
| [tar zstd 压缩解压完全指南](tools/tar-zstd-guide.md) | Zstandard 压缩算法入门，tar 配合 zstd 的常用操作，与 gzip/bzip2/xz 的对比选型 |
| [Wine & winetricks 完全指南](tools/wine-winetricks-详解.md) | Wine 兼容层原理与安装，WINEPREFIX 多环境隔离，winetricks 组件管理，字体安装，中文乱码修复 |
| [Python 入门指南](tools/python-入门指南.md) | Python 3 基础语法，文件操作（pathlib/shutil），邮件处理（IMAP/SMTP），游戏自动化脚本（pynput/pyautogui），命令行工具编写 |
| [Rust 入门指南](tools/rust-入门指南.md) | Rust 基础语法，所有权与借用，枚举与模式匹配，Trait 与泛型，错误处理，Cargo 与 crates.io，与 C/C++/Go/Python 对比 |
| [Flatpak 完全指南](tools/flatpak-完全指南.md) | Flatpak 沙盒化应用分发框架，安装配置、Flathub 仓库、权限管理（Flatseal）、主题统一、国内镜像加速、常见问题 |
| [Rust 模拟鼠标输入方案](tools/rust-模拟鼠标输入方案.md) | 从 X11 到 Arduino 硬件的 4 种鼠标模拟方案，反作弊检测层级分析，行为随机化技术，设备切换检测与规避 |
| [虚拟局域网联机方案](tools/虚拟局域网联机方案.md) | 用云服务器实现异地局域网游戏联机，WireGuard / Tailscale / ZeroTier / n2n / OpenVPN 五种方案全解 |
| [Caddy 反向代理完全指南](tools/caddy-反向代理指南.md) | Caddy 入门到泛域名证书，常见自建服务反向代理模板、访问控制、日志与调试 |
| [SSH 端口转发完全指南](tools/ssh-端口转发指南.md) | SSH 隧道详解：`-L` 本地转发、`-R` 远程转发、`-D` 动态代理，附带常用场景和管理命令 |
| [Tailscale 完全指南](tools/tailscale-完全指南.md) | 基于 WireGuard 的零配置 VPN，安装接入、MagicDNS、Exit Node 出口节点、Subnet Router 子网路由、SSH 集成、Headscale 自建 |
| [LLM 会话上下文与故障转移原理](tools/llm-会话上下文与failover.md) | LLM 无状态本质、cc-switch failover 上下文无损原理、O(n²) 长对话性能衰减、Compaction/Prompt Cache 缓解策略 |
| [ECC 插件 Java 开发指南](tools/ecc-java开发指南.md) | Claude Code ECC 插件体系的 Java 开发用法：agents/skills/rules/commands 全览，构建修复、代码审查、TDD、安全加固工作流 |
| [ECC Plan 命令完全指南](tools/ecc-plan命令指南.md) | ECC 规划命令家族全解：/plan、/plan-prd、PRP 深度流程（/prp-prd→/prp-plan→/prp-implement）、/plan-orchestrate 桥接，含选择指南 |
| [Vim 入门指南](tools/vim-入门指南.md) | Vim 模式化编辑思想，四模式对照，高频操作速查（移动/编辑/搜索替换/分屏/宏），最小 .vimrc 配置与学习路径 |
| [MySQL 与 PostgreSQL 分区详解](tools/mysql-pg分区详解.md) | 两种数据库分区全解：分区类型/建表语法、归档操作（EXCHANGE vs DETACH）、pg_partman 自动化、硬限制对比与分区裁剪优化 |

### Linux 标准与基础

| 文档 | 简介 |
|------|------|
| [XDG Base Directory 规范详解](standards/xdg-base-directory-详解.md) | `.config` `.cache` `.local` 的目录规律，XDG 环境变量体系，dotfile 迁移指南与备份策略 |
| [文件系统层级标准（FHS）详解](standards/fhs-文件系统层级标准-详解.md) | `/` 下所有目录的命名逻辑与用途，`/usr merge`、`/var`、与 XDG 的对应关系 |

## 目录结构

```
linux-doc/
├── archlinux/    ← Arch Linux 运维
├── btrfs/        ← Btrfs 文件系统专题
├── tools/        ← 压缩等工具使用
├── standards/    ← Linux 标准与基础规范
├── README.md
└── CLAUDE.md
```

## 技术栈

- **系统**: Arch Linux
- **文件系统**: Btrfs（RAID0、子卷、快照、CoW、透明压缩）
- **压缩**: Zstandard (zstd)、tar
- **引导**: GRUB、EFI
- **标准**: XDG Base Directory、freedesktop.org 规范

## 文档约定

- 使用**简体中文**撰写
- 技术术语、命令、代码块保留原始语言（英文）
- 代码块标注语言类型（`bash`、`sh`、`powershell` 等）
- 命令示例优先使用 `-a`（auto-compress）简写形式
- 每个文档末尾包含**常见问题**章节
- 命令行中的占位参数使用 `/path/to/...` 形式

## 参考

- [Arch Wiki](https://wiki.archlinux.org/)
- [Btrfs 官方文档](https://btrfs.readthedocs.io/)
- [Zstandard](https://facebook.github.io/zstd/)
