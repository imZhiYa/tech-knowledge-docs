# 🚆 SpringBoot 骨干：Spring 如何创建对象（系列入口 00）

> 本篇文章只回答一个问题：**Spring 把一个"类"变成"可用对象"，全程做了什么？为什么必须这么做？**
> 事件机制、自动装配、运行时刻不在本篇，它们建立在"创建"之上——先把地基挖到底。

---

# ⚠️ 版本与证据边界

| 维度 | 本文承诺 |
| ---- | ---- |
| 代码实证 | 全部来自 `knowledge/springboot/experiments/code/` 下的 15 个可运行 demo，本机实测输出原样引用 |
| 实测环境 | macOS + JDK 21.0.11（Azul Zulu）+ spring-context **6.1.14**（对应 Boot 3.3.x / Framework 6.1） |
| Specification | 生命周期钩子顺序、注入解析链的语义（@Primary/@Qualifier）、父子容器委派语义 |
| Implementation | 三级缓存、Full 模式 CGLIB 子类化、Lite 模式行为、Aware 处理器内部顺序、static 注入点忽略（版本间可变，标注 6.1.14）；循环依赖默认值：裸容器 allowCircularReferences=true、Boot 3.3.5 默认 false（demo04.DisableCircularApp + demo15.BootCircularApp 实测对照） |
| 未覆盖 | 不承诺任何性能数字；不涉及自动装配细节（01 篇/03 篇）；Environment 机制只留结论（完整展开在 01 篇配置体系） |

---

# 🏷️ 关键词

容器 / BeanDefinition / 注册表 / 单例池 / 生命周期 / BeanPostProcessor / populateBean / 依赖注入解析链 / @Value 占位符解析 / 三级缓存 / 循环依赖

---

# 🗂️ 目录

- Level 1 容器诞生：为什么 `new` 会死
- Level 2 一个 Bean 的一生：生命周期钩子板
- Level 3 属性填充：依赖注入的解析链
- Level 4 循环依赖：三级缓存与"被禁止的好特性"
- 全篇因果链总图
- 线上案例（4 个）
- 面试自查表
- 坑与细节（10 个）
- 版本勘误表
- 生产决策卡（3 张）

---

# 为什么需要这篇

徒弟：

> 我用 `new UserService()` 十年了，SpringBoot 注解一把梭也写了三年。为什么还要研究"创建对象"？

老陈：

> 因为你线上踩的坑，九成不是注解没背熟，而是**不知道对象是怎么到你手里的**：
> - 升级 Boot 2.6+ 后启动直接炸——循环依赖的环突然解不开；
> - `@Bean` 方法在普通类里被调用两次，出现了两个实例——线上资源重复创建；
> - 配置中心改了值，`@Value` 就是不刷新——你根本不知道值是谁解析的；
> - 加了 `@Async` 后原来能跑的项目启动失败——代理和循环依赖的碰撞。
>
> 这四个事故的根因，全部在"创建对象"这一层。这一篇把因果链打通，后面的事件、自动装配、Web 容器都只是在这条链上开的口子。

---

# 🏭 全文唯一比喻：地铁线路图

- **线路图** = 容器（注册表 + 单例池）：谁和谁相连、在哪换乘，一张图讲清。
- **换乘站** = 依赖注入（populateBean + resolveDependency）：两个 Bean 之间靠容器搭桥。
- **安检/关卡** = 生命周期钩子（BeanPostProcessor）：每个 Bean 过站时被检查、被改装（AOP 代理就是在这里贴的"防伪标签"）。

```text
现实世界（地铁）                          技术世界（Spring）
乘客想从 A 到 D                     业务代码想用一个 Bean
   │ 不用自己走完全程                   │ 不自己 new，交给容器
   ▼                                   ▼
线路图：A→换乘站→B→C→D          ←→   容器：注册表 + 单例池
   │ 换乘站 = 装配点                   │ populateBean + resolveDependency
   ▼                                   ▼
安检关卡 = 每次过站的检查          ←→   BeanPostProcessor（AOP 代理在这里）
   ▼                                   ▼
到达 D                          ←→   Bean 可注入、应用就绪
```

**全文不再出现第二个比喻。**

---

# 正文：四个 Level 打穿"创建对象"因果链

<a id="level-1"></a>

## Level 1 容器诞生：为什么 `new` 会死

### 前置知识关卡

* [ ] 知道"耦合"指什么（一个类依赖另一个类的具体实现）
* [ ] 知道反射能动态创建对象、调用方法
* [ ] 知道单例模式（进程内一个实例）

### 1.1 Why：`new` 和工厂各欠了什么账

徒弟：

> 我用 `new UserService()` 不就行了？为什么要搞个"容器"出来？

老陈：

> `new` 确实解决了"创建对象"，但是留下了三笔账——
> 第一，**耦合账**：A 里 `new B()`，B 的构造改了，A 跟着改；B 要换成实现 C，A 也要改；
> 第二，**生命周期账**：谁负责初始化、谁负责销毁？散落在每个调用方，没人管；
> 第三，**测试账**：A 依赖的是 B 的"具体类"，想 mock 掉 B，A 的代码写死了，没法替换。

| 方案 | 解决什么 | 留下什么问题 |
| ---- | ---- | ---- |
| 直接 new | 创建对象 | 耦合、生命周期失控、测试困难 |
| 工厂模式（静态工厂） | 把创建逻辑集中 | 工厂自身成为新依赖：工厂套工厂，处处是工厂 |
| IoC 容器 | 统一接管"创建+装配+生命周期" | 引入"魔法"：谁创建的、什么时候销毁，对调用方不可见（用可观测性补偿） |

**为什么工厂会死**：`UserServiceFactory` 解决"别 new 了"，但 `UserServiceFactory` 里还是要 `new OrderService()`；当依赖链变长，你得到的是"工厂依赖工厂依赖工厂"。**问题不是"谁 new"，而是"创建这个动作散落在哪"——需要一个统一的地方存"怎么创建"的信息。**

### 1.2 代码实证：手写 45 行迷你容器

先别看 Spring，我们用手写一个"注册表 + 反射 + 单例池"三件套，亲眼确认这三样东西够不够用。
完整代码：`experiments/code/src/demo01/MyMiniIoC.java`，核心逻辑：

```java
public class MyMiniIoC {
    private final Map<String, Class<?>> registry = new HashMap<>();   // 注册表（BeanDefinition 元数据）
    private final Map<String, Object> singletonPool = new HashMap<>(); // 单例池

    public Object getBean(String name) throws Exception {
        Object cached = singletonPool.get(name);
        if (cached != null) return cached;                            // 单例复用
        Class<?> clazz = registry.get(name);
        Object instance = createInstance(clazz);                       // 反射创建
        singletonPool.put(name, instance);
        return instance;
    }
    // createInstance：找构造器，按参数类型查注册表找依赖 → newInstance(dep)
}
```

实测输出（`./run.sh demo01.MyMiniIoCApp`）：

```text
两次 getBean 是否同一实例（单例池生效）: true
构造器依赖注入成功: true
```

**这个 45 行的容器已经具备了 Spring 的核心骨架**：注册表存"怎么创建"、getBean 反射创建、单例池复用。下面看看 Spring 原生的 `DefaultListableBeanFactory` 是不是同样的三件套。

### 1.3 代码实证：DefaultListableBeanFactory 原始用法

绕过注解、绕过 Boot，直接操作 Spring 容器的最底层。
完整代码：`experiments/code/src/demo01/DefaultListableBeanFactoryApp.java`：

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();

// ① 注册"说明书"（BeanDefinition 元数据），而不是对象——此刻容器没有任何实例
RootBeanDefinition orderDef = new RootBeanDefinition(OrderService.class);
factory.registerBeanDefinition("orderService", orderDef);

RootBeanDefinition userDef = new RootBeanDefinition(UserService.class);
MutablePropertyValues pv = new MutablePropertyValues();
pv.add("orderService", new RuntimeBeanReference("orderService")); // 属性引用另一个 Bean 名
userDef.setPropertyValues(pv);
factory.registerBeanDefinition("userService", userDef);

System.out.println("[注册] 只注册了 BeanDefinition 说明书，单例池当前数量: " + factory.getSingletonCount());

UserService s1 = factory.getBean("userService", UserService.class);
UserService s2 = factory.getBean("userService", UserService.class);
System.out.println("singleton scope 两次 getBean 同一实例: " + (s1 == s2));
```

实测输出（`./run.sh demo01.DefaultListableBeanFactoryApp`）：

```text
[注册] 只注册了 BeanDefinition 说明书，单例池当前数量: 0
singleton scope 两次 getBean 同一实例: true
依赖被注入: true
单例池当前数量: 2
prototype scope 两次 getBean 不同实例: true
```

**三组数字就是容器的全部真相**：
- 注册阶段单例池是 0——容器里只有"说明书"（BeanDefinition），没有对象；
- singleton 两次 getBean 同一实例——单例池复用（此时池里 2 个：userService + 因属性引用被顺带创建的 orderService）；
- prototype 两次 getBean 不同实例——scope 就是"创建策略"参数。

### 1.4 What：容器的核心数据结构

```text
IoC 容器 = 注册表（BeanDefinition 元数据）
         + 反射（按元数据创建实例）
         + 单例池（默认 scope=singleton，复用实例）

    Map<String, BeanDefinition>  →  创建 →  Map<String, Object>（单例池）
        "userService"                       │
        类名/构造参数/依赖列表               ▼
        作用域/初始化方法/销毁方法      userService 实例（复用）
```

- **核心对象**：`BeanDefinition`（怎么创建这个 Bean 的元数据）、`BeanFactory`（容器接口）、`DefaultListableBeanFactory`（完整实现，Implementation）
- **关键不变量**：Bean 的使用方**只依赖类型/名称，不依赖创建过程**；单例池中同一名称至多一个实例

### 1.5 How：源码路径（Framework 6.1.14 Implementation）

平时用 `new AnnotationConfigApplicationContext(App.class)` 时，容器内部执行的主路径：

```text
AnnotationConfigApplicationContext 构造
  └─▶ refresh()                                   AbstractApplicationContext
        ├─▶ invokeBeanFactoryPostProcessors()     ← 解析配置类、处理 @ComponentScan/@Import/@Bean
        │        （ConfigurationClassPostProcessor：把注解翻译成 BeanDefinition 注册表）
        └─▶ finishBeanFactoryInitialization()     ← 创建所有非懒加载单例
              └─▶ preInstantiateSingletons()      DefaultListableBeanFactory
                    └─▶ getBean(name)
                          └─▶ doGetBean()
                                └─▶ createBean()
                                      └─▶ doCreateBean()   ← 一个 Bean 的完整生产线（Level 2 主角）
