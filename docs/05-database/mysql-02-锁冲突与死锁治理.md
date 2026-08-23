# 🐬 MySQL 锁冲突与死锁治理——两个事务的死亡缠绕

## 写在最前面的承诺

> 这不是一份死锁面试八股合集，更不是"四个必要条件"的背诵卡片。
>
> 本文只追踪一个主线：一笔**秒杀扣减事务 T1**（下单扣库存）与它的**镜像对手 T2**（取消回滚补库存），从各自的第一条 UPDATE 开始，到其中一个被 InnoDB 强制回滚为止——以及这场相遇之后，从 SQL 层、应用层、架构层一路治到库存中心的终局形态。
>
> **全文唯一比喻**：mysql-01「24 小时无人图书馆」宇宙的延续——**便签系统**。两位读者各持一本书的"正在使用"便签，同时伸手要对方手里的书；巡场管理员发现循环等待，强制收回"代价最小"的一本。
>
> **承接**：mysql-01 Level 5《死锁检测与处理》只回答了"InnoDB 怎么处理死锁"，结尾留下三笔账。本文逐笔接走。

---

## 承接上一篇的账

mysql-01 Level 5 结尾留下的新账：

| # | 没回答的问题 | 本文哪一层接 |
|---|---|---|
| ① | 死锁检测机制懂了，为什么电商扣库存场景格外频繁死锁？ | Level 2 |
| ② | 线上怎么把环抓出来、定位到具体两条 SQL？ | Level 3 |
| ③ | 定位之后，SQL 层 / 应用层 / 架构层分别怎么治？治到什么程度算到头？ | Level 4-8 |

**主线进度**：①并发必然产生锁等待 → ②等待成环=死锁 → ③三大成因在扣库存场景聚集 → ④监控定位 → ⑤三层治理 → ⑥场景适配（什么病开什么药）→ ⑦大厂库存中心终局形态 → ⑧决策卡收口

## 🏷️ 关键词

`Deadlock` `wait-for graph` `两阶段锁协议` `Gap Lock` `Insert Intention Lock` `RR vs RC` `条件更新` `重试幂等` `分布式锁` `Redis 预扣` `热点拆行` `队列串行化` `场景适配` `库存中心` `对账`

## ⚠️ 版本与证据边界

| 项目 | 声明 |
|---|---|
| **基线版本** | MySQL 5.7.44，与系列 mysql-01 一致 |
| **对照版本** | MySQL 8.0.x，仅在语义变化处标注 |
| **已查证** | `innodb_deadlock_detect` 为 **MySQL 5.7.15 新增**【MySQL 5.7 Release Notes, news-5-7-15】；victim 选择=InnoDB 尝试回滚"较小"的事务，大小按插入/更新/删除的行数衡量【Reference Manual: Deadlocks】；`Innodb_deadlocks` 为官方 status counter（累计死锁次数）【官方 Server Status Variables 文档】 |
| **默认值** | `innodb_lock_wait_timeout=50s`、`innodb_deadlock_detect=ON`【官方默认值，前者 mysql-01 已载】 |
| **错误码** | 1213=`ER_LOCK_DEADLOCK`、1205=`ER_LOCK_WAIT_TIMEOUT`【官方错误码定义】 |
| **视图差异** | `INNODB_LOCKS`/`INNODB_LOCK_WAITS` 在 8.0 移除 → `performance_schema.data_locks`/`data_lock_waits`；`sys.innodb_lock_waits` 两代都有（8.0 基于 performance_schema 重写）——差异表 mysql-01 Level 5 已载，本文回链不重展开 |
| **日志示例** | 文中死锁日志为**示意脱敏简化**：字段结构真实、具体数值虚构 |
| **数字纪律** | 所有吞吐/延迟数字 `[须实测]` 或 `[教学量级]`；公开技术分享中的大促峰值等量级说法同样不作 benchmark 依据；"1000 并发→100 并发"类换算为**教学示意** |
| **跨篇去重** | 锁类型/兼容矩阵/Next-Key Lock 细节回链 mysql-01 Level 5；MVCC 快照读回链 mysql-01 Level 6；binlog 两阶段提交回链 mysql-01 Level 7；Seata/全局锁回链 13-distributed-tx；分库分表回链 12-sharding |

## 🗂️ 目录

