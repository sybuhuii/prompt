你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须先完整阅读并遵守根目录`AGENTS.md`。其中已有架构、安全、前端、编译及输出规范，本提示词不重复展开。

已完成第六阶段前三批：

1.HITL审批领域模型。
2.AgentCheckpoint及内存CheckpointStore。
3.Checkpoint版本条件更新。
4.ToolRiskLevel和ToolApprovalPolicy。
5.ToolApprovalInterceptor及危险工具执行前中断。
6.ReAct SUSPEND路由。
7.ToolExecution cursor和buffer。
8.ApprovalDecisionService。
9.Checkpoint恢复抢占。
10.ReactCheckpointStateMapper。
11.ReactResumeEngine。
12.批准、拒绝、重复审批和恢复流程。
13.approval_demo_agent及演示危险工具。
14.节点重跑恢复语义。

当前只执行第六阶段第4批：

1.实现当前用户待审批查询服务。
2.实现HITL查询、审批并恢复HTTP接口。
3.使用现有Session认证保护所有HITL接口。
4.实现前端HITL审批中心。
5.实现危险工具中断、批准、拒绝和恢复演示页面。
6.实现多次挂起、重复点击和Session失效处理。
7.完成第六阶段整体联调。

本批不实现Supervisor恢复、跨用户审批、数据库Checkpoint、跨进程恢复和SSE。

# 一、执行前检查

先检查：

1.根目录`AGENTS.md`。
2.第六阶段前三批全部实际代码。
3.CheckpointStore最终接口。
4.AgentCheckpoint、PendingApproval、InterruptPayload。
5.ApprovalDecisionService。
6.ApprovalResumeApplicationService。
7.ReactResumeResult和AgentResult。
8.Checkpoint状态生命周期。
9.SessionAuthenticationInterceptor。
10.UserSession请求attribute名称。
11.现有Controller和DTO风格。
12.统一错误响应与HTTP状态映射。
13.第五阶段agent-web实际目录结构。
14.统一HTTP客户端、认证状态和路由守卫。
15.Agent调用页面和权限演示页面。
16.POST `/api/agent/invoke`实际请求和响应字段。
17.GET `/api/framework/agents`实际返回结构。
18.approval_demo_agent是否受Sample开关控制。
19.当前前后端端口和Vite代理配置。
20.当前Node包管理器及锁文件。

要求：

1.严格根据真实接口增量实现。
2.不得创建第二套认证、HTTP客户端或前端工程。
3.不得重新实现批准、拒绝和恢复核心逻辑。
4.不得让Controller直接操作CheckpointStore。
5.前序代码有阻断问题时只做最小修复。
6.不得修改无关ReAct、Supervisor、ACL和用户管理代码。
7.不得提前实现范围外能力。

# 二、HITL查询模型

在agent-application增加或复用不可变模型：

`PendingApprovalSummary`

至少包含：

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

增加或复用：

`PendingApprovalDetail`

至少包含：

- Summary全部字段
- nodeName
- safeArguments
- createdAt
- updatedAt
- checkpointVersion

要求：

1.不得返回Checkpoint stateData。
2.不得返回sessionId。
3.不得返回RunContext。
4.不得返回原始工具参数。
5.不得返回完整operationFingerprint。
6.不得返回密码、密钥、Token或完整权限集合。
7.safeArguments必须使用Checkpoint中已经脱敏的值。
8.不得在查询时重新读取原始工具参数生成safeArguments。
9.集合和Map返回不可变快照。
10.status使用现有ApprovalStatus。

# 三、PendingApprovalQueryService

在agent-application实现纯Java：

`PendingApprovalQueryService`

依赖：

- CheckpointStore
- CheckpointValidator

至少提供：

```java
Collection<PendingApprovalSummary> findPending(UserSession operator);
PendingApprovalDetail getPending(UserSession operator,String runId);
```

职责：

