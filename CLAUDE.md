# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个 Linux 技术文档仓库，主要涵盖 Arch Linux 运维、备份还原、Btrfs 文件系统、压缩工具以及 Linux 基础标准规范。

## 目录结构

```
linux-doc/
├── archlinux/    ← Arch Linux 运维（系统备份还原等）
├── btrfs/        ← Btrfs 文件系统专题（RAID0、子卷、CoW、压缩工具等）
├── tools/        ← 工具使用指南（按主题细分）
│   ├── dev/      ← 开发/编程（python、rust、vim、ecc 等）
│   ├── ai/       ← AI 工具（LLM 会话、Hermes 等）
│   ├── network/  ← 网络/远程（ssh、tailscale、caddy 等）
│   └── system/   ← 系统/桌面（flatpak、wine、tar-zstd 等）
├── standards/    ← Linux 标准与基础规范（XDG 等）
├── README.md
└── CLAUDE.md
```

新建文档时，按主题放入对应子目录。

## 文档约定

- 所有文档使用**简体中文**撰写
- 技术术语、命令、代码块保留原始语言（英文）
- 使用 GitHub Flavored Markdown 格式
- 代码块标注语言类型（bash, sh 等）
- 命令示例优先使用 `-a`（auto-compress）简写形式

## 文档风格

- 操作步骤使用编号层级（`## 1.`, `### 1.1`）
- 表格用于对比选择（方案对比、参数对比、场景推荐）
- 引用块（`>`）用于提示、注意、警告信息
- 命令行中的占位参数使用 `/path/to/...` 形式
- 每个文档末尾包含**常见问题**章节