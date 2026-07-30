新的第八阶段第2批只完成**短期记忆的Checkpoint存储基础设施**，不接入ReAct、Supervisor、正式API和HITL恢复链。

本批目标是：

```text
稳定线程状态
→保存为THREAD_MEMORY Checkpoint
→按userId+threadId加载
→不同用户和线程严格隔离
```

任务要求短期记忆按`thread_id`保存会话状态，并与按`userId`存储长期事实的`MemoryStore`保持独立。

建议保存为：

```text
.kscc-prompts/phase8-batch2.md
```

KSCC执行指令：

```text
先完整读取根目录`AGENTS.md`和`.kscc-prompts/phase8-batch2.md`，确认提示词末尾存在`===PHASE8_BATCH2_END===`。

修改代码前，先输出不超过60行的“本批实施清单”，逐项列出：
1.必须新增的类型；
2.必须修改的已有类型；
3.CheckpointStore新增方法；
4.InMemoryCheckpointStore索引调整；
5.错误码和Spring Bean；
6.明确禁止修改的运行链；
7.编译与验收项。

实施清单输出后无需等待确认，继续执行。

提示词中使用“必须、名称固定、方法固定、字段包含”的内容均为强制交付项，不得使用自创同义类型替代。已有同名类型时最小扩展，不存在时新增。只有明确标注为“建议、可选”的内容允许调整。

缺少任一文件、结束标记缺失，或现有AgentCheckpoint、CheckpointStore、InMemoryCheckpointStore不存在时，立即停止并列出准确阻塞项，不得重新创建第二套Checkpoint体系。

完成后必须逐项核对实施清单，不得仅以编译通过作为完成标准。
```

````markdown
你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须完整阅读并遵守根目录`AGENTS.md`。

当前已完成：

1.身份、Session、RBAC、工具ACL和多用户隔离。
2.单Agent ReAct执行引擎。
3.Supervisor多Agent编排。
4.HITL中断、审批、Checkpoint保存和恢复。
5.上下文消息数裁剪、Token裁剪和LLM摘要。
6.上下文窗口正式接入ReAct与Supervisor。
7.长期记忆领域模型、MemoryStore和InMemoryMemoryStore。

当前只执行第八阶段第2批：

1.区分HITL恢复Checkpoint与线程短期记忆Checkpoint。
2.扩展现有CheckpointStore的线程查询能力。
3.为InMemoryCheckpointStore建立用户线程隔离索引。
4.定义可持久化的ThreadConversationState。
5.实现ThreadConversationState与AgentCheckpoint之间的映射。
6.实现线程状态保存和加载服务。
7.实现threadId生成与格式校验。
8.实现同一用户同一threadId的进程内执行互斥基础设施。
9.验证不同用户和不同threadId不会串读。

本批不接入ReactAgentEngine，不接入SupervisorEngine，不修改正式Controller，不修改请求DTO，不修改HITL恢复流程，不新增前端。

# 一、强制边界

必须遵守：

1.不得重新创建`AgentCheckpoint`。
2.不得重新创建`CheckpointStore`。
3.不得重新创建`InMemoryCheckpointStore`。
4.不得新增`ShortTermMemoryStore`。
5.不得新增`ConversationStore`。
6.不得使用MemoryStore保存会话状态。
7.不得使用CheckpointStore保存长期用户偏好。
8.不得修改ReAct图结构。
9.不得修改Supervisor图结构。
10.不得修改ReasonNode。
11.不得修改正式Agent和Supervisor API。
12.不得修改HITL恢复执行流程。
13.不得修改前端。
14.不得提前实现同threadId多轮调用。
15.不得在未读完提示词前开始修改代码。
16.不得用编译通过代替功能验收。

# 二、执行前检查

必须先检查以下真实代码：

1.`AgentCheckpoint`所在模块和全部字段。
2.`CheckpointStatus`全部枚举值。
3.`CheckpointStore`现有接口方法。
4.`InMemoryCheckpointStore`内部主索引。
5.`InMemoryCheckpointStore`已有辅助索引。
6.按runId保存、加载和删除的语义。
7.按threadId查询是否已经存在。
8.Checkpoint版本更新方法。
9.HITL创建Checkpoint的全部调用点。
10.`CheckpointValidator`。
11.`CheckpointIdGenerator`。
12.`ExecutionType`或现有执行类型枚举。
13.`ReactCheckpointStateMapper`。
14.`ReactAgentState`和`SupervisorAgentState`。
15.`ContextWindowSnapshot`。
16.`ContextProcessingTrace`。
17.`AgentMessage`父类型。
18.现有Clock Bean。
19.现有错误码。
20.现有基础设施配置类。
21.第八阶段第1批MemoryStore代码。

