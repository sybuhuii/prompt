你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须完整阅读并遵守根目录`AGENTS.md`。

已完成第七阶段前两批：

1.`ContextTrimRequest`、`ContextTrimResult`和`ContextTrimmer`。
2.`ContextMessageGroup`、`ContextMessageGroupType`和`ContextMessageGrouper`。
3.`ContextMessageHistoryValidator`。
4.`MessageCountContextTrimmer`。
5.`TokenCounter`和`HeuristicTokenCounter`。
6.`ContextTokenBudget`和预算计算器。
7.`TokenCountContextTrimmer`。
8.`ContextProcessingRequest`和`ContextProcessingResult`。
9.`ContextProcessingPipeline`。
10.System消息永久保留。
11.ToolCall和ToolResult原子配对。
12.消息数与Token数裁剪。

当前只执行第七阶段第3批：

1.定义框架生成的摘要消息。
2.实现摘要触发判断。
3.选择需要摘要的旧消息。
4.使用LLM生成摘要。
5.将旧摘要和旧消息合并成一条新摘要。
6.将摘要接入`ContextProcessingPipeline`。
7.摘要后继续执行消息数和Token数裁剪。
8.实现摘要失败降级。
9.扩展配置、诊断和框架查询。

本批不接入ReAct ReasonNode和Supervisor ReasonNode，不修改前端，不实现MemoryStore，不保存跨会话用户偏好。

# 一、强制执行规则

1.本提示词指定的类型名、字段语义、枚举值和诊断码属于强制交付项。
2.不得用自创的近似类型替代指定类型。
3.已有同名类型时最小扩展；不存在时按要求新增。
4.不得通过修改`AgentDefinition`承载上下文配置。
5.不得修改Controller请求DTO。
6.不得修改前端。
7.不得创建摘要专用Controller或隐藏调试接口。
8.不得在未读完提示词前开始修改代码。
9.不得只实现LLM调用而省略选择、替换、降级和最终预算验证。
10.关键前置类型缺失时停止并报告，不得在本批临时另造替代类型。

# 二、执行前检查

必须检查：

1.根目录`AGENTS.md`。
2.第七阶段前两批全部实际代码。
3.`AgentMessage`及全部子类型。
4.`SystemAgentMessage`、`UserAgentMessage`、`AssistantAgentMessage`、`ToolAgentMessage`。
5.`ContextMessageGroupType`实际枚举值。
6.`ContextMessageGrouper`分组规则。
7.`ContextMessageHistoryValidator`校验规则。
8.`MessageCountContextTrimmer`。
9.`TokenCountContextTrimmer`。
10.`ContextProcessingPipeline`真实构造器和执行顺序。
11.`ContextProcessingRequest`和`ContextProcessingResult`真实字段。
12.`ContextTokenBudget`。
13.`TokenCounter`。
14.`ModelRequest`和`ModelResponse`。
15.`ModelInvocationGateway`。
16.模型调用是否支持最大输出Token配置。
17.模型响应中的content和ToolCall结构。
18.现有敏感值脱敏能力。
19.现有上下文配置属性。
20.现有`/api/framework/context`或等价查询能力。
21.现有错误码和诊断码定义。
22.Spring Bean实际装配关系。

以下前置类型必须真实存在：

- `ContextMessageGroup`
- `ContextMessageGroupType`
- `ContextMessageGrouper`
- `ContextMessageHistoryValidator`
- `TokenCounter`
- `ContextTokenBudget`
- `ContextProcessingRequest`
- `ContextProcessingResult`
- `ContextProcessingPipeline`

任一关键类型不存在，或被不等价近似类型替代时，停止执行并列出准确阻塞项，不得继续在错误基础上堆叠代码。

# 三、固定摘要语义

本批采用以下语义：

1.摘要只压缩较旧的非System消息。
2.原始System消息永不进入摘要源。
3.最新若干原子消息组保留原文。
4.最后一条真实User消息必须保留原文。
5.最后User消息之后的消息必须保留原文。
6.ToolCall及对应ToolResult不能拆分。
7.工具交互组只能整体进入摘要源或整体保留。
8.已有旧摘要参与下一次摘要，并被新摘要替换。
9.最终上下文最多只允许存在一条框架摘要消息。
10.摘要由LLM生成，不允许本地字符串拼接冒充LLM摘要。
11.摘要完成后继续执行消息数裁剪和Token裁剪。
12.最终上下文仍必须满足Token硬预算。
13.摘要失败时默认降级为不摘要，继续执行普通裁剪。
14.摘要失败不得直接伪造摘要文本。
15.摘要调用不允许使用工具。
16.摘要模型不得获得userId、sessionId、roles、permissions或RunContext。

