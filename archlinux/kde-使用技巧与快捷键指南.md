# KDE Plasma 使用技巧与快捷键完全指南

## 1. KDE Plasma 设计哲学

KDE Plasma 的核心设计理念是：**默认简洁，按需强大**。几乎每一个默认行为都可以在系统设置里调整，每一个面板组件都可以替换。你可以把它配置得像 Windows、像 macOS、或者完全是你自己的形状。

---

## 2. 窗口管理快捷键（日常最高频）

### 2.1 窗口操作

| 快捷键 | 功能 |
|--------|------|
| `Alt + F3` | 窗口操作菜单（最小化/最大化/移动到桌面/置顶等） |
| `Alt + Space` | KRunner 启动器（见下文） |
| `Alt + 鼠标左键拖拽` | **任意位置拖拽窗口**（不必抓标题栏） |
| `Alt + 鼠标右键拖拽` | 任意位置调整窗口大小 |
| `Alt + 鼠标滚轮` | 在窗口上滚动 = 调整透明度 |
| `Meta + PgUp/PgDn` | 窗口切换到上/下一个虚拟桌面 |
| `Meta + Shift + PgUp/PgDn` | 把当前窗口**移动**到上/下一个虚拟桌面 |
| `Meta + 方向键` | 窗口贴靠（左半屏/右半屏/最大化/最小化） |
| `F11` | 窗口全屏（大部分应用通用） |

### 2.2 虚拟桌面

| 快捷键 | 功能 |
|--------|------|
| `Meta + Ctrl + F1~F4` | 切换到对应虚拟桌面 |
| `Meta + Ctrl + 方向键` | 在虚拟桌面网格中移动 |
| `Ctrl + F1~F4` | 切换到对应虚拟桌面（不带 Meta） |

### 2.3 平铺技巧

KDE 自带基础平铺，不需要 i3：

| 快捷键 | 功能 |
|--------|------|
| `Meta + ←` / `Meta + →` | 半屏贴靠 |
| `Meta + ↑` | 最大化 |
| `Meta + ↓` | 还原/最小化 |
| `Meta + Home` | 快速平铺（Quarter Tiling，四分之一屏） |

按住 `Meta` 时拖动窗口，会显示网格，松手即贴靠到对应格子。

---

## 3. KRunner —— KDE 的万能启动器

> **这是 KDE 最被低估的功能，没有之一。**

按 `Alt + Space`（或 `Alt + F2`）呼出。

### 3.1 你可以在里面做的事

| 输入 | 功能 |
|------|------|
| 应用名（如 `firefox`） | 启动应用 |
| `= 2+3*4` | 计算器（输入 `=` 开头） |
| `kill <进程名>` | 强制结束进程 |
| `systemctl restart xxx` | systemd 服务控制 |
| `wp <关键词>` | 搜索 Wikipedia |
| `dd`、`define`、`spell` | 词典、拼写检查 |
| `file:///path` | 文件浏览 |
| `ssh user@host` | 快速 SSH |
| 文件名 | 直接搜索文件并打开 |
| 网址（如 `archlinux.org`） | 直接在浏览器打开 |

### 3.2 KRunner 插件

系统设置 → 搜索 → KRunner → 勾选/排序你需要的插件。

**推荐开启：**

| 插件 | 用途 |
|------|------|
| 应用程序 | 启动应用 |
| 命令行 | 直接运行命令 |
| 文件搜索 | 搜索文件并打开 |
| 计算器 | `=` 开头即为计算 |
| systemd 单元 | systemctl 控制 |
| 窗口列表 | 切换已打开窗口 |
| 书签 | 浏览器书签直接搜索 |
| 拼写检查 | 文本拼写纠正 |

---

## 4. Dolphin 文件管理器技巧

### 4.1 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl + T` | 在新标签页打开 |
| `Ctrl + Shift + T` | 恢复刚关闭的标签页 |
| `F3` | 分屏模式（左右两栏） |
| `Ctrl + Shift + N` | 新建文件夹 |
| `F2` | 重命名 |
| `Shift + F4` | 在此处打开终端 |
| `F4` | 打开/关闭内嵌终端面板 |
| `Ctrl + L` | 切换到地址栏编辑模式 |
| `Ctrl + D` | 添加到"位置"侧边栏 |
| `Ctrl + .` | 显示/隐藏隐藏文件 |
| `Ctrl + 1/2/3` | 切换视图（图标/紧凑/详情） |
| `Ctrl + Shift + P` | 打开文件属性对话框 |
| `F11` | 信息面板（右侧预览/元数据） |
| `Ctrl + Shift + A` | 显示/隐藏侧边栏 |

