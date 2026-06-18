# Btrfs compsize 命令详解

> `compsize` 是 Btrfs 专属工具，用于精确计算透明压缩的实际效果。它能告诉你文件在磁盘上**真实占用**了多少空间，以及压缩为你省下了多少。

## 1. 为什么要用 compsize

在 Btrfs 上启用了透明压缩（zstd / lzo / zlib）后，传统工具会给你错误的信息：

| 工具 | 报告内容 | 反映压缩？ | 反映 CoW 共享？ |
|------|---------|-----------|----------------|
| `ls -lh` | 文件逻辑大小 | ❌ | ❌ |
| `du -sh` | 文件逻辑大小 × 硬链数 | ❌ | ❌ |
| `df -h` | 文件系统真实剩余 | ✅ | ✅ |
| `compsize` | 每个文件的真实磁盘占用 | ✅ | ✅ |

一个例子：

```bash
# 一个 10GB 的虚拟机镜像，Btrfs 挂载时启用了 compress=zstd
$ ls -lh vm.qcow2
-rw-r--r-- 1 user user 10G  Jun 18 12:00 vm.qcow2

$ du -sh vm.qcow2
10G     vm.qcow2

$ compsize vm.qcow2
Type       Perc     Disk Usage   Uncompressed Referenced
TOTAL       68%      6.8G          10G          10G
zstd        68%      6.8G          10G          10G
```

> `ls` 和 `du` 都告诉你 10GB，但磁盘上只占了 6.8GB。`df` 的剩余空间也对得上 6.8GB——`compsize` 帮你解释了为什么。

---

## 2. 安装

```bash
# Arch Linux
sudo pacman -S compsize

# Ubuntu / Debian
sudo apt install compsize

# Fedora
sudo dnf install compsize
```

无需任何配置，装完即用。

---

## 3. 基本用法

### 3.1 查看单个文件

```bash
compsize /path/to/file
```

输出示例：

```
Type       Perc     Disk Usage   Uncompressed Referenced
TOTAL       42%      4.2M          10M          10M
none       100%      1.0M          1.0M         1.0M
zstd        33%      3.2M          9.0M         9.0M
```

### 3.2 查看整个目录（递归）

```bash
compsize /mnt/@games
compsize /home
```

输出示例：

```
Type       Perc     Disk Usage   Uncompressed Referenced
TOTAL       57%       11G          19G          20G
none       100%      2.3G         2.3G         3.2G
zstd        35%      8.7G          17G          17G
```

### 3.3 常用选项

```bash
compsize -b /path         # 以字节为单位显示（默认自适应 KB/MB/GB）
compsize -x /path         # 不跨文件系统边界（类似 du -x）
compsize -bx /path        # 组合使用
```

---

## 4. 输出字段详解

这是理解 `compsize` 最核心的部分。

### 4.1 行（Type）

| Type | 含义 |
|------|------|
| `TOTAL` | 总计汇总行 |
| `none` | 未压缩的 extent |
| `zstd` | 使用 zstd 算法压缩的 extent |
| `lzo` | 使用 lzo 算法压缩的 extent |
| `zlib` | 使用 zlib 算法压缩的 extent |

一个文件可能同时包含多种类型的 extent——部分数据不可压缩（如已加密或已压缩格式），分区为 `none`；其余部分被 `zstd` 压缩。

### 4.2 列

| 列名 | 含义 | 通俗理解 |
|------|------|---------|
| **Perc** | `Disk Usage / Uncompressed × 100%` | 压缩后占用是原始大小的百分之几。42% 即省了 58% |
| **Disk Usage** | extent 在磁盘上的**真实物理占用**（压缩后大小） | 这就是「盘上实际占了多少字节」 |
| **Uncompressed** | 如果该 extent 没有压缩，会占多少空间 | 对比 Disk Usage 算出压缩率。**共享 extent 只计一次** |
| **Referenced** | 文件的**逻辑大小**（`ls -l` 看到的），所有文件指向该 extent 的总和 | 被 reflink 复制 3 份 = ×3 |

