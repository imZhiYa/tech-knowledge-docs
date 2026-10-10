# 🐬 MySQL 事务与隔离级别实战

> **本文定位**：从事务语义出发，通过并发时间线和双会话 SQL 实验，解释 ACID、四种隔离级别、快照读与当前读、事务控制语句，以及线上事务的边界设计与失败恢复。
>
> **主线案例**：账户余额与电商库存。我们先观察并发如何破坏业务不变量，再逐步引入事务、隔离级别、锁、MVCC 和工程治理。
>
> **学习目标**：读完后，不仅能解释“RR 为什么可重复读”，还应该能在本地 MySQL 复现并发行为，识别业务代码中的事务边界问题，并根据一致性要求选择可验证的方案。

---

## 0. 阅读地图与证据边界

### 本文回答什么

| 层次 | 核心问题 | 本文交付 |
|---|---|---|
| 事务语义 | 为什么多条 SQL 需要作为一个整体？ | ACID、事务边界、业务不变量 |
| 并发异常 | 每条 SQL 都成功，为什么最终结果仍可能错误？ | 脏读、不可重复读、幻读、丢失更新、写偏斜 |
| 隔离语义 | 四种隔离级别分别承诺什么？ | 标准语义与 InnoDB 行为的对照 |
| 内部机制 | RR 的普通查询为什么看旧值，锁定查询却可能看到新值？ | Read View、快照读、当前读、锁 |
| 操作方式 | 如何开启、提交、回滚以及控制事务？ | `autocommit`、`SAVEPOINT`、隐式提交 |
| 工程实践 | 出错后如何重试？如何避免长事务？ | 幂等、重试、监控、Java/Spring 边界 |

### 与本系列其他文档的边界

| 文档 | 职责 | 本文如何衔接 |
|---|---|---|
| [mysql-01-InnoDB深度解析.md](./mysql-01-InnoDB深度解析.md) | InnoDB 存储、锁、MVCC、日志与 UPDATE 全链路 | 本文解释事务行为，内部数据结构回链该文 Level 5/6/7 |
| [mysql-02-锁冲突与死锁治理.md](./mysql-02-锁冲突与死锁治理.md) | 死锁定位、SQL/应用/架构治理 | 本文说明错误语义与重试边界，不重复完整死锁治理方案 |
| [mysql-03-主从复制与高可用.md](./mysql-03-主从复制与高可用.md) | Binlog 复制、故障切换和高可用 | 提交与持久化的细节回链对应章节 |
| [mysql-04-备份恢复与PITR.md](./mysql-04-备份恢复与PITR.md) | 数据备份、时间点恢复与容灾 | 事务提交不等于备份，恢复能力由独立机制保证 |
| `docs/13-distributed-tx/` | 跨库、跨服务事务 | 本地 InnoDB 事务的边界无法自动扩展到外部服务 |

### 版本与证据边界

| 类型 | 基线与约定 |
|---|---|
| SQL 语义 | 以事务与隔离级别的通用语义说明为主；明确区分规范概念和 MySQL 实现 |
| 实现基线 | MySQL 5.7.44；对照 MySQL 8.0.x / 8.4 LTS。代码示例尽量兼容 5.7，版本差异单独标注 |
| 存储引擎 | 核心实验使用 InnoDB。事务语义不能直接推广到不支持事务的存储引擎 |
| 执行结果 | 实验结果基于给定的隔离级别、索引、事务顺序和默认配置；请在目标版本亲自复核 |
| 性能数据 | 不给出未经实测的吞吐或延迟数字；性能结论需要结合数据量、索引和负载压测 |

> **术语纪律**：SQL 标准定义“应该防止什么”；InnoDB 决定“通过什么机制实现”；应用代码决定“这些机制是否覆盖了完整业务不变量”。三者不能混为一谈。

---

## Level 1：事务到底要解决什么问题？

### 1.1 Why：单条 SQL 正确，不代表业务整体正确

假设银行账户 A 有 100 元，账户 B 有 50 元，需要从 A 向 B 转账 30 元。业务规则是：

1. A 扣减 30 元；
2. B 增加 30 元；
3. 两步必须共同成功或共同失败；
4. A 的余额不能变成负数。

如果只执行两条独立 SQL：

```sql
UPDATE account SET balance = balance - 30 WHERE id = 'A';
UPDATE account SET balance = balance + 30 WHERE id = 'B';
```

第一条成功、第二条失败时，钱会凭空减少 30 元。即使两条语句单独都具备原子性，也不能保证它们作为一个业务动作的整体原子性。

正确做法是将相关写操作放到同一个本地事务中：

```sql
START TRANSACTION;

UPDATE account
SET balance = balance - 30
WHERE id = 'A' AND balance >= 30;

-- 应用必须检查第一条 UPDATE 的影响行数是否为 1。
-- 若为 0，说明账户不存在或余额不足，应回滚，而不是继续入账。

UPDATE account
SET balance = balance + 30
WHERE id = 'B';

-- 同样应检查目标账户是否存在，并处理异常。
COMMIT;
```

这里的示例展示事务结构。生产实现必须在应用侧根据每条 SQL 的影响行数决定是否继续；任何一步失败，都应回滚整个转账事务。不要把“SQL 没抛异常”当成“业务条件已经满足”。

### 1.2 What：ACID 是什么，不是什么

| 属性 | 要表达的保证 | InnoDB / 应用中的对应机制 | 常见误解 |
|---|---|---|---|
| **Atomicity 原子性** | 一个事务的修改要么整体生效，要么整体撤销 | Undo、事务提交/回滚流程；事务内所有相关表都必须支持事务 | 把多条自动提交的 SQL 当成一个事务 |
| **Consistency 一致性** | 事务执行前后，应用要求的数据不变量应保持成立 | 数据库约束、原子更新、事务隔离与应用业务校验共同维护 | 认为数据库开启事务后，会自动知道所有业务规则 |
| **Isolation 隔离性** | 并发事务之间按隔离规则观察彼此的操作 | 锁、MVCC、隔离级别 | 认为隔离级别越高，就能自动解决所有业务竞态 |
| **Durability 持久性** | 事务提交后，在承诺的故障模型和配置范围内保留已提交结果 | Redo Log、刷盘策略、操作系统与存储设备的持久化语义 | 认为客户端收到 COMMIT 就等于能抵御任意硬件故障或整套机房丢失 |

