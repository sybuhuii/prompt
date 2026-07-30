你正在继续开发基于SpringAI+LangGraph4j的通用Agent框架。

已完成：
1.Maven模块化单体架构。
2.agent-core领域模型和稳定接口。
3.AgentRegistry、ToolRegistry、SupervisorRegistry及Provider自动注册。
4.ToolInvocationGateway及工具拦截器链。
5.ModelInvocationGateway及SpringAI适配。
6.完整单Agent ReAct执行引擎。
7.Supervisor中心调度及多Agent执行引擎。
8.Sample Agent、Sample Supervisor及开发验证API。

当前只执行第五阶段第1批：建立身份、用户、角色、权限和会话领域模型，完善SessionStore抽象，并提供线程安全的内存存储实现。

本批只建立身份与权限基础设施，不实现HTTP登录接口、不实现认证Filter、不保护现有Agent接口、不实现工具ACL拦截、不实现前端页面。

一、阶段背景

框架需要统一解决：

1.谁在调用Agent。
2.当前用户属于哪些角色。
3.当前用户具备哪些工具权限。
4.sessionId是否合法。
5.后续如何将身份上下文传入RunContext。
6.如何保证不同用户的会话和数据不串读。

任务要求：

1.登录成功后创建Session并返回sessionId。
2.每次Agent调用校验sessionId。
3.伪造或不存在的sessionId在入口处拒绝。
4.至少支持admin和visitor两类角色。
5.权限粒度达到工具级。
6.工具权限最终由框架统一拦截。
7.必须保留SessionStore接口，并提供内存实现。
8.身份上下文后续对LLM隐藏，但对节点和工具可见。

二、执行前检查

先完整检查全部pom.xml及现有代码：

1.UserSession。
2.SessionStore。
3.RunContext。
4.AgentErrorCode。
5.AgentFrameworkException。
6.ToolDefinition、ToolInvocation、ToolInvocationGateway。
7.AgentRegistry、ToolRegistry、SupervisorRegistry。
8.现有内存Store的代码风格。
9.现有@Configuration和@Bean装配方式。
10.现有Sample配置。
11.当前Java版本。
12.是否已经存在User、Role、Permission、Session相关模型或接口。

要求：

1.以当前真实代码和接口为准增量实现。
2.现有UserSession和SessionStore能够复用时必须复用。
3.不得创建UserSessionV2、NewSessionStore等重复抽象。
4.发现前序代码存在阻断问题时只做最小修复。
5.不得大规模重构ReAct、Supervisor、ModelGateway或ToolGateway。
6.不要修改无关代码。
7.不要实现本批范围外功能。
8.不要生成测试脚本。
9.不要编写或修改README、使用说明、升级记录、验收报告。
10.不要执行git commit或git push。

三、架构边界

继续保持：

agent-core：
-身份、用户、角色、权限、会话领域模型。
-Store和安全相关稳定接口。
-不得依赖Spring、SpringSecurity、SpringAI、LangGraph4j、Jackson或数据库框架。

agent-runtime：
-后续工具ACL和运行身份规则。
-本批原则上不实现ACL。
-不得存放用户持久化实现。

agent-application：
-后续登录和用户管理用例。
-本批原则上不新增登录服务。

agent-infrastructure：
-内存Store实现。
-密码哈希等技术适配。
-Spring Bean装配。

agent-api：
-本批不新增Controller、Filter和DTO。

agent-bootstrap：
-本批不创建登录页面。
-允许后续提供可关闭的Sample用户配置。
-本批不直接创建用户或会话。

四、身份领域模型

在agent-core现有安全或认证包中创建或复用清晰的包，例如：

com.ksyun.agent.core.security

不要因为示例名称与现有包不同而强制搬迁已有类型。

实现或完善不可变模型：

UserAccount

建议字段：

String userId
String username
String credentialHash
Set<String> roleNames
boolean enabled

要求：

1.优先使用record或等价不可变类型。
2.userId不能为空。
3.username不能为空。
4.credentialHash不能为空。
5.roleNames使用不可变Set。
6.roleNames不能为空。
7.roleNames不得包含空字符串。
8.enabled表示用户是否允许登录。
9.不得保存明文密码。
10.不得包含HttpServletRequest、SpringSecurity Authentication或JWT类型。
11.不得包含SessionStore等服务对象。
12.不得包含用户运行中的消息或Agent State。
13.用户名标准化规则必须明确；建议trim后保持原大小写或统一小写，但整个系统只能采用一种规则。
14.用户名是否区分大小写必须由统一规范决定，不能由各Store自行决定。