### 4.3 三个「大小」的关系

这是最容易混淆的地方。用一个具体例子说明：

```
假设你有一个 10MB 的文本文件，Btrfs 用 zstd 压缩写入了：

  Referenced   = 10M   ← ls -l 看到的值
  Uncompressed = 10M   ← 如果没有压缩，这个 extent 本身是 10MB
  Disk Usage   = 3.0M  ← 压缩后盘上只写了 3MB

Perc = 3.0M / 10M = 30%
```

现在用 `cp --reflink=always` 复制 3 份（CoW 共享同一个 extent）：

```
  Referenced   = 30M   ← 3 份文件，每份 10MB 逻辑大小，合计 30MB
  Uncompressed = 10M   ← extent 本身只有一个，不压缩的话是 10MB（不重复计！）
  Disk Usage   = 3.0M  ← extent 在盘上只占 3MB（不重复计！）

Perc = 3.0M / 10M = 30%
```

> **Uncompressed 和 Disk Usage 是「物理层」的——它们计算 extent 本身，共享的 extent 不重复计入。Referenced 是「逻辑层」的——计算的是所有指向该 extent 的文件的逻辑大小之和。**

这意味着：
- 如果大量文件通过 reflink 共享 extent，`Referenced` 会远大于 `Disk Usage`
- 如果文件没有压缩也没有 reflink 共享，三个值完全相等

---

## 5. 实用场景

### 5.1 评估压缩收益率

```bash
# 看看压缩到底省了多少空间
compsize /mnt/@games

# 示例输出：
# TOTAL       65%       65G         100G         100G
# → 压缩省了 35GB
```

### 5.2 检查哪些文件没有被压缩

```bash
compsize /mnt/@data | grep '^none'
```

如果 `none` 行的比例意外地大，可能原因：

| 原因 | 排查方法 |
|------|---------|
| 文件或目录设了 `chattr +C`（nodatacow） | `lsattr /path/to/file` 查看是否有 `C` 标志 |
| 挂载时用了 `nodatacow` 选项 | `mount \| grep btrfs` 检查挂载参数 |
| 文件是 incompressible 格式 | 已加密文件、已压缩格式（`.jpg`、`.mp4`、`.zip`）压缩无效，Btrfs 会跳过 |
| 压缩算法不匹配 | 文件是用 `lzo` 压缩的，当前挂载参数是 `compress=zstd`，Btrfs 不会自动转换 |

### 5.3 对比压缩算法转换前后

```bash
# 转换前：记录基准
compsize /mnt/data > before.txt

# 从 lzo 转换到 zstd（原地重压缩）
btrfs filesystem defragment -r -czstd /mnt/data

# 转换后：对比收益
compsize /mnt/data > after.txt
diff before.txt after.txt
```

### 5.4 验证 nodatacow 是否生效

```bash
# 如果目录已设 chattr +C，所有文件应在 'none' 行
compsize /mnt/@steam

# 预期：TOTAL 的 Perc 接近 100%，全部在 'none' 行
```

### 5.5 找出压缩率最高的文件

compsize 本身不提供逐文件细分，但可以写脚本遍历：

```bash
# 找出当前目录下压缩最狠的文件（按压缩率排序）
for f in *; do
  [ -f "$f" ] || continue
  compsize "$f" | awk -v name="$f" '/TOTAL/{print $2, name}' 
done | sort -n | head -20
```

输出示例：

```
12%  access.log
18%  debug.log
25%  config.json
...
```

---

## 6. 原理简述

`compsize` **不读取文件内容**，而是直接查询 Btrfs 文件系统的元数据：

```
用户态                        内核态
compsize                      
  │                           
  ├─ BTRFS_IOC_TREE_SEARCH ──→ extent tree
  │   (ioctl)                  ├─ extent_item: 压缩算法、压缩后大小
  │                            ├─ extent_ref:   引用计数（几份文件指向它）
  │                            └─ ...
  │   ←── 返回 extent 元数据 ──┤
  │                           
  └─ 汇总统计 → 输出
```

