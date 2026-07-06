# Linux 文件系统层级标准（FHS）详解

## 1. 什么是 FHS？

**FHS（Filesystem Hierarchy Standard，文件系统层级标准）** 定义了 Linux 发行版中目录的布局、命名和用途。它回答了"这个文件该放哪里？"这一根本问题——没有 FHS，每个发行版的目录结构都不同，软件将无法移植。

> **一句话总结：** 规定了 `/` 下所有目录的名字、含义和里面应该放什么东西。

FHS 由 Linux Foundation 维护，最新版本为 3.0（2015 年发布）。绝大多数 Linux 发行版（包括 Arch Linux）都遵循该标准，尽管各自有细微偏差。

## 2. 目录命名的两大来源

Linux 目录命名大致分为两派：

| 来源 | 代表目录 | 命名逻辑 |
|------|----------|----------|
| **Unix 历史缩写** | `etc` `usr` `var` `opt` `bin` `sbin` `lib` | 短的、缩写的、省打字（1970 年代终端慢） |
| **自描述单词** | `boot` `home` `root` `media` `run` `srv` | 名字即含义，一目了然 |

> **历史趣闻：** Unix 早期，终端是电传打字机，速度极慢（约 10 字符/秒）。打 `etc` 比 `configuration` 快得多，打 `bin` 比 `programs` 快得多。这些缩写从 1971 年第一版 Unix 沿用至今。

## 3. 根目录完整清单

### 3.1 命令与程序

| 目录 | 全称 | 用途 | 典型内容 |
|------|------|------|----------|
| `/bin` | Binaries | 所有用户必需的基本命令 | `ls` `cp` `cat` `sh` `echo` `mkdir` |
| `/sbin` | System Binaries | 系统管理员命令（通常需 root） | `fdisk` `mkfs` `iptables` `sysctl` `mount` |
| `/usr/bin` | User Binaries | 非必需用户程序 | `vim` `python` `firefox` `git` `gcc` |
| `/usr/sbin` | User System Binaries | 非必需系统服务守护进程 | `sshd` `httpd` `cron` `NetworkManager` |

> **`/usr merge`（见第 6 节）后，`/bin` 和 `/sbin` 在现代 Arch Linux 上已变为指向 `/usr/bin` 的符号链接。**

### 3.2 库与内核

| 目录 | 全称 | 用途 | 典型内容 |
|------|------|------|----------|
| `/lib` | Libraries | `/bin` 和 `/sbin` 程序依赖的共享库 | `libc.so.6` `ld-linux.so.2` |
| `/lib64` | 64-bit Libraries | 64 位架构的库（部分发行版） | 同 `/lib` 的 64 位版本 |
| `/usr/lib` | User Libraries | 用户程序的共享库 | `libgtk.so` `libpython3.so` |
| `/usr/lib/modules` | Kernel Modules | 内核模块 | `ext4.ko.zst` `nvidia.ko.zst` |

### 3.3 配置与数据

| 目录 | 全称 | 用途 | 典型内容 |
|------|------|------|----------|
| `/etc` | Et Cetera（杂项） | 系统级配置文件 | `fstab` `passwd` `ssh/` `systemd/` `pacman.conf` |
| `/usr/share` | Shared Data | 架构无关的只读数据 | man 手册、图标、字体、locale 翻译、文档 |
| `/usr/local` | Local | 管理员手动编译安装的软件 | 自建 `/usr/local/bin/` `/usr/local/etc/` 等 |
| `/var` | Variable | 运行时内容变化的数据 | 日志、缓存、队列、锁文件、包管理器数据库 |

### 3.4 系统与硬件

| 目录 | 全称 | 用途 | 典型内容 |
|------|------|------|----------|
| `/dev` | Devices | 设备文件（一切皆文件） | `sda` `tty` `null` `random` `zero` |
| `/proc` | Process | 进程与内核信息（虚拟文件系统） | `cpuinfo` `meminfo` `/proc/<PID>/` |
| `/sys` | System | 内核设备/驱动信息（虚拟文件系统） | `/sys/class/` `/sys/bus/` `/sys/block/` |

> **`/proc` 和 `/sys` 不占用磁盘空间**——它们是内核向用户空间暴露信息的窗口，文件内容由内核即时生成。

### 3.5 临时与运行时

| 目录 | 全称 | 用途 | 典型内容 |
|------|------|------|----------|
| `/tmp` | Temporary | 所有用户的临时文件 | 编辑器临时文件、编译中间产物 |
| `/run` | Runtime | 本次启动的运行时数据（tmpfs） | PID 文件、套接字、锁文件 |
| `/var/tmp` | Variable Temporary | 持久性临时文件（重启后保留） | 长时间运行的任务中间文件 |

