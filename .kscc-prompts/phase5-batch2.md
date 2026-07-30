你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven模块化单体架构。
2.agent-core领域模型及稳定接口。
3.AgentRegistry、ToolRegistry、SupervisorRegistry及Provider自动注册。
4.ToolInvocationGateway及工具拦截器链。
5.ModelInvocationGateway及SpringAI适配。
6.完整单Agent ReAct执行引擎。
7.Supervisor中心调度及多Agent执行引擎。
8.Sample Agent、Sample Supervisor及开发验证API。
9.UserAccount、RoleDefinition、PermissionCode、UserSession。
10.UserStore、RoleStore、SessionStore及线程安全内存实现。
11.RolePermissionResolver、CredentialHasher、SessionIdGenerator。
12.ADMIN与VISITOR示例角色。

当前只执行第五阶段第2批：

1.实现用户名密码登录。
2.登录成功后创建并保存Session。
3.实现Session统一校验。
4.实现登出和当前身份查询。
5.使用Session保护正式Agent和Supervisor调用接口。
6.从已校验Session构造RunContext。
7.保证客户端不能伪造userId、roles和permissions。
8.为Postman验证提供可关闭的Sample用户。

本批不实现工具ACL拦截，不实现权限拒绝回灌LLM，不实现用户/角色管理接口，不实现前端页面，不实现JWT、OAuth或Spring Security过滤链。

一、最终调用链

实现以下认证链路：

登录：

POST /api/auth/login
→AuthenticationApplicationService
→UserStore查询用户
→CredentialHasher校验密码
→RolePermissionResolver解析权限
→SessionIdGenerator生成sessionId
→SessionStore保存UserSession
→返回sessionId

正式Agent调用：

POST /api/agent/invoke
请求头X-Session-Id
→SessionAuthenticationInterceptor
→SessionValidationService
→SessionStore查询真实Session
→校验有效性
→AuthenticatedAgentApplicationService
→从Session构造RunContext
→ReactAgentEngine
→返回AgentResult

正式Supervisor调用：

POST /api/supervisor/invoke
请求头X-Session-Id
→SessionAuthenticationInterceptor
→SessionValidationService
→AuthenticatedSupervisorApplicationService
→从Session构造RunContext
→SupervisorEngine
→子Agent继承身份上下文
→返回AgentResult

强制要求：

1.客户端只能提交sessionId，不能决定userId、roles或permissions。
2.RunContext身份必须来自服务器已校验的UserSession。
3.无效Session必须在HTTP入口处拒绝。
4.无效请求不得进入ReactAgentEngine或SupervisorEngine。
5.身份上下文不得发送给LLM。
6.身份上下文必须对节点和工具调用链可见。
7.本批暂不根据permissions拒绝工具调用。

二、执行前检查

先完整检查全部pom.xml和现有代码：

1.UserAccount。
2.RoleDefinition。
3.PermissionCode、ToolPermissionCodes。
4.UserSession。
5.UserStore、RoleStore、SessionStore。
6.InMemoryUserStore、InMemoryRoleStore、InMemorySessionStore。
7.RolePermissionResolver。
8.CredentialHasher。
9.SessionIdGenerator。
10.RunContext。
11.AgentTask、AgentResult。
12.ReactAgentEngine。
13.SupervisorEngine。
14.ReactDevApplicationService。
15.SupervisorDevApplicationService。
16.现有开发Controller和统一异常处理。
17.AgentErrorCode。
18.AgentFrameworkException。
19.现有Spring MVC配置。
20.现有配置属性命名方式。
21.SampleSecurityConfiguration或现有Sample配置。
22.当前UserSession是否包含expiresAt、active及username。
23.当前RunContext的实际构造参数。
24.当前Java和Spring Boot版本。

要求：

1.以现有真实接口、包名和构造方法为准增量实现。
2.不得创建UserSessionV2、SecurityContextV2、NewRunContext等重复模型。
3.已有抽象能够复用时必须复用。
4.发现第1批存在阻断问题时只做最小修复并说明。
5.不得大规模重构ReAct、Supervisor、ModelGateway或ToolGateway。
6.不得修改无关代码。
7.不得为了完成认证引入整套Spring Security Web。
8.不得生成测试脚本。
9.不得编写或修改README、使用说明、升级记录、验收报告。
10.不得执行git commit或git push。