**重点：一致性更多是事务要维护的目标，而不是数据库凭空创造的业务规则。**

例如，数据库可以通过 `CHECK`（需注意 MySQL 版本）、`NOT NULL`、唯一约束和外键维护部分不变量；但“订单只能发货一次”“支付金额必须与订单金额完全一致”“一个用户同时只能有一个生效任务”等业务规则，仍需要合适的状态条件、约束和事务设计共同实现。

持久性也有边界。提交成功后的数据能否抵御断电、磁盘损坏、主库故障或整个可用区故障，还取决于日志刷盘策略、复制确认策略、硬件是否诚实地报告持久化完成，以及备份和容灾设计。ACID 的 D 不等于备份和高可用。

### 1.3 本地事务的边界

一个 InnoDB 本地事务可以原子地管理同一 MySQL 实例内、支持事务的表的修改。但它不会自动覆盖：

- Redis 的写入；
- Kafka/RocketMQ 消息的发送；
- HTTP/RPC 调用的远端修改；
- 另一个没有参与同一事务协议的数据库；
- 已经发送给用户的短信、邮件或外部支付请求。

如果在本地事务中先更新数据库，再调用远端服务，远端失败时本地可以回滚，但远端已经成功的副作用无法靠本地 `ROLLBACK` 撤销。跨系统一致性应根据业务选择 Outbox、可靠消息、Saga、TCC 或其他明确方案，参见分布式事务专题。

### 本层留下的新账

事务让多个修改成为一个整体，却仍需要决定并发事务可以看到什么。下一层从两个事务同时修改账户余额开始，观察并发异常如何产生。

---

## Level 2：并发事务为什么会出错？

### 2.1 丢失更新：应用拿旧值覆盖新值

初始余额为 100。T1 要增加 20，T2 要增加 30。两个应用线程先读取余额，再在应用内计算绝对值：

```text
T1: SELECT balance → 100
T2: SELECT balance → 100
T1: 应用计算 100 + 20 = 120
T2: 应用计算 100 + 30 = 130
T1: UPDATE balance = 120; COMMIT
T2: UPDATE balance = 130; COMMIT

最终余额 = 130，而不是 150。
```

问题的关键不是“两个 UPDATE 一定没有锁”。InnoDB 更新同一条记录时仍会进行必要的并发控制；真正的问题是第二个应用线程把旧快照算出的绝对值 `130` 写回，覆盖了已经完成的 `120`。

更稳妥的设计通常有三种：

**方案 A：让数据库原子地执行增量。**

```sql
UPDATE account
SET balance = balance + 20
WHERE id = 1;
```

不要先 `SELECT`，再在应用内计算绝对余额，最后写回常量。数据库会在并发控制下完成字段增量更新。

**方案 B：乐观锁版本号。**

```sql
UPDATE account
SET balance = ?, version = version + 1
WHERE id = ? AND version = ?;
```

应用提交时带上读取到的 `version`。影响行数为 1 表示更新成功；为 0 表示版本已经变化，需要重新读取并根据业务决定重试或返回冲突。

**方案 C：先锁定再读取和修改。**

```sql
START TRANSACTION;
SELECT balance FROM account WHERE id = 1 FOR UPDATE;
-- 在持锁期间进行必要的业务判断和更新
UPDATE account SET balance = balance + 20 WHERE id = 1;
COMMIT;
```

该方法适用于必须先读取多个字段才能作出决策的情况，但会增加锁等待，需要缩短事务并确保访问顺序一致。简单计数或余额增量，通常优先考虑条件更新或原子算术。

### 2.2 脏读：读到了最终会被撤销的修改

```text
T1: BEGIN
T1: UPDATE product SET price = 50 WHERE id = 1   -- 尚未提交
T2: SELECT price FROM product WHERE id = 1        -- 若允许脏读，可能读到 50
T1: ROLLBACK
```

T2 读取的 50 元从未作为已提交状态对其他事务正式生效。`READ UNCOMMITTED` 允许这种脏读；`READ COMMITTED`、`REPEATABLE READ` 和 `SERIALIZABLE` 都避免普通读取直接暴露其他事务未提交的修改。

**注意：即使某个隔离级别允许脏读，也不意味着它取消了所有写锁。** 隔离级别描述可见性与并发规则，不等同于“数据库有锁 / 没有锁”的开关。

### 2.3 不可重复读：同一事务两次读同一行得到不同版本

```text
T1: BEGIN
T1: SELECT price → 100
T2: BEGIN
T2: UPDATE product SET price = 200 WHERE id = 1
T2: COMMIT
T1: SELECT price → 200（在 READ COMMITTED 下可能发生）
```

在 `READ COMMITTED` 中，每次一致性读取会建立新的快照，所以 T1 的第二次普通读取可以看到 T2 已提交的修改。在 InnoDB `REPEATABLE READ` 下，同一事务的普通一致性读取复用首次一致性读取建立的快照，因此这个场景中的第二次普通读取仍通常看到 100。

### 2.4 幻读：范围查询结果集合发生变化

```text
T1: BEGIN
T1: SELECT id FROM orders WHERE amount > 100;  -- 返回 3 行
T2: INSERT INTO orders(id, amount) VALUES (99, 150)
T2: COMMIT
T1: SELECT id FROM orders WHERE amount > 100;  -- 可能变为 4 行
```

这里关心的是“查询集合”，而不仅是某条已有记录的字段值发生变化。

需要区分两个层面：

- SQL 标准中的幻读，是隔离现象的定义；
- InnoDB 如何阻止它，取决于读取方式与隔离级别。

在 InnoDB RR 下，同一事务的普通快照读通常复用既有快照，因此后续普通范围查询不会把快照建立后才提交的新行加入结果。对于锁定范围的当前读，则主要依靠记录锁、间隙锁与 Next-Key Lock 等机制防止相应范围内的并发插入。不能只用一句“RR 完全没有幻读”概括所有混合读取场景；特别是同一事务混用普通快照读与锁定读时，两种读取可能观察到不同的数据状态。

