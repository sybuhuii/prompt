你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须先完整阅读并遵守根目录`AGENTS.md`。其中已有模块边界、LangGraph4j、工具网关、安全、编译和输出规范，本提示词不重复展开。

已完成第六阶段第1批：

1.HITL审批领域模型。
2.AgentCheckpoint及CheckpointStatus。
3.CheckpointStore抽象。
4.InMemoryCheckpointStore。
5.Checkpoint版本条件更新。
6.CheckpointValidator。
7.safeArguments脱敏能力。
8.节点重跑恢复语义。

当前只执行第六阶段第2批：

1.为工具定义增加或复用风险级别。
2.实现危险工具审批策略。
3.将审批闸门接入ToolInvocationGateway。
4.危险工具执行前产生统一中断信号。
5.ReAct捕获中断并保存完整Checkpoint。
6.ReAct进入SUSPENDED终止状态。
7.增加安全的危险工具演示Agent。
8.验证工具没有在审批前真实执行。

本批不实现批准、拒绝、恢复API，不实现前端审批页面，不实现Supervisor挂起恢复。

# 一、执行前检查

先检查：

1.根目录`AGENTS.md`。
2.第1批实际生成的审批和Checkpoint模型。
3.AgentCheckpoint字段及CheckpointStore真实方法。
4.CheckpointValidator和ID生成器。
5.InterruptPayload、PendingApproval、ApprovalDecision。
6.ToolDefinition、ToolCall、ToolInvocation、ToolResult。
7.ToolRegistry和ToolInvocationGateway。
8.现有工具拦截器接口及链推进方式。
9.ToolExceptionHandlingInterceptor。
10.ToolAuditInterceptor。
11.ToolAccessControlInterceptor。
12.ToolArgumentValidationInterceptor。
13.TerminalToolExecutor。
14.DefaultReactToolExecutionNode。
15.DefaultReactObserveNode。
16.ReactAgentState及StateKeys。
17.ReactAgentGraphFactory和路由实现。
18.ReactNodeNames、ReactRoute、RunStatus。
19.AgentResult和ReactAgentEngine。
20.Sample Agent及内置工具注册方式。
21.正式和开发Agent接口响应结构。
22.当前ToolExecutionNode是否支持一次返回多个ToolCall。

要求：

1.以现有真实代码为准增量实现。
2.优先复用第1批类型，不创建第二套审批或Checkpoint模型。
3.已有风险等级或requiresApproval字段时直接复用。
4.前序实现存在阻断问题时只做最小修复并说明。
5.不得重构认证、ACL、Supervisor和模型调用主链。
6.不得提前实现第3批恢复流程。

# 二、本批固定语义

本批采用以下语义：

1.审批中断发生在危险工具真实执行之前。
2.无权限工具先由ACL拒绝，不创建审批。
3.参数非法的危险工具不创建审批。
4.参数合法且需要审批时产生PendingApproval。
5.真实工具不得执行。
6.ReAct保存Checkpoint并进入SUSPENDED。
7.本次HTTP调用正常返回挂起结果，不返回500。
8.当前runId保持不变，供后续恢复。
9.恢复采用节点重跑，不从Java方法某一行继续。
10.恢复时ToolExecutionNode根据保存的执行游标继续。
11.本批尚无恢复入口，因此Checkpoint只能查询Store，不能继续执行。
12.一个运行同时只处理一个待审批工具。
13.同一次模型响应有多个危险ToolCall时，逐个审批，不批量批准。

# 三、工具风险级别

检查是否已有`ToolRiskLevel`。

若不存在，在合适的core工具领域包中增加：

- SAFE
- SENSITIVE
- DANGEROUS

要求：

1.默认普通工具为SAFE。
2.本批默认只有DANGEROUS必须人工审批。
3.SENSITIVE保留扩展语义，本批默认不强制审批。
4.不得通过工具名称字符串判断风险。
5.不得通过Agent名称判断风险。
6.不得通过用户角色判断风险。
7.风险等级属于ToolDefinition稳定元数据。
8.已有工具构造方式尽量保持兼容。
9.如果ToolDefinition是record且增加字段会影响调用点，应统一修复现有调用点或增加兼容工厂方法。
10.不得同时保留`riskLevel`和语义重复的第二套布尔字段。

