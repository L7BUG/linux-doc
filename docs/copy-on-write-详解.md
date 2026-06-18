# 写时复制（Copy-on-Write）详解

> 本文从内核机制、文件系统实现、编程接口三个维度，系统性地拆解写时复制（CoW）的原理、应用场景、性能权衡，以及日常运维中的实践要点。

## 1. 概念与动机

### 1.1 一句话定义

**写时复制是一种懒惰优化策略：多个调用者共享同一份数据资源，只有当某个调用者试图*修改*这份数据时，系统才真正为其创建一份私有副本。**

直觉类比——"几个人传阅同一份杂志，谁要往上面写字，先给他单独复印一本再写。"

### 1.2 要解决什么问题

假设没有 CoW，系统只有两种办法：

| 策略 | 做法 | 问题 |
|------|------|------|
| **总是共享** | 所有人指向同一份物理数据 | 一个人修改，所有人的数据都被篡改 |
| **立即复制** | 创建资源时就把数据完整复制一份 | 耗时（大内存进程 fork 要复制 GB 级数据）、浪费空间、大量复制后立刻被 `exec()` 丢弃 |

CoW 是第三条路：**先共享，直到有人要写再复制。**

---

## 2. 机制拆解

### 2.1 内核内存 CoW 流程

这是 Linux 内核中最经典、最底层的 CoW 实现，由 MMU（内存管理单元）和缺页异常（page fault）处理程序协同完成。

```
步骤 ①  父进程调用 fork()
┌─────────────────────────────────────┐
│  物理页帧 0x7f3a0000               │
│  数据: "Hello, World"               │
│  _refcount: 2 (父 + 子)            │
│  PTE 标记: READ_ONLY（父 & 子）     │
└─────────────────────────────────────┘
     ↑ PTE              ↑ PTE
   父进程页表           子进程页表

步骤 ②  子进程执行: str [addr], 'X'  ← 试图写入只读页
         ↓
        MMU 检测到写保护 → 触发 #PF（Page Fault）
         ↓
        内核 do_wp_page() 判断 _refcount > 1 → 需要 CoW

步骤 ③  内核执行 CoW
┌──────────────────────┐    ┌──────────────────────┐
│  物理页帧 0x7f3a0000 │    │  物理页帧 0x9b2c1000 │
│  数据: "Hello, World"│    │  数据: "Xello, World"│
│  _refcount: 1 (父)   │    │  _refcount: 1 (子)   │
│  PTE: READ_ONLY      │    │  PTE: READ_WRITE     │
└──────────────────────┘    └──────────────────────┘
       ↑                             ↑
     父进程                         子进程
```

**关键数据结构：**

- **`_refcount`（页引用计数）**：记录有多少个进程页表指向该物理页。`_refcount == 1` 意味着独占，可以安全原地写入；`> 1` 意味着共享，必须复制。
- **PTE（Page Table Entry）**：fork 时父子进程的 PTE 都被设为只读，即便原页是可写的。这确保任何一方的写入尝试都会被 MMU 拦截。

### 2.2 文件系统 CoW 流程（Btrfs 为例）

文件系统级的 CoW 思路对称但机制独立——不依赖 MMU，而是由文件系统的写路径直接实现。

```
初始状态：文件 inode 指向 extent A
┌─────────────────────┐
│    extent A (4K)    │    数据: "Hello, World"
│    refs: 1          │    磁盘位置: LBA 1000
└─────────────────────┘
          ↑
     文件 inode

用户写入: echo "X" | dd of=/file bs=1 seek=0 count=1

步骤 ①  Btrfs 分配新 extent
┌─────────────────────┐    ┌─────────────────────┐
│    extent A (4K)    │    │    extent B (4K)    │
│    refs: 1          │    │    refs: 1          │
│    磁盘: LBA 1000    │    │    磁盘: LBA 2000    │
└─────────────────────┘    └─────────────────────┘

步骤 ②  复制 extent A → extent B，在 extent B 上修改
┌─────────────────────┐    ┌─────────────────────┐
│    extent A (4K)    │    │    extent B (4K)    │
│    数据: 不变        │    │    数据: "Xello..."  │
│    refs: 1          │    │    refs: 1          │
└─────────────────────┘    └─────────────────────┘
                                ↑
                           文件 inode（更新指向）

步骤 ③  更新元数据 → 写入日志树 → 事务提交
         旧 extent A 的引用计数减 1，若为 0 则释放
```

