# Arch Linux KDE 应用套件详解

> 详解 `plasma-meta`、`kde-applications-meta` 及其 12 个子分类元包的结构、内容与选装策略。

## 1. 理解核心概念

### 1.1 元包 vs 包组

在开始之前，必须区分两个容易混淆的概念：

| | 元包（meta-package） | 包组（group） |
|------|------|------|
| **本质** | 一个空壳包，通过依赖声明拉取其他包 | pacman 定义的包分类标签 |
| **安装** | `pacman -S plasma-meta` | `pacman -S plasma` |
| **成员标记** | 被拉取的包标记为"作为依赖安装" | 组内每个包标记为"显式安装" |
| **卸载行为** | `pacman -Rns plasma-meta` 一键带走全部依赖 | 需逐个卸载，组本身不会自动清理 |
| **典型例子** | `plasma-meta`、`kde-applications-meta` | `plasma`、`kde-applications` |

> **选择原则：** 想要「干净卸载」选元包；想要「精细控制每个组件的去留」选包组。

### 1.2 两种顶层包的关系

```
                    KDE 软件栈
                         │
        ┌────────────────┼────────────────┐
        │                                 │
   plasma-meta                    kde-applications-meta
   （桌面环境）                    （应用套件）
        │                                 │
  ┌─────┼─────┐              ┌────┼────┬───┴───┬────┐
  │     │     │              │    │    │       │    │
  │  kwin  plasma-        无障碍 教育  游戏   ... KDevelop
  │        workspace       (12 个子分类元包，约 183 个应用)
  │
  └─ 桌面外壳、面板、窗口管理、主题等
```

> `plasma-meta` 管桌面环境本身（面板、窗口、设置），`kde-applications-meta` 管上层应用（文件管理器、编辑器、播放器、游戏等）。两者互相独立，可以只装其中一个。

---

## 2. plasma-meta — 桌面环境

### 2.1 三种安装 Plasma 的方式

| 包名 | 类型 | 内容 | 适用场景 |
|------|------|------|---------|
| `plasma-meta` | 元包 | 完整 Plasma 桌面 + 基础应用 | **推荐**——可干净卸载 |
| `plasma` | 包组 | 与 `plasma-meta` 内容相同 | 需要精细控制每个组件 |
| `plasma-desktop` | 元包 | 最简 Plasma（仅桌面外壳） | 极简安装，后续手动补应用 |

### 2.2 plasma-meta 包含的核心组件

```
plasma-meta
├── plasma-workspace      ← 桌面外壳、面板、通知系统
├── kwin                  ← 窗口管理器 / Wayland 合成器
├── plasma-desktop        ← 桌面本身（图标、壁纸、右键菜单）
├── systemsettings        ← 系统设置
├── dolphin               ← 文件管理器
├── konsole               ← 终端模拟器
├── sddm-kcm              ← SDDM 登录管理器配置模块
├── kscreen               ← 多显示器管理
├── powerdevil            ← 电源管理（休眠、亮度、电量）
├── plasma-nm             ← 网络管理面板组件
├── plasma-pa             ← 音频控制面板组件
├── plasma-firewall       ← 防火墙配置
├── breeze                ← 默认主题（图标、光标、窗口装饰）
├── discover              ← 软件中心
├── drkonqi               ← 崩溃处理对话框
├── kde-gtk-config        ← GTK 应用外观统一配置
├── kgamma                ← 显示器色彩校准
├── kinfocenter            ← 系统信息中心
└── ...
```

### 2.3 常用操作

```bash
# 最小安装：只装桌面本身
sudo pacman -S plasma-desktop

# 标准安装：完整 Plasma 体验（推荐）
sudo pacman -S plasma-meta

# 包组安装：相同内容，不同卸载行为
sudo pacman -S plasma

# 完全卸载
sudo pacman -Rns plasma-meta
```

---

## 3. kde-applications-meta — 应用套件

### 3.1 层级结构

`kde-applications-meta` 是一个**顶层元包**，自身为空壳，通过 12 个子元包拉取约 **183 个应用**：

