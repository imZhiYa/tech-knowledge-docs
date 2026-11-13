# 04｜从用例到 Java 代码：让职责边界真正落到工程结构

## 本篇目标

把 AssignLead 用例落到 Java 模块：入口适配、应用编排、领域行为、持久化端口和基础设施实现各自做什么。重点不是复制包名，而是能解释依赖方向。

## 1. 先看一次用例的运行路径

~~~text
HTTP Request
    |
    v
LeadController                 Interface Adapter
    |
    v
AssignLeadUseCase              Application Port
    |
    v
AssignLeadHandler              Application
    |             |          -- SalesRepQueryPort
    v
LeadRepository                 Domain/Application Port
    |
    v
Lead.assignTo()                Domain Behavior
    |
    v
LeadRepository.save()          Port
    |
    v
MySqlLeadRepository            Infrastructure Adapter
    |
    v
Database
~~~

这里的端口归属取决于谁拥有抽象：若接口表达领域聚合的加载保存能力，可以由领域模块定义；若只是应用用例所需的外部查询能力，可以由应用层定义。不要把所有接口机械地放进 Domain。

## 2. 建议的模块结构

~~~text
lead/
├── interfaces/
│   └── http/
│       ├── LeadController.java
│       └── AssignLeadRequest.java
├── application/
│   ├── AssignLeadUseCase.java
│   ├── AssignLeadHandler.java
│   ├── SalesRepQueryPort.java
│   └── AssignLeadResult.java
├── domain/
│   ├── Lead.java
│   ├── LeadId.java
│   ├── LeadStatus.java
│   ├── SalesRepId.java
│   ├── LeadRepository.java
│   └── LeadAssigned.java
└── infrastructure/
    └── persistence/
        ├── LeadPO.java
        ├── LeadMapper.java
        └── MySqlLeadRepository.java
~~~

包结构只是一个可选示例。小型模块可以减少目录；关键是依赖方向和职责边界，而不是目录数量。

## 3. Application 负责用例编排

~~~java
public interface AssignLeadUseCase {
    AssignLeadResult handle(AssignLeadCommand command);
}

public record AssignLeadCommand(
    UUID leadId,
    UUID salesRepId,
    UUID operatorId
) {}

public record AssignLeadResult(
    UUID leadId,
    String status
) {}
~~~

应用服务的职责：

- 接收用例命令。
- 检查调用者身份和应用级权限（领域规则仍应在可信边界内保护）。
- 加载需要的聚合和外部事实。
- 调用领域行为。
- 在事务中保存聚合和 Outbox 记录（若该用例需要可靠发布事件）。
- 返回用例结果。

不要把复杂状态迁移规则写在 Handler 中，否则领域规则又散回应用层。

## 4. 领域仓储表达聚合持久化意图

~~~java
public interface LeadRepository {
    Optional<Lead> findById(LeadId id);
    void save(Lead lead);
}
~~~

Repository 不等于所有 SQL 的统一入口。它通常围绕聚合根提供加载和保存能力。复杂报表查询可以使用独立 Query Service 或 Read Model，不必强行通过聚合仓储查询。

## 5. Infrastructure 负责技术实现

~~~java
public final class MySqlLeadRepository implements LeadRepository {
    private final LeadJpaRepository jpaRepository;
    private final LeadMapper mapper;

    public MySqlLeadRepository(
            LeadJpaRepository jpaRepository,
            LeadMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }

    @Override
    public Optional<Lead> findById(LeadId id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }

    @Override
    public void save(Lead lead) {
        jpaRepository.save(mapper.toPersistence(lead));
    }
}
~~~

这里的 LeadJpaRepository 是基础设施内部的 ORM 接口。领域模型不应该依赖 JPA Entity、Spring Data 或数据库 SDK。

生产实现还需要正确处理乐观锁异常、事务边界、唯一约束和数据库异常映射。示例省略了这些框架细节，不应直接作为完整生产代码。

## 6. Controller 不承载业务规则

~~~java
@RestController
@RequestMapping("/leads")
public class LeadController {
    private final AssignLeadUseCase assignLeadUseCase;

    public LeadController(AssignLeadUseCase assignLeadUseCase) {
        this.assignLeadUseCase = assignLeadUseCase;
    }

    @PostMapping("/{leadId}/assignment")
    public AssignLeadResponse assign(
            @PathVariable UUID leadId,
            @RequestBody AssignLeadRequest request) {
        var result = assignLeadUseCase.handle(
            new AssignLeadCommand(
                leadId,
                request.salesRepId(),
                request.operatorId()
            )
        );
        return new AssignLeadResponse(result.leadId(), result.status());
    }
}
~~~

HTTP DTO 是接口契约，不是领域模型。API 字段变化不应该自动改变领域对象；领域模型变化也不应该未经评估就破坏 API。

## 7. 事务边界由用例定义

一次分配可能包含：

~~~text
BEGIN
  读取 Lead
  校验目标销售可用
  调用 Lead.assignTo()
  更新 Lead（带 version 条件）
  写入 Outbox（若需通知其他上下文）
COMMIT
~~~

如果目标销售查询来自远程服务，不能把远程网络调用随意放进长时间数据库事务。可以在事务前获取可验证的事实，或采用本地快照/契约并明确其时效性；具体策略取决于权限与业务一致性要求。

## 8. 为什么不是所有项目都要完整四层？

| 场景 | 可以选择 |
|---|---|
| 简单 CRUD，几乎没有业务规则 | Controller + Application Service + DAO |
| 有少量规则但复杂度有限 | 模块化 Service + 明确校验 |
| 多状态、多角色、规则频繁变化 | 领域模型保护核心不变量 |
| 查询复杂、以分析为主 | Query Model / SQL / 数据平台 |
| 多上下文独立演进 | 明确契约，必要时独立部署 |

DDD 不要求每个表都创建 Entity、Repository、Factory、DomainService。抽象必须解决实际变化或规则保护问题。

## 本篇交付物

- 一个用例时序图。
- 一个职责清晰的模块结构。
- Application、Domain、Infrastructure 的依赖关系。
- 事务范围和异常映射表。
- 单元测试与集成测试边界。

**口诀：应用层编排用例，领域层保护规则，基础设施实现技术细节。**