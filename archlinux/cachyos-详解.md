# CachyOS 详解

## 1. 概述

**CachyOS** 是一个基于 **Arch Linux** 的滚动更新发行版，以**性能优化**为核心设计理念。它紧密跟随上游 Arch，同时通过编译器优化、调度器定制、系统级调优三大手段显著提升桌面响应速度和游戏性能。在 DistroWatch 上曾位列第一，是目前 Linux 社区关注度最高的发行版之一。

### 1.1 与 Arch Linux 的关系

| 维度 | Arch Linux | CachyOS |
|------|-----------|---------|
| **仓库** | 官方 + AUR | 官方 + CachyOS 优化仓库 + AUR |
| **内核** | 1 个 vanilla 内核 | 8+ 优化内核变体 |
| **安装方式** | 手动 CLI（或 archinstall） | Calamares 图形安装器 |
| **快照** | 手动配置 | Btrfs + Snapper 预配置 |
| **硬件检测** | 手动 | chwd 自动检测 |
| **桌面环境** | 手动安装 | 安装器提供 18 种选择 |
| **包优化** | x86-64 基线 | x86-64-v3 / v4 / znver4 多级优化 |

> **注意：** CachyOS 不是 Manjaro 那样走独立仓库路线的衍生版——它完全兼容 Arch 生态，甚至提供脚本将现有 Arch 系统"转换"为 CachyOS。

---

## 2. 核心优化体系

CachyOS 的性能优势来自四个层面的系统级优化。

### 2.1 CPU 微架构级包优化

CachyOS 对 Arch 软件包按 CPU 指令集级别**重新编译**，使二进制文件能利用现代 CPU 的扩展指令集：

| 仓库 | 指令集级别 | 最低 CPU 要求 | 代表型号 |
|------|-----------|---------------|----------|
| `x86-64-v3` | AVX / AVX2 / SSE4.2 | Intel Haswell / AMD Excavator (2013+) | i7-4790K, Ryzen 5 1600 |
| `x86-64-v4` | AVX-512 | Intel Skylake-X / AMD Zen 4+ | i9-7980XE, Ryzen 7 7700 |
| `znver4` | AMD Zen 4/5 专项 | Ryzen 7000 及更新 | Ryzen 7 7800X3D |

安装程序会自动检测 CPU 并启用对应仓库，用户无需手动干预。相比普通 Arch 的 `x86-64` 基线包，优化后的二进制可带来 **10%–25%** 的吞吐提升（视工作负载而定）。

### 2.2 多调度器内核阵容

CachyOS 维护了 8 个以上的内核变体，核心差异在于 **CPU 调度器**：

| 内核变体 | 调度器 | 编译优化 | 适用场景 |
|----------|--------|----------|----------|
| `linux-cachyos` | BORE | Clang + ThinLTO | **默认**，通用桌面/开发 |
| `linux-cachyos-eevdf` | EEVDF | 标准 | 追求原始吞吐 |
| `linux-cachyos-lto` | EEVDF | Clang ThinLTO + AutoFDO + Propeller | 极致吞吐，编译耗时较长 |
| `linux-cachyos-hardened` | BORE | 安全加固 | 安全敏感环境 |
| `linux-cachyos-rt-bore` | BORE + PREEMPT_RT | 实时抢占 | 专业音频、工业控制 |
| `linux-cachyos-deckify` | BORE | 手持优化 | Steam Deck / ROG Ally / Legion Go |
| `linux-cachyos-server` | EEVDF | 服务器优化 | 服务器部署 |
| `linux-cachyos-lts` | BORE | 标准 | 长期稳定兜底 |

#### BORE 调度器

**BORE**（Burst-Oriented Response Enhancer）由 Masahito Suzuki（firelzrd）开发，是 CachyOS 默认内核的核心竞争力。它在 EEVDF 之上增加了**"突发性"评分**机制：

- 频繁自愿让出 CPU 的任务（I/O、用户输入、GUI 事件）→ **低突发分 → 优先调度**
- 持续霸占 CPU 的任务（编译、渲染、科学计算）→ **高突发分 → 被降级惩罚**

> **效果：** 在后台全核编译的同时，桌面动画、浏览器滚动、打字输入仍然流畅——这是 vanilla 内核难以做到的。

BORE 可调参数（`/proc/sys/kernel/`）：

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `sched_bore` | 1 | 启用/禁用 BORE |
| `sched_burst_penalty_offset` | 24 | 低于此分值的任务视为"短突发" |
| `sched_burst_penalty_scale` | 1536 | 长突发惩罚的激进程度 |
| `sched_burst_smoothness` | 1 | 突发分平滑过渡系数 |

#### sched-ext 支持

所有 CachyOS 内核都启用了 **sched-ext**（可扩展调度器类），允许通过 eBPF 动态加载调度器，无需重启：

