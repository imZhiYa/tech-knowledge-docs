# 🚀 SpringBoot 事务与数据层（系列 05）

> 本篇文章回答三个问题：
> 1. **数据源**：为什么不能裸写 JDBC？`DataSource` 到底是什么？Spring Boot 凭什么"不用配"就能连上 H2？
> 2. **事务**：`@Transactional` 一行注解是怎么变成"开事务 → 提交/回滚"的？默认回滚规则为什么是"只回滚运行时异常"？
> 3. **失效**：为什么同一类里 `this.method()` 调用带 `@Transactional` 的方法会静默失效？生产上最常踩的坑是什么？
>
> 前置：00 篇的四层创建链（代理怎么造的）、03 篇的自动装配（条件链怎么裁决的）、02 篇的事件机制（事务事件在 02 篇已讲，本篇补上事务本体）。本篇的账本（H2 内存库）会贯穿全文。

---

# ⚠️ 版本与证据边界

| 维度 | 本文承诺 |
| ---- | ---- |
| 代码实证 | `knowledge/springboot/experiments/code/` 下 demo12×4（ds.DataSourceApp / tx.TxBasicsApp / tx.PropagationApp / tx.SelfInvocationApp），本机实测输出原样引用 |
| 实测环境 | macOS + JDK 21.0.11（Azul Zulu）+ spring-boot **3.3.5** + spring-jdbc / spring-tx **6.1.14** + H2 **2.2.224** + HikariCP **5.1.0** + slf4j-api 2.0.16（版本均出自 spring-boot-dependencies-3.3.5.pom BOM） |
| 运行方式 | `cd knowledge/springboot/experiments/code && ./build.sh && ./run.sh demo12.XXX`（lib/ 41 个 jar，全部本地下载；DataSource 双跑法见实验复现节） |
| Specification | `javax.sql.DataSource` 契约（getConnection）、`PlatformTransactionManager` 契约（getTransaction/commit/rollback）、`@Transactional` 语义（回滚规则、传播、隔离级别的规范语义，来自 Javadoc 与 Spring 参考文档）、`@EnableTransactionManagement` 语义、JDBC Connection 事务语义（autocommit） |
| Implementation | DataSourceAutoConfiguration 条件链分支（3.3.5 实测：Hikari 分支 vs 内嵌分支）、TransactionAutoConfiguration 嵌套 @Configuration 提供 @EnableTransactionManagement（3.3.5 反编译 javap 实证）、JdbcTransactionManager bean（实测）、AccountService 为 CGLIB 代理（实测 isAopProxy=true）、事务内 isActualTransactionActive=true + afterCompletion 回调（实测）、CGLIB 代理字段注入行为（实测：代理实例上注入字段为 null） |
| 待验证 | 四个隔离级别的真实锁行为（本实验只用默认隔离级别，未实测脏读/不可重复读/幻读）；悲观锁/乐观锁；多数据源（AbstractRoutingDataSource）；XA 分布式事务；`@Transactional` 加在 private 方法/非 public 方法上的真实行为（规范要求 public，未实测破坏性场景） |
| 未覆盖 | SQL 本身；MyBatis/JPA 的具体整合（01 篇有 MyBatis 整合思路）；连接池参数调优数值（不承诺任何性能数字）；分布式事务中间件（Seata 等） |

---

# 🏷️ 关键词

DataSource | 连接池 | HikariCP | H2 内嵌库 | JdbcTemplate | PlatformTransactionManager | JdbcTransactionManager | @Transactional | 回滚规则 | rollbackFor | 传播行为 | REQUIRED | REQUIRES_NEW | 自调用失效 | CGLIB 代理 | @Lazy 自注入 | DataSourceAutoConfiguration | TransactionAutoConfiguration | afterCompletion

---

# 🗂️ 目录

- Level 1 数据访问抽象：JDBC 裸写为什么失败
- Level 2 事务抽象：PlatformTransactionManager 是什么
- Level 3 @Transactional 语义：回滚规则与传播行为（含全部实测）
- Level 4 代理机制：声明式事务怎么生效，怎么失效
- Level 5 数据层整合全景：自动配置的三条线
- Level 6 生产实践：失效排查 / 长事务 / 边界
- 全篇因果链总图
- 线上案例（3 个）
- 面试自查表
- 坑与细节（8 个）
- 版本勘误表
- 生产决策卡（3 张）
- 跨语言视角
- 系列索引

---

# 🏭 全文唯一比喻：记账员

一间账房（DataSource 连接池）雇着若干记账员（数据库连接）。要记账（写数据库）时，从账房领一个记账员（getConnection），用完归还（close 回池）——而不是每次现招一个人再辞退（JDBC 裸写每次新建连接）。

**一笔账（一个事务）必须一次记完**：要么全部入账（commit），要么全部划掉（rollback）。半笔账 = 账目对不上 = 生产事故。

全文主线角色：**一笔数据库事务的一生**（从"领记账员"到"入账/划掉"）。

---

# Level 1 数据访问抽象：JDBC 裸写为什么失败

## 为什么需要 DataSource

最原始的 JDBC 写法（Java 1.1 就有的 API）：

```java
Connection conn = DriverManager.getConnection(url, user, pass);
Statement st = conn.createStatement();
st.executeUpdate("insert into account ...");
conn.close();
```

这套写法的三个死穴：

1. **连接是一次性的**：每次 `DriverManager.getConnection` 都新建 TCP 连接 + 认证握手。数据库建连接是昂贵操作（网络往返 + 会话初始化），用完即焚 → 高并发下连接风暴，数据库先死。
2. **连接管理散落业务**：每个方法都要 try/finally 关连接，忘关 = 连接泄漏 = 数据库连接数耗尽。
3. **无法统一管控**：没有池化、没有超时、没有监控的挂点。

> **因果链**：连接昂贵 + 代码散乱 → 需要"连接工厂 + 池化 + 统一入口" → 标准答案 `javax.sql.DataSource`。

## DataSource 是什么

`DataSource`（javax.sql，规范）只承诺一件事：**给我一个 `Connection`**（`getConnection()`）。它是一个工厂，工厂内部怎么造（直连、池化、路由到多个库）对调用方完全不可见。

关键不变量：

- **应用面向 DataSource 编程，不面向具体连接池**。
- 连接池（HikariCP/DBCP）只是 DataSource 的一种实现；没有连接池时，"内嵌数据源"也是一种实现。
- 换数据库/换池 → 换实现类 → 业务代码一行不改。

## 自动配置的双分支：一个 jar 改变数据源类型（实测）

Spring Boot 的 `DataSourceAutoConfiguration` 用条件链裁决"给你装哪种 DataSource"（03 篇的条件家族在这里实战）：

```
DataSourceAutoConfiguration（3.3.5，实测行为）
├─ classpath 有连接池（HikariCP）→ PooledDataSourceConfiguration → HikariDataSource
└─ 无连接池，但有内嵌库（h2/derby/hsqldb）→ EmbeddedDataSourceConfiguration → 内嵌数据源
```

**双跑法**（与 04 篇 WebFlux 双跑法同一思想：classpath 决定自动配置分支）：

```
跑法 1：java -cp "out:$(find lib -name '*.jar' | tr '\n' ':')" demo12.ds.DataSourceApp
跑法 2：java -cp "out:$(find lib -name '*.jar' ! -name 'HikariCP*' | tr '\n' ':')" demo12.ds.DataSourceApp
```

真实输出（JDK 21.0.11 + spring-boot 3.3.5，本机）：

```
[数据源] com.zaxxer.hikari.HikariDataSource
[查询] count=0（H2 内存库，启动时为空）
```
```
[数据源] org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseFactory$EmbeddedDataSourceProxy
（内嵌数据源代理，JdbcTemplate 同样可用）
```

