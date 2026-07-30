你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven模块化单体架构。
2.agent-core领域模型及稳定接口。
3.AgentRegistry、ToolRegistry、SupervisorRegistry及Provider自动注册。
4.ToolInvocationGateway和工具拦截器链。
5.CalculatorTool、CurrentTimeTool、EchoTool、TextSearchTool。
6.ModelInvocationGateway及SpringAI适配。
7.通用ReAct状态图和完整Reason→Act→Observe循环。
8.utility_agent、calculator_agent及单Agent开发验证API。
9.SupervisorDefinition、SupervisorAgentState、Router及父图。
10.SupervisorReason、Dispatch、Aggregate、Complete、MaxIterations、Failure节点。
11.DefaultSupervisorEngine及完整Supervisor中心调度循环。

当前只执行第四阶段第3批：注册Sample Supervisor，提供Supervisor查询能力和受配置控制的多Agent开发调用API，并完成Postman联调。

本批完成后的完整链路：

HTTP请求
→SupervisorDevApplicationService
→SupervisorRegistry查找SupervisorDefinition
→SupervisorEngine
→LangGraph4j Supervisor父图
→SupervisorReasonNode调用模型
→模型返回DISPATCH决策
→DispatchNode调用ReactAgentEngine
→专业子Agent执行ReAct
→子Agent返回AgentResult
→AggregateNode回灌观察消息
→Supervisor再次决策
→FINISH
→返回最终AgentResult

本批不实现登录权限、Tool ACL、HITL、Checkpoint持久化、上下文裁剪、长期记忆、RAG、MCP、真实金山云业务或并行Agent调度。

一、执行前检查

先完整检查：

1.全部pom.xml。
2.SupervisorDefinition、SupervisorProvider。
3.SupervisorRegistry、DefaultSupervisorRegistry、SupervisorProviderRegistrar。
4.SupervisorEngine、DefaultSupervisorEngine。
5.SupervisorAgentState、StateKeys、StateSchema。
6.全部Supervisor节点、Router、GraphFactory。
7.AgentRegistry、ReactAgentEngine。
8.SampleAgentProvider及Sample Agent配置。
9.FrameworkQueryService和现有framework Controller。
10.ReactDevApplicationService和React开发API。
11.RunIdGenerator、RunContext、AgentTask、AgentResult。
12.agent.dev-api.enabled、agent.sample.enabled、agent.model.enabled现有配置。
13.现有SpringBean条件装配。

要求：

1.以当前实际接口、包结构、错误码、Java版本、SpringAI版本和LangGraph4j版本为准增量实现。
2.不得创建第二套SupervisorRegistry、SupervisorEngine、AgentTask、AgentResult或RunContext。
3.发现第2批阻断问题时只做最小修复并说明。
4.不得大规模重构现有Supervisor和ReAct运行时。
5.不要修改无关代码。
6.不要生成测试脚本。
7.不要生成或修改README、使用说明、升级记录、验收报告等文档。
8.不要执行git commit或git push。

二、架构边界

继续保持：

agent-core：
-领域模型和稳定SPI。
-不得依赖Spring、SpringAI或LangGraph4j。

agent-runtime：
-ReAct和Supervisor运行规则。
-允许依赖LangGraph4j。
-不得依赖SpringAI、SpringWeb或agent-infrastructure。

agent-application：
-组织Supervisor开发调用用例。
-不得依赖SpringAI、SpringWeb或LangGraph4j。

agent-infrastructure：
-SpringBean装配。
-不得存放Sample Supervisor业务定义。

agent-api：
-Controller、请求响应DTO和异常映射。
-不得直接调用模型、工具、子Agent或CompiledGraph。

agent-bootstrap：
-启动装配和Sample定义。
-允许存放仅用于验证框架的Sample Supervisor。
-不得实现Supervisor运行逻辑。

三、LangGraph4j原生API规则

检查现有SupervisorGraphFactory并确保：

1.同步节点实现NodeAction<SupervisorAgentState>。
2.同步节点通过node_async注册。
3.同步Router通过edge_async注册。
4.异步节点实现AsyncNodeAction并直接注册。
5.异步Router实现AsyncEdgeAction并直接注册。

