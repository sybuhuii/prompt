你正在继续开发一个基于SpringAI+LangGraph4j的通用Agent框架。第一阶段已完成Maven多模块骨架、agent-core核心抽象、Agent/Tool注册中心、运行时扩展接口及SpringBoot装配。

当前只执行第二阶段第1批：实现通用Tool执行引擎和拦截器链。本批不实现内置工具、SpringAI模型调用、ReAct、Supervisor、权限、HITL、记忆和持久化。

一、执行原则 
1.先完整检查当前目录、全部pom.xml及第一阶段已有代码，以现有接口签名和包结构为准增量实现。
2.不要重复创建已有类型，不要创建职责相同但名称不同的第二套接口。
3.发现第一阶段接口存在阻断本批实现的问题时，只做最小兼容修改并说明原因，禁止大规模重构。
4.继续遵循模块化单体架构：
agent-core：领域模型和稳定接口
agent-runtime：Tool执行规则和运行机制
agent-infrastructure：SpringBean装配及具体技术适配
agent-application：应用用例
agent-api：HTTP入口
agent-bootstrap：启动组装
5.agent-runtime不得依赖SpringAI、LangGraph4j、SpringWeb或具体业务。
6.本批不要生成测试脚本，不要生成或修改README、说明、升级记录、验收报告等文档。

二、本批目标
实现以下统一执行链：

ToolInvocation
→ToolInvocationGateway
→ToolInterceptor链
→TerminalToolExecutor
→ToolRegistry查找AgentTool
→AgentTool.execute
→ToolResult

后续ACL、HITL、限流、幂等、审计都必须能插入同一拦截器链，任何上层代码不得绕过ToolInvocationGateway直接执行工具。

三、实现DefaultToolInvocationGateway
在agent-runtime现有tool包中实现：
DefaultToolInvocationGateway implements ToolInvocationGateway

职责：
1.接收ToolInvocation。
2.校验invocation、toolCall、toolName、runContext等必要对象，非法输入返回结构化失败ToolResult。
3.保存一份按顺序排列的不可变ToolInterceptor列表。
4.每次invoke都创建新的执行链实例，禁止复用带可变下标的链对象，确保线程安全。
5.调用拦截器链，最后进入TerminalToolExecutor。
6.不得直接包含具体工具业务逻辑。
7.不得捕获异常后静默返回空值。

四、实现拦截器排序机制
检查现有ToolInterceptor接口。若没有排序能力，允许进行最小修改，增加框架无关的排序方法，例如：
default int order(){return 0;}

禁止依赖Spring的Ordered或@Order，避免agent-runtime与Spring耦合。

约定：
order值越小越先进入，即处于调用链外层。
建议顺序：
ToolExceptionHandlingInterceptor：-1000
ToolAuditInterceptor：-500
ToolArgumentValidationInterceptor：0

DefaultToolInvocationGateway构造时完成排序，并保存不可变快照。

五、实现DefaultToolExecutionChain
实现线程安全的拦截器链：
DefaultToolExecutionChain implements ToolExecutionChain

要求：
1.持有不可变interceptor列表、terminal chain和当前索引。
2.proceed时：
-仍有拦截器：调用当前拦截器，并向其传入索引+1的新链对象。
-没有拦截器：调用TerminalToolExecutor。
3.禁止把currentIndex设计成会被多线程共享修改的成员变量。
4.禁止使用ThreadLocal保存链位置。
5.同一个Gateway必须能够并发执行多个ToolInvocation。

六、实现TerminalToolExecutor
实现：
TerminalToolExecutor implements ToolExecutionChain

依赖ToolRegistry。

执行过程：
1.根据invocation.toolCall.name从ToolRegistry获取AgentTool。
2.调用AgentTool.execute(invocation)。
3.工具返回null时视为异常，不允许向上传递null。
4.本类只负责最终工具查找和执行，不实现参数校验、权限、审批或审计。
5.找不到工具时使用现有TOOL_NOT_FOUND框架异常，由外层异常拦截器统一转换。

