# 🚑 SpringBoot 生产实践：一次"静默失效"的急诊（系列 07）

> 本篇文章回答三个问题：
> 1. **为什么**：00-06 篇建立的机制知识，在生产故障面前为什么"用不上"？机制与症状之间缺了什么？
> 2. **是什么**：一次"数据写一半、无任何报错"的线上事故，从分诊到手术的完整排查——为什么最终根因是 05 篇和 06 篇讲过的两个失效机制叠加？
> 3. **怎么用**：怎么把 00-06 的机制图谱变成一张"急诊检查单"？排查方法论是什么？怎么预防下一次？
>
> 前置：00 篇的四层创建链（代理怎么来的）、01 篇的配置体系、02 篇的事件机制、03 篇的自动装配（条件评估）、04 篇的请求链路、05 篇的事务（自调用失效/回滚规则/事务激活打点）、06 篇的 AOP（代理边界：private/static/final、advisor 收集、aspectjweaver 缺失）。本篇主线角色：**一次线上故障**。

---

# ⚠️ 版本与证据边界

| 维度 | 本文承诺 |
| ---- | ---- |
| 事故性质 | **教学推演场景**：本文的"线上事故"是教学构建（因果链每一环都锚定 00-06 已实测机制），**不是真实事故记录**；不包含任何编造的数字、时间线、benchmark。排查过程与结论可直接复现 |
| 代码实证 | `knowledge/springboot/experiments/code/` 下 demo14×2（triage.TriageApp 复现事故 + 内置急诊检查单 6 项输出；publish.StartupTimerApp 启动计时/懒初始化/timeline 三跑法）+ demo15×2（BootCircularApp Boot 默认禁环 / RefreshFailApp refresh 失败自清理）+ demo16.GracefulShutdownApp（immediate/graceful 停机对照）+ demo18.AotGenerationApp（Spring AOT 引擎端到端：生成 5 个源码文件 + CGLIB 代理字节码 + 170 个 RuntimeHints）+ demo19×2（buffered.StartupProfilerApp 启动端点四方法：actuator startup 端点 + BufferingApplicationStartup + /actuator/startup 耗时快照 353 步 / jfr.JfrStartupApp JFR 深度剖析 6138 事件），本机实测输出原样引用；回看 00-06 全部实验（demo01–demo19，共 58 个含 main 的可运行入口文件） |
| 实测环境 | macOS + JDK 21.0.11（Azul Zulu）+ spring-boot **3.3.5** + spring-tx **6.1.14** + H2 内存库 |
| 运行方式 | `cd knowledge/springboot/experiments/code && ./build.sh && ./run.sh demo14.triage.TriageApp`（lib/ 43 个 jar） |
| Specification | 回滚规则（只回滚 RuntimeException/Error）、传播语义、@Transactional 语义（05 篇）、AspectJ 表达式语义、@EnableTransactionManagement 语义（06 篇） |
| Implementation | 本机 6.1.14 实测：JdbcTransactionManager bean 类名 `org.springframework.jdbc.support.JdbcTransactionManager`、事务 advisor 数=3（demo14 包）、bean 为 CGLIB 代理 `$$SpringCGLIB$$0`、bad 路径事务激活=false 且 count=1 残留、good 路径激活=true 且回滚（TriageApp 输出）；启动计时与 lazy 差异、ApplicationStartup 步骤树大头（config-classes.parse=376ms、beandef-registry.post-process=382ms）（StartupTimerApp 三跑法，本机仅机制演示）；StartupEndpoint 类存在于 spring-boot-actuator-3.3.5 jar（jar 实证）；Boot 3.3.5 默认禁循环依赖（demo15.BootCircularApp 实测）；refresh 失败自动销毁已创建单例 + getBean 拒绝 + 不支持二次 refresh（demo15.RefreshFailApp 实测）；优雅停机 immediate 中断在途请求 / graceful 排空完成后才退出（demo16 实测，本机时序数字仅机制演示）；3.3.5 无 server.shutdown-timeout（Shutdown 枚举仅 GRACEFUL/IMMEDIATE + CountDownLatch.await() 无参等待，javap/字节码实证） |
| 待验证 | 无 @Order 时多切面顺序不确定性（06 篇）；AspectJ LTW 真实表现（06 篇文档语义）；不同版本 JDK 代理类名格式；线程上下文在异步线程丢失的具体行为（04 篇未实测该点，本篇不展开）；GraalVM Native Image 构建/运行表现、AOT 启动加速数字、AppCDS/懒初始化全局开关等生产效果（Spring AOT 引擎"生成什么"已实测 demo18；启动步骤树已实测 demo19（端点/JFR 两通道），**加速数字与 native 行为需按业务测量**，未编造 benchmark） |
| 未覆盖 | 分布式事务/补偿；消息队列失败语义；数据库锁与隔离级别实战（05 篇标注待验证）；真实监控平台（Prometheus/Grafana）集成细节 |

---

# 🏷️ 关键词

生产实践 | 故障排查 | 静默失效 | 急诊检查单 | 分诊 | 症状指纹 | 排除法 | @Transactional 失效 | private 修饰符 | 自调用 | 事务激活打点 | afterCompletion | 重构引入缺陷 | 预防医学 | 可观测性 | 故障演练 | 回归验证 | 慢发布 | 启动优化 | 懒初始化 | ApplicationStartup | timeline | Spring AOT | GraalVM Native Image | AppCDS | 蓝绿发布 | 探针 | 优雅停机 | SIGTERM | 排空 | shutdown-timeout | refresh 失败 | 循环依赖默认值 | @MockitoBean

---

# 🗂️ 目录

- Level 1 为什么需要生产实践篇：机制与症状之间的"翻译层"
- Level 2 事故现场：一次"数据写一半"的线上故障（教学推演）
- Level 3 分诊：症状 → 候选机制（症状指纹库）
- Level 4 急诊检查单：7 项检查，逐项排除（demo14 实测）
- Level 5 手术：根因确认与修复方案决策
- Level 6 ICU 与预防医学：验证、评审检查点、可观测性、演练
- Level 7 第二类病例：慢发布（10 分钟）——无 AOT 与有 AOT 的优化路径（7.6 交叉补充：AOT/Native/CDS 三技术区分）
- Level 8 优雅停机协议：打烊不能砍单（demo16 实测）
- 排查方法论：四步法（Transfer）
- 00-06 机制收束总图
- 面试自查表
- 版本勘误表
- 生产决策卡（5 张）
- 系列索引

---

# 🏭 全文唯一比喻：急诊室

一家医院（研发团队）的急诊科。病人（一次线上故障）被推进来，救治流程：

- **分诊**：护士看一眼症状，先判断"急不急、可能是哪个科室的病"——不诊断，只分流。
- **检查**：抽血、CT、心电图——**每一台仪器测一个明确的指标**，用指标说话，不靠猜。
- **手术**：指标锁定病灶后，才开刀。
- **ICU**：术后观察，确认真的好了（回归验证）。
- **预防医学**：疫苗、体检、健康档案——让下一次得病概率变低。

本系列的比喻对照：

| 急诊室 | 生产故障排查 |
| ---- | ---- |
| 症状 | 报错、数据异常、性能劣化（现象层） |
| 分诊 | 症状 → 候选机制（"可能是哪一层失效"） |
| 检查单 | 每项检查 = 一个**机制指纹**（00-06 篇实测过的可观察证据） |
| 手术 | 根因修复（决策卡选方案） |
| ICU | 实验复现 + 回归验证 |
| 预防医学 | 评审检查点 / 测试覆盖 / 可观测性 / 演练 |

**为什么必须用"指纹"而不是"经验"**：静默失效类故障（无报错、无日志）最危险——没有症状指向时，只能靠"机制知识反查"。本系列 00-06 的每个实验，本质上都是在给每个机制拍"指纹照片"（代理类名、advisor 数量、事务激活标志、afterCompletion 回调……）。07 篇就是把指纹照片整理成一张急诊检查单。

---

# Level 1 为什么需要生产实践篇：机制与症状之间的"翻译层"

## 1.1 知识的"正向"与"反向"

00-06 篇的学习方向都是**正向**：从机制出发，观察它怎么工作。

```
机制（源码/规范）──正向──► 行为（实测输出）
```

生产排查的方向**相反**：从现象出发，找机制。

```
症状（现场现象）──反向──► 机制（哪一层失效了？）
```

老陈（架构师）：前六篇让你能"讲清楚一个机制"，但这不等于能"排掉一个故障"。排故障的起点是症状，症状是模糊的——"数据写一半""偶发""本地复现不了"——它不告诉你先查哪层。缺的是一层**翻译器**：把症状翻译成候选机制，再把候选机制翻译成可验证的指纹。

徒弟：那是不是把文档背熟就行了？哪个机制都能讲，总有一个命中。

老陈：背熟文档是"症状 → 机制"的**一选多**——你猜了七个可能，然后呢？挨个试？生产环境不允许瞎试。正确姿势是**排除法**：每项检查一次排除一批候选，几轮下来只剩一个。所以检查单的顺序是有讲究的——**先做便宜且区分度高的检查**（看一个类名排除一半可能），**后做贵且区分度低的**（改代码、加日志重发）。

## 1.2 为什么"静默失效"是排查的终极难度

静默失效 = 系统行为不符合预期，但**没有任何报错**。它之所以难：

- 没有堆栈 → 日志分析无从下手。
- 现象偶发（依赖调用路径/配置/环境）→ 复现不了 = 验证不了。
- 后果延迟（数据错乱几天后才被发现）→ 现场已被污染。

