# Pacman 完全使用指南

## 1. 概述

`pacman`（**pac**kage **man**ager）是 Arch Linux 的官方包管理器，由 Arch 社区独立开发。它结合了简单的二进制包格式和易用的构建系统，目标是让用户**轻松管理软件包**——无论是来自官方仓库还是用户自己构建的包。

### 1.1 核心概念

| 概念 | 说明 |
|------|------|
| **软件包（Package）** | `.pkg.tar.zst` 格式的压缩包，包含程序、库及元信息 |
| **仓库（Repository）** | 软件包的分组集合，如 `core`、`extra`、`multilib` |
| **数据库（Database）** | 本地 `/var/lib/pacman/` 下的包状态记录 |
| **Pacman 配置文件** | `/etc/pacman.conf` — 仓库源、镜像、选项 |
| **镜像列表** | `/etc/pacman.d/mirrorlist` — 下载源地址 |

### 1.2 选项命名规则

pacman 的操作选项遵循一套命名规则：

| 分类 | 选项前缀 | 含义 | 示例 |
|------|----------|------|------|
| 同步/安装 | `-S` | **S**ync | `pacman -S firefox` |
| 删除 | `-R` | **R**emove | `pacman -R firefox` |
| 查询数据库 | `-Q` | **Q**uery | `pacman -Qi firefox` |
| 查询文件 | `-F` | **F**ile | `pacman -F /usr/bin/firefox` |
| 本地安装 | `-U` | **U**pgrade | `pacman -U file.pkg.tar.zst` |

子选项命名规则（可组合）：

| 子选项 | 含义 | 搭配示例 |
|--------|------|----------|
| `-s` | 递归/搜索（recur**s**ive） | `-Rs`（递归删除）、`-Ss`（搜索） |
| `-y` | 刷新数据库（refre**s**h） | `-Sy`、`-Syu` |
| `-u` | 升级（**u**pgrade） | `-Syu` |
| `-i` | 信息（**i**nfo） | `-Qi`、`-Si` |
| `-l` | 列表（**l**ist） | `-Ql`、`-Fl` |
| `-d` | 依赖（**d**ependency） | `-Qd`、`-Rdd` |
| `-t` | 未使用的（unneeded） | `-Qdt` |
| `-n` | 无保存（**n**osave） | `-Rsn` |
| `-c` | 清理/变更日志（**c**lean） | `-Sc`（清理缓存）、`-Qc`（查看变更日志） |
| `-e` | 显式安装（**e**xplicit） | `-Qe`、`-D --asexplicit` |
| `-k` | 检查（chec**k**） | `-Qk`（检查包文件完整性） |

---

## 2. 基本包操作

### 2.1 安装软件包

```bash
# 安装单个包
pacman -S firefox

# 安装多个包（空格分隔）
pacman -S firefox vim htop

# 从文件安装本地包
pacman -U /path/to/package.pkg.tar.zst

# 从 URL 直接安装
pacman -U https://example.com/package.pkg.tar.zst
```

### 2.2 删除软件包

```bash
# 仅删除包，保留依赖
pacman -R firefox

# 删除包及其未被其他包需要的依赖（推荐日常使用）
pacman -Rs firefox

# 删除包、依赖 + 全局配置文件（谨慎使用）
pacman -Rsn firefox

# 强制删除（不检查依赖关系——可能导致系统损坏）
pacman -Rdd firefox

# 递归删除整个依赖链（删除包以及所有依赖它的包）
pacman -Rsc firefox
```

> **⚠️ 注意**：`-Rsn` 会删除全局 `.conf` 文件。如果不确定，先用 `-Rs`，之后手动清理 `/etc` 下的配置文件。

### 2.3 升级系统

```bash
# 全面系统升级（最常用——先同步数据库，再升级）
pacman -Syu

# 仅同步数据库不升级（一般不单独使用）
pacman -Sy

# 仅升级已安装的包（不同步数据库）
pacman -Su

# 强制刷新数据库 + 升级（换镜像后或数据库异常时使用）
pacman -Syyu
```

> **⚠️ 部分升级警告**：永远不要单独 `pacman -Sy` 然后安装包——这会创建"部分升级"状态（旧包依赖新库或反之），可能导致系统崩溃。安装新包前务必先全面升级：`pacman -Syu package`。

### 2.4 搜索与查询

#### 搜索

```bash
# 搜索仓库中的包（按名称+描述）
pacman -Ss firefox

# 搜索已安装的包
pacman -Qs firefox

# 远程搜索文件属于哪个包（按名称查）
pacman -Fs firefox

# 远程搜索文件属于哪个包（按完整路径查）
pacman -F /usr/bin/firefox
```

#### 查看信息