```

**最反直觉的一步**：容器里存的不是对象，是"怎么造对象的说明书"。

徒弟：

> 容器不是把对象都 new 好放 Map 里吗？那 BeanDefinition 是干嘛的？

老陈：

> 直接放对象，就是"更聪明的工厂"；但工厂不知道"这个对象有哪些依赖、要不要 AOP、初始化方法是什么"。
> BeanDefinition 是**元数据**——延迟到容器需要时，按说明书反射创建、填充、包装。
> 这一层抽象才是后续所有机制（AOP、自动装配、条件装配）能插进来的地基。
> 你在 1.3 的 demo 里看到了铁证：注册完成后单例池数量是 0。

### 1.6 深挖：getBean 决策树（四分支实测）

`getBean` 是容器的**唯一出口**，`doGetBean` 内部是一棵决策树。完整代码：`experiments/code/src/demo01/GetBeanTreeApp.java`。

实测输出（`./run.sh demo01.GetBeanTreeApp`）：

```text
[①] 创建 SingletonService（只应该出现一次）
[①] singleton 命中缓存，不再走创建（上面的打印只出现一次）: true
[②] getBean("svcFactory") 拿到产品: demo01.GetBeanTreeApp$Service
[②] getBean("&svcFactory") 拿到工厂本身: demo01.GetBeanTreeApp$SvcFactory
[③] getBean(Class) 单候选解析成功: true
[④] product 实际类名: demo01.GetBeanTreeApp$LazyProduct$$SpringCGLIB$$0（Spring 6 repackaged cglib 命名 $$SpringCGLIB$$）
[④] @Lazy 注入的是代理: true
[④] 代理注入时真实实例未创建: true
[④] 代理首次调用后才触发真实创建: true
[③] getBean(Class) 多候选: NoUniqueBeanDefinitionException（候选: getBeanTreeApp.DualServiceA,getBeanTreeApp.DualServiceB）
```

**四组输出对应决策树的四个分支**（`doGetBean`，Framework 6.1.14 Implementation）：

```text
getBean(name) / getBean(Class)
  │
  ├─▶ ① 单例：先查一级缓存（getSingleton(name) 命中即返回，不重走创建）
  │
  ├─▶ ② FactoryBean 分支：bean 名以 "&" 开头 → 返回 FactoryBean 本身；
  │        否则返回 FactoryBean.getObject() 的产品（产品按 isSingleton 决定是否缓存）
  │
  ├─▶ ③ byType 分支（getBean(Class)）：resolveBean 按类型匹配
  │        单候选 → 返回；多候选 → NoUniqueBeanDefinitionException（消息里列出全部候选）
  │
  └─▶ ④ @Lazy 分支：注入点带 @Lazy → 不解析真实实例，注入 CGLIB 代理
           （目标 bean 自身 @Lazy 时连 preInstantiateSingletons 都不创建它；
             代理首次方法调用才触发 getBean 真实创建）
```

**四个分支各自回答一个线上问题**：
- ① 为什么"拿 Bean"便宜：命中缓存，不重走生产线；
- ② 为什么第三方框架能"藏"创建逻辑：FactoryBean 是容器给框架留的"自定义创建通道"（MyBatis 的 MapperFactoryBean、Redis 的连接工厂全是它）；
- ③ 为什么 getBean(Class) 会炸：类型歧义在 byType 解析时显式失败，消息把候选列全；
- ④ 为什么 @Lazy 能破循环依赖：依赖点拿的是代理，真实创建被推迟（Level 4 的缓兵之计，注意它治标不治本）。

### 1.7 深挖：父子容器（五事实实测）

容器不是只能有一个——**子容器可以委派父容器**。这是 SSM 时代 Spring MVC 双容器（Root WebApplicationContext 为父、DispatcherServlet 的 WebApplicationContext 为子）的机制底座；Boot 时代默认单容器（SpringApplication 只建一个 context），但理解父子容器才能看懂遗留项目事故。完整代码：`experiments/code/src/demo01/ParentChildContextApp.java`。

实测输出（`./run.sh demo01.ParentChildContextApp`）：

```text
[①] 子容器 getBean 父容器独有 bean: 成功，实例来自父容器: true
[②] 父容器 getBean 子容器独有 bean: NoSuchBeanDefinitionException
[③] 同名 bean：子容器优先 -> 子容器实例
[④] 子容器 @Autowired 注入父容器 bean: true
[④] 子容器 @Autowired 同名 bean 注入的是自己的: 子容器实例
[⑤] 同一类父子各注册一次，两个不同实例: true
```

**五个事实就是父子容器的全部语义**：

```text
① 子容器 getBean 找不到 → 向上委派父容器（getBean 链式查找，AbstractBeanFactory 的 parentBeanFactory）
② 查找是单向的：父容器看不到子容器的 bean（NoSuchBeanDefinitionException）
③ 同名 bean：子容器优先（子覆盖父的查找结果）
④ @Autowired 依赖解析同样向上：子容器可以注入父容器的 bean
⑤ 同一个类在父子容器各注册一次 → 两个独立实例
```

**线上事故的根因（对应 ⑤）**：SSM 双容器时代，Spring 的 XML 和 MVC 的 XML 各扫一次包 → Service 在父容器一份、子容器一份 → 事务代理只加在父容器那份上，子容器注入到的是"裸的"Service → **@Transactional 静默失效**。排查口诀：先问"这个 Bean 在几个容器里"（`parentBeanFactory` 链走一遍，看 getBean 命中哪个）。

> 📌 父子容器语义是 Specification（委派查找、子优先、单向可见）；但"双容器扫描导致重复实例"是工程配置错误，不是容器设计缺陷——Boot 单容器默认值正是为了消灭这类事故。

### 1.8 坑：本层最容易翻车的三处

| 坑 | 真相 | 翻车现场 |
| ---- | ---- | ---- |
| 单例复用 | singleton 是默认且唯一的"缓存策略" | 把有状态对象当单例，线程之间互相污染 |
| prototype 不回收 | prototype 创建后容器不管销毁 | 高频 getBean(prototype) 泄漏资源（容器不跟踪销毁） |
| bean 名是字符串 | 注册表以名字为 key | 同名注册会覆盖（`registerBeanDefinition` 同名校验默认放行） |

### 1.9 Transfer

- **元数据驱动创建**（把"怎么做"从代码变成数据）：插件系统、规则引擎、K8s 的 spec 都是同一思想——行为参数化。
- **单例池 = 共享状态缓存**：任何"注册表 + 缓存 + 全局同步"的组合都要回答"创建时别的线程看到什么"。
- **依赖倒置**：调用方面向接口 + 容器提供实现，这是所有可测试架构的地基。
- **查找链 = 职责边界**：子容器委派父容器，和类加载器的双亲委派是同一种"先看自己、再看上级"的结构——隔离与复用的平衡点。

**本层留下的账**：容器只会"创建"，但一个对象创建完通常还不能用——**属性没填、初始化没做、该有的代理没有**。谁来完成这些？这逼出下一层：生命周期。

---
<a id="level-2"></a>

## Level 2 一个 Bean 的一生：生命周期钩子板

### 前置知识关卡

* [ ] 知道 Level 1 的 BeanDefinition 注册表
* [ ] 知道反射可以给私有字段赋值（setAccessible）
* [ ] 知道 AOP 代理 = 包装原对象

### 2.1 Why：裸 new 出来的对象是"空的"

徒弟：

> Bean 不就是 `反射 new 出来 → 放进 Map`？为什么有这么多阶段？

老陈：

> 裸 new 确实能造出对象，但是留下了一个账——**新造出来的对象是"空的"**：
> 依赖还没注入、配置还没绑定、初始化逻辑（连数据库、预热缓存）还没跑、
> 该被 AOP 代理的还没被包装。
> 如果容器只负责 new，这些"装配"逻辑又会散回业务代码——回到 Level 1 的三笔账。
> 所以 Spring 把"从裸对象到可用 Bean"的每一步都变成**可插拔的钩子**。
> **钩子板是设计核心：你不插钩子，容器按默认路径走；你插了钩子，容器在固定位置叫你。**

### 2.1.5 生命周期八步速查：员工入职到离职

**Spring Bean 的生命周期本质是"容器代为管理一个对象从出生到销毁的全过程"**：核心四步是**实例化 → 属性注入 → 初始化 → 销毁**，中间穿插**Aware 回调**和 **BeanPostProcessor 前后置增强**这两大扩展点。

一句话区分两大扩展点：**Aware 是 Bean 向容器"要"东西（感知环境），BeanPostProcessor 是容器对 Bean"做"东西（加工改造）**。

```text
容器启动 / getBean 触发
        │
        ▼
① 实例化   ← 招聘进门（人到了，两手空空）    createBeanInstance()
        │
        ▼
② 属性填充 ← 发工牌、配电脑、分工位（循环依赖的根源就在这）  populateBean()
        │
        ▼
③ Aware 回调 ← 告知公司信息：名字、部门、HR 系统  initializeBean() 内部
        │
        ▼
④ BPP 前置 ← 入职体检与背景核查（★ @PostConstruct 在此触发）  postProcessBeforeInitialization()
        │
        ▼
⑤ 初始化   ← 岗前培训：学业务、熟悉流程  afterPropertiesSet → init-method
        │
        ▼
⑥ BPP 后置 ← 发正式工装、分配导师（★ AOP 代理在此创建）  postProcessAfterInitialization()
        │
        ▼
⑦ 使用     ← 正式上班：放入单例池 singletonObjects（原型 Bean 每次 getBean 重复①~⑥）
        │
        ▼（容器关闭 context.close()）
