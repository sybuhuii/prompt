你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须完整阅读并遵守根目录`AGENTS.md`。

当前已完成：

1.身份、Session、RBAC、工具ACL和多用户隔离。
2.单Agent ReAct执行引擎。
3.Supervisor多Agent编排。
4.HITL中断、审批、Checkpoint保存和恢复。
5.上下文消息数裁剪、Token裁剪和LLM摘要。
6.上下文窗口正式接入ReAct与Supervisor。
7.长期记忆MemoryStore和InMemoryMemoryStore。
8.CheckpointPurpose区分HITL_RECOVERY与THREAD_MEMORY。
9.CheckpointStore按userId、threadId和purpose查询。
10.InMemoryCheckpointStore线程辅助索引。
11.ThreadConversationState。
12.ThreadCheckpointStateMapper。
13.ThreadConversationCheckpointService。
14.ThreadIdGenerator和ThreadIdValidator。
15.ThreadExecutionCoordinator和ThreadExecutionLease。

当前只执行第八阶段第3批：

1.实现ReactAgentState与ThreadConversationState之间的映射。
2.扩展ReactAgentEngine，支持新线程和已有线程执行。
3.同一threadId的每次调用生成新的runId。
4.恢复上一轮完整消息和ContextWindowSnapshot。
5.追加当前User消息，不重复添加System消息。
6.正常完成后保存新的THREAD_MEMORY Checkpoint。
7.失败、挂起时不覆盖上一轮稳定状态。
8.正式Agent API支持可选threadId。
9.同一用户同一threadId执行互斥。
10.阻止存在HITL挂起运行的线程接收普通新消息。
11.保持多用户和多线程严格隔离。

本批不实现Supervisor线程续接，不修改HITL批准/拒绝后的线程状态同步，不修改前端，不实现长期记忆注入。

# 一、强制边界

必须遵守：

1.不得重新实现CheckpointStore。
2.不得重新实现InMemoryCheckpointStore。
3.不得新增ConversationStore。
4.不得新增ShortTermMemoryStore。
5.不得使用MemoryStore保存会话消息。
6.不得使用CheckpointStore保存长期用户偏好。
7.不得覆盖ReactAgentState中的完整messages。
8.不得把ContextWindowSnapshot中的裁剪消息当作完整历史。
9.不得修改SupervisorEngine。
10.不得修改Supervisor正式API。
11.不得修改Supervisor图。
12.不得修改HITL批准和拒绝恢复流程。
13.不得实现HITL恢复完成后保存THREAD_MEMORY。
14.不得修改前端。
15.不得实现线程列表、详情和删除接口。
16.不得让客户端提交runId、userId、消息历史或Checkpoint。
17.不得在未读完提示词前开始修改代码。
18.不得以编译通过代替功能验收。

# 二、执行前检查

必须检查真实代码：

1.`ReactAgentEngine`当前公开方法。
2.`ReactAgentEngine`如何创建初始ReactAgentState。
3.`ReactAgentGraphFactory`和CompiledGraph调用方式。
4.`ReactAgentState`的StateKey和类型安全访问器。
5.ReAct初始System消息生成位置。
6.当前User消息写入位置。
7.iteration、status、pendingToolCalls、toolResults等字段。
8.`ContextWindowSnapshot`和`ContextProcessingTrace`的StateKey。
9.`AgentResult`及其状态字段。
10.Complete、MaxIterations、Suspend和Failure节点。
11.`RunContext`、`AgentTask`和RunId生成方式。
12.`AuthenticatedAgentApplicationService`。
13.正式Agent Controller。
14.正式Agent请求和响应DTO。
15.现有Session认证拦截方式。
16.`CheckpointPurpose`。
17.`ThreadConversationState`。
18.`ThreadCheckpointStateMapper`。
19.`ThreadConversationCheckpointService`。
20.`ThreadExecutionCoordinator`。
21.`ThreadExecutionLease`。
22.`ThreadIdGenerator`。
23.`ThreadIdValidator`。
24.`CheckpointStore.findByThreadId`。
25.现有PendingApproval或挂起Checkpoint查询方法。
26.现有错误码和HTTP异常映射。
27.现有Spring Bean配置。

以下类型必须真实存在且语义基本符合第2批：

- `CheckpointPurpose`
- `ThreadConversationState`
- `ThreadCheckpointStateMapper`
- `ThreadConversationCheckpointService`
- `ThreadExecutionCoordinator`
- `ThreadExecutionLease`
- `ThreadIdGenerator`
- `ThreadIdValidator`
- `ReactAgentEngine`
- `ReactAgentState`

