# DDD 事件与最终一致性：从 Outbox 到幂等、版本演进与补偿

> 这不是一篇“如何发消息”的教程。
>
> 本文只追踪一个主线：**事件 P——`StrategyPublished`，策略聚合发布后，事件如何可靠地到达在线决策、实验和其他上下文。**
>
> `ddd-04` 结尾留下了一笔账：`ExposurePublisher` 和 `StrategyPublished` 都还不确定是否可靠。P 要穿过：
>
> ```text
> 领域事实
>   -> 集成事件
>   -> Outbox 与本地事务
>   -> 可靠投递
>   -> 幂等消费
>   -> 版本演进
>   -> 失败补偿与对账
> ```
>
> 本文的目标不是让你相信“消息不会丢”，而是说清楚：**哪些不变量由谁来保证，哪些窗口只能靠业务补偿关闭。**
>
> 本文大量回链 `knowledge/mq` 系列的投递语义、重复、幂等结论，以及 `knowledge/springboot` 的事务边界知识，不在 DDD 内部重复推导。

# 🏷️ 关键词与主线进度

关键词：

```text
Domain Event | Integration Event | Outbox | Local Transaction
At-least-once | Idempotency | Event Versioning | Compensation
Retry | DLQ | 对账 | 最终一致性 | 千人千面
```

主线进度：

```text
① ddd-04 留下的事件可靠性账
② 领域事件和集成事件的区别
③ Outbox 如何与本地事务绑定
④ 重复投递与幂等消费
⑤ 事件 Schema 版本如何演进
⑥ 消费失败、补偿与对账
⑦ 最终一致性的生产验证
```

---

# ⚠️ 版本与证据边界

| 类型 | 本文承诺 |
| --- | --- |
| Specification | 投递语义三分类（at-most-once / at-least-once / exactly-once）是分布式系统通用语义；具体 Broker 行为以所用版本为准 |
| Implementation | Java 代码是本系列示例；Spring 事务、Kafka/RocketMQ 具体默认值不写死，实验时按实际版本核证 |
| Version | Java 示例按 JDK 21 表达；MQ 产品行为回链 `knowledge/mq` 基线 |
| Need Benchmark | 本文没有性能数字；Outbox 轮询周期、积压上限和延迟需要结合负载实测 |
| Design Assumption | 策略发布走本地事务 + Outbox；曝光事实按业务允许窗口异步传播 |

必须区分五层语义：

```text
领域事件：某个聚合内部发生了业务事实
集成事件：跨上下文传播事实的契约和传输
Outbox：与本地事务绑定的可靠发布表
投递语义：消息最多/至少被送达几次
业务处理：消费后产生什么业务结果
```

---

# 🗂️ 目录

