# Wine & winetricks 完全指南

## 1. 什么是 Wine？

**Wine**（**W**ine **I**s **N**ot an **E**mulator）是一个**兼容层**，让 Linux/macOS 系统能够直接运行 Windows 程序。

### 1.1 不是虚拟机，不是模拟器

Wine **不模拟** CPU、不创建虚拟硬件、不需要 Windows 许可证。它做的事只有一件：

> 把 Windows API 调用**实时翻译**成 Linux 系统调用

| 方案 | 原理 | 性能 | 需要 Windows 许可证 |
|------|------|------|---------------------|
| **Wine** | API 翻译层 | ≈ 原生（无损耗） | 不需要 |
| **虚拟机（VirtualBox/VMware）** | 模拟完整硬件 + 装完整 Windows | 有损耗 | 需要 |
| **双系统** | 原生运行 | 原生 | 需要 |

**一句话总结：** Wine 相当于在 Linux 上实现了一套 Windows API，程序以为自己跑在 Windows 上，实际上调用的是 Linux 内核。

### 1.2 Wine 的局限性

| 能做的 | 不能/做不好的 |
|--------|---------------|
| 运行大量 Windows 桌面软件 | 最新的 AAA 游戏（复杂反作弊/驱动依赖） |
| 办公软件、开发工具 | 需要硬件驱动的程序（打印机驱动等） |
| 中小型游戏 | 依赖未实现 API 的软件 |
| .NET Framework / VC++ 程序 | 内核级驱动（`.sys`） |

---

## 2. 安装

### 2.1 Arch Linux

```bash
# 安装 Wine（含 32 位支持）
sudo pacman -S wine

# 推荐一并安装的附属工具
sudo pacman -S wine-mono wine-gecko winetricks
```

| 包名 | 作用 |
|------|------|
| `wine` | Wine 主程序 |
| `wine-mono` | .NET Framework 的开源替代实现 |
| `wine-gecko` | IE/HTML 渲染引擎替代 |
| `winetricks` | Wine 辅助配置脚本（详见第 7 节） |

### 2.2 其他发行版

```bash
# Ubuntu/Debian
sudo apt install wine winetricks

# Fedora
sudo dnf install wine winetricks
```

---

## 3. 基本用法

### 3.1 运行 Windows 程序

```bash
# 运行 .exe 文件
wine /path/to/program.exe

# 运行安装程序
wine /path/to/setup.exe

# 指定工作目录
cd /path/to/game && wine game.exe
```

### 3.2 运行 MSI 安装包

```bash
wine msiexec /i /path/to/installer.msi
```

### 3.3 打开 Windows 程序关联的文件

```bash
# 用 Wine 中的记事本打开文本文件
wine notepad /path/to/file.txt

# 用 Wine 中的命令提示符执行命令
wine cmd /c "echo Hello from Wine"
```

### 3.4 查看已安装的程序

```bash
# 列出已安装程序
wine uninstaller --list
```

---

## 4. Wine 目录结构与文件映射

### 4.1 默认目录

Wine 默认在 `~/.wine/` 下创建模拟的 Windows 环境。首次运行 `wine` 或 `winecfg` 时会自动初始化。

```
~/.wine/
├── drive_c/           ← 模拟的 C 盘
│   ├── windows/       ← Windows 系统目录
│   │   ├── system32/  ← 64 位系统文件
│   │   ├── syswow64/  ← 32 位系统文件
│   │   └── Fonts/     ← 字体目录
│   ├── Program Files/     ← 64 位程序
│   ├── Program Files (x86)/ ← 32 位程序
│   └── users/
│       └── <用户名>/
│           ├── Desktop/        ← 桌面
│           ├── My Documents/   ← 我的文档
│           ├── Downloads/      ← 下载
│           └── AppData/        ← 程序数据
├── dosdevices/        ← 盘符映射（c: → drive_c 等）
├── system.reg         ← HKEY_LOCAL_MACHINE 注册表
├── user.reg           ← HKEY_CURRENT_USER 注册表
└── userdef.reg        ← HKEY_USERS 注册表
```

### 4.2 Windows ↔ Linux 路径对照

