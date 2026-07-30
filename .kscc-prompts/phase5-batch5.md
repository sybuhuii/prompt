你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

已完成：

1.用户、角色、权限和Session领域模型。
2.UserStore、RoleStore、SessionStore及内存实现。
3.用户名密码登录与X-Session-Id认证。
4.正式Agent和Supervisor认证入口。
5.工具级ACL统一拦截及权限失败回灌LLM。
6.ADMIN与VISITOR权限差异。
7.用户和角色管理后端接口。
8.角色、用户状态或密码变化后的Session撤销。

当前只执行第五阶段第5批：

1.实现前后端分离的Web动态页面。
2.实现登录、登出和当前身份展示。
3.实现用户管理页面。
4.实现角色与权限管理页面。
5.实现单Agent和Supervisor调用页面。
6.实现ADMIN与VISITOR工具权限差异演示。
7.完成前后端整体联调。

本批完成后，第五阶段应形成可现场演示的完整身份与权限子系统。

本批不实现HITL、Checkpoint、上下文裁剪、长期记忆、RAG、MCP或真实金山云业务。

一、执行前检查

先检查：

1.仓库是否已有前端项目。
2.已有前端框架、目录、依赖和页面风格。
3.POST /api/auth/login。
4.POST /api/auth/logout。
5.GET /api/auth/me。
6.POST /api/agent/invoke。
7.POST /api/supervisor/invoke。
8.GET、POST、PUT /api/admin/users相关接口。
9.GET、POST、PUT /api/admin/roles相关接口。
10.GET /api/framework/tools。
11.GET /api/framework/agents。
12.GET /api/framework/supervisors。
13.统一成功和异常响应结构。
14.X-Session-Id请求头规范。
15.后端CORS配置。
16.当前Node和npm版本要求。
17.现有application.yml和启动端口。

要求：

1.以当前真实接口和返回字段为准实现。
2.不得假设提示词示例字段与实际DTO完全一致。
3.已有前端时增量开发，不创建重复前端工程。
4.没有前端时，在仓库根目录创建独立前端目录：
agent-web
5.前端使用Vue 3+Vite。
6.只使用JavaScript，不使用TypeScript。
7.不得修改已有后端接口以迁就错误的前端假设。
8.发现后端阻断问题时只做最小修复并说明。
9.不得重构ReAct、Supervisor、认证和ACL核心逻辑。
10.不要生成测试脚本。
11.不要编写或修改README、使用说明、升级记录或验收报告。
12.不要执行git commit或git push。

二、前端技术要求

若不存在现有前端，使用：

Vue 3
Vite
Vue Router
原生fetch或Axios二选一

要求：

1.仅使用JavaScript。
2.禁止创建.ts、.tsx文件。
3.不得引入TypeScript配置。
4.优先复用现有UI组件库。
5.若项目没有UI组件库，不强制引入大型组件库。
6.可使用原生CSS实现清晰、可用的后台页面。
7.避免引入与功能无关的大量依赖。
8.所有依赖版本以当前npm生态兼容版本为准。
9.不得在前端实现模型调用。
10.不得让前端直接连接Spring AI或模型供应商。
11.不得把API Key写入前端。
12.不得在前端实现真正的权限判断作为安全边界。
13.后端始终是认证和授权的最终依据。

三、前端目录建议

如果新建agent-web，建议结构：

agent-web/
├─src/
│  ├─api/
│  │  ├─http.js
│  │  ├─auth.js
│  │  ├─agent.js
│  │  ├─supervisor.js
│  │  ├─users.js
│  │  ├─roles.js
│  │  └─framework.js
│  ├─components/
│  ├─layouts/
│  ├─router/
│  ├─stores/
│  ├─views/
│  │  ├─LoginView.vue
│  │  ├─DashboardView.vue
│  │  ├─AgentInvokeView.vue
│  │  ├─SupervisorInvokeView.vue
│  │  ├─PermissionDemoView.vue
│  │  ├─UserManagementView.vue
│  │  └─RoleManagementView.vue
│  ├─App.vue
│  └─main.js
├─index.html
├─package.json
└─vite.config.js