更新框架工具查询结果，使`GET /api/framework/tools`可安全展示riskLevel。

不得展示工具实现类或敏感配置。

# 四、审批策略

在agent-runtime实现纯Java接口：

`ToolApprovalPolicy`

建议方法：

`ToolApprovalRequirement evaluate(ToolDefinition definition,ToolCall call,RunContext context);`

实现不可变：

`ToolApprovalRequirement`

至少表达：

- required
- riskLevel
- reasonCode
- displayReason

实现：

`DefaultToolApprovalPolicy`

规则：

1.SAFE返回不需要审批。
2.SENSITIVE本批返回不需要审批。
3.DANGEROUS返回需要审批。
4.不得根据ADMIN角色绕过。
5.不得根据`tool:*:invoke`绕过。
6.ACL授权和人工审批是两层独立约束。
7.即使用户拥有工具调用权限，危险工具仍需审批。
8.RunContext缺失时失败关闭。
9.ToolDefinition缺失时沿用现有工具不存在错误。
10.displayReason必须适合前端展示。
11.不得包含完整工具参数、权限集合、Session ID或模型思维过程。
12.策略保持无状态、线程安全且不依赖Spring。

# 五、审批与工具调用绑定

审批必须精确绑定本次ToolCall，防止批准一个操作后执行另一个操作。

检查`InterruptPayload`，缺失时最小增加：

- toolCallId
- operationFingerprint

`operationFingerprint`应根据以下信息生成：

- runId
- toolCallId
- toolName
- 原始arguments

实现纯Java：

`ToolOperationFingerprint`

要求：

1.使用JDK SHA-256。
2.不得引入额外加密依赖。
3.输入顺序固定。
4.不得把原始arguments写入日志。
5.不得把fingerprint作为身份认证凭证。
6.fingerprint用于确认恢复时工具调用未被替换。
7.同一保存的ToolCall应生成相同fingerprint。
8.toolName、toolCallId或参数改变时fingerprint必须改变。
9.不得在fingerprint中加入随机值。
10.不得把完整fingerprint发送给LLM。

# 六、ToolInvocation审批上下文

检查现有`ToolInvocation`。

最小扩展，使其可携带：

`Optional<PendingApproval> approval`

或当前代码风格下的等价可选字段。

要求：

1.首次普通工具调用approval为空。
2.本批中危险工具首次调用approval为空。
3.第3批恢复时再从Checkpoint传入已决策PendingApproval。
4.不得从HTTP请求Body直接构造approval。
5.不得由模型生成approval。
6.不得把approval放进RunContext。
7.不得创建ToolInvocationV2。
8.修改所有现有构造点并保持默认行为兼容。
9.普通工具和现有开发接口不受影响。

# 七、统一中断信号

在agent-runtime实现：

`AgentInterruptSignal extends RuntimeException`

至少携带：

- PendingApproval pendingApproval

要求：

1.它是运行时控制信号，不是普通工具失败。
2.不得携带ReactAgentState。
3.不得携带Spring或Servlet对象。
4.不得携带完整UserSession。
5.不得包含密码、密钥和Session ID。
6.message只使用安全中断说明。
7.ToolApprovalInterceptor产生信号。
8.ToolExecutionNode捕获信号。
9.其他普通异常不得伪装成中断。
10.日志不得以系统故障ERROR级别记录正常中断。

该信号未来可扩展到节点中断，但本批只实现工具审批中断。

# 八、ToolApprovalInterceptor

在现有工具拦截器包中实现：

`ToolApprovalInterceptor`

依赖：

- ToolRegistry
- ToolApprovalPolicy
- ApprovalIdGenerator
- SensitiveValueSanitizer
- ToolOperationFingerprint
- Clock

执行流程：