本系列的机制知识对静默失效是**刚需**：报错时堆栈会指引你；静默失效时，**只有机制知识能告诉你"哪里可能悄悄断了"**。而"哪里可能悄悄断"恰好是 00-06 每篇的"失效"部分：

| 篇 | 机制 | 静默失效形态（实测过的） |
| -- | ---- | ---- |
| 00 | 容器创建链 | bean 不是代理/属性没注入（UnwrapApp：代理字段 null） |
| 01 | 配置体系 | 配置项来源被覆盖、优先级错（demo05 多来源实测） |
| 03 | 自动装配 | 条件不满足 → bean 缺失或被替代（demo10 条件评估报告实测） |
| 05 | 事务 | 自调用失效、检查异常不回滚、代理字段 null NPE（demo12×4 实测） |
| 06 | AOP | 无 aspectjweaver 用户切面静默失效（role 过滤）、private/static/final 注解失效（VisibilityApp/TxVisibilityApp 实测） |

## 1.3 本篇章的定位

07 篇不做三件事：

- 不教新的 Spring 机制（机制都在 00-06）。
- 不列"经验清单"（"遇到过这种错就查这个"——那是八股）。
- 不编造事故细节（场景是教学推演，每一环可复现）。

只做一件事：**把 00-06 的机制图谱编译成一张可执行的急诊检查单**，并用一次完整的故障演示它的用法。

---

# Level 2 事故现场：一次"数据写一半"的线上故障（教学推演）

> ⚠️ 场景为教学构建：为演示排查方法而构造的故障，因果链每一环都锚定 00-06 已实测的机制；不包含真实事故的任何具体数字。复现脚本：`demo14.triage.TriageApp`。

## 2.1 现场

支付服务最近一次版本上线后，某天运营发现：**一批用户的订单状态更新了一半**——订单表已扣款，但支付流水表少了几行，且**没有任何报错日志**。

关键现场信息：

- 只影响**一条业务路径**（支付），同服务其他功能正常。
- **本地环境和测试环境复现不出**（同样的测试用例都通过）。
- 版本唯一变更：一次"重构"——把支付方法里的逻辑拆成了多个小方法。
- 数据库日志显示：部分 insert 已提交，但**没有对应的 rollback 记录**（也没报错）。

## 2.2 第一个错误动作

值班同学的第一反应：**看日志**。没有异常日志。第二反应：**加日志重发**——把事务方法里的关键点全部 System.out，然后重新触发支付。依然"一切正常"。

徒弟：这不科学！线上坏了，本地全好，加了日志也没用——难道是数据问题？

老陈：先别往数据上想。注意一个细节：**加日志打点的人，把日志加在了事务方法里面**。如果问题出在"事务方法压根没被事务包裹"，那打点当然看不出问题——你测的是业务代码，不是事务。这就是没带检查单的代价：**你验证的是你猜的，不是该查的**。分诊，然后按检查单走。

---

# Level 3 分诊：症状 → 候选机制（症状指纹库）

## 3.1 症状拆解

把现场信息拆成最小症状，每个症状只允许做"机制层"的联想（**分诊不做诊断**）：

| # | 症状 | 机制层联想（候选） | 指向 |
| - | ---- | ---- | -- |
| S1 | 部分写入成功，部分缺失 | 事务没生效（原子性被破坏） | 05/06 |
| S2 | 无任何报错 | 事务机制**静默失效**类（非回滚失败） | 06 |
| S3 | 本地/测试环境复现不出 | 环境差异（配置、代理模式、依赖）或路径差异（调用方不同） | 01/06 |
| S4 | 只有重构涉及的方法出问题 | 注解位置/可见性/调用结构变化 | 06/05 |
| S5 | 无 rollback 记录 | 事务从未开启（不是回滚失败） | 05 |

## 3.2 分诊结论：候选机制集

S2 + S5 是最强信号：**事务从来没开启过**（而不是开启后回滚失败——回滚失败至少会有报错或 rollback 记录）。结合 S4（重构引入），候选收敛到 06 篇的两个"静默失效"机制 + 05 篇的调用路径机制：

1. **候选 A**：事务注解所在方法**不可被代理覆写**（private/static/final）→ 注解被读取但拦截器永不触发（06 篇 TxVisibilityApp 前置条件）。
2. **候选 B**：事务方法被**同 bean 内部自调用**（this.xxx()）→ 绕过代理（05 篇 SelfInvocationApp）。
3. **候选 C**：代理本身没生成/advisor 没收集（如无 aspectjweaver 的 role 过滤）→ 但那样会是**全部**事务方法失效，与 S1"只有一条路径"矛盾，先压低优先级。
4. **候选 D**：回滚规则问题（检查异常不回滚）→ 但 S5 无 rollback 记录 + 无报错不符合，压低。

徒弟：所以直接改代码？把 private 改成 public 试试？

老陈：你选对了怀疑对象，但"改代码试试"是手术，不是检查。手术要有病灶影像支撑。现在去做检查——检查单上每一项都有明确的**判定标准**，做完 6 项，病灶会自己现形。

---

# Level 4 急诊检查单：7 项检查，逐项排除

> 检查单的设计原则：**每项检查测一个指纹，判定标准是"排除了什么"**；先便宜后贵。下面每一项都能在 `demo14.triage.TriageApp` 的真实输出中找到对应（附实测输出）。

## 检查 1：这个 bean 是代理吗？

**指纹**：bean 实际类名（CGLIB 代理 = `$$SpringCGLIB$$N`；JDK 代理 = `$ProxyN`；裸 bean = 目标类名）。

**怎么测**：`bean.getClass().getName()` + `AopUtils.isAopProxy(bean)`（00 篇的创建链：代理在 BPP 阶段生成，06 篇 ProxyInternalsApp 实测过类名格式）。

**判定**：是代理 → **排除"没代理"**；不是代理 → 问题在更前面（00 篇创建链/自动装配）。

```
真实输出（TriageApp）：
[检查1] bean 类 = demo14.triage.TriageApp$PaymentService$$SpringCGLIB$$0（代理 ✓ CGLIB）
```

**分诊含义**：事务代理存在（CGLIB，Boot 3 默认），候选 C 的"代理没生成"分支被排除——但也说明代理模式是 CGLIB，排除了"JDK 代理导致非 public 方法不可代理"这一条（06 篇前置条件：JDK 代理下 protected/包可见才失效；CGLIB 下它们能生效——**但 private 仍然不行**，检查 6 见分晓）。

## 检查 2：事务管理器装配了吗？

**指纹**：`PlatformTransactionManager` bean 的类名（05 篇实测：本机为 `org.springframework.jdbc.support.JdbcTransactionManager`）。

**怎么测**：`ctx.getBean(PlatformTransactionManager.class).getClass().getName()`。

**判定**：类名正确 → **排除"没装配/装配错管理器"**（03 篇自动装配：DataSource 条件链失败会导致换管理器或缺失——这是"本地好线上坏"的重要候选，必须实测排除）。

```
真实输出（TriageApp）：
[检查2] 事务管理器 = org.springframework.jdbc.support.JdbcTransactionManager
```

## 检查 3：事务 advisor 收集了吗？

**指纹**：`AbstractAdvisorAutoProxyCreator.findCandidateAdvisors()` 的数量（06 篇 ProxyKindApp 跑法 3 的实测：**无 aspectjweaver 时用户 Advisor 被 role 过滤静默失效**，但事务 advisor 是 role=2 不受影响）。

**怎么测**：反射调用 `internalAutoProxyCreator` bean 的 `findCandidateAdvisors()`。

**判定**：>0 → **排除"advisor 收集链断裂"**。注意区分：自定义切面依赖 aspectjweaver 存在；**框架事务 advisor 是基础设施（role=2），aspectjweaver 缺失不影响它**——所以此项只排除"整个 AOP 装配链没起来"，不排除"某条方法路径没被匹配"。

```
真实输出（TriageApp）：
[检查3] 事务 advisor 数 = 3（基础设施 advisor；demo14 包无自定义切面）
```

## 检查 4：事务真的激活了吗？（分水岭）

**指纹**：`TransactionSynchronizationManager.isActualTransactionActive()`——方法执行时当前线程有没有活跃事务（05 篇 SelfInvocationApp 同款打点）。

**怎么测**：在事务方法体内打点（这正是值班同学"加日志"的正确姿势——**打点位置必须在方法体内、且打的是事务状态而不是业务状态**）。

**判定**：true = 拦截器已介入；false = **拦截器从未触发**——问题锁定在"代理没拦住这个方法"，而不是"事务语义错误"。

```
真实输出（TriageApp）：
[检查4-bad] public pay() 调 private @Transactional applyTx()：事务激活=false
[检查4-good] public @Transactional payGood()：事务激活=true
```

**这一步是分水岭**：检查 1-3 证明"代理在、管理器在、advisor 在"，检查 4 证明"这个方法的调用路径上，拦截器没介入"。剩下的问题只有一个：**为什么代理没拦住这条调用路径？** 候选 A（修饰符）和候选 B（自调用）进入决赛圈。

## 检查 5：同步回调佐证拦截是否介入

**指纹**：`TransactionSynchronization.afterCompletion` 回调（05 篇 TxBasicsApp 实测过 COMMITTED/ROLLED_BACK 输出）。

**怎么测**：方法内 `registerSynchronization(...)` 注册回调，事务结束自动触发。

