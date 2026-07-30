你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven多模块架构和agent-core领域抽象。
2.AgentRegistry、ToolRegistry及Provider自动注册。
3.ToolInvocationGateway、参数校验、异常治理和审计拦截器链。
4.CalculatorTool、CurrentTimeTool、EchoTool、TextSearchTool。
5.ModelInvocationGateway和SpringAI适配。
6.ReactAgentState、ReactRouter、节点契约、ReactAgentGraphFactory及ReAct图骨架。

当前只执行第三阶段第2批：实现通用ReAct节点、完整Reason→Act→Observe循环和DefaultReactAgentEngine。

本批不实现Supervisor、示例Agent、聊天Controller、ACL、HITL、Checkpoint持久化、上下文裁剪或长期记忆。

一、执行前检查
1.先完整检查全部pom.xml及以下现有代码：
-AgentDefinition、AgentTask、AgentResult
-AgentMessage及各消息类型
-ModelRequest、ModelResponse、ModelInvocationGateway
-ToolCall、ToolResult、ToolInvocationGateway、ToolRegistry
-ReactAgentState、ReactStateKeys、ReactStopReason、ReactRoute
-所有React节点契约
-ReactRouter
-ReactAgentGraphFactory
-ReactAgentEngine
-ReactExecutionValidator
2.以当前项目实际接口、错误码、包结构、Java版本及LangGraph4j版本为准增量实现。
3.不得创建职责相同但名称不同的第二套消息、工具、Agent、State或网关模型。
4.发现上一批存在阻断问题时只做最小修复，并在最终输出中说明。
5.如果ReactAgentGraphFactory仍存在wrapNode(Object)、instanceof节点分派或手写wrapRouter，先按第二部分要求修正，再实现本批节点。
6.不要大规模重构前两阶段已完成代码。

二、LangGraph4j原生API强制规则
agent-runtime已经允许直接依赖LangGraph4j，本批必须优先使用LangGraph4j原生运行接口。

当前ModelInvocationGateway和ToolInvocationGateway均为同步接口，因此本批节点优先采用同步NodeAction：

1.ReactReasonNode、ReactToolExecutionNode、ReactObserveNode、ReactCompleteNode、ReactMaxIterationsNode、ReactFailureNode应直接继承或兼容：
NodeAction<ReactAgentState>

2.各具体节点直接实现对应节点契约。

3.ReactAgentGraphFactory注册同步节点时必须使用当前LangGraph4j版本原生：
node_async(node)

4.ReactRouter为同步路由时，应实现EdgeAction<ReactAgentState>或提供与其兼容的方法引用，GraphFactory使用：
edge_async(...)

5.如果现有节点已合理实现AsyncNodeAction，则直接注册，不得再次包装。

强制禁止：
-wrapNode(Object)
-wrapRouter(Object)
-使用Object接收节点或路由
-通过instanceof链识别节点类型
-通过反射识别节点类型
-为每种节点在GraphFactory中增加转换分支
-自行重复实现node_async或edge_async
-创建与NodeAction/AsyncNodeAction语义重复但不兼容的平行接口
-为了形式统一把同步代码包装成无意义的自定义异步层

GraphFactory只负责：
-构建Channels
-注册节点
-注册边
-注册条件路由
-编译图

不得承担节点类型识别、模型调用、工具调用或Spring Bean查找。

三、迭代次数语义
统一规定：

iteration表示“已经完成的Reason模型调用次数”。

规则：
1.初始iteration=0。
2.每成功完成一次模型调用后iteration+1。
3.ObserveNode、ToolExecutionNode和终止节点不得增加iteration。
4.maxIterations读取AgentDefinition.maxIterations，不得硬编码。
5.当某次Reason调用后模型不再请求工具并给出最终回答，即使iteration刚好等于maxIterations，也允许正常COMPLETE。
6.当某次Reason调用后仍返回ToolCall且iteration已达到maxIterations，不再执行这些新ToolCall，进入MAX_ITERATIONS。
7.不得在Channel中使用Integer::sum后又由节点手动+1。
8.避免off-by-one：最多允许执行maxIterations次模型Reason调用。

四、状态更新规则
所有节点只返回当前节点产生的State增量，不得每次复制整个State。

根据现有Channel配置遵守：

1.messages为追加语义：
-节点只返回本轮新增消息。
-不得返回完整历史消息。
-不得造成嵌套List或历史消息重复。

