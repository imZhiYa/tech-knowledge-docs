# 🐬 MySQL 备份恢复与 PITR——最后一次从废墟里翻出那笔账

## 写在最前面的承诺

> 这不是一份 mysqldump 参数速查表，也不是"全备+binlog"的八股流程。
>
> 本文只追踪一条主线：**一条 UPDATE 已经提交、已经复制到从库、甚至所有数据库实例都已经不可用之后，如何靠备份介质和 binlog 补页，把数据恢复到某个精确到秒甚至到事务的时点**——以及为了这一刻，生产系统平时要付出什么代价、做哪些演练。
>
> **全文唯一比喻**：延续 mysql-01「24 小时无人图书馆」、mysql-02「便签系统」、mysql-03「总馆-分馆传抄网络」的宇宙——**档案馆-缩微胶片体系**：
>
> - `全量备份` = 档案馆里定期拍摄的"全馆账本缩微胶片"
> - `增量备份` = 两次全量之间的"变更补页"
> - `binlog` = 总馆每天产生的"传抄补页流水"
> - `PITR` = 按补页上的日期和页码，把账本翻回到某一天某一笔之前
> - `恢复演练` = 定期把胶片拿出来重新誊一遍，确认还能读得出来

**承接**：mysql-03 收口了"同一份数据的跨机器存活"，结尾留下一句话：*复制能防"机器没了"，但防不了"人删了/逻辑坏了/全机房没了"*。本文接走这最后一笔账。

---

## 承接上一篇的账

mysql-03 结尾留下的新账：

| # | 上一篇留下的问题 | 本文哪一层接 |
|---|---|---|
| ① | 复制能防机器没了，但防不了"人删了/逻辑坏了/全机房没了" | Level 1 |
| ② | 当所有副本都带着同一个错误往前走，最后的兜底是什么？ | Level 1-5 |
| ③ | 备份 + binlog 时间点恢复怎么做到"精确到事务"？ | Level 5-6 |

**主线进度**：①备份是最后防线 → ②逻辑/物理、全量/增量怎么选 → ③一致性备份怎么拿坐标 → ④增量与 LSN 怎么接续 → ⑤PITR 全链操作 → ⑥GTID 下的恢复边界 → ⑦生产备份体系 → ⑧RPO/RTO 与容灾演进

## 🏷️ 关键词

`mysqldump` `xtrabackup` `全量备份` `增量备份` `逻辑备份` `物理备份` `一致性快照` `FLUSH TABLES WITH READ LOCK` `LSN` `binlog` `mysqlbinlog` `PITR` `GTID` `gtid_purged` `恢复演练` `RPO` `RTO` `异地容灾`

## ⚠️ 版本与证据边界

| 项目 | 声明 |
|---|---|
| **基线版本** | MySQL 5.7.44，与系列 mysql-01/02/03 一致 |
| **对照版本** | MySQL 8.0.x / 8.4 LTS，仅在语义变化处标注 |
| **已查证** | `mysqldump --single-transaction` 对 InnoDB 使用一致性快照，避免长时间全局读锁【官方手册语义】；`xtrabackup` 为 Percona 开源物理备份工具，非 MySQL 官方组件【公开工具文档】；`mysqlbinlog --start-datetime/--stop-datetime/--start-position/--stop-position` 为官方 binlog 处理参数【官方手册语义】 |
| **版本差异** | 8.0.26 起 `mysqldump --master-data` 更名/推荐为 `--source-data`【官方术语变更】；`binlog_expire_logs_seconds` 在 8.0 替代 5.7 的 `expire_logs_days`【官方参数变更】 |
| **默认值** | `expire_logs_days=0`（5.7 默认不自动清理，需显式设置）【官方默认值】；`sync_binlog=1` 双一配置已在 mysql-01/03 覆盖 |
| **日志示例** | 文中备份/恢复命令输出为**示意脱敏简化**：字段结构真实、具体路径/时间/LSN 数值虚构 |
| **数字纪律** | 所有备份耗时、恢复耗时、压缩率、RPO/RTO 数字 `[须实测]` 或 `[教学量级]`；具体选型以环境实测为准 |
| **跨篇去重** | binlog 生成/复制机制回链 mysql-01/03；GTID 基础回链 mysql-03；锁与一致性回链 mysql-02；分库分表后的备份策略回链 12-sharding |

## 🗂️ 目录