同一个应用、同一行 `DataSource ds = ctx.getBean(DataSource.class)`，**只删一个 jar，DataSource 类型就变了**——这是自动装配"classpath 即配置"的实证。

## 连接池归还语义（为什么要用"领用-归还"想问题）

- 池化后 `conn.close()` 不是真关闭，是**归还池子**（HikariProxy 的 close 语义）——所以 03 篇讲的 Bean 生命周期和这里不一样：**连接的生命周期由池管理，不由应用管理**。
- 事务与连接的关系：**一个事务内必须复用同一个连接**（同一笔账必须同一个记账员记，否则两边各记一半）。这个机制在 Level 2 展开。

**连接池是请求预算的一部分，不是无限资源**（24 章 03 章对照，Boot 4.1 基线文档语义）：

```text
请求 T 的耗时预算 = 排队等连接池 + 拿连接 + 执行 SQL + 归还
池耗尽 → 后续请求全部阻塞在"等待连接"（池等待进入请求 T 的预算）
```

**两个反直觉结论**（24 章 03 章文档语义）：
- **池不是越大越快**：连接是 DB 侧资源，池增大 → DB 侧连接/内存/上下文开销上升，超过 DB 处理能力反而更慢（且池上限超过并发需求时，多出来的连接只是占位）；
- **"连接池 timeout"通常是症状不是根因**：根因可能是长事务（REQUIRES_NEW 嵌套、事务里做远程调用）、锁等待、慢 SQL、外部 I/O、连接泄漏或下游放大——先查"谁占着连接不还"，再调池参数。
- **手工 `Connection` 绕过池与事务绑定**（24 章 03 章 P-TX02，文档语义）：自己 `dataSource.getConnection()` 直连，可能脱离当前线程绑定的事务（各写各的）并绕过池管理——这是"事务没生效"的一个隐蔽分支（Level 4 失效清单补充）。

---

# Level 2 事务抽象：PlatformTransactionManager 是什么

## 为什么需要事务管理器

数据库连接自带事务能力（JDBC 默认 autocommit=true，每条 SQL 独立提交）。要"两笔写入原子"，编程式写法是：

```java
conn.setAutoCommit(false);
try {
    insertA(conn);
    insertB(conn);
    conn.commit();
} catch (Exception e) {
    conn.rollback();
}
```

死穴与 Level 1 同构：

1. **事务控制代码散落业务方法**：每个需要原子性的方法都写一遍 try/catch/commit/rollback。
2. **事务与连接强耦合**：Spring 的事务抽象要解决"**开事务不碰连接**"——业务代码不知道、也不需要知道连接在哪。
3. **资源类型隔离**：JDBC 事务（Connection）、JPA 事务（EntityManager）、JMS 事务（Session）各有各的事务机制——需要统一入口。

> **因果链**：原子性需求 + 控制代码散落 + 资源类型多样 → 需要统一的事务抽象 → `PlatformTransactionManager`（策略接口）。

## PlatformTransactionManager 契约（Specification）

三个方法（Spring 事务抽象的全部核心）：

| 方法 | 语义 |
| ---- | ---- |
| `getTransaction(TransactionDefinition)` | 开启/加入一个事务，返回事务状态对象 |
| `commit(TransactionStatus)` | 提交（若判定回滚则实际回滚） |
| `rollback(TransactionStatus)` | 回滚 |

`TransactionDefinition` 描述一个事务的**属性**：隔离级别、传播行为、超时、只读。

JDBC 场景的实现：`DataSourceTransactionManager`（Spring 6 后为 `JdbcTransactionManager`，继承前者）。

## 线程绑定连接：一个事务为什么只有一个连接（Implementation）

同一个事务内所有 SQL 必须走同一个 Connection——Spring 的实现是**线程绑定**：

```
TransactionSynchronizationManager（ThreadLocal 持有）
├─ 事务开启：DataSourceUtils.getConnection → 取连接并标记"由事务管理"
├─ 后续 JdbcTemplate 调用：同一线程 → 同一个绑定连接（autocommit=false）
└─ 事务结束：commit/rollback + 归还连接（autocommit 复位）
```

这就是"一个事务 = 一个连接 = 一个线程"的底层来源，也是 Level 4 自调用失效的伏笔。

## 反编译实证：谁把事务管理器装进来的

`TransactionAutoConfiguration`（Boot 3.3.5，javap 反编译）结构：

```
@AutoConfiguration
@ConditionalOnClass(PlatformTransactionManager.class)
@EnableTransactionManagement   ← 嵌套 @Configuration 提供
  └─ 条件上：有 DataSourceTransactionManager/事务管理器 bean
      → 自动注册为 PlatformTransactionManager
```

注意嵌套结构：**`@EnableTransactionManagement` 是被一个嵌套 `@Configuration` 提供的**——这解释了为什么"没写 `@EnableTransactionManagement` 事务也能用"（Boot 默认开了），也解释了覆盖通道（03 篇）：自己定义 `PlatformTransactionManager` bean 即可替换默认。

实测（demo12.tx.TxBasicsApp 真实输出）：

```
[诊断] PlatformTransactionManager bean = org.springframework.jdbc.support.JdbcTransactionManager
[诊断] AccountService 是否代理 = true
```

两个关键事实：

1. 默认管理器是 **JdbcTransactionManager**（不是网上常说的 DataSourceTransactionManager——后者是它的父类，Spring 6.1 改名，行为不变。这是"命名变化，语义不变"的典型例子：代码组织变化 ≠ 语义变化）。
2. **AccountService 是代理**——声明式事务的第一块基石。

---

# Level 3 @Transactional 语义：回滚规则与传播行为

## 为什么是声明式

编程式（Level 2 的 try/catch 写法）与声明式（`@Transactional`）的取舍：

| 维度 | 编程式 | 声明式 |
| ---- | ---- | ---- |
| 控制粒度 | 任意（可精细控制提交点） | 方法级 |
| 代码侵入 | 业务代码全是事务样板 | 一个注解 |
| 可读性 | 事务逻辑淹没业务 | 事务意图一目了然 |
| 失效模式 | 显式调用，不会失效 | 代理失效（Level 4） |

声明式的本质：**AOP 代理在方法调用前开事务、返回后提交、抛异常回滚**——业务方法只写业务，事务是"环绕的透明层"（00 篇四层创建链的实战应用）。

## 回滚规则（Specification）：为什么默认只回滚运行时异常

规范语义：

- 未指定 `rollbackFor` 时，**只对 `RuntimeException` 和 `Error` 回滚**；检查异常（`Exception`）默认不回滚。
- 需要改变时显式 `rollbackFor = Exception.class`（或其类名版本 `rollbackForClassName`）。

**为什么是这个默认值**（这是最常见的面试坑，也是因果链的关键）：

- **检查异常 = 业务预期的错误**：如"余额不足"。业务可能希望**部分写入保留**（比如"下单失败但优惠券已扣"？不——更常见的例子：库存扣减成功但通知失败）——至少规范设计者的意图是：**检查异常表示"调用方预期要处理的错误"，事务不应擅自全部回滚**。
- **运行时异常 = 程序错误**：空指针、唯一键冲突、断言失败——这种错误下**没有任何写入是可信任的**，必须原子回滚。

> 注意：这是 **Specification 语义**（Javadoc 明确），不是某版本实现。

## 四场景实测（demo12.tx.TxBasicsApp）

测试方法：每个场景先插 2 行再抛异常（或不抛），最后查 count 对比。主键用 baseId 隔离场景，避免撞键。

真实输出（JDK 21.0.11 + spring-boot 3.3.5 + H2 内存库，本机）：

