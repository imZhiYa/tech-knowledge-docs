# 🐬 MySQL 主从复制与高可用——一条 UPDATE 的第二次生命

## 写在最前面的承诺

> 这不是复制命令手册，更不是"主从延迟怎么查"的单一技巧。
>
> 本文只追踪一条主线：**一条 UPDATE 在单机 InnoDB 里已经提交成功后，如何作为 binlog 事件跨越网络，在另一台 MySQL 上获得第二次生命**——以及当总馆（主库）失火时，分馆（从库）如何升格、丢多少、怎么不丢、怎么让应用几乎无感。
>
> **全文唯一比喻**：延续 mysql-01「24 小时无人图书馆」与 mysql-02「便签系统」的宇宙——**总馆-分馆传抄网络**：
>
> - `binlog` = 总馆的"传抄记录"（mysql-01 已定义）
> - `dump thread` = 总馆抄写员，把新抄页送给分馆
> - `relay log` = 分馆收发室的中转抄本
> - `SQL thread` = 分馆誊写员，把中转抄本誊进分馆正架
> - `主从切换` = 总馆失火，分馆升格为新总馆，并通知所有读者改道

**承接**：mysql-02 收口了"单机并发代价"的整条因果链，结尾留下一句话：*这一切都假设数据在一台机器上*。本文从这台机器之外接走。

---

## 承接上一篇的账

mysql-02 结尾留下的新账：

| # | 上一篇留下的问题 | 本文哪一层接 |
|---|---|---|
| ① | 数据和算力在一台机器上时，单点故障怎么办？ | Level 1 |
| ② | binlog 这个"传抄记录"如何组织成总馆-分馆网络？ | Level 2-3 |
| ③ | 主库挂掉的瞬间，谁顶上、丢多少、怎么不丢？ | Level 4-7 |

**主线进度**：①单点是风险 → ②复制=让 binlog 有第二份生命 → ③异步有延迟窗口 → ④半同步把窗口逼近 0 → ⑤GTID 让切换不再找坐标 → ⑥并行回放追上延迟 → ⑦高可用选型与切换 → ⑧读写分离与生产治理

## 🏷️ 关键词

`binlog` `relay log` `dump thread` `SQL thread` `异步复制` `半同步复制` `AFTER_SYNC` `AFTER_COMMIT` `RPO` `RTO` `GTID` `AUTO_POSITION` `并行复制` `LOGICAL_CLOCK` `WRITESET` `MGR` `Orchestrator` `MHA` `主从切换` `读写分离` `读己之写`

## ⚠️ 版本与证据边界

| 项目 | 声明 |
|---|---|
| **基线版本** | MySQL 5.7.44，与系列 mysql-01/02 一致 |
| **对照版本** | MySQL 8.0.x / 8.4 LTS，仅在语义变化处标注 |
| **已查证** | `SHOW SLAVE STATUS` 在 8.0.22 起更名/推荐为 `SHOW REPLICA STATUS`【官方术语变更】；MySQL 5.7 起半同步默认 `rpl_semi_sync_master_wait_point=AFTER_SYNC`【官方手册语义】；`gtid_mode` 与 `enforce_gtid_consistency` 为 GTID 开启的硬前提【官方手册语义】 |
| **默认值** | 复制默认异步；`rpl_semi_sync_master_timeout=10000ms`【官方默认值】；`sync_binlog=1` 与 `innodb_flush_log_at_trx_commit=1` 是双一配置基线【官方参数】 |
| **错误码/状态** | `SHOW SLAVE STATUS` 关键字段 `Seconds_Behind_Master`、`Slave_IO_Running`、`Slave_SQL_Running`【官方状态定义】；GTID 相关 `ER_GTID_NEXT_TYPE_UNDEFINED_GROUP` 等不在本文展开 |
| **视图差异** | 5.7 用 `SHOW SLAVE STATUS`、`SHOW MASTER STATUS`；8.0 推荐 `SHOW REPLICA STATUS`、`SHOW BINARY LOG STATUS`；`performance_schema.replication_*` 两代都有但表名/字段有差异 |
| **日志示例** | 文中的复制状态输出为**示意脱敏简化**：字段结构真实、具体坐标/时间数值虚构 |
| **数字纪律** | 所有吞吐/延迟/故障恢复数字 `[须实测]` 或 `[教学量级]`；公开分享中的"秒级切换/零丢失"等说法须与具体架构绑定，不当作通用 benchmark |
| **跨篇去重** | binlog 写入链路、redo/undo、双一配置细节回链 mysql-01 Level 7；锁/死锁治理回链 mysql-02；分库分表解决"多份数据的分片路由"，不替代复制，回链 12-sharding；跨库最终一致性回链 13-distributed-tx |

## 🗂️ 目录

