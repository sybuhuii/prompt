你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须先完整阅读并遵守根目录`AGENTS.md`。其中已有模块边界、LangGraph4j、工具网关、安全、编译和输出规范，本提示词不重复展开。

已完成第六阶段前两批：

1.HITL审批领域模型。
2.AgentCheckpoint、CheckpointStatus及CheckpointStore。
3.InMemoryCheckpointStore和version条件更新。
4.CheckpointValidator及参数脱敏。
5.ToolRiskLevel和ToolApprovalPolicy。
6.ToolApprovalInterceptor。
7.AgentInterruptSignal。
8.危险工具执行前中断。
9.ToolExecution cursor和buffer。
10.ReAct SUSPEND路由及挂起AgentResult。
11.approval_demo_agent及演示危险工具。
12.节点重跑恢复语义。

当前只执行第六阶段第3批：

1.实现人工批准和拒绝的应用服务。
2.安全更新Checkpoint中的PendingApproval。
3.实现Checkpoint恢复抢占，防止并发重复恢复。
4.从Checkpoint重建ReactAgentState。
5.从中断的execute_tools节点重新执行。
6.批准后真实执行危险工具。
7.拒绝后生成失败ToolResult并回灌模型。
8.一次运行遇到多个危险工具时支持再次挂起。
9.运行完成或失败后处理Checkpoint终态。
10.验证重复审批、重复恢复和用户隔离。

本批不实现HTTP Controller，不实现Web审批页面，不实现Supervisor恢复，不实现跨进程持久化。

# 一、执行前检查

先检查：

1.根目录`AGENTS.md`。
2.第六阶段前两批全部实际代码。
3.AgentCheckpoint及其version/status字段。
4.CheckpointStore真实接口。
5.InMemoryCheckpointStore条件更新实现。
6.PendingApproval、ApprovalStatus、ApprovalDecision。
7.InterruptPayload及operationFingerprint。
8.ToolInvocation中的approval字段。
9.ToolApprovalInterceptor各状态分支。
10.ToolOperationFingerprint。
11.ReactCheckpointService。
12.ReactAgentState及全部StateKeys。
13.DefaultReactToolExecutionNode。
14.ReactAgentGraphFactory。
15.ReactAgentEngine。
16.DefaultReactSuspendNode。
17.AgentResult及AuthenticatedAgentRunResult。
18.UserSession和RunContext。
19.CheckpointValidator。
20.AgentErrorCode及异常映射。
21.DemoRecordStore和delete_demo_record。
22.现有Application Service和Spring装配方式。
23.当前LangGraph4j版本是否支持指定节点恢复或Command resume。

要求：

1.以当前真实接口和依赖版本为准。
2.优先复用已有类型，不创建第二套Checkpoint、Approval或React引擎。
3.前序实现有阻断问题时只做最小修复。
4.不得重写完整ReAct图。
5.不得修改认证、用户管理、ACL和Supervisor主链。
6.不得提前实现第4批HTTP和前端功能。

# 二、固定恢复语义

继续采用“节点重跑”：

1.Checkpoint保存于危险工具真正执行之前。
2.Checkpoint中的恢复节点为`execute_tools`。
3.恢复时不重新进入本轮Reason。
4.恢复时不重新让模型生成ToolCall。
5.恢复时保留原pendingToolCalls。
6.恢复时保留cursor。
7.恢复时保留已经完成的executionBuffer。
8.恢复时将人工决定注入当前ToolInvocation。
9.批准后从当前危险ToolCall继续执行。
10.拒绝后将当前ToolCall转成失败ToolResult。
11.处理完当前ToolCall后继续处理后续ToolCall。
12.本轮全部工具完成后进入Observe。
13.Observe回灌工具结果后再进入下一轮Reason。
14.不得从Java方法某一行继续。
15.不得重新执行cursor之前已完成的工具。
16.不得重新调用产生当前ToolCall的Reason模型。

# 三、审批动作模型

检查是否已有等价类型。

如不存在，在agent-application或core合适位置增加：

`ApprovalAction`

只包含：

- APPROVE
- REJECT

要求：

1.不要与ApprovalStatus重复表达PENDING。
2.客户端未来只能提交APPROVE或REJECT。
3.不能提交APPROVED、REJECTED之外的任意字符串。
4.不得提供AUTO_APPROVE。
5.不得提供ADMIN_OVERRIDE。
6.不得提供模型审批动作。

增加不可变命令模型：

`ApprovalDecisionCommand`

至少包含：

