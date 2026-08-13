# 🧭 Dubbo 系列：从订单 O 的一生到生产选型

**一次 RPC 调用（订单 O）从 `@DubboReference` 到 Provider 方法返回，每一站挂载什么机制、为什么、怎么选。**

> 主线比喻：一座外卖中央厨房。O 的一生 = 收银台（Proxy）→ 验单台（Filter）→ 查门店表（服务发现）→ 派单（Router/LB/Cluster）→ 打包 + 信封 + 骑手（序列化/协议/Netty）→ 后厨出餐（线程池 + 业务）→ 回执原路返回。

---

## 📚 系列导航

| # | 文件 | 主题 | 主线 | 比喻 |
| - | ---- | ---- | ---- | ---- |
| 00 | [dubbo-00-backbone.md](dubbo-00-backbone.md) | 一次 RPC 的一生 | 订单 O 的一生 | 外卖中央厨房总图 |
| 01 | [dubbo-01-protocol.md](dubbo-01-protocol.md) | 传输协议 | O 的运输过程 | 配送信封 |
| 02 | [dubbo-02-serialization.md](dubbo-02-serialization.md) | 序列化 | O 的打包盒 | 打包盒 |
| 03 | [dubbo-03-registry.md](dubbo-03-registry.md) | 服务发现 | 可接单门店表 | 门店营业状态表 |
| 04 | [dubbo-04-registry-protocol.md](dubbo-04-registry-protocol.md) | 注册中心与分布式协议 | 门店表维护方式 | 网点数据同步 |
| 05 | [dubbo-05-loadbalance-cluster.md](dubbo-05-loadbalance-cluster.md) | 集群容错与负载均衡 | O 的派单决策 | 派单与改派 |
| 06 | [dubbo-06-service-governance-generic.md](dubbo-06-service-governance-generic.md) | 服务治理与泛化调用 | 调度规则 | 总部经营规则 |
| 07 | [dubbo-07-threadmodel-quality.md](dubbo-07-threadmodel-quality.md) | 线程模型与质量保障总账 | 骑手与后厨 | 骑手 vs 后厨 |
| 08 | [dubbo-08-production.md](dubbo-08-production.md) | 生产实践与选型 | O 的全程回访 | 投诉回访 |

**因果链**：00 主干（一次调用的一生）→ 01 信封（怎么运）→ 02 盒子（怎么打包）→ 03~04（往哪送、表由谁维护）→ 05（送给谁/失败怎么办）→ 06（规则）→ 07（质量保障总账：线程与预算）→ 08（打烊与回访：选型收口）。

---

## ⚠️ 版本与证据边界（系列统一）

| 项目 | 基线 | 说明 |
| ---- | ---- | ---- |
| Dubbo Java | **Apache Dubbo 3.3.4** | 所有 Implementation 结论均指该版本；上一代对照 2.7.x（2.7.23 终版） |
| 运行环境 | macOS（arm64）+ JDK 21.0.11 + Maven 3.9.16 + 纯 API（无 Spring） | E00~E08 实测环境 |
| 注册中心 | Nacos 2.4.3（standalone）；对照 ZooKeeper 3.9.5 | 03/04 篇实测环境 |
| 本机实测 | E00~E08（共 9 轮） | 实验记录见下"实验证据" |
| 性能声明 | 本机单机串行压测仅作**方向性参考**，不可外推 | 不承诺任何生产 benchmark |
| 证据纪律 | 所有断言分三层标注：Specification（官方文档口径）/ Implementation（源码实测，标版本）/ 待验证 | 未实测项一律显式标注，不写死数字 |

各篇特有边界（待验证项）：

| 篇 | 关键实测 | 明确待验证/未覆盖 |
| ---- | ---- | ---- |
| 00 | E00：线程命名、Filter 同线程、同步调用线程语义 | 性能数字只作数量级参考 |
| 01 | E01：协议压测方向性、curl/gRPC 帧互通 | `tri-exception-code` 精确码值语义 |
| 02 | E01 §2.1：序列化压测/字节产物；2.7.x ThreadLocal 泄漏源码 tag 逐版本核验 | 裸 Serialization 与 RPC 全链路安全检查行为差异根因 |
| 03 | E03：接口级/应用级三态注册数据实测 | 跨机、多副本、权限与网络拓扑故障 |
| 04 | E04：故障窗口实测（Nacos 剔除 1.1s / ZK 锁释放 13.1s / 杀 Nacos 派单 ≥68s） | Distro/JRaft 多节点行为（机制层） |
| 05 | E05：派单分布、kill 重试边界、retries 对比 | 条件路由下发受阻（纯 API 级联未打通） |
| 06 | E06：泛化调用 demo 跑通（3/3） | OOM 机制以 issue 锚点证锚，未本机复现 |
| 07 | E07：线程池默认参数、250 并发饱和拒绝、dispatcher 边界 | 内核是否内置半开熔断器 |
| 08 | 复用 E00~E08 | 优雅停机 PING 无 ack 场景、QoS 注册中心 offline |

---

## 🧪 实验证据

每篇文章的关键结论都有对应实验记录支撑（源码 + 日志原样引用）。实验记录将随仓库一并发布。

| 实验 | 主题 | 实验记录文件 | 对应篇 |
| ---- | ---- | ---- | ---- |
| E00 | 一次调用完整链路打印 | `experiment-000-callchain.md` | 00 |
| E01 | 协议/序列化方向性压测、curl 互通 | `experiment-001-protocol.md` | 01 / 02 |
| E03 | 注册三态数据、残留、订阅通知 | `experiment-002-registry.md` | 03 |
| E04 | 注册中心故障窗口（杀 Nacos / ZK 锁） | `experiment-004-registry-protocol.md` | 04 |
| E05 | 派单分布、kill 重试边界、条件路由受阻 | `experiment-005-loadbalance-cluster.md` | 05 |
| E06 | 泛化调用、治理规则下发 | `experiment-006-service-governance-generic.md` | 06 |
| E07 | 线程池参数、饱和行为、dispatcher、QoS | `experiment-007-threadmodel.md` | 07 |
| E08 | 优雅停机实测（"送完在途"不可靠） | `experiment-008-graceful-shutdown.md` | 08 |

> 正文中的"E0X 实测""3.3.4 源码实证"等标注即指向上述实验记录；GitHub issue 编号（如 #7770）可按编号在 apache/dubbo 仓库检索。

---

## 🎯 阅读建议

- **走马观花**：只读 00 篇（总图）+ 08 篇（选型收口）
- **按需深挖**：哪一站出问题读哪篇——01 协议、02 序列化、03/04 注册中心、05 派单容错、06 治理、07 线程与预算
- **每篇自洽**：每篇都有"能力地图 → 因果链 → 自测 → 坑与细节 → 决策卡"，可独立阅读
