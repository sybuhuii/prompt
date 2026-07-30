请严格按照仓库根目录 AGENTS.md、`.kscc-prompts/phase7-batch3.md` 和 `.kscc-prompts/phase7-batch4.md` 修复当前代码。以仓库真实包名、接口、依赖版本和现有实现为准，不要凭记忆创建框架 API。

本次只修复 Phase7 Batch3、Batch4 的以下问题，不得修改 AGENTS.md，不得修改提示词文件，不得执行 git commit/push，不得新增 README、验收报告或无关测试，不得进行无关重构。优先复用现有类型，禁止创建 V2/New/Another 等重复抽象。完成全部修改后按照 AGENTS.md 只执行一次完整编译和构建。

一、修复 SummaryAgentMessage Token 计算

检查 `HeuristicTokenCounter`。

当前实现没有处理 `SummaryAgentMessage`，导致摘要正文不参与 Token 计算，`summaryMaxTokens` 校验和上下文预算失效。

要求：

1. 为 `SummaryAgentMessage` 增加明确分支。
2. 使用与其他文本消息相同的文本 Token 估算方式计算 `content`。
3. 保留每条消息固定开销。
4. 不得使用反射、Object 分派或把 Summary 当成 System 强制转换。
5. 确认摘要生成后的 Token 校验和最终窗口 Token 统计都使用修复后的计数结果。

二、修复摘要源选择和最近组保留算法

检查 `ContextSummarySelector`。

要求同时满足：

1. 保留全部 System 消息。
2. 保留最新 User 消息所在的完整消息组及其后的所有组。
3. 至少保留最后 `recentGroupsToPreserve` 个完整 content group。
4. 上述两个保留范围取并集，不得出现要求保留4组却只保留2组的情况。
5. 不得拆分 Assistant ToolCall 与配套 ToolResult 原子组。
6. 摘要源只能是保留范围之前、允许摘要的完整旧 content group。
7. 必须保持所有保留消息在原始历史中的全局相对顺序，禁止把穿插在历史中的所有 System 消息集中移动到最前面。
8. 旧 `SummaryAgentMessage` 被新摘要替换时，必须正确计入：
    - sourceMessageCount
    - sourceTokenCount
    - sourceGroupCount
    - summarizedMessageCount
9. `firstSourceIndex`、`insertionIndex` 必须基于原始消息索引计算，不能基于重排后的列表。
10. 返回集合必须使用防御性复制。

修复时请重点检查当前 `preserveFromIndex` 条件。正确语义是：取“最新 User 所在组起点”和“最后 N 组起点”中更靠前的位置。

三、修复摘要合并顺序

检查 `ContextSummaryMerger`。

要求：

1. 真正使用 Selector 计算出的精确插入位置。
2. 用一条新 `SummaryAgentMessage` 替换被摘要的旧消息和旧摘要。
3. 保留 System 和未摘要消息的原始全局相对顺序。
4. 禁止把所有 System 消息统一移动到新摘要之前。
5. 合并完成后必须经过现有消息结构校验器验证。
6. 不得产生两个 Summary。
7. 不得把 Summary 插入未完成的工具调用组。
8. 返回 `List.copyOf(...)` 或等价的真实不可变快照。

四、修复 SOURCE_TOO_SMALL 诊断不可达

当前 Selector 在源 Token 少于 `summaryMinSourceTokens` 时直接返回空选择，Pipeline 随后把它报告为 `SUMMARY_SKIPPED_NO_SOURCE`，导致 `SUMMARY_SKIPPED_SOURCE_TOO_SMALL` 分支不可达。

要求：

1. “确实没有可摘要源”和“存在可摘要源但 Token 太少”必须能区分。
2. 有源但 Token 太少时必须返回：
   `SUMMARY_SKIPPED_SOURCE_TOO_SMALL`
3. 无可摘要旧消息时才返回：
   `SUMMARY_SKIPPED_NO_SOURCE`