### 4.2 右键菜单增强（Service Menu）

```bash
# 安装社区服务菜单
sudo pacman -S kde-servicemenu-rootactions  # 右键以 root 操作文件
```

服务菜单存放位置：
- 系统级：`/usr/share/kio/servicemenus/`
- 用户级：`~/.local/share/kio/servicemenus/`

一个简单的自定义服务菜单示例——为 `.tar.zst` 文件添加右键解压：

```bash
# 创建 ~/.local/share/kio/servicemenus/extract-tar-zst.desktop
cat > ~/.local/share/kio/servicemenus/extract-tar-zst.desktop << 'EOF'
[Desktop Entry]
Type=Service
ServiceTypes=KonqPopupMenu/Plugin
MimeType=application/zstd;application/x-tarzst
Actions=extractHere
X-KDE-Priority=TopLevel

[Desktop Action extractHere]
Name=此处解压 (.tar.zst)
Icon=utilities-file-archiver
Exec=tar -xaf "%f" -C "%d"
EOF
```

### 4.3 批量重命名

选中多个文件 → `F2` → 弹出批量重命名对话框，支持：
- 查找替换
- 添加前缀/后缀
- 编号序列（`###` 格式）
- 正则表达式替换

---

## 5. 虚拟桌面与活动（Activities）

### 5.1 虚拟桌面 vs 活动

KDE 有两个容易混淆的概念：

| | 虚拟桌面 (Virtual Desktop) | 活动 (Activity) |
|--|---------------------------|-----------------|
| 窗口 | 不同桌面有不同窗口 | 不同活动可以**完全不同** |
| 壁纸 | 统一（或每个桌面不同） | 每个活动独立设置 |
| 面板 | 共享 | 每个活动可以不同面板 |
| 文件夹视图 | 统一 | 不同活动可以关联不同文件夹 |
| 快捷键 | `Meta + Ctrl + ←→` | `Meta + Tab` |
| 适用场景 | 多任务分组（工作/聊天/浏览） | 完全切换工作模式（开发/设计/日常） |

> **日常建议：** 虚拟桌面就够了。活动适合「上班 vs 个人」这种需要完全切换上下文（包括壁纸、面板、文件）的场景。

### 5.2 虚拟桌面设置建议

系统设置 → 工作区行为 → 虚拟桌面：
- **行数：1**（水平排列，滚轮切换最直观）
- 桌面数量看需求，一般 4 个足够
- 启用「鼠标滚轮切换桌面」（在桌面空白处滚动）
- 启用「窗口在桌面之间跟随」

---

## 6. KWin 窗口管理器进阶

### 6.1 桌面特效

系统设置 → 工作区行为 → 桌面特效：

**必开推荐：**

| 特效 | 功能 |
|------|------|
| 窗口抖动 | 拖动窗口时有弹性效果（视觉反馈） |
| 半透明窗口 | 移动/调整窗口时半透明，看到下方内容 |
| 魔灯 | 窗口最小化/还原时渐隐，不突兀 |
| 桌面立方体 | 3D 立方体切换桌面（炫酷，不实用但好玩） |
| 窗口呈现 | `Ctrl + F9` 展开所有窗口，`Ctrl + F10` 展开当前桌面窗口 |
| 缩放 | `Meta + =` 放大，`Meta + -` 缩小，`Meta + 0` 还原 |

### 6.2 窗口规则（KWin Rules）

KDE 的隐藏宝藏，针对特定窗口**永久设定行为**。

系统设置 → 窗口管理 → 窗口规则 → 新建：

```
示例规则：让某个应用始终在指定桌面打开
┌──────────────────────────────────────┐
│ 窗口匹配 → 窗口类 → "应用窗口类名"    │
│ 大小和位置 → 桌面 → 强制 → 桌面 2     │
│ 行为 → 置顶 → 强制 → 是              │
└──────────────────────────────────────┘
```

