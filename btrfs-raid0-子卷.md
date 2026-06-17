# Btrfs RAID0 与子卷实战指南

> 本文教你如何用 Btrfs 组建 RAID0（条带化）提升读写性能，同时利用子卷（subvolume）实现灵活的布局管理。

## 1. 理解核心概念

### 1.1 RAID0 在 Btrfs 中的含义

Btrfs 的 RAID0 和传统 mdadm RAID0 原理一样：数据被**条带化（stripe）** 写入多块设备，写入时多盘并行，读取时多盘并行。

```
传统单盘写入：        Btrfs RAID0 写入：
  磁盘A                磁盘A        磁盘B
  ████████             ████░░░░    ░░░░████
  ████████      →      ████░░░░    ░░░░████
  ████████             ████░░░░    ░░░░████
  耗时 = T             耗时 ≈ T/2
```

**特点：**
- 顺序读写速度 ≈ 磁盘数 × 单盘速度
- 无冗余，任意一块盘坏 = 全部数据丢失
- 可用容量 = 磁盘数 × 最小磁盘的大小

### 1.2 子卷是什么

子卷（subvolume）是 Btrfs 中的独立命名空间，**不是分区**，但可以像分区一样独立挂载。

| 对比项 | 传统分区 | Btrfs 子卷 |
|--------|---------|------------|
| 大小限制 | 分区时固定 | **动态共享**存储池，无硬边界 |
| 调整大小 | 需要 resize2fs 等工具 | 无需调整，自动伸缩 |
| 快照 | 不支持 | **秒级快照，不占额外空间** |
| 独立挂载 | 支持 | 支持（通过 `subvol=` 参数） |

---

## 2. 前期准备

### 2.1 确认磁盘

```bash
# 列出所有块设备，确认磁盘名称和容量
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
```

