# 🚀 SpringBoot Web 请求链路与运行时刻（系列 04）

> 本篇文章回答两个问题：
> 1. **启动时刻**：`SpringApplication.run()` 里到底发生了什么？内嵌 Tomcat 是怎么被拉起来的？"启动完成"的定义是什么？
> 2. **运行时刻**：一次 HTTP 请求从网线到 Controller 再到 JSON 响应，完整走过了哪些环节？Actuator 的 health / readiness / liveness 分别告诉你什么？K8s 探针凭什么"敢"把一个进程视为就绪？
>
> 前置：02 篇的事件机制（启动事件全景）、03 篇的自动装配五门（DispatcherServlet 本身就是自动配置注册的）。本篇把"从 run() 到一次请求返回"这条主线一次性走完，补上 00 篇 Level 7 的运行时扩写。

---

# ⚠️ 版本与证据边界

| 维度 | 本文承诺 |
| ---- | ---- |
| 代码实证 | `knowledge/springboot/experiments/code/` 下 demo11×3（RunTraceApp / web.WebTraceApp / actuator.ActuatorApp），本机实测输出原样引用 |
| 实测环境 | macOS + JDK 21.0.11（Azul Zulu）+ spring-boot **3.3.5** + spring-web/-webmvc/-webflux **6.1.14** + tomcat-embed-core **10.1.31** + spring-boot-actuator(-autoconfigure) 3.3.5 + micrometer 1.13.6 + jackson **2.17.2** + reactor-core **3.6.11** + reactor-netty **1.1.23** + netty **4.1.114.Final**（版本均出自 spring-boot-dependencies-3.3.5.pom BOM 与 reactor-bom 2023.0.11） |
| Specification | Servlet 规范（jakarta.servlet 6.0：Filter/生命周期）、DispatcherServlet 的 MVC 处理链语义、WebFilter 链语义（响应式栈）、Reactive Streams 背压语义、health 端点组语义（readiness/liveness）、K8s 探针语义 |
| Implementation | SpringApplication.run 字节码调用序列（3.3.5 反编译，16 条）、AbstractApplicationContext.refresh 12 步（6.1.14 反编译）、WebApplicationType 探测（含"两个栈 jar 都在时 MVC 优先"实测）、ServletWebServerApplicationContext.onRefresh → createWebServer、EmbeddedTomcat 实例化时机（BPP 首个经过的 bean，实测）、Tomcat 线程名 http-nio-8080-exec-1（实测）、Netty EventLoop 线程名 reactor-http-nio-N（实测）、probes.enabled 开关行为（404→200 实测）、startup 端点默认 404 且开 probes 后仍 404（2026-08-06 实测）、refresh 失败自清理（demo15.RefreshFailApp 实测：自动销毁已创建单例 + getBean 拒绝 + 不支持二次 refresh）、conditions 端点 JSON 结构 notMatched/matched（实测）、扩展点全景时序（EnvironmentPostProcessor 位点：3.3.5 spring.factories + EnvironmentPostProcessorApplicationListener 反编译；ApplicationContextInitializer/CommandLineRunner 位点：demo11 实测） |
| 待验证 | 不同硬件/负载下 Tomcat 线程池真实吞吐（本文不承诺任何性能数字）；非 macOS 环境端口占用行为细节；Boot 2.x 的 probes 默认行为差异；去掉 web jar 后 WebApplicationType 回退 NONE（机制推导，未实测）；背压在高负载下的真实表现细节 |
| 未覆盖 | Servlet 规范内部实现细节（HTTP 协议解析）；Tomcat 连接器 NIO 源码逐行；WebFlux 的函数式路由（RouterFunction）；WebFlux 与阻塞代码混用的详细分析；Spring Security 过滤器链 |

---

# 🏷️ 关键词

SpringApplication.run | 启动事件序列 | refresh 12 步 | WebApplicationType | 内嵌容器 | EmbeddedTomcat | onRefresh | ServletWebServerInitializedEvent | DispatcherServlet | HandlerMapping | HandlerAdapter | Interceptor | HttpMessageConverter | Filter | 连接器线程 | Actuator | health | readiness | liveness | probes | conditions 端点 | K8s 探针 | 优雅停机

---

# 🗂️ 目录

- Level 1 问题的起点：run() 之后发生了什么
- Level 2 启动时刻：SpringApplication.run 全流程
- Level 3 内嵌容器：Tomcat 怎么被拉起来
- Level 4 请求链路：一次 HTTP 请求的完整旅程（含 WebFlux 另一条栈）
- Level 5 可观测性：Actuator 与 K8s 就绪语义
- Level 6 生产实践：启动慢 / 停机 / 探针 / 排查
- 全篇因果链总图
- 线上案例（3 个）
- 面试自查表
- 坑与细节（8 个）
- 版本勘误表
- 生产决策卡（3 张）
- 跨语言视角

---

# 🏭 全文唯一比喻：外卖配送

把**一个 SpringBoot 应用 = 一家外卖店**，贯穿两阶段：

- **启动时刻 = 开店流程**：老板按下 `run()`（营业开关），从"选址装修（创建上下文）"到"后厨验收（refresh）"到"正式营业（ApplicationReadyEvent）"；
- **运行时刻 = 一次配送**：顾客下单（HTTP 请求）→ 平台分单（DispatcherServlet）→ 骑手取餐（Tomcat 连接器线程）→ 后厨炒菜（Controller）→ 打包（HttpMessageConverter）→ 送达（HTTP 响应）；
- **可观测性 = 外卖平台的商家看板**：health 告诉你"店还开着吗"、readiness 告诉你"现在能不能接单"、liveness 告诉你"老板还有气吗"。

---

# Level 1 问题的起点：run() 之后发生了什么

## 为什么需要本篇

00–03 篇回答的都是"**对象怎么来**"：容器创建、配置加载、事件广播、自动装配。它们有一个共同的盲区：

> **`run()` 那一行代码之后，程序到底是怎么活起来的？**

- 01 篇讲过：`run()` 只是入口，真正干活的在 refresh()；
- 03 篇讲过：自动配置把 DispatcherServlet、Tomcat 工厂这些 Bean **定义**注册了进去；
- 但没人回答：**Tomcat 的 socket 是谁在哪个步骤打开的？一个 HTTP 请求进来后，代码执行顺序是什么？"启动完成"有没有一个准确的时间点定义？**

这就是本篇的任务：把 **启动时刻（秒级）** 和 **运行时刻（毫秒级）** 两段时间线完整走一遍。

## 两阶段地图

```
启动时刻（一次，秒级）                      运行时刻（每请求，毫秒级）
─────────────────────────                  ─────────────────────────
SpringApplication.run()                    HTTP 请求到达
  └─ prepareEnvironment                     └─ Tomcat 连接器线程接手
  └─ createApplicationContext               └─ Filter 链
  └─ prepareContext                         └─ DispatcherServlet
  └─ refresh()  ← 关键，12 步                └─ HandlerMapping 找处理器
  └─ started / callRunners / ready          └─ Interceptor 前后拦截
     （ready = 官方定义的"启动完成"）        └─ Controller 执行
                                            └─ HttpMessageConverter 写响应
                                            └─ 响应返回客户端
```

## 本篇的证据链条

三个实测 demo，一一对应两条时间线 + 一个观察窗口：

| demo | 对应 | 打点方式 |
| ---- | ---- | ---- |
| `demo11.RunTraceApp` | 启动时刻 | 事件监听器 + Initializer + BFPP + BPP + Runner 五类回调 |
| `demo11.web.WebTraceApp` | 运行时刻 | Filter / Interceptor / Controller / 响应状态码 / 线程名 |
| `demo11.actuator.ActuatorApp` | 观察窗口 | 三个探针端点 + conditions 端点 |
| `demo11.webflux.WebFluxApp` | 运行时刻（另一条栈） | 双跑法：同代码两组 classpath，实测 SERVLET / REACTIVE 分支 |

---

# Level 2 启动时刻：SpringApplication.run 全流程

## 为什么需要八个步骤

01 篇讲过 run() 的"八步"，那是**读者视角的骨架**。真正的执行序列远比八步密——因为 Spring 把"启动"拆成了可插拔的扩展点链：每个环节都留了钩子（事件、Initializer、回调），让框架和用户代码都能在正确时机介入。

旧方案（Spring 2.5 时代）的问题：启动逻辑写死在 `AbstractApplicationContext.refresh()` 里，你想在"环境准备后、上下文创建前"插一段逻辑？没有钩子，只能继承重写。**所以 Boot 把启动流程显式化、事件化**：每一步发布事件，每一步留扩展点。

