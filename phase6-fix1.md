你正在修复基于 Spring AI + LangGraph4j 的通用 Agent 框架中 Phase6 Batch1 与 Batch2 的错误实现。

必须先完整阅读并严格遵守根目录 AGENTS.md，以及：

- .kscc-prompts/phase6-batch1.md
- .kscc-prompts/phase6-batch2.md

本次目标是修复现有实现，不是重新创建另一套 V2/New 类型。必须以仓库真实代码、真实依赖版本和现有调用链为准。

禁止执行 git commit、git push、创建分支或修改远程仓库。
禁止创建 README、验收报告、测试脚本或测试工程。
禁止提前实现 Phase6 Batch3 的 approve、reject、resume、审批查询 API 和前端页面。
禁止引入数据库、Redis、分布式锁、SSE。
禁止使用 Fake 工具、固定成功结果或隐藏权限绕过。
完成前必须真实编译，不能只修复第一个编译错误后停止。

# 一、先修复当前编译错误

当前已知错误：

agent-runtime/src/main/java/com/ksyun/agent/runtime/react/node/DefaultReactToolExecutionNode.java

代码引用了不存在的：

AgentErrorCode.INVALID_STATE

这里是图内部状态错误，优先复用：

AgentErrorCode.INTERNAL_ERROR

不要仅为这一处新增语义重复的 INVALID_STATE 错误码。

修复后继续完整编译并处理后续真实编译错误。

# 二、修正 Batch1 HITL 和 Checkpoint 基础模型

## 2.1 ApprovalDecision

当前 ApprovalDecision 只是 APPROVE/REJECT 枚举，不符合要求。

将其最小调整为不可变模型，至少包含：

- approvalId
- ApprovalStatus status
- decidedBy
- comment
- decidedAt

约束：

1. status 只能是 APPROVED 或 REJECTED。
2. 不允许 PENDING 作为人工决定。
3. approvalId、decidedBy、decidedAt 非空。
4. comment 允许为空，但需统一将 null 规范化为空字符串。
5. 不包含 Session、HTTP、密码、权限集合或模型输出身份。
6. 修复现有 AgentRuntime 等真实引用，禁止创建 ApprovalDecisionV2。

## 2.2 InterruptPayload

在 agent-core 的审批领域包中增加或完善不可变 InterruptPayload，至少包含：

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
- toolCallId
- operationFingerprint

operationType 至少支持 TOOL 和 NODE。

约束：

1. safeArguments 只能保存脱敏和长度限制后的参数。
2. 不得保存 sessionId、密码、token、credentialHash、API Key。
3. 不得保存 Spring AI、LangGraph4j、Servlet、异常或 Bean。
4. reason 只能是安全展示说明，不是模型思维链。
5. 不得把原始 ToolCall arguments 放入可展示 payload。
6. operationFingerprint 不得发送给 LLM。

删除或停止使用“无权限工具可通过人工审批强制执行”的语义。ACL 拒绝必须直接拒绝，不能创建审批。

## 2.3 PendingApproval

复用现有 PendingApproval 名称，调整为至少包含：

- InterruptPayload payload
- ApprovalStatus status
- ApprovalDecision decision
- createdAt
- updatedAt

约束：

1. PENDING 时 decision 必须为空。
2. APPROVED 或 REJECTED 时 decision 必须存在。
3. decision 的 approvalId 和 status 必须与 PendingApproval 一致。
4. 状态转换生成新对象，禁止修改旧对象。
5. 不得直接保存用于展示的原始 ToolCall。
6. Map 和集合必须防御性复制并返回不可变快照。

## 2.4 AgentCheckpoint

复用 AgentCheckpoint，至少使其包含：

- checkpointId
- runId
- threadId
- userId
- sessionId 或明确的安全关联字段
- executionType
- agentName
- nodeName
- stateData
- pendingApproval
- status
- version
- createdAt
- updatedAt

新增或复用明确枚举：

CheckpointExecutionType：
- REACT_AGENT
- SUPERVISOR

CheckpointStatus：
- SUSPENDED
- RESUMING
- COMPLETED
- FAILED

不要继续使用 RunStatus.INTERRUPTED 代替 CheckpointStatus。

