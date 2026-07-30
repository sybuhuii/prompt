你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须完整阅读并遵守根目录`AGENTS.md`。

当前已经完成：

1.身份、Session、RBAC、工具ACL和多用户隔离。
2.单Agent ReAct执行引擎。
3.Supervisor中心路由与子Agent ReAct执行。
4.HITL中断、审批、Checkpoint保存和ReAct恢复。
5.上下文消息数裁剪、Token裁剪和LLM摘要。
6.ContextWindowSnapshot及ReAct、Supervisor上下文接入。
7.长期记忆MemoryStore和InMemoryMemoryStore。
8.CheckpointPurpose区分HITL_RECOVERY与THREAD_MEMORY。
9.CheckpointStore按userId、threadId和purpose查询。
10.ThreadConversationState和ThreadConversationCheckpointService。
11.ThreadExecutionCoordinator和ThreadExecutionLease。
12.ReactThreadConversationStateMapper。
13.ReactAgentEngine.executeThread。
14.正式Agent API支持可选threadId。
15.同threadId的ReAct多轮续接。
16.失败、挂起不覆盖上一轮稳定THREAD_MEMORY。

当前只执行第八阶段第4批：

1.实现SupervisorAgentState与ThreadConversationState之间的映射。
2.扩展SupervisorEngine支持新线程和已有线程执行。
3.正式Supervisor API支持可选threadId。
4.保存和加载Supervisor父线程的THREAD_MEMORY。
5.保证每次Supervisor调用生成新runId但保持原threadId。
6.保证Supervisor成员子Agent每次仍使用fresh context。
7.扩展ReAct HITL恢复结果，使其能够提取稳定ThreadConversationState。
8.HITL批准或拒绝恢复完成后保存最新THREAD_MEMORY。
9.HITL恢复过程中使用ThreadExecutionCoordinator防止并发普通请求。
10.明确THREAD_MEMORY保存与HITL_RECOVERY清理顺序。
11.完成Supervisor续接和HITL线程同步的后端联调。

本批不修改前端，不实现长期记忆注入，不自动提取用户偏好，不实现Supervisor的中断恢复。

# 一、强制边界

必须遵守：

1.不得重新实现CheckpointStore。
2.不得重新实现InMemoryCheckpointStore。
3.不得新增ConversationStore。
4.不得新增ShortTermMemoryStore。
5.不得使用MemoryStore保存Supervisor或ReAct会话状态。
6.不得使用CheckpointStore保存长期用户偏好。
7.不得覆盖SupervisorAgentState中的完整messages。
8.不得把ContextWindowSnapshot.windowMessages作为完整历史保存。
9.不得改变Supervisor中心路由模式。
10.不得让Supervisor成员子Agent继承父线程完整历史。
11.不得让子Agent继承父ContextWindowSnapshot。
12.不得实现Supervisor interrupt/resume。
13.不得修改工具ACL和HITL审批指纹语义。
14.不得修改危险工具本身。
15.不得修改前端。
16.不得实现线程列表、详情和删除API。
17.不得让客户端提交runId、userId、Checkpoint或消息历史。
18.不得在未读完提示词前修改代码。
19.不得仅以编译通过宣称完成。
20.不得提前实现第八阶段长期记忆运行链集成。

# 二、执行前检查

必须检查真实代码：

1.`SupervisorEngine`当前公开方法。
2.`SupervisorEngine`如何创建初始SupervisorAgentState。
3.`SupervisorGraphFactory`和CompiledGraph执行方式。
4.`SupervisorAgentState`及全部StateKey。
5.Supervisor初始System消息创建位置。
6.当前User消息加入位置。
7.Supervisor iteration、decision、dispatch、aggregate和finalResult字段。
8.子Agent调用及fresh context创建逻辑。
9.`DefaultSupervisorReasonNode`上下文窗口接入。
10.`ContextWindowSnapshot`和`ContextProcessingTrace`的StateKey。
11.Supervisor Complete、MaxIterations和Failure节点。
12.Supervisor最终结果类型。
13.`AuthenticatedSupervisorApplicationService`。
14.正式Supervisor Controller。
15.正式Supervisor请求和响应DTO。
16.`ThreadConversationState`。
17.`ThreadConversationCheckpointService`。
18.`ThreadExecutionCoordinator`。
19.`ThreadExecutionLease`。
20.`ThreadIdGenerator`和`ThreadIdValidator`。
21.`ThreadExecutionOutcome`。
22.`ReactThreadConversationStateMapper`。
23.`ReactThreadPersistencePolicy`或现有集中判断逻辑。
24.`ReactResumeEngine`。
25.`ApprovalDecisionService`。
26.`ApprovalResumeApplicationService`。
27.`CheckpointResumeCoordinator`。
28.`ReactCheckpointStateMapper`。
29.`ReactCheckpointLifecycleService`。
30.HITL恢复成功、再次挂起和失败的现有处理顺序。
31.现有错误码和HTTP异常映射。
32.现有Spring Bean配置。
33.现有`GET /api/framework/memory`查询结果。

