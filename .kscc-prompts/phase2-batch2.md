你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven多模块架构。
2.agent-core领域模型和接口。
3.AgentRegistry、ToolRegistry及Provider自动注册。
4.ToolInvocationGateway、拦截器链、参数校验、异常处理和审计。

当前只执行第二阶段第2批：实现通用内置工具及ToolProvider注册，验证工具注册链路。本批不实现SpringAI模型调用、ReAct、Supervisor、权限、HITL、记忆或持久化。

一、执行原则
1.先检查全部pom.xml、agent-core中的Tool相关模型、agent-runtime工具执行链、agent-infrastructure配置及现有Provider注册逻辑。
2.以现有接口签名、包名和错误码为准增量实现，不重复创建同职责类型。
3.发现上一批存在阻断问题时仅做最小修复，并在最终结果中说明。
4.继续遵循最新架构：
-agent-core：稳定领域模型和接口。
-agent-runtime：通用工具执行规则。
-agent-infrastructure：内置工具和Spring装配。
-agent-api：对外接口。
-agent-bootstrap：启动组装。
5.内置工具不得放入agent-runtime或agent-core。
6.工具实现保持普通Java类，不添加@Component、@Service。
7.由agent-infrastructure中的@Configuration和@Bean完成Spring装配。
8.不要生成测试脚本。
9.不要生成或修改README、使用说明、升级记录、验收报告等文档。
10.不要执行git commit或git push。

二、本批目标
在agent-infrastructure中实现以下低风险内置工具：
1.CalculatorTool
2.CurrentTimeTool
3.EchoTool
4.TextSearchTool
5.BuiltinToolProvider

完成后应通过已有ToolProviderRegistrar自动注册，GET /api/framework/tools能够查询到工具元信息。

三、目录建议
在agent-infrastructure中创建或复用：
com.ksyun.agent.infrastructure.tool.builtin

建议文件：
CalculatorTool.java
CurrentTimeTool.java
EchoTool.java
TextSearchTool.java
BuiltinToolProvider.java
BuiltinToolConfiguration.java

如当前项目已有更合理且职责一致的包结构，可沿用现有结构，不要重复创建第二套目录。

四、通用工具要求
所有工具必须：
1.实现agent-core中的AgentTool。
2.通过definition()返回完整ToolDefinition。
3.通过execute(ToolInvocation)返回ToolResult。
4.不得返回null。
5.不得直接抛出可预期的参数异常。
6.参数错误返回ToolResult.failure，使用现有INVALID_ARGUMENT错误码。
7.执行失败返回TOOL_EXECUTION_FAILED。
8.metadata使用不可变Map。
9.ToolDefinition中的inputSchema必须是合法JSONSchema。
10.JSONSchema类型统一为：
{
"type":"object",
"properties":{...},
"required":[...],
"additionalProperties":false
}
11.工具名称使用稳定的小写snake_case。
12.工具风险等级均为LOW。
13.当前不设置真实业务权限；requiredPermission为空字符串或使用项目现有公共权限约定，不得自行引入RBAC逻辑。
14.工具不得自行实现ACL、审批、审计、限流等横切逻辑。
15.不得绕过ToolInvocationGateway设计其他执行入口。
16.不得调用Shell、PowerShell、cmd或脚本引擎。
17.不得读取本地文件、环境变量或系统敏感信息。

五、CalculatorTool
工具名称：
calculator

描述：
执行安全的基础算术表达式计算。

输入：
expression:string，必填。

支持：
+、-、*、/
圆括号
整数
小数
一元正号和负号
表达式中的普通空白