约束：

1. SUSPENDED 必须有 pendingApproval。
2. COMPLETED 不得继续恢复。
3. version >= 0。
4. createdAt、updatedAt 非空。
5. stateData 不得为空。
6. stateData 必须是真正的不可变快照。
7. 不能只使用 Collections.unmodifiableMap(originalMap)，必须先复制。
8. 对状态中的已知 List、Set、Map 做防御性复制，防止原 State 后续变化修改 Checkpoint。
9. 不保存 CompiledGraph、Gateway、Registry、Spring Bean、模型客户端或异常。
10. 注释中明确“节点重跑”恢复语义。

## 2.5 CheckpointStore

复用 agent-core 中现有 CheckpointStore，至少提供：

void save(AgentCheckpoint checkpoint);

Optional<AgentCheckpoint> load(String runId);

Optional<AgentCheckpoint> loadByThreadId(String threadId);

Collection<AgentCheckpoint> findPendingByUserId(String userId);

boolean updateIfVersionMatches(
    AgentCheckpoint checkpoint,
    long expectedVersion
);

void delete(String runId);

语义：

1. save 只负责首次创建。
2. 新 Checkpoint 的 version 必须为 0。
3. 已存在相同 runId 时不得无条件覆盖。
4. 相同 Checkpoint 重复 save 可幂等；不同内容必须明确冲突。
5. 正常版本冲突通过 updateIfVersionMatches 返回 false，不使用异常作为正常控制流。
6. 成功更新后的 version 必须为 expectedVersion + 1。
7. 查询不存在返回 Optional.empty。
8. 查询集合返回不可变快照。
9. 保留现有方法仅限确有兼容需要，不增加无用接口。

## 2.6 InMemoryCheckpointStore

修复现有实现：

1. 使用 ConcurrentHashMap，以 runId 为主键。
2. 禁止 synchronized(runId.intern())。
3. 禁止 static Map。
4. 所有写操作必须使用同一套明确的实例级并发控制，确保主存储与 threadId 索引一致。
5. delete、save、update 和 deleteByThreadId 如保留，必须共享一致的写入同步策略。
6. threadId 变化时必须清理旧索引。
7. 不得锁住全部普通只读查询。
8. findPendingByUserId 只能返回：
   - userId 精确匹配
   - CheckpointStatus.SUSPENDED
   - ApprovalStatus.PENDING
9. 按 requestedAt 或 createdAt 升序稳定排序。
10. 不同用户不得串读。
11. delete 不存在时幂等。
12. load 返回的对象必须是完整不可变快照。

## 2.7 ID、Validator、错误码和脱敏

如无可复用的合适生成器，在 agent-core 定义：

- CheckpointIdGenerator
- ApprovalIdGenerator

在 agent-infrastructure 提供无状态 UUID 实现，并通过 @Bean 装配。

在 agent-runtime 增加纯 Java CheckpointValidator，校验：

- 必要字段
- runId、threadId、userId、nodeName
- version
- stateData
- SUSPENDED 与 pendingApproval
- ApprovalStatus 与 ApprovalDecision 一致性

失败使用 AgentFrameworkException 和明确错误码。

按缺失情况最小补充：

- CHECKPOINT_CONFLICT
- RUN_SUSPENDED
- APPROVAL_NOT_FOUND
- APPROVAL_ALREADY_DECIDED
- INVALID_APPROVAL_DECISION
- APPROVAL_REJECTED

实现或复用 SensitiveValueSanitizer：

1. key 忽略大小写。
2. 至少屏蔽 password、credential、token、apiKey、secret、authorization、sessionId。
3. 敏感值替换为 ***。
4. 限制单个值长度和整体 payload 长度。
5. 不实现复杂反射或通用递归序列化框架。
6. 不记录完整工具参数。

# 三、修正 Batch2 工具审批链

## 3.1 ToolRiskLevel

现有 LOW、MEDIUM、HIGH 可以复用为 SAFE、SENSITIVE、DANGEROUS 的等价语义，不必为了改名破坏已有调用点。

固定规则：

- LOW：不审批
- MEDIUM：本批不审批
- HIGH：必须审批