以下类型必须真实存在：

- `ThreadConversationState`
- `ThreadConversationCheckpointService`
- `ThreadExecutionCoordinator`
- `ThreadExecutionLease`
- `ThreadExecutionOutcome`
- `ReactThreadConversationStateMapper`
- `SupervisorEngine`
- `SupervisorAgentState`
- `ReactResumeEngine`
- `ApprovalResumeApplicationService`

任一关键类型不存在时立即停止。

不得为了继续执行而临时创建第二套近似类型。

# 三、固定标识和状态语义

必须保持：

```text
sessionId：认证当前调用者
threadId：一段持续多轮会话
runId：一次具体执行
checkpointId：一份状态快照
```

CheckpointPurpose继续保持：

```text
HITL_RECOVERY：恢复某一次被中断的run
THREAD_MEMORY：续接一段已稳定完成的thread
```

规则：

1.Supervisor同threadId的每次调用使用新runId。
2.HITL恢复继续使用原被中断runId。
3.HITL恢复完成后生成的THREAD_MEMORY记录，其lastCompletedRunId仍是原被中断runId。
4.THREAD_MEMORY状态必须为COMPLETED。
5.THREAD_MEMORY不得包含PendingApproval。
6.HITL_RECOVERY不得作为下一轮正常对话历史直接加载。
7.正常线程续接只加载THREAD_MEMORY。
8.审批恢复只加载指定runId的HITL_RECOVERY。
9.两种Checkpoint可以在同一threadId下短暂同时存在。
10.完成同步后清理目标HITL_RECOVERY，但不得删除THREAD_MEMORY。

# 四、SupervisorThreadConversationStateMapper

在`agent-runtime`必须新增纯Java：

`SupervisorThreadConversationStateMapper`

职责：

1.构建新Supervisor线程的初始State。
2.根据已有ThreadConversationState构建续接State。
3.从稳定完成的SupervisorAgentState提取ThreadConversationState。
4.清理上一轮Supervisor临时运行字段。
5.不得访问CheckpointStore。
6.不得执行模型。
7.不得调用子Agent。
8.不得依赖Spring。

必须提供或实现等价方法：

```java
SupervisorAgentState createInitialState(
    SupervisorDefinition definition,
    AgentTask task,
    RunContext runContext
);

SupervisorAgentState createContinuedState(
    SupervisorDefinition definition,
    AgentTask task,
    RunContext runContext,
    ThreadConversationState previousState
);

ThreadConversationState extractStableState(
    String supervisorName,
    String runId,
    SupervisorAgentState finalState,
    Instant updatedAt
);
```

不得把初始化和提取逻辑散落到Controller及Application Service。

# 五、新Supervisor线程初始化

`createInitialState`必须保持现有SupervisorEngine行为。

至少包括：

1.创建Supervisor规定的System消息。
2.追加当前User消息。
3.设置当前RunContext。
4.设置当前SupervisorDefinition或必要标识。
5.iteration初始化为0。
6.status初始化为运行状态。
7.currentDecision为空。
8.dispatchTarget为空。
9.subAgentResults为空。
10.aggregateBuffer为空。
11.finalResult为空。
12.failure信息为空。
13.ContextWindowSnapshot为空。
14.latestContextTrace为空。
15.使用当前runId和threadId。
16.不得访问MemoryStore。
17.不得读取旧Checkpoint。
18.不得提前创建子Agent状态。

如果当前Supervisor还有其他必要StateKey，必须完整保留初始化语义。

# 六、续接Supervisor线程初始化

`createContinuedState`必须执行：

1.校验previousState.executionType为SUPERVISOR。
2.校验previousState.participantName等于当前supervisorName。
3.复制上一轮完整messages。
4.不得修改previousState.messages。
5.不得重复插入System消息。
6.追加当前一条User消息。
7.设置新的RunContext。
8.使用新的runId。
9.保持原threadId。
10.使用新的taskId。
11.iteration重置为0。
12.status重置为运行状态。
13.currentDecision清空。
14.dispatchTarget清空。
15.subAgentResults清空。
16.aggregateBuffer清空。
17.currentSubAgent清空。
18.pendingDispatch清空。
19.finalResult清空。
20.failure信息清空。
21.恢复previousState.contextWindowSnapshot。
22.恢复previousState.latestContextTrace。
23.不得恢复上一轮成员子Agent的ReactAgentState。
24.不得恢复上一轮子Agent的pendingToolCalls。
25.不得恢复上一轮子Agent的ContextWindowSnapshot。
26.不得恢复上一轮RunContext。
27.不得恢复上一轮Session ID。
28.当前用户身份和权限必须来自当前已验证Session。
29.不得用旧状态权限覆盖当前Session。
30.不得重新执行上一轮子Agent调用。

实际StateKey名称以仓库为准，但清理语义必须完整。