```bash
# 查看仓库中包的详细信息
pacman -Si firefox

# 查看已安装包的详细信息
pacman -Qi firefox

# 查看包的依赖树（双层 -ii 显示更多信息）
pacman -Sii firefox
```

#### 列出文件

```bash
# 列出仓库中的包拥有的文件
pacman -Fl firefox

# 列出已安装的包拥有的文件
pacman -Ql firefox

# 查询某个文件属于哪个已安装的包
pacman -Qo /usr/bin/firefox

# 查询某个文件属于哪个仓库中的包（即使未安装也能查）
pacman -F /usr/bin/firefox
```

#### 按安装原因查询

```bash
# 列出所有显式安装的包（用户主动安装的）
pacman -Qe

# 列出所有作为依赖安装的包
pacman -Qd

# 列出孤立包（作为依赖安装、但不再被任何包需要）
pacman -Qdt
```

#### 检查完整性

```bash
# 检查所有已安装包的文件完整性
pacman -Qk

# 检查特定包
pacman -Qk firefox

# 仅显示有问题的包
pacman -Qk 2>/dev/null | grep -v ' 0 missing'

# 查看包的 changelog（变更日志）
pacman -Qc firefox
```

#### 查看升级列表

```bash
# 查看有哪些可升级的包（不实际升级）
pacman -Qu

# 配合 checkupdates 脚本查看更新
checkupdates
```

### 2.5 缓存管理

```bash
# 查看缓存目录大小
du -sh /var/cache/pacman/pkg/

# 清理未安装包的所有缓存版本
pacman -Sc

# 清理全部包缓存（包括当前安装包的旧版本）
pacman -Scc

# 仅清理已卸载包的缓存（保留当前已安装包的各版本）
paccache -rk 1                          # 保留最近 1 版
paccache -rk 2                          # 保留最近 2 版

# 仅保留未安装包的缓存
paccache -ruk0
```

> `paccache` 来自 `pacman-contrib` 包，比 `pacman -Sc` 更灵活。

### 2.6 跳过特定包的升级

编辑 `/etc/pacman.conf`：

```ini
# 跳过特定包
IgnorePkg = linux linux-headers

# 跳过整组包
IgnoreGroup = gnome kde-applications
```

临时跳过（单次生效）：
```bash
pacman -Syu --ignore=linux
```

### 2.7 查看操作历史

```bash
# pacman 日志文件
cat /var/log/pacman.log

# 用 tail 查看最近的记录
tail -n 50 /var/log/pacman.log

# 搜索特定包的安装记录
grep firefox /var/log/pacman.log
```

---

## 3. 仓库管理

### 3.1 仓库层级

Arch Linux 官方仓库按层级划分：

| 仓库 | 说明 | 典型内容 |
|------|------|----------|
| `core` | 系统核心包 | `linux`、`systemd`、`pacman`、`glibc` |
| `extra` | 社区维护包 | `firefox`、`vim`、`git`、`python` |
| `multilib` | 32 位兼容包 | `wine`、`steam`、`lib32-glibc` |
| `testing` | 测试包（不推荐日常使用） | 预发布版本 |

### 3.2 启用/禁用仓库

编辑 `/etc/pacman.conf`：

```ini
[core]
Include = /etc/pacman.d/mirrorlist

[extra]
Include = /etc/pacman.d/mirrorlist

# 启用 multilib（需要 32 位支持时取消注释）
[multilib]
Include = /etc/pacman.d/mirrorlist

# 禁用 testing 仓库（注释掉即可）
#[testing]
#Include = /etc/pacman.d/mirrorlist
```

### 3.3 换镜像源

```bash
# 查看当前使用的镜像
grep -v '^#\|^$' /etc/pacman.d/mirrorlist

# 使用 reflector 自动选择最快的镜像（推荐）
reflector --country China --age 12 --protocol https --sort rate \
          --save /etc/pacman.d/mirrorlist

# 仅列出排名不做保存
reflector --country China --age 12 --protocol https --sort rate

# 手动编辑镜像列表
vim /etc/pacman.d/mirrorlist
```

> **提示**：换镜像后必须执行 `pacman -Syyu`（两个 `y`）强制重新下载数据库。

### 3.4 添加非官方仓库

以 `archlinuxcn`（Arch Linux 中文社区仓库）为例：

```bash
# 在 /etc/pacman.conf 末尾添加：
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch

# 导入 GPG 密钥
pacman -Sy archlinuxcn-keyring

# 如需手动导入
pacman-key --lsign-key "farseerfc@archlinux.org"
```

---

## 4. 包管理工具

### 4.1 `pacman-db-upgrade` — 数据库格式升级

```bash
# pacman 大版本更新后可能需要
pacman-db-upgrade
```

### 4.2 `pacman-key` — GPG 密钥管理

