# 📢 SpringBoot 事件机制与容器通信（系列 02）

> 本篇文章回答两个问题：
> 1. **一次业务事件**（下单成功、支付完成、缓存失效）如何从 `publishEvent` 出发，被广播给所有感兴趣的监听器——同步还是异步？异常怎么穿透？类型怎么匹配？
> 2. **早期事件与事务事件**：为什么 @Bean 监听器收不到启动早期事件？Boot 启动到底广播了哪几个事件？`@TransactionalEventListener` 为什么"提交后才执行"？
>
> 前置：00 篇的创建链（BeanPostProcessor、refresh 主路径）。本篇在创建链之上新增"监听器"角色——事件机制不引入新 Bean 类型，只引入"发布-订阅"这一种通信方式。

---

# ⚠️ 版本与证据边界

| 维度 | 本文承诺 |
| ---- | ---- |
| 代码实证 | `knowledge/springboot/experiments/code/` 下的 demo08×5 与 demo09，本机实测输出原样引用 |
| 实测环境 | macOS + JDK 21.0.11（Azul Zulu）+ spring-context **6.1.14** + spring-tx 6.1.14 + spring-boot **3.3.5** |
| 运行方式 | `cd knowledge/springboot/experiments/code && ./build.sh && ./run.sh demo08.XXXApp` |
| Specification | 观察者模式语义、@EventListener/@TransactionalEventListener 契约、发布-订阅时机（提交后执行） |
| Implementation | 广播器结构、earlyApplicationEvents 缓存回放、@EventListener 注册链、AFTER_COMMIT 触发点（6.1 反编译 `TransactionalApplicationListenerSynchronization`）、Boot 启动事件时序（标注 3.3.5/6.1.14） |
| 待验证 | 真实生产框架的异步/事务事件组合细节（如 RocketMQ 事务消息的 Spring 集成）、跨版本 5.3 与 6.x 的事务事件触发点差异细节 |
| 未覆盖 | 不承诺任何性能数字；Web 层事件（04 篇）；Spring Cloud 事件 |

---

# 🏷️ 关键词

事件 | 观察者模式 | publishEvent | ApplicationEventMulticaster | @EventListener | 同步/异步 | 异常穿透 | earlyApplicationEvents | 启动事件 | @TransactionalEventListener | AFTER_COMMIT | TransactionSynchronization | fallbackExecution

---

# 🗂️ 目录

- Level 1 为什么需要事件：直接调用死在哪
- Level 2 同步事件核心：广播器、注册链、类型匹配、异常穿透
- Level 3 异步事件：为什么同步会阻塞发布者
- Level 4 早期事件与启动事件全景
- Level 5 事务事件：为什么"提交后才执行"
- Level 6 生产实践：事件排查方法论
- 全篇因果链总图
- 线上案例（3 个）
- 面试自查表
- 坑与细节（10 个）
- 版本勘误表
- 生产决策卡（3 张）
- 跨语言视角
- 系列索引

---

# 🏭 全文唯一比喻：公告栏广播

办公室公告栏是"事件机制"的唯一比喻：

- **贴告示**（publishEvent）：任何部门都能贴一张告示（发布事件），贴完就走，不用挨个通知。
- **自己来看**（监听器订阅）：谁感兴趣谁来公告栏看（监听器按类型订阅），不感兴趣的人完全无感。
- **广播员**（ApplicationEventMulticaster）：公告栏本身——谁贴的告示它都广播给所有"订阅了这个分类"的人。
- **贴告示的人**（发布者）：贴完就走，但**告示会不会被立刻读走取决于广播员是当场递还是转交**——这就是同步/异步的差别。
- **公告栏上的旧告示**（早期事件）：办公室刚开张、广播员还没上岗时贴的告示，会被收进抽屉，广播员上岗后统一回放——但"后来才报名订阅的人"（@EventListener 方法）看不到这些旧告示。

```text
现实世界（公告栏）                      技术世界（Spring 事件）
业务部门写告示                    模块调用 publishEvent(...)
   │ 贴到公告栏                        │ 交给 ApplicationEventMulticaster
   ▼                                    ▼
广播员当场念给在场的人            ←→   同步：发布线程直接调用监听器
广播员转交给别人念                ←→   异步：TaskExecutor 线程池调用
广播员把告示锁进抽屉（没上班时）  ←→   earlyApplicationEvents 缓存（多播器未就绪）
   │ 上班后回放                        │ registerListeners 阶段回放
   ▼                                    ▼
   （已报名的人听到；后报名听不到）     （已注册的 ApplicationListener 收到；@EventListener 方法收不到）
```

**全文不再出现第二个比喻。**

---

# 正文：六个 Level 打穿"事件机制"

<a id="level-1"></a>

## Level 1 为什么需要事件：直接调用死在哪

### 前置知识关卡

* [ ] 知道"模块间怎么通知"的三个候选方案（直接调方法 / 轮询 / 事件）
* [ ] 知道 00 篇的 Bean 生命周期（监听器本身也是 Bean）
* [ ] 知道"发布-订阅"与"点对点"的差别

### 1.1 Why：直接调方法的账

徒弟：

> 下单成功后要发短信、写日志、清缓存——直接在 `orderService.create()` 后面依次调用不就行了？为什么要搞事件？

老陈：

> 直接调用能跑通 Demo，但是留下三笔账——
> 第一，**耦合账**：`OrderService` 要知道"下单后要做哪些事"，每加一个动作（积分、风控、审计）都要改 `OrderService` 的代码；
> 第二，**顺序账**：三个动作的先后顺序写死在业务代码里，想调整顺序就要动核心业务代码；
> 第三，**失败账**：发短信失败会直接炸掉下单主流程——**旁路动作的失败和核心业务的成败被绑死了**。
> 事件机制把"下单"和"下单后做什么"解耦：`OrderService` 只贴一张"下单成功"的告示，谁关心谁来听；旁路失败不再连坐主流程（用异步）；顺序由监听器自己声明。

| 方案 | 解决什么 | 留下什么问题 |
| ---- | ---- | ---- |
| 直接调用 | 能跑 | 耦合（加动作改业务代码）、顺序写死、旁路失败连坐 |
| 轮询（定时查状态） | 不侵入业务 | 延迟、空转浪费、状态要靠查才知道 |
| 事件（发布-订阅） | 发布与消费解耦、可多监听、可异步 | 调用链不透明（不读代码不知道谁在听）、调试难、异常语义要设计 |

**为什么轮询会"死"**：轮询是"消费者主动去问"，事件是"发布者主动宣告"——前者延迟取决于轮询间隔，后者即时；前者消费者要自己维护"我关心什么"，后者由广播器按类型分发。**事件是"推"的观察者模式，轮询是"拉"的轮询模式**——事件省掉了"拉"的延迟与空转。

### 1.2 What：三张核心结构图

**① 一次发布的生命线**

```text
publishEvent(event)
   ├─ 包装：普通对象包成 ApplicationEvent（PayloadApplicationEvent）
   ├─ 早期判断：多播器没就绪？→ 进 earlyApplicationEvents 缓存（Level 4）
   └─ 多播器就绪 → multicastEvent(event, eventType)
         ├─ 按类型过滤：找"支持这个事件类型"的监听器（supportsEventType，Level 2 深挖）
         ├─ 排序：@Order 决定执行顺序
         └─ 逐个调用：同步直接调 / 异步交给 TaskExecutor（Level 3）
```

**② 监听器的三种注册姿势**（启动时序上依次可用）

| 姿势 | 方式 | 早期事件能收到吗 | 谁注册的 |
| ---- | ---- | ---- | ---- |
| 显式 addApplicationListener | `ctx.addApplicationListener(...)` / SpringApplication.addListeners | ✅（能收到回放和 run 事件） | 应用代码/框架 |
| implements ApplicationListener 的 Bean | @Component + implements ApplicationListener | ✅（registerListeners 阶段提前实例化） | 容器（registerListeners） |
| @EventListener 方法 | 方法上标 @EventListener | ❌（处理器在回放之后才创建，Level 4 实证） | EventListenerMethodProcessor |