要求：
1.禁止使用ScriptEngine、JavaScript引擎、SpEL、OGNL、MVEL、反射或Runtime.exec。
2.禁止使用eval类能力。
3.实现安全的表达式词法解析和计算逻辑，推荐递归下降或等价安全算法。
4.使用BigDecimal进行计算，避免明显浮点精度问题。
5.正确处理运算优先级和括号。
6.除法使用合理精度和舍入策略，并避免无限小数异常。
7.除数为0时返回INVALID_ARGUMENT。
8.非法字符、括号不匹配、表达式不完整时返回INVALID_ARGUMENT。
9.expression为空或纯空白时返回INVALID_ARGUMENT。
10.限制表达式长度，例如不超过1000字符，防止异常资源消耗。
11.输出应是规范数字字符串，尽量去除无意义尾随0。
12.不要支持变量、函数、幂运算、科学函数或任意代码执行。

示例：
12*(3+5)→96
-2.5+4→1.5

六、CurrentTimeTool
工具名称：
current_time

描述：
查询指定IANA时区的当前时间。

输入：
timezone:string，可选。

要求：
1.timezone为空或未提供时默认UTC。
2.使用java.time.ZoneId校验时区。
3.使用ZonedDateTime获取时间。
4.输出使用ISO_OFFSET_DATE_TIME格式。
5.非法时区返回INVALID_ARGUMENT。
6.metadata可包含实际使用的timezone，但必须使用不可变Map。
7.不得根据LLM或用户输入修改JVM默认时区。
8.不得依赖外部时间API或网络请求。

示例时区：
UTC
Asia/Shanghai
Asia/Tokyo

七、EchoTool
工具名称：
echo

描述：
原样返回输入文本，用于验证工具调用链。

输入：
text:string，必填。

要求：
1.text为null时返回INVALID_ARGUMENT。
2.允许空字符串，但需明确保持原样。
3.限制最大长度，例如4000字符，超限返回INVALID_ARGUMENT。
4.返回内容与输入一致。
5.不得对文本执行命令、模板、表达式或代码解析。
6.不得在日志中输出完整文本内容。

八、TextSearchTool
工具名称：
text_search

描述：
在用户提供的文本中搜索关键词并返回匹配行。

输入：
text:string，必填。
keyword:string，必填。
caseSensitive:boolean，可选，默认false。
maxMatches:integer，可选，默认20，范围1至100。

要求：
1.仅搜索输入参数中的text，不读取本地文件。
2.按换行拆分文本。
3.返回匹配行号和匹配行内容。
4.行号从1开始。
5.caseSensitive=false时执行大小写不敏感匹配。
6.keyword为空时返回INVALID_ARGUMENT。
7.text为null时返回INVALID_ARGUMENT。
8.maxMatches越界时返回INVALID_ARGUMENT。
9.匹配数量达到maxMatches后停止。
10.限制text总长度，例如不超过100000字符。
11.限制单条返回行长度，例如超过500字符时截断并标记。
12.限制最终content总长度，避免工具结果无限增大。
13.无匹配时返回成功结果，内容明确表示未找到匹配项。
14.metadata可包含matchCount、truncated等非敏感信息。
15.不得使用Shell grep命令。
16.不得直接把完整原始text写入日志或metadata。

建议输出格式：
1:第一条匹配内容
8:第二条匹配内容

九、参数读取
检查当前ToolCall.arguments实际类型。

要求：
1.统一从ToolInvocation.toolCall().arguments()读取参数。
2.不能假设参数一定存在或类型一定正确。
3.提供小型、职责明确的参数读取辅助方法，避免4个工具重复大量强制转换代码。
4.辅助类放在builtin内部并保持包级或final，不要污染agent-core。
5.布尔值和整数支持JSON反序列化后常见Java类型，但不得进行危险或模糊转换。
6.所有参数校验错误使用安全、明确的信息。
7.不要把底层ClassCastException直接暴露给调用方。

十、BuiltinToolProvider
实现：
BuiltinToolProvider implements ToolProvider

要求：
1.通过构造器接收内置工具集合，或由配置类显式传入4个工具。
2.provideTools()返回不可变集合。
3.不得每次调用provideTools()都重新创建工具。
4.不得返回null。
5.不得在Provider内部访问ApplicationContext。
6.不得直接调用ToolRegistry.register。
7.注册行为继续由第一阶段已有ToolProviderRegistrar负责。
8.工具名称必须唯一。
9.不要通过启动类手工注册工具。

十一、Spring装配
在agent-infrastructure中新增或完善BuiltinToolConfiguration。

要求：
1.使用@Configuration和@Bean创建4个工具及BuiltinToolProvider。
2.如果将各工具声明为AgentTool Bean，确认ProviderRegistrar不会重复扫描并重复注册。
3.优先确保所有工具只通过BuiltinToolProvider注册一次。
4.复用已有ToolProviderRegistrar自动注册流程。
5.不要修改agent-bootstrap启动类进行手工注册。
6.没有SpringAI配置和模型密钥时，应用仍应正常启动。
7.不要增加具体业务Agent或业务Tool。
8.避免Bean循环依赖。
9.若当前Provider自动注册依赖ApplicationRunner、SmartInitializingSingleton或等价机制，应沿用现有方式，不要重写第二套注册系统。

十二、工具元信息API
检查已有：
GET /api/framework/tools

要求：
1.启动后能查询到：
calculator
current_time
echo
text_search
2.不能暴露工具Java实现类名。
3.不能暴露完整内部异常信息。
4.是否暴露inputSchema应遵循当前DTO设计；若当前接口不暴露Schema，不要为本批强行修改。
5.不要新增直接执行任意工具的公共HTTP接口。
6.不要新增允许用户自行构造RunContext、userId或权限的接口。

十三、依赖要求
1.优先使用JDK标准库完成4个工具。
2.不要引入表达式执行框架。
3.不要引入Shell工具库。
4.不要引入Apache Commons等仅为少量字符串处理服务的大依赖。
5.如无需新增依赖，不要修改pom。
6.新增依赖必须说明用途，并在父pom统一管理版本。
7.不得引入SpringAI、LangGraph4j相关依赖，本批尚未进行模型适配。

十四、架构边界
禁止：
1.修改agent-core使其依赖Spring。
2.修改agent-runtime使其依赖agent-infrastructure。
3.将内置工具放入agent-core。
4.在工具内部实现权限校验。
5.在工具内部实现人工审批。
6.在工具内部实现审计日志。
7.Controller直接调用AgentTool.execute。
8.实现ReAct循环。
9.实现Supervisor。
10.实现SpringAI模型调用。
11.实现SessionStore、CheckpointStore、MemoryStore内存版。
12.实现RAG、MCP、前端或真实金山云业务。
13.增加与本批无关的复杂抽象或插件系统。

十五、代码质量
1.所有类职责单一。
2.优先使用构造器依赖。
3.公共集合使用不可变快照。
4.不使用Lombok。
5.不使用字段注入。
6.不吞异常。
7.不返回null。
8.错误信息可理解但不泄露内部实现。
9.工具实现应为无状态或线程安全。
10.不得在实例字段中保存某次ToolInvocation的数据。
11.不要复制第一阶段已有的参数校验、异常处理和审计逻辑。
12.ToolArgumentValidationInterceptor负责Schema层校验，具体工具只负责无法由Schema表达的业务约束，如除数为0、时区是否合法、长度限制等。

十六、编译和启动验收
完成后执行：
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许，启动agent-bootstrap并验证：
GET /api/framework/health
GET /api/framework/tools

检查：
1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.应用无模型密钥也能启动。
4.4个工具均注册成功。
5.没有重复注册异常。
6.工具列表无重复名称。
7.agent-runtime没有新增Spring依赖。
8.没有提前实现本批外功能。
9.代码审查确认CalculatorTool不存在任意代码执行风险。
10.代码审查确认TextSearchTool不访问文件系统或Shell。

不要为了验收创建测试脚本、测试Controller或临时业务代码。

十七、最终输出
只输出：
1.新增和修改文件清单。
2.4个工具的名称、输入和行为。
3.BuiltinToolProvider及自动注册流程。
4.CalculatorTool的安全计算方案。
5.线程安全和输入限制措施。
6.新增依赖及用途；无新增依赖则明确说明。
7.编译、打包和启动验证结果。
8.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE2_BATCH2_END===