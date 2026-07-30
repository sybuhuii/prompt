你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

已完成：
1.模块化单体架构及core/runtime/application/infrastructure/api/bootstrap边界。
2.ToolInvocationGateway及参数校验、异常治理、审计拦截器链。
3.完整单Agent ReAct执行引擎。
4.Supervisor中心调度及多Agent执行引擎。
5.UserAccount、RoleDefinition、PermissionCode、UserSession。
6.UserStore、RoleStore、SessionStore及内存实现。
7.ADMIN、VISITOR角色及工具权限编码。
8.登录、Session校验、X-Session-Id认证。
9.正式Agent和Supervisor认证接口。
10.Session身份到RunContext及父子Agent身份透传。

当前只执行第五阶段第3批：

1.实现工具权限判断规则。
2.实现Tool ACL统一拦截器。
3.把ACL接入现有ToolInvocationGateway拦截器链。
4.越权时不执行真实工具，而是返回失败ToolResult。
5.让失败ToolResult经过ObserveNode回灌LLM。
6.保证ADMIN、VISITOR及开发API权限行为明确。
7.通过Postman验证权限差异。

本批不实现用户和角色管理接口，不实现动态修改权限，不实现前端页面，不实现HITL、记忆或Checkpoint。

一、执行前检查

先检查全部pom.xml以及：

1.ToolInvocationGateway及默认实现。
2.ToolInvocationInterceptor或现有拦截器契约。
3.ToolArgumentValidationInterceptor。
4.ToolExceptionHandlingInterceptor。
5.ToolAuditInterceptor。
6.TerminalToolExecutor。
7.ToolInvocation、ToolCall、ToolResult、ToolDefinition。
8.RunContext。
9.PermissionCode、ToolPermissionCodes。
10.AgentErrorCode和AgentFrameworkException。
11.ReactToolExecutionNode、ReactObserveNode。
12.SupervisorDispatchNode中的子RunContext继承。
13.AuthenticatedAgentApplicationService。
14.AuthenticatedSupervisorApplicationService。
15.ReactDevApplicationService和SupervisorDevApplicationService。
16.ADMIN和VISITOR实际权限配置。
17.现有Spring Tool链装配顺序。
18.现有日志和统一异常处理。

要求：

1.以当前实际接口、构造器和包结构为准增量实现。
2.不得创建第二套ToolInvocationGateway、ToolResult或RunContext。
3.已有权限编码工具能够复用时必须复用。
4.发现前序代码阻断时只做最小修复并说明。
5.不得大规模重构ReAct、Supervisor或认证模块。
6.不得修改无关代码。
7.不要生成测试脚本。
8.不要修改README、说明、升级记录或验收报告。
9.不要执行git commit或git push。

二、核心授权语义

权限判断只依据：

RunContext.permissions

不得依据：

1.客户端请求Body。
2.HTTP请求头中的自定义角色或权限。
3.模型输出。
4.Agent名称。
5.用户名字符串。
6.硬编码ADMIN角色判断。
7.ThreadLocal安全上下文。
8.Spring Security Authentication。
9.工具内部判断。

工具调用权限格式沿用现有ToolPermissionCodes：

具体权限：

tool:{toolName}:invoke

全工具通配权限：

tool:*:invoke

示例：

tool:calculator:invoke
tool:current_time:invoke
tool:echo:invoke
tool:text_search:invoke
tool:*:invoke

授权规则：

1.拥有tool:*:invoke时允许调用任意已注册工具。
2.拥有tool:{当前工具名}:invoke时允许调用该工具。
3.两者都没有时拒绝。
4.角色名称ADMIN本身不代表自动放行。
5.角色名称VISITOR本身不代表自动拒绝。
6.最终权限由Session创建时保存的permissions快照决定。
7.权限集合为空时默认拒绝全部工具。
8.RunContext为空或身份信息不完整时必须失败关闭，不得默认放行。
9.工具不存在的处理继续沿用现有ToolRegistry和Gateway语义，ACL不替代工具存在性校验。
10.AgentDefinition.allowedTools与用户ACL是两层不同约束：
-allowedTools决定模型可以请求哪些工具。
-ACL决定当前用户是否有权真正执行。
-两层都通过才允许执行。

三、权限判断模型

在agent-runtime中创建清晰包，例如：

