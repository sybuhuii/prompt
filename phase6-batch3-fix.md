你现在需要修复当前项目 Phase6 Batch3 的实现。必须先完整阅读并遵守根目录 AGENTS.md，再完整阅读 `.kscc-prompts/phase6-batch3.md`，然后检查仓库真实代码、实际依赖版本和现有实现。

当前 `master` 已包含 KSCC 对 Phase6 Batch1～Batch4 的实现。本次不重做 Batch1、Batch2，不提前扩展或重构 Batch4，只修复 Batch3 的审批决策、Checkpoint、ReAct 恢复执行及必要的集成问题。

必须基于当前代码独立分析和实现，不要寻找、复制或依赖其他 Agent 生成的补丁、stash、commit、reflog或工作区残留内容。

## 一、范围约束

本次只处理 Phase6 Batch3：

1. 审批批准与拒绝的应用服务。
2. PendingApproval 的原子更新。
3. Checkpoint 恢复抢占。
4. ReactAgentState 重建。
5. 从 `execute_tools` 节点恢复。
6. 批准后真实执行危险工具。
7. 拒绝后生成失败 ToolResult，经 Observe 回灌模型。
8. 多危险工具连续挂起。
9. Checkpoint 完成、再次挂起和失败生命周期。
10. 决策幂等、并发恢复和用户隔离。
11. LangGraph4j 状态复制所需的序列化兼容性。
12. 无模型配置下审批决策能力的条件装配。

禁止：

- 新增或重写审批 Controller、HTTP API、前端页面和前端请求代码。
- 实现 Supervisor 恢复。
- 引入数据库、Redis、分布式锁、SSE或定时清理。
- 新增测试脚本、测试工程、README或验收文档。
- 修改 AGENTS.md 或 `.kscc-prompts`。
- 执行 git commit、push、merge、reset。
- 无关重构 Batch1、Batch2 或 Batch4。
- 创建第二套 ReAct 引擎、Checkpoint、Gateway或重复抽象。

如果 Batch3 的最小修复导致已有 Batch4 调用点无法编译，只允许进行必要的兼容性调整，不得借机重构或扩展 Batch4，并在最终输出中单独列明。

## 二、修改前检查

必须先检查：

- 根目录及相关模块 `pom.xml`。
- LangGraph4j 的真实版本和实际 API。
- `AgentCheckpoint`、`CheckpointStatus`、`CheckpointStore`。
- `InMemoryCheckpointStore`。
- `PendingApproval`、`ApprovalDecision`、`InterruptPayload`。
- `CheckpointValidator`、`ReactResumeValidator`。
- `ReactCheckpointService`。
- `ReactAgentState` 及其 StateKeys。
- `ReactAgentGraphFactory`。
- `DefaultReactToolExecutionNode`。
- `ToolApprovalInterceptor`。
- `ToolOperationFingerprint`。
- `ReactResumeEngine`。
- `ApprovalDecisionService`。
- `ApprovalResumeApplicationService`。
- Spring 配置和条件装配。
- `DemoRecordStore`、`DeleteDemoRecordTool`。
- `AgentErrorCode` 和异常映射。

以仓库真实构造器、字段、接口和框架 API 为准，不得凭记忆创造方法。

## 三、Checkpoint 和 Store 修复

检查并修复以下语义：

1. `AgentCheckpoint` 的状态约束必须完整，尤其是：
   - `SUSPENDED` 必须具有待审批数据。
   - `RESUMING` 必须具有已作出最终决定的审批。
   - `COMPLETED`、`FAILED` 不得再次恢复。
2. Checkpoint 的 `stateData` 必须是递归不可变快照：
   - Map、List、Set 等嵌套集合均需复制。
   - 不允许调用方修改快照后影响 Store 内数据。
3. `updateIfVersionMatches` 必须同时满足：
   - 当前版本等于 expectedVersion。
   - 新对象版本严格等于 `expectedVersion + 1`。
   - runId、checkpointId、userId、threadId、executionType 等稳定身份不得被替换。
4. 不得通过先读后无条件覆盖实现条件更新。
5. 同一 runId 的竞争必须原子处理，不同 runId 可以并发。
6. 禁止使用 JVM 全局锁串行全部恢复。
7. 重复提交相同审批决定时：
   - 返回已有决定。
   - 不修改 comment、decidedAt、decidedBy。
   - 不增加 Checkpoint version。
8. 相反决定必须返回 `APPROVAL_ALREADY_DECIDED`。
9. Checkpoint 删除必须避免误删同一 runId 后续产生的新 Checkpoint：
   - 如果当前接口不支持，最小增加按 checkpointId/version 条件删除能力。
   - 不允许完成旧恢复后无条件按 runId 删除。
