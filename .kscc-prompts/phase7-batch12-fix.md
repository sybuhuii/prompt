# Phase 7 Batch 1、Batch 2 审查与修复要求

请严格按照仓库根目录 `AGENTS.md`、`.kscc-prompts/phase7-batch1.md` 和 `.kscc-prompts/phase7-batch2.md` 审查并修复当前实现。

本次只修复 Phase7 Batch1、Batch2，不得提前实现或重写 Batch3、Batch4。已经正确的代码不要重复修改。不得修改 AGENTS.md、提示词文件、README，不得新增无关测试，不得执行 git commit/push，不得进行无关重构。

开始修改前必须检查当前真实代码、模块 pom.xml、消息模型、工具调用模型、错误码、配置类和实际构造器。禁止凭记忆猜测接口。

## 一、修复消息历史结构校验

### 重点检查

- `ContextMessageValidator`
- `ContextMessageGrouper`
- `ContextMessageGroup`
- `ContextMessageGroupType`
- `AssistantAgentMessage`
- `ToolAgentMessage`
- `ToolCall`

当前校验存在接受非法工具消息顺序的风险。

### 验证规则

1. System、User、普通 Assistant 分别形成独立消息组。
2. 带 ToolCall 的 Assistant 必须与其后连续出现的 ToolResult 形成一个完整原子组。
3. ToolResult 不允许脱离前置 Assistant ToolCall 单独出现。
4. ToolResult 必须紧跟对应的 Assistant ToolCall 组，二者之间不得插入：
    - User
    - System
    - 普通 Assistant
    - 另一个带 ToolCall 的 Assistant
5. 每个 ToolCall 必须且只能对应一个 ToolResult。
6. ToolResult 的 `toolCallId` 必须与 ToolCall ID 精确匹配。
7. ToolResult 的 `toolName` 必须与对应 ToolCall 名称精确匹配。
8. 不得接受：
    - 缺少结果的 ToolCall
    - 重复 ToolResult
    - 未知 toolCallId
    - ToolResult 数量不足或过多
    - ToolResult 顺序错乱
    - 同一个 ToolResult 被多个 ToolCall 复用
9. 如果提示词要求 ToolResult 与 ToolCall 顺序一一对应，必须按原始顺序验证，不能只用 Map 或 Set 判断“最终都存在”。
10. 校验失败必须使用已有 `AgentErrorCode` 和 `AgentFrameworkException`，不得抛无语义 RuntimeException。
11. 异常响应不得包含完整消息、完整工具参数或敏感内容。

## 二、重写有问题的消息分组逻辑

当前 Grouper 存在根据起止索引范围二次重建消息组的风险，可能把无关消息卷入工具组、重复消息或遗漏消息。

### 要求

1. 使用明确的单次顺序扫描构建消息组。
2. 每条输入消息必须恰好属于一个组。
3. 组内消息必须来自原历史中的连续区间。
4. 不得先收集若干索引，再使用最小/最大索引范围重新截取。
5. 不得通过反射、Object 或大量 `instanceof` 分派创建通用分组框架。
6. 工具组必须包含：
    - 一条带 ToolCall 的 Assistant
    - 对应的全部 ToolResult
7. 工具组的 `startIndex`、`endIndex` 必须精确对应原始历史。
8. 所有组按原始历史顺序返回。
9. Grouper 不得静默修复非法历史，非法结构应由 Validator 明确拒绝。
10. 输入和输出集合必须使用防御性复制。
11. 不得把输入 List 的可变引用保存在 `ContextMessageGroup` 内。

### 修复后确认

所有 `group.messages` 拼接后的结果必须与原始 `messages` 完全相同：数量相同、顺序相同、对象语义相同，不得重复、遗漏或重排。

## 三、修复 `ContextTokenBudget` 可伪造问题

重点检查 `ContextTokenBudget`。

当前调用者可能通过公开构造器传入彼此不一致的字段，例如最大 Token、各项保留 Token 与有效输入预算不匹配。

### 要求

`effectiveInputTokens` 必须由以下值唯一计算，不能由调用者随意指定：

```text
maxContextTokens
- reservedOutputTokens
- reservedProtocolTokens
- safetyMarginTokens
```

