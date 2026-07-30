你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

已完成：
1.用户、角色、权限和Session领域模型。
2.UserStore、RoleStore、SessionStore及内存实现。
3.用户名密码登录和X-Session-Id认证。
4.正式Agent、Supervisor认证入口。
5.RunContext身份透传。
6.工具级ACL统一拦截。
7.ADMIN与VISITOR工具权限差异。

当前只执行第五阶段第4批：

1.实现用户管理应用服务和管理API。
2.实现角色管理应用服务和管理API。
3.管理员可创建用户、修改角色、启用或禁用用户、重置密码。
4.管理员可查询、创建角色及修改角色权限。
5.权限、角色、密码或用户状态变化后，撤销受影响用户的旧Session。
6.使用权限编码保护管理接口，禁止根据ADMIN角色名称硬编码放行。
7.完成Postman验证。

本批不实现前端页面，不实现删除用户和删除角色，不实现数据库持久化。

一、执行前检查

先检查：

1.UserAccount、RoleDefinition、UserSession。
2.UserStore、RoleStore、SessionStore及内存实现。
3.PermissionCode、ToolPermissionCodes。
4.RolePermissionResolver。
5.CredentialHasher。
6.SessionValidationService。
7.AuthenticationApplicationService。
8.SessionAuthenticationInterceptor。
9.统一异常和HTTP状态映射。
10.SampleSecurityConfiguration。
11.ADMIN和VISITOR当前权限。
12.agent-api现有DTO、Controller代码风格。

要求：

1.以当前真实接口和包结构增量实现。
2.不得创建第二套User、Role、Session或Store。
3.发现前序阻断问题时只做最小修复。
4.不得重构ReAct、Supervisor或Tool ACL。
5.不生成测试脚本。
6.不修改README或其他说明文档。
7.不执行git commit或git push。

二、管理权限编码

在agent-core现有security包中增加或复用：

SecurityPermissionCodes

至少定义：

security:user:read
security:user:write
security:role:read
security:role:write
security:session:revoke

要求：

1.使用PermissionCode或现有统一编码规范。
2.不得在Controller和ApplicationService中散落字符串。
3.不得把ADMIN角色名称直接视为授权依据。
4.不得使用username=admin作为放行条件。
5.ADMIN示例角色应拥有上述全部权限。
6.VISITOR默认不拥有任何管理权限。
7.ADMIN原有tool:*:invoke权限保持不变。
8.权限匹配使用完全匹配。
9.本批不增加复杂权限继承和通配规则。

三、通用管理权限校验

在agent-application实现纯Java：

PermissionAuthorizationService

建议方法：

void requirePermission(
UserSession session,
String permissionCode
);

boolean hasPermission(
UserSession session,
String permissionCode
);

要求：

1.只读取已验证UserSession.permissions。
2.session为空时拒绝。
3.permissionCode为空时拒绝。
4.权限缺失抛PERMISSION_DENIED。
5.不得检查角色名称。
6.不得访问HttpServletRequest。
7.不得使用ThreadLocal。
8.不得访问SessionStore。
9.不得记录完整permissions。
10.保持无状态和线程安全。

该服务用于管理API授权。

工具ACL继续使用ToolPermissionEvaluator，不得将两者混为一套工具执行逻辑。

四、扩展UserStore

检查现有UserStore。

需要支持明确更新用户。

优先增加：

void update(UserAccount user);

或项目风格一致的等价方法。

要求：

1.save继续表示创建新用户。
2.update只更新已存在用户。
3.不存在时抛USER_NOT_FOUND。
4.不得使用save静默覆盖已有用户。
5.userId创建后不可修改。
6.username创建后本批不允许修改。
7.update主要允许修改：
-roleNames
-enabled
-credentialHash
8.InMemoryUserStore更新双索引时保持一致。
9.不得产生旧username脏索引。
10.并发更新必须线程安全。
11.不得返回可变UserAccount。
12.不得暴露credentialHash到API。

五、扩展RoleStore

检查现有RoleStore。

增加明确更新方法：

void update(RoleDefinition role);

要求：