具体目录以现有前端为准，不要为了匹配建议大规模搬迁。

四、API基础地址

开发环境使用Vite代理访问后端。

建议：

前端：
http://localhost:5173

后端：
http://localhost:8080

vite.config.js中配置：

/api
→后端地址

要求：

1.组件中不得散落完整后端URL。
2.统一使用相对路径/api。
3.后端地址允许通过Vite环境变量配置。
4.只提交不包含敏感信息的配置。
5.不得在前端.env中写模型API Key。
6.不要创建包含真实密码的环境文件。
7.若现有后端CORS已经支持代理模式，不做无关修改。
8.若必须增加CORS，只允许前端开发地址，不能使用无限制通配配置配合敏感认证头。
9.生产部署方案不在本批实现。

五、Session前端存储

登录成功后保存sessionId。

本演示系统优先使用：

sessionStorage

要求：

1.统一封装Session访问，不允许页面直接散落sessionStorage调用。
2.建议创建：
authSessionStore
3.至少提供：
getSessionId
setSessionId
clearSession
4.不得把密码保存到localStorage或sessionStorage。
5.不得把credentialHash保存到浏览器。
6.不得持久化完整UserSession。
7.当前用户资料可保存在内存状态，并通过/api/auth/me重新获取。
8.关闭浏览器标签后sessionStorage清除是可接受的。
9.本批不改用Cookie。
10.本批不实现Refresh Token。
11.sessionId不得展示在普通页面。
12.控制台不得打印sessionId。

六、统一HTTP客户端

实现统一HTTP客户端。

职责：

1.请求/api/auth/login时不附加X-Session-Id。
2.其他受保护接口自动附加：
X-Session-Id
3.不得把sessionId放入URL。
4.不得把sessionId放进请求Body。
5.统一处理JSON请求和响应。
6.统一处理后端错误结构。
7.HTTP401时：
-清除本地Session。
-清除当前用户状态。
-跳转登录页。
-提示登录已失效。
8.HTTP403时：
-保留登录状态。
-显示权限不足。
-不得自动退出。
9.HTTP404、409、500和503显示后端安全错误信息。
10.不得显示Java堆栈。
11.不得把完整响应对象直接渲染到页面。
12.网络异常显示明确提示。
13.登录请求失败时不得泄漏用户名是否存在。
14.不得在日志中打印请求头。
15.不得自动重试具有写操作的管理请求。

七、认证状态管理

实现轻量认证状态管理。

可以使用：

1.Vue reactive/composable。
2.现有Pinia，如果项目已经引入。
3.不得仅为了本批强制引入新的大型状态库。

状态至少包含：

currentUser
authenticated
loading

currentUser来源：

GET /api/auth/me

至少表达：

userId
username
roles
permissions
createdAt
expiresAt（后端存在时）

要求：

1.应用启动时如果存在sessionId，调用/api/auth/me恢复状态。
2.恢复失败时清理本地Session并进入登录页。
3.不得根据用户名猜测角色。
4.不得在前端伪造权限。
5.导航显示可根据当前permissions优化，但后端仍是最终校验。
6.用户角色或权限更新导致Session被撤销后，下次请求应正确跳转登录页。
7.不得缓存credentialHash。
8.不得显示sessionId。

八、登录页面

实现：

/login

页面包含：

1.用户名输入框。
2.密码输入框。
3.登录按钮。
4.提交加载状态。
5.错误提示。
6.Sample账号提示区域。

Sample账号区域只能展示：

admin
visitor

不得展示环境变量中的密码。

登录流程：

1.校验用户名和密码非空。
2.调用POST /api/auth/login。
3.保存返回的sessionId。
4.调用GET /api/auth/me获取当前身份。
5.进入首页。
6.不得在浏览器控制台打印密码。
7.不得自动填充真实Sample密码。
8.登录失败后清空密码输入。
9.连续提交时禁用按钮。
10.不得区分用户不存在和密码错误。
11.已登录用户访问/login时跳转首页。

九、主布局

实现基础后台布局：

1.顶部栏。
2.侧边导航。
3.当前用户名。
4.角色标签。
5.退出按钮。
6.主内容区域。