com.ksyun.agent.runtime.tool.authorization

实现不可变：

ToolAuthorizationDecision

建议字段：

boolean allowed
String requiredPermission
String reasonCode

要求：

1.allowed表示是否允许。
2.requiredPermission保存当前工具所需权限编码。
3.reasonCode只保存稳定的非敏感原因，例如：
ALLOWED_EXACT
ALLOWED_WILDCARD
MISSING_PERMISSION
MISSING_CONTEXT
4.不得保存完整权限集合。
5.不得保存UserSession。
6.不得保存密码、sessionId或HTTP对象。
7.不得依赖Spring。
8.不得使用null表达决策。

实现接口：

ToolPermissionEvaluator

建议方法：

ToolAuthorizationDecision evaluate(
RunContext context,
String toolName
);

要求：

1.保持纯Java。
2.不依赖ToolRegistry。
3.不执行工具。
4.不抛出权限拒绝异常作为正常控制流。
5.输入非法时返回拒绝决策。
6.不得修改RunContext。
7.保持无状态和线程安全。

实现：

DefaultToolPermissionEvaluator

职责：

1.校验toolName非空。
2.通过ToolPermissionCodes生成精确权限。
3.检查通配权限。
4.检查精确权限。
5.通配权限优先或精确权限优先均可，但行为必须确定。
6.权限匹配必须完全匹配，不使用contains、startsWith或模糊匹配。
7.不得把tool:text:invoke错误匹配到tool:text_search:invoke。
8.不得根据roles做特殊放行。
9.返回ToolAuthorizationDecision。
10.不得记录完整permissions。

四、实现Tool ACL拦截器

在agent-runtime现有工具拦截器包中实现：

ToolAccessControlInterceptor

实现现有ToolInvocationInterceptor接口。

依赖：

ToolPermissionEvaluator

使用构造器注入，不添加Spring注解。

执行流程：

1.读取ToolInvocation。
2.取得ToolCall及toolName。
3.取得RunContext。
4.调用ToolPermissionEvaluator.evaluate。
5.允许时调用拦截器链next。
6.拒绝时不得调用next。
7.拒绝时不得解析或执行工具参数。
8.拒绝时不得直接抛出普通权限异常终止ReAct。
9.拒绝时构造success=false的ToolResult。
10.errorCode使用PERMISSION_DENIED。
11.content使用可安全回灌LLM的说明。
12.不得返回null。

建议安全content：

当前用户没有调用工具“{toolName}”的权限。请不要假设工具已经执行，可向用户说明权限不足，或选择其他已授权工具。

要求：

1.可以包含工具名称。
2.不得包含用户完整权限集合。
3.不得包含角色全集。
4.不得包含sessionId。
5.不得包含权限判断内部实现。
6.不得包含Java类名或堆栈。
7.不得伪造工具执行结果。
8.ToolResult必须保持与原ToolCall的关联语义。
9.使用项目现有ToolResult字段和工厂方法，不创建第二套结果模型。
10.如ToolResult支持metadata，可仅加入非敏感信息：
-denied=true
-requiredPermission
不得加入RunContext或Session。
11.拦截器必须无状态、可并发复用。
12.不得缓存用户权限。
13.不得访问SessionStore、UserStore或RoleStore。
14.不得在每次工具调用时重新解析角色。
15.不得使用ThreadLocal。

五、拦截器顺序

检查当前DefaultToolInvocationGateway及Spring配置中的真实链顺序。

目标顺序应保证：

1.异常治理覆盖整个调用链。
2.审计能够记录允许和拒绝的工具调用。
3.ACL在参数校验和真实工具执行之前完成。
4.越权用户不能通过参数校验错误探测工具内部细节。
5.只有授权通过后才进入参数校验。
6.只有授权且参数合法才进入TerminalToolExecutor。

推荐逻辑顺序：

ToolExceptionHandlingInterceptor
→ToolAuditInterceptor
→ToolAccessControlInterceptor
→ToolArgumentValidationInterceptor
→TerminalToolExecutor

具体列表顺序必须结合现有链实现判断，不得机械照抄。

验收语义：

1.权限拒绝必须被审计。
2.权限拒绝不进入参数校验。
3.权限拒绝不进入AgentTool.execute。
4.工具异常仍由异常治理拦截器处理。
5.不得重复执行任何拦截器。
6.不得恢复共享可变index。
7.不得使用ThreadLocal推进链。
8.同一Gateway支持并发请求。