常用规则场景：
- 让某个应用始终在指定桌面打开
- 让某个应用始终置顶
- 让某个应用跳过任务栏
- 禁止某个应用接收焦点（Steam 好友列表等）
- 关闭窗口时最小化到托盘（如 Discord）

### 6.3 快捷切窗方式

除了 `Alt + Tab`，还有：

| 快捷键 | 功能 |
|--------|------|
| `Alt + Tab` | 标准窗口切换 |
| `Ctrl + F9` | 所有桌面的所有窗口平铺展示 |
| `Ctrl + F10` | 当前桌面窗口平铺展示 |
| `Meta + W` | Exposé 风格窗口概览 |
| `Ctrl + F7` | 当前应用的所有窗口（同应用窗口切换） |

---

## 7. 面板自定义

右键面板 → 进入编辑模式，KDE 的面板是**完全自由**的。

### 7.1 推荐面板组件

| 组件 | 用途 |
|------|------|
| 图标任务管理器 (Icons-only Task Manager) | 类似 Windows 7 的任务栏 |
| 系统托盘 | 后台应用图标 + 通知中心 |
| 数字时钟 | 显示日期时间，右键可配置时区和日历 |
| 全局菜单 | macOS 风格顶部菜单栏（需装 `appmenu-gtk-module`） |
| 虚拟桌面分页器 | 桌面切换 + 预览缩略图 |
| 颜色选择器 | 取色工具 |

### 7.2 面板位置与布局

常见布局方案：

```
方案一：底部单面板（Windows 风格）
└── 应用菜单 ── 任务栏 ────────── 系统托盘 ─ 时钟 ──┘

方案二：顶部 + 底部（macOS 风格）
┌── 应用菜单 ─ 全局菜单 ──── 系统托盘 ─ 时钟 ──┐ (顶部)

└────── 图标任务栏 ──────┘                     (底部 Dock)

方案三：左侧面板（Unity 风格）
┌────┐
│ 图 │
│ 标 │
│ 任 │
│ 务 │
│ 栏 │
└────┘
```

### 7.3 面板快捷键

| 操作 | 功能 |
|------|------|
| 右键面板空白处 | 编辑面板布局 |
| `Meta + 1~9` | 启动/聚焦任务栏第 N 个固定应用 |
| 鼠标滚轮在任务栏图标上 | 切换同一应用的多个窗口 |

---

## 8. Spectacle 截图工具

| 快捷键 | 功能 |
|--------|------|
| `PrtSc` | 全屏截图 |
| `Meta + PrtSc` | 当前活动窗口截图 |
| `Meta + Shift + PrtSc` | 矩形区域截图 |

Spectacle 支持：
- **延时截图**（定时器）
- **矩形/窗口/全屏/自定义区域**
- **截图后直接标注**（箭头、文字、模糊、高亮）
- **直接复制到剪贴板**或保存

自定义保存路径和文件名模式：
```
~/Pictures/Screenshots/Screenshot_%Y%m%d_%H%M%S.png
```

---

## 9. 系统设置宝藏功能

### 9.1 快捷唤醒（Accessibility）

系统设置 → 辅助功能 → 快捷唤醒：
- `Meta + M`：快速开关鼠标（键盘党福音）
- `Meta + 小键盘 +/-`：鼠标光标放大镜

### 9.2 自定义快捷键

系统设置 → 快捷键 → 自定义快捷键 → 新建 → 命令/URL：

你的脚本可以直接绑定到全局快捷键：

```
示例：
名称：清理下载文件夹
触发器：Meta + Shift + D
命令：python3 /home/l/scripts/clean_downloads.py
```

### 9.3 夜间护眼（Night Color）

系统设置 → 显示和监视器 → 夜间颜色：
- 日落后自动降低色温（减少蓝光）
- 可手动设定色温值（推荐 4000K~4500K）
- 支持自定义开启时间

### 9.4 触控板手势（Wayland）

如果使用 Wayland（Arch 上 KDE 默认已经是 Wayland）：

| 手势 | 功能 |
|------|------|
| 三指上滑 | 窗口概览 |
| 三指下滑 | 显示桌面 |
| 三指左/右滑 | 切换虚拟桌面 |
| 四指上滑 | 桌面网格 |
| 四指下滑 | 所有窗口平铺 |
| Ctrl + 双指缩放 | 屏幕缩放 |

---

## 10. 命令行快捷操作

