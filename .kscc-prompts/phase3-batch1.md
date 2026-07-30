你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven多模块架构和核心领域抽象。
2.AgentRegistry、ToolRegistry及Provider自动注册。
3.ToolInvocationGateway、参数校验、异常处理和审计拦截器链。
4.CalculatorTool、CurrentTimeTool、EchoTool、TextSearchTool。
5.ModelInvocationGateway、SpringAI模型适配、消息和ToolCall双向转换。
6.模型调用链与工具执行链目前相互独立。

当前只执行第三阶段第1批：建立通用ReAct执行模型、状态结构、节点契约和LangGraph4j图骨架。本批不实现真实Reason模型调用、不执行工具、不开放聊天接口。

一、总体目标
为后续形成以下循环准备稳定骨架：

START
→reason
→route_after_reason
├─TOOL_CALL→execute_tools→observe→reason
├─COMPLETE→complete→END
└─MAX_ITERATIONS→max_iterations_fallback→END

本批只建立状态、路由、节点契约和图结构，下一批再连接ModelInvocationGateway和ToolInvocationGateway。

二、执行原则
1.先检查全部pom.xml、现有AgentDefinition、AgentTask、AgentResult、AgentMessage、ModelInvocationGateway、ToolInvocationGateway、AgentRuntime及Registry代码。
2.以现有接口、包名、错误码和Java版本为准增量实现。
3.不要创建第二套Agent、Message、ToolCall、ModelRequest或ToolResult模型。
4.发现前序代码存在阻断问题时，只做最小兼容修改并说明。
5.继续遵循最新模块化架构：
-agent-core：稳定且框架无关的领域模型。
-agent-runtime：ReAct运行规则、状态、节点契约和图编排。
-agent-infrastructure：SpringAI、SpringBean和具体技术装配。
-agent-application：应用用例。
-agent-api：HTTP入口。
-agent-bootstrap：启动组装。
6.本批允许agent-runtime引入LangGraph4j，因为图编排属于运行时核心；但不得引入SpringAI、SpringWeb或agent-infrastructure。
7.agent-core仍不得依赖LangGraph4j。
8.不要生成测试脚本。
9.不要生成或修改README、使用说明、升级记录、验收报告等文档。
10.不要执行git commit或git push。

三、LangGraph4j依赖
1.检查项目当前是否已有LangGraph4j依赖。
2.若没有，在父pom统一管理与当前Java/SpringBoot兼容的LangGraph4j版本。
3.仅在需要直接使用图API的模块声明依赖，优先放在agent-runtime。
4.版本号不得散落在多个子模块。
5.不得添加LangChain4j依赖替代LangGraph4j。
6.使用当前实际依赖版本的API，编译失败时根据真实API修正，不得照搬不兼容示例。
7.不要引入与本批无关的checkpoint、数据库或模型Starter。

四、ReAct停止原因
在agent-runtime创建包：
com.ksyun.agent.runtime.react

实现枚举：
ReactStopReason

至少包含：
MODEL_COMPLETED
MAX_ITERATIONS_REACHED
MODEL_ERROR
TOOL_ERROR
INVALID_STATE

该枚举描述ReAct执行结束原因，不替代现有RunStatus。

五、ReAct执行状态
实现框架运行时状态模型ReactAgentState。

注意：
1.它属于agent-runtime，不放入agent-core。
2.它可以采用LangGraph4j推荐的State表示方式，但对外访问应通过明确的key和访问方法，禁止各节点散落硬编码字符串。
3.若LangGraph4j要求Map型State，可封装统一StateKeys和类型安全读取方法。
4.不要把SpringAI Message、ChatResponse或ToolCallback放入State。

状态至少需要表达：
AgentDefinition agentDefinition
AgentTask task
RunContext runContext
List<AgentMessage> messages
List<ToolCall> pendingToolCalls
List<ToolResult> latestToolResults
List<ToolExecutionTrace> toolTraces
int iteration
AgentResult finalResult
ReactStopReason stopReason
String failureMessage

要求：
1.messages保持顺序。
2.pendingToolCalls和latestToolResults每轮允许覆盖。
3.toolTraces采用追加合并语义。
4.iteration采用覆盖或明确递增语义，不能因Reducer重复相加。
5.finalResult、stopReason采用覆盖语义。
6.集合读取返回不可变快照。
7.null集合按空集合处理。
8.不得在State中保存ChatModel、ModelClient、Registry、Gateway等运行组件。
9.不得把roles、permissions等RunContext内容拼入LLM消息。
10.状态应能在后续Checkpoint中序列化或映射，不依赖不可序列化的Spring对象。

