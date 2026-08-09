# 🚀 SpringBoot 横切面与 AOP（系列 06）

> 本篇文章回答三个问题：
> 1. **为什么**：日志、鉴权、事务、性能监控这些横切逻辑散落在每个方法里，为什么继承/装饰器都救不了？AOP 抽象了什么？
> 2. **是什么**：`@Aspect` / `@Around` 到底怎么生效的？JDK 动态代理和 CGLIB 什么区别？为什么 Spring Boot 3 默认用 CGLIB？
> 3. **怎么失效**：为什么 05 篇的 `@Transactional` 只是 AOP 的一个实例？为什么没有 aspectjweaver 时自定义切面会静默失效？
>
> 前置：00 篇的四层创建链（BPP 是"安检关卡"，AOP 代理在这里贴标签）、03 篇的自动装配（AopAutoConfiguration 双分支只测了配置评估）、05 篇的事务代理（已经用了 AOP，本篇补上本体）。本篇主线角色：**一次方法调用穿越层层关卡**。

---

# ⚠️ 版本与证据边界

| 维度 | 本文承诺 |
| ---- | ---- |
| 代码实证 | `knowledge/springboot/experiments/code/` 下 demo13×8（aspect.ProxyKindApp / AdviceOrderApp / PointcutApp / AspectOrderApp / ProxyInternalsApp / UnwrapApp / VisibilityApp / TxVisibilityApp）+ demo17 自研权限切面 starter（permcheck + PermissionStarterApp），本机实测输出原样引用 |
| 实测环境 | macOS + JDK 21.0.11（Azul Zulu）+ spring-boot **3.3.5** + spring-aop **6.1.14** + aspectjweaver **1.9.22.1**（版本出自 spring-boot-dependencies-3.3.5.pom BOM） |
| 运行方式 | `cd knowledge/springboot/experiments/code && ./build.sh && ./run.sh demo13.aspect.XXX`（lib/ 43 个 jar；ProxyKindApp 三种跑法见实验复现节） |
| Specification | AspectJ 切点表达式语法（execution/within/@annotation 语义）、`@Transactional` 回滚与传播语义（05 篇）、Spring AOP 注解语义（@Aspect/@Before/@Around/@After/@Order）、AspectJ 绑定参数规则（@annotation(参数) 参数类型须与注解类型匹配） |
| Implementation | Boot 3.3.5 AopAutoConfiguration 双分支行为（实测：有 aspectjweaver → AnnotationAwareAspectJAutoProxyCreator 收集全部 Advisor；无 aspectjweaver → InfrastructureAdvisorAutoProxyCreator 只收 ROLE_INFRASTRUCTURE advisor，用户 Advisor 被过滤，javap 反编译 isEligibleAdvisorBean 实证）、Boot 3 默认 proxyTargetClass=true 走 CGLIB（实测：有 aspectjweaver 也 CGLIB）、JDK 动态代理类名 demo13.aspect.$Proxy76（实测）、CGLIB 类名 $$SpringCGLIB$$N（实测）、五种通知触发顺序（实测）、@Order 洋葱模型（实测）、代理字段注入只填目标实例（实测）、CGLIB 代理 toString 转发目标对象（实测）、ProxyFactory 无 advisor 也出代理（实测）、static/private/final 方法不可被代理覆写（VisibilityApp 实测：final 匹配=true 但拦不住、static/private 匹配=false）、@Transactional 在 protected/包可见方法上 CGLIB 外部调用生效（TxVisibilityApp 实测 + AbstractFallbackTransactionAttributeSource.allowPublicMethodsOnly()=false javap 实证）、自调用失效与四种修复（05 篇 SelfInvocationApp 实测回看：跨 bean / @Lazy 自注入 / context.getBean / @EnableAspectJAutoProxy(exposeProxy=true)+AopContext.currentProxy，含链外 IllegalStateException 实测） |
| 待验证 | 无 @Order 时多切面顺序的真实不确定性（机制推导，未做多轮实验统计）；AspectJ 编译期织入（LTW）的真实表现（本文只讲运行时代理路线，前置条件矩阵中 ajc/LTW 能力为 AspectJ 文档语义未实测）；不同 JDK 版本动态代理类名格式差异 |
| 跨版本对照 | `@Proxyable`（Framework 7.0.x 新能力：`@Proxyable(INTERFACES)` / `@Proxyable(TARGET_CLASS)` 按 Bean 指定代理策略，24 章 02 章 Boot 4.1 基线文档语义）——**3.3.5/6.1.14 无此注解**（本系列实测环境不适用，标注待验证）；全局默认与 Boot 配置仍按目标版本核对 |
| 未覆盖 | AspectJ 的 ajc 编译期织入/LTW；AspectJ 全部切点指示符（target/args/this 等未实测）；Spring Security 的 AOP 用法细节；缓存/异步 starter 的完整实现（只做机制识别；自研权限切面 starter 已实测，demo17——机制同构，验证的是"装配机关"） |

---

# 🏷️ 关键词

AOP | 横切关注点 | 切点 Pointcut | 通知 Advice | 切面 Aspect | @Before | @After | @AfterReturning | @AfterThrowing | @Around | execution | within | @annotation | JDK 动态代理 | CGLIB | proxyTargetClass | AnnotationAwareAspectJAutoProxyCreator | InfrastructureAdvisorAutoProxyCreator | Advisor | @Order | 洋葱模型 | @EnableAspectJAutoProxy | aspectjweaver | 自调用失效 | 解包

---

# 🗂️ 目录

- Level 1 为什么需要 AOP：横切关注点与三波失败方案
- Level 2 核心机制：运行时动态代理（JDK vs CGLIB 实测）
- Level 3 语法体系：切点表达式与五种通知（全实测）
- Level 4 装配机制：从注解到代理的完整链路（含代理边界：static/private/final 实测 + 自调用失效专节）
- Level 5 应用地图：Boot 生态里 AOP 的家族成员
- Level 6 生产实践：失效排查 / 切点过宽 / 顺序事故
- 全篇因果链总图
- 线上案例（3 个）
- 面试自查表
- 坑与细节（10 个）
- 版本勘误表
- 生产决策卡（3 张）
- 跨语言视角
- 系列索引

---

# 🏭 全文唯一比喻：关卡哨兵

一座大楼（应用）的每个走廊（方法）都要经过**哨兵关卡**（切面）。关卡的职责：

- **切点（Pointcut）**：哨兵只查"符合特征的旅客"——白名单规则（表达式）。
- **通知（Advice）**：检查动作本身——"放行前查证件"（@Before）、"放行后记录"（@After）、"通关全流程把关"（@Around）。
- **切面（Aspect）**：一个关卡 = 规则 + 动作打包。

旅客（一次方法调用）从大门进入，穿过**一层层关卡**（多切面），到达目的地（目标方法），再原路返回。**楼层管理员（容器）在旅客进楼时就把关卡布置在走廊上**——这就是代理。

全文主线角色：**一次方法调用穿越层层关卡**。

---

# Level 1 为什么需要 AOP：横切关注点与三波失败方案

## 问题：横切关注点

要在一个系统里给所有写操作加"操作日志 + 权限校验 + 性能计时"，直觉方案：

```java
public void transfer(Account from, Account to, long amount) {
    long start = System.currentTimeMillis();
    checkPermission(currentUser, "transfer");   // 鉴权
    try {
        // 业务
    } finally {
        auditLog("transfer", ...);              // 审计
        perfLog(start);                          // 性能
    }
}
```

每加一个方法，这段样板复制一遍。三样东西混在一起：

- **核心关注点**：转账逻辑本身。
- **横切关注点**：鉴权/审计/计时——**横跨所有业务方法**，与业务无关。

横切关注点的本质矛盾：**它属于"每个方法"，但不属于"任何一个方法"**。

## 三波失败方案

### 1. 继承（模板方法）

`BaseService` 里写样板，子类调用：

```
BaseService.checkAndCall(method)
  ├─ BaseTransferService / BaseOrderService ...
  └─ 新横切点（如加"限流"）→ 改基类 → 影响所有子类
```