1.读取ToolInvocation、ToolCall和RunContext。
2.通过ToolRegistry取得真实ToolDefinition。
3.调用ToolApprovalPolicy。
4.不需要审批时继续next。
5.需要审批且approval为空时：
- 生成approvalId。
- 计算operationFingerprint。
- 对工具参数生成safeArguments。
- 构造InterruptPayload。
- 构造PENDING状态的PendingApproval。
- 抛出AgentInterruptSignal。
  6.不得调用next。
  7.不得执行TerminalToolExecutor。
  8.不得构造伪造的成功ToolResult。

同时预留并实现已存在approval时的确定语义：

## approval=PENDING

1.校验审批与当前runId、userId、toolCallId、toolName和fingerprint匹配。
2.匹配后重新抛出同一个PendingApproval的中断信号。
3.不得生成新的approvalId。
4.不执行工具。

## approval=APPROVED

1.严格校验绑定信息。
2.校验通过后调用next。
3.不得再次中断。
4.不得因为用户是ADMIN跳过绑定校验。

## approval=REJECTED

1.严格校验绑定信息。
2.不得调用next。
3.返回`ToolResult.success=false`。
4.errorCode使用`APPROVAL_REJECTED`或已有等价错误码。
5.content使用安全说明：
`人工审批已拒绝该工具操作，工具未执行。`
6.后续由Observe回灌模型。

## approval非法或不匹配

1.抛`INVALID_APPROVAL_DECISION`或已有等价框架异常。
2.不得执行工具。
3.不得自动创建新审批覆盖旧审批。

本批不会从外部产生APPROVED或REJECTED，但这些分支应实现完整，供第3批直接复用。

# 九、拦截器顺序

目标治理语义：

`异常治理→审计→ACL→参数校验→人工审批→真实执行`

要求：

1.ACL拒绝时不创建审批。
2.参数非法时不创建审批。
3.只有授权且参数合法的危险工具才创建审批。
4.审批中断必须被审计为SUSPENDED或APPROVAL_REQUIRED。
5.审批中断不得被异常拦截器转换成普通失败ToolResult。
6.ToolExceptionHandlingInterceptor捕获`AgentInterruptSignal`时必须原样重新抛出。
7.ToolAuditInterceptor捕获中断时记录安全状态后原样抛出。
8.不得把中断记录成工具执行成功。
9.不得注册两次ToolApprovalInterceptor。
10.不得依赖Spring Bean偶然顺序。
11.结合现有拦截器链推进方向确定真实注册顺序。
12.最终输出说明实际顺序。

# 十、ToolExecution执行游标

为支持节点重跑且不重复执行审批前已经完成的安全工具，需要在React状态中增加或复用：

- TOOL_EXECUTION_CURSOR
- TOOL_EXECUTION_BUFFER
- PENDING_APPROVAL
- CHECKPOINT_ID

语义：

`TOOL_EXECUTION_CURSOR`

- 当前准备执行的ToolCall下标。
- 初始为0。
- 每成功完成一个工具后加1。
- 在危险工具中断时保持为该危险工具下标。
- 使用覆盖Channel。

`TOOL_EXECUTION_BUFFER`

- 保存本轮已经完成但尚未Observe的ToolResult。
- 初始为空。
- 中断时随Checkpoint保存。
- 恢复后继续累积。
- 全部工具完成后一次性交给现有latestToolResults。
- 使用覆盖Channel，禁止Appender导致重复。

`PENDING_APPROVAL`

- 当前唯一待审批项。
- 普通运行时为空。
- 中断时保存PendingApproval。
- 使用覆盖Channel。

`CHECKPOINT_ID`

- 当前挂起Checkpoint标识。
- 普通运行时为空。
- 不发送给LLM。

要求：

1.使用统一StateKeys和类型安全访问器。
2.不得在ReactAgentState重复声明普通Java字段。
3.不得散落字符串Key。
4.不得使用ThreadLocal保存游标。
5.不得把游标保存在ToolApprovalInterceptor实例字段。
6.本批先完成状态保存；第3批实现恢复读取。