六、ReactStateKeys
如采用Map型State，集中定义：
ReactStateKeys

至少包含：
AGENT_DEFINITION
TASK
RUN_CONTEXT
MESSAGES
PENDING_TOOL_CALLS
LATEST_TOOL_RESULTS
TOOL_TRACES
ITERATION
FINAL_RESULT
STOP_REASON
FAILURE_MESSAGE

要求：
1.禁止节点自行写字符串key。
2.常量不可修改。
3.提供安全的状态读取辅助方法。
4.读取类型不符时抛出带INVALID_ARGUMENT或INTERNAL_ERROR的清晰框架异常。
5.不要实现大量无意义getter包装。

七、ToolExecutionTrace
检查项目是否已有职责相同的ToolTrace或执行轨迹模型。

若已有，复用并只做必要补充。
若没有，在agent-runtime.react中实现不可变ToolExecutionTrace，建议字段：
String toolCallId
String toolName
boolean success
String errorCode
long durationMillis
Instant startedAt
Instant finishedAt

要求：
1.不保存完整工具参数。
2.不保存完整工具返回内容。
3.不保存异常对象。
4.字段适合后续审计、前端轨迹和Checkpoint。

八、ReAct路由枚举
实现：
ReactRoute

至少包含：
EXECUTE_TOOLS
COMPLETE
MAX_ITERATIONS
FAIL

语义：
EXECUTE_TOOLS：模型返回一个或多个ToolCall。
COMPLETE：模型不再请求工具并产生最终回答。
MAX_ITERATIONS：达到AgentDefinition.maxIterations。
FAIL：状态或节点执行失败。

九、节点契约
在agent-runtime.react.node中定义职责清晰的节点接口或抽象契约。

至少包括：
ReactReasonNode
ReactToolExecutionNode
ReactObserveNode
ReactCompleteNode
ReactMaxIterationsNode
ReactFailureNode

要求：
1.节点契约与当前LangGraph4j节点API适配。
2.本批不要创建返回固定答案或假ToolCall的伪实现。
3.允许暂时只定义接口，由下一批实现具体节点。
4.节点输入输出必须围绕ReactAgentState。
5.不得直接依赖Controller、ApplicationService或Spring容器。
6.不得在节点内部通过ApplicationContext查找Bean。
7.不得加入@Component、@Service、@Autowired。
8.后续具体节点应通过构造器接收ModelInvocationGateway、ToolInvocationGateway等依赖。

十、路由器契约
实现：
ReactRouter

职责：
根据当前状态决定ReactRoute。

本批可实现完整路由规则，因为不依赖真实模型和工具：

路由优先级建议：
1.若stopReason表示失败或failureMessage存在→FAIL。
2.若iteration达到或超过agentDefinition.maxIterations且仍需继续→MAX_ITERATIONS。
3.若pendingToolCalls非空→EXECUTE_TOOLS。
4.若finalResult已存在或模型已明确完成→COMPLETE。
5.其他非法状态→FAIL。

要求：
1.路由逻辑纯Java、无模型调用。
2.路由结果确定且可解释。
3.maxIterations必须读取AgentDefinition配置，不硬编码10或20。
4.边界语义必须明确，例如iteration表示“已经完成的Reason轮数”。
5.在代码注释中说明iteration递增时机，避免下一批出现off-by-one问题。
6.禁止在Router中执行工具或调用模型。

十一、ReAct图工厂契约
实现：
ReactAgentGraphFactory

职责：
根据节点实现构建通用ReAct StateGraph。

要求：
1.通过构造器接收各节点或节点工厂，不从Spring容器主动查找。
2.注册以下逻辑节点：
reason
execute_tools
observe
complete
max_iterations_fallback
failure
3.建立以下边：
START→reason
reason→条件路由
EXECUTE_TOOLS→execute_tools
execute_tools→observe
observe→reason
COMPLETE→complete
complete→END
MAX_ITERATIONS→max_iterations_fallback
max_iterations_fallback→END
FAIL→failure
failure→END
4.如果当前LangGraph4j条件边API需要返回字符串节点名，集中维护节点名常量。
5.不得把节点名字符串散落在多个类中。
6.图工厂本身不保存某次运行的可变State。
7.同一个工厂可以安全创建或复用已编译图，具体方式依据LangGraph4j线程安全语义设计。
8.如果当前版本CompiledGraph线程安全语义不明确，优先由工厂创建并在最终输出说明生命周期选择。
9.本批允许因为具体节点实现尚未提供而只完成“接收节点后可构图”的工厂；不得提供返回固定结果的假节点。
10.不得实现Supervisor图。

