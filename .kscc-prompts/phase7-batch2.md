你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须先完整阅读并遵守根目录`AGENTS.md`。其中已有模块边界、LangGraph4j、不可变性、安全、编译和输出规范，本提示词不重复展开。

已完成第七阶段第1批：

1.ContextTrimmer稳定接口。
2.ContextTrimRequest和ContextTrimResult。
3.ContextMessageGroup及工具消息原子分组。
4.ContextMessageHistoryValidator。
5.MessageCountContextTrimmer。
6.System消息永久保留。
7.ToolCall与ToolResult完整配对。
8.最新用户输入保护。
9.TokenCounter接口。
10.HeuristicTokenCounter默认估算实现。
11.上下文配置和基础查询能力。

当前只执行第七阶段第2批：

1.建立模型上下文Token预算模型。
2.实现按Token数量裁剪。
3.保证System消息和工具消息原子组完整。
4.实现消息数裁剪与Token裁剪的统一流水线。
5.输出完整Token占用和裁剪诊断。
6.扩展上下文配置及框架查询能力。
7.为第3批LLM摘要压缩提供稳定入口。

本批不调用LLM摘要，不接入ReAct或Supervisor ReasonNode，不修改前端，不实现MemoryStore。

# 一、执行前检查

先检查：

1.根目录`AGENTS.md`。
2.第七阶段第1批实际生成的全部代码。
3.ContextTrimmer真实接口。
4.ContextTrimRequest和ContextTrimResult真实字段。
5.MessageCountContextTrimmer。
6.ContextMessageGrouper。
7.ContextMessageHistoryValidator。
8.ContextMessageGroup及groupType。
9.TokenCounter和HeuristicTokenCounter。
10.AgentMessage及全部子类型。
11.ToolCall和ToolAgentMessage配对字段。
12.现有上下文配置属性。
13.FrameworkQueryService及上下文查询接口。
14.AgentErrorCode和AgentFrameworkException。
15.当前Java版本。
16.当前集合不可变处理方式。
17.现有Spring Bean条件装配方式。

要求：

1.以第1批实际实现为准增量开发。
2.已有类型可扩展时不得创建语义重复的V2模型。
3.不得复制消息分组和工具配对算法。
4.按Token裁剪必须复用ContextMessageGrouper。
5.非法历史必须复用ContextMessageHistoryValidator。
6.前序代码存在阻断问题时只做最小修复。
7.不得提前接入ReasonNode和模型调用链。
8.不得提前实现LLM摘要。

# 二、Token预算语义

模型总上下文窗口不仅包含历史消息，还需要为模型输出及协议开销预留空间。

本批统一采用：

```text
有效消息预算
=模型上下文窗口
-预留输出Token
-协议及工具定义预留Token
-安全余量Token
```

必须满足：

```text
maxContextTokens
>reservedOutputTokens
+reservedProtocolTokens
+safetyMarginTokens
```

定义不可变：

`ContextTokenBudget`

至少包含：

- maxContextTokens
- reservedOutputTokens
- reservedProtocolTokens
- safetyMarginTokens
- availableMessageTokens

要求：

1.所有字段必须大于等于0。
2.maxContextTokens必须大于0。
3.availableMessageTokens由其他字段计算，客户端不能直接指定。
4.availableMessageTokens必须大于0。
5.使用long进行中间计算，防止int溢出。
6.最终Token数量使用int时必须检查范围。
7.不得依赖具体模型供应商类。
8.不得把API Key和模型凭证放入预算模型。
9.模型保持不可变。
10.不得使用Map表达固定预算字段。

# 三、ContextTokenBudgetCalculator

在agent-runtime实现纯Java：

`ContextTokenBudgetCalculator`

建议方法：

```java
ContextTokenBudget calculate(
    int maxContextTokens,
    int reservedOutputTokens,
    int reservedProtocolTokens,
    int safetyMarginTokens
);
```

职责：

1.校验全部配置。
2.计算availableMessageTokens。
3.配置非法时抛INVALID_CONTEXT_CONFIGURATION。
4.不得自动把负数修正为0。
5.不得自动放大模型窗口。
6.不得调用TokenCounter。
7.不得调用模型。
8.保持无状态和线程安全。

如果第1批已有等价预算类型，最小扩展并复用。