- runId
- approvalId
- action
- comment

要求：

1.不包含userId、roles、permissions。
2.当前用户身份由已验证UserSession提供。
3.comment允许为空。
4.comment设置合理长度上限。
5.不得包含HTTP或Servlet类型。
6.不得包含完整Checkpoint。
7.不得允许客户端提交decidedAt和decidedBy。

# 四、审批结果模型

增加或复用不可变结果：

`ApprovalDecisionResult`

至少表达：

- runId
- threadId
- approvalId
- status
- operationName
- decidedBy
- comment
- decidedAt
- checkpointVersion

要求：

1.不得返回stateData。
2.不得返回Session ID。
3.不得返回原始工具参数。
4.safeArguments留给后续审批详情接口。
5.不得返回完整UserSession或RunContext。
6.不得返回完整operationFingerprint。
7.不得返回密码、密钥和权限集合。

# 五、ApprovalDecisionService

在agent-application实现纯Java：

`ApprovalDecisionService`

依赖：

- CheckpointStore
- CheckpointValidator
- Clock

建议方法：

`ApprovalDecisionResult decide(UserSession operator,ApprovalDecisionCommand command);`

职责：

1.校验operator来自有效认证流程。
2.校验runId、approvalId和action。
3.通过runId加载Checkpoint。
4.不存在时抛CHECKPOINT_NOT_FOUND。
5.校验Checkpoint.userId等于operator.userId。
6.本批不允许管理员跨用户审批。
7.校验Checkpoint.status=SUSPENDED。
8.校验PendingApproval存在。
9.校验approvalId完全匹配。
10.校验当前审批状态。
11.创建ApprovalDecision。
12.更新PendingApproval。
13.更新Checkpoint.updatedAt。
14.Checkpoint.version加1。
15.使用`updateIfVersionMatches`原子提交。
16.版本冲突时抛CHECKPOINT_CONFLICT。
17.不得直接执行恢复。
18.不得调用模型或工具。
19.不得删除Checkpoint。
20.不得修改Store中已保存对象。

## PENDING状态

允许首次APPROVE或REJECT。

`decidedBy`必须来自：

`operator.userId`

不得来自请求Body。

`decidedAt`必须来自注入的Clock。

## 已APPROVED状态

再次提交相同APPROVE时：

1.视为幂等。
2.返回已有决定。
3.不得增加version。
4.不得修改decidedAt和comment。

再次提交REJECT时：

1.抛APPROVAL_ALREADY_DECIDED。
2.不得覆盖原决定。

## 已REJECTED状态

再次提交相同REJECT时幂等返回。

再次提交APPROVE时抛APPROVAL_ALREADY_DECIDED。

## Checkpoint状态非法

以下情况不得审批：

- RESUMING
- COMPLETED
- FAILED

使用现有明确错误码，不得返回INTERNAL_ERROR。

# 六、恢复申请模型

在agent-application增加或复用：

`ReactResumeResult`

至少包含：

- runId
- threadId
- agentName
- status
- content
- approvalId
- errorCode
- metadata

优先复用现有`AuthenticatedAgentRunResult`。

不要仅为名称一致复制AgentResult字段。

要求：

1.恢复继续使用原runId。
2.恢复继续使用原threadId。
3.不得生成新的顶层runId。
4.响应不得包含Session ID和Checkpoint stateData。
5.再次挂起时返回新的approvalId。
6.完成时返回正常最终AgentResult。
7.失败时使用现有结构化错误。

# 七、ReactCheckpointStateMapper

检查ReactCheckpointService当前如何保存stateData。

如果存在分散的强转和字符串Key，实现纯Java：

`ReactCheckpointStateMapper`

职责：

1.从ReactAgentState生成不可变stateData快照。
2.从AgentCheckpoint重建ReactAgentState。
3.集中处理类型校验。
4.集中处理缺失字段。
5.不得依赖Spring。
6.不得调用模型或工具。
7.不得访问CheckpointStore。
8.不得生成新的runId或threadId。

恢复至少需要：

- AgentTask
- AgentDefinition或其稳定标识
- RunContext
- messages
- pendingToolCalls
- latestToolResults
- executionBuffer
- executionCursor
- pendingApproval
- iteration
- maxIterations
- currentStatus
- checkpointId
- finalResult
- 其他当前图必须字段

恢复覆盖规则：