导航至少包含：

首页
单Agent调用
Supervisor调用
权限差异演示

拥有security:user:read时显示：

用户管理

拥有security:role:read时显示：

角色管理

要求：

1.导航隐藏只用于体验优化，不是安全边界。
2.用户直接输入管理页面URL时仍会请求后端，权限不足应显示403。
3.退出调用POST /api/auth/logout。
4.退出成功或Session已失效都清除前端状态。
5.不得显示sessionId。
6.不得显示完整权限集合在顶部栏。
7.可以在个人信息区域查看权限详情。

十、路由保护

实现Vue Router路由守卫。

规则：

1.未登录访问受保护页面时跳转/login。
2.已登录访问/login时跳转首页。
3.管理页面可配置前端permission元数据：
-security:user:read
-security:role:read
4.缺少权限时进入403提示页或显示权限不足页面。
5.不得只依赖路由守卫保护后端接口。
6.刷新页面后先恢复认证状态，再决定跳转。
7.避免恢复状态过程中反复跳转。
8.不存在路由显示简洁404页面。

十一、首页

实现DashboardView。

展示：

1.当前用户。
2.角色。
3.Session创建时间。
4.过期时间（存在时）。
5.工具权限摘要。
6.可用Agent数量。
7.可用Supervisor数量。

数据来源：

GET /api/auth/me
GET /api/framework/agents
GET /api/framework/supervisors
GET /api/framework/tools

要求：

1.不得显示systemPrompt。
2.不得显示模型配置。
3.不得显示sessionId。
4.工具列表只代表框架已注册工具，不表示用户全部有权调用。
5.页面需明确区分：
框架注册工具
当前用户工具权限
6.权限通配符tool:*:invoke应显示为“全部工具调用权限”。
7.具体权限显示为可理解标签。

十二、单Agent调用页面

实现：

/agents

功能：

1.加载GET /api/framework/agents。
2.选择agentName。
3.输入message。
4.调用POST /api/agent/invoke。
5.展示最终AgentResult。
6.展示runId和threadId。
7.展示success、content、errorCode和安全metadata。
8.不得展示内部消息历史。
9.不得展示ToolCall完整参数。
10.不得允许客户端设置：
-userId
-sessionId
-roles
-permissions
-systemPrompt
-allowedTools
-maxIterations
11.请求期间禁用提交按钮。
12.支持清空结果。
13.模型不可用时展示明确错误。
14.不得调用/api/dev/react/invoke。

十三、Supervisor调用页面

实现：

/supervisors

功能：

1.加载GET /api/framework/supervisors。
2.选择supervisorName。
3.输入message。
4.调用POST /api/supervisor/invoke。
5.展示最终结果。
6.展示runId和threadId。
7.展示安全metadata。
8.不得展示Supervisor完整消息。
9.不得展示子Agent完整State。
10.不得允许客户端设置memberAgents。
11.不得调用/api/dev/supervisor/invoke。
12.请求期间显示执行中状态。
13.执行可能较慢时保持页面可用，不重复提交。

十四、权限差异演示页面

实现核心演示页面：

/permission-demo

目的：

使用同一业务请求展示ADMIN和VISITOR的工具权限差异。

页面至少提供以下预设场景：

场景一：允许工具

calculator：

请必须使用calculator计算12*(3+5)，并根据真实工具结果回答。

场景二：受限工具

text_search：

请必须使用text_search在以下文本中搜索Java，并根据真实工具结果回答：
Java Agent Framework
Python Service
Java Runtime

场景三：current_time：

请必须使用current_time查询Asia/Shanghai当前时间。

要求：