⑧ 销毁     ← 离职交接：归还设备、清工位（原型 Bean 容器不管）  @PreDestroy → destroy → destroy-method
```

| 阶段 | 生活类比 | 关键动作 / 回调 | 源码入口 |
|------|---------|---------------|---------|
| ① 实例化 | 招聘进门，人到了但两手空空 | 反射调用构造器 / 工厂方法，生成"裸对象" | `createBeanInstance()` |
| ② 属性填充 | 发工牌、配电脑、分工位 | `@Autowired`、`@Value`、setter 注入依赖——**循环依赖的根源就在这**（依赖递归触发 getBean） | `populateBean()` |
| ③ Aware 回调 | 告知公司信息：你的名字、部门 | `BeanNameAware` → `BeanClassLoaderAware` → `BeanFactoryAware`（`ApplicationContextAware` 实际在下一阶段由 `ApplicationContextAwareProcessor` 执行，2.6 实证） | `initializeBean()` 内部 `invokeAwareMethods()` |
| ④ BPP 前置 | 入职培训前的体检与背景核查 | `postProcessBeforeInitialization()`，对**所有 Bean 全局生效**；`@PostConstruct` 在此被 `CommonAnnotationBeanPostProcessor` 触发——严格说它不是"初始化方法"，是"前置处理阶段被执行的注解" | `applyBeanPostProcessorsBeforeInitialization()` |
| ⑤ 初始化 | 岗前培训，学业务、熟悉流程 | `InitializingBean.afterPropertiesSet()` → 自定义 `init-method` | `invokeInitMethods()` |
| ⑥ BPP 后置 | 培训后发正式工装、分配导师 | `postProcessAfterInitialization()`，**AOP 代理在此创建**（`AbstractAutoProxyCreator` 匹配切点则返回代理替代原对象） | `applyBeanPostProcessorsAfterInitialization()` |
| ⑦ 使用 | 正式上班干活 | 放入单例池 `singletonObjects`，对外提供服务；原型作用域不入池，每次 getBean 重复①~⑥ | `addSingleton()` 后缓存命中返回 |
| ⑧ 销毁 | 离职交接，归还设备、清工位 | `@PreDestroy` → `DisposableBean.destroy()` → 自定义 `destroy-method`；**原型 Bean 容器不管销毁**，交出去就撒手（GC 回收） | `close()` → `destroyBeans` → `DisposableBeanAdapter.destroy()` |

把 Bean 想成一个新员工，容器就是公司 HR 系统：

- **① 实例化——招聘进门**：HR 把人招来了，有了工号，但工位没分、电脑没配、什么都不会干。对应容器调构造器 `new` 出对象，成员变量全为 null，依赖一个没注入。
- **② 属性填充——发装备**：发工牌、配电脑、分工位、介绍同事。对应 `populateBean()` 执行 `@Autowired` 注入依赖 Bean、`@Value` 注入配置值。这一步依赖的 Bean 还没创建，会递归先去创建它。
- **③ Aware 回调——告知公司信息**：HR 说"你叫 orderService，属于销售部（BeanFactory），公司组织架构图在这（ApplicationContext）"。让 Bean 感知自己在容器里的身份和环境。
- **④ BPP 前置——入职体检**：正式培训前做体检、背景核查。`@PostConstruct` 就是被 `CommonAnnotationBeanPostProcessor` 在这一步触发的。
- **⑤ 初始化——岗前培训**：体检通过后培训，学业务、熟悉流程、完成上岗准备：`afterPropertiesSet()` → `init-method`。
- **⑥ BPP 后置——发工装配导师**：发正式工装、分配带教导师、可能再包装一层。**AOP 代理就是在这步创建的**——匹配切点就返回代理替代原始对象。`@Transactional`、`@Async` 能生效就是因为进容器的不是原对象而是代理。
- **⑦ 使用——正式上班**：培训完毕、装备齐全，进单例池开始干活。`getBean()` / `@Autowired` 拿到的就是它。原型 Bean 每次 getBean 都重复①到⑥。
- **⑧ 销毁——离职交接**：容器关闭，办理离职：归还电脑、清理工位。**原型 Bean 容器不管**——就像外包人员项目结束就走，公司不管后续。

> 记忆锚：**四步主干是骨架（实例化→注入→初始化→销毁），两大扩展点是抓手（Aware 要、BPP 做）**。这张表是 Level 2 的"地图"，接下来 2.2 的实测输出把每一步钉死。

### 2.2 代码实证：一个 Bean 的一生（11 个钩子全打印）

完整代码：`experiments/code/src/demo02/LifecycleApp.java`。
把主线角色一生能实现的钩子全实现一遍，容器按什么顺序调用，全打出来。

实测输出（`./run.sh demo02.LifecycleApp`）：

```text
1. 实例化（构造器）
2. BeanNameAware: 我的名字是 lifecycleBean
3. BeanFactoryAware
4. BeanPostProcessor.postProcessBeforeInitialization
5. @PostConstruct（CommonAnnotationBeanPostProcessor 调用）
6. InitializingBean.afterPropertiesSet
7. initMethod(customInit)
8. BeanPostProcessor.postProcessAfterInitialization（★ AOP 代理在这里包装）
[使用期] serve() 被调用
9. @PreDestroy
10. DisposableBean.destroy
11. destroyMethod(customDestroy)
[容器关闭] 关闭动作完成
```

**把输出和源码路径对上**（`doCreateBean` 内部，Framework 6.1.14 Implementation）：

```text
doCreateBean
  ├─ 1  createBeanInstance()       ← 实例化（构造器/工厂方法/Supplier）
  ├─ 2~3 initializeBean() 内部: Aware 组回调（BeanNameAware → BeanFactoryAware → ...）
  ├─ 4   postProcessBeforeInitialization（每个 BeanPostProcessor 依次）
  ├─ 5   @PostConstruct（CommonAnnotationBeanPostProcessor 在 4 这一步内部触发）
  ├─ 6   InitializingBean.afterPropertiesSet
  ├─ 7   initMethod（BeanDefinition 里指定的方法）
  ├─ 8   postProcessAfterInitialization（★ AbstractAutoProxyCreator 在这里生成 AOP 代理）
  └─ 9~11 容器关闭: @PreDestroy → DisposableBean.destroy → destroyMethod
```

### 2.3 深挖：完整时序图——populateBean 和三级缓存到底插在哪

2.2 的 11 行实测里**没有出现**两个关键环节：属性填充（populateBean）和三级缓存（addSingletonFactory）。因为它们不属于"打印的钩子"，而是悄悄发生的。把 doCreateBean 生产线完整展开（Framework 6.1.14 Implementation）：

```text
doCreateBean(beanName)
  │
  ├─ ① createBeanInstance()          实例化（构造器注入在这里完成）
  │
  ├─ ② addSingletonFactory()         进三级缓存（若：单例 && allowCircularReferences && 创建中）
  │      ——必须在 populateBean 之前：因为 populateBean 里解析依赖时可能立刻要"早期引用"
  │
  ├─ ③ populateBean()                ★ 属性填充：@Autowired/@Value/setter 全部在这里
  │
  └─ ④ initializeBean()              初始化：
        ├─ invokeAwareMethods()       BeanNameAware → BeanFactoryAware（2、3）
        ├─ BPP.postProcessBeforeInitialization（4：ApplicationContextAwareProcessor、
        │     CommonAnnotationBeanPostProcessor 在这里触发 @PostConstruct=5）
        ├─ InitializingBean.afterPropertiesSet（6）→ initMethod（7）
        └─ BPP.postProcessAfterInitialization（8：★ AOP 代理在这里包装）
  │
  └─ ⑤ 完成 → addSingleton 进一级缓存，同时清理二级/三级（Level 4 的主角）
```

**两个最容易记错的时序**：
- **populateBean 在 Aware 之前**：所以 2.2 实测里 BeanNameAware 回调时 @Value 字段已经有值（2.6 小节用专门 demo 钉死这一点）；
- **addSingletonFactory 在 populateBean 之前**：因为 populateBean 的依赖解析随时可能触发 getBean 环——三级缓存必须先就位（Level 4 展开）。

**扩展点全景：谁在处理什么对象**（三类后处理器不能混为一谈——24 章 01 章对照，Boot 4.1 基线文档语义，与本篇 3.3.5/6.1.14 实测一致）：

| 扩展点 | 处理对象 | 典型动作 |
| ---- | ---- | ---- |
| BeanDefinitionRegistryPostProcessor | 注册表 | 增删改 BeanDefinition（ConfigurationClassPostProcessor 是注册了它才拿到"注解→定义"解析权） |
| BeanFactoryPostProcessor | BeanDefinition / BeanFactory | 修改元数据（如占位符替换） |
| BeanPostProcessor | Bean 实例 | 注入、代理、包装（AOP 代理站，第 8 步） |

> 📌 最容易错的一点：**BPP.beforeInitialization 不是构造器之前**——实例已经创建并完成属性填充（populateBean），只是还没执行初始化回调（对应 2.3 时序图：before 在第 ④ 步 initializeBean 内部，晚于 ③）。
> 由"处理对象"分层可直接推导：BDRPP/BFPP 操作的是"图纸"（BeanDefinition），BPP 操作的是"成品"（实例）——**改动时机决定能力边界**：想改 Bean 定义只能用前两类，想包装实例只能用 BPP。

**容器关闭全序**（不只是 Bean 销毁回调——Framework 6.1.14 `AbstractApplicationContext.doClose()` 反编译实证）：

```text
context.close() / JVM shutdown hook
  ├─ ① publishEvent(ContextClosedEvent)    ← 最先：监听器此时还能安全访问容器
  ├─ ② lifecycleProcessor.onClose()        ← SmartLifecycle 按 phase 倒序 stop（Specification）
  ├─ ③ destroyBeans()                      ← 单例销毁：@PreDestroy → DisposableBean.destroy → destroyMethod
  ├─ ④ closeBeanFactory()                  ← 关闭 BeanFactory 本身
  ├─ ⑤ onClose()                           ← 子类模板钩子（Web 容器等在这里关）
  └─ ⑥ resetCommonCaches()                 ← 类缓存重置（热部署工具依赖）