**判定**：good 路径输出 `ROLLED_BACK` → 拦截器介入证据完整；bad 路径**无回调输出** → 佐证拦截器从未介入。附带生产知识：**无事务时 registerSynchronization 会抛 IllegalStateException**（同步回调只能在事务内注册）——写回调前必须确认事务激活。

```
真实输出（TriageApp）：
[检查5-good] afterCompletion=ROLLED_BACK（回滚）
```

## 检查 6：方法可见性与调用路径（决赛圈）

**指纹**：目标方法的修饰符 + 调用链（06 篇 VisibilityApp：private/static 连表达式匹配都不通过、final 匹配≠拦截；05 篇 SelfInvocationApp：自调用绕过代理）。

**怎么测**：读代码——**事务方法是不是 private/static/final？是谁调的？调用方和被调用方是不是同一个 bean？** 检查 4 已把范围缩到这两点，纯代码阅读即可判定，不需要再动环境。

**判定**（TriageApp 对应结构）：

```
重构后形态（事故代码）：
public void pay(int id) {
    ...
    applyTx(id);              ← 自调用（同 bean 内部，绕过代理）
}

@Transactional                ← 注解在 private 方法上
private void applyTx(int id) {
    jdbc.update(...);          ← autocommit 直接提交
    throw new RuntimeException(...);  ← 无事务可回滚
}
```

**双重失效合流**（这是本案例的根因，06 篇 TxVisibilityApp 的修饰符失效 + 05 篇 SelfInvocationApp 的自调用失效叠加）：

1. `applyTx` 是 **private**：CGLIB 子类无法覆写 private 方法（06 篇实测：private 连表达式匹配都不通过），**任何模式下拦截器都够不着它**。
2. `pay()` 内 `this.applyTx()` 是**自调用**：`this` 指向目标对象而非代理（05 篇实测），**即使 applyTx 是 public 也照样失效**。
3. 后果量化（TriageApp 检查 6）：

```
[检查6] bad 路径抛异常后 count=1（残留 1 行 → 事务未生效！private 注解 + 自调用双重失效）
[检查6] good 路径抛异常后 count=1（仍是 bad 残留的 1 行，good 自己的插入已回滚 → 事务生效）
```

对照：`payGood()` 把 `@Transactional` 留在 public 方法上、由外部调用 → 回滚干净。

## 检查 7：回滚规则排除

**指纹**：异常类型（05 篇 TxBasicsApp：默认只回滚 RuntimeException/Error；检查异常需 rollbackFor）。

**判定**：业务异常是 `RuntimeException` 子类 → 默认规则就会回滚 → **排除回滚规则问题**。若此处是检查异常，则根因候选多一个 D（回滚规则），需检查 `rollbackFor` 配置。

## 检查单收束：病灶确诊

```
检查1 代理存在（CGLIB）          → 排除"没代理"
检查2 事务管理器正确            → 排除"装配错"
检查3 advisor 已收集（=3）      → 排除"装配链断"
检查4 bad 路径事务激活=false     → 锁定"拦截器没介入"  ★分水岭
检查5 good 路径有回调/bad 没有   → 佐证拦截器介入差异
检查6 private 方法 + 自调用      → 根因：双重失效
检查7 RuntimeException          → 排除回滚规则问题
```

徒弟：所以根因就是**重构时把注解跟着业务代码搬进了 private helper**？这么简单的事故，绕了这么大一圈？

老陈：对，根因是"简单"的。但注意：**检查 1-3 帮你排除的候选，每一个都曾是线上真实事故**（无 aspectjweaver、装配链断裂、代理没生成——06 篇跑法 3 实测过的静默失效）。没有检查单，值班同学会在这三个候选上各试一轮"加日志/改配置/重启"，每轮都是几小时。检查单的价值不是解这道题，是**保证每题都按同样的顺序用最便宜的代价解开**。

---

# Level 5 手术：根因确认与修复方案决策

## 5.1 修复目标

让 `pay()` 整条路径处于一个事务中。注意修复的**约束**：逻辑拆成 helper 是合理重构（方法职责单一），**目标不是"把代码改回去"，而是"让拆出来的方法仍在事务内"**。

## 5.2 方案对比（生产决策卡 1：事务边界放哪）

| 方案 | 做法 | 生效条件 | 优点 | 代价/风险 | 适用场景 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| A 注解回到 public | `@Transactional` 移回 `pay()`（public 入口），helper 保持无注解 | 外部调用 pay() 经代理 | 最简、无新依赖、行为与重构前一致 | helper 若被别处直接调用则事务边界丢失（需确认调用面） | **默认首选** |
| B @Lazy 自注入 | `@Autowired @Lazy PaymentService self;` 内部 `self.applyTx()` | 事务注解留在 helper 上 + 经代理调用 | 保留 helper 内注解语义，边界精确到 helper | 多一层引用、自注入有循环依赖心智负担（Boot 2.6+ 默认禁止普通自引用，@Lazy 是标准解法）；helper 必须非 private | helper 语义独立、需被多处调用 |
| C TransactionTemplate | helper 内 `transactionTemplate.execute(...)` | 无代理依赖，编程式 | 不依赖代理，private 也成；边界极精确 | 事务代码与业务代码混写，回滚规则要手写 lambda 异常传播 | 事务逻辑在深层/不可代理处 |
| D 拆独立 bean | 把 applyTx 拆到 `PaymentTxService`（public + 注入调用） | 跨 bean 调用天然经代理 | 结构最清晰，可测试性最好 | 类/装配成本，过度拆分 | 事务逻辑复杂、需要独立测试 |

**决策**：方案 A。理由：事故根因是"注解位置 + 调用路径"，最小修改恢复原语义；方案 B/C/D 留作演进选项（规则：**事务边界 = 一个 bean 的 public 外部入口**，这个入口要稳定、可被代理覆盖、经外部调用）。

## 5.3 手术后的代码形态

```
修复后：
@Transactional                    ← 注解回到 public 入口
public void pay(int id) {
    ... 逻辑A ...
    applyTx(id);                  ← helper 无注解，仍在事务内
}
private void applyTx(int id) { ... }
```

事务边界回到 `pay()`——与重构前一致；helper 的拆分保留（代码组织改进不丢失）。

**为什么这个形态在机制上是正确的**（对照 5.1 的约束）：

1. **失效双条件已消解**。自调用失效的本质是"注解 + 绕过代理的调用路径"同时存在：注解在 helper 上（private、被 `this.applyTx()` 调用）→ 注解从未被代理解析。修复后：helper 上**没有注解了**（"注解+自调用"的组合不存在）；`pay()` 有注解但它是外部入口——外部调用必然经 CGLIB 代理进入（Spring 拿到的 bean 就是代理，SelfInvocationApp 断言实证），事务拦截器在方法进入前开启事务；
2. **helper 不需要自己的代理**。事务上下文是**线程绑定**的（`TransactionSynchronizationManager`，ThreadLocal）：事务在 `pay()` 入口开启后，同一线程调用栈内的任何方法（包括 private helper）都处于该事务中——helper 执行时的 insert 用的是事务绑定的那个连接。事务是"线程的上下文"，不是"方法身上的标签"；
3. **helper 保持无注解是干净的**。事务边界只由 `pay()` 一处声明；若在 helper 上再标 @Transactional（private 方法标注解本身无效，见 06 篇坑 4）只会制造下一个坑。

一句话收束：**修复后的形态 = 事务边界回到"代理能拦截的入口"，helper 留在事务内是因为它在该入口的调用栈中**——这正是"让拆出来的方法仍在事务内"（5.1 约束）的机制答案。

---

# Level 6 ICU 与预防医学：验证、评审、可观测性、演练

## 6.1 ICU：修复验证必须"复现 + 回归"两步

验证不是"再跑一遍测试"（测试本来就全过——这正是事故没被发现的原因）。验证要回答：**故障路径现在真的被事务包住了吗？**

1. **复现脚本回归**：`demo14.triage.TriageApp` 的 good 路径（`payGood`）就是修复后的行为——事务激活=true、抛异常后回滚干净。**把故障形态固化成测试**，而不是删掉故障复现脚本。
2. **指纹断言**：在修复后代码上重跑检查 4/5/6——事务激活=true、afterCompletion=ROLLED_BACK、count 无残留。**用指纹判死，不用"看起来正常"判死**。
3. **回归影响面**：确认无其他事务方法被连带改变（检查 1-3 的输出不变）。

## 6.2 预防医学 1：代码评审检查点（把机制知识前置）

事故根因是"注解位置 + 调用路径"，这两点**在代码评审阶段就能肉眼发现**。评审检查点（全部锚定 05/06 篇实测机制）：

- `@Transactional`/`@Cacheable`/`@Async` 只允许出现在：**public + 非 final + 经外部调用**的方法上（06 篇 VisibilityApp/TxVisibilityApp）。
- **自调用代码审查**：方法体内 `this.xxx()` 调用带事务注解的方法 = 疑似失效（05 篇 SelfInvocationApp）。
- **重构审查专项**：搬动带注解方法时，检查注解是否"跟着搬"了（本事故的直接来源）。
- **新事务方法必查**：调用面（谁调它？经代理吗？）与异常路径（抛的是运行时异常吗？）。

## 6.3 预防医学 2：测试覆盖（把失效路径变成用例）

"本地复现不出"的真相通常是**覆盖盲区**，不是环境差异。把静默失效的已知形态固化成用例：

