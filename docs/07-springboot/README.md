# 🚆 Spring Boot 系列：从"创建对象"到"生产急诊"

**一个类怎么变成可用 Bean、框架怎么被整合、事件怎么广播、自动装配怎么生效、HTTP 请求怎么走到 Controller、事务与 AOP 为什么静默失效——最后用一场"数据写一半、无任何报错"的生产事故把它们全部串起来。**

> 主线：00 篇打穿地基（创建对象），01~06 篇在地基上逐层开口子（整合/事件/装配/Web/事务/AOP），07 篇是急诊收口——用前六篇的机制图谱排查真实事故。

---

## 📚 系列导航

| # | 文件 | 主题 | 一句话 |
| - | ---- | ---- | ---- |
| 00 | [springboot-00-backbone.md](springboot-00-backbone.md) | 🚆 创建对象 | 类 → BeanDefinition → 容器 → 生命周期 → 依赖注入 → 三级缓存 → 可用 Bean 的完整因果链 |
| 01 | [springboot-01-framework-integration.md](springboot-01-framework-integration.md) | 🛃 第三方框架整合与配置体系 | starter 报关、@Conditional 验货、四扩展点翻译；Environment/PropertySource/Binder/配置刷新 |
| 02 | [springboot-02-events.md](springboot-02-events.md) | 📢 事件机制与容器通信 | publishEvent → 多播器 → 同步/异步语义 → 早期缓存回放 → 事务事件 |
| 03 | [springboot-03-auto-configuration.md](springboot-03-auto-configuration.md) | 🎫 自动装配深挖 | imports 白名单 → 候选收集/排除/排序 → 条件过滤 → 注册成 Bean；条件评估报告 |
| 04 | [springboot-04-web-request-and-runtime.md](springboot-04-web-request-and-runtime.md) | 🚀 Web 请求链路与运行时刻 | SpringApplication.run 全程 + 内嵌 Tomcat 拉起 + HTTP 请求全链路 + 探针语义 |
| 05 | [springboot-05-transaction-and-data-layer.md](springboot-05-transaction-and-data-layer.md) | 🚀 事务与数据层 | DataSource 免配置之谜、@Transactional 生效链路、默认回滚规则、this.method() 静默失效 |
| 06 | [springboot-06-aop.md](springboot-06-aop.md) | 🚀 横切面与 AOP | @Aspect 生效机制、JDK vs CGLIB、Boot 3 默认 CGLIB 的原因、无 aspectjweaver 静默失效 |
| 07 | [springboot-07-production-practice.md](springboot-07-production-practice.md) | 🚑 生产实践：一次"静默失效"的急诊 | 05+06 两个失效机制叠加的线上事故、分诊到手术的排查方法论、急诊检查单 |

**因果链**：00 创建对象（地基）→ 01 框架整合 + 配置体系（外来 Bean 怎么进场）→ 02 事件（容器内通信）→ 03 自动装配（加依赖就能用的全部机关）→ 04 Web 请求与运行时刻（启动 + 请求两条运行时链路）→ 05 事务与数据层 → 06 AOP（事务为什么是 AOP 的实例）→ 07 生产急诊（机制图谱变检查单）。

---

## ⚠️ 版本与证据边界（系列统一）

| 项目 | 基线 |
| ---- | ---- |
| Spring Framework | **6.1.14**（对应 Boot 3.3.x） |
| Spring Boot | **3.3.5**（版本均出自 spring-boot-dependencies-3.3.5.pom BOM） |
| 运行环境 | macOS + JDK 21.0.11（Azul Zulu） |
| 证据纪律 | 全部结论有可运行 demo 复现（每篇 Implementation 行标注 demo 编号与实测输出）；反编译结论标注版本（6.1.14 / 3.3.5）；不承诺任何性能数字 |

各篇详细的 Specification / Implementation / 未覆盖边界见每篇文首的"⚠️ 版本与证据边界"表。

---

## 🎯 阅读建议

- **走马观花**：只读 00 篇（地基）+ 07 篇（急诊收口）——07 篇的根因就是 05/06 两个失效机制，读完能倒推回去
- **按需深挖**：整合框架踩坑读 01；事件不触发读 02；"加了依赖没生效"读 03；启动/请求/探针问题读 04；事务不生效读 05；切面不生效读 06
- **每篇自洽**：每篇都有"为什么 → 是什么 → 怎么失效 → 全篇因果链总图"，可独立阅读
