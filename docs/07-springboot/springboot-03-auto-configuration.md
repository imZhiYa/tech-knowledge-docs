# 🎫 SpringBoot 自动装配深挖（系列 03）

> 本篇文章回答两个问题：
> 1. **一个 starter 的生效过程**：加一个依赖就能用，背后到底有几个"门"？候选是怎么被收集、被排除、被排序、被条件过滤、最终注册成 Bean 的？
> 2. **条件机制的本质**：`@ConditionalOnClass` 为什么不加载类就能判断？`@ConditionalOnMissingBean` 为什么"让位"给用户？`@ConditionalOnProperty` 的属性名到底宽松不宽松？条件评估报告里 Positive / Negative 是什么？
>
> 前置：01 篇 Level 2 的自动装配加载链概览（imports 白名单 → DeferredImportSelector → @Conditional 三问 → 条件评估报告）。本篇不重复链路，深挖**每个门的内部实现**：排除通道、排序算法、条件家族、覆盖通道、评估报告代码访问。

---

# ⚠️ 版本与证据边界

| 维度 | 本文承诺 |
| ---- | ---- |
| 代码实证 | `knowledge/springboot/experiments/code/` 下的 demo10×8，本机实测输出原样引用 |
| 实测环境 | macOS + JDK 21.0.11（Azul Zulu）+ spring-context **6.1.14** + spring-boot **3.3.5** + spring-boot-autoconfigure 3.3.5 |
| 运行方式 | `cd knowledge/springboot/experiments/code && ./build.sh && ./run.sh demo10.XXXApp` |
| Specification | `@ConditionalOn*` 注解语义、@ConditionalOnMissingBean 让位契约、@AutoConfiguration 注解属性（before/after）、排除通道语义、EnableAutoConfiguration 开关属性 |
| Implementation | ImportCandidates 加载路径、AutoConfigurationSorter 排序算法（getOrder 元数据表优先/注解兜底 + before/after 拓扑）、OnClassCondition/OnBeanCondition 的类继承差异、OnPropertyCondition 的匹配实现、报告 source 粒度（类名#方法名）与 Exclusions List 语义、@AutoConfigureOrder 默认值 0（3.3.5 反编译/反射） |
| 待验证 | 非 3.3.5 版本的 imports 候选数量与内置自动配置类清单细节；actuator conditions 端点在自定义 Actuator 端口下的 JSON 结构细节 |
| 未覆盖 | 不承诺任何性能数字；自动配置与 AOT/GraalVM 的编译期处理；Spring Cloud 的自动配置组合 |

---

# 🏷️ 关键词

自动装配 | AutoConfiguration.imports | ImportCandidates | DeferredImportSelector | AutoConfigurationGroup | 排除通道 | ENABLED_OVERRIDE_PROPERTY | AutoConfigurationSorter | @AutoConfiguration | @AutoConfigureOrder | 条件家族 | SpringBootCondition | FilteringSpringBootCondition | ConfigurationCondition | Phase 机制 | ASM 元数据 | 评估报告 | @AutoConfigurationPackage | 覆盖通道

---

# 🗂️ 目录

- Level 0 五种注册通道：Bean 定义从哪来（0.6~0.8 含 destroyMethod 推断坑 + 重复执行两案例）
- Level 1 为什么深挖：自动装配生命周期的五个门
- Level 2 候选收集与排除：进口名单与黑名单
- Level 3 排序：免签协议的优先级
- Level 4 条件家族：逐条检查协议的四个实现
- Level 5 覆盖通道：用户永远排在自动配置前面
- Level 6 生产实践：条件出问题怎么排查
- 全篇因果链总图
- 线上案例（3 个）
- 面试自查表
- 坑与细节（10 个）
- 版本勘误表
- 生产决策卡（3 张）
- 跨语言视角
- 系列索引

---

# 🏭 全文唯一比喻：免签通道

把 `@SpringBootApplication` 启动当作出境大厅：

- **免签名单**：`AutoConfiguration.imports` 白名单——就像"哪些国家的公民可以免签入境"；
- **黑名单**：`exclude` 排除——"名单上这些人今天拒签"；
- **优先级**：`@AutoConfigureBefore/After`——"免签协议按两国关系排序，谁先谁后处理"；
- **逐条条件**：`@ConditionalOn*`——"免签不等于随便进：护照有效期、目的地限制、携带物品逐一检查"；
- **评估报告**：`ConditionEvaluationReport`——"每个旅客被放行或被拒，边检都有记录（Positive / Negative）"；
- **用户优先**：`@ConditionalOnMissingBean`——"官方默认安排让位于旅客自备安排"。

全文一个比喻、一个主线角色：**一个 starter 的生效过程**（一位"免签旅客"从名单到入境的全流程）。

---

# Level 0 五种注册通道：Bean 定义从哪来

> 主线角色"免签旅客"（自动配置类）在过五道门之前，先认识**登记处**：
> 容器里任何一个 Bean 定义，来源只有五种——本篇主线（自动装配）用的正是其中两种的组合。

## 0.1 为什么先讲注册通道

01 篇讲了"一个 Bean 的一生"（定义 → 实例化 → 初始化），00 篇建立了创建链因果——但都没回答一个前置问题：**Bean 定义本身从哪来？**

先立这张全景图再进五道门，有两个收益：

1. Level 2 起出现的 `ConfigurationClassBeanDefinitionReader`、`ImportCandidates` 等名字，可以立刻对号入座——它们是某条通道的组件；
2. 读者日后在任何 Spring 项目里看到"Bean 不生效"，第一反应是"它走的是哪条通道？"，而不是漫无目的地猜。

## 0.2 五通道总图

```text
                        BeanDefinitionRegistry（登记处）
                                   ▲
   ┌──────────┬──────────┬─────────┼──────────┬─────────────┐
   │          │          │         │          │             │
XML 文件     组件扫描     @Bean      @Import     程序化注册
XmlBeanDefi- ClassPath-  配置类解析   Configura-  registerBean
nitionReader BeanDefini- (方法名=    tionClass-  Definition
(<bean>标签) tionScanner  Bean名)     Parser      (BFPP 阶段)
                                   (形态1: 类 / 形态2: ImportSelector
                                    / 形态3: ImportBeanDefinitionRegistrar)
```

五种入口最终都执行同一个动作：**向 `BeanDefinitionRegistry` 注册一个 `BeanDefinition`**。之后 getBean 的创建链（实例化 → 属性填充 → 初始化）完全同一套——这就是 00 篇"一个 Bean 的一生"对五种方式通用的原因。

## 0.3 五种方式的形态与使用场景

| 通道 | 代码形态 | 本质 | 什么时候用 | 什么时候别用 |
|---|---|---|---|---|
| XML `<bean>` | 外部 XML 文件，`@ImportResource`/XmlBeanDefinitionReader 解析 | 定义与代码分离，外部声明 | 遗留系统；**无法改源码的第三方类**；希望对象图集中可见 | 新项目（被配置类 + @ConfigurationProperties 取代）；需要类型安全的组装 |
| @Bean | `@Configuration` 类的方法 | 类型安全声明 + 方法体内可编程组装 | 第三方类封装；**复杂构造/初始化逻辑**；Boot 自动配置类内部主力 | 自家简单组件（@Component 更省）；无初始化逻辑时 |
| @Component | 类上注解，组件扫描 | 类自我标记"我是组件" | **自家业务代码**分层（@Service/@Repository/@Controller 是语义变体） | 第三方类（改不了源码）；需要精细控制命名/条件时 |
| @Import | 配置类上显式导入 | 显式声明模块依赖 | 模块化组装；**ImportSelector 按条件选择性导入**；ImportBeanDefinitionRegistrar 直接注册定义 | 同包扫描已覆盖的场景（重复导入无害但冗余） |
| 程序化注册 | BFPP 里 `registerBeanDefinition` / `GenericApplicationContext.registerBean` | 纯代码直达容器内部 | **框架/插件系统**；按运行时状态决定注册谁；在 BFPP 阶段改定义 | 业务代码手写注册逻辑（声明式足够） |

**选型口诀**：先问"代码归谁管"——自家代码 → @Component；第三方/需组装 → @Bean；遗留/纯外部 → XML；模块化/条件导入 → @Import；动态/框架 → 程序化。

## 0.4 实证：五种通道同容器共存（demo10.BeanRegisterWaysApp）

一个 App 里五条通道各注册一个 Greeter，全部 getBean 成功（真实输出，JDK 21.0.11 + Boot 3.3.5）：

```text
[通道] greeterXml 已注册=true prefix=xml
[通道] greeterBean 已注册=true prefix=bean
[通道] greeterComponent 已注册=true prefix=component
[通道] greeterSelector 已注册=true prefix=selector
[通道] greeterManual 已注册=true prefix=manual
[验证] 同容器同生命周期：greeterXml getBean 两次返回同一实例 = true

[定义] BeanDefinition 指纹（5 个定义都注册在同一个 BeanDefinitionRegistry）：
  greeterXml       beanClass=Greeter           factoryMethod=null            resource=class path resource [demo10/way/beans.xml]
  greeterBean      beanClass=-                 factoryMethod=greeterBean     resource=importedcfg.JavaConfig
  greeterComponent beanClass=GreeterComponent  factoryMethod=null            resource=file [.../out/demo10/way/GreeterComponent.class]
  greeterSelector  beanClass=-                 factoryMethod=greeterSelector resource=class path resource [importedcfg/SelectorTargetConfig.class]
  greeterManual    beanClass=Greeter           factoryMethod=null            resource=null
```

**指纹就是"定义从哪来"的物证**（同一个 registry 里的 5 条定义，用 BeanDefinition 自带字段就能溯源）：

| bean | beanClass | factoryMethod | resource | 通道来源 |
|---|---|---|---|---|
| greeterXml | Greeter | null | class path resource [beans.xml] | XML 文件 |
| greeterBean | -（注册时懒解析） | greeterBean | importedcfg.JavaConfig | @Bean 方法 |
| greeterComponent | GreeterComponent | null | GreeterComponent.class | 组件扫描 |
| greeterSelector | -（注册时懒解析） | greeterSelector | SelectorTargetConfig.class | ImportSelector 导入 |
| greeterManual | Greeter | null | null（无来源描述） | 纯代码注册 |

三条可迁移结论：

1. **@Bean 与 ImportSelector 产物的指纹高度相似**（都是 factoryMethodName + 无 beanClass 注册）——因为它们最终都走 `ConfigurationClassBeanDefinitionReader`，本 demo 里 ImportSelector 返回的配置类里的 @Bean 与直接 @Import 的 @Bean 本质是同一注册路径；
2. **XML 与扫描的定义在注册时就有具体类名和 resource**（reader/scanner 直接把类解析进定义）；
3. **程序化注册的定义最"裸"**（无 resource 无 factoryMethod），因为它的定义完全由代码手写——这也是为什么框架要"深度定制"定义时选这条通道。

## 0.5 与主线衔接：自动装配用的是哪两个通道

现在可以回答"免签旅客的登记处到底长什么样"：

```
入口：@EnableAutoConfiguration = @Import(AutoConfigurationImportSelector.class)
       ↓                        ↑
  形态 2（ImportSelector）←──────┘  决定"名单"：读 imports 文件 + 排除 + 排序 + 条件过滤
       ↓
落地：ConfigurationClassBeanDefinitionReader（ConfigurationClassParser 的注册器）
       ↓                        ↑
  标准 @Bean 注册路径 ←──────────┘  决定"落地"：把候选配置类的 @Bean 方法注册成定义（门 5）
```

**自动装配 = @Import 的 ImportSelector 形态（决定名单）+ 标准配置类注册路径（落地）**。Level 1 起，五个门拆的就是"名单怎么定"和"落地前还有什么裁决"。

## 0.6 坑：@Bean 的 destroyMethod 默认推断 "(inferred)"

**一句话**：`@Bean` 的 `destroyMethod` 默认值不是"不销毁"，而是 `"(inferred)"`——只要 @Bean 方法返回的对象里有 public 无参的 `close()` 或 `shutdown()`，Spring 就自动把它登记为销毁回调，容器关闭时无脑调用，哪怕这个方法根本不是用来释放资源的。

**为什么是这个默认值**：Spring 4.3 引入的便利特性——`@Bean` 通常用来封装第三方类（如线程池、连接），这些类往往恰好有 `close()`/`shutdown()`，自动推断省去手写。代价：**命名即契约**，业务类一旦取了这两个名字就无意识进入销毁流程。

**推断机制**（spring-beans 6.1.14 `DisposableBeanAdapter.inferDestroyMethodsIfNecessary` 反编译，方法名是复数、返回 `String[]`，支持多销毁方法）：

```text
触发条件：destroyMethodName == "(inferred)"        ← @Bean 的默认值
         或 (destroyMethodName == null 且 类实现 AutoCloseable)
                       ↓
     已实现 DisposableBean 接口？ → 是 → 不推断（走接口 destroy()）
                       ↓
     不是 AutoCloseable → 依次尝试 Class.getMethod("close", 无参)
                          → 命中 → "close"
                          → NoSuchMethodException → getMethod("shutdown", 无参)
                          → 都没有 → 不推断
     是 AutoCloseable → 直接 "close"（走 AutoCloseable.close() 接口调用）
```

`Class.getMethod` 的语义决定了三个匹配条件：

1. **public**——`private close()` 匹配不到（getMethod 只返回 public 方法）；
2. **无参**——`close(String)` 匹配不到（显式查无参版本）；
3. **继承层次任意层级都算**——父类声明的 public `close()`、接口 default 方法都匹配；
4. close 优先于 shutdown——两个都有时只调 close（实测只输出 close）。

**关键差异表**（实测验证，不是文档推演）：

| 注册方式 | 普通类有 close() | 实现 AutoCloseable | 实测 |
|---|---|---|---|
| @Bean（默认） | 推断 → 调 close | 推断 → 调 close | ✅ 两者都调 |
| @Bean(destroyMethod = "") | 显式关闭推断 | 显式关闭推断 | ✅ 都不调 |
| @Component 扫描 | **不推断** | **仍推断** → 调 close | ✅ |
| XML（不写 destroy-method） | **不推断** | **仍推断** → 调 close | ✅ |
| XML destroy-method="(inferred)" | 推断 | 推断 | 文档语义 |

**容易说错的点**："@Component/@Service 扫描注册的 Bean 不推断"——只对**普通类**成立（扫描注册的 destroyMethodName 为 null，没有 "(inferred)"）；但只要类实现了 `AutoCloseable`，无论 @Bean / 扫描 / XML 注册，容器关闭时都会调 `close()`（反编译第二分支：`name == null && closeable`）。

**生产建议**：

1. 业务类避免命名 `close()`/`shutdown()`——尤其是无参 public 版本；
2. 封装第三方类的 @Bean 若其 close() 不是资源释放语义，写 `@Bean(destroyMethod = "")` 显式关闭；
3. 排查入口：容器关闭日志里出现"意外调用 close()"→ 反查 bean 类名，看命名 + 注册方式即可定位，无需怀疑容器。

**实证**：`demo10.destroy.DestroyInferenceApp`（15 个场景同容器实测，JDK 21.0.11 + Boot 3.3.5）：

```text
[销毁] closableThing: close() 被调用            ← 普通类 public close() 推断命中
[销毁] closeShutdownThing: close() 被调用       ← 同时有 shutdown()，只调 close
[销毁] inheritedCloseThing: 父类 close() 被调用  ← 继承层次的 public 方法也算
[销毁] autoCloseableThing: close() 被调用       ← @Bean + AutoCloseable
[销毁] shutdownOnlyThing: shutdown() 被调用     ← close 找不到 → 退而求其次
[销毁] scanAutoCloseThing: close() 被调用       ← 扫描 + AutoCloseable 仍推断
[销毁] xmlAutoCloseThing: close() 被调用        ← XML + AutoCloseable 仍推断
（privateCloseThing / argCloseThing / explicitEmptyThing / scanCloseThing / xmlCloseThing 未出现 = 未推断）
```

## 0.7 案例三：close 被调用两次 + NPE 警告

**场景**：两个 `@Bean` 方法返回**同一个实例**（共享单例）：

```java
@Bean SharedCloseThing sharedA() { return SHARED; }
@Bean SharedCloseThing sharedB() { return SHARED; }
```

**为什么 close 会调两次**：销毁单位是 **bean 定义**不是实例——容器对每个定义各创建一个 `DisposableBeanAdapter`（都推断出 close），两个 adapter 先后销毁同一个实例 → close() 调两次：

```text
[销毁] sharedCloseThing: close() 第 1 次被调用（同一实例两个 bean 定义）
[销毁] sharedCloseThing: close() 第 2 次被调用（同一实例两个 bean 定义）
[销毁] sharedCloseThing: 资源已释放——重复 close 抛异常，Spring 捕获记 WARN，不中断后续销毁
```

第二次调用时资源已释放（内部字段为 null）→ 抛异常。`DisposableBeanAdapter.invokeCustomDestroyMethod` 捕获（Exception table：InvocationTargetException/ExecutionException）→ `logDestroyMethodException` 记 WARN → **不中断销毁流程**，后续 bean 照常销毁、容器正常关闭。

**修复**：显式 `@Bean(destroyMethod = "")` 只保留一个销毁回调（共享实例只销毁一次）；或避免两个定义共享同一实例（用包装类）。

## 0.8 案例四：@PreDestroy + 推断的 shutdown = 重复执行

**场景**：bean 同时有 `@PreDestroy` 方法和被推断命中的 `close()`/`shutdown()`：

```java
@PreDestroy void cleanup() { ... }   // 注解通道
public void shutdown() { ... }       // 推断通道（同做清理 → 重复执行）
```

**为什么重复执行**：销毁流程 `destroy()` 里 **@PreDestroy（BPP 通道）先执行，推断方法（自定义销毁通道）后执行**，两条通道互不知情：

```text
[销毁] preDestroyShutdownThing: @PreDestroy cleanup() 执行
[销毁] preDestroyShutdownThing: shutdown()（destroyMethod 推断通道）执行
```

**关键边界——内置去重只保护"同名"**（6.1.14 反编译实证）：

1. 创建阶段 `InitDestroyAnnotationBeanPostProcessor$LifecycleMetadata.checkInitDestroyMethods` 把 **@PreDestroy 方法名**注册进 `mbd.externallyManagedDestroyMethods`；
2. `DisposableBeanAdapter` 构造器检查 `hasAnyExternallyManagedDestroyMethod(推断出的方法名)` → **命中即跳过推断**。

所以 `@PreDestroy` 直接标在 `close()` 上（同名）→ 推断被跳过 → **只执行一次**（实测唯一一次输出即 @PreDestroy 通道）；`@PreDestroy` 标在别的名字（cleanup）→ 与推断的 shutdown() 各执行一次 = **重复清理**。

**修复**：显式 `@Bean(destroyMethod = "")` 关闭推断通道——@PreDestroy 已负责清理时，这是消除重复执行的直接手段。

**生产建议补充**：销毁日志里出现"同一清理逻辑执行两次"→ 先区分是"同方法两次"（怀疑共享实例，案例三）还是"两个方法各一次"（怀疑 @PreDestroy + 推断，案例四），再看注册方式定位。

---

# Level 1 为什么深挖：自动装配生命周期的五个门

## 1.1 01 篇停在哪里

01 篇回答了"为什么加个依赖就能用"的**链路**：

```text
META-INF/spring/...AutoConfiguration.imports（白名单）
  → ImportSelector（延迟导入）
  → 候选自动配置类
  → @Conditional 逐个裁决
  → 通过的注册成配置类 → 生成 Bean
```

但 01 篇没有回答三个"怎么"：

1. 候选**怎么**被排除？（@SpringBootApplication(exclude=...) 传给谁、记录在哪？）
2. 自动配置类**怎么**排序？（imports 文件里写在前面的就先生效吗？）
3. 条件**怎么**裁决？（类条件为什么不用加载类？bean 条件为什么能看到用户 bean？属性条件为什么精确匹配？）

## 1.2 五个门的总览

一个自动配置类从"名单"到"生效"，过五道门（3.3.5 Implementation 全景）：

```text
AutoConfigurationImportSelector（DeferredImportSelector + Ordered，javap 实证）
  │
  ├─ 门 1 候选收集：ImportCandidates.load 读 META-INF/spring/{FQCN}.imports
  │         ↓
  ├─ 门 2 排除过滤：getExclusionFilter() 剔除 @SpringBootApplication(exclude)
  │        + spring.autoconfigure.exclude 配置（exclude/excludeName 三通道合并）
  │         ↓
  ├─ 门 3 类条件预过滤：OnClassCondition / OnWebApplicationCondition
  │        （无 ConfigurationPhase → 注册前就能过滤，ASM 元数据不加载类）
  │         ↓
  ├─ 门 4 排序：AutoConfigurationSorter.getInPriorityOrder
  │        （@AutoConfigureOrder 初排 + before/after 拓扑重排，3.3.5 反编译实证）
  │         ↓
  └─ 门 5 注册 + 完整条件评估：ConfigurationClassBeanDefinitionReader
           （OnBeanCondition 等 REGISTER_BEAN 阶段条件 → Bean 定义注册）
```

**五个门的本质分工**：门 1/2/3 在"导入前"做减法（成本最低，只读元数据）；门 4 决定处理顺序；门 5 在"注册时"做最终裁决（此时能看到已注册的用户定义——覆盖通道的机制本体）。

**为什么分两段过滤？** 类条件（门 3）只依赖 classpath 元数据，可以在导入阶段用 ASM 批量判断、不实例化任何东西；bean 条件（门 5）依赖"谁已经注册了"，必须等用户配置处理完。**这是"先减后决"的两段式设计——先砍掉不可能通过的，再对剩下的逐条裁决**。

---

# Level 2 候选收集与排除：进口名单与黑名单

## 2.1 候选收集：ImportCandidates 与 imports 文件

**机制本体（3.3.5 Implementation）**：

```java
// 静态工厂，按注解类名定位文件：
//   META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
ImportCandidates.load(EnableAutoConfiguration.class, classLoader);
```

- 文件位置 = `META-INF/spring/{注解类全名}.imports`（`org.springframework.boot.autoconfigure.AutoConfiguration.imports`）；
- **每个 jar 都能贡献一行或多行**——starter 只要在自己 jar 里放这个文件，就被"免签名单"收录；
- spring-boot-autoconfigure-3.3.5.jar 自带白名单（本机实测该文件存在，**152 行**，如 AopAutoConfiguration、DataSourceAutoConfiguration、LifecycleAutoConfiguration……）；
- 读取者是 `AutoConfigurationImportSelector`——javap 实证（3.3.5）：

```text
public class AutoConfigurationImportSelector
  implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
             BeanFactoryAware, EnvironmentAware, Ordered {
  public String[] selectImports(AnnotationMetadata);
  public Predicate<String> getExclusionFilter();     ← 门 2 的黑名单过滤
  public Class<? extends DeferredImportSelector.Group> getImportGroup();  ← AutoConfigurationGroup
  public int getOrder();                             ← Ordered
}
```

**为什么用 DeferredImportSelector？** 这是覆盖通道的地基（Level 5 展开）：普通 @Import 在解析用户配置的过程中立即处理；Deferred 延迟到**用户配置全部解析之后**统一处理——自动配置因此"永远排在用户后面"，条件评估时用户 Bean 已经注册。

**为什么用 Group？** 多个 jar 的候选混在一起，先收集进 AutoConfigurationGroup 再统一"去重 + 排除 + 排序 + 过滤"，保证顺序可控。

## 2.2 排除三通道

**通道 1：注解属性**（@EnableAutoConfiguration 声明，javap 实证）：

```java
public interface EnableAutoConfiguration extends Annotation {
  public static final String ENABLED_OVERRIDE_PROPERTY;   // "spring.boot.enableautoconfiguration"
  public abstract Class<?>[] exclude();
  public abstract String[] excludeName();
}
```

```java
@SpringBootApplication(exclude = LifecycleAutoConfiguration.class)   // demo10.ExclusionApp
```

**通道 2：配置文件属性** `spring.autoconfigure.exclude=xxx.XXXAutoConfiguration`。**实测生效（本机 3.3.5）**：设置 `spring.autoconfigure.exclude=...AopAutoConfiguration` 后，Exclusions 报告出现该配置类，且与注解通道（`exclude=LifecycleAutoConfiguration.class`）**合并进同一个排除列表**——两通道共同作用，没有优先级覆盖关系。

**通道 3：总开关** `spring.boot.enableautoconfiguration=false`（ENABLED_OVERRIDE_PROPERTY，javap 实证）——整条免签通道关闭。

## 2.3 实证：排除动作被评估报告记录（demo10.ExclusionApp）

实测输出（本机 Boot 3.3.5）：

```text
[配置] @SpringBootApplication(exclude=LifecycleAutoConfiguration.class)
[报告] Exclusions: [org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration]
[验证] LifecycleAutoConfiguration 未注册（排除生效），容器正常启动
```

- 排除动作不仅生效，还写进 `ConditionEvaluationReport.recordExclusions`，事后用 `getExclusions()` 读回——**可观测性是自动装配的第一设计原则**（后面 Level 4 的评估报告同理）；
- 排除发生在**候选注册之前**（getExclusionFilter 在导入阶段剔除），被排除的类根本不会走到条件评估。

**关键点**：排除的语义是"今天拒签"，不是"取消协议"——spring-boot-autoconfigure 的 imports 名单不动，只是本次启动不执行该自动配置类。

---

# Level 3 排序：免签协议的优先级

## 3.1 排序器：AutoConfigurationSorter

javap 实证（3.3.5，包私有类）：

```text
class org.springframework.boot.autoconfigure.AutoConfigurationSorter {
  AutoConfigurationSorter(MetadataReaderFactory, AutoConfigurationMetadata);
  List<String> getInPriorityOrder(Collection<String>);
}
```

- 包私有——排序器不对外暴露，只有 AutoConfigurationImportSelector 内部使用；
- 输入：候选类名集合；输出：排序后的名单。

## 3.2 排序依据：@AutoConfiguration 的 before/after 属性

javap 实证（3.3.5）：

```text
public interface AutoConfiguration extends Annotation {
  public abstract String value();
  public abstract Class<?>[] before();
  public abstract String[] beforeName();
  public abstract Class<?>[] after();
  public abstract String[] afterName();
}
```

**注意：@AutoConfiguration 没有 order 属性**——自动配置类之间的全局顺序用独立的 `@AutoConfigureOrder`（默认 `value = 0`，反射实测 3.3.5，不是 LOWEST_PRECEDENCE）。before/after 描述**相对依赖**，@AutoConfigureOrder 描述**绝对优先级**，两者混用——排序器（`AutoConfigurationSorter.getInPriorityOrder`，javap 反编译 3.3.5）实际做**两段排序**：先按 `getOrder()` 初排（order 来源：AutoConfigurationMetadata 元数据表的 `AutoConfigureOrder` 键优先，未命中再回退到注解属性），再按 before/after 依赖做**拓扑重排**。

**为什么拓扑排序而不是简单排序？** 因为 before/after 是"偏序"（A before B、C before D，A 与 C 无关系）——必须按依赖图排，不能按单一数值排。这与 Maven 依赖排序、构建工具的任务编排是同一个算法家族。**order 只能表达"级别"，表达不了"谁先于谁"的偏序**——这就是为什么 @AutoConfigureOrder 只做初排、before/after 才是最终裁决者。

## 3.3 实证：before 依赖生效（demo10.OrderingApp）

实验设计：imports 文件里 **OrdB 写在 OrdA 之前**（候选收集顺序 B 先），但 OrdA 声明 `@AutoConfiguration(before = OrdB.class)`。

实测输出（本机 Boot 3.3.5）：

```text
[imports] 候选收集顺序：autoext.OrdB → autoext.OrdA（B 在前）
[声明] OrdA 标注 @AutoConfiguration(before=OrdB.class)
[容器] beanDefinitionNames（demo10-order-*）：[demo10-order-a, demo10-order-b]
[结论] 排序器按 before 依赖重排——A 在 B 前（拓扑排序生效）
```

**怎么观察的**：自动配置类按排序后的顺序处理，Bean 定义注册顺序（`getBeanDefinitionNames()`）跟着走——imports 顺序被拓扑重排覆盖。**imports 文件里写在前面的不一定先生效，before/after 才是权威**。

---

# Level 4 条件家族：逐条检查协议的四个实现

## 4.1 模板方法：SpringBootCondition

javap 实证（3.3.5）：

```text
public abstract class SpringBootCondition implements Condition {
  public final boolean matches(ConditionContext, AnnotatedTypeMetadata);   ← final 模板
  protected final void logOutcome(String, ConditionOutcome);
  public abstract ConditionOutcome getMatchOutcome(ConditionContext, AnnotatedTypeMetadata);  ← 子类实现裁决
  protected final boolean anyMatches(...);
  protected final boolean matches(ConditionContext, AnnotatedTypeMetadata, Condition);
}
```

**`matches` 是 final 模板方法**：所有 `@ConditionalOn*` 条件的裁决都走同一管道——调用子类的 `getMatchOutcome`，把结果**写进 ConditionEvaluationReport**（recordConditionEvaluation），再输出到日志。**这就是评估报告为什么完整**：不是额外埋点，而是模板方法的设计使然。

## 4.2 Phase 机制：类条件先过滤、bean 条件后裁决

javap 实证（3.3.5）——**两个条件的类继承差异决定了评估时机**：

```text
class OnClassCondition extends FilteringSpringBootCondition { ... }
     // 无 ConfigurationPhase —— 可以在候选注册前批量过滤（门 3）

class OnBeanCondition extends FilteringSpringBootCondition
     implements ConfigurationCondition {
  public ConfigurationPhase getConfigurationPhase();   // REGISTER_BEAN
}
```

```text
SpringBootCondition.matches（final 模板）
  ├─ OnClassCondition（无 phase）→ 门 3 预过滤（导入阶段，ASM 元数据）
  └─ OnBeanCondition（REGISTER_BEAN）→ 门 5 注册阶段裁决（能看到已注册定义）
```

**因果**：`@ConditionalOnMissingBean` 的问题域是"容器里有没有这个 Bean"——容器在注册阶段才有答案，所以 OnBeanCondition 必须实现 ConfigurationCondition 声明 REGISTER_BEAN；`@ConditionalOnClass` 的问题域是"classpath 有没有这个类"——导入阶段就能答，所以它不做约束，还能承担门 3 的批量预过滤（省掉注册阶段的开销）。

## 4.3 类条件：ASM 元数据，不加载类（demo10.ClassConditionApp）

**机制本体**：`@ConditionalOnClass(name=...)` 判断"类是否在 classpath"，依据是 **MetadataReaderFactory 的 ASM 元数据**（读 .class 字节码头），**不触发类加载**。

实验设计：自动配置类 `ClassAutoConfigN` 标注 `@ConditionalOnClass(name = "com.example.never.Exists")`（classpath 不存在）；对照 `ClassAutoConfigP` 标注 `java.util.ArrayList`。

实测输出（本机 Boot 3.3.5）：

```text
[验证] com.example.never.Exists 在 classpath？Class.forName 探测 = false
[验证] 启动全程未抛 NoClassDefFoundError——条件检测未加载类（ASM 元数据）
[报告] autoext.ClassAutoConfigP
  [条件] OnClassCondition → 匹配：@ConditionalOnClass found required class 'java.util.ArrayList'
[报告] autoext.ClassAutoConfigN
  [条件] OnClassCondition → 不匹配：@ConditionalOnClass did not find required class 'com.example.never.Exists'
[容器] demo10-class-p 已注册=true；demo10-class-n 已注册=false
```

**为什么必须用元数据？** 如果 Class.forName 探测，不存在的类会抛 ClassNotFoundException 污染启动；更关键的是**加载类有副作用**（static 初始化、类锁竞争），自动配置要扫描几百个候选，逐类加载不可接受。ASM 只读字节码文件头，成本最低。

**生产意义**：`@ConditionalOnClass` 的类名写错（拼写/包名），效果是"条件静默不匹配，自动配置静默不生效"——不会报错，只能从评估报告发现。

## 4.4 bean 条件：让位契约（demo10.OverrideApp）

**机制本体**：`@ConditionalOnMissingBean` 在 REGISTER_BEAN 阶段用 `beanFactory.getBeanNamesForType` 探测**已注册的 Bean 定义**（不需要实例化）。由于自动配置类被 DeferredImportSelector 延迟到用户配置之后处理（Level 5），评估时用户定义已在注册表里。

实验设计：`ServiceAutoConfig`（imports 注册）提供默认 Greeter，`@ConditionalOnMissingBean(Greeter.class)`；用户配置类内部声明 `@Bean Greeter`。

实测输出（本机 Boot 3.3.5）：

```text
[用户] 用户 @Bean Greeter 已注册
[报告] autoext.ServiceAutoConfig 的条件链：
  [条件] OnBeanCondition → 不匹配：@ConditionalOnMissingBean (types: autoext.Greeter; SearchStrategy: all) found beans of type 'autoext.Greeter' userGreeter
[容器] demo10-default-greeter 已注册=false；注入的 Greeter = Greeter(用户定义)
[机制] DeferredImportSelector 延迟导入 → 自动配置类后处理 → 条件能"看见"用户 bean
```

**三个要点**：

1. 条件消息里能看到具体的 bean 名（`found beans ... userGreeter`）——评估报告可审计到"谁让谁让位"；
2. 让位的不是"配置类整体"而是**那个 @Bean 方法**——条件标注在方法上，方法不注册，配置类本身可能仍然 Positive；
3. **Demo 纪律**（本次实验血泪教训，见坑 8）：`@Bean` 方法必须声明在 `@SpringBootApplication` 配置类**内部**——声明在外部类里，方法不会作为配置方法被收集，定义根本不注册，条件探测永远"看不到"它，让位自然失败。

## 4.5 属性条件：精确匹配，没有宽松匹配（demo10.PropertyConditionApp）

**机制本体**（javap 反编译 `OnPropertyCondition$Spec.collectProperties`，3.3.5）：

```text
PropertyResolver.containsProperty(name)   ← 精确 key 查找
PropertyResolver.getProperty(name)        ← 精确 key 取值
isMatch: equalsIgnoreCase                 ← 值比较不区分大小写
```

实验设计：`PropAutoConfig` 标注 `@ConditionalOnProperty(name = "demo10.flag")`（matchIfMissing 默认 false），四次启动切换系统属性。

实测输出（本机 Boot 3.3.5）：

```text
[场景1] 未配置 demo10.flag → demo10-prop-bean 已注册=false（matchIfMissing=false）
[场景2] systemProperties: demo10.flag=true → demo10-prop-bean 已注册=true
[场景3] systemProperties: DEMO10_FLAG=true → demo10-prop-bean 已注册=false（无宽松匹配）
[场景4] systemProperties: demo10.flag=TRUE → demo10-prop-bean 已注册=true（值比较 equalsIgnoreCase）
```

**这是最容易踩的语义差异**（详见勘误表）：**宽松匹配（relaxed binding）是 @ConfigurationProperties 绑定阶段的特性，不是 @ConditionalOnProperty 的**——Boot 2 的 `RelaxedPropertyResolver`（属性条件用它做过宽松匹配）已在 Boot 3 移除。配置写 `DEMO10_FLAG=true`，属性绑定能拿到，**条件却判不匹配**——排查线上"配置明明写了却不生效"先查这个。

## 4.6 条件评估报告：Positive / Negative / Exclusions（demo10.ReportApp）

**数据结构**（javap 实证 3.3.5）：

```text
ConditionEvaluationReport
  getConditionAndOutcomesBySource()   → Map<source, ConditionAndOutcomes>（每个条件源的条件链）
  getExclusions()                     → 被排除的类（List，非 Set——见下方注意）
  getUnconditionalClasses()           → 无任何条件的类
  static get/find(BeanFactory)        → 从容器获取
```

**source 粒度（实测 3.3.5）**：key 是**类名或"类名#方法名"**——类级条件的 key 是整个配置类（如 `autoext.ServiceAutoConfig`），标在 @Bean 方法上的条件（如 ServiceAutoConfig 的 `@ConditionalOnMissingBean`）key 是 `autoext.ServiceAutoConfig#defaultGreeter`。**排查时用前缀匹配（startsWith）才能同时覆盖类级与方法级条件**（demo10.OverrideApp 的写法）。

**注意（实测观察）**：`getExclusions()` 返回 List（javap 字段 `exclusions:Ljava/util/List;`），同一排除类可能**重复出现**——自动配置选择器每处理一轮候选就记录一次（本实验环境因 demo10 同包 7 个 @SpringBootApplication 互相扫描触发多轮，观察到同一类最多重复 7 次）。单配置类项目一般无重复，但**读报告做断言时按去重后的集合判断**。

实验设计：直接访问报告，对比两个内置自动配置类的裁决——`DataSourceAutoConfiguration`（本实验 classpath 无 spring-jdbc，应 Negative）与 `AopAutoConfiguration`（有 spring-aop，应 Positive）。

实测输出（本机 Boot 3.3.5）：

```text
[报告] DataSourceAutoConfiguration（本实验 classpath 无 spring-jdbc，预期 Negative）
  [条件] OnClassCondition → 不匹配：@ConditionalOnClass did not find required class 'org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType'
[报告] AopAutoConfiguration（本实验 classpath 有 spring-aop，预期 Positive）
  [条件] OnPropertyCondition → 匹配：@ConditionalOnProperty (spring.aop.auto=true) matched
  [条件] OnClassCondition → 不匹配：@ConditionalOnClass did not find required class 'org.aspectj.weaver.Advice'   ← AopAutoConfiguration$AspectJAutoProxyingConfiguration
  [条件] OnClassCondition → 匹配：@ConditionalOnMissingClass did not find unwanted class 'org.aspectj.weaver.Advice'  ← AopAutoConfiguration$ClassProxyingConfiguration
[验证] 同一套条件机制，只因为 classpath 不同 → 裁决不同（逐条检查免签协议）
```

**这个输出本身就是一堂课**：

- **同一个自动配置类，两个内层配置类形成互补**：`$AspectJAutoProxyingConfiguration`（@ConditionalOnClass aspectjweaver → Negative）负责"有 AspectJ 时 JDK 代理"，`$ClassProxyingConfiguration`（@ConditionalOnMissingClass aspectjweaver → Positive）负责"没有 AspectJ 时 CGLIB 代理"——**一个功能两个分支，用正反两个类条件切换实现**，这是 starter 设计的经典模式（对称条件切换）；
- DataSource 场景：缺 `spring-jdbc` 依赖 → 整条数据源自动配置静默关闭，启动正常——**自动配置的降级是静默的，这正是评估报告存在的理由**；
- 报告可用代码访问（本次实验演示），也可通过 `--debug` 启动参数打印（Level 6）。

---

# Level 5 覆盖通道：用户永远排在自动配置前面

## 5.0 自动配置的三种结果（16 章固定结论，Boot 4.1 基线文档语义）

```text
不适用 → 不注册（条件否决，Negative matches）
适用且用户未接管 → 提供默认实现（自动配置的正常形态）
适用但用户已接管 → Back-off 让位（@ConditionalOnMissingBean 命中用户 Bean）
```

**因果**：无条件创建会破坏用户选择，多个 Starter 可能创建同类设施——所以自动配置的每一步都是"先问后做"（问条件、问用户有没有定义）。排查"自动配置为什么没生效"先判断它是哪种结果：**不注册（条件）、被覆盖（让位）、还是根本没进候选（imports/排除）**——三种结果对应三处机制位点（Level 2/4/5）。

## 5.1 机制本体：DeferredImportSelector 的延迟

```text
ConfigurationClassParser.parse（6.1.14 源码）
  ├─ 解析用户配置类（@SpringBootApplication 主类、@ComponentScan 结果）
  ├─ 收集 DeferredImportSelector（EnableAutoConfiguration 是其中之一）
  └─ parse 末尾：processDeferredImportSelectors()
        └─ AutoConfigurationGroup.process：统一收集 → 排除 → 过滤 → 排序 → selectImports
```

**因果链**：用户配置先解析（含 @Bean 定义收集）→ loadBeanDefinitions 先注册用户 Bean 定义 → 自动配置类后处理 → `@ConditionalOnMissingBean` 评估时用户定义已在注册表 → "found beans ... userGreeter" → 让位。

**这就是"约定优于配置，用户优先"的实现机制**——不是框架读心术，是处理时序的安排。

## 5.2 四覆盖通道总图

```text
                用户想覆盖自动配置的行为
                          │
        ┌─────────────────┼─────────────────┬──────────────────┐
        ▼                 ▼                 ▼                  ▼
   通道 1             通道 2            通道 3               通道 4
exclude 注解       spring.autoconfigure  自定义 Bean 覆盖      属性开关
@SpringBootApplication  .exclude 配置    @ConditionalOnMissingBean
(exclude=...)                           （自动配置让位）      @ConditionalOnProperty
        │                 │                 │                  │
        └─────────────────┴────────┬────────┴──────────────────┘
                                   ▼
              四者都记录在 ConditionEvaluationReport（Exclusions / Positive / Negative）
```

| 通道 | 语义 | 场景 | 实测 |
| ---- | ---- | ---- | ---- |
| exclude 注解 | 整个自动配置类拒签 | 某自动配置与本项目冲突 | demo10.ExclusionApp |
| 配置排除 | 运维/部署期关闭 | 环境差异（测试环境关某功能） | 同 exclude 机制，配置化 |
| 自定义 Bean | 默认实现让位 | 替换默认对象（数据源、RestTemplate） | demo10.OverrideApp |
| 属性开关 | 条件匹配翻转 | 功能开关（spring.aop.proxy-target-class） | demo10.PropertyConditionApp |

## 5.3 @AutoConfigurationPackage：主类包的注册通道

反射实测（3.3.5）：

```text
@EnableAutoConfiguration 的元注解 = @AutoConfigurationPackage + @Import(...)
@SpringBootApplication  的元注解 = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
```

- `@AutoConfigurationPackage` 由 `@EnableAutoConfiguration` 携带（不在 @SpringBootApplication 上直接标注）；
- 作用：把**主类所在包**注册给 `AutoConfigurationPackages`（启动时容器里有 `org.springframework.boot.autoconfigure.AutoConfigurationPackages` 这个 Bean，BFPP 列表实测可见）——JPA 实体扫描、MyBatis 的 Mapper 扫描等"按启动包定位"的机制都从这里取包名；
- 这意味着：主类位置决定扫描根包，**主类放错包 = 实体/Mapper 扫不到**（经典启动事故，见坑 7）。

---

# Level 6 生产实践：条件出问题怎么排查

## 6.1 条件故障的四个症状

| 症状 | 根因方向 | 第一排查动作 |
| ---- | ---- | ---- |
| 自动配置静默不生效（Bean 缺失） | 条件不匹配（缺依赖类 / 缺属性 / 被排除） | 评估报告查 Negative |
| NoUniqueBeanDefinitionException（双 Bean） | 覆盖失败：用户定义 + 默认实现并存 | 查 OnBeanCondition 消息里"found beans" |
| 启动慢（大量类被过滤） | 类条件过多/过重（每个候选都做 ASM 扫描） | --debug 看过滤数量 |
| 配置写了不生效 | 属性条件 key 拼写 / 宽松匹配误区 | 查 containsProperty 语义（坑 4） |
| 注入失败：类型匹配不上 / Back-off 误判 | **@Bean 返回类型过宽**：`@Bean Object client()` 让条件和候选者预测变弱（16 章 P-A04，文档语义）——条件按返回类型推断、注入点按声明类型匹配，返回 Object 等于"登记表只写某设备" | 返回合适的具体公开类型（接口或实现类） |
| 条件成立了，Bean 还是创建失败 | **条件成立 ≠ 构造/绑定/初始化成功**：Condition 只决定"要不要注册"，实例化失败（构造器抛异常）、属性绑定失败、后处理失败是注册之后的独立故障点（09 章 Level 5，文档语义） | 启动日志查 BeanCreationException 根因；区分"没注册"与"注册了但创建失败"——评估报告只能证明前者 |
| 自动配置间顺序错了但依赖没错 | **@AutoConfigureBefore/After 只排自动配置之间的顺序，不替代构造器依赖**：它在 imports 处理阶段生效（03 篇 Level 3 机制），Bean 实例级依赖仍靠注入声明（16 章文档语义） | 检查注入点是否显式声明依赖；排序注解改的是候选评估顺序不是实例化顺序 |

## 6.2 三个排查工具

**工具 1：--debug 启动**（或 `logging.level.org.springframework.boot.autoconfigure=DEBUG`）：

```text
CONDITIONS EVALUATION REPORT          ← 节标题（3.3.5 反编译 ConditionEvaluationReportMessage 实证）
Positive matches:   ← 放行的配置类（含每条条件消息）
Negative matches:   ← 拒签的配置类（含"did not find..."原因）
Exclusions:         ← 被排除的配置类
Unconditional classes: ← 无条件的配置类（总是生效）
```

格式来源说明：本实验环境无日志实现（jul 关闭 + 无 slf4j 绑定），`--debug` 未直接实测；四个节标题与 `%n   %s matched:` 结构来自对 `ConditionEvaluationReportLogger` / `ConditionEvaluationReportMessage`（spring-boot-autoconfigure 3.3.5）的 javap 反编译，与工具 3 代码访问的是同一个 ConditionEvaluationReport 数据源。

**工具 2：actuator conditions 端点**：运行期访问 `actuator/conditions`（生产可观测），JSON 结构与 --debug 同源（都是 ConditionEvaluationReport）。

**工具 3：代码访问**（demo10.ReportApp 演示）：`ConditionEvaluationReport.get(beanFactory)` 直接读条件链——**自动化测试里断言"某个自动配置是否按预期让位"就用它**。

## 6.3 自定义 starter 的条件设计建议

```text
条件设计三问（写 starter 前自问）：
  1. 我的能力依赖哪些类？→ @ConditionalOnClass（类名写全限定名，防拼写错）
  2. 用户可能想替换什么？→ 默认 Bean 加 @ConditionalOnMissingBean（留让位通道）
  3. 什么行为应该可配？→ @ConditionalOnProperty(matchIfMissing=true)（默认生效、可关）
```

- **类条件能少则少**：每个 @ConditionalOnClass 都要 ASM 扫描 + 常驻评估报告噪音；
- **对称条件切换**（AopAutoConfiguration 模式）：`@ConditionalOnClass(X)` + `@ConditionalOnMissingClass(X)` 两个内层配置类互补，比一个条件里写复杂逻辑清晰；
- **条件消息要可读**：条件消息进评估报告，也是线上排查线索。

---

# 🔗 全篇因果链总图

```text
为什么"加个依赖就能用"？
  │
  ├─ 因为 Spring 预置了免签名单：META-INF/spring/*.AutoConfiguration.imports（每个 jar 可贡献）
  │     └─ ImportCandidates.load 收集（门 1）
  │           └─ 黑名单先剔除：exclude/excludeName/spring.autoconfigure.exclude（门 2）
  │                 │           └─ 记录进 ConditionEvaluationReport.Exclusions（可观测）
  │                 └─ 类条件预过滤：OnClassCondition（门 3，ASM 元数据不加载类）
  │                       └─ 拓扑排序：AutoConfigurationSorter（门 4，order 初排 + before/after 重排）
  │                             └─ 延迟导入 DeferredImportSelector（用户配置之后才处理）
  │                                   └─ 注册 + 完整条件评估（门 5）
  │                                         ├─ OnBeanCondition：REGISTER_BEAN 阶段，
  │                                         │    能看到用户已注册定义 → 让位（覆盖通道）
  │                                         ├─ OnPropertyCondition：精确 key + equalsIgnoreCase
  │                                         └─ 结果全部记录进 ConditionEvaluationReport
  │                                               └─ --debug / actuator / 代码访问（可观测）
  │
  └─ 因为评估是可观测的：Positive/Negative/Exclusions/Unconditional 四分类
        └─ 生产排查的第一现场（症状 → 报告 → 根因）
```

---

# 🏥 线上案例（3 个）

## Case 1：加了个依赖，数据源自动配置突然没了

**现象**：升级 Spring Boot 后，应用日志 `--debug` 里 DataSourceAutoConfiguration 从 Positive 变 Negative，数据源 Bean 消失。
**排查**：评估报告显示 `OnClassCondition did not find required class 'EmbeddedDatabaseType'` → 检查依赖树：某依赖升级把 `spring-jdbc` 排除掉了（demo10.ReportApp 复现了完全相同的 Negative 场景）。
**根因**：自动配置的静默降级——类条件不满足就静默关闭。
**修复**：补回 spring-jdbc 依赖；教训 = **自动配置行为变化先看评估报告，别猜**。

## Case 2：用户自定义 Bean 和默认实现同时存在

**现象**：`NoUniqueBeanDefinitionException: expected single matching bean but found 2`，一个用户的、一个自动配置的。
**排查**：评估报告里 OnBeanCondition 消息显示 `found beans of type 'xxx' userGreeter`——条件**匹配了**（找到用户 Bean），说明自动配置**没让位**。
**根因排查清单**：① 自动配置的 @Bean 有没有写 @ConditionalOnMissingBean；② 用户 Bean 是不是声明在配置类**外部**（坑 8——定义根本不注册，条件探测不到）；③ 用户 Bean 是不是由自动配置/延迟导入机制注册（处理顺序不保证在自动配置组之前，让位契约只对普通用户配置成立）。
**修复**：把 @Bean 移回配置类内部；或改用户实现类型让类型匹配更精确。

## Case 3：配置明明写了，条件却不生效

**现象**：运维配置 `DEMO10_FLAG=true`（环境变量风格），@ConditionalOnProperty(name="demo10.flag") 一直不匹配。
**排查**：demo10.PropertyConditionApp 场景 3 实测——**@ConditionalOnProperty 精确查 key，不做宽松匹配**（Boot 3 移除 RelaxedPropertyResolver）。环境变量风格 key 是 @ConfigurationProperties 的宽松绑定能力，不是条件注解的能力。
**根因**：宽松匹配语义边界混淆（勘误表第 1 条）。
**修复**：属性条件用精确 key；跨部署环境想用环境变量，配置映射层（环境变量 → 配置项）转换，或写 `@ConfigurationProperties` 前缀 + 在配置类方法上做条件。

---

# 📋 面试自查表（16 行）

| # | 问题 | 一句话答案 | 证据 |
| ---- | ---- | ---- | ---- |
| 1 | 加依赖为什么就能用？ | imports 白名单 + 延迟导入 + 条件裁决，全自动 | 门 1-5 |
| 2 | imports 文件在哪？ | `META-INF/spring/{注解全名}.imports` | ImportCandidates.load |
| 3 | 谁读它？ | AutoConfigurationImportSelector（DeferredImportSelector） | javap 3.3.5 |
| 4 | exclude 传给了谁？ | getExclusionFilter，注册前剔除 | demo10.ExclusionApp |
| 5 | 排除有记录吗？ | ConditionEvaluationReport.Exclusions | demo10.ExclusionApp |
| 6 | imports 顺序就是生效顺序吗？ | 不是，before/after 拓扑重排 | demo10.OrderingApp |
| 7 | @AutoConfiguration 有什么属性？ | value/before/beforeName/after/afterName，无 order | javap 3.3.5 |
| 8 | 类条件为什么不加载类？ | ASM 元数据检测 classpath | demo10.ClassConditionApp |
| 9 | OnClassCondition 什么时候评估？ | 导入阶段可预过滤（无 ConfigurationPhase） | javap 3.3.5 |
| 10 | OnBeanCondition 什么时候评估？ | REGISTER_BEAN（bean 定义阶段） | javap 3.3.5 |
| 11 | 条件结果记录在哪？ | SpringBootCondition.matches final 模板方法 → 报告 | javap 3.3.5 |
| 12 | 用户 Bean 为什么能覆盖？ | DeferredImportSelector 延迟 → 自动配置后处理 | demo10.OverrideApp |
| 13 | 属性条件宽松匹配吗？ | 不宽松；精确 key + 值 equalsIgnoreCase | 反编译 + demo10 场景 3/4 |
| 14 | 评估报告怎么访问？ | --debug / actuator/conditions / 代码 API | demo10.ReportApp |
| 15 | @AutoConfigurationPackage 在哪？ | @EnableAutoConfiguration 上（启动包注册） | 反射实测 |
| 16 | 自动配置类为什么延迟导入？ | 让用户配置先注册 → 条件能"看见"用户定义 | 6.1.14 源码 |

---

# 🕳️ 坑与细节（10 个）

1. **宽松匹配语义边界**：@ConditionalOnProperty 精确查 key（Boot 3 实测）；@ConfigurationProperties 才宽松绑定。环境变量风格 key 喂条件注解 = 永不匹配。
2. **条件静默不匹配**：类名拼写错 / 依赖缺失 → 自动配置静默关闭，不报错。症状是"功能没了"，第一现场是评估报告 Negative。
3. **imports 顺序 ≠ 生效顺序**：before/after 拓扑排序才是权威；排错顺序先看报告。
4. **exclude 是黑名单不是删协议**：spring-boot-autoconfigure 的 imports 不动，只是本次拒签。
5. **条件标注位置**：@ConditionalOnMissingBean 标在 @Bean 方法上 = 只让这个默认实现让位；标在配置类上 = 整个类跳过（包含所有 @Bean）。
6. **对称条件切换**（AopAutoConfiguration 模式）：正反两个类条件互补分支，评估报告里能看到"一边 Negative 一边 Positive"是正常设计。
7. **主类位置决定扫描根**：@AutoConfigurationPackage 注册主类包 → 主类放错包，实体/Mapper/组件扫描全漂移。
8. **@Bean 必须在配置类内部**（Demo 纪律）：声明在配置类外部的方法不会被当作配置方法收集——编译产物里没有工厂方法定义，@ConditionalOnMissingBean 永远探测不到，覆盖静默失败。本次实验曾因此误判机制 2 小时（实验教训：先验证最小可运行例子，再判断机制）。
9. **同包多个 @SpringBootApplication 互扫**：组件扫描会注册同包所有 @Configuration 候选（包括别的 App 的配置类），且**每个候选都会触发一次自动配置导入**——演示环境观察到评估报告排除列表出现重复项（4.6）。演示环境要隔离包；生产上启动类与其他配置类同包是常态，但**同一个包不要放两个 @SpringBootApplication**。
10. **@Bean 默认推断销毁方法**：destroyMethod 默认 `"(inferred)"`，返回对象有 public 无参 close()/shutdown() 就会被容器在关闭时无脑调用（0.6~0.8 的 15 场景实测）。业务类命名避开这两个方法名；封装第三方类时按需 `@Bean(destroyMethod = "")`。

---

# 🧾 版本勘误表

| 版本 | 行为 | 说明 |
| ---- | ---- | ---- |
| Boot 2.x | @ConditionalOnProperty 经 RelaxedPropertyResolver 支持宽松属性名匹配 | 属性条件的宽松匹配曾是真实行为（待实测具体 2.x 小版本细节） |
| **Boot 3.x（本文实测）** | @ConditionalOnProperty 用 Environment 精确 key 查找，无宽松匹配 | **语义变化**（不是代码组织变化）：RelaxedPropertyResolver 已移除；宽松匹配仅保留在 @ConfigurationProperties 绑定层 |
| Boot 2.7+ | @AutoConfiguration 注解引入（before/after 属性），自动配置类可迁移自 @Configuration | 代码组织演进 + 新注解契约 |
| Boot 3.x（本文实测） | @EnableAutoConfiguration 仍携带 @AutoConfigurationPackage；@SpringBootApplication = SpringBootConfiguration + EnableAutoConfiguration + ComponentScan | 与 2.x 的注解组合一致（待验证 2.x 具体版本细节） |
| 3.3.5（本文实测） | 内置自动配置白名单 152 行（含 Aop/DataSource/Lifecycle 等）；AopAutoConfiguration 双层结构实测 | Implementation 细节，随版本漂移 |

---

# 🃏 生产决策卡（3 张）

## Decision Card 1：什么时候该写自动配置

- **场景**：团队内多服务复用同一套组件配置（连接池、鉴权过滤器、统一对象映射）。
- **判断**：重复配置 ≥ 2 个服务 → 抽 starter；只有一个服务 → @Configuration 直接写，别过度设计。
- **差异分层决策树**（以权限切面为例：A 业务线需要 {a,b,c} 三个权限，C 业务线只要 {a,c}）：
  - 各业务线**完全一致**的部分（"必须登录"这类 a 权限）→ 进 starter 公共代码
  - 各业务线**不同**的部分（b/c 权限、校验规则、拒绝行为）→ 外置：数据层走注解参数、配置层走 resolver 实现选择、逻辑层走属性开关——**禁止写死在公共代码里**
  - 判定口诀：**能参数化的进注解，能选择的进配置，能开关的进属性；公共代码里不允许出现业务线分支**
- **Mechanism → Decision**：

```text
starter 结构（01 篇 demo06 已展示）：
  starter 模块（依赖管理，无代码）
  ├─ auto-configuration 模块：@AutoConfiguration + 条件 + @Bean
  └─ META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports（白名单登记）
```

- **禁止决策**：禁止把无条件的 @Bean 塞进自动配置（Unconditional class 无法被用户覆盖且必然生效）；禁止条件依赖"运行时状态"（自动配置在注册阶段裁决，运行时才确定的值无法参与）；**禁止高风险能力默认生效**——鉴权/安全类 starter 必须用 `@ConditionalOnProperty(enabled, matchIfMissing=false)` 显式开启（红线）：**加依赖 ≠ 激活，激活必须是你写的那一行配置**。
- **爆炸半径控制（三道防线）**：① 公共代码零业务分支；② 差异全部外置（注解参数/配置实现/属性开关）；③ 默认关闭、显式开启——业务线误引了 starter 也默认不激活，不会静默改变行为。
- **验收指标**：加依赖即生效（单测断言 Bean 存在）；用户覆盖通道测试（断言让位）；评估报告噪音可控（条件数量/启动日志可读性）；**红线验证**（未配 enabled 时断言 Bean 不存在）。

## Decision Card 2：用户覆盖该用哪个通道

- **场景**：要替换自动配置提供的默认实现。
- **判断**：能自己写 @Bean 就写 @Bean（通道 3，最可控）；只想去掉某个自动配置用 exclude；想按环境开关用属性条件；想彻底关自动装配用总开关。
- **Mechanism → Decision**：

```java
@SpringBootApplication
static class BootConfig {
    @Bean
    public Greeter userGreeter() { return new Greeter("用户定义"); }  // 让自动配置让位
}
```

- **禁止决策**：禁止为覆盖而加 exclude（exclude 一刀切，无法细粒度）；禁止两个用户 Bean 同类型（NoUniqueBeanDefinitionException 是自找的）。
- **验收指标**：单测断言注入的是用户版本；评估报告显示 OnBeanCondition 不匹配（让位有据可查）。

## Decision Card 3：自动配置条件怎么写

- **场景**：自定义 starter 的条件设计。
- **判断**：能力依赖类 → @ConditionalOnClass；可替换对象 → @ConditionalOnMissingBean；功能开关 → @ConditionalOnProperty(matchIfMissing=true)；互斥实现 → 对称条件切换（正反两个内层配置类）。
- **Mechanism → Decision**：

```java
@AutoConfiguration
public class ServiceAutoConfig {
    @Bean("demo10-default-greeter")
    @ConditionalOnMissingBean(Greeter.class)
    public Greeter defaultGreeter() { return new Greeter("默认"); }
}
```

- **禁止决策**：禁止类条件与 bean 条件混用导致双阶段语义混乱；禁止在条件消息里堆砌实现细节（评估报告要可读）。
- **验收指标**：Positive/Negative 各自有明确理由（消息可读）；覆盖场景单测覆盖；--debug 输出一次能讲清"为什么生效/不生效"。

---

<a id="cross-language"></a>

## 🌍 跨语言视角

| 语言/生态 | 做法 | 对应 Spring 的什么 | 账 |
| ---- | ---- | ---- | ---- |
| Java（JDBC） | 驱动自动发现（ServiceLoader 的 META-INF/services） | imports 白名单 + 加载 | 只发现不裁决（无能力条件） |
| Java（SPI 演进） | ServiceLoader → Spring Boot imports + 条件 | 名单 + 逐条条件 | 裁决自动化 vs 隐式行为需评估报告 |
| Go | build tags（编译期）+ init() 注册 | 条件在编译期完成 | 无运行期降级，编译期静态 |
| Rust | cargo features（编译期特性开关） | 属性条件 | 编译期确定，无法运行期切换 |
| 前端（bundle） | tree-shaking + 特性探测（navigator API 检查） | 类条件（能力检测） | 能力检测在浏览器端，无集中报告 |
| K8s | label selector / node affinity | 条件 + 排除 | 声明式调度条件，可观测性强（kubectl describe） |

**底层思想是否一致？**

一致的部分：

- **能力检测优先于加载**：所有生态都在"加载前先问 classpath/环境/能力"——ASM 元数据、编译期特性、运行时 feature detection 是同一思想。
- **名单 + 条件 = 可插拔组合**：白名单（哪里来）+ 条件（什么情况生效）分离，是 SPI 设计的普遍范式。
- **排除通道是运维的最后一手**：任何自动机制都要留"手工关掉"的出口（exclude、features off、label anti-affinity）。

不一致的部分：

- **Spring 的两段式过滤**（导入前类条件 + 注册时 bean 条件）是"先减后决"的独特设计——编译期生态没有"运行时让位"，前端没有集中裁决记录。
- **评估报告是 Spring 的独有资产**：Go/Rust 的条件在编译期，失败即编译错误；Spring 运行期静默降级，必须靠报告补可观测性——**静默 + 可观测是一对设计**，不能只取一半。

---

<a id="series-index"></a>

## 🧭 系列索引（00 篇为入口）

| 篇 | 主题 | 主线角色 | 比喻 | 本系列位置 |
| ---- | ---- | ---- | ---- | ---- |
| 00 | 容器如何创建对象 | 一个 Bean 的一生 | 地铁线路图 | 地基：创建链因果全通（15 demo 实测） |
| 01 | 框架整合 + 配置体系 | 待接入的框架 | 海关通关 | 在 00 的创建链上开扩展点；Level 4 完整展开配置体系 |
| 02 | 事件机制与容器通信 | 一次业务事件 | 公告栏广播 | 发布-订阅通信机制；早期事件/事务事件/启动全景（demo08×5 + demo09 实测） |
| **03（本篇）** | 自动装配深挖 | 一个 starter 的生效过程 | 免签通道 | 候选收集/排除/排序/条件家族/覆盖通道/评估报告（demo10×8 实测） |
| 04（已完成） | Web 请求链路与运行时刻 | 一次 HTTP 请求 | 外卖配送 | 原 00 篇 Level 7 移入扩写（demo11×4 实测（含 WebFlux 双跑法）） |
| 05（已完成：demo12×4 实测） | 事务与数据层 | 一笔数据库事务 | 记账员 | 事务边界与数据层（含事务消息衔接） |
| 06（已完成：demo13×8 实测） | 横切面与 AOP | 一次方法调用 | 关卡哨兵 | 代理机制本体与切面体系（JDK/CGLIB 双分支实测） |
| 07（规划） | 生产实践 | 线上一次故障 | 急诊室 | 收束 |

---

# ✅ Final Review Checklist

- [ ] 是否解释了为什么存在？（免签名单免去手工配置 → 自动装配；两段式过滤先减后决；延迟导入保证用户优先）
- [ ] 是否说明旧方案为什么失败？（手工 @Configuration 每服务重复配置；XML 时代框架整合靠手工 import）
- [ ] 是否形成完整因果链？（名单 → 排除 → 类条件预过滤 → 拓扑排序 → 延迟导入 → 注册裁决 → 评估报告可观测）
- [ ] 是否区分规范和实现？（@ConditionalOn* 语义、让位契约、@AutoConfiguration 属性为 Specification；ImportCandidates 路径、排序器结构、OnClassCondition/OnBeanCondition 类继承、OnPropertyCondition 匹配实现为 3.3.5 Implementation）
- [ ] 是否区分语义变化与代码组织变化？（Boot 3 属性条件去宽松匹配 = 语义变化；@AutoConfiguration 注解引入 = 新契约 + 代码组织）
- [ ] 代码实例是否全部实测？（demo10×8 输出原样引用，可复跑；条件匹配实现、注解结构经 javap/反射验证）
- [ ] 是否包含 Trade-off？（静默降级 vs 报错失败——靠评估报告补偿；类条件预过滤省开销 vs 二阶段语义复杂度；自动配置 vs 手工配置的维护成本）
- [ ] 是否能指导生产决策？（3 张决策卡：何时写 starter / 覆盖通道选择 / 条件设计三问）
- [ ] 是否存在未经证明的数字？（无编造 benchmark；2.x 宽松匹配细节与内置白名单行数标注待实测）
- [ ] 是否只有一个比喻？（免签通道）是否只有一个主线角色？（一个 starter 的生效过程）
- [ ] 随机抽查断言：@ConditionalOnProperty 无宽松匹配（javap 反编译 + demo10 场景 3）、OnBeanCondition REGISTER_BEAN（javap）、ASM 不加载类（demo10.ClassConditionApp）、before 拓扑排序（demo10.OrderingApp）、让位契约（demo10.OverrideApp）、@EnableAutoConfiguration 注解组合（反射实测）、AopAutoConfiguration 双层对称条件（demo10.ReportApp）、@AutoConfigureOrder 默认值 0（反射实测）、配置通道 spring.autoconfigure.exclude 与注解通道合并（实测）、报告 source 粒度与 Exclusions List 重复（实测）——均有证据来源。