1.save用于创建新角色。
2.update只更新已有角色。
3.不存在时抛ROLE_NOT_FOUND。
4.roleName创建后不可修改。
5.update只替换description和permissionCodes。
6.不得静默创建不存在角色。
7.InMemoryRoleStore更新必须线程安全。
8.返回值和集合保持不可变。
9.不得把角色更新与用户更新混在Store中。
10.不得访问SessionStore。

六、Session批量撤销服务

在agent-application实现：

UserSessionRevocationService

依赖：

SessionStore

建议方法：

int revokeAllForUser(String userId);

int revokeAllForUsers(Collection<String> userIds);

职责：

1.使用SessionStore.findByUserId查询。
2.逐个删除对应sessionId。
3.仅删除目标用户Session。
4.不得删除其他用户Session。
5.返回实际撤销数量。
6.不存在Session时返回0。
7.操作幂等。
8.不得记录sessionId。
9.不得使用Session ID前缀推断用户。
10.保持无状态。
11.不得在此服务中修改用户或角色。

七、用户管理应用服务

在agent-application实现：

UserManagementApplicationService

依赖：

UserStore
RoleStore
RolePermissionResolver
CredentialHasher
PermissionAuthorizationService
UserSessionRevocationService
现有UserId生成器或安全ID生成方式

如项目没有UserIdGenerator，可新增稳定接口和安全实现，但不得使用数据库自增假设。

至少提供：

Collection<UserSummary> listUsers(UserSession operator);

UserSummary createUser(
UserSession operator,
String username,
CharSequence password,
Set<String> roleNames
);

UserSummary updateUser(
UserSession operator,
String userId,
Set<String> roleNames,
boolean enabled
);

void resetPassword(
UserSession operator,
String userId,
CharSequence newPassword
);

要求：

1.listUsers需要security:user:read。
2.createUser、updateUser、resetPassword需要security:user:write。
3.所有operator必须来自已认证Session。
4.Controller不得自行判断权限。
5.创建用户时：
-校验username和password。
-用户名使用现有统一规范化规则。
-用户名不可重复。
-校验全部角色存在。
-使用CredentialHasher.hash。
-不得保存明文密码。
-默认enabled=true。
6.更新用户时：
-只修改roleNames和enabled。
-校验全部角色存在。
-不得修改userId和username。
7.密码重置时：
-使用CredentialHasher.hash。
-不得返回新密码或hash。
8.用户角色、状态或密码发生变化后：
-撤销该用户全部旧Session。
9.创建用户不创建Session。
10.创建用户不自动登录。
11.列表和响应不得包含credentialHash。
12.UserSummary至少包含：
-userId
-username
-roleNames
-enabled
13.UserSummary使用不可变类型。
14.不得把permissions直接存进UserAccount。
15.用户有效权限仍在登录时通过RolePermissionResolver计算。
16.不得接受客户端直接提交permissionCodes。

八、自我锁定保护

为了防止当前管理者误操作，本批采用简单且确定的安全规则：

1.管理员不得通过用户管理API禁用自己的账户。
2.管理员不得通过用户管理API修改自己的角色。
3.管理员不得通过管理员重置接口重置自己的密码。
4.自助修改密码不在本批实现。
5.识别当前用户必须比较operator.userId和目标userId。
6.不得根据username判断是否为本人。
7.违反时返回INVALID_ARGUMENT或明确安全错误。
8.不得硬编码admin用户ID。

本批不实现“最后一个管理员”复杂检测。

九、角色管理应用服务

在agent-application实现：

RoleManagementApplicationService

依赖：

RoleStore
UserStore
PermissionAuthorizationService
UserSessionRevocationService

至少提供：

Collection<RoleSummary> listRoles(UserSession operator);

RoleSummary createRole(
UserSession operator,
String roleName,
String description,
Set<String> permissionCodes
);

RoleSummary updateRolePermissions(
UserSession operator,
String roleName,
String description,
Set<String> permissionCodes
);

要求：