五、角色领域模型

实现或复用：

RoleDefinition

建议字段：

String roleName
String description
Set<String> permissionCodes

要求：

1.roleName不能为空。
2.角色名称采用稳定格式，例如：
ADMIN
VISITOR
3.permissionCodes使用不可变Set。
4.permissionCodes不得包含空字符串。
5.角色不得直接保存UserAccount列表。
6.角色不得依赖ToolRegistry。
7.角色不得保存AgentTool实例。
8.角色定义只表达权限，不执行权限判断。
9.不得硬编码只有admin和visitor，框架应允许未来增加角色。
10.本阶段至少能表达admin和visitor的工具权限差异。

六、权限编码模型

实现或复用统一权限值对象：

PermissionCode

建议：

public record PermissionCode(String value)

要求：

1.value不能为空。
2.统一trim。
3.禁止空格和不可见字符。
4.权限编码格式必须集中定义。
5.业务代码不得散落字符串拼接。

增加工具权限编码工具类或值对象：

ToolPermissionCodes

至少支持：

tool:{toolName}:invoke

以及管理员通配权限：

tool:*:invoke

示例：

tool:calculator:invoke
tool:current_time:invoke
tool:echo:invoke
tool:text_search:invoke
tool:*:invoke

要求：

1.提供创建具体工具调用权限的方法，例如：
ToolPermissionCodes.invoke(String toolName)

2.提供全工具调用权限常量或方法。
3.工具名称不能为空。
4.工具名称使用ToolDefinition现有命名规范。
5.不得在各工具实现内部拼接权限字符串。
6.不得把角色名称直接当权限。
7.不得在PermissionCode中依赖ToolRegistry。
8.不得使用“admin自动绕过全部安全校验”的隐藏逻辑。
9.ADMIN是否可调用全部工具，应通过tool:*:invoke权限表达。
10.后续ACL判断同时支持精确权限和通配权限。

如现有项目已经使用Set<String>表示RunContext.permissions：
-可继续保留Set<String>。
-PermissionCode作为构建和校验边界。
-不要为了形式统一大规模修改RunContext。
-必须保证权限编码只在一个位置生成。

七、会话领域模型

检查现有UserSession。

UserSession至少应表达：

String sessionId
String userId
String username
Set<String> roles
Set<String> permissions
Instant createdAt
Instant expiresAt或等价可选过期时间
boolean active或等价状态

如果现有模型字段略有不同，以实际模型为准。

要求：

1.不得创建第二套Session模型。
2.sessionId不能为空。
3.userId不能为空。
4.roles和permissions使用不可变Set。
5.Session创建时保存角色和权限快照。
6.不得保存明文密码或credentialHash。
7.不得保存HttpSession、ServletRequest或Spring Security对象。
8.不得保存ModelClient、AgentState或消息历史。
9.不得允许调用方修改已有Session。
10.若现有UserSession没有过期字段：
-本批不强制加入复杂TTL机制。
-但应提供active或可验证状态。
11.若已有expiresAt：
-提供isExpired(Instant now)或等价判断。
-不得直接在模型内部调用Instant.now()形成不可测试的隐式时间。
12.会话是否合法最终由SessionStore中的真实记录决定，不能仅根据sessionId格式判断。

八、Store接口

在agent-core定义或复用：

UserStore
RoleStore
SessionStore

UserStore建议方法：

void save(UserAccount user);
Optional<UserAccount> findById(String userId);
Optional<UserAccount> findByUsername(String username);
Collection<UserAccount> list();
boolean existsByUsername(String username);

RoleStore建议方法：

void save(RoleDefinition role);
Optional<RoleDefinition> find(String roleName);
RoleDefinition getRequired(String roleName);
Collection<RoleDefinition> list();

SessionStore必须保留现有接口，不得重新命名。

SessionStore至少支持当前阶段所需能力：

void save(UserSession session);
Optional<UserSession> find(String sessionId);
void delete(String sessionId);
Collection<UserSession> findByUserId(String userId);

若现有接口已经使用其他等价方法名，优先保持兼容，不要无意义重命名。

要求：