# 四、Token裁剪请求

检查第1批ContextTrimRequest。

优先采用最小兼容方案，不破坏MessageCountContextTrimmer。

可以选择：

方案一：为ContextTrimRequest增加可选的ContextTokenBudget，并提供兼容工厂方法。

方案二：新增明确的`TokenContextTrimRequest`，但不得复制messages和公共校验逻辑。

推荐字段：

- messages
- tokenBudget

可选：

- additionalReservedTokens

要求：

1.messages为不可变快照。
2.tokenBudget不能为空。
3.additionalReservedTokens用于单次请求额外预留，默认0。
4.额外预留不得为负。
5.最终可用消息Token：
`tokenBudget.availableMessageTokens-additionalReservedTokens`
6.最终可用消息Token必须大于0。
7.不得允许调用者直接提交estimatedTokenCount。
8.不得包含Spring AI类型。
9.不得包含RunContext或UserSession。
10.不得使用无类型Map表达限制。

# 五、Token裁剪结果

复用或最小扩展ContextTrimResult。

至少能够表达：

- originalMessageCount
- retainedMessageCount
- removedMessageCount
- estimatedTokensBefore
- estimatedTokensAfter
- maxContextTokens
- availableMessageTokens
- additionalReservedTokens
- effectiveMessageBudget
- tokenUtilizationBefore
- tokenUtilizationAfter
- withinTokenBudget
- retainedSystemMessageCount
- retainedNonSystemMessageCount
- diagnostics

要求：

1.tokenUtilization使用double或明确比例值。
2.除预算为0外不得出现NaN或Infinity。
3.正常结果withinTokenBudget必须为true。
4.不得通过返回超预算结果表示成功。
5.不得返回完整原始Prompt副本。
6.retainedMessages保持不可变和原始顺序。
7.诊断集合不可变。
8.不得包含用户身份和权限。
9.不得包含完整工具参数副本。
10.不得依赖Spring。

如ContextTrimResult当前不适合表达两种策略，可增加统一的`ContextProcessingResult`，但不得复制所有消息模型。

# 六、TokenCountContextTrimmer

在agent-runtime实现：

`TokenCountContextTrimmer`

优先实现现有ContextTrimmer接口；若接口无法承载Token预算，以最小方式扩展接口或增加专用接口。

依赖：

- ContextMessageHistoryValidator
- ContextMessageGrouper
- TokenCounter

算法：

1.校验请求和预算。
2.验证消息历史。
3.将消息划分为原子组。
4.计算全部消息原始Token数。
5.提取全部System组。
6.计算System组Token总量。
7.计算有效消息预算。
8.System组永久保留。
9.从最后一个非System原子组向前选择。
10.每次只能选择完整原子组。
11.加入后不超过有效预算才保留。
12.最终按照原始消息下标恢复顺序。
13.计算裁剪后Token数。
14.确认结果不超过有效预算。
15.输入列表不得被修改。

# 七、System消息预算规则

System消息永久保留，但Token预算是硬限制。

规则：

1.全部System消息必须计入Token预算。
2.System消息不允许被裁剪。
3.如果System消息Token总量已经大于有效消息预算：
- 立即抛CONTEXT_BUDGET_EXCEEDED。
- 不得删除System消息。
- 不得截断System消息内容。
- 不得返回超预算结果继续调用模型。
  4.System消息等于预算时：
- 只保留System消息。
- 不再保留非System消息。
  5.System消息保持全局原始顺序。
  6.不得合并多条System消息。
  7.不得把System消息移到错误位置。
  8.不得把权限和Session信息注入System消息。

# 八、工具原子组预算规则

ToolCall与对应ToolResult必须整体处理。

规则：

1.TOOL_INTERACTION组Token数等于组内全部消息Token之和。
2.组内Assistant ToolCall及全部ToolAgentMessage必须一起保留或删除。
3.不得只保留Assistant ToolCall。
4.不得只保留部分ToolResult。
5.不得按单条消息分别决定工具组。
6.不得按toolName模糊关联。
7.不得修改ToolCall参数以缩短Token。
8.不得截断ToolResult内容。
9.内容级压缩留到第3批摘要，不在本批实现。

如果最新必须保留的工具原子组本身无法放入剩余预算：