统一流水线顺序调整为：

```text
消息历史校验
→摘要触发判断
→旧消息选择
→LLM摘要
→用摘要替换旧消息
→按消息数裁剪
→按Token数裁剪
→最终配对与Token验证
```

# 四、SummaryAgentMessage

在`agent-core`现有消息包中必须新增：

`SummaryAgentMessage`

它必须属于现有`AgentMessage`层次。

字段和公共消息属性按当前消息模型风格实现，至少表达：

- content
- generatedAt

要求：

1.`content`不能为空。
2.`content`必须trim。
3.`generatedAt`不能为空。
4.使用不可变类型。
5.不得包含原始消息列表。
6.不得包含完整Prompt。
7.不得包含模型响应对象。
8.不得包含RunContext。
9.不得包含用户身份和权限。
10.不得包含Spring AI类型。
11.不得创建与`SummaryAgentMessage`语义重复的其他消息类型。
12.若`AgentMessage`为sealed类型，必须正确更新允许的子类型。
13.后续映射到LLM时视为System角色，但本批不修改Spring AI Mapper。
14.不得直接使用普通`SystemAgentMessage`替代，因为需要区分原始System消息与框架摘要。

# 五、分组和历史校验调整

必须修改现有：

`ContextMessageGroupType`

新增且名称固定：

`SUMMARY`

必须修改：

`ContextMessageGrouper`

规则：

1.`SummaryAgentMessage`形成单独的SUMMARY原子组。
2.SUMMARY组包含且只包含一条摘要消息。
3.SUMMARY组不得并入TOOL_INTERACTION。
4.SUMMARY组不得被识别为普通Assistant消息。
5.原始System消息仍使用SYSTEM组。
6.工具配对规则保持不变。

必须修改：

`ContextMessageHistoryValidator`

新增规则：

1.上下文中最多存在一条`SummaryAgentMessage`。
2.摘要content不能为空。
3.摘要不得位于未完成的工具交互内部。
4.摘要不得作为ToolResult。
5.发现多条摘要时抛`INVALID_MESSAGE_HISTORY`或已有等价错误。
6.不得静默选择其中一条并删除其他摘要。

必须修改：

`MessageCountContextTrimmer`

规则：

1.SUMMARY组视为框架保留组。
2.SUMMARY消息不计入`maxMessages`。
3.SUMMARY组与SYSTEM组都必须保留。
4.最多只能保留一条摘要。
5.摘要仍计入最终消息总数。
6.原有工具配对和最新用户保护不能被破坏。

必须修改：

`TokenCountContextTrimmer`

规则：

1.SUMMARY组计入Token预算。
2.SUMMARY组默认属于强制保留上下文。
3.原始SYSTEM组优先级高于SUMMARY组。
4.摘要加System和最新用户消息超过预算时明确失败。
5.不得截断摘要文本。
6.不得静默删除原始System消息。

# 六、摘要请求模型

在`agent-core`必须新增不可变：

`ContextSummaryRequest`

字段至少包含：

- sourceMessages
- existingSummary
- maxSummaryTokens

要求：

1.`sourceMessages`类型为不可变`List<AgentMessage>`。
2.源消息不能为空。
3.源消息不得包含原始System消息。
4.源消息允许包含最多一条旧`SummaryAgentMessage`。
5.`existingSummary`使用`Optional<SummaryAgentMessage>`或项目统一可选语义。
6.不得同时在sourceMessages和existingSummary重复保存同一个摘要对象。
7.`maxSummaryTokens`必须大于0。
8.不得包含ModelRequest。
9.不得包含模型名称、API Key和供应商客户端。
10.不得包含UserSession或RunContext。
11.不得使用Map替代固定字段。

# 七、摘要结果模型

在`agent-core`必须新增不可变：

`ContextSummaryResult`

字段至少包含：

