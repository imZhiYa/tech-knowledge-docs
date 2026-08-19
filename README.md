# 🚀 架构师硬核知识库 · Architect's Hardcore Knowledge Base

**从内核原理解构计算机底层基石，拒绝技术八股。**

_Deconstructing the fundamental building blocks of computing from the kernel up, not just memorizing APIs._

> 闭门代码验证 · 极端场景推演 · 杜绝技术碎片化

[![Stars](https://img.shields.io/github/stars/imZhiYa/tech-knowledge-docs?style=for-the-badge&logo=github&color=yellow)](https://github.com/imZhiYa/tech-knowledge-docs/stargazers)
[![MIT License](https://img.shields.io/badge/license-MIT-blue?style=for-the-badge)](./LICENSE)
![Active Development](https://img.shields.io/badge/status-🛠_Active_Development-yellow?style=for-the-badge)
![Docs](https://img.shields.io/badge/docs-10_篇-success?style=for-the-badge)
![Java](https://img.shields.io/badge/JDK-8%20%7C%2021%20LTS-orange?style=for-the-badge&logo=openjdk)
![Last Commit](https://img.shields.io/github/last-commit/imZhiYa/tech-knowledge-docs?style=for-the-badge)

---

## ✨ 为什么这个仓库不一样？

市面上 90% 的技术博客是 **“知识点罗列 + 源码粘贴”**。这个仓库做的是相反的事：

| 普通笔记 | 本仓库 |
| :--- | :--- |
| 告诉你 `HashMap` 在 JDK8 用了红黑树 | 推导 **为什么是8，不是7或9**？为什么阈值是64？`hash()` 为什么要高低位异或？ |
| 告诉你 `ArrayList` 是数组，`LinkedList` 是链表 | 用 **内存布局 + CPU Cache Line + System.arraycopy native优化** 证明为什么 99% 场景 ArrayList 都更快 |
| 告诉你线程池参数怎么配 | 用 6 个**可运行的反直觉实验**证明 `core=max=1` 也会 `poolSize=0`，`submit` 会吞异常，`newWorkStealingPool` 会导致 JVM 提前退出 |

> **每篇文档都是一次结案，而不是一次搬运。**

### 🎯 核心理念

| 原则 | 说明 |
| --- | --- |
| 🔬 **代码验证 Code Verified** | 所有反直觉结论必须有可运行的最小复现，拒绝“听说” |
| ⚡ **极端推演 Edge Cases** | 考虑溢出、并发竞争、OOM、页表抖动、伪共享等边界 |
| 🧩 **因果链完整 Causality** | 知识点之间有 Why → What → How → Transfer 的推导，而不是孤立记忆 |
| 🏗️ **架构视角 Architect View** | 回答：这个底层设计，对上层 P99、容量治理、故障排查意味着什么 |
| 📌 **诚实边界 Evidence** | 私有字段、性能数字、预览特性一律标注版本，宁可说“须实测” |

### ✍️ 写作方法论：四层递进 + 全文唯一比喻

每篇万字长文都遵守同一套写作契约：

```
Why      → 上一代方案死在哪？不这么做要付出什么代价？
What     → 新结构是什么？（ASCII 图 + 对照表，不堆术语）
How      → 主线角色的慢动作（带编号时序，标出线性化点）
Transfer → 能迁移到哪些别的设计问题？
🔴 口诀   → 全层只需记住的一句话，面试/故障时直接复述
```

- **全文一个比喻**：AQS 是「只有一个工位的工厂」，线程池是「中央厨房」，Collection 是「智能图书馆」
- **全文一条主线**：只跟踪一个角色走完全程（AQS 跟踪线程 B，线程池跟踪订单 T，Collection 跟踪查询请求 Q）
- **每层留一笔账**：上一层解决不了的问题，正是下一层存在的理由

---

## 📂 知识图谱 · Knowledge Map

### 目录结构

```text
docs/
├── 01-cs-foundation/               # 📐 计算机科学基石
│   ├── binary/                     # 二进制底层思维与位运算
│   │   └── 二进制底层思维与位运算.md
│   ├── data-structures/            # 🌳 树形数据结构
│   │   └── 🌳 树形数据结构.md
│   ├── os-memory/                  # 🧠 虚拟内存
│   │   └── 🧠 虚拟内存.md
│   └── networking/                  # 🌐 网络与 I/O 模型
│       └── 🌐 高性能网络编程原理.md
├── 02-jvm/                         # ☕ JVM 运行时
│   └── ☕ JVM 运行时机制深度解析.md
├── 03-concurrency/                 # 🔐 并发与执行引擎
│   ├── 🔐 AQS 核心机制深度解析.md
│   └── 🧵 Java 线程池深度解析.md
├── 04-collections/                 # 🗄️ 集合框架
│   └── 🗄️ Java Collection.md
├── 05-database/                    # 🐬 数据库与存储引擎
│   └── 🐬MySQL-InnoDB-深度解析.md
├── 06-redis/                       # ⚡ Redis 与缓存系统
│   └── ⚡Redis深度解析.md
├── 07-springboot/                  # 🚀 Spring 与 SpringBoot（8 篇系列）
│   └── README.md                   # 系列导航（8 篇清单见子 README）
├── 08-dubbo/                       # 🧭 Dubbo 全链路（9 篇系列）
│   └── README.md                   # 系列导航（9 篇清单见子 README）
├── 09-mq/                          # 📨 MQ 全链路（11 篇系列）
│   └── README.md                   # 系列导航（11 篇清单见子 README）
├── 10-elasticsearch/               # 🔎 ES 全链路（8 篇系列）
│   └── README.md                   # 系列导航（8 篇清单见子 README）
├── 11-ddd/                         # 🏛️ DDD 全链路（7 篇系列）
│   └── README.md                   # 系列导航（7 篇清单见子 README）
└── 12-sharding/                    # 🔀 分库分表全链路（6 篇系列）
    └── README.md                   # 系列导航（6 篇清单见子 README）
```

### 已完成 · Completed

| 模块 | 文档 | 核心硬核点 | 适合谁 |
| --- | --- | --- | --- |
| 📐 **二进制** | [二进制底层思维与位运算](./docs/01-cs-foundation/binary/) | 原码/反码/补码的数学推导、布尔代数、位运算 hack、ALU 电路推导 | 想打穿位运算黑盒的 |
| 🌳 **数据结构** | [树形数据结构](./docs/01-cs-foundation/data-structures/) | 递归/非递归/Morris 三种遍历、BST/AVL/红黑树的旋转不变量推导 | 面试被问红黑树就慌的 |
| 🧠 **虚拟内存** | [虚拟内存](./docs/01-cs-foundation/os-memory/) | 分页/多级页表/TLB/缺页中断/页面置换/MMU 全流程地址翻译 | 被问到“为什么 mmap 快”的 |
| 🌐 **网络编程** | [高性能网络编程原理](./docs/01-cs-foundation/networking/🌐%20高性能网络编程原理.md) **[NEW]** | 从 I/O、BIO、NIO、多路复用、AIO 一路推导到 Reactor 的状态所有权；覆盖 framing、TLS、backlog、半开连接、部分写、背压、ack、drain 与模型选择 | 被“Selector 为什么不等于 Reactor / `write()` 成功算不算完成 / Netty 为什么不能阻塞”问住的 |
| ☕ **JVM** | [JVM 运行时机制](./docs/02-jvm/) | 类加载双亲委派的破与立、对象布局、GC 算法与收集器、JMM Happens-Before | 排查 OOM/GC 问题的 |
| 🔐 **AQS** | [AQS 核心机制](./docs/03-concurrency/) | CLH 隐形队列/前驱接力/dummy head、Condition 双队列、共享传播、与 ObjectMonitor 对照表 | 吃透 ReentrantLock/CountDownLatch 的 |
| 🧵 **线程池** | [Java 线程池](./docs/03-concurrency/) | ctl 位打包、execute 三道门、Worker 与 AQS 的关系、打烊协议、虚拟线程与响应式对比 | 线上线程池告警/堆积的 |
| 🗄️ **集合框架** | [Java Collection](./docs/04-collections/) **[NEW]** | HashMap 寻址/扰动/树化阈值8与64推导、TreeMap 为何选红黑树不选AVL、LinkedHashMap LRU、ConcurrentHashMap JDK7->8 演进与 helpTransfer、Fail-Fast/Fail-Safe 本质 | 被 ConcurrentModificationException / HashMap 死循环问住的 |
| 🐬 **MySQL InnoDB** | [MySQL-InnoDB-深度解析](./docs/05-database/) **[NEW]** | 9层递进：Page/Extent 行格式与行溢出、Buffer Pool 改良LRU、B+ 树容量与最左前缀、事务隔离矩阵、Record/Gap/Next-Key 锁、Read View 版本可见性、Redo/Undo/Binlog 两阶段提交、一条 UPDATE 全链路、EXPLAIN 调优实战；18 坑 + 勘误表 + 5 张生产决策卡 | 被"RR 为什么能防幻读 / UPDATE 加什么锁 / COUNT(*) 为什么慢"问住的 |
| ⚡ **Redis** | [Redis 深度解析](./docs/06-redis/⚡Redis深度解析.md) **[NEW]** | 9层递进：内存与事件循环、type×encoding、过期与淘汰、RDB/AOF、复制与 Sentinel、Cluster 槽位与自治协议、Stream/HLL 场景总装；分布式锁、缓存治理、跨机房与 10 张生产决策卡 | 被“Redis 单线程为什么快 / 主从为什么丢写 / Cluster 为什么 CROSSSLOT / 锁为什么还要 fencing”问住的 |
| 🚀 **SpringBoot** | [SpringBoot 全链路 8 篇系列](./docs/07-springboot/) **[NEW]** | IoC 创建链→框架整合→事件→自动配置→Web 运行时→事务→AOP→生产实践（启动慢排查/AOT/优雅停机）；全部结论由 demo01–demo19 本机实测背书（actuator startup 端点 353 步、JFR 6138 事件、AOT 引擎生成 170 hints）；实验代码存于本地知识库未随仓发布 | 被"事务为什么没回滚 / 启动为什么慢 / kill 为什么丢请求"问住的 |
| 🧭 **Dubbo** | [Dubbo 全链路 9 篇系列](./docs/08-dubbo/) **[NEW]** | 一次 RPC 的一生：Proxy→Filter→服务发现→Router/LB/Cluster→序列化/协议/Netty→线程池→优雅停机与选型总装；全部结论由 E00~E08 九轮本机实测背书（注册三态条目数据、杀 Nacos 派单窗口 ≥68s、ZK 锁释放 13.1s vs Nacos 剔除 1.1s、kill 实例重试边界、250 并发线程池饱和拒绝、triple"送完在途"不可靠）；每篇含坑表/勘误表/决策卡，实验记录后续随仓发布 | 被"No provider / 线程池 EXHAUSTED / mock 不生效 / 停机丢请求"问住的 |
| 📨 **MQ** | [MQ 全链路 11 篇系列](./docs/09-mq/) **[NEW]** | 消息 E 的一生：为什么存在（同步调用三笔账）→ 队列 vs 日志模型 → 存储引擎（顺序写/page cache/稀疏索引/零拷贝）→ 投递语义三道防线 → 顺序重复与幂等 → 消费模型推拉与集群广播 → 再平衡与扩容（分区数 = 并行度上限）→ 事务/延迟/死信 → 副本与高可用（ISR/选举/脑裂）→ 生产选型总控 → 生产问题急诊（DB 有单消息没了 / 千万 Lag / 订单链路乱序重复补偿）；RocketMQ 5.x / Kafka 3.5.x KRaft / RabbitMQ 3.13.x / Pulsar 3.x 四产品机制对照，含故障→文章反向索引与 12 神话核对 | 被"Kafka 为什么能扛堆积 / 重复消费怎么幂等 / 事务消息为什么救不了分布式事务"问住的 |
| 🔎 **ES** | [ES 全链路 8 篇系列](./docs/10-elasticsearch/) **[NEW]** | Doc D 的一生：为什么存在（三铁律 + 12 神话总表）→ 落盘（近实时/更新=删+写/段合并）→ 高可用（副本复制/选举 quorum/脑裂防线）→ 检索聚合（倒排/BM25/近似聚合）→ 失效实战（深分页/429/409/wildcard 代价）→ 工程治理（mapping 锁死/reindex/ILM/snapshot）→ 调优进阶（指标先行/瓶颈分层/容量边界）→ 场景选型（四问框架/何时离开 ES）；ES 8.15.3 本机实测背书，唯一比喻"新书进馆"，配套 dev-lab/es-demo EX-01~06 可复跑 | 被"写入成功为什么搜不到 / 深分页为什么越来越慢 / 副本为什么是写性能税 / 什么时候不该用 ES"问住的 |
| 🏛️ **DDD** | [DDD 全链路 7 篇系列](./docs/11-ddd/) **[NEW]** | 请求 R 的一生：为什么需要领域边界（大泥球三笔账）→ 统一语言与业务边界（六上下文划分）→ 上下文映射与协作（8+1 关系/防腐层/发布语言）→ 实体值对象与聚合（不变量/状态机/集合封装）→ 一次推荐请求如何落到代码（端口适配/降级链）→ 事件 Outbox 与最终一致性（同事务登记/幂等重试）→ 生产决策与选择性 DDD（选型矩阵/决策卡）；业务场景电商"千人千面"推荐，唯一比喻"专业窗口办理业务"，配套 dev-lab/ddd-demo EX-01~07 可复跑 | 被"Service 为什么越改越大 / 为什么不能所有模块都上 DDD / 事件到底可靠不可靠"问住的 |
| 🔀 **Sharding** | [分库分表全链路 6 篇系列](./docs/12-sharding/) **[NEW]** | 订单表 T 的一生：为什么存在（容量墙/性能墙两堵墙 + 三板斧为什么治标）→ 分片键与分片策略（MOD/RANGE/HASH/一致性哈希、三类倾斜机制、ID 可路由）→ 中间件架构与一次 SQL 的生命（解析→路由→改写→执行→归并）→ 分布式查询的代价（广播/深翻页 LIMIT 下推/跨片 JOIN 静默丢数据/全局唯一失效）→ 扩容迁移与生产治理（搬迁比例 gcd 公式、双写+校验+切流+回滚四件套）→ 选型与边界（拆 vs NewSQL vs 不拆）；唯一比喻"图书馆体系"，配套 dev-lab/sharding-demo EX-01~07 可复跑（ShardingSphere-JDBC 5.4.1 实测背书） | 被"取模为什么会有热点 / 深翻页为什么越来越慢 / 跨片 JOIN 为什么数据对不上 / 扩容为什么搬一半"问住的 |

### 🔥 最近更新

| 日期 | 内容 |
| --- | --- |
| **2026-08-19** | **新增《🔀 分库分表全链路》系列 6 篇** — 订单表 T 的一生（图书馆体系）：从"为什么存在"（容量墙/性能墙两堵墙 + 三板斧为什么治标）出发，经分片键与分片策略（MOD/RANGE/HASH/一致性哈希四算法、连续键 vs 块状键三类倾斜、ID 可路由基因法）、中间件架构与一次 SQL 的生命（解析→路由→改写→执行→归并）、分布式查询的代价（广播线性放大/深翻页 LIMIT 下推/跨片 JOIN/全局唯一失效）、扩容迁移与生产治理（搬迁比例 gcd 公式、双写+校验+切流+回滚四件套），收于选型与边界（拆 vs NewSQL vs 不拆）；全部结论由 dev-lab/sharding-demo EX-01~07 实测背书（ShardingSphere-JDBC 5.4.1 + MySQL 8.0.36：雪花低速率键集中 66.7%、同库分片 UNION ALL 合并下推、无绑定 JOIN 静默丢数据 260/1000 行、迁移比例 0.5000、连接按库不按片），verify-sharding-demos.sh 可复跑 |
| **2026-08-18** | **新增《🏛️ DDD 全链路》系列 7 篇** — 请求 R 的一生（专业窗口办理业务）：从"为什么需要领域边界"（大泥球三笔账）出发，经统一语言与业务边界（画像/商品/策略/实验/在线决策/反馈六上下文）、上下文映射与协作（8+1 关系/防腐层/发布语言）、实体值对象与聚合（不变量/状态机/集合封装）、一次推荐请求如何落到代码（端口适配/降级链）、事件 Outbox 与最终一致性（同事务登记/幂等重试），收于生产决策与选择性 DDD（选型矩阵/决策卡）；业务场景电商"千人千面"推荐，配套 dev-lab/ddd-demo EX-01~07（聚合规则 13 断言/ACL 翻译/ArchUnit 边界/全链路降级/Outbox 幂等/契约演进/选型矩阵），verify-ddd-demos.sh 可复跑 |
| **2026-08-17** | **新增《🔎 ES 全链路》系列 8 篇** — Doc D 的一生（新书进馆）：从"为什么存在"（三铁律 + 12 神话总表）出发，经落盘（近实时/更新=删+写/段合并）、高可用（副本复制/选举 quorum/脑裂防线）、检索聚合（倒排/BM25/近似聚合）、失效实战（深分页/429/409/wildcard 代价）、工程治理（mapping 锁死/reindex/ILM/snapshot）、调优进阶（指标先行/瓶颈分层/容量边界），收于场景选型（四问框架/何时离开 ES）；全部结论由 ES 8.15.3 本机实验背书，配套 dev-lab/es-demo EX-01~06（批量摊薄 50 倍/可见性 1s 拍/副本代价/深分页候选放大/filter vs query/cardinality 精度），run-all.sh 可复跑 |
| **2026-08-16** | **新增《📨 MQ 全链路》系列 11 篇** — 消息 E 的一生（城市货运中转站）：从"为什么存在"（同步调用三笔账：耦合/雪崩/无削峰）出发，经队列 vs 日志模型、存储引擎（顺序写 + page cache + 稀疏索引 + 零拷贝）、投递语义三道防线、顺序重复与幂等、消费模型推拉与集群广播、再平衡与扩容、事务/延迟/死信、副本与高可用（ISR/选举/脑裂）、生产选型总控，收于生产问题急诊（DB 有单 Kafka 无消息 / 千万 Lag / 订单链路乱序·重复·补偿）；RocketMQ 5.x / Kafka 3.5.x KRaft / RabbitMQ 3.13.x / Pulsar 3.x 四产品机制对照，附故障→文章反向索引与 12 神话核对 |
| **2026-08-13** | **新增《🧭 Dubbo 全链路》系列 9 篇** — 从"订单 O 的一生"总图出发，逐站打穿协议/序列化/服务发现/注册中心协议/负载均衡容错/治理与泛化/线程模型与质量保障/生产实践；全部结论由 E00~E08 九轮本机实测背书：注册三态条目数据、杀 Nacos 派单窗口 ≥68s、ZK 锁释放 13.1s vs Nacos 剔除 1.1s、kill 实例重试边界、250 并发线程池饱和拒绝（SynchronousQueue 快速失败）、triple 优雅停机"送完在途"不可靠（PING ack 即断连）；含 2.7.x hessian2 ThreadLocal OOM 源码 tag 逐版本核验与修复路径 |
| **2026-08-09** | **新增《🚀 SpringBoot 全链路》系列 8 篇** — 从 IoC 创建链（ConfigurationClassPostProcessor→BeanDefinition→实例化→销毁）到事件/自动配置/Web 运行时/事务/AOP/生产实践（慢发布与 AOT 实测、优雅停机）；每个机制结论均由 demo01–demo19 可运行实验背书：事务失效双案例、AOT 引擎端到端（生成 5 源码 + CGLIB 代理字节码 + 170 RuntimeHints）、启动慢排查四方法（actuator startup 端点 353 步 / JFR 6138 事件）、immediate vs graceful 停机对照 |
| **2026-08-03** | **新增《🌐 高性能网络编程原理》** — 从 I/O 的等待分类出发，经 BIO 陪等、NIO 空转、多路复用/AIO 的事件语义，推导 Reactor 的状态所有权；覆盖背压、业务 ack、drain 与 4 张生产决策卡。 |
| **2026-08-03** | **新增《⚡ Redis 深度解析》** — 9层递进(L1-L9)，从事件循环、内存编码一路推导到过期淘汰、持久化、高可用、Cluster、Stream/HLL 与生产架构评审；含分布式锁、缓存治理、勘误表、合书自测与 10 张生产决策卡 |
| **2026-08-01** | **新增《🐬 MySQL InnoDB 存储引擎深度解析》** — 9层递进(L1-L9)，Level 1 到 Level 9 覆盖页存储/Buffer Pool/B+树/事务/锁/MVCC/日志三剑客/全链路/调优，18 坑 + 勘误表 + 5 张生产决策卡 + 合书自测，全文唯一比喻「24 小时无人图书馆」 |
| **2026-07-29** | **新增《🗄️ Java Collection 框架深度解析》** — 8层递进(L1-L8 + 5.5/7.5/7.6扩展)，全家族覆盖 ArrayList/LinkedList/HashMap/TreeMap/LinkedHashMap/ConcurrentHashMap + EnumSet/IdentityHashMap/CopyOnWrite 速查表，Fail-Fast 全链路拆解，10+ 张生产选型决策卡 |
| 2026-07-28 | 新增《🧵 Java 线程池深度解析》—— 7 层递进 + 13 坑 + 8 张生产决策卡 + 6 个可运行反直觉实验 |
| 2026-07-24 | 新增《🔐 AQS 核心机制深度解析》—— 从 CLH 接力到生产架构决策，5张决策卡 |
| 2026-07-12 | 新增《☕ JVM 运行时机制深度解析》—— 类加载到 GC 全链路 |

---

## 🗺️ 阅读路径 · Reading Paths

**文档很厚，不必线性读。按你的目标切入：**

#### 🎯 我要准备面试（60分钟速通版）

```text
📐 二进制 → 🌳 数据结构 → 🧠 虚拟内存 → 🌐 高性能网络编程 → ☕ JVM → 🔐 AQS → 🧵 线程池 → 🗄️ Collection → 🐬 MySQL InnoDB → ⚡ Redis
```

每篇末尾的 🔴 **口诀** 串起来就是电梯版复述稿；每篇的「合书自测」是面试官视角的灵魂拷问。

**面试高频必看：**
- `HashMap` 为什么线程不安全？JDK8 为什么要树化？ → [🗄️ Collection · Level 4]
- `ArrayList` vs `LinkedList` 真的只是数组和链表的区别吗？ → [🗄️ Collection · Level 2]
- `ConcurrentHashMap` JDK7 和 JDK8 实现有什么区别？ → [🗄️ Collection · Level 6]
- 线程池用 `submit` 还是 `execute`？ → [🧵 线程池 · Lab 3]
- BIO、NIO、AIO 是性能等级还是不同的等待/通知模型？→ [🌐 网络编程 · 全局认知地图](./docs/01-cs-foundation/networking/🌐%20高性能网络编程原理.md)
- `Selector` / `epoll` 通知可读后，为什么不能直接处理一条完整请求？→ [🌐 网络编程 · NIO 与多路复用](./docs/01-cs-foundation/networking/🌐%20高性能网络编程原理.md)
- 本地 `write()` 返回成功，为什么还不能标记业务完成？→ [🌐 网络编程 · Reactor 状态所有权](./docs/01-cs-foundation/networking/🌐%20高性能网络编程原理.md)
- InnoDB 的 RR 隔离级别真的完全防幻读吗？ → [🐬 InnoDB · Level 5/坑 2]
- `UPDATE WHERE 无索引` 为什么会锁全表？ → [🐬 InnoDB · 坑 1]
- `COUNT(*)` 在 InnoDB 里为什么慢？ → [🐬 InnoDB · 坑 4]
- Redis 为什么单线程却能支撑高并发？→ [⚡ Redis · Level 2：事件循环与线性化点](./docs/06-redis/⚡Redis深度解析.md)
- Redis 的 `TTL` 为什么不是业务定时器？大 key / 热 key 怎么治理？→ [⚡ Redis · Level 4：过期、淘汰与生产治理](./docs/06-redis/⚡Redis深度解析.md)
- `SET NX PX + Lua` 为什么仍需要 fencing token？→ [⚡ Redis · 分布式锁与生产决策卡](./docs/06-redis/⚡Redis深度解析.md)
- 为什么服务发现倾向 AP，而分布式锁/配置必须 CP？→ [🧭 Dubbo · 04 注册中心协议篇](./docs/08-dubbo/dubbo-04-registry-protocol.md)
- failover 重试为什么兜不住"在途被切断"？业务异常为什么不重试？→ [🧭 Dubbo · 05 负载均衡与集群容错篇](./docs/08-dubbo/dubbo-05-loadbalance-cluster.md)
- 一次 RPC 调用从 @DubboReference 到业务方法返回经过哪些站？→ [🧭 Dubbo · 00 骨干篇](./docs/08-dubbo/dubbo-00-backbone.md)
- 为什么说"send 返回 ≠ 已持久化"？重复消费为什么靠幂等而非"不重发"？→ [📨 MQ · 03 投递语义篇](./docs/09-mq/mq-03-投递语义与可靠性.md)
- 消费堆积为什么"加机器"不一定解决？→ [📨 MQ · 06 再平衡与扩容篇](./docs/09-mq/mq-06-再平衡与扩容.md)
- 写入成功为什么搜不到？更新为什么是"删+写"？→ [🔎 ES · 01 存储内核篇](./docs/10-elasticsearch/es-01-存储内核与落盘.md)
- 深分页为什么越来越慢？聚合为什么是"近似"？→ [🔎 ES · 03 检索与聚合篇](./docs/10-elasticsearch/es-03-检索与聚合.md)
- 为什么"分库分表 ≠ 水平拆分"？三板斧为什么治标不治本？→ [🔀 Sharding · 00 单库之墙篇](./docs/12-sharding/shard-00-单库之墙与拆分谱系.md)
- 取模分片为什么会有热点？雪花 ID 为什么更集中？→ [🔀 Sharding · 01 分片键与分片策略篇](./docs/12-sharding/shard-01-分片键与分片策略.md)
- 深翻页为什么越来越慢？游标分页为什么恒定？→ [🔀 Sharding · 03 分布式查询的代价篇](./docs/12-sharding/shard-03-分布式查询的代价.md)
- 为什么 8 片扩 16 片要搬一半数据？→ [🔀 Sharding · 04 扩容迁移与生产治理篇](./docs/12-sharding/shard-04-扩容迁移与生产治理.md)

#### 🔧 我在排查线上问题

| 线上症状 | 直接去看 |
| --- | --- |
| `ConcurrentModificationException` / 遍历时删除报错 | [🗄️ Collection · Fail-Fast vs Fail-Safe](./docs/04-collections/) |
| `HashMap` 扩容死链 / CPU 100% | [🗄️ Collection · Level 4 扩容时序 + 坑位](./docs/04-collections/) |
| 线程池队列堆积 / 大量 BLOCKED / 任务被丢弃 | [🧵 线程池 · 线上排查工具箱](./docs/03-concurrency/) |
| 锁竞争、P99 突刺、死锁 | [🔐 AQS · 生产决策卡 & P99 决策树](./docs/03-concurrency/) |
| GC 频繁 / 内存溢出 / Metaspace 飙升 | [☕ JVM 运行时](./docs/02-jvm/) |
| 缺页、SWAP、内存映射异常 | [🧠 虚拟内存](./docs/01-cs-foundation/os-memory/) |
| `connect timeout` / `TcpExtListenOverflows` / 半开连接 / FD 持续增长 | [🌐 网络编程 · BIO 与建连生命周期](./docs/01-cs-foundation/networking/🌐%20高性能网络编程原理.md) |
| event loop CPU 高 / `OP_WRITE` 空转 / 半包粘包 / Selector 行为异常 | [🌐 网络编程 · NIO 与多路复用](./docs/01-cs-foundation/networking/🌐%20高性能网络编程原理.md) |
| outbound queue 堆积 / 慢客户端 / drain 重试 / ack 超时 | [🌐 网络编程 · 背压、ack、drain](./docs/01-cs-foundation/networking/🌐%20高性能网络编程原理.md) |
| 慢 SQL / `Waiting for table metadata lock` / 死锁 / 磁盘空间不释放 | [🐬 InnoDB · 坑与调优](./docs/05-database/) |
| Redis P99 突刺 / `blocked_clients` / 大 key 阻塞 | [⚡ Redis · 事件循环、阻塞半径与大 key 治理](./docs/06-redis/⚡Redis深度解析.md) |
| 缓存命中率断崖 / DB 回源暴增 / 雪崩击穿 | [⚡ Redis · 缓存三兄弟与生产闭环](./docs/06-redis/⚡Redis深度解析.md) |
| `MOVED` / `ASK` / `CROSSSLOT` / 主从切换异常 | [⚡ Redis · Cluster、复制与高可用](./docs/06-redis/⚡Redis深度解析.md) |
| `@Transactional` 没生效 / 自调用 / 事务没回滚 | [🚀 SpringBoot · 05 事务篇：急诊检查单 7 项](./docs/07-springboot/springboot-05-transaction-and-data-layer.md) |
| 启动慢 / 发布回滚 10 分钟 / 想上 AOT | [🚀 SpringBoot · 07 生产篇 7.2/7.4：指纹先行 + AOT 实测](./docs/07-springboot/springboot-07-production-practice.md) |
| 停机丢请求 / SIGTERM 后流量中断 | [🚀 SpringBoot · 07 生产篇 Level 8：先拒新、再排空、后关资源](./docs/07-springboot/springboot-07-production-practice.md) |
| 代理失效 / 切面没生效 / CGLIB vs JDK | [🚀 SpringBoot · 06 AOP 篇：代理机制与决策卡](./docs/07-springboot/springboot-06-aop.md) |
| `No provider available` / 空地址 / 注册后仍调不到 | [🧭 Dubbo · 03 注册中心篇](./docs/08-dubbo/dubbo-03-registry.md) |
| 线程池 EXHAUSTED / Consumer 收到 CANCELLED | [🧭 Dubbo · 07 线程模型篇：饱和行为与故障读图](./docs/08-dubbo/dubbo-07-threadmodel-quality.md) |
| mock 不生效 / 优雅停机丢请求 / 停机瞬间 CANCELLED | [🧭 Dubbo · 08 生产实践篇：打烊与回访](./docs/08-dubbo/dubbo-08-production.md) |
| 2.7.x 大响应 OOM / ThreadLocal 泄漏 | [🧭 Dubbo · 02 序列化篇：ThreadLocal 泄漏机制与修复路径](./docs/08-dubbo/dubbo-02-serialization.md) |
| 消息丢失 / 发送成功却不落盘 / DB 有单、MQ 无消息 | [📨 MQ · 03 投递语义与可靠性篇](./docs/09-mq/mq-03-投递语义与可靠性.md) |
| 重复消费 / 乱序 / 幂等失效 | [📨 MQ · 04 顺序重复与幂等篇](./docs/09-mq/mq-04-顺序重复与幂等.md) |
| 消费堆积 / Lag 只增不减 / 再平衡风暴 | [📨 MQ · 06 再平衡与扩容篇](./docs/09-mq/mq-06-再平衡与扩容.md) |
| 事务消息失效 / 延迟不准 / 死信堆积 | [📨 MQ · 07 事务延迟死信篇](./docs/09-mq/mq-07-事务延迟死信.md) |
| ES 写入后搜不到 / 明明有数据却查不到 / 段越来越多 | [🔎 ES · 01 存储内核篇](./docs/10-elasticsearch/es-01-存储内核与落盘.md) |
| 深分页超时 / 10000 窗口拒绝 / 聚合结果对不上 | [🔎 ES · 03 检索与聚合篇](./docs/10-elasticsearch/es-03-检索与聚合.md) |
| bulk 429 拒绝 / 并发更新 409 冲突 / wildcard 慢查询 | [🔎 ES · 04 失效实战篇](./docs/10-elasticsearch/es-04-失效实战与排查.md) |
| 节点失联 / 脑裂 / 副本 unassigned / 磁盘水位告警 | [🔎 ES · 02 高可用篇](./docs/10-elasticsearch/es-02-高可用与故障恢复.md) |
| 取模分片写入全打一片 / 数据分布不均 | [🔀 Sharding · 01 分片键与分片策略篇：三类倾斜机制](./docs/12-sharding/shard-01-分片键与分片策略.md) |
| 无路由键查询越来越慢 / SQL 变慢排查 | [🔀 Sharding · 02 中间件架构篇：一次 SQL 的生命](./docs/12-sharding/shard-02-中间件架构与一次SQL的生命.md) |
| 深翻页越来越慢 / 跨片 JOIN 数据对不上 / 唯一约束失效 | [🔀 Sharding · 03 分布式查询的代价篇](./docs/12-sharding/shard-03-分布式查询的代价.md) |
| 扩容搬迁 / 迁移期数据不一致 / 双写怎么切 | [🔀 Sharding · 04 扩容迁移与生产治理篇](./docs/12-sharding/shard-04-扩容迁移与生产治理.md) |

#### 🏗️ 我在做架构评审

直接跳到每篇末尾的 **生产决策卡** 与 **设计记录清单**：

- 🗄️ **Collection**：10+ 张决策卡（List 选型 / Map 选型 / LRU 实现 / 并发容器选型 / 遍历与删除 / Queue 替代 Stack / Enum 优化 / 容量治理 / 避坑清单）
- 🧵 **线程池**：8 张决策卡（核心链路 / 埋点 / 批处理 / P99 排障 / @Async 避坑 / 虚拟线程迁移 / 动态线程池 / 舱壁隔离）
- 🌐 **网络编程**：4 张决策卡（长连接事件循环 / 慢下游隔离 / 慢客户端与大响应 / 低并发或虚拟线程模型）；覆盖连接准入、背压、ack、drain 验收指标
- 🔐 **AQS**：5 张决策卡（分片锁 / Semaphore 限流 / 生产者消费者 / P99 决策树 / Virtual Threads 迁移）
- 🐬 **InnoDB**：5 张决策卡（隔离级别选型 / Buffer Pool 容量规划 / 索引设计 / 日志与持久性配置 / 慢查询排查 SOP）
- ⚡ **Redis**：10 张决策卡（Cache-Aside / 淘汰与容量 / RDB-AOF / Sentinel / Cluster / 分布式锁 / Lua 与 Function / Cluster 配置 / 跨机房容灾 / RESP3 Tracking）
- 🚀 **SpringBoot**：每篇决策卡（框架整合选型 / 事件异步化 / 自动装配自研 / 探针与停机 / 事务传播与回滚规则 / AOP 选型 / 启动优化 / 停机模式选型）
- 🧭 **Dubbo**：9 篇每篇决策卡（协议与序列化 / 注册中心选型与 register-mode 迁移 / 语义选型 CP-AP / 负载均衡与重试预算 / 治理与泛化边界 / 线程预算表 / 优雅停机与验收）
- 📨 **MQ**：11 篇每篇决策卡（MQ 选型矩阵与模型分野 / 存储与刷盘窗口 / 投递语义与确认级别 / 幂等与顺序边界 / 消费模型与背压 / 分区扩容规划 / 事务与死信治理 / 副本 ISR 与脑裂防线 / 生产验收指标 / 故障诊断 SOP）
- 🔎 **ES**：8 篇每篇决策卡（场景四问 / 索引与 mapping 规划 / 副本容错预算 / 检索选型与深分页 / 失效排查 SOP / mapping 治理与 ILM / 调优验收指标 / 选型终局）
- 🔀 **Sharding**：6 篇每篇决策卡（拆不拆四问 / 分片键四合一选型 / 中间件形态 JDBC-Proxy-Cluster / 跨片查询治理 / 扩容四件套与验收 / 四维选型矩阵）

每张卡都包含「**不能做的错误决策**」与「**验收指标/埋点**」两栏，可直接贴到 RFC 里。

---

## 🧪 代码验证 · Verified by Code

本库坚持「**结论必须可复现**」。

- **线程池** 6 大反直觉实验：幽灵 Worker(`poolSize=0`) / 无界队列废掉 `max` / `submit` 吞异常 / `workStealingPool` 守护线程导致任务丢失 / 五种提交方式阻塞点对比 / `InheritableThreadLocal` 在池化场景天然失效
- **AQS** 实验：Condition `await/signal` 时 `firstWaiter` 到 `lastWaiter` 的节点迁移、共享锁传播的 `setHeadAndPropagate` 边界
- **Collection** 实验：HashMap 扰动函数碰撞率对比、树化阈值8在不同负载因子下的实测、Fail-Fast 的 `modCount` 竞态窗口、ConcurrentHashMap 扩容 `helpTransfer` 加速效果
- **Redis** 验证边界：事件循环线性化点、编码阈值、过期/淘汰、RDB/AOF、复制切换、Cluster 重定向、Stream PEL 与 HLL 误差；性能数字始终绑定版本、硬件与压测方法
- **Dubbo** 九轮实测（E00~E08，Dubbo 3.3.4 + Nacos 2.4.3 + ZK 3.9.5）：一次调用全链路打印、协议/序列化方向性压测、注册三态条目数据、杀 Nacos 派单窗口 ≥68s、ZK 锁释放 13.1s vs Nacos 剔除 1.1s、kill 实例重试边界、泛化调用、250 并发线程池饱和拒绝、triple 优雅停机"送完在途"不可靠；2.7.x hessian2 ThreadLocal 泄漏经源码 tag 逐版本核验（实验记录后续随仓发布）
- **MQ** 六轮机制验证已跑（EX-01~06，macOS arm64 + colima 跨 VM，Kafka 3.5.2 / RabbitMQ 3.13 / Redis 7 / MySQL 8.4，教学量级 FAST 档）：单副本 acks=1 vs all 确认差异 6.1%（ISR=1 退化机制）、批量拉取 100 vs 10 次 poll 往返、poll 间隔超时确定性被踢 + rebalance 循环、Redis 淘汰后幂等方案漏 100% vs DB 唯一索引 0（正确性必须落 DB）、Lag 推算偏差 10.9%、RabbitMQ CQv1 vs v2 堆积形态差异、乱序注入状态机拦截 + 无上限重试活锁；EX-07 参数敏感性全矩阵未跑（需多 broker 拓扑）；实验代码与结果表见 [dev-lab/mq-demo](https://github.com/imZhiYa/dev-lab/tree/main/mq-demo)（run-all.sh ~10min 可复跑）
- **ES** 六轮机制验证已跑（EX-01~06，macOS arm64 + colima 跨 VM，ES 8.15.3 × 3 节点各 512m 堆，教学量级 FAST 档）：批量摊薄单条 363 → bulk 5000 达 18,038 docs/s（约 50 倍）、可见性延迟 P50=989ms（1s 拍）且 refresh_interval=-1 不可见持续、副本代价单 VM 仅 -3.7%（loopback 拓扑边界：跨机网络才是副本税主战场）、深分页候选 900 倍只换 2 倍延迟 + 10000 窗口 400 拒绝实录、filter vs query 首次 8 倍差 + query cache 段级边界、cardinality threshold 100→3.73% 误差 vs 40000→0%；实验代码与结果表见 [dev-lab/es-demo](https://github.com/imZhiYa/dev-lab/tree/main/es-demo)（run-all.sh ~10min 可复跑）
- **Sharding** 八轮验证已跑（SmokeApp + EX-01~07，macOS arm64 + colima 跨 VM，ShardingSphere-JDBC 5.4.1 + MySQL 8.0.36 单容器 4 database，教学量级 FAST 档）：连续键 1 万行 8 片各 1250 均匀 vs 雪花低速率键 1.5 万行集中片 0/1（最大片 66.7%）、带键路由 1 条物理 SQL vs 无键广播（5.4.1 实测同库两片合并 UNION ALL 下推共 4 条）、AVG 改写 SUM+COUNT、IN 按数据源拆分（同片 1 条/跨库 2 条）、offset=9000 每片下推 LIMIT 0, 9020（41ms）vs keyset LIMIT 20（4ms）、**无绑定 JOIN 静默丢数据**（期望 1000 行只回 260 行，比报错更危险）、8→16 搬迁比例 SQL 实测 0.5000 与 gcd 公式一致、广播 8 片后仅 4 连接（连接按库不按片，上限 = 4 库 × 池大小 4 = 16）；实验代码与结果表见 [dev-lab/sharding-demo](https://github.com/imZhiYa/dev-lab/tree/main/sharding-demo)（run-all.sh ~5min 可复跑）

> 文档中引用的所有实验输出 **均为真实运行结果**，非手写示意。以 OpenJDK 21 LTS `jdk21u` 为基线。

---

## 📐 版本与证据边界 · Evidence Policy

技术文档最大的风险不是写错，而是 **让读者无法判断哪里可能过时**。本库统一标准：

- **标注基线版本**：如「以 OpenJDK 21 LTS 为准，源码取自 `jdk21u`，与 JDK 8 做差异对比」
- **标注核实日期**：引用预览特性时写明「最后核实于 YYYY-MM」，便于判断是否滞后
- **区分语义变化与代码重构**：不把「改了写法」说成「改了行为」
- **私有实现不背字段**：`ObjectMonitor._cxq`、AQS `Node.status`、`HashMap.TreeNode` 旋转细节等属实现细节，只讲不变量
- **性能数字必须实测**：文中量级仅供教学参考，一律注明「以你的部署压测结果定案」
- **公开自我修正**：发现此前结论有误时保留勘误表，而非悄悄改掉

---

## 🛠️ 配套工具 · Companion Tools

| 项目 | 说明 | 链接 |
| --- | --- | --- |
| 🧪 **cs-visual-tools** | CS 可视化交互工具（树结构动画 / 位运算 / 虚拟内存页表动画） | [→ 查看](https://github.com/imZhiYa/cs-visual-tools) |
| 🧬 **dev-lab** | 底层技术沙盒与基准测试代码（Binary & Benchmark Lab） | [→ 查看](https://github.com/imZhiYa/dev-lab) |

---

## 🔍 如何使用 · Getting Started

```bash
git clone https://github.com/imZhiYa/tech-knowledge-docs.git
cd tech-knowledge-docs
# 直接用 Typora / Obsidian / VS Code 打开 docs/ 阅读，ASCII 图在等宽字体下最佳
```

**阅读建议**

1.  文档使用 **显式 ASCII 锚点**，GitHub / VS Code / Typora 预览均可稳定跳转
2.  篇幅较长的文档开头都有「带着问题来的走这条快速路径」索引表，5 分钟定位
3.  建议配合官方源码阅读：看源码前先 `java -version`，再打开 **同版本** 源码，别用 JDK8 的理解去套 JDK21

**每篇文档包含 8 个标准模块**

| 模块 | 作用 |
| --- | --- |
| 📍 能力地图 | 分层列出「要打穿的认知墙」与「通关标准」 |
| 🏭 唯一比喻地图 | 一张 ASCII 图 + 比喻与技术概念的对照表 |
| 🟢 Level 1..N | Why / What / How / Transfer 四段递进，唯一主线角色慢动作 |
| 🧪 合书自测 | 一页时序图 + 必须答出的不变量，面试自测用 |
| ⚠️ 坑与细节 | 错误代码 → 错因 → 线上后果 → 修正 |
| 📊 竖切总表 | 横轴时间、纵轴维度的全景对照，T0-T8 |
| 📚 版本勘误 | ❌ 常见说法 vs ✅ 更准确的说法 |
| 🏆 生产决策卡 | 场景 → 决策 → 错误做法 → 验收指标 |

---

## 🤝 参与贡献 · Contributing

欢迎任何形式的贡献，提交 [Issue](https://github.com/imZhiYa/tech-knowledge-docs/issues) 或 PR 均可。

**尤其欢迎这几类**

- 🐛 **事实性纠错**：指出版本判断、源码引用、结论推导中的错误（请附源码链接或可复现步骤）
- 🧪 **补充验证代码**：为某个结论提供可运行的最小复现，放在 `dev-lab`
- 📝 **补充生产案例**：真实踩坑经历比理论更有价值，匿名也可

**提交规范**

```bash
docs(scope): 新增《emoji 标题》     # 新增文档
docs(scope): 修正 xxx              # 内容修正
chore: xxx                         # 构建 / 配置
```

---

## 📄 许可证 · License

本项目基于 [MIT License](./LICENSE) 开源。文档内容欢迎转载，请注明出处。

---

**从二进制到集合框架再到数据库存储引擎，构建不可动摇的计算机知识体系**

如果这些文档帮到了你，欢迎点一个 ⭐ **Star**，这会让更多人看到这份硬核知识库。

[@imZhiYa](https://github.com/imZhiYa) · 2026