```
[场景1 正常运行] insert 2 行 → count=2（提交）
[场景2 运行时异常] insert 2 行后抛 RuntimeException → count=2（回滚，只留场景1数据）
[场景3 检查异常] main 捕获到: java.lang.Exception → count=4（默认不回滚，数据保留）
[场景4 rollbackFor] insert 2 行后抛 Exception → count=4（rollbackFor 生效，回滚）
```

配套打点（事务内）：

```
[事务] insertTwo 事务激活=true
[连接] DataSource 类=com.zaxxer.hikari.HikariDataSource
[同步] afterCompletion=COMMITTED（提交）     ← 场景1/3
[同步] afterCompletion=ROLLED_BACK（回滚）   ← 场景2/4
```

证据链完整：事务激活（`TransactionSynchronizationManager.isActualTransactionActive()`）→ 连接确实是 Hikari 的 → `TransactionSynchronization.afterCompletion` 报告最终状态。**count 与 afterCompletion 相互印证**：COMMITTED 时数据留下，ROLLED_BACK 时数据消失。

## 意外发现（开发测试时的真实踩坑）：唯一键冲突会把事务整个卷走

写测试时用固定主键 1/2，场景 1 提交后，场景 2 再插同主键 → 抛 `DuplicateKeyException` → **整个事务回滚**，与"业务逻辑"无关。

```
唯一键冲突（DuplicateKeyException）
  └─ DataAccessException 体系 = RuntimeException（spring-dao 统一转换）
      └─ 默认回滚规则命中 → 事务回滚
```

**生产含义**：数据库唯一约束是最后防线——**线上"并发下唯一键冲突"不仅那一条 insert 失败，同事务里所有已写数据全部回滚**。这不是 Bug，是运行时异常语义的必然结果（Level 4 讲完代理机制会回看这个因果链）。

**更隐蔽的变体：catch 住异常后，外层 commit 时抛 UnexpectedRollbackException**（机制描述来自 24 章 03 章，Boot 4.1 基线文档语义——本机未实测，标注待验证）：

```text
Outer REQUIRED 开启事务 P1
  → Inner REQUIRED 加入 P1
  → Inner 失败 → 内层把 P1 标记 rollback-only（"已判定死刑"）
  → Inner 的 catch 把异常吞掉，正常返回
  → Outer 不知道出过事，继续业务 → commit
  → commit 发现 P1 已被标记 rollback-only，不能提交
  → 抛 UnexpectedRollbackException（而不是"成功提交"）
```

**因果**：rollback-only 是事务级标记，一旦被内层标记，外层 catch 只能吞掉异常、吞不掉标记——**commit 时框架拒绝"明知已回滚却报告成功"**，宁可抛 UnexpectedRollbackException 让外层感知。修复方向：内层异常要么继续向外抛（让外层也走回滚）、要么内层用 REQUIRES_NEW 独立事务（03 章文档语义）——"catch 掉就没事了"是这条链上最贵的误判（本块与 503 行同源：24 章 03 章文档语义，本机未实测，标注待验证）。

## 传播行为：REQUIRED vs REQUIRES_NEW 实测（demo12.tx.PropagationApp）

`TransactionDefinition.PROPAGATION_*` 定义"**事务方法调事务方法时，事务怎么协同**"。两个最重要的：

| 传播 | 语义 | 典型场景 |
| ---- | ---- | ---- |
| REQUIRED（默认） | 外层有事务则**加入**；没有则新建 | 绝大多数业务：同一次操作同进退 |
| REQUIRES_NEW | **挂起**外层事务，新建独立事务；外层回滚不影响它 | 审计日志、记账流水：**无论如何都要落库** |

实测场景：外层方法（`@Transactional` REQUIRED，由 main 直接调）抛运行时异常，内层分别用 REQUIRED / REQUIRES_NEW 写账本（ledger 表）——外层有事务，"加入 vs 挂起"才有可比性。

真实输出（JDK 21.0.11 + spring-boot 3.3.5 + H2 内存库，本机）：

```
[场景A REQUIRED] 外层调内层（REQUIRED）→ 同一事务，外层抛运行时异常 → 内层 count=0（内层写入被外层回滚一起撤销）
[场景B REQUIRES_NEW] 外层调内层（REQUIRES_NEW）→ 独立事务，外层抛运行时异常 → 内层 count=1（内层已独立提交，保留）
```

同一种"外层回滚"，两种传播两种结局：

- REQUIRED：**同生共死**——内层写入被一起回滚（count=0）。
- REQUIRES_NEW：**独立成家**——内层已独立提交，外层的灾难不影响它（count=1）。

**生产要点**：记账流水/审计日志必须 REQUIRES_NEW，否则"主流程失败 → 日志也没了"——事故复盘时连记录都没有。代价：挂起/恢复外层事务（连接切换），多一次提交，连接占用时间变长（Level 6 会讲长事务）。

## 三层模型：逻辑、物理、资源（24 章 03 章对照，Boot 4.1 基线文档语义，与本文实测机制一致）

| 层 | 是什么 | 边界 |
| ---- | ---- | ---- |
| 逻辑事务 | 应用方法声明的业务边界（@Transactional 标记） | Spring 的代理层 |
| 物理事务 | 数据库/资源真正开启的事务 | 数据库层 |
| 资源事务 | Connection/EntityManager 的实际绑定与释放 | 连接层（Level 2 的线程绑定） |

**因果**：REQUIRED 加入 = 多个逻辑事务共享一个物理事务（多 DAO 同进退）；REQUIRES_NEW = 新逻辑事务 = **新物理事务 = 多占一条连接**（挂起期间外层连接也还握着）——**嵌套 REQUIRES_NEW 是连接压力的放大器**（"所有嵌套都独立提交 → 额外连接、锁和池耗尽"）。NESTED（Savepoint）是第三条路：同一物理事务内开 Savepoint，内层回滚只回到 Savepoint（物理事务不换、连接不增）——但 Savepoint 支持依赖数据库（24 章 03 章文档语义，本机未实测，标注待验证）。

---

# Level 4 代理机制：声明式事务怎么生效，怎么失效

## 生效机制：代理是唯一的入口

`@Transactional` 不是魔法，是 **AOP 代理**（00 篇四层创建链产物）：

```
调用方 → UserService 代理（CGLIB）── 事务拦截器 ──→ 目标对象
                    │                      │
                    │                1. 开事务（getTransaction）
                    │                2. 执行目标方法
                    │                3. 正常 → commit / 异常 → rollback
```

关键不变量：**只有"从代理对象发出的调用"才被拦截**。调用方手里的 bean 是代理；代理内部的 `this` 是目标对象。

## 自调用失效：静默的坑（实测）

`this.method()` 自调用 = **直接调目标对象方法，绕过代理** → 事务拦截器不执行 → 注解形同虚设。

```
public class UserService {
    @Transactional
    public void saveAndThrow(int id) { ... 内部抛异常 ... }

    public void selfCallFailure(int id) {
        this.saveAndThrow(id);   // 自调用 → 绕过代理 → 事务不生效
    }
}
```

**为什么是"静默失效"而不是报错**：代理是运行时装配的，方法照样执行、数据照样写入——只是没有事务环绕。**没有任何异常提示**，只能通过"回滚是否发生"来发现。

真实输出（demo12.tx.SelfInvocationApp，失败版）：

```
[打点] transactionalSaveAndThrow 事务激活=false   ← 自调用：事务没开！
[失败版 this.save()] 自调用 @Transactional 方法（内部抛异常）→ count=1（事务没生效！数据残留）
```

`isActualTransactionActive()=false` 实锤：异常抛了，但没有事务可回滚，数据残留。

## 修复：四种拿到代理的方式（实测）

