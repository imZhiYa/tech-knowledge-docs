# 🔐 AQS 核心机制深度解析 V1：从 CLH 接力到生产架构决策

> **写在最前面的承诺**
>
> 这不是一份“字段 + 方法名”的 AQS 知识点清单。它只讲一件事：**一座只有一个核心工位的工厂，如何让拿不到工位的工人不空转、不漏叫、不插乱，并在原料未到、有人离场、多个许可证、虚拟线程到来时仍然不乱。**
>
> 全文只跟踪一位工人——**徒弟线程 B** 的一次 `ReentrantLock.lock()`。每一层都按 **Why → What → How → Transfer** 推进：上一代方案死在哪，为什么必须长出新结构，新结构怎样在慢动作里工作，又留下什么新账。
>
> 说明一点：从 Level 5 开始，会出现另一批独立角色（X/Y/Z/W 等）去演示共享许可证场景——他们和主线的 B 不是同一条因果链，只是借用工厂的同一套舞台。看到陌生字母不必往主线上套，文中会明确提示。

> 🏷️ `AQS` `CLH` `dummy head` `state` `Condition` `ObjectMonitor` `_cxq` `_EntryList` `_WaitSet` `park/unpark` `Futex` `Virtual Threads`
>
> 🧵 主线 · ⑤-AQS/6 · 同步器的通用等待协议
>
> ①已合 ②已合 ③已合 ④已合 【⑤-AQS 专题】 ⑥

> ⚠️ **版本与证据边界**
>
> - Java 公开 API 语义以 **OpenJDK 21 LTS** 为基线。JDK 8 的 AQS `waitStatus/SIGNAL` 代码会作为“协议骨架”使用；JDK 21 私有 Node 字段、状态位、清理算法已演进，绝不混称为同一份逐行源码。
> - AQS 在 JDK 19 时代的源码中已呈现与 JDK 8 明显不同的 Node/获取路径组织；JDK 21 的 `status` 使用 `WAITING/COND/CANCELLED` 等状态位，而不是 JDK 8 教材中的 `waitStatus/SIGNAL/CONDITION/PROPAGATE`。这些名称只用于定位源码版本，**面试与设计评审应复述等待、取消、传播等不变量，而不是默写私有字段**；Virtual Thread 的 park/unmount 语义需要结合对应 JDK 源码与 JEP 444 验证。
> - `_cxq`、`_EntryList`、`_WaitSet` 是 HotSpot `ObjectMonitor` 的典型实现结构，不是 Java 语言规范；实现可随 JVM、平台和补丁版本改变。
> - 性能量级、工具命令、伪代码均为教学/诊断参考。请以部署 JDK、硬件、内核、cgroup 与压测结果定案。

---

## 🗂️ 目录

> 📌 以下目录使用**显式 ASCII 锚点**，不依赖不同 Markdown 渲染器对中文、emoji、全角符号的 slug 规则；GitHub、VS Code 与常见文档预览均可稳定跳转。