1.listRoles需要security:role:read。
2.createRole和updateRolePermissions需要security:role:write。
3.roleName统一trim和规范化。
4.角色名称不能为空。
5.创建时角色不得已存在。
6.更新时角色必须存在。
7.permissionCodes逐项通过PermissionCode校验。
8.权限集合使用不可变Set。
9.不得根据ToolRegistry判断权限是否存在。
10.允许预先配置未来扩展权限。
11.不得把角色名称当权限。
12.不得修改UserAccount列表。
13.修改角色权限后：
-查找所有包含该角色的用户。
-撤销这些用户全部旧Session。
14.这样用户必须重新登录，获得新的权限快照。
15.角色描述可更新。
16.本批不支持修改roleName。
17.本批不删除角色。
18.RoleSummary至少包含：
-roleName
-description
-permissionCodes
19.不得返回内部Store实现信息。

十、Sample角色权限更新

更新Sample ADMIN角色：

至少包含：

tool:*:invoke
security:user:read
security:user:write
security:role:read
security:role:write
security:session:revoke

VISITOR保持现有有限工具权限，不增加管理权限。

要求：

1.ADMIN权限仍通过权限编码表达。
2.不得在PermissionAuthorizationService中特判ADMIN。
3.不得在Controller中特判ADMIN。
4.agent.sample.enabled=false时不注册Sample角色。
5.不得静默覆盖用户自定义同名角色。
6.如果Sample角色注册发生冲突，沿用现有安全处理。

十一、用户管理Controller

在agent-api实现：

UserManagementController

路径统一：

/api/admin/users

接口建议：

GET /api/admin/users

POST /api/admin/users

PUT /api/admin/users/{userId}

POST /api/admin/users/{userId}/reset-password

要求：

1.全部由SessionAuthenticationInterceptor保护。
2.从@RequestAttribute读取已验证UserSession。
3.Controller只依赖UserManagementApplicationService。
4.不得注入UserStore、RoleStore或SessionStore。
5.不得在Controller中判断角色或权限。
6.不得允许请求提交：
-userId用于创建
-credentialHash
-permissions
-sessionId
7.创建请求只包含：
-username
-password
-roleNames
8.更新请求只包含：
-roleNames
-enabled
9.重置密码请求只包含：
-newPassword
10.响应不得包含密码或credentialHash。
11.列表返回专用DTO。
12.创建成功建议HTTP201。
13.更新成功HTTP200。
14.重复用户名返回HTTP409。
15.用户不存在返回HTTP404。
16.权限不足返回HTTP403。
17.参数错误返回HTTP400。
18.不得返回Java异常或堆栈。

十二、角色管理Controller

在agent-api实现：

RoleManagementController

统一路径：

/api/admin/roles

接口建议：

GET /api/admin/roles

POST /api/admin/roles

PUT /api/admin/roles/{roleName}

要求：

1.全部由SessionAuthenticationInterceptor保护。
2.从@RequestAttribute读取UserSession。
3.Controller只依赖RoleManagementApplicationService。
4.不得注入RoleStore。
5.不得在Controller中硬编码ADMIN。
6.创建请求包含：
-roleName
-description
-permissionCodes
7.更新请求包含：
-description
-permissionCodes
8.路径roleName与Body不得重复表达不同名称。
9.不得允许修改roleName。
10.返回专用Role DTO。
11.重复角色返回HTTP409。
12.角色不存在返回HTTP404。
13.权限不足返回HTTP403。
14.参数错误返回HTTP400。
15.不得暴露内部Store类型。

十三、认证拦截路径

调整SessionAuthenticationInterceptor的MVC注册：

增加保护：