十二、ReactAgentEngine接口
检查现有AgentRuntime接口，不要直接破坏它。

在agent-runtime.react中定义更聚焦的接口：
ReactAgentEngine

建议方法：
AgentResult execute(
AgentDefinition definition,
AgentTask task,
RunContext context
);

要求：
1.这是单Agent ReAct执行入口。
2.本批只定义接口，不创建返回固定结果的假实现。
3.后续实现将使用ReactAgentGraphFactory编译和执行图。
4.不要在本批实现resume；审批恢复属于HITL阶段。
5.不要让业务调用者直接操作CompiledGraph。
6.不要让接口暴露LangGraph4j类型。

十三、运行请求校验器
实现纯Java：
ReactExecutionValidator

校验：
1.AgentDefinition非空。
2.AgentTask非空。
3.RunContext非空。
4.AgentTask.agentName与AgentDefinition.name一致；如现有设计允许空agentName，应明确兼容规则。
5.maxIterations大于0。
6.task.instruction非空。
7.AgentDefinition.allowedTools不为null。
8.校验失败抛出已有AgentFrameworkException和INVALID_ARGUMENT。
9.不调用模型或工具。
10.不修改输入对象。

十四、Reducer和状态合并
根据当前LangGraph4j版本配置Reducer/Channel策略。

要求：
1.messages：后续节点能够追加消息，且不会覆盖历史。
2.toolTraces：追加。
3.pendingToolCalls：覆盖。
4.latestToolResults：覆盖。
5.iteration：覆盖，节点明确写入新值；不要配置Integer::sum后又在节点手动+1。
6.finalResult：覆盖。
7.stopReason：覆盖。
8.failureMessage：覆盖。
9.避免使用无类型Map合并导致ClassCastException。
10.在最终输出中说明每个关键字段的合并语义。

十五、Spring装配边界
本批主要完成runtime代码。

如ReactAgentGraphFactory或Router需要成为SpringBean：
1.类本身保持纯Java，不加@Component。
2.在agent-infrastructure中通过@Configuration和@Bean装配。
3.如果具体节点尚未实现，暂时不要强行创建无法满足依赖的GraphFactory Bean。
4.应用在本批完成后仍必须正常启动。
5.不得创建假的ModelInvocationGateway或假的ToolInvocationGateway。
6.不得修改bootstrap启动类手工new组件。

十六、本批禁止实现
禁止：
1.真实ReasonNode调用ModelInvocationGateway。
2.真实ToolExecutionNode调用ToolInvocationGateway。
3.ToolResult转ToolAgentMessage。
4.Reason→Act→Observe完整循环执行。
5.示例Agent。
6.Supervisor。
7.AgentDispatcher。
8.ReAct聊天Controller或开发API。
9.Session、RBAC、ACL。
10.HITL、interrupt/resume。
11.CheckpointStore内存实现。
12.上下文裁剪和摘要。
13.长期记忆。
14.RAG、MCP或金山云业务。
15.并行工具执行。
16.流式输出。
17.前端页面。
18.为通过编译创建返回固定答案的伪节点或伪模型。

十七、代码质量
1.不使用Lombok。
2.优先使用record、enum、final类和构造器依赖。
3.集合使用不可变快照。
4.节点、Router、GraphFactory职责单一。
5.不使用字段注入。
6.不通过静态全局变量保存运行状态。
7.不使用ThreadLocal保存ReAct State。
8.错误信息明确但不泄露内部实现。
9.不要大规模修改第二阶段已完成代码。
10.不要为了抽象再增加Facade、Delegate、Manager等无必要层级。

十八、编译验收
执行：
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

检查：
1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core没有LangGraph4j、SpringAI或Spring import。
4.agent-runtime允许有LangGraph4j import，但没有SpringAI和SpringWeb import。
5.现有health/tools/agents/model开发接口不受影响。
6.应用在无模型配置时仍可启动。
7.没有Bean循环依赖。
8.没有假节点或固定结果实现。
9.ReactRouter的maxIterations边界清晰。
10.State字段合并语义符合要求。
11.没有实现Supervisor或完整ReAct循环。
12.git diff中没有无关修改。

十九、最终输出
只输出：
1.新增和修改文件清单。
2.LangGraph4j依赖及放置模块。
3.ReactAgentState字段及作用。
4.关键State字段的Reducer/合并语义。
5.iteration定义及递增时机约定。
6.ReactRouter路由优先级。
7.ReactAgentGraphFactory图结构。
8.节点契约及下一批需要实现的节点。
9.是否进行了SpringBean装配以及原因。
10.编译和打包结果。
11.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE3_BATCH1_END===