任一关键类型缺失时立即停止。

不得为了继续执行而临时创建第二套近似类型。

# 三、固定执行语义

必须保持：

```text
sessionId：认证调用者
threadId：一段多轮会话
runId：当前一次Agent运行
taskId：当前一次任务
checkpointId：某一份状态快照
```

同一threadId的两次请求：

```text
threadId=thread-001

第一次：
runId=run-001
用户：我叫小李

第二次：
runId=run-002
用户：我叫什么？
```

规则：

1.threadId保持不变。
2.runId必须不同。
3.taskId必须不同。
4.第二次加载第一次成功完成后的完整消息。
5.第二次追加新的User消息。
6.第二次不得重复加入初始System消息。
7.第二次不得恢复第一次的iteration。
8.第二次不得恢复临时Tool执行状态。
9.第二次必须恢复ContextWindowSnapshot。
10.第二次必须使用当前Session构建新的RunContext。

# 四、ThreadExecutionOutcome

在`agent-runtime`必须新增不可变：

`ThreadExecutionOutcome`

字段名称固定：

- `result`
- `conversationState`

类型语义：

```java
AgentResult result
Optional<ThreadConversationState> conversationState
```

要求：

1.result不能为空。
2.conversationState不得为null。
3.成功稳定终态时conversationState存在。
4.SUSPENDED时conversationState为空。
5.FAILED时conversationState为空。
6.不存在稳定状态时不得伪造空ThreadConversationState。
7.不得包含ReactAgentState。
8.不得包含CompiledGraph。
9.不得包含UserSession。
10.不得包含RunContext。
11.不得包含Session ID。
12.不得包含Checkpoint。
13.模型保持不可变。
14.不得依赖Spring。

如已有完全等价类型，复用并补齐固定语义，不得新增V2。

# 五、ReactThreadConversationStateMapper

在`agent-runtime`必须新增纯Java：

`ReactThreadConversationStateMapper`

职责：

1.根据新任务构建新的ReactAgentState。
2.根据已有ThreadConversationState构建续接ReactAgentState。
3.从执行结束后的ReactAgentState提取ThreadConversationState。
4.集中清理上一轮临时运行字段。
5.不得访问CheckpointStore。
6.不得执行模型或工具。

必须提供或实现等价方法：

```java
ReactAgentState createInitialState(
    AgentDefinition definition,
    AgentTask task,
    RunContext runContext
);

ReactAgentState createContinuedState(
    AgentDefinition definition,
    AgentTask task,
    RunContext runContext,
    ThreadConversationState previousState
);

ThreadConversationState extractStableState(
    String agentName,
    String runId,
    ReactAgentState finalState,
    Instant updatedAt
);
```

不得把这些逻辑散落到Application Service和Controller。

# 六、新线程State初始化

`createInitialState`必须保持当前ReactAgentEngine既有行为。

至少包括：

1.创建Agent规定的System消息。
2.追加本次User消息。
3.设置当前RunContext。
4.设置当前AgentDefinition或现有必要标识。
5.iteration初始化为0。
6.status初始化为运行状态。
7.pendingToolCalls为空。
8.toolResults为空。
9.executionBuffer为空。
10.pendingApproval为空。
11.finalResult为空。
12.ContextWindowSnapshot为空。
13.latestContextTrace为空。
14.使用当前runId和threadId。
15.不得写入MemoryStore。
16.不得读取旧Checkpoint。

如果当前Engine初始化还有其他必要StateKey，必须保持兼容。

不得为线程能力删除已有初始化字段。

# 七、续接线程State初始化

`createContinuedState`必须执行：

1.校验previousState.executionType为REACT_AGENT。
2.校验previousState.participantName等于当前agentName。
3.读取previousState.messages。
4.复制完整消息列表。
5.不得修改previousState.messages。
6.不得重复插入System消息。
7.追加当前AgentTask中的User消息。
8.设置新的RunContext。
9.设置新的runId。
10.保持原threadId。
11.设置新的taskId。
12.iteration重置为0。
13.status重置为运行状态。
14.pendingToolCalls清空。
15.toolResults清空。
16.executionBuffer清空。
17.pendingApproval清空。
18.currentToolCall清空。
19.executionCursor清空。
20.finalResult清空。
21.failure信息清空。
22.恢复previousState.contextWindowSnapshot。
23.恢复previousState.latestContextTrace。
24.不得恢复上一轮Tool调用游标。
25.不得重新执行上一轮工具。
26.不得恢复上一轮Assistant最终结果为当前finalResult。
27.不得恢复上一轮RunContext。
28.不得恢复上一轮sessionId。
29.当前权限必须来自当前已验证Session。
30.不得从旧State读取权限覆盖当前Session。