**③ 监听器执行时的参数注入**：@EventListener 方法除了事件参数，还可以声明 `ApplicationEventPublisher`、`@Value` 等参数——adapter 执行时从容器解析（`ApplicationListenerMethodAdapter.resolveArguments`，6.1 Implementation）。

### 1.3 Transfer

- **"推 vs 拉"是通用判断**：缓存失效（推事件 vs 拉版本号）、配置变更（推 vs 轮询）——推省延迟，拉省复杂度，选型先问"延迟敏感吗"。
- **旁路动作降级**：任何"主流程之外的增值动作"（通知、审计、埋点）都应该是订阅者而不是主流程代码——这决定了它能被异步化、被独立失败、被开关控制。

**本层留下的账**：贴告示很简单，但公告栏（广播器）内部怎么工作？告示怎么被分类派发？——这逼出下一层：同步事件核心。

---

<a id="level-2"></a>

## Level 2 同步事件核心：广播器、注册链、类型匹配、异常穿透

### 前置知识关卡

* [ ] 知道 BeanPostProcessor 回调时机（00 篇 Level 2）
* [ ] 知道 @Order / Ordered 的作用
* [ ] 知道 JDK 泛型 ResolvableType 的存在

### 2.1 Why：一个广播器为什么够用

徒弟：

> 每个事件都要广播给多个监听器，为什么 Spring 只用一个 ApplicationEventMulticaster？

老陈：

> 因为"广播"这个动作本身是**无状态的**：给谁广播、按什么顺序，全由监听器集合 + 事件类型决定，广播器只需要一张"监听器注册表"。
> 一张注册表 + 一次类型匹配 + 一次遍历调用——这就是同步广播的全部。
> 异步化也不需要第二个广播器：给广播器配一个 TaskExecutor 就行（Level 3）。**一个广播器 + 可插拔执行器**，就是 Spring 事件异步的完整设计。

### 2.2 代码实证：同步广播三件事（demo08.EventSyncApp）

完整代码：`experiments/code/src/demo08/EventSyncApp.java`。

实测输出（`./run.sh demo08.EventSyncApp`）：

```text
[发布] publishEvent 开始（线程=main）
[监听1] SMS 同步收到: order=1001（线程=main）
[监听2] 日志同步收到: order=1001（线程=main）
[监听3] 基类监听: ApplicationEvent 也收到 order=1001（线程=main）
[发布] publishEvent 返回——同步广播，发布线程被阻塞到监听器跑完
[异常] 监听器抛异常，publishEvent 冒泡给发布者（同步穿透）
```

**六行输出钉死同步广播的四件事**：
1. **一次发布、多个监听器**：SMS、日志、基类监听三个监听器都收到同一个事件——广播不是单播；
2. **同步**：监听器全部在 `main` 线程执行（与发布线程相同）——监听器跑多久，发布线程就等多久；
3. **类型匹配（含父类）**：`@EventListener` 监听 `ApplicationEvent` 基类也收到子类事件——匹配规则是"监听类型能否容纳事件类型"（`ResolvableType.isAssignableFrom`），不是精确相等；
4. **异常穿透**：监听器抛 `IllegalStateException` → `publishEvent` 直接把异常冒泡给发布者——**同步广播没有异常隔离**，这是同步事件最重要的生产语义。

### 2.3 深挖：@EventListener 的注册链（6.1.14 Implementation）

```text
容器启动（finishBeanFactoryInitialization 阶段的末尾）
  └─ EventListenerMethodProcessor（SmartInitializingSingleton，javap 实证 6.1.14：
       implements SmartInitializingSingleton + BeanFactoryPostProcessor）
        └─ afterSingletonsInstantiated：所有单例创建完成后回调，扫描每个 bean 的方法
              ├─ 找到标 @EventListener / @TransactionalEventListener 的方法
              ├─ 按方法参数推断监听类型（ResolvableType）
              ├─ 用 EventListenerFactory 创建 ApplicationListenerMethodAdapter
              │     ├─ DefaultEventListenerFactory → @EventListener
              │     └─ TransactionalEventListenerFactory → @TransactionalEventListener（Level 5）
              └─ 注册进 ApplicationEventMulticaster（addApplicationListener）
```

**关键位点**：@EventListener 的处理器 `EventListenerMethodProcessor` 是容器内置注册的 **SmartInitializingSingleton**（不是 BeanPostProcessor）——它在 `finishBeanFactoryInitialization` **所有单例创建完之后**通过 `afterSingletonsInstantiated` 回调处理——**这决定了"@EventListener 方法收不到早期事件"**（Level 4 的因果本体）。

### 2.4 深挖：类型匹配与缓存（6.1.14 Implementation）

```text
multicastEvent(event, eventType)
  └─ getApplicationListeners(event, type)：查缓存（ListenerCacheKey → 监听器列表）
        ├─ 缓存未命中 → 遍历注册表，逐个 supportsEventType(eventType)
        │     ├─ GenericApplicationListener：自己决定支持什么类型
        │     └─ 普通 ApplicationListener：按泛型签名推断（ApplicationListener<OrderCreatedEvent>）
        └─ 结果按 @Order / Ordered 排序后缓存
```

**为什么监听基类能收到子类事件**：adapter 的 `supportsEventType` 用 `ResolvableType.isAssignableFrom`——监听 `ApplicationEvent`（父）可以容纳 `OrderCreatedEvent`（子），所以匹配成功（demo08 第 3 行实证）。**匹配是"能容纳"，不是"类型相同"**——监听父类 = 收所有子类事件，这是坑 7 的根源。

**泛型事件的隐藏陷阱（demo08.GenericEventApp 实测）**：

一句话版：**泛型事件不是"收不到"，而是"全收误收"**——事件载荷写成泛型类 `GenericEvent<T>`，监听器分别监听 `<String>` 和 `<Integer>`，发布 `GenericEvent<String>` 后**两个监听器都执行了**。

类比版（广播/点名）：广播站喊的不是"订单已创建的各位来一下"，而是只喊了"GenericEvent 的来一下"——点名册上只写了姓氏没写全名，于是姓这个姓的全站了起来。不是点名喊了两拨人，是**点名册上信息不全**。

机制版（擦除双端信息差 + 放水匹配）：发布端 `ResolvableType.forInstance(event)` 从运行时 Class 推导类型，泛型类的 Class 拿不到类型参数（类型擦除）→ 事件类型退化为 raw `GenericEvent`；匹配端 adapter 的 `supportsEventType` 看到 raw 类型，对任意泛型签名都"放水匹配"（raw 可容纳任何 `<X>`）。两端其实都"诚实"：发布端确实**不知道**具体参数，匹配端确实**匹配一切泛型**——是擦除把信息抹掉了，不是任何一端犯错。这就是坑 9 的机制本体。

修复提示：要么换专用事件类（每个场景一个类，类型信息完整）；要么让事件实现 `ResolvableTypeProvider` 显式提供 `getResolvableType()`——**泛型事件想精确路由，必须有人把类型参数"亲口告诉"发布端**（`ResolvableType.forInstance` 反编译实证：先 `instanceof ResolvableTypeProvider` 调 `getResolvableType()`，非空即返回，否则才回退 `forClass(getClass())` 擦除路径——Spring 6.1.14），擦除机制不会帮你。

### 2.5 How：同步广播的源码路径（6.1.14）

```text
AbstractApplicationContext.publishEvent
  └─ SimpleApplicationEventMulticaster.multicastEvent
        ├─ 有 TaskExecutor？→ executor.execute(task)     ← 异步分支（Level 3）
        └─ 无 → task.run()                              ← 同步分支：当前线程直接跑
              └─ 监听器调用 → 异常处理二选一（默认 errorHandler=null）：
                    无 errorHandler → 直接上抛（同步穿透）
                    配了 errorHandler → 交给它（可吞掉，同步也不穿透）
```

**最反直觉的一步**：Spring 事件**默认就是同步的**。`SimpleApplicationEventMulticaster` 的 executor 字段默认 null，`multicastEvent` 里 `executor == null ? run() : execute()`——异步是要显式配置的，不是默认能力；异常处理同理（errorHandler 默认 null，同步穿透是默认行为，配置 errorHandler 可改变）。

### 2.6 生产警示：照着 demo08 写业务，四个风险等着你

徒弟：