七、实现ToolArgumentValidationInterceptor
职责：
1.根据ToolRegistry找到ToolDefinition。
2.读取ToolDefinition.inputSchema。
3.将ToolCall.arguments转换为JSON对象。
4.根据JSONSchema校验必填字段、字段类型、additionalProperties等约束。
5.校验失败时直接返回ToolResult.failure，不进入后续执行链。
6.使用现有INVALID_ARGUMENT错误码；错误内容应指出主要校验问题，但不要暴露底层堆栈。
7.arguments为null时按空Map处理。
8.inputSchema为空时按无参数工具处理：仅允许空arguments。
9.inputSchema不是合法JSON或Schema配置错误时，记录完整日志，并返回结构化TOOL_EXECUTION_FAILED或INTERNAL_ERROR，不抛出未处理异常。
10.不要自行编写只支持required/type的伪JSONSchema解析器。优先复用当前项目已有JSONSchema能力；若项目没有，可在agent-runtime中增加一个轻量、与Spring无关且兼容当前Java版本的JSONSchema校验依赖。版本统一放到父pom依赖管理，不在子模块散落版本号。
11.不要引入SpringAI工具Schema类型。

八、实现ToolExceptionHandlingInterceptor
职责：
1.作为最外层拦截器捕获整个Tool执行链异常。
2.捕获AgentFrameworkException时，将其错误码和安全错误信息转换为ToolResult.failure。
3.捕获其他异常时：
-日志记录toolName、toolCallId、runId及完整异常堆栈。
-向调用方返回TOOL_EXECUTION_FAILED和通用安全提示。
-不得把堆栈、密钥、文件路径或底层异常细节放入ToolResult.content。
4.Error等JVM严重错误不要随意吞掉。
5.不得返回null。

九、实现ToolAuditInterceptor
本批只实现结构化日志审计，不实现数据库审计。

记录：
toolName
toolCallId
runId
threadId
userId
开始时间
结束时间
耗时
success
errorCode

要求：
1.使用SLF4J。
2.不记录完整原始参数，避免敏感数据泄露；最多记录参数字段名。
3.不记录完整工具返回内容。
4.无论后续成功、返回失败ToolResult或抛出异常，都应尽可能记录结束信息。
5.审计拦截器不能修改业务执行结果。

十、SpringBoot装配
agent-runtime中的实现保持纯Java，不添加@Component。

在agent-infrastructure现有配置类中增量装配：
TerminalToolExecutor
ToolArgumentValidationInterceptor
ToolExceptionHandlingInterceptor
ToolAuditInterceptor
DefaultToolInvocationGateway

要求：
1.复用第一阶段已有ToolRegistry Bean。
2.通过List<ToolInterceptor>注入全部拦截器。
3.没有任何已注册Tool时应用仍可启动。
4.不得在启动类中手工new全部组件。
5.避免重复定义第一阶段已有Registry Bean。
6.不得引入具体业务Tool。

十一、错误处理和返回规范
1.所有可预期失败都返回结构化ToolResult。
2.至少覆盖：
INVALID_ARGUMENT
TOOL_NOT_FOUND
TOOL_EXECUTION_FAILED
INTERNAL_ERROR
3.ToolResult中的metadata使用不可变Map。
4.不得使用null表示执行失败。
5.不得直接将异常对象放入metadata。
6.保持与第一阶段ToolResult实际静态工厂方法兼容；若现有工厂方法不足，只做最小补充。

十二、架构边界
禁止：
1.agent-core依赖agent-runtime或Spring。
2.agent-runtime依赖agent-infrastructure。
3.具体工具自行实现参数校验、ACL、审批等通用逻辑。
4.Controller或ApplicationService直接调用AgentTool.execute。
5.使用SpringAI自动执行工具。
6.实现ToolPermissionInterceptor。
7.实现ToolApprovalInterceptor。
8.实现ReAct循环或Supervisor。
9.实现CalculatorTool、CurrentTimeTool等内置工具，本批留到第二阶段第2批。
10.实现SessionStore、CheckpointStore、MemoryStore内存版。
11.增加开发调用Controller或前端页面。

十三、质量要求
1.所有公共类职责单一。
2.构造参数做必要校验。
3.集合使用不可变快照。
4.执行链必须支持并发。
5.日志中不得输出敏感参数和完整工具结果。
6.不要为追求设计模式创建无实际职责的类。
7.不要修改与本批无关的代码。
8.不要执行git commit或git push。

十四、验收
完成后执行：
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

检查：
1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-runtime没有SpringAI、LangGraph4j、SpringWeb import。
4.SpringBean不存在循环依赖。
5.ToolInvocationGateway和拦截器链完成装配。
6.没有注册Tool时应用仍能启动。
7.没有提前实现本批范围外功能。
8.使用代码审查确认DefaultToolExecutionChain不存在共享可变索引问题。

十五、最终输出
只输出：
1.新增和修改文件清单。
2.Tool调用链执行顺序。
3.各拦截器职责。
4.线程安全实现方式。
5.新增依赖及用途。
6.编译和打包结果。
7.发现但未处理的非本批问题。

编译或打包失败时继续修复，直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE2_BATCH1_END===