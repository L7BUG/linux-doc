---
name: linux-doc-write-guide
description: "linux-doc 仓库文档写作全流程：调研→tools/ 命名→写文档→README 索引→提交推送"
user-invocable: false
origin: auto-extracted
---

# linux-doc 文档写作工作流

**Extracted:** 2026-08-03
**Context:** 用户要求"把 X 写成文档/介绍下 X"时，往 linux-doc 仓库新增技术文档

## Problem

用户在本仓库持续沉淀技术文档，但每次写作的流程约定（命名、索引、提交）散落在各次对话中，不沉淀则每次重新摸索。

## Solution

1. **调研**：先读 `~/.claude/plugins/cache/ecc/` 或系统资源获取真实信息，不凭记忆编造
2. **命名**：`tools/<主题>-完全指南.md` / `-详解.md` / `-入门指南.md`（参考 tailscale-完全指南.md）
3. **写作**：遵循 CLAUDE.md 约定——简体中文、编号层级 `## 1.`、表格对比、引用块提示、末尾**常见问题**章节
4. **索引**：README.md "工具"表格末尾追加一行（`[标题](tools/文件名.md) | 一句话简介`）
5. **提交**：`git add` + commit `docs: 新增 <标题>` + push（用户确认后）

## When to Use

- 用户说"写个文档给我看看 / 详细介绍下 X 并写进文档"
- 在 linux-doc 仓库新增任何 .md 文件
