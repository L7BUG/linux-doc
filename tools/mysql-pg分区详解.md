# MySQL 与 PostgreSQL 分区详解

## 1. 概述

两种数据库的分区理念相同：**逻辑一张表、物理按片存储**，业务 SQL 零感知。实现和工具链差异很大：MySQL 8.0 把分区移入 InnoDB 内部，PostgreSQL 使用声明式分区（每分区是独立表）。

分区的核心价值：

- **业务透明**：MyBatis / JPA 正常写 SQL，分区完全无感——相对手工分表的决定性优势
- **归档一条命令**：无迁移窗口、无漏数据
- **查询裁剪**：优化器自动只扫相关分区

## 2. MySQL 分区

### 2.1 分区类型

| 类型 | 用法 | 适用 |
|------|------|------|
| `RANGE` | 按值范围切分（必须整数表达式） | 时间维度冷热分离（主流） |
| `RANGE COLUMNS` | 按范围切分，**可直接用日期/字符串/多列** | 按时间分区首选，不用包函数 |
| `LIST` / `LIST COLUMNS` | 按枚举值切分 | 按地区/类型分片 |
| `HASH` / `KEY` | 按哈希平均分布 | 均匀打散，无冷热概念 |

### 2.2 按月分区示例

```sql
CREATE TABLE orders (
  id BIGINT NOT NULL,
  order_no VARCHAR(32) NOT NULL,
  user_id BIGINT NOT NULL,
  created_at DATETIME NOT NULL,
  PRIMARY KEY (id, created_at)   -- ⚠️ 分区键必须在主键里！
) PARTITION BY RANGE COLUMNS (created_at) (
  PARTITION p202606 VALUES LESS THAN ('2026-07-01'),
  PARTITION p202607 VALUES LESS THAN ('2026-08-01'),
  PARTITION p202608 VALUES LESS THAN ('2026-09-01'),
  PARTITION pmax VALUES LESS THAN (MAXVALUE)   -- 兜底分区，防止新数据无处安放
);
```

### 2.3 日常管理操作

```sql
-- 新增下月分区（按月滚动建分区，通常用 event 定时任务）
ALTER TABLE orders ADD PARTITION (PARTITION p202609 VALUES LESS THAN ('2026-10-01'));

-- 归档：把 2025-06 分区直接变成独立表搬走（业务无感）
-- 先建同结构的普通表 orders_202506，然后：
ALTER TABLE orders EXCHANGE PARTITION p202506 WITH TABLE orders_202506;

-- 清空一个分区的数据（比 DELETE 快几个数量级）
ALTER TABLE orders TRUNCATE PARTITION p202506;

-- 删除分区（连同数据）
ALTER TABLE orders DROP PARTITION p202506;

-- 拆分/合并分区
ALTER TABLE orders REORGANIZE PARTITION pmax INTO (...);
```

`EXCHANGE PARTITION` 是归档的核心：**一条命令把分区和普通表对调**，业务无感知，没有迁移窗口。

### 2.4 查询优化：分区裁剪

```sql
-- 优化器自动只扫 7、8 月两个分区，其他分区碰都不碰
SELECT * FROM orders WHERE created_at BETWEEN '2026-07-01' AND '2026-08-31';

-- ❌ 裁剪失效：对分区键包了函数，优化器无法判断范围
SELECT * FROM orders WHERE DATE(created_at) = '2026-07-15';  -- 全表扫描所有分区
```

### 2.5 MySQL 分区的硬限制

| 限制 | 影响 |
|------|------|
| **分区键必须在所有唯一键中** | 主键不能是纯 `id`，必须 `(id, created_at)` 复合——最坑的一条，需在表结构设计阶段定好 |
| 不支持外键 | 分区表不能做被引用方（8.0 仍是） |
| 不支持 FULLTEXT / SPATIAL 索引 | 全文搜索别想用分区表 |
| 单表最多 **8192** 个分区 | 按月 20 年才 240 个，没事；按天 20 年 7300 个就爆了 |
| 分区键必须是整数表达式或 COLUMNS 直接列 | `RANGE` 要用 `TO_DAYS(created_at)` 包一层，`RANGE COLUMNS` 不用 |

> MySQL 8.0 把分区实现移进了 InnoDB 内部（5.7 时代是独立的通用分区层），性能和稳定性大幅改善——**用 MySQL 分区必须 8.0+**。

## 3. PostgreSQL 分区

### 3.1 声明式分区（PG 10+，PG 12 成熟，PG 15+ 完善）

PG 的分区是**建表语句级声明**，父表负责路由，子表是真实物理表（每分区独立 rel，可独立管理）：

```sql
-- 父表：定义结构 + 分区策略，不存数据
CREATE TABLE orders (
  id BIGINT,
  order_no VARCHAR(32) NOT NULL,
  user_id BIGINT NOT NULL,
  amount NUMERIC(12,2) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (id, created_at)    -- ⚠️ 同样的约束：唯一键必须含分区键
) PARTITION BY RANGE (created_at);

-- 子分区：真实存储，可独立建索引/约束/统计
CREATE TABLE orders_202607 PARTITION OF orders
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
CREATE TABLE orders_202608 PARTITION OF orders
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

### 3.2 归档操作：DETACH / ATTACH（PG 最优雅的部分）

```sql
-- 归档：拆下来变成独立普通表，业务查询瞬间不可见该分区（无迁移、无锁表）
ALTER TABLE orders DETACH PARTITION orders_202506;

-- PG 13+：不阻塞的分离
ALTER TABLE orders DETACH PARTITION orders_202506 CONCURRENTLY;

-- 再把独立表搬到归档库/转成表导出
-- 迁移完删除或归档存储

