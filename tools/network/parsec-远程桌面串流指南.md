# Parsec 远程桌面完全指南

> Parsec 是一款高性能远程桌面串流软件，由 Unity Technologies 开发运营。
> 支持 4K@60fps、极低延迟的 P2P 加密连接，广泛用于远程游戏、创意工作和团队协作。

## 1. Parsec 是什么

Parsec 本质上是一个**远程桌面解决方案**，但它的设计目标与传统远程桌面（如 RDP、VNC）截然不同：

| 对比维度 | 传统远程桌面 (RDP/VNC) | Parsec |
|----------|------------------------|--------|
| 核心目标 | 办公/运维操作 | 游戏串流 + 创意工作 |
| 延迟 | 50-200ms | < 1ms（局域网）|
| 帧率 | 通常 30fps | 最高 240fps |
| 画面质量 | 压缩明显 | 4:4:4 色彩、4K |
| 游戏手柄 | 有限支持 | 完整透传 |
| 跨平台 | 有限 | Windows/macOS/Linux/Android/iOS/Web |

### 1.1 核心特性

- **极低延迟**：专有编码技术 + P2P 直连，局域网延迟可忽略不计
- **高帧率高分辨率**：最高 240fps（1080p）/ 60fps（4K），支持 4:4:4 色彩模式
- **加密 P2P 连接**：数据不经 Parsec 服务器中转，端到端加密
- **多显示器支持**：最多同时串流 3 个显示器（Warp 及以上）
- **外设透传**：键盘、鼠标、游戏手柄、Wacom 数位板完整支持
- **会话共享**：多人协作、沙发合作（couch co-op）
- **SOC 2 Type 2 认证**：企业级安全标准

### 1.2 典型使用场景

- **远程游戏**：在任意设备上串流家中高性能游戏 PC
- **创意工作**：视频剪辑、音乐制作、美术设计（支持 Wacom 压感笔）
- **软件开发**：远程访问开发机，4:4:4 色彩保证代码清晰
- **团队协作**：远程代码审查、关卡设计评审、屏幕共享
- **影视后期**：编辑器远程访问渲染工作站

## 2. 平台支持与系统要求

### 2.1 支持平台

| 平台 | 最低版本 | 主机（托管） | 客户端（接收） |
|------|----------|:------------:|:--------------:|
| Windows | Windows 10+ | ✅ | ✅ |
| macOS | macOS 10.15+ | ✅ | ✅ |
| Linux | Ubuntu 22.04 LTS Desktop | ❌ | ✅ |
| Android | — | ❌ | ✅ |
| iOS | — | ❌ | ✅ |
| Web (Chrome) | — | ❌ | ✅ |