### 2.5 写偏斜：每个事务都遵守自己的快照，组合后却破坏业务规则

假设排班系统要求“值班医生至少有一人在线”。表中 Alice 和 Bob 都在线。两个事务分别查询当前在线人数，结果均为 2，然后各自把不同医生设置为离线：

```text
T1 快照读：在线人数 = 2
T2 快照读：在线人数 = 2
T1: UPDATE doctor SET on_call = 0 WHERE name = 'Alice'; COMMIT
T2: UPDATE doctor SET on_call = 0 WHERE name = 'Bob'; COMMIT

最终在线人数 = 0，业务不变量被破坏。
```

两个事务修改的是不同记录，因此未必产生同一行上的写冲突。快照隔离式行为可以允许它们分别提交。简单的脏读 / 不可重复读 / 幻读表格并不能覆盖所有业务不变量风险。

更可靠的方案包括：

- 用数据库约束表达能够直接表达的不变量；
- 将“是否允许最后一个人离线”的判断串行化到一条共享记录上；
- 使用 `SELECT ... FOR UPDATE` 锁定用于判定业务规则的共同资源，并确保索引和锁范围符合预期；
- 在确实需要更强并发约束时，评估 `SERIALIZABLE` 或显式串行化设计。

**关键原则：隔离级别不是业务规则的替代品。** 对跨多行、计数、配额、库存、唯一生效状态等规则，必须明确哪一个数据库约束或并发控制机制使不变量在并发下仍成立。

### 2.6 并发异常速查

| 现象 | 本质 | 常见防护方式 | 注意事项 |
|---|---|---|---|
| 脏读 | 读取未提交数据 | RC / RR / SERIALIZABLE | RU 允许脏读 |
| 不可重复读 | 同行再次读取时版本变化 | RR 快照读、锁定读取 | RC 每次一致性读使用新快照 |
| 幻读 | 同一条件的结果集合变化 | RR 快照读；需要保护当前范围时使用合适范围锁 | 普通快照读和当前读不是同一种行为 |
| 丢失更新 | 基于旧值计算后覆盖新值 | 原子更新、版本条件更新、锁定读 | RR 不会自动修复应用写回旧计算结果的问题 |
| 写偏斜 | 对不同记录的并发修改共同破坏跨行不变量 | 约束、共同资源加锁、串行化 | 必须按业务不变量设计，而不是只背隔离矩阵 |

---

## Level 3：四种隔离级别如何改变并发行为？

### 3.1 先看标准语义，再看 InnoDB 行为

| 隔离级别 | 脏读 | 不可重复读 | 标准定义中的幻读风险 | InnoDB 中最重要的读取特点 |
|---|---|---|---|---|
| `READ UNCOMMITTED` | 允许 | 允许 | 允许 | 普通读取可能看到其他事务未提交的版本；写操作仍有并发控制 |
| `READ COMMITTED` | 防止 | 允许 | 允许 | 每次一致性读建立新 Read View；大多数普通搜索场景不使用间隙锁，但外键检查、重复键检查等场景是例外 |
| `REPEATABLE READ` | 防止 | 防止 | 需结合读取方式理解 | 默认级别；同一事务的普通一致性读复用首次一致性读建立的快照；当前读按锁定规则运行 |
| `SERIALIZABLE` | 防止 | 防止 | 防止 | 更强地限制并发读写；事务中普通读取通常采用共享锁定行为，具体需结合是否启用 autocommit 与事务状态理解 |

这张表讲的是行为，不是简单的性能排行。隔离级别提高后，阻塞、死锁、重试与吞吐之间会出现权衡；实际成本还受 SQL 访问范围、索引、事务长度和冲突热点影响。

### 3.2 RC 与 RR 的关键区别

**READ COMMITTED（RC）**

- 每条普通一致性读都基于当次读取建立的新快照；
- 同一事务的两次 `SELECT` 可能看到两次查询之间已提交的修改；
- 通常减少部分间隙锁带来的插入阻塞，但并不是完全没有间隙锁；
- 常见于希望每次读取尽可能反映新提交数据的应用，但必须接受事务内多次查询结果可能变化。

**REPEATABLE READ（RR）**

- InnoDB 默认隔离级别；
- 同一事务普通一致性读的 Read View 通常由首次一致性读取建立，并在事务中复用；
- 锁定读、`UPDATE` 和 `DELETE` 不等同于普通快照读；对于唯一索引等值条件与范围条件，加锁方式也有区别；
- 能够提供稳定的普通读取快照，但不应据此认为所有跨行业务规则自动获得串行化保证。

### 3.3 为什么说隔离级别是一个“契约”

选择隔离级别之前，应先明确业务需要哪种观察语义：

- **只要求每条查询不读取未提交修改**：RC 可能满足，但需要接受同一事务多次查询读到不同的已提交版本。
- **需要同一事务中的普通读取保持同一视图**：RR 通常更自然。
- **需要多个读取和写入共同保护复杂不变量**：还要分析锁定范围、约束以及并发冲突，必要时使用锁定读或串行化，而不是仅切换隔离级别。
- **读操作用于决策，接着要修改同一资源**：评估条件更新、版本号或 `SELECT ... FOR UPDATE`，避免读取与修改之间出现竞态。

没有一种隔离级别能让所有业务都同时获得最低延迟、最高并发和最强语义保证。选型依据应是明确的一致性要求与实测结果，而不是“级别越高越好”或“RC 一定更快”。

---

## Level 4：快照读、当前读与 Read View

### 4.1 先区分两类读取

| 类型 | 典型 SQL | 目标 | 读取和并发处理的主要特征 |
|---|---|---|---|
| 快照读（Consistent Read） | `SELECT ...` 普通查询 | 读取满足 Read View 可见性规则的数据版本 | InnoDB 通常通过 MVCC 读取可见版本，避免普通读和写操作总是互相阻塞 |
| 当前读 / 锁定读 | `SELECT ... FOR UPDATE`、`SELECT ... FOR SHARE`；修改语句也按当前数据处理 | 读取并锁定后续决策/修改所需的记录或范围 | 需要取得相应锁；与其他事务存在冲突时可能等待 |

