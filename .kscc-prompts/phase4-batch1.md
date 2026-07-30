你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven模块化单体架构。
2.agent-core领域模型及稳定接口。
3.AgentRegistry、ToolRegistry和Provider自动注册。
4.ToolInvocationGateway及工具拦截器链。
5.内置工具。
6.ModelInvocationGateway和SpringAI适配。
7.通用ReAct状态图、完整Reason→Act→Observe循环。
8.Sample ReAct Agent及开发验证API。

当前只执行第四阶段第1批：建立通用Supervisor领域定义、注册中心、运行State、节点契约、Router和LangGraph4j父图骨架。

本批不实现真实Supervisor模型调用，不执行子Agent，不实现结果汇总逻辑，不开放Supervisor HTTP接口。

一、阶段目标

为后续实现以下中心调度循环建立稳定骨架：

START
→supervisor_reason
→条件路由
├─DISPATCH→dispatch_agents→aggregate_results→supervisor_reason
├─COMPLETE→complete→END
├─MAX_ITERATIONS→max_iterations_fallback→END
└─FAIL→failure→END

Supervisor负责：
1.分析用户总任务。
2.选择一个或多个专业子Agent。
3.为每个子Agent生成独立AgentTask。
4.接收标准AgentResult。
5.根据已有结果决定继续分派或完成。
6.最终汇总回答。

所有子Agent继续复用现有ReactAgentEngine，不允许Supervisor自行执行专业工具。

二、执行原则

1.先完整检查全部pom.xml以及现有：
-AgentDefinition、AgentTask、AgentResult、AgentProvider
-AgentRegistry、AgentProviderRegistrar
-ReactAgentEngine、ReactAgentState、ReactAgentGraphFactory
-RunContext、RunIdGenerator
-AgentErrorCode、AgentFrameworkException
-Spring配置和模块依赖
2.以当前真实接口、包结构、Java版本和LangGraph4j版本为准增量实现。
3.不得创建第二套AgentTask、AgentResult、RunContext或AgentRegistry。
4.发现前序代码阻断本批时，只做最小兼容修改并说明。
5.不得大规模重构已有ReAct、ModelGateway或ToolGateway。
6.继续遵循：
-agent-core：稳定且框架无关的领域模型和SPI。
-agent-runtime：Supervisor运行规则和LangGraph4j编排。
-agent-infrastructure：Spring装配。
-agent-application：应用用例。
-agent-api：HTTP入口。
-agent-bootstrap：启动和Sample配置。
7.agent-runtime允许使用LangGraph4j，但不得依赖SpringAI、SpringWeb或agent-infrastructure。
8.agent-core不得依赖LangGraph4j或Spring。
9.不要生成测试脚本。
10.不要生成或修改README、说明、升级记录、验收报告。
11.不要执行git commit或git push。

三、Supervisor领域定义

在agent-core创建或复用：

com.ksyun.agent.core.supervisor

实现：

SupervisorDefinition

建议字段：
String name
String description
String systemPrompt
Set<String> memberAgents
int maxIterations

要求：
1.使用不可变record或等价不可变类型。
2.name不能为空。
3.description和systemPrompt按现有核心模型空值规范处理。
4.memberAgents不能为空且至少包含一个Agent名称。
5.memberAgents使用不可变Set。
6.memberAgents中不得出现空名称。
7.maxIterations必须大于0。
8.不包含SpringAI、LangGraph4j或Spring类型。
9.不直接保存AgentDefinition对象，使用Agent名称降低配置耦合。
10.不包含具体金山云业务字段。

实现：

SupervisorProvider

方法：
Collection<SupervisorDefinition> provideSupervisors();

要求：
1.作为未来业务模块批量注册Supervisor的SPI。
2.不得依赖Spring容器。
3.不得直接操作SupervisorRegistry。

四、Supervisor注册中心

在agent-runtime现有registry包中实现：

SupervisorRegistry

建议方法：
void register(SupervisorDefinition definition);
Optional<SupervisorDefinition> find(String name);
SupervisorDefinition getRequired(String name);
Collection<SupervisorDefinition> list();
boolean contains(String name);

实现：
DefaultSupervisorRegistry

要求：
1.使用ConcurrentHashMap。
2.线程安全。
3.名称为空时拒绝注册。
4.同名重复注册明确失败，不得静默覆盖。
5.list返回不可变快照。
6.getRequired找不到时抛AgentFrameworkException。
7.如AgentErrorCode缺少SUPERVISOR_NOT_FOUND，允许做最小补充。
8.不得添加Spring注解。
9.不得访问ApplicationContext。
10.不得保存某次运行状态。

实现：
SupervisorProviderRegistrar