1.使用Checkpoint中的stateData重建。
2.pendingApproval替换为最新已决策版本。
3.currentStatus改为RUNNING或项目现有恢复状态。
4.finalResult清空。
5.checkpointId保持当前Checkpoint ID。
6.pendingToolCalls保持不变。
7.executionCursor保持不变。
8.executionBuffer保持不变。
9.messages保持中断时快照。
10.iteration保持不变。
11.不得重新执行Reason。
12.不得把Checkpoint状态直接作为可变Map交给图。

如果现有State构造器要求Map：

1.传入独立不可变快照。
2.图内部根据LangGraph4j机制产生新状态。
3.不得修改Store内的stateData。

# 八、恢复前完整校验

在runtime实现或扩展：

`ReactResumeValidator`

至少校验：

1.executionType=REACT_AGENT。
2.Checkpoint.status=SUSPENDED。
3.PendingApproval存在。
4.PendingApproval状态为APPROVED或REJECTED。
5.decision存在。
6.resumeNode或nodeName为execute_tools。
7.runId、threadId和userId有效。
8.stateData有效。
9.pendingToolCalls非空。
10.cursor处于合法范围。
11.cursor对应ToolCall ID等于InterruptPayload.toolCallId。
12.cursor对应toolName等于operationName。
13.重新计算operationFingerprint并完全匹配。
14.RunContext.userId等于Checkpoint.userId。
15.RunContext.runId和threadId与Checkpoint一致。
16.审批decidedBy非空。
17.Checkpoint未处于COMPLETED或FAILED。
18.不得信任stateData中与顶层Checkpoint冲突的身份字段。

校验失败：

1.不得抢占Checkpoint。
2.不得执行模型。
3.不得执行工具。
4.抛明确框架异常。

# 九、Checkpoint恢复抢占

防止两个请求同时恢复同一运行。

实现纯Java：

`CheckpointResumeCoordinator`

依赖：

- CheckpointStore
- CheckpointValidator
- ReactResumeValidator
- Clock

职责：

`AgentCheckpoint claimForResume(String runId,String userId);`

流程：

1.load当前Checkpoint。
2.校验归属用户。
3.校验SUSPENDED。
4.校验审批已有最终决定。
5.执行完整恢复前校验。
6.创建status=RESUMING的新Checkpoint。
7.version加1。
8.updatedAt使用Clock。
9.通过`updateIfVersionMatches`原子更新。
10.更新成功返回RESUMING Checkpoint。
11.失败抛CHECKPOINT_CONFLICT。
12.不得循环无限重试。
13.不得无条件覆盖Checkpoint。
14.不得执行图。
15.不得删除Checkpoint。

并发语义：

1.只有一个请求能够从SUSPENDED切换到RESUMING。
2.第二个请求看到RESUMING时立即拒绝。
3.不得让两个请求真实执行同一个危险工具。
4.不得通过JVM全局锁串行所有runId。
5.不同runId允许并发恢复。

# 十、React恢复图入口

先检查当前LangGraph4j版本的真实能力。

优先顺序：

1.若当前版本原生支持从指定节点携带状态恢复，直接使用原生API。
2.若没有可靠原生入口，在现有`ReactAgentGraphFactory`中增加最小的显式恢复构图能力。
3.不得猜测不存在的resume API。
4.不得添加`Object`包装器或反射。

可接受的回退设计：

`compileForResume(String resumeNodeName)`

规则：

1.只允许白名单节点`execute_tools`。
2.START直接进入execute_tools。
3.后续边、条件路由和节点与正常图保持一致。
4.不得复制一整套节点实现。
5.不得重新进入reason。
6.不得根据任意客户端nodeName构建图。
7.不支持任意未知节点恢复。
8.恢复节点必须来自Checkpoint并通过白名单校验。

正常`compile()`行为保持不变。

# 十一、ReactResumeEngine

在agent-runtime实现纯Java：

`ReactResumeEngine`

依赖建议：

- CheckpointResumeCoordinator
- ReactCheckpointStateMapper
- ReactAgentGraphFactory
- ReactCheckpointLifecycleService
- 现有节点或编译图依赖

建议方法：

`AgentResult resume(String runId,String userId);`

如现有架构更适合传入已抢占Checkpoint，可调整方法，但应用层不得直接操作CompiledGraph。

恢复流程：

