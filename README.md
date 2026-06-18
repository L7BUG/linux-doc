# Linux 技术文档

以 Arch Linux 为基线的运维笔记，涵盖系统备份还原、文件系统、压缩工具等实用主题。

## 文档索引

| 文档 | 简介 |
|------|------|
| [全盘还原 Arch Linux](全盘还原archlinux.md) | 从 Live ISO（U 盘）环境出发，通过备份文件（`.tar.zst`）完整还原 Arch Linux 系统，含分区、挂载、chroot、引导修复全流程 |
| [写时复制（Copy-on-Write）详解](docs/copy-on-write-详解.md) | 从内核 fork()、mmap、Btrfs 文件系统三重视角拆解 CoW 原理，性能权衡，日志 vs CoW 范式对比 |
| [Btrfs RAID0 与子卷实战指南](btrfs-raid0-子卷.md) | Btrfs 组建 RAID0 提升性能，子卷布局规划，快照/回滚/还原操作，snapper 定时快照，WinBtrfs 双系统共享数据 |
| [tar zstd 压缩解压完全指南](tar-zstd-guide.md) | Zstandard 压缩算法入门，tar 配合 zstd 的常用操作，与 gzip/bzip2/xz 的对比选型 |

## 技术栈

- **系统**: Arch Linux
- **文件系统**: Btrfs（RAID0、子卷、快照）
- **压缩**: Zstandard (zstd)、tar
- **引导**: GRUB、EFI

## 文档约定

- 使用**简体中文**撰写
- 技术术语、命令、代码块保留原始语言（英文）
- 代码块标注语言类型（`bash`、`sh`、`powershell` 等）
- 命令示例优先使用 `-a`（auto-compress）简写形式
- 每个文档末尾包含**常见问题**章节
- 命令行中的占位参数使用 `/path/to/...` 形式

## 参考

- [Arch Wiki](https://wiki.archlinux.org/)
- [Btrfs 官方文档](https://btrfs.readthedocs.io/)
- [Zstandard](https://facebook.github.io/zstd/)
