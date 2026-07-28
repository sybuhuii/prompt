你现在需要修复当前项目 Phase6 Batch3、Batch4 审查中剩余的问题。

必须先完整阅读并遵守根目录 `AGENTS.md`，再阅读：

- `.kscc-prompts/phase6-batch3.md`
- `.kscc-prompts/phase6-batch4.md`
- 当前仓库真实代码和实际依赖版本

当前基线为包含前一次修复的 `master`。后端已经能够编译、打包，前端也能构建；本次问题主要是运行语义、并发安全、错误映射和前端状态同步。

必须独立检查并实现，不得寻找、复制或依赖其他Agent生成的补丁、stash、reflog或工作区残留。

## 一、范围

只修复本提示明确列出的问题，不重做Phase6，不提前实现Phase7。

禁止：

1. 无关重构认证、ACL、普通ReAct和Supervisor。
2. 新增第二套Checkpoint、ResumeEngine、Gateway或HTTP客户端。
3. 新增裸resume HTTP接口。
4. 引入数据库、Redis、分布式锁、SSE或WebSocket。
5. 新增测试脚本、README或验收文档。
6. 修改AGENTS.md和 `.kscc-prompts`。
7. 执行git commit、push、merge、reset。
8. 使用Fake模型或隐藏调试Controller。

## 二、修复再次挂起冲突

当前问题：

- `ReactCheckpointService` 在恢复后再次挂起时生成新的checkpointId。
- `InMemoryCheckpointStore.updateIfVersionMatches`要求checkpointId保持不变。
- 因此同一runId遇到第二个危险工具时必然发生 `CHECKPOINT_CONFLICT`。

修复要求：

1. 同一runId整个恢复生命周期保持原checkpointId。
2. 再次挂起只生成新的approvalId。
3. 保留原：
    - runId
    - threadId
    - userId
    - checkpointId
    - executionType
    - agentName
    - createdAt
4. 更新：
    - pendingApproval
    - stateData
    - executionCursor
    - executionBuffer
    - status=SUSPENDED
    - updatedAt
    - version=expectedVersion+1
5. 使用 `updateIfVersionMatches`。
6. 不得删除旧Checkpoint后重新save。
7. 第二个危险工具必须获得新的approvalId。
8. cursor之前的工具不得重复执行。

## 三、修复Checkpoint Store

检查并修复 `InMemoryCheckpointStore`：

### 稳定身份

条件更新时必须拒绝以下字段发生变化：

- runId
- checkpointId
- threadId
- userId
- sessionId
- executionType
- agentName
- createdAt

nodeName只有在现有生命周期明确允许时才能变化；当前恢复语义应保持 `execute_tools`。

不得再保留“threadId变化时更新索引”的正常更新路径。

### 版本

1. 当前存储版本必须等于expectedVersion。
2. 新版本必须严格等于 `expectedVersion + 1`。
3. 不存在时不能update。
4. 重复创建不能覆盖已有记录。

### 幂等保存

当前幂等判断不能只比较checkpointId、version和status。

必须比较完整业务内容，至少包括：

- 稳定身份
- status
- version
- nodeName
- pendingApproval
- stateData
- createdAt
- updatedAt

内容不同必须返回明确冲突，不能静默当成相同保存。

### 并发

当前实例级全局writeLock会串行所有runId，必须移除。

要求：

1. 同一runId的save/update/delete保持原子。
2. 不同runId可以并发。
3. 可使用按runId分段锁、不可变推进状态或ConcurrentHashMap原子能力。
4. 禁止 `runId.intern()`。
5. 禁止JVM全局锁。
6. 禁止ThreadLocal。
7. 主索引和threadId索引必须保持一致。
8. 不得产生无限增长且无法清理的锁对象。

### 条件删除

`deleteIfVersionMatches`必须同时匹配：

- runId
- checkpointId
- version

不得误删同一runId后续再次挂起的新版本。

## 四、修复深不可变快照

检查：

- AgentCheckpoint.stateData
- InterruptPayload.safeArguments
- PendingApprovalDetail.safeArguments
- Store返回值

要求：