1.通过CheckpointResumeCoordinator抢占。
2.取得RESUMING Checkpoint。
3.从Checkpoint重建ReactAgentState。
4.将已决策PendingApproval放入State。
5.从execute_tools节点启动恢复图。
6.ToolExecutionNode读取cursor。
7.只为cursor对应ToolCall传入PendingApproval。
8.调用ToolInvocationGateway。
9.批准时真实执行工具。
10.拒绝时得到失败ToolResult。
11.完成当前ToolCall后清空PendingApproval。
12.cursor前进。
13.继续处理后续ToolCall。
14.全部工具完成进入Observe。
15.模型根据工具结果继续Reason。
16.最终返回AgentResult。
17.保持原runId和threadId。
18.不得生成新的登录Session。
19.不得重新调用首次Reason。
20.不得自动提升权限。

# 十二、ToolExecutionNode恢复行为

检查并完善`DefaultReactToolExecutionNode`。

普通首次执行：

- approval为空。

恢复执行：

1.读取State中的PendingApproval。
2.仅当当前cursor ToolCall与审批绑定完全匹配时传入ToolInvocation。
3.不得把同一审批传给其他ToolCall。
4.ToolApprovalInterceptor再次验证fingerprint。
5.APPROVED时调用真实工具。
6.REJECTED时返回失败ToolResult。
7.取得ToolResult后加入executionBuffer。
8.cursor加1。
9.清空pendingApproval。
10.如果下一个ToolCall为危险工具：
- 创建新的PendingApproval。
- 再次中断。
- 不重复执行之前工具。
  11.所有ToolCall完成后将buffer写入latestToolResults。
  12.清空buffer。
  13.cursor重置0。
  14.进入Observe。

批准执行成功后，不得继续保留旧approval用于下一轮工具调用。

# 十三、多次挂起

同一runId可能在恢复后再次遇到危险工具。

修改或完善`ReactCheckpointService.suspend`：

## 首次挂起

不存在Checkpoint：

1.创建version=0。
2.status=SUSPENDED。
3.save。

## 恢复后再次挂起

当前Store中存在：

- 同一runId
- status=RESUMING
- version与恢复抢占版本一致

处理：

1.使用新PendingApproval。
2.保存最新stateData、cursor和buffer。
3.status改为SUSPENDED。
4.version加1。
5.使用条件更新。
6.不得创建新的顶层runId。
7.不得删除旧Checkpoint后重新save。
8.不得复用旧approvalId。
9.不同危险ToolCall必须拥有不同approvalId。
10.版本冲突抛CHECKPOINT_CONFLICT。

禁止：

1.RESUMING之外的Checkpoint被任意覆盖。
2.旧SUSPENDED审批被新审批静默覆盖。
3.不同runId相互替换。

# 十四、Checkpoint生命周期服务

在agent-runtime实现：

`ReactCheckpointLifecycleService`

依赖：

- CheckpointStore
- Clock

职责至少包含：

- complete(runId,expectedVersion)
- fail(runId,expectedVersion,errorCode)
- removeTerminal(runId)

建议语义：

## 恢复运行成功完成

1.将Checkpoint状态从RESUMING更新为COMPLETED。
2.version加1。
3.条件更新。
4.随后删除Checkpoint。
5.删除不存在时幂等。
6.最终AgentResult不因清理日志失败而伪造失败。
7.若状态更新发生冲突，记录安全错误并返回CHECKPOINT_CONFLICT，不得静默删除他人新Checkpoint。

## 恢复运行再次挂起

1.由ReactCheckpointService更新为SUSPENDED。
2.生命周期服务不得删除。
3.返回新的挂起AgentResult。

## 恢复运行框架失败

1.将Checkpoint更新为FAILED。
2.保存安全errorCode，不保存异常对象或堆栈。
3.默认保留FAILED Checkpoint，供诊断。
4.本批不实现失败Checkpoint查询API和清理任务。
5.FAILED状态不能再次resume。

## 普通工具失败

如果工具以`ToolResult.success=false`返回，并被模型正常处理，不应把Checkpoint标记FAILED。

# 十五、工具执行幂等

框架本批必须提供两层防护：

第一层：Checkpoint恢复抢占

- 同一runId只有一个恢复请求进入执行。

第二层：危险工具自身幂等要求

- 节点重跑无法完全消除“工具产生副作用后进程立即崩溃”的风险。
- 危险工具应使用operationFingerprint或业务幂等键防止重复副作用。

对演示`DeleteDemoRecordTool`：

1.使用recordId删除保持幂等。
2.记录不存在时返回“已经不存在”，不抛系统异常。
3.重复删除不产生额外副作用。
4.结果metadata可表达alreadyAbsent。
5.不得为了幂等在工具中绕过审批。
6.不得使用approvalId作为授权凭证。