# 十一、改造ToolExecutionNode

最小改造现有`DefaultReactToolExecutionNode`。

正常执行流程：

1.读取pendingToolCalls。
2.读取cursor，默认0。
3.读取executionBuffer，默认空。
4.从cursor开始顺序执行ToolCall。
5.为每个调用创建ToolInvocation。
6.首次运行approval为空。
7.调用ToolInvocationGateway。
8.正常取得ToolResult后加入buffer。
9.cursor前进到下一项。
10.全部完成后：
- 将完整buffer写入现有latestToolResults。
- 清空executionBuffer。
- cursor重置为0。
- 清空pendingApproval和checkpointId。
- 进入Observe。

捕获`AgentInterruptSignal`时：

1.不得把它转换成ToolResult。
2.不得继续执行后续ToolCall。
3.不得调用Observe。
4.保持cursor指向当前危险ToolCall。
5.保留已完成工具的executionBuffer。
6.调用ReactCheckpointService保存Checkpoint。
7.将pendingApproval、checkpointId和RunStatus.SUSPENDED写入State。
8.返回状态增量，使图进入Suspend节点。
9.不增加iteration。
10.不删除pendingToolCalls。
11.不修改原ToolCall。
12.Checkpoint保存失败时进入框架失败路径，不得伪装为已挂起。

除AgentInterruptSignal外，普通工具失败继续沿用已有ToolResult语义。

# 十二、ReactCheckpointService

在agent-runtime实现纯Java：

`ReactCheckpointService`

依赖：

- CheckpointStore
- CheckpointIdGenerator
- CheckpointValidator
- Clock

建议职责：

`AgentCheckpoint suspend(ReactAgentState state,String nodeName,PendingApproval approval,int cursor,List<ToolResult> buffer);`

要求：

1.构造完整且不可变的React状态快照。
2.应用cursor、buffer、pendingApproval和SUSPENDED状态覆盖值。
3.不得直接保存可变ReactAgentState实例。
4.不得保存CompiledGraph、Gateway、Registry或Spring Bean。
5.不得保存模型客户端和异常对象。
6.保存executionType=REACT_AGENT。
7.nodeName使用真实工具执行节点名。
8.runId、threadId和userId从现有State/RunContext取得。
9.新Checkpoint version=0。
10.状态为SUSPENDED。
11.调用CheckpointValidator。
12.通过CheckpointStore.save保存。
13.保存后返回Checkpoint。
14.不得直接执行恢复。
15.不得删除旧Checkpoint。
16.同一runId已经存在挂起Checkpoint时：
- 相同approvalId可幂等返回已有Checkpoint。
- 不同approvalId必须报CHECKPOINT_CONFLICT。
  17.不得在日志打印完整stateData或工具参数。

# 十三、ReAct图挂起分支

扩展现有图定义，新增或复用：

- ReactNodeNames.SUSPEND
- ReactRoute.SUSPEND
- DefaultReactSuspendNode

将`execute_tools`后的固定边改为条件路由：

- RunStatus.SUSPENDED→suspend
- 明确框架失败→failure
- 正常执行完成→observe

要求：

1.不要使用`Object`包装路由。
2.使用项目当前LangGraph4j原生EdgeAction/AsyncEdgeAction方式。
3.suspend节点不调用模型。
4.suspend节点不调用工具。
5.suspend节点不修改Checkpoint。
6.suspend节点构造明确的挂起AgentResult。
7.suspend后进入END。
8.不得路由到failure。
9.不得路由到observe。
10.普通工具链路保持原样。

# 十四、挂起AgentResult

优先复用现有AgentResult和RunStatus。

挂起结果至少表达：

- status=SUSPENDED
- runId
- threadId
- content=`运行已暂停，等待人工审批。`
- approvalId
- operationName
- riskLevel
- requestedAt

要求：

