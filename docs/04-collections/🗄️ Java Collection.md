# 🗄️ Java Collection

## 写在最前面的承诺

> 这不是一份知识点清单，而是一份**因果链推导手册**。
> 
> **唯一比喻**：一座智能图书馆系统 —— 书架是容器，索引卡是哈希表，管理员是迭代器，借阅规则是并发控制。
> 
> **唯一主线角色**：一个查询请求 Q，从 `map.get(key)` 调用开始，穿越哈希桶、红黑树、并发锁，最终返回结果的完整旅程。
>
> 读完本文，你应该能在面试中回答："为什么 HashMap 在 JDK 8 要改成红黑树？"时，不是背"因为链表太长"，而是能推导出：**链表长度超过8时，退化为O(n)的查找在热点Key场景下会成为性能瓶颈，而红黑树的O(log n)查找配合64的容量阈值，在时间和空间上找到了平衡点**。

---

## 🏷️ 关键词行 + 🧵 主线进度条

**关键词**：`Collection` `List` `Set` `Map` `Iterator` `Fail-Fast` `Fail-Safe` `HashMap` `ConcurrentHashMap` `红黑树` `哈希冲突` `扩容` `负载因子`

---

## ⚠️ 版本与证据边界

| 类别 | 说明 |
|------|------|
| **基线版本** | OpenJDK 21 LTS 与 JDK 8 做真实差异对比 |
| **私有实现** | `HashMap.TreeNode` 的红黑树旋转细节、`ConcurrentHashMap` 的 `CounterCell` 数组布局 |
| **教学量级** | 所有性能对比数据均为教学参考，**以你的部署实测为准** |
| **必须实测** | 高并发场景下 `ConcurrentHashMap` vs `Collections.synchronizedMap` 的吞吐量差异 |
| **版本边界** | 文中标注 `[JDK 8+]` 或 `[JDK 21+]` 的特性，低于该版本行为可能不同 |

---

## 📐 覆盖范围声明（Scope Boundary）

> 本文标题是"Collection 框架深度解析"，但**不是全景扫描**。以下是明确的覆盖边界：

| 分类 | 本文覆盖 | 本文未覆盖 | 原因 |
|------|---------|---------------------------|------|
| **List** | ArrayList、LinkedList、Vector | CopyOnWriteArrayList（正文提及但未深挖） | 前三者是面试高频 |
| **Set** | HashSet、TreeSet、**EnumSet**、**CopyOnWriteArraySet**（速查） | LinkedHashSet | 速查覆盖 |
| **Map** | HashMap、TreeMap、LinkedHashMap、**EnumMap**、**IdentityHashMap**、**ConcurrentSkipListMap**（速查） | WeakHashMap（正文提及机制但未深挖） | 速查覆盖 |
| **Queue/Deque** | 概览性介绍（ArrayDeque、PriorityQueue） | 未深挖 | 篇幅足够撑起独立文档 |
| **并发容器** | ConcurrentHashMap（深挖）、CopyOnWriteArrayList/ArraySet（速查）、概览性介绍 BlockingQueue 家族 | ConcurrentLinkedQueue/Deque、BlockingQueue 各实现类深挖 | 这条线与线程池强耦合，不在这里展开 |
| **接口** | **NavigableMap/NavigableSet**（速查） | SortedMap/SortedSet（被 Navigable 覆盖） | 速查覆盖 |

**如果你在面试中被问到本文未覆盖的内容**：用本文 Level 1-8 建立的"Why → What → How → Transfer"框架自行推导，原理是相通的。

---

## 🗂️ 目录

<a id="toc"></a>

