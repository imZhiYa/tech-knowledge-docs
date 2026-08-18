# DDD 战术建模：从策略发布到实体、值对象与聚合

> 这不是一篇 Entity、Value Object、Aggregate 的名词解释。
>
> 本文只追踪一个主线：**策略 S——一份推荐策略从草稿被发布，直到成为在线决策可以使用的版本。**
>
> `ddd-02` 已经解决了上下文之间如何交换契约。现在进入在线决策内部，看 S 如何穿过：
>
> ```text
> 普通数据
>   -> 有身份的实体
>   -> 有值语义的值对象
>   -> 聚合一致性边界
>   -> 仓储加载与保存
>   -> 贫血/充血与读模型决策
> ```
>
> 本文不把所有数据都强行建成领域对象，而是回答：**哪些规则值得被模型保护，哪些数据应该继续保持简单。**

# 🏷️ 关键词与主线进度

关键词：

```text
Entity | Value Object | Aggregate | Aggregate Root
Identity | Immutability | Equality | Repository
Optimistic Lock | Anemic Model | Rich Model | Read Model
StrategyVersion | 千人千面
```

主线进度：

```text
① ddd-02 契约已经进入在线决策上下文
② 策略 S 为什么不能只是几列状态字段
③ 身份和业务键如何区分
④ 值对象如何保护含义和不可变性
⑤ 聚合如何划一致性边界
⑥ 聚合如何加载、保存并面对并发
⑦ 哪些数据不该使用重型领域模型
```

---

# ⚠️ 版本与证据边界

| 类型 | 本文承诺 |
| --- | --- |
| Specification | DDD 没有统一代码规范；聚合大小、ID 类型和持久化方式属于设计选择 |
| Implementation | Java 示例是领域模型示例；JPA/Spring 片段只表达适配方式，不代表框架强制行为 |
| Version | Java 示例按 JDK 21 语法表达；具体 ORM 行为需要在后续实验中核证 |
| Need Benchmark | 本文不提供性能数字；ID、聚合大小和持久化方案需要结合负载实测 |
| Design Assumption | `RecommendationStrategy` 是策略上下文内需要保护发布规则的候选聚合 |

本文必须区分四种版本：

```text
schemaVersion：契约结构版本
snapshotVersion：特征或数据快照版本
strategyVersion：业务策略版本
persistenceRevision：持久化并发修订号
```

四者不能因为都叫 `version` 就共用一个字段。

---

# 🗂️ 目录