1.不得返回完整Checkpoint。
2.不得返回stateData。
3.不得返回原始工具参数。
4.可返回已脱敏safeArguments，但若现有响应不适合则留到审批查询API。
5.不得返回Session ID。
6.不得返回完整roles和permissions。
7.不得使用INTERNAL_ERROR表示挂起。
8.HTTP层应正常返回该结果，通常为200。
9.现有Agent接口响应DTO缺少status时做最小扩展。
10.开发Agent接口也应能看到挂起状态。
11.不得为挂起结果创建另一套AgentResult类型。

# 十五、Engine行为

检查`ReactAgentEngine`。

必须保证：

1.图进入SUSPEND后能读取挂起finalResult。
2.不会把SUSPENDED转换成FAILED。
3.不会因为Checkpoint存在而自动恢复。
4.不会删除Checkpoint。
5.不会继续执行后续节点。
6.不会再次调用模型。
7.不会再次执行危险工具。
8.普通COMPLETE、FAIL和MAX_ITERATIONS行为不变。
9.无挂起时现有Agent调用完全兼容。

# 十六、安全演示工具

增加仅用于Sample环境的内存演示能力。

推荐实现：

- `DemoRecordStore`
- `InMemoryDemoRecordStore`
- `ListDemoRecordsTool`
- `DeleteDemoRecordTool`
- `approval_demo_agent`

## DemoRecordStore

初始可包含：

- demo-1
- demo-2

要求：

1.仅保存非敏感演示数据。
2.线程安全。
3.不使用static Map。
4.list返回不可变快照。
5.delete按recordId删除。
6.重复删除应幂等，返回已不存在状态。
7.不得连接真实数据库。
8.不得删除真实文件或外部资源。

## list_demo_records

- riskLevel=SAFE。
- 返回当前演示记录。
- 无副作用。

## delete_demo_record

参数至少包含：

- recordId
- reason

要求：

1.riskLevel=DANGEROUS。
2.描述明确标注需要人工审批。
3.真实执行时才调用DemoRecordStore.delete。
4.审批前不得产生任何删除副作用。
5.返回结构化删除结果。
6.重复执行不得产生异常副作用。
7.不得删除项目文件、数据库表或用户数据。

## approval_demo_agent

允许工具：

- list_demo_records
- delete_demo_record

要求：

1.仅在`agent.sample.enabled=true`时注册。
2.使用现有通用ReAct引擎。
3.不得实现专属循环。
4.不得绕过ACL。
5.不得绕过ToolApprovalInterceptor。
6.不得在System Prompt中声称工具已经执行。
7.提示模型根据真实ToolResult回答。

ADMIN通过`tool:*:invoke`可到达审批环节。

VISITOR若没有`tool:delete_demo_record:invoke`，应先被ACL拒绝，不产生审批。

开发API使用现有显式dev通配权限，可到达审批环节，但不得绕过审批。

# 十七、Spring装配

在现有工具和React配置中装配：

- ToolApprovalPolicy
- ToolOperationFingerprint
- ToolApprovalInterceptor
- ReactCheckpointService
- DefaultReactSuspendNode
- DemoRecordStore及Sample工具

要求：

1.runtime实现不添加Spring注解。
2.使用现有@Configuration和@Bean。
3.默认策略允许后续自定义Bean替换。
4.不得创建第二个ToolInvocationGateway。
5.不得创建第二套拦截器链。
6.不得重复注册ToolApprovalInterceptor。
7.无模型配置时Bean装配仍应成功。
8.Sample关闭时不注册演示Agent和演示工具。
9.CheckpointStore继续复用第1批默认实现。
10.不得创建Fake恢复服务。

# 十八、多用户与安全

1.Checkpoint.userId来自当前RunContext。
2.审批payload.userId来自当前RunContext。
3.客户端不能提交approvalId或PendingApproval触发自动批准。
4.模型不能输出批准决定。
5.ADMIN权限不能绕过人工审批。
6.VISITOR ACL拒绝时不得创建Checkpoint。
7.不同用户的Checkpoint不得混用。
8.operationFingerprint不匹配时不得执行工具。
9.审批前工具参数只以脱敏形式对外展示。
10.不得记录Session ID、完整参数、stateData或审批完整对象。
11.中断属于控制状态，不记录为系统异常。
12.身份信息不得发送给LLM。

