你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven模块化单体架构。
2.agent-core领域模型和稳定接口。
3.AgentRegistry、ToolRegistry及Provider自动注册。
4.ToolInvocationGateway和工具拦截器链。
5.内置工具。
6.ModelInvocationGateway和SpringAI模型适配。
7.通用ReAct状态图及完整Reason→Act→Observe循环。
8.Sample ReAct Agent和开发验证API。
9.SupervisorDefinition、SupervisorRegistry、SupervisorAgentState、节点契约、Router及SupervisorGraphFactory父图骨架。

当前只执行第四阶段第2批：实现Supervisor结构化决策、子Agent顺序分派、结果聚合、终止节点和DefaultSupervisorEngine，打通完整Supervisor循环。

本批不注册Sample Supervisor，不提供Supervisor HTTP接口，不实现并行调度、权限、HITL、Checkpoint、上下文裁剪、记忆或真实金山云业务。

一、目标执行链

实现：

SupervisorEngine
→SupervisorGraph
→SupervisorReasonNode
→ModelInvocationGateway
→模型返回结构化Supervisor决策
→SupervisorDecisionParser
→SupervisorRouter
→SupervisorDispatchNode
→ReactAgentEngine执行专业子Agent
→SupervisorAggregateNode
→将子Agent结果作为观察写回Supervisor消息
→再次SupervisorReasonNode
→模型决定继续DISPATCH或FINISH
→SupervisorCompleteNode
→最终AgentResult

强制要求：

1.Supervisor只负责分析、拆分、分派和汇总。
2.Supervisor不得直接调用业务工具。
3.子Agent必须通过现有ReactAgentEngine执行。
4.子Agent内部继续使用现有LangGraph4j ReAct图。
5.子Agent不得被包装成SpringAI自动执行工具。
6.本批子Agent按顺序执行，不实现并行。
7.不实现think_tool。
8.不保存或要求模型输出隐藏推理过程，只保存简洁decisionSummary。

二、执行前检查

先完整检查全部pom.xml及现有：

1.SupervisorDefinition、SupervisorProvider。
2.SupervisorRegistry、DefaultSupervisorRegistry。
3.SupervisorDecision、SupervisorAction。
4.SupervisorAgentState、SupervisorStateKeys、SupervisorStateSchema。
5.SupervisorStopReason、SupervisorRoute。
6.全部Supervisor节点契约。
7.SupervisorRouter、SupervisorGraphFactory。
8.SupervisorEngine、SupervisorExecutionValidator。
9.AgentRegistry、AgentDefinition、AgentTask、AgentResult。
10.ReactAgentEngine。
11.ModelInvocationGateway、ModelRequest、ModelResponse。
12.AgentMessage及各消息类型。
13.RunContext、RunIdGenerator。
14.AgentErrorCode、AgentFrameworkException。
15.现有Spring装配方式。
16.当前实际LangGraph4j版本和图执行API。

要求：

1.以现有接口签名、包结构和实际依赖版本为准增量实现。
2.不得创建第二套AgentTask、AgentResult、RunContext、AgentRegistry或ModelGateway。
3.发现第1批存在阻断问题时只做最小修复，并在最终输出中说明。
4.不得大规模重构现有ReAct、工具网关和模型网关。
5.不要修改与本批无关的代码。
6.不要生成测试脚本。
7.不要生成或修改README、说明、升级记录、验收报告等文档。
8.不要执行git commit或git push。

三、架构边界

继续保持：

agent-core：
-稳定领域模型和SPI。
-不得依赖Spring、SpringAI或LangGraph4j。

agent-runtime：
-Supervisor运行规则、节点、Router和LangGraph4j图。
-可以依赖LangGraph4j。
-不得依赖SpringAI、SpringWeb或agent-infrastructure。

agent-infrastructure：
-SpringBean装配。
-模型结构化响应解析技术适配。
-不得存放具体业务Supervisor定义。

agent-application：
-本批不新增Supervisor应用服务。

agent-api：
-本批不新增Supervisor Controller。