- 集成测试必须**从代理入口调用**（`@SpringBootTest` 注入的 bean 是代理；直接 new 目标类 = 测试的代理被绕开，测不出拦截语义）。
- 失败路径要测：**抛异常的用例**验证回滚（05 篇 TxBasicsApp 场景 2/3/4 就是模板）。
- 事务生效断言：`TransactionSynchronizationManager.isActualTransactionActive()` 在测试里可直接断言（检查 4 的自动化形态）。

## 6.4 预防医学 3：可观测性（把静默变成有声）

静默失效的可怕之处是**没有声音**。生产可观测性的目标：让"事务该开没开"变得有声。

- 事务打点/日志：方法入口打 `isActualTransactionActive()`（成本极低，检查 4 的常驻形态）。
- 数据一致性核对任务：对账/巡检定时核对跨表数据（本事故的直接发现手段）。
- 数据库侧：确认连接的 autocommit 状态与事务提交/回滚日志可查。

## 6.5 预防医学 4：故障演练

把"事务静默失效"做成**人为注入的演练**（构造一个 private @Transactional 方法，观察监控/测试能否抓出差异）。演练的价值：**验证你的监控和用例真的能发现这类故障**——如果演练没被发现，说明预防手段本身是无效的。

---

# Level 7 第二类病例：慢发布（发布/回滚 10 分钟）——无 AOT 与有 AOT 的优化路径

> 场景延续教学推演：发布（或回滚）一次要 10 分钟，线上每次发版都很痛苦。这一节把急诊方法论（先测指纹、再动手术）用在**性能故障**上。⚠️ 除 StartupTimerApp 本机实测外，AOT/GraalVM/AppCDS/K8s 内容均为官方文档语义（标注待实测），未编造任何 benchmark。

## 7.1 因果拆解：为什么发布/回滚会慢

发布慢不是单一原因，先把因果拆开（急诊室的分诊同样适用于性能问题）：

```
发布总耗时 ≈ 启动耗时 × 滚动批次数
            + 每批就绪等待（readiness 判定 + 探测间隔）
            + 探针重试惩罚（liveness 超时 < 启动时间 → 反复重启 → 永远不就绪 → 发布超时）
```

**回滚为什么也慢**：回滚 = 重新发布旧版本 = **把启动慢的代价再付一次**。`kubectl rollout undo` 只是换镜像重发，不是"退回去"。

三个放大器，对应三条优化杠杆：

| 放大器 | 机制 | 杠杆 |
| ---- | ---- | ---- |
| 应用启动慢 | 组件扫描 + 条件评估 + 反射 + 重初始化（连接池/RPC 客户端/线程池） | 启动优化（7.3/7.4） |
| 滚动节奏差 | 每批都要等就绪；批次多、探测间隔长 | 发布架构优化（批次/就绪判定/预热） |
| 探针死循环 | liveness 超时小于启动时间 → 反复重启（K8s 文档语义） | 探针参数与启动时间匹配 |

**杠杆最大的优化在发布架构层，不在应用层**：让回滚不依赖重启——蓝绿/金丝雀部署下，回滚 = 流量切换（秒级），而不是重新发布旧版本（10 分钟）。启动优化是兜底，让"必须重启"的场景（发布新版本）也快。

## 7.2 体检先行：启动耗时指纹（先测后改，不凭感觉加参数）

与事故排查同一条方法论：**先拿指纹，再动手术**。启动慢的"指纹"有三个层次，从粗到细：

**指纹 1：总耗时与 lazy 差异**（`demo14.publish.StartupTimerApp` 跑法 1/2，本机实测，仅机制演示）：

```
跑法1 默认：    应用对象构建=66~79ms；run（启动）=2197~2380ms；首次 getBean=0ms（启动期已初始化）
跑法2 lazy：    应用对象构建=66~102ms；run（启动）=1453~1646ms；首次 getBean=506~507ms（推迟初始化在此触发）
```

**指纹 2：ApplicationStartup 步骤树**（Boot 3 内置，`spring.main.startup=timeline` 或编程式 `BufferingApplicationStartup`；actuator 的 `StartupEndpoint`（/actuator/startup）输出同一数据）。本机实测前 10 步（StartupTimerApp，7.2 的步骤树指纹）：

```
spring.boot.application.starting = 17ms
spring.boot.application.environment-prepared = 87ms
spring.context.config-classes.parse = 376ms      ← 大头：配置类解析（含组件扫描）
spring.context.beandef-registry.post-process = 382ms  ← 大头：BeanDefinition 后处理（含条件评估）
（注意步骤存在并行，耗时不可简单相加）
```

**指纹 2 端到端：StartupEndpoint + BufferingApplicationStartup（demo19.buffered.StartupProfilerApp，3.3.5 本机实测）**——排查启动慢的四个方法：

1. **引入依赖并暴露端点**：`spring-boot-actuator` + `management.endpoints.web.exposure.include=startup`（demo 内 `setDefaultProperties` 注入）；
2. **启动类注册 `BufferingApplicationStartup`**：`app.setApplicationStartup(new BufferingApplicationStartup(4096))`；
3. **访问端点获取耗时**：启动完成后 `GET /actuator/startup` → HTTP 200，**353 个步骤**，响应结构实测为 `{"springBootVersion":"3.3.5","timeline":{"startTime":...,"events":[{startupStep:{name,id,tags}, duration:"PT0.026322S",...}]}}`（步骤在 `timeline.events[].startupStep`，duration 是 ISO-8601）；
4. **FlightRecorderApplicationStartup 深度剖析**（demo19.jfr.JfrStartupApp）：启动步骤写成 JFR 事件，与 GC/类加载事件同时间轴交叉分析。

Top 步骤实测（demo19 故意放了一个构造 sleep 2.5s 的慢 bean，模拟连接池初始化）：

```
spring.context.refresh                       6631 ms  ← 总闸（含所有 bean）
spring.beans.instantiate                     2520 ms  tags: beanName=...SlowBean  ← 慢 bean 精确定位
spring.beans.instantiate                     2511 ms  tags: beanName=...SlowBean
spring.context.beandef-registry.post-process  447 ms  tags: postProcessor=ConfigurationClassPostProcessor...
spring.context.config-classes.parse           440 ms  tags: classCount=149
spring.boot.webserver.create                  324 ms  tags: factory=TomcatServletWebServerFactory...
```

**两种指纹互相印证**：`config-classes.parse` 376ms（StartupTimerApp）vs 440ms/classCount=149（demo19，actuator 引入后类更多）——同一批步骤名、量级一致，这就是"指纹库"：换项目先对齐指纹基线，再谈优化。

JFR 深度剖析实测（demo19.jfr，6138 个事件）：
- Spring 提交的事件类型名 = **`org.springframework.core.metrics.jfr.FlightRecorderStartupEvent`**（不是 `jdk.ApplicationStartup`，实测确认）；`jfr print --events org.springframework.core.metrics.jfr.FlightRecorderStartupEvent out/startup-recording.jfr` 可转文本，事件带 **stackTrace（记录步骤发出点）** + tags（`beanName=...`）；
- 与 JVM 事件同流：`jdk.GCPhaseParallel x1535`、`jdk.NativeLibrary x1049`、`jdk.ObjectAllocationSample x325` 与启动步骤同一 .jfr 时间轴 → 回答"慢是业务 bean 还是 JVM 层面"（例如步骤树显示 bean 慢，但同窗口大量 GC 事件 → 先看 GC 再看 bean）；
- 版本事实（jar 实证）：Boot 3.3.5 的 `BufferingApplicationStartup` 在 **spring-boot jar**（`org.springframework.boot.context.metrics.buffering` 包），spring-core 6.1.14 的 metrics 包只有 `DefaultApplicationStartup` + jfr 实现；`StartupEndpoint` 构造器只接受 `BufferingApplicationStartup`。

**指纹 3：事件级序列**（04 篇 RunTraceApp 的启动序列方法：run 反编译 16 条序列）——细到每个事件时点。

**体检结论示例**：若步骤树显示 `config-classes.parse` 是大头 → 优先裁剪扫描范围/排除自动配置；若大头是某 bean 初始化（如连接池）→ 优先懒初始化/参数调优；若大头是类加载（JVM 层）→ AppCDS。**措施要与指纹匹配，否则是瞎调参**。

## 7.3 无 AOT 路线（JVM 上优化）

按杠杆从大到小、风险从小到大排序：

### 第一层：发布架构层（最大杠杆）

- **蓝绿/金丝雀 + 流量切换**：回滚 = 切流不重启（10 分钟 → 秒级）。这是"回滚慢"问题的**正解**——不是让回滚变快，而是让回滚不需要重启。
- **滚动批次调优**：批次大小、每批就绪等待时间与启动时间匹配；启动 2 分钟却每批只等 30 秒，等于就绪探测还没过就进入下一批（或一直超时）。
- **探针节奏**：liveness 超时必须大于"最坏启动时间"（否则反复重启，永远不就绪）；readiness 应包含业务就绪（连接池建连、缓存预热完成）而非仅端口可连。
- **启动后预热**：readiness 通过前，用 warmup 请求把 JIT 编译成本移到发布流程内（工程惯例，效果需按业务实测）。

### 第二层：应用层（Spring 配置）