```

**因果**：为什么 ContextClosedEvent 必须最先？因为事件监听器需要"容器还可用"才能完成最后清理（Flush 观测、释放外部资源）；等销毁开始后再发事件，监听器里 getBean 就危险了。**为什么 SmartLifecycle.stop 在 destroyBeans 之前**（24 章 14 章对照，Boot 4.1 基线文档语义）：stop 是"服务级"停机（先拒新、停对外服务），销毁是"实例级"收尾（再毁对象）——生产陷阱：在 @PreDestroy 里才"关服务"往往太晚。
**时间预算**：SmartLifecycle 停止受 `spring.lifecycle.timeout-per-shutdown-phase` 约束（6.1.14 的 DefaultLifecycleProcessor 字段实证，属性语义待实测）——但 Boot 3.3.5 的 WebServer 优雅停机是另一条路（04 篇 Level 5 与 07 篇 Level 8 展开，**3.3.5 无 shutdown-timeout，排空无限等待，实测**）。

### 2.4 What：为什么初始化回调有"三种写法"

`@PostConstruct`（注解，声明式）、`InitializingBean`（接口，类型强约束）、`initMethod`（配置指定）解决同一件事（初始化），是历史的三个时代并存——注解时代最推荐，接口与配置是兼容包袱。
**执行顺序是 Specification**：先 @PostConstruct，再 afterPropertiesSet，最后 initMethod——6.1.14 实测输出 5→6→7 与之一致。

> 📌 最容易被误解的一点：**AOP 代理不是"额外特性"，它是生命周期的固定一站**。
> `BeanPostProcessor.postProcessAfterInitialization` 是代理工厂（AbstractAutoProxyCreator）插桩的位置——**你拿到的 Bean 可能不是原对象，而是代理**。
> 这也是 Level 4 循环依赖里"提前暴露"问题的根源。

### 2.5 代码实证：Full 模式 vs Lite 模式（@Bean 方法级单例的真相）

`@Bean` 方法在 `@Configuration` 类里（Full 模式）调用多次返回同一实例；在普通 `@Component` 里（Lite 模式）呢？
完整代码：`experiments/code/src/demo02/FullVsLiteApp.java`。

实测输出（`./run.sh demo02.FullVsLiteApp`）：

```text
Full 模式：调用两次 @Bean 方法，同一实例? true
Lite 模式：调用两次 @Bean 方法，同一实例? false  ← 每次调用都执行方法体！
```

**因果**（Implementation，6.1.14）：

```text
@Configuration 类（Full）  → CGLIB 生成子类，@Bean 方法被拦截，多次调用返回同一个单例（方法级缓存）
@Bean 在普通类/@Component（Lite）→ 不拦截，每次调用执行方法体 → 每次 new！
```

> 📌 **Lite 模式下 `@Bean` 方法每次调用都会执行方法体**——如果你在普通类里写 `@Bean` 方法并多次调用，会得到多个实例。
> Full 模式的"方法级单例"是 CGLIB 子类化带来的（Implementation），不是 `@Bean` 注解的语义。
> 线上翻车：把 `@Configuration` 误写成 `@Component`，配置类里的 `@Bean` 方法被调用两次 → 两个连接池。

### 2.6 代码实证：Aware 家族——"populateBean 先于 Aware"的铁证

完整代码：`experiments/code/src/demo02/AwareFamilyApp.java`。
让一个 Bean 同时实现 Aware 家族并持有 @Value 字段，在**每个回调里读这个字段**——如果 populateBean 真的先于 initializeBean，那么从 BeanNameAware 起字段就应该已经有值。

实测输出（`./run.sh demo02.AwareFamilyApp`）：

```text
1. 实例化（构造器）：@Value 字段 = null   ← 字段还没注入
2. BeanNameAware：@Value 字段 = order-service   ← populateBean 已完成！
3. BeanFactoryAware：@Value 字段 = order-service
4. EnvironmentAware：@Value 字段 = order-service（ApplicationContextAwareProcessor 内部先于 ContextAware）
5. ApplicationContextAware：@Value 字段 = order-service（同一个 BPP.before 处理器）
-- 自定义 BPP.before 触发（第二拨 Aware 之后、@PostConstruct 之前）
6. @PostConstruct：@Value 字段 = order-service
```

**七行输出钉死四件事**：

1. **populateBean 先于所有 Aware**：构造器里字段是 null，BeanNameAware 时已经是 order-service——字段注入在"实例化之后、initializeBean 之前"完成（对应 2.3 时序图 ③）；
2. **Aware 不是一个阶段，是两拨**：BeanNameAware/BeanFactoryAware 由 `initializeBean` 直接调用；EnvironmentAware/ApplicationContextAware 由 **ApplicationContextAwareProcessor**（一个 BeanPostProcessor）在 BPP.before 阶段调用；
3. **处理器内部顺序**：EnvironmentAware 先于 ApplicationContextAware（实测 4 在 5 前，Implementation，6.1.14）；
4. **第二拨 Aware 先于链上所有其他 BPP**：自定义 BPP 与 @PostConstruct 都发生在第二拨 Aware 之后（实测顺序：4/5 → 自定义 BPP → 6），因为 ApplicationContextAwareProcessor 永远在 BPP 链首（见下）。

**BPP 链实测**（同一 demo 打印 `getBeanPostProcessors()`，Implementation，6.1.14）：

```text
ApplicationContextAwareProcessor   ← 第二拨 Aware 的执行者，链首
ImportAwareBeanPostProcessor
BeanPostProcessorChecker
CustomBPP（自定义）
CommonAnnotationBeanPostProcessor ← @PostConstruct 的执行者
AutowiredAnnotationBeanPostProcessor
ApplicationListenerDetector
```

**为什么自定义 BPP 反在 @PostConstruct 之前**（链序不是注册顺序直推，两处机关）：
- **MergedBeanDefinitionPostProcessor 被"复制"到专用组最后注册**：`registerBeanPostProcessors` 把实现了该接口的处理器（CommonAnnotation/Autowired）额外收集进"merged 组"，等普通组注册完才注册它；而 `AbstractBeanFactory.addBeanPostProcessors` 是 **removeAll + addAll**——已注册过的处理器会被**从原位移除、链尾重放**。于是自定义 BPP 相对"前移"；
- **CommonAnnotation 在 Autowired 之前**：同为 PriorityOrdered 组，组内按 `getOrder()` 排序——CommonAnnotation 的 order 是 0（父类 `InitDestroyAnnotationBeanPostProcessor` 字段默认值），Autowired 是 `LOWEST_PRECEDENCE`（无 @Order 时的兜底值），0 排前面。

> 📌 这解释了 2.2 的 11 钩子实测为什么是 `4. BPP.before → 5. @PostConstruct`：demo02.LifecycleApp 里打印第 4 行的就是自定义 BPP——它恰好落在两拨之间，撞上了"merged 组后注册"的机关。别背顺序，背机制：**谁是链首（ApplicationContextAwareProcessor）、谁会被链尾重放（MergedBeanDefinitionPostProcessor）**。

**Aware 家族全览**（解决"Bean 怎么拿到容器能力"）：

| Aware | 拿到什么 | 注入者 |
| ---- | ---- | ---- |
| BeanNameAware | 自己在容器里的名字 | initializeBean 直接调用 |
| BeanFactoryAware | 自己所在的 BeanFactory | initializeBean 直接调用 |
| EnvironmentAware | Environment（配置汇总视图） | ApplicationContextAwareProcessor（BPP.before） |
| ApplicationContextAware | ApplicationContext（容器全能力） | ApplicationContextAwareProcessor（BPP.before） |
| BeanClassLoaderAware | 类加载器 | initializeBean 直接调用 |

> 📌 **为什么需要 Aware**：Bean 偶尔要反过来用容器能力（拿配置、发事件、找其他 Bean）。Aware 把"容器能力"显式注入，替代 Bean 自己去 static 里找容器——这是"依赖注入"原则的自我约束：容器能力也只通过注入给。

### 2.7 What：Bean 从哪来（四路注册）

| 注册方式 | 机制 | 典型场景 |
| ---- | ---- | ---- |
| 组件扫描 @ComponentScan | ClassPathBeanDefinitionScanner 扫包，默认只认 @Component 族（@Service/@Repository/@Controller/@Configuration 等） | 自己的业务类 |
| @Bean | @Configuration 里的工厂方法，方法返回值注册为 Bean | 第三方类/需要构造逻辑的 Bean |
| @Import | 直接导入类/ImportSelector/ImportBeanDefinitionRegistrar | 批量注册、条件导入 |
| @Conditional | 注册前先问"条件成立吗"（ConditionEvaluator 判定） | 环境/依赖/属性驱动的装配 |

四路注册都指向同一个终点：**把 BeanDefinition 写进注册表**（Level 1 的三件套），随后的创建路径完全一样。

### 2.8 坑：本层最容易翻车的六处

| 坑 | 真相 | 翻车现场 |
| ---- | ---- | ---- |
| 所有 BPP 也是 Bean | 容器必须先完成 BeanPostProcessor 的注册，再创建普通单例 | 在 BPP 里 @Autowired 一个普通 Bean → 循环/找不到（依赖注入时序问题，别在 BPP 里做正常业务注入） |
| BPP 依赖业务 Bean → 提前创建 | BPP 必须早注册才能处理后续 Bean；它若依赖业务 Bean，该 Bean 会被**提前实例化**，此时部分 BPP 尚未注册（24 章 01 章 P-I09，Boot 4.1 基线文档语义） | 被提前创建的业务 Bean 错过 AOP 代理/其他后处理——"这个 Bean 怎么没有事务/切面生效"（06 篇代理失效根因之一）；修正：BPP 用 `static @Bean` 注册、通过 ObjectProvider 延迟获取（3.3+ 场景 2 展开） |
| destroyMethod 推断 | @Bean 默认推断 `close()`/`shutdown()` 为销毁方法（6.x 行为） | 自定义类恰好有 close() 方法，被容器意外调用 |
| @PostConstruct 的包名 | Spring 6 用 jakarta.annotation（javax 已移除——语义变化） | 老代码升级后 @PostConstruct 静默失效 |
| 把 BPP 当成单个 Bean 的回调 | BeanPostProcessor 是**全局**的：注册一次对容器里所有 Bean 生效（所以叫"处理器"不叫"回调"） | 在某个业务类里"实现了 BPP 接口"期待只处理自己 → 它对所有 Bean 生效，行为不可控；@PostConstruct/@Autowired 能工作，背后都是内置 BPP 在统一处理 |
| 构造器里拿依赖 | 实例化先于属性填充：构造器执行时依赖还没注入（只有构造器注入的参数可用） | 在构造器里 `this.dao` 为 null（以为 @Autowired 字段已注入）——依赖得通过构造器参数或延迟到 populateBean 之后 |

### 2.9 Transfer

- **钩子与回调是同一种实现方式**：都是"框架在固定时机调用你实现的接口方法"（IoC 的落点之一）——差别只在注册与驱动：回调显式注册、事件驱动（ApplicationListener.onApplicationEvent）；钩子实现即生效、流程驱动（Aware/InitializingBean/BPP，每个 Bean 创建必然经过）。凡是"对象从创建到销毁每步交给框架"的体系（Android 的 Activity 生命周期、K8s 的 Pod 生命周期、Web 容器 Servlet 生命周期），全是同一套"调用点板子"。
- **阶段顺序就是契约**：AOP 代理必须发生在初始化之后（否则增强不了初始化方法）、属性填充之后（否则代理里依赖是空的）——顺序不是随便排的，每一步都是上一步的账。
- **元数据先行**：先有 BeanDefinition 再谈创建，让"条件装配"（@Conditional）和"运行时注册"（Registrar）成为可能。
- **能力注入而非能力自取**：容器能力（配置、事件、容器引用）也通过注入提供（Aware/注入点），而不是 Bean 去 static 里捞——这条纪律让一切可测试。

**本层留下的账**：钩子有了，但"属性填充"这一步里 **@Autowired 到底怎么找到依赖**？找到多个候选怎么办？这逼出下一层：注入解析链。

---
<a id="level-3"></a>

## Level 3 属性填充：依赖注入的解析链

### 前置知识关卡

* [ ] 知道 Level 2 的 populateBean 与 InstantiationAwareBeanPostProcessor
* [ ] 知道 @Qualifier/@Primary 是"限定候选"的注解
* [ ] 知道类型转换（String → int 等）

### 3.1 Why：`@Autowired` 的两笔账

徒弟：

> `@Autowired` 写上去不就完了？它自己会找。

老陈：

> 它确实会找，但是留下了两个账——**找错了**和**找不到**：
> 第一，按类型找，接口有两个实现（`Greeter` 有 `GreeterA/B`），找谁？——歧义账；
> 第二，依赖不存在、或类型转换失败，启动期才炸——**Spring 的注入错误全部在启动期暴露，代价是启动失败**。
> 所以需要一个**确定的解析链**：类型 → 限定 → 兜底 → 名称，把"找依赖"变成可预期、可调试的过程。

### 3.2 代码实证：解析链五场景

完整代码：`experiments/code/src/demo03/ResolutionChainApp.java`。
同一个接口 `Greeter`，五个容器分别制造五种命运。

实测输出（`./run.sh demo03.ResolutionChainApp`）：

```text
[场景1 单候选] 注入成功，hello() = A
[场景2 两候选无提示] 启动失败，异常 = NoUniqueBeanDefinitionException
  消息首行: No qualifying bean of type 'demo03.ResolutionChainApp$Greeter' available: expected single matching bean but found 2: greeterA,greeterB