- summaryMessage
- sourceMessageCount
- sourceTokenCount
- summaryTokenCount
- existingSummaryReplaced

要求：

1.`summaryMessage`必须为`SummaryAgentMessage`。
2.sourceMessageCount必须大于0。
3.sourceTokenCount必须大于0。
4.summaryTokenCount必须大于0。
5.summaryTokenCount不得超过请求的maxSummaryTokens。
6.existingSummaryReplaced准确反映是否合并过旧摘要。
7.不得返回模型原始响应。
8.不得返回摘要Prompt。
9.不得返回完整源消息副本。
10.不得返回ToolCall。
11.不得包含Spring AI类型。

# 八、ContextSummarizer接口

在`agent-core`必须新增：

```java
public interface ContextSummarizer {
    ContextSummaryResult summarize(ContextSummaryRequest request);
}
```

要求：

1.接口位于稳定模块。
2.不得依赖Spring。
3.不得依赖Spring AI类型。
4.不得依赖LangGraph4j。
5.不得返回null。
6.不得保存跨请求状态。
7.不得把摘要策略和消息选择混入该接口。
8.接口只负责把已选择的源消息压缩成摘要。

# 九、摘要触发器

在`agent-runtime`必须新增纯Java：

`ContextSummaryTrigger`

建议方法：

```java
ContextSummaryTriggerDecision evaluate(
    int currentTokenCount,
    int effectiveMessageBudget,
    double triggerRatio
);
```

必须新增不可变：

`ContextSummaryTriggerDecision`

至少表达：

- triggered
- currentTokenCount
- effectiveMessageBudget
- triggerTokenCount
- utilization

触发规则：

1.`triggerTokenCount=ceil(effectiveMessageBudget*triggerRatio)`。
2.currentTokenCount大于或等于triggerTokenCount时触发。
3.currentTokenCount低于阈值时不触发。
4.triggerRatio必须大于0且小于1。
5.effectiveMessageBudget必须大于0。
6.utilization计算不得产生NaN或Infinity。
7.使用long或BigDecimal避免中间溢出。
8.不得根据消息数量触发摘要。
9.不得调用模型。
10.保持无状态和线程安全。

# 十、摘要源选择器

在`agent-runtime`必须新增纯Java：

`ContextSummarySelector`

必须新增不可变：

`ContextSummarySelection`

至少包含：

- sourceMessages
- retainedMessages
- existingSummary
- sourceMessageCount
- sourceTokenCount
- sourceGroupCount
- firstSourceIndex
- insertionIndex

选择器依赖：

- `ContextMessageGrouper`
- `TokenCounter`

建议方法：

```java
ContextSummarySelection select(
    List<AgentMessage> messages,
    int recentGroupsToPreserve,
    int minSourceTokens
);
```

选择规则：

1.先完成消息分组。
2.全部原始SYSTEM组必须保留原文。
3.最后`recentGroupsToPreserve`个非SYSTEM、非SUMMARY原子组必须保留原文。
4.最后一条真实User消息所在组必须保留原文。
5.最后User消息之后的全部组必须保留原文。
6.未完成的当前轮工具组不得进入摘要源。
7.旧SUMMARY组可作为旧摘要参与合并。
8.除旧SUMMARY外，摘要源只能包含非SYSTEM旧原子组。
9.工具交互组必须整体进入源或整体保留。
10.不得拆分TOOL_INTERACTION。
11.摘要源必须是较旧消息构成的确定性集合。
12.摘要源消息顺序保持原始顺序。
13.retainedMessages保持原始顺序。
14.不得修改输入列表。
15.不得根据content文本判断消息角色。
16.不得根据工具名称进行配对。
17.相同输入必须得到相同选择结果。
18.不得使用HashSet无序迭代决定结果。

跳过摘要的情况：

1.没有可摘要旧消息。
2.可摘要源Token少于`minSourceTokens`。
3.所有消息都属于必须保留的最近上下文。
4.只存在System消息。
5.只存在一条旧摘要且没有新增旧消息可合并。

选择器不得调用模型。

# 十一、摘要替换规则

摘要成功后，将源消息替换为一条新的`SummaryAgentMessage`。

必须实现纯Java：

`ContextSummaryMerger`

建议方法：