1.抛CONTEXT_BUDGET_EXCEEDED。
2.不得使用MessageCount裁剪中的atomicGroupOvershoot继续超预算。
3.Token预算属于硬限制。
4.错误中可以说明“最新工具交互超过可用上下文预算”。
5.错误中不得包含完整工具参数和结果。

# 九、最新用户输入保护

最后一条真实UserAgentMessage必须优先保留。

算法要求：

1.识别最后一条UserAgentMessage。
2.确定它所在原子组。
3.将该组视为强制保留组。
4.System组和该强制组必须先计入预算。
5.剩余预算再从最新到最旧选择其他组。
6.不得为了旧Assistant消息删除最新User消息。
7.不得根据content前缀判断消息类型。
8.Supervisor内部观察消息不得误认为真实用户输入。
9.如果没有User消息，不强制保留用户组。
10.如果System组加最新用户组超过预算：
- 抛CONTEXT_BUDGET_EXCEEDED。
- 不得静默删除最新用户输入。
11.如果最新用户消息之后存在完整工具交互或Assistant结果，保持它们的原子关系和时间顺序。
12.不得生成不符合原始顺序的消息列表。

# 十、选择算法

建议采用确定性原子组选择算法：

1.构建全部ContextMessageGroup。
2.标记全部SYSTEM组为mandatory。
3.标记最后User消息所在组为mandatory。
4.如最后User消息之后存在不能拆分的当前轮工具组，根据现有消息结构将其纳入mandatory。
5.计算mandatory组Token总量。
6.超过预算则明确失败。
7.从最后一个组向前遍历。
8.跳过已经mandatory的组。
9.加入后不超过预算则保留。
10.加入后超过预算则跳过该旧组。
11.继续检查更旧组没有实际意义时可停止，但行为必须确定。
12.最终按startIndex排序输出。
13.再次验证工具配对。
14.再次Token计数。
15.最终Token不得超过预算。

要求：

1.算法不能依赖HashSet无序迭代决定输出。
2.相同输入和预算得到相同结果。
3.不得使用贪心算法之外的复杂全局优化。
4.优先保留最近消息，而不是尝试用许多短旧消息替代较新的长消息。
5.不能拆分原子组。
6.不得修改原消息对象。

# 十一、严格预算与MessageCount差异

明确区分：

## MessageCountContextTrimmer

1.maxMessages是软目标。
2.最新原子组超过消息数时可完整保留。
3.通过atomicGroupOvershoot记录超出数量。

## TokenCountContextTrimmer

1.Token预算是硬上限。
2.不得使用Token overshoot。
3.强制消息无法放入时必须失败。
4.不能把超预算请求发送给模型。
5.不得通过estimatedTokensAfter伪造在预算内。

最终代码注释和诊断需体现该差异。

# 十二、Token计数一致性

TokenCountContextTrimmer必须只使用注入的TokenCounter。

禁止：

1.自己实现第二套字符估算。
2.使用String.length直接作为Token数。
3.在不同步骤使用不同计数规则。
4.裁剪前后漏算ToolCall参数。
5.漏算ToolAgentMessage内容。
6.对同一消息重复施加不一致固定开销。
7.按消息role猜测实际子类而不使用现有模型。

要求：

1.原始Token数由`tokenCounter.count(allMessages)`得到。
2.组Token数由`tokenCounter.count(group.messages())`得到。
3.最终Token数重新对retainedMessages完整计数。
4.不得只累加组缓存值而不做最终校验。
5.如果最终重新计数超过预算，抛CONTEXT_BUDGET_EXCEEDED。
6.TokenCounter异常转换为明确上下文错误。
7.不得记录完整消息内容。

# 十三、统一上下文处理请求

在agent-core增加或复用不可变：

`ContextProcessingRequest`

至少包含：

- messages
- maxMessages
- tokenBudget
- additionalReservedTokens
- messageCountTrimmingEnabled
- tokenTrimmingEnabled

要求：

1.messages不可变。
2.至少启用一种裁剪策略。
3.maxMessages仅在消息数裁剪启用时必须大于0。
4.tokenBudget仅在Token裁剪启用时必须存在。
5.不得加入摘要字段，本批尚未实现摘要。
6.不得包含模型客户端。
7.不得包含Session和权限。
8.不得允许调用者提交Token统计结果。
9.提供清晰工厂方法，避免构造参数顺序错误。
10.不要使用Builder引入大量可选null字段，优先record和静态工厂。