MySQL 8.0 支持 `FOR SHARE`；MySQL 5.7 常见写法是 `LOCK IN SHARE MODE`。示例优先使用两代均支持的 `FOR UPDATE`。

并非所有看起来像 SELECT 的语句都必然是普通快照读。例如 `SELECT ... FOR UPDATE` 明确是锁定读；带有 `INSERT ... SELECT` 等结构的语句，其读取和锁定行为也有专门规则。不要只根据 SQL 开头的关键字判断它是否使用快照读。

### 4.2 Read View 的职责

Read View 可以理解为：**在某次一致性读取中，定义哪些事务的修改对当前事务可见的一组边界信息。** 它不是整张表的数据副本，也不是在 `BEGIN` 一定时刻把所有数据复制一遍。

在 InnoDB 的实现和相关版本中，理解可见性时会遇到这些概念：

- `m_ids`：创建 Read View 时仍处于活跃状态的一组事务 ID；
- `min_trx_id`：当时活跃事务 ID 的下界；
- `max_trx_id`：分配给后续事务的边界（不是简单等同于“当前最大已提交事务 ID”）；
- 记录的事务版本信息与 Undo 版本链：帮助判断某条记录的哪个版本对这个 Read View 可见。

概念性判断逻辑可以简化理解为：

1. 如果记录版本对应的事务在建立快照时已经提交，且不属于快照中仍活跃的事务，则该版本可能可见；
2. 如果该版本由建立快照时仍活跃的事务产生，则当前快照通常不能直接看到它，需要沿版本链寻找更早且可见的版本；
3. 若记录当前版本不可见，则 InnoDB 可借助 Undo 信息逐步重建旧版本，直到找到符合快照的版本或判断该行不可见。

这只是帮助理解的概念模型，不是完整源码算法。要精确到边界条件、事务 ID 分配和内部结构，请回链 [mysql-01 Level 6：MVCC](./mysql-01-InnoDB深度解析.md)。

### 4.3 为什么 RR 不是“BEGIN 时立刻复制快照”

在默认 RR 模式下，普通一致性读的快照通常在事务第一次执行一致性读时建立，而不是把 `BEGIN` 当作无条件复制所有表数据的时刻。`START TRANSACTION WITH CONSISTENT SNAPSHOT` 可以显式请求一致性快照，但其效果与隔离级别有关；在不支持该一致性快照语义的隔离级别下，不能用它强行制造 RR 行为。

这也意味着：事务开始与 Read View 创建是不同事件；事务 ID 的分配、快照可见性和实际修改记录同样不是同一个时间点。

### 4.4 双会话实验：RR 下快照读与当前读可能看到不同值

**准备数据（执行一次）：**

```sql
DROP TABLE IF EXISTS tx_account;
CREATE TABLE tx_account (
    id BIGINT PRIMARY KEY,
    balance DECIMAL(12, 2) NOT NULL,
    version BIGINT NOT NULL DEFAULT 0
) ENGINE=InnoDB;

INSERT INTO tx_account(id, balance, version)
VALUES (1, 100.00, 0);
```

确认会话 A、B 使用 RR：

```sql
SELECT @@transaction_isolation;
-- MySQL 5.7 使用：SELECT @@tx_isolation;
```

**会话 A：**

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM tx_account WHERE id = 1;
-- 100.00；建立本事务的普通一致性读快照
```

**会话 B：**

```sql
START TRANSACTION;
UPDATE tx_account SET balance = 200.00 WHERE id = 1;
COMMIT;
```

**回到会话 A：**

```sql
-- 快照读：仍然读取旧快照中的版本
SELECT balance FROM tx_account WHERE id = 1;
-- 预期：100.00

-- 当前锁定读：需要按当前数据和锁规则读取
SELECT balance FROM tx_account WHERE id = 1 FOR UPDATE;
-- 在会话 B 已提交、且不存在后续阻塞修改的情况下，预期可看到 200.00

COMMIT;
```

**观察结论：** 两条 SELECT 都在事务 A 内执行，但它们使用的读取语义不同。第一条普通 SELECT 受既有 Read View 约束；第二条是锁定读，按当前数据与锁定规则处理。

不要把该实验扩写成“所有 `FOR UPDATE` 在任何场景都无条件拿到全局最新值”。如果有其他事务持锁、尚未提交的更改或复杂语句结构，结果和等待行为还需要结合实际并发顺序分析。

---

## Level 5：事务控制语句与生命周期

### 5.1 `autocommit`：单条语句默认是自己的事务

MySQL 默认 `autocommit = 1`。在没有显式开启多语句事务时，一条成功执行的语句通常在语句结束时提交。

```sql
SELECT @@autocommit;

UPDATE tx_account
SET balance = balance + 1
WHERE id = 1;
-- 在默认自动提交模式下，这条成功的 UPDATE 作为独立事务提交。

START TRANSACTION;
UPDATE tx_account SET balance = balance + 10 WHERE id = 1;
UPDATE tx_account SET balance = balance + 20 WHERE id = 1;
ROLLBACK;
-- 两条更新都属于同一事务，ROLLBACK 撤销这两条事务修改。
```

`SET autocommit = 0` 会改变当前会话的自动提交模式。它属于 session 级设置，不应在连接池应用中随意修改而不恢复；否则连接被复用时，下一位请求可能继承非预期的事务状态。应用一般应通过 JDBC/Spring 的事务 API 管理事务，而不是每个业务方法任意切换全局连接状态。

### 5.2 `START TRANSACTION`、`COMMIT` 与 `ROLLBACK`

```sql
START TRANSACTION;  -- 也可以使用 BEGIN
-- 一组相关的 DML 操作
COMMIT;             -- 提交当前事务

