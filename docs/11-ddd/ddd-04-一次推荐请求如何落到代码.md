# DDD 代码落地：一次推荐请求如何穿过 Controller、Domain 和基础设施

> 这不是一篇分层架构模板介绍。
>
> 本文只追踪一个主线：**推荐请求 R——用户打开首页后，系统生成一组个性化商品推荐。**
>
> `ddd-03` 已经把策略聚合、值对象和仓储边界立起来了。现在把 R 放回真实代码：
>
> ```text
> HTTP/RPC
>   -> Controller
>   -> Application Handler
>   -> Domain Policy / Read Model
>   -> Redis / Model / MQ Adapter
>   -> Response
> ```
>
> 本文要回答的不是“每层叫什么”，而是：**每层应该保护什么变化，谁可以调用谁，事务和安全边界放在哪里。**

# 🏷️ 关键词与主线进度

关键词：

```text
Controller | Application Handler | Domain Policy | Repository
Port | Adapter | Redis | Model Service | MQ
Transaction | Security | Logging | ArchUnit | Java Module
模块化单体 | 千人千面
```

主线进度：

```text
① ddd-03 留下的代码落地问题
② Controller 为什么不能承载用例
③ Application Handler 如何编排一次请求
④ Domain 如何保持纯度
⑤ Port/Adapter 如何接入 Redis、模型和 MQ
⑥ ArchUnit 与 Java Module 如何守住边界
⑦ 运行时、事务、安全和生产验证
```

---

# ⚠️ 版本与证据边界

| 类型 | 本文承诺 |
| --- | --- |
| Specification | DDD 不规定唯一包结构，也不规定必须使用 Spring、JPA 或 ArchUnit |
| Implementation | Java/Spring 代码是本系列示例；具体框架行为按版本实验核证 |
| Version | Java 代码按 JDK 21 表达；Spring Boot 3.3.x 作为现有知识体系的适配基线 |
| Need Benchmark | 本文没有性能数字；推荐链路延迟、线程和连接池容量需要实测 |
| Design Assumption | 采用模块化单体先验证边界，后续才评估独立部署 |

特别区分：

```text
ArchUnit：运行时执行的架构测试，不是 Java 编译器规则
Java Module：编译期模块可见性和依赖约束
Package 约定：团队代码组织纪律，单独使用时约束最弱
```

---

# 🗂️ 目录