# 十四、统一上下文处理结果

增加或复用：

`ContextProcessingResult`

至少包含：

- processedMessages
- originalMessageCount
- processedMessageCount
- removedMessageCount
- originalTokenCount
- processedTokenCount
- effectiveMessageBudget
- messageCountTrimmed
- tokenTrimmed
- summaryApplied
- withinBudget
- diagnostics

本批：

`summaryApplied=false`

要求：

1.processedMessages不可变。
2.withinBudget必须反映最终真实计数。
3.不能只复制最后一个Trimmer的统计。
4.removedMessageCount按初始输入和最终输出计算。
5.originalTokenCount基于初始消息。
6.processedTokenCount基于最终消息。
7.不得返回中间可变集合。
8.不得包含完整Prompt字符串。
9.不得依赖Spring。
10.为第3批摘要预留结果语义，但不实现摘要内容。

# 十五、ContextProcessingPipeline

在agent-runtime实现纯Java：

`ContextProcessingPipeline`

或复用已有统一处理器。

依赖：

- MessageCountContextTrimmer
- TokenCountContextTrimmer
- TokenCounter

建议方法：

```java
ContextProcessingResult process(ContextProcessingRequest request);
```

默认执行顺序：

```text
消息历史验证
→按消息数裁剪
→按Token数裁剪
→最终Token校验
→输出统一结果
```

要求：

1.消息数裁剪关闭时跳过。
2.Token裁剪关闭时跳过。
3.两者都启用时必须先消息数后Token数。
4.不得先Token裁剪后又恢复已删除消息。
5.每一步输入均使用上一步输出。
6.最终再次验证工具配对。
7.最终再次计算Token数。
8.最终超过预算时明确失败。
9.不得调用LLM。
10.不得保存跨请求状态。
11.不得修改ReactAgentState。
12.不得访问CheckpointStore或MemoryStore。
13.保持无状态和线程安全。
14.不得吞掉某一步诊断。
15.诊断集合去重并保持稳定顺序。
16.不得返回null。

# 十六、处理诊断

在第1批诊断基础上增加或复用：

- MESSAGE_COUNT_TRIM_APPLIED
- TOKEN_TRIM_APPLIED
- TOKEN_BUDGET_NOT_EXCEEDED
- SYSTEM_TOKEN_BUDGET_EXCEEDED
- MANDATORY_CONTEXT_TOO_LARGE
- TOOL_GROUP_SKIPPED_FOR_TOKEN_BUDGET
- OLD_MESSAGES_REMOVED_FOR_TOKEN_BUDGET
- LATEST_USER_MESSAGE_PRESERVED
- FINAL_TOKEN_BUDGET_VERIFIED
- NO_TOKEN_TRIMMING_REQUIRED

要求：

1.诊断仅使用安全代码。
2.不得包含消息正文。
3.不得包含工具参数。
4.不得包含用户身份。
5.相同诊断不重复。
6.失败时通过异常错误码表达，不能只写diagnostic后继续。
7.成功结果不能包含表示超预算但仍发送模型的含糊状态。

# 十七、错误码

检查现有AgentErrorCode。

缺失时最小补充或复用：

- CONTEXT_BUDGET_EXCEEDED
- INVALID_CONTEXT_CONFIGURATION
- TOKEN_COUNT_FAILED
- CONTEXT_PROCESSING_FAILED

语义：

1.System消息超过预算：
CONTEXT_BUDGET_EXCEEDED。
2.System加最新用户输入超过预算：
CONTEXT_BUDGET_EXCEEDED。
3.最新工具原子组超过预算：
CONTEXT_BUDGET_EXCEEDED。
4.预算字段非法：
INVALID_CONTEXT_CONFIGURATION。
5.TokenCounter执行失败：
TOKEN_COUNT_FAILED。
6.普通未知流水线异常：
CONTEXT_PROCESSING_FAILED。

要求：

1.不得创建多个重复的TOKEN_LIMIT_EXCEEDED错误码。
2.不得将预算不足转换为MODEL_INVOCATION_FAILED。
3.不得将非法工具配对转换为Token错误。
4.异常信息不得包含完整消息和工具参数。
5.保留原异常cause供服务端诊断。