如果实际StateKey名称不同，使用真实StateKey。

不得创建平行的普通Java字段绕过Map State。

# 八、System消息规则

新线程：

1.按当前AgentDefinition生成System消息。
2.只插入一次。

续接线程：

1.不得再次生成System消息。
2.必须保留上一轮完整历史中的原System消息。
3.不得删除旧System消息。
4.不得将System消息替换为SummaryAgentMessage。
5.不得将当前AgentDefinition的新提示词覆盖旧会话System消息。

participantName不匹配时必须拒绝续接，而不是重新生成另一Agent的System消息。

# 九、用户消息规则

每次正式调用必须只追加一条当前User消息。

要求：

1.消息内容来自当前请求。
2.消息不能为空。
3.不得允许客户端指定role。
4.不得允许客户端提交Assistant消息。
5.不得允许客户端提交Tool消息。
6.不得允许客户端提交System消息。
7.不得重复追加同一请求消息。
8.不得在Application Service和Engine各追加一次。
9.必须明确只有一个位置负责追加。
10.推荐由ReactThreadConversationStateMapper负责。

# 十、稳定状态提取

`extractStableState`必须从最终ReactAgentState提取：

- executionType=REACT_AGENT
- participantName=agentName
- 完整messages
- contextWindowSnapshot
- latestContextTrace
- lastCompletedRunId
- updatedAt

要求：

1.messages必须是完整状态消息，不是processedMessages。
2.不得使用ContextWindowSnapshot.windowMessages代替完整历史。
3.不得包含pendingToolCalls。
4.不得包含tool execution buffer。
5.不得包含pendingApproval。
6.不得包含executionCursor。
7.不得包含failure异常。
8.不得包含RunContext。
9.不得包含Session。
10.不得包含permissions。
11.不得包含finalResult对象。
12.不得包含Spring AI Message。
13.返回ThreadConversationState不可变。
14.提取前确认消息历史不存在未完成ToolCall。
15.提取前确认没有PendingApproval。
16.提取前确认当前状态属于稳定终态。
17.不稳定状态抛THREAD_CHECKPOINT_INVALID或返回不可持久化结果。
18.不得自动伪造缺失ToolResult。

# 十一、可持久化终态

必须集中定义判断逻辑，不能在多个服务中用不同规则判断。

可以在Mapper或独立纯Java策略中增加：

`ReactThreadPersistencePolicy`

方法建议：

```java
boolean isPersistable(
    AgentResult result,
    ReactAgentState finalState
);
```

允许持久化：

1.正常COMPLETE。
2.如果MaxIterations节点产生了完整最终Assistant结果，且：
- 没有PendingApproval；
- 没有PendingToolCall；
- 工具历史配对完整；
  可作为稳定终态保存。

禁止持久化：

1.SUSPENDED。
2.FAILURE。
3.模型调用异常。
4.工具执行中断。
5.存在PendingApproval。
6.存在未完成ToolCall。
7.消息历史非法。
8.状态映射失败。

要求：

1.失败轮不得覆盖旧稳定Checkpoint。
2.挂起轮不得保存THREAD_MEMORY。
3.不能只根据HTTP 200判断是否可保存。
4.不能只根据AgentResult.content非空判断。
5.判断必须可测试且确定。

# 十二、扩展ReactAgentEngine

必须最小扩展现有：

`ReactAgentEngine`

新增方法，名称固定：

```java
ThreadExecutionOutcome executeThread(
    AgentDefinition definition,
    AgentTask task,
    RunContext runContext,
    Optional<ThreadConversationState> previousState
);
```

保留原有公开`execute`方法。

推荐兼容方式：

```java
public AgentResult execute(...) {
    return executeThread(
        definition,
        task,
        runContext,
        Optional.empty()
    ).result();
}
```

但必须以真实接口为准，不得破坏已有调用方。

执行流程：

1.校验definition、task和runContext。
2.previousState为空时创建新State。
3.previousState存在时创建续接State。
4.调用现有CompiledGraph。
5.保持原有流式或同步执行方式。
6.取得最终ReactAgentState。
7.取得AgentResult。
8.调用持久化策略判断是否稳定。
9.稳定时提取ThreadConversationState。
10.不稳定时conversationState为空。
11.返回ThreadExecutionOutcome。

要求：

