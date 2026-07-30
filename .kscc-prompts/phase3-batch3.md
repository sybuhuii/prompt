你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven多模块架构和agent-core领域抽象。
2.AgentRegistry、ToolRegistry及Provider自动注册。
3.ToolInvocationGateway、参数校验、异常治理和审计拦截器链。
4.CalculatorTool、CurrentTimeTool、EchoTool、TextSearchTool。
5.ModelInvocationGateway及SpringAI模型适配。
6.ReactAgentState、ReactRouter、ReactAgentGraphFactory。
7.Reason、ToolExecution、Observe、Complete、MaxIterations、Failure节点。
8.DefaultReactAgentEngine及完整Reason→Act→Observe循环。

当前只执行第三阶段第3批：注册用于验证框架能力的Sample ReAct Agent，提供受配置控制的单Agent开发调用入口，并完成Postman联调。

本批完成后应能通过HTTP请求执行：
用户请求
→查询AgentDefinition
→DefaultReactAgentEngine
→ReasonNode调用模型
→模型返回ToolCall
→ToolInvocationGateway执行工具
→ObserveNode回灌ToolAgentMessage
→再次Reason
→模型生成最终回答
→返回AgentResult

本批不实现Supervisor、多Agent编排、登录权限、ACL、HITL、Checkpoint持久化、上下文裁剪、长期记忆、RAG或真实金山云业务。

一、执行前检查
1.先完整检查全部pom.xml及以下现有代码：
-AgentDefinition、AgentProvider、AgentRegistry
-AgentTask、AgentResult、RunContext
-ReactAgentEngine、DefaultReactAgentEngine
-ReactAgentState、ReactStateKeys、ReactStateSchema或Channel配置
-ReactAgentGraphFactory、ReactRouter
-全部ReAct节点
-RunIdGenerator
-现有FrameworkQueryService
-现有GET /api/framework/agents
-现有agent.dev-api配置和模型开发接口
-SpringAI和ReAct相关Spring配置
2.以当前实际接口、包结构、错误码、Java版本、SpringAI版本和LangGraph4j版本为准增量实现。
3.不得创建职责相同但名称不同的第二套AgentRegistry、ReactAgentEngine、AgentTask、RunContext或AgentResult。
4.发现第三阶段第2批存在阻断问题时只做最小修复，并在最终输出中说明。
5.不得大规模重构已经完成的模型网关、工具网关和ReAct引擎。
6.不要修改与本批无关的功能。

二、继续遵循最新架构
模块职责必须保持：

agent-core：
-稳定领域模型和接口
-不得依赖SpringAI、LangGraph4j或Spring

agent-runtime：
-ReAct运行规则和LangGraph4j编排
-不得依赖SpringAI、SpringWeb或agent-infrastructure

agent-application：
-组织单Agent执行用例
-不得依赖SpringAI或SpringWeb

agent-infrastructure：
-SpringAI适配
-SpringBean装配
-不得存放Sample Agent业务定义

agent-api：
-HTTP DTO、Controller和异常响应
-不得直接操作LangGraph4j、ChatModel、AgentTool

agent-bootstrap：
-启动、装配和可关闭的Sample配置
-允许存放少量仅用于演示框架能力的Sample Agent定义
-不得实现ReAct、模型调用或工具执行逻辑

三、LangGraph4j原生API规则
1.React节点继续使用当前LangGraph4j原生NodeAction或AsyncNodeAction。
2.同步节点通过node_async注册。
3.异步节点直接注册。
4.同步路由通过edge_async注册。
5.异步路由直接注册。
6.ReactAgentGraphFactory中禁止重新出现：
-wrapNode(Object)
-wrapRouter(Object)
-instanceof节点分派
-反射节点分派
-自定义node_async等价包装
7.本批不得改变已有ReAct图结构，除非发现明确阻断执行的问题。
8.不得让Controller或ApplicationService直接操作CompiledGraph。

四、ReactAgentState检查
1.ReactAgentState继续继承LangGraph4j AgentState。
2.不得在子类中重复定义与父类Map相同的普通Java状态字段，避免双份状态不一致。
3.状态访问必须通过一种集中方式保证类型安全：
-ReactAgentState实例访问方法，或
-现有ReactStateKeys类型安全访问方法
4.若当前已有一种完整方案，继续复用，不要再建立第二套访问层。
5.禁止节点中散落字符串key和未经检查的强制类型转换。
6.本批不重新设计State和Reducer，只检查现有实现能否正确完成端到端运行。

五、实现Sample Agent定义
在agent-bootstrap中创建或复用仅用于框架演示的包，例如：