# 十八、配置扩展

在现有`agent.context`配置中增加：

```yaml
agent:
  context:
    enabled: true
    message-count-trimming-enabled: true
    max-messages: 20
    token-trimming-enabled: true
    max-context-tokens: 8192
    reserved-output-tokens: 1024
    reserved-protocol-tokens: 256
    safety-margin-tokens: 128
```

要求：

1.字段使用类型安全配置属性。
2.max-context-tokens必须大于0。
3.所有预留值必须大于等于0。
4.有效消息预算必须大于0。
5.max-messages必须大于0。
6.默认值集中定义。
7.不得在多个类中硬编码不同预算。
8.不得加入summary-threshold，本批不实现摘要。
9.不得加入模型API Key。
10.不得根据模型名称字符串隐式猜测窗口大小。
11.未来允许业务配置不同模型预算。
12.配置非法时启动失败并给出明确原因。

如果项目现有配置字段命名不同，保持现有命名风格。

# 十九、默认ContextProcessingPipeline Bean

在agent-infrastructure装配：

- ContextTokenBudgetCalculator
- TokenCountContextTrimmer
- ContextProcessingPipeline

继续复用：

- ContextMessageHistoryValidator
- ContextMessageGrouper
- MessageCountContextTrimmer
- TokenCounter

要求：

1.runtime实现不添加Spring注解。
2.使用`@Configuration`和`@Bean`。
3.不得创建第二个TokenCounter。
4.不得创建第二个ContextMessageGrouper。
5.不得重复注册消息数裁剪器。
6.默认Pipeline允许通过自定义Bean替换。
7.无模型配置时上下文Pipeline仍能装配。
8.本批不注入ModelInvocationGateway。
9.本批不注入ReactReasonNode。
10.不得产生Bean循环依赖。
11.不得在bootstrap启动类手工new。
12.不得创建Fake摘要器。

# 二十、框架查询能力

扩展现有上下文查询结果，至少展示：

- enabled
- messageCountTrimmingEnabled
- maxMessages
- tokenTrimmingEnabled
- maxContextTokens
- reservedOutputTokens
- reservedProtocolTokens
- safetyMarginTokens
- availableMessageTokens
- tokenCounterType
- exactTokenCount=false
- pipelineOrder

`pipelineOrder`显示：

```text
MESSAGE_COUNT,TOKEN_COUNT
```

要求：

1.不得暴露Bean完整包名。
2.不得暴露模型API Key。
3.不得暴露用户消息。
4.不得提供任意消息裁剪的公开调试API。
5.查询接口无模型配置也可使用。
6.继续复用现有FrameworkQueryService。
7.不得创建第二套框架查询Controller。

# 二十一、本批仍不接入模型调用链

不得修改以下运行行为：

1.DefaultReactReasonNode仍使用原有消息列表。
2.DefaultSupervisorReasonNode仍使用原有消息列表。
3.ModelInvocationGateway仍接收原有请求。
4.SpringAiModelClient行为不变。
5.ReactAgentState的messages不被裁剪覆盖。
6.SupervisorAgentState的messages不被裁剪覆盖。
7.Checkpoint保存完整State。
8.HITL恢复流程不裁剪Checkpoint。
9.Agent正式API请求结构不变。
10.前端对话页面不变。

原因：

1.第3批还需加入LLM摘要压缩。
2.第4批再统一接入ReAct和Supervisor。
3.避免一部分模型请求已裁剪、一部分未裁剪。
4.避免摘要前后出现两套运行行为。

本批只完成可独立验证的Token预算与流水线。

# 二十二、边界场景

通过代码检查或现有验证能力确认：

## 场景1：未超预算

输入Token低于有效消息预算。

预期：

1.消息不被Token裁剪。
2.processedTokenCount等于原始Token数。
3.withinBudget=true。
4.诊断包含NO_TOKEN_TRIMMING_REQUIRED。

## 场景2：普通长对话

System加多轮User/Assistant消息超过预算。

预期：

1.System全部保留。
2.最近消息优先。
3.旧消息被删除。
4.最终Token不超过预算。
5.最新User消息保留。

## 场景3：System过大