**修复本质**：让"事务方法调用"显式经过代理对象。代理就在容器里——`ctx.getBean(UserService.class)` 取出来的**本来就是代理**。所以除了 `@Autowired @Lazy self` 注入自身代理，还可以直接从容器取，或在代理调用链内取当前代理。

**方式 1：@Autowired @Lazy 自注入**（最常用，字段注入代理）

```java
@Autowired @Lazy private UserService self;

public void selfCallFixed(int id) {
    self.saveAndThrow(id);   // 走代理 → 事务生效
}
```

为什么 `@Lazy`：自注入是循环依赖（创建自己需要自己），`@Lazy` 让 Spring 先注入一个代理占位，实际解析推迟到第一次使用——绕开循环依赖的过早创建问题（00 篇创建链相关）。

**方式 2：context.getBean(Class) 从容器取代理**（本次补充实测）

```java
@Autowired private ApplicationContext context;

public void selfCallFixedByContext(int id) {
    UserService proxy = context.getBean(UserService.class);  // 容器里的 bean 就是代理
    proxy.saveAndThrow(id);   // 走代理 → 事务生效
}
```

**为什么容器取出的就是代理**：声明式事务 = bean 创建后套 AOP 代理，容器对外暴露的永远是代理。getBean 只是"取"它，不存在"取到裸对象"的可能（除非显式 AopUtils.getTargetClass 解包）。

**方式 3：@EnableAspectJAutoProxy(exposeProxy=true) + AopContext.currentProxy()**（本次补充实测）

```java
@SpringBootApplication
@EnableAspectJAutoProxy(exposeProxy = true)   // 开启代理曝光
static class BootConfig { }

public void selfCallFixedByAopContext(int id) {
    UserService proxy = (UserService) AopContext.currentProxy();  // 取"当前链上的代理"
    proxy.saveAndThrow(id);   // 走代理 → 事务生效
}
```

机制（spring-aop 6.1.14 反编译实证）：

```text
@EnableAspectJAutoProxy(exposeProxy=true)
  └─ AspectJAutoProxyRegistrar → 读属性 "exposeProxy"
       └─ AopConfigUtils.forceAutoProxyCreatorToExposeProxy → 设置到 auto proxy creator
            └─ CglibAopProxy$DynamicAdvisedInterceptor.intercept：
                 exposeProxy=true → AopContext.setCurrentProxy(proxy)（调用进入时）
                 finally → 恢复旧值（正常/异常双路径都有，嵌套代理调用语义正确）
                      └─ AopContext.currentProxy() 读取（ThreadLocal）
```

**边界（实测）**：`currentProxy()` 只能在**代理调用链内**调用——从代理进入的方法里取得到；链外（栈上没有代理调用）直接抛 `IllegalStateException: Cannot find current proxy...`。实测输出：

```
[链外 AopContext.currentProxy()] 抛 IllegalStateException：Cannot find current proxy: Set 'exposeProxy' property
on Advised to 'true' to make it available, and ensure that AopContext.currentProxy() is invoked in the same
thread as the AOP invocation context.
```

**方式 4：跨 bean 调用**（事务方法放另一个 bean，天然走代理）——最"正统"的结构，无任何技巧，但要求事务方法独立成 bean（本文 PropagationApp 的 REQUIRES_NEW 就是跨 bean 修复）。

真实输出（修复版 + 诊断，本次补充）：

```
[诊断] 目标实例内 this.self 类 = demo12.tx.SelfInvocationApp$UserService$$SpringCGLIB$$0；AOP 代理 = true
[打点] transactionalSaveAndThrow 事务激活=true    ← 走代理：事务开了！
[修复版 self.save()] 注入代理调用（内部抛异常）→ count=1（回滚生效，失败版数据仍在）
[诊断] context.getBean 类 = demo12.tx.SelfInvocationApp$UserService$$SpringCGLIB$$1；AOP 代理 = true
[打点] transactionalSaveAndThrow 事务激活=true    ← 容器代理：事务开了！
[修复版 ctx.getBean()] 容器取代理调用（内部抛异常）→ count=1（回滚生效）
[诊断] AopContext.currentProxy 类 = demo12.tx.SelfInvocationApp$UserService$$SpringCGLIB$$1；AOP 代理 = true
[打点] transactionalSaveAndThrow 事务激活=true    ← 当前链上代理：事务开了！
[修复版 AopContext] 调用链内取当前代理（内部抛异常）→ count=1（回滚生效）
[断言] ctx.getBean 两次 == : true（单例：容器每次返回同一代理实例）
[断言] 方法内 this 类 = demo12.tx.SelfInvocationApp$UserService（纯目标类：方法在 target 上执行，字段注入就在它身上）
[断言] 返回 this 却被换成代理 : true（processReturnType：retVal==target 时替换为 proxy——调用方拿到的仍是代理，AOP 能力不丢）
[断言] 代理运行时类型 != 目标类 : true（proxy.getClass() = SelfInvocationApp$UserService$$SpringCGLIB$$1，原始类型只靠 AopUtils.getTargetClass 获取 = UserService）
```

四个关键事实：

1. **三种方式拿到的都是 CGLIB 代理**，isAopProxy=true——`@Lazy` 自注入是 `$$0`（延迟解析包装代理，注入阶段生成）；`context.getBean` 与 `AopContext.currentProxy` 都是 `$$1`（容器单例事务代理，初始化完成后 BPP 包装）——**AopContext 返回的就是当前调用链上的代理实例**（从 `$$1` 代理进入的方法，currentProxy 就是 `$$1`），与 getBean 是同一个对象；都拦截调用、都让事务生效——**修复的是"调用路径"，不是"代理身份"**；
2. **事务激活从 false 变 true**——同一段业务代码，绕不绕代理，天壤之别。
3. **代理 = 独立 target + 返回值替换（断言实验 + UnwrapApp + 字节码三重实证）**：`ctx.getBean` 两次是**同一对象**（单例缓存）；代理 `$$1` 内部持有独立的 **target**（纯目标类实例，`getTargetSource().getTarget()` 可解包）——**方法调用被拦截器转发到 target 上执行**（方法内 `this` 类 = 纯目标类，字段注入发生在 target 上，代理实例自身字段为 null，UnwrapApp 实证）；代理与目标类的差别在运行时类型（`proxy.getClass()` = `$$SpringCGLIB$$` 子类 vs `AopUtils.getTargetClass()`）。
4. **"返回自身"会被换成代理（新知识点，spring-aop 6.1.14 字节码实证）**：`CglibAopProxy.intercept` 返回前调用 `processReturnType(proxy, target, method, args, retVal)`——**当 retVal != null 且 retVal == target 时，替换为 proxy**（除非方法声明类实现 `RawTargetAccess`）。所以 `rawSelf()`（`return this`）在方法内 `this` 是 target，调用方拿到的却是代理——**Spring 保证"返回自身"的方法，返回值仍然具备 AOP 能力**。这解释了为什么"目标实例内 `this.self` 是 `$$0` 懒代理"而 `this` 本身却是纯目标类：字段注入在 target 上、方法执行也在 target 上，二者并不矛盾。

**四种方式对比（Trade-off）**：

| 方式 | 机制 | 适用 | 代价 |
| ---- | ---- | ---- | ---- |
| @Lazy self 自注入 | 注入阶段生成包装代理 | 自调用修复最常用 | 循环依赖技巧，字段注入风格 |
| context.getBean | 单例缓存直接取 | 偶发使用、工具方法 | 需要注入 ApplicationContext（贫血封装） |
| exposeProxy + currentProxy | ThreadLocal 曝光当前代理 | 深层自调用链（A→B→A 想拿代理） | 全局开关 exposeProxy=true（所有代理调用多一次 set/还原）；链外调用抛异常；强转代码可读性差 |
| 跨 bean 调用 | 事务方法独立 bean | 结构最正统 | 要拆类/拆方法，重构成本 |