> **对比内存 CoW**：文件系统 CoW 的"复制"发生在 IO 提交层，触发者是文件系统驱动的写操作而非 MMU 异常。复用粒度是 extent（可变长度，Btrfs 通常 128K+），而非固定的 4K 页。

---

## 3. 三大应用场景

### 3.1 fork() — 进程创建

```c
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>

int global = 42;  // 数据段中的全局变量

int main() {
    pid_t pid = fork();

    if (pid == 0) {
        // 子进程
        printf("[子] fork 刚完成时: global = %d (与父共享同一物理页)\n", global);
        global = 100;  // ← 触发 CoW！
        printf("[子] 修改后: global = %d (现在有自己的副本了)\n", global);
    } else {
        wait(NULL);
        printf("[父] global = %d (不受子进程修改影响)\n", global);
    }
    return 0;
}
```

```text
# 输出：
[子] fork 刚完成时: global = 42 (与父共享同一物理页)
[子] 修改后: global = 100 (现在有自己的副本了)
[父] global = 42 (不受子进程修改影响)
```

**关键流程：**

1. `fork()` → 内核复制父进程页表，所有 PTE 标记只读，`_refcount` 递增
2. 子进程写 `global` → MMU 触发写保护错误 → 内核 `do_wp_page()` 分配新页、复制 4K、更新子进程 PTE
3. 父进程的 `global` 永远不受影响——它们的物理页在我（fork 后）、写（写入时）分离了

> **`vfork()` vs `fork()`**：`vfork()` 不实现 CoW，而是让子进程*借用*父进程的地址空间直到子进程 `exec()` 或 `_exit()`，这期间父进程被阻塞。它的存在理由是性能——在 CoW 还不成熟的时代，`vfork()` 避免了复制页表的开销。现代内核中 `fork()` + CoW 已经足够高效，`vfork()` 仅用于对延迟极度敏感的嵌入式场景。

### 3.2 mmap MAP_PRIVATE — 文件私有映射

```c
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

int fd = open("/lib/libc.so.6", O_RDONLY);

// 多个进程都这样映射同一个 .so 文件
void *addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                  MAP_PRIVATE, fd, 0);
//              ↑ 关键标志：私有映射
```

**MAP_PRIVATE 的 CoW 行为：**

| 操作 | 行为 |
|------|------|
| 读取映射区域 | 直接从 page cache 读取，所有进程共享同一份物理页 |
| 写入映射区域 | 触发 CoW → 分配匿名页 → 复制文件页 → 写入 → 该进程的 PTE 指向匿名页 |
| 写回文件 (`msync`) | **不会**写回原文件——私有映射的修改对其他进程和磁盘上的文件都不可见 |

**经典用例——动态链接器：**

ELF 文件的 `.got.plt`（全局偏移表）段在加载时需要进行符号重定位。链接器将 `.text` 页保持 `MAP_PRIVATE | PROT_READ`（共享只读），只对需要修补的 GOT 条目所在页标记 `PROT_WRITE`——写入时 CoW 产生私有副本，代价仅 4K，而非复制整个 `libc.so`。

### 3.3 Btrfs/ZFS — 文件系统级 CoW

文件系统级 CoW 是 CoW 哲学从内存到存储的延伸。它解决了另一个维度的问题：**如何保证写操作的原子性和数据完整性**。