**死因**：横切点会持续增加（鉴权、审计、计时、限流、重试……），基类变成"万金油"——**一个正交的横切点混进继承链的每一个方法**；且"父类方法"与"业务方法"仍然耦合在一起。

### 2. 装饰器（静态代理）

```
PermissionDecorator(service) → AuditDecorator(permissionService) → 业务
```

**死因**：装饰器要**逐个方法手工转发**（接口有几个方法就要写几个转发）；加一个横切点 = 再包一层 = 全部调用点要改；**装饰器链的顺序硬编码在组装代码里**。

### 3. 过滤器（按请求拦截）

Servlet Filter / Interceptor 能拦"请求"，但拦不住"方法调用"——**粒度不对**：一次请求里可能调 3 次写操作，只想拦第 2 次。

## AOP 的抽象：把"拦截"提升为一等公民

```
切点（哪里的方法） + 通知（做什么）+ 切面（打包）
```

- 横切逻辑**只写一次**，放在切面里。
- 拦截**按方法签名/注解/类型**声明（切点表达式），不写转发代码。
- 业务方法**完全不知道**自己被拦截（没有继承、没有装饰器调用点）。

> **因果链**：横切点散落 + 三波方案各自死因（继承耦合、装饰器手工转发、过滤器粒度错）→ 需要"声明式拦截" → AOP：切点 + 通知 + 代理。

**代价（Trade-off）**：拦截是**运行时透明的**——业务方法不知道有拦截 → 失效时也是静默的（05 篇自调用失效、本篇 Level 4 的 role 过滤都是这类）。

---

# Level 2 核心机制：运行时动态代理（JDK vs CGLIB 实测）

## 为什么 Spring AOP 用"运行时代理"而不是"编译期织入"

实现"方法被拦截"有两条路线：

| 路线 | 机制 | 代表 | 特点 |
| ---- | ---- | ---- | ---- |
| 编译期/加载期织入 | 改字节码，把通知织进方法体 | AspectJ ajc / LTW | 性能好、可织 final/private；部署复杂（编译器/代理器） |
| 运行时代理 | 创建目标对象的**代理对象**，调用走代理 | Spring AOP | 零编译依赖、纯 jar 部署；只能代理"可重写的调用" |

Spring 选择运行时代理：**与容器（IoC）天然契合**——Bean 本身就是容器创建的，容器顺手换成代理对象即可（00 篇创建链：BPP 在 postProcessAfterInitialization 包装）。

> 注意：Spring 的 `@Aspect/@Before` 等**注解语法是借用 AspectJ 的**（注解类型由 aspectjweaver 提供，不是 spring-aop）——**语法借用 ≠ 织入机制**。实测证据见 Level 4 的 ClassNotFoundException。

## JDK 动态代理 vs CGLIB（实测）

两种代理实现：

| 维度 | JDK 动态代理 | CGLIB |
| ---- | ---- | ---- |
| 前提 | 目标**必须有接口** | 无接口也可（子类化目标类） |
| 机制 | 反射 + Proxy 生成接口实现类 | 字节码生成目标类的**子类** |
| 类名（实测） | `demo13.aspect.$Proxy76` | `...$GreetingImpl$$SpringCGLIB$$0` |
| 局限 | 只能代理接口方法 | 不能代理 final 类/final 方法 |

> 类名里的数字编号（$Proxy76 / $$SpringCGLIB$$0）是每次运行时生成的序号，**同一代码每次运行编号不同**（实测中多次出现 $Proxy76/$Proxy94/$Proxy95）——重要的是**类名模式**（`$Proxy` 前缀 vs `$$SpringCGLIB$$` 后缀），不是具体编号。

**代理选择规则（DefaultAopProxyFactory 语义）**：目标有接口且未强制 `proxyTargetClass` → JDK；否则 CGLIB。

## 实测一：Boot 3 默认 CGLIB（demo13.aspect.ProxyKindApp 跑法 1/2）

```
跑法 1（全 lib，默认配置）：
[有接口 GreetingImpl] 类=demo13.aspect.ProxyKindApp$GreetingImpl$$SpringCGLIB$$0；AOP 代理=true
[无接口 PlainService] 类=demo13.aspect.ProxyKindApp$PlainService$$SpringCGLIB$$0；AOP 代理=true
```
```
跑法 2（-Dapp.proxyTargetClass=false，显式关掉 CGLIB）：
[有接口 GreetingImpl] 类=demo13.aspect.$Proxy76；AOP 代理=true
[无接口 PlainService] 类=demo13.aspect.ProxyKindApp$PlainService$$SpringCGLIB$$0；AOP 代理=true
```

两个关键事实：

1. **Boot 3.x 默认 `spring.aop.proxy-target-class=true` → 有接口也走 CGLIB**（`$$SpringCGLIB$$0`）——即使 aspectjweaver 在 classpath。Boot 2.x 默认 JDK 代理、3.x 改默认 CGLIB——**语义变化**（影响：接口方法之外，类自身方法也被代理）。
2. **关掉 CGLIB 后：有接口 → JDK 动态代理（`$Proxy76`）；无接口 → 仍然 CGLIB**（JDK 代理无法代理无接口类，Spring 自动回退）。

## 实测二（重大发现）：无 aspectjweaver 时自定义切面静默失效

03 篇实测过 AopAutoConfiguration 双分支的**配置评估**：有 aspectjweaver → $AspectJAutoProxyingConfiguration Positive；无 → $ClassProxyingConfiguration Positive（CGLIB）。本篇补上**行为差异**（ProxyKindApp 跑法 3，去 aspectjweaver）：

```
跑法 3（去 aspectjweaver；用户自定义 Advisor 存在，canApply=true）：
[诊断] creator 实际类型 = InfrastructureAdvisorAutoProxyCreator
[诊断] findCandidateAdvisors 数量=1（只有事务 advisor；pingAdvisor 被过滤）
[有接口 GreetingImpl] AOP 代理=false
[无接口 PlainService] AOP 代理=false
```

**有 advisor、canApply=true，但 bean 不被代理**！反编译定位（javap，Spring 6.1）：

```
InfrastructureAdvisorAutoProxyCreator.isEligibleAdvisorBean(beanName)
  = beanFactory.getBeanDefinition(beanName).getRole() == ROLE_INFRASTRUCTURE(2)
```

两个 creator 的差异：

| 有 aspectjweaver（跑法 1） | 无 aspectjweaver（跑法 3） |
| ---- | ---- |
| `AnnotationAwareAspectJAutoProxyCreator` | `InfrastructureAdvisorAutoProxyCreator` |
| 收集：**所有 Advisor bean + 所有 @Aspect**（实测 findCandidateAdvisors=20，含 demo13 包内全部切面） | 收集：**只收 role=ROLE_INFRASTRUCTURE 的 Advisor**（实测=1） |
| 用户切面/Advisor 生效 ✓ | 用户切面/Advisor **被过滤** ✗ |

**因果链**：

```
无 aspectjweaver
  → AopAutoConfiguration 走 ClassProxyingConfiguration
  → 注册 InfrastructureAdvisorAutoProxyCreator（只服务框架内部 advisor）
  → 用户自定义 Advisor/@Aspect 全部不生效
  → 静默：没有报错，只是不代理
```

**生产含义**：这就是为什么 Boot 的 `spring-boot-starter-aop` 必须带 aspectjweaver——**它不是可选项，是用户切面的生效前提**。只引 spring-aop 而没引 aspectjweaver 的自定义切面，全部静默失效（事务/缓存等框架自身功能正常——它们是 role=2 的基础设施 advisor）。

---

# Level 3 语法体系：切点表达式与五种通知（全实测）

## 切点表达式：三种最常用指示符（实测命中矩阵）

demo13.aspect.PointcutApp：三个服务方法 × 三个切点，实测命中矩阵：

```
                    ServiceA.methodA()  ServiceA.methodB()  ServiceB.methodC()
execution(* ...     ●（方法名前缀）      ●                   ○
  method*(..))
@annotation(Marked)  ●（方法带注解）      ○                   ○
within(...ServiceB)  ○                   ○                   ●
```

真实输出（JDK 21.0.11 + spring-boot 3.3.5，本机）：