## 反编译实证：run(String[]) 的完整调用序列

`javap -p -c` 反编译 `SpringApplication.class`（3.3.5）后，`run(String...)` 的字节码顺序（方法调用序列）：

```
1.  Startup 计时（new SpringApplication$Startup 记录启动耗时）
2.  shutdownHook.enableShutdownHookAddition()
    ← 打开 shutdown hook 注册开关。钩子本体是全局单例
      SpringApplicationShutdownHook（Boot 3.2+ 引入），run() 里只开开关
3.  configureHeadlessProperty（设置 java.awt.headless=true，服务器场景）
4.  getRunListeners（从 spring.factories 加载 SpringApplicationRunListener）
5.  listeners.starting（→ ApplicationStartingEvent）
6.  prepareEnvironment（加载配置源：命令行/系统属性/环境变量/application.yml）
     └─ listeners.environmentPrepared（→ ApplicationEnvironmentPreparedEvent）
7.  printBanner
8.  createApplicationContext（按 WebApplicationType 决定上下文类）
9.  context.setApplicationStartup（观测埋点，Spring 5.3 引入的 ApplicationStartup API）
10. prepareContext：
      ├─ applyInitializers（用户 Initializer 先执行 → ApplicationContextInitializedEvent）
      ├─ contextPrepared（listeners 钩子）
      ├─ bootstrapContext.close()
      ├─ 循环引用开关（allowCircularReferences 按属性）
      └─ KeepAlive 保活（注册保活线程，run() 返回后 JVM 不退出）
11. refreshContext（核心！见下）
12. afterRefresh（钩子，默认空）+ StartupInfoLogger.logStarted（启动耗时日志）
13. listeners.started（→ ApplicationStartedEvent）
14. callRunners（ApplicationRunner / CommandLineRunner）
15. listeners.ready（→ ApplicationReadyEvent）
16. return context
```

徒弟：

> 16 步，比我记的"八步"密一倍——为什么是 16 步，不是 8 步 20 步？这些步骤的顺序是随便排的吗？

老陈：

> 不随便。16 步可以归成**开店流程**（外卖店，接 Level 4 的配送体系）：备料 → 租店 → 装修 → 开业，四个阶段。

```text
备料（1-7）     Startup 计时 → shutdown 钩子开关 → headless → RunListener →
                starting → prepareEnvironment（配置源）→ banner
                为什么先备料？因为后面 prepareContext 第一步就要
                setEnvironment（把环境塞进上下文）——环境不先备好，
                上下文没东西可读（spring.main.*、server.* 都从这里绑）

租店（8-9）     createApplicationContext → setApplicationStartup
                注意：上下文类型（SERVLET/REACTIVE/NONE）不是配置决定的，
                是 classpath 探测决定的（Level 3 WebApplicationType 一节）
                ——这就是为什么"加依赖"能改变启动行为

装修（10-11）   prepareContext（Initializer 执行 + 监听器注册）→ refresh（12 步）
                装修最久，所有 Bean 都在这里造出来——后面单独拆 12 步

开业（12-16）   afterRefresh → started → callRunners → ready → return
                为什么 Runner 在 started 与 ready 之间？"开演前最后一次彩排"——
                ready 一旦发布，外界（K8s/监控）就认为应用可以对外服务了
```

> **记忆锚点**：不是背 16 步，是背四个阶段 + 每个阶段之间的扩展点。16 步里其实只有 6 处是"钩子位"（starting / environmentPrepared / contextPrepared+applyInitializers / started / callRunners / ready），其余是硬步骤——钩子位才是 Boot 把它拆这么密的真实原因：**启动流程被显式化，就是为了在每个间隙都能插代码**。

## refresh() 才是真正的"开业仪式"：12 步（6.1.14 反编译）

```
refresh()（AbstractApplicationContext，6.1.14 反编译调用序列）
 1. prepareRefresh（设置启动时间、初始化属性源）
 2. obtainFreshBeanFactory（重新解析 Bean 定义）
 3. prepareBeanFactory（注册默认 Bean：environment/systemProperties 等）
 4. postProcessBeanFactory（子类扩展点）
 5. invokeBeanFactoryPostProcessors（★ 所有 BFPP，含自动配置注册——03 篇的门 5）
 6. registerBeanPostProcessors（BPP 注册）
 7. initMessageSource（国际化）
 8. initApplicationEventMulticaster（事件广播器）
 9. onRefresh（★ Web 场景 = 内嵌容器启动点！见 Level 3）
10. registerListeners（把监听器 bean 注册进广播器）
11. finishBeanFactoryInitialization（★ 单例 bean 全部实例化）
12. finishRefresh（内部依次：resetCommonCaches 清反射/注解缓存 → initLifecycleProcessor → 发布 ContextRefreshedEvent）
```

（注：`resetCommonCaches` 是 `finishRefresh()` 内部第一步调用，不是 refresh() 的独立步骤——本序列为反编译实证的 12 个顶层方法调用。）

徒弟：

> 12 步能不能合并？比如第 5、6 步都是"注册处理器"，第 7、8 步都是"初始化基础设施"，干嘛拆这么细？

老陈：

> 不能合并——12 步是**装修工序**，每一步的先后由"上一步产出什么、下一步需要什么"决定。三步记法：

```text
图纸先行（1-6）：prepareRefresh → obtainFreshBeanFactory → prepareBeanFactory
               → postProcessBeanFactory → invokeBeanFactoryPostProcessors
               → registerBeanPostProcessors
  为什么 BFPP 必须先于 BPP？BFPP 改的是 Bean 定义（图纸），BPP 处理的是
  Bean 实例（房子）——图纸没定，先请监理（BPP）进来没用。
  为什么第 3 步就得注册 environment/systemProperties 这些默认 Bean？
  后面所有处理器都要用它们。

布管线（7-9）：initMessageSource → initApplicationEventMulticaster → onRefresh
  为什么广播器必须在 registerListeners 之前？没广播器，监听器注册给谁？
  为什么 onRefresh（Web 场景=创建内嵌容器）卡在这里？
  因为它要的 WebServer 工厂 Bean 必须已定义（前 6 步完成）。
  注意：onRefresh 只"创建"WebServer 对象，不发布 ServletWebServerInitializedEvent——
  该事件在 finishRefresh（第 12 步）的 WebServerStartStopLifecycle（SmartLifecycle）
  start() 里才发布（反编译实证，见 Level 3）——所以 @EventListener 能收到它。

完工（10-12）：registerListeners → finishBeanFactoryInitialization
               → finishRefresh
  为什么单例实例化压到最后？因为它是"完工验收"——前面所有步骤
  都在为它铺路：图纸、处理器、广播器、服务器，全是它的前置。
  为什么 registerListeners 在 finishBeanFactoryInitialization 之前？
  监听器要提前注册好，单例创建过程中的事件才不丢（02 篇早期事件）。
```

> **记忆锚点**：三阶段 = 图纸（定义阶段）→ 管线（基础设施）→ 完工（实例化）。任何一步"为什么在这个位置"的问题，都答"它前面的步骤产出了它需要的东西"——顺序即依赖链，不是风格问题。

## refresh 失败会发生什么：半初始化 Context 的自清理（实测）

完整代码：`springboot-demo/src/main/java/demo15/RefreshFailApp.java`（6.1.14 实测，2026-08-06）。

场景：单例实例化阶段（第 11 步 finishBeanFactoryInitialization）一个 Bean 的 `@PostConstruct` 抛异常。真实输出：

```text
[创建] first(失败前) 构造完成
[创建] Boom 进入 @PostConstruct
[销毁] first(失败前)          ← refresh 失败分支自动销毁已创建单例！
[1] refresh 失败，根因: IllegalStateException : Boom: 初始化爆炸
[2] 失败后 ctx.isActive() = false
[3] 失败后 getBean(first) 抛: IllegalStateException : ... has not been refreshed yet
[4] 失败后 getBean(last) 抛: IllegalStateException : ... has not been refreshed yet
[5] close() 完成（无异常）
[6] close 后再 refresh 抛: IllegalStateException : GenericApplicationContext does not
    support multiple refresh attempts: just call 'refresh' once
```

**因果链**（Framework 6.1.14 Implementation）：refresh 第 11 步任何单例创建失败 → 异常向上抛 → refresh 的 catch 分支执行 `destroyBeans()`（**连失败前已创建的单例也销毁**）并 `cancelRefresh`（active=false）→ 此后容器所有 getBean 直接抛 `IllegalStateException`，**不允许继续使用**；想恢复只有 new 一个新容器（不支持二次 refresh）。

