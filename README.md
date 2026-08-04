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
└── 06-redis/                       # ⚡ Redis 与缓存系统 (NEW)
    └── ⚡Redis深度解析.md
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

### 🔥 最近更新

| 日期 | 内容 |
| --- | --- |
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

#### 🏗️ 我在做架构评审

直接跳到每篇末尾的 **生产决策卡** 与 **设计记录清单**：

- 🗄️ **Collection**：10+ 张决策卡（List 选型 / Map 选型 / LRU 实现 / 并发容器选型 / 遍历与删除 / Queue 替代 Stack / Enum 优化 / 容量治理 / 避坑清单）
- 🧵 **线程池**：8 张决策卡（核心链路 / 埋点 / 批处理 / P99 排障 / @Async 避坑 / 虚拟线程迁移 / 动态线程池 / 舱壁隔离）
- 🌐 **网络编程**：4 张决策卡（长连接事件循环 / 慢下游隔离 / 慢客户端与大响应 / 低并发或虚拟线程模型）；覆盖连接准入、背压、ack、drain 验收指标
- 🔐 **AQS**：5 张决策卡（分片锁 / Semaphore 限流 / 生产者消费者 / P99 决策树 / Virtual Threads 迁移）
- 🐬 **InnoDB**：5 张决策卡（隔离级别选型 / Buffer Pool 容量规划 / 索引设计 / 日志与持久性配置 / 慢查询排查 SOP）
- ⚡ **Redis**：10 张决策卡（Cache-Aside / 淘汰与容量 / RDB-AOF / Sentinel / Cluster / 分布式锁 / Lua 与 Function / Cluster 配置 / 跨机房容灾 / RESP3 Tracking）

每张卡都包含「**不能做的错误决策**」与「**验收指标/埋点**」两栏，可直接贴到 RFC 里。

---

## 🧪 代码验证 · Verified by Code

本库坚持「**结论必须可复现**」。

- **线程池** 6 大反直觉实验：幽灵 Worker(`poolSize=0`) / 无界队列废掉 `max` / `submit` 吞异常 / `workStealingPool` 守护线程导致任务丢失 / 五种提交方式阻塞点对比 / `InheritableThreadLocal` 在池化场景天然失效
- **AQS** 实验：Condition `await/signal` 时 `firstWaiter` 到 `lastWaiter` 的节点迁移、共享锁传播的 `setHeadAndPropagate` 边界
- **Collection** 实验：HashMap 扰动函数碰撞率对比、树化阈值8在不同负载因子下的实测、Fail-Fast 的 `modCount` 竞态窗口、ConcurrentHashMap 扩容 `helpTransfer` 加速效果
- **Redis** 验证边界：事件循环线性化点、编码阈值、过期/淘汰、RDB/AOF、复制切换、Cluster 重定向、Stream PEL 与 HLL 误差；性能数字始终绑定版本、硬件与压测方法

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