六、错误码语义

检查AgentErrorCode。

缺失时最小增加：

PERMISSION_DENIED

不要增加语义重复错误码，例如：

TOOL_PERMISSION_DENIED
ACCESS_FORBIDDEN
ACL_FAILED

除非项目已经存在其中之一，应优先复用。

要求：

1.ToolResult中的errorCode使用现有类型。
2.权限拒绝属于可观察的工具失败，不属于框架崩溃。
3.不得因为权限拒绝直接进入ReactFailureNode。
4.不得把权限拒绝转换成INTERNAL_ERROR。
5.正式HTTP接口本身已通过Session认证；工具越权发生在Agent运行内部，因此通常仍返回Agent最终结果，而不是立即HTTP403。
6.直接的权限管理HTTP操作将来才使用HTTP403，本批不实现。
7.若模型成功解释权限不足，最终AgentResult可以success=true。
8.若Agent无法在迭代限制内处理，则沿用现有终止语义。

七、ReAct权限失败回灌

检查DefaultReactToolExecutionNode和DefaultReactObserveNode。

必须保证：

1.ToolAccessControlInterceptor返回success=false的ToolResult。
2.ToolExecutionNode正常收集该ToolResult。
3.ToolExecutionTrace记录：
-toolName
-success=false
-errorCode=PERMISSION_DENIED
-duration
4.ToolExecutionNode不得看到PERMISSION_DENIED后直接设置TOOL_ERROR。
5.ObserveNode必须为该结果创建ToolAgentMessage。
6.ToolAgentMessage.error=true。
7.ToolAgentMessage使用原ToolCall.id。
8.ToolAgentMessage使用原toolName。
9.ToolAgentMessage.content使用ACL生成的安全content。
10.Observe后重新进入ReasonNode。
11.模型能看到“权限不足”的工具观察。
12.模型不得看到完整permissions、roles或sessionId。
13.现有普通工具失败回灌行为不能被破坏。
14.如现有节点已经满足，只做代码审查，不重复重构。
15.不得在ObserveNode中再次做ACL判断。

八、Supervisor身份继承

检查SupervisorDispatchNode。

必须保证每个子Agent RunContext继承父RunContext的：

1.userId。
2.sessionId。
3.roles。
4.permissions。

同时保持：

1.childRunId独立。
2.childThreadId独立。
3.AgentTask独立。
4.ReactAgentState独立。
5.不得把父Supervisor消息传给子Agent。
6.不得根据目标Agent重新提升权限。
7.不得给calculator_agent自动附加calculator权限。
8.不得给utility_agent自动附加全权限。
9.子Agent工具调用最终仍经过ToolAccessControlInterceptor。
10.同一visitor通过Supervisor调用text_search时也必须被拒绝。

若现有继承实现正确，不做无关重构。

九、开发API权限策略

现有开发API：

POST /api/dev/react/invoke
POST /api/dev/supervisor/invoke

此前可能使用：

roles=空
permissions=空

接入ACL后，这会导致所有开发工具测试被拒绝。

本批将开发API身份明确设置为普通权限上下文，而不是在ACL中增加dev绕过。

要求：

1.ReactDevApplicationService创建的RunContext使用：
userId=dev-user
sessionId=dev-session
roles可使用DEV或保持现有安全值
permissions包含tool:*:invoke

2.SupervisorDevApplicationService创建的父RunContext同样包含tool:*:invoke。
3.开发API仍受agent.dev-api.enabled控制。
4.不得在ToolAccessControlInterceptor中判断userId=dev-user后直接放行。
5.不得根据/api/dev路径绕过ACL。
6.开发API和正式API都走完全相同的ACL判断。
7.差别仅在于开发RunContext被服务器显式授予通配权限。
8.客户端仍不能通过请求Body提交permissions。
9.正式visitor不得获得开发权限。
10.agent.dev-api.enabled=false时开发入口仍不可访问。

十、ADMIN和VISITOR权限

检查第1批实际Sample角色配置。

目标语义：

ADMIN：

tool:*:invoke

VISITOR至少允许：

tool:calculator:invoke
tool:current_time:invoke
tool:echo:invoke

VISITOR默认不允许：

tool:text_search:invoke

要求：