```
kde-applications-meta
├── kde-accessibility-meta    ← 无障碍辅助      （5 个）
├── kde-education-meta        ← 教育            （21 个）
├── kde-games-meta            ← 游戏            （41 个）
├── kde-graphics-meta         ← 图形图像        （14 个）
├── kde-multimedia-meta       ← 多媒体          （14 个）
├── kde-network-meta          ← 网络            （16 个）
├── kde-office-meta           ← 办公            （2 个）
├── kde-pim-meta              ← 个人信息管理    （14 个）
├── kde-sdk-meta              ← 软件开发        （13 个）
├── kde-system-meta           ← 系统管理        （8 个）
├── kde-utilities-meta        ← 实用工具        （32 个）
└── kdevelop-meta             ← KDevelop IDE    （3 个）
```

### 3.2 各分类详细说明

#### 3.2.1 kde-accessibility-meta — 无障碍辅助（5 个）

| 应用 | 用途 |
|------|------|
| `accessibility-inspector` | 无障碍属性检查与诊断 |
| `kmag` | 屏幕放大镜 |
| `kmousetool` | 鼠标辅助点击（为操作鼠标困难的用户设计） |
| `kmouth` | 文本—语音合成器 |
| `kontrast` | 色彩对比度检查工具（验证可读性合规） |

#### 3.2.2 kde-education-meta — 教育（21 个）

| 应用 | 用途 |
|------|------|
| `marble` | 虚拟地球仪（类似 Google Earth） |
| `kalgebra` | 3D 图形计算器 |
| `kalzium` | 化学元素周期表 |
| `kgeography` | 地理学习（国家、首都、旗帜） |
| `ktouch` | 打字练习 |
| `kturtle` | Logo 编程语言教学环境 |
| `parley` | 闪卡记忆（间隔重复） |
| `minuet` | 音乐听力训练（音程、和弦辨识） |
| `cantor` | 数学/科学前端（可接 Julia、Python、R、Sage 等后端） |
| `step` | 交互式物理模拟器 |
| `blinken` | Simon Says 记忆游戏 |
| `kanagram` | 字母乱序猜词 |
| `khangman` | 猜词游戏（Hangman） |
| `klettres` | 字母和音节学习 |
| `kwordquiz` | 词汇测验 |
| `kbruch` | 分数练习 |
| `kmplot` | 数学函数绘图 |
| `kig` | 交互式几何 |
| `rocs` | 图论教学 |
| `kiten` | 日语参考/学习工具（汉字字典） |
| `artikulate` | 发音训练 |

#### 3.2.3 kde-games-meta — 游戏（41 个）

覆盖面最广的分类：

| 类型 | 应用 | 说明 |
|------|------|------|
| 卡牌类 | `kpat` | 纸牌合集（Klondike、FreeCell 等十余种） |
| | `lskat` | 德国 Skat 纸牌 |
| | `klickety` | 同色消除（类似 SameGame） |
| 棋类 | `knights` | 国际象棋 |
| | `kreversi` | 黑白棋（Reversi） |
| | `kshisen` | 四川省麻将（连连看） |
| | `kmahjongg` | 麻将接龙 |
| 益智类 | `ksudoku` | 数独 |
| | `kmines` | 扫雷 |
| | `katomic` | 原子分子拼图（Sokoban 类） |
| | `picmi` | 数图（Nonogram） |
| | `knetwalk` | 管道连接 |
| 动作/街机 | `kbounce` | 接球（JezzBall） |
| | `kblocks` | 俄罗斯方块 |
| | `ksnakeduel` | 贪食蛇对战 |
| | `granatier` | 炸弹人克隆 |
| | `kapman` | 吃豆人 |
| | `kollision` | 小球躲避 |
| 策略类 | `kajongg` | 四人麻将 |
| | `konquest` | 银河征服 |
| | `ksirk` | Risk 世界征服 |
| | `kigo` | 围棋 |
| | `kfourinline` | 四子棋 |
| 其他 | `kblackbox` | 射线推理 |
| | `kdiamond` | 钻石迷情 |
| | `kubrick` | 魔方 |
| | `palapeli` | 拼图 |
| | `bomber`、`bovo`、`killbots`、`ksquares` 等 | ... |

