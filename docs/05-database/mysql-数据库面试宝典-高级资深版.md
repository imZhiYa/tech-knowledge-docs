# 🐬 MySQL 数据库面试宝典（高级 / 资深版）

> **重点**：MySQL InnoDB 内核（索引 / 事务 / 锁 / 日志 / 崩溃恢复），并覆盖优化器、复制与高可用、分库分表、缓存一致性、线上排障、设计题、SQL 手写题。
>
> **版本基准**：MySQL 8.0 / 8.4 LTS，版本差异处会标注。MySQL 8.0 已于 2026 年 4 月结束支持（EOL），新项目一般选 8.4 LTS。
>
> **难度标记**：★ 基础 ｜ ★★ 进阶 ｜ ★★★ 资深（常作为追问）
>
> **资深答题框架**：**原理 → 取舍 → 线上经验 → 边界与例外**。只背结论是高级，能讲清"为什么这样设计、代价是什么、踩过什么坑"才是资深。

---

## 目录

- [使用说明](#study-guide)

0. [知识地图](#0-%E7%9F%A5%E8%AF%86%E5%9C%B0%E5%9B%BE)
1. [InnoDB 架构](#1-innodb-%E6%9E%B6%E6%9E%84)
2. [索引](#2-%E7%B4%A2%E5%BC%95)
3. [事务与 MVCC](#3-%E4%BA%8B%E5%8A%A1%E4%B8%8E-mvcc)
4. [锁](#4-%E9%94%81)
5. [日志与崩溃恢复](#5-%E6%97%A5%E5%BF%97%E4%B8%8E%E5%B4%A9%E6%BA%83%E6%81%A2%E5%A4%8D)
6. [SQL 执行、优化器与调优](#6-sql-%E6%89%A7%E8%A1%8C%E4%BC%98%E5%8C%96%E5%99%A8%E4%B8%8E%E8%B0%83%E4%BC%98)
7. [复制与高可用](#7-%E5%A4%8D%E5%88%B6%E4%B8%8E%E9%AB%98%E5%8F%AF%E7%94%A8)
8. [分库分表与分布式](#8-%E5%88%86%E5%BA%93%E5%88%86%E8%A1%A8%E4%B8%8E%E5%88%86%E5%B8%83%E5%BC%8F)
9. [缓存与一致性](#9-%E7%BC%93%E5%AD%98%E4%B8%8E%E4%B8%80%E8%87%B4%E6%80%A7)
10. [存储引擎与数据库选型对比](#10-%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E%E4%B8%8E%E6%95%B0%E6%8D%AE%E5%BA%93%E9%80%89%E5%9E%8B%E5%AF%B9%E6%AF%94)
11. [线上故障排查](#11-%E7%BA%BF%E4%B8%8A%E6%95%85%E9%9A%9C%E6%8E%92%E6%9F%A5)
12. [系统设计 / 场景题](#12-%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1--%E5%9C%BA%E6%99%AF%E9%A2%98)
13. [SQL 手写题](#13-sql-%E6%89%8B%E5%86%99%E9%A2%98)
14. [常见错误认知纠正](#14-%E5%B8%B8%E8%A7%81%E9%94%99%E8%AF%AF%E8%AE%A4%E7%9F%A5%E7%BA%A0%E6%AD%A3)
15. [版本差异速查](#15-%E7%89%88%E6%9C%AC%E5%B7%AE%E5%BC%82%E9%80%9F%E6%9F%A5)
16. [追问清单与答题策略](#16-%E8%BF%BD%E9%97%AE%E6%B8%85%E5%8D%95%E4%B8%8E%E7%AD%94%E9%A2%98%E7%AD%96%E7%95%A5)

---

---
<a id="study-guide"></a>
## 使用说明

这份宝典作为数据库高级 / 资深工程师面试复习的**总纲与检查清单**。正文沿用原稿的章节安排、难度标记和复习优先级；本次整理主要统一标题、增加仓库导航和学习路径，不将未经单独核验的版本细节视为新的权威结论。

### 推荐复习路径

1. **先按 P0 复习**：索引、事务与 MVCC、锁、Redo / Undo / Binlog、复制与延迟、慢 SQL 排查。每个主题至少能从原理解释到实际故障。
2. **再用 P1 补齐工程能力**：加锁规则、死锁诊断、崩溃恢复、MDL、并行复制、分库分表、分布式事务与缓存一致性。
3. **最后用 P2 拉开深度**：优化器成本模型、LSM 与 B+ 树、MySQL 与 PostgreSQL、NewSQL、高可用和容量规划。
4. **按资深答题框架复述**：先给结论，再讲机制；接着说明方案收益与代价，补充线上经验、边界条件和版本差异。
5. **通过 SQL / 场景题验证**：不能只复述定义，要能画出事务时序、解释执行计划，并说清楚监控指标和故障止血动作。

### 与仓库深度专题的对应关系

| 面试宝典主题 | 推荐继续阅读 | 深入目标 |
| --- | --- | --- |
| InnoDB 架构、索引、MVCC、Redo / Undo / Binlog | [mysql-01-InnoDB深度解析.md](./mysql-01-InnoDB深度解析.md) | 从单条 UPDATE 的执行链路理解内部机制 |
| 锁、间隙锁、死锁与故障治理 | [mysql-02-锁冲突与死锁治理.md](./mysql-02-锁冲突与死锁治理.md) | 从锁等待与等待图推导线上治理方案 |
| 复制、GTID、主从延迟与高可用 | [mysql-03-主从复制与高可用.md](./mysql-03-主从复制与高可用.md) | 理解事务如何跨实例复制及故障切换 |
| 备份、恢复与 PITR | [mysql-04-备份恢复与PITR.md](./mysql-04-备份恢复与PITR.md) | 将备份链、日志位点和恢复目标串联 |
| 事务边界、隔离级别、快照读 / 当前读实验 | [mysql-05-事务与隔离级别实战.md](./mysql-05-事务与隔离级别实战.md) | 通过双会话 SQL 验证 RC / RR 与并发异常 |

> **证据与版本边界**：本文保留原稿中的核心内容与表达框架。涉及参数默认值、具体小版本行为、生命周期及实现细节时，应以目标部署版本的官方文档和可复现实验为准；本次整理不代表对所有知识点进行了逐条审计。

---

## 0. 知识地图

```
                    ┌────────────── Server 层 ──────────────┐
 客户端 ─ 连接器 ─ 解析器 ─ 优化器 ─ 执行器 ─ binlog（逻辑日志，归档/复制）
                                         │
                    ┌────────────── InnoDB 层 ──────────────┐
                    │ 内存：Buffer Pool / Change Buffer /      │
                    │       Adaptive Hash Index / Log Buffer  │
                    │ 磁盘：表空间(ibd) / Redo / Undo /         │
                    │       Doublewrite / 系统表空间            │
                    │ 机制：B+树索引 / MVCC / 行锁 / 崩溃恢复     │
                    └─────────────────────────────────────────┘
```

**四条主线**（几乎所有题目都能挂到其中一条上）：

| 主线 | 核心机制 | 解决的问题 |
| --- | --- | --- |
| 索引 | B+ 树、聚簇/二级索引 | 查得快 |
| 事务 | undo + MVCC + 锁 | 隔离性、原子性 |
| 日志 | redo + binlog + doublewrite | 持久性、崩溃安全、复制 |
| 分布式 | 复制、分片、一致性 | 扩展性、可用性 |

---

## 1. InnoDB 架构

### Q1.1 InnoDB 的整体架构？一条 UPDATE 的完整执行流程？★★

**内存结构**

- **Buffer Pool**：缓存数据页和索引页（默认页 16KB），是 InnoDB 最重要的内存区域。
- **Change Buffer**：缓冲对**非唯一二级索引**页的变更（页不在 Buffer Pool 时）。
- **Adaptive Hash Index（AHI）**：对热点等值查询自动建哈希索引。
- **Log Buffer**：redo log 的内存缓冲（`innodb_log_buffer_size`）。

**磁盘结构**

- 表空间：系统表空间（ibdata1）、独立表空间（`innodb_file_per_table`，默认开启）、通用表空间、Undo 表空间（8.0 起独立，默认 2 个）、临时表空间。
- Redo log（8.0.30+ 在 `#innodb_redo` 目录，由 `innodb_redo_log_capacity` 控制总容量，可动态调整；之前是 `ib_logfile0/1`）。
- Doublewrite buffer（8.0.20 起为独立文件 `#ib_*.dblwr`）。
- 8.0 起数据字典也存放在 InnoDB 表中（`mysql.ibd`），支持原子 DDL。

**后台线程**：Master Thread、IO Threads（read/write/log）、Purge Thread（清理 undo）、Page Cleaner Thread（刷脏页）。

**UPDATE 执行流程**（`update t set c=c+1 where id=1`）

1. 执行器让引擎取 `id=1` 的行：Buffer Pool 命中则直接返回，否则从磁盘读页进入 Buffer Pool。
2. 对该行加 X 行锁；写 **undo log**（记录旧值，用于回滚和 MVCC）。
3. 修改 Buffer Pool 中的数据页（变成脏页），把修改写入 **redo log buffer**（以 mtr 为单位保证原子）。
4. 更新二级索引：页在 Buffer Pool 则直接改；不在且为非唯一索引则写入 Change Buffer。
5. 提交：**两阶段提交**（redo prepare → 写 binlog → redo commit），按刷盘策略 fsync。
6. 释放锁；脏页后续异步刷盘（经 doublewrite）；undo 在不再被任何 ReadView 需要后由 purge 回收。

> 追问：为什么不在提交时直接把数据页刷盘？——数据页是随机写且 16KB，redo 是顺序写且体量小，这就是 **WAL（Write-Ahead Logging）**：把随机写变顺序写。

### Q1.2 页、区、段、行是什么？一页能存多大的行？★★

- **页（Page）**：InnoDB 磁盘管理的最小单位，默认 16KB（`innodb_page_size` 可选 4/8/16/32/64KB，初始化后不可改）。
- **区（Extent）**：连续 64 个页 = 1MB，保证空间局部性，便于顺序 IO。
- **段（Segment）**：B+ 树的叶子节点段、非叶子节点段（以及回滚段等）。
- **页结构**：File Header → Page Header → Infimum/Supremum 虚拟记录 → User Records → Free Space → **Page Directory（槽，二分查找）** → File Trailer（校验，配合 doublewrite 检测页损坏）。
- 页内记录通过**单向链表**按主键有序串联；页目录每个槽管理 4\~8 条记录，页内先二分定位槽再顺序扫描。
- **行格式**：Redundant、Compact、**Dynamic（5.7+ 默认）**、Compressed。Dynamic 下大字段（TEXT/BLOB/长 VARCHAR）整体存溢出页，行内只留 20 字节指针。
- **行大小限制**：一页至少存 2 行，因此行内数据约不超过 8KB（大字段溢出不计入）。

### Q1.3 Buffer Pool 是怎么管理的？LRU 做了什么改进？★★

- 三个链表：**Free List**（空闲页）、**LRU List**（缓存页）、**Flush List**（脏页，按最早修改 LSN 排序）。
- **改进的 LRU（midpoint insertion）**：LRU 分为 young 区和 old 区（默认 old 占 `innodb_old_blocks_pct=37%`）。新读入的页先插入 old 区头部；只有在 old 区停留超过 `innodb_old_blocks_time`（默认 1000ms）后再次被访问，才晋升到 young 区。
- **解决的问题**：全表扫描 / 预读会一次性读入大量"只用一次"的页，普通 LRU 会把真正的热数据挤出去（缓存污染）。
- **预读**：线性预读（`innodb_read_ahead_threshold`，默认 56，顺序访问一个区内超过阈值个页时预读下一个区）；随机预读（默认关闭）。
- **Buffer Pool 预热**：`innodb_buffer_pool_dump_at_shutdown` / `load_at_startup`（默认开启），重启后恢复热点页，避免冷启动抖动。
- **调优**：专用机器 Buffer Pool 一般设为物理内存的 50%\~75%；`innodb_buffer_pool_instances` 拆分实例降低 mutex 争用；5.7+ 支持在线调整大小（以 chunk 为单位）。
- **命中率**：`Innodb_buffer_pool_read_requests` 与 `Innodb_buffer_pool_reads` 之比，OLTP 通常要求 99% 以上。

### Q1.4 Change Buffer 是什么？为什么唯一索引用不了？★★★

- **作用**：对**非唯一二级索引**的 INSERT / UPDATE / DELETE，如果目标页不在 Buffer Pool 中，不立即把页读入，而是把变更记录到 Change Buffer，之后合并（merge）。省掉了大量随机读 IO。
- **merge 时机**：该页被读取时、后台 Master Thread 周期性合并、数据库正常关闭时、Change Buffer 空间不足时。
- **为什么唯一索引不行**：写入前必须判断唯一性，必须把页读进内存才能判断，此时已经付出了读 IO，缓冲没有意义。
- **适用场景**：写多读少、写入后马上被读的概率低（账单、日志类）。如果写入后马上查询，会频繁触发 merge，反而增加维护代价。
- **参数**：`innodb_change_buffer_max_size`（默认 25%，最大 50%）、`innodb_change_buffering`。
- **持久化**：Change Buffer 本身也有对应的 redo 保证崩溃安全，其内容最终落在系统表空间中。
- 8.4 起 `innodb_change_buffering` 默认值调整为 none（见[版本差异](#15-%E7%89%88%E6%9C%AC%E5%B7%AE%E5%BC%82%E9%80%9F%E6%9F%A5)），生产需实测收益再决定是否开启。

### Q1.5 Doublewrite Buffer 解决什么问题？★★★

- **问题：部分页写（partial page write / torn page）**。InnoDB 页 16KB，而文件系统/磁盘的原子写单位通常是 4KB（或 512B），断电时 16KB 的页可能只写了一部分，页已损坏。
- **为什么 redo 救不了**：redo 是**物理逻辑日志**（"对某页某偏移做某操作"），需要页本身是完好的、LSN 对得上才能重放。页损坏则无法基于它重放。
- **机制**：刷脏页时，先把脏页顺序写入 doublewrite 区域并 fsync，再写入各自的数据文件位置。崩溃恢复时如果发现数据页校验失败，用 doublewrite 里的完整副本覆盖，再应用 redo。
- **代价**：多一次顺序写，性能损耗通常在个位数百分比。
- **何时可以关闭**：底层存储保证 16KB 原子写（如部分云盘/ZFS/Fusion-io 原子写、某些 RDS 实现）；`innodb_doublewrite` 可调整。
- 不要和 redo / binlog 混淆：doublewrite 保护的是**数据页本身的完整性**。

### Q1.6 Adaptive Hash Index 是什么？什么时候该关？★★★

- InnoDB 监控索引访问模式，对频繁以**等值条件**访问的索引页，在内存中自动构建哈希索引，把 B+ 树多层查找缩短为一次哈希查找。只支持等值，不支持范围/排序。
- 全局受 `btr_search` latch 保护，高并发下可能成为瓶颈；8.0 通过 `innodb_adaptive_hash_index_parts`（默认 8）分区缓解。
- 观察 `SHOW ENGINE INNODB STATUS` 的 `SEMAPHORES` 和 `INSERT BUFFER AND ADAPTIVE HASH INDEX` 段，看到 btr_search 相关等待很多时应考虑关闭。
- 8.4 起默认关闭。

---

## 2. 索引

### Q2.1 为什么 MySQL 用 B+ 树做索引？和 B 树、红黑树、哈希、跳表、LSM 对比？★

| 结构 | 问题 |
| --- | --- |
| 二叉树 / 红黑树 | 树高太大，每层一次磁盘 IO，磁盘场景下 IO 次数过多 |
| 哈希 | 只支持等值，不支持范围、排序、前缀；有哈希冲突 |
| B 树 | 非叶子节点也存数据，单页容纳键更少 → 树更高；范围查询需要中序遍历，不友好 |
| **B+ 树** | 非叶子只存键+指针（扇出大，树矮，一般 3\~4 层）；叶子存数据并**双向链表**相连，范围扫描、排序、分页天然高效；查询路径长度稳定 |
| 跳表 | 内存结构，层数随机，磁盘 IO 不可控（Redis ZSET、LevelDB MemTable 使用） |
| LSM-Tree | 写友好（顺序写、合并），读放大、写放大、空间放大（RocksDB、HBase、MyRocks） |

**三层 B+ 树能存多少行（经典估算）**：主键 BIGINT 8B + 页指针 6B ≈ 14B → 非叶子页可放 16KB/14B ≈ 1170 个指针；叶子页按每行 1KB 放 16 行；3 层 ≈ 1170 × 1170 × 16 ≈ **2000 万行**。这就是"单表 2000 万"经验值的由来，但它只是特定行宽下的估算，不是硬限制。

### Q2.2 聚簇索引与二级索引？回表是什么？★

- **聚簇索引**：叶子节点存**完整行数据**。选择规则：显式主键 → 第一个 NOT NULL 的唯一索引 → 隐藏的 6 字节 `DB_ROW_ID`。表数据本身就是按主键组织的 B+ 树（索引组织表）。
- **二级索引**：叶子节点存**索引列 + 主键值**。通过二级索引查询需要拿主键再到聚簇索引查整行，即**回表**。
- **推论**：
  - 主键越短，所有二级索引越小，这是不建议用 UUID / 长字符串做主键的原因之一。
  - 二级索引查询需要两次 B+ 树查找；覆盖索引可以避免回表。
  - 回表是**随机 IO**，当命中行数占比较大（经验上 20%\~30% 以上）时，优化器宁可选全表扫描。

### Q2.3 主键怎么设计？自增 vs UUID vs 雪花？★★

- **自增主键**：顺序插入，页写满再分配新页，**无页分裂**，页利用率高（约 15/16），空间紧凑。缺点：
  - 分库分表下无法保证全局唯一；
  - 暴露业务量（可被遍历、推测订单量）；
  - 高并发插入存在自增锁竞争（`innodb_autoinc_lock_mode`：0 表锁、1 连续模式、2 交错模式，**8.0 默认 2**，要求 `binlog_format=ROW`）；
  - 自增值回溯：**5.7 及之前重启后会按 max(id)+1 重新计算**，8.0 起自增计数器持久化（写入 redo）；
  - 溢出：INT UNSIGNED 约 42.9 亿用尽后继续插入会报主键冲突，核心业务直接用 BIGINT。
- **UUID（随机）**：插入位置随机 → 频繁**页分裂**、页碎片、Buffer Pool 命中差；主键 36 字符导致所有二级索引膨胀。
- **雪花 / 趋势递增 ID**（时间戳 + 机器位 + 序列）：全局唯一、大致递增，兼顾分布式与索引友好。注意时钟回拨、机器号分配、JS 精度（超 2^53 需转字符串）。
- **号段模式**（Leaf-segment）：DB 每次取一段，应用内存分配，双 buffer 预取，吞吐高，趋势递增。
- 业务上**不要用业务字段（手机号、身份证）做主键**。

### Q2.4 页分裂与页合并？如何避免？★★★

- 页满后插入新记录需要分裂：新建页、迁移约一半记录、更新父节点指针。随机主键导致分裂频繁，产生碎片，页利用率常降到 50%\~70%。
- 删除使页利用率低于阈值（`MERGE_THRESHOLD`，默认 50%）时尝试与相邻页合并。
- 避免：主键趋势递增；批量导入时按主键排序后写入；**避免大量随机 UPDATE 变长字段**导致行变长迁移；定期重建碎片严重的表（`ALTER TABLE t ENGINE=InnoDB` 或 `OPTIMIZE TABLE`，务必走 Online DDL，在低峰执行或用 gh-ost / pt-osc）。

### Q2.5 联合索引、最左前缀、覆盖索引、索引下推？★★

- **联合索引 (a, b, c)**：按 a、b、c 依次排序。可用于 `a`、`a,b`、`a,b,c` 以及 `a` 等值 + `b` 范围；**范围查询之后的列无法继续用索引定位**（`a=1 and b>2 and c=3`，c 只能靠 ICP 过滤）。
- **最左前缀**：优化器会重排 WHERE 里条件的顺序，所以 `where b=1 and a=1` 同样能用 (a, b)；但只有 `b` 则不能用（8.0.13 起的 **Skip Scan** 在 a 基数很小时可例外）。
- **覆盖索引**：查询所需列全部在索引中，无需回表（Extra：`Using index`）。典型优化手段，如把常查询的列加入联合索引尾部。
- **索引下推 ICP**（5.6）：以 `(name, age)` 索引、`where name like '张%' and age=10` 为例。
  - 无 ICP：存储引擎按 name 前缀逐条取出主键并回表，再由 Server 层判断 age；
  - 有 ICP：存储引擎在二级索引内直接判断 age，**不满足的不回表**（Extra：`Using index condition`）。
  - 只对二级索引有效（聚簇索引已经读整行，无意义）。
- **MRR**（Multi-Range Read）：二级索引扫描出一批主键后先排序再回表，把随机 IO 变顺序（Extra：`Using MRR`）。
- **选择联合索引字段顺序的原则**：等值条件列在前，范围列在后；区分度高的在前（但要结合最左前缀与查询模式，不是绝对）；能让一个索引服务多个查询、能覆盖排序（`order by`）更好。

### Q2.6 索引失效的常见场景？★

1. 对索引列做**函数/运算/表达式**：`where date(create_time)='...'`、`where id+1=10`（8.0.13 起可建函数索引）。
2. **隐式类型转换**：列为 VARCHAR，用数字比较 `where phone=13800000000`，会把**列**转为数字，导致无法用索引；反过来数字列与字符串比较则不会失效（字符串常量被转成数字）。
3. **字符集/排序规则不一致的 join**：如 utf8 与 utf8mb4 关联，转换发生在被驱动表的列上，导致被驱动表索引失效。
4. 前导通配 `like '%abc'`；`like 'abc%'` 可以用索引。
5. 违反最左前缀。
6. `or` 两侧不是都有索引（有时会走 index merge，仍需评估）。
7. `!=`、`not in`、`is not null`：是否走索引取决于选择性与统计信息，**不是绝对失效**。
8. 优化器**评估全表扫描更便宜**（回表占比过高、小表）。
9. 统计信息不准导致选错索引（见 Q2.8）。

> 资深加分：解释"为什么"。比如隐式转换失效的本质是 `CAST(col AS ...)` 相当于对列做函数运算，B+ 树里存的是原值，无法使用有序性。

### Q2.7 索引不是越多越好？★

- 每个二级索引都是一棵独立 B+ 树，INSERT/DELETE 要维护所有索引，UPDATE 索引列要删旧增新，占用磁盘和 Buffer Pool，增加优化器选择成本和锁冲突。
- 建议：单表索引控制在 5 个左右，联合索引代替多个单列索引，定期检查无用索引（`sys.schema_unused_indexes`、`sys.schema_redundant_indexes`），8.0 可先设为**不可见索引**（`ALTER TABLE ... ALTER INDEX x INVISIBLE`）观察一段时间再删。
- 前缀索引：`index(col(10))` 省空间，但无法用于覆盖索引与 `order by`；用 `count(distinct left(col, n))/count(*)` 评估区分度。

### Q2.8 优化器为什么会选错索引？如何处理？★★★

- 优化器基于 **成本模型**：扫描行数估算 + 回表代价 + 是否需要临时表/排序。行数估算来自**统计信息**：
  - `innodb_stats_persistent`（默认 ON，持久化统计），采样 `innodb_stats_persistent_sample_pages`（默认 20 页），所以 cardinality 是**估算值**。
  - 表数据变化超过 10% 时自动重算（`innodb_stats_auto_recalc`）。
- 选错的典型原因：统计信息过期/采样不准；`order by ... limit` 时优化器为避免排序而选择了主键扫描；回表代价估算偏差；大量删除后索引页中残留未清理的 delete-mark 记录。
- 处理：
  1. `ANALYZE TABLE t` 重新采样（低峰执行）；
  2. 加大采样页数（表级 `STATS_SAMPLE_PAGES`）；
  3. `FORCE INDEX` / 8.0 的优化器 Hint（`/*+ INDEX(t idx) */`）；
  4. 改写 SQL（给 order by 加辅助列、`order by a, id`）；
  5. 建更合适的联合索引；
  6. 8.0 可对列建**直方图**（`ANALYZE TABLE t UPDATE HISTOGRAM ON col`）改善非索引列的选择性估计。
- 排查工具：`EXPLAIN FORMAT=JSON`、`optimizer_trace`、`EXPLAIN ANALYZE`（8.0.18+）。

### Q2.9 ORDER BY 是怎么执行的？filesort 是什么？★★★

- **filesort** 不一定在磁盘上，指 Server 层的排序过程（Extra：`Using filesort`）。排序缓冲区大小为 `sort_buffer_size`。
- **全字段排序**：把查询需要的所有字段放进 sort_buffer，按排序键排完直接返回，不需要再回表。
- **rowid 排序**：当单行太大（超过 `max_length_for_sort_data`）时只放排序键+主键，排完再回表取其他字段，多一次回表。
- 数据量超过 sort_buffer → **外部排序**（分成多个临时文件，各自排序后归并，观察 `Sort_merge_passes`）。
- `order by ... limit N` 且 N 较小时，8.0 使用**优先队列（堆）**，只维护 N 个元素，不需要完整排序。
- **优化**：建立 `(where 列, order by 列)` 的联合索引，利用索引有序性直接避免 filesort；覆盖索引进一步避免回表。
- `group by` 默认也可能使用临时表，8.0 起不再隐式排序。

### Q2.10 count(*) 为什么慢？count(*)、count(1)、count(列) 的区别？★★

- InnoDB 支持 MVCC，同一时刻不同事务看到的行数可能不同，所以**不能像 MyISAM 一样直接存储总行数**，必须扫描。
- `count(*)` 和 `count(1)`：不取值，只统计行数，**语义与效率基本一致，优化器会选择最小的二级索引扫描**（8.0.13+ 还可以并行扫描）；`count(主键)`：需要取主键并判空；`count(列)`：需要判断是否为 NULL，不统计 NULL。效率大致：`count(*) ≈ count(1) > count(主键) > count(非索引列)`。
- 大表精确计数的做法：
  - 单独的计数表/Redis 计数（需要与业务写入保持一致，可用事务内更新计数行）；
  - 近似值可以用 `SHOW TABLE STATUS` 或 `information_schema.tables.table_rows`（误差可达 40%\~50%）；
  - 离线统计汇总。

### Q2.11 Online DDL 与 MDL 锁阻塞？★★★

- **ALGORITHM**：
  - `INSTANT`：只改元数据（8.0.12 起支持加列，8.0.29 起支持任意位置加/删列），瞬间完成；
  - `INPLACE`：在原表上操作，不拷贝数据（加索引、改列类型的部分场景），期间允许并发 DML，操作的最后阶段需要短暂的排他 MDL；
  - `COPY`：建临时表拷贝数据，期间阻塞写。
- **MDL（元数据锁）问题**：事务中的任何 SELECT 都会对表加 MDL 读锁，持有到事务结束。DDL 需要 MDL 写锁，如果有**长事务/未提交事务**持有读锁，DDL 会等待；而 DDL 在等待队列中时，后续所有对该表的查询（需要 MDL 读锁）也会被阻塞 → **整张表不可读写，连接数暴涨**。
- **防范**：
  1. 先查长事务 `information_schema.innodb_trx`，`sys.schema_table_lock_waits`；
  2. 设置 `lock_wait_timeout`（或 DDL 语句里 `WAIT n` / `NOWAIT`），拿不到锁就放弃并重试；
  3. 业务低峰执行；
  4. 大表使用 **gh-ost / pt-online-schema-change**：创建影子表 → 全量拷贝 + 增量同步（gh-ost 读 binlog，pt-osc 用触发器）→ 原子 rename 切换。gh-ost 无触发器、可暂停、可限速，对主库影响更小。

---

## 3. 事务与 MVCC

### Q3.1 ACID 分别靠什么保证？★

| 特性 | 实现 |
| --- | --- |
| **原子性 A** | undo log（失败时回滚到事务前的状态） |
| **持久性 D** | redo log（WAL，提交时刷盘；崩溃后重放）+ doublewrite |
| **隔离性 I** | 锁 + MVCC |
| **一致性 C** | 以上三者共同保证，再加上业务约束（主键、唯一、外键、CHECK 等）。一致性是目的，A/I/D 是手段 |

### Q3.2 四种隔离级别与并发问题？★

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | InnoDB 实现 |
| --- | --- | --- | --- | --- |
| READ UNCOMMITTED | 可能 | 可能 | 可能 | 直接读最新版本，不看 ReadView |
| READ COMMITTED（RC） | 否 | 可能 | 可能 | **每次** SELECT 生成新 ReadView；加锁读只加记录锁 |
| REPEATABLE READ（RR，默认） | 否 | 否 | 快照读不会；当前读可能 | **首次**一致性读时生成 ReadView，整个事务复用；加锁读使用 next-key lock |
| SERIALIZABLE | 否 | 否 | 否 | 普通 SELECT 隐式变为 `LOCK IN SHARE MODE` |

- **脏读**：读到其他事务未提交的数据。
- **不可重复读**：同一事务内两次读同一行，数据**被修改**。
- **幻读**：同一事务内两次范围查询，**行数变多/变少**（新增或删除了符合条件的行）。

### Q3.3 MVCC 的原理？★★

**三个组成部分**

1. **隐藏列**：`DB_TRX_ID`（6B，最后修改该行的事务 ID）、`DB_ROLL_PTR`（7B，指向 undo log 中上一个版本）、`DB_ROW_ID`（6B，无主键时的隐藏主键）。
2. **undo log 版本链**：每次修改，旧版本写入 undo，通过 `DB_ROLL_PTR` 串成链。
3. **ReadView**：事务进行快照读时生成，包含：
   - `m_ids`：生成时刻**活跃（未提交）事务**的 ID 列表；
   - `min_trx_id`（m_up_limit_id）：`m_ids` 中的最小值；
   - `max_trx_id`（m_low_limit_id）：生成时刻**将要分配给下一个事务**的 ID；
   - `creator_trx_id`：创建该 ReadView 的事务自己的 ID。

**可见性判断**（对版本链上每个版本的 `trx_id`）

```
trx_id == creator_trx_id           → 可见（自己改的）
trx_id <  min_trx_id               → 可见（生成前已提交）
trx_id >= max_trx_id               → 不可见（生成后才开启的事务）
min_trx_id <= trx_id < max_trx_id:
    trx_id 在 m_ids 中              → 不可见（生成时还未提交）
    trx_id 不在 m_ids 中            → 可见（已提交）
不可见则沿 roll_ptr 找上一个版本，重复判断
```

**RC 与 RR 的本质差别**：ReadView 的生成时机——RC 每次 SELECT 重新生成，所以能读到其他事务新提交的内容；RR 只在首次一致性读生成一次（`START TRANSACTION WITH CONSISTENT SNAPSHOT` 会立即生成）。

### Q3.4 快照读与当前读？RR 能彻底解决幻读吗？★★★

- **快照读**：普通 `SELECT`，读 ReadView 对应的历史版本，不加锁。
- **当前读**：`SELECT ... FOR UPDATE / LOCK IN SHARE MODE`、`UPDATE`、`DELETE`、`INSERT`，读最新已提交版本并加锁。
- **纯快照读场景**：RR 下 ReadView 固定，不会出现幻读。
- **纯当前读场景**：next-key lock（记录锁 + 间隙锁）锁住范围，阻止其他事务插入，避免幻读。
- **混用会出现幻读**（经典反例）：

```sql
-- 表 t(id pk, name)，当前无 id=3
-- 事务 A
BEGIN;
SELECT * FROM t WHERE id = 3;          -- 快照读，无记录
-- 事务 B：INSERT INTO t VALUES (3,'x'); COMMIT;
UPDATE t SET name='y' WHERE id = 3;    -- 当前读，成功更新（读到了 B 提交的行）
SELECT * FROM t WHERE id = 3;          -- 快照读，现在能看到 id=3 → 幻读
```

- 原因：UPDATE 把该行的 `DB_TRX_ID` 改成了事务 A 自己，ReadView 判断 `trx_id == creator_trx_id` 可见。
- 结论：**RR 在"纯快照读"或"纯当前读"下不出现幻读，混用时仍可能出现**；标准 SQL 中 RR 本就允许幻读，InnoDB 是做得更好，不是完全解决。若要避免，需要在事务开头就用加锁读。

### Q3.5 为什么很多互联网公司把隔离级别设为 RC？★★★

- **RR 的问题**：next-key lock / 间隙锁锁定范围大，并发插入互相阻塞，**死锁概率高**，尤其无索引或范围更新时可能退化为近似锁表。
- **RC 的优势**：
  - 加锁读只加记录锁，没有间隙锁，锁冲突和死锁更少；
  - 不满足 WHERE 条件的行锁会在 Server 层判断后提前释放；
  - 每次读都是最新已提交数据，业务语义更直观；
  - UPDATE 时有 semi-consistent read 优化。
- **代价**：
  - 必须使用 `binlog_format=ROW`，STATEMENT 格式在 RC 下会导致主从数据不一致（InnoDB 会拒绝）；
  - 不可重复读、幻读需要业务接受或自行用加锁读解决；
  - 事务内多次查询结果可能不同，报表/长事务一致性读需要额外考虑。
- 选择依据：业务对一致性读的要求 + 并发写冲突程度。金融强一致场景可能更偏好 RR + 严格加锁规范。

### Q3.6 undo log 的类型与 purge？长事务的危害？★★★

- **insert undo**：仅用于回滚，事务提交后即可释放。
- **update undo**（含 delete）：既用于回滚，也用于 MVCC 版本链，**提交后不能立即删除**，需要等所有可能引用它的 ReadView 都消失后由 **purge 线程**回收。
- **delete 实现**：先把记录标记为 delete-mark（逻辑删除），由 purge 在确认无事务可见后才物理删除并释放空间。
- **长事务危害**：
  1. 阻止 purge，undo 无限增长（`SHOW ENGINE INNODB STATUS` 中 `History list length` 持续增大），回滚段膨胀占用磁盘、查询变慢（版本链过长）；
  2. 长时间持有行锁、MDL 锁，造成大面积阻塞；
  3. 主从延迟（从库要等大事务整体重放）；
  4. 回滚代价极大，崩溃后恢复慢。
- **治理**：监控 `information_schema.innodb_trx`（`trx_started`），设置合理超时；业务上避免事务里做 RPC/外部调用；`autocommit=1`；大批量操作拆分；`max_execution_time` 限制查询。

### Q3.7 二级索引如何做 MVCC？★★★

- 二级索引记录里**没有** `DB_TRX_ID` 和 `DB_ROLL_PTR`。每个二级索引页的头部有 **`PAGE_MAX_TRX_ID`**（修改过该页的最大事务 ID）。
- 判断某条二级索引记录对当前 ReadView 是否可见：如果 `PAGE_MAX_TRX_ID` 小于 ReadView 的 `min_trx_id`，该页所有记录都可见，**无需回表**；否则必须回表到聚簇索引按版本链判断可见性。
- 所以覆盖索引在**页被较新事务修改过**时也可能需要回表确认。

### Q3.8 事务使用的最佳实践与大事务拆分？★★

- 事务尽可能短：只包含必须原子的 DB 操作，**不要在事务中调用外部服务 / 等待用户 / 大量计算**。
- 加锁顺序保持一致；热点行的更新放到事务**最后**，缩短持有时间。
- 批量更新：按主键范围分批（每批 500\~2000 行），每批提交一次，避免一次性大事务造成锁/undo/binlog/主从延迟。
- 8.0 的 `SELECT ... FOR UPDATE SKIP LOCKED / NOWAIT` 适合任务队列、抢占式调度（避免多个消费者互相等待）。
- 幂等设计：唯一索引、状态机条件更新（`where status=旧状态`）。

---

## 4. 锁

### Q4.1 InnoDB 有哪些锁？★★

| 类别 | 说明 |
| --- | --- |
| **共享锁 S / 排他锁 X** | 行级锁的两种基本模式 |
| **意向锁 IS / IX** | **表级**锁，表示事务稍后会在某些行上加 S/X 锁。作用：表锁请求（如 `LOCK TABLES`、DDL）无需遍历整张表检查行锁，只要检查意向锁即可 |
| **记录锁 Record Lock** | 锁住索引记录本身 |
| **间隙锁 Gap Lock** | 锁住索引记录之间的间隙（开区间），只在 RR 下出现，防止其他事务在间隙插入 |
| **临键锁 Next-Key Lock** | 记录锁 + 该记录前面的间隙，左开右闭区间 `(a, b]`，是 RR 下加锁的基本单位 |
| **插入意向锁 Insert Intention** | 一种特殊的间隙锁，INSERT 在插入前设置，多个事务向同一间隙的不同位置插入时互不阻塞 |
| **自增锁 AUTO-INC** | 表级特殊锁，保证自增值连续（受 `innodb_autoinc_lock_mode` 影响） |
| **MDL** | Server 层元数据锁，保护表结构 |

**兼容矩阵**（表级意向锁）

|  | IS | IX | S | X |
| --- | --- | --- | --- | --- |
| **IS** | ✓ | ✓ | ✓ | ✗ |
| **IX** | ✓ | ✓ | ✗ | ✗ |
| **S** | ✓ | ✗ | ✓ | ✗ |
| **X** | ✗ | ✗ | ✗ | ✗ |

**间隙锁的特点**：间隙锁之间**不互斥**（多个事务可以同时持有同一间隙的间隙锁），但间隙锁会阻塞**插入意向锁**。这正是后面典型死锁的成因。

### Q4.2 RR 下的加锁规则？★★★

> 以下为业界通用总结（丁奇《MySQL 实战 45 讲》），**以 5.7 / 8.0 早期为基准**，不同小版本略有差异，面试时说明"规则 + 版本可能不同"。

1. **原则 1**：加锁的基本单位是 next-key lock（左开右闭）。
2. **原则 2**：查找过程中**访问到的对象**才会加锁。
3. **优化 1**：索引上的**等值查询**，给**唯一索引**加锁时，next-key lock 退化为**记录锁**。
4. **优化 2**：索引上的等值查询，向右遍历且最后一个值**不满足**等值条件时，next-key lock 退化为**间隙锁**。
5. **唯一索引的范围查询**会访问到第一个不满足条件的值为止（该行为在后续 8.0 版本中有调整）。

**示例**：表 `t(id PK, c 普通索引, d)`，数据 id/c 为 0, 5, 10, 15, 20, 25，RR 级别。

| SQL | 加锁情况 | 说明 |
| --- | --- | --- |
| `where id=10 for update` | 仅 id=10 记录锁 | 唯一索引等值命中，退化为记录锁 |
| `where id=7 for update` | 间隙锁 (5,10) | 记录不存在，退化为间隙锁 |
| `where id>=10 and id<11 for update` | id=10 记录锁 + (10,15\] next-key | 范围查询，向右访问到 15 才停止 |
| `where c=10 lock in share mode` | c 索引 (5,10\] + (10,15) 间隙 | 普通索引等值：先 next-key，再向右遍历到 15 不满足，退化为间隙锁 |
| 上一条改成只查覆盖索引列 `select id ...` | 主键上不加锁 | `lock in share mode` 且覆盖索引不访问主键；**`for update` 会顺便给主键加锁** |
| `where d=5 for update`（d 无索引） | 扫描聚簇索引所有记录，全部加 next-key lock | 相当于**锁全表**，RR 下尤其危险 |

- **LIMIT 的作用**：加 `limit` 后扫描到足够行数即停止，不会继续向右加间隙锁，能缩小锁范围。
- **RC 下**：只有记录锁；全表扫描时不满足条件的行锁会在 Server 层判断后立即释放（`unlock_row`），但**加锁过程中仍有被短暂持有**的开销。
- **唯一键冲突**：INSERT 遇到重复键会在冲突索引记录上加 **S 锁（next-key）**，这是并发 insert 死锁的常见来源。

### Q4.3 什么情况下行锁会变成"锁表"？★★

- **没有走索引**：UPDATE/DELETE/SELECT FOR UPDATE 的条件不能使用索引，InnoDB 需要扫描聚簇索引全部记录，RR 下所有记录和间隙都被锁，**效果等同表锁**。
- 索引失效（隐式转换、函数）同理。
- 所以："InnoDB 行锁是加在索引上的，没有索引或索引失效就会锁住所有扫描到的行。"
- 排查：`EXPLAIN` 看 `type=ALL`；`performance_schema.data_locks` 查看实际锁。

### Q4.4 死锁是如何产生的？如何排查和预防？★★★

**经典场景一：加锁顺序不一致**

```
T1: update t set ... where id=1;   T2: update t set ... where id=2;
T1: update t set ... where id=2;   T2: update t set ... where id=1;   → 死锁
```

**经典场景二：间隙锁兼容 + 插入意向锁互斥（RR）**

| 时刻 | T1 | T2 |
| --- | --- | --- |
| 1 | `select * from t where id=7 for update`（持有 (5,10) 间隙锁） | `select * from t where id=8 for update`（也持有 (5,10) 间隙锁，间隙锁互相兼容） |
| 2 | `insert id=7` → 要插入意向锁，被 T2 的间隙锁阻塞 | `insert id=8` → 被 T1 的间隙锁阻塞 → **死锁** |

**场景三**：并发 `insert ... on duplicate key update` / 唯一键冲突的 S 锁升级 X 锁；二级索引与主键加锁顺序不同（一个事务先锁二级索引再锁主键，另一个相反）。

**检测与处理**

- `innodb_deadlock_detect`（默认 ON）：维护 **wait-for graph**，发现环后选择**代价最小（修改行数少 / undo 少）**的事务回滚。**死锁回滚的是整个事务**；而锁等待超时（`innodb_lock_wait_timeout`，默认 50s）默认只回滚**当前语句**（`innodb_rollback_on_timeout` 默认 OFF）。
- 热点行场景下，大量事务争同一行，死锁检测本身（O(n²)）会耗尽 CPU。解决：在应用层控制并发（排队/拆分行）；可关闭检测，但要缩短 `innodb_lock_wait_timeout`。
- **排查**：
  - `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK`（只记录最近一次）；
  - `innodb_print_all_deadlocks=ON` 把所有死锁写入 error log；
  - 8.0：`performance_schema.data_locks`、`data_lock_waits`，`sys.innodb_lock_waits`；5.7：`information_schema.innodb_locks / innodb_lock_waits`。
- **预防**：
  1. 事务尽量短小；
  2. 按固定顺序访问表和行（如按主键升序）；
  3. 更新/删除走**精确的唯一索引**，避免范围、无索引；
  4. 降低隔离级别到 RC；
  5. 批量更新先排序再执行；
  6. 失败重试（捕获 1213 错误码，退避重试）。

### Q4.5 悲观锁与乐观锁？如何设计库存扣减？★★

- **悲观锁**：`SELECT ... FOR UPDATE` 先锁后改，适合写冲突多、一致性要求强的场景；需缩短事务，防死锁。
- **乐观锁**：加 `version` 或状态条件，更新时校验，失败重试。适合冲突少的场景：

  ```sql
  UPDATE stock SET num = num - 1, version = version + 1
  WHERE sku_id = ? AND version = ? AND num > 0;
  ```

  冲突高时大量重试，反而不如直接排队。
- **库存扣减推荐**：**单条原子 SQL** `UPDATE stock SET num=num-? WHERE sku_id=? AND num>=?`，通过 `affected_rows` 判断是否成功，不需要先 SELECT 再 UPDATE（避免 check-then-act 的竞态和多余的锁持有）。

### Q4.6 MDL 与表锁？★★

- 见 [Q2.11](#q211-online-ddl-%E4%B8%8E-mdl-%E9%94%81%E9%98%BB%E5%A1%9E)。重点：长事务 + DDL = 整表阻塞。
- `LOCK TABLES`、`FLUSH TABLES WITH READ LOCK`（FTWRL，全局读锁，旧版备份使用，会阻塞所有写）。8.0 引入 **Backup Lock**（`LOCK INSTANCE FOR BACKUP`）阻止 DDL 但允许 DML，物理备份工具使用。
- `mysqldump --single-transaction`：利用 RR 的一致性快照，在不加全局锁的情况下备份 InnoDB 表。

---

## 5. 日志与崩溃恢复

### Q5.1 redo log / undo log / binlog 的区别？★

|  | redo log | undo log | binlog |
| --- | --- | --- | --- |
| 所属 | InnoDB 引擎层 | InnoDB 引擎层 | Server 层 |
| 内容 | 物理逻辑日志：某页某位置的修改 | 逻辑日志：反向操作/旧版本 | 逻辑日志：SQL 语句或行变更（取决于格式） |
| 写入方式 | 循环写，空间固定 | 随事务产生，由 purge 清理 | **追加写**，满了切下一个文件 |
| 作用 | **崩溃恢复（crash-safe）**，保证持久性 | **回滚 + MVCC**，保证原子性和隔离性 | **复制、数据恢复（PITR）、数据订阅**（canal） |
| 恢复方向 | 前滚 | 回滚 | 重放（逻辑） |

### Q5.2 redo log 的原理？★★

- **WAL**：先写日志，后写数据页；数据页异步刷盘。日志是顺序写，数据页是随机写。
- **LSN**：日志序列号，单调递增，标识 redo 写入位置；每个数据页记录其 `FIL_PAGE_LSN`，恢复时对比决定是否需要重放。
- **checkpoint**：表示该 LSN 之前的脏页都已落盘，redo 中此前的部分可被覆盖。Fuzzy Checkpoint 在后台逐步推进。
- **mtr（mini-transaction）**：对页的一组修改的原子单位，redo 以 mtr 为单位写入，保证页级修改的原子性。
- **redo 写满怎么办**：checkpoint 无法推进 → 必须强制刷脏页 → 业务出现**停顿/抖动**。所以 redo 容量不能太小（8.0.30+ 调 `innodb_redo_log_capacity`，旧版调 `innodb_log_file_size`）。
- **8.0 无锁 redo 写入（8.0.11+）**：重构了 log buffer 的并发写入与刷盘，大幅提高高并发写入的吞吐。

### Q5.3 `innodb_flush_log_at_trx_commit` 与 `sync_binlog`？双 1 是什么？★★

**`innodb_flush_log_at_trx_commit`**

| 值 | 行为 | 风险 |
| --- | --- | --- |
| 0 | 每秒由后台线程写入并刷盘 | MySQL 进程崩溃也会丢最近约 1s 的事务 |
| **1** | 每次提交都写入并 fsync | 不丢（真正的 ACID，默认） |
| 2 | 每次提交写入 OS Page Cache，每秒 fsync | 进程崩溃不丢，**OS 崩溃/掉电**丢最近约 1s |

**`sync_binlog`**：0（由 OS 决定何时刷盘）、1（每次提交刷盘）、N（每 N 个事务组提交刷盘）。

**双 1** = `innodb_flush_log_at_trx_commit=1` + `sync_binlog=1`，强一致、最安全；写入 TPS 受 fsync 延迟限制。**组提交**缓解这一代价。

### Q5.4 组提交（Group Commit）是什么？★★★

- 问题：每个事务都 fsync 太慢。
- 做法：多个并发事务的提交被分成三个阶段，**每个阶段有一个 leader 带领一组事务批量处理**：
  1. **Flush 阶段**：把一批事务的 binlog 写入 binlog 文件（OS 缓存），并按顺序分配 commit 顺序；
  2. **Sync 阶段**：对 binlog 做**一次 fsync**；
  3. **Commit 阶段**：按顺序完成 InnoDB 的 commit，redo commit。
- 一次 fsync 服务多个事务，吞吐大幅提升。调节 `binlog_group_commit_sync_delay`（微秒，等待更多事务加入）与 `binlog_group_commit_sync_no_delay_count`，用延迟换吞吐。
- **与并行复制的关系**：同一组提交的事务在主库上没有冲突，可以在从库并行回放（5.7 LOGICAL_CLOCK 的基础）。

### Q5.5 为什么需要两阶段提交？崩溃恢复如何判断？★★★

**目的**：保证 redo log 与 binlog 的**一致性**。若不一致，主库恢复后的数据与从库/备份（依赖 binlog）不一致。

**流程**

1. **Prepare**：InnoDB 写 redo log，状态为 prepare（并将 XID 写入）；
2. 写 **binlog**；
3. **Commit**：InnoDB 将 redo log 标记 commit。

**崩溃点分析**

| 崩溃时刻 | 恢复结果 |
| --- | --- |
| prepare 之前 | 事务未开始持久化，回滚，无影响 |
| redo prepare 完成，binlog **未写完** | binlog 无完整事务 → **回滚**（主从都没有该事务，一致） |
| binlog 写完，redo **未 commit** | redo 为 prepare 状态，通过 XID 在 binlog 中找到**完整事务** → **提交**（binlog 已有，从库会应用，主库必须提交才能一致） |
| redo commit 之后 | 正常 |

- **binlog 完整性判断**：row 格式末尾有 `XID event`；statement 格式末尾有 `COMMIT`；配合 `binlog_checksum`（CRC32）校验。
- **为什么不能只靠 redo 或只靠 binlog？**
  - 只有 redo：循环写覆盖，**没有归档能力**，无法 PITR；也无法做主从复制（Server 层概念）。
  - 只有 binlog：没有 checkpoint 信息，无法知道哪些修改已经落到数据页、哪些没有，**不具备 crash-safe 能力**；而且 binlog 是逻辑日志，不能精确恢复到页的状态（MyISAM 就没有 crash-safe）。
- **性能代价**：多一次 fsync（prepare 的 redo 刷盘 + binlog 刷盘），所以才有组提交优化。

### Q5.6 binlog 三种格式的区别？如何选？★★

| 格式 | 内容 | 优点 | 缺点 |
| --- | --- | --- | --- |
| STATEMENT | SQL 原文 | 日志小 | 非确定性函数（`now()`、`uuid()`、`rand()`、`limit` 无 order by）、触发器、UDF 等可能导致主从不一致 |
| **ROW**（推荐，8.0 默认） | 每一行变更前后的值 | 精确，利于数据恢复、flashback、CDC | 日志量大，批量更新时日志体积大 |
| MIXED | 默认 STATEMENT，必要时自动切 ROW | 折中 | 判断规则复杂，不够透明 |

- `binlog_row_image`：`FULL`（默认，记录所有列）、`MINIMAL`（只记录变更列和主键，更省空间；但对数据恢复/订阅不友好）。
- 使用 RC 隔离级别、并行复制 WRITESET、CDC/闪回都要求 **ROW**。
- 注意：**ROW 格式下无主键的表**，从库回放每行都要全表扫描，会造成严重主从延迟 → 表必须有主键。

### Q5.7 InnoDB 崩溃恢复流程？★★★

1. 读取 checkpoint LSN；
2. **前滚（redo apply）**：从 checkpoint 开始扫描 redo，对比页的 LSN，重放没有落盘的修改；
3. 结合 binlog 判断 prepare 状态的事务是提交还是回滚（Q5.5）；
4. **回滚（undo）**：对未提交的事务，利用 undo 日志在后台回滚；
5. 恢复时可借助 doublewrite 修复损坏页；
6. 由于回滚在后台线程异步进行，大事务崩溃后回滚可能耗时很久，期间可能持有锁。

### Q5.8 脏页刷新与"抖动"问题？★★★

**刷脏页的触发**：① redo log 写满（checkpoint 需要推进）；② Buffer Pool 空闲页不足，需要淘汰脏页；③ 后台空闲时刷；④ 正常关闭。

**抖动表现**：SQL 偶尔变慢，监控出现周期性毛刺，原因通常是刷脏页占满 IO。

**影响因素和参数**

- `innodb_io_capacity`：告诉 InnoDB 磁盘 IOPS 能力，应设为接近磁盘真实随机 IOPS（SSD 设置过低 → 脏页刷不动堆积 → 抖动）。`innodb_io_capacity_max` 是上限。
- `innodb_max_dirty_pages_pct`（90）/`innodb_max_dirty_pages_pct_lwm`：脏页比例上限及开始提前刷的低水位。
- `innodb_flush_neighbors`：刷脏页时是否连带邻接页。机械盘建议 1（减少寻道），SSD 建议 0（8.0 默认 0）。
- `innodb_flush_method=O_DIRECT`：绕过 OS Page Cache，避免双重缓存。
- 加大 redo 容量。
- 监控 `Innodb_buffer_pool_pages_dirty / pages_total`，以及 redo 的 checkpoint age。

---

## 6. SQL 执行、优化器与调优

### Q6.1 一条 SQL 的执行过程？★

1. **连接器**：认证、权限、维持连接（`wait_timeout` 空闲超时）。
2. ~~查询缓存~~：8.0 已**移除**。
3. **解析器**：词法分析、语法分析，生成语法树。
4. **预处理器**：检查表/字段是否存在、解析别名、权限。
5. **优化器**：选择索引、决定 join 顺序、改写 SQL，生成执行计划（基于成本）。
6. **执行器**：调用存储引擎接口逐行执行，返回结果；更新操作还要涉及 binlog、redo、两阶段提交。

### Q6.2 EXPLAIN 各字段怎么看？★★

| 字段 | 关键点 |
| --- | --- |
| `type` | 从好到差：`system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL`。要求至少 `range`，最好 `ref` 及以上；`index` 是全索引扫描；`ALL` 是全表扫描 |
| `possible_keys` / `key` | 可能使用 / 实际使用的索引 |
| `key_len` | 实际使用的索引长度，用来判断联合索引用到了几列。计算：INT 4，BIGINT 8，DATETIME 5（含小数秒会增加），TIMESTAMP 4，`VARCHAR(n)` 的 utf8mb4 = `n*4 + 2`，`CHAR(n)` utf8mb4 = `n*4`；**允许 NULL 再 +1** |
| `ref` | 与索引比较的列或常量 |
| `rows` | **估算**需要扫描的行数 |
| `filtered` | 存储引擎返回行经 Server 层过滤后剩余的百分比 |
| `Extra` | `Using index`（覆盖索引）、`Using index condition`（ICP）、`Using where`（Server 层过滤）、`Using temporary`（用临时表，常见于 group by / distinct）、`Using filesort`（额外排序）、`Using join buffer (hash join / Block Nested Loop)`（被驱动表无可用索引）、`Using MRR`、`Backward index scan`、`Select tables optimized away` |

- `EXPLAIN ANALYZE`（8.0.18+）：**真正执行**并给出每个节点的实际耗时、实际行数，对比估算行数，是定位估算偏差的利器。
- `EXPLAIN FORMAT=TREE`：以树形展示执行流程。
- `optimizer_trace`：查看优化器为什么选这个计划。

### Q6.3 Join 的实现算法？驱动表选择？★★★

- **Index Nested-Loop Join（INLJ）**：驱动表每一行，到被驱动表的**索引**上查找，复杂度约 `N × log(M)`。被驱动表 join 列有索引时使用。
- **Block Nested-Loop Join（BNL）**：被驱动表无索引时，把驱动表数据分块放入 `join_buffer`，每块扫描一遍被驱动表。复杂度 `N × M` 级，严重时会污染 Buffer Pool。
- **Hash Join（8.0.18+）**：对较小的表建哈希表，扫描另一表探测。**8.0.20 起取代 BNL**，对等值 join 无索引时性能显著提升。
- **BKA（Batched Key Access）**：配合 MRR，批量取出 join key 再回表，减少随机 IO。
- **驱动表选择**：核心原则是"**小结果集驱动大结果集**"——指的是**经过 WHERE 过滤之后**参与 join 的行数少的表做驱动表，**并且被驱动表的 join 列有索引**。LEFT JOIN 的驱动表通常固定为左表（优化器可能在可行时转化）。
- 优化：
  1. 被驱动表的 join 字段**建索引**；
  2. join 字段类型、字符集、排序规则**必须一致**（否则索引失效）；
  3. 必要时用 `STRAIGHT_JOIN` 强制顺序；
  4. 减少 join 的表数量（一般不超过 3 张）；
  5. 大表之间的关联放到应用层或离线。

### Q6.4 `IN` 与 `EXISTS`？子查询优化？★★

- 老理论："外表大用 IN，外表小用 EXISTS"。现代版本优化器已经做了 **semi-join 优化**（materialization、FirstMatch、LooseScan、Duplicate Weedout），5.6 之前 `IN (subquery)` 会被改写成依赖子查询导致很慢，5.6+ 基本不用手工区分。
- 建议：用 `EXPLAIN` 验证；能改成 `JOIN` 的就改（注意去重）；`NOT IN` 注意 **NULL 陷阱**（子查询结果含 NULL 时结果恒为空），优先用 `NOT EXISTS`。

### Q6.5 深分页怎么优化？`limit 1000000, 10` 为什么慢？★★

- **为什么慢**：需要先**扫描并丢弃**前 100 万行，若 `select *`，每一行还要回表，开销巨大。
- **方案**：
  1. **游标 / 记录上次位置（seek method）**：`where id > last_id order by id limit 10`，复杂度与页码无关（强烈推荐，适合"下一页"型）；
  2. **延迟关联**：先在覆盖索引上取出目标页的主键，再回表：

     ```sql
     SELECT t.* FROM t
     JOIN (SELECT id FROM t WHERE ... ORDER BY id LIMIT 1000000, 10) x ON t.id = x.id;
     ```
  3. 业务限制最大页码（搜索引擎、Google 都不允许无限翻页）；
  4. 使用 ES 等搜索引擎；
  5. 分库分表下的深分页更麻烦（各分片取前 `offset+limit` 行再归并），同样优先游标方案。
- **分页稳定性**：排序字段不唯一会导致分页重复/漏数据，`order by` 加上主键作为二级排序键。

### Q6.6 慢查询优化的完整套路？★★

1. **发现**：`slow_query_log`、`long_query_time`、`log_queries_not_using_indexes`；`pt-query-digest` 汇总；`performance_schema.events_statements_summary_by_digest`、`sys.statement_analysis`；云厂商提供的 SQL 洞察。
2. **分析**：`EXPLAIN` / `EXPLAIN ANALYZE`；看 `rows`、`type`、`Extra`；`SHOW PROFILE`（已废弃）→ 用 `performance_schema`。
3. **常见手段**：
   - 加/改索引、利用覆盖索引、联合索引顺序；
   - 避免 `select *`；
   - 避免函数/隐式转换；
   - 改写子查询、分页；
   - 拆分复杂 SQL（大 SQL 拆多个小 SQL，减少锁时间）；
   - 冷热分离/归档，减少数据量；
   - 读写分离、缓存。
4. **验证**：对比优化前后的扫描行数、响应时间、锁持有时间；在从库或预发验证。
5. **治理**：SQL 上线审核（SQLAdvisor / goInception / Archery），索引规范，慢 SQL 告警。

### Q6.7 写入相关优化？★★

- **批量写入**：一条 `INSERT` 多行（建议每批几百\~几千行）；`LOAD DATA INFILE` 最快；放在一个事务里提交。
- `INSERT ... ON DUPLICATE KEY UPDATE`：存在即更新，依赖唯一索引。注意：多个唯一索引时的行为不确定、自增 ID 会被浪费、可能产生死锁。
- **`REPLACE INTO`**：实质是 `DELETE + INSERT`，会改变自增 ID，触发 DELETE 的外键/触发器行为，**不推荐**。
- `INSERT IGNORE`：忽略唯一键冲突，但同时会吞掉其他类型的错误（类型截断等降级为警告），谨慎使用。
- 大表**批量删除**：分批 `delete ... limit N`（带索引条件），每批提交，避免大事务；或者用分区表 `DROP PARTITION`；或建新表迁移后 rename。
- 大表**批量更新**：按主键范围分批。

### Q6.8 关键参数调优？★★

| 参数 | 建议 |
| --- | --- |
| `innodb_buffer_pool_size` | 专用机物理内存的 50%\~75%；注意预留给连接线程缓冲与 OS |
| `innodb_flush_log_at_trx_commit` / `sync_binlog` | 强一致用 1/1；极端写入优先可权衡 |
| `innodb_flush_method` | `O_DIRECT` |
| `innodb_io_capacity` / `_max` | 与磁盘 IOPS 匹配（SSD 通常数千到上万） |
| `innodb_redo_log_capacity`（8.0.30+）/`innodb_log_file_size` | 足够大，避免 redo 写满导致抖动 |
| `max_connections` | 配合连接池，而不是无限增大；每个连接都有线程栈和缓冲区开销 |
| `sort_buffer_size` / `join_buffer_size` / `tmp_table_size` | **每个连接/每次操作**分配，不要盲目调大，高并发下可能撑爆内存 |
| `innodb_lock_wait_timeout` | 默认 50s 过长，互联网业务常调为 5\~10s |
| `transaction_isolation` | 按业务选择 RC / RR |
| `binlog_expire_logs_seconds` | 控制 binlog 保留（8.0 起替代 `expire_logs_days`） |
| `innodb_dedicated_server` | 8.0 提供的自动配置，对专用服务器会按内存自动设置 buffer pool、redo 等 |

---

## 7. 复制与高可用

### Q7.1 主从复制原理？★

1. 主库提交事务后将变更写入 **binlog**；
2. 主库 **binlog dump 线程** 把 binlog 推送给从库；
3. 从库 **IO 线程** 接收并写入本地 **relay log**；
4. 从库 **SQL 线程**（并行复制下为 coordinator + 多个 worker）读取 relay log 并回放。

**复制模式**

- **异步复制**：主库不等从库确认，性能最好，故障切换可能丢数据。
- **半同步复制**：主库提交后至少等一个从库确认收到 binlog。`AFTER_SYNC`（无损复制，5.7 默认）在**提交之前**等待，保证主库故障时已提交事务一定存在于某个从库；`AFTER_COMMIT`（5.5/5.6）可能存在幻读问题。超时会**退化为异步**。
- **全同步 / 组复制（MGR）**：基于 Paxos 变种，多数派确认，支持单主 / 多主，自动选主，故障检测。要求 InnoDB、每表有主键、GTID、ROW。大事务和网络波动会影响吞吐，有流控。

### Q7.2 主从延迟的原因和解决方案？★★★

**原因**

1. **大事务**：主库执行 10 分钟的事务，从库也要至少 10 分钟，且期间后续事务排队。
2. **从库单线程回放**：主库并发写、从库串行回放。
3. 从库**硬件/IO 配置差**，或从库承担了大量读压力/备份。
4. **ROW 格式 + 无主键表**：每行变更都全表扫描。
5. 大 DDL（Online DDL 在从库需要重放，期间阻塞复制）。
6. 锁等待：从库的长事务/查询与回放线程冲突。
7. 网络抖动、跨机房复制。

**并行复制演进**

| 版本 | 策略 |
| --- | --- |
| 5.6 | 按**库**并行（单库场景无效） |
| 5.7 | **LOGICAL_CLOCK**：主库组提交内的事务（`last_committed` 相同）无冲突，可并行 |
| 8.0 | **WRITESET**（`binlog_transaction_dependency_tracking=WRITESET`）：基于写集合（行的主键/唯一键哈希）判断冲突，没有冲突就并行，并行度远高于 LOGICAL_CLOCK |

相关参数：`replica_parallel_workers`（旧名 `slave_parallel_workers`）、`replica_parallel_type=LOGICAL_CLOCK`、`replica_preserve_commit_order`。

**解决**：拆大事务；开启并行复制；提升从库配置；表必须有主键；读压力分流；监控延迟（`Seconds_Behind_Source` 不够准确，更推荐 **heartbeat 表 / pt-heartbeat**）。

### Q7.3 读写分离下如何解决"读到旧数据"？★★★

1. **强制走主库**：对一致性要求高的查询（刚写完立即读）。
2. **延迟读 / 判断主从无延迟**：
   - 查 `Seconds_Behind_Source = 0`（精度到秒，不严谨）；
   - 对比主从位点：`Master_Log_File/Read_Master_Log_Pos` 与 `Relay_Master_Log_File/Exec_Master_Log_Pos` 是否相同；
   - **GTID 方式**：`WAIT_FOR_EXECUTED_GTID_SET(gtid, timeout)` 等待从库执行到指定事务，再读；
   - 位点方式：`SELECT MASTER_POS_WAIT(file, pos, timeout)`。
3. **半同步**（只保证收到，不保证已回放，仍要配合 2）。
4. **缓存兜底**：写入后在缓存里标记"某用户 N 秒内读主"（会话粘滞）。
5. 中间件（ProxySQL / ShardingSphere / MaxScale）支持"写后读主"策略、GTID 一致性读。

### Q7.4 GTID 与 binlog 位点复制？★★

- **GTID**：`server_uuid:transaction_id`，全局唯一标识一个事务。
- **优势**：从库通过 `MASTER_AUTO_POSITION=1`（8.4 为 `SOURCE_AUTO_POSITION=1`）自动定位同步点，**故障切换不用手动找位点**；便于判断事务是否已执行；MGR 依赖。
- **限制**：不支持 `CREATE TABLE ... SELECT`（在 GTID 模式下受限）、事务内混合事务/非事务引擎、临时表规则等。
- **GTID 空洞/错误事务修复**：在从库上注入空事务（`SET GTID_NEXT='uuid:n'; BEGIN; COMMIT;`）跳过某个事务。

### Q7.5 高可用方案对比？★★★

| 方案 | 说明 | 注意点 |
| --- | --- | --- |
| **MHA** | 脚本 + manager 监控，主挂时选择最新从库并补齐 relay log | 老牌，需防脑裂；社区较少维护 |
| **Orchestrator** | 拓扑发现、自动故障转移、可视化 | 配合 Consul/VIP/ProxySQL 使用 |
| **MySQL InnoDB Cluster** | MGR + MySQL Shell + MySQL Router | 官方方案，自动选主，强一致 |
| **Keepalived + VIP** | 浮动 IP 切换 | 需要防双主脑裂 |
| **ProxySQL / MaxScale** | 代理层读写分离 + 故障摘除 | 代理自身需要高可用 |
| **云 RDS / PolarDB / Aurora** | 托管，存算分离、共享存储、多副本 | 需要理解其与原生 MySQL 的差异 |

- **主备切换策略**：**可靠性优先**（等从库追平再切换，有短暂不可写）vs **可用性优先**（不等追平直接切，可能数据丢失/不一致）。
- **脑裂**：旧主未真正下线而又提升了新主，导致双写。通过 fencing（STONITH、VIP 回收、`super_read_only`、云平台 API 隔离）和多数派仲裁防范。
- **半同步 + 多数派 + 自动选主** 是比较稳妥的组合。

### Q7.6 备份与恢复？误删数据怎么办？★★

- **备份类型**：逻辑备份（`mysqldump`/`mysqlpump`/`mydumper`，可读性强但慢）；物理备份（**Percona XtraBackup / MySQL Enterprise Backup**，拷贝数据文件 + redo，快）；**Clone Plugin**（8.0.17，在线克隆实例，快速搭建从库）；存储/云快照。
- **PITR**：全量备份 + 之后的 binlog，恢复到指定时间点/位点/GTID（`mysqlbinlog --start-position / --stop-datetime`）。
- **误操作处理**：
  - `DELETE`/`UPDATE` 误操作：ROW 格式下用 **flashback 工具**（binlog2sql、MyFlash、dbatools）生成反向 SQL；
  - `DROP TABLE/DATABASE`：无法 flashback，用**最近全备 + binlog PITR**，在独立实例恢复后迁回；
  - **延迟复制从库**（`CHANGE REPLICATION SOURCE TO SOURCE_DELAY=3600`）：提供 1 小时反悔窗口；
  - 预防：`sql_safe_updates`、最小权限、生产变更审批/审计平台、禁止直连生产库。
- **恢复演练**：没有演练过的备份等于没有备份，要定期验证备份可恢复、恢复耗时（RTO）、数据丢失窗口（RPO）。

---

## 8. 分库分表与分布式

### Q8.1 什么时候需要分库分表？之前该做什么？★★

**不是越早越好**：分库分表引入跨库 join、分布式事务、全局排序分页、运维复杂度等一系列问题。

推荐演进路径：

1. **SQL / 索引优化**、表结构优化；
2. **缓存**（Redis）扛读；
3. **读写分离**；
4. **冷热分离 / 历史数据归档**（减小单表）；
5. **垂直拆分**（业务拆库：订单库、用户库、商品库）；
6. **水平拆分**（单表数据量过大或单库写入吞吐达到瓶颈）。

> "单表超 2000 万就要拆"只是经验值，真正的决策依据是：**单表容量（行宽、索引）、单库 QPS/TPS、IO 与磁盘容量、DDL 与备份耗时**。

### Q8.2 垂直拆分与水平拆分？分片策略？★★

- **垂直分库**：按业务域拆，各自独立数据库，微服务天然对应。
- **垂直分表**：把冷热字段/大字段（TEXT）拆到扩展表，主表更紧凑，Buffer Pool 命中更高。
- **水平分表/分库**：同一张表的数据按分片键拆到多个表/库。
- **分片策略**：

| 策略 | 优点 | 缺点 |
| --- | --- | --- |
| **Hash 取模** | 数据均匀 | 扩容需大量数据迁移 |
| **Range（范围/时间）** | 扩容简单、便于归档 | 热点集中在最新分片 |
| **一致性 Hash（+虚拟节点）** | 扩容时迁移量小 | 实现复杂，范围查询不友好 |
| **逻辑分片 + 映射**（预先分成 1024 个逻辑库，映射到少量物理库） | **扩容只需迁移逻辑库**，不需要重新 hash，业界常用 | 需要中间件/配置支持 |
| **地理/租户分片** | 数据本地化、隔离 | 大租户倾斜 |
| **目录表（Lookup）** | 灵活 | 目录表本身成为瓶颈，需缓存 |

### Q8.3 如何选择分片键？非分片键怎么查？★★★

**选择原则**：高频查询条件（让大多数查询只落一个分片）、**散列均匀**（避免数据倾斜与热点）、**不可变**（避免更新分片键导致迁移）、与业务强相关（如订单按 user_id）。

**非分片键查询的方案**

1. **冗余映射表**：`order_no → user_id`（单独的路由表/索引表，可用 KV 或另一套分片）。
2. **基因法**：把分片键的末几位嵌入主键。例如 `order_id` 末 N 位 = `user_id % 分片数`，按 order_id 或 user_id 都能路由到同一个分片。
3. **异构索引**：用 **Canal/Flink CDC/DTS 订阅 binlog** 同步到 ES 或另一套按其他维度分片的库，支持多维查询、运营后台检索。
4. **广播查询 + 聚合**：每个分片都查，应用层合并（仅适合低频、后台场景）。
5. 多维度场景（买家查订单、卖家查订单）：**双写/双库**，两份数据按不同分片键存储，异步保证最终一致。

### Q8.4 分库分表后遇到的问题与对策？★★★

| 问题 | 对策 |
| --- | --- |
| **全局唯一 ID** | 雪花、号段（Leaf/Tinyid）、Redis INCR、数据库自增步长（多个库设置不同起点和步长）；UUID 不推荐用作主键 |
| **跨库 join** | 尽量避免；小表（字典表）做**广播表**每库一份；业务层组装；宽表/ES |
| **聚合 / 排序 / 分页** | 各分片分别查询 + 归并，深分页的代价是 `offset+limit` 的数据量；用游标、限制页码；统计走离线/OLAP |
| **跨库事务** | 2PC/XA（性能差）、**TCC**、**Saga**、**本地消息表**、**RocketMQ 事务消息**（半消息 + 回查）、最大努力通知、Seata（AT 模式基于 undo 镜像） |
| **扩容 / 数据迁移** | 逻辑分片预留；**双写 + 全量迁移 + 增量同步（binlog）+ 校验 + 灰度切读 + 切写 + 回滚预案** |
| **DDL** | 逐个分片灰度执行、工具化批量执行 |
| **数据倾斜 / 热点** | 选择合适分片键，大租户单独分片，热点 key 加盐 |
| **唯一约束** | 全局唯一索引做不到，需要路由表或应用层保证；让唯一键包含分片键 |

### Q8.5 分布式事务方案对比？★★★

| 方案 | 一致性 | 特点 | 适用 |
| --- | --- | --- | --- |
| **2PC / XA** | 强一致 | 协调者 + 参与者；同步阻塞，协调者单点；性能差 | 小规模、强一致需求 |
| **TCC** | 最终一致 | Try（预留资源）/ Confirm / Cancel；业务侵入强，需要幂等、防悬挂、空回滚 | 资金、库存等核心链路 |
| **Saga** | 最终一致 | 一系列本地事务，失败则执行反向补偿；无隔离性 | 长流程 |
| **本地消息表** | 最终一致 | 业务与消息在同一本地事务，后台轮询投递，消费端幂等 | 通用、简单可靠 |
| **事务消息（RocketMQ）** | 最终一致 | 半消息 → 执行本地事务 → 提交/回滚，Broker 回查 | 异步解耦 |
| **Seata AT** | 最终一致 | 拦截 SQL 自动生成前后镜像（undo），全局锁保证写隔离 | 低侵入，注意全局锁性能 |

- **幂等**是所有最终一致方案的基础（唯一键、状态机、去重表、token）。
- **能不用分布式事务就不用**：通过合理的业务边界划分，让强一致的操作落在单库内。

### Q8.6 分区表（Partition）适用吗？★★

- MySQL 原生分区：RANGE、LIST、HASH、KEY。**对应用透明**，但仍在单实例内。
- **限制**：分区键必须是**主键/所有唯一索引的一部分**；不支持外键；跨分区查询没有分区剪裁时会扫描所有分区；每个分区是独立的 B+ 树，开销大于想象。
- **适用**：按时间的日志/流水类数据，通过 `DROP PARTITION` 秒级清理历史数据；查询基本带分区键。
- **不适用**：希望靠分区提升大表整体查询性能（索引维护仍然是每个分区一棵树）、需要水平扩展写入能力（仍在单机）。

### Q8.7 NewSQL / 分布式数据库？★★★

- **TiDB**：计算（TiDB Server，无状态 SQL 层）+ 存储（TiKV，Raft 多副本 + RocksDB）+ PD（元数据、调度、全局时间戳 TSO）。事务基于 **Percolator** 模型（2PC + MVCC + 乐观事务，4.0 起支持悲观事务）。兼容 MySQL 协议，水平扩展，自动分片（Region），强一致。
- **OceanBase**：Paxos 多副本，LSM-Tree 存储，多租户，金融级高可用。
- **云原生（Aurora / PolarDB）**：存算分离，日志即数据库（redo 下沉到存储层），共享存储、读副本秒级扩展。
- **选型思路**：数据量大且无法分库分表治理、需要强一致水平扩展 → NewSQL；追求兼容性与成熟生态、规模可控 → MySQL + 中间件。不要为了新技术而选。

---

## 9. 缓存与一致性

### Q9.1 缓存与数据库的一致性，应该先更新 DB 还是先删缓存？★★★

**推荐：Cache Aside（旁路缓存）——读：先读缓存，未命中读 DB 并回填；写：先更新 DB，再删除缓存。**

- **为什么是"删除"而不是"更新"缓存**：并发写时更新缓存会因为顺序错乱留下脏数据；缓存的计算成本可能很高，写多读少时更新缓存是浪费；删除是**懒加载**，下次读时再重建。
- **为什么先更新 DB、再删缓存**：
  - 先删缓存再更新 DB：删完缓存后、DB 更新前，另一个读请求把**旧值回填缓存**，之后 DB 更新完成 → 缓存长期是旧值，概率较高（读 DB 比写 DB 更快完成的窗口大）。
  - 先更新 DB 再删缓存：需要"读请求读取旧值后、在写请求删缓存之后才回填"的极端时序才出现脏数据，要求读比写慢（读缓存 miss 后查 DB 并回填通常比写 DB 快），**概率很低**。
- **删除失败怎么办**：
  1. 重试（应用内重试 / 异步重试）；
  2. **消息队列异步删除**，失败后重投；
  3. **订阅 binlog（Canal / Flink CDC）异步删除缓存**，业务代码无侵入，是比较稳妥的做法；
  4. **设置过期时间兜底**，保证最终一致。
- **延迟双删**：更新 DB 前后各删一次缓存，第二次延迟（如 500ms\~1s）以覆盖并发读回填的旧值。缺点：延迟时间难以确定、依旧存在窗口，更像是补救，**不是银弹**。
- **主从延迟引发的脏缓存**：写主库后删缓存，读请求从库读到**旧数据回填缓存**。对策：缓存回填时读主库 / 延迟删 / 通过订阅**从库**的 binlog 来删缓存（保证从库已更新）。
- **强一致**：加分布式锁/读写锁，代价是性能下降；大多数场景接受**最终一致**。

### Q9.2 缓存穿透、击穿、雪崩？★★

| 问题 | 描述 | 对策 |
| --- | --- | --- |
| **穿透** | 查询**不存在**的数据，每次都打到 DB | 布隆过滤器、缓存空值（短 TTL）、参数校验 |
| **击穿** | **热点 key** 过期瞬间，大量并发打到 DB | 互斥锁（只让一个线程重建）、逻辑过期（异步刷新）、热点永不过期、singleflight 合并请求 |
| **雪崩** | **大量 key 同时过期**或缓存集群宕机 | 过期时间加随机值、多级缓存（本地 + Redis）、限流降级熔断、Redis 高可用（主从/集群）、预热 |
| **热 key** | 单个 key 流量过大，压垮单节点 | 本地缓存、key 加后缀打散（多副本）、读从库 |
| **大 key** | 单个 value 过大，阻塞、网络开销 | 拆分、压缩、避免一次性获取 |

---

## 10. 存储引擎与数据库选型对比

### Q10.1 InnoDB vs MyISAM？★

|  | InnoDB | MyISAM |
| --- | --- | --- |
| 事务 | 支持 | 不支持 |
| 锁粒度 | 行锁（+表锁、MDL） | 表锁 |
| 崩溃恢复 | crash-safe（redo） | 无，易损坏 |
| 外键 | 支持 | 不支持 |
| 索引 | 聚簇索引（数据在主键 B+ 树） | 非聚簇（索引与数据分开存） |
| MVCC | 支持 | 不支持 |
| 行数统计 | 需要扫描 | 直接存储 |
| 全文索引 | 5.6+ 支持 | 支持 |

MySQL 8.0 起系统表也迁移到 InnoDB，MyISAM 基本退出生产场景。

### Q10.2 MySQL vs PostgreSQL？★★★

| 维度 | MySQL InnoDB | PostgreSQL |
| --- | --- | --- |
| **MVCC 实现** | 原地更新 + **undo log** 版本链，purge 回收 | **堆表多版本**：新版本写入新元组，旧版本留在表中，由 **VACUUM** 回收（表膨胀、需调优 autovacuum） |
| **表组织** | 聚簇索引（索引组织表） | 堆表，所有索引都指向 ctid（行指针） |
| **二级索引** | 叶子存主键，需回表（二次查找） | 叶子存 ctid，直接定位堆（但更新可能需要 HOT 优化避免索引更新） |
| **隔离级别** | RR 默认，next-key lock | 默认 RC；其 RR 实际是**快照隔离（SI）**，无幻读；SERIALIZABLE 采用 **SSI（可串行化快照隔离）** |
| **复制** | 基于 binlog 的逻辑复制 | WAL 物理流复制 + 逻辑复制 |
| **功能** | 生态成熟、运维简单 | 类型丰富（JSONB、数组、GIS）、窗口函数/CTE 强、扩展性好（PostGIS 等）、复杂查询优化器更强 |

### Q10.3 B+ 树与 LSM-Tree？★★★

|  | B+ 树 | LSM-Tree |
| --- | --- | --- |
| 写入 | 随机写（原地更新，页分裂） | **顺序写**（MemTable → SSTable，后台 compaction） |
| 读取 | 稳定，O(log N) | 可能需要查多层，读放大，依赖 Bloom Filter |
| 放大 | 写放大（页大小），空间碎片 | **写放大**（compaction）、**空间放大**、读放大 |
| 适合 | 读多写少、范围查询、OLTP | 写多读少、海量数据（日志、时序、KV） |
| 代表 | InnoDB、PostgreSQL | RocksDB、LevelDB、HBase、Cassandra、MyRocks、TiKV |

### Q10.4 OLTP vs OLAP？NoSQL 如何选型？★★

- **OLTP**（交易）：短事务、点查、高并发 → MySQL/PostgreSQL；**OLAP**（分析）：大批量扫描、聚合 → **列式存储**（ClickHouse、StarRocks、Doris、Druid），压缩比高、向量化执行。HTAP：TiDB（TiFlash 列存副本）。
- NoSQL：
  - **Redis**：缓存、计数、排行榜、分布式锁、会话；
  - **MongoDB**：文档型，灵活 schema；
  - **HBase/Cassandra**：海量宽表、写多；
  - **Elasticsearch**：全文检索、日志分析（倒排索引）；
  - **时序数据库**（InfluxDB、TDengine）、**图数据库**（Neo4j）。
- 选型要看：数据模型、一致性要求、读写比例、查询模式、数据规模、团队技术栈与运维能力。

### Q10.5 CAP / BASE / 一致性协议？★★★

- **CAP**：分布式系统在网络分区（P）发生时，只能在一致性（C）与可用性（A）间二选一。P 是无法避免的，所以实际是 CP 或 AP 之选；**PACELC** 补充：无分区时在延迟（L）与一致性（C）间权衡。
- **BASE**：基本可用（Basically Available）、软状态（Soft state）、最终一致（Eventually consistent）。
- **Raft**：Leader 选举 + 日志复制 + 多数派提交，易理解（etcd、TiKV、Consul）；**Paxos**：理论基础（MGR、OceanBase 变种）。
- MySQL 主从异步复制属于 AP 倾向；MGR / TiDB / 半同步更倾向 CP。

---

## 11. 线上故障排查

### Q11.1 "SQL 突然变慢了"的排查框架？★★★

1. **确认范围**：是某条 SQL 还是整个实例？是突发还是渐进？（监控：QPS、TPS、连接数、CPU、IO、慢查询数、锁等待、主从延迟）
2. **看当前正在跑什么**：`SHOW FULL PROCESSLIST`、`information_schema.processlist`、`sys.session`；有没有大量 `Waiting for table metadata lock`（MDL）、`Sending data`、`Copying to tmp table`、`Waiting for lock`。
3. **单条 SQL 变慢的几类原因**：
   - **执行计划变化**：统计信息变化 / 数据分布变化 / 索引选错（对比历史 EXPLAIN，`ANALYZE TABLE`，Hint）；
   - **数据量增长**：量变到质变；
   - **锁等待**：行锁/MDL 被长事务阻塞（`innodb_trx`、`data_lock_waits`）；
   - **刷脏页/redo 满导致的抖动**（看 IO 毛刺、`innodb_io_capacity`、checkpoint age）；
   - **Buffer Pool 命中率下降**（重启后冷启动、大查询冲刷缓存、内存被占用）；
   - **undo 过长 / 版本链过长**导致快照读慢（`History list length`）；
   - **资源争用**：CPU、磁盘 IO、网络、被其他业务的慢 SQL 挤占；
   - **备份/DDL 正在执行**。
4. **止血**：KILL 阻塞源头（长事务/慢查询）；限流、降级、熔断；扩容；切流量；回滚变更。
5. **根因与复盘**：保留现场（`SHOW ENGINE INNODB STATUS`、processlist、慢日志），复盘并加监控告警。

### Q11.2 CPU 飙高怎么排查？★★

1. `top -H` 找到 mysqld 里最耗 CPU 的线程，对应 `performance_schema.threads.thread_os_id` 找到 SQL；
2. 看慢查询、`Handler_read_rnd_next`（全表扫描）、`Select_scan`、`Sort_merge_passes`、`Created_tmp_disk_tables`；
3. 找到 TOP SQL（pt-query-digest、`events_statements_summary_by_digest`）；
4. 常见原因：缺失索引、一个大查询并发放大、统计汇总查询打到线上主库、连接风暴、死锁检测消耗（热点行）、AHI 争用。

### Q11.3 连接数打满怎么处理？★★

- 现象：`Too many connections`，应用报错/超时。
- 排查：`SHOW PROCESSLIST` 看大多数连接在做什么（Sleep？还是被 MDL/锁阻塞？）。
  - **大量 Sleep**：应用连接池泄漏/未归还、`wait_timeout` 过长；
  - **大量 Waiting for lock / MDL**：某个长事务或 DDL 引发的雪崩；
  - **大量慢查询**：一条慢 SQL 被高并发放大。
- 处理：先 KILL 阻塞源头；临时调大 `max_connections` 只是缓冲；根治靠**连接池限流**、SQL 优化、业务熔断、超时设置合理。
- 预防：应用连接池最大连接数 × 实例数 \< `max_connections`；设置 `max_execution_time`、`innodb_lock_wait_timeout`；中间件限流。

### Q11.4 自增 ID 用完了怎么办？★★
- INT SIGNED 上限约 21 亿，UNSIGNED 约 42.9 亿。用完后继续插入会报 `Duplicate entry` 主键冲突。
- 提前监控 `information_schema.tables.auto_increment` 与类型最大值的比例；
- 解决：在线把主键改为 `BIGINT`（`ALTER TABLE` 大表耗时长，使用 gh-ost / pt-osc）；
- 注意：**无主键表的隐藏 `DB_ROW_ID` 是 6 字节，溢出后会覆盖旧数据**（回绕），所以每张表必须显式定义主键。
- 事务 ID（`trx_id`）、`thread_id` 等也有上限，但业务很少触及。

### Q11.5 磁盘空间爆满 / 表空间碎片？★★

- 原因：binlog/relay log 未清理（`binlog_expire_logs_seconds`）；undo 膨胀（长事务）；大事务产生的临时文件；慢日志/general log；大表。
- **DELETE 不释放磁盘空间**：只是标记删除，页被复用，物理文件不会缩小。回收空间需要 `OPTIMIZE TABLE` / `ALTER TABLE t ENGINE=InnoDB`（重建表，大表用 gh-ost），或者用分区表 `DROP PARTITION`、`TRUNCATE`。
- undo 表空间：8.0 支持 `innodb_undo_log_truncate` 自动收缩；5.7 及之前系统表空间 ibdata1 只增不减，需要重建实例。

### Q11.6 误执行了无 WHERE 的 DELETE/UPDATE？★★

见 Q7.6：**ROW 格式下 flashback**（binlog2sql、MyFlash），立刻停止相关写入，在从库/新实例上生成反向 SQL 验证后回放；若无 binlog 或格式不对，则靠**备份 + PITR**。事后：开启 `sql_safe_updates`，关键库配置延迟从库，变更走审批平台。

---

## 12. 系统设计 / 场景题

### Q12.1 秒杀系统的数据库层怎么设计？★★★

1. **流量漏斗**：前端按钮防重复、CDN 静态化、网关限流、**验证码/答题削峰**，让真正到 DB 的请求数降到可承受量级。
2. **库存预扣**：Redis 里用 **Lua 脚本**原子扣减库存（防超卖），扣减成功才允许进入下单；Redis 与 DB 库存需要对账。
3. **异步下单**：扣减成功后写入 MQ，下单服务异步消费，落库削峰；用户通过轮询/推送拿结果。
4. **DB 防超卖**：落库时 `UPDATE stock SET num=num-1 WHERE sku_id=? AND num>0`，用 `affected_rows` 判断；**唯一索引防重复下单**（`user_id + sku_id`）。
5. **热点行问题**：同一行的 UPDATE 互相串行（行锁持有到事务提交），TPS 上限受单次事务耗时限制。优化：
   - **库存分桶**：一个 SKU 拆成多行（如 10 桶），随机或哈希分配到不同行，总库存 = 各桶之和；
   - 缩短事务，热点 UPDATE 放事务最后；
   - 部分云数据库提供热点行更新优化（组提交/排队）。
6. **兜底**：库存回补（订单超时未支付释放库存）、对账、补偿。

### Q12.2 转账 / 账户余额的设计？★★★

- 事务内**按固定顺序**（如账号 ID 升序）加锁两个账户，避免死锁；
- `UPDATE account SET balance=balance-? WHERE id=? AND balance>=?` 保证不透支；
- **流水表** + **唯一流水号**保证**幂等**（重试不重复扣款）；余额表是缓存性质，**以流水为准**，定期对账；
- 跨库/跨服务：TCC / 本地消息表 / 事务消息，记账服务统一管理，保证最终一致；
- 使用 `DECIMAL` 或以"分"为单位存 `BIGINT`，**绝不用 FLOAT/DOUBLE** 存金额。

### Q12.3 订单系统分库分表设计？★★★

- **分片键**：`user_id`（用户查自己订单是最高频场景）。
- **订单号**：雪花/号段，**嵌入 user_id 基因**（末几位），使按 `order_id` 查询也能路由。
- **商家侧查询**（按 seller_id）：**异构索引表**（按 seller_id 分片，异步同步）或同步到 ES。
- **订单与订单明细**：用相同分片键，保证同库，避免跨库 join。
- **历史订单**：冷热分离，3\~6 个月前的数据归档到另一个库/表/OLAP。
- **分页**：游标 + 时间范围；后台报表走离线数仓。
- **扩容**：预分 N 个逻辑库/表，扩容时迁移逻辑库，双写 + 增量同步 + 校验 + 灰度。
- **一致性**：订单 ↔ 库存 ↔ 支付通过 MQ 事务消息/TCC 保证最终一致，各环节幂等。

### Q12.4 如何设计一张用户表？字段类型怎么选？★★

- 主键 `BIGINT UNSIGNED`（雪花/号段）；唯一索引：`phone`/`email`（手机号用 `VARCHAR(20)` 或加密哈希字段做唯一索引）。
- **密码**：存加盐哈希（bcrypt/argon2），不存明文。
- **时间**：`DATETIME`（不受时区影响、范围到 9999 年）vs `TIMESTAMP`（4 字节、**2038 问题**、受会话时区影响）。统一用 UTC 或统一存本地并约定。
- **金额**：`DECIMAL(m,n)` 或分存 `BIGINT`。
- **状态**：`TINYINT` 而非 `ENUM`（ENUM 改动需要 DDL，排序按序号而非字面）。
- **字符串**：`VARCHAR` 长度够用即可（超长会影响排序/临时表的内存分配）；`TEXT/BLOB` 拆到扩展表。
- **NULL**：尽量 `NOT NULL` + 默认值（NULL 使索引/统计/比较更复杂，`count(col)`、`not in` 有坑，占用额外 NULL 标志位）。
- **字符集**：`utf8mb4`（`utf8` 是三字节阉割版，不能存 emoji）；8.0 默认排序规则 `utf8mb4_0900_ai_ci`（不区分大小写，口音不敏感），区分大小写需要 `_bin` 或 `_as_cs`。
- **软删除 vs 物理删除**：软删除会使唯一索引冲突（已删除记录占用唯一值），可用 `(phone, deleted_at)` 联合唯一，或归档表；同时软删除会让数据持续膨胀，需要定期归档。
- **必备字段**：`id`、`created_at`、`updated_at`（`ON UPDATE CURRENT_TIMESTAMP`）、必要时 `version`、`creator`。

### Q12.5 接口幂等怎么保证？★★

1. **唯一索引**：以业务唯一键（请求流水号、订单号）建唯一索引，重复插入失败即视为已处理。
2. **状态机/条件更新**：`UPDATE ... SET status='PAID' WHERE order_id=? AND status='CREATED'`，已处理的 `affected_rows=0`。
3. **去重表 / Token**：请求前获取 token，提交时 Redis `SETNX`/DB 唯一键校验并消费。
4. **乐观锁 version**。
5. 消息消费端：消息 ID 去重表 + 本地事务。

### Q12.6 数据库连接池大小怎么定？★★

- 经验公式（HikariCP 的推荐）：`连接数 ≈ CPU 核数 × 2 + 有效磁盘数`，**并不是越大越好**；过多的连接会增加上下文切换、锁争用，DB 反而更慢。
- 估算：`并发请求数 × 事务平均耗时`，结合 DB 能承受的活跃连接数（`Threads_running`）。
- 全局约束：`单实例连接池最大值 × 应用实例数 ≤ MySQL max_connections`，并预留运维连接。
- 配合：连接最大存活时间 \< `wait_timeout`；开启连接健康检查；设置合理的超时。

---

## 13. SQL 手写题

> 假设 MySQL 8.0，支持窗口函数与 CTE。

**① 每个部门薪资前 N 名（含并列）**

```sql
SELECT * FROM (
  SELECT e.*, DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk
  FROM employee e
) t WHERE rk <= 3;
```

> `ROW_NUMBER`（唯一序号）、`RANK`（并列后跳号）、`DENSE_RANK`（并列不跳号）的区别要能说清。

**② 第二高的薪水（不存在返回 NULL）**

```sql
SELECT (SELECT DISTINCT salary FROM employee ORDER BY salary DESC LIMIT 1 OFFSET 1) AS second_salary;
```

**③ 删除重复数据，保留 id 最小的一条**

```sql
DELETE e1 FROM email e1
JOIN email e2 ON e1.email = e2.email AND e1.id > e2.id;
```

**④ 连续登录 ≥ 3 天的用户**

```sql
SELECT DISTINCT user_id FROM (
  SELECT user_id, login_date,
         DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY) AS grp
  FROM (SELECT DISTINCT user_id, login_date FROM login_log) d
) t
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

> 思路：日期减去行号，**连续的日期得到相同的值**，按它分组计数。

**⑤ 行转列（每个学生各科成绩）**

```sql
SELECT student,
       MAX(CASE WHEN subject='math'    THEN score END) AS math,
       MAX(CASE WHEN subject='english' THEN score END) AS english
FROM score GROUP BY student;
```

**⑥ 累计求和 / 移动平均**

```sql
SELECT dt, amount,
       SUM(amount) OVER (ORDER BY dt) AS cum_amount,
       AVG(amount) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM daily_sales;
```

**⑦ 递归查询组织树（CTE）**

```sql
WITH RECURSIVE tree AS (
  SELECT id, name, parent_id, 1 AS lvl FROM dept WHERE id = 1
  UNION ALL
  SELECT d.id, d.name, d.parent_id, t.lvl + 1 FROM dept d JOIN tree t ON d.parent_id = t.id
)
SELECT * FROM tree;
```

**⑧ 查询没有订单的用户**（`LEFT JOIN ... IS NULL` 或 `NOT EXISTS`，**避免 NOT IN 的 NULL 陷阱**）

```sql
SELECT u.* FROM user u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

**⑨ 同比 / 环比**：用 `LAG(value, 1) OVER (ORDER BY month)` 取上期值再计算。

---

## 14. 常见错误认知纠正

| 错误说法 | 正确认知 |
| --- | --- |
| "RR 完全解决了幻读" | 快照读靠 MVCC、当前读靠 next-key lock，**两者混用仍可能出现幻读**（Q3.4） |
| "B+ 树比 B 树快是因为 IO 少" | 更准确：扇出更大、树更矮、范围扫描靠叶子链表、查询路径稳定 |
| "count(1) 比 count(\*) 快" | 语义和效率基本相同，优化器对 `count(*)` 做了专门优化 |
| "索引越多越好" | 增加写入代价、空间、优化器负担，需要权衡 |
| "`like 'abc%'` 不走索引" | 前缀匹配**可以**走索引，`like '%abc'` 才不行 |
| "`!=`、`is not null` 一定不走索引" | 取决于选择性和统计信息 |
| "delete 之后空间就释放了" | 只是标记删除，需要重建表才能回收物理空间 |
| "锁表是因为用了行锁" | 行锁加在索引上，**无索引/索引失效**才会扫全表加锁 |
| "加了 for update 就不会有幻读/不会有问题" | 加锁范围取决于索引和隔离级别，可能带来死锁、阻塞 |
| "先删缓存再更新 DB 是对的" | 更推荐先更新 DB 再删缓存，并有重试/binlog 订阅兜底 |
| "唯一索引比普通索引查询更快" | 查询性能差别很小；写入时唯一索引不能用 change buffer，更新代价略高 |
| "InnoDB 默认隔离级别 RR 所以不会有不可重复读" | 默认 RR，但很多公司改成 RC；要看实际配置 |
| "主从延迟看 `Seconds_Behind_Master` 就够了" | 它不准确（在 SQL 线程空闲、网络中断、IO 线程延迟时会失真），应结合位点/GTID/心跳表 |
| "单表超 2000 万必须分表" | 经验估算值，要结合行宽、索引、访问模式、硬件 |
| "分区表能提高并发写入能力" | 仍在单实例内，写入吞吐不提升，主要价值在于数据生命周期管理 |

---

## 15. 版本差异速查

| 版本 | 重要变化（面试常问） |
| --- | --- |
| **5.6** | 在线 DDL、ICP、MRR、InnoDB 全文索引、GTID、多线程复制（按库） |
| **5.7** | LOGICAL_CLOCK 并行复制、无损半同步（AFTER_SYNC）、JSON 类型、生成列、`sys` schema、在线调整 Buffer Pool 大小、`innodb_deadlock_detect` |
| **8.0** | 移除查询缓存、数据字典存入 InnoDB（**原子 DDL**）、窗口函数、CTE（含递归）、**降序索引**、**函数索引**、**不可见索引**、直方图、**INSTANT DDL**、**Hash Join**、`EXPLAIN ANALYZE`、**自增值持久化**、`SET PERSIST`、角色、Clone Plugin、无锁 redo、`SKIP LOCKED / NOWAIT`、WRITESET 并行复制、默认字符集 utf8mb4、默认 RR |
| **8.0.30+** | `innodb_redo_log_capacity`，redo 容量在线调整 |
| **8.4 LTS** | 长期支持版本。复制相关语法全面改用 `SOURCE/REPLICA` 命名（`CHANGE REPLICATION SOURCE TO`、`SHOW REPLICA STATUS`，旧的 MASTER/SLAVE 语法被移除）；`mysql_native_password` 默认禁用；部分 InnoDB 参数默认值调整（如自适应哈希索引默认关闭、`innodb_change_buffering` 默认 none）。**以官方发布说明为准** |

> MySQL 8.0 已于 2026 年 4 月 EOL；8.4 为当前长期支持线，另有 9.x 创新版。回答"你们用的什么版本"时，说明版本与升级考虑是加分项。

---

## 16. 追问清单与答题策略

### 高频追问（面试官的"连环炮"）

- 聚簇索引一定比二级索引快吗？回表代价什么时候会让优化器放弃索引？
- RC 与 RR 下，同一条 UPDATE 的加锁范围有什么不同？画出来。
- 为什么间隙锁之间不冲突？这会导致什么死锁？
- 两阶段提交中，如果 binlog 写完、redo 未 commit 时崩溃，恢复时怎么判断？为什么这样判断？
- redo 为什么能 crash-safe 而 binlog 不能？既然有 redo，还要 binlog 做什么？
- doublewrite 和 redo 的区别？为什么 redo 恢复不了坏页？
- Change Buffer 和 Buffer Pool 的关系？唯一索引为何不能用？
- 大事务在主库、从库、崩溃恢复各带来什么问题？
- 主从延迟有哪些成因？WRITESET 是如何判断可并行的？
- 如何不停机给 10 亿行大表加索引？gh-ost 的原理与风险？
- 分库分表之后，如何做 `order by ... limit` 的全局分页？
- 缓存与 DB 不一致的所有可能场景，分别如何兜底？
- 你们遇到过最严重的线上数据库事故是什么？根因、止血、复盘、预防？

### 答题策略

1. **先结论，再展开**：先用一句话给出结论，再讲原理，面试官能快速判断你的层次。
2. **用"取舍"思维**：任何方案都要说出收益和代价（如：RC 更少死锁，代价是不可重复读与必须使用 ROW binlog）。
3. **主动关联**：从索引讲到回表、从回表讲到覆盖索引、再到 ICP，体现体系化。
4. **画图/举例**：加锁规则、MVCC 可见性、两阶段提交，用具体例子（表结构 + 数据 + 时序）说明。
5. **结合线上经验**：把"知识点"落到"我们曾经遇到什么问题、怎么发现、怎么解决、如何避免"。
6. **区分版本**：主动说明"在 5.7 / 8.0 / 8.4 中的行为不同"，说明你关注细节。
7. **不确定就说边界**："这个我了解到的是 X，具体版本差异我会查官方文档验证。"比硬编强。

### 复习优先级

- **P0（必须能讲清）**：B+ 树与聚簇/二级索引、索引失效与优化、MVCC + ReadView、隔离级别与幻读、行锁/间隙锁/next-key、redo/undo/binlog 与两阶段提交、主从复制与延迟、慢 SQL 排查。
- **P1（资深加分）**：加锁规则案例、死锁分析、崩溃恢复、doublewrite、Change Buffer、脏页刷新与抖动、Online DDL 与 MDL、并行复制、分库分表与分布式事务、缓存一致性。
- **P2（拔高）**：优化器成本模型与直方图、LSM vs B+ 树、MySQL vs PostgreSQL、NewSQL 架构、MGR 与高可用、云原生数据库、容量规划与稳定性治理。

---

*文档基于 MySQL 8.0 / 8.4 的通用行为整理。具体参数默认值与行为请以所用版本的官方文档为准。*