4. 不得通过字符串判断原因。
5. 优先使用现有结构表达状态；如确实需要扩展 Selection，做最小、类型安全的扩展。

五、修复摘要 Prompt 的敏感信息泄漏

检查 `ContextSummaryPromptBuilder` 和现有 `SensitiveValueSanitizer` 实现。

当前只把 Tool 文本放进键名为 `content` 的 Map，无法过滤正文中的 password、token、API Key；User、Assistant 和旧摘要正文也没有统一脱敏。

要求：

1. 所有可能进入摘要模型的文本必须统一脱敏：
    - User content
    - Assistant content
    - ToolResult content
    - existing summary content
2. 不得把 ToolCall 完整参数发送给摘要模型，只允许发送必要的工具名和安全描述。
3. 不得仅依赖 Map 的字段名识别敏感值，必须支持文本正文中的常见敏感内容。
4. 优先扩展或复用现有 sanitizer，不得另建职责重复的 sanitizer。
5. 不得把密码、credentialHash、sessionId、API Key、Bearer Token、完整权限集合发送给模型。
6. 保留固定、不可被历史消息覆盖的摘要 System Prompt。
7. 不得把历史消息拼接成可以突破边界标签的未隔离 Prompt。
8. 不要无依据地把每条消息固定截断为500字符。如果需要控制输入，应根据已有 Token 预算和提示词要求处理，不能静默丢失摘要源的重要内容。

六、修复摘要模型最大输出 Token

检查 `LlmContextSummarizer` 和当前实际的 ModelRequest/Spring AI 适配器。

要求：

1. 摘要模型只调用一次。
2. tools 必须为空。
3. 将 `request.maxSummaryTokens()` 通过当前仓库真实支持的 ModelRequest option 传给模型。
4. 不得凭记忆创造 option 名；先检查当前 Spring AI 适配器实际读取的 option。
5. 模型返回后继续使用 TokenCounter 做二次校验。
6. ToolCall、空文本、空白文本、超长摘要均按现有结构化错误处理并降级到裁剪。
7. 不得添加 Fake Model 或固定摘要。
8. 摘要统计必须包含被替换的旧摘要。

七、修复摘要 Bean 和配置校验

检查 `AgentFrameworkAutoConfiguration`、`ContextProperties`、`ContextSummaryOptions` 和相关装配。

要求：

1. `ContextSummarizer` 只有在以下条件同时成立时才创建：
    - `agent.context.summary.enabled=true`
    - `ModelInvocationGateway` 存在
    - 用户没有提供自定义 `ContextSummarizer`
2. 摘要关闭时不得强制创建摘要器或要求模型存在。
3. 无模型配置时，非模型上下文处理能力必须正常启动。
4. 增加配置组合校验：
    - summary max tokens 必须大于0
    - recent groups 至少为1
    - trigger ratio 在合法区间
    - min source tokens 大于0
    - 在启用相关 Token 预算时，`max-summary-tokens` 必须小于有效上下文输入预算
5. 配置错误应在启动阶段给出明确错误，不能运行到模型调用时才失败。
6. 消除重复维护的默认值来源，按照 Batch3 提示词要求集中默认配置；不要创建平行配置体系。

八、修复无模型环境的 Context Demo

当前 `ContextDemoApplicationService` 整体受 `@ConditionalOnBean(ModelInvocationGateway.class)` 限制，导致 `invokeModel=false` 时无模型环境仍然不可用。

要求：

1. `ContextDemoApplicationService` 必须在没有模型 Bean 时也能创建。
2. `ModelInvocationGateway` 使用 `ObjectProvider`、`Optional` 或仓库现有的可选依赖方式注入。
3. `invokeModel=false`：
    - 只生成合成长历史；
    - 调用真实 ContextProcessingPipeline；
    - 返回真实裁剪、摘要可用性和诊断结果；
    - 不调用模型；
    - 无模型配置也必须成功。
