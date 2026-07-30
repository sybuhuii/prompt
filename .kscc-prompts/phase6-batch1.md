你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须先完整阅读并遵守根目录`AGENTS.md`。其中已有架构、依赖、代码质量、安全、编译及输出规范，本提示词不再重复。

当前只执行第六阶段第1批：建立HITL中断与Checkpoint基础模型，完善CheckpointStore抽象，并提供线程安全内存实现。

本批不修改ReAct图，不拦截工具，不真正中断运行，不实现approve/resume API，不实现前端页面。

# 一、执行前检查

先检查：

1.`AgentCheckpoint`
2.`CheckpointStore`
3.现有Approval、Interrupt、Resume相关类型
4.`RunContext`
5.`AgentTask`、`AgentResult`
6.`ReactAgentState`、`SupervisorAgentState`
7.`ToolCall`、`ToolInvocation`、`ToolRiskLevel`
8.`AgentErrorCode`
9.现有内存Store实现和Spring装配方式
10.根目录`AGENTS.md`

要求：

1.优先复用第一阶段已有`AgentCheckpoint`、`CheckpointStore`和审批类型。
2.禁止创建`CheckpointV2`、`NewCheckpointStore`等重复抽象。
3.现有模型缺少必要字段时做最小兼容扩展。
4.不得为了本批修改ReAct、Supervisor或ToolInvocationGateway。
5.不得实现范围外功能。

# 二、恢复语义

本框架明确采用“节点重跑”恢复语义：

1.中断发生在危险操作真正执行之前。
2.中断时保存当前图状态、待审批操作和恢复节点。
3.恢复后从保存的节点重新执行，而不是从Java方法中间继续。
4.节点重新执行时必须先读取审批结果。
5.已批准则允许继续。
6.已拒绝则形成安全失败结果，不执行危险操作。
7.仍处于待审批状态时不得继续执行。
8.后续执行节点必须考虑幂等性。
9.本批只定义该语义所需模型，不实现实际恢复。

在相关模型注释中明确该语义，不创建README或额外文档。

# 三、HITL领域模型

检查现有审批类型，复用或最小完善以下语义。

## 3.1 ApprovalStatus

至少表达：

- PENDING
- APPROVED
- REJECTED

如现有枚举已有等价值，直接复用。

不要加入复杂多级审批状态。

## 3.2 ApprovalDecision

不可变模型，至少包含：

- approvalId
- status
- decidedBy
- comment
- decidedAt

要求：

1.状态只能是APPROVED或REJECTED。
2.不得使用PENDING表示人工决定。
3.decidedBy不能为空。
4.comment允许为空，但需统一空值规范。
5.decidedAt不能为空。
6.不得包含密码、Session完整内容或HTTP对象。
7.不得允许模型生成或控制人工审批身份。

## 3.3 InterruptPayload

不可变模型，表达中断时展示给审批人的信息，至少包含：

- approvalId
- runId
- threadId
- userId
- agentName
- nodeName
- reason
- operationType
- operationName
- safeArguments
- riskLevel
- requestedAt

要求：

1.safeArguments只能保存经过脱敏和长度限制的参数。
2.不得保存API Key、密码、credentialHash或Session ID。
3.不得直接保存任意异常对象。
4.不得保存Spring AI或LangGraph4j对象。
5.operationType至少能区分TOOL和NODE。
6.当前主要支持TOOL，但模型应允许未来节点级中断。
7.riskLevel复用现有`ToolRiskLevel`；若其位于不合适模块，检查依赖后最小调整，不复制枚举。
8.reason是可展示的安全说明，不保存详细思维链。

## 3.4 PendingApproval

不可变模型，至少包含：

- InterruptPayload payload
- ApprovalStatus status
- ApprovalDecision decision
- createdAt
- updatedAt

要求：

1.初始status=PENDING，decision为空。
2.APPROVED或REJECTED时decision必须存在。
3.PENDING时decision必须为空。
4.不得通过null表达整个PendingApproval不存在；查询不存在使用Optional。
5.状态转换后生成新对象，不修改旧对象。

# 四、Checkpoint模型

检查已有`AgentCheckpoint`，使其能够完整表达一次挂起执行。

至少应保存：

- checkpointId
- runId
- threadId
- userId
- sessionId或其安全关联信息
- executionType
- agentName
- nodeName
- stateData
- pendingApproval
- status
- version
- createdAt
- updatedAt