**生产含义**（对照 24 章 04 章 P-BS05/P-BS08，Boot 4.1 基线文档语义，机制一致）：
- **失败后不要继续用这个 Context**——不是"可能拿到残缺对象"，是容器层面直接拒绝（实测 [3][4]）；
- `WebServerFactoryCustomizer` / Runner 里做业务初始化（连库、预热、远程调用）的失败 = 启动失败，但失败现场难查（P-BS05）；启动任务应该进 `ApplicationRunner`，且失败要区分"可重试的依赖不可用"与"永久配置错误"——退出码语义决定发布系统是重试还是回滚（FailureAnalyzer 只增强可读性，不替代异常证据，24 章 04 章文档语义）。
- Boot 失败时的 FailureAnalysis 结构（Description/Action/Cause）是给运维的指引，不是给程序继续运行的信号。

## RunTraceApp 实测：五类回调的真实先后

`demo11.RunTraceApp` 同时挂载了事件监听器、Initializer、BFPP、BPP、Runner，真实输出（本机，classpath 22 个 jar）：

```
[事件] ApplicationStartingEvent
[事件] ApplicationEnvironmentPreparedEvent
[阶段] prepareContext：Initializer 执行（applyInitializers）
[事件] ApplicationContextInitializedEvent
[事件] ApplicationPreparedEvent
[阶段] refresh 第 5 步 invokeBeanFactoryPostProcessors：BFPP 执行（此时已注册 Bean 定义数 = 361）
[阶段] registerBeanPostProcessors 之后，第一个经过 BPP 的 bean =
       org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryConfiguration$EmbeddedTomcat
       ← onRefresh（第 9 步）触发 Web 服务器创建，强制实例化工厂单例
[事件] ServletWebServerInitializedEvent  ← finishRefresh 内 WebServerStartStopLifecycle.start() 发布（见下方"重要"段）
[事件] ContextRefreshedEvent（finishRefresh 发布）
[事件] ApplicationStartedEvent
[Runner] ApplicationRunner 执行（callRunners）
[事件] ApplicationReadyEvent
[阶段] run() 返回——上下文类 = AnnotationConfigServletWebServerApplicationContext（SERVLET）
```

（真实输出中 `ApplicationStartedEvent` 之后、`ApplicationReadyEvent` 之后、`ctx.close()` 时各有一行 `AvailabilityChangeEvent`（可用性状态变更广播，共 3 处），此处省略噪声行；五类回调的顺序与上表完全一致。Bean 定义数随 classpath 变化：22 jar 时 280，38 jar（加 webflux/netty/jsr310）后 361——任何固定数字都只是本机快照，机制不变。）

三个关键证据：

1. **事件顺序与反编译序列完全一致**：`ApplicationReadyEvent` 永远在最后，Runner 在 `started` 与 `ready` 之间——这就是官方定义的"启动完成"时刻；
2. **BFPP 时已有 361 个 Bean 定义**：全部来自 03 篇的门 5（自动配置注册）——数字随 classpath 变化（本机 38 jar 下的快照），**机制不变**；
3. **第一个经过 BPP 的 bean 是 EmbeddedTomcat**：它不是最后才启动的，而是**在 refresh 第 9 步 onRefresh 就被强制实例化**——这直接引向 Level 3。

## 扩展点全景：一条时间轴上的所有钩子

RunTraceApp 只挂了五类回调，是"点"。这张表把 Spring 层（00 篇）与 Boot 层（本篇）的扩展点全部放到同一条时间轴上，按启动 → 运行 → 关闭三阶段分组——**每一个扩展点的能力边界都由它的触发时机决定**（改动时机决定能力边界，00 篇 2.3）：

```text
run()
├─ EnvironmentPostProcessor               ← 容器创建前（prepareEnvironment，3.3.5 反编译实证）
├─ ApplicationContextInitializer          ← 容器创建后、refresh 前（prepareContext，demo11 实测）
├─ refresh() 12 步
│   ├─ ⑤ invokeBeanFactoryPostProcessors：BeanDefinitionRegistryPostProcessor → BeanFactoryPostProcessor
│   ├─ ⑥ registerBeanPostProcessors：BeanPostProcessor 注册（真正调用在 Bean 创建时）
│   └─ ⑪ finishBeanFactoryInitialization：每建一个 Bean 依次过
│        InstantiationAwareBeanPostProcessor（实例化/填充）
│        → BeanPostProcessor.before + ApplicationContextAware（第二拨，链首）
│        → InitializingBean / @PostConstruct / initMethod
│        → BeanPostProcessor.after（★ AOP 代理）
│        └─ 末尾：SmartInitializingSingleton（@EventListener 靠它注册，02 篇）
├─ started → callRunners（CommandLineRunner/ApplicationRunner，demo11 实测）→ ready
├─ 运行期：ApplicationListener（事件，02 篇）｜请求链路扩展点（Level 4）
└─ 关闭：ContextClosedEvent → SmartLifecycle.stop → @PreDestroy/DisposableBean/destroyMethod
    （00 篇 2.3 doClose 反编译实证）
```

### 启动阶段

| 扩展点 | 触发时机 | 拿到什么 | 典型用途 |
| ---- | ---- | ---- | ---- |
| EnvironmentPostProcessor | run 的 prepareEnvironment（容器创建前） | ConfigurableEnvironment | 改 PropertySource、注入默认值；框架自带 ConfigDataEnvironmentPostProcessor（01 篇） |
| ApplicationContextInitializer | prepareContext 第 1 步（容器已建、未 refresh） | ConfigurableApplicationContext | 注册监听器、替换广播器（02 篇） |
| BeanDefinitionRegistryPostProcessor | refresh ⑤ 第一批 | BeanDefinitionRegistry | 增删改 BeanDefinition；ConfigurationClassPostProcessor 靠它拿"注解→定义"解析权（00 篇 2.3） |
| BeanFactoryPostProcessor | refresh ⑤ 第二批 | ConfigurableListableBeanFactory | 改元数据（占位符替换） |
| BeanPostProcessor（注册） | refresh ⑥ registerBeanPostProcessors | — | 只完成注册，调用在 Bean 创建时（见下） |
| InstantiationAwareBeanPostProcessor | 每个 Bean：实例化前/后 + populateBean | Bean 实例 | 拦截实例化、自定义属性注入（01 篇 DubboRefBeanPostProcessor 同构） |
| BeanPostProcessor（调用） | 每个 Bean：initializeBean 的 before/after | Bean 实例 | 注入、包装；after 是 AOP 代理站（00 篇 2.2） |
| ApplicationContextAware | 每个 Bean：initializeBean 内（BPP.before 链首） | ApplicationContext | 拿容器能力（00 篇 2.6 实测） |
| InitializingBean（@PostConstruct/initMethod 同族） | 每个 Bean：BPP.before 之后 | — | 初始化回调（00 篇 2.2） |
| SmartInitializingSingleton | refresh ⑪ 末尾（所有单例建完） | — | 一次性后处理；@EventListener 注册（02 篇 javap 实证） |
| CommandLineRunner / ApplicationRunner | started 之后、ready 之前（callRunners） | ApplicationArguments | 启动收尾任务（demo09/demo11 实测） |

### 运行阶段

| 扩展点 | 触发时机 | 拿到什么 | 典型用途 |
| ---- | ---- | ---- | ---- |
| ApplicationListener（三种注册） | 任意事件发布时 | 事件对象 | 业务解耦、跨模块通知；@EventListener 收不到启动早期事件（02 篇实测） |
| Servlet Filter / HandlerInterceptor 等 | 每次 HTTP 请求链路 | 请求/响应 | 鉴权、日志、上下文传递（Level 4 WebTraceApp 实测） |

### 关闭阶段（doClose 顺序，00 篇 2.3 反编译实证）

| 扩展点 | 触发时机 | 拿到什么 | 典型用途 |
| ---- | ---- | ---- | ---- |
| ContextClosedEvent 监听器 | 关闭最先（容器仍可用） | 事件 | 最后清理（Flush 观测、外部资源释放） |
| SmartLifecycle | ② stop（按 phase 倒序） | — | 服务级停机：先拒新、停对外服务（07 篇） |
| @PreDestroy / DisposableBean / destroyMethod | ③ destroyBeans | — | 实例级收尾：线程池、连接释放 |

**三条可直接推导的生产结论**：