| Windows 路径 | Linux 实际路径 |
|---|---|
| `C:\` | `~/.wine/drive_c/` |
| `C:\windows\` | `~/.wine/drive_c/windows/` |
| `C:\Program Files\` | `~/.wine/drive_c/Program Files/` |
| 桌面 | `~/.wine/drive_c/users/$USER/Desktop/` |
| 我的文档 | `~/.wine/drive_c/users/$USER/My Documents/` |
| 下载 | `~/.wine/drive_c/users/$USER/Downloads/` |

### 4.3 快速访问 Wine 文件

```bash
# 在文件管理器中打开 Wine C 盘
xdg-open ~/.wine/drive_c/

# 创建方便的符号链接到 Home
ln -s ~/.wine/drive_c/users/$USER/Desktop ~/wine-桌面
ln -s ~/.wine/drive_c/users/$USER/My\ Documents ~/wine-文档
```

> **提示：** Wine 中的文件就是普通 Linux 文件，不需要任何"导出"操作。直接用文件管理器或命令行操作即可。

---

## 5. Wine 配置

### 5.1 图形配置面板

```bash
winecfg
```

打开后可以设置：

| 标签页 | 功能 |
|--------|------|
| **应用程序** | 按程序分别设置 Windows 版本（Win7/Win10/Win11 等） |
| **函数库** | 覆盖 DLL 文件（用原生 Windows DLL 替代 Wine 内置实现） |
| **显示** | 屏幕分辨率、虚拟桌面模式 |
| **桌面整合** | 主题、配色、文件夹映射 |
| **驱动器** | 管理盘符映射 |
| **音讯** | 音频驱动选择 |
| **关于** | Wine 版本信息 |

### 5.2 注册表编辑器

```bash
wine regedit
```

和 Windows 上的 regedit 完全一样，用于修改注册表项。

### 5.3 控制面板

```bash
wine control
```

打开模拟的 Windows 控制面板，包含添加/删除程序、字体管理等。

---

## 6. WINEPREFIX — 多环境隔离

`WINEPREFIX` 是 Wine 最重要的概念之一：**一个 Prefix 等于一个独立的 Windows 环境**，各自拥有独立的 C 盘、注册表、已安装程序。

### 6.1 为什么需要多个 Prefix？

- 不同程序需要**不同 Windows 版本**设置
- 避免程序之间的 DLL/注册表冲突
- 实验性安装不影响主环境
- 干净卸载：删掉整个 Prefix 即可

### 6.2 基本用法

```bash
# 创建一个独立的 Wine 环境
WINEPREFIX=~/.wine-apps/photoshop winecfg

# 在指定环境中安装程序
WINEPREFIX=~/.wine-apps/photoshop wine /path/to/setup.exe

# 在指定环境中运行程序
WINEPREFIX=~/.wine-apps/photoshop wine "C:\Program Files\Adobe\Photoshop\Photoshop.exe"
```

### 6.3 便捷脚本

在 `~/.bashrc` 或 `~/.zshrc` 中创建别名：

```bash
alias wine-ps='WINEPREFIX=~/.wine-apps/photoshop wine'
alias wine-qq='WINEPREFIX=~/.wine-apps/qq wine'
```

使用时直接：

```bash
wine-ps "C:\Program Files\Adobe\Photoshop\Photoshop.exe"
```

### 6.4 管理 Prefix

```bash
# 查看所有已创建的 Prefix
ls ~/.wine* ~/.wine-apps/

# 删除一个 Prefix（相当于格式化该 Windows"系统"）
rm -rf ~/.wine-apps/旧的程序名
```

---

## 7. winetricks 是什么？

**winetricks** 是 Wine 的**辅助配置脚本**，用于一键安装各种 Windows 组件、运行时、字体等。

### 7.1 为什么需要它？

Wine 只提供 Windows API 翻译，**不附带**下列内容：

- Visual C++ 运行库（`vcrun`）
- .NET Framework
- DirectX 组件
- 微软核心字体
- 中/日/韩文字体

没有 winetricks 时，你得自己找安装包 → 下载 → 跑安装程序 → 调注册表。winetricks 把这一切自动化。

### 7.2 安装

```bash
# Arch Linux
sudo pacman -S winetricks

# Ubuntu/Debian
sudo apt install winetricks
```

### 7.3 三种使用方式

```bash
# 1. 图形界面（勾选要装的组件）
winetricks --gui