#### 3.2.4 kde-graphics-meta — 图形图像（14 个）

| 应用 | 用途 |
|------|------|
| `gwenview` | **图片浏览器**（KDE 默认首选） |
| `okular` | **全能文档阅读器**（PDF、ePub、DjVu、漫画、CHM 等格式） |
| `kolourpaint` | 画图（KDE 版 MSPaint，比想象中好用） |
| `koko` | 照片管理器 |
| `skanlite` | 扫描仪前端 |
| `kruler` | 屏幕标尺（像素测量） |
| `kcolorchooser` | 屏幕取色器 |
| `kamera` | 数码相机导入工具 |
| `svgpart` | SVG 矢量图查看 KPart 组件 |
| `kdegraphics-thumbnailers` | 图形文件缩略图插件（PS/RAW 等格式） |
| `colord-kde` | 色彩管理前端 |
| `arianna` | EPUB 电子书阅读器 |
| `kgraphviewer` | Graphviz DOT 图形查看器 |
| `kimagemapeditor` | HTML 图片热区编辑器 |

#### 3.2.5 kde-multimedia-meta — 多媒体（14 个）

| 应用 | 用途 |
|------|------|
| `elisa` | 现代音乐播放器（触控友好） |
| `juk` | 经典音乐管理器/播放器 |
| `dragon` | 极简视频播放器 |
| `kdenlive` | **视频剪辑器**——KDE 旗舰应用之一，MLT 框架驱动 |
| `k3b` | 光盘刻录（CD/DVD/Blu-ray） |
| `kmix` | 音量混音器 |
| `audex` | CD 抓轨（支持 CDDB 元数据） |
| `audiotube` | YouTube 音乐客户端 |
| `kasts` | 播客客户端 |
| `plasmatube` | YouTube 视频客户端 |
| `kwave` | 音频编辑器 |
| `kamoso` | 摄像头/自拍应用 |
| `ffmpegthumbs` | 视频文件缩略图（FFmpeg 后端） |
| `audiocd-kio` | 音频 CD 的 KIO 协议支持 |

#### 3.2.6 kde-network-meta — 网络（16 个）

| 应用 | 用途 |
|------|------|
| `kdeconnect` | **手机—电脑互联**（通知同步、文件传输、剪贴板共享、远程输入） |
| `falkon` | 轻量 Web 浏览器（基于 QtWebEngine） |
| `angelfish` | 移动端 Web 浏览器 |
| `konqueror` | KDE 经典全能浏览器/文件管理器 |
| `ktorrent` | BT 下载器 |
| `kget` | 下载管理器 |
| `neochat` | Matrix 聊天客户端 |
| `tokodon` | Mastodon 客户端 |
| `konversation` | IRC 客户端 |
| `alligator` | RSS 订阅阅读器 |
| `krdc` | 远程桌面客户端（RDP/VNC） |
| `krfb` | 远程桌面服务端（VNC） |
| `kio-extras` | 额外 KIO 协议（sftp、smb、mtp、nfs 等） |
| `kio-gdrive` | Google Drive 网络挂载 |
| `kio-zeroconf` | 局域网服务发现（Avahi/Bonjour） |
| `kdenetwork-filesharing` | Samba 网络共享配置 |

#### 3.2.7 kde-office-meta — 办公（2 个）

| 应用 | 用途 |
|------|------|
| `calligra` | KDE 办公套件（Words、Sheets、Stage、Flow 等） |
| `ghostwriter` | Markdown 写作编辑器（专注无干扰写作模式） |

> 日常办公通常用 LibreOffice。`calligra` 更轻量，与 KDE 深度集成，适合轻度使用。`ghostwriter` 是优秀的 Markdown 编辑器。

#### 3.2.8 kde-pim-meta — 个人信息管理（14 个）