agent-bootstrap：
-本批不注册Sample Supervisor。
-不得实现Supervisor运行逻辑。

四、LangGraph4j原生API强制规则

1.同步节点直接实现NodeAction<SupervisorAgentState>。
2.同步节点在SupervisorGraphFactory中通过LangGraph4j原生node_async(node)注册。
3.同步Router通过LangGraph4j原生edge_async(...)注册。
4.异步节点若实际存在，则实现AsyncNodeAction并直接注册。
5.异步Router若实际存在，则实现AsyncEdgeAction并直接注册。

禁止：

1.wrapNode(Object)。
2.wrapRouter(Object)。
3.Object类型节点参数。
4.instanceof节点类型分派。
5.反射节点类型分派。
6.自定义node_async或edge_async等价包装。
7.创建与NodeAction或AsyncNodeAction语义重复但不兼容的接口。
8.GraphFactory调用模型或子Agent。
9.GraphFactory主动从Spring容器查找依赖。
10.GraphFactory保存某次运行State。

五、Supervisor模型输出协议

Supervisor不得使用ToolCall调度子Agent。

Supervisor模型必须输出单个严格JSON对象。

DISPATCH示例：

{
"action":"DISPATCH",
"tasks":[
{
"agentName":"calculator_agent",
"instruction":"使用计算工具计算12*(3+5)，返回准确结果",
"context":{}
}
],
"decisionSummary":"该任务需要专业计算Agent处理",
"finalAnswer":""
}

FINISH示例：

{
"action":"FINISH",
"tasks":[],
"decisionSummary":"现有子Agent结果已经足够回答用户",
"finalAnswer":"计算结果是96"
}

要求：

1.action只能是DISPATCH或FINISH。
2.DISPATCH时tasks至少一个。
3.FINISH时tasks必须为空。
4.FINISH时finalAnswer不能为空。
5.decisionSummary只允许简洁决策摘要，不要求或保存详细思维链。
6.模型任务中不得包含taskId，taskId由框架生成。
7.模型不得提供runId、threadId、userId、sessionId、roles或permissions。
8.context只能是普通JSON对象。
9.模型不得返回ToolCall。
10.模型不得使用子Agent名称以外的目标。

六、增加非可信模型输出草稿模型

不要让模型输出直接成为可执行AgentTask。

在agent-runtime的supervisor包或明确子包中实现不可变内部模型：

SupervisorTaskDraft

建议字段：
String agentName
String instruction
Map<String,Object> context

SupervisorDecisionDraft

建议字段：
SupervisorAction action
List<SupervisorTaskDraft> tasks
String decisionSummary
String finalAnswer

要求：

1.这两个类型只表示未经完全信任和校验的模型输出。
2.不得放入agent-core。
3.集合和Map使用不可变快照。
4.null集合按空集合处理。
5.不得包含taskId、runId或安全身份。
6.不得包含SpringAI、Jackson或LangGraph4j类型。
7.经过ReasonNode校验和规范化后，才能转换为正式SupervisorDecision和AgentTask。

七、SupervisorDecisionParser接口

在agent-runtime中定义：

SupervisorDecisionParser

建议方法：

SupervisorDecisionDraft parse(String content);

职责：

1.解析模型返回的严格JSON文本。
2.不得调用模型。
3.不得访问AgentRegistry。
4.不得生成AgentTask或runId。
5.不得执行子Agent。
6.不得依赖Spring容器。

实现：

JacksonSupervisorDecisionParser

放在agent-infrastructure的明确包中，例如：

com.ksyun.agent.infrastructure.supervisor

使用项目已有Jackson ObjectMapper。

要求：