```
传统 in-place 写入（ext4/xfs）：
┌──────────────────────────────────────────┐
│  写入 "Hello" → 找到数据块 → 直接覆写     │
│                                            │
│  ⚠ 如果在覆写过程中断电：                  │
│     → 旧数据已被破坏，新数据未写完          │
│     → 文件内容损坏                          │
└──────────────────────────────────────────┘

CoW 写入（Btrfs/ZFS）：
┌──────────────────────────────────────────┐
│  写入 "Hello" → 分配新块 → 写入新块        │
│     → 原子地更新元数据指针（事务提交）       │
│     → 旧块保持完整直到事务确认为止           │
│                                            │
│  ✅ 任意时刻断电：                          │
│     → 元数据指向要么是旧块（完整）           │
│     → 要么是新块（完整）                    │
│     → 不存在"一半旧一半新"的中间态           │
└──────────────────────────────────────────┘
```

**文件系统 CoW 衍生出的高阶能力：**

| 能力 | 实现方式 | 你文档中的对应章节 |
|------|---------|-------------------|
| **快照（snapshot）** | 创建快照只是增加所有相关 extent 的引用计数，瞬间完成 | `btrfs-raid0-子卷.md` §5.6 |
| **子卷克隆** | 从快照 `btrfs subvolume snapshot` 创建新子卷，初始零额外空间 | `btrfs-raid0-子卷.md` §5.6 |
| **数据校验** | 每个 extent 带 checksum，读取时验证 | 该文档 §4.3 `mkfs.btrfs -m raid0 -d raid0` |
| **透明压缩** | CoW 写入新 extent 的同时压缩 | — |
| **Send/Receive** | 基于 CoW 的增量传输，比较 extent 树的差异 | — |

---

## 4. 性能剖析

### 4.1 成本模型

CoW 不是免费的。理解其开销有助于判断何时启用、何时绕过。

| 开销类型 | 内存 CoW（fork） | 文件系统 CoW（Btrfs） |
|----------|-------------------|------------------------|
| **触发延迟** | 首次写入触发 page fault（~几微秒），后续同页写入零开销 | 每次覆写触发 extent 分配 + 复制（~几十微秒 + IO） |
| **写放大** | 无——粒度恰好等于页大小（4K） | **有**——修改 1 字节也要复制整个 extent（通常 128K+） |
| **碎片化** | 无——物理页分配本身不产生碎片 | **有**——连续文件的多次修改散布到不同物理位置 |
| **元数据开销** | 忽略不计 | 额外的 extent 树、引用计数、checksum 树 |

### 4.2 典型压测对比

```bash
# 场景：随机 4K 写入吞吐量对比
fio --name=randwrite --rw=randwrite --bs=4k --size=1G \
    --numjobs=1 --filename=/mnt/ext4/testfile      # ext4
fio --name=randwrite --rw=randwrite --bs=4k --size=1G \
    --numjobs=1 --filename=/mnt/btrfs/testfile     # Btrfs (CoW 开)
fio --name=randwrite --rw=randwrite --bs=4k --size=1G \
    --numjobs=1 --filename=/mnt/btrfs-nodatacow/testfile  # Btrfs (CoW 关)
```

> **趋势**：ext4 ≈ Btrfs `nodatacow` > Btrfs CoW。差距在机械硬盘上更明显（寻道开销 + CoW 写放大），NVMe 上大幅缩小。

### 4.3 什么时候该禁用 CoW

```bash
# 典型场景：虚拟机磁盘镜像
chattr +C /var/lib/libvirt/images/vm-disk.qcow2

# 典型场景：数据库数据目录
mkdir /mnt/btrfs-data/postgres
chattr +C /mnt/btrfs-data/postgres
# 或在挂载时禁用
mount -o subvol=@pgdata,nodatacow /dev/sda1 /var/lib/postgres
```