- [Level 1：Controller 为什么不是业务入口](#level-1)
- [Level 2：Application Handler 如何编排请求](#level-2)
- [Level 3：Domain、Repository 和读模型如何连接](#level-3)
- [Level 4：Port/Adapter 如何接入外部技术](#level-4)
- [Level 5：ArchUnit 与 Java Module 如何守边界](#level-5)
- [Level 6：事务、安全、失败和生产验证](#level-6)
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
| Level 1 | “Controller 调 Service 就是架构” | 能区分协议入口和业务用例入口 |
| Level 2 | “Application 层就是万能 Service” | 能说明编排、事务、安全和业务规则的边界 |
| Level 3 | “Domain 必须加载所有聚合” | 能区分写模型聚合和在线读模型 |
| Level 4 | “Port 只是接口改名” | 能展示 Redis、模型、MQ 的具体适配器 |
| Level 5 | “分包就能守住边界” | 能区分 ArchUnit 测试和 Java Module 编译约束 |
| Level 6 | “代码能跑就完成落地” | 能用事务、超时、降级、指标和故障验证链路 |

---

# 全文唯一比喻：一份业务申请在多个专业窗口办理

## ASCII Diagram

```text
用户提交推荐申请 R
        |
        v
+-------------------+
| 接待窗口          | 接收请求、身份和场景
+-------------------+
        |
        v
+-------------------+
| 编排窗口          | 按用例安排办理顺序
+-------------------+
        |
        v
+-------------------+
| 规则窗口          | 判断结果是否符合业务规则
+-------------------+
        |
        v
+-------------------+
| 外部协作窗口      | 连接画像、模型、消息系统
+-------------------+
        |
        v
+-------------------+
| 回执窗口          | 返回结果并记录业务事实
+-------------------+
```

## Mapping Table

| 比喻元素 | 技术概念 | 唯一职责 |
| --- | --- | --- |
| 接待窗口 | Interface/Controller | 把外部协议转换成用例输入 |
| 编排窗口 | Application Handler | 组织一次业务用例的步骤 |
| 规则窗口 | Domain Model/Policy | 保护业务规则和不变量 |
| 外部协作窗口 | Port/Adapter | 把技术能力接入内部模型 |
| 回执窗口 | Response/Event Publisher | 返回结果或发布业务事实 |
| 窗口之间的内部通道 | Dependency Direction | 限制谁可以依赖谁 |

比喻只用于理解职责，不能把每个窗口机械等同于一个 Java 包或微服务。

---

# 正文：六个 Level 打穿一次请求的代码链

<a id="level-1"></a>

## Level 1：Controller 为什么不是业务入口

一句话认知墙：**Controller 是协议入口，不应该成为业务规则的总入口。**

### 前置知识关卡

- [ ] 知道 HTTP 请求会经过反序列化和参数校验
- [ ] 知道 ddd-03 中的策略规则由领域模型保护
- [ ] 能区分接口协议和业务用例

### Why：Controller 里写业务会留下什么账

**徒弟：**

> Controller 里直接查 Redis、调模型、过滤商品，离请求最近，写起来最快。

**老陈：**

> 它确实能快速交付接口，但 Controller 会同时知道 HTTP、用户身份、缓存 Key、模型参数和推荐规则。以后换成 RPC、消息消费或批处理入口时，你要复制一套业务逻辑。

常见失控路径：

```text
Controller
  -> 解析参数
  -> 校验权限
  -> 查 Redis
  -> 查数据库
  -> 调模型
  -> 写日志
  -> 发 MQ
  -> 拼响应
```

| 写法 | 解决什么 | 留下什么问题 |
| --- | --- | --- |
| Controller 直接编排 | 开发快 | HTTP 和业务规则绑死 |
| Service 接收所有对象 | 入口变薄 | Service 变成上帝对象 |
| 每个入口复制流程 | 各入口独立 | 规则修改后出现分叉 |

### What：接口入口和用例入口分开

```text
Controller：我收到什么协议数据？
Application Handler：这次业务要按什么顺序办？
Domain：什么情况下允许这样办？
Adapter：具体调用哪种技术？
```

一个推荐请求的边界：

```text
HTTP GET /recommendations
  -> GetRecommendationsQuery
  -> GetRecommendationsHandler
  -> RecommendationPolicy
  -> FeatureProviderPort / RankerPort
  -> RecommendationResponse
```

### How：Controller 只做协议转换

```java
@RestController
final class RecommendationController {
    private final GetRecommendationsHandler handler;

    @GetMapping("/recommendations")
    RecommendationResponse get(
            @RequestParam String userId,
            @RequestParam String scene,
            Authentication authentication
    ) {
        Actor actor = Actor.from(authentication);
        GetRecommendationsQuery query =
                new GetRecommendationsQuery(userId, scene);
        return handler.handle(actor, query);
    }
}
```

这里的 `Authentication` 是接口适配对象，不能直接传进领域层。Controller 只负责：

```text
解析协议
转换输入
提取身份
调用应用用例
转换输出
```

**徒弟：**

> 参数校验也不能放 Controller 吗？

**老陈：**

> 格式校验可以放入口，例如字符串是否为空；业务校验不能只放入口，例如用户是否允许个性化、策略是否生效。格式是协议问题，业务规则属于领域问题。

### Transfer：入口和用例分离的迁移

1. HTTP、RPC 和消息消费者可以共享同一个 Application Handler。
2. 同一个用例可以被定时任务或后台管理入口调用。
3. 测试可以绕过网络层直接验证业务用例。

本层留下的新账：**Controller 变薄了，但谁负责把一次请求按正确顺序组织起来？**下一层进入 Application Handler。

### Level Ending

```text
口诀：Controller 接协议，Handler 接用例。
```

---

<a id="level-2"></a>

## Level 2：Application Handler 如何编排请求

一句话认知墙：**Application 层负责“怎么办”，Domain 层负责“允许不允许”。**

### 前置知识关卡

- [ ] 能说出 Controller 不应该承担哪些职责
- [ ] 知道推荐请求需要画像、策略、候选和模型
- [ ] 理解事务不能无脑包住所有远程调用

### Why：万能 Service 为什么还会回来

**徒弟：**

> 那我把所有代码从 Controller 搬到 `RecommendationService`，这个 Service 就是 Application 层了吧？

**老陈：**

> 只有名字变了，责任没有变。Application Handler 应该编排一次用例，但不应该把“商品是否可推荐”“用户是否允许个性化”这些规则也写成一堆流程判断。

Application 层需要处理的变化：

```text
用例步骤顺序
事务边界
身份和权限入口
日志、追踪和指标
端口调用
异常转换和响应编排
```

Domain 层需要处理的变化：

```text
策略能否发布
候选是否符合业务规则
用户是否允许个性化
结果如何过滤和兜底
```

### What：Handler 的职责表

| Handler 可以负责 | Handler 不应该负责 |
| --- | --- |
| 校验当前用户是否能调用用例 | 直接修改领域对象字段 |
| 组织读取特征和策略的顺序 | 定义所有商品和画像规则 |
| 调用 Port 获取外部能力 | 创建 Redis/HTTP/MQ 客户端 |
| 记录用例耗时和失败 | 把第三方异常直接暴露给领域层 |
| 决定哪些动作同步/异步 | 用日志替代业务事实 |

### How：完整 Handler 链路

```java
final class GetRecommendationsHandler {
    private final RecommendationAuthorization authorization;
    private final PublishedStrategyQuery strategyQuery;
    private final FeatureProviderPort featureProvider;
    private final CandidateRecallPort recall;
    private final RankerPort ranker;
    private final RecommendationPolicy policy;
    private final ExposurePublisher exposurePublisher;

    RecommendationResponse handle(
            Actor actor,
            GetRecommendationsQuery query
    ) {
        authorization.check(actor, query);

        PublishedStrategyView strategy =
                strategyQuery.findFor(query.scene())
                        .orElseGet(PublishedStrategyView::fallback);

        FeatureSnapshot features =
                featureProvider.load(query.userId(), query.scene());

        CandidateSet candidates =
                recall.recall(query, features, strategy);

        RankedCandidates ranked =
                ranker.rank(candidates, strategy);

        RecommendationResult result =
                policy.decide(query, features, ranked, strategy);

        exposurePublisher.publish(
                RecommendationExposed.from(query, result)
        );

        return RecommendationResponse.from(result);
    }
}
```

这段 Handler 做的是流程编排：

```text
鉴权 -> 读策略 -> 读特征 -> 召回 -> 排序 -> 领域过滤 -> 发布曝光 -> 返回
```

它没有直接创建外部客户端，也没有把 `Authentication`、Redis Hash 或模型 SDK 对象传进领域层。

反面写法是把所有技术和业务都塞回一个 Service：

```java
// ❌ 示意：万能 Service 同时承担鉴权、存储、模型、业务规则和消息发送
final class RecommendationService {
    private final StringRedisTemplate redis;
    private final ModelHttpClient modelClient;
    private final KafkaTemplate<String, Object> kafka;

    RecommendationResponse recommend(String userId, String scene) {
        if (!checkPermission(userId)) {
            throw new ForbiddenException();
        }

        Map<Object, Object> features = redis.opsForHash()
                .entries("recommendation:features:" + userId);
        Strategy strategy = strategyDao.findByScene(scene);
        ModelResponse response = modelClient.rank(
                buildRequest(features, strategy)
        );
        List<Item> result = filter(response.getItems(), features);
        kafka.send("exposure", new ExposureEvent(userId, result));
        return new RecommendationResponse(result);
    }
}
```

它的问题不是“代码行数多”，而是每个变化都要修改同一个类：

```text
鉴权变化 -> 改 Service
Redis 结构变化 -> 改 Service
模型协议变化 -> 改 Service
过滤规则变化 -> 改 Service
MQ 可靠性变化 -> 改 Service
```

正面 Handler 把这些变化拆成授权组件、Query/Repository、Domain Policy 和 Port；Handler 只保留用例顺序。

### 事务、日志和安全放在哪里

```text
安全身份提取：Controller/接口适配层
用例权限判断：Application 层
业务资格判断：Domain 层
用例日志/Trace/Metrics：Application 层
状态写入事务：Application 层边界
规则正确性：Domain 层
```

推荐查询通常不需要为了调用 Redis 和模型服务而开启长事务：

```text
错误：开启数据库事务 -> 调 Redis -> 调模型 -> 发 MQ -> 返回
正确：短读事务或读模型查询 -> 外部调用按超时预算执行 -> 异步发布事实
```

如果是策略发布写用例，则由 Application Handler 开启本地事务，加载聚合、执行领域行为、保存聚合；这与在线推荐读路径不是同一条事务链。

**徒弟：**

> 那 Handler 里岂不是也有很多业务判断？

**老陈：**

> 流程判断可以有，业务不变量不能被 Handler 私有化。比如“没有发布策略就走兜底”是用例编排；“策略为什么可以发布”应该由策略聚合保护。

### Transfer：应用层编排的迁移

1. 支付 Handler 编排扣款、订单状态和通知，不自己定义支付合法性。
2. 审批 Handler 编排提交、审批人选择和结果发布，不自己修改审批状态。
3. 物流 Handler 编排库存、路线和通知，不把仓库内部状态直接传给外部。

本层留下的新账：**Handler 知道调用 Domain，但 Domain 如何访问 Repository 和外部能力而不依赖框架？**下一层进入领域端口和 Repository。

### Level Ending

```text
口诀：Application 编排流程，Domain 守住规则。
```

---

<a id="level-3"></a>

## Level 3：Domain、Repository 和读模型如何连接

一句话认知墙：**领域层可以依赖自己定义的接口，但不应该知道接口背后的 Redis、JPA 和 HTTP。**

### 前置知识关卡

- [ ] 能说出 Application 和 Domain 的责任区别
- [ ] 知道 ddd-03 中 Repository 以聚合为边界
- [ ] 理解在线查询不一定加载完整聚合

### Why：领域层直接依赖技术会失去什么

**徒弟：**

> 在 `RecommendationPolicy` 里直接注入 `RedisTemplate`，读取特征不是更方便吗？

**老陈：**

> 方便的是当前实现，失去的是替换和验证能力。领域规则开始依赖缓存 Key、序列化格式和网络异常；以后换特征平台，连业务测试都要启动 Redis。

### What：端口的三类职责

```text
Query Port：读取领域需要的事实或视图
Command Port：请求外部系统执行动作
Event Port：发布已经发生的业务事实
```

领域层只看接口：

```java
public interface FeatureProviderPort {
    FeatureSnapshot load(String userId, String scene);
}

public interface RankerPort {
    RankedCandidates rank(
            CandidateSet candidates,
            PublishedStrategyView strategy
    );
}

public interface ExposurePublisher {
    void publish(RecommendationExposed event);
}
```

策略写模型仍然使用 `StrategyRepository`：

```java
public interface StrategyRepository {
    Optional<RecommendationStrategy> findById(StrategyId id);

    void save(RecommendationStrategy strategy);
}
```

在线推荐查询可以使用单独的查询端口：

```java
public interface PublishedStrategyQuery {
    Optional<PublishedStrategyView> findFor(String scene);
}
```

这两个接口不要强行合成一个 Repository：

```text
StrategyRepository：保护聚合写模型
PublishedStrategyQuery：提供在线查询读模型
```

### How：Controller → Handler → Domain → Repository

推荐请求的热路径通常使用读模型，但策略发布写路径必须穿过聚合和仓储：

```java
@RestController
final class StrategyAdminController {
    private final PublishStrategyHandler handler;

    @PostMapping("/strategies/{id}/publish")
    void publish(@PathVariable UUID id, Authentication authentication) {
        handler.handle(
                new PublishStrategyCommand(new StrategyId(id)),
                Actor.from(authentication)
        );
    }
}
```

```java
final class PublishStrategyHandler {
    private final StrategyAuthorization authorization;
    private final StrategyRepository repository;

    @Transactional
    void handle(PublishStrategyCommand command, Actor actor) {
        authorization.check(actor, command);

        RecommendationStrategy strategy = repository.findById(command.id())
                .orElseThrow(() -> new StrategyNotFound(command.id()));

        strategy.publish();
        repository.save(strategy);
    }
}
```

```text
StrategyAdminController
  -> PublishStrategyHandler
  -> StrategyRepository.findById
  -> RecommendationStrategy.publish
  -> StrategyRepository.save
  -> 本地事务提交
```

这里的 `@Transactional` 属于应用层适配实现；领域层只知道“策略发布”这个行为，不知道事务代理如何工作。

### How：读路径的完整链路

写路径需要加载聚合并保存状态；在线读路径通常读取已经发布的视图，不为了查询重新加载完整聚合：

```java
@RestController
final class RecommendationReadController {
    private final GetRecommendationsReadHandler handler;

    @GetMapping("/recommendations")
    RecommendationResponse get(
            @RequestParam String userId,
            @RequestParam String scene,
            Authentication authentication
    ) {
        return handler.handle(
                Actor.from(authentication),
                new GetRecommendationsQuery(userId, scene)
        );
    }
}
```

```java
final class GetRecommendationsReadHandler {
    private final RecommendationAuthorization authorization;
    private final PublishedStrategyQuery strategyQuery;
    private final FeatureProviderPort featureProvider;
    private final CandidateRecallPort recall;
    private final RankerPort ranker;
    private final RecommendationPolicy policy;

    RecommendationResponse handle(
            Actor actor,
            GetRecommendationsQuery query
    ) {
        authorization.check(actor, query);

        PublishedStrategyView strategy = strategyQuery
                .findFor(query.scene())
                .orElseGet(PublishedStrategyView::fallback);
        FeatureSnapshot features = featureProvider
                .load(query.userId(), query.scene());
        CandidateSet candidates = recall
                .recall(query, features, strategy);
        RankedCandidates ranked = ranker.rank(candidates, strategy);
        RecommendationResult result = policy
                .decide(query, features, ranked, strategy);

        return RecommendationResponse.from(result);
    }
}
```

```java
final class PublishedStrategyQueryAdapter
        implements PublishedStrategyQuery {
    private final PublishedStrategyViewStore store;

    @Override
    public Optional<PublishedStrategyView> findFor(String scene) {
        return store.findPublishedByScene(scene);
    }
}
```

读路径的依赖关系是：

```text
RecommendationReadController
  -> GetRecommendationsReadHandler
  -> PublishedStrategyQuery / FeatureProviderPort
  -> RecommendationPolicy
  -> RecommendationResponse
```

它与写路径的区别必须保留：

```text
写路径：StrategyRepository -> RecommendationStrategy.publish() -> save
读路径：PublishedStrategyQuery -> PublishedStrategyView -> policy.decide()
```

**徒弟：**

> 既然在线推荐只读 `PublishedStrategyView`，是不是可以完全删除 `RecommendationStrategy` 聚合？

**老陈：**

> 不能。读模型负责快速回答“当前查到什么”，聚合负责保护“什么状态允许被写”。删除写模型后，策略发布规则会重新散落到后台接口和脚本中。

### Transfer：端口和读写分离的迁移

1. 支付领域通过 `PaymentGatewayPort` 接入不同支付渠道。
2. 搜索领域通过 `SearchIndexPort` 接入 ES 或其他搜索引擎。
3. 通知领域通过 `NotificationPort` 接入短信、邮件和站内信。

本层留下的新账：**端口只是抽象，真实系统还要有 Redis、模型服务和 MQ 的适配器。**下一层把这些技术细节放回基础设施边界。

### Level Ending

```text
口诀：领域拥有接口，适配器拥有技术。
```

---

<a id="level-4"></a>

## Level 4：Port/Adapter 如何接入 Redis、模型和 MQ

一句话认知墙：**适配器不是把第三方 SDK 原样暴露出去，而是把技术结果翻译成本地端口能理解的语义。**

### 前置知识关卡

- [ ] 能说出 FeatureProviderPort、RankerPort 和 ExposurePublisher 的职责
- [ ] 知道外部 DTO/SDK 对象不能直接进入领域层
- [ ] 理解适配器需要处理超时、空值和异常

### Why：直接把 SDK 放进业务代码会留下什么账

**徒弟：**

> Redis 读取就是 `opsForHash().entries()`，模型调用就是 `client.rank()`，直接写在 Handler 里不是更直观吗？

**老陈：**

> 直观的是 SDK，不是业务。这样写以后，Handler 会同时知道 Redis Key、模型请求 JSON、MQ topic 和业务返回结构。技术替换时，业务编排也要一起重写。

### What：适配器的转换边界

```text
Domain Port
  -> Adapter
  -> Third-party SDK / Network / Serialization
  -> Adapter converts result
  -> Domain-friendly type
```

适配器必须负责：

```text
Key 和 URL
序列化/反序列化
超时和异常转换
空值和非法值处理
重试边界
第三方对象到本地对象的转换
```

### How：Redis 特征适配器

```java
@Component
final class RedisFeatureProvider implements FeatureProviderPort {
    private final StringRedisTemplate redis;

    RedisFeatureProvider(StringRedisTemplate redis) {
        this.redis = redis;
    }

    @Override
    public FeatureSnapshot load(String userId, String scene) {
        String key = "recommendation:features:" + userId;
        try {
            Map<Object, Object> values = redis.opsForHash().entries(key);

            boolean allowed = Boolean.parseBoolean(
                    String.valueOf(values.getOrDefault(
                            "personalizationAllowed", "false"))
            );

            Set<String> interests = parseInterests(values.get("interests"));
            return new FeatureSnapshot(userId, scene, allowed, interests);
        } catch (RedisConnectionFailureException ex) {
            throw new FeatureUnavailable("redis connection failed", ex);
        }
    }
}
```

领域层不知道 Redis Key，也不接受 `Map<Object, Object>`。适配器把它转换成 `FeatureSnapshot`，并把 Redis 连接失败转换成内部的 `FeatureUnavailable`。具体异常类型和超时配置随 Redis 客户端版本变化，实验时按实际依赖核证。

### How：模型服务适配器

```java
@Component
final class HttpModelRanker implements RankerPort {
    private final ModelHttpClient client;

    HttpModelRanker(ModelHttpClient client) {
        this.client = client;
    }

    @Override
    public RankedCandidates rank(
            CandidateSet candidates,
            PublishedStrategyView strategy
    ) {
        ModelRequest request = ModelRequest.from(candidates, strategy);
        try {
            ModelResponse response = client.rank(request);
            return RankedCandidates.from(response);
        } catch (ModelTimeoutException ex) {
            throw new RankingUnavailable(ex);
        }
    }
}
```

超时如何降级由 Application/Domain 的业务策略决定，模型适配器只把第三方异常翻译成内部可识别的异常语义。

### How：MQ 曝光适配器

```java
@Component
final class KafkaExposurePublisher implements ExposurePublisher {
    private final KafkaTemplate<String, RecommendationExposed> kafka;

    KafkaExposurePublisher(
            KafkaTemplate<String, RecommendationExposed> kafka
    ) {
        this.kafka = kafka;
    }

    @Override
    public void publish(RecommendationExposed event) {
        kafka.send(
                "recommendation-exposed",
                event.userId(),
                event
        );
    }
}
```

这里仅展示端口到 MQ 客户端的适配关系，不宣称 `send` 返回就代表业务事件可靠完成。投递确认、Outbox、重复和补偿在 `ddd-05` 处理。

### Transfer：适配器的迁移

1. Redis 换成 Caffeine 或特征平台，只替换 `FeatureProviderPort` 实现。
2. 模型从 HTTP 换成 gRPC，只替换 `RankerPort` 适配器。
3. Kafka 换成 RocketMQ，只替换事件发布适配器和可靠性实现。

本层留下的新账：**代码边界可以靠约定，但团队会不会误用内部类、直接依赖框架？**下一层用 ArchUnit 和 Java Module 保护依赖方向。

### Level Ending

```text
口诀：SDK 留在适配器，语义回到领域内。
```

---

<a id="level-5"></a>

## Level 5：ArchUnit 与 Java Module 如何守边界

一句话认知墙：**目录结构只是提醒，自动化约束才会让错误依赖在代码评审之外也暴露出来。**

### 前置知识关卡

- [ ] 知道领域层、应用层和适配器的依赖方向
- [ ] 理解 ArchUnit 是测试而不是编译器
- [ ] 知道 Java Module 可以限制模块可见性

### Why：为什么只靠团队约定不够

**徒弟：**

> 代码评审时提醒大家不要在 Domain 里引入 Spring，不就够了吗？

**老陈：**

> 约定在项目小、人员固定时有效，但依赖会随着复制粘贴和临时排障逐渐穿透。架构约束要能自动失败，否则边界只能靠记忆维护。

### What：两类约束不要混淆

```text
ArchUnit
  运行测试时检查已编译的类依赖
  适合快速表达包级架构规则
  失败时阻止测试/构建通过

Java Module
  编译期定义 requires/exports
  适合限制模块可见性和依赖
  需要按模块组织构建
```

两者可以一起使用：

```text
Java Module：编译期先挡住一部分非法依赖
ArchUnit：测试更多业务包和上下文规则
```

### How：ArchUnit 架构测试示例

以下示例按 ArchUnit 常见 API 表达，具体依赖版本和包名需要在实验工程核证：

```java
@AnalyzeClasses(packages = "com.example.recommendation")
class ArchitectureRulesTest {

    @ArchTest
    static final ArchRule domain_must_not_depend_on_framework =
            noClasses()
                    .that().resideInAPackage("..domain..")
                    .should().dependOnClassesThat()
                    .resideInAnyPackage(
                            "org.springframework..",
                            "jakarta.persistence..",
                            "org.apache.kafka..",
                            "org.springframework.data.redis.."
                    );

    @ArchTest
    static final ArchRule domain_must_not_depend_on_adapters =
            noClasses()
                    .that().resideInAnyPackage("..domain..", "..application..")
                    .should().dependOnClassesThat()
                    .resideInAnyPackage("..adapter..", "..infrastructure..");

    @ArchTest
    static final ArchRule contexts_must_not_share_internal_models =
            noClasses()
                    .that().resideInAPackage("..decision..")
                    .should().dependOnClassesThat()
                    .resideInAnyPackage("..profile.internal..", "..catalog.internal..");
}
```

最后一条规则直接对应 ddd-02 的上下文边界：在线决策可以依赖 `FeatureSnapshotContract`，但不能依赖画像或商品上下文的内部模型。这不是“编译规则”的准确说法，它是构建阶段执行的架构测试；它能防止代码依赖长期漂移，但不能代替业务边界评审。

### How：Java Module 可见性示例

领域模块只导出本地需要的端口和模型：

```java
module recommendation.domain {
    exports com.example.recommendation.domain;
    exports com.example.recommendation.port;
}
```

基础设施模块依赖领域模块，但领域模块不需要依赖 Spring：

```java
module recommendation.infrastructure {
    requires recommendation.domain;
    requires spring.context;
    requires spring.data.redis;
}
```

模块系统的作用是限制可见性，不是自动替你划分领域边界。包名、模块名和业务责任仍然需要保持一致。

### Transfer：架构约束的迁移

1. Go 可以用 `internal` package 加测试规则限制内部依赖。
2. Rust 可以用 crate/module 的 `pub` 可见性控制边界。
3. 多模块 Maven/Gradle 工程可以把上下文拆成编译依赖单元。

本层留下的新账：**依赖方向守住了，但真实请求仍会遇到事务、超时、降级和观测问题。**下一层把完整链路放到生产环境验收。

### Level Ending

```text
口诀：约定提醒边界，自动化守住边界。
```

---

<a id="level-6"></a>

## Level 6：事务、安全、失败和生产验证

一句话认知墙：**代码分层完成不代表系统可靠，真正的验收要看边界遇到失败时是否仍然可控。**

### 前置知识关卡

- [ ] 能画出 Controller 到 Adapter 的完整链路
- [ ] 能说出事务不应包住哪些远程调用
- [ ] 能区分领域规则失败和基础设施失败

### Why：代码能跑为什么还不够

**徒弟：**

> Controller、Handler、Domain、Adapter 都有了，请求能返回结果，DDD 代码就落地了吧？

**老陈：**

> 还要问四件事：模型服务超时怎么办，Redis 不可用怎么办，曝光事件失败怎么办，错误依赖怎么阻止。分层只是结构，生产还需要时间预算、失败归属和可观测性。

### What：一次请求的完整链路

```text
① HTTP 请求进入 Controller
② Controller 提取 Actor 和 Query
③ Handler 做用例权限检查
④ Handler 读取 PublishedStrategyView
⑤ FeatureProvider 从 Redis/特征平台读取快照
⑥ CandidateRecallPort 召回候选
⑦ RankerPort 调用模型服务
⑧ RecommendationPolicy 执行业务过滤和兜底
⑨ Handler 生成 Response
⑩ ExposurePublisher 发布曝光事实
⑪ Metrics/Trace 记录每个阶段耗时和失败
```

失败要有归属：

| 失败 | 主要负责人 | 处理方向 |
| --- | --- | --- |
| 参数格式错误 | Interface | 返回协议错误 |
| 无权调用场景 | Application/Security | 拒绝请求并记录审计 |
| 策略规则不允许 | Domain | 返回业务拒绝或兜底 |
| Redis 超时 | Adapter/Application | 超时、旧快照或降级 |
| 模型服务不可用 | Adapter/Application | fallback、熔断或限流 |
| 事件发布失败 | Event Adapter/Outbox | 重试、积压、补偿 |
| 架构依赖违规 | Build/Test | 阻止构建或合并 |

### How：事务、日志和安全验收

```text
写用例：
  Application Handler 开启本地事务
  -> 加载聚合
  -> Domain 行为
  -> Repository 保存
  -> 事务提交

在线读用例：
  读取读模型/缓存
  -> 调外部能力时使用独立超时预算
  -> 不持有无意义的长数据库事务
  -> 返回结果后异步发布事实
```

日志和指标建议按用例边界记录：

```text
traceId
userId 的脱敏标识
scene
strategyVersion
featureSnapshotVersion
ranker latency
fallback reason
event publish result
```

安全边界：

```text
Controller：提取认证信息
Application：判断调用者能否执行这个用例
Domain：判断业务规则是否允许状态变化
Infrastructure：执行凭证、网络和存储适配
```

### 生产验证指标

```text
recommendation_request_latency_p99
feature_provider_latency
ranker_latency
fallback_ratio
contract_decode_error_rate
event_error_rate
event_consume_lag
DB lock wait
CPU / Memory / IO
```

阈值应来自业务 SLO、正常基线和容量实验，不在本文编造固定数字。

**徒弟：**

> 如果模型服务慢，直接把超时时间调大，让它最终返回不就行了？

**老陈：**

> 超时时间不是越大越可靠。它要和整个请求预算、线程池、连接池和用户可接受等待绑定。模型慢时，业务需要决定返回旧结果、热门结果、部分结果还是错误，而不是把等待无限延长。

一个明确的业务降级决策可以写成代码：

```java
RecommendationResult result;
try {
    RankedCandidates ranked = ranker.rank(candidates, strategy);
    result = policy.decide(query, features, ranked, strategy);
} catch (RankingUnavailable ex) {
    // 这里是业务决策：模型不可用时返回热门商品兜底。
    result = policy.decideWithFallback(query, features, strategy);
    metrics.counter("fallback.ranking_unavailable").increment();
}
```

这段代码的关键不在 `catch`，而在兜底结果由 `RecommendationPolicy` 定义。适配器只报告“排序能力不可用”，不能自行决定返回哪些商品。

### Transfer：从代码到生产的方法

1. 用领域单元测试验证规则，不启动基础设施。
2. 用应用集成测试验证事务、权限和端口编排。
3. 用契约测试验证 Redis、模型和 MQ 适配器的输入输出。
4. 用架构测试验证依赖方向。
5. 用故障注入验证超时、降级、事件积压和恢复。

本篇完成了从 Controller 到生产指标的代码链路。下一篇进入领域事件、Outbox 和最终一致性，清算 `ExposurePublisher` 以及策略发布事实的可靠传播问题。

### Level Ending

```text
口诀：代码分层只是开始，失败可控才算落地。
```

---

# 🧪 合书自测

<a id="self-test"></a>

## 完整时间线

<a id="timeline"></a>

```text
T0  HTTP 请求进入 RecommendationController
    |
    v
T1  转换为 Query，提取 Actor
    |
    v
T2  GetRecommendationsHandler 做权限和用例编排
    |
    v
T3  读取策略视图和特征快照
    |
    v
T4  召回、排序、Domain Policy 过滤
    |
    v
T5  模型失败时执行 fallback
    |
    v
T6  返回 Response
    |
    v
T7  异步发布 Exposure 事实并记录指标
```

## 自测问题

| 问题 | 必须回答的不变量 |
| --- | --- |
| Controller 负责什么？ | 协议转换和入口身份，不承载完整业务规则 |
| Application Handler 负责什么？ | 编排用例、权限、事务边界和外部端口调用 |
| Domain 负责什么？ | 业务规则、不变量和本地模型行为 |
| Repository 负责什么？ | 聚合加载和保存，不是所有查询的统一入口 |
| Port/Adapter 的关系是什么？ | 领域定义端口，基础设施实现适配器 |
| ArchUnit 是编译规则吗？ | 不是，是构建阶段执行的架构测试 |
| Java Module 解决什么？ | 编译期模块依赖和可见性 |
| 读路径是否必须加载聚合？ | 不必须，在线查询可使用读模型 |
| 事务是否包住模型 HTTP 调用？ | 不应无脑包住，应按业务写入边界设计 |
| 模型超时谁决定兜底？ | Application/Domain 共同按业务语义决定，Adapter 只报告失败 |

---

# ⚠️ 坑与细节

<a id="pitfalls"></a>

## 坑 1：Controller 直接调用所有外部系统

错误理解：Controller 里直接查 Redis、调模型、发 MQ 最快。

原因：把协议入口当成业务编排入口。

比喻后果：接待窗口自己去找所有专业窗口办理，换一种申请入口就要复制整套流程。

线上现象：HTTP、RPC 和任务入口出现不同业务结果。

修正：Controller 转换输入，Application Handler 编排用例。

---

## 坑 2：Application Handler 变成万能 Service

错误理解：所有判断放到 Handler 就能统一控制。

原因：没有区分流程编排和业务不变量。

比喻后果：编排窗口开始替每个专业窗口审批档案。

线上现象：业务规则只能通过某个入口生效，其他入口绕过后产生不一致。

修正：Handler 编排步骤，Domain 保护规则。

---

## 坑 3：领域层直接依赖框架

错误理解：在领域对象里注入 `RedisTemplate`、`KafkaTemplate` 或 HTTP Client。

原因：把适配器实现和业务规则绑定。

比喻后果：规则窗口必须认识外部设备，设备更换就要重写规则。

线上现象：领域单元测试必须启动完整容器，框架升级影响业务代码。

修正：领域层定义 Port，基础设施层实现 Adapter。

---

## 坑 4：事务包住所有远程调用

错误理解：一个请求从头到尾用一个事务最安全。

原因：把数据库一致性和网络调用等待混为一谈。

比喻后果：窗口为了等待外部回复，一直占着内部档案和办理席位。

线上现象：数据库连接、线程和模型调用一起堆积。

修正：事务覆盖本地状态写入；远程调用使用独立超时、重试和降级。

---

## 坑 5：Port 只是把 SDK 方法复制一遍

错误理解：`FeatureProviderPort` 只是 `RedisTemplate` 方法的同名包装。

原因：没有发生技术语义到业务语义的转换。

比喻后果：交接窗口只转交设备按钮，没有交接真正的业务结果。

线上现象：领域层仍然暴露 Redis Hash、HTTP Response 和 MQ Result。

修正：Port 表达领域需要的能力，Adapter 负责协议和技术细节。

---

## 坑 6：ArchUnit 被当成编译器

错误理解：写了 ArchUnit 就能阻止源码编译依赖。

原因：没有区分测试期检查和编译期模块可见性。

比喻后果：窗口办完手续后才发现内部档案被拿走，不能把它当门禁。

线上现象：架构测试没接入构建，违规依赖仍然进入主干。

修正：ArchUnit 接入构建测试；需要编译期隔离时使用 Java Module 或多模块依赖。

---

## 坑 7：读模型也加载完整聚合

错误理解：所有查询都通过 Repository 加载领域聚合，模型最统一。

原因：把写模型的完整性要求带到了查询路径。

比喻后果：用户只想看窗口公告，却被要求调出完整档案审批。

线上现象：在线请求加载无关对象，延迟和数据库压力增加。

修正：查询使用 Read Model、缓存或特征存储，写入规则仍由聚合保护。

---

## 坑 8：适配器异常直接泄漏到接口

错误理解：Redis timeout、HTTP 500、MQ exception 原样返回给用户。

原因：没有把基础设施失败转换成业务可理解的结果。

比喻后果：窗口把内部设备错误代码直接交给申请人，不说明能否补办或降级。

线上现象：接口错误码随底层组件变化，客户端无法稳定处理。

修正：Adapter 翻译技术异常，Application/Domain 根据业务语义决定重试、兜底或拒绝。

---

# 📊 竖切总表

| 时间 | 位置 | 动作 | 不变量 | 风险 |
| --- | --- | --- | --- | --- |
| T0 | Controller | 解析请求和身份 | 协议对象不进入领域层 | Controller 承载规则 |
| T1 | Handler | 编排用例 | 权限、步骤和事务边界明确 | Handler 变万能 Service |
| T2 | Domain | 执行业务规则 | 不变量由模型保护 | 领域依赖框架 |
| T3 | Query/Repository | 获取读模型或聚合 | 读写路径目标分开 | 所有查询加载聚合 |
| T4 | Port | 表达领域需要的能力 | 不暴露 SDK 语义 | Port 只是 SDK 包装 |
| T5 | Adapter | 连接 Redis、模型和 MQ | 技术异常被转换 | 第三方异常泄漏 |
| T6 | Response | 返回结果或兜底 | 用户响应遵循业务确认语义 | 无限等待 |
| T7 | Metrics/Event | 记录指标和事实 | 失败可观察、可补偿 | 只看接口返回成功 |

---

# 📚 概念勘误表

<a id="errata"></a>

| 错误说法 | 准确说法 |
| --- | --- |
| Controller 就是业务入口 | Controller 是协议入口，业务用例入口是 Application Handler |
| Application 层不能有任何业务判断 | Application 可以做用例编排和权限判断，核心不变量应由 Domain 保护 |
| Domain 层必须完全没有接口 | Domain 可以依赖自己定义的 Port，不应依赖具体技术实现 |
| Repository 是 DAO 的别名 | Repository 以聚合为边界，查询模型可以使用独立 Query Port |
| 事务应该包住整个推荐请求 | 事务应覆盖必要的本地状态写入，不应无脑包住远程调用 |
| ArchUnit 是编译期依赖限制 | ArchUnit 是架构测试；编译期可见性由 Java Module 等机制提供 |
| Port 就是把 SDK 方法复制成接口 | Port 表达领域需要的能力，Adapter 转换技术语义 |
| 模型服务超时由 Adapter 自己决定业务结果 | Adapter 报告技术失败，Application/Domain 按业务策略决定降级 |
| 读模型越完整越接近领域模型 | 读模型只为查询形态服务，不拥有写模型规则 |

其中两点我以前也讲得过于绝对：

```text
我以前容易把“领域层不依赖框架”讲成“领域层不能有任何接口”；准确说法是可以依赖领域自己定义的端口。
我以前容易把“事务边界”讲成“包住一次请求”；准确说法是事务只覆盖需要本地一致的状态写入。
```

---

# 🏆 生产决策卡

<a id="decisions"></a>

## Decision Card 1：应用层事务包到哪里

### 场景

推荐请求需要读取策略、访问 Redis、调用模型并发布曝光。

### 判断

在线读请求不应为了完整路径持有数据库事务；策略发布等写用例才需要由 Application Handler 定义本地事务边界。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 领域聚合状态写入 | 由 Application Handler 开启本地事务 |
| Redis/模型 HTTP 调用 | 独立超时预算，不持有无意义的数据库事务 |
| 曝光事件 | 根据可靠性要求使用 Outbox 或异步发布 |
| 查询读模型 | 使用只读查询、缓存或特征存储 |

### Code

```java
@Transactional
void publishStrategy(PublishStrategyCommand command) {
    RecommendationStrategy strategy = repository.findById(command.id())
            .orElseThrow();
    strategy.publish();
    repository.save(strategy);
}
```

### 禁止决策

```text
从读取特征到模型返回再到 MQ 发送，全部包在一个数据库事务里。
```

### 验收指标

```text
transaction duration
DB connection usage
model latency
recommendation_request_latency_p99
fallback_ratio
Error rate
```

---

## Decision Card 2：端口还是直接 SDK

### 场景

推荐领域需要读取 Redis 特征、调用模型和发布曝光。

### 判断

领域层依赖 Port，基础设施实现 Adapter；Port 表达业务需要，不能照抄 SDK 方法。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 领域需要用户特征 | `FeatureProviderPort` |
| 领域需要排序能力 | `RankerPort` |
| 领域需要发布事实 | `ExposurePublisher` |
| Redis/HTTP/MQ 细节 | 只在 Adapter 中出现 |

### Code

```text
RecommendationPolicy
  -> FeatureProviderPort
  -> RankerPort
  -> ExposurePublisher
```

### 禁止决策

```text
在 RecommendationPolicy 里注入 RedisTemplate、KafkaTemplate 或模型 SDK。
```

### 验收指标

```text
领域模块对框架包的依赖数量
adapter error rate
feature_provider_latency
ranker_latency
CPU
Memory
```

---

## Decision Card 3：ArchUnit 还是 Java Module

### 场景

团队需要阻止领域层依赖 Spring、JPA、Redis 和 MQ。

### 判断

ArchUnit 用于架构测试，Java Module 用于编译期可见性；大型模块可以两者同时使用。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 快速检查包依赖 | ArchUnit |
| 编译期限制导出和 requires | Java Module |
| 多模块构建隔离 | Maven/Gradle module |
| 业务边界和责任判断 | 不能由工具替代 |

### Code

```java
@ArchTest
static final ArchRule domain_free_of_framework =
        noClasses().that().resideInAPackage("..domain..")
                .should().dependOnClassesThat()
                .resideInAnyPackage("org.springframework..", "jakarta.persistence..");
```

### 禁止决策

```text
只写 ArchUnit 测试但不接入构建，也不处理测试失败。
```

### 验收指标

```text
架构测试失败次数
非法依赖数量
构建阻断次数
构建 latency
Error rate
```

---

## Decision Card 4：完整聚合还是读模型

### 场景

在线请求需要读取已发布策略和用户特征。

### 判断

写路径通过聚合保护规则，读路径使用 Published View、缓存和特征存储；不要用完整聚合替代查询设计。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 策略发布 | `RecommendationStrategy` 聚合 |
| 在线策略读取 | `PublishedStrategyView` |
| 用户特征读取 | `FeatureSnapshot`/特征存储 |
| 搜索和候选查询 | 查询端口/搜索读模型 |

### Code

```java
PublishedStrategyView strategy = strategyQuery.findFor(scene)
        .orElseGet(PublishedStrategyView::fallback);
```

### 禁止决策

```text
每个推荐请求都加载并修改完整 Strategy 聚合。
```

### 验收指标

```text
aggregate load count
read model latency
recommendation_request_latency_p99
DB query count
Memory
```

---

# 🌍 跨语言视角

<a id="cross-language"></a>

## Java

```text
Controller/Handler 表达入口和编排
interface 表达 Port
Adapter 接入 Redis、模型和 MQ
ArchUnit/Module 保护依赖方向
```

## Go

```text
HTTP handler 接收协议
use-case service 编排
interface 表达 Port
package/internal 保护模块边界
```

## Rust

```text
web handler 接收协议
application service 编排
trait 表达 Port
crate/module 和 pub 可见性保护依赖
```

## 跨语言仍成立的判断力

```text
入口不等于业务规则。
应用层编排，领域层保护不变量。
外部技术通过端口进入，不能反向污染领域。
编译/测试约束必须进入自动化流水线。
生产验证要覆盖延迟、失败、降级和事件语义。
```

---

# 系列承接：下一篇留下的问题

本篇已经回答：

```text
Controller、Handler、Domain 和 Adapter 如何分工
Repository 和读模型如何接入请求链路
Redis、模型服务和 MQ 如何通过 Port 接入
ArchUnit 与 Java Module 如何守边界
事务、安全、超时、降级和指标如何落地
```

但 `ExposurePublisher` 和 `StrategyPublished` 仍然面对可靠性问题：数据库提交了，事件是否一定发布？事件重复如何处理？消费者失败后如何补偿？

这就是 `ddd-05-事件Outbox与最终一致性.md` 的入口。
