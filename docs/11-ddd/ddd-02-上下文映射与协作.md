# DDD 上下文映射与协作：从特征快照交接到可靠边界

> 这不是一篇 Context Mapping 模式清单。
>
> 本文只追踪一个主线：**事件 E——`FeatureSnapshotPublished`，画像上下文发布特征快照，在线决策上下文接收并翻译它。**
>
> `ddd-01` 已经把窗口划出来了，但窗口之间不能永远各自关门。E 要穿过：
>
> ```text
> 上下文边界
>   -> 关系选择
>   -> 契约设计
>   -> ACL 翻译
>   -> 同步/事件协作
>   -> 版本与故障治理
> ```
>
> 本文回答的不是“有哪些模式”，而是：**两个上下文为什么要这样连接，连接后牺牲了什么，出了问题谁负责。**
>
> 为便于叙事，本文把跨上下文交接称为事件 E；它在边界外属于集成事件。上下文内部的领域事件与集成事件的详细区别留到 `ddd-05`。
>
> R 与 E 的关系是：用户打开首页触发推荐请求 R，R 需要用户特征，画像上下文因此发布 E。R 是请求，E 是跨上下文事实。

# 🏷️ 关键词与主线进度

关键词：

```text
Context Mapping | ACL | Published Language | Open Host Service
Customer-Supplier | Conformist | Shared Kernel | FeatureSnapshotContract
同步调用 | 领域事件 | 契约演进 | 千人千面
```

主线进度：

```text
① ddd-01 留下的交接问题
② Context Map 到底描述什么
③ 8 种关系与 1 个反模式如何选择
④ 契约和 ACL 如何保护本地模型
⑤ 同步调用与领域事件如何取舍
⑥ 生产上如何验证协作关系
```

---

# ⚠️ 版本与证据边界

| 类型 | 本文承诺 |
| --- | --- |
| Specification | Context Map 不是网络协议，模式名称不规定具体 HTTP、RPC 或 MQ 实现 |
| Implementation | `FeatureSnapshotContract`、ACL 和模块关系是本文示例设计 |
| Version | Java 示例按 JDK 21 语法表达；本文不依赖 Spring/MQ 具体版本行为 |
| Need Benchmark | 本文不提供性能数字；同步/事件选择只给出验证指标 |
| Design Assumption | 画像上下文和在线决策上下文属于两个逻辑边界，实际系统需要通过业务责任验证 |

本文必须区分：

```text
关系模式：描述上下文之间的依赖和权力关系
契约：双方同意交换的字段和语义
传输：HTTP、RPC、MQ 或数据库同步
可靠性：重试、幂等、Outbox、补偿等工程机制
```

Context Map 只解决“怎么协作”，不自动解决“消息是否可靠”。

---

# 🗂️ 目录

