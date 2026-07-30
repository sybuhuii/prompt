你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven多模块工程及模块边界。
2.agent-core领域模型和接口。
3.AgentRegistry、ToolRegistry及Provider自动注册。
4.ToolInvocationGateway、线程安全拦截器链、参数校验、异常处理和审计。
5.CalculatorTool、CurrentTimeTool、EchoTool、TextSearchTool及BuiltinToolProvider。

当前只执行第二阶段第3批：接入SpringAI，实现框架ModelClient适配、ModelInvocationGateway具体实现、消息和工具定义转换，并提供受配置控制的单次模型调用开发接口。

本批只完成“一次模型调用”。模型可以返回普通文本或ToolCall，但不得自动执行工具，不得实现ReAct循环和Supervisor。

一、执行原则
1.先完整检查当前仓库、全部pom.xml、agent-core模型、agent-runtime模型接口、agent-infrastructure配置及agent-api现有代码。
2.以现有类型、接口签名、错误码、包结构和SpringBoot版本为准增量实现。
3.不要创建职责相同但名称不同的第二套ModelClient、ModelRequest、ModelResponse、ToolCall或消息模型。
4.发现前两批存在阻断本批的问题时，只做最小兼容修复，并在最终输出中说明。
5.继续遵循架构：
-agent-core：框架无关的领域模型和稳定接口。
-agent-runtime：模型调用规则和网关实现，保持纯Java。
-agent-infrastructure：SpringAI适配和SpringBean装配。
-agent-application：单次模型调用用例协调。
-agent-api：仅提供受配置控制的开发验证接口。
-agent-bootstrap：启动组装，不写业务逻辑。
6.不要生成测试脚本。
7.不要生成或修改README、使用说明、升级记录、验收报告等文档。
8.不要执行git commit或git push。

二、本批完成后的调用链
实现：

HTTP开发接口
→ModelDevApplicationService
→ModelInvocationGateway
→ModelClient
→SpringAiModelClient
→SpringAI ChatModel
→模型响应
→框架ModelResponse

模型响应分为：
1.普通Assistant文本，toolCalls为空。
2.Assistant消息携带一个或多个ToolCall。

本批不得继续执行ToolCall。工具执行仍只能由已有ToolInvocationGateway负责，第三阶段ReAct引擎再把两条链路连接起来。

三、SpringAI依赖
1.检查当前SpringBoot版本和已有依赖，选择与其兼容的稳定SpringAI版本。
2.SpringAI版本统一放在根pom的dependencyManagement/BOM中，不在子模块散落版本号。
3.SpringAI相关依赖只允许添加到agent-infrastructure；如agent-bootstrap仅为启动所必需，可通过模块传递依赖获得，不要在多个模块重复声明。
4.优先基于SpringAI的ChatModel低层接口适配，不要让agent-core或agent-runtime直接依赖ChatClient、ChatModel、Prompt、Message、ToolCallback等SpringAI类型。
5.如项目未确定模型供应商，使用SpringAI支持OpenAI兼容接口的模型Starter，使base-url、api-key、model可通过环境变量配置。
6.不要硬编码API Key、模型地址或模型名称。
7.不得把真实密钥写入application.yml、代码、日志或Git跟踪文件。
8.添加依赖后必须以实际编译结果为准调整API，禁止照抄不兼容版本的示例代码。

四、实现DefaultModelInvocationGateway
在agent-runtime现有model包中实现：

DefaultModelInvocationGateway implements ModelInvocationGateway

依赖：
ModelClient