生产建议：**首选方式 4（跨 bean）**——结构干净无技巧；方式 1 次之（改造成本低）；方式 3 的 exposeProxy 是全局开关，为修一个自调用开全局 ThreadLocal 曝光，收益/代价不成比例，**不推荐日常使用**，仅当"调用链内取代理"有真实需求（如 A 方法内需要把代理传给工具类）时才开。

> ⚠️ 开发期插曲（实证后保留原解释）：早期在 main 里写 `svc.self.getClass()` 触发 **NPE**（`Cannot invoke "Object.getClass()" because "<local3>.self" is null`）——**`svc` 是从容器拿到的代理，代理实例自身的注入字段是 null**（字段注入只发生在 target 上，UnwrapApp 实证），所以 `svc.self` 直接 NPE。**注意不要被"方法内 `this.self` 正常"误导**：方法内 `this` 是 target（方法在 target 上执行），target 的字段当然齐全；代理实例的字段才是 null。业务方法内访问注入引用用 `this` 没问题（this = target），但从容器拿代理后反射/访问代理实例字段是空——这是"代理 ≠ 目标"的运行时证据，与关键事实 3/4 完全自洽。

## 失效场景清单（排查地图）

| 场景 | 症状 | 判定 |
| ---- | ---- | ---- |
| `this` 自调用 | 事务激活=false，数据残留 | 实测（本文） |
| 同一类内调同类的另一 @Transactional 方法 | 同上 | 同一机制 |
| 在另一个非代理对象里调 | 该对象的方法没走代理 | 依赖注入的是目标还是代理 |
| 事务方法非 public（规范要求） | 行为不符合预期 | Specification 边界，未实测破坏性场景（待验证） |
| 异常被 catch 在方法内部 | 事务方法"正常返回"→ 提交 | 违反"异常外抛才回滚" |
| 检查异常未配 rollbackFor | 数据保留 | Level 3 场景 3 实测 |

## 回看：唯一键冲突因果链闭环

Level 3 的"唯一键冲突卷走整个事务"，现在能完整推导了：

```
insert 撞唯一键
  → 连接层抛 SQLException
  → JdbcTemplate 统一转成 DataAccessException（运行时异常）
  → 异常穿过目标方法 → 事务拦截器捕获
  → 默认回滚规则命中（RuntimeException）
  → rollback 整个事务
```

**不是"数据库约束导致回滚"，而是"运行时异常语义导致回滚"**——数据库只是触发者。

---

# Level 5 数据层整合全景：自动配置的三条线

## 三条自动配置线（Boot 3.3.5，均为条件链裁决）

```
        DataSourceAutoConfiguration        DataSourceTransactionManagerAutoConfiguration    JdbcTemplateAutoConfiguration
            │ 条件链：连接池/内嵌库               │ 条件：classpath 有 DataSourceTransactionManager    │ 条件：单 DataSource + JdbcTemplate 类
            ▼                                   ▼                                            ▼
        DataSource bean  ──────────────────►  PlatformTransactionManager bean  ──────────►  JdbcTemplate bean
            （Hikari / 内嵌）                     （JdbcTransactionManager）                    （自动配置的模板）
```

一条因果链串起全文三样东西：

1. **DataSource**（Level 1）：池化连接工厂——账房的记账员们。
2. **PlatformTransactionManager**（Level 2）：事务开关——"这笔账怎么记"的规则制定者。
3. **JdbcTemplate**：SQL 执行入口——业务方法里的"记一笔"动作。它的每次 execute 都从 DataSource 取连接；在事务内时拿到的是事务绑定的那个连接。

## 参数与默认行为（Implementation）

- 什么都不配 → H2 内存库（`spring.datasource.url` 未设时由 EmbeddedDatabase 约定生成，实测：`[查询] count=0` 可用）。
- `spring.datasource.url/username/password` 显式配 MySQL/PostgreSQL 等。
- 多数据源：`DataSourceAutoConfiguration` 的 `@ConditionalOnSingleCandidate` 语义——**只有一个 DataSource 候选时才自动配 JdbcTemplate/事务管理器**；多数据源必须自己定义（本文不展开，标注方向）。

## 事务属性配置入口（Specification）

| 属性 | 作用 | 默认 |
| ---- | ---- | ---- |
| `@Transactional(rollbackFor=...)` | 回滚规则 | 仅运行时异常/Error |
| `@Transactional(propagation=...)` | 传播 | REQUIRED |
| `@Transactional(isolation=...)` | 隔离级别 | 数据库默认（本实验未实测隔离行为，待验证） |
| `@Transactional(timeout=...)` | 事务超时 | 无（数据库默认） |
| `@Transactional(readOnly=...)` | 只读提示 | false |

**readOnly=true 的精确语义（2026-08-07 讨论定稿）**：

```text
@Transactional(readOnly = true)
  └─ 解析成 TransactionDefinition.isReadOnly() = true
       └─ 事务管理器 doBegin 时读这个位，做具体动作：

路径 1：JDBC（DataSourceTransactionManager，默认）
  └─ DataSourceUtils.prepareConnectionForTransaction(con, definition)
       └─ 字节码实证（spring-jdbc 6.1.14）：isReadOnly()==true
          → connection.setReadOnly(true)（真实 JDBC 调用）
       └─ 之后的语义交给数据库：
            ├─ MySQL InnoDB：连接标记只读 → 只读事务优化（待实测）
            ├─ PostgreSQL 等：事务真正只读 → 事务内写被数据库拒绝（待实测）
            └─ 不支持的驱动：静默忽略

路径 2：JPA/Hibernate（JpaTransactionManager）
  └─ 修改 Hibernate 的 flush 模式（查询前不自动 flush 脏数据）
     → readOnly 最实在的性能动作（本机无 spring-orm，待验证）
```

**readOnly 只在"新开事务"时生效（传播与属性的交互，易错点）**：REQUIRED 传播下，方法已在事务中时（写方法内、或被 @Transactional 方法调用），`readOnly=true` **加入既有事务，标签不改变既有事务**（事务属性以第一个事务为准）——标了也白标。**判定条件是两个：多步读 + 无事务环境**，不是"多步读"一个。

```text
读操作加不加事务的决策表：
  单查询读（selectById、简单列表）        → 不加（单条 SELECT 本身原子，连接用完即还）
  多步读 + 无事务环境（详情页多次查询）  → @Transactional(readOnly=true)
       （两次查询若无事务=两次独立连接/快照，并发写下结果可能不一致）
  多步读 + 已在事务里                    → 不加（加了也不生效，沿用既有事务）
  写 / 读写混合                          → @Transactional（不能用 readOnly=true，
                                             路由中间件可能让你走从库）
```

**readOnly 不做什么**：Spring 默认不拦截写（JDBC 路径下事务内 UPDATE 照样提交，想拦写靠数据库拒绝或 `enforceReadOnly=true`，机制待验证）；事务照常开启（连接、隔离级别、传播不变）。**生产上 readOnly 的最大用途是读写分离路由**（路由中间件读 `isReadOnly()` 判断主/从库）——声明只读却偷偷写 = 主从不一致风险。

---

# Level 6 生产实践：失效排查 / 长事务 / 边界

## 1. 事务失效排查清单（遇到"没回滚"按此走）