> demo08 流程清晰、输出漂亮，我下单业务里直接这么用——发个事件让短信和日志旁路处理，行吗？

老陈：

> 不行——demo08 是"机制演示"，它刻意打印了同步 + 穿透这两个**生产里最危险**的默认行为。你照抄进下单主链路，等于把短信和日志计时器接在了主线程上。四个风险逐个说：

**风险 1：同步穿透 —— 旁路动作变成主链路延迟**。demo08 第 5 行：发布线程 `main` 被阻塞到监听器跑完。生产里监听器里随便一次 Redis 慢查询、一次外呼超时，订单接口就跟着慢——02 篇线上案例 1 "下单接口偶发超时"就是它，监控里背锅的永远是接口，不是你的监听器。

**风险 2：异常穿透 —— 监听器炸了，主流程跟着炸**。demo08 第 6 行：`publishEvent` 把监听器异常直接上抛给发布者（`invokeListener` 里 errorHandler 默认 null 时的路径，2.5 已钉）。日志写失败了、缓存清理抛了 `SQLException`，下单接口直接 500——**你为了"解耦"而加的事件，反而让旁路故障反向耦合进主链路**。

**风险 3（本文新增，最隐蔽）：异常中断后续监听器 —— 前一个炸了，后一个全部收不到**。反编译 6.1.14 字节码实证：`multicastEvent` 的 `for` 循环遍历监听器集合，循环体内**没有任何 catch**，`invokeListener` 抛出的异常直接跳出循环。所以监听器抛异常不是"这个监听器失败"——是**它及它之后的所有监听器集体失败**。你的"日志监听器"排在"SMS 监听器"后面，SMS 一炸，本次事件的日志直接缺失。

```text
multicastEvent：for (listener : listeners) {
                    invokeListener(listener);   // 抛异常 → 直接跳出循环
                }
                未执行的监听器：事件丢失，无日志、无补偿
```

**风险 4：内存态广播 —— 进程重启即丢，跨服务即无**。监听器注册表在 `ApplicationEventMulticaster` 内存里，进程崩溃事件就丢了；监听器必须和发布者在同一 JVM（跨进程？用 MQ，见 Decision Card 1）。

---

**什么场景才适合 Spring Event？决策卡（对照选）**：

| 是否满足 | 场景 |
| ---- | ---- |
| ✅ 适合 | 同进程内**旁路**（记录审计日志、失效缓存、发内部统计）；失败**可容忍**（丢了不误伤主业务）；需要共享发布线程的上下文（ThreadLocal 传递、事务传播） |
| ✅ 适合 | @TransactionalEventListener AFTER_COMMIT（Level 5）：事务提交后做的事，天然与主事务解耦，且失败不影响已提交事务 |
| ❌ 不适合 | 跨服务/跨进程通信（要持久化、可回溯 → MQ）；强一致关键路径（事件是"尽力广播"，不是可靠投递）；慢外部依赖（短信/邮件/推送，监听器慢=主线程慢）；需要重试/幂等/死信（Spring 事件没有这些——要换 MQ） |

> 📌 一句话：**Spring Event 是"同进程内的解耦"，不是"可靠的消息"**。它解耦的是调用点（发布者不认识监听者），不提供任何投递保证。判断标准就一条——**这次广播失败/丢失，主业务还成立吗？** 成立就用；不成立，事件是你的刀刃，不是你的盾牌。

### 2.7 Transfer

- **注册表 + 可插拔执行器**：一套匹配逻辑、两种执行模式——"逻辑与执行解耦"（策略模式）的经典应用：JDBC DriverManager（驱动注册表 + 按 URL 选择）、Netty 的线程模型（业务逻辑与 EventLoop 绑定）都是"注册 + 选择/执行"分离。
- **匹配是"能容纳"**：接口/基类监听会收到全部子类——理解"包容性匹配"才能理解为什么一个监听器收到了一堆不该收的事件。

**本层留下的账**：同步广播会把发布线程绑死，旁路动作一慢，主流程跟着慢——怎么解？——下一层：异步事件。

---

<a id="level-3"></a>

## Level 3 异步事件：为什么同步会阻塞发布者

### 前置知识关卡

* [ ] 知道 @EnableAsync + @Async 的代理机制（AOP 代理，01 篇 demo07 的代理范式）
* [ ] 知道线程池概念（TaskExecutor）

### 3.1 Why：同步的账

徒弟：

> 同步广播看起来没毛病，为什么还要异步？

老陈：

> 同步广播有一个致命场景——**旁路动作慢，主流程跟着慢**。
> 发短信、写日志、清缓存都是毫秒到秒级的 IO——如果全部同步执行，一次下单的耗时 = 核心业务耗时 + 所有旁路动作耗时之和。
> 而"下单"的语义是"订单落库成功"，跟短信/日志/缓存**没有因果关系**——这些动作不应该阻塞主流程。
> 异步的代价是：**异常不再穿透**（监听器炸了发布者无感）、**执行时机不可控**（提交线程池后不知道何时执行）——用异步换解耦，就要接受这两笔账。

| 方案 | 优点 | 代价 |
| ---- | ---- | ---- |
| 同步广播（默认） | 时序可控、异常可见、结果可依赖 | 旁路慢则主流程慢 |
| 异步（@Async 或广播器配 executor） | 发布者立即返回、旁路失败不连坐 | 异常被吞（默认打日志）、无返回结果、乱序 |
| 消息队列（跨进程） | 解耦最强、可持久化、可重放 | 运维成本、一致性语义（05/06 篇展开） |

### 3.2 代码实证：@Async + @EventListener（demo08.AsyncEventApp）

完整代码：`experiments/code/src/demo08/AsyncEventApp.java`。

实测输出（`./run.sh demo08.AsyncEventApp`）：

```text
[发布] publishEvent 开始（线程=main）
[发布] publishEvent 立即返回——异步广播不等监听器
[异步异常] 监听器抛错不冒泡，发布者正常继续（异常由异步执行器吞掉）
[异步监听] 线程=async-event-1，收到 msg=hello-async
```

**四行输出钉死异步的三件事**：
1. **发布者立即返回**：`publishEvent` 不等待监听器（异步监听器 300ms 后执行，发布者不等）；
2. **异常不穿透**：异步监听器抛异常，发布者正常继续——异常被线程池的默认处理（`TaskUtils` 日志记录）吞掉；**这是异步的最大坑：监听器失败完全静默**（除非自定义 `AsyncUncaughtExceptionHandler`）；
3. **执行在线程池**：监听器跑在 `async-event-1` 线程（自定义 executor），不在 main 线程。

**实现原理**：@Async 给监听器 bean 生成 AOP 代理，adapter 持有的 bean 是代理（监听器在线程池执行本身证明调用走了代理）→ 调用走 `AsyncExecutionInterceptor` → 提交到 executor（6.1 实测：`async-event-1` 线程执行）。

### 3.3 两种异步姿势（等效，但语义不同）

```text
姿势 A：广播器配 executor（全局异步）
  SimpleApplicationEventMulticaster.setTaskExecutor(executor)
  → 所有事件都异步（无法区分"哪些事件要异步"）
  → 在 Boot 里通过 ApplicationContextInitializer 或自定义 Bean 替换广播器

姿势 B：@Async 标注在 @EventListener 方法上（按方法异步，demo 实测）
  → 只有标了 @Async 的监听器异步，其余保持同步
  → 需要 @EnableAsync + executor Bean
```

**生产建议**：默认用姿势 B——异步是监听器自己的选择，而不是广播器的全局行为；全局异步会让"本该同步完成"的监听器（比如审计落库）也变成异步，语义丢失。

### 3.4 Transfer

- **"快路径返回 + 慢路径异步"**：任何"核心动作 + 增值动作"的系统（支付回调 + 通知、请求 + 埋点）都该先问"这个动作和核心结果有因果关系吗"——没有，就异步。
- **异步的代价是异常静默**：引入异步的同时必须引入"失败可观测"（异常处理器 + 监控告警），否则故障全部静默。

**本层留下的账**：异步/同步都讲完了，但有一个更隐蔽的时序问题——**容器自己启动时广播的事件**，监听器能收到吗？——下一层：早期事件与启动事件全景。

---

<a id="level-4"></a>