| 应用 | 用途 |
|------|------|
| `kmail` | 邮件客户端 |
| `korganizer` | 日历/日程管理 |
| `kaddressbook` | 通讯录 |
| `akregator` | RSS 阅读器 |
| `kontact` | 以上所有的一体化前端（类似 Outlook） |
| `kleopatra` | 证书/GPG 密钥管理器 |
| `kdepim-addons` | PIM 套件扩展插件 |
| `akonadiconsole` | Akonadi 存储后端调试控制台 |
| `akonadi-calendar-tools` | 日历命令行工具 |
| `itinerary` | 旅行行程管理（自动提取预订邮件中的行程信息） |
| `kalarm` | 闹钟/提醒 |
| `grantlee-editor` | 邮件/文档模板编辑器 |
| `merkuro` | 新一代日历/联系人应用 |
| `zanshin` | GTD（Getting Things Done）待办事项管理 |

> Akonadi 是 KDE PIM 的数据存储后端，首次启动会初始化数据库（MySQL/MariaDB 或 SQLite）。如果只用 `kmail` 等单个应用，系统会自动拉取所需的最小 Akonadi 组件。

#### 3.2.9 kde-sdk-meta — 软件开发（13 个）

| 应用 | 用途 |
|------|------|
| `kcachegrind` | 性能分析可视化（Valgrind/Callgrind 前端） |
| `kompare` | 差异/补丁文件可视化对比 |
| `lokalize` | 翻译/本地化辅助工具 |
| `umbrello` | UML 建模工具 |
| `kapptemplate` | 项目模板生成器 |
| `kdesdk-kio` | SDK 的 KIO 协议插件 |
| `kdesdk-thumbnailers` | 代码/工程文件缩略图 |
| `dolphin-plugins` | Dolphin 文件管理器扩展（Git、SVN、Mercurial 集成） |
| `kirigami-gallery` | Kirigami UI 框架控件示例浏览器 |
| `massif-visualizer` | 内存分析可视化（Massif 前端） |
| `kde-dev-scripts` | 开发辅助脚本集合 |
| `kde-dev-utils` | 开发辅助工具 |
| `poxml` | XML 翻译辅助工具 |

#### 3.2.10 kde-system-meta — 系统管理（8 个）

| 应用 | 用途 |
|------|------|
| `dolphin` | **文件管理器**（KDE 核心应用之一） |
| `partitionmanager` | 分区编辑器（KDE 版 GParted） |
| `ksystemlog` | 系统日志查看器（syslog/journald） |
| `khelpcenter` | KDE 帮助中心 |
| `kcron` | Cron 定时任务图形化编辑器 |
| `kjournald` | journald 日志查看器 |
| `kio-admin` | 管理员权限的 KIO 协议（`admin://` 路径，无需 `sudo dolphin`） |
| `kde-inotify-survey` | inotify 文件监控调试工具 |

#### 3.2.11 kde-utilities-meta — 实用工具（32 个）

最大的分类，日常高频使用：

| 应用 | 用途 |
|------|------|
| `konsole` | **终端模拟器** |
| `kate` | **高级文本编辑器**（支持 LSP、多光标、Git 集成） |
| `ark` | **压缩包管理器**（zip/tar/7z/rar 等） |
| `kcalc` | 计算器 |
| `kfind` | 文件搜索 |
| `filelight` | 磁盘空间扇形图（快速定位空间占用） |
| `yakuake` | 下拉式终端（Quake 风格，快捷键弹出/收起） |
| `kbackup` | 备份工具 |
| `kdf` | 磁盘使用查看（类似 `df` 命令的 GUI） |
| `kgpg` | GPG 密钥管理器 |
| `kwalletmanager` | KDE 密码钱包管理器 |
| `sweeper` | 隐私清理（浏览器缓存、历史记录等） |
| `kcharselect` | 特殊字符选择器（Unicode，直接插入或复制） |
| `kweather` | 天气预报 |
| `ktimer` | 倒计时/秒表 |
| `kteatime` | 泡茶计时器（经典 KDE 小玩意） |
| `krecorder` | 屏幕录制 |
| `isoimagewriter` | USB 启动盘写入工具 |
| `kdebugsettings` | 调试日志分类配置 |
| `kalm` | 呼吸/冥想助手 |
| `keysmith` | 二步验证码（TOTP）生成器 |
| `qrca` | 二维码扫描器/生成器 |
| `skanpage` | 文档扫描与 OCR |
| `telly-skout` | 电视节目表 |
| `kongress` | 会议/活动助手 |
| `markdownpart` | Markdown 预览 KPart 组件 |
| `francis` | 番茄钟 |
| `kclock` | 世界时钟/闹钟 |
| `kalk` | 科学计算器（移动端风格） |
| `ktrip` | 公共交通查询 |
| `keditbookmarks` | 书签编辑器 |
| `kdialog` | Shell 脚本对话框（可从脚本调用 KDE 对话框） |

