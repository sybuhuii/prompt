你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须先完整阅读并遵守根目录`AGENTS.md`。其中已有模块边界、第三方模型隔离、不可变性、安全、编译和输出规范，本提示词不重复展开。

已完成：

1.通用Agent领域模型与Registry。
2.Spring AI模型适配。
3.ToolInvocationGateway及工具治理链。
4.单Agent ReAct执行引擎。
5.Supervisor多Agent编排。
6.身份、Session、RBAC和工具ACL。
7.HITL中断、Checkpoint、审批和恢复。
8.Web动态页面。

当前只执行第七阶段第1批：

1.建立上下文管理稳定模型和SPI。
2.分析AgentMessage中的工具调用配对关系。
3.实现System消息永久保留规则。
4.实现按消息数量裁剪。
5.定义TokenCounter并提供默认估算实现。
6.提供上下文裁剪诊断结果。
7.完成Spring Bean装配。

本批不把裁剪接入ReasonNode或ModelInvocationGateway，不实现按Token裁剪，不调用LLM摘要，不实现跨轮会话存储，不修改前端。

# 一、执行前检查

先检查：

1.根目录`AGENTS.md`。
2.AgentMessage及全部子类型。
3.SystemAgentMessage。
4.UserAgentMessage。
5.AssistantAgentMessage。
6.ToolAgentMessage。
7.ToolCall及ToolCall ID字段。
8.ModelRequest和ModelResponse。
9.DefaultReactReasonNode。
10.DefaultSupervisorReasonNode。
11.ReactAgentState和SupervisorAgentState。
12.AgentErrorCode和AgentFrameworkException。
13.现有Mapper、Registry和配置方式。
14.当前消息集合是否保证不可变。
15.Assistant消息如何保存ToolCall。
16.Tool消息如何关联原ToolCall。
17.消息是否存在role、content、metadata等字段。
18.项目中是否已有TokenCounter、ContextManager、MessageTrimmer等抽象。
19.Spring AI当前版本是否提供可复用Token计数接口。
20.现有工具调用消息在一次多ToolCall场景中的排列方式。

要求：

1.以实际代码为准增量实现。
2.已有上下文抽象能够复用时必须复用。
3.不得创建第二套AgentMessage。
4.不得让Spring AI Message进入core稳定接口。
5.前序代码存在阻断问题时只做最小修复。
6.不得修改ReAct和Supervisor图结构。
7.不得提前实现第2批及后续功能。

# 二、上下文管理边界

本阶段上下文管理负责：

1.从已有消息历史中选择本次模型调用需要携带的消息。
2.保留System消息。
3.保留最近消息。
4.维护ToolCall和ToolResult完整性。
5.控制消息数和Token数。
6.后续在必要时摘要旧消息。
7.输出处理后的消息及诊断指标。

本阶段上下文管理不负责：

1.用户登录和权限。
2.保存长期用户偏好。
3.跨会话MemoryStore。
4.Checkpoint中断恢复。
5.执行模型。
6.执行工具。
7.修改Agent定义。
8.生成runId或threadId。
9.保存聊天记录到数据库。
10.决定Agent路由。

上下文管理是模型调用前的透明中间层。

# 三、稳定领域模型

在agent-core合适的context包中新增或复用不可变模型。

## 3.1 ContextTrimRequest

至少包含：

- messages
- maxMessages

要求：

1.messages使用`List<AgentMessage>`或现有稳定父类型。
2.构造时复制为不可变列表。
3.maxMessages必须大于0。
4.maxMessages表示希望保留的非System消息数量。
5.System消息不计入maxMessages。
6.不得包含Spring AI Message。
7.不得包含ChatClient或ChatModel。
8.不得包含HTTP对象。
9.本批不加入maxTokens和summaryThreshold，留到后续扩展。
10.不要为了未来字段使用无类型Map。

## 3.2 ContextTrimResult

至少包含：

- originalMessages
- retainedMessages
- originalMessageCount
- retainedMessageCount
- removedMessageCount
- retainedSystemMessageCount
- retainedNonSystemMessageCount
- targetMaxMessages
- atomicGroupOvershoot
- diagnostics

要求：