com.ksyun.agent.bootstrap.sample

实现：
SampleAgentProvider implements AgentProvider

该Provider只返回AgentDefinition，不实现Agent运行逻辑。

至少提供2个Sample Agent：

1.utility_agent

建议定义：
name=utility_agent

description：
通用工具型Agent，可根据请求使用计算、时间、回显和文本搜索工具。

systemPrompt要求：
-根据用户任务判断是否需要使用提供的工具。
-需要工具时返回正确ToolCall，不得伪造工具执行结果。
-收到ToolAgentMessage后，根据真实工具结果继续推理。
-工具失败时根据失败观察调整策略或向用户解释。
-信息足够时停止调用工具并返回最终回答。
-不要在没有工具结果时声称工具已经执行。

allowedTools：
calculator
current_time
echo
text_search

maxIterations：
建议6至8，选择合理值并说明。

2.calculator_agent

建议定义：
name=calculator_agent

description：
只负责需要算术计算的任务。

systemPrompt要求：
-涉及算术表达式时必须优先使用calculator工具。
-不得自行编造计算结果。
-收到工具结果后再生成最终回答。
-无法完成非计算任务时明确说明职责范围。

allowedTools：
calculator

maxIterations：
建议4。

要求：
1.Agent名称稳定、小写snake_case。
2.allowedTools使用不可变Set。
3.systemPrompt不得包含API Key、权限信息或用户身份。
4.Sample Agent不添加@Component。
5.Sample Agent不直接依赖ModelInvocationGateway、ToolInvocationGateway或SpringAI。
6.Sample Agent不实现execute方法，不创建自定义Agent类绕过ReactAgentEngine。
7.Sample Agent只通过AgentProvider提供定义。
8.provideAgents()返回不可变集合。
9.不得在Provider中直接调用AgentRegistry.register。
10.不得创建Supervisor或多Agent关系。
11.Sample Agent只用于验证框架能力，不包含真实金山云业务。

六、Sample Agent Spring装配
在agent-bootstrap中创建配置类，例如：

SampleAgentConfiguration

要求：
1.使用@Configuration和@Bean创建SampleAgentProvider。
2.通过已有AgentProviderRegistrar自动注册。
3.不得在启动类中手工调用AgentRegistry.register。
4.配置受以下属性控制：
agent.sample.enabled
5.建议matchIfMissing=true，便于当前开发阶段启动后直接看到Sample Agent。
6.设置agent.sample.enabled=false时，不注册Sample Agent。
7.不得重复创建AgentRegistry或AgentProviderRegistrar。
8.不得产生重复注册。
9.无模型配置时Sample Agent仍可注册，应用仍可启动。
10.GET /api/framework/agents应能看到Sample Agent基础元信息。
11.接口不得暴露完整systemPrompt。
12.允许返回agentName、description、maxIterations和非敏感工具名称；如现有DTO不包含allowedTools，不强制修改。

七、实现ReactDevApplicationService
在agent-application中创建：

com.ksyun.agent.application.react.ReactDevApplicationService

该类保持纯Java，不添加@Component、Service或Autowired。

依赖：
AgentRegistry
ReactAgentEngine
RunIdGenerator

职责：
1.提供一次开发环境单Agent ReAct执行。
2.方法输入只允许：
-agentName
-message
3.不得允许调用方提交：
-userId
-sessionId
-runId
-roles
-permissions
-systemPrompt
-toolNames
-ToolDefinition
-maxIterations
4.校验agentName和message非空。
5.通过AgentRegistry.getRequired(agentName)取得AgentDefinition。
6.不得根据请求临时创建AgentDefinition。
7.生成runId。
8.生成threadId，例如：
dev-thread-{runId}
9.生成taskId，例如：
dev-task-{runId}
10.构造AgentTask：
-taskId为生成值
-agentName与AgentDefinition.name一致
-instruction使用message
-context为空且不可变
11.构造固定开发RunContext：
-userId=dev-user
-sessionId=dev-session
-threadId=生成值
-runId=生成值
-roles为空
-permissions为空
12.调用ReactAgentEngine.execute(definition,task,context)。
13.不得直接调用ModelInvocationGateway。
14.不得直接调用ToolInvocationGateway。
15.不得操作CompiledGraph。
16.不得自行实现Reason→Act→Observe循环。
17.不得修改AgentDefinition中的allowedTools或maxIterations。
18.返回包含以下信息的应用层结果：
-runId
-threadId
-agentName
-AgentResult
19.AgentResult为null时转换为INTERNAL_ERROR。
20.保持无状态和线程安全。