# 七、Supervisor System和User消息规则

## 新线程

1.System消息按当前SupervisorDefinition生成。
2.只插入一次。
3.当前User消息只追加一次。

## 续接线程

1.不得重新插入System消息。
2.保留历史中的原System消息。
3.追加当前User消息。
4.不得在Application Service和Mapper各追加一次。
5.必须明确只有一个位置负责追加User消息。
6.推荐由SupervisorThreadConversationStateMapper负责。

禁止：

1.客户端提交role。
2.客户端提交System消息。
3.客户端提交Assistant消息。
4.客户端提交子Agent观察消息。
5.客户端提交Tool消息。
6.根据消息content判断角色。

# 八、Supervisor稳定状态提取

`extractStableState`必须提取：

- executionType=SUPERVISOR
- participantName=supervisorName
- 完整messages
- contextWindowSnapshot
- latestContextTrace
- lastCompletedRunId
- updatedAt

要求：

1.messages必须来自Supervisor完整State。
2.不得使用ContextWindowSnapshot.windowMessages代替完整历史。
3.不得保存currentDecision。
4.不得保存dispatchTarget。
5.不得保存subAgentResults临时集合。
6.不得保存aggregateBuffer。
7.不得保存当前子Agent运行State。
8.不得保存RunContext。
9.不得保存Session。
10.不得保存permissions。
11.不得保存failure异常。
12.不得保存finalResult对象。
13.不得保存CompiledGraph。
14.不得保存Spring AI Message。
15.不得保存MemoryEntry。
16.消息历史必须合法。
17.不得存在未完成的父级路由状态。
18.提取结果必须不可变。

父消息历史中已经作为普通观察或Assistant消息写入的子Agent业务结果可以保留。

不得把子Agent内部完整State额外塞入ThreadConversationState。

# 九、SupervisorThreadPersistencePolicy

在`agent-runtime`必须新增或复用集中策略：

`SupervisorThreadPersistencePolicy`

建议方法：

```java
boolean isPersistable(
    SupervisorResult result,
    SupervisorAgentState finalState
);
```

如实际结果类型不同，使用真实类型。

允许持久化：

1.正常COMPLETE。
2.MaxIterations产生稳定最终结果，并且：
- 没有未完成dispatch；
- 没有正在运行的子Agent；
- 消息历史完整；
- 没有失败状态。

禁止持久化：

1.FAILURE。
2.模型调用异常。
3.子Agent尚未返回。
4.仍存在pendingDispatch。
5.父状态不完整。
6.状态映射失败。
7.子Agent返回SUSPENDED而父流程未形成稳定终态。
8.任何需要未来Supervisor恢复的中间状态。

要求：

1.失败轮不得覆盖旧稳定THREAD_MEMORY。
2.不能仅根据结果content非空判断。
3.不能仅根据HTTP 200判断。
4.判断逻辑必须集中且确定。
5.不得实现Supervisor中断恢复。

# 十、扩展SupervisorEngine

必须最小扩展现有：

`SupervisorEngine`

新增方法，名称固定：

```java
ThreadExecutionOutcome executeThread(
    SupervisorDefinition definition,
    AgentTask task,
    RunContext runContext,
    Optional<ThreadConversationState> previousState
);
```

如果`ThreadExecutionOutcome.result`当前只接受`AgentResult`，必须先检查Supervisor结果是否已经统一为AgentResult。

处理规则：

1.已有统一AgentResult：
- 直接复用ThreadExecutionOutcome。
  2.Supervisor结果不是AgentResult：
- 不得通过Object强塞；
- 可以最小泛型化现有ThreadExecutionOutcome；
- 或新增名称明确的`SupervisorThreadExecutionOutcome`。
  3.只有在类型系统无法安全复用时才允许新增：
  `SupervisorThreadExecutionOutcome`
  4.不得使用Map或Object表示结果。

执行流程：

1.校验definition、task和runContext。
2.previousState为空时创建新SupervisorAgentState。
3.previousState存在时创建续接State。
4.执行现有Supervisor图。
5.取得最终SupervisorAgentState。
6.取得现有Supervisor结果。
7.调用SupervisorThreadPersistencePolicy。
8.稳定时提取ThreadConversationState。
9.不稳定时conversationState为空。
10.返回线程执行结果。

要求：

1.不得在SupervisorEngine中访问CheckpointStore。
2.不得在Engine中保存THREAD_MEMORY。
3.不得生成threadId。
4.不得生成runId。
5.不得修改Supervisor路由图。
6.不得修改SupervisorDecision解析。
7.不得修改成员Agent白名单。
8.不得修改子Agent执行顺序。
9.不得把previousState传给子Agent。
10.不得改变子Agentfresh context语义。
11.保留原有无状态execute方法。
12.旧execute方法可以委托executeThread并传Optional.empty。
13.不得造成execute与executeThread互相递归。
14.同一Engine Bean必须支持并发。

# 十一、子Agent fresh context强制规则