1.originalMessages是否需要返回应根据安全和内存占用判断；若无必要可只保留统计值。
2.retainedMessages必须保持原始顺序。
3.retainedMessages为不可变列表。
4.removedMessageCount不得小于0。
5.atomicGroupOvershoot表示为维护工具消息完整性而超过目标的数量。
6.diagnostics只保存安全、稳定的诊断信息。
7.不得保存完整Prompt副本。
8.不得保存RunContext、Session或权限。
9.不得依赖Spring。
10.不得使用null表达空集合。

如已有统一处理结果模型，最小扩展，不复制。

# 四、ContextTrimmer接口

在agent-core定义或复用：

```java
public interface ContextTrimmer {
    ContextTrimResult trim(ContextTrimRequest request);
}
```

要求：

1.接口保持框架无关。
2.不依赖Spring AI。
3.不依赖LangGraph4j。
4.不依赖具体模型供应商。
5.不执行模型。
6.不保存状态。
7.不修改输入消息集合。
8.不得返回null。
9.允许后续增加Token和摘要实现。
10.不要在接口中加入大量当前未使用方法。

# 五、消息原子组

工具调用消息不能被裁剪成不完整历史。

在agent-runtime实现内部不可变模型：

`ContextMessageGroup`

至少包含：

- messages
- startIndex
- endIndex
- groupType
- atomic

`ContextMessageGroupType`至少表达：

- SYSTEM
- NORMAL
- TOOL_INTERACTION

要求：

1.SYSTEM组包含单条System消息。
2.NORMAL组通常包含单条普通用户或助手消息。
3.TOOL_INTERACTION组包含：
- 发出ToolCall的Assistant消息
- 与其ToolCall对应的全部ToolAgentMessage
  4.一个Assistant消息包含多个ToolCall时，它与全部对应Tool结果构成一个原子组。
  5.原子组只能整体保留或整体移除。
  6.组内消息保持原始顺序。
  7.不得修改原AgentMessage。
  8.不得复制ToolCall ID。
  9.不得依赖Spring。
  10.该模型仅供上下文运行时使用，不暴露给HTTP接口。

# 六、工具消息配对分析

在agent-runtime实现纯Java：

`ContextMessageGrouper`

建议方法：

```java
List<ContextMessageGroup> group(List<AgentMessage> messages);
```

必须依据项目真实消息字段实现，不得根据content文本猜测ToolCall。

配对规则：

1.遍历原始消息列表。
2.System消息单独形成SYSTEM组。
3.没有ToolCall的User或Assistant消息形成NORMAL组。
4.包含ToolCall的Assistant消息开始一个TOOL_INTERACTION组。
5.读取该Assistant消息中的全部ToolCall ID。
6.收集后续与这些ID对应的ToolAgentMessage。
7.只有全部ToolCall都拥有匹配Tool结果时，该组才算完整。
8.ToolCall ID匹配必须完全匹配。
9.不得使用工具名称代替ToolCall ID配对。
10.多个ToolCall返回顺序可以与声明顺序不同，但原消息顺序必须保留。
11.不得把属于下一次Assistant ToolCall的结果加入当前组。
12.遇到下一条普通User或Assistant消息时，当前工具组必须已经完整。
13.System消息不得被并入工具组。
14.一个ToolAgentMessage只能属于一个组。
15.不得使用static可变状态。
16.保持无状态和线程安全。

# 七、非法消息历史

以下情况属于非法工具消息历史：

1.ToolAgentMessage找不到对应Assistant ToolCall。
2.Assistant声明的ToolCall缺少ToolAgentMessage。
3.相同ToolCall ID出现多个结果。
4.ToolCall ID为空。
5.ToolAgentMessage关联ID为空。
6.同一ToolCall ID在多个Assistant消息中重复。
7.工具交互尚未结束就出现下一条普通对话消息。
8.消息集合中存在null元素。

实现或复用：

`ContextMessageHistoryValidator`

要求：

1.在分组前或分组过程中验证。
2.非法时使用明确结构化错误。
3.不得静默删除孤立Tool消息。
4.不得自动伪造ToolResult补齐。
5.不得根据工具名称模糊配对。
6.错误信息可包含消息下标和安全ToolCall短标识。
7.不得包含完整工具参数或结果。
8.不得记录完整消息历史。
9.不得把非法历史继续发送给模型。
10.保持纯Java。

如AgentErrorCode缺失，最小增加或复用：

- INVALID_MESSAGE_HISTORY
- TOOL_MESSAGE_PAIRING_FAILED
- INVALID_CONTEXT_CONFIGURATION

不得增加多个语义重复错误码。