下文约定：
- `/dev/sda` — 第一块磁盘（如 500GB）
- `/dev/sdb` — 第二块磁盘（如 500GB）
- 如果磁盘容量不同，参考 [2.2 异容量磁盘处理](#22-异容量磁盘处理)

### 2.2 异容量磁盘处理

如果两块磁盘容量不同（如 500GB + 1TB），你有两种选择：

| 方案 | 操作 | 可用空间 | 性能 |
|------|------|---------|------|
| **分区对齐（推荐）** | 大磁盘分出一个等大分区 | 2 × 较小盘 | 全速 RAID0 |
| **混用 single 模式** | 大磁盘多出空间用 `single` 分配 | 两盘之和 | RAID0 区域全速 |

**分区对齐示例**（500GB + 1TB）：

```bash
# 小盘：整盘一个分区
parted -s /dev/sda mklabel gpt
parted -s /dev/sda mkpart primary 0% 100%
# /dev/sda1 = 500G

# 大盘：分 500G 给 RAID0，剩余另用
parted -s /dev/sdb mklabel gpt
parted -s /dev/sdb mkpart primary 0% 500GB
parted -s /dev/sdb mkpart primary 500GB 100%
# /dev/sdb1 = 500G（参与 RAID0）
# /dev/sdb2 = 500G（单独用途）
```

> **物理提醒**：如果是机械硬盘（HDD），避免在 `/dev/sdb2` 上同时高负载读写，否则磁头寻道会拖慢 RAID0 性能。SSD 不受此影响。

---

## 3. 组建 Btrfs RAID0

### 3.1 格式化

```bash
# 元数据 RAID1（推荐）：文件系统结构有冗余，即使数据坏也能定位丢失的文件
# 数据 RAID0：条带化，追求速度
mkfs.btrfs -m raid1 -d raid0 /dev/sda1 /dev/sdb1

# 或者纯 RAID0（元数据也不冗余，空间利用率最高但不推荐）
mkfs.btrfs -m raid0 -d raid0 /dev/sda1 /dev/sdb1
```

> **为什么推荐 `-m raid1`？** 元数据体量极小（通常几十 MB ~ 几百 MB），冗余开销可忽略，但能保护文件系统的"目录索引"不因单盘损坏而崩溃。数据 RAID0 本身就无冗余，至少让文件系统结构有备份。

### 3.2 挂载并验证

```bash
# 临时挂载
mount /dev/sda1 /mnt

# 查看 RAID 配置和设备使用情况
btrfs filesystem df /mnt
btrfs filesystem usage /mnt
```

输出示例：

```
Data, RAID0: total=200.00GiB, used=50.00GiB
    /dev/sda1  100.00GiB
    /dev/sdb1  100.00GiB
Metadata, RAID1: total=2.00GiB, used=256.00MiB
    /dev/sda1    2.00GiB
    /dev/sdb1    2.00GiB
```

确认 `Data` 行显示 `RAID0`，`Metadata` 行显示你指定的 profile。

---

## 4. 规划并创建子卷

### 4.1 推荐的子卷布局

```
Btrfs 文件系统顶层（不直接使用）
├── @              → 挂载到 /
├── @home          → 挂载到 /home
├── @var           → 挂载到 /var（隔离日志和缓存）
├── @swap          → 挂载到 /swap（如果需要交换文件）
└── @snapshots     → 挂载到 /.snapshots（快照存放）
```

**为什么要拆分子卷？**

| 子卷 | 隔离目的 | 典型场景 |
|------|---------|---------|
| `@` | 系统根 | `pacman -Syu` 前快照，滚挂回滚 |
| `@home` | 用户数据 | 重装系统时保留 `/home`，不丢失个人文件 |
| `@var` | 可变数据 | 日志爆满不影响系统快照，可单独设策略 |
| `@snapshots` | 快照自身 | 避免快照递归把自己快照进去 |

### 4.2 创建子卷

```bash
# 先挂载 Btrfs 文件系统顶层（top-level，subvolid=5）
mount /dev/sda1 /mnt

# 创建子卷
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@snapshots

# （可选）为交换文件准备子卷，禁用 CoW
btrfs subvolume create /mnt/@swap

# 卸载顶层
umount /mnt
```

> **命名约定**：`@` 前缀是 Ubuntu 和 openSUSE 的惯例，不是硬性要求。你可以用任何名称（如 `root`、`home` 等），但加 `@` 便于一眼分辨子卷。

### 4.3 按最终布局挂载子卷

```bash
# 挂载根子卷
mount -o compress=zstd,noatime,subvol=@ /dev/sda1 /mnt

# 创建挂载点并挂载其余子卷
mount --mkdir -o compress=zstd,noatime,subvol=@home /dev/sda1 /mnt/home
mount --mkdir -o compress=zstd,noatime,subvol=@var   /dev/sda1 /mnt/var
mount --mkdir -o compress=zstd,noatime,subvol=@snapshots /dev/sda1 /mnt/.snapshots
```

**挂载参数说明：**

| 参数 | 作用 | 备注 |
|------|------|------|
| `compress=zstd` | 透明压缩，默认级别 3 | 比 `compress=zstd:1` 压缩比更高 |
| `compress=zstd:1` | zstd 等级 1 | 速度优先，适合 NVMe SSD |
| `noatime` | 不更新文件访问时间 | 减少写操作，推荐开启 |
| `subvol=@` | 指定挂载的子卷 | 必须是已创建的子卷名 |
| `ssd` | 启用 SSD 优化 | 现代内核通常自动检测 |
| `discard=async` | 异步 TRIM | SSD 推荐，内核 6.2+ 支持 |

---

## 5. 快照与系统还原

### 5.1 核心认知：快照不穿透嵌套子卷

Btrfs 快照**只捕获目标子卷自身的数据**，不会自动包含嵌套挂载的其他子卷。

```
/dev/sda2  (Btrfs 文件系统顶层)
├── @              → /
│   └── home/      → 空目录（运行时 @home 挂载到这里）
│   └── var/       → 空目录（运行时 @var 挂载到这里）
├── @home          → /home
├── @var           → /var
└── @snapshots     → /.snapshots
```

当运行 `btrfs subvolume snapshot /mnt/@ /mnt/@snapshots/root-backup` 时：
- ✅ `@` 子卷的内容（`/usr`、`/etc`、`/boot` 等）
- ❌ `@home` 的内容（`/home/user` 等）— 这是另一个独立子卷
- ❌ `@var` 的内容（`/var/log` 等）— 同样独立

> 类比：`@`、`@home`、`@var` 是三本独立的书。给 `@` 拍快照等于翻拍这一本书，不会自动把旁边两本也拍进去。

### 5.2 创建"全系统"快照

要覆盖整个系统，必须**分别快照每个子卷**，保持同一时刻：

```bash
# 手动方式
DATE=$(date +%Y%m%d-%H%M%S)

btrfs subvolume snapshot -r /               /.snapshots/@-${DATE}
btrfs subvolume snapshot -r /home           /.snapshots/@home-${DATE}
btrfs subvolume snapshot -r /var            /.snapshots/@var-${DATE}
```

`-r` 创建只读快照，防止日后误修改。如果需要在快照中临时修改，去掉 `-r` 即可创建可读写快照。

#### 用 snapper 自动化

```bash
# 安装
pacman -S snapper

# 为根子卷创建配置
snapper -c root create-config /

# 创建快照（自动覆盖所有被 snapper 管理的子卷）
snapper -c root create -d "pacman -Syu 前全系统快照"

# 查看快照列表
snapper list
```

> **提示**：`snapper` 支持定时快照（每小时/每天/每周）和自动清理旧快照，详见 `/etc/snapper/configs/root`。

### 5.3 快照回滚（在线环境）

如果系统还能启动，直接从快照回滚：

```bash
# 查看现有子卷和快照
btrfs subvolume list /

# 进入 Btrfs 顶层操作
mount -o subvolid=5 /dev/sda2 /mnt

# 将当前 @ 改名备份（而非直接删除，保留回退路径）
mv /mnt/@ /mnt/@-broken

# 从健康快照创建新的 @（可读写，秒级完成）
btrfs subvolume snapshot /mnt/@snapshots/@-20260615-ok /mnt/@

# 同样处理 @home、@var
mv /mnt/@home /mnt/@home-broken
btrfs subvolume snapshot /mnt/@snapshots/@home-20260615-ok /mnt/@home

# 重启
reboot
```

### 5.4 快照还原（Live ISO 环境）

系统完全无法启动时，从 Arch ISO（或其他 Live USB）进入操作。

#### 场景设定

```
/.snapshots/
├── @-20260615-ok        ← 已知正常的快照（只读）
├── @home-20260615-ok
├── @var-20260615-ok
├── @-20260617-broken    ← 今天的快照（挂了）
├── @home-20260617-broken
└── @var-20260617-broken
```

#### 操作步骤

```bash
# 1. 挂载 Btrfs 顶层
mount /dev/sda2 /mnt

# 2. 查看所有子卷，确认快照存在
btrfs subvolume list /mnt

# 3. 将坏掉的 @ 改名备份（保留回退路径，而非直接删除）
mv /mnt/@ /mnt/@-broken

# 4. 从健康的只读快照创建新的可读写 @
btrfs subvolume snapshot /mnt/@snapshots/@-20260615-ok /mnt/@

# 5. 同样处理 @home、@var
mv /mnt/@home /mnt/@home-broken
btrfs subvolume snapshot /mnt/@snapshots/@home-20260615-ok /mnt/@home

mv /mnt/@var /mnt/@var-broken
btrfs subvolume snapshot /mnt/@snapshots/@var-20260615-ok /mnt/@var

# 6. 清理：确认系统正常后删除备份（释放空间）
# btrfs subvolume delete /mnt/@-broken
# btrfs subvolume delete /mnt/@home-broken

# 7. 卸载顶层
umount /mnt
```

> **为什么用 `mv` 改名而不是 `btrfs subvolume delete` 直接删除？**
> `mv` 是瞬时操作，且保留了旧子卷作为回退——万一操作失误或新快照也有问题，改回原名即可恢复。
> 确认系统启动正常后再删除 `@-broken` 释放空间。

#### 5.4.1 重新挂载系统并修复引导

```bash
# 挂载新恢复的子卷
mount -o compress=zstd,subvol=@ /dev/sda2 /mnt
mount --mkdir -o compress=zstd,subvol=@home /dev/sda2 /mnt/home
mount --mkdir -o compress=zstd,subvol=@var  /dev/sda2 /mnt/var
mount --mkdir /dev/sda1 /mnt/efi

# chroot
arch-chroot /mnt

# 重新生成 initramfs
mkinitcpio -P

# 更新 GRUB
grub-mkconfig -o /boot/grub/grub.cfg

# 退出并重启
exit
umount -R /mnt
reboot
```

### 5.5 还原路径对照

```
还原前（系统挂了）：              还原后（系统恢复）：

@ (坏的)          改名 ──→        @-broken (保留备查)
@home (坏的)      改名 ──→        @home-broken
@var (坏的)       改名 ──→        @var-broken

@ (新)            克隆自 ──→      @-20260615-ok (只读快照保留不变)
@home (新)        克隆自 ──→      @home-20260615-ok
@var (新)         克隆自 ──→      @var-20260615-ok

@snapshots        不动 ──→        @snapshots (所有历史快照完好)
```

> **关键**：从只读快照 `btrfs subvolume snapshot` 创建新子卷，新子卷默认**可读写**。基于 CoW，新旧 `@` 最初共享所有数据块——不占额外空间，直到你开始写新数据。

### 5.6 定时快照配置（snapper）

```bash
# 安装后自动启用定时器
systemctl enable --now snapper-timeline.timer
systemctl enable --now snapper-cleanup.timer

# 查看定时快照策略
cat /etc/snapper/configs/root
```

关键配置项：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `TIMELINE_CREATE` | yes | 是否启用定时快照 |
| `TIMELINE_MIN_AGE` | 1800 | 距上次快照的最小间隔（秒） |
| `TIMELINE_LIMIT_HOURLY` | 10 | 保留多少小时级快照 |
| `TIMELINE_LIMIT_DAILY` | 10 | 保留多少天级快照 |
| `TIMELINE_LIMIT_WEEKLY` | 0 | 保留多少周级快照 |
| `TIMELINE_LIMIT_MONTHLY` | 10 | 保留多少月级快照 |
| `TIMELINE_LIMIT_YEARLY` | 10 | 保留多少年级快照 |

---

## 6. 写入 fstab 实现开机自动挂载

### 6.1 获取 UUID

```bash
blkid -s UUID -o value /dev/sda1
# 输出示例: a1b2c3d4-5678-90ab-cdef-1234567890ab
```

### 6.2 fstab 条目

```bash
# /etc/fstab
UUID=a1b2c3d4-5678-90ab-cdef-1234567890ab  /              btrfs  compress=zstd,noatime,subvol=@            0  0
UUID=a1b2c3d4-5678-90ab-cdef-1234567890ab  /home          btrfs  compress=zstd,noatime,subvol=@home        0  0
UUID=a1b2c3d4-5678-90ab-cdef-1234567890ab  /var           btrfs  compress=zstd,noatime,subvol=@var         0  0
UUID=a1b2c3d4-5678-90ab-cdef-1234567890ab  /.snapshots    btrfs  compress=zstd,noatime,subvol=@snapshots   0  0
```

> **关键点**：所有子卷的 UUID **相同**（它们来自同一个 Btrfs 文件系统），依赖 `subvol=` 区分挂载目标。

### 6.3 验证 fstab

```bash
# 卸载所有，重新按 fstab 挂载，确认无误
umount -R /mnt
mount -a

# 或用 systemd 验证
systemd-fstab-generator /run/systemd/generator / /boot
ls /run/systemd/generator/*.mount
```

---

## 7. 扩容与维护

### 7.1 添加新磁盘

```bash
# 在线添加第三块盘
btrfs device add /dev/sdc /mountpoint

# 重新平衡数据到三盘（RAID0 条带会扩展到新盘）
btrfs balance start -dconvert=raid0 -mconvert=raid1 /mountpoint
```

> **Balance 耗时较长**，建议在低负载时段执行。可以分批进行，中断后下次执行会继续。

### 7.2 替换故障盘

```bash
# 在线替换（数据从旧盘迁移到新盘）
btrfs replace start /dev/sda1 /dev/sdc1 /mountpoint

# 查看替换进度
btrfs replace status /mountpoint
```

### 7.3 查看文件系统健康状态

```bash
# 设备信息
btrfs device stats /mountpoint

# 文件系统使用情况
btrfs filesystem usage /mountpoint

# 数据完整性校验（类似 RAID 的 "scrub"）
btrfs scrub start /mountpoint
btrfs scrub status /mountpoint
```

---

## 8. 交换文件（Swap File）

Btrfs 上创建 swap file 需要特殊处理：

```bash
# 在 @swap 子卷中创建
btrfs subvolume create /mnt/@swap
mount -o subvol=@swap /dev/sda1 /swap

# 创建空文件并禁用 CoW + 压缩
touch /swap/swapfile
chattr +C /swap/swapfile          # 禁用 CoW（swap 必须）
chmod 600 /swap/swapfile

# 填充并启用
dd if=/dev/zero of=/swap/swapfile bs=1M count=8192
mkswap /swap/swapfile
swapon /swap/swapfile

# fstab 条目
# /swap/swapfile  none  swap  defaults  0  0
```

---

## 9. 常见问题

### RAID0 真的能提升性能吗？

**能。** 两块独立的物理磁盘（即使是通过分区"切"出来的），各自有独立的 I/O 通道，Btrfs 条带化可以同时向两块盘读写。理论顺序速度翻倍，随机 IOPS 也等于两盘之和。

### 元数据 RAID1 会拖慢性能吗？

**基本不会。** 元数据体量很小（通常 1~5 GB），对整体带宽的影响可忽略。写入元数据时多一份副本的延迟也很微小。空间开销换来的安全收益远大于性能代价。

### swap 文件为什么必须禁用 CoW？

CoW 机制会导致 swap 的写入路径复杂化——内核需要直接操作磁盘块来换页，CoW 的"写时复制"语义与此冲突。`chattr +C` 让该文件绕过 CoW，直接原地覆写。

### 一块盘坏了，RAID0 数据能恢复吗？

**不能。** RAID0 是纯条带，没有冗余。一块坏 = 全部数据不可用。这就是为什么强烈建议元数据用 RAID1——至少文件系统的"目录树"结构还能读出来，能知道丢了哪些文件。

### 我能从 RAID0 迁移到 RAID1 吗？

可以，通过在线 balance 转换：

```bash
# 需要先添加足够的磁盘（总空间 ≥ 已用数据 × 2）
btrfs balance start -dconvert=raid1 -mconvert=raid1 /mountpoint
```

### Balance 和 Scrub 有什么区别？

| 操作 | 作用 | 适用场景 |
|------|------|---------|
| `scrub` | 校验所有数据的校验和，发现并修复错误 | 定期巡检（建议每月） |
| `balance` | 重新分配数据块到各设备，整理布局 | 添加/移除设备后，raid profile 转换 |

### 子卷可以嵌套吗？

可以。Btrfs 子卷像目录一样可以嵌套创建：

```bash
btrfs subvolume create /mnt/@home/alice
btrfs subvolume create /mnt/@home/bob
```

但注意：**父级子卷的快照不会包含子子卷的内容**。快照只捕获子卷自身的数据。

---

## 10. Windows 环境下使用 Btrfs（WinBtrfs）

> Btrfs 并非 Linux 专属。通过开源的 **WinBtrfs** 驱动，Windows 也能原生读写 Btrfs 分区，实现双系统数据共享。

### 10.1 WinBtrfs 简介

WinBtrfs（[github.com/maharmstone/btrfs](https://github.com/maharmstone/btrfs)）是一个 Windows 内核级文件系统驱动，由 Windows 内核开发者维护，Apache 2.0 协议开源。

安装后，Windows 会将 Btrfs 分区识别为普通分区并自动分配盘符，像 NTFS/exFAT 一样使用。

```powershell
# 安装（PowerShell 管理员）
winget install winbtrfs
# 或从 GitHub Releases 下载安装包手动安装
```

### 10.2 功能支持一览

| 功能 | 支持状态 | 备注 |
|------|---------|------|
| 基础读写 | ✅ 稳定 | 与 NTFS/exFAT 同等体验 |
| 透明压缩（zstd / lzo） | ✅ | 与 Linux 侧互通，无需额外配置 |
| RAID0 / RAID1 / RAID10 | ✅ | 文件系统层条带化，Windows 无感 |
| 子卷挂载 | ✅ | 可通过属性页管理 |
| 快照 | ✅ | 创建、删除、回滚均可操作 |
| 从 Btrfs 启动 Windows | ❌ 不支持 | 系统盘仍需 NTFS |
| send / receive | ❌ 不支持 | 增量备份功能不可用 |
| 校验和自修复 | 部分支持 | 检测损坏，修复依赖 RAID1 |

### 10.3 最佳场景：双系统共享数据盘

Linux / Windows 双系统下，传统方案各有限制：

| 方案 | Linux 侧 | Windows 侧 |
|------|---------|------------|
| NTFS 数据盘 | 内核 5.15+ 用 `ntfs3`，但无压缩/校验/快照 | 原生 |
| exFAT 数据盘 | 可用，但无权限、无压缩、无校验 | 原生 |
| **Btrfs + WinBtrfs** | **全部特性** | **读写可用，压缩/RAID 生效** |

**布局示例：**

```
/dev/sdb → Btrfs（单盘或 RAID0）
├── @shared-data    → Linux: /mnt/data
│                     Windows: D:\
├── @shared-media   → Linux: /mnt/media
│                     Windows: M:\
└── @snapshots      → 快照，恢复误删文件
```

### 10.4 透明压缩在 Windows 下的效果

WinBtrfs 完整支持 zstd 透明压缩，在 Linux 侧已启用压缩的 Btrfs 分区，Windows 侧读写时自动解压/压缩，应用层完全无感。

```powershell
# Windows 侧启用压缩（如果 Linux 侧未设）
btrfs property set D:\ compression zstd
```

**实测收益：**
- 文本/日志/源码类文件：压缩比 3:1 ~ 5:1
- 游戏资源包/虚拟机磁盘（已压缩格式）：压缩无效，但也不会膨胀
- NVMe SSD 上反而可能**加速**——写的数据量减小，I/O 瓶颈降低

### 10.5 RAID0 在 Windows 下的性能

WinBtrfs 直接在文件系统层处理条带化，Windows 本身不知道下面有多块物理盘：

```
Windows 应用层       程序读写 D:\ → 单盘视角
     ↓
WinBtrfs 驱动        条带化到 /dev/sda + /dev/sdb
     ↓
物理层               两块盘同时吞吐数据
```

| 场景 | 性能表现 |
|------|---------|
| 顺序读写大文件（视频、游戏打包） | ✅ 接近翻倍 |
| 多任务并发读写 | ✅ 各占各盘，互不拖慢 |
| 两块 SATA SSD 组 RAID0 | ⭐⭐⭐⭐ 顺序速度追上入门 NVMe |
| 一块 NVMe 再组 RAID0 | ⭐ 提升有限，PCIe 带宽先到瓶颈 |
| 同盘分区组 RAID0 | ❌ 负优化——磁头在同盘上来回寻道 |

### 10.6 注意事项

> **不要从 Btrfs 启动 Windows。** WinBtrfs 不能替代 NTFS 做系统盘。C 盘必须是 NTFS。
>
> **权限不互通。** Btrfs 不支持 Windows ACL，Windows 也不认 Linux uid/gid。通过 Linux 侧挂载选项 `uid=<你的uid>,gid=<你的gid>` 可对齐普通用户权限。
>
> **RAID5/RAID6 不要用。** 无论在 Linux 还是 Windows 侧，Btrfs RAID5/6 的 write hole 问题未解决，数据可靠性无保障。
>
> **只做数据盘，不做系统盘。** 这是 WinBtrfs 的设计定位——让 Btrfs 成为双系统间共享数据的桥梁。

---

## 参考

- [Arch Wiki: Btrfs](https://wiki.archlinux.org/title/Btrfs)
- [Btrfs 官方文档](https://btrfs.readthedocs.io/)
- [WinBtrfs](https://github.com/maharmstone/btrfs) — Windows 下的 Btrfs 驱动
- `man btrfs-filesystem`、`man btrfs-subvolume`、`man btrfs-balance`