2.toolTraces为追加语义：
-ToolExecutionNode只返回本轮新增轨迹。

3.pendingToolCalls为覆盖语义：
-ReasonNode返回本轮全部ToolCall。
-ObserveNode处理完后覆盖为空列表。

4.latestToolResults为覆盖语义：
-ToolExecutionNode返回本轮完整结果列表。
-ObserveNode处理完后覆盖为空列表。

5.iteration为覆盖语义：
-ReasonNode返回新的明确值。

6.finalResult、stopReason、failureMessage及失败错误码采用覆盖语义。

7.若当前State缺少failureErrorCode或等价字段，允许在agent-runtime内做最小补充：
AgentErrorCode failureErrorCode
并同步增加StateKey和覆盖Channel。

8.状态中不得保存：
-ChatModel
-ChatClient
-ModelClient
-ToolRegistry
-ToolInvocationGateway
-SpringAI消息
-CompiledGraph
-异常对象
-Spring Bean

五、实现DefaultReactReasonNode
在agent-runtime的react.node包中实现：
DefaultReactReasonNode implements ReactReasonNode

依赖：
ModelInvocationGateway
ToolRegistry

使用构造器注入，保持纯Java，不添加Spring注解。

执行职责：

1.从State读取并校验：
-AgentDefinition
-AgentTask
-RunContext
-messages
-iteration

2.如果iteration已经达到maxIterations且本节点尚未进行新的模型调用：
-不得继续调用模型。
-设置stopReason=MAX_ITERATIONS_REACHED。
-路由至MAX_ITERATIONS。

3.根据AgentDefinition.allowedTools构造本轮允许暴露给模型的ToolDefinition：
-只暴露allowedTools中明确列出的工具。
-allowedTools为空时发送空工具列表。
-不得把ToolRegistry中的全部工具自动暴露给模型。
-工具不存在时设置failureErrorCode=TOOL_NOT_FOUND、stopReason=TOOL_ERROR及安全failureMessage。
-不得直接执行工具。

4.使用当前messages构造ModelRequest：
-messages保持原顺序。
-tools为本Agent允许的工具定义。
-options使用不可变空Map或当前已有安全默认配置。
-不得把RunContext、roles、permissions、sessionId、userId自动转换成LLM消息。
-不得把AgentTask.context直接通过toString拼进Prompt。
-AgentTask.context本批保留在State中，后续由上下文管理能力规范注入。

5.调用ModelInvocationGateway一次。

6.处理ModelResponse：
-取得AssistantAgentMessage。
-将本轮AssistantAgentMessage追加到messages。
-将response中的全部ToolCall覆盖到pendingToolCalls。
-清空latestToolResults。
-iteration明确加1。
-不得只保留第一个ToolCall。

7.如果模型返回一个或多个ToolCall：
-允许Assistant文本同时存在。
-不要设置MODEL_COMPLETED。
-由Router决定EXECUTE_TOOLS或MAX_ITERATIONS。

8.如果模型未返回ToolCall：
-content非空时设置stopReason=MODEL_COMPLETED。
-不得在ReasonNode内直接构造最终AgentResult，最终结果由CompleteNode统一构造。

9.如果模型既无有效文本也无ToolCall：
-设置failureErrorCode=MODEL_INVOCATION_FAILED。
-设置stopReason=MODEL_ERROR。
-设置安全failureMessage。
-路由至FAILURE。

10.异常处理：
-AgentFrameworkException保留其AgentErrorCode。
-普通Exception转换为MODEL_INVOCATION_FAILED。
-记录runId、agentName、iteration和完整服务端异常。
-不得把Prompt、消息全文、密钥、模型原始响应或堆栈写入failureMessage。
-不要捕获Error等JVM严重错误。

六、实现DefaultReactToolExecutionNode
实现：
DefaultReactToolExecutionNode implements ReactToolExecutionNode

依赖：
ToolInvocationGateway

职责：

1.读取pendingToolCalls和RunContext。
2.pendingToolCalls为空时：
-设置failureErrorCode=INTERNAL_ERROR。
-设置stopReason=INVALID_STATE。
-设置安全failureMessage。
-不进入工具执行。