# 八、System消息保留规则

裁剪后必须永久保留全部System消息。

规则：

1.System消息不计入maxMessages。
2.System消息无论出现在历史哪个位置都保留。
3.System消息之间保持原始顺序。
4.System消息与其他保留消息合并后保持全局原始顺序。
5.不得只保留第一条System消息。
6.不得将多条System消息拼接成一条。
7.不得修改System消息content。
8.不得把身份权限等内部信息新增到System消息。
9.不得把System消息移动到错误的相对位置。
10.后续摘要也不得替代System消息。

# 九、按消息数裁剪

在agent-runtime实现：

`MessageCountContextTrimmer implements ContextTrimmer`

依赖：

- ContextMessageGrouper
- ContextMessageHistoryValidator
- TokenCounter（只用于诊断，可选）

算法：

1.校验请求。
2.验证消息历史。
3.将消息划分为原子组。
4.全部SYSTEM组永久保留。
5.从最后一个非SYSTEM组开始向前选择。
6.选择完整原子组，直到达到maxMessages目标。
7.不得拆分TOOL_INTERACTION组。
8.完成选择后按原始下标升序输出。
9.输入列表不得被修改。
10.每次调用独立计算。

maxMessages只统计非System消息。

普通情况：

- 保留最近不超过maxMessages条非System消息。

原子组边界情况：

1.加入某个TOOL_INTERACTION组会超过maxMessages时，不得只保留其中一部分。
2.如果已经选择了更新的消息组，则默认不再选择这个超限旧组。
3.如果最新的非System组本身就超过maxMessages，为保证最新交互完整，仍保留整个最新组。
4.这种情况记录`atomicGroupOvershoot`。
5.除最新原子组无法拆分的情况外，不应超过目标。
6.不得为了严格满足数量而破坏工具消息配对。

示例一：

```text
System
User-1
Assistant-1
User-2
Assistant-2
```

maxMessages=2，结果：

```text
System
User-2
Assistant-2
```

示例二：

```text
System
User
Assistant(toolCall-A)
ToolResult-A
Assistant(final)
```

maxMessages=2，最近组为：

```text
Assistant(final)
```

再加入工具交互组会超过目标，因此结果可以是：

```text
System
Assistant(final)
```

只要不会形成孤立Tool消息。

示例三：

```text
System
Assistant(toolCall-A,toolCall-B)
ToolResult-A
ToolResult-B
```

maxMessages=2，但最新原子组有3条，必须完整保留并记录overshoot=1。

不得为示例硬编码算法。

# 十、最新用户输入保护

模型调用上下文通常必须包含当前用户输入。

实现以下保护：

1.识别最后一条UserAgentMessage。
2.裁剪结果必须包含最后一条用户消息。
3.如果最后一条用户消息位于已选择的最新组中，正常保留。
4.如果历史结构导致最后用户消息未被选择，应调整选择结果。
5.不得为保留旧Assistant消息而删除最新用户消息。
6.如果maxMessages配置不足以保留最新用户消息及其后续完整工具组，允许原子组最小超限。
7.记录相应diagnostic。
8.没有User消息的内部模型请求允许正常处理。
9.不得把Supervisor内部观察消息错误识别为真实用户输入，按实际消息类型判断。
10.不得根据content前缀判断角色。

# 十一、TokenCounter接口

任务要求提供单条及多条消息的Token估算能力。

在agent-core定义或复用：

```java
public interface TokenCounter {
    int count(AgentMessage message);
    int count(Collection<? extends AgentMessage> messages);
}
```

要求：

1.稳定框架SPI。
2.不依赖Spring AI具体实现。
3.单条消息不能为空。
4.消息集合不能包含null。
5.空集合返回0。
6.返回值不得为负数。
7.集合计数必须包含每条消息固定开销。
8.工具名称、ToolCall参数和ToolResult内容必须计入。
9.不得只统计content字段。
10.不得调用LLM完成计数。
11.允许未来替换为模型供应商精确Tokenizer。
12.不得把模型API Key传入计数器。

# 十二、默认Token估算实现

在agent-runtime实现：

`HeuristicTokenCounter implements TokenCounter`

本实现是模型无关估算器，不宣称精确。

至少统计：

1.System/User/Assistant文本content。
2.Assistant中的ToolCall ID、工具名及参数。
3.ToolAgentMessage中的ToolCall ID、工具名、content及安全错误字段。
4.每条消息固定结构开销。
5.消息集合的整体结构开销。