```bash
# 使用 scx 管理器热切换调度器
sudo scx_rusty    # Rusty — 注重延迟的调度器（Rust 实现）
sudo scx_lavd     # LAVD — 延迟感知调度器
sudo scx_bpfland  # bpfland — 主打公平和低功耗
```

### 2.3 cachyos-settings 系统调优

`cachyos-settings` 是一个元包，安装后自动应用以下优化。

#### ZRAM 内存压缩

```bash
# /etc/systemd/zram-generator.conf（默认值）
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
```

| 参数 | 值 | 说明 |
|------|-----|------|
| 压缩算法 | `zstd` | 高压缩比、低延迟 |
| `vm.swappiness` | 150 | 专为纯 ZRAM（无磁盘 swap）场景调优 |
| 再压缩 | 已关闭 | 团队测试认为无实际收益 |

#### I/O 调度器

通过 udev 规则按设备类型自动匹配：

| 设备类型 | 调度器 | 原因 |
|----------|--------|------|
| HDD（机械硬盘） | `bfq` | 低延迟桌面调度，公平分配带宽 |
| SATA SSD | `mq-deadline` | 多队列场景开销低 |
| NVMe SSD | `kyber`（2026.04 起） | 混合负载下响应更优，替代 `none` |

#### Ananicy Cpp 进程优先级管理

```bash
# 安装后自动运行
systemctl status ananicy-cpp
```

> 使用 sched-ext 调度器（如 `scx_bpfland`）时，应关闭 ananicy-cpp，避免优先级策略冲突。

#### NVIDIA 显卡优化

```bash
# 自动应用的内核参数
NVreg_UsePageAttributeTable=1   # 提升 CPU 侧性能
NVreg_EnablePCIeGen3=1          # 强制 PCIe 3.0
nvidia-drm.modeset=1            # 启用 DRM 模式设置
```

#### 其他调优

- **PCI 延迟服务**：缩短设备响应延迟
- **systemd 超时缩减**：加快开关机速度
- **Intel 音频省电关闭**：避免音频爆音

### 2.4 基准测试

| 测试场景 | 对比方 | 结果 |
|----------|--------|------|
| Xonotic 3D 游戏 | vs openSUSE Tumbleweed | CachyOS 快 **17%** |
| Xonotic 3D 游戏 | vs Fedora Workstation 42 | CachyOS 快 **22%** |

---

## 3. 安装过程

### 3.1 系统要求

| 组件 | 最低 | 推荐 |
|------|------|------|
| CPU | x86-64-v3（Intel Haswell 2013+ / AMD Ryzen） | 同上 |
| RAM | 3 GB | 8 GB |
| 存储 | 30 GB | 50 GB (SSD/NVMe) |
| GPU | NVIDIA GTX 900+ / AMD GCN 1.0+ / Intel HD/Arc | 同上 |

> ⚠️ **CachyOS 不为老旧硬件设计。** 如果你的 CPU 早于 2013 年，应使用普通 Arch Linux 或其他轻量发行版。

### 3.2 安装器选项

CachyOS 使用 **Calamares** 图形安装器：

**引导加载器：**

| 加载器 | 说明 |
|--------|------|
| **Limine** | 2026年1月起成为默认，轻量快速 |
| GRUB | 传统选择，兼容性好 |
| systemd-boot | 仅 UEFI，配置简洁 |
| rEFInd | macOS 风格界面，多系统引导友好 |

**文件系统：**

| 文件系统 | 说明 |
|----------|------|
| **Btrfs** | 默认，支持快照、压缩、子卷 |
| EXT4 | 传统可靠 |
| XFS | 大文件性能优异 |
| ZFS | 高级数据完整性 |
| F2FS | Flash 存储优化 |
| bcachefs | 新一代 CoW 文件系统 |

**桌面环境：** 安装器提供 18 种选择，涵盖 KDE Plasma（默认）、GNOME、Xfce、MATE、Cinnamon、Budgie、LXQt、Hyprland、Sway、i3、bspwm、COSMIC 等。

### 3.3 安装后首次启动

```bash
# 1. 运行硬件检测，确保驱动正确安装
sudo chwd --force

# 2. 打开 CachyOS Hello 应用，进行个性化配置
#    - 游戏组件安装
#    - DNS-over-HTTPS（通过 blocky 实现）
#    - VRAM 管理
#    - 内核切换

# 3. 系统更新
sudo pacman -Syu
```

---

## 4. 软件包管理

### 4.1 仓库结构

```
[CachyOS 优化仓库]   ← 重编译的优化包，覆盖官方同名包
[Arch 官方仓库]      ← core, extra, multilib
[AUR]               ← 社区维护包
```

```bash
# 查看包来源
pacman -Qi <包名> | grep -i repository
```