3.按照模型返回顺序依次执行全部ToolCall。
4.本批不并行执行工具。
5.每个ToolCall构造：
ToolInvocation(toolCall,runContext)
6.每个工具必须通过ToolInvocationGateway执行。
7.禁止直接调用：
-AgentTool.execute
-ToolRegistry.getRequired(...).execute
-SpringAI ToolCallback
8.保持ToolCall和ToolResult顺序一一对应。
9.将全部ToolResult覆盖写入latestToolResults。
10.为每个调用生成ToolExecutionTrace：
-toolCallId
-toolName
-success
-errorCode
-durationMillis
-startedAt
-finishedAt
11.轨迹不得保存完整参数、完整结果或异常对象。
12.ToolResult.success=false属于可观察的工具失败：
-正常保存结果。
-不要立即结束整个ReAct。
-后续ObserveNode将失败结果作为ToolAgentMessage回灌模型，让模型决定如何处理。
13.只有发生无法形成有效ToolResult的框架内部异常时，才设置：
-stopReason=TOOL_ERROR
-failureErrorCode=TOOL_EXECUTION_FAILED或原框架错误码
-failureMessage=安全提示
14.不得修改messages。
15.不得增加iteration。
16.不得清空pendingToolCalls，ObserveNode还需要使用它完成结果配对。

七、实现DefaultReactObserveNode
实现：
DefaultReactObserveNode implements ReactObserveNode

职责：

1.读取：
-pendingToolCalls
-latestToolResults

2.要求二者：
-均非null
-数量一致
-顺序一一对应

3.数量不一致或状态非法时：
-设置failureErrorCode=INTERNAL_ERROR。
-设置stopReason=INVALID_STATE。
-设置安全failureMessage。
-路由至FAILURE。

4.为每组ToolCall和ToolResult构造ToolAgentMessage：
-toolCallId=原ToolCall.id
-toolName=原ToolCall.name
-content=ToolResult.content
-error=!ToolResult.success

5.ToolResult失败也必须生成ToolAgentMessage，不能丢弃。

6.将本轮所有ToolAgentMessage追加到messages。

7.处理完成后覆盖：
pendingToolCalls=空列表
latestToolResults=空列表

8.不得增加iteration。
9.不得调用模型。
10.不得执行工具。
11.不得设置MODEL_COMPLETED。
12.不得将ToolResult.metadata、异常对象或完整审计信息拼进消息，除非当前ToolResult.content已经是经过Gateway治理的安全内容。

八、实现DefaultReactCompleteNode
实现：
DefaultReactCompleteNode implements ReactCompleteNode

职责：

1.确认stopReason=MODEL_COMPLETED。
2.从messages中找到最后一个AssistantAgentMessage。
3.确认该消息不再包含ToolCall。
4.使用其content构造成功AgentResult。
5.AgentResult.agentName使用AgentDefinition.name。
6.AgentResult.content使用最终Assistant文本。
7.evidence当前使用空列表，不伪造证据。
8.metadata可包含非敏感信息：
-iteration
-toolExecutionCount
-stopReason
9.metadata使用不可变Map。
10.设置finalResult。
11.不得再次调用模型。
12.不得执行工具。
13.如果找不到合法最终Assistant消息，则转为失败结果或设置INVALID_STATE，不得返回null。

九、实现DefaultReactMaxIterationsNode
实现：
DefaultReactMaxIterationsNode implements ReactMaxIterationsNode

职责：

1.设置stopReason=MAX_ITERATIONS_REACHED。
2.构造明确的失败AgentResult。
3.errorCode使用MAX_ITERATIONS_REACHED。
4.content使用安全且可理解的提示，例如：
“Agent已达到最大迭代次数，未能在限制内完成任务。”
5.metadata可包含：
-iteration
-maxIterations
-agentName
6.清空未执行的pendingToolCalls和latestToolResults，避免上层误认为仍可继续。
7.不得执行达到上限时模型新生成的ToolCall。
8.不得调用模型或工具。
9.不得返回null。

十、实现DefaultReactFailureNode
实现：
DefaultReactFailureNode implements ReactFailureNode

职责：

1.根据failureErrorCode和stopReason构造失败AgentResult。
2.若failureErrorCode为空，使用INTERNAL_ERROR。
3.使用安全failureMessage；为空时使用通用错误提示。
4.映射至少覆盖：
-MODEL_ERROR→MODEL_INVOCATION_FAILED
-TOOL_ERROR→TOOL_EXECUTION_FAILED
-INVALID_STATE→INTERNAL_ERROR
5.设置finalResult。
6.不得返回原始异常、堆栈、API Key、文件路径或底层实现类。
7.不得调用模型或工具。
8.不得返回null。