System消息本身超过预算。

预期：

1.抛CONTEXT_BUDGET_EXCEEDED。
2.不得删除或截断System。
3.不得返回成功结果。

## 场景4：最新User过大

System加最新User消息超过预算。

预期：

1.抛CONTEXT_BUDGET_EXCEEDED。
2.不得删除最新User输入。
3.不得静默超预算。

## 场景5：完整工具组

Assistant ToolCall和ToolResult整体可以放入预算。

预期：

1.全部保留。
2.配对完整。
3.顺序不变。

## 场景6：旧工具组放不下

较旧工具组加入后超过预算。

预期：

1.整个旧工具组被删除。
2.不得保留其中部分消息。
3.最终历史合法。

## 场景7：最新工具组过大

最新必须保留工具组本身超过预算。

预期：

1.明确失败。
2.不得使用overshoot继续执行。
3.不得拆分工具组。

## 场景8：消息数加Token双重裁剪

原始消息同时超过maxMessages和Token预算。

预期：

1.先按消息数裁剪。
2.再按Token裁剪。
3.最终统计以原始输入和最终输出为准。
4.最终不超过Token预算。

## 场景9：非法历史

存在孤立ToolResult或缺失ToolResult。

预期：

1.在裁剪前失败。
2.不得通过删除非法消息伪装修复。
3.错误语义仍为消息配对错误。

## 场景10：中文和英文

分别处理中文多轮和英文多轮。

预期：

1.Token估算稳定。
2.算法不依赖平台默认字符集。
3.最终预算判断一致。

不得为验证增加隐藏Controller或测试脚本。

# 二十三、性能要求

1.消息分组为O(n)。
2.Token计数调用次数应保持合理。
3.允许对单次process内部的原子组Token数进行局部缓存。
4.缓存仅存在于当前方法调用。
5.不得跨请求缓存AgentMessage或Token结果。
6.不得使用static缓存。
7.不得使用消息content作为全局缓存Key。
8.不得引入复杂动态规划。
9.不得为全部请求增加全局锁。
10.同一Pipeline Bean支持并发调用。

# 二十四、本批禁止实现

禁止：

1.LLM摘要压缩。
2.摘要Prompt。
3.Summary消息模型。
4.summaryThreshold配置。
5.摘要失败降级。
6.接入ReAct ReasonNode。
7.接入Supervisor ReasonNode。
8.修改ModelInvocationGateway。
9.修改SpringAiModelClient。
10.修改Checkpoint内容。
11.跨threadId消息存储。
12.MemoryStore。
13.长期用户偏好。
14.前端上下文页面。
15.SSE。
16.精确供应商Tokenizer网络调用。
17.数据库或Redis。
18.隐藏调试Controller。
19.测试脚本、README和Git操作。

# 二十五、编译验收

执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

确认：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.ContextTokenBudget模型保持框架无关。
4.TokenCountContextTrimmer复用统一消息分组。
5.TokenCountContextTrimmer只使用TokenCounter。
6.System消息永久保留。
7.最新用户输入保留。
8.ToolCall和ToolResult不被拆散。
9.Token预算属于硬上限。
10.强制上下文过大时明确失败。
11.最终Token重新计数并验证。
12.流水线顺序为消息数裁剪后Token裁剪。
13.ContextProcessingResult统计准确。
14.无模型配置时上下文Bean可装配。
15.框架查询显示预算和流水线配置。
16.没有调用LLM摘要。
17.没有接入ReAct和Supervisor。
18.现有认证、HITL、ACL和前端未被破坏。
19.git diff没有无关修改。

# 二十六、最终输出

只输出：

1.新增和修改文件清单。
2.ContextTokenBudget字段及计算公式。
3.Token裁剪请求和结果模型。
4.TokenCountContextTrimmer算法。
5.System消息预算不足处理。
6.最新用户输入保护方式。
7.工具原子组Token裁剪规则。
8.MessageCount与TokenCount硬软限制差异。
9.ContextProcessingPipeline执行顺序。
10.ContextProcessingResult统计字段。
11.新增错误码及触发条件。
12.配置及Spring Bean装配。
13.框架查询结果。
14.边界场景检查结果。
15.编译和打包结果。
16.发现但未处理的问题。

编译或打包失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE7_BATCH2_END===