```
回滚没生效？
├─ 1. 异常真的外抛了吗？被 catch 在方法内 → 提交（最常见）
├─ 2. 异常类型命中默认规则吗？检查异常没配 rollbackFor → 不回滚
├─ 3. 调用真的经过代理吗？this 自调用 → 失效（本文实测）
├─ 4. 方法可见性是 public 吗？（规范要求，破坏性场景未实测）
└─ 5. 同类内 this 调 @Transactional 方法 → 用 self 代理（本文实测修复）
```

## 2. 长事务：最大的隐性成本

事务 = 连接占用 + 行锁持有。事务越长：

- 连接池被占用的连接越多 → 池耗尽 → 其他线程排队（线程与连接双重资源竞争）。
- 行锁持有越久 → 并发写入阻塞 → 锁等待超时。

**生产铁律**：

- 事务方法只做"原子性必需的写入"，**查询/计算/外部调用放事务外**（外部 HTTP 调用塞进事务 = 经典事故：事务里等外部接口，接口又慢，连接+锁全卡住）。
- 一笔事务的边界 = 一次"要么全成要么全不成"的业务动作的边界，**不是方法的边界**。

## 3. 唯一键冲突与重试

并发写同一主键/唯一键 → 一个成功一个 `DuplicateKeyException` → 整个事务回滚（Level 3 实测）。生产应对：

- 幂等键（业务唯一键而非自增主键）设计。
- 冲突时重试整个事务（幂等写入可重试；非幂等写入重试前必须确认前一事务确实回滚了——**回滚未确认就重试 = 重复扣款**）。
- 绝不 catch 掉 DuplicateKeyException 继续往下走（事务已标记回滚，继续写只会越写越乱——Spring 事务已标记 rollback-only，后续 commit 会抛 UnexpectedRollbackException；机制描述见 Level 3 交叉补充块：24 章 03 章文档语义，本机未实测，标注待验证）。

## 4. 分布式边界：@Transactional 管不到的地方

### 机制根源：本地事务的原子性是"单连接"的原子性

```text
本地事务：TransactionManager → 一个 DataSource → 一个 Connection → 一个数据库会话
  原子性 = BEGIN/COMMIT/ROLLBACK 发给同一个数据库实例

跨边界时：两个库 = 两个连接 = 两套事务 = 没有统一协调者
  → A 提交成功、B 提交失败时，A 不知道 B 失败，无法撤销自己
```

一句话：`@Transactional` 管不到跨边界，本质是**缺少统一协调者**，不是缺少注解。

### 边界 1：跨库（含分库分表）

**先澄清一个易混淆点：分表 ≠ 分库，破坏事务的是"分库"**：

```text
分表（同库同实例，order_0/order_1 在同一 MySQL 实例）
  → 所有 SQL 仍走同一 DataSource/连接 → @Transactional 依然有效

分库（多个实例/DataSource，user_db + order_db）
  → SQL 落在多个连接 → 本地事务各自为政 → 边界出现
```

分库分表制造的典型场景：

- **跨分片写**：按 user_id 分片，一笔转账同时更新分片 1 的 A 和分片 2 的 B——两个分片各自本地事务，A 成功 B 失败无法回滚；
- **无路由键查询**：`WHERE status='PAID'` 无路由键 → 广播全分片再汇总——这是性能问题不是事务问题，但会放大事务问题；
- **分片后的全局主键**：自增 ID 在分片下重复 → 雪花算法等全局 ID（连带约束，与事务无直接关系）。

**分库分表下的第一优先解：路由键设计 = 事务边界设计**——让"一个业务事务"的所有 SQL 落在同一分片内（订单+明细+日志全按 order_id 分片）→ 事务仍落单库单连接 → 本地事务解决，不需要任何分布式事务。判定口诀：**路由键选得对，大多数事务留在单片内；选得散，每个事务都是跨库事务**。

**真跨片时的强一致方案（XA 两阶段提交）及代价**：

```text
prepare 阶段：所有参与者锁定资源、就绪 → commit 阶段：全部提交
致命点：prepare 后某参与者崩溃 → 协调者无法决定提交/回滚
       → 悬挂事务（资源锁死不释放），需人工/恢复流程干预
Trade-off：锁持有时间 = 最慢参与者 + 协调开销；参与者越多越慢
```

所以互联网公司普遍**放弃强一致**，转向 Saga/补偿（每个库本地事务 + 失败反向补偿，核心依赖幂等）或最终一致性。

### 边界 2：跨服务（HTTP/gRPC 调对方写库）

**关键认知：跨服务不是 XA 能解决的**——XA 要求所有参与者暴露事务资源接口给同一个全局事务管理器，HTTP 服务不是 XA 资源。方案：

```text
Saga（链式）：A 本地事务成功 → 调 B → B 失败 → 沿链路反向补偿
TCC：try（预留）→ confirm（确认）→ cancel（撤销）
     （库存 try 预占、confirm 扣减、cancel 释放）
```

架构师视角：跨服务一致性是**业务建模问题**——先问"业务允许多短暂的不一致？"（必须立即 = 设计上收敛写点；秒级/分钟级 = Saga/Outbox）。

### 边界 3：数据库 + 消息队列（写库 + 发 MQ）

经典死循环：先发消息后写库 → 消息发出、库回滚 → 消费者处理了不存在的数据；先写库后发消息 → 库提交、消息发送失败 → 消费者永远不知道。

Spring 的半程方案（02 篇已实测）：`@TransactionalEventListener(AFTER_COMMIT)`——**提交后再回调发消息**，解决"回滚后消息已发"的错位。但注意它的边界：

```text
AFTER_COMMIT 解决：消息发出的时机（提交后）
AFTER_COMMIT 不解决：消息发出的成功（回调里 MQ 挂了 → 消息丢了）
→ 生产必须补：重试（本地重试队列）或 Outbox 表轮询/CDC
```

**Outbox 是最终一致性的事实标准**（Microservices.io 模式）：业务写 + outbox 写**同一本地事务** → 后台任务发 MQ → 消费端幂等。它赢的根本原因：**把"跨边界"问题改写成"单库本地事务 + 异步对账"问题，不需要全局协调者**。

### 决策框架：三个问题按顺序问

```text
1. 能避免吗？           → 单库 + 聚合设计 / 路由键收敛（最好的分布式事务=没有分布式事务）
2. 不一致窗口能接受吗？  → 能 → 最终一致性（Outbox / Saga / 事务消息）
                          不能 → 往下
3. 必须强一致且参与者少 → XA/JTA（金融强一致场景，接受悬挂恢复代价）
```

风险提示：XA 保证一致性，但把**锁持有时间、悬挂风险、运维复杂度**作为代价；版本/驱动支持参差（MySQL 的 XA 行为、ShardingSphere 的 XA/本地消息表集成、Seata 等中间件**均未实测，待验证**，本文不承诺其行为）。

### Transfer：事务 vs 一致性是两个层次

```
单点状态原子变化 = 数据库事务（MVCC 快照、锁）——本文主题
多点状态收敛一致 = 分布式一致性（Raft/Paxos、消息系统 exactly-once）
Kafka 事务性 producer = "broker 内事务 + 幂等"的组合——同一个"尽量收敛边界"思路
```

方法论：任何分布式一致性问题，第一步不是选框架，是**画边界**——把边界收敛到单库能解决的地方（Outbox 就是把"消息"也搬进单库事务），剩下的才是协调者的事。

## 5. 测试方法论：怎么验证回滚真的发生

用 count 断言法（本文所有 demo 就是这个模式）：

```
before = count()
执行事务方法（预期抛异常）
after = count()
断言 after == before          ← 回滚生效
断言 after == before + N      ← 回滚失效（自调用/规则不对）
```

**没有这个断言的事务测试，等于没测**——因为自调用失效是静默的（Level 4 实测）。

---

# 全篇因果链总图

