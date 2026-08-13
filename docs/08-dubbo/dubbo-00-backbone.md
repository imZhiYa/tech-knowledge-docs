# 🍜 Dubbo 骨干：订单 O 的一生——一次 RPC 调用到底发生了什么（系列入口 00）

> 本篇文章只回答一个问题：**一次 RPC 调用（订单 O）从 `@DubboReference` 到 provider 方法返回，沿途每一站挂载什么机制？为什么核心是 Invoker，而不是 Proxy 或注解？**
> 本文是系列主干：先立起"一次调用的一生"总图，01~08 篇是对各站的逐章打穿。

---

# 🏷️ 关键词

RPC / Invoker / Protocol / Exporter / Invocation / Proxy 透明化 / Filter 链 / SPI 微内核 / 服务发现 / 负载均衡 / 容错 / 序列化 / Triple / HTTP/2 / 业务线程池 / 分布式全景

---

# 🗂️ 目录

- [能力地图](#level-1)
- [全文唯一比喻地图](#level-2)
- [Level 1 上一代方案死在哪：从 `new` 到 2.7.x 的五笔账](#level-3)
- [Level 2 O 的起点：核心领域模型——为什么是 Invoker](#level-4)
- [Level 3 O 的调度：查门店表 + 分单/派单/改派](#level-5)
- [Level 4 O 的运输：打包 + 配送信封](#level-6)
- [Level 5 O 到后厨：执行与回程](#level-7)
- [Level 6 全链路总图 + 从 O 看分布式全景](#level-8)
- [合书自测](#level-9)
- [坑与细节](#level-10)
- [版本勘误表](#level-11)
- [生产决策卡](#level-12)
- [跨语言 / 跨运行时视角](#level-13)

---

# 为什么需要这篇

徒弟：

> 我用 `@DubboReference` 调服务调了三年，注解写上就能用，为什么还要研究"一次调用发生了什么"？

老陈：

> 因为你线上踩的坑，九成不是注解没背熟，而是**不知道订单 O 沿途经过哪些站**：
> - 服务注册了一堆地址，调用却总超时——你以为是网络，其实是被代理软件劫持了 host（E00 实测）；
> - failover 重试了 2 次，订单被扣了 3 次库存——你根本不知道重试发生在哪一站；
> - 换了 Triple 协议性能没变好——你不知道协议的优势边界在哪；
> - 泛化调用内存涨上天——你不知道它绕开了什么（06 篇）。
>
> 这一篇把调用链的每一站立起来：每站挂什么机制、每站留下什么账。后面的 01~08 篇，就是在每一站上打深井。

---

<a id="level-1"></a>

## 📍 能力地图

| 层级 | 要打穿的认知墙 | 通关标准 |
|---|---|---|
| 旧方案与框架收编 | "发个 HTTP 就算远程调用" | 说出五笔账各自的死法，以及 RPC 框架统一收编的问题清单（透明化/编解码/传输/地址/决策/失败/治理/运维）。 |
| 领域模型 | "核心是注解，魔法在 Proxy" | 背出官方原文：Protocol + Invoker + Exporter 三件套即可完成非透明 RPC；Proxy 只做透明化。 |
| 调度（发现/路由/容错） | "每笔请求都查一次注册中心" | 说出门店表是本地维护的候选集；Router=谁有资格、LoadBalance=选谁、Cluster=失败怎么办。 |
| 运输（序列化/协议/网络） | "协议就是序列化" | 说出信封（边界/配对/状态）与盒子（对象↔字节）正交，协议与序列化可任意组合。 |
| 执行与回程（线程/Filter） | "业务跑在 IO 线程上" | 说出业务在 DubboServerHandler 线程池执行、Filter 与业务同线程（E00 实测）。 |
| 全链路总图 | "会调，但画不出链路" | 画出订单 O 六站总图，并把每站映射到一类分布式通用问题（一致性/可用性/失败语义/成员变更/治理）。 |

---

<a id="level-2"></a>

## 🏭 全文唯一比喻地图

一句话总览：

```text
顾客下单（业务代码）→ 收银员接单（Proxy）→ 验单台（Consumer Filter）
  → 查门店表（本地地址集合）→ 分单/派单/改派（Router/LoadBalance/Cluster）
  → 打包（序列化）→ 装信封（协议）→ 骑手送达（Netty）
  → 后厨接单出餐（Exporter + 线程模型 + 业务实现）→ 回执送回顾客（Result）
```

**比喻元素 ↔ Dubbo 技术概念映射表**（每个元素只负责一件事）：

| 比喻元素 | Dubbo 技术概念 | 它负责的唯一一件事 |
| ---- | ---- | ---- |
| 顾客订单 O | 一次 `Invocation`；常见实现视角为 `RpcInvocation` | 携带方法、参数和调用上下文，表达"我要扣库存" |
| 前台收银员 | Consumer Proxy + Invoker | 将本地接口调用接入 Dubbo 调用链 |
| 验单台 | Consumer Filter | 在出站前做追踪、鉴权、超时预算、上下文传播等横切处理 |
| 可接单门店表 | Consumer 本地维护的 Provider 地址集合 | 为调用提供候选实例；**不是每笔请求临时查注册中心** |
| 总部门店系统 | 注册中心 / 服务发现 | 发布并传播实例、地址及相关服务发现信息的变更 |
| 分单规则 | Router | 决定订单 O **允许去哪些**实例 |
| 派单算法 | LoadBalance | 从允许实例中选择**具体哪一个** |
| 改派政策 | Cluster 容错 | 定义失败后是返回、重试、忽略还是采取其他策略 |
| 打包盒 | 序列化：Protobuf、Hessian2、JSON 等 | 将参数和结果编码为字节 |
| 配送信封 | Triple / HTTP/2、Dubbo 协议等 | 规定消息边界、头信息、状态与流控等通信语义 |
| 骑手 | Netty EventLoop / Channel | 非阻塞地处理连接、读写事件与连接复用 |
| 后厨 | Provider Exporter、服务端分派与业务实现 | 定位并执行真正的 `greet()` / `deduct()` 业务逻辑 |
| 出餐调度 | Provider 执行资源 / 线程模型 | 控制执行并发、排队、隔离和拒绝 |
| 回执 | `Result`；常见实现视角为 `RpcResult` | 将成功值或异常语义返回给 Consumer |
| 菜单与规章 | 配置中心、元数据中心 | 下发治理规则、维护服务契约和映射信息 |
| 运营看板 | Metrics / Tracing / Logs | 为一次订单的路径和异常提供可验证证据 |

> 📌 **比喻边界（祛魅）**：
> - 骑手**不是**"一次请求一个线程"。高并发 I/O 恰恰依赖少量 EventLoop 线程复用连接、处理很多读写事件（07 篇展开）。
> - 门店表**不是**每笔请求查注册中心：Consumer 本地维护（订阅 + 缓存 + 增量通知），一次调用只在本地内存里挑实例（03 篇展开）。
> - 改派**不是**无限重试：有次数上限、超时与幂等约束——否则就是"failover 重试 2 次，订单被扣 3 次库存"的事故（05/08 篇展开）。
> - 打包盒**不是**信封：序列化和协议是两个正交维度，可以任意组合（02 篇展开）。
> - 收银员**不炒菜**：Proxy 只做透明化转发，不执行任何业务逻辑（2.4 官方原文）。

**全文只使用这一个比喻。** 订单 O = 一次 RPC 调用（`RpcInvocation`），是本系列唯一主线角色。

---

# 正文：六个 Level 打穿"一次 RPC 调用"

<a id="level-3"></a>

## Level 1 上一代方案死在哪：从 `new` 到 2.7.x 的五笔账

### 👶 前置知识关卡

* [ ] 知道"远程调用"与"本地调用"的区别（跨进程、网络传输）
* [ ] 知道序列化（对象 → 字节）与反序列化（字节 → 对象）
* [ ] 知道 TCP 连接、HTTP 协议的基本概念

### 1.1 Why：没有 RPC 框架的时代

徒弟：

> 调用另一个服务的代码，不就是"发个 HTTP 请求"吗？为什么非要搞个 RPC 框架出来？

老陈：

> HTTP 确实能跨进程，但它把每一件小事都留给了你：连接谁（IP 写死？）、怎么编码（JSON 手拼？）、超时多久、失败重不重试、对方挂了你怎么办、换了台机器你改不改配置？RPC 框架不是"换一种发请求的方式"，而是**把"远程调用"这一整类问题的答案，从每个项目复制粘贴的代码，收进一个框架统一解决**。

### 1.2 五笔账：旧方案各自的死法

| 方案 | 解决什么 | 留下的致命账 |
| ---- | ---- | ---- |
| 本地 `new` / 同进程调用 | 简单 | 服务拆开跨进程后**完全不可见**：没有地址、没有网络语义、没法独立部署 |
| 手动 HTTP + JSON（RestTemplate/HttpClient） | 跨进程可用 | 连接管理散落各处、IP 写死（实例变更=改代码）、无负载均衡、超时/重试/错误处理各写各的、无治理 |
| RMI（Java 原生远程调用） | 跨 JVM 方法调用 | 绑定 Java 语言、序列化弱、穿透防火墙难、无服务治理 |
| WebService / SOAP | 跨语言 | 重、慢、XML 膨胀、调试地狱 |
| Dubbo 2.7.x | 治理能力齐备 | URL 爆炸（2 万条/消费端常驻内存 40%）、私有协议穿透差、跨语言难、泛化调用内存问题（06 篇清算） |

**为什么前四笔账注定要还**：它们把"远程调用"拆成了无数个"你自己维护的细节"。服务从 1 个变 10 个、实例从 1 台变 100 台时，这些细节的维护成本指数上涨。需要一个框架把以下能力**统一收编**：

```text
RPC 框架统一收编的问题清单
├── 透明化：调远程像调本地（Proxy）
├── 编解码：序列化协议（包在盒子里）
├── 传输：连接复用、异步化（配送信封）
├── 地址：服务发现 + 注册中心（门店表）
├── 决策：负载均衡 + 路由（分单/派单/改派）
├── 失败：容错策略（改派/放弃）
├── 治理：限流、熔断、降级、可观测（总部规则）
└── 运维：优雅上下线、配置中心（打烊/规章）
```

**2.7.x 为什么也算"上一代"**：它把上述问题都解决了，但解决方式留下新账——**接口级服务发现**让注册中心存了巨量冗余 URL（一个实例的每个接口一条，官方 benchmark（工行团队）数据：500 实例 × 8 接口 × 5 协议 = 2 万 URL），消费端常驻内存超 40%；**私有二进制协议**（TCP + Hessian2）让 Dubbo 服务难以穿过云原生网关、难以跨语言。3.x 的两次大手术（应用级发现 + Triple 协议）就是清算这两笔账（03/01 篇）。

### Transfer——迁移到其他设计问题

- **"统一收编问题清单"是中间件存在的通用逻辑**：缓存中间件收编"读放大"，消息中间件收编"异步解耦"，RPC 框架收编"远程调用细节"。评估任何中间件，先问"它收编了哪类问题的哪些细节"。
- **演进链 = 每代方案都是上一代的还账**：2.7 的账（URL 爆炸/私协议）→ 3.x 的手术（应用级发现/Triple）→ 08 篇的选型决策全部基于这条链。

> 本层留下的新账：知道需要一个框架，但**调用是怎么发起的、核心模型是什么**还没讲——这逼出下一层：Consumer 端装配与核心领域模型（Level 2）。
>
> 🔴 **口诀**：框架不是"换一种发请求的方式"，是把远程调用的细节统一收编成体系；每代新方案都在还上一代的账。

---

<a id="level-4"></a>

## Level 2 O 的起点：核心领域模型——为什么是 Invoker

### 👶 前置知识关卡

* [ ] 知道动态代理（对接口生成代理类，拦截方法调用）
* [ ] 知道 SPI（接口 + 配置文件声明实现，运行时加载）

### 2.1 祛魅开场：直觉陷阱

徒弟：

> Dubbo 的魔法肯定是 `@DubboReference` 注解和动态代理——注解一写，远程调用就透明了。

老陈：

> 先打碎这个直觉。官方架构文档里说得很清楚：**RPC 的核心不是 Proxy，不是注解，而是 Invoker**。Proxy 只是把 Invoker "包装成业务接口"的皮；注解只是配置装配的入口。这层认知不建立，后面看任何 Dubbo 源码都会迷路。

### 2.2 官方核心领域模型（Specification，官方「代码架构」文档原文）

> **Invocation 是会话域**，它持有调用过程中的变量，比如方法名，参数等。
>
> **Invoker 是实体域**，它是 Dubbo 的核心模型，其它模型都向它靠拢，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
>
> **Protocol 是服务域**，它是 Invoker 暴露和引用的主功能入口，它负责 Invoker 的生命周期管理。

**用比喻钉死三个域**：

```text
订单 O（Invocation 会话域）  = 订单小票：方法名 + 参数 + 附加信息
可执行体（Invoker 实体域）   = 能干活的人/机器：本地店员 / 远程分店 / 一群分店（集群）
Protocol（服务域）           = 门店体系的注册机关：负责"开门（export）"和"发工牌（refer）"
```

> 对照映射表：**同一个 Invoker 在表里被拆成两端**——消费端叫"收银员"（Proxy + Invoker），提供端叫"后厨"（Exporter + 业务 Invoker）。一张订单小票（O）从收银员手上递到后厨，中间的验单、派单、运输全是围绕它展开。

三个域的关系一句话：**Protocol 负责造出 Invoker（refer/export），Invoker 接受 Invocation 并执行，执行结果就是 RpcResult**。

### 2.3 官方核心层：三件套即可完成非透明 RPC（Specification，原文）

> 在 RPC 中，Protocol 是核心层，也就是**只要有 Protocol + Invoker + Exporter 就可以完成非透明的 RPC 调用**，然后在 Invoker 的主过程上 Filter 拦截点。

| 三件套 | 职责 | 官方契约要点 |
| ---- | ---- | ---- |
| `Protocol` | `export(invoker)` 暴露服务；`refer(type, url)` 引用服务 | refer 返回的 Invoker 由协议实现，**通常在其内部发送远程请求**；export 收到的 Invoker 由框架提供，协议不关心 |
| `Invoker` | 可执行体，`invoke(invocation)` | 本地实现 / 远程实现 / 集群实现三种形态 |
| `Exporter` | `export()` 的返回值 | 持有 Invoker 的引用，用于 `unexport()` 撤销暴露 |

**"非透明"三个字是关键**：没有 Proxy 时，调用方手里是裸的 Invoker，你得手动 `invoker.invoke(new RpcInvocation(...))`——能跑，但不像调本地方法。**透明化是 Proxy 的事，不是协议的事**（2.4）。

### 2.4 官方 Proxy 层：透明化的皮（Specification，原文）

> Proxy 层封装了所有接口的透明化代理，而在其它层都以 Invoker 为中心，只有到了暴露给用户使用时，才用 Proxy 将 Invoker 转成接口，或将接口实现转成 Invoker，也就是**去掉 Proxy 层 RPC 是可以 Run 的**，只是不那么透明，不那么看起来像调本地服务一样调远程服务。

```text
消费者端：Invoker →（ProxyFactory.getProxy）→ 业务接口，业务代码像调本地一样调用
提供者端：业务实现 →（ProxyFactory.getInvoker）→ AbstractProxyInvoker，交给 Protocol.export
```

**一句话**：Proxy 是"外卖下单页面"，Invoker 是"后厨接单的人"。页面可以换（JDK 代理/Javassist/自定义），后厨才是干活的。

### 2.5 官方 Filter 层：Invoker 主流程上的拦截点（Specification，原文）

> 在 Invoker 的主过程上 Filter 拦截点。

**官方设计里这些横切功能是 Filter 的一环**：超时执行（TimeoutFilter）、mock（MockClusterInvoker）、泛化（GenericImplFilter/GenericFilter）、echo（EchoFilter）、token（TokenFilter）、accesslog（AccessLogFilter）、monitor（MonitorFilter），通过 SPI 挂在 Invoker 主流程上。（注意边界：**重试不属于 Filter，是 Cluster 层** FailoverClusterInvoker 的循环，05 篇；**内置限流类 Filter 是并发/执行数维度**——ActiveLimitFilter（actives）/ ExecuteLimitFilter（executes）/ TpsLimitFilter（tps），3.3.4 `dubbo-rpc-api` 实证；全局维度 QPS/速率治理需外部网关或扩展，06 篇。）这意味着：

```text
业务调用
  ▼
Proxy 代理
  ▼
Filter 链（消费者侧）：ConsumerContext → ... → 你的业务 Filter
  ▼
ClusterInvoker（伪装成单个 Invoker 的集群，Level 3）
  ▼
协议层 Invoker（真正发网络请求的，Level 4）
  ───────── 网络 ─────────
协议层 Invoker（接收请求）
  ▼
Filter 链（提供者侧）：Echo → ClassLoader → ... → 你的业务 Filter
  ▼
业务 Invoker（AbstractProxyInvoker，调你的实现类）
```

**E00 实测**（`experiments/experiment-000-callchain.md`，原样引用）：

```text
Consumer 端：
  proxy class = org.example.dubbo.demo.api.GreetingServiceDubboProxy0
  [CONSUMER TraceFilter] enter, thread=ConsumerApp.main(),
      invoker=org.apache.dubbo.rpc.listener.ListenerInvokerWrapper, method=greet, ...
Provider 端：
  [PROVIDER TraceFilter] enter, thread=DubboServerHandler-198.xx.x.x:50051-thread-2,
      invoker=...FilterChainBuilder$CopyOfFilterChainNode, method=greet, ...
  [PROVIDER Business] greet() executing, thread=DubboServerHandler-198.xx.x.x:50051-thread-2, ...
```

四行输出钉死四个 Implementation 事实（3.3.4）：

1. **代理类名 `GreetingServiceDubboProxy0`**——不是 JDK 动态代理的 `$Proxy0`，Dubbo 默认用 Javassist 生成，类名 = 接口名 + `DubboProxyN`；
2. **Consumer 侧 Filter 入口看到的 Invoker 是 `ListenerInvokerWrapper`**——协议包装链（ProtocolListenerWrapper 等）包在最外层，业务 Filter 在它内部；
3. **Provider 侧 Filter 节点是 `FilterChainBuilder$CopyOfFilterChainNode`**——3.x 集群 Filter 链的构建方式；
4. **线程命名 `DubboServerHandler-<host>:<port>-thread-N`**——Provider 业务线程池的线程，Filter 链与业务方法在同一线程执行（07 篇展开）。

### 2.6 官方设计理念：Microkernel + Plugin（Specification，原文）

> 采用 Microkernel + Plugin 模型；微内核只负责组装插件。Dubbo 的功能本身也是通过扩展点实现的，意味着 **Dubbo 的所有功能点都可以被用户自定义和替换**。

**这就是"换协议/换注册中心/换负载均衡都只是改配置"的根本原因**：Protocol、Registry、LoadBalance、Cluster、Filter、Serialization 全是扩展点，各自有 SPI 配置文件声明实现。E00 的意外收获是这条规则的活例子：

> 配置协议时写 `"triple"` 报错 `No such extension org.apache.dubbo.rpc.Protocol by name triple`——查 SPI 文件才发现 3.3.4 里 Triple 协议的注册名是 **`tri`**（`tri=org.apache.dubbo.rpc.protocol.tri.TripleProtocol`）。**SPI 名字是配置文件里写的，不是注解语义**（Implementation，3.3.4；报错格式已用 javap 反编译 `ExtensionLoader.class` 验证：`No such extension <type> by name <name>`）。

### 2.7 装配总图：O 的出发仪式

```text
@DubboReference / ReferenceConfig
  ├─→ Protocol.refer(type, url)          ← 问"哪个协议、什么地址"（服务域）
  │      └─→ 协议实现返回 Invoker        ← 远程实现（内部带网络客户端）
  ├─→ 包上 Cluster（伪装成一个 Invoker） ← 外围概念（Level 3/05 篇）
  ├─→ 包上 Filter 链                    ← 拦截点（2.5）
  └─→ ProxyFactory.getProxy(invoker)    ← 透明化：变成业务接口
        └─→ 业务代码调用 service.greet(...)
              └─→ O（RpcInvocation）诞生，进入 Invoker.invoke(O)
```

### Transfer——迁移到其他设计问题

- **任何框架都要分清"核心抽象 vs 外包装"**：gRPC 的 Channel/Stub、Spring 的 BeanFactory vs 注解——注解/代理是装配的皮，抽象才是骨；读源码先找骨。
- **Microkernel + Plugin（SPI）是中间件的标准架构**：SLF4J、JDBC Driver 都是"接口 + 配置文件声明实现"，换实现不改代码。
- **可插拔 = 把"会变的"与"不变的"分家**：协议、注册中心、负载均衡是变化点，隔离在扩展点后，核心链路保持稳定。

> 本层留下的新账：核心模型立起来了，O 要出发了——**但它要打给"谁"**？需要知道哪些门店在营业。这逼出下一层：调度（Level 3）。
>
> 🔴 **口诀**：核心是 Invoker，Proxy 只是皮；认准"抽象、装配、包装"三层，看源码不迷路。

---

<a id="level-5"></a>

## Level 3 O 的调度：查门店表 + 分单/派单/改派

### 👶 前置知识关卡

* [ ] 知道注册中心的概念（服务地址的集中登记处）
* [ ] 知道"路由"和"负载均衡"的区别（选谁 vs 怎么分配）

### 3.1 Why：不能写死一个地址

徒弟：

> 服务地址写配置文件里不就行了？要什么注册中心？

老陈：

> 写死地址 = 回到 Level 1 手动 HTTP 的账：**实例变更、扩缩容、故障剔除，都要改配置重启**。注册中心的价值是"**门店营业状态表**"：谁在营业、地址在哪、状态变了即时通知。调用方永远问表，不记地址。

### 3.2 分单/派单/改派三连（简版，05 篇打穿）

```text
O 的调度链（AbstractClusterInvoker 内部）
  ├─① Directory.list()         ← 拿到该服务的全部可用 Invoker（门店表快照）
  ├─② Router.route()           ← 路由：过滤掉不接单的门店（分组/标签/条件，06 篇）
  ├─③ LoadBalance.select()     ← 负载均衡：从剩余门店里选一家（随机/轮询/P2C，05 篇）
  └─④ 容错策略接管              ← 选中的店不接单？failover 改派 / failfast 直接报错（05 篇）
```

**官方原文定调 Cluster 的定位**（Specification）：

> Cluster 是外围概念，Cluster 的目的是**将多个 Invoker 伪装成一个 Invoker**，这样其它人只要关注 Protocol 层 Invoker 即可，加上 Cluster 或者去掉 Cluster 对其它层都不会造成影响，因为只有一个提供者时，是不需要 Cluster 的。

**这句话是 05 篇的总纲**：Cluster 不是协议的一部分，是"伪装术"——业务代码只看到"一个服务"，背后是一群实例 + 路由 + 负载均衡 + 容错。

> **祛魅**：上面整套"查表 → 分单 → 派单 → 改派"发生在 Consumer 进程内，不产生任何网络请求。门店表是**本地内存里的候选地址集合**（订阅注册中心增量推送维护），一次调用只做本地内存操作 + 一次真正的网络调用（打给最终选中的那一家）。"每笔请求都去问注册中心"是新手常见误解（03 篇展开）。

### 3.3 门店表的两种画法（简版，03/04 篇打穿）

| 画法 | 机制 | 账 |
| ---- | ---- | ---- |
| 2.7.x 接口级 | 每个接口一条地址数据（RPC Service → Instance） | URL 爆炸：2 万条、消费端内存 40%、推送风暴 |
| 3.x 应用级 | 每个实例一条（Application → Instance），元数据另存 | 注册中心瘦身（内存 -75% 官方数据），代价是引入 MetadataService 点对点拉取 |

### Transfer——迁移到其他设计问题

- **"本地缓存 + 增量通知"是成员变更的通用解法**：K8s watch、DNS TTL、Eureka 二级缓存都是"查本地、不查远端"——一次调用不能依赖实时查询注册中心。
- **决策分层的判据：变化频率不同就分层，相同才合并**：资格（Router）跟业务策略变、选择（LoadBalance）跟运行时状态变、兜底（Cluster）跟成本预算变——写死在一起就是改一个需求动一次调用方代码。
- **候选集是容错的原料**：没有候选集就没有"换一家"的可能（05 篇展开）。

> 本层留下的新账：分单派单选好了门店、地址也对，但**"订单怎么运过去"**？——打包 + 配送信封。这逼出下一层（Level 4）。
>
> 🔴 **口诀**：查表查本地、决策分三层；资格、选择、兜底三个时点三种账。

---

<a id="level-6"></a>

## Level 4 O 的运输：打包 + 配送信封

### 👶 前置知识关卡

* [ ] 知道序列化（对象 ↔ 字节）
* [ ] 知道 HTTP/2 的多路复用与 TCP 长连接

### 4.1 打包盒：序列化（简版，02 篇打穿）

O（方法名 + 参数 + 附加信息）在网络上是字节，怎么编码由序列化器决定：

| 序列化 | 定位 | 关键账 |
| ---- | ---- | ---- |
| Hessian2 | 2.x 默认，Java 原生对象友好 | 性能一般、跨语言弱、反序列化安全面（06 篇） |
| Protobuf | Triple 首选，IDL 约束 | 强契约、跨语言、体积小；需要 `.proto` 定义 |
| JSON/Fastjson2 | 简单场景/网关 | 人类可读但慢且无类型约束 |

**注意**：序列化和协议是两个维度——打包盒 ≠ 配送信封。dubbo 协议可以配 JSON 序列化，Triple 也可以配 Hessian2/JSON（协议与序列化是正交的 SPI 扩展点，02 篇展开组合实测）。

### 4.2 配送信封：协议（简版，01 篇打穿）

| 通道 | 基线定位 | 优势边界 |
| ---- | ---- | ---- |
| dubbo 协议（TCP + 私有二进制） | 2.7 默认，3.x 仅迁移对照 | 直连场景吞吐略优（官方 benchmark），但穿透差、跨语言难 |
| **Triple（HTTP/2 + Protobuf，兼容 gRPC）** | **3.x 主推** | 网关穿透、Stream、跨语言、云原生（01 篇祛魅：直连性能无优势） |

**配送靠"骑手"（Netty EventLoop）**：骑手不是"一次请求一个骑手"——少量 EventLoop 线程复用连接、非阻塞地处理大量读写事件，一次调用不独占一条连接（07 篇展开）。

> **祛魅**：别把"高并发"想象成"线程越多越快"。Dubbo 的 IO 部分恰恰是**少量线程复用连接**：1000 个并发请求共用几条连接、几个 EventLoop 线程，谁的事件就绪谁被处理。真正吃线程的是后厨（业务线程池），那是另一笔账（07 篇）。

### 4.3 E00 实测佐证（Transport 层）

Provider 导出日志（3.3.4 DEBUG）：

```text
Start NettyPortUnificationServer bind /0.0.0.0:50051
Using dubbo protocol to export org.apache.dubbo.metadata.MetadataService service on port -1
Use random available port(20880) for protocol dubbo
```

**意外事实**：即使业务只用 Triple，Provider 仍会为内置的 `MetadataService`（应用级服务发现的元数据服务，03 篇）开一个 dubbo 协议的随机端口（20880）——这就是 03 篇要打的"元数据拉取链路"。

### Transfer——迁移到其他设计问题

- **"信封与盒子正交"是网络中间件的通用切分**：gRPC、Thrift、Kafka 都分"帧格式 + 序列化"两层，先分清两层再谈性能（01/02 篇打穿）。
- **协议答六问**：边界、配对、元数据、状态、复用、协商——任何传输层设计都是这六问的答案集。
- **私有 vs 开放的取舍**：性能收益 × 生态锁定 vs 互操作性 × 接入成本；先问"我的生态里谁要读这封信"。

> 本层留下的新账：订单运到后厨了，**谁接单、怎么执行、怎么回**？这逼出下一层（Level 5）。
>
> 🔴 **口诀**：先分信封和盒子，再谈性能；协议问六件事，答法决定生态。

---

<a id="level-7"></a>

## Level 5 O 到后厨：执行与回程

### 👶 前置知识关卡

* [ ] 知道线程池
* [ ] 知道"同步调用"与"异步回调"的区别

### 5.1 Provider 端五步（简版，07 篇打穿）

```text
① Netty 收到字节（IO 线程）        ← 骑手送到门口
② 反序列化还原 O（RpcInvocation） ← 拆包
③ 派给业务线程池                   ← 后厨接单（E00：DubboServerHandler-...-thread-2）
④ Filter 链（提供者侧）逐站安检    ← 后厨的工序检查
⑤ AbstractProxyInvoker 调业务方法 ← 真正炒菜
⑥ 结果（RpcResult）序列化原路返回  ← 出餐 + 回执
```

### 5.2 E00 实测（原样引用）

```text
[PROVIDER TraceFilter] enter, thread=DubboServerHandler-198.xx.x.x:50051-thread-2, ...
  [PROVIDER Business] greet() executing, thread=DubboServerHandler-198.xx.x.x:50051-thread-2, ...
[PROVIDER TraceFilter] exit, result=Hello O (sequence=1), cooked by DubboServerHandler-198.xx.x.x:50051-thread-2
```

两个事实：

1. **业务线程池与 IO 线程是两类线程**：请求由 IO 线程接收，业务在 `DubboServerHandler-...` 线程池执行；3.3.4 triple 的默认实现是 `limited`（max=200、core=0），详见 07 篇与 E07；
2. **Filter 链与业务方法在同一线程**：Filter 在业务线程上执行，不是额外线程（这是"Filter 里别做耗时操作"的原因）。

### 5.3 回程

O 不原路回：Provider 的 `RpcResult` 序列化后走同一连接返回，Consumer 端由调用线程（同步模式）或回调（异步模式）取回。E00 同步模式下调用线程就是 `main`（见 Consumer 端实测）。

> 说人话：同步调用 = 顾客站在柜台等出餐（调用线程阻塞等回执）；异步调用 = 先回家等电话（回调）。同一个骑手（连接）可以同时送很多单，所以"等出餐"不占运输通道（07 篇展开）。

### Transfer——迁移到其他设计问题

- **IO 线程与业务线程分离是高并发服务器的通用架构**：Netty、Tomcat、gRPC 都这么做——收信与干活分开，防止一次慢业务卡死整个连接层（07 篇打穿）。
- **同步 vs 异步是调用者视角的契约选择**：同 HTTP 请求/回调、消息队列的 push/pull，契约决定线程占用方式。
- **Filter 链 = 中间件管线模式**：Servlet Filter、Koa middleware、gRPC interceptor 同构——横切关注点（超时/重试/追踪）串成链，链序即语义。

> 本层留下的新账：六站全图齐了，但每一站都只是"挂了个名字"——**每站留下的账，就是后续 01~08 篇的深井位置**。
>
> 🔴 **口诀**：IO 线程收信、业务线程干活；Filter 是管线不是特例。

---

<a id="level-8"></a>

## Level 6 全链路总图 + 从 O 看分布式全景

### 6.1 订单 O 的一生总图

```text
        ┌──────────────────────── 一座外卖中央厨房 ────────────────────────┐
        │                                                                  │
 顾客   收银员        分单·派单·改派        打包·信封·骑手          后厨·出餐调度
 (业务)  (Proxy)    (Router/LB/Cluster)  (序列化/协议/Netty)   (Exporter/线程/业务)
   │      │                │                    │                  │
   │  greet(O)            │                    │                  │
   │──────►│ 包装 O         │                    │                  │
   │      │──────►│ Directory.list（门店表，03/04 篇）             │
   │      │       │ Router.route（06 篇）                          │
   │      │       │ LoadBalance.select（05 篇）                    │
   │      │       │ 容错策略（05 篇）                               │
   │      │       │──────►│ 序列化打包（02 篇）                    │
   │      │       │       │ Triple/HTTP/2 信封（01 篇）            │
   │      │       │       │ Netty 骑手送单（07 篇）                  │
   │      │       │       │─────────────────────────────────────────►│ 拆包反序列化
   │      │       │       │                                          │ 出餐调度·业务线程池（07 篇）
   │      │       │       │                                          │ Filter 链（06 篇）
   │      │       │       │                                          │ 业务方法
   │      │       │       │◄──────────────────────────────────────────│ 回执 RpcResult 原路返回
   │◄─────│◄──────│◄──────│◄──────────────────────────────────────────│
    │  得到结果（超时/重试/幂等语义，05/08 篇）                            │
    └─────────────────────────────────────────────────────────────────────┘
```

### 6.1.1 总图的边界：一次调用（送单）≠ 门店开业（export）

6.1 画的是**运行时**视角——O 送单的全程。但它不包含 Provider 启动时"门店开业"那条链路：**Exporter 在总图里只是一个站点名，它其实是两条链路的交点**：

```text
启动期链路（开店，一次性）：
  ServiceConfig.export
    → doExport（组装 URL、依赖检查）
    → Protocol.export(wrapperInvoker)
    → Exporter 监听端口（open server）
    → 注册中心登记（register，03 篇的"总部登记"账）

运行时链路（接单，每次调用）：
  请求到达 Exporter → 收包 → 反序列化 → 业务线程池 → Filter → 业务 Invoker
    → RpcResult 回程（Level 5 五步）
```

> 三问锚定：**开店链路**回答"门店什么时候、以什么顺序对外营业"（03 篇打穿登记，08 篇打穿打烊/unexport）；**接单链路**回答"订单到了之后谁干活"（07 篇打穿线程）。总图里 Provider 侧那一小格（Exporter/线程/业务）是接单链路的压缩版，不是 export 流程本身。

### 6.2 从订单 O 看分布式全景：每一站对应一类分布式通用问题

| 订单 O 的站点 | 机制 | 分布式通用问题 | 通用解法家族 |
| ---- | ---- | ---- | ---- |
| 查门店表 | 服务发现（应用级 vs 接口级） | 成员变更：谁在、地址是什么 | 注册表 + 订阅通知（03 篇） |
| 门店表由谁维护 | Nacos 2.x vs ZK | **CP vs AP**：地址数据的一致性语义 | 一致性与可用性的取舍（04 篇） |
| 选哪家分店 | 负载均衡（随机/轮询/P2C） | 分布式决策：流量往哪放 | 概率决策 / 状态感知决策（05 篇） |
| 这家不接单 | 容错策略（failover 等 5 种） | 客户端侧可用性策略 | 重试 / 快速失败 / 隔离（05 篇） |
| 超时 / 改派 | timeout / 重试次数 | **失败语义**：失败窗口的数学放大 | 超时 × 重试 = 最坏等待时间（08 篇） |
| 总部规则 | 路由/限流/熔断/降级 | 流量治理 | 规则引擎 + 自适应（06 篇） |
| 打烊 / 回访 | 优雅上下线 / 可观测性 | 成员变更与故障定位 | 先拒新单再关店 + 全链路追踪（08 篇） |

> **学习切入点**：Dubbo 的价值不止是"会调远程方法"——它把分布式系统的通用问题（一致性、可用性、失败语义、成员变更、流量治理）全部具象化成一个可运行的框架。学一个 Dubbo 机制，问三个问题：**它解决哪类分布式通用问题？牺牲了什么（Trade-off）？这个解法还能迁移到哪？**（各篇 Transfer 部分按此展开。）

### Transfer——迁移到其他设计问题

- **"每一站 = 一类分布式通用问题"是学习任何分布式框架的元方法**：先问机制解决哪类问题、牺牲了什么（Trade-off）、解法能迁移到哪——学框架，不如先学会提问。
- **横切与竖切互为验证**：本篇的总图是横切（一次调用看全貌）；各篇的竖切总表（T0~T37）是竖切（同一时点跨篇比对）——两张视角互相兜底。

> 本层留下的新账：**哪一站都不是"挂个名字"就完**——01~08 篇逐站打深井；学完每一篇，回来确认本篇欠的账是否还上。
>
> 🔴 **口诀**：六站总图是地图，深井才是矿；每站问三问——解决什么、牺牲什么、迁移到哪。

---

# 全篇因果链总图

```text
Level 1：本地 new / HTTP / RMI / SOAP 的账 → 需要统一收编"远程调用细节"的框架
  ▼
Level 2：官方领域模型 —— Protocol 造 Invoker，Invoker 收 Invocation，Proxy 只做透明化，
         Filter 是拦截点，SPI 是装配器（核心不是注解！）
  ▼
Level 3：O 要打给谁 → 门店表（服务发现）+ 分单/派单/改派（Router/LoadBalance/Cluster）
  ▼
Level 4：O 怎么运 → 打包盒（序列化）+ 配送信封（Triple/HTTP/2）
  ▼
Level 5：O 到后厨 → 反序列化 → 业务线程池 → Filter → 业务方法 → RpcResult 回程
  ▼
Level 6：总图 + 分布式全景（每站 = 一类分布式通用问题的具体答案）
  ▼
01~08 篇：每站打深井（协议/序列化/发现/注册中心/容错/治理/线程与质量/生产）
```

---

<a id="level-9"></a>

## 📝 合书自测

| 自测题 | 必须答出的不变量 |
|---|---|
| 官方领域模型三件套是什么？ | Invocation（会话域）/ Invoker（实体域·核心）/ Protocol（服务域）；官方原文：**Protocol + Invoker + Exporter 即可完成非透明 RPC**。 |
| Proxy 是核心吗？ | 不是。去掉 Proxy 层 RPC 可以 Run，只是不透明——Proxy 是皮，不是核心。 |
| Cluster 是核心概念吗？ | 是外围概念——官方原文：把多个 Invoker **伪装成一个**。 |
| 为什么换协议/注册中心/负载均衡只是改配置？ | Microkernel + Plugin（SPI 装配器）：扩展点即插即换，这是架构级收益。 |
| 旧方案（HTTP/RMI/SOAP）各欠了什么账？ | HTTP=细节散落各写各的；RMI=绑死 Java；SOAP=重慢 XML 膨胀；2.7=URL 爆炸 + 私有协议穿透差。 |
| E00 四个实测事实是什么？ | DubboProxy0/Javassist（代理生成）、ListenerInvokerWrapper（过滤器包装）、CopyOfFilterChainNode（Filter 链）、DubboServerHandler 线程名（业务线程池）。 |
| 每站对应哪类分布式通用问题？ | 门店表=成员变更（注册表+订阅）；CP/AP=一致性语义；派单=分布式决策；改派=失败语义放大；总部规则=流量治理；打烊=优雅停机、回访=可观测。 |
| 能画出六站总图吗？ | 收银台（Proxy）→ 验单台（Filter）→ 门店表（本地候选集）→ 分单/派单/改派（Router/LB/Cluster）→ 打包+信封+骑手（序列化/协议/Netty）→ 后厨出餐（线程池+业务）→ 回执原路返回。 |

---

<a id="level-10"></a>

## ⚠️ 坑与细节

| # | 错误代码/决策 | 错因 | 中央厨房里的后果 | 线上现象 | 修正 |
|---:|---|---|---|---|---|
| 1 | 以为核心是注解/Proxy | 把装配入口当核心 | 看源码先找代理生成逻辑 | 找不到核心链路、读源码迷路 | 从 Invoker 出发读调用链（本篇 L2）。 |
| 2 | 配置写 `triple` | Triple 的 SPI 名想当然 | `No such extension ... by name triple` | E00 实测直接报错 | SPI 注册名是 **tri**（3.3.4）。 |
| 3 | 以为协议名 = 序列化 | 信封与盒子混为一谈 | 以为换 Triple 就自动 Protobuf | 换了协议序列化没换 | 两个独立维度，可任意组合（02 篇）。 |
| 4 | 以为业务只有一种协议端口 | 忽略应用级发现内置机制 | Provider 为 MetadataService 另开 dubbo 协议端口 | 端口清单多了 20880 以为被入侵 | 应用级发现内置行为（E00 实测）。 |
| 5 | 本机代理软件 TUN 网卡不处理 | 忽略 `getLocalHost()` 污染 | 服务 host 变成虚拟网卡 IP | 连接被劫持 reset、互相连不上 | 排查口诀见实验记录 §4（E00 实测）。 |
| 6 | 以为 Filter 是额外特性 | 低估拦截点地位 | 超时/重试/mock/generic/echo 全在 Filter 链上 | 自定义 Filter 放错链序行为失控 | 把 Filter 当"管线"，链序即语义。 |
| 7 | 以为 Provider 业务跑在 IO 线程 | 混淆两类线程 | 业务阻塞以为不影响事件循环 | 误判阻塞影响面 | 业务在 DubboServerHandler 线程池（07 篇展开）。 |
| 8 | 以为同步调用有额外线程 | 误解同步语义 | 找"响应线程"找不到 | 线程 dump 与现实不符 | 同步模式调用线程就是业务线程（E00 实测）。 |

---

<a id="level-11"></a>

## 📚 版本勘误表

| ❌ 常见说法 | ✅ 更准确的说法 |
|---|---|
| 代理类是 `接口名+DubboProxyN` 的通用规律 | 3.3.4 Implementation：Javassist 生成；换 ProxyFactory 扩展点则变（官方允许自定义）。 |
| Triple 的 SPI 名是 `triple` | 3.3.4 Implementation：是 **tri**（配置文件声明，非注解语义；版本间可变）。 |
| 业务服务只有一个端口 | 3.3.4 Implementation：应用级发现内置机制——MetadataService 另开 dubbo 协议随机端口。 |
| `dubbo.network.interface.preferred` 能改 `getLocalHost()` 结果 | 3.3.4 Implementation（实测）：`getLocalAddress0` 先走 `InetAddress.getLocalHost()`，preferred 只在 V6 分支生效——行为细节，版本间可变。 |
| "三件套完成非透明 RPC"是某版本私货 | Specification（官方「代码架构」文档原文，逐字引用，见 research R2）。 |
| "Cluster 伪装一个 Invoker"是作者的总结 | Specification（官方文档原文，逐字引用，见 research R4）。 |

---

<a id="level-12"></a>

## 🎯 生产决策卡（3 张）

## 决策卡 1：新项目协议选什么

- **Decision**：Triple（HTTP/2），跨语言/网关/云原生需求时尤其如此。
- **Reason**：开放协议 + gRPC 兼容；直连性能无优势但差异在场景外（01 篇）。
- **Trade-off**：dubbo 协议直连吞吐略优（官方 benchmark），但那是"迁移期"才值得留的账。
- **Validation**：本机 demo 直连压测（方向性参考）+ 网关场景压测。

## 决策卡 2：调用方 URL 写什么

- **Decision**：不写死 IP；开发环境直连时避免 localhost（会被替换成本机 host）。
- **Reason**：E00 实测：localhost 触发替换逻辑，代理软件网卡环境会连错地址。
- **Trade-off**：写物理网卡 IP 在本机 demo 可行，但线上必须走注册中心。
- **Validation**：`curl --http2-prior-knowledge` 先验证端口可达再连。

## 决策卡 3：学习/排障入口

- **Decision**：遇到 Dubbo 问题，先问"O 卡在哪一站"（六站总图），再进那一站深挖。
- **Reason**：每站的机制边界清晰（发现/路由/容错/协议/线程各司其职）。
- **Trade-off**：需要先建立总图认知（本篇文章）。
- **Validation**：E00 的 TraceFilter 是现成的"逐站打印"模板。

---

<a id="level-13"></a>

## 🌍 跨语言 / 跨运行时视角

| 中央厨房机制 | 分布式通用问题 | 跨语言同构对象 |
|---|---|---|
| 门店表（服务发现） | 成员变更：谁在、地址是什么 | 注册表 + 订阅通知：K8s Endpoint/DNS、Consul、Eureka 健康检查 |
| 台账维护（注册中心） | CP vs AP：地址数据的一致性语义 | Raft/ZAB/gossip 家族：etcd、ZK、Cassandra 的语义选择 |
| 派单（负载均衡） | 分布式决策：流量往哪放 | Nginx upstream、gRPC LB、数据库连接池选连接 |
| 改派（容错策略） | 客户端侧可用性策略 | DNS 多 IP 重试、消息投递换 broker、HTTP 重试 |
| 超时 × 重试 | 失败语义：失败窗口的数学放大 | 预算算式：最坏等待 = 超时 ×（重试+1） |
| 总部规则（治理） | 流量治理 | Envoy xDS 动态下发、网关限流熔断 |
| 打烊 / 回访（上下线/观测） | 成员变更与故障定位 | K8s preStop + readiness、OpenTelemetry 全链路追踪 |

### 跨语言后仍成立的 6 条判断力

1. **Invoker 是骨，Proxy 是皮**——任何 RPC 框架（gRPC、Thrift）都以"调用抽象"为内核，透明层只是包装。
2. **调用前先有候选集**——注册表 + 订阅是分布式系统的通用原料；没有候选集就没有逃生门。
3. **控制面与数据面分离**——注册中心挂了数据面照跑（本地缓存），排查先分"列表问题"和"调用问题"。
4. **失败语义要算账**——超时 × 重试是最坏等待，幂等是重试的前提，参数是预算不是直觉。
5. **规则与执行分离**——规则热更（Router/Configurator）＝不发版改行为；这是所有治理系统的共同架构。
6. **可观测性属于机制的一部分**——先拒新单再关店（优雅停机）、逐站打印（链路追踪）是设计出来的，不是后补的。

---

> 🔴 **电梯版复述**：
>
> **一次 RPC 调用 = 订单 O 走六站：收银台（Proxy 透明化）→ 验单台（Filter 横切）→ 查门店表（本地候选集）→ 分单/派单/改派（Router 资格 / LoadBalance 选择 / Cluster 失败兜底）→ 打包 + 信封 + 骑手（序列化 / 协议 / Netty）→ 后厨出餐（线程池 + 业务方法）→ 回执原路返回。核心是 Invoker 不是注解；每一站对应一类分布式通用问题（成员变更、一致性、分布式决策、失败语义、治理、可观测）。01~06 篇就是在每一站上打深井。**
>
> 下一篇：**《🧭 01｜配送信封：Triple 与 dubbo 协议的对决》**——订单 O 离开收银台后的第一站深井：信封（协议）到底决定什么？为什么 3.x 换成了 Triple（HTTP/2）？直连性能不占优为什么还要迁？