1.不得根据roleName=ADMIN硬编码放行。
2.ADMIN依赖tool:*:invoke获得全权限。
3.VISITOR依赖具体权限获得部分权限。
4.登录创建Session时保存权限快照。
5.ACL只读取RunContext.permissions。
6.已有Session不会因RoleStore修改自动更新。
7.本批不实现角色权限动态更新。
8.权限变更后重新登录生成新Session的语义可保留到下一批。
9.如果实际Sample权限与上述不同，以现有明确设计为准，但必须能演示至少一个工具权限差异。
10.最终输出说明实际差异工具。

十一、Spring Bean装配

在agent-infrastructure现有工具配置中装配：

ToolPermissionEvaluator
→DefaultToolPermissionEvaluator

ToolAccessControlInterceptor

并接入现有ToolInvocationGateway拦截器集合。

要求：

1.使用@Configuration和@Bean。
2.runtime实现不添加@Component、Service或Autowired。
3.不得创建第二个DefaultToolInvocationGateway。
4.不得创建第二套拦截器链。
5.不得在bootstrap启动类手工new。
6.不得产生Bean循环依赖。
7.无模型配置时Tool Gateway和ACL仍能装配。
8.无用户或Session时应用仍能启动。
9.允许未来通过自定义ToolPermissionEvaluator Bean替换默认实现。
10.优先使用@ConditionalOnMissingBean或项目现有扩展方式。
11.拦截器顺序必须明确且稳定。
12.不要依赖Spring的未声明排序偶然行为。
13.如使用List<ToolInvocationInterceptor>自动注入，应通过@Order或显式Bean组装保证顺序。
14.不得让同一ACL拦截器注册两次。

十二、审计和日志

权限拒绝日志至少可记录：

1.runId。
2.userId。
3.toolName。
4.authorization=DENIED。
5.reasonCode。

允许记录username时应遵循现有日志规范。

禁止记录：

1.sessionId。
2.完整permissions。
3.完整roles。
4.工具完整参数。
5.工具完整结果。
6.密码或credentialHash。
7.完整Prompt。
8.完整messages。
9.API Key。

要求：

1.拒绝调用必须进入现有工具审计链。
2.不能因为拒绝而跳过审计。
3.不要建立新的数据库审计系统。
4.不要引入Kafka、Prometheus或OpenTelemetry。
5.不得把审计信息直接返回客户端。
6.不得把完整requiredPermission集合反馈给LLM。
7.单个requiredPermission编码可以作为安全诊断信息，但优先只返回通用权限不足说明。

十三、正式Agent接口验证

正式接口：

POST /api/agent/invoke
X-Session-Id: ...

必须保证：

1.Session认证仍在入口完成。
2.RunContext.permissions来自UserSession。
3.Controller不能提交permissions。
4.AgentDefinition.allowedTools仍生效。
5.Tool ACL最终执行生效。
6.没有权限时真实工具不执行。
7.权限失败通过ToolResult回灌模型。
8.响应不得返回Session或RunContext。
9.权限失败不得导致NullPointerException或HTTP500。
10.模型不可用时按现有模型错误处理。

十四、正式Supervisor接口验证

正式接口：

POST /api/supervisor/invoke
X-Session-Id: ...

必须保证：

1.父RunContext来自UserSession。
2.子Agent继承父permissions。
3.每个子Agent工具调用经过相同ACL。
4.Supervisor本身不直接执行业务工具。
5.子Agent权限失败结果可被子Agent模型解释。
6.子Agent最终AgentResult再反馈Supervisor。
7.Supervisor不得提升子Agent权限。
8.一个子Agent权限失败不应直接破坏整个父图。
9.客户端不能通过memberAgents或请求字段控制权限。
10.父子runId和threadId隔离保持不变。

十五、Postman验收

前置条件：

1.agent.sample.enabled=true。
2.配置Sample用户密码。
3.agent.model.enabled=true。
4.模型配置有效。
5.utility_agent允许使用text_search、calculator、current_time等工具。

场景1：admin登录

POST /api/auth/login

{
"username":"admin",
"password":"配置的admin密码"
}

保存返回sessionId为adminSession。

预期：

1.roles包含ADMIN。
2.permissions包含tool:*:invoke。
3.不返回credentialHash。

场景2：visitor登录

POST /api/auth/login