1.页面显示当前登录用户及角色。
2.页面显示当前用户对应的相关工具权限。
3.选择场景后调用正式：
POST /api/agent/invoke
4.不得调用开发API。
5.默认使用utility_agent；calculator场景可选择calculator_agent。
6.展示最终回答。
7.展示工具权限预期：
-允许
-预计拒绝
-由通配权限允许
8.该预期只用于界面说明，实际结果以后端ACL为准。
9.页面需说明权限拒绝不是登录失败。
10.权限拒绝后如果模型成功解释，HTTP响应可能仍success=true。
11.页面不得伪造工具是否真实执行。
12.可以根据AgentResult metadata展示有限状态，但不得编造toolExecuted字段。
13.如果后端没有公开工具轨迹，不要为了页面增加内部轨迹泄漏接口。
14.页面提示用户分别用admin和visitor登录测试同一请求。
15.不得在同一浏览器页面保存两个用户的sessionId。
16.切换用户必须先退出再登录，或使用不同浏览器窗口。

十五、用户管理页面

实现：

/admin/users

只有拥有security:user:read权限的用户显示入口。

功能：

1.调用GET /api/admin/users。
2.表格展示：
-userId
-username
-roleNames
-enabled
3.创建用户。
4.修改用户角色和enabled状态。
5.重置用户密码。
6.操作后刷新列表。

创建表单：

username
password
roleNames

更新表单：

roleNames
enabled

重置密码表单：

newPassword

要求：

1.角色选项从GET /api/admin/roles获取。
2.不得允许提交permissionCodes给用户。
3.不得显示credentialHash。
4.不得显示用户Session。
5.不得支持删除用户。
6.不得支持修改username。
7.操作当前用户被后端拒绝时，显示明确提示。
8.重置密码输入提交后立即清空。
9.创建或更新成功显示提示。
10.HTTP409显示用户名冲突。
11.HTTP403显示权限不足。
12.Session被撤销的目标用户应在下一次请求时退出。
13.管理员自己的Session不应因操作其他用户被清除。
14.不得在控制台打印用户密码。

十六、角色管理页面

实现：

/admin/roles

只有拥有security:role:read权限时显示入口。

功能：

1.调用GET /api/admin/roles。
2.展示：
-roleName
-description
-permissionCodes
3.创建角色。
4.更新角色描述和权限。
5.不得删除角色。
6.不得修改roleName。

权限编辑要求：

1.支持工具权限：
tool:*:invoke
tool:calculator:invoke
tool:current_time:invoke
tool:echo:invoke
tool:text_search:invoke
2.支持管理权限：
security:user:read
security:user:write
security:role:read
security:role:write
security:session:revoke
3.权限选项集中配置，不得在多个组件重复散落。
4.允许展示后端已有但前端未预置的权限编码。
5.未知权限不能在编辑时被静默删除。
6.更新角色后提示：
受影响用户的旧Session将失效，需要重新登录。
7.不得把ADMIN角色设为前端特殊不可编辑，除非后端已有明确约束。
8.所有安全校验以后端为准。
9.不得在前端自动给ADMIN补权限。
10.权限集合提交前去重。

十七、个人身份信息

提供可在首页或独立区域查看的当前身份信息：

1.userId。
2.username。
3.roles。
4.permissions。
5.createdAt。
6.expiresAt。

要求：

1.来源必须是/api/auth/me。
2.不得显示sessionId。
3.权限按工具权限和管理权限分组。
4.通配工具权限单独说明。
5.不根据角色名推断未返回的权限。
6.Session过期后自动回登录页。

十八、Session失效体验

必须正确处理以下情况：

1.管理员修改用户角色。
2.管理员禁用用户。
3.管理员重置用户密码。
4.管理员修改用户所属角色的权限。
5.用户旧Session被后端撤销。

旧Session下一次请求收到401时：

1.清除sessionStorage中的sessionId。
2.清除当前用户状态。
3.跳转/login。
4.显示：
“登录状态已失效，请重新登录。”
5.不得无限重试原请求。
6.不得保留表单中的密码。
7.不得把401错误当作普通权限不足。
8.403时不得清除Session。

十九、前端权限显示规则

前端可实现：

hasPermission(code)

hasAnyPermission(codes)

hasToolPermission(toolName)

工具权限判断：

1.存在tool:*:invoke时显示允许。
2.存在tool:{toolName}:invoke时显示允许。
3.否则显示无权限。

要求：