[场景3 有@Primary] 注入成功，hello() = B(@Primary)
[场景4 有@Qualifier] 注入成功，hello() = A
[场景5 名称回退] 注入成功，hello() = B(名字=greeter)
```

注意场景 2 的异常消息：`expected single matching bean but found 2: greeterA,greeterB`——**Spring 把候选全列给你了**，排查就是看这个列表。

### 3.3 What：解析链（`doResolveDependency`，Framework 6.1.14 Implementation）

```text
① 类型匹配 findAutowireCandidates（收集所有类型兼容候选）
      ↓ 候选 > 1
② @Qualifier 缩小（值匹配 bean 名称/限定符）
      ↓ 仍 > 1
③ @Primary 标注者胜出
      ↓ 仍 > 1
④ 按字段名回退（候选名 == 字段名）
      ↓ 仍 > 1
⑤ 启动失败：NoUniqueBeanDefinitionException
```

| 方案 | 解决什么 | 留下什么问题 |
| ---- | ---- | ---- |
| 字段直接 new（Level 1 已死） | 无 | 耦合 |
| 按类型注入（@Autowired 默认） | 单候选场景最直观 | 多候选歧义 → 启动失败 |
| 类型 + @Qualifier | 同类型多实例精确指定 | 注解与代码耦合；改名要同步改 |
| 类型 + @Primary | 多候选时"默认选我" | 只有一个主；被 @Qualifier 覆盖 |
| 类型 → 名称回退 | 兜底：按字段名匹配候选名 | 隐式依赖命名约定，改名即失效 |

> 📌 最容易被误解的一点：**@Autowired 是"先按类型，多候选才用名称回退"**，不是"按字段名找 Bean"。
> 很多人背成"按名字注入"——那是 @Resource 的行为（@Resource 先按名称，Implementation 语义不同）。两者不能混着背。

### 3.3+ ObjectProvider：把"容器查找"推迟到调用时

**Why——直接 `@Autowired Foo` 欠的四笔账**：

1. **可选依赖**：Foo 可能不存在（如监控库、可选组件）→ 被迫 `required = false` + 一堆 null 判断；
2. **注入时机提前**：注入点解析时 Foo 就被创建，哪怕业务根本用不到它；
3. **多候选歧义**：@Primary/@Qualifier 之外的"运行时再决定"不灵活；
4. **时序问题**：在 BPP / 配置类早期阶段直接拿依赖，会把业务 Bean **提前实例化**（2.8 坑表的经典翻车）。

**What——本质**：注入 `ObjectProvider<Foo>` 时，注入的是**一个"查找能力"（懒引用），不是实例**——把"容器查找"从注入阶段推迟到 `getObject()` 调用时刻。解析链本身（@Primary/@Qualifier/@Order/名称回退）**一条不变**，只是执行时间变了。

**API 五种打开方式**（spring-beans 6.1.14 接口反编译确认）：

| API | 语义 | 场景 |
|---|---|---|
| `getObject()` | 必有，没有抛 NoSuchBeanDefinitionException | 延迟但不缺省 |
| `getIfAvailable()` | 没有返回 null | 可选依赖 |
| `getIfAvailable(Supplier)` | 没有用默认值 | 默认实现回退 |
| `ifAvailable(Consumer)` | 有才消费，否则不动作 | 优雅判空 |
| `getIfUnique()` | **唯一候选才返回，否则 null**（含 @Primary 细则，见下） | 多实现运行时裁决 |
| `stream()` / `orderedStream()` | 所有候选（后者按 @Order） | 策略集合遍历 |

**How——源码锚点**：`doResolveDependency` 识别注入点类型是 `ObjectProvider` → 直接返回懒引用（不解析目标）；调用时走完整 `getBean` → 同一套解析链。所以 @Primary/@Qualifier/@Order 在 ObjectProvider 上**全部生效**（实测场景 4：有 @Primary 时 `getIfAvailable()` 取主候选）。

**使用场景**（生产里最常碰到的六类）：

1. **可选依赖**：监控/上报组件存在才注册——`getIfAvailable()` 判空；
2. **BPP 与配置类里的延迟获取**：避免提前实例化业务 Bean（2.8 坑表的官方修法，03 篇 0.6 也用它）；
3. **运行时按条件选择实现**：策略模式多实现，`stream()` 遍历或 `getIfUnique()` 唯一性裁决；
4. **多候选歧义消除**：不想加 @Primary 时，运行时用 `getIfUnique()` 判空兜底；
5. **Boot 自动配置内部的标准用法**：配置类方法参数 `ObjectProvider<Foo>`（Boot 源码大量使用——条件不存在时不崩，存在才取）；
6. **默认值回退**：`getIfAvailable(() -> new DefaultImpl())`。

**实测钉死的两个机制细节**（demo03.provider.ObjectProviderApp，JDK 21.0.11 + Boot 3.3.5）：

```text
[场景3] 三候选（含 @Primary）：getIfUnique() = B(@Primary)（多候选但有唯一 @Primary → 返回 @Primary）
[场景3b] 两候选（无 @Primary）：getIfUnique() = null（多候选且无 @Primary → null）
[场景5] TimingConsumer 构造器执行 = ObjectProvider 已注入，LazyService 未创建
[场景5] getObject() 调用时：lazy = LazyService（@Lazy 单例，getObject() 首次触发创建）
[场景4] 有 @Primary：getIfAvailable() = B(@Primary)
```

1. **`getIfUnique()` 不是"多候选一律 null"**——多候选但有**唯一** @Primary 时返回 @Primary；无 @Primary 的多候选才返回 null（requireUnique 语义：先按优先级挑"最优唯一"）；
2. **ObjectProvider 只延迟"解析动作"，不延迟"单例创建"**——普通 @Component 单例在 preInstantiateSingletons 照样被创建；要实现"调用才创建"需配合 `@Lazy`（场景 5 的组合就是 ObjectProvider + @Lazy 单例）。

**生产规则**：高频热路径别每调一次 `getObject()` 就重走一次解析链（懒引用不等于缓存结果，需要缓存时把结果存在局部变量/字段）；"我要的是可选、延迟、运行时裁决"用 ObjectProvider，"我要的就是确定性的那个 Bean"直接用注入，别为用而用。

### 3.4 代码实证：三种注入姿势与"注入时机"

完整代码：`experiments/code/src/demo03/InjectionStylesApp.java`。

实测输出（`./run.sh demo03.InjectionStylesApp`）：

```text
[字段注入] 构造器执行时字段还是 null: true
[字段注入] @PostConstruct 时字段已注入: true
[setter注入] setter 被调用: true
[构造器注入] 构造器执行时 greeter 已注入: true（实例化期完成）
三种姿势全部可用: true
```

**这四行输出直接钉死了两个事实**：

1. **构造器注入发生在"实例化"阶段**——构造器执行时依赖已经拿到（而且是 `final` 字段，不可变）；字段/setter 注入发生在更晚的"属性填充"（populateBean）阶段——字段注入的 Bean 在构造器里依赖还是 null。
2. **字段注入靠的是反射暴力开锁**：`AutowiredAnnotationBeanPostProcessor` 拿到字段元数据，`setAccessible(true)` 直接写值。代价是违背封装、字段不可 final、测试时需要容器帮你初始化。

| 姿势 | 注入时机 | 依赖可变性 | 容器哲学 |
| ---- | ---- | ---- | ---- |
| 构造器注入 | 实例化（createBeanInstance） | final 不可变 | 推荐：依赖关系显式、可测试 |
| setter 注入 | 属性填充（populateBean） | 可变 | 可选：重配置场景 |
| 字段注入 | 属性填充（populateBean） | 可变 | 不推荐：隐藏依赖、靠反射、循环依赖温床 |

### 3.5 深挖：`@Value("${...}")` 到底是谁解析的（四场景实证）

完整代码：`experiments/code/src/demo03/ValueApp.java`，资源 `src/demo03/app.properties`。

实测输出（`./run.sh demo03.ValueApp`）：

```text
[无解析器·String字段] name 注入的是字面量: "${app.name}"
[无解析器·int字段] 启动失败: NumberFormatException (For input string: "${app.port}")
[有解析器] name = order-service, port = 8080 (类型: Integer), 缺失key回退 = defaultVal
[缺key无默认] 启动失败: IllegalArgumentException
  消息首行: Could not resolve placeholder 'no.such.key' in value "${no.such.key}"