禁止：

-wrapNode(Object)
-wrapRouter(Object)
-Object节点参数
-instanceof节点分派
-反射节点分派
-自定义node_async或edge_async等价包装
-Controller或ApplicationService直接操作CompiledGraph

本批不得改变现有父图结构，除非存在明确阻断执行的问题。

四、注册Sample Supervisor

在agent-bootstrap现有sample包中创建或复用：

SampleSupervisorProvider implements SupervisorProvider

至少注册一个Sample Supervisor：

name：
general_supervisor

description：
通用多Agent调度Supervisor，可根据用户任务选择计算Agent或通用工具Agent，并汇总子Agent结果。

memberAgents：
utility_agent
calculator_agent

maxIterations：
建议4至6，选择合理值并说明。

systemPrompt要求：

1.负责分析总任务、选择成员Agent、生成明确子任务和汇总最终答案。
2.计算类任务优先选择calculator_agent。
3.时间、文本搜索、回显或综合工具任务可选择utility_agent。
4.复杂任务允许分派多个子任务，但本框架当前按顺序执行。
5.不得自行执行专业工具。
6.不得伪造子Agent执行结果。
7.收到子Agent观察结果后，判断是否需要继续分派。
8.信息充分时返回FINISH。
9.必须遵守现有Supervisor严格JSON决策协议。
10.不得输出Markdown、代码围栏或JSON以外内容。
11.不得输出详细思维链，只输出简洁decisionSummary。
12.不得选择memberAgents之外的Agent。
13.不得把子Agent当作SpringAI ToolCall。
14.不得包含API Key、用户权限、RunContext或真实业务敏感信息。

SampleSupervisorProvider要求：

1.只提供SupervisorDefinition。
2.不得直接调用SupervisorRegistry.register。
3.不得依赖SupervisorEngine、ModelInvocationGateway或ReactAgentEngine。
4.provideSupervisors()返回不可变集合。
5.不得每次调用重新创建可变定义。
6.不得添加@Component或Service。
7.不得包含真实金山云业务逻辑。

五、Sample Supervisor配置

在agent-bootstrap中创建或完善：

SampleSupervisorConfiguration

要求：

1.使用@Configuration和@Bean创建SampleSupervisorProvider。
2.通过已有SupervisorProviderRegistrar自动注册。
3.不得在启动类手工调用register。
4.受配置控制：

agent.sample.enabled=true

5.复用现有agent.sample.enabled，不新增职责重复的开关。
6.agent.sample.enabled=false时：
-general_supervisor不注册。
-utility_agent和calculator_agent也继续遵循现有Sample开关。
7.不得重复创建SupervisorRegistry或Registrar。
8.无模型配置时Sample Supervisor仍可注册，应用仍能启动。
9.不得产生重复注册异常。
10.不得创建第二套Sample注册机制。

六、扩展FrameworkQueryService

检查现有FrameworkQueryService。

增量增加：

Collection<SupervisorDefinition> listSupervisors();

要求：

1.通过SupervisorRegistry查询。
2.不直接访问Spring容器。
3.保持现有agents和tools查询行为不变。
4.不得在Application层依赖具体DefaultSupervisorRegistry实现。
5.不得修改Supervisor定义。
6.返回不可变快照或依赖Registry已有不可变返回。

七、Supervisor查询API

在现有framework Controller中增量提供：

GET /api/framework/supervisors

或按照现有Controller职责拆分一个清晰的查询Controller。

响应至少可包含：

name
description
memberAgents
maxIterations

要求：

1.不得暴露完整systemPrompt。
2.不得暴露Java实现类名。
3.不得暴露模型配置。
4.不得暴露RunContext、用户权限、API Key或内部State。
5.memberAgents可以返回非敏感Agent名称。
6.使用专门响应DTO。
7.不得直接返回SupervisorDefinition领域对象。
8.agent.sample.enabled=true时至少包含general_supervisor。
9.agent.sample.enabled=false时不包含Sample Supervisor。
10.没有Supervisor时返回空集合，不返回500。
11.不得破坏现有：
GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents

八、实现SupervisorDevApplicationService

在agent-application中创建：

com.ksyun.agent.application.supervisor.SupervisorDevApplicationService

保持纯Java，不添加@Component、Service或Autowired。

依赖：

SupervisorRegistry
SupervisorEngine
RunIdGenerator

职责：

1.提供开发环境下一次Supervisor多Agent执行。
2.输入只允许：
-supervisorName
-message

3.禁止调用方提交：
-userId
-sessionId
-threadId
-runId
-roles
-permissions
-systemPrompt
-memberAgents
-maxIterations
-AgentTask列表
-AgentDefinition
-SupervisorDefinition

4.校验supervisorName和message非空。
5.通过SupervisorRegistry.getRequired查询SupervisorDefinition。
6.不得根据请求临时创建SupervisorDefinition。
7.生成父runId。
8.生成父threadId，例如：
dev-supervisor-thread-{runId}
9.生成rootTaskId，例如：
dev-supervisor-task-{runId}
10.构造root AgentTask：
-taskId为生成值
-agentName=SupervisorDefinition.name
-instruction=message
-context为空且不可变
11.构造固定开发RunContext：
-userId=dev-user
-sessionId=dev-session
-threadId=生成父threadId
-runId=生成父runId
-roles为空
-permissions为空
12.调用：
SupervisorEngine.execute(definition,rootTask,runContext)
13.不得直接调用ModelInvocationGateway。
14.不得直接调用ReactAgentEngine。
15.不得直接调用ToolInvocationGateway。
16.不得自行实现Supervisor循环。
17.不得操作CompiledGraph。
18.不得修改definition.memberAgents或maxIterations。
19.保持无状态和线程安全。
20.AgentResult为null时转换为INTERNAL_ERROR。

定义不可变应用层返回对象：

SupervisorDevRunResult

建议字段：

String runId
String threadId
String supervisorName
AgentResult result

不得包含SpringAI或LangGraph4j类型。

九、SupervisorDevApplicationService装配

在agent-infrastructure现有配置中增量装配：

SupervisorDevApplicationService

要求：

1.通过@Configuration和@Bean创建。
2.依赖已有SupervisorRegistry、SupervisorEngine和RunIdGenerator。
3.仅在SupervisorEngine Bean存在时创建。
4.不得创建Fake SupervisorEngine。
5.不得在bootstrap启动类手工new。
6.不得创建第二套RunIdGenerator。
7.不得产生Bean循环依赖。
8.无模型配置、SupervisorEngine未装配时应用仍能启动。

十、开发验证API

在agent-api实现：

POST /api/dev/supervisor/invoke

仅当：

agent.dev-api.enabled=true

并且SupervisorDevApplicationService Bean存在时启用。

请求DTO建议：

{
"supervisorName":"general_supervisor",
"message":"请计算12*(3+5)，并告诉我当前Asia/Shanghai时间"
}

响应DTO建议：

{
"runId":"...",
"threadId":"...",
"supervisorName":"general_supervisor",
"success":true,
"content":"...",
"errorCode":null,
"evidence":[],
"metadata":{
"iteration":2,
"dispatchedTaskCount":2,
"agentResultCount":2,
"successfulAgentCount":2,
"failedAgentCount":0,
"stopReason":"COMPLETED"
}
}

实际字段以现有AgentResult为准，不得创建第二套AgentResult。

要求：

1.Controller只依赖SupervisorDevApplicationService。
2.Controller不得注入：
-ChatModel
-ChatClient
-ModelClient
-ModelInvocationGateway
-ReactAgentEngine
-ToolInvocationGateway
-AgentRegistry
-SupervisorRegistry
-AgentTool
-CompiledGraph
3.Controller不得自行构造SupervisorDefinition或AgentTask。
4.Controller不得允许客户端提交身份、安全上下文、memberAgents或maxIterations。
5.使用请求和响应DTO。
6.不得暴露完整systemPrompt。
7.不得暴露子Agent完整结果列表、消息历史、ToolCall参数或内部State。
8.未知Supervisor返回SUPERVISOR_NOT_FOUND。
9.参数错误返回HTTP400或现有统一INVALID_ARGUMENT响应。
10.模型不可用时返回明确503或现有MODEL_INVOCATION_FAILED映射。
11.agent.dev-api.enabled=false时接口不可访问。
12.不得新增生产多Agent聊天接口。
13.不得实现SSE或流式输出。