两个临时目录的关键区别：

| 特性 | `/tmp` | `/var/tmp` |
|------|--------|------------|
| 重启后 | 通常清空 | **保留** |
| 自动清理 | systemd-tmpfiles 定时清理（默认 10 天） | 默认 30 天 |
| 适用场景 | 短命的临时数据 | 需跨重启保留的临时数据 |

### 3.6 启动与引导

| 目录 | 用途 | 典型内容 |
|------|------|----------|
| `/boot` | 系统启动所需文件 | 内核镜像（`vmlinuz-linux`）、initramfs、GRUB 配置 |

### 3.7 用户与权限

| 目录 | 用途 |
|------|------|
| `/home` | 普通用户的家目录 |
| `/root` | root 用户的独立家目录 |

> **为什么 root 的家目录不放在 `/home/root`？** 因为 `/home` 可能是一个独立分区或网络挂载，万一挂载失败，root 仍需能登录系统修问题。所以 root 的家目录放在根文件系统上。

### 3.8 挂载与外部设备

| 目录 | 全称 | 用途 | 谁管理 |
|------|------|------|--------|
| `/mnt` | Mount | 管理员手动临时挂载 | 人（`mount /dev/sdb1 /mnt`） |
| `/media` | Media | 可移动设备自动挂载 | udisks/系统（U 盘插上自动挂载到 `/media/user/LABEL/`） |

### 3.9 可选与第三方

| 目录 | 全称 | 用途 | 典型内容 |
|------|------|------|----------|
| `/opt` | Optional | 第三方商业/独立软件 | `/opt/google/chrome/` `/opt/idea/` |
| `/srv` | Service | 服务器对外提供的数据 | `/srv/http/`（网站文件）`/srv/ftp/` |

### 3.10 完整目录树一览

```
/
├── bin  → usr/bin    ← 符号链接：人人用的基础命令
├── sbin → usr/bin    ← 符号链接：管理员用的系统命令
├── lib  → usr/lib    ← 符号链接：程序依赖的共享库
├── lib64 → usr/lib   ← 符号链接：64 位共享库
│
├── etc/              ← 系统级配置（杂货铺）
├── usr/              ← 只读系统资源（用户程序的主体）
│   ├── bin/          ←  非必需用户命令
│   ├── sbin/         ←  非必需系统守护进程
│   ├── lib/          ←  用户程序共享库
│   ├── share/        ←  架构无关数据（文档、图标、翻译）
│   └── local/        ←  管理员手动编译安装的软件
│
├── var/              ← 可变数据（日志、缓存、队列、包管理器 DB）
│   ├── log/          ←  系统与服务日志
│   ├── cache/        ←  应用缓存
│   └── lib/          ←  应用状态数据（如 pacman 数据库）
│
├── dev/              ← 设备文件（磁盘、键盘、终端…）
├── proc/             ← 内核进程信息（虚拟）
├── sys/              ← 内核设备信息（虚拟）
│
├── boot/             ← 内核与引导程序
├── tmp/              ← 临时文件（重启清空）
├── run/              ← 本次启动的运行时数据（tmpfs）
│
├── home/             ← 普通用户的家目录
├── root/             ← root 用户的家目录
│
├── mnt/              ← 手动临时挂载
├── media/            ← U 盘光盘自动挂载
│
├── opt/              ← 第三方可选软件
└── srv/              ← 服务对外提供的数据
```

## 4. 设计逻辑

整棵目录树看似复杂，实则只沿着**两个维度**分化：

### 4.1 维度一：可共享 vs 仅本机

```
可跨网络共享（NFS）           仅本机
─────────────────────────────────────────
/usr（只读）                  /var（每台机器独立写日志/缓存）
/home（NFS 挂载给多台机器）   /tmp  /run
                             /dev  /proc  /sys
```

这是 Unix 工作站时代的遗产：一台文件服务器导出 `/usr`（只读），几十台无盘工作站通过网络挂载它。`/usr` 共享、`/var` 独立。

### 4.2 维度二：只读 vs 可写

```
只读                          可写
──────────────────────────────────────
/usr  /usr/share              /var  /tmp  /run
/bin  /lib  /sbin（历史）     /home  /root
/etc（理论上，实际上常改）
```

### 4.3 三层级结构化

```
/                   ← 系统启动最小集合（静态）
├── /usr/           ← 操作系统的主体（静态、共享）
├── /usr/local/     ← 站点级本地安装（静态、非共享）
└── /var/           ← 运行时数据（动态）
```

这是 FHS 设计中最精妙的部分——**同一软件可以三层共存**：

```bash
/usr/bin/myapp           # 包管理器安装的
/usr/local/bin/myapp     # 手动编译的（覆盖包管理器版本）
# 因为 PATH 通常先搜 /usr/local/bin
```

