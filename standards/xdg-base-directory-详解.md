# Linux XDG Base Directory 规范详解

## 1. 什么是 XDG Base Directory 规范？

**XDG Base Directory Specification** 是 freedesktop.org（前身为 X Desktop Group,简称 XDG）制定的一套目录规范,定义了应用程序在用户 `$HOME` 下存放文件的标准化路径。

> **一句话总结:** 把「配置」「数据」「缓存」「状态」分到不同目录,而不是像上世纪那样全部散落在 `~` 下。

规范最初于 2003 年发布(0.6 版),2010 年更新到 0.8 版,至今仍是 Linux 桌面生态最重要的基础规范之一。

## 2. 环境变量与默认路径

### 2.1 用户级目录(一套四变量)

| 环境变量 | 默认值 | 用途 |
|----------|--------|------|
| `XDG_CONFIG_HOME` | `~/.config` | 用户**配置文件** |
| `XDG_CACHE_HOME` | `~/.cache` | 用户**缓存数据** |
| `XDG_DATA_HOME` | `~/.local/share` | 用户**数据文件** |
| `XDG_STATE_HOME` | `~/.local/state` | 用户**状态数据** |

### 2.2 系统级目录(搜索路径)

| 环境变量 | 默认值 | 用途 |
|----------|--------|------|
| `XDG_CONFIG_DIRS` | `/etc/xdg` | 系统级配置搜索路径(`:` 分隔) |
| `XDG_DATA_DIRS` | `/usr/local/share:/usr/share` | 系统级数据搜索路径(`:` 分隔) |

### 2.3 运行时目录

| 环境变量 | 默认值 | 用途 |
|----------|--------|------|
| `XDG_RUNTIME_DIR` | `/run/user/$UID` | 运行时临时文件(套接字、管道、锁等) |

> **注意:** `XDG_RUNTIME_DIR` 必须由 pam_systemd 或等效机制创建,目录权限为 `0700`,所属为用户本人。系统重启后内容自动清除。

### 2.4 用户自定义目录(xdg-user-dirs)

除 Base Directory 规范外,还有一套 **xdg-user-dirs** 规范,定义文档、下载等目录:

| 环境变量 | 默认值 |
|----------|--------|
| `XDG_DESKTOP_DIR` | `$HOME/Desktop` |
| `XDG_DOWNLOAD_DIR` | `$HOME/Downloads` |
| `XDG_DOCUMENTS_DIR` | `$HOME/Documents` |
| `XDG_MUSIC_DIR` | `$HOME/Music` |
| `XDG_PICTURES_DIR` | `$HOME/Pictures` |
| `XDG_VIDEOS_DIR` | `$HOME/Videos` |
| `XDG_TEMPLATES_DIR` | `$HOME/Templates` |
| `XDG_PUBLICSHARE_DIR` | `$HOME/Public` |

配置文件位于 `~/.config/user-dirs.dirs`,可用 `xdg-user-dirs-update` 命令管理。

## 3. 各目录详解

### 3.1 `~/.config` — 配置文件(`XDG_CONFIG_HOME`)

存放应用程序的用户级配置。文件格式与内容完全由应用决定(INI、JSON、YAML、TOML、XML、纯文本等均可)。

```
~/.config/
├── nvim/              ← Neovim 编辑器配置
│   ├── init.lua
│   └── lua/
├── git/
│   └── config         ← Git 用户配置(身份、别名等)
├── fish/              ← Fish shell 配置
├── i3/ 或 sway/       ← 窗口管理器配置
├── waybar/            ← Waybar 状态栏配置
├── alacritty/         ← Alacritty 终端配置
├── fontconfig/        ← 字体配置
├── htop/              ← htop 配置
├── pulse/ 或 pipewire/ ← 音频配置
├── systemd/user/      ← 用户级 systemd 服务
├── gtk-3.0/           ← GTK 3 配置
├── gtk-4.0/           ← GTK 4 配置
└── Code/ 或 Code - Insiders/  ← VS Code 用户配置
```

### 3.2 `~/.cache` — 缓存数据(`XDG_CACHE_HOME`)

存放可安全删除的非关键数据。**删除该目录不应导致应用崩溃或数据丢失**,应用应在缓存缺失时自动重建。