- **懒初始化**（`spring.main.lazy-initialization=true`）：实测 trade-off——本机启动 2.2s→1.6s（省的是 SlowInitService 的 500ms 初始化 + 少量连带 bean），但**首次 getBean 触发初始化 +507ms**：首请求慢、且初始化错误从"启动时爆炸"变成"首次请求时爆炸"（错误延迟暴露）。官方文档也明确全局懒初始化会改变行为语义（文档语义）。适合"启动时间敏感、首请求可容忍、错误可在就绪探针期暴露"的场景；有损的就绪检查（readiness 里 getBean 全量触发）可以中和错误延迟问题。
- **裁剪组件扫描**：`@SpringBootApplication(scanBasePackages=...)` 缩小扫描范围 → 直接削减步骤树里 `config-classes.parse` 这个大头的输入量（实测大头 376ms）。
- **排除无用自动配置**：`spring.autoconfigure.exclude` / `@SpringBootApplication(exclude=...)` → 削减 `beandef-registry.post-process` 与条件评估量。
- **连接池懒建连**：HikariCP 默认启动即创建连接（默认 minimumIdle 与 maximumPoolSize 相同，文档语义），数据库连接慢时启动被拖长；按业务调 `minimumIdle` / `initializationFailTimeout`（效果待实测）。
- **@Lazy 标注个别重 bean**：不用全局开关，只推迟确定的重初始化（比全局 lazy 更可控）。

### 第三层：JVM 层

- **AppCDS**（应用类共享归档）：把应用类加载结果落盘，启动时内存映射加载（JDK 官方特性：JDK 12+ 默认启用 CDS 用于 JDK 类；应用类共享需显式 dump + `-XX:SharedArchiveFile`，文档语义）。效果依赖类加载在启动耗时的占比，**需按业务实测**（步骤树能告诉你占比）。
- **类路径瘦身**：减少依赖 jar 数 → 减少类加载与扫描量（与 03 篇"自动配置来自类路径存在性"联动：jar 少了，自动配置分支也少了）。
- **JIT 参数**：不列"调参清单"——收益必须由步骤树/实测背书，否则是玄学。

## 7.4 有 AOT 路线（Spring AOT 与 GraalVM Native Image）

> 以下为 Spring Boot 3.3.5 官方文档语义；**Spring AOT 引擎已本机实测**（demo18，spring-context 6.1.14，无需 GraalVM）；**GraalVM Native Image 本机无环境，未实测**。与 06 篇代理机制的衔接按实测/推导区分标注。

### 先分清两个概念（最常见的混淆点）

| 概念 | 做什么 | 跑在哪 | 效果 |
| ---- | ---- | ---- | ---- |
| **Spring AOT 引擎** | 构建期分析应用，生成 Bean 定义注册代码 + RuntimeHints（反射/资源/序列化/代理），构建期评估 @Conditional | 仍在 JVM（JIT） | 启动减少运行时扫描/反射/条件评估 → 启动变快；**构建变慢** |
| **GraalVM Native Image** | 把应用 AOT 编译成原生可执行文件（含 SubstrateVM 运行时） | 原生可执行文件（无 JIT） | 启动毫秒级、内存低（官方文档语义）；**构建数分钟级**、动态能力受限 |

### Spring AOT 做了什么（与 00-06 篇机制的衔接）

- **Bean 定义**：00 篇的创建链中"运行时解析配置类 → BeanDefinition"这一步，AOT 在**构建期**完成并生成注册代码（BeanFactoryInitializationAotProcessor，文档语义）。
- **条件评估**：03 篇的 @Conditional 在**构建期**评估并裁剪 → 运行时不再评估。
- **代理生成**：06 篇的 CGLIB/JDK 动态代理由 AOT 在**构建期**生成（代理类字节码/hint），运行时不再动态生成（文档语义）——这是 06 篇"代理是运行时机制"在 AOT 路线下的根本差异。
- **反射**：@Autowired 注入等反射操作转为构建期生成的调用代码或 RuntimeHints（文档语义）。

### 实测：Spring AOT 引擎端到端（demo18，6.1.14）

demo18 直接调用 AOT 引擎（复刻 Boot maven 插件 `process-aot` 的核心调用链：`GenericApplicationContext` 只加载 bean 定义不 refresh → `ApplicationContextAotGenerator.processAheadOfTime` → 生成代码 + hints），把"生成了什么"完整打出来：

**产物**：5 个源码文件 + 2 个 CGLIB 代理类字节码（`Cfg$$SpringCGLIB$$0.class` + FastClass）+ 170 个反射 RuntimeHints。

**实测三件事被前移到构建期**（生成代码直接抄录）：

```java
// ① 组件扫描/工厂方法 → 直接注册代码（运行时不再扫描 demo18 包）
public void registerBeanDefinitions(DefaultListableBeanFactory beanFactory) {
    beanFactory.registerBeanDefinition("aotGenerationApp.Cfg", Cfg.getCfgBeanDefinition());
    beanFactory.registerBeanDefinition("aotGenerationApp.Greeter", Greeter.getGreeterBeanDefinition());
    beanFactory.registerBeanDefinition("greeter", Cfg.getGreeterBeanDefinition());   // @Bean 工厂方法
}
// ② CGLIB 代理构建期生成：BeanDefinition 直接拿方法引用，运行时零动态生成
beanDefinition.setInstanceSupplier(AotGenerationApp$Cfg$$SpringCGLIB$$0::new);      // 代理类已编译进产物
// ③ 扫描到的 @Component → 构造器引用
beanDefinition.setInstanceSupplier(AotGenerationApp.Greeter::new);
```

与 00-06 篇的衔接（从机制推导到实证）：

1. **00 篇"运行时解析配置类 → BeanDefinition"这一步，被 AOT 编译成直接代码**：BeanDefinition 不再是运行时解析产物，而是构建期编译产物（`new RootBeanDefinition(...)` + `setInstanceSupplier(...)`）；
2. **06 篇"代理是运行时机制"在 AOT 路线的变化**：代理**语义没变**（还是 CGLIB 子类、方法引用直接 new），**变的是生成时机**——字节码在构建期生成，运行时只是加载（输出里 `[文件:CLASS] ...Cfg$$SpringCGLIB$$0.class` 即实证）；
3. **RuntimeHints 是 native 的配套解药**：170 个反射类型 hint 里包含生成的代理/FastClass 类——普通 JVM 反射天然可用，native 下缺 hint 即运行期失败，AOT 引擎自动收集（实测 `@Reflective` 触发收集链路）。

> ⚠️ 边界（不越界承诺）：demo18 只实证**引擎生成了什么**；**启动加速数字、GraalVM Native 构建/运行仍为官方文档语义**（本机无 GraalVM）。生产选型必须有业务实测背书（7.3 口径：先测指纹再动手术）。

### GraalVM Native Image 的代价（Trade-off 是选型核心）

- **构建慢**：数分钟级（官方文档语义），CI 节奏改变。
- **动态能力受限**：反射/动态代理/动态类加载不再天然可用，缺 hint 即运行期失败；Spring AOT 生成的 RuntimeHints 是配套解药，但第三方库可能缺 hint。
- **无 JIT 分层**：峰值性能依赖编译质量；GC/内存模型不同（文档语义），传统 JVM 调优经验不直接适用。
- **调试困难**：无动态字节码、堆栈/工具链差异。
- **生态约束**：Spring 支持矩阵内可用（Boot 3.x + GraalVM 支持是官方路线，3.3.5 文档语义），但自研动态能力（如运行时生成类、Groovy 脚本）直接出局。

### 三条路线对比（决策输入）

| 维度 | 无 AOT（7.3 优化） | Spring AOT（JVM 模式） | GraalVM Native |
| ---- | ---- | ---- | ---- |
| 启动目标 | 数十秒级（随优化） | 秒级（文档语义） | 毫秒级（文档语义） |
| 构建成本 | 无变化 | 变慢（构建期分析） | 数分钟级 |
| 动态能力（反射/代理/类加载） | 完整（JVM 生态） | 完整（hint 只是加速） | 受限（需 hint） |
| 监控/热部署/调优经验 | 沿用 JVM 全家桶 | 沿用 JVM 全家桶 | 新工具链 |
| 适用 | 大部分常规业务 | 启动敏感 + 保留 JVM 生态 | FaaS/无服务器/大规模副本/启动成本敏感 |
| 本系列证据 | 本机实测（StartupTimerApp） | 文档语义，待实测 | 文档语义，待实测 |

## 7.5 决策卡（并入"生产决策卡"第 4 张）

见文末决策卡 4。核心口径：**先发布架构（回滚切流），再应用层（扫描裁剪/懒初始化），再 JVM 层（AppCDS）；AOT 按启动要求与动态能力依赖选型，且必须有测量背书**。

## 7.6 22 章交叉补充：AOT 不是"一个技术名"（Boot 4.1 基线文档语义）

对照 24 章 22 章（Boot 4.1 基线）与 7.4 的衔接，三个"提前做工作"的技术必须分开验收：

| 技术 | 介入阶段 | 产物 | 与 7.4 的关系 |
| ---- | ---- | ---- | ---- |
| Spring AOT | Spring 构建期 | Java source / bytecode / 代理 / hints | 7.4 的"Spring AOT 引擎"，在 JVM 跑 |
| Native Image | 构建期闭包分析 | 平台相关 executable | 7.4 的"GraalVM Native" |
| CDS / JDK AOT Cache | JVM 启动/运行时 | 类元数据归档 | JVM 层优化（7.3 第三层），≠ Native |

**因果**：三者都"提前做事"，但改的是不同阶段——Spring AOT 改构建期对象图生成，Native 改交付形态，CDS 只是 JVM 类元数据复用。**用 CDS 实验代替 Native 验收 = 验错了对象**（22 章 P-AOT11）。