10. Store 的 create/update/delete 语义必须分离，返回不可变快照。

## 四、审批决策服务

修复 `ApprovalDecisionService`：

1. 保持纯 Java，不使用 Spring 注解或 Web 类型。
2. operator 必须来自已验证 `UserSession`。
3. userId、decidedBy、decidedAt 不得来自请求命令。
4. 根据 runId 加载 Checkpoint，并校验归属用户。
5. 本批不允许管理员跨用户审批。
6. 只允许对 `SUSPENDED + PENDING` 的审批作首次决定。
7. approvalId 必须完全匹配。
8. decidedBy 使用 `operator.userId`。
9. decidedAt 使用注入的 `Clock`。
10. 使用 version 条件更新。
11. 决策服务只记录决定：
    - 不恢复图。
    - 不执行模型。
    - 不执行工具。
    - 不删除 Checkpoint。
12. comment 允许为空，但必须规范化并限制长度。
13. 相同决定幂等，相反决定冲突。
14. 不得直接修改已经存入 Store 的对象。

## 五、恢复前完整校验

修复 `ReactResumeValidator`，在抢占之前至少校验：

1. executionType 为 `REACT_AGENT`。
2. Checkpoint 状态为 `SUSPENDED`。
3. PendingApproval 存在。
4. 审批状态是 `APPROVED` 或 `REJECTED`。
5. ApprovalDecision 存在且字段有效。
6. resumeNode/nodeName 只能是 `execute_tools`。
7. runId、threadId、userId 合法且顶层信息一致。
8. stateData 能完整重建。
9. pendingToolCalls 非空。
10. cursor 范围合法。
11. cursor 对应 ToolCall ID 等于审批绑定的 toolCallId。
12. toolName 等于 operationName。
13. 使用现有 `ToolOperationFingerprint` 重新计算指纹并完全匹配。
14. State 中 RunContext 的 userId、runId、threadId 与 Checkpoint 顶层完全一致。
15. decidedBy 非空。
16. 不信任 stateData 中与顶层 Checkpoint 冲突的身份字段。

任何校验失败时：

- 不得把 Checkpoint 改为 `RESUMING`。
- 不得调用模型或工具。
- 抛出明确的结构化框架异常。
- 不得统一伪装成 `INTERNAL_ERROR`。

## 六、恢复抢占

修复或实现恢复协调逻辑：

1. 先加载并完整校验 Checkpoint。
2. 使用 `updateIfVersionMatches` 将状态从 `SUSPENDED` 原子切换到 `RESUMING`。
3. version 严格加一，updatedAt 使用 Clock。
4. 同一 runId 只有一个恢复请求能够抢占成功。
5. 第二个请求必须得到 `RUN_ALREADY_RESUMING` 或 `CHECKPOINT_CONFLICT`。
6. 不得无限重试。
7. 不得无条件覆盖或先删除再保存。
8. 不得在协调器中执行图、模型或工具。
9. 模型能力不可用时，应在抢占前返回明确的 `MODEL_NOT_AVAILABLE` 或现有等价错误，避免把 Checkpoint 永久留在 `RESUMING`。

## 七、State 重建与序列化

集中处理 Checkpoint 与 `ReactAgentState` 的映射。存在等价 Mapper 时复用，不要机械新增重复类。

必须保证：

1. 从 Checkpoint 的不可变 stateData 重建新 State。
2. 保留原：
   - AgentTask
   - AgentDefinition或稳定标识
   - RunContext
   - messages
   - pendingToolCalls
   - executionBuffer
   - executionCursor
   - iteration
   - maxIterations
   - checkpointId
   - runId
   - threadId
3. pendingApproval 替换为最新已决定对象。
4. currentStatus 切换为恢复运行状态。
5. finalResult 清空。
6. 不修改 Store 中保存的 stateData。
7. 不生成新的顶层 runId、threadId或Session。
8. 不重新调用产生当前 ToolCall 的 Reason。

当前 LangGraph4j 版本在运行期间会通过 Java 序列化复制图状态。必须检查所有实际存入 State/stateData 的对象树是否可序列化，至少包括审批相关对象及其嵌套对象。

已知需要重点检查：

- `PendingApproval`
- `ApprovalDecision`
- `InterruptPayload`
- AgentResult、RunContext、ToolResult、UserSession 等实际进入状态的数据
- 嵌套 Map/List 及其元素

要求：

- 只为确实进入图状态的稳定领域对象增加必要的 `Serializable` 支持。
- 声明稳定的 `serialVersionUID`。
- 不得让 agent-core 依赖 LangGraph4j。
- 不得通过 Jackson、反射或 Object 通用转换规避问题。
- 必须避免运行时出现：
  `java.io.NotSerializableException: com.ksyun.agent.core.approval.PendingApproval`