ToolDefinition.riskLevel 不得为 null。普通工具默认 LOW，必要时增加兼容构造器或紧凑构造器。

不得根据工具名、Agent 名、ADMIN 角色或通配权限判断是否需要审批。

## 3.2 ToolApprovalPolicy

将当前过于简单的 DangerousToolApprovalPolicy 最小重构为提示词要求的统一语义，优先使用名称：

ToolApprovalPolicy

方法：

ToolApprovalRequirement evaluate(
    ToolDefinition definition,
    ToolCall call,
    RunContext context
);

ToolApprovalRequirement 为不可变模型，至少包含：

- required
- riskLevel
- reasonCode
- displayReason

DefaultToolApprovalPolicy 规则：

1. LOW、MEDIUM 不审批。
2. HIGH 必须审批。
3. ADMIN 不绕过。
4. tool:*:invoke 不绕过。
5. RunContext 缺失时失败关闭。
6. ToolDefinition 缺失沿用 TOOL_NOT_FOUND。
7. displayReason 不包含参数、Session、权限集合和思维链。
8. runtime 类型保持纯 Java，不添加 Spring 注解。

禁止同时保留两套语义重复的审批策略。

## 3.3 ToolOperationFingerprint

在 runtime 实现纯 Java ToolOperationFingerprint。

根据固定顺序计算 SHA-256：

- runId
- toolCallId
- toolName
- 原始 arguments 的确定性表示

要求：

1. 同一 ToolCall 稳定产生相同 fingerprint。
2. 名称、ID 或参数改变时 fingerprint 改变。
3. Map key 顺序必须确定。
4. 不引入额外加密依赖。
5. 不记录原始参数。
6. fingerprint 只用于绑定验证，不是身份凭证。

## 3.4 ToolInvocation

复用现有 ToolInvocation，增加：

Optional<PendingApproval> approval

保留两参数兼容构造器，使现有调用默认 approval=Optional.empty()。

approval：

- 不得来自 HTTP Body。
- 不得由模型创建。
- 不得放入 RunContext。
- 不得创建 ToolInvocationV2。

## 3.5 AgentInterruptSignal

在 agent-runtime 实现：

AgentInterruptSignal extends RuntimeException

只携带 PendingApproval。

约束：

1. 它是控制信号，不是普通工具失败。
2. message 只能是安全说明。
3. 不携带 ReactAgentState、Session、Spring、Servlet、密码和密钥。
4. 只有 ToolApprovalInterceptor 创建。
5. 普通异常不能伪装为中断。

## 3.6 ToolApprovalInterceptor

用符合提示词语义的 ToolApprovalInterceptor 替换或重构当前 ToolApprovalGateInterceptor，禁止并存两套重复拦截器。

依赖：

- ToolRegistry
- ToolApprovalPolicy
- ApprovalIdGenerator
- SensitiveValueSanitizer
- ToolOperationFingerprint
- Clock

首次危险调用：

1. 从 ToolRegistry 获取真实 ToolDefinition。
2. 调用 ToolApprovalPolicy。
3. 不需要审批则 chain.proceed。
4. 需要审批且 invocation.approval 为空：
   - 生成 approvalId
   - 计算 fingerprint
   - 脱敏参数
   - 构造 InterruptPayload
   - 构造 PENDING PendingApproval
   - 抛 AgentInterruptSignal
5. 不得调用下游和 Terminal。
6. 不得返回伪成功 ToolResult。

已有 approval 时：

PENDING：
- 严格校验 runId、userId、toolCallId、toolName、fingerprint。
- 匹配后重新抛出相同 PendingApproval。
- 不生成新 approvalId。
- 不执行工具。

APPROVED：
- 严格校验绑定信息。
- 校验通过后 chain.proceed。
- 不再次中断。

REJECTED：
- 严格校验绑定信息。
- 不执行下游。
- 返回 ToolResult.success=false。
- errorCode=APPROVAL_REJECTED。
- 安全说明为“人工审批已拒绝该工具操作，工具未执行。”

非法或绑定不匹配：
- 抛 INVALID_APPROVAL_DECISION。
- 不执行工具。
- 不创建新审批覆盖旧审批。

# 四、修正拦截器传播和顺序

实际顺序必须明确为：