```
~/.cache/
├── pip/               ← pip 下载的 wheel 包缓存
├── yay/ 或 paru/      ← AUR helper 的构建缓存
├── mozilla/           ← Firefox 缓存
├── chromium/          ← Chromium 缓存
├── thumbnails/        ← 图片缩略图缓存
└── mesa_shader_cache/ ← GPU 着色器缓存(删除后首启会卡)
```

> **清理技巧:** 怀疑某应用行为异常、或磁盘空间紧张时,可以优先清空 `~/.cache/` 下对应目录。系统级清理用 `rm -rf ~/.cache/*` 也完全安全(最多首启慢几秒)。

### 3.3 `~/.local/share` — 数据文件(`XDG_DATA_HOME`)

存放用户级持久数据。**这是应用程序最重要的用户数据目录**,备份系统时不可忽略。

```
~/.local/share/
├── fonts/             ← 用户安装的字体(~/.fonts 的现代替代)
├── applications/      ← 用户 .desktop 入口文件
├── icons/             ← 图标主题
├── themes/            ← GTK/Qt 主题
├── Trash/             ← 回收站(FreeDesktop Trash 规范)
├── Steam/             ← Steam 游戏库数据
├── konsole/           ← Konsole 书签和配置
├── fish/              ← Fish shell 历史(fish_history)
├── nvim/              ← Neovim 插件、undo、swap 等运行时数据
├── keyrings/          ← GNOME Keyring 密钥环
├── recent-used/       ← 最近访问文件列表(xbel 格式)
├── backgrounds/       ← 壁纸
├── gnupg/             ← GnuPG 密钥环(若设置 GNUPGHOME)
└── flatpak/           ← Flatpak 应用数据
```

### 3.4 `~/.local/state` — 状态数据(`XDG_STATE_HOME`)

介于缓存和数据之间:记录了应用的状态但并非用户创建的数据。例如 shell 历史命令、less 的阅读位置、视频播放器的播放进度等。丢失后体验受损但不影响核心数据。

```
~/.local/state/
├── bash/              ← Bash 历史(.bash_history 的现代替代)
├── less/              ← less 阅读位置记忆
├── wireplumber/       ← WirePlumber 会话状态
└── mpv/               ← mpv 播放进度
```

> **规范对比:** `XDG_STATE_HOME` 是 2021 年新增(规范 0.8 版),比前三个晚得多,因此支持的应用相对较少。许多应用仍将状态数据混放在 `.local/share/` 或 `.config/` 中。

### 3.5 `XDG_RUNTIME_DIR` — 运行时数据

与前三者最大的不同:**这是内存文件系统(tmpfs)上的临时目录,重启后消失**。

- 典型路径:`/run/user/1000`(1000 是 UID)
- 权限:`0700`,仅用户本人可访问
- 生命周期:用户登录时创建,最后一次会话结束时销毁

```
/run/user/1000/
├── bus                ← D-Bus 会话总线套接字
├── pipewire-0         ← PipeWire 音频套接字
├── wayland-1          ← Wayland 显示服务套接字
├── pulse/             ← PulseAudio 套接字
└── systemd/           ← 用户 systemd 实例
```

### 3.6 `XDG_CONFIG_DIRS` 与 `XDG_DATA_DIRS` — 系统级路径

这两个是**搜索路径列表**(`:` 分隔),应用按顺序查找,第一个匹配的优先。用户级目录(`XDG_CONFIG_HOME`、`XDG_DATA_HOME`)未显式列入搜索路径但**隐式优先**。

**优先级(从高到低):**

```
用户配置:   $XDG_CONFIG_HOME          → /etc/xdg                  (含子目录)
用户数据:   $XDG_DATA_HOME            → /usr/local/share → /usr/share
```

**示例:** 应用查找图标时:
1. 先查 `~/.local/share/icons/`(用户安装的图标)
2. 再查 `/usr/local/share/icons/`(管理员额外安装的)
3. 最后查 `/usr/share/icons/`(软件包管理器安装的)

这保证了用户可以覆盖系统级配置和数据。

## 4. 设计逻辑与原则

### 4.1 四维分离模型