Supervisor线程续接时，父线程拥有历史，但成员子Agent仍必须fresh。

每次dispatch：

1.创建新的子runId。
2.创建新的子threadId或现有派生执行标识。
3.创建新的子AgentTask。
4.创建新的子RunContext。
5.调用ReactAgentEngine无历史执行入口，或：
`executeThread(...,Optional.empty())`
6.不得加载父threadId的THREAD_MEMORY。
7.不得加载子Agent上一次运行的THREAD_MEMORY。
8.不得向子Agent传递父ContextWindowSnapshot。
9.不得向子Agent传递父完整messages。
10.只传递Supervisor当前明确分派的任务内容。
11.子Agent完成后只将业务结果按现有方式回灌父State。
12.不得将子AgentThreadConversationState保存为父线程状态。
13.不得自动保存Supervisor内部临时子Agent线程。
14.不得将父threadId直接作为子Agent正常续接threadId。

必须搜索并确认不存在：

```text
supervisor previousState → child React previousState
```

# 十二、正式Supervisor应用服务

必须修改现有：

`AuthenticatedSupervisorApplicationService`

支持可选threadId。

推荐方法：

```java
AuthenticatedSupervisorRunResult invoke(
    UserSession session,
    String supervisorName,
    String message,
    Optional<String> requestedThreadId
);
```

旧方法必须保持兼容并委托新方法。

不得复制两套执行逻辑。

# 十三、正式Supervisor调用流程

必须按以下顺序：

## 基础校验

1.校验UserSession。
2.校验supervisorName。
3.校验message。
4.查询SupervisorDefinition。
5.不得从请求读取userId和permissions。

## 确定threadId

threadId为空：

1.服务器生成threadId。
2.校验生成结果。
3.previousState为空。

threadId存在：

1.trim。
2.ThreadIdValidator校验。
3.不得自动创建客户端指定的未知threadId。

## 获取Lease

1.使用userId+threadId调用ThreadExecutionCoordinator.acquire。
2.使用try-with-resources。
3.Lease覆盖检查、加载、执行和保存。
4.同一线程并发返回THREAD_BUSY。

## 检查和加载

1.检查同一userId+threadId是否存在活跃HITL_RECOVERY。
2.存在时返回THREAD_SUSPENDED。
3.调用ThreadConversationCheckpointService.load：
- executionType=SUPERVISOR；
- participantName=supervisorName。
  4.请求携带threadId但load为空时抛THREAD_NOT_FOUND。
  5.participant不匹配时抛THREAD_PARTICIPANT_MISMATCH。
  6.不得泄漏其他用户线程。

## 本次运行

1.生成新runId。
2.生成新taskId。
3.构造AgentTask。
4.构造当前RunContext。
5.身份和权限来自当前Session。
6.调用SupervisorEngine.executeThread。
7.不得调用旧无状态execute入口。

## 保存

1.线程执行结果包含conversationState时保存THREAD_MEMORY。
2.使用当前userId、threadId、runId。
3.失败或不稳定时不保存。
4.不得删除旧稳定状态。
5.不得将子Agent状态保存为父线程Checkpoint。

## 返回

至少返回：

- runId
- threadId
- 原有Supervisor结果

不得返回：

- ThreadConversationState
- 完整messages
- Checkpoint
- 子Agent内部State
- ContextWindowSnapshot
- UserSession
- permissions

# 十四、正式Supervisor DTO

必须最小扩展正式Supervisor请求DTO：

```java
String threadId
```

请求示例：

```json
{
  "supervisorName": "general_supervisor",
  "message": "继续分析刚才的问题",
  "threadId": "可选"
}
```

要求：

1.threadId可选。
2.null或空白表示新线程。
3.非空表示续接。
4.不得增加runId。
5.不得增加userId。
6.不得增加messages。
7.不得增加Checkpoint。
8.不得增加子Agent状态。
9.旧请求保持兼容。
10.Controller不得直接访问CheckpointStore。

响应必须包含：

- runId
- threadId
- 原有Supervisor安全结果字段

不得返回Checkpoint ID和完整历史。

# 十五、Supervisor错误映射

继续复用：

- INVALID_THREAD_ID
- THREAD_NOT_FOUND
- THREAD_BUSY
- THREAD_SUSPENDED
- THREAD_PARTICIPANT_MISMATCH
- THREAD_CHECKPOINT_INVALID

HTTP映射保持：

- INVALID_THREAD_ID → 400
- THREAD_NOT_FOUND → 404
- THREAD_PARTICIPANT_MISMATCH → 安全404
- THREAD_BUSY → 409
- THREAD_SUSPENDED → 409
- THREAD_CHECKPOINT_INVALID → 500

要求：

1.不得增加Supervisor专属同义错误码。
2.不得泄漏其他用户线程。
3.不得返回父线程消息数量作为404详情。
4.不得返回子Agent名称列表作为错误详情。
5.继续复用统一异常处理器。