1. Map、List、Set必须递归复制。
2. Set中的嵌套集合也必须递归处理。
3. `Collections.unmodifiableMap(original)`不算防御性复制。
4. 不得保留调用方传入的可变Map引用。
5. safeArguments只能使用已脱敏值。
6. 不得查询原始ToolCall重新生成safeArguments。
7. Map key类型不符合预期时必须明确拒绝，不能静默丢弃字段。
8. 对外返回不可变快照。

## 五、修复ReactResumeValidator

当前问题：

- 没有重新计算operationFingerprint。
- 对pendingToolCalls和cursor直接强转。
- RunContext缺失或类型错误时没有拒绝。
- 对身份冲突的State存在“修正后继续”的行为。

恢复抢占前必须完整校验：

1. executionType=REACT_AGENT。
2. status=SUSPENDED。
3. PendingApproval存在。
4. 状态为APPROVED或REJECTED。
5. ApprovalDecision存在。
6. decidedBy非空。
7. nodeName严格为execute_tools。
8. stateData存在且结构合法。
9. pendingToolCalls必须是合法的ToolCall列表。
10. cursor必须是合法整数且处于范围内。
11. cursor对应ToolCall ID严格等于payload.toolCallId。
12. toolName严格等于operationName。
13. TOOL审批的operationFingerprint必须非空。
14. 使用现有 `ToolOperationFingerprint`，根据原runId和cursor ToolCall重新计算并完全匹配。
15. payload中的runId、threadId、userId、agentName、nodeName必须与Checkpoint顶层一致。
16. RunContext必须存在且类型正确。
17. RunContext中的runId、threadId、userId必须与Checkpoint顶层一致。
18. 不得信任或修正冲突的身份字段后继续恢复。

所有类型错误、缺失字段和不匹配必须抛明确的：

- INVALID_APPROVAL_DECISION
- CHECKPOINT_NOT_RESUMABLE

不得抛出ClassCastException或统一变成INTERNAL_ERROR。

Validator需要使用指纹计算能力时，通过明确构造器依赖注入，并同步修复Spring装配；不得创建静态全局访问或反射。

## 六、修复ToolExecutionNode审批绑定

`DefaultReactToolExecutionNode`只有在当前cursor ToolCall与审批完全绑定时才能注入审批。

必须验证：

- runId
- threadId
- userId
- toolCallId
- operationName/toolName
- operationFingerprint

不匹配时：

1. 立即抛结构化异常。
2. 不得把approval留空后重新产生一个审批。
3. 不得执行工具。
4. 不得覆盖旧审批。
5. 不得进入新的挂起状态掩盖错误。

`ToolApprovalInterceptor`必须再次执行相同安全校验：

1. TOOL审批指纹为空属于非法审批。
2. 不允许通过 `fingerprint != null` 条件跳过校验。
3. APPROVED只有完整绑定成功后才能执行Terminal。
4. REJECTED完整绑定成功后返回APPROVAL_REJECTED ToolResult。
5. 拒绝结果继续进入Observe。

## 七、修复State重建

`ReactCheckpointStateMapper`不得把冲突的RunContext“修正后继续”。

要求：

1. Validator负责在抢占前拒绝身份冲突。
2. Mapper只根据已经验证的Checkpoint重建State。
3. RunContext缺失或类型错误时抛结构化异常。
4. 保留原：
    - runId
    - threadId
    - userId
    - roles
    - permissions
    - messages
    - pendingToolCalls
    - executionCursor
    - executionBuffer
    - iteration
    - maxIterations
    - checkpointId
5. pendingApproval替换为最新已决定版本。
6. currentStatus设置为运行状态。
7. finalResult、旧failure字段和旧stopReason按恢复语义清理。
8. 不修改Store中的stateData。
9. 不生成新Session、runId或threadId。
10. 不重新进入产生当前ToolCall的Reason。

## 八、修复ReactResumeEngine失败生命周期

当前问题：

- Checkpoint先抢占为RESUMING。
- State重建位于try/catch之外。
- 最终结果读取也没有完整纳入失败处理。
- 这些步骤失败会让Checkpoint永久停留在RESUMING。

修复要求：