| 场景 | 建议 | 原因 |
|------|------|------|
| **虚拟机磁盘镜像** | 禁用 CoW | 虚拟机内部已做 CoW（qcow2）→ 双重 CoW 导致写放大爆炸 |
| **数据库（PostgreSQL/MySQL）** | 禁用 CoW | 数据库的 WAL / 随机写模式与 CoW extent 级复制严重冲突 |
| **swap 文件** | **必须禁用** | 内核需要直接操作磁盘块换页，CoW 的"写时复制"语义与之冲突 |
| **普通文本文档、代码仓库** | 保持 CoW | 受益于快照、校验、压缩，写放大可忽略 |
| **媒体文件（视频/图片库）** | 保持 CoW | 多为顺序大块写入，CoW 写放大比例低 |

---

## 5. CoW 与日志（Journaling）的范式对决

这是理解现代文件系统的关键分水岭。

```
日志（Journaling）- ext4/xfs 的做法：

    [日志区]               [数据区]
   ┌─────────┐           ┌─────────┐
   │ 记录意图 │  ─────→  │ 执行修改 │  → 确认 → 回收日志
   │ "我要把   │  先写日志  │ 原地覆写  │
   │  块X改成Y"│           │          │
   └─────────┘           └─────────┘

    保证：崩溃后回放日志 → 恢复到一致状态


CoW - Btrfs/ZFS 的做法：

   ┌─────────┐           ┌─────────┐         ┌─────────┐
   │ 旧数据块  │           │ 新数据块  │         │ 元数据根  │
   │ (保持完整) │           │ (新分配)  │  ─────→ │ (原子切换) │
   └─────────┘           └─────────┘         └─────────┘

    保证：旧数据永远不被覆盖 → 崩溃后要么全旧、要么全新
```

| 维度 | Journaling（ext4） | CoW（Btrfs） |
|------|-------------------|--------------|
| **数据保护策略** | 先写日志，再写数据（预写式） | 永远不覆盖旧数据，最后原子切指针 |
| **崩溃恢复** | 回放日志（可能慢） | 遍历到最后一个一致的事务根（快） |
| **写放大来源** | 日志：数据写两次（日志 + 最终位置） | extent 复制：大块覆写时复制整个 extent |
| **天然快照** | 不支持 | **天然支持**（快照只是冻结引用） |
| **天然校验** | 不支持 | **天然支持**（新写 extent 时顺带计算 checksum） |
| **碎片化风险** | 低 | 中 ~ 高（取决于写入模式） |

---

## 6. 编程视角：手动利用 CoW

### 6.1 Rust `std::borrow::Cow`

标准库直接暴露了 CoW 作为抽象类型：

```rust
use std::borrow::Cow;

fn process(data: &str) -> Cow<str> {
    if data.contains("magic") {
        // 需要修改 → 触发复制
        Cow::Owned(data.replace("magic", "processed"))
    } else {
        // 不需要修改 → 返回借用，零复制
        Cow::Borrowed(data)
    }
}

let input = "hello world";
let output = process(input);  // 无 magic → 零复制
```

### 6.2 Linux 内核 `do_wp_page()` 简化逻辑

内核中真正实现 fork CoW 的核心函数（极度简化版）：

```c
// mm/memory.c — 简化伪代码，仅展示核心逻辑
static int do_wp_page(struct vm_fault *vmf) {
    struct page *old_page = vmf->page;

    // 情况 1：引用计数为 1 → 无人共享，直接写
    if (page_count(old_page) == 1) {
        pte_mkwrite(vmf->pte);           // 恢复写权限
        return 0;                        // 零复制开销
    }

    // 情况 2：引用计数 > 1 → 必须 CoW
    struct page *new_page = alloc_page(GFP_HIGHUSER);  // 分配新物理页
    copy_user_highpage(new_page, old_page);            // 复制 4K 内容
    set_pte_at(vmf->vma->vm_mm, vmf->address,
               vmf->pte, mk_pte(new_page, vmf->vma->vm_page_prot));
    page_remove_rmap(old_page);                        // 旧页引用减 1
    return 0;
}
```

关键洞察就 7 个字：**引用计数决定行为**——`_refcount == 1` 则原地写，`> 1` 则复制。