**构建期 Condition 的账**（22 章 P-AOT05 + Level 5）：AOT 在构建期评估 `@ConditionalOnProperty` 等并裁剪对象图——**运行时只改属性值，不能创建构建期没生成的 Bean**。所以 AOT 路线上"配置开关"分两类：

```text
影响 Bean 图存在性的开关（@ConditionalOnProperty 等）→ 改配置 = 重新构建
不影响 Bean 图的值（超时/限流/路由/灰度比例）→ 运行时随意改
```

**AOT 生成物审计**（22 章 P-AOT09）：生成 source/hints 进 CI 差异比对——依赖升级悄悄改变对象图时，构建产物 diff 是唯一早期告警（"升级后 Native 突然失败"的预防）。

**Native 交付验收 ≠ 编译成功**（22 章 Level 3/6 + P-AOT02）：

```text
compile success ≠ start success ≠ Ready success ≠ DB/MQ success ≠ Trace/Metrics success ≠ graceful shutdown success
```

7.4 的结论不变：AOT 按启动要求与动态能力依赖选型；22 章补充的验收纪律——**Native Smoke 必须覆盖 Ready、HTTP、DB/MQ、观测、SIGTERM 五项**，且保留 JVM 回滚产物（P-AOT12：首次 Native 上线必须有旧 JVM image 可回滚）。

---

# Level 8 优雅停机协议：打烊不能砍单（实测）

> 版本边界：协议顺序来自 24 章 04/14 章（Boot 4.1 基线文档语义）；**机制行为为 3.3.5 本机实测**（demo16，2026-08-06）；`shutdown-timeout` 等 4.1 能力与 3.3.5 差异已逐一标注。

## 8.1 Why：为什么"直接 kill"会死

**徒弟**：

> SIGTERM 直接杀进程，平台会自动重启，重启后客户端重试就好。

**老陈**：

> 重试能恢复的前提是你设计了幂等、重试和持久化。强退会中断：在途 HTTP 响应、数据库事务、MQ Ack/Offset、Outbox Relay、Executor 任务、Trace/Metrics Flush（24 章 14 章 Level 3 文档语义）。**停机的正确目标不是"快死"，是"把在途订单送完再打烊"**。

## 8.2 实测：immediate vs graceful（3.3.5，demo16.GracefulShutdownApp）

完整代码：`experiments/code/src/demo16/GracefulShutdownApp.java`；慢接口 sleep 6s，SIGTERM 时请求已跑 1s（剩余 ~5s）。

| 场景 | SIGTERM 后新请求 | 在途 6s 请求 | 进程退出耗时 | 实测依据 |
| ---- | ---- | ---- | ---- | ---- |
| immediate（默认） | 000 连接失败 | **被中断**（curl 退出码 52 空回复） | ~1s | 本机实测 |
| graceful | 000 连接失败（拒新） | **完成**（slow-done，SIGTERM 后 +5010ms 正好跑完剩余 5s） | +5229ms（排空完才退） | 本机实测 |

**机制**（3.3.5 Implementation）：`server.shutdown=graceful` → TomcatWebServer 包装 GracefulShutdown：SIGTERM → connector.pause()（不接新连接）→ 等在途请求归零 → 才走 Context 关闭。而 immediate 模式 SIGTERM 后 Tomcat 立即停止 → 在途请求被掐断。

## 8.3 停机协议全序：先拒新、再排空、后关资源

```text
SIGTERM
  → Readiness = REFUSING_TRAFFIC（平台摘流）
  → WebServer 不接新请求
  → 排空在途 HTTP T
  → 停止 MQ Consumer / Scheduler / Executor 产生新任务
  → 完成事务 / Ack / Outbox
  → Flush 观测
  → 关闭 Client / Pool / Context
  → 有界超时兜底 → 强制退出 + 恢复模型
```

**顺序为什么不能乱**（24 章 14 章 Level 3 文档语义）：
- **先关连接池再排请求 = 正在排空的请求全部饿死**——关资源必须在排空之后；
- **只关 WebServer ≠ 停止所有生产者**：MQ、定时任务、Executor、异步事件仍在产生新工作——"停止产生新任务"是独立一步，最容易被漏；
- **超时必须有界**（P-BS09）：某个 T 卡在 DB lock/外部调用时，无限等待 = 实例永远 Terminating，平台重启风暴。

## 8.4 分层停机：每层自己的排空语义（24 章 14 章 Level 4，文档语义）

| 层 | 排空什么 | 漏了会怎样 |
| ---- | ---- | ---- |
| HTTP | 在途请求完成、拒新 | 响应中断（8.2 immediate 实测即此场景） |
| MQ | Consumer 完成当前处理、Ack 落盘 | Ack 未确认 → 消息重投（幂等兜底） |
| Executor | 已提交任务执行完、拒绝新提交 | 任务中断、数据半写 |
| 事务 | 在途事务提交/回滚完成 | 半提交；AFTER_COMMIT 同步回调未执行 → 事件缺失（05 篇机制） |

**强退后的兜底不是"重试"，是恢复模型**：幂等（消息重投可安全再消费）、Outbox 对账（02/05 篇机制）、最终一致——停机协议和业务一致性是同一套设计。

## 8.5 版本差异：3.3.5 与 4.1 的"有界排空"（字节码实证）

| 能力 | 3.3.5（本机实证） | Boot 4.1（24 章文档语义） |
| ---- | ---- | ---- |
| server.shutdown=graceful | ✅ 生效（8.2 实测） | ✅ |
| server.shutdown-timeout | ❌ **不存在**：属性被宽松绑定静默忽略（实测 100ms/2s 均无效） | ✅ 有界排空 |
| 排空等待 | **无限等待**：Tomcat GracefulShutdown 用 `CountDownLatch.await()` 无参等待（字节码实证） | 有界，超时强制退出 |
| spring.lifecycle.timeout-per-shutdown-phase | 字段存在（6.1.14 DefaultLifecycleProcessor 实证），SmartLifecycle 停止阶段超时（属性语义待实测） | ✅ |

**生产含义**：3.3.5 上配了 graceful 但没有 4.1 的 timeout 属性兜底——**必须在应用自己层做有界排空**（Consumer/Executor 关闭前先拒新再 wait 有限时间），或升级到有 timeout 的版本。8.2 实测的"+5229ms 排空完成才退出"就是无界等待的温和形态：请求永远不结束，进程永远不退出。

## 8.6 停机检查单（并入急诊检查单）

1. 停机模式是什么？（`server.shutdown` 配置 + 实测验证，不是默认）
2. SIGTERM 后在途请求：完成还是中断？（8.2 复现法：慢接口 + kill -TERM）
3. SIGTERM 后新请求：被拒还是继续接入？
4. 生产者全停了吗？（MQ Consumer / Scheduler / Executor / 异步事件）
5. 排空有界吗？（3.3.5 无 shutdown-timeout → 应用层兜底）
6. 事务/Outbox 在停机期间的一致性？（对账 + 幂等）
7. 观测 Flush 完成？（Trace/Metrics 不丢）
8. 回滚路径？（新版本 Ready 失败 → 切回旧版本镜像，蓝绿切流）

---

# 排查方法论：四步法（Transfer）

07 篇不只是"一次事故的复盘"，它沉淀的方法可以迁移到任何"机制类系统"的故障排查：

## 第一步：症状拆解（不诊断）

把现场信息拆成最小症状，每条症状只做**机制层联想**（哪个机制负责这块行为）。禁止在这个阶段猜根因——猜根因会让你只验证"你猜的那个"，忽略其他候选（本案例值班同学的教训）。

## 第二步：候选机制排序

按**先验概率 + 症状强度**排序。本案例：S2（无报错）+ S5（无 rollback 记录）把概率集中在"拦截器没介入"，而非"回滚失败"。

## 第三步：检查单逐项排除（先便宜后贵）

每项检查 = 一个指纹 + 一个"排除了什么"的判定。检查顺序按**代价 × 区分度**：看类名（免费，区分度极高）→ 反射数 advisor（免费）→ 打点（要改代码，区分度高）→ 读代码（免费但需要知道看哪里）。

**关键纪律**：检查单输出必须如实记录，**没有指纹支撑的结论不算结论**。

## 第四步：手术 + ICU

根因确认后才动代码（方案对比决策卡），修复后必须"复现脚本回归 + 指纹断言"双验证。

## 迁移到其他领域

这套方法不依赖 Spring：

- **缓存失效排查**：症状（数据过期不刷新）→ 候选（TTL/Key 设计/双写时序）→ 指纹（缓存命中率、key 内容、写入时间戳）→ 检查单排除。
- **分布式系统**：症状（响应超时）→ 候选（网络/线程池/依赖慢/锁竞争）→ 指纹（耗时分布、线程 dump、依赖耗时）→ 逐项排除。
- **通用原则**：**机制先于经验**（知道哪层负责什么，才知道该测什么）；**指纹先于猜测**（用可验证的指标剪枝候选）；**排除法先于改代码**（不带着猜测动生产）。

---

# 00-06 机制收束总图

一张表收束整个系列：每篇的核心机制 → 它的静默失效形态 → 对应的检查指纹（全部已在对应篇目实测）：