1. 模型能力检查在抢占前完成。
2. 完整Validator校验在抢占前完成。
3. 抢占成功后，下列步骤必须全部进入统一失败生命周期：
    - State重建
    - 恢复图编译
    - 图调用
    - 最终State读取
    - AgentResult读取和类型检查
4. 任一步骤发生框架异常：
    - 条件更新RESUMING→FAILED
    - 保存安全errorCode
    - 不保存异常对象或堆栈
5. 普通ToolResult失败经模型正常处理，不得标记Checkpoint FAILED。
6. 再次挂起时不得标记FAILED或删除。
7. finalResult缺失必须标记FAILED。
8. 不得留下无法再次处理的RESUMING Checkpoint。

## 九、恢复图并发安全

当前实现缓存并复用一个 `CompiledGraph`，但没有证明LangGraph4j 1.8.20支持多请求并发复用。

要求：

1. 检查当前实际版本和源码/API语义。
2. 如果无法确认CompiledGraph线程安全，每次resume独立编译恢复图。
3. 不得在共享图实例中保存请求State。
4. 不得用全局锁串行图执行。
5. 不得跨请求复用可变ReactAgentState。
6. 正常图和恢复图继续复用节点定义，但不共享请求状态。

## 十、Checkpoint生命周期服务

按Phase6 Batch3要求集中生命周期语义，优先实现或恢复：

`ReactCheckpointLifecycleService`

至少处理：

- complete
- fail
- removeTerminal

要求：

1. 完成：
    - RESUMING→COMPLETED条件更新
    - version+1
    - 再按checkpointId/version条件删除
2. 失败：
    - RESUMING→FAILED条件更新
    - 保存安全errorCode
    - 默认保留FAILED Checkpoint
3. 再次挂起：
    - 由ReactCheckpointService更新为SUSPENDED
    - 生命周期服务不得删除
4. 版本冲突不能只记录日志后返回成功。
5. 生命周期版本冲突必须返回CHECKPOINT_CONFLICT。
6. 普通清理日志失败不能伪造工具执行失败。
7. 不得静默吞掉状态更新失败。
8. ReactResumeEngine不要继续内嵌重复生命周期实现。

## 十一、修复审批用户隔离和HTTP错误

当前问题：

- 其他用户决定审批时应用层返回PERMISSION_DENIED。
- HitlApprovalController的局部错误映射没有处理它。
- 最终可能返回500并暴露“用户不拥有此Checkpoint”。

要求：

1. 查询、决定和恢复其他用户runId统一使用安全NOT_FOUND语义。
2. 不得泄漏runId是否存在、属于谁或当前状态。
3. 不改变Checkpoint版本。
4. 不执行工具或模型。
5. Controller不得把内部异常message直接返回客户端。
6. 使用统一安全错误消息。
7. 其他用户详情和决定均返回404或项目已有安全等价状态。
8. 不得根据ADMIN角色跨用户访问。

## 十二、统一Controller和错误映射

`HitlApprovalController`必须保持薄层：

1. 只读取已验证UserSession。
2. 构造命令。
3. 调用Application Service。
4. 映射成功DTO。
5. 不直接维护一套 `mapErrorToHttpStatus`。
6. 不重复构造错误JSON。
7. AgentFrameworkException统一交给GlobalExceptionHandler。
8. 无模型时通过结构化MODEL_NOT_AVAILABLE异常进入统一503。
9. 不直接访问CheckpointStore、ResumeEngine或CompiledGraph。

GlobalExceptionHandler必须补充：

### 400

- HttpMessageNotReadableException
- DTO构造过程包装的非法参数
- IllegalArgumentException
- INVALID_ARGUMENT
- INVALID_APPROVAL_DECISION
- CHECKPOINT_NOT_RESUMABLE

非法action、空approvalId和超长comment必须返回400，不能返回500。

### 404

- CHECKPOINT_NOT_FOUND
- APPROVAL_NOT_FOUND
- 跨用户安全隐藏结果

### 409

- APPROVAL_ALREADY_DECIDED
- CHECKPOINT_CONFLICT
- RUN_ALREADY_RESUMING

### 503

