<div align="center">

# 🚀 架构师硬核知识库 · Architect's Hardcore Knowledge Base

**从内核原理解构计算机底层基石**

_Deconstructing the fundamental building blocks of computing from the kernel up._

闭门代码验证 · 极端场景推演 · 杜绝技术碎片化

[![Stars](https://img.shields.io/github/stars/imZhiYa/tech-knowledge-docs?style=for-the-badge)](https://github.com/imZhiYa/tech-knowledge-docs/stargazers)
[![MIT License](https://img.shields.io/badge/license-MIT-blue?style=for-the-badge)](./LICENSE)
![Active Development](https://img.shields.io/badge/status-%F0%9F%9B%A0_Active_Development-yellow?style=for-the-badge)
![Docs](https://img.shields.io/badge/docs-6_篇-success?style=for-the-badge)

</div>

---

## 📖 关于本项目 · About

这是一个 **系统架构师视角** 的计算机科学底层知识库。不满足于「会用 API」，而是深入内核原理，用代码验证每一个结论，推导每一个极端场景。

### 🎯 核心理念

| 原则 | 说明 |
|------|------|
| 🔬 **代码验证** | 所有理论结论必须附有可运行的代码证明，闭门验证而非道听途说 |
| ⚡ **极端推演** | 考虑边界条件、溢出、并发竞争、硬件限制等极端场景 |
| 🧩 **杜绝碎片化** | 知识点之间建立逻辑链接，形成可推导的知识网络而非孤立记忆 |
| 🏗️ **架构视角** | 以系统架构师的高度理解底层设计决策对上层系统的影响 |
| 📌 **诚实免责** | 私有实现、性能数字、预览特性一律标注版本与证据边界，宁可说「须实测」 |

### ✍️ 文档写法：四层递进 + 唯一比喻

每篇深度解析都遵循同一套结构，而非知识点罗列：

```text
Why    → 上一代方案死在哪？不这么做会付出什么代价
What   → 新结构是什么（ASCII 图 + 对照表，不堆术语）
How    → 主线角色的慢动作（带编号时序，标出线性化点）
Transfer → 能迁移到哪些别的设计问题
🔴 口诀  → 全层只需记住的一句话
```

- **全文一个比喻**：AQS 是「只有一个工位的工厂」，线程池是「中央厨房」
- **全文一条主线**：只跟踪一个角色走完全程（AQS 跟踪线程 B，线程池跟踪订单 T）
- **每层留一笔账**：上一层解决不了的问题，正是下一层存在的理由

---

## 📚 已完成 · Completed

```text
docs/
├── 01-cs-foundation/                       # 📐 计算机科学基础
│   ├── binary/
│   │   └── 01-二进制底层思维与位运算.md
│   ├── data-structures/
│   │   └── 🌳 树形数据结构.md
│   └── os-memory/
│       └── 🧠 虚拟内存.md
├── 02-jvm/                                 # ☕ JVM 运行时
│   └── ☕ JVM 运行时机制深度解析.md
└── 03-concurrency/                         # 🔐 并发原语
    ├── 🔐 AQS 核心机制深度解析.md
    └── 🧵 Java 线程池深度解析.md
```

| 章节 | 核心内容 |
|------|---------|
| [📐 二进制思维](./docs/01-cs-foundation/binary/) | 进制转换、原码反码补码、布尔代数、位运算技巧、ALU 逻辑推导 |
| [🌳 树形数据结构](./docs/01-cs-foundation/data-structures/) | 二叉树遍历（递归 / 非递归 / Morris）、BST、AVL 旋转、红黑树性质推导 |
| [🧠 虚拟内存](./docs/01-cs-foundation/os-memory/) | 分页机制、页表结构、TLB 缓存、缺页中断、页面置换算法、MMU 地址翻译 |
| [☕ JVM 运行时机制](./docs/02-jvm/) | 类加载、运行时数据区、对象布局、GC 算法与收集器、内存模型 |
| [🔐 AQS 核心机制](./docs/03-concurrency/) | CLH 前驱接力、dummy head、Condition 双队列、共享传播、与 ObjectMonitor 对照 |
| [🧵 Java 线程池](./docs/03-concurrency/) | ctl 位打包、execute 三道门、Worker 与 AQS、打烊协议、虚拟线程与响应式 |

### 🔥 最近更新

| 日期 | 内容 |
|------|------|
| 2026-07-28 | 新增《🧵 Java 线程池深度解析》—— 7 层递进 + 13 坑 + 8 张生产决策卡 + 6 个可运行实验 |
| 2026-07-24 | 新增《🔐 AQS 核心机制深度解析》—— 从 CLH 接力到生产架构决策 |
| 2026-07-12 | 新增《☕ JVM 运行时机制深度解析》 |

---

## 🗺️ 阅读路径 · Reading Paths

文档体量较大，**不必线性通读**。按你的目标选路径：

<details>
<summary><b>🎯 我要准备面试</b></summary>

```text
📐 二进制 → 🌳 数据结构 → 🧠 虚拟内存 → ☕ JVM → 🔐 AQS → 🧵 线程池
```
每层末尾的 🔴 **口诀**串起来就是一份电梯版复述稿；
每篇的「合书自测」章节可用来检验是否真的打通。

</details>

<details>
<summary><b>🔧 我在排查线上问题</b></summary>

| 症状 | 去哪 |
|---|---|
| 线程池队列堆积 / 大量 BLOCKED | [🧵 线程池 · 线上排查工具箱](./docs/03-concurrency/) |
| 锁竞争、P99 突刺 | [🔐 AQS · 生产决策卡](./docs/03-concurrency/) |
| GC 频繁 / 内存溢出 | [☕ JVM 运行时](./docs/02-jvm/) |
| 缺页、内存映射异常 | [🧠 虚拟内存](./docs/01-cs-foundation/os-memory/) |

</details>

<details>
<summary><b>🏗️ 我在做架构评审</b></summary>

直接看每篇末尾的 **生产决策卡** 与 **设计记录清单**：
- 🧵 线程池：8 张决策卡（核心链路 / 埋点 / 批处理 / P99 排障 / @Async / 虚拟线程迁移 / 动态线程池 / 舱壁隔离）
- 🔐 AQS：5 张决策卡（分片锁 / Semaphore 限流 / 生产者消费者 / P99 决策树 / Virtual Threads）

每张卡都包含「**不能做的错误决策**」与「**验收指标**」两栏。

</details>

<details>
<summary><b>☕ 我只关心 Java 并发</b></summary>

```text
🔐 AQS（同步器地基）→ 🧵 线程池（任务执行与容量治理）
```
两篇共用同一套工厂比喻，AQS 讲「等不到资源怎么办」，线程池讲「任务多于执行者怎么办」。

</details>

---

## 🧪 代码验证 · Verified by Code

本库坚持「**结论必须可复现**」。以《🧵 Java 线程池深度解析》为例，六个最反直觉的论断都配了可运行实验：

| 实验 | 验证的论断 |
|------|-----------|
| Lab 1 | 幽灵 Worker：`core=max=1` 却出现 `poolSize=0`，队列有任务却无人消费 |
| Lab 2 | 无界队列废掉 `maximumPoolSize`：同样 `core=2/max=200`，实测 2 线程 vs 90 线程 |
| Lab 3 | `submit` 吞异常：同一个异常，`execute` 打印、`submit` 完全静默 |
| Lab 4 | `newWorkStealingPool` 守护线程：任务没跑完 JVM 已退出，无任何告警 |
| Lab 5 | 五种提交方式：阻塞点、异常去向、`invokeAll` 超时后的 `CancellationException` |
| Lab 6 | `InheritableThreadLocal` 在池化场景天然失效：上下文永远停在第一个请求 |

> 文档中引用的所有实验输出**均为真实运行结果**，非手写示意。

---

## 📐 版本与证据边界 · Evidence Policy

技术文档最大的风险不是写错，而是**让读者无法判断哪里可能过时**。本库的做法：

- **标注基线版本**：如「以 OpenJDK 21 LTS 为准，源码取自 `jdk21u`」
- **标注核实日期**：引用预览特性时写明「最后核实于 YYYY-MM」，便于判断是否滞后
- **区分语义变化与代码重构**：不把「改了写法」说成「改了行为」
- **私有实现不背字段**：`ObjectMonitor._cxq`、AQS `Node.status` 等属实现细节，只讲不变量
- **性能数字必须实测**：文中量级仅供教学参考，一律注明「以你的部署压测结果定案」
- **公开自我修正**：发现此前结论有误时保留修正记录，而非悄悄改掉

---

## 🛠 配套工具 · Companion Tools

| 项目 | 说明 | 链接 |
|------|------|------|
| 🧪 **cs-visual-tools** | CS 可视化交互工具（树结构 / 位运算 / 虚拟内存动画） | [→ 查看](https://github.com/imZhiYa/cs-visual-tools) |
| 🧬 **dev-lab** | 底层技术沙盒与基准测试代码（Binary & Benchmark Lab） | [→ 查看](https://github.com/imZhiYa/dev-lab) |

---

## 🔍 如何使用 · Getting Started

```bash
git clone https://github.com/imZhiYa/tech-knowledge-docs.git
cd tech-knowledge-docs
```

**阅读建议**

- 文档使用**显式 ASCII 锚点**，GitHub / VS Code / Typora 预览均可稳定跳转
- 篇幅较长的文档开头都有「带着问题来的走这条快速路径」索引表
- 建议配合官方源码阅读：看源码前先 `java -version`，再打开**同版本**源码

**每篇文档包含**

| 模块 | 作用 |
|------|------|
| 📍 能力地图 | 分层列出「要打穿的认知墙」与「通关标准」 |
| 🏭 唯一比喻地图 | 一张 ASCII 图 + 比喻与技术概念的对照表 |
| 🟢 Level 1..N | Why / What / How / Transfer 四段递进 |
| 🧪 合书自测 | 一页时序图 + 必须答出的不变量 |
| ⚠️ 坑与细节 | 错误代码 → 错因 → 线上后果 → 修正 |
| 📊 竖切总表 | 横轴时间、纵轴维度的全景对照 |
| 📚 版本勘误 | ❌ 常见说法 vs ✅ 更准确的说法 |
| 🏆 生产决策卡 | 场景 → 决策 → 错误做法 → 验收指标 |

---


## 🤝 参与贡献 · Contributing

欢迎任何形式的贡献，提交 [Issue](https://github.com/imZhiYa/tech-knowledge-docs/issues) 或 PR 均可。

**尤其欢迎这几类**

- 🐛 **事实性纠错**：指出版本判断、源码引用、结论推导中的错误（请附源码链接或可复现步骤）
- 🧪 **补充验证代码**：为某个结论提供可运行的最小复现
- 📝 **补充生产案例**：真实踩坑经历比理论更有价值

**提交规范**

```bash
docs(scope): 新增《emoji 标题》     # 新增文档
docs(scope): 修正 xxx              # 内容修正
chore: xxx                        # 构建 / 配置
```

---

## 📄 许可证 · License

本项目基于 [MIT License](./LICENSE) 开源。文档内容欢迎转载，请注明出处。

---

<div align="center">

**从底层到顶层，构建不可动摇的计算机知识体系**

如果这些文档帮到了你，欢迎点一个 ⭐ Star

[@imZhiYa](https://github.com/imZhiYa)

</div>