1.operator不能为空。
2.通过operator.userId查询当前用户待审批Checkpoint。
3.只返回CheckpointStatus.SUSPENDED。
4.只返回ApprovalStatus.PENDING。
5.不返回RESUMING、COMPLETED和FAILED。
6.不返回已APPROVED或已REJECTED但尚未恢复的数据。
7.列表按requestedAt升序排列。
8.getPending通过runId加载。
9.校验Checkpoint.userId等于operator.userId。
10.校验Checkpoint和PendingApproval结构有效。
11.不存在或不属于当前用户时使用安全的NOT_FOUND语义。
12.不得泄漏其他用户是否存在该runId。
13.不得访问HttpServletRequest。
14.不得调用模型、工具和恢复引擎。
15.不得修改Checkpoint。
16.不得删除Checkpoint。
17.不得使用ThreadLocal。
18.保持无状态和线程安全。

本批不允许ADMIN跨用户查看审批。

# 四、审批并恢复请求模型

在agent-api创建：

`ApprovalDecisionRequest`

字段：

- approvalId
- action
- comment

要求：

1.action只能为APPROVE或REJECT。
2.使用现有ApprovalAction。
3.runId来自URL路径，不在Body重复提交。
4.不得允许提交userId。
5.不得允许提交decidedBy。
6.不得允许提交decidedAt。
7.不得允许提交checkpointVersion。
8.不得允许提交operationFingerprint。
9.不得允许提交permissions或roles。
10.comment允许为空。
11.comment进行trim和长度限制。
12.非法action返回400。
13.不得使用任意字符串静默映射为APPROVE。

# 五、HITL响应DTO

在agent-api创建专用DTO：

- PendingApprovalSummaryResponse
- PendingApprovalDetailResponse
- ApprovalResumeResponse

ApprovalResumeResponse应根据现有ReactResumeResult或AgentResult映射，至少表达：

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
- metadata

要求：

1.恢复完成时返回最终Agent结果。
2.恢复后再次挂起时status=SUSPENDED，并返回新的approvalId。
3.拒绝后模型正常解释时可返回success=true。
4.模型或框架失败时使用现有结构化错误。
5.不得返回Session ID。
6.不得返回Checkpoint stateData。
7.不得返回完整消息历史。
8.不得返回ToolTrace。
9.不得返回内部Graph State。
10.不得返回原始工具参数。
11.不得返回Java异常对象。
12.不得为挂起结果创建第二套状态枚举。

# 六、HITL Controller

在agent-api实现：

`HitlApprovalController`

统一路径：

`/api/hitl`

接口一：

```text
GET /api/hitl/approvals
```

职责：

1.读取已验证UserSession。
2.调用PendingApprovalQueryService.findPending。
3.返回当前用户待审批列表。
4.不得读取客户端userId。
5.不得直接访问CheckpointStore。

接口二：

```text
GET /api/hitl/approvals/{runId}
```

职责：

1.读取已验证UserSession。
2.调用PendingApprovalQueryService.getPending。
3.返回审批安全详情。
4.不得返回stateData和原始参数。
5.其他用户runId返回安全的404或现有等价错误。

接口三：

```text
POST /api/hitl/approvals/{runId}/decide-and-resume
```

请求：

```json
{
  "approvalId":"...",
  "action":"APPROVE",
  "comment":"确认执行"
}
```

职责：

1.读取已验证UserSession。
2.构造ApprovalDecisionCommand。
3.decidedBy只能由服务端UserSession提供。
4.调用ApprovalResumeApplicationService.decideAndResume。
5.不得直接调用ApprovalDecisionService和ReactResumeEngine分别拼接业务流程，除非现有Application Service没有该方法；优先补充Application Service。
6.不得直接访问CheckpointStore。
7.不得直接执行工具。
8.不得自行构造ReactAgentState。
9.返回ApprovalResumeResponse。
10.批准和拒绝都通过同一接口。
11.恢复后再次挂起时正常返回新审批信息。
12.重复提交由后端版本和幂等语义处理。
13.不得因为客户端重复点击执行两次危险工具。

本批不提供单独的裸resume HTTP接口，防止客户端绕过人工决定直接请求恢复。