可定义不可变应用层返回对象：
ReactDevRunResult

建议字段：
String runId
String threadId
String agentName
AgentResult result

不得把LangGraph4j或SpringAI类型放入返回对象。

八、ReactDevApplicationService装配
在agent-infrastructure现有配置体系中增量装配ReactDevApplicationService。

要求：
1.通过@Configuration和@Bean创建。
2.依赖已有AgentRegistry、ReactAgentEngine、RunIdGenerator。
3.只在ReactAgentEngine Bean存在时创建。
4.不得创建假的ReactAgentEngine。
5.不得在bootstrap启动类手工new。
6.不得产生Bean循环依赖。
7.无模型配置导致ReactAgentEngine不存在时，应用仍可正常启动。
8.不得重复创建第二套RunIdGenerator。

九、开发验证API
在agent-api中实现受配置控制的接口：

POST /api/dev/react/invoke

仅当：
agent.dev-api.enabled=true
且
ReactDevApplicationService Bean存在
时启用。

建议Controller名称：
ReactDevController

建议请求DTO：
{
"agentName":"utility_agent",
"message":"请使用calculator计算12*(3+5)，并根据工具结果回答"
}

响应DTO建议：
{
"runId":"...",
"threadId":"...",
"agentName":"utility_agent",
"success":true,
"content":"96",
"errorCode":null,
"evidence":[],
"metadata":{
"iteration":2,
"toolExecutionCount":1,
"stopReason":"MODEL_COMPLETED"
}
}

实际字段以当前AgentResult模型为准，不得为了匹配示例重复定义AgentResult。

要求：
1.Controller只依赖ReactDevApplicationService。
2.Controller不得注入：
-ChatModel
-ChatClient
-ModelClient
-ModelInvocationGateway
-ToolInvocationGateway
-AgentTool
-ToolRegistry
-CompiledGraph
3.Controller不得自行构造AgentDefinition。
4.Controller不得允许客户端传入RunContext、安全身份、systemPrompt、allowedTools或maxIterations。
5.使用请求和响应DTO，不直接暴露领域对象中的内部实现细节。
6.不得返回完整systemPrompt。
7.不得返回SpringAI或LangGraph4j对象。
8.未知Agent返回项目统一的AGENT_NOT_FOUND错误。
9.参数错误返回HTTP400或现有统一异常格式。
10.执行失败按现有错误映射返回合理状态码。
11.模型不可用时：
-如果Controller因条件装配不存在，返回404可以接受；
-如果Controller存在但执行能力不可用，返回503或统一MODEL_INVOCATION_FAILED；
-必须选择一种明确、稳定的方式并说明。
12.agent.dev-api.enabled=false时接口不可访问。
13.不得新增生产聊天接口。
14.不得新增直接执行任意工具的公共接口。
15.不得实现SSE或流式输出。

十、现有Agent查询接口
检查：
GET /api/framework/agents

要求：
1.agent.sample.enabled=true时至少返回：
utility_agent
calculator_agent
2.不得重复。
3.不得暴露完整systemPrompt。
4.不得暴露Java实现类名。
5.不得暴露RunContext、安全身份或模型密钥。
6.agent.sample.enabled=false时不返回Sample Agent。
7.不得为本批重写整套查询API。

十一、完整ReAct执行要求
通过开发API执行时必须使用现有完整链路：

ReactDevController
→ReactDevApplicationService
→ReactAgentEngine
→LangGraph4j ReAct图
→ReasonNode
→ModelInvocationGateway
→SpringAiModelClient
→模型返回ToolCall
→ReactRouter
→ToolExecutionNode
→ToolInvocationGateway
→拦截器链
→AgentTool
→ToolResult
→ObserveNode
→ToolAgentMessage
→ReasonNode
→模型最终回答
→CompleteNode
→AgentResult

强制禁止以下捷径：
1.Controller直接调用模型。
2.ApplicationService直接调用模型。
3.Controller直接执行工具。
4.ApplicationService直接执行工具。
5.Sample Agent自己调用工具。
6.使用SpringAI自动工具执行。
7.模型返回ToolCall后在Controller中手工循环。
8.不经过LangGraph4j图直接拼接ModelGateway和ToolGateway。
9.为了Postman成功返回固定答案。
10.创建Fake ModelClient或Fake Tool。
11.绕过AgentDefinition.allowedTools暴露全部工具。

十二、Agent工具隔离
必须验证AgentDefinition.allowedTools生效。