# 十六、ReAct HITL恢复线程结果

必须最小扩展：

`ReactResumeEngine`

目标：

恢复执行完成后能够返回：

- AgentResult
- 可选ThreadConversationState

优先复用：

`ThreadExecutionOutcome`

必须新增或保留兼容方法，例如：

```java
ThreadExecutionOutcome resumeThread(
    AgentCheckpoint checkpoint,
    ApprovalDecision decision,
    RunContext runContext
);
```

原有返回AgentResult的方法必须保持兼容。

执行流程：

1.校验checkpoint.purpose=HITL_RECOVERY。
2.恢复ReactAgentState。
3.从原中断节点继续现有恢复图。
4.取得最终ReactAgentState。
5.取得AgentResult。
6.复用ReactThreadPersistencePolicy判断是否稳定。
7.稳定时使用ReactThreadConversationStateMapper提取状态。
8.不稳定时conversationState为空。
9.返回ThreadExecutionOutcome。

要求：

1.不得重新执行Reason作为恢复起点，除非现有节点重跑语义明确要求。
2.保持现有execute_tools恢复节点。
3.不得修改批准/拒绝注入规则。
4.不得修改操作指纹校验。
5.不得修改工具幂等性语义。
6.再次SUSPENDED时conversationState为空。
7.FAILED时conversationState为空。
8.批准后正常完成可以生成稳定状态。
9.拒绝后模型形成最终答复并正常完成，也可以生成稳定状态。
10.不得在ReactResumeEngine中直接保存THREAD_MEMORY。
11.不得访问MemoryStore。
12.不得生成新的runId。
13.恢复继续使用原runId和threadId。

# 十七、ApprovalResumeApplicationService并发保护

必须修改现有：

`ApprovalResumeApplicationService`

或实际负责`decideAndResume`的应用服务。

流程必须调整为：

1.根据当前已认证UserSession查找待审批Checkpoint。
2.验证Checkpoint属于当前userId。
3.读取threadId。
4.使用ThreadExecutionCoordinator.acquire(userId,threadId)。
5.在Lease内完成审批决定和恢复。
6.不得在Lease外先将Checkpoint永久修改为RESUMING后再等待。
7.继续使用现有CheckpointResumeCoordinator抢占version。
8.同一审批多标签页仍只能一个恢复成功。
9.普通Agent请求与审批恢复不能同时运行同一线程。
10.异常路径必须释放Lease。

要求：

1.Lease Key必须为Checkpoint.userId+threadId。
2.不得信任请求体中的userId。
3.不得信任请求体中的threadId。
4.审批请求仍以runId和approvalId定位。
5.不得改变approve/reject幂等语义。
6.不得改变版本冲突409语义。
7.不得把THREAD_MEMORY加入待审批列表。

# 十八、HITL恢复完成后的THREAD_MEMORY保存

恢复得到ThreadExecutionOutcome后：

## conversationState存在

必须：

1.确认result为稳定终态。
2.调用ThreadConversationCheckpointService.save：
- userId=Checkpoint.userId；
- threadId=Checkpoint.threadId；
- runId=原HITL runId；
- state=outcome.conversationState。
  3.保存新的THREAD_MEMORY成功后，才清理对应HITL_RECOVERY。
  4.不得先删除HITL_RECOVERY再保存THREAD_MEMORY。
  5.不得删除同threadId旧THREAD_MEMORY，交由Service现有最新记录策略处理。
  6.不得保存到MemoryStore。
  7.最终API返回原AgentResult。

## conversationState为空且再次SUSPENDED

必须：

1.保留或更新新的HITL_RECOVERY。
2.不得保存THREAD_MEMORY。
3.不得清理新的待审批Checkpoint。
4.同一threadId继续禁止普通消息。

## conversationState为空且FAILED

必须：

1.不得覆盖旧THREAD_MEMORY。
2.按现有生命周期将HITL_RECOVERY标记FAILED或保留诊断。
3.不得宣称线程已经成功续接。
4.不得删除旧稳定线程状态。

# 十九、保存和清理原子顺序

单机内采用以下顺序：

```text
获取ThreadExecutionLease
→审批决定及Checkpoint版本抢占
→恢复ReAct
→得到稳定ThreadConversationState
→保存新THREAD_MEMORY
→确认保存成功
→清理目标HITL_RECOVERY
→释放Lease
```

要求：

1.THREAD_MEMORY保存失败时，不得删除HITL_RECOVERY。
2.保存失败时返回明确框架错误。
3.不得返回恢复成功但线程状态未保存的假成功。
4.重复请求依赖现有审批版本和状态返回409。
5.不得删除其他runId的HITL_RECOVERY。
6.不得删除其他purpose的Checkpoint。
7.不得删除其他用户Checkpoint。
8.不得使用数据库事务，本批仅保证进程内顺序。
9.最终输出必须明确该过程不是分布式事务。
10.不得添加静默补偿线程。

# 二十、HITL拒绝语义

