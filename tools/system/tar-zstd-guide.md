# tar zstd 压缩解压完全指南

## 1. 什么是 zstd？

**Zstandard (zstd)** 是 Facebook 开发的一款现代压缩算法，核心特点：

| 特性 | 说明 |
|------|------|
| **极快解压** | 解压速度远超 gzip/bzip2/xz |
| **可调压缩率** | 1~19 级（甚至支持 --ultra -20 到 -22） |
| **压缩比优秀** | 接近 xz，远好于 gzip |
| **流式支持** | 适合管道处理和实时压缩 |

**一句话总结：** 压缩速度和 xz 相当（或略快），解压速度比肩 gzip，压缩率接近 xz。

## 2. 基本用法

### 2.1 压缩（创建 .tar.zst 归档）

```bash
# 基本用法
tar --zstd -cf archive.tar.zst /path/to/dir

# 简写（推荐）
tar -caf archive.tar.zst /path/to/dir
```

**参数说明：**
- `-c` — create，创建归档
- `-a` — auto-compress，**根据后缀名自动选择压缩算法**（`.tar.zst` 自动用 zstd）
- `-f` — file，指定归档文件名

### 2.2 解压

```bash
# 基本用法
tar --zstd -xf archive.tar.zst

# 简写（推荐）
tar -xaf archive.tar.zst

# 解压到指定目录
tar -xaf archive.tar.zst -C /target/dir
```

### 2.3 列出归档内容（不解压）

```bash
tar -tf archive.tar.zst

# 详细信息（类似 ls -l）
tar -tvf archive.tar.zst
```

## 3. 压缩级别控制

zstd 默认压缩级别是 **3**。有两种设置方式：

### 3.1 方式一：`-I` 指定压缩程序（推荐）

`-I`（即 `--use-compress-program`）直接在命令行里传入压缩器和参数，**最灵活**：

```bash
# 基本格式
tar -I 'zstd -10' -cf archive.tar.zst /path/to/dir

# 快速压缩（级别 1，速度优先）
tar -I 'zstd -1' -cf fast.tar.zst /path/to/dir

# 默认级别（级别 3，均衡）—— 可省略 -3
tar -I zstd -cf normal.tar.zst /path/to/dir

# 高压缩（级别 10，体积优先）
tar -I 'zstd -10' -cf small.tar.zst /path/to/dir

# 极限压缩（级别 19，最慢但最小）
tar -I 'zstd -19' -cf tiny.tar.zst /path/to/dir

# 终极压缩（--ultra 模式，级别 20-22，需要更多内存）
tar -I 'zstd --ultra -22' -cf ultra.tar.zst /path/to/dir

# 组合：压缩级别 + 多线程 + 进度显示
tar -I 'zstd -15 -T0 -v' -cf archive.tar.zst /path/to/dir
```

> ⚠️ **注意：** `-I` 和 `-a`（auto-compress）**不能同时使用**。`-I` 已接管压缩过程，`-a` 会按后缀再次选择压缩器导致冲突。所以写 **`-cf`** 而非 `-caf`。

### 3.2 方式二：环境变量

通过 `ZSTD_CLEVEL` 设置级别，配合 `-a` 自动识别 `.tar.zst` 后缀：

```bash
# 级别 1
ZSTD_CLEVEL=1 tar -caf fast.tar.zst /path/to/dir

# 级别 10
ZSTD_CLEVEL=10 tar -caf small.tar.zst /path/to/dir

# 级别 19
ZSTD_CLEVEL=19 tar -caf tiny.tar.zst /path/to/dir

# --ultra 模式（级别 20-22）
ZSTD_CLEVEL=22 tar -caf ultra.tar.zst /path/to/dir
```

### 3.3 三种写法对照

| 方式 | 示例 | 优点 | 缺点 |
|------|------|------|------|
| `-I 'zstd -N'` | `tar -I 'zstd -10' -cf a.tar.zst dir` | 灵活，可传任意参数 | 不能和 `-a` 并用 |
| `ZSTD_CLEVEL=N` | `ZSTD_CLEVEL=10 tar -caf a.tar.zst dir` | 配合 `-a` 自动选压缩器 | 只能设级别，不能传其他参数 |
| `tar --zstd` | `tar --zstd -cf a.tar.zst dir` | 简洁 | 只能用默认级别，不可调参 |

### 常用压缩级别建议

| 级别 | 速度 | 压缩率 | 适用场景 |
|------|------|--------|----------|
| 1 | 极快 | 一般 | CI/CD 构建产物、临时备份 |
| 3 | 快（默认） | 良好 | **日常使用首选** |
| 5-7 | 中等 | 较好 | 网络传输、发布包 |
| 10-15 | 较慢 | 很好 | 长期归档存储 |
| 19 | 很慢 | 最佳 | 一次性归档、分发 |
| 20-22 | 极慢 | 极致 | 冷存储（需大量内存） |

## 4. 多线程加速

zstd 支持多线程压缩（`-T` 参数），同样分两种写法：

### 4.1 `-I` 方式（推荐）

```bash
# 自动使用全部 CPU 核心
tar -I 'zstd -T0' -cf fast.tar.zst /path/to/dir

# 指定 4 个线程
tar -I 'zstd -T4' -cf fast.tar.zst /path/to/dir

# 压缩级别 + 线程 一步到位
tar -I 'zstd -10 -T0' -cf parallel.tar.zst /path/to/dir

# 解压也可多线程（-d 解压，-c 输出到 stdout）
tar -I 'zstd -d -T0' -xf archive.tar.zst
```