必须确认以下类型真实存在：

- `AgentCheckpoint`
- `CheckpointStore`
- `InMemoryCheckpointStore`
- `CheckpointValidator`
- `CheckpointIdGenerator`

任一类型不存在时立即停止，不能创建近似替代品。

如果已有线程查询或Checkpoint用途字段：

1.检查是否完全满足本批语义。
2.完全等价则复用。
3.部分等价则最小扩展。
4.不得再创建第二套相同概念。

# 三、固定标识语义

必须保持：

```text
sessionId：认证当前调用者
threadId：一段持续多轮的会话
runId：某一次具体Agent执行
checkpointId：某一份状态快照
```

本批只建立threadId短期状态存储能力。

要求：

1.threadId不是sessionId。
2.threadId不是runId。
3.threadId不得编码sessionId。
4.threadId不得编码用户密码或权限。
5.长期记忆不得使用threadId作为用户命名空间。
6.线程Checkpoint查询必须同时使用userId和threadId。
7.不能只通过threadId查询任意用户状态。
8.不同用户允许拥有文本相同的threadId。
9.不同用户同名threadId必须保存为独立状态。
10.客户端是否能够提交threadId留到第3批实现。

# 四、CheckpointPurpose

在`agent-core`现有Checkpoint领域包中必须新增或复用：

`CheckpointPurpose`

枚举值名称固定：

```java
HITL_RECOVERY,
THREAD_MEMORY
```

语义：

## HITL_RECOVERY

用于：

1.某一次runId中断。
2.危险工具审批。
3.从中断节点恢复。
4.保存PendingApproval。
5.保存执行游标和临时运行状态。

## THREAD_MEMORY

用于：

1.保存一次正常执行结束后的稳定会话状态。
2.支持同一threadId后续调用续接。
3.保存完整会话消息。
4.保存上下文窗口Snapshot。
5.按userId+threadId加载。
6.不得保存PendingApproval。
7.不得表示Java代码行级恢复。

要求：

1.不得用metadata字符串替代CheckpointPurpose。
2.不得用CheckpointStatus代替CheckpointPurpose。
3.不得用ExecutionType代替CheckpointPurpose。
4.ExecutionType继续表示REACT_AGENT或SUPERVISOR。
5.所有现有HITL Checkpoint必须明确属于HITL_RECOVERY。
6.不得把旧HITL状态默认为THREAD_MEMORY。
7.新增THREAD_MEMORY时必须显式设置purpose。
8.不得增加第三个本批未要求的purpose。

# 五、最小扩展AgentCheckpoint

必须最小修改现有：

`AgentCheckpoint`

增加：

```java
CheckpointPurpose purpose
```

要求：

1.purpose不能为空。
2.保持AgentCheckpoint不可变。
3.不得删除现有字段。
4.不得改变runId的原有语义。
5.不得改变version的原有语义。
6.不得改变HITL状态字段。
7.不得创建AgentCheckpointV2。
8.不得创建ThreadCheckpoint子类。
9.不得复制整个Checkpoint模型。
10.现有HITL构造调用必须补充HITL_RECOVERY。
11.新增兼容构造器时，兼容构造器只能安全地默认HITL_RECOVERY。
12.不得让普通调用在未指定purpose时默认THREAD_MEMORY。
13.序列化或stateData结构不得因新增字段损坏。

如果AgentCheckpoint已经存在等价purpose字段：

1.复用该字段。
2.确保其为强类型枚举。
3.确保枚举包含上述两个固定值。
4.不得再增加第二个用途字段。

# 六、CheckpointValidator扩展

必须修改现有：

`CheckpointValidator`

根据purpose执行不同校验。

## HITL_RECOVERY

保持现有第六阶段规则，不得弱化：