1.不得在Engine中访问CheckpointStore。
2.不得在Engine中保存THREAD_MEMORY。
3.不得在Engine中访问SessionStore。
4.不得在Engine中生成threadId。
5.不得在Engine中生成runId。
6.不得修改ReAct图节点。
7.不得修改工具调用链。
8.不得修改ReasonNode上下文处理逻辑。
9.不得修改HITL中断信号。
10.不得吞掉AgentInterruptSignal。
11.不得让executeThread无限递归调用execute。
12.原execute行为必须兼容。
13.同一个Engine Bean支持并发运行。

# 十三、ThreadConversationCheckpointService扩展

必须最小修改现有：

`ThreadConversationCheckpointService`

增加：

```java
boolean hasActiveHitlRun(
    String userId,
    String threadId
);
```

实现规则：

1.查询相同userId和threadId的HITL_RECOVERY Checkpoint。
2.仅以下状态视为活跃：
- SUSPENDED
- RESUMING
  3.COMPLETED不视为活跃。
  4.FAILED不视为活跃，除非现有HITL语义明确仍需人工处理。
  5.不得查询THREAD_MEMORY作为HITL状态。
  6.不得只按threadId查询。
  7.不得访问MemoryStore。
  8.返回确定boolean。
  9.不得删除任何Checkpoint。
  10.不得修改审批状态。

本批只使用该方法阻止普通Agent请求进入挂起线程。

不修改HITL恢复完成后的THREAD_MEMORY保存。

# 十四、正式Agent应用服务

必须修改现有：

`AuthenticatedAgentApplicationService`

支持可选threadId。

推荐新增或扩展方法：

```java
AuthenticatedAgentRunResult invoke(
    UserSession session,
    String agentName,
    String message,
    Optional<String> requestedThreadId
);
```

如现有返回类型名称不同，最小扩展实际类型。

旧方法必须保持兼容：

```java
invoke(session, agentName, message)
```

旧方法应委托新方法并使用空threadId。

不得复制两套执行逻辑。

# 十五、正式Agent调用流程

`AuthenticatedAgentApplicationService`必须按以下顺序执行：

## 基础校验

1.校验UserSession。
2.校验agentName。
3.校验message。
4.查询AgentDefinition。
5.不得从请求读取userId。
6.不得从请求读取permissions。

## threadId为空

1.调用ThreadIdGenerator生成新threadId。
2.调用ThreadIdValidator校验生成结果。
3.previousState=Optional.empty。
4.不得查询不存在的Checkpoint。

## threadId存在

1.trim请求threadId。
2.调用ThreadIdValidator。
3.调用ThreadConversationCheckpointService.load：
- userId来自当前Session；
- executionType=REACT_AGENT；
- participantName=agentName。
  4.load为空时抛THREAD_NOT_FOUND。
  5.不得将不存在的客户端threadId自动创建为新线程。
  6.participant不匹配时沿用THREAD_PARTICIPANT_MISMATCH。
  7.不得泄露该threadId是否属于其他用户。

## 挂起检查

1.调用hasActiveHitlRun(userId,threadId)。
2.为true时抛THREAD_SUSPENDED。
3.不得启动新run。
4.不得取得执行Lease后才发现挂起，避免无意义占用。
5.不得删除挂起Checkpoint。

## 并发Lease

1.调用ThreadExecutionCoordinator.acquire(userId,threadId)。
2.使用try-with-resources。
3.Lease获取失败抛THREAD_BUSY。
4.所有执行、保存和异常路径均必须释放Lease。

## 当前运行

1.生成新的runId。
2.生成新的taskId。
3.构造新的AgentTask。
4.构造新的RunContext。
5.threadId使用确定后的threadId。
6.userId来自当前Session。
7.角色和权限来自当前Session。
8.调用ReactAgentEngine.executeThread。
9.不得调用旧无状态execute方法。

## 保存

1.outcome.conversationState存在时调用ThreadConversationCheckpointService.save。
2.save使用当前userId、threadId和runId。
3.保存失败时返回明确框架错误。
4.outcome.conversationState为空时不保存。
5.SUSPENDED不保存THREAD_MEMORY。
6.FAILED不保存THREAD_MEMORY。
7.失败不能删除之前稳定Checkpoint。
8.先前稳定状态必须保持可加载。

## 返回

至少返回：

- runId
- threadId
- AgentResult或现有安全响应字段

不得返回：

- ThreadConversationState
- 完整messages
- Checkpoint
- ContextWindowSnapshot
- UserSession
- permissions

# 十六、AuthenticatedAgentRunResult