```

**这四行输出是 @Value 的全部真相**：

1. 没有任何解析器时，`@Value("${app.name}")` 注入的就是**字面量字符串** `"${app.name}"`——注解自己不做任何解析；
2. 没有解析器时 int 字段直接启动失败（`"${app.port}"` 不是数字）——**类型转换发生在占位符解析之后**；
3. 注册 `PropertySourcesPlaceholderConfigurer` 后，`${}` 被替换成 Environment 里的值，再走类型转换（String → int），缺失的 key 用 `:` 后的默认值兜底；
4. key 缺失且无默认值 → 启动失败，异常消息直接点名是哪个占位符。

**因果链**：`@Value` 只是"声明注入源"→ 真正解析 `${}` 的是 `PropertySourcesPlaceholderConfigurer`（一个 BeanFactoryPostProcessor）→ 它把占位符替换成 Environment 里的值 → 才交给类型转换。

**这解释了配置中心热更新的经典事故**：@Value 的值取决于"解析时刻 Environment 里有什么"。Apollo/Nacos 要刷新 @Value，必须让"新值进 Environment + 重新注入"（SpringValueProcessor / RefreshScope 机制），裸 @Value 不会自己刷新（线上案例 3）。

### 3.6 专题：@Value 为什么是 NULL（六场景实测，对应生产"排查 NULL"）

> 你在生产遇到过 `@Value` 字段是 null 对吧？这不是运气问题，是六个确定场景之一。全部实测，逐一对号。
> 完整代码：`experiments/code/src/demo03/ValueNullApp.java`。

实测输出（`./run.sh demo03.ValueNullApp`）：

```text
[A] static 字段 @Value: null（null = 未注入）
[C1] 构造器里读 @Value 字段: null（null = 属性填充在构造器之后）
[C2] @PostConstruct 里读 @Value 字段: order-service（populateBean 已完成）
[D] static setter @Value: null（未注入 = static 方法注入点被忽略）
[B] new 出来的对象 @Value: null（null = 容器不管理它）
[E] System property 覆盖 properties 同名 key: system-cover
```

**六个场景对照表**（A/B/C/D/E 全部实测，6.1.14）：

| # | 场景 | 结果 | 根因（机制链） |
| ---- | ---- | ---- | ---- |
| A | `static` 字段上的 @Value | **null** | 注入点收集（AutowiredAnnotationBeanPostProcessor）不处理 static 字段/方法（6.1.14 实测，勿背成通用规律） |
| B | `new` 出来的对象里的 @Value | **null** | 对象不在容器里，没有人执行 populateBean——注入是"容器管理"的入场券 |
| C1 | 构造器里读 @Value 字段 | **null** | 属性填充在构造器之后（Level 3.4 已证：字段注入在 populateBean） |
| C2 | @PostConstruct 里读 @Value 字段 | 有值 | populateBean 已完成（2.3 时序图：populateBean 在 initializeBean 之前） |
| D | `static setter` 上的 @Value | **null** | static 方法注入点被忽略（同 A） |
| E | 同名 key 存在于多个来源 | 高优先级赢 | Environment 是 PropertySource 的优先级堆叠视图：System properties > OS 环境变量 > 文件（实测 system-cover 覆盖 file-value） |

**线上排查"@Value 是 null"的五问清单**（按概率排序）：

```text
① 这个对象是容器管理的吗？       ——不是 Bean（new 的/静态工具类里访问）→ 场景 B
② 字段/方法是 static 吗？        ——是 → 场景 A/D
③ 是在构造器里读的吗？           ——是 → 场景 C1（换成字段注入点读取，或注入进构造器参数）
④ key 真的存在吗？拼写对了吗？   ——错 → 启动期就炸（无默认值时），不会等到 null（见 3.5 场景 4）
⑤ 有没有被更高优先级覆盖？       ——有 → 场景 E（System property / OS env 抢了文件的值）
```

> 📌 **启动期炸 ≠ null 场景**：key 缺失（无默认值）时容器在启动期直接失败（3.5 场景 4），根本到不了"运行期读到 null"。运行期拿到 null 的场景只有 A/B/C1/D 四类——它们都是"**注入根本没发生**"，而不是"值解析失败"。这个区分是排查的第一步。

> 📎 本层只用"优先级"这个**结论**（System properties > OS env > 文件）；Environment 的完整机制——PropertySource 堆叠如何构成、默认来源从哪来、profile 与 config data 如何进入、Binder 如何绑定——在 01 篇配置体系（Level 4）展开。

### 3.7 What：@Value vs @ConfigurationProperties（八股高频，知其所以然）

| 维度 | @Value | @ConfigurationProperties |
| ---- | ---- | ---- |
| 绑定方式 | 单值注入（${} 占位符 + SpEL） | 整组前缀绑定（Binder，宽松绑定 kebab-case→camelCase） |
| 校验 | 无 | 支持 @Validated + Bean Validation |
| 类型安全 | 弱（字符串手动转） | 强（编译期类型 + 绑定错误启动期暴露） |
| 复用 | 每处各写一遍 | 一个类集中管理，可注入多处 |
| 热刷新 | 默认不支持（需框架支持，如 Apollo 的 SpringValueProcessor） | 支持（配合 RefreshScope/配置中心刷新机制） |

**适用边界**：单值、简单、一次性 → @Value；一组相关配置、需要校验、需要热更新 → @ConfigurationProperties。

### 3.8 坑：本层最容易翻车的三处

| 坑 | 真相 | 翻车现场 |
| ---- | ---- | ---- |
| 注入错误在启动期炸 | 是特性不是 bug——把"配置对不上"的窗口从运行期挪到启动期 | 以为"运行期会优雅降级"，实际启动直接失败 |
| @Value 解析不了 | `${}` 依赖 Environment + PSC（Boot 里自动配好，裸 Spring 要自己注册） | 裸 Spring demo 里 @Value 全是字面量 |
| 字段注入的环 | 字段注入把环藏起来，让循环依赖"能跑" | 升级 Boot 2.6+ 启动爆炸（Level 4 + 线上案例 1） |
| static 字段 @Value | static 注入点被忽略（6.1.14 实测） | 工具类里 static @Value 永远是 null（线上案例 4） |
| Environment 优先级 | System properties > OS env > 文件；同名 key 高优先级赢 | 本地能跑、线上 null：某台机器有同名环境变量覆盖 |

### 3.9 Transfer

- **歧义消解三件套（Qualifier/Primary/名称回退）是通用模式**：配置中心的多环境取值、路由的多服务匹配、规则引擎的多规则命中，全是"精确指定 → 默认兜底 → 命名约定"三层消解。
- **启动期失败优于运行期失败**：把"配置对不上"的窗口从运行期挪到启动期，是配置治理的第一性原理。
- **默认值是"歧义时怎么办"的回答**：任何可配置系统都要回答三个问题：候选谁优先（Primary）、精确怎么指（Qualifier）、都没了怎么办（名称回退/默认值）。

**本层留下的账**：注入解析链是**有向的**——A 依赖 B、B 依赖 A 时，解析会转圈：**循环依赖**。这逼出下一层：三级缓存。

---
<a id="level-4"></a>

## Level 4 循环依赖：三级缓存与"被禁止的好特性"

### 前置知识关卡

* [ ] 知道 Level 2 的创建顺序（实例化 → 属性填充 → 初始化）
* [ ] 知道构造器注入发生在实例化阶段（早于属性填充）
* [ ] 知道 AOP 代理在 postProcessAfterInitialization（晚于属性填充）

### 4.1 Why：为什么 Spring 会"支持"循环依赖

徒弟：

> A 依赖 B、B 依赖 A，报个错让开发者改不就行了？为什么要支持循环依赖？

老陈：

> 报错确实是 Boot 2.6+ 的默认做法（**语义变化**），
> 但 Spring 原生是支持字段/setter 循环依赖的——为什么？
> 因为**早期代码大量用字段注入**（Level 3 说过，字段注入能让循环"藏起来"），
> 直接禁掉会让整个生态的旧项目升级即炸。
> 所以 Spring 的取舍是：默认允许（早期）、Boot 2.6 起默认禁止（新项目）、
> 需要时 `spring.main.allow-circular-references=true` 显式放开。
> **循环依赖从来不是"好特性"，是字段注入时代的兼容包袱。**

| 方案 | 解决什么 | 留下什么问题 |
| ---- | ---- | ---- |
| 构造器注入 + 禁止循环 | 依赖关系在编译期显式、无隐藏环 | 结构上有环时无法启动（必须重构） |
| 字段注入 + 三级缓存 | 允许 setter/字段环 | 环被隐藏，AOP/代理环仍可能失败 |
| 三级缓存（DefaultSingletonBeanRegistry） | 让"属性填充阶段的环"能绕过去 | 只对"实例化已完成、正在填充"的环有效；构造器环、代理环仍死 |

### 4.2 代码实证：字段注入的环，能跑通吗？

完整代码：`experiments/code/src/demo04/FieldCircularApp.java`。
A 依赖 B、B 依赖 A（全部字段/setter 注入），并验证"B 在属性填充阶段拿到的 A 与最终 A 是同一引用"。

实测输出（`./run.sh demo04.FieldCircularApp`）：

```text
B 的属性填充阶段拿到 A（早期引用）
A 创建完成
环成立: A.b != null && B.a != null -> true
B 手里的 A 与容器最终暴露的 A 是同一引用: true
属性填充时刻的 A 与最终 A 是同一引用: true
```

**引用一致性 = 三级缓存存在的意义**：如果 B 拿到的是"裸 A"、而其他人拿到的是"最终 A"，同一个 Bean 就出现了两个身份——数据不一致的定时炸弹。三级缓存用 ObjectFactory 延迟生成"早期引用"，保证全容器只有一个版本。

**同款环，裸 Framework 与 Boot 行为不同**（交叉验证定案，2026-08-06 实测）：

| 启动路径 | 默认 allowCircularReferences | 实测 |
| ---- | ---- | ---- |
| 裸 `AnnotationConfigApplicationContext`（demo04） | true | 字段环创建成功 |
| Boot `SpringApplication`（demo15.BootCircularApp，3.3.5） | **false（默认禁止）** | 启动失败 `BeanCurrentlyInCreationException` |

```text
同一套组件代码，换启动入口行为翻转：
  demo04.DisableCircularApp  裸容器 + 显式关闭 → 失败（与 Boot 默认行为一致）
  demo15.BootCircularApp     Boot 3.3.5 默认参数 → 失败（"升级 Boot 2.6+ 后启动爆炸"的机制实证）
  demo15.BootCircularApp     带 --spring.main.allow-circular-references=true → 启动成功（放行开关实测生效）
```

**因果**：三级缓存是 DefaultListableBeanFactory 的实现能力（默认开）；但 Boot 从 2.6 起在 SpringApplication 启动路径上把开关置为 false（文档语义）——**为什么敢关**：循环依赖是"可运行但不健康"的设计（绕开了构造器注入的不可变约束、掩盖职责边界），Boot 选择默认拒绝而非默认容忍。本机 3.3.5 实测确认默认即失败，无需任何配置。
**生产含义**：老项目升级 Boot 2.6+ 后"字段环突然爆炸"不是新 bug，是默认值翻转（见文末版本勘误表）；想临时放行可用 `spring.main.allow-circular-references=true`（3.3.5 属性，2026-08-06 实测放行成功——应急开关，根因要拆环）。放行路径的绑定链（3.3.5 反编译实证）：命令行参数进 `SimpleCommandLinePropertySource` → `prepareEnvironment` 内 `bindToSpringApplication` → `Binder.bind("spring.main", Bindable.ofInstance(this))` → `setAllowCircularReferences` → run 阶段传播到 `BeanFactory.setAllowCircularReferences`（先于 refresh）。

### 4.3 What：三级缓存为什么是三级（Framework 6.1.14 Implementation）

```text
一级 singletonObjects：完整创建好的 Bean（权威事实）
二级 earlySingletonObjects：提前暴露的"早期引用"（已实例化、未完成填充/初始化）
三级 singletonFactories：ObjectFactory 工厂（真正"创建早期引用"的动作，延迟执行）
```

**为什么两级不够**（这是全篇最经典的八股因果）：

```text
只有一级+二级：A 实例化后直接放"裸的 A"到二级缓存
    → B 拿到裸 A，但 A 后续需要 AOP 代理
    → 代理在 afterInitialization 才生成
    → B 手里是"裸 A"，别人手里是"A 代理" → 同一 Bean 两种身份，引用不一致！

加三级（ObjectFactory）：二级缓存存"怎么生成早期引用"的工厂
    → 需要时调工厂：如果该 Bean 命中 AOP 场景，工厂直接给"早期代理"
    → 此后 afterInitialization 通过 earlyProxyReferences 识别"已暴露过"，不再二次包装
    → 全容器引用一致
```

```text
创建 A：
  ① 实例化 A（裸对象）        → 加入三级缓存：singletonFactories[A]
  ② 填充 A：需要 B → getBean(B)
        └─▶ 创建 B：实例化 B → 加入三级缓存
             填充 B：需要 A → getBean(A)
                 一级没有 → 二级没有 → 三级有 → 调工厂拿到"早期 A"
                 → 放入二级缓存 earlySingletonObjects[A]
                 → B 注入早期 A
             初始化 B 完成 → 一级缓存[B]（addSingleton：一级入 + 删三级 + 删二级；B 未提前暴露，删二级为空操作）
  ③ A 拿到 B → 注入 → 初始化 A 完成 → 一级缓存[A]（addSingleton：一级入完整 A，同时清除三级[A]与二级[A]中的早期引用——早期 A 从未进过一级，是"清除"不是"替换"）