异常治理 → 审计 → ACL → 参数校验 → 人工审批 → Terminal

当前审批位于参数校验之前，必须调整。

为每个拦截器设置明确且唯一的 order，例如：

- Exception：-1000
- Audit：-500
- ACL：-200
- ArgumentValidation：-100
- Approval：0

不要依赖 Spring 注入列表的偶然顺序。

修改 ToolExceptionHandlingInterceptor：

- 捕获 AgentInterruptSignal 时原样抛出。
- 不转换成 ToolResult。
- 其他 AgentFrameworkException 保持现有安全失败语义。

修改 ToolAuditInterceptor：

- 捕获 AgentInterruptSignal 时记录安全的 APPROVAL_REQUIRED/SUSPENDED。
- 原样抛出。
- 不记录为成功。
- 不记录完整参数、Session、权限集合或 stateData。
- 不把正常中断记录为系统 ERROR。

必须保证：

- ACL 拒绝时不创建审批。
- 参数非法时不创建审批。
- 只有授权且参数合法的 HIGH 工具才创建审批。

# 五、实现 ToolExecution cursor 和 buffer

在 ReactStateKeys、ReactAgentGraphFactory channels 和 DefaultReactAgentEngine 初始状态中增加：

- TOOL_EXECUTION_CURSOR
- TOOL_EXECUTION_BUFFER
- PENDING_APPROVAL
- CHECKPOINT_ID
- 如现有状态缺失，增加明确 RUN_STATUS

语义：

TOOL_EXECUTION_CURSOR：
- 当前待执行 ToolCall 下标
- 初始 0
- 每完成一个 ToolCall 后 +1
- 中断时保持当前危险工具下标
- 覆盖 Channel

TOOL_EXECUTION_BUFFER：
- 保存本轮已经完成但尚未 Observe 的 ToolResult
- 初始空
- 中断时保存
- 恢复后继续累积
- 全部完成后一次性写入 latestToolResults
- 覆盖 Channel，禁止 Appender

PENDING_APPROVAL：
- 当前唯一审批
- 普通运行为空
- 中断时保存
- 覆盖 Channel

CHECKPOINT_ID：
- 当前挂起 Checkpoint ID
- 普通运行为空
- 不发送给 LLM

所有读取通过 ReactStateKeys 类型安全方法，禁止散落字符串 Key，禁止在 ReactAgentState 中重复声明普通字段。

# 六、改造 DefaultReactToolExecutionNode

最小改造现有节点：

正常流程：

1. 读取 pendingToolCalls。
2. 读取 cursor，默认 0。
3. 读取 executionBuffer，默认空。
4. 从 cursor 顺序执行。
5. 创建 ToolInvocation。
6. 首次运行 approval 为空。
7. 调用 ToolInvocationGateway。
8. 正常 ToolResult，包括 ACL 或参数失败结果，都加入 buffer。
9. cursor 前进。
10. 全部完成后：
    - 完整 buffer 写入 latestToolResults
    - executionBuffer 清空
    - cursor 重置 0
    - pendingApproval、checkpointId 清空
    - 正常进入 observe

捕获 AgentInterruptSignal：

1. 不转成 ToolResult。
2. 不执行后续 ToolCall。
3. 不进入 Observe。
4. cursor 保持当前下标。
5. buffer 保留。
6. 立即调用 ReactCheckpointService 保存 Checkpoint。
7. 写入 pendingApproval、checkpointId、RunStatus.SUSPENDED。
8. 返回状态增量。
9. 不增加 iteration。
10. 不删除 pendingToolCalls。
11. 不修改 ToolCall。
12. 保存失败必须进入框架失败路径，不能伪装成挂起。

注入 ReactCheckpointService 时通过构造器和 Spring @Bean 装配，runtime 类不加 Spring 注解。

# 七、重写 ReactCheckpointService 的职责

ReactCheckpointService 依赖：

- CheckpointStore
- CheckpointIdGenerator
- CheckpointValidator
- Clock

提供类似：

AgentCheckpoint suspend(
    ReactAgentState state,
    String nodeName,
    PendingApproval approval,
    int cursor,
    List<ToolResult> buffer
);

要求：