职责：
1.接收SupervisorRegistry及多个SupervisorProvider。
2.注册所有Provider提供的SupervisorDefinition。
3.Provider为空时应用仍可启动。
4.Provider返回null时按空集合处理或明确失败，但不能产生难以定位的空指针。
5.重复名称异常不得吞掉。
6.不得创建Sample Supervisor。

五、Supervisor动作和决策模型

在agent-runtime创建：

com.ksyun.agent.runtime.supervisor

实现枚举：

SupervisorAction
- DISPATCH
- FINISH

本阶段不加入THINK工具，不实现自然语言反思工具。

实现不可变：

SupervisorDecision

建议字段：
SupervisorAction action
List<AgentTask> tasks
String decisionSummary
String finalAnswer

要求：
1.action不能为空。
2.tasks使用不可变List。
3.DISPATCH时tasks不能为空。
4.FINISH时tasks必须为空。
5.FINISH时finalAnswer不能为空。
6.decisionSummary用于保存简洁决策依据，不保存冗长推理过程。
7.不得包含SpringAI类型。
8.不得包含CompiledGraph或节点对象。
9.不得创建反思ToolCall。
10.不得让Supervisor通过ToolInvocationGateway调用子Agent。

六、Supervisor停止原因

实现：

SupervisorStopReason

至少包括：
COMPLETED
MAX_ITERATIONS_REACHED
MODEL_ERROR
AGENT_ERROR
INVALID_STATE

该枚举描述Supervisor图终止原因，不替代RunStatus。

实现：

SupervisorRoute

至少包括：
DISPATCH
COMPLETE
MAX_ITERATIONS
FAIL

七、SupervisorState

实现：

SupervisorAgentState extends LangGraph4j AgentState

要求：
1.放在agent-runtime，不放agent-core。
2.状态值继续存储在AgentState父类Map中。
3.不得在子类中重复声明普通Java状态字段，避免形成双份状态。
4.通过ReactAgentState已经采用的同一种方式提供类型安全访问。
5.如果ReactAgentState使用实例访问方法，则SupervisorAgentState也使用实例访问方法。
6.如果ReactAgentState通过集中StateKeys访问，则沿用相同风格，不创建第二套模式。
7.禁止各节点散落硬编码字符串key。
8.不得保存ChatModel、ModelClient、Registry、Gateway、SpringBean、CompiledGraph或异常对象。

状态至少表达：

SupervisorDefinition supervisorDefinition
AgentTask rootTask
RunContext runContext
List<AgentMessage> supervisorMessages
SupervisorDecision decision
List<AgentTask> pendingTasks
List<AgentResult> latestAgentResults
List<AgentResult> agentResults
int iteration
AgentResult finalResult
SupervisorStopReason stopReason
AgentErrorCode failureErrorCode
String failureMessage

字段语义：
1.supervisorMessages：Supervisor自己的消息历史。
2.pendingTasks：本轮准备分派的子Agent任务。
3.latestAgentResults：本轮子Agent执行结果。
4.agentResults：所有历史子Agent结果。
5.iteration：已经完成的Supervisor模型调用次数。
6.finalResult：整个多Agent任务最终结果。
7.子Agent自己的ReAct消息和工具轨迹不得直接复制到SupervisorState。
8.Supervisor只接收标准AgentResult。

八、StateKeys和StateSchema

实现或复用统一风格：

SupervisorStateKeys
SupervisorStateSchema

至少定义：
SUPERVISOR_DEFINITION
ROOT_TASK
RUN_CONTEXT
SUPERVISOR_MESSAGES
DECISION
PENDING_TASKS
LATEST_AGENT_RESULTS
AGENT_RESULTS
ITERATION
FINAL_RESULT
STOP_REASON
FAILURE_ERROR_CODE
FAILURE_MESSAGE

Channel合并语义：

1.supervisorMessages：追加。
2.agentResults：追加。
3.decision：覆盖。
4.pendingTasks：覆盖。
5.latestAgentResults：覆盖。
6.iteration：覆盖，不使用Integer::sum。
7.finalResult：覆盖。
8.stopReason：覆盖。
9.failureErrorCode：覆盖。
10.failureMessage：覆盖。
11.supervisorDefinition、rootTask、runContext：初始化后保持稳定，使用覆盖Channel。
12.节点返回追加字段时只能返回本轮新增内容，不得返回完整历史。
13.避免追加List后产生嵌套List。
14.集合读取返回不可变快照。
15.null集合按空集合处理。

九、迭代次数语义

统一规定：

iteration表示“已经完成的Supervisor模型调用次数”。

规则：
1.初始iteration=0。
2.每成功完成一次Supervisor模型调用后+1。
3.DispatchNode、AggregateNode和终止节点不得增加iteration。
4.maxIterations读取SupervisorDefinition.maxIterations。
5.最后一次允许的Supervisor调用若生成FINISH，允许正常完成。
6.最后一次允许的Supervisor调用若仍生成DISPATCH，不再执行新任务，进入MAX_ITERATIONS。
7.最多允许执行maxIterations次Supervisor模型调用。
8.不得在Channel中配置Integer::sum后又手动+1。