```bash
# 初始化密钥环（新系统安装时）
pacman-key --init
pacman-key --populate archlinux

# 刷新所有密钥
pacman-key --refresh-keys

# 手动添加信任密钥
pacman-key --recv-keys KEYID
pacman-key --lsign-key KEYID
```

### 4.3 `pacman-contrib` 实用工具

```bash
pacman -S pacman-contrib
```

| 工具 | 功能 |
|------|------|
| `paccache` | 灵活的包缓存清理 |
| `checkupdates` | 安全地检查更新（不修改数据库） |
| `pacdiff` | 查找并合并 `.pacnew` 配置文件 |
| `paclist` | 列出某仓库中的所有已安装包 |
| `paclog-pkglist` | 从日志重建已安装包列表 |
| `pacsort` | 排序/合并包列表 |
| `rankmirrors` | 按速度排序镜像 |
| `updpkgsums` | 更新 PKGBUILD 中的校验和 |

---

## 5. AUR 与辅助工具

### 5.1 什么是 AUR

**Arch User Repository**（AUR）是社区驱动的包仓库，托管的是 PKGBUILD 构建脚本而非预编译二进制包。用户下载 PKGBUILD 后在本地编译安装。

> **⚠️ 安全提醒**：AUR 包由社区贡献，无人审核。安装前务必检查 PKGBUILD 和 `.install` 文件内容。

### 5.2 手动从 AUR 构建

```bash
# 克隆 AUR 包
git clone https://aur.archlinux.org/package-name.git
cd package-name

# 审查 PKGBUILD（安全步骤）
cat PKGBUILD

# 构建并安装（-s 自动安装依赖，-i 安装构建好的包）
makepkg -si

# 仅构建不安装
makepkg -s

# 构建后清理
makepkg -sic
```

### 5.3 AUR 助手 — yay