检查现有正式Agent应用层返回类型。

如果已经能表达runId、threadId和AgentResult，直接复用。

如果不能，必须新增或最小扩展：

`AuthenticatedAgentRunResult`

字段至少包含：

- `runId`
- `threadId`
- `result`

要求：

1.runId不能为空。
2.threadId不能为空。
3.result不能为空。
4.模型不可变。
5.不得包含Session ID。
6.不得包含userId。
7.不得包含Checkpoint。
8.不得包含完整State。
9.不得包含ThreadConversationState。
10.不得依赖Spring Web。

不得创建`AuthenticatedAgentRunResultV2`。

# 十七、正式Agent请求DTO

必须最小扩展现有正式Agent请求DTO，增加：

```java
String threadId
```

请求示例：

```json
{
  "agentName": "utility_agent",
  "message": "我叫什么？",
  "threadId": "可选"
}
```

规则：

1.threadId可选。
2.null或空白表示新线程。
3.非空时表示续接已有线程。
4.不得增加runId字段。
5.不得增加userId字段。
6.不得增加messages字段。
7.不得增加contextSnapshot字段。
8.不得增加Checkpoint字段。
9.threadId格式校验由应用服务统一完成。
10.Controller不得自行加载Checkpoint。
11.旧请求不带threadId仍必须兼容。

如果agentName来自URL而不是DTO，保持现有接口风格，只增加threadId。

# 十八、正式Agent响应DTO

必须确保正式Agent响应包含：

- runId
- threadId
- 原有Agent结果字段

要求：

1.threadId必须是服务器最终使用的threadId。
2.新线程返回服务器生成的threadId。
3.续接线程返回请求中的同一threadId。
4.不得返回Checkpoint ID。
5.不得返回完整消息。
6.不得返回ContextWindowSnapshot。
7.不得返回用户身份。
8.不得返回permissions。
9.不得破坏现有前端读取的结果字段。
10.可在现有响应DTO上最小增加字段。

# 十九、Controller边界

正式Agent Controller只能：

1.读取认证拦截器注入的UserSession。
2.读取请求DTO。
3.调用AuthenticatedAgentApplicationService。
4.映射应用结果为响应DTO。

禁止：

1.Controller生成runId。
2.Controller生成threadId。
3.Controller访问CheckpointStore。
4.Controller访问ThreadConversationCheckpointService。
5.Controller访问ReactAgentEngine。
6.Controller直接追加消息。
7.Controller读取客户端userId。
8.Controller处理ThreadExecutionLease。
9.Controller根据threadId判断用户归属。
10.Controller捕获所有异常后返回200。

# 二十、错误码

检查现有`AgentErrorCode`。

缺失时最小新增以下固定错误码：

- `THREAD_NOT_FOUND`
- `THREAD_SUSPENDED`

继续复用第2批：

- `INVALID_THREAD_ID`
- `THREAD_BUSY`
- `THREAD_PARTICIPANT_MISMATCH`
- `THREAD_CHECKPOINT_INVALID`

语义：

## THREAD_NOT_FOUND

客户端提交的threadId在当前userId的THREAD_MEMORY中不存在。

要求：

1.其他用户拥有相同threadId时仍返回THREAD_NOT_FOUND。
2.不得提示“线程属于其他用户”。
3.不得自动创建新线程。

## THREAD_SUSPENDED

相同userId和threadId存在活跃HITL_RECOVERY。

要求：

1.不得启动普通新run。
2.提示先完成审批。
3.不得修改挂起Checkpoint。

# 二十一、HTTP状态映射

必须扩展现有统一异常映射：

- `INVALID_THREAD_ID` → 400
- `THREAD_NOT_FOUND` → 404
- `THREAD_PARTICIPANT_MISMATCH` → 404或项目统一安全404
- `THREAD_BUSY` → 409
- `THREAD_SUSPENDED` → 409
- `THREAD_CHECKPOINT_INVALID` → 500

要求：

1.不得将THREAD_BUSY返回500。
2.不得将THREAD_NOT_FOUND返回401。
3.不得泄露其他用户线程。
4.错误响应不得包含消息历史。
5.错误响应不得包含Checkpoint stateData。
6.错误响应不得包含Session ID。
7.继续复用统一错误响应格式。
8.不得在Controller手写一套错误结构。

# 二十二、ContextWindowSnapshot续接

续接线程必须正确恢复第七阶段上下文窗口。

规则：