每个 extent 在 Btrfs 的 `extent tree` 中记录了：

| 元数据字段 | 对应 compsize 输出 |
|-----------|-------------------|
| `compression` | Type（none / zstd / lzo / zlib） |
| `size`（extent 逻辑大小） | Uncompressed |
| `disk_num_bytes`（盘上实际占用） | Disk Usage |
| 所有指向该 extent 的文件逻辑大小之和 | Referenced |

> 因为只读元数据，不碰文件内容，`compsize` 跑得非常快——即使扫描几百 GB 的目录，也只需要几秒。

---

## 7. 与 beets / btdu 的对比

Btrfs 生态中还有其他分析工具，各有侧重：

| 工具 | 定位 | 输出 |
|------|------|------|
| `compsize` | 压缩率统计 | 按压缩类型汇总的磁盘占用 |
| `btdu` | 磁盘使用分析 | 交互式 TUI，按 extent 类型、子卷、路径逐层展开 |
| `btrfs filesystem du` | Btrfs 专用 du | 按子卷显示独占/共享空间 |
| `btrfs filesystem usage` | 文件系统整体概览 | 已分配/已用/剩余、RAID profile 分布 |

```bash
# btdu 可以交互式深入到每个子卷/目录看压缩率
sudo btdu /mnt

# 快速看整个文件系统用了多少、剩了多少
btrfs filesystem usage /mnt
```

---

## 8. 常见问题

### Q1: compsize 显示 100% 但我开了压缩？

**答：** 可能原因依次排查：

1. **文件本身不可压缩**——`file /path/to/file` 看看是不是已压缩格式
2. **挂载选项没生效**——`mount | grep btrfs` 确认有 `compress=zstd`
3. **文件创建时未设置压缩**——Btrfs 不会自动重压缩已有文件，需要 `btrfs filesystem defragment -czstd` 手动触发
4. **文件设置了 nodatacow**——`lsattr /path/to/file`，有 `C` 标志则跳过压缩

### Q2: compsize 和 df 报告的大小对不上？

**答：** `compsize` 报告的 `Disk Usage` 只包括数据 extents，不包括：

- 元数据（extent tree、csum tree 等）
- 块分配开销（chunk 内未写满的空间）
- RAID 冗余（如 RAID1 会翻倍）

`df` 报告的文件系统总占用 = 数据 + 元数据 + 分配开销 + RAID 冗余。所以 `compsize` 的总计通常比 `df` 的已用空间小。

### Q3: 为什么 Referenced 比 Disk Usage 大很多？

**答：** 三个可能原因，从常见到罕见：

1. **压缩**——最直接的原因。`Perc` 远小于 100% 时就是它。
2. **reflink 复制**——用 `cp --reflink` 或快照复制了大量文件，CoW 共享 extent。用 `btrfs filesystem du` 可看清独占 vs 共享。
3. **两者叠加**——reflink 共享 + 压缩同时生效。

### Q4: compsize 需要 root 权限吗？

**答：** 一般情况下**不需要**——读取 extent 元数据不需要特权。但如果目录权限阻止你访问某些文件，`compsize` 会跳过它们并打印 `Permission denied`。

### Q5: 如何让 compsize 只统计某一类压缩算法？

**答：** `compsize` 本身不支持按压缩类型过滤，但可以用 `grep`：

```bash
# 只看 zstd 压缩的文件占比
compsize /mnt/data | grep 'zstd'

# 只看未压缩的
compsize /mnt/data | grep 'none'
```

---

## 参考

- [compsize GitHub](https://github.com/kilobyte/compsize)
- [Arch Wiki: Btrfs - Compression](https://wiki.archlinux.org/title/Btrfs#Compression)
- [Btrfs 官方文档: Compression](https://btrfs.readthedocs.io/en/latest/Compression.html)
- `man 8 compsize`