如当前ToolInvocation能够安全传递idempotencyKey，可将operationFingerprint作为只读幂等键传给工具。

若修改工具接口会造成大规模破坏，则不要强行修改；在相关接口注释中明确危险业务工具需使用业务幂等键。

不要在本批引入分布式幂等数据库。

# 十六、ApprovalResumeApplicationService

在agent-application实现纯Java：

`ApprovalResumeApplicationService`

依赖：

- ApprovalDecisionService
- ReactResumeEngine

至少提供：

`ApprovalDecisionResult decide(UserSession operator,ApprovalDecisionCommand command);`

`ReactResumeResult resume(UserSession operator,String runId);`

可增加便捷方法：

`ReactResumeResult decideAndResume(UserSession operator,ApprovalDecisionCommand command);`

要求：

1.operator必须来自认证入口。
2.不得从command读取userId。
3.decide只记录决定，不执行工具。
4.resume只恢复已决定的Checkpoint。
5.decideAndResume先决定再恢复。
6.相同决定重复提交应保持幂等。
7.决定成功但恢复发生版本冲突时返回明确冲突。
8.不得自动重试危险工具。
9.不得访问HttpServletRequest。
10.不得依赖Spring Web。
11.不得直接调用模型或ToolRegistry。
12.不得返回完整Checkpoint。

第4批Controller应只依赖该Application Service和查询服务。

# 十七、Spring装配

在agent-infrastructure装配：

- ApprovalDecisionService
- ReactCheckpointStateMapper
- ReactResumeValidator
- CheckpointResumeCoordinator
- ReactCheckpointLifecycleService
- ReactResumeEngine
- ApprovalResumeApplicationService

要求：

1.runtime/application实现保持纯Java。
2.通过`@Configuration`和`@Bean`装配。
3.复用现有CheckpointStore、Clock、GraphFactory和节点Bean。
4.不得创建第二套React节点。
5.不得创建第二个ToolInvocationGateway。
6.不得产生Bean循环依赖。
7.无模型配置时审批决定服务应能装配。
8.无模型能力时实际resume返回明确MODEL_NOT_AVAILABLE或现有等价错误。
9.不得创建Fake模型。
10.本批不新增Controller。

# 十八、审批和恢复错误码

检查并复用现有错误码。

缺失时最小补充：

- APPROVAL_NOT_FOUND
- APPROVAL_ALREADY_DECIDED
- INVALID_APPROVAL_DECISION
- CHECKPOINT_NOT_FOUND
- CHECKPOINT_CONFLICT
- CHECKPOINT_NOT_RESUMABLE
- RUN_ALREADY_RESUMING
- RESUME_FAILED

不要增加语义重复错误码。

语义：

1.不存在Checkpoint不能伪装成Session错误。
2.用户不属于Checkpoint时，使用PERMISSION_DENIED或安全的NOT_FOUND语义。
3.审批冲突不是INTERNAL_ERROR。
4.重复恢复不是模型错误。
5.恢复后的工具普通失败仍走ToolResult。
6.审批拒绝不是系统异常。
7.审批拒绝结果应回灌模型。

# 十九、多用户隔离

必须保证：

1.审批用户只能操作自己的Checkpoint。
2.判断依据为operator.userId和Checkpoint.userId。
3.不得根据username判断归属。
4.不得通过approvalId单独加载并执行。
5.恢复必须同时使用runId和当前用户身份。
6.Checkpoint中的RunContext.userId必须与顶层userId一致。
7.不同用户相同threadId也不得串读。
8.管理员跨用户审批本批不实现。
9.模型不能生成ApprovalDecision。
10.客户端不能提交decidedBy、decidedAt和operationFingerprint。
11.不得使用ThreadLocal保存审批人或恢复状态。
12.不得记录sessionId、完整参数和stateData。

# 二十、验证场景

在不新增HTTP接口的前提下，使用现有应用层调用能力、已有调试入口或最小启动验证完成能够执行的场景。

不得为验证新增隐藏Controller。

## 场景1：批准后恢复

1.ADMIN调用approval_demo_agent删除demo-1。
2.首次结果为SUSPENDED。
3.Checkpoint状态为SUSPENDED、审批为PENDING。
4.调用ApprovalDecisionService提交APPROVE。
5.Checkpoint审批变成APPROVED。
6.调用ReactResumeEngine或Application Service恢复。
7.Checkpoint原子切换为RESUMING。
8.从execute_tools开始。
9.不重新调用首次Reason。
10.delete_demo_record真实执行一次。
11.进入Observe和后续Reason。
12.最终返回完成结果。
13.demo-1已不存在。
14.Checkpoint完成后被清理。