# 2. 命令行安装指定组件
winetricks <组件名>

# 3. 列出可用的组件
winetricks list-all          # 全部
winetricks apps list         # 按应用的静默安装脚本
winetricks dlls list         # DLL/运行时组件
winetricks fonts list        # 字体包
```

### 7.4 常用组件速查

#### 运行库（DLL）

```bash
winetricks vcrun2019             # Visual C++ 2019 运行库（最常用）
winetricks vcrun6                # VC++ 6.0（老程序）
winetricks vcrun2005 vcrun2008 vcrun2010 vcrun2012 vcrun2013 vcrun2015
winetricks dotnet48              # .NET Framework 4.8
winetricks dotnet40              # .NET 4.0
winetricks directx9              # DirectX 9
winetricks d3dx9 d3dx10 d3dx11   # Direct3D 组件
```

#### 字体

```bash
winetricks corefonts             # 微软核心字体（Arial, Times, Courier 等）
winetricks cjkfonts              # 中日韩字体（解决中文乱码 ⭐）
winetricks allfonts              # 全部字体包（体积较大）
winetricks fakechinese           # 仿宋、黑体等中文字体
```

#### 其他有用组件

```bash
winetricks comctl32              # Windows 通用控件（解决界面错乱）
winetricks riched20 riched30     # 富文本编辑控件
winetricks gdiplus               # GDI+ 图形库
winetricks mfc40 mfc42           # MFC 运行库
winetricks mdac28                # 数据库访问组件
winetricks vkd3d                 # Direct3D 12 → Vulkan 转换
```

---

## 8. 字体安装详解

Wine 默认只有少量基础字体，运行中文程序时常见**乱码、方块字（豆腐块）**。以下是三种安装字体的方式。

### 8.1 方法一：winetricks 安装（推荐，最省事）

```bash
# 一键安装中日韩字体
winetricks cjkfonts
```

> **说明：** `cjkfonts` 会下载安装文泉驿微米黑、Source Han Sans 等字体。安装后重启 Wine 程序即可生效。

### 8.2 方法二：直接复制字体文件

```bash
# 复制单个字体
cp /path/to/字体.ttf ~/.wine/drive_c/windows/Fonts/