十一、调整ReactRouter
根据本批完整语义检查并修正ReactRouter。

路由优先级必须明确：

1.存在failureErrorCode，或stopReason属于MODEL_ERROR、TOOL_ERROR、INVALID_STATE：
→FAIL

2.stopReason=MODEL_COMPLETED且pendingToolCalls为空：
→COMPLETE
即使iteration刚好等于maxIterations，也允许正常完成。

3.pendingToolCalls非空且iteration>=maxIterations：
→MAX_ITERATIONS
不得执行这些新ToolCall。

4.pendingToolCalls非空：
→EXECUTE_TOOLS

5.iteration>=maxIterations：
→MAX_ITERATIONS

6.其他无法解释的状态：
→FAIL，并确保有INVALID_STATE和安全failureMessage。

Router必须：
-保持纯Java
-不调用模型
-不执行工具
-不修改业务数据，除非当前LangGraph4j路由契约绝对要求；优先只做判断
-不硬编码最大次数

十二、图结构要求
ReactAgentGraphFactory必须保持以下结构：

START→reason

reason条件路由：
EXECUTE_TOOLS→execute_tools
COMPLETE→complete
MAX_ITERATIONS→max_iterations_fallback
FAIL→failure

execute_tools→observe
observe→reason

complete→END
max_iterations_fallback→END
failure→END

要求：
1.同步节点使用node_async。
2.同步Router使用edge_async。
3.不存在wrapNode(Object)。
4.不存在instanceof节点分派。
5.GraphFactory不保存某次运行State。
6.GraphFactory不调用模型或工具。
7.节点名和路由映射集中维护。
8.图编译异常转换为现有AgentFrameworkException和INTERNAL_ERROR，保留cause，不使用无语义RuntimeException。

十三、实现DefaultReactAgentEngine
实现：
DefaultReactAgentEngine implements ReactAgentEngine

依赖：
ReactExecutionValidator
ReactAgentGraphFactory或CompiledGraph

保持纯Java，不添加Spring注解。

职责：

1.execute(definition,task,context)开始时调用ReactExecutionValidator。
2.构造初始messages：
-systemPrompt非空时添加SystemAgentMessage。
-添加UserAgentMessage，content仅使用task.instruction。
-不得把RunContext、安全权限或task.context.toString直接注入消息。
3.构造初始State：
-agentDefinition
-task
-runContext
-messages
-pendingToolCalls=空
-latestToolResults=空
-toolTraces=空
-iteration=0
-finalResult=null
-stopReason=null
-failureMessage=null
-failureErrorCode=null
4.通过ReactAgentGraphFactory构建或取得CompiledGraph。
5.使用当前LangGraph4j版本真实invoke/invokeFinal等API执行。
6.不得向接口调用者暴露CompiledGraph或LangGraph4j类型。
7.执行结束后读取最终State中的finalResult。
8.finalResult为空时抛出INTERNAL_ERROR，不得返回null。
9.图执行异常：
-AgentFrameworkException保留错误码。
-普通Exception转换为INTERNAL_ERROR。
-记录runId、threadId、agentName及服务端异常。
-不得泄露State全文和消息内容。
10.不得实现resume，HITL阶段再实现。
11.不得保存全局可变State。
12.同一Engine实例必须支持并发执行不同runId。
13.不得使用ThreadLocal保存State。

十四、CompiledGraph生命周期
先检查当前LangGraph4j版本CompiledGraph的线程安全及invoke语义。

优先方案：
1.若CompiledGraph可安全并发执行且图不保存运行State：
-在Engine或配置装配阶段编译一次。
-保存为final字段并复用。

2.若当前版本线程安全语义无法确认：
-每次execute由GraphFactory创建独立CompiledGraph。
-在最终输出中说明选择原因。

禁止：
-使用static可变CompiledGraph。
-在CompiledGraph对象中手工存放某次State。
-通过全局锁串行化所有无关runId，除非框架版本确实要求且必须说明。

十五、Spring装配
所有runtime实现保持纯Java，不添加：
@Component
@Service
@Autowired
@Configuration

在agent-infrastructure中新增或完善配置类，装配：