职责：
1.校验ModelRequest和RunContext。
2.messages为null或空时，抛出或转换为现有INVALID_ARGUMENT框架异常。
3.调用ModelClient.generate(request)。
4.ModelClient返回null时转换为MODEL_INVOCATION_FAILED。
5.捕获AgentFrameworkException并保留原错误码。
6.捕获普通Exception，记录安全日志并转换为MODEL_INVOCATION_FAILED。
7.日志记录：
-runId
-threadId
-userId
-消息数量
-工具定义数量
-开始时间
-耗时
-是否成功
-错误码
8.不得记录完整Prompt、完整消息内容、API Key、模型响应全文或工具参数。
9.保持普通Java实现，不添加@Component、@Service、@Autowired等Spring注解。
10.构造器校验依赖非空。
11.不得实现重试、熔断、模型路由、上下文裁剪、记忆注入或Token预算，本批只预留以后扩展位置。

五、实现SpringAiMessageMapper
在agent-infrastructure创建或复用包：

com.ksyun.agent.infrastructure.springai

实现：
SpringAiMessageMapper

职责：
将框架AgentMessage转换为当前SpringAI版本对应的消息类型。

必须支持：
1.SystemAgentMessage→SpringAI SystemMessage。
2.UserAgentMessage→SpringAI UserMessage。
3.AssistantAgentMessage→SpringAI AssistantMessage。
4.AssistantAgentMessage中的ToolCall必须转换为SpringAI可识别的assistant tool call结构。
5.ToolAgentMessage→SpringAI ToolResponseMessage或当前版本对应的工具结果消息。
6.保持toolCallId、toolName、content和error语义。
7.不允许通过toString粗暴转换。
8.遇到不支持的消息类型时抛出清晰框架异常。
9.Mapper保持无状态和线程安全。
10.不得把RunContext转换成LLM消息。
11.userId、roles、permissions、sessionId等运行时安全信息不得自动注入Prompt。

若当前SpringAI版本对Assistant ToolCall或ToolResponseMessage构造方式不同，应按实际API正确适配并保证编译通过。

六、实现SpringAiToolMapper
实现：
SpringAiToolMapper

职责：
将框架ToolDefinition转换成当前SpringAI版本能够发送给模型的工具定义。

要求：
1.保留name、description和inputSchema。
2.inputSchema必须作为JSONSchema使用，不能转成普通描述文字。
3.只向模型暴露ModelRequest.tools中明确传入的工具。
4.不得自动把ToolRegistry中的全部工具发送给模型。
5.不得向模型暴露requiredPermission、内部Java类名或其他安全策略信息。
6.不得在Mapper中查找或执行AgentTool。
7.不得创建可以直接绕过ToolInvocationGateway执行AgentTool的回调。
8.如SpringAI必须通过ToolCallback描述工具，回调只能用于声明Schema，并且必须确保SpringAI内部工具自动执行被关闭。
9.如果当前SpringAI版本无法在不提供Callback的情况下声明工具，创建一个明确的“禁止执行”定义适配器；即使配置错误导致内部执行，也必须快速失败，绝不能调用真实AgentTool。
10.Mapper保持无状态和线程安全。

七、严格关闭SpringAI内部工具自动执行
这是本批最重要的架构要求。

必须实现：
1.SpringAI只负责把工具Schema发送给模型。
2.SpringAI只负责返回模型生成的ToolCall。
3.SpringAI不得调用CalculatorTool、CurrentTimeTool、EchoTool、TextSearchTool或任何AgentTool。
4.真实工具执行只能走：
ToolInvocationGateway
→拦截器链
→TerminalToolExecutor
→AgentTool.execute

根据当前SpringAI版本，使用官方支持的“关闭内部工具执行”配置或API，例如对应版本中的internalToolExecutionEnabled(false)或等价机制。

不得仅依赖“模型应该不会执行”的假设。

完成后通过代码审查确认：
-SpringAiModelClient中没有AgentTool.execute调用。
-SpringAiToolMapper中没有AgentTool.execute调用。
-没有SpringAI ToolCallback调用ToolRegistry或ToolInvocationGateway。
-一次模型调用返回ToolCall后立即结束，不继续调用模型。

八、实现SpringAiResponseMapper
实现：
SpringAiResponseMapper

职责：
将SpringAI返回结果转换为框架ModelResponse。