#### 3.2.12 kdevelop-meta — KDevelop IDE（3 个）

| 应用 | 用途 |
|------|------|
| `kdevelop` | C++/Python/PHP 集成开发环境 |
| `kdevelop-php` | PHP 语言支持插件 |
| `kdevelop-python` | Python 语言支持插件 |

---

## 4. 选装策略

### 4.1 方案对比

| 方案 | 命令示例 | 安装量 | 适用场景 |
|------|---------|--------|---------|
| **全量安装** | `pacman -S kde-applications-meta` | ~183 个 | ❌ 不推荐，大量无用应用 |
| **按分类选装** | `pacman -S kde-utilities-meta kde-graphics-meta` | 按需 | ⭐ 推荐，保留卸载便利性 |
| **逐个选装** | `pacman -S dolphin gwenview okular kate` | 最少 | 极致精简，但无聚合卸载 |

### 4.2 推荐最小实用组合

```bash
# 桌面环境
sudo pacman -S plasma-meta

# 日常必装（按分类）
sudo pacman -S \
  kde-utilities-meta \    # konsole、kate、ark、kcalc、filelight、yakuake
  kde-graphics-meta \     # gwenview、okular、spectacle
  kde-system-meta \       # dolphin、partitionmanager
  kde-network-meta        # kdeconnect、kio-extras、kio-gdrive
```

### 4.3 卸载

```bash
# 元包方式卸载——一步清理所有
sudo pacman -Rns plasma-meta
sudo pacman -Rns kde-applications-meta

# 如果用的是包组安装，每个应用需单独卸载
sudo pacman -Rns dolphin gwenview okular kate ...
```

---

## 5. 常见问题

### Q1: 装完 plasma-meta 后没有文件管理器？

**答：** `dolphin` 在 `kde-system-meta` 中，而 `plasma-meta` 只拉桌面环境本身。手动安装：

```bash
sudo pacman -S kde-system-meta
# 或只装 dolphin
sudo pacman -S dolphin
```

### Q2: kmail 拉了一大堆 Akonadi 依赖，可以不要吗？

**答：** `kmail` 依赖 Akonadi 存储后端。如果不用 PIM 功能，不要装 `kde-pim-meta`，直接跳过。轻量邮件需求可用 Thunderbird 或 Webmail。

### Q3: 如何查看某个元包具体包含哪些应用？

```bash
# 查看元包的直接依赖
pacman -Si kde-utilities-meta | grep 依赖于

# 递归展开所有依赖（树状）
pactree kde-utilities-meta
```

### Q4: 卸载 plasma-meta 会删掉我装的 GTK 应用吗？

**答：** 不会。`plasma-meta` 只删除自己的依赖。你显式安装的 GTK 应用（如 GIMP、Firefox）不受影响。

### Q5: 包组安装的包怎么一键清理？

**答：** 无法一键清理，这是包组的本质限制。可以用以下脚本找出 KDE 相关的显式安装包：

```bash
pacman -Qqe | grep -E '^(plasma|kde|kf5|kirigami|breeze)' > kde-packages.txt
# 审查后卸载
sudo pacman -Rns $(cat kde-packages.txt)
```

或者更好的做法：从一开头就用元包安装。

---

## 参考

- [Arch Wiki: KDE](https://wiki.archlinux.org/title/KDE)
- [Arch Wiki: Meta package and package group](https://wiki.archlinux.org/title/Meta_package_and_package_group)
- [KDE Applications 官方页面](https://apps.kde.org/)
- `man pacman`