1.不得在agent-runtime直接依赖Jackson。
2.使用ObjectMapper或ObjectReader解析。
3.不要修改全局ObjectMapper行为，优先创建安全ObjectReader或ObjectMapper副本。
4.限制响应最大长度，例如65536字符。
5.允许去除一层完整的```json代码围栏，但不得从任意自然语言中正则提取JSON片段。
6.空内容返回MODEL_INVOCATION_FAILED。
7.JSON非法返回MODEL_INVOCATION_FAILED。
8.action非法返回MODEL_INVOCATION_FAILED。
9.字段类型不符返回MODEL_INVOCATION_FAILED。
10.未知字段应拒绝或采用明确严格策略，不得静默接受危险字段。
11.不得把原始响应全文放入客户端错误。
12.服务端日志可以记录响应长度和解析错误类型，但不得记录完整响应。
13.不得把Jackson对象泄漏到runtime接口。
14.保持无状态和线程安全。

八、SupervisorPromptBuilder

在agent-runtime实现纯Java：

SupervisorPromptBuilder

依赖：
AgentRegistry

职责：

1.根据SupervisorDefinition构造Supervisor系统提示词。
2.只列出memberAgents中的Agent。
3.每个成员只向Supervisor公开：
-agentName
-description
4.不得公开：
-子Agent完整systemPrompt
-子Agent权限信息
-内部实现类
-模型配置
-RunContext
5.加入严格JSON输出协议。
6.明确告诉模型：
-只允许选择memberAgents。
-需要专业能力时返回DISPATCH。
-子Agent结果足够时返回FINISH。
-不得伪造子Agent执行结果。
-不得调用工具。
-不得输出Markdown、代码围栏或JSON之外的文字。
-不得输出隐藏推理过程，只输出简洁decisionSummary。
7.保持无状态。
8.系统提示词内容不得包含具体金山云业务。
9.SupervisorDefinition.systemPrompt作为业务自定义前缀或核心指令保留。
10.生成结果不能为空。

九、DefaultSupervisorReasonNode

实现：

DefaultSupervisorReasonNode implements SupervisorReasonNode

依赖：

1.ModelInvocationGateway。
2.SupervisorDecisionParser。
3.AgentRegistry。
4.RunIdGenerator。

使用构造器注入，保持纯Java，不添加Spring注解。

执行职责：

1.读取并校验：
-SupervisorDefinition
-rootTask
-RunContext
-supervisorMessages
-iteration

2.若iteration已经达到maxIterations且尚未进行新模型调用：
-不得继续调用模型。
-设置stopReason=MAX_ITERATIONS_REACHED。
-由Router进入MAX_ITERATIONS。

3.构造ModelRequest：
-messages使用当前supervisorMessages。
-tools必须为空。
-options使用不可变空Map或现有安全默认值。
-不得把任何子Agent作为ToolDefinition发送给模型。
-不得把RunContext转成消息。
-不得自动暴露userId、sessionId、roles或permissions。

4.调用ModelInvocationGateway一次。

5.模型响应要求：
-AssistantAgentMessage不能为空。
-toolCalls必须为空。
-若模型返回ToolCall，视为MODEL_INVOCATION_FAILED。
-content不能为空。

6.将本轮AssistantAgentMessage追加到supervisorMessages。

7.调用SupervisorDecisionParser解析content。

8.对Draft进行严格校验和规范化：

DISPATCH：
-任务数量至少1。
-任务数量设置合理安全上限，例如最多10个。
-agentName必须在SupervisorDefinition.memberAgents中。
-agentName必须存在于AgentRegistry。
-instruction不能为空。
-context使用不可变Map。
-context中禁止以下保留字段：
userId
sessionId
threadId
runId
roles
permissions
systemPrompt
allowedTools
maxIterations
-每个任务由RunIdGenerator生成框架taskId。
-不得信任模型生成的任何身份或执行ID。
-转换为正式AgentTask列表。
-构造SupervisorDecision(action=DISPATCH,...)。
-覆盖pendingTasks为本轮任务。
-清空latestAgentResults。
-不得设置COMPLETED。

FINISH：
-tasks必须为空。
-finalAnswer不能为空。
-构造SupervisorDecision(action=FINISH,...)。
-pendingTasks覆盖为空。
-latestAgentResults覆盖为空。
-设置stopReason=COMPLETED。
-不得在ReasonNode直接构造最终AgentResult。

9.每成功完成一次模型调用后iteration+1。

10.异常处理：
-AgentFrameworkException保留错误码。
-模型异常使用MODEL_INVOCATION_FAILED。
-解析失败使用MODEL_INVOCATION_FAILED。
-Agent不存在使用AGENT_NOT_FOUND。
-其他异常转换为INTERNAL_ERROR。
-设置failureErrorCode、failureMessage和对应stopReason。
-不得把完整Prompt、模型响应、JSON内容、堆栈或密钥写入failureMessage。
-服务端日志记录runId、supervisorName、iteration和完整异常堆栈。
-不要捕获Error等JVM严重错误。

11.不得执行子Agent。
12.不得调用ToolInvocationGateway。
13.不得修改历史agentResults。

十、子Agent执行上下文

DefaultSupervisorDispatchNode执行每个AgentTask时，必须创建独立子RunContext。

身份继承：

1.userId继承父RunContext。
2.sessionId继承父RunContext。
3.roles继承父RunContext。
4.permissions继承父RunContext。

运行隔离：

1.每个子任务使用新的childRunId。
2.每个子任务使用独立childThreadId。
3.childThreadId可基于父threadId和taskId安全派生。
4.不得让多个子Agent共享同一个ReactAgentState。
5.不得复用父Supervisor runId作为子Agent runId。
6.不得把父Supervisor完整消息历史传给子Agent。
7.不得把其他子Agent结果直接塞入当前子Agent消息。
8.子Agent只接收：
-AgentDefinition.systemPrompt
-AgentTask.instruction
-ReactAgentEngine现有允许的任务上下文
9.不得修改父RunContext对象。
10.不得允许模型控制子RunContext中的身份字段。

十一、DefaultSupervisorDispatchNode

实现：

DefaultSupervisorDispatchNode implements SupervisorDispatchNode

依赖：

1.AgentRegistry。
2.ReactAgentEngine。
3.RunIdGenerator。

职责：

1.读取：
-SupervisorDefinition
-RunContext
-pendingTasks

2.pendingTasks为空时：
-设置failureErrorCode=INTERNAL_ERROR。
-设置stopReason=INVALID_STATE。
-设置安全failureMessage。
-不得继续执行。

3.本批按照pendingTasks顺序逐个执行，不并行。

4.对每个任务：
-确认agentName属于memberAgents。
-通过AgentRegistry.getRequired获取AgentDefinition。
-创建独立childRunId。
-创建独立childThreadId。
-创建子RunContext。
-调用ReactAgentEngine.execute(agentDefinition,agentTask,childContext)。

5.强制禁止：
-直接调用ModelInvocationGateway。
-直接调用ToolInvocationGateway。
-直接调用AgentTool。
-自己实现ReAct循环。
-把子Agent包装成SpringAI ToolCallback。
-通过ApplicationContext查找Agent。
-并行执行子Agent。

6.每个子Agent返回的AgentResult都保存到本轮latestAgentResults。

7.一个子Agent返回success=false时：
-将失败AgentResult正常保留。
-不立即终止整个Supervisor。
-由AggregateNode把失败结果反馈给Supervisor模型。
-Supervisor可决定重试、换Agent或结束。

8.调用ReactAgentEngine抛出AgentFrameworkException时：
-不要让单个子Agent异常直接破坏整个父图。
-转换为对应子Agent失败AgentResult。
-保留安全errorCode和安全content。
-不得返回堆栈。

9.发生无法归属到某个子任务的框架内部异常时：
-设置stopReason=AGENT_ERROR。
-设置failureErrorCode。
-设置安全failureMessage。
-进入Failure。

10.为每个子Agent结果补充非敏感metadata：
-parentRunId
-childRunId
-childThreadId
-taskId
不得加入userId、sessionId、roles、permissions或完整消息。

11.latestAgentResults覆盖写入本轮完整结果列表。
12.不得直接追加到历史agentResults，由AggregateNode统一追加。
13.不得修改supervisorMessages。
14.不得增加Supervisor iteration。
15.不得清空pendingTasks，AggregateNode需要配对。
16.不得返回null。

十二、SupervisorObservationFormatter

实现纯Java：

SupervisorObservationFormatter

职责：

将本轮AgentTask和AgentResult格式化为Supervisor下一轮可读取的安全观察消息。

要求：

1.任务和结果数量必须一致。
2.按顺序配对。
3.每项至少包含：
-taskId
-agentName
-success
-content
-errorCode
-有限数量evidence
4.不得包含：
-子Agent完整messages
-子AgentsystemPrompt
-ToolCall完整参数
-ToolResult完整内容之外的内部对象
-工具轨迹
-异常对象
-RunContext身份字段
-SpringAI对象
-LangGraph4j对象
5.限制单个result.content长度，例如最多4000字符。
6.限制整条观察消息总长度，例如最多12000字符。
7.发生截断时明确标记truncated=true。
8.不要实现完整上下文裁剪或LLM摘要，本批只做安全边界限制。
9.输出格式应稳定、清晰，推荐JSON或明确分段文本。
10.不得要求模型生成或阅读隐藏思维链。
11.保持无状态和线程安全。

十三、DefaultSupervisorAggregateNode

实现：

DefaultSupervisorAggregateNode implements SupervisorAggregateNode

依赖：
SupervisorObservationFormatter

职责：

1.读取：
-pendingTasks
-latestAgentResults

2.校验：
-均非空。
-数量一致。
-顺序可配对。

3.状态非法时：
-设置failureErrorCode=INTERNAL_ERROR。
-设置stopReason=INVALID_STATE。
-设置安全failureMessage。
-进入FAILURE。

4.将本轮latestAgentResults追加到历史agentResults。
5.使用SupervisorObservationFormatter生成观察内容。
6.将观察内容封装为UserAgentMessage并追加到supervisorMessages。
7.不要使用ToolAgentMessage表示子Agent结果，因为子Agent不是模型工具调用，避免破坏ToolCall/ToolResult配对协议。
8.处理完成后覆盖：
-pendingTasks=空列表
-latestAgentResults=空列表
-decision=null或当前State允许的空值
9.不得增加iteration。
10.不得调用模型。
11.不得执行子Agent。
12.不得重新格式化完整历史结果。
13.不得把子Agent内部ReAct状态复制到SupervisorState。
14.不得返回null。

十四、调整SupervisorRouter

检查并修正SupervisorRouter，路由优先级必须为：

1.存在failureErrorCode，或stopReason属于MODEL_ERROR、AGENT_ERROR、INVALID_STATE：
→FAIL

2.decision.action=FINISH且pendingTasks为空：
→COMPLETE
即使iteration刚好达到maxIterations，也允许正常完成。

3.decision.action=DISPATCH且pendingTasks非空且iteration>=maxIterations：
→MAX_ITERATIONS
不得执行本轮新任务。

4.decision.action=DISPATCH且pendingTasks非空：
→DISPATCH

5.iteration>=maxIterations：
→MAX_ITERATIONS

6.其他无法解释的状态：
→FAIL

要求：

1.Router保持纯Java。
2.Router不调用模型。
3.Router不执行子Agent。
4.Router不修改State。
5.不得硬编码最大次数。
6.不得使用Object、反射或instanceof进行节点分派。
7.不得加入think_tool。

十五、DefaultSupervisorCompleteNode

实现：

DefaultSupervisorCompleteNode implements SupervisorCompleteNode

职责：

1.确认当前decision.action=FINISH。
2.确认finalAnswer非空。
3.使用SupervisorDefinition.name作为AgentResult.agentName。
4.使用decision.finalAnswer作为最终content。
5.从历史agentResults中收集evidence：
-只收集非空evidence。
-保持原顺序。
-去重。
-设置合理数量上限。
6.构造成功AgentResult。
7.metadata可包含：
-iteration
-dispatchedTaskCount
-agentResultCount
-successfulAgentCount
-failedAgentCount
-stopReason
8.metadata使用不可变Map。
9.设置stopReason=COMPLETED。
10.设置finalResult。
11.不得再次调用模型。
12.不得执行子Agent。
13.不得返回完整agentResults、messages或内部State。
14.非法状态时进入失败语义，不得返回null。

十六、DefaultSupervisorMaxIterationsNode

实现：

DefaultSupervisorMaxIterationsNode implements SupervisorMaxIterationsNode

职责：

1.设置stopReason=MAX_ITERATIONS_REACHED。
2.构造失败AgentResult。
3.errorCode使用MAX_ITERATIONS_REACHED。
4.content使用安全提示：
“Supervisor已达到最大调度迭代次数，未能在限制内完成任务。”
5.metadata可包含：
-iteration
-maxIterations
-agentResultCount
-supervisorName
6.清空未执行pendingTasks和latestAgentResults。
7.不得执行达到上限时新生成的子任务。
8.不得调用模型或子Agent。
9.不得返回null。

十七、DefaultSupervisorFailureNode

实现：

DefaultSupervisorFailureNode implements SupervisorFailureNode

职责：

1.根据failureErrorCode、failureMessage和stopReason构造失败AgentResult。
2.failureErrorCode为空时使用INTERNAL_ERROR。
3.failureMessage为空时使用安全通用提示。
4.映射至少覆盖：
-MODEL_ERROR→MODEL_INVOCATION_FAILED
-AGENT_ERROR→AGENT执行相关错误或INTERNAL_ERROR
-INVALID_STATE→INTERNAL_ERROR
5.设置finalResult。
6.不得返回堆栈、API Key、文件路径、内部类名或完整模型响应。
7.不得调用模型或子Agent。
8.不得返回null。

十八、Supervisor图结构

SupervisorGraphFactory必须保持：

START→supervisor_reason

supervisor_reason条件路由：
DISPATCH→dispatch_agents
COMPLETE→complete
MAX_ITERATIONS→max_iterations_fallback
FAIL→failure

dispatch_agents→aggregate_results
aggregate_results→supervisor_reason

complete→END
max_iterations_fallback→END
failure→END

要求：

1.同步节点通过node_async注册。
2.同步Router通过edge_async注册。
3.不存在wrapNode(Object)。
4.不存在instanceof节点分派。
5.GraphFactory不调用模型。
6.GraphFactory不执行子Agent。
7.GraphFactory不保存某次运行State。
8.节点名集中维护。
9.编译异常转换为AgentFrameworkException和INTERNAL_ERROR，保留cause。

十九、DefaultSupervisorEngine

实现：

DefaultSupervisorEngine implements SupervisorEngine

依赖：

1.SupervisorExecutionValidator。
2.SupervisorPromptBuilder。
3.SupervisorGraphFactory或CompiledGraph。

保持纯Java，不添加Spring注解。

职责：

1.execute开始时调用SupervisorExecutionValidator。
2.使用SupervisorPromptBuilder构造系统提示词。
3.构造初始supervisorMessages：
-SystemAgentMessage：Supervisor系统提示词。
-UserAgentMessage：rootTask.instruction。
4.不得把rootTask.context通过toString直接拼进消息。
5.不得把RunContext或安全身份放入消息。
6.构造初始State：
-supervisorDefinition
-rootTask
-runContext
-supervisorMessages
-decision=null
-pendingTasks=空
-latestAgentResults=空
-agentResults=空
-iteration=0
-finalResult=null
-stopReason=null
-failureErrorCode=null
-failureMessage=null
7.通过当前LangGraph4j真实invoke、invokeFinal或等价API执行父图。
8.不得让调用方直接操作CompiledGraph。
9.执行结束后读取finalResult。
10.finalResult为空时抛INTERNAL_ERROR，不得返回null。
11.异常处理：
-AgentFrameworkException保留错误码。
-普通Exception转换为INTERNAL_ERROR。
-记录parentRunId、threadId、supervisorName及服务端异常。
-不得记录完整State、消息、Prompt或子Agent结果全文。
12.不得实现resume。
13.不得保存全局可变State。
14.不得使用ThreadLocal保存State。
15.同一Engine实例应支持不同runId并发执行。
16.不得实现Supervisor HTTP接口。

二十、CompiledGraph生命周期

先检查当前LangGraph4j版本CompiledGraph线程安全语义。

优先：

1.若CompiledGraph可安全并发且不保存运行State：
-编译一次并以final字段复用。

2.若线程安全语义无法确认：
-每次execute通过SupervisorGraphFactory创建独立CompiledGraph。
-在最终输出中说明原因。

禁止：

1.static可变CompiledGraph。
2.在CompiledGraph中保存某次运行State。
3.使用全局锁串行化所有Supervisor请求，除非当前框架明确要求并说明。
4.跨请求复用SupervisorAgentState。

二十一、Spring装配

所有runtime实现保持纯Java，不添加：

@Component
@Service
@Autowired
@Configuration

在agent-infrastructure新增或完善配置类，装配：

1.JacksonSupervisorDecisionParser。
2.SupervisorPromptBuilder。
3.DefaultSupervisorReasonNode。
4.DefaultSupervisorDispatchNode。
5.SupervisorObservationFormatter。
6.DefaultSupervisorAggregateNode。
7.DefaultSupervisorCompleteNode。
8.DefaultSupervisorMaxIterationsNode。
9.DefaultSupervisorFailureNode。
10.SupervisorRouter。
11.SupervisorExecutionValidator。
12.SupervisorGraphFactory。
13.DefaultSupervisorEngine。

要求：

1.复用已有ModelInvocationGateway。
2.复用已有AgentRegistry。
3.复用已有ReactAgentEngine。
4.复用已有RunIdGenerator。
5.复用已有ObjectMapper，但不得修改其全局配置。
6.使用@Configuration和@Bean。
7.不得在bootstrap启动类中手工new。
8.不得创建Fake ModelClient或Fake ReactAgentEngine。
9.模型能力不存在时，SupervisorEngine相关Bean可以条件装配。
10.无模型配置时应用仍能启动。
11.没有SupervisorProvider时应用仍能启动。
12.不得产生Bean循环依赖。
13.本批不注册Sample Supervisor。
14.本批不创建ApplicationService或Controller。

二十二、Supervisor与子Agent结果语义

1.子Agent success=false不等于父Supervisor立即失败。
2.子Agent失败结果必须反馈给Supervisor模型。
3.Supervisor可以：
-重新分派同一Agent。
-选择其他Agent。
-根据已有失败结果返回解释。
4.只有父图模型调用失败、State损坏、无法执行分派等框架错误才进入FailureNode。
5.不得丢弃失败AgentResult。
6.不得把子Agent内部异常堆栈反馈给Supervisor模型。
7.不得把子Agent完整ToolTrace反馈给Supervisor模型。
8.不得把子Agent完整消息历史反馈给Supervisor模型。

二十三、Supervisor最大迭代语义

iteration表示已经完成的Supervisor模型调用次数。

规则：

1.初始0。
2.每次SupervisorReasonNode成功调用模型后+1。
3.DispatchNode、AggregateNode和终止节点不增加。
4.最多调用maxIterations次Supervisor模型。
5.最后一次允许调用返回FINISH时正常完成。
6.最后一次允许调用返回DISPATCH时，不执行新任务，进入MAX_ITERATIONS。
7.不得使用Integer::sum后又手动加1。
8.不得让子Agent ReAct iteration影响Supervisor iteration。

二十四、安全要求

1.模型输出属于非可信输入，必须经过Parser、白名单和字段校验。
2.模型不能决定：
-userId
-sessionId
-threadId
-runId
-roles
-permissions
-systemPrompt
-allowedTools
-maxIterations
3.Supervisor只能选择memberAgents。
4.子Agent只能使用自身AgentDefinition.allowedTools。
5.Supervisor不能绕过ToolInvocationGateway执行工具。
6.不得把RunContext发送给模型。
7.日志不得记录：
-完整Prompt
-完整supervisorMessages
-完整子Agent结果
-完整工具参数
-API Key
-用户权限全集
8.客户端安全错误不得包含堆栈和底层异常。
9.不得保存详细思维链，只保存简洁decisionSummary。

二十五、本批禁止实现

禁止：

1.Sample Supervisor。
2.SupervisorProvider业务实现。
3.Supervisor开发Controller。
4.Supervisor Postman接口。
5.多Supervisor全局路由。
6.子Agent并行执行。
7.线程池分派。
8.动态并发上限。
9.think_tool。
10.Supervisor直接调用业务Tool。
11.子Agent作为SpringAI ToolCallback。
12.登录、Session和RBAC。
13.Tool ACL。
14.HITL、interrupt/resume。
15.CheckpointStore内存实现。
16.上下文裁剪、TokenCounter和摘要。
17.长期记忆。
18.RAG、MCP和真实金山云业务。
19.SSE和流式输出。
20.前端页面。
21.模型重试、熔断和多模型路由。
22.测试脚本。
23.README、使用说明、升级记录和验收报告。
24.git commit和git push。

二十六、代码质量

1.不使用Lombok。
2.优先record、enum、final类。
3.使用构造器注入。
4.集合和Map使用不可变快照。
5.不使用字段注入。
6.不通过ApplicationContext主动查Bean。
7.不吞异常。
8.不返回null表示失败。
9.不通过Object、instanceof或反射进行节点分派。
10.不创建无意义Facade、Delegate、Manager或Adapter。
11.Supervisor节点只负责编排和State更新。
12.模型技术细节继续留在ModelInvocationGateway和SpringAI适配。
13.子Agent执行继续完全复用ReactAgentEngine。
14.所有代码以实际编译和运行结果为准。
15.不要为通过编译创建固定Decision、固定AgentResult或假模型。

二十七、编译验收

完成后执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许启动应用，并确认原有接口不受影响：

GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents
POST /api/dev/react/invoke

检查：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core无SpringAI、LangGraph4j和Spring依赖。
4.agent-runtime可依赖LangGraph4j，但无SpringAI、SpringWeb和agent-infrastructure依赖。
5.agent-application未增加SpringAI或LangGraph4j依赖。
6.SupervisorGraphFactory不存在wrapNode(Object)。
7.不存在instanceof节点分派。
8.同步节点通过node_async注册。
9.同步Router通过edge_async注册。
10.SupervisorReasonNode每轮只调用模型一次。
11.Supervisor模型请求不包含工具定义。
12.模型返回ToolCall时进入安全失败。
13.模型任务目标只能是memberAgents。
14.模型不能控制taskId和RunContext。
15.DispatchNode只通过ReactAgentEngine执行子Agent。
16.子Agent按顺序执行。
17.一个子Agent失败不会直接终止父图。
18.AggregateNode正确追加历史结果和观察消息。
19.子Agent结果不使用ToolAgentMessage回灌Supervisor。
20.Supervisor iteration只在Reason模型调用后增加。
21.达到上限后不执行新子任务。
22.最后一次允许的模型调用返回FINISH时可正常完成。
23.所有终止路径finalResult不为空。
24.无模型配置时应用仍能启动。
25.没有Sample Supervisor或Supervisor API。
26.现有单Agent ReAct功能未被破坏。
27.git diff不存在无关修改。

二十八、最终输出

只输出：

1.新增和修改文件清单。
2.Supervisor模型JSON协议。
3.非可信Draft到正式SupervisorDecision的转换流程。
4.SupervisorPromptBuilder公开给模型的信息。
5.SupervisorReasonNode职责和校验。
6.子Agent RunContext隔离规则。
7.DispatchNode顺序执行流程。
8.子Agent失败如何反馈Supervisor。
9.AggregateNode观察消息格式和截断策略。
10.SupervisorRouter最终路由优先级。
11.完整Supervisor状态流转。
12.CompiledGraph生命周期选择。
13.SpringBean和条件装配方式。
14.编译、打包和启动验证结果。
15.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE4_BATCH2_END===