要求：
1.提取Assistant文本内容。
2.提取全部ToolCall，不只取第一个。
3.每个ToolCall转换为框架ToolCall：
-id
-name
-arguments
4.ToolCall arguments必须解析成Map<String,Object>，不能把JSON参数字符串原样塞进错误类型字段。
5.工具参数为空时使用不可变空Map。
6.保留模型生成的toolCallId；若供应商未返回ID，生成当前响应内唯一且稳定的替代ID并在metadata标记。
7.提取TokenUsage：
-inputTokens
-outputTokens
-totalTokens
8.供应商未返回Token用量时使用安全默认值，并在metadata标记usageUnavailable，不得伪造真实Token数。
9.可在metadata保存非敏感信息：
-model
-finishReason
-responseId
-usageUnavailable
10.metadata必须使用不可变Map。
11.不得把底层SpringAI对象直接放进metadata。
12.不得把完整响应、Prompt或密钥写入metadata。
13.没有文本但存在ToolCall属于合法响应。
14.既无文本也无ToolCall时，返回安全的空Assistant消息或MODEL_INVOCATION_FAILED，按照现有领域模型选择一致方案并说明。

九、实现SpringAiModelClient
实现：

SpringAiModelClient implements ModelClient

依赖：
ChatModel
SpringAiMessageMapper
SpringAiToolMapper
SpringAiResponseMapper

职责：
1.接收框架ModelRequest。
2.转换消息。
3.转换ModelRequest.tools。
4.根据ModelRequest.options应用允许的模型参数。
5.构造SpringAI Prompt/ChatOptions。
6.明确关闭内部工具自动执行。
7.调用ChatModel。
8.将结果转换为框架ModelResponse。
9.模型异常转换为AgentFrameworkException，错误码MODEL_INVOCATION_FAILED。
10.不得吞异常或返回null。
11.不得直接使用ToolRegistry。
12.不得执行AgentTool。
13.不得包含ReAct循环。
14.不得在类中硬编码模型供应商、model名称、temperature或API Key。
15.类本身保持普通Java类，不添加@Component；由infrastructure配置类创建Bean。

十、ModelRequest.options处理
检查现有ModelRequest.options字段。

本批只允许支持少量安全、通用选项，例如：
model
temperature
maxTokens

要求：
1.未提供时使用SpringAI配置中的默认值。
2.类型错误时返回INVALID_ARGUMENT。
3.temperature限制在合理范围。
4.maxTokens必须为正数并设置合理上限。
5.忽略或拒绝未知选项，行为必须明确且有日志。
6.不得允许客户端通过options传入：
-apiKey
-baseUrl
-proxy
-任意Java类名
-ToolCallback
-内部工具执行开关
7.如果当前SpringAI版本无法在单次请求覆盖某选项，可安全忽略并在metadata说明，不要使用反射修改内部对象。

十一、SpringBean装配
在agent-infrastructure新增或完善配置类，例如：

SpringAiModelConfiguration

负责创建：
SpringAiMessageMapper
SpringAiToolMapper
SpringAiResponseMapper
SpringAiModelClient
DefaultModelInvocationGateway

要求：
1.DefaultModelInvocationGateway实现位于agent-runtime，配置类位于agent-infrastructure。
2.使用@Configuration和@Bean。
3.不得在agent-bootstrap启动类中手工new这些对象。
4.避免Bean循环依赖。
5.ChatModel不存在时，应用仍应能够启动。
6.可使用@ConditionalOnBean(ChatModel.class)、@ConditionalOnProperty或等价方式控制模型相关Bean。
7.模型未配置时，不影响：
GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents
8.不得创建假的ModelClient返回伪造答案。
9.若使用条件装配，缺少模型配置时开发调用接口应返回明确的“模型未配置/不可用”，而不是NullPointerException。
10.不要启用SpringAI自动工具执行。

十二、配置与密钥
在不泄露密钥的前提下完善必要配置。