1. **越靠前的钩子，能动的越少**：EnvironmentPostProcessor 连容器都没有，只能动 Environment；ApplicationContextInitializer 有了容器但没有 Bean——想"注册一个 Bean 去改配置"是阶段错位，得用 BFPP；
2. **BeanPostProcessor 注册早、生效晚**：refresh ⑥ 只注册，真正调用在 ⑪ 每个 Bean 创建时——想在"第一个 Bean 创建前"做全局动作的钩子不是 BPP（调用时第一个 Bean 已在建），也不是 SmartInitializingSingleton（它在所有 Bean 建完之后），是 BFPP（定义阶段就动手，天然早于任何实例化）；
3. **"就绪"类钩子认准时序锚点**：CommandLineRunner 的契约锚点是"started 之后、ready 之前"（实测），SmartLifecycle.stop 的契约锚点是"ContextClosedEvent 之后、销毁之前"（反编译）——**顺序本身是契约**（02/07 篇的实测陷阱都源于此）。

徒弟：

> 十六个扩展点，光看表格根本记不住——有没有办法只记三个？

老陈：

> 有。别背钩子，背**三幕戏**：搭台 → 布景 → 开演。钩子全是"幕与幕、场与场之间的间隙"，记住每幕的**入口钩子和出口钩子**，中间的都可以现场推。

```text
第一幕 搭台（容器创建前）：EnvironmentPostProcessor → Initializer
  入口：EnvironmentPostProcessor（连容器都没有，只能动 Environment）
  出口：ApplicationContextInitializer（有容器、没 Bean）
  幕内逻辑：环境备好 → 把容器壳子立起来

第二幕 布景（refresh 12 步）：BFPP/BDRPP → BPP → onRefresh → 单例创建
  入口：BeanDefinitionRegistryPostProcessor（第一批，改图纸）
  出口：SmartInitializingSingleton（所有单例建完，最后补一刀）
  幕内逻辑：图纸 → 处理器 → 基础设施 → 逐个造 Bean

第三幕 开演（started 之后 → ready → 运行）：Runners → 运行期监听器 → 请求链路 → 关闭
  入口：CommandLineRunner / ApplicationRunner（started 后、ready 前）
  出口：ContextClosedEvent 监听器（关门前最后清理）
  幕内逻辑：宣布开演 → 接客 → 打烊
```

> **推演练习**："想在第一个 Bean 创建前做点全局动作"——幕间间隙查：BPP 注册早生效晚（②中段，注册在第 6 步、调用在单例创建时），BFPP 是"图纸阶段"（②入口，定义层面先改），SmartInitializingSingleton 是"完工后"（②出口，所有单例建完）——正确答案是 **BFPP**（在定义阶段就动手，天然早于任何实例化），BPP 和 SmartInitializingSingleton 都不行。**钩子 = 步骤间隙**：只要三步记法（①run 四阶段 ②refresh 三阶段 ③三幕戏）在脑里，任何钩子问你"什么时候动、能拿到什么"，都是从间隙的位置推出来的，不是背出来的。

## 关键不变量

> **事件顺序不可重排**。任何监听器如果依赖 `ApplicationReadyEvent` 之前的状态（比如 Tomcat 已监听端口），这个顺序就是契约。生产上"在 ready 前发请求会连接拒绝"就是这个顺序的直接后果（注：Tomcat 在 ready 前已监听端口，但此时应用层未就绪，是否可接流量取决于 readiness 语义，见 Level 5）。

**三个"就绪"信号不可混用**（24 章 04 章对照，Boot 4.1 基线文档语义，与 3.3.5 实测一致）：

| 信号 | 级别 | 含义 | 谁能用它 |
| ---- | ---- | ---- | ---- |
| ContextRefreshedEvent | Spring Context | 对象图建好、单例创建完成 | 应用内部代码 |
| ApplicationReadyEvent | Boot 应用 | Runner 执行完、启动流程收口 | 应用内部代码 |
| Readiness（平台流量信号） | 平台 | 可以接流量（依赖就绪） | LB / K8s Service |

**因果**：三层信号对应三层"能用"——Context 能用 ≠ 应用能用 ≠ 平台能放流量。把 `ContextRefreshedEvent` 监听器当"上线完成"、或把 readiness 当"进程活着"，都是层级错位（P-BS01 把 Started 当 Ready 的变体）。RunTraceApp 实测里三者在时间上依次出现（refresh 第 12 步 → started → ready），但语义边界是设计出来的：监听器依赖的关系（如"Tomcat 已监听"）只保证到对应信号为止。

---

# Level 3 内嵌容器：Tomcat 怎么被拉起来

## 为什么需要内嵌容器

外置 war 时代的痛：

- 每个环境要装 Tomcat，版本由运维管，开发环境和生产环境可能不一致；
- 应用打包成 war 放进去，容器启动顺序与应用启动顺序耦合；
- "跑起来"要两步：装容器 + 部署应用。

内嵌容器的本质：**把容器变成应用的一部分**。`tomcat-embed-core` 是个普通 jar，Boot 用 `TomcatServletWebServerFactory` 编程式创建 Tomcat 实例——应用启动即容器启动，应用退出即容器退出。代价：每个应用自带一份容器（内存/磁盘略多），换容器 = 换依赖（tomcat ↔ jetty ↔ undertow 三选一），这正是内嵌方案的核心 trade-off。

## WebApplicationType 探测：ClassPath 决定一切

`SpringApplication.createApplicationContext` 之前先探测 `WebApplicationType`：

```
classpath 有 org.springframework.web.reactive.DispatcherHandler
  且 无 org.springframework.web.servlet.DispatcherServlet / 无 Jersey → REACTIVE
classpath 有 jakarta.servlet.Servlet + org.springframework.web.context.ConfigurableWebApplicationContext
                                                                        → SERVLET
都没有                                                              → NONE
```

注意：SERVLET 的判定只看**两个类是否存在**（jakarta.servlet.Servlet + ConfigurableWebApplicationContext），与 WebApplicationInitializer 无关——后者是"程序化注册"用的扩展点，不参与类型判定。

实测证据（本项目）：`RunTraceApp` 在 classpath 含 spring-web + tomcat 的情况下实测为 SERVLET 分支——创建的上下文类是 `AnnotationConfigServletWebServerApplicationContext`，启动流程自动多出"启动内嵌容器"这一步；同一机制推导：去掉 web jar 应回退 NONE（**待验证**，未实测）。**同一个代码，加一个依赖，行为就变**——这就是 Boot "约定大于配置"最极端的一面。

徒弟：

> 为什么不用配置项声明"我是 Web 应用"，非要去 classpath 里翻类？这不隐蔽吗？

老陈：

> 隐蔽，但这是刻意选的——**插电即用**：外卖店租什么店面，取决于你买了什么设备（依赖），而不是嘴上说"我要开网店"。

```text
方案对比：
  配置声明：application.properties 写 web.type=servlet
    → 依赖与声明可能不一致（写了 SERVLET 却没引 web 依赖 → 启动炸）
    → 多一个必须"想对"的配置，少一个自动推导
  classpath 探测（Boot 选的）：
    → 依赖即声明：你引了 spring-web + tomcat，行为就是 Web 应用
    → 没有配置可写错，也没有配置可忘记
```

> 代价也在明处：**行为由依赖决定，加依赖不加配置，行为就悄悄变**——`RunTraceApp` 的 classpath 从 22 个 jar 加到 38 个（加 webflux/netty/jsr310 等）后，Bean 定义从 280 涨到 361，全是自动配置探测出来的。生产含义：**改依赖 = 改行为，必须过评审**（03 篇的门 5 就是这一条的另一面）。记住判定只看**两个类是否存在**，与 `WebApplicationInitializer` 无关——后者是"程序化注册"扩展点，不参与类型判定，别再混了。

## onRefresh：Web 场景的容器启动点

`ServletWebServerApplicationContext.onRefresh()` 覆盖了 refresh 第 9 步：

```
onRefresh()（ServletWebServerApplicationContext）
  └─ createWebServer()
       ├─ ServletWebServerFactory（从容器取 EmbeddedTomcat 工厂 bean）
       └─ factory.getWebServer(selfInitialize)
            ├─ new Tomcat() + Connector（默认协议 HTTP/1.1 NIO）
            ├─ prepareContext（注册 ServletContext 初始化器）
            └─ 返回 TomcatWebServer（只创建对象：构造里 initialize() 调用
               Tomcat.start() 但先 disableBindOnInit()——端口延迟绑定）
```