1.ThreadConversationState保存完整messages。
2.ThreadConversationState保存ContextWindowSnapshot。
3.续接初始化恢复ContextWindowSnapshot。
4.追加新User消息后进入Reason。
5.ContextWindowManager读取上一轮Snapshot。
6.只处理完整历史中尚未被Snapshot消费的新消息。
7.不得重新摘要已消费的旧消息。
8.不得用snapshot.windowMessages覆盖完整messages。
9.latestContextTrace可恢复为上一轮最新安全统计。
10.本轮第一次Reason后产生新的Snapshot和Trace。
11.执行完成后保存更新后的Snapshot。
12.不同threadId的Snapshot不能共享。
13.不同用户的Snapshot不能共享。
14.不得将SummaryAgentMessage写入MemoryStore。

# 二十三、工具调用续接安全

上一轮已完成工具交互：

```text
Assistant ToolCall
ToolResult
Assistant Final
```

下一轮续接时：

1.完整工具交互保留在messages。
2.不得重新执行旧ToolCall。
3.pendingToolCalls必须为空。
4.currentToolCall必须为空。
5.executionCursor必须清空。
6.新Reason只根据消息历史理解旧工具结果。
7.ContextMessageHistoryValidator仍应认为历史合法。
8.不得删除旧ToolResult导致孤立ToolCall。
9.不得将旧ToolResult重新加入本轮观察缓冲区。
10.不得重新触发旧危险工具审批。

# 二十四、失败和挂起语义

## 首次新线程失败

1.不保存THREAD_MEMORY。
2.返回本轮失败结果或现有异常。
3.生成的threadId可以返回，也可以不返回，必须结合现有响应语义明确。
4.推荐只有真正开始运行后返回runId和threadId。
5.不得保存损坏状态。

## 已有线程新一轮失败

1.不保存新THREAD_MEMORY。
2.旧稳定Checkpoint保留。
3.下一次续接从上一轮成功状态开始。
4.失败轮User消息不写入稳定线程。
5.不得先覆盖旧状态再执行。

## 新一轮SUSPENDED

1.由现有HITL流程保存HITL_RECOVERY。
2.本批不保存THREAD_MEMORY。
3.上一轮稳定THREAD_MEMORY保留。
4.线程被hasActiveHitlRun阻止普通新消息。
5.批准或拒绝后的线程状态同步留到第4批。

# 二十五、同线程并发

正式Agent应用服务必须使用ThreadExecutionCoordinator。

场景：

```text
user-A + thread-1
```

请求一运行中时，请求二进入。

预期：

1.请求一持有Lease。
2.请求二得到THREAD_BUSY。
3.请求二不加载并执行模型。
4.请求二不保存Checkpoint。
5.请求一结束后释放Lease。
6.之后请求可以继续。

要求：

1.Lease范围覆盖加载、执行和保存。
2.不能只保护保存阶段。
3.检查HITL后必须在Lease内重新确认线程状态是否需要时，可采用最小双重检查。
4.不得使用全局锁阻塞其他线程。
5.异常路径必须释放Lease。
6.不同userId相同threadId允许并发。
7.同userId不同threadId允许并发。

为避免加载后被并发更新，推荐顺序：

```text
确定threadId
→检查格式
→获取Lease
→检查HITL
→加载previousState
→执行
→保存
→释放Lease
```

若采用此顺序，以一致性为准。

不得在Lease之外加载旧状态后再执行。

# 二十六、多用户隔离

必须保证：

1.userId只能来自已验证UserSession。
2.load同时使用userId和threadId。
3.save同时使用userId和threadId。
4.Lease Key同时使用userId和threadId。
5.hasActiveHitlRun同时使用userId和threadId。
6.客户端threadId不能改变用户身份。
7.用户A不能读取用户B的历史。
8.用户A不能覆盖用户B的Checkpoint。
9.用户A不能判断用户B是否存在该threadId。
10.不同用户相同threadId允许独立存在。
11.重新登录后userId相同即可继续。
12.不得依赖旧sessionId。
13.不得把用户身份发送给LLM。
14.不得把权限写入ThreadConversationState。
15.当前RunContext权限必须来自当前Session。

# 二十七、Spring Bean装配

必须装配：

- `ReactThreadConversationStateMapper`
- `ReactThreadPersistencePolicy`，如果单独实现

必须修改现有ReactAgentEngine Bean装配，使其获得所需Mapper和Policy。

必须修改AuthenticatedAgentApplicationService Bean装配，使其获得：

- ThreadConversationCheckpointService
- ThreadExecutionCoordinator
- ThreadIdGenerator
- ThreadIdValidator
- ReactAgentEngine
- RunIdGenerator
- 现有TaskIdGenerator或等价能力