十、节点契约

在：

com.ksyun.agent.runtime.supervisor.node

定义：

SupervisorReasonNode
SupervisorDispatchNode
SupervisorAggregateNode
SupervisorCompleteNode
SupervisorMaxIterationsNode
SupervisorFailureNode

本批只定义契约，不实现真实业务节点。

强制要求：
1.当前后续节点以同步网关和ReactAgentEngine为基础，优先直接继承：
NodeAction<SupervisorAgentState>
2.不得定义与NodeAction语义重复但不兼容的apply接口。
3.不得使用Object作为节点类型。
4.不得使用instanceof分派节点。
5.不得通过ApplicationContext查Bean。
6.不得添加@Component、Service或Autowired。
7.下一批具体实现通过构造器接收依赖。
8.不得创建返回固定Decision、固定AgentResult或固定答案的假实现。
9.不得在本批调用模型。
10.不得在本批执行子Agent。

十一、SupervisorRouter

实现纯Java：

SupervisorRouter

同步路由优先直接实现或兼容：
EdgeAction<SupervisorAgentState>

路由优先级：

1.存在failureErrorCode，或stopReason属于MODEL_ERROR、AGENT_ERROR、INVALID_STATE：
→FAIL

2.decision.action=FINISH且pendingTasks为空：
→COMPLETE
即使iteration刚好达到maxIterations，也允许正常完成。

3.decision.action=DISPATCH且pendingTasks非空且iteration>=maxIterations：
→MAX_ITERATIONS

4.decision.action=DISPATCH且pendingTasks非空：
→DISPATCH

5.iteration>=maxIterations：
→MAX_ITERATIONS

6.其他无法解释的状态：
→FAIL

要求：
1.Router不调用模型。
2.Router不执行Agent。
3.Router不修改State。
4.不得硬编码最大次数。
5.路由结果确定且可解释。
6.不得加入think_tool逻辑。
7.不得使用Object、反射或instanceof识别状态类型。

十二、节点名称

集中定义：

SupervisorNodeNames

至少包含：
SUPERVISOR_REASON
DISPATCH_AGENTS
AGGREGATE_RESULTS
COMPLETE
MAX_ITERATIONS_FALLBACK
FAILURE

禁止在GraphFactory和其他类中散落节点名字面量。

十三、SupervisorGraphFactory

实现：

SupervisorGraphFactory

通过构造器接收：
SupervisorReasonNode
SupervisorDispatchNode
SupervisorAggregateNode
SupervisorCompleteNode
SupervisorMaxIterationsNode
SupervisorFailureNode
SupervisorRouter

图结构：

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

LangGraph4j原生API强制规则：

1.同步节点实现NodeAction<SupervisorAgentState>。
2.同步节点使用LangGraph4j原生node_async(node)注册。
3.同步Router使用LangGraph4j原生edge_async(...)注册。
4.异步节点若存在则直接注册AsyncNodeAction。
5.异步Router若存在则直接注册AsyncEdgeAction。

禁止：
-wrapNode(Object)
-wrapRouter(Object)
-instanceof节点分派
-反射分派
-自定义node_async等价包装
-GraphFactory主动查找SpringBean
-GraphFactory调用模型或子Agent
-GraphFactory保存某次运行State

GraphFactory只负责：
1.构建Channel。
2.注册节点。
3.注册普通边。
4.注册条件路由。
5.编译图。

图编译异常使用现有AgentFrameworkException和INTERNAL_ERROR，保留cause，不使用无语义RuntimeException。

十四、SupervisorEngine接口

定义：

SupervisorEngine

建议方法：

AgentResult execute(
SupervisorDefinition definition,
AgentTask rootTask,
RunContext context
);

要求：
1.这是多Agent Supervisor统一执行入口。
2.接口不得暴露LangGraph4j类型。
3.本批只定义接口，不实现固定结果假实现。
4.不实现resume。
5.不允许调用方直接操作CompiledGraph。
6.最终结果继续使用现有AgentResult。
7.不建立第二套SupervisorResult。

十五、SupervisorExecutionValidator

实现纯Java：

SupervisorExecutionValidator

依赖：
AgentRegistry

校验：
1.SupervisorDefinition非空。
2.rootTask非空。
3.RunContext非空。
4.rootTask.agentName与SupervisorDefinition.name一致；若现有AgentTask允许空agentName，明确兼容规则。
5.maxIterations大于0。
6.rootTask.instruction非空。
7.memberAgents非空。
8.memberAgents中每个Agent都存在于AgentRegistry。
9.memberAgents不得包含重复名称。
10.不存在的成员Agent使用AGENT_NOT_FOUND。
11.校验失败使用AgentFrameworkException和现有错误码。
12.不调用模型。
13.不执行Agent。
14.不修改输入对象。