[yay](https://github.com/Jguer/yay) 是目前最流行的 AUR 助手，语法与 pacman 高度兼容：

```bash
# 安装 yay（它本身在 AUR 中）
git clone https://aur.archlinux.org/yay.git
cd yay && makepkg -si
```

#### yay 常用操作

```bash
# 安装包（自动判断官方仓库还是 AUR）
yay -S firefox           # 官方仓库
yay -S aur-package       # AUR

# 系统全面升级（含 AUR）
yay -Syu

# 仅升级 AUR 包
yay -Sua

# 搜索（含 AUR）
yay -Ss keyword

# 删除包
yay -R package
yay -Rs package          # 递归删除依赖

# 清理缓存
yay -Sc

# 清理孤立包
yay -Yc

# 查看包信息
yay -Si package

# 打印系统信息（诊断用）
yay -Ps

# 生成 PKGBUILD 但让用户手动审查
yay -G package
```

#### 其他常见 AUR 助手

| 助手 | 特点 |
|------|------|
| `paru` | Rust 实现，与 yay 功能相似 |
| `aura` | Haskell 实现，支持快照回滚 |
| `pacaur` | 最小化设计，但已停止维护 |
| `trizen` | Perl 实现 |

---

## 6. Pacman 钩子（Hooks）

pacman 钩子是 `/usr/share/libalpm/hooks/` 下的配置文件，在特定包安装/升级/删除前后自动触发脚本。

### 6.1 查看已注册的钩子

```bash
ls /usr/share/libalpm/hooks/

# 查看钩子内容
cat /usr/share/libalpm/hooks/mkinitcpio-install.hook
```

### 6.2 常见系统钩子

| 钩子 | 触发条件 | 作用 |
|------|----------|------|
| `mkinitcpio` | 内核更新 | 重建 initramfs |
| `systemd-boot` | systemd 更新 | 更新引导入口 |
| `pacman-mirrorupgrade` | 镜像列表包更新 | 更新 mirrorlist |
| `gtk-update-icon-cache` | 图标主题更新 | 重建图标缓存 |
| `update-desktop-database` | `.desktop` 文件变化 | 更新应用菜单 |
| `fontconfig` | 字体变化 | 重建字体缓存 |
| `texinfo` | info 文档更新 | 更新 info 目录 |

### 6.3 钩子文件结构

```ini
[Trigger]
Type = Path          # 触发类型：Path 或 Package
Target = usr/lib/modules/*/vmlinuz   # 监控的路径
Operation = Install
Operation = Upgrade
Operation = Remove

[Action]
Description = Updating linux initcpios...
When = PostTransaction
Exec = /usr/bin/mkinitcpio -P
NeedsTargets
```

---

## 7. 配置文件详解

`/etc/pacman.conf` 关键选项：

```ini
#
# /etc/pacman.conf
#
# 参见 man pacman.conf

[options]
# 架构（留空则自动检测）
Architecture = auto

# 并行下载数（默认 1，建议设为 5）
ParallelDownloads = 5

# 彩色输出
Color

# 下载前后都检查可用磁盘空间
CheckSpace

# 详细信息（显示每个包的大小、版本差异等）
VerbosePkgLists

# 并行编译（makepkg 用）
# MAKEFLAGS="-j$(nproc)"

# 跳过不需要的包升级
# IgnorePkg = linux
# IgnoreGroup = gnome

# 包签名检查级别
SigLevel = Required DatabaseOptional
LocalFileSigLevel = Optional

# 不包括的文件（从包中排除）
# NoExtract = etc/fstab
# NoUpgrade = etc/fstab
```

---

## 8. 常见操作速查

### 8.1 日常维护流程

```bash
# 每周一次全面维护
pacman -Syu                   # 系统升级
pacman -Qdtq | pacman -Rns -  # 清理孤立包
paccache -rk 2                # 清理旧版缓存
pacdiff                       # 合并 .pacnew 配置
```

### 8.2 查找并删除孤立包

```bash
# 查找孤立包
pacman -Qdt

# 安全删除（-q 只输出包名，无孤立包时不会报错）
pacman -Qdtq | xargs -r sudo pacman -Rns

# 或用 yay
yay -Yc
```

### 8.3 降级某个包

```bash
# 从 pacman 缓存中找旧版本
ls /var/cache/pacman/pkg/ | grep firefox
pacman -U /var/cache/pacman/pkg/firefox-130.0-1-x86_64.pkg.tar.zst

# 或用 downgrade 工具
yay -S downgrade
downgrade firefox
```

### 8.4 重新安装所有已安装的包

```bash
# 从损坏恢复
pacman -Qqn | pacman -S -
```

### 8.5 查找系统中不在任何包中的文件

```bash
yay -S lostfiles
lostfiles
```

### 8.6 查找被修改的配置文件

```bash
# 查找 .pacnew 文件
find /etc -name "*.pacnew"

# 比对差异
pacdiff
diff /etc/lightdm/lightdm.conf /etc/lightdm/lightdm.conf.pacnew
```

---

## 9. 常见问题

### Q1: GPG 密钥错误 `signature is unknown trust`

```bash
# 刷新密钥环
pacman -Sy archlinux-keyring && pacman -Su

# 或手动刷新
pacman-key --refresh-keys

# 如果持续不生效，重置密钥环
pacman-key --init
pacman-key --populate archlinux
```

### Q2: 数据库文件损坏 `failed to commit transaction`

```bash
# 强制更新数据库
pacman -Syyu

# 如果依然失败，删除数据库缓存后重试
rm -rf /var/lib/pacman/sync/*
pacman -Syyu
```

### Q3: 提示文件已存在 `exists in filesystem`

```bash
# 检查是哪个包拥有冲突文件
pacman -Qo /path/to/file

# 如果冲突文件来自旧包或手动安装，使用 --overwrite
pacman -S --overwrite '/path/to/file' package

# 全局覆盖（谨慎使用）
pacman -S --overwrite '*' package
```

### Q4: 锁文件存在 `unable to lock database`

```bash
# 查看是否有其他 pacman 进程
ps aux | grep pacman

# 如果没有，删除陈旧的锁文件
rm /var/lib/pacman/db.lck
```

### Q5: 空间不足 `not enough free disk space`

```bash
# 清理所有缓存
pacman -Scc

# 使用 paccache 更安全地清理
paccache -rk 1

# 检查是哪个目录占用大
du -sh /var/cache/pacman/pkg/
du -sh ~/.cache/yay/
```

### Q6: `-Sy` vs `-Syy` 什么时候用？

- **`-Sy`**：如果数据库版本比本地已有新才下载（节省带宽）
- **`-Syy`**：强制重新下载全部数据库——换镜像后、数据库异常时使用

### Q7: 如何锁定某个包的版本？

在 `/etc/pacman.conf` 中添加：
```ini
IgnorePkg = linux nvidia
```

对于 AUR 包，`IgnorePkg` 只对官方仓库有效。AUR 助手通常有自己的忽略方式：
```bash
yay --ignore aur-package -Syu
```

### Q8: 如何知道一个包是从哪个仓库安装的？

```bash
# 方法 1：查看包的来源仓库
paclist core       # 显示来自 core 仓库的已安装包
paclist extra      # 显示来自 extra 仓库的已安装包

# 方法 2：查看包信息
pacman -Qi firefox | grep -i 'repository\|required'
```

---

## 10. 参考资料

- [Arch Wiki: pacman](https://wiki.archlinux.org/title/Pacman)
- [Arch Wiki: pacman/Tips and tricks](https://wiki.archlinux.org/title/Pacman/Tips_and_tricks)
- [man pacman](https://man.archlinux.org/man/pacman.8)
- [man pacman.conf](https://man.archlinux.org/man/pacman.conf.5)