- MODEL_NOT_AVAILABLE
- MODEL_INVOCATION_FAILED

遵循Batch4提示词，将MODEL_INVOCATION_FAILED映射为503，不再由HITL Controller单独映射。

响应不得包含：

- 堆栈
- 内部类名
- 文件路径
- stateData
- 原始异常message中的敏感细节

## 十三、恢复响应必须保留threadId

当前问题：

Controller调用 `ApprovalResumeResponse.from(runId, "", result)`，导致threadId永远为空。

修复要求：

1. 应用层恢复结果必须安全表达：
    - runId
    - threadId
    - agentName
    - status
    - success
    - content
    - errorCode
    - approvalId
    - operationName
    - riskLevel
2. 优先修复或复用Phase6要求的 `ReactResumeResult`。
3. 不得让Controller自行查询Checkpoint补threadId。
4. 不得传空字符串或生成新threadId。
5. 使用原Checkpoint中的runId和threadId。
6. 再次挂起返回新的approvalId。
7. 不得返回Session ID、stateData、ToolTrace或完整消息。

## 十四、日志修复

1. 不记录完整approvalId，使用安全短标识。
2. 不记录sessionId。
3. 不记录原始arguments、fingerprint或stateData。
4. 不记录完整权限。
5. Controller不得记录审批Body。
6. 冲突记录WARN。
7. 真实框架失败记录ERROR并保留服务端cause。
8. 正常拒绝和SUSPENDED不是ERROR。

## 十五、修复前端提交后刷新

当前问题：

- `executeConfirm`先设置 `isSubmitting=true`。
- 成功、再次挂起和409分支随后调用 `loadPending()`。
- `loadPending()`在isSubmitting=true时直接return。
- 因而所有提交后的刷新实际没有执行。

修复要求：

1. 轮询触发的查询在提交期间暂停。
2. 提交完成后的显式刷新必须能够执行。
3. 可以区分：
    - poll刷新
    - 用户手动刷新
    - 提交后强制刷新
4. 或先解除提交状态，再统一刷新。
5. 不得产生并发写请求。
6. 再次挂起后立即显示新审批。
7. 409后立即：
    - 关闭旧详情
    - 标记旧审批失效
    - 重新查询列表
8. 不得等待下一次8秒轮询才更新。
9. 旧approvalId不得继续操作。
10. 修复当前“先替换列表、再尝试标记已移除旧项”的无效逻辑。

## 十六、修复旧等待状态

当前问题：

恢复成功后只设置latestResumeResult，没有清除或更新demoSuspended，页面会同时显示：

- 运行等待审批
- 运行已完成

修复要求：

1. 初次SUSPENDED时显示等待审批。
2. 同一runId恢复COMPLETED后：
    - 清除旧demoSuspended，或
    - 将其更新为最终状态
3. FAILED同样更新当前状态。
4. 再次SUSPENDED时：
    - 用新approvalId替换旧审批
    - 显示“运行再次等待审批”
5. latestResumeResult是该runId恢复后的权威状态。
6. 已完成审批从列表消失后关闭旧详情。
7. 页面不得同时显示两个互相冲突的当前状态。

## 十七、修复拒绝结果展示

当前页面只有在：

`status=COMPLETED && errorCode=APPROVAL_REJECTED`

时才显示“危险工具未执行”。

但拒绝结果经过Observe和Reason后，最终AgentResult可能COMPLETED且errorCode为空。

修复要求：

1. 页面记录本次提交动作是APPROVE还是REJECT。
2. REJECT请求正常完成后明确显示：
   `危险工具未执行，Agent已收到拒绝结果。`
3. 不依赖最终errorCode仍为APPROVAL_REJECTED。
4. 不得显示成“工具已经执行但失败”。
5. 仍展示模型最终解释。
6. APPROVE不得显示拒绝提示。

## 十八、修复VISITOR结果展示

VISITOR没有危险工具ACL时，Agent可能正常完成并在content中解释权限拒绝，而不是返回FAILED。

前端不能把所有非SUSPENDED、非FAILED结果显示为“模型不可用”。

要求：