```
--- 调用 methodA（带 @Marked） ---
  [切点2 @annotation: 命中]
  [切点1 execution: 命中]
  [ServiceA.methodA]
--- 调用 methodB（无注解，method 前缀命中） ---
  [切点1 execution: 命中]
  [ServiceA.methodB]
--- 调用 methodC（within 命中） ---
  [切点3 within: 命中]
  [ServiceB.methodC]
```

三种指示符语义（Specification）：

| 指示符 | 匹配维度 | 典型写法 |
| ---- | ---- | ---- |
| `execution` | 方法签名（修饰符/返回类型/类名.方法名/参数） | `execution(* com.x.Service.do*(..))` |
| `within` | 类型（类/包，含子类） | `within(com.x..*)` |
| `@annotation` | 方法上的注解 | `@annotation(MyLog)` |

`*` 通配任意（一个位置）、`..` 通配任意参数（多层）。

**补充：`this` vs `target` 与"切点过宽"**（24 章 02 章对照，Boot 4.1 基线文档语义）：

```text
T → Proxy（this 指向代理对象）→ Target（target 指向目标对象）
```

- **`this` 匹配代理对象的类型，`target` 匹配目标对象的类型**——目标对象实现了接口而代理类是 JDK Proxy 时，两者结果不同（本机未实测该差异，标注待验证）；
- **切点过宽的代价**（P-A03，文档语义）：`execution(* *(..))` 会把基础设施方法、健康检查、代理辅助方法全部纳入——日志噪声只是小问题，**事务/重试/递归语义被改变、资源放大**才是危险；修正：包限定 + 公开方法 + 显式注解，让切点有业务边界（06 篇装配链一节同一纪律）。

## 五种通知：正常/异常双路径（实测触发顺序）

demo13.aspect.AdviceOrderApp 真实输出：

```
=== 正常路径 ===
[Around] 开始
[Before] 目标方法执行前
[目标] bizWork 执行, 参数=ok
[AfterReturning] 目标方法正常返回
[After] 最终通知（finally 语义）
[Around] 结束
=== 异常路径 ===
[Around] 开始
[Before] 目标方法执行前
[目标] bizWork 执行, 参数=bad
[AfterThrowing] 目标方法抛异常
[After] 最终通知（finally 语义）
[main] 捕获到: IllegalStateException
```

顺序语义（两条路径对照）：

| 通知 | 时机 | 正常路径 | 异常路径 |
| ---- | ---- | ---- | ---- |
| @Around | 包裹全部（proceed 前/后） | 开始→结束 | 开始→（无结束，异常向外传播） |
| @Before | 目标方法前 | ✓ | ✓ |
| 目标方法 | 业务本体 | ✓ | 抛异常 |
| @AfterReturning | 正常返回后 | ✓ | ✗（互斥） |
| @AfterThrowing | 异常抛出后 | ✗（互斥） | ✓ |
| @After | 最终（finally 语义） | ✓ | ✓ |

**生产关键**：@After 是 finally 语义（两条路径都执行）；@AfterReturning/@AfterThrowing 互斥；**@Around 不调 `proceed()` 等于把目标方法整个吞掉**（可用于熔断/短路）。

## 多切面：@Order 与洋葱模型（实测）

demo13.aspect.AspectOrderApp 真实输出（两个切面命中同一方法）：

```
[AspectA(Order1) Around] 开始
[AspectA(Order1) Before]
[AspectB(Order2) Around] 开始
[AspectB(Order2) Before]
[目标] bizWork 执行
[AspectB(Order2) After]
[AspectB(Order2) Around] 结束
[AspectA(Order1) After]
[AspectA(Order1) Around] 结束
```

**洋葱模型**：@Order 值小的在外层。进入顺序 A→B（值升序），**退出顺序 B→A（后进先出）**——与函数调用栈同构。

**生产含义**：无 @Order 时多切面顺序不可依赖（bean 解析顺序决定，机制推导未做多轮实验统计，待验证）。

**重试切面的副作用：proceed() 多次执行**（24 章 02 章对照，Boot 4.1 基线文档语义）：

```text
Around 重试逻辑：for (尝试) { proceed() }   ← 每次 proceed 都是完整执行一次目标+内层切面
```

- **每次 proceed() 都会重放整个内层洋葱**（内层切面、目标方法全部重新执行）——重试的副作用面 = 内层所有切面的副作用面之和；
- **与事务组合的危险**（02 章文档语义）：重试可能发生在同一个外层事务里——第一次失败的 rollback-only 标记、锁、连接占用会影响后续尝试；重试必须绑定：幂等键、可重试异常白名单、总时间预算；
- 结论：重试切面要放在洋葱最外层（先于事务判定失败），且"可重试异常"要明确（与 05 篇回滚规则同源的设计问题）。

---

# Level 4 装配机制：从注解到代理的完整链路

## 装配链全景

```
@SpringBootApplication（含组件扫描 + 自动配置）
  └─ AopAutoConfiguration（03 篇条件链，双分支）
      └─ @EnableAspectJAutoProxy（Boot 自动开，无需手写）
          └─ AnnotationAwareAspectJAutoProxyCreator
             （BeanPostProcessor！00 篇"安检关卡"本体）
              └─ 每个 bean 实例化后 postProcessAfterInitialization：
                  1. findCandidateAdvisors（收集 Advisor bean + @Aspect）
                  2. canApply 逐个匹配（切点 vs 目标类方法）
                  3. 命中 → ProxyFactory 创建代理 → 替换容器里的 bean
```

实证（demo13.aspect.ProxyInternalsApp）：

```
[诊断] autoProxyCreator 是 BeanPostProcessor: true
[手动 ProxyFactory 无 advisor] 类=demo13.aspect.$Proxy94；isAopProxy=true；advisor 数=0
[自动代理 GreetingImpl] 类=...$GreetingImpl$$SpringCGLIB$$0
[自动代理] advisor 数=3（LogAspect 的 2 个 + 事务 advisor 1 个）
```

三个事实：

1. **creator 是 BeanPostProcessor**——00 篇的"安检关卡"落点：每个 bean 过站时被检查（有 advisor 命中才换代理）。
2. **ProxyFactory 手动创建：无 advisor 也出代理**（`$Proxy94`）——与自动 creator 不同（自动 creator 无 advisor 时不代理，跑法 3 实测）——**"代理是否创建"在两条路上语义不同**：手动 = 总是代理；自动 = 有 advisor 才代理。
3. **自动代理的 advisor 链可查**（`((Advised) proxy).getAdvisors()`）——代理对象实现 `Advised` 接口。

## 代理对象 vs 目标对象（实测：05 篇 NPE 的机制解释）

demo13.aspect.UnwrapApp 真实输出：

```
[容器 bean] 类=...UserService$$SpringCGLIB$$1（isAopProxy=true）
[容器 bean 的 self 字段] null ← 代理实例上字段未注入
[AopUtils.getTargetClass] 类=demo13.aspect.UnwrapApp$UserService
[解包 target] 类=demo13.aspect.UnwrapApp$UserService
[解包 target 的 self 字段类] demo13.aspect.UnwrapApp$UserService$$SpringCGLIB$$0；isAopProxy=true
[容器 bean instanceof UserService] true
[容器 bean toString] demo13.aspect.UnwrapApp$UserService@7d088813（代理转发给目标 toString）
```

完整闭环 05 篇 Level 4 的 NPE：

- **字段注入发生在目标实例上**，代理实例的注入字段为 null（05 篇 `svc.self` NPE 根因）。
- **解包**：`((Advised) proxy).getTargetSource().getTarget()` 拿到原始目标——目标实例内部 `self` 已被注入（@Lazy 代理，isAopProxy=true）。
- `instanceof` 为 true：CGLIB 是子类，类型关系保持。
- **toString 被代理转发给目标对象**（DynamicAdvisedInterceptor 保持 toString 语义）——所以"打印代理对象"看到的是目标类名，`getClass()` 才是代理类名。
- **返回自身被换成代理**（05 篇关键事实 4，6.1.14 字节码实证）：方法 `return this` 时，`CglibAopProxy.processReturnType` 检测到返回值 `== target` 会替换为 proxy（除非实现 `RawTargetAccess`）——"返回 this"的方法调用方拿到的仍是代理，AOP 能力不丢。