十一、完整调用链要求

Supervisor开发接口必须走：

SupervisorDevController
→SupervisorDevApplicationService
→SupervisorRegistry
→SupervisorEngine
→SupervisorGraph
→SupervisorReasonNode
→ModelInvocationGateway
→SupervisorDecisionParser
→SupervisorRouter
→SupervisorDispatchNode
→ReactAgentEngine
→子Agent ReAct图
→ToolInvocationGateway
→AgentTool
→AgentResult
→SupervisorAggregateNode
→再次SupervisorReasonNode
→SupervisorCompleteNode
→最终AgentResult

强制禁止捷径：

1.Controller直接调用模型。
2.ApplicationService直接调用模型。
3.Controller直接调用ReactAgentEngine。
4.ApplicationService直接调用ReactAgentEngine。
5.Controller直接调用工具。
6.Supervisor直接调用工具。
7.把子Agent注册为SpringAI ToolCallback。
8.使用SpringAI自动执行子Agent。
9.在Controller中手工循环分派Agent。
10.绕过LangGraph4j父图直接拼接Reason和Dispatch。
11.返回固定答案。
12.创建Fake ModelClient、Fake SupervisorEngine或Fake AgentResult。

十二、Supervisor成员白名单

必须验证：

1.general_supervisor只能选择：
-utility_agent
-calculator_agent
2.SupervisorReasonNode只向模型公开memberAgents中的名称和描述。
3.不得自动向Supervisor公开AgentRegistry全部Agent。
4.模型选择非成员Agent时必须拒绝。
5.客户端不得覆盖memberAgents。
6.子Agent仍只能访问各自AgentDefinition.allowedTools。
7.Supervisor不得把成员Agent当作工具定义发送给模型。

十三、Sample Agent和Sample Supervisor关系

确认：

general_supervisor
├── utility_agent
└── calculator_agent

要求：

1.general_supervisor不是AgentDefinition。
2.utility_agent和calculator_agent不是SupervisorDefinition。
3.SupervisorRegistry和AgentRegistry继续保持独立。
4.不得将Supervisor同时注册进AgentRegistry。
5.不得将子Agent注册进SupervisorRegistry。
6.不得建立一个混合Registry。
7.Supervisor通过成员Agent名称查询AgentRegistry。
8.子Agent通过ReactAgentEngine执行。

十四、结果和元数据

检查SupervisorCompleteNode现有实现。

成功metadata建议包含：

iteration
dispatchedTaskCount
agentResultCount
successfulAgentCount
failedAgentCount
stopReason

要求：

1.不得把完整agentResults放进metadata。
2.不得把Supervisor消息历史放进metadata。
3.不得把子Agent消息或工具轨迹放进metadata。
4.不得包含异常对象。
5.不得包含SpringAI或LangGraph4j对象。
6.metadata使用不可变Map。
7.如果现有实现已经满足，不重复修改。
8.若缺少Postman验证必须字段，只做最小补充并说明。

十五、日志验证

本批不引入新监控系统，只复用结构化日志。

应能观察：

parentRunId
parentThreadId
supervisorName
supervisorIteration
decisionAction
dispatchTaskCount
targetAgentName
childRunId
childThreadId
childAgentSuccess
finalStopReason
finalSuccess

要求：

1.不得记录完整Prompt。
2.不得记录完整Supervisor JSON响应。
3.不得记录完整子Agent结果。
4.不得记录工具完整参数和结果。
5.不得记录API Key。
6.不得记录roles、permissions全集。
7.不得记录异常对象到客户端。
8.不要引入Kafka、Prometheus或OpenTelemetry。

十六、配置

复用：

agent.dev-api.enabled
agent.sample.enabled
agent.model.enabled

要求：