拒绝危险工具后：

1.匹配的ToolInvocation携带REJECTED决策。
2.工具不得实际执行。
3.生成结构化APPROVAL_REJECTED ToolResult。
4.ToolResult回灌Observe。
5.ReAct可继续Reason并生成最终答复。
6.最终完成状态包含拒绝结果对应的完整消息。
7.该完整历史可以保存为THREAD_MEMORY。
8.下一轮同threadId能够看到用户拒绝了该操作。
9.不得把拒绝当成执行失败直接丢弃会话状态。
10.只有恢复最终进入FAILED时才不保存稳定状态。

不得修改第六阶段既有拒绝工具语义，只补充线程状态同步。

# 二十一、HITL批准语义

批准危险工具后：

1.校验Approval ID、ToolCall ID、toolName和fingerprint。
2.只允许匹配的当前ToolCall执行。
3.工具执行一次。
4.ToolResult进入Observe。
5.ReAct继续完成。
6.完整历史包含审批后工具结果。
7.稳定完成后保存THREAD_MEMORY。
8.下一轮同threadId可以基于工具结果继续。
9.不得重复执行已批准工具。
10.不得因THREAD_MEMORY保存而再次调用工具。

# 二十二、ContextWindowSnapshot同步

HITL恢复完成后保存的ThreadConversationState必须包含：

- 完整messages
- 最新ContextWindowSnapshot
- 最新ContextProcessingTrace

规则：

1.挂起前Snapshot已由HITL Checkpoint恢复。
2.恢复执行到Observe后新增ToolResult。
3.再次Reason时ContextWindowManager只吸收新增消息。
4.生成更新后的Snapshot。
5.稳定完成后提取更新Snapshot。
6.保存为THREAD_MEMORY。
7.下一次普通请求加载该Snapshot。
8.不得重新摘要挂起前已消费的旧历史。
9.不得把windowMessages当作完整messages。
10.拒绝路径同样保存最新Snapshot。
11.再次SUSPENDED时只保存HITL恢复状态，不保存稳定线程Snapshot。
12.不得把SummaryAgentMessage写入长期MemoryStore。

# 二十三、Supervisor ContextWindowSnapshot续接

Supervisor续接时：

1.ThreadConversationState恢复父ContextWindowSnapshot。
2.追加当前User消息。
3.Supervisor Reason只吸收新增消息。
4.不得重新摘要父线程旧历史。
5.子Agent不继承父Snapshot。
6.子Agent各自从空Snapshot开始。
7.子Agent结果回灌父消息后，由父Snapshot后续吸收。
8.父完整messages持续保存。
9.父THREAD_MEMORY保存最新Snapshot和Trace。
10.不同Supervisor threadId不得共享Snapshot。

# 二十四、Supervisor与Agent线程类型隔离

必须保证：

1.RE_ACT_AGENT线程不能由Supervisor续接。
2.SUPERVISOR线程不能由普通Agent续接。
3.相同participantName但executionType不同仍拒绝。
4.同一Supervisor线程不能切换supervisorName。
5.同一Agent线程不能切换agentName。
6.类型或名称不匹配抛THREAD_PARTICIPANT_MISMATCH。
7.不得自动转换线程类型。
8.不得重新生成System消息伪装成新类型。
9.不得删除原线程状态。
10.安全HTTP响应不得说明原线程实际类型和名称。

# 二十五、框架查询能力

必须扩展现有：

```text
GET /api/framework/memory
```

更新或增加：

- `reactThreadContinuationSupported=true`
- `supervisorThreadContinuationSupported=true`
- `agentApiAcceptsThreadId=true`
- `supervisorApiAcceptsThreadId=true`
- `serverGeneratesThreadId=true`
- `newRunPerInvocation=true`
- `subAgentsUseFreshContext=true`
- `hitlResumeThreadSyncSupported=true`
- `hitlThreadSyncOrder=SAVE_THREAD_MEMORY_THEN_DELETE_HITL`
- `sameThreadExecutionSerialized=true`
- `failedRunDoesNotOverwriteStableState=true`
- `storesSeparated=true`

要求：

1.字段必须反映真实实现。
2.不得暴露用户线程。
3.不得暴露Checkpoint数量。
4.不得暴露消息或摘要内容。
5.不得暴露Bean完整类名。
6.不得创建第二个框架查询Controller。
7.无模型配置时接口仍可查询。

# 二十六、Spring Bean装配

必须新增或装配：

- `SupervisorThreadConversationStateMapper`
- `SupervisorThreadPersistencePolicy`

必须修改SupervisorEngine Bean，使其获得所需Mapper和Policy。

必须修改AuthenticatedSupervisorApplicationService Bean，使其获得：

- ThreadConversationCheckpointService
- ThreadExecutionCoordinator
- ThreadIdGenerator
- ThreadIdValidator
- SupervisorEngine
- RunIdGenerator
- TaskIdGenerator或现有等价能力