START TRANSACTION;
-- 一组相关的 DML 操作
ROLLBACK;           -- 撤销当前事务尚未提交的修改
```

`COMMIT` 的逻辑含义是结束当前事务并提交其修改；具体持久化顺序涉及 Redo Log 与 Binlog 的协调提交、刷盘配置及崩溃恢复，详细原理回链 [mysql-01 Level 7：Redo / Undo / Binlog](./mysql-01-InnoDB深度解析.md)。

显式执行 `START TRANSACTION` 前，如果当前会话已存在未结束事务，需要特别小心：开启新事务可能结束此前事务。不要把 `BEGIN` 理解成任意嵌套的事务块。

### 5.3 `SAVEPOINT`：部分回滚，不是嵌套事务

```sql
START TRANSACTION;

UPDATE tx_account SET balance = balance - 10 WHERE id = 1;
SAVEPOINT after_debit;

UPDATE tx_account SET balance = balance + 10 WHERE id = 2;

-- 发现第二步不应该保留，回到保存点
ROLLBACK TO SAVEPOINT after_debit;

-- 可继续执行其他 SQL；此时外层事务仍然打开
COMMIT;
```

`SAVEPOINT` 提供的是当前事务内的部分回滚位置，并不会创建一个可以独立提交、独立持久化的嵌套事务。即使回滚到了保存点，外层事务仍然需要正确地 `COMMIT` 或 `ROLLBACK`。

### 5.4 隐式提交：不是所有 SQL 都能被 ROLLBACK 撤销

很多 DDL 会导致隐式提交，例如常见的 `CREATE TABLE`、`ALTER TABLE`、`DROP TABLE`、`TRUNCATE TABLE` 等。具体完整列表及例外应以当前目标版本的 MySQL 手册为准。

```sql
START TRANSACTION;
UPDATE tx_account SET balance = balance + 1 WHERE id = 1;

CREATE TABLE tx_demo(id BIGINT PRIMARY KEY) ENGINE=InnoDB;
-- 某些 DDL 会隐式提交此前事务；不能认为下面的 ROLLBACK 能撤销前面的 UPDATE。

ROLLBACK;
```

上面是说明风险的示意，不建议在生产环境中用这段 SQL 验证。需要“数据修改 + 表结构变更”时，应先确认数据库版本的 DDL 原子性、隐式提交规则和具体操作语义，通常把 DDL 与业务 DML 分开规划。

### 5.5 `READ ONLY`、`READ WRITE` 与隔离级别设置

```sql
-- 设置当前会话后续事务的默认隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 设置下一笔事务的隔离级别（只作用于下一笔事务）
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

START TRANSACTION READ ONLY;
SELECT COUNT(*) FROM tx_account;
COMMIT;
```

只读事务声明用于表达访问意图，并可能让引擎采用适合只读访问的内部路径；不要把它当作替代权限控制的安全边界。隔离级别的修改也应限定作用范围：确认是 `SESSION`、`GLOBAL` 还是“下一笔事务”，避免配置泄漏到连接池中的其他请求。

---

## Level 6：双会话实验室

> **实验规则**：使用两个独立连接（会话 A / 会话 B），不要把两段 SQL 顺序粘贴到同一个会话里。每组实验结束后显式提交或回滚，避免遗留事务影响后续结果。涉及阻塞的实验，必要时在第三个连接中查看进程和锁等待信息。

### 实验 1：RC 与 RR 对普通一致性读的差异

重置数据：

```sql
UPDATE tx_account SET balance = 100.00, version = 0 WHERE id = 1;
```

**先测 RR：**

会话 A：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM tx_account WHERE id = 1; -- 100
```

会话 B：

```sql
UPDATE tx_account SET balance = 200.00 WHERE id = 1;
COMMIT;
```

会话 A：

```sql
SELECT balance FROM tx_account WHERE id = 1; -- 预期仍为 100
COMMIT;
```

**再测 RC：**

先在事务结束后把测试数据重置为 100：

```sql
UPDATE tx_account SET balance = 100.00 WHERE id = 1;
```

会话 A：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM tx_account WHERE id = 1; -- 假设重置后为 100
```

会话 B：

```sql
UPDATE tx_account SET balance = 200.00 WHERE id = 1;
COMMIT;
```

会话 A：

```sql
SELECT balance FROM tx_account WHERE id = 1; -- 预期读取新的已提交值 200
COMMIT;
```

实验要点：在 RC 下，每次普通一致性读使用新的快照；在 RR 下，普通一致性读在事务中复用其快照。若准备数据或会话顺序不同，结果当然可能不同。

### 实验 2：复现应用层丢失更新

重置数据：

```sql
UPDATE tx_account SET balance = 100.00, version = 0 WHERE id = 1;
```

会话 A：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM tx_account WHERE id = 1;
-- 应用把返回值 100 放在内存中，准备写回 120；普通 SELECT 没有锁住这条记录。
```

会话 B：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM tx_account WHERE id = 1;
-- 应用同样读到 100，准备写回 130。
-- 暂时不要 COMMIT；普通 SELECT 不会因此锁住目标记录。
```

会话 A：

```sql
UPDATE tx_account SET balance = 120.00 WHERE id = 1;
COMMIT;
```

会话 B：

```sql
UPDATE tx_account SET balance = 130.00 WHERE id = 1;
COMMIT;