1.只用于页面展示和导航控制。
2.不能替代后端ToolAccessControlInterceptor。
3.不能阻止用户构造HTTP请求。
4.不能修改当前用户权限。
5.不能根据roleName=ADMIN直接返回true。
6.权限来源只能是/api/auth/me。
7.不得信任浏览器本地手工修改后的状态作为安全依据。

二十、样式和交互

页面应达到现场演示可用标准：

1.布局清晰。
2.登录页居中。
3.表格可读。
4.表单标签完整。
5.按钮具有加载和禁用状态。
6.成功和错误提示明显。
7.角色、权限使用标签展示。
8.危险操作如禁用用户、重置密码需要确认。
9.不得使用浏览器原始alert作为唯一交互方式，除非项目没有任何提示组件。
10.不得追求复杂动画。
11.支持常见桌面浏览器宽度。
12.无需专门适配移动端。
13.页面文字使用中文。
14.不得混入无关示例页面。

二十一、后端最小修改

原则上本批以前端实现为主。

仅允许后端最小修复：

1.补充必要CORS配置。
2.修复现有DTO字段与接口明显不一致。
3.修复统一错误响应无法被前端解析的问题。
4.修复管理接口路径冲突。
5.修复后端确实阻断联调的问题。

禁止：

1.重写认证架构。
2.重写Tool ACL。
3.改变Session语义。
4.为了前端方便返回credentialHash。
5.返回完整UserSession。
6.返回模型Prompt。
7.返回ToolTrace和内部消息。
8.新增不受认证保护的管理接口。
9.通过前端代理绕过后端认证。

发现后端问题时必须做最小修改并在最终输出说明。

二十二、页面访问权限

公开页面：

/login

认证后页面：

/
/agents
/supervisors
/permission-demo

管理页面：

/admin/users
要求security:user:read

/admin/roles
要求security:role:read

要求：

1.没有Session访问认证页面跳转/login。
2.VISITOR不显示管理菜单。
3.VISITOR直接访问管理路由显示403或跳转无权限页。
4.后端管理API仍返回403。
5.ADMIN可以访问全部管理页面。
6.不得根据username判断是否显示管理页面。

二十三、整体演示流程

应能现场完成：

演示一：ADMIN

1.使用admin登录。
2.首页显示ADMIN和tool:*:invoke。
3.打开权限差异页面。
4.执行calculator，成功。
5.执行text_search，成功。
6.打开用户管理页面。
7.创建新VISITOR用户。
8.打开角色管理页面查看ADMIN与VISITOR权限。

演示二：VISITOR

1.admin退出。
2.使用visitor登录。
3.首页显示有限工具权限。
4.执行calculator，成功。
5.使用相同text_search请求。
6.真实工具被ACL拒绝。
7.Agent最终说明权限不足。
8.侧边栏不显示用户和角色管理。
9.直接访问管理页面无权限。
10.直接请求管理API返回403。

演示三：权限动态变化

1.admin登录。
2.给VISITOR角色增加tool:text_search:invoke。
3.系统撤销所有VISITOR旧Session。
4.visitor旧页面下一次请求返回401。
5.visitor重新登录。
6.再次执行text_search。
7.工具调用成功。
8.证明Session权限快照和重新登录生效。

二十四、错误处理

页面需合理处理：

400：
显示参数错误。

401：
清理Session并跳转登录。

403：
显示权限不足，不退出。

404：
显示资源不存在。

409：
显示用户名或角色冲突。

500：
显示系统内部错误，不展示堆栈。

503：
显示模型服务暂不可用。

要求：

1.优先显示后端安全message。
2.无message时使用前端通用文案。
3.不得展示HTML异常页。
4.不得展示Java类名。
5.不得展示原始Axios/fetch错误对象。
6.不得吞掉错误而无提示。

二十五、前端构建

在agent-web执行：

npm install
npm run build

如已有锁文件：

1.优先使用对应包管理器。
2.不要同时创建package-lock.json和其他锁文件。
3.不得随意删除现有锁文件。
4.构建必须成功。
5.不得依赖全局安装包。
6.不得跳过真实构建。

后端继续执行：

mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests

要求：