建议属性：
agent.model.enabled
agent.dev-api.enabled

模型供应商配置使用SpringAI当前版本的标准属性，并允许通过环境变量提供，例如：
MODEL_API_KEY
MODEL_BASE_URL
MODEL_NAME

具体属性名应按实际SpringAI Starter要求实现。

要求：
1.默认情况下没有密钥也能启动应用。
2.不要写真实密钥。
3.不要创建包含真实密钥的.env。
4.不要覆盖用户已有本地配置。
5.不要提交application-local.yml。
6.正式application.yml只允许包含安全默认值和环境变量占位符。
7.日志不得打印API Key。
8.如果项目已有配置命名规范，沿用现有规范。

十三、Application单次调用服务
在agent-application创建或复用：

ModelDevApplicationService

职责：
1.只负责开发验证用的一次模型调用。
2.依赖ModelInvocationGateway和ToolRegistry抽象。
3.接收：
-userMessage
-toolNames
-可选options
4.校验userMessage非空。
5.根据toolNames从ToolRegistry获取ToolDefinition。
6.toolNames为空时不向模型提供工具。
7.不存在的工具名称返回TOOL_NOT_FOUND。
8.构造框架ModelRequest。
9.构造固定的开发RunContext或由API内部生成：
-userId=dev-user
-sessionId=dev-session
-threadId自动生成或固定开发值
-runId由RunIdGenerator生成
-roles和permissions为空
10.不得允许客户端提交userId、roles、permissions、sessionId。
11.调用ModelInvocationGateway一次。
12.不得执行返回的ToolCall。
13.不得再次调用模型。
14.不得实现ReAct循环。
15.该服务是开发验证用例，不作为未来正式Agent业务入口。

如果当前架构中Application层不能直接依赖ToolRegistry实现，应依赖现有ToolRegistry接口，不得直接访问Spring ApplicationContext。

十四、开发验证API
在agent-api提供受配置控制的开发接口：

POST /api/dev/model/invoke

仅当：
agent.dev-api.enabled=true
时启用。

建议请求DTO：
{
"message":"帮我计算12*(3+5)",
"toolNames":["calculator"],
"options":{
"temperature":0
}
}

响应DTO建议包含：
{
"runId":"...",
"content":"...",
"toolCalls":[
{
"id":"...",
"name":"calculator",
"arguments":{
"expression":"12*(3+5)"
}
}
],
"tokenUsage":{
"inputTokens":0,
"outputTokens":0,
"totalTokens":0
},
"metadata":{}
}

要求：
1.Controller只调用ModelDevApplicationService。
2.Controller不得直接注入ChatModel、ChatClient、ModelClient或ToolRegistry。
3.Controller不得执行ToolCall。
4.Controller不得允许客户端传入RunContext、安全身份或自定义工具Schema。
5.使用DTO，不直接暴露SpringAI对象。
6.不暴露完整systemPrompt。
7.模型未启用或未配置时返回明确的HTTP 503或统一业务错误。
8.参数错误返回HTTP 400或项目现有统一异常格式。
9.开发API关闭时接口不存在或返回404。
10.不要新增公开的通用生产模型调用接口。
11.不要新增ReAct聊天接口。
12.如果项目已有dev-api配置类和Controller结构，增量复用，不建立第二套。

十五、可选system消息
开发验证时可在ModelDevApplicationService内部使用固定且简短的SystemAgentMessage，例如：
“You are a tool-aware assistant. Return a tool call when a provided tool is required.”

要求：
1.不要允许客户端任意提交systemPrompt。
2.不要包含权限、userId、sessionId等安全上下文。
3.不要要求模型伪造工具执行结果。
4.当需要工具时，只让模型返回ToolCall，不声称工具已经执行。

十六、错误处理
至少覆盖：
INVALID_ARGUMENT
TOOL_NOT_FOUND
MODEL_INVOCATION_FAILED
INTERNAL_ERROR

