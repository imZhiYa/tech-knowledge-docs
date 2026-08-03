# 🧵 Java 线程池深度解析

> **写在最前面的承诺**
>
> 这不是一份"七大参数 + 四种拒绝策略"的线程池八股清单。它只讲一件事：**一座中央厨房，如何在订单洪峰、厨师罢工、灶台爆炸、老板宣布打烊的同时，既不丢单、不重复出餐、不把厨房挤爆，还能在虚拟线程和响应式框架进场后仍然算得清自己的容量。**
>
> 全文只跟踪一份订单——**订单 T** 的一次 `executor.execute(T)`。每一层都按 **Why → What → How → Transfer** 推进：上一代方案死在哪，为什么必须长出新结构，新结构怎样在慢动作里工作，又留下什么新账。
>
> 说明一点：从 Level 6 开始会出现另一批角色（周期任务 P、分治任务 F、响应式流元素 E）。他们各自演示定时池、ForkJoinPool 和 Reactor 的场景，**和主线的 T 不是同一条因果链**，只是借用同一座厨房的舞台。看到陌生字母不必往主线上套，文中会明确提示。

> 🏷️ `ThreadPoolExecutor` `ctl` `Worker` `AQS` `getTask` `RejectedExecutionHandler` `ScheduledThreadPoolExecutor` `ForkJoinPool` `work-stealing` `CompletableFuture` `Virtual Threads` `Reactor Schedulers` `背压`