```java
List<AgentMessage> merge(
    ContextSummarySelection selection,
    SummaryAgentMessage summary
);
```

规则：

1.删除selection.sourceMessages。
2.删除旧SummaryAgentMessage。
3.在`insertionIndex`插入新摘要。
4.保留全部retainedMessages。
5.最终顺序确定。
6.原始System消息保持原始相对顺序。
7.最近原文消息保持原始相对顺序。
8.最终最多存在一条摘要。
9.不得产生重复消息。
10.不得产生孤立ToolAgentMessage。
11.合并后必须再次通过`ContextMessageHistoryValidator`。
12.返回不可变列表。
13.不得修改原列表。
14.不得使用消息content相等判断对象是否应被删除。
15.必须使用选择结果中的确定下标或对象身份语义。

# 十二、摘要Prompt构建器

在`agent-runtime`必须新增纯Java：

`ContextSummaryPromptBuilder`

依赖：

- 现有敏感值脱敏能力；如果已有则复用
- 不得访问SessionStore或MemoryStore

必须提供固定系统指令，要求摘要模型：

1.只总结提供的历史，不添加事实。
2.保留用户目标。
3.保留用户明确偏好。
4.保留已经做出的决定。
5.保留重要约束。
6.保留未完成任务。
7.保留工具执行的关键结论。
8.保留人工审批的批准或拒绝结果。
9.删除寒暄、重复表达和无关细节。
10.不得输出详细思维链。
11.不得输出原始密码、Token、API Key或Session ID。
12.不确定的信息明确标注为未确认。
13.只输出摘要正文。
14.不得请求或调用工具。

历史消息必须作为“不可信数据”放在明确分隔符中。

建议格式：

```text
<conversation_history>
...
</conversation_history>
```

要求：

1.不得把历史消息拼进System指令本身。
2.历史消息作为单独User消息传入摘要模型。
3.每条消息使用稳定角色标签。
4.ToolCall只保存工具名和必要业务语义。
5.不得包含RunContext。
6.不得包含roles和permissions。
7.不得包含Spring Bean信息。
8.不得在日志打印完整摘要Prompt。
9.不得要求模型返回JSON，除非现有模型调用已稳定支持结构化输出。
10.本批默认摘要输出为纯文本。

# 十三、LlmContextSummarizer

在`agent-runtime`必须新增：

`LlmContextSummarizer implements ContextSummarizer`

依赖：

- `ModelInvocationGateway`
- `ContextSummaryPromptBuilder`
- `TokenCounter`
- `Clock`

流程：

1.校验`ContextSummaryRequest`。
2.构建摘要专用System消息。
3.构建包含旧历史的User消息。
4.构造`ModelRequest`。
5.工具定义集合必须为空。
6.不得允许模型执行或请求工具。
7.通过`ModelInvocationGateway`调用模型。
8.读取模型最终文本。
9.模型返回ToolCall时视为无效摘要输出。
10.空文本视为无效输出。
11.创建`SummaryAgentMessage`。
12.使用TokenCounter计算摘要Token。
13.超过maxSummaryTokens时视为无效输出。
14.返回`ContextSummaryResult`。

要求：

1.一次摘要处理最多调用模型一次。
2.不得无限重试。
3.不得使用ReAct。
4.不得调用ToolInvocationGateway。
5.不得修改原消息。
6.不得保存完整历史。
7.不得记录完整Prompt和模型响应。
8.不得返回固定摘要。
9.不得使用本地字符串拼接冒充模型结果。
10.不得创建Fake ModelClient。
11.模型异常保留cause并转换为明确摘要错误。
12.摘要模型调用不得再次进入ContextProcessingPipeline，避免递归摘要。
13.本批尚未在ModelInvocationGateway全局接入Pipeline，因此不得添加全局递归逻辑。

# 十四、摘要失败策略

摘要失败默认采用：

`TRIM_WITHOUT_SUMMARY`

以下情况视为摘要失败：

1.模型调用异常。
2.模型返回空内容。
3.模型返回ToolCall。
4.摘要Token超过maxSummaryTokens。
5.摘要消息构造失败。
6.摘要结果不符合校验规则。

失败后：