SELECT balance FROM tx_account WHERE id = 1;
-- 最终可能是 130，而非预期的 150。
```

为什么会发生：会话 B 虽然较早读取到了 100，但其 UPDATE 写入的是一个常量 130；它没有表达“在最新余额上增加 30”这个业务意图。

**改用原子更新：**

```sql
UPDATE tx_account SET balance = balance + 20 WHERE id = 1;
UPDATE tx_account SET balance = balance + 30 WHERE id = 1;
```

两条增量在并发执行时受 InnoDB 的更新锁与当前记录处理规则约束，最终都能反映在余额中（假定两条语句均成功且没有其他业务规则拒绝更新）。如果业务必须先读取后决策，使用版本条件或锁定读，而不是盲目写回旧值。

**改用乐观锁：**

```sql
-- 假设读取时 balance=100, version=0
UPDATE tx_account
SET balance = 120.00, version = version + 1
WHERE id = 1 AND version = 0;
```

- 影响行数为 1：版本匹配，本次更新成功；
- 影响行数为 0：版本已经发生变化，应用必须重新读取，并按业务规则重算或报告冲突。

### 实验 3：唯一键等值查询与范围锁不是一回事

锁的对象与具体锁定范围取决于索引、搜索条件、隔离级别和执行计划。可先观察唯一主键的精确匹配：

会话 A：

```sql
START TRANSACTION;
SELECT * FROM tx_account WHERE id = 1 FOR UPDATE;
-- 持有目标记录的排他锁，事务未结束前不要 COMMIT。
```

会话 B：

```sql
UPDATE tx_account SET balance = balance + 1 WHERE id = 1;
-- 等待会话 A 释放冲突锁。
```

会话 A：

```sql
COMMIT;
```

会话 B 的更新随后才能继续完成。

这组实验表明，`FOR UPDATE` 不只是“读取最新值”的语法，它还会获取必要的锁，并改变其他事务的执行时机。若将条件改成范围谓词、普通二级索引谓词或不存在的键值，实际锁定范围会有所不同，详见 InnoDB 专题的锁章节。

### 实验 4：死锁和锁等待超时不是同一种失败

死锁是等待关系形成环；当 InnoDB 死锁检测开启时，通常会选择一个事务回滚以打破循环。锁等待超时则是等待超过限制后触发错误，默认行为通常回滚当前失败语句，而不一定回滚整个事务。

| 失败类型 | 常见 MySQL 错误码 | 默认处理重点 | 应用恢复原则 |
|---|---:|---|---|
| Deadlock | `1213` | InnoDB 回滚一个事务以打破死锁 | 按整笔事务重新执行；重试必须有限次且可幂等 |
| Lock wait timeout | `1205` | 默认回滚当前超时语句；受 `innodb_rollback_on_timeout` 等设置影响 | 先明确事务还剩哪些语句生效，再决定回滚并重试哪一层 |

不要在捕获到任何数据库异常后都盲目执行“同一条 SQL 重试”。一旦事务处于部分失败状态，直接重试其中一条语句可能导致业务状态不一致。

---

## Level 7：事务的生产级设计与故障治理

### 7.1 事务应该尽可能短，但不能短到破坏业务原子性

事务持有记录锁或范围锁的时间越长，其他事务遇到冲突的机会通常越大。以下做法通常会不必要地延长事务：

- 开启事务后调用第三方 HTTP/RPC；
- 在事务中等待用户交互、消息消费或线程池任务；
- 一边遍历大量数据，一边逐条加锁更新；
- 长时间保持开启却不提交的只读事务，导致旧版本无法及时清理；
- 无索引地执行大范围 UPDATE / DELETE，扫描并锁定大量记录。

推荐顺序是：在事务开始前准备好不依赖数据库锁的输入；事务内只执行维持一致性所必需的读取与写入；尽快提交；事务外再进行可重试的远端副作用，并用 Outbox / 可靠消息等机制衔接需要跨系统一致性的场景。

不过，“事务越短越好”不是允许随意拆分原子业务的借口。如果把原本必须整体提交的两步拆成两个独立事务，就需要另行设计补偿、状态机和对账机制。

### 7.2 用 SQL 自身维护条件，缩小竞态窗口

**库存扣减：**

```sql
UPDATE inventory
SET stock = stock - 1
WHERE sku_id = ?
  AND stock > 0;
```

应用检查影响行数：

- `1`：扣减成功；
- `0`：SKU 不存在或库存不足，需要按业务区分处理。

**状态机迁移：**

```sql
UPDATE orders
SET status = 'PAID', paid_at = CURRENT_TIMESTAMP
WHERE id = ?
  AND status = 'WAIT_PAY';
```

把“当前状态必须为 WAIT_PAY”放进 WHERE 条件，可以避免两个并发请求都基于旧状态做出成功判断。依然要检查影响行数，并对数据库外的支付回调、消息重放等实现幂等处理。

**余额不能为负：**

```sql
UPDATE account
SET balance = balance - ?
WHERE id = ?
  AND balance >= ?;