**重要：onRefresh 结束 ≠ 端口已监听**（反编译实证 + 实测）。真正的端口绑定和 `ServletWebServerInitializedEvent` 发布发生在 **finishRefresh（第 12 步）内部**：`LifecycleProcessor.onRefresh()` → `WebServerStartStopLifecycle.start()`（SmartLifecycle，反编译实证）→ `webServer.start()`（绑定 8080 开始 accept）→ `publishEvent(ServletWebServerInitializedEvent)`，然后才 `publishEvent(ContextRefreshedEvent)`。

**实测验证（2026-08-07）**：`@EventListener` 监听 `ServletWebServerInitializedEvent` **能收到**——因为事件发布在 finishRefresh 阶段，此时 @EventListener 处理器（第 11 步末尾注册）早已就绪。所以监听器收到的是"端口已绑定"时刻，不是 onRefresh 时刻。

**证据链（RunTraceApp 实测）**：为什么第一个经过 BPP 的 bean 是 `EmbeddedTomcat`？因为 `ServletWebServerFactoryConfiguration$EmbeddedTomcat` 这个 `@Configuration` 类本身是个单例 bean，而 `onRefresh` 在第 9 步就要它实例化来创建服务器——所以容器强制提前实例化它，它就成了 BPP 阶段第一个被"过一遍"的 bean。**这个顺序不是巧合，是 onRefresh 强制要求的**。

## 事件顺序的证据

```
ServletWebServerInitializedEvent   ← finishRefresh 内部：LifecycleProcessor.onRefresh()
                                    → WebServerStartStopLifecycle.start() 里
                                      端口绑定完成后发布（反编译实证）
ContextRefreshedEvent              ← finishRefresh 内部最后一步
```

实测（本机 3.3.5 + 2026-08-07 @EventListener 验证）：`ServletWebServerInitializedEvent` 永远在 `ContextRefreshedEvent` **之前**（两者都在第 12 步内部，先 SmartLifecycle 启动、后发布事件）——所以任何监听 ContextRefreshedEvent 的代码可以安全地假设"端口已监听"。

## 启动完成的官方定义

```
SpringApplication.run() 返回 + ApplicationReadyEvent 发布 = 启动完成
```

注意区分三个概念：

| 概念 | 含义 | 标志 |
| ---- | ---- | ---- |
| 端口已监听 | Tomcat 开始 accept | ServletWebServerInitializedEvent |
| 容器已就绪 | 所有单例实例化完、事件发布 | ContextRefreshedEvent |
| 应用已就绪 | Runner 执行完、可接外部信号 | ApplicationReadyEvent |

---

# Level 4 请求链路：一次 HTTP 请求的完整旅程

## 架构总览（运行时刻全景）

```
客户端
  │  GET /hello?name=world
  ▼
Tomcat Connector（http-nio-8080-exec-N 线程池取线程）
  │  解析 HTTP 报文 → 构造 HttpServletRequest
  ▼
Filter 链（doFilter → 继续往下传 → 返回后收尾）
  │
  ▼
DispatcherServlet（MVC 总入口）
  ├─ HandlerMapping：找到处理该路径的 Handler（@GetMapping → HandlerMethod）
  │     └─ 返回 HandlerExecutionChain（Handler + 拦截器列表）
  ├─ HandlerAdapter：参数适配 + 方法执行
  │     ├─ DispatcherServlet 先调用 chain.applyPreHandle（拦截器按注册顺序）
  │     ├─ adapter.handle()：Controller 方法执行（后厨炒菜）
  │     ├─ DispatcherServlet 再调 chain.applyPostHandle（响应即将生成）
  ├─ HttpMessageConverter：把返回值序列化成 JSON（Jackson）
  └─ DispatcherServlet 最后调 chain.triggerAfterCompletion（无论异常与否）
  │
  ▼
Filter 返回 → 响应体写入 Socket → 客户端拿到 HTTP 响应
```

## WebTraceApp 实测：完整打点

`demo11.web.WebTraceApp` 注册了一个 Filter、一个 Interceptor、一个 Controller，并用 HttpClient 自请求 `GET /hello?name=world`，真实输出：

```
[监听器] WebServer 启动完成：端口=8080
[M2] run() 返回
[请求] Filter.doFilter 进入
[请求] Interceptor.preHandle
[请求] Controller 执行：hello("world") → 返回 Map
[请求] Interceptor.postHandle（响应即将生成）
[请求] Interceptor.afterCompletion（响应已完成）
[请求] Filter.doFilter 返回（响应已生成）
[响应] HTTP 200 body={"message":"hello world","thread":"http-nio-8080-exec-1"}
```

（响应体由 HashMap 序列化，键序不保证，内容一致。）

三个关键证据：

1. **执行顺序 = 规范的拦截器契约**：`preHandle → Controller → postHandle → afterCompletion`，Filter 包在最外面（最早进入、最晚返回）；
2. **响应体里的 `"thread":"http-nio-8080-exec-1"`**：这是 Controller 里记录 `Thread.currentThread().getName()` 的结果——**证明请求确实跑在 Tomcat 连接器线程池的 worker 线程上**，而不是主线程（主线程在 `run()` 返回后干别的去了）；
3. **`[监听器] WebServer 启动完成：端口=8080` 出现在 run() 返回前**：再次印证 Level 3——容器启动是 refresh 的一部分，发生在 ready 之前。

## 分角色拆解（外卖配送映射）

| 环节 | 代码 | 比喻 | 职责 |
| ---- | ---- | ---- | ---- |
| 网络接收 | Tomcat Connector | 骑手接单 | 解析 HTTP、管理连接、线程池调度 |
| 前置过滤 | Filter 链 | 门卫检查 | 编码、鉴权（Spring Security 就是 Filter）、日志 |
| 总控分发 | DispatcherServlet | 平台分单 | 唯一 Servlet，负责所有请求的调度 |
| 路由匹配 | HandlerMapping | 查店名 | 路径 → 处理方法（@GetMapping 映射） |
| 参数适配 | HandlerAdapter | 核对订单 | 请求参数 → 方法参数（@RequestParam 等） |
| 业务执行 | Controller | 后厨炒菜 | 业务代码，只关心入参出参 |
| 序列化 | HttpMessageConverter | 打包出餐 | Java 对象 → JSON（Jackson） |
| 响应写出 | 连接器线程 | 骑手送达 | 字节写入 Socket |

**Controller 耗时 ≠ 完整请求耗时**（24 章 05 章 P-W01，Boot 4.1 基线文档语义）：

```text
完整请求耗时 = 连接队列等待 + 线程池排队 + Filter 链 + HandlerMapping 路由
             + 参数绑定 + Controller（业务）+ 返回值序列化 + 写回 Socket
```

**因果**：WebTraceApp 打点里 Controller 只占中间一段——Filter、序列化、写回都在路径上。只埋 Controller 的耗时监控（或只盯"业务慢"）会漏掉：线程池排队（连接器 worker 耗尽）、序列化（大响应体）、写回（慢客户端）。生产定位"接口慢"必须从 Filter 入口计时到响应写完，再逐段拆（07 篇四步法的第一刀）。


## 为什么是 DispatcherServlet 而不是每个 URL 一个 Servlet

传统 servlet：每个 URL 一个 Servlet 类，web.xml 里逐条声明——URL 越多配置文件越长，公共逻辑（鉴权/编码）无法集中。

DispatcherServlet 是**唯一入口 + 路由器**：所有请求进同一扇门，由 HandlerMapping 在内存表里查路由。公共逻辑集中在 Filter/Interceptor，路由变化只改注解。**代价**：一次请求多经过几层间接跳转（路由查找、适配器调用），换来的是"加接口不用改配置文件"。

## 另一条路：WebFlux（REACTIVE 栈）——双跑法实测

### 为什么存在另一条路

Level 6 的线程边界揭示了一个问题：**Servlet 栈一个请求占一个连接器线程**（阻塞模型）。高并发 I/O 密集场景（网关、消息推送、大量外部调用）下，线程池耗尽 = 请求全超时。

WebFlux 换了一种资源模型：**少量 EventLoop 线程 + 非阻塞 I/O**。线程不等待，而是"数据到了再干活"——同一个线程同时服务大量连接。这不是"更快"，而是**同样资源下能支撑的连接数模型不同**（本文不承诺任何性能数字）。

### 什么时候被选中：双跑法实测

Level 3 讲过探测条件，但有一个实测要点：**两个栈的 jar 都在时，MVC 优先**。

`demo11.webflux.WebFluxApp` 只写了 WebFlux 风格代码（`WebFilter` + `@RestController`），用两组 classpath 跑：