- [Level 1：策略表为什么保护不了业务规则](#level-1)
- [Level 2：实体身份与 ID 设计](#level-2)
- [Level 3：值对象如何保护业务含义](#level-3)
- [Level 4：聚合如何划一致性边界](#level-4)
- [Level 5：聚合如何加载和持久化](#level-5)
- [Level 6：贫血模型、充血模型与读模型选择](#level-6)
- [完整时间线](#timeline)
- [合书自测](#self-test)
- [坑与细节](#pitfalls)
- [概念勘误表](#errata)
- [生产决策卡](#decisions)
- [跨语言视角](#cross-language)

---

# 📍 能力地图

| 层级 | 要打穿的认知墙 | 通关标准 |
| --- | --- | --- |
| Level 1 | “策略表有状态字段就够了” | 能说出状态值背后的发布规则 |
| Level 2 | “ID 就是数据库主键” | 能区分实体身份、业务键和持久化键 |
| Level 3 | “值对象只是小 DTO” | 能用不可变性和值相等保护业务含义 |
| Level 4 | “聚合就是一堆相关对象” | 能按不变量和一致性边界划聚合 |
| Level 5 | “Repository 就是 DAO 改名” | 能画出聚合加载、修改、保存和并发冲突路径 |
| Level 6 | “所有数据都应该充血建模” | 能判断领域模型、读模型和普通数据结构的边界 |

---

# 全文唯一比喻：一份业务申请在专业窗口内形成正式档案

## ASCII Diagram

```text
策略申请 S 进入策略窗口
        |
        v
+-------------------+
| 草稿档案          | 规则还可以编辑
+-------------------+
        |
        | 校验并申请发布
        v
+-------------------+
| 正式档案          | 版本和规则受到保护
+-------------------+
        |
        | 保存到档案柜
        v
+-------------------+
| Repository        | 负责取出和保存完整档案
+-------------------+
```

## Mapping Table

| 比喻元素 | 技术概念 | 唯一职责 |
| --- | --- | --- |
| 档案编号 | Entity Identity | 跨状态变化识别同一个对象 |
| 档案中的含义字段 | Value Object | 让值带着业务规则和相等语义 |
| 一份完整正式档案 | Aggregate | 保护必须一起成立的不变量 |
| 档案负责人 | Aggregate Root | 控制外部修改入口 |
| 档案柜 | Repository | 加载和保存聚合 |
| 草稿到正式的办理动作 | Domain Behavior | 表达业务状态变化 |
| 档案修订号 | Optimistic Lock Version | 发现并发修改冲突 |

比喻只帮助理解身份、状态和边界；具体持久化、事务和并发语义以代码与实验为准。

---

# 正文：六个 Level 打穿战术建模

<a id="level-1"></a>

## Level 1：策略表为什么保护不了业务规则

一句话认知墙：**数据库里有 `status = PUBLISHED`，不代表系统真的保护了“策略已经合法发布”。**

### 前置知识关卡

- [ ] 知道 ddd-02 中的 `PublishedStrategyContract`
- [ ] 能区分字段状态和业务状态迁移
- [ ] 理解在线推荐需要使用已经发布的策略版本

### Why：普通数据对象留下了什么账

**徒弟：**

> 策略不就是一张表吗？`status` 从 `DRAFT` 改成 `PUBLISHED`，再把 `version` 加一，发布就完成了。

**老陈：**

> 这个写法能完成一次状态修改，但它没有保护发布规则。任何调用方只要拿到对象，都可能这样改：
>
> ```java
> strategy.setStatus("PUBLISHED");
> strategy.setVersion(strategy.getVersion() + 1);
> ```
>
> 但发布前可能还需要检查：
>
> ```text
> 策略场景是否明确
> 候选规则是否完整
> 流量比例是否合法
> 模型和召回配置是否匹配
> 当前版本是否已经被其他请求使用
> ```
>
> 如果这些规则散落在 Controller、Service、SQL 和运营脚本中，状态字段只是结果，不是保护规则的模型。

普通数据对象常见的路径：

```text
查询表
  -> 改字段
  -> 保存表
  -> 其他代码继续改字段
```

| 方案 | 解决什么 | 留下什么问题 |
| --- | --- | --- |
| 直接改状态字段 | 开发简单 | 谁都能跳过发布规则 |
| Service 集中校验 | 规则暂时集中 | 其他入口仍可能绕过 Service |
| 数据库约束 | 保护部分格式 | 难以表达完整业务流程 |
| 领域行为 | 状态变化经过明确入口 | 需要建模和测试成本 |

### What：从“数据字段”变成“业务对象”

策略 S 至少包含四类信息：

```text
身份：它是哪一份策略
值：它针对什么场景、使用哪个版本
状态：草稿、已发布、已下线
行为：创建、校验、发布、下线
```

DDD 的战术建模地图：

```text
有身份并持续变化       -> Entity
由属性定义且应不可变     -> Value Object
必须一起保持一致         -> Aggregate
聚合唯一外部入口         -> Aggregate Root
加载和保存聚合           -> Repository
规则不属于单个对象       -> Domain Service
已经发生的业务事实       -> Domain Event
```

### How：先写业务行为，再决定对象类型

```text
① 写出策略发布前必须满足的条件
② 写出发布成功后允许和禁止的变化
③ 判断哪些值需要自己的业务语义
④ 判断哪些状态必须一起保护
⑤ 最后才选择 Entity、Value Object 和 Aggregate
```

**徒弟：**

> 那是不是所有 `setStatus` 都应该改成一个方法？

**老陈：**

> 不是为了消灭 setter 而消灭 setter，而是要问：这个状态变化有没有业务前提。如果有，就让拥有规则的对象控制入口；如果只是查询投影或缓存数据，普通数据结构完全可以。

### Transfer：状态保护还能迁移到哪里

1. 订单从待支付到已支付，需要支付事实和状态规则。
2. 实验从草稿到运行中，需要分流比例和目标人群校验。
3. 优惠券从未领取到已使用，需要身份、时间和使用条件。

本层留下的新账：**策略需要被识别为同一个对象，但数据库主键、业务编号和版本字段不是一回事。**下一层进入实体身份和 ID 设计。

### Level Ending

```text
口诀：字段记录状态，行为保护状态。
```

---

<a id="level-2"></a>

## Level 2：实体身份与 ID 设计

一句话认知墙：**实体的身份回答“是不是同一个对象”，不等于“数据库用什么列存它”。**

### 前置知识关卡

- [ ] 能说出策略 S 的状态变化
- [ ] 能区分业务编号、数据库主键和版本号
- [ ] 知道实体即使属性变化，身份仍然可以保持不变

### Why：三种 ID 为什么不能混成一个

**徒弟：**

> 数据库自增主键已经唯一了，直接把它当 `StrategyId` 不就行了吗？

**老陈：**

> 它确实能在一张表里唯一标识一行，但它还承担了三个不同问题：
>
> ```text
> 数据库如何定位一行
> 业务如何让运营识别策略
> 领域如何跨状态变化识别同一份策略
> ```
>
> 一旦这三个问题共用一个字段，数据库迁移、外部接口和领域模型就会互相绑定。

### What：三类标识的区别

| 标识 | 负责什么 | 示例 | 主要代价 |
| --- | --- | --- | --- |
| 领域身份 ID | 在模型中识别同一个实体 | `StrategyId` | 需要明确生成和传递方式 |
| 业务标识 | 让业务人员或外部系统识别 | `homepage-personalization` | 可能改名，不能随意当永久身份 |
| 数据库主键 | 在某个存储中定位记录 | `id` 自增列 | 绑定持久化，跨系统语义弱 |

常见生成方案：

| 方案 | 优点 | 代价 | 适用判断 |
| --- | --- | --- | --- |
| UUID | 生成简单，跨边界冲突概率低 | 不适合直接给人阅读，存储和索引取舍需实测 | 分布式生成、领域对象身份 |
| 业务标识 | 人能理解，便于运营和接口 | 业务名称可能变化，容易被误当永久身份 | 查询、展示、幂等键的一部分 |
| 数据库自增 | 数据库内简单、顺序自然 | 依赖数据库，跨库和离线创建受限 | 单库内部持久化键 |

本文示例采用“领域身份和业务标识分离”：

```java
public record StrategyId(UUID value) {
    public StrategyId {
        Objects.requireNonNull(value, "value");
    }

    public static StrategyId newId() {
        return new StrategyId(UUID.randomUUID());
    }
}

public record StrategyCode(String value) {
    public StrategyCode {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("strategy code is blank");
        }
    }
}
```

这里没有声称 UUID 在所有场景都优于其他方案。它只是本示例的领域身份选择；具体索引、写入和分片成本必须实测。

### How：用身份测试实体

```text
两个对象的属性完全相同，身份不同 -> 不是同一个实体
同一个对象的状态改变，身份不变 -> 仍是同一个实体
业务名称改变，领域身份不变 -> 仍是同一个实体
数据库迁移，领域身份不变 -> 业务模型不应被存储绑死
```

策略版本还要单独建模：

```text
StrategyId：哪一份策略
StrategyCode：业务上怎么称呼它
StrategyVersion：这份策略的业务版本
persistenceRevision：数据库并发修订号
```

**徒弟：**

> `StrategyVersion` 也有数字，数据库 `@Version` 也有数字，直接共用不就少一个字段了吗？

**老陈：**

> 不能。业务版本表示“规则对外发布了第几版”，并发修订号表示“这行数据被保存了几次”。一次业务发布可能触发多次持久化修订；两者的变化原因不同，必须分开。

### Transfer：身份设计的迁移

1. 订单的订单号可以是业务标识，内部仍需要领域身份。
2. 用户的登录名可以变化，用户身份不能随登录名变化。
3. 实验的实验代码便于运营识别，但不应替代内部实体身份。

本层留下的新账：**实体身份已经明确，但金额、场景、比例和版本这些值仍然可能被普通字符串和数字误用。**下一层用值对象保护业务含义。

### Level Ending

```text
口诀：身份识别对象，业务键服务人，主键服务库。
```

---

<a id="level-3"></a>

## Level 3：值对象如何保护业务含义

一句话认知墙：**值对象不是“小类”，而是让一个值携带自己的含义、合法范围和相等规则。**

### 前置知识关卡

- [ ] 能区分实体身份和业务标识
- [ ] 知道两个数字相等不代表两个业务含义相等
- [ ] 理解不可变性可以减少状态被意外修改

### Why：裸字符串和裸数字为什么危险

**徒弟：**

> `scene` 用 `String`，`traffic` 用 `int`，代码简单，校验放到 Service 里就行了。

**老陈：**

> 这样写把业务含义藏在变量名和注释里。`80` 到底是 80%、80 个用户，还是 80 秒？`homepage` 是场景代码、URL，还是一个页面名称？类型本身没有阻止误用。

常见危险：

```java
strategy.setTraffic(80);
strategy.setScene("homepage");
strategy.setVersion(3);
```

这三行都能编译，但代码没有告诉读者：

```text
80 的单位是什么
homepage 是否允许为空
3 是业务版本还是持久化修订
```

### What：值对象的三个特征

```text
没有独立身份
由属性决定相等性
创建后不应被悄悄修改
```

示例：

```java
public record StrategyVersion(int value) {
    public StrategyVersion {
        if (value < 1) {
            throw new IllegalArgumentException("strategy version must be positive");
        }
    }
}

public record TrafficRatio(int percentage) {
    public TrafficRatio {
        if (percentage < 0 || percentage > 100) {
            throw new IllegalArgumentException("percentage must be between 0 and 100");
        }
    }
}

public record StrategyScene(String value) {
    public StrategyScene {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("scene is blank");
        }
    }
}
```

这三个类型都没有自己的数据库身份：

```text
StrategyVersion(3) == StrategyVersion(3)
TrafficRatio(80) != TrafficRatio(90)
StrategyScene("homepage") == StrategyScene("homepage")
```

### How：不可变和值相等如何落地

```text
① 构造时校验合法范围
② 不提供修改内部状态的方法
③ 需要新值时创建新对象
④ 相等性比较属性，而不是对象地址
⑤ 集合字段使用不可变副本
```

**徒弟：**

> Java `record` 自动提供 `equals` 和 `hashCode`，那所有 record 都是合格值对象吗？

**老陈：**

> 不是。`record` 只提供语法和基于组件的相等性实现，不替你检查业务合法性，也不保证嵌套对象深度不可变。值对象是否成立，仍然要看它有没有完整语义、边界校验和正确的生命周期。

例如：

```java
public record FeatureTags(Set<String> values) {
    public FeatureTags {
        Objects.requireNonNull(values, "values");
        values = Set.copyOf(values);
    }
}
```

`Set.copyOf` 保护了集合结构，但集合中的元素如果本身可变，仍要继续检查。值对象的不可变性是设计责任，不是注解自动赠送的能力。

### Transfer：值对象的迁移

1. `Money` 应该携带币种和金额规则，而不是两个裸数字。
2. `Address` 应该表达完整地址，而不是一组散落字符串。
3. `TimeWindow` 应该保护开始时间不晚于结束时间。
4. `FeatureFreshness` 可以表达数据新鲜度单位和允许窗口。

本层留下的新账：**值的含义和合法范围已经被保护，但多个值之间的规则应该放在哪一起？**下一层进入聚合和一致性边界。

### Level Ending

```text
口诀：值对象保护含义，实体保护身份。
```

---

<a id="level-4"></a>

## Level 4：聚合如何划一致性边界

一句话认知墙：**聚合不是“相关对象的集合”，而是“必须一起保持正确的一组状态”。**

### 前置知识关卡

- [ ] 能区分实体和值对象
- [ ] 能说出至少一条策略发布不变量
- [ ] 知道在线推荐请求不一定是聚合

### Why：为什么不能把整个推荐平台做成一个聚合

**徒弟：**

> 用户、商品、策略、实验都参与推荐，那就放进一个 `RecommendationAggregate`，一次事务全部保证一致，不是最稳吗？

**老陈：**

> 这会把不同变化速度、不同容量和不同责任的对象绑在一个一致性边界里。用户特征更新、商品上下架和策略发布不是一个业务动作，也不应该因为一次推荐请求而互相锁住。

聚合边界应该由不变量推导：

```text
哪些对象必须在一次业务动作中同时成立？
哪些对象可以只通过 ID 关联？
哪些状态变化可以通过事件稍后同步？
```

### What：Strategy 聚合候选

本文把策略发布规则放进 `RecommendationStrategy` 聚合：

```text
RecommendationStrategy（聚合根）
├── StrategyId
├── StrategyCode
├── StrategyScene
├── StrategyVersion
├── StrategyRules
├── TrafficRatio
└── LifecycleStatus
```

聚合内部需要保护的规则示例：

```text
策略必须有场景才能发布
流量比例必须在合法范围内
没有完成校验的草稿不能发布
已发布版本不能被原地悄悄修改
下线只能发生在已发布状态
```

聚合根是外部唯一入口：

```text
外部代码 -> RecommendationStrategy.publish()
外部代码 -X-> strategy.rules.add(...)
外部代码 -X-> strategy.status = PUBLISHED
```

### How：小聚合、ID 引用和跨聚合规则

```java
public final class RecommendationStrategy {
    private final StrategyId id;
    private final StrategyCode code;
    private final StrategyScene scene;
    private StrategyVersion version;
    private LifecycleStatus status;
    private final List<StrategyRule> rules;
    private long expectedPersistenceRevision;

    public void validateCanPublish() {
        if (rules.isEmpty()) {
            throw new IllegalStateException("strategy rules are empty");
        }
        if (status != LifecycleStatus.DRAFT) {
            throw new IllegalStateException("only draft strategy can be published");
        }
    }

    public void publish() {
        validateCanPublish();
        status = LifecycleStatus.PUBLISHED;
    }

    public long expectedPersistenceRevision() {
        return expectedPersistenceRevision;
    }

    void markPersisted(long revision) {
        expectedPersistenceRevision = revision;
    }
}
```

集合也不能把内部修改权交给外部：

```java
public List<StrategyRule> rules() {
    return Collections.unmodifiableList(rules);
}

public void addRule(StrategyRule rule) {
    if (status != LifecycleStatus.DRAFT) {
        throw new IllegalStateException("can only add rules to draft");
    }
    rules.add(Objects.requireNonNull(rule, "rule"));
}
```

`unmodifiableList` 只阻止调用方通过返回值修改集合；如果需要固定快照，可以返回 `List.copyOf(rules)`。两者的选择属于接口语义，不要只为了“看起来安全”随意替换。

这里的代码省略构造器、访问器和 import，重点是：状态和集合都只能通过业务行为变化。

聚合边界的三个原则：

```text
聚合尽量小：只装必须一起保护的状态
聚合之间用 ID：避免形成巨大对象图
跨聚合规则显式协调：不要偷偷把所有对象塞进一个事务
```

“同一场景只能有一个生效策略”可能不是单个 `RecommendationStrategy` 能独立保护的不变量，它可能需要：

```text
策略发布协调器
唯一约束
版本选择服务
或一个专门的 StrategyRelease 聚合
```

不能为了让一句规则落进一个聚合，就盲目扩大聚合边界。

跨聚合规则需要应用层协调，而且“先查询再保存”本身不是并发保护：

```java
@Transactional
public void handle(PublishStrategyCommand command) {
    RecommendationStrategy strategy = strategies.findById(command.strategyId())
            .orElseThrow(() -> new StrategyNotFound(command.strategyId()));

    strategy.validateCanPublish();

    // publication slot 由数据库唯一约束或原子操作保护；不能只靠 exists 查询。
    publicationSlots.claim(command.scene(), strategy.id());

    strategy.publish();
    strategies.save(strategy);
}
```

`claim` 和策略保存必须处在同一个本地事务，或者由一个明确的流程/补偿机制协调。若两个聚合位于不同上下文，就不能假设这个本地事务仍然成立，需要转入事件和最终一致性设计。

**徒弟：**

> 既然跨聚合规则麻烦，把所有策略版本都装进一个聚合不就简单了吗？

**老陈：**

> 只有当它们必须一起保持一致，并且数量和并发规模可控时才合理。否则你只是用一个越来越大的聚合掩盖了跨对象协作问题。聚合边界不是为了让代码少写，而是为了让一致性范围诚实。

### Transfer：聚合边界的迁移

1. 订单聚合保护订单状态和订单项规则，不把支付和库存对象整个装进来。
2. 实验聚合保护实验生命周期，不把每个用户的行为历史装进来。
3. 库存聚合保护某个库存单元的扣减规则，不把整个商品目录装进来。

本层留下的新账：**聚合边界确定了，但它如何从数据库变回完整对象，保存时如何处理并发？**下一层进入仓储和持久化。

### Level Ending

```text
口诀：聚合不是越大越稳，而是边界必须诚实。
```

---

<a id="level-5"></a>

## Level 5：聚合如何加载和持久化

一句话认知墙：**Repository 不是 DAO 的改名，而是让领域模型不必知道数据存在哪里。**

### 前置知识关卡

- [ ] 能说出聚合根的外部入口
- [ ] 能区分业务版本和持久化并发修订号
- [ ] 知道领域模型不应该直接依赖数据库客户端

### Why：领域逻辑直接写数据库会留下什么账

**徒弟：**

> `StrategyMapper.updateStatus(id, "PUBLISHED")` 一行就能完成发布，为什么还要加载聚合？

**老陈：**

> 这行代码快，但它绕过了策略的规则。它没有验证规则是否完整，也没有确认当前状态是否仍然是草稿，更没有表达发布动作本身。
>
> 如果业务规则需要被保护，就应该先拿到聚合，让聚合执行行为，再保存聚合。

### What：Repository 的位置和职责

领域层只声明自己需要的能力：

```java
public interface StrategyRepository {
    Optional<RecommendationStrategy> findById(StrategyId id);

    void save(RecommendationStrategy strategy);
}
```

实现放在基础设施层：

```text
Domain
  -> StrategyRepository（接口）

Infrastructure
  -> JpaStrategyRepository（实现）
  -> StrategyRecord（持久化对象）
  -> StrategyMapper（Record <-> Domain）
```

下面是一个 JPA 适配器示意。代码省略 import、规则表映射和构造器，重点看三件事：持久化对象不等于领域对象、Mapper 负责转换、`@Version` 负责并发修订检查。

```java
@Entity
class StrategyRecord {
    @Id
    UUID id;

    String strategyCode;
    String scene;
    int strategyVersion;
    String status;

    @Version
    long persistenceRevision;

    // List<StrategyRuleRecord> rules;
}

interface StrategyRecordJpaRepository
        extends JpaRepository<StrategyRecord, UUID> {
}

final class JpaStrategyRepository implements StrategyRepository {
    private final StrategyRecordJpaRepository records;
    private final StrategyMapper mapper;

    @Override
    public Optional<RecommendationStrategy> findById(StrategyId id) {
        return records.findById(id.value())
                .map(mapper::toDomain);
    }

    @Override
    public void save(RecommendationStrategy strategy) {
        StrategyRecord record = mapper.toRecord(strategy);
        try {
            StrategyRecord saved = records.saveAndFlush(record);
            strategy.markPersisted(saved.persistenceRevision);
        } catch (OptimisticLockingFailureException ex) {
            throw new ConcurrentStrategyUpdate(strategy.id(), ex);
        }
    }
}

final class StrategyMapper {
    RecommendationStrategy toDomain(StrategyRecord record) {
        // 领域聚合提供仅供仓储使用的 reconstitute(...) 工厂。
        return RecommendationStrategy.reconstitute(
                new StrategyId(record.id),
                new StrategyCode(record.strategyCode),
                new StrategyScene(record.scene),
                new StrategyVersion(record.strategyVersion),
                LifecycleStatus.valueOf(record.status),
                /* rules */ List.of(),
                record.persistenceRevision
        );
    }

    StrategyRecord toRecord(RecommendationStrategy strategy) {
        StrategyRecord record = new StrategyRecord();
        record.id = strategy.id().value();
        record.strategyCode = strategy.code().value();
        record.scene = strategy.scene().value();
        record.strategyVersion = strategy.version().value();
        record.status = strategy.status().name();
        record.persistenceRevision = strategy.expectedPersistenceRevision();
        return record;
    }
}
```

这段实现表达的并发路径是：

```text
加载 revision=10
  -> 修改领域聚合
  -> 保存时携带 revision=10
  -> ORM/数据库检查版本
  -> 成功后生成 revision=11
```

另一个请求如果也拿着 `revision=10` 保存，就应该抛出并发冲突，而不是覆盖已经提交的策略。`@Version` 的具体 SQL、异常类型和 flush 时机属于 JPA/Spring Data 实现，后续实验必须按实际版本验证。

`expectedPersistenceRevision` 是本文示例的简化实现。更严格的领域纯度方案，可以把它放到 Unit of Work 或持久化上下文，而不让聚合看到这个技术字段；两种方案都有成本，不能把其中一种写成 DDD 规范。

Repository 的对象通常是聚合根，不是每个值对象或每张表都创建一个 Repository。

### How：一次发布的加载和保存链路

```text
① Controller 接收 PublishStrategy
② Application Service 开启本地事务
③ StrategyRepository 加载 RecommendationStrategy 聚合
④ strategy.publish() 执行业务规则
⑤ StrategyRepository 保存完整聚合
⑥ 事务提交
⑦ 后续发布 StrategyPublished 事实
```

领域层和持久化对象分离：

```java
// 持久化适配器示意，具体 JPA 行为待实验核证
class StrategyRecord {
    UUID id;
    String strategyCode;
    int strategyVersion;
    String status;
    long persistenceRevision;
}
```

字段语义必须分开：

```text
strategyVersion：业务发布了第几版
persistenceRevision：这条记录用于并发检测的修订号
```

如果两个请求同时加载草稿：

```text
请求 A：加载 revision=10
请求 B：加载 revision=10
请求 A：发布并保存成功，revision=11
请求 B：仍用 revision=10 保存
```

请求 B 应该被识别为并发冲突，而不是静默覆盖 A。具体的乐观锁注解、SQL 条件和异常行为属于 ORM/数据库实现，需要用实验验证。

**徒弟：**

> Repository 抽象了数据库，那是不是可以把所有查询都塞进 Repository？

**老陈：**

> Repository 的核心职责是聚合的加载和保存。在线推荐的候选查询、报表聚合和搜索读模型，通常不应该伪装成聚合 Repository。查询模型和写模型可以有不同的访问方式。

### Transfer：持久化隔离的迁移

1. 订单聚合可以由关系数据库保存，搜索结果由 ES 读模型提供。
2. 推荐策略配置可以走事务存储，在线读取走缓存或本地快照。
3. 外部支付状态可以通过适配器映射成本地领域状态。

本层留下的新账：**不是所有读数据都需要加载聚合。**下一层比较贫血模型、充血模型和读模型，决定 DDD 应该覆盖哪些路径。

### Level Ending

```text
口诀：领域决定怎么改，仓储决定放哪里。
```

---

<a id="level-6"></a>

## Level 6：贫血模型、充血模型与读模型选择

一句话认知墙：**领域模型负责保护规则，读模型负责快速回答问题；两者不必长成同一种对象。**

### 前置知识关卡

- [ ] 能说出聚合为什么是边界而不是大对象
- [ ] 能画出 Repository 的接口和适配器方向
- [ ] 能区分写入规则和在线查询数据

### Why：贫血和充血为什么不是绝对站队

**徒弟：**

> 贫血模型一定是反模式，所有 Entity 都应该把规则写进方法里吧？

**老陈：**

> 在策略发布、支付、订单这类规则密集的写模型中，只有 getter/setter 的对象很容易让规则散落。但推荐特征、报表行和搜索结果本来就是读数据，强行给它们增加复杂行为反而是浪费。

### What：同一个发布动作的两种写法

贫血写法：

```java
// 规则集中在 Service，但任何其他入口仍可能绕过它
public void publish(StrategyEntity entity) {
    validate(entity);
    entity.setStatus("PUBLISHED");
    entity.setStrategyVersion(entity.getStrategyVersion() + 1);
    repository.save(entity);
}
```

充血写法：

```java
public void publish(StrategyId id) {
    RecommendationStrategy strategy = repository.findById(id)
            .orElseThrow(() -> new StrategyNotFound(id));

    strategy.publish();
    repository.save(strategy);
}
```

充血不是“方法越多越好”，而是让聚合拥有它必须保护的规则。应用服务仍然负责流程、事务和外部协作。

### What：领域模型还是读数据判断矩阵

| 判断问题 | 是 | 否 |
| --- | --- | --- |
| 是否有稳定身份？ | 考虑 Entity | 继续判断值对象或普通数据 |
| 是否有状态变化规则？ | 考虑领域行为 | 可能只是 DTO/读模型 |
| 是否必须保护不变量？ | 考虑 Aggregate | 不要强行建聚合 |
| 是否主要用于查询展示？ | 可以使用 Read Model | 进入写模型判断 |
| 是否需要高吞吐、低延迟读取？ | 优先缓存/读模型/特征存储 | 可以使用领域仓储 |
| 是否需要跨边界传输？ | 使用 Contract/DTO + ACL | 使用本地模型 |

明确“不需要领域模型”的对象示例：

| 对象 | 为什么不需要领域模型 | 推荐形态 |
| --- | --- | --- |
| `RecommendationRequest` | 一次查询输入，没有独立生命周期 | Request DTO/record |
| `CandidateScore` | 模型排序的中间结果，不拥有业务状态 | 算法数据结构 |
| `PublishedStrategyView` | 给在线查询使用的投影 | Read Model/record |
| `FeatureSnapshotResponse` | 跨上下文传输结构 | Contract/DTO |
| `RecommendationHttpResponse` | 接口展示格式 | Response DTO |
| `CacheEntry` | 缓存存储形态，不拥有业务规则 | 缓存数据结构 |

“不需要领域模型”不等于“不需要类型”。它只是说明这个对象不应该承担 Entity、Aggregate 或 Repository 的职责。

四种常见对象不要混为一谈：

```text
领域对象：保护业务规则
持久化对象：适配数据库结构
传输对象：适配上下文契约
读模型：为查询性能和展示形态组织数据
```

### How：把策略写模型和推荐读模型分开

```text
策略配置写路径
  -> RecommendationStrategy 聚合
  -> StrategyRepository
  -> 事务存储

在线推荐读路径
  -> PublishedStrategyReadModel
  -> 缓存/特征存储
  -> RecommendationRequest
```

读模型可以是一个简单的不可变投影：

```java
public record PublishedStrategyView(
        StrategyCode code,
        StrategyScene scene,
        StrategyVersion version,
        TrafficRatio traffic,
        Instant publishedAt
) {
}
```

它的职责是让查询拿到需要的字段，不负责执行“发布策略”这个业务动作，也不应该反向修改 `RecommendationStrategy` 聚合。

发布策略时，写模型保护规则；发布事实发生后，读模型异步刷新。这样在线推荐不必为了每次查询加载完整聚合，也不意味着读模型可以反过来修改聚合。

**徒弟：**

> 既然读模型是异步刷新的，为什么不直接让读模型成为唯一数据？

**老陈：**

> 读模型擅长回答“现在怎么查”，不一定擅长保护“什么状态允许被写”。如果它丢失、重建或延迟，系统仍需要一个拥有业务规则的写模型作为事实来源。

### Transfer：选择性 DDD 的边界

1. 推荐策略发布和实验生命周期适合丰富领域模型。
2. 用户特征快照和候选列表适合读模型或数据结构。
3. 搜索、报表和离线训练不要因为名称带“业务”就强行变成聚合。
4. 一个系统可以同时拥有充血写模型、贫血查询模型和算法数据模型。

本篇战术建模到此闭合。下一篇把这些对象放回一次完整推荐请求：应用层如何编排，领域层如何保持纯度，基础设施端口如何接入真实技术。

### Level Ending

```text
口诀：写模型护规则，读模型护查询。
```

---

# 🧪 合书自测

<a id="self-test"></a>

## 完整时间线

<a id="timeline"></a>

```text
T0  策略 S 处于草稿状态
    |
    v
T1  加载 RecommendationStrategy 聚合
    |
    v
T2  校验场景、规则和流量比例
    |
    v
T3  strategy.publish() 改变业务状态
    |
    v
T4  Repository 保存聚合并检查并发修订号
    |
    v
T5  事务提交
    |
    v
T6  发布策略事实，刷新在线读模型
    |
    v
T7  推荐请求读取读模型，不直接修改策略聚合
```

## 自测问题

| 问题 | 必须回答的不变量 |
| --- | --- |
| Entity 和 Value Object 的区别是什么？ | Entity 由身份识别，Value Object 由属性和值相等识别 |
| UUID、业务标识和数据库主键是否相同？ | 三者服务不同责任，不能默认共用 |
| `record` 是否自动成为值对象？ | 还需要完整语义、校验和不可变性 |
| 聚合边界由什么决定？ | 必须一起保持的一组业务不变量 |
| 聚合外部如何修改内部状态？ | 只能通过聚合根的业务行为 |
| Repository 主要保存什么？ | 聚合根，而不是每个表或每个值对象 |
| 业务版本和持久化修订号是否相同？ | 不同，前者表达业务发布，后者表达并发检测 |
| 贫血模型是否永远错误？ | 规则密集的写模型需保护行为，简单读模型可以保持简单 |
| 推荐请求是否一定加载聚合？ | 不一定，在线查询通常使用读模型或特征存储 |

---

# ⚠️ 坑与细节

<a id="pitfalls"></a>

## 坑 1：把数据库自增 ID 当成领域身份

错误理解：只要数据库主键唯一，它就是领域 Entity ID。

原因：把持久化定位和业务身份混成一个概念。

比喻后果：档案柜编号变化，窗口就认为业务对象换了一个。

线上现象：跨库、导入、同步或外部接口变化时，业务标识被存储细节牵着走。

修正：领域身份、业务标识和数据库主键分别建模。

---

## 坑 2：用字符串和数字表达所有值

错误理解：`String scene`、`int traffic` 足够简单。

原因：业务单位、范围和语义被隐藏在变量名中。

比喻后果：窗口表格上只有数字，没有说明它代表比例、人数还是时间。

线上现象：单位误用、空值传播和非法范围只能在运行期暴露。

修正：为稳定且有规则的值建立值对象。

---

## 坑 3：把聚合做成全局大对象

错误理解：把用户、商品、策略、实验和库存都放进一个聚合最一致。

原因：没有从不变量推导一致性边界。

比喻后果：一个窗口档案变成整个城市总档案，任何小修改都要牵动全部内容。

线上现象：加载慢、事务大、并发冲突和跨团队修改增加。

修正：聚合只包含必须一起保持正确的状态，跨聚合使用 ID 和显式协作。

---

## 坑 4：聚合内部集合可以被外部直接修改

错误理解：返回 `List<StrategyRule>`，调用方自己添加规则更灵活。

原因：聚合根失去了状态修改权。

比喻后果：任何窗口都能直接往正式档案里塞材料，不需要经过审核。

线上现象：规则不完整的策略进入发布流程，错误只能在后面暴露。

修正：返回不可变视图或副本，通过聚合根方法添加和删除规则。

---

## 坑 5：用业务版本代替并发修订号

错误理解：`strategyVersion` 加一就能解决并发覆盖。

原因：业务版本和持久化并发检测表达不同事实。

比喻后果：档案业务版本和档案柜修改次数写在同一格，无法判断冲突来源。

线上现象：两个发布请求互相覆盖，版本号看起来正常但规则丢失。

修正：分开 `strategyVersion` 和 `persistenceRevision`，并用存储条件检查并发。

---

## 坑 6：Repository 只是 DAO 换了名字

错误理解：每张表创建一个 Repository，所有查询和更新都塞进去。

原因：没有理解 Repository 是聚合的持久化边界。

比喻后果：每个窗口都能直接从档案柜拿半份档案修改，没人负责完整档案。

线上现象：领域规则绕过聚合，查询和写入模型互相污染。

修正：聚合根使用 Repository；报表、搜索和读模型使用适合查询的访问方式。

---

## 坑 7：贫血模型和充血模型二选一站队

错误理解：所有对象都必须有行为，或者所有对象都应该只是数据。

原因：忽略了写模型和读模型的不同目标。

比喻后果：简单查询也要走复杂审核，复杂档案却没有规则保护。

线上现象：代码复杂度和业务复杂度错配。

修正：规则密集写模型保护行为，查询和特征读模型保持简单。

---

## 坑 8：只用 ORM 关系自动决定聚合边界

错误理解：JPA 的一对多关系就是一个聚合。

原因：数据库关联描述存储关系，不自动描述业务一致性关系。

比喻后果：档案柜里的相邻文件被误认为必须由同一个窗口同时审批。

线上现象：懒加载链路过长、事务范围扩大、聚合边界被 ORM 反向决定。

修正：先从业务不变量决定聚合，再设计持久化映射。

---

# 📊 竖切总表

| 时间 | 位置 | 动作 | 不变量 | 风险 |
| --- | --- | --- | --- | --- |
| T0 | 策略上下文 | 创建草稿 S | 身份和业务键明确 | 把主键当全部身份 |
| T1 | 领域模型 | 创建值对象 | 值有含义、范围和相等性 | 裸字符串/数字误用 |
| T2 | 聚合根 | 执行发布行为 | 未校验策略不能发布 | 外部直接改状态 |
| T3 | 应用层 | 编排加载、修改和保存 | 事务覆盖业务用例 | Service/DAO 绕过规则 |
| T4 | Repository | 加载和保存聚合 | 完整聚合作为边界 | 每表一个 Repository |
| T5 | 持久化 | 检查并发修订号 | 并发修改不能静默覆盖 | 业务版本混用 |
| T6 | 事件/读模型 | 刷新在线策略视图 | 读模型不能反向修改聚合 | 读写模型混在一起 |
| T7 | 在线请求 | 查询推荐结果 | 低延迟路径不加载无关聚合 | 用领域模型替代读模型 |

---

# 📚 概念勘误表

<a id="errata"></a>

| 错误说法 | 准确说法 |
| --- | --- |
| Entity 就是数据库表映射类 | Entity 的核心是身份和生命周期连续性，是否映射一张表是实现选择 |
| UUID 一定比自增 ID 好 | ID 选择取决于生成位置、外部暴露、存储和索引约束 |
| 业务编号可以永远当 Entity ID | 业务编号可能改名或有上下文含义，通常应与稳定身份分开 |
| `record` 自动就是值对象 | 还需要业务校验、不可变性和完整值语义 |
| 聚合越大一致性越强 | 聚合越大可能带来加载、锁和并发成本，边界必须由不变量决定 |
| 聚合之间应该持有对象引用 | 通常用 ID 引用，避免跨聚合对象图和隐式加载 |
| Repository 就是 DAO 的别名 | Repository 以聚合为持久化边界，查询模型可以使用其他访问方式 |
| ORM 关系自动决定聚合边界 | 业务不变量先决定聚合，ORM 只负责实现映射 |
| 贫血模型永远错误 | 规则密集写模型需要行为，简单读模型可以保持数据形态 |

其中两点我以前也讲得过于绝对：

```text
我以前容易把“充血模型”讲成所有对象的默认答案；准确说法是先区分写模型和读模型。
我以前容易把“聚合根”讲成一张表的根对象；准确说法是聚合根是业务一致性边界的外部入口。
```

---

# 🏆 生产决策卡

<a id="decisions"></a>

## Decision Card 1：领域身份使用什么 ID

### 场景

策略需要在模块、事件和数据库之间传递身份，团队准备直接复用数据库自增主键。

### 判断

先区分领域身份、业务标识和数据库主键。默认示例使用独立 `StrategyId` 和 `StrategyCode`，具体 ID 类型按生成、存储和外部暴露约束验证。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 领域跨边界传递 | 使用稳定领域身份 |
| 运营和外部查询 | 使用可读业务标识 |
| 单库内部定位 | 可以使用数据库主键 |
| 分布式生成 | 评估 UUID 等方案的存储和索引代价 |

### Code

```java
public record StrategyId(UUID value) {}
public record StrategyCode(String value) {}
```

### 禁止决策

```text
因为数据库主键唯一，所以它自动就是所有上下文的业务身份。
```

### 验收指标

```text
跨上下文 ID 解析失败数
业务标识重复率
ID 生成冲突数
索引写入 latency
存储 Memory
Error rate
```

---

## Decision Card 2：这个对象是聚合还是读数据

### 场景

团队准备为 `UserFeature`、`RecommendationRequest`、`CandidateItem` 都创建 Entity 和 Repository。

### 判断

只把拥有身份、状态变化和必须保护不变量的对象建成领域模型；高吞吐查询优先使用读模型。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 有稳定身份和生命周期 | 考虑 Entity |
| 值有完整含义和边界 | 考虑 Value Object |
| 必须一起保持规则 | 考虑 Aggregate |
| 只用于查询展示 | 使用 Read Model/DTO |
| 主要是算法中间数据 | 使用特征/算法数据结构 |

### Code

```text
策略发布 -> RecommendationStrategy 聚合
在线推荐 -> PublishedStrategyReadModel
模型输入 -> RankingInput
```

### 禁止决策

```text
凡是业务名词都创建聚合，凡是查询都加载完整领域对象。
```

### 验收指标

```text
聚合加载次数
在线查询数据库访问次数
recommendation_request_latency_p99
feature_provider_latency
CPU
Memory
```

---

## Decision Card 3：聚合边界如何确定

### 场景

策略、实验、用户画像和商品状态都参与推荐，团队准备放入一个大聚合。

### 判断

从必须一起保持的不变量出发；能通过 ID 和事件协作的对象不要强行放进同一个聚合。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 策略发布前规则必须完整 | 放入 Strategy 聚合 |
| 实验生命周期独立变化 | 独立 Experiment 聚合 |
| 商品可售状态由商品上下文拥有 | 通过契约/事件提供状态 |
| 同一场景唯一生效版本 | 评估协调器、唯一约束或专门聚合 |

### Code

```java
strategy.publish();
// 不直接修改 strategy.status，也不加载整个推荐平台
```

### 禁止决策

```text
为了减少跨聚合协作，把所有相关对象塞进一个全局聚合。
```

### 验收指标

```text
聚合平均加载对象数量
事务 duration
并发冲突率
数据库锁等待
CPU
Memory
```

---

## Decision Card 4：贫血模型还是充血模型

### 场景

团队争论所有领域对象是否都必须包含行为。

### 判断

规则密集的写模型使用行为保护；简单查询、搜索结果和特征快照保持数据形态。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 状态迁移有前置条件 | 让领域对象控制行为 |
| 查询只需要投影 | 使用 Read Model |
| 业务规则跨多个对象 | 使用应用服务/领域服务协调 |
| 只有字段搬运 | 不强行建立复杂模型 |

### Code

```java
strategy.publish();       // 写模型：规则由聚合保护
readModel.findByScene();  // 读模型：为查询组织数据
```

### 禁止决策

```text
用“充血模型”或“贫血模型”作为全系统统一信仰，不看业务规则密度。
```

### 验收指标

```text
规则测试场景数
读模型查询 latency
写模型事务 duration
recommendation_request_latency_p99
Error rate
```

---

# 🌍 跨语言视角

<a id="cross-language"></a>

## Java

```text
record 表达值对象和契约
class 表达需要身份和行为的实体
interface 表达 Repository/Port
module/package 控制聚合和上下文边界
```

## Go

```text
struct 表达实体、值和持久化映射
方法接收者表达领域行为
interface 表达 Repository/Port
package/internal 隐藏聚合内部状态
```

## Rust

```text
struct/enum 表达值和状态
impl 表达领域行为
trait 表达 Repository/Port
pub 可见性控制聚合根入口
```

## 跨语言仍成立的判断力

```text
身份决定对象是否是同一个实体。
值语义决定对象是否应该不可变。
不变量决定聚合边界。
持久化结构不能反过来决定业务模型。
读模型和写模型可以使用不同的数据形态。
```

---

# 系列承接：下一篇留下的问题

本篇已经回答：

```text
哪些对象需要身份
哪些值应该成为值对象
聚合如何保护不变量
Repository 如何加载和保存聚合
贫血模型、充血模型和读模型如何选择
```

但策略 S 已经可以被正确加载、修改和保存，接下来还需要把一次推荐请求完整串起来：谁负责编排，事务在哪里开始，领域层如何不依赖框架，Redis、模型服务和 MQ 如何通过端口接入？

这就是 `ddd-04-一次推荐请求如何落到代码.md` 的入口。