## Level 4 早期事件与启动事件全景

### 前置知识关卡

* [ ] 知道 refresh() 的主步骤顺序（00 篇 Level 1.5）
* [ ] 知道 @EventListener 的注册时机（Level 2.3：finishBeanFactoryInitialization 阶段）

### 4.1 Why：启动早期的事件，监听器为什么收不到

徒弟：

> 容器启动过程中会发布事件吗？我写了 @EventListener 监听 `ApplicationStartedEvent`，怎么从来没收到？

老陈：

> 先分清楚两个阶段——
> **容器内部的事件**（refresh 过程中）：多播器没就绪时发布的，会被收进 `earlyApplicationEvents` 缓存，多播器就绪后回放；
> **Boot run 的早期事件**（Starting/EnvironmentPrepared 等）：发生在容器 **refresh() 之前**——容器要么还没创建，要么已创建但未就绪（Bean 一个都还没注册），@EventListener 方法作为容器 Bean 当然收不到。
> 两个阶段的共同点：**"后来的订阅者"收不到"早先的告示"**——公告栏刚开张时贴的告示，只有当时已报名的人能看到。

### 4.2 代码实证：earlyApplicationEvents 缓存与回放（demo08.EarlyEventApp）

完整代码：`experiments/code/src/demo08/EarlyEventApp.java`。

实测输出（`./run.sh demo08.EarlyEventApp`）：

```text
[BFPP] refresh 早期发布事件（多播器未初始化 → 进 earlyApplicationEvents 缓存）
[DirectListener] 收到早期事件（registerListeners 回放时被提前实例化）
[验证] 直接 ApplicationListener 收到=true；@EventListener 方法收到=false
```

**三行输出钉死早期事件机制**：
1. **缓存**：`BeanFactoryPostProcessor` 阶段（refresh 第 5 步）发布事件时，多播器还没创建（第 8 步）→ 事件进 `earlyApplicationEvents` 缓存，**不丢但延迟广播**；
2. **回放**：`registerListeners`（第 10 步）会**提前实例化**所有 `implements ApplicationListener` 的 Bean（`getApplicationListenerBeans()`）→ 回放缓存事件 → `DirectListener` 收到；
3. **@EventListener 收不到**：`EventListenerMethodProcessor` 在 `finishBeanFactoryInitialization` 才创建——**回放早已结束**，`MethodListener` 收不到（`methodReceived=false` 实测）。

**为什么容器要"缓存 + 回放"而不是"丢弃"**：早期事件（如 `ContextRefreshedEvent` 前的自定义启动标记事件）对"直接注册的监听器"仍有意义——缓存是"等广播员上岗再回放"，而不是"开张前的告示直接作废"。

### 4.3 代码实证：Boot 启动事件全景（demo09.BootEventsApp）

完整代码：`experiments/code/src/demo09/BootEventsApp.java`。

实测输出（`./run.sh demo09.BootEventsApp`）：

```text
[事件] ApplicationStartingEvent
[事件] ApplicationEnvironmentPreparedEvent
[事件] ApplicationContextInitializedEvent
[事件] ApplicationPreparedEvent
[事件] ContextRefreshedEvent
[事件] ApplicationStartedEvent
[事件] AvailabilityChangeEvent(CORRECT)
[Runner] ApplicationRunner 执行
[事件] ApplicationReadyEvent
[事件] AvailabilityChangeEvent(ACCEPTING_TRAFFIC)
[运行] run() 返回——全部启动事件已广播完毕
[事件] ContextClosedEvent
```

**十二行输出 = Boot 启动时序全景**（3.3.5 实测）：

```text
SpringApplication.run
  ├─ refresh() 之前（run listeners 广播）：
  │     ApplicationStartingEvent（run 最开始，容器未创建）
  │     ApplicationEnvironmentPreparedEvent（Environment 就绪，容器未创建）
  │     ApplicationContextInitializedEvent（context 创建完，未 refresh）
  │     ApplicationPreparedEvent（容器准备完成，未 refresh）
  ├─ refresh 内部：
  │     ContextRefreshedEvent（refresh 完成，容器发布）
  ├─ run 收尾（run listeners 广播）：
  │     ApplicationStartedEvent（run 方法返回前）
  │     AvailabilityChangeEvent（LivenessState=CORRECT，启动状态就绪）
  │     ApplicationRunner 执行（实测位点：Started 之后、Ready 之前）
  │     ApplicationReadyEvent（应用就绪）
  │     AvailabilityChangeEvent（ReadinessState=ACCEPTING_TRAFFIC，接受流量）
  └─ 关闭（容器发布）：
        ContextClosedEvent（ctx.close()）
```

**关键认知**：
- 前 4 个事件发生在 **refresh() 之前**：Starting/EnvironmentPrepared 时容器还没创建；ContextInitialized/Prepared 时容器已创建但**未 refresh（Bean 一个都没注册）**——这个阶段只有 `SpringApplication.addListeners` 注册的监听器（或从 `META-INF/spring.factories` 加载的 ApplicationListener，Boot 3 的该加载路径**待验证**）能收到，**@Bean/@EventListener 一概收不到**（它们要等容器启动过程中注册）；
- `ContextRefreshedEvent` 与 Started/Ready 之后的事件，容器已就绪，@EventListener 能收到；
- `AvailabilityChangeEvent` 的 payload 实测为 `CORRECT`（Liveness）与 `ACCEPTING_TRAFFIC`（Readiness，Boot 3.3.5）——**K8s 探针语义的事件载体**；
- `ApplicationRunner` 实测位点：在 `ApplicationStartedEvent` 与 `AvailabilityChangeEvent(CORRECT)` 之后、`ApplicationReadyEvent` 之前执行（demo09 实测，Boot 3.3.5）。

### 4.4 How：refresh 中事件机制的三个位点（6.1.14 Implementation）

```text
prepareRefresh：earlyApplicationEvents = 新 LinkedHashSet（缓存容器就位）
invokeBeanFactoryPostProcessors（第 5 步）：早期发布 → 进缓存（demo08 实证）
initApplicationEventMulticaster（第 8 步）：创建 SimpleApplicationEventMulticaster
registerListeners（第 10 步）：
  ├─ 加入显式 addApplicationListener 的
  ├─ getApplicationListenerBeans()：提前实例化 implements ApplicationListener 的 Bean
  └─ 回放 earlyApplicationEvents → 缓存置 null
finishBeanFactoryInitialization：单例全部创建完 → SmartInitializingSingleton 回调 → @EventListener 注册
```

### 4.5 Transfer

- **"先订阅后广播"的时序定律**：任何发布-订阅系统，"事件在订阅者注册前发布"都会丢——解决办法三选一：延迟广播（缓存回放，Spring 的做法）、订阅者早注册（Boot 的 spring.factories/addListeners）、消费者从 offset 重放（MQ，迟到订阅者可以拉历史数据）。
- **启动时序 = 可调试资产**：把启动事件打印出来（demo09 的做法），任何"启动阶段行为异常"都能按事件定位——可观测性从"第一个日志"开始。

**本层留下的账**：事件从发布到监听的链路通了，但**"发布时事务还没提交，监听器读到的是旧数据"**怎么办？——下一层：事务事件。

---

<a id="level-5"></a>

## Level 5 事务事件：为什么"提交后才执行"

### 前置知识关卡

* [ ] 知道"事务提交后数据才可见"（读已提交语义）
* [ ] 知道 @Transactional 的切面执行时机（05 篇展开，这里只用到提交时机）

### 5.1 Why：事务内直接发布事件的账

徒弟：

> 我在 `@Transactional` 方法里发布事件，监听器里查数据库——为什么查不到刚写的数据？

老陈：

> 因为监听器同步执行时，**事务还没提交**——你自己在事务里查得到自己的写入，但监听器查到的是"已提交视图"（隔离级别 READ_COMMITTED 下）。
> 更危险的是反向：**监听器里的操作（发短信、调外部接口）在事务回滚时已经发生了**——事务最终回滚，但短信已经发出去了。
> `@TransactionalEventListener` 就是为这两笔账设计的：**把监听器推迟到事务提交之后再执行**——提交前不触发，提交成功才触发，回滚了就不触发。