# 批量复制系统字体
cp /usr/share/fonts/TTF/*.ttf ~/.wine/drive_c/windows/Fonts/

# 从 Windows 分区提取（如果有双系统）
cp /mnt/windows/Windows/Fonts/msyh*.ttf ~/.wine/drive_c/windows/Fonts/    # 微软雅黑
cp /mnt/windows/Windows/Fonts/simsun*.ttc ~/.wine/drive_c/windows/Fonts/  # 宋体
```

### 8.3 方法三：安装系统级中文字体包

```bash
# 文泉驿（轻量）
sudo pacman -S wqy-zenhei wqy-microhei

# 思源字体（Adobe/Google，覆盖全面）
sudo pacman -S adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts

# Noto 字体（Google，多语言）
sudo pacman -S noto-fonts-cjk

# 更新字体缓存
fc-cache -fv
```

系统安装的字体 Wine 通常能自动识别。如果识别不到，手动复制到 `~/.wine/drive_c/windows/Fonts/`。

### 8.4 验证字体

```bash
# 查看 Wine 字体目录
ls ~/.wine/drive_c/windows/Fonts/

# 检查注册表中的字体注册
wine regedit
# 导航到: HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Fonts
```

---

## 9. 常见问题与故障排查

### Q: 程序启动后没反应/闪退

```bash
# 从终端启动，查看错误输出（最重要的排查手段！）
wine /path/to/program.exe 2>&1 | tee wine-error.log

# 常见错误关键词:
# "fixme"  — 功能未完全实现，通常不影响运行
# "err"    — 错误，可能是 DLL 缺失或配置问题
# "wine: cannot find" — 路径错误
```

### Q: 中文显示方框/乱码

原因：缺少中文字体。

```bash
# 最快修复
winetricks cjkfonts

# 如果还不行，检查 locale 设置
locale
# 确保 LANG 包含 zh_CN.UTF-8
```

### Q: 程序提示"缺少 MFC42.DLL"或"缺少 MSVCP140.DLL"

缺少 VC++ 运行库：

```bash
winetricks vcrun2019        # MSVCP140.DLL, VCRUNTIME140.DLL
winetricks mfc42            # MFC42.DLL
```

> **技巧：** 把报错中提到的 DLL 名在 [WineHQ AppDB](https://appdb.winehq.org/) 搜索，通常能找到对应的 winetricks 组件。

### Q: 如何完全卸载一个 Windows 程序？

```bash
# 方法一：用 Wine 自带的卸载程序
wine uninstaller
# 在图形界面中选择程序 → 卸载

# 方法二：如果用了独立 Prefix，直接删除整个 Prefix
rm -rf ~/.wine-apps/程序名
```

### Q: 32 位程序 vs 64 位程序

```bash
# 创建纯 32 位 Prefix（老程序专治）
WINEARCH=win32 WINEPREFIX=~/.wine32 winecfg

# 创建纯 64 位 Prefix（默认）
WINEARCH=win64 WINEPREFIX=~/.wine64 winecfg
```

> **注意：** `WINEARCH` 只能在创建 Prefix 时指定一次，之后不可更改。

### Q: 如何更换 Windows 版本？

某些程序需要特定 Windows 版本（如只支持 Windows 7）：

```bash
# 打开 winecfg → "应用程序" 标签页 → Windows 版本下拉框
winecfg

# 或命令行直接设
winetricks win7       # 设为 Windows 7
winetricks win10      # 设为 Windows 10
winetricks win11      # 设为 Windows 11
```

### Q: 怎么知道我的程序能不能跑？

在 [WineHQ AppDB](https://appdb.winehq.org/) 搜索程序名，查看其他用户的评分和报告：

| 评分 | 含义 |
|------|------|
| **Platinum** | 开箱即用 |
| **Gold** | 简单调校后可用 |
| **Silver** | 有一些问题但不影响核心功能 |
| **Bronze** | 能跑但影响使用 |
| **Garbage** | 基本跑不了 |

### Q: Wine 程序无法联网

```bash
# 检查 Wine 网络配置
wine control

# 部分程序可能需要
winetricks winhttp
```

### Q: 文件关联：双击 .exe 直接用 Wine 打开

大多数文件管理器在安装 Wine 后会自动关联 `.exe` 文件。如果不行：

```bash
# 检查是否注册了 binfmt
ls /proc/sys/fs/binfmt_misc/wine

# 如果没有，手动注册
sudo systemctl restart systemd-binfmt
```

---

## 10. 衍生方案：Proton、Bottles、Lutris

除了原生 Wine，社区还有以下更易用的封装方案：

| 方案 | 定位 | 适用场景 |
|------|------|----------|
| **Proton** | Valve 维护的 Wine 分支 | Steam 游戏（Steam 内置） |
| **Proton-GE** | 社区增强版 Proton | 对 Steam 版不兼容的游戏 |
| **Bottles** | GUI 管理工具 | 桌面应用 + 轻度游戏 |
| **Lutris** | 游戏启动器 | 游戏为主，含 Wine/Proton 管理 |

```bash
# 安装 Bottles（Flatpak）
flatpak install flathub com.usebottles.bottles

# 安装 Lutris
sudo pacman -S lutris
```

> **建议：** 纯桌面软件用原生 `wine` + `winetricks`；Steam 游戏用 Proton；Epic/GOG 游戏用 Lutris 或 Bottles。

---

## 11. 总结：新装 Wine 后推荐的操作流程

```bash
# 1. 安装
sudo pacman -S wine winetricks

# 2. 初始化（首次自动创建 ~/.wine）
winecfg

# 3. 安装必要组件
winetricks corefonts          # 基础英文字体
winetricks cjkfonts           # 中日韩字体（中文不乱码）
winetricks vcrun2019          # VC++ 运行库（很多程序依赖）

# 4. 安装/运行你的 Windows 程序
wine /path/to/setup.exe
wine "C:\Program Files\某程序\某程序.exe"
```

---

## 参考

- [WineHQ 官方文档](https://www.winehq.org/docs/)
- [WineHQ AppDB](https://appdb.winehq.org/) — 程序兼容性数据库
- [winetricks GitHub](https://github.com/Winetricks/winetricks)
- [Arch Wiki — Wine](https://wiki.archlinux.org/title/Wine)