如业务服务已明确支持分离决定和恢复，可保留内部方法，但HTTP主流程统一使用decide-and-resume。

# 七、Session认证保护

调整现有MVC拦截配置，保护：

```text
/api/hitl/**
```

要求：

1.缺少X-Session-Id返回401。
2.伪造或过期Session返回401。
3.无效请求不得进入Controller。
4.认证拦截器只校验Session，不读取Checkpoint。
5.审批归属校验由Application Service完成。
6.不得新增第二套认证Filter。
7.不得使用ThreadLocal当前用户。
8.不得允许通过query参数传sessionId。
9.OPTIONS继续按现有CORS策略处理。
10.前端统一HTTP客户端自动附加X-Session-Id。

# 八、错误与HTTP状态码

检查现有全局异常映射，最小完善：

HTTP 400：

- INVALID_ARGUMENT
- INVALID_APPROVAL_DECISION
- CHECKPOINT_NOT_RESUMABLE

HTTP 401：

- SESSION_INVALID
- SESSION_EXPIRED

HTTP 404：

- CHECKPOINT_NOT_FOUND
- APPROVAL_NOT_FOUND
- 其他用户runId的安全隐藏结果

HTTP 409：

- APPROVAL_ALREADY_DECIDED
- CHECKPOINT_CONFLICT
- RUN_ALREADY_RESUMING

HTTP 500：

- RESUME_FAILED
- INTERNAL_ERROR

HTTP 503：

- MODEL_NOT_AVAILABLE
- MODEL_INVOCATION_FAILED

要求：

1.审批拒绝本身不是HTTP错误。
2.REJECT请求可以正常返回恢复后的AgentResult。
3.恢复后再次SUSPENDED不是HTTP错误。
4.重复相同决定可幂等返回200。
5.冲突决定返回409。
6.重复恢复不得返回500。
7.不得暴露Checkpoint是否属于其他用户。
8.不得返回堆栈、内部类名和stateData。
9.继续使用现有统一JSON错误响应。
10.不得建立第二套异常响应体系。

# 九、日志要求

允许记录：

- operatorUserId
- runId
- threadId
- approvalId的安全短标识
- action
- operationName
- checkpointVersion
- resumeStatus
- finalStatus

禁止记录：

- sessionId
- X-Session-Id
- 完整approvalId与完整敏感参数组合
- operationFingerprint
- 原始arguments
- stateData
- 完整UserSession
- 密码、Token和API Key
- 完整Prompt和messages

要求：

1.审批和恢复属于业务事件。
2.正常SUSPENDED不得记录为系统ERROR。
3.版本冲突可记录WARN。
4.真实框架失败才记录ERROR及服务端cause。
5.客户端响应不返回日志内部字段。

# 十、前端API封装

在现有agent-web的api目录增量实现：

`hitl.js`

至少提供：

```javascript
listPendingApprovals()
getPendingApproval(runId)
decideAndResume(runId,payload)
```

要求：

1.复用现有统一HTTP客户端。
2.不得直接使用裸fetch绕过统一401/403处理。
3.不得手工读取和拼接sessionId。
4.不得把sessionId放入URL或Body。
5.不得调用`/api/dev/**`。
6.不得打印请求Body和响应中的敏感字段。
7.不得自动重试decide-and-resume写请求。
8.网络错误使用现有统一错误结构。
9.409需显示“审批状态已发生变化，请刷新”。
10.401继续清理Session并跳转登录。

# 十一、前端HITL审批中心

在现有前端新增：

`HitlApprovalView.vue`

路由：

```text
/hitl
```

侧边导航显示：

```text
人机审批
```

所有已登录用户都可以访问自己的审批中心。

页面至少包含：

1.待审批列表。
2.审批详情。
3.批准按钮。
4.拒绝按钮。
5.审批意见输入框。
6.手动刷新按钮。
7.处理中的加载状态。
8.空列表状态。
9.错误提示。
10.最近一次恢复结果展示。