1.不要新增职责重复开关。
2.不要写真实API Key。
3.模型配置继续通过现有环境变量提供。
4.无模型配置时应用仍能启动。
5.无模型配置时以下接口正常：
GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents
GET /api/framework/supervisors
6.不要创建application-local.yml。
7.不要覆盖用户已有模型配置。
8.不要创建包含真实密钥的.env。

十七、Postman验收

模型配置可用且：

agent.dev-api.enabled=true
agent.sample.enabled=true
agent.model.enabled=true

时验证。

场景1：查询Supervisor

GET /api/framework/supervisors

预期至少包含：

general_supervisor

并包含：

utility_agent
calculator_agent

不得包含完整systemPrompt。

场景2：单一计算任务

POST /api/dev/supervisor/invoke
Content-Type:application/json

{
"supervisorName":"general_supervisor",
"message":"请计算12*(3+5)，必须使用合适的专业Agent，并根据真实工具结果回答"
}

预期：

1.Supervisor返回DISPATCH。
2.目标优先为calculator_agent。
3.calculator_agent内部执行calculator工具。
4.返回子Agent结果。
5.AggregateNode回灌观察。
6.Supervisor再次调用模型并返回FINISH。
7.最终content包含96。
8.success=true。
9.dispatchedTaskCount至少为1。
10.agentResultCount至少为1。
11.stopReason=COMPLETED。
12.日志中可看到父runId和独立childRunId。

场景3：综合多Agent任务

{
"supervisorName":"general_supervisor",
"message":"请完成两个任务：第一，计算20/4；第二，查询Asia/Shanghai当前时间。请综合返回结果。"
}

预期：

1.Supervisor可生成两个任务，或分两轮调度。
2.计算任务由calculator_agent或utility_agent完成。
3.时间任务由utility_agent完成。
4.子Agent按顺序执行。
5.每个子任务有独立childRunId和childThreadId。
6.最终回答同时包含计算结果5和当前时间。
7.不得把子Agent内部消息直接返回客户端。

场景4：工具失败后的Supervisor处理

{
"supervisorName":"general_supervisor",
"message":"请使用专业Agent计算10/0，并说明无法完成的原因"
}

预期：

1.子Agent工具调用返回INVALID_ARGUMENT。
2.子Agent可生成解释性AgentResult。
3.即使子Agent最终success=false，也不得直接使父图崩溃。
4.AggregateNode将失败结果安全反馈给Supervisor。
5.Supervisor生成最终解释。
6.客户端不得看到堆栈。

场景5：非计算普通任务

{
"supervisorName":"general_supervisor",
"message":"请让合适的Agent用一句话介绍这个通用Agent框架"
}

预期：

1.Supervisor选择utility_agent。
2.子Agent完成后由Supervisor汇总。
3.不得由Controller直接调用utility_agent。

场景6：未知Supervisor

{
"supervisorName":"not_exist",
"message":"测试"
}

预期：

SUPERVISOR_NOT_FOUND

且：

-不调用模型。
-不执行子Agent。

场景7：空参数

{
"supervisorName":"",
"message":""
}

预期：

HTTP400或INVALID_ARGUMENT。

场景8：关闭开发API

agent.dev-api.enabled=false

预期：

POST /api/dev/supervisor/invoke不可访问。

场景9：关闭Sample

agent.sample.enabled=false

预期：

1.GET /api/framework/supervisors不包含general_supervisor。
2.调用general_supervisor返回SUPERVISOR_NOT_FOUND。
3.Sample Agent也遵循现有开关。

场景10：无模型配置

预期：

1.应用仍能启动。
2.framework查询接口正常。
3.Supervisor开发接口返回404或明确503。
4.不得出现BeanCreationException或NullPointerException。

十八、Supervisor迭代验证

真实模型行为不稳定，不得为强行触发最大次数创建Fake模型、固定循环Supervisor或隐藏后门。

通过代码审查确认：

1.iteration只在Supervisor模型调用后增加。
2.最后一次允许调用返回FINISH时可正常完成。
3.最后一次允许调用返回DISPATCH时不执行新任务。
4.子Agent ReAct iteration不影响Supervisor iteration。
5.达到上限后返回MAX_ITERATIONS_REACHED。
6.不得新增loop_supervisor。