- [Level 1：为什么需要复制——单点不是故障问题，是可用性问题](#level-1)
- [Level 2：异步复制——一条 UPDATE 的第二次生命](#level-2)
- [Level 3：异步的代价——延迟、读己之写与数据丢失窗口](#level-3)
- [Level 4：半同步复制——把窗口逼近 0](#level-4)
- [Level 5：GTID 与主从切换——不再数坐标](#level-5)
- [Level 6：并行复制——让第二次生命跑得快](#level-6)
- [Level 7：高可用架构——MHA/Orchestrator 与 MGR 的抉择](#level-7)
- [Level 8：生产治理——读写分离、监控与演进路径](#level-8)
- [📋 落地清单](#landing-checklist)
- [🧪 合书自测](#self-test)
- [⚠️ 坑与细节](#pitfalls)
- [📚 勘误表（素材纠错）](#errata)
- [🏆 生产决策卡](#decision-cards)
- [🌍 跨系统迁移](#cross-system)
- [📎 系列承接](#series-next)

<a name="level-1"></a>
## Level 1：为什么需要复制——单点不是故障问题，是可用性问题

### 👶 前置知识关卡

- ✅ mysql-01 已理解 binlog 是"传抄记录"，redo 是"草稿纸"？
- ✅ mysql-02 已接受"一台机器上并发正确性可治"？
- ✅ 知道 RPO/RTO 的基本含义：RPO=最多丢多少数据，RTO=最多停多久服务？

### Why——徒弟："数据库不是有 WAL 和备份吗？为什么还要复制？"

**老陈**：WAL 解决的是**这台机器自己的崩溃恢复**——机器没坏，redo 能把它拉起来。备份解决的是**历史灾难**——但恢复点通常是昨天的凌晨，RPO 按小时算。

复制解决的是另一件事：**正在服务的这台机器，下一秒就整体不可用（断电、机房故障、硬件损坏、系统升级）时，另一台机器已经热着，能立刻接替**。

```
单机时代：
  应用 ──► MySQL A ──► 硬盘
                │
                └── 机器宕机 → 服务中断，等维修 → RTO=小时级
                备份在冷柜里 → RPO=小时~天级

复制时代：
  应用 ──► MySQL A（总馆）──binlog──► MySQL B（分馆，热备）
                │                          │
                └── A 宕机                 └── B 升格，应用切到 B
                RPO 取决于复制方式          RTO 取决于切换编排
```

### What——复制与备份、分库分表的边界

| 技术 | 解决什么 | 不解决什么 |
|---|---|---|
| WAL/redo | 单机崩溃后不丢已提交事务 | 机器整体没了的可用性 |
| 备份 + binlog | 误删、机房全毁后的历史恢复 | 秒级接管服务 |
| **主从复制** | **同一份数据的副本热存活** | 容量不够、多份数据路由、跨库一致性 |
| 分库分表 | 数据量/写入量超过单机墙 | 单副本故障时的 HA（仍需副本） |
| 分布式事务 | 跨库/跨服务一致提交 | 单库高可用（仍需复制） |

> 📌 **一句话边界**：复制让"一份数据"有多个热副本；分库分表让"多份数据"分散到多个节点；分布式事务让"多节点上的多份数据"保持业务一致。三者是不同维度，不是互相替代。

### How——RPO/RTO 是需求，不是参数

```
金融支付：   RPO≈0（不丢一笔），RTO 秒~分钟级
电商交易：   RPO 可接受秒级，RTO 分钟级
内部系统：   RPO 分钟级，RTO 半小时级也可以
```

没有绝对的好坏，只有**业务愿意为 RPO/RTO 付多少成本**：半同步、MGR、三副本、专线、切换演练，都是成本。

**本层留下的新账**：知道要复制了——但 binlog 怎么变成另一台机器上的数据？→ 逼出 Level 2。

> 🔴 **口诀**：WAL 救单机，备份救历史，复制救热备；RPO/RTO 是需求不是参数。

<a name="level-2"></a>
## Level 2：异步复制——一条 UPDATE 的第二次生命

### 👶 前置知识关卡

- ✅ binlog 里记录的是"传抄记录"，不是 redo 那种物理页？
- ✅ MySQL 主从需要 server_id 唯一、binlog 开启？
- ✅ 知道复制线程三个角色：主库 dump thread、从库 IO thread、从库 SQL thread？

### Why——徒弟："复制不就是把 binlog 文件拷过去再执行一遍吗？"

**老陈**：大方向对，但机制远比"拷文件"精细：它是一条**实时日志流**，不是周期性文件拷贝；从库收到后也不是立刻执行，而是先收进 relay log，再由单/多线程异步回放。

### What——复制链路

```
总馆（Master）                         分馆（Slave）
┌───────────────────┐               ┌──────────────────────┐
│ 业务事务            │               │  relay log            │
│   → InnoDB commit  │               │   （中转抄本）          │
│   → binlog 顺序写   │               │        ▲              │
│         │          │               │        │ 写            │
│         ▼          │  网络传输       │   IO thread（收发员）   │
│  dump thread       │──────────────►│        │              │
│  （抄写员）          │               │        ▼              │
│  读 binlog event    │               │  SQL thread（誊写员）   │
│  发给从库            │               │   → 在从库重放事务      │
└───────────────────┘               │   → 从库 InnoDB         │
                                    └──────────────────────┘
```

核心事实（⚠️ 以下以 `sync_binlog=1`、`innodb_flush_log_at_trx_commit=1` 双一配置为前提，组提交与 fsync 时序因版本/参数而异）：

1. 主库事务提交涉及 **InnoDB redo、binlog、组提交与 fsync 的协调**，并非"本地提交+binlog"线性两步；异步复制的含义是**提交路径不等待副本 ACK**，不等价于所有提交步骤只有一个简单先后顺序。
2. **IO thread** 从主库拉取 binlog event，**先写 relay log**；**SQL thread** 再读 relay log 应用。两段式让网络抖动与本地回放解耦。
3. **relay log 是从库自己的中转站**，即使主库断网，已拉到的日志仍可继续回放；断网期间新事务继续堆积在主库 binlog，恢复后 IO thread 从断点续传。

### How——启动一个异步复制的最小姿势

```sql
-- 主库：开启 binlog 并设置唯一 server_id
[mysqld]
server_id = 1
log_bin = mysql-bin
binlog_format = ROW          -- 生产基线，RC/GTID/并行复制都更友好
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1

-- 从库：设置不同 server_id
[mysqld]
server_id = 2
relay_log = relay-bin
read_only = ON               -- 防止误写从库
```

```sql
-- 从库执行
CHANGE MASTER TO
  MASTER_HOST='192.168.1.10',
  MASTER_USER='repl',
  MASTER_PASSWORD='***',
  MASTER_LOG_FILE='mysql-bin.000042',
  MASTER_LOG_POS=154;

START SLAVE;

SHOW SLAVE STATUS\G
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Seconds_Behind_Master: 0
```

> 📌 这里 `MASTER_LOG_FILE/MASTER_LOG_POS` 就是**坐标复制**：从哪份传抄记录的哪一页开始续抄。下一代会用 GTID 替代这套坐标，Level 5 接。

### 主库什么时候写 binlog？

回链 mysql-01：事务提交时按 `sync_binlog` 策略落盘。异步复制不改变主库本地提交路径——**主库不会为了等从库而变慢**，这是异步的最大优点，也是 RPO>0 的根源。

**本层留下的新账**：链路通了，但"不等待"带来的延迟和丢失窗口有多大？→ 逼出 Level 3。

> 🔴 **口诀**：主库提交不等从库，IO 收报 SQL 誊写；relay 是中转站，坐标决定续抄点。

<a name="level-3"></a>
## Level 3：异步的代价——延迟、读己之写与数据丢失窗口

### 👶 前置知识关卡

- ✅ 理解 `Seconds_Behind_Master` 是"当前回放到的 binlog 时间与主库写入时间的差"？
- ✅ 知道"主从延迟大"通常不等于 IO 断，而是 SQL 回放慢？
- ✅ 会看 `SHOW SLAVE STATUS` 的坐标字段？

### Why——徒弟："异步这么快，为什么还要半同步？"

**老陈**：因为异步把"我告诉你成功了"和"你那边真的有"彻底分开。这个缝隙平时看不见，一旦主库宕机，就变成真金白银的数据丢失。

### What——两个典型问题

#### 问题 A：延迟（滞后）

```
主库：T0 提交事务 → T1 写 binlog → T2 dump 发出
从库：T3 收到写 relay → T4 回放完成

Seconds_Behind_Master ≈ T4 - T1
```

延迟来源：

| 来源 | 说明 |
|---|---|
| 大事务 | 一条 UPDATE 影响百万行 → binlog 巨大，传输/回放都慢 |
| 从库硬件弱 | 从库承担读流量，回放与查询抢 CPU/IO |
| 串行回放 | 5.6 默认单 SQL 线程，主库并发再高，从库也只能一条条来 |
| 长事务 | 主库还没提交，binlog 还没产生，从库自然追不上 |
| 网络 | 带宽/延迟，尤其跨机房 |

#### 问题 B：读己之写

读写分离后，应用写完主库立刻读从库，可能读不到刚写的数据：

```
用户下单：写主库成功 → 读从库查订单 → 从库还没回放 → “订单不存在” → 用户以为没下单
```

这不是 bug，是**异步复制天然的一致性窗口**。治理不是消灭延迟，而是让路由知道"这条读必须看到刚才的写"。

### How——读己之写与延迟的初步对策

**读己之写**常见手段：

1. **关键读走主库**：下单结果页、支付结果、库存扣减确认等强一致读强制主库；
2. **会话/请求绑定**：写后 N 秒内同一用户的读都路由到主库（或等待从库追上）；
3. **版本号/时间戳校验**：读从库时带上"至少看到哪个 binlog 位点"，未达则路由主库。

**延迟**先定位再优化：

```sql
SHOW SLAVE STATUS\G
-- 重点看：
--   Slave_IO_Running / Slave_SQL_Running
--   Seconds_Behind_Master
--   Master_Log_File / Read_Master_Log_Pos
--   Relay_Master_Log_File / Exec_Master_Log_Pos
```

如果 `Slave_SQL_Running=No`，先看 error log；如果 `Seconds_Behind_Master` 持续增长，优先排查大事务和从库负载；如果长期为 NULL，检查是否 NTP/时钟问题或 SQL 线程停止。

> 🚨 **大事务是主从延迟的第一杀手**。mysql-01 坑 7 说的是大事务让 undo 膨胀、锁持有久；在复制里它还会让 binlog 大、relay 大、回放久——同一颗雷炸两个战场。

### RPO 窗口——为什么异步复制会丢数据

```
时间轴：
T1 主库提交事务X并返回客户端“成功”
T2 主库 binlog 已写，但 dump 还没发出 / 从库还没收到
T3 主库宕机
→ 事务X只存在于主库，从库没有 → 切换后丢失
```

窗口大小 = **主库提交成功 到 从库实际收到并持久化** 的时间差。正常情况下毫秒级；主库瞬时崩溃、网络抖动、从库落后时可能变成秒级甚至更大。

**本层留下的新账**：异步的 RPO 窗口由"不等从库"造成——那如果让主库等从库确认一下呢？→ 逼出 Level 4。

> 🔴 **口诀**：异步快在不等，危在会丢；读己之写靠路由，大事务是延迟元凶。

<a name="level-4"></a>
## Level 4：半同步复制——把窗口逼近 0

### 👶 前置知识关卡

- ✅ 理解"异步 = 主库提交不等待从库"？
- ✅ 理解"ACK = 从库确认收到"？
- ✅ 知道 `rpl_semi_sync_master_wait_point` 有两个值？

### Why——徒弟："能不能让主库等从库收到再告诉客户端成功？"

**老陈**：能，这就是半同步。但它不是简单地加一个等待，关键在于**等在哪一刻**。等错位置，会出现"主库已提交、从库没有、客户端还不知道"的幽灵。

### What——半同步的两种等待点

```
AFTER_SYNC（5.7 默认）：
主库写 binlog 并 sync 到磁盘
  → 发给从库
  → 等从库 ACK
  → 收到 ACK 后 才提交 InnoDB 事务
  → 再返回客户端成功

AFTER_COMMIT：
主库写 binlog 并 sync
  → binlog 事件发给从库
  → 提交 InnoDB 事务（本机可见）
  → 等从库 ACK
  → 再返回客户端成功
```

| 等待点 | 主库 InnoDB 提交时机 | 客户端成功返回时机 | 风险 |
|---|---|---|---|
| AFTER_SYNC | 等 ACK 之后 | ACK 之后 | 主库等 ACK 期间，事务还没提交，其他会话不可见；主库故障时事务不会出现在旧主库 |
| AFTER_COMMIT | 等 ACK 之前 | ACK 之后 | 主库已提交但客户端还没收到成功；若主库在 ACK 前宕机，从库可能没有该事务，旧主库上却已提交——**幻影提交/phantom commit** |

> 📌 5.7 默认 `AFTER_SYNC` 正是为了消除 AFTER_COMMIT 的幻影提交问题。这也是为什么很多老文章讲的"半同步先提交再等 ACK"在新版本里已经不是默认。

### How——半同步怎么开

```ini
[mysqld]
# 主库
plugin-load = "rpl_semi_sync_master=semisync_master.so"
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_master_wait_point = AFTER_SYNC
rpl_semi_sync_master_timeout = 1000      # ms，等不到 ACK 多久后降级异步

# 从库
plugin-load = "rpl_semi_sync_slave=semisync_slave.so"
rpl_semi_sync_slave_enabled = 1
```

```sql
-- 确认状态
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';
SHOW STATUS LIKE 'Rpl_semi_sync_master_clients';
SHOW STATUS LIKE 'Rpl_semi_sync_master_tx_avg_wait_time';
SHOW STATUS LIKE 'Rpl_semi_sync_master_no_tx';
```

- `Rpl_semi_sync_master_status: ON` 表示半同步生效；
- `Rpl_semi_sync_master_no_tx` 表示降级为异步的事务数——**这个指标是 RPO 告警的关键**。

### 半同步的代价与降级

| 代价/风险 | 说明 |
|---|---|
| 提交延迟增加 | 每次提交要等至少一个从库 ACK，跨机房时尤其明显 |
| 可用性下降 | 从库故障/网络分区时，如果一直等，主库也会被拖死 |
| 自动降级 | 超过 `rpl_semi_sync_master_timeout` 未收到 ACK，主库自动退回异步 |
| 降级后 RPO 回到异步窗口 | 这是最大的隐蔽风险：半同步平时 RPO≈0，降级期间 RPO>0 |
| 至少一个从库 ACK ≠ 多数派 | 主库+这个从库同时故障仍可能丢；MGR 的多数派是另一个层级 |

> 🚨 **半同步不是"零丢失保险"**。它是把 RPO 从"总是有窗口"压缩到"平时接近 0、降级时回到异步窗口"。要更强的保证，需要引入多数派/组成员确认（MGR）或外部协调者。

**本层留下的新账**：即使半同步，切换时也要知道从库到底缺哪些事务；靠 `MASTER_LOG_FILE/POS` 手动找断点太容易错。→ 逼出 Level 5。

> 🔴 **口诀**：AFTER_SYNC 先等后提交，AFTER_COMMIT 有幽灵；超时降级异步，RPO 重新开窗。

<a name="level-5"></a>
## Level 5：GTID 与主从切换——不再数坐标

### 👶 前置知识关卡

- ✅ 理解坐标复制 `MASTER_LOG_FILE/POS` 的脆弱点？
- ✅ 知道 `server_uuid:transaction_id` 形式的 GTID？
- ✅ 理解"切换 = 让从库补完缺的事务，然后把它当主库"？

### Why——徒弟："坐标复制不也能断点续传吗？为什么还要 GTID？"

**老陈**：坐标复制的问题不在"能不能续"，而在**切换时怎么确认"哪台从库已经完整"**。你拿着两个文件位置去比对，可能因为中继日志、purge、不同链路而算错。GTID 把"每个事务是谁"变成全局唯一 ID，从库只需回答：**我执行过哪些 GTID**。

### What——GTID 核心概念

```
GTID = source_id:transaction_id
例如：3E11FA47-71CA-11E1-9E33-C80AA9429562:23
```

- 每个事务在原始主库上分配一个 GTID，并随 binlog 传播；
- 从库执行后记入自己的 `gtid_executed`；
- 主库的 `gtid_executed` 表示"本实例已经执行过哪些事务"；
- 复制时使用 `MASTER_AUTO_POSITION=1`，双方交换 `gtid_executed`，从库自动请求主库中"自己缺的那段"。

开启 GTID 的硬前提【官方手册语义】：

```ini
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = mysql-bin
log_slave_updates = ON
server_id = 1
```

- `log_slave_updates`：从库重放事务时也写自己的 binlog，这样从库晋升后能把日志继续传给下一级；
- `enforce_gtid_consistency = ON`：禁止 GTID 不安全的语句（如 `CREATE TABLE ... SELECT` 等）。

### How——GTID 复制与切换

```sql
-- 从库连接主库：不再指定 file/pos
CHANGE MASTER TO
  MASTER_HOST='192.168.1.10',
  MASTER_USER='repl',
  MASTER_PASSWORD='***',
  MASTER_AUTO_POSITION=1;

START SLAVE;
SHOW SLAVE STATUS\G
-- Retrieved_Gtid_Set: 3E11FA47-...:1-100
-- Executed_Gtid_Set:  3E11FA47-...:1-100
```

**主库故障后的选主与补账（⚠️ 旧主必须先 fencing）**：

> ⚠️ **旧主库不能"自动接回"**。官方对 `AFTER_SYNC` 的说明指出：源库故障切换后，旧源库不应直接作为可用源库重新加入，应先隔离或丢弃后重建，见[官方半同步文档](https://dev.mysql.com/doc/refman/8.0/en/replication-options-source.html)。旧主可能存在：已提交但未复制出去的事务、errant transaction、`AFTER_SYNC` 场景下未提交的 binlog、或仍被应用写入造成双写。

```
0. 隔离/围栏旧主库：VIP/代理摘除、置 read_only/super_read_only=ON、防火墙阻断应用写入、确认无新写入
1. 候选从库集合里，选 Executed_Gtid_Set 最全/最接近主库的从库
2. 对候选从库执行 STOP SLAVE;
3. 提升候选从库为新的主库：
   执行 RESET SLAVE ALL 清掉复制元数据（不要 RESET MASTER，
   否则会清掉用于服务其他从库的 binlog 历史）
   SET GLOBAL read_only=OFF;
4. 其余从库 CHANGE MASTER TO ... MASTER_AUTO_POSITION=1
   指向新主库；新主库通过 AUTO_POSITION 补发给它们缺失的 GTID
5. 切换流量到新主库（VIP/DNS/代理）
6. 旧主库：先校验 GTID 集合与是否存在 errant transaction
   → 若无 errant 且确认无未复制事务，可作为从库指向新主库
   → 否则需重建（全量备份重搭）或注入空事务处理后再加入
   AUTO_POSITION 仅能跳过冲突 GTID，不能修复数据分歧本身
```

> 📌 GTID 不是"自动选主"，它只是让"补发哪些事务"这件事变得可计算。选主、VIP 漂移、健康检查、应用切换仍然需要外部编排（Level 7）。

### 切换中的关键检查

```sql
-- 候选从库是否已经追到主库最新？
SELECT WAIT_FOR_EXECUTED_GTID_SET('3E11FA47-...:1-100', 10);
-- 返回值 0 表示达到；非 0 表示超时/未达到

-- 查看实例已执行和已清理的 GTID
SHOW MASTER STATUS;
SHOW GLOBAL VARIABLES LIKE 'gtid_purged';
```

- `gtid_executed` 包含本机执行过、但 binlog 可能已 purge 的事务；
- `gtid_purged` 表示 binlog 里已经没有的旧事务；新从库全量初始化时要先设置 `gtid_purged` 再开始复制。

**本层留下的新账**：切换不再数坐标了，但新主库要扛住流量，回放速度必须跟得上——单线程 SQL thread 够吗？→ 逼出 Level 6。

> 🔴 **口诀**：GTID 让事务有身份证，AUTO_POSITION 自动补缺；切换先比 executed，purged 别忘交代。

<a name="level-6"></a>
## Level 6：并行复制——让第二次生命跑得快

### 👶 前置知识关卡

- ✅ 知道主库并发事务在 binlog 里是"提交序"排列？
- ✅ 理解从库回放必须保证**因果一致**，不能把有依赖的事务乱序执行？
- ✅ 了解 5.6 到 8.0 并行复制参数演进？

### Why——徒弟："主库 1000 TPS，从库单线程回放，不追不上吗？"

**老陈**：对。5.6 之前从库基本单线程回放，主库再并发，从库只能一条条执行——这是主从延迟最大的结构性来源。并行复制的目标不是"乱序快跑"，而是**在保证因果一致的前提下，把互不依赖的事务分给多个线程同时回放**。

### What——三代并行复制

| 阶段 | 并行粒度 | 原理 | 局限性 |
|---|---|---|---|
| 5.6 | 按库并行 | `slave_parallel_type=DATABASE` | 电商单库多表，几乎吃不到并行 |
| 5.7 | 按提交序并行 | `LOGICAL_CLOCK`：同一时刻在主库并发提交（无锁冲突）的事务，可以在从库并行回放 | 主库并发度不够时并行度有限 |
| 8.0 | 按写集并行 | `WRITESET`：通过行写集合判断事务是否操作不同行，把更多无冲突事务并行 | 需要 ROW binlog + 写集提取，内存/CPU 有额外开销 |

### How——参数配置

```ini
# 5.7 基线
[mysqld]
slave_parallel_type = LOGICAL_CLOCK
slave_parallel_workers = 8
binlog_group_commit_sync_delay = 0        # 可微调，增加主库同组提交机会
binlog_group_commit_sync_no_delay_count = 0

# 8.0 进一步
binlog_transaction_dependency_tracking = WRITESET
transaction_write_set_extraction = XXHASH64
```

核心原则：

1. **并行不能破坏因果**：同一行的两个事务，后一个可能依赖前一个的结果，必须串行；
2. **提交序是安全默认**：主库能并发提交，说明它们在锁层面互不阻塞，从库并行回放不会产生与主库不同的结果；
3. **WRITESET 放宽的是"时间相邻"限制**：即使主库不是同一时刻提交，只要写集不重叠，从库也可以并行。

> ⚠️ **外部可见的提交顺序不天然保证**：多线程回放时，事务在从库的**提交顺序**可能与主库不同，出现"commit gap"；需关注 `slave_preserve_commit_order`（5.7）/`replica_preserve_commit_order`（8.0.27+ 术语变更，8.0.27 后行为有变化）是否开启，且 `Seconds_Behind_Master` 不能单独代表真实延迟，需结合 GTID 与业务可见性判断，见[官方复制一致性文档](https://dev.mysql.com/doc/refman/8.0/en/replication-features-transaction-inconsistencies.html)。

### 大事务仍是一票否决

并行复制解决的是"很多小事务并发"的延迟；**一个大事务**仍然会让从库长时间单线程执行。治理手段不是调大 workers，而是：

- 拆分大事务（分批 UPDATE/DELETE，回链 mysql-01 坑 7）；
- 避免一次生成超大 binlog（如大范围 UPDATE）；
- 关注 `SHOW SLAVE STATUS` 的 `SQL_Delay`/`Seconds_Behind_Master` 趋势（但不以单一指标判定延迟，见上）。

**本层留下的新账**：从库追得上了，但主库故障时谁来决定晋升、怎么让应用无感切换？→ 逼出 Level 7。

> 🔴 **口诀**：并行不破坏因果，提交序保底，WRITESET 提上限；大事务面前 workers 也白搭，提交顺序靠 preserve_order。

<a name="level-7"></a>
## Level 7：高可用架构——MHA/Orchestrator 与 MGR 的抉择

### 👶 前置知识关卡

- ✅ 理解 RPO/RTO 与复制方式的对应关系？
- ✅ 知道"选主"和"补日志"是两件事？
- ✅ 理解多数派（quorum）与单点确认的区别？

### Why——徒弟："有了 GTID 和并行复制，切换是不是就自动了？"

**老陈**：GTID 解决了"补哪些日志"，并行复制解决了"回放速度"，但还缺一个**外部大脑**：谁来发现主库挂了？谁在多个从库中选新主？谁把 VIP/域名切过去？谁来处理旧主库回归？这些 GTID 都不管。

### What——两条主流路线

#### 路线 A：半同步 + 外部编排（MHA / Orchestrator）

```
应用 ──► VIP/DNS ──► 主库
                       │ 半同步
                  ┌────┴────┐
                从库A      从库B
                 Orchestrator/MHA 监控、选主、切换
```

- **MHA**：经典开源方案，擅长故障检测、选最新从库、补齐 relay log、快速切换；但本身不提供强一致仲裁，需要配合半同步降低丢数据概率。
- **Orchestrator**：更现代的拓扑管理，能发现复制拓扑、自动/半自动切换、有 Web UI 和 API；生产上常与半同步/GTID 搭配。

优点：架构直观，MySQL 原生复制语义，DBA 可控性强；可跨机房部署；升级/演练灵活。
缺点：切换决策依赖外部组件，存在"脑裂"风险，通常需要 VIP/故障域/仲裁机制约束；RPO 仍取决于半同步是否生效。

#### 路线 B：MGR（MySQL Group Replication）

```
┌─────────────┐
│ Primary     │
└──────┬──────┘
       │ 组通信 / Paxos 类协议
  ┌────┴────┬────┴────┐
 │ Secondary│Secondary│
 └──────────┘─────────┘
 多数派确认提交，成员自动故障检测
```

- **单主模式**：一个 Primary 接受写，其他 Secondary 提供读（也可不提供）；Primary 故障时组内多数派自动选新主。
- **多主模式**：多个节点可写，但需要应用处理写冲突；生产通常更谨慎。
- **关键差异**：事务提交需要**组内多数派确认**（多数成员已接收并对事务顺序达成一致），不是"一个从库 ACK"；但多数派确认**不等价于"所有成员已完成应用"**，也不等价于任何故障模型下绝对零丢失——单成员仍可能存在 applier backlog、流控积压，读不同成员可见性不同，且需结合一致性模式与是否已隔离旧主综合判断。详见[官方 Flow Control 文档](https://dev.mysql.com/doc/refman/8.0/en/group-replication-flow-control.html)。

MGR 优点：内建成员管理、自动故障检测、多数派一致性，减少外部编排脑裂问题。
MGR 缺点：网络分区时可用性受多数派约束（少数派不可写）；成员落后/积压时仍可能出现"多数派已确认但少数成员未应用"的时间窗；对网络质量/延迟敏感；跨机房部署要仔细设计；运维心智和排查工具与传统复制有差异。需补充评估：① 组成员是否仍有 applier backlog ② 单主切换时旧主是否已被隔离 ③ 网络分区是否导致少数派停止写入 ④ 应用实际读的是哪个成员及使用何种 `group_replication_consistency` 模式。

### 决策对照表

| 维度 | 半同步 + MHA/Orchestrator | MGR |
|---|---|---|
| 数据安全 | RPO≈0（半同步未降级时，单从库 ACK） | 多数派已接收并定序，单成员仍可能有 backlog，不等价于绝对零丢失 |
| 自动切换 | 外部组件完成 | 组内自动 |
| 脑裂风险 | 需要 VIP/仲裁设计 | 多数派天然防脑裂 |
| 运维熟悉度 | 高，DBA 可控 | 中，新机制学习成本 |
| 网络要求 | 跨机房可用但延迟敏感 | 更敏感，多数派要求 |
| 灵活拓扑 | 主从树、级联、读写分离自由 | 组内相对规整 |
| 适用 | 大部分业务、传统 MySQL 运维体系 | 对自动化和强一致有要求的内部平台/云数据库 |

> 📌 **没有银弹**。MGR 不是"升级版半同步"，而是用"组成员多数派"换"内建自动化和更一致的仲裁"，但代价是网络/部署约束。很多大厂在传统核心库仍用"半同步 + 自研/Orchestrator 切换"，在中间件/云数据库层用 MGR 类方案。

### 切换演练是架构的一部分

无论哪条路线，都要反复演练：

```
1. 杀掉主库（或模拟网络隔离）
2. 记录：RPO 实际丢多少？RTO 实际多少？
3. 检查：从库是否自动/半自动接管？应用是否自动重连？
4. 检查：旧主库恢复后是否自动作为从库回归？有没有重复事务？
5. 检查：读写分离中间件的拓扑是否更新？
```

> 🚨 **没演练过的高可用不是高可用，是高风险**。

**本层留下的新账**：主从高可用架构选好了——但生产上还有读写分离、监控、容量、降级等一堆事。→ 逼出 Level 8。

> 🔴 **口诀**：GTID 管补账，编排管选主，MGR 管多数派；脑裂靠仲裁，RTO 靠演练。

<a name="level-8"></a>
## Level 8：生产治理——读写分离、监控与演进路径

### 👶 前置知识关卡

- ✅ 理解异步/半同步/GTID/并行复制/高可用选型？
- ✅ 知道应用侧连接池、路由层、配置中心的基本概念？

### What——生产环境的完整视图

```
                  ┌───────────────────────────────┐
                  │           应用层                 │
                  │  写库连接池 / 读库连接池 / 强制主库路由 │
                  └───────────────┬───────────────┘
                                  │
                  ┌───────────────▼───────────────┐
                  │   中间件/路由层（Proxy/数据源）     │
                  │   读写分离、只读流量分发、拓扑感知   │
                  └───────────────┬───────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
   ┌────▼────┐              ┌─────▼─────┐            ┌──────▼─────┐
   │ 主库      │──半同步/异步──►│ 从库A      │            │ 从库B      │
   │ 读写     │              │ 只读+备份  │            │ 只读/容灾   │
   └─────────┘              └───────────┘            └────────────┘
        │                         │                         │
        └──────────── 监控/告警/切换编排/对账 ────────────────┘
```

### ① 读写分离与一致性路由

- **基本策略**：写走主库，读走从库；从库故障自动摘除，不拖垮主库；
- **读己之写**：写后强一致读绑定主库，或用"等待从库追上再读"（Level 3）；
- **事务内读**：开启事务后所有读都走主库，避免一个事务内读到不同副本；
- **延迟敏感读**：订单详情、余额等按业务等级配置路由；
- **从库只读**：`read_only=ON`，必要时 `super_read_only=ON`，防止误写。

### ② 监控指标

```sql
-- 5.7 基线
SHOW MASTER STATUS;
SHOW SLAVE STATUS\G

-- 关键指标：
--   Slave_IO_Running / Slave_SQL_Running
--   Seconds_Behind_Master
--   Retrieved_Gtid_Set / Executed_Gtid_Set
--   Relay_Log_Space
```

| 指标 | 告警建议 |
|---|---|
| `Slave_IO_Running` / `Slave_SQL_Running` 不为 Yes | 立即告警，复制中断 |
| `Seconds_Behind_Master` 持续上升 | 大事务/从库负载/并行复制不足 |
| `Rpl_semi_sync_master_no_tx` 突增 | 半同步降级为异步，RPO 窗口打开 |
| `Rpl_semi_sync_master_status` 变为 OFF | 半同步不可用 |
| 从库磁盘空间（relay log 堆积） | 复制中断/大事务导致磁盘打满 |
| 主库 binlog 磁盘空间 | 从库长时间断连导致 binlog 堆积 |

### ③ 备份与复制的配合

复制不是备份：

- 从库可以用来做备份，但**误删/逻辑损坏会通过复制传播**；
- 必须保留独立于复制链路的全量+binlog 备份（mysql-04 范围）；
- 延迟复制（`CHANGE MASTER TO MASTER_DELAY=N`）可做"慢一点但从库"，用于误操作保护。

### ④ 容量与拓扑演进

```
阶段1：单主单从
       主库承担写+读，从库热备/少量读
阶段2：一主多从
       从库水平扩展读，读写分离；主库仍单点写
阶段3：半同步/GTID/Orchestrator
       RPO 压缩到近 0，切换半自动/自动
阶段4：MGR/三副本/单元化
       组内多数派、容灾级 RPO，跨机房部署
阶段5：分库分表/中间件
       单机写容量不够时，水平拆分（回链 12-sharding）
```

> 📌 **复制解决"热副本"，分库分表解决"容量墙"**。不要用无限加从库来扛写瓶颈——从库只解决读扩展，不解决单机写上限。

### ⑤ 降级预案

```
主库故障：
  自动切换未生效 → 手动提升候选从库 + 切换 VIP/DNS
  半同步降级 → 尽快恢复从库/网络，评估当前 RPO 风险
从库故障：
  读写分离摘除该从库；写不受影响；读能力下降
复制中断：
  先看 IO/SQL 线程状态与 error log，不要盲目 START SLAVE
  大事务卡住 → 等待/优化，必要时限流
```

### 演进路径：五阶段

```
阶段1 单机直连            → 单点，RTO 靠人修，RPO 靠备份
阶段2 异步主从            → 读扩展+热备，RPO 有窗口
阶段3 半同步+GTID         → RPO 近 0，切换可计算
阶段4 并行复制+编排        → 延迟受控，切换自动化
阶段5 MGR/单元化          → 多数派一致，容灾级（跨机房/异地多活故事 → mysql-04 备份兜底）
```

**本层留下的新账：主从复制让"同一份数据"在另一台机器上存活，但万一整条复制链路都断了、所有副本都没了，或者有人误删了数据呢？→ 下一篇 mysql-04《备份恢复与 PITR》接走。**

> 🔴 **口诀**：读走从库防延迟，写后强读走主库；半同步降级要告警；复制不是备份，备份才是最后底线。

<a name="landing-checklist"></a>
## 📋 落地清单

```
✅ 必做（搭建时）：
   1. 每实例 server_id 唯一，开启 log_bin、log_slave_updates
   2. 生产 binlog_format=ROW，双一配置 sync_binlog=1 + innodb_flush_log_at_trx_commit=1
   3. 从库 read_only=ON，必要时 super_read_only=ON
   4. 监控 Slave_IO_Running / Slave_SQL_Running / Seconds_Behind_Master
   5. 评估是否开启半同步（至少核心链路），并监控降级计数
   6. 建立独立备份链路，不能只依赖从库

📊 上线后：
   7. 读写分离路由：事务内读主库、写后强一致读主库
   8. 大事务治理：拆分大 UPDATE/DELETE，监控 binlog 大小与延迟
   9. 每季度做一次主从切换演练，记录真实 RPO/RTO
   10. 定期核对主从数据一致性（如 pt-table-checksum 类工具）

🚀 触发式升级（监控信号驱动）：
   11. RPO 要求变严 / 半同步降级频繁 → 完善半同步 + 外部编排，或评估 MGR
   12. 从库延迟成为瓶颈 → 并行复制调优（LOGICAL_CLOCK / WRITESET）
   13. 读流量持续上涨 → 增加从库/读写分离，但写容量墙要靠分库分表
```

<a name="self-test"></a>
## 🧪 合书自测

1. 用一句话区分 WAL、备份、复制、分库分表各自解决的问题。
2. 画出 binlog → dump thread → 网络 → relay log → SQL thread 的数据流。
3. 异步复制为什么 RPO>0？RPO 窗口从哪个时刻到哪个时刻？
4. `Seconds_Behind_Master` 为 NULL 可能意味着什么？
5. 读写分离中"读己之写"怎么解决？举三种手段。
6. 半同步 AFTER_SYNC 和 AFTER_COMMIT 的区别是什么？为什么 5.7 默认 AFTER_SYNC？
7. 半同步超时降级为异步后，哪个状态变量会暴露风险？
8. GTID 相比坐标复制，解决的核心问题是什么？
9. `MASTER_AUTO_POSITION=1` 时从库如何知道要请求哪些 binlog？
10. 并行复制为什么不能简单地把所有事务并行回放？三种并行粒度的演进是什么？
11. MGR 与"半同步+Orchestrator"的核心差异是什么？脑裂分别怎么防？
12. 为什么说"复制不是备份"？至少说出两个理由。

<a name="pitfalls"></a>
## ⚠️ 坑与细节

1. **"从库追不上就加并行线程"不是万能**：大事务、无主键表、从库读负载高都会让并行失效。
2. **`Seconds_Behind_Master` 不是精确延迟**：它基于时间戳估算，时钟不一致/大事务/长事务会失真。
3. **半同步不是绝对零丢失**：超时降级、ACK 后从库未持久化、主从同时宕机都可能丢。
4. **AFTER_COMMIT 的幻影提交**：老文章常把半同步描述成"先提交再等 ACK"，那是旧等待点；5.7 默认 AFTER_SYNC 已经改变语义。
5. **GTID 开启后 `CREATE TABLE ... SELECT` 默认被限制**：`enforce_gtid_consistency=ON` 下要用先建表再 INSERT 的方式。
6. **`log_slave_updates=OFF` 会导致从库晋升后无法把历史事务传给下一级**：级联/切换会断链。
7. **`read_only=ON` 挡不住超级账号**：应用账号不能是超级账号，否则误写从库照样发生。
8. **复制中断后盲目 `START SLAVE`**：不先看 error log，可能反复在同一个错误上卡死。
9. **从库 binlog/relay 无限增长**：长时间断连或大事务堆积会打满磁盘。
10. **读写分离把事务内读路由到从库**：同一个事务里读到主库和从库不同状态，等于破坏事务一致性。
11. **主库 binlog 被 purge 后，掉队从库无法追上**：需要重新全量初始化，不能用 AUTO_POSITION 硬补。
12. **MGR 跨机房部署只加节点不看延迟**：多数派确认会让每个事务等最慢的多数派成员，延迟放大。
13. **把复制延迟当成主库性能问题**：先分清楚是 IO/SQL 线程断、大事务、从库负载还是网络。
14. **没有演练过的自动切换**：第一次真故障时大概率发现 VIP、DNS、连接池、监控拓扑都没准备好。

<a name="errata"></a>
## 📚 勘误表（对常见素材/面试答案的纠错）

| # | 流传说法 | 纠正 | 依据 |
|---|---|---|---|
| 1 | "半同步就是主库等从库执行完才返回" | 半同步通常只等**从库收到并写入 relay log 的 ACK**，不等 SQL 线程回放完成 | 官方半同步复制语义 |
| 2 | "半同步 RPO=0，所以一定不丢" | 半同步有超时降级、ACK 持久化、双故障等边界，RPO 接近 0 而非绝对 0 | 官方降级机制 + 分布式系统常识 |
| 3 | "MGR 是 MySQL 8.0 才有的" | Group Replication 在 5.7.17 作为插件引入，8.0 持续演进 | MySQL 5.7 Release Notes / 官方文档【已查证】 |
| 4 | "`SHOW SLAVE STATUS` 永远这么写" | 8.0.22 起推荐/更名 `SHOW REPLICA STATUS`，5.7 仍是 `SHOW SLAVE STATUS` | 官方术语变更 |
| 5 | "从库追不上就升级从库硬件" | 先看是不是大事务/串行回放/读负载，硬件不是第一解 | 延迟成因分析 |
| 6 | "GTID 能自动切换" | GTID 只解决事务标识与补发，选主/切换仍需外部编排或 MGR | GTID 文档语义 |

<a name="decision-cards"></a>
## 🏆 生产决策卡

### 卡 1 · RPO/RTO 目标 → 复制选型

| 需求 | 推荐方案 | 必须配套 |
|---|---|---|
| RPO 分钟级、RTO 小时级 | 异步复制 + 备份 | 监控延迟与磁盘 |
| RPO 秒级、RTO 分钟级 | 异步/半同步 + 读写分离 | 大事务治理、切换演练 |
| RPO≈0、RTO 秒~分钟级 | 半同步(AFTER_SYNC) + GTID + Orchestrator/MHA | 降级监控、VIP/域名切换 |
| RPO≈0、自动容灾、避免外部脑裂 | MGR（单主） | 网络延迟评估、多数派容量 |

### 卡 2 · 并行复制调优

| 现状 | 动作 |
|---|---|
| 5.7 多库业务 | `slave_parallel_type=DATABASE` 不一定有用；换 `LOGICAL_CLOCK` |
| 主库并发提交高、从库仍慢 | 调大 `slave_parallel_workers`，检查大事务 |
| 8.0 且 ROW binlog | 评估 `WRITESET` + `transaction_write_set_extraction=XXHASH64` |
| 有大事务 | 先拆分大事务，再谈并行参数 |

### 卡 3 · 读写分离路由

| 场景 | 路由 |
|---|---|
| 普通查询、报表、非实时列表 | 从库 |
| 事务内任何读 | 主库 |
| 写后立即读（订单结果、支付状态） | 主库，或等待从库追上 |
| 从库故障 | 摘除从库，降级到主库/其他从库 |

### 卡 4 · 切换预案检查

- [ ] 候选从库的 `Executed_Gtid_Set` 最新且可验证
- [ ] 半同步状态 ON，`no_tx` 低
- [ ] VIP/DNS 切换脚本经过演练
- [ ] 应用连接池支持快速重连/拓扑感知
- [ ] 旧主库回归流程已定义（先只读、补数据、再恢复角色）
- [ ] 每次演练记录真实 RPO/RTO

<a name="cross-system"></a>
## 🌍 跨系统迁移

| 系统 | 同构问题 | 可迁移的解法 |
|---|---|---|
| Redis 主从复制 | 异步复制有丢数据窗口、从库延迟 | 半同步/WAIT 命令、读写分离路由、哨兵/Cluster 自动故障转移 |
| Kafka 分区副本 | 副本不同步、ISR、acks 语义 | `acks=all` ≈ 多数派/半同步确认；ISR ≈ 从库健康集合 |
| 分布式存储（etcd/Raft） | 单点、多数派、脑裂 | 多数派确认、quorum、自动选主；与 MySQL 半同步对比理解一致性与可用性取舍 |
| 微服务多实例 | 实例故障切换、流量摘除 | 健康检查、服务发现、优雅上下线 ≈ 主从拓扑管理 |
| 分布式事务 | 多节点一致性、补偿 | 复制解决副本一致性；跨库事务一致性回链 13-distributed-tx |

<a name="series-next"></a>
## 📎 系列承接

> **本篇收口"同一份数据的跨机器存活"**：从异步复制到半同步、GTID、并行复制，再到 MHA/Orchestrator 与 MGR——本质是一句话：**用日志驱动副本，用确认压缩窗口，用编排接管故障**。
>
> **下一篇（mysql-04《备份恢复与 PITR》）承接的问题**：复制能防"机器没了"，但防不了"人删了/逻辑坏了/全机房没了"。当所有副本都带着同一个错误往前走时，最后的兜底是**备份 + binlog 时间点恢复**。
>
> mysql-01 开头承诺的六层保证中，"可追溯"由复制承接，"可溯源"将由备份恢复正式闭环。