| 跑法 | classpath | 实测结果（WebApplicationType 分支） |
| ---- | ---- | ---- |
| 1 | 全 lib（含 spring-webmvc + tomcat-embed） | **SERVLET**（MVC 优先，代码被 Servlet 栈接管） |
| 2 | 去掉 spring-webmvc + tomcat-embed | **REACTIVE**（WebFlux 栈生效） |

跑法 1 实测（同一份 WebFlux 代码，跑在 Servlet 栈）：

```
[事件] WebServerInitializedEvent（TomcatWebServer，端口 8080）
[探测] 上下文类 = org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext
[请求] Controller 执行：hello("flux")（http-nio-8080-exec-2）
[响应] HTTP 200 body={"message":"hello flux","thread":"http-nio-8080-exec-2"}
```

跑法 2 实测（去掉 2 个 jar，REACTIVE 栈）：

```
[事件] WebServerInitializedEvent（NettyWebServer，端口 8080）
[探测] 上下文类 = org.springframework.boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext
[请求] WebFilter 链进入（reactor-http-nio-2）
[请求] Controller 执行：hello("flux")（reactor-http-nio-2）
[请求] WebFilter 链返回（reactor-http-nio-2）
[响应] HTTP 200 body={"thread":"reactor-http-nio-2","message":"hello flux"}
```

三个实测证据：

1. **classpath 决定请求栈**：同一份代码，加/去两个 jar（webmvc + tomcat）就切换栈——WebApplicationType 探测的直接后果；
2. **线程名暴露线程模型**：Servlet 栈 `http-nio-8080-exec-N`（Tomcat 连接器线程池，阻塞），Reactive 栈 `reactor-http-nio-N`（Netty EventLoop，非阻塞）；两次打印在同一个 EventLoop 线程上——没有"请求线程切换"；
3. **WebFilter 只在 REACTIVE 栈生效**：跑法 1 没有任何 WebFilter 打点——WebFilter 是响应式栈的扩展点，Servlet 栈的对应物是 Filter（Level 4）。

### 请求链路对比

```
Servlet 栈                                   WebFlux 栈
────────────────                          ─────────────────
Tomcat Connector 线程池                     Netty EventLoop（少量常驻线程）
  └─ Filter 链（javax/jakarta.servlet）      └─ WebFilter 链（org.springframework.web.server）
  └─ DispatcherServlet                       └─ DispatcherHandler（MVC 总控的响应式版）
  └─ HandlerMapping → HandlerAdapter         └─ HandlerMapping → HandlerAdapter
  └─ Controller（同步）                      └─ Controller（返回 Mono/Flux 或直接返回值）
  └─ HttpMessageConverter（阻塞写）           └─ 序列化写回（非阻塞写出）
```

关键差异：Servlet 栈每个环节**同步占线程**；WebFlux 栈全程无阻塞等待，配合 **Reactive Streams 背压**（下游处理不过来时向上游传递信号限制数据速率）——消费者慢，生产者就被拉闸，而不是排队堆积。

### 生产语义

- **探针不变**：Actuator health/readiness/liveness 与栈无关（Level 5 全部适用）；
- **EventLoop 上禁止阻塞**：Controller 里 `Thread.sleep()` / 同步 JDBC 会直接卡死整个 EventLoop（不只一个请求！）——这是 WebFlux 最大的生产坑；
- **决策**：默认 Servlet 栈；只有"极高并发 + I/O 密集 + 团队熟悉响应式"才选 WebFlux。**不要因为"性能更好"选它**——两种栈的吞吐差异依赖具体负载，阻塞代码写进 WebFlux 反而更糟。

---

# Level 5 可观测性：Actuator 与 K8s 就绪语义

## 为什么需要 Actuator

进程"活着"≠"能干活"：

- JVM 活着，但线程池打满、连接池耗尽、依赖的 Redis 挂了——这时进程活着，业务已死；
- K8s 需要区分"进程存在"（liveness）与"可以接流量"（readiness），不然会把流量打进一个正在重启的应用。

**Actuator 就是 Boot 内置的探针出口**：`health` 端点把"是否健康"的定义开放给开发者，每个组件（磁盘、Redis、DB）贡献一个 indicator。

## 实测：三个端点

`demo11.actuator.ActuatorApp` 真实输出：

```
[端点] GET /actuator/health → HTTP 200 {"status":"UP","components":{"diskSpace":{"status":"UP","details":{...}},"livenessState":{"status":"UP"},"ping":{"status":"UP"},"readinessState":{"status":"UP"}},"groups":["liveness","readiness"]}
[端点] GET /actuator/health/readiness → HTTP 200 {"status":"UP"}
[端点] GET /actuator/health/liveness → HTTP 200 {"status":"UP"}
```

两个实测发现：

1. **默认只暴露 health 端点**：`management.endpoints.web.exposure.include` 不配置的话，conditions 等端点 404。这是安全设计——生产上暴露过多内部信息有风险；
2. **探针组默认是关的**：`/actuator/health/readiness` 与 `/actuator/health/liveness` 在默认配置下 **HTTP 404**，必须显式设置 `management.endpoint.health.probes.enabled=true` 才 200。**这是最常见的生产配置遗漏之一**——K8s 探针配好了，但 Boot 侧没开探针组，探针永远打不到。

开启 probes 后，health 响应里多了 `"groups":["liveness","readiness"]` 和两个 `xxxState` 组件——这就是探针组生效的标志。

补充实测（2026-08-06 交叉验证）：`/actuator/startup`（StartupEndpoint）**默认 404，且开启 probes 后仍然 404**——它要 `management.endpoint.startup.enabled=true` 显式启用（3.3.5 实测：无任何配置时 health=200、liveness=404、readiness=404、startup=404；开 probes 后 liveness/readiness=200，startup 仍 404）。三个"探针可打"不是同一次开关，排查时逐个确认。

**状态链不变量**（24 章 14 章对照，Boot 4.1 基线文档语义，与实测一致）：

```text
进程存在 ≠ 端口监听 ≠ Context refresh ≠ ApplicationReady ≠ Readiness 放流量
```

**因果**：端口 LISTEN 只说明 Connector 绑定了 Socket（refresh 第 9 步），不代表 Context 建好（第 11 步）、更不代表依赖就绪——把"端口通"当"健康"是 P-BS02（端口监听当业务可用）。同一链路反过来也成立：SIGTERM 后 WebServer 先拒新、Context 后关——停机是启动的反向序列（07 篇 Level 8 展开）。

## K8s 探针语义映射

| K8s 探针 | Actuator 端点 | 语义 | 失败后果 |
| ---- | ---- | ---- | ---- |
| livenessProbe | /actuator/health/liveness | 进程还活着吗（自我存活） | 连续失败 → 重启容器 |
| readinessProbe | /actuator/health/readiness | 能接流量吗（依赖就绪） | 失败 → 从 Service 摘除，不杀容器 |
| startupProbe | K8s 侧探针（Boot 无专属端点，指向 liveness/readiness 端点） | 启动完成了吗（防启动抖动误杀） | 失败 → 继续等待启动 |

关键设计：**readiness 失败 ≠ 重启**。依赖 Redis 抖动时，readiness 变 DOWN，流量摘走，进程活着等待恢复——如果 liveness 也把"依赖"算进去，抖动就会被误杀重启。

## conditions 端点：03 篇报告的可观测版本

`/actuator/conditions` 返回的就是 03 篇条件评估报告的运行时版本。实测解析（3.3.5 JSON 结构）：

```
"DataSourceAutoConfiguration":{"notMatched":[{"condition":"OnClassCondition","message":"@ConditionalOnClass did not find required class 'org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType'"}],"matched":[]}
```

```
[报告] conditions 里 DataSourceAutoConfiguration → NEGATIVE（有未满足条件）
[报告] conditions 里 AopAutoConfiguration → POSITIVE（条件全通过，放行）
```

**跨篇闭环**：03 篇 ReportApp 在启动时打印的条件裁决（DataSource NEGATIVE / Aop POSITIVE），与 Actuator conditions 端点完全一致——**评估报告 = 启动时快照 = conditions 端点数据源**。生产排查"某个自动配置为什么没生效"，第一站就是看 `/actuator/conditions` 里它为什么 NEGATIVE（缺类 / 缺属性 / 被排除）。

---

# Level 6 生产实践：启动慢 / 停机 / 探针 / 排查

## 1. 启动慢怎么定位

启动耗时分布（大致，非 benchmark）：