1.不得生成假摘要。
2.不得将异常文本作为摘要。
3.不得中断普通上下文裁剪。
4.使用原始消息继续执行MessageCount裁剪。
5.再执行TokenCount裁剪。
6.记录诊断`SUMMARY_FAILED_FALLBACK_TO_TRIMMING`。
7.最终仍必须满足Token预算。
8.强制上下文仍超预算时继续抛`CONTEXT_BUDGET_EXCEEDED`。
9.服务端可记录安全WARN。
10.不得在日志记录完整旧消息和摘要Prompt。

本批不增加复杂可配置失败策略枚举。

# 十五、修改ContextProcessingRequest

必须最小扩展现有：

`ContextProcessingRequest`

增加摘要处理所需字段或子配置，至少表达：

- summaryEnabled
- summaryTriggerRatio
- summaryMinSourceTokens
- summaryMaxTokens
- summaryRecentGroupsToPreserve

要求：

1.不得创建`ContextProcessingRequestV2`。
2.summaryTriggerRatio必须大于0且小于1。
3.summaryMinSourceTokens必须大于0。
4.summaryMaxTokens必须大于0。
5.summaryRecentGroupsToPreserve必须大于等于1。
6.摘要关闭时仍允许普通消息和Token裁剪。
7.不得把模型客户端放进请求。
8.不得允许外部HTTP客户端控制这些字段。
9.这些值由框架配置构造。
10.不得修改Agent API请求DTO。

如已有独立`ContextSummaryOptions`更清晰，可以新增该不可变值对象并作为字段，但名称必须固定为：

`ContextSummaryOptions`

不得使用自由Map。

# 十六、修改ContextProcessingResult

必须最小扩展现有：

`ContextProcessingResult`

增加：

- summaryApplied
- summaryTriggered
- summarizedMessageCount
- summarySourceTokenCount
- summaryTokenCount
- existingSummaryReplaced

要求：

1.不得创建`ContextProcessingResultV2`。
2.未触发摘要时所有摘要统计为0或false。
3.触发但失败降级时：
- summaryTriggered=true
- summaryApplied=false
  4.成功摘要时summaryApplied=true。
  5.summarizedMessageCount准确统计被摘要替换的消息。
  6.summaryTokenCount来自TokenCounter。
  7.不得返回摘要Prompt。
  8.不得返回完整源消息。
  9.processedMessages必须包含最终摘要消息。
  10.最终Token统计基于完整processedMessages重新计算。

# 十七、修改ContextProcessingPipeline

必须修改现有：

`ContextProcessingPipeline`

新增依赖：

- `ContextSummaryTrigger`
- `ContextSummarySelector`
- `ContextSummaryMerger`
- 可选`ContextSummarizer`

不得创建第二套Pipeline。

执行流程：

1.验证原始消息历史。
2.计算原始Token数。
3.计算有效消息预算。
4.摘要关闭时跳过摘要。
5.摘要开启时执行触发判断。
6.未达到阈值时跳过摘要。
7.达到阈值时调用ContextSummarySelector。
8.无足够摘要源时跳过摘要。
9.存在源且ContextSummarizer可用时调用LLM摘要。
10.摘要成功后通过ContextSummaryMerger替换旧消息。
11.摘要失败时使用原始消息继续。
12.对摘要处理后的消息执行MessageCount裁剪。
13.对上一步结果执行TokenCount裁剪。
14.再次验证工具消息配对。
15.再次验证最多一条SummaryAgentMessage。
16.重新计算最终Token数。
17.最终超过预算时明确失败。
18.合并全部诊断。
19.返回准确ContextProcessingResult。

要求：

1.摘要必须发生在消息数和Token数裁剪之前。
2.不得先删除旧消息后再尝试摘要。
3.摘要失败不得导致旧消息被部分删除后丢失。
4.调用摘要前保留原始不可变消息快照。
5.每次process最多调用摘要模型一次。
6.不得跨请求保存摘要结果。
7.不得访问ReactAgentState。
8.不得访问CheckpointStore和MemoryStore。
9.不得使用ThreadLocal。
10.同一Pipeline支持并发调用。

# 十八、旧摘要合并

当输入已经包含一条`SummaryAgentMessage`时：