**强转陷阱**（24 章 02 章 P-A06，Boot 4.1 基线文档语义）：JDK 代理模式下 `(OrderService) proxy` **抛 ClassCastException**——JDK Proxy 只实现接口，不是实现类的子类。修正：一律依赖接口（`OrderService` 接口）而不是实现类。CGLIB 模式 `instanceof` 为 true（上文实测），所以"接口注入 + 强转实现类"只在切到 JDK 代理时爆炸——排查 ClassCastException 时先确认代理模式。

## 代理的边界：哪些方法拦不住（static / private / final / 可见性实测）

Spring AOP 是**运行时代理**，拦截能力 = "代理能覆写目标方法"的能力。`demo13.aspect.VisibilityApp` 用 `within()` 类级切点 + execution 精确切点实测六种方法（JDK 21.0.11 + spring-boot 3.3.5 + CGLIB）：

```
[bean 类] demo13.aspect.VisibilityApp$TargetService$$SpringCGLIB$$0
--- 调用 doPublic（外部经代理） ---      [切面拦截] doPublic     ✅
--- 调用 doProtected（外部经代理） ---   [切面拦截] doProtected  ✅
--- 调用 doPackage（外部经代理） ---     [切面拦截] doPackage    ✅
--- 调用 doFinal（final 方法） ---       （无拦截）             ❌
--- 调用 doStatic（静态方法） ---        （无拦截）             ❌
--- 调用 caller（内部调 private/final）---[切面拦截] caller      ✅（只拦外层）
```

命中矩阵（真实输出）：

| 方法 | 外部经代理调用 | 切面拦截？ | 原因 |
| ---- | ---- | ---- | ---- |
| public | ✅ | ✅ | CGLIB 子类可覆写 |
| protected | ✅（同包可调） | ✅ | CGLIB 子类可覆写 |
| package-private | ✅（同包可调） | ✅ | CGLIB 子类可覆写 |
| final | ✅ | ❌ | **final 不可覆写**（CGLIB 子类化失效） |
| static | ❌（类名直调） | ❌ | **静态方法绑定类**，调用不经实例代理 |
| private | ❌（类外不可见） | ❌ | **不可见 + 不可覆写**；类内部 `this` 调用也不经过代理 |

### 关键区分：表达式"匹配" ≠ 代理"拦截"（且两者分修饰符不同）

很多人以为"切点表达式命中了就会拦截"。实测打脸（同一 App，补充验证）：

```
[验证] matches(doFinal)=true                          → final：表达式匹配成功，运行时拦不住
[验证] matches(doStatic)=false                        → static：表达式层面就不匹配
[验证] matches(doPrivate, 表达式写 private)=false     → private：表达式层面就不匹配
--- 调用 doFinal（final 方法） ---
[目标] doFinal          ← 无任何拦截！execution 精确切点也不生效
```

三种修饰符的"匹配/拦截"组合完全不同：

| 方法 | 表达式匹配（Spring 语境） | 运行时拦截 | 组合类型 |
| ---- | ---- | ---- | ---- |
| final | ✅ | ❌ | **匹配 ≠ 拦截** |
| static | ❌ | ❌ | 根本不匹配，更无从拦截 |
| private | ❌（即使显式写 `private` 修饰符） | ❌ | 根本不匹配，更无从拦截 |

**为什么 Spring AOP 语境下 static/private 连表达式都不匹配**：Spring 的 `AspectJExpressionPointcut` 底层用 aspectjweaver 的匹配器，但服务于"运行时代理"——对代理无法覆写的方法（static/private），匹配器直接返回不匹配。**匹配与拦截被绑定在"代理能力"内**（与 AspectJ 编译期织入的语义不同，见下面前置条件矩阵）。

### 前置条件矩阵：private 到底能不能切（三条技术路线）

"private 能不能代理"没有统一答案，取决于**技术路线**及其**前置条件**：

| 路线 | 机制 | private | final | static | 前置条件 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| Spring AOP + JDK 动态代理 | 代理实现接口 | ❌ 接口无 private 业务方法 | ❌ | ❌ | 目标必须实现接口；`proxyTargetClass=false` |
| Spring AOP + CGLIB | 子类覆写 | ❌ 子类不可见 | ❌ 不可覆写 | ❌ 绑定类 | 非 final 类；方法非 final、可覆写（public/protected/包可见） |
| **AspectJ 编译期织入（ajc）** | **直接改写字节码，织入方法体** | ✅ 可织 | ✅ 可织 | ✅ 可织（构造器/字段也可） | **编译期用 ajc**（AspectJ maven 插件）或**加载期 LTW**（javaagent + `META-INF/aop.xml`）；切面须为 AspectJ 织入形态（能力为 AspectJ 文档语义，**未实测**，见版本边界待验证） |
| AspectJ 加载期织入（LTW） | javaagent 类加载时织入 | ✅ 可织 | ✅ 可织 | ✅ 可织 | `-javaagent:spring-instrument` + aop.xml 配置 |

**为什么 ajc 能织 private/final/static**：织入不依赖"覆写"——ajc 把通知代码**直接写进目标方法体**（方法执行时即织入生效），可见性与覆写都不构成限制。这就是"运行时代理 vs 编译期织入"的本质差异（Level 2 表格的实战注脚）。

**Spring 生态里的织入路线**：`spring-aspects` 模块提供 `@Transactional` 的 AspectJ 织入版（`AnnotationTransactionAspect`）——用 ajc/LTW 织入后，**private 方法上的事务注解也能生效、自调用也能生效**（织入在方法体，不依赖代理）——代价是构建/启动配置复杂（ajc 编译或 javaagent），这也是绝大多数项目选运行时代理的现实原因。

### 生产含义

先厘清一个常见误解：**自定义切面 与 框架注解（@Transactional/@Cacheable）对可见性的要求不同**。

**自定义 @Aspect 切面**（实测，VisibilityApp）：public / protected / 包可见 方法都能拦（CGLIB 可覆写，within/execution 表达式命中即拦截）——private / static / final 拦不住（final 匹配≠拦截，static/private 连匹配都不通过）。

**框架注解 @Transactional**（实测，TxVisibilityApp，6.1.14 + CGLIB + 外部经代理调用）：

```
[public 事务方法] 抛异常后 count=0（回滚 → 事务生效）
[protected 事务方法] 抛异常后 count=0（回滚 → 事务生效！）
[包可见事务方法] 抛异常后 count=0（回滚 → 事务生效！）
```

**protected / 包可见方法上的 @Transactional 也生效**——两个条件缺一不可（javap 实证）：

1. 注解读取不限制可见性：`AbstractFallbackTransactionAttributeSource.allowPublicMethodsOnly()` 返回 **false**（javap 反编译 spring-tx 6.1.14：`iconst_0`），非 public 方法的注解照样被读取。
2. CGLIB 代理能覆写 protected/包可见方法（VisibilityApp 实测）。

那为什么官方文档建议"只标 public"？**因为生效是有前置条件的**：

- **JDK 动态代理模式**（Boot 2.x 默认 / `proxyTargetClass=false`）：代理基于接口，接口方法都是 public——protected/包可见方法**根本不在代理能力内** → 注解不生效。CGLIB 模式下才可能生效。
- **private / static / final**：任何模式、任何调用方式都不生效（private 不可见不可覆写、static 绑定类、final 不可覆写）。
- **自调用（this.xxx()）**：任何可见性都失效（下节专讲）。
- **依赖"外部经代理调用"**：main 里直接调 protected 方法走的是代理（同包可见）→ 生效；若某处绕过代理直接调目标对象 → 失效。

**生产结论**：

- `@Transactional` / `@Cacheable` 标在 private / static / final 方法上 = **完全无效**（不报错，静默）。
- protected / 包可见方法：CGLIB 模式（Boot 3 默认）外部调用可生效，但**依赖代理模式与调用路径**——文档建议 public 是为了跨模式、跨版本行为一致（Boot 2.x 默认 JDK 代理下 protected 就不生效）。
- 设计"可切面化"的服务类：方法 public + 非 final；类非 final。

