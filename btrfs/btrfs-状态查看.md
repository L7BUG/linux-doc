# Btrfs 状态查看与健康检查

> 本文聚焦「如何查看 Btrfs 的运行状态」——文件系统用量、设备健康、数据校验、子卷布局、挂载与压缩确认。压缩率的**精确测量**请见 [compsize 命令详解](./btrfs-compsize-详解.md)，本文只做速查。

> **约定：** 下文以本机真实环境为例——根文件系统 `/` 位于 `/dev/nvme0n1p6`，挂载参数含 `compress=zstd:1`。你的设备名和数值可能不同，替换即可。标注「示例输出」的代码块为演示值，非你机器的真实数字。

---

## 1. 文件系统总览

### 1.1 有哪些 Btrfs、由哪些设备组成

```bash
btrfs filesystem show          # 无需 root
```

示例输出：

```
Label: none  uuid: 3f8b0c2a-...-9d1e
	Total devices 1 FS bytes used 48.21GiB
	devid    1 size 100.00GiB used 55.00GiB path /dev/nvme0n1p6
```

| 字段 | 含义 |
|------|------|
| `Total devices` | 该文件系统由几块设备组成（单盘为 1） |
| `FS bytes used` | 文件系统真实使用量（含数据 + 元数据） |
| `size` | 设备总容量 |
| `used` | 已**分配**给 Btrfs 的块大小（≥ 实际写入量，见 1.2） |

### 1.2 空间到底用了多少（df 为什么不够用）

`df -h` 只给一个笼统的已用/剩余，看不到 Btrfs 的**分配 vs 实占**差异，也看不到元数据开销。看用量用 `usage`：

```bash
sudo btrfs filesystem usage /
```

示例输出：

```
Overall:
    Device size:                 100.00GiB
    Device allocated:             55.00GiB
    Device unallocated:           45.00GiB
    Used:                         48.21GiB
    Free (estimated):             50.15GiB      (min: 27.65GiB)

Data,single:        Size:50.00GiB, Used:45.10GiB (90.20%)
Metadata,DUP:       Size:4.00GiB,  Used:3.10GiB  (77.50%)
System,DUP:         Size:32.00MiB, Used:16.00KiB (0.05%)
```

| 字段 | 含义 |
|------|------|
| **Device allocated** | Btrfs 已从裸设备**划走**的块（划走 ≠ 写满） |
| **Device unallocated** | 尚未划分的裸空间——真正的「余量池」 |
| **Used** | 这些块里**实际写入**的数据量 |
| **Free (estimated)** | 预估可再写入量；`min` 是元数据全用 DUP 时的保守下限 |
| **Data,single** | 数据块，单副本（单盘默认） |
| **Metadata,DUP** | 元数据块，同盘存两份（默认，防单点元数据损坏） |

> **关键认知：** `Device allocated` 涨上去但 `Used` 不高是正常现象——块一旦划给 Data 就不会自动还给 Metadata。只有当 `Device unallocated` 逼近 0 时才需要 `balance`（见下方常见问题）。

### 1.3 分配概览（快速版）

```bash
sudo btrfs filesystem df /
```

比 `usage` 更简短，只看 Data / Metadata / System 各自的 Size 和 Used，适合一眼扫。

---

## 2. 设备健康状态（最该定期看的）

### 2.1 设备错误统计

Btrfs 会累计每块设备的 I/O 与损坏错误，这是判断盘是否健康的第一手信息：

```bash
sudo btrfs device stats /
```

示例输出（健康时全为 0）：

```
[/dev/nvme0n1p6].write_io_errs     0
[/dev/nvme0n1p6].read_io_errs      0
[/dev/nvme0n1p6].flush_io_errs     0
[/dev/nvme0n1p6].corruption_errs   0
[/dev/nvme0n1p6].generation_errs   0
```

| 字段 | 含义 | 非 0 意味着 |
|------|------|-------------|
| `write_io_errs` | 写操作失败次数 | 盘可能有坏块或接口/供电问题 |
| `read_io_errs` | 读操作失败次数 | 同上，读取路径异常 |
| `flush_io_errs` | flush（落盘同步）失败次数 | 掉电保护/固件问题风险 |
| `corruption_errs` | 校验和不匹配（检测到数据损坏） | **最需警惕**——静默损坏被 Btrfs 抓到了 |
| `generation_errs` | generation 号不一致（元数据版本错乱） | 元数据一致性问题，常伴随异常掉电 |