1.旧摘要不能直接永久累积。
2.旧摘要作为历史压缩结果参与新摘要。
3.新摘要必须同时覆盖：
- 旧摘要中的重要内容
- 本次新增进入摘要源的旧消息
  4.摘要成功后旧摘要被删除。
  5.最终只保留新摘要。
  6.`existingSummaryReplaced=true`。
  7.摘要失败时保留旧摘要和原消息，不得删除旧摘要。
  8.不得出现摘要套摘要无限增长。
  9.不得将旧摘要当作原始System消息永久重复保留。
  10.旧摘要仍需计入摘要源Token数。

# 十九、诊断码

必须在现有上下文诊断枚举中新增，名称固定：

- `SUMMARY_TRIGGERED`
- `SUMMARY_APPLIED`
- `SUMMARY_SKIPPED_DISABLED`
- `SUMMARY_SKIPPED_BELOW_THRESHOLD`
- `SUMMARY_SKIPPED_NO_SOURCE`
- `SUMMARY_SKIPPED_SOURCE_TOO_SMALL`
- `SUMMARY_UNAVAILABLE`
- `SUMMARY_FAILED_FALLBACK_TO_TRIMMING`
- `EXISTING_SUMMARY_REPLACED`
- `FINAL_TOKEN_BUDGET_VERIFIED`

要求：

1.不得使用自由字符串替代。
2.不得创建第二套摘要诊断Map。
3.同一诊断只出现一次。
4.诊断顺序必须稳定。
5.诊断不得携带消息正文。
6.诊断不得携带异常堆栈。
7.诊断不得携带用户身份和工具参数。

# 二十、错误码

检查现有`AgentErrorCode`。

缺失时最小新增，名称固定：

- `CONTEXT_SUMMARY_FAILED`
- `INVALID_CONTEXT_SUMMARY_OUTPUT`

语义：

1.模型调用失败：
`CONTEXT_SUMMARY_FAILED`
2.空输出、ToolCall输出或超长输出：
`INVALID_CONTEXT_SUMMARY_OUTPUT`
3.Pipeline默认捕获这些摘要错误并降级。
4.直接调用ContextSummarizer时仍保留结构化错误。
5.不得把摘要失败转换为`MODEL_INVOCATION_FAILED`返回给最终用户。
6.不得把工具配对错误转换为摘要错误。
7.不得创建多个同义摘要错误码。

# 二十一、配置扩展

必须扩展现有`agent.context`配置：

```yaml
agent:
  context:
    summary:
      enabled: true
      trigger-ratio: 0.80
      min-source-tokens: 512
      max-summary-tokens: 512
      recent-groups-to-preserve: 4
```

要求：

1.使用类型安全配置属性。
2.enabled默认值明确。
3.trigger-ratio必须大于0且小于1。
4.min-source-tokens必须大于0。
5.max-summary-tokens必须大于0。
6.recent-groups-to-preserve必须大于等于1。
7.max-summary-tokens必须小于有效消息预算。
8.默认值只集中定义一次。
9.不得在Pipeline中硬编码另一套默认值。
10.不得配置模型API Key。
11.不得通过模型名称猜测摘要阈值。
12.配置非法时启动失败并给出明确原因。

# 二十二、Spring Bean装配

在`agent-infrastructure`现有上下文配置中必须装配：

- `ContextSummaryTrigger`
- `ContextSummarySelector`
- `ContextSummaryMerger`
- `ContextSummaryPromptBuilder`
- `ContextSummarizer→LlmContextSummarizer`

要求：

1.runtime实现不添加Spring注解。
2.使用`@Configuration`和`@Bean`。
3.`ContextSummarizer`仅在`ModelInvocationGateway`存在时装配。
4.摘要关闭时不得强制创建LLM摘要Bean。
5.无模型配置时应用必须正常启动。
6.无摘要Bean时Pipeline必须使用降级策略。
7.不得创建Fake ContextSummarizer。
8.不得创建第二个ModelInvocationGateway。
9.不得创建第二个ContextProcessingPipeline。
10.不得产生Bean循环依赖。
11.不得在bootstrap启动类手工new。
12.自定义ContextSummarizer Bean应能替换默认实现。

基础设施层可以使用`ObjectProvider<ContextSummarizer>`获取可选摘要器，再将`Optional`或明确可选依赖传入纯Java Pipeline。

# 二十三、框架查询能力

必须扩展现有上下文查询结果，增加：