{
"username":"visitor",
"password":"配置的visitor密码"
}

保存sessionId为visitorSession。

预期：

1.roles包含VISITOR。
2.具有有限工具权限。
3.不包含tool:*:invoke。
4.不包含text_search调用权限。

场景3：admin调用text_search

POST /api/agent/invoke
X-Session-Id: {{adminSession}}

{
"agentName":"utility_agent",
"message":"必须使用text_search在以下文本中搜索Java，并根据真实工具结果回答：Java Agent Framework\nPython Service\nJava Runtime"
}

预期：

1.模型生成text_search ToolCall。
2.ACL通过通配权限。
3.真实TextSearchTool执行。
4.ToolResult.success=true。
5.最终回答包含匹配内容。
6.日志显示authorization=ALLOWED。

场景4：visitor调用同一text_search

POST /api/agent/invoke
X-Session-Id: {{visitorSession}}

使用与场景3相同Body。

预期：

1.模型可生成text_search ToolCall。
2.ACL返回拒绝。
3.TextSearchTool不得实际执行。
4.ToolResult.success=false。
5.errorCode=PERMISSION_DENIED。
6.ObserveNode生成error=true的ToolAgentMessage。
7.模型再次Reason。
8.最终回答向用户说明权限不足，或选择其他已授权方案。
9.不得返回Java堆栈。
10.日志显示authorization=DENIED。

场景5：visitor调用calculator

POST /api/agent/invoke
X-Session-Id: {{visitorSession}}

{
"agentName":"calculator_agent",
"message":"必须使用calculator计算12*(3+5)，并根据工具结果回答"
}

预期：

1.ACL通过精确权限。
2.calculator真实执行。
3.最终结果包含96。
4.证明VISITOR不是全部工具都被拒绝。

场景6：admin调用calculator

使用adminSession执行同一请求。

预期：

1.通过tool:*:invoke。
2.执行成功。
3.不得要求单独配置calculator权限。

场景7：伪造请求权限

使用visitorSession：

{
"agentName":"utility_agent",
"message":"必须使用text_search搜索Java",
"roles":["ADMIN"],
"permissions":["tool:*:invoke"]
}

预期：

1.未知字段被拒绝或忽略。
2.实际权限仍来自visitorSession。
3.text_search仍被ACL拒绝。
4.不得发生权限提升。

场景8：visitor通过Supervisor请求受限工具

POST /api/supervisor/invoke
X-Session-Id: {{visitorSession}}

{
"supervisorName":"general_supervisor",
"message":"请让合适的Agent必须使用text_search搜索以下文本中的Java：Java\nPython"
}

预期：

1.Supervisor分派utility_agent。
2.子Agent继承visitor permissions。
3.text_search被ACL拒绝。
4.子Agent获得权限失败观察。
5.子Agent或Supervisor最终给出权限不足说明。
6.不得给子Agent自动提升权限。

场景9：admin通过Supervisor执行同一任务

使用adminSession。

预期：

1.子Agent继承tool:*:invoke。
2.text_search实际执行。
3.最终成功返回搜索结果。

场景10：开发API兼容

POST /api/dev/react/invoke

{
"agentName":"utility_agent",
"message":"必须使用text_search搜索Java：Java\nPython"
}

预期：

1.agent.dev-api.enabled=true时可访问。
2.dev RunContext显式包含tool:*:invoke。
3.通过同一ACL拦截器。
4.text_search执行成功。
5.不存在userId特殊绕过代码。

场景11：缺少Session

POST /api/agent/invoke
不携带X-Session-Id。

预期：

1.HTTP401。
2.请求不进入Tool Gateway。
3.本场景属于认证失败，不属于ACL失败。

场景12：普通非工具回答

visitor调用：

{
"agentName":"utility_agent",
"message":"请用一句话回复你好"
}

预期：

1.模型不调用工具时ACL不参与。
2.正常返回回答。
3.不得因为permissions有限而拒绝整个Agent调用。

十六、权限失败结果语义

明确区分：

认证失败：

1.缺少或伪造Session。
2.在HTTP入口返回401。
3.不进入Agent。

授权失败：