- [📍 使用说明：六级 AQS 能力地图](#aqs-roadmap)
- [🏭 全书唯一工厂地图](#factory-map)
- [🟢 Level 1：为什么不能一直抢工位](#level-1)
- [🟢 Level 2：CLH 前驱接力](#level-2)
- [🟢 Level 3：dummy head 哑节点](#level-3)
- [🟢 Level 4：Condition 两本登记册](#level-4)
- [🟢 Level 5：共享、取消、超时、中断](#level-5)
- [🟢 Level 6：AQS 与 ObjectMonitor](#level-6)
- [🧪 合书自测：一页时序图](#self-check)
- [⚠️ 使用 AQS/Lock 的坑与细节](#pitfalls)
- [📊 AQS 竖切总表 T0–T8](#t0-t8)
- [📚 版本勘误与延伸阅读](#references)
- [🏆 从机制到生产架构决策](#production-decisions)
- [🌍 跨语言 / 跨运行时视角](#cross-runtime)

---

<a id="aqs-roadmap"></a>
## 使用说明：六级 AQS 能力地图

| 层级 | 要打穿的认知墙 | 通关标准 |
|---|---|---|
| L1 | 锁不是一个 boolean，而是资源账本 + 等待协议 | 说清 `state` 为什么能是锁、许可、倒计时 |
| L2 | CLH 不是“一个链表” | 手推 B 为什么只盯前驱，而不是所有人盯 `state` |
| L3 | 哑节点不是装饰 | 解释它如何统一首节点、交班、取消和 GC |
| L4 | 等锁与等条件不是同一种等待 | 画出 `await → signal → 重新 acquire` 的跨队流程 |
| L5 | 真实线程会离场、超时、共享资源 | 说清取消为什么不能挡住后继、共享为何不 `unparkAll` |
| L6 | AQS 不是唯一锁哲学 | 对照 ObjectMonitor 队列，并完成锁边界设计 |

---

<a id="factory-map"></a>
## 全书唯一工厂地图

```text
                    ┌──────────── 核心工位 ────────────┐
                    │ state：占用账本 / 许可证账本      │
                    │ owner：当前正在操作工位的工人     │
                    └──────────────────────────────────┘
                                   ▲
                                   │ tryAcquire 成功才可进入
                                   │
正门同步队列（CLH 变体）           │ release 后给后继一次机会
H(dummy) <── A <── B(徒弟) <── C ──┘
  ▲          ▲      ▲
  │          │      └─ B 只关心前驱 A 是否完成交班
  │          └─ A 成功后会成为新的 head
  └─ 不干活的“值班班长”：统一的队首前驱与进度游标，
     且不是工厂开工就常驻——第一次真正发生排队时才被雇来（见 L3）

侧门 ConditionQueue（等业务条件，不等工位）
C1(等原料) -> C2(等质检)
       signal 后：从侧门转到正门队尾，再重新争工位
```

| 工厂元素 | AQS / Lock 概念 | 它负责的唯一事情 |
|---|---|---|
| 核心工位 | 受保护资源 | 同一时刻谁能操作共享状态 |
| 账本 | `volatile int state` | 记录资源数量/占用/重入数 |
| 工位规则 | `tryAcquire*`、`tryRelease*` | 定义何时能领资源、何时能归还 |
| 正门取号册 | AQS 同步队列 | 组织拿不到资源的候选人 |
| 前一张号牌 | `prev` | 后继的局部依赖与回溯线 |
| 号牌背面“下一位” | `next` | 释放时的快速找后继线 |
| 值班班长 | dummy `head` | 统一首节点、交班与进度推进——班长既是“人”也是“进度指针”，二者是同一件事的两种说法 |
| 睡眠室 | `LockSupport.park` | 失败者不烧 CPU 地等待 |
| 叫号许可 | `unpark` | 让一个候选人重新尝试，不转移所有权 |
| 原料休息室登记簿 | `ConditionObject` | 等业务 predicate，而不是等锁 |

> 🧵 暗线起点：徒弟 B 调用 `lock()`；此刻 B 还没拿号，也没资格进工位。全书不断问：**B 在哪本登记册？state 是多少？谁有责任叫他？**

---

<a id="level-1"></a>
## Level 1：先回答为什么不能一直抢工位

### Why——CAS 失败后一直重试，为什么会把工厂拖垮？

**徒弟**：工位空闲时 `CAS(state, 0, 1)`；失败就再 CAS。很简单，为什么需要一整套 AQS？

**老陈**：你只看到了 A 一个人。现在让 A 在工位里做 2ms 的 RPC，B 到 Z 同时来抢：

```java
while (!compareAndSetState(0, 1)) {
    // 我没事干，但我持续抢同一个 state
}
```

每个失败者都在围攻同一份账本。账本所在 cache line 在多核之间反复争写权限；工位只服务一人，其他人却把 CPU、互连和功耗花在“确认工位仍被占用”上。

| 朴素方案 | 它本来想解决什么 | 它留下的致命账 |
|---|---|---|
| CAS 自旋 | 极短临界区快速抢占 | 长临界区下 CPU 空转、热点 cache line 抖动 |
| `synchronized` | 固定 monitor 互斥 | 资源语义与条件组织方式固定，难自定义共享许可等 |
| 一个全局等待管理员 | 统一排队叫号 | 管理员本身成为串行热点 |
| `notifyAll` | 确保有人醒 | 大量无资格工人一起冲工位，惊群 |

AQS 要解决的不是“让 CAS 永远成功”，而是：**失败后怎样既不烧 CPU，又能让下一位不漏掉机会，同时把“资源能否给”留给上层同步器自己定义。**

### What——`state` 是账本，AQS 是工厂制度

```text
子类（业务账本）                  AQS（通用制度）
────────────────                  ─────────────────────
state=重入次数         ───────►    排队、park、unpark、取消
state=剩余许可证       ───────►    不知道“1”代表什么
state=未完成任务数     ───────►    只知道 try* 成败与传播
```

| 同步器 | `state` 的业务解释 | `tryAcquire*` 要回答的问题 |
|---|---|---|
| `ReentrantLock` | 0 空闲；正数为 owner 的重入层数 | 我是空闲工位或同一 owner 吗？ |
| `Semaphore` | 剩余 permit | 仓库还有足够许可证吗？ |
| `CountDownLatch` | 剩余未完成数 | 所有前置任务是否归零？ |
| 读写锁 | 读计数与写计数的编码 | 当前读写状态是否兼容？ |

> 📌 AQS 不知道 B 是想拿锁还是拿许可；它只要求子类给出一个可靠答案：**现在能否获取？释放后是否真的可交接？**

### How——B 第一次 `lock()`：快路径为什么没有队列？

```java
// 教学伪代码，展示职责边界
public void lock() {
    if (casState(0, 1)) {
        owner = currentThread();        // B 若成功，直接进工位
    } else {
        acquire(1);                     // 失败才进入 AQS 慢路径
    }
}
```

慢动作：A 已持锁，`state=1`。B 的 CAS 失败；B 不该继续转圈。它要去正门取号。若 A 恰好就是 B 自己，`tryAcquire` 会把重入数 1 改成 2；B 第一次 `unlock` 只减到 1，工位还没交班；第二次归零才可以推进后继。

这里还有一个容易被忽略的事实：**这一刻工厂里根本没有队列，也没有班长。** 无竞争时，AQS 不会预先摆一个空队列等着——`head`、`tail` 此刻都是 `null`。队列结构是被“第一次真正的竞争”催生出来的，不是工厂开工自带的编制。这一点会在 L3 详细展开。

### Transfer——先分资源账本，再谈锁

设计锁之前先问：

1. **我保护什么不变量？** 一组字段、库存、连接、任务完成状态？
2. **资源能多人同时持有吗？** 不能是独占；能是共享许可。
3. **失败者应等“工位空”还是等“业务条件成立”？** 后者不是普通锁等待，L4 才能解决。

L1 留下的问题是：B 去正门取号后，为什么不让所有人继续看账本？B 又该盯谁？这逼出 CLH。

> 🔴 **口诀**：`state` 是账本不是锁；`try*` 判资源，失败才取号。

---

<a id="level-2"></a>
## Level 2：CLH 核心——前驱接力而非全体围攻

### 👶 前置知识关卡

- [ ] 能解释为什么失败 CAS 自旋会把所有核聚集到一个热点吗？
- [ ] 知道 AQS 不决定 `state` 的业务含义吗？

L1 已经交代清楚：CAS 失败后 B 要调用 `acquire(1)` 走慢路径。这一层只回答一个问题——**`acquire(1)` 内部，B 到底该盯着谁等？**

### Why——为什么 B 不能“随便 park”，必须排到 A 后面？

B 若 CAS 失败就直接 `park()`，会有三个无人回答的问题：B 睡在哪里？A 释放时怎么知道叫 B？C 超时离场后谁接替？更致命的是：A 可能在 B 真正睡着前释放，导致 B 后睡、再没人叫。

原始 CLH 的回答是：**别让所有等待者盯工位账本；让每个等待者只盯前一张号牌。**

```text
全局围攻（坏）：A/B/C/D 全部反复读写 [state]

前驱接力（好）：H <- A <- B <- C
                         ↑    ↑
                    B 只关心 A  C 只关心 B
```

**老陈**：CLH 的灵魂不是“有链表”，更不是“FIFO”三个字；是**把全局资源竞争拆成相邻节点的局部交班责任**。原始 CLH 是自旋锁，AQS 将它改造成阻塞队列变体：短暂协调，确认可睡后 park。

### What——原始 CLH、MCS 与 AQS 的边界

| 维度 | 原始 CLH | MCS | AQS 同步队列 |
|---|---|---|---|
| 等待者主要观察 | 前驱节点状态 | 自己节点状态 | 前驱关系决定资格；实际用 park 等待 |
| 本质 | 队列自旋锁 | 队列自旋锁 | 阻塞同步框架 |
| 释放后 | 前驱状态变化被后继观察 | 前驱写后继本地标志 | 后继被唤醒后重新 `tryAcquire` |
| 取消/超时/Condition | 非核心 | 非核心 | 核心协议 |

所以准确表述是：**AQS 使用借鉴 CLH/MCS 思想的阻塞同步队列变体，不是原始 CLH 锁。** 这套设计思路的一手出处是 Doug Lea 的论文 *The java.util.concurrent Synchronizer Framework*（2004），感兴趣可以对照原文看当初的取舍理由，而不是只看后人转述。

#### Why 再挖一层：AQS 为什么更像 CLH 变体，而不是把 MCS 原样搬进来？

先纠正提问：AQS 并没有在“纯 CLH”与“纯 MCS”之间二选一；它为了阻塞、取消、Condition 和共享传播，已经不是任一原始自旋锁。但它的**排队资格判断**更自然地继承了 CLH 的前驱视角。

```text
MCS 的核心：前驱必须可靠地找到并写后继的本地 waiting 标志；
              后继主要自旋自己的节点。

AQS 的核心：后继通过 prev 知道“我排在谁后面”；
              只有 predecessor == head 才获得重新 tryAcquire 的资格；
              若 next 尚未补齐或已取消，仍能从 tail 沿 prev 回溯恢复。
```

因果在于：AQS 的入队只把 **tail CAS** 设为线性化点。`node.prev=pred` 在提交前已建立，因此提交成功后新节点立刻拥有可靠的回溯与取消修复依据；`pred.next=node` 是随后补的叫号快链，允许暂时缺失。原始 MCS 更依赖“前驱向我写入/链接”的后继协作，作为纯自旋锁很好；但 AQS 还要容忍线程 park、超时取消、Condition 转队、共享传播，不能把正常进度完全押在前驱即时完成对后继字段的写入上。

> 📌 所以别背成“CLH 比 MCS 更快”。准确的白板答案是：**AQS 选择以 prev 前驱关系作为资格与恢复主线，再用 next 提供常态唤醒快路径；这个不对称设计让单一 tail CAS、取消回溯和阻塞唤醒能同时成立。** 具体 cache 效果、字段布局和 JDK 私有实现必须实测/看版本。

### How——B 如何拿号：先 `prev`，再 tail CAS，后补 `next`

为使因果看得最清楚，先用 **JDK 8 形态**慢放。JDK 21 的私有方法和状态位已重构（见前言的 JDK 19 重写说明），但“单一入队提交点 + 前驱回溯 + 正向快链”的设计不变量不变。

```text
初态：H 是 dummy head，tail=H
（注：若这是工厂第一次发生竞争，H 本身此刻才被懒创建出来——见 L3）

① B：B.prev=H
② C：C.prev=H
③ B：CAS(tail, H, B) 成功      ← B 正式入队的线性化点
④ C：CAS(tail, H, C) 失败
⑤ B：H.next=B                  ← 快链，允许晚于提交点
⑥ C：读 tail=B；C.prev=B
⑦ C：CAS(tail, B, C) 成功      ← C 正式入队
⑧ C：B.next=C
```

| 设计动作 | 工厂动作 | 为什么不可少 |
|---|---|---|
| `node.prev=pred` | 先把手系到前一张号牌 | 提交后马上有可靠回溯线 |
| CAS tail | 原子换掉“最后一位”牌 | 一个提交点决定排队先后 |
| `pred.next=node` | 在前一张号牌背面补“下一位” | 给叫号快路径，但不承担唯一正确性 |

**徒弟**：`next` 都有了，为什么释放时还要从 tail 沿 `prev` 回找？

**老陈**：看第③与第⑤之间。tail 已经是 B，H.next 却可能还是空。H 若在这时释放，只信 `next` 会漏掉 B。旧版 `unparkSuccessor` 的反向扫描正是在给这个瞬态擦屁股：**prev 是提交前就设好的恢复链；next 是常态快速叫号链。**

#### park 前为什么必须握手？

```text
错误时序：
B：准备睡
A：释放，没看到 B 已声明“我会睡”，没叫 B
B：现在 park
=> 工位空了，B 却睡死
```

正确协议是：B 不能一入队就睡；它先确认前驱/队列状态已经表达“释放时请叫后继”，才 `park`。旧 JDK 8 里常由前驱的 `SIGNAL` 表达；JDK 21 使用 `WAITING` 等新状态位和当前 acquire 协议完成同方向握手。

`unpark(B)` 若早于 `park(B)` 也不会丢：`LockSupport` 对每线程保留至多一张 permit，后续 park 消费 permit 后立即返回。但 permit 不能取代队列协议——它无法回答“该叫谁”“取消者怎么跳过”。

### Transfer——CLH 的四条工程方法

1. **局部化等待，不等于没有共享热点**：`state/tail/head` 仍会竞争，但不再让所有失败者长期轰 state。
2. **一个线性化点优于全字段强一致**：tail CAS 定资格，其他链接最终补齐，罕见路径回扫兜底。
3. **通知只给重试机会**：B 被叫醒还得 `tryAcquire`；消息系统/任务编排也应重验条件。
4. **快路径干净，恢复路径完整**：取消、next 暂空不污染常态，却必须保证进度。

B 已经在 A 后排好，但“第一个真实排队者前面是谁”“A 成功后队首如何推进”仍没有统一答案。下一层的哑节点不是小细节，而是这套协议的铰链。

> 🔴 **口诀**：只盯前号不围账；先系 prev 再换 tail；next 快叫，prev 兜底；先承诺再睡觉。

---

<a id="level-3"></a>
## Level 3：dummy head——为什么队首必须坐着一位不干活的班长

### 👶 前置知识关卡

- [ ] tail CAS 为什么是入队提交点？
- [ ] 为什么 next 可能暂时落后于 tail？
- [ ] 为什么 B 被 unpark 后仍必须重试获取？

### Why——没有哑节点，第一个人会让整套协议裂成两套

没有 H 时，第一位真实等待者 A 的 `prev=null`。于是每个核心操作都冒出特殊分支：

```text
A 排第一：A 的前驱是谁？
释放者：谁负责找/叫第一个真实等待者？
A 成功：队首要设为 null 还是 A？
A 取消：后继从哪里开始回溯？
```

这不是多写几个 `if` 的问题。并发协议最怕“首节点一套规则，其他节点一套规则”；取消、唤醒、入队交错后，特殊分支会相乘。

### What——H 是“已交班进度游标”，不是当前 owner，也不是常驻编制

```text
工位中：A 正在干活，B/C 在等

head=H(dummy) -> B -> C     owner=A

A release 后：H 负责让 B 获得一次重试机会
B 成功后：head=B -> C       owner=B
旧 H 可断开引用
```

| dummy head 做什么 | 它消除的边界问题 |
|---|---|
| 给第一个真实 waiter 一个统一前驱 | `node.prev == head` 成为统一“排第一”判断 |
| 做释放路径的起点 | 释放者从 head 推进可用后继 |
| 做队列进度游标 | 成功者变新 head，队列整体前移 |
| 断开旧引用 | 旧 head 与 waiter 引用可清理，利于 GC |
| 让取消回溯有统一边界 | 从 tail 沿 prev 回找终止于 head |

> 📌 **最容易错的点一**：`head` 不是“当前持锁线程”的权威记录。`ReentrantLock` 的 owner 由其 Sync/AbstractOwnableSynchronizer 维护；head 只表示**已经完成一次队列交接的进度位置**。
>
> 📌 **最容易错的点二**：班长不是工厂开工就雇好、常年坐在那的角色。空队列时 `head == tail == null`；**创建 H 的不是已经持锁的 A，而是第一个获取失败、被迫入队的线程 B**。B 进入入队慢路径，在发现 `tail == null` 时竞争性地 CAS 初始化一个空 Node 为 `head`，再令 `tail=head`；随后 B 才把自己的真实 Node 接在 H 后。若另一个失败线程 C 同时初始化，只有 CAS 成功者完成建队，失败者重读队尾再接入。这与 L1 的“无竞争快路径不建队列”是同一件事的两面：**没有竞争就没有班长，也没有登记册**。

### How——B 从“排第一”到“成为新班长”的完整慢动作

```text
初态：owner=A，state=1
      H(dummy) -> B -> C

① A unlock：tryRelease 使 state=0，owner 清空
② H 的后继 B 被 unpark：B 获得一次重试机会
③ B 醒来：发现 B.prev==head(H)，所以优先 tryAcquire
④ B CAS state:0→1 成功；owner=B
⑤ B 变为新 head；旧 H 的关联断开
⑥ C 的有效前驱现在是 B(head)
```

注意第②与第④之间允许新来者竞争：非公平 `ReentrantLock` 可能让路人 N 在 B 获得 CPU 前先 CAS 成功。**排第一只给 B 优先尝试资格，不是把工位产权预先过户。** 公平锁会在空闲时检查 `hasQueuedPredecessors()`，减少这种插队；它也不承诺 OS 调度意义的绝对 FIFO。

#### 取消为什么仍能自愈？

```text
H -> A(X cancelled) -> B -> C

B 发现前驱 A 已废：沿 prev 跳过 A，重新把自己依附到有效前驱
释放者发现 next 不可靠/已取消：沿 prev 从 tail 回找可 signal 的候选
```

哑节点提供统一的“回溯到这里为止”边界。JDK 8 可从 `cancelAcquire/unparkSuccessor` 读到这类逻辑；JDK 21 使用不同的私有清理组织（如 `cleanQueue`）。不能背函数名，必须守住不变量：**废号不能永久挡住活号。**

### Transfer——哨兵节点的通用价值

dummy head 是一种把“零/一/多”统一成“总有前驱”的设计。链表虚拟头、树虚拟根、工作流虚拟开始节点都在做同一件事：**用一个无业务含义的节点，买掉最危险的边界分支。**

B 现在能等工位了，可 B 还会遇到另一种情况：它已经拿到工位，却发现原料没到。此时若 B 继续占工位等原料，生产者永远进不来。于是必须有第二本登记册。

> 🔴 **口诀**：head 是进度不是 owner，也不是常驻编制；哑班长不干活，却让人人都有前驱；成功即前移，废号可回溯。

---

<a id="level-4"></a>
## Level 4：两本登记册——Condition 为什么必须离开主队列

### 👶 前置知识关卡

- [ ] 能区分 owner 与 head 吗？
- [ ] 为什么排第一不等于已拥有锁？
- [ ] 为什么取消节点必须被跳过？

### Why——“等工位”和“等原料”为什么不能记在同一本册子？

消费者 B 已获得工位，发现 `queue.isEmpty()`。它真正等的不是“锁空闲”，而是“生产者放入数据”。如果 B 留在正门同步队列，还占着锁，生产者无法进入工位把数据放进去；如果 B 释放锁却仍在正门队列，它会在每次工位空闲时醒来抢锁，徒增无效竞争。

**徒弟**：那就 `wait()` 一下？

**老陈**：可以，但那是 `ObjectMonitor` 的 WaitSet 模型。AQS/Lock 选择把“同步竞争”与“业务条件”解耦：每个 Lock 可以有多个 Condition，每个 Condition 是一本专用登记册。

### What——主队列只等资源；侧队列只等 predicate

```text
正门同步队列：等“我何时有资格重试拿工位”
H -> A -> C

侧门 ConditionQueue：等“业务条件何时成立”
B(等非空) -> D(等非空)
E(等非满)              // 可以是另一份 ConditionQueue
```

| 问题 | 同步队列 | ConditionQueue |
|---|---|---|
| 等什么 | 锁/许可可否获取 | 某个业务 predicate 是否成立 |
| 是否持锁进入 | 失败获取后进入 | 必须已独占持锁才能 `await/signal` |
| 醒后是否可直接执行 | 不可，仍需 `tryAcquire` | 不可，先转入同步队列再竞争 |
| 链接 | AQS Node prev/next | ConditionNode 的 `nextWaiter` 单链 |

> ⚠️ `Condition` 不依赖 `Object.wait()`；AQS `ConditionObject` 是自己的等待队列与 `LockSupport` 协议。两者最深的方法论却相同：**通知不是资源所有权。**

### How——B 的 `await → signal → reacquire`：完整跨队慢动作

```text
初态：B 已重入两层锁，state=2，predicate=队列非空=false

① B 创建 ConditionNode，加入“非空”侧门登记册
② B fullyRelease(savedState=2)：完整释放两层，而不是只减一层
   → state=0，正门后继得到进工位机会
③ B park：此时 B 不在正门竞争工位，而在侧门等原料
④ 生产者 P 获锁，put(item)，把 predicate 改为 true
⑤ P 在同一把锁保护下 signal(notEmpty)
⑥ B 的节点从 ConditionQueue 转移到正门同步队列尾部
⑦ P unlock；B 被推进后重新 tryAcquire
⑧ B 成功后恢复 savedState=2，回到 await 返回处
⑨ B 再次检查 while(predicate)；只有条件真的成立才 consume
```

```java
lock.lock();
try {
    while (queue.isEmpty()) {   // 不能 if
        notEmpty.await();
    }
    return queue.remove();
} finally {
    lock.unlock();
}
```

为什么必须 `while`？signal 后 B 可能长期未被调度；其他消费者可能先拿走数据；还允许虚假唤醒、超时和中断交错。**锁保护的是“检查 predicate + 修改 predicate”的原子关系，不是“某次 signal 必定兑现”。**

### Transfer——把“通知”设计成状态迁移，不是回调

| 错误想法 | AQS/Condition 的纠正 |
|---|---|
| signal 就等于条件已永久满足 | signal 只将候选者转入资源竞争路径 |
| 被唤醒就直接执行业务 | 先重新拿锁，再在 while 中复检 predicate |
| 一个等待队列够所有事情用 | 资源竞争与业务条件的唤醒原因不同，应分册 |
| await 只释放一次即可 | 必须完整释放重入状态，否则生产者进不来 |

L4 把独占锁与条件协作讲完整了，但许可证、门闩等不是“一次只能放一人”；而真实等待者会被中断、超时、取消。下一层要让工厂在这些破坏性事件下继续前进。

> 🔴 **口诀**：正门等工位，侧门等条件；await 全放锁，signal 只转队；醒来先抢锁，条件必须 while。

---

<a id="level-5"></a>
## Level 5：完整生产协议——共享、取消、超时和中断

### 👶 前置知识关卡

- [ ] 为什么 Condition signal 后仍要回正门同步队列？
- [ ] 为什么 `while` 比 `if` 正确？
- [ ] 为什么 `await` 需要完整释放重入数？

> 从这里开始，共享许可的例子会引入一批新角色（X/Y/Z/W），他们是货架许可证的独立等待者，和主线的 B（打包台的独占锁等待者）不是同一条叙事线，只是共用同一个工厂场景。全景复盘会把这两条线重新对齐到一起。

### Why——独占交班为什么仍不足？

仓库有 3 张许可证时，独占模式只叫一个人会浪费资源；但把所有人叫醒又会制造惊群。另一个现实是：等待者可能超时离开，可能被中断，也可能从 ConditionQueue 被 signal 与中断同时撞上。工厂不允许一张废号卡住整条队。

### What——共享传播与离场合同

| 模式/事件 | 工厂含义 | AQS 关键语义 |
|---|---|---|
| 独占 | 一个工位一次一个人 | `tryAcquire` true/false |
| 共享 | 多张许可证 | `tryAcquireShared`：负=失败；0=成功但不宜传播；正=成功且还有余量 |
| 超时 | 取号有效期到 | 取消并让队列恢复进度 |
| 可中断获取 | 接到停工通知 | 取消等待，抛 `InterruptedException` |
| 不可中断获取 | 先完成交班再处理通知 | 记录中断，成功后补回 interrupt 标记 |

### How——许可传播与废号自愈

#### 1. 共享许可：不 `unparkAll` 的三步

许可证如何传播只需要一条主干逻辑，下面在全景复盘里会用一个更完整的四人版本再走一遍，这里先给最小闭环：

```text
初态：Semaphore permits=2；正门 H -> X -> Y -> Z

① X 获取：permits 2→1，仍有余量；传播，让 Y 有机会尝试
② Y 获取：permits 1→0，余量耗尽；停止传播
③ Z 继续等待；某次 release 使 permits 0→1，再推进 Z
```

共享模式的本质是：**被叫醒的人自己 CAS 扣减账本；成功结果告诉协议“还有没有必要继续叫下一位”。** 旧版 `doReleaseShared/setHeadAndPropagate/PROPAGATE` 是理解这一点的好历史入口；JDK 21 私有实现不同（见前言的 JDK 19 重写说明），不能再把 `PROPAGATE=-3` 说成当前 Node 常量。

#### 2. 取消：一个等待者走了，为什么后继不能傻等？

```text
正常：H -> A -> B -> C
B 超时：H -> A -> B(X) -> C

C：发现前驱 B 已取消，沿 prev 跳过废号，建立有效前驱
释放者：next 若为空/废，沿 tail.prev 回扫找到可 signal 的候选
清理：逐步断开废节点连接，减少无用引用与后续成本
```

取消协议的目标不是“立刻把链表修成绝对漂亮”，而是两条：

1. **安全**：已取消者绝不能又获得资源；
2. **活性**：取消者绝不能永久挡住活后继。

#### 3. 中断：为什么 `lock()` 与 `lockInterruptibly()` 不同？

```text
lock()：即使 park 时被中断，也继续完成获取；内部清标记、记录，成功后 selfInterrupt。
lockInterruptibly()：被中断就取消等待，抛 InterruptedException。
tryLock(timeout)：死线到或中断，分别返回 false 或抛异常。
```

“不响应中断”不等于“吞掉中断”。AQS 的“清 + 记 + 补”是内部循环可继续与外部合同不丢失之间的折中。

#### 4. 共享传播的最后一个避坑：释放者不负责“叫醒全部”

共享模式防漏唤醒的精髓不是释放者递归扫描整条队列，而是**接力传播**：释放动作推进一个适格后继；该后继成功扣减 state 后，若返回值表明仍有余量/仍应传播，它再推进下一位。这样既不会因并发 release 与后继 park 的时序而丢失可用许可，也不会一次把全队拉起来制造惊群。

```text
release 只负责把“货架上可能还有货”的事实交给第一位候选
    → X 成功 acquireShared，发现还有货
        → X 继续推进 Y
            → Y 成功后发现货空，传播自然停止
```

旧 JDK 8 的 `PROPAGATE` 是为保证这类共享进度而存在的历史实现线索；JDK 21 不应照背该常量，但要守住这个语义：**任何一次可用许可的产生，最终不能因竞争时序而永远困在队列前面。**

#### 5. 两个常被漏掉的 API 合同坑

**无参 `tryLock()` 会破坏公平队列。** 即使构造的是 `new ReentrantLock(true)`，`tryLock()` 也会立即尝试获取，文档明确说明它不遵守公平设置；它适合“抢不到就立即走”的业务。若需要“零等待但尊重公平队列”的语义，应使用 `tryLock(0, TimeUnit.SECONDS)`（它是可中断的定时获取合同，调用方必须处理 `InterruptedException`）。

**Semaphore 许可可以被错误释放到初始值之上。** `release()` 不知道这张许可证是不是当前线程真正 acquire 得到的；重复 release 会让 `state` 增大，造成下游并发边界失效。对每一次成功 acquire，必须建立唯一、可审计、`finally` 配对的 release 路径；必要时封装资源租约，而不是把 Semaphore 裸暴露给任意业务代码。

### Transfer——生产同步器的健壮性检查表

- [ ] 共享资源是否确实能并发持有？许可数是否来自下游真实容量？
- [ ] 每个等待 API 的中断合同是否明确？
- [ ] 超时/取消后，后继能否仍然前进？
- [ ] Condition 的 predicate 是否与 signal 在同一锁保护下？
- [ ] 是否误把“被唤醒”当成“业务条件已满足”？

L5 已说明 AQS 怎么做自己的队列协议；但架构选型还必须回答：为什么不用 `synchronized`？它底层的 `_cxq/_EntryList/_WaitSet` 又是怎样的排队哲学？以及 Virtual Threads 到来后，什么资源仍然稀缺？

> 🔴 **口诀**：共享看余量，余量才传播；取消先让路，中断看合同；队列不求立刻漂亮，只求后继能前进。

---

<a id="level-6"></a>
## Level 6：架构师篇——AQS 与 ObjectMonitor 的两套排队哲学

### 👶 前置知识关卡

- [ ] 共享模式为什么不能粗暴 `unparkAll`？
- [ ] 取消节点的两个不变量是什么？
- [ ] `lock()` 与 `lockInterruptibly()` 如何处理 interrupt？

### Why——为什么“会 AQS”不等于会选锁？

错误选型通常来自一个错觉：`synchronized`、`ReentrantLock`、`Semaphore` 只是不同语法。其实它们的资源模型、排队入口、条件队列、诊断方式和可扩展边界都不同。

架构师不该先问“哪个快”，而该问：**我要保护什么不变量？失败者等资源还是等条件？资源是否允许共享？临界区是否含慢活？竞争是否能按 key 分片？**

### What——同一个问题下，ObjectMonitor 与 AQS 如何回答？

先声明边界：下表的 `_cxq`、`_EntryList`、`_WaitSet` 描述 HotSpot `ObjectMonitor` 的典型内部模型，是理解实现的好地图；**它们不是 Java 规范，不应写成跨版本绝对承诺。**

用同一套工厂语言先对齐一遍：如果说 AQS 是“正门取号 + 侧门等原料”，那 `synchronized` 的工厂长这样——

```text
_cxq：工位门口一个没有号码牌的“临时挤门区”，新来的人直接挤在最前面
      （典型实现是后进先出的栈，谁刚挤进来谁离工位最近）
_EntryList：监视器内部认为“下一批有资格重新竞争”的候选人名单，
           由监视器自己的策略决定何时把 _cxq 里的人搬过来整理
_WaitSet：喊了 wait() 之后去的休息室，等的是业务条件，不是工位——
          和 AQS 的侧门 ConditionQueue 是同一个角色，只是换了一套实现
```

区别在于：AQS 把“正门怎么排、谁先谁后”这套规则做成了你能读到源码、能扩展的显式协议；`synchronized` 把等价的规则做成了 JVM 内部黑盒，你只能观察行为，不能定制。

```text
synchronized / ObjectMonitor（典型 HotSpot 概念图）

新竞争者 ──> _cxq（挤门区，常见栈式/LIFO 倾向）
                   │ 监视器按实现策略整理/交接
                   ▼
              _EntryList（等待重新竞争 monitor 的候选名单）
                   │ monitor 可用时
                   ▼
              重新竞争 owner

Object.wait() ──> _WaitSet（休息室，等待业务条件）
notify/notifyAll ─> 从 WaitSet 移向可重新竞争 monitor 的路径

AQS / ReentrantLock

新竞争者 ──> tail CAS → CLH 同步队列（正门取号，前驱局部接力）
                          │ release/signalNext
                          ▼
                       重新 tryAcquire state

Condition.await() ──> ConditionQueue（侧门，等待业务条件）
signal() ────────────> 转入 AQS 同步队列，再重新 tryAcquire
```

| 同一问题 | `synchronized` / ObjectMonitor | `ReentrantLock` / AQS |
|---|---|---|
| 资源状态 | 对象头、轻/重量级锁与 ObjectMonitor 等 JVM 路径 | 子类自定义 `state` |
| 竞争入口 | `_cxq` 等 monitor 竞争结构（挤门区） | tail CAS 入 AQS 同步队列（正门取号） |
| 队列主思想 | JVM 内部 monitor 策略管理 | 显式前驱依赖、CLH 变体接力 |
| 重新竞争者 | `_EntryList` 等待 monitor 机会 | head 后继优先得到 `tryAcquire` 机会 |
| 条件等待 | `Object.wait()` → `_WaitSet` | `Condition.await()` → ConditionQueue |
| 条件通知后 | `notify` 不直接交 monitor，仍要竞争 | `signal` 不直接交锁，转同步队列再竞争 |
| 可定时/可中断获取 | monitor API 不直接提供 | `tryLock(timeout)`、`lockInterruptibly()` |
| 多条件队列 | 一个对象 monitor 的 wait-set 语义 | 一个 Lock 可创建多个 Condition |
| 自定义同步器 | 不适合作为业务扩展框架 | AQS 钩子可实现独占/共享状态机 |

**徒弟**：`_cxq` 是 LIFO，AQS 是 FIFO，所以 AQS 一定公平？

**老陈**：这是第二个坑。HotSpot monitor 的具体入队/重排策略是实现细节；AQS 的同步队列也只提供候选秩序，非公平 Lock 仍允许新来者在释放窗口 CAS 插队。真正的结论是：**两者都不是“绝对 OS 调度 FIFO”；AQS 把公平策略和资源语义暴露为可组合框架，ObjectMonitor 把更多策略留在 JVM 内部。**

### How——从 `lock()` 到 futex，再到 Virtual Thread

```text
ReentrantLock.lock()
 → AQS.tryAcquire / enqueue / park
 → JDK 21 AQS 的内部原子操作使用 `jdk.internal.misc.Unsafe` 相关机制（这是私有实现细节，不是应用代码应依赖的 API）
    → HotSpot 将热点原子操作编译为对应平台的原子指令
 → HotSpot JIT intrinsic
 → x86 常见 lock cmpxchg；AArch64 常见 LL/SC 或 LSE（架构相关）
 → cache coherence 获取写权限
 → LockSupport.park
 → 平台线程在 Linux 上通常经 JVM/POSIX 线程库，可能使用 FUTEX_WAIT/FUTEX_WAKE
 → 被唤醒成为 runnable；调度器决定何时实际运行
```

这是一条**常见 Linux/HotSpot 竖切**，不是每次必经的规范路径：已有 permit、虚假唤醒、超时、JIT 状态、OS 实现、Virtual Thread 都会改变细节。

JDK 21 Virtual Thread 下，B 在适合的 `park` 点通常可从 carrier 卸载：

```text
B(virtual) park → continuation 卸载 → carrier 去运行别的 virtual thread
A release/unpark(B) → B 成为可运行候选 → 某个 carrier 恢复 B → B 重新 tryAcquire
```

这也是为什么前言要单独强调 JDK 19 那次 AQS 重写——旧的 `waitStatus` 整型状态机是围绕“平台线程 park/unpark”设计的；虚拟线程需要在 park 点判断能否安全卸载 continuation，新的位状态字段正是为这个判断服务的。不变的是：state 仍可能热点，锁内 RPC 仍会阻塞进度，下游数据库连接仍只有有限数量。Virtual Thread 降低“等待占 OS 线程”的成本，**不增加数据库、CPU、第三方 API、锁保护数据结构的容量**。

#### 锁的方法论：五问后才准选 API

| 问题 | 设计动作 | 常见错误 |
|---|---|---|
| 1. 保护什么不变量？ | 先画字段/状态转换，再定锁边界 | 先选锁，再把一大片代码塞进去 |
| 2. 等资源还是等条件？ | 资源用同步队列；条件用 Condition/WaitSet | 用轮询或一个混乱队列表达所有等待 |
| 3. 独占还是共享？ | 锁、读写、Semaphore、Latch 匹配容量 | 用线程池大小冒充下游限流 |
| 4. 锁内有没有慢活？ | RPC/SQL/I/O/回调移锁外，或拆状态机 | 先换公平/非公平，根因不动 |
| 5. 热点能否分片？ | 按 key striping、分桶、局部队列 | 所有订单/租户共用一把全局锁 |

#### 选型矩阵：先按业务约束，不按“哪个更高级”

| 场景 / 约束 | 首选 | 机制依据 | 禁忌与审查点 |
|---|---|---|---|
| 高并发、极短内存更新（计数、指针） | `Atomic*`；高写竞争计数选 `LongAdder` | CAS 不必 park；LongAdder 分散热点 | 不要为了“统一”套一把锁；需要全局强一致读取时 LongAdder 未必合适 |
| 通用对象互斥、追求简洁 | `synchronized` | JVM monitor，异常自动释放 | 临界区仍不能做慢 RPC/SQL/I/O |
| 超时、可中断、公平、多个业务条件 | `ReentrantLock + Condition` | AQS 明确提供 timed/interruption/多 Condition 合同 | `lock()` 后必须 `finally unlock()`；不要拿 Condition 代替跨服务协作 |
| 读远多于写 | `ReentrantReadWriteLock`；经验证可用 `StampedLock` 乐观读 | 读读共享；StampedLock 可先乐观读再校验 | 不做读锁→写锁升级；写频繁时读写锁可能不如普通锁 |
| 秒杀、连接池、下游 API 并发边界 | `Semaphore` | 共享 state 直接表示真实 permit | acquire/release 必配对；防止重复 release 许可膨胀 |
| 一次性并行任务汇总 | `CountDownLatch` | state 倒计时归零后共享放行 | 一次性，不能 reset；需要循环栅栏看 `CyclicBarrier`/`Phaser` |
| 按 key 的高竞争业务 | 分片锁 / striping / 分区队列 | 从根上减少同一 AQS state 的竞争 | 不要用全局锁“排得更公平”掩盖错误粒度 |

#### JDK 21 Virtual Threads：改变等待成本，不改变资源边界

JDK 21（JEP 444）下，必须把两句话同时成立地讲出来：

1. **Virtual Thread 在可卸载的 `LockSupport.park()` 路径通常可 unmount，carrier 可去执行别的虚拟线程。** 因此 `ReentrantLock`/AQS 的阻塞等待通常比“阻塞一个平台线程”更适配虚拟线程调度模型。
2. **JDK 21 中，虚拟线程在 `synchronized` 块/方法内发生阻塞，或执行 native/foreign 调用时，可能 pin 住 carrier。** 此时 carrier 不能被卸载复用；JFR 的 `jdk.VirtualThreadPinned` 是验证证据。后续 JDK 的 JEP 491 改进不能倒灌成对 JDK 21 的结论。

这导出的是**有条件的决策**，不是“以后禁用 synchronized”：

| JDK 21 场景 | 决策 |
|---|---|
| 短、纯内存的对象互斥 | `synchronized` 仍然清晰、合理；不要为虚拟线程机械替换 |
| `synchronized` 临界区内可能执行长 I/O / 阻塞调用 | 重构为锁外 I/O；若确需显式锁合同，可评估 `ReentrantLock`，并以 JFR 验证 pinning |
| 大量 virtual thread 访问 DB/HTTP 下游 | 仍须用 Semaphore/连接池/客户端并发上限定义容量，绝不能因线程便宜取消资源边界 |
| 怀疑 carrier 被拖住 | 开 JFR 查 `VirtualThreadPinned`，再定位 synchronized/native 调用；不要仅凭理论归因 |

> ⚠️ 更优先的规则始终是：**不要在任何锁内做长阻塞 I/O。** 用 ReentrantLock 替换 synchronized 可以避开某些 JDK 21 pinning 路径，但不能修复“临界区设计过大”这一根因。

#### 生产验证：别用“感觉”判锁

```bash
# 连续 dump：先定位 owner 以及 owner 是否卡慢活
for i in 1 2 3; do jcmd $PID Thread.print -l > /tmp/td.$i; sleep 5; done

# JFR：锁等待、park、调度需与业务延迟同图分析
jcmd $PID JFR.start name=aqs settings=profile duration=60s filename=/tmp/aqs.jfr

# CPU / wall 区分；wall 中 park 宽不等于队列长度
./profiler.sh -e cpu  -d 30 -t -f /tmp/cpu.html  $PID
./profiler.sh -e wall -d 30 -t -f /tmp/wall.html $PID

# 灰度使用：futex 多只能证明阻塞活动，不能单点定根因
strace -ff -tt -T -e trace=futex -p $PID -o /tmp/futex
cat /sys/fs/cgroup/cpu.stat 2>/dev/null || true
```

### Transfer——最终锁选型与架构结论

| 需求 | 首选 | 原因 |
|---|---|---|
| 简单对象内互斥 | `synchronized` | 语义直接、JVM 原生支持 |
| 可中断/定时/多个条件 | `ReentrantLock + Condition` | AQS 暴露了所需合同 |
| 限制下游并发 | `Semaphore` | `state` 直接表达真实 permit 边界 |
| 等一批前置任务 | `CountDownLatch` / `CompletableFuture` | 语义比手写状态机清晰 |
| 读多写少、可容忍乐观重试 | `StampedLock` 的乐观读模式 | **不继承 AQS**，有独立的 `WNode` 等等待队列实现；但用 stamp/version 作为资源账本、以队列协调悲观路径的设计哲学与 AQS 相似。乐观读路径不排队、不 park，校验失败才降级。 |
| 高竞争按 key 隔离 | 分片锁/striping | 避免全局 state 热点 |
| Virtual Thread 服务 | 每请求 virtual thread + 显式资源限流 | 不用传统线程池“复用线程”掩盖下游容量 |

> 📌 **架构师结论**：AQS 解决进程内“候选人如何有序等待并重新验证资源”的问题；它不替代跨服务背压、连接池容量、幂等、熔断或超时传播。先定义资源边界，再决定是否由 AQS 来排队。

> 🔴 **口诀**：Monitor 有 cxq、Entry、WaitSet；AQS 有主队、Condition；通知从不交所有权，先画不变量再选锁。

---

<a id="self-check"></a>
## 合书自测：一页时序图，不要再线性复读

> **使用方法**：合上 L1–L6，只看下图，自己口述五段。说不出“B 在哪本册子、state 是多少、谁负责推进 B”，再回对应 Level 复习。这一节是面试冲刺自测，不提供前文的逐句复述。

```text
【独占主线：订单打包台】

T0  无竞争：state=0
    B lock → tryAcquire/CAS 成功 → owner=B
    （没有 Node、head、tail；没有队列）

T1  A 已持锁，B/C 失败：state=1, owner=A
    B 首次进入 enq：竞争 CAS 初始化 H(dummy head)，再接 B
    C：prev=B，tail CAS 成功后接在 B 后
    H -> B -> C

T2  A release：state 1→0
    H 推进 B；B 被 unpark，只获得 tryAcquire 的机会
    B 成功：owner=B，head 从 H 前移为 B；C 等 B
    [非公平锁：路人仍可能在 B 运行前 CAS 插队]

T3  B 已持锁但队列空：notEmpty predicate=false
    B await → 完整释放 state → B 进 notEmpty ConditionQueue 并 park
    P 在同锁内 put + signal
    B：ConditionQueue → 同步队列 tail → 重新 acquire → while 复检 predicate

T4  B 超时：H -> B(X) -> C
    B 永远不能再获锁；C/释放者跳过 B(X)，队列进度不断

【共享支线：货架 Semaphore(3)】
X 成功：3→2，仍有余量，接力推进 Y
Y 成功：2→1，接力推进 Z
Z 成功：1→0，传播停止；W 继续 park
某次 release：0→1，再推进 W
```

| 自测题 | 必须答出的不变量 |
|---|---|
| H 是 owner 吗？ | 不是。H/head 是队列进度游标；owner 由同步器维护。 |
| B 被 unpark 就一定拿锁吗？ | 不一定。unpark 只给重试机会，`tryAcquire` 才交资源。 |
| signal 就能让 B 消费订单吗？ | 不能。先从 ConditionQueue 转到同步队列，再获取锁、while 复检。 |
| B 取消后为什么 C 仍能前进？ | 取消者不能再获资源，也不能永久阻塞活后继。 |
| 共享 release 为什么不叫醒全队？ | 用后继接力传播余量，避免惊群且保证许可进度不丢。 |

---

<a id="pitfalls"></a>
## 使用 AQS/Lock 的坑与细节：从代码味道到生产后果

下面不是 API 罗列，而是每个坑都回到工厂里：哪本登记册写错了、哪次交班被误解了、线上会看到什么。

### 坑 1：把 `ReentrantLock` 当成“更高级的 synchronized”，无理由替换

```java
// 没有定时、可中断、多个 Condition、显式公平等需求时：
private final Object lock = new Object();
synchronized (lock) { update(); }
```

**错因**：为了“底层是 AQS”而改 Lock，增加 `finally unlock()` 遗漏风险，却没有获得任何语义收益。

**后果**：异常分支漏 `unlock()`，整座工厂停工；`synchronized` 原本的简单对象不变量被复杂化。

**修正**：简单对象内互斥优先 `synchronized`；需要 `tryLock(timeout)`、`lockInterruptibly()`、多 Condition、可选公平或队列观测时再选 `ReentrantLock`。

### 坑 2：`lock()` 后没放 `finally`，或 lock 放在 try 里

```java
// 错：lock 成功后，业务异常会永久占工位
lock.lock();
doWork();
lock.unlock();

// 对：lock 在 try 前；unlock 必须在 finally
lock.lock();
try {
    doWork();
} finally {
    lock.unlock();
}
```

**工厂后果**：owner 异常退出没有交班；所有等待者都 park。线程 dump 能看到一串等待者，却未必再看到 owner 的正常业务栈（例如 owner 已死于异常路径）。

### 坑 3：锁内 RPC、SQL、磁盘 I/O、回调

```java
lock.lock();
try {
    state.update();
    remoteClient.call(); // 错：不可控慢活占着核心工位
} finally { lock.unlock(); }
```

**错因**：把“必须原子保护的内存状态”与“耗时外部操作”混成一段。

**后果**：队列变长只是症状；公平/非公平都救不了 2s 下游调用。Virtual Thread 也只能让等待者更便宜，不能让锁交班更快。

**修正**：锁内只完成状态快照/状态转换；锁外调用；必要时用版本号、状态机或两阶段补偿处理回写。

### 坑 4：Condition 用 `if`，或 predicate 与 signal 不在同一把锁内

```java
// 错：虚假唤醒、竞争者抢走数据、signal 早到都能穿透
if (queue.isEmpty()) notEmpty.await();

// 对
while (queue.isEmpty()) notEmpty.await();

// 错：在锁外更新 ready，再 signal，检查—等待之间可丢关系
ready = true;
lock.lock(); try { condition.signal(); } finally { lock.unlock(); }

// 对：状态变化与 signal 在同一把锁保护下
lock.lock();
try { ready = true; condition.signal(); }
finally { lock.unlock(); }
```

**工厂后果**：B 被领回正门后，条件早被别人改变；或 B 刚检查完 false、还没登记到侧门，P 已经 signal，B 可能永久睡下。

### 坑 5：把 `signal()` 当成“精准且立即执行”

`signal()` 只是从**同一个 ConditionQueue**选择一个候选，转移到正门同步队列。它不保证：谁先被 OS 调度、谁先拿锁、业务条件仍然成立。

- 要表达“可能有很多人都可继续”，才考虑 `signalAll()`；
- `signalAll()` 不是默认安全按钮，它会让所有人从侧门回正门竞争，可能造成惊群；
- 选择 `signal` 还是 `signalAll` 必须由 predicate 的语义决定，而不是“怕丢通知”。

### 坑 6：把公平锁当成延迟优化开关

| 错误直觉 | 真相 |
|---|---|
| 公平锁必慢 | 高竞争下非公平常吞吐更高；低竞争差异可能很小 |
| 非公平必然导致饥饿 | 有风险但不是每负载必现；需观察等待分布 |
| P99 高就改非公平 | 先查 owner 是否锁内慢活、锁是否过粗、cgroup 是否限流 |

**工厂解释**：公平锁更强调“正门老号先尝试”，非公平锁允许路人趁门刚开抢先进。若打包台每次只做 50ns，抢占可能省下调度成本；若每次抱着 RPC 2 秒，谁先进都救不了队列。

### 坑 7：把 `tryLock()` 失败当成“系统忙，立刻降级”

无参 `tryLock()` 常是立即尝试，可能绕过公平锁的排队规则；它适合明确的快速失败语义，不适合用作“我想公平等一会”的替代。需要等待上限时使用：

```java
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    try { work(); } finally { lock.unlock(); }
} else {
    fallback(); // 必须是业务上正确的降级，而非吞掉写入
}
```

还要决定 interrupt 合同：超时获取是否应响应取消？如果应响应，调用链不能把 `InterruptedException` 随手吞掉。

### 坑 8：把 `getQueueLength()` 当精确监控指标或控制依据

AQS 队列观测方法在并发下是估算/快照；线程可能正在入队、取消、被唤醒。它适合诊断趋势，不适合写成：

```java
if (lock.getQueueLength() > 100) { // 错：拿近似值做正确性/硬限流
    reject();
}
```

**修正**：业务限流用独立的 Semaphore、队列容量或网关限流；AQS 观测值只作为告警与诊断信号。

### 坑 9：自己继承 AQS “造一个简单锁”

自定义同步器至少要对这些合同负责：state 原子转移、owner、重入溢出、独占/共享返回值、取消、中断、超时、Condition 的 `isHeldExclusively`、序列化、观测语义。少一个边角，压测低概率时序就能把工厂锁死。

**默认策略**：先用 JDK 的 `ReentrantLock`、`Semaphore`、`CountDownLatch`、`Phaser`、`StampedLock` 或并发容器；只有确实存在无法表达的资源状态机，才进入自定义 AQS 设计评审。

### 坑 10：Virtual Thread 下以为“阻塞免费、连接无限”

```text
错误：100 万 virtual thread → 同时请求 DB
现实：DB 连接池可能只有 100；下游 QPS 可能只有 5000
结果：等待者不占 carrier，不代表下游不会被压垮
```

用 `Semaphore` 把**真实下游容量**写成 permit；同时通过 JFR 检查 pinning。JDK 21 中 `synchronized` 内阻塞、native/foreign 调用等可导致 virtual thread pin carrier；不要以为“用了 AQS”就自动没有调度问题。

### 坑 11：从火焰图 `park` 宽度直接推出“锁队列长”

`park` 的 wall 时间只说明线程在等待；可能等锁、等 Condition、等调度、等 cgroup 配额、等下游释放 permit。正确诊断顺序：

```text
连续 thread dump → owner 在哪
JFR/业务指标      → 等待类型与 P99 是否相关
CPU/wall profiler  → CPU 热点还是墙钟阻塞
cgroup/perf/strace → 调度与 futex 作为补充证据
```

> ⚠️ `strace -e futex` 频繁只说明阻塞原语活动频繁；它不单独证明“futex 风暴就是根因”。

### 坑 12：把 JDK 8 的 AQS 图当 JDK 21 的字段事实

JDK 8 教材里的 `waitStatus`、`SIGNAL=-1`、`CONDITION=-2`、`PROPAGATE=-3` 对理解经典协议有价值；JDK 21 的 `Node` 使用 `status` 位、`WAITING/COND/CANCELLED` 等不同内部组织；不要把 JDK 8 教材的私有常量直接套入。**看源码时先 `java -version`，再打开同版本源码。**

---

<a id="t0-t8"></a>
## AQS 竖切总表 T0–T8

| 维度 | T0 入口 | T1 state | T2 CLH 入队 | T3 park 握手 | T4 唤醒 | T5 Condition | T6 共享 | T7 取消 | T8 诊断 |
|---|---|---|---|---|---|---|---|---|---|
| 位置 | 调 `lock` | 试账本 | 正门取号 | 候工休息 | 获重试权 | 侧门等条件 | 领 permit | 离场废号 | 监控取证 |
| 关键动作 | `tryAcquire` | CAS/owner | prev + tail CAS | 先意图后 park | unpark 后再试 | 转入主队列 | 按余量传播 | 跳过/清理 | dump/JFR/perf |
| 不变量 | API 合同 | 状态原子 | tail 是提交点 | 不漏睡 | 信号非所有权 | predicate 必须重验 | 不惊群 | 不挡后继 | 多证据关联 |
| 典型坑 | 选错 API | `volatile++` | next 永远可靠 | park 必进内核 | 醒来就有锁 | `if await` | `unparkAll` | 取消不修复 | 单图定罪 |

---

<a id="references"></a>
## 附录：版本勘误与延伸阅读

| ❌ 常见说法 | ✅ 更准确的说法 |
|---|---|
| AQS 用的是 CLH 锁 | AQS 是借鉴 CLH/MCS 思想的**阻塞同步队列变体**。 |
| `park()` 会耗 CPU / 必然 futex | 它不是忙等，但可能立即返回；平台、虚拟线程、OS 决定实际路径。 |
| 公平锁必然慢 | 高竞争下非公平常吞吐更好；公平可改善等待分布，须压测。 |
| Condition 底层就是 `Object.wait()` | 二者队列与实现不同；方法论同为“通知后重新竞争”。 |
| JDK 21 仍有 `SIGNAL=-1` | 这是 JDK 8 常见协议；21 的 Node 使用不同状态位与内部组织，这次重写发生在 JDK 19，为支撑虚拟线程。 |
| dummy head 是工厂开工就有的编制 | head/tail 初始为 null；哑节点在第一次真正发生排队竞争时才被懒创建。 |

- Doug Lea, *The java.util.concurrent Synchronizer Framework*（2004）——AQS 设计思路的一手出处，CLH/MCS 取舍的原始论证
- OpenJDK 21：[`AbstractQueuedSynchronizer.java`](https://github.com/openjdk/jdk21/blob/master/src/java.base/share/classes/java/util/concurrent/locks/AbstractQueuedSynchronizer.java)、[`ReentrantLock.java`](https://github.com/openjdk/jdk21/blob/master/src/java.base/share/classes/java/util/concurrent/locks/ReentrantLock.java)
- HotSpot：`ObjectMonitor` 源码与部署版本配套阅读；不要把某个版本的 `_cxq/_EntryList` 当 Java 规范。
- Linux：[futex(2)](https://man7.org/linux/man-pages/man2/futex.2.html)、[futex(7)](https://man7.org/linux/man-pages/man7/futex.7.html)
- JEP：[JEP 444 Virtual Threads](https://openjdk.org/jeps/444)

> **最后只留一个问题**：徒弟 B 此刻在工厂的哪本登记册？`state` 是什么？谁有责任让他获得下一次重试？能稳定回答这三句，AQS 的主干才算真正长进脑子里。

---

<a id="production-decisions"></a>
## 终章：从“我懂 AQS”倒推到“我在生产做了什么决策”

前面解释了工厂怎么排队；这一章只做一件事：把机制翻译成架构师能签字的决定。读完 AQS 如果只能回答“`park` 和 `unpark` 怎么走”，那只是懂零件；能回答“这个系统为什么不用全局锁、为什么这里必须用 Semaphore、为什么 P99 不能先改公平锁”，才算把零件装成生产判断力。

### 先建立决策链：不要从 API 名称开始

```text
业务目标 / SLA / 下游容量
        ↓
要保护的不变量是什么？谁可以并发？
        ↓
失败者到底在等资源，还是等业务条件？
        ↓
锁内是否有慢活？热点能否按 key 拆开？
        ↓
选择 synchronized / ReentrantLock / Condition / Semaphore / 队列 / 无锁结构
        ↓
定义超时、中断、降级和观测指标
        ↓
压测验证吞吐、P99/P999、最大等待、错误率和下游饱和度
```

> 📌 **决策铁律**：AQS 是“等待协议”，不是“性能药”。只要资源边界、临界区或下游容量定义错了，换公平锁、调 park、增加线程都只是把排队从一个地方搬到另一个地方。

### 生产决策卡 1：订单状态更新——为什么不是一把全局锁？

#### 场景与判断

订单状态机要求同一订单不能同时从 `PAID` 转两次；但不同订单天然可并行。若用一把全局 `ReentrantLock`：

```text
所有订单 → 一份 state / 一条 AQS 队列
一个订单的慢 SQL / RPC
→ 所有无关订单 B/C/D 都在同一正门排队
→ 队列等待上升，P99 上升，超时重试又进一步放大竞争
```

| AQS 机制判断 | 架构决策 |
|---|---|
| AQS 队列只会把竞争有序化，不会减少竞争 | 按 `orderId` 做分片锁 / striping，而不是全局锁 |
| `state` 的正确语义是“同一订单状态迁移互斥” | 锁粒度绑定订单键，不绑定整个服务 |
| 锁内 RPC 会使 owner 持锁时间由下游 P99 决定 | 锁内只完成内存状态迁移/版本校验；RPC 放锁外 |
| 锁外调用可能出现并发回写 | 用版本号、幂等键、CAS 或持久化状态机收口 |

#### 决策落地

```java
Lock lock = locks[spread(orderId) & (locks.length - 1)];
lock.lock();
try {
    // 只做：读取状态、校验转换、写入 pending + version
    transitionToPending(orderId);
} finally {
    lock.unlock();
}

// 锁外：不可控慢活
PaymentResult r = paymentClient.pay(orderId, idempotencyKey);

// 再以 version/CAS/幂等语义回写最终状态
completeTransition(orderId, r);
```

#### 不能做的错误决策

- 看到队列长就把公平锁改成非公平锁；
- 把整个“状态校验 + RPC + DB 回写”包进一把锁求“简单”；
- 用 `tryLock()` 失败直接吞订单，假装是降级。

#### 验收指标

`持锁 wall time`、`锁等待 P99`、`每分片队列倾斜度`、`支付下游 P99/错误率`、`状态版本冲突率`。只有这些一起改善，才说明拆锁成功；仅吞吐上涨而状态冲突/补偿增多，是错误优化。

### 生产决策卡 2：连接池与第三方 API——为什么用 Semaphore，而不是扩大线程池？

#### 场景与判断

数据库连接只有 80 条，第三方 API 只允许 300 并发。Virtual Thread 可以轻松创建十万请求，但它不能制造第 81 条连接或第 301 个下游并发槽位。

| AQS 机制判断 | 架构决策 |
|---|---|
| 共享 state 正好表达“剩余可用容量” | `Semaphore(80)` / `Semaphore(300)` 放在资源入口 |
| 共享传播让可用 permit 被有序消费 | 不用 `unparkAll`，不让所有请求同时冲下游 |
| 超时/中断是 API 合同的一部分 | acquire 设置业务 deadline；失败走正确的限流/排队响应 |
| 线程池限制的是执行线程，不等于下游容量 | 用线程池管理 CPU 工作；用 permit 管理外部资源 |

#### 决策落地

```java
if (!dbPermits.tryAcquire(150, TimeUnit.MILLISECONDS)) {
    throw new ServiceBusyException("db concurrency saturated");
}
try {
    return queryDatabase();
} finally {
    dbPermits.release();
}
```

permit 数不是拍脑袋常量。起点应来自：连接池最大连接、下游最大并发、单请求平均/尾延迟、错误率拐点和压测结果。例如连接池 100 并不意味着 permit 一定设 100：还要为运维查询、事务长尾、连接回收、其他业务租户留下余量。

#### 不能做的错误决策

- 为“提高吞吐”把虚拟线程全部直通 DB；
- 用固定线程池大小隐式限制连接，导致 CPU 工作和 I/O 等待混在同一个容量阀门；
- `tryAcquire` 超时后无限重试，形成排队—超时—重试放大器。

#### 验收指标

`permit wait P50/P99`、`acquire timeout rate`、`连接池 active/pending`、`DB P99/错误率`、`请求端重试率`。资源饱和时，期望结果是**快速且可解释地拒绝/降级**，不是所有请求一起慢死。

### 生产决策卡 3：生产者消费者——什么时候要 Condition，什么时候直接用 BlockingQueue？

#### 场景与判断

业务只是在有界队列上 `put/take`，自己手写 `ReentrantLock + notEmpty + notFull` 通常没有收益；JDK 的 `ArrayBlockingQueue` 已经把两个 Condition、signal、取消/中断合同封装好。

| 需求 | 决策 |
|---|---|
| 普通有界生产消费 | 优先 `ArrayBlockingQueue` / `LinkedBlockingQueue` |
| 只需异步交接 | `CompletableFuture`、消息队列或 Reactor 模型按场景选择 |
| 一个锁下确有多个强相关业务 predicate | `ReentrantLock` + 多个命名 Condition |
| 条件跨进程/跨服务 | 不要拿 JVM Condition 硬做；用消息、状态机、数据库/协调系统协议 |

若必须手写 Condition，评审必须写出：predicate 的定义、谁在同锁内修改它、哪次修改 `signal` 哪个 Condition、为什么是 `signal` 不是 `signalAll`、等待超时后业务如何收口。

#### 不能做的错误决策

- `if (empty) await()`；
- 一个“万能 Condition”既等非空又等非满；
- `signalAll()` 当作“肯定不会漏通知”的万能按钮；
- 锁外改 predicate，锁内只 signal。

#### 验收指标

`await 数与 signal 数的业务比例`、`signal 后成功消费比例`、`队列深度`、`生产/消费滞后`、`超时率`。若大量被 signal 的线程回来仍不满足条件，说明 predicate 设计或唤醒粒度不对。

### 生产决策卡 4：P99 突刺——什么时候改锁策略，什么时候禁止改？

#### 先给出值班决策树

```text
P99 上升
 ├─ 连续 dump 中同一 owner 长时间卡 RPC/SQL/I/O？
 │    └─ 是：拆锁内慢活 / 分片；禁止先改公平策略
 ├─ owner 快速轮换，但大量线程等同一全局 key？
 │    └─ 是：按 key 分片、合并批处理或重审数据结构
 ├─ 锁等待不高，但 cgroup throttled / run queue 高？
 │    └─ 是：查 CPU 配额、调度、carrier pinning；不是 AQS 队列根因
 ├─ Condition await 多且 signal 后大多失败？
 │    └─ 是：查 predicate、while、signal 粒度
 └─ 临界区极短、竞争稳定、出现明显插队等待不均？
      └─ 才进入公平/非公平 A/B 压测决策
```

#### 公平/非公平真正的决策条件

| 业务约束 | 更可能的选择 | 需要验证什么 |
|---|---|---|
| 吞吐敏感、临界区极短、允许等待分布波动 | 非公平 | 总吞吐、CPU、P99/P999、最大等待时间 |
| 明确反饥饿 SLA、长等待比平均吞吐更危险 | 公平或上层排队策略 | 最大等待、租户公平度、尾延迟 |
| 锁内慢 RPC/SQL | 都不是第一决策 | 先拆临界区 |
| 全局热点数据 | 都不是第一决策 | 先分片/改数据模型 |

### 生产决策卡 5：Virtual Threads——把“线程模型”与“资源模型”拆开

#### 判断

Virtual Thread 使 park 的载体成本降低，但并不改变 AQS 的三件事：`state` 仍是资源账本、队列仍是等待秩序、unpark 后仍须调度和重新获取。

#### 决策

```text
请求并发模型：每请求一个 Virtual Thread（简化编程模型）
CPU 边界：Executor / CPU 并行度控制
DB/HTTP 下游边界：Semaphore / 连接池 / 客户端并发上限
跨服务背压：队列容量、超时、熔断、限流
```

不要把四个边界混进一个固定线程池。特别是“线程池 200”同时承担 CPU、DB、HTTP 三种容量控制时，任何一个下游抖动都会挤占另两类工作的线程。

#### 验收指标

JFR VirtualThreadPinned 事件、carrier 利用率、下游 permit 等待、连接池 pending、请求 P99/P999。若 carrier 被 pin，先找 `synchronized` 块内阻塞或 native 调用等真实原因；不要根据“虚拟线程理论上可卸载”直接下结论。

### 最终交付：技术评审必须写出的《同步决策记录》

每一个新锁、Condition、Semaphore 或自定义 AQS，同步评审单至少要有以下字段：

```text
1. 不变量：保护哪些字段/状态转换？不保护什么？
2. 资源模型：独占还是共享？state/permit 的业务单位是什么？
3. 等待模型：等资源还是等 predicate？Condition 有几条、各自 predicate 是什么？
4. 边界：锁内最大允许做什么；RPC/SQL/回调为何在锁内或锁外？
5. 取消：中断、超时、服务关闭时的业务结果是什么？
6. 公平：是否存在饥饿 SLA？为什么选择公平或非公平？
7. 容量：permit/队列长度从哪些数据推导？拒绝与降级语义是什么？
8. 可观测性：owner、持锁 wall time、等待、超时、下游饱和分别如何监控？
9. 压测：成功标准同时包含吞吐、P99/P999、最大等待、错误率和资源利用率。
```

> 🏆 **最终答案**：从“我懂 AQS”倒推回生产，不是说“我知道 CLH”。而是能写下：**因为 CLH 只能有序等待、不能降低资源竞争，所以我先分片；因为 Condition 的 signal 不交所有权，所以我用 while 复检；因为共享 state 表达真实容量，所以我用 Semaphore 保护下游；因为 park 宽不等于锁队列长，所以我先抓 owner 和 cgroup 证据。** 这些才是 AQS 在生产里真正值钱的判断力。

---

<a id="cross-runtime"></a>
## 加分视野：这不是 Java 独有——不同运行时如何解决同一座工厂的问题

AQS 的“资源账本 + 等待者秩序 + 条件等待 + 唤醒后复验”不是 Java 特产。不同语言和运行时只是把工厂制度放在了不同层：有的由语言运行时托管，有的交给标准库，有的把最后停车动作直接交给内核。

> ⚠️ 下表用于建立**问题同构性**，不是宣称不同运行时的队列公平性、内部字段或调度策略完全等价。尤其是 Go runtime、libc 与操作系统版本都可能改变实现；不要把实现习惯当语言规范。

| Java / AQS 工厂问题 | Go | C++ 标准库 | POSIX / Linux | 不变的工程不变量 |
|---|---|---|---|---|
| 独占进入核心工位 | `sync.Mutex` | `std::mutex` | `pthread_mutex_t` | 同一临界区同一时刻只能有一个 owner |
| 等业务 predicate | `Condition.await()` | `sync.Cond.Wait()`；很多场景直接用 channel | `std::condition_variable::wait` | 必须释放 mutex 后等待；醒来仍要复检条件 |
| 传递数据/容量边界 | `BlockingQueue` / `Semaphore` | buffered `chan T` 可同时承载交接和容量 | 队列 + mutex/cond 自建 | 数据/许可的容量不能被线程数量掩盖 |
| 等一次事件完成 | `CountDownLatch` / Future | `WaitGroup` / channel close | cond/eventfd 等组合 | 等待者需有明确完成信号与取消语义 |
| 线程停车/唤醒 | `LockSupport.park/unpark` | runtime `gopark/goready` 等内部机制 | `condition_variable::wait/notify_*` | 唤醒不是获得资源，醒来必须重新检查 |
| Linux 终极阻塞原语 | HotSpot/线程库路径可能间接使用 futex | libstdc++/libc++ 配合 pthread 实现可能间接使用 futex | `futex(FUTEX_WAIT/FUTEX_WAKE)` | 只有满足期望值才睡；唤醒数量/调度不等于业务正确性 |

### 1. Go：channel 是“传送带 + 容量阀门”，Mutex 是运行时托管的工位锁

```go
// 有界传送带：容量本身就是背压边界
jobs := make(chan Job, 100)

// 发送者在传送带满时阻塞；接收者在空时阻塞
jobs <- job
job := <-jobs
```

Go channel 适合把“交付一个任务”和“最多积压多少任务”合并表达；这接近 Java 里 `BlockingQueue` 的生产消费场景，而不是一般意义上替代所有 `ReentrantLock + Condition`。如果要保护多字段不变量，仍需 `sync.Mutex`；若等待一个 predicate，仍须遵守与 Java `while (!predicate) await()` 同构的纪律：**持 mutex 检查条件，循环等待，醒来重检。**

```go
mu.Lock()
for !ready {
    cond.Wait() // Wait 原子地释放并在返回前重新获取 mu
}
consume()
mu.Unlock()
```

Go 的 `sync.Mutex` 有 runtime 层面的自旋、饥饿模式等优化，但其具体队列策略不是应用代码可以依赖的 FIFO 合同。和 AQS 非公平锁一样：不要从“某次实现常常如此”推导业务正确性。

### 2. C++：`condition_variable` 把“侧门转回正门”的纪律写进 API 用法

```cpp
std::unique_lock<std::mutex> lk(mu);
cv.wait(lk, [&] { return ready; }); // 等价于 while (!ready) cv.wait(lk)
consume();
```

`std::condition_variable` 的 predicate 重载正是为了防止 `if` 等待、虚假唤醒和通知/竞争交错。`notify_one()` 与 Java `signal()` 一样，只通知一个等待者：**它不移交 mutex 所有权**；被通知的线程还必须等到 mutex 可用并复检 predicate。C++ 标准不承诺 `std::mutex` 的公平性或内部队列形状，因此把“绝对 FIFO”写进业务 SLA 同样是错的。

### 3. pthread / futex：把 AQS 的“最后一层停车”抽出来看

```text
应用层条件：ready 是否成立？                ← Java/Go/C++ 业务代码负责
互斥与条件协议：mutex + condvar            ← pthread/标准库负责
真正阻塞/唤醒：libc 可能调用 futex WAIT/WAKE ← Linux 内核负责
调度：被 WAKE 的线程成为 runnable           ← 调度器决定何时真正运行
```

Linux futex 的关键价值是“无竞争尽量留在用户态；确认需要等待时，按某个用户态地址和值进入内核等待；变化后唤醒若干等待者”。它并不知道你的订单、锁 owner、Condition predicate，更不保证“被 WAKE 的线程立即拿到资源”。这和 AQS 的分层完全呼应：**内核提供停车原语，用户态协议决定谁应该停车、谁应被推进、醒来后如何验证。**

### 4. 跨语言后仍然成立的六条判断力

1. **先画不变量，再选同步原语**：Mutex、AQS、channel 都不是业务状态机的替代品。
2. **通知不等于条件成立，也不等于资源所有权移交**：Java `signal`、C++ `notify_one`、Go `Cond.Signal` 都必须配合循环复检。
3. **线程/协程便宜不等于下游资源无限**：Go goroutine、Java virtual thread 都仍需连接池、信号量、队列容量与超时。
4. **队列顺序不是默认 SLA**：AQS、Go runtime、pthread、C++ library 的具体调度都可能演进；若业务要公平，要显式设计和测量。
5. **最后的 park/futex 不是根因诊断终点**：它只说明有人等；先找 owner、predicate、下游容量和调度限制。
6. **能用标准库就不要手写等待协议**：Java 用 `BlockingQueue`/`Semaphore`，Go 用 channel/`sync`，C++ 用 RAII mutex/condition_variable；自造队列最容易漏取消、超时、虚假唤醒与异常释放。

> 🏁 **收尾视角**：Doug Lea 的 AQS、Go 的 runtime 同步器、C++ 的 mutex/condition_variable、Linux futex，看起来是不同技术栈，实则都在回答同一个问题：当资源暂时不可得时，怎样让等待者不浪费 CPU、不会漏掉进度、又能在醒来时安全地重新确认世界。语言变了，工厂的账本、登记册和交班纪律没有变。