/api/admin/**

继续保护：

/api/agent/**
/api/supervisor/**
/api/auth/me
/api/auth/logout

继续排除：

/api/auth/login
/api/framework/**
/api/dev/**
静态资源
/error
OPTIONS

要求：

1./api/admin/**缺少Session时返回401。
2.Session合法但缺少管理权限时返回403。
3.认证和授权必须区分。
4.不得把管理权限判断放进认证拦截器。
5.认证拦截器只负责Session真实性。

十四、Session权限快照语义

明确采用：

登录时生成权限快照。

规则：

1.UserSession.permissions在登录时由角色解析。
2.旧Session不会实时读取RoleStore。
3.用户角色变化后撤销该用户全部Session。
4.用户密码变化后撤销该用户全部Session。
5.用户被禁用后撤销其Session。
6.角色权限变化后撤销所有受影响用户Session。
7.用户必须重新登录获得新权限。
8.不得直接修改已有UserSession.permissions。
9.不得尝试在每次请求时重新解析角色。
10.SessionValidationService仍检查用户enabled状态。

十五、错误码和HTTP映射

检查现有AgentErrorCode，缺失时最小增加或复用：

USER_ALREADY_EXISTS
ROLE_ALREADY_EXISTS

优先复用已有：

USER_NOT_FOUND
ROLE_NOT_FOUND
PERMISSION_DENIED
INVALID_ARGUMENT

HTTP映射：

400：
INVALID_ARGUMENT

401：
SESSION_INVALID
SESSION_EXPIRED

403：
PERMISSION_DENIED

404：
USER_NOT_FOUND
ROLE_NOT_FOUND

409：
USER_ALREADY_EXISTS
ROLE_ALREADY_EXISTS

500：
INTERNAL_ERROR

要求：

1.不得增加多个重复错误码。
2.不得把重复用户返回500。
3.不得泄漏密码、hash或sessionId。
4.不得返回内部类名。
5.继续使用现有统一JSON错误响应。

十六、日志要求

允许记录：

1.operatorUserId。
2.目标userId。
3.roleName。
4.操作类型。
5.撤销Session数量。
6.操作成功或失败。

禁止记录：

1.密码。
2.credentialHash。
3.sessionId。
4.完整permissions集合。
5.完整UserSession。
6.API Key。
7.完整请求Body。

要求：

1.密码重置日志不得包含密码。
2.Session撤销日志只记录数量。
3.不得建立数据库审计。
4.不得引入新的监控平台。

十七、Postman验收

前置：

1.agent.sample.enabled=true。
2.ADMIN、VISITOR用户可登录。
3.使用admin登录得到adminSession。
4.使用visitor登录得到visitorSession。

场景1：admin查询用户

GET /api/admin/users
X-Session-Id: {{adminSession}}

预期：

1.HTTP200。
2.包含admin和visitor。
3.不包含credentialHash。

场景2：visitor查询用户

GET /api/admin/users
X-Session-Id: {{visitorSession}}

预期：

1.HTTP403。
2.PERMISSION_DENIED。
3.不得返回用户数据。

场景3：admin创建用户

POST /api/admin/users
X-Session-Id: {{adminSession}}

{
"username":"operator",
"password":"Operator-Test-Password",
"roleNames":["VISITOR"]
}

预期：

1.HTTP201。
2.返回operator用户。
3.不返回密码或hash。
4.operator可以登录。
5.operator初始工具权限与VISITOR一致。

场景4：重复用户名

再次创建operator。

预期：

1.HTTP409。
2.USER_ALREADY_EXISTS。
3.不得覆盖原用户。

场景5：修改用户角色

先使用operator登录得到operatorSession。

PUT /api/admin/users/{operatorUserId}
X-Session-Id: {{adminSession}}

{
"roleNames":["ADMIN"],
"enabled":true
}

预期：

1.更新成功。
2.operator旧Session被撤销。
3.旧operatorSession访问/api/auth/me返回401。
4.operator重新登录后拥有ADMIN管理权限。

场景6：禁用用户

PUT /api/admin/users/{operatorUserId}

{
"roleNames":["VISITOR"],
"enabled":false
}

预期：

1.全部operator Session撤销。
2.operator不能继续登录。
3.现有Session返回401。

场景7：重置密码

重新启用operator后调用：

POST /api/admin/users/{operatorUserId}/reset-password

{
"newPassword":"New-Operator-Password"
}

预期：

1.旧Session全部撤销。
2.旧密码登录失败。
3.新密码登录成功。
4.响应和日志不包含密码。

场景8：admin查询角色

GET /api/admin/roles
X-Session-Id: {{adminSession}}

预期：

1.包含ADMIN和VISITOR。
2.ADMIN包含管理权限编码。
3.VISITOR不包含管理权限。

场景9：visitor查询角色

使用visitorSession调用。

预期HTTP403。

场景10：创建角色

POST /api/admin/roles

{
"roleName":"OPERATOR",
"description":"普通操作人员",
"permissionCodes":[
"tool:calculator:invoke",
"tool:current_time:invoke"
]
}

预期：

1.HTTP201。
2.角色保存成功。
3.不自动创建用户。

场景11：修改VISITOR权限

先使用visitor登录得到旧visitorSession。

PUT /api/admin/roles/VISITOR

{
"description":"访客用户",
"permissionCodes":[
"tool:calculator:invoke",
"tool:current_time:invoke",
"tool:echo:invoke",
"tool:text_search:invoke"
]
}

预期：

1.角色更新成功。
2.所有VISITOR用户旧Session被撤销。
3.旧visitorSession返回401。
4.visitor重新登录后获得text_search权限。
5.重新调用text_search可通过ACL。

场景12：禁止修改自己

admin尝试修改自己的角色或enabled状态。

预期：

1.HTTP400或明确安全错误。
2.admin当前Session仍有效。
3.不得锁死管理入口。

场景13：无Session访问

GET /api/admin/users

预期HTTP401。

场景14：伪造管理员字段

visitor调用管理接口并在Body中加入：

{
"roles":["ADMIN"],
"permissions":["security:user:write"]
}

预期：

1.仍然HTTP403。
2.权限只来自visitor Session。
3.不得提升权限。

十八、线程安全与一致性

1.InMemoryUserStore更新保持双索引一致。
2.InMemoryRoleStore更新为原子替换。
3.不同用户管理请求不能串改。
4.不得使用static当前用户。
5.不得使用ThreadLocal。
6.应用服务无请求级字段。
7.Session撤销只作用于目标用户。
8.角色更新撤销所有受影响用户Session。
9.不得缓存管理授权结果。
10.不得在Controller中保存UserSession。

十九、本批禁止实现

禁止：

1.删除用户。
2.删除角色。
3.修改username。
4.修改roleName。
5.客户端直接给用户设置permissions。
6.前端页面。
7.数据库、Redis或文件持久化。
8.Spring Security Web。
9.JWT和OAuth。
10.用户自助注册。
11.用户自助修改密码。
12.忘记密码和验证码。
13.刷新Session权限而不重新登录。
14.会话自动续期。
15.HITL。
16.Checkpoint和记忆。
17.RAG和MCP。
18.测试脚本。
19.README、说明和验收报告。
20.git commit和git push。

二十、编译验收

执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

如环境允许启动验证：

POST /api/auth/login
GET /api/auth/me
GET /api/admin/users
POST /api/admin/users
PUT /api/admin/users/{userId}
POST /api/admin/users/{userId}/reset-password
GET /api/admin/roles
POST /api/admin/roles
PUT /api/admin/roles/{roleName}

检查：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.Controller不直接访问Store。
4.ApplicationService统一执行管理授权。
5.不存在ADMIN角色硬编码放行。
6.VISITOR访问管理API返回403。
7.管理API无Session返回401。
8.用户角色变化后Session被撤销。
9.用户密码变化后Session被撤销。
10.用户禁用后Session被撤销。
11.角色权限变化后受影响用户Session被撤销。
12.响应不包含credentialHash。
13.日志不包含密码和sessionId。
14.现有认证、Tool ACL、ReAct和Supervisor功能未被破坏。
15.没有实现本批范围外功能。
16.git diff没有无关修改。

二十一、最终输出

只输出：

1.新增和修改文件清单。
2.SecurityPermissionCodes定义。
3.PermissionAuthorizationService授权规则。
4.UserStore和RoleStore更新语义。
5.UserManagementApplicationService功能。
6.RoleManagementApplicationService功能。
7.用户和角色管理API。
8.自我锁定保护规则。
9.Session权限快照及撤销策略。
10.ADMIN和VISITOR最终管理权限差异。
11.错误码与HTTP映射。
12.编译、打包和启动结果。
13.实际完成的Postman验证。
14.未完成验证及准确原因。
15.发现但未处理的非本批问题。

编译或打包失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE5_BATCH4_END===