```
连接昂贵且散乱
  → DataSource 抽象 + 连接池（Level 1：双分支实测）
      → 事务控制散落 + 多资源类型
          → PlatformTransactionManager 抽象（Level 2：JdbcTransactionManager 实测）
              → 业务代码不想写事务样板
                  → 声明式 @Transactional（Level 3：四场景 + 双传播实测）
                      → 声明式依赖代理
                          → 自调用绕过代理 = 静默失效（Level 4：实测 + @Lazy 修复）
                              → 生产：排查清单 / 长事务 / 唯一键冲突 / 分布式边界（Level 6）
```

---

# 线上案例（3 个）

## 案例 1：扣款成功但回滚了——唯一键冲突吞数据

症状：下单接口偶发"扣了款但订单没生成"。

排查：唯一键（订单号）并发撞车 → `DuplicateKeyException`（运行时异常）→ **同事务里已执行的扣款回滚**。用户看到"钱扣了"是因为扣款先行、订单插入撞键，最终整笔回滚——但**中间态被对账系统捕获**，误报扣款。

结论：事务里先做可能撞键的写入（或先查后插 + 唯一键兜底），把"业务校验"和"原子性兜底"分开认知；对账误报要在重试后二次核对（Level 6.3）。

## 案例 2：审计日志总是不落库——REQUIRES_NEW 没生效

症状：主流程失败时审计日志丢失，复盘没数据。

初查：日志方法标了 `@Transactional(propagation = REQUIRES_NEW)`，看起来没问题。

真相：**日志方法是在同类内 `this.xxx()` 调用的**（自调用，Level 4 实测的失效模式）——REQUIRES_NEW 根本没机会生效，反而因为"加入了外层事务"，外层回滚把日志一起卷走。

结论：REQUIRES_NEW 方法必须**跨 bean 调用**或自注入代理（本文 PropagationApp 开发期真实踩坑：`this.innerRequiresNew()` 实测加入外层事务，重构为跨 bean 后 REQUIRES_NEW 才生效——对应输出 [场景B] count=1）。

## 案例 3：回滚验证测试全绿，上线还是出事——检查异常默认不回滚

症状：测试"事务回滚"全过，生产里业务异常下数据仍保留。

真相：测试用 `RuntimeException` 断言回滚（全绿），生产业务抛的是**自定义检查异常**——默认规则不回滚（Level 3 场景 3 实测：`[场景3 检查异常] count=4（默认不回滚）`）。

结论：**回滚规则测试必须覆盖真实异常类型**——凡是"业务可预期失败"统一 `rollbackFor = 具体异常.class`，并写一条该异常的 count 断言（Level 6.5）。

---

# 面试自查表

| 问题 | 答案要点 | 证据 |
| ---- | ---- | ---- |
| @Transactional 默认回滚什么？为什么？ | RuntimeException/Error；检查异常=业务预期，运行时=程序错误 | Specification（Level 3） |
| 检查异常想回滚怎么办？ | rollbackFor=Exception.class 或 rollbackForClassName | 场景 4 实测 |
| REQUIRED 和 REQUIRES_NEW 区别？ | 加入 vs 挂起新建；外层回滚是否波及 | PropagationApp 实测 |
| 什么场景必须 REQUIRES_NEW？ | 审计日志/流水：无论如何都要落库 | Level 3 |
| 自调用为什么失效？ | 代理只拦"从代理发出的调用"，this 直达目标对象 | Level 4 实测（激活=false） |
| 怎么修复自调用？ | 让调用显式走代理：跨 bean / @Lazy 自注入 / context.getBean / exposeProxy+AopContext（四种实测，首选跨 bean） | Level 4 实测（激活=true，四种方式实测） |
| 一个事务为什么只有一个连接？ | TransactionSynchronizationManager 线程绑定 | Level 2 |
| 唯一键冲突为什么会卷走整个事务？ | DataAccessException=运行时异常→默认回滚 | Level 3 意外发现 |
| 事务里能调外部 HTTP 吗？ | 不能：长事务=连接+锁双重占用 | Level 6.2 |
| @Transactional 能管分布式事务吗？ | 不能：只覆盖一个连接；跨边界用事务消息/最终一致性 | Level 6.4 |
| 为什么不直接用 XA 全覆盖？ | prepare 后崩溃=悬挂事务、锁持有时间长、参与者越多越慢；跨服务根本不可用（HTTP 不是 XA 资源） | Level 6.4 |
| 分表会破坏事务吗？ | 不会：同库同连接本地事务照常；破坏事务的是分库 | Level 6.4 |

---

# 坑与细节（8 个）

## 坑 1：字段注入只填目标实例，不填代理实例

从容器拿到的 bean 是代理，`svc.self` 是 null（代理实例上注入字段为空）——NPE 实测：`Cannot invoke "Object.getClass()" because "<local3>.self" is null`。诊断/打点要用 `this.self`（目标实例内部视角）。

## 坑 2：自调用失效没有报错，只有数据对不上

代理是运行时装配，方法照常执行——静默失效只能靠 count 断言（Level 6.5）发现。**所有事务方法调用点都要问一句：这是代理调用吗？**

## 坑 3：唯一键冲突 = 运行时异常 = 整个事务回滚

`DuplicateKeyException`（DataAccessException 体系）命中默认回滚规则——不是"只回滚那一条 insert"。写测试用固定主键时 100% 会撞上（本文开发期实测）。

## 坑 4：检查异常默认不回滚

业务里抛 `Exception`（自定义检查异常），默认提交——`rollbackFor` 没配，回滚测试就要按真实异常类型写（案例 3）。

## 坑 5：异常被 catch 在事务方法内部 = 提交