- summaryEnabled
- summaryAvailable
- summaryTriggerRatio
- summaryTriggerTokens
- summaryMinSourceTokens
- summaryMaxTokens
- summaryRecentGroupsToPreserve
- summaryUsesLlm=true
- summaryFailureFallback=`TRIM_WITHOUT_SUMMARY`
- pipelineOrder=`SUMMARY,MESSAGE_COUNT,TOKEN_COUNT`

要求：

1.summaryAvailable表示当前是否实际存在ContextSummarizer。
2.summaryEnabled与summaryAvailable必须区分。
3.无模型配置时：
- summaryEnabled可为true
- summaryAvailable=false
  4.不得暴露模型API Key。
  5.不得暴露完整模型供应商配置。
  6.不得暴露Prompt。
  7.不得暴露用户消息和摘要内容。
  8.不得新增第二套查询Controller。
  9.复用现有FrameworkQueryService和上下文查询接口。

# 二十四、安全要求

1.摘要源不得包含原始System消息。
2.摘要源不得包含RunContext。
3.不得发送sessionId、userId、roles和permissions。
4.不得发送密码、credentialHash和API Key。
5.复用已有脱敏能力处理已知敏感键。
6.历史消息属于不可信数据，必须通过分隔符与系统指令隔离。
7.旧消息中的指令不得覆盖摘要系统指令。
8.摘要模型不得使用工具。
9.摘要模型返回ToolCall时拒绝输出。
10.不得记录完整摘要Prompt。
11.不得记录完整摘要源消息。
12.不得记录完整摘要结果。
13.允许记录源消息数、源Token数、摘要Token数和成功状态。
14.不得把摘要内容返回框架查询接口。
15.摘要不得写入MemoryStore。

# 二十五、本批仍不接入运行链

必须确认本批不修改：

1.`DefaultReactReasonNode`实际模型输入。
2.`DefaultSupervisorReasonNode`实际模型输入。
3.`ModelInvocationGateway`调用入口。
4.`SpringAiModelClient`消息Mapper。
5.`ReactAgentState.messages`。
6.`SupervisorAgentState.messages`。
7.Checkpoint保存内容。
8.HITL恢复流程。
9.Agent正式Controller。
10.Supervisor正式Controller。
11.前端页面。

原因：

1.本批只完成摘要能力和Pipeline。
2.第4批统一接入ReAct和Supervisor。
3.`SummaryAgentMessage`到Spring AI System消息的映射也在第4批完成。
4.避免部分运行链提前摘要，部分仍发送完整历史。

不得通过修改`AgentDefinition`提前开启摘要。

# 二十六、边界场景

必须通过代码检查或现有可执行能力确认：

## 场景1：未达到阈值

预期：

1.不调用摘要模型。
2.summaryTriggered=false。
3.summaryApplied=false。
4.诊断包含`SUMMARY_SKIPPED_BELOW_THRESHOLD`。
5.继续执行普通裁剪。

## 场景2：摘要关闭

预期：

1.不调用摘要模型。
2.诊断包含`SUMMARY_SKIPPED_DISABLED`。
3.普通裁剪正常。

## 场景3：达到阈值且有旧消息

预期：

1.选择旧原子组。
2.保留System和最近原文。
3.调用模型一次。
4.生成SummaryAgentMessage。
5.摘要后执行消息和Token裁剪。
6.最终Token不超过预算。

## 场景4：没有可摘要旧消息

预期：

1.不调用模型。
2.诊断包含`SUMMARY_SKIPPED_NO_SOURCE`。
3.不产生空摘要。

## 场景5：源Token太少

预期：

1.不调用模型。
2.诊断包含`SUMMARY_SKIPPED_SOURCE_TOO_SMALL`。
3.继续普通裁剪。

## 场景6：已有旧摘要

预期：

1.旧摘要参与新摘要。
2.新摘要替换旧摘要。
3.最终只有一条SummaryAgentMessage。
4.existingSummaryReplaced=true。
5.诊断包含`EXISTING_SUMMARY_REPLACED`。

## 场景7：模型返回空文本

预期：

1.摘要失败。
2.不插入摘要。
3.保留原消息进入普通裁剪。
4.诊断包含`SUMMARY_FAILED_FALLBACK_TO_TRIMMING`。

## 场景8：模型返回ToolCall