三、架构边界

继续保持：

agent-core：
-身份、Session及稳定接口。
-不得依赖Spring、SpringWeb、SpringSecurity、SpringAI、LangGraph4j或Jackson。

agent-runtime：
-ReAct、Supervisor和后续工具ACL。
-本批不实现HTTP认证。
-不得依赖SpringWeb。

agent-application：
-登录、登出、Session校验和认证后的Agent/Supervisor调用用例。
-保持纯Java。
-不得依赖SpringWeb、SpringAI或LangGraph4j。

agent-infrastructure：
-应用服务和Clock等Spring Bean装配。
-密码哈希和内存Store技术实现。
-不得包含Controller。

agent-api：
-登录、登出、当前身份、正式Agent和Supervisor HTTP入口。
-Session认证拦截器及MVC注册。
-不得直接调用ModelInvocationGateway或ToolInvocationGateway。

agent-bootstrap：
-Sample用户定义和安全配置。
-不得实现登录流程。
-不得实现Session校验逻辑。

四、认证结果模型

在agent-application创建或复用不可变结果模型：

AuthenticationResult

建议字段：

String sessionId
String userId
String username
Set<String> roles
Set<String> permissions
Instant createdAt
Instant expiresAt

要求：

1.字段以当前UserSession实际结构为准。
2.使用record或等价不可变类型。
3.roles和permissions返回不可变Set。
4.不得包含credentialHash。
5.不得包含原始密码。
6.不得包含UserAccount完整对象。
7.不得包含Spring Security Authentication。
8.不得包含ServletRequest。
9.若UserSession没有expiresAt，则不要为了匹配建议字段重复增加第二套过期语义。
10.不得返回SessionStore实现类信息。

五、SessionValidationService

在agent-application实现纯Java：

SessionValidationService


依赖：

SessionStore
UserStore
Clock

建议方法：

UserSession requireValidSession(String sessionId);

职责：

1.sessionId为空时抛SESSION_INVALID或现有等价错误。
2.通过SessionStore.find查询真实Session。
3.查不到时抛SESSION_NOT_FOUND或SESSION_INVALID。
4.不得通过sessionId格式推断用户身份。
5.校验Session是否active。
6.若UserSession包含expiresAt：
-使用注入的Clock判断是否过期。
-过期时抛SESSION_EXPIRED。
-允许明确调用SessionStore.delete删除已过期Session。
-不得在普通Store.find内部静默删除。
7.通过UserStore确认对应用户仍存在。
8.确认用户enabled=true。
9.用户已禁用时拒绝Session。
10.不得信任客户端提交的userId。
11.不得重新从HTTP请求构造roles和permissions。
12.返回SessionStore中的不可变UserSession。
13.不得修改Session。
14.不得自动续期。
15.不得创建新Session。
16.不得使用ThreadLocal。
17.不得记录sessionId。
18.保持无状态和线程安全。

错误语义：

-缺少Session：SESSION_INVALID。
-伪造或不存在Session：SESSION_INVALID或SESSION_NOT_FOUND。
-过期Session：SESSION_EXPIRED。
-用户不存在或被禁用：SESSION_INVALID或USER_DISABLED。

HTTP层最终都应映射为401，避免泄漏Session是否曾真实存在。

六、AuthenticationApplicationService

在agent-application实现纯Java：

AuthenticationApplicationService

依赖：

UserStore
RolePermissionResolver
CredentialHasher
SessionIdGenerator
SessionStore
SessionValidationService
Clock

职责至少包括：

AuthenticationResult login(String username, CharSequence credential);

void logout(String sessionId);

UserSession currentSession(String sessionId);

登录流程：