列表至少展示：

- operationName
- agentName
- riskLevel
- reason
- requestedAt
- runId短标识
- 当前状态

详情至少展示：

- operationType
- operationName
- riskLevel
- nodeName
- safeArguments
- reason
- requestedAt

要求：

1.不得显示sessionId。
2.不得显示Checkpoint stateData。
3.不得显示operationFingerprint。
4.不得显示原始工具参数。
5.不得显示完整权限集合。
6.safeArguments使用结构化只读展示。
7.敏感值`***`保持遮蔽，不允许页面恢复。
8.批准和拒绝前需要二次确认。
9.批准确认文案明确：
`批准后将实际执行该危险操作。`
10.拒绝确认文案明确：
`拒绝后工具不会执行，Agent将根据拒绝结果继续。`
11.处理期间禁用批准和拒绝按钮。
12.同一审批不得并发提交多次。
13.提交成功后刷新待审批列表。
14.恢复完成时展示最终Agent回答。
15.恢复后再次挂起时展示新审批，并刷新列表。
16.409时提示刷新，不自动重复提交。
17.401时按统一客户端逻辑退出登录。
18.不得使用角色名决定能否批准。
19.用户只能看到后端返回的自己的审批。
20.不得在前端自行修改审批状态。

# 十二、待审批刷新策略

本批不实现SSE。

页面采用：

1.进入页面立即查询。
2.提供手动刷新。
3.页面可见时每5至10秒轮询一次。
4.页面不可见或离开路由时停止轮询。
5.审批或恢复进行中暂停轮询。
6.组件卸载时清除timer。
7.不得创建多个重复轮询任务。
8.轮询失败时保留当前页面并提示。
9.401交给统一HTTP客户端处理。
10.不得高于每5秒一次。
11.不得在全局后台持续轮询。

# 十三、HITL演示区域

在审批中心页面或单独复用现有Agent调用页面增加“危险操作演示”区域。

优先放在`HitlApprovalView.vue`顶部，避免创建过多页面。

演示功能：

1.检查框架Agent列表中是否存在`approval_demo_agent`。
2.输入或选择演示recordId：
- demo-1
- demo-2
  3.输入删除原因。
  4.点击“发起危险操作”。
  5.调用正式：
  `POST /api/agent/invoke`
  6.agentName固定使用`approval_demo_agent`。
  7.message要求模型使用`delete_demo_record`删除指定记录。
  8.返回SUSPENDED时显示：
- runId短标识
- approvalId短标识
- operationName
- riskLevel
- 等待审批提示
  9.自动刷新待审批列表。
  10.不得调用开发Agent接口。

建议请求提示：

```text
请必须使用delete_demo_record删除记录{recordId}，原因为：{reason}。必须依据真实工具结果回答，不得假设工具已经执行。
```

要求：

1.演示数据必须是Sample内存数据。
2.不得允许输入文件路径、数据库表名或任意外部资源。
3.recordId限制为合理长度和安全字符。
4.不得根据模型文本假设工具已执行。
5.实际结果以后端AgentResult和审批流程为准。
6.Sample关闭或Agent不存在时显示明确不可用提示。
7.模型不可用时显示503提示。
8.发起过程中禁用重复提交。
9.不得在前端生成approvalId。
10.不得在前端创建Checkpoint。

# 十四、审批操作交互

点击批准：

1.确认当前审批仍为PENDING。
2.弹出确认区域或现有Modal。
3.显示安全操作名称和safeArguments。
4.提交action=APPROVE。
5.调用decideAndResume。
6.按钮进入loading。
7.不得允许第二次点击。
8.成功后展示恢复结果。
9.刷新待审批列表。

点击拒绝：

1.确认当前审批仍为PENDING。
2.可填写拒绝原因。
3.提交action=REJECT。
4.调用decideAndResume。
5.工具不得执行。
6.展示Agent对拒绝结果的最终回答。
7.刷新列表。

要求：