## 5. `/usr` 详解——最容易被误解的目录

### 5.1 为什么叫 `usr`？

**不是 `user` 的缩写。**

最初（Unix 第一版），`/usr` 确实是用户家目录。但很快用户目录迁到了 `/home`（事实上）和后来标准化的 `/home`（FHS 后）。`/usr` 转而成为存放**操作系统主体**的地方。

如今普遍接受的解读是 **Unix System Resources**（Unix 系统资源）。

### 5.2 `/usr` 的子目录

```
/usr/
├── bin/              ← 用户命令（vim, git, python, gcc...）
├── sbin/             ← 系统守护进程（sshd, httpd...）
├── lib/              ← 共享库（.so 文件）
├── lib/modules/      ← 内核模块
├── include/          ← C/C++ 头文件（.h）
├── share/            ← 架构无关数据
│   ├── man/          ←  man 手册页
│   ├── doc/          ←  软件文档
│   ├── icons/        ←  系统图标
│   ├── fonts/        ←  系统字体
│   ├── locale/       ←  本地化翻译
│   └── licenses/     ←  软件许可证
├── src/              ← 内核源码（可选）
└── local/            ← 管理员手动编译安装的软件
    ├── bin/          ←  手动安装的命令
    ├── etc/          ←  手动安装的配置
    ├── lib/          ←  手动安装的库
    └── share/        ←  手动安装的数据
```

### 5.3 `/usr/local` vs `/opt`

| 特性 | `/usr/local` | `/opt` |
|------|-------------|--------|
| 安装方式 | 管理员手动从源码编译 | 第三方预编译二进制包 |
| 结构 | 遵循 FHS（有 `bin/` `lib/` `share/` 等） | 自包含（`/opt/app/` 下面一切独立） |
| 典型例子 | `./configure && make && make install` | Google Chrome、IntelliJ IDEA |
| 路径风格 | 像系统的一部分 | 像 Windows 的 `C:\Program Files\` |

## 6. 现代变化：`/usr merge`

### 6.1 什么是 `/usr merge`？

传统上，`/bin`、`/sbin`、`/lib`（根目录下的）和 `/usr/bin`、`/usr/sbin`、`/usr/lib` 是分开的：

- `/bin` → 系统启动和单用户模式必需的基本命令
- `/usr/bin` → 完整多用户模式下的所有其他命令

但在 2010 年代，大多发行版（Arch 在 2013 年）做了 **`/usr merge`**：将 `/bin`、`/sbin`、`/lib*` 统一到 `/usr` 下，原路径变成符号链接。

### 6.2 你的系统上

```bash
$ ls -l /bin /sbin /lib /lib64
lrwxrwxrwx 1 root root 7  /bin -> usr/bin
lrwxrwxrwx 1 root root 7  /sbin -> usr/bin
lrwxrwxrwx 1 root root 7  /lib -> usr/lib
lrwxrwxrwx 1 root root 7  /lib64 -> usr/lib
```

### 6.3 为什么？

当初区分"最小启动命令"和"全功能命令"的背景是 `/usr` 可能是一个独立的、需要单独挂载的分区。如今 initramfs 能处理所有挂载逻辑，区分已无意义。合并后：

- 目录结构更简洁
- 兼容性更好（其它 Unix 系统原本就不区分）
- 软件打包更简单（不用纠结命令放 `/bin` 还是 `/usr/bin`）

## 7. `/var` 详解——"会变的数据"

`/var`（variable）是与 `/usr`（read-only in spirit）形成对比的核心概念。

```
/var/
├── log/              ← 系统与服务日志
│   ├── pacman.log    ←  pacman 安装历史
│   ├── Xorg.0.log    ←  X/Wayland 日志
│   └── journal/      ←  systemd 日志
│
├── cache/            ← 应用缓存
│   ├── pacman/       ←  pacman 下载的包缓存（/var/cache/pacman/pkg/）
│   └── fontconfig/   ←  字体缓存
│
├── lib/              ← 应用持久状态
│   ├── pacman/       ←  pacman 数据库（已安装包列表）
│   ├── systemd/      ←  systemd 状态
│   └── docker/       ←  Docker 镜像与容器数据
│
├── tmp/              ← 持久临时文件（重启保留）
├── spool/            ← 队列数据（打印队列、邮件队列）
└── lock/ → ../run/lock  ← 符号链接到 /run/lock
```

> **`/var/lib` vs `~/.local/share`:** 前者是系统级应用状态（由 root 管理），后者是用户级数据（由用户管理），它们的关系如同 `/etc` 之于 `~/.config`——层级不同，逻辑相同。

## 8. 与用户 XDG 目录的对应关系

前一篇文档讲的 XDG 用户目录与 FHS 系统目录存在清晰的**镜像关系**：

| 层级 | 系统级（FHS） | 用户级（XDG） |
|------|-------------|-------------|
| 配置文件 | `/etc/` | `~/.config/` |
| 数据文件 | `/usr/share/` | `~/.local/share/` |
| 缓存 | `/var/cache/` | `~/.cache/` |
| 状态 | `/var/lib/` | `~/.local/state/` |
| 运行时 | `/run/` | `$XDG_RUNTIME_DIR`（`/run/user/$UID/`） |

**理解了 FHS，XDG 就是它的用户级翻版。** 两套规范共享同一套设计哲学：配置/数据/缓存/状态分离，层级优先，可备份性。

## 9. 实用技巧

### 9.1 查找文件属于哪个包

```bash
# 查看某文件由哪个包安装
pacman -Qo /usr/bin/vim
# → /usr/bin/vim is owned by vim 9.x.x