十六、子Agent上下文边界

本批只定义规则，不执行子Agent。

后续DispatchNode必须遵循：

1.每个子Agent获得独立AgentTask。
2.每个子Agent使用现有ReactAgentEngine执行。
3.子Agent使用fresh context：
-AgentDefinition.systemPrompt
-Supervisor生成的task.instruction
-必要且明确的task.context
4.不得把Supervisor完整消息历史传给子Agent。
5.不得把其他子Agent的完整ReAct消息传给当前子Agent。
6.不得共享ReactAgentState。
7.子Agent完成后只返回标准AgentResult。
8.Supervisor不得直接访问子Agent内部ToolMessage或SpringAI消息。
9.RunContext中的userId、sessionId、threadId保持父任务身份语义。
10.每个子任务后续应使用独立taskId；runId派生规则留到下一批明确。
11.本批不得实现实际分派。

十七、Spring装配

本批装配：

DefaultSupervisorRegistry
SupervisorProviderRegistrar
SupervisorExecutionValidator

要求：
1.runtime实现保持纯Java，不添加Spring注解。
2.在agent-infrastructure通过@Configuration和@Bean装配。
3.复用已有AgentRegistry。
4.没有SupervisorProvider时应用仍可启动。
5.不得创建Sample Supervisor。
6.具体Supervisor节点尚未实现时，不强行装配无法使用的SupervisorGraphFactory或SupervisorEngine。
7.不得创建Fake节点或Fake SupervisorEngine。
8.不得修改bootstrap启动类手工new组件。
9.不得产生Bean循环依赖。
10.无模型配置时应用仍能启动。

十八、本批禁止实现

禁止：
1.真实Supervisor模型调用。
2.SupervisorDecision JSON解析。
3.子Agent执行。
4.ReactAgentEngine调用。
5.结果聚合逻辑。
6.Sample Supervisor。
7.Supervisor开发Controller。
8.Supervisor Postman接口。
9.子Agent并行执行。
10.Supervisor调用业务Tool。
11.think_tool。
12.Supervisor作为SpringAI自动工具。
13.登录、Session和RBAC。
14.Tool ACL。
15.HITL、interrupt/resume。
16.CheckpointStore内存实现。
17.上下文裁剪和摘要。
18.长期记忆。
19.RAG、MCP和真实金山云业务。
20.SSE和前端。
21.模型重试、熔断或多模型路由。
22.为了编译创建固定结果节点。
23.测试脚本。
24.README、说明、报告。
25.git commit和git push。

十九、代码质量

1.不使用Lombok。
2.优先record、enum、final类和构造器依赖。
3.集合使用不可变快照。
4.不使用字段注入。
5.不使用ApplicationContext主动查Bean。
6.不使用static可变State。
7.不使用ThreadLocal保存SupervisorState。
8.不返回null表示失败。
9.不通过Object、instanceof或反射进行节点分派。
10.不创建无意义Facade、Delegate、Manager或Adapter。
11.不复制现有AgentRegistry、ReactEngine或ModelGateway能力。
12.GraphFactory和Router职责单一。
13.所有代码以实际编译结果为准。

二十、编译验收

完成后执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许启动应用，并确认已有接口不受影响：

GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents

检查：
1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core无LangGraph4j、SpringAI和Spring依赖。
4.agent-runtime可依赖LangGraph4j，但无SpringAI、SpringWeb和agent-infrastructure依赖。
5.SupervisorRegistry线程安全。
6.重复Supervisor注册明确失败。
7.没有Provider时应用正常启动。
8.SupervisorState没有重复Java字段。
9.StateKeys集中维护。
10.Channel合并语义正确。
11.iteration为覆盖语义。
12.SupervisorGraphFactory不存在wrapNode(Object)。
13.不存在instanceof节点分派。
14.同步节点使用node_async。
15.同步Router使用edge_async。
16.没有实现真实模型调用或子Agent执行。
17.没有Sample Supervisor或HTTP入口。
18.现有ReAct功能未被破坏。
19.git diff没有无关修改。

二十一、最终输出

只输出：
1.新增和修改文件清单。
2.SupervisorDefinition字段及约束。
3.Supervisor注册流程。
4.SupervisorAgentState字段及作用。
5.各State字段Channel合并语义。
6.iteration定义和边界。
7.SupervisorRouter路由优先级。
8.SupervisorGraphFactory图结构。
9.节点契约和下一批实现内容。
10.子Agent fresh context边界。
11.SpringBean装配方式。
12.编译、打包和启动验证结果。
13.发现但未处理的非本批问题。

编译或打包失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE4_BATCH1_END===