```
                    持久性
                      ↑
     ~/.config       +++       配置:删了回到默认,需备份
     ~/.local/share  +++       数据:删了就没了,必须备份
     ~/.local/state   ++       状态:删了体验下降,可选备份
     ~/.cache          +       缓存:随时可删,不备份
     /run/user/$UID    -       运行时:重启就消失
                      ↓
```

### 4.2 核心设计原则

| 原则 | 说明 |
|------|------|
| **可删除性** | `.cache` 随时可删,应用不应崩溃 |
| **可备份性** | 备份 `.config` + `.local/share` 即可恢复绝大部分应用状态 |
| **隔离性** | 配置、数据、缓存三者互不污染 |
| **可覆盖性** | 用户级 > 系统级,允许个性化 |
| **无硬编码** | 所有路径通过环境变量获取,不发散 |

### 4.3 判断某文件该放哪

```
这是配置吗?(应用设置、偏好)        → ~/.config
这是用户创建/下载的数据吗?          → ~/.local/share
这是暂时的、可自动重建的吗?          → ~/.cache
这是会话级别的吗?(套接字、锁)      → $XDG_RUNTIME_DIR
这是日志/历史/最近记录吗?            → ~/.local/state
```

## 5. 历史背景:`$HOME` 的 dotfile 乱象

### 5.1 问题的起源

在 XDG 规范出现之前,Unix 应用遵循一个简单传统:将配置和数据以**隐藏文件/目录**的形式直接放在 `$HOME` 下。文件名以 `.` 开头使其在 `ls` 中默认不可见,避免视觉干扰。

这导致了著名的 **dotfile 泛滥**问题:

```bash
# 没有 XDG 规范的 $HOME 大概长这样:
~/
├── .bashrc
├── .bash_history
├── .profile
├── .ssh/
├── .gnupg/
├── .docker/
├── .npm/
├── .cargo/
├── .rustup/
├── .mozilla/
├── .thunderbird/
├── .config/         ← 有些应用迁移了...
├── .java/
├── .gradle/
├── .android/
├── .vscode/
├── .wine/
├── .steam/
├── .zshrc
├── .zsh_history
├── .python_history
├── .lesshst
├── .node_repl_history
├── .Xauthority
├── .ICEauthority
├── ...还有几十个
```

### 5.2 迁移难度

少数应用积极响应了 XDG 规范并迁移了默认路径,但许多老牌项目选择**保持向后兼容**——因为路径变更会影响数以亿计的用户和无数脚本。部分项目采取了妥协方案:

| 策略 | 示例 | 说明 |
|------|------|------|
| **全量迁移** | fish shell, nvim, fd, bat, mpv | 新项目无历史包袱 |
| **先旧后新 fallback** | Git | 有 `~/.gitconfig` 就用它,否则用 `~/.config/git/config` |
| **环境变量控制** | GnuPG, Docker, Cargo | 设置 `GNUPGHOME` 等可改变路径 |
| **坚决不迁移** | OpenSSH, Bash, Zsh | 路径硬编码或向后兼容约束太强 |

### 5.3 仍常见的非规范路径

| 路径 | 应用 | 应放位置 |
|------|------|----------|
| `~/.ssh/` | OpenSSH | `~/.config/ssh/` 或 `~/.local/share/ssh/` |
| `~/.gnupg/` | GnuPG | 可设 `GNUPGHOME` 迁移 |
| `~/.docker/` | Docker CLI | 可设 `DOCKER_CONFIG` 迁移 |
| `~/.npm/` | npm | npm 把配置和缓存混一起了 |
| `~/.cargo/` | Rust Cargo | 可设 `CARGO_HOME` 迁移 |
| `~/.rustup/` | Rustup | 可设 `RUSTUP_HOME` 迁移 |
| `~/.mozilla/` | Firefox | 无环境变量覆盖,代码中硬编码 |
| `~/.wine/` | Wine | 可设 `WINEPREFIX` 迁移 |
| `~/.python_history` | Python REPL | 无官方环境变量支持 |

> **现实检查:** 接受 `~` 下有这些 dotfile 比强制执行规范更务实。规范是目标,不是苛求。

## 6. 查看与设置

### 6.1 查看当前值