4. `invokeModel=true` 且模型不存在时，返回明确的 `MODEL_NOT_AVAILABLE`，不得影响纯处理模式。
5. 不得创建 Fake ModelClient 或固定模型结果。
6. `/api/framework/context` 中的 `demoAvailable` 应表示纯处理 Demo 是否可用，不能简单等同于模型是否存在。
7. `summaryAvailable` 应根据真实摘要器是否存在计算。

九、修复 Demo RunContext 身份来源

当前 Demo 使用硬编码的 `demo-user`、`demo-session`、`demo-thread`。

要求：

1. userId、sessionId、roles、permissions 必须来自 Controller 注入并验证后的 `UserSession`。
2. runId 使用服务器端 `RunIdGenerator`。
3. threadId 如有需要必须由服务器生成，不得由客户端提交，也不得硬编码。
4. 不得把 RunContext、userId、sessionId、roles、permissions发送给模型。
5. 日志不得记录 sessionId 和完整权限集合。
6. 不得根据用户名或 ADMIN 角色特殊放行。

同时修复模型调用状态：

- 实际开始调用模型后，即使模型返回异常，也不能显示成“模型从未调用”。
- 可以使用现有字段的准确语义，或在不破坏接口的前提下最小增加 `modelInvocationAttempted` / `modelInvocationSucceeded`。
- 前后端字段语义必须一致。

十、修复 ContextWindowSnapshot 等不可变对象

检查以下类型及同类 Phase7 类型：

- `ContextWindowSnapshot`
- `ContextWindowUpdate`
- `ContextSummarySelection`
- `ContextDemoResult`
- 其他保存消息、诊断码或集合的 record

要求：

1. 构造时使用 `List.copyOf`、`Set.copyOf`、`Map.copyOf` 做防御性复制。
2. 不能只用 `Collections.unmodifiableList(input)` 包装调用者传入的可变集合。
3. `ContextWindowUpdate.modelMessages` 必须与 `snapshot.windowMessages()` 一致。
4. `ContextWindowUpdate.trace` 必须与 `snapshot.latestTrace()` 一致；构造器必须校验或直接采用 snapshot 中的值。
5. Checkpoint 保存和恢复后不得因外部集合修改而改变窗口内容。
6. 不得引入可变 static 状态或 ThreadLocal。

十一、修复 ReAct/Supervisor 上下文异常闭环

检查：

- `DefaultReactReasonNode`
- `DefaultSupervisorReasonNode`
- React/Supervisor failure node
- 现有 AgentErrorCode 和失败状态字段

当前 `contextWindowManager.update(...)` 在模型调用 try/catch 外执行，上下文处理异常会直接逃出图。

要求：

1. 捕获上下文处理产生的 `AgentFrameworkException`。
2. 使用现有图失败状态和失败节点形成标准 `AgentResult`。
3. 不得把上下文处理错误错误映射成模型调用失败。
4. failure node 必须保留已有的上下文错误码，不能全部退化为 `INTERNAL_ERROR`。
5. 如果已经产生最新 Trace，应把 Trace 合并进失败结果 metadata。
6. 不得把异常堆栈、内部类名、文件路径或原始 Prompt 返回客户端。
7. ReAct 与 Supervisor 使用一致的错误治理语义。
8. 上下文关闭时继续直接使用完整历史，不创建 snapshot/trace。
9. 上下文处理成功后，模型只接收窗口消息，完整 state.messages 仍保留完整历史。

十二、修复 Context Demo Controller

当前 Controller 自己构造错误 Map、映射状态码并返回原始异常 message。

要求：