预期：

1.视为非法摘要输出。
2.不执行工具。
3.不插入摘要。
4.降级普通裁剪。

## 场景9：摘要过长

预期：

1.TokenCounter确认超过maxSummaryTokens。
2.拒绝摘要。
3.不截断摘要文本。
4.降级普通裁剪。

## 场景10：摘要后仍超预算

预期：

1.继续执行TokenCountContextTrimmer。
2.最终满足预算。
3.强制上下文仍过大时抛`CONTEXT_BUDGET_EXCEEDED`。
4.不得返回超预算成功结果。

## 场景11：工具原子组

预期：

1.工具组整体进入摘要源或整体保留。
2.不得只摘要ToolResult。
3.不得产生孤立工具消息。

## 场景12：无模型配置

预期：

1.应用正常启动。
2.summaryAvailable=false。
3.Pipeline降级普通裁剪。
4.不得因缺少ModelInvocationGateway导致Bean启动失败。

不得为验证新增隐藏Controller或测试脚本。

# 二十七、性能与并发

1.每次Pipeline调用最多执行一次摘要模型调用。
2.ContextSummarySelector保持O(n)。
3.Token局部缓存只允许存在于当前调用。
4.不得跨请求缓存AgentMessage。
5.不得使用static可变状态。
6.不得使用ThreadLocal。
7.不得为所有摘要请求增加全局锁。
8.同一Pipeline和Summarizer Bean支持并发调用。
9.不得无限重试摘要模型。
10.摘要失败后只降级一次，不得再次调用摘要模型。

# 二十八、本批禁止实现

禁止：

1.把Pipeline接入ReAct ReasonNode。
2.把Pipeline接入Supervisor ReasonNode。
3.修改Spring AI消息Mapper。
4.修改AgentDefinition。
5.修改Agent API请求字段。
6.修改Controller DTO。
7.前端上下文页面。
8.跨threadId消息存储。
9.MemoryStore。
10.长期用户偏好。
11.数据库或Redis。
12.摘要历史持久化。
13.摘要模型工具调用。
14.摘要自动重试循环。
15.供应商专用Tokenizer网络调用。
16.摘要调试Controller。
17.SSE。
18.测试脚本、README和Git操作。

# 二十九、编译验收

执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

必须逐项确认：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.`SummaryAgentMessage`真实存在。
4.`ContextSummaryRequest`真实存在。
5.`ContextSummaryResult`真实存在。
6.`ContextSummarizer`真实存在。
7.`ContextSummaryTrigger`真实存在。
8.`ContextSummarySelector`真实存在。
9.`ContextSummaryMerger`真实存在。
10.`ContextSummaryPromptBuilder`真实存在。
11.`LlmContextSummarizer`真实存在。
12.`ContextMessageGroupType`包含SUMMARY。
13.历史校验最多允许一条摘要。
14.消息数裁剪保留摘要。
15.Token裁剪计算摘要Token。
16.旧摘要会被新摘要替换。
17.Pipeline顺序为摘要→消息数→Token数。
18.摘要失败会降级普通裁剪。
19.摘要模型调用不携带工具。
20.无模型配置时应用可启动。
21.配置和框架查询字段完整。
22.没有修改AgentDefinition。
23.没有修改ReAct和Supervisor模型输入。
24.没有前端和记忆功能。
25.git diff没有无关修改。

# 三十、最终输出

最终输出必须逐项对应实施清单，只输出：

1.新增和修改文件清单。
2.`SummaryAgentMessage`结构。
3.摘要请求和结果模型。
4.`ContextSummarizer`接口。
5.摘要触发计算规则。
6.摘要源选择规则。
7.旧摘要替换方式。
8.摘要Prompt安全约束。
9.`LlmContextSummarizer`调用流程。
10.摘要失败降级流程。
11.`ContextProcessingPipeline`最终顺序。
12.新增诊断码和错误码。
13.配置和Spring Bean装配。
14.无模型配置时的行为。
15.框架查询结果。
16.边界场景实际检查结果。
17.编译和打包结果。
18.未完成验证及准确原因。
19.发现但未处理的问题。

不得将计划描述为已完成，不得仅输出“编译通过”，不得遗漏强制交付类型。

编译或打包失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE7_BATCH3_END===