具体字段以现有模型为准，禁止无意义重命名。

## 4.1 executionType

至少区分：

- REACT_AGENT
- SUPERVISOR

如已有图类型枚举则复用。

## 4.2 CheckpointStatus

至少表达：

- SUSPENDED
- RESUMING
- COMPLETED
- FAILED

本批实际主要保存SUSPENDED。

要求：

1.不得使用普通字符串散落表示状态。
2.状态不能为空。
3.SUSPENDED时必须存在pendingApproval。
4.COMPLETED时不得继续被恢复。
5.version必须大于等于0，用于后续并发恢复控制。
6.createdAt和updatedAt不能为空。
7.模型保持不可变。

## 4.3 stateData

stateData保存恢复图运行所需的状态快照。

要求：

1.本批使用`Map<String,Object>`或现有等价结构时，必须创建不可变快照。
2.不得直接保存可变`ReactAgentState`或`SupervisorAgentState`实例。
3.不得保存CompiledGraph、Gateway、Registry、Spring Bean或异常对象。
4.不得保存模型客户端。
5.不得把UserSession完整对象塞入stateData。
6.允许保存已有领域消息、任务、结果和RunContext。
7.后续持久化适配需要序列化，本批不引入数据库和复杂Codec。
8.内存实现必须保证调用方修改原Map不会改变已保存Checkpoint。
9.读取时返回独立不可变快照。

# 五、CheckpointStore

必须保留现有`CheckpointStore`名称和位置。

接口至少支持：

```java
void save(AgentCheckpoint checkpoint);
Optional<AgentCheckpoint> load(String runId);
Optional<AgentCheckpoint> loadByThreadId(String threadId);
Collection<AgentCheckpoint> findPendingByUserId(String userId);
void delete(String runId);
```

若现有方法使用find而不是load，优先兼容已有命名，不为形式统一破坏接口。

要求：

1.按runId保存和恢复。
2.支持按threadId查询。
3.支持按userId查询当前待审批Checkpoint。
4.不得只根据客户端userId返回数据，后续应用服务仍需校验身份。
5.未找到返回Optional.empty。
6.查询集合返回不可变快照。
7.接口位于agent-core。
8.接口不得依赖Spring、LangGraph4j或数据库。
9.Store不负责生成ID。
10.Store不负责审批决策。
11.Store不负责执行恢复。
12.Store不负责调用模型或工具。

如已有CheckpointStore签名与上述不同：

1.保留现有方法。
2.只增加当前功能必需的最小方法。
3.最终输出说明实际接口。

六、并发更新语义

为了防止同一运行被重复审批或重复恢复，CheckpointStore必须支持基于version的条件更新。

优先增加：

boolean updateIfVersionMatches(
    AgentCheckpoint checkpoint,
    long expectedVersion
);

或现有风格的等价方法。

规则：

1.保存新Checkpoint时version=0。
2.每次成功状态更新后version+1。
3.expectedVersion不匹配时返回false。
4.不得静默覆盖较新状态。
5.不得以异常作为正常版本冲突控制流。
6.后续审批和恢复必须使用该能力。
7.本批只提供Store能力，不实现审批流程。

七、内存CheckpointStore

在agent-infrastructure实现或完善：

InMemoryCheckpointStore implements CheckpointStore

要求：

1.使用ConcurrentHashMap，以runId为主键。
2.支持threadId和userId查询。
3.不得使用static Map。
4.save拒绝null。
5.相同runId保存不同Checkpoint时不得无条件覆盖。
6.首次save用于创建；更新使用version条件更新。
7.相同Checkpoint重复save可幂等或明确拒绝，行为必须稳定。
8.load返回不可变Checkpoint快照。
9.delete不存在时幂等。
10.findPendingByUserId只返回：

userId匹配
status=SUSPENDED
approvalStatus=PENDING
11.不同用户不得串读。
12.按requestedAt或createdAt稳定排序，建议时间升序。
13.threadId索引和runId主存储必须保持一致。
14.并发保存、更新、删除时不能产生脏索引。
15.可使用局部锁或ConcurrentHashMap原子操作保证多索引一致性。
16.不得锁住全部只读查询。
17.不得自动清理过期Checkpoint。
18.不得执行恢复。
19.不得添加Spring组件注解。
八、Checkpoint ID生成

检查现有ID生成器。

如没有适合的Checkpoint或Approval ID生成器，在agent-core定义：

CheckpointIdGenerator
ApprovalIdGenerator

