你现在需要修复当前项目 Phase6 Batch4 的实现。

必须先完整阅读并遵守根目录 `AGENTS.md`，再完整阅读 `.kscc-prompts/phase6-batch4.md`，然后检查仓库真实代码、依赖版本、后端接口和前端结构。

Phase6 Batch1、Batch2 已完成；Batch3 的审批决策、Checkpoint抢占、状态重建、批准/拒绝恢复和序列化问题应当已经修复。本次不要重做 Batch1～Batch3，只修复 Batch4 的查询服务、HTTP接口、Session保护、错误映射、HITL前端和整体联调问题。

必须基于当前代码独立分析和实现，不要寻找、复制或依赖其他 Agent 生成的补丁、stash、commit、reflog或工作区残留内容。

## 一、本次范围

本次只处理：

1. 当前用户待审批查询模型和查询服务。
2. HITL审批查询、详情、决定并恢复HTTP接口。
3. `/api/hitl/**` 的Session认证保护。
4. 审批相关错误码到HTTP状态的统一映射。
5. Vue3 HITL审批中心。
6. 危险操作演示区域。
7. 轮询、重复点击、409冲突和401失效处理。
8. 恢复结果及再次挂起结果展示。
9. Phase6批准、拒绝和权限隔离整体联调。
10. Batch4联调所必需的最小后端兼容修复。

禁止：

- 重写Batch3的Checkpoint和React恢复引擎。
- 新增裸resume HTTP接口。
- Controller直接访问CheckpointStore。
- 前端调用 `/api/dev/**`。
- 实现Supervisor审批或恢复。
- 实现管理员跨用户审批。
- 引入数据库、Redis、SSE、WebSocket、分布式锁或审批历史。
- 新增第二套认证、HTTP客户端、错误响应或前端工程。
- 新增测试脚本、README、验收文档。
- 修改AGENTS.md或 `.kscc-prompts`。
- 执行git commit、push、merge、reset。
- 无关重构认证、ACL、ReAct、Supervisor和用户管理。

如果Batch3仍有阻断Batch4编译或联调的问题，只允许做最小修复，并在最终结果中单独列明。不得以Batch4修复为由重写Batch3。

## 二、修改前检查

必须先检查真实代码：

1. 根目录及相关模块 `pom.xml`。
2. `CheckpointStore` 最终接口和内存实现。
3. `AgentCheckpoint`、`PendingApproval`、`InterruptPayload`。
4. `ApprovalDecisionService`。
5. `ApprovalResumeApplicationService`。
6. `ReactResumeResult`、`AgentResult`及挂起结果字段。
7. `PendingApprovalQueryService`及查询模型是否已经存在。
8. `SessionAuthenticationInterceptor`。
9. UserSession存入请求attribute的真实名称和读取方式。
10. 现有Controller、DTO和统一异常响应风格。
11. `GlobalExceptionHandler`或现有异常映射。
12. MVC拦截路径、排除路径和CORS配置。
13. `/api/agent/invoke`真实请求与响应。
14. `/api/framework/agents`真实返回结构。
15. `approval_demo_agent`和演示工具的Sample开关。
16. `agent-web`目录结构、package.json和锁文件。
17. 统一HTTP客户端、Session存储和401/403处理。
18. Vue Router路由守卫和主布局导航。
19. 当前是否同时存在重复的 `approval.js`、`hitl.js`或重复HITL页面。
20. Vite代理、前后端端口及构建配置。

不得根据提示词猜测构造器、字段或响应格式，必须以真实代码为准。

## 三、查询模型

在 `agent-application` 中修复或复用不可变模型：

### PendingApprovalSummary

至少安全表达：

- runId
- threadId
- agentName
- approvalId
- operationType
- operationName
- riskLevel
- reason
- requestedAt
- status

### PendingApprovalDetail

至少表达：

- Summary全部字段
- nodeName
- safeArguments
- createdAt
- updatedAt
- checkpointVersion

必须保证：

1. 使用不可变模型和不可变集合快照。
2. safeArguments只能来自Checkpoint中已经保存的脱敏值。
3. 查询时不得重新读取原始ToolCall参数生成safeArguments。
4. 不返回stateData。
5. 不返回RunContext。
6. 不返回sessionId。
7. 不返回原始工具参数。
8. 不返回operationFingerprint。
9. 不返回权限集合、密码、Token、API Key或完整消息。
10. status复用现有ApprovalStatus，不创建第二套状态枚举。

## 四、PendingApprovalQueryService

修复或实现纯Java、无状态、线程安全的：