建议算法：

1.基于Unicode code point或UTF-8字节数估算。
2.中文、日文、韩文字符按接近每字符1 token估算。
3.英文、数字连续文本按约每4字符1 token估算。
4.标点、JSON结构和空白采用保守估算。
5.结果向上取整。
6.每条消息增加可配置或固定的小额结构开销。
7.ToolCall参数按实际序列化文本长度估算。
8.不需要引入第三方Tokenizer依赖。

要求：

1.算法必须确定，相同输入得到相同结果。
2.不能返回0给非空消息。
3.不能使用默认平台字符集。
4.不得把完整消息写入日志。
5.保持无状态和线程安全。
6.类注释明确它是估算值。
7.不得声称与某个模型Tokenizer完全一致。
8.后续可以通过自定义TokenCounter Bean替换。

如项目已有可靠模型TokenCounter，优先适配稳定SPI，不重复实现。

# 十三、裁剪诊断

ContextTrimResult的diagnostics至少表达：

- SYSTEM_MESSAGES_PRESERVED
- RECENT_MESSAGES_TRIMMED
- TOOL_GROUP_PRESERVED
- ATOMIC_GROUP_OVERSHOOT
- LATEST_USER_MESSAGE_PRESERVED
- NO_TRIMMING_REQUIRED

可使用enum或稳定字符串代码。

要求：

1.不得存储完整消息内容。
2.不得存储工具参数。
3.不得存储权限和身份。
4.同一诊断不重复添加。
5.顺序保持稳定。
6.仅用于日志、调试和后续页面展示。
7.不得根据diagnostics改变授权行为。

ContextTrimResult还应能够报告：

- estimatedTokensBefore
- estimatedTokensAfter

如果本批接入TokenCounter。

# 十四、线程安全与不可变性

1.ContextTrimmer必须无状态。
2.ContextMessageGrouper必须无状态。
3.ContextMessageHistoryValidator必须无状态。
4.TokenCounter必须可并发复用。
5.不得使用static可变集合。
6.不得使用ThreadLocal。
7.不得缓存上一次请求消息。
8.不得修改输入AgentMessage。
9.不得返回输入可变集合的直接引用。
10.不得跨用户复用裁剪结果。
11.同一Bean必须支持并发Agent运行。
12.不得为所有请求增加全局锁。

# 十五、Spring装配

在agent-infrastructure现有配置体系中装配：

- ContextMessageHistoryValidator
- ContextMessageGrouper
- TokenCounter→HeuristicTokenCounter
- ContextTrimmer→MessageCountContextTrimmer

要求：

1.runtime实现不添加Spring注解。
2.使用`@Configuration`和`@Bean`。
3.TokenCounter允许通过`@ConditionalOnMissingBean`替换。
4.ContextTrimmer允许未来替换。
5.不得装配第二个ModelInvocationGateway。
6.不得装配第二个ReactAgentEngine。
7.无模型配置时上下文Bean仍能启动。
8.不得创建Fake模型进行摘要。
9.不得在bootstrap主启动类手工new。
10.不得产生循环依赖。

如未来需要同时存在多个ContextTrimmer，不要现在引入复杂Qualifier体系；本批只提供按消息数量的默认实现。

# 十六、配置

增加或复用配置：

```yaml
agent:
  context:
    enabled: true
    max-messages: 20
```

要求：

1.使用类型安全配置属性。
2.max-messages必须大于0。
3.默认值合理且集中定义。
4.不得在多个类中硬编码不同默认值。
5.enabled本批只控制Bean或后续处理开关。
6.本批尚未接入ReasonNode，因此开启后不改变现有模型请求。
7.不得加入尚未实现的summary配置。
8.不得加入max-tokens配置，留到第2批。
9.配置无效时启动失败并给出明确原因。
10.不得把配置保存进State。

# 十七、框架查询能力

最小扩展现有FrameworkQueryService或配置查询结果，使框架可查询当前上下文能力：

至少表达：

- enabled
- strategy=MESSAGE_COUNT
- maxMessages
- tokenCounterType
- exactTokenCount=false

可以扩展已有框架能力接口，或增加：

```text
GET /api/framework/context
```

要求：

1.优先复用现有框架查询Controller。
2.接口只读。
3.不得要求模型配置。
4.不得暴露Bean实现类完整包名。
5.不得暴露Prompt、消息或用户数据。
6.不得直接提供裁剪执行HTTP接口。
7.不得增加隐藏dev接口。
8.如增加新接口，保持统一响应风格。