> **全 0 = 目前健康。** 只要有一项非 0，就应尽快 `scrub`（见 2.2）+ 备份重要数据，并用 `smartctl -a /dev/nvme0n1` 查 SSD 自身健康。

累计计数确认无误、想重新计数时可清零：

```bash
sudo btrfs device stats -z /     # -z: 读取后归零
```

### 2.2 数据校验：scrub

`scrub` 会读取整个文件系统的所有数据与元数据，用校验和逐一验证；单盘能**发现**损坏，多盘（RAID1/10）还能**自动修复**。

```bash
sudo btrfs scrub start /         # 后台启动
sudo btrfs scrub status /        # 查看进度 / 结果
```

`status` 示例输出：

```
UUID:             3f8b0c2a-...-9d1e
Scrub started:    Mon Jul 13 10:00:00 2026
Status:           finished
Duration:         0:03:12
Total to scrub:   48.21GiB
Rate:             257.00MiB/s
Error summary:    no errors found
```

| 关键行 | 看什么 |
|--------|--------|
| `Status` | `running` / `finished` / `aborted` |
| `Error summary` | `no errors found` 即健康；出现 `csum errors` 需立即处理 |

> **SSD 也建议定期 scrub**（如每月一次），它能提前暴露静默损坏。自动化见常见问题 Q3。

---

## 3. 子卷与快照状态

### 3.1 列出所有子卷

```bash
sudo btrfs subvolume list /
```

示例输出：

```
ID 256 gen 1200 top level 5 path @snapshots
ID 257 gen 1198 top level 5 path .snapshots/1
```

> **注意：** 本机根挂载为顶层子卷（`subvolid=5`，即 `subvol=/`），没有采用 `@`/`@home` 布局，因此子卷列表可能很简单甚至为空。这不影响使用，只是快照管理不如 `@` 布局规整。

### 3.2 查看某子卷详情

```bash
sudo btrfs subvolume show /
```

会显示 UUID、Creation time、Snapshot(s)、以及 `Flags`（是否 `readonly`）等——判断一个子卷是不是只读快照就看这里。

---

## 4. 挂载与压缩状态

### 4.1 确认挂载参数（压缩是否真的开着）

```bash
findmnt -t btrfs                 # 无需 root，最直观
# 或
cat /proc/mounts | grep btrfs
```

本机实际输出：

```
TARGET SOURCE         FSTYPE OPTIONS
/      /dev/nvme0n1p6 btrfs  rw,noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvolid=5,subvol=/
```

关键选项解读：

| 选项 | 含义 |
|------|------|
| `compress=zstd:1` | 透明压缩已开启，算法 zstd，**等级 1** |
| `ssd` | SSD 优化的块分配策略 |
| `discard=async` | 异步 TRIM，回收已删除块（对 SSD 友好） |
| `space_cache=v2` | 空闲空间缓存 v2（大文件系统挂载更快） |
| `noatime` | 不更新文件访问时间，减少写放大 |
| `subvolid=5,subvol=/` | 挂载的是顶层子卷 |

> **关于等级：** zstd 在 Btrfs 支持 `1`–`15`，默认 `3`。`zstd:1` 偏向**速度**、压缩率最低；若你更看重省空间、CPU 有富余，可在 `/etc/fstab` 改成 `compress=zstd:3` 再重挂载。注意：改选项只对**之后写入**的数据生效，已有文件需手动重压缩（见 4.2）。

### 4.2 一眼看压缩率（详见 compsize 专文）

确认某目录实际压缩效果：

```bash
compsize /path/to/dir
```

示例输出：

```
Type       Perc     Disk Usage   Uncompressed Referenced
TOTAL       57%       11G          19G          20G
none       100%      2.3G         2.3G         3.2G
zstd        35%      8.7G          17G          17G
```

`Perc` 即「压缩后占原始的百分之几」，57% 表示省了 43%。想给已有文件补压缩：