1.不得使用浏览器原始confirm作为唯一交互，优先复用现有Modal；无组件时可实现简单确认面板。
2.审批意见提交后清空。
3.不得在控制台打印审批请求。
4.请求失败后按钮恢复。
5.409后关闭旧详情并重新拉取。
6.不得客户端乐观设置APPROVED或REJECTED。
7.以后端响应为准。

# 十五、恢复结果展示

恢复结果区域至少展示：

- status
- success
- content
- errorCode
- runId
- threadId
- operationName
- 新approvalId（再次挂起时）

要求：

1.不展示完整metadata原始JSON。
2.仅展示经过白名单选择的安全metadata。
3.status=COMPLETED时显示“运行已完成”。
4.status=SUSPENDED时显示“运行再次等待审批”。
5.status=FAILED时显示安全失败原因。
6.拒绝后Agent正常解释时仍可显示成功完成。
7.不得将APPROVAL_REJECTED显示为工具已执行失败。
8.应明确显示：
`危险工具未执行，Agent已收到拒绝结果。`
9.不得显示完整消息历史和ToolTrace。

# 十六、路由和导航

在现有Vue Router中增加：

```text
/hitl
```

要求：

1.需要认证。
2.不要求ADMIN角色。
3.不要求管理权限。
4.所有用户只能管理自己的审批。
5.未登录跳转`/login`。
6.Session失效时按统一401逻辑处理。
7.侧边导航显示“人机审批”。
8.可显示待审批数量角标。
9.数量角标失败不得阻断主布局。
10.不得在每次导航重复创建轮询。
11.VISITOR若ACL无危险工具权限，审批列表通常为空，但页面仍可访问。

# 十七、权限与审批关系演示

必须保持：

## ADMIN

1.拥有危险工具ACL权限。
2.调用delete_demo_record后进入SUSPENDED。
3.ADMIN不能自动绕过人工审批。
4.批准后才真实执行。
5.拒绝后工具不执行。

## VISITOR

若没有`tool:delete_demo_record:invoke`：

1.工具在ACL阶段被拒绝。
2.不创建PendingApproval。
3.不创建Checkpoint。
4.审批中心列表为空。
5.Agent根据PERMISSION_DENIED回答。

## dev API

1.前端不得调用。
2.后端dev用户即使拥有通配工具权限，仍不得绕过审批。
3.agent.dev-api.enabled=false时开发入口关闭。

不得根据角色名在前端硬编码上述结果，实际以后端权限快照和ACL结果为准。

# 十八、重复与并发操作体验

必须正确处理：

1.同一按钮快速点击两次。
2.两个浏览器标签同时打开同一审批。
3.一个标签批准后另一个标签再次拒绝。
4.一个恢复请求正在执行时再次请求。
5.审批完成后页面仍保留旧详情。

前端要求：

1.当前页面提交期间禁用按钮。
2.409显示审批已变化。
3.收到409后重新查询。
4.旧详情不再允许提交。
5.不得自动重试写请求。

后端要求：

1.继续依赖Checkpoint version进行最终防护。
2.只有一个恢复抢占成功。
3.危险工具最多执行一次。
4.冲突不得静默覆盖。
5.不得依赖前端按钮禁用保证安全。

# 十九、Checkpoint终态展示

待审批列表只显示PENDING。

本批不新增完整历史审批页面。

恢复完成后：

1.Checkpoint按第3批策略标记完成并删除。
2.待审批列表中不再出现。
3.前端保留本次响应结果用于展示。
4.刷新页面后无需显示已完成历史。

恢复失败后：

1.FAILED Checkpoint不进入待审批列表。
2.前端展示本次失败响应。
3.不提供继续恢复按钮。
4.失败历史查询不在本批实现。

再次挂起后：

1.同一runId重新进入待审批列表。
2.使用新的approvalId。
3.页面显示新的operationName和safeArguments。
4.旧approvalId不可继续提交。

# 二十、后端最小修复范围

原则上不重构前三批核心逻辑。

仅允许：