| 方案 | 事务回滚时 | 数据可见性 | 适用 |
| ---- | ---- | ---- | ---- |
| 普通 @EventListener（同步） | 监听器动作已发生（无法撤回） | 读到未提交数据（视隔离级别） | 不依赖事务结果的动作 |
| @TransactionalEventListener(AFTER_COMMIT) | 不触发（省了无法撤回的动作） | 提交后可见（读到最终数据） | 依赖事务结果的动作（通知、MQ 发送、缓存刷新） |
| 事务消息（MQ） | 消息不发送 | 跨进程可靠 | 跨服务一致性（05/06 篇展开） |

### 5.2 代码实证：AFTER_COMMIT 的完整机制（demo08.TransactionalEventApp）

完整代码：`experiments/code/src/demo08/TransactionalEventApp.java`。

实测输出（`./run.sh demo08.TransactionalEventApp`）：

```text
[无事务] 发布 OrderPaidEvent（无事务边界）
[fallback] 监听器执行（fallback=true）
[无事务] AFTER_COMMIT(fallback=false) 执行=false；fallback=true 立即执行=true
[有事务] 模拟真实事务边界（同步上下文 + 真实事务激活）
[有事务] 发布后立即检查：两个监听器都未执行（提交前不触发）
[有事务] 注册的事务同步器数量=2
[提交] 手动触发 afterCompletion(STATUS_COMMITTED)
[提交] AFTER_COMMIT(fallback=false) 监听器执行
[fallback] 监听器执行（fallback=true）
[有事务] 提交后 AFTER_COMMIT 执行=true；fallback 执行=true
```

**十行输出钉死事务事件的四条机制**：

1. **无事务时不执行**（默认）：`fallbackExecution=false` 时，无事务边界发布事件 → 监听器**被丢弃**（第 2 行：只有 fallback=true 的执行）；
2. **fallback=true 补上无事务场景**：无事务时立即执行（第 2-3 行）——这是"发布方可能没开事务"时的兜底开关；
3. **有事务时"提交前不触发"**：发布后立即检查两个监听器都没执行——**发布 ≠ 执行**，只是注册了一个"提交回调"（第 6-7 行：注册了 2 个事务同步器）；
4. **提交成功才触发**：手动触发 `afterCompletion(STATUS_COMMITTED)` 后两个监听器才执行（第 8-9 行）。

### 5.3 深挖：AFTER_COMMIT 的触发点（6.1.14 反编译实证，本文最容易被忽视的细节）

反编译 `spring-tx 6.1.14` 的 `TransactionalApplicationListenerSynchronization$PlatformSynchronization`：

```java
// 反编译简化（javap 验证，6.1.14）
public void beforeCommit(boolean readOnly) {
    if (getTransactionPhase() == TransactionPhase.BEFORE_COMMIT) {
        processEventWithCallbacks();        // BEFORE_COMMIT：提交前执行
    }
}
public void afterCompletion(int status) {   // 注意：没有重写 afterCommit()
    if (getTransactionPhase() == TransactionPhase.AFTER_COMMIT && status == STATUS_COMMITTED) {
        processEventWithCallbacks();        // AFTER_COMMIT：提交成功（status==0）才执行
    }
    // AFTER_ROLLBACK：status == STATUS_ROLLED_BACK 才执行
    // AFTER_COMPLETION：无条件执行
}
```

**两个反直觉事实**：
1. **AFTER_COMMIT 不是"afterCommit 回调"**：6.1.14 里触发点是 `afterCompletion(STATUS_COMMITTED)`——**提交成功（status=0）才会执行，回滚（status=1）不执行**；语义是"事务以提交状态结束"，不是"提交动作发生的那一刻"；
2. **BEFORE_COMMIT 走 `beforeCommit`**：它执行时数据还没提交（脏数据视图），只适合"事务内动作"（如更新缓存锁）。

**demo 为什么用 `afterCompletion(STATUS_COMMITTED)` 手动触发**：真实提交路径由事务管理器在 commit 时调用同步器的 `afterCompletion`——demo 不引入数据库，用 `TransactionSynchronizationManager.initSynchronization() + setActualTransactionActive(true)` 模拟事务边界，再手动触发回调，**精确还原触发点**。

### 5.4 完整机制链（发布 → 提交 → 执行）

```text
@Transactional 方法内 publishEvent
  └─ 适配器 onApplicationEvent（6.1.14 TransactionalApplicationListenerMethodAdapter）：
        TransactionSynchronizationManager.isSynchronizationActive()   ← 事务同步上下文激活？
        && isActualTransactionActive()                                ← 真实事务激活？
        ├─ 都成立 → registerSynchronization(PlatformSynchronization)   ← 注册提交回调，不执行
        ├─ 否则 fallbackExecution? → 立即执行（兜底）
        └─ 否则 → 丢弃（无事务 + 无 fallback，demo 第 1-3 行实测）
  事务提交（事务管理器 commit）
  └─ TransactionSynchronizationUtils.invokeAfterCompletion(STATUS_COMMITTED)
        └─ PlatformSynchronization.afterCompletion(0) → processEventWithCallbacks → 监听器执行
```

**生产铁律**：**"事务内发布 + 提交后执行"要求监听器动作必须是"可重试、可补偿"的**——提交后执行不等于一定执行（提交后进程崩溃、回调抛异常），重要动作（发 MQ）应走事务消息或至少补任务表。

### 5.5 Transfer

- **"延迟到稳定点执行"**：任何"依赖某个事务/批处理结果的动作"，都应该绑定到"结果稳定点"而不是"动作发起点"——事务消息、Kafka 的 exactly-once、两阶段提交都是同一思想：**动作与结果的一致性绑定**。
- **兜底开关（fallback）**：默认丢弃 + 显式兜底是严谨的默认值设计——系统行为失败时应"安全失败"而不是"不确定地执行"。

**本层留下的账**：机制全通了，但生产里事件问题为什么最难排查？——最后一层：生产实践与排查方法论。

---

<a id="level-6"></a>

## Level 6 生产实践：事件排查方法论

### 前置知识关卡

* [ ] 知道同步/异步/早期/事务四个场景的机制（Level 2-5）
* [ ] 知道线程栈采样、日志链路（traceId）等基础排查手段

### 6.1 Why：事件问题为什么难排查

徒弟：

> 事件机制看起来就一个 publishEvent，出问题为什么那么难查？

老陈：

> 因为**调用链断了**：直接调用时，`A → B → C` 的调用栈清清楚楚；事件广播后，**发布者和监听器之间没有调用关系**——你不知道谁在听、谁没听、在哪个线程跑、什么时候跑。
> 所以排查事件问题的第一件事不是看代码，是**回答四个问题**：发了吗？谁在听？匹配了吗？执行了吗？——四个问题对应四个机制位点（Level 2-5）。

### 6.2 How："事件没生效"四问排查法

```text
问题：监听器没执行 / 执行了但结果不对
  ① 发了吗？→ 发布点打点 / 断点；确认 publishEvent 真的被调用了
       ├─ 发布点在事务内？→ 检查事务事件分支（Level 5）
  ② 谁在听？→ 打印 ApplicationEventMulticaster 的监听器注册表
       （调试：context.getApplicationEventMulticaster().getApplicationListeners()）
       └─ 监听器注册了吗？→ 检查 @EventListener 所在类是不是容器 Bean（没被扫描 = 最常见的"没注册"）
  ③ 匹配了吗？→ 检查事件类型与监听方法参数类型
       ├─ 监听父类 → 会收到子类事件（误收，坑 7）
       └─ 泛型事件（GenericEvent<T> 载荷）→ 全收误收：所有泛型参数的监听器都会执行（坑 9）
  ④ 执行了吗？→ 在监听器里打点
       ├─ 同步：执行了 → 看是否抛异常被外层吞掉
       ├─ 异步：是否异常被 AsyncUncaughtExceptionHandler 静默（坑 4）
       └─ 事务：提交了吗？回滚了吗？（AFTER_COMMIT 只有提交成功才执行，Level 5.3）
```