```

**addSingleton 的完整职责**（6.1.14 `DefaultSingletonBeanRegistry` 字节码实证）：每次 Bean 完成初始化，统一执行 `一级 put + 三级 remove + 二级 remove + registeredSingletons.add`——"完成态"只允许出现在一级，二级/三级里该 Bean 的所有痕迹（工厂、早期引用）一律清除。所以 4.3 时序里 A、B 两条线做的是**同一个动作**，差异只在"删二级"对 A 是有效操作（二级里真有早期 A），对 B 是空操作（B 从未被提前引用）。

- **核心不变量**：提前暴露的引用与最终 Bean 是**同一个引用**（或同一个代理链）；三级缓存中每个 Bean 至多保留一条
- **适用范围**（必须背清边界）：只解决 **setter/字段注入**的环；**构造器注入的环、@Async 代理的环、prototype scope 的环**全部失败

### 4.4 代码实证：三种"救不了"的环

完整代码：`experiments/code/src/demo04/` 下三个 App。

**① 构造器注入的环**（`CtorCircularApp`）——构造器注入发生在实例化阶段，此时**还没有任何对象可以提前暴露**，三级缓存无从救起：

```text
启动失败: BeanCurrentlyInCreationException
消息首行: Error creating bean with name 'ctorCircularApp.CtorA': Requested bean is currently in creation: Is there an unresolvable circular reference?
```

**①的解法：构造器参数上加 @Lazy——打破构造器环的唯一兼容方案**（`LazyCtorCircularApp`）：

```text
[验证] CtorA 构造：b = demo04.LazyCtorCircularApp$CtorB$$SpringCGLIB$$0（懒代理）
[验证] CtorB 构造执行（A 已就绪——环退化成顺序链；B 作为普通单例在启动时照样创建）
[验证] 容器启动成功：构造器环被 @Lazy 参数打破
[验证] A→B 调用：B
```

机制：参数上加 `@Lazy` → 注入点**不解析真实实例**，注入 CGLIB 代理（`$$SpringCGLIB$$` 命名，Spring 6 repackaged cglib）。打破环的本质是——**A 创建时不需要先创建 B（拿到的是代理）**，环退化成"顺序链"：A 完成 → B 创建（A 已就绪）→ 启动成功。

两个必须钉死的语义：

1. **@Lazy 推迟的是"解析动作"，不是"单例创建"**——B 作为普通单例，容器启动时（preInstantiateSingletons）照样会被创建（实测输出第二行）；和 3.3+ 的 ObjectProvider 同理，"调用才创建"需要 B 自身也 @Lazy；
2. **"唯一兼容方案"的限定语义**——在"保持构造器注入风格、依赖类型不变、调用方代码零改动"的前提下，@Lazy 是唯一解法；`ObjectProvider` 参数也能破环（同样是懒引用），但调用方必须改成 `getObject()`——获取形态变了，不如 @Lazy 透明。

> ⚠️ 治标不治本：@Lazy 是把环变成顺序链，**环本身还在**（1017 行条目）。生产上优先消除环（抽依赖、分层），@Lazy 是"改造不动/三方库强环"时的兜底。

**② 显式关掉开关**（`DisableCircularApp`）——`allowCircularReferences` 是容器的开关，关掉后字段环也死。**Boot 2.6+ 的 SpringApplication 启动时对容器默认执行了这个关闭动作**（Boot 语义变化），这就是"升级后老项目启动爆炸"的根因：

```text
[默认开启] 字段环创建成功: true
[显式关闭] 字段环启动失败: BeanCurrentlyInCreationException
消息首行: Error creating bean with name 'disableCircularApp.A': Requested bean is currently in creation: Is there an unresolvable circular reference?
```

**③ @Async 代理的环**（`AsyncCircularApp`）——B 在属性填充阶段拿到"原始版本"的 AsyncA，但 AsyncA 最终被 @Async 包装成代理。容器发现"同一 Bean 出现了两个版本"→ 直接拒绝：

```text
启动失败: BeanCurrentlyInCreationException
消息首行: Error creating bean with name 'asyncCircularApp.AsyncA': Bean with name 'asyncCircularApp.AsyncA' has been injected into other beans [asyncCircularApp.AsyncB] in its raw version as part of a circular reference, but has eventually been wrapped...
```

**这条异常消息值得逐字读**："has been injected into other beans in its **raw version** ... but has eventually been **wrapped**"——它就是 4.3 那段"为什么两级不够"的因果在运行时的原话。

### 4.5 How：三级缓存的接入点（Framework 6.1.14 Implementation）

```text
doCreateBean 实例化完成、合并 BeanDefinition 处理后：
  单例 && allowCircularReferences && 当前在创建中
      └─▶ addSingletonFactory(beanName, ObjectFactory)   ← 进三级缓存
getBean 依赖方找不到完整 Bean → getSingleton(beanName, false)
      └─▶ 一级没有 → 二级没有 → 三级有 → 调工厂生成早期引用 → 进二级缓存
原 Bean 完成初始化 → addSingleton(beanName, bean) → 进一级缓存 → 二级/三级清理
```

**最反直觉的一步**：循环依赖能跑通，本质是"**发布不完整对象**"——违背单例池"要么没有、要么完整"的直觉。

徒弟：

> 那提前暴露的 A 是"没初始化完的 A"，别人拿到它调用方法不会出事吗？

老陈：

> 会——所以它从来不是免费的。
> 早期引用暴露时，A 的属性和初始化还没完成，
> 如果 B 在此时调用 A 的方法，拿到的可能是"半成品"。
> 这就是为什么 Boot 2.6 默认禁止：**让循环依赖显式失败，
> 比让半成品对象在运行时悄悄出错更安全**。
> 容器哲学：宁可启动失败，不可运行时悬着。

### 4.6 坑：本层最容易翻车的三处

| 坑 | 真相 | 翻车现场 |
| ---- | ---- | ---- |
| 三级缓存不是规范 | 是 DefaultSingletonBeanRegistry 的实现细节，版本间可变 | 面试背"规范保证必须三级"是八股病 |
| @Lazy 能"缓解"环 | @Lazy 注入的是代理，绕开了解析时刻的环——**把环变成顺序链，不消除环本身**；且推迟的是"解析动作"不是"单例创建"（4.4① 解法实测） | 治标不治本：环依旧在；以为"B 永远不创建"其实是错的（普通单例照样在启动时创建） |
| 升级 2.6 后环爆炸 | Boot 默认关掉了 allowCircularReferences（语义变化） | 老项目升级启动失败，第一排查项就是循环依赖 |

### 4.7 Transfer

- **"发布不完整对象"的通用账**：任何缓存系统都要回答"别人能不能看到还没写完的值"（写缓存时先写完整再发布 vs 先占位后填充）；JMM 的双检锁、K8s 的 ready 门控都是同一问题的不同答案。
- **兼容包袱 vs 新纪律**：旧特性因生态被保留，新版本默认收紧（Boot 2.6 禁环）——升级时的"默认值变化"就是最贵的坑，必须逐项核对语义变化清单。
- **实现细节不是规范**：学源码要分清"Spring 怎么做的"和"Spring 承诺什么"，把 DefaultSingletonBeanRegistry 当规范背是八股病的典型症状。

**本层收账**：到这里，"创建对象"的完整因果链闭合——**类 → 说明书（BeanDefinition）→ 容器（注册表+反射+单例池）→ 钩子板（生命周期）→ 换乘站（依赖注入）→ 兜底（三级缓存）→ 可用 Bean**。

---
<a id="summary-graph"></a>

## 全篇因果链总图

```text
你的类（@Service/@Bean/...）
   │ ① 被翻译成
   ▼
BeanDefinition 说明书 ──▶ 注册表（Level 1）
   │ ② refresh → finishBeanFactoryInitialization → preInstantiateSingletons
   ▼
doCreateBean 生产线（Level 2）
   ├─ 实例化 ────────────────────────── 构造器注入在这里完成（Level 3）
   ├─ addSingletonFactory 进三级缓存 ── 循环依赖的兜底（Level 4）
   ├─ populateBean ─────────────────── 字段/setter 注入 + @Value 在这里（Level 3）
   ├─ Aware → BPP.before → @PostConstruct → afterPropertiesSet → initMethod
   ├─ BPP.after ────────────────────── ★ AOP 代理在这里包装
   └─ 进一级缓存 → 可注入、可 getBean