## 八、恢复图入口

检查 LangGraph4j 1.8.20 的真实能力。

如果没有可靠的原生指定节点恢复入口，在现有 `ReactAgentGraphFactory` 中做最小恢复构图：

1. 只允许白名单节点 `execute_tools`。
2. START 直接进入 `execute_tools`。
3. 后续节点、边和路由复用正常图定义。
4. 不复制节点实现。
5. 正常 `compile()` 行为保持不变。
6. 禁止任意客户端 nodeName。
7. 禁止反射、`Object` 包装和伪造不存在的 API。

恢复必须从 `execute_tools` 开始，不能重新进入首次 Reason。

## 九、ToolExecutionNode 恢复行为

修复 `DefaultReactToolExecutionNode`：

1. 普通首次执行时 approval 为空。
2. 恢复时只允许把 PendingApproval 注入 cursor 对应的那个 ToolCall。
3. 注入前必须完整匹配：
   - toolCallId
   - toolName
   - arguments fingerprint
   - runId和相关绑定信息
4. 不匹配时禁止执行工具，并返回明确结构化错误。
5. 不得把同一审批传给其他 ToolCall。
6. 所有真实执行仍必须经过 `ToolInvocationGateway`。
7. APPROVED：
   - 由 `ToolApprovalInterceptor` 再次验证审批和指纹。
   - 只执行一次真实工具。
8. REJECTED：
   - 不调用 TerminalToolExecutor。
   - 返回 `success=false`、`APPROVAL_REJECTED` 的 ToolResult。
   - 进入 executionBuffer。
   - 之后经 Observe 形成 `error=true` 的 ToolAgentMessage 回灌模型。
   - 不得直接在 Application Service 中伪造最终回答。
9. 当前 ToolCall 处理完成后：
   - cursor 前进。
   - 清空 pendingApproval。
   - 已完成结果保留在 executionBuffer。
10. 遇到后续危险 ToolCall：
    - 创建新的 PendingApproval和approvalId。
    - 再次挂起。
    - 不重新执行 cursor 之前的工具。
11. 全部 ToolCall 完成后：
    - buffer 转入 latestToolResults。
    - 清空 buffer。
    - cursor 重置。
    - 进入 Observe。

## 十、多次挂起和生命周期

修复 `ReactCheckpointService` 和生命周期服务：

1. 首次挂起：
   - 创建 version=0、status=SUSPENDED 的 Checkpoint。
2. 恢复后再次挂起：
   - 只能从当前 `RESUMING` Checkpoint 条件更新。
   - 保存最新 State、cursor、buffer和新 PendingApproval。
   - 状态回到 `SUSPENDED`。
   - version 加一。
   - 使用新 approvalId。
   - 禁止删除旧 Checkpoint后重新 save。
3. 恢复完成：
   - `RESUMING -> COMPLETED` 条件更新。
   - version 加一。
   - 再按 checkpointId/version 条件删除。
4. 恢复框架失败：
   - `RESUMING -> FAILED` 条件更新。
   - 保存安全 errorCode。
   - 不保存异常对象、堆栈或内部敏感数据。
   - 默认保留 FAILED Checkpoint供诊断。
5. 普通工具以 `ToolResult.success=false` 返回并被模型正常处理时，不得把 Checkpoint 标记为 FAILED。
6. 清理冲突不得误删其他请求创建的新审批。
7. 清理日志失败不得把一个真实成功的 AgentResult伪造成普通工具失败，但版本冲突必须明确暴露，不能静默吞掉。

## 十一、ReactResumeEngine

恢复流程必须是：

1. 检查恢复依赖和模型能力。
2. 完整验证。
3. 原子抢占为 RESUMING。
4. 从 Checkpoint 重建 State。
5. 编译白名单恢复图。
6. 从 `execute_tools` 执行。
7. 批准时真实执行危险工具一次。
8. 拒绝时生成失败 ToolResult并进入 Observe。
9. 后续 Reason继续生成最终回答。
10. 保持原 runId和threadId。
11. 再次挂起时返回新的 approvalId。
12. 正常完成时处理 Checkpoint终态和条件清理。
13. 框架异常时将已抢占 Checkpoint标记为 FAILED。
14. State重建、恢复图编译和图执行均必须纳入失败生命周期处理。
15. 不得创建新Session、提升权限或绕过Gateway。
16. 不得重新执行产生当前 ToolCall 的首次 Reason。

## 十二、工具业务幂等