1. Controller 保持薄层，只完成 DTO 转换并调用 Application Service。
2. 参数校验优先使用仓库现有 DTO 校验和统一异常处理方式。
3. 复用现有 `AgentFrameworkException`、`AgentErrorCode` 和全局 HTTP 异常映射。
4. 不得在 Controller 内维护重复的错误码到 HTTP 状态映射。
5. 不得把不安全的内部异常 message 原样返回。
6. 无模型但 `invokeModel=false` 时 Controller 必须能够正常调用 Demo Service。

十三、修复 Batch4 前端验收偏差

检查 `ContextManagementView.vue`、Context API DTO 和相关页面。

要求：

1. 后端直接返回并由前端展示以下统计，不得由前端自行推导：
    - removedMessageCount
    - tokenUsageRatio 或提示词要求的等价统计
    - 其他 Phase7 Batch4 明确要求由后端提供的统计
2. `runId` 页面只显示短 ID，复用项目已有 `shortId` 方式；请求和内部状态仍保存完整 runId。
3. 补全所有 ContextTrimDiagnostic 的中文标签，包括但不限于：
    - SUMMARY_SKIPPED_SOURCE_TOO_SMALL
    - SYSTEM_TOKEN_BUDGET_EXCEEDED
    - MANDATORY_CONTEXT_TOO_LARGE
    - TOOL_GROUP_SKIPPED_DUE_TO_BUDGET
    - OLD_MESSAGES_DROPPED
    - 以及枚举中其他可能返回的诊断码
4. 未知诊断码要提供安全、可理解的兜底显示，不要直接把内部枚举当主要用户文案。
5. capability 请求失败不得空 catch，页面应展示安全错误信息。
6. 请求期间继续禁用重复提交。
7. 前端不得调用 `/api/dev/**`。
8. Session、401、403 继续使用现有统一 HTTP 客户端处理。
9. 前端不得保存密码、API Key、hash 或把 sessionId 放入 URL/Body。
10. 完成后执行真实前端生产构建。

十四、回归检查

修复后逐项检查：

1. Summary 正文 Token 被正确计算。
2. 最近 N 个完整组和最新 User 之后的所有组均保留。
3. System 全局相对顺序不变。
4. 工具调用组不被拆分。
5. 再次摘要只保留一条新 Summary，旧 Summary 被正确计入统计。
6. 源太小时返回 `SUMMARY_SKIPPED_SOURCE_TOO_SMALL`。
7. 摘要 Prompt 不包含密码、Token、API Key、sessionId 或完整 Tool 参数。
8. 摘要模型调用 tools 为空且带最大输出 Token 限制。
9. 摘要关闭时不创建摘要器。
10. 无模型启动成功。
11. 无模型环境下 `invokeModel=false` 的 Demo 成功。
12. 无模型环境下 `invokeModel=true` 返回明确不可用结果。
13. ReAct/Supervisor 的完整历史不被窗口替换。
14. Checkpoint 恢复后窗口和 Trace 可继续增量处理。
15. 上下文异常形成标准失败 AgentResult，并保留准确错误码。
16. 前端显示短 runId，所有统计来自后端。
17. 不新增 Fake、固定答案、隐藏接口或权限绕过。

十五、最终验证和输出

完成全部修改后，严格按照 AGENTS.md 的构建要求只执行一次完整验证：

1. 后端 clean compile。
2. agent-bootstrap package。
3. 如果修改前端，执行真实前端生产构建。
4. 如环境允许，验证无模型启动以及 Context capability/纯处理 Demo。
5. 不要为了通过编译跳过模块或注释功能。

最终回复必须列出：

1. 新增和修改的文件。
2. 每个问题对应的修复方式。
3. Batch3 摘要选择、生成、合并、降级的完整调用链。
4. Batch4 ReAct/Supervisor 窗口更新、Checkpoint 和 metadata 调用链。
5. 无模型和摘要关闭时的条件装配行为。
6. 编译、打包、前端构建和接口验证的真实结果。
7. 未完成验证及准确原因。
8. 发现但未处理的非本次问题。

不得把未执行的验证描述成已通过。