对外入口：getBean = 决策树（缓存命中/FactoryBean/&name/byType/@Lazy，Level 1.6）
扩展结构：父子容器 = 单向委派查找（子 → 父，同名子优先，Level 1.7）
```

**七字口诀**：

```text
new 死三账容器生，钩子七站养 Bean，
换乘站里解析链，三级缓存兜底环。
```

---

<a id="prod-cases"></a>

## 🏥 线上案例：三个真实复盘（全部命中本篇四层）

> 复盘格式：现象 → 根因 → 机制链 → 修复 → 预防。数字只做量级描述，不做精确承诺（待实测）。

### 案例 1：升级 Boot 2.6+ 后启动失败——循环依赖"突然"爆炸

- **现象**：老项目从 Boot 2.3 升 2.6+，启动报 `BeanCurrentlyInCreationException`，一堆 Bean 报错。
- **根因**：项目大量字段注入，隐式存在 A↔B 环。Boot 2.6 起 `SpringApplication` 默认对容器设置 `allowCircularReferences=false`（语义变化）。
- **机制链**：Level 4——开关关闭 → 三级缓存失去作用 → 字段环启动失败。
- **修复**：`spring.main.allow-circular-references=true` 临时放开（治标），架构上拆分 A/B 的环（治本：改为构造器注入 + 按依赖方向重构，环消失则根本不需要开关）。
- **预防**：新代码禁止字段注入环；依赖方向自上而下（Service 层不反向依赖 Controller）；CI 里加启动冒烟测试。

### 案例 2：`@Bean` 方法被调用两次，出现了两个连接池

- **现象**：自定义 starter 里一个 `@Bean RedisConnectionFactory`，线上出现两个连接池，连接数翻倍。
- **根因**：配置类误写成 `@Component`（Lite 模式），别处调用了该 `@Bean` 方法，方法体重新执行。
- **机制链**：Level 2——Lite 模式不拦截 `@Bean` 方法，每次调用执行方法体。
- **修复**：配置类改回 `@Configuration`（Full 模式，CGLIB 拦截 `@Bean` 方法返回容器单例）；注意**不能**靠"把 `@Bean` 方法改成 `static`"来修——static 方法不被 CGLIB 子类覆盖，外部调用照样执行方法体（6.1.14 行为，Implementation）。
- **预防**：配置类统一 `@Configuration`；code review 规则：`@Bean` 方法不要从外部直接调用，要拿实例就 `getBean` 或注入。

### 案例 3：配置中心热更新不生效——`@Value` 的"静态注入"账

- **现象**：Apollo 改了配置，`@Value` 字段不刷新；重启才生效。
- **根因**：`@Value` 在创建时解析一次（占位符 → Environment 当时的值），之后不重新解析（Level 3 四场景实证）。
- **机制链**：Level 3——PSC 解析发生在 populateBean；`@Value` 字段没有重新注入机制。
- **修复**：改用 `@ConfigurationProperties` + 配置中心刷新机制（Apollo 的 `@ApolloConfigChangeListener` / Nacos `@RefreshScope`）；或用 `Environment` 主动 `getProperty`。
- **预防**：需要热更新的配置一律走 `@ConfigurationProperties`；`@Value` 只用于一次性静态值。

### 案例 4：`@Value` 字段是 NULL——六场景排查实录

- **现象**：某服务某配置偶发 null，本地复现不了；日志里只有 NPE，没有启动失败。
- **根因**：先答五问（3.6 清单）——①对象是容器管理的吗？②字段/方法是 static 吗？③是在构造器里读的吗？④key 拼写对吗？⑤被更高优先级覆盖了吗？真实事故通常落在 **A（static 字段）**、**B（new 出来的对象）**、**E（OS 环境变量同名覆盖）** 三类。
- **机制链**：Level 3——static 注入点被忽略（A/D 实测）；非容器对象无 populateBean（B 实测）；Environment 优先级堆叠（E 实测）。
- **修复**：A→去掉 static，改容器管理实例注入；B→把 new 改造成 Bean 或从构造器参数注入；E→排查机器环境变量（`env | grep KEY`），改 key 名或显式在配置里覆盖。
- **预防**：代码评审禁止 static 字段注入；工具类需要配置时用构造器参数传入；@Value key 全局唯一命名（加模块前缀）。

---

<a id="interview"></a>

## 🎤 面试八股自查表（本篇范围的知其所以然）

| 八股问题 | 一句话因果答案 | 证据在哪 |
| ---- | ---- | ---- |
| 为什么需要 IoC 容器？ | new 欠三笔账（耦合/生命周期/测试），工厂解决不了"创建动作散落" | Level 1.1 |
| 容器里存的是对象吗？ | 不，存 BeanDefinition 说明书，创建是反射 + 单例池 | Level 1.3 实测：注册后单例池=0 |
| getBean 的完整决策分支？ | 缓存命中 / FactoryBean 产品与 &name / byType 单多候选 / @Lazy 代理 | Level 1.6 四分支实测 |
| 父子容器怎么查找 Bean？ | 子 → 父单向委派；同名子优先；父看不到子 | Level 1.7 五事实实测 |
| 一个 Bean 的生命周期顺序？ | 实例化 → populateBean → Aware → BPP.before → @PostConstruct → afterPropertiesSet → initMethod → BPP.after → 销毁三步 | Level 2.2 实测 11 行 |
| populateBean 在 Aware 之前还是之后？ | 之前——BeanNameAware 回调里 @Value 字段已有值 | Level 2.6 实测 6 行 |
| @PostConstruct / InitializingBean / initMethod 顺序？ | 注解 → 接口 → 配置（Specification） | Level 2.2 实测 5→6→7 |
| @Bean 方法调用多次会怎样？ | Full 模式 CGLIB 拦截返回同一实例；Lite 模式每次 new | Level 2.5 实测 |
| @Autowired 怎么找 Bean？ | 类型 → Qualifier → Primary → 名称回退 → NoUnique 失败 | Level 3.2 五场景实测 |
| @Value 的 ${} 谁解析？ | PropertySourcesPlaceholderConfigurer，不是注解自己 | Level 3.5 四场景实测 |
| @Value 为什么可能是 null？ | 四个场景：static 注入点被忽略 / 非容器对象 / 构造器时机 / static setter | Level 3.6 六场景实测 |
| 构造器注入和字段注入区别？ | 时机：实例化期 vs 属性填充期；可变性：final vs 非 final | Level 3.4 实测 |
| 为什么三级缓存是三级？ | 二级给"裸 A"，代理在 after 生成 → 引用不一致；三级延迟生成早期引用 | Level 4.3 |
| 循环依赖什么情况救不了？ | 构造器环、@Async 代理环、prototype 环 | Level 4.4 三个实测 |
| 为什么 Boot 2.6+ 默认禁止循环依赖？ | 发布不完整对象的代价 > 兼容便利；显式失败优于运行时悬着 | Level 4.5 |

---

<a id="pitfalls"></a>

## ⚠️ 坑与细节（10 个真实误解）

### 坑 1：单例 = 线程安全

单例只是"一个实例"；它内部有可变状态照样线程不安全。`@Service` 的类字段若被多个请求写，需要自己加锁或改为无状态。

### 坑 2：prototype Bean 被容器管理生命周期

prototype 创建后容器不再跟踪，销毁回调（@PreDestroy）不会执行——它由使用者负责清理（Level 1.8）。

### 坑 3：@Component 的默认命名规则

默认 bean 名 = 类名首字母小写；**嵌套静态类会带外层类名前缀**（1.6 实测：`GetBeanTreeApp.DualServiceA` 的候选名是 `getBeanTreeApp.DualServiceA`）——@Qualifier 和 @Resource 指名时要写对。

### 坑 4：Lite 模式的 @Bean 是"每次 new"

`@Configuration` 必须写全。`@Component` 类里写 `@Bean` 方法、又手动调它 = 多实例（Level 2.5 实测 + 线上案例 2）。

### 坑 5：@PostConstruct 在 BPP.before 里被调用，不是独立阶段

所以"在所有 BeanPostProcessor 之前执行"是错的——它是 CommonAnnotationBeanPostProcessor 这个 BPP 的 before 内部动作（Level 2.2 实测：4 打印在前、5 在后，但 4 包含 5）。

### 坑 6：字段注入的"暴力开锁"

字段注入靠 `setAccessible(true)` 写私有字段；JDK 17+ 模块化应用（JPMS）里强反射有运行时警告/报错，非模块 classpath 应用不受影响（本文 demo 就是后者）——能注入 ≠ 推荐注入（Level 3.4）。

### 坑 7：@Autowired 不是"按名字找"

默认按类型；名字只是最后兜底。@Resource 才先按名字（Level 3.3）。

### 坑 8：循环依赖报错不会直接说"循环"

`BeanCurrentlyInCreationException` 的 "Requested bean is currently in creation" 才是标志；开 debug 日志（Boot：`logging.level.org.springframework.beans.factory=debug`）能打印循环链。

### 坑 9：@Lazy 治标不治本

@Lazy 注入代理、延迟解析，环没消失只是被推迟；且 @Lazy 代理和真实 Bean 类型不一致时会踩 instanceof 坑。

### 坑 10：销毁方法推断

@Bean 默认推断 close/shutdown 为销毁方法——自定义类若恰好有 public `close()`，容器关闭时会意外调用（Level 2.8）。

---

<a id="errata"></a>

## 📚 版本勘误表

> 勘误格式：我曾讲错的 → 真相。区分 Specification / Implementation / 语义变化。

| 我曾讲错的 | 真相 | 性质 |
| ---- | ---- | ---- |
| "三级缓存是 Spring 的规范" | 是 DefaultSingletonBeanRegistry 的实现细节（Implementation） | Implementation |
| "Boot 默认禁止循环依赖"（说成 Framework 行为） | Framework 默认允许（allowCircularReferences=true）；是 Boot 2.6+ 的 SpringApplication 启动时关闭（语义变化）——2026-08-06 实测定案：demo15.BootCircularApp（Boot 3.3.5 默认参数字段环启动失败 BeanCurrentlyInCreationException）vs demo04.DisableCircularApp（裸容器显式关闭后同样失败） | 语义变化（Boot）+ 实测 |
| "没有 @Value 就注入失败" | 没有 PSC 时 String 字段注入的是字面量 `${...}`；int 等类型字段才启动失败 | 实测（Level 3.5） |
| "@PostConstruct 是独立生命周期阶段" | 它被 CommonAnnotationBeanPostProcessor 在 postProcessBeforeInitialization 内调用 | Implementation |
| "构造器注入只是写法不同" | 时机完全不同：实例化期 vs 属性填充期；final 不可变 vs 可变 | 实测（Level 3.4） |
| "javax.annotation.PostConstruct" | Spring 6 / Boot 3 已迁移 jakarta（javax 移除） | 语义变化 |
| "@Bean 方法自带单例语义" | 方法级单例来自 @Configuration 的 CGLIB 子类化；Lite 模式下每次调用都执行 | Implementation |

---

<a id="decisions"></a>

## 🏆 生产决策卡

### Decision Card 1：升级 Boot 2.6+，循环依赖爆炸怎么办

- **场景**：老项目升级，启动报 BeanCurrentlyInCreationException。
- **判断**：先确认是字段/setter 环（看异常 bean 名），再决定治标还是治本。
- **Mechanism → Decision**：allowCircularReferences 是容器开关 → 治标：`spring.main.allow-circular-references=true`；治本：重构依赖方向 + 构造器注入。
- **Code**：

```yaml
spring:
  main:
    allow-circular-references: true   # 治标，仅临时
```

- **禁止决策**：不允许"加 @Lazy 绕一圈就算完"（调用期才解析，事故后置）。
- **验收指标**：启动通过 + 无环的构造器注入审计清单；重构后移除该开关再验证。

### Decision Card 2：注入姿势选型（新项目）

- **场景**：新模块，依赖关系明确。
- **判断**：构造器注入优先（依赖显式、final、可测试）；确实需要运行时替换依赖才考虑 setter。
- **Mechanism → Decision**：构造器注入在实例化期完成 → 依赖缺失在启动即暴露；字段注入隐藏依赖且滋生环 → 禁止默认使用。
- **Code**：

```java
public class OrderService {
    private final OrderRepository repo;
    public OrderService(OrderRepository repo) { this.repo = repo; }   // 构造器注入
}
```

- **禁止决策**：新代码禁止字段注入 @Autowired 私有字段（除非无可避免的框架回调场景）。
- **验收指标**：新模块零字段注入；依赖图无环（自定义 BeanFactoryPostProcessor 遍历 BeanDefinition 的属性引用/构造参数引用，构建依赖图后查环，待实测）。

### Decision Card 3：配置用 @Value 还是 @ConfigurationProperties

- **场景**：接入配置中心，一组业务配置需要热更新。
- **判断**：单值一次性 → @Value；一组 + 校验 + 热更新 → @ConfigurationProperties。
- **Mechanism → Decision**：@Value 解析一次（创建时刻）；@ConfigurationProperties 走 Binder + RefreshScope 可重绑定。
- **Code**：

```java
@ConfigurationProperties(prefix = "app.order")
@Validated
public class OrderProps {
    @NotNull private String callbackUrl;   // 绑定失败启动期暴露
}
```

- **禁止决策**：禁止用 @Value 管理超过 3 个同组配置（配置治理成灾）。
- **验收指标**：配置中心改值 → 热更新生效（@Value 场景验证不生效、@ConfigurationProperties 场景验证生效）。