1.SUSPENDED状态需要PendingApproval。
2.恢复所需节点、状态数据和版本必须完整。
3.审批绑定信息必须有效。
4.继续允许现有SUSPENDED、RESUMING、FAILED等状态。
5.不得因本批修改破坏原HITL校验。

## THREAD_MEMORY

必须校验：

1.purpose必须为THREAD_MEMORY。
2.status必须为COMPLETED。
3.userId不能为空。
4.threadId不能为空。
5.runId不能为空。
6.executionType不能为空。
7.pendingApproval必须为空。
8.stateData不能为空。
9.stateData必须能够恢复ThreadConversationState。
10.不得包含待审批对象。
11.不得包含未完成工具调用。
12.不得包含执行异常对象。
13.不得包含完整UserSession。
14.不得包含Session ID。
15.不得使用HITL的SUSPENDED校验规则。

错误必须复用或新增：

`THREAD_CHECKPOINT_INVALID`

错误信息不得包含完整stateData和消息正文。

# 七、CheckpointStore线程查询方法

必须最小扩展现有：

`CheckpointStore`

新增以下方法，名称和参数固定：

```java
List<AgentCheckpoint> findByThreadId(
    String userId,
    String threadId,
    CheckpointPurpose purpose
);

Optional<AgentCheckpoint> loadLatestByThreadId(
    String userId,
    String threadId,
    CheckpointPurpose purpose
);
```

要求：

1.必须同时传入userId。
2.必须同时传入threadId。
3.必须传入purpose。
4.不得新增只传threadId的正式短期记忆方法。
5.不得返回其他用户Checkpoint。
6.不得返回其他purpose的Checkpoint。
7.findByThreadId返回不可变List。
8.findByThreadId不存在数据时返回空List。
9.loadLatestByThreadId不存在时返回Optional.empty。
10.不得返回null。
11.不得删除现有按runId加载方法。
12.不得改变现有HITL按runId恢复语义。
13.Store接口不得依赖Spring。
14.Store接口不得依赖Servlet。
15.Store接口不得依赖MemoryStore。

最新Checkpoint选择规则：

1.优先比较updatedAt。
2.updatedAt较新的优先。
3.updatedAt相同时比较version。
4.version相同时使用checkpointId或runId稳定排序。
5.相同数据多次查询必须得到相同结果。
6.不得依赖ConcurrentHashMap遍历顺序。

# 八、ThreadCheckpointKey

在`agent-core`Checkpoint领域包中必须新增不可变：

`ThreadCheckpointKey`

字段名称固定：

- `userId`
- `threadId`
- `purpose`

要求：

1.三个字段均不能为空。
2.三个字段统一trim。
3.适合作为ConcurrentHashMap的Key。
4.必须具有稳定值相等语义。
5.不得包含sessionId。
6.不得包含runId。
7.不得包含executionType。
8.不得使用`userId + ":" + threadId`字符串拼接代替。
9.不得依赖Spring。
10.不得使用可变字段。

该Key只负责索引用户线程和用途。

Agent或Supervisor类型匹配由ThreadConversationState和加载服务校验。

# 九、InMemoryCheckpointStore索引

必须增量修改现有：

`InMemoryCheckpointStore`

保留原有runId或checkpointId主索引。

新增线程辅助索引，建议结构：

```java
ConcurrentHashMap<ThreadCheckpointKey, Set<String>>
```

集合保存checkpointId或现有主索引ID。

允许使用其他线程安全结构，但必须满足全部行为。

要求：

1.不得替换或破坏原主索引。
2.不得使用static Map。
3.不得使用普通HashMap。
4.不得使用全局锁串行全部请求。
5.save时更新主索引和线程索引。
6.delete时同步清理线程索引。
7.版本更新时保持线程索引一致。
8.purpose发生变化时正确移除旧索引并加入新索引。
9.辅助索引不得保留已删除Checkpoint ID。
10.索引集合为空时删除ThreadCheckpointKey。
11.不得让索引Key无限残留。
12.findByThreadId不得返回内部集合视图。
13.findByThreadId必须返回不可变快照。
14.findByThreadId必须过滤不存在的主索引记录。
15.loadLatestByThreadId使用稳定比较器。
16.不同userId相同threadId必须进入不同索引。
17.HITL_RECOVERY和THREAD_MEMORY必须进入不同索引。
18.待审批查询不得返回THREAD_MEMORY。
19.按runId加载现有功能必须保持不变。
20.同一个Bean必须支持并发访问。