必须修改ReactResumeEngine Bean，使其获得：

- ReactThreadConversationStateMapper
- ReactThreadPersistencePolicy
- Clock

必须修改ApprovalResumeApplicationService Bean，使其获得：

- ThreadExecutionCoordinator
- ThreadConversationCheckpointService

要求：

1.runtime类不得添加Spring注解。
2.使用现有`@Configuration`和`@Bean`。
3.不得创建第二个SupervisorEngine。
4.不得创建第二个ReactResumeEngine。
5.不得创建第二个CheckpointStore。
6.不得创建第二个ThreadExecutionCoordinator。
7.不得创建Fake Engine。
8.不得产生循环依赖。
9.无模型配置时Bean仍可装配。
10.真正调用时无模型沿用现有模型不可用错误。
11.不得在bootstrap启动类手工new。
12.不得修改长期MemoryStore Bean。

# 二十七、实际验证场景

必须完成当前环境能够执行的验证。

## 场景1：新Supervisor线程

请求不带threadId：

```json
{
  "supervisorName":"general_supervisor",
  "message":"计算12乘以8"
}
```

预期：

1.服务器生成threadId。
2.生成runId。
3.父Supervisor使用新State。
4.子Agent使用fresh context。
5.父流程完成后保存SUPERVISOR类型THREAD_MEMORY。
6.响应包含threadId。

## 场景2：Supervisor同线程续接

第二次携带第一次threadId：

```json
{
  "supervisorName":"general_supervisor",
  "message":"再把刚才的结果加10",
  "threadId":"..."
}
```

预期：

1.threadId不变。
2.runId改变。
3.加载父Supervisor完整历史。
4.不重复加入System消息。
5.追加当前User消息。
6.父ContextWindowSnapshot续接。
7.子Agent仍fresh。
8.完成后保存新的THREAD_MEMORY。

如果无真实模型，只验证状态和Checkpoint行为，并明确未验证语义回答。

## 场景3：父子上下文隔离

预期：

1.父previousState不传给子Agent。
2.子AgentState不包含父完整历史。
3.子AgentContextWindowSnapshot为空开始。
4.父只接收子Agent业务结果。
5.子内部临时State不进入父THREAD_MEMORY。

## 场景4：Agent线程用于Supervisor

使用普通Agent创建的threadId调用Supervisor。

预期：

1.返回THREAD_PARTICIPANT_MISMATCH。
2.使用安全404。
3.不得暴露原Agent名称。
4.原线程状态不变。

## 场景5：不同Supervisor名称

使用general_supervisor线程调用另一个Supervisor。

预期：

1.安全拒绝。
2.不得重新生成System消息。
3.不得覆盖原状态。

## 场景6：Supervisor并发

同一用户同一Supervisor threadId并发请求。

预期：

1.一个获得Lease。
2.另一个THREAD_BUSY。
3.不会双重保存。
4.不同threadId可并发。

## 场景7：HITL批准后同步

危险Agent调用进入SUSPENDED。

批准并恢复完成后：

1.工具只执行一次。
2.恢复结果稳定完成。
3.先保存THREAD_MEMORY。
4.再删除目标HITL_RECOVERY。
5.同threadId下一轮能够加载批准后的完整历史。
6.下一轮不会重新执行旧工具。

## 场景8：HITL拒绝后同步

拒绝并恢复完成后：

1.危险工具不执行。
2.APPROVAL_REJECTED回灌。
3.Agent形成最终答复。
4.保存THREAD_MEMORY。
5.下一轮历史包含拒绝结果。
6.HITL_RECOVERY被清理。

## 场景9：再次挂起

恢复后又遇到另一个危险调用。

预期：

1.创建或更新新的HITL_RECOVERY。
2.不保存THREAD_MEMORY。
3.不清理新的待审批记录。
4.普通新消息继续被THREAD_SUSPENDED阻止。

## 场景10：恢复失败

预期：

1.不保存新THREAD_MEMORY。
2.旧稳定THREAD_MEMORY保留。
3.HITL状态按现有失败策略保存。
4.不得返回线程同步成功。

## 场景11：审批与普通请求并发

审批恢复持有Lease时，同threadId普通Agent请求进入。

预期：

1.普通请求返回THREAD_BUSY或THREAD_SUSPENDED。
2.不会与恢复并发修改状态。
3.不会双重执行工具。
4.不会产生两个最新THREAD_MEMORY。

## 场景12：跨用户审批

用户B尝试审批用户A的run。

预期：

1.安全拒绝。
2.不得获取用户A的Lease。
3.不得加载用户A线程状态。
4.不得泄漏threadId和操作参数。

不得为验证新增隐藏Controller或测试脚本。

# 二十八、日志和安全

允许记录：

- userId
- threadId安全短标识
- runId
- executionType
- participantName
- newThread或continuedThread
- HITL批准或拒绝状态
- CheckpointPurpose
- 保存和清理结果
- 消息数量
- Lease获取状态