要求：
1.SpringAI底层异常不能原样返回客户端。
2.客户端响应不得包含堆栈、API Key、baseUrl中的敏感查询参数或底层实现类。
3.后端日志记录完整异常堆栈，但不得记录密钥和完整Prompt。
4.ModelClient、Mapper和Gateway不得返回null。
5.TokenUsage和metadata使用安全默认值及不可变结构。
6.如果模型供应商返回无法解析的ToolCall参数，应返回MODEL_INVOCATION_FAILED或将该ToolCall标记为解析失败，行为必须清晰一致。

十七、架构边界
禁止：
1.agent-core依赖SpringAI。
2.agent-runtime依赖SpringAI、SpringWeb或agent-infrastructure。
3.agent-application直接依赖SpringAI。
4.agent-api直接调用ChatModel或ChatClient。
5.SpringAI自动执行工具。
6.SpringAiModelClient直接调用AgentTool。
7.Controller直接调用AgentTool.execute。
8.实现Reason→Act→Observe循环。
9.实现Supervisor、AgentDispatcher或多Agent编排。
10.实现ACL、RBAC、HITL或Checkpoint。
11.实现上下文裁剪、摘要和长期记忆。
12.实现RAG、MCP或真实金山云业务。
13.实现模型重试、熔断、负载均衡或多模型路由。
14.为了验证而添加假的模型结果。
15.修改与本批无关的内置工具逻辑。
16.新增前端页面。

十八、Postman验收场景
完成后，在模型配置可用且agent.dev-api.enabled=true时，使用Postman验证。

场景1：普通文本回答
POST /api/dev/model/invoke
{
"message":"你好，请只回复你好",
"toolNames":[]
}

预期：
-content有普通文本。
-toolCalls为空。
-只调用模型一次。

场景2：模型返回ToolCall
POST /api/dev/model/invoke
{
"message":"请使用calculator计算12*(3+5)，不要自己计算",
"toolNames":["calculator"],
"options":{
"temperature":0
}
}

预期：
-toolCalls中包含calculator。
-arguments中包含expression。
-接口不返回真实计算结果96，除非模型文本自行错误声称；框架不得执行calculator。
-后端日志中没有ToolInvocationGateway执行记录。

场景3：未知工具
{
"message":"测试",
"toolNames":["not_exist"]
}

预期：
返回TOOL_NOT_FOUND。

场景4：模型未配置
agent.model.enabled=false或没有ChatModel。

预期：
应用正常启动。
框架health/tools/agents接口正常。
模型开发接口返回503或明确“模型不可用”。

场景5：关闭开发API
agent.dev-api.enabled=false。

预期：
POST /api/dev/model/invoke不可访问。

不要为了验收创建测试脚本或临时假实现。

十九、编译和启动验收
完成后执行：
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

检查：
1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core和agent-runtime没有SpringAI import。
4.agent-application没有SpringAI import。
5.Spring Bean无循环依赖。
6.无模型配置时应用仍能启动。
7.已有health/tools/agents接口不受影响。
8.开启模型配置后单次模型调用可用。
9.模型能返回普通文本。
10.模型能返回一个或多个ToolCall。
11.ToolCall未被SpringAI或本批代码自动执行。
12.一次请求只发生一次模型调用。
13.没有提前实现ReAct或Supervisor。
14.没有真实密钥进入Git跟踪文件。
15.检查git diff，确认未修改与本批无关的代码。

二十、最终输出
只输出：
1.新增和修改文件清单。
2.SpringAI依赖版本和选择依据。
3.模块依赖是否发生变化。
4.消息转换流程。
5.工具定义转换及关闭自动执行的实现方式。
6.模型响应和ToolCall转换流程。
7.ModelInvocationGateway职责。
8.开发API调用方式。
9.所需环境变量，不包含真实值。
10.编译、打包及启动验证结果。
11.Postman验证结果；无法调用真实模型时说明准确原因，不得伪造。
12.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE2_BATCH3_END===