`PendingApprovalQueryService`

至少提供等价能力：

```java
Collection<PendingApprovalSummary> findPending(UserSession operator);
PendingApprovalDetail getPending(UserSession operator, String runId);
```
规则：
operator不能为空且必须来自已验证Session。
通过operator.userId查询当前用户Checkpoint。
列表只返回：CheckpointStatus.SUSPENDED
ApprovalStatus.PENDING

不返回：RESUMING
COMPLETED
FAILED
APPROVED
REJECTED

按requestedAt升序排列。
getPending 通过runId加载后必须校验Checkpoint归属。
不存在和属于其他用户必须使用相同的安全NOT_FOUND语义。
不得泄漏其他用户是否拥有该runId。
必须校验Checkpoint和PendingApproval结构。
不调用模型、工具或恢复引擎。
不修改或删除Checkpoint。
不依赖Spring Web、HttpServletRequest或ThreadLocal。
本批不允许ADMIN跨用户查询。
如果CheckpointStore缺少按用户查询能力，只允许最小增加安全查询方法；不得把查询业务塞进Store。
五、请求DTO
在 agent-api 中修复或复用：
ApprovalDecisionRequest
字段只能包括：
approvalId
action
comment
要求：
action使用现有ApprovalAction，只能是APPROVE或REJECT。
runId来自URL路径。
Body不得包含：userId
decidedBy
decidedAt
checkpointVersion
operationFingerprint
permissions
roles
RunContext

comment允许为空。
comment必须trim并限制合理长度。
空approvalId和非法action返回400。
不得将未知字符串、空字符串或解析失败静默映射成APPROVE。
使用项目现有DTO和校验风格，不要为此引入新的大型依赖。
检查是否同时存在 ApprovalDecideRequest 和 ApprovalDecisionRequest 等职责重复DTO。如果确属同一接口的重复实现，应统一为一套，并安全修复所有调用点，不保留V2或兼容壳。
六、响应DTO
修复或创建专用DTO：
PendingApprovalSummaryResponse
PendingApprovalDetailResponse
ApprovalResumeResponse
ApprovalResumeResponse至少安全表达：
runId
threadId
agentName
status
success
content
errorCode
approvalId
operationName
riskLevel
安全白名单metadata
规则：
恢复完成返回最终Agent结果。
恢复后再次挂起：status=SUSPENDED
返回新的approvalId
返回新的operationName和riskLevel

拒绝后模型正常生成最终解释时，可以是：status=COMPLETED
success=true
这表示Agent流程完成，不表示危险工具执行成功。

不得返回：Session ID
Checkpoint
stateData
RunContext
完整消息历史
ToolTrace
Graph State
原始工具参数
Java异常对象