> ⚠️ **版本与证据边界**
>
> 🗓️ **最后核实日期：2026-07-28。** 本文引用了大量**仍在预览、随时可能变更**的 API（结构化并发已到 JEP 533 第七次预览、Spring Boot 虚拟线程调度器行为、Reactor 默认值）。**如果你在此日期之后阅读并发现不一致，请优先相信你的 JDK/框架版本文档——那说明本文滞后了，而不是你错了。** 这是"没有一个数字可以直接抄"原则的自然延伸：**没有一个版本判断可以脱离核实日期。**
>
> - Java 公开 API 语义以 **OpenJDK 21 LTS** 为基线，源码引用取自 `jdk21u`。**JDK 8 的代码会作为"协议骨架"对照使用**，用于展示哪些是语义变化、哪些只是代码组织重构——这两者绝不混称。
> - 一个必须先说清的诚实结论：**`ThreadPoolExecutor` 从 JDK 8 到 JDK 21，核心执行语义几乎没有变。** `execute` 的三步决策、`ctl` 的位打包、`Worker` 继承 AQS、`getTask` 的回收循环，逐行比对后结构一致。真正变化的是 `addWorker`/`getTask` 中运行状态判断的**写法**（JDK 8 用 `rs >= SHUTDOWN` 配合嵌套否定，后改为 `runStateAtLeast(c, SHUTDOWN)`），以及 `CAPACITY` 更名为 `COUNT_MASK`。**这是可读性重构，不是行为变更，且它落在 JDK 10 —— 出处是 [JDK-8186056](https://bugs.openjdk.org/browse/JDK-8186056)（commit `c3664b7f38a0`，Doug Lea 提交，2017-10-03），不是 JDK 21**（完整证据链见 Level 2）。任何声称"JDK 21 线程池被重写了"的说法都需要拿出 diff——本文也为自己的这个说法给出了 JBS 编号、commit 哈希与可复现命令。
>   - ⚠️ 两处**已被本文自己推翻**的早期说法，一并列在这里以免读者只看到旧版：① "重构发生在 JDK 9~10 之间"——**精确答案是 JDK 10**（JDK 9 仍用 `CAPACITY`）；② "JDK 11 与 JDK 21 的 `addWorker` 逐字相同"——**不成立**，差一行 `t.start()` → `container.start(t)`（JDK 19 的 JEP 444 引入，不改变执行语义）。
> - 真正的时代变化不在 `ThreadPoolExecutor` 内部，而在它**外部**：虚拟线程（JEP 444，JDK 21 正式）让"线程即稀缺资源"这一前提失效，从而动摇了池化本身的理由。这是本文 Level 7 的核心。
> - **关于 pinning 的重要版本修正**：JDK 21 中虚拟线程在 `synchronized` 块内阻塞会 pin 住 carrier；**JEP 491（JDK 24 正式）已将监视器归属从 carrier 迁移到虚拟线程本身，消除了这类 pinning**。本文会同时给出"JDK 21 该怎么办"和"JDK 24+ 之后哪些建议已过期"，请按你的部署版本取用。
> - `ForkJoinPool` 的内部实现（工作队列布局、扫描算法、补偿线程策略）在 JDK 8→21 之间有实质演进，且属于私有实现。本文只讲**不变的设计意图**，不背字段。
> - Reactor 的 `Schedulers` 默认值取自 Project Reactor 官方 Javadoc，但**默认值可被系统属性覆盖且随版本演进**，生产必须以你依赖的具体版本为准。
> - 性能量级、线程数公式、监控命令均为教学/诊断参考。**没有一个数字可以直接抄进生产**，请以你的 JDK、硬件、cgroup 配额与压测结果定案。

---

## 🗂️ 目录

> 📌 以下目录使用**显式 ASCII 锚点**，不依赖不同 Markdown 渲染器对中文、emoji、全角符号的 slug 规则；GitHub、VS Code 与常见文档预览均可稳定跳转。

- [📍 使用说明：七级线程池能力地图](#roadmap)
- [🏭 全书唯一厨房地图](#kitchen-map)
- [🟢 Level 1：先回答为什么不能来一单开一个厨师](#level-1)
- [🟢 Level 2：ctl —— 为什么状态与计数必须挤进一个 int](#level-2)
- [🟢 Level 3：execute 的三道门 —— 以及那次能救命的复查](#level-3)
- [🟢 Level 4：Worker 为什么继承 AQS —— 一把故意不可重入的锁](#level-4)
- [🟢 Level 5：打烊协议 —— shutdown、shutdownNow 与那个没人接的返回值](#level-5)
- [🟢 Level 6：厨房家族 —— 定时池、分治池与那个全局共享的默认池](#level-6)
- [🟢 Level 7：虚拟线程与响应式 —— 当"线程"不再稀缺](#level-7)
- [🔬 Level 7.5：硬件视角补充 —— 从对象字段到缓存行](#level-7-5)
- [🧪 合书自测：一页时序图](#self-check)
- [⚠️ 线程池的坑与细节](#pitfalls)
- [📮 提交方式对比：execute / submit / invokeAll / invokeAny / CompletableFuture](#submission)
- [📊 线程池竖切总表 T0–T8](#t0-t8)
- [📚 版本勘误与延伸阅读](#references)
- [🔧 线上排查工具箱：从"服务变慢了"到定位根因](#troubleshooting)
- [🧰 实战速查：默认池的雷 + 配置模板 + Review 清单](#cheatsheet)
- [🏆 终章：从"我懂线程池"倒推到"我在生产做了什么决策"](#production-decisions)
- [🌍 加分视野：这不是 Java 独有——不同运行时如何解决同一座厨房的问题](#cross-runtime)

> 🔬 **配套可运行实验（两个）**
>
> 1. [`demo/ThreadPoolLabs.java`](../../demo/ThreadPoolLabs.java) —— 五个最反直觉的论断（幽灵 Worker、无界队列废掉 max、submit 吞异常、守护线程静默丢任务、五种提交方式差异）都配了可复现代码。
>    `javac -encoding UTF-8 ThreadPoolLabs.java && java ThreadPoolLabs`，JDK 8+ 均可运行（这些语义 JDK 8→21 未变）。
> 2. [`demo/CountMaskEquivalenceLab.java`](../../demo/CountMaskEquivalenceLab.java) —— 验证 `CAPACITY`→`COUNT_MASK` 那次重构**到底等不等价**：穷举状态判断（等价）+ 找出人数上限的分歧取值（不等价）+ 真实线程池复现。
>    **需分别用 JDK 8 与 JDK 21 各跑一次，对比 Lab 3 的输出。** 详见 [Level 2](#level-2)。
>
> 文中引用的输出均为真实运行结果。

---

<a id="roadmap"></a>

## 📍 使用说明：七级线程池能力地图

### 🚀 带着问题来的？走这条快速路径

> 📌 本文体量已接近一本小册子，**默认的线性通关不是唯一读法**。如果你是带着具体问题来的，直接跳：

| 你现在的问题 | 直接去 |
|---|---|
| 线上出事了，要排查 | [🔧 线上排查工具箱](#troubleshooting)（先看"三个数判定表"） |
| 要新建一个池，参数怎么定 | [🧰 实战速查](#cheatsheet) 第 2–3 节（模板 + 从 SLA 反推） |
| Code Review / 架构评审 | [🧰 实战速查](#cheatsheet) 第 4 节（16 条清单）+ [🏆 决策卡](#production-decisions) |
| 虚拟线程 pinning、要不要迁移 | [Level 7](#level-7) + [决策卡 6](#production-decisions) |
| Spring 项目：`@Async`/`@Scheduled` 怎么配 | [决策卡 5](#production-decisions)、[决策卡 6](#production-decisions) |
| TraceId 跨线程池丢了 | [坑 4-B：`InheritableThreadLocal` 为何失效](#pitfalls) |
| `submit` / `invokeAll` 该用哪个 | [📮 提交方式对比](#submission) |
| 想上动态线程池 / 舱壁隔离框架 | [决策卡 7](#production-decisions)、[决策卡 8](#production-decisions) |
| 面试要讲原理 | Level 1→7 顺序读，每层末尾的 🔴 口诀是复述稿 |
| 只想验证某个结论是真的 | [`demo/ThreadPoolLabs.java`](../../demo/ThreadPoolLabs.java) 五个可运行实验 |

### 七级能力地图

| 层级 | 要打穿的认知墙 | 通关标准 |
|---|---|---|
| L1 | 线程池不是"复用线程"，而是**容量契约** | 说清它同时管控的四种资源，而不只是线程 |
| L2 | `ctl` 不是"炫技的位运算" | 解释为什么状态与计数**必须**原子地一起读 |
| L3 | 三道门不是 if-else 顺序 | 说清为什么入队后必须**复查**，以及复查漏了会怎样 |
| L4 | Worker 继承 AQS 不是图省事 | 解释为什么它用**不可重入**锁，以及 `setState(-1)` 防的是什么 |
| L5 | shutdown 不是"停止" | 说清两种打烊的语义差异与 `shutdownNow` 返回值的责任 |
| L6 | 池不止一种，默认池是共享的 | 说清定时池为何 `max` 无效、`commonPool` 为何会互相拖垮 |
| L7 | 虚拟线程让"池化线程"失去理由 | 说清为什么**不要池化虚拟线程**，以及容量阀门该搬到哪 |
| L7.5 | 再往下还有硬件（选修） | 说清 `mainLock` 何时才热、伪共享为何"先测再改"、虚拟线程的栈成本转移到了哪 |

---

<a id="kitchen-map"></a>

## 🏭 全书唯一厨房地图

```text
                    ┌──────────── 中央厨房 ────────────┐
                    │  ctl：一个 int = 营业状态 + 在岗人数 │
                    │  高 3 位：RUNNING/SHUTDOWN/STOP...  │
                    │  低 29 位：workerCount              │
                    └────────────────────────────────────┘
                                    ▲
      ┌─────────────────────────────┼─────────────────────────────┐
      │                             │                             │
  ① 核心厨师席位              ② 订单传送带                  ③ 临时厨师席位
  corePoolSize=2             workQueue (有界!)            max - core = 3
  ┌────┐ ┌────┐              ┌──┬──┬──┬──┐               ┌────┐┌────┐┌────┐
  │ W1 │ │ W2 │  ←─ 满了 ──→ │T3│T4│T5│  │ ←─ 也满了 ─→  │ W3 ││ W4 ││ W5 │
  └────┘ └────┘              └──┴──┴──┴──┘               └────┘└────┘└────┘
   常驻，不裁                  缓冲，不是无限               闲置超 keepAlive 就裁
                                    │
                                    ▼ 三道门全关
                            ④ 门口的保安 RejectedExecutionHandler
                              Abort / CallerRuns / Discard / DiscardOldest
                              ★ 保安的处置方式是业务决策，不是技术细节

  订单 T ──execute()──▶ 门1：核心席位有空？ ──否──▶ 门2：传送带塞得下？
                              │是                        │否
                              ▼                          ▼
                         开新核心厨师              门3：临时席位有空？
                                                      │否
                                                      ▼
                                                    保安处置
```

| 厨房元素 | 线程池概念 | 它负责的唯一一件事 |
|---|---|---|
| 营业状态牌 | `ctl` 高 3 位 | 现在还收不收单、还出不出餐 |
| 在岗人数牌 | `ctl` 低 29 位 | 当前有多少 Worker 活着 |
| 核心厨师 | `corePoolSize` 内的 Worker | 常驻产能，默认不因空闲被裁 |
| 临时厨师 | 超出 core 的 Worker | 应急产能，空闲超时即裁 |
| 传送带 | `workQueue` | 缓冲突发，**它的长度就是你的延迟上限** |
| 传送带类型 | Array/Linked/Synchronous | 决定"先扩人还是先排队"这一根本策略 |
| 门口保安 | `RejectedExecutionHandler` | 容量耗尽时的**业务语义**，不是技术兜底 |
| 厨师的工牌锁 | `Worker extends AQS` | 标记"我正在炒菜，别中断我" |
| 厨师的下班判定 | `getTask()` 返回 null | 唯一的线程回收出口 |
| 厨师长的收工令 | `shutdown()` / `shutdownNow()` | 两种完全不同的打烊语义 |

> 🧵 暗线起点：订单 T 调用 `execute(T)`。此刻 T 还没被任何厨师接手，也还不在传送带上。全书不断问：**T 现在在哪？`ctl` 是多少？谁有责任执行它？如果此刻宕机，T 会丢吗？**

---

<a id="level-1"></a>

## 🟢 Level 1：先回答为什么不能来一单开一个厨师

### Why——`new Thread()` 每次都创建，到底死在哪？

**徒弟**：来一个任务开一个线程，代码最简单，为什么需要一整套线程池？

```java
public void handle(Request req) {
    new Thread(() -> process(req)).start();   // 简单、直观、能跑
}
```

**老陈**：它在压测里能跑，在大促里会死。你只算了"创建线程"这一件事的成本，但实际上你同时放弃了四道闸门。

先说创建成本本身。一个平台线程（platform thread）是 1:1 映射到 OS 线程的：需要 `clone` 系统调用、内核栈与线程控制块、以及一块默认量级在**几百 KB 到 1MB**（`-Xss` 决定，随平台不同）的用户栈。这个量级的意义不在于"慢"，而在于**它是不可压缩的常量成本**：你的任务可能只跑 200μs，创建与销毁的开销却是同一量级甚至更高。

但真正致命的不是成本，是**没有闸门**：

| 朴素方案 | 它本来想解决什么 | 它留下的致命账 |
|---|---|---|
| `new Thread()` 每任务 | 并发执行，代码直观 | 无上限。10 万请求 = 尝试 10 万 OS 线程 → OOM / `OutOfMemoryError: unable to create native thread` |
| 手工维护一组线程 | 复用，省去创建开销 | 要自己写队列、回收、异常隔离、关闭——每一处都是并发陷阱 |
| `Executors.newFixedThreadPool` | 官方封装，一行搞定 | **队列无界**。任务堆积转化为堆内存增长，OOM 从"线程数"换到"堆" |
| `Executors.newCachedThreadPool` | 弹性伸缩，不排队 | **线程数无上限**（`Integer.MAX_VALUE`），洪峰直接打爆 |

**老陈**：看出规律了吗？`new Thread` 的问题不是"慢"，是**没有拒绝的能力**。一个系统只要不能拒绝，它就一定会在某个流量水位以整体崩溃的方式拒绝——而且是在你最不想要的时刻，用最难看的方式。

### What——线程池的真正定义：四种资源的容量契约

绝大多数人把线程池理解成"复用线程的工具"。这个定义太窄，窄到会让你在虚拟线程时代得出完全错误的结论（Level 7 会算这笔账）。准确的定义是：

```text
线程池 = 一份同时管控四种资源的容量契约

  ① 并发度   ← corePoolSize / maximumPoolSize
  ② 排队量   ← workQueue 容量（= 你愿意承受的延迟上限）
  ③ 拒绝语义 ← RejectedExecutionHandler（= 超出容量时的业务行为）
  ④ 生命周期 ← shutdown / awaitTermination（= 优雅退出的能力）
```

| 你以为它管的 | 它实际还管的 | 忽略后的线上表现 |
|---|---|---|
| 线程复用 | 并发上限 | 下游被打爆，雪崩 |
| 线程复用 | 排队上限 | 队列无界 → 堆 OOM，或延迟无限增长 |
| 线程复用 | 拒绝行为 | 默认抛异常，调用方未处理 → 请求 500 |
| 线程复用 | 关闭语义 | 发布时任务被硬中断，数据不一致 |

> 📌 **最容易错的点**：`corePoolSize` 不是"最小线程数"，`maximumPoolSize` 也不是"你能达到的并发度"。在**有界队列**下，只有队列**塞满之后**才会创建超过 core 的线程。这意味着：**队列越大，`maximumPoolSize` 越形同虚设。** 用 `LinkedBlockingQueue`（无界）时，`maximumPoolSize` **永远不会生效**——这是线上事故的头号来源，Level 3 会用时序图把它钉死。

### How——T 的第一次 `execute()`：先看清它可能的四种归宿

订单 T 进入厨房时，只有四种结局，没有第五种：

```text
T ──execute()──┬──▶ ① 立刻被新开的核心厨师执行   （core 未满）
               ├──▶ ② 躺在传送带上等待           （core 满，队列未满）
               ├──▶ ③ 被新开的临时厨师执行       （队列满，max 未满）
               └──▶ ④ 交给门口保安处置           （全满）
```

```java
// 教学伪代码，展示职责边界（真实实现见 Level 3）
public void execute(Runnable command) {
    if (在岗人数 < corePoolSize && 开新厨师成功(command)) return;   // 门 1
    if (还在营业 && 传送带.offer(command))          { 复查(); return; }  // 门 2
    if (开新厨师成功(command))                       return;         // 门 3
    保安.处置(command, this);                                        // 门 4
}
```

**徒弟**：为什么门 2 成功入队后还要"复查"？任务不是已经安全躺在传送带上了吗？

**老陈**：这是全篇最精妙也最容易被跳过的三行代码。留到 Level 3，我用一个能让任务**永远不被执行**的时序把它讲透。现在你只要记住一个反直觉的事实：**任务成功进入队列，不等于有人会来取它。**

### Transfer——设计任何"池"之前必须先回答的三问

线程池只是"资源池"这一模式的一个实例。连接池、对象池、内存池、协程池都在回答同一组问题：

1. **稀缺的到底是什么？** 是线程、连接、许可，还是下游的处理能力？搞错这个，后面全错。
2. **超出容量时的正确业务行为是什么？** 排队（接受延迟）、拒绝（快速失败）、降级（返回兜底），还是背压（让上游变慢）？**这是产品决策，不是技术默认值。**
3. **谁来观测容量水位？** 没有 `queue.size()`、`activeCount`、`rejectedCount` 的池，等于没有仪表盘的锅炉。

L1 留下的问题是：厨房必须**同时**知道"还营不营业"和"在岗几个人"。为什么不能用两个变量分别存？这逼出 `ctl`。

> 🔴 **口诀**：池不是省线程，是立容量契约；无界即无闸，拒绝是业务决策。

---

<a id="level-2"></a>

## 🟢 Level 2：ctl —— 为什么状态与计数必须挤进一个 int

### 👶 前置知识关卡

- [ ] 能说出线程池管控的四种资源，而不只是"复用线程"吗？
- [ ] 知道有界队列下 `maximumPoolSize` 何时才生效吗？
- [ ] 能说清 `new Thread()` 真正致命的是成本还是"没有拒绝能力"吗？

### Why——两个变量存两件事，为什么会撕裂？

**徒弟**：营业状态和在岗人数是两件事，用两个字段不是更清晰？

```java
private volatile int runState;       // RUNNING / SHUTDOWN / STOP
private final AtomicInteger workerCount = new AtomicInteger();
```

**老陈**：清晰，但**不正确**。并发下，两个独立的原子变量组合起来**不是原子的**。看这个时序：

```text
线程 A（提交任务）              线程 B（调用 shutdown）
─────────────────────────      ─────────────────────────
① 读 runState = RUNNING
   （判断：还营业，可以开人）
                               ② 写 runState = SHUTDOWN
                               ③ 遍历现有 Worker 发中断
④ workerCount.CAS 0 → 1
⑤ 创建并启动新 Worker          ← 它诞生在 shutdown 之后，
                                  没人给它发过中断，
                                  它可能永远阻塞在 take() 上
```

结果：**厨房已经宣布打烊，却在打烊后凭空多出一个厨师。** 这个厨师是"幽灵厨师"：`shutdown()` 的中断遍历发生在它出生之前，因此它错过了唯一一次被通知的机会。如果队列此时为空且它阻塞在 `workQueue.take()` 上，它会永远等下去，池永远无法到达 `TERMINATED`，`awaitTermination` 永远超时，你的服务永远优雅停不下来。

这就是"检查—执行"竞态（check-then-act race）的经典形态：**你检查的那个状态，在你执行时可能已经不成立了。**

| 朴素方案 | 它本来想解决什么 | 它留下的致命账 |
|---|---|---|
| 两个 volatile 变量 | 语义清晰，各管各的 | 组合读写非原子 → 幽灵 Worker、状态撕裂 |
| 加一把全局锁保护两者 | 保证组合原子性 | `execute` 是**最热路径**，每次提交都抢全局锁 → 串行化瓶颈 |
| 只用一个状态，人数另算 | 减少变量 | 无法在一次 CAS 中同时表达"状态没变且人数+1" |

### What——ctl：一个 int，两个真相，一次 CAS

`ThreadPoolExecutor` 的答案是把两件事塞进**一个** `AtomicInteger`：

```java
// OpenJDK 21，ThreadPoolExecutor.java（与 JDK 8 逐位一致）
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;      // = 29
private static final int COUNT_MASK = (1 << COUNT_BITS) - 1; // 低 29 位掩码

private static final int RUNNING    = -1 << COUNT_BITS;      // 111 高位
private static final int SHUTDOWN   =  0 << COUNT_BITS;      // 000
private static final int STOP       =  1 << COUNT_BITS;      // 001
private static final int TIDYING    =  2 << COUNT_BITS;      // 010
private static final int TERMINATED =  3 << COUNT_BITS;      // 011

private static int runStateOf(int c)     { return c & ~COUNT_MASK; }
private static int workerCountOf(int c)  { return c & COUNT_MASK; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

> 📌 **版本对照（已锁定到具体 commit 与 JBS 编号）**：JDK 8 里这个掩码常量叫 `CAPACITY`，后来改名为 `COUNT_MASK`。`CAPACITY` 容易被误读为"池的容量"，而它实际是"计数字段的位掩码"。
>
> **这次改名的确切出处（可点开核对，不必相信本文）**：
>
> | 项 | 值 |
> |---|---|
> | **JBS** | [**JDK-8186056**](https://bugs.openjdk.org/browse/JDK-8186056) "Miscellaneous changes imported from jsr166 CVS 2017-09" |
> | **fixVersion** | **10**（resolved 2017-10-03） |
> | **commit** | [`c3664b7f38a0`](https://github.com/openjdk/jdk/commit/c3664b7f38a0)，作者 **Doug Lea**，Reviewed-by: martin, psandoz |
> | **同一 commit 内** | `CAPACITY`→`COUNT_MASK` **与** `addWorker`/`getTask` 的 `runStateAtLeast` 改写是**一次提交完成的**，不是两次演进 |
>
> ⚠️ **一处自我修正**：本文早期版本称这次改名发生在 "JDK 21"，**这是错的**，正确答案是 **JDK 10**。逐版本实测（`CAPACITY` / `COUNT_MASK` 出现次数）：
>
> ```text
> JDK 8   CAPACITY=5   COUNT_MASK=0      ← 旧名
> JDK 9   CAPACITY=5   COUNT_MASK=0      ← 旧名（所以不是"9~10 之间"，就是 10）
> JDK 10  CAPACITY=0   COUNT_MASK=6      ← ★ 改名落地于此
> JDK 11  CAPACITY=0   COUNT_MASK=6
> JDK 21  CAPACITY=0   COUNT_MASK=6
> ```
>
> ⚠️ **第二处自我修正**：本文早期版本称"**JDK 11 与 JDK 21 的 `addWorker` 逐字完全相同**"，**这也是错的**。实际 `diff` 有且仅有一行：
>
> ```diff
> - t.start();              // JDK 11
> + container.start(t);     // JDK 21
> ```
>
> 这行由 **JDK 19 的 JEP 444（虚拟线程，[JDK-8284161](https://bugs.openjdk.org/browse/JDK-8284161)）** 引入：Worker 线程被注册进 `SharedThreadContainer`，使 `jcmd Thread.dump_to_file` 能显示线程的容器归属。**它不改变执行语义，但"逐字相同"这个说法本身不成立。**
>
> 📌 这条修正是本文"版本判断必须可验证"原则的一次实践：**我最初的说法没有拿出 diff，所以它是不可信的——哪怕主结论（语义等价）方向是对的，时间点和"逐字相同"两处细节都错了。** 你可以用下面的命令在 30 秒内复核到 commit 级别：
>
> ```bash
> # 1. 确认改名落在哪个版本（逐版本抓源码数出现次数）
> for v in 8 9 10 11 17 21; do
>   [ $v -le 9 ] && P="jdk${v}u/master/jdk/src/share/classes" || P="jdk${v}u/master/src/java.base/share/classes"
>   U="https://raw.githubusercontent.com/openjdk/$P/java/util/concurrent/ThreadPoolExecutor.java"
>   echo "JDK $v: CAPACITY=$(curl -s $U|grep -c CAPACITY) COUNT_MASK=$(curl -s $U|grep -c COUNT_MASK)"
> done
>
> # 2. 直接看那次改名的原始 diff（含 runStateAtLeast 改写）
> curl -s https://github.com/openjdk/jdk/commit/c3664b7f38a0.patch \
>   | grep -E '^[+-].*(COUNT_MASK|CAPACITY|runStateAtLeast)'
> ```

内存布局：

```text
 31 30 29 │ 28 27 26 ... 2 1 0
 ─────────┼────────────────────
  高 3 位  │      低 29 位
  运行状态 │    workerCount
          │  最大 2^29 - 1 ≈ 5.36 亿
```

**为什么状态值要用位移而非 0/1/2/3？** 因为这样设计后，五个状态在 int 上是**单调递增**的：

```text
RUNNING(-536870912) < SHUTDOWN(0) < STOP(536870912) < TIDYING < TERMINATED
```

于是所有状态判断退化成一次整数比较，且**无需先解包**：

```java
private static boolean runStateLessThan(int c, int s) { return c < s; }
private static boolean runStateAtLeast(int c, int s)  { return c >= s; }
private static boolean isRunning(int c)               { return c < SHUTDOWN; }
```

`isRunning(c)` 就是 `c < 0`——因为只有 `RUNNING` 的高位是 1（负数）。**一个符号位判断，就回答了"还收不收单"。**

### How——五种营业状态的完整迁移图

```text
        RUNNING  ──── shutdown() ────▶  SHUTDOWN
        收单+出餐                        不收单，但把传送带上的做完
           │                                  │
           │ shutdownNow()                    │ 队列空 且 人数=0
           │                                  │
           ▼                                  ▼
         STOP    ────────────────────▶     TIDYING
      不收单+不出餐+中断在炒的菜        所有任务终结，人数归零
      （返回未执行的任务清单）          正在执行 terminated() 钩子
                                             │
                                             │ terminated() 返回
                                             ▼
                                        TERMINATED
                                      awaitTermination 才会返回
```

| 状态 | 收新单？ | 做传送带上的单？ | 中断正在炒的菜？ | 触发方式 |
|---|---|---|---|---|
| RUNNING | ✅ | ✅ | ❌ | 初始状态 |
| SHUTDOWN | ❌ | ✅ | ❌（只中断闲置的） | `shutdown()` |
| STOP | ❌ | ❌ | ✅ | `shutdownNow()` |
| TIDYING | ❌ | ❌ | — | 自动：队列空 + 人数 0 |
| TERMINATED | ❌ | ❌ | — | `terminated()` 钩子执行完 |

现在看 `ctl` 如何消灭幽灵厨师。这是 JDK 21 `addWorker` 的真实开头：

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (int c = ctl.get();;) {
        // 已经 SHUTDOWN 且（已 STOP 或 带着新任务 或 队列已空）→ 拒绝开人
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP)
                || firstTask != null
                || workQueue.isEmpty()))
            return false;

        for (;;) {
            if (workerCountOf(c) >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;              // ★ 一次 CAS 同时校验了状态与人数
            c = ctl.get();                // CAS 失败，重读
            if (runStateAtLeast(c, SHUTDOWN))
                continue retry;           // 状态变了，回到外层重新判断
            // 否则只是人数变了，重试内层
        }
    }
    // ... 真正创建 Worker
```

关键在 `compareAndIncrementWorkerCount(c)`：它 CAS 的是**整个 ctl**。这意味着，如果在我读 `c` 之后、CAS 之前，有人调用了 `shutdown()`（改变了高 3 位），那么 `ctl` 的整数值已变，**我的 CAS 必然失败**。失败后重读，发现 `runStateAtLeast(c, SHUTDOWN)`，`continue retry` 回到外层，被第一个 if 拦下，返回 false。

> 📌 **这就是 ctl 存在的全部理由**：不是为了省 4 个字节内存，而是为了让"状态未变"与"人数+1"成为**同一次 CAS 的原子结果**。省内存是副产品，防撕裂才是目的。

**徒弟**：JDK 8 的这段代码看起来不太一样？

**老陈**：写法不同，语义相同。JDK 8 是：

```java
// JDK 8
int rs = runStateOf(c);
if (rs >= SHUTDOWN &&
    ! (rs == SHUTDOWN &&
       firstTask == null &&
       ! workQueue.isEmpty()))
    return false;
```

那个双重否定 `! (A && B && !C)` 极其难读。JDK 21 用德摩根律展开成正向的 `runStateAtLeast(c, STOP) || firstTask != null || workQueue.isEmpty()`。**你可以自己验证：两者真值表完全相同**（穷举五种状态 × 七种人数 × firstTask 是否为 null × 队列是否为空 = 140 组，零处不一致）。

#### ⚠️ 但"语义完全等价"这个结论有一个必须标注的例外

同一次 [JDK-8186056](https://bugs.openjdk.org/browse/JDK-8186056) 还改写了**人数上限的判断**，这一处**不是纯粹的写法变更**：

```java
// JDK 8/9
int wc = workerCountOf(c);
if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
    return false;

// JDK 10+
if (workerCountOf(c) >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
    return false;
```

- 旧写法的有效上限是 `min(bound, CAPACITY)`——**先截断到 5.36 亿**
- 新写法的有效上限是 `bound & COUNT_MASK`——**取低 29 位，会回绕**

当 `corePoolSize`/`maximumPoolSize` 恰为 **2^29 的整数倍**时，`bound & COUNT_MASK == 0`，`addWorker` 直接返回 false：

```text
bound=2^29 (536870912)  → 旧 min()=536870911   新 &=0           *** 行为不同 ***
bound=2^30 (1073741824) → 旧 min()=536870911   新 &=0           *** 行为不同 ***
bound=600000000         → 旧 min()=536870911   新 &=63129088    *** 行为不同 ***
bound=Integer.MAX_VALUE → 旧 min()=536870911   新 &=536870911   相同（低 29 位全 1）
```

**三个真实 JDK 上的实测**（`core = max = 2^29`，构造函数完全接受该值，然后提交一个任务）：

```text
JDK 1.8.0_492 → task executed within 3s? true    poolSize=1  queued=0   ✅ 任务执行
JDK 11        → task executed within 3s? false   poolSize=0  queued=1   ❌ 永久滞留
JDK 21.0.12   → task executed within 3s? false   poolSize=0  queued=1   ❌ 永久滞留
```

> 🔬 **可复现**：[`demo/CountMaskEquivalenceLab.java`](../../demo/CountMaskEquivalenceLab.java) —— Lab 1 穷举证明状态判断改写**完全等价**（245 组零不一致），Lab 2 列出人数上限的分歧取值，Lab 3 在真实线程池上复现。`javac -encoding UTF-8 CountMaskEquivalenceLab.java && java CountMaskEquivalenceLab`，**分别用 JDK 8 和 JDK 21 跑一遍，Lab 3 的输出不同**。上面三行数据即为该程序的真实输出。
>
> 📌 **怎么定性这件事**：这是一个**实践中几乎不可能触发**的边界（没人会把线程数设成 5.36 亿），JDK 也在 `corePoolSize`/`maximumPoolSize` 的字段注释里明确写了 *"the effective limit is `corePoolSize & COUNT_MASK`"*——**它是被文档化的行为，不是 bug**。
>
> 但它足以支撑一个更精确的表述：这次重构**在任何现实参数下语义等价，在数学上并不严格等价**。本文选择把它写出来，是因为全文反复要求别人"拿出 diff"——那么我自己给出的"纯粹的命名改进""值和用法完全相同"这类断言，也必须接受同样强度的检验。**穷举验证之后，"完全相同"这四个字就该改成"在现实取值域内相同"。**

这是本文反复强调的方法论——看到版本差异，先问"是语义变了，还是只是写法变了"，而不是先假设"新版本重写了"；但同样重要的是：**当你判定"只是写法变了"时，要把这个判断也穷举一遍，而不是看一眼就下结论。**

### Transfer——位打包的适用边界

`ctl` 是一个漂亮的设计，但它**不是可以随便模仿的技巧**。用它之前问三个问题：

1. **这几个字段是否真的必须原子地一起变？** 如果只是"读起来方便"，不值得牺牲可读性。
2. **字段的取值范围是否可静态确定？** `ctl` 敢用 29 位，是因为 5.36 亿线程在物理上不可能达到。你的字段有这个保证吗？
3. **是否已有更合适的表达？** 现代 Java 里，`AtomicReference<不可变记录>` 往往是更清晰的选择：

```java
record PoolState(RunState state, int workerCount) {}
private final AtomicReference<PoolState> state = new AtomicReference<>(...);
```

代价是每次 CAS 都要分配一个对象。在 `execute` 这种**每请求都走**的超热路径上，Doug Lea 选择了位打包；在你的业务代码里，除非你也在写每秒千万次调用的基础设施，否则请优先选可读性。

L2 解决了"状态和人数如何原子地一起看"。但 T 提交时具体走哪道门、为什么门 2 之后还要复查——`ctl` 只是提供了工具，决策逻辑在 `execute` 里。

> 🔴 **口诀**：高 3 位管营业，低 29 位管人头；一次 CAS 定两件事，防的是幽灵厨师。

---

<a id="level-3"></a>

## 🟢 Level 3：execute 的三道门 —— 以及那次能救命的复查

### 👶 前置知识关卡

- [ ] `isRunning(c)` 为什么只是一次符号位判断？
- [ ] 为什么 `addWorker` 的 CAS 失败后要区分"状态变了"和"人数变了"？
- [ ] 五种运行状态为什么必须单调递增？

### Why——为什么是"先排队后扩容"，而不是反过来？

**徒弟**：任务来了，直觉上应该先把厨师招满（扩到 max），实在忙不过来才排队。为什么 JDK 是反过来的？

**老陈**：因为**创建线程是有成本的，而排队几乎没有**。这个顺序背后是一条明确的价值判断：

```text
JDK 的策略：core 用满 → 队列用满 → 才扩到 max
隐含前提：  线程是昂贵资源，队列是廉价缓冲
            突发流量大多是"短暂尖峰"，排一下队就过去了
            为一个 50ms 的尖峰创建 100 个线程，是浪费
```

这个判断在**平台线程 + 有界队列**的世界里是对的。但它有两个著名的反直觉后果：

**后果一（致命）**：如果队列是**无界**的，第二道门永远不会失败，**第三道门永远不会被触达**，`maximumPoolSize` 变成一个纯粹的装饰参数。

```java
// Executors.newFixedThreadPool 的真实构造
new ThreadPoolExecutor(n, n, 0L, MILLISECONDS,
                       new LinkedBlockingQueue<Runnable>());  // ← 无界！
// 这里 core == max，所以看不出问题。但如果你写：
new ThreadPoolExecutor(2, 200, 60L, SECONDS,
                       new LinkedBlockingQueue<>());          // ← 无界！
// 那个 200 是假的。池永远只有 2 个线程，其余任务无限堆积到 OOM。
```

#### 🔬 亲手验证：那个 "200" 到底有多假

配套代码 `java ThreadPoolLabs 2`。实验用**两个只有队列类型不同、其余参数完全相同**的池做对照：

```java
// A 组：core=2, max=200, 无界队列
new ThreadPoolExecutor(2, 200, 60, SECONDS, new LinkedBlockingQueue<>());
// B 组：core=2, max=200, 有界队列（容量 10）
new ThreadPoolExecutor(2, 200, 60, SECONDS, new ArrayBlockingQueue<>(10));
// 各提交 100 个会阻塞住的任务，然后观察 poolSize
```

**真实运行输出**：

```text
  A 组：core=2, max=200, queue=LinkedBlockingQueue()【无界】
     提交 100 个阻塞任务后 → poolSize=2, queueSize=98
     ✅ poolSize 停在 2（= corePoolSize），max=200 从未生效

  B 组：core=2, max=200, queue=ArrayBlockingQueue(10)【有界】—— 其余参数完全相同
     提交 100 个阻塞任务后 → poolSize=90, queueSize=10
     ✅ poolSize 涨到 90，第三道门被触达，max 真正生效
```

> 📌 **`2` 和 `90` 的差距，全部来自那一个队列参数。** A 组的 `200` 不是"暂时没用上"，而是**在这套配置下永远不可能生效**——它是一个纯粹的心理安慰。
>
> 更值得警惕的是 A 组的失败模式：它**不会**报"线程不够"，也不会拒绝任何任务。它只是安静地让 `queueSize` 一路涨到 98、980、98000……直到堆内存耗尽。**这是一种没有任何早期告警信号的故障。**

**后果二**：Tomcat 等 Web 容器**故意不用**这个策略。它们的诉求是"先扩容保延迟，再排队"，因此实现了自定义队列（`TaskQueue`），在 `offer()` 里判断"如果线程数还没到 max，就返回 false 假装队列满"，逼迫线程池走第三道门。**知道了 JDK 的策略，你才看得懂 Tomcat 为什么要这么绕。**

### What——三道门的完整决策表

```java
// OpenJDK 21，与 JDK 8 逐行一致
public void execute(Runnable command) {
    if (command == null) throw new NullPointerException();
    int c = ctl.get();
    // ── 门 1：核心席位没坐满？直接开人
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true)) return;
        c = ctl.get();                       // 开人失败（并发/状态变化），重读
    }
    // ── 门 2：还营业 且 塞得进传送带？
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();             // ★★★ 复查，本层的主角
        if (!isRunning(recheck) && remove(command))
            reject(command);                 // 复查发现已打烊 → 撤回并拒绝
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);          // 复查发现没人了 → 补一个空手厨师
    }
    // ── 门 3：队列满了，还能开临时厨师吗？
    else if (!addWorker(command, false))
        reject(command);                     // ── 门 4：保安处置
}
```

| 门 | 判断条件 | 通过后 | 失败则 |
|---|---|---|---|
| 门 1 | `workerCount < corePoolSize` | 创建**核心** Worker，T 作为它的 firstTask | 落到门 2 |
| 门 2 | `isRunning && queue.offer(T)` 成功 | T 在队列中等待 → **必须复查** | 落到门 3 |
| 门 3 | `addWorker(T, false)` 成功 | 创建**临时** Worker（受 max 限制） | 落到门 4 |
| 门 4 | — | — | `reject(T)`，交给 handler |

> 📌 **最容易错的点**：门 1 判断的是 `workerCount < corePoolSize`，**不是"有没有空闲线程"**。这意味着即使现有 2 个核心线程都在闲着，只要 `workerCount(2) < corePoolSize(5)`，第 3 个任务仍会**创建新线程**而不是复用空闲的。线程池在 core 未满阶段是"宁可开人也不复用"——因为判断"谁空闲"需要额外的同步开销，而 core 阶段的线程反正要常驻。

### How——那三行复查：一个能让任务永远饿死的时序

**徒弟**：任务已经成功 `offer` 进队列了，这不就安全了吗？为什么还要复查？

**老陈**：因为**队列里有任务，不代表有人会来取。** 我给你演示两种能让 T 永远躺在传送带上的时序。

**场景 A：allowCoreThreadTimeOut 下的"全员离场"**

前提：`corePoolSize=1`，`allowCoreThreadTimeOut(true)`（核心线程也会因空闲被回收）。

```text
时刻   提交线程（带着 T）              唯一的厨师 W1
────  ────────────────────────      ──────────────────────────
 t1                                  poll(keepAlive) 超时返回 null
 t2   读 ctl：workerCount = 1
      → 门 1 判断 1 < 1 为假，跳过
 t3                                  getTask() 返回 null，决定下班
                                     CAS workerCount 1 → 0
 t4                                  processWorkerExit：从 workers 移除
 t5   queue.offer(T) 成功 ✅
      T 静静躺在传送带上
      ───────────────────────────────────────────────
      此刻：队列有 1 个任务，在岗厨师 0 人
      如果没有复查 → T 永远不会被执行 ❌
      ───────────────────────────────────────────────
 t6   recheck = ctl.get()
      workerCountOf(recheck) == 0  ✅ 命中！
 t7   addWorker(null, false)
      → 开一个"空手厨师"W2，它不带 firstTask，
        直接去传送带上取 T
 t8                                  W2 执行 T ✅ 得救
```

**关键洞察**：`addWorker(null, false)` 传的第一个参数是 `null`——这个 Worker **不带自己的任务**，它诞生的唯一使命就是去队列里捡被遗弃的任务。这是 JDK 为"队列非空但无人消费"这一活性缺口打的补丁。

**场景 B：offer 与 shutdown 交错**

```text
提交线程                        关闭线程
──────────────────────      ──────────────────────
① isRunning(c) 为真
                            ② shutdown()：ctl 高位改为 SHUTDOWN
                            ③ 中断所有**闲置**的 Worker
④ queue.offer(T) 成功
   ─────────────────────────────────────────────
   T 进入了一个已经打烊的厨房的传送带。
   SHUTDOWN 语义是"做完存量"，所以 T 理论上会被执行；
   但如果所有 Worker 已在 ③ 之后退出，T 就无人认领。
   ─────────────────────────────────────────────
⑤ recheck 发现 !isRunning
⑥ remove(T) 成功撤回
⑦ reject(T) → 调用方立刻得到 RejectedExecutionException
   （明确失败，好过静默丢失）
```

#### 🔬 亲手验证：让幽灵状态在你的机器上真实出现

这是全文最反直觉的洞察，所以它值得一个能跑的实验。配套代码见 [`demo/ThreadPoolLabs.java`](../../demo/ThreadPoolLabs.java)，`java ThreadPoolLabs 1` 即可运行。

实验设计的关键在于：**先绕过 `execute()` 直接 `getQueue().offer()`，人为制造一个"补丁不存在"的世界**，证明这个缺口是真实的；再走正常 `execute()`，证明补丁确实生效。

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        1, 1, 200, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());
pool.allowCoreThreadTimeOut(true);          // ★ 核心线程也会被回收

pool.execute(() -> {});                     // 预热，创建出那个唯一的 Worker
Thread.sleep(600);                          // 等超过 keepAlive，让它退出

pool.getQueue().offer(task);                // ★ 绕过 execute，直接塞进队列
// 此刻：poolSize=0, queueSize=1 —— 没有任何人会来取它
```

**在 JDK 21 上的真实运行输出**（这部分语义 JDK 8→21 完全一致）：

```text
  预热后        : poolSize=1, queueSize=0
  等待 600ms，让唯一的 Worker 因空闲超时而退出...
  回收后        : poolSize=0  ← ★ 池里一个线程都没有了

  【模拟缺口】绕过 execute()，直接 getQueue().offer() 塞入任务：
  塞入后        : poolSize=0, queueSize=1
  等待 800ms 后 : queueSize=1, 任务执行次数=0
  ✅ 论断成立：任务【永远躺在队列里】，无人执行。

  【补丁生效】现在用正常的 execute() 提交，触发那三行 recheck：
  ✅ 被 pool-1-thread-2 执行 —— execute() 发现 workerCount==0，补了一个空手厨师
  ★ 顺带证明：刚补的这个 Worker 也把之前那个幽灵任务捡走了 → 执行次数=1
```

> 📌 **注意最后一行的额外收获**：那个被"遗弃"的幽灵任务，最终被后来补上的空手 Worker 一并捡走了。这恰好演示了 `addWorker(null, false)` 的设计意图——它补的不是"某一个特定任务的执行者"，而是**队列的消费能力本身**。
>
> 同时请留意 `core = max = 1` 却出现 `poolSize = 0`：这直接证伪了"corePoolSize 是最小线程数"这个流传极广的说法。

> 📌 这两个分支体现了同一条工程哲学：**并发系统里，"我刚才检查过"是一句没有价值的话。** 任何跨越了状态发布点的判断，都必须在关键操作后重新验证。这与 AQS 中"被 unpark 后仍须重新 `tryAcquire`"、Condition 中"必须用 `while` 而非 `if` 复检 predicate"是**同一个不变量的三种表现**。

**徒弟**：那 `remove(command)` 失败会怎样？

**老陈**：说明在你想撤回之前，已经有 Worker 把它取走执行了。那就不该拒绝——任务已经在执行路上，拒绝它会造成"既执行了又报告失败"的双重语义。所以代码是 `if (!isRunning(recheck) && remove(command))`，**两个条件都成立才拒绝**。这个 `&&` 的短路顺序也是有讲究的。

### Transfer——队列类型：一个参数决定整个池的性格

三道门的行为完全被队列类型改写。这是选型时最该看的一张表：

| 队列类型 | 门 2 的行为 | 池的性格 | 适用 | 危险 |
|---|---|---|---|---|
| `SynchronousQueue` | 无存储，**只有正好有空闲线程接手才成功** | 极度倾向扩容 | `newCachedThreadPool`；任务短、要求低延迟 | 配 `max=MAX_VALUE` 时线程无限增长 |
| `ArrayBlockingQueue(n)` | 有界，满则失败 | 平衡：先排队后扩容 | **生产推荐默认** | n 设太大则 max 失效 |
| `LinkedBlockingQueue()` | **永不失败** | 永不扩容，永不拒绝 | 几乎没有 | **堆 OOM 头号元凶** |
| `LinkedBlockingQueue(n)` | 有界，满则失败 | 同 Array，吞吐略高 | 生产可用 | 每元素多一次节点分配 |
| `PriorityBlockingQueue` | 无界 + 按优先级出队 | 永不扩容 | 有优先级诉求时 | 无界；且低优先级任务可能饿死 |
| `DelayedWorkQueue` | 定时池专用 | `max` 完全无效 | `ScheduledThreadPoolExecutor` | 见 [Level 6](#level-6) |

**队列容量的真正含义**：它不是"能存多少任务"，而是**你承诺给用户的延迟上限**。

```text
队列容量 = 1000，平均处理 20ms，并发 8 → 最坏排队延迟 ≈ 1000 / 8 × 20ms = 2.5 秒

如果你的 SLA 是 P99 < 500ms，那么容量 1000 是错的——
它保证的是"任务不丢"，代价是"任务在超时后才被执行"，
这在业务上等价于丢失，还额外浪费了一次计算。

正确做法：队列容量 ← 从 SLA 反推，而不是从"怕丢任务"出发。
```

> 📌 **反直觉但极其重要**：**大队列不是保护，是延迟炸弹。** 一个塞了 10 万任务的队列，意味着队尾的任务要等待极长时间才被执行，那时它的上游调用早已超时重试。你不仅没救活它，还让系统做了双倍的无用功。宁可**快速拒绝并让上游重试或降级**，也不要把任务藏进一个它注定超时的队列里。

L3 讲清了 T 如何被分配。但 T 被 Worker 接手后，如果有人此刻调用 `shutdownNow()`，正在炒的这盘菜会被中断吗？这需要 Worker 自己有一个"我正在忙"的标记。

> 🔴 **口诀**：core→队列→max 三道门；无界队列废掉 max；入队后必复查，队空无人要补人。

---

<a id="level-4"></a>

## 🟢 Level 4：Worker 为什么继承 AQS —— 一把故意不可重入的锁

### 👶 前置知识关卡

- [ ] 为什么 `offer` 成功后还要 `recheck`？
- [ ] `addWorker(null, false)` 的 `null` 解决了什么活性问题？
- [ ] 无界队列为什么会让 `maximumPoolSize` 失效？

> 📌 本层会用到 AQS 的知识。如果"`state` 是资源账本、`tryAcquire` 由子类定义语义"这两句话对你还陌生，建议先读同目录的《🔐 AQS 核心机制深度解析》。这里只用到它最基础的一层。

### Why——`shutdownNow` 要中断线程，但绝不能中断"正在炒的菜"

`shutdownNow()` 的语义是"立刻停止，中断正在执行的任务"。而 `shutdown()` 的语义是"不收新单，把存量做完"。后者也需要中断线程——**因为闲置的 Worker 正阻塞在 `workQueue.take()` 上，不中断它们，它们永远醒不来，池永远无法终止。**

于是产生了一个精确的需求：

```text
shutdown() 需要：中断【闲置的】Worker（让它们从 take() 醒来，发现该下班了）
           禁止：中断【正在执行任务的】Worker（那会打断业务逻辑，破坏 SHUTDOWN 的"做完存量"承诺）

问题：如何判断一个 Worker "正在执行任务"？
```

**徒弟**：加一个 `volatile boolean busy` 字段不行吗？

**老陈**：`busy` 标记会撕裂。看这个时序：

```text
Worker W                          shutdown 线程
──────────────────────────       ──────────────────────────
① getTask() 返回任务 T
                                  ② 读 W.busy == false（还没来得及置位）
③ busy = true
④ 开始执行 T                      ⑤ interrupt(W) ← 打断了正在执行的 T ❌
```

`busy` 的"读"和"设置"之间有窗口。你需要的不是一个标记，而是一个**互斥的、支持 tryLock 语义的门闩**：中断方必须"抢到锁才能中断"，而 Worker 在执行任务期间**持有**这把锁，让中断方抢不到。

### What——Worker：一个 34 行的极简 AQS 子类

```java
// OpenJDK 21，ThreadPoolExecutor.Worker（与 JDK 8 一致）
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;          // 这个 Worker 跑在哪个线程上
    Runnable firstTask;           // 它出生时带的第一个任务（可为 null）
    volatile long completedTasks; // 完成计数

    Worker(Runnable firstTask) {
        setState(-1);             // ★★★ 关键：出生时置 -1，禁止中断
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() { runWorker(this); }

    // 0 = 未锁，1 = 已锁
    protected boolean isHeldExclusively() { return getState() != 0; }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {        // ★ 只允许 0→1
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;                          // ★ 已持有者再来也失败 → 不可重入
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }
}
```

三个设计决策，每一个都有明确目的：

| 设计 | 表面看 | 真实目的 |
|---|---|---|
| `state` 只在 0/1 间跳，`tryAcquire` 只接受 `0→1` | 一把简陋的锁 | **故意不可重入**。如果可重入，任务里再次触发加锁路径就会让"我正在执行"的判定失真 |
| 构造时 `setState(-1)` | 一个奇怪的初值 | -1 时 `tryAcquire` 的 `CAS(0,1)` 必然失败 → **从创建到 `runWorker` 开始的窗口内，谁也中断不了它** |
| 继承 AQS 而非用 `ReentrantLock` | 省一个字段 | 省的是**每个 Worker 一个额外对象**；更重要的是明确表达"这不是通用锁，是执行状态标记" |

> 📌 **`setState(-1)` 防的到底是什么？** Worker 对象创建后、`thread.start()` 前后，它已经被加入 `workers` 集合，因此 `interruptIdleWorkers()` 能遍历到它。如果此时它的 state 是 0，中断方可以 `tryLock` 成功并中断它——但此时它**还没开始跑 `runWorker`**，这个中断会被"预支"，导致它一进入循环就看到中断标志，行为混乱。`runWorker` 的第一件事就是 `w.unlock()`（把 state 从 -1 置为 0），**明确宣告"我准备好了，现在可以被中断了"**。

### How——runWorker：厨师的一生

```java
// 简化自 OpenJDK 21，保留全部关键分支
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();                        // ① state: -1 → 0，允许中断
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {   // ② 取任务
            w.lock();                  // ③ 上锁：宣告"我正在炒菜，别中断我"
            // ④ 如果池已 STOP，确保线程带中断标志；否则确保清除中断标志
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)))
                && !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);          // ⑤ 钩子
                try {
                    task.run();                   // ⑥ ★ 真正执行业务
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex); throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();            // ⑦ 解锁：炒完了，现在可以中断我
            }
        }
        completedAbruptly = false;     // ⑧ 正常退出（getTask 返回 null）
    } finally {
        processWorkerExit(w, completedAbruptly);  // ⑨ 善后
    }
}
```

对应的中断方：

```java
private void interruptIdleWorkers(boolean onlyOne) {
    // ... 持 mainLock
    for (Worker w : workers) {
        Thread t = w.thread;
        if (!t.isInterrupted() && w.tryLock()) {   // ★ 抢得到锁 = 它没在炒菜
            try { t.interrupt(); }
            catch (SecurityException ignore) {}
            finally { w.unlock(); }
        }
        // 抢不到锁 = 它正在炒菜 = 跳过，不中断
        if (onlyOne) break;
    }
}
```

**整个机制浓缩成一句话**：`w.tryLock()` 成功与否，就是"这个厨师现在是否空闲"的**权威且原子**的答案。

慢动作，订单 T 正在被 W1 执行时老板喊 `shutdown()`：

```text
时刻   Worker W1                        shutdown 线程
────  ──────────────────────────      ──────────────────────────
 t1   getTask() 返回 T
 t2   w.lock()  → state 0→1
 t3   task.run() 开始执行 T
 t4                                    shutdown()：ctl → SHUTDOWN
 t5                                    interruptIdleWorkers()
 t6                                    对 W1：tryLock() → CAS(0,1) 失败 ❌
                                       → 跳过，不中断 W1 ✅
                                       （T 得以完整执行完毕）
 t7   T 执行完，w.unlock() → state 1→0
 t8   getTask()：发现 SHUTDOWN 且队列空
      → decrementWorkerCount()，返回 null
 t9   跳出 while，processWorkerExit
 t10  tryTerminate() → 队列空且人数 0 → TIDYING → terminated() → TERMINATED
```

### How 补充——`getTask()`：全池唯一的线程回收出口

线程池的"弹性收缩"没有独立的回收器线程，全部逻辑就在 `getTask()` 的返回值里：

```java
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        // A. 池要关了 且（已 STOP 或 队列已空）→ 明确下班
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);
        // B. 我这个 Worker 是否受"空闲超时"约束？
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // C. 人数超标 或（受约束且上轮已超时）→ 裁掉自己
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c)) return null;
            continue;                                  // CAS 失败就重来
        }
        try {
            Runnable r = timed
                ? workQueue.poll(keepAliveTime, NANOSECONDS)  // 临时厨师：限时等
                : workQueue.take();                            // 核心厨师：无限等
            if (r != null) return r;
            timedOut = true;                                   // 空等一轮
        } catch (InterruptedException retry) {
            timedOut = false;      // ★ 被中断不算超时，重来一轮重新判断状态
        }
    }
}
```

| 判断 | 含义 | 反直觉之处 |
|---|---|---|
| `timed = allowCoreThreadTimeOut \|\| wc > corePoolSize` | 我是否会因空闲被裁 | **谁是"核心线程"是动态判定的**。Worker 对象里没有 `isCore` 字段！第 3 个线程在 `wc=5` 时是"临时工"，在别人退出后 `wc=2` 时又变回"核心工"。**核心/临时是身份的动态计算，不是出生时的标签** |
| `(wc > 1 \|\| workQueue.isEmpty())` | 裁人的安全阀 | 防止裁掉最后一个 Worker 而队列还有任务。**至少留一个人守着非空的传送带** |
| `catch (InterruptedException retry) { timedOut = false; }` | 中断不导致退出 | 中断只是"请重新检查状态"，不是"下班令"。**下班的唯一依据是 ctl 与队列的真实状态，而非中断信号** |

> 📌 **一个常被误解的点**：`allowCoreThreadTimeOut(true)` 之后，池可能收缩到 **0 个线程**。这正是 Level 3 场景 A 里那三行复查存在的原因——两层机制在此闭环。你现在应该能独立推导出：如果没有 `getTask` 的这个特性，`execute` 里的 `workerCountOf(recheck) == 0` 分支就永远不会被触发。**这两处代码是同一个设计决策的两端。**

### Transfer——"用锁的可获取性表达状态"这一模式

Worker 的做法可以抽象成一个通用模式：

```text
不要用 volatile 标记表达"我正在做某事"——标记的读与写之间有窗口。
改用一把锁：做事期间持有它，外部通过 tryLock() 的成败原子地判定状态。
```

适用场景：
- **优雅关闭**：只中断/回收"确实空闲"的资源（连接池、消费者线程组）
- **健康检查**：`tryLock` 失败即"正忙"，无需额外心跳字段
- **幂等守卫**：`tryLock` 成功者执行，失败者跳过，天然去重

不适用场景：
- 需要**统计**有多少个在忙（tryLock 只能问单个，且会有 false negative）
- 持有时间极长（外部探测方会长期得不到答案，退化为"永远在忙"）

L4 讲清了单个 Worker 的生死与中断。但当老板同时对整个厨房喊话时，`shutdown` 与 `shutdownNow` 的差别远比"温柔 / 强硬"复杂——尤其是 `shutdownNow` 的返回值，它把一个沉重的责任交给了调用方。

> 🔴 **口诀**：Worker 是不可重入锁；抢得到锁才敢中断；-1 挡早中断；核心身份靠计算，不靠标签。

---

<a id="level-5"></a>

## 🟢 Level 5：打烊协议 —— shutdown、shutdownNow 与那个没人接的返回值

### 👶 前置知识关卡

- [ ] `w.tryLock()` 失败为什么等价于"这个 Worker 正在执行任务"？
- [ ] Worker 里为什么没有 `isCore` 字段？
- [ ] `getTask()` 中 `InterruptedException` 为什么不导致 Worker 退出？

### Why——为什么"停止线程池"必须有两种，且都不能是 `Thread.stop()`

Java 早在 1.2 就废弃了 `Thread.stop()`，原因是它会在**任意字节码边界**抛出 `ThreadDeath`，可能让对象停留在被修改到一半的状态，且 `synchronized` 块的锁会被释放——把一个不一致的状态暴露给其他线程。因此，Java 世界里**没有安全的"强制终止线程"**，只有**协作式中断**：你发一个信号，任务自己决定何时响应。

这就带来一个必然结论：

```text
线程池的关闭，本质上是一场需要【任务代码配合】的协商。
线程池能做的：
  ① 不再接收新任务             ← 它完全可控
  ② 不再从队列取新任务         ← 它完全可控
  ③ 对正在执行的任务发中断信号  ← 它只能"发信号"
  ④ 让任务真正停下来           ← ★ 它做不到，取决于你的任务代码
```

**如果你的任务代码吞掉 `InterruptedException` 且不重设中断标志，或者压根不检查中断标志，那么 `shutdownNow()` 对它毫无作用。** 这是本层最重要的一句话。

### What——两种打烊的语义对照

| 维度 | `shutdown()` | `shutdownNow()` |
|---|---|---|
| 目标状态 | SHUTDOWN | STOP |
| 拒绝新任务 | ✅ | ✅ |
| 已在队列的任务 | **继续执行完** | **不执行**，从队列排空并返回 |
| 正在执行的任务 | 不中断，让它跑完 | **发中断信号** |
| 中断哪些线程 | 仅 `tryLock` 成功的（闲置的） | **全部**（`interruptWorkers`，不管忙闲） |
| 返回值 | `void` | `List<Runnable>` ← **烫手山芋** |
| 典型用途 | 优雅发布、正常退出 | 超时兜底、紧急止损 |

```java
// 语义对照
public void shutdown() {
    // ... 持 mainLock
    advanceRunState(SHUTDOWN);
    interruptIdleWorkers();      // ★ 只中断闲置的
    onShutdown();
    tryTerminate();
}

public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    // ... 持 mainLock
    advanceRunState(STOP);
    interruptWorkers();          // ★ 中断全部，不做 tryLock 判断
    tasks = drainQueue();        // ★ 把队列里未执行的任务全部倒出来
    tryTerminate();
    return tasks;                // ★ 谁来处理这些任务？
}
```

> 📌 **`shutdownNow()` 返回值是全 JDK 最容易被无视的返回值之一。** 那个 `List<Runnable>` 里装着**已经被接受、承诺过要执行、但最终没有执行**的任务。如果你写 `pool.shutdownNow();` 而不接收返回值，你就静默丢弃了这批任务。在支付、订单、消息投递场景，这就是**数据丢失**。正确姿势至少要落盘或回投消息队列：
>
> ```java
> List<Runnable> pending = pool.shutdownNow();
> if (!pending.isEmpty()) {
>     log.error("shutdownNow 丢弃 {} 个未执行任务，开始补偿落盘", pending.size());
>     pending.forEach(this::persistForReplay);   // 落盘 / 回投 MQ / 告警
> }
> ```

### How——生产级优雅关闭的完整慢动作

单独调用 `shutdown()` 是不够的——它**不阻塞**，方法立刻返回，此时任务可能还在跑。标准的两阶段关闭：

```java
public void gracefulShutdown(ExecutorService pool, Duration graceful, Duration force) {
    pool.shutdown();                              // ① 停止收新单
    try {
        // ② 给存量任务一段宽限期
        if (!pool.awaitTermination(graceful.toMillis(), MILLISECONDS)) {
            log.warn("宽限期 {} 内未完成，转入强制关闭", graceful);
            List<Runnable> dropped = pool.shutdownNow();   // ③ 中断 + 排空队列
            persistOrAlert(dropped);                       // ④ ★ 处理烫手山芋
            // ⑤ 再等一小段：中断信号需要时间被任务响应
            if (!pool.awaitTermination(force.toMillis(), MILLISECONDS)) {
                log.error("强制关闭后仍有任务未响应中断——检查任务是否吞了 InterruptedException");
            }
        }
    } catch (InterruptedException ie) {
        pool.shutdownNow();
        Thread.currentThread().interrupt();       // ⑥ ★ 恢复中断标志
    }
}
```

时序全景：

```text
 t0  业务正常运行：W1 执行 T1，队列有 [T2, T3]，W2 闲置在 take()

 t1  shutdown()
     ├─ ctl 高位 → SHUTDOWN（新的 execute 一律走 reject）
     ├─ interruptIdleWorkers()
     │    ├─ W1: tryLock 失败（正在执行 T1）→ 跳过 ✅
     │    └─ W2: tryLock 成功（闲置）→ interrupt
     └─ 立即返回（不阻塞！）

 t2  W2 从 take() 抛出 InterruptedException → getTask 循环重来
     → 发现 SHUTDOWN 但队列非空 → 继续取 → 拿到 T2 → 执行
     ★ 注意：SHUTDOWN 下 Worker 会继续消费队列，这才是"做完存量"

 t3  W1 完成 T1 → unlock → getTask → 取到 T3 → 执行

 t4  队列空。W1/W2 的 getTask 命中
     runStateAtLeast(SHUTDOWN) && workQueue.isEmpty()
     → decrementWorkerCount()，返回 null → Worker 退出

 t5  最后一个 Worker 退出时 tryTerminate()：
     队列空 ✅ 且 workerCount == 0 ✅
     → CAS ctl → TIDYING → 执行 terminated() 钩子 → TERMINATED
     → termination.signalAll() → awaitTermination 返回 true ✅
```

**徒弟**：既然 `shutdown()` 后 Worker 还会继续消费队列，为什么 t1 还要中断闲置的 W2？

**老陈**：因为 W2 阻塞在 `workQueue.take()` 上——**如果队列恰好是空的，`take()` 会永远阻塞**。它需要一次中断把它从 `take()` 里唤醒，才有机会重新执行 `getTask()` 顶部的状态判断，看到"该下班了"。中断在这里不是"停止"，而是**"请重新检查状态"**。这与 AQS 中"`unpark` 只给一次重试机会，不转移所有权"是完全同构的思想：**信号只触发重新决策，决策依据始终是共享状态本身。**

### How 补充——Spring Boot 场景下最常见的关闭失效

```java
// ❌ 常见错误：@PreDestroy 里只调 shutdown，且没有超时兜底
@PreDestroy
public void destroy() {
    pool.shutdown();   // 不阻塞，方法立刻返回
}
// 后果：JVM 继续走关闭流程，可能在任务跑完前进程就退出了
```

```java
// ✅ Spring 的正确姿势（如果用 ThreadPoolTaskExecutor）
@Bean
public ThreadPoolTaskExecutor bizExecutor() {
    var ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(8);
    ex.setMaxPoolSize(32);
    ex.setQueueCapacity(500);
    ex.setThreadNamePrefix("biz-");
    ex.setWaitForTasksToCompleteOnShutdown(true);  // ★ 等存量做完
    ex.setAwaitTerminationSeconds(30);             // ★ 但最多等 30 秒
    ex.setRejectedExecutionHandler(new CallerRunsPolicy());
    return ex;
}
```

> 📌 **两个参数必须成对设置**。只设 `waitForTasksToCompleteOnShutdown(true)` 而不设 `awaitTerminationSeconds`，等于承诺"无限期等待"——一个卡死的任务会让你的 Pod 一直无法退出，直到被 K8s 的 `terminationGracePeriodSeconds` 强杀（默认 30s），那时反而是最粗暴的终止。**记得让 `awaitTerminationSeconds` 明显小于 K8s 的宽限期**，否则你的优雅关闭逻辑根本没机会跑完。

### Transfer——关闭协议的三条通用原则

1. **两阶段关闭是底线**：先温柔（停止接收 + 等待），再强硬（中断 + 排空），且**强硬阶段之后仍要再等一次**——中断只是信号，响应需要时间。
2. **未完成的工作必须有归宿**：`shutdownNow()` 的返回值、消息的 unack、事务的未提交，都必须显式处理。**静默丢弃是最坏的选择，因为它不产生任何告警。**
3. **关闭的时间预算必须小于外部的强杀预算**：K8s `terminationGracePeriodSeconds`、网关超时、注册中心摘除延迟——你的优雅关闭必须能在这些窗口内完成，否则等于没写。

L5 完成了单个 `ThreadPoolExecutor` 的全生命周期。但生产系统里从来不止一种池：定时任务、并行计算、异步编排各有各的池，而它们中有一个是**全 JVM 共享的**。

> 🔴 **口诀**：shutdown 做完存量，shutdownNow 中断排空；中断只是"请重查"；返回值不接就是丢数据。

---

<a id="level-6"></a>

## 🟢 Level 6：厨房家族 —— 定时池、分治池与那个全局共享的默认池

### 👶 前置知识关卡

- [ ] `shutdown()` 为什么不阻塞？必须配合什么方法？
- [ ] 为什么 `shutdown()` 也要中断闲置 Worker？
- [ ] 优雅关闭的时间预算为什么必须小于 K8s 宽限期？

> 从这里开始引入新角色：**周期任务 P**（定时池）、**分治任务 F**（ForkJoinPool）、**流元素 E**（响应式）。他们和主线的订单 T **不是同一条因果链**，只是共用同一座厨房的舞台。全景复盘时会把线索重新对齐。

### Why——为什么不能"一个池打天下"

**徒弟**：既然 `ThreadPoolExecutor` 这么完备，为什么还要 `ScheduledThreadPoolExecutor` 和 `ForkJoinPool`？

**老陈**：因为它们要解决的**任务形态**根本不同。三种任务形态，三种队列哲学：

| 任务形态 | 关键诉求 | 普通 TPE 为什么不够 |
|---|---|---|
| 提交即执行（订单 T） | 吞吐 + 有界延迟 | ✅ 正合适 |
| **定时/周期**（P） | 按时间排序出队，且周期任务要**自我重投** | FIFO 队列无法按 delay 排序 |
| **分治递归**（F） | 子任务由工作线程自己产生，且**父任务等子任务** | 单一共享队列成为热点；且父等子会**耗尽线程** |

第三行是最深的：如果用普通线程池跑分治任务，`池大小 = 4`，一个任务 fork 出 4 个子任务后 `join` 等待——**4 个线程全在 join，没人执行子任务，死锁。** 这不是配置问题，是模型不匹配。

### What——三种池的定位对照

| | `ThreadPoolExecutor` | `ScheduledThreadPoolExecutor` | `ForkJoinPool` |
|---|---|---|---|
| 队列 | 你指定 | `DelayedWorkQueue`（堆，按时间排序） | **每线程一个双端队列** + 全局提交队列 |
| 任务窃取 | ❌ | ❌ | ✅ work-stealing |
| `maximumPoolSize` | 有效（配有界队列） | **完全无效**（见下） | 不适用（用 parallelism） |
| 适用 | I/O 密集、通用异步 | 定时、心跳、重试 | CPU 密集分治、并行流 |
| 阻塞任务 | 可以（但要调参） | 可以（但会挤占定时精度） | **不建议**（会饿死其他任务） |

### How 1——定时池：为什么 `maximumPoolSize` 是个摆设

```java
// OpenJDK 21，ScheduledThreadPoolExecutor 的构造
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,         // ← max 传的是 MAX_VALUE
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());                 // ← 无界队列
}
```

回到 Level 3 的三道门：`DelayedWorkQueue` 是**无界**的，`offer` 永不失败，因此**第三道门永远不会被触达**。`Integer.MAX_VALUE` 这个 max 值只是形式上的占位。

**结论**：`ScheduledThreadPoolExecutor` 的并发度**完全由 `corePoolSize` 决定**。

这带来一个经典事故：

```text
scheduler = Executors.newSingleThreadScheduledExecutor();   // corePoolSize = 1
scheduler.scheduleAtFixedRate(健康检查任务, 0, 1, SECONDS);   // 每秒一次
scheduler.scheduleAtFixedRate(数据同步任务, 0, 5, SECONDS);   // 每 5 秒一次

某天数据同步任务因下游超时卡了 3 分钟
  → 唯一的线程被占用
  → 健康检查任务 3 分钟没执行
  → 注册中心认为节点死亡，摘除流量
  → 一个无关的下游抖动，导致本节点被摘 ❌
```

更隐蔽的是**未捕获异常导致周期任务静默消失**：

```java
scheduler.scheduleAtFixedRate(() -> {
    doSomething();          // 如果这里抛出未捕获的 RuntimeException
}, 0, 1, SECONDS);
// → 这个周期任务【永久停止】，不再执行，且默认不打印任何日志 ❌❌❌
```

> 📌 **这是我职业生涯里见过最多的"任务莫名其妙不跑了"的根因。** `scheduleAtFixedRate` 的契约是：任务抛出异常则该周期任务被取消，后续不再执行。而异常被封装在 `ScheduledFutureTask` 里，你不调 `future.get()` 就永远看不到它。

**唯一正确的写法**：把整个任务体包在 try-catch 里，一个异常都不许漏。

```java
scheduler.scheduleAtFixedRate(() -> {
    try {
        doSomething();
    } catch (Throwable t) {                    // ★ 必须是 Throwable，不是 Exception
        log.error("周期任务异常，已吞掉以保证后续调度", t);
    }
}, 0, 1, SECONDS);
```

**定时池的三条铁律**：
1. `corePoolSize` 必须 > 1，且按"最慢任务的最坏耗时"估算，绝不用 `newSingleThreadScheduledExecutor` 跑多个业务任务
2. **每个周期任务体必须自带 try-catch(Throwable)**
3. 关键定时任务（分布式调度、跨节点协调）应交给专业调度框架（XXL-Job、Quartz 集群、K8s CronJob），**JVM 内的定时池不具备节点故障转移能力**

`scheduleAtFixedRate` vs `scheduleWithFixedDelay` 的区别也必须说清：

```text
scheduleAtFixedRate(task, 0, 5, SECONDS)     ← 每 5 秒【开始】一次
  任务耗时 2s：  [0-2s]执行  [5-7s]执行  [10-12s]执行   ✅ 节奏稳定
  任务耗时 8s：  [0-8s]执行  [8-16s]立即执行 ...        ⚠️ 追赶模式，无重叠但无间隙

scheduleWithFixedDelay(task, 0, 5, SECONDS)  ← 上次【结束】后再等 5 秒
  任务耗时 2s：  [0-2s]执行  [7-9s]执行  [14-16s]执行   ✅ 保证间隔
  任务耗时 8s：  [0-8s]执行  [13-21s]执行                ✅ 永远有 5s 喘息
```

选择依据：**需要固定节奏（如指标采集）用 `atFixedRate`；需要保证下游喘息（如轮询远程 API）用 `withFixedDelay`。** 后者在防止雪崩上更安全。

### How 2——ForkJoinPool：work-stealing 与那个共享的 commonPool

分治任务 F 的执行模型完全不同：

```text
普通线程池：            ForkJoinPool：
                       
  提交队列(共享)          每个 Worker 一个 Deque（双端队列）
      │                  
  ┌───┴───┐             W1: [F1a][F1b]  ←push/pop 自己这头（LIFO，缓存友好）
  ▼   ▼   ▼                      ▲
 W1  W2  W3                      │ steal（从另一头，FIFO，偷最老最大的任务）
 所有人抢同一个队列 →热点   W2: [空] ──┘
                          
                       W2 没活干时，随机挑一个同伴，从它队列【另一头】偷
```

三个设计要点：

| 设计 | 原因 |
|---|---|
| 自己这头 LIFO | 最近 push 的子任务，数据大概率还在 L1/L2 缓存里 |
| 偷取从另一头 FIFO | 队列头部是最"老"的任务，通常粒度最大，偷一次能换来更多工作，减少偷取频率 |
| 每线程独立队列 | 消除单一队列的争用热点 |

**`ForkJoinPool.commonPool()` 是本层最大的坑。** 它被这些地方共享：

```java
// 全部默认跑在同一个 commonPool 上：
list.parallelStream().map(...).collect(...);          // 并行流
CompletableFuture.supplyAsync(() -> ...);             // 不传 Executor 的异步
Arrays.parallelSort(arr);                             // 并行排序
```

它的默认并行度（OpenJDK 21 源码）：

```java
pc = Math.max(1, Runtime.getRuntime().availableProcessors() - 1);
```

**注意是 CPU 数减 1**（因为提交任务的调用线程也会参与执行）。在一台 4 核容器里，`commonPool` 只有 **3 个** 工作线程。

于是：

```text
你的报表模块：list.parallelStream().forEach(item -> httpClient.call(item));
              ← 阻塞式 HTTP，占满 commonPool 的 3 个线程，每个卡 2 秒

同时，另一个完全无关的模块：
              CompletableFuture.supplyAsync(() -> quickCalc())
              ← 排在 commonPool 队列后面，被迫等 2 秒 ❌

两个毫无关系的业务，通过一个你从未显式创建过的池，产生了强耦合。
```

> 📌 **`commonPool` 是为 CPU 密集型分治设计的，它的线程数按 CPU 核数配，前提假设是"每个线程都在满负荷计算"。往里放任何阻塞操作（HTTP、JDBC、文件 I/O），都是在浪费一个本该满转的 CPU 槽位，并连累全 JVM 所有使用者。**

正确做法：

```java
// ❌ 阻塞任务扔给 commonPool
CompletableFuture.supplyAsync(() -> restTemplate.getForObject(url, String.class));

// ✅ 显式传入你自己的、有界的、可监控的池
private static final Executor IO_POOL = new ThreadPoolExecutor(
        16, 64, 60L, SECONDS,
        new ArrayBlockingQueue<>(200),
        new NamedThreadFactory("io-pool"),
        new ThreadPoolExecutor.CallerRunsPolicy());

CompletableFuture.supplyAsync(() -> restTemplate.getForObject(url, String.class), IO_POOL);
```

**关于 `CompletableFuture` 一个鲜为人知的细节**（OpenJDK 21 源码）：

```java
private static final boolean USE_COMMON_POOL = (ForkJoinPool.getCommonPoolParallelism() > 1);
private static final Executor ASYNC_POOL = USE_COMMON_POOL
        ? ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```

**如果 JVM 只有 1 个可用 CPU**（单核容器、或 cgroup 限制到 1 核），`commonPool` 并行度为 1，`USE_COMMON_POOL` 为 false，`CompletableFuture` 会退化成 **`ThreadPerTaskExecutor`——每个任务新建一个线程**！这意味着你的代码在多核机器上跑得好好的，一旦部署到单核容器，就变成了无限制的 `new Thread()`。

> 📌 这是一个真实存在的、由**环境差异触发的行为突变**。它再次证明本文的核心主张：**不要依赖默认值，显式指定你的 Executor。**

`asyncXxx` 方法的调度语义也要说清：

| 写法 | 在哪个线程执行 |
|---|---|
| `thenApply(fn)` | **不确定**：如果前一阶段已完成，在**调用 `thenApply` 的线程**执行；否则在**完成前一阶段的线程**执行 |
| `thenApplyAsync(fn)` | 默认 `ASYNC_POOL`（通常是 commonPool） |
| `thenApplyAsync(fn, myPool)` | `myPool` ✅ **生产唯一推荐** |

> 📌 `thenApply` 的"不确定"是回调地狱的常见来源：一段本该跑在业务线程池的代码，可能实际跑在 Netty 的 I/O 线程上，你在里面做阻塞操作就会卡死整个 EventLoop。**在框架代码里链式编排时，永远显式指定 Executor。**

### Transfer——池的隔离原则（舱壁模式）

```text
❌ 一个池打天下
   订单 + 支付 + 报表 + 日志 → 同一个池
   报表慢查询占满线程 → 支付超时 → 雪崩

✅ 按【失败域】隔离，而不是按业务模块隔离
   核心交易池   (core=16, queue=100,  AbortPolicy)      ← 宁可拒绝，不可延迟
   下游调用池   (core=32, queue=500,  CallerRunsPolicy) ← 可以慢，不可以丢
   报表分析池   (core=4,  queue=1000, DiscardOldest)    ← 慢无所谓，可丢旧
   定时任务池   (core=8,  自带异常兜底)                  ← 独立，不与业务共池
```

**隔离的判据不是"业务是否相关"，而是"它们是否应该共享失败"。** 报表跑挂了不该影响支付——那它们就必须在不同的池里。

L6 讲的三种池，都建立在同一个前提上：**线程是稀缺资源，所以必须池化**。JDK 21 正式交付的虚拟线程，直接把这个前提删掉了。

> 🔴 **口诀**：定时池 max 是摆设、异常吞任务；commonPool 全局共享莫放阻塞；池按失败域隔离。

---

<a id="level-7"></a>

## 🟢 Level 7：虚拟线程与响应式 —— 当"线程"不再稀缺

### 👶 前置知识关卡

- [ ] `ScheduledThreadPoolExecutor` 的 `maximumPoolSize` 为什么无效？
- [ ] `commonPool` 的默认并行度为什么是 `CPU - 1`？
- [ ] `thenApply` 与 `thenApplyAsync` 的执行线程差异？

> 本层引入流元素 **E**（响应式）。它与主线订单 T 不是同一条因果链。但本层结尾会把 T、E 和虚拟线程重新对齐到同一个容量模型上。

### Why——池化的三个前提，虚拟线程删掉了两个

回到 Level 1。线程池存在的理由是：

```text
前提 ①：创建线程昂贵（系统调用 + 内核栈 + ~1MB 用户栈）
前提 ②：线程数量有上限（几千个就吃不消）
前提 ③：需要容量阀门（并发度、排队量、拒绝语义）
```

虚拟线程（JEP 444，JDK 21 正式特性）改变了什么：

| 前提 | 平台线程 | 虚拟线程 | 结论 |
|---|---|---|---|
| ① 创建成本 | 系统调用 + MB 级栈 | **堆上的对象**，栈按需增长的 chunk | ❌ **前提失效** |
| ② 数量上限 | 数千 | **数百万**（受堆大小约束） | ❌ **前提失效** |
| ③ 容量阀门 | 靠池大小 | **虚拟线程没有提供任何阀门** | ✅ **前提仍然成立，且更迫切** |

**这推出一个极其重要、也极其反直觉的结论：**

> 🚨 **不要池化虚拟线程。** 池化的目的是复用昂贵资源；虚拟线程廉价到"用完就扔"比"归还池子"更划算。给虚拟线程建池，是把它最大的优势（无限量、即用即弃）亲手砍掉，还额外引入了 ThreadLocal 泄漏等池化固有问题。

OpenJDK 官方给的用法是**每任务一个虚拟线程**：

```java
// ✅ 正确：不是池，是"每任务一线程"的执行器
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> { handleRequest(i); return null; }));
}   // try-with-resources 自动 shutdown + awaitTermination

// ❌ 错误：把虚拟线程塞进固定池，等于给跑车装拖拉机引擎
var vtFactory = Thread.ofVirtual().factory();
new ThreadPoolExecutor(10, 10, ..., vtFactory);   // 并发度被锁死在 10，毫无意义
```

> 📌 `newVirtualThreadPerTaskExecutor()` 这个名字里没有 "Pool"，是**故意的**。它是一个 `ExecutorService`，但它不池化任何东西——每次 `submit` 都创建一个全新的虚拟线程。

### What——虚拟线程的执行模型与那个"藏起来的"载体池

```text
    100 万个虚拟线程（heap 上的对象，各自带一个可增长的栈 chunk）
      VT1  VT2  VT3  VT4 ... VT1000000
        │    │    │    │
        └────┴──┬─┴────┘  mount / unmount（挂载与卸载）
                ▼
    ┌───────────────────────────────┐
    │  载体线程池（ForkJoinPool）      │  ← 这才是真正的 OS 线程
    │  默认 parallelism =            │     数量 ≈ CPU 核数
    │    availableProcessors()       │
    │  CT1  CT2  CT3  CT4            │
    └───────────────────────────────┘
                ▼
           OS 调度器 / CPU 核

    关键：当 VT 遇到阻塞 I/O 时，它【卸载】自己，
          把 CT 让给别的 VT。CT 永远不空等。
```

载体池的配置（OpenJDK 21 `VirtualThread.createDefaultScheduler()`）：

```java
parallelism = Integer.getInteger("jdk.virtualThreadScheduler.parallelism",
                                 Runtime.getRuntime().availableProcessors());
maxPoolSize = Integer.getInteger("jdk.virtualThreadScheduler.maxPoolSize",
                                 Math.max(parallelism, 256));
minRunnable = Math.max(parallelism / 2, 1);
return new ForkJoinPool(parallelism, factory, handler, asyncMode,
                        0, maxPoolSize, minRunnable, pool -> true, 30, SECONDS);
```

> 📌 载体池是一个 `ForkJoinPool`，但它**不是** `commonPool`——两者是独立的。`maxPoolSize` 默认 256 是为了应对**临时阻塞（pinning）时的补偿**，不是常规并行度。**不要随便调这些系统属性**，除非你有 JFR 数据支撑。

### How——pinning：JDK 21 的坑与 JDK 24 的修复

**这是全文版本敏感度最高的部分，请对照你的部署版本阅读。**

**在 JDK 21～23**，虚拟线程在以下情况会 **pin**（钉住载体线程，无法卸载）：

1. 在 `synchronized` 块/方法内部阻塞
2. 执行 native 方法 / FFM 调用时阻塞

为什么？因为 JDK 21 的 HotSpot 把 monitor 的归属记录在**载体线程**上。虚拟线程若在持有 monitor 时卸载，monitor 的所有权记录就会错乱，所以 JVM 干脆禁止卸载。

```java
// JDK 21 下的定时炸弹
public synchronized String fetch() {       // synchronized
    return httpClient.send(req, ofString()).body();   // 阻塞 I/O → VT 无法卸载
}
// 100 万个 VT 都走这里 → 载体池（≈CPU核数个线程）被全部钉死 → 吞吐归零，甚至饿死
```

**JDK 21 的应对**：把 `synchronized` 换成 `ReentrantLock`（AQS 的 park 是 Loom 感知的，会正确卸载）：

```java
private final ReentrantLock lock = new ReentrantLock();
public String fetch() {
    lock.lock();
    try { return httpClient.send(req, ofString()).body(); }   // ✅ 阻塞时 VT 正常卸载
    finally { lock.unlock(); }
}
```

**但是——JDK 24 起，这条建议已经过期。** JEP 491（Synchronize Virtual Threads without Pinning，JDK 24 正式交付）把 monitor 的归属从 carrier 迁移到了**虚拟线程本身**，虚拟线程现在可以在持有 monitor 的情况下卸载与重新挂载。

| 场景 | JDK 21–23 | JDK 24+ |
|---|---|---|
| `synchronized` 内阻塞 | ❌ pin 住 carrier | ✅ 正常卸载 |
| `Object.wait()` | ❌ pin | ✅ 正常卸载 |
| native / FFM 调用中阻塞 | ❌ pin | ❌ **仍然 pin**（Java-native 边界的固有限制） |
| 类初始化器中阻塞 | ❌ pin | ⚠️ 仍可能 pin，JFR 会记录 |
| 诊断手段 | `-Djdk.tracePinnedThreads`（JDK 24 已移除） | JFR `jdk.VirtualThreadPinned` 事件（覆盖场景更广） |

> 📌 **给架构评审的版本决策**：如果你在 JDK 21 LTS 上，`synchronized` → `ReentrantLock` 的改造对**热路径**是必要的；如果你已经或计划升到 JDK 24+，这项改造的收益大幅下降，**不要为它做大规模重构**。无论哪个版本，都应该用 JFR 的 `jdk.VirtualThreadPinned` 事件**实测**，而不是凭猜测改代码。

**最关键的一点，用虚拟线程也要用信号量：**

```text
❌ 错误推理：虚拟线程可以开 100 万个 → 我可以同时发 100 万个 DB 查询
✅ 现实：数据库连接池只有 100 条；下游 API 限流 300 QPS
        虚拟线程解决的是"等待的载体成本"，
        它【不能凭空制造第 101 条连接】。
```

```java
// ✅ 虚拟线程 + 信号量：线程模型与资源模型分离
private static final Semaphore DB_PERMITS = new Semaphore(80);

void handle(Request r) {                      // 跑在虚拟线程上，可以有 100 万个
    if (!DB_PERMITS.tryAcquire(150, MILLISECONDS)) {
        throw new ServiceBusyException("db saturated");   // 快速失败，可解释
    }
    try { queryDb(r); }
    finally { DB_PERMITS.release(); }
}
```

> 📌 **这是虚拟线程时代最重要的一句话**：**线程数不再是容量阀门，但你仍然需要容量阀门。** 阀门从"线程池大小"搬家到了"Semaphore / 连接池 / 限流器"。删掉了阀门却以为自己变快了，只是把崩溃点从本服务推给了下游。

### How 补充——结构化并发：一个必须用"时序图而非代码"来讲的特性

结构化并发解决的是"一组相关任务的生命周期管理"。**但它是本文唯一一个我拒绝给出可编译代码示例的特性**，理由在下面的版本时间线里。

**先讲不变的设计思想**（这部分从 JDK 19 到今天没变过）：

```text
非结构化（CompletableFuture / 裸 submit）：
  ┌─ 主任务 ──────────────────────────▶ return（提前返回了！）
  ├─ 子任务 A ──────────────▶ 完成
  └─ 子任务 B ─────────────────────────────────────▶ 还在跑，没人管它
     ★ 主任务已返回，B 成了孤儿：无人 join、失败无人知、取消无人做

结构化：
  ┌─ scope 开始
  │   ├─ 子任务 A ────────▶ 失败！
  │   └─ 子任务 B ──╳ 被自动取消（短路）
  └─ scope 结束（作用域退出）
     ★ 不变量：离开代码块时，所有子任务必然已终结（完成或取消）
       —— 与 try-with-resources 管理资源是同一个思想
```

它相对 `CompletableFuture.allOf()` 的三个硬收益：**一个失败自动取消其余**（不浪费下游资源）、**异常传播路径清晰**（不用在 `CompletionException` 里剥洋葱）、**线程转储能显示父子层级**。

**现在讲为什么不给代码——这是一条罕见的、七次预览仍在改的 API 时间线**：

| JDK | JEP | API 形态 | 关键变化 |
|---|---|---|---|
| 19–20 | 428/437 | 孵化器（`jdk.incubator.concurrent`） | 最初形态 |
| 21–24 | 453 / **462** / 480 / 499 | `new StructuredTaskScope.ShutdownOnFailure()` | **构造函数 + 子类继承**；需 `join()` 后手动 `throwIfFailed()`。JDK 21 的 JEP 453 首次转为预览（`fork` 返回 `Subtask` 而非 `Future`），22/23/24 三次原样再预览 |
| **25** | **505** | **`StructuredTaskScope.open(Joiner)`** | **实质重构**：构造函数→静态工厂；`ShutdownOnFailure`/`ShutdownOnSuccess` **子类被删除**，改用组合式 `Joiner`；`join()` 直接返回结果或抛 `FailedException`（消除了"忘记调 throwIfFailed"的隐患）；`joinUntil` 改为在配置中声明超时 |
| **26** | **525** | 同上，细节微调 | `Joiner.onTimeout()` 回调；`allSuccessfulOrThrow()` 返回 `List` 而非 `Stream`；`anySuccessfulResultOrThrow()` **更名**为 `anySuccessfulOrThrow()`；`open()` 的配置参数改为 `UnaryOperator` |
| **27** | **533** | 同上，异常语义改造 | **第七次预览**。`StructuredTaskScope` 与 `Joiner` 新增第三个类型参数 `R_X`（`join()` 可抛出的异常类型）；三个标准 joiner 改抛 **`ExecutionException`** 而非预览专用的 `FailedException`；`onTimeout()` 改名为 `timeout()`；`awaitAll()` 被移除 |

> ⚠️ **写在最显眼处**：结构化并发从 JDK 19 孵化至今**仍未定稿**，JDK 27 是它的**第七次预览**。JDK 25 的 JEP 505 是一次**实质性重构**（构造函数→静态工厂、子类→`Joiner`），JDK 27 的 JEP 533 又改了**异常类型**——这意味着即使是"catch 哪个异常"这种最基本的代码形状，也还在变。
>
> **因此本文不提供可编译示例。** 任何你在网上看到的结构化并发代码，都必须先确认它写于哪个 JDK 版本。如果你要用：
> 1. `java -version` 确认版本
> 2. 直接查**该版本**的 `StructuredTaskScope` Javadoc（不是搜索引擎结果）
> 3. 加 `--enable-preview` 编译，并接受**下次升级 JDK 时代码可能编译不过**
>
> 这正是本文"没有一个写法可以直接抄进生产"原则的最典型应用场景。**一个 API 在七次预览后仍在改名，就不该出现在任何以"可复制"为目的的文档里。**

📌 **给架构评审的判断**：结构化并发的**思想**（生命周期绑定词法作用域）现在就可以用来指导设计——比如审视你的 `CompletableFuture` 编排里有没有"主流程返回了但子任务还在跑"的孤儿任务。但它的**代码**，等定稿再引入生产。

### How 补充——响应式：Reactor 的调度模型与线程池的关系

流元素 E 走的是完全不同的路：**不阻塞线程，而是回调驱动 + 背压**。

```java
Flux.fromIterable(orderIds)
    .flatMap(id -> webClient.get().uri("/order/{id}", id).retrieve()
                            .bodyToMono(Order.class), 32)   // ★ 32 = 并发度阀门
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe(this::handle);
```

Reactor 的调度器与线程池的对应关系：

| Scheduler | 底层 | 默认大小（Reactor 3.x） | 用途 |
|---|---|---|---|
| `Schedulers.parallel()` | 固定 worker 池 | `DEFAULT_POOL_SIZE` = CPU 核数 | **CPU 密集**，绝不能放阻塞操作 |
| `Schedulers.boundedElastic()` | 弹性有界 | `DEFAULT_BOUNDED_ELASTIC_SIZE` = **10 × CPU**；每线程队列上限 `DEFAULT_BOUNDED_ELASTIC_QUEUESIZE` = **100000** | **专门用来兜阻塞调用**（JDBC、老式 HTTP 客户端） |
| `Schedulers.immediate()` | 不切换 | — | 保持在当前线程 |
| `Schedulers.single()` | 单线程 | 1 | 需要串行化的场景 |

#### Why——为什么必须有 `boundedElastic`，而不是把 `parallel()` 调大？

**徒弟**：既然阻塞任务也要线程，为什么不干脆把 `parallel()` 的池子调大，允许在里面阻塞？何必多一个调度器？

**老陈**：因为那样会**毁掉 `parallel()` 唯一的定价锚点**。

```text
parallel() 的容量推导链：
  worker 数 = CPU 核数
        ↓  为什么这个数字是【可推导】的？
  因为前提是"每个 worker 始终在满负荷计算"
        ↓
  在这个前提下，N 个核最多同时算 N 件事，多开线程只增加上下文切换
        ↓
  ★ 所以"CPU 核数"是一个【有物理依据的锚点】，不是拍脑袋的经验值

一旦允许在 parallel() 里阻塞，这条链的第一环就断了：
  worker 阻塞时不消耗 CPU，于是"核数"不再是合理上限
        ↓
  正确的线程数变成 N × (1 + 等待时间/计算时间)
        ↓
  而"等待时间"取决于下游 P99 —— 它是【运行时才知道、且持续波动】的量
        ↓
  ★ 结论：你【再也无法静态推算】parallel() 该开多大
        ↓
  最终只能靠猜，猜小了整条响应式管道停摆，猜大了失去 parallel() 存在的意义
```

所以 Reactor 的选择不是"多写一个调度器"，而是**用两个调度器把两种不可通约的容量模型隔开**：

| | `parallel()` | `boundedElastic()` |
|---|---|---|
| 容量锚点 | **CPU 核数**（物理上限，可静态推导） | **下游能承受的并发数**（业务上限，需实测） |
| 定价依据 | 硬件 | 下游 P99 与容量 |
| 混用后果 | 锚点失效，容量无法推导 | — |

> 📌 **这与 Level 7 结尾的"线程模型与资源模型必须拆开"是同一条因果链的两个投影，请对照阅读**：
>
> - 在**虚拟线程**语境下，它表现为："线程数不再是阀门，阀门要搬到 Semaphore"
> - 在**响应式**语境下，它表现为："CPU 密集与阻塞 I/O 必须用不同调度器，因为它们的容量锚点不同"
> - 在**平台线程池**语境下，它表现为 Level 6 的"按失败域隔离"——报表池与交易池不能共用
>
> 三处说的都是同一件事：**当两类工作的容量由不同的物理量决定时，把它们塞进同一个容量参数，就等于放弃了对其中至少一类的容量控制。** 这是本文最核心的一条判断力，也是它在三种并发模型下的统一形态。

> ⚠️ 这些默认值可通过系统属性 `reactor.schedulers.defaultPoolSize`、`reactor.schedulers.defaultBoundedElasticSize`、`reactor.schedulers.defaultBoundedElasticQueueSize` 覆盖，且**随 Reactor 版本演进**。上表基于 Reactor 3.x 官方 Javadoc，生产请核对你依赖的具体版本。

**响应式最容易犯的错**：在 `parallel()` 调度器上做阻塞调用。

```java
// ❌ 灾难：parallel() 只有 CPU 核数个线程，全部阻塞 = 整个响应式管道停摆
Flux.range(1, 1000)
    .publishOn(Schedulers.parallel())
    .map(i -> jdbcTemplate.queryForObject(...))    // 阻塞！
    .subscribe();

// ✅ 阻塞调用必须隔离到 boundedElastic
Flux.range(1, 1000)
    .flatMap(i -> Mono.fromCallable(() -> jdbcTemplate.queryForObject(...))
                      .subscribeOn(Schedulers.boundedElastic()))
    .subscribe();
```

**三种并发模型的容量阀门对照——这是本层的收口**：

| | 线程池（T） | 虚拟线程 | 响应式（E） |
|---|---|---|---|
| 并发的载体 | 平台线程 | 虚拟线程 | 事件循环 + 回调 |
| **并发度阀门** | `maximumPoolSize` | **Semaphore**（必须自己加） | `flatMap` 的 `concurrency` 参数 |
| **排队阀门** | `workQueue` 容量 | **无内建**（需自己加队列/限流） | 背压请求量（`request(n)`） |
| **拒绝语义** | `RejectedExecutionHandler` | 自己实现（如 `tryAcquire` 失败） | `onBackpressureDrop/Buffer/Latest` |
| 编程模型 | 命令式，直观 | **命令式，直观**（最大优势） | 声明式，学习曲线陡 |
| 调试栈 | 完整 | **完整**（虚拟线程栈可读） | 破碎（异步栈难追） |
| 适用 | 通用 | I/O 密集、高并发请求 | 流式、需要精细背压 |

> 📌 **给架构评审的选型判断**：如果你在 JDK 21+ 且主要痛点是"I/O 阻塞导致线程不够用"，**虚拟线程通常是比响应式更好的选择**——它以近乎为零的心智负担换来同等的扩展性，而响应式要求整条链路（包括驱动、中间件）全部非阻塞，一处阻塞就前功尽弃。响应式的独特价值在**精细的背压控制**和**流式数据处理**，不在"提高并发数"。**如果你选响应式的理由只是"性能好"，那很可能选错了。**

### Transfer——把"线程模型"与"资源模型"彻底拆开

这是本文最想留给你的一句话：

```text
线程模型（谁来执行）        资源模型（能同时用多少）
─────────────────────      ──────────────────────────
平台线程池                  ← 历史上，这两者被【同一个参数】耦合
虚拟线程                       "线程池大小 = 20" 同时表达了
响应式事件循环                  "20 个执行者" 和 "20 的并发上限"

虚拟线程把它们强行拆开了：
  执行者可以有 100 万个 → 你必须【显式】声明资源上限
  
  CPU 边界      → Executor 并行度 / parallel() 池
  DB 边界       → 连接池大小 + Semaphore
  下游 API 边界 → Semaphore + 熔断 + 限流
  跨服务背压    → 队列容量 + 超时 + 降级
```

> 🔴 **口诀**：虚拟线程别池化；阀门从池大小搬到信号量；JDK21 防 pin 换锁，JDK24 已修复；响应式选它是为背压，不是为快。

---

<a id="level-7-5"></a>

## 🔬 Level 7.5：硬件视角补充 —— 从对象字段到缓存行

> 📌 **这是一节可跳过的补充。** 它不在主线因果链上，服务于两类读者：正在做**极限性能调优**的人，以及在面试里被追问"再往下一层是什么"的人。三个话题的共同点是：**它们都属于"你不应该主动优化，但必须知道它存在"的范畴。**
>
> ⚠️ 本节不提供任何性能数字。所有结论都以"如何观测"而非"能快多少"的形式给出——因为这些效应高度依赖 CPU 型号、缓存行大小、JVM 版本与对象布局，任何脱离实测的量化都是误导。

### 1. `mainLock`：线程池内部那把被忽视的全局锁

前面 Level 4 讲了 Worker 自己的锁，但 `ThreadPoolExecutor` 还有**另一把锁**，它管的是整个 workers 集合：

```java
private final ReentrantLock mainLock = new ReentrantLock();
private final HashSet<Worker> workers = new HashSet<>();   // ★ 非线程安全，全靠 mainLock 保护
```

谁会抢它？

| 操作 | 为什么要抢 mainLock | 频率 |
|---|---|---|
| `addWorker` | 往 `workers` 里加元素 | 每次创建线程 |
| `processWorkerExit` | 从 `workers` 里删元素 | 每次线程退出 |
| `interruptIdleWorkers` | **遍历整个 workers 集合** | 每次 shutdown / setCorePoolSize |
| `getPoolSize` / `getActiveCount` | 读取集合状态 | ★ **每次监控采集** |
| `tryTerminate` | 检查是否可终止 | 每次 worker 退出 |

**因果链**：

```text
mainLock 是【整个池共享的一把锁】
        ↓
addWorker / 线程退出 / 监控采集 都要串行地抢它
        ↓
池越大（corePoolSize=500），interruptIdleWorkers 遍历越久，持锁时间越长
        ↓
若同时有高频的线程创建/销毁（keepAlive 很短 + 流量抖动）
        ↓
★ mainLock 本身成为争用热点
```

**但请注意这个关键限定**：`execute()` 的**正常路径不碰 mainLock**——任务入队走的是 `workQueue.offer()`（并发队列，无需 mainLock），只有"需要创建新线程"时才会抢。所以：

> 📌 **稳态运行的线程池，mainLock 几乎不争用**。它成为问题只在三种场景：① 线程频繁创建销毁（keepAlive 过短 + 流量锯齿）；② 池极大且频繁调用 `shutdown`/`setCorePoolSize`；③ **监控采集过于频繁**——`getPoolSize()`、`getActiveCount()` 每次都要抢 mainLock，如果你每秒采集几十次且池很大，监控本身就成了干扰源。

#### 缝合：这正是 ForkJoinPool 要用 work-stealing 的深层原因

现在回头看 Level 6 讲的 work-stealing，它的意图就完整了：

```text
ThreadPoolExecutor 的共享结构：
  · 一个共享 workQueue  → 所有 Worker 抢同一个队列的锁/CAS 点
  · 一个共享 workers 集合 → 由 mainLock 串行保护
        ↓
  规模越大，共享结构的争用越显著

ForkJoinPool 的回答：
  · 每个 Worker 一个独立 Deque → 常态下【零争用】（只有自己 push/pop）
  · 只有偷取时才跨线程访问   → 把争用从"常态"降级为"例外"
        ↓
  ★ work-stealing 的本质，是把 Level 4 那种"共享结构 + 全局锁"的模型，
    换成"局部结构 + 偶发协调"的模型
```

这与 AQS 里"CLH 把全局资源竞争拆成相邻节点的局部交班责任"是**同一个思想的第三次出现**：

| 层次 | 全局争用的形态 | 局部化的方案 |
|---|---|---|
| AQS | 所有线程 CAS 同一个 `state` | CLH：只盯前驱节点 |
| ThreadPoolExecutor | 所有 Worker 抢 `workQueue` / `mainLock` | —（它接受了这个代价，换取语义简单） |
| ForkJoinPool | 同上 | work-stealing：每线程一个 Deque |

> 📌 **`ThreadPoolExecutor` 没有做局部化，不是设计缺陷，是取舍**：它要保证 FIFO 语义、精确的容量控制和可预测的拒绝行为，这些都需要一个**全局可见的队列**。ForkJoinPool 放弃了全局 FIFO（分治任务不关心顺序），才换来了局部化的自由。**看到一个设计"没有采用更快的方案"时，先找它放弃了什么语义。**

### 2. 伪共享：`ctl` 与相邻字段的缓存行竞争

`ThreadPoolExecutor` 的字段声明顺序大致是：

```java
private final AtomicInteger ctl;        // ★ 超高频 CAS（每次 execute / 每次 worker 状态变化）
private final BlockingQueue<Runnable> workQueue;
private final ReentrantLock mainLock;
private final HashSet<Worker> workers;
private long completedTaskCount;        // ★ 普通 long，worker 退出时在 mainLock 内累加
private volatile int corePoolSize;      // ★ 高频【读】
private volatile int maximumPoolSize;   // ★ 高频【读】
```

**因果链**：

```text
CPU 缓存的最小传输单位是【缓存行】（x86 通常 64 字节）
        ↓
JVM 对象布局中，相邻声明的字段【很可能】落在同一条缓存行里
        ↓
核 A 对 ctl 做 CAS → 该缓存行在核 A 上进入 Modified 状态
        ↓
核 B 只是想【读】corePoolSize（execute 路径上每次都读）
        ↓
★ 但它俩在同一条缓存行 → 核 B 的读会因缓存一致性协议而失效/重新获取
        ↓
两个逻辑上毫不相干的字段，因物理相邻而互相拖累 = 伪共享（false sharing）
```

> ⚠️ **必须强调三个限定，否则这段话会被误用**：
>
> 1. **JVM 会重排字段顺序**（`-XX:FieldsAllocationStyle`、对象头对齐等），源码里的声明顺序**不等于**内存布局顺序。想知道真实布局，用 **JOL**（Java Object Layout）：`ClassLayout.parseClass(ThreadPoolExecutor.class).toPrintable()`。
> 2. `AtomicInteger` 是**引用字段**，`ctl` 引用指向的那个 `AtomicInteger` 对象在堆的别处，其 `value` 字段与 `ThreadPoolExecutor` 的其他字段**不一定**相邻。真正的争用点在 `AtomicInteger.value` 所在的缓存行。
> 3. **JDK 没有给 `ctl` 加 `@Contended`**，这本身就是一个信号：Doug Lea 显然评估过，认为在典型负载下不值得为此付出内存填充的代价。

**这个"评估过"不是我的臆测——因为同一批作者在别处确实主动加了。** 两个 JDK 21 的实证反例：

```java
// java.util.concurrent.atomic.Striped64（LongAdder / DoubleAdder 的基类）
@jdk.internal.vm.annotation.Contended static final class Cell {
    volatile long value;
    ...
}
// 类注释原文："...(via @Contended) to reduce cache contention."

// java.util.concurrent.ConcurrentHashMap
/** A padded cell for distributing counts. Adapted from LongAdder and Striped64. */
@jdk.internal.vm.annotation.Contended static final class CounterCell {
    volatile long value;
}
```

**为什么这两处值得、`ctl` 不值得？** 因果很清晰：

| | `Striped64.Cell` / `CounterCell` | `ThreadPoolExecutor.ctl` |
|---|---|---|
| 设计意图 | **故意让多个核各写各的槽位** | 全池共用一个计数 |
| 争用形态 | N 个核**并行写 N 个相邻对象** → 伪共享是**主要瓶颈** | 单个热点字段 → 真共享（true sharing）本来就无法避免 |
| 加 padding 的收益 | 消除唯一的伪竞争 → 收益直接 | 只能隔开邻居，**`ctl` 自身的 CAS 争用一点没少** |
| 代价 | 每个 Cell 多占一条缓存行，但 Cell 数量 ≈ CPU 核数，可控 | 每个池对象都变大，而池可能有几十上百个 |

> 📌 **一句话总结这个判断**：`@Contended` 治的是**伪**共享（逻辑无关的字段挤在一条缓存行）。`Striped64` 的分槽设计本就是为了把竞争打散，剩下的唯一障碍就是伪共享，所以加它收益极大；而 `ctl` 面对的是**真**共享——所有线程本来就要争这一个值，隔开邻居并不能减少这份争用。**分不清真假共享，就会把 `@Contended` 当万能药到处撒。**
>
> 📌 **所以本节的正确用法不是"去优化它"，而是"把它列入排查清单"**：如果 JMH 或异步 profiler 显示 `execute()` 路径上**CAS 重试率异常高**、或 `perf c2c` 报告显著的缓存行争用，那么字段布局是**候选原因之一**——但在此之前，先怀疑更常见的原因（任务本身太慢、队列争用、GC）。

**观测手段**（只给命令，不给结论）：

```bash
# 1. 看真实对象布局（需引入 JOL 依赖）
java -jar jol-cli.jar internals java.util.concurrent.ThreadPoolExecutor

# 2. Linux 上定位缓存行争用（需 perf 权限）
perf c2c record -a -- sleep 10 && perf c2c report

# 3. 异步 profiler 看 CAS 热点
./profiler.sh -e cycles -d 30 -f cas.html <pid>
```

### 3. 虚拟线程的栈：`StackChunk` 与年轻代压力

这是虚拟线程从"能用"到"敢大规模用"之间**最大的未知数**，值得比坑 4 里那一句话更多的篇幅。

```text
平台线程的栈：            虚拟线程的栈：
  · 在【线程栈内存】上      · 在【Java 堆】上，以 StackChunk 对象形式存在
  · 由 OS 管理              · 由 GC 管理
  · 不参与 GC               · ★ 参与 GC
  · 固定大小（-Xss）         · 按需增长/收缩
```

**因果链**：

```text
虚拟线程 unmount（卸载）时，需要把当前栈帧从 carrier 的栈上【复制到堆】
        ↓
mount（重新挂载）时，再从堆复制回 carrier 栈
        ↓
★ 每次阻塞/唤醒都可能伴随一次栈拷贝与 StackChunk 对象的分配
        ↓
100 万个虚拟线程 + 高频阻塞唤醒
        ↓
年轻代分配速率显著上升；StackChunk 是特殊对象，GC 需要扫描其中的引用
        ↓
可能的表现：YGC 频率上升、GC 扫描成本增加
```

> ⚠️ **诚实边界**：StackChunk 的 GC 处理有专门优化（如惰性复制、只扫描变化部分），HotSpot 团队持续在改进这块，且不同 GC（G1 / ZGC / Shenandoah）表现不同。**我不给任何量化结论**——这正是需要你实测的地方。

**必须实测的观测清单**（在你决定把百万级虚拟线程放上生产前）：

```bash
# 1. 年轻代分配速率与 GC 频率
java -Xlog:gc*,gc+heap=debug:file=gc.log:time,uptime -jar app.jar
#    重点看：YGC 间隔是否随虚拟线程数上升而缩短

# 2. JFR 采样对象分配（找出 StackChunk 占比）
jcmd <pid> JFR.start settings=profile filename=vt.jfr duration=120s
jfr print --events jdk.ObjectAllocationSample vt.jfr | grep -i stackchunk
#    也可用 jdk.ObjectAllocationInNewTLAB 交叉验证

# 3. 堆内存构成
jcmd <pid> GC.heap_info
jcmd <pid> GC.class_histogram | head -30      # 看 StackChunk 实例数与占用

# 4. 对照实验（最有说服力）
#    同一负载下，分别用平台线程池与虚拟线程跑，对比：
#    · YGC 次数 / 总 GC 暂停时间
#    · 堆占用峰值
#    · 端到端 P99 / P999
```

> 📌 **给架构评审的判断**：虚拟线程把"栈内存"从**堆外的、不参与 GC 的资源**，变成了**堆内的、参与 GC 的资源**。这是一次**成本转移**，不是成本消失。在 I/O 密集且每个虚拟线程栈很浅的场景（典型 Web 请求），这笔交易非常划算；但如果你的任务**调用栈很深**（深层框架嵌套、递归）**且并发极高**，这笔账需要实测才能算清。
>
> **这也是为什么"虚拟线程要配合 Semaphore"这条建议有第二层价值**：限制并发不只是保护下游，也是在**控制同时存活的 StackChunk 数量**。

> 🔴 **口诀**：mainLock 只在建/毁线程时热，监控别刷太勤；伪共享先测再改，JDK 不加 @Contended 自有道理；虚拟线程把栈搬进了堆，省的是内核、花的是 GC。

---

<a id="self-check"></a>

## 🧪 合书自测：一页时序图

```text
【主线：订单 T 的一生】core=2, max=4, queue=ArrayBlockingQueue(2)

T0  池刚创建：ctl = ctlOf(RUNNING, 0)，workers 为空，队列空
    ★ 线程不是构造时创建的，是第一次 execute 时懒创建的

T1  execute(T1)：门 1 命中（workerCount 0 < core 2）
    → addWorker(T1, true)：CAS ctl 人数 0→1（一次 CAS 同时校验状态）
    → new Worker(T1)：setState(-1) 禁止早中断
    → thread.start() → runWorker → w.unlock() 置 0，允许中断
    → 执行 T1

T2  execute(T2)：门 1 命中（1 < 2）→ 开 W2 → 执行 T2

T3  execute(T3)：门 1 不通过（2 == 2）
    → 门 2：isRunning ✅ && queue.offer ✅ → T3 入队
    → ★ recheck：仍 RUNNING 且 workerCount=2≠0 → 什么都不做，正确
    execute(T4)：同样入队。队列 [T3, T4] 已满

T4  execute(T5)：门 1 ❌，门 2 ❌（队列满）
    → 门 3：addWorker(T5, false)，人数 2→3 < max 4 ✅
    → 开临时厨师 W3 执行 T5
    ★ 注意 T5 比 T3/T4 后提交，却先执行 —— 线程池不保证 FIFO

T5  execute(T6) → W4（人数 4 = max）
    execute(T7)：三道门全关 → reject(T7) → 保安处置
    ★ 用 AbortPolicy 则抛异常；CallerRunsPolicy 则由提交线程自己执行

T6  高峰过去。W3/W4 的 getTask()：
    timed = (wc=4 > core=2) = true → poll(60s) 超时返回 null
    → timedOut=true → 下一轮命中裁员条件 → CAS 人数-- → 返回 null → 退出
    ★ 收缩到 core=2。W1/W2 的 timed=false → take() 无限等，不被裁

T7  shutdown()：ctl → SHUTDOWN
    interruptIdleWorkers：W1 正执行(tryLock 失败)→跳过；W2 闲置→中断
    W2 从 take() 醒来 → getTask 重判 → 队列还有存量 → 继续消费
    ★ SHUTDOWN = 不收新单但做完存量

T8  队列空 + 所有 Worker 退出 → tryTerminate()
    → CAS TIDYING → terminated() 钩子 → TERMINATED
    → termination.signalAll() → awaitTermination 返回 true

【支线 P：定时池】corePoolSize=1，两个周期任务
  P1 每 1s，P2 每 5s。P2 卡住 3 分钟 → P1 三分钟没跑 → 健康检查失败
  若 P1 抛未捕获异常 → P1 永久静默消失 ❌

【支线 F：ForkJoinPool.commonPool()】4 核容器 → parallelism = 3
  报表模块 parallelStream 里做阻塞 HTTP → 占满 3 个槽
  无关模块的 CompletableFuture.supplyAsync 被迫排队 ❌

【支线 VT：虚拟线程】
  10 万 VT 各自 handleRequest → 载体池仅 ≈CPU 核数个 CT
  JDK21：若在 synchronized 内阻塞 → pin 住 CT → 吞吐归零
  JDK24+：JEP 491 已修复此类 pinning，仅剩 native/FFM 场景
  ★ 但无论哪个版本：DB 连接仍只有 80 条 → 必须 Semaphore
```

| 自测题 | 必须答出的不变量 |
|---|---|
| `maximumPoolSize` 什么时候生效？ | 仅当队列 `offer` **失败**时。无界队列下永不生效。 |
| 为什么 `offer` 成功后要 recheck？ | 队列有任务 ≠ 有人消费。状态可能已变，或 Worker 可能已全部退出。 |
| Worker 怎么知道自己是"核心"线程？ | **不知道**。没有 `isCore` 字段，靠 `getTask()` 里 `wc > corePoolSize` 动态计算。 |
| `shutdown()` 为什么要中断闲置 Worker？ | 它们阻塞在 `take()` 上。中断是"请重新检查状态"，不是"停止"。 |
| `shutdownNow()` 的返回值必须怎么处理？ | 必须落盘/回投/告警。静默丢弃 = 数据丢失且无告警。 |
| 虚拟线程该池化吗？ | **绝不**。用 `newVirtualThreadPerTaskExecutor()`；容量阀门改用 Semaphore。 |
| 定时任务为什么会"莫名不跑了"？ | 未捕获异常导致周期任务被取消，且默认不打日志。必须自带 try-catch(Throwable)。 |

---

<a id="pitfalls"></a>

## ⚠️ 线程池的坑与细节：从代码味道到生产后果

### 坑 1：用 `Executors` 工厂方法（阿里规约明令禁止）

```java
// ❌ 全部有雷
Executors.newFixedThreadPool(n);      // 无界 LinkedBlockingQueue → 堆 OOM
Executors.newSingleThreadExecutor();  // 同上
Executors.newCachedThreadPool();      // max = Integer.MAX_VALUE → 线程数爆炸
Executors.newScheduledThreadPool(n);  // 无界 DelayedWorkQueue → 堆 OOM
Executors.newWorkStealingPool();      // ★ 性质完全不同的坑，见下
```

| 工厂方法 | 底层实现 | 埋的雷 | 失败形态 |
|---|---|---|---|
| `newFixedThreadPool(n)` | TPE | 无界 `LinkedBlockingQueue` | 堆 OOM |
| `newSingleThreadExecutor()` | TPE | 同上 | 堆 OOM |
| `newCachedThreadPool()` | TPE | `max = Integer.MAX_VALUE` | 线程数爆炸 |
| `newScheduledThreadPool(n)` | TPE | 无界 `DelayedWorkQueue` | 堆 OOM |
| **`newWorkStealingPool()`** | **ForkJoinPool** | **工作线程全是守护线程** | ⚠️ **任务静默丢失** |

#### `newWorkStealingPool()`：一个性质完全不同的坑

前四个的失败形态都是"**资源耗尽**"——它们至少会以 OOM、GC 频繁、线程数告警等方式**发出信号**。第五个不一样：

```java
// OpenJDK 21，Executors.newWorkStealingPool() 的真身
public static ExecutorService newWorkStealingPool() {
    return new ForkJoinPool(
        Runtime.getRuntime().availableProcessors(),        // ★ 注意：不减 1
        ForkJoinPool.defaultForkJoinWorkerThreadFactory, null, true);
}
```

它返回的是 `ForkJoinPool`，而 `ForkJoinPool` 的 Javadoc 有一句决定性的声明：

> *"All worker threads are initialized with `Thread#isDaemon` set `true`."*

**守护线程不阻止 JVM 退出。** 因果链就此闭合：

```text
工作线程是 daemon
      ↓
main 方法返回后，JVM 发现【只剩守护线程】
      ↓
JVM 直接退出，不等它们
      ↓
★ 队列里没跑完的任务、正在跑到一半的任务，全部随进程消失
      ↓
没有异常、没有日志、没有退出码异常 —— 完全静默
```

**🔬 亲手验证**（`java ThreadPoolLabs 4`），两组代码**只差 `Executors` 那一行**：

```text
Step 1 · 确认线程类型：
  ForkJoinPool-1-worker-3      isDaemon=true   ← ★ FJP 工作线程全是守护线程
  pool-1-thread-1              isDaemon=false  ← 对照：TPE 默认【非】守护线程

Step 3 · 独立进程实跑（各提交 3 个耗时 1 秒的任务，然后 main 直接返回）：

  A 组 · newWorkStealingPool()
    > main 方法结束（没有 shutdown / awaitTermination）
    ★ 进程直接退出，三个『完成』一个都没打印 —— 任务被静默杀死

  B 组 · newFixedThreadPool(3)
    > main 方法结束（没有 shutdown / awaitTermination）
    >   完成 1 / 完成 3 / 完成 2
    ★ 非守护线程阻止了 JVM 退出，任务全部跑完；
      但也因此，忘记 shutdown() 会让进程【永远挂不掉】（实测被 timeout 强杀）
```

> 📌 **这两个极端恰好构成一组完美的反面教材**：`newWorkStealingPool` **丢数据**（进程秒退），`newFixedThreadPool` **挂进程**（永不退出）。**两个默认值，两种相反的错误，没有一个能直接用。** 它们共同指向同一条铁律：**任何线程池都必须显式 `shutdown()` + `awaitTermination()`**，这是 Level 5 打烊协议存在的全部意义。
>
> 这个坑最容易命中的场景是**小工具类程序、批处理脚本、单元测试、`main` 方法里的一次性任务**——恰恰是那些"就跑一下、不需要那么讲究"的地方。而它与 Level 6 的呼应也在这里：**`newWorkStealingPool` 借用的是 ForkJoinPool 体系**，所以它继承的不只是 work-stealing 的优点，还有 daemon 线程、以及"为 CPU 密集分治设计、不适合阻塞任务"的全部前提。

**修正**：永远手写 `new ThreadPoolExecutor(...)`，七个参数全部显式：

```java
new ThreadPoolExecutor(
    8, 32, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(200),                 // ★ 必须有界
    new NamedThreadFactory("order-biz"),           // ★ 必须命名，否则 dump 里全是 pool-1-thread-N
    new ThreadPoolExecutor.CallerRunsPolicy());    // ★ 必须显式选择拒绝语义
```

**线程命名的价值**：线上 jstack 出来 500 个 `pool-3-thread-127`，你无法知道它属于哪个业务。命名后可以直接 `grep 'order-biz'` 定位。**这是零成本的可观测性投资。**

### 坑 2：拒绝策略选错，把拒绝当技术细节

#### 先回答一个没人问但很关键的问题：JDK 为什么只内置这四种？

**徒弟**：最常用的"降级到 MQ""落盘重放""限流告警"，JDK 一个都没内置。是懒得写吗？

**老陈**：不是懒，是**它写不了**。看 `reject()` 在哪里执行：

```java
final void reject(Runnable command) {
    handler.rejectedExecution(command, this);   // ★ 就在【调用 execute() 的那个线程】上，同步执行
}
```

`rejectedExecution` 没有开新线程，没有异步化，它就是 `execute()` 调用栈的一部分。因果链就此闭合：

```text
reject 发生在调用者线程 + 同步执行
        ↓
拒绝逻辑的耗时，会【直接累加到 execute() 的耗时上】
        ↓
而 execute() 通常在你的业务主链路上（Web 请求线程、上游消费者线程）
        ↓
如果内置策略里有 I/O（发 MQ、写磁盘、调告警接口）：
  · 下游 MQ 抖动 → execute() 卡住 → 提交线程被拖垮
  · 而此刻【线程池已经饱和】，正是系统最脆弱的时刻
  · 结果：拒绝逻辑本身成为压垮系统的最后一根稻草
        ↓
所以 JDK 的四个内置策略【必须】满足：同步、无 I/O、无阻塞、不可能失败
```

对照验证这个约束——四个策略确实全部满足：

| 策略 | 它做的唯一一件事 | 耗时量级 |
|---|---|---|
| `AbortPolicy` | `throw new RejectedExecutionException()` | 纳秒级 |
| `DiscardPolicy` | 空方法体，什么都不做 | 零 |
| `DiscardOldestPolicy` | `queue.poll()` 后重试一次 `execute` | 纳秒级 |
| `CallerRunsPolicy` | `r.run()` 直接跑 | ⚠️ **唯一的例外** |

> 📌 **`CallerRunsPolicy` 是这条规则里唯一"耗时不可控"的策略——而这恰恰是它的设计意图**：它故意让提交者付出执行任务的时间代价，以此产生背压。所以它不是违反了约束，而是**把"阻塞提交者"从副作用变成了主功能**。这也解释了为什么它在决策卡 1（Web 线程提交）是禁忌、在决策卡 3（批处理主线程提交）是最优解——同一个机制，价值完全取决于"谁是提交者"。

**结论**：JDK 划的边界是"内置策略只做 O(1) 的本地决策；任何涉及 I/O、网络、持久化的复杂降级，必须由你自定义，并由你自己负责它的超时与失败"。**这不是 JDK 的缺失，是它在替你守住 `execute()` 的性能契约。** 明白这一点，你写自定义 handler 时就会知道：**里面的每一行 I/O，都是在给主链路加延迟**——所以正确的写法是异步投递 + 快速失败，而不是同步等 MQ 确认。

#### 四种内置策略的选择

| 策略 | 行为 | 适用 | 危险 |
|---|---|---|---|
| `AbortPolicy`（默认） | 抛 `RejectedExecutionException` | 核心链路，**宁可失败也要让调用方知道** | 调用方不 catch → 500 |
| `CallerRunsPolicy` | **提交线程自己执行** | 需要天然背压 | ⚠️ 若提交者是 Web 容器线程，会**阻塞接收新请求**——这既是特性也是风险 |
| `DiscardPolicy` | 静默丢弃 | 几乎没有 | **无声无息丢数据**，最危险 |
| `DiscardOldestPolicy` | 丢队头最老的 | 只关心最新值（如行情推送） | 丢的是等最久的任务 |

> 📌 **`CallerRunsPolicy` 的双面性**：它常被推荐为"最安全"，因为它提供背压——提交者被迫自己干活，自然就慢下来了。但如果提交者是 Tomcat 的请求线程，它执行任务期间**无法接收新请求**，压力从线程池传导到了容器。这可能正是你要的（真背压），也可能是灾难（整个服务卡住）。**必须显式想清楚"谁是提交者"。**

**自定义拒绝策略往往才是正解**：

```java
new RejectedExecutionHandler() {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        REJECT_COUNTER.increment();                    // ① 打点，让监控看得见
        log.warn("池饱和: active={} queue={} completed={}",
                 e.getActiveCount(), e.getQueue().size(), e.getCompletedTaskCount());
        if (!fallbackToMq(r)) {                        // ② 降级：转异步/落盘
            throw new RejectedExecutionException("saturated, and MQ fallback failed");
        }
    }
};
```

### 坑 3：任务里吞掉异常，导致问题永久隐身

```java
// ❌ submit() 的异常被封装在 Future 里，不 get() 就永远看不见
pool.submit(() -> { throw new RuntimeException("boom"); });   // 静默！

// execute() 的异常会走线程的 UncaughtExceptionHandler（至少能打印）
pool.execute(() -> { throw new RuntimeException("boom"); });  // 会打印
```

> 🔗 **源码级解释**（`FutureTask.run()` 里 `catch (Throwable ex) { setException(ex); }` 那一行）与五种提交方式的完整对比，见 [📮 提交方式对比](#submission)。
>
> 📌 **`submit` 与 `execute` 的异常语义差异是"任务静默失败"的头号原因。** `submit` 把 `Runnable` 包装成 `FutureTask`，异常被捕获并存入 Future 的状态里，只有调用 `get()` 才会以 `ExecutionException` 抛出。如果你 `submit` 后从不 `get`，异常就永远消失了。

#### 🔬 亲手验证：同一个异常，一个打印、一个静默

配套代码 `java ThreadPoolLabs 3`。**同一个 `Runnable` 对象**，只改提交方式：

```java
Runnable boom = () -> { throw new IllegalStateException("boom!"); };
pool.execute(boom);   // ①
pool.submit(boom);    // ②  完全相同的任务
```

**真实运行输出**：

```text
  ① 用 execute() 提交一个抛异常的任务：
  🔔 UncaughtExceptionHandler 捕获到: boom!
     ↑ 异常走了 UncaughtExceptionHandler，至少你能看见它

  ② 用 submit() 提交【完全相同】的任务：
     ↑ ……什么都没有。控制台一片安静。

  ③ 只有当你调用 future.get() 时，它才浮出水面：
     ✅ f.get() 抛出 ExecutionException，cause = java.lang.IllegalStateException: boom!

  ④ 真实项目里的典型写法 —— 提交一批任务后从不 get()：
     提交了 5 个必定失败的任务，控制台输出的错误信息条数：0
     ★ 这 5 个失败【永久消失】，监控看不到、日志查不到。
```

> 📌 第 ④ 步才是真正的杀伤力所在：**批量 `submit` 后不 `get`，是极其常见的写法**（"我只是想让它异步跑，不关心返回值"）。此时任何异常都不会留下痕迹——不是"日志级别不够"，而是**根本没有产生日志**。你的监控、告警、APM 全部失明。

**双保险**：

```java
// ① ThreadFactory 里设置 UncaughtExceptionHandler
new NamedThreadFactory("biz").setUncaughtExceptionHandler(
    (t, e) -> log.error("线程 {} 未捕获异常", t.getName(), e));

// ② 覆写 afterExecute，同时覆盖 execute 和 submit 两条路径
protected void afterExecute(Runnable r, Throwable t) {
    super.afterExecute(r, t);
    if (t == null && r instanceof Future<?> f && f.isDone()) {
        try { f.get(); }
        catch (CancellationException ce) { /* 正常取消，忽略 */ }
        catch (ExecutionException ee)    { t = ee.getCause(); }
        catch (InterruptedException ie)  { Thread.currentThread().interrupt(); }
    }
    if (t != null) log.error("任务执行异常", t);
}
```

### 坑 4：ThreadLocal 未清理，导致数据串号

```java
// ❌ 线程复用 = ThreadLocal 也被复用
private static final ThreadLocal<UserContext> CTX = new ThreadLocal<>();

pool.execute(() -> {
    CTX.set(new UserContext(currentUserId));
    doWork();
    // 忘了 remove() → 下一个任务复用这个线程，读到【上一个用户的身份】❌❌❌
});
```

**这是最严重的一类生产事故**：不是崩溃，是**用户 A 看到了用户 B 的数据**。而且它只在线程被复用时出现，测试环境低并发下往往复现不了。

```java
// ✅ 必须 finally remove
pool.execute(() -> {
    CTX.set(new UserContext(uid));
    try { doWork(); } finally { CTX.remove(); }
});
```

> 📌 虚拟线程下这个问题**天然消失**（每任务一个新线程，不复用），这是虚拟线程被低估的一个安全性优势。但如果你在 JDK 21+ 用虚拟线程，应优先考虑 `ScopedValue`（预览特性）而非 `ThreadLocal`——它不可变、有明确作用域，且对大量线程的内存开销远低于 ThreadLocal。
>
> ⚠️ 但要注意另一笔账：虚拟线程虽然消除了 ThreadLocal 串号，却把**栈内存搬进了堆**。百万级虚拟线程场景下，`ThreadLocal` 副本数与 `StackChunk` 会一起放大堆压力。这部分的实测方法见 [Level 7.5](#level-7-5)。

#### 坑 4-B：`InheritableThreadLocal` 在线程池里**天然失效**——这不是清理问题

上面讲的是"忘记 `remove()`"，属于**使用不当**。但还有一个**性质完全不同**的问题：`InheritableThreadLocal` 在线程池场景下，**语义本身就是错的**，写得再规范也没用。

**因果链在于"拷贝发生的时刻"**：

```java
// Thread 构造时（且仅此一次）从父线程拷贝
if (parent.inheritableThreadLocals != null)
    this.inheritableThreadLocals =
        ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```

```text
InheritableThreadLocal 的设计前提：
    每次异步 = new Thread() = 拷贝一次父线程上下文  ✅ 语义成立

线程池的核心价值：
    ★ 复用 Worker，不为每个任务创建线程
        ↓
Worker 只在【第一次被创建】时拷贝了一次上下文
        ↓
那一刻的"父线程"是【碰巧第一个触发扩容的那个请求线程】
        ↓
★ 之后所有任务拿到的，都是那个【不相关的、早已结束的请求】的上下文
```

```java
static final InheritableThreadLocal<String> TRACE = new InheritableThreadLocal<>();

// 请求 A（第一个请求，触发 Worker 创建）
TRACE.set("trace-A");
pool.submit(() -> log.info(TRACE.get()));   // "trace-A" ✅ 碰巧对了

// 请求 B（复用同一个 Worker，不再创建线程 → 不再拷贝）
TRACE.set("trace-B");
pool.submit(() -> log.info(TRACE.get()));   // ★ 仍是 "trace-A" ❌❌❌
```

> 📌 **这就是它比坑 4 更危险的地方**：坑 4 的串号至少还是"上一个任务的值"，有一定时序关联；而这里拿到的是**池初始化那一刻某个随机请求的值**，可能已经过去了几小时。而且它**在低并发测试中几乎必然"看起来是对的"**——因为第一个请求恰好就是拷贝源。

**解法：`TransmittableThreadLocal`（TTL，阿里开源）**

TTL 的思路不是改拷贝时机，而是**把上下文捕获从"线程创建时"挪到"任务提交时"**：

```text
InheritableThreadLocal：  拷贝时机 = Thread 构造   → 与任务无关 ❌
TransmittableThreadLocal：捕获时机 = 任务【提交】时 → 与任务一一对应 ✅
                          执行前 replay，执行后 restore
```

```java
// 方式一：包装任务
static final TransmittableThreadLocal<String> TRACE = new TransmittableThreadLocal<>();
TRACE.set("trace-B");
pool.submit(TtlRunnable.get(() -> log.info(TRACE.get())));   // ✅ "trace-B"

// 方式二（推荐）：包装线程池，业务代码零侵入
ExecutorService pool = TtlExecutors.getTtlExecutorService(rawPool);
pool.submit(() -> log.info(TRACE.get()));                    // ✅ 自动传递

// 方式三：Java Agent，连包装都不用（对既有代码改动最小）
// -javaagent:transmittable-thread-local-2.x.x.jar
```

| 方案 | 拷贝/捕获时机 | 池化场景 | 典型用途 |
|---|---|---|---|
| `ThreadLocal` | 无传递 | ✅ 可用（须 `remove()`） | 线程内缓存 |
| `InheritableThreadLocal` | **线程创建时** | ❌ **语义失效** | 仅适合 `new Thread()` |
| `TransmittableThreadLocal` | **任务提交时** | ✅ 正确 | TraceId、租户、灰度标、`SecurityContext` |
| `ScopedValue`（JDK 21 预览） | 显式作用域绑定 | ✅（配合虚拟线程） | 新项目、不可变上下文 |

> 📌 **这也解释了一个常见困惑**："为什么我用了 Spring 的 `@Async`，MDC 里的 TraceId 就丢了？" 因为 MDC 底层多为 `ThreadLocal`/`InheritableThreadLocal`，跨线程池不会自动传。正解是 `TaskDecorator`（见决策卡 5）或 TTL，**而不是换个 `ThreadLocal` 子类**。
>
> ⚠️ TTL 也有边界：它在**提交时**捕获快照，若提交后、执行前又修改了上下文，任务看到的仍是提交那一刻的值——这通常正是你要的，但要清楚它是快照语义。

### 坑 5：线程池大小按公式硬套

流传最广的公式：

```text
CPU 密集：N + 1
I/O 密集：2N   或   N × (1 + 平均等待时间 / 平均计算时间)
```

**这些公式的价值是提供起点和思考框架，不是给出答案。** 理由：

1. 真实任务混合了 CPU 与 I/O，"等待/计算比"不是常数，随下游 P99 波动
2. **容器里 `Runtime.availableProcessors()` 可能不等于你的 CPU 配额**。JDK 8u191+ 和 JDK 10+ 已能感知 cgroup limits，但如果只设了 `cpu.shares`（K8s 的 `requests`）而没设 `cpu.cfs_quota`（`limits`），JVM 看到的仍是宿主机核数
3. 多个线程池共享同一份 CPU 配额，各自按"CPU 数"配置会**严重超配**
4. cgroup CPU throttling 会让实际可用算力远低于名义核数

```bash
# 先搞清楚你到底有多少 CPU
$ cat /sys/fs/cgroup/cpu.max              # cgroup v2: "200000 100000" = 2 核
$ cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us # cgroup v1
$ jcmd <pid> VM.flags | grep ActiveProcessorCount
$ jshell -> Runtime.getRuntime().availableProcessors()

# 关键：检查是否被 throttle（nr_throttled 持续增长 = CPU 不够）
$ cat /sys/fs/cgroup/cpu.stat | grep throttled
```

**正确方法**：公式给起点 → 压测找拐点 → 监控持续校准。**拐点的判据是"吞吐不再上升而 P99 开始恶化的那一点"**，不是"CPU 跑满"。

### 坑 6：把线程池当分布式调度用

```java
// ❌ 单机定时池做业务调度
scheduler.scheduleAtFixedRate(this::结算对账, 0, 1, HOURS);
// 部署 3 个副本 → 对账跑 3 次 → 数据重复 ❌
// 该副本挂了 → 对账永远不跑，且无人知道 ❌
```

**JVM 内的定时池没有：** 分布式互斥、故障转移、执行历史、失败重试、手动触发、执行时长告警。这些都需要专业调度框架（XXL-Job、Elastic-Job、Quartz 集群模式、K8s CronJob）。**JVM 定时池只适合进程内的技术性任务**：缓存刷新、连接保活、本地指标上报。

### 坑 7：用 `getActiveCount()` 等观测方法做控制逻辑

```java
// ❌ 拿近似值做正确性判断
if (pool.getActiveCount() < pool.getMaximumPoolSize()) {
    pool.execute(task);   // 检查与执行之间状态早已变化
}
```

`getActiveCount()` 遍历 workers 统计 `isLocked()`，是**并发下的估算快照**。`getQueue().size()` 同理。

**修正**：它们只用于**监控和告警**。需要控制并发时用 `Semaphore`；需要判断能否提交时直接 `execute` 并处理 `RejectedExecutionException`——**让拒绝策略成为唯一的容量决策点。**

### 坑 8：`prestartAllCoreThreads` 与冷启动毛刺

线程池是**懒创建**的：构造完成时 `workerCount = 0`。这意味着服务刚启动、流量刚打进来时，前 `corePoolSize` 个请求每个都要承担一次线程创建的开销。

```java
pool.prestartAllCoreThreads();   // 预热：提前把核心线程全部创建好
```

在对冷启动 P99 敏感的场景（金丝雀发布刚放量、Serverless 冷启动），这一行能消掉一个明显的毛刺。**代价是常驻线程的内存占用提前发生**——如果你的服务有大量低频线程池，预热全部反而是浪费。

### 坑 9：动态调参不生效，或调了反而更糟

`ThreadPoolExecutor` 支持运行时调参（美团技术团队的动态线程池方案即基于此）：

```java
pool.setCorePoolSize(16);
pool.setMaximumPoolSize(64);
pool.setKeepAliveTime(30, SECONDS);
```

**但队列容量改不了**——`ArrayBlockingQueue` 的 `capacity` 是 `final`。想动态调队列，必须自定义一个把 `capacity` 改成非 final 的队列实现。

> 📌 **调参的顺序有讲究**：调**大** max 时先调 max 再调 core；调**小** 时先调 core 再调 max。否则中间态可能出现 `core > max`，`setCorePoolSize` 内部虽有保护，但会产生不必要的线程创建/回收抖动。

**更重要的是**：`setCorePoolSize` 调小后，多余的线程**不会立即销毁**，而是在下次 `getTask()` 时才被回收。所以调参效果有延迟，别调完立刻看指标就下结论。

### 坑 10：`invokeAll` 的隐藏阻塞与超时陷阱

```java
// ❌ invokeAll 会阻塞直到【全部】完成，没有超时
List<Future<R>> futures = pool.invokeAll(tasks);   // 一个慢任务拖垮全部

// ✅ 带超时版本
List<Future<R>> futures = pool.invokeAll(tasks, 3, SECONDS);
// ★ 但注意：超时后【未完成的任务会被 cancel】，
//   已完成的 Future 正常返回，未完成的 get() 抛 CancellationException
for (Future<R> f : futures) {
    try { results.add(f.get()); }
    catch (CancellationException ce) { results.add(fallbackValue()); }
}
```

### 坑 11：从火焰图 park 宽度直接推断"线程池不够大"

`park` 的 wall time 只说明线程在等待，可能是：等队列任务（正常，说明池**空闲**）、等锁、等下游、等 CPU 配额。

**正确诊断顺序**：

```text
① 业务指标   → P99 是否真的恶化？是全链路还是某一段？
② 线程池指标 → queueSize 持续 > 0？rejectedCount > 0？activeCount 是否长期打满？
   ★ 若 queueSize ≈ 0 而 activeCount 打满 → 确实需要扩容或减少单任务耗时
   ★ 若 queueSize 持续增长      → 消费能力不足，先查单任务为什么慢
   ★ 若 activeCount 低但 P99 高 → 根本不是线程池问题，去查下游或 GC
③ 连续 jstack → 线程卡在哪个栈？是业务代码还是 getTask()？
   ★ 大量线程停在 getTask() 的 poll/take = 池【很闲】，别扩容
④ cgroup     → nr_throttled 是否增长？CPU 配额是否不足？
⑤ GC 日志    → 是否 GC 停顿导致的假性延迟？
```

> ⚠️ **最常见的误判**：看到"线程都在 park"就扩大线程池。如果它们 park 在 `getTask()` 里，那是线程**闲得没活干**，扩容只会让更多线程闲着，还增加了调度开销。
>
> 🔧 这条判别法的**真实 jstack 输出对照**（闲置 Worker vs 繁忙 Worker 的栈长什么样），见 [线上排查工具箱 2.3 节](#troubleshooting)。

### 坑 12：把 JDK 8 的线程池文章当 JDK 21 的事实（以及反过来）

本文开头已声明：**`ThreadPoolExecutor` 核心语义在 JDK 8→21 之间没变**，变的是 `addWorker`/`getTask` 中状态判断的写法，以及 `CAPACITY` → `COUNT_MASK` 的更名——**这次重构落在 JDK 10（[JDK-8186056](https://bugs.openjdk.org/browse/JDK-8186056)，Doug Lea 提交）**；JDK 11 与 JDK 21 的 `addWorker` 之间只差一行 `t.start()` → `container.start(t)`（JDK 19 的 JEP 444 引入，不改变执行语义）。完整证据链与那个"在 2^29 边界上并不严格等价"的例外，见 [Level 2](#level-2)。

反过来的误区同样常见：**以为 JDK 21 有了虚拟线程，`ThreadPoolExecutor` 就过时了。** 不对。CPU 密集型任务、需要严格并发上限的场景、与遗留代码集成的场景，平台线程池仍是正确选择。虚拟线程替代的是"用大线程池扛 I/O 阻塞"这一种用法，不是全部。

> 📌 **看源码前先 `java -version`；读文章先看它写于哪个版本。** 尤其是 pinning 相关的建议——2023-2024 年的绝大多数文章写于 JEP 491 之前，它们关于"必须把 synchronized 换成 ReentrantLock"的建议在 JDK 24+ 上已经过期。

### 坑 13：锁内做慢 I/O —— 一个线程持锁，全池 BLOCKED

这是坑 3（锁内 RPC）在**线程池语境下的放大版**，也是线上 `jstack` 里最常见的一类"全员卡死"现场，值得单独编号。

```java
// ❌ 池有 32 个线程，但它们永远只有 1 个在真正干活
public synchronized void updateInventory(Long skuId) {   // 或 synchronized(LOCK){}
    Inventory inv = repo.find(skuId);
    remoteClient.deduct(skuId);        // ★ 慢 I/O 在锁内，耗时由下游 P99 决定
    repo.save(inv);
}
```

**因果链——为什么它比"单纯的慢"严重得多**：

```text
一个 Worker 持锁做 2s 的 RPC
      ↓
其余 31 个 Worker 全部 BLOCKED 在同一把锁上
      ↓
★ 池的有效并发度从 32 【坍缩为 1】
      ↓
队列以 (QPS − 0.5/s) 的速度无限增长
      ↓
队列满 → 触发拒绝 → 上游超时重试 → QPS 进一步上升
      ↓
★ 正反馈循环：你以为是"池太小"，其实是【串行化】
```

> 📌 **这个坑的欺骗性在于它的所有表象都指向"扩容"**：队列在涨、任务在拒绝、`activeCount` 打满。但把 `maximumPoolSize` 从 32 调到 128，结果是 **127 个线程 BLOCKED 而不是 31 个**——有效并发度**仍然是 1**。这是"扩线程池解决不了的问题"里最典型的一种。

**线上样貌**（`jstack` 里的三个特征，缺一不可）：

```text
① 大量线程 java.lang.Thread.State: BLOCKED (on object monitor)
② 它们的栈里都有   - waiting to lock <0x00000000e2ded2b8>
③ 有且仅有一个线程 - locked <0x00000000e2ded2b8>   ← 同一个地址，它就是元凶
   且这个线程的栈顶是 I/O 调用（socketRead / sleep / SQL 执行）
```

**修正**——把锁的粒度从"整个方法"缩到"必须原子的那几行"：

```java
// ✅ 方案一：锁内只做内存状态转换，I/O 移到锁外（见 Level 5 决策卡 1）
Inventory snapshot;
synchronized (LOCK) { snapshot = repo.find(skuId); }   // 纳秒级
remoteClient.deduct(skuId);                            // 锁外，慢也不阻塞别人
synchronized (LOCK) { repo.saveWithVersion(snapshot); } // CAS/版本号收口

// ✅ 方案二：按 key 分片，让不同 SKU 互不阻塞
Lock lock = locks[spread(skuId) & (locks.length - 1)];

// ✅ 方案三：需要超时/可中断时，用 ReentrantLock 替代 synchronized
if (!lock.tryLock(100, MILLISECONDS)) { return fallback(); }
```

> 🔧 **完整的取证过程**（如何从一次 `jstack` 里通过比对锁地址找出持有者）见 [线上排查工具箱 2.3 节](#troubleshooting)，那里有真实的 dump 输出。

---

<a id="submission"></a>

## 📮 提交方式对比：execute / submit / invokeAll / invokeAny / CompletableFuture

> 📌 五种提交方式的差别**不在语法，在三件事**：任务被包装成了什么、异常去了哪、在哪一步阻塞。选错的代价通常不是报错，而是**静默失败**或**莫名其妙的延迟**。
>
> ⚠️ 本节所有行为结论均有 [`demo/ThreadPoolLabs.java`](../../demo/ThreadPoolLabs.java) Lab 5 实测支撑（`java ThreadPoolLabs 5`），输出为真实运行结果。

### 1. 一切差异的源头：`submit` 比 `execute` 多套了一层

先看 JDK 21 源码，三个 `submit` 重载**全部**归结为同一个动作：

```java
// AbstractExecutorService，三个重载殊途同归
public Future<?> submit(Runnable task) {
    RunnableFuture<Void> ftask = newTaskFor(task, null);   // ★ 包装
    execute(ftask);                                        // ★ 最终还是走 execute
    return ftask;
}
public <T> Future<T> submit(Runnable task, T result) { ... newTaskFor(task, result); execute(ftask); }
public <T> Future<T> submit(Callable<T> task)        { ... newTaskFor(task);         execute(ftask); }
```

```text
execute(task)                    submit(task)
     │                                │
     │                          newTaskFor(task)
     │                                │
     │                          ┌─────▼──────┐
     │                          │ FutureTask │ ← ★ 多出来的这一层
     │                          └─────┬──────┘
     ▼                                ▼
  Worker 拿到【你的对象】        Worker 拿到【FutureTask】
```

**实测确认**（Lab 5-A）——`submit` 一个 `Runnable`，Worker 实际拿到的是：

```text
submit 的 Runnable 被包成 -> java.util.concurrent.Executors$RunnableAdapter
```

**这一层就是异常消失的地方**。`FutureTask.run()` 的真实代码：

```java
try {
    result = c.call();
    ran = true;
} catch (Throwable ex) {
    result = null;
    ran = false;
    setException(ex);        // ★★★ 异常被【存进 Future 的状态】，不再向上抛
}
```

> 📌 **因果闭环**：异常没有传播到 `Thread.run()`，所以 `UncaughtExceptionHandler` **永远不会被触发**；它躺在 `FutureTask` 的 `outcome` 字段里，**只有 `get()` 才能把它取出来**。这就是坑 3 的源码级解释——不是"日志级别不够"，而是**异常在设计上就被改成了返回值**。

### 2. 五种方式全对比

| 提交方式 | 返回 | 阻塞点 | 任务异常去向 | 典型用途 |
|---|---|---|---|---|
| `execute(Runnable)` | `void` | 不阻塞 | ✅ 抛到 `UncaughtExceptionHandler`（**看得见**） | 纯异步，不关心结果 |
| `submit(Runnable)` | `Future<?>` | 不阻塞 | ⚠️ 存进 Future，**不 `get()` 就永久消失** | 需要知道"做完了没" |
| `submit(Callable<T>)` | `Future<T>` | 不阻塞 | ⚠️ 同上 | 需要返回值 |
| `invokeAll(tasks)` | `List<Future<T>>` | ★ **阻塞到全部完成** | 存进各自 Future | 批量任务，全都要 |
| `invokeAny(tasks)` | `T` | ★ **阻塞到第一个成功** | 全失败才抛 | 冗余请求，谁快用谁 |
| `CompletableFuture` | `CF<T>` | 不阻塞（回调驱动） | 存进 CF，需 `exceptionally`/`handle` | 编排、组合、依赖链 |

### 3. `invokeAll` / `invokeAny`：两个被低估的阻塞语义

#### 3.1 `invokeAll` 会被最慢的任务拖住

源码里它在提交完所有任务后，**逐个 `f.get()` 等待**：

```java
for (Callable<T> t : tasks) { RunnableFuture<T> f = newTaskFor(t); futures.add(f); execute(f); }
for (int i = 0; i < size; i++) {
    Future<T> f = futures.get(i);
    if (!f.isDone()) {
        try { f.get(); } catch (CancellationException | ExecutionException ignore) {}  // ★ 阻塞在这
    }
}
```

**实测**（两个任务分别耗时 300ms / 1500ms）：

```text
返回耗时 1501ms，全部 isDone=true
★ 被最慢的那个拖住 —— 批量场景要警惕长尾
```

> 📌 注意那个 `catch (...) { ignore }`：**`invokeAll` 本身不会因为某个任务失败而抛异常**，它只保证"全部终结"。失败信息全部留在各自的 `Future` 里，**你必须逐个 `get()` 才能发现**。

#### 3.2 `invokeAny`：谁先成功用谁，其余取消

**实测**（1500ms 的任务排在前面，300ms 的排在后面）：

```text
返回 "快"，耗时 302ms —— 其余任务被取消
```

它天然适合**冗余请求（hedged request）**：同时问三个副本，谁先回用谁。但要注意它会真的**并发占用三份资源**，在下游容量紧张时要配合 Semaphore。

#### 3.3 ⚠️ `invokeAll(timeout)` 超时的真实后果

这是最容易写错的一个。**超时不是"返回已完成的部分"，而是"取消未完成的"**：

```text
invokeAll(timeout=800ms)，任务耗时 100ms / 5000ms
  任务0 -> 完成
  任务1 -> ★ CancellationException（已被 cancel）
```

```java
// ❌ 超时后这样写，整批结果直接崩掉
List<Future<R>> futures = pool.invokeAll(tasks, 3, TimeUnit.SECONDS);
return futures.stream().map(Future::get).toList();   // 未完成的 get() 抛 CancellationException

// ✅ 必须逐个处理，区分三种结局
List<R> results = new ArrayList<>();
for (Future<R> f : pool.invokeAll(tasks, 3, TimeUnit.SECONDS)) {
    try { results.add(f.get()); }
    catch (CancellationException ce) { results.add(fallback()); }      // 超时被取消
    catch (ExecutionException ee)    { log.error("失败", ee.getCause()); results.add(fallback()); }
}
```

### 4. 决策树：到底该用哪个

```text
需要返回值 / 需要知道任务是否失败？
 ├─ 否 → execute()
 │        ★ 唯一异常"看得见"的方式，配好 UncaughtExceptionHandler 即可
 └─ 是 ↓
     一批任务，还是单个？
      ├─ 单个 → submit() + 【必须】get() 或覆写 afterExecute
      └─ 一批 ↓
          全都要结果？
           ├─ 是 → invokeAll(tasks, timeout)  ★ 务必带超时 + catch CancellationException
           └─ 否，谁快用谁 → invokeAny(tasks, timeout)
          
     需要编排（A 完了做 B、多个结果合并、失败降级）？
      └─ CompletableFuture + 【显式传 Executor】（见 Level 6）
```

> 📌 **最容易被忽略的一条**：`execute` 是唯一让异常"自然浮现"的提交方式。很多团队为了"统一风格"全用 `submit`，反而把 `UncaughtExceptionHandler` 这条最省事的告警通道给关掉了。**如果你不需要返回值，`execute` 就是更好的选择。**

### 5. 提交方式与前面各层的联系

| 提交方式 | 它触发的是哪一层的机制 |
|---|---|
| 全部五种 | 最终都调 `execute()` → [Level 3 的三道门](#level-3) |
| `submit` 系列 | 包装成 `FutureTask` → [坑 3 的异常吞噬](#pitfalls) |
| `invokeAll/Any` | 内部逐个 `execute` + `get` → 一次提交**多个**任务，可能瞬间打满队列 |
| `CompletableFuture` | 不传 Executor 时走 `commonPool` → [Level 6 的默认池陷阱](#level-6) |

> ⚠️ **一个容易踩的组合坑**：`invokeAll` 一次提交 1000 个任务到一个 `queueCapacity=100` 的池，**会在提交过程中就触发拒绝**，且 `invokeAll` 内部的 `catch (Throwable t) { cancelAll(futures); throw t; }` 会把**已提交的任务全部取消**。批量提交前，先确认队列容量够。

> 🔴 **口诀**：submit 多包一层，异常存进 Future；execute 异常看得见；invokeAll 等最慢，超时即取消。

---

<a id="t0-t8"></a>

## 📊 线程池竖切总表 T0–T8

| 维度 | T0 构造 | T1 提交 | T2 门 1 | T3 门 2 | T4 门 3/4 | T5 执行 | T6 回收 | T7 关闭 | T8 诊断 |
|---|---|---|---|---|---|---|---|---|---|
| **位置** | `new TPE(...)` | `execute(T)` | core 未满 | 入队 | 扩容/拒绝 | `runWorker` | `getTask` 返 null | `shutdown*` | 监控取证 |
| **ctl 变化** | `(RUNNING,0)` | 读 ctl | 人数 CAS+1 | 不变 | 人数 CAS+1 | 不变 | 人数 CAS-1 | 高位推进 | 只读 |
| **关键动作** | 参数校验、**不建线程** | 判空 | `addWorker(T,true)` | `offer` + **recheck** | `addWorker(T,false)` / `reject` | `lock`→`run`→`unlock` | `poll` 超时 | 中断 + 排空 | dump/JFR/指标 |
| **不变量** | 懒创建 | 四种归宿之一 | 一次 CAS 定状态与人数 | **入队≠有人消费** | 拒绝是业务语义 | 持锁期间不可被中断 | 至少留一人守非空队列 | 中断=请重查，非停止 | 多证据交叉 |
| **典型坑** | 用 `Executors` | 吞异常 | 以为会复用空闲线程 | 忽略 recheck | 默认 Abort 未处理 | ThreadLocal 未清 | 以为 core 永不回收 | 不接 `shutdownNow` 返回值 | 单图定罪 |
| **虚拟线程下** | 无需构造池 | `submit` 即新建 VT | — | — | **需 Semaphore 代替** | 可能 pin(≤JDK23) | 无回收概念 | try-with-resources | JFR `VirtualThreadPinned` |

---

<a id="references"></a>

## 📚 版本勘误与延伸阅读

| ❌ 常见说法 | ✅ 更准确的说法 |
|---|---|
| JDK 21 重写了线程池 | **`ThreadPoolExecutor` 核心语义未变**。变的是 `addWorker`/`getTask` 中状态判断的写法（德摩根律展开）和 `CAPACITY`→`COUNT_MASK` 更名。真正的变化在外部：虚拟线程。 |
| 任务多了就会创建到 `maximumPoolSize` | 仅当**队列 offer 失败**才扩容。无界队列下 max 永不生效。 |
| `corePoolSize` 是最小线程数 | 是"不因空闲被回收的线程数上限"。池可以有 0 个线程（懒创建 / `allowCoreThreadTimeOut`）。 |
| Worker 分核心和非核心两种 | **没有这个字段**。身份由 `getTask()` 中 `wc > corePoolSize` 动态计算，同一个 Worker 的身份会随人数变化而改变。 |
| `shutdown()` 会停止正在执行的任务 | 不会。它只中断**闲置**的 Worker，且中断的语义是"请重新检查状态"。 |
| 虚拟线程要配合线程池使用 | **恰恰相反**。用 `newVirtualThreadPerTaskExecutor()`；池化虚拟线程等于放弃它的核心优势。 |
| 虚拟线程下 `synchronized` 一定会 pin | **JDK 21–23 会；JDK 24+（JEP 491）已修复**，monitor 归属已迁移到虚拟线程。仅 native/FFM 阻塞仍会 pin。 |
| 用 `-Djdk.tracePinnedThreads` 排查 pinning | 该选项在 **JDK 24 已移除**。改用 JFR 的 `jdk.VirtualThreadPinned` 事件，覆盖场景更广。 |
| 线程数按 `2N` / `N+1` 公式定 | 公式是**起点**不是答案。容器里 `availableProcessors()` 可能失真，且必须以压测拐点定案。 |
| `CompletableFuture` 默认用 commonPool | 通常是。但**单核环境下**（`commonPool` 并行度 ≤1）会退化为 `ThreadPerTaskExecutor`——每任务新建线程。 |
| `parallelStream` 有自己的线程池 | 它用**全 JVM 共享的 `commonPool`**，默认并行度 `CPU - 1`。放阻塞操作会连累所有使用者。 |
| 响应式比线程池"性能更好" | 响应式的独特价值是**精细背压**和流式处理。若只为提高 I/O 并发，JDK 21+ 的虚拟线程通常是心智负担更低的选择。 |
| `@Async` 默认用 `SimpleAsyncTaskExecutor`（不池化） | **只在纯 Spring Framework 下成立**。Spring Boot 的 `TaskExecutionAutoConfiguration` 会提供 `ThreadPoolTaskExecutor`，但其默认 `queue-capacity = Integer.MAX_VALUE`——**等价于一个 `newFixedThreadPool(8)`**。两者失败模式不同（线程爆炸 vs 堆堆积），但都不能直接上生产。 |
| 结构化并发的写法是 `new StructuredTaskScope.ShutdownOnFailure()` | **这是 JDK 21–24 的旧预览语法**。JDK 25 的 JEP 505 已重构为 `StructuredTaskScope.open(Joiner)`，两个 `Shutdown*` 子类被删除；JDK 26 的 JEP 525 改了方法名与返回类型；JDK 27 的 JEP 533 又把异常类型从 `FailedException` 改成了 `ExecutionException`。**它至今（第七次预览）仍未定稿。** |
| 开了 `spring.threads.virtual.enabled=true` 就万事大吉 | 它会让 `@Async` 换成虚拟线程版 `SimpleAsyncTaskExecutor`，**忽略所有 pool 配置**——你原先设的有界队列会静默失效。必须同步把容量阀门改用 `Semaphore` 或 `concurrencyLimit` 架起来。 |
| 虚拟线程"没有栈内存开销" | 栈从**堆外**搬到了**堆内**（`StackChunk` 对象），是**成本转移**而非消失，且它现在参与 GC。深栈 + 超高并发场景需实测年轻代压力。 |
| `CAPACITY`→`COUNT_MASK` 的重构发生在 JDK 21 | **错，落在 JDK 10**，出处是 [JDK-8186056](https://bugs.openjdk.org/browse/JDK-8186056)（commit `c3664b7f38a0`，Doug Lea），同一 commit 一并完成了 `runStateAtLeast` 改写。实测：JDK 8/9 用 `CAPACITY`，JDK 10 起为 `COUNT_MASK`。见 [Level 2](#level-2)。 |
| JDK 11 与 JDK 21 的 `addWorker` 逐字相同 | **不成立**（本文早期版本的说法）。`diff` 有一行：`t.start()` → `container.start(t)`，由 JDK 19 的 JEP 444（[JDK-8284161](https://bugs.openjdk.org/browse/JDK-8284161)）引入，用于把 Worker 注册进 `SharedThreadContainer`。**不改变执行语义，但"逐字相同"是错的。** |
| `CAPACITY`→`COUNT_MASK` 是纯改名，语义完全等价 | **在现实取值域内等价，数学上不严格等价**。旧代码上限是 `min(bound, CAPACITY)`，新代码是 `bound & COUNT_MASK`。当 `core`/`max` 为 2^29 整数倍时结果为 0——实测 JDK 8 正常执行任务，JDK 11/21 任务永久滞留队列。这是**被 Javadoc 文档化的行为**（*"effective limit is `corePoolSize & COUNT_MASK`"*），非 bug，但断言"完全相同"不准确。见 [Level 2](#level-2)。 |
| `InheritableThreadLocal` 能在线程池里传递上下文 | **语义上就不成立**。它只在 `Thread` 构造时拷贝一次，而 Worker 是复用的 → 任务拿到的是"池初始化那一刻某个无关请求"的值。需用 `TransmittableThreadLocal` 或 `TaskDecorator`。见 [坑 4-B](#pitfalls)。 |
| 能改 core/max 就算动态线程池了 | 那只是第一块拼图。真正的动态线程池还要解决配置中心推送、**队列容量可变**（`ArrayBlockingQueue` 的 capacity 是 `final`）、监控看板、告警、变更审计。见 [决策卡 7](#production-decisions)。 |
| `@Contended` 应该多用来防伪共享 | 它治的是**伪**共享。`ctl` 面对的是**真**共享（大家本来就要争同一个值），加 padding 无效。JDK 自己只在 `Striped64.Cell`、`ConcurrentHashMap.CounterCell` 这类**分槽**结构上用它。见 [Level 7.5](#level-7-5)。 |
| `Executors` 的坑都是"无界"（队列或线程） | **`newWorkStealingPool()` 是例外**：它返回 `ForkJoinPool`，工作线程**全是守护线程**，`main` 结束时未完成任务被**静默杀死**——不是资源耗尽，是任务丢失且无任何告警。 |
| 虚拟线程调度器下 `@Scheduled` 一定会并发执行 | 更准确：**保证的来源变了**。`fixedDelay` 靠 Spring 单条调度线程仍不重叠；**`fixedRate`/`cron` 走 thread-per-execution，重叠成为可能**。且该行为随 Spring 版本演进，必须在你的版本上实测。 |
| `spring.threads.virtual.enabled=true` 只影响 Web 层 | 它是**全局开关**，同时改掉 Web、`@Async`、`@Scheduled` 三处。三者风险递增；可通过显式定义 `TaskScheduler` Bean 让定时任务"退出"该开关。 |

### 一手资料

- **源码**（请对照你的部署版本）
  - [OpenJDK 21 `ThreadPoolExecutor.java`](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/util/concurrent/ThreadPoolExecutor.java) —— 重点读类注释里 Doug Lea 亲笔写的设计说明
  - [OpenJDK 21 `VirtualThread.java`](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/lang/VirtualThread.java) —— `createDefaultScheduler()` 是载体池的真相
  - [OpenJDK 21 `ForkJoinPool.java`](https://github.com/openjdk/jdk21u/blob/master/src/java.base/share/classes/java/util/concurrent/ForkJoinPool.java)
- **JEP**
  - [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)（JDK 21 正式）
  - [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)（JDK 24 正式）
  - [JEP 453: Structured Concurrency](https://openjdk.org/jeps/453)（预览，API 后续有演进）
- **书籍**：Brian Goetz《Java Concurrency in Practice》第 6、8 章——虽写于 Java 5 时代，但"任务与执行策略解耦""饥饿死锁"等论述至今未过时
- **Reactor**：[Schedulers Javadoc](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html) —— 默认值随版本变化，以你的依赖版本为准

> **最后只留一个问题**：订单 T 此刻在厨房的哪个位置？`ctl` 是多少？谁有责任执行它？如果此刻进程被 `kill -9`，T 会怎样？能稳定回答这四句，线程池的主干才算真正长进脑子里。

---

<a id="troubleshooting"></a>

## 🔧 线上排查工具箱：从"服务变慢了"到定位根因

> 📌 本节是**值班手册**，按"你手上有什么证据"组织，不按"有哪些工具"组织。全文各处散落的诊断手段在这里收拢成一条可执行的路径。
>
> ⚠️ 下面所有 `jstack` 输出**均为真实运行结果**（JDK 11 实测，代码见文末复现说明），不是手写示意。

### 0. 先建立分诊顺序：不要一上来就 jstack

```text
每一步都要能回答"下一步查什么"，否则就是在碰运气。

① 业务指标   → P99 真的恶化了吗？是全链路还是某一段？
                └─ 若下游 P99 同步恶化 → 直接跳到 ④，别查线程池
② 线程池指标 → queueSize / activeCount / rejectedCount 三个数
                └─ 这三个数的组合直接决定后续方向（见第 1 节判定表）
③ jstack     → 线程卡在哪个栈？业务代码还是 getTask()？
                └─ ★ 这一步能区分"池不够用"和"池闲得慌"
④ 下游 & 环境 → DB/RPC 的 P99、连接池 pending、cgroup throttle、GC
⑤ JFR/profiler → 前四步定位到范围后，才用重武器做精确取证
```

### 1. 三个数的判定表：不用工具就能定方向

先看监控里的三个数（没接监控的话，第 2 节教你从 dump 里挖）：

| queueSize | activeCount | 结论 | 该做什么 |
|---|---|---|---|
| ≈ 0 | 低 | **池很闲**，问题不在这 | 查 GC、下游、网络、cgroup |
| ≈ 0 | 打满 | 刚好饱和，但没积压 | 观察，接近拐点 |
| **持续增长** | 打满 | **消费能力不足** | ★ 先查单任务为什么慢，**不是**先扩容 |
| 持续增长 | **低** | ⚠️ **异常信号** | Worker 死了/卡在非业务栈；查 ③ |
| 满 + 有拒绝 | 打满 | 真饱和 | 确认下游能承接后再扩，或做降级 |

> 📌 **第四行是最容易被忽略的异常**：队列在涨，但没几个线程在干活。这通常意味着 Worker 线程卡在了**不该卡的地方**（比如 `getTask` 之外的死锁），或者线程因未捕获异常反复死亡重建。**这种情况扩容完全无效。**

### 2. jstack：三条命令 + 一个"一眼判别法"

#### 2.1 先看全局分布

```bash
# 线程状态分布 —— 第一眼看什么
jstack <pid> | grep 'java.lang.Thread.State' | sort | uniq -c | sort -rn
```

真实输出：

```text
      7    java.lang.Thread.State: RUNNABLE
      2    java.lang.Thread.State: WAITING (parking)
      2    java.lang.Thread.State: TIMED_WAITING (sleeping)
      2    java.lang.Thread.State: BLOCKED (on object monitor)     ← ★ 有锁竞争
      1    java.lang.Thread.State: WAITING (on object monitor)
```

**`BLOCKED` 是最值钱的信号**：它意味着有线程在抢 `synchronized` 锁抢不到。如果 `BLOCKED` 数量可观，直接跳到 2.3 找持锁者。

#### 2.2 按池名分组——这就是坑 1 强调线程命名的回报

```bash
# 每个业务池各有多少线程（前提：你按坑 1 的要求命名了线程）
jstack <pid> | grep -E '^"(order-biz|idle-pool)' | sed 's/-[0-9]*"/"/' | awk '{print $1}' | sort | uniq -c
```

```text
      2 "idle-pool"
      3 "order-biz"
```

> 📌 如果你的输出是一堆 `pool-3-thread-127`，**这一步就废了**——你无法知道哪个池属于哪个业务。这就是"线程命名是零成本可观测性投资"的具体兑现场景。

#### 2.3 ★ 一眼判别法：这个池是"忙"还是"闲"

**这是本节最有用的一招。** 看 Worker 线程栈的**倒数第三行**：

```bash
# 闲置 Worker 数（停在 getTask 里等活）
jstack <pid> | grep -c 'ThreadPoolExecutor.getTask'
```

**闲置 Worker 的真实栈**（停在 `getTask` → `take`）：

```text
"idle-pool-368" #13 ... waiting on condition
   java.lang.Thread.State: WAITING (parking)
	at jdk.internal.misc.Unsafe.park(java.base@11/Native Method)
	at java.util.concurrent.locks.LockSupport.park(...)
	at java.util.concurrent.LinkedBlockingQueue.take(...)
	at java.util.concurrent.ThreadPoolExecutor.getTask(...)      ← ★ 关键标志
	at java.util.concurrent.ThreadPoolExecutor.runWorker(...)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(...)
```

**繁忙 Worker 的真实栈**（`runWorker` 上面直接是业务代码）：

```text
"order-biz-25" #10 ... waiting on condition
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(java.base@11/Native Method)
	at Sick.lambda$main$1(Sick.java:9)                           ← ★ 业务代码
	- locked <0x00000000e2ded2b8> (a java.lang.Object)           ← ★ 它持有锁！
	at java.util.concurrent.ThreadPoolExecutor.runWorker(...)

"order-biz-749" #11 ... waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
	at Sick.lambda$main$1(Sick.java:9)
	- waiting to lock <0x00000000e2ded2b8> (a java.lang.Object)  ← ★ 同一个锁地址
	at java.util.concurrent.ThreadPoolExecutor.runWorker(...)
```

> 📌 **判别规则**：
> - 栈里有 `getTask` → 这个 Worker **闲着**，在等任务
> - `runWorker` 上面直接是**你的业务类** → 这个 Worker **正在干活**
>
> **"大量线程 park"绝不等于"线程池不够用"。** 如果它们都 park 在 `getTask` 里，恰恰说明池**闲得慌**，此时扩容只会让更多线程闲着。这是坑 11 那句话的实证版本。

**锁竞争的取证**：注意上面两段栈里的 `- locked <0x...e2ded2b8>` 和 `- waiting to lock <0x...e2ded2b8>`——**地址相同**。这就找到了完整因果：`order-biz-25` 持有锁并在 `sleep`（模拟锁内慢 RPC），另外两个线程全部 `BLOCKED` 等它。**这正是 [坑 13"锁内做慢 I/O"](#pitfalls) 的线上样貌**——池有 3 个线程，有效并发度却是 1。

```bash
# 找出谁持有大家在等的那把锁
jstack <pid> | grep -B 5 'waiting to lock <0x...>'   # 先拿到锁地址
jstack <pid> | grep -B 5 'locked <0x...>'            # 再找持有者
jstack <pid> | grep -i deadlock -A 30                # 顺手查死锁（JVM 会自动检测）
```

#### 2.4 连续采样：区分"卡住"和"慢"

单次 dump 只是快照，**判断不了是卡死还是正常繁忙**：

```bash
for i in $(seq 1 5); do jstack <pid> > dump_$i.txt; sleep 2; done
# 同一个线程在 5 次 dump 里都停在【同一行】→ 卡死
# 每次栈都不同 → 只是繁忙，不是卡死
```

### 3. 不改代码、不接监控，从堆里挖线程池状态

有时线上没接 Micrometer，又不能重启。两条路：

```bash
# 路线 A：看队列/任务对象的数量级
jcmd <pid> GC.class_histogram | grep -E 'BlockingQueue|\$\$Lambda'

# 路线 B（推荐）：Arthas 直接读线程池对象的字段
vmtool --action getInstances --className java.util.concurrent.ThreadPoolExecutor --limit 10 \
       --express 'instances.![toString()]'
# ThreadPoolExecutor 的 toString() 自带 pool size / active / queued / completed
```

Arthas 其余高频命令：

```bash
thread -n 5                    # CPU 占用最高的 5 个线程
thread -b                      # ★ 找出阻塞其他线程的那一个（自动分析）
thread --state BLOCKED         # 只看 BLOCKED 线程
trace *Service execute -n 5    # 方法级耗时，定位"单任务为什么慢"
watch *Service *  '{params,returnObj,throwExp}' -x 2   # 抓静默失败的异常
```

> 📌 `thread -b` 是 Arthas 最被低估的命令：它自动完成了 2.3 节里"找锁地址 → 找持有者"的全过程，直接告诉你**是哪个线程在阻塞全场**。

### 4. 环境层：别把外部限制当成线程池问题

```bash
# CPU 配额（容器里最常见的假性"线程池不够"）
cat /sys/fs/cgroup/cpu.max                    # v2: "200000 100000" = 2 核
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us       # v1
jcmd <pid> VM.flags | grep ActiveProcessorCount

# ★ 是否被限流 —— nr_throttled 持续增长 = CPU 不够，扩线程只会更糟
cat /sys/fs/cgroup/cpu.stat | grep throttled

# GC 是否才是真凶
jstat -gcutil <pid> 1000 10
jcmd <pid> GC.heap_info
```

> 📌 **`nr_throttled` 持续增长时，任何线程池调优都是徒劳**：你的进程正在被内核周期性冻结。此时扩大线程池只会加剧上下文切换。**正确动作是加 CPU 配额或减少计算量。**

### 5. JFR：精确取证（前四步定位到范围后再用）

```bash
# 通用采集
jcmd <pid> JFR.start name=diag settings=profile duration=120s filename=/tmp/diag.jfr
jcmd <pid> JFR.dump name=diag filename=/tmp/diag.jfr

# 虚拟线程 pinning（JDK 21+，替代已被 JDK 24 移除的 -Djdk.tracePinnedThreads）
jfr summary /tmp/diag.jfr | grep -i pinned
jfr print --events jdk.VirtualThreadPinned /tmp/diag.jfr

# 虚拟线程感知的线程转储（JDK 21+）
jcmd <pid> Thread.dump_to_file -format=json /tmp/vt.json

# 锁竞争 / 线程park 事件
jfr print --events jdk.JavaMonitorEnter,jdk.ThreadPark /tmp/diag.jfr | head -50
```

### 6. 症状 → 根因速查表

| 线上症状 | 最可能的根因 | 定位命令 | 对应章节 |
|---|---|---|---|
| P99 突刺，队列不长 | 下游慢 / GC / cgroup throttle | `cpu.stat`、`jstat -gcutil` | 决策卡 4 |
| 队列持续增长 | 单任务变慢（下游慢或锁内串行化） | `trace`、连续 jstack | [坑 13](#pitfalls) |
| 大量线程 park | **多半是池闲着** | `grep -c getTask` | 2.3 节 |
| 大量 BLOCKED | 锁内做慢 I/O，池被串行化 | `thread -b`、比对锁地址 | [坑 13](#pitfalls) |
| 任务提交了没执行 | 无界队列积压 / daemon 线程 | `GC.class_histogram` | [坑 1](#pitfalls) |
| 任务失败无日志 | `submit` 吞异常 | `watch` 抓异常 | [坑 3](#pitfalls) |
| 定时任务不跑了 | 未捕获异常取消了周期 | 检查 try-catch | [Level 6](#level-6) |
| 服务关不掉 | 非 daemon 线程 / 忘记 shutdown | `jstack` 看存活线程 | [Level 5](#level-5) |
| 用户看到别人的数据 | ThreadLocal 未清理 | 审计 `remove()` | [坑 4](#pitfalls) |
| 虚拟线程吞吐上不去 | pinning（JDK≤23） | JFR `VirtualThreadPinned` | [Level 7](#level-7) |

> 🔬 **复现本节现场**：起一个 `core=max=3` 的池，3 个任务在 `synchronized` 块里 `sleep`，再灌 500 个任务进队列，另起一个闲置池做对照。用上面的命令依次采集，即可得到与本节完全一致的输出形态。

> 🔴 **口诀**：先看三个数再动手；栈里有 getTask 就是闲；锁地址一比找元凶；throttled 在涨调啥都白搭。

---

<a id="cheatsheet"></a>

## 🧰 实战速查：默认池的雷 + 配置模板 + Review 清单

> 📌 前面各层讲的是"为什么"。这一节是**查阅用的**：把散落在全文的结论收拢成三张表和一组模板，方便你在写代码、做 Code Review 时直接对照。
>
> ⚠️ 模板里的**数字全部是占位示例**，必须按第 3 小节的方法用你自己的数据替换。**照抄数字 = 换一种方式犯错。**

### 1. 五种 `Executors` 默认池：一句话结论

> 📌 完整的因果分析、等价构造与可运行 demo 见 [坑 1](#pitfalls)。这里只留结论，供 Review 时快速比对。

| 工厂方法 | 一句话结论 |
|---|---|
| `newFixedThreadPool(n)` | 队列无界 → 堆 OOM |
| `newSingleThreadExecutor()` | 同上，且并发度锁死为 1 |
| `newCachedThreadPool()` | 线程数无上限 → 线程爆炸 |
| `newScheduledThreadPool(n)` | 队列无界且 `max` 失效；任务异常会静默取消周期 |
| `newWorkStealingPool()` | ⚠️ ForkJoinPool + **守护线程** → 任务**静默丢失**，无任何告警 |

**记忆锚点**：前四个是 `ThreadPoolExecutor` 的**参数**没配好（无界队列或无界线程），第五个是**换了个体系**（ForkJoinPool + 守护线程）。**前四个会喊疼，第五个不会。**

### 2. 生产模板（JDK 8 / JDK 21 通用 + JDK 21 专属）

#### 2.1 通用基座：一个"什么都想到了"的池

以下代码 **JDK 8 与 JDK 21 完全通用**——这正是本文反复强调的"`ThreadPoolExecutor` 核心语义未变"的实证。

```java
/**
 * 生产级线程池模板。七个参数 + 三项增强，全部显式声明。
 * JDK 8 / 21 通用（JDK 8 需把 var 换成具体类型、lambda 保持不变）。
 */
public final class BizExecutors {

    public static ThreadPoolExecutor create(String name, int core, int max,
                                            int queueCap, MeterRegistry registry) {
        ThreadPoolExecutor pool = new ThreadPoolExecutor(
                core,                                   // ① 核心线程数
                max,                                    // ② 最大线程数
                60L, TimeUnit.SECONDS,                  // ③④ 空闲回收时间
                new ArrayBlockingQueue<>(queueCap),     // ⑤ ★ 必须有界
                namedFactory(name),                     // ⑥ ★ 必须命名
                rejectedHandler(name, registry)         // ⑦ ★ 必须显式选择
        ) {
            /** 增强一：兜住 submit 吞掉的异常（见坑 3） */
            @Override
            protected void afterExecute(Runnable r, Throwable t) {
                super.afterExecute(r, t);
                if (t == null && r instanceof Future<?>) {
                    Future<?> f = (Future<?>) r;
                    if (f.isDone()) {
                        try { f.get(); }
                        catch (CancellationException ce) { return; }
                        catch (ExecutionException ee)    { t = ee.getCause(); }
                        catch (InterruptedException ie)  { Thread.currentThread().interrupt(); return; }
                    }
                }
                if (t != null) log.error("[{}] 任务执行异常", name, t);
            }
        };
        // 增强二：预热，消除冷启动毛刺（见坑 8）
        pool.prestartAllCoreThreads();
        // 增强三：接入监控
        ExecutorServiceMetrics.monitor(registry, pool, name);
        return pool;
    }

    /** 线程命名 + 未捕获异常兜底 + 非守护（确保 JVM 等它跑完） */
    private static ThreadFactory namedFactory(String name) {
        AtomicInteger seq = new AtomicInteger(1);
        return r -> {
            Thread t = new Thread(r, name + "-" + seq.getAndIncrement());
            t.setDaemon(false);                          // ★ 显式声明，不留悬念
            t.setUncaughtExceptionHandler(
                (thread, ex) -> log.error("[{}] 未捕获异常", thread.getName(), ex));
            return t;
        };
    }

    /** 拒绝策略：打点 + 告警 + 抛出（复杂降级必须自定义，见坑 2 的因果分析） */
    private static RejectedExecutionHandler rejectedHandler(String name, MeterRegistry reg) {
        Counter rejected = reg.counter("pool.rejected", "pool", name);
        return (r, executor) -> {
            rejected.increment();
            log.warn("[{}] 池饱和 active={} queue={} completed={}", name,
                    executor.getActiveCount(), executor.getQueue().size(),
                    executor.getCompletedTaskCount());
            throw new RejectedExecutionException(name + " saturated");
        };
    }
}
```

**配套的优雅关闭**（Level 5 的落地版，任何池都该有）：

```java
public static void shutdownGracefully(ExecutorService pool, String name,
                                      long gracefulSec, long forceSec) {
    pool.shutdown();
    try {
        if (!pool.awaitTermination(gracefulSec, TimeUnit.SECONDS)) {
            List<Runnable> dropped = pool.shutdownNow();          // ★ 接住返回值
            log.error("[{}] 强制关闭，丢弃 {} 个未执行任务", name, dropped.size());
            dropped.forEach(BizExecutors::persistForReplay);      // ★ 落盘/回投 MQ
            if (!pool.awaitTermination(forceSec, TimeUnit.SECONDS))
                log.error("[{}] 仍有任务未响应中断——检查是否吞了 InterruptedException", name);
        }
    } catch (InterruptedException e) {
        pool.shutdownNow();
        Thread.currentThread().interrupt();                       // ★ 恢复中断标志
    }
}
```

#### 2.2 JDK 21 专属：虚拟线程 + 信号量

```java
/** JDK 21+：线程模型与资源模型彻底分离 */
public class VirtualThreadService implements AutoCloseable {

    // 执行者：无限量、不池化
    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

    // 容量阀门：从池大小【搬家】到这里 —— 每个下游一个
    private final Semaphore dbPermits   = new Semaphore(80);   // ← 连接池上限
    private final Semaphore httpPermits = new Semaphore(300);  // ← 下游限流

    public Future<Result> handle(Request req) {
        return executor.submit(() -> {
            if (!dbPermits.tryAcquire(150, TimeUnit.MILLISECONDS))
                throw new ServiceBusyException("db saturated");   // 快速失败，可解释
            try { return query(req); }
            finally { dbPermits.release(); }
        });
    }

    @Override
    public void close() { executor.close(); }   // JDK 19+：shutdown + awaitTermination
}
```

> 📌 `ExecutorService` 从 JDK 19 起实现了 `AutoCloseable`，`close()` 等价于 `shutdown()` + 无限期 `awaitTermination()`。配合 try-with-resources 很优雅，**但要注意它是"无限期等待"**——生产上如果任务可能卡死，仍应使用带超时的显式关闭。

#### 2.3 两个版本的选型对照

| 场景 | JDK 8 | JDK 21 |
|---|---|---|
| CPU 密集计算 | `TPE(CPU+1, ...)` | **同左**（虚拟线程无帮助） |
| I/O 密集、高并发 | `TPE(大 core, 有界队列)` + 反复调参 | ✅ **虚拟线程 + Semaphore** |
| 定时任务 | `ScheduledTPE(n>1)` + try-catch | 同左；**慎用虚拟线程调度器**（见决策卡 6） |
| 分治计算 | `ForkJoinPool`（自建，勿用 common） | 同左 |
| 异步编排 | `CompletableFuture` + **显式 Executor** | 同左，或结构化并发（**待定稿**） |
| 下游容量控制 | `Semaphore` | `Semaphore`（**更加必需**） |

### 3. 参数怎么定：从 SLA 反推，而不是套公式

```text
Step 1 · 先量出三个数（压测/APM，不要拍脑袋）
   T_task = 单任务平均耗时          例：50ms
   T_cpu  = 其中纯 CPU 耗时          例：5ms   → I/O 占比 90%
   QPS    = 目标吞吐                 例：200/s

Step 2 · 并发度起点（利特尔法则，比 2N 公式可靠）
   并发数 = QPS × T_task = 200 × 0.05s = 10
   corePoolSize ≈ 10（I/O 密集可上浮，CPU 密集则以 CPU 核数封顶）
   ★ 先用 cgroup 确认你真有几个核：cat /sys/fs/cgroup/cpu.max

Step 3 · 队列容量【从 SLA 反推】—— 这是最常被搞错的一步
   可容忍排队延迟 = SLA_P99 − T_task = 200ms − 50ms = 150ms
   队列容量 = 可容忍排队延迟 ÷ T_task × 并发数
            = 0.15s ÷ 0.05s × 10 = 30
   ★ 不是"怕丢任务就设大"，而是"超过这个数，任务执行了也已超时"

Step 4 · maximumPoolSize
   = corePoolSize × 突发倍数（1.5~3），且必须校验下游能承接
   ★ 前提：队列有界，否则这个值永远不生效

Step 5 · 压测验证（成功标准必须【同时】满足）
   吞吐达标 ∧ P99/P999 达标 ∧ 拒绝率可解释 ∧ 下游未饱和 ∧ 无 CPU throttle
```

> 📌 **`N+1` / `2N` 公式的正确用法是"给一个数量级的起点"**，不是给答案。上面这套从 SLA 反推的方法，好处是**每个数字都能在评审时说出依据**——而"我用了 2N 公式"在追问下站不住。

### 4. Code Review 十六条检查清单

写完/评审线程池代码时逐条过：

```text
【创建】
□  1. 没有使用任何 Executors.newXxx()（五种全禁）
□  2. 队列是有界的，且容量能说出从哪个 SLA 推导而来
□  3. 线程有业务含义的命名前缀（jstack 里能 grep 到）
□  4. 拒绝策略是显式选择的，且想清楚了"谁是提交者"
□  5. 自定义拒绝策略里没有同步 I/O（会拖慢 execute 本身）

【执行】
□  6. 任务体最外层有 try-catch(Throwable)，尤其是定时任务
□  7. 用 submit 的地方，要么 get()，要么覆写了 afterExecute
□  8. ThreadLocal 在 finally 里 remove()（虚拟线程下考虑 ScopedValue）
□  9. MDC / TraceId / SecurityContext 通过 TaskDecorator 透传
□ 10. 锁内、事务内没有提交任务后又同步等待它（自死锁风险）

【关闭】
□ 11. 有 shutdown() + awaitTermination() 的两阶段关闭
□ 12. shutdownNow() 的返回值被接住并落盘/告警
□ 13. 关闭超时 < K8s terminationGracePeriodSeconds

【隔离与观测】
□ 14. 按失败域隔离，核心链路不与报表/日志共池
□ 15. queueSize / activeCount / rejectedCount 已接入监控告警
□ 16. 没有用 getActiveCount() 等估算值做控制逻辑（只用于观测）
```

### 5. 五个最高频的"以为没用线程池、其实用了"

| 你写的代码 | 实际用的池 | 风险 |
|---|---|---|
| `list.parallelStream()` | `ForkJoinPool.commonPool()`（`CPU-1`） | 全 JVM 共享；放阻塞操作连累所有人 |
| `CompletableFuture.supplyAsync(fn)` | 同上；**单核环境退化为每任务新建线程** | 同上 + 环境差异导致行为突变 |
| `@Async`（Spring Boot） | `applicationTaskExecutor`（**queue 无界**） | 等价 `newFixedThreadPool(8)` |
| `@Async`（纯 Spring） | `SimpleAsyncTaskExecutor` | **完全不池化**，每任务一线程 |
| `@Scheduled` | `taskScheduler`（**默认 poolSize=1**） | 一个任务卡住，全部定时任务停摆 |

> 📌 **这张表是全文最值得截图的一张。** 这五处的共同点是：**你没有写任何 `new ThreadPoolExecutor`，但你的代码正在使用线程池**——而且用的都是不适合生产的默认值。**排查线上问题时，先问一句"这段代码跑在哪个池上"，往往比看十张火焰图更快。**

---

<a id="production-decisions"></a>

## 🏆 终章：从"我懂线程池"倒推到"我在生产做了什么决策"

前面解释了厨房如何运转；这一章只做一件事：把机制翻译成架构评审能签字的决定。读完线程池如果只能背出七个参数，那只是懂零件；能回答"为什么这个池必须有界、为什么这里要用 CallerRuns、为什么 P99 高了不能先扩线程"，才算把零件装成了判断力。

### 先建立决策链：不要从参数开始

```text
业务 SLA（P99 延迟 / 可用性 / 数据不可丢）
  ↓
这批任务是 CPU 密集、I/O 密集，还是混合？
  ↓
真正的瓶颈资源是什么？（CPU / DB 连接 / 下游 QPS / 内存）
  ↓
容量耗尽时，业务上正确的行为是什么？（拒绝 / 排队 / 降级 / 背压）
  ↓
选择：平台线程池 / 虚拟线程 + Semaphore / 响应式
  ↓
定义：并发度、排队上限、拒绝语义、超时、关闭策略
  ↓
压测验证：吞吐、P99/P999、拒绝率、下游饱和度、CPU throttle
  ↓
上线后：监控 + 动态调参能力 + 容量告警
```

> 📌 **决策铁律**：线程池是**容量契约**，不是性能药。如果瓶颈在下游 P99 或数据库连接，调大线程池只会让更多线程一起等——把排队从线程池搬到了下游，还额外消耗了内存和调度。

### 生产决策卡 1：核心交易链路——为什么必须有界且宁可拒绝

**场景**：支付下单接口，P99 SLA 200ms，日常 2000 QPS，大促峰值 20000 QPS。

| 机制判断 | 架构决策 |
|---|---|
| 无界队列 = 延迟无上限 = 违反 SLA | `ArrayBlockingQueue(100)`，容量从 SLA 反推：100/32 × 20ms ≈ 62ms 排队上限 |
| 排队超过 SLA 的任务，执行了也没意义 | 宁可**快速拒绝**让上游降级，也不让任务在队列里烂掉 |
| 默认 `AbortPolicy` 抛异常，调用方必须处理 | 自定义 handler：打点 + 告警 + 返回明确的"系统繁忙" |
| 交易与非交易共池会互相拖垮 | 独立池，独立监控，独立告警阈值 |

```java
@Bean("paymentExecutor")
public ThreadPoolExecutor paymentExecutor(MeterRegistry registry) {
    var pool = new ThreadPoolExecutor(
        32, 64, 60L, SECONDS,
        new ArrayBlockingQueue<>(100),
        new NamedThreadFactory("payment"),
        (r, e) -> {
            registry.counter("pool.rejected", "pool", "payment").increment();
            throw new ServiceBusyException("支付系统繁忙，请稍后重试");
        });
    pool.prestartAllCoreThreads();                     // 消除冷启动毛刺
    ExecutorServiceMetrics.monitor(registry, pool, "payment");   // Micrometer 埋点
    return pool;
}
```

**不能做的错误决策**
- 为了"不丢单"把队列设成 10000——那不是不丢单，是让单子超时后才被处理，做双倍无用功
- 用 `DiscardPolicy` "优雅"处理峰值——静默丢弃支付请求，事故等级最高
- 峰值时手工把 max 从 64 调到 512——先确认瓶颈是不是线程，通常不是

**验收指标**：`rejectedCount`（应为 0 或可解释的尖峰）、`queueSize P99`（应远小于容量）、`activeCount / maxPoolSize`（水位）、端到端 P99、下游 DB/RPC 的 P99 与错误率。**只有这些一起健康才算通过。**

### 生产决策卡 2：异步日志/埋点——为什么这里可以丢

**场景**：用户行为埋点，允许少量丢失，绝不允许影响主链路。

| 机制判断 | 架构决策 |
|---|---|
| 埋点失败不影响业务正确性 | 允许丢，但**必须可观测地丢** |
| 绝不能阻塞主链路 | **禁用 `CallerRunsPolicy`**——它会让业务线程去写日志 |
| 突发流量可以缓冲 | 较大队列（5000）+ 少量线程（2-4） |

```java
new ThreadPoolExecutor(2, 4, 60L, SECONDS,
    new ArrayBlockingQueue<>(5000),
    new NamedThreadFactory("tracking"),
    (r, e) -> DROPPED.increment());   // 静默丢弃，但【打点可见】
```

> 📌 **这是本文唯一推荐"丢弃"的场景**，且必须打点。**"静默丢弃"和"打点后丢弃"是两个完全不同的工程决策**——前者是事故，后者是设计。

**不能做的错误决策**：给埋点池用 `CallerRunsPolicy`（埋点系统抖动会直接拖慢主链路，本末倒置）。

### 生产决策卡 3：批处理 / 数据导出——为什么用 CallerRuns

**场景**：后台批量导出 100 万行，允许慢，但不能 OOM，不能丢数据。

| 机制判断 | 架构决策 |
|---|---|
| 生产者（读 DB）远快于消费者（写文件） | 需要**背压**让生产者慢下来 |
| `CallerRunsPolicy` 让提交者自己执行 = 天然背压 | ✅ 这里提交者是批处理主线程，阻塞它正是我们要的 |
| 数据不能丢 | 不用任何 Discard 策略 |

```java
var pool = new ThreadPoolExecutor(4, 8, 60L, SECONDS,
    new ArrayBlockingQueue<>(50),
    new NamedThreadFactory("export"),
    new ThreadPoolExecutor.CallerRunsPolicy());   // ★ 提交者被迫干活 = 自动限速
```

> 📌 **同一个 `CallerRunsPolicy`，在决策卡 1 是禁忌，在这里是最优解。** 差别只在一个问题：**提交者是谁？** 是 Web 容器线程（阻塞它 = 服务不可用）还是批处理主线程（阻塞它 = 正确的限速）。**拒绝策略没有"最佳实践"，只有"是否匹配你的提交者"。**

### 生产决策卡 4：P99 突刺——什么时候扩线程池，什么时候禁止扩

**先给值班决策树**：

```text
P99 上升
 ├─ 线程池 queueSize 长期 ≈ 0 且 activeCount 也不高？
 │   └─ 是：根本不是线程池问题 → 查 GC / 下游 / 网络 / cgroup throttle
 ├─ queueSize 持续增长，activeCount 打满？
 │   ├─ 单任务耗时是否变长？（下游 P99 上升？）
 │   │   └─ 是：先治下游，扩线程只会让下游更惨 ❌
 │   └─ 单任务耗时正常，纯粹是流量涨了？
 │       └─ 是：才考虑扩容，且要同步确认下游能承接 ✅
 ├─ rejectedCount > 0 但 CPU 很低？
 │   └─ 是：任务在等 I/O → 考虑虚拟线程或增大池（确认下游容量后）
 ├─ CPU 打满 / cgroup nr_throttled 增长？
 │   └─ 是：扩线程反而加剧上下文切换 → 扩 Pod 或优化算法 ❌
 └─ 大量线程停在 getTask() 的 poll/take？
     └─ 池很闲，问题在别处。禁止扩容 ❌
```

| 症状 | 错误反应 | 正确动作 |
|---|---|---|
| 队列堆积 | 无脑扩 max | 先看单任务耗时，通常是下游慢 |
| 频繁拒绝 | 扩大队列 | 先问"这些任务执行了还有意义吗" |
| CPU 打满 | 扩线程池 | 扩 Pod / 优化算法 / 减少任务量 |
| 线程都在 park | 扩线程池 | 看 park 在哪：`getTask()` = 池闲着 |

**排障最小命令集**（完整方法论、真实 dump 输出与症状速查表见 [🔧 线上排查工具箱](#troubleshooting)）：

```bash
# 1. 池是忙还是闲？—— 栈里有 getTask 就是闲（详见工具箱 2.3）
jstack <pid> | grep -c 'ThreadPoolExecutor.getTask'

# 2. 是否被 CPU 配额限流？—— 这个在涨，调线程池全白搭
cat /sys/fs/cgroup/cpu.stat | grep -E 'nr_throttled|throttled_usec'

# 3. 谁在阻塞全场？（Arthas 自动分析持锁者）
thread -b
```

### 生产决策卡 5：Spring `@Async`——那个你从未创建、却一直在用的执行器

**这张卡命中率最高**：它比"用了 `Executors` 工厂方法"更隐蔽，因为绝大多数人**不知道 `@Async` 背后有一个执行器**，更不知道它是谁。

#### 先破除一个流传最广的说法

网上（包括很多中文博客）会告诉你："`@Async` 默认用 `SimpleAsyncTaskExecutor`，来一个任务开一个线程。"

**这句话在纯 Spring Framework 下成立，在 Spring Boot 下不成立——而这个差异本身才是真正的坑。**

```text
纯 Spring Framework（只有 @EnableAsync，无 Boot 自动配置）
  → 找不到 TaskExecutor bean → 回退到 SimpleAsyncTaskExecutor
  → ★ 不池化，每任务 new 一个线程，无上限
  → 正是 Level 1 开篇批判的 new Thread() 反模式

Spring Boot（有 TaskExecutionAutoConfiguration）
  → 自动配置一个 ThreadPoolTaskExecutor
  → ★ 但它的默认参数是：core=8，max=Integer.MAX_VALUE，queue=Integer.MAX_VALUE
```

**看出问题了吗？** 拿这组默认值回到 Level 3 的三道门：

```text
queueCapacity = Integer.MAX_VALUE（实际上无界）
  → 门 2 的 offer() 永不失败
  → 门 3 永远不会被触达
  → maxSize = Integer.MAX_VALUE 完全是装饰
  → 池永远只有 8 个线程，其余任务【无限堆积到堆 OOM】
```

> 📌 **Spring Boot 的默认 `@Async` 执行器，本质上就是一个 `Executors.newFixedThreadPool(8)`** ——正是坑 1 里明令禁止的那个东西，只不过换了个马甲，并且**由框架自动配置，你甚至没有一行代码能让你意识到它的存在**。
>
> 两种默认值的失败模式不同，但都是失败：纯 Spring 是**线程数爆炸**，Spring Boot 是**队列无限堆积**。**没有一种是可以直接上生产的。**

#### 更隐蔽的三个衍生问题

| 问题 | 表现 | 根因 |
|---|---|---|
| **所有 `@Async` 共用一个池** | 发邮件任务卡住 → 订单异步落库也卡住 | 未指定 `@Async("xxx")` 时全部走 `applicationTaskExecutor`，**零隔离** |
| **同类内调用 `@Async` 失效** | 方法同步执行，且无任何报错 | Spring AOP 基于代理，**类内自调用不走代理** |
| **返回 `void` 的异常彻底消失** | 任务失败无日志、无告警 | 需要 `AsyncUncaughtExceptionHandler`，默认只有一行 warn |

#### 决策落地

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    /** 默认执行器：覆盖 Spring Boot 的无界队列自动配置 */
    @Override
    public Executor getAsyncExecutor() {
        var ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(8);
        ex.setMaxPoolSize(32);
        ex.setQueueCapacity(200);                      // ★ 必须有界，否则 max 失效
        ex.setThreadNamePrefix("async-default-");
        ex.setRejectedExecutionHandler(new CallerRunsPolicy());
        ex.setWaitForTasksToCompleteOnShutdown(true);  // ★ 与 L5 的优雅关闭呼应
        ex.setAwaitTerminationSeconds(20);             // ★ 必须 < K8s 宽限期
        ex.setTaskDecorator(new MdcTaskDecorator());   // ★ 透传 TraceId，否则异步日志断链
        ex.initialize();
        return ex;
    }

    /** void 返回值的异常兜底——否则异常彻底消失 */
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            log.error("@Async 方法 {} 未捕获异常", method.getName(), ex);
    }

    /** 按失败域隔离：邮件慢不应拖垮订单 */
    @Bean("mailExecutor")
    public Executor mailExecutor() { /* core=2, max=4, queue=500, 可丢弃 */ }
}

// 使用时【显式指定】，不要依赖默认池
@Async("mailExecutor")
public void sendMail(String to) { ... }
```

**配置文件方式**（适合只需调参、不需多池的场景）：

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 8
        max-size: 32
        queue-capacity: 200      # ★ 这一行是全部要点：把无界改成有界
        keep-alive: 60s
      thread-name-prefix: async-
      shutdown:
        await-termination: true
        await-termination-period: 20s
```

> 📌 **如果这张卡你只记一件事**：`spring.task.execution.pool.queue-capacity` 默认是 `Integer.MAX_VALUE`。**设置它，就等于把 Level 3 的第三道门重新打开。**

#### 不能做的错误决策

- 以为"没配就是没用线程池"——它一直在用，只是你没看见
- 所有 `@Async` 都用默认池，指望它"反正是异步的，慢点无所谓"——**一个慢任务会堵死全部异步能力**
- 用 `@Async` 处理必须可靠的业务（支付回调、库存扣减）——**进程重启时队列里的任务直接蒸发**，这类场景应该用 MQ
- 在同一个类里调用自己的 `@Async` 方法，然后困惑于"为什么没异步"

#### 验收指标

`spring.task.execution` 相关的 Micrometer 指标（`executor.queued`、`executor.active`、`executor.pool.size`）、`@Async` 任务的执行耗时直方图、`AsyncUncaughtExceptionHandler` 的异常计数、以及**应用关闭时是否有任务被丢弃的日志**。

### 生产决策卡 6：JDK 21 迁移——虚拟线程用还是不用

**判断矩阵**：

| 你的场景 | 建议 | 理由 |
|---|---|---|
| I/O 密集 Web 服务，线程池经常打满 | ✅ **强烈推荐虚拟线程** | 最大收益场景，改动小 |
| CPU 密集计算 | ❌ 继续用平台线程池 | 虚拟线程不增加 CPU 并行度，只增加调度开销 |
| 大量 `synchronized` 热路径 + JDK 21 | ⚠️ 先测 pinning | JFR 确认后再决定是否改 `ReentrantLock` |
| 大量 `synchronized` 热路径 + JDK 24+ | ✅ 可直接用 | JEP 491 已消除该类 pinning |
| 重度依赖 ThreadLocal 做上下文 | ⚠️ 评估内存 | 每 VT 一份副本，百万 VT 会放大内存；考虑 `ScopedValue` |
| 遗留代码用了大量 native/JNI | ⚠️ 谨慎 | native 阻塞在任何版本都会 pin |

**迁移路径**（不要一步到位）：

```text
阶段 1：把 Semaphore 加到所有下游资源入口（DB、HTTP、缓存）
        ★ 这一步在平台线程池上就该做，且是虚拟线程的前置条件
阶段 2：挑一个非核心的 I/O 密集接口，切到虚拟线程，观察一周
        指标：P99、carrier 利用率、JFR pinned 事件、下游 permit 等待
阶段 3：确认无 pinning 且指标改善，逐步扩大范围
阶段 4：核心链路最后切，且保留回滚开关（一个配置项切换 Executor）
```

#### Spring 项目具体怎么开

读完前面所有原理，你最可能问的下一个问题是"道理我都懂了，Spring 里怎么落地"。答案分两档：

```yaml
# Spring Boot 3.2+：一行配置，作用于内嵌 Tomcat/Jetty 的【请求处理线程】
spring:
  threads:
    virtual:
      enabled: true
```

这一行同时会影响：内嵌 Web 容器的请求处理、`@Async` 的默认执行器（`TaskExecutionAutoConfiguration` 会改为提供基于虚拟线程的 `SimpleAsyncTaskExecutor`）、`@Scheduled` 的调度器（换成 `SimpleAsyncTaskScheduler`）。

**Spring Boot 3.1 及更早**没有这个开关，需要手动替换协议处理器的执行器：

```java
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
    return handler -> handler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}
```

> ⚠️ **开启前必须先读懂这两条，否则这一行配置会伤到你**：
>
> 1. **它会让 `@Async` 的池化容量阀门消失。** 虚拟线程版的 `SimpleAsyncTaskExecutor` **忽略所有 pool 相关配置**（`queue-capacity`、`max-size` 全部失效）。你在决策卡 5 里辛苦设置的有界队列，会因为这一行配置而**静默失效**。这不是 bug，是设计——但你必须知道，并把阀门改用 `Semaphore` 重新架起来（`SimpleAsyncTaskExecutor` 也提供了 `concurrencyLimit` 可设）。
> 2. **它不改变下游容量。** 请求处理线程无限了，数据库连接池还是那 80 条。**开启虚拟线程的同一个 PR 里，就应该把 Semaphore 加到下游入口**——否则你只是把排队从 Tomcat 线程池挪到了 HikariCP 的等待队列，还顺手删掉了原本能保护下游的那道闸门。
> 3. **它会动摇 `@Scheduled` 隐式依赖了几十年的"不并发"契约**（详见下面一节，这条最隐蔽）。
>
> 📌 这三条正好构成一份完整的检查清单：**换掉线程模型之后，哪些隐式契约会跟着松动。** 第 1 条松动的是容量阀门，第 2 条松动的是下游保护，第 3 条松动的是**任务间的互斥保证**。

#### ⚠️ 第三条展开：`@Scheduled` 的"同一任务不并发"保证会怎样

平台线程时代，`ScheduledThreadPoolExecutor` 提供了一个**免费但极少被显式意识到**的保证。`scheduleAtFixedRate` 的 Javadoc 原文：

> *"If any execution of this task takes longer than its period, then subsequent executions may start late, but will not concurrently execute."*

**同一个周期任务的两次执行，绝不会重叠。** 大量 `@Scheduled` 方法（对账、清理、同步）**从未加过任何锁，却正确运行了很多年**——它们隐式依赖的就是这句话。

换成虚拟线程调度器（`SimpleAsyncTaskScheduler`）后，Spring 官方 Javadoc 的措辞变成了：

> *"using a single scheduler thread and **executing every scheduled task in an individual separate thread**"*
>
> *"…a single scheduler thread, in order to provide traditional fixed-delay semantics! **Prefer the use of fixed rates or cron triggers** instead which are a better fit with this thread-per-task scheduler variant."*

**这里的实际行为比想象中更微妙，`fixedDelay` 和 `fixedRate`/`cron` 走向了两个相反的方向**（以下为 Spring Framework 6.1 时期的实测行为，见 spring-framework issues #31900、#33408）：

| 触发方式 | 平台线程（`ThreadPoolTaskScheduler`） | 虚拟线程（`SimpleAsyncTaskScheduler`） |
|---|---|---|
| `fixedDelay` | 同任务不重叠 | 同任务不重叠（**靠单条调度线程串行化来保住语义**） |
| `fixedRate` / `cron` | 同任务不重叠（JDK 契约保证） | ⚠️ **每次触发都开新线程，同任务执行可能重叠** |
| 不同任务之间 | 受 `poolSize` 限制，可并行 | ⚠️ 曾出现**长任务占住调度线程、拖住其他任务**的问题 |

> 📌 **所以准确的表述不是"保证一定消失"，而是"保证的来源变了，且对 `fixedRate`/`cron` 不再成立"**：
>
> - 平台线程下，"不并发"由 **JDK 的 `ScheduledThreadPoolExecutor` 契约**提供，对所有触发方式一视同仁；
> - 虚拟线程下，`fixedDelay` 的"不并发"由 **Spring 的单条调度线程**兜住（这也是为什么它会连带产生"一个慢任务阻塞其他任务"的副作用）；而 `fixedRate`/`cron` 走的是 **thread-per-execution**，**重叠成为可能**。
>
> ⚠️ 这部分行为是**框架实现细节，随 Spring 版本演进**（相关 issue 已被持续修复调整）。**不要把本表当契约**，要在你的版本上实测：写一个 `@Scheduled(fixedRate = 1000)` 且内部 `sleep(5000)` 的任务，打印 `Thread.currentThread()`，看是否出现重叠。

**迁移前必须做的一件事**：审计所有 `@Scheduled` 方法，回答"**如果它和自己的上一次执行同时跑，会怎样**"。

```java
// ❌ 隐式依赖"不并发"：升级后可能出现两次执行同时对账、重复扣款
@Scheduled(fixedRate = 60000)
public void settle() {
    List<Order> pending = repo.findPending();   // 两个执行可能读到同一批
    pending.forEach(this::doSettle);            // → 重复结算
}

// ✅ 方案一：显式互斥（单机）——不再依赖调度器的隐式保证
private final ReentrantLock lock = new ReentrantLock();
@Scheduled(fixedRate = 60000)
public void settle() {
    if (!lock.tryLock()) { log.warn("上一轮结算未完成，跳过本轮"); return; }
    try { doSettle(); } finally { lock.unlock(); }
}

// ✅ 方案二：分布式锁（多副本部署时【本来就该有】，与虚拟线程无关）
//            ShedLock / Redisson / 数据库唯一键
// ✅ 方案三：让任务本身幂等（最健壮，但改造成本最高）
// ✅ 方案四：保持定时任务用平台线程调度器，只让 Web 层用虚拟线程
@Bean
public TaskScheduler taskScheduler() {
    var s = new ThreadPoolTaskScheduler();      // 显式声明，不受全局开关影响
    s.setPoolSize(4);
    s.setThreadNamePrefix("sched-");
    return s;
}
```

> 📌 **方案四值得特别推荐给保守迁移**：`spring.threads.virtual.enabled=true` 是一个**全局开关**，它同时改掉 Web、`@Async`、`@Scheduled` 三处。而这三处的风险完全不同——Web 层收益最大风险最小，定时任务收益最小风险最大。**显式定义 `TaskScheduler` Bean 就能让定时任务从这个全局开关里"退出"**，这是"想要虚拟线程的好处、又不想承担定时任务并发风险"时最直接的做法。
>
> 这也回到 Level 6 的老结论：**JVM 内的定时池本来就不适合承载关键业务调度。** 如果你的 `@Scheduled` 里跑的是对账、结算这类任务，那它早就该在专业调度框架（XXL-Job、ShedLock）里了——虚拟线程只是把这个一直存在的架构债**提前引爆**了而已。

```java
// 让线程模型可配置，是安全迁移的关键
@Bean
public ExecutorService bizExecutor(@Value("${app.use-virtual-threads:false}") boolean useVt) {
    return useVt
        ? Executors.newVirtualThreadPerTaskExecutor()
        : new ThreadPoolExecutor(32, 64, 60L, SECONDS,
              new ArrayBlockingQueue<>(200),
              new NamedThreadFactory("biz"),
              new ThreadPoolExecutor.CallerRunsPolicy());
}
```

**验收指标**：JFR `jdk.VirtualThreadPinned` 事件数（应接近 0）、carrier 线程利用率、下游 Semaphore 等待时间 P99、连接池 pending、端到端 P99/P999、堆内存（VT 的栈在堆上）。

### 生产决策卡 7：动态线程池——"能改参数"离"动态线程池"还差多远

#### 先破除一个误解

坑 9 讲过 `ThreadPoolExecutor` 支持运行时调参：

```java
pool.setCorePoolSize(16);
pool.setMaximumPoolSize(64);
pool.setKeepAliveTime(30, SECONDS);
```

很多人以为"这就是动态线程池了"。**不是。这只是动态线程池的第一块拼图，而且是最简单的那块。** JDK 只给了"能改"，没给"改得安全、改得可见、改得可追溯"。

```text
JDK 原生能力：            真正的动态线程池还要解决：
  ✅ 改 core/max/keepAlive   ① 参数从哪来？→ 配置中心（Nacos/Apollo）推送，不重启
  ❌ 改队列容量              ② 改了有没有效？→ 多维度监控看板
  ❌ 配置持久化              ③ 什么时候该改？→ 阈值告警（队列积压、拒绝、活跃度）
  ❌ 变更可追溯              ④ 谁改的、改成啥？→ 变更审计与回滚
  ❌ 监控与告警              ⑤ 改错了怎么办？→ 灰度、上下限保护
```

#### 那块 JDK 补不上的拼图：队列容量

```java
// ArrayBlockingQueue 的 capacity 是 final —— 这是"改不了队列"的根因
final Object[] items;   // 数组长度即容量，无法扩缩
```

业界的通行解法是**自定义一个可变容量队列**（DynamicTp、Hippo4j 都是这么做的）：

```java
public class ResizableCapacityLinkedBlockingQueue<E> extends LinkedBlockingQueue<E> {
    // 把 LinkedBlockingQueue 的 private final int capacity 换成非 final
    // 并暴露 setCapacity(int)，配合原有的 notFull 条件唤醒
    public void setCapacity(int capacity) { /* 调整 + signalNotFull */ }
}
```

> 📌 **为什么必须是 `LinkedBlockingQueue` 而不是 `ArrayBlockingQueue`**：前者是链表，容量只是一个计数上限，改数字即可；后者是预分配数组，改容量意味着重新分配并搬迁元素，无法在并发下安全完成。**这是数据结构决定的，不是 JDK 偷懒。**

#### 主流方案对比

| 方案 | 定位 | 关键能力 |
|---|---|---|
| **DynamicTp**（美团系思路开源实现） | 配置中心驱动 | Nacos/Apollo 推送、多框架适配（Tomcat/Dubbo/RocketMQ 的池也能管）、丰富通知 |
| **Hippo4j**（京东系） | 带控制台 | 独立 Server + 控制台、线程池实例管理、运行时监控 |
| **自研**（大厂常见） | 贴合内部体系 | 对接自家配置中心/监控/告警，成本最低但要自己维护 |
| **仅用 Micrometer + 手工调参** | 轻量 | 只有观测，没有推送；适合池数量少的小系统 |

> ⚠️ 上述项目均为社区/公司开源产品，**能力与维护状态随时间变化**，选型前请核对当前版本文档与活跃度。本文只讲**它们共同解决的问题**，不为任何一个背书。

#### 落地要点（比选哪个框架更重要）

```text
① 变更必须有上下限保护：max 不能被误配成 10000（守住机器与下游容量）
② 调参顺序有讲究（见坑 9）：调大先 max 后 core；调小先 core 后 max
③ 效果有延迟：setCorePoolSize 调小后，多余线程在下次 getTask() 才回收
④ 监控必须【先于】调参能力上线 —— 没有看板的动态调参是蒙眼开车
⑤ 变更审计：谁、何时、从什么改到什么、当时的指标是什么
```

#### 不能做的错误决策

- 上了动态线程池，就不再认真做容量推导——**它是应急手段，不是免于设计的借口**
- 把它当性能银弹：如果瓶颈在下游 P99 或锁串行化（[坑 13](#pitfalls)），调多少参数都没用
- 允许在生产随意调参而无审批、无审计、无回滚

#### 验收指标

配置推送成功率与生效延迟、调参前后的吞吐/P99 对比、告警触发到人工介入的时长、**误配拦截次数**（上下限保护起作用的次数）。

### 生产决策卡 8：舱壁隔离——自己分池 vs Sentinel vs Resilience4j

Level 6 的结论是"按失败域隔离"，但**大厂实践里很少手写一堆 `ThreadPoolExecutor` 来做隔离**，而是用现成的舱壁（Bulkhead）框架。

#### 两种隔离方式的本质区别

这两个选项恰好是全文核心论点——**"线程模型"与"资源模型"要分开**——的工业化呈现：

```text
线程池隔离（Thread Pool Bulkhead）
  为每个依赖分配【独立线程池】
  ✅ 强隔离：下游卡死只耗尽自己的池；天然支持超时（调用方不被阻塞）
  ❌ 成本高：每个依赖一套线程 + 上下文切换；ThreadLocal 上下文需传递（见坑 4-B）

信号量隔离（Semaphore Bulkhead）
  共用调用方线程，仅用 Semaphore 限制【并发数】
  ✅ 轻量：无额外线程、无上下文切换、无上下文传递问题
  ❌ 无法主动超时：调用在【调用方线程】上执行，卡住就是卡住
```

> 📌 **虚拟线程改变了这道选择题的答案**。线程池隔离的最大成本是"线程贵"，而虚拟线程让这个前提失效。于是在 JDK 21+：**"每依赖一个虚拟线程 + Semaphore 限并发"同时拿到了两者的优点**——轻量、可超时、强隔离。这正是 Level 7 那句"阀门从池大小搬到信号量"的落地形态。

#### 框架选型

| 方案 | 隔离方式 | 适用场景 | 注意 |
|---|---|---|---|
| **手写 `ThreadPoolExecutor`** | 线程池 | 池数量少（<10）、需要完全掌控参数与拒绝语义 | 熔断/降级/统计要自己写 |
| **Resilience4j** | **两种都支持**（`ThreadPoolBulkhead` / `SemaphoreBulkhead`） | Spring Boot 技术栈、需要熔断+重试+限流+舱壁组合 | 函数式 API，与 CompletableFuture 契合好 |
| **Sentinel** | 并发线程数限流（近似信号量）+ 熔断降级 | 阿里系、需要控制台与规则动态下发、多维度流控 | 强项是流量治理与规则中心，不只是舱壁 |
| **Istio / 服务网格** | 连接池 + 最大待处理请求 | 跨语言、想把治理下沉到基础设施 | 只能管进程外调用，管不了 JVM 内的池 |

```java
// Resilience4j：显式二选一，语义清晰
ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(16).coreThreadPoolSize(8).queueCapacity(100).build();   // 线程池隔离

BulkheadConfig.custom()
    .maxConcurrentCalls(80)                                                     // 信号量隔离
    .maxWaitDuration(Duration.ofMillis(150)).build();
```

#### 决策原则

```text
调用是进程内计算？          → 不需要舱壁，用普通池 + Semaphore 即可
调用是进程外 I/O？ ↓
  需要【主动超时】切断？
   ├─ 是 + 平台线程 → 线程池隔离（Resilience4j ThreadPoolBulkhead）
   ├─ 是 + JDK 21+  → ★ 虚拟线程 + Semaphore（最优解）
   └─ 否，只想限并发 → 信号量隔离（更轻量）
依赖数量 > 10 且需要动态规则？ → 上 Sentinel / Resilience4j，别手写
```

> ⚠️ **一个常被忽略的坑**：用线程池隔离时，**熔断器的统计口径要和池对齐**。如果熔断器统计的是"调用异常率"，而任务在**提交阶段**就被线程池拒绝（`RejectedExecutionException`），这类失败是否计入熔断窗口，各框架处理不同——**必须实测确认，否则会出现"下游已经挂了但熔断器不开"或"池一满就误熔断"**。

### 最终交付：技术评审必须写出的《线程池设计记录》

每一个新建的线程池，评审单至少要有这些字段：

```text
1. 任务画像：CPU/IO 比例？平均与 P99 耗时？是否可重试？是否幂等？
2. 容量推导：core/max/queue 分别从哪个数据推出？（不许写"经验值"）
3. 真实瓶颈：CPU？DB 连接？下游 QPS？—— 线程数是否真的是阀门？
4. 拒绝语义：满了怎么办？谁是提交者？拒绝后业务如何收口？
5. 隔离边界：为什么需要独立的池？它与哪些池共享失败域？
6. 异常处理：任务异常怎么捕获？submit 的 Future 谁来 get？
7. 上下文传递：ThreadLocal 如何清理？MDC/TraceId 如何透传？
8. 关闭策略：优雅关闭多久？shutdownNow 的返回值怎么处理？小于 K8s 宽限期吗？
9. 可观测性：queueSize / activeCount / rejectedCount / 任务耗时直方图 如何上报？
10. 压测结论：吞吐拐点在哪？P99 拐点在哪？拒绝率曲线？下游是否先饱和？
11. 动态调参：是否接入动态配置？调参的安全边界是什么？
12. 版本前提：JDK 版本？是否计划迁移虚拟线程？迁移后本设计哪些字段失效？
```

> 🏆 **最终答案**：从"我懂线程池"倒推回生产，不是说"我知道七个参数"。而是能写下：**因为无界队列会把 OOM 从线程换到堆，所以我用有界队列；因为排队超过 SLA 的任务执行了也没意义，所以我从 SLA 反推队列容量；因为提交者是 Web 线程，所以我不能用 CallerRuns；因为虚拟线程不能凭空制造第 101 条数据库连接，所以我在下游入口放 Semaphore；因为 P99 高时线程都停在 getTask()，所以我拒绝扩容并转去查下游。** 这些才是线程池在生产里真正值钱的判断力。

---

<a id="cross-runtime"></a>

## 🌍 加分视野：这不是 Java 独有——不同运行时如何解决同一座厨房的问题

"任务多、执行者少、必须有容量阀门"不是 Java 特产。不同语言只是把厨房制度放在了不同层。

> ⚠️ 下表用于建立**问题同构性**，不是宣称实现等价。各运行时的调度策略、公平性、内部结构都在演进，不要把实现习惯当语言规范。

| Java 厨房问题 | Go | Node.js | Rust (Tokio) | Erlang/Elixir | 不变的工程不变量 |
|---|---|---|---|---|---|
| 执行者从哪来 | goroutine（M:N，runtime 调度） | 单事件循环 + libuv 线程池 | async task + work-stealing runtime | 轻量进程（BEAM 调度器） | 执行者比 OS 线程廉价，但**不是免费** |
| 并发度阀门 | **无内建**，用 buffered channel 或 `semaphore.Weighted` | `UV_THREADPOOL_SIZE`（默认 4） | `Semaphore` / `buffer_unordered(n)` | 进程池（poolboy 等） | **必须显式声明，没有语言帮你兜底** |
| 排队阀门 | channel 容量 | 事件队列（无界！） | mpsc channel 容量 | 邮箱（无界，需自己限） | 无界队列在任何语言都是 OOM 源 |
| 拒绝/背压 | `select` + `default` 实现非阻塞发送 | 手工实现 | `try_send` / `Semaphore::try_acquire` | 手工实现 | **拒绝语义永远是业务决策** |
| 阻塞调用的隔离 | runtime 自动处理（sysmon 抢占） | **必须**扔给 worker_threads | **必须** `spawn_blocking` | 天然隔离（抢占式调度） | 阻塞操作污染事件循环，是跨语言头号坑 |

### 1. Go：goroutine 廉价，但阀门要自己造

```go
// ❌ 和"虚拟线程可以开 100 万个"是同一个错误
for _, id := range ids {
    go fetch(id)          // 10 万个 goroutine 同时打 DB
}

// ✅ 用带缓冲 channel 当信号量
sem := make(chan struct{}, 80)         // ← 就是 Semaphore(80)
for _, id := range ids {
    sem <- struct{}{}                  // acquire（满了就阻塞 = 背压）
    go func(id int) {
        defer func() { <-sem }()       // release
        fetch(id)
    }(id)
}
```

**Go 和 Java 虚拟线程走到了同一个结论**：执行者廉价之后，**容量阀门必须显式声明**。Go 社区管这叫"worker pool pattern"，Java 21 之后你需要的是同一个东西，只是叫 `Semaphore`。

### 2. Node.js：单线程事件循环，阻塞就是全局停摆

```javascript
// ❌ CPU 密集操作阻塞唯一的事件循环 → 整个服务停止响应
app.get('/hash', (req, res) => {
    const h = crypto.pbkdf2Sync(pw, salt, 1000000, 64, 'sha512');  // 同步！
    res.send(h);
});

// ✅ 扔给 libuv 线程池（默认仅 4 个，可通过 UV_THREADPOOL_SIZE 调整）
crypto.pbkdf2(pw, salt, 1000000, 64, 'sha512', (err, h) => res.send(h));
```

Node 把 Java 里"不要在 `parallel()` 调度器上做阻塞调用"的教训，用最极端的方式教给每个开发者：**只有一个事件循环，堵住就全完。** 这与 Reactor 的 `Schedulers.parallel()` 禁止阻塞是**完全同构**的纪律。

### 3. Rust Tokio：类型系统强制你区分阻塞与非阻塞

```rust
// Tokio 的 work-stealing runtime，和 ForkJoinPool 思想一致
// 阻塞操作必须显式隔离，否则会饿死其他 task
let result = tokio::task::spawn_blocking(|| {
    std::fs::read_to_string("big.txt")     // 同步 I/O
}).await?;

// 并发度阀门
use tokio::sync::Semaphore;
let sem = Arc::new(Semaphore::new(80));
let permit = sem.clone().acquire_owned().await?;
```

Rust 的独特之处：`async fn` 与同步函数在**类型层面**就是不同的东西，编译器帮你发现"在 async 上下文里调用阻塞函数"。Java 没有这层保护——**这正是为什么 Java 的响应式编程更容易出错**：一个不小心的 `jdbcTemplate.query()` 混进 `Flux` 链，编译器不会警告你。

### 4. 跨语言后仍然成立的六条判断力

1. **执行者廉价 ≠ 资源无限**。goroutine、虚拟线程、async task 都不能凭空制造数据库连接。**容量阀门必须显式声明。**
2. **无界队列在任何语言都是 OOM 源**。Node 的事件队列、Erlang 的邮箱、Go 的无缓冲 channel 误用，本质与 `LinkedBlockingQueue` 相同。
3. **阻塞操作必须隔离**。Reactor 的 `boundedElastic`、Node 的 `worker_threads`、Tokio 的 `spawn_blocking`——同一个纪律的三种语法。
4. **拒绝语义永远是业务决策**，没有任何语言能替你决定"满了之后该丢谁"。
5. **优雅关闭需要任务代码配合**。Java 的中断、Go 的 `context.Context`、Rust 的 `CancellationToken`——都只是**信号**，真正停下来取决于任务是否检查它。
6. **能用标准库就别手写调度器**。`ThreadPoolExecutor`、`errgroup`、`tokio::spawn` 里藏着大量你想不到的边界处理（取消、超时、异常、内存序）。

> 🏁 **收尾视角**：Doug Lea 的 `ThreadPoolExecutor`、Go 的 GMP 调度器、Node 的事件循环、Tokio 的 work-stealing runtime、BEAM 的抢占式调度——看起来是五套技术栈，实则都在回答同一个问题：**当待办的工作多于可用的执行者时，谁先做、谁等待、谁被拒绝、以及如何在这一切之上，仍然算得清自己到底能承接多少。** 语言变了，厨房的传送带、席位表和门口的保安没有变。