优先使用现有 `calculate(...)` 工厂创建预算。

如果保留公开构造器，构造器必须重新计算并验证派生字段完全一致；不能接受伪造结果。

更合适时可以改成 private constructor + public static factory，但必须检查全部调用点，不能破坏现有模块。

校验条件：

- `maxContextTokens > 0`
- 各 `reserved` 值 `>= 0`
- 保留值总和小于 `maxContextTokens`
- `effectiveInputTokens > 0`

计算过程使用 long 防止 int 溢出，确认合法后再安全转换。
不得静默钳制非法值。
配置错误应尽早抛出明确结构化错误。
不得允许请求 DTO 或客户端直接构造可信预算。
所有预算必须来自服务器配置。

## 四、修复 `TokenCounter` 空依赖问题

检查所有依赖 `TokenCounter` 的 Batch1、Batch2 类型，包括但不限于：

- `ContextSummaryTrigger`（如已存在，但不要修改 Batch3 业务）
- `ContextMessageGrouper`
- `MessageCountContextTrimmer`
- `TokenCountContextTrimmer`
- `ContextProcessingPipeline`
- 相关 Selector、统计类型

### 要求

- 构造器对 `TokenCounter` 使用 `Objects.requireNonNull`。
- 其他必需依赖同样必须 fail fast。
- 禁止允许 null 后在运行到一半时触发 NPE。
- 禁止 null 时自动创建隐藏默认实现。
- 默认 `HeuristicTokenCounter` 只能在 Spring 配置类中通过 `@Bean` 和 `@ConditionalOnMissingBean` 装配。
- runtime 保持纯 Java，不添加 Spring 注解。
- 用户提供自定义 TokenCounter 时，所有 Trimmer、Pipeline 和统计必须使用同一个被装配的实例。

## 五、修复不可变集合实现

检查所有 Phase7 Batch1、Batch2 的 record、结果和分组类型，例如：

- `ContextMessageGroup`
- `ContextTrimRequest`
- `ContextTrimResult`
- `ContextProcessingRequest`
- `ContextProcessingResult`
- 诊断码集合
- retained/removed message 集合
- metadata 或统计集合

### 要求

构造时使用：

- `List.copyOf`
- `Set.copyOf`
- `Map.copyOf`

不得只使用：

```java
Collections.unmodifiableList(input)
```

- 因为调用者仍可通过原始 List 修改内部数据。
- 不得直接返回内部可变集合。
- 空集合返回 List.of()，不得返回 null。
- 对外返回不可变快照。
- 集合元素不允许为 null。
- 不要对本类新建且不再泄漏的局部集合做无意义的多层包装，但领域对象边界必须防御性复制。

## 六、修复消息数裁剪的强制保留范围

检查 `MessageCountContextTrimmer`。

必须按照 Batch2 提示词准确实现强制保留语义：

- 保留全部 System 消息。
- 保留最新 User 消息所在完整组。
- 保留最新 User 消息之后的所有完整组。
- 如果最新 User 后存在工具调用组，必须整体保留。
- 最新且仍属于当前对话的完整工具组不得被错误当作普通旧消息删除。
- 工具组不可拆分。
- 从旧到新删除非强制保留组，直到满足消息数预算。
- 如果为了保持原子组而略微超过消息数上限，按照 Batch2 提示词记录 `ATOMIC_GROUP_OVERSHOOT` 或提示词规定的对应诊断。

- 不得为了硬凑 `maxMessages` 拆开 Assistant ToolCall 和 ToolResult。
- 不得删除最新 User 后的 Assistant 最终回答。
- 输入本来满足预算时不得重排消息，返回 `NO_TRIMMING_REQUIRED`。
- `removedMessageCount` 必须由真实差值或实际移除结果生成，不得固定填写。
- `retainedMessages` 拼接顺序必须与原始历史一致。

## 七、修复 Token 裁剪的最新工具组处理

检查 `TokenCountContextTrimmer`。

### 要求