（版本注意：以上"protected 生效"是 **6.1.14 + CGLIB 实测**；不同版本/模式的差异见版本勘误表。）

## 自调用失效（Self Invocation）：代理拦截的盲区

### 本质：拦截点只在"从代理对象发出的调用"

代理是**包装**：调用方手里的引用是代理，代理内部的 `this` 是目标对象。

```
调用方 ──► 代理（事务拦截器在这里）──► 目标对象
                ▲
                └ 目标对象内部的 this.xxx() 调用直接落在目标对象上，
                  不经过代理 —— 拦截器根本没有执行机会
```

```
public void selfCall() {
    this.transactionalSave();   // 自调用：绕过代理 → @Transactional 静默失效
}
```

05 篇实测证据（SelfInvocationApp，真实输出）：

```
[打点] transactionalSaveAndThrow 事务激活=false   ← this 自调用：事务没开！
[打点] transactionalSaveAndThrow 事务激活=true    ← self 代理调用：事务开了！
```

### 为什么是静默的

方法照样执行、数据照样写入——只是少了拦截器环绕。**没有任何异常**，只能靠"回滚是否发生"这类行为断言发现（05 篇 count 断言法）。

### 影响面：不止事务

所有注解型能力在自调用下全部失效：`@Transactional` / `@Cacheable` / `@Async` / 自定义 `@Aspect` 切面——**同一机制，同一盲区**。

### 修复方案全景（含前置条件）

| 方案 | 做法 | 前置条件 | 状态 |
| ---- | ---- | ---- | ---- |
| @Lazy 自注入 | `@Autowired @Lazy private UserService self;` 然后 `self.xxx()` | Boot 2.6+ 循环依赖默认禁止（官方 release notes 行为），`@Autowired` 普通自引用会报错；**@Lazy 是标准解法**（推迟解析绕过循环依赖） | 05 篇实测（激活 false→true） |
| 拆 bean 跨类调用 | 把事务方法放另一个 Service，注入调用 | 无 | 05 篇 PropagationApp 实测（REQUIRES_NEW 跨 bean 才生效） |
| TransactionTemplate | 编程式事务 `new TransactionTemplate(txManager).execute(...)` | 需要手动管理事务回调；放弃声明式 | 未实测（标注） |
| AspectJ 织入（spring-aspects） | 用 ajc/LTW 织入 `AnnotationTransactionAspect` | ajc 编译或 javaagent 构建配置；自调用/private 全部生效（织入方法体） | 机制识别（Level 4 前置条件矩阵） |

### 为什么 Spring 不自动解决自调用

自调用发生在**目标对象内部**——Spring 的代理在**对象外部**，容器无法感知"目标对象内部的 this 调用"（没有运行时能挂钩的内部调用点）。这是运行时代理的根本限制：**拦截能力边界 = 代理可见的调用边界**。要消除这个盲区只有两条路：不在对象内自调（方案 1/2）或让拦截发生在方法体内部（方案 4 织入）。

### 面试口径

"Spring 事务为什么同类内 this 调用不生效？怎么修？"——答：代理只拦外部调用；this 直达目标对象；修法 = @Lazy 自注入 / 拆 bean / 编程式事务 / AspectJ 织入；并指出这是运行时代理与编译期织入的本质差异。

## 与 05 篇的衔接：@Transactional 只是 AOP 的一个实例

05 篇的所有实测（自调用失效、@Lazy 修复、count 断言）在 AOP 视角下是同一个机制：

```
@Transactional = 一个"切点命中所有 @Transactional 方法"的切面
  └─ TransactionInterceptor（@Around 语义：开事务 → proceed → 提交/回滚）
      └─ 自调用失效 = 调用绕过代理 = 拦截器根本没机会跑
```

**事务/缓存/异步/重试——全是"通知 + 切点"的实例**，这就是 Level 5 的应用地图。

---

# Level 5 应用地图：Boot 生态里 AOP 的家族成员

## 一个机制，全家使用

| 能力 | 切点 | 通知（拦截器） | 状态 |
| ---- | ---- | ---- | ---- |
| 事务（05 篇实测） | @Transactional 注解 | TransactionInterceptor | 本篇回看 ✓ |
| 缓存 | @Cacheable/@CacheEvict | CacheInterceptor | 机制识别（配置类见下） |
| 异步 | @Async | AsyncExecutionInterceptor | 机制识别 |
| 定时 | @Scheduled | ScheduledAnnotationBeanPostProcessor | 机制识别 |
| 重试（spring-retry） | @Retryable | RetryOperationsInterceptor | 机制识别 |
| 安全 | @PreAuthorize 等 | 方法安全拦截器 | 机制识别 |

**识别方法**（不实测每个 starter 的完整行为，标注机制识别）：所有注解型能力 = `@EnableXxx + 一个 BPP/creator + 一个 Advisor（role=2 或切点扫描注解）`。与事务同构。

## 关键差异：role 与优先级

- 框架自带能力（事务/缓存）的 advisor 大多注册为 **ROLE_INFRASTRUCTURE**——这也是 Level 2 跑法 3 里"只有事务 advisor 生效"的原因。
- 用户切面（@Aspect）由 aspectjweaver 路线提供（Level 2 跑法 1）。

## 新增横切能力的推荐姿势

```java
@Aspect
@Component
public class RateLimitAspect {
    @Around("@annotation(rateLimit)")   // 注解驱动切点
    public Object around(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable { ... }
}
```

**为什么注解驱动最稳**：`@annotation` 切点把"要拦截谁"显式化（谁加注解谁被拦），比 execution 按包名/方法名匹配更可控、可查、可迁移。

---

# Level 6 生产实践：失效排查 / 切点过宽 / 顺序事故

## 1. 失效排查清单（遇到"切面没生效"按此走）

```
切面没生效？
├─ 1. 切面类是 @Component 吗？（@Aspect 不是组件注解，本文开发期踩坑）
├─ 2. classpath 有 aspectjweaver 吗？（没有 → Infrastructure 分支过滤用户切面，Level 2 实测）
├─ 3. 调用真的经过代理吗？（this 自调用 → 绕过，05 篇实测）
├─ 4. 切点表达式命中吗？（命中矩阵实测法：每个切点配一个 @Before 打点）
├─ 5. 表达式解析失败？（@annotation 绑定参数类型错误 → 该通知静默失效，Level 3 实测）
├─ 6. 目标方法是 public 吗？（static / private / final 方法不可被代理覆写——
│   表达式匹配成功也拦不住，Level 4 实测：AspectJExpressionPointcut.matches(doFinal)=true
│   但调用无拦截）
└─ 7. 方法所在类/方法不是 final 吗？（final 类不能 CGLIB 子类化，Level 4 实测）
```

> 第 6/7 条最容易静默：`@Transactional` / `@Cacheable` 标在 private/static/final 方法上**完全无效且不报错**——运行时一切正常，只是没有事务/缓存。排查口诀：**先问"这个方法能被代理覆写吗"**。

## 2. 切点过宽：误伤与失血

- `execution(* com.x..*.*(..))` 全包通配：连框架内部对象都可能命中（如配置类代理、事务管理器）→ **性能损耗 + 意外行为**。
- 生产铁律：**切点最小化**——能 `@annotation` 就别 `execution` 通配；表达式里显式排除框架包（`&& !within(org.springframework..*)`）。

## 3. @Order 事故：横切逻辑顺序敏感

鉴权与审计的典型顺序错误：审计先记、鉴权后查 → 未授权请求也进了审计（或鉴权抛异常审计丢了）。**依赖切面顺序 = 依赖 @Order 纪律**：

- 外层（值小）：鉴权/限流（失败即短路，内层不跑）。
- 内层（值大）：事务（包裹最短）、业务日志。

## 4. 通知内吞异常 / 抛异常

- @Around 里 catch 了业务异常不往外抛 = **业务方看不到失败**（静默吞错，与 catch 在事务方法内同类，05 篇坑 5）。
- @After/@AfterReturning 里抛异常会**覆盖业务结果**（或让 AfterReturning 变异常路径）——横切逻辑要"弱失败"（记录不抛）。