| 篇 | 核心机制 | 静默失效形态（实测） | 急诊指纹（怎么验） | 实验 |
| -- | ---- | ---- | ---- | -- |
| 00 | 四层创建链/BPP | bean 未按预期创建/属性未注入 | bean 类名、isAopProxy（06 篇指纹复用） | demo01（00 篇） |
| 01 | 配置体系 | 配置来源被覆盖、优先级错 | PropertySource 打印、环境对照 | demo05×3 |
| 02 | 事件机制 | 监听器未触发、时序错 | 启动日志、事件打点 | demo08×5、demo09 |
| 03 | 自动装配 | 条件不满足 → bean 缺失/被替代 | 条件评估报告（自动配置报告） | demo10×6 |
| 04 | Web 请求链路 | 请求链路/启动序列与预期不符 | RunTraceApp/WebTraceApp 双栈实测 | demo11×4 |
| 05 | 事务 | 自调用失效、检查异常不回滚、代理字段 null | 事务激活打点、afterCompletion、count 残留 | demo12×4 |
| 06 | AOP | 无 aspectjweaver 用户切面失效、private/static/final 注解失效 | advisor 数、代理类名、可见性检查 | demo13×8 |
| 07 | 排查方法论 | 双重失效叠加（本篇事故）；Level 8：停机中断在途请求 | 检查单 7 项 + 停机检查单 8 项 | demo14、demo15×2、demo16、demo18、demo19×2 |

**收束观点**：本系列全部实验（demo01–demo19，58 个可运行入口）共同构成一个"指纹库"。遇到任何静默失效，先翻指纹库，再动生产。

---

# 面试自查表

| 问题 | 答案（本系列实测支撑） |
| ---- | ---- |
| @Transactional 不生效，你的排查顺序？ | 检查单：代理类名 → 事务管理器 → advisor 数 → 事务激活打点（分水岭）→ 同步回调佐证 → 可见性/调用路径 → 回滚规则 |
| private 方法上的 @Transactional 为什么永远失效？ | CGLIB 子类无法覆写 private（VisibilityApp 实测 private 连表达式匹配都不通过）；注解被读取（allowPublicMethodsOnly=false，javap 实证）但拦截器永远够不着 |
| protected/包可见方法上的 @Transactional 呢？ | 6.1.14 + CGLIB + 外部经代理调用时生效（TxVisibilityApp 实测 count=0 回滚）；JDK 代理模式下不生效；文档建议 public 是为跨模式一致 |
| 自调用为什么失效？修复方式？ | this.xxx() 走目标对象不经过代理（SelfInvocationApp 实测）；修复：注解上移 public 入口 / @Lazy 自注入 / TransactionTemplate / 拆 bean |
| 无 aspectjweaver 时自定义切面会怎样？ | 用户 Advisor 被 InfrastructureAdvisorAutoProxyCreator 的 role 过滤静默失效（ProxyKindApp 跑法 3 + javap 实证）；框架事务 advisor 是 role=2 不受影响 |
| 为什么"本地好线上坏"？ | 环境差异（配置/代理模式/依赖）或覆盖盲区；用检查单逐项排除，不靠猜 |
| 怎么预防静默失效类故障？ | 评审检查点（注解只标 public 外部入口）、测试覆盖（代理入口 + 失败路径 + 事务激活断言）、可观测性（激活打点/对账）、故障演练 |
| 发布/回滚慢（如 10 分钟），怎么优化？ | 先测指纹（StartupTimerApp 计时 / ApplicationStartup 步骤树 / 启动序列）再改：发布架构层（蓝绿切流=回滚不重启、探针节奏）→ 应用层（扫描裁剪、排除自动配置、懒初始化）→ JVM 层（AppCDS）；AOT 按启动要求选型 |
| 懒初始化（lazy-initialization）的 trade-off？ | 实测：启动 2.2s→1.6s，但首次访问触发初始化 +507ms（首请求慢、初始化错误延迟暴露）；全局开关改变行为语义（文档语义），有损的就绪检查可中和 |
| Spring AOT 和 GraalVM Native 的区别？ | Spring AOT = 构建期生成 Bean 定义/hint/代理（仍在 JVM 跑，启动变快）；Native Image = AOT 编译成原生可执行文件（毫秒级启动、低内存，但构建慢、动态能力受限）——官方文档语义，未实测 |
| 优雅停机该不该开？ | 该（在途请求一致性）；但 3.3.5 的 graceful 是无限等待排空（字节码实证 + demo16 实测），必须有应用层有界兜底；`server.shutdown-timeout` 是 4.1 能力 |
| SIGTERM 后新请求为什么 000/连接失败？ | graceful/immediate 都会先停接收新连接（demo16 实测两模式均 000）——差异只在"在途请求完成还是中断"（graceful 完成 +5s、immediate 中断 curl 52） |
| refresh 失败后容器还能用吗？ | 不能：自动销毁已创建单例、isActive=false、getBean 抛 IllegalStateException、不支持二次 refresh（demo15.RefreshFailApp 实测）；恢复 = new Context |
| @MockBean 和 @MockitoBean 什么关系？ | @MockBean 是 Boot 注解（3.3.5 用）；@MockitoBean 是 **Framework 6.2+** 的注解（jar 实证：6.1.14 无、6.2.8 有），Boot 4.1（Framework 7.x）用 @MockitoBean |

---

# 版本勘误表

| 常见说法 | 本系列实测结论 |
| ---- | ---- |
| "@Transactional 必须标 public 方法" | 不准确：6.1.14 + CGLIB + 外部经代理调用时 protected/包可见生效（TxVisibilityApp）；"只标 public"是跨模式/跨版本的稳妥建议（JDK 代理模式下非 public 不生效） |
| "事务不生效 = 忘加 @EnableTransactionManagement" | 本系列语境（@SpringBootApplication 自带）不成立：装配链检查（TriageApp 检查 2/3）实证管理器与 advisor 均在；失效多发生在代理路径层 |
| "重构搬注解不会影响行为" | 会：注解跟着代码搬进 private helper = 双重静默失效（本文事故 + TriageApp 实测） |
| "加日志就能发现静默失效" | 打点位置决定一切：打业务状态看不到事务问题；必须打事务状态指纹（isActualTransactionActive / afterCompletion） |
| "server.shutdown-timeout 配了就能有界排空" | 3.3.5 上不存在该属性（Shutdown 枚举只有 GRACEFUL/IMMEDIATE，javap 实证；100ms/2s 实测均被静默忽略）——3.3.5 的 graceful 排空无限等待（字节码实证）；该属性是 3.4+/4.1 能力 |
| "循环依赖裸容器和 Boot 一样" | 不一样：裸 Framework 默认允许（demo04 实测），Boot 3.3.5 默认禁止（demo15.BootCircularApp 实测 BeanCurrentlyInCreationException）——"升级 Boot 2.6+ 字段环突然爆炸"的机制实证 |

---

# 生产决策卡（4 张）

## 决策卡 1：事务边界放哪（Level 5 方案表）

- **Decision**：`@Transactional` 标在 public 外部入口方法；helper 拆分保留但无注解。
- **Reason**：代理只拦"外部经代理的调用"（05/06 篇实测）；入口方法天然满足 public + 外部调用。
- **Alternative**：@Lazy 自注入（边界精确到 helper）、TransactionTemplate（不依赖代理）、拆 bean（结构清晰）。
- **Trade-off**：入口边界粗（helper 复用需小心调用面）；替代方案各自付出"自注入心智/代码混写/类成本"。
- **Validation**：修复后重跑检查 4/5/6 指纹（事务激活=true、回滚干净）。

## 决策卡 2：失效排查手段选择

- **Decision**：先免费高区分度（类名、advisor 数），再打点，最后读代码定位。
- **Reason**：排除法剪枝最快（06 篇：一个类名排除"没代理"，advisor 数排除"装配链断"）。
- **Alternative**：直接加日志重发（值班同学首错：验证的是业务代码不是机制）。
- **Trade-off**：检查单需要机制知识储备（00-06 实测）；换来的是生产环境零试错。
- **Validation**：TriageApp 6 项输出顺序即检查单执行序。

## 决策卡 3：预防投入分配

- **Decision**：评审检查点（免费）> 失败路径用例（低）> 可观测性打点（低）> 故障演练（周期性）。
- **Reason**：本事故根因（注解位置+调用路径）评审肉眼可见；演练验证其余三层真的有效。
- **Alternative**：只靠监控告警（发现不了"静默"故障——静默的定义就是没告警）。
- **Trade-off**：评审依赖团队机制知识；演练消耗周期时间。
- **Validation**：演练能抓出注入的失效 = 预防体系有效（6.5）。

## 决策卡 4：慢发布优化路线选型（Level 7）

- **Decision**：先发布架构层（回滚切流 + 探针/批次调优），再应用层（扫描裁剪 / 懒初始化 / 排除自动配置），再 JVM 层（AppCDS）；启动要求苛刻才评估 AOT 路线。
- **Reason**：回滚慢的正解是"回滚不重启"（蓝绿/金丝雀流量切换）；启动优化按"杠杆/风险比"排序，先便宜后贵（与排查检查单同一条方法论）；AOT 是最后手段不是第一手段。
- **Alternative**：直接上 GraalVM Native（启动毫秒级），代价是构建数分钟级、动态能力受限、调试与工具链重构（7.4 表格）。
- **Trade-off**：无 AOT 方案保留完整 JVM 生态、零构建成本，但启动收益有限且需逐层测量；AOT 路线收益大、成本与约束也大。
- **Validation**：StartupTimerApp 三跑法本机测量 + 生产就绪时间/发布耗时 KPI 持续监控；AOT 方案必须先在测试环境跑通 RuntimeHints 全套（反射/代理/资源），再谈上线。

## 决策卡 5：停机模式选型（Level 8）