原子性要求：

1.不得出现主索引保存成功但线程索引永久缺失。
2.不得出现删除主记录后线程索引仍永久指向旧ID。
3.允许在单进程内使用细粒度同步或ConcurrentHashMap原子方法。
4.不得引入数据库事务。
5.不得引入分布式锁。

# 十、ThreadConversationState

在`agent-runtime`合适的memory或checkpoint包中必须新增不可变：

`ThreadConversationState`

字段名称固定：

- `executionType`
- `participantName`
- `messages`
- `contextWindowSnapshot`
- `latestContextTrace`
- `lastCompletedRunId`
- `updatedAt`

要求：

1.executionType不能为空。
2.participantName不能为空并trim。
3.messages不能为空。
4.messages保存完整会话历史。
5.messages构造时复制为不可变List。
6.messages中不得存在null。
7.contextWindowSnapshot使用Optional或项目统一可选值。
8.latestContextTrace使用Optional或项目统一可选值。
9.lastCompletedRunId不能为空。
10.updatedAt不能为空。
11.不得包含UserSession。
12.不得包含sessionId。
13.不得包含roles或permissions。
14.不得包含RunContext。
15.不得包含PendingApproval。
16.不得包含pendingToolCalls。
17.不得包含Tool执行游标。
18.不得包含executionBuffer。
19.不得包含异常对象。
20.不得包含最终HTTP响应。
21.不得包含LangGraph4j CompiledGraph。
22.不得包含Spring AI Message。
23.不得包含MemoryEntry。
24.不得使用Map替代固定字段。

participantName语义：

- REACT_AGENT时保存agentName。
- SUPERVISOR时保存supervisorName。

本批只定义和存储该模型，不从实际Engine提取状态。

# 十一、ThreadCheckpointStateKeys

在`agent-runtime`必须新增常量类：

`ThreadCheckpointStateKeys`

至少包含名称固定的键：

```java
THREAD_CONVERSATION_STATE
```

要求：

1.不得在多个类散落字符串。
2.不得使用可修改public集合。
3.不得使用用户输入作为StateKey。
4.不得把userId作为stateData键。
5.不得把消息正文拼接到键名。
6.如现有统一StateKey体系适合复用，可以接入，但固定键名必须存在。

# 十二、ThreadCheckpointStateMapper

在`agent-runtime`必须新增纯Java：

`ThreadCheckpointStateMapper`

本批职责仅限于：

1.将ThreadConversationState写入Checkpoint stateData。
2.从THREAD_MEMORY Checkpoint恢复ThreadConversationState。
3.校验stateData中的类型。
4.不得从ReactAgentState提取。
5.不得从SupervisorAgentState提取。
6.不得修改Engine。
7.不得访问CheckpointStore。

必须提供以下方法或等价签名：

```java
Map<String, Object> toStateData(
    ThreadConversationState state
);

ThreadConversationState fromCheckpoint(
    AgentCheckpoint checkpoint
);
```

要求：

1.toStateData返回不可变Map。
2.只写入THREAD_CONVERSATION_STATE。
3.不得保存完整RunContext。
4.不得保存Session。
5.不得保存权限。
6.不得保存PendingApproval。
7.不得复制一份消息到其他stateData键。
8.fromCheckpoint必须校验purpose=THREAD_MEMORY。
9.fromCheckpoint必须校验status=COMPLETED。
10.fromCheckpoint必须校验值为ThreadConversationState。
11.缺失或类型错误时抛THREAD_CHECKPOINT_INVALID。
12.不得进行不安全的任意反射转换。
13.不得通过JSON字符串绕过类型校验。
14.不得修改原stateData。
15.保持无状态和线程安全。

第3批再扩展从完成后的ReactAgentState提取ThreadConversationState。

第4批再扩展Supervisor和HITL恢复后的提取。

# 十三、ThreadConversationCheckpointService

在`agent-runtime`必须新增纯Java：

`ThreadConversationCheckpointService`

依赖：

- `CheckpointStore`
- `CheckpointValidator`
- `ThreadCheckpointStateMapper`
- `CheckpointIdGenerator`
- `Clock`

必须提供：