- 所有 System 消息都属于强制上下文。
- 最新 User 所在组及其后的所有组属于强制上下文。
- 最新 User 后出现的工具调用组必须整体保留。
- 不得只保留 ToolResult 而删除发起 ToolCall 的 Assistant。
- 不得只保留 Assistant ToolCall 而删除部分 ToolResult。
- 旧工具组只能整组保留或整组删除。
- 必须按照组的完整 Token 成本做预算，不能逐消息计算后拆组。
- System 消息本身超过预算时，抛出或返回提示词要求的 `SYSTEM_TOKEN_BUDGET_EXCEEDED`。
- 全部强制上下文超过预算时，使用 `MANDATORY_CONTEXT_TOO_LARGE`。
- 普通旧工具组因预算不足被整体跳过时，记录 `TOOL_GROUP_SKIPPED_DUE_TO_BUDGET`。
- 普通旧消息被删除时，记录 `OLD_MESSAGES_DROPPED`。
- 最终返回前必须重新对完整 `retainedMessages` 计数。
- 最终 Token 未超过有效预算时记录 `FINAL_TOKEN_BUDGET_VERIFIED`。
- 不得只相信增减过程中的临时计数。
- Token 加法使用 long 或安全加法，防止溢出。
- 不允许返回一个自称满足预算、实际重新计数后超预算的结果。

## 八、修复最新工具组识别方式

不得通过“列表末尾若干消息”“最后一个 ToolResult”或简单索引范围猜测最新工具组。

### 要求

- 必须基于 Validator 验证后的 ContextMessageGroup 判断。
- 使用明确的 group type。
- 最新 User 之后的所有组按原始顺序进入 mandatory groups。
- 最新 User 之前的工具组才可能成为可裁剪旧组。
- 没有 User 消息时，严格按照 Batch2 提示词规定的 fallback 处理，不得自行假定用户边界。
- 只有 System 消息时必须安全返回。
- 历史以完整工具组结束时必须安全处理。
- 历史存在非法未完成工具组时必须拒绝，不能裁剪后伪装成合法历史。

## 九、修复裁剪结果统计和诊断

检查 `ContextTrimResult`、`ContextProcessingResult` 和相关 Trace/Capability 数据。

### 要求

以下数据必须根据真实输入输出计算：

- `originalMessageCount`
- `retainedMessageCount`
- `removedMessageCount`
- `originalTokenCount`
- `retainedTokenCount`
- `effectiveMessageBudget`
- `messageCountTrimmed`
- `tokenTrimmed`

- 不得使用固定结果。
- removedMessageCount 不得出现负数。
- Token 统计使用同一个 TokenCounter。
- 诊断码使用类型安全 enum，不使用字符串拼接判断。
- 诊断集合去重且保持提示词要求的稳定顺序。
- 没有裁剪时不能错误报告 TRIM_APPLIED。
- 实际删除消息时必须报告对应裁剪诊断。
- System、latest User、工具组保留诊断必须与真实结果一致。
- 不得把内部异常、完整消息或工具参数放入 diagnostics。

## 十、修复 Batch2 Pipeline 顺序和最终校验

检查 `ContextProcessingPipeline`，但不要重写 Batch3 已添加的摘要流程。

### 要求

1. 如果摘要启用并触发：先摘要。
2. 消息数裁剪。
3. Token 裁剪。
4. 最终结构校验。
5. 最终 Token 重新计数。

- 本次重点修复 Batch1、Batch2 基础能力，不得删除 Batch3 摘要阶段。
- 每个阶段的输出必须作为下一个阶段的输入。
- 不得让 Token 裁剪重新使用原始历史，覆盖消息数裁剪结果。
- 最终结果必须经过 Validator。
- 最终结果中的工具组必须完整。
- 最终 Token 统计必须基于最终消息列表。
- 任一阶段失败时使用现有结构化错误治理。
- 禁止吞异常后返回固定成功结果。
- 无需裁剪时必须保持原消息全局顺序。

## 十一、修复配置与条件装配

### 重点检查

- `ContextProperties`
- `AgentFrameworkAutoConfiguration`
- Batch1、Batch2 相关配置类
- capability 查询实现

### 要求