## 5. 代理与序列化/反射

- 代理对象序列化/深拷贝会带上代理结构（CGLIB 子类字段）；`getClass()` 不是目标类——**需要目标类时用 AopUtils.getTargetClass**（Level 4 实测）。

## 6. 测试方法论

- 切面行为测试：对**代理 bean** 调方法断言副作用（与 05 篇 count 断言同法）。
- 验证代理：`AopUtils.isAopProxy(bean)`（本文所有 demo 的输出基石）。

---

# 全篇因果链总图

```
横切关注点散落（鉴权/日志/计时）
  → 继承/装饰器/过滤器三波失败（耦合/手工转发/粒度错）
      → AOP 抽象：切点 + 通知 + 切面（Level 1）
          → 运行时代理实现：JDK vs CGLIB（Level 2：默认 CGLIB 实测）
              → 语法体系：表达式 + 五种通知 + 顺序（Level 3：全实测）
                  → 装配链：AopAutoConfiguration → creator(BPP) → Advisor → 代理（Level 4）
                      → 框架家族：事务/缓存/异步/重试（Level 5）
                          → 生产：失效排查/切点最小化/顺序纪律（Level 6）
```

---

# 线上案例（3 个）

## 案例 1：加了 aspectjweaver 依赖切面还是不生效

症状：自定义 @Aspect 日志切面，依赖也加了（spring-aop），运行无输出。

排查：**spring-aop ≠ aspectjweaver**——`@Aspect` 注解类型来自 aspectjweaver。只引 spring-aop（无 aspectjweaver）时走 InfrastructureAdvisorAutoProxyCreator（Level 2 跑法 3 实测）——**用户切面被 role 过滤，静默不代理**。没有任何报错，只有"没输出"。

结论：**用户切面必须引 aspectjweaver（用 starter-aop）**；本文跑法 3 就是这个案例的复现。

## 案例 2：方法上明明加了 @Marked 注解，切点不命中

症状：`@annotation(marked)` 的通知不触发，其他通知正常。

真相：**绑定参数类型用了 `java.lang.annotation.Annotation` 基类**（本文 PointcutApp 开发期真实踩坑）——AspectJ 绑定要求参数类型与注解类型匹配，基类绑定导致该通知解析失败**静默失效**。改为绑定具体注解类型 `Marked` → 生效。

结论：@annotation 绑定参数用**具体注解类型**；表达式解析失败不会报错，用"每切点一个打点"验证命中矩阵。

## 案例 3：审计日志打印的是代理对象，字段全空

症状：日志里打了业务对象，字段全 null/格式不对。

真相：容器里拿到的是代理（Level 4 实测：**代理实例的注入字段为 null**，字段注入发生在目标实例）；toString 虽被转发给目标（Level 4 实测），但**直接反射/序列化代理对象**拿到的字段就是空的。

结论：业务方法里拿"自己"用 `this` 外的注入引用（代理）；反射目标类字段前先 `AopUtils.getTargetClass` 或解包 `getTargetSource().getTarget()`。

---

# 面试自查表

| 问题 | 答案要点 | 证据 |
| ---- | ---- | ---- |
| Spring AOP 的实现机制？ | 运行时代理（JDK 动态代理/CGLIB），非编译期织入；语法借用 AspectJ | Level 2 |
| JDK 代理与 CGLIB 区别？ | 接口 vs 子类；类名 $Proxy vs $$SpringCGLIB$$；final 限制 | 跑法 2 实测 |
| Boot 3 默认用哪个？ | CGLIB（proxyTargetClass=true 默认），有 aspectjweaver 也 CGLIB | 跑法 1 实测 |
| 五种通知顺序？ | Around→Before→目标→（Returning/Throwing 互斥）→After→Around 尾 | AdviceOrderApp 实测 |
| @After 与 @AfterReturning 区别？ | finally 语义 vs 仅正常路径 | 双路径实测 |
| 多切面顺序？ | @Order 值小在外层，洋葱模型，退出逆序 | AspectOrderApp 实测 |
| 为什么切面不生效？ | 排查清单 6 条（组件注解/weaver/自调用/表达式/绑定/可见性） | Level 6.1 |
| 为什么用户切面必须有 aspectjweaver？ | 无它 → Infrastructure creator 只收 role=2 advisor | 跑法 3 + javap 实测 |
| 代理与目标对象的字段差异？ | 注入只填目标实例；代理字段 null；getTargetClass 取目标类 | UnwrapApp 实测 |
| @Transactional 与 AOP 的关系？ | 事务 = 注解切点 + TransactionInterceptor 通知 | 05 篇 + Level 4 衔接 |
| private/static/final 方法能切吗？ | Spring AOP 不能（final 匹配≠拦截，static/private 不匹配）；ajc/LTW 织入能（文档语义，未实测） | VisibilityApp + 前置条件矩阵 |
| @Transactional 标在 protected/包可见方法上生效吗？ | 6.1.14 + CGLIB + 外部经代理调用：生效（allowPublicMethodsOnly=false，javap 实证）；JDK 代理模式下不生效；private/static/final 任何模式不生效 | TxVisibilityApp + javap 实测 |
| 同类内 this 调 @Transactional 为什么不生效？ | 代理只拦外部调用；this 直达目标对象；@Lazy 自注入修复 | 05 篇 + 自调用专节 |
| 怎么做一个自定义能力 starter？ | 注解 + @Aspect + 自动配置类：imports 收录一行 → 条件链 → @Bean 注册切面 → creator 收集；条件关闭 = 注解静默失效（无报错） | demo17 权限切面 starter 实证 |

---

# 坑与细节（10 个）

## 坑 1：@Aspect 不是组件注解

`@Aspect` 单独不会注册为 bean——必须配合 `@Component`（或配置类里 @Bean）。忘了 = 切面整个不存在（本文开发期踩坑）。

## 坑 2：aspectjweaver 缺失 → 用户切面静默失效

`InfrastructureAdvisorAutoProxyCreator.isEligibleAdvisorBean` 只收 role=2（javap 反编译实证）——无 aspectjweaver 时用户 Advisor 被过滤，无任何报错（跑法 3 实测）。**这是 Boot 3.3.5 双分支的行为差异，03 篇没测到的新事实**。

## 坑 3：@annotation 绑定参数用基类 → 通知静默失效

`@annotation` 有两种用法，先分清再踩坑：

| 用法 | 写法 | 语义 |
| ---- | ---- | ---- |
| **纯匹配** | `@annotation(com.example.Marked)`（不带参数名） | 只判断"目标方法上有没有该注解"，**不取注解实例**，通知里拿不到注解属性 |
| **绑定参数** | `@annotation(marked)` + 通知参数 `Marked marked` | 切点参数名与通知参数名一致 → AspectJ 把**注解实例**绑定进通知，可读注解属性（如 `marked.value()`） |

绑定规则（AspectJ 语义）：**绑定参数类型必须与注解类型精确匹配**。用 `java.lang.annotation.Annotation` 基类声明绑定参数 → 绑定解析失败，**通知整体静默失效、不报错**（PointcutApp 实测：该通知不执行，同切面其他通知正常）——排查时注意"通知没执行"与"切点没匹配"症状相同，先看绑定参数类型。

注意：踩坑的永远是**绑定写法**（想要注解属性时的写法）；纯匹配写法不带参数名、无绑定失败问题。若想"匹配任意注解"（基类语义），用 `@within(java.lang.annotation.Annotation)` 或换 `args()`/自定义切点，不要用基类绑定的 `@annotation`。

## 坑 4：@Transactional/@Cacheable 标在 private/static/final 方法上 = 完全无效

代理只能覆写"非 final 的实例方法"（CGLIB 子类化）——private/static 方法**连表达式匹配都不通过**，final 方法**匹配成功也拦不住**（VisibilityApp 实测：`matches(doFinal)=true`、`matches(doStatic)=false`、`matches(doPrivate)=false`）。这些方法上的一切注解型能力（事务/缓存/异步/自定义切面）**静默失效**，无任何报错。**注解型能力标 private/static/final 无效**；protected/包可见在 CGLIB 模式下外部调用可生效（TxVisibilityApp 实测，6.1.14）但依赖代理模式（JDK 代理下无效），生产仍建议 public；AspectJ 织入路线（ajc/LTW）除外（前置条件见 Level 4 矩阵）。