十九、线程安全

1.SampleSupervisorProvider无可变运行状态。
2.SupervisorDevApplicationService无请求级实例字段。
3.每次请求生成独立父runId和threadId。
4.每个子任务生成独立childRunId和childThreadId。
5.不得使用ThreadLocal保存SupervisorAgentState。
6.不得使用static可变State。
7.不得跨请求复用SupervisorAgentState。
8.同一SupervisorEngine实例支持多个runId并发执行。
9.子Agent本批仍按顺序执行，不新增线程池。
10.不得通过全局锁串行所有Supervisor请求。

二十、本批禁止实现

禁止：

1.真实金山云业务。
2.全局Supervisor或两级Supervisor。
3.多个Supervisor之间路由。
4.并行子Agent。
5.线程池分派。
6.Agent并发上限。
7.think_tool。
8.Supervisor直接调用业务工具。
9.登录、Session和RBAC。
10.Tool ACL。
11.HITL、interrupt/resume。
12.CheckpointStore内存实现。
13.上下文裁剪和摘要。
14.长期记忆。
15.RAG和MCP。
16.SSE和流式输出。
17.前端页面。
18.模型重试、熔断或多模型路由。
19.动态插件市场。
20.测试脚本。
21.README、说明、升级记录、验收报告。
22.git commit和git push。

二十一、代码质量

1.不使用Lombok。
2.使用构造器注入。
3.集合和Map使用不可变快照。
4.不使用字段注入。
5.不通过ApplicationContext主动查Bean。
6.不吞异常。
7.不返回null表示失败。
8.不通过Object、instanceof或反射分派节点。
9.不创建无意义Facade、Delegate、Manager或Adapter。
10.Controller保持薄层。
11.ApplicationService只组织用例。
12.Sample Supervisor只提供定义。
13.SupervisorEngine继续复用ReactAgentEngine。
14.所有代码以实际编译和运行结果为准。
15.不得为Postman成功返回固定结果。

二十二、编译和启动验收

执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许，分别验证：

1.无模型配置启动：
-应用成功启动。
-health/tools/agents/supervisors查询接口正常。
-Sample定义可查询。

2.有模型配置启动：
-Supervisor开发API可访问。
-单一计算任务成功。
-综合多Agent任务成功。
-工具失败反馈场景可完成。

检查：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core无SpringAI、LangGraph4j和Spring依赖。
4.agent-runtime无SpringAI、SpringWeb和agent-infrastructure依赖。
5.agent-application无SpringAI、SpringWeb和LangGraph4j依赖。
6.Controller未直接调用模型、ReactAgentEngine或工具。
7.SampleSupervisorProvider未直接调用SupervisorEngine。
8.SupervisorProvider自动注册成功。
9.SupervisorRegistry和AgentRegistry保持分离。
10.general_supervisor只能选择memberAgents。
11.Supervisor未通过ToolCall调度Agent。
12.DispatchNode仍只通过ReactAgentEngine执行子Agent。
13.子Agent按顺序执行。
14.每个子Agent具有独立childRunId和childThreadId。
15.子Agent失败不会直接破坏父图。
16.AggregateNode使用UserAgentMessage回灌，不使用ToolAgentMessage。
17.SupervisorGraphFactory不存在wrapNode(Object)。
18.同步节点继续通过node_async注册。
19.现有单AgentReAct API不受影响。
20.没有实现本批范围外功能。
21.git diff不存在无关修改。

二十三、最终输出

只输出：

1.新增和修改文件清单。
2.general_supervisor定义和memberAgents。
3.Sample Supervisor自动注册流程。
4.Supervisor查询API。
5.SupervisorDevApplicationService执行流程。
6.Supervisor开发API请求和响应格式。
7.完整多Agent调用链。
8.Supervisor成员白名单如何生效。
9.父子RunContext及runId/threadId隔离。
10.子Agent失败如何反馈Supervisor。
11.条件装配和无模型启动行为。
12.编译、打包和启动结果。
13.已实际完成的Postman验证结果。
14.因缺少模型配置无法验证的场景及准确原因。
15.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE4_BATCH3_END===