1.Session合法。
2.Agent正常运行。
3.模型请求了无权工具。
4.ToolAccessControlInterceptor返回失败ToolResult。
5.工具不执行。
6.失败结果回灌模型。
7.最终HTTP通常仍是200，响应为Agent最终回答。
8.不得把工具权限失败直接转换成HTTP401。
9.不得把工具权限失败默认转换成HTTP403中断Agent。
10.未来直接管理权限的HTTP接口才使用403。

十七、多用户隔离

必须保证：

1.每次ACL判断只读取当前ToolInvocation.runContext。
2.不同runId之间不共享权限状态。
3.不同用户之间不缓存授权结果。
4.不得使用static当前用户。
5.不得使用ThreadLocal用户。
6.不得把上一次调用permissions保存到拦截器字段。
7.同一ToolAccessControlInterceptor支持并发用户。
8.admin请求不能使visitor后续请求获得权限。
9.visitor拒绝不能影响admin执行。
10.父子Agent只继承自己的父RunContext。
11.身份信息不发送给LLM。
12.工具和节点可读取RunContext，但不得修改它。

十八、本批禁止实现

禁止：

1.在CalculatorTool、TextSearchTool等具体工具中写权限判断。
2.根据roleName=ADMIN直接绕过。
3.在Controller中判断工具权限。
4.在ReasonNode中作为唯一权限控制。
5.模型决定是否有权限。
6.工具权限动态更新接口。
7.用户管理Controller。
8.角色管理Controller。
9.权限管理Controller。
10.强制注销旧Session。
11.Session权限实时刷新。
12.前端页面。
13.Spring Security Web。
14.JWT或OAuth。
15.HITL。
16.CheckpointStore。
17.上下文裁剪。
18.长期记忆。
19.RAG和MCP。
20.真实金山云业务。
21.新增并行工具执行。
22.模型重试、熔断或多模型路由。
23.测试脚本。
24.README、说明、升级记录或验收报告。
25.git commit和git push。

十九、代码质量

1.不使用Lombok。
2.使用构造器注入。
3.runtime实现保持纯Java。
4.集合使用不可变快照。
5.不使用字段注入。
6.不通过ApplicationContext主动查Bean。
7.不吞异常。
8.不返回null。
9.不使用共享可变拦截器索引。
10.不使用Object、反射或instanceof做节点分派。
11.不创建第二套工具执行链。
12.不创建无意义Facade、Delegate或Manager。
13.ACL逻辑只存在于统一拦截层。
14.权限匹配必须精确。
15.授权失败必须失败关闭。
16.所有实现以实际编译和运行结果为准。

二十、编译和启动验收

执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许启动并验证：

GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents
GET /api/framework/supervisors
POST /api/auth/login
POST /api/agent/invoke
POST /api/supervisor/invoke
POST /api/dev/react/invoke

检查：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core没有SpringWeb、SpringAI和LangGraph4j依赖。
4.agent-runtime没有SpringWeb和agent-infrastructure依赖。
5.ToolPermissionEvaluator保持纯Java。
6.ToolAccessControlInterceptor位于统一工具链。
7.具体工具中不存在权限判断。
8.ADMIN通过tool:*:invoke获得全权限。
9.VISITOR通过精确权限调用允许工具。
10.VISITOR调用text_search被拒绝。
11.拒绝时真实工具不执行。
12.拒绝结果进入ToolExecutionNode。
13.ObserveNode生成error=true的ToolAgentMessage。
14.模型可根据权限失败继续回答。
15.工具拒绝不直接进入ReactFailureNode。
16.Supervisor子Agent继承父permissions。
17.开发API通过显式通配权限保持可用。
18.不存在dev-user特殊绕过。
19.无模型配置时应用仍能启动。
20.现有认证和ReAct功能未被破坏。
21.git diff没有无关修改。

二十一、最终输出

只输出：

1.新增和修改文件清单。
2.ToolAuthorizationDecision结构。
3.ToolPermissionEvaluator匹配规则。
4.ToolAccessControlInterceptor执行流程。
5.最终拦截器顺序及原因。
6.权限拒绝ToolResult格式。
7.拒绝结果如何经过Observe回灌LLM。
8.ADMIN和VISITOR实际权限差异。
9.开发API权限策略。
10.Supervisor父子权限继承验证。
11.日志和审计行为。
12.编译、打包和启动结果。
13.实际完成的Postman验证结果。
14.因缺少模型或密码配置未完成的验证及原因。
15.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE5_BATCH3_END===