## 坑 5：@Around 不调 proceed = 吞掉目标方法

短路/熔断可用；但 catch 后不重抛 = 业务方无感知（静默吞错）。

## 坑 6：@After 里抛异常覆盖业务结果

@After 异常会改变整体结果语义——横切逻辑弱失败（try/catch 记录，不往外抛）。

## 坑 7：代理实例的字段是 null

字段注入只填目标实例（UnwrapApp 实测）——"从容器拿代理 → 反射字段"是空；业务方法内用注入引用（代理）而非 this 反射。

## 坑 8：切点过宽误伤框架

`execution(* com.x..*.*(..))` 全通配可能命中框架内部对象；用 `@annotation` 驱动或显式排除 `within(org.springframework..*)`。

## 坑 9：main 异常未 close → JVM 不退（SERVLET 模式）

代理对象解包时 ClassCastException 崩在 main → ctx.close() 未执行 → Tomcat 保活（05 篇坑 8 在本篇开发期二次踩中：UnwrapApp 初版 isAopProxy=false 时强转 Advised 崩溃）。**解包前先判 `instanceof Advised`**。

## 坑 10：同类内 this 调用 = 一切注解型能力静默失效

`this.xxx()` 自调用绕过代理（拦截点只在"从代理发出的调用"）——事务/缓存/异步/自定义切面全部不生效且**不报错**（Level 4 自调用专节）。代码评审时遇到"方法内部调同类 @Transactional 方法"要直接标记。

代理对象解包时 ClassCastException 崩在 main → ctx.close() 未执行 → Tomcat 保活（05 篇坑 8 在本篇开发期二次踩中：UnwrapApp 初版 isAopProxy=false 时强转 Advised 崩溃）。**解包前先判 `instanceof Advised`**。

---

# 版本勘误表

| 说法 | 正确版本（本文实证） |
| ---- | ---- |
| "Boot 默认 JDK 动态代理" | Boot **3.x 默认 CGLIB**（proxyTargetClass=true），2.x 才是 JDK——语义变化 |
| "用户切面只要有 spring-aop 就生效" | 必须 aspectjweaver（注解类型提供者 + AnnotationAware 分支） |
| "有 aspectjweaver 时 Boot 走 JDK 代理" | 实测：有 aspectjweaver 也 CGLIB（配置分支归配置分支，代理类型由 proxyTargetClass 决定） |
| "代理对象能直接反射到注入字段" | 代理实例字段 null；注入在目标实例（实测） |
| "@annotation(参数) 可以绑定任意类型" | 必须匹配具体注解类型，基类绑定静默失败（实测） |
| "@Aspect 注解是 spring-aop 提供的" | 来自 aspectjweaver（去 weaver 后 ClassNotFoundException: org.aspectj.lang.ProceedingJoinPoint 实测） |
| "切点表达式能匹配 private/static 方法" | Spring AOP 语境实测：final 匹配= true 但拦不住；static/private 匹配= false |
| "@Transactional 必须标在 public 方法上，否则不生效" | **不准确**：6.1.14 + CGLIB + 外部经代理调用时 protected/包可见生效（TxVisibilityApp 实测 count=0 回滚）；javap 实证 allowPublicMethodsOnly()=false；"只标 public"是跨模式/跨版本（JDK 代理模式）的稳妥建议，不是 6.1.14 的硬性行为 |

---

# 生产决策卡（3 张）

## 决策卡 1：代理类型

```
Decision: 用 Boot 默认（CGLIB），不手动改 proxy-target-class
Reason:   CGLIB 覆盖无接口类；Boot 3 默认已如此（实测）；改 JDK 只省反射转型的微小开销，却引入接口约束
Alternative: proxyTargetClass=false（弃：无接口类回退 CGLIB，接口类才 JDK，行为分叉）
Trade-off: CGLIB 不能代理 final 类/方法（设计时避开）；反射/序列化时类名与目标不同（getTargetClass 解）
Validation: ProxyKindApp 跑法 1/2 复现两种类名
```

## 决策卡 2：切点驱动方式

```
Decision: 优先 @annotation 注解驱动，其次 execution 白名单
Reason:   注解把"谁被拦"显式化，可查可迁移；execution 通配易误伤（Level 6.2）
Alternative: 包级 within（弃：粒度粗，新类自动进切点，风险不可见）
Trade-off: 注解驱动要求业务方法加注解（侵入业务代码一行），换可控性
Validation: PointcutApp 命中矩阵验证表达式预期
```

## 决策卡 3：自定义横切 vs 框架能力

```
Decision: 能复用框架能力（事务/缓存/重试）就不写自定义切面；自定义的用 @Aspect + starter-aop
Reason:   框架 advisor 是 role=2 基础设施，无需 weaver 也生效；自定义切面依赖 aspectjweaver 分支（Level 2 实测）
Alternative: 手动 Advisor bean（弃：受 role 过滤与创建时序影响，可移植性差）
Trade-off: 自定义切面灵活但职责自担（失效排查、顺序、性能）；框架能力语义固定
Validation: 自研权限切面 starter 端到端实证（demo17，双跑法：装配生效 / 条件关闭静默失效）；切面不生效先走 Level 6.1 六条清单
```

### 决策卡 3 的实证收尾：权限切面 starter 端到端（demo17）

"自定义的用 @Aspect + starter-aop"——这句话的完整机关（imports 收录 → 条件装配 → 切面收集 → 代理拦截）用 demo17 一次性实测：自己写一个**权限切面 starter**。

**结构**（单 classpath 模拟独立 artifact——permcheck 包不在任何 scanBasePackages 内，类完全靠自动配置注册，与真实 starter jar"类在依赖里、不参与业务扫描"同构）：

```
src/META-INF/spring/...AutoConfiguration.imports   ← demo17 贡献一行（"每个 jar 贡献一行"的第三个 jar）
src/demo17/permcheck/                                ← "starter jar"的类
  RequireRole                业务注解（@Target(METHOD) @Retention(RUNTIME)）
  RoleContext                ThreadLocal 角色上下文（与 TransactionSynchronizationManager 同构）
  PermissionAspect           @Aspect @Around("@annotation(requireRole)")——绑定参数正确写法
  PermissionAutoConfiguration @AutoConfiguration + @ConditionalOnClass + @ConditionalOnProperty + @Bean
src/demo17/app/OrderService                           ← 业务方：adminOnly() 标 @RequireRole("ADMIN")
```

**链路**（本文全部机制的串联）：imports 收录 → 条件链 → `@Bean` 注册 @Aspect → AnnotationAwareAspectJAutoProxyCreator 收集（aspectjweaver 分支）→ OrderService 变 CGLIB 代理 → 调用被切面校验。

**主场景实测**（`./run.sh demo17.PermissionStarterApp`）：

```
[装配] PermissionAutoConfiguration 条件通过 → 注册 PermissionAspect（模拟"引了 starter"）
[装配] 容器中 permissionAspect bean 存在 = true
[装配] OrderService 是 AOP 代理 = true
--- 场景 1：ADMIN 调用（应放行） ---
[切面] 拦截 OrderService.adminOnly()：@RequireRole("ADMIN") 绑定参数生效；当前角色 = ADMIN
[放行] 管理员操作执行成功
--- 场景 2：USER 调用（应拒绝） ---
[切面] 拦截 OrderService.adminOnly()：@RequireRole("ADMIN") 绑定参数生效；当前角色 = USER
[拒绝] 无权限被拦截：需要角色 ADMIN，当前角色 USER
```

**对照场景实测**（`./run.sh demo17.PermissionStarterApp --demo17.permission.enabled=false`——命令行属性关闭条件 = 模拟"没引 starter"）：

```
[装配] 容器中 permissionAspect bean 存在 = false
[装配] OrderService 是 AOP 代理 = false
--- 场景 2：USER 调用（应拒绝） ---
[放行] 管理员操作执行成功
[异常] 竟然放行了（切面失效！）
```