1.统一规范化username。
2.username或credential为空时返回认证失败。
3.通过UserStore.findByUsername查询。
4.用户不存在和密码错误对客户端返回相同认证错误。
5.不得通过错误信息暴露用户名是否存在。
6.用户enabled=false时拒绝登录。
7.使用CredentialHasher.matches校验。
8.不得自行比较明文密码。
9.不得记录credential或credentialHash。
10.通过RolePermissionResolver根据用户roleNames解析权限。
11.不得信任UserAccount中客户端可修改的权限字段。
12.使用SessionIdGenerator创建不可预测sessionId。
13.创建UserSession：
-sessionId使用生成值。
-userId来自UserAccount。
-username来自UserAccount。
-roles来自UserAccount.roleNames。
-permissions来自RolePermissionResolver结果。
-createdAt使用Clock。
-active=true。
14.若现有UserSession有expiresAt：
-读取统一Session TTL配置。
-使用createdAt加TTL。
-不得在业务代码中硬编码多个不同TTL。
15.通过SessionStore.save保存。
16.保存成功后返回AuthenticationResult。
17.保存失败时不得返回sessionId。
18.不得生成JWT。
19.不得将Session写入日志。
20.每次登录默认创建独立Session，不强制踢掉旧Session。

登出流程：

1.先通过SessionValidationService校验当前Session。
2.只删除当前sessionId。
3.不得删除该用户全部Session。
4.删除操作幂等。
5.登出后原sessionId不能继续使用。
6.不得要求客户端提交userId。

currentSession：

1.通过SessionValidationService读取。
2.不得绕过校验直接SessionStore.find。
3.只返回当前Session。

异常处理：

1.未知用户名和密码错误统一使用AUTHENTICATION_FAILED或INVALID_CREDENTIALS。
2.不得泄漏用户名是否存在。
3.普通异常转换为INTERNAL_ERROR。
4.不得捕获Error等JVM严重错误。
5.不得在错误消息中包含密码、哈希或sessionId。

七、Session TTL配置

检查UserSession是否已有expiresAt。

情况一：已有expiresAt：

增加配置：

agent.security.session.ttl

建议安全默认值：

PT8H

要求：

1.使用java.time.Duration。
2.必须大于0。
3.创建Session时统一使用。
4.校验时使用Clock。
5.本批不实现自动续期。
6.本批不实现refresh token。
7.不得在多个类中分别读取配置字符串。
8.通过配置属性对象或@Bean传递Duration。

情况二：UserSession没有expiresAt：

1.本批不要为了TTL大规模改造。
2.使用active状态完成基础Session校验。
3.在最终输出中说明当前未实现过期机制。
4.不得创建与UserSession并行的SessionExpiry对象。

八、HTTP Session传递规范

统一使用请求头：

X-Session-Id

在agent-api集中定义：

SessionHttpHeaders

例如：

public static final String SESSION_ID = "X-Session-Id";

要求：

1.不得在多个Controller散落字符串。
2.sessionId不得放入URL查询参数。
3.sessionId不得放入Agent请求Body。
4.本批不使用Cookie。
5.本批不使用Authorization Bearer。
6.响应中只有登录接口返回sessionId。
7.其他接口不得重复返回sessionId。
8.日志不得记录该请求头内容。

九、Session认证拦截器

在agent-api实现：

SessionAuthenticationInterceptor implements HandlerInterceptor

优先使用Spring MVC HandlerInterceptor，不引入Spring Security过滤链。

依赖：

SessionValidationService

职责：

1.对受保护接口统一读取X-Session-Id。
2.缺失或空值时直接认证失败。
3.调用SessionValidationService.requireValidSession。
4.验证成功后将UserSession放入当前HttpServletRequest的attribute。
5.不得放入ThreadLocal。
6.不得修改全局静态上下文。
7.不得自行创建RunContext。
8.不得调用ReactAgentEngine。
9.不得调用SupervisorEngine。
10.不得调用工具。
11.不得把Session写进日志。
12.异常交给现有统一异常处理。
13.无效Session请求不得进入Controller方法。
14.OPTIONS预检请求按照现有CORS策略放行。
15.不得拦截静态资源。

集中定义请求属性Key：

AuthenticatedSessionAttributes.SESSION

要求：

1.不得使用随意字符串。
2.Controller通过@RequestAttribute读取。
3.属性值使用UserSession或现有稳定认证身份模型。
4.不得把HttpServletRequest传入ApplicationService。

十、MVC拦截路径

通过WebMvcConfigurer注册SessionAuthenticationInterceptor。