### 4.2 Shelly GUI 包管理器

2026年4月起，**Shelly** 成为默认图形化包管理器：

| 特性 | 说明 |
|------|------|
| 技术栈 | GTK4 + libalpm |
| AUR | 内置支持（yay/paru 后端） |
| Flatpak | 浏览与安装 |
| AppImage | 管理与集成 |

### 4.3 命令行包管理

完全兼容 `pacman` 语法，AUR 操作使用 `yay` 或 `paru`：

```bash
sudo pacman -Syu               # 系统更新
sudo pacman -S <包名>           # 安装官方包
yay -S <aur包名>               # 安装 AUR 包
```

---

## 5. 内核管理

### 5.1 切换内核

```bash
# GUI 方式
cachyos-kernel-manager

# 命令行方式
sudo pacman -S linux-cachyos-bore
sudo pacman -S linux-cachyos-deckify
```

### 5.2 内核选择建议

| 使用场景 | 推荐内核 |
|----------|----------|
| 日常桌面/开发 | `linux-cachyos`（默认，BORE） |
| 游戏 + 后台编译 | `linux-cachyos` 或 `linux-cachyos-lto` |
| 专业音频制作 | `linux-cachyos-rt-bore` |
| 服务器 | `linux-cachyos-server` |
| Steam Deck 等手持设备 | `linux-cachyos-deckify` |
| 安全敏感环境 | `linux-cachyos-hardened` |
| 系统兜底 | `linux-cachyos-lts` |

> LTS 内核会随默认安装一同部署。如果新版内核出现兼容性问题，在引导菜单中选择 LTS 内核启动即可。

---

## 6. 游戏特性

### 6.1 Proton-CachyOS

CachyOS 维护了增强版 Proton，在 Valve 官方版本基础上增加：

| 特性 | 说明 |
|------|------|
| DLSS 版本升级器 | 强制使用最新 DLSS DLL |
| XeSS 版本升级器 | Intel 超分技术版本升级 |
| FSR 4 支持 RDNA 3 | 将 FSR 4 画质提升解锁给旧 GPU |
| FSR 3.1 升级器 | 自动升级空间超分辨率版本 |
| 每游戏着色器缓存 | 防止着色器缓存被其他游戏驱逐 |
| NTSync | NT 同步原语，减少 Wine 同步开销 |
| dxvk-gplasync | 替代 DXVK，减少图形管线编译卡顿 |
| dxvk-sarek | 为不支持 Vulkan 1.3 的老旧 GPU 提供支持 |

```bash
sudo pacman -S proton-cachyos
```

### 6.2 一键游戏配置

通过 **CachyOS Hello** 应用点击"安装游戏组件"即可自动部署：

- Steam、Lutris、Heroic Games Launcher
- Wine / Wine-GE
- Gamemode（游戏优化守护进程）
- MangoHud（性能叠加层）
- 32 位图形库

### 6.3 Handheld Edition

官方支持以下手持设备：

- Steam Deck（LCD / OLED）
- ASUS ROG Ally / ROG Ally X
- Lenovo Legion Go / Legion Go S

使用 `linux-cachyos-deckify` 内核，包含手柄映射和功耗管理优化。

---

## 7. Btrfs 快照与回滚

### 7.1 默认子卷布局

```
/              ← @ 子卷，Snapper 快照目标
/home          ← @home 子卷，不在快照范围内
/root          ← @root 子卷
/var/cache     ← @cache 子卷，包缓存不被快照
/var/log       ← @log 子卷，日志不被快照
/srv           ← @srv 子卷
```

### 7.2 Snapper 默认配置

```ini
# /etc/snapper/configs/root
SPACE_LIMIT="0.5"          # 快照最多占 50% 空间
FREE_LIMIT="0.2"           # 空闲低于 20% 触发清理
NUMBER_LIMIT="50"          # 最多保留 50 个快照
TIMELINE_CREATE="no"       # 时间线快照默认关闭
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
```

> 每次执行 `pacman` 操作（安装/更新/删除）时，Snapper 自动创建**预/后快照对**。时间线快照需手动启用：
> ```bash
> sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
> ```

### 7.3 常用操作

```bash
snapper list                           # 列出快照
sudo snapper create -d "手动快照说明"    # 创建快照
snapper diff 1..2                      # 对比两个快照的差异
sudo snapper rollback 5                # 回滚到指定快照
```

### 7.4 通过引导菜单回滚

Limine 和 GRUB 会自动生成快照引导条目。系统更新失败时：

1. 重启，在引导菜单中选择一个快照条目启动
2. 进入快照的只读环境后执行回滚：
   ```bash
   sudo snapper rollback <快照号>
   ```

> systemd-boot 用户不享受此功能，需通过 Live USB 或 chroot 手动回滚。

### 7.5 空间管理