-- 反向：把历史表重新挂回分区
ALTER TABLE orders ATTACH PARTITION orders_202506
  FOR VALUES FROM ('2025-06-01') TO ('2025-07-01');
```

### 3.3 自动建分区：pg_partman

MySQL 要自己写 event 滚分区，PG 有官方生态插件：

```sql
CREATE EXTENSION pg_partman;

-- 每月 1 号自动创建下月分区，保留 36 个月
SELECT partman.create_parent(
  p_parent_table => 'public.orders',
  p_control => 'created_at',          -- 分区键
  p_type => 'range',                  -- range/list/hash
  p_interval => '1 month',            -- 粒度
  p_premake => 3                      -- 提前预建 3 个
);
-- 后台定时任务：SELECT partman.run_maintenance();
```

### 3.4 PG 分区特性（比 MySQL 强的地方）

| 能力 | 说明 |
|------|------|
| **外键支持** | 分区表可以建 FK、可以被其他表 FK 引用（MySQL 不行） |
| **索引自动化** | 父表建索引 → 自动给所有分区建（含未来新建的）；DETACH 后索引独立存在 |
| **partition-wise join**（PG 12+） | 两张大分区表 join 时按分区对齐 join，不再全量（默认关闭，`enable_partitionwise_join=on`） |
| **partition-wise aggregate**（PG 11+） | 每个分区先聚合再汇总 |
| **UPDATE 跨分区自动移动** | 更新导致分区键改变时，行自动迁移到目标分区（MySQL 会报错/做行移动） |
| **多级子分区**（PG 12+） | 月分区下再按 hash 二级分区 |
| **HASH 分区**（PG 11+） | 支持均匀打散 |
| **ANALYZE 父表**（PG 14+） | 一次分析汇总所有子表统计 |

### 3.5 注意事项

- 分区数不要失控：每个分区是一个独立 rel（表文件），**建议分区总数控制在几百到一两千内**，超过后 planner 开销和目录膨胀明显
- 至少 **PG 12+**（PG 10-11 的声明式分区性能一般），2026 年普遍 PG 15/16/17
- 裁剪要求同样苛刻：`WHERE created_at >= ? AND created_at < ?` 这种范围条件才能裁；`date_trunc('month', created_at) = ...` 就废了

## 4. 核心对比

| 维度 | MySQL 8.0 | PostgreSQL 12+ |
|------|-----------|----------------|
| 分区语法 | `PARTITION BY RANGE COLUMNS` + 显式分区定义 | `PARTITION BY RANGE` + `PARTITION OF` 子表 |
| 归档操作 | `EXCHANGE PARTITION`（对调表） | `DETACH PARTITION`（拆表，PG 13+ 无锁） |
| 自动建分区 | 自己写 event + 存储过程 | `pg_partman` 一行配置 |
| 唯一键限制 | 必须含分区键 | 必须含分区键（相同） |
| 外键 | ❌ 不支持 | ✅ 双向支持 |
| 分区对齐 join | ❌ | ✅ partition-wise join |
| 物理形态 | 分区是表内概念（InnoDB 内部） | 分区是独立表（每分区一个 rel，可独立管理） |
| 索引 | 分区级索引 | 父表统一声明自动下发 |
| 调优项 | 几乎无 | `enable_partitionwise_join/aggregate`、每分区统计 |

## 5. 实践要点（两种库通用）

1. **粒度选自然月**，别按天——20 年才 240 个分区，按天 7300 个会失控
2. **主键设计要带分区键**（`(id, created_at)`），这是建表前就要定的，后改等于重建表
3. **查询条件永远给分区键范围**：`created_at >= ? AND created_at < ?`，配合 `BETWEEN` 或 JDBC 参数化，才能触发裁剪
4. **业务代码零感知**：MyBatis / JPA 正常写 SQL，分区完全透明
5. 兜底分区（MySQL `MAXVALUE` / PG 未来分区）可建可不建——建了防失控，但裁剪时多一个全扫描目标，量大了建议定期维护不建兜底

## 6. 选型建议

| 你的情况 | 选 |
|---------|-----|
| 已在 MySQL 生态（binlog/Canal/DTS 依赖） | MySQL 8.0 分区，接受"无外键、event 手动滚分区" |
| 能选 PG / 新项目 | PostgreSQL + pg_partman，归档和自动化体验完整高一个档次 |
| 需要解决"分库"而非"时间切片" | 那属于 ShardingSphere / TiDB 的领域，不在分区范畴 |

## 7. 常见问题

**Q1：分区表能替代手工分表（当天表/120 天表）吗？**
能。分区表就是手工分表的现代替代：逻辑单表、业务无感、归档一条命令（EXCHANGE / DETACH），同时消灭跨表 UNION 和路由代码。

**Q2：分区键为什么必须在主键里？**
两种数据库都有此限制：分区路由依赖分区键，如果唯一约束不含分区键，同一唯一值可能分布在不同分区，唯一性校验无法成立。所以订单表主键设计为 `(id, created_at)`。

**Q3：分区表支持事务吗？**
支持。InnoDB 分区表事务与普通表完全一致；PG 分区表同样参与事务（DETACH/ATTACH 除外，那是 DDL）。

**Q4：分区数和查询性能的关系？**
裁剪生效时只扫目标分区，性能与分区数无关；但分区数过多会增加 DDL、统计信息和 planner 的固定开销，所以建议总量控制在几千内，按月建分区完全不用担心。

**Q5：`WHERE DATE(created_at) = ...` 为什么慢？**
对分区键使用函数后，优化器无法推导分区范围，裁剪失效导致全部分区扫描。正确写法是给范围：`created_at >= '2026-07-15 00:00:00' AND created_at < '2026-07-16 00:00:00'`。