- [📍 能力地图](#capability-map)
- [🏭 全文唯一比喻地图](#metaphor-map)
- [Level 1：为什么需要 Collection 框架？](#level-1)
- [Level 2：List 的三种面孔 —— ArrayList vs LinkedList vs Vector](#level-2)
- [Level 3：哈希的本质 —— 从数组到 HashMap 的演进](#level-3)
- [Level 4：HashMap 源码级拆解 —— 从 put() 到红黑树](#level-4)
- [Level 5：TreeMap 与排序 —— 红黑树的另一面](#level-5)
- [Level 5.5：LinkedHashMap 与 LRU —— HashMap 的有序变体](#level-5-5)
- [Level 6：并发容器 —— 从 Collections.synchronizedMap 到 ConcurrentHashMap](#level-6)
- [Level 7：Iterator 与 Fail-Fast/Fail-Safe 机制](#level-7)
- [Level 7.5：Queue/Deque 家族概览 —— 被忽视的第三条线](#level-7-5)
- [Level 7.6：扩展容器速查 —— CopyOnWrite/Enum/Identity/SkipList/Navigable](#level-7-6)
- [Level 8：生产决策 —— 如何在架构评审中选型](#level-8)
- [🧪 合书自测](#self-test)
- [⚠️ 坑与细节](#pitfalls)
- [📊 竖切总表 T0–T8](#timeline)
- [📚 版本勘误表](#errata)
- [🏆 生产决策卡](#decision-cards)
- [🌍 跨语言/跨运行时视角](#cross-language)

---

<a id="capability-map"></a>

## 📍 能力地图

| 层级 | 要打穿的认知墙 | 通关标准 | 覆盖家族 |
|------|---------------|---------|---------|
| Level 1 | 为什么不能只用数组？ | 能说出数组的3个致命缺陷 | 全局 |
| Level 2 | ArrayList 和 LinkedList 的真实差异 | 能用内存布局解释为什么 ArrayList 在大多数场景更快 | **List** |
| Level 3 | 哈希函数的本质是什么？ | 能从模运算推导出 HashMap 的寻址过程 | **Map** |
| Level 4 | HashMap 的 put() 到底经历了什么？ | 能画出完整的时序图，包括扩容和树化 | **Map** |
| Level 5 | TreeMap 为什么用红黑树而不是 AVL？ | 能从**删除旋转次数**和工程实现角度解释，知道插入两者都是 O(1) | **Map** |
| Level 5.5 | LinkedHashMap 如何实现 LRU？ | 能手写一个基于 LinkedHashMap 的 LRU 缓存，说清 `accessOrder` 和 `removeEldestEntry` | **Map** |
| Level 6 | ConcurrentHashMap 如何实现线程安全？ | 能对比 JDK 7 分段锁和 JDK 8 CAS 的演进，能说清扩容时的 `helpTransfer` | **并发容器** |
| Level 7 | 为什么遍历时删除元素会报错？ | 能区分 Fail-Fast 和 Fail-Safe 的实现机制 | 全局 |
| Level 7.5 | Queue/Deque/BlockingQueue 家族长什么样？ | 能说出 ArrayDeque 为什么替代 Stack，BlockingQueue 的四种实现区别 | **Queue**（概览） |
| Level 7.6 | 那些"不常见但问到就加分"的容器 | 能说清 EnumSet 的位运算、IdentityHashMap 的 == 比较、SkipList 的结构 | **扩展容器** |
| Level 8 | 面对业务场景如何选型？ | 能输出决策矩阵，包含"不能做的错误决策"和"对应 Level 哪个机制" | 全局 |

---

<a id="metaphor-map"></a>

## 🏭 全文唯一比喻地图

```
┌─────────────────────────────────────────────────────────────────┐
│                     智能图书馆系统                              │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ 比喻元素         │ 技术概念         │ 它负责的唯一一件事           │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ 书架            │ 底层数组         │ 存储数据的物理载体           │
│ 索引卡          │ 哈希表           │ 快速定位目标书籍             │
│ 分类标签        │ 哈希函数         │ 将书名映射到书架位置         │
│ 同一架位多本书   │ 哈希冲突         │ 不同书映射到同一位置         │
│ 链式挂书        │ 链表             │ 解决冲突的线性方案           │
│ 专题书柜        │ 红黑树           │ 解决冲突的对数方案           │
│ 图书馆扩建      │ 扩容             │ 书架不够时的迁移             │
│ 借阅登记本      │ Iterator         │ 遍历所有书籍的有序方式       │
│ 闭馆通知        │ Fail-Fast        │ 遍历时修改的快速失败         │
│ 副本借阅        │ Fail-Safe        │ 遍历时修改的安全方案         │
│ 多窗口借还      │ ConcurrentHashMap │ 并发访问的无锁/分段方案      │
│ 借阅规则        │ 并发控制         │ 防止多人同时操作同一本书     │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

---

<a id="level-1"></a>

## Level 1：为什么需要 Collection 框架？

### 👶 前置知识关卡

- [ ] Java 中数组的声明和初始化方式？
- [ ] 数组的 `length` 属性和 `Arrays` 工具类的区别？
- [ ] 什么是泛型（Generics）？

### Why——上一代方案死在哪

**徒弟**：老师，我直接用数组存数据不就行了吗？为什么要学这么复杂的 Collection 框架？

**老陈**：好问题。我们先看数组的三个致命缺陷：

| 缺陷 | 具体表现 | 线上事故场景 |
|------|---------|-------------|
| **固定大小** | `int[] arr = new int[10]` 创建后无法扩容 | 订单量突增，数组越界，系统崩溃 |
| **类型不安全** | `Object[] arr = new Object[10]` 可以存任何类型 | 运行时 ClassCastException，难以定位 |
| **功能贫乏** | 没有 `contains()`、`remove()`、`sort()` 等方法 | 每次都要手写二分查找、手写排序，代码重复率高 |

**徒弟**：那我封装一个工具类不就行了？

**老陈**：可以，但你封装的工具类，别人也在封装。100 个团队封装 100 个"动态数组"，接口不统一、性能不保证、bug 各不同。**Collection 框架的本质是：JDK 官方帮你封装好、经过千万次压测、接口统一的标准库**。

```
朴素方案：直接用数组
├── 优点：性能极致（连续内存、CPU 缓存友好）
├── 缺点：固定大小、类型不安全、功能贫乏
└── 致命账：每次业务变化都要重写底层代码，无法复用

Collection 框架方案：
├── 优点：动态扩容、类型安全、功能丰富、接口统一
├── 缺点：有一定的性能开销（对象头、额外字段）
└── 致命账：需要理解不同容器的适用场景（本文重点）
```

### What——Collection 框架是什么

```
                    Iterable
                       │
                   Collection
                   /    |    \
                  /     |     \
               List    Set    Queue
              / | \    / \      |
           Arr  Link  Hash Tree Priority
           ay   ed   Set  Set  Queue
           List List
             
                  Map（独立体系）
                  / | \
              Hash Tree Linked
              Map  Map  HashMap
```

> 📌 **最容易被误解的一点**：`Map` 不是 `Collection` 的子接口，它们是并列关系。`Map` 存储键值对，`Collection` 存储单个元素。

### How——主线角色的慢动作

```
小明要管理 100 个订单 ID：
① 用数组：int[] orders = new int[100]; // 固定大小，订单超了就崩
② 用 ArrayList：List<Integer> orders = new ArrayList<>(); // 自动扩容，类型安全
③ orders.add(orderId); // 简单一行，底层帮你处理扩容、索引、边界检查
④ orders.contains(orderId); // O(n) 查找，线性扫描
⑤ 想要 O(1) 查找？→ 引出 HashMap（Level 3-4 的故事）
```

**徒弟**：ArrayList 的 `add()` 是 O(1) 吗？

**老陈**：**均摊 O(1)**。因为扩容时需要复制整个数组，单次扩容是 O(n)，但分摊到每次 add 就是 O(1)。这个"均摊"是面试高频考点，记住：**最坏 O(n)，均摊 O(1)**。

### Transfer——迁移到其他设计问题

1. **封装优于裸用**：无论什么语言，优先用标准库容器，而不是自己造轮子
2. **接口优于实现**：声明时用 `List` 而不是 `ArrayList`，方便后续替换
3. **先功能后性能**：先用最简单的容器满足需求，性能不够再优化

> 🔴 **口诀**：数组是毛坯房，Collection 是精装修，先住进去再考虑要不要改造。

---

<a id="level-2"></a>

## Level 2：List 的三种面孔 —— ArrayList vs LinkedList vs Vector

### 👶 前置知识关卡

- [ ] 什么是连续内存和离散内存？
- [ ] CPU 缓存行（Cache Line）的概念？
- [ ] 什么是时间复杂度的"均摊"？

### Why——上一代方案死在哪

**徒弟**：ArrayList 和 LinkedList 不就是一个快一个慢吗？

**老陈**：这个说法对，但不够深。我们先看一个真实场景：

| 场景 | ArrayList | LinkedList | 谁赢？ |
|------|-----------|------------|--------|
| 随机访问 `get(9999)` | O(1)，直接算地址 | O(n)，从头遍历 | ArrayList **完胜** |
| 尾部追加 `add(e)` | 均摊 O(1)，偶尔扩容 O(n) | O(1)，直接挂节点 | 平手 |
| 中间插入 `add(5000, e)` | O(n)，后面元素全移 | O(1)，改指针（如果定位到） | LinkedList 理论胜 |
| 内存占用 | 连续，CPU 缓存友好 | 离散，每个节点额外 24 字节指针开销 | ArrayList **完胜** |

**徒弟**：那中间插入 LinkedList 应该赢啊？

**老陈**：**理论上是，但现实中不是**。因为 LinkedList 要先定位到第 5000 个节点，这个定位本身就是 O(n)。而 ArrayList 的 `System.arraycopy()` 是 native 方法，CPU 有专门的指令优化，实际速度远超理论复杂度。

```
ArrayList 内存布局（连续）：
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  ← 索引访问：O(1)
└───┴───┴───┴───┴───┴───┴───┴───┘
  ↑                               ↑
  首地址                          末地址
  CPU 缓存命中率：高（预取机制）

LinkedList 内存布局（离散）：
[Node0] → [Node1] → [Node2] → [Node3] → ...
  ↑           ↑           ↑           ↑
 0x1000     0x3456     0x789A     0xBCDE   ← 内存地址不连续
  prev       prev       prev       prev     ← 每个节点额外 24 字节
  next       next       next       next     ← (prev + next + item) × 8
```

### What——三种 List 的本质区别

| 特性 | ArrayList | LinkedList | Vector |
|------|-----------|------------|--------|
| **底层结构** | 动态数组 | 双向链表 | 动态数组 |
| **扩容策略** | 1.5 倍（`oldCapacity + (oldCapacity >> 1)`） | 无需扩容（逐个挂节点） | 2 倍（`newCapacity = oldCapacity * 2`） |
| **线程安全** | ❌ | ❌ | ✅（`synchronized`） |
| **随机访问** | O(1) | O(n) | O(1) |
| **插入删除** | O(n) | O(1)（定位后） | O(n) |
| **内存开销** | 少（只存数据） | 多（24字节指针/节点） | 少 |
| **推荐场景** | 绝大多数场景 | 频繁头部插入删除 | ❌ 已过时，用 `Collections.synchronizedList` |

> 📌 **最容易被误解的一点**：`Vector` 已经过时，不要在新代码中使用。它的 `synchronized` 是方法级锁，粒度太粗，并发性能差。

### How——ArrayList 扩容的慢动作

```java
// JDK 21 ArrayList.add() 源码（简化版）
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // ① 检查是否需要扩容
    elementData[size++] = e;           // ② 直接赋值，O(1)
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity); // 默认容量 10
    }
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++; // fail-fast 计数器
    if (minCapacity - elementData.length > 0) {
        grow(minCapacity); // ③ 触发扩容
    }
}

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // ④ 1.5 倍扩容
    if (newCapacity - minCapacity < 0) {
        newCapacity = minCapacity;
    }
    elementData = Arrays.copyOf(elementData, newCapacity); // ⑤ 复制整个数组
}
```

**徒弟**：为什么要 1.5 倍而不是 2 倍？

**老陈**：这是一个来自 C++ 容器设计的经典论证。1.5 倍扩容时，新分配的内存在未来可能复用之前释放的内存（因为 1 + 1.5 = 2.5 < 2 × 1.5 = 3，旧内存块的大小小于后续某次需要的大小，理论上有机会被重用）。2 倍扩容时，之前释放的内存永远不够用，碎片会持续累积。

> ⚠️ **免责声明**：这个论证在 C++ 手动内存管理下成立（malloc/free 的碎片复用行为较可预测）。但在 Java JVM 堆上，GC 的内存回收/分配行为取决于具体垃圾回收器实现（G1、ZGC、Shenandoah 等策略各不同），**是否真的复用了"释放"的数组内存没有统一结论**。1.5 倍这个数值在 Java 中更多是一个"经验值"，而非严格的数学最优解。教学参考，以你的部署实测为准。

### Transfer——迁移到其他设计问题

1. **不要凭直觉选 LinkedList**：99% 的场景 ArrayList 更快，除非你能证明"头部频繁插入且从不随机访问"
2. **容量预估**：如果知道大概数量，`new ArrayList<>(10000)` 可以避免多次扩容
3. **subList 是视图**：`list.subList(0, 5)` 返回的是原 list 的视图，修改会影响原 list

> 🔴 **口诀**：ArrayList 是默认选择，LinkedList 是特殊情况，Vector 是历史遗留。

**本层留下的新账**：ArrayList 的 `contains()` 是 O(n) 查找，100 万个元素遍历一次要多久？→ 引出 HashSet/HashMap（Level 3-4）

---

<a id="level-3"></a>

## Level 3：哈希的本质 —— 从数组到 HashMap 的演进

### 👶 前置知识关卡

- [ ] 数组的随机访问为什么是 O(1)？
- [ ] 什么是模运算（取余）？
- [ ] 为什么两个不同的输入可能有相同的输出（鸽巢原理）？

### Why——上一代方案死在哪

**徒弟**：ArrayList 的 `contains()` 是 O(n)，我能不能用二分查找优化到 O(log n)？

**老陈**：可以，但需要排序。排序是 O(n log n)，而且每次插入都要维护有序性。有没有 O(1) 的查找？

**徒弟**：数组直接索引访问是 O(1)，但 key 不一定是整数啊...

**老陈**：**这就是哈希的核心思想：把任意类型的 key 映射成整数索引**。

| 方案 | 查找复杂度 | 插入复杂度 | 问题 |
|------|-----------|-----------|------|
| 线性扫描 | O(n) | O(1) | 太慢 |
| 二分查找 | O(log n) | O(n)（维护有序） | 插入太慢 |
| 哈希表 | **O(1) 均摊** | **O(1) 均摊** | 冲突处理 |

### What——哈希函数是什么

```
哈希函数的本质：key → 整数 → 数组索引

"apple" → hash("apple") = 96354 → 96354 % 16 = 2 → 存入 table[2]
"banana" → hash("banana") = 123456 → 123456 % 16 = 0 → 存入 table[0]
"cherry" → hash("cherry") = 789012 → 789012 % 16 = 4 → 存入 table[4]
```

> 📌 **最容易被误解的一点**：哈希值 ≠ 数组索引。`hashCode()` 返回的是 32 位整数，数组长度通常远小于 2^32，所以还需要**取模运算**得到索引。

### How——寻址过程的慢动作

```
① 调用 key.hashCode()     → 得到 32 位整数 hash
② 高 16 位异或低 16 位      → 扰动函数，减少冲突（JDK 8 优化）
③ hash & (n - 1)           → 等价于 hash % n，但更快（位运算）
④ 得到数组索引 i            → table[i] 就是目标桶

为什么用 & 而不是 %？
- n 是 2 的幂次时，hash & (n-1) == hash % n
- & 是 CPU 指令，1 个时钟周期
- % 需要除法，几十个时钟周期
```

**徒弟**：如果两个 key 算出来的索引一样怎么办？

**老陈**：这就是**哈希冲突**。解决方案有两种：

```
方案 1：链地址法（Java 7 及以前）
┌───┐
│ 0 │ → [Entry] → [Entry] → [Entry] → null
├───┤
│ 1 │ → [Entry] → null
├───┤
│ 2 │ → null
├───┤
│ 3 │ → [Entry] → [Entry] → null
└───┘

方案 2：链地址法 + 红黑树（Java 8+，链表长度 > 8 且容量 ≥ 64）
┌───┐
│ 0 │ → [TreeNode]（红黑树根）
├───┤     ↙     ↘
│ 1 │ → [Entry] → null    [TreeNode]  [TreeNode]
├───┤
│ 2 │ → null
├───┤
│ 3 │ → [Entry] → [Entry] → null（长度 ≤ 6，保持链表）
└───┘
```

### Transfer——迁移到其他设计问题

1. **哈希函数的质量决定了性能**：均匀分布是关键，避免所有 key 都映射到同一个桶
2. **负载因子（load factor）**：`元素数量 / 桶数量`，默认 0.75，超过就扩容
3. **为什么是 0.75**：数学推导，0.75 在空间和时间之间找到平衡点（太低浪费空间，太高冲突增多）

> 🔴 **口诀**：哈希是用空间换时间的艺术，冲突是代价，负载因子是平衡点。

**本层留下的新账**：负载因子超过 0.75 就扩容，扩容时所有元素要重新哈希？这也太慢了吧？→ 引出 HashMap 的扩容优化（Level 4）

---

<a id="level-4"></a>

## Level 4：HashMap 源码级拆解 —— 从 put() 到红黑树

### 👶 前置知识关卡

- [ ] 什么是链地址法解决哈希冲突？
- [ ] 红黑树的基本性质（左根右、根叶黑、不红红、黑路同）？
- [ ] 为什么 HashMap 的容量必须是 2 的幂次？

### Why——上一代方案死在哪

**徒弟**：HashMap 的 `put()` 不就是算个索引然后存进去吗？有什么复杂的？

**老陈**：我们来看一个真实场景。假设有一个热点 key `"user:login:count"`，每秒被访问 10 万次。如果所有热点 key 都冲突到同一个桶，链表长度达到 1000：

```
桶 5：[Entry] → [Entry] → ... → [Entry]（1000 个节点）
                                ↑
                            每次 get() 都要遍历 1000 个节点
                            O(1) 退化成 O(1000)
```

**徒弟**：那我换个好的哈希函数不就行了？

**老陈**：哈希函数只能减少冲突，不能完全避免。而且恶意用户可以构造大量相同哈希值的 key（**哈希碰撞攻击**），让服务器 CPU 飙到 100%。

### What——HashMap 的结构演进

```
JDK 7：数组 + 链表
┌───┐
│ 0 │ → null
├───┤
│ 1 │ → [Entry] → [Entry] → null
├───┤
│ 2 │ → null
├───┤
│ 3 │ → [Entry] → null
└───┘
问题：链表太长时 O(n) 查找

JDK 8+：数组 + 链表 + 红黑树
┌───┐
│ 0 │ → null
├───┤
│ 1 │ → [TreeNode]（红黑树根，链表长度 > 8 且容量 ≥ 64 时树化）
├───┤     ↙     ↘
│ 2 │ → null    [TreeNode]  [TreeNode]
├───┤
│ 3 │ → [Entry] → [Entry] → null（长度 ≤ 6 时退化回链表）
└───┘
```

> 📌 **最容易被误解的一点**：树化有两个条件：**链表长度 > 8** 且 **容量 ≥ 64**。如果容量 < 64，优先扩容而不是树化。

### How——put() 的完整时序

```java
// JDK 21 HashMap.put() 源码（简化版）
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // ① 如果 table 为空，初始化（懒加载）
    if ((tab = table) == null || (n = tab.length) == 0) {
        n = (tab = resize()).length; // 默认容量 16
    }
    
    // ② 计算索引，如果桶为空，直接放入
    if ((p = tab[i = (n - 1) & hash]) == null) {
        tab[i] = newNode(hash, key, value, null);
    } else {
        Node<K,V> e; K k;
        
        // ③ 如果桶的第一个节点就是目标 key
        if (p.hash == hash && 
            ((k = p.key) == key || (key != null && key.equals(k)))) {
            e = p;
        }
        // ④ 如果是红黑树节点，走红黑树插入
        else if (p instanceof TreeNode) {
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        }
        // ⑤ 链表遍历，尾插法（JDK 8 改为尾插，解决 JDK 7 的死循环问题）
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) { // 链表长度 > 8
                        treeifyBin(tab, hash); // ⑥ 树化（如果容量 ≥ 64）
                    }
                    break;
                }
                if (e.hash == hash && 
                    ((k = e.key) == key || (key != null && key.equals(k)))) {
                    break; // 找到相同 key
                }
                p = e;
            }
        }
        
        // ⑦ 覆盖旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null) {
                e.value = value;
            }
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    ++modCount;
    
    // ⑧ 超过阈值，扩容
    if (++size > threshold) {
        resize();
    }
    afterNodeInsertion(evict);
    return null;
}
```

**徒弟**：为什么 JDK 8 要改成尾插法？

**老陈**：JDK 7 的头插法在多线程扩容时会导致**链表成环**，形成死循环：

```
JDK 7 头插法扩容（多线程场景）：
线程 A：A → B → null  （头插后）
线程 B：B → A → null  （并发头插）
结果：A.next = B, B.next = A  → 死循环！

JDK 8 尾插法：
始终从尾部追加，不会改变已有节点的顺序，避免成环
```

**徒弟**：但 HashMap 不是线程安全的啊，为什么还要修复？

**老陈**：**不是为了保证线程安全，而是为了防止死循环**。即使 HashMap 不保证并发正确性，也不应该让 JVM 卡死。这是一个"防御性编程"的思想。

### 扩容的慢动作

```
扩容前（容量 16，阈值 12，已有 12 个元素）：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12 │13 │14 │15 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

插入第 13 个元素，触发扩容：
① 新建容量 32 的数组
② 遍历旧数组每个桶
③ 对每个元素，hash & 31（新容量-1），要么留在原位，要么移到 原位置+16

扩容后（容量 32，阈值 24）：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12 │13 │14 │15 │ ← 原位置
├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
│16 │17 │18 │19 │20 │21 │22 │23 │24 │25 │26 │27 │28 │29 │30 │31 │ ← 原位置+16
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

**徒弟**：扩容时要重新计算所有元素的位置，这不是 O(n) 吗？

**老陈**：是的，但这是**均摊**。扩容后，下次扩容要等到元素数量再翻倍。而且 JDK 8 优化了：`(hash & oldCap) == 0` 的元素留在原位，`!= 0` 的移到 `原位置 + oldCap`，不需要重新计算哈希值。

### Transfer——迁移到其他设计问题

1. **初始容量估算**：如果知道大概元素数量，`new HashMap<>(expectedSize / 0.75)` 可以避免多次扩容
2. **Key 的要求**：必须正确实现 `hashCode()` 和 `equals()`，否则会导致"存进去取不出来"
3. **线程不安全**：多线程环境下使用 `ConcurrentHashMap`（Level 6 详解）

> 🔴 **口诀**：HashMap 是数组+链表+红黑树的混合体，负载因子 0.75 是平衡点，链表长度 8 是树化阈值。

**本层留下的新账**：HashMap 不保证顺序，如果需要按 key 排序怎么办？→ 引出 TreeMap（Level 5）

---

<a id="level-5"></a>

## Level 5：TreeMap 与排序 —— 红黑树的另一面

### 👶 前置知识关卡

- [ ] 什么是二叉搜索树（BST）？
- [ ] BST 在什么情况下会退化成链表？
- [ ] 红黑树的五个性质？

### Why——上一代方案死在哪

**徒弟**：我需要按 key 排序输出，HashMap 做不到啊？

**老陈**：HashMap 的 key 是无序的（实际按哈希值存储）。你有几个选择：

| 方案 | 时间复杂度 | 问题 |
|------|-----------|------|
| 每次排序：`new TreeMap<>(map)` | O(n log n) | 每次都要排序，太慢 |
| 用 `LinkedHashMap` | O(1) 插入 | 只保证插入顺序，不保证 key 大小顺序 |
| 用 `TreeMap` | O(log n) 插入/查找 | **保证 key 有序** |

### What——TreeMap 是什么

```
TreeMap 底层：红黑树

          [5]
         /   \
       [3]    [8]
      / \    / \
    [1] [4] [6] [9]
    
中序遍历：1 → 3 → 4 → 5 → 6 → 8 → 9（有序！）
```

> 📌 **最容易被误解的一点**：TreeMap 的 key 必须实现 `Comparable` 接口，或者在构造时传入 `Comparator`。否则会抛 `ClassCastException`。

### How——为什么用红黑树而不是 AVL 树

| 特性 | AVL 树 | 红黑树 |
|------|--------|--------|
| **平衡标准** | 严格平衡（高度差 ≤ 1） | 近似平衡（最长路径 ≤ 2 × 最短路径） |
| **查找** | O(log n)，略快 | O(log n) |
| **插入/删除** | 可能需要多次旋转 | **最多 2-3 次旋转** |
| **实现复杂度** | 高 | **低** |
| **工程选择** | 查找密集场景 | **通用场景（Java/C++/Linux 内核）** |

**徒弟**：为什么 Java 选择了红黑树？

**老陈**：**工程决策**。这里有个关键区分——**插入 vs 删除**：

```
插入旋转次数：
┌──────────────────────────────────────────────────────────────┐
│ 红黑树插入：                                                 │
│ - 最好：0 次（直接染色）                                     │
│ - 最坏：2 次基础旋转 + 若干次染色                            │
│ - 平均：接近 1 次                                            │
│                                                              │
│ AVL 树插入：                                                 │
│ - 最好：0 次（已经平衡）                                     │
│ - 最坏：1 次旋转（单旋或双旋，均为 1 次再平衡操作）           │
│ - 关键：一旦在最低失衡节点完成旋转，子树高度恢复到插入前的值  │
│         失衡不会向上传播，所以 O(1) 即可终止                  │
└──────────────────────────────────────────────────────────────┘

删除旋转次数（真正的分水岭）：
┌──────────────────────────────────────────────────────────────┐
│ 红黑树删除：                                                 │
│ - 最坏：3 次旋转                                             │
│                                                              │
│ AVL 树删除：                                                 │
│ - 最坏：O(log n) 次旋转                                      │
│ - 关键：删除可能导致子树高度变化，失衡一路传播到根            │
└──────────────────────────────────────────────────────────────┘
```

**徒弟**：那插入时两者差不多，为什么还要选红黑树？

**老陈**：因为**删除才是决定性因素**。实际应用中，增删改查是混合的，删除操作会触发 AVL 的最坏情况 O(log n) 次旋转，而红黑树始终稳定在常数次。更关键的是：

1. **实现复杂度**：AVL 需要维护每个节点的高度字段，红黑树只需 1 位颜色标记
2. **工程一致性**：Java/C++/Linux 内核统一选择红黑树，生态成熟、bug 少
3. **均摊稳定**：红黑树的旋转次数上界是常数，写延迟可预测

> 📌 **面试高频追问**："AVL 插入最坏几次旋转？"答案是 **1 次**（单旋或双旋）。不要答 O(log n)，那是删除。如果面试官问"那为什么不选 AVL？"，把论证聚焦到删除的 O(log n) 旋转和工程实现复杂度上。

### 红黑树四种失衡情况与旋转图解

```
红黑树的五个性质（再确认）：
1. 每个节点非红即黑
2. 根节点是黑色
3. 叶子节点（NIL）是黑色
4. 红色节点的两个子节点必须是黑色（不红红）
5. 从任一节点到其所有叶子的路径上，黑色节点数量相同（黑路同）

四种失衡情况（插入导致）：

情况 1：LL 失衡（左左）→ 右旋
        [G]                [P]
       /                  /   \
     [P]        →      [X]    [G]
     /
   [X]
  新插入的红色节点 X 在 P 的左侧，P 在 G 的左侧
  解决：以 G 为轴右旋，P 变成新根

情况 2：RR 失衡（右右）→ 左旋
   [G]                    [P]
      \                  /   \
      [P]      →      [G]    [X]
        \
         [X]
  新插入的红色节点 X 在 P 的右侧，P 在 G 的右侧
  解决：以 G 为轴左旋，P 变成新根

情况 3：LR 失衡（左右）→ 先左旋再右旋
      [G]              [G]              [X]
     /                /                /   \
   [P]      →      [X]      →      [P]    [G]
      \            /
      [X]        [P]
  新插入的红色节点 X 在 P 的右侧，P 在 G 的左侧
  解决：先以 P 为轴左旋变成 LL，再以 G 为轴右旋

情况 4：RL 失衡（右左）→ 先右旋再左旋
   [G]                [G]                  [X]
      \                  \                /   \
      [P]      →        [X]      →    [G]    [P]
     /                    \
   [X]                    [P]
  新插入的红色节点 X 在 P 的左侧，P 在 G 的右侧
  解决：先以 P 为轴右旋变成 RR，再以 G 为轴左旋
```

> 📌 **插入 vs 删除的传播性**：插入时，新节点是红色。如果父节点也是红色（违反"不红红"），通过旋转+染色可以把问题"吸收"掉——旋转后子树高度不变，祖父以上节点不受影响。但删除时，如果删掉一个黑色节点，某条路径少了一个黑色节点（违反"黑路同"），修复时可能需要把兄弟节点变色，这会导致兄弟子树也少一个黑色节点，问题向上蔓延直到根。

### Transfer——迁移到其他设计问题

1. **TreeMap 是有序的**：`firstKey()`、`lastKey()`、`headMap()`、`tailMap()` 都是 O(log n)
2. **TreeMap 不允许 null key**：因为 null 无法比较大小
3. **TreeMap 的 `containsValue()` 是 O(n)**：因为值没有排序

> 🔴 **口诀**：HashMap 是无序的快枪手，TreeMap 是有序的慢郎中，按需选择。

**本层留下的新账**：HashMap 无序、TreeMap 有序但慢，有没有"保持插入顺序"的方案？→ 引出 LinkedHashMap（Level 5.5）

---

<a id="level-5-5"></a>

## Level 5.5：LinkedHashMap 与 LRU —— HashMap 的有序变体

### 👶 前置知识关卡

- [ ] HashMap 的 Node 节点除了 next 还有什么字段？
- [ ] 什么是 LRU（Least Recently Used）淘汰策略？
- [ ] 双向链表的插入/删除为什么是 O(1)？

### Why——上一代方案死在哪

**徒弟**：我需要一个缓存，按访问顺序淘汰最久没用的 key。HashMap 不保证顺序，TreeMap 按 key 排序不是我要的，怎么办？

**老陈**：三种方案对比：

| 方案 | 插入顺序 | 访问顺序 | O(1) get/put | 自动淘汰 |
|------|---------|---------|-------------|---------|
| `HashMap` | ❌ | ❌ | ✅ | ❌ |
| `TreeMap` | ❌ | ❌（按 key 排序） | O(log n) | ❌ |
| `LinkedHashMap` | ✅ | ✅（`accessOrder=true`） | ✅ | ✅（重写 `removeEldestEntry`） |

**LinkedHashMap 就是 HashMap + 双向链表**，链表维护顺序，哈希表维护查找性能，两者叠加就是 O(1) 查找 + 有序遍历。

### What——LinkedHashMap 的结构

```
HashMap 底层（无序）：
┌───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │  ← 桶，遍历顺序取决于哈希值，不可预测
└───┴───┴───┴───┘

LinkedHashMap 底层（有序）：
┌───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │  ← 桶（继承 HashMap）
└───┴───┴───┴───┘
        ↓
head ↔ A ↔ B ↔ C ↔ D ↔ tail   ← 额外的双向链表，维护插入/访问顺序
```

> 📌 **最容易被误解的一点**：`LinkedHashMap` 继承自 `HashMap`，**哈希表部分完全一样**，只是额外维护了一条双向链表。所以它的 `get()`/`put()` 仍然是 O(1)，不会变成 O(n) 或 O(log n)。

### How——LRU 缓存的慢动作

```java
// 基于 LinkedHashMap 实现 LRU 缓存（面试高频手写题）
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxCapacity;

    public LRUCache(int maxCapacity) {
        // initialCapacity: 初始容量
        // loadFactor: 负载因子
        // accessOrder: true = 按访问顺序（LRU），false = 按插入顺序
        super(maxCapacity, 0.75f, true);
        this.maxCapacity = maxCapacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // 当 size 超过最大容量时，自动移除最老的 entry
        return size() > maxCapacity;
    }
}

// 使用
LRUCache<String, Session> cache = new LRUCache<>(1000);
cache.put("user:123", session1);  // 插入，链表尾部
cache.get("user:123");            // 访问，移到链表尾部（最近使用）
cache.put("user:456", session2);  // 插入，如果超过 1000，链表头部（最久未用）自动淘汰
```

```
accessOrder=true 时的访问行为：

put("A", 1)：head ↔ A ↔ tail
put("B", 2)：head ↔ A ↔ B ↔ tail
get("A")：   head ↔ B ↔ A ↔ tail  ← A 被访问，移到尾部
put("C", 3)：head ↔ B ↔ A ↔ C ↔ tail  ← 如果容量=3，刚好满
put("D", 4)：head ↔ A ↔ C ↔ D ↔ tail  ← B 被淘汰（最久未访问）
```

**徒弟**：`removeEldestEntry` 是在哪里调用的？

**老陈**：在 `put()` 方法的末尾，`afterNodeInsertion()` 里调用。JDK 源码：

```java
// HashMap.afterNodeInsertion() —— LinkedHashMap 重写了这个方法
void afterNodeInsertion(boolean evict) {
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        // 移除链表头部（最久未用的 entry）
        removeNode(hash(first.key), first.key, first.value, false, true);
    }
}
```

### Transfer——迁移到其他设计问题

1. **面试手写 LRU**：继承 `LinkedHashMap` 是最简洁的方案，3 行核心代码
2. **生产环境**：用 Caffeine（Java 8+）或 Guava Cache，它们基于更复杂的 W-TinyLFU 算法，命中率比 LRU 更高
3. **`accessOrder=false`（默认）**：按插入顺序遍历，类似 `LinkedHashSet` 的行为
4. **线程不安全**：`LinkedHashMap` 不是线程安全的，并发场景需要外部加锁或用 `ConcurrentLinkedHashMap`（Caffeine 的前身）

> 🔴 **口诀**：HashMap 是无序快枪手，TreeMap 是有序慢郎中，LinkedHashMap 是保持队形的快枪手——O(1) 查找 + 有序遍历 + LRU 淘汰，三合一。

---

### 📌 不可变集合与工具类（简要提及）

```java
// JDK 9+ 不可变集合工厂（现在生产代码里很常见）
List<String> names = List.of("Alice", "Bob", "Charlie");  // 不可变，add/remove 抛异常
Map<String, Integer> scores = Map.of("Alice", 90, "Bob", 85);  // 不可变
Set<String> tags = Set.of("java", "collection");  // 不可变

// JDK 8 及以前的不可变包装
List<String> unmodifiable = Collections.unmodifiableList(list);  // 视图，不是副本
```

> ⚠️ `Collections.unmodifiableXxx()` 返回的是**视图**，修改原 list 会影响它。`List.of()` 返回的是**真正的不可变对象**，底层没有任何可变引用。

---

**本层留下的新账**：HashMap、TreeMap、LinkedHashMap 都不是线程安全的，高并发怎么办？→ 引出 ConcurrentHashMap（Level 6）

---

<a id="level-6"></a>

## Level 6：并发容器 —— 从 Collections.synchronizedMap 到 ConcurrentHashMap

### 👶 前置知识关卡

- [ ] 什么是 synchronized？什么是 CAS？
- [ ] 什么是分段锁（Segment Locking）？
- [ ] 什么是 CPU 缓存行（Cache Line）？

### Why——上一代方案死在哪

**徒弟**：多线程环境下，我用 `Collections.synchronizedMap(new HashMap<>())` 不就行了？

**老陈**：我们来压测一下：

| 方案 | 100 线程并发读写 | 问题 |
|------|-----------------|------|
| `synchronizedMap` | 吞吐量 10 万 QPS | **所有操作串行化**，锁粒度太粗 |
| `ConcurrentHashMap`（JDK 7） | 吞吐量 50 万 QPS | 分段锁，16 个段 |
| `ConcurrentHashMap`（JDK 8+） | 吞吐量 100 万 QPS | **CAS + synchronized（桶级别）** |

**徒弟**：为什么 JDK 8 要放弃分段锁？

**老陈**：分段锁的问题是**锁粒度固定**。假设你有 16 个段，每个段有 100 万个 key，遍历整个 Map 需要锁住所有 16 个段，退化成串行。JDK 8 改为**桶级别锁**，只锁住发生冲突的那个桶。

### What——ConcurrentHashMap 的演进

```
JDK 7：Segment 分段锁
┌─────────────────────────────────────────────┐
│ ConcurrentHashMap                            │
│ ┌──────┬──────┬──────┬──────┬──────┬──────┐ │
│ │Seg 0 │Seg 1 │Seg 2 │Seg 3 │ ...  │Seg 15│ │
│ └──────┴──────┴──────┴──────┴──────┴──────┘ │
│  ↑ 每个 Segment 是一个 ReentrantLock        │
│  ↑ 最多 16 个线程并发                        │
└─────────────────────────────────────────────┘

JDK 8+：CAS + synchronized（桶级别）
┌─────────────────────────────────────────────┐
│ ConcurrentHashMap                            │
│ ┌───┬───┬───┬───┬───┬───┬───┬───┐           │
│ │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │...│ ← 桶     │
│ └───┴───┴───┴───┴───┴───┴───┴───┘           │
│  ↑ 空桶：CAS 插入（无锁）                    │
│  ↑ 非空桶：synchronized（锁住桶头节点）       │
│  ↑ 红黑树：synchronized（锁住树根节点）       │
└─────────────────────────────────────────────┘
```

> 📌 **最容易被误解的一点**：`ConcurrentHashMap` 不允许 null key 和 null value。`HashMap` 允许。这是设计决策，因为 null 在并发环境下语义模糊（是 key 不存在，还是 value 就是 null？）。

### How——put() 的并发安全时序

```java
// JDK 21 ConcurrentHashMap.put() 源码（简化版）
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode()); // 扰动函数
    
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        
        // ① 桶为空，CAS 插入（无锁）
        if (tab == null || (n = tab.length) == 0) {
            tab = initTable();
        } else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null))) {
                break; // CAS 成功
            }
            // CAS 失败，说明有其他线程插入了，重试
        }
        // ② 正在扩容，帮助扩容
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
        }
        // ③ 桶非空，synchronized 锁住桶头节点
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) { // 双重检查
                    if (fh >= 0) {
                        // 链表遍历，尾插法
                        for (Node<K,V> e = f;; ++binCount) {
                            if (e.hash == hash && 
                                ((ek = e.key) == key || key.equals(ek))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent) {
                                    e.val = value;
                                }
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                if (binCount >= TREEIFY_THRESHOLD - 1) {
                                    treeifyBin(tab, i); // 树化
                                }
                                break;
                            }
                        }
                    } else if (f instanceof TreeBin) {
                        // 红黑树插入
                        Node<K,V> p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value);
                        if (p != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent) {
                                p.val = value;
                            }
                        }
                    }
                }
            }
            if (oldVal != null) {
                return oldVal;
            }
        }
    }
    
    addCount(1L, binCount); // ④ 计数器（CAS + CounterCell 数组，减少竞争）
    return null;
}
```

**徒弟**：为什么计数器要用 `CounterCell` 数组？

**老陈**：如果只有一个 `size` 变量，所有线程都要 CAS 更新它，CAS 失败率会很高。`CounterCell` 数组的思路是：**每个线程更新自己的 CounterCell，最后汇总**。类似 `LongAdder` 的设计（JDK 8+）。

```
单变量 CAS：
Thread1 ─→ CAS(size) ──→ 失败，重试
Thread2 ─→ CAS(size) ──→ 失败，重试
Thread3 ─→ CAS(size) ──→ 成功

CounterCell 数组：
Thread1 ─→ CAS(counter[0]) ──→ 成功
Thread2 ─→ CAS(counter[1]) ──→ 成功
Thread3 ─→ CAS(counter[2]) ──→ 成功
汇总：size = counter[0] + counter[1] + counter[2]
```

### 扩容时的多线程协助机制（transfer）

```
ConcurrentHashMap 扩容的核心创新：多线程协助搬迁

传统 HashMap 扩容：
┌────────────────────────────────────────────┐
│ 只有触发扩容的那一个线程负责搬迁所有桶       │
│ 其他线程必须等待 → 扩容期间吞吐量骤降       │
└────────────────────────────────────────────┘

ConcurrentHashMap 扩容（JDK 8+）：
┌────────────────────────────────────────────┐
│ 任何线程发现桶上是 ForwardingNode（MOVED）   │
│ → 说明该桶正在搬迁                          │
│ → 该线程主动帮忙搬迁其他桶（helpTransfer）   │
│ → 多线程并行搬迁，扩容速度与线程数成正比     │
└────────────────────────────────────────────┘

搬迁过程时序：
┌─────────────────────────────────────────────────────────────┐
│ T0: 线程 A 触发扩容                                         │
│     - 新建 nextTable（2倍大小）                              │
│     - 设置 transferIndex = oldTable.length                  │
│                                                             │
│ T1: 线程 A 开始搬迁                                         │
│     - 从 transferIndex 向前领取一段桶（stride=16）           │
│     - 遍历这 16 个桶，逐个搬到 nextTable                    │
│     - 搬完的桶放 ForwardingNode（hash=MOVED）               │
│                                                             │
│ T2: 线程 B 访问 table[i]                                    │
│     - 发现 table[i] 是 ForwardingNode                       │
│     - 调用 helpTransfer()，加入搬迁                         │
│     - 从 transferIndex 领取下一段桶                         │
│                                                             │
│ T3: 所有桶搬迁完成                                          │
│     - table = nextTable                                     │
│     - nextTable = null                                      │
└─────────────────────────────────────────────────────────────┘

ForwardingNode 的作用：
┌─────────────────────────────────────────────────────────────┐
│ - hash = MOVED（特殊标记值 -1）                              │
│ - 持有 nextTable 引用                                        │
│ - get() 遇到它 → 转发到 nextTable 查找                      │
│ - put() 遇到它 → 帮助搬迁（helpTransfer）                   │
│ - 相当于一个"路障+指示牌"：这个桶已经搬走了，去新地址找       │
└─────────────────────────────────────────────────────────────┘
```

> 📌 **面试高频追问**："ConcurrentHashMap 扩容时其他线程在做什么？"答案是：**其他线程不会阻塞，而是主动帮忙搬迁**（helpTransfer）。这是 ConcurrentHashMap 吞吐量远超 synchronizedMap 的关键原因之一——连扩容这种"重量级操作"都被分摊到多个线程了。

### Transfer——迁移到其他设计问题

1. **不要用 `Collections.synchronizedMap`**：除了只读场景，`ConcurrentHashMap` 在所有场景都更好
2. **`size()` 是近似值**：`ConcurrentHashMap.size()` 返回的是估计值，不是精确值
3. **复合操作需要额外同步**：`if (!map.containsKey(key)) map.put(key, value)` 不是原子的

> 🔴 **口诀**：JDK 8 之后，ConcurrentHashMap 是并发 Map 的唯一正确选择。

**本层留下的新账**：遍历 ConcurrentHashMap 时，其他线程在修改怎么办？→ 引出 Iterator 和 Fail-Fast/Fail-Safe（Level 7）

---

<a id="level-7"></a>

## Level 7：Iterator 与 Fail-Fast/Fail-Safe 机制

### 👶 前置知识关卡

- [ ] 什么是快速失败（Fail-Fast）？
- [ ] 什么是安全失败（Fail-Safe）？
- [ ] `modCount` 的作用？

### Why——上一代方案死在哪

**徒弟**：我遍历 ArrayList 时删除元素，报了 `ConcurrentModificationException`，为什么？

**老陈**：这是 **Fail-Fast** 机制。看这个例子：

```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C", "D"));
for (String s : list) {
    if ("B".equals(s)) {
        list.remove(s); // ❌ 抛出 ConcurrentModificationException
    }
}
```

**徒弟**：为什么？

**老陈**：因为 `for-each` 循环底层是 Iterator：

```java
// 编译器将 for-each 转换成：
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if ("B".equals(s)) {
        list.remove(s); // 直接调用 list.remove()，modCount++
    }
}
// 下次调用 it.next() 时，检查 modCount 是否变化
// 变化了 → 抛出 ConcurrentModificationException
```

### What——Fail-Fast vs Fail-Safe

| 特性 | Fail-Fast | Fail-Safe |
|------|-----------|-----------|
| **代表** | ArrayList, HashMap, HashSet | CopyOnWriteArrayList, ConcurrentHashMap |
| **机制** | 检测 `modCount` 变化 | 遍历的是**快照**或**弱一致性视图** |
| **行为** | 抛 `ConcurrentModificationException` | 不抛异常，但可能看不到最新修改 |
| **适用场景** | 单线程或外部加锁 | 并发环境 |

```
Fail-Fast 机制：
┌─────────────────────────────────────────┐
│ ArrayList                                │
│ ┌─────────────────────────────────────┐ │
│ │ modCount = 5                        │ │
│ │ iterator.expectedModCount = 5       │ │ ← 创建时记录
│ │                                     │ │
│ │ list.remove() → modCount = 6        │ │ ← 修改了
│ │ it.next() → 检查 6 != 5            │ │ ← 不一致！
│ │            → 抛出 ConcurrentMod... │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘

Fail-Safe 机制（CopyOnWriteArrayList）：
┌─────────────────────────────────────────┐
│ CopyOnWriteArrayList                     │
│ ┌─────────────────────────────────────┐ │
│ │ 遍历时：拿到当前数组的快照（引用）    │ │
│ │ 删除时：复制一份新数组，在新数组上删除 │ │
│ │         然后替换引用（volatile）      │ │
│ │                                     │ │
│ │ 遍历者看到的还是旧数组，不受影响      │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

> 📌 **最容易被误解的一点**：Fail-Fast 不是为了保证线程安全，而是为了**尽早发现问题**。它是一个"善意的崩溃"，比"静默的数据错误"好得多。

### How——安全遍历的慢动作

```java
// 方案 1：使用 Iterator 的 remove()（单线程安全）
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if ("B".equals(s)) {
        it.remove(); // ✅ 使用 Iterator 的 remove，会同步更新 expectedModCount
    }
}

// 方案 2：使用 removeIf()（JDK 8+，内部使用 Iterator）
list.removeIf("B"::equals);

// 方案 3：使用 CopyOnWriteArrayList（并发安全，但写操作代价高）
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>(list);
for (String s : cowList) {
    if ("B".equals(s)) {
        cowList.remove(s); // ✅ 不会抛异常，但遍历者看不到这个删除
    }
}

// 方案 4：使用 ConcurrentHashMap 的弱一致性迭代器
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put("A", "1");
map.put("B", "2");
for (Map.Entry<String, String> entry : map.entrySet()) {
    if ("B".equals(entry.getKey())) {
        map.remove("B"); // ✅ 不会抛异常，但遍历可能看到也可能看不到这个删除
    }
}
```

**徒弟**：CopyOnWriteArrayList 的写操作代价高在哪里？

**老陈**：每次 `add()`、`remove()` 都要**复制整个数组**。假设数组有 100 万个元素，删除一个元素的代价是 O(n)。所以它只适合**读多写少**的场景，比如事件监听器列表。

### Transfer——迁移到其他设计问题

1. **单线程遍历时删除**：使用 `Iterator.remove()` 或 `removeIf()`
2. **多线程并发读写**：使用 `ConcurrentHashMap` 或 `CopyOnWriteArrayList`
3. **不要在 for-each 中直接调用 `list.remove()`**：这是最常见的坑

> 🔴 **口诀**：Fail-Fast 是善意的崩溃，Fail-Safe 是安静的妥协，按需选择。

**本层留下的新账**：面试中如何选择容器？→ 引出生产决策（Level 8）。但在那之前，我们还需要认识 Queue/Deque 家族——Collection 框架的第三条线。

---

<a id="level-7-5"></a>

## Level 7.5：Queue/Deque 家族概览 —— 被忽视的第三条线

> ⚠️ 本层是**概览**，不是深挖。Queue/Deque + BlockingQueue + 线程池这条线足够撑起一篇独立文档，这里只画全景图和讲清"为什么你需要知道它"。

### 👶 前置知识关卡

- [ ] 什么是 FIFO（先进先出）和 LIFO（后进先出）？
- [ ] 什么是循环数组？
- [ ] 什么是生产者-消费者模型？

### Why——为什么需要 Queue/Deque

**徒弟**：ArrayList 不就能当队列用吗？`add()` 入队，`remove(0)` 出队。

**老陈**：`remove(0)` 是 O(n)，因为要移动所有元素。而且语义不对——队列的接口应该只有"入队"和"出队"，不应该暴露随机访问。

| 场景 | 错误选择 | 正确选择 | 原因 |
|------|---------|---------|------|
| 栈（LIFO） | `Stack`（extends Vector） | `ArrayDeque` | `Stack` 是遗留类，`synchronized` 浪费；`ArrayDeque` 性能更好 |
| 队列（FIFO） | `LinkedList` | `ArrayDeque` | `ArrayDeque` 循环数组，CPU 缓存友好，均摊 O(1) |
| 优先队列 | 自己写堆 | `PriorityQueue` | 标准库实现，`siftUp`/`siftDown` 经过充分测试 |
| 阻塞队列 | 手写 wait/notify | `BlockingQueue` 实现类 | 生产者-消费者模型的标准解，线程池内部就用它 |

### What——Queue/Deque 全景图

```
                    Iterable
                       │
                   Collection
                       │
            ┌──────────┴──────────┐
            │                     │
          Queue                  Deque（双端队列）
         /    \                /        \
  PriorityQueue  BlockingQueue  ArrayDeque  LinkedList
                   /  |  \        ↑           ↑
          Array    Linked  Synchronous  也实现了Deque
          Blocking  Blocking    Queue
          Queue     Queue
                    ↑
              DelayQueue
              PriorityBlockingQueue
              LinkedTransferQueue
```

> 📌 **最容易被误解的一点**：`LinkedList` 同时实现了 `List` 和 `Deque` 两个接口，但它做队列的性能不如 `ArrayDeque`（内存不连续，CPU 缓存不友好）。**官方推荐用 `ArrayDeque` 替代 `Stack` 和 `LinkedList` 做栈/队列**。

### How——四种 BlockingQueue 实现对比

```
┌──────────────────────┬──────────────┬──────────────┬───────────────────────┐
│ 实现类                │ 底层结构      │ 阻塞行为      │ 典型场景              │
├──────────────────────┼──────────────┼──────────────┼───────────────────────┤
│ ArrayBlockingQueue   │ 数组（有界）  │ 满时 put 阻塞 │ 生产者-消费者（固定容量）│
│                      │              │ 空时 take 阻塞│ ThreadPoolExecutor 默认│
├──────────────────────┼──────────────┼──────────────┼───────────────────────┤
│ LinkedBlockingQueue  │ 链表（可有界）│ 同上          │ 生产者-消费者（弹性容量）│
│                      │ 默认 Integer │              │ Executors.newFixed... │
│                      │ .MAX_VALUE   │              │                       │
├──────────────────────┼──────────────┼──────────────┼───────────────────────┤
│ SynchronousQueue     │ 无存储       │ put 等待 take │ 直接传递，不缓存       │
│                      │ （直接交接）  │ take 等待 put │ Executors.newCached.. │
├──────────────────────┼──────────────┼──────────────┼───────────────────────┤
│ DelayQueue           │ PriorityQueue│ 只有到期才能取 │ 定时任务、订单超时取消  │
│                      │              │              │                       │
└──────────────────────┴──────────────┴──────────────┴───────────────────────┘
```

**徒弟**：BlockingQueue 和线程池有什么关系？

**老陈**：**线程池 `ThreadPoolExecutor` 的核心就是 BlockingQueue**。当你调用 `executor.submit(task)` 时，任务被放入 BlockingQueue；工作线程从队列里 `take()` 任务执行。不同队列实现决定了线程池的行为：

```
ThreadPoolExecutor 的任务流向：
┌──────────┐    submit()    ┌──────────────────┐    take()     ┌──────────┐
│ 调用者    │ ──────────→  │ BlockingQueue     │ ←──────────  │ 工作线程  │
│          │              │ （任务缓冲区）      │              │          │
└──────────┘              └──────────────────┘              └──────────┘
                                ↑
                          队列满了 → 执行拒绝策略
                          
- FixedThreadPool  → LinkedBlockingQueue（无界，任务堆积可能 OOM）
- CachedThreadPool → SynchronousQueue（不缓存，线程不够就新建）
- ScheduledThreadPool → DelayedWorkQueue（定时任务）
```

> ⚠️ **面试高频考点**：`FixedThreadPool` 用的是无界 `LinkedBlockingQueue`，如果任务提交速度持续超过处理速度，队列会无限增长导致 **OOM**。生产环境建议用有界队列 + 自定义拒绝策略。

### 并发容器全景一览

| 容器 | 底层结构 | 线程安全机制 | 典型场景 |
|------|---------|-------------|---------|
| `ConcurrentHashMap` | 数组+链表+红黑树 | CAS + synchronized（桶级别） | 高并发 Map |
| `ConcurrentSkipListMap` | 跳表 | CAS（无锁） | 高并发有序 Map |
| `ConcurrentSkipListSet` | 跳表 | CAS（无锁） | 高并发有序 Set |
| `ConcurrentLinkedQueue` | 链表 | CAS（无锁） | 高并发 FIFO 队列 |
| `ConcurrentLinkedDeque` | 链表 | CAS（无锁） | 高并发双端队列 |
| `CopyOnWriteArrayList` | 数组 | 写时复制 | 读多写少 List |
| `CopyOnWriteArraySet` | 基于 COWArrayList | 写时复制 | 读多写少 Set |
| `ArrayBlockingQueue` | 数组 | ReentrantLock + Condition | 生产者-消费者 |
| `LinkedBlockingQueue` | 链表 | 两把锁（putLock/takeLock） | 生产者-消费者 |
| `SynchronousQueue` | 无存储 | CAS + LockSupport | 直接传递 |

> 📌 **`LinkedBlockingQueue` 用两把锁**（putLock 和 takeLock），put 和 take 可以并发执行，吞吐量比 `ArrayBlockingQueue` 的单锁更高。但 `ArrayBlockingQueue` 有界，不会 OOM，生产环境更安全。

### Transfer——迁移到其他设计问题

1. **用 `ArrayDeque` 替代 `Stack` 和 `LinkedList` 做栈/队列**：性能更好，是官方推荐
2. **线程池选型的本质是选 BlockingQueue**：无界队列危险，有界队列 + 合理拒绝策略是生产标配
3. **无锁队列（CAS）vs 锁队列**：CAS 在低竞争下更快，高竞争下可能退化（自旋浪费 CPU）
4. **BlockingQueue 的 `put()`/`take()` 会抛 `InterruptedException`**：必须处理，不要吞异常

> 🔴 **口诀**：ArrayDeque 是栈和队列的默认选择，BlockingQueue 是生产者-消费者的标准解，线程池的行为由队列决定。

**本层留下的新账**：Queue/Deque 讲完了，还有一些特殊用途的容器（并发有序 Map、枚举专用容器、不可变视图等）散落在各个角落，面试偶尔会问到。→ 引出扩展容器速查（Level 7.6）

---

<a id="level-7-6"></a>

## Level 7.6：扩展容器速查 —— 那些"不常见但问到就加分"的容器

> ⚠️ 本层是**速查手册**，不是六层深挖。每个容器按 **What → When → How → Key Internals → 面试一句话** 结构写，够用即可。

---

### 7.6.1 CopyOnWriteArrayList —— 读多写少的并发 List

**What**：写时复制（Copy-On-Write）的线程安全 List。每次 `add()`/`set()`/`remove()` 都复制整个底层数组，写入新副本后用 `volatile` 替换引用。遍历器拿到的是创建时的数组快照，永远不会抛 `ConcurrentModificationException`。

**When**：读多写少（读:写 ≥ 100:1），比如事件监听器列表、配置白名单、观察者模式的订阅者列表。

**How**：

```java
CopyOnWriteArrayList<EventListener> listeners = new CopyOnWriteArrayList<>();
listeners.add(listener1);  // 复制整个数组，O(n)
listeners.add(listener2);  // 再次复制，O(n)

// 遍历（零成本，不会复制）
for (EventListener l : listeners) {
    l.onEvent(event);  // 拿到的是快照，其他线程的 add/remove 不影响本次遍历
}
```

**Key Internals**：

```
内部结构：
┌─────────────────────────────────────────────────┐
│ CopyOnWriteArrayList                             │
│   lock = new ReentrantLock()                     │
│   array = volatile Object[]  ← 指向当前数组      │
└─────────────────────────────────────────────────┘

add() 过程：
① 加 lock
② 复制 array → 新数组（长度+1）
③ 新数组末尾放元素
④ array = 新数组（volatile 写，保证可见性）
⑤ 释放 lock

iterator() 过程：
① 记录当前 array 引用（快照）
② 遍历时只读这个快照，不受后续修改影响
```

> 📌 **最容易被误解的一点**：`CopyOnWriteArrayList` 的迭代器是**弱一致性**的，不支持 `remove()` 操作（调用会抛 `UnsupportedOperationException`）。要删除元素，必须直接调用 `list.remove()`。

**面试一句话**："CopyOnWriteArrayList 适合读多写少场景，写操作 O(n) 复制整个数组，遍历拿到的是快照不会 CME。"

---

### 7.6.2 CopyOnWriteArraySet —— 读多写少的并发 Set

**What**：基于 `CopyOnWriteArrayList` 实现的线程安全 Set。`add()` 时先遍历检查是否已存在，不存在才复制数组追加。

**When**：与 `CopyOnWriteArrayList` 相同场景，但需要去重。

```java
CopyOnWriteArraySet<String> tags = new CopyOnWriteArraySet<>();
tags.add("java");
tags.add("java");  // 已存在，不复制，直接返回 false
```

**Key Internals**：底层就是 `CopyOnWriteArrayList`，`add()` 内部调用 `list.addIfAbsent()`，先 O(n) 遍历检查，再 O(n) 复制。所以 `add()` 最坏是 **O(n)**。

**面试一句话**："CopyOnWriteArraySet 底层是 COWArrayList，add 最坏 O(n)，适合读多写少的小集合。"

---

### 7.6.3 EnumSet —— 枚举专用的位运算 Set

**What**：专为枚举类型设计的高性能 Set，底层用 **`long` 位向量**实现，每个枚举值对应一个 bit 位。

**When**：存储枚举值的集合，比如权限标记、状态机状态、星期几。

```java
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

// 创建方式
EnumSet<Day> weekdays = EnumSet.of(Day.MON, Day.TUE, Day.WED, Day.THU, Day.FRI);
EnumSet<Day> weekend = EnumSet.of(Day.SAT, Day.SUN);
EnumSet<Day> all = EnumSet.allOf(Day.class);
EnumSet<Day> range = EnumSet.range(Day.MON, Day.FRI);

// 集合运算（位运算，极快）
EnumSet<Day> union = EnumSet.copyOf(weekdays);
union.addAll(weekend);         // 并集
EnumSet<Day> intersection = EnumSet.copyOf(weekdays);
intersection.retainAll(weekend); // 交集
```

**Key Internals**：

```
EnumSet<Day> 存储结构（Day 有 7 个值）：

weekdays = EnumSet.of(MON, TUE, WED, THU, FRI)

位向量（long）：
bit:  6   5   4   3   2   1   0
      SUN SAT FRI THU WED TUE MON
      0   0   1   1   1   1   1   = 0b00011111 = 31

contains(Day.WED)  → (bits & (1L << 2)) != 0  → true   ← O(1)，位运算
add(Day.SAT)       → bits |= (1L << 5)                   ← O(1)，位运算
remove(Day.MON)    → bits &= ~(1L << 0)                  ← O(1)，位运算

枚举值超过 64 个时，自动切换到 long[] 数组（RegularEnumSet vs JumboEnumSet）
```

> 📌 **最容易被误解的一点**：`EnumSet` 不是线程安全的。需要并发访问时要外部加锁或用 `Collections.synchronizedSet()`。

**面试一句话**："EnumSet 底层是 long 位向量，add/contains/remove 都是 O(1) 位运算，枚举场景性能碾压 HashSet。"

---

### 7.6.4 EnumMap —— 枚举专用的高性能 Map

**What**：Key 必须是枚举类型的 Map，底层用 **枚举的 ordinal 作为数组索引**，直接寻址，不需要哈希。

**When**：Key 是枚举的场景，比如按星期统计、按状态聚合、按优先级分组。

```java
enum Status { PENDING, PROCESSING, SUCCESS, FAILED }

Map<Status, Integer> countMap = new EnumMap<>(Status.class);
countMap.put(Status.PENDING, 10);
countMap.put(Status.SUCCESS, 95);

// 遍历顺序 = 枚举声明顺序（ordinal 顺序）
for (Map.Entry<Status, Integer> entry : countMap.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
```

**Key Internals**：

```
EnumMap 底层结构：
┌────────────────────────────────────────────┐
│ EnumMap<Status, Integer>                   │
│   keyUniverse = Status.class.getEnumConstants()  │
│   vals = Object[4]   ← 数组长度 = 枚举值个数      │
└────────────────────────────────────────────┘

get(Status.SUCCESS)  → vals[2]           ← ordinal=2，直接数组索引，O(1)
put(Status.FAILED, 1) → vals[3] = 1      ← ordinal=3，直接数组索引，O(1)

对比 HashMap：
- HashMap：key.hashCode() → 扰动函数 → & (n-1) → 桶索引 → 可能冲突 → 链表/红黑树
- EnumMap：key.ordinal() → 直接数组索引，零冲突，零开销
```

> 📌 **与 HashMap 的关键区别**：`EnumMap` 不允许 null key（枚举值不可能是 null），但允许 null value。遍历顺序是枚举声明顺序，不是插入顺序。

**面试一句话**："EnumMap 用 ordinal 做数组索引，零哈希零冲突，Key 是枚举时性能碾压 HashMap。"

---

### 7.6.5 IdentityHashMap —— 用 `==` 而不是 `equals()` 比较

**What**：用 **引用相等**（`==`）而不是 `equals()` 来比较 key 的 Map。底层是**开放寻址法**（数组 + 线性探测），不是链地址法。

**When**：极少使用。典型场景：
- **对象图遍历时检测循环引用**（深拷贝/序列化框架）
- 需要区分 `new String("a")` 和 `new String("a")` 这两个不同对象

```java
// 深拷贝防循环引用的典型用法
Map<Object, Object> visited = new IdentityHashMap<>();
public Object deepCopy(Object obj) {
    if (visited.containsKey(obj)) {  // 用 == 判断是否已经拷贝过
        return visited.get(obj);      // 返回已拷贝的副本
    }
    Object copy = doCopy(obj);
    visited.put(obj, copy);
    // ...递归拷贝子对象
    return copy;
}
```

**Key Internals**：

```
IdentityHashMap 底层：开放寻址法（线性探测）

table = Object[2 * capacity]  ← key 和 value 交替存放
table[0] = key0, table[1] = value0, table[2] = key1, table[3] = value1, ...

hash(key) = System.identityHashCode(key)  ← 直接用 JVM 的 identity hash
寻址：i = hash & (table.length - 1)
冲突：线性探测，i = (i + 2) % table.length

对比 HashMap：
- HashMap：key.equals() + key.hashCode() → 链地址法
- IdentityHashMap：key == otherKey + System.identityHashCode() → 开放寻址法
```

> ⚠️ **极度危险**：`IdentityHashMap` 违反了 `Map` 接口的通用契约（应该用 `equals` 比较）。只在**非常确定需要引用相等**的场景使用，误用会导致"明明是同一个字符串但 get 返回 null"的诡异 bug。

**面试一句话**："IdentityHashMap 用 == 比较 key，底层开放寻址法，典型用途是深拷贝时检测循环引用。"

---

### 7.6.6 ConcurrentSkipListMap —— 无锁并发有序 Map

**What**：基于 **跳表（Skip List）** 实现的并发有序 Map，所有操作都是 **CAS 无锁**，支持高并发的 `O(log n)` 插入/查找/删除。

**When**：需要**并发 + 有序**的场景。替代 `Collections.synchronizedSortedMap(new TreeMap<>())`。

```java
ConcurrentSkipListMap<String, Session> map = new ConcurrentSkipListMap<>();
map.put("user:100", session1);
map.put("user:200", session2);
map.put("user:150", session3);

// 有序遍历（按 key 排序）
for (Map.Entry<String, Session> entry : map.entrySet()) {
    // user:100 → user:150 → user:200
}

// NavigableMap 接口方法
map.firstKey();           // "user:100"
map.lastKey();            // "user:200"
map.floorKey("user:160"); // "user:150"（≤160 的最大 key）
map.ceilingKey("user:110"); // "user:150"（≥110 的最小 key）
map.headMap("user:150");  // {user:100=...}
map.tailMap("user:150");  // {user:150=..., user:200=...}
```

**Key Internals —— 跳表结构**：

```
跳表（Skip List）= 多层有序链表 + 随机化索引

Level 3:  head ─────────────────────────────────────→ [200] → null
Level 2:  head ──────────────→ [150] ───────────────→ [200] → null
Level 1:  head ───→ [100] ──→ [150] ───→ [170] ────→ [200] → null
Level 0:  head ──→ [100] ──→ [130] ──→ [150] ──→ [170] ──→ [200] → null
                           ↑
                      数据层（完整有序链表）

查找 150：
① 从 Level 3 的 head 开始
② Level 3: head → 200（超过 150，下降）
③ Level 2: head → 150（找到！）
查找路径：head → 200(跳过) → 150(命中)，只访问 2 个节点
对比链表：需要遍历 head → 100 → 130 → 150，访问 4 个节点

跳表 vs 红黑树：
┌──────────────┬──────────────┬────────────────────────┐
│              │ 红黑树       │ 跳表                    │
├──────────────┼──────────────┼────────────────────────┤
│ 查找         │ O(log n)     │ O(log n)               │
│ 插入/删除    │ O(log n)，旋转│ O(log n)，CAS 无锁       │
│ 实现复杂度   │ 高（旋转+染色）│ 低（随机化+指针调整）       │
│ 并发友好     │ 差（旋转影响多节点）│ 好（局部修改）         │
│ 有序遍历     │ 中序遍历     │ Level 0 链表直接遍历       │
│ 空间         │ O(n)         │ O(n)（常数因子更大，约 2 倍指针开销）  │
└──────────────┴──────────────┴────────────────────────┘

为什么并发场景选跳表而不是红黑树？
- 红黑树旋转时需要修改多个节点的指针，并发下需要锁住整棵子树
- 跳表插入时只需要修改前后几个节点的指针，CAS 局部修改即可
- 跳表的随机化层级天然适合无锁编程
```

> 📌 **最容易被误解的一点**：`ConcurrentSkipListMap` 不是"并发版 TreeMap"，它底层完全不是红黑树，是跳表。但它实现了 `ConcurrentNavigableMap` 接口，API 和 `TreeMap` 几乎一样。

**面试一句话**："ConcurrentSkipListMap 是 CAS 无锁的并发有序 Map，底层跳表，替代 synchronized TreeMap。"

---

### 7.6.7 NavigableMap / NavigableSet 接口 —— SortedMap 的增强版

**徒弟**：TreeMap 不是已经有 `Comparator` 排序了吗？`SortedMap` 接口也有 `firstKey()`、`headMap()` 这些方法，`NavigableMap` 是干嘛的？

**老陈**：先理清接口继承关系：

```
Map（无序，JDK 1.0）
  │
  └── SortedMap（有序，JDK 1.2）
        │  提供：comparator()、firstKey()、lastKey()
        │        headMap(toKey)、tailMap(fromKey)、subMap(from, to)
        │
        └── NavigableMap（有序 + 近邻查询，JDK 6+）
              │  新增：floorKey()、ceilingKey()、lowerKey()、higherKey()
              │        floorEntry()、ceilingEntry()、pollFirstEntry()、pollLastEntry()
              │        descendingMap()、navigableKeySet()
              │
              ├── TreeMap         ← 实现了 NavigableMap（底层红黑树）
              └── ConcurrentSkipListMap ← 实现了 NavigableMap（底层跳表）
```

**三者的关系**：

| 层级 | 它负责什么 | TreeMap 中对应的字段/方法 |
|------|-----------|-------------------------|
| **Comparator** | 排序规则（怎么比大小） | `private final Comparator<? super K> comparator;` |
| **SortedMap** | 基本有序操作（首尾 key、范围视图） | `firstKey()`、`headMap()`、`subMap()` |
| **NavigableMap** | 近邻查询（找"离我最近的"） | `floorKey()`、`ceilingKey()`、`lowerKey()`、`higherKey()` |

> 📌 **一句话总结**：`Comparator` 决定"怎么排"，`SortedMap` 提供"基本有序操作"，`NavigableMap` 在此基础上增加了"近邻查询"能力。`TreeMap` 实现了 `NavigableMap`，所以三者的方法**都能用**。

**When**：需要"找离某个 key 最近的元素"的场景——限流窗口、区间查询、排行榜。这些能力是 `SortedMap` 接口**没有的**，只有 `NavigableMap` 才提供。

```java
TreeMap<Integer, String> scoreboard = new TreeMap<>();
scoreboard.put(1, "Alice");
scoreboard.put(5, "Bob");
scoreboard.put(10, "Charlie");
scoreboard.put(15, "David");

// ─── SortedMap 就有的方法（JDK 1.2）───
scoreboard.firstKey();                           // 1
scoreboard.lastKey();                            // 15
scoreboard.headMap(10);                          // {1=A, 5=B}      ← key < 10
scoreboard.tailMap(5);                           // {5=B, 10=C, 15=D} ← key ≥ 5
scoreboard.subMap(5, 10);                        // {5=B}            ← 5 ≤ key < 10
scoreboard.comparator();                         // null（自然排序）

// ─── NavigableMap 新增的方法（JDK 6+）───
scoreboard.floorKey(7);      // 5    ← ≤7 的最大 key
scoreboard.ceilingKey(7);    // 10   ← ≥7 的最小 key
scoreboard.lowerKey(10);     // 5    ← <10 的最大 key（不含 10）
scoreboard.higherKey(10);    // 15   ← >10 的最小 key（不含 10）
scoreboard.pollFirstEntry(); // {1=A} ← 取出并移除最小 entry
scoreboard.descendingMap();  // {15=D, 10=C, 5=B}  ← 逆序视图
```

**典型生产场景——滑动窗口限流**：

```java
// 基于 NavigableMap 的滑动窗口限流
public class SlidingWindowRateLimiter {
    private final NavigableMap<Long, Integer> window = new TreeMap<>();
    private final int maxRequests;
    private final long windowSizeMs;

    public boolean allowRequest() {
        long now = System.currentTimeMillis();
        // 清除窗口外的旧数据（SortedMap 的 headMap）
        window.headMap(now - windowSizeMs).clear();
        // 统计当前窗口内的请求数
        int count = window.values().stream().mapToInt(Integer::intValue).sum();
        if (count >= maxRequests) {
            return false;
        }
        window.put(now, 1);
        return true;
    }
}
```

> 📌 **`floor` vs `lower` 的区别**：`floorKey(10)` 返回 ≤ 10 的最大 key（**包含 10 本身**），`lowerKey(10)` 返回 < 10 的最大 key（**不包含 10**）。`ceiling` vs `higher` 同理。这是面试高频坑。

**面试一句话**："SortedMap 提供基本有序操作，NavigableMap 在此基础上增加了 floor/ceiling 近邻查询。TreeMap 实现了 NavigableMap，所以都能用。限流窗口、区间查询场景直接调 floorKey/ceilingKey。"

---

### 速查对比总表

| 容器 | 底层结构 | 线程安全 | 时间复杂度 | 典型场景 |
|------|---------|---------|-----------|---------|
| `CopyOnWriteArrayList` | 数组（写时复制） | ✅ | 读 O(1)，写 O(n) | 事件监听器、配置白名单 |
| `CopyOnWriteArraySet` | COWArrayList | ✅ | 读 O(1)，写 O(n) | 读多写少的小集合 |
| `EnumSet` | long 位向量 | ❌ | O(1) 位运算 | 枚举值集合、权限标记 |
| `EnumMap` | 枚举 ordinal 索引数组 | ❌ | O(1) 直接寻址 | Key 是枚举的 Map |
| `IdentityHashMap` | 开放寻址法 | ❌ | O(1) 均摊 | 深拷贝循环引用检测 |
| `ConcurrentSkipListMap` | 跳表 | ✅（CAS） | O(log n) | 并发有序 Map |
| `ConcurrentSkipListSet` | 跳表 | ✅（CAS） | O(log n) | 并发有序 Set |
| `NavigableMap`（接口） | — | 取决于实现 | — | 近邻查询、区间查询 |

---

### 📌 null 允许情况对比（面试高频坑）

> 记忆口诀：**并发容器一律禁 null，排序容器禁 null key，其余基本都放行**

| 容器 | null key / null 元素 | null value | 原因 |
|------|---------------------|------------|------|
| **List 家族** | | | |
| `ArrayList` | ✅ | — | 底层数组，null 是合法元素 |
| `LinkedList` | ✅ | — | 底层链表，null 是合法元素 |
| `Vector` | ✅ | — | 同 ArrayList |
| `CopyOnWriteArrayList` | ✅ | — | 同 ArrayList |
| **Set 家族** | | | |
| `HashSet` | ✅ | — | 底层 HashMap，null 有专门桶（hash=0） |
| `TreeSet` | ❌ **NPE** | — | 底层 TreeMap，null 无法比较大小 |
| `LinkedHashSet` | ✅ | — | 底层 LinkedHashMap |
| `EnumSet` | ❌ **NPE** | — | 枚举值不可能是 null |
| `CopyOnWriteArraySet` | ✅ | — | 底层 COWArrayList |
| **Map 家族** | | | |
| `HashMap` | ✅ | ✅ | null key 有专门桶（hash=0），null value 用常量占位 |
| `TreeMap` | ❌ **NPE** | ✅ | comparator.compare(null, x) 会抛 NPE |
| `LinkedHashMap` | ✅ | ✅ | 继承 HashMap |
| `ConcurrentHashMap` | ❌ **NPE** | ❌ **NPE** | 设计决策：并发下 null 语义模糊（是 key 不存在还是 value 就是 null？） |
| `ConcurrentSkipListMap` | ❌ **NPE** | ❌ **NPE** | 底层跳表需要比较，null 无法比较 |
| `EnumMap` | ❌ **NPE** | ✅ | 枚举值不可能是 null |
| `IdentityHashMap` | ✅ | ✅ | 底层开放寻址法，null key hash=0 |
| `WeakHashMap` | ✅ | ✅ | 同 HashMap |
| **Queue/Deque 家族** | | | |
| `ArrayDeque` | ❌ **NPE** | — | 源码显式 `if (e == null) throw new NPE()` |
| `PriorityQueue` | ❌ **NPE** | — | null 无法比较优先级 |
| `LinkedList`（作 Deque） | ✅ | — | 底层链表允许 null |
| `ArrayBlockingQueue` | ❌ **NPE** | — | 源码显式检查 null |
| `LinkedBlockingQueue` | ❌ **NPE** | — | 源码显式检查 null |
| `SynchronousQueue` | ❌ **NPE** | — | 源码显式检查 null |

> 📌 **面试高频追问**："为什么 ConcurrentHashMap 不允许 null key？"
> 答：`HashMap` 允许 null key 是因为单线程下，`get()` 返回 null 可以用 `containsKey()` 区分"key 不存在"还是"value 就是 null"。但 `ConcurrentHashMap` 在并发环境下，`containsKey()` 和 `get()` 之间可能被其他线程修改，无法可靠区分，所以**设计上直接禁掉 null**，避免歧义。

---

### 📌 排序/有序性对比

| 容器 | 是否有序 | 有序类型 | 排序依据 | 遍历顺序 |
|------|---------|---------|---------|---------|
| **List 家族** | | | | |
| `ArrayList` | ✅ | 插入顺序 | — | 按索引 0→n |
| `LinkedList` | ✅ | 插入顺序 | — | 按节点链 0→n |
| `CopyOnWriteArrayList` | ✅ | 插入顺序 | — | 按索引 0→n |
| **Set 家族** | | | | |
| `HashSet` | ❌ | 无序 | — | 取决于哈希值，不可预测 |
| `TreeSet` | ✅ | **排序** | `Comparable` 或 `Comparator` | 中序遍历（升序） |
| `LinkedHashSet` | ✅ | 插入顺序 | — | 按插入先后 |
| `EnumSet` | ✅ | **ordinal 顺序** | 枚举声明顺序 | ordinal 0→n |
| `CopyOnWriteArraySet` | ✅ | 插入顺序 | — | 按插入先后 |
| **Map 家族** | | | | |
| `HashMap` | ❌ | 无序 | — | 取决于哈希值，不可预测 |
| `TreeMap` | ✅ | **排序** | `Comparable` 或 `Comparator` | 中序遍历（升序） |
| `LinkedHashMap` | ✅ | 插入顺序 **或** 访问顺序 | `accessOrder` 参数 | false=插入顺序，true=访问顺序（LRU） |
| `ConcurrentHashMap` | ❌ | 无序 | — | 取决于哈希值，不可预测 |
| `ConcurrentSkipListMap` | ✅ | **排序** | `Comparable` 或 `Comparator` | 跳表 Level 0 链表（升序） |
| `EnumMap` | ✅ | **ordinal 顺序** | 枚举声明顺序 | ordinal 0→n |
| `IdentityHashMap` | ❌ | 无序 | — | 取决于 identityHashCode |
| `WeakHashMap` | ❌ | 无序 | — | 取决于哈希值 |
| **Queue/Deque 家族** | | | | |
| `ArrayDeque` | ✅ | FIFO/LIFO | — | 按入队顺序 |
| `PriorityQueue` | ✅ | **优先级排序** | `Comparable` 或 `Comparator` | 堆顶优先（不保证全序） |
| `ConcurrentSkipListSet` | ✅ | **排序** | `Comparable` 或 `Comparator` | 升序 |

> 📌 **`PriorityQueue` 的遍历不保证全序**：`iterator()` 返回的顺序**不是优先级顺序**，只是底层堆数组的顺序。要按优先级取出，必须循环调用 `poll()`。

---

**本层留下的新账**：所有容器都认识了，面试和生产中更重要的是"面对业务场景怎么选"→ 引出生产决策（Level 8）

---

<a id="level-8"></a>

## Level 8：生产决策 —— 如何在架构评审中选型

### 👶 前置知识关卡

- [ ] ArrayList 和 LinkedList 的真实差异？
- [ ] HashMap 和 TreeMap 的适用场景？
- [ ] ConcurrentHashMap 和 Collections.synchronizedMap 的性能差异？

### Why——上一代方案死在哪

**徒弟**：面试时问我"你会怎么选择容器？"，我应该从哪些维度回答？

**老陈**：**不能只说"ArrayList 最常用"**。架构评审要的是**决策矩阵**：什么场景、选什么、为什么、不能做什么。

### What——选型决策矩阵

| 场景 | 推荐 | 不推荐 | 原因 |
|------|------|--------|------|
| **单线程，随机访问** | `ArrayList` | `LinkedList` | CPU 缓存友好，O(1) 随机访问 |
| **单线程，频繁头部插入** | `LinkedList` | `ArrayList` | O(1) 头插，但要确认不会随机访问 |
| **单线程，Key-Value 查找** | `HashMap` | `TreeMap` | O(1) 查找，无序 |
| **单线程，Key-Value 有序** | `TreeMap` | `HashMap` | O(log n) 查找，有序 |
| **多线程，并发读写 Map** | `ConcurrentHashMap` | `Collections.synchronizedMap` | 桶级别锁，高并发 |
| **多线程，并发读写 List** | `CopyOnWriteArrayList` | `Collections.synchronizedList` | 读多写少场景 |
| **多线程，阻塞队列** | `LinkedBlockingQueue` | `Collections.synchronizedList` | 生产者-消费者模式 |
| **需要去重** | `HashSet` | `ArrayList` | O(1) 去重 |
| **需要去重且有序** | `TreeSet` | `HashSet` | O(log n) 去重，有序 |

### How——主线角色的完整决策流程

```
场景：设计一个用户会话缓存，要求：
1. 高并发读写
2. Key 是 userId（String）
3. Value 是 Session 对象
4. 需要支持按登录时间排序查询

决策过程：
① 需要 Key-Value → Map 体系
② 高并发 → ConcurrentHashMap 或 Collections.synchronizedMap
③ 需要排序 → TreeMap 或 LinkedHashMap
④ 综合：ConcurrentHashMap + 定期排序

方案对比：
┌─────────────────────────────────────────────────────────────┐
│ 方案 A：ConcurrentHashMap + 定期排序                        │
├─────────────────────────────────────────────────────────────┤
│ 优点：高并发读写 O(1)                                       │
│ 缺点：排序需要额外 O(n log n)，有延迟                        │
│ 适用：读多写少，排序实时性要求不高                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 方案 B：ConcurrentSkipListMap（JDK 6+）                     │
├─────────────────────────────────────────────────────────────┤
│ 优点：并发有序 Map，O(log n) 插入/查找/删除                  │
│ 缺点：空间开销大（跳表），O(log n) 比 O(1) 慢                │
│ 适用：需要并发且有序，对性能要求不是极致                     │
└─────────────────────────────────────────────────────────────┘

决策：方案 A（大多数场景足够） 或 方案 B（实时排序需求）
```

### Transfer——迁移到其他设计问题

1. **初始容量估算**：`new HashMap<>(expectedSize / 0.75)` 避免多次扩容
2. **Key 的不可变性**：String、Integer 等不可变类型是最佳 Key
3. **避免内存泄漏**：`WeakHashMap` 的 Key 是弱引用，GC 时自动清理

> 🔴 **口诀**：选容器不是背 API，是做架构决策：场景 → 需求 → 权衡 → 选择。

---

<a id="self-test"></a>

## 🧪 合书自测

### 时序图：HashMap.put() 的完整旅程

```
T0: 调用 map.put("user:123", session)
T1: 计算 hash("user:123") = 98765
T2: 索引 = 98765 & (16-1) = 5
T3: 检查 table[5] == null？
    ├── 是 → CAS 插入新节点，结束
    └── 否 → 进入 T4
T4: table[5] 是链表还是红黑树？
    ├── 链表 → 遍历，找相同 key 或追加到尾部
    │   └── 链表长度 > 8 且容量 ≥ 64 → 树化
    └── 红黑树 → 按红黑树规则插入
T5: 覆盖旧值（如果 key 已存在）
T6: size++，检查是否需要扩容
    └── size > threshold → resize()
        ├── 新建 2 倍大小的数组
        └── 遍历旧数组，重新分配位置
T7: 返回旧值（如果 key 已存在）或 null
```

### 自测题

| # | 题目 | 必须答出的不变量 |
|---|------|-----------------|
| T1 | HashMap 的容量为什么必须是 2 的幂次？ | `hash & (n-1)` 等价于 `hash % n`，位运算更快 |
| T2 | HashMap 在什么条件下会树化？ | 链表长度 > 8 **且** 容量 ≥ 64 |
| T3 | ConcurrentHashMap 如何实现无锁插入？ | 桶为空时用 CAS，桶非空时用 synchronized 锁住桶头 |
| T4 | 为什么 JDK 8 要把 HashMap 改成尾插法？ | 解决 JDK 7 头插法在多线程扩容时导致的链表成环问题 |
| T5 | CopyOnWriteArrayList 适合什么场景？ | 读多写少，写操作代价高（复制整个数组） |
| T6 | 为什么 HashMap 的负载因子是 0.75？ | 数学推导，空间和时间的平衡点 |
| T7 | 为什么选红黑树而不是 AVL 树？ | AVL 删除最坏 O(log n) 次旋转，红黑树最多 3 次；实现复杂度更低 |
| T8 | ConcurrentHashMap 的 size() 返回精确值吗？ | 返回估计值，不是精确值 |
| T9 | 为什么 LinkedList 在大多数场景比 ArrayList 慢？ | CPU 缓存不友好，额外 24 字节指针开销 |
| T10 | 如何在 for-each 循环中安全删除元素？ | 使用 Iterator.remove() 或 removeIf() |
| T11 | LinkedHashMap 如何实现 LRU？ | accessOrder=true + 重写 removeEldestEntry |
| T12 | 为什么用 ArrayDeque 替代 Stack？ | Stack 是遗留类，synchronized 浪费；ArrayDeque 循环数组，CPU 缓存友好 |
| T13 | ConcurrentHashMap 扩容时其他线程在做什么？ | 帮助搬迁（helpTransfer），多线程并行扩容 |
| T14 | 红黑树插入/删除最坏几次旋转？ | 插入最坏 2 次，删除最坏 3 次；AVL 删除才是 O(log n) |
| T15 | FixedThreadPool 有什么隐患？ | 无界 LinkedBlockingQueue，任务堆积可能 OOM |
| T16 | EnumSet 底层用什么实现？ | long 位向量，add/contains/remove 都是 O(1) 位运算 |
| T17 | IdentityHashMap 用什么比较 key？ | ==（引用相等），不是 equals()，底层开放寻址法 |
| T18 | ConcurrentSkipListMap 底层是什么？ | 跳表（不是红黑树），CAS 无锁，O(log n) |
| T19 | NavigableMap 的 floorKey 和 lowerKey 区别？ | floorKey 包含自身，lowerKey 不包含 |
| T20 | EnumMap 为什么比 HashMap 快？ | 用 ordinal 做数组索引，零哈希零冲突，O(1) 直接寻址 |

---

<a id="pitfalls"></a>

## ⚠️ 坑与细节

### 坑 1：HashMap 的 key 没有正确重写 hashCode() 和 equals()

```java
// 错误代码
class User {
    String id;
    // 没有重写 hashCode() 和 equals()
}

Map<User, String> map = new HashMap<>();
User u1 = new User("123");
map.put(u1, "session1");

User u2 = new User("123");
map.get(u2); // ❌ 返回 null！因为 u1 和 u2 是不同对象，hashCode 不同

// 正确代码
class User {
    String id;
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(id, user.id);
    }
}
```

**错因**：没有重写 `hashCode()` 和 `equals()`，默认使用 Object 的实现（内存地址）。

**比喻里的后果**：同一本书用不同的借书证借，图书馆认为是两本书。

**线上现象**：`map.put()` 成功，但 `map.get()` 返回 null，数据"丢失"。

**修正**：Key 必须正确重写 `hashCode()` 和 `equals()`，或者使用不可变类型（String、Integer）。

---

### 坑 2：在 for-each 循环中直接调用 list.remove()

```java
// 错误代码
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
for (String s : list) {
    if ("B".equals(s)) {
        list.remove(s); // ❌ ConcurrentModificationException
    }
}

// 正确代码
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if ("B".equals(s)) {
        it.remove(); // ✅
    }
}

// 或者使用 JDK 8+ 的 removeIf
list.removeIf("B"::equals); // ✅
```

**错因**：`list.remove()` 修改了 `modCount`，但 Iterator 的 `expectedModCount` 没有同步。

**比喻里的后果**：你在图书馆删了一本书，但借阅登记本上还记着，下次查的时候发现不一致。

**线上现象**：遍历时删除元素抛出 `ConcurrentModificationException`。

**修正**：使用 `Iterator.remove()` 或 `removeIf()`。

---

### 坑 3：HashMap 在多线程环境下使用

```java
// 错误代码
Map<String, String> map = new HashMap<>();
// 线程 A
map.put("key1", "value1");
// 线程 B
map.put("key2", "value2");
// 可能导致数据丢失、死循环（JDK 7）、覆盖

// 正确代码
Map<String, String> map = new ConcurrentHashMap<>();
```

**错因**：HashMap 不是线程安全的，多线程并发修改会导致不可预期的行为。

**比喻里的后果**：两个管理员同时在同一个书架上放书，书架乱了。

**线上现象**：数据丢失、死循环（JDK 7）、CPU 飙高。

**修正**：使用 `ConcurrentHashMap` 或外部加锁。

---

### 坑 4：TreeMap 的 key 没有实现 Comparable

```java
// 错误代码
Map<Object, String> map = new TreeMap<>();
map.put(new Object(), "value"); // ❌ ClassCastException

// 正确代码
Map<String, String> map = new TreeMap<>(); // String 实现了 Comparable
// 或者传入 Comparator
Map<User, String> map = new TreeMap<>(Comparator.comparing(User::getId));
```

**错因**：TreeMap 需要比较 key 的大小，如果没有实现 `Comparable` 或传入 `Comparator`，无法比较。

**比喻里的后果**：图书馆的书没有编号，管理员不知道怎么排序。

**线上现象**：`put()` 时抛出 `ClassCastException`。

**修正**：Key 实现 `Comparable` 或在构造时传入 `Comparator`。

---

### 坑 5：ArrayList 的 subList() 是视图不是副本

```java
// 错误代码
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C", "D", "E"));
List<String> sub = list.subList(1, 3); // ["B", "C"]
sub.clear(); // ❌ 原 list 也变了！list = ["A", "D", "E"]

// 正确代码
List<String> sub = new ArrayList<>(list.subList(1, 3)); // 创建副本
```

**错因**：`subList()` 返回的是原 list 的视图，修改会影响原 list。

**比喻里的后果**：你借了图书馆的一个书架分区的书，结果把整个书架的书都删了。

**线上现象**：修改 subList 后，原 list 数据丢失。

**修正**：`new ArrayList<>(list.subList(...))` 创建副本。

---

### 坑 6：HashMap 的初始容量设置不合理

```java
// 错误代码
Map<String, String> map = new HashMap<>(1000); // 预期 1000 个元素
// 实际存储 1000 个元素时，会触发多次扩容（750, 1125, ...）

// 正确代码
Map<String, String> map = new HashMap<>((int) (1000 / 0.75) + 1); // 1334
// 或者使用 Guava
Map<String, String> map = Maps.newHashMapWithExpectedSize(1000);
```

**错因**：初始容量是桶数量，不是元素数量。阈值 = 容量 × 负载因子。

**比喻里的后果**：图书馆建了 1000 个书架，但只能放 750 本书，剩下的要临时扩建。

**线上现象**：频繁扩容，性能下降。

**修正**：`initialCapacity = expectedSize / 0.75 + 1`。

---

### 坑 7：ConcurrentHashMap 的复合操作不是原子的

```java
// 错误代码
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
if (!map.containsKey("key")) {
    map.put("key", "value"); // ❌ 竞态条件：两个线程可能都通过 containsKey 检查
}

// 正确代码
map.putIfAbsent("key", "value"); // ✅ 原子操作

// 或者使用 compute()
map.compute("key", (k, v) -> v == null ? "value" : v); // ✅
```

**错因**：`containsKey()` 和 `put()` 是两个独立操作，中间可能被其他线程插入。

**比喻里的后果**：两个管理员都检查到某个书架没有某本书，都去放，导致重复。

**线上现象**：数据重复或覆盖。

**修正**：使用 `putIfAbsent()`、`compute()`、`merge()` 等原子操作。

---

### 坑 8：HashSet 的去重依赖 hashCode() 和 equals()

```java
// 错误代码
Set<User> set = new HashSet<>();
User u1 = new User("123");
set.add(u1);

User u2 = new User("123");
set.contains(u2); // ❌ 返回 false！因为没有重写 hashCode() 和 equals()

// 正确代码
class User {
    String id;
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(id, user.id);
    }
}
```

**错因**：HashSet 底层是 HashMap，去重依赖 `hashCode()` 和 `equals()`。

**比喻里的后果**：图书馆认为同一本书的不同副本是不同的书。

**线上现象**：Set 中出现"重复"元素。

**修正**：Key 必须正确重写 `hashCode()` 和 `equals()`。

---

### 坑 9：CopyOnWriteArrayList 的写操作代价高

```java
// 错误代码
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
for (int i = 0; i < 100000; i++) {
    list.add("item" + i); // ❌ 每次 add 都复制整个数组，O(n)
}

// 正确代码：读多写少场景才用 CopyOnWriteArrayList
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
// 初始化时批量添加
List<String> initData = new ArrayList<>();
for (int i = 0; i < 100000; i++) {
    initData.add("item" + i);
}
list.addAll(initData);
// 之后主要是读操作
for (String s : list) {
    // 读操作，不会复制
}
```

**错因**：CopyOnWriteArrayList 的写操作会复制整个数组，写密集场景性能极差。

**比喻里的后果**：图书馆每次加一本书都要重新抄写整个目录。

**线上现象**：写操作延迟高，内存占用飙升。

**修正**：只在读多写少场景使用 CopyOnWriteArrayList。

---

### 坑 10：HashMap 的 key 使用可变对象

```java
// 错误代码
List<String> key = new ArrayList<>(Arrays.asList("A", "B"));
Map<List<String>, String> map = new HashMap<>();
map.put(key, "value");

key.add("C"); // ❌ 修改了 key，hashCode 变了
map.get(key); // ❌ 返回 null！因为桶位置变了

// 正确代码：使用不可变对象作为 key
String key = "A-B-C";
Map<String, String> map = new HashMap<>();
map.put(key, "value");
```

**错因**：HashMap 的 key 在放入后如果被修改，会导致 hashCode 变化，无法找到原来的桶。

**比喻里的后果**：你把书放在编号 100 的书架上，然后把编号改成了 200，下次找书时去 200 号书架找，找不到。

**线上现象**：`put()` 成功，但 `get()` 返回 null，数据"丢失"。

**修正**：使用不可变对象作为 key（String、Integer、自定义不可变类）。

---

### 坑 11：LinkedHashMap 的 accessOrder 默认是 false

```java
// 错误代码：期望实现 LRU，但 accessOrder 没设为 true
Map<String, Session> cache = new LinkedHashMap<>(1000, 0.75f, false); // ❌ false 是默认值
cache.put("A", session1);
cache.put("B", session2);
cache.get("A"); // 访问 A，但 A 不会移到尾部！因为 accessOrder=false

// 正确代码
Map<String, Session> cache = new LinkedHashMap<>(1000, 0.75f, true); // ✅ accessOrder=true
```

**错因**：`accessOrder=false`（默认）只维护**插入顺序**，`accessOrder=true` 才维护**访问顺序**（LRU 语义）。

**比喻里的后果**：图书馆的"最近借阅"列表，实际上记录的是"入库顺序"，每次查书不会更新位置。

**线上现象**：LRU 缓存不淘汰最久未访问的 key，而是淘汰最早插入的 key，缓存命中率远低于预期。

**修正**：构造 `LinkedHashMap` 时显式传入 `accessOrder=true`。

---

### 坑 12：FixedThreadPool 使用无界队列导致 OOM

```java
// 错误代码
ExecutorService executor = Executors.newFixedThreadPool(10); // 内部用 LinkedBlockingQueue（无界）
// 如果任务提交速度持续超过处理速度，队列无限增长 → OOM

// 正确代码
ExecutorService executor = new ThreadPoolExecutor(
    10, 10, 0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>(1000), // 有界队列
    new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略：调用者线程执行
);
```

**错因**：`FixedThreadPool` 内部使用无界 `LinkedBlockingQueue`，任务堆积没有上限。

**比喻里的后果**：图书馆的待归还书架无限增长，最终仓库爆满，整个图书馆系统崩溃。

**线上现象**：服务运行一段时间后 OOM，堆 dump 中看到大量待执行任务对象。

**修正**：使用 `ThreadPoolExecutor` 显式构造，设置有界队列 + 合理拒绝策略。

---

<a id="timeline"></a>

## 📊 竖切总表 T0–T8

| 阶段 | 位置 | 关键动作 | 不变量 | 典型坑 |
|------|------|---------|--------|--------|
| T0 | 容器选型 | 确定用 List/Set/Map | 根据业务需求选择 | 选错容器类型 |
| T1 | 具体实现 | 选择 ArrayList/LinkedList/HashMap 等 | 根据性能需求选择 | 凭直觉选 LinkedList |
| T2 | 初始化 | 设置初始容量和负载因子 | `capacity = expectedSize / 0.75 + 1` | 初始容量设置不合理 |
| T3 | 插入元素 | `add()` / `put()` | HashMap 需要正确重写 hashCode/equals | key 没有正确重写 |
| T4 | 查找元素 | `get()` / `contains()` | HashMap O(1)，TreeMap O(log n) | 用错容器导致 O(n) |
| T5 | 删除元素 | `remove()` | 使用 Iterator.remove() 或 removeIf() | for-each 中直接 remove |
| T6 | 遍历 | `for-each` / `Iterator` | Fail-Fast/Fail-Safe 的选择 | 并发遍历时修改 
| T7 | 扩容 | 自动扩容 | HashMap 2倍，ArrayList 1.5倍 | 频繁扩容导致性能下降
| T8 | 并发访问 | 多线程读写 | 使用 ConcurrentHashMap / CopyOnWriteArrayList | HashMap 多线程使用 |

---

<a id="errata"></a>

## 📚 版本勘误表

| ❌ 常见说法 | ✅ 更准确的说法 |
|------------|----------------|
| "ArrayList 比 LinkedList 快" | ArrayList 在**大多数场景**更快，因为 CPU 缓存友好和 O(1) 随机访问。LinkedList 在**头部频繁插入且从不随机访问**的场景可能更快 |
| "HashMap 是无序的" | HashMap 是**按哈希值存储**的，不是完全随机的。遍历顺序是确定的，但不保证与插入顺序一致 |
| "ConcurrentHashMap 是线程安全的" | ConcurrentHashMap 是**线程安全的**，但复合操作（如 check-then-act）仍然需要额外同步 |
| "红黑树比 AVL 树快" | 红黑树和 AVL 树的**插入**旋转次数接近（均为 O(1)）。真正的分水岭是**删除**：AVL 删除最坏 O(log n) 次旋转，红黑树最多 3 次。选红黑树是综合考虑删除性能和实现复杂度的工程决策 |
| "HashMap 的初始容量是 16" | HashMap 的**默认初始容量**是 16，但实际容量是**大于等于初始容量的最小 2 的幂次** |
| "CopyOnWriteArrayList 是线程安全的" | CopyOnWriteArrayList 是线程安全的，但**写操作代价高**（复制整个数组），只适合读多写少场景 |
| "TreeMap 不允许 null key" | TreeMap 不允许 null key 是因为**无法比较大小**，不是设计限制。如果传入的 Comparator 允许 null，可以插入 null key |
| "HashMap 的负载因子是 0.75" | HashMap 的**默认负载因子**是 0.75，可以自定义。但不建议修改，因为 0.75 是空间和时间的平衡点 |
| "LinkedList 的插入是 O(1)" | LinkedList 的插入是 O(1) **在定位到节点之后**。如果需要先定位（如按索引插入），定位本身是 O(n) |
| "ConcurrentHashMap 的 size() 返回精确值" | ConcurrentHashMap 的 size() 返回的是**估计值**，不是精确值。如果需要精确值，使用 `mappingCount()` |
| "Stack 是 Java 中实现栈的标准方式" | `Stack` 是遗留类（extends Vector），**官方推荐用 `ArrayDeque` 替代**，性能更好 |
| "LinkedHashMap 按访问顺序排序" | 默认 `accessOrder=false`，按**插入顺序**排序。只有 `accessOrder=true` 才按**访问顺序**（LRU 语义） |
| "ConcurrentHashMap 扩容时其他线程阻塞等待" | 其他线程**不阻塞**，而是主动帮忙搬迁（helpTransfer），多线程并行扩容 |
| "红黑树插入旋转 O(log n) 次" | 红黑树插入最坏 **2 次**旋转，删除最坏 **3 次**旋转。O(log n) 旋转是 **AVL 树删除**的情况 |
| "Collections.unmodifiableList 返回不可变对象" | 返回的是**视图**，修改原 list 会影响它。`List.of()` 返回的才是**真正的不可变对象** |

---

<a id="decision-cards"></a>

## 🏆 生产决策卡

### 决策卡 1：用户会话缓存

**场景与判断**：
- 需要存储用户会话，Key 是 userId，Value 是 Session 对象
- 高并发读写（10 万 QPS）
- 需要支持按登录时间排序查询

**机制到决策映射表**：

| 需求 | 机制 | 对应回链 | 决策 |
|------|------|---------|------|
| 高并发读写 | ConcurrentHashMap（桶级别锁） | Level 6：CAS + synchronized 桶级锁 | ✅ 使用 ConcurrentHashMap |
| 按时间排序 | TreeMap / ConcurrentSkipListMap | Level 5：红黑树有序性；Level 7.5：跳表无锁 | ⚠️ 需要额外排序或使用 SkipList |
| 过期清理 | WeakHashMap / 定时清理 | Level 8：弱引用机制 | ⚠️ WeakHashMap 不保证及时清理 |

**落地代码**：

```java
// 方案 1：ConcurrentHashMap + 定期排序（大多数场景）
public class SessionCache {
    private final ConcurrentHashMap<String, Session> cache = new ConcurrentHashMap<>();
    private volatile List<Map.Entry<String, Session>> sortedView = Collections.emptyList();
    
    public void put(String userId, Session session) {
        cache.put(userId, session);
    }
    
    public Session get(String userId) {
        return cache.get(userId);
    }
    
    // 定期更新排序视图（异步）
    @Scheduled(fixedRate = 60000) // 每分钟
    public void updateSortedView() {
        sortedView = cache.entrySet().stream()
            .sorted(Map.Entry.comparingByValue(Comparator.comparing(Session::getLoginTime)))
            .collect(Collectors.toList());
    }
    
    public List<Map.Entry<String, Session>> getSortedView() {
        return sortedView;
    }
}

// 方案 2：ConcurrentSkipListMap（实时排序需求）
public class SessionCache {
    private final ConcurrentSkipListMap<String, Session> cache = 
        new ConcurrentSkipListMap<>(Comparator.comparing(Session::getLoginTime));
    
    public void put(String userId, Session session) {
        cache.put(userId, session);
    }
    
    public Session get(String userId) {
        return cache.get(userId);
    }
    
    public List<Map.Entry<String, Session>> getSortedView() {
        return new ArrayList<>(cache.entrySet());
    }
}
```

**不能做的错误决策**：
- ❌ 使用 `Collections.synchronizedMap(new HashMap<>())`：锁粒度太粗，并发性能差
- ❌ 使用 `TreeMap`：不支持高并发，会抛 `ConcurrentModificationException`
- ❌ 在 put() 时直接排序：O(n log n) 太慢，影响写入性能

**验收指标**：
- 吞吐量：≥ 10 万 QPS（读写混合）
- 延迟：P99 < 10ms
- 排序查询延迟：P99 < 100ms（方案 1）或 P99 < 10ms（方案 2）
- 内存占用：每个 Session 约 1KB，100 万用户约 1GB

---

### 决策卡 2：订单列表分页查询

**场景与判断**：
- 需要存储订单列表，按时间倒序排列
- 支持分页查询（offset + limit）
- 单线程或外部加锁

**机制到决策映射表**：

| 需求 | 机制 | 对应回链 | 决策 |
|------|------|---------|------|
| 按时间倒序 | ArrayList + 排序 | Level 2：ArrayList 连续内存，CPU 缓存友好 | ✅ ArrayList + 排序（大多数场景） |
| 分页查询 | subList() | Level 2：subList 是视图不是副本 | ⚠️ subList() 是视图，需要创建副本 |
| 频繁头部插入 | LinkedList | Level 2：LinkedList 头插 O(1) 但随机访问 O(n) | ⚠️ 如果确实频繁头部插入才用 LinkedList |

**落地代码**：

```java
public class OrderList {
    private final List<Order> orders = new ArrayList<>();
    
    public void addOrder(Order order) {
        orders.add(order);
    }
    
    // 按时间倒序
    public List<Order> getOrdersByTimeDesc() {
        List<Order> sorted = new ArrayList<>(orders);
        sorted.sort(Comparator.comparing(Order::getCreateTime).reversed());
        return sorted;
    }
    
    // 分页查询
    public List<Order> getPage(int page, int pageSize) {
        List<Order> sorted = getOrdersByTimeDesc();
        int start = (page - 1) * pageSize;
        int end = Math.min(start + pageSize, sorted.size());
        return new ArrayList<>(sorted.subList(start, end)); // 创建副本
    }
}
```

**不能做的错误决策**：
- ❌ 直接使用 `list.subList()` 返回：会暴露内部数据，修改会影响原 list
- ❌ 使用 `LinkedList` 存储：随机访问 O(n)，分页查询太慢
- ❌ 每次查询都排序：O(n log n) 太慢，应该增量维护排序

**验收指标**：
- 插入延迟：P99 < 1ms
- 分页查询延迟：P99 < 10ms（100 万订单）
- 内存占用：每个 Order 约 500B，100 万订单约 500MB

---

### 决策卡 3：分布式锁

**场景与判断**：
- 需要实现分布式锁
- 锁的粒度是用户级别
- 需要支持超时和可重入

**机制到决策映射表**：

| 需求 | 机制 | 对应回链 | 决策 |
|------|------|---------|------|
| 分布式锁 | Redis / ZooKeeper / 数据库 | 超出本文范围（外部存储） | ⚠️ 需要外部存储 |
| 用户级别粒度 | ConcurrentHashMap + computeIfAbsent | Level 6：CAS + synchronized 桶级锁 | ⚠️ 只是本地缓存，不是分布式锁 |
| 超时 | tryLock(timeout, unit) | 超出本文范围（AQS 限时获取锁，见《AQS 核心机制深度解析》） | ⚠️ 需要处理死锁 |

**落地代码**：

```java
// 本地锁（单机场景）
public class LocalLock {
    private final ConcurrentHashMap<String, ReentrantLock> locks = new ConcurrentHashMap<>();
    
    public void lock(String userId) {
        locks.computeIfAbsent(userId, k -> new ReentrantLock()).lock();
    }
    
    public void unlock(String userId) {
        ReentrantLock lock = locks.get(userId);
        if (lock != null && lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
    
    public boolean tryLock(String userId, long timeout, TimeUnit unit) throws InterruptedException {
        ReentrantLock lock = locks.computeIfAbsent(userId, k -> new ReentrantLock());
        return lock.tryLock(timeout, unit);
    }
}

// 分布式锁（多机场景，使用 Redis）
public class DistributedLock {
    private final RedissonClient redisson;
    
    public void lock(String userId) {
        RLock lock = redisson.getLock("lock:user:" + userId);
        lock.lock();
    }
    
    public void unlock(String userId) {
        RLock lock = redisson.getLock("lock:user:" + userId);
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
    
    public boolean tryLock(String userId, long timeout, TimeUnit unit) throws InterruptedException {
        RLock lock = redisson.getLock("lock:user:" + userId);
        return lock.tryLock(timeout, unit);
    }
}
```

**不能做的错误决策**：
- ❌ 使用 `HashMap` 存储锁：不支持并发，会导致死锁
- ❌ 使用 `synchronized`：粒度太粗，无法实现用户级别锁
- ❌ 不设置超时：会导致死锁，资源无法释放

**验收指标**：
- 锁获取延迟：P99 < 10ms
- 锁释放延迟：P99 < 1ms
- 死锁率：0（通过超时机制保证）

---

### 决策卡 4：事件监听器列表

**场景与判断**：
- 需要存储事件监听器
- 读多写少（注册监听器少，触发事件多）
- 需要支持并发遍历

**机制到决策映射表**：

| 需求 | 机制 | 对应回链 | 决策 |
|------|------|---------|------|
| 读多写少 | CopyOnWriteArrayList | Level 7：写时复制，O(n) 写代价 | ✅ 使用 CopyOnWriteArrayList |
| 并发遍历 | Fail-Safe 快照机制 | Level 7：遍历的是创建时的数组快照 | ✅ CopyOnWriteArrayList 天然支持 |
| 遍历时修改 | 写时复制隔离 | Level 7：修改不影响已创建的迭代器 | ✅ 不会抛异常 |

**落地代码**：

```java
public class EventManager {
    private final CopyOnWriteArrayList<EventListener> listeners = new CopyOnWriteArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);
    }
    
    public void removeListener(EventListener listener) {
        listeners.remove(listener);
    }
    
    public void fireEvent(Event event) {
        for (EventListener listener : listeners) {
            listener.onEvent(event); // 遍历的是快照，不会受并发修改影响
        }
    }
}
```

**不能做的错误决策**：
- ❌ 使用 `ArrayList`：遍历时删除会抛 `ConcurrentModificationException`
- ❌ 使用 `Collections.synchronizedList`：锁粒度太粗，并发遍历性能差
- ❌ 使用 `Vector`：已过时，性能差

**验收指标**：
- 事件触发延迟：P99 < 1ms
- 并发遍历吞吐量：≥ 10 万次/秒
- 内存占用：每个监听器约 1KB，100 个监听器约 100KB

---

<a id="cross-language"></a>

## 🌍 跨语言/跨运行时视角

### 跨语言对比

| Java | Python | Go | JavaScript | C++ |
|------|--------|-----|------------|-----|
| `ArrayList` | `list` | `[]slice` | `Array` | `std::vector` |
| `LinkedList` | `collections.deque` | `list.List` | - | `std::list` |
| `HashMap` | `dict` | `map[K]V` | `Object/Map` | `std::unordered_map` |
| `TreeMap` | `SortedDict` (第三方) | - | - | `std::map` |
| `ConcurrentHashMap` | - | `sync.Map` | - | C++ 标准库无内置并发哈希表；常用 Intel TBB `concurrent_hash_map` 或 `std::unordered_map` + `std::shared_mutex` 封装 |
| `HashSet` | `set` | `map[K]struct{}` | `Set` | `std::unordered_set` |
| `TreeSet` | `SortedSet` (第三方) | - | - | `std::set` |

### 跨语言后仍成立的 N 条判断力

1. **数组/列表是默认选择**：任何语言的动态数组（ArrayList/Vector/slice）都是最常用的容器，因为 CPU 缓存友好
2. **哈希表是查找利器**：任何语言的哈希表（HashMap/dict/map）都是 O(1) 查找，但需要正确实现哈希函数
3. **链表很少用**：除了特殊场景（频繁头部插入删除），链表的性能通常不如数组
4. **并发容器是独立体系**：任何语言的并发容器都有特殊设计（分段锁、CAS、无锁等），不能直接用普通容器
5. **迭代时修改要小心**：任何语言在遍历时修改容器都需要注意，要么用迭代器的方法，要么用并发容器
6. **初始容量要预估**：任何语言的哈希表都需要预估初始容量，避免频繁扩容
7. **Key 要不可变**：任何语言的哈希表 Key 都应该是不可变的，否则会导致数据丢失
8. **负载因子是平衡点**：任何语言的哈希表都有负载因子的概念，0.75 是通用的平衡点
9. **红黑树是通用方案**：C++、Java、Go 的有序容器底层都是红黑树（或类似结构），因为插入删除的旋转次数稳定
10. **性能要实测**：任何语言的容器性能都受具体场景影响，理论复杂度只是参考，必须实测

---

## 📝 总结口诀

```
① 数组是毛坯房，Collection 是精装修
② ArrayList 是默认选择，LinkedList 是特殊情况
③ 哈希是用空间换时间的艺术
④ HashMap 是数组+链表+红黑树的混合体
⑤ TreeMap 是有序的慢郎中，LinkedHashMap 是保持队形的快枪手
⑥ JDK 8 之后，ConcurrentHashMap 是并发 Map 的唯一正确选择
⑦ Fail-Fast 是善意的崩溃，Fail-Safe 是安静的妥协
⑧ ArrayDeque 是栈和队列的默认选择，BlockingQueue 是生产者-消费者的标准解
⑨ 选容器不是背 API，是做架构决策
⑩ Key 要不可变，hashCode/equals 要正确
⑪ 红黑树插入最多 2 次旋转，删除最多 3 次旋转
```