```bash
# 逐一查看
echo "XDG_CONFIG_HOME: ${XDG_CONFIG_HOME:-$HOME/.config}"
echo "XDG_CACHE_HOME:  ${XDG_CACHE_HOME:-$HOME/.cache}"
echo "XDG_DATA_HOME:   ${XDG_DATA_HOME:-$HOME/.local/share}"
echo "XDG_STATE_HOME:  ${XDG_STATE_HOME:-$HOME/.local/state}"
echo "XDG_RUNTIME_DIR: ${XDG_RUNTIME_DIR:-未设置}"

# 系统级搜索路径
echo "XDG_CONFIG_DIRS: ${XDG_CONFIG_DIRS:-/etc/xdg}"
echo "XDG_DATA_DIRS:   ${XDG_DATA_DIRS:-/usr/local/share:/usr/share}"
```

大多数发行版(包括 Arch Linux)**不主动设置这些变量**,应用通过 fallback 默认值工作。只有当用户想自定义位置时才需要设置。

### 6.2 自定义目录位置

在某些特殊场景下你可能想修改默认路径:

```bash
# 场景:SSD 空间有限,将缓存移到机械硬盘
export XDG_CACHE_HOME=/mnt/hdd/cache

# 场景:多发行版共享配置
export XDG_CONFIG_HOME=/mnt/shared/dotfiles/config

# 建议写在 ~/.config/environment.d/xdg.conf 中(systemd 用户实例读取)
# 或 ~/.pam_environment(影响全局 PAM 会话)
```

```ini
# ~/.config/environment.d/xdg.conf
XDG_CACHE_HOME=/mnt/hdd/cache
XDG_STATE_HOME=/mnt/hdd/state
```

> **警告:** 修改这些值会影响所有遵循 XDG 规范的应用。只改一个变量可能导致应用在非预期位置创建目录,务必全面测试后再设为永久。

### 6.3 迁移旧 dotfile 到规范路径

以下是一个集中清理 `$HOME` 的模板(**执行前请确保理解每条命令的含义**):

```bash
# --- GnuPG ---
# 目标:将 ~/.gnupg 移到 ~/.local/share/gnupg
export GNUPGHOME="$XDG_DATA_HOME/gnupg"
mkdir -p "$GNUPGHOME"
chmod 700 "$GNUPGHOME"
# 如果已有 ~/.gnupg,先迁移再设置环境变量
[ -d ~/.gnupg ] && mv ~/.gnupg/* "$GNUPGHOME/" && rmdir ~/.gnupg

# --- Docker CLI ---
export DOCKER_CONFIG="$XDG_CONFIG_HOME/docker"

# --- Rust / Cargo ---
export CARGO_HOME="$XDG_DATA_HOME/cargo"
export RUSTUP_HOME="$XDG_DATA_HOME/rustup"

# --- Node.js ---
export NPM_CONFIG_USERCONFIG="$XDG_CONFIG_HOME/npm/npmrc"
export NODE_REPL_HISTORY="$XDG_STATE_HOME/node_repl_history"

# --- Python ---
export PYTHONSTARTUP="$XDG_CONFIG_HOME/python/pythonrc"
export PIP_CONFIG_FILE="$XDG_CONFIG_HOME/pip/pip.conf"

# --- Less ---
export LESSHISTFILE="$XDG_STATE_HOME/less/history"

# --- wget ---
export WGETRC="$XDG_CONFIG_HOME/wget/wgetrc"

# --- CUDA ---
export CUDA_CACHE_PATH="$XDG_CACHE_HOME/nv"
```

将这些 `export` 写入 shell 配置文件(`~/.config/fish/config.fish` 或 `~/.config/zsh/.zshrc` 等)。

## 7. 应用兼容性速查

### 7.1 原生支持 XDG 规范的应用

这些应用**开箱即用**遵守规范,无需额外配置:

| 应用 | 配置 | 数据 | 缓存 | 状态 |
|------|------|------|------|------|
| **Neovim** | `~/.config/nvim/` | `~/.local/share/nvim/` | `~/.cache/nvim/` | `~/.local/state/nvim/` |
| **fish shell** | `~/.config/fish/` | `~/.local/share/fish/` | — | — |
| **Helix** | `~/.config/helix/` | `~/.local/share/helix/` | `~/.cache/helix/` | — |
| **mpv** | `~/.config/mpv/` | — | — | `~/.local/state/mpv/` |
| **alacritty** | `~/.config/alacritty/` | — | — | — |
| **kitty** | `~/.config/kitty/` | — | `~/.cache/kitty/` | — |
| **i3 / Sway** | `~/.config/i3/` 或 `~/.config/sway/` | — | — | — |
| **waybar** | `~/.config/waybar/` | — | — | — |
| **direnv** | `~/.config/direnv/` | `~/.local/share/direnv/` | — | — |
| **fd** | — | — | — | —(纯 CLI 工具,无持久状态) |
| **bat** | `~/.config/bat/` | — | `~/.cache/bat/` | — |
| **ripgrep** | `~/.config/ripgrep/` | — | — | — |
| **starship** | `~/.config/starship.toml` | — | — | — |

### 7.2 需要手动设置的应用

通过环境变量引导到规范路径:

```bash
# 写入 shell 配置文件
export GNUPGHOME="$XDG_DATA_HOME/gnupg"
export DOCKER_CONFIG="$XDG_CONFIG_HOME/docker"
export CARGO_HOME="$XDG_DATA_HOME/cargo"
export RUSTUP_HOME="$XDG_DATA_HOME/rustup"
export GRADLE_USER_HOME="$XDG_DATA_HOME/gradle"
export ANDROID_USER_HOME="$XDG_DATA_HOME/android"
export WINEPREFIX="$XDG_DATA_HOME/wine"
export NUGET_PACKAGES="$XDG_CACHE_HOME/NuGetPackages"
export CUDA_CACHE_PATH="$XDG_CACHE_HOME/nv"
export _JAVA_OPTIONS="-Djava.util.prefs.userRoot=$XDG_CONFIG_HOME/java"
```

### 7.3 顽固分子(无法迁移)

| 应用 | 硬编码路径 | 原因 |
|------|-----------|------|
| **OpenSSH** | `~/.ssh/` | 安全关键路径,多工具硬编码依赖 |
| **Bash** | `~/.bashrc`, `~/.bash_history` | POSIX shell 兼容性,不能要求 XDG |
| **Zsh** | `~/.zshrc`, `~/.zsh_history` | 历史惯例,`ZDOTDIR` 可改配置位置但不能改历史 |
| **Firefox** | `~/.mozilla/` | 无环境变量覆盖,代码中硬编码 |
| **Thunderbird** | `~/.thunderbird/` | 同 Firefox 代码库 |

## 8. 实用技巧

### 8.1 备份策略

利用 XDG 的目录分离原则制定**最小备份集**:

```bash
# 只需要备份这两个目录即可恢复绝大部分应用状态
rsync -avP \
  ~/.config/ \
  ~/.local/share/ \
  /backup/location/

# 可选:状态文件(shell 历史等)
# rsync -avP ~/.local/state/ /backup/state/
```

**不需要备份的:**
- `~/.cache/` — 会自动重建
- `$XDG_RUNTIME_DIR` — 重启就消失
- `~/.local/share/Trash/` — 垃圾桶

### 8.2 清理与空间回收

```bash
# 查看各目录占用
du -sh ~/.cache ~/.config ~/.local/share ~/.local/state 2>/dev/null

# 安全清理所有缓存
rm -rf ~/.cache/*

# 针对性清理大缓存
rm -rf ~/.cache/yay/*       # AUR 构建缓存(动辄数 GB)
rm -rf ~/.cache/pip/*        # pip 下载缓存
rm -rf ~/.cache/thumbnails/* # 缩略图缓存

# 查看系统中最大的 10 个用户缓存目录
du -sh ~/.cache/*/ 2>/dev/null | sort -rh | head -10
```

### 8.3 迁移到新系统

```bash
# 在旧系统上
cd ~
tar -caf xdg-backup.tar.zst .config .local/share .local/state

# 传输到新系统后
tar -xaf xdg-backup.tar.zst -C ~/
```

### 8.4 诊断应用实际使用的路径

```bash
# strace 追踪应用访问了哪些文件
strace -e trace=open,openat,stat -f <应用程序命令> 2>&1 | grep "$HOME"

# 示例:查看 firefox 访问了哪些点文件
strace -e trace=openat -f firefox 2>&1 | grep -oP '"/home/[^"]+' | sort -u
```

### 8.5 检查系统 XDG 一致性