# 十九、本批禁止实现

禁止：

1.approve接口。
2.reject接口。
3.resume接口。
4.待审批列表API。
5.审批Controller。
6.Web审批页面。
7.Supervisor Checkpoint。
8.Supervisor挂起恢复。
9.节点级自定义审批演示。
10.数据库或Redis Checkpoint。
11.跨进程恢复。
12.分布式锁。
13.Checkpoint TTL。
14.SSE。
15.真实邮件、转账、删除业务。
16.自动批准ADMIN操作。
17.一次批准多个危险ToolCall。
18.测试脚本、README和Git操作。

# 二十、运行验收

执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

环境允许时启动并验证：

## 场景1：安全工具

调用`approval_demo_agent`要求使用`list_demo_records`。

预期：

1.正常执行。
2.不创建Checkpoint。
3.不进入SUSPENDED。
4.返回demo-1和demo-2。

## 场景2：ADMIN危险工具

使用ADMIN正式Session调用`approval_demo_agent`：

`请使用delete_demo_record删除demo-1，原因为HITL演示。`

预期：

1.ACL通过。
2.参数校验通过。
3.ToolApprovalInterceptor创建PendingApproval。
4.DeleteDemoRecordTool未执行。
5.Checkpoint保存成功。
6.AgentResult.status=SUSPENDED。
7.返回runId和approvalId。
8.不进入Observe。
9.不再次调用模型。

## 场景3：确认无副作用

再次调用安全的`list_demo_records`。

预期：

1.demo-1仍存在。
2.证明审批前未执行删除。

## 场景4：VISITOR危险工具

使用VISITOR Session请求删除demo-1。

预期：

1.ACL返回PERMISSION_DENIED。
2.不创建PendingApproval。
3.不创建Checkpoint。
4.失败ToolResult按现有流程回灌模型。
5.不进入SUSPENDED。

## 场景5：开发API

`agent.dev-api.enabled=true`时使用开发Agent接口执行危险请求。

预期：

1.dev权限通过ACL。
2.仍进入人工审批。
3.不得因为dev-user绕过审批。
4.返回SUSPENDED。

## 场景6：工具查询

`GET /api/framework/tools`

预期：

1.list_demo_records显示SAFE。
2.delete_demo_record显示DANGEROUS。
3.不暴露实现类和内部审批策略。

# 二十一、编译检查

确认：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.风险等级位于稳定领域模型。
4.ToolApprovalPolicy保持纯Java。
5.ToolApprovalInterceptor位于统一工具链。
6.拦截顺序符合ACL→参数校验→审批→执行。
7.AgentInterruptSignal未被异常治理转换。
8.ToolExecutionNode保存cursor和buffer。
9.中断Checkpoint保存完整状态快照。
10.危险工具审批前没有执行。
11.SUSPENDED不被当作FAILED。
12.普通工具、认证、ACL和Supervisor功能未被破坏。
13.没有批准、拒绝和恢复接口。
14.没有前端和范围外修改。
15.git diff无无关修改。

# 二十二、最终输出

只输出：

1.新增和修改文件清单。
2.ToolRiskLevel最终设计。
3.ToolApprovalPolicy规则。
4.ToolApprovalInterceptor处理流程。
5.审批与ToolCall绑定方式。
6.实际工具拦截器顺序。
7.AgentInterruptSignal传播路径。
8.ToolExecution cursor和buffer语义。
9.ReactCheckpointService保存内容。
10.ReAct新增SUSPEND路由。
11.挂起AgentResult格式。
12.Sample危险工具和Agent。
13.审批前无副作用验证结果。
14.ADMIN、VISITOR和dev权限差异。
15.编译、打包和实际运行验证。
16.未完成验证及准确原因。
17.发现但未处理的问题。

编译或打包失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE6_BATCH2_END===