禁止记录：

- Session ID
- 完整messages
- Summary正文
- Tool参数
- ToolResult正文
- PendingApproval payload
- permissions
- Checkpoint stateData
- 完整ModelRequest
- 完整ModelResponse
- 用户密码和API Key

要求：

1.拒绝跨用户访问时不得打印对方历史。
2.错误响应不得暴露Checkpoint结构。
3.工具执行错误不得打印完整参数。
4.线程同步失败可记录错误码和cause类型。
5.不得在前端或HTTP结果中暴露ThreadConversationState。

# 二十九、本批禁止实现

明确禁止：

1.Supervisor interrupt/resume。
2.Supervisor审批Checkpoint。
3.父Supervisor中断状态恢复。
4.长期MemoryStore注入ReasonNode。
5.LLM自动提取用户偏好。
6.长期记忆工具。
7.记忆检索工具。
8.前端threadId续接功能。
9.前端记忆页面。
10.线程列表API。
11.线程详情API。
12.线程删除API。
13.数据库。
14.Redis。
15.文件持久化。
16.分布式锁。
17.Checkpoint TTL。
18.失败轮自动重试。
19.客户端提交消息历史。
20.客户端提交runId。
21.客户端提交userId。
22.修改ToolApprovalInterceptor审批语义。
23.修改危险工具实现。
24.隐藏调试Controller。
25.测试脚本、README和Git操作。

# 三十、编译验收

必须执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

必须逐项确认：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.`SupervisorThreadConversationStateMapper`真实存在。
4.`SupervisorThreadPersistencePolicy`存在或等价集中策略存在。
5.SupervisorEngine包含线程执行方法。
6.旧Supervisor execute方法保持兼容。
7.新Supervisor线程只插入一次System消息。
8.续接Supervisor线程不重复插入System消息。
9.续接线程追加一条当前User消息。
10.续接线程使用新runId。
11.续接线程保持原threadId。
12.Supervisor临时路由字段被重置。
13.父ContextWindowSnapshot得到恢复。
14.子Agent保持fresh context。
15.子Agent不加载父THREAD_MEMORY。
16.稳定完成后提取父ThreadConversationState。
17.失败和不稳定结果不保存。
18.正式Supervisor API支持可选threadId。
19.正式Supervisor响应包含threadId。
20.Supervisor同线程并发得到THREAD_BUSY。
21.Agent与Supervisor线程不能混用。
22.ReactResumeEngine能够返回可选稳定线程状态。
23.ApprovalResumeApplicationService使用线程Lease。
24.HITL恢复完成先保存THREAD_MEMORY再清理HITL_RECOVERY。
25.批准后工具不重复执行。
26.拒绝后稳定对话状态可保存。
27.再次SUSPENDED不保存THREAD_MEMORY。
28.恢复失败不覆盖旧稳定状态。
29.ContextWindowSnapshot在HITL恢复后正确同步。
30.CheckpointStore和MemoryStore仍严格分离。
31.没有实现Supervisor中断恢复。
32.没有修改前端。
33.没有长期记忆模型注入。
34.git diff没有无关修改。

编译通过后必须搜索确认：

1.不存在父previousState传给子Agent。
2.不存在子Agent加载父threadId THREAD_MEMORY。
3.不存在ReactResumeEngine直接保存Checkpoint。
4.不存在先删除HITL_RECOVERY再保存THREAD_MEMORY。
5.不存在审批查询返回THREAD_MEMORY。
6.不存在Supervisor线程状态写入MemoryStore。
7.不存在客户端提交runId和userId。
8.不存在第二个CheckpointStore。
9.不存在Supervisor中断恢复相关伪实现。

# 三十一、最终输出

最终输出必须逐项对应实施清单，只输出：

1.新增和修改文件清单。
2.SupervisorThreadConversationStateMapper新线程初始化规则。
3.Supervisor续接初始化及临时字段清理规则。
4.Supervisor稳定状态提取规则。
5.SupervisorEngine线程执行流程。
6.旧Supervisor execute兼容方式。
7.正式Supervisor应用服务的新线程与续接流程。
8.正式Supervisor请求和响应DTO调整。
9.父Supervisor与子Agent上下文隔离方式。
10.Supervisor并发Lease使用范围。
11.ReactResumeEngine线程结果扩展。
12.HITL审批恢复中的Lease范围。
13.THREAD_MEMORY保存与HITL_RECOVERY清理顺序。
14.批准、拒绝、再次挂起和失败的状态保存语义。
15.ContextWindowSnapshot同步方式。
16.Agent和Supervisor线程类型隔离。
17.错误码和HTTP映射。
18.框架查询能力。
19.实际完成的验证场景。
20.因模型或环境限制未完成的验证及原因。
21.编译和打包结果。
22.发现但未处理的问题。

不得把计划描述为已完成。

不得只输出“编译通过”。

不得遗漏任一强制交付类型。

编译或打包失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE8_BATCH4_END===