保护：

/api/agent/**
/api/supervisor/**
/api/auth/me
/api/auth/logout

明确排除：

/api/auth/login
/api/framework/**
/api/dev/**
/error
静态资源
OPTIONS请求

要求：

1.开发API仍由agent.dev-api.enabled控制。
2.开发API当前可以保持原有dev-user语义。
3.不得让开发API与正式认证API共享同一路径。
4.正式API必须始终要求Session。
5.不得通过Controller内重复写Session校验代替统一拦截器。
6.不得拦截应用健康检查和框架查询接口。
7.无模型配置时登录、登出和me接口仍应可用。

十一、认证Controller

在agent-api实现：

AuthenticationController

接口一：

POST /api/auth/login

请求：

{
"username":"admin",
"password":"..."
}

要求：

1.使用专门LoginRequest DTO。
2.password使用String仅限HTTP DTO边界，传入应用层时作为CharSequence。
3.校验username和password非空。
4.调用AuthenticationApplicationService.login。
5.不得直接访问UserStore。
6.不得直接调用CredentialHasher。
7.不得直接创建UserSession。
8.登录成功返回AuthenticationResult映射后的DTO。
9.错误凭证返回HTTP401。
10.不得区分“用户不存在”和“密码错误”。
11.响应不得返回credentialHash。
12.不得设置Cookie。
13.不得返回模型或框架信息。

接口二：

POST /api/auth/logout

要求：

1.由SessionAuthenticationInterceptor保护。
2.从请求attribute取得已验证UserSession。
3.调用AuthenticationApplicationService.logout(sessionId)。
4.不得从请求Body重新读取sessionId。
5.成功返回204或项目统一成功响应。
6.登出后原Session立即失效。

接口三：

GET /api/auth/me

要求：

1.由SessionAuthenticationInterceptor保护。
2.从request attribute读取已验证Session。
3.返回：
-userId
-username
-roles
-permissions
-createdAt
-expiresAt（存在时）
4.不得返回sessionId。
5.不得返回credentialHash。
6.不得返回UserStore内部对象。
7.不得重新查询客户端传入userId。

十二、正式Agent应用服务

在agent-application实现：

AuthenticatedAgentApplicationService

保持纯Java，不添加Spring注解。

依赖：

AgentRegistry
ReactAgentEngine
RunIdGenerator

建议方法：

AuthenticatedAgentRunResult invoke(
UserSession session,
String agentName,
String message
);

职责：

1.session必须来自已验证入口。
2.校验session、agentName、message非空。
3.通过AgentRegistry.getRequired查询AgentDefinition。
4.不得根据请求临时创建AgentDefinition。
5.生成独立runId。
6.生成threadId，例如：
user-{userId}-thread-{runId}
7.生成taskId。
8.构造AgentTask：
-agentName使用注册定义名称。
-instruction使用message。
-context为空且不可变。
9.构造RunContext：
-userId=session.userId。
-sessionId=session.sessionId。
-threadId=生成值。
-runId=生成值。
-roles=session.roles。
-permissions=session.permissions。
10.不得使用请求Body中的身份字段。
11.调用ReactAgentEngine.execute。
12.不得直接调用模型。
13.不得直接调用工具。
14.不得自己实现ReAct循环。
15.不得把RunContext发送给LLM。
16.返回runId、threadId、agentName和AgentResult。
17.不得返回sessionId。
18.保持无状态和线程安全。
19.AgentResult为空时转换为INTERNAL_ERROR。

可定义不可变：

AuthenticatedAgentRunResult

不得包含SpringAI、Servlet或LangGraph4j类型。

十三、正式Supervisor应用服务

在agent-application实现：

AuthenticatedSupervisorApplicationService

依赖：

SupervisorRegistry
SupervisorEngine
RunIdGenerator

建议方法：

AuthenticatedSupervisorRunResult invoke(
UserSession session,
String supervisorName,
String message
);

职责：

1.session必须来自已验证入口。
2.校验参数。
3.通过SupervisorRegistry.getRequired查询SupervisorDefinition。
4.生成父runId。
5.生成父threadId，例如：
user-{userId}-supervisor-thread-{runId}
6.生成rootTaskId。
7.构造root AgentTask。
8.构造RunContext：
-userId来自Session。
-sessionId来自Session。
-roles来自Session。
-permissions来自Session。
9.调用SupervisorEngine.execute。
10.不得直接调用ReactAgentEngine。
11.不得直接调用模型或工具。
12.不得自行实现Supervisor循环。
13.子Agent身份继承沿用现有SupervisorDispatchNode逻辑。
14.不得返回sessionId。
15.保持无状态和线程安全。
16.AgentResult为空时转换为INTERNAL_ERROR。

十四、正式Agent Controller

在agent-api实现：

AuthenticatedAgentController

接口：

POST /api/agent/invoke

请求：

{
"agentName":"utility_agent",
"message":"请查询当前时间"
}

要求：

1.必须由SessionAuthenticationInterceptor保护。
2.通过@RequestAttribute取得已验证UserSession。
3.Controller只依赖AuthenticatedAgentApplicationService。
4.不得注入SessionStore。
5.不得注入AgentRegistry。
6.不得注入ReactAgentEngine。
7.不得直接调用模型或工具。
8.不得允许Body包含：
-userId
-sessionId
-roles
-permissions
-systemPrompt
-allowedTools
-maxIterations
9.使用专门请求和响应DTO。
10.响应可包含：
-runId
-threadId
-agentName
-success
-content
-errorCode
-evidence
-metadata
11.不得返回sessionId。
12.不得返回RunContext。
13.不得返回完整消息历史。
14.无效Session在进入Controller前返回401。
15.模型不可用时按现有错误映射返回503或明确模型错误。

十五、正式Supervisor Controller

在agent-api实现：

AuthenticatedSupervisorController

接口：

POST /api/supervisor/invoke

请求：

{
"supervisorName":"general_supervisor",
"message":"请计算12*(3+5)"
}

要求：

1.必须由SessionAuthenticationInterceptor保护。
2.从request attribute读取UserSession。
3.Controller只依赖AuthenticatedSupervisorApplicationService。
4.不得注入SupervisorRegistry。
5.不得注入SupervisorEngine。
6.不得注入ReactAgentEngine。
7.不得直接调用模型或工具。
8.不得允许客户端提交memberAgents或身份字段。
9.使用专门请求和响应DTO。
10.不得返回sessionId、RunContext或内部State。
11.无效Session在进入Controller前返回401。
12.未知Supervisor使用SUPERVISOR_NOT_FOUND。
13.模型不可用时使用现有明确错误响应。

十六、RunContext身份传递

必须检查并确认：

1.正式Agent调用的RunContext.userId来自Session。
2.RunContext.sessionId来自Session。
3.roles来自Session。
4.permissions来自Session。
5.Supervisor父RunContext来自Session。
6.SupervisorDispatchNode创建子RunContext时继承：
-userId
-sessionId
-roles
-permissions
7.子Agent使用独立childRunId和childThreadId。
8.客户端无法控制任何身份字段。
9.ReasonNode和SupervisorReasonNode不得把RunContext转成消息。
10.ModelRequest不得包含roles、permissions或sessionId。
11.ToolInvocation继续携带RunContext，供第3批ACL使用。
12.不得修改ToolInvocationGateway绕过身份上下文。
13.不得使用ThreadLocal传递身份。

十七、Sample用户

为了Postman和后续前端演示，在agent-bootstrap提供受配置控制的Sample用户：

admin
角色：ADMIN

visitor
角色：VISITOR

要求：

1.仅在agent.sample.enabled=true时创建。
2.用户名稳定：
-admin
-visitor
3.密码不得以明文硬编码进Java代码。
4.密码通过环境变量或安全配置提供：

AGENT_SAMPLE_ADMIN_PASSWORD
AGENT_SAMPLE_VISITOR_PASSWORD

5.建议配置映射：

agent.sample.security.admin-password
agent.sample.security.visitor-password

6.默认值为空。
7.密码为空时：
-应用仍能启动。
-对应Sample用户不创建。
-输出不包含密码的安全警告。
8.使用CredentialHasher.hash后保存。
9.UserAccount只保存credentialHash。
10.admin绑定ADMIN角色。
11.visitor绑定VISITOR角色。
12.保存前确认角色已存在。
13.不得为Sample用户创建固定Session。
14.不得打印密码、哈希或sessionId。
15.不得静默覆盖同名非Sample用户。
16.如果用户已存在：
-相同userId和角色可幂等跳过。
-冲突时明确失败或安全告警，不得覆盖。
17.采用现有启动注册模式。
18.允许使用ApplicationRunner或专用Seeder，但必须保证角色先完成注册。
19.不得在Spring Boot主启动类直接编写初始化逻辑。
20.agent.sample.enabled=false时不创建Sample用户。

十八、Spring Bean装配

在agent-infrastructure或现有配置中装配：

Clock
SessionValidationService
AuthenticationApplicationService
AuthenticatedAgentApplicationService
AuthenticatedSupervisorApplicationService

要求：

1.使用@Configuration和@Bean。
2.application层实现保持纯Java。
3.使用Clock.systemUTC()作为默认Clock。
4.允许用户通过自定义Clock Bean替换。
5.不得创建重复Store。
6.复用已有CredentialHasher。
7.复用已有SessionIdGenerator。
8.复用已有RunIdGenerator。
9.复用已有ReactAgentEngine和SupervisorEngine。
10.ReactAgentEngine不存在时：
-登录相关Bean仍可创建。
-AuthenticatedAgentApplicationService可以条件装配。
-Agent正式Controller可以条件装配。
11.SupervisorEngine不存在时：
-登录相关功能仍可用。
-Supervisor正式入口可以条件装配。
12.不得创建Fake Engine。
13.不得产生Bean循环依赖。
14.无模型配置时应用仍能启动和登录。
15.不得在bootstrap主类中手工new应用服务。

十九、统一异常和HTTP状态码

检查现有异常映射，最小补充：

HTTP 400：
-请求字段为空。
-参数格式错误。

HTTP 401：
-AUTHENTICATION_FAILED。
-INVALID_CREDENTIALS。
-SESSION_NOT_FOUND。
-SESSION_INVALID。
-SESSION_EXPIRED。
-USER_DISABLED发生在认证入口时。

HTTP 403：
-PERMISSION_DENIED。
本批暂不主动产生工具权限拒绝，但先保证映射存在。

HTTP 404：
-AGENT_NOT_FOUND。
-SUPERVISOR_NOT_FOUND。

HTTP 503：
-MODEL_INVOCATION_FAILED或模型能力未装配。

HTTP 500：
-INTERNAL_ERROR。

要求：

1.不存在Session和伪造Session对客户端统一表现为401。
2.错误响应不得包含：
-sessionId
-password
-credentialHash
-堆栈
-内部类名
-完整请求头
3.登录失败不得暴露用户是否存在。
4.不得直接返回Spring异常对象。
5.不得把Servlet异常页面返回为HTML，继续使用统一JSON错误格式。
6.不要重复建立第二套异常响应体系。

二十、日志和安全

允许记录：

1.登录成功或失败事件。
2.userId或username的脱敏标识。
3.runId。
4.threadId。
5.agentName。
6.supervisorName。
7.认证失败类型。
8.最终执行状态。

禁止记录：

1.明文密码。
2.credentialHash。
3.sessionId。
4.X-Session-Id请求头。
5.完整UserSession。
6.完整roles和permissions集合。
7.完整Prompt。
8.完整消息历史。
9.API Key。
10.完整工具参数。

要求：

1.登录失败日志不得帮助攻击者枚举用户。
2.客户端错误中不得包含安全内部判断细节。
3.不得使用ThreadLocal保存登录用户。
4.不得使用static全局当前用户。
5.不得在MDC中保存sessionId。
6.可在MDC中保存runId，但必须遵循现有实现。

二十一、开发API边界

现有：

POST /api/dev/react/invoke
POST /api/dev/supervisor/invoke

要求：

1.继续只受agent.dev-api.enabled控制。
2.保持原有dev-user身份。
3.不得把开发API改造成正式认证接口。
4.不得让开发API自动读取X-Session-Id。
5.不得删除开发API，避免破坏前序验收。
6.明确保证正式环境可通过agent.dev-api.enabled=false关闭。
7.正式Agent和Supervisor入口使用新的非/dev路径。
8.后续前端只能调用正式认证接口。

二十二、Postman验收

配置：

agent.sample.enabled=true
agent.dev-api.enabled=true或false均可
agent.model.enabled根据模型测试需要设置

环境变量：

AGENT_SAMPLE_ADMIN_PASSWORD=自定义测试密码
AGENT_SAMPLE_VISITOR_PASSWORD=自定义测试密码

场景1：admin登录

POST /api/auth/login

{
"username":"admin",
"password":"配置的admin密码"
}

预期：

1.HTTP200。
2.返回sessionId。
3.roles包含ADMIN。
4.permissions包含tool:*:invoke。
5.不返回credentialHash。
6.日志不打印sessionId。

场景2：visitor登录

{
"username":"visitor",
"password":"配置的visitor密码"
}

预期：

1.HTTP200。
2.roles包含VISITOR。
3.permissions只包含第1批配置的有限工具权限。
4.不包含tool:*:invoke。

场景3：密码错误

{
"username":"admin",
"password":"wrong"
}

预期：

1.HTTP401。
2.返回统一AUTHENTICATION_FAILED或INVALID_CREDENTIALS。
3.不说明用户名存在。
4.不创建Session。

场景4：未知用户

{
"username":"not_exist",
"password":"wrong"
}

预期：

1.与密码错误相同的HTTP状态和客户端错误语义。
2.不得泄漏用户不存在。

场景5：查询当前身份

GET /api/auth/me
X-Session-Id: 登录返回值

预期：

1.HTTP200。
2.返回userId、username、roles、permissions。
3.不返回sessionId。
4.不返回credentialHash。

场景6：缺少Session访问me

GET /api/auth/me

预期：

HTTP401。

场景7：伪造Session

GET /api/auth/me
X-Session-Id: fake-session-id

预期：

1.HTTP401。
2.请求不进入Controller业务方法。
3.客户端不获知Session是否曾存在。

场景8：正式Agent调用

POST /api/agent/invoke
X-Session-Id: admin登录返回值

{
"agentName":"utility_agent",
"message":"请用一句话回复你好"
}

预期：

1.通过认证。
2.RunContext.userId来自admin Session。
3.模型可用时返回AgentResult。
4.客户端不能提交userId或permissions。
5.响应不返回sessionId。

场景9：正式Supervisor调用

POST /api/supervisor/invoke
X-Session-Id: visitor登录返回值

{
"supervisorName":"general_supervisor",
"message":"请计算12*(3+5)"
}

预期：

1.通过认证。
2.父RunContext身份来自visitor Session。
3.子Agent继承相同userId、sessionId、roles、permissions。
4.父子runId和threadId仍独立。
5.本批暂不因工具权限拒绝visitor。

场景10：请求Body伪造身份

{
"agentName":"utility_agent",
"message":"测试",
"userId":"admin",
"roles":["ADMIN"],
"permissions":["tool:*:invoke"]
}

预期：

1.未知字段被拒绝或忽略。
2.实际身份仍来自Session。
3.不得把visitor提升为ADMIN。

场景11：登出

POST /api/auth/logout
X-Session-Id: 登录返回值

预期：

1.成功。
2.再次使用该Session访问/api/auth/me返回401。
3.不得影响同一用户其他Session。

场景12：两个用户隔离

1.admin登录，得到sessionA。
2.visitor登录，得到sessionB。
3.分别调用正式Agent接口。
4.日志中的userId分别对应各自Session。
5.不得发生Session交叉读取。
6.本阶段尚未实现记忆，不要求验证记忆数据。

场景13：关闭Sample

agent.sample.enabled=false

预期：

1.admin和visitor不会自动创建。
2.应用仍可启动。
3.登录返回统一认证失败。

场景14：无模型配置

预期：

1.应用成功启动。
2.登录、登出、me接口可用。
3.正式Agent/Supervisor接口返回404或明确503。
4.不得因缺少模型Bean导致认证模块启动失败。

二十三、多用户隔离要求

本批必须保证：

1.SessionStore按sessionId返回唯一UserSession。
2.请求中的userId不能覆盖Session.userId。
3.Agent RunContext.userId来自Session。
4.Supervisor RunContext.userId来自Session。
5.父子Agent继承相同用户身份。
6.不同用户的runId和threadId独立。
7.不得使用全局当前用户。
8.不得使用ThreadLocal。
9.不得把UserSession保存进LangGraph4j State。
10.State中只保存现有RunContext。
11.不得把身份信息发送给LLM。
12.本批不实现MemoryStore隔离。
13.本批不实现CheckpointStore隔离。
14.后续记忆和Checkpoint必须以userId/threadId作为隔离键。

二十四、本批禁止实现

禁止：

1.Tool ACL拦截器。
2.根据permissions拒绝具体工具。
3.PERMISSION_DENIED回灌LLM。
4.在AgentTool内部写权限判断。
5.用户管理Controller。
6.角色管理Controller。
7.动态修改角色权限。
8.管理员创建或禁用用户接口。
9.前端登录页面。
10.前端用户管理页面。
11.Spring Security Web过滤链。
12.JWT。
13.Refresh Token。
14.Cookie Session。
15.OAuth。
16.验证码。
17.会话自动续期。
18.同用户单点登录。
19.数据库、Redis或文件持久化。
20.HITL。
21.CheckpointStore。
22.上下文裁剪。
23.长期记忆。
24.RAG、MCP和真实金山云业务。
25.SSE。
26.测试脚本。
27.README、使用说明、升级记录或验收报告。
28.git commit和git push。

二十五、代码质量

1.不使用Lombok。
2.优先record和final类。
3.使用构造器注入。
4.集合返回不可变快照。
5.不使用字段注入。
6.不通过ApplicationContext主动查Bean。
7.不吞异常。
8.不返回null表示认证失败。
9.不使用ThreadLocal。
10.不使用static当前用户。
11.Controller保持薄层。
12.ApplicationService负责用例协调。
13.SessionValidationService只负责Session真实性和有效性。
14.认证逻辑不得散落在各Controller。
15.密码校验不得散落在Controller或Store。
16.RunContext构建只使用服务器端Session。
17.不得创建无意义Facade、Delegate或Manager。
18.所有代码以实际编译和启动结果为准。

二十六、编译和启动验收

完成后执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许启动应用，至少验证：

GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents
GET /api/framework/supervisors
POST /api/auth/login
GET /api/auth/me
POST /api/auth/logout

有模型配置时继续验证：

POST /api/agent/invoke
POST /api/supervisor/invoke

检查：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core无SpringWeb、SpringSecurity、SpringAI和LangGraph4j依赖。
4.agent-application无SpringWeb、SpringAI和LangGraph4j依赖。
5.无模型配置时认证模块仍能启动。
6.登录使用CredentialHasher。
7.密码和哈希不进入日志。
8.Session ID由SessionIdGenerator生成。
9.Session保存到SessionStore。
10.缺失Session返回401。
11.伪造Session返回401。
12.无效请求不进入Engine。
13.登出后Session失效。
14.正式Agent RunContext身份来自Session。
15.正式Supervisor RunContext身份来自Session。
16.子Agent继承父身份上下文。
17.客户端不能伪造userId、roles或permissions。
18.身份信息没有发送给LLM。
19.没有实现工具ACL。
20.没有实现用户管理和前端。
21.现有开发API未被破坏。
22.git diff不存在无关修改。

二十七、最终输出

只输出：

1.新增和修改文件清单。
2.登录流程。
3.Session创建字段和权限快照来源。
4.SessionValidationService校验顺序。
5.X-Session-Id传递规范。
6.SessionAuthenticationInterceptor保护路径。
7.正式Agent接口调用流程。
8.正式Supervisor接口调用流程。
9.RunContext身份构建及父子Agent继承规则。
10.Sample admin和visitor创建方式。
11.Session TTL实际实现情况。
12.错误码与HTTP状态映射。
13.无模型配置时的启动和接口行为。
14.编译、打包和启动验证结果。
15.实际完成的Postman验证。
16.因缺少模型或密码配置未完成的验证及准确原因。
17.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE5_BATCH2_END===