要求：

1.runtime类不得添加Spring注解。
2.使用现有`@Configuration`和`@Bean`。
3.不得创建第二个ReactAgentEngine。
4.不得创建第二个CheckpointStore。
5.不得创建第二个ThreadExecutionCoordinator。
6.不得创建Fake Engine。
7.不得产生循环依赖。
8.无模型配置时Bean仍可装配。
9.真正调用时无模型则沿用现有模型不可用错误。
10.不得在bootstrap启动类手工new。
11.不得修改Supervisor Bean。

# 二十八、框架查询能力

必须扩展现有：

```text
GET /api/framework/memory
```

至少新增或确认：

- `reactThreadContinuationSupported=true`
- `agentApiAcceptsThreadId=true`
- `serverGeneratesThreadId=true`
- `newRunPerInvocation=true`
- `completeHistoryPreserved=true`
- `contextWindowSnapshotContinued=true`
- `sameThreadExecutionSerialized=true`
- `failedRunDoesNotOverwriteStableState=true`
- `supervisorThreadContinuationSupported=false`
- `hitlResumeThreadSyncSupported=false`

要求：

1.后两个字段必须真实反映本批尚未实现。
2.不得伪造Supervisor已支持。
3.不得伪造HITL恢复后已同步。
4.不得暴露用户线程列表。
5.不得暴露消息内容。
6.不得暴露Checkpoint数量。
7.不得暴露实现类完整包名。
8.不得新增第二个框架查询Controller。

# 二十九、实际验证场景

必须完成能够在当前环境执行的验证。

## 场景1：首次调用

请求：

```json
{
  "agentName": "utility_agent",
  "message": "我叫小李"
}
```

预期：

1.服务器生成threadId。
2.服务器生成runId。
3.ReactAgentEngine使用新线程State。
4.执行完成后保存THREAD_MEMORY。
5.响应包含threadId。
6.Checkpoint中participantName为utility_agent。
7.Checkpoint purpose为THREAD_MEMORY。

## 场景2：同线程第二次调用

请求：

```json
{
  "agentName": "utility_agent",
  "message": "我刚才说我叫什么？",
  "threadId": "第一次返回的threadId"
}
```

预期：

1.threadId不变。
2.runId改变。
3.加载上一轮完整messages。
4.不重复添加System消息。
5.追加一条当前User消息。
6.模型能够看到上一轮内容。
7.执行完成后保存新的THREAD_MEMORY。
8.loadLatest返回第二轮状态。

如果没有真实模型，只能验证状态初始化、保存和消息数量，必须明确说明未验证模型语义回答。

## 场景3：不带threadId新建第二线程

预期：

1.生成不同threadId。
2.不加载第一线程历史。
3.两个Checkpoint独立。

## 场景4：其他用户threadId

用户B提交用户A的threadId。

预期：

1.返回THREAD_NOT_FOUND。
2.不返回participantName。
3.不返回消息数量。
4.不访问用户A状态。

## 场景5：不同Agent复用threadId

utility_agent线程使用calculator_agent继续。

预期：

1.返回THREAD_PARTICIPANT_MISMATCH。
2.HTTP使用安全404。
3.原线程状态不变。
4.不得把utility历史传给calculator。

## 场景6：并发执行

同一用户同一threadId并发请求。

预期：

1.一个请求获得Lease。
2.另一个返回THREAD_BUSY。
3.不存在双重保存覆盖。
4.不同threadId仍可并发。

## 场景7：失败不覆盖

已有稳定Checkpoint后模拟或遇到模型失败。

预期：

1.不保存新THREAD_MEMORY。
2.旧稳定状态仍可load。
3.失败轮User消息不进入稳定状态。

## 场景8：上下文窗口续接

已有线程产生ContextWindowSnapshot后继续请求。

预期：

1.恢复原Snapshot。
2.只吸收新增User消息。
3.不重复摘要旧历史。
4.完整messages继续增长。
5.模型输入窗口受预算限制。

## 场景9：工具历史续接

上一轮包含完整ToolCall和ToolResult。

预期：

1.续接后工具历史仍配对。
2.旧工具不重新执行。
3.新Reason可读取旧结果。
4.pendingToolCalls为空。

## 场景10：HITL挂起线程

危险工具已挂起后，使用相同threadId发送普通请求。

预期：

1.返回THREAD_SUSPENDED。
2.不启动新run。
3.不修改HITL Checkpoint。
4.不覆盖上一轮稳定THREAD_MEMORY。