```bash
# 列出 ~ 下一级所有 dotfile(不包括 . 和 ..)
comm -13 \
  <(ls -d ~/.config ~/.cache ~/.local ~/.ssh ~/.gnupg ~/.mozilla 2>/dev/null | sort) \
  <(ls -d ~/.*/ 2>/dev/null | grep -v '/\.$\|/\.\.$' | sort)

# 这会显示所有"不守规矩"的 dotfile 目录
```

## 9. 常见问题

### Q: 不设置环境变量,应用怎么知道默认路径?

规范要求:如果环境变量未设置或为空,应用必须使用默认值。即:

```c
// 规范要求的伪代码逻辑
const char* config_home = getenv("XDG_CONFIG_HOME");
if (!config_home || config_home[0] == '\0') {
    config_home = "$HOME/.config";  // fallback
}
```

绝大多数应用和库(GLib、Qt 等)都内置了这个逻辑。

### Q: `~/.config` 和 `XDG_CONFIG_HOME` 的区别?

- `~/.config` 是**默认值**,当 `XDG_CONFIG_HOME` 未设置时使用
- `XDG_CONFIG_HOME` 是环境变量,可以覆盖默认路径
- 应用应读取环境变量,而非硬编码 `~/.config`

简言之:`~/.config` 是起点,`$XDG_CONFIG_HOME` 是方向盘。

### Q: `~/.local/share` 和 `/usr/share` 的关系?

两者遵循同一套目录结构规范(如 `applications/`、`icons/`、`fonts/` 等子目录),只是**层级不同**:

| 层级 | 路径 | 谁管理 |
|------|------|--------|
| 用户 | `~/.local/share/` | 用户手动或用户级工具 |
| 系统(管理员) | `/usr/local/share/` | 管理员手动安装 |
| 系统(包管理器) | `/usr/share/` | pacman / apt 管理 |

应用通过 `XDG_DATA_DIRS` 从高到低查找。

### Q: Mac 和 Windows 上有类似规范吗?

- **macOS:** 有类似设计——`~/Library/Preferences/`(配置)、`~/Library/Caches/`(缓存)、`~/Library/Application Support/`(数据),但这是 Apple 自己的规范,不基于 XDG
- **Windows:** `%APPDATA%`(配置/数据混合)、`%LOCALAPPDATA%`(本地数据/缓存混合),由 Microsoft 定义,同样不基于 XDG

跨平台应用通常会在代码中根据操作系统选择不同路径。

### Q: 如何让不守规范的应用 "搬家"?

三种策略按推荐程度排列:

1. **查文档找环境变量** — 很多应用提供了 `*_HOME` 类的变量(如 `GNUPGHOME`、`CARGO_HOME`),设置后即可迁移
2. **符号链接** — `ln -s ~/.local/share/gnupg ~/.gnupg`,对硬编码路径有效的 hack
3. **接受现实** — 对 SSH、Bash 等基础设施工具,不值得折腾

### Q: `~/.local/state` 为什么要单独存在?

历史的答案:`~/.local/share` 承担了太多职责。用户安装的字体和 shell 历史被放在同一棵树下,备份时要么全盘复制要么都舍弃。`XDG_STATE_HOME`(2021 年规范 0.8 版)把这些"丢了也无伤大雅但丢了又有点烦"的状态数据单独分离出来,让备份策略更加精细。

### Q: Arch Linux 有什么特别的 XDG 相关包和工具吗?

```bash
# xdg-user-dirs — 管理 文档/下载 等用户目录
pacman -Qi xdg-user-dirs 2>/dev/null | head -5

# xdg-utils — xdg-open 等命令行工具
pacman -Qi xdg-utils 2>/dev/null | head -5

# 在线规范文档
# https://specifications.freedesktop.org/basedir-spec/latest/
```

> **注意:** XDG Base Directory 的 man page 在 Arch 上由 `extra/xorg-docs` 提供(`man 7 XDG`),但这并非默认安装的包。

## 参考

- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)
- [Arch Wiki — XDG Base Directory](https://wiki.archlinux.org/title/XDG_Base_Directory)
- [Arch Wiki — XDG user directories](https://wiki.archlinux.org/title/XDG_user_directories)
- [GLib 开发者文档 — XDG 路径函数](https://docs.gtk.org/glib/func.get_user_config_dir.html)