- [Level 1：死锁为什么必然存在——预防为什么贵过检测](#level-1)
- [Level 2：三大成因——为什么偏偏是扣库存场景](#level-2)
- [Level 3：监控定位体系——把环抓出来](#level-3)
- [Level 4：SQL/事务层治理——破坏循环等待](#level-4)
- [Level 5：应用层治理——消除与消化冲突](#level-5)
- [Level 6：架构层治理——根治热点行](#level-6)
- [Level 7：场景适配——秒杀和普通高并发是两种病](#level-7)
- [Level 8：大厂库存中心——从治死锁到让竞争不发生](#level-8)
- [📋 落地清单](#landing-checklist)
- [🧪 合书自测](#self-test)
- [⚠️ 坑与细节](#pitfalls)
- [📚 勘误表（素材纠错）](#errata)
- [🏆 生产决策卡](#decision-cards)
- [🌍 跨系统迁移](#cross-system)

<a name="level-1"></a>
## Level 1：死锁为什么必然存在——预防为什么贵过检测

### 👶 前置知识关卡

- ✅ mysql-01 Level 5 的 S/X/IX/IS 兼容矩阵？
- ✅ InnoDB 采用两阶段锁协议：语句执行中逐步获取锁，**commit 时统一释放**【手册语义】？
- ✅ 图论常识：有向图出现环意味着什么？

### Why——徒弟："既然四条件背得出，让数据库直接禁止死锁不就行了？"

**老陈**：好想法。预防死锁 = 破坏四个必要条件之一。我们逐条推演它在数据库里的真实代价：

| 预防方案 | 破坏哪个条件 | 具体做法 | 在数据库里贵在哪 |
|---|---|---|---|
| 资源全局排序 | 循环等待 | 所有事务按同一全序申请锁 | 必须**执行前**预知全部加锁集合——但 WHERE 条件运行时才展开、优化器可能换索引、批量语句扫哪些行取决于数据分布。预知未来做不到 |
| 一次性拿齐所有锁再干活 | 持有并等待 | 事务开始先拿齐全部锁 | 同样要预知集合；且持锁时间被拉长到整个事务时长，并发度崩塌 `[教学量级]` |
| 允许抢占剥夺他人锁 | 不可剥夺 | 强制收回 | 本质就是"回滚别人"——那正是检测后 victim 回滚做的事，还失去了"挑代价最小的回滚"的选择权 |
| 什么都不做 | —— | 鸵鸟 | 靠 `innodb_lock_wait_timeout`（默认 50s）兜底，高并发下等待雪崩式堆积 |

> 📌 **结论**：预防的前提是**提前知道未来**，而数据库事务的加锁路径是运行时才展开的。四个条件在数据库里不可能被完全消除——互斥是数据一致性的根基。所以工程最优解不是阻止环形成，而是操作系统理论里的第三条路：**放任环出现 → wait-for 图实时检测 → 发现环立刻回滚代价最小的一方 → 应用层重试**。这就是 InnoDB 默认 `innodb_deadlock_detect=ON` 的原因。
>
> 💡 **面试加分点**：主动指出"InnoDB 本质是**检测恢复策略**而非纯预防"，比只背解决方案清单高一个层次。

### What——四个理论条件在 InnoDB 里的对应物

| 理论条件 | InnoDB 对应物 |
|---|---|
| **互斥** | X 记录锁排他（兼容矩阵见 mysql-01 Level 5） |
| **持有并等待** | 两阶段锁协议：一条 UPDATE 扫描中逐行加锁，全部持有到 commit |
| **不可剥夺** | 行锁没有强制收回机制，只能等事务自己 commit/rollback |
| **循环等待** | wait-for graph 中出现环 |

```
wait-for graph（等待图）：

        等待 orders#5001 的 X 锁
T1 ────────────────────────────▶ T2
 ▲                                │
 │        等待 inventory#1001 的 X 锁 │
 └────────────────────────────────┘

节点 = 事务；边 = "T1 正等待 T2 持有的锁"
边构成环 = 死锁。每次加锁进入等待时触发一次环检测。
```

**victim 怎么选**【官方手册语义】：InnoDB 尝试回滚"较小"的事务——大小按该事务**插入/更新/删除的行数**衡量。所以别指望"总是回滚后提交的那个"。

### How——最小死锁现场

```sql
-- 表：inventory(id PK), orders(id PK)

-- T1（下单）：先扣库存
BEGIN;
UPDATE inventory SET stock = stock - 1 WHERE id = 1001;   -- 持有 inventory#1001 X 锁

-- T2（取消回滚）：先改订单
BEGIN;
UPDATE orders SET status = 'CANCELLED' WHERE id = 5001;   -- 持有 orders#5001 X 锁

-- T1 继续改订单
UPDATE orders SET status = 'PAID' WHERE id = 5001;        -- 等 T2 ……
-- T2 继续补库存
UPDATE inventory SET stock = stock + 1 WHERE id = 1001;   -- 等 T1 → 环！
```

其中一方立即收到：

```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

注意：1213 是**秒级报错**（检测到环即返回），而 1205 lock wait timeout 要等满 50s 默认超时——这是两种失败的本质区别。

### Transfer

预防贵/检测准/超时笨的三角权衡，在 JVM synchronized 嵌套、分布式锁获取顺序、操作系统资源分配中完全同构（见文末跨系统迁移节）。

**本层留下的新账**：机制懂了——但同样的锁系统跑了几十年，为什么偏偏电商扣库存场景天天死锁？→ 逼出 Level 2。

> 🔴 **口诀**：预防要先知未来，数据库做不到；放任成环立刻检，回滚小的最划算。

<a name="level-2"></a>
## Level 2：三大成因——为什么偏偏是扣库存场景

### 👶 前置知识关卡

- ✅ Record Lock / Gap Lock / Next-Key Lock 的定义（mysql-01 Level 5）？
- ✅ 插入意向锁（Insert Intention Lock）：INSERT 前设置的等待信号，彼此互不阻塞，但会等待他人的 Gap Lock【手册语义】？
- ✅ EXPLAIN 各列含义？

### Why——徒弟："随机撞车而已吧，运气问题。"

**老陈**："不是运气。扣库存场景有三个**结构性特征**：跨表操作（订单+库存）、批量操作（多 SKU）、高并发镜像流（下单 vs 取消/回滚）。每个特征都精确命中一类死锁成因。"

### Case A：加锁顺序不一致——镜像流天然反序

```sql
-- T1 下单：扣减多个 SKU
UPDATE inventory SET stock = stock - n
WHERE sku_id IN (1001, 1002, 1003);

-- T2 回滚：补回多个 SKU（业务代码里列表顺序来自订单明细，恰好相反）
UPDATE inventory SET stock = stock + n
WHERE sku_id IN (1003, 1002, 1001);
```

关键认知：**加锁顺序由索引扫描顺序决定，不由 IN 列表的书写顺序决定** `[机制推导]`。单表同索引下两者通常一致（都按索引升序），真正的风险在：

1. 两个事务走了**不同的索引路径**（一个走主键、一个走二级索引再回表），交叉访问顺序不同；
2. **跨表**镜像流：T1 先库存后订单、T2 先订单后库存——这正是 Level 1 最小现场的来源，也是下单/取消业务里最真实的形态。

```
T1: 库存#1001 ──▶ 订单#5001
T2: 订单#5001 ──▶ 库存#1001     ← 两条路径反向交叉 = 环
```

### Case B：RR 下的间隙锁 × 插入意向锁——"锁不存在的行"

这是最反直觉的一类：**两条 SQL 操作的行根本不存在，照样死锁**。

前置事实【手册语义】：Gap Lock 之间**兼容共存**（两个事务可以同时持有同一间隙的间隙锁）；插入意向锁彼此不阻塞；但**插入意向锁要等待持有该间隙 Gap Lock 的所有事务释放**。插入**不同位置**的插入意向锁本身可以并存，插入**同一索引位置**（如同一唯一键）时，还涉及重复键检查与记录锁冲突，见[官方锁设置文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)。

```sql
-- 表 t(id PK/唯一键)，现有 id=20、id=30，隔离级别 RR

-- T1：查询不存在的 id=25，加间隙锁 (20,30)
SELECT * FROM t WHERE id = 25 FOR UPDATE;

-- T2：查询同一个不存在的 id=25，间隙锁兼容 → 也拿到 (20,30) ✓
SELECT * FROM t WHERE id = 25 FOR UPDATE;

-- T1：往这个间隙插同一键 → 插入意向锁被 T2 持有的间隙锁挡住，进入等待
INSERT INTO t VALUES (25);

-- T2：也往这个间隙插同一键 → 插入意向锁被 T1 持有的间隙锁挡住
-- 且同一唯一键还涉及重复键检查/记录锁冲突
INSERT INTO t VALUES (25);      -- → 环！死锁，一方被回滚
```

> 📌 **精确说明**：环的形成是"Gap Lock 共存 → 两事务对同一键的插入意向/记录锁互相等待"，而非"两个插入意向锁互相等待"——插入不同位置的插入意向锁本身不互斥。

对应到业务：`BETWEEN` 条件的 UPDATE/DELETE、对不存在记录的 `SELECT ... FOR UPDATE` 后再 INSERT（典型的"查不到就插入"模式），都是高危动作。

> 🔗 **呼应 mysql-01 坑表**：`REPLACE INTO` 的 DELETE+INSERT 语义在 RR 下同样放大间隙锁范围、显著增加死锁概率——mysql-01 Level 5 已给过对照表，此处不再展开。

### Case C：WHERE 无索引——锁的不是"那行"，是"扫过的每一行"

```sql
-- order_no 上没有索引
UPDATE inventory SET status = 'LOCKED' WHERE order_no = 'O20260822001';
```

**真相纠正**：这不是"表锁"。InnoDB 依然是**行锁**，但因为无法用索引定位，存储引擎全表扫描，**扫过的每一行都被加 X 锁**，且在 RR 下这些锁保持到事务结束【机制语义】。效果近似锁全表，机制完全是行锁。

用 EXPLAIN 初步估算扫描范围的方法（⚠️ 仅估算）：

```sql
EXPLAIN UPDATE inventory SET status='LOCKED' WHERE order_no='O20260822001';
-- type=ALL, rows 为优化器估算值
-- 仅用于判断"是否走索引/扫描范围大致多大"，不是运行时实际加锁行数
-- 实际锁范围需结合执行计划、索引结构（二级索引→聚簇索引回表）、隔离级别
-- 与 SHOW ENGINE INNODB STATUS 的锁信息交叉验证
```

### 结构性聚集：三个特征 × 三类成因

| 扣库存场景特征 | 命中的成因 | 放大系数来源 |
|---|---|---|
| 跨表操作（订单+库存） | Case A 反序 | 下单/取消是天然镜像流 |
| 批量多 SKU | Case A 交叉 | SKU 越多，路径交叉概率越高 |
| 范围条件 + "不存在就插入" | Case B 间隙锁 | 补货、新品上架高频触发 |
| 历史遗留 SQL 无索引 | Case C 放大 | 一条烂 SQL 殃及全表 |

### 普通高并发下单的根因概率排序

线上排查时，按这个顺序逐个核对，命中率从高到低：

| 排序 | 根因 | 典型形态 | 频率 |
|---|---|---|---|
| ① | 购物车结算：一个事务扣多个 SKU，列表没排序 | Case A 批量交叉 | 最常见 |
| ② | 下单事务跨多表（订单/库存/券/积分），各处顺序不一致 | Case A 跨表反序 | 很常见 |
| ③ | 事务里有 RPC 调用 / 发消息，持锁时间被拉长 | 持锁窗口放大，撞车概率上升 | 很常见 |
| ④ | RR 隔离级别下范围查询产生间隙锁冲突 | Case B | 偶发 |
| ⑤ | UPDATE 条件没走索引，锁范围扩大 | Case C | 上线前就该拦住 |

**这张表本身就是诊断工具**：死锁告警来了，从上往下逐个核对，九成案例落在前三行。

**本层留下的新账**：成因清楚了——但这些锁在线上**怎么看见**？总不能等用户投诉才知道吧？→ 逼出 Level 3。

> 🔴 **口诀**：反序易环、空隙也咬人、无索锁全表；排查先查排序和事务。

<a name="level-3"></a>
## Level 3：监控定位体系——把环抓出来

### 👶 前置知识关卡

- ✅ 会连 MySQL 命令行、能看 error log 路径？
- ✅ 知道应用异常日志去哪捞（如 Java 的 `DeadlockLoserDataAccessException`）？

### What——三层观测面

```
┌────────────────────────────────────────────────────┐
│ ① 事后证据链：SHOW ENGINE INNODB STATUS             │
│    └─ LATEST DETECTED DEADLOCK 段（最近一次）       │
│    └─ innodb_print_all_deadlocks=ON → error log    │
│    └─ pt-deadlock-logger 持续采集归档               │
├────────────────────────────────────────────────────┤
│ ② 实时快照：正在发生的锁等待                         │
│    5.7: information_schema.INNODB_TRX / LOCK_WAITS │
│    8.0: performance_schema.data_locks/_waits       │
│         sys.innodb_lock_waits（两代通用入口）        │
├────────────────────────────────────────────────────┤
│ ③ 应用侧指标：错误码计数                             │
│    1213 死锁（可重试）/ 1205 超时（说明长阻塞）      │
└────────────────────────────────────────────────────┘
```

### How——读懂一次死锁日志

```sql
SHOW ENGINE INNODB STATUS\G
-- 重点看 LATEST DETECTED DEADLOCK 段
```

以下为**示意脱敏简化样例**（字段结构真实、数值虚构）：

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
*** (1) TRANSACTION:
TRANSACTION 42198, ACTIVE 0 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
INSERT INTO t VALUES (25)
*** (1) HOLDS THE LOCK(S):          ← 它手里拿着什么
RECORD LOCKS space id 58 page no 3 n bits 72 index PRIMARY
lock_mode X locks gap                ← 持有的间隙锁
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS ... insert intention waiting   ← 插入意向锁在等
*** (2) TRANSACTION:
TRANSACTION 42197, ACTIVE 0 sec inserting
INSERT INTO t VALUES (25)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS ... lock_mode X locks gap   ← 对方拿着间隙锁
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS ... insert intention waiting   ← 插入意向锁在等
*** WE ROLL BACK TRANSACTION (2)    ← 官方裁决：回滚代价小的一方
```

解读五要素：

1. **两个事务各自的 SQL**——直接拿到案发现场；
2. **HOLDS vs WAITING FOR**——谁拿什么、等什么；
3. **锁模式**：`lock_mode X`（记录锁）、`lock_mode X locks rec but not gap`、`insert intention waiting`（间隙相关）——对应 Level 2 的 Case B；
4. 涉及的**索引名**（index PRIMARY / index idx_xxx）——判断是否走了预期索引；
5. **WE ROLL BACK TRANSACTION (n)**——victim 是谁。

### 监控与告警 SOP

```sql
-- 把所有死锁（不只最近一次）写进 error log【动态全局参数，官方文档】
SET GLOBAL innodb_print_all_deadlocks = ON;

-- 趋势指标：累计死锁次数计数器【官方 status counter，已查证】
SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks';

-- 实时观察当前锁等待（5.7 示例）
SELECT * FROM information_schema.INNODB_TRX\G            -- 长事务在哪
SELECT * FROM information_schema.INNODB_LOCK_WAITS;      -- 谁等谁

-- 8.0 用 performance_schema / sys（推荐直接用 sys 入口）
SELECT * FROM sys.innodb_lock_waits\G
```

- 生产建议用 Percona Toolkit 的 **pt-deadlock-logger** 持续轮询死锁日志落库归档，做趋势分析；
- 应用侧把 **1213 与 1205 分开计数告警**：
  - 1213 偶发 + 重试成功率高 → 属于正常摩擦，观察趋势即可；
  - 1213 突增 → 说明新代码引入了反序或无索引 SQL；
  - 1205 出现 → 存在长持锁事务，比死锁更危险（没人回滚它，大家都在干等）。

### 定位五步法

```
① 从死锁日志提取两条 SQL 与锁模式
② EXPLAIN 两条 SQL → 确认各自走了哪个索引、扫描多少行
③ 判断成因归类：反序(A)/间隙(B)/放大(C)
④ 回到业务代码还原事务边界：@Transactional 范围内还有什么调用？
⑤ 归因后按 Level 4/5/6 对号入座治理
```

**本层留下的新账**：环抓出来了、根因归好类了——**怎么治**？从离数据库最近的一层开始 → 逼出 Level 4。

> 🔴 **口诀**：日志看 HOLD 和 WAIT，1213 重试 1205 危，EXPLAIN 还原锁范围。

<a name="level-4"></a>
## Level 4：SQL/事务层治理——破坏循环等待

### 👶 前置知识关卡

- ✅ Level 2 三大成因的归类能力？
- ✅ 团队有上线前 SQL 审核（EXPLAIN 门槛）流程吗？

### 先看全景：所有方案都在拆四条件

治理方案林林总总，本质都映射到破坏某一个必要条件——这张总表后面三层反复用到：

| 解决方案 | 破坏的条件 | 具体机制 |
|---|---|---|
| 统一加锁顺序 | 循环等待 | 所有事务单向申请资源，等待图无法成环 |
| 一次性锁定全部资源 | 持有并等待 | 要么拿到全部锁，要么都不拿（数据库里不可行，Level 1 已证） |
| 缩小事务 / 禁止事务内 RPC | 持有并等待 | 压缩持锁时间窗口，降低撞车概率 |
| 条件更新 / 乐观锁 | 互斥（弱化） | 把长持锁窗口变成瞬时原子操作 |
| Redis/Lua 原子扣减 | 互斥（转移） | 互斥收敛到原子操作，DB 无锁竞争 |
| 热点拆行 / 分桶 | 互斥（分散） | 增加资源数量，降低单资源竞争 |
| 死锁检测 + 回滚牺牲者 + 重试 | 不可剥夺 | 强制剥夺一方资源打破僵局（InnoDB 默认） |
| 锁等待超时 | 不可剥夺 | 超时强制失败，兜底 |

> 📌 一句话总结全部方案的哲学：**要么让锁竞争不发生，要么让等待不成环，要么让僵局可恢复**。

### ① 统一加锁顺序——直接拆掉"环"的一条边

**徒弟**："听起来简单，我先给你看我们现在的代码。"

❌ **问题代码一：跨表加锁顺序不一致**

```java
// ========== 下单服务：先订单表 → 后库存表 ==========
@Transactional
public void createOrder(OrderReq req) {
    orderMapper.insert(buildOrder(req));        // ① 锁住 orders 行
    inventoryMapper.deduct(req.getSkuId(), 1);  // ② 等待 inventory 行锁 ← 阻塞点
}

// ========== 回滚服务：先库存表 → 后订单表 ==========
@Transactional
public void cancelOrder(Long orderId) {
    Order order = orderMapper.selectById(orderId);
    inventoryMapper.restore(order.getSkuId(), 1); // ① 锁住 inventory 行
    orderMapper.updateStatus(orderId, CANCELED);  // ② 等待 orders 行锁 ← 阻塞点
}
```

死锁时序：

```
T1(下单):  锁 orders ✓ ──────── 等 inventory ✗（被 T2 持有）
T2(回滚):        锁 inventory ✓ ──── 等 orders ✗（被 T1 持有）
→ 循环等待 → InnoDB 回滚牺牲者 → 高频下单+取消 → 频繁死锁
```

❌ **问题代码二：批量扣减不排序**

```java
@Transactional
public void batchDeduct(List<Long> skuIds) {
    // 顺序由调用方决定！购物车传 [5,1,3]，后台传 [3,1,5]
    for (Long skuId : skuIds) {
        inventoryMapper.deduct(skuId, 1);
    }
}
// T1: 锁5✓→锁1✓→等3✗   T2: 锁3✓→等1✗ → 死锁
```

✅ **修复：全团队约定"永远先库存后订单" + 批量强制升序**

```java
@Transactional
public void createOrder(OrderReq req) {
    // ① 先库存表，按 sku_id 升序加锁
    List<Long> sortedIds = req.getSkuIds().stream().sorted().collect(toList());
    for (Long skuId : sortedIds) {
        int rows = inventoryMapper.deduct(skuId, 1);
        if (rows == 0) throw new BizException("库存不足");
    }
    // ② 后订单表
    orderMapper.insert(buildOrder(req));
}

@Transactional
public void cancelOrder(Long orderId) {
    Order order = orderMapper.selectById(orderId); // 只读快照不加锁
    inventoryMapper.restore(order.getSkuId(), 1);  // ① 先库存（同一规则）
    orderMapper.updateStatus(orderId, CANCELED);   // ② 后订单
}
```

**四个容易踩空的注意事项**：

1. 按**实际被加锁的键**排序，不是按业务字段排序；
2. 不依赖 SQL 内部执行顺序，**应用层显式排**；
3. 并发事务必须走**同一条索引访问路径**（否则排序白排，见 Case A 分析）；
4. 排序只能防 Case A，**防不住 Case B 间隙锁**——需要配合 Level 4④ 的 RC 降级。

### ② 缩小事务、缩短持锁时间

- 事务内**禁止 RPC 调用、发 MQ、复杂计算**——每多等 100ms，锁就多被持有 100ms；
- 副作用移出事务：用 Spring `TransactionSynchronization` 的 **afterCommit 回调**，提交成功后再发 MQ / 通知——事务回滚时副作用根本不会触发，一举两得；
- 校验逻辑提到事务外，事务里只留核心写操作。

### ③ 让每条写 SQL 都走索引精确制导

- 给 `order_no` 这类过滤列补索引，Case C 直接消失；
- 优先用**唯一索引点查**：Next-Key Lock 退化为纯 Record Lock，锁范围最小（退化规则见 mysql-01 Level 5 How 节）；
- 上线前 SQL 审核卡 `type=ALL` 的写语句。

### ④ RR → RC：用一致性预算换锁面积

收益【手册语义】：RC 下**多数**普通 DML 不再使用间隙锁，仅外键检查、重复键检查、锁定读等特殊路径仍可能产生锁冲突——Level 2 的 Case B 大幅收窄但不等价于整类消失；RC 也不能保证业务层不发生死锁，死锁主要取决于写操作、资源访问顺序和事务结构，隔离级别并非唯一决定因素，见[官方死锁文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)。

代价与前提，必须一起说清：

- RC 下不再防幻读，一致性语义交给应用层设计；
- RC 与语句格式复制存在不安全场景，生产通常采用 **ROW** 格式；具体行为需按语句类型和 MySQL 版本验证，不宜表述为"一律报错"；
- 5.7 的 `innodb_locks_unsafe_for_binlog` 在 8.0 已移除，RC 行为两代统一（mysql-01 Level 5 差异表已载）。

**适用判断**：互联网高并发库存/订单类业务多数接受 RC；资金台账类保持 RR 并靠排序+小事务治理。

**本层留下的新账**：SQL 层治完，剩下的冲突是"业务语义决定的真竞争"——**应用侧还能做什么**？→ 逼出 Level 5。

> 🔴 **口诀**：同序拆环、事务缩短、索引制导、RC 缩面（配 ROW）。

<a name="level-5"></a>
## Level 5：应用层治理——消除与消化冲突

### ① 条件更新与乐观锁：把"判断+扣减"压进一条语句

```xml
<!-- 条件更新版：原子扣减 + 防超卖 -->
<update id="deduct">
    UPDATE inventory SET stock = stock - #{num}
    WHERE sku_id = #{skuId} AND stock >= #{num}
</update>

<!-- 乐观锁版：版本号 CAS（WHERE 必须带旧 version 才叫 CAS） -->
<update id="deductOptimistic">
    UPDATE inventory SET stock = stock - #{num}, version = version + 1
    WHERE sku_id = #{skuId} AND version = #{oldVersion} AND stock >= #{num}
</update>
```

```java
int rows = inventoryMapper.deductOptimistic(skuId, num, oldVersion); // oldVersion 来自快照读，不加锁
if (rows == 0) throw new BizException("库存不足或版本冲突"); // 失败即返回，不持有长锁窗口
```

为什么它天然少死锁：条件更新把"判断+扣减"压进一条 UPDATE；乐观锁虽然要先读版本，但那次读是普通快照读、不加锁，真正持锁的只有最后一条 UPDATE——锁窗口从整个三段式缩短到一条语句（InnoDB 行锁仍持有到 commit，只是持有窗口极短）。

> 📚 **术语辨析（素材常见错误）**：`stock >= n` 是**原子条件扣减**，防的是超卖；`version` 字段 CAS 才是经典意义的**乐观锁**，防的是丢失更新。两者都消除了显式锁等待窗口，但机制不同——面试混用会被追问击穿。

### ② 死锁重试：接受摩擦，但要幂等

死锁不可能降为零（只要还有并发写），工程姿态是：**偶发 1213 → 自动重试 → 重试必须幂等**。

```java
@Retryable(retryFor = DeadlockLoserDataAccessException.class,  // Spring 对 1213 的翻译
           maxAttempts = 3,
           backoff = @Backoff(delay = 100, multiplier = 2))    // 指数退避
@Transactional
public void createOrderWithRetry(OrderReq req) {
    createOrder(req);
}

@Recover
public void recover(DeadlockLoserDataAccessException e, OrderReq req) {
    log.error("下单重试耗尽, req={}", req, e);
    throw new BizException("系统繁忙，请稍后重试");
}
```

> ⚠️ **注意**：`@Retryable` 和 `@Transactional` 同 bean 内自调用时，调用不会经过代理，重试可能不生效——生产建议把重试方法放在独立 Bean，或通过代理调用。

幂等的落地手段（缺一不可）：

- 业务单号唯一约束（重试重复插入直接被唯一键挡住）；
- 状态机校验（只有 `INIT→DEDUCTED` 一个方向，重复请求落在终态直接返回）；
- 只重试**写事务整体**，且重试前重新读取最新状态，不带旧值继续写。

### ③ 分布式锁串行化：把并发掐死在 DB 门前

```java
public void deductStock(Long skuId, int num) {
    RLock lock = redissonClient.getLock("lock:stock:" + skuId); // 锁粒度=单个SKU
    boolean acquired = false;
    try {
        acquired = lock.tryLock(3, 10, TimeUnit.SECONDS); // 最多等3s，持有10s自动释放防宕机
        if (!acquired) throw new BizException("系统繁忙");
        int currentVersion = inventoryMapper.selectVersion(skuId); // 快照读，不加锁
        int rows = inventoryMapper.deductOptimistic(skuId, num, currentVersion); // DB乐观锁双保险
        if (rows == 0) throw new BizException("库存不足或版本冲突");
    } finally {
        if (acquired) lock.unlock();   // ★ 必须放 finally
    }
}
```

三个要点：

1. **SKU 粒度**而非全局一把锁——全局锁把并发打到 1；
2. **必须设持有超时**（leaseTime），进程宕机后锁能自动释放；
3. **DB 层保留乐观锁**作双保险——分布式锁失效（如主从切换丢锁，回链 redis 主题）时防超卖。

### ④ Redis + Lua 预扣：把 DB 冲突挡在前置层

```lua
-- deduct.lua：原子扣减
local raw = redis.call('GET', KEYS[1])
local stock = tonumber(raw)
if not stock then return -2                       -- key 不存在或非法值
elseif stock < tonumber(ARGV[1]) then return -1   -- 库存不足
end
redis.call('DECRBY', KEYS[1], ARGV[1])
return 1
```

```java
public Long createOrder(OrderReq req) {
    Long ret = redisTemplate.execute(deductScript,
            List.of("stock:" + req.getSkuId()), req.getNum());  // ① Redis原子预扣
    if (ret != 1) throw new BizException("库存不足");
    mqProducer.send("order-create", String.valueOf(req.getSkuId()), req); // ② 发MQ
    return generatePreOrderId(req);
}

// 消费端：低并发落库，复用"统一排序+乐观锁"逻辑
@RabbitListener(queues = "order-create")
public void onMessage(OrderMsg msg) {
    retryTemplate.execute(ctx -> { createOrder(msg); return null; });
}
```

架构链路：

```
用户请求 → [Redis Lua 原子预扣] → [MQ 按 skuId 分区] → [消费者串行落库]
              无锁竞争              同SKU同队列           单线程+乐观锁
                ↑                      ↑                    ↑
             互斥收敛               排队串行            兜底防环
```

代价必须诚实列出：

- 引入最终一致性：Redis 扣了 DB 没落（崩溃/MQ 丢失）→ 需对账兜底（债务回链 13-distributed-tx）；
- 回滚链路变复杂（取消要逆向补回 Redis 与 DB 两层）；
- 运维面多一个组件。

**本层留下的新账**：常规冲突都消化了，还剩最后一种怪物——单个 SKU 的**极端热点行**（万人抢同一行）。任何行锁方案在这里都会塌 → 逼出 Level 6。

> 🔴 **口诀**：条件更新消窗口，重试必须配幂等，分布式锁防丢锁，预扣欠下对账债。

<a name="level-6"></a>
## Level 6：架构层治理——根治热点行

### ① 热点 SKU 拆行/分桶：把一行锁摊薄成 N 行

```sql
CREATE TABLE inventory_shard (
    sku_id   BIGINT,
    shard_no INT,
    stock    INT,
    PRIMARY KEY (sku_id, shard_no)
); -- 拆成 N 个分片，总库存 = SUM(stock)
```

```java
@Transactional
public void deductHot(Long skuId, int num) {
    for (Integer no : shuffledShards()) {          // 随机起点，避免全挤在 shard_0
        if (mapper.deductShard(skuId, no, num) > 0) return;
    }
    throw new BizException("库存不足");
}
```

效果示意 `[教学量级]`：千人抢 1 把锁 → 百人抢 10 把锁——锁热点被摊薄。代价：总量判断复杂化（某分片不足≠卖光，需遍历或二次路由）；对账逻辑随分片数翻倍。

### ② 队列串行化：让冲突根本不同框

```java
// 以 Kafka 为例：按 sku_id 哈希分区，同 SKU 进入同一分区
kafkaTemplate.send("stock-change", String.valueOf(skuId), message);
// 消费端：分区内顺序消费，单分区内同 SKU 写者永远只有一个
@KafkaListener(topics = "stock-change", concurrency = "1") // 生产可按分区数调大，单分区内仍顺序
```

同一 SKU 的变更按 `hash(sku_id)` 投到同一分区，分区内顺序处理——同一行的写者永远只有一个，死锁失去存在前提。RabbitMQ 场景则按 SKU 建多队列并绑定路由键，每队列 `concurrency=1`。代价：链路延迟上升；吞吐上限≈分区数×单线程消费速率 `[须实测]`；消费端故障时积压需预案。

### ③ `innodb_deadlock_detect=OFF`：极端场景的最后手段

【已查证：该参数为 **MySQL 5.7.15 新增**，见官方 Release Notes news-5-7-15——素材常见的"8.0 才有"是错的】

原理：超高并发打同一热点行时，大量线程排队，每次等待触发环检测，检测本身成为 CPU 热点。官方注释的场景正是"numerous threads wait for the same lock"。

准入清单（全部满足才考虑）：

- [ ] 冲突集中在少数热点行、且业务确认**不会形成真环**（单一 UPDATE 模式、无跨表反序）；
- [ ] 同时调小 `innodb_lock_wait_timeout`（如 1~2s），否则真死锁要等 50s 默认值才暴露；
- [ ] 有监控兜底：1205 错误率告警 + 定期抽样核对是否有"永不成功"的锁对；
- [ ] 有快速回滚开关（动态参数，可在线切回 ON）。

风险一句话：关掉巡场管理员后，真死锁不会消失，只会变成 1205 慢失败。

### ④ 加分项：引入分布式事务后的全局锁死锁

引入 Seata 这类框架后，死锁会升级为**跨服务的全局锁循环等待**。应对（完整机制回链 13-distributed-tx）：

- 缩短 `@GlobalTransactional` 范围，只包必要写操作；
- 服务调用链保持单向（A→B→C），禁止环形依赖；
- 能用消息最终一致性替代强一致的，优先替代（本地消息表/事务消息）。

**本层留下的新账**：单机武器库齐了——排序、乐观锁、分布式锁、预扣、拆行、串行化。**但所有场景都值得动用重武器吗？**秒杀和普通高并发下单，得的根本不是同一种病 → 逼出 Level 7。

> 🔴 **口诀**：拆行摊薄热点，队列不同框，关检测先立准入清单。

<a name="level-7"></a>
## Level 7：场景适配——秒杀和普通高并发是两种病

### Why——徒弟："武器这么多，是不是都上才稳？"

**老陈**："恰恰相反。**用错药的代价比病本身还高**。先分清你得的是哪种病。"

### What——两种病的对照表

| 维度 | 秒杀 | 普通高并发下单 |
|---|---|---|
| 热点分布 | 极端集中在少数 SKU | **分散**在大量 SKU 上 |
| 响应要求 | 可接受"排队中" | **必须同步返回**结果 |
| 核心矛盾 | 单行锁热点 | **事务交叉、加锁顺序、锁范围** |

**结论**：普通高并发下单的死锁，**大概率来自代码层面的顺序问题，而不是资源热点**——拆行、MQ 串行化这些重武器基本用不上。这是工程规范问题多于架构问题。

### How——普通高并发的四板斧

`[经验值]` 这四件事做到位，绝大多数死锁问题消失：

| 板斧 | 内容 | 对应 Level |
|---|---|---|
| ① 统一加锁顺序 | 批量 sort() 升序 + 跨表全局定序（写进团队规范） | L4 |
| ② 条件更新扣减 | `WHERE stock >= n` 原子扣减 | L5 |
| ③ 事务瘦身 | RPC/MQ 移出事务，用 afterCommit 回调 | L4 |
| ④ 重试 + 幂等 | @Retryable + 订单号预生成防重 | L5 |

辅助项：隔离级别评估降为 RC（L4④）。

### 什么时候才升级到 Redis？——监控触发的决策表

不要拍脑袋上架构，让监控信号说话：

| 监控信号 | 阈值参考 | 动作 |
|---|---|---|
| `Innodb_deadlocks` 增速 | 重试体系就位后仍持续上涨 | 先排查是否还有漏网的顺序问题 |
| 死锁集中在个别 SKU | Top10 占死锁的大头（占比阈值按基线定 `[须实测]`） | **仅对这些 SKU** 上 Redis 预扣 |
| 锁等待 P99 | 超出业务容忍（如 >100ms `[须实测]`） | 考虑缓存前置 + 异步落库 |
| 单库 TPS 瓶颈 | 写入逼近实例上限 `[须实测]` | 分库分表（回链 12-sharding） |

> 📌 主流做法是**混合策略**：只有热点收敛到少数 SKU 时，才对其单独改造；其余走普通链路。为分散流量上全套秒杀架构，是过度设计。

**本层留下的新账**：病诊断对了——那真到了热点收敛、量大到 DB 物理瓶颈的那天，**终局形态长什么样**？→ 逼出 Level 8。

> 🔴 **口诀**：普通下单是慢性病，四板斧就是处方；秒杀才是急症，重武器只打热点。

<a name="level-8"></a>
## Level 8：大厂库存中心——从治死锁到让竞争不发生

### Why——核心思想的跃迁

> 大厂不是把死锁"解决"得更好，而是通过架构**让 DB 行锁竞争根本不发生**。

算一笔账：公开技术分享中大促峰值的量级常见为几十万 TPS，而 MySQL 单实例写入上限通常在数千至数万 TPS 区间——两者相差一到两个数量级 `[均为公开分享量级说法，须实测]`。**同步扣 DB 在物理上不可能**，缓存前置不是优化，是唯一选择。

图书馆宇宙里这相当于：不再是每个分馆各自扛读者排队，而是建了**联盟中央调度台**——读者先在调度台预登记（Redis 预扣），登记单排队送进总馆慢慢誊写（MQ+落库），总馆和调度台定期核账（对账）。

### What——标准架构：库存独立服务 + Redis 前置 + 异步落库

```
用户下单 ──► 交易服务 ──► 库存服务（独立微服务）
                            │ ① Redis/Tair Lua 原子预扣（同步返回）
                            │ ② 发库存变更消息到 MQ
                            ▼ 削峰
                          落库消费者：聚合批量写 MySQL
                          （单 SKU 串行 + 批量合并 UPDATE）
                            ▼
                          对账系统：Redis ↔ DB 定时核对+补偿
```

头部电商（阿里 Tair+单元化、京东库存分层、美团分桶等公开分享）基本都是这个骨架的变体 `[公开架构分享，非官方披露]`。

### How——六个关键组件

**① Redis 扣减也要分桶**——单 key 也是热点：

```java
int bucket = ThreadLocalRandom.current().nextInt(BUCKET_COUNT);
Long ret = luaDeduct("stock:" + skuId + ":bucket_" + bucket, num);
if (ret == -1 && totalStock(skuId) >= num) retryOtherBuckets(skuId, num);
// 单桶不足但总量够 → 轮询其他桶；总库存判断要独立于扣减
```

**② 落库聚合批量**——锁次数从"每单一次"降到"每 SKU 每周期一次"：

```java
@Scheduled(fixedDelay = 200)
public void flushToDb() {
    Map<Long, Integer> merged = buffer.drainAndMergeBySku(); // 按SKU聚合净变更
    merged.forEach((skuId, delta) -> mapper.batchAdjust(skuId, delta));
}
```

**③ 库存分层，允许可控超卖**：销售库存（Redis，可适度超发如 105% `[业务策略示例，非通用标准]`）与实物库存（DB/WMS）分离，履约环节兜底超卖订单——**用业务容忍度换技术一致性**。

**④ 回滚链路**：30 分钟延迟消息触发关单回补；回补前先 CAS 订单状态保证幂等（防延迟消息重复触发）。

**⑤ 对账体系**：Redis 总库存 ↔ DB 总库存 ↔ 订单流水推算值，三方核对，差异自动补偿或告警。

> 🚨 **没有对账的异步架构不敢上生产。**

**⑥ 兜底三板斧**：

- **热点探测**：实时统计 SKU 频次，爆款自动切分桶模式；
- **限流**：库存服务入口按 SKU 维度限流；
- **降级**：Redis 挂了切 DB 乐观锁链路（慢但正确）——**宁可拒绝服务，不可超卖**。

### DB 层手段

分库分表（按卖家/SKU 哈希，回链 12-sharding）；大促前对预测爆款动态拆行、大促后合并；批量合并 UPDATE；事务只包最小写集合。

### 规范依然是地基

统一锁序、事务内禁 RPC（静态扫描卡点）、强制幂等、全链路压测不达标不准上线——在大厂，Level 4/5 的"四板斧"变成**代码规约 + Code Review 卡点 + 自动化检测**。治理的最低层和最高层在这里汇合：**架构消灭热点，规范防住长尾**。

### 演进路径：五个阶段

```
阶段1 单体直扣DB          → 死锁频发（本文起点的问题）
阶段2 排序+乐观锁+重试     → 死锁基本消除 [吞吐须实测]
阶段3 缓存前置+MQ异步      → DB只做持久化 [吞吐须实测]
阶段4 分桶+热点探测+对账    → 大促量级 [吞吐须实测]
阶段5 单元化/异地多活       → 容灾级别（跨机器存活的完整故事 → mysql-03）
```

> 📌 每个阶段都是被上一阶段的**监控信号**逼出来的，不是一步到位设计出来的——这正是 Level 7 决策表的延伸。

**本层留下的新账：单机账闭合；跨机器的新账由文末系列承接另立。**

```
并发写 → 锁等待 → 可能成环 → 检测回滚（成本）
                                  ↑
治理链路把这条链逐环剪断：
L4 排序/小事务/索引    → 减少等待与环的形成
L5 条件更新/预扣/串行化 → 减少进入锁世界的请求
L6 拆行/OFF            → 改变冲突的结构本身
L7 场景适配            → 决定什么病用什么药
L8 库存中心            → 让竞争根本不发生
```

但这一切都假设数据在一台机器上。如果这台机器挂了呢？binlog 在 mysql-01 里只是"传抄记录"的比喻——传抄网络怎么组织、一台机器倒了另一台怎么顶上，这是下一篇的账。

> 🔴 **口诀**：大厂不治死锁，治"死锁的存在条件"；对账是异步架构的命根子。

<a name="landing-checklist"></a>
## 📋 落地清单

```
✅ 必做（上线前）：
   1. 批量操作 sort() 升序
   2. 跨表操作全局定序（规范 + Code Review 卡点）
   3. 扣库存改条件更新/乐观锁
   4. 事务瘦身：RPC/MQ 移出事务（afterCommit）
   5. UPDATE 语句确认走索引（EXPLAIN 卡 type=ALL）
   6. 死锁异常重试 + 业务幂等
   7. 隔离级别评估降为 RC（配 ROW binlog）

📊 上线后：
   8. 监控 Innodb_deadlocks、锁等待 P99，1213/1205 分开告警
   9. 定期分析 SHOW ENGINE INNODB STATUS 死锁日志（pt-deadlock-logger 归档）

🚀 触发式升级（监控信号驱动，见 Level 7 决策表）：
   10. 热点收敛到少数 SKU → 对其单独引入 Redis 预扣
   11. 量大到 DB 物理瓶颈 → 缓存前置 + MQ 异步 + 对账补偿
```

<a name="self-test"></a>
## 🧪 合书自测

1. 为什么 InnoDB 不预防死锁而选择检测+回滚？预防三方案各贵在哪？
2. victim 的选择规则是什么？为什么不能假设"总是回滚后到的那个"？
3. 画出"查不到就插入"的两会话死锁时序，指出哪一步依赖了"间隙锁兼容共存"？
4. 无索引 UPDATE 锁的是什么？和表锁的区别？如何用 EXPLAIN 预估锁范围？
5. 1213 和 1205 分别意味着什么？哪个更需要紧急告警？
6. 普通高并发下单的死锁根因概率排序，前三位是什么？
7. 统一排序的四个注意事项是什么？为什么排序防不住间隙锁？
8. `stock >= n` 条件更新和版本号乐观锁的区别？分别防什么问题？
9. 重试机制的幂等怎么落地？举两个手段。@Recover 的作用？
10. 分布式锁为什么必须设持有超时？为什么要保留 DB 乐观锁？
11. 关闭 `innodb_deadlock_detect` 前必须满足哪些准入条件？风险是什么？
12. 秒杀和普通高并发下单的核心矛盾区别？四板斧是哪四个？
13. 什么监控信号触发"对个别 SKU 上 Redis 预扣"？
14. 为什么说"没有对账的异步架构不敢上生产"？三方对账对的是哪三方？

<a name="pitfalls"></a>
## ⚠️ 坑与细节

1. **"无索引=升级为表锁"是误称**：机制仍是行锁，只是全扫描逐行都锁，效果近似。面试说"表锁"会被追问击穿。
2. **IN 列表书写顺序不影响加锁顺序**：顺序由索引扫描路径决定；风险在不同索引路径与跨表反序。
3. **间隙锁之间兼容共存**是两会话死锁的前提，也是"锁不存在的行也会死锁"的原因。
4. **插入意向锁彼此不阻塞**：两个 INSERT 互等的是对方的**间隙锁**，不是彼此。
5. **victim ≠ 总是后到的**：按影响行数衡量，大事务可能反而安全。
6. **1213 秒级失败、1205 干等 50s**：监控上后者更危险，因为没有自动裁决方。
7. **RC 不是免费午餐**：幻读防护交还应用，且 ROW binlog 是硬前提。
8. **重试不幂等 = 重复扣款**：唯一约束 + 状态机是底线配置。
9. **`REPLACE INTO` 高危**：DELETE+INSERT 语义放大间隙锁（详见 mysql-01 坑表）。
10. **事务内 RPC/MQ 是持锁时间炸弹**：网络抖动 1s = 所有锁多持 1s。
11. **关闭检测后忘调 timeout**：真死锁以 50s 为单位慢性发作，比开着检测更隐蔽。
12. **热点拆行后忘了余数合并与对账**：分片间失衡会让部分分片提前售罄、部分积压。
13. **分布式锁的 leaseTime 太短**：业务没执行完锁先过期，等于没锁——靠 DB 乐观锁兜底。
14. **Redis 预扣后忘了对账**：崩溃或 MQ 丢失造成 Redis 与 DB 不一致，静默超卖或静默少卖。
15. **分桶后"单桶不足"不等于售罄**：必须二次路由其他桶，否则白白拒单。
16. **可控超卖只学一半**：超发比例必须和履约兜底配套，否则变成真超卖事故。
17. **降级方向搞反**：Redis 故障时应切"慢但正确"的 DB 乐观锁链路，而不是直接拒绝一切写。
18. **Seata 全局锁死锁被当成 DB 死锁排查**：现象相似、战场不同，先看全局事务再谈行锁。

<a name="errata"></a>
## 📚 勘误表（对常见素材/面试答案的纠错）

| # | 流传说法 | 纠正 | 依据 |
|---|---|---|---|
| 1 | "`innodb_deadlock_detect` 是 8.0 参数" | **5.7.15 新增** | MySQL 5.7 Release Notes（news-5-7-15）【已查证】 |
| 2 | "WHERE 没走索引会升级为表锁" | 全扫描**逐行加锁**，RR 下保持到 commit；机制是行锁 | InnoDB 锁机制文档语义 |
| 3 | "`stock>=n` 这种叫乐观锁" | 版本号 CAS 才是乐观锁；这是**原子条件扣减**，防超卖不是防丢失更新 | 术语定义区分 |
| 4 | "死锁了重启事务就行，不用管幂等" | 重试是第二次执行，无幂等则可能双倍扣减 | 唯一约束 + 状态机语义 |
| 5 | "MySQL 单机写入极限约 1 万 TPS"当定值引用 | 数千至数万 TPS 的**区间量级**，强依赖硬件与事务模型 | 数字纪律 `[须实测]` |

<a name="decision-cards"></a>
## 🏆 生产决策卡

### 卡 1 · 死锁治理选型（先诊断场景，再选武器）

| 场景 | 首选方案 | 兜底 |
|---|---|---|
| 普通高并发下单 | 四板斧：排序+条件更新+事务瘦身+重试幂等 | RC（配 ROW）+ SQL 审核 |
| 偶发间隙锁冲突 | RC 降级 + 消除"查不到就插入"模式 | 死锁日志归档分析 |
| 热点 SKU 集中 | 仅热点 SKU 上 Redis 预扣 + MQ 异步落库 | 拆行 + 余数合并 |
| 单点极限热点 | 队列串行化 | deadlock_detect OFF（准入清单） |

### 卡 2 · 隔离级别决策

| 问题 | 选 RR | 选 RC |
|---|---|---|
| 是否依赖间隙锁防幻读 | 是 | 否（应用层自证） |
| binlog 能否全量 ROW | 可以 | 可以 |
| 死锁/间隙锁冲突是否为主要矛盾 | 否 | 是 |

### 卡 3 · 关闭死锁检测准入清单

确认无环模式 ✔ / 调小 lock_wait_timeout ✔ / 1205 告警就位 ✔ / 动态开关可随时切回 ✔ —— 四项缺一不放行。

### 卡 4 · 架构升级时机（监控信号驱动）

| 信号 | 动作 |
|---|---|
| 死锁集中于少数 SKU | 仅对热点 SKU 引入 Redis 预扣 |
| 锁等待 P99 超容忍线 | 缓存前置 + 异步落库 + 对账 |
| 单库写入逼近上限 | 分库分表（回链 12-sharding） |
| 大促量级 + 容灾要求 | 库存中心全架构 + 单元化（→ mysql-03/04） |

<a name="cross-system"></a>
## 🌍 跨系统迁移

| 系统 | 同构问题 | 可迁移的解法 |
|---|---|---|
| JVM synchronized/JUC | 嵌套锁反序死锁（jstack 可见死锁报告） | 同样适用"资源排序"；jstack ≈ wait-for graph 快照 |
| 分布式锁 | 多客户端互持对方要的锁 | 按 key 全序获取；fencing token ≈ 让"不可剥夺"变得可仲裁 |
| 操作系统 | 资源分配图死锁 | 银行家算法=预防路线的教科书形态（同样败于"预知未来"） |
| 微服务调用 | 循环依赖互相等待（服务级"死锁"） | 调用方向约定 ≈ 加锁全序 |
| Seata/分布式事务 | 全局锁循环等待 | 缩短全局事务范围、单向调用链（回链 13-distributed-tx） |

## 系列承接

> **本篇收口单机并发的故事**：锁是并发正确性的成本，死锁是成本的极端形态。从 SQL 排序到库存中心，本质只有一句话——**大厂不治死锁，治死锁的存在条件**。
>
> **下一篇（mysql-03《主从复制与高可用》）承接的问题**：以上一切的前提是"数据和算力在一台机器上"。当这台机器成为单点——binlog 这个"传抄记录"如何组织成总馆-分馆网络？主库挂掉的瞬间，谁顶上、丢多少、怎么不丢？
>
> mysql-01 开头承诺的六层保证中，"可追溯"一环在此正式开工。