1.前端构建失败时继续修复。
2.后端编译失败时继续修复。
3.不得伪造成功结果。
4.前端构建产物不强制复制到Spring Boot static目录。
5.保持前后端分离启动。
6.不得为本批引入Docker。

二十六、联调验收

启动后端和前端，实际验证：

1.登录页可访问。
2.admin登录成功。
3.visitor登录成功。
4.密码错误有正确提示。
5.刷新页面可恢复登录状态。
6.登出成功。
7.Session失效后跳转登录。
8.Agent页面可调用正式接口。
9.Supervisor页面可调用正式接口。
10.ADMIN调用calculator成功。
11.VISITOR调用calculator成功。
12.ADMIN调用text_search成功。
13.VISITOR调用text_search被ACL拒绝并由模型解释。
14.ADMIN可访问用户管理。
15.VISITOR访问用户管理返回无权限。
16.ADMIN可创建用户。
17.ADMIN可修改用户角色。
18.ADMIN可禁用用户。
19.ADMIN可重置密码。
20.ADMIN可查看和修改角色权限。
21.角色权限变化后受影响Session失效。
22.重新登录后获得新权限。
23.前端不发送userId、roles和permissions。
24.前端请求自动携带X-Session-Id。
25.响应和页面不出现credentialHash或sessionId。
26.浏览器控制台无明显运行错误。
27.前端生产构建成功。
28.后端编译和打包成功。

二十七、安全检查

必须检查：

1.前端源码没有Sample密码。
2.前端源码没有模型API Key。
3.前端源码没有固定sessionId。
4.密码不进入sessionStorage或localStorage。
5.浏览器控制台不打印密码和sessionId。
6.管理权限不根据ADMIN字符串硬编码。
7.工具权限不根据角色名称硬编码。
8.前端不能提升后端权限。
9.所有管理请求使用正式认证接口。
10.未登录用户不能调用正式Agent和Supervisor接口。
11.开发API未被前端调用。
12.后端依旧是最终权限边界。

二十八、本批禁止实现

禁止：

1.TypeScript。
2.React、Angular或其他第二套前端框架。
3.前端直连模型。
4.前端保存API Key。
5.前端硬编码Sample密码。
6.前端使用开发Agent接口。
7.前端自行伪造工具结果。
8.删除用户。
9.删除角色。
10.用户自助注册。
11.找回密码。
12.JWT、OAuth和Cookie认证。
13.Spring Security Web改造。
14.数据库和Redis。
15.HITL。
16.Checkpoint。
17.上下文裁剪。
18.长期记忆。
19.RAG和MCP。
20.SSE。
21.真实金山云业务。
22.Docker部署。
23.测试脚本。
24.README、使用说明、升级记录或验收报告。
25.git commit和git push。

二十九、代码质量

1.组件职责清晰。
2.API调用集中在api目录。
3.Session访问集中封装。
4.不在每个页面重复实现401处理。
5.不在组件中散落完整URL。
6.不复制大量重复表单逻辑。
7.不过度抽象简单页面。
8.不创建无意义Manager、Facade或Util。
9.表单状态在请求后正确清理。
10.异步请求使用try/finally恢复loading。
11.列表渲染使用稳定key。
12.不直接修改props。
13.不忽略Promise错误。
14.不使用TypeScript语法。
15.所有实现以真实构建和联调结果为准。

三十、最终输出

只输出：

1.新增和修改文件清单。
2.前端技术栈和目录。
3.Session前端存储方式。
4.HTTP客户端和401/403处理。
5.路由和页面权限规则。
6.登录与登出流程。
7.单Agent和Supervisor调用页面。
8.权限差异演示页面。
9.用户管理页面。
10.角色与权限管理页面。
11.后端最小修改。
12.ADMIN和VISITOR完整演示结果。
13.Session权限变更和重新登录演示结果。
14.前端安装和构建结果。
15.后端编译和打包结果。
16.实际完成的浏览器联调结果。
17.因缺少模型、Sample密码或环境无法完成的验证及准确原因。
18.发现但未处理的非本批问题。

前端或后端构建失败时继续修复直到成功；无法修复时说明准确阻塞原因，不得伪造成功结果。

===PHASE5_BATCH5_END===