## 场景2：拒绝后恢复

1.重新创建或使用demo-2。
2.触发删除审批。
3.提交REJECT。
4.恢复执行。
5.ToolApprovalInterceptor不调用TerminalToolExecutor。
6.生成APPROVAL_REJECTED失败ToolResult。
7.Observe生成error=true的ToolAgentMessage。
8.模型说明操作被拒绝。
9.demo-2仍存在。
10.Checkpoint完成后清理。

## 场景3：重复相同决定

对PENDING审批提交APPROVE两次。

预期：

1.第一次成功。
2.第二次幂等返回已有决定。
3.version不重复增加。
4.decidedAt不改变。

## 场景4：冲突决定

先APPROVE，再REJECT。

预期：

1.第二次返回APPROVAL_ALREADY_DECIDED。
2.原APPROVE不被覆盖。

## 场景5：并发或重复恢复

同一runId连续调用resume两次。

预期：

1.只有一个请求从SUSPENDED切换RESUMING。
2.另一个得到RUN_ALREADY_RESUMING或CHECKPOINT_CONFLICT。
3.危险工具最多执行一次。

## 场景6：错误用户恢复

visitor尝试恢复admin的runId。

预期：

1.拒绝。
2.不改变Checkpoint版本。
3.不执行工具。
4.不泄漏stateData和审批细节。

## 场景7：多危险工具

同一次模型响应包含两个顺序危险ToolCall时：

1.第一个产生审批并挂起。
2.批准恢复后只执行第一个。
3.处理到第二个时再次挂起。
4.第二个生成新approvalId。
5.第一个不得重复执行。
6.cursor和buffer保持正确。

若当前模型难以稳定生成两个ToolCall，可完成代码级检查并准确说明未进行真实模型验证，不得伪造。

## 场景8：指纹不匹配

模拟或代码审查确认：

1.ToolCall ID改变。
2.toolName改变。
3.arguments改变。

任一变化均不得执行工具，应返回INVALID_APPROVAL_DECISION或CHECKPOINT_NOT_RESUMABLE。

# 二十一、本批禁止实现

禁止：

1.审批Controller。
2.approve/reject HTTP接口。
3.resume HTTP接口。
4.待审批列表API。
5.Web审批页面。
6.Supervisor审批和恢复。
7.任意节点恢复。
8.跨进程恢复。
9.数据库或Redis Checkpoint。
10.分布式锁。
11.自动批准ADMIN。
12.模型自行批准。
13.管理员跨用户审批。
14.Checkpoint TTL和定时清理。
15.SSE。
16.真实邮件、转账或数据删除。
17.新增隐藏调试接口。
18.测试脚本、README和Git操作。

# 二十二、编译验收

执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

确认：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.ApprovalDecisionService保持纯Java。
4.ReactResumeEngine保持纯Java。
5.恢复使用原runId和threadId。
6.恢复不重新进入首次Reason。
7.Checkpoint抢占使用version条件更新。
8.并发恢复只有一个成功。
9.批准后真实工具只执行一次。
10.拒绝后工具不执行。
11.拒绝结果进入Observe。
12.恢复后再次危险操作能够重新挂起。
13.cursor之前的工具不重复执行。
14.完成后Checkpoint被清理。
15.失败Checkpoint不会被再次恢复。
16.不同用户不能操作对方Checkpoint。
17.没有新增HTTP接口和前端。
18.没有修改Supervisor恢复流程。
19.现有认证、ACL、普通ReAct和Supervisor未被破坏。
20.git diff没有无关修改。

# 二十三、最终输出

只输出：

1.新增和修改文件清单。
2.ApprovalDecisionService状态转换规则。
3.重复和冲突决定处理。
4.ReactCheckpointStateMapper恢复字段。
5.ReactResumeValidator校验顺序。
6.Checkpoint恢复抢占流程。
7.React恢复图入口实现方式。
8.ReactResumeEngine完整流程。
9.ToolExecutionNode恢复时approval注入方式。
10.多危险工具再次挂起流程。
11.Checkpoint完成、失败和清理策略。
12.工具幂等保障及剩余边界。
13.用户隔离规则。
14.编译和打包结果。
15.实际完成的批准、拒绝和恢复验证。
16.因模型或环境限制未完成的验证及原因。
17.发现但未处理的问题。

编译或打包失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE6_BATCH3_END===