```
prepareEnvironment（配置加载）    ~ 毫秒级
refresh：BFPP（自动配置注册）     ~ 慢的元凶之一（03 篇的 300+ 个定义，随 classpath 变化）
refresh：finishBeanFactoryInitialization（单例实例化） ← 最重
    ├─ DataSource 连接池初始化（连库！）
    ├─ RedisTemplate / 各种客户端
    └─ 每个 @PostConstruct
```

排查顺序：

1. `--debug` 看启动日志分阶段耗时；
2. 哪个 Bean 实例化最慢：单例实例化阶段加日志/观察初始化顺序；
3. 外部依赖（DB/Redis）是否在启动时强连通——**启动即连库是启动慢的第一元凶**，考虑懒连接；
4. 如果启动时不需要的外部系统，把连接初始化移到 ready 之后异步做。

## 2. 优雅停机（Graceful Shutdown）

问题：直接 kill 进程 → 正在处理的请求被中断、用户拿到空响应。

- JVM shutdown hook（Boot 3.2+ 的全局单例 `SpringApplicationShutdownHook`，run() 第 2 步开启注册开关）保证 JVM 退出时触发上下文 `close()`；
- `server.shutdown=graceful`（Boot 2.3+）：停止接收新连接，等存量请求处理完再销毁容器；
- `spring.lifecycle.timeout-per-shutdown-phase`：每阶段最长等待时间。

决策：**生产必须配 graceful**，但等待时间要设上限——否则下游故障时，存量请求永远处理不完，进程退不了。

## 3. 探针配置决策卡

```
management.endpoint.health.probes.enabled=true     ← 必须（否则探针 404）
management.endpoints.web.exposure.include=health   ← 最小暴露（别全开）
management.endpoint.health.show-details=never      ← 生产别暴露明细
```

注意：**show-details=always 只能用于内部环境**，生产暴露组件明细（磁盘路径、依赖状态）等于给攻击者递地图。

## 4. 请求链路排查手段

- 本文 WebTraceApp 的打点方式就是最小可行方案：Filter/Interceptor 加日志，看链路顺序；
- 生产：日志里带 traceId（MDC），跨 Filter→Controller→Service 串联；
- 慢请求定位顺序：先看耗时在哪个环节（Filter/Controller/序列化），再看是 CPU 还是 IO。

## 5. 运行时刻的线程边界（必须理解）

```
main 线程：run() 返回后 main 方法结束；JVM 进程不退，靠 prepareContext 注册的
          KeepAlive 保活机制（配合内嵌容器线程）
http-nio-8080-exec-N：Tomcat 连接器 worker 线程池（默认 maxThreads=200，Tomcat 连接器配置）
   ├─ 每个请求占用一个线程（阻塞模型）
   ├─ Controller 里 sleep / 调远程 → 线程被占住
   └─ 线程池耗尽 → 新请求排队（请求超时/拒绝）
```

**生产铁律**：Controller 线程是"连接器线程"而非"业务线程"。任何阻塞操作（DB、远程调用）都会占住 worker 线程；并发高时线程池耗尽，表现为"进程活着、请求全超时"——这时 health 还 UP，readiness 应该 DOWN（配合探针才能自动摘流）。

## 6. 决策卡：内嵌 vs 外置

| 维度 | 内嵌（Boot 默认） | 外置 war |
| ---- | ---- | ---- |
| 环境一致性 | 容器版本跟着应用走 | 容器版本由运维定 |
| 部署 | 一个 jar 直接跑 | 装容器 + 部署 war |
| 容器切换 | 换依赖即可（tomcat/jetty/undertow） | 换容器环境 |
| 内存 | 每应用一份容器 | 容器共享 |
| 适用 | 微服务/独立服务（默认选择） | 强约束的企业统一容器规范 |

**决策**：默认内嵌；只有"公司强制统一容器版本"这种组织约束才选外置。

---

# 全篇因果链总图

```
01 篇（Bean 从哪来）   03 篇（自动配置五门）         04 篇
     │                      │                        │
     ▼                      ▼                        ▼
refresh 是容器核心  ──→  门 5 注册 361 个 Bean 定义（本机快照） ──→  refresh 第 9 步 onRefresh
     │                                                │
     │                    03 篇条件评估报告           └─→ 强制实例化 EmbeddedTomcat
     │                      │                        │
     │                      ▼                        ▼
     │              DispatcherServlet 工厂           Tomcat 端口绑定（8080）
     │                      │                        │
     │                      ▼                        ▼
     │              [ServletWebServerInitializedEvent]  →  [ContextRefreshedEvent]
     │                                                     →  [ApplicationReadyEvent]
     │                                                          = 启动完成
     ▼
  运行时刻：请求 → Connector 线程 → Filter → DispatcherServlet
            → HandlerMapping → Interceptor → Controller
            → HttpMessageConverter → 响应
     │
     ▼
  Actuator health/readiness/liveness = 探针出口
     │
     ▼
  K8s 决定：readiness 摘流 / liveness 重启 / 流量放行
```

---

# 线上案例（3 个）

## 案例 1：K8s 探针一直红，应用其实好好的

现象：Pod 被反复重启，但应用日志正常、接口可用。

排查链路：

1. 先看探针端点：`curl /actuator/health/liveness` → 404；
2. 404 不是"不健康"，是**端点不存在**——探针组没开；
3. 结论：`management.endpoint.health.probes.enabled=true` 未配置（本文 Level 5 实测的 404→200 就是这个案例的复现）。

要点：**探针红之前，先确认探针打得通**。404 和 503 是两种完全不同的信号。

## 案例 2：进程活着，接口全超时

现象：health 显示 UP，但所有请求超时；K8s 没有动作。

排查链路：

1. 看 Tomcat 线程池：worker 线程全被占用（`http-nio-8080-exec-*` 全部 RUNNABLE 且堆栈停在远程调用）；
2. 根因：某接口调下游 Redis/HTTP，下游慢 → 阻塞模型下连接器线程被占满；
3. 修复：a) 给下游调用加超时；b) 上游限流；c) **把依赖状态纳入 readiness**——下游不可用时 readiness 自动 DOWN，流量摘走避免雪崩。

要点：health/readiness/liveness 三个信号职责不同（Level 5 语义表），**进程存活信号不能代替依赖就绪信号**。

## 案例 3：启动时连库，一挂全挂

现象：DB 重启期间发版，应用启动失败，滚动发布全红。

排查链路：

1. 启动日志：`finishBeanFactoryInitialization` 阶段，DataSource bean 实例化时连库超时 → 启动失败；
2. 根因：单例实例化是启动必经步骤（Level 2 第 11 步），DataSource 默认初始化即建连池；
3. 修复：a) `spring.datasource.hikari.initialization-fail-timeout` 放宽/关闭；b) readiness 承担"依赖未就绪"的信号，启动阶段探针用 startupProbe 等待；c) 发版顺序：先恢复 DB。

要点：**启动时刻的每一步失败 = 发版失败**。依赖不可用时，宁可让应用起来但 readiness DOWN（摘流等待），也不要启动即崩（发布全红）。

---

# 面试自查表

| 问题 | 答案要点 |
| ---- | ---- |
| run() 里做了哪几件大事 | 环境准备、创建上下文、prepareContext、refresh（12 步）、事件广播、Runner、ready |
| refresh 第 9 步 onRefresh 在 Web 场景干什么 | ServletWebServerApplicationContext 覆盖它 → createWebServer → 启动 Tomcat |
| 端口绑定完成的事件是什么 | ServletWebServerInitializedEvent（在 ContextRefreshedEvent 之前，实测） |
| WebApplicationType 怎么决定 | classpath 探测：有 reactive DispatcherHandler（且无 MVC/Jersey）→ REACTIVE；有 jakarta.servlet.Servlet + ConfigurableWebApplicationContext → SERVLET；否则 NONE（实测：加 jar 行为即变） |
| 一次请求完整链路 | Connector 线程 → Filter → DispatcherServlet → HandlerMapping → Interceptor(preHandle) → Controller → postHandle → Converter → afterCompletion → Filter 返回 |
| 请求跑在哪个线程 | http-nio-8080-exec-N（连接器线程池，实测线程名） |
| readiness 和 liveness 区别 | readiness=能接流量（依赖就绪），失败摘流不杀；liveness=进程存活，失败重启 |
| 探针 404 是什么原因 | probes.enabled 未开启（实测 404→200） |
| 启动完成官方定义 | ApplicationReadyEvent 发布 / run() 返回（Runner 执行完） |
| 优雅停机怎么配 | server.shutdown=graceful；**3.3.5 无 server.shutdown-timeout（排空无限等待，实测），超时上限需应用层自建**（07 篇 Level 8 展开） |
| refresh 失败后容器还能用吗 | 不能：自动销毁已创建单例 → isActive=false → getBean 抛 IllegalStateException → 不支持二次 refresh（demo15.RefreshFailApp 实测） |
| startup 端点默认可用吗 | 不可用：默认 404，开 probes 后仍 404；需 management.endpoint.startup.enabled=true 显式启用（3.3.5 实测） |
| 内嵌 vs 外置 | 环境一致性/单 jar 部署 vs 企业统一容器规范 |
| Servlet 栈和 WebFlux 栈差异 | classpath 决定栈（两个都在时 MVC 优先，实测）；Tomcat 阻塞线程池 vs Netty EventLoop（reactor-http-nio-N，实测）；WebFilter vs Filter；背压 |