1.补充查询所需的CheckpointStore方法。
2.修复Checkpoint不可变快照问题。
3.补充decideAndResume应用方法。
4.修复ReactResumeResult映射字段。
5.修复重复恢复错误码映射。
6.修复恢复后再次挂起无法返回新approvalId。
7.修复HITL Controller联调阻断。
8.补充CORS对现有开发前端的支持。

禁止：

1.重写ReactAgentEngine。
2.重写ToolInvocationGateway。
3.重写ToolApprovalInterceptor。
4.改成“从Java中断行继续”。
5.删除version并发控制。
6.允许裸resume绕过决定。
7.为前端暴露stateData和ToolTrace。
8.添加隐藏调试Controller。
9.添加跨用户管理员审批。

# 二十一、安全检查

必须检查：

1.审批列表只返回当前用户。
2.通过其他用户runId查询不会泄漏详情。
3.决定和恢复均以当前Session.userId校验。
4.客户端不能提交decidedBy。
5.客户端不能提交Checkpoint版本覆盖Store。
6.客户端不能提交operationFingerprint。
7.客户端不能提交原始ToolCall。
8.safeArguments已脱敏。
9.响应中无Session ID。
10.响应中无stateData。
11.日志中无完整工具参数。
12.批准前危险工具无副作用。
13.拒绝后危险工具不执行。
14.ADMIN不会自动批准。
15.模型不能审批。
16.前端不能伪造审批成功。
17.重复点击不会重复执行工具。
18.正式前端不调用开发API。

# 二十二、浏览器联调场景

前置：

1.`agent.sample.enabled=true`。
2.ADMIN用户能够登录。
3.VISITOR用户能够登录。
4.模型配置有效。
5.`approval_demo_agent`已注册。
6.demo-1和demo-2存在。

## 场景1：ADMIN触发中断

1.ADMIN登录。
2.进入人机审批页面。
3.选择demo-1。
4.发起危险删除。
5.Agent返回SUSPENDED。
6.页面出现待审批记录。
7.safeArguments已脱敏且可读。
8.demo-1尚未删除。

## 场景2：ADMIN批准

1.打开待审批详情。
2.点击批准。
3.确认后提交。
4.Checkpoint从SUSPENDED切换RESUMING。
5.危险工具真实执行一次。
6.Agent进入Observe和后续Reason。
7.页面展示最终结果。
8.待审批记录消失。
9.再次列出演示记录时demo-1不存在。

## 场景3：ADMIN拒绝

1.对demo-2发起删除。
2.审批列表出现。
3.点击拒绝。
4.delete_demo_record不执行。
5.Agent收到APPROVAL_REJECTED观察。
6.页面展示拒绝后的最终说明。
7.demo-2仍存在。
8.待审批记录消失。

## 场景4：重复提交

1.两个浏览器标签打开同一审批。
2.第一个标签批准。
3.第二个标签拒绝。

预期：

1.第二个返回409或安全冲突。
2.页面提示刷新。
3.危险工具只执行一次。
4.原审批决定不被覆盖。

## 场景5：VISITOR

1.VISITOR登录。
2.进入人机审批页面。
3.发起删除演示。

预期：

1.ACL拒绝危险工具。
2.Agent不进入SUSPENDED。
3.待审批列表不新增记录。
4.页面显示权限不足。
5.不得因为VISITOR打开审批页面获得审批能力。

## 场景6：Session失效

1.用户停留在审批页。
2.管理员修改该用户角色或禁用账户，使旧Session撤销。
3.页面下次轮询收到401。

预期：

1.清除本地Session。
2.停止轮询。
3.跳转登录页。
4.提示登录状态失效。
5.不得继续提交审批。

## 场景7：再次挂起

当一次运行包含两个危险工具时：

1.批准第一个。
2.恢复后遇到第二个。
3.响应status=SUSPENDED。
4.页面刷新后显示新的approvalId。
5.旧审批不能再次操作。
6.第一个工具不重复执行。

模型不能稳定生成两个ToolCall时，完成代码检查并准确说明未实际验证。