**最反直觉的一步**：**"没执行"的常见真相是"发布点根本没到"**——事务回滚导致提交后的代码没执行、`@Async` 方法里发布导致线程上下文丢失（事务同步器是线程绑定的！`TransactionSynchronizationManager` 是 ThreadLocal——**在异步线程发布事务事件，同步上下文不在**，事件被当"无事务"处理，坑 8）。先确认发布点，再查监听器。

### 6.3 三个高频生产问题的根因速查

| 现象 | 根因 | 机制位点 |
| ---- | ---- | ---- |
| 监听器抛异常，主流程失败 | 同步广播异常穿透 | Level 2.5 |
| 监听器执行了，但读不到刚写的数据 | 事务内发布 + 同步监听（未提交视图） | Level 5.1 |
| 事务回滚了，短信/通知却发出去了 | 事务内发布 + 同步监听器执行了外部动作 | Level 5.1 |
| 启动阶段事件收不到 | 早期事件 + @EventListener 注册晚 | Level 4.2 |
| 异步监听器悄悄失败 | 线程池吞异常（默认只打日志） | Level 3.2 |
| 事务事件没执行 | 发布时不在事务中 / fallback=false / 事务回滚 | Level 5.3 |
| 同一事件被处理两次 | **父子 Context 事件向上传播**（子 Context 发布的事件会传给父 Context 的监听器）——如果父子都注册了同一逻辑的监听器，或同一 Context 在两个体系里各注册一次，就重复处理（24 章 10 章对照，Boot 4.1 基线文档语义） | 事件传播 |
| 停机阶段发布的事件丢失/线程异常 | **Context 关闭中仍发布内存事件**——事件系统和资源正在关闭，任务可能丢失、监听器线程可能异常（24 章 10 章 P-E12，文档语义）；纪律：先拒绝生产（拒新），再 drain（与 07 篇 Level 8 停机协议同一套因果） | 关闭时序 |

**事件可靠性的固定边界**（24 章 10 章固定结论，文档语义）：

```text
Spring Event = 当前 JVM / ApplicationContext 内的分发机制（进程内、不持久化）
Outbox / MQ = 可靠、持久化、跨服务的投递机制
```

**因果**：Spring Event 解耦的是"当前进程内的调用关系"，不改变"跨进程可靠性"——进程崩溃、Context 关闭、异步队列丢弃都不会自动变成持久化消息。要跨服务可靠通知，先把事件写进业务表（Outbox），再由 Relay 投递（05 篇事务边界机制 + 07 篇 Level 8 强退兜底）。"重启后消息消失"不是事件机制的 bug，是**选错了投递机制**。

### 6.4 Transfer

- **"调用链断裂的系统必须留下决策记录"**：事件、消息、异步任务——凡是调用链不可见的地方，都要"发起点打点 + 执行点打点 + 失败点打点"，否则排查全靠猜（与 01 篇 Negative matches 是同一方法论：**自动化 + 可观测不可分割**）。
- **线程上下文是隐藏依赖**：事务（ThreadLocal 同步器）、安全上下文、traceId——跨线程传递必须显式设计（`TaskDecorator` 包装）——"异步化"从来不只是换线程，而是重新设计上下文传递。

**本层收账**：事件机制的完整因果链闭合——**发布（publishEvent）→ 多播器分发（匹配 + 排序）→ 同步/异步执行（异常语义）→ 早期缓存回放（时序）→ 事务绑定（提交后执行）→ 排查四问**。

---

<a id="summary-graph"></a>

## 全篇因果链总图

```text
publishEvent（贴告示）
   │ ① 多播器没就绪？→ earlyApplicationEvents 缓存（Level 4）
   │ ② 就绪 → ApplicationEventMulticaster.multicastEvent
   ▼
类型匹配（能容纳就收）
   ├─ ResolvableType.isAssignableFrom（父类监听收子类事件）
   ├─ @Order 排序（执行顺序）
   ▼
执行（同步 or 异步）
   ├─ 同步（默认）：发布线程直接调 → 异常穿透（Level 2）
   ├─ 异步（@Async/executor）：线程池调 → 异常静默（Level 3）
   └─ 事务事件：注册 TransactionSynchronization（提交回调）
         └─ afterCompletion(STATUS_COMMITTED) 才执行（Level 5）

监听器注册三姿势（启动时序决定能否收早期事件）：
   addApplicationListener（最早，收 run 事件）
   implements ApplicationListener（registerListeners 提前实例化，收回放）
   @EventListener（finishBeanFactoryInitialization 末尾 SmartInitializingSingleton 注册，收不到早期事件）

排查四问：发了吗？谁在听？匹配了吗？执行了吗？（Level 6）
```

**六字口诀**：

```text
贴告示人走，广播员分发，
同步必穿透，异步静默吞，
早期看注册，事务等提交，
排查四问走，链路不靠猜。
```

---

<a id="prod-cases"></a>

## 🏥 线上案例：三个真实复盘（全部命中本篇四层机制）

> 复盘格式：现象 → 根因 → 机制链 → 修复 → 预防。数字只做量级描述，不做精确承诺（待实测）。

### 案例 1：下单接口偶发超时，监控显示短信服务慢——同步广播的穿透账

- **现象**：下单接口 P99 突增，排查发现时间全耗在"发短信"上；短信服务偶发 3 秒+。
- **根因**：下单成功事件走 `@EventListener` **同步**监听，短信发送阻塞了主流程；短信慢 = 下单慢（demo08.EventSyncApp 第 1-5 行实测：监听器与发布者同线程）。
- **机制链**：Level 2.5——`SimpleApplicationEventMulticaster` 默认无 executor，`task.run()` 在发布线程执行。
- **修复**：短信监听器改为 `@Async`（姿势 B，Level 3.3）+ 自定义 `AsyncUncaughtExceptionHandler` 打点告警；下单接口不再等短信。
- **预防**：代码评审规则"事件监听器内禁止慢 IO，必须异步化"；监控加上"发布点返回时长 vs 监听器时长"。

### 案例 2：事务回滚了，用户却收到"支付成功"短信——事务内同步监听的账

- **现象**：某笔支付事务回滚，但用户收到了成功短信；对账发现短信发送在回滚前已发生。
- **根因**：支付成功事件在 `@Transactional` 方法内发布，`@EventListener` 同步执行短信发送——监听器动作发生在提交前，回滚无法撤回（Level 5.1 表：普通 @EventListener 的账）。
- **机制链**：Level 5.4——同步监听器在"注册同步器/提交回调"之前就直接执行了外部动作。
- **修复**：改 `@TransactionalEventListener(phase = AFTER_COMMIT)` + `fallbackExecution = false`——回滚不触发，提交成功才发短信（demo08.TransactionalEventApp 第 4-9 行实测）。
- **预防**：规则"依赖事务结果的动作一律事务事件或事务消息"；发布点评审"这个监听器在回滚时该不该发生？"

### 案例 3：监听 ApplicationStartingEvent 的 @EventListener 永远不执行——早期事件注册时序账

- **现象**：团队写了 `@EventListener(ApplicationStartingEvent.class)` 的监听器做启动埋点，日志显示从未执行。
- **根因**：`ApplicationStartingEvent` 是 run 的第一个事件，**容器还没创建**（demo09 第 1 行实测）；@EventListener 方法要在容器启动过程（finishBeanFactoryInitialization 末尾）才注册——**事件发生时订阅者根本不存在**（Level 4.2/4.3 的时序定律）。
- **机制链**：Level 4.3 时序全景——Starting/EnvironmentPrepared 发生在容器创建前；ContextInitialized/Prepared 发生在容器已创建但未 refresh 时（Bean 未注册）。
- **修复**：早期事件用 `SpringApplication.addListeners` 注册（demo09 的做法，实测收到全部 6 个 run 事件）；如果只是要"容器就绪后"做动作，直接用 `ApplicationRunner/CommandLineRunner`（实测位点：Started 之后、Ready 之前，比 @EventListener(ApplicationStartedEvent) 更直观）。
- **预防**：启动期行为与事件时机绑定前，先查"这个事件发生时容器注册到哪一步了"（demo09 的打印法）；不要用 @EventListener 监听 refresh 之前的事件。

---

<a id="interview"></a>

## 🎤 面试八股自查表（本篇范围的知其所以然）

