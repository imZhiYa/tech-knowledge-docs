# 🧭 02｜打包盒：序列化——订单 O 被打包成什么

> 承接 `dubbo-01-protocol.md`：O 选好了配送信封（Triple/dubbo 协议），但上车前必须装进打包盒（序列化）。本文回答：**盒子（序列化）到底决定什么？为什么默认是 Hessian2？Kryo 更快为什么不默认？跨语言互通到底卡在哪？**

---

# 🏷️ 关键词

序列化 / Hessian2 / Kryo / fastjson2 / protobuf / contentTypeId / 序列化安全检查 / allowlist / 读写对称 / 泛化调用 / PojoUtils / protobuf-wrapper / prefer-serialization / 序列化升级 / ThreadLocal / INPUT_TL / 内存泄漏 / OOM

---

# 🗂️ 目录

- [能力地图](#level-1)
- [全文唯一比喻地图](#level-2)
- [Level 1 为什么需要盒子：socket 只有字节流，盒子要回答七个问题](#level-3)
- [Level 1.3 全局导图：一表看全 Dubbo 序列化家族](#level-4)
- [Level 2 Hessian2：默认盒子——凭什么当了十几年默认](#level-5)
- [Level 3 Kryo：更快的盒子——为什么快、为什么不默认](#level-6)
- [Level 4 protobuf：互通的盒子——信封是 gRPC 的，盒子也得是](#level-7)
- [Level 5 决策：生产怎么选盒子](#level-8)
- [合书自测](#level-9)
- [坑与细节](#level-10)
- [竖切总表 T33–T37](#level-11)
- [版本勘误表](#level-12)
- [生产决策卡](#level-13)
- [跨语言 / 跨运行时视角](#level-14)

---

<a id="level-1"></a>

## 📍 能力地图

| 层级 | 要打穿的认知墙 | 通关标准 |
|---|---|---|
| 盒子职责（七问） | "序列化就是 `ObjectOutputStream`" | 说出盒子七问（类型信息/编码/循环引用/多态/演进/性能/安全）与"信封与盒子正交"。 |
| 默认盒子 Hessian2 | "默认 = 最好的" | 说出 hessian2 默认源于历史路径 + 兼容最广，不是性能；3.2.0+ 协商机制已在顶替它。 |
| Hessian2 两笔债 | "默认 = 放心用" | 说出 ThreadLocal 泄漏机制链（INPUT_TL 持响应缓冲、2.7.x 全系未修、3.3.4 已删）与安全检查 STRICT 的拦截行为（4-21）。 |
| Kryo | "更快就该当默认" | 说出快的原因（注册 ID/变长编码/自描述更简）+ 不默认的四笔账（跨语言断供/安全检查盲区/兼容/扩展库）。 |
| protobuf | "配置一下就能 gRPC 互通" | 说出三种类型信息策略（全名/注册表/schema），只有 schema 协议外共享才真互通（E01 grpc-status: 2）。 |
| 决策 | "选性能最优的" | 四象限决策 + "默认值 = 约束最多的值，不是最快的值"。 |

---

<a id="level-2"></a>

## 🏭 全文唯一比喻地图

```text
订单 O（对象图：属性、循环引用、继承关系）
   │ 盒子要回答七个问题
   ▼
┌─ 打包盒（序列化）──────────────────────────┐
│ ① 类型信息：全名 / 注册 ID / schema 共享   │
│ ② 编码：字符串/数字怎么编码               │
│ ③ 循环引用：引用 ID / 不支持              │
│ ④ 多态继承：记录具体类名                  │
│ ⑤ 演进：schema 纪律 / 反射即席            │
│ ⑥ 性能：反射标记 vs 预编译                │
│ ⑦ 安全：反序列化类检查（allowlist）       │
└────────────────────────────────────────────┘
   │ 盒子上贴标签：contentTypeId（协议头 5 bits）
   ▼
装进配送信封（01 篇：Triple/dubbo 协议）→ 骑手送往后厨
```

| 盒子元素 | Dubbo 技术对象 | 它负责的唯一一件事 |
| ---- | ---- | ---- |
| 盒子样式 | 序列化方案（Hessian2/Kryo/fastjson2/protobuf） | 对象 ↔ 字节 |
| 盒内物品清单 | 类型信息策略（全名 / 注册 ID / schema） | 告诉对端怎么还原对象 |
| 盒子标签 | contentTypeId（协议头低 5 位，MASK=31） | 对端按 ID 选反序列化器 |
| 易碎品标记 | 循环引用 / 多态支持 | 复杂对象图能否无损还原 |
| 盒子厂商 | 官方内置（hessian2/fastjson2）vs 扩展库（kryo 等） | 维护责任与安全边界 |

> 📌 比喻边界：**盒子不决定信封**——换盒子协议头不变（只换标签 ID）；跨语言真互通要盒子 = protobuf（schema 共享），不是"配置一下"就能解决的（E01 grpc-status: 2 实测）。

---

<a id="level-3"></a>

## Level 1 为什么需要盒子：socket 只有字节流，盒子要回答七个问题

### 👶 前置知识关卡

* [ ] 知道 RPC 调用链里"序列化"发生在网络传输前（00 篇 Level 4 的"打包"）
* [ ] 知道 TCP 是字节流，没有对象的概念

### 1.1 徒弟的直觉

徒弟：

> 序列化不就是 `ObjectOutputStream` 往流里一写、对面 `readObject` 读出来吗？为什么 Dubbo 要搞出 Hessian2、Kryo、fastjson2 这么多盒子？

老陈：

> Java 自带 `ObjectOutputStream`（JDK 序列化）就是第一个盒子——它把整个类图都写进去（类名、字段名、字段类型、继承结构、serialVersionUID……），所以它**最重、最慢、最不安全**（历史上多起反序列化 RCE）。线上没人敢用它做 RPC。盒子这门手艺，本质上是用"空间换时间 / 约定换容量"的工程学。

### 1.2 盒子的职责清单（每个序列化方案必须回答的问题）

序列化和协议（信封）是**正交**的两层：信封管"边界/配对/状态"，盒子管"对象 ↔ 字节"。盒子要回答：

| 盒子要回答的问题 | 各方案的回答 | 后果 |
| ---- | ---- | ---- |
| 对象的类型信息怎么传 | 全名（JDK/hessian2/kryo 默认）vs 注册 ID（kryo 可配置）vs 协议外已知（protobuf schema） | 字节大小、跨语言能力 |
| 字符串/数字怎么编码 | 定长/变长/UTF 优化 | 小 payload 差异大 |
| 循环引用怎么表示 | 引用 ID（JDK/hessian2 支持）vs 不支持 | 复杂对象图成败 |
| 继承/多态怎么还原 | 记录具体类名 | 跨语言难 |
| 字段增删怎么兼容 | schema 演进（protobuf）vs 反射即席（hessian2） | 版本演进成本 |
| 性能特征 | 反射标记开销 vs 预编译 | 吞吐 |
| 安全性 | 反序列化类检查 | RCE 攻击面 |

> **说人话**：盒子之间没有"谁绝对更好"，只有"在哪个约束下更好"——同语言内部、跨语言、性能敏感、安全敏感，答案不同（Level 5 决策）。

### Transfer——迁移到其他设计问题

- **盒子七问 = 序列化器评估清单**：拿到任何序列化方案（JSON/Avro/Thrift/MessagePack…）先答七问（类型信息/编码/循环引用/多态/演进/性能/安全）——答完就能定位它在谱系的位置。
- **JDK 序列化的死法是"默认即最重"的反例**：类图全写、慢、RCE 史——官方默认从不等于最优，只承载历史与兼容。

> 本层留下的新账：问题清单有了，但 **Dubbo 究竟有哪些盒子、ID 各是多少**？——排障需要第一张查询表（Level 1.3 导图）。
>
> 🔴 **口诀**：序列化先答七问；类型信息怎么传是分水岭。

---

<a id="level-4"></a>

## Level 1.3 全局导图：一表看全 Dubbo 序列化家族

### 1.3.1 为什么需要这张表

盒子是**协议头的 5 bits** 选出来的（Level 3.4）：对端按 `contentTypeId` 选反序列化器。所以"有哪些盒子、ID 是多少、在哪个版本可用"是排障的**第一张查询表**——看到 `SerializationType mismatch` 或 `Unsupported serialization` 时，先查它。

### 1.3.2 导图（ID 全部 javap 实证；性能/体积为定性参考，官方无统一 benchmark，本系列仅 hessian2/kryo/fastjson2 有 E01 实测）

| 序列化 | 跨语言 | 性能* | 体积* | 安全性 | 协议头 ID | 3.3.4 内置 | 适用协议/模式 | 配置值 | 典型场景 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| **hessian2** | 较好（官方列 Java/Go/C++/PHP/Python/.NET） | 中 | 中 | 3.2+ STRICT 白名单覆盖 | **2** | ✅ | dubbo（默认）；triple wrapper 内层 | `hessian2`（SPI 名，E01 实测）；官方文档示例写 `hessian`（别名支持待验证） | Dubbo2 默认、内部兼容首选 |
| **fastjson2** | 好（JSON 文本跨语言 / JSONB 二进制） | 中 | 中偏大 | 3.2+ STRICT 白名单覆盖 | **23** | ✅ | dubbo；triple wrapper 内层 | `fastjson2` | 3.2.0+ 协商默认目标（两端有依赖时）、调试友好 |
| **protobuf** | 极好（云原生标准） | 高 | 很小 | 强类型 schema，无反射反序列化 | **22** | ❌（不占 SPI，IDL 内置） | triple IDL 模式默认 | `protobuf` | 多语言 / Mesh / 真·gRPC 互通 |
| **protobuf-json** | 极好 | 中 | 大 | 同 protobuf | **21** | ❌ | triple IDL 可选 | `protobuf-json` | triple 下 JSON 调试 |
| **kryo** | 否（Java 专属） | 高 | 很小 | 无框架安全检查覆盖 | **8** | ❌（扩展库 3.3.1） | dubbo；triple wrapper 内层 | `kryo` | 纯 Java 极致性能 |
| **fst** | 否（Java 专属） | 高 | 很小 | 类似 kryo，无安全检查覆盖 | **9** | ❌ | dubbo | `fst` | 纯 Java 高性能备选 |
| **protostuff** | 一般 | 高 | 小 | 反射，无安全检查覆盖 | **12** | ❌ | dubbo | `protostuff` | Java 接口但近似 protobuf 编码 |
| **avro** | 好 | 高 | 小 | 带 schema | **11** | ❌ | dubbo | `avro` | 大数据场景、schema 演进 |
| **msgpack** | 好（Java/C++/Python） | 高 | 小 | 无安全检查覆盖 | 常量 **27**（3.3.4 `Constants.java` 实证），未内置 | ❌ | dubbo | `msgpack` | 二进制 JSON 跨语言 |
| **gson** | 好（JSON 文本） | 中 | 中偏大 | 安全 | **16** | ❌ | dubbo | `gson` | 文本调试场景 |
| **jdk** | 否 | 差 | 大 | 反序列化漏洞史 | **3** | ❌（3.3 起不再内置捆绑） | dubbo | `java` | 不推荐 |
| **fastjson** | 好 | 中 | 中 | 旧版漏洞史，已让位 fastjson2 | **6** | ❌ | dubbo（2.x 时代） | `fastjson` | 2.x 遗留系统 |

\* 性能/体积为**定性参考**（业界共识级，非本系列实测；实测只有 E01 的 hessian2/kryo/fastjson2 三组，见 §3.1）。

### 1.3.3 导图的三个读法（关键不变量）

```text
① ID 是 wire 契约：5 bits 选盒子的编号必须在版本间稳定，否则两端协议头解码错位
   （ID 依据：2.7.9 javap 实证；fastjson2=23 为 3.3.4 javap；kryo=8 另经 2.7.23 源码确认）
② 3.3.4 内置只有 hessian2 + fastjson2（+wrapper 包装扩展）：其余全部走官方扩展库
   （dubbo-spi-extensions，如 org.apache.dubbo.extensions:dubbo-serialization-kryo:3.3.1）
③ triple Java 接口模式 = protobuf-wrapper 双层：数据先 hessian2 序列化，再由 protobuf
   包装传输；wrapper 内声明 serializeType 字段（3.3.4 javap 实证 TripleRequestWrapper）
   → 理论上 wrapper 内层可换任意 dubbo 支持的序列化（依据 serializeType 字段存在，
   推断；官方文档仅描述 hessian 内层，换内层的配置路径待验证）——这是"信封 gRPC + 盒子自选"的接缝
```

> **排障用法**：Consumer 报 `serialization not supported` / 两端序列化不一致时——先对协议头第 3 字节低 5 位（`SERIALIZATION_MASK=31`，3.3.4 实证）解出 ID，再查本表定位盒子；注意 2.x 与 3.x 内置集不同（3.3.4 移除了 jdk/fst/avro/protostuff/gson 等内置），老服务升 3.x 前先核对两端 ID 集。

### Transfer——迁移到其他设计问题

- **导图式学习是排障与选型的通用起点**：家族 + ID + 场景一张表（同 HTTP 状态码表、K8s 资源表），先有查询表再谈理解。
- **"ID 是 wire 契约"适用于任何线格式**：字段编号（protobuf）、魔法头（dubbo）、content-type——版本间漂移就是两端解码错位的事故（T36）。

> 本层留下的新账：盒子有十几种，但**默认的那个凭什么当了十几年默认**？（Level 2 Hessian2）
>
> 🔴 **口诀**：ID 是 wire 契约；两端 ID 集一致才谈换盒子。

---

<a id="level-5"></a>

## Level 2 Hessian2：默认盒子——凭什么当了十几年默认

### 👶 前置知识关卡

* [ ] 知道 Dubbo 2.x 的默认序列化是 Hessian2
* [ ] 知道"默认"与"唯一"的区别

### 2.1 它是 2.x 打下的江山

**为什么 2.x 选 Hessian2 当默认**（官方序列化文档）：

```text
兼容性好：Java 对象图直接映射，无需 .proto/编译期生成
性能好：紧凑二进制，比 JDK 序列化小一个量级
跨语言：官方宣称 Java/Go/C/C++/PHP/Python/.NET 都有实现
→ 这是 2.x 时代"Java 为主、偶尔有异语言消费者"的最优解
```

官方 javadoc 原话（3.x 仍成立）：**"hessian2 is the default serialization protocol for dubbo"**——默认扩展名是 hessian2（javadoc 的 Specification；3.3.4 中 `Hessian2Serialization.contentTypeId=2` 是该实现的 ID 常量，javap 实证，与默认性无关）。

### 2.2 3.2.0 起的自动协商：默认正在被 fastjson2 顶替

官方文档（Implementation 3.2.0+）：**序列化协议自动协商**——官方原文："如果满足条件**两端都为 Dubbo3 特定版本 + 存在 Fastjson2 相关依赖**，则会自动使用 fastjson2 序列化协议，否则使用 hessian2"，协商对用户透明无感。

```text
升级路径（官方 prefer-serialization 机制，3.2.0+）：
服务端：dubbo.provider.serialization=hessian2          # 兜底
       dubbo.provider.prefer-serialization=fastjson2,hessian2  # 协商顺序
客户端：优先按服务端 prefer 列表选；不支持才用 serialization
→ 目的：序列化协议平滑升级，两端版本不一致时不中断
```

> **因果链**：2.x 只有 hessian2 → 3.x 新增 fastjson2 并搞出协商升级机制（动机推断：hessian-lite 是外部依赖 caucho、fastjson2 自研可控——**待验证**）→ 但两端版本不齐会"客户端序列化不了"（官方原文）→ 所以搞出"服务端宣告 prefer 列表 + 客户端按序协商 + 兜底"的机制。**协议升级问题的通用解法：服务端声明能力、客户端降级协商。**

### 2.3 Hessian2 的两笔债：内存泄漏史 + 安全检查

**债一：ThreadLocal 泄漏（2.7.x 实测翻车，R9 证据链）**

- #6847（2.7.8）：`Hessian2ObjectInput` 持线程局部序列化器（2.7.8 源码实证：此时**尚无 cleanup 方法**）→ 500 线程 × 单请求 10M → **10 分钟 OOM**；
- #7770（2.7.11）：ThreadLocal（`INPUT_TL`）持有大对象，**同期 Kryo 则正常**——不同盒子的资源管理风格不同。

完整机制链（issue 锚点推导，非本机复现）：

```text
2.7.x Hessian2ObjectInput 用 INPUT_TL（ThreadLocal）缓存序列化器实例
       ↓
cleanup() 只调用 reset()，不调用 remove()
       ↓
Provider 业务线程池核心线程长期存活（DubboServerHandler-*，07 篇）
       ↓
ThreadLocal：key 弱引用、value 强引用 → 线程不死，value 链不断
       ↓
每个线程各挂一份大响应缓冲（InputStream 链）
       ↓
泄漏速率 = 线程数 × 单响应大小 → 500 线程 × 10M = 10 分钟 OOM
```

三笔账：

| 账 | 内容 | 证据 |
|---|---|---|
| 谁在漏 | `Hessian2ObjectInput` 的 `INPUT_TL`：2.7.8 无 cleanup；2.7.11 起只 reset 不 remove | #6847（2.7.8）、#7770（2.7.11），本地源码 tag 实证 |
| 为什么 Kryo 没事 | 持有物差异：Kryo 的 ThreadLocal 持**配置对象**（类注册表），Hessian2 的 INPUT_TL 持**携带响应缓冲的流对象** | #7770 同期对比 + 2.7.23 `ThreadLocalKryoFactory` 源码 |
| 3.x 现状 | 3.x 重构过序列化与上下文管理，但 3.0.11 仍有 FutureContext 同源问题报告（#10827：20M 对象 × 150 次 OOM）；`future.sync.set` 默认 **true**（#14727 官方确认源码），**3.4 计划改 false**（#14785，breaking change），配套 `future.clear.once=true`；3.3.4 修复状态**待验证** | 06 篇机制一展开 |

故障诊断路径（线上遇到"堆涨不停"时）：

```text
1. 抓 heap dump，查 ThreadLocal 引用链：
   DubboServerHandler-*-thread-N
     └─ ThreadLocalMap
          └─ INPUT_TL → Hessian2ObjectInput → 大缓冲对象
2. 特征确认：泄漏对象都挂在"长期存活线程"上，且响应越大涨得越快
3. 版本定位：2.7.x 大响应 + hessian2 → 高度怀疑 INPUT_TL；3.x 查 FutureContext 与 future.sync.set
```

> **版本必须区分**：2.7.x 的 `INPUT_TL` 泄漏与 3.x 的 `FutureContext` 残余不是同一实现——3.x 重构过序列化与上下文管理，但**不声称 3.3.4 已彻底修复**（未本机复现，标"待验证"）。生产预警靠堆对象增长与 GC 频率，不等 OOM 日志。

**债一的修复路径（2.7.x 怎么办）**：

先给结论：**整条 2.7.x 线都没有修复这个泄漏**（本地 dubbo 源码 2.7.x 全 tag 逐一验证）：

| 版本 | cleanup 状态 | 证据 |
|---|---|---|
| 2.7.8（#6847 报告版） | **无 cleanup 方法**，INPUT_TL 完全无清理 | `dubbo-2.7.8` tag 源码 |
| 2.7.11（#7770 报告版） | 新增 `cleanup()`，只 `mH2i.reset()` 不 remove | `dubbo-2.7.11` tag 源码 |
| 2.7.23（2.7.x 终版） | 仍然只 reset 不 remove | `dubbo-2.7.23` tag 源码 |
| 3.3.4 | **INPUT_TL 已删除**，构造改为 `new Hessian2Input(is)` | 当前源码 |

#7770 讨论给出的修复 diff（`INPUT_TL.remove()`）没有进入任何 2.7.x 版本。

四条修复路径（按优先级）：

1. **根治：升级 3.x**——3.3.4 源码实证：SerializerFactory 改由 `Hessian2FactoryManager`（BeanFactory 级）管理，INPUT_TL 消失。唯一官方根治路径。
2. **2.7.x 内换序列化**：`setSerialization("kryo")`（两端一致，写法同 3.3 节）。2.7.x 时代 kryo 是官方内置模块（3.x 才移出到扩展库，见坑 4），改配置即可。#7770 同期对比 Kryo 正常——但源码真相更精细：Kryo 的 `ThreadLocalKryoFactory` **也用 ThreadLocal**，区别在**持有物**：Kryo 的 ThreadLocal 持配置对象（类注册表，无请求数据）；Hessian2 的 INPUT_TL 持**携带响应缓冲的流对象**。泄漏根源是"ThreadLocal 持了请求级数据"，不是"用了 ThreadLocal"。
3. **2.7.x 内自定义 Serialization 扩展**：Serialization 是 @SPI 扩展点，复制 hessian2 实现、把 `cleanup()` 改成 `INPUT_TL.remove()`，注册为自定义名字。即 #7770 的 diff，官方未落地，只能自建（E01 有自实现 Kryo 扩展的完整走法可参照）。
4. **工程规避（不能改代码时）**：泄漏速率 ≈ 线程数 × 单响应大小——限大响应接口、控业务线程数、堆增长 + GC 频率监控、周期性重启兜底。

> **误用警告**：`-Dfuture.sync.set` 开关对 INPUT_TL 泄漏**无效**（它治的是 FutureContext 残余），且官方确认 2.7/3.0 根本没有该开关（#14727）——参数层没有逃生舱，别指望开关救命。

**债二：反序列化是攻击面（3.x 的回应）**：
- 3.1.6+ 引入**序列化安全检查**（官方文档：目前仅 Hessian2、fastjson2、泛化调用受支持）；3.1 默认 `WARN`，**3.2 默认 `STRICT`**；
- 3.3.4 实测（E01 §2.1.3）：自定义类不在 allowlist 时，`STRICT` 模式直接抛 `IllegalArgumentException`（error code 4-21）；**官方解法：类名写入 classpath 的 `security/serialize.allowlist`**；
- 反序列化类检查的绕过与回归（R9）：#10868（自 2.7.5 起泛化 List 泛型丢失，3.1.2 修复）、#16270（3.2.16 复现：泛化 Map 无 `class` 键可绕过 Serializable 检查，报告人核实 upstream 3.3/3.4 代码相同，修复状态待验证）——安全检查不等于绝对安全（06 篇泛化调用展开）。

> **祛魅**：hessian2 不是因为"最先进"而默认，是因为**历史路径 + 兼容性最广**。3.x 已经在用协商机制把它往下顶，但存量兼容使它退而不死。

### Transfer——迁移到其他设计问题

- **"默认值 = 历史兼容的沉淀"**：任何框架的默认值（gRPC 默认 protobuf、Kafka 默认分区策略）都是约束最多的选择，不是最快的。
- **格式升级的通用模式 = 服务端宣告 + 客户端协商 + 兜底**：prefer-serialization 这套机制可平移到任何"两端版本可能不齐"的格式演进（协议、配置格式、schema 版本）。
- **ThreadLocal 泄漏史是资源管理教科书**：cleanup 缺失或只 reset 不 remove → 线程多 + 对象大 = OOM；"std::thread_local 持有资源"在任何语言都要警惕（02 篇 R9）。

> 本层留下的新账：默认盒子胜在兼容，但**性能呢**？——更快的盒子存在吗？它为什么不当默认？（Level 3 Kryo）
>
> 🔴 **口诀**：默认值不是最快的，是约束最多的；协商 = 宣告 + 降级 + 兜底。

---

<a id="level-6"></a>

## Level 3 Kryo：更快的盒子——为什么快、为什么不默认

### 👶 前置知识关卡

* [ ] 知道 E01 自实现 Kryo 扩展的背景（3.3.4 不内置 kryo）
* [ ] 知道字节大小与 CPU 开销是两回事

### 3.1 为什么它快：E01 实测 + 机制解释

E01 §2.1.1 本机 8 组压测（同机同 JVM，方向性参考）：

```text
                    p50(small/large)   QPS(small/large)   相对 hessian2
dubbo + hessian2    486/418us          1771/2059          基准
dubbo + kryo        354/377us          2472/2389          快 ~25%
triple + hessian2   693/600us          1256/1386          基准
triple + kryo       543/499us          1548/1665          快 ~20%
```

SerializeDump 字节产物（同一 `GreetingRequest{name='DubboDemo', sequence=42}`）：

```text
Hessian2   contentTypeId=2  size=72   hex: 43 30 2a 6f 72 67 ...   （43='C' 类标记）
Kryo       contentTypeId=8  size=54   hex: 01 00 6f 72 67 2e 64 ...（前导字节+类名，前导字节确切语义未深挖）
Fastjson2  contentTypeId=23 size=80   hex: 00 00 00 4c 92 73 ...  （JSONB 长度头+object 标记）

大字符串（~500B）时：Kryo=560 / Hessian2=578 / Fastjson2=587——差距急剧收窄
```

**为什么快（机制）**：

```text
① 注册制：类可用注册 ID 替代全名（kryo 支持预注册类获得短 ID；本实验未用注册优化，
   默认 writeClassAndObject 仍写类名——注册优化的 Dubbo 集成方式待验证）
② 变长编码：整数/长度用变长，小值省字节（如 sequence=42 只 1 字节）
③ 自描述更啰嗦：hessian2 先写 class 定义（'C' + 类名 + 字段描述符），kryo 直接写类标记+数据（E01 hexdump 可见）
④ 大小对象摊薄：内容字节占主导时盒子开销占比下降（E01 实测吻合）
```

### 3.2 为什么不默认：四个账

| 账 | 内容 |
| ---- | ---- |
| **跨语言断供** | Kryo 是纯 Java 库，无官方多语言支持（无官方 .NET 移植），跨语言生态远不如 hessian2 |
| **安全检查不覆盖** | 官方文档：3.x 序列化安全检查**仅支持 Hessian2、Fastjson2 与泛化调用**——Kryo 不在受检范围，反序列化攻击面无框架保护 |
| **版本/兼容** | 对象图强耦合字段顺序，类演进敏感 |
| **不随主版本发布** | `org.apache.dubbo:dubbo-serialization-kryo` 止步 2.7.23；3.x 由官方扩展库 `org.apache.dubbo.extensions:dubbo-serialization-kryo`（Central 实证有 **3.3.1**）提供 |

> **祛魅**：Kryo 快，但快是"同语言内部、安全可控"前提下的快。**默认值永远不是"最快的"，而是"约束最多的"**——这是架构选型的通用思维。

### 3.3 Microkernel 活教材：往 3.3.4 里插一个盒子

3.3.4 内置 SPI 只有 hessian2/fastjson2（javap 实证）。**插一个 kryo 盒子 = 官方 Microkernel+Plugin 契约的完整走一遍**（E01 §6 代码）：

```text
实现 Serialization（contentTypeId=8，对齐 2.7.23 常量）
 + ObjectOutput（writeClassAndObject：写类标记+数据）
 + ObjectInput（三个 readObject 重载全部 readClassAndObject）
 + ThreadLocal<Kryo>（Kryo 非线程安全；注意与 2.3 节 hessian2 的 INPUT_TL 不同——Kryo 的 ThreadLocal 持**配置对象**、无请求数据，不会重演 2.7.x 的泄漏）
注册：META-INF/dubbo/internal/org.apache.dubbo.common.serialize.Serialization
使用：Provider ProtocolConfig.setSerialization("kryo")；Consumer URL ?serialization=kryo
```

**坑 3 铁律（E01 实测，2.1.2）**：**ObjectInput 的三个 `readObject` 重载必须与写侧完全对称**。漏掉 `readObject(Class, Type)` 的对称实现 → 读偏移错位 → 偶发 `unregistered class ID`（只在 payload 变大时出现）——这类 bug 的迷惑性在于**它不总是失败**。

### 3.4 contentTypeId 是怎么搭便车传的（信封与盒子的接口）

`ExchangeCodec.SERIALIZATION_MASK = 31`（3.3.4 javap 实证）：**dubbo 协议头用 5 bits 记录序列化 ID**——信封上贴一张"盒子标签"，对端按标签选盒子。这就是"信封与盒子正交"的物理接口：

```text
dubbo 协议头 16B 的标志字节（第 3 字节）：高 3 位 = 请求/双向/事件标志，
低 5 位 = 序列化 ID（0~31，ExchangeCodec.SERIALIZATION_MASK=31 实证）
盒子 ID 表（实测）：hessian2=2, kryo=8, fastjson2=23
→ 只要两端认同一套 ID，协议不变就能换盒子（这也是 prefer-serialization 协商的载体）
```

### Transfer——迁移到其他设计问题

- **"快"必须带前提陈述**：注册 ID/变长编码/自描述更简的快，前提是同语言、安全可控、小 payload（E01 实测大对象差距收窄）——跨前提比较是伪命题。
- **Microkernel 活教材**：往框架里插实现 = SPI 契约全集演练（实现接口 + 注册文件 + 配置接入），E01 自实现 Kryo 就是模板。
- **读写对称是二进制协议实现者的铁律**：编解码由同一契约驱动；漏一个重载就是"不总失败"的偶发故障（坑 2）。

> 本层留下的新账：快和兼容都有了，但**跨语言呢**？——gRPC 生态里，盒子不对会怎样？（Level 4 protobuf）
>
> 🔴 **口诀**：性能陈述必须带前提；读写对称是二进制协议的铁律。

---

<a id="level-7"></a>

## Level 4 protobuf：互通的盒子——信封是 gRPC 的，盒子也得是

### 👶 前置知识关卡

* [ ] 记得 01 篇 Level 4 的实测：gRPC 帧调用 Triple，协议层全对但业务失败（grpc-status: 2）
* [ ] 知道 protobuf 需要 .proto 定义 + 代码生成

### 4.1 承接 01 篇：信封对，盒子不对

01 篇实测：curl 构造标准 gRPC 帧调用 Triple，返回 `HTTP/2 200 + application/grpc + grpc-status: 2`——**信封 100% gRPC，业务却失败**。为什么？因为 POJO 接口默认走 Hessian2 盒子（Implementation），纯 gRPC 客户端按 protobuf 解码 → 字节对不上。

**结论（01 篇 4.3）：协议兼容 ≠ 序列化互通。互通 = 信封对齐 + 盒子对齐，两层缺一不可。**

### 4.2 官方给的三种模式（Specification：官方序列化文档）

| triple 编程模式 | 盒子 | 说明 |
| ---- | ---- | ---- |
| IDL（.proto） | **protobuf** / protobuf-json | 默认、跨语言互通标准路径；相对 protobuf-wrapper 性能更好（官方：wrapper 性能略有下降） |
| Java 接口 | **protobuf-wrapper** | **两次序列化**：数据先 hessian 再 protobuf——为兼容 Java 接口模型，官方原话"易用性较好但性能略有下降" |
| Java 接口（默认） | hessian2 | 01 篇 grpc-status: 2 的场景 |

> **说人话**：想让 Go/Python 的 gRPC 客户端直接调你的 Dubbo 服务，不是改个配置就行——**接口得用 .proto 定义、盒子得是 protobuf**。protobuf-wrapper 是给"不想动接口定义"的 Java 用户的妥协桥。

### 4.3 盒子的 schema 哲学：三种类型信息策略

```text
① 全名随包走（hessian2/kryo 默认）：自描述，跨语言难，字节重
② 注册表（kryo 优化）：进程内 ID，字节最轻，仅同语言
③ schema 协议外共享（protobuf）：.proto 双方共有，最省字节且跨语言，
   代价是"改 schema 要生成代码+双端发布"——版本演进有纪律
```

**迁移价值**：这三策略是**所有** RPC/消息系统的序列化设计谱系（Thrift、Avro、Kafka 都在 ①/③ 之间选位）。

### Transfer——迁移到其他设计问题

- **三种类型信息策略 = 序列化设计的谱系两端**：自描述（全名随包）vs schema 共享（.proto/Thrift IDL/Avro schema）——评估任何新序列化先定位它在谱系哪端。
- **"改 schema 要发版"是演进代价的通用模型**：schema 纪律（protobuf 只加字段不改编号、向后兼容规则）是所有跨语言契约的公共课。
- **wrapper/桥模式在兼容期到处可见**：protobuf-wrapper（两次序列化）与网关转换层同构——用一层损失换兼容路径，性能损失明码标价。

> 本层留下的新账：盒子全家桶都看完了——**生产到底怎么选**？（Level 5 决策）
>
> 🔴 **口诀**：互通 = 信封对齐 + 盒子对齐；schema 演进靠纪律不靠运气。

---

<a id="level-8"></a>

## Level 5 决策：生产怎么选盒子

### 👶 前置知识关卡

* [ ] 知道"默认值 ≠ 最优值，默认值 = 约束最多的值"
* [ ] 知道升级要平滑（prefer-serialization 机制）

### 5.1 四象限决策

| 场景 | 选什么 | 理由 |
| ---- | ---- | ---- |
| 纯 Java 内部、无多语言、无网关解包 | hessian2（默认，不动）或 kryo | 性能与兼容都够；要性能上 kryo 但先验序列化安全检查是否接受 |
| 需要 Go/Python 客户端 / gRPC 生态 | **protobuf（IDL 模式）** | 唯一真正互通的盒子（Level 4） |
| 网关/边车要解析 payload | hessian2/fastjson2（官方文档提醒 sidecar 拦截 payload 需留意兼容） | 协商机制选，避免"两端协商出的盒子边车不认识" |
| 性能敏感且无跨语言 | kryo（官方扩展库 `org.apache.dubbo.extensions`） | 字节最轻（E01 实测 54B vs 72B）；但**不在安全检查覆盖内**，内网可信环境用 |
| 存量升级 fastjson2 | prefer-serialization 平滑协商 | 3.2.0+ 官方机制，先验证两端依赖 |

### 5.2 决策卡（详见文末）

1. **新服务**：默认不动（hessian2 协商），除非有跨语言/性能硬约束；
2. **跨语言**：上 protobuf IDL，别指望"配置一下"；
3. **自定义盒子**：生产优先用官方扩展库（`org.apache.dubbo.extensions`），自实现只用于教学/学习 SPI。

### 5.3 Trade-off 总账

```text
收益：字节小、快、跨语言、安全
代价：配置/依赖、schema 纪律、安全检查盲区、类演进耦合、调试黑盒化
验证：E01 方向性压测 + 生产灰度（同协议换盒子对比 RT/throughput/错误率）
     + 序列化安全审计（QoS 有 Serialization Security Audit 命令）
```

### Transfer——迁移到其他设计问题

- **四象限决策法（场景 × 候选）是选型模板**：先列约束（跨语言/性能/安全/演进）再选技术，约束驱动而非偏好驱动。
- **"默认不动，除非硬约束"是最低成本策略**：跟默认 = 共享整个生态的兼容与安全维护，个性选择的成本要自己背。
- **安全检查的清单思维**：任何"把外部字节变对象"的环节（RPC/MQ/缓存/反序列化框架）上线前先问：类检查有没有、allowlist 有没有、绕过路径有没有被验证过。

> 本层留下的新账：打包盒选好了，但下一步是**"打给谁"**——门店表从哪来、由谁维护？03 篇接上。
>
> 🔴 **口诀**：先列约束再选盒子；没有硬约束就跟默认。

---

<a id="level-9"></a>

## 📝 合书自测

| 自测题 | 必须答出的不变量 |
|---|---|
| 盒子七问是什么？ | 类型信息 / 编码 / 循环引用 / 多态 / 演进 / 性能 / 安全。 |
| 为什么默认 hessian2 而不是最快的？ | 历史路径 + 兼容最广（官方 javadoc："hessian2 is the default"）；3.2.0+ 协商机制已把它往下顶（fastjson2）。 |
| 3.2.0+ 协商机制怎么工作？ | 服务端 `prefer-serialization` 宣告顺序 + 客户端按序协商 + `serialization` 兜底；两端有 fastjson2 依赖才协商。 |
| Kryo 为什么快、为什么不默认？ | 快：注册 ID / 变长编码 / 自描述更简（E01 实测快 ~25%）；不默认：跨语言断供、安全检查不覆盖、类演进敏感、3.x 靠扩展库（3.3.1）。 |
| 5 bits 怎么选盒子？ | `ExchangeCodec.SERIALIZATION_MASK=31`：协议头第 3 字节低 5 位 = contentTypeId（hessian2=2 / kryo=8 / fastjson2=23）。 |
| 为什么 gRPC 互通必须 protobuf？ | 类型信息三种策略（全名/注册表/schema）；只有 schema 协议外共享（.proto）能跨语言还原对象（E01 grpc-status: 2）。 |
| 安全检查是什么？ | 3.1.6+ 引入、3.1 WARN / 3.2 默认 STRICT、仅 Hessian2/Fastjson2/泛化受支持；自定义类进 `security/serialize.allowlist`（4-21 拦截）。 |
| 读写对称铁律？ | ObjectInput 三个 `readObject` 重载必须与写侧对称；漏一个 = 读偏移错位 → 偶发 unregistered class ID（E01 坑 3）。 |
| Hessian2 ThreadLocal 泄漏为什么 2.7.x 全系未修？ | 2.7.8 无 cleanup、2.7.11+ 只 reset 不 remove（源码 tag 实证）；根源是 ThreadLocal 持了请求级数据（响应缓冲）；3.3.4 已删除 INPUT_TL。 |

---

<a id="level-10"></a>

## ⚠️ 坑与细节

| # | 错误代码/决策 | 错因 | 中央厨房里的后果 | 线上现象 | 修正 |
|---:|---|---|---|---|---|
| 1 | 以为"换盒子性能就翻倍" | 信封与盒子正交没分清 | 小 payload 信封开销占大头 | 大对象上 kryo 收益小 | 压测按 payload 尺寸分组看（E01 8 组数据）。 |
| 2 | 三个 readObject 重载漏一个对称 | 读写契约不完整 | 读偏移错位 | 偶发 unregistered class ID（只在大 payload 出现） | 三个重载全实现（E01 坑 3 铁律）。 |
| 3 | `readObject(Class)` 对接口类型 | 实例化假设错误 | Map 接口实例化失败 | missing no-arg constructor（E01 坑 2） | 传实现类或走接口重载。 |
| 4 | 以为 kryo 有 3.3.4 | 版本没核对 | `org.apache.dubbo:dubbo-serialization-kryo` 止步 2.7.23 | mvn 404 | 用官方扩展库 `org.apache.dubbo.extensions`（3.3.1）。 |
| 5 | 自定义类被 STRICT 拦截 | 安全默认没核对 | 3.2+ 默认 STRICT | IllegalArgumentException（error code 4-21） | 类写入 `security/serialize.allowlist`。 |
| 6 | 以为安全检查覆盖所有盒子 | 官方支持面想当然 | 官方仅支持 Hessian2/Fastjson2/泛化 | Kryo 无保护裸奔 | 内网可信环境才用 kryo（02 篇 L3 账）。 |
| 7 | Hessian2 ThreadLocal 泄漏 | cleanup 缺失（2.7.8）或只 reset 不 remove（2.7.11+） | #6847/#7770：500 线程 × 10M 响应 | 2.7.x 10 分钟 OOM | heap dump 查 `INPUT_TL` 引用链；修复路径（升 3.x / 换 kryo / 自建扩展）见 2.3 节。 |
| 8 | 泛化调用绕过类检查 | 攻击面想当然 | #16270：Map 无 `class` 键绕开 Serializable 检查 | 网关改动一行翻转行为 | 泛化入口白名单 + 版本核验（06 篇展开）。 |
| 9 | 反序列化泛型丢失 | 版本行为不熟 | #10868：2.7.x 泛化返回 List 泛型丢失 | 上线后类型强转炸 | 3.1.2+ 修复；升级前验证泛化路径。 |
| 10 | 想"配置一下"就 gRPC 互通 | 信封与盒子两层契约只对齐一层 | 信封对 + 盒子错 | grpc-status: 2（E01 实测） | 要互通：.proto + triple IDL + protobuf。 |

---

<a id="level-11"></a>

## 📊 竖切总表 T33–T37

| 维度 | T33 选盒子 | T34 编码 | T35 解码 | T36 标签 | T37 安全 |
|---|---|---|---|---|---|
| 位置 | 配置期/协商期 | Consumer 出站 | Provider 入站 | 协议头 | 反序列化 |
| 关键动作 | 按场景选序列化 | 对象 → 字节 | 字节 → 对象 | contentTypeId 5 bits | allowlist 检查 |
| 不变量 | 两端同 ID | 读写对称 | 读写对称 | ID 版本间稳定 | 类受检才还原 |
| 典型坑 | 两端配置不一致 | readObject 重载缺失 | 读偏移错位 | ID 漂移（2.x/3.x 内置集不同） | STRICT 拦截自定义类 |
| 可观测 | 协商结果/两端 ID | 字节产物（SerializeDump） | 解码异常 | SerializationType mismatch | 安全审计日志/4-21 |

---

<a id="level-12"></a>

## 📚 版本勘误表

| ❌ 常见说法 | ✅ 更准确的说法 |
|---|---|
| "hessian2 的 ID=2"是通用事实 | 3.3.4 Implementation（javap）：`Hessian2Serialization.contentTypeId=2`；ID 是 wire 契约，版本间应稳定。 |
| "fastjson2 的 ID=23"是通用事实 | 3.3.4 Implementation（javap）：`FastJson2Serialization.contentTypeId=23`。 |
| "kryo 的 ID=8" | 2.7.23 Implementation（源码）：E01 自实现对齐 + 2.7.23 源码二次确认。 |
| 协议头 5 bits 记序列化 ID | 3.3.4 Implementation（javap）：`ExchangeCodec.SERIALIZATION_MASK=31`。 |
| 全量 ID 表是 3.3.4 的定义 | 2.7.9 Implementation（javap）：hessian2=2…protobuf=22 全表；2.7.23 未本机复核，kryo=8 已另经源码确认。 |
| 3.3.4 内置所有常用盒子 | 3.3.4 Implementation（javap）：聚合 jar + SPI 文件核对，内置仅 hessian2/fastjson2+wrapper；jdk/fst/avro/protostuff/gson/msgpack/protobuf 均不在。 |
| "2.7.x hessian2 一定 OOM" | 泄漏触发条件是"线程长期存活 × 大响应 × 高频"三者叠加（#6847 是 500 线程 × 10M 场景）；小响应低频不一定触发；**整条 2.7.x 线未修复（源码 tag 实证），3.3.4 已删除 INPUT_TL**（2.3 节修复路径）。 |
| triple wrapper 内层固定 hessian2 | 3.3.4 Implementation（javap）：`TripleRequestWrapper` 私有字段 `serializeType`；换内层配置路径**待验证**。 |
| 默认序列化永远是 hessian2 | Specification（官方 javadoc）："hessian2 is the default serialization protocol"；3.2.0+ Implementation：两端有 fastjson2 依赖时自动协商为 fastjson2。 |
| 安全检查覆盖所有序列化 | Implementation（官方文档）：3.1.6+ 引入、3.1 WARN / 3.2 STRICT、仅 Hessian2/Fastjson2/泛化；3.3.4 `DEFAULT_STATUS=STRICT`（javap）。 |
| Java 接口在 triple 下直接用 protobuf | Specification（官方序列化文档）：triple + Java 接口 = protobuf-wrapper 两次序列化（先 hessian 再 protobuf），性能略降；IDL 模式才是真 protobuf。 |
| 裸 Serialization 与 RPC 全链路安全检查行为一致 | 3.3.4 Implementation：**待验证**（E01 §2.1.3 实测现象，根因未定位，禁止臆测）。 |

---

<a id="level-13"></a>

## 🎯 生产决策卡

## 决策卡 1：新服务用什么盒子

- **Decision**：跟随默认（hessian2，3.2.0+ 有 fastjson2 依赖时协商），除非有硬约束。
- **Reason**：默认承载兼容与安全约束；没约束的优化都是过度设计。
- **Alternative**：性能敏感且纯 Java → kryo + 官方扩展库；跨语言 → protobuf IDL。
- **Trade-off**：跟默认牺牲一点字节效率，换兼容面与安全检查覆盖。
- **Validation**：E01 方向性压测 + 生产灰度 RT/throughput/错误率。

## 决策卡 2：跨语言 / gRPC 互通怎么办

- **Decision**：接口改 .proto + triple IDL 模式（protobuf 盒子）。
- **Reason**：只有 schema 共享的盒子能做到真互通；hessian2 盒子在 gRPC 客户端前必炸（E01 grpc-status: 2）。
- **Alternative**：protobuf-wrapper（不想动接口定义，性能略降，官方承认）。
- **Trade-off**：schema 纪律（改 .proto 双端发版）vs 互通能力。
- **Validation**：标准 gRPC 客户端（Go/Python）端到端调通。

## 决策卡 3：自实现/引入新盒子

- **Decision**：优先 `org.apache.dubbo.extensions` 官方扩展库；自实现仅教学。
- **Reason**：扩展库维护者与版本对齐成本更低；自实现必须背"读写对称铁律"。
- **Trade-off**：学习价值 vs 生产维护成本。
- **Validation**：E01 自实现 Kryo 全套（SPI + 8 组压测 + 3 坑）即为验收模板。

---

<a id="level-14"></a>

## 🌍 跨语言 / 跨运行时视角

| Dubbo 视角 | 跨语言同构对象 | 不变的问题 |
|---|---|---|
| 盒子七问 | 所有序列化方案（JSON/Avro/Thrift/Protobuf/Kryo/MessagePack） | 对象 ↔ 字节要回答同一组问题 |
| 类型信息三策略（全名/注册表/schema） | Avro schema、Thrift IDL、Kafka 消息格式、JSON 的自描述 | 类型信息策略决定字节效率、跨语言能力、演进纪律 |
| 5 bits 盒子标签 | HTTP Content-Type、MQ 消息 type 字段 | 编码选择必须随字节一起传输 |
| 读写对称铁律 | protobuf/Thrift 生成代码（同一契约生成两端） | 编解码由同一契约驱动才不会漂移 |
| 反序列化安全 | Java `ObjectInputStream` 历史 RCE、Python pickle 漏洞 | 任何"外部字节变对象"的环节都是攻击面 |

### 跨语言后仍成立的 5 条判断力

1. **类型信息的三种策略（全名/注册表/schema）是所有序列化方案的分水岭**——拿到一个新序列化库，先问"类型信息怎么传"；答案决定字节效率、跨语言能力、演进纪律。
2. **"默认值 = 约束最多的值，不是最快的值"**——任何框架的默认选型都承载历史兼容与安全约束；优化之前先问约束。
3. **格式升级要"服务端宣告 + 客户端协商 + 兜底"**——prefer-serialization 是灰度升级的通用模式，任何"两端版本可能不齐"的格式演进都能用。
4. **读侧与写侧必须完全对称**——三个 readObject 重载漏一个就是偶发故障（E01 坑 3），这是所有二进制协议实现的铁律。
5. **安全性是反序列化的第一性约束**——任何"把外部字节变对象"的环节（RPC、MQ、缓存）都要问：类检查有没有？allowlist 有没有？

---

> 🔴 **电梯版复述**：
>
> **盒子（序列化）回答七问，"类型信息怎么传"是分水岭：全名随包（hessian2/kryo）、注册表（kryo 优化）、schema 协议外共享（protobuf）。默认 hessian2 是历史 + 兼容的产物，不是最优；3.2.0+ 用"服务端宣告 + 客户端协商 + 兜底"平滑顶替它。Kryo 快但断供跨语言、无安全检查覆盖；要真 gRPC 互通只有 .proto + protobuf IDL（E01 grpc-status: 2 是分界线）。协议头 5 bits（MASK=31）是信封与盒子的物理接口——"默认值 = 约束最多的值"是选型第一定律。**
>
> 下一篇：**《🧭 03｜注册中心：菜单为什么不能塞进目录》**——O 打包好了，接下来回答"打给谁"：门店表从哪来、登记什么（接口级 vs 应用级的写放大账）、怎么送达、何时不可靠。