---

# 坑与细节（8 个）

## 坑 1：主类缺 @SpringBootApplication → MissingWebServerFactoryBeanException

现象：`Unable to start web server ... no ServletWebServerFactory bean`，应用启动失败。

根因链：ServletWebServerFactory 是自动配置注册的（03 篇的门）；自动配置的入口是 `@SpringBootApplication` → `@EnableAutoConfiguration`；**主类丢了注解 = 自动配置全灭** = 没有 Web 容器工厂。

实际排障教训（本项目开发期真实经历）：备份文件里丢失了 `@SpringBootApplication`，数小时被误导——先查机制，后查代码。**排查顺序：先确认自动配置有没有跑（--debug / conditions 端点），再深入机制。**

## 坑 2：同一包多个 @SpringBootApplication 互扫

主类所在包被组件扫描——同包多个带 `@SpringBootApplication` 的类会互相扫描，导致：a) 自动配置重复注册；b) 歧义映射（Ambiguous mapping）；c) 启动速度下降。

**约定**：每个包只放一个主类；实验代码用不同包隔离（本项目 demo11 / demo11.web / demo11.actuator 三个包）。

## 坑 3：out/ 残留旧 class 污染运行

javac 单文件编译不会清掉旧 class。改了包名/删了类后，`out/` 下残留旧 class 会导致诡异行为（如 "Ambiguous mapping"——旧类还在扫描范围里）。

**约定**：改包名/删类后必须 `./build.sh` 全量重建（本项目 build.sh 先清 out/）。

## 坑 4：默认只暴露 health，其他端点全 404

`/actuator/conditions` 等端点默认 404——需要 `management.endpoints.web.exposure.include`。**最小暴露原则**：只开需要看的端点。

## 坑 5：8080 端口被占

内嵌 Tomcat 默认 8080（实测端口=8080）。端口占用时启动失败，抛出端口绑定异常（BindException，Tomcat 启动失败日志）。改 `server.port` 或换端口。**注意**：实验环境的端口冲突不是框架问题。

## 坑 6：startupProbe 缺失 → 启动抖动被误判

应用启动慢（比如 30s），liveness 探针按 10s 间隔检查——启动期 liveness 失败被误杀重启。K8s 的 startupProbe 解决这个：启动期用 startupProbe 放宽（与 liveness 同端点），启动完成后再交给 liveness 常规检查。这是 K8s 侧配置，Boot 侧只需保证探针组已开启（Level 5 的 probes.enabled）。

## 坑 7：show-details=always 泄密

health 组件明细（磁盘路径、依赖状态）在公网暴露 = 信息泄露。生产用 `show-details=never`，内部环境才开 always。

## 坑 8：关闭钩子顺序依赖

RunTraceApp 实测 `ctx.close()` 会触发 AvailabilityChangeEvent（可用性状态变更）再 ContextClosedEvent。**生产代码不要在 ContextClosedEvent 里依赖其他 bean 还活着**——关闭顺序不保证，JVM 退出钩子阶段依赖方可能已销毁。

## 坑 9：手工 javac 编译缺 `-parameters` → @RequestParam 全 500

现象（本项目真实踩坑）：Controller 能映射（/ping 200），但带 `@RequestParam String name` 的接口 500，异常 `Name for argument of type [java.lang.String] not specified, and parameter name information not available via reflection`。

根因：Spring 6.1 默认从参数名元数据解析 @RequestParam 的名称；javac 需 `-parameters` 标志才会把参数名写进字节码。Maven/Gradle（spring-boot-starter-parent 的 maven-compiler-plugin 默认配置）自带该标志，**手工 javac 不会**。

修复：`javac -parameters`（本项目 build.sh 已加）。教训：**手工编译环境 ≠ 构建工具环境**，参数名元数据是 Spring 6.1+ 的隐式依赖。

---

# 版本勘误表

| 项 | 勘误内容 |
| ---- | ---- |
| 八步 vs 反编译序列 | "run 八步"是读者视角骨架；3.3.5 反编译为 16 条调用序列（含 prepareContext 子步骤）；**run() 里没有直接 registerShutdownHook 调用**——只有 enableShutdownHookAddition 开开关（钩子本体是全局单例 SpringApplicationShutdownHook，Boot 3.2+） |
| refresh "13 步" | 勘误：6.1.14 反编译 refresh() 为 **12 个顶层方法调用**；resetCommonCaches 是 finishRefresh() 内部的第一个调用（清反射/注解缓存），不是独立步骤 |
| Bean 定义数（274→280→361） | 数字随 classpath 变化（22 jar 时 280，38 jar 时 361），机制不变；**任何文章写死 Bean 定义数都是错的**，只能作为本机快照 |
| ServletWebServerInitializedEvent 时序 | 实测在 ContextRefreshedEvent 之前（Boot 3.3.5）；2.x 行为需按版本验证 |
| probes 默认值 | 默认关闭（实测 404）；"Boot 自带探针"的说法不准确——自带端点但需显式开启 |
| conditions JSON 结构 | 3.3.5 为 `{类名:{notMatched:[...],matched:[]}}`；2.x 结构不同（report 数组），按版本验证 |

---

# 生产决策卡（3 张）

## 决策卡 1：探针配置

```
Decision: 启用 probes + 最小暴露 + 不暴露明细
Reason:   探针 404（默认关）+ 信息泄露风险（show-details）
Alternative: 自建 /health 接口（弃：重复造轮子，无组件指标）
Trade-off: 配置项增加，换取 K8s 自动摘流/重启能力
Validation: 三端点 curl 全 200；K8s 下故意停依赖，观察 readiness DOWN 且容器不重启
```

## 决策卡 2：内嵌 vs 外置容器

```
Decision: 默认内嵌（Boot 默认），强组织约束才外置
Reason:   环境一致性 + 单 jar 部署 + 容器切换只改依赖
Alternative: 外置 war（弃：两套容器管理，启动顺序耦合）
Trade-off: 每应用多一份容器内存；容器升级需重建镜像
Validation: 发版流程中验证 jar 包开箱即跑，无容器差异
```

## 决策卡 3：启动期依赖强连通

```
Decision: 启动不强连外部依赖；依赖状态进 readiness
Reason:   启动即连库 → 依赖挂则发版全红（案例 3）；readiness 本应承担依赖信号
Alternative: 启动时重试/等待依赖（弃：发布阻塞、超时不可控）
Trade-off: 应用可能"半就绪"启动（readiness DOWN 期间），换取发版不依赖外部系统
Validation: DB 停机状态下发版：应用起来、readiness DOWN、探针不误杀
```

---

# 跨语言视角

- **Node.js/Express**：`app.listen()` 显式启动 HTTP 服务——"容器启动"与"框架就绪"是开发者手写的两行；Boot 把它变成 refresh 的一个步骤。K8s 探针语义通用（readiness/liveness）。
- **Go net/http**：`http.ListenAndServe` 采用多 goroutine 并发模型——每个连接由独立的 goroutine 处理，没有"线程池耗尽"的线程占用问题，但**资源耗尽的表现**（连接积压、超时、内存上涨）与 Tomcat 线程池耗尽是同族问题；readiness 摘流是通用解法。
- **Python/Django**：WSGI 服务器（gunicorn/uwsgi）= 连接器层，与 Django 应用解耦——内嵌/外置的取舍本质上就是"服务器进程与应用进程是否同生命周期"。
- **通用方法论**：**启动序列显式化 + 事件化 + 探针化**是服务化框架的共同演进方向（Spring 的事件链、Node 的 listen 回调、K8s 的探针协议）；"进程存活 ≠ 服务就绪"是所有语言的生产铁律。