# 十八、本批不接入运行链

必须确认本批不修改：

1.DefaultReactReasonNode的模型请求消息。
2.DefaultSupervisorReasonNode的模型请求消息。
3.ModelInvocationGateway的消息列表。
4.SpringAiModelClient的消息映射。
5.ReactAgentState中的messages。
6.SupervisorAgentState中的messages。
7.Checkpoint保存内容。
8.HITL恢复逻辑。
9.Agent正式API请求结构。
10.前端对话页面。

原因：

本批先验证裁剪算法和基础模型，第2批完成Token预算后再统一接入。

不得让部分节点提前使用裁剪、另一些节点仍发送完整消息，形成不一致行为。

# 十九、边界场景检查

通过代码检查或已有验证能力确认：

## 空消息

1.返回空retainedMessages。
2.统计全部为0。
3.不得抛NullPointerException。

## 只有System消息

1.全部保留。
2.非System数量为0。
3.不认为超出maxMessages。

## 数量未超限

1.保持全部消息。
2.保持对象顺序。
3.diagnostics包含NO_TRIMMING_REQUIRED。

## 普通消息超限

1.保留全部System消息。
2.保留最近N条非System消息。
3.最新User消息仍存在。

## 单工具调用

1.Assistant ToolCall与ToolResult同时保留或同时移除。
2.不得产生孤立ToolAgentMessage。

## 多工具调用

1.一个Assistant的多个ToolCall与全部结果构成原子组。
2.不得只保留部分ToolResult。

## 最新原子组超过限制

1.完整保留。
2.atomicGroupOvershoot正确。
3.不破坏工具配对。

## 非法历史

1.孤立ToolResult被明确拒绝。
2.缺失ToolResult被明确拒绝。
3.重复ToolResult被明确拒绝。
4.错误响应不泄露工具参数。

## Token计数

1.空集合为0。
2.非空消息大于0。
3.包含ToolCall的消息高于相同纯文本消息的合理计数。
4.相同输入结果稳定。
5.中文和英文均可估算。

不得为了验证增加隐藏Controller或测试脚本。

# 二十、本批禁止实现

禁止：

1.按Token数量裁剪。
2.Token预算预留。
3.LLM摘要。
4.摘要Prompt。
5.摘要消息类型。
6.将ContextTrimmer接入ReasonNode。
7.修改ModelInvocationGateway。
8.跨请求保存消息。
9.ConversationHistoryStore。
10.MemoryStore实现。
11.按threadId恢复完整对话。
12.长期用户画像。
13.数据库或Redis。
14.前端上下文页面。
15.上下文SSE事件。
16.模型供应商精确Tokenizer依赖。
17.隐藏调试Controller。
18.测试脚本、README和Git操作。

# 二十一、编译验收

执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

确认：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.ContextTrimmer和TokenCounter位于稳定模块。
4.默认实现位于runtime。
5.实现不依赖Spring AI消息类型。
6.System消息永久保留。
7.maxMessages只统计非System消息。
8.ToolCall和ToolResult不会被拆散。
9.多ToolCall原子组完整。
10.最新用户输入保留。
11.非法历史明确失败。
12.输入和输出集合保持不可变。
13.TokenCounter支持单条和集合。
14.默认TokenCounter明确属于估算。
15.上下文Bean无模型配置也能装配。
16.没有修改ReAct和Supervisor模型输入。
17.没有实现Token裁剪和摘要。
18.现有认证、HITL和前端功能未被破坏。
19.git diff没有无关修改。

# 二十二、最终输出

只输出：

1.新增和修改文件清单。
2.ContextTrimRequest和ContextTrimResult结构。
3.ContextTrimmer接口。
4.ContextMessageGroup划分规则。
5.ToolCall与ToolResult配对算法。
6.非法消息历史处理方式。
7.System消息保留规则。
8.MessageCountContextTrimmer裁剪算法。
9.最新用户输入保护规则。
10.TokenCounter接口。
11.HeuristicTokenCounter估算规则。
12.atomicGroupOvershoot语义。
13.配置及Spring Bean装配。
14.框架查询能力。
15.边界场景检查结果。
16.编译和打包结果。
17.发现但未处理的问题。

编译或打包失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE7_BATCH1_END===