| 八股问题 | 一句话因果答案 | 证据在哪 |
| ---- | ---- | ---- |
| Spring 事件机制是什么？ | 观察者模式容器化：publishEvent → 多播器按类型分发 → 监听器执行 | Level 2 |
| 事件默认同步还是异步？ | 同步（SimpleApplicationEventMulticaster 默认无 executor，run() 直接调） | Level 2.5 |
| 异步化有几种姿势？ | 广播器配 executor（全局）/ @Async 标注监听器方法（按方法，推荐） | Level 3.3 |
| 同步事件监听器抛异常会怎样？ | 异常直接冒泡给发布者（同步穿透，无隔离） | Level 2.2 实测 |
| 异步监听器抛异常会怎样？ | 被线程池吞掉（默认只打日志），发布者无感 | Level 3.2 实测 |
| @EventListener 怎么注册的？ | EventListenerMethodProcessor（SmartInitializingSingleton）在 finishBeanFactoryInitialization 末尾扫描方法生成 adapter | Level 2.3 |
| 为什么 @EventListener 收不到早期事件？ | 注册时机晚于 earlyApplicationEvents 回放 | Level 4.2 实测 |
| 早期事件怎么办？ | 缓存 + registerListeners 回放（implements ApplicationListener 的 Bean 能收到） | Level 4.2 实测 |
| Boot 启动广播哪些事件？ | Starting → EnvironmentPrepared → ContextInitialized → Prepared → （refresh）→ Started → Ready | Level 4.3 实测 |
| 前 4 个 run 事件谁收得到？ | addListeners 注册的监听器（容器未就绪，@Bean 收不到） | Level 4.3 |
| @TransactionalEventListener 什么时候执行？ | 事务以提交状态结束（afterCompletion(STATUS_COMMITTED)），不是提交动作瞬间 | Level 5.3 反编译 |
| fallbackExecution 是什么？ | 无事务边界时兜底立即执行；默认 false = 无事务时丢弃 | Level 5.2 实测 |
| 事务事件不生效先查什么？ | 发布时 isSynchronizationActive && isActualTransactionActive 是否成立 | Level 5.4 |
| 事件 vs MQ？ | 进程内（无持久化/不跨 JVM）vs 跨进程（持久化/可重放/事务消息） | Level 3.1 表 |
| 监听器执行顺序怎么控制？ | @Order / Ordered，多播器匹配后排序（AnnotationAwareOrderComparator） | Level 2.4 |
| 泛型事件为什么不推荐？ | 发布端拿不到运行时泛型参数（擦除 → raw 匹配），所有泛型监听器全收误收 | demo08.GenericEventApp 实测：String/Integer 都执行 |

---

<a id="pitfalls"></a>

## ⚠️ 坑与细节（10 个真实误解）

### 坑 1：Spring 事件默认是异步的

真相：默认同步（多播器 executor 为 null）。线上现象：以为发布返回就完事，实际发布线程被慢监听器拖住（demo08 第 1-5 行实测）。

### 坑 2：监听器抛异常不影响发布者

真相：同步穿透——异常直接冒泡，主流程跟着失败（demo08 第 6 行实测）。修复：监听器内 try-catch 或异步化。

### 坑 3：@EventListener 的方法类一定要是容器 Bean

真相：没注册成 Bean = 处理器扫描不到 = 监听器"不存在"。线上现象：写了 @EventListener 从不执行，查半天发现类没被组件扫描覆盖（排查四问的②）。

### 坑 4：异步监听器失败会告警

真相：默认被线程池吞掉（TaskUtils 记日志，无监控无告警）。线上现象：异步任务悄悄失败，业务受损却无感知。修复：自定义 AsyncUncaughtExceptionHandler + 告警。

### 坑 5：@TransactionalEventListener 没开事务也会执行

真相：默认 fallbackExecution=false——无事务边界发布 = 事件被丢弃（demo08 第 3 行实测）。

### 坑 6：AFTER_COMMIT 是"提交动作那一刻"执行

真相：6.1.14 实现是 afterCompletion(STATUS_COMMITTED)——提交**成功**才执行，回滚不执行（反编译实证）。语义是"事务以提交态结束"。

### 坑 7：监听父类事件只收父类

真相：匹配是"能容纳"（ResolvableType.isAssignableFrom）——监听 ApplicationEvent = 收所有事件（demo08 第 3 行实测），事件量大的系统会性能浪费/误处理。

### 坑 8：在异步线程发布事务事件

真相：TransactionSynchronizationManager 是 ThreadLocal——异步线程里没有事务同步上下文，事务事件被当"无事务"丢弃（fallback=false 时）。线上现象：@Async 方法里发布事务事件，监听器永远不执行。

### 坑 9：泛型事件能精确路由

真相：**不能——会"全收误收"**（demo08.GenericEventApp 实测：发布 `GenericEvent<String>`，监听 `<String>` 和 `<Integer>` 的两个监听器**都执行了**）。原因：发布端用 `ResolvableType.forInstance(event)` 推导事件类型，泛型类的运行时 Class 拿不到类型参数（擦除）→ 退化为 raw 类型 → raw 匹配所有泛型签名。线上现象：泛型事件广播给了一堆不该收的监听器。修复：用专用事件类（每个场景一个类），别用泛型容器当事件载荷；若坚持泛型事件，实现 `ResolvableTypeProvider` 手动提供类型（详见 2.4）。

### 坑 10：事件监听器里做事务操作没问题

真相：事务边界由 `@Transactional` 切面决定，与监听器无关——但**执行线程和传播行为**要设计：同步监听器在发布线程执行，其 `@Transactional`（默认传播 REQUIRED）可能**加入发布线程的现有事务**（如果发布点在事务内）或新开事务；异步监听器在池线程执行，一定是新事务。事务事件 + 监听器内事务的组合（如"提交后执行、但监听器自己又要开事务"）要显式设计传播行为（05 篇展开）。

---

<a id="errata"></a>

## 📚 版本勘误表

> 勘误格式：我曾讲错的 → 真相。区分 Specification / Implementation / 语义变化。

| 我曾讲错的 | 真相 | 性质 |
| ---- | ---- | ---- |
| "Spring 事件是异步的" | 默认同步：多播器 executor 为 null，run() 直接调；异步需显式配置 | Implementation |
| "监听器异常不影响发布者" | 同步穿透：异常直接冒泡（无 try-catch） | Implementation |
| "AFTER_COMMIT 在 afterCommit 回调触发" | 6.1.14 走 afterCompletion(STATUS_COMMITTED)——提交成功才触发（反编译 PlatformSynchronization） | Implementation |
| "@TransactionalEventListener 无事务也执行" | 默认 fallback=false 丢弃；fallback=true 才兜底执行 | Specification |
| "启动早期事件所有监听器都能收" | 只有先注册的能收：addListeners / implements ApplicationListener 收回放；@EventListener 收不到 | Implementation（实测） |
| "监听器类是 Bean 就一定被处理" | 必须等 EventListenerMethodProcessor 扫描；启动早期事件在它注册前已回放 | Implementation |
| "@EventListener 的处理器是 BeanPostProcessor" | 它是 SmartInitializingSingleton（javap 实证 6.1.14：implements SmartInitializingSingleton + BeanFactoryPostProcessor），处理时机在 afterSingletonsInstantiated——不是 BPP | Implementation（javap 验证） |
| "事务事件在异步线程发布照常生效" | 事务同步上下文是 ThreadLocal，异步线程没有——被当无事务处理 | Implementation |

坦白记录（我之前也讲错过）：

- 我讲过"AFTER_COMMIT 就是事务提交时调用 afterCommit 回调"——反编译 6.1.14 发现触发点是 `afterCompletion(status)` 的提交状态判断，且类名已重构为 `TransactionalApplicationListenerMethodAdapter` / `TransactionalApplicationListenerSynchronization`（6.x 代码组织变化，行为契约不变）。
- 我讲过"多播器 + executor 是唯一的异步方式"——@Async 标注监听器方法同样生效（demo08.AsyncEventApp 实测），且是生产推荐姿势。

---

<a id="decisions"></a>

## 🏆 生产决策卡

### Decision Card 1：事件还是 MQ

- **场景**：模块间要解耦通知。
- **判断**：进程内 + 无持久化需求 → 事件；跨进程 / 需要持久化、重放、削峰 → MQ。
- **Mechanism → Decision**：