1. approval 由 ToolApprovalInterceptor 创建，Service 不重新生成 approvalId。
2. 创建完整状态快照，并覆盖 cursor、buffer、pendingApproval、SUSPENDED。
3. executionType=REACT_AGENT。
4. nodeName 使用真实 execute_tools 节点名。
5. runId、threadId、userId 来自 RunContext。
6. 新 Checkpoint version=0。
7. status=CheckpointStatus.SUSPENDED。
8. 保存前调用 CheckpointValidator。
9. 使用 CheckpointStore.save。
10. 同一 runId 和相同 approvalId 可幂等返回已有 Checkpoint。
11. 不同 approvalId 必须抛 CHECKPOINT_CONFLICT。
12. 不实现恢复。
13. 不删除旧 Checkpoint。
14. 不记录完整 stateData、参数或审批对象。
15. 使用注入的 Clock 和 ID 生成器，不直接调用 Instant.now/UUID.randomUUID。

# 八、修正 ReAct 图挂起分支

当前 execute_tools 仍固定连接 observe，必须改成条件路由。

新增明确的同步 EdgeAction 路由，例如 ReactToolExecutionRouter：

- RunStatus.SUSPENDED → SUSPEND
- 明确框架失败 → FAILURE
- 正常执行完成 → OBSERVE

使用当前 LangGraph4j 1.8.20 的原生 edge_async(router)。

禁止 Object 包装、反射和 instanceof 分派。

DefaultReactSuspendNode：

1. 不调用模型。
2. 不调用工具。
3. 不保存、修改或删除 Checkpoint。
4. 只构造挂起 AgentResult。
5. 完成后进入 END。
6. 不进入 Observe。
7. 不进入 Failure。

# 九、挂起 AgentResult 和 API 响应

复用 AgentResult，不创建第二套结果类型。

为 RunStatus 增加或复用 SUSPENDED。

AgentResult 必须明确表达 status，并增加兼容工厂方法，保证现有 COMPLETE、FAILED 等调用点正常。

挂起结果至少包含：

- status=SUSPENDED
- content=运行已暂停，等待人工审批。
- approvalId
- operationName
- riskLevel
- requestedAt

可将审批安全信息放入不可变 metadata，但不得包含：

- Checkpoint
- stateData
- 原始参数
- sessionId
- roles
- permissions
- 完整 fingerprint

增加类似 AgentResult.suspended(...) 的工厂，禁止用普通 failure 冒充挂起。

修改正式和开发 Agent 响应 DTO，至少安全展示 status。正式 HTTP 调用遇到 SUSPENDED 正常返回 200，不映射为 500。

保持普通 COMPLETE、FAIL、MAX_ITERATIONS、认证、ACL 和 Supervisor 行为兼容。

# 十、替换错误的 Sample 危险工具

删除或停止注册当前 FileDeleteTool 和 admin_agent 的 Phase6 演示用途。

当前 FileDeleteTool 返回固定“模拟删除成功”，不符合要求，也无法验证审批前无副作用。

按 Batch2 实现：

- DemoRecordStore
- InMemoryDemoRecordStore
- ListDemoRecordsTool
- DeleteDemoRecordTool
- approval_demo_agent

DemoRecordStore：

- 初始包含 demo-1、demo-2
- 线程安全
- 不使用 static Map
- list 返回不可变快照
- delete 只删除内存演示记录
- 重复删除幂等
- 不访问文件、数据库和外部资源

list_demo_records：

- LOW
- 无副作用
- 返回真实内存记录

delete_demo_record：

- HIGH
- 参数包含 recordId、reason
- 只有 Terminal 真正执行时才调用 Store.delete
- 返回真实、结构化删除结果
- 不得返回固定成功结果

approval_demo_agent：

- allowedTools 仅包含 list_demo_records、delete_demo_record
- 使用通用 ReActEngine
- 不实现专属循环
- 不绕过 ACL 和审批
- prompt 要求根据真实 ToolResult 回答

Sample Store、工具和 Agent 必须仅在：

agent.sample.enabled=true

时注册。不得 matchIfMissing 自动开启本批危险演示工具。