# 列出某包安装了哪些文件
pacman -Ql vim | head -20
```

### 9.2 排查"手动装的 vs 包管理器装的"

```bash
# /usr/local 下的文件不属于任何包，一定是手动装的
find /usr/local -type f 2>/dev/null

# /opt 下的通常也是手动或第三方安装的
ls /opt/
```

### 9.3 磁盘空间诊断

```bash
# 查看根文件系统各目录占用
du -sh --threshold=100M /*/ 2>/dev/null | sort -rh

# /var 是大户的可能性最高（日志、缓存、Docker、pacman 缓存）
du -sh /var/log /var/cache /var/lib/docker 2>/dev/null

# pacman 缓存清理（保留最近的 3 个版本）
paccache -r
# 或全部清空（可从仓库重新下载）
sudo pacman -Scc
```

### 9.4 快速定位配置文件

```bash
# 找某应用的所有文件
pacman -Ql <package> | grep -E '(etc|conf|config)'

# 或全局搜索
find /etc -name '*application*'
```

## 10. 常见问题

### Q: `/bin` 和 `/usr/bin` 到底有什么区别？

历史上 `/bin` 存放系统启动必需的命令（`mount`、`sh`、`ls`），`/usr/bin` 存放非必需的命令（`vim`、`gcc`）。但 **`/usr merge` 之后**，在现代 Arch Linux 上 `/bin` 已是 `/usr/bin` 的符号链接，两者没有区别。

### Q: 为什么有些发行版不遵守 FHS？

大多数发行版遵循 FHS 的**主体框架**，但在细节上各有偏差：

- **NixOS / GNU Guix** 彻底抛弃 FHS，每个软件包装在自己的独立目录（`/nix/store/<hash>-package-version/`），"依赖地狱"消失了但也失去了传统 FHS 的直觉性
- **Fedora** 在 `/usr/lib64` vs `/usr/lib` 上与 Arch 略有不同
- **Android**（基于 Linux 内核）完全不使用 FHS

### Q: `/usr/local` 和 `/opt` 放哪个？

| 情况 | 放哪 |
|------|------|
| 从源码 `./configure && make && make install` | `/usr/local` |
| 下载的第三方二进制包，想隔离 | `/opt/<name>/` |
| 不确定 | `/usr/local`（更常见的习惯） |

### Q: 能像 Windows 那样把软件装到自定义路径吗？

技术上可以——但 Linux 生态的惯例是遵循 FHS。把软件装到 `/opt` 或 `/usr/local` 是最接近 "自定义安装路径" 的方式。如果需要完全隔离，Flatpak/Snap 等容器化的方式更推荐。

### Q: `/var` 占用很大空间怎么办？

```bash
# 1. 查谁最大
du -sh /var/*/ 2>/dev/null | sort -rh | head -10

# 2. 按情况处理
sudo journalctl --vacuum-size=500M     # systemd 日志限 500MB
sudo paccache -rk 2                    # pacman 缓存只保留 2 个版本
sudo pacman -Scc                       # 全部清空 pacman 缓存
sudo docker system prune -a            # 清理 Docker 无用数据
```

### Q: FHS 规范文档在哪里看？

```bash
# Arch 上 man hier 提供了概览
man 7 hier

# 在线版本
# https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html
# https://wiki.archlinux.org/title/File_system
```

## 参考

- [FHS 3.0 官方规范](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)
- [Arch Wiki — File system](https://wiki.archlinux.org/title/File_system)
- [Arch Wiki — /usr merge](https://wiki.archlinux.org/title/Frequently_asked_questions#Does_Arch_follow_the_Linux_Foundation.27s_Filesystem_Hierarchy_Standard_.28FHS.29.3F)
- [man 7 hier](https://man.archlinux.org/man/hier.7)