---

## 7. 常见问题

### fork 后 exec 马上替换整个地址空间，CoW 岂不是白设了？

**不会。** 这正是 CoW 设计最优雅的地方。fork + exec 的模式下：

1. fork 时：设置 CoW 的代价仅是**复制页表**（几 KB），而非复制整个地址空间（可能 GB）
2. exec 时：内核直接销毁子进程的页表，释放所有页引用
3. 那些从未被子进程写入的物理页——它们只是 `_refcount` 先加 1（fork 时）又减 1（exec 时），**从未被物理复制过**

没有 CoW 的 fork + exec 需要完整复制一遍进程内存再立刻释放——纯浪费。

### 如果父进程也在写同一页，会发生什么？

父进程和子进程各自触发 CoW。假设某 4K 页 `refcount = 2`（父子共享）：

- 子进程先写 → `refcount > 1` → 复制新页 → 子进程 PTE 指向新页 → 旧页 `refcount` 减为 1
- 父进程后写 → `refcount == 1`（独占了） → 内核直接恢复该页的写权限，**不再复制**

每个页最多被复制一次——先写的人付出复制代价，后写的人免费。

### CoW 导致的"内存超额"风险是什么？

当多个子进程 fork 自同一个大内存父进程时，如果它们都大规模写入不同页，系统可能 OOM：

```bash
# 父进程占用 1 GB
# 依次 fork 10 个子进程
# 每个子进程写入全部 1 GB 的不同页
# 物理需求：1 GB（父） + 10 × 1 GB（子） = 11 GB
# 但 CoW 起初只报了 ~1 GB 的物理占用
```

Linux 用 `vm.overcommit_memory` 控制这种行为：
- `0`（默认）：启发式——允许 CoW 乐观共享，但拒绝明显的疯狂分配
- `1`：始终允许超额分配（最宽松）
- `2`：严格拒绝超额——`CommitLimit = swap + RAM × overcommit_ratio%`

### swap 文件为什么必须禁用 CoW？

这是你在 `btrfs-raid0-子卷.md` 中已经实践过的操作。原因：

1. swap 子系统的 I/O 路径绕过文件系统 VFS 层，直接操作 `bdev`（块设备）上的扇区
2. 当内核需要换出页面时，它必须能直接定位到 swap 文件所在的磁盘块地址
3. CoW 的"写入时新分配 extent → 元数据指针变化"使 swap 文件的块映射不稳定
4. `chattr +C` 确保 swap 文件使用 in-place 写入，块映射固定不变

```bash
btrfs subvolume create /mnt/@swap
chattr +C /mnt/@swap/swapfile   # 必须在写入任何数据前设置
```

### Btrfs 的快照真的不占空间吗？

快照创建时确实不占额外数据空间——它只是给所有相关的 extent 增加引用计数。但有两个隐性成本：

1. **元数据开销**：快照本身需要独立的文件树根节点和少量元数据块（通常几 MB）
2. **空间"债务"**：随着原始子卷和快照各自写入，CoW 分离的 extent 会逐渐消耗空间。删除快照前，被快照引用的旧 extent 无法被回收——这就是"删了文件但磁盘空间不释放"的原因

```bash
# 检查共享 vs 独占空间
btrfs filesystem du /mnt/btrfs-root
btrfs qgroup show /mnt/btrfs-root
```

---

## 参考

- [Linux Kernel — `mm/memory.c` `do_wp_page()`](https://elixir.bootlin.com/linux/latest/source/mm/memory.c)
- [Btrfs Wiki — Copy-on-Write](https://btrfs.readthedocs.io/en/latest/Glossary.html#copy-on-write)
- [The Design and Implementation of the FreeBSD Operating System — Chapter on Virtual Memory](https://www.freebsd.org)
- 本仓库：`btrfs-raid0-子卷.md` — Btrfs CoW 的实战应用（快照、swap、`chattr +C`）
