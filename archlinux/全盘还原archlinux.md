# 全盘还原 Arch Linux

> 前提：你有一份完整的 Linux 系统备份文件（如 `linux-backup.tar.zst` 或 `linux-backup.tar.gz`），
> 并通过 Arch 安装 ISO（U 盘）启动进入了 Live 环境。

## 1. 分区

### 1.1 查看现有分区

```bash
fdisk -l
```

确认目标磁盘（下文以 `/dev/sda` 为例，请替换为实际磁盘）。

### 1.2 创建分区（如果没有）

```bash
fdisk /dev/sda
```

**最少需要两个分区：**

| 分区 | 类型 | 建议大小 | 说明 |
|------|------|----------|------|
| `/dev/sda1` | EFI System (类型 `1`) | 512M ~ 1G | `/efi` 或 `/boot/efi`，**必须** |
| `/dev/sda2` | Linux filesystem (类型 `20`) | 剩余空间 | `/`，**必须** |

`/home` 分区可选，可按需创建 `/dev/sda3`。

## 2. 格式化与挂载

根据你备份时使用的文件系统，选择对应的方案。

### 2.1 ext4 方案

```bash
# 格式化
mkfs.ext4 /dev/sda2

# 格式化 EFI 分区（通常只需一次，如果已有则跳过）
mkfs.fat -F 32 /dev/sda1

# 挂载
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/efi
```

### 2.2 btrfs 方案（Flat 布局，无子卷）

```bash
# 格式化
mkfs.btrfs /dev/sda2
mkfs.fat -F 32 /dev/sda1

# 挂载（启用 zstd 压缩）
mount -o compress=zstd:1 /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/efi
```

### 2.3 btrfs 方案（子卷布局，推荐）

```bash
# 格式化
mkfs.btrfs /dev/sda2
mkfs.fat -F 32 /dev/sda1

# 先挂载顶层，创建子卷
mount /dev/sda2 /mnt
btrfs subvolume create /mnt/@          # 根子卷
btrfs subvolume create /mnt/@home      # /home 子卷
btrfs subvolume create /mnt/@log       # /var/log 子卷（可选）
btrfs subvolume create /mnt/@pkg       # /var/cache/pacman/pkg 子卷（可选）
btrfs subvolume create /mnt/@snapshots # 快照子卷（可选）
umount /mnt

# 挂载子卷
mount -o compress=zstd:1,subvol=@ /dev/sda2 /mnt
mount --mkdir -o compress=zstd:1,subvol=@home /dev/sda2 /mnt/home
mount --mkdir -o compress=zstd:1,subvol=@log /dev/sda2 /mnt/var/log
mount --mkdir -o compress=zstd:1,subvol=@pkg /dev/sda2 /mnt/var/cache/pacman/pkg
mount --mkdir -o compress=zstd:1,subvol=@snapshots /dev/sda2 /mnt/.snapshots

# 挂载 EFI
mount --mkdir /dev/sda1 /mnt/efi
```

## 3. 解压备份

```bash
# 根据备份压缩格式选择：

# gzip 压缩 (.tar.gz)
tar -xzvf linux-backup.tar.gz -C /mnt

# zstd 压缩 (.tar.zst)
tar --zstd -xvf linux-backup.tar.zst -C /mnt

# 或者让 tar 自动检测（推荐）
tar -xvf linux-backup.tar.zst -C /mnt
```

## 4. chroot 前的准备

### 4.1 重新生成 fstab

```bash
genfstab -U /mnt > /mnt/etc/fstab
```

> **注意**：用 `>` 覆盖，不要用 `>>` 追加，否则每次执行都会重复条目。

### 4.2 补充必要软件包（btrfs 用户）

如果使用 btrfs，确保 `btrfs-progs` 已安装：

```bash
pacstrap /mnt btrfs-progs
```

### 4.3 检查并修正 fstab（可选）

```bash
cat /mnt/etc/fstab
```

确认挂载选项与你之前的配置一致。如果使用子卷布局，检查 `subvol=` 是否正确。

## 5. chroot 进入系统

```bash
arch-chroot /mnt
```

以下操作均在 chroot 环境内执行。

## 6. 修复引导

### 6.1 重装内核（确保 initramfs 匹配当前硬件）

```bash
pacman -S linux
```

如果备份中已包含内核且版本匹配，可跳过。但通常建议重装以确保 `/boot` 下的文件完整。

### 6.2 重新生成 initramfs

```bash
mkinitcpio -P
```

### 6.3 安装 / 更新 GRUB

```bash
# 安装 GRUB 到 EFI
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB

# 生成 GRUB 配置
grub-mkconfig -o /boot/grub/grub.cfg
```

> **提示**：如果 EFI 分区挂载在 `/boot/efi`，将 `--efi-directory=/efi` 改为 `--efi-directory=/boot/efi`。

### 6.4 使用 systemd-boot 的替代方案

如果你的备份用的是 systemd-boot：

```bash
bootctl install
bootctl update
```

## 7. 退出并重启

```bash
exit                # 退出 chroot
umount -R /mnt      # 卸载所有挂载
reboot              # 重启
```

## 常见问题

- **重启后进入 GRUB rescue**：检查 `--efi-directory` 路径是否与实际挂载点一致，以及 EFI 分区是否已挂载后再执行 `grub-install`。
- **btrfs 相关错误**：确认 `btrfs-progs` 已安装，且 fstab 中 `subvol=` 参数正确。
- **fstab 不匹配**：Live 环境下的 `/dev/sda` 可能与目标系统的磁盘名不同（如 NVMe 为 `/dev/nvme0n1`），`genfstab -U` 使用 UUID 可避免此问题。
- **网卡不工作**：chroot 后运行 `pacman -S networkmanager`（或你使用的网络管理器）并启用服务。