`DeleteDemoRecordTool` 必须保持业务幂等：

1. 按 recordId删除。
2. 记录已不存在时返回正常的“already absent”结果。
3. 重复删除不产生额外副作用。
4. metadata可表达 `alreadyAbsent`。
5. 不得在工具内部绕过审批或实现ACL。
6. approvalId不能作为授权凭证。

## 十三、Spring装配

1. runtime和application保持纯 Java。
2. Bean统一通过 infrastructure的 `@Configuration + @Bean` 装配。
3. 使用 `@ConditionalOnMissingBean` 支持替换。
4. 不创建第二套节点、Gateway或Engine。
5. 避免循环依赖。
6. 无模型配置时：
   - Checkpoint Store、审批决策等非模型能力正常装配。
   - 实际 resume 返回明确模型不可用错误。
   - 不创建Fake模型。
7. 不得为了装配方便把 Spring 注解加到 runtime/application类上。

## 十四、安全和错误语义

复用现有错误码，缺少时最小补充，禁止重复语义：

- APPROVAL_NOT_FOUND
- APPROVAL_ALREADY_DECIDED
- INVALID_APPROVAL_DECISION
- CHECKPOINT_NOT_FOUND
- CHECKPOINT_CONFLICT
- CHECKPOINT_NOT_RESUMABLE
- RUN_ALREADY_RESUMING
- RESUME_FAILED
- MODEL_NOT_AVAILABLE或现有等价码

保证：

1. 错误用户不能审批或恢复其他用户的 Checkpoint。
2. 用户不匹配时不改变版本、不执行工具、不泄漏 stateData。
3. 不根据 username、角色名 ADMIN 或 approvalId单独授权。
4. 不记录 sessionId、完整参数、完整权限或 stateData。
5. 审批拒绝是普通工具失败，不是系统异常。
6. 客户端不能提交 decidedBy、decidedAt、userId、权限或 fingerprint。

## 十五、必须验证的场景

不得新增隐藏 Controller或测试脚本。使用已有能力进行真实验证；环境无法执行时必须准确说明，不能伪造。

至少检查：

1. APPROVE：
   - 初次运行 SUSPENDED。
   - 决策后恢复。
   - 从 execute_tools开始。
   - 危险工具真实执行一次。
   - Observe和后续Reason正常。
   - 最终 COMPLETED。
   - Checkpoint条件清理。
2. REJECT：
   - TerminalToolExecutor不执行。
   - 返回 APPROVAL_REJECTED ToolResult。
   - Observe产生 error=true消息。
   - 模型说明操作被拒绝。
   - 业务数据保持不变。
3. 重复相同决定：
   - 幂等。
   - version不增加。
   - 决策时间和comment不改变。
4. 冲突决定：
   - 返回 APPROVAL_ALREADY_DECIDED。
   - 原决定不被覆盖。
5. 重复或并发resume：
   - 只有一个请求抢占成功。
   - 工具最多执行一次。
6. 错误用户：
   - 拒绝。
   - 不改变Checkpoint。
   - 不泄漏数据。
7. 两个危险ToolCall：
   - 第一个批准后继续到第二个。
   - 第二个产生新approvalId并再次挂起。
   - 第一个不重复执行。
   - cursor和buffer正确。
8. 指纹不匹配：
   - 修改toolCallId、toolName或arguments任一项均不得执行工具。
9. 序列化：
   - 真实挂起和恢复过程中不能再出现 NotSerializableException。
10. 回归：
   - 认证、ACL、普通ReAct、Supervisor和Batch1/2行为不被破坏。

## 十六、构建要求

完成全部修改后，按照 AGENTS.md 的 Windows 构建环境执行要求，只在最终修改完成后执行规定的编译和打包命令各一次：

```powershell
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```
如果涉及了前端文件，说明已经违反本批范围；应撤销无关前端修改。本批不需要前端构建。
构建失败必须继续定位和修复；不能伪造成功。
十七、最终输出
最终只报告：
新增和修改文件。
每个文件对应的Batch3缺陷。
审批决定状态转换和幂等规则。
Checkpoint版本更新、恢复抢占和条件删除流程。
State重建字段及序列化处理。
ReactResumeValidator完整校验顺序。
恢复图入口。
ToolExecutionNode审批注入及拒绝回灌流程。
多危险工具再次挂起流程。
Checkpoint完成、失败和清理策略。
用户隔离和错误码处理。
Spring条件装配行为。
编译及打包的真实结果。
实际完成的批准、拒绝、重复决定、并发恢复和序列化验证。
未验证项目及准确原因。
发现但未处理的Batch4或其他非本批问题。
不要把计划、推测或未执行的验证描述成已完成结果。