事务拦截器只看"方法返回 vs 抛异常"。方法内 catch 掉异常正常返回 → 提交。想回滚就让它往外抛，或在 catch 里 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`（未实测，标注）。

## 坑 6：REQUIRES_NEW 用 this 调 = 白写

自调用下传播注解不生效，反而加入外层事务被卷走（PropagationApp 开发期真实踩坑）。必须跨 bean 或 self 代理调用。

## 坑 7：slf4j-api 缺失 = 连接池直接 ClassNotFoundException

HikariCP 依赖 slf4j-api（2.0.16）；缺了抛 `org.slf4j.LoggerFactory` 找不到。无 provider 时只有 NOP 警告，无害但看不到 Hikari 日志。

## 坑 8：SERVLET 模式 main 异常崩溃，JVM 不退

main 里抛未捕获异常且没执行 `ctx.close()` → Tomcat 线程保活 → JVM 挂住（实验期真实事故：NPE 后进程不退出、端口占用、后续启动超时）。**排查"应用退不掉"先看 main 是否安全退出**；脚本里用 `pkill -f 类名` 清理。

---

# 版本勘误表

| 说法 | 正确版本（本文实证） |
| ---- | ---- |
| "默认事务管理器是 DataSourceTransactionManager" | Spring 6.1 起 bean 是 **JdbcTransactionManager**（继承 DataSourceTransactionManager，行为兼容；命名变化 ≠ 语义变化） |
| "@EnableTransactionManagement 要自己写" | Boot 3.3.5 由 TransactionAutoConfiguration 嵌套 @Configuration 自动提供（javap 实证） |
| "H2 无配置不可用" | 实测可用：Hikari 分支 `[查询] count=0` |
| "事务内连接是 autocommit 的" | 事务内连接由事务管理（autocommit=false，提交/回滚后归还）——Level 2 线程绑定机制 |
| "内嵌数据源类名是 EmbeddedDatabase" | 实测类名 `EmbeddedDatabaseFactory$EmbeddedDataSourceProxy` |

---

# 生产决策卡（3 张）

## 决策卡 1：回滚规则

```
Decision: 业务异常统一指定 rollbackFor（具体异常类），不用默认
Reason:   默认只回滚运行时异常；业务"预期失败"是检查异常，默认会提交（场景 3 实测）
Alternative: 全部用运行时异常（弃：破坏检查异常语义，调用方无法预期）
Trade-off: 显式 rollbackFor 增加注解噪音，换来回滚意图明确
Validation: 每个事务方法配一条"真实异常类型"的 count 回滚断言（Level 6.5）
```

## 决策卡 2：REQUIRED vs REQUIRES_NEW

```
Decision: 默认 REQUIRED；只有"无论如何都要落库"才 REQUIRES_NEW
Reason:   REQUIRES_NEW 多一次提交 + 挂起外层 + 连接占用变长；但审计/流水不落库=事故无记录
Alternative: 事件监听（@TransactionalEventListener 提交后写）——能异步就别同线程 REQUIRES_NEW
Trade-off: REQUIRES_NEW 解耦外层失败，代价是局部原子性（内层已提交无法撤销）
Validation: 用 PropagationApp 双场景复现验证期望语义（跨 bean 调用）
```

## 决策卡 3：声明式 vs 编程式

```
Decision: 默认声明式（@Transactional 方法级）；同事务内多次提交/精细控制才编程式
Reason:   声明式零侵入；编程式样板代码散落、易错
Alternative: TransactionTemplate（编程式但有事务回调封装，Spring 提供的中间方案，本文未实测）
Trade-off: 声明式绑定方法边界 + 代理依赖（自调用失效），换代码可读性
Validation: 事务失效排查清单（Level 6.1）过一遍
```

---

# 跨语言视角

- **Python/Django**：`@transaction.atomic` 装饰器 = 与 `@Transactional` 完全同构的"代理/装饰器拦截"模式；同样存在"self 调用绕过装饰器"问题（装饰器只包外部调用）。Django 的 `set_rollback()` 对应 Spring 的 `setRollbackOnly()`。
- **Go**：没有框架级声明式事务（无动态代理语言特性）；`database/sql` 的 `Begin/Commit/Rollback` 是纯编程式——Go 社区"函数式事务"（`WithTx(ctx, fn)`）本质是**手工封装代理**。语言没有代理，就自己造回调。
- **Node.js/Prisma**：`prisma.$transaction(async tx => ...)` 回调式事务——把事务边界收敛成**一个回调函数**，从语言层面杜绝"事务控制散落"，是比注解更严格的边界声明（但也失去了"方法级透明"）。
- **通用方法论**：**"透明拦截"（声明式）vs "显式回调"（编程式）** 是所有语言处理横切关注点的两条路；Spring 的代理是隐式拦截，代价就是"绕过代理=失效"——**任何声明式方案，都必须问"拦截点在哪，绕过它的后果是什么"**。

---

# 系列索引

```
00 容器如何创建对象（已重写：四层创建链 + 15 个实测 demo）
01 框架整合 + 配置体系（已重写：6 Level，配置体系独立 Level 4，6 个实测 demo）
02 事件机制与容器通信（已重写：6 Level，5 个实测 demo + 启动事件全景）
03 自动装配深挖（已重写：6 Level，6 个实测 demo：demo10×6）
04 Web 请求链路与运行时刻（已重写：6 Level，4 个实测 demo：demo11.RunTraceApp / WebTraceApp / ActuatorApp / WebFluxApp（双跑法））
05 事务与数据层（本篇：6 Level，4 个实测 demo：demo12.ds.DataSourceApp / tx.TxBasicsApp / tx.PropagationApp / tx.SelfInvocationApp）
06 横切面与 AOP（已完成：6 Level，6 个实测 demo：demo13.aspect.ProxyKindApp / AdviceOrderApp / PointcutApp / AspectOrderApp / ProxyInternalsApp / UnwrapApp / VisibilityApp / TxVisibilityApp）
07 生产实践（已完成：急诊室比喻 + 检查单 7 项 + Level 7 慢发布（指纹测量/三层优化/AOT 选型 + 24 章交叉补充）+ Level 8 优雅停机（demo16 实测 immediate/graceful）+ 决策卡 5 张；实测 demo14×2 + demo15×2 + demo16；与 Boot 4.1 对照线交叉校验）
```

---

# 实验复现

```
cd knowledge/springboot/experiments/code
./build.sh
./run.sh demo12.ds.DataSourceApp            # 数据源分支（默认全 lib → Hikari）
./run.sh demo12.tx.TxBasicsApp              # 回滚规则 4 场景 + afterCompletion 打点
./run.sh demo12.tx.PropagationApp           # REQUIRED vs REQUIRES_NEW
./run.sh demo12.tx.SelfInvocationApp        # 自调用失效 + @Lazy 修复
# DataSource 双跑法（同代码、两组 classpath 实测 classpath 决定数据源类型）：
java -cp "out:$(find lib -name '*.jar' | tr '\n' ':')" demo12.ds.DataSourceApp                        # 跑法 1：Hikari 分支
java -cp "out:$(find lib -name '*.jar' ! -name 'HikariCP*' | tr '\n' ':')" demo12.ds.DataSourceApp   # 跑法 2：内嵌数据源分支
```

四个 App 的关键输出都已固化在各自源文件头部注释，与本文引用一致。

---

# ✅ Final Review Checklist

- [ ] 是否解释了为什么存在？（连接昂贵 + 控制散落 → DataSource；原子性需求 + 多资源 → PlatformTransactionManager；不想写样板 → 声明式代理）
- [ ] 是否说明旧方案为什么失败？（DriverManager 每次新建连接 = 连接风暴；编程式 try/catch 散落业务代码）
- [ ] 是否形成完整因果链？（DataSource → 事务管理器 → @Transactional 语义 → 代理机制 → 自调用失效 → 生产排查，总图在文中）
- [ ] 是否区分规范和实现？（回滚规则/传播语义/DataSource 契约为 Specification；自动配置条件链、JdbcTransactionManager、CGLIB 行为、afterCompletion 为 3.3.5 Implementation）
- [ ] 是否区分语义变化与代码组织变化？（DataSourceTransactionManager → JdbcTransactionManager = 命名变化行为兼容，明确标注非语义变化）
- [ ] 代码实例是否全部实测？（demo12×4 输出原样引用，可复跑；@EnableTransactionManagement 嵌套结构 javap 实证；代理实例字段为 null 的 NPE 实测）
- [ ] 是否包含 Trade-off？（声明式 vs 编程式；REQUIRED vs REQUIRES_NEW 代价；REQUIRES_NEW 解耦 vs 局部原子性丢失）
- [ ] 是否能指导生产决策？（3 张决策卡：回滚规则 / 传播选择 / 声明式边界 + 失效排查清单）
- [ ] 是否存在未经证明的数字？（无编造 benchmark；隔离级别锁行为、setRollbackOnly、UnexpectedRollbackException 路径、非 public 事务方法均标注待验证）
- [ ] 是否只有一个比喻？（记账员）是否只有一个主线角色？（一笔数据库事务的一生）
- [ ] 随机抽查断言：默认回滚仅运行时异常（TxBasicsApp 场景 2/3 实测）、检查异常 rollbackFor 生效（场景 4）、REQUIRES_NEW 独立提交（PropagationApp 场景 B count=1）、自调用事务激活=false + 修复后 true（SelfInvocationApp 打点：@Lazy 自注入 / context.getBean / exposeProxy+AopContext 三种代理通道实测 + 链外 IllegalStateException 实测）、Hikari vs 内嵌双分支（DataSourceApp 双跑法）、JdbcTransactionManager bean + 代理=true（TxBasicsApp 诊断）、代理实例字段 null（NPE 实测）——均有证据来源。