在agent-infrastructure提供安全、无状态实现。

要求：

1.普通UUID v4可接受。
2.不得使用数据库自增假设。
3.不得包含userId、sessionId和工具参数。
4.不得在日志记录审批ID与敏感参数的组合。
5.已有通用安全ID生成器可复用时不要重复创建。

九、Checkpoint状态校验

在agent-runtime实现纯Java：

CheckpointValidator

职责：

1.校验Checkpoint必要字段。
2.校验runId、threadId、userId非空。
3.校验version合法。
4.校验SUSPENDED状态存在PendingApproval。
5.校验审批状态和decision一致。
6.校验stateData不为空。
7.校验nodeName存在。
8.不访问Spring容器。
9.不调用模型和工具。
10.不修改Checkpoint。
11.失败使用现有AgentFrameworkException及明确错误码。

如错误码缺失，最小补充：

CHECKPOINT_NOT_FOUND
CHECKPOINT_CONFLICT
RUN_SUSPENDED
APPROVAL_NOT_FOUND
APPROVAL_ALREADY_DECIDED
INVALID_APPROVAL_DECISION

不要增加语义重复错误码。

十、Spring装配

在agent-infrastructure通过配置类装配：

CheckpointStore→InMemoryCheckpointStore
CheckpointValidator
CheckpointIdGenerator（如新增）
ApprovalIdGenerator（如新增）

要求：

1.使用@Configuration和@Bean。
2.默认Store允许通过@ConditionalOnMissingBean替换。
3.不得创建数据库或Redis实现。
4.不得在bootstrap启动类手工new。
5.无模型配置时Checkpoint能力仍能正常装配。
6.不得装配Fake审批服务。
7.本批不装配中断拦截器和恢复Engine。

十一、多用户隔离

必须保证：

1.Checkpoint明确保存userId、runId和threadId。
2.按userId查询不得返回其他用户Checkpoint。
3.后续审批操作必须同时校验当前Session.userId和Checkpoint.userId。
4.管理员跨用户审批策略本批不实现。
5.不得仅凭approvalId返回Checkpoint给任意用户。
6.sessionId不得出现在可展示InterruptPayload中。
7.身份信息不得发送给LLM。
8.不得使用ThreadLocal保存当前Checkpoint或审批人。

十二、安全限制

1.safeArguments必须进行浅层脱敏和长度限制。
2.本批可定义SensitiveValueSanitizer接口或复用已有能力，但不要实现复杂递归序列化框架。
3.至少屏蔽以下Key：

password
credential
token
apiKey
secret
authorization
sessionId
4.Key匹配忽略大小写。
5.敏感值替换为***。
6.单个值和整体payload设置合理长度上限。
7.不得把完整工具参数写入日志。
8.不得把stateData通过查询接口直接暴露。
9.本批不新增HTTP接口。
十三、本批禁止实现

禁止：

1.修改ReAct或Supervisor图结构。
2.Tool审批拦截器。
3.危险工具策略判断。
4.真正抛出中断。
5.调用LangGraph4j resume。
6.审批Application Service。
7.approve/reject Controller。
8.待审批查询Controller。
9.Web审批页面。
10.邮件、删除、转账等真实危险业务工具。
11.数据库或Redis Checkpoint。
12.Checkpoint TTL自动清理。
13.跨进程恢复。
14.分布式锁。
15.SSE。
16.测试脚本、README和Git操作。

十四、编译验收

执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

检查：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.复用了已有AgentCheckpoint和CheckpointStore。
4.CheckpointStore位于agent-core。
5.InMemoryCheckpointStore位于agent-infrastructure。
6.Checkpoint模型不依赖SpringAI或LangGraph4j。
7.stateData为不可变快照。
8.多用户查询不会串读。
9.version条件更新不会静默覆盖。
10.没有修改ReAct和Supervisor图。
11.没有中断、恢复或HTTP接口。
12.没有范围外修改。

十五、最终输出

只输出：

1.新增和修改文件。
2.复用或调整的已有模型。
3.AgentCheckpoint最终字段。
4.CheckpointStore最终接口。
5.节点重跑恢复语义。
6.PendingApproval状态约束。
7.InMemoryCheckpointStore线程安全及索引策略。
8.version条件更新策略。
9.safeArguments脱敏规则。
10.Spring Bean装配。
11.编译和打包结果。
12.发现但未处理的问题。

编译失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE6_BATCH1_END===