```

要注意金额精度、传入参数是否始终是非负数，以及“扣减金额”与条件金额是否使用了同一个受验证的业务值。跨多行约束仍需更完整的事务设计。

### 7.3 如何选择悲观锁、乐观锁和原子更新

| 方案 | 适用场景 | 优点 | 成本与风险 |
|---|---|---|---|
| 原子条件更新 | 加减计数、库存扣减、状态条件迁移 | 一条 SQL 完成判断与修改，减少读改写窗口 | 复杂业务规则难以仅靠单条 SQL 表达；必须检查影响行数 |
| 乐观锁 / 版本号 | 冲突较少，允许冲突时重读重算 | 通常不需要提前锁定；并发冲突显式可见 | 冲突时要处理重试；重试逻辑需防止重复副作用 |
| `SELECT ... FOR UPDATE` | 必须先读取多字段，再根据结果作出决策 | 在决策窗口内锁定相关行，行为直观 | 增加等待、死锁与锁范围设计成本；必须使用合理索引 |
| 唯一约束 / 外键 / 检查约束 | 数据库能够直接表达的结构性不变量 | 不依赖所有应用调用路径都正确 | 约束设计有适用边界；错误需映射为业务冲突 |
| 串行化 / 队列化 | 强一致性关键路径、热点资源 | 并发规则更容易推理 | 并发度和吞吐可能下降，需评估排队与可用性 |

不要将分布式锁当作所有数据库并发问题的默认解法。若数据库本身就管理目标记录，优先评估事务、条件更新、约束和数据库锁是否足够；引入外部锁还会新增租约、过期、锁持有者失效、网络分区以及数据库实际写入是否受 fencing token 保护等问题。

### 7.4 重试要重试正确的边界

**死锁 `1213`：** InnoDB 通常回滚被选为 victim 的整笔事务。应用应该从事务入口重新执行完整业务步骤，而不是只重试导致死锁的最后一条 SQL。

**锁等待超时 `1205`：** 默认设置下通常只回滚失败语句。事务中的其他语句可能仍然属于未结束事务；是否整笔回滚还受 `innodb_rollback_on_timeout` 等设置影响。对复杂业务，最稳妥的恢复策略通常是明确回滚整个业务事务，再从事务入口重新执行，而不是依赖超时后的部分状态。

**重试策略建议：**

1. 只对明确可重试的瞬时错误重试；
2. 采用有限重试次数、指数退避和随机抖动，避免大量请求同时重放；
3. 重新执行完整事务，并重新读取参与决策的数据；
4. 业务写操作要么天然幂等，要么使用唯一业务请求号 / 幂等记录；
5. 外部副作用不得随事务重试无条件重复发送；
6. 若网络在 COMMIT 返回前断开，客户端可能无法判断提交是否已经成功，不能简单认定“失败后再提交一次”，应通过业务请求号查询结果或进行幂等重放。

### 7.5 长事务为什么危险

长事务的成本不仅是“锁占得久”。对于 InnoDB，旧版本的可见性和清理与活跃事务有关。长期活跃的事务可能让更老版本仍然需要保留，使 Undo 历史积累、Purge 清理落后，并增加后续查询和存储维护压力。

生产中至少要监控：

- 最老的活跃事务持续时间；
- 长时间未提交的连接与事务数量；
- 锁等待时长、超时数量和死锁数量；
- Undo history / Purge 相关指标（按目标版本确认具体指标名）；
- 慢 SQL、未走索引的大范围修改；
- 事务开始到提交的耗时分布，而不仅是单条 SQL 的耗时。

排查时结合 `information_schema.innodb_trx`、`SHOW ENGINE INNODB STATUS`，以及相应版本的 Performance Schema / sys 视图。MySQL 8.0 与 5.7 的锁信息视图存在变化，执行前应按版本核验。

### 7.6 Java / Spring 应用事务边界

数据库事务最终由连接和数据库执行，但 Java 应用框架会决定事务在哪里开始、什么时候结束、哪些异常触发回滚。

**常见注意点：**

- Spring 基于代理的声明式事务中，同一个对象内的自调用可能绕过代理，导致预期的 `@Transactional` 不生效；
- 默认回滚规则通常针对 `RuntimeException` 和 `Error`，受检异常默认不一定触发回滚；应依据实际 Spring 版本和 `rollbackFor` 配置确认；
- 异常被方法内部捕获且没有重新抛出时，事务拦截器可能认为方法正常完成，不能指望一定回滚；
- 事务通常绑定当前执行线程 / 事务上下文。在新线程、异步任务或 `CompletableFuture` 中执行的代码，不应默认认为会加入调用方的本地事务；
- 连接池复用连接，必须确保事务正常结束，不要把连接的自动提交设置、隔离级别或只读属性意外泄漏给下一请求；
- 不要在数据库事务中等待长时间远端调用；如果需要跨服务一致性，应设计明确的消息与补偿协议。

建议把本地事务边界放在能表达完整业务原子性的应用服务方法上，而不是让 Repository 的每个简单方法各自提交。若业务需要多个写操作作为一个整体，则由一个明确的事务边界统一组织。

---

## Level 8：高频误区与决策卡

### 8.1 高风险误区

| 误区 | 正确理解 |
|---|---|
| “开启事务后，所有 SQL 一定能回滚” | DDL 隐式提交、非事务引擎、外部系统副作用都有例外 |
| “RR 会自动防止丢失更新” | 基于旧快照计算绝对值后再写回，仍可能发生应用层覆盖；使用原子更新、版本号或锁定读 |
| “RR 下每一种读都读取同一个快照” | 这是普通一致性读的主要语义；锁定读与修改语句遵循当前数据和锁定规则 |
| “SELECT 是读操作，所以不会阻塞” | `FOR UPDATE` / `FOR SHARE` 是锁定读；某些语句结构和隔离级别会改变并发行为 |
| “RC 完全没有间隙锁” | 一般搜索场景减少间隙锁，但外键和重复键检查等情况仍可能需要它 |
| “死锁是数据库异常，出现一次就说明系统设计失败” | 死锁是并发执行中的可能结果；应用必须按事务边界处理重试，频发时再分析 SQL、锁顺序与热点 |
| “遇到 1205，只重试那条语句即可” | 默认仅失败语句被回滚，但外层事务可能仍未结束；必须理解事务当前状态，必要时整笔回滚 |
| “COMMIT 返回成功就等于整个系统永久不丢数据” | 提交持久性依赖配置和故障模型；复制、备份、容灾是独立层次 |
| “把数据库操作放进 `@Transactional` 就自动安全” | 代理、自调用、异常规则、线程上下文、事务范围以及数据库锁定语义仍然重要 |

### 8.2 生产决策卡

| 业务情况 | 优先评估 | 需要验证的关键点 |
|---|---|---|
| 计数器、库存扣减、余额增减 | 条件更新 / 原子算术 | 影响行数、边界条件、幂等和重试 |
| 读取多个字段后决定是否修改 | `SELECT ... FOR UPDATE` 或版本号 | 锁范围、索引、冲突处理和死锁 |
| 事务内重复读取要求稳定结果 | RR 普通快照读 | 所有读取是否为快照读，有无混入锁定读 |
| 高并发报表，希望读取新提交数据 | 评估 RC | 多条查询结果可能不同时点；汇总结果一致性要求是什么 |
| 复杂跨行不变量 | 数据库约束、共同资源锁、串行化设计 | 并发时不变量能否始终成立，是否需要集成测试 |
| 写数据库后发布消息 | Outbox / 可靠消息等 | DB 与消息中间件之间的失败窗口、重复投递和消费幂等 |
| 遇到死锁 | 事务级重试 + 死锁分析 | 事务是否整体重试、访问顺序、索引与热点 |
| 遇到长事务 | 缩短事务、调整访问模式 | 最老事务、锁等待、Undo/Purge、外部调用耗时 |

### 8.3 事务设计上线检查清单

- [ ] 事务边界和业务不变量已经明确；所有必须原子生效的 DML 都在同一个事务中。
- [ ] 相关表使用支持事务的存储引擎，且没有误用会造成隐式提交的语句。
- [ ] 对每条条件更新检查影响行数；业务状态迁移包含必要的旧状态条件。
- [ ] 读取后写回旧值的路径已评估丢失更新，必要时使用原子 SQL、版本号或锁定读。
- [ ] 锁定读和范围查询有合适索引；通过实际执行计划确认扫描与锁定范围。
- [ ] 事务内不包含不必要的网络调用、用户等待和长时间计算。
- [ ] 已实现死锁与超时的明确恢复策略；重试次数有限，业务副作用具备幂等保护。
- [ ] 已考虑 COMMIT 响应丢失导致的结果不确定性，并能通过业务请求号查询或幂等恢复。
- [ ] 监控事务持续时间、最老活跃事务、锁等待、死锁、超时和慢 SQL。
- [ ] RC / RR 等隔离级别行为已在目标 MySQL 版本和真实事务顺序下复核。
- [ ] 跨库、消息与远端 API 的一致性使用独立设计，不依赖本地事务“自动扩散”。

---

## Level 9：合书自测

请先尝试自己回答，再查看参考结论。

### Q1. ACID 中的一致性是数据库自动保证的吗？

**参考结论：** 不是。ACID 的 C 表示事务应维护业务与数据不变量；数据库约束、原子性、隔离性和应用业务逻辑共同帮助实现它，但数据库不会自动知道所有业务规则。

### Q2. InnoDB 默认 RR，为什么基于旧值的更新仍可能丢失更新？

**参考结论：** 普通快照读不等于锁定读。应用可能读取旧值、在内存中计算出常量，再将该常量覆盖写回。优先使用原子增量、版本条件或锁定读。

### Q3. RR 下 `BEGIN` 会立刻创建所有表的快照吗？

**参考结论：** 不会。普通一致性读的 Read View 通常在第一次一致性读取时建立；显式 `WITH CONSISTENT SNAPSHOT` 是独立语义，而且受隔离级别限制。

### Q4. 同一个 RR 事务中，普通 `SELECT` 与 `SELECT ... FOR UPDATE` 为什么可能看到不同值？

**参考结论：** 前者通常基于同一 Read View 做快照读；后者是锁定读，按当前记录和锁定规则执行。两者的目标和可见性处理不同。

### Q5. RC 是否完全不会使用间隙锁？

**参考结论：** 不是。一般搜索与索引扫描场景下 RC 会减少间隙锁，但外键约束检查、重复键检查等场景仍可能使用间隙锁。

### Q6. 一条 UPDATE 影响行数为 0，是否一定发生数据库错误？

**参考结论：** 不一定。可能是目标记录不存在，也可能是条件不匹配（如库存不足、旧状态不符或版本已变化）。应用需要按 SQL 设计的业务含义解释 0 行，而不是只检查异常。

### Q7. 死锁和锁等待超时有什么区别？

**参考结论：** 死锁是循环等待，检测开启时通常回滚 victim 的整笔事务；锁等待超时是等待超过阈值，默认通常只回滚超时的语句。两者的事务恢复边界不同。

### Q8. 为什么不建议在事务中调用第三方 API？

**参考结论：** 网络延迟会延长锁和事务上下文的持有时间；远端成功而本地事务回滚时，数据库回滚不能撤销远端副作用。应明确拆分本地事务与跨系统可靠交互。

### Q9. 什么情况下值得使用 `SELECT ... FOR UPDATE`？

**参考结论：** 业务必须读取当前记录并基于读取结果作出修改决策，且需要在决策区间阻止冲突更新时。简单计数和扣减常可用一条条件更新表达，避免无必要的先读后写。

### Q10. 提交前连接断开，客户端能否直接认定事务失败？

**参考结论：** 不能总是认定失败。服务端可能已经提交，但客户端没有收到结果。应使用业务唯一请求号、幂等机制和结果查询处理提交结果不确定问题。

---

## Level 10：参考资料与跨章节索引

### MySQL 官方文档

以下链接以 MySQL 8.4 Reference Manual 为主。涉及 5.7 / 8.0 / 8.4 差异时，应进一步切换到对应版本文档核验。

1. [InnoDB Transaction Model](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-model.html) —— InnoDB 事务模型总览。
2. [Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html) —— 四种隔离级别及 InnoDB 语义。
3. [Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html) —— 快照读、Read View 与一致性读取。
4. [Locking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html) —— `FOR UPDATE` / `FOR SHARE` 等锁定读。
5. [START TRANSACTION, COMMIT, and ROLLBACK](https://dev.mysql.com/doc/refman/8.4/en/commit.html) —— 事务控制、`autocommit` 和读写模式。
6. [Statements That Cause an Implicit Commit](https://dev.mysql.com/doc/refman/8.4/en/implicit-commit.html) —— 会导致隐式提交的语句与例外。
7. [InnoDB Error Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html) —— 语句回滚、事务回滚与错误处理。
8. [Deadlocks in InnoDB](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks.html) —— 死锁形成与处理。
9. [How to Minimize and Handle Deadlocks](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks-handling.html) —— 应用层和 SQL 层的死锁治理。

### 仓库内回链

- [InnoDB 深度解析](./mysql-01-InnoDB深度解析.md)：Level 4 事务与 ACID、Level 5 锁机制、Level 6 MVCC、Level 7 日志三剑客。
- [锁冲突与死锁治理](./mysql-02-锁冲突与死锁治理.md)：死锁现场、监控定位、重试策略和架构治理。
- [主从复制与高可用](./mysql-03-主从复制与高可用.md)：提交日志、复制与故障切换的后续链路。
- [备份恢复与 PITR](./mysql-04-备份恢复与PITR.md)：提交后的数据保护、备份和时间点恢复。

> **全篇最终结论：** 事务不是给 SQL 套一个 `BEGIN ... COMMIT` 就结束了。正确的事务设计必须同时明确业务不变量、隔离语义、读取方式、锁定范围、失败后的恢复边界，以及外部副作用如何实现幂等。只有这些层次一起设计和验证，才能把“数据库语句执行成功”转化为“业务状态在并发和故障下仍然正确”。