- **Decision**：`server.shutdown=graceful` + 应用层有界排空（Consumer/Executor 先拒新再限时等待）；3.3.5 无 `shutdown-timeout`，有界兜底自己做；升级 4.x 后可交给 `server.shutdown-timeout`。
- **Reason**：8.2 实测——immediate 中断在途请求（curl 52），graceful 排空完成；但 graceful 在 3.3.5 是**无限等待**（字节码实证），没有有界超时就有"永远 Terminating"风险。
- **Alternative**：immediate + 强依赖客户端重试/幂等（能接受中断的业务）；4.1 的 shutdown-timeout（升级路线，文档语义）。
- **Trade-off**：graceful 牺牲停机时长（等排空）换在途一致性；有界兜底要自己在应用层实现（停机预算必须 [固定环境实测]）。
- **Validation**：8.6 检查单 8 项 + demo16 复现脚本（immediate/graceful 两场景输出对照）。

---

# 系列索引

```
00 容器如何创建对象（已重写：四层创建链 + 15 个实测 demo）
01 框架整合 + 配置体系（已重写：6 Level，配置体系独立 Level 4，6 个实测 demo：demo01.ServiceLoaderApp + demo05×3 + demo06 + demo07）
02 事件机制与容器通信（已重写：6 Level，5 个实测 demo：demo08×5 + demo09 启动事件全景）
03 自动装配深挖（已重写：6 Level，6 个实测 demo：demo10×6：类条件 ASM/评估报告代码访问/排除/排序/覆盖/属性条件）
04 Web 请求链路与运行时刻（已重写：6 Level，4 个实测 demo：demo11.RunTraceApp/WebTraceApp/ActuatorApp/WebFluxApp（双跑法），run 反编译 16 条序列 + refresh 12 步 + 内嵌容器 + 请求链路 + Servlet/WebFlux 双栈实测 + 探针 404→200 实测）
05 事务与数据层（已完成：6 Level，4 个实测 demo：demo12.ds.DataSourceApp / tx.TxBasicsApp / tx.PropagationApp / tx.SelfInvocationApp）
06 横切面与 AOP（已完成：6 Level，8 个实测 demo：demo13.aspect.ProxyKindApp（JDK/CGLIB 双分支 + creator 双分支行为差异）/ AdviceOrderApp / PointcutApp / AspectOrderApp / ProxyInternalsApp / UnwrapApp / VisibilityApp / TxVisibilityApp；重大发现：无 aspectjweaver 时用户切面被 InfrastructureAdvisorAutoProxyCreator 的 role 过滤静默失效）
07 生产实践（本篇：急诊室比喻，一次静默失效事故的排查全程，检查单 7 项 + 00-06 机制收束总图；Level 7 慢发布：指纹测量 + 三层优化 + AOT 选型（22 章交叉补充：AOT/Native/CDS 三技术区分 + 构建期 Condition + 生成物审计）；Level 8 优雅停机协议：demo16 实测 immediate/graceful 对照 + 3.3.5 无 shutdown-timeout 字节码实证 + 停机检查单 8 项 + 决策卡 5；实测 demo14.triage.TriageApp + demo14.publish.StartupTimerApp + demo15×2（Boot 循环依赖/refresh 失败）+ demo16.GracefulShutdownApp）
```

# 实验复现

```bash
cd knowledge/springboot/experiments/code
./build.sh && ./run.sh demo14.triage.TriageApp   # 事故复现：检查单 6 项输出 + 双重失效残留对比
./run.sh demo12.tx.SelfInvocationApp            # 回看：自调用失效与 @Lazy 修复（05 篇）
./run.sh demo12.tx.TxBasicsApp                  # 回看：回滚规则与事务激活打点（05 篇）
./run.sh demo13.aspect.TxVisibilityApp          # 回看：@Transactional 可见性实测（06 篇）
./run.sh demo13.aspect.ProxyKindApp             # 回看：三跑法 + advisor 收集（06 篇）
./run.sh demo14.publish.StartupTimerApp          # 启动计时（跑法1 默认）
java -Dspring.main.lazy-initialization=true -cp "out:$(find lib -name '*.jar' | tr '\n' ':')" demo14.publish.StartupTimerApp   # 跑法2 lazy 对比
java -Dspring.main.startup=timeline -cp "out:$(find lib -name '*.jar' | tr '\n' ':')" demo14.publish.StartupTimerApp          # 跑法3 步骤树
./run.sh demo15.BootCircularApp             # Boot 3.3.5 默认禁循环依赖（与 demo04 对照）
./run.sh demo15.RefreshFailApp              # refresh 失败：自清理 + 容器拒绝继续使用
./run.sh demo16.GracefulShutdownApp --server.port=18080                         # 停机模式1 immediate
./run.sh demo16.GracefulShutdownApp --server.port=18080 --server.shutdown=graceful   # 停机模式2 graceful（排空完成才退）
./run.sh demo18.AotGenerationApp          # Spring AOT 引擎端到端：生成 5 个源码文件 + CGLIB 代理字节码 + 170 个 RuntimeHints
./run.sh demo19.buffered.StartupProfilerApp   # 启动慢排查 1-3：actuator startup 端点 + BufferingApplicationStartup + /actuator/startup 耗时快照（353 步，慢 bean 精确定位）
./run.sh demo19.jfr.JfrStartupApp             # 启动慢排查 4：FlightRecorderApplicationStartup 深度剖析 → out/startup-recording.jfr
export JAVA_HOME=.../azul-21.0.11/Contents/Home && export PATH="$JAVA_HOME/bin:$PATH"
jfr print --events org.springframework.core.metrics.jfr.FlightRecorderStartupEvent out/startup-recording.jfr   # JFR 步骤事件转文本（含 stackTrace/tags）
```

TriageApp 的检查单输出已固化在源文件头部注释，与本文引用一致。

---

# ✅ Final Review Checklist

- [ ] 是否解释了为什么存在？（机制知识是正向的，故障排查是反向的；静默失效没有报错可依赖，只有机制指纹可用——需要"翻译层"）
- [ ] 是否说明旧方案为什么失败？（值班同学的"看日志/加日志重发"验证的是业务代码不是机制；分诊-检查-手术流程保证生产环境零试错）
- [ ] 是否形成完整因果链？（症状 S1-S5 → 分诊收敛候选 A-D → 检查单 7 项逐项排除 → 检查 4 分水岭锁定"拦截器没介入" → 检查 6 双重失效根因 → 方案决策 → ICU 验证 → 预防医学；Level 7：慢发布三个放大器 → 指纹测量 → 三层优化 → AOT 选型）
- [ ] 是否区分规范和实现？（回滚规则/事务语义为 Specification；JdbcTransactionManager 类名、advisor 数=3、CGLIB 类名、激活打点结果为本机 6.1.14 Implementation）
- [ ] 是否区分语义变化与代码组织变化？（本事故的"重构"= 代码组织变化，但注解位置变化导致行为变化——正是"组织变化引发语义变化"的实例）
- [ ] 代码实例是否全部实测？（demo14×2 输出原样引用、可复跑：TriageApp 检查单 6 项 + StartupTimerApp 三跑法计时；AOT/GraalVM/AppCDS/K8s 探针为官方文档语义，已标注待实测；回看实验全部锚定 00-06 已实测输出）
- [ ] 是否包含 Trade-off？（修复方案 A/B/C/D 对比；排查手段选择；预防投入分配；慢发布三路线对比；停机模式选型——5 张决策卡）
- [ ] 是否能指导生产决策？（检查单可执行、决策卡 5 张、评审检查点 4 条、测试/可观测性/演练落地、慢发布优化路线、停机协议检查单 8 项）
- [ ] 是否存在未经证明的数字？（无编造 benchmark/时间线/事故细节；事故场景显式标注"教学推演"；StartupTimerApp 计时明确标注"本机机制演示非生产 benchmark"；AOT/GraalVM/AppCDS 效果全部标注官方文档语义待实测；demo16 停机时序同样仅本机机制演示）
- [ ] 是否只有一个比喻？（急诊室：分诊/检查单/手术/ICU/预防医学）是否只有一个主线角色？（一次线上故障）
- [ ] 随机抽查断言：bad 路径事务激活=false 且 count=1 残留（TriageApp 实测）、good 路径回滚（实测）、事务管理器类名 org.springframework.jdbc.support.JdbcTransactionManager（实测）、advisor 数=3（反射实测）、bean 为 CGLIB 代理（类名实测）、registerSynchronization 无事务抛 IllegalStateException（demo14 开发过程实测）、lazy 开关启动 2.2s→1.6s 且首次 getBean +507ms（StartupTimerApp 实测）、步骤树大头 config-classes.parse=376ms（实测）、StartupEndpoint 类存在于 3.3.5 jar（jar 实证）、Boot 3.3.5 默认禁循环依赖（demo15.BootCircularApp 实测）、refresh 失败自动销毁已创建单例且 getBean 抛 has not been refreshed yet（demo15.RefreshFailApp 实测）、immediate 中断在途请求 curl 52 / graceful 排空完成 +5010ms（demo16 实测）、3.3.5 Shutdown 枚举无 timeout（javap 实证）、doClose 顺序 ContextClosedEvent→LifecycleProcessor.onClose→destroyBeans（字节码实证）、@MockitoBean 6.2+ 引入（6.1.14 无 / 6.2.8 有 jar 实证）、probes 默认 404 且 startup 开启 probes 后仍 404（实测）、AOT/Native/AppCDS 效果为文档语义（未实测，已标注）——均有证据来源。