```java
Optional<ThreadConversationState> load(
    String userId,
    String threadId,
    ExecutionType executionType,
    String participantName
);

AgentCheckpoint save(
    String userId,
    String threadId,
    String runId,
    ThreadConversationState state
);
```

如项目中的执行类型名称不是ExecutionType，使用实际强类型，但不得改用字符串。

## load流程

1.校验userId。
2.校验threadId。
3.校验executionType。
4.校验participantName。
5.调用：
   `loadLatestByThreadId(userId,threadId,THREAD_MEMORY)`
6.不存在返回Optional.empty。
7.校验Checkpoint.userId等于输入userId。
8.校验Checkpoint.threadId等于输入threadId。
9.校验purpose=THREAD_MEMORY。
10.校验status=COMPLETED。
11.通过ThreadCheckpointStateMapper恢复。
12.校验state.executionType匹配。
13.校验state.participantName匹配。
14.不匹配时抛THREAD_PARTICIPANT_MISMATCH。
15.不得加载其他用户状态。
16.不得加载HITL_RECOVERY。
17.不得修改加载出的状态。
18.不得访问MemoryStore。

## save流程

1.校验userId、threadId、runId和state。
2.校验state.lastCompletedRunId等于runId。
3.生成新的checkpointId。
4.purpose固定为THREAD_MEMORY。
5.status固定为COMPLETED。
6.pendingApproval固定为空。
7.executionType来自state。
8.agentName或participant字段按现有AgentCheckpoint结构填充。
9.stateData来自ThreadCheckpointStateMapper。
10.createdAt和updatedAt使用Clock。
11.version初始值使用现有Checkpoint约定。
12.通过CheckpointValidator校验。
13.先保存新Checkpoint。
14.保存成功后查询同一userId、threadId、THREAD_MEMORY的旧Checkpoint。
15.只删除比新Checkpoint旧的THREAD_MEMORY记录。
16.不得删除刚保存的新Checkpoint。
17.不得删除HITL_RECOVERY。
18.不得删除其他用户Checkpoint。
19.不得删除其他threadId。
20.清理旧记录失败时，新记录仍应可通过loadLatest加载。
21.不得先删除旧记录再保存新记录。
22.不得把ThreadConversationState写入MemoryStore。

本批不提供根据Controller请求创建ThreadConversationState的能力。

# 十四、ThreadIdGenerator

检查现有是否存在可复用的threadId生成器。

如果不存在，必须在稳定模块新增：

`ThreadIdGenerator`

方法名称固定：

```java
String generate();
```

在基础设施提供默认实现：

`UuidThreadIdGenerator`

要求：

1.使用UUID或等价不可预测ID。
2.不得使用递增整数。
3.不得使用当前时间戳作为唯一ID。
4.不得包含userId。
5.不得包含sessionId。
6.不得包含agentName。
7.不得访问CheckpointStore。
8.不得生成空字符串。
9.允许自定义Bean替换默认实现。

如果已有等价生成器：

1.直接复用。
2.不得增加第二个ThreadIdGenerator。
3.最终输出说明实际使用类型。

# 十五、ThreadIdValidator

在`agent-runtime`必须新增纯Java：

`ThreadIdValidator`

必须提供：

```java
void validate(String threadId);
```

规则：

1.threadId不能为空。
2.trim后不能为空。
3.最大长度固定或配置为128。
4.只允许：
   - 英文字母
   - 数字
   - `-`
   - `_`