1. COMPLETED结果正常展示后端content。
2. TOOL_ACCESS_DENIED或PERMISSION_DENIED显示权限不足。
3. 只有503或MODEL_NOT_AVAILABLE才显示模型不可用。
4. VISITOR权限拒绝不得创建PendingApproval或Checkpoint。
5. 不根据角色名硬编码结果。

## 十九、前端错误和重复提交

1. `loadPending`失败时保留已有列表并显示非破坏性错误提示。
2. 不得静默吞掉所有错误。
3. 401继续由统一HTTP客户端处理并跳转登录。
4. 409显示明确冲突提示。
5. `executeConfirm`入口增加显式重复执行保护，不能只依赖按钮disabled。
6. `triggerDangerousOp`同样增加入口保护。
7. 写请求不得自动重试。
8. 轮询timer和visibility监听必须在卸载时清理。
9. 同一组件最多一个轮询任务。
10. 页面隐藏时停止轮询，恢复可见时只创建一个timer。

## 二十、必须验证

不得新增测试脚本或隐藏Controller。使用现有接口和页面完成可以执行的验证。

### 后端错误语义

1. 缺少Session访问HITL：401。
2. 非法action：400。
3. 空approvalId：400。
4. 超长comment：400。
5. 不存在runId：404。
6. visitor访问admin的runId：安全404，不泄漏信息。
7. 相反决定：409。
8. 重复恢复：409，不返回500。
9. 模型不可用：503。

### 批准

1. 初次调用SUSPENDED。
2. 查询列表出现PENDING。
3. APPROVE后危险工具真实执行一次。
4. 返回原runId和threadId。
5. 最终COMPLETED。
6. Checkpoint条件清理。
7. 页面不再显示旧等待状态。

### 拒绝

1. REJECT后Terminal不执行。
2. 生成APPROVAL_REJECTED ToolResult。
3. 经Observe回灌模型。
4. 最终页面显示危险工具未执行。
5. demo记录仍存在。
6. Checkpoint完成并清理。

### 多危险工具

1. 第一个审批后恢复。
2. 第二个危险工具产生新approvalId。
3. checkpointId保持不变。
4. version正确增加。
5. 第一个工具不重复执行。
6. 页面立即显示第二个审批。
7. 旧approvalId不能再次使用。

### 并发

1. 同一runId只有一个resume抢占成功。
2. 不同runId可以并发。
3. 危险工具最多执行一次。
4. 生命周期冲突返回明确409。

### 前端

1. 409后立即刷新，不等待轮询。
2. 恢复完成后旧等待状态消失。
3. 轮询失败显示提示。
4. VISITOR权限拒绝不显示为模型不可用。
5. Session失效后停止轮询并跳转登录。

## 二十一、构建

遵守AGENTS.md中公司电脑的构建路径要求，不扫描磁盘寻找其他Java或Maven。

全部修改完成后只执行规定的最终构建：

1. 后端clean compile一次。
2. bootstrap聚合package一次。
3. 前端生产build一次。

不要在修改过程中反复执行最终编译命令。先完成静态检查，再执行最终构建。

构建失败必须报告真实错误，不得伪造成功。

## 二十二、最终输出

最终只报告：

1. 新增和修改文件。
2. 再次挂起checkpointId、approvalId和version处理。
3. Store稳定身份、幂等和并发实现。
4. 深不可变快照处理。
5. ReactResumeValidator完整校验顺序。
6. 指纹重新计算和ToolExecution审批绑定。
7. State重建规则。
8. ResumeEngine失败生命周期。
9. CompiledGraph并发处理选择及依据。
10. Checkpoint完成、失败和条件删除策略。
11. 跨用户安全NOT_FOUND处理。
12. DTO校验和HTTP状态映射。
13. 恢复响应runId/threadId来源。
14. 前端提交后刷新、409和轮询处理。
15. 旧等待状态同步方式。
16. 拒绝和VISITOR结果展示。
17. 后端编译和打包结果。
18. 前端生产构建结果。
19. 实际完成的接口和浏览器验证。
20. 未完成验证及准确原因。
21. 发现但未处理的非本批问题。

不要把计划、静态推测或未执行的验证描述成已完成结果。