> ⚠️ **重要提示**：Linux 目前仅支持作为**客户端**（接收串流），不支持作为主机托管游戏/桌面。
> 如需 Linux 主机串流，请考虑 [Moonlight + Sunshine](https://github.com/LizardByte/Sunshine) 方案。

### 2.2 主机（Host）硬件要求

| 组件 | 最低要求 | 推荐配置 |
|------|----------|----------|
| GPU | NVIDIA GTX 600+ / AMD RX 400+ | NVIDIA RTX 系列（支持 NVENC HEVC） |
| CPU | 双核处理器 | 四核+（Intel i5 / AMD Ryzen 5+） |
| RAM | 4 GB | 8 GB+ |
| 网络 | 5 Mbps 上行（720p/60fps） | 20+ Mbps 上行（4K/60fps） |

> 💡 **关键点**：GPU 硬件编码器（NVENC / AMD AMF）是性能的关键。使用 CPU 软编码会显著增加延迟。
> 推荐使用**有线以太网**而非 Wi-Fi，以获得稳定的低延迟体验。

### 2.3 客户端硬件要求

客户端要求很低，基本上任何能运行 Parsec 应用的设备即可：
- 现代浏览器（Chrome）即可使用 Web 客户端
- Android/iOS 设备均可流畅接收串流
- 推荐使用支持硬件解码的设备以降低功耗

## 3. 定价方案

| 方案 | 适用场景 | 月付 | 年付（每月） |
|------|----------|------|-------------|
| **Personal（免费）** | 个人游戏/远程访问 | 免费 | 免费 |
| **Warp** | 独立创作者/专业人士 | $9.99 | $8.33 |
| **Teams** | 混合/分布式团队 | $35.00 | $30.00 |
| **Enterprise** | 大型组织 | — | $45.00 |

### 3.1 各方案功能对比

| 功能 | Personal | Warp | Teams | Enterprise |
|------|:--------:|:----:|:-----:|:----------:|
| 低延迟 60fps 串流 | ✅ | ✅ | ✅ | ✅ |
| 键鼠/手柄支持 | ✅ | ✅ | ✅ | ✅ |
| 加密 P2P 连接 | ✅ | ✅ | ✅ | ✅ |
| 单链路桌面共享 | ✅ | ✅ | ✅ | ✅ |
| 多显示器（最多 3 个） | ❌ | ✅ | ✅ | ✅ |
| 4:4:4 色彩模式 | ❌ | ✅ | ✅ | ✅ |
| 隐私模式 | ❌ | ✅ | ✅ | ✅ |
| Wacom 压感笔 | ❌ | ✅ | ✅ | ✅ |
| 管理员面板 | ❌ | ❌ | ✅ | ✅ |
| SSO 集成 | ❌ | ❌ | ✅ | ✅ |
| 访客访问 | ❌ | ❌ | ✅ | ✅ |
| 审计日志 | ❌ | ❌ | ✅（7 天） | ✅（完整） |
| Teams API | ❌ | ❌ | ❌ | ✅ |
| 高性能中继服务器 | ❌ | ❌ | ❌ | ✅ |
| SCIM 配置 | ❌ | ❌ | ❌ | ✅ |

> 💡 **选购建议**：个人远程游戏使用**免费版**即可满足需求。
> 专业创意工作（需要 4:4:4 色彩或多显示器）选择 **Warp**。
> 团队使用选择 **Teams**（提供 14 天免费试用）。

## 4. 安装与配置

### 4.1 Windows

1. 访问 [parsec.app/downloads](https://parsec.app/downloads) 下载 Windows 版安装包
2. 运行安装程序，几秒内完成安装
3. 启动 Parsec，使用账号登录

### 4.2 macOS

1. 从官网下载 macOS 版本
2. 安装后授予必要的辅助功能权限（系统偏好设置 → 安全性与隐私）
3. macOS 主机需要 Metal 渲染支持（macOS 10.15+）

### 4.3 Linux（仅客户端）

#### Arch Linux（AUR 安装）

```bash
# 使用 yay 安装
yay -S parsec-bin

# 或使用 paru 安装
paru -S parsec-bin
```

> 💡 `parsec-bin` 使用预编译二进制包，推荐使用。`parsec`（非 bin）需要从源码编译。

#### Ubuntu / Debian

```bash
wget https://s3.amazonaws.com/parsec-pkg/parsec-linux.deb
sudo dpkg -i parsec-linux.deb
sudo apt-get install -f
```

#### Flatpak

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub com.parsecgaming.parsec
```

#### Arch Linux 补充依赖

```bash
# 确保 GPU 驱动正常（以 NVIDIA 为例）
sudo pacman -S nvidia nvidia-utils lib32-nvidia-utils

# 常见依赖
sudo pacman -S libpng libjpeg-turbo vulkan-icd-loader lib32-vulkan-icd-loader
```

#### 启动 Parsec

```bash
# 从终端启动
parsec

# 或从应用程序菜单中找到 Parsec 图标启动
```

### 4.4 Android / iOS

- Android：Google Play 商店搜索 "Parsec" 下载安装
- iOS：App Store 搜索 "Parsec" 下载安装
- 安装后登录账号即可连接到你的主机

### 4.5 Web 客户端

直接在 Chrome 浏览器中访问 [app.parsec.app](https://app.parsec.app)，无需安装任何软件。

## 5. 使用指南

### 5.1 首次配置主机（Windows/macOS）

1. **创建账号**：访问 [parsec.app](https://parsec.app) 注册账号
2. **登录 Parsec**：在主机上启动 Parsec 并登录
3. **设置为托管模式**：确保 Parsec 设置中 "Hosting" 已启用
4. **添加设备**：在客户端设备上登录同一账号，即可看到并连接到主机

### 5.2 连接设置（重要参数）

在 Parsec 设置中，以下参数直接影响串流质量：

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| Resolution | 串流分辨率 | 1080p（平衡）/ 4K（高质量） |
| FPS | 帧率 | 60fps（一般）/ 120-240fps（竞技） |
| Decoder | 解码器 | 硬件解码（H.264/H.265） |
| Bitrate | 码率 | 自动（或手动 20-50 Mbps） |
| Encoder | 编码器 | 硬件编码（NVENC/AMF） |
| Display | 显示器 | 选择要串流的显示器 |

> ⚠️ **局域网 vs 广域网**：免费版支持同一账号下跨网络连接，但需要通过 **"Partner"（合作伙伴）** 机制。
> Warp 及以上方案支持随时随地连接自己的设备，无需额外配置。

### 5.3 会话共享与多人协作

Parsec 独特的**会话共享**功能允许多人同时连接到同一台主机：

- **沙发合作（Couch Co-op）**：远程好友加入你的游戏，如同坐在同一沙发前
- **创意协作**：团队成员同时查看和操作同一设计/代码项目
- **来宾控制**：主机可以限制来宾可以看到和操作的应用程序

使用方式：
1. 主机端点击 "Share" 生成邀请链接
2. 将链接发送给需要连接的人
3. 对方点击链接即可加入会话

### 5.4 游戏手柄配置

Parsec 支持完整的手柄透传：

- **本地手柄**：直接在客户端使用手柄，Parsec 会自动透传到主机
- **多人手柄**：多个客户端可以各自使用不同手柄
- **手柄映射**：Parsec 会自动处理手柄映射，一般无需手动配置
- **支持的手柄**：Xbox、PlayStation、Switch Pro 等主流手柄

## 6. 网络配置与优化

### 6.1 端口与协议

- **默认端口**：UDP 8000-8001
- **协议**：优先使用 UDP（TCP 作为备选）
- **连接方式**：P2P 直连 > Parsec 中继服务器

### 6.2 防火墙配置

如果需要在主机上开放防火墙端口：

```bash
# UFW（Ubuntu/Debian）
sudo ufw allow 8000:8001/udp

# firewalld（Fedora/RHEL）
sudo firewall-cmd --permanent --add-port=8000-8001/udp
sudo firewall-cmd --reload
```

### 6.3 性能优化建议

| 场景 | 优化建议 |
|------|----------|
| 局域网游戏 | 有线以太网 + 硬件编码 + 60fps |
| 广域网远程 | Warp 方案 + 自动码率 + 有线连接 |
| 创意工作 | 4:4:4 色彩 + 适当降低帧率换画质 |
| 竞技游戏 | 高帧率（120-240fps）+ 降低分辨率 |

### 6.4 Wi-Fi 环境注意事项

如果只能使用 Wi-Fi：
- 推荐使用 **5GHz Wi-Fi**（避免 2.4GHz 的干扰和高延迟）
- 尽量靠近路由器，减少信号衰减
- 开启 Wi-Fi 6（802.11ax）以获得更低延迟
- 避免与大量设备共享同一 Wi-Fi 网络

## 7. 安全性

- **P2P 加密连接**：主机与客户端之间数据直接传输，不经过 Parsec 服务器
- **SOC 2 Type 2 认证**：通过独立第三方安全审计
- **多因素认证（MFA）**：支持在账号设置中启用
- **来宾权限控制**：主机可以精确控制来宾的访问范围
- **Teams/Enterprise**：支持 SSO、SAML、SCIM 和审计日志

## 8. 常见问题

### 8.1 Linux 客户端画面卡顿

**原因与解决**：
- 检查 GPU 驱动是否正确安装：`nvidia-smi`（NVIDIA）或 `glxinfo | grep "OpenGL renderer"`
- 确保使用硬件解码而非 CPU 软解码
- 降低串流分辨率或帧率进行测试
- 检查网络延迟：`ping <主机IP>`

### 8.2 无法连接到主机

**排查步骤**：
1. 确认主机端 Parsec 已启动并登录
2. 检查防火墙是否放行 UDP 8000-8001 端口
3. 确认两台设备在同一账号下（或已添加为合作伙伴）
4. 检查路由器是否启用了 UPnP（建议开启）
5. 如果跨网络，确认网络连接正常且 Parsec 服务可用

### 8.3 延迟过高

**优化方向**：
- 从 Wi-Fi 切换到有线以太网
- 启用硬件编码（GPU NVENC / AMD AMF）
- 降低分辨率和帧率
- 关闭主机上占用带宽的应用
- 检查是否有其他设备在大量使用网络

### 8.4 Linux 下 Parsec 无法启动

**常见原因**：
```bash
# 检查 GPU 驱动
glxinfo | grep "OpenGL renderer"

# 确保用户在 video 组中
sudo usermod -aG video $USER

# 检查依赖
ldd /opt/parsec/parsec-service
```

### 8.5 画面模糊/色彩失真

- 确认使用**硬件解码**（H.264 或 H.265）
- Warp 及以上方案可开启 **4:4:4 色彩模式**
- 提高码率设置（带宽允许的情况下）
- 避免在低带宽环境下使用高分辨率

## 9. 替代方案对比

如果你的需求不完全匹配 Parsec，以下替代方案值得考虑：

| 方案 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| **Parsec** | 画质好、延迟低、跨平台 | Linux 仅客户端、付费方案较贵 | 游戏 + 创意工作 |
| **Moonlight + Sunshine** | 完全免费开源、延迟极低 | 配置较复杂、需安装 Sunshine | 纯游戏串流 |
| **Steam Remote Play** | 与 Steam 深度集成、免费 | 仅 Steam 游戏体验好 | Steam 用户 |
| **RustDesk** | 开源、自托管 | 游戏性能不如 Parsec | 远程运维/办公 |
| **ToDesk / 向日葵** | 国内优化好、免费版功能多 | 国际延迟高、画质一般 | 国内远程办公 |

> 💡 **Linux 用户特别提示**：如果你需要 Linux 作为**主机**串流游戏，
> 推荐使用 [Sunshine](https://github.com/LizardByte/Sunshine)（主机）+ **Moonlight**（客户端）组合，
> 这是目前 Linux 上最成熟的游戏串流开源方案。

## 10. 实用技巧

### 10.1 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl + Alt + Shift + X` | 退出全屏 / 显示 Parsec 菜单 |
| `Ctrl + Alt + Shift + M` | 静音/取消静音 |

### 10.2 提升体验的小技巧

- **使用有线连接**：无论是主机还是客户端，有线以太网始终是最佳选择
- **关闭主机上的通知**：避免弹窗干扰游戏体验（Parsec 隐私模式可隐藏主机桌面）
- **固定 IP**：为家庭主机设置固定局域网 IP，方便管理和端口转发
- **自动启动**：将 Parsec 设置为开机自启动，随时可以远程连接
- **带宽预留**：在路由器中为 Parsec 主机设置 QoS 优先级

### 10.3 高级：Headless 服务器使用

Parsec 可以用于无显示器的服务器（Headless），但需要注意：
- Windows：需要虚拟显示器驱动（如 Virtual Display Driver）或 HDMI 虚拟插头
- macOS：本身支持 Headless 模式
- Linux：Parsec 官方不支持 Linux 主机，此场景请使用 Sunshine

---

> 📝 **最后更新**：2026-08-17 | 信息来源：[parsec.app](https://parsec.app)