5.不得包含空白。
6.不得包含 `/`。
7.不得包含 `\`。
8.不得包含 `..`路径语义。
9.不得包含换行。
10.不得自动修正非法ID。
11.非法时抛INVALID_THREAD_ID。
12.错误信息不得回显完整超长输入。
13.保持无状态和线程安全。

本批仅实现校验器，不修改HTTP请求DTO。

# 十六、ThreadExecutionCoordinator

在`agent-runtime`必须新增线程安全：

`ThreadExecutionCoordinator`

必须新增：

`ThreadExecutionLease implements AutoCloseable`

建议接口：

```java
ThreadExecutionLease acquire(
    String userId,
    String threadId
);
```

语义：

1.隔离Key为userId+threadId。
2.同一Key同一时刻只能有一个Lease。
3.不同threadId允许并发。
4.不同用户相同threadId允许并发。
5.Key已占用时抛THREAD_BUSY。
6.Lease关闭后释放。
7.close必须幂等。
8.Application Service将在第3批使用try-with-resources。
9.本批只实现基础设施，不接入执行链。

实现要求：

1.不得使用static Map。
2.不得使用ThreadLocal。
3.不得使用全局ReentrantLock阻塞全部线程。
4.可使用ConcurrentHashMap和原子占用标记。
5.释放后应清理无用Key。
6.不得让Map随已结束线程无限增长。
7.不得阻塞等待很长时间。
8.本批采用立即失败，不实现排队。
9.不得实现分布式锁。
10.同一Lease重复close不得抛异常。
11.不得允许错误Lease释放其他请求的占用。
12.不得记录Session ID。

允许新增内部不可变复合Key，但不得与ThreadCheckpointKey混淆用途。

# 十七、错误码

检查现有`AgentErrorCode`。

缺失时最小新增以下固定错误码：

- `INVALID_THREAD_ID`
- `THREAD_BUSY`
- `THREAD_PARTICIPANT_MISMATCH`
- `THREAD_CHECKPOINT_INVALID`

语义：

## INVALID_THREAD_ID

threadId格式非法。

## THREAD_BUSY

同一userId和threadId已经存在执行Lease。

## THREAD_PARTICIPANT_MISMATCH

存储的Agent或Supervisor名称、执行类型与请求期望不一致。

## THREAD_CHECKPOINT_INVALID

THREAD_MEMORY Checkpoint结构损坏或不满足稳定状态约束。

要求：

1.不得新增多个同义错误码。
2.不得把THREAD_BUSY转换为INTERNAL_ERROR。
3.不得把THREAD_CHECKPOINT_INVALID转换为MEMORY_STORE_FAILED。
4.不得把Checkpoint错误转换为长期MemoryStore错误。
5.错误信息不得包含完整消息。
6.错误信息不得包含stateData。
7.错误信息不得包含Session ID。
8.保留原始cause供服务端诊断。

本批不新增THREAD_NOT_FOUND。

不存在状态时load返回Optional.empty。

第3批正式API接入时再定义THREAD_NOT_FOUND的HTTP语义。

# 十八、Spring Bean装配

在`agent-infrastructure`现有Checkpoint或Memory配置类中装配：

- `ThreadCheckpointStateMapper`
- `ThreadConversationCheckpointService`
- `ThreadIdValidator`
- `ThreadExecutionCoordinator`
- `ThreadIdGenerator`默认实现（仅在不存在时）

要求：

1.runtime类不得添加Spring注解。
2.使用`@Configuration`和`@Bean`。
3.复用现有CheckpointStore Bean。
4.不得创建第二个CheckpointStore。
5.不得创建第二个InMemoryCheckpointStore。
6.ThreadIdGenerator使用`@ConditionalOnMissingBean`。
7.不得注入ReactAgentEngine。
8.不得注入SupervisorEngine。
9.不得注入ModelInvocationGateway。
10.不得注入MemoryStore到ThreadConversationCheckpointService。
11.不得产生循环依赖。
12.无模型配置时这些Bean仍能装配。
13.不得在bootstrap主类手工new。
14.不得创建Fake Engine或Fake CheckpointStore。

本批不增加新的配置文件字段，避免与现有HITL Checkpoint开关冲突。

# 十九、现有HITL兼容性

修改CheckpointPurpose后，必须检查所有HITL Checkpoint创建路径。

必须确保：

1.HITL中断保存的Checkpoint purpose=HITL_RECOVERY。
2.审批查询只返回HITL_RECOVERY。
3.恢复协调器只处理HITL_RECOVERY。
4.审批决定服务不得加载THREAD_MEMORY。
5.恢复完成时现有删除逻辑只删除目标HITL Checkpoint。
6.不得删除同threadId的THREAD_MEMORY。
7.现有PendingApproval查询行为不变。
8.现有runId恢复行为不变。
9.现有version条件更新行为不变。
10.现有危险工具审批指纹行为不变。

本批只做purpose字段兼容调整。

禁止在本批新增“HITL恢复完成后保存THREAD_MEMORY”逻辑。

该逻辑放到第4批。

# 二十、短期与长期记忆隔离

必须确认：

## CheckpointStore

存储：

- HITL恢复状态
- 线程完整消息状态
- ContextWindowSnapshot
- ContextProcessingTrace

隔离：

- userId+threadId
- HITL按runId恢复

## MemoryStore

存储：

- PROFILE
- PREFERENCE
- FACT
- RULE

隔离：

- userId+namespace+key

禁止：

1.ThreadConversationCheckpointService依赖MemoryStore。
2.LongTermMemoryApplicationService依赖CheckpointStore。
3.ThreadConversationState包含MemoryEntry。
4.MemoryEntry包含AgentCheckpoint。
5.把完整消息写入MemoryStore。
6.把用户偏好写进ThreadCheckpointStateKeys。
7.把CheckpointPurpose加入MemoryCategory。
8.让MemoryStore继承CheckpointStore。
9.让CheckpointStore继承MemoryStore。
10.新增无类型GenericStore统一两者。

# 二十一、多用户隔离场景

必须通过代码检查或现有可执行能力确认：

## 场景1：相同用户和线程

保存：

```text
userId=user-A
threadId=thread-1
purpose=THREAD_MEMORY
```

预期：

1.可以通过相同userId和threadId加载。
2.返回最新稳定状态。
3.不得返回HITL_RECOVERY记录。

## 场景2：不同用户相同threadId

保存：

```text
user-A/thread-1
user-B/thread-1
```

预期：

1.形成两个ThreadCheckpointKey。
2.A只能加载A。
3.B只能加载B。
4.两者不会覆盖。
5.删除A状态不影响B。

## 场景3：同用户不同threadId

保存：

```text
user-A/thread-1
user-A/thread-2
```

预期：

1.两个线程独立。
2.thread-1查询不得返回thread-2。
3.线程索引集合不共享。

## 场景4：不同purpose

同一userId和threadId存在：

- HITL_RECOVERY
- THREAD_MEMORY

预期：

1.两类Checkpoint可以同时存在。
2.loadLatestByThreadId传THREAD_MEMORY只返回线程状态。
3.审批查询只返回HITL_RECOVERY。
4.删除其中一种不得删除另一种。

## 场景5：最新记录选择

同一线程保存多个THREAD_MEMORY Checkpoint。

预期：

1.loadLatest稳定选择updatedAt最新记录。
2.updatedAt相同按version和ID稳定选择。
3.不依赖Map遍历顺序。

## 场景6：非法线程状态

THREAD_MEMORY包含PendingApproval或状态不是COMPLETED。

预期：

1.CheckpointValidator拒绝。
2.抛THREAD_CHECKPOINT_INVALID或已有明确校验错误。
3.不得保存非法状态。

## 场景7：participant不匹配

保存的是：

```text
executionType=REACT_AGENT
participantName=utility_agent
```

load期望：

```text
calculator_agent
```

预期：

1.抛THREAD_PARTICIPANT_MISMATCH。
2.不得返回消息。
3.不得修改Checkpoint。

## 场景8：并发Lease

同一userId和threadId连续acquire。

预期：

1.第一次成功。
2.第二次抛THREAD_BUSY。
3.第一次close后可再次acquire。
4.重复close安全。
5.不同threadId可同时acquire。

不得为了验证新增隐藏Controller或测试脚本。

# 二十二、性能和线程安全

1.InMemoryCheckpointStore查询不得依赖不稳定Map顺序。
2.主索引和线程索引均必须线程安全。
3.ThreadConversationCheckpointService保持无状态。
4.ThreadCheckpointStateMapper保持无状态。
5.ThreadIdValidator保持无状态。
6.ThreadExecutionCoordinator支持并发。
7.不得使用static可变集合。
8.不得使用ThreadLocal。
9.不得缓存当前用户。
10.不得缓存上一次加载的ThreadConversationState。
11.不得跨用户共享消息对象。
12.返回集合必须为不可变快照。
13.不得为所有Checkpoint操作添加单个全局锁。
14.不得引入数据库、Redis或文件锁。
15.线程查询目标复杂度应基于辅助索引，而不是每次遍历全部用户数据。

# 二十三、日志与安全

允许记录：

- checkpointId
- runId
- userId
- threadId
- purpose
- executionType
- participantName
- version
- 操作类型
- 成功或失败

禁止记录：

- Session ID
- 完整messages
- Summary内容
- Tool参数
- ToolResult正文
- PendingApproval payload
- stateData完整内容
- UserSession
- roles
- permissions
- MemoryEntry value

要求：

1.错误响应不得返回其他用户是否拥有相同threadId。
2.本批没有HTTP接口，不新增HTTP响应DTO。
3.日志中的threadId可以使用短标识或现有安全日志策略。
4.Checkpoint损坏可记录服务端错误，但不能打印完整状态。

# 二十四、本批禁止实现

明确禁止：

1.ReactAgentEngine线程续接。
2.SupervisorEngine线程续接。
3.修改AuthenticatedAgentApplicationService。
4.修改AuthenticatedSupervisorApplicationService。
5.正式Agent请求增加threadId。
6.正式Supervisor请求增加threadId。
7.THREAD_NOT_FOUND HTTP映射。
8.HITL恢复完成后保存线程状态。
9.挂起线程阻止普通请求。
10.从ReactAgentState提取ThreadConversationState。
11.从SupervisorAgentState提取ThreadConversationState。
12.自动生成并保存新线程。
13.前端线程列表。
14.前端会话续接。
15.记忆管理页面。
16.长期记忆注入模型。
17.LLM自动提取用户偏好。
18.记忆工具。
19.数据库。
20.Redis。
21.文件持久化。
22.分布式锁。
23.Checkpoint TTL。
24.隐藏调试Controller。
25.测试脚本、README和Git操作。

# 二十五、编译验收

必须执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

必须逐项确认：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.`CheckpointPurpose`真实存在。
4.CheckpointPurpose包含HITL_RECOVERY和THREAD_MEMORY。
5.AgentCheckpoint包含强类型purpose。
6.现有HITL创建点显式使用HITL_RECOVERY。
7.`ThreadCheckpointKey`真实存在。
8.CheckpointStore包含两个固定线程查询方法。
9.InMemoryCheckpointStore维护线程辅助索引。
10.线程查询同时使用userId、threadId和purpose。
11.loadLatest选择规则稳定。
12.`ThreadConversationState`真实存在。
13.`ThreadCheckpointStateKeys`真实存在。
14.`ThreadCheckpointStateMapper`真实存在。
15.`ThreadConversationCheckpointService`真实存在。
16.`ThreadIdValidator`真实存在。
17.ThreadIdGenerator存在或正确复用。
18.`ThreadExecutionCoordinator`真实存在。
19.`ThreadExecutionLease`真实存在。
20.THREAD_MEMORY只允许COMPLETED且无PendingApproval。
21.HITL_RECOVERY现有恢复行为未被破坏。
22.CheckpointStore和MemoryStore仍为独立抽象。
23.没有修改ReAct和Supervisor执行链。
24.没有修改正式API。
25.没有修改前端。
26.没有数据库或Redis。
27.git diff没有无关修改。

编译通过后，必须重新搜索代码确认：

1.不存在第二个CheckpointStore。
2.不存在ShortTermMemoryStore。
3.不存在ConversationStore。
4.不存在使用自由字符串表示CheckpointPurpose的地方。
5.不存在只按threadId执行短期记忆查询的新调用。
6.不存在THREAD_MEMORY进入审批查询的情况。

# 二十六、最终输出

最终输出必须逐项对应实施清单，只输出：

1.新增和修改文件清单。
2.CheckpointPurpose的两种用途。
3.AgentCheckpoint兼容调整。
4.CheckpointValidator新增规则。
5.CheckpointStore新增方法签名。
6.InMemoryCheckpointStore辅助索引结构。
7.loadLatest稳定选择规则。
8.ThreadConversationState字段。
9.ThreadCheckpointStateMapper映射规则。
10.ThreadConversationCheckpointService保存和加载流程。
11.ThreadId生成和校验。
12.ThreadExecutionCoordinator并发语义。
13.HITL兼容性检查结果。
14.CheckpointStore与MemoryStore分层情况。
15.多用户和多线程隔离检查结果。
16.编译和打包结果。
17.未完成验证及准确原因。
18.发现但未处理的问题。

不得把计划描述为已完成。

不得只输出“编译通过”。

不得遗漏任一强制交付类型。

编译或打包失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE8_BATCH2_END===
````