要求：
1.utility_agent只能看到：
calculator
current_time
echo
text_search
2.calculator_agent只能看到：
calculator
3.ReasonNode不得因为ToolRegistry中存在其他工具就自动全部暴露。
4.Controller请求不得覆盖allowedTools。
5.工具白名单只负责决定模型可看到哪些工具；未来ACL仍将在ToolInvocationGateway执行。
6.本批不实现角色权限。

十三、AgentResult元数据
检查CompleteNode、MaxIterationsNode、FailureNode现有结果。

成功结果metadata建议至少包含：
iteration
toolExecutionCount
stopReason

失败结果metadata建议包含非敏感信息：
iteration
stopReason
agentName

要求：
1.使用不可变Map。
2.不得加入完整messages。
3.不得加入完整Prompt。
4.不得加入工具完整参数或返回内容。
5.不得加入异常对象。
6.不得加入ChatResponse、CompiledGraph等底层对象。
7.若第二批已正确实现，不重复修改。
8.若缺少本批Postman验证所必需的最小字段，可做最小补充并说明。

十四、日志和可观察性
本批只使用现有结构化日志，不引入新的监控平台。

至少应能从日志观察：
runId
threadId
agentName
Reason调用次数
模型调用开始和结果
ToolCall名称
Tool执行成功或失败
最终stopReason
最终success

要求：
1.不得记录完整Prompt。
2.不得记录完整messages。
3.不得记录完整工具参数。
4.不得记录完整工具结果。
5.不得记录API Key。
6.不得记录用户权限全集。
7.不得新建数据库审计。
8.不得引入Kafka、Prometheus或OpenTelemetry。

十五、配置
复用现有：
agent.dev-api.enabled
agent.model.enabled

新增：
agent.sample.enabled

要求：
1.不要写真实API Key。
2.模型配置继续通过环境变量提供。
3.无模型配置时应用仍能启动。
4.无模型配置时以下接口仍应可用：
GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents
5.不要创建或提交application-local.yml。
6.不要创建包含真实密钥的.env。
7.不要覆盖用户已有模型配置。
8.正式配置文件只添加安全默认值或环境变量占位符。

十六、Postman验收
模型配置可用且：
agent.dev-api.enabled=true
agent.sample.enabled=true
时，使用Postman验证。

场景1：查询Sample Agent
GET /api/framework/agents

预期至少包含：
utility_agent
calculator_agent

不得包含完整systemPrompt。

场景2：普通文本回答
POST /api/dev/react/invoke
Content-Type:application/json

{
"agentName":"utility_agent",
"message":"你好，请用一句话回复"
}

预期：
-success=true
-content有最终回答
-不调用工具或toolExecutionCount=0
-stopReason=MODEL_COMPLETED

场景3：Calculator完整ReAct
{
"agentName":"utility_agent",
"message":"请使用calculator计算12*(3+5)，必须根据工具结果回答"
}

预期：
1.模型第一次返回calculator ToolCall。
2.ToolExecutionNode通过ToolInvocationGateway执行。
3.ObserveNode生成ToolAgentMessage。
4.模型第二次根据结果生成最终回答。
5.success=true。
6.content包含96。
7.iteration通常至少为2。
8.toolExecutionCount至少为1。
9.stopReason=MODEL_COMPLETED。
10.日志中可看到ToolInvocationGateway执行记录。

场景4：CalculatorAgent
{
"agentName":"calculator_agent",
"message":"请使用calculator计算-2.5+4"
}

预期：
-success=true
-content包含1.5
-只向模型暴露calculator

场景5：工具失败回灌
{
"agentName":"calculator_agent",
"message":"请使用calculator计算10/0，并根据工具返回结果向我解释"
}

预期：
1.calculator返回INVALID_ARGUMENT。
2.ReAct不应因为普通ToolResult失败直接进入FailureNode。
3.ObserveNode仍生成error=true的ToolAgentMessage。
4.模型根据失败结果生成最终说明。
5.最终可能success=true，因为Agent成功解释工具失败。
6.日志中能看到工具失败审计。
7.客户端不能看到堆栈。

场景6：时间工具
{
"agentName":"utility_agent",
"message":"请使用current_time查询Asia/Shanghai当前时间"
}

预期：
-模型调用current_time。
-最终回答基于真实工具结果。
-不得由模型伪造时间。

场景7：文本搜索
{
"agentName":"utility_agent",
"message":"请使用text_search在以下文本中查找Java：Java is useful\nPython is useful\nJava Agent Framework"
}

预期：
-调用text_search。
-最终回答包含匹配内容或匹配行信息。