### 4.2 环境变量方式

```bash
# 多线程压缩
ZSTD_NBTHREADS=0 tar -caf fast.tar.zst /path/to/dir

# 指定 4 个线程
ZSTD_NBTHREADS=4 tar -caf fast.tar.zst /path/to/dir

# 同时设置压缩级别和线程数
ZSTD_CLEVEL=5 ZSTD_NBTHREADS=0 tar -caf parallel.tar.zst /path/to/dir
```

> **注意：** 多线程压缩会**略微降低压缩率**（不同线程处理的数据块独立），但对大文件加速显著。解压时多线程效果不明显，因为解压本来就很快。

## 5. 与其他格式对比

```bash
# 同一目录，不同压缩格式的速度和体积对比
# gzip（最快，体积大）
time tar -caf test.tar.gz /path/to/dir

# zstd 默认（速度快，体积小）
time tar -caf test.tar.zst /path/to/dir

# xz（最慢，体积最小）
time tar -caf test.tar.xz /path/to/dir

# bzip2（速度/体积都一般，不推荐）
time tar -caf test.tar.bz2 /path/to/dir
```

**实际测试参考（100MB 源码目录）：**

| 格式 | 压缩时间 | 解压时间 | 体积 |
|------|----------|----------|------|
| gzip -6 | 2.1s | 0.4s | 28MB |
| **zstd -3** | **1.8s** | **0.3s** | **24MB** |
| zstd -10 | 8.5s | 0.3s | 20MB |
| xz -6 | 18.0s | 2.1s | 19MB |

zstd 默认级别就做到了又快又小。

## 6. 进阶技巧

### 6.1 配合管道（避免中间文件）

```bash
# 压缩到远程服务器（不需要本地临时文件）
tar -caf - /path/to/dir | ssh user@host "cat > backup.tar.zst"

# 从远程拉取并解压
ssh user@host "tar -caf - /path/to/dir" | tar -xaf -

# 直接压缩并通过 rsync 风格传输
tar -caf - /path/to/dir | pv | ssh user@host "cat > backup.tar.zst"
```

### 6.2 带进度条

```bash
# 使用 pv 显示进度（需安装 pv）
tar -caf - /path/to/dir | pv -s $(du -sb /path/to/dir | awk '{print $1}') > archive.tar.zst

# 解压时也显示进度
pv archive.tar.zst | tar -xaf -
```

### 6.3 排除文件/目录

```bash
# 排除 node_modules 和 .git
tar -caf project.tar.zst \
  --exclude=node_modules \
  --exclude=.git \
  --exclude='*.log' \
  /path/to/project

# 从排除列表文件读取
tar -caf project.tar.zst -X exclude.txt /path/to/project
```

### 6.4 增量备份（配合 find + 时间戳）

```bash
# 仅备份最近 7 天修改的文件
find /path/to/dir -type f -mtime -7 | tar -caf recent.tar.zst -T -
```

### 6.5 压缩已存在的 .tar 文件

```bash
# 将 .tar 转为 .tar.zst（保留原文件）
zstd archive.tar -o archive.tar.zst

# 删除原文件
zstd --rm archive.tar

# 解压 .tar.zst 回 .tar
zstd -d archive.tar.zst -o archive.tar
# 或
unzstd archive.tar.zst
```

## 7. 常见问题

### Q: tar 不支持 --zstd 怎么办？

旧版 tar 可能没有 `--zstd` 选项，用管道方式：

```bash
# 压缩
tar -cf - /path/to/dir | zstd -o archive.tar.zst

# 解压
zstd -d -c archive.tar.zst | tar -xf -

# 列出内容
zstd -d -c archive.tar.zst | tar -tf -
```

### Q: 如何验证压缩包完整性？

```bash
# 测试归档是否完整（不解压）
tar --zstd -tf archive.tar.zst > /dev/null && echo "OK" || echo "CORRUPT"

# 或用 zstd 直接测试
zstd -t archive.tar.zst
```

### Q: zstd 压缩时会消耗多少内存？

| 压缩级别 | 大致内存 |
|----------|----------|
| 1-10 | ~1-4 MB |
| 11-15 | ~8-64 MB |
| 16-19 | ~128-256 MB |
| 20-22 (--ultra) | ~512MB-2GB |

日常使用（级别 1-10）内存消耗很低，远低于 xz。

### Q: .tar.zst 和 .tzst 有什么区别？

**没有区别，只是后缀不同。** `.tar.zst` 是常见写法，`.tzst` 是短写形式。`tar -a` 两种都能识别。

## 8. 总结：什么时候用 zstd？

| 场景 | 推荐 | 不推荐 |
|------|------|--------|
| 日常备份/归档 | ✅ zstd -3 | |
| CI/CD 构建缓存 | ✅ zstd -1 | |
| 网络传输/发布 | ✅ zstd -5 | |
| 长期冷存储 | ✅ zstd -15 | |
| 需要兼容老旧系统 | | ❌ 用 gzip |
| 需要极致压缩（不在乎时间） | | ❌ 用 xz |
| 分发到不装 zstd 的环境 | | ❌ 用 gzip/zip |

**核心优势：** zstd 是"我可以选最快的，也能选最小的，解压始终超快"的全能选手。绝大多数场景下它都比 gzip 更优，可以放心作为默认压缩格式。