- [Level 1：边界画好后为什么还会互相污染](#level-1)
- [Level 2：8 种关系与 1 个反模式如何选择](#level-2)
- [Level 3：契约与 ACL 如何保护本地模型](#level-3)
- [Level 4：同步调用和领域事件如何取舍](#level-4)
- [Level 5：生产上如何验证协作关系](#level-5)
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
| Level 1 | “边界划好就互不影响” | 能解释共享内部模型如何重新打穿边界 |
| Level 2 | “所有上下文关系都一样” | 能说明 8 种关系各自解决什么、牺牲什么，并识别大泥球 |
| Level 3 | “DTO 传过去就是协作完成” | 能设计契约、版本和 ACL 翻译层 |
| Level 4 | “异步事件天然更解耦” | 能按确认语义选择同步、事件或本地读模型 |
| Level 5 | “接口调通就算集成成功” | 能用契约、指标、重试和故障演练验证关系 |

---

# 全文唯一比喻：一份业务申请在多个专业窗口办理

## ASCII Diagram

```text
画像窗口完成一项业务
        |
        | 交接：FeatureSnapshotPublished
        v
+-------------------+       +-------------------+
| 画像窗口          | ----> | 决策窗口          |
| 拥有画像和授权    | 契约  | 拥有推荐规则      |
+-------------------+       +-------------------+
        |                              |
        | 内部档案不能直接拿走          | 本地模型不能被外部修改
        v                              v
   Profile Model                 Decision Model
```

## Mapping Table

| 比喻元素 | 技术概念 | 唯一职责 |
| --- | --- | --- |
| 两个专业窗口 | 两个 Bounded Context | 各自拥有内部模型和业务规则 |
| 窗口之间的关系 | Context Mapping | 说明谁依赖谁、如何合作 |
| 正式交接单 | Contract / Published Language | 明确交换字段和语义 |
| 窗口工作人员翻译 | Anti-Corruption Layer | 把外部模型转换成本地模型 |
| 柜台现场询问 | Synchronous Call | 当前办理必须立即得到答案 |
| 办结后发送通知 | Domain/Integration Event | 事实发生后让其他窗口稍后处理 |
| 交接失败登记 | Retry / Idempotency / Compensation | 处理重复、失败和延迟 |

比喻只用于解释边界协作。事件投递、事务和网络行为仍以技术实现和实验结果为准。

---

# 正文：五个 Level 打穿上下文映射与协作

<a id="level-1"></a>

## Level 1：边界画好后为什么还会互相污染

一句话认知墙：**上下文分开只是“不共用模型”的开始，真正困难的是它们必须交换信息。**

### 前置知识关卡

- [ ] 知道 ddd-01 已经划分了画像、商品、策略、在线决策等上下文
- [ ] 知道同一个现实用户在不同上下文可以有不同模型
- [ ] 能区分“交换事实”和“直接修改对方状态”

### Why：边界为什么会再次被打穿

**徒弟：**

> 画像上下文已经有 `UserProfile` 了，在线决策直接调用它的 getter，拿到兴趣和授权状态，不就最简单吗？

**老陈：**

> 这个方案确实减少了数据转换，但它把画像内部模型暴露给了在线决策。以后画像团队把 `interestScore` 改成分层标签，或者把授权状态拆成多个维度，在线决策必须跟着理解内部变化。
>
> 边界已经画出来了，依赖却从门缝里钻了回去。

常见的污染路径：

```text
共享对方 Entity
  -> 直接调用对方内部 Repository
  -> 直接读取对方数据库表
  -> 复制对方内部规则
  -> 两边都修改同一个业务状态
```

| 直觉方案 | 立即得到什么 | 后续留下什么账 |
| --- | --- | --- |
| 共享领域对象 | 少写转换代码 | 内部语义被外部依赖 |
| 共享数据库表 | 查询方便 | 所有者不清，状态可被绕过规则修改 |
| 直接同步调用 | 当前数据新 | 上游抖动传到下游，延迟绑定 |
| 直接发布任意 JSON | 接入速度快 | 字段语义、版本和兼容性失控 |

### What：Context Map 描述的不是箭头，而是关系

Context Map 至少要回答：

```text
谁拥有模型？
谁依赖谁？
谁可以决定契约？
谁承担翻译成本？
上下文之间交换的是命令、查询还是事实？
```

以 E 为例：

```text
画像上下文：拥有用户特征和授权状态
在线决策上下文：拥有推荐请求的业务规则
E：画像已经发布一个可供决策使用的特征快照
```

E 不应该表达为“请在线决策修改你的用户对象”，而应该表达为：

```text
FeatureSnapshotPublished：某个用户的某个特征快照已经发布
```

### How：先画责任，再画通信

```text
① 写出两个上下文各自拥有的规则
② 写出本次协作的业务目的
③ 判断交换的是命令、查询还是事实
④ 决定谁负责转换模型
⑤ 再选择 HTTP、RPC、MQ 或本地读模型
```

**徒弟：**

> 既然最终都要调用接口或发消息，那先把通信方式定下来，再补业务语义不是更快吗？

**老陈：**

> 通信方式只解决“怎么把字节送过去”，不解决“送过去的东西谁负责、能不能重放、字段是什么意思”。先定传输，常常会让协议结构反过来决定业务边界。

### Transfer：上下文协作的通用问题

1. 支付和库存协作时，要先定义谁拥有扣减和支付状态。
2. 教学和成绩协作时，要先定义“作业已提交”和“成绩已发布”分别属于谁。
3. 物流和通知协作时，要先区分运单事实和通知动作。

本层留下的新账：**两个上下文需要协作，但它们之间有很多种关系。**如果不先判断关系，ACL、共享内核和直接调用就会被当成同一种东西。下一层建立 8 种关系和 1 个反模式的选择地图。

### Level Ending

```text
口诀：边界隔离模型，映射管理协作。
```

---

<a id="level-2"></a>

## Level 2：8 种关系与 1 个反模式如何选择

一句话认知墙：**Context Mapping 不是模式背诵，而是明确“谁向谁让步，谁承担变化成本”。**

### 前置知识关卡

- [ ] 能说出画像和在线决策各自拥有的规则
- [ ] 能区分共享对象、共享契约和共享部署
- [ ] 知道关系选择本身也是架构决策

### Why：为什么不能所有关系都用 ACL

**徒弟：**

> 外部模型可能污染我，那所有上下文都加 ACL，不就统一解决了吗？

**老陈：**

> ACL 确实能保护本地模型，但它把翻译成本压给了本地团队。如果两个团队本来就共同维护一套稳定语言，强行翻译只会增加重复代码；如果上下游关系很弱，甚至不值得集成，ACL 也不是免费方案。

每种关系都是一次成本交换：

```text
共享成本
  vs
翻译成本
  vs
依赖成本
  vs
重复建设成本
```

### What：标准关系地图

下面按 Context Mapping 常见关系整理。**大泥球不是推荐的协作模式，而是边界失控后的反模式**，单独列出用于识别风险。

| 关系 | 人话解释 | 何时选择 | 画像/决策示例 | 主要代价 |
| --- | --- | --- | --- | --- |
| Partnership 合作关系 | 两边必须一起演进，谁也不能单方面决定全部契约 | 双方发布节奏和业务语义必须共同协调 | 实验管理和在线决策共同定义分流语义 | 协调会议和发布节奏绑定 |
| Shared Kernel 共享内核 | 两边共同维护一小块稳定模型或代码 | 共享内容很小、稳定且双方愿意共同治理 | 极少量稳定的实验分组类型 | 修改必须双方评审，耦合很紧 |
| Customer-Supplier 客户-供应商 | 上游提供能力，下游是明确客户；下游需求影响供应方优先级 | 上游有明确能力，下游是稳定消费者 | 画像为决策提供特征快照 | 上游需要持续满足下游契约 |
| Conformist 遵奉者 | 下游接受上游模型，不要求上游为自己适配 | 上游格式已是组织或行业标准，下游无需独立解释 | 决策方直接接受平台统一的特征格式 | 下游失去模型自主权 |
| Anti-Corruption Layer 防腐层 | 下游在边界处翻译，保护本地模型不被污染 | 外部模型变化快，或本地语义不能被上游决定 | 决策方把 Profile Contract 转成本地特征 | 翻译代码、字段映射和维护成本 |
| Open Host Service 开放主机服务 | 上游提供稳定、可被多个消费者使用的服务协议 | 一个上游能力需要服务多个下游 | 画像平台提供统一特征查询接口 | 上游要维护长期兼容协议 |
| Published Language 发布语言 | 双方使用公开、稳定、可演进的共享契约 | 多个上下文需要理解同一份交换语言 | `FeatureSnapshotContract` 或标准事件 Schema | 契约版本和兼容治理成本 |
| Separate Ways 各行其道 | 集成成本高于收益，双方明确不集成 | 集成价值低，重复少量数据更便宜 | 非核心推荐场景各自维护简单标签 | 数据重复、能力重复建设 |
| Big Ball of Mud 大泥球 | 没有明确关系，内部对象、表和调用互相穿透 | 不是选择条件，而是需要识别和治理的失控状态 | 推荐、画像、商品直接读写彼此数据 | 变化不可预测，无法定位责任 |

### How：四步选关系

```text
第一问：两边是否必须一起演进？
  是 -> Partnership 或 Shared Kernel 候选

第二问：谁拥有上游模型，谁是稳定消费者？
  是 -> Customer-Supplier / Conformist / ACL 候选

第三问：是否需要多个消费者使用同一协议？
  是 -> Open Host Service + Published Language 候选

第四问：集成价值是否高于维护成本？
  否 -> Separate Ways
  不判断就互相直读 -> Big Ball of Mud 风险
```

**重要区分：**

```text
Open Host Service：提供稳定服务入口
Published Language：规定交换数据的共同语言
```

两者可以一起使用，也可以分开使用。一个 HTTP 接口不自动成为 Published Language；一份 Schema 也不自动说明谁提供服务。

### Transfer：关系模式的迁移

1. 支付平台和订单平台可能是 Customer-Supplier，而不是共享订单实体。
2. 外部身份认证系统通常可以是 Conformist：业务系统接受它的用户标识。
3. 历史遗留系统适合用 ACL 隔离，不应该把遗留模型直接扩散到新系统。
4. 两个低价值报表系统可以 Separate Ways，避免为了数据统一建立高成本集成链。

本层留下的新账：**关系类型已经选了，但具体交换什么字段、字段由谁解释，还没有定义。**下一层进入契约和 ACL。

### Level Ending

```text
口诀：关系先定权责，再选技术通道。
```

---

<a id="level-3"></a>

## Level 3：契约与 ACL 如何保护本地模型

一句话认知墙：**跨上下文传递的不是对方的对象，而是本地愿意理解的一份契约。**

### 前置知识关卡

- [ ] 能区分 Context Mapping 关系和通信协议
- [ ] 能说出为什么不共享 `Profile` Entity
- [ ] 知道 DTO 是传输结构，不自动等于领域模型

### Why：DTO 为什么还不够

**徒弟：**

> 我们定义一个 `ProfileDTO`，通过接口传给决策模块，已经解耦了吧？

**老陈：**

> DTO 只解决“传什么字段”的问题，还没有解决“字段是什么意思”和“谁负责翻译”。如果决策模块直接把 `ProfileDTO` 当成自己的领域对象，DTO 的字段变化仍然会侵入本地规则。

边界内外至少需要两种模型：

```text
外部契约模型：为了协作稳定，字段可序列化、可版本化
本地领域模型：为了本地规则，名称、约束和行为符合本地语言
```

### What：完整契约示例

画像上下文发布的不是 `Profile` 内部 Entity，而是一份明确的快照契约：

以下代码省略 `import` 和类型定义，重点是契约边界；可运行版本放入后续实验代码。

```java
public record FeatureSnapshotContract(
        String userId,
        long snapshotVersion,
        boolean personalizationAllowed,
        Set<String> inferredInterests
) {
    public FeatureSnapshotContract {
        Objects.requireNonNull(userId, "userId");
        Objects.requireNonNull(inferredInterests, "inferredInterests");
        inferredInterests = Set.copyOf(inferredInterests);
    }
}
```

这份契约表达的是：

```text
哪个用户
哪一个快照版本
是否允许个性化
允许交给下游的推算兴趣
```

在线决策上下文不直接使用它，而是转换为自己的模型：

```java
public record RecommendationFeatures(
        DecisionUserId userId,
        FeatureVersion version,
        boolean personalizationAllowed,
        Set<InterestTag> interests
) {
    public RecommendationFeatures {
        Objects.requireNonNull(userId, "userId");
        Objects.requireNonNull(version, "version");
        Objects.requireNonNull(interests, "interests");
        interests = Set.copyOf(interests);
    }
}
```

### What：ACL 放在哪一侧

如果目标是保护在线决策上下文，ACL 通常放在决策侧的输入适配器：

```text
画像上下文
  -> FeatureSnapshotContract
  -> DecisionInboundAdapter
  -> ProfileToDecisionTranslator（ACL）
  -> RecommendationFeatures
  -> 在线决策领域规则
```

示例翻译器：

```java
final class ProfileToDecisionTranslator {
    private final TranslationMetrics metrics;

    ProfileToDecisionTranslator(TranslationMetrics metrics) {
        this.metrics = metrics;
    }

    RecommendationFeatures translate(FeatureSnapshotContract source) {
        Set<InterestTag> tags = source.inferredInterests().stream()
                .map(this::safeParseTag)
                .flatMap(Optional::stream)
                .collect(Collectors.toUnmodifiableSet());

        int dropped = source.inferredInterests().size() - tags.size();
        if (dropped > 0) {
            metrics.droppedTags(dropped);
        }

        return new RecommendationFeatures(
                DecisionUserId.of(source.userId()),
                FeatureVersion.of(source.snapshotVersion()),
                source.personalizationAllowed(),
                tags
        );
    }

    private Optional<InterestTag> safeParseTag(String raw) {
        try {
            return Optional.of(InterestTag.of(raw));
        } catch (IllegalArgumentException ex) {
            return Optional.empty();
        }
    }
}
```

这里的处理策略是“丢弃非法标签并记录指标”，不是通用答案。高风险字段可能应该拒绝整份契约、进入死信或触发人工补偿；策略必须由业务后果决定。

ACL 的位置不是固定模板，判断原则只有一句：

> **谁要保护自己的模型，翻译层就靠近谁。**

DTO 和领域对象不能混用：

| 对象 | 负责什么 | 是否可直接进入领域规则 |
| --- | --- | --- |
| `FeatureSnapshotContract` | 跨上下文传输和版本兼容 | 不直接进入 |
| `RecommendationFeatures` | 在线决策语义和约束 | 可以 |
| `Profile` Entity | 画像上下文内部状态 | 不可以 |
| `Map<String, Object>` | 临时弱类型数据 | 不应该 |

### How：契约设计的五个问题

```text
① 字段的业务含义是什么？
② 字段缺失时，下游如何处理？
③ 字段增加时，旧消费者能否继续工作？
④ 版本号代表数据新鲜度还是契约版本？
⑤ 这个契约是查询结果、命令还是已发生事实？
```

**徒弟：**

> 字段名和类型都写在 JSON 里了，业务含义让双方口头约定就行吧？

**老陈：**

> 口头约定不会进入重试、回放和排障。`inferredInterests` 是用户主动标签还是模型推断？`snapshotVersion` 是数据版本还是协议版本？如果这些不写入契约说明，消费者只能猜。

### Transfer：ACL 的迁移

1. 支付上下文把外部支付渠道状态翻译成本地 `PaymentStatus`。
2. 物流上下文把仓储系统的库存状态翻译成本地 `ReservableStock`。
3. 教育上下文把外部身份系统的用户状态翻译成本地 `StudentIdentity`。

本层留下的新账：**契约已经有了，但当前请求到底该同步拿数据，还是等待异步事件？**下一层比较两种协作方式。

### Level Ending

```text
口诀：传递契约，不传递内部模型。
```

---

<a id="level-4"></a>

## Level 4：同步调用和领域事件如何取舍

一句话认知墙：**事件不是“更高级的调用”，同步和异步表达的是不同业务确认语义。**

### 前置知识关卡

- [ ] 能写出 `FeatureSnapshotContract` 的字段语义
- [ ] 能解释 ACL 为什么放在本地模型边界
- [ ] 知道同步调用会传递等待和故障

### Why：为什么不能统一使用事件

**徒弟：**

> 事件能解耦上下文，画像变化全部发事件，在线决策自己消费，不就不用同步调用了吗？

**老陈：**

> 事件适合表达“已经发生”的事实，但不一定适合当前请求马上要用的数据。如果用户刚刚关闭个性化，推荐请求必须立即知道这个授权变化，单靠异步事件可能会存在一段旧数据窗口。
>
> 反过来，如果每次曝光都同步调用反馈系统，反馈系统抖动就会拖慢推荐接口。两种场景的确认语义不同。

### What：四种协作方式

| 方式 | 表达什么 | 适合场景 | 主要代价 |
| --- | --- | --- | --- |
| 同步查询 | “现在请告诉我答案” | 当前响应必须使用最新结果 | 等待、超时、故障传导 |
| 同步命令 | “现在请你执行动作” | 当前流程必须确认动作结果 | 上下文时间耦合，重试需谨慎 |
| 领域/集成事件 | “某件事已经发生” | 允许稍后处理、广播给多个消费者 | 延迟、重复、版本和补偿 |
| 本地读模型 | “我已经准备好可查询的副本” | 在线低延迟读、可接受短暂旧数据 | 更新延迟、重建和新鲜度治理 |

推荐场景的一种可行组合：

```text
用户授权变化
  -> 关键路径同步确认或本地强约束

画像特征更新
  -> FeatureSnapshotPublished 事件
  -> 决策侧更新本地读模型

推荐请求 R
  -> 读取决策侧特征快照
  -> 不在热路径等待画像上下文完整处理

曝光事实
  -> 异步发布
  -> 反馈上下文消费、统计和补偿
```

### How：用确认语义做决策

```text
第一问：当前用户响应必须知道结果吗？
  是 -> 同步查询/命令或本地已准备好的读模型

第二问：动作完成后，其他上下文稍后处理可以吗？
  可以 -> 领域/集成事件

第三问：读取是否对延迟敏感？
  是 -> 本地读模型、缓存或特征存储

第四问：旧数据会造成不可接受的业务错误吗？
  会 -> 缩短新鲜度窗口、同步确认或拒绝旧版本
```

把四问落到本文主线，可以先得到这张决策表。表中的允许窗口和告警阈值必须由业务 SLO 与实测确定，不写成通用数字。

| 业务场景 | 数据新鲜度要求 | 确认语义 | 推荐方式 |
| --- | --- | --- | --- |
| 推荐请求读取特征 | 允许短暂旧数据 | 当前请求必须快速完成 | 本地读模型/特征存储 + 后台事件刷新 |
| 用户关闭个性化 | 当前请求必须知道 | 必须立即确认 | 同步查询、强约束或请求前置校验 |
| 曝光记录 | 允许异步处理 | 结果已返回，事实随后传播 | 集成事件 + 幂等消费 |
| 商品下架通知 | 允许业务窗口内传播 | 商品状态已发生变化 | 集成事件 + 本地缓存刷新；高风险场景增加同步校验 |

这张表不是让团队背“同步还是异步”，而是强迫大家先回答：

```text
旧数据最多能存在多久？
当前请求需要确认什么？
失败后返回旧数据、兜底结果，还是拒绝请求？
```

事件链路的时序：

```text
① 画像上下文完成特征更新
② 画像上下文产生内部领域事实
③ 转换为对外集成契约
④ 可靠发布 FeatureSnapshotPublished
⑤ 决策上下文接收事件
⑥ ACL 转换成本地 RecommendationFeatures
⑦ 更新本地读模型
⑧ 后续推荐请求读取本地快照
```

**徒弟：**

> 事件发布成功，决策侧最终就一定能拿到新特征吧？

**老陈：**

> 不一定。发布成功只说明某个环节接受了事件，后面还有消费失败、重复、乱序、过期和读模型更新失败。事件把时间解耦了，也把可靠性问题变成了工程责任。Outbox、幂等和补偿要在 ddd-05 继续清算。

### Transfer：同步和事件的迁移

1. 下单后的通知和积分适合事件，下单本身的库存资格不能简单当作通知。
2. 审批完成后的消息推送适合事件，当前页面是否允许提交需要同步判断。
3. 商品搜索索引更新适合事件和读模型，订单提交不应等待搜索索引完成。

本层留下的新账：**协作语义选好了，但事件会不会丢、契约会不会变、重复消费会不会错？**下一层进入生产验证。

### Level Ending

```text
口诀：同步确认结果，事件传播事实。
```

---

<a id="level-5"></a>

## Level 5：生产上如何验证协作关系

一句话认知墙：**接口能调通，只能证明路径存在；不能证明模型边界、契约演进和失败语义正确。**

### 前置知识关卡

- [ ] 能为一个上下文关系选择同步、事件或读模型
- [ ] 能说明契约和领域对象为什么分离
- [ ] 知道事件可靠性不是 Context Map 自动提供的

### Why：设计图为什么会在生产失效

**徒弟：**

> 画像和决策的接口已经联调成功，Context Map 应该就完成了吧？

**老陈：**

> 联调成功只证明幸运路径。生产还会出现：画像发送新字段，旧消费者反序列化失败；事件重复，曝光被记两次；决策使用过期特征，推荐结果违反授权；上游慢，在线请求线程全部等待。

协作关系必须验证四件事：

```text
语义：双方是否理解同一个字段
边界：是否只能通过契约协作
时间：旧数据允许存在多久
失败：重复、超时、积压和版本不兼容如何处理
```

### What：推荐场景的建议关系图

这个关系图基于 `ddd-01` 划分的六个上下文：用户画像、商品目录、推荐策略、实验管理、在线决策和反馈分析。

```text
Profile Context
  -- Published Language + Event --> Decision Context
  -- ACL（决策侧）-------------> Decision Model

Catalog Context
  -- RecommendableItemContract --> Decision Context

Strategy Context
  -- PublishedStrategyContract -> Decision Context

Experiment Context
  -- AssignmentContract --------> Decision Context

Decision Context
  -- RecommendationExposed Event -> Feedback Context
```

这不是唯一答案，而是一个可审查的初始方案：

```text
画像/商品/策略拥有各自语义
决策上下文拥有在线组合规则
契约负责稳定交接
ACL 保护决策侧模型
事件承担允许延迟的事实传播
```

### How：五类生产验收

#### 语义验收

```text
给出一个字段，产品、画像、决策和测试能否说出同一含义？
字段缺失、空值和未知枚举是否有明确处理方式？
```

#### 依赖验收

```text
决策模块是否直接 import Profile Entity？
是否直接查询画像数据库？
是否把外部 DTO 当成本地领域对象？
```

#### 演进验收

```text
新增字段能否兼容旧消费者？
枚举增加时旧代码如何处理？
契约版本和数据快照版本是否混淆？
```

#### 故障验收

```text
事件重复会不会重复更新读模型？
事件积压时推荐是否可以使用旧快照或兜底？
模型服务超时时，是否会拖垮在线请求？
上游不可用时，谁负责告警和补偿？
```

#### 指标验收

| 指标 | 含义 | 告警逻辑 |
| --- | --- | --- |
| `feature_snapshot_freshness` | 当前特征快照距生成的时间 | 超过业务允许窗口告警，窗口待业务确认 |
| `feature_provider_latency` | 特征读取耗时 | 超过在线请求预算或基线异常告警 |
| `event_consume_lag` | 事件尚未被消费的延迟/积压 | 超过恢复预算或持续增长告警 |
| `contract_decode_error_rate` | 契约解析失败比例 | 高于正常基线或出现新版本集中失败告警 |
| `acl_translation_dropped_tag_count` | ACL 翻译时被丢弃的非法标签数 | 出现非预期增长告警 |
| `duplicate_event_count` | 重复事件数量 | 出现非预期增长告警 |
| `fallback_ratio` | 推荐走兜底的比例 | 高于业务基线告警 |
| `recommendation_request_latency_p99` | 推荐请求 P99 延迟 | 超过服务 SLO 告警 |
| `event_error_rate` | 事件消费/处理失败比例 | 高于服务基线告警 |
| `CPU / Memory / IO` | 运行资源使用 | 按容量基线和系统 SLO 告警 |

**徒弟：**

> 这些指标是不是基础设施团队的事情？业务上下文只要定义接口就够了吧？

**老陈：**

> 指标属于运行平台，但指标含义属于业务边界。`feature_snapshot_freshness` 超过允许窗口，不只是“消息慢”，而是推荐业务正在使用过期画像。架构评审必须把技术指标翻译成业务后果。

### Transfer：协作关系的通用复盘方法

1. 事故出现时先问：是模型污染、契约错误、时间窗口还是投递失败？
2. 变更出现时先问：谁拥有字段，谁承担兼容成本？
3. 延迟出现时先问：这个动作必须同步确认，还是可以转成事实事件？
4. 团队调整时先问：关系模式是否仍然匹配双方权责？

本篇的协作设计到此闭合。下一篇要进入聚合、实体和值对象：**契约把上下文连接起来后，边界内部如何保护自己的状态和不变量？**

### Level Ending

```text
口诀：集成成功不是调用成功，而是失败仍有归属。
```

---

# 🧪 合书自测

<a id="self-test"></a>

## 完整时间线

<a id="timeline"></a>

```text
T0  画像上下文完成一次特征更新
    |
    v
T1  产生 FeatureSnapshotPublished 事实
    |
    v
T2  选择 Published Language 和事件契约
    |
    v
T3  决策上下文接收 FeatureSnapshotContract
    |
    v
T4  ACL 翻译为 RecommendationFeatures
    |
    v
T5  更新本地读模型或特征快照
    |
    v
T6  推荐请求读取本地数据并执行领域过滤
    |
    v
T7  事件重复、延迟、版本变化进入生产验证
```

## 自测问题

| 问题 | 必须回答的不变量 |
| --- | --- |
| Context Map 描述什么？ | 上下文之间的责任、依赖、权力和翻译关系 |
| Shared Kernel 和 Published Language 区别是什么？ | 前者共享一小块实现/模型，后者共享交换契约 |
| ACL 保护谁？ | 保护本地模型的一方承担翻译成本 |
| Open Host Service 和 Published Language 区别是什么？ | 前者是稳定服务入口，后者是稳定交换语言 |
| Big Ball of Mud 是什么？ | 没有明确协作关系的边界失控状态，不是推荐模式 |
| 什么时候用同步调用？ | 当前业务响应必须立即确认结果 |
| 什么时候用事件？ | 事实已发生，下游可以延迟处理或广播 |
| DTO 能否直接当领域对象？ | 不能，契约模型和本地业务语义需要隔离 |
| 事件发布成功是否等于业务完成？ | 不等于，还要处理消费、重复、版本和补偿 |

---

# ⚠️ 坑与细节

<a id="pitfalls"></a>

## 坑 1：把 8 种关系当成 8 个框架组件

错误理解：每个上下文必须从 8 种关系中选一个固定组件。

原因：Context Mapping 描述的是关系和权责，不是运行时库。

比喻后果：窗口不是因为贴了某个牌子就自动知道谁负责交接。

线上现象：模式名称写进文档，但契约、重试和负责人仍然不清楚。

修正：先写权责和变化成本，再选择关系名称。

---

## 坑 2：把大泥球当成一种可选架构

错误理解：大泥球也是 Context Mapping 的一种正常模式。

原因：忽略它是边界失控后的反面描述。

比喻后果：所有窗口共用一套档案、互相改章，最后没人知道哪张章有效。

线上现象：跨模块直读表、循环依赖和改一处崩多处。

修正：把大泥球当作诊断信号，追踪具体的共享对象、共享表和隐式调用。

---

## 坑 3：把 ACL 理解成 `BeanUtils.copyProperties`

错误理解：把外部 DTO 的字段复制到另一个 DTO 就完成 ACL，等价于调用 `BeanUtils.copyProperties`。

原因：没有发生语义翻译，只发生了结构搬运。

比喻后果：窗口换了表格格式，却没有重新确认办理含义。

线上现象：`interestScore`、`inferredInterests` 和用户主动标签被当成同一种数据。

反面代码：

```java
// 示意：直接依赖画像上下文内部 Entity
public List<Item> recommend(UserProfile profile) {
    if (profile.getInterestScore() > 0.8) {
        return recommendSportsItems();
    }
    return recommendPopularItems();
}
```

正面代码：

```java
// 示意：只依赖决策上下文自己的模型
public List<Item> recommend(RecommendationFeatures features) {
    if (features.hasHighInterestTag()) {
        return recommendSportsItems();
    }
    return recommendPopularItems();
}
```

修正：翻译器必须把外部语义转换成本地概念，并处理缺失、未知和非法值。

---

## 坑 4：共享内核没有治理人

错误理解：公共代码放进 `common-domain`，所有模块自由修改。

原因：共享内核只有共享，没有共同维护规则。

比喻后果：所有窗口都能修改公共档案，出了问题无法确认责任。

线上现象：公共类升级导致多个上下文同时回归。

修正：共享内核必须足够小、版本受控、变更需要共同评审；否则改成发布契约或本地复制。

---

## 坑 5：事件替代了所有同步查询

错误理解：任何跨上下文调用都应该改成事件。

原因：忽略当前请求的确认语义和数据新鲜度要求。

比喻后果：用户在一个窗口等待办结，系统却只发了通知，没人当场确认结果。

线上现象：用户授权已经变化，推荐仍短暂使用旧特征。

修正：当前响应必须确认的结果使用同步或本地已准备好的读模型。

---

## 坑 6：同步调用没有超时预算

错误理解：当前数据要新，所以在线请求同步调用所有上游。

原因：把新鲜度要求翻译成无限等待。

比喻后果：一个窗口排队，所有后续窗口都停止办理。

线上现象：上游抖动传导到推荐接口，线程池和连接池耗尽。

修正：为同步调用定义超时、旧数据窗口和兜底策略。

---

## 坑 7：契约版本等于数据版本

错误理解：`version` 字段只要存在，就能同时表示 Schema 版本和快照版本。

原因：协议演进和业务数据新鲜度是两件事。

比喻后果：窗口表格版本和申请档案版本混在一起，无法判断是格式变了还是数据更新了。

线上现象：消费者错误处理旧数据，或者新字段兼容判断失效。

修正：分开命名 `schemaVersion`、`snapshotVersion` 或等价语义。

---

## 坑 8：把上游对象直接传给下游

错误理解：`Profile` Entity 能传过去就不用定义 Contract。

原因：把所有权和传输便利混为一谈。

比喻后果：窗口工作人员把内部档案原件带到另一个窗口，后者开始直接改原件。

线上现象：下游字段依赖扩散，上游无法独立演进。

修正：跨上下文只传契约、标识或已定义的事实。

---

# 📊 竖切总表

| 时间 | 位置 | 动作 | 不变量 | 风险 |
| --- | --- | --- | --- | --- |
| T0 | 画像上下文 | 完成特征更新 | 画像规则由画像拥有 | 下游直接修改画像状态 |
| T1 | 画像领域 | 产生已发布事实 | 事实名称和语义明确 | 把内部日志当事件 |
| T2 | 契约层 | 定义快照字段和版本 | 契约与领域对象分离 | JSON 字段无语义 |
| T3 | 决策适配器 | 接收外部契约 | 外部模型不进入领域层 | 直接传 Entity |
| T4 | ACL | 转换本地模型 | 本地规则只依赖本地语义 | 只复制字段不翻译 |
| T5 | 读模型 | 更新可查询快照 | 重复更新不破坏结果 | 没有幂等 |
| T6 | 在线请求 | 读取特征并决策 | 旧数据窗口和兜底明确 | 无限同步等待 |
| T7 | 生产治理 | 观察延迟、积压和版本 | 失败有负责人 | 集成成功只看接口返回 |

---

# 📚 概念勘误表

<a id="errata"></a>

| 错误说法 | 准确说法 |
| --- | --- |
| Context Map 就是一张服务调用图 | Context Map 描述上下文之间的模型关系、权责和协作方式 |
| ACL 是所有关系的默认答案 | ACL 保护本地模型但要承担翻译成本，关系选择取决于权责和变化 |
| Shared Kernel 就是公共工具包 | Shared Kernel 共享的是受共同治理的小块领域模型或代码 |
| Open Host Service 就是任意 HTTP 接口 | OHS 是面向多个消费者的稳定服务协议，仍需明确语义和演进策略 |
| Published Language 就是 JSON | JSON 是格式，Published Language 是双方约定的稳定业务语言/契约 |
| 大泥球是九种模式之一，可以主动选择 | 大泥球是边界失控的反模式，用来识别风险 |
| 事件比同步调用更解耦，所以总应该用事件 | 事件改变确认语义，适合可延迟事实传播，不替代所有同步查询 |
| 事件发布成功就是下游处理成功 | 发布、投递、消费、处理和业务落库是不同确认点 |

其中两点我以前也讲得过于绝对：

```text
我以前容易把 ACL 讲成跨上下文协作的标准答案；准确说法是先判断权责和变化成本。
我以前容易把事件讲成同步调用的“更松耦合版本”；准确说法是事件表达已发生事实，确认语义完全不同。
```

---

# 🏆 生产决策卡

<a id="decisions"></a>

## Decision Card 1：ACL 还是遵奉者

### 场景

画像平台已经提供统一特征格式，在线决策团队需要接入。

### 判断

如果统一格式稳定、下游业务不需要独立解释，可以遵奉；如果下游有自己的业务语义和变化节奏，使用 ACL。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 上游模型就是行业/组织标准 | 可考虑 Conformist |
| 下游需要保护本地语义 | 在下游边界放 ACL |
| 多个消费者共同使用契约 | Published Language + OHS |

### Code

```text
ProfileContract
  -> DecisionInboundAdapter
  -> ProfileToDecisionTranslator
  -> RecommendationFeatures
```

### 禁止决策

```text
为了少写映射代码，直接把上游 Entity 放进下游领域层。
```

### 验收指标

```text
contract_decode_error_rate
ACL translation error rate
跨上下文内部类依赖数量
feature_provider_latency
CPU
Memory
```

---

## Decision Card 2：同步调用还是领域事件

### 场景

在线推荐需要用户特征，画像上下文可以提供同步查询，也可以发布更新事件。

### 判断

当前响应必须确认且旧数据不可接受时使用同步或本地强约束；允许延迟且需要广播时使用事件。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 当前请求必须得到答案 | 同步查询/命令或本地读模型 |
| 事实发生后下游稍后处理 | 领域/集成事件 |
| 在线低延迟读取 | 本地快照、缓存或特征存储 |
| 事件失败可补偿 | Outbox + 重试 + 幂等 |

### Code

```text
FeatureSnapshotPublished
  -> Decision Consumer
  -> ACL
  -> Local Feature Read Model
```

### 禁止决策

```text
所有跨上下文调用都改成事件，不评估旧数据窗口和当前请求语义。
```

### 验收指标

```text
feature_snapshot_freshness
event_consume_lag
fallback_ratio
recommendation_request_latency_p99
event_error_rate
IO
```

---

## Decision Card 3：DTO 还是领域对象

### 场景

团队准备让 `FeatureSnapshotContract` 直接进入在线决策规则。

### 判断

契约负责跨边界稳定传输，领域对象负责本地语义和规则；两者通常需要显式转换。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 跨上下文传输 | Contract/DTO |
| 本地业务判断 | Domain Model |
| 外部模型不可信或变化快 | ACL |
| 双方共同治理的小块稳定模型 | Shared Kernel，严格限制范围 |

### Code

```java
RecommendationFeatures local = translator.translate(contract);
policy.validate(local);
```

### 禁止决策

```text
把外部 DTO 直接当成本地领域对象，省略语义翻译。
```

### 验收指标

```text
契约变更导致的领域代码修改数量
ACL translation error rate
contract_decode_error_rate
单元测试场景数
Memory
Error rate
```

---

## Decision Card 4：是否值得做上下文集成

### 场景

两个低价值模块需要交换少量数据，团队准备建设一条长期同步链路。

### 判断

如果集成收益小于契约、运维、重试和补偿成本，选择 Separate Ways，接受有限的数据重复。

### Mechanism -> Decision

| 机制 | 工程决策 |
| --- | --- |
| 必须共同演进 | Partnership |
| 小块稳定模型长期共享 | Shared Kernel |
| 多个消费者需要统一协议 | OHS + Published Language |
| 集成价值低 | Separate Ways |
| 不明确关系就直接互相读写 | 判定为 Big Ball of Mud 风险 |

### Code

```text
Separate Ways：各自维护本地只读标签
而不是建立双向写入的共享标签表。
```

### 禁止决策

```text
为了“数据一致”，把所有上下文接成双向同步网。
```

### 验收指标

```text
跨上下文调用数量
契约维护成本
event_consume_lag
数据差异对账数量
跨链路 error rate
运维告警数量
```

---

# 🌍 跨语言视角

<a id="cross-language"></a>

## Java

```text
record 表达契约
interface 表达端口
adapter 实现 ACL 和外部协议
package/module 控制上下文可见性
```

## Go

```text
struct 表达契约和本地模型
interface 表达端口
package/internal 隐藏上下文实现
显式 mapper 完成 ACL 翻译
```

## Rust

```text
struct/enum 表达契约和状态
trait 表达端口
module/crate 控制可见性
显式 From/转换函数完成模型翻译
```

## 跨语言仍成立的判断力

```text
契约不是内部对象的别名。
翻译成本必须由某一侧明确承担。
同步/异步选择取决于业务确认语义。
事件传输不等于业务处理完成。
关系模式必须能解释谁拥有规则和变化成本。
```

---

# 系列承接：下一篇留下的问题

本篇已经回答：

```text
上下文之间为什么需要映射
8 种关系各自交换什么成本，另加一个边界失控反模式
契约和领域对象为什么分离
ACL 如何保护本地模型
同步调用和事件如何选择
生产上如何验证协作关系
```

但在线决策已经把外部契约翻译成了本地模型。下一篇要回答：

> **本地模型中的哪些对象有身份？哪些只是值？哪些规则必须在同一个一致性边界内保护？**

这就是 `ddd-03-实体值对象与聚合.md` 的入口。