# 二十三、Postman接口验收

## 查询待审批

```http
GET /api/hitl/approvals
X-Session-Id: {{sessionId}}
```

预期：

1.只返回当前用户PENDING审批。
2.不返回stateData。
3.无审批返回空数组。

## 查询详情

```http
GET /api/hitl/approvals/{runId}
X-Session-Id: {{sessionId}}
```

预期返回safeArguments和安全元数据。

## 批准并恢复

```http
POST /api/hitl/approvals/{runId}/decide-and-resume
X-Session-Id: {{sessionId}}
Content-Type: application/json

{
  "approvalId":"...",
  "action":"APPROVE",
  "comment":"确认执行"
}
```

预期：

1.工具真实执行。
2.返回最终结果或新的SUSPENDED。
3.不得返回Checkpoint。

## 拒绝并恢复

```json
{
  "approvalId":"...",
  "action":"REJECT",
  "comment":"不允许执行"
}
```

预期：

1.工具不执行。
2.拒绝ToolResult进入Observe。
3.返回Agent最终说明。

## 伪造runId

使用其他用户runId。

预期：

1.404或安全拒绝。
2.不泄漏审批详情。
3.不改变Checkpoint。

## 缺少Session

预期HTTP401，请求不进入查询或恢复逻辑。

# 二十四、前端构建

在agent-web中根据现有锁文件执行对应命令。

例如使用npm时：

```bash
npm install
npm run build
```

要求：

1.不得删除现有锁文件。
2.不得同时创建多种锁文件。
3.不得引入TypeScript。
4.前端构建必须真实成功。
5.不得因为构建警告伪造成功。
6.浏览器控制台无明显运行错误。
7.轮询timer能正确清理。

后端执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

# 二十五、本批禁止实现

禁止：

1.Supervisor中断恢复。
2.Supervisor审批页面。
3.管理员跨用户审批。
4.审批委托和多人审批。
5.审批历史页面。
6.数据库、Redis或文件Checkpoint。
7.跨进程恢复验证。
8.分布式锁。
9.Checkpoint TTL。
10.审批超时自动拒绝。
11.自动批准规则。
12.模型自行批准。
13.裸resume公开接口。
14.SSE和WebSocket。
15.真实邮件、转账和数据删除。
16.删除或修改AGENTS.md。
17.测试脚本、README和Git操作。

# 二十六、最终验收

确认：

1.后端全部模块编译通过。
2.agent-bootstrap打包通过。
3.前端生产构建通过。
4.HITL接口受Session认证保护。
5.用户只能查看自己的审批。
6.Controller不直接访问CheckpointStore。
7.待审批查询不返回敏感状态。
8.批准后危险工具真实执行一次。
9.拒绝后危险工具不执行。
10.恢复继续使用原runId和threadId。
11.恢复不重新执行首次Reason。
12.重复提交由version防护。
13.恢复后再次挂起能够展示新审批。
14.旧审批不能再次使用。
15.前端使用正式Agent和HITL接口。
16.前端没有调用开发API。
17.前端没有保存密码和Checkpoint。
18.401和409处理正确。
19.轮询不会重复创建。
20.现有认证、ACL、ReAct和Supervisor未被破坏。
21.git diff没有无关修改。

# 二十七、最终输出

只输出：

1.新增和修改文件清单。
2.PendingApprovalQueryService查询规则。
3.HITL接口路径和请求响应结构。
4.身份校验和多用户隔离方式。
5.decide-and-resume完整调用链。
6.错误码与HTTP状态映射。
7.前端HITL审批中心结构。
8.轮询和Session失效处理。
9.危险操作演示流程。
10.ADMIN和VISITOR实际行为差异。
11.重复提交和版本冲突验证。
12.批准、拒绝及再次挂起验证。
13.前端构建结果。
14.后端编译和打包结果。
15.实际完成的浏览器和Postman验证。
16.因模型或环境限制未完成的验证及准确原因。
17.发现但未处理的问题。

前端或后端构建失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE6_BATCH4_END===