场景8：未知Agent
{
"agentName":"not_exist",
"message":"测试"
}

预期：
-返回AGENT_NOT_FOUND。
-不得调用模型。
-不得执行工具。

场景9：参数错误
{
"agentName":"",
"message":""
}

预期：
-HTTP400或统一INVALID_ARGUMENT响应。

场景10：关闭开发API
agent.dev-api.enabled=false

预期：
POST /api/dev/react/invoke不可访问。

场景11：关闭Sample
agent.sample.enabled=false

预期：
GET /api/framework/agents不包含Sample Agent。
调用utility_agent返回AGENT_NOT_FOUND。

场景12：无模型配置
预期：
-应用仍能启动。
-health/tools/agents正常。
-react开发接口按选择的条件装配策略返回404或明确503。
-不得出现NullPointerException。

十七、MaxIterations验证
真实模型行为不完全确定，不要为强制触发MaxIterations创建伪模型、固定循环Agent或隐藏测试后门。

要求：
1.通过代码审查确认maxIterations逻辑仍正确。
2.确认达到上限后不执行新ToolCall。
3.确认最后一次允许的Reason正常完成时可COMPLETE。
4.不要新增loop_agent。
5.不要新增仅用于制造死循环的Tool。
6.不要为了Postman测试破坏真实执行路径。

十八、线程安全
1.SampleAgentProvider无可变运行状态。
2.ReactDevApplicationService无某次请求状态字段。
3.每次请求生成独立runId、threadId和AgentTask。
4.不得使用ThreadLocal保存ReactAgentState。
5.不得使用static可变State。
6.同一ReactAgentEngine实例应支持不同runId并发执行。
7.CompiledGraph生命周期沿用第二批已确定方案。
8.不得通过全局锁串行所有请求。

十九、本批禁止实现
禁止：
1.Supervisor。
2.AgentDispatcher。
3.多Agent调度。
4.父图/子图多Agent编排。
5.登录、Session和RBAC。
6.Tool ACL。
7.HITL、interrupt/resume。
8.CheckpointStore内存实现。
9.上下文裁剪、TokenCounter和摘要。
10.长期记忆。
11.RAG。
12.MCP。
13.真实金山云业务。
14.SSE和流式输出。
15.并行工具执行。
16.模型重试、熔断和多模型路由。
17.前端页面。
18.动态插件市场。
19.测试脚本。
20.README、使用说明、升级记录、验收报告。
21.git commit和git push。

二十、代码质量
1.不使用Lombok。
2.使用构造器注入。
3.集合使用不可变快照。
4.不使用字段注入。
5.不通过ApplicationContext主动查找Bean。
6.不吞异常。
7.不返回null。
8.不通过Object、instanceof或反射进行节点分派。
9.不创建无意义Facade、Delegate、Manager或Adapter。
10.不重复实现ModelGateway、ToolGateway和ReactEngine已有逻辑。
11.Controller保持薄层。
12.ApplicationService只负责用例协调。
13.Sample Agent只提供定义。
14.所有代码以实际编译和运行结果为准。

二十一、编译和启动验收
完成后执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许，分别验证：

1.无模型配置启动：
-应用成功启动。
-health/tools/agents接口正常。
-Sample Agent可查询。

2.模型配置可用时启动：
-POST /api/dev/react/invoke可访问。
-普通回答场景成功。
-calculator完整ReAct场景成功。
-工具失败回灌场景成功。

检查：
1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core无SpringAI、LangGraph4j和Spring依赖。
4.agent-runtime无SpringAI、SpringWeb和agent-infrastructure依赖。
5.agent-application无SpringAI和SpringWeb依赖。
6.Controller未直接调用模型或工具。
7.Sample Agent未直接调用模型或工具。
8.AgentProvider自动注册成功。
9.工具白名单生效。
10.SpringAI未自动执行工具。
11.完整工具执行经过ToolInvocationGateway。
12.ReactAgentGraphFactory没有wrapNode(Object)。
13.没有实现本批范围外功能。
14.git diff无无关修改。

二十二、最终输出
只输出：
1.新增和修改文件清单。
2.Sample Agent定义和allowedTools。
3.Sample Agent自动注册流程。
4.ReactDevApplicationService执行流程。
5.React开发API请求和响应格式。
6.完整ReAct调用链。
7.工具白名单如何生效。
8.工具失败如何回灌模型。
9.条件装配和无模型启动行为。
10.编译、打包和启动结果。
11.已实际完成的Postman验证结果。
12.因缺少模型配置而无法验证的场景及准确原因。
13.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE3_BATCH3_END===