不得为验证新增隐藏Controller或测试脚本。

# 三十、日志和安全

允许记录：

- userId
- threadId安全短标识
- runId
- agentName
- newThread或continuedThread
- previousMessageCount
- finalMessageCount
- Checkpoint保存结果
- AgentResult状态
- Lease获取结果

禁止记录：

- Session ID
- 完整消息正文
- Summary正文
- Tool参数
- ToolResult正文
- PendingApproval payload
- permissions
- Checkpoint stateData
- UserSession
- 完整ModelRequest
- 完整ModelResponse

要求：

1.跨用户访问失败不得打印对方线程内容。
2.错误响应不得回显超长非法threadId。
3.线程续接日志不得输出完整用户问题。
4.模型失败可记录错误码和cause类型，不记录Prompt。

# 三十一、本批禁止实现

明确禁止：

1.SupervisorEngine线程续接。
2.AuthenticatedSupervisorApplicationService修改。
3.Supervisor请求DTO增加threadId。
4.Supervisor Controller修改。
5.HITL批准后保存THREAD_MEMORY。
6.HITL拒绝后保存THREAD_MEMORY。
7.ReactResumeEngine返回ThreadExecutionOutcome。
8.ApprovalResumeApplicationService修改。
9.前端会话续接。
10.前端threadId输入框。
11.线程列表API。
12.线程详情API。
13.线程删除API。
14.MemoryStore注入ReasonNode。
15.LLM自动提取用户偏好。
16.记忆工具。
17.数据库。
18.Redis。
19.文件持久化。
20.分布式锁。
21.Checkpoint TTL。
22.失败轮自动重试。
23.客户端提交消息历史。
24.客户端提交runId。
25.客户端提交userId。
26.隐藏调试Controller。
27.测试脚本、README和Git操作。

# 三十二、编译验收

必须执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

必须逐项确认：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.`ThreadExecutionOutcome`真实存在。
4.`ReactThreadConversationStateMapper`真实存在。
5.`ReactThreadPersistencePolicy`存在或等价集中策略存在。
6.ReactAgentEngine包含executeThread方法。
7.旧execute方法保持兼容。
8.新线程只添加一次System消息。
9.续接线程不重复添加System消息。
10.续接线程追加当前User消息。
11.续接线程使用新runId。
12.续接线程保持原threadId。
13.iteration和临时Tool字段被重置。
14.ContextWindowSnapshot得到恢复。
15.稳定完成后提取ThreadConversationState。
16.SUSPENDED不生成稳定线程状态。
17.FAILED不生成稳定线程状态。
18.AuthenticatedAgentApplicationService支持可选threadId。
19.客户端threadId不存在返回THREAD_NOT_FOUND。
20.同一线程并发返回THREAD_BUSY。
21.挂起线程返回THREAD_SUSPENDED。
22.正式Agent响应包含threadId。
23.不同用户不能串读。
24.不同Agent不能共用线程。
25.失败执行不覆盖旧Checkpoint。
26.没有修改Supervisor。
27.没有修改HITL恢复流程。
28.没有修改前端。
29.CheckpointStore和MemoryStore仍严格分离。
30.git diff没有无关修改。

编译通过后必须搜索确认：

1.不存在第二个ConversationStore。
2.不存在第二个ShortTermMemoryStore。
3.不存在Controller直接访问CheckpointStore。
4.不存在客户端提交userId和runId。
5.不存在将processedMessages保存为完整线程历史。
6.不存在续接时重复插入System消息。
7.不存在Engine直接保存Checkpoint。
8.不存在Supervisor线程续接相关改动。
9.不存在HITL恢复完成后线程同步的提前实现。

# 三十三、最终输出

最终输出必须逐项对应实施清单，只输出：

1.新增和修改文件清单。
2.ThreadExecutionOutcome结构。
3.ReactThreadConversationStateMapper的新线程初始化规则。
4.ReactThreadConversationStateMapper的续接初始化规则。
5.临时StateKey清理范围。
6.稳定线程状态提取规则。
7.ReactAgentEngine.executeThread执行流程。
8.旧execute方法兼容方式。
9.AuthenticatedAgentApplicationService的新线程与续接流程。
10.threadId、runId和taskId语义。
11.正式请求和响应DTO调整。
12.ThreadExecutionCoordinator使用范围。
13.HITL挂起线程阻止方式。
14.ContextWindowSnapshot续接方式。
15.失败和挂起不覆盖稳定状态的方式。
16.多用户及不同Agent隔离方式。
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

===PHASE8_BATCH3_END===