DefaultReactReasonNode
DefaultReactToolExecutionNode
DefaultReactObserveNode
DefaultReactCompleteNode
DefaultReactMaxIterationsNode
DefaultReactFailureNode
ReactRouter
ReactExecutionValidator
ReactAgentGraphFactory
DefaultReactAgentEngine

要求：
1.复用现有ModelInvocationGateway、ToolInvocationGateway和ToolRegistry Bean。
2.使用构造器和@Bean装配。
3.模型相关Bean不存在时，应用仍能正常启动。
4.ReactEngine相关Bean可使用@ConditionalOnBean(ModelInvocationGateway.class)或等价条件。
5.不得创建Fake ModelClient或伪造ModelInvocationGateway。
6.不得在bootstrap启动类中手工new组件。
7.不得产生Bean循环依赖。
8.没有注册Agent时应用仍能启动。
9.本批不注册示例Agent。

十六、错误和安全要求
1.所有失败最终必须产生结构化AgentResult或明确AgentFrameworkException。
2.不得使用null表示失败。
3.日志不得记录：
-完整Prompt
-完整messages
-工具完整参数
-工具完整结果
-API Key
-用户权限全集
4.工具普通失败必须回灌模型，而不是直接终止ReAct。
5.模型失败、State损坏、工具链内部异常才进入FailureNode。
6.不得让SpringAI内部自动执行工具。
7.真实工具执行只能经过ToolInvocationGateway。
8.不得把RunContext发送给LLM。

十七、本批禁止实现
禁止：
1.Supervisor。
2.AgentDispatcher。
3.多Agent编排。
4.示例Agent或AgentProvider。
5.ReAct聊天Controller。
6.Postman开发API。
7.SSE或流式输出。
8.并行工具执行。
9.Session、RBAC、Tool ACL。
10.HITL、interrupt/resume。
11.CheckpointStore内存实现。
12.上下文裁剪、TokenCounter和摘要。
13.长期记忆。
14.RAG、MCP和真实金山云业务。
15.模型重试、熔断和多模型路由。
16.前端页面。
17.为了验收增加固定模型答案或假ToolCall。
18.创建测试脚本。
19.修改README、使用说明、升级记录或验收报告。
20.执行git commit或git push。

十八、代码质量
1.不使用Lombok。
2.使用构造器注入。
3.集合使用不可变快照。
4.节点类职责单一。
5.不使用字段注入。
6.不使用ApplicationContext主动查Bean。
7.不吞异常。
8.不通过Object、instanceof或反射进行节点分派。
9.不创建无意义Facade、Delegate、Manager或Adapter。
10.不复制第二阶段ToolGateway和ModelGateway已有治理逻辑。
11.节点只负责编排和State更新，模型、工具具体技术细节留在现有Gateway和Adapter中。
12.所有代码以实际编译通过为准。

十九、编译验收
完成后执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许，启动agent-bootstrap，确认现有接口不受影响：
GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents

检查：
1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core无LangGraph4j、SpringAI和Spring依赖。
4.agent-runtime可依赖LangGraph4j，但无SpringAI、SpringWeb和agent-infrastructure依赖。
5.ReactAgentGraphFactory不存在wrapNode(Object)。
6.不存在instanceof节点分派。
7.同步节点通过node_async注册。
8.同步Router通过edge_async注册。
9.ReasonNode只调用ModelInvocationGateway一次。
10.ToolExecutionNode只通过ToolInvocationGateway执行工具。
11.ObserveNode正确生成ToolAgentMessage。
12.普通ToolResult失败能够回灌模型。
13.iteration只在Reason模型调用完成后增加。
14.达到maxIterations后不执行新ToolCall。
15.模型在最后一次允许的Reason中完成时可正常结束。
16.finalResult在所有终止路径均不为空。
17.应用无模型配置时仍能启动。
18.没有实现本批范围外功能。
19.git diff中无无关修改。

二十、最终输出
只输出：
1.新增和修改文件清单。
2.六个节点的职责和依赖。
3.完整ReAct状态流转。
4.iteration和maxIterations边界语义。
5.ToolCall与ToolResult配对方式。
6.普通工具失败如何回灌模型。
7.ReactRouter最终路由优先级。
8.GraphFactory使用的LangGraph4j原生API。
9.CompiledGraph生命周期及线程安全选择。
10.SpringBean装配和条件装配方式。
11.编译、打包和启动验证结果。
12.发现但未处理的非本批问题。

编译或打包失败时继续修复，直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE3_BATCH2_END===