- runtime/core 中不得添加 Spring 注解。
- Bean 统一在 infrastructure 的 `@Configuration` 中装配。
- 默认实现使用 `@ConditionalOnMissingBean`。
- 无模型配置时，`TokenCounter`、Validator、Grouper、Trimmer、Pipeline 和 Context capability 必须可以正常启动。
- Batch1、Batch2 的纯上下文能力不得依赖 `ModelInvocationGateway`。
- `agent.context.enabled=false` 时，运行链路应使用完整历史，不执行裁剪。
- message-count trimming 和 token trimming 的开关必须分别生效。
- 两个裁剪开关都关闭时，不得由 RequestFactory 私自重新启用消息数裁剪。
- 不得把配置关闭解释为“使用默认开启”。

配置数值必须在启动阶段校验：

- `maxMessages > 0`
- `maxContextTokens > 0`
- `reserved` 值 `>= 0`
- 有效输入预算 `> 0`

capability 返回真实配置和真实 Bean 可用性，不得固定填写。

## 十二、修复 `ContextProcessingRequestFactory` 的关闭语义

如果当前 `ContextProcessingRequestFactory` 在两个裁剪开关都关闭时自动打开 message count trimming，必须修复。

### 要求

- 严格保留配置中的两个开关值。
- 两个开关均关闭时，创建合法的“无需裁剪”请求或由 WindowManager 直接跳过 Pipeline。
- 不得静默篡改用户配置。
- 禁止为了绕过构造器校验而虚构裁剪开关。
- Request 自身应允许表达提示词规定的关闭状态。
- context 总开关关闭时不创建 snapshot/trace，不改变模型消息历史。

## 十三、边界情况回归检查

必须逐项检查以下场景：

- 空消息列表。
- 只有一条 System。
- 多条 System 穿插在历史中。
- 只有一条 User。
- User + 普通 Assistant。
- User + Assistant ToolCall + 全部 ToolResult。
- 一个 Assistant 包含多个 ToolCall。
- 多个 ToolResult 顺序正确。
- 多个 ToolResult 顺序错误。
- 缺少一个 ToolResult。
- 重复 ToolResult。
- 孤立 ToolResult。
- ToolCall 与 ToolResult 工具名不一致。
- 工具组中间插入 User/System/Assistant。
- 历史以完整工具组结束。
- 最新 User 后存在一个或多个完整工具组。
- maxMessages 小于强制保留消息数。
- System Token 已超过预算。
- mandatory groups Token 超过预算。
- 一个旧工具组加入后会超过预算。
- 无需裁剪。
- 两个裁剪开关均关闭。
- 自定义 TokenCounter Bean 替换默认实现。
- 外部修改传入 List 后，领域对象内容不发生变化。
- 直接尝试构造不一致的 `ContextTokenBudget` 时被拒绝。

不得为了这些检查新增独立测试工程或测试脚本。可以运行仓库已有测试；如仓库已有相关测试，应修复并运行。

## 十四、禁止事项

- 不得添加 Fake Model、Fake Tool 或固定上下文结果。
- 不得通过删除 Validator 绕过非法历史。
- 不得把工具组简单当作若干独立消息裁剪。
- 不得使用反射、Object 通用业务参数或 instanceof 类型分派框架。
- 不得创建重复的 Grouper、Validator、Trimmer V2。
- 不得修改 Controller、登录、权限、HITL 等无关功能。
- 不得提前重写 Batch3 摘要业务或 Batch4 页面。
- 不得修改 Git 配置或执行提交。
- 不得把未执行的验证描述成通过。

## 十五、最终验证

完成全部代码修改后，按照 AGENTS.md 的 Windows 环境和命令只执行一次完整构建验证：

```powershell
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

- 如果本次没有修改前端，不要无意义修改前端文件；可按照提示词和 AGENTS.md 判断是否需要前端构建。
- 如果已有相关测试，运行必要的现有测试。
- 构建失败必须继续修复，不能通过跳过模块、删除代码或注释功能伪装成功。

### 最终回复必须包含

- 新增和修改的文件。
- 每个问题的具体修复方式。
- Validator 和 Grouper 的消息处理规则。
- MessageCount 与 TokenCount Trimmer 的完整保留/裁剪规则。
- ContextTokenBudget 如何保证不可伪造。
- 条件装配和配置关闭时的行为。
- 编译、打包及已有测试的真实结果。
- 未完成验证及准确原因。
- 发现但未处理的非 Batch1、Batch2 问题。

不得把计划写成完成结果。