```bash
sudo btrfs filesystem defragment -r -czstd /path/to/dir
```

> 字段辨析（Disk Usage / Uncompressed / Referenced）、CoW 共享对数字的影响、按压缩率排序找文件等，**完整内容见 [compsize 命令详解](./btrfs-compsize-详解.md)**，此处不重复。

---

## 5. 常用状态命令速查表

| 命令 | 作用 | 需 root |
|------|------|---------|
| `btrfs filesystem show` | 文件系统与设备总览 | 否 |
| `btrfs filesystem usage /` | 空间分配 vs 实占（推荐） | 是 |
| `btrfs filesystem df /` | 数据/元数据分配概览 | 是 |
| `btrfs device stats /` | 设备错误累计（健康检查） | 是 |
| `btrfs scrub start / status /` | 校验数据完整性 | 是 |
| `btrfs subvolume list /` | 列出子卷/快照 | 是 |
| `btrfs subvolume show /` | 子卷详情（含只读标志） | 是 |
| `findmnt -t btrfs` | 确认挂载参数与压缩选项 | 否 |
| `compsize /path` | 查看透明压缩率 | 否 |

---

## 6. 常见问题

### Q1: `df -h` 和 `btrfs filesystem usage` 的数字对不上？

**答：** 正常。`df` 不理解 Btrfs 的「分配 vs 实占」和元数据 DUP。以 `btrfs filesystem usage` 为准——它区分了 `Device allocated`（已划走的块）、`Used`（实际写入）和 `Device unallocated`（真正余量）。

### Q2: `device stats` 里某个错误计数不为 0，严重吗？

**答：** 分情况：

1. **先看是哪一项**——`corruption_errs` / `generation_errs` 最需警惕（数据/元数据损坏）；I/O 类错误多为盘或接口问题。
2. **立即 `scrub`**——`sudo btrfs scrub start /`，看 `Error summary`。单盘只能发现损坏；多盘 RAID1/10 可自动修复。
3. **备份 + 查盘**——`smartctl -a /dev/nvme0n1` 看 SSD 健康度（媒体错误、寿命）。
4. **确认后清零**——`sudo btrfs device stats -z /`，之后观察是否再增长。计数持续上涨说明硬件在恶化。

### Q3: 怎么让系统定期自动 scrub？

**答：** btrfs-progs 自带 systemd 定时器（Arch 上随 `btrfs-progs` 提供）：

```bash
# 对根文件系统启用每月定时 scrub
sudo systemctl enable --now btrfs-scrub@-.timer

# 查看已启用的 scrub 定时器
systemctl list-timers 'btrfs-scrub@*'
```

> 模板实例名里的 `-` 代表挂载点 `/`（systemd 转义规则）。其他挂载点如 `/home` 对应 `btrfs-scrub@home.timer`。

### Q4: 哪些状态命令需要 root？

**答：** 只读「看一眼」的 `btrfs filesystem show`、`findmnt`、`compsize` 通常**不需要**；而 `usage`、`df`、`device stats`、`subvolume list`、`scrub` 需要 root（访问文件系统内部结构）。

### Q5: 我挂的是 `compress=zstd:1`，怎么确认老文件到底压没压？

**答：** 挂载选项只影响**之后写入**的数据，改选项前的老文件保持原状。用 `compsize /path` 看某目录的真实压缩分布——若 `none` 行占比很大，多半是老文件未压缩或本身不可压缩。补压缩：`sudo btrfs filesystem defragment -r -czstd /path`。详见 [compsize 命令详解](./btrfs-compsize-详解.md)。

---

## 参考

- [Btrfs 官方文档: btrfs-filesystem](https://btrfs.readthedocs.io/en/latest/btrfs-filesystem.html)
- [Btrfs 官方文档: btrfs-device](https://btrfs.readthedocs.io/en/latest/btrfs-device.html)
- [Btrfs 官方文档: btrfs-scrub](https://btrfs.readthedocs.io/en/latest/btrfs-scrub.html)
- [Arch Wiki: Btrfs](https://wiki.archlinux.org/title/Btrfs)
- 本仓库：[compsize 命令详解](./btrfs-compsize-详解.md)、[Copy-on-Write 详解](./copy-on-write-详解.md)
