# Linux 技术文档

以 Arch Linux 为基线的运维笔记，涵盖系统备份还原、文件系统、压缩工具等实用主题。

## 文档索引

### Arch Linux 运维
| 文档 | 简介 |
|------|------|
| [全盘还原 Arch Linux](archlinux/全盘还原archlinux.md) | 从 Live ISO（U 盘）环境出发，通过备份文件（`.tar.zst`）完整还原 Arch Linux 系统，含分区、挂载、chroot、引导修复全流程 |

### Btrfs 文件系统
| 文档 | 简介 |
|------|------|
| [写时复制（Copy-on-Write）详解](btrfs/copy-on-write-详解.md) | 从内核 fork()、mmap、Btrfs 文件系统三重视角拆解 CoW 原理，性能权衡，日志 vs CoW 范式对比 |
| [Btrfs RAID0 与子卷实战指南](btrfs/btrfs-raid0-子卷.md) | Btrfs 组建 RAID0 提升性能，子卷布局规划，快照/回滚/还原操作，snapper 定时快照，WinBtrfs 双系统共享数据 |
| [compsize 命令详解](btrfs/btrfs-compsize-详解.md) | 精确计算 Btrfs 透明压缩的实际效果，Disk Usage / Uncompressed / Referenced 字段辨析，验证 nodatacow 是否生效 |

### 压缩工具
| 文档 | 简介 |
|------|------|
| [tar zstd 压缩解压完全指南](tools/tar-zstd-guide.md) | Zstandard 压缩算法入门，tar 配合 zstd 的常用操作，与 gzip/bzip2/xz 的对比选型 |

## 目录结构

```
linux-doc/
├── archlinux/    ← Arch Linux 运维
├── btrfs/        ← Btrfs 文件系统专题
├── tools/        ← 压缩等工具使用
├── README.md
└── CLAUDE.md
```

## 技术栈

- **系统**: Arch Linux
- **文件系统**: Btrfs（RAID0、子卷、快照、CoW、透明压缩）
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