- [Level 1：领域事件和集成事件为什么不是一回事](#level-1)
- [Level 2：Outbox 如何与本地事务绑定](#level-2)
- [Level 3：重复投递与幂等消费](#level-3)
- [Level 4：事件 Schema 版本如何演进](#level-4)
- [Level 5：消费失败、补偿与对账](#level-5)
- [Level 6：最终一致性的生产验证](#level-6)
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
| Level 1 | “领域事件就是发到 MQ 的消息” | 能区分领域事实、集成契约和传输实现 |
| Level 2 | “事务里直接发 MQ 就行” | 能解释双写没有原子性，画出 Outbox 状态流转 |
| Level 3 | “消息有 ACK 就不会重复” | 能用事件 ID/业务键设计幂等，不依赖 Broker 承诺 |
| Level 4 | “字段想加就加” | 能设计兼容演进，区分 schemaVersion 与业务版本 |
| Level 5 | “失败就重试到成功” | 能设计重试上限、死信、补偿与对账 |
| Level 6 | “最终一致就是乱” | 能用时间窗口、指标和故障演练验收一致性 |

---

# 全文唯一比喻：一份业务申请在多个专业窗口办理

## ASCII Diagram

```text
策略窗口办结一件事
        |
        | 记录：已发布
        v
+-------------------+
| 档案与交接登记本  | 档案和通知登记同一次落笔
+-------------------+
        |
        | 之后由专员投递
        v
+-------------------+
| 通知投递窗口      | 可能重复送达
+-------------------+
        |
        | 各窗口按登记编号去重
        v
+-------------------+
| 画像/决策/实验窗口 | 各按自己的规则处理
+-------------------+
```

## Mapping Table

| 比喻元素 | 技术概念 | 唯一职责 |
| --- | --- | --- |
| 策略窗口办结 | Domain Event | 表达聚合内部发生的业务事实 |
| 跨窗口的通知格式 | Integration Event | 表达跨上下文传播的稳定契约 |
| 档案与交接登记本同次落笔 | Outbox + Local Transaction | 让业务状态和待发事件一起提交 |
| 之后由专员投递 | Outbox Publisher | 从登记本异步发布 |
| 可能重复送达 | At-least-once 投递 | 不承诺恰好一次 |
| 按登记编号去重 | Idempotency | 消费端处理重复 |
| 各窗口按自己的规则处理 | 消费端业务处理 | 只负责自己的业务结果 |

比喻只帮助理解“先登记、后投递、可能重复”，不承诺任何具体 Broker 的恰好一次行为。

---

# 正文：六个 Level 打穿事件与最终一致性

<a id="level-1"></a>

## Level 1：领域事件和集成事件为什么不是一回事

一句话认知墙：**领域事件是“发生了什么”，集成事件是“怎么告诉别的上下文”。**

### 前置知识关卡

- [ ] 知道 ddd-03 中策略聚合的 `publish()` 行为
- [ ] 知道 ddd-02 中跨上下文使用契约和 ACL
- [ ] 能区分“业务事实”和“消息格式”

### Why：把两者混成一个会留下什么账

**徒弟：**

> `StrategyPublished` 发到 Kafka 就是领域事件，领域事件就是发到 MQ 的消息，为什么还要分成两种？

**老陈：**

> 因为领域事件属于本地业务语义，集成事件属于跨上下文契约。混在一起会出现三类问题：
>
> ```text
> 本地对象改名，所有消费者被迫适配内部模型
> 领域对象直接序列化，字段泄漏和版本漂移
> 领域层开始依赖 MQ 客户端，规则只能在有 Broker 时测试
> ```
>
> 策略上下文内部的 `RecommendationStrategy.publish()` 产生的是领域事实；把这个事实告诉其他上下文，才是集成事件。

### What：两者边界

| 维度 | 领域事件 | 集成事件 |
| --- | --- | --- |
| 作用范围 | 上下文内部 | 上下文之间 |
| 表达内容 | 聚合发生了业务事实 | 其他上下文需要的事实契约 |
| 模型边界 | 可以携带领域类型 | 使用稳定契约字段 |
| 传输 | 进程内可直接分发 | 通过 MQ/HTTP 等传输 |
| 变化原因 | 本地业务规则 | 跨上下文契约演进 |

策略发布事件的本地事实：

```java
public record StrategyPublished(
        StrategyId strategyId,
        StrategyScene scene,
        StrategyVersion version
) {
}
```

跨上下文契约可以独立演进：

```java
public record StrategyPublishedIntegrationEvent(
        String strategyId,
        String scene,
        int strategyVersion,
        int schemaVersion,
        Instant publishedAt
) {
}
```

两边字段可以不同，但含义必须能通过 ACL/翻译显式映射。`schemaVersion` 描述契约结构，`strategyVersion` 描述业务版本。

### How：先有领域事实，再谈集成

```text
① 聚合行为产生领域事实
② 确认事实需要被外部知道
③ 定义跨上下文契约
④ 在边界处翻译成集成事件
⑤ 可靠发布
⑥ 消费方按契约处理
```

**徒弟：**

> 是不是所有领域事件都必须变成集成事件发出去？

**老陈：**

> 不是。只有跨上下文有业务价值的事实才需要传播。内部统计、审计临时状态等并不需要所有消费者都收到。事件治理的第一步不是“多发”，而是“谁需要知道”。

### Transfer：领域与集成事件的迁移

1. 支付成功是支付上下文的事实，通知积分系统的是集成事件。
2. 订单创建是订单上下文的事实，通知库存系统的是集成事件。
3. 实验分流是实验上下文的事实，同步在线决策的是集成事件。

本层留下的新账：**集成事件要发出去，但数据库事务和消息发送之间没有原子性。**下一层看 Outbox 如何把“状态变更”和“待发事件”绑在一起。

### Level Ending

```text
口诀：领域事件管内情，集成事件管契约。
```

---

<a id="level-2"></a>

## Level 2：Outbox 如何与本地事务绑定

一句话认知墙：**数据库和 MQ 是两个系统，它们之间没有原子事务；Outbox 用本地事务先落表，再异步投递。**

### 前置知识关卡

- [ ] 能说出 ddd-04 中写路径的本地事务
- [ ] 知道 MQ 的 at-least-once 投递语义
- [ ] 理解“send 返回成功”不等于 Broker 持久化确认

### Why：事务里直接发 MQ 为什么不可靠

**徒弟：**

> `@Transactional` 方法里保存策略后调用 `kafkaTemplate.send()`，回滚时消息已经发出去，不是正好吗？

**老陈：**

> 这个写法有两个窗口：
>
> ```text
> 窗口一：DB 提交了，MQ 发送失败 -> 状态变了，事件丢了
> 窗口二：MQ 发成功了，DB 回滚了 -> 事件有了，状态没了
> ```
>
> 数据库事务只管理数据库连接。消息发送在事务之外，无论顺序如何，两边都可能“一半成功”。这不是编码不小心，而是两个系统之间没有共享事务。

双写一致问题的通用结构：

```text
DB + MQ 没有原子事务
  -> 只能选一个方向：先 DB 还是先 MQ
  -> 无论哪个方向，都留一个失败窗口
  -> 用 Outbox 把“待发事件”写进 DB 事务
  -> 再由后台可靠投递
```

### What：Outbox 的最小结构

```text
strategy 表：策略聚合状态
outbox 表：待发事件登记本
```

同一个本地事务写入两者：

```java
@Transactional
public void publish(PublishStrategyCommand command) {
    RecommendationStrategy strategy = strategyRepository
            .findById(command.strategyId())
            .orElseThrow(() -> new StrategyNotFound(command.strategyId()));

    strategy.publish();
    strategyRepository.save(strategy);

    outboxRepository.append(
            StrategyPublishedIntegrationEvent.from(strategy)
    );
}
```

Outbox 记录示意：

```java
class OutboxEntry {
    UUID eventId;
    String eventType;
    String payload;
    String schemaVersion;
    OutboxStatus status; // PENDING / PUBLISHED
    Instant createdAt;
}
```

状态流转：

```text
PENDING
  -> 轮询/通知到 Publisher
  -> 投递成功 -> PUBLISHED
  -> 投递失败 -> 保留 PENDING，下一轮重试
```

### How：发布链路

```text
① 本地事务：保存策略 + 写 Outbox
② 事务提交
③ Outbox Publisher 拉取 PENDING 记录
④ 调用 MQ 发送集成事件
⑤ 标记 PUBLISHED
⑥ 消费方接收、处理、ack
```

**重要边界：**

```text
Publisher 标记 PUBLISHED 表示“已交给 Broker”
不等于“消费方已处理成功”
消费方的 ack 表示“我不再要这条消息”
也不等于业务结果正确（回链 mq-03 的确认语义）
```

Outbox 轮询周期、批量大小、积压上限和补偿任务需要结合负载实验，本文不写固定数字。

Publisher 的实现示意如下。调度间隔、批量和发送超时都来自配置，不写死：

```java
@Component
final class OutboxPublisher {
    private final OutboxRepository outbox;
    private final IntegrationEventSender sender;
    private final OutboxPublisherProperties properties;

    @Scheduled(fixedDelayString = "${outbox.publisher.delay-ms}")
    public void publish() {
        List<OutboxEntry> pending =
                outbox.findPending(properties.batchSize());
        for (OutboxEntry entry : pending) {
            try {
                sender.send(entry);
                outbox.markPublished(entry.eventId());
            } catch (Exception ex) {
                log.warn(
                        "outbox publish failed, eventId={}, type={}",
                        entry.eventId(), entry.eventType(), ex
                );
                // 保留 PENDING，下一轮重试
            }
        }
    }
}
```

两个要额外强调的边界：

```text
发送失败保留 PENDING：下一轮重试，直到投递成功
发送成功但标记失败：会重复发送，消费端必须幂等
同步等待 Broker 确认：换可靠性，代价是吞吐和调度线程占用，
  等待时长与批量都必须按业务窗口配置，不能照抄示例值
```

具体注解语法、调度实现和确认 API 随所用框架和 MQ 客户端版本变化，实验时按实际依赖核证。

**徒弟：**

> 如果 Publisher 发送成功但标记失败，重试不就重复发了吗？

**老陈：**

> 会。所以消费端必须幂等。Outbox 用 at-least-once 换取可靠，把重复问题交给下游业务层解决。这也正是下一层的主题。

### Transfer：Outbox 的迁移

1. DB + ES 的索引刷新可以使用 Outbox 模式。
2. DB + 缓存失效通知可以使用 Outbox 模式。
3. 任何“本地事务完成后必须触发外部动作”的场景都适用。

本层留下的新账：**事件至少投递一次，重复必然出现。**消费端如何识别重复、保证业务只生效一次？下一层进入幂等。

### Level Ending

```text
口诀：状态和事件同事务，投递由后台补可靠。
```

---

<a id="level-3"></a>

## Level 3：重复投递与幂等消费

一句话认知墙：**重复无法靠 MQ 消除，只能靠消费端的业务幂等。**

### 前置知识关卡

- [ ] 知道 at-least-once 会重复投递
- [ ] 能说出 Outbox 重试的重复窗口
- [ ] 理解 ack 与业务成功是两件事

### Why：为什么不能依赖 Broker 去重

**徒弟：**

> MQ 有消息 ID，还有 ACK，Consumer 收到后处理，处理完再 ack，不就自然幂等了吗？

**老陈：**

> 处理完再 ack 只能减少窗口，不能消除重复：
>
> ```text
> 处理成功 -> ack 前进程崩溃 -> 重新投递
> 处理成功 -> ack 超时 -> 重新投递
> Rebalance -> 已处理消息再次分配
> 位点回拨 -> 一段消息重新消费
> ```
>
> 这些重复都发生在“业务已经成功”之后。Broker 无法知道你的业务结果，只能把重复交给业务幂等处理。

### What：两种幂等设计

| 方案 | 思路 | 代价 | 适用 |
| --- | --- | --- | --- |
| 事件 ID 去重 | 用 `eventId` 查已处理表/唯一约束 | 需要记录处理历史或依赖约束 | 大多数事件 |
| 业务幂等键 | 用业务唯一键（如策略版本、订单号）保证结果 | 需要真实业务键 | 业务动作有自然键 |
| Inbox + 去重约束 | 接收侧记录 eventId 再处理 | 与 Outbox 对偶，额外存储 | 需要强顺序或重复防御 |

消费处理逻辑示意：

```java
@Transactional
public void onStrategyPublished(StrategyPublishedIntegrationEvent event) {
    if (processedEvents.exists(event.eventId())) {
        return; // 已处理，重复投递直接忽略
    }
    // 处理：刷新读模型或执行本地业务
    strategyReadModelStore.refresh(
            event.strategyId(),
            event.scene(),
            event.strategyVersion()
    );
    processedEvents.mark(event.eventId());
}
```

这段代码要回答三个问题：

```text
幂等键是什么：eventId
“处理成功”的标志：processedEvents 记录与读模型刷新在同一事务
重复时跳过：第二次进来直接 return
```

**禁止决策：**

```text
用“消息只发一次”的假设省略幂等处理。
```

### How：幂等键粒度怎么选

```text
事件 ID：适合所有事件，成本是额外记录/表
业务键：适合有自然唯一性的业务事实
例如：
  StrategyPublished -> eventId + (strategyId, version) 唯一
  RecommendationExposed -> 曝光 ID + userId/requestId 组合
  UserClickedItem -> 点击事件 ID
```

曝光类事件如果没有自然业务键，可以组合：

```text
userId + requestId + itemId + exposedAt
```

组合键要能稳定表达“这是同一次曝光的同一条记录”，不要用随机数拼凑。

业务幂等键示例：

```java
@Transactional
public void onStrategyPublished(StrategyPublishedIntegrationEvent event) {
    boolean exists = readModelStore.existsByStrategyIdAndVersion(
            event.strategyId(),
            event.strategyVersion()
    );
    if (exists) {
        return;
    }
    readModelStore.refresh(
            event.strategyId(),
            event.scene(),
            event.strategyVersion()
    );
}
```

这个写法省掉了 `processedEvents` 表，但有一个必须补上的前提：

```text
“先查后写”本身不是并发保护。
两个重复事件同时进来，都可能查到 exists=false。
业务键必须配唯一约束，或者用原子的 upsert 语义，
让第二次写入不可能覆盖成功。
```

例如在 `strategy_id + strategy_version` 上建唯一约束，第二次刷新会冲突而被幂等跳过，而不是重复执行。

**徒弟：**

> 幂等处理后，重复消息对业务没有任何影响吗？

**老陈：**

> 对业务结果没有影响，但会消耗流量、线程和去重查询。幂等是正确性底线，不是性能优化。流量异常升高仍然要告警和排查。

### Transfer：幂等的迁移

1. 支付回调用支付流水号去重。
2. 库存扣减用订单号和库存单元组合去重。
3. 搜索引擎刷新用事件 ID 去重。

本层留下的新账：**事件可以幂等处理，但事件格式本身会变化。**新增字段、改枚举、改类型，旧消费者怎么办？下一层进入事件版本演进。

### Level Ending

```text
口诀：重复是事实，幂等是设计。
```

---

<a id="level-4"></a>

## Level 4：事件 Schema 版本如何演进

一句话认知墙：**事件字段一旦被多个消费者使用，演进就必须考虑“旧消费者还活着”。**

### 前置知识关卡

- [ ] 能区分 schemaVersion 和业务版本
- [ ] 知道消费者升级和生产者升级不会同步完成
- [ ] 理解兼容策略由消费端事实决定

### Why：字段想加就加为什么危险

**徒弟：**

> 事件多一个字段，生产者先发，消费者都升级就完事了，能出什么问题？

**老陈：**

> 问题是“都升级”不会同时发生：
>
> ```text
> 新生产者发布新字段 -> 旧消费者还在运行
> 旧消费者反序列化失败 -> 事件进入死信
> 新消费者先上线 -> 字段为 null 时行为未定义
> 枚举新增 -> 旧 switch 没有分支
> ```
>
> 版本演进不是“加字段”，而是管理新旧共存窗口。

### What：版本字段不要混用

```text
schemaVersion：契约结构版本，用于兼容判断
strategyVersion：业务版本，表达业务变化
eventId：本次事件实例唯一标识
```

集成事件带 `schemaVersion`：

```java
public record StrategyPublishedIntegrationEvent(
        String strategyId,
        String scene,
        int strategyVersion,
        int schemaVersion,
        Instant publishedAt
) {
}
```

### How：兼容策略

| 变化类型 | 兼容判断 | 处理方式 |
| --- | --- | --- |
| 新增可选字段 | 通常向后兼容 | 新字段带默认值，旧消费者忽略 |
| 新增枚举值 | 旧消费者可能不认识 | 消费端提供未知值处理分支 |
| 修改字段类型 | 不兼容 | 升版本/双轨发布/新旧字段并存 |
| 删除字段 | 需要确认无人使用 | 停用周期 + 版本升级 |
| 重命名字段 | 不兼容 | 新旧字段并存过渡 |

生产上的常用过渡：

```text
生产者双写：新旧字段同时发布
消费者双读：优先读新字段，回退旧字段
全部消费者升级后：删除旧字段，升 schemaVersion
```

消费者双读的代码示意：

```java
public void onEvent(StrategyPublishedIntegrationEvent event) {
    String ownerType;
    if (event.schemaVersion() >= 2) {
        ownerType = event.ownerType();
    } else {
        ownerType = "unknown";
    }
    // 处理逻辑只依赖本地解析后的 ownerType
}
```

双读的纪律：

```text
旧版本事件缺失新字段 -> 用业务默认值，而不是让处理逻辑崩溃
新字段为 null -> 同样要有明确语义，不能和“未提供”混淆
switch/枚举分支 -> 补默认分支，防止未知值中断消费
```

**徒弟：**

> 用同一个 `version` 字段既表示 Schema 版本又表示业务版本，代码少一个字段不好吗？

**老陈：**

> 不好。业务版本随每次发布递增，Schema 版本只在契约结构变化时递增。合并后，消费者无法判断“这次事件是格式变了，还是业务内容变了”，兼容判断会全部失效。

### Transfer：版本演进的迁移

1. 数据库表加列和 API 加字段都要考虑旧客户端。
2. 配置格式演进要考虑旧配置读取。
3. 事件、RPC、API 和共享库都遵循同一套“兼容窗口”思想。

本层留下的新账：**新老版本可能共存，失败也可能长期存在。**重试到最后还是失败怎么办？下一层进入补偿、死信和对账。

### Level Ending

```text
口诀：旧消费者还在，演进就不是加字段那么简单。
```

---

<a id="level-5"></a>

## Level 5：消费失败、补偿与对账

一句话认知墙：**重试有上限，失败要有归属，最终一致要靠对账兜底。**

### 前置知识关卡

- [ ] 知道重试次数是有限资源
- [ ] 能区分重试、死信、补偿和回滚
- [ ] 理解对账是最后一道网

### Why：失败一直重试为什么不行

**徒弟：**

> 消费失败就重试，无限重试到成功为止，消息就不会丢了。

**老陈：**

> 无限重试会占用消费线程、阻塞后续消息、放大系统抖动。更重要的是：有些失败不是暂时的，例如永久性数据错误、非法字段或业务永远不满足的条件。重试到成功是幻觉，需要把失败分类。

### What：失败分类

| 失败类型 | 例子 | 处理 |
| --- | --- | --- |
| 暂时失败 | 网络抖动、DB 连接超时 | 重试 |
| 永久数据失败 | 字段非法、契约损坏 | 死信 + 归因修复 |
| 业务条件不满足 | 依赖状态未就绪 | 重试 + 补偿/人工 |
| 超时不可判定 | 未知是否已处理 | 幂等 + 对账 |
| 积压失败 | 消费能力不足 | 扩容/降级/隔离 |

### How：补偿事务和数据库回滚不是一回事

分布式跨上下文动作不能靠数据库回滚收回。例如策略发布后，决策上下文刷新读模型失败：

```text
错误想法：把策略发布事务回滚掉
正确做法：策略发布已成功，刷新失败进入重试/死信/对账
```

补偿动作示例：

```text
策略下线通知失败
  -> 重试
  -> 仍失败 -> 死信
  -> 运维/自动任务检查
  -> 按策略重发或人工下线
```

补偿动作本身要幂等：

```text
扣积分失败 -> 补偿发积分 -> 补偿也可能重复
```

### How：对账兜底

```text
① 定义对账周期和主键（如策略版本）
② 比较源状态和目标状态
③ 发现差异
④ 生成补偿任务
⑤ 补偿也要幂等
```

策略发布对账示例：

```text
源：策略表状态 = PUBLISHED, version = 3
目标：读模型 PublishedStrategyView.version = 2
差异 -> 触发刷新读模型补偿任务
```

对账不是替代实时链路，而是覆盖“谁都不知道失败”的窗口。对账周期、差异阈值和补偿任务积压必须按业务容忍度设置。

对账任务代码示意。调度周期由业务允许窗口决定，不写硬编码 cron；具体调度注解按所用框架核证：

```java
@Component
final class StrategyPublishReconciler {
    private final StrategySourceQuery source;
    private final PublishedStrategyViewStore target;
    private final CompensationService compensation;

    @Scheduled(fixedDelayString = "${reconcile.strategy.delay-ms}")
    public void reconcile() {
        List<StrategyDiff> diffs = source.findDiffs(target);
        for (StrategyDiff diff : diffs) {
            compensation.refreshReadModel(diff.strategyId());
        }
        metrics.gauge("reconcile.diff_count", diffs.size());
    }
}
```

三个必须记住的纪律：

```text
对账本身要能持续运行：一次扫不完就分批，不能压垮源库和读模型
补偿任务要幂等：刷新读模型重复执行结果一致
差异数要变成指标：持续为 0 或持续增长都要有解释
```

**徒弟：**

> 死信队列里的事件堆满后怎么办？

**老陈：**

> 死信不是终点，是待处置的异常件。需要归因、修复、重放、过期和告警。堆满说明某类事件有系统性缺陷，要回到事件源头修复，而不是把死信当垃圾桶。

### Transfer：补偿的迁移

1. 订单超时未支付 -> 自动取消 + 恢复库存。
2. 发票生成失败 -> 重试/人工重开。
3. 搜索结果刷新失败 -> 对账重建索引。

本层留下的新账：**机制都齐了，最终一致到底“最终”多久、怎么验收？**下一层进入生产验证。

### Level Ending

```text
口诀：重试有限度，失败有归属，对账是底线。
```

---

<a id="level-6"></a>

## Level 6：最终一致性的生产验证

一句话认知墙：**最终一致不是“随便多久”，而是明确的时间窗口和可验证的业务结果。**

### 前置知识关卡

- [ ] 能说出 Outbox、幂等、版本和补偿的各自职责
- [ ] 能画出事件从聚合到消费者的链路
- [ ] 理解每个环节都可能延迟或失败

### Why：最终一致为什么需要验收

**徒弟：**

> 最终一致就是允许延迟，最终会一致，还需要验证什么？

**老陈：**

> “最终”必须有个时间边界。策略发布了 10 分钟，在线推荐还在用旧策略，这算一致吗？如果业务不允许，这就是事故。最终一致性需要明确：允许多久、怎么观测、怎么恢复。

### What：一条事件链路的完整验收

```text
策略发布
  -> 本地事务提交
  -> Outbox 记录
  -> Publisher 投递
  -> Broker 存储/复制
  -> 消费方接收
  -> 幂等检查
  -> 业务处理
  -> 读模型/状态更新
```

每个环节都有指标：

| 环节 | 指标 | 告警依据 |
| --- | --- | --- |
| 发布 | outbox_pending | 积压超过恢复预算 |
| 投递 | publish_error_rate | 高于正常基线 |
| 消费 | event_consume_lag | 持续增长或超过业务窗口 |
| 幂等 | duplicate_event_count | 非预期增长 |
| 业务处理 | 处理结果和失败原因 | 分类告警 |
| 一致性 | 对账差异数 | 出现差异即触发补偿 |
| 延迟 | 端到端事件延迟 | 超过业务允许窗口 |

### How：故障演练场景

```text
① Outbox 投递失败 -> 确认重试和积压指标
② 消费方重复收到同一事件 -> 确认幂等结果
③ 消费方处理失败 -> 确认重试/死信/对账
④ 事件 Schema 升级 -> 确认新旧消费者共存
⑤ Broker 短暂不可用 -> 确认发布不阻塞、积压恢复
⑥ 对账差异 -> 确认补偿任务生效
```

**最终一致性验收标准：**

```text
业务窗口内一致：达标
超窗口但可恢复：可接受，需要告警
无法自动恢复：需要人工介入
无观测手段：不可上线
```

**徒弟：**

> 如果某条链路既不想加 Outbox，又不想做对账，能靠日志查吗？

**老陈：**

> 日志能辅助排查，不能保证恢复。没有可靠发布和幂等，就没有最终一致的工程保证；没有对账，就无法确认“最终”何时到达。日志不是兜底机制，是观测材料。

### Transfer：最终一致的迁移

1. 缓存与数据库的最终一致需要过期策略和补偿。
2. 搜索索引与主库的最终一致需要重放和对账。
3. 跨系统状态同步需要明确窗口、观测和恢复手段。

本篇完成了从领域事件到最终一致的闭环。下一篇进入 DDD 的生产决策：什么时候不做 DDD、核心子域判断矩阵、何时从模块化单体拆微服务，并总结全系列。

### Level Ending

```text
口诀：最终一致要有时间窗口，还要有人盯着它。
```

---

# 🧪 合书自测

<a id="self-test"></a>

## 完整时间线

<a id="timeline"></a>

```text
T0  策略聚合执行 publish()
    |
    v
T1  本地事务：保存策略 + 写 Outbox
    |
    v
T2  事务提交
    |
    v
T3  Publisher 拉取 PENDING 记录
    |
    v
T4  投递 StrategyPublishedIntegrationEvent
    |
    v
T5  Broker 存储/复制/投递给消费方
    |
    v
T6  消费方幂等检查
    |
    v
T7  业务处理并刷新读模型
    |
    v
T8  失败重试/死信/对账/补偿
```

## 自测问题

| 问题 | 必须回答的不变量 |
| --- | --- |
| 领域事件和集成事件的区别？ | 领域事件管本地事实，集成事件管跨上下文契约 |
| DB 和 MQ 之间有没有原子事务？ | 没有，双写只能选方向并留兜底 |
| Outbox 解决什么？ | 让“状态变更”和“待发事件”在本地事务中一起提交 |
| Publisher 标记 PUBLISHED 说明什么？ | 只说明已交给 Broker，不说明消费成功 |
| 重复为什么无法靠 MQ 消除？ | ack 前后都存在业务已成功的重投窗口 |
| 幂等键怎么选？ | eventId 或业务自然键，保证重复处理结果相同 |
| schemaVersion 和业务版本能合并吗？ | 不能，前者描述契约结构，后者描述业务变化 |
| 无限重试为什么不对？ | 会放大故障且无法处理永久失败 |
| 补偿和数据库回滚是一回事吗？ | 不是，跨上下文动作靠业务补偿，且补偿要幂等 |
| 最终一致怎么验收？ | 明确时间窗口、指标、对账和恢复手段 |

---

# ⚠️ 坑与细节

<a id="pitfalls"></a>

## 坑 1：把领域事件直接序列化发给外部

错误理解：`StrategyPublished` 对象直接 JSON 化发到 MQ。

原因：把本地模型和跨上下文契约混在一起。

比喻后果：窗口把内部档案原件塞进信封寄给所有窗口，内部一改格式，所有收件人都要重新理解。

线上现象：领域类改名/加字段导致所有消费者回归。

修正：领域事件在边界处翻译成集成事件契约。

---

## 坑 2：事务方法里直接调 MQ

错误理解：`@Transactional` 里 `kafkaTemplate.send()` 就是可靠发布。

原因：数据库事务管不到消息系统，双写没有原子性。

比喻后果：档案落了笔，通知却不一定寄出；或者通知寄出了，档案又撤回了。

线上现象：DB 有状态、MQ 没消息，或反之。

修正：Outbox 同事务登记，后台可靠投递。

---

## 坑 3：相信“发一次就不会重复”

错误理解：消息去重是 Broker 的默认承诺。

原因：at-least-once 才是工程现实，重复来自 ack 窗口、rebalance 和位点回拨。

比喻后果：通知专员可能把同一份通知送两遍，收件窗口不能假设只收到一遍。

线上现象：曝光记两次、读模型刷新两次。

修正：消费端业务幂等，不依赖 Broker 承诺。

---

## 坑 4：只查 eventId 但不把它和业务处理放同一事务

错误理解：先查去重表，再处理，两者分开提交。

原因：去重标记和业务处理之间出现窗口。

比喻后果：登记了“已收到”却还没办理，重试时被跳过。

线上现象：消息丢失或处理半截。

修正：去重标记和业务处理在同一本地事务提交。

---

## 坑 5：把 schemaVersion 和业务版本混用

错误理解：一个 `version` 字段两用更简洁。

原因：契约结构和业务内容变化节奏不同。

比喻后果：收件人分不清“表格格式改了”还是“业务内容变了”。

线上现象：兼容判断失效，旧消费者误处理新事件。

修正：schemaVersion 和业务版本分开，独立演进。

---

## 坑 6：新增枚举值没有未知值处理

错误理解：新增枚举后所有消费者都会同步升级。

原因：消费者升级有先后，旧消费者会收到不认识的值。

比喻后果：窗口收到没见过的业务代码，直接拒办。

线上现象：未知枚举导致消费失败、死信堆积。

修正：消费端提供未知值处理分支，再灰度升级。

---

## 坑 7：无限重试放大故障

错误理解：失败就重试，直到成功。

原因：重试是有限资源，会阻塞队列并放大后端压力。

比喻后果：窗口反复拨打一个永远打不通的电话，后面排队的申请全部等待。

线上现象：消费线程被占满，积压快速上涨。

修正：重试上限 + 死信 + 归因 + 对账补偿。

---

## 坑 8：把死信队列当垃圾桶

错误理解：进死信就算处理完了。

原因：死信是待处置的异常件，不是终点。

比喻后果：异常件堆在仓库无人处理，业务差异长期存在。

线上现象：死信积压，对账差异增长，最终一致性失效。

修正：死信必须归因、修复、重放、过期和告警。

---

## 坑 9：没有对账就宣称最终一致

错误理解：有重试和死信就足够。

原因：还有“谁都不知道失败”的窗口。

比喻后果：通知被风吹走了，寄件人和收件人都以为正常。

线上现象：DB 状态和读模型/下游长期不一致。

修正：对账比对源和目标，生成补偿任务，差异触发告警。

---

## 坑 10：把日志当兜底机制

错误理解：出问题查日志就能恢复。

原因：日志是观测材料，不能自动恢复状态。

比喻后果：监控录像能看过程，不能把办错的手续自动改回来。

线上现象：事件丢失后只能人工逐条补数据。

修正：可靠发布 + 幂等 + 对账 + 补偿构成恢复链路，日志用于定位。

---

# 📊 竖切总表

| 时间 | 位置 | 动作 | 不变量 | 风险 |
| --- | --- | --- | --- | --- |
| T0 | 策略聚合 | 产生领域事实 | 领域对象不外发 | 直接序列化领域对象 |
| T1 | 应用层 | 本地事务保存 + Outbox | 状态与事件同事务 | 事务里直接发 MQ |
| T2 | Publisher | 投递集成事件 | 至少一次投递 | 重复投递 |
| T3 | Broker | 存储/复制/投递 | 不承诺恰好一次 | 位点回拨 |
| T4 | 消费方 | 幂等检查 + 处理 | 去重与业务同事务 | 去重和处理分离 |
| T5 | 版本层 | Schema 兼容演进 | 旧消费者共存 | 新增枚举无处理 |
| T6 | 失败治理 | 重试/死信/补偿 | 失败有归属 | 无限重试 |
| T7 | 对账 | 源目标比对 | 差异触发补偿 | 无对账宣称一致 |

---

# 📚 概念勘误表

<a id="errata"></a>

| 错误说法 | 准确说法 |
| --- | --- |
| 领域事件就是发到 MQ 的消息 | 领域事件是本地业务事实，集成事件才是跨上下文契约 |
| 事务里发消息是可靠发布 | DB 和 MQ 没有原子事务，Outbox 用本地事务先登记 |
| MQ 有 ACK 就不会重复 | ack 前后都存在重投窗口，重复靠消费端幂等 |
| 处理完再 ack 就是恰好一次 | 只能缩小窗口，业务已成功后的重投无法靠 ack 消除 |
| 一个 version 字段可以两用 | schemaVersion 和业务版本表达不同变化，必须分开 |
| 无限重试保证最终成功 | 重试是有限资源，永久失败需要死信和补偿 |
| 补偿就是数据库回滚 | 跨上下文动作靠业务补偿，补偿本身要幂等 |
| 死信队列是处理终点 | 死信是待处置异常件，需要归因和重放 |
| 最终一致等于允许无限延迟 | 最终一致需要明确时间窗口和验收标准 |
| 日志可以代替对账 | 日志是观测材料，对账才是恢复机制 |

其中两点我以前也讲得过于绝对：

```text
我以前容易把 Outbox 讲成“消息不丢的保证”；准确说法是它保证“状态和待发事件同事务”，投递和消费仍可能失败。
我以前容易把“最终一致”讲成“自动恢复”；准确说法是最终一致依赖幂等、补偿和对账共同成立。
```

---

# 🏆 生产决策卡

<a id="decisions"></a>

## Decision Card 1：要不要用 Outbox

### 场景

策略发布后需要通知在线决策、实验和反馈上下文。

### 判断

如果“状态变更”和“事件发送”必须同生共死，使用 Outbox；如果允许事件延迟且可以接受定期对账，才考虑更轻的方案。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 状态和事件必须同事务 | Outbox 模式 |
| 事件允许延迟、量小 | 可评估轻量方案 + 对账 |
| 高吞吐日志流水 | 不需要业务 Outbox（回链 mq 支线隔离） |
| 事件有审计/回溯要求 | Outbox + 事件留存 |

### Code

```text
@Transactional
save(strategy);
outbox.append(StrategyPublishedIntegrationEvent.from(strategy));
```

### 禁止决策

```text
在事务方法里直接调 MQ，然后宣称事件不会丢。
```

### 验收指标

```text
outbox_pending
publish_error_rate
end_to_end_event_latency
event_consume_lag
duplicate_event_count
```

---

## Decision Card 2：幂等键用 eventId 还是业务键

### 场景

`StrategyPublished` 和 `RecommendationExposed` 都可能重复投递。

### 判断

有自然业务键的事件优先用业务键 + eventId 双保险；无自然键的曝光类事件用事件 ID 或组合键。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 有自然业务键 | 业务键唯一约束 + eventId 记录 |
| 无自然键 | eventId 去重表 |
| 需要强顺序/接收防御 | Inbox 模式 |
| 补偿任务 | 补偿动作同样幂等 |

### Code

```text
processedEvents.mark(event.eventId()) + 业务处理同一事务
strategyReadModelStore.refresh(...) 幂等更新
```

### 禁止决策

```text
依赖 Broker 去重，消费端不做幂等。
```

### 验收指标

```text
duplicate_event_count
幂等冲突数
业务重复执行数
event_consume_lag
Error rate
```

---

## Decision Card 3：事件 Schema 如何演进

### 场景

`StrategyPublishedIntegrationEvent` 需要新增字段并支持多个消费者版本共存。

### 判断

先加可选字段并给默认值，消费者双读，全部升级后再升 schemaVersion 并删除旧字段。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 新增可选字段 | 向后兼容，直接加默认值 |
| 新增枚举值 | 消费端先支持未知值处理 |
| 类型变更/删除字段 | 双轨发布 + 双读过渡 |
| 结构变化完成 | 升级 schemaVersion |

### Code

```text
生产者：新旧字段双写
消费者：优先新字段，回退旧字段
稳定后：删旧字段，schemaVersion+1
```

### 禁止决策

```text
在没有任何兼容窗口的情况下直接改字段类型或删除字段。
```

### 验收指标

```text
contract_decode_error_rate
旧消费者处理新事件失败数
schemaVersion 分布
事件兼容测试场景数
Error rate
```

---

## Decision Card 4：失败重试、死信还是补偿

### 场景

决策上下文消费 `StrategyPublished` 后刷新读模型失败。

### 判断

暂时失败重试，永久失败进死信，业务窗口超时进入对账补偿。重试次数和间隔按业务窗口设置，不无限重试。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 网络/DB 抖动 | 有限重试 |
| 字段非法/永久失败 | 死信 + 归因修复 |
| 业务条件未满足 | 补偿任务/人工 |
| 无法确认是否处理 | 幂等 + 对账 |
| 积压增长 | 扩容/降级/隔离 |

### Code

```text
onStrategyPublished(event):
  已处理 -> return
  处理成功 -> markProcessed
  重试上限 -> 死信
  对账发现差异 -> 补偿刷新读模型
```

### 禁止决策

```text
无限重试到成功，死信不处置，不对账。
```

### 验收指标

```text
retry_count
dlq_depth
对账差异数
补偿任务积压
event_consume_lag
Error rate
```

---

# 🌍 跨语言视角

<a id="cross-language"></a>

## Java

```text
@Transactional + Outbox 表
record 表达集成事件契约
去重表/唯一约束表达幂等
对账任务比对源和目标
```

## Go

```text
数据库事务内写 outbox 表
struct 表达事件契约
消费者用业务键/事件 ID 去重
定时对账任务
```

## Rust

```text
数据库事务内写 outbox 表
struct/enum 表达事件契约
去重表/唯一约束
补偿任务状态机
```

## 跨语言仍成立的判断力

```text
数据库和消息系统没有原子事务。
可靠发布靠本地事务登记，不靠发送调用成功。
重复靠业务幂等，不靠 Broker 承诺。
Schema 演进必须照顾旧消费者。
最终一致需要时间窗口、指标、对账和补偿。
```

---

# 系列承接：下一篇留下的问题

本篇已经回答：

```text
领域事件和集成事件如何区分
Outbox 如何绑定本地事务
重复如何靠幂等消化
事件 Schema 如何演进
失败如何重试、补偿和对账
最终一致如何验收
```

但整条 DDD 系列还需要一次收口：什么业务值得用 DDD，什么业务应该用 CRUD 或读模型；核心子域如何判断；什么时候从模块化单体拆成微服务。

这就是 `ddd-06-生产决策与选择性DDD.md` 的入口。