不为挂起状态创建第二套枚举。
metadata只允许白名单字段，不直接透传任意Map。
保持原runId和threadId，不生成新的顶层身份。
七、HITL Controller
修复：
HitlApprovalController
统一路径必须是：
/api/hitl
精确提供：
GET /api/hitl/approvals
GET /api/hitl/approvals/{runId}
POST /api/hitl/approvals/{runId}/decide-and-resume
查询列表
从现有认证请求attribute读取已验证UserSession。
调用 PendingApprovalQueryService.findPending。
返回当前用户待审批列表。
不读取客户端userId。
不访问CheckpointStore。
查询详情
读取已验证UserSession。
调用 PendingApprovalQueryService.getPending。
返回安全详情。
其他用户runId按安全404处理。
不返回stateData、原始参数或指纹。
决定并恢复
请求示例：
{
"approvalId": "apr-...",
"action": "APPROVE",
"comment": "确认执行"
}
流程：
从Session读取operator。
runId从路径获取。
构造ApprovalDecisionCommand。
调用 ApprovalResumeApplicationService.decideAndResume。
映射为ApprovalResumeResponse。
批准和拒绝共用同一接口。
再次挂起正常返回SUSPENDED和新approvalId。
重复提交交给后端幂等和version控制。
禁止：
Controller直接访问CheckpointStore。
Controller自行调用模型、工具或CompiledGraph。
Controller分别拼装DecisionService和ResumeEngine。
Controller自行构造ReactAgentState。
提供公开裸resume接口。
根据用户名或ADMIN角色绕过审批。
Controller必须保持薄层。
八、Session认证保护
在现有MVC拦截配置中保护：
/api/hitl/**
必须保证：
缺少 X-Session-Id 返回401。
伪造、失效、撤销或过期Session返回401。
无效请求不进入Controller和Application Service。
OPTIONS继续按现有CORS策略处理。
认证拦截器只负责Session认证，不查询Checkpoint。
Checkpoint归属由Application Service校验。
不新增第二套Filter或认证机制。
不使用ThreadLocal。
sessionId不能放入URL、query或Body。
不在日志中记录sessionId。
九、错误与HTTP映射
复用现有统一异常响应，最小修复映射：
HTTP 400
INVALID_ARGUMENT
INVALID_APPROVAL_DECISION
CHECKPOINT_NOT_RESUMABLE
HTTP 401
SESSION_INVALID
SESSION_EXPIRED
HTTP 404
CHECKPOINT_NOT_FOUND
APPROVAL_NOT_FOUND
其他用户runId的安全隐藏结果
HTTP 409
APPROVAL_ALREADY_DECIDED
CHECKPOINT_CONFLICT
RUN_ALREADY_RESUMING
HTTP 500
RESUME_FAILED
INTERNAL_ERROR
HTTP 503
MODEL_NOT_AVAILABLE
MODEL_INVOCATION_FAILED
要求：
审批拒绝不是HTTP错误。
REJECT可以返回200及恢复后的AgentResult。
再次SUSPENDED不是HTTP错误。
重复相同决定可幂等返回200。
相反决定和版本冲突返回409。
重复恢复不能错误映射为500。
不存在和其他用户数据使用安全的404语义。
响应不暴露堆栈、内部类名、文件路径、stateData或异常对象。
不创建第二套错误响应。
保留服务端cause用于日志，但不得返回客户端。
十、无模型条件装配
必须检查Spring配置：
PendingApprovalQueryService和查询Controller不依赖模型能力。
无模型配置时：应用可以启动。
登录、Session、Checkpoint Store和HITL查询可以装配。
待审批列表可以安全查询。

实际decide-and-resume需要模型继续Reason时，返回明确503模型不可用错误。
模型不可用时不得把Checkpoint错误抢占后永久留在RESUMING。
不创建Fake ModelClient。
runtime/application保持纯Java。
Bean通过infrastructure中的Configuration统一装配。
使用ConditionalOnMissingBean支持替换，避免循环依赖。
十一、前端API封装
在现有 agent-web/src/api 中保留一套HITL API封装，例如：
hitl.js
至少提供：
listPendingApprovals()
getPendingApproval(runId)
decideAndResume(runId, payload)
要求：
复用现有统一HTTP客户端。
不直接使用裸fetch绕过401/403处理。
不手工读取或拼接sessionId。
不把sessionId放进URL或Body。
不调用 /api/dev/**。
不自动重试decide-and-resume写请求。
不在console打印审批Body或敏感响应。
409转换为现有统一错误，供页面提示刷新。
401继续使用统一逻辑清理Session并跳转登录。
网络错误保持现有统一错误结构。
如果同时存在 approval.js 和 hitl.js 且职责重复：
检查全部真实引用。
统一成现有命名和路由匹配的一套。
删除无引用的重复封装。
不保留只转发的兼容文件。
不影响其他已有API模块。
十二、HITL路由和导航
增加或修复：
/hitl
规则：
需要登录。
不要求ADMIN角色。
不要求管理权限。
ADMIN和VISITOR均可进入自己的审批中心。
未登录跳转 /login。
Session失效交给统一401逻辑。
侧边导航显示“人机审批”。
不得根据角色名决定页面是否可访问。
不得在每次导航创建新的轮询任务。
如实现待审批数量角标，失败不能阻断主布局。
十三、HitlApprovalView
修复或实现：
HitlApprovalView.vue
页面至少包含：
危险操作演示。
待审批列表。
审批安全详情。
审批意见输入框。
批准和拒绝按钮。
二次确认面板或现有Modal。
手动刷新。
空列表状态。
加载和提交状态。
错误提示。
最近一次恢复结果。
列表至少显示：
operationName
agentName
riskLevel
reason
requestedAt
runId短标识
status
详情至少显示：
operationType
operationName
riskLevel
nodeName
safeArguments
reason
requestedAt
安全要求：
不显示sessionId。
不显示Checkpoint stateData。
不显示operationFingerprint。
不显示原始工具参数。
不显示完整权限集合。
safeArguments结构化只读展示。
***等脱敏值保持遮蔽。
页面不得尝试恢复被遮蔽数据。
不允许前端自行修改审批状态。
只能以后端返回结果为准。
十四、审批交互
点击批准：
当前详情必须仍是PENDING。
显示安全操作名和safeArguments。
明确提示：
批准后将实际执行该危险操作。
提交 action=APPROVE。
调用decideAndResume。
提交期间禁用批准和拒绝。
禁止重复点击。
成功后展示恢复结果并刷新列表。
点击拒绝：
当前详情必须仍是PENDING。
允许填写拒绝原因。
明确提示：
拒绝后工具不会执行，Agent将根据拒绝结果继续。
提交 action=REJECT。
成功后展示Agent对拒绝结果的最终回答。
刷新待审批列表。
共同要求：
不把浏览器原生confirm作为唯一交互。
优先复用现有Modal；无组件时实现简单确认面板。
审批意见提交成功后清空。
失败后恢复按钮可用状态。
不在前端乐观设置APPROVED或REJECTED。
409后：提示“审批状态已发生变化，请刷新”。
关闭或失效旧详情。
重新获取列表。
不自动重试写请求。

两个标签并发操作时，最终安全边界必须依赖后端version，不依赖按钮禁用。
十五、轮询
不实现SSE，采用页面内轮询：
进入页面立即查询。
提供手动刷新。
页面可见时每5～10秒查询一次。
频率不得高于每5秒一次。
页面隐藏时停止。
离开路由或组件卸载时停止。
审批/恢复请求执行期间暂停。
一个组件最多存在一个timer。
不得因为响应式watch重复创建timer。
轮询失败保留现有内容并显示非破坏性提示。
401交给统一HTTP客户端。
不在全局布局长期轮询。
必须检查：
visibilitychange监听是否在卸载时清除。
interval/timeout是否清除。
重新可见时是否只恢复一个轮询任务。
十六、危险操作演示
优先放在HITL页面顶部，不创建额外页面。
要求：
通过正式 /api/framework/agents 或现有等价查询确认 approval_demo_agent 已注册。
Agent不存在或Sample关闭时显示明确不可用。
只允许选择：demo-1
demo-2

删除原因限制合理长度。
发起请求使用正式：
POST /api/agent/invoke
agentName固定为：
approval_demo_agent
message明确要求模型使用 delete_demo_record，并依据真实工具结果回答。
不调用任何 /api/dev/**。
发起期间禁用重复提交。
不在前端生成approvalId或Checkpoint。
返回SUSPENDED时显示：runId短标识
approvalId短标识
operationName
riskLevel
等待审批提示

随后刷新待审批列表。
不得根据模型文本判断工具是否执行，必须以后端状态和结果为准。
模型不可用时显示安全503提示。
Sample账号密码不得硬编码；测试所需密码通过AGENTS.md规定的环境配置提供。
建议消息：
请必须使用delete_demo_record删除记录{recordId}，原因为：{reason}。必须依据真实工具结果回答，不得假设工具已经执行。
十七、恢复结果展示与状态同步
恢复结果区域至少展示：
status
success
content
errorCode
runId
threadId
operationName
再次挂起时的新approvalId
展示规则：
COMPLETED显示“运行已完成”。
SUSPENDED显示“运行再次等待审批”。
FAILED显示安全失败原因。
拒绝后Agent正常解释时可以显示“运行已完成”。
拒绝结果必须明确显示：
危险工具未执行，Agent已收到拒绝结果。
不得把 APPROVAL_REJECTED 描述为危险工具已经执行但失败。
metadata只展示安全白名单字段。
不展示完整JSON、消息历史或ToolTrace。
必须修复初次调用状态与恢复结果不同步的问题：
初次调用SUSPENDED时可以显示“运行等待审批”。
批准或拒绝恢复完成后，旧的“运行等待审批”不得继续作为当前状态显示。
恢复结果必须成为该runId的最新权威状态。
可以清除旧挂起横幅，或将其更新为COMPLETED/FAILED。
再次挂起时更新为新的approvalId和新操作。
不得出现页面同时显示“运行等待审批”和“运行已完成”而无法区分当前状态。
已完成审批从列表消失后，旧详情必须关闭或标记失效。
十八、权限关系
保持后端实际权限语义：
ADMIN
具有危险工具ACL时，调用进入SUSPENDED。
不能自动绕过人工审批。
批准后才真实执行。
拒绝后不执行。
VISITOR
如果没有 tool:delete_demo_record:invoke：
在ACL阶段拒绝。
不创建PendingApproval。
不创建Checkpoint。
审批列表为空。
Agent根据PERMISSION_DENIED回答。
要求：
前端不能根据角色名硬编码结果。
页面访问权不等于危险工具权限。
后端ACL和Session权限快照始终是最终安全边界。
dev用户或通配工具权限也不能绕过人工审批。
前端不得调用dev API。
十九、日志
允许记录：
operatorUserId
runId
threadId
approvalId安全短标识
action
operationName
checkpointVersion
resumeStatus
finalStatus
禁止：
sessionId
X-Session-Id
原始arguments
operationFingerprint
stateData
完整UserSession
权限全集
密码、Token、API Key
完整Prompt和messages
正常SUSPENDED不是ERROR；版本冲突可记录WARN；真实框架失败才记录ERROR并保留服务端cause。
二十、必须验证的后端接口
使用真实Session和现有正式接口验证，不得新增隐藏Controller。
未认证
GET /api/hitl/approvals
预期：
HTTP 401
不进入查询服务
空列表
GET /api/hitl/approvals
X-Session-Id: 有效Session
预期：
HTTP 200
返回 []
不返回stateData
查询不存在或其他用户runId
预期：
HTTP 404或项目现有安全等价响应
不泄漏归属信息
非法action
预期：
HTTP 400
不作出审批决定
不改变Checkpoint
批准并恢复
预期：
初次Agent调用返回SUSPENDED。
查询列表出现PENDING审批。
详情只包含safeArguments。
POST APPROVE返回最终COMPLETED或新SUSPENDED。
工具真实执行一次。
完成后列表不再显示旧审批。
拒绝并恢复
预期：
POST REJECT正常返回。
工具不执行。
Agent收到拒绝观察结果。
返回最终说明。
待审批记录消失。
重复/冲突操作
预期：
相同决定按Batch3幂等语义处理。
相反决定或并发抢占返回409。
不返回500。
工具最多执行一次。
二十一、必须验证的浏览器场景
在真实环境具备Sample用户、模型和approval_demo_agent时验证：
ADMIN触发demo-1审批，批准后demo-1删除。
ADMIN触发demo-2审批，拒绝后demo-2仍存在。
两个标签操作同一审批，后操作得到409并刷新。
VISITOR发起危险操作时ACL拒绝，不产生审批。
Session撤销后，下一次轮询收到401、清理Session、停止轮询并跳转登录。
恢复完成后页面不再保留误导性的“运行等待审批”当前状态。
再次挂起时页面展示新approvalId，旧审批不能操作。
如果模型不能稳定生成两个危险ToolCall，可以只完成代码级检查，但必须明确写出未完成真实验证，不得伪造。
二十二、构建要求
遵守AGENTS.md中的Windows路径和“全部修改完成后只执行一次编译”要求，不得扫描磁盘或注册表寻找Java/Maven。
所有修改完成后：
后端规定的clean compile命令执行一次。
bootstrap package命令执行一次。
前端根据现有锁文件执行生产构建一次。
本次不需要重复执行编译命令来试错；必须先完成静态检查再进行最终构建。
不删除或重建锁文件。
不同时创建多种锁文件。
不引入TypeScript。
没有新增依赖时不要无意义执行install。
构建失败必须报告准确错误；不要伪造成功。
二十三、最终检查
确认：
Controller只依赖Application Service和查询服务。
HITL接口全部受Session保护。
用户只能查询和处理自己的审批。
DTO和响应没有敏感字段。
safeArguments没有从原始参数重新生成。
批准后工具真实执行一次。
拒绝后工具不执行但Agent继续。
重复操作由后端version保护。
401和409前端处理正确。
轮询timer和visibility监听正确清理。
恢复完成后旧等待状态已同步更新。
再次挂起能展示新审批。
前端只调用正式接口。
不存在重复HITL API封装。
无模型时非模型能力可以启动。
现有认证、ACL、普通ReAct和Supervisor没有被破坏。
git diff中没有无关修改。
二十四、最终输出
最终只报告：
新增和修改文件。
每个文件修复的Batch4问题。
PendingApprovalQueryService过滤、排序和隔离规则。
HITL接口路径及请求响应。
Controller到Application Service的调用链。
Session认证和多用户隔离方式。
错误码与HTTP状态映射。
前端API封装及统一401/409处理。
HITL页面结构和安全展示规则。
轮询启动、暂停和销毁逻辑。
危险操作演示流程。
恢复结果和旧等待状态同步方式。
ADMIN、VISITOR和Session失效的实际行为。
批准、拒绝、重复提交和再次挂起的验证结果。
后端编译和bootstrap打包结果。
前端生产构建结果。
实际完成的接口和浏览器验证。
未完成验证及准确原因。
发现但未处理的非Batch4问题。
不要把计划、推测或未执行的验证描述成已经完成。