- [Level 1：为什么备份是最后一道防线——复制不是备份](#level-1)
- [Level 2：备份分类——逻辑 vs 物理、全量 vs 增量](#level-2)
- [Level 3：一致性备份——快照、锁与 binlog 坐标](#level-3)
- [Level 4：增量备份——LSN 接续与断层风险](#level-4)
- [Level 5：PITR——全量 + binlog 翻到指定时点](#level-5)
- [Level 6：GTID 下的 PITR——恢复后如何接回复制拓扑](#level-6)
- [Level 7：生产备份体系——策略、加密、异地与演练](#level-7)
- [Level 8：RPO/RTO 设计——从备份到容灾的演进路径](#level-8)
- [📋 落地清单](#landing-checklist)
- [🧪 合书自测](#self-test)
- [⚠️ 坑与细节](#pitfalls)
- [📚 勘误表（素材纠错）](#errata)
- [🏆 生产决策卡](#decision-cards)
- [🌍 跨系统迁移](#cross-system)
- [📎 系列承接](#series-next)

<a name="level-1"></a>
## Level 1：为什么备份是最后一道防线——复制不是备份

### 👶 前置知识关卡

- ✅ 理解复制是"同一份数据的多个热副本"（mysql-03）？
- ✅ 理解 binlog 是"传抄记录"（mysql-01/03）？
- ✅ 知道 RPO/RTO 的基本含义？

### Why——徒弟："有主从复制还不够吗？"

**老陈**：复制解决的是"机器没了"。但下面这些场景，复制全都没用：

| 场景 | 为什么复制救不了 |
|---|---|
| 业务 Bug 批量 UPDATE/DELETE | 错误 SQL 会原样复制到所有从库，所有副本一起坏 |
| 误删表/库 | DROP/TRUNCATE 是复制事件，从库同样执行 |
| 黑客或勒索 | 如果连备份一起加密，那才是真灾难 |
| 全机房故障/区域故障 | 同城/同机房多副本可能同时不可用 |
| 逻辑损坏蔓延 | 数据页损坏、错误数据被复制，热副本只会忠实传播 |

复制能提供高可用，但它和备份是**两个维度**：

```
复制：同一份数据，多台机器同时热着。
备份：同一份数据的"历史底片"，平时冷着，必要时重建。

复制解决"正在服务的机器挂了"；
备份解决"所有副本都坏了/都错了"。
```

### What——备份恢复在数据库保证里的位置

```
mysql-01 单机：WAL 保证崩溃恢复
mysql-03 跨机：复制保证热副本存活
mysql-04 兜底：备份 + binlog 保证灾难后可重建到任意时点
```

- WAL 防的是"这台机器崩溃后丢已提交事务"；
- 复制防的是"这台机器没了，另一台顶上"；
- 备份防的是"所有机器都没了，或者所有机器都带着同一个错误"。

### How——RPO/RTO 在备份语境里重新定义

```
RPO：最多丢多少数据 = 故障时点 - 最近可恢复点
    其中"最近可恢复点"取决于：最近可用全量/增量备份 + 备份点到故障点是否有连续 binlog + 恢复链是否验证过
RTO：多久恢复可用 = 全量恢复时间 + 增量/binlog 应用时间 + 应用切换时间
```

> 📌 备份不是"每天导一次 SQL"就完事。它的 RPO 由"最后一次可用备份 + 到故障点的 binlog"共同决定；它的 RTO 由恢复耗时和演练熟练度决定。

**本层留下的新账**：知道备份重要了——但备份有很多种，哪些适合什么场景？→ 逼出 Level 2。

> 🔴 **口诀**：复制防机器没了，备份防所有都坏了；错误 SQL 会复制，备份才是后悔药。

<a name="level-2"></a>
## Level 2：备份分类——逻辑 vs 物理、全量 vs 增量

### 👶 前置知识关卡

- ✅ 知道 InnoDB 数据文件、redo log、binlog 的区别？
- ✅ 知道 mysqldump 是官方逻辑备份工具？
- ✅ 知道 xtrabackup 是物理热备工具？

### Why——徒弟："mysqldump 不是一直够用吗？"

**老陈**：小库够用，大库不够。备份工具选型本质上是在**可移植性、恢复速度、备份影响、增量能力**之间做 trade-off。

### What——两种备份原理

| 维度 | 逻辑备份（mysqldump） | 物理备份（xtrabackup 类） |
|---|---|---|
| 备份内容 | SQL 语句/记录数据 | 数据文件 + redo log 等物理文件 |
| 恢复方式 | 在目标库执行 SQL | 拷贝文件 + prepare/apply log |
| 体积 | 通常更大，文本/压缩后看数据 | 接近数据文件大小，压缩空间有限 |
| 恢复速度 | 慢，逐条执行 SQL | 快，文件级恢复 |
| 跨版本/跨平台 | 相对灵活 | 通常要求同大版本/同平台 |
| 备份影响 | `--single-transaction` 对 InnoDB 影响较小；大表仍耗 IO/CPU | 热备时额外 IO，但对业务侵入相对可控 |
| 增量能力 | 弱，通常靠全量+binlog | 强，支持基于 LSN 的增量备份 |

### 全量 vs 增量

```
全量备份：某一时刻的完整数据快照。
增量备份：从上一次备份以来发生变化的数据页/事务。
binlog：连续的事务日志，可作为“增量到任意时点”的补充。
```

生产常见组合：

```
每周日全量 + 每天增量 + 持续保留 binlog
恢复时：最近全量 → 之后所有增量 → binlog 追到目标时点
```

### How——选型初步判断

| 场景 | 推荐 |
|---|---|
| 小库、跨版本迁移、需要导出部分表 | `mysqldump` 逻辑备份 |
| 大库、要求恢复快、需要增量 | `xtrabackup` 物理备份 |
| 云数据库 | 使用云厂商备份/快照 + binlog 订阅 |
| 分库分表 | 按分片分别备份 + 全局 binlog 策略（回链 12-sharding） |

> 📌 **没有万能工具**。很多团队全量用物理、小表/迁移用逻辑、binlog 全部保留，组合使用。

**本层留下的新账**：工具选好了——但备份出来的数据是"一致"的吗？怎么保证备份点不乱？→ 逼出 Level 3。

> 🔴 **口诀**：逻辑可移植恢复慢，物理体积小恢复快；全量定基调，增量省空间，binlog 补到秒。

<a name="level-3"></a>
## Level 3：一致性备份——快照、锁与 binlog 坐标

### 👶 前置知识关卡

- ✅ 理解 InnoDB 的 MVCC 一致性快照？
- ✅ 知道 binlog 与 InnoDB 事务的提交顺序（mysql-01 Level 7）？
- ✅ 知道 `FLUSH TABLES WITH READ LOCK` 会阻塞写？

### Why——徒弟："备份不就是把文件拷出来吗，为什么要讲一致性？"

**老陈**：因为数据库文件不是一张静态照片。备份过程中事务还在提交，redo、数据文件、binlog 各自在动。如果各文件不在同一个时间点，恢复出来就是一本页码错乱的账。

### What——一致性备份要满足什么

一个可用备份必须能回答：

```
这份备份对应 binlog 的哪个位置？
```

有了这个坐标，后续才能从备份点继续追 binlog 做 PITR。

常见一致性手段：

| 手段 | 适用 | 代价 |
|---|---|---|
| `mysqldump --single-transaction --master-data=2` | InnoDB 为主的小/中库 | 通过 MVCC 快照获得一致性读；不锁 InnoDB 业务写，但 DDL 仍需小心 |
| `FLUSH TABLES WITH READ LOCK` + 物理拷贝 | 混合引擎/需要全局一致点 | 全局只读，阻塞写入，持锁时间越短越好 |
| `xtrabackup` 物理热备 | 大库 InnoDB | 备份期间拷贝数据文件 + redo，最后通过 redo apply 与未完成事务处理得到一致性 |
| 云厂商快照 | 云环境 | 依赖厂商实现，通常配合 binlog 坐标 |

### How——mysqldump 的一致性坐标（分环境示例）

> ⚠️ **勿直接复制 `OFF` 到 GTID 环境**：GTID 环境必须让备份记录 `executed_gtid_set`，否则恢复后无法接回拓扑。下例拆成两类。

```bash
# GTID 环境（默认/推荐）：保持 AUTO/ON，备份文件头会写 GTID_PURGED
# 5.7 基线：--single-transaction 对 InnoDB 做一致性快照
mysqldump \
  --single-transaction \
  --master-data=2 \
  --set-gtid-purged=AUTO \
  --routines --triggers --events \
  -u backup -p \
  mydb > mydb_full.sql
# 文件头可见：
# -- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000123', MASTER_LOG_POS=456;
# SET @@GLOBAL.GTID_PURGED='...:1-500';

# 非 GTID 环境，或明确不需要记录 GTID 的独立备份
mysqldump \
  --single-transaction \
  --master-data=2 \
  --set-gtid-purged=OFF \
  --routines --triggers --events \
  -u backup -p \
  mydb > mydb_full.sql
```

- `--single-transaction` 依赖 InnoDB MVCC，不锁普通 InnoDB 写；
- 但它**不保护 DDL**，备份期间如果有 `ALTER TABLE`，可能导致快照与元数据不一致；
- `--master-data=2` 记录 binlog 坐标，8.0.26 后建议用 `--source-data=2`；
- GTID 环境务必用 `AUTO/ON`；`OFF` 仅适合非 GTID 环境、或"独立分析恢复且明确不接回拓扑"的场景，见 Level 6 分流。

### How——xtrabackup 的一致性恢复

```bash
# 全量备份
xtrabackup --backup --target-dir=/backup/base --host=127.0.0.1

# prepare：应用 redo log，使数据文件达到一致状态
xtrabackup --prepare --target-dir=/backup/base

# 恢复：把 prepare 后的文件拷回数据目录
xtrabackup --copy-back --target-dir=/backup/base
```

`--prepare` 是物理备份的关键：备份时拷贝的数据文件可能处于不同时间点，必须通过 redo log 前滚到一致位点。

**本层留下的新账**：全量备份拿到一致点了——但每天全量太贵，怎么只备份变化？→ 逼出 Level 4。

> 🔴 **口诀**：一致性不是"文件没坏"，而是"数据文件与 binlog 坐标对得上"；逻辑靠快照，物理靠 prepare。

<a name="level-4"></a>
## Level 4：增量备份——LSN 接续与断层风险

### 👶 前置知识关卡

- ✅ 知道 InnoDB 的 LSN（Log Sequence Number）概念？
- ✅ 理解物理备份可以按数据页变化做增量？
- ✅ 知道 binlog 是逻辑增量，和物理增量是两条线？

### Why——徒弟："全量备份太占地方，能不能只备变化？"

**老陈**：能。物理增量备份基于 LSN，逻辑增量则主要靠 binlog。两条线要分清，否则恢复时容易接错。

### What——两条增量线

| 增量类型 | 机制 | 恢复时怎么用 |
|---|---|---|
| 物理增量（xtrabackup --incremental） | 基于 LSN，备份自上次备份以来变化的数据页 | 按顺序叠加到全量之上 |
| 逻辑增量（binlog） | 记录事务级变更 | 在全量/增量恢复后，用 mysqlbinlog 回放到目标时点 |

### How——xtrabackup 增量链

```bash
# 0. 全量
xtrabackup --backup --target-dir=/backup/full

# 1. 第一次增量
xtrabackup --backup --target-dir=/backup/inc1 \
  --incremental-basedir=/backup/full

# 2. 第二次增量
xtrabackup --backup --target-dir=/backup/inc2 \
  --incremental-basedir=/backup/inc1

# 恢复：先 prepare 全量，再按顺序 apply 增量
xtrabackup --prepare --apply-log-only --target-dir=/backup/full
xtrabackup --prepare --apply-log-only --incremental-dir=/backup/inc1 --target-dir=/backup/full
xtrabackup --prepare --apply-log-only --incremental-dir=/backup/inc2 --target-dir=/backup/full
xtrabackup --prepare --target-dir=/backup/full
```

注意：

- **增量必须按顺序 apply**，跳过一个增量 = 断层；
- 每次增量基于上一次备份的 LSN，不能混用不同链路的增量；
- 最后一步 `--prepare` 不带 `--apply-log-only`，用于完成最终 prepare 并处理未提交事务。

### 断层风险

```
全量 T0 → 增量 T1 → 增量 T2 → 故障 T3
如果 T2 丢失，物理增量链断在 T1：
  - 若 binlog 从 T1 到 T3 完整保留，仍可用 T0 + T1 + binlog 恢复到 T3，只是恢复时间变长；
  - 若 binlog 也在 T1 后被 purge，才真正出现无法补齐的断层。
```

> 📌 **增量备份不能替代 binlog 保留**。物理增量负责"少拷数据"，binlog 负责"补到任意秒"，两者配合才完整。

**本层留下的新账**：有了全量+增量+binlog，怎么真正"翻到指定时点"？→ 逼出 Level 5。

> 🔴 **口诀**：物理增量按 LSN 串，逻辑增量靠 binlog；增量断档不要怕，binlog 补不上才是真断层。

<a name="level-5"></a>
## Level 5：PITR——全量 + binlog 翻到指定时点

### 👶 前置知识关卡

- ✅ 知道备份里记录 binlog 坐标？
- ✅ 会用 `mysqlbinlog` 查看/过滤 binlog？
- ✅ 理解恢复是"先还原到备份点，再重放到目标点"？

### Why——徒弟："误删了一张表，直接恢复昨天的全量不就行了？"

**老陈**：如果你能接受丢掉昨天到现在的所有数据，确实可以。但绝大多数业务希望把损失控制在误操作发生前的一瞬间——这就要 PITR（Point-In-Time Recovery）。

### What——PITR 的本质

```
目标 = 恢复到故障/误操作之前的某个时点 T

步骤：
1. 找到最近可用全量备份（假设备份点在 B）
2. 恢复全量备份到临时实例或原实例
3. 从备份点 B 之后的所有 binlog 中，回放到 T
4. T 之后的变更不应用
```

关键不是"把所有 binlog 都执行一遍"，而是**精确停在某个时间/位置/事务之前**。

### How——基本命令链

```bash
# 1. 恢复最近全量（以 mysqldump 为例）
mysql -u root -p < mydb_full.sql

# 2. 查看备份里的 binlog 坐标
grep "CHANGE MASTER TO" mydb_full.sql
# -- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000123', MASTER_LOG_POS=456;

# 3. 用 mysqlbinlog 解析从该坐标开始的 binlog，过滤到目标时间
#    ⚠️ 多个 binlog 文件必须通过同一连接回放，否则可能破坏临时表/事务上下文
#    官方强调：多文件通过多个连接回放会丢失上下文，见 mysqlbinlog 文档
mysqlbinlog \
  --start-position=456 \
  --stop-datetime='2026-08-22 14:30:00' \
  mysql-bin.000123 mysql-bin.000124 ... \
  | mysql -u root -p
# 等价：mysqlbinlog ... > pitr.sql && mysql -u root -p < pitr.sql
# 但推荐管道单连接，避免多连接并发回放

# 4. 精确停点建议
#    --stop-datetime 受执行机器时区影响，且可能停在事务中间的事件边界
#    生产更推荐：先定位误操作事务的 GTID/position，再用
#    --stop-position / --exclude-gtids 精确停在事务边界
```

如果误操作是 `DELETE`，通常要找到**误操作前最后一个事务的位置**：

```bash
# 先缩小时间范围再查看 binlog 内容，定位误操作
mysqlbinlog --base64-output=DECODE-ROWS -vv \
  --start-datetime='2026-08-22 10:00:00' \
  --stop-datetime='2026-08-22 15:00:00' \
  mysql-bin.000124 | grep -n "### DELETE"
```

> 🚨 **恢复前先备份当前坏库**。如果恢复过程中操作失误，至少还有一次重来的机会。

### 恢复顺序与常见错误

```
✅ 正确：
   全量备份 → 增量备份（如有）→ binlog 到目标时点

❌ 错误：
   直接对正在运行的生产库执行恢复 SQL
   不确认 binlog 起始坐标，导致重复/遗漏
   把所有 binlog 全量执行，把误操作也恢复了
```

**本层留下的新账**：普通坐标恢复能做——但 GTID 环境里，恢复后怎么接回主从复制？怎么避免 GTID 冲突？→ 逼出 Level 6。

> 🔴 **口诀**：PITR 不是全量重放，而是"全量打底 + binlog 精确刹车"；先备份坏库，再找坐标，最后停到目标前。

<a name="level-6"></a>
## Level 6：GTID 下的 PITR——恢复后如何接回复制拓扑

### 👶 前置知识关卡

- ✅ 理解 GTID = `source_id:transaction_id`（mysql-03）？
- ✅ 理解 `gtid_executed` 与 `gtid_purged` 的区别？
- ✅ 理解复制拓扑中"所有节点执行的 GTID 集合"要一致？

### Why——徒弟："GTID 环境下恢复完，为什么从库接不回去？"

**老陈**：因为备份/恢复会改变实例的 `gtid_executed`。如果恢复后的实例认为自己已经执行过某些 GTID，而 binlog 重放时又带着相同 GTID，可能被跳过；如果 `gtid_purged` 设置不对，从库又会请求到不存在的日志。

### What——GTID PITR 的两个关键点

```
1. 恢复后要正确设置 gtid_purged = 备份点已执行过的 GTID 集合
2. 重放 binlog 时要决定：
   - 保留原 GTID（适合接回原复制拓扑）
   - 还是 --skip-gtids 生成新 GTID（适合脱离原拓扑的独立恢复）
```

### How——GTID 恢复的两条分流（⚠️ 高危操作前置校验）

> ⚠️ **前置校验（不满足则停止）**：以下两条流程**只适用于"全新、已确认无业务写入、GTID 集合为空"的实例**；执行前必须同时验证：① MySQL 版本（5.7 与 8.0 对 `gtid_purged` 的前置条件不同）② `SELECT @@gtid_executed, @@gtid_purged` 当前值 ③ 备份文件的 GTID 记录方式（`mysqldump --set-gtid-purged` 默认 `AUTO` 会在文件头写 `SET @@GLOBAL.GTID_PURGED`，`xtrabackup` 在 `xtrabackup_binlog_info`/`xtrabackup_info` 记录 `executed_gtid_set`，物理/逻辑备份记录方式不同，不能假设都有）④ 恢复目标是"独立分析"还是"接回原拓扑"。MySQL 官方明确：`gtid_purged` 只能包含 `gtid_executed` 的子集，且仅能在 `gtid_executed` 为空或特定条件下修改，版本间行为有差异，详见[官方 GTID 生命周期文档](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids-lifecycle.html)。

#### 流程 A：独立 PITR 分析实例（不接回原拓扑）

适用于：临时拉起一个"只读分析库"验证误操作前数据，不参与原复制拓扑。

```bash
# 0. 确认实例为空：gtid_executed 为空
SELECT @@GLOBAL.gtid_executed; -- 必须为空集

# 1. 恢复全量备份（逻辑备份示例）
mysql -u root -p < mydb_full.sql
# 若备份文件头已包含 SET @@GLOBAL.GTID_PURGED，则此步已设置，无需手工再设

# 2. 若备份未设置且实例为空，才手工设置（择一）
#    注意：RESET MASTER 会清空本实例所有 binlog，属高风险操作，仅在已确认为空的临时实例上使用
# RESET MASTER;  -- 仅当 @@gtid_executed 非空但确认可丢弃时才考虑，默认不执行
SET GLOBAL gtid_purged = '3E11FA47-71CA-11E1-9E33-C80AA9429562:1-500';

# 3. 回放后续 binlog，生成新 GTID 更省心（与原拓扑隔离）
mysqlbinlog --skip-gtids \
  --start-position=<备份点POS> \
  --stop-datetime='2026-08-22 14:30:00' \
  mysql-bin.000125 ... | mysql -u root -p
# --skip-gtids 会让恢复事务获得新 GTID，与旧拓扑不一致，但分析库无需一致
```

#### 流程 B：恢复后重新加入原复制拓扑（需保留原 GTID）

适用于：重建故障主库或从库，并让其重新与拓扑中的其他节点建立 GTID 复制。

```bash
# 1. 同流程 A 恢复全量，并正确设置 gtid_purged = 备份点已执行集合
#    必须验证：gtid_purged ⊆ gtid_executed，且不与实例已有 GTID 冲突

# 2. 回放后续 binlog，保留原 GTID（不加 --skip-gtids）
mysqlbinlog \
  --start-position=<备份点POS> \
  --stop-datetime='2026-08-22 14:30:00' \
  mysql-bin.000125 ... | mysql -u root -p

# 3. 校验：恢复实例的 gtid_executed 必须"覆盖"拓扑中其他节点的已执行集合
SELECT @@GLOBAL.gtid_executed; -- 与原主库对比

# 4. 以从库身份加入：CHANGE MASTER TO ... MASTER_AUTO_POSITION=1
#    若出现 errant transaction（本实例有而拓扑其他节点没有的 GTID），需先通过
#    注入空事务或重建解决，不可直接忽略
```

> 📌 GTID 让 PITR 的"边界"更清晰，但也要求恢复操作者必须理解 `gtid_executed` / `gtid_purged`。否则恢复出来的库可能"数据看起来对，复制拓扑接不回去"。

### 恢复后接回复制的通用校验

```
1. 确认恢复实例的 gtid_executed 覆盖所需事务，且 gtid_purged 设置已完成
2. 若作为新主库，需确保从库的 gtid_executed 不包含恢复实例缺失的事务
3. 若从库有备份点之后、新主库没有的事务（errant），需先补齐或重新初始化该从库
4. 切勿在已承载业务的实例上执行 RESET MASTER
```

**本层留下的新账**：一次恢复能做了——但生产上备份策略、加密、异地、演练怎么系统化？→ 逼出 Level 7。

> 🔴 **口诀**：GTID 恢复先设 purged，保留原 GTID 好接回；skip-gtids 适合独立恢复，接回拓扑要三思。

<a name="level-7"></a>
## Level 7：生产备份体系——策略、加密、异地与演练

### 👶 前置知识关卡

- ✅ 知道 RPO/RTO 是业务需求？
- ✅ 知道备份不仅要"能备"，还要"能恢复"？
- ✅ 知道安全事件里备份可能被一起加密/删除？

### Why——徒弟："有备份脚本不就行了吗？"

**老陈**：备份脚本只是开始。生产备份体系要回答：

```
备份多久做一次？保留多久？
放在哪里？是否加密？是否防删？
有没有人定期验证？恢复一次要多久？
```

### What——生产备份体系的六个要素

| 要素 | 关键问题 |
|---|---|
| 策略 | 全量周期、增量周期、binlog 保留时长 |
| 存储 | 本地盘、对象存储、异地/离线存储 |
| 安全 | 加密、访问控制、防勒索删除（不可变存储） |
| 验证 | 定期恢复到 staging，校验数据与可用性 |
| 监控 | 备份失败、备份超时、备份大小异常、binlog 保留不足 |
| 演练 | 季度/半年真实恢复演练，记录 RTO |

### How——一个可落地的策略示例

```
全量：每周日凌晨 02:00 物理备份
增量：每天凌晨 02:30 增量备份
binlog：至少保留 7 天，且覆盖"最近一次全量 + 增量"到当前
备份存储：本地快速盘 + 异地对象存储双副本
加密：备份文件加密，密钥独立管理
验证：每周自动恢复到 staging 并跑关键查询
演练：每季度一次全量恢复演练，记录 RTO
```

> 🚨 **binlog 保留时长不能只看天数**。它必须覆盖"最近一次可用的全量/增量备份"到当前时间。如果全量备份失败，而 binlog 只保留 1 天，可能直接失去恢复能力。

### 常见监控项

```sql
-- 查看 binlog 保留情况（5.7）
SHOW BINARY LOGS;

-- 查看过期设置（5.7）
SHOW VARIABLES LIKE 'expire_logs_days';
-- 8.0 使用 binlog_expire_logs_seconds
```

```bash
# 备份脚本必须监控退出码
xtrabackup --backup --target-dir=/backup/$(date +%F) || alert

# 恢复演练要记录关键耗时
time mysql -u root -p < full.sql
time mysql -u root -p < pitr.sql
```

**本层留下的新账**：备份体系有了——但容灾设计里，RPO/RTO 怎么从"能恢复"走向"按业务目标恢复"？→ 逼出 Level 8。

> 🔴 **口诀**：备份三件事：能备、能恢复、能防删；binlog 保留必须覆盖最近可用备份，否则备份链会断。

<a name="level-8"></a>
## Level 8：RPO/RTO 设计——从备份到容灾的演进路径

### 👶 前置知识关卡

- ✅ 理解 RPO/RTO 在复制和备份中的不同含义？
- ✅ 知道"热备"和"冷备"的恢复速度差异？
- ✅ 理解备份不是越高频越好，成本与风险要平衡？

### Why——徒弟："能不能每分钟都全量备份？"

**老陈**：能，但没必要。全量备份的成本（存储、IO、对业务影响）会随频率快速上升。RPO/RTO 设计是**用不同手段组合出目标**，而不是把所有手段都开到最大。

### What——备份频率与 RPO/RTO 的关系

```
全量备份周期：决定"最坏情况下要从多老的底片开始重建"
增量备份周期：决定"要补多少增量"
binlog 保留：决定"能精确恢复到多近的时点"
恢复演练：决定"真正恢复要多久"
```

| 目标 | 常用组合 |
|---|---|
| RPO=小时级，RTO=小时级 | 每天全量 + binlog 保留数天 |
| RPO=分钟级，RTO=小时级 | 每天全量 + 增量 + binlog 持续保留 |
| RPO≈0，RTO=分钟级 | 半同步/复制 + 备份兜底 + 自动化切换（mysql-03 + 本文） |
| 区域级容灾 | 异地备份 + 异地从库/集群 + 恢复预案 |

### How——演进路径

```
阶段1 每天全量           → 能恢复，RPO=1天，RTO=小时~天
阶段2 全量+binlog        → 能 PITR，RPO=分钟级，RTO 仍看全量恢复
阶段3 全量+增量+binlog   → 存储成本下降，RPO 不变，RTO 略降
阶段4 异地备份+加密+演练  → 防机房级故障，RPO/RTO 可预期
阶段5 复制+备份+自动化切换 → 热备兜底 + 冷备兜底，RPO≈0 可用性更高
```

> 📌 **备份和复制不是二选一**。复制负责"常态高可用"，备份负责"最后兜底"。两者叠加才能覆盖"机器故障 + 逻辑损坏 + 区域灾难"。

### 容灾设计中的备份角色

```
主库/从库全挂 → 用异地备份重建
主库被误删数据 → 用全量+binlog PITR
从库延迟/数据不一致 → 用备份重建该从库
版本升级失败 → 用升级前备份回滚
```

### 决策原则

1. **先定业务 RPO/RTO，再反推备份频率和恢复路径**；
2. **备份要能防删防加密**，否则勒索场景下等于没有；
3. **恢复演练的耗时才是真实的 RTO**，不是脚本上的理论值；
4. **binlog 保留长度 = 最近可用备份到当前的最短路径**，不能拍脑袋定 1 天或 7 天。

**本层留下的新账：无——mysql 系列从单机到并发、复制、备份的完整故事闭环。**

```
单机崩溃   → WAL（mysql-01）
并发冲突   → 锁与死锁治理（mysql-02）
机器故障   → 主从复制与高可用（mysql-03）
全部都没了 → 备份恢复与 PITR（mysql-04）
```

> 🔴 **口诀**：RPO 由备份链决定，RTO 由演练决定；复制管热备，备份管冷底，防删防加密才是真兜底。

<a name="landing-checklist"></a>
## 📋 落地清单

```
✅ 必做（搭建备份体系时）：
   1. 确定全量/增量/binlog 保留策略，且 binlog 保留覆盖最近可用备份
   2. 至少一种全量备份（逻辑或物理），并记录 binlog 坐标
   3. 备份文件加密，密钥与备份分离管理
   4. 备份存储至少双副本，其中一份异地/离线
   5. 备份脚本必须监控退出码与产物大小，失败立即告警

📊 上线后：
   6. 每周自动恢复到 staging，跑关键查询验证
   7. 每季度一次真实恢复演练，记录实际 RTO
   8. 定期检查 binlog 保留与 purge 设置，防止备份链断裂
   9. 恢复操作前先对当前坏库再备一份，避免二次事故

🚀 触发式升级：
   10. 业务 RPO/RTO 变严 → 增加增量/缩短全量周期/引入异地备份
   11. 大库恢复太慢 → 从逻辑备份切物理备份，或使用云快照
   12. 出现勒索/安全事件 → 启用不可变存储/离线备份，并演练从零重建
```

<a name="self-test"></a>
## 🧪 合书自测

1. 为什么说"复制不是备份"？至少举两个复制救不了的场景。
2. 逻辑备份和物理备份的核心区别是什么？各自适合什么场景？
3. `mysqldump --single-transaction` 为什么能减少锁？它防不住什么？
4. `--master-data=2` 在备份里记录了什么？为什么 PITR 需要它？
5. xtrabackup 的 `--prepare` 在做什么？为什么物理备份必须做这一步？
6. 增量备份的 LSN 接续如果断了，会有什么后果？
7. PITR 的完整步骤是什么？为什么不能直接全量重放 binlog？
8. 误删数据后，如何定位 binlog 里误操作前的位置？
9. GTID 环境下恢复后，为什么要设置 `gtid_purged`？
10. `--skip-gtids` 什么时候可以用？什么时候不建议用？
11. 为什么 binlog 保留时长不能只看天数？
12. 备份体系里的"验证"和"演练"分别解决什么问题？
13. RPO 和 RTO 分别由什么决定？

<a name="pitfalls"></a>
## ⚠️ 坑与细节

1. **复制不是备份**：错误 SQL、DROP、勒索都会通过复制传播。
2. **`--single-transaction` 不防 DDL**：备份期间 DDL 可能导致备份不一致。
3. **物理备份不 prepare 不能直接用**：直接拷数据文件可能处于不一致状态。
4. **增量备份顺序不能乱**：跳过一个增量 = 数据断层。
5. **binlog 保留太短**：全量失败或恢复点太老时，binlog 已 purge 就补不回来。
6. **恢复前不备份坏库**：一旦恢复操作失误，可能连重来的机会都没有。
7. **GTID 恢复后不设 `gtid_purged`**：新实例复制会请求到不存在的日志，或认为自己少了事务。
8. **PITR 把所有 binlog 全量重放**：会把误操作也恢复进去。
9. **备份文件不加密/不异地**：机房灾难或勒索时等于没有备份。
10. **只备份不验证**：到真正恢复时才发现备份文件损坏/缺页。
11. **用从库备份但不确认延迟**：如果从库落后，备份点可能缺最近事务；要记录从库的复制坐标并评估可接受性。
12. **全量恢复后忘了追增量/binlog**：只恢复到备份点，RPO 会等于备份周期。
13. **8.0 仍用 `expire_logs_days` 思维**：8.0 推荐 `binlog_expire_logs_seconds`，单位不同容易配错。
14. **`mysqldump --master-data` 在 8.0.26+ 已弃用**：新脚本应使用 `--source-data`，旧参数在未来版本可能被移除，老脚本要尽快迁移。
15. **恢复演练不记录耗时**：没有实测 RTO，容灾承诺就是空话。
16. **只备份数据不备份账号/配置/定时任务**：重建实例后可能连应用都连不上。

<a name="errata"></a>
## 📚 勘误表（对常见素材/面试答案的纠错）

| # | 流传说法 | 纠正 | 依据 |
|---|---|---|---|
| 1 | "mysqldump 全库备份会锁表很久" | 对 InnoDB 使用 `--single-transaction` 时基于 MVCC 快照，不会长时间锁 InnoDB 写；但 DDL/非事务表仍需注意 | 官方手册语义 |
| 2 | "xtrabackup 拷完文件就能用" | 必须先 `--prepare` 应用 redo log，得到一致性数据文件 | Percona XtraBackup 文档 |
| 3 | "增量备份可以随便选一个增量恢复" | 必须从全量开始按顺序 apply，链路不能断 | XtraBackup 增量恢复语义 |
| 4 | "PITR 就是把 binlog 全部执行一遍" | 必须从备份坐标开始，并停在目标时点之前，否则误操作也被恢复 | mysqlbinlog 语义 |
| 5 | "GTID 恢复后直接 CHANGE MASTER 就行" | 需要先确认/设置 `gtid_purged`，否则复制请求可能超出 binlog 范围 | GTID 文档语义 |
| 6 | "备份保留 7 天就够了" | 保留时长必须覆盖"最近可用备份到当前"的恢复链，不能只看天数 | 备份链设计 |

<a name="decision-cards"></a>
## 🏆 生产决策卡

### 卡 1 · 备份工具选型

| 场景 | 推荐 |
|---|---|
| 小库、跨版本迁移、按表导出 | `mysqldump` 逻辑备份 |
| 大库、恢复速度优先、需要增量 | `xtrabackup` 物理备份 |
| 云数据库 | 云快照 + binlog 订阅 |
| 分库分表 | 分片分别备份 + 全局 binlog 策略（回链 12-sharding） |

### 卡 2 · 备份频率与 RPO

| 业务容忍 | 推荐策略 |
|---|---|
| RPO 小时级 | 每天全量 + binlog 保留数天 |
| RPO 分钟级 | 每天全量 + 每天增量 + binlog 持续保留 |
| RPO≈0 | 半同步/复制 + 备份兜底 + 自动切换（mysql-03 + 本文） |

### 卡 3 · 恢复前检查

- [ ] 已确认备份文件的 binlog 坐标/GTID 集合
- [ ] 已确认目标时点（时间/位置/GTID）
- [ ] 已对当前坏库再做一份备份
- [ ] 已准备好目标实例/存储空间
- [ ] 已确认 binlog 文件完整且未 purge
- [ ] 恢复后计划做数据校验（行数/抽样/关键业务查询）

### 卡 4 · 生产备份体系成熟度

| 级别 | 特征 |
|---|---|
| L1 有备份 | 有全量脚本，但没验证、没异地、没监控 |
| L2 可恢复 | 有定期恢复验证，能按文档恢复 |
| L3 可预期 | 有 RPO/RTO 目标、增量+binlog、演练记录 |
| L4 防灾难 | 异地/不可变存储、加密、季度演练、可重建整个集群 |

<a name="cross-system"></a>
## 🌍 跨系统迁移

| 系统 | 同构问题 | 可迁移的解法 |
|---|---|---|
| Redis | RDB/AOF 持久化与主从复制 | RDB≈全量，AOF≈binlog；AOF 重放≈PITR |
| Kafka | 日志保留、分区副本 | 副本≈复制；日志保留期≈binlog 保留；消费位点≈恢复坐标 |
| 分布式存储 | 快照、对象版本 | 快照≈全量备份；对象版本/回收站≈防误删 |
| 微服务 | 数据库误操作、发布回滚 | 发布前备份/迁移脚本≈PITR 前置；回滚≈恢复到上一版本 |
| 云原生 | PV 快照、etcd 备份 | 卷快照≈物理备份；etcd snapshot + wal≈全量+日志 |

<a name="series-next"></a>
## 📎 系列承接

> **本篇收口"所有副本都不可用/都错误之后"的最后一层兜底**：从备份工具、一致性、增量、PITR、GTID 到生产备份体系，本质是一句话——**复制让数据活着，备份让数据可以重来**。
>
> **mysql 系列到此闭环**：
>
> ```
> mysql-01：一条 UPDATE 在单机里怎么不丢、不坏、不错
> mysql-02：并发冲突与死锁怎么治理
> mysql-03：同一份数据怎么跨机器存活
> mysql-04：所有副本都坏了之后怎么从备份里重来
> ```
>
> 后续如果继续延伸，可以进入分库分表容量治理（12-sharding）或分布式事务（13-distributed-tx），那里是另一个维度的故事。