```bash
sudo btrfs filesystem usage /          # 查看 Btrfs 使用情况
sudo snapper -c root set-config NUMBER_LIMIT=20  # 调低快照上限
```

> **警告：** 默认 50 个快照上限对小型 SSD（<100GB）或大量 Flatpak/Docker 场景可能过于宽松，建议酌情调低至 20，避免空间耗尽。

---

## 8. 核心工具链

| 工具 | 语言 | 功能 |
|------|------|------|
| **CachyOS Hello** | Python | 安装后欢迎应用：游戏组件、DNS、VRAM、内核管理 |
| **chwd** | Rust | 自动硬件检测与驱动安装 |
| **kernel-manager** | — | GUI 内核安装与切换 |
| **Shelly** | GTK4 | 图形化包管理器（AUR/Flatpak/AppImage） |
| **cachy-chroot** | Shell | 一键 chroot 到已有安装 |
| **Cachy-Update** | — | 系统托盘更新通知 |
| **Ananicy Cpp** | C++ | 进程优先级自动管理 |

### 8.1 chwd 硬件检测

```bash
sudo chwd --force
# 自动检测：NVIDIA / AMD / Intel GPU 驱动
#          手持设备配置（Steam Deck, ROG Ally 等）
#          T2 MacBook 驱动
#          Intel Media SDK / VPL GPU Runtime
```

### 8.2 cachy-chroot

```bash
# 从 Live USB 修复系统
sudo mount /dev/nvme0n1p2 /mnt
sudo cachy-chroot /mnt
# 已进入目标系统的 shell 环境
```

---

## 9. 常见问题

### Q1: CachyOS 和 Arch Linux 的包可以混用吗？

完全可以。CachyOS 仓库完全兼容 Arch 官方仓库。优化仓库中的包只是对官方同名包进行了重编译，安装后无缝工作。`pacman` 和 AUR 助手的使用方式与 Arch 一致。

### Q2: 我的 CPU 不支持 x86-64-v3，能用 CachyOS 吗？

不能。CachyOS 明确要求 x86-64-v3 级别的 CPU。如果你的 CPU 早于 2013 年（Sandy Bridge、Ivy Bridge、Piledriver 等架构），应使用普通 Arch Linux。

### Q3: 应该选择哪个内核变体？

大多数用户使用默认的 `linux-cachyos`（BORE 调度器）即可。如果你需要在重负载下保持桌面流畅（如边编译边游戏），这也是最优选择。只在特定场景（实时音频、服务器、手持设备）才需要切换。

### Q4: 如何从 Arch Linux 迁移到 CachyOS？

```bash
# 1. 添加 CachyOS 仓库
wget https://mirror.cachyos.org/cachyos-repo.tar.xz
tar xvf cachyos-repo.tar.xz
cd cachyos-repo
sudo ./cachyos-rate-mirrors.sh

# 2. 安装 CachyOS 核心组件
sudo pacman -S linux-cachyos linux-cachyos-headers cachyos-settings

# 3. 更新引导加载器配置
sudo grub-mkconfig -o /boot/grub/grub.cfg  # GRUB 用户
```

> 迁移前务必创建系统快照或备份。

### Q5: Btrfs 快照占满空间怎么办？

```bash
# 紧急清理
sudo snapper delete $(snapper list -t snapshot | tail -20 | awk '{print $1}' | tr '\n' ' ')

# 调低上限
sudo snapper -c root set-config NUMBER_LIMIT=20

# 清理包缓存
sudo pacman -Sc
```

### Q6: sched-ext 和 BORE 有什么区别？

BORE 是内核**内置**调度器补丁，开机即用。sched-ext 是**动态加载**的 eBPF 调度器，需要手动启动（如 `sudo scx_rusty`），灵活但需用户主动管理。两者可以共存——日常用 BORE，特殊场景临时切到 sched-ext。

### Q7: ananicy-cpp 需要手动配置吗？

不需要。安装后自动运行，使用社区维护的优先级规则。若使用 sched-ext 调度器，建议关闭以避免冲突：

```bash
sudo systemctl disable --now ananicy-cpp
```

---

## 10. 参考资料

- [CachyOS 官方网站](https://cachyos.org/)
- [CachyOS Wiki](https://wiki.cachyos.org/)
- [CachyOS GitHub — linux-cachyos 内核](https://github.com/CachyOS/linux-cachyos)
- [CachyOS GitHub — chwd 硬件检测](https://github.com/CachyOS/chwd)
- [CachyOS Forum](https://discuss.cachyos.org/)
- [Linux Magazine — Performance Driven](https://www.linuxpromagazine.com/Issues/2026/302/CachyOS/)
- [Phoronix — CachyOS 新闻](https://www.phoronix.com/search/CachyOS)
- [Arch Wiki](https://wiki.archlinux.org/)