三个实证钉死的事实：

1. **装配入口只有一处**：切面不在业务扫描范围内，不存在"业务方漏 @ComponentScan"的歧义——**"引没引 starter" = 条件通过与否**，链路因果干净；
2. **失效是静默的**：条件关闭后，bean 不存在、业务对象不是代理，但**注解方法照常执行、零报错**——权限漏洞是"安静"的，这正是 Level 6.1 排查清单存在的意义（"切面没生效"不会主动喊你）；
3. **绑定参数正确写法贯穿链路**：`@annotation(requireRole)` + `RequireRole requireRole` 精确类型绑定（坑 3 的正面示范），切面才能读到 `requireRole.value()`。

生产含义：真实 starter 与单 classpath 模拟的差异只是"类在不在业务 classpath"（真 starter 的类在独立 jar，连类都不存在时 @ConditionalOnClass(name=...) 用 ASM 跳过不触发类加载，03 篇已实证）；**装配机关本身一字不差**——这就是"加一个依赖就能用"的全部答案。

---

# 跨语言视角

- **Python**：`@decorator` 装饰器就是"运行时代理"的同构物——但装饰器**只包被装饰的那个函数**，没有"切点表达式"批量声明；Django middleware 是**请求级**拦截（粒度和 Servlet Filter 同族）；`@transaction.atomic` 与 `@Transactional` 完全同构（也受"self 调用绕过"影响）。
- **Go**：**无动态代理**（无反射代理生成机制）——横切要么手写包装（装饰器模式，静态转发），要么用中间件模式（net/http 的 handler 链，请求级）。Go 社区用"中间件链"弥补，但**方法级 AOP 在 Go 没有语言级答案**——这解释了为什么 Go 生态的"切面"都收敛到 HTTP/函数边界。
- **Java 生态**：AspectJ 织入（编译期）提供比 Spring AOP 更彻底的能力（final/private/构造器），代价是构建期侵入——"机制强度 vs 部署复杂度"的经典取舍。
- **通用方法论**：**"声明式横切"（AOP）vs "显式封装"（装饰器/中间件）**是所有语言面对横切关注点的两条路；声明式的共同代价是**透明性失效静默**（拦截点不可见）——任何声明式方案都要回答"绕过它会发生什么"（自调用/缺依赖/表达式不命中）。

---

# 系列索引

```
00 容器如何创建对象（已重写：四层创建链 + 15 个实测 demo）
01 框架整合 + 配置体系（已重写：6 Level，配置体系独立 Level 4，6 个实测 demo）
02 事件机制与容器通信（已重写：6 Level，5 个实测 demo + 启动事件全景）
03 自动装配深挖（已重写：6 Level，6 个实测 demo：demo10×6）
04 Web 请求链路与运行时刻（已重写：6 Level，4 个实测 demo：demo11.RunTraceApp / WebTraceApp / ActuatorApp / WebFluxApp（双跑法））
05 事务与数据层（已重写：6 Level，4 个实测 demo：demo12.ds.DataSourceApp / tx.TxBasicsApp / tx.PropagationApp / tx.SelfInvocationApp）
06 横切面与 AOP（本篇：6 Level，8 个实测 demo：demo13.aspect.ProxyKindApp / AdviceOrderApp / PointcutApp / AspectOrderApp / ProxyInternalsApp / UnwrapApp / VisibilityApp / TxVisibilityApp + demo17 权限切面 starter 端到端实证）
07 生产实践（已完成：急诊室比喻 + 检查单 7 项 + Level 7 慢发布（指纹测量/三层优化/AOT 选型 + 24 章交叉补充）+ Level 8 优雅停机（demo16 实测 immediate/graceful）+ 决策卡 5 张；实测 demo14×2 + demo15×2 + demo16；与 Boot 4.1 对照线交叉校验）
```

---

# 实验复现

```
cd knowledge/springboot/experiments/code
./build.sh
./run.sh demo13.aspect.ProxyKindApp        # JDK/CGLIB + creator 双分支（跑法 1）
./run.sh demo13.aspect.AdviceOrderApp      # 五种通知正常/异常双路径
./run.sh demo13.aspect.PointcutApp         # 切点命中矩阵（execution/@annotation/within）
./run.sh demo13.aspect.AspectOrderApp      # 多切面 @Order 洋葱模型
./run.sh demo13.aspect.ProxyInternalsApp   # creator 是 BPP / ProxyFactory / Advised
./run.sh demo13.aspect.UnwrapApp           # 代理 vs 目标（字段/toString 差异）
./run.sh demo13.aspect.VisibilityApp        # 代理边界：static/private/final/可见性命中矩阵
./run.sh demo13.aspect.TxVisibilityApp      # 框架注解可见性：@Transactional 在 protected/包可见方法上的实测
./run.sh demo17.PermissionStarterApp        # 权限切面 starter 端到端（装配 → 拦截 → 放行/拒绝）
./run.sh demo17.PermissionStarterApp --demo17.permission.enabled=false   # 对照：条件关闭 → 切面静默失效
# ProxyKindApp 其他两种跑法：
java -Dapp.proxyTargetClass=false -cp "out:$(find lib -name '*.jar' | tr '\n' ':')" demo13.aspect.ProxyKindApp   # 跑法 2：JDK 分支
java -cp "out:$(find lib -name '*.jar' ! -name 'aspectjweaver*' | tr '\n' ':')" demo13.aspect.ProxyKindApp      # 跑法 3：无 weaver → 用户切面失效
```

六个 App 的关键输出都已固化在各自源文件头部注释，与本文引用一致。

---

# ✅ Final Review Checklist

- [ ] 是否解释了为什么存在？（横切关注点散落 + 三波失败方案 → AOP 的切点/通知/切面抽象）
- [ ] 是否说明旧方案为什么失败？（继承耦合横切点、装饰器手工转发、过滤器粒度错）
- [ ] 是否形成完整因果链？（横切问题 → 抽象 → 运行时代理 → 语法体系 → 装配链 → 框架家族 → 生产实践，总图在文中）
- [ ] 是否区分规范和实现？（AspectJ 表达式/绑定规则、通知语义为 Specification；AopAutoConfiguration 双分支行为、creator 类型差异、类名、默认 CGLIB 为 3.3.5 Implementation）
- [ ] 是否区分语义变化与代码组织变化？（Boot 2→3 默认代理类型变化 = 语义变化；creator 类名/分支归属 = 代码组织；jdk 动态代理类名格式差异标注待验证）
- [ ] 代码实例是否全部实测？（demo13×8 输出原样引用，可复跑；InfrastructureAdvisorAutoProxyCreator.isEligibleAdvisorBean 的 role 过滤、AbstractFallbackTransactionAttributeSource.allowPublicMethodsOnly()=false 均经 javap 反编译实证）
- [ ] 是否包含 Trade-off？（运行时代理 vs 编译期织入；CGLIB vs JDK；注解驱动 vs 通配；自定义切面 vs 框架能力）
- [ ] 是否能指导生产决策？（3 张决策卡：代理类型 / 切点驱动 / 自定义横切 + 失效排查六条清单）
- [ ] 是否存在未经证明的数字？（无编造 benchmark；无 @Order 时顺序不确定性、非 public 方法行为、AspectJ LTW 均标注待验证）
- [ ] 是否只有一个比喻？（关卡哨兵）是否只有一个主线角色？（一次方法调用穿越层层关卡）
- [ ] 随机抽查断言：Boot3 默认 CGLIB 且 aspectjweaver 存在也 CGLIB（跑法 1）、proxyTargetClass=false 时接口→JDK $Proxy76 / 无接口→CGLIB（跑法 2）、无 weaver 时用户 Advisor 被 role 过滤（跑法 3 + javap）、五种通知双路径顺序（AdviceOrderApp）、命中矩阵（PointcutApp）、洋葱模型（AspectOrderApp）、creator 是 BPP（ProxyInternalsApp）、代理字段 null 且 toString 转发（UnwrapApp）、@Aspect 注解类型来自 aspectjweaver（ClassNotFoundException 实测）——均有证据来源。
