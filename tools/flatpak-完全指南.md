# Flatpak 完全指南

## 1. 什么是 Flatpak？

**Flatpak** 是一个 Linux 桌面应用的沙盒化分发与运行框架，由 freedesktop.org 社区维护。它的核心目标只有两个：

> **跨发行版通用** — 应用开发者打包一次，所有发行版都能跑；**安全隔离** — 每个应用运行在独立沙箱中，默认无法触碰宿主系统和用户文件。

### 1.1 定位：不是替代品，是互补

Flatpak **不替代** pacman/apt/dnf，而是和它们分工协作：

| 维度 | pacman（系统包管理） | Flatpak |
|------|---------------------|---------|
| **管理范围** | 内核、驱动、系统库、CLI 工具 | 桌面 GUI 应用 |
| **依赖方式** | 链接系统共享库 | 自带 Runtime，不碰系统库 |
| **更新模型** | 跟随发行版节奏 | 应用开发者直接推送 |
| **安全隔离** | 无（应用可访问整个系统） | 沙箱 + 按需授权 |
| **典型场景** | `base-devel`、`htop`、`git`、`python` | Firefox、LibreOffice、GIMP、Steam |

**一句话：** 系统底层用 pacman，桌面应用尽量用 Flatpak。

### 1.2 核心概念

```
┌─────────────────────────────┐
│   Flatpak 应用               │  ← 如 org.mozilla.firefox
├─────────────────────────────┤
│   Runtime（运行时）          │  ← 稳定基础库（glibc、GTK、Qt...）
├─────────────────────────────┤
│   Flatpak 框架              │  ← Bubblewrap 沙箱 / Portal 通道
├─────────────────────────────┤
│   宿主 Linux 系统            │  ← 内核、驱动、Wayland/PulseAudio
└─────────────────────────────┘
```

| 概念 | 说明 | 类比 |
|------|------|------|
| **Runtime** | 应用依赖的基础运行环境，多个应用可共享同一个 Runtime | 相当于 "应用的操作系统层" |
| **SDK** | Runtime + 编译工具链，仅构建时需要 | 相当于 `base-devel` |
| **Remote** | 分发 Flatpak 的远程仓库 | 相当于 pacman 镜像源 |
| **Portal** | 沙箱与宿主之间的受控通道（打开文件、截屏等） | 相当于手机 App 权限弹窗 |
| **Bubblewrap** | Flatpak 底层沙箱引擎，利用内核 namespace 隔离 | 轻量容器，无需 Docker 依赖 |

### 1.3 Flatpak vs Snap vs AppImage

| 特性 | Flatpak | Snap | AppImage |
|------|---------|------|----------|
| **沙箱安全** | ✅ Bubblewrap 强制隔离 | ✅ AppArmor | ❌ 无 |
| **自动更新** | ✅ 可选 | ✅ 强制 | ❌ 手动替换 |
| **去中心化** | ✅ 多仓库 | ❌ 仅 Snap Store | ✅ 单文件 |
| **运行时共享** | ✅ 共用 Runtime，节省空间 | ❌ 各自打包 core | ❌ 各自打包 |
| **桌面集成** | ✅ 自动 .desktop + 图标 | ✅ 良好 | ⚠️ 需手动 |
| **启动速度** | 中等（首次较慢） | 较慢 | 快 |
| **后端闭源** | ❌ 完全开源 | ⚠️ 服务端闭源 | ❌ 完全开源 |

---

## 2. 安装

### 2.1 安装 Flatpak 核心

```bash
# 安装 Flatpak 框架
sudo pacman -S flatpak

# 推荐一并安装
sudo pacman -S xdg-desktop-portal          # Portal 标准（文件选择器、截屏等）
sudo pacman -S xdg-desktop-portal-kde      # KDE Plasma 下的 Portal 后端
# 或
sudo pacman -S xdg-desktop-portal-gtk      # GNOME/GTK 环境下的 Portal 后端
sudo pacman -S flatpak-kcm                  # KDE 系统设置中的 Flatpak 管理面板（可选）
```