```text
决策清单：
□ 发布者和监听器在同一个 JVM？→ 事件（否则 MQ）
□ 监听器失败需要重试/重放？→ MQ（事件无持久化，失败即丢）
□ 需要异步削峰？→ 两者都可（事件用线程池，MQ 更可靠）
□ 依赖事务结果？→ 事件用 @TransactionalEventListener；MQ 用事务消息（05/06 篇）
```

- **禁止决策**：禁止"先事件后改 MQ"的模糊中间态（两套机制并存，语义混乱）；禁止事件承载跨进程需求后手写 socket。
- **验收指标**：解耦度（新增动作零改动发布方）；失败可观测性（丢事件数 = 0，告警覆盖）。

### Decision Card 2：同步还是异步

- **场景**：新写一个事件监听器。
- **判断**：与核心结果有因果关系 → 同步（异常要穿透、时序要保证）；旁路增值动作 → 异步（@Async）+ 异常处理器告警。
- **Mechanism → Decision**：

```text
□ 监听器失败 = 核心流程失败？→ 同步（让错误暴露）
□ 监听器失败 = 旁路损失？→ 异步 + 告警
□ 监听器读"刚写入的数据"？→ @TransactionalEventListener(AFTER_COMMIT)（提交后可见）
□ 监听器对外部有副作用（短信/MQ）？→ 事务事件，回滚时不发
```

- **禁止决策**：禁止"怕阻塞就全局异步"（语义全丢）；禁止异步监听器不做异常处理（静默失败）。
- **验收指标**：主流程时延（异步化前后 P99 对比）；旁路动作失败率与告警及时性。

### Decision Card 3：事务事件的应用边界

- **场景**：事务内要触发外部动作（通知/MQ/缓存刷新）。
- **判断**：AFTER_COMMIT + 幂等设计；回滚时的补偿由"提交回调不触发"保证；不能承受"提交后进程崩溃"的 → 事务消息。
- **Mechanism → Decision**：

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, fallbackExecution = false)
public void onPaid(OrderPaidEvent e) {
    // 提交成功后：发 MQ / 清缓存（动作可重试、幂等）
}
```

- **禁止决策**：禁止在监听器里做"不可重试"的外部动作（提交后仍可能失败）；禁止把 fallbackExecution=true 当默认值（无事务时静默丢弃才是安全默认）。
- **验收指标**：回滚时外部动作触发次数 = 0；提交后动作失败重试成功率；幂等键冲突率。

---

<a id="cross-language"></a>

## 🌍 跨语言视角

| 语言/生态 | 做法 | 对应 Spring 的什么 | 账 |
| ---- | ---- | ---- | ---- |
| Java（Spring） | 容器内事件：多播器 + 注解驱动 + 事务绑定 | 事件机制本体 | 调用链断裂需打点补偿；默认同步 |
| Java（Guava EventBus） | 轻量进程内事件总线（无容器耦合） | 多播器 + @Subscribe | 无生命周期管理、无事务绑定 |
| Go | 无标准事件；channel/goroutine 手写 pub-sub；watermill 等库 | 观察者模式手写版 | 无框架级时序保证；胜在显式 |
| Rust | event-listener / async-broadcast crate | 同步/异步广播 | 借用检查器约束回调生命周期，显式而安全 |
| 前端（DOM） | EventTarget：addEventListener + dispatchEvent | 监听器注册表 + 发布 | 无进程内事务概念；传播机制（捕获/冒泡）更复杂 |
| Kafka（跨进程） | 主题 + 分区 + 消费组 | 事件机制 + MQ 的合体 | 持久化/重放/事务消息，运维重 |

**底层思想是否一致？**

一致的部分：

- **发布-订阅三件套**：注册（订阅）、匹配（类型/主题）、分发（执行）——任何语言任何规模都存在。
- **同步 vs 异步是执行策略**：广播语义相同，执行策略（线程、进程、队列）可插拔。
- **订阅时序定律**：先订阅后广播——Kafka 消费者组从 offset 消费、Spring 缓存回放、前端 DOM 冒泡，都在解决"订阅者就绪前的事件怎么办"。

不一致的部分：

- **事务绑定是 Spring 的独有能力**：事件与事务提交绑定（@TransactionalEventListener）在 Go/Rust/前端都不存在——它们要么手动推迟，要么根本没有"事务"概念。
- **自动化的代价**：Spring 注解把注册/匹配全自动化，代价是调用链不可见；Go/Rust 显式注册回调，代价是样板代码。

---

<a id="series-index"></a>

## 🧭 系列索引（00 篇为入口）

| 篇 | 主题 | 主线角色 | 比喻 | 本系列位置 |
| ---- | ---- | ---- | ---- | ---- |
| 00 | 容器如何创建对象 | 一个 Bean 的一生 | 地铁线路图 | 地基：创建链因果全通（15 demo 实测） |
| 01 | 框架整合 + 配置体系 | 待接入的框架 | 海关通关 | 在 00 的创建链上开扩展点；Level 4 完整展开配置体系 |
| **02（本篇）** | 事件机制与容器通信 | 一次业务事件 | 公告栏广播 | 发布-订阅通信机制；早期事件/事务事件/启动全景（demo08×5 + demo09 实测） |
| **03（已完成）** | 自动装配深挖 | 一个 starter 的生效过程 | 免签通道 | 候选收集/排除/排序/条件家族/覆盖通道/评估报告（demo10×6 实测） |
| 04（已完成） | Web 请求链路与运行时刻 | 一次 HTTP 请求 | 外卖配送 | 原 00 篇 Level 7 移入扩写（demo11×4 实测（含 WebFlux 双跑法）） |
| 05（已完成：demo12×4 实测） | 事务与数据层 | 一笔数据库事务 | 记账员 | 事务边界与数据层（含事务消息衔接） |
| 06（已完成：demo13×8 实测） | 横切面与 AOP | 一次方法调用 | 关卡哨兵 | 代理机制本体与切面体系（JDK/CGLIB 双分支实测） |
| 07（规划） | 生产实践 | 线上一次故障 | 急诊室 | 收束 |

---

# ✅ Final Review Checklist

- [ ] 是否解释了为什么存在？（直接调用三笔账 → 事件；同步阻塞账 → 异步；事务回滚连坐账 → 事务事件）
- [ ] 是否说明旧方案为什么失败？（直接调用耦合/连坐、轮询延迟/空转、事务内同步监听读到未提交/回滚无法撤回）
- [ ] 是否形成完整因果链？（发布 → 缓存/多播 → 匹配排序 → 同步/异步执行 → 事务绑定 → 排查四问）
- [ ] 是否区分规范和实现？（@EventListener/@TransactionalEventListener 契约、提交后执行语义为 Specification；广播器结构、缓存回放、注册链、afterCompletion 触发点为 6.1.14 Implementation）
- [ ] 是否区分语义变化与代码组织变化？（6.x 事务事件类名重构为代码组织变化，行为契约不变）
- [ ] 代码实例是否全部实测？（demo08×5、demo09 输出原样引用，可复跑；6.1 事务事件触发点经 javap 反编译验证）
- [ ] 是否包含 Trade-off？（同步穿透 vs 异步静默；事件 vs MQ；事务事件 vs 事务消息；fallback 兜底 vs 静默丢弃）
- [ ] 是否能指导生产决策？（3 张决策卡：事件 vs MQ / 同步 vs 异步 / 事务事件边界）
- [ ] 是否存在未经证明的数字？（无编造 benchmark；Boot 3 的 spring.factories 加载 ApplicationListener 路径标注待验证）
- [ ] 是否只有一个比喻？（公告栏广播）是否只有一个主线角色？（一次业务事件）
- [ ] 随机抽查断言：同步穿透（demo08 实测）、异步线程（demo08 实测）、早期事件回放（demo08 实测）、Boot 事件顺序与 Runner 位点（demo09 实测）、AFTER_COMMIT 触发点（javap 反编译 6.1.14）、泛型事件全收误收（demo08.GenericEventApp 实测）、EventListenerMethodProcessor 是 SmartInitializingSingleton（javap 验证 6.1.14）——均有证据来源。