```bash
# 从命令行打开系统设置子页面
systemsettings kcm_kwin_scripts       # KWin 脚本
systemsettings kcm_keys               # 快捷键
systemsettings kcm_nightcolor         # 夜间颜色
systemsettings kcm_kwinrules          # 窗口规则
systemsettings kcm_desktoppaths       # 桌面路径

# 列出所有可用的设置模块
kcmshell6 --list

# 重启 Plasma（崩溃后恢复，不影响运行中的应用）
plasmashell --replace &
# 或
systemctl --user restart plasma-plasmashell

# 重启 KWin（不影响面板和应用）
kwin_x11 --replace &            # X11
kwin_wayland --replace &        # Wayland

# 锁定屏幕
loginctl lock-session

# 切换虚拟桌面（脚本用）
qdbus org.kde.kglobalaccel /component/kwin invokeShortcut "Switch to Desktop 2"
```

---

## 11. 个人推荐配置清单

### 11.1 必改设置

**屏幕边缘热角：**
系统设置 → 工作区行为 → 屏幕边缘 → 把「左上角」设为「显示桌面」或「窗口概览」（类 macOS 热角）。

**桌面特效：**
系统设置 → 工作区行为 → 桌面特效 → 关闭「滑动弹出」（如果觉得通知动画太慢）。

**焦点策略：**
系统设置 → 工作区行为 → 窗口行为 → 焦点：
- 焦点跟随鼠标
- 焦点窃取防护：高

**Dolphin 配置：**
- 设置 → 配置 Dolphin → 启动 → 显示完整路径
- 设置 → 配置 Dolphin → 导航 → 双击打开文件/文件夹
- 在标题栏显示完整路径

**面板高度：**
右键面板 → 进入编辑模式 → 拖动调整高度 → 推荐 40~48px。

### 11.2 推荐安装的配套工具

```bash
# 系统监控（类似 Windows 任务管理器的高级版）
sudo pacman -S plasma-systemmonitor

# 下拉终端（F12 呼出）
sudo pacman -S yakuake

# GTK 应用在 KDE 下的统一外观
sudo pacman -S breeze-gtk
# 然后系统设置 → 外观 → 应用程序风格 → 配置 GTK 风格 → Breeze

# Qt 主题引擎（更细腻的主题定制）
sudo pacman -S kvantum
```

---

## 12. 常见问题

### Q1: 为什么 Chrome/VS Code 等 GTK 应用在 KDE 下看起来丑？

```bash
# 安装 Breeze GTK 主题
sudo pacman -S breeze-gtk

# 系统设置 → 外观 → 应用程序风格 → 配置 GNOME/GTK 应用程序风格
# 选择 Breeze → 应用

# 重启应用即可
```

### Q2: 窗口标题栏太大/太小？

系统设置 → 外观 → 窗口装饰 → 编辑 Breeze 主题 → 调整按钮大小与边框宽度。或直接换窗口装饰（下载社区预设）。

### Q3: KRunner 搜不到某个应用？

```bash
# 重建应用缓存
kbuildsycoca6

# 检查 .desktop 文件位置
ls ~/.local/share/applications/
ls /usr/share/applications/
```

### Q4: 桌面崩溃了怎么办？

```bash
# 重启 Plasma Shell（不影响已运行的应用）
plasmashell --replace &

# 如果整个会话卡死
# Ctrl + Alt + F2 切换到 tty2
# 登录后执行:
systemctl --user restart plasma-plasmashell
```

### Q5: 怎么备份我的所有 KDE 设置？

```bash
# KDE 配置全部在 ~/.config/ 下，以 k 或 plasma 开头的目录：
cp -r \
  ~/.config/kde* \
  ~/.config/plasma* \
  ~/.config/kwin* \
  ~/.config/krunner* \
  ~/.config/dolphin* \
  ~/.config/spectaclerc \
  ~/.local/share/plasma \
  ~/.local/share/kwin \
  /path/to/backup/

# 恢复时直接复制回来即可
```

---

## 参考

- [KDE UserBase Wiki](https://userbase.kde.org/)
- [KDE 快捷键完整列表](https://docs.kde.org/stable5/en/khelpcenter/fundamentals/shortcuts.html)
- [Arch Wiki — KDE](https://wiki.archlinux.org/title/KDE)
- [KDE 社区论坛](https://discuss.kde.org/)