| 包名 | 作用 | 是否必要 |
|------|------|----------|
| `flatpak` | Flatpak 核心框架 | **必须** |
| `xdg-desktop-portal` | 提供文件选择器、截屏、屏幕共享等受控通道 | **强烈推荐** |
| `xdg-desktop-portal-kde` | KDE/Qt 应用对应的 Portal 后端 | KDE 用户推荐 |
| `xdg-desktop-portal-gtk` | GTK 应用对应的 Portal 后端 | GTK 桌面用户推荐 |
| `flatpak-kcm` | 在 KDE 系统设置中嵌入 Flatpak 权限管理 | 可选 |

> **注意：** KDE 和 GTK 的 Portal 后端可以同时安装，系统会自动为不同应用选择正确的后端。不装 Portal 会导致部分应用无法弹出文件选择对话框。

### 2.2 添加 Flathub 仓库

Flathub（[https://flathub.org](https://flathub.org)）是 Flatpak 生态中最主流的社区仓库，截至 2026 年已有 2500+ 应用：

```bash
# 添加 Flathub 远程仓库
flatpak remote-add --if-not-exists flathub \
  https://dl.flathub.org/repo/flathub.flatpakrepo
```

`--if-not-exists` 确保不会重复添加。完成后用 `flatpak remotes` 验证：

```bash
flatpak remotes
# 输出示例：
# Name     Options
# flathub  system
```

---

## 3. 基本使用

### 3.1 搜索

```bash
# 按名称搜索
flatpak search firefox

# 查看字段含义
flatpak search --columns=name,description,application,version firefox
```

Application ID 是 Flatpak 应用的唯一标识，采用反向域名格式（如 `org.mozilla.firefox`、`com.spotify.Client`）。

### 3.2 安装

```bash
# 标准安装（系统级，需 sudo）
flatpak install flathub org.mozilla.firefox

# 用户级安装（无需 sudo，安装到 ~/.local/share/flatpak）
flatpak install --user flathub org.mozilla.firefox

# 从本地 .flatpakref 安装
flatpak install /path/to/app.flatpakref
```

| 模式 | 安装位置 | 需要 sudo | 多用户共享 | 推荐场景 |
|------|----------|-----------|------------|----------|
| `--system`（默认） | `/var/lib/flatpak` | 是 | ✅ | 多用户系统 |
| `--user` | `~/.local/share/flatpak` | 否 | ❌ | 个人电脑 |

### 3.3 运行

安装后 Flatpak 应用自动在系统菜单中注册，无需命令行启动。如需从终端启动：

```bash
flatpak run org.mozilla.firefox
```

### 3.4 管理与更新

```bash
# 列出已安装
flatpak list                     # 全部
flatpak list --app               # 仅应用
flatpak list                  --columns=name,size,origin  # 查看占用

# 更新
flatpak update                   # 更新全部
flatpak update org.mozilla.firefox  # 更新指定应用

# 卸载
flatpak uninstall org.mozilla.firefox
flatpak uninstall --delete-data org.mozilla.firefox  # 同时删除用户数据

# 清理不再被任何应用使用的 Runtime
flatpak uninstall --unused
```

> **重要：** 卸载应用后其 Runtime 不会自动删除。定期运行 `flatpak uninstall --unused` 可以回收不少空间。

---

## 4. 仓库管理

### 4.1 基本操作

```bash
# 列出所有远程仓库
flatpak remotes -d     # -d 显示 URL

# 添加仓库
flatpak remote-add --if-not-exists <名称> <URL>

# 从 .flatpakrepo 文件添加
flatpak remote-add --from <名称> /path/to/repo.flatpakrepo

# 删除仓库
flatpak remote-delete <名称>

# 查看仓库中的应用信息
flatpak remote-info flathub org.gimp.GIMP

# 查看仓库中所有可用更新
flatpak remote-ls --updates flathub
```

### 4.2 常用仓库

| 仓库 | 说明 | Runtime 模式 |
|------|------|-------------|
| **Flathub** | 最大社区仓库 | `flatpak remote-add flathub https://dl.flathub.org/repo/flathub.flatpakrepo` |
| **Flathub Beta** | 测试版分支 | `flatpak remote-add flathub-beta https://flathub.org/beta-repo/flathub-beta.flatpakrepo` |
| **GNOME Nightly** | GNOME 每夜构建 | `flatpak remote-add gnome-nightly https://nightly.gnome.org/gnome-nightly.flatpakrepo` |
| **KDE Nightly** | KDE 每夜构建 | `flatpak remote-add kdeapps https://distribute.kde.org/kdeapps.flatpakrepo` |

---

## 5. 沙箱与权限

### 5.1 设计理念

Flatpak 的权限模型遵循**最小权限原则**：
- 应用默认看不到宿主文件系统、网络受限、无法访问其他应用数据
- 需要权限时通过 **Portal** 受控申请，用户可同意或拒绝
- 行为类似手机 App 的权限系统

### 5.2 查看应用权限

```bash
# 查看完整元数据（含权限列表）
flatpak info -m org.mozilla.firefox

# 仅查看权限策略
flatpak info --show-permissions org.mozilla.firefox
```

### 5.3 常见权限项解读

| 权限 | 作用 | 风险等级 |
|------|------|----------|
| `--socket=wayland` | 访问 Wayland 显示 | ✅ 正常 |
| `--socket=x11` | 访问 X11 显示 | ⚠️ 可监听其他应用输入 |
| `--socket=fallback-x11` | 仅在无 Wayland 时降级到 X11 | ✅ 推荐 |
| `--share=network` | 访问网络 | 取决于应用性质 |
| `--device=dri` | GPU 直接渲染 | ⚠️ 可访问显存 |
| `--filesystem=host` | 访问整个文件系统 | 🔴 等效于无沙箱 |
| `--filesystem=home` | 访问整个家目录 | 🔴 高风险 |
| `--filesystem=xdg-download` | 仅访问下载目录 | ✅ 正常 |
| `--filesystem=xdg-pictures:ro` | 仅读取图片目录 | ✅ 合理 |
| `--socket=pulseaudio` | 访问音频 | ✅ 正常 |
| `--talk-name=org.freedesktop.secrets` | 访问密钥环 | ⚠️ 可读取密码 |

> **经验法则：** 如果一个应用申请了 `--filesystem=host` 或 `--filesystem=home` 而你对此有疑虑，可以去 Flathub 官方页面查看维护者的说明，或使用 Flatseal 收紧权限。

### 5.4 Flatseal：图形化权限管理（推荐）

Flatseal 让你不碰命令行就能管理所有 Flatpak 应用的权限：

```bash
flatpak install flathub com.github.tchx84.Flatseal
```

界面左栏是应用列表，右栏是该应用的权限开关（文件系统、网络、设备等），修改即时生效。

### 5.5 命令行覆写权限

```bash
# 授予额外目录访问
flatpak override --user --filesystem=/path/to/projects org.example.App

# 禁止网络访问（仅限用户级）
flatpak override --user --unshare=network org.example.App

# 设置环境变量
flatpak override --user --env=GTK_THEME=Arc-Dark org.example.App

# 查看当前覆写
flatpak override --user --show org.example.App

# 重置该应用的所有覆写
flatpak override --user --reset org.example.App
```

| 选项 | 含义 |
|------|------|
| `--user` | 仅对当前用户生效（推荐，无需 sudo） |
| `--system` | 全局生效（需 sudo） |
| `--filesystem=<路径>` | 允许访问指定路径 |
| `--nofilesystem=<路径>` | 禁止访问指定路径 |
| `--share=<子系统>` | 授予权限 |
| `--unshare=<子系统>` | 撤销权限 |
| `--env=<KEY>=<VAL>` | 设置环境变量 |

---

## 6. 主题与外观统一

Flatpak 应用默认使用 Runtime 自带主题，可能与系统桌面风格不一致。

### 6.1 GTK 主题

```bash
# 搜索 Flatpak 可用的 GTK 主题
flatpak search Gtk3theme

# 安装与系统匹配的主题
flatpak install flathub org.gtk.Gtk3theme.Arc-Dark

# 查看已安装的主题扩展
flatpak list --columns=name,application | grep Gtk3theme
```

> **原理：** Flatpak 主题以 extension 形式分发。安装了 `org.gtk.Gtk3theme.xxx` 后，所有使用该 Runtime 的应用自动匹配。

### 6.2 KDE/Qt 主题

```bash
# 安装 KDE 样式扩展
flatpak install flathub org.kde.KStyle.Adwaita

# 安装 QGnomePlatform（让 Qt 应用适配 GTK 桌面）
flatpak install flathub org.kde.PlatformTheme.QGnomePlatform
```

### 6.3 手动方案

如果 Flatpak 版主题不可用，可通过覆写让应用直接读取系统的主题文件：

```bash
# 让应用只读访问系统 GTK 配置
flatpak override --user --filesystem=xdg-config/gtk-3.0:ro

# 指定主题名称
flatpak override --user --env=GTK_THEME=Arc-Dark
```

---

## 7. 性能优化

### 7.1 国内镜像加速

Flathub 官方服务器在海外，国内用户可切换镜像源：

```bash
# 上海交通大学镜像（推荐）
flatpak remote-modify flathub --url=https://mirror.sjtu.edu.cn/flathub

# 南京大学镜像
flatpak remote-modify flathub --url=https://mirror.nju.edu.cn/flathub

# 中国科学技术大学镜像
flatpak remote-modify flathub --url=https://mirrors.ustc.edu.cn/flathub

# 恢复官方源
flatpak remote-modify flathub --url=https://dl.flathub.org/repo/
```

> **验证：** 切换后运行 `flatpak update` 测试下载速度。若镜像更新延迟较大，切回官方源。

### 7.2 磁盘空间管理

```bash
# 查看各应用 / Runtime 占用
flatpak list --columns=name,size,application,branch

# 清理未使用的 Runtime（最有效的释放手段）
flatpak uninstall --unused

# 查看 Flatpak 安装占用总量
du -sh /var/lib/flatpak ~/.local/share/flatpak 2>/dev/null

# 修复本地仓库（去重、清理）
sudo flatpak repair
```

### 7.3 系统级 vs 用户级，单用户场景选哪个？

如果你是单用户电脑，`--user` 更省事（无需 sudo），但 `--system` 没有实质性性能差距。选哪个主要看习惯——两种安装方式的应用启动性能一致。

---

## 8. 应用数据管理

### 8.1 数据存放位置

每个 Flatpak 应用的用户数据独立存放在：

```
~/.var/app/<Application-ID>/
├── config/   ← 配置文件
├── data/     ← 应用数据（书签、存档等）
└── cache/    ← 缓存
```

```bash
# 查看各应用数据占用
du -sh ~/.var/app/*/

# 备份某个应用的数据
cp -r ~/.var/app/org.mozilla.firefox ~/backups/firefox-flatpak-data
```

### 8.2 重置应用

```bash
# 方法一：删除数据目录
rm -rf ~/.var/app/org.mozilla.firefox

# 方法二：卸载时同时删除数据
flatpak uninstall --delete-data org.mozilla.firefox
```

---

## 9. 常见问题

### Q1: Flatpak 应用中文显示为方块？

**原因：** Flatpak Runtime 不自带中文字体。

**解决：**
```bash
# 确保系统安装了中文字体
sudo pacman -S noto-fonts-cjk adobe-source-han-sans-cn-fonts

# 如仍未解决，让应用只读访问系统字体目录
flatpak override --user --filesystem=/usr/share/fonts:ro <Application-ID>
```

### Q2: Flatpak 应用无法使用 fcitx5/ibus 输入法？

**原因：** 输入法框架需要通过环境变量和 socket 与沙箱通信。

**解决（以 fcitx5 为例）：**
```bash
# 安装 fcitx5 Portal
flatpak install flathub org.fcitx.Fcitx5

# 为应用设置输入法环境变量
flatpak override --user \
  --env=GTK_IM_MODULE=fcitx \
  --env=QT_IM_MODULE=fcitx \
  --env=XMODIFIERS=@im=fcitx \
  org.mozilla.firefox
```

### Q3: 应用报告 "No such ref" 错误？

**原因：** Application ID 写错、仓库未添加、或仓库元数据过期。

**排查：**
```bash
# 1. 确认 Application ID 是否准确
flatpak search <关键搜索词>

# 2. 确认远程仓库存在
flatpak remotes

# 3. 更新仓库元数据
flatpak update --appstream
```

### Q4: `flatpak update` 速度太慢？

**解决：**
```bash
# 切换到国内镜像（见 7.1 节）
flatpak remote-modify flathub --url=https://mirror.sjtu.edu.cn/flathub

# 分应用更新，不一次更新全部
flatpak update org.mozilla.firefox
flatpak update com.spotify.Client
```

### Q5: 同时装了 Flatpak 版和系统版的应用，会冲突吗？

**答：** 不会。两者安装路径完全不同，运行时互不干扰。唯一的"冲突"是：两个版本都会注册 `.desktop` 菜单项，同名的可能互相覆盖，通常以后安装的为准。可在桌面环境的启动器中查看详情来区分。

### Q6: Flatpak 能不能装 CLI 工具？

**答：** 理论上可以，但不推荐。Flatpak 的沙箱设计会严重阻碍 CLI 工具访问文件、进程和系统资源。CLI 工具建议用 `pacman`、`pipx`、`cargo install` 等原生方式安装。

### Q7: 卸载所有 Flatpak 后残留怎么处理？

```bash
# 列出并卸载全部应用
flatpak list --app --columns=application | tail -n +1 | xargs -r flatpak uninstall -y

# 卸载全部 Runtime
flatpak list --runtime --columns=application | tail -n +1 | xargs -r flatpak uninstall -y

# 删除用户数据
rm -rf ~/.var/app/

# 卸载 Flatpak 自身
sudo pacman -Rs flatpak
```

---

## 10. 常用扁平化推荐

以下是 Arch Linux 上推荐以 Flatpak 方式安装的常用应用：

| 分类 | 应用 | Application ID |
|------|------|----------------|
| **浏览器** | Firefox | `org.mozilla.firefox` |
| **浏览器** | Chromium | `org.chromium.Chromium` |
| **办公** | LibreOffice | `org.libreoffice.LibreOffice` |
| **图像** | GIMP | `org.gimp.GIMP` |
| **图像** | Inkscape | `org.inkscape.Inkscape` |
| **媒体** | VLC | `org.videolan.VLC` |
| **音乐** | Spotify | `com.spotify.Client` |
| **通讯** | Slack | `com.slack.Slack` |
| **通讯** | Discord | `com.discordapp.Discord` |
| **开发** | VS Code | `com.visualstudio.code` |
| **游戏** | Steam | `com.valvesoftware.Steam` |
| **游戏** | Heroic Games Launcher | `com.heroicgameslauncher.hgl` |
| **工具** | Flatseal | `com.github.tchx84.Flatseal` |
| **工具** | Bottles（Wine 管理） | `com.usebottles.bottles` |