ADMIN 的 tool:*:invoke 只能通过 ACL，不能绕过审批。
VISITOR 无 delete 权限时必须由 ACL 拒绝，不创建 Checkpoint。
dev 通配权限可以到达审批，但不能绕过审批。

# 十一、Spring 装配

通过现有 @Configuration/@AutoConfiguration 和 @Bean 装配：

- CheckpointStore
- CheckpointValidator
- CheckpointIdGenerator
- ApprovalIdGenerator
- SensitiveValueSanitizer
- ToolApprovalPolicy
- ToolOperationFingerprint
- ToolApprovalInterceptor
- ReactCheckpointService
- DefaultReactSuspendNode
- ToolExecutionRouter
- DemoRecordStore 和 Sample 工具

要求：

1. runtime 和 application 不加 Spring 注解。
2. 默认实现使用 @ConditionalOnMissingBean 允许替换。
3. 不创建第二个 ToolInvocationGateway。
4. 不创建第二套拦截器链。
5. ToolApprovalInterceptor 只注册一次。
6. Checkpoint 基础 Bean 在没有模型配置时也能装配。
7. Sample 关闭时不注册危险演示工具和 Agent。
8. 不在 bootstrap 主启动类中手工 new。

# 十二、编译与验证

必须依次真实执行：

mvn clean compile -DskipTests

mvn -pl agent-bootstrap -am package -DskipTests

如果失败，继续根据真实错误修复，直到两条命令都成功；不能只报告第一个错误。

环境允许时启动并验证：

1. list_demo_records：
   - 正常执行
   - 不创建 Checkpoint
   - 返回 demo-1、demo-2

2. ADMIN 删除 demo-1：
   - ACL 通过
   - 参数校验通过
   - 创建 PENDING
   - DeleteDemoRecordTool 未执行
   - 保存 Checkpoint
   - 返回 SUSPENDED
   - 不进入 Observe
   - 不再次调用模型

3. 再次 list：
   - demo-1 仍存在

4. VISITOR 删除：
   - PERMISSION_DENIED
   - 不创建审批
   - 不创建 Checkpoint
   - 失败 ToolResult 回灌模型
   - 不挂起

5. dev API：
   - ACL 通过
   - 仍需审批
   - 不自动批准

6. GET /api/framework/tools：
   - list_demo_records 显示 LOW
   - delete_demo_record 显示 HIGH
   - 不暴露实现类和内部策略

无法启动时必须说明准确环境阻塞原因，不能伪造验证结果。

# 十三、范围控制

禁止实现：

- approve API
- reject API
- resume API
- 待审批查询 API
- 审批 Controller
- Web 审批页面
- Supervisor Checkpoint 或恢复
- 自动恢复
- 数据库、Redis、TTL、分布式锁
- 真实文件删除、邮件、转账
- ADMIN 自动批准
- 一次批准多个 ToolCall
- Fake Model 或固定工具成功结果
- 新测试脚本和 README
- Git 操作

不要修改 .kscc-prompts 中的提示词文件。
不要提交 .kscc-prompts。
如需处理忽略规则，只在 .gitignore 增加：

.kscc-prompts/

不得执行 git rm、git commit 或其他 Git 写操作。

# 十四、最终输出

最终只报告：

1. 新增和修改文件。
2. 当前编译错误如何修复。
3. ApprovalDecision、InterruptPayload、PendingApproval 最终结构。
4. AgentCheckpoint 最终字段和约束。
5. CheckpointStore 最终接口。
6. InMemoryCheckpointStore 并发与索引策略。
7. version 条件更新语义。
8. safeArguments 脱敏规则。
9. ToolApprovalPolicy 规则。
10. ToolApprovalInterceptor 完整流程。
11. 实际拦截器顺序。
12. AgentInterruptSignal 传播路径。
13. cursor 和 buffer 语义。
14. ReactCheckpointService 保存内容。
15. execute_tools 后的实际条件路由。
16. SUSPENDED AgentResult 和 HTTP 响应。
17. Sample 工具和 Agent。
18. ADMIN、VISITOR、dev 行为差异。
19. 两条 Maven 命令的真实结果。
20. 启动和接口验证结果。
21. 未完成验证及准确原因。
22. 发现但未处理的非本批问题。

不得把计划、预期结果或未执行验证描述成已经完成。