1.所有Store接口位于agent-core。
2.不得依赖ConcurrentHashMap。
3.不得依赖Spring。
4.不得依赖数据库或Redis。
5.查询不存在返回Optional.empty。
6.写入null明确拒绝。
7.list和查询集合语义明确。
8.不使用null表示未找到。
9.SessionStore是本模块必交付抽象。
10.本批不设计分页。
11.本批不设计事务。
12.本批不实现数据库持久化。
13.本批不在Store中完成登录。
14.本批不在Store中计算角色权限。
15.不得在Store中生成sessionId。

九、角色权限解析服务

在agent-core定义纯Java接口：

RolePermissionResolver

建议方法：

Set<String> resolvePermissions(Set<String> roleNames);

在agent-runtime或agent-application实现纯Java：

DefaultRolePermissionResolver

依赖：

RoleStore

职责：

1.根据角色名称查找RoleDefinition。
2.合并所有角色permissionCodes。
3.去重。
4.返回不可变Set。
5.未知角色明确抛出AgentFrameworkException。
6.不得静默忽略未知角色。
7.不得访问ToolRegistry。
8.不得判断某个具体工具是否可调用。
9.不得读取当前线程或Spring Security上下文。
10.不得保存请求级状态。

如项目现有模块依赖方向不允许agent-runtime依赖RoleStore的实现：
-只依赖agent-core中的RoleStore接口。
-不要反转现有模块依赖。
-不要将DefaultRolePermissionResolver放入agent-infrastructure，除非现有架构明确要求。

十、身份错误码

检查AgentErrorCode。

只在缺失时最小增加：

AUTHENTICATION_FAILED
INVALID_CREDENTIALS
SESSION_NOT_FOUND
SESSION_INVALID
SESSION_EXPIRED
USER_NOT_FOUND
USER_DISABLED
ROLE_NOT_FOUND
PERMISSION_DENIED

要求：

1.不要创建与现有错误码语义重复的新值。
2.本批不实现HTTP状态码映射。
3.后续：
-无效凭证对应401。
-无效或伪造Session对应401。
-已认证但无权限对应403。
4.不要把用户名是否存在等敏感细节泄漏给登录调用方。
5.Store的普通未找到优先使用Optional；只有getRequired或业务校验时抛异常。

十一、内存UserStore

在agent-infrastructure实现：

InMemoryUserStore implements UserStore

要求：

1.使用ConcurrentHashMap。
2.同时支持按userId和username查询。
3.两个索引必须保持一致。
4.用户名唯一。
5.userId唯一。
6.重复保存同一userId但用户名不同，不得造成脏索引。
7.重复用户名绑定不同userId必须明确拒绝。
8.不得静默覆盖其他用户。
9.save前完成参数校验。
10.list返回不可变快照。
11.返回UserAccount不可变对象。
12.线程安全。
13.不得使用synchronized锁住全部读操作。
14.确需保证双索引原子性时，可使用局部锁或ConcurrentHashMap原子操作，并说明策略。
15.不得添加@Component。
16.不得写入文件。
17.不得内置admin或visitor用户。
18.不得存储明文密码。

十二、内存RoleStore

实现：

InMemoryRoleStore implements RoleStore

要求：

1.使用ConcurrentHashMap。
2.roleName唯一。
3.重复角色注册默认明确失败，不静默覆盖。
4.如需要更新角色，后续通过单独管理用例实现，本批不将save同时设计为隐式覆盖。
5.list返回不可变快照。
6.getRequired不存在时抛ROLE_NOT_FOUND。
7.线程安全。
8.不得添加@Component。
9.不得内置具体角色。
10.不得访问UserStore。

十三、内存SessionStore

实现或完善：

InMemorySessionStore implements SessionStore

要求：

1.使用ConcurrentHashMap，以sessionId为主键。
2.线程安全。
3.save拒绝null。
4.sessionId重复时：
-同一不可变Session重复保存可幂等。
-不同Session占用相同sessionId必须拒绝。
5.find返回Optional。
6.delete不存在时允许幂等。
7.findByUserId只返回该用户的会话。
8.findByUserId返回不可变快照。
9.不同userId之间不得串读。
10.不得通过sessionId前缀推断userId。
11.不得使用static全局Map。
12.不得添加@Component。
13.不得在Store中生成sessionId。
14.不得在Store中自动续期。
15.若UserSession有expiresAt：
-find仍返回原始记录。
-过期判断由后续Session验证服务执行。
-不得在普通find中悄悄删除会话。
16.不得保存RunContext、AgentState或消息。
17.不得将Session写入日志。
18.无Session时应用仍可正常启动。

十四、凭证哈希抽象

在agent-core定义：

CredentialHasher

建议方法：

String hash(CharSequence rawCredential);
boolean matches(CharSequence rawCredential, String encodedCredential);

要求：

1.不得在agent-core依赖Spring Security。
2.不得提供明文比较实现。
3.不得记录rawCredential。
4.参数为空明确拒绝。
5.接口不限定具体哈希算法。

在agent-infrastructure提供安全实现。

优先方案：

1.若项目已有安全密码哈希依赖和实现，直接复用。
2.若没有，可仅引入spring-security-crypto模块，不引入整套Spring Security Web。
3.使用BCryptPasswordEncoder或等价成熟算法。
4.不得自研SHA-256(password)等不带安全工作因子的简单方案。
5.不得使用MD5。
6.不得使用可逆加密保存密码。
7.不得在日志中记录密码或哈希。
8.实现保持无状态和线程安全。
9.不得添加@Component，通过@Bean装配。

注意：
本批只提供哈希能力，不实现登录流程。

十五、Session ID生成器

在agent-core定义或复用：

SessionIdGenerator

方法：

String generate();

在agent-infrastructure实现：

SecureRandomSessionIdGenerator

要求：

1.使用SecureRandom。
2.生成至少128 bit不可预测随机值。
3.使用URL安全编码。
4.不得使用用户名、时间戳、递增ID或UUID版本1生成可推断sessionId。
5.不得在sessionId中编码userId。
6.不得返回空字符串。
7.保持线程安全。
8.不得记录生成的sessionId。
9.不得添加@Component。
10.不得在SessionStore中生成ID。

如果项目已有足够安全的通用随机ID生成器：
-可复用。
-必须确认其不可预测性。
-普通RunIdGenerator若仅使用递增值或普通UUID语义不明确，不得直接作为认证Session ID。

十六、Sample角色定义

本批可以在agent-bootstrap中提供受开关控制的Sample角色定义，但不得创建Sample用户或Session。

优先创建：

SampleSecurityConfiguration

受配置控制：

agent.sample.enabled=true

提供两个RoleDefinition Bean或RoleProvider机制：

ADMIN：
permissionCodes：
tool:*:invoke

VISITOR：
至少允许：
tool:calculator:invoke
tool:current_time:invoke
tool:echo:invoke

默认不允许：
tool:text_search:invoke

具体visitor允许范围应清晰、可演示。

要求：

1.如果项目已有RoleProvider/Registrar模式，使用Provider自动注册。
2.如果本批新增RoleProvider和RoleProviderRegistrar，应遵循AgentProvider、ToolProvider和SupervisorProvider相同模式。
3.不得在bootstrap启动类手工调用RoleStore.save。
4.不得创建明文Sample密码。
5.不得创建Sample Session。
6.无模型配置时Sample角色仍可注册。
7.agent.sample.enabled=false时不注册Sample角色。
8.不得把ADMIN角色写成代码中的特殊绕过条件。
9.ADMIN的全权限通过tool:*:invoke表达。
10.VISITOR权限通过具体permissionCode表达。

如增加RoleProvider：

放在agent-core：
RoleProvider

方法：
Collection<RoleDefinition> provideRoles();

放在agent-runtime或适合的注册层：
RoleProviderRegistrar

要求：
-Provider不直接操作Spring容器。
-Registrar不吞重复角色异常。
-没有Provider时应用正常启动。
-不得创建第二套RoleRegistry；RoleStore就是角色定义的存取边界。

十七、Spring Bean装配

在agent-infrastructure新增或完善配置类，装配：

UserStore → InMemoryUserStore
RoleStore → InMemoryRoleStore
SessionStore → InMemorySessionStore
RolePermissionResolver → DefaultRolePermissionResolver
CredentialHasher → 安全哈希实现
SessionIdGenerator → SecureRandomSessionIdGenerator
RoleProviderRegistrar（若新增）

要求：

1.使用@Configuration和@Bean。
2.runtime和core实现不添加Spring注解。
3.不得在bootstrap启动类中手工new。
4.不得产生Bean循环依赖。
5.不得创建重复Store Bean。
6.允许用户未来通过自定义Bean替换内存实现。
7.优先使用@ConditionalOnMissingBean或项目现有扩展方式。
8.没有Sample RoleProvider时应用仍能启动。
9.无模型配置时身份Store仍能装配。
10.不要求数据库。
11.不要求Redis。
12.不引入Spring Security Web过滤链。
13.本批不保护Controller。

十八、多用户隔离基础规则

本批只建立数据结构和Store边界，暂不接入Agent API。

必须明确保证：

1.UserSession明确关联唯一userId。
2.SessionStore按sessionId查询真实Session。
3.SessionStore按userId查询时严格过滤。
4.不能通过客户端传入的userId代替Session中的userId。
5.后续RunContext.userId必须来自已校验Session。
6.客户端未来只提交sessionId，不直接决定roles和permissions。
7.roles和permissions从Session快照或服务器端用户角色解析获得。
8.身份信息不得发送给LLM。
9.工具和节点可通过RunContext读取身份。
10.本批不修改ReactDevApplicationService和SupervisorDevApplicationService的固定dev-user逻辑。
11.正式认证入口将在下一批新增，开发API是否继续保留由配置控制。

十九、本批禁止实现

禁止：

1.登录Controller。
2.用户管理Controller。
3.角色管理Controller。
4.Session认证Filter或Interceptor。
5.Authorization Header解析。
6.Cookie认证。
7.JWT。
8.Agent接口身份保护。
9.Supervisor接口身份保护。
10.Tool ACL拦截器。
11.权限拒绝回灌LLM。
12.用户记忆隔离实现。
13.HITL审批人权限。
14.CheckpointStore。
15.上下文裁剪。
16.长期记忆。
17.前端页面。
18.管理员创建用户页面。
19.数据库、Redis或文件持久化。
20.会话自动续期。
21.刷新Token。
22.验证码。
23.第三方OAuth。
24.Spring Security Web过滤链。
25.测试脚本。
26.README、使用说明、升级记录或验收报告。
27.git commit和git push。

二十、代码质量

1.不使用Lombok。
2.优先record、enum和final类。
3.使用构造器注入。
4.集合返回不可变快照。
5.不返回null表示未找到。
6.不使用字段注入。
7.不通过ApplicationContext主动查Bean。
8.不使用static可变Map。
9.不把身份状态放入ThreadLocal。
10.不吞异常。
11.不记录密码、密码哈希或sessionId。
12.不创建无意义Facade、Delegate或Manager。
13.不将权限判断分散到具体工具。
14.不得将ADMIN角色写成特殊if绕过安全机制。
15.所有实现以实际编译结果为准。

二十一、编译验收

完成后执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许启动应用，确认已有接口不受影响：

GET /api/framework/health
GET /api/framework/tools
GET /api/framework/agents
GET /api/framework/supervisors

检查：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.agent-core无Spring、SpringAI、LangGraph4j和数据库依赖。
4.agent-runtime无SpringWeb和agent-infrastructure依赖。
5.UserAccount不保存明文密码。
6.RoleDefinition使用权限集合。
7.ADMIN全工具权限通过tool:*:invoke表示。
8.VISITOR只具备明确的部分工具权限。
9.SessionStore接口位于agent-core。
10.InMemorySessionStore线程安全。
11.Session按userId查询不会串读。
12.InMemoryUserStore双索引保持一致。
13.用户名和userId重复不会静默覆盖。
14.密码哈希使用成熟安全算法。
15.Session ID具有不可预测性。
16.没有登录API。
17.没有认证Filter。
18.没有工具ACL拦截。
19.没有前端页面。
20.现有ReAct和Supervisor接口未被破坏。
21.git diff没有无关修改。

二十二、最终输出

只输出：

1.新增和修改文件清单。
2.UserAccount字段和不变式。
3.RoleDefinition及权限编码规则。
4.ADMIN与VISITOR权限差异。
5.UserSession最终字段及会话语义。
6.UserStore、RoleStore、SessionStore接口。
7.三个内存Store的线程安全策略。
8.RolePermissionResolver合并规则。
9.CredentialHasher实现及依赖。
10.SessionIdGenerator安全策略。
11.Sample角色自动注册方式。
12.Spring Bean装配和可替换方式。
13.多用户隔离基础边界。
14.编译、打包和启动验证结果。
15.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE5_BATCH1_END===