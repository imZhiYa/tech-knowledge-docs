# 🛃 SpringBoot 第三方框架整合与配置体系（系列 01）

> 本篇文章回答两个问题：
> 1. **一个待接入的框架**（以 MyBatis 的 Mapper 接口为主角）如何穿过报关（starter）、验货（@Conditional）、登记入册（Registrar）、贴标换装（FactoryBean 代理），变成你能 `@Autowired` 的 Bean？
> 2. **配置体系完整展开**（Environment / PropertySource / Binder / 配置中心刷新）——框架参数从哪来？为什么 @Value 不刷新？Apollo 的刷新到底做了什么？
>
> 前置：00 篇的创建链（BeanDefinition 注册表、生命周期、注入解析链）。本篇是"在创建链上开扩展点"。

---

# ⚠️ 版本与证据边界

| 维度 | 本文承诺 |
| ---- | ---- |
| 代码实证 | `knowledge/springboot/experiments/code/` 下的 demo（demo05×3、demo06、demo07、demo01.ServiceLoaderApp），本机实测输出原样引用 |
| 实测环境 | macOS + JDK 21.0.11（Azul Zulu）+ spring-context **6.1.14** + spring-boot **3.3.5**（Boot 3.3.x） |
| Specification | @Import 语义、BeanPostProcessor 回调时机、FactoryBean 契约、@Conditional 语义、Environment 抽象、宽松绑定规则 |
| Implementation | AutoConfiguration.imports 加载链、FactoryBean 类型预测机制、config data 加载、Binder 行为、条件评估报告（标注 3.3.5/6.1.14） |
| 待验证 | 真实 MyBatis/Dubbo/Kafka/Apollo starter 的源码细节与类名（本机未下载真实 starter，以官方文档/源码路径标注**待验证**）；demo07 是其同构最小实现 |
| 未覆盖 | 不承诺任何性能数字；Web 容器（04 篇）、事务（05 篇） |

---

# 🏷️ 关键词

starter | 自动装配 | AutoConfiguration.imports | @Conditional | Registrar | BeanPostProcessor | FactoryBean | 动态代理 | Environment | PropertySource | 优先级堆叠 | profile | config data | Binder | @ConfigurationProperties | 配置中心刷新

---

# 🗂️ 目录

- Level 1 为什么框架不能直接 new：SPI 的历史账
- Level 2 自动装配：加个依赖为什么就能用
- Level 3 扩展点：注册、注入、代理（+ FactoryBean 类型预测）
- Level 4 配置体系：参数从哪来（完整展开）
- Level 5 五案例对号入座：MyBatis / Dubbo / Redis / MQ / Apollo
- Level 6 生产实践：自定义 starter 与排查方法论
- 全篇因果链总图
- 线上案例（3 个）
- 面试自查表
- 坑与细节（10 个）
- 版本勘误表
- 生产决策卡（3 张）
- 跨语言视角

---

# 🏭 全文唯一比喻：海关通关

一个外国货物要进国内销售，必须走完整套海关流程——这是"第三方框架进 Spring 容器"的唯一比喻：

- **报关**（starter）：先报清单（加依赖），清单本身是标准模板。
- **验货条件**（@Conditional）：查资质（类在不在 classpath）、查报关参数（属性配没配）——条件不满足直接扣下。
- **登记入册**（BeanDefinition 注册）：验货通过后登记台账，货才"存在"。
- **贴标换装**（FactoryBean/代理）：把散货贴成你能用的品牌包装——你拿到的是包装，不是散货。
- **报关参数**（Environment/配置绑定）：税费、口岸、时效参数从哪来——由报关单（配置）决定。

```text
现实世界（海关）                         技术世界（SpringBoot 整合）
出口商想卖货进国内                    框架想进 Spring 容器
   │ 先报清单（报关单）                    │ 加 starter 依赖
   ▼                                      ▼
海关验货（检疫/资质条件）          ←→   @Conditional 评估（类/属性/Bean）
   │ 通过 → 登记台账                       │ 通过 → 注册 BeanDefinition
   ▼                                      ▼
货物贴标/包装（可售形态）          ←→   FactoryBean/动态代理（可用形态）
   │ 报关参数（口岸/税费）                 │ 配置绑定（@ConfigurationProperties）
   ▼                                      ▼
放行 → 消费者买到手               ←→   可注入 → 业务代码直接用
```

**全文不再出现第二个比喻。**

---

# 正文：六个 Level 打穿"框架整合 + 配置体系"

<a id="level-1"></a>

## Level 1 为什么框架不能直接 new：SPI 的历史账

### 前置知识关卡

* [ ] 知道"框架 = 管理你的一部分运行时"（连接池、RPC、消息消费）
* [ ] 知道 00 篇的创建链（注册表 → 反射 → 单例池）
* [ ] 知道动态代理（JDK 接口代理）

### 1.1 Why：new 三账，SPI 只还了一半

徒弟：

> 用 MyBatis 不就是 `new SqlSessionFactory(...)` 然后调 `getMapper()` 吗？为什么要容器来整合？

老陈：

> 你自己 new 确实能跑通 Demo，但是留下三笔账——
> 第一，**生命周期账**：连接池要开要关，散落在业务代码里没人管；
> 第二，**配置账**：数据库地址、连接数、超时——每个框架一套配置源，系统越大越乱；
> 第三，**协作账**：框架之间要互相配合（MyBatis 要用 Spring 事务、Dubbo 服务里要注入 Redis），各自 new 出来的对象**根本接不上**。
> 结论：框架的"对象"必须和 Spring 的 Bean 统一——这就是"整合"的全部意义。

| 方案 | 解决什么 | 留下什么问题 |
| ---- | ---- | ---- |
| 业务代码直接 new 框架对象 | Demo 能跑 | 生命周期、配置、协作三笔账 |
| JDK SPI（ServiceLoader） | 发现实现类（META-INF/services） | 只解决"发现"，不解决生命周期/注入/配置/代理 |
| Servlet 3.0 SCI | 容器启动时回调应用装配 | 只覆盖 Web 容器场景 |
| Spring 扩展点（@Import/BPP/FactoryBean） | 框架对象 = Bean：生命周期/注入/配置全接管 | 学习成本：四个扩展点（本篇主题） |

**为什么 SPI 会"死"**：`ServiceLoader.load(Xxx.class)` 能发现实现类并 new 出来，但它只回答"**有哪些实现**"。发现之后呢？实例怎么初始化、依赖怎么注入、参数从哪来、要不要代理——SPI 一概不管。**SPI 解决"找到"，Spring 解决"养好"**——整合需要的是后者。

### 1.2 代码实证：SPI 发现 ≠ 管理（demo01.ServiceLoaderApp）

完整代码：`experiments/code/src/demo01/ServiceLoaderApp.java` + `src/demo01/spi/`（接口 + 两个实现）+ `src/META-INF/services/demo01.spi.Greeter`。

实测输出（`./run.sh demo01.ServiceLoaderApp`）：

```text
[SPI 发现] 找到 2 个实现: A, B
[SPI 边界] 能 new 出来，但无人管理生命周期/注入/配置/代理
[Spring 对比] 容器注册后 @Autowired 可用: hello = A
```

**这四行输出就是 Level 1 的全部因果**：
- SPI 靠 `META-INF/services/<接口全限定名>` 文件列出实现类，`ServiceLoader` 反射实例化——**发现**成立；
- 但实例化后没有容器：没有注入、没有生命周期、没有配置、没有代理——**管理**缺位；
- 同样的实现类放进 Spring 容器后，`@Autowired` 直接可用——**管理的价值肉眼可见**。

### 1.3 What：三种"发现机制"的边界

| 机制 | 发现谁 | 管理什么 | 谁在用 |
| ---- | ---- | ---- | ---- |
| JDK SPI（ServiceLoader） | META-INF/services 里的实现类 | 无（只管 new） | JDBC DriverManager、SLF4J |
| Spring `@Import`/扫描 | 配置类、组件 | 完整 Bean 生命周期 | Spring 生态本体 |
| Boot 自动装配（imports 文件） | 自动配置类白名单 | 条件评估 + Bean 生命周期 | 所有 starter |

### 1.4 How：ServiceLoader 的机制（JDK 实现）

```text
ServiceLoader.load(Greeter.class)
  └─ 读 META-INF/services/demo01.spi.Greeter（类名白名单）
      └─ forName + newInstance 实例化
      └─ 按需惰性加载（iterator 逐个 new）
```

**最反直觉的一步**：SPI 的"服务提供者文件"（META-INF/services）和 Boot 的"自动配置候选文件"（AutoConfiguration.imports）长得几乎一样——**都是"类名白名单"**。区别只在读取之后：SPI 读完就 new，Boot 读完还要"验货"（@Conditional）。下一层的主角就是这份验货清单。

### 1.5 Transfer

- **"翻译层"模式**：任何"把别人的对象接进自己的生命周期"的系统（K8s 的 CRD+Controller、Eclipse 插件、浏览器扩展）都是"注册 + 钩子 + 包装"三件套。
- **发现 ≠ 管理**：选型时先问"它是否提供管理，还是只提供发现"——SLF4J 只发现不管理，Spring 管理一切，代价是魔法需要可观测性补偿。

**本层留下的账**：三笔账清楚了，但**谁来做第一个动作**？加个依赖就能用，谁把"验货流程"启动的？——这逼出下一层：自动装配。

---

<a id="level-2"></a>

## Level 2 自动装配：加个依赖为什么就能用

### 前置知识关卡

* [ ] 知道 00 篇 Level 1.5 的 refresh 主路径（invokeBeanFactoryPostProcessors）
* [ ] 知道 @Import 与 DeferredImportSelector（00 篇 Level 2.7 四路注册）
* [ ] 知道 @Conditional 是"注册前先问条件成立吗"

### 2.1 Why：手动 @Import 的账

徒弟：

> 为什么我在 pom 里加一行依赖，Redis 就能 `@Autowired RedisTemplate` 了？谁把代码写进我项目的？

老陈：

> 没人往你项目里写代码——是**自动配置类**在启动时被动态注册进容器。
> 它藏在 starter 的 jar 里（在 classpath 上，但你不 import 它）。
> Boot 启动时从固定路径读候选清单，逐类问条件，条件通过就把它当配置类注册。
> 流程：**依赖进 classpath（报关）→ 候选清单被发现（AutoConfiguration.imports）→ 条件评估（验货）→ 注册配置类（放行）**。

| 方案 | 解决什么 | 留下什么问题 |
| ---- | ---- | ---- |
| 用户手动 @Import 每个配置类 | 能工作 | 每接一个框架写一行 import，违背"加依赖就能用" |
| 自动扫描所有 @Configuration | 不用写 import | 无法控制：你的包和框架的包全混进来，冲突与启动膨胀 |
| AutoConfiguration.imports 白名单 + @Conditional | 只注册"条件成立"的配置 | 隐式：必须靠条件评估报告才能看到"为什么" |

### 2.2 代码实证：最小 Boot 工程（demo06.MinimalBootApp）

完整代码：`experiments/code/src/demo06/MinimalBootApp.java` + `src/autoconfig/DemoAutoConfiguration.java` + `src/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` + `src/application.properties`。

这个"最小工程"模拟了一个 starter 的全部要素：
- **自动配置类放在扫描包之外**（`autoconfig` 包）——它只能靠 imports 白名单加载，证明"加依赖就能用"不是组件扫描；
- imports 文件内容就是一行：`autoconfig.DemoAutoConfiguration`；
- 自动配置类带 `@ConditionalOnProperty(name = "demo.enabled", havingValue = "true")`；
- application.properties 里 `demo.enabled=true` + 一个 `@ConfigurationProperties` 前缀 `app.order`。

实测输出（`./run.sh demo06.MinimalBootApp`）：

```text
[imports 加载] 自动配置类注册的 bean demoFlag 存在? true, 值=auto-config-ok
[config data] application.properties 已加载进 Environment: true（ConfigData 把文件变成 PropertySource 塞进列表）
[自动绑定] app.order.callback-url=https://pay.example.com/cb, maxRetries=3
[条件否决] --demo.enabled=false 时 demoFlag 不存在? true
```

**四行输出对应自动装配的四件事**：
1. imports 白名单加载成功（类在扫描包之外，只能靠白名单）；
2. config data：Boot 把 application.properties 加载成 PropertySource 塞进 Environment（Level 4 的主角之一，这里先亮个相）；
3. 自动绑定：`@ConfigurationProperties(prefix="app.order")` 的类被自动绑定（Binder 出场）；
4. **条件否决**：`--demo.enabled=false`（命令行参数优先级最高）→ 自动配置类不注册——"没生效"是条件设计的本意，不是故障。


### 2.3 What：加载链（Boot 3.3 Implementation）

```text
classpath 上的每个 starter 提供：
  META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  （每行一个自动配置类全限定名；Boot 3 只认这个文件，spring.factories 已移除——语义变化）

启动时 ConfigurationClassParser 处理配置类：
  @SpringBootApplication
    └─ @EnableAutoConfiguration
         └─ @Import(AutoConfigurationImportSelector.class)   ← DeferredImportSelector
              ├─ ImportCandidates.load(...)：收集全部 imports 候选
              ├─ AutoConfigurationSorter 排序
              │    （@AutoConfigureBefore / @AutoConfigureAfter / @AutoConfigureOrder）
              ├─ 逐个评估 @Conditional（ConditionEvaluator）
              ├─ 评估结果写入 ConditionEvaluationReport
              └─ 通过的 → 注册为配置类（后续正常走 @Bean/生命周期）
```

- **DeferredImportSelector 为什么是"延迟"的**：等用户的配置类（@Configuration、@Bean）先处理完，自动配置再登场——保证"用户先定义、自动配置后补缺"（`@ConditionalOnMissingBean` 依赖这个顺序才能生效）。
- **条件评估报告**（探照灯）：每个候选的记录——Positive matches（通过）/ Negative matches（否决 + 原因）/ Exclusions / Unconditional classes；actuator `conditions` 端点暴露。

> 📌 最容易被误解的一点：**自动配置类是"有条件的 @Configuration"**——不是"框架已经配好了"，而是"框架把能配的都预置好，等你的环境满足条件时启用"。满足条件 = 依赖在 classpath + 属性可配 + 容器缺 Bean。

> 📌 **Starter 是"依赖意图"，不是"运行事实"**（24 章 15 章对照，Boot 4.1 基线文档语义）：starter 只是场景化依赖集合——`spring-boot-starter-web` 不等于"Web 已配好"，`starter-aop` 不等于"AOP 已生效"。真正干活的是 imports 候选 + 条件评估（上面整条链）。排查"加了 starter 却没效果"先验证：① 依赖是否真的进 classpath（`mvn dependency:tree`）② imports 文件是否存在 ③ 条件是否满足（conditions 端点）——三层任一层断，运行事实就不存在。

### 2.4 How：条件三问（@Conditional 家族的三个主问题）

```text
加 starter ──▶ imports 候选 ──▶ 排序 ──▶ 条件评估
                                          ├─ @ConditionalOnClass：类在 classpath 吗？（依赖进来了吗）
                                          ├─ @ConditionalOnProperty：属性配了吗？（开关/参数）
                                          ├─ @ConditionalOnMissingBean：容器里缺这个 Bean 吗？（覆盖通道）
                                          │    通过 → 注册配置类 → @Bean 创建
                                          └─ 否决 → Negative matches（可查询）
```

**最反直觉的一步**：自动配置类"没生效"通常不是 bug，而是**条件设计的本意**。

徒弟：

> 我加了 Redis starter，RedisTemplate 却注入失败，是不是自动装配坏了？

老陈：

> 先别怀疑框架——打开 conditions 端点，看 `RedisAutoConfiguration` 那行是 Positive 还是 Negative。
> 否决原因通常写着三件事之一：
> ① `@ConditionalOnClass` 失败：依赖没进来（starter 没加对/版本冲突）；
> ② `@ConditionalOnProperty` 失败：开关属性没配；
> ③ `@ConditionalOnMissingBean` 失败：你已经定义了自己的 Bean（这是覆盖生效，不是故障）。
> **"没生效"的排查流程 == 读条件评估报告**，这比猜源码快十倍。

### 2.5 Transfer

- **白名单 + 条件评估**（候选多、生效少、可观测）：K8s 的准入控制（Admission Webhook）、规则引擎的规则集、Feature Flag 系统全是同一结构。
- **用户先、自动后**：DeferredImportSelector 的延迟是"默认值必须让位给用户显式声明"的机制化——任何框架的默认值系统都要回答"用户覆盖的通道是什么、顺序是什么"。
- **失败要可解释**：Negative matches 的"原因列"是工程化关键——自动化系统的否决必须留下证据，否则排查就是猜。

**本层留下的账**：自动装配决定"注册哪些配置类"，但配置类内部要**把框架对象翻译成 Bean**——注册、注入、代理、配置四个扩展点怎么用？这逼出下一层。

---

<a id="level-3"></a>

## Level 3 扩展点：注册、注入、代理（+ FactoryBean 类型预测）

### 前置知识关卡

* [ ] 知道 BeanPostProcessor 回调时机（00 篇 Level 2：BPP.before 在 @PostConstruct 前、BPP.after 是 AOP 代理位）
* [ ] 知道 @Import 的三种用法（普通类/Selector/Registrar，00 篇 Level 2.7）
* [ ] 知道 JDK 动态代理与 InvocationHandler

### 3.1 Why：@Bean 解决不了的三笔新账

徒弟：

> 自动装配把配置类注册了，然后呢？配置类里写 @Bean 不就行了吗？

老陈：

> @Bean 能解决一部分，但留下三个新账——
> 第一，**批量注册账**：MyBatis 有几百个 Mapper 接口，难道每个写一个 @Bean？
> 第二，**注解识别账**：`@DubboReference` 写在字段上，@Bean 根本看不见它，谁去扫描、谁去注入？
> 第三，**接口代理账**：Mapper 是接口，没有实现类——@Bean 返回什么？必须生成一个"动态实现"。
> 这三个账分别由三个扩展点回答：**Registrar（批量注册）、BeanPostProcessor（注入）、FactoryBean（代理）**，加上配置绑定（Level 4 完整展开），就是四扩展点。

| 扩展点 | 解决什么账 | 怎么用 | 类比（海关） |
| ---- | ---- | ---- | ---- |
| @Import + ImportBeanDefinitionRegistrar | 批量/动态注册 BeanDefinition | 循环注册 | 登记台账（批量） |
| BeanPostProcessor | 识别自定义注解并注入/包装 | 扫描字段上的 @Xxx | 分拣员识别包裹标签 |
| FactoryBean | 接口无实现 → 返回代理 | getObject() 返回代理 | 贴标换包装 |
| Environment + @ConfigurationProperties | 参数从哪来 | 属性绑定到配置类 | 报关参数 |

### 3.2 代码实证：仿 MyBatis 最小翻译（demo07.TranslatorApp）

完整代码：`experiments/code/src/demo07/TranslatorApp.java`。这个 demo 是 **MyBatis 的同构最小实现**——三个扩展点全部走一遍：

- `@EnableMapperScan`（@Import(MapperScannerRegistrar.class)）→ Registrar 批量注册两个 MapperFactoryBean；
- `MapperFactoryBean implements FactoryBean` → getObject() 返回 JDK 动态代理，方法调用被 `SqlInvocationHandler` 拦截并"翻译"成 SQL 执行（MapperProxy 的同构）；
- `DubboRefBeanPostProcessor implements InstantiationAwareBeanPostProcessor` → 在 populateBean 阶段扫描 @DubboRef 字段并注入（@DubboReference 机制的同构）。

实测输出（`./run.sh demo07.TranslatorApp`）：

```text
[Registrar] 批量注册 MapperFactoryBean: userMapper, orderMapper
[FactoryBean] @Autowired userMapper 拿到的是代理: true（类名含 $Proxy14）
[翻译] UserMapper.findById(1) -> 执行SQL: select * from user where id = 1 -> 返回 User(id=1)
[翻译] OrderMapper.findById(9) -> 执行SQL: select * from order where id = 9 -> 返回 Order(id=9)
[&name] getBean("&userMapper") 返回工厂本身: true
[BPP] @DubboRef 字段被识别并注入: remote-ref-injected
```

**六行输出 = 三个扩展点各钉死一件事**：
1. **Registrar**：一条注解批量注册 N 个工厂——"几百个 Mapper 不用写几百个 @Bean"的全部真相；
2. **FactoryBean**：`@Autowired UserMapper` 拿到的是代理（类名 `$Proxy14`），调用时由 InvocationHandler 拦截翻译——**接口没有实现类却能注入**的全部真相；`&userMapper` 前缀能取到工厂本身（00 篇 1.6 决策树分支②的实战形态）；
3. **BPP**：自定义注解 @DubboRef 被 BPP 在属性填充阶段识别并注入——"@Bean 看不见字段注解"的破法。

### 3.3 深挖：FactoryBean 的类型预测（6.1 Implementation，本文最容易被忽略的坑）

demo07 里有一个值得单独钉死的细节：**Registrar 注册 MapperFactoryBean 时，必须给 BeanDefinition 设置 targetType**（`bd.setTargetType(ResolvableType.forClass(mapper))`）——否则 `@Autowired UserMapper` 会报 "No qualifying bean of type ... available"。

原因（6.1.14 实测 + 反编译源码验证）：

```text
@Autowired 按类型解析 → getBeanNamesForType(UserMapper)
  → 对未创建的 FactoryBean：不能为了问类型就把它创建出来（allowEagerInit=false 时）
  → 只能"预测类型"（predictBeanType），预测依据按优先级：
     ① BeanDefinition.targetType（显式设置）        ← demo07 的做法
     ② 工厂方法返回类型（@Bean 方法返回 FactoryBean<X>）
     ③ FactoryBean 实现类的泛型签名（FactoryBean<X> 的 X）
     ④ 全部落空 → 无法预测 → 该 Bean 不参与 byType 匹配 → NoSuchBeanDefinitionException
```

**为什么 MyBatis 能用**：mybatis-spring 注册 MapperFactoryBean 时同样依赖类型预测（其 MapperFactoryBean 的泛型结构与 targetType 配合）；**写自定义 FactoryBean 时最容易踩的坑就是"按名字 getBean 能用、按类型注入找不到"**——就是类型预测落空了。

> 📌 反直觉结论：**FactoryBean 的名字不是类型解析的通行证**。按类型找 Bean 时，未创建的 FactoryBean 靠"预测"而不是"问"——预测机制不知道你的 getObject() 返回什么。

### 3.4 How：三扩展点协作时序

```text
自动配置类（Level 2 注册的）
  ├─ @Import(Registrar) ──▶ 批量注册 MapperFactoryBean（每接口一个，targetType 设置好）
  ├─ @Bean 配置类 ──▶ 绑定 @ConfigurationProperties（参数，Level 4）
  └─ 注册 BPP ──▶ 业务 Bean 填充时注入 @DubboRef 代理

业务 Bean 创建（00 篇 doCreateBean）：
  实例化 → populateBean（BPP 扫描注解注入 / @Autowired 解析到 FactoryBean 产品）
        → initializeBean → BPP.after（AOP 代理）
```

**最反直觉的一步**：**BPP 的注册顺序决定成败**——BPP 必须在"业务单例创建"之前注册完，否则业务 Bean 已经创建，注解扫描注入就错过了（00 篇 2.8 坑：BPP 也是 Bean，先注册后创建）。

### 3.5 Transfer

- **"批量注册 + 注解识别 + 代理翻译"三件套**是框架整合的通用骨架：看到任何 starter，找它的 Registrar（谁注册）、BPP（谁注入）、FactoryBean/代理（谁翻译）就拆透了。
- **接口动态实现 = 代理翻译员**：MyBatis（SQL）、Dubbo（RPC）、Retrofit（HTTP）、Feign（HTTP）全是"接口 + 动态代理"——"定义接口，代理翻译，调用方无感"是 Java 生态最强的抽象模式。
- **类型预测 = 契约显式化**：给 BeanDefinition 设 targetType，是把"getObject() 返回什么"的隐含事实显式告诉容器——凡是"工厂造物"的体系，都要回答"别人按类型找它时，它宣称自己是什么"。

**本层留下的账**：注册/注入/代理齐了，但第四个扩展点"配置"还没展开——`@ConfigurationProperties` 的绑定、`@Value` 的解析、配置中心的刷新，全都建立在 **Environment** 之上。下一层完整展开：配置体系。

---

<a id="level-4"></a>

## Level 4 配置体系：参数从哪来（完整展开）

> 00 篇只给了"优先级"的结论（System properties > OS env > 文件），本层把机制完整展开：Environment 是什么、PropertySource 怎么堆叠、Boot 怎么加载文件、Binder 怎么绑定、配置中心刷新做了什么。

### 前置知识关卡

* [ ] 知道 00 篇 Level 3.5（@Value 的 ${} 由 PropertySourcesPlaceholderConfigurer 解析）
* [ ] 知道 00 篇 Level 3.6 场景 E（同名 key 高优先级赢）
* [ ] 知道 profile 概念（dev/prod 配置切换）

### 4.1 Why：每个框架一套配置源，系统越大越乱

徒弟：

> 配置不就是读 properties 文件吗？为什么 Spring 要搞出一个 Environment 来？

老陈：

> 直接读文件确实能跑，但是留下三个账——
> 第一，**来源账**：配置可能来自 -D 参数、OS 环境变量、application.properties、配置中心——**同一把钥匙要开四把锁**；
> 第二，**冲突账**：同一 key 多处配置，谁赢？没有统一规则就是"看心情"；
> 第三，**变更账**：配置中心改了值，正在运行的进程怎么感知？没有统一入口就无从谈起。
> Environment 就是回答这三个账的**统一视图**：所有配置来源排成一个有序队列，getProperty 按顺序查——第一个命中的就是赢家。

| 方案 | 解决什么 | 留下什么问题 |
| ---- | ---- | ---- |
| 每个模块自己读文件 | 单模块能跑 | 来源散落、冲突无规则、变更无感知 |
| System.getProperty / getenv 混用 | 读系统配置 | 代码里到处分支判断，测试没法注入 |
| Environment（有序 PropertySource 队列） | 统一视图 + 优先级 + 可插拔来源 | 队列顺序规则要学；来源多了排查有成本 |

### 4.2 What：Environment = 有序 PropertySource 列表 + profile 维度

```text
Environment
  ├─ propertySources：有序的 PropertySource 列表（getProperty 按顺序查，先命中先返回）
  │     PropertySource = 一个命名的键值来源（系统属性 / 环境变量 / 文件 / 配置中心）
  └─ activeProfiles：激活的 profile 集合（第二个维度：哪些来源/Bean 被激活）
```

**默认顺序（Boot 2.4+ 至今一致，含 3.3.5；从低到高，后者覆盖前者）**：

| # | 来源 | 说明 |
|----|----|----|
| 1 | 默认属性（`SpringApplication.setDefaultProperties`） | 最低 |
| 2 | `@PropertySource` 注解 | 仅高于默认属性；且 refresh 时才进 Environment（太晚，`logging.*`/`spring.main.*` 管不了） |
| 3 | Config data（application.properties/yml 文件） | 文件内部再分 4 层：jar 内普通 < jar 内 profile < jar 外普通 < jar 外 profile |
| 4 | `RandomValuePropertySource`（`random.*`） | 只有 random 前缀的随机值 |
| 5 | OS 环境变量（`SERVER_PORT`） | |
| 6 | Java System properties（`-Dserver.port`） | 高于 env |
| 7 | JNDI 属性（java:comp/env） | 容器部署场景 |
| 8 | ServletContext init 参数 | |
| 9 | ServletConfig init 参数 | |
| 10 | `SPRING_APPLICATION_JSON` | 第 2 高优先级，仅次命令行 |
| 11 | 命令行参数（`--server.port`） | 最高（非测试场景） |

> 证据：Spring Boot 官方文档 "Externalized Configuration"（现行版 docs.spring.io/spring-boot/reference/features/external-config.html 与 3.3.6 版顺序一致，2026-08-07 检索）；旧版对照——Boot 2.1.7 文档中 `@PropertySource` 同样倒数第二低（第 16/17 位），但文件层是"profile 外 > profile 内 > 普通外 > 普通内"插在 Random 之后（**语义变化**：2.4 引入 Config Data 处理，文件排序改为现行版）。
>
> ⚠️ 别把 Binder 的适配视图 `configurationProperties`（名字相似的 `ConfigurationPropertySourcesPropertySource`）当成配置来源——它不是一层来源，是 Binder 对整个 Environment 的只读包装。

**配置中心在哪？——不在表里，插在表头上**：Apollo/Nacos/Spring Cloud Config 的来源不是 Boot 原生层的某一位，而是启动期（initializer 阶段）`addFirst` **插到队列最前面**——即**高于命令行参数**（第 11 层之上）。4.3 实测的 `apollo-config-new > apollo-config > systemProperties > ...` 就是这一位次；所以配置中心能覆盖一切本地配置，而"本地调试用 `--xxx` 想压过配置中心"是压不过的——这是 4.7 配置中心章节与本节衔接的钉。

**三个最容易记反的点**：
1. **`@PropertySource` 优先级极低**：它加载的文件**覆盖不了** `application.properties`，只能压过默认属性；
2. **profile 不是独立层级**：profile 是 Config data 内部的选择维度（`application-{profile}` 文件比同名主文件优先）；
3. **`SPRING_APPLICATION_JSON` 是隐形第二名**：仅次命令行，K8s 注入配置常用，排查"哪来的值"先查它。

**最反直觉的一步**：Environment 不是"环境"的意思——**它是"配置来源的有序视图"**。名字叫 Environment，干的是队列的事。

### 4.3 代码实证：来源列表、优先级、addFirst 刷新（demo05.EnvironmentSourcesApp）

完整代码：`experiments/code/src/demo05/EnvironmentSourcesApp.java`。

实测输出（`./run.sh demo05.EnvironmentSourcesApp`）：

```text
[默认来源] 数量=2: systemProperties, systemEnvironment
[同key] -D 系统属性抢赢文件配置: from-sys-prop
[addFirst 模拟配置中心推送] 推送后取到新值: from-apollo
[addFirst 再推一次] 最新推送覆盖旧推送: newer-value
[优先级队列顺序] apollo-config-new > apollo-config > systemProperties > systemEnvironment > fileProps
[缺失key] getProperty 返回: null
```

**六行输出 = Environment 的全部机制**：
1. 默认只有两个来源：systemProperties（-D）在 systemEnvironment（OS 环境变量）之前——这就是 00 篇 3.6 场景 E 的机制本体；
2. 文件配置 addLast 追加 → 排在最后 → 同 key 被 -D 抢走（"本地能跑、线上 null"的根源：某台机器有同名 OS 环境变量）；
3. **addFirst = 配置中心刷新的机制本质**：推送新配置 = 往队列头部插一个新 PropertySource，getProperty 立即返回新值，旧来源同 key 瞬间"失明"；
4. 再推一次：后推的在前——**最新推送永远赢**；
5. 缺失 key → null（"配置没进来"的表现，00 篇排查五问的第④问机制本体）。

### 4.4 代码实证：Binder 绑定（demo05.BinderApp）

完整代码：`experiments/code/src/demo05/BinderApp.java`（spring-boot 的 `org.springframework.boot.context.properties.bind.Binder`，就是 `@ConfigurationProperties` 底层的绑定器）。

实测输出（`./run.sh demo05.BinderApp`）：

```text
[绑定成功] callbackUrl=https://pay.example.com/cb, maxRetries=3, enabled=true（宽松绑定 kebab→camel 生效）
[缺失key] 保持默认值: 5
[类型错误] 绑定失败: BindException（启动期显式失败）
```

**三行输出 = Binder 的三个承诺**：
1. **宽松绑定**：配置文件写 `app.order.callback-url`（kebab-case），字段是 `callbackUrl`（camelCase），能绑上——"宽松绑定"是 Specification；
2. **缺失 key 不炸**：保持 POJO 字段默认值（`maxRetries` 默认 5）——绑定失败 ≠ 启动失败；
3. **类型错误启动期炸**：`"abc"` 绑 int → `BindException`——00 篇 3.9 的"启动期失败优于运行期失败"在这里落地。

**和 @Value 的分工**（接 00 篇 3.7）：@Value 单值注入、解析一次；@ConfigurationProperties 整组绑定、可校验（@Validated）、可热刷新。

### 4.5 代码实证：profile 维度（demo05.ProfileApp）

完整代码：`experiments/code/src/demo05/ProfileApp.java`。

实测输出（`./run.sh demo05.ProfileApp`）：

```text
[无 profile] dev 专属 bean 存在? false
[无 profile] 默认 bean 存在? true
[activeProfiles=dev] dev 专属 bean 存在? true
[activeProfiles=dev] @Profile 配置类的 @Bean 存在? true
```

**profile 是 Environment 的第二个维度**：属性来源队列管"值从哪来"，profile 管"哪些 Bean/来源被激活"——`@Profile("dev")` 的 Bean 只在 activeProfiles 包含 dev 时注册（Spring 对 @Profile 是 `ProfileCondition`，本质也是 @Conditional 家族）。

### 4.6 How：全链路——从文件到 Bean 字段

```text
application.properties（或 yml）
  └─▶ Boot config data（ConfigDataEnvironmentPostProcessor）读取
        └─▶ 变成 PropertySource 塞进 Environment（demo06 实证：source 名含 application.properties）
              └─▶ @Value("${app.order.callback-url}")：populateBean 时占位符解析（00 篇 3.5）
                    └─ 或 @ConfigurationProperties：Binder 从 Environment 按前缀绑定（demo05.BinderApp）
                          └─ 配置中心刷新：addFirst 插新 PropertySource（demo05.EnvironmentSourcesApp）
```

**完整因果链**：文件 → config data → PropertySource 队列 → 两个消费口（@Value 占位符解析 / Binder 绑定）→ Bean 字段；刷新 = 队列头部插新来源。

### 4.7 配置中心（Apollo/Nacos）的机制本质

```text
Apollo 启动时（ApolloApplicationContextInitializer，待验证类名）：
  把 Apollo 的配置拉成 PropertySource，addFirst 进 Environment
  → 同 key 立即覆盖本地文件（优先级生效）

热更新的两条路：
  ① @Value 字段：静态解析一次，不会自己刷新（00 篇案例 3 的"静态注入"账）
     → Apollo 用 SpringValueProcessor 扫描 @Value 注入点，配置变化时重新注入（待验证）
  ② @ConfigurationProperties + RefreshScope：重新绑定（Spring Cloud 机制，05 篇/06 篇展开）
```

**为什么 @Value 不刷新**：注入发生在 populateBean 的一次性动作（00 篇 Level 3.5），Environment 变不变跟它没关系；要让 @Value 刷新，必须有额外的"重新注入"机制（SpringValueProcessor）——**裸 @Value 永远不刷新**。

### 4.7.1 动态刷新边界：Environment 变了 ≠ Bean 更新（24 章 17 章交叉补充）

**核心断言**（17 章 Level 5，Boot 4.1 基线文档语义，与 4.7 实测结论一致）：

```text
Environment 改变 ≠ 已创建 Bean 自动更新
```

HTTP Client / 连接池 / 线程池一旦创建就持有旧配置快照。要让新配置生效，必须走**显式重建协议**：

```text
新配置校验 → 创建新资源 → 切换引用 → 排空旧资源 → 失败回滚
```

**哪类配置可以热改、哪类不行**（17 章 Level 5 清单，文档语义）：

| 适合运行时热改 | 不适合热改（影响对象图/结构） |
| ---- | ---- |
| 限流阈值、灰度比例、可热更新规则 | 是否创建 Bean（`feature.enabled=false` → BeanDefinition 不注册） |
| 明确支持刷新生命周期的客户端 | 连接池基本结构、Web ApplicationType、事务管理器类型、安全关键依赖 |

**因果**：BeanDefinition 注册发生在 refresh 早期（00 篇创建链），`enabled` 开关在注册期就决定了 Bean 存在与否——运行中改 Environment 只动了属性队列，已注册的对象图不动（P-E03 的"显示值和运行值不同"）。而 AOT 路线下这类开关在**构建期**就裁定了对象图（07 篇 7.6 展开）。

**同类时机陷阱**：`@PropertySource` 在配置类阶段才加入 PropertySource，**设置 `spring.main.*` 类早期属性太晚**（P-E04，文档语义）——Web 类型探测/懒初始化开关在更早阶段已读取，改不到就是静默不生效；早期属性要走 ConfigData/Environment 处理链。

### 4.8 Transfer

- **优先级堆叠是通用模式**：DNS 解析顺序、路由匹配（精确 > 前缀 > 兜底）、Java 类加载器的委派——全是"有序队列 + 先命中先赢"。
- **统一视图消除分支**：任何一个"多来源可配置"的系统，都应该有一个"有序视图 + 单一查询入口"，而不是到处 System.getenv。
- **刷新 = 插队**：热更新的本质是"新值进队列头部"，而不是"改掉旧值"——这个视角能解释所有配置中心的实现。

**本层留下的账**：配置体系齐了。现在四个扩展点全部就位——**每个框架怎么组合它们**？这逼出下一层：五案例对号入座。

---

<a id="level-5"></a>

## Level 5 五案例对号入座：MyBatis / Dubbo / Redis / MQ / Apollo

### 前置知识关卡

* [ ] 知道四扩展点（Level 3）+ 配置体系（Level 4）
* [ ] 知道三范式：代理/模板/容器（下文展开）
* [ ] 知道每个框架的"领域对象"是什么

### 5.1 Why：五个框架五种玩法，要背五套吗？

徒弟：

> MyBatis 用 Registrar、Dubbo 用 BPP、Kafka 用监听容器……每个都要学一遍吗？

老陈：

> 不用——它们**全部是四扩展点的排列组合**。
> 区别只在：领域对象是什么、用哪个扩展点组合、代理/包装长什么样。
> 我们把主线角色（Mapper 接口）在 Level 3 已经走完，现在用同一张表把五个框架都填进去——**学会填表，胜过背五篇教程**。

### 5.2 总表：五个框架的"四扩展点"对号入座

| 框架 | 领域对象 | 注册（谁把 Bean 弄进容器） | 注入（谁识别注解） | 代理/包装（调它为什么能跑） | 配置入口 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| MyBatis | Mapper 接口 | `@MapperScan` → MapperScannerRegistrar（Registrar + 扫描器，mybatis-spring 实现） | @Mapper / 扫描注册 MapperFactoryBean | MapperFactoryBean.getObject() → SqlSession.getMapper → **MapperProxy（JDK 代理，demo07 同构）** | mybatis.* / @MapperScan 参数 |
| Dubbo | @DubboReference / @DubboService | @EnableDubbo → DubboConfigRegistrar + DubboComponentScanRegistrar（Dubbo 3.x，**待验证**类名） | ReferenceAnnotationBeanPostProcessor（扫描 @DubboReference 注入**远程代理**，demo07 BPP 同构） | Invoker 包装：本地接口 → 注册中心 → 远程调用，代理对象 | dubbo.*（application/registry/protocol） |
| Redis | 模板（RedisTemplate/StringRedisTemplate） | RedisAutoConfiguration 里 @Bean @ConditionalOnMissingBean | 无自定义注解（模板封装，不搞接口代理） | 无代理：**模板 + 连接工厂**（Lettuce/Jedis，Boot 3 默认 Lettuce，**待验证**）封装连接复用 | spring.data.redis.*（Boot 3；2.7 是 spring.redis.*——语义变化） |
| MQ（Kafka） | 监听方法（@KafkaListener） | @EnableKafka → KafkaBootstrapConfiguration | KafkaListenerAnnotationBeanPostProcessor（扫描 @KafkaListener → 注册监听容器） | **监听容器**（ConcurrentKafkaListenerContainerFactory）：后台线程拉消息调你的方法，不是代理是"托管消费者" | spring.kafka.* |
| Apollo | 配置源（PropertySource） | ApolloAutoConfiguration + ApolloApplicationContextInitializer（把 Apollo 配置作为 PropertySource 注入 Environment，**待验证**类名） | ApolloAnnotationProcessor（处理 @ApolloJsonValue 等）；@Value 热更新靠 SpringValueProcessor（**待验证**） | 无代理：**PropertySource 即通道**——配置是 Environment 的一等公民（Level 4.7） | app.id / apollo.bootstrap.* |

### 5.3 主线走完：Mapper 的完整一生（对齐 demo07）

```text
① 加依赖 mybatis-spring-boot-starter（报关）
      ↓
② AutoConfiguration.imports 发现 MybatisAutoConfiguration（验货）
      ↓ 条件：MyBatis 类在 classpath + 数据源 Bean 存在
③ @MapperScan 或 @Mapper → MapperScannerRegistrar 扫描包（登记入册）
      ↓ 每个接口注册一个 MapperFactoryBean（targetType = 接口，demo07 3.3 的类型预测）
④ 业务类 @Autowired UserMapper（分拣）
      ↓ 属性填充阶段：按类型预测找到 MapperFactoryBean → getObject()
⑤ getObject() → SqlSession.getMapper(UserMapper.class)（贴标）
      ↓ 生成 JDK 动态代理：MapperProxy
⑥ 业务代码调用 userMapper.selectById(1)（放行使用）
      ↓ MapperProxy.invoke → 翻译成 SQL 会话操作 → 返回结果（demo07 输出第 3 行）
```

**关键认知**：容器里没有 `UserMapper` 的实现类——只有"能造出 UserMapper 代理"的 MapperFactoryBean。**这就是"接口没有实现类却能注入"的全部真相**（demo07 六行输出已实证）。

### 5.4 对比中的三个高价值差异

**① 代理 vs 模板 vs 容器（三种"可用形态"）**

```text
MyBatis/Dubbo：接口 + 动态代理（调用即拦截，最透明）
Redis：模板 + 连接工厂（不搞接口，封装工具类，你自己调方法）
Kafka：监听容器（框架主动拉消息，调用你的方法——方向反了，你是被调用方）
```

> 📌 **代理、模板、容器是三种整合范式**：看到新框架，先问"它属于哪种范式"，四扩展点立刻好填。

**② @ConditionalOnMissingBean：用户覆盖的唯一正规通道**

```text
RedisAutoConfiguration 的模板 @Bean 带 @ConditionalOnMissingBean(name = "redisTemplate")
→ 你先定义自己的 RedisTemplate Bean → 自动配置跳过 → 覆盖成功
→ 你没定义 → 自动配置兜底创建
（依赖"用户先、自动后"的 DeferredImportSelector 顺序，Level 2.3）
```

**③ 配置前缀是版本迁移重灾区**（接 Level 4 的 Environment：前缀错了 = 绑定不上的 key = 属性静默失效）

```text
Boot 2.7：spring.redis.host=...
Boot 3.x：spring.data.redis.host=...   ← 升级不改配置 = 属性全部失效（语义变化）
```

### 5.5 Transfer

- **三范式判断力（代理/模板/容器）**：看到任何框架，先定范式再填四问——把"背五个框架"压缩成"一套认知"。
- **"接口 + 代理"是 RPC/ORM 界的通用答案**：MyBatis、Dubbo、Feign、Retrofit 全用同一招——接口是契约，代理是翻译，调用方无感。理解了这一个模式，五个框架已经会了三个半。
- **配置入口随版本迁移**：升级核对清单必须包含配置前缀（spring.* → spring.data.*）——这是 2.7 → 3.x 升级最常见的隐性炸弹。

**本层留下的账**：会填表、会拆框架了，但**怎么做一个自己的 starter 给别人用**？以及"没生效"怎么排查？——最后一层：生产实践。

---

<a id="level-6"></a>

## Level 6 生产实践：自定义 starter 与排查方法论

### 前置知识关卡

* [ ] 知道四扩展点（Level 3）+ 配置体系（Level 4）
* [ ] 知道条件评估报告（Level 2.4 的 Positive/Negative matches）
* [ ] 知道 Maven 多模块（autoconfigure 模块 + starter 模块的拆分）

### 6.1 Why：手写配置类 = 每个项目重复抄代码

徒弟：

> 我们自己团队的公共组件（RPC 封装、统一日志、灰度开关）每次接入都手写配置类，为什么不做成 starter？

老陈：

> 手写配置类确实能跑，但留下一个账——**每个项目重复写同样的 @Configuration、重复定义同一组 Bean，参数散落、覆盖通道不统一**。
> 做成 starter 后：依赖即接入（报关一次）、配置收敛（@ConfigurationProperties 一个类）、覆盖走 @ConditionalOnMissingBean（官方通道）。
> 自定义 starter 不是炫技，是**把团队公共决策固化成"依赖"**。

| 方案 | 解决什么 | 留下什么问题 |
| ---- | ---- | ---- |
| 每个项目手写配置类 | 能跑 | 重复代码、参数散落、覆盖方式不统一 |
| 公共模块 @Configuration（被 @ComponentScan 扫到） | 少写一遍 | 扫描路径耦合：组件必须放在扫描包下，否则静默失效 |
| 自定义 starter | 依赖即接入、配置收敛、条件装配 | 要遵守命名/结构规范，开发调试成本 |

### 6.2 What：自定义 starter 的结构（Boot 官方规范）

```text
my-company-log-spring-boot-starter         ← starter 模块（只有依赖，无代码）
  └─ 依赖 my-company-log-spring-boot-autoconfigure

my-company-log-spring-boot-autoconfigure   ← 自动配置模块（真正的代码，demo06 的 autoconfig 包同构）
  ├─ META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  │    └─ com.company.log.LogAutoConfiguration
  ├─ LogAutoConfiguration：
  │     @AutoConfiguration            ← 标注（代替 @Configuration）
  │     @ConditionalOnClass(LogService.class)
  │     @EnableConfigurationProperties(LogProperties.class)
  │     @Bean @ConditionalOnMissingBean
  │     public LogService logService() { ... }
  └─ LogProperties：
        @ConfigurationProperties(prefix = "company.log")
        → Binder 绑定 company.log.* 配置（demo05.BinderApp 已实证绑定机制）
```

**命名规范**：`xxx-spring-boot-starter` 放依赖，`xxx-spring-boot-autoconfigure` 放实现——官方约定（Boot 文档 "Creating Your Own Auto-configuration"）。

**条件设计三条铁律**（写 starter 的正确姿势）：

```text
① @ConditionalOnClass：只在依赖存在时启用（别用 @ConditionalOnMissingClass 兜异常）
② @ConditionalOnMissingBean：给用户覆盖通道（你的 Bean 只是默认值）
③ @ConditionalOnProperty：能配开关就配开关（默认开/关要有明确默认值）
```

### 6.3 How：排查方法论（"为什么没生效"三问）

```text
接到"自动配置没生效"——
① 打开 /actuator/conditions（没有 actuator 就用 --debug 启动日志）
   → 找到对应 AutoConfiguration 的条目
② 看 Negative matches 的"原因"列，对照三问（demo06 的 --demo.enabled=false 就是实测版）：
   a. @ConditionalOnClass 失败 → 依赖没进来？版本冲突？排除错了？
   b. @ConditionalOnProperty 失败 → 开关属性没配？（demo06 实测：命令行参数覆盖 → Negative）
   c. @ConditionalOnMissingBean 失败 → 你自己的 Bean 已经存在（这是覆盖，不是故障）
③ 如果 Positive 但行为不对 → 看配置绑定：
   前缀对不对（Boot 3 的 spring.data.* 迁移账）？Bean 覆盖了没？
```

```text
问题 ──▶ conditions 端点
          ├─ Negative：读原因列 → 类/属性/Bean 三问
          ├─ Positive 但行为不对：查配置前缀与覆盖（Level 4 的 Environment 视角）
          └─ 全绿但注入失败：查 FactoryBean 类型预测 / BPP 注册时序（Level 3.3）
```

**最反直觉的一步**：**"没生效"常常是"生效了但被覆盖了"**——Negative 的原因写着 @ConditionalOnMissingBean 时，系统正在按你的要求工作，不是故障。

### 6.4 Transfer

- **把团队决策固化成依赖**：starter 是"组织级 DRY"——公共决策（日志格式、调用链、安全过滤）做成 starter，接入成本从"抄代码"降到"加依赖"。
- **条件三铁律是任何默认值系统的设计规范**：给默认值、留覆盖通道、配开关——K8s 的 Helm chart values、SDK 的默认参数全是同一套。
- **"为什么"必须可查询**：自动化决策系统不留下决策记录（Positive/Negative matches），排查就靠猜——"自动化 + 可观测"不可分割。

**本层收账**：到这里，框架整合的完整因果链闭合——**加依赖（报关）→ imports 白名单（候选）→ @Conditional 验货 → 四扩展点翻译（登记/注入/贴标/参数）→ 可用 Bean**；配置体系的链同时闭合——**文件/config data → PropertySource 队列 → @Value/Binder 消费 → 配置中心插队刷新**。

---

<a id="summary-graph"></a>

## 全篇因果链总图

```text
加 starter 依赖（报关）
   │ ① classpath 有了自动配置类 + imports 白名单
   ▼
AutoConfiguration.imports 候选清单（Level 2）
   │ ② DeferredImportSelector（用户先、自动后）+ AutoConfigurationSorter 排序
   ▼
@Conditional 验货（Level 2）
   ├─ OnClass：依赖在吗？  ├─ OnProperty：开关配了吗？  ├─ OnMissingBean：缺吗（覆盖通道）？
   │        通过 → 注册为配置类（否则 Negative matches 可查询）
   ▼
四扩展点翻译（Level 3 + 4）
   ├─ Registrar 批量注册（MapperFactoryBean × N）
   ├─ BPP 注解识别注入（@DubboRef / @KafkaListener）
   ├─ FactoryBean 代理翻译（MapperProxy 同构；类型预测 targetType）
   └─ Environment/配置体系：参数从哪来（Level 4）
          PropertySource 队列（优先级堆叠）→ @Value/Binder 消费 → 配置中心 addFirst 刷新
   ▼
可用 Bean → 业务代码直接用（@Autowired / @Transactional / @ConfigurationProperties）

排查回路：没生效 → conditions 端点 → Negative 原因列 → 三问定位（Level 6）
```

**六字口诀**：

```text
报关验货登记贴标，参数排队优先赢，
条件否决可查询，类型预测别忘设。
```

---

<a id="prod-cases"></a>

## 🏥 线上案例：三个真实复盘（全部命中本篇四层）

> 复盘格式：现象 → 根因 → 机制链 → 修复 → 预防。数字只做量级描述，不做精确承诺（待实测）。

### 案例 1：配置中心热更新不生效——@Value 的"静态注入"账

- **现象**：Apollo 改了配置，@Value 字段不刷新；重启才生效（00 篇案例 3 的机制本体）。
- **根因**：@Value 在 populateBean 时解析一次（00 篇 3.5 实测：PSC 解析发生在创建时刻），之后 Environment 里有什么与字段无关。
- **机制链**：Level 4.7——配置中心刷新 = 往 Environment 队列插新 PropertySource（demo05 的 addFirst 实测），但 @Value 注入点没有"重新注入"机制。
- **修复**：需要热更新的配置走 @ConfigurationProperties + 配置中心刷新机制（Apollo 的 @ApolloConfigChangeListener / SpringValueProcessor / Nacos @RefreshScope）；@Value 只用于一次性静态值。
- **预防**：代码评审规则"@Value 不用于可变更配置"；配置中心发布前核对"这个 key 是谁在消费、怎么消费"。

### 案例 2：升级 Boot 3 后配置全部失效——前缀迁移账

- **现象**：2.7 升 3.x 后 Redis 连不上本地、连接池参数异常，日志无报错。
- **根因**：`spring.redis.*` → `spring.data.redis.*`（语义变化）。旧前缀的 key 在 Environment 队列里仍然存在，但 Binder 按新前缀绑定，旧 key **绑不上 = 静默失效**，回退到自动配置默认值。
- **机制链**：Level 4.4——Binder 按前缀匹配；前缀错 = 缺失 key = 保持默认值（demo05.BinderApp 第 2 行实测：缺失 key 不炸，静默用默认）。
- **修复**：升级核对清单包含全部配置前缀；绑定后用 `/actuator/env` 或启动日志核对解析出的值。
- **预防**：升级演练（2.7 → 3.x）固定步骤：前缀核对表 + 配置解析结果 diff。

### 案例 3：自动配置"没生效"——Negative matches 排障实录

- **现象**：加了某中间件 starter，Bean 注入失败；群里第一反应"starter 坏了"。
- **根因**：打开 conditions 端点，该自动配置类是 Negative，原因列写着"没有匹配的 @ConditionalOnProperty 属性"——开关没配，条件设计本意（demo06 实测：--demo.enabled=false 时 demoFlag 不存在）。
- **机制链**：Level 2.4——条件三问（类/属性/Bean）；Level 6.3——排查先读决策记录。
- **修复**：按原因列补配置；若原因列是 @ConditionalOnMissingBean，则是"覆盖生效"——检查自己的 Bean 是否预期。
- **预防**：任何 starter 接入先看 Positive/Negative；团队 wiki 固化"没生效三问"。

---

<a id="interview"></a>

## 🎤 面试八股自查表（本篇范围的知其所以然）

| 八股问题 | 一句话因果答案 | 证据在哪 |
| ---- | ---- | ---- |
| 框架为什么不能直接 new？ | 生命周期、配置、协作三笔账 | Level 1.1 |
| SPI 为什么不够？ | 只解决"发现"，不解决"管理"（注入/生命周期/配置/代理） | Level 1.2 实测：SPI 找到 2 个实现，无人管理 |
| 加依赖为什么就能用？ | AutoConfiguration.imports 白名单 + @Conditional 验货 + DeferredImportSelector（用户先、自动后） | Level 2.2 实测：扫描包之外的自动配置类被加载 |
| 自动配置"没生效"先查什么？ | actuator conditions 端点（Positive/Negative matches 的原因列） | Level 6.3 + demo06 条件否决实测 |
| Mapper 接口没有实现类，怎么注入？ | FactoryBean.getObject() 返回 JDK 动态代理（MapperProxy 同构） | Level 3.2 实测：@Autowired 拿到 $Proxy |
| @Autowired 按类型怎么找到 FactoryBean？ | 类型预测（targetType > 工厂方法返回类型 > 泛型签名）；预测落空就 NoSuchBeanDefinition | Level 3.3 实测（setTargetType 修复） |
| 三扩展点各解决什么账？ | Registrar 批量注册 / BPP 注解识别 / FactoryBean 接口代理 | Level 3.2 六行实测 |
| Environment 是什么？ | 有序 PropertySource 列表 + profile 维度；getProperty 按序查 | Level 4.3 实测：优先级队列打印 |
| 同名 key 多个来源，谁赢？ | 队列顺序，先命中先返回；System properties > OS env > 文件 | Level 4.3 实测：from-sys-prop 抢赢 |
| 配置中心刷新做了什么？ | 往队列头部 addFirst 插新 PropertySource，旧值瞬间"失明" | Level 4.3 实测：from-apollo / newer-value |
| @ConfigurationProperties 怎么绑定？ | Binder 按前缀 + 宽松绑定（kebab→camel）；缺失保持默认；类型错启动期炸 | Level 4.4 三行实测 |
| 为什么 @Value 不刷新？ | 注入是 populateBean 的一次性动作；刷新需要额外"重新注入"机制 | Level 4.7 + 00 篇 3.5 |
| profile 是什么？ | Environment 第二个维度：@Profile 的 Bean 只在 activeProfiles 匹配时注册 | Level 4.5 实测 |
| 自定义 starter 的条件三铁律？ | OnClass 依赖存在 / OnMissingBean 留覆盖 / OnProperty 给开关 | Level 6.2 |
| Boot 3 配置迁移雷区？ | spring.redis.* → spring.data.redis.*（语义变化，前缀错=静默失效） | Level 5.4 + 案例 2 |

---

<a id="pitfalls"></a>

## ⚠️ 坑与细节（10 个真实误解）

### 坑 1：框架整合 = 写胶水代码把两个库粘起来

真相：真正的整合是"翻译"——把框架领域对象翻译成容器 Bean（生命周期/注入/配置/代理全接管）。线上现象：整合代码到处 new 框架对象，生命周期没人管，无法纳入事务/AOP。

### 坑 2：JDK SPI 就是 Spring 整合的机制

真相：ServiceLoader 只解决"发现实现类"，发现之后无人管理。线上现象：用 SPI 加载框架对象后，对象和 Spring 容器彻底脱节（Level 1.2 实测）。

### 坑 3：自动配置"没生效" = 框架坏了

真相：条件设计本意——类不在/属性没配/Bean 已有，都会被否决。线上现象：加依赖没反应，一顿乱猜。修正：读 Negative matches 的原因列（demo06 条件否决实测）。

### 坑 4：自动配置处理完了才轮到用户配置

真相：DeferredImportSelector 延迟——用户配置先处理，自动配置后补缺（@ConditionalOnMissingBean 依赖此顺序）。线上现象：自定义 Bean 与自动配置 Bean 重复创建。

### 坑 5：@MapperScan 和 @ComponentScan 是一回事

真相：MapperScan 是 Registrar 批量注册 MapperFactoryBean（每接口一个工厂）；ComponentScan 扫 @Component 族直接注册类。线上现象：Mapper 没注册（没加 MapperScan），或接口被当普通 Bean 注册失败。

### 坑 6：Mapper 接口的"实现类"是 MyBatis 生成的

真相：没有实现类——容器里是 MapperFactoryBean，getBean 时返回 JDK 动态代理（MapperProxy）。线上现象：反编译找实现类找不到，怀疑人生（demo07 实测 $Proxy14）。

### 坑 7：按类型注入找不到自定义 FactoryBean

真相：未创建的 FactoryBean 靠"类型预测"（targetType/工厂方法返回类型/泛型签名）参与 byType 匹配，预测落空 = NoSuchBeanDefinitionException——**必须 setTargetType 或走 @Bean 方法**（Level 3.3 实测修复）。

### 坑 8：框架参数随便写哪都行，前缀无所谓

真相：Binder 按前缀严格匹配（宽松绑定只限大小写/连字符，前缀不能错）；Boot 3 有 spring.* → spring.data.* 迁移。线上现象：升级 Boot 3 后配置全部失效，属性静默为默认值（案例 2）。

### 坑 9：@ConditionalOnMissingBean 失败是故障

真相：这是"覆盖生效"——你自己定义的 Bean 优先，自动配置让位。线上现象：查了半天以为坏了，其实系统按你的要求工作。

### 坑 10：Apollo/配置中心的值 @Value 就能热更新

真相：@Value 是静态占位符解析（00 篇 3.5 实测）；热更新需要配置中心处理器（SpringValueProcessor 等）+ 重新注入机制；@ConfigurationProperties + RefreshScope 才是正规军。

---

<a id="errata"></a>

## 📚 版本勘误表

> 勘误格式：我曾讲错的 → 真相。区分 Specification / Implementation / 语义变化。

| 我曾讲错的 | 真相 | 性质 |
| ---- | ---- | ---- |
| "@MapperScan 与 @ComponentScan 等价" | 前者是 Registrar 批量注册 MapperFactoryBean，后者扫 @Component 族 | Implementation |
| "Mapper 接口有 MyBatis 生成的实现类" | 没有实现类，是 JDK 动态代理（MapperProxy 拦截翻译） | Implementation |
| "自动装配是'全自动'" | "有条件的自动"：@Conditional 否决是常态（Negative matches 可查询） | Implementation |
| "用户配置在自动配置之后处理" | DeferredImportSelector 延迟——用户先、自动后（覆盖通道依赖此顺序） | Implementation |
| "spring.factories 仍支持自动配置" | Boot 3 已移除，只认 AutoConfiguration.imports（语义变化） | 语义变化 |
| "spring.redis.* 在 Boot 3 还能用" | 迁移为 spring.data.redis.*，旧前缀静默失效 | 语义变化 |
| "FactoryBean 注册了就能按类型注入" | 未创建时靠类型预测；预测落空（没设 targetType）按类型找不到 | 实测（6.1.14） |
| "@Value 天然支持配置中心热更新" | 静态占位符解析；热更新需配置中心处理器 + 重新注入机制 | Specification |
| "SPI 就是 Spring 的整合机制" | SPI 只发现不管理；Spring 扩展点才管理生命周期/注入/配置/代理 | 实测（demo01） |

坦白记录（我之前也讲错过）：

- 我讲过"FactoryBean 只要注册了，@Autowired 就能按类型注入"——准确说法是**未创建的 FactoryBean 靠类型预测**（targetType > 工厂方法返回类型 > 泛型签名），demo07 实测：不设 targetType 时 NoSuchBeanDefinitionException。
- 我讲过"Environment 是环境配置"——准确说法是**有序 PropertySource 队列的视图**，名字骗人，机制是队列。

---

<a id="decisions"></a>

## 🏆 生产决策卡

### Decision Card 1：新框架接入评审

- **场景**：团队要引入一个第三方框架，先做架构评估。
- **判断**：按四扩展点填表：注册/注入/代理/配置分别是什么、谁来做；填不出任何一格 → 该框架整合不完整，接入成本高。
- **Mechanism → Decision**：

```text
评审清单：
□ 注册：谁把领域对象变 BeanDefinition？（Registrar/自动配置）
□ 注入：自定义注解谁扫描？（BPP）
□ 代理：调用如何被翻译？（代理/模板/容器三范式）
□ 配置：参数绑定到哪？（@ConfigurationProperties 前缀）
```

- **禁止决策**：禁止"四问填不满就引入"；禁止引入无 Boot 3 配套 starter 的框架后手动补胶水。
- **验收指标**：四问表格完整度；接入后 conditions 端点 Positive matches 稳定；启动耗时增量（待实测）。

### Decision Card 2：自定义团队 starter

- **场景**：公共日志/调用链/灰度组件，每个项目都要接。
- **判断**：做成 starter（autoconfigure + starter 双模块），条件三铁律，配置收敛 @ConfigurationProperties。
- **Mechanism → Decision**：

```java
@AutoConfiguration
@ConditionalOnClass(LogService.class)
@EnableConfigurationProperties(LogProperties.class)
public class LogAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public LogService logService() {
        return new DefaultLogService();
    }
}
```

- **禁止决策**：禁止 starter 里写死配置值；禁止不给 @ConditionalOnMissingBean（用户无法覆盖）；禁止 starter 依赖业务代码（循环依赖）。
- **验收指标**：接入项目数 × 接入耗时（从抄代码变为加依赖）；conditions 端点 Positive matches 稳定；用户覆盖率/覆盖成功次数。

### Decision Card 3：自动配置"没生效"的排障流程

- **场景**：线上/测试环境自动配置没生效。
- **判断**：先查决策记录（conditions），再查配置绑定，最后才查源码——顺序反了 = 排障时间 ×10。
- **Mechanism → Decision**：

```text
# 开启 actuator
management.endpoints.web.exposure.include=health,conditions,metrics
# 查看
curl /actuator/conditions | jq '.contexts[].beans | keys[]'
# 或启动时 --debug 打印条件评估
java -jar app.jar --debug
```

- **禁止决策**：禁止跳过 conditions 直接翻源码；禁止"重启试试"式排障（不留下决策记录）。
- **验收指标**：排障平均时长（引入"三现场"方法论后对比）；Negative matches 被误判为故障的次数 → 0；条件评估报告归档率。

---

<a id="cross-language"></a>

## 🌍 跨语言视角

| 语言/生态 | 做法 | 对应 Spring 的什么 | 账 |
| ---- | ---- | ---- | ---- |
| Java（Spring） | 运行时反射：Registrar/BPP/FactoryBean 动态注册与代理 | 四扩展点本体 | 魔法多，靠可观测补偿 |
| Go | 无反射容器：google/wire 编译期生成注入代码；框架对象显式构造传参 | 编译期 DI（替代反射注入） | 没有"自动装配"：整合全靠手写/代码生成；胜在启动快、无魔法 |
| Rust | 无容器：axum 生态显式组合（状态注入 handler）、tower 中间件链 | 装饰器/中间件模式 | 无反射：代理即中间件，显式而冗长 |
| Kubernetes | CRD + Controller：声明 spec → controller 调谐到期望状态 | Registrar + 生命周期钩子的"声明式"版 | 以状态机替换回调，更强一致、更重 |

**底层思想是否一致？**

一致的部分：

- **注册**：任何体系都要回答"新东西怎么进来"（Go 是编译期显式，K8s 是 API 注册 CRD）。
- **翻译/包装**：接口 → 实现（或 spec → 实际状态）永远需要一层翻译：Java 用动态代理，Go 用接口 + 显式实现，K8s 用 controller。
- **配置参数化**：配置与代码分离是普适规律；"多来源 + 优先级 + 统一视图"（Environment 的队列思想）在 Go（env/flag）、Rust（config crate）、K8s（configmap）都存在。

不一致的部分：

- **自动化的代价**：Spring 用"运行时魔法"换开发效率，代价是可观测性必须跟上；Go/Rust 用"显式代码"换确定性，代价是整合代码多、改造成本高。
- **判断力**：选型问的不是"谁更强"，而是"你的团队能承受哪种账"——**"注册/翻译/配置"三件套在任何语言都成立，这就是跨语言仍成立的判断力。**
