你正在继续开发基于Spring AI+LangGraph4j的通用Agent框架。

必须完整阅读并遵守根目录`AGENTS.md`。

已完成：

1.身份、Session、RBAC、工具ACL和多用户隔离。
2.单Agent ReAct执行引擎。
3.Supervisor多Agent编排。
4.HITL中断、Checkpoint、审批和恢复。
5.上下文消息数裁剪、Token裁剪和LLM摘要。
6.ReAct和Supervisor上下文窗口接入。
7.Web动态页面。

当前只执行第八阶段第1批：

1.建立长期记忆稳定领域模型。
2.交付`MemoryStore`接口。
3.提供线程安全`InMemoryMemoryStore`。
4.实现基于已认证用户的长期记忆应用服务。
5.按`userId+namespace+key`严格隔离数据。
6.实现长期记忆配置和Spring Bean装配。
7.提供安全的框架能力查询。

本批不修改CheckpointStore，不实现短期会话续接，不让LLM自动提取记忆，不把记忆注入ReasonNode，不新增前端页面。

# 一、强制执行规则

1.本提示词指定的类型名、字段语义、方法和错误码属于强制交付项。
2.不得使用`UserMemoryV2`、`MemoryRepository`等近似类型替代指定类型。
3.已有同名类型完全等价时复用；部分满足时最小扩展。
4.`MemoryStore`与`CheckpointStore`必须保持两套独立抽象。
5.不得把长期记忆存进Checkpoint。
6.不得把运行State或完整消息历史存进MemoryStore。
7.不得让客户端决定userId。
8.不得修改AgentDefinition保存用户记忆。
9.不得提前接入ReAct、Supervisor或模型调用。
10.不得在未读完提示词前修改代码。

# 二、执行前检查

必须检查：

1.根目录`AGENTS.md`。
2.现有`MemoryStore`是否已经存在。
3.现有Memory、Profile、Preference、Fact相关模型。
4.`CheckpointStore`和`AgentCheckpoint`。
5.`UserSession`和`RunContext`。
6.`UserStore`和现有内存Store代码风格。
7.现有ID生成器。
8.现有`Clock` Bean。
9.`AgentErrorCode`和异常体系。
10.现有Application Service实现方式。
11.现有Spring配置和条件装配方式。
12.`FrameworkQueryService`及框架查询Controller。
13.当前Java版本。
14.集合不可变处理方式。
15.项目是否已存在命名空间或复合Key模型。

要求：

1.以仓库真实代码为准。
2.若已有`MemoryStore`，不得新建第二个同名或近似接口。
3.若已有接口不满足本批要求，做最小兼容扩展。
4.发现前序阻断问题时只做最小修复。
5.不得修改ReAct、Supervisor、HITL和上下文主链。
6.不得提前开发第2批短期记忆。

# 三、记忆分层边界

必须明确：

## 短期记忆

使用：

`CheckpointStore`

隔离键：

`threadId`

保存内容：

- 会话内运行状态
- 消息历史
- 中间变量
- 待执行节点
- HITL恢复状态

生命周期：

- 与会话或执行线程绑定

本批不实现。

## 长期记忆

使用：

`MemoryStore`

隔离键：

`userId+namespace`

保存内容：

- 用户画像
- 用户偏好
- 用户事实
- 用户长期规则

生命周期：

- 跨threadId
- 跨多次会话
- 独立于一次Agent运行

本批实现。

禁止：

1.使用MemoryStore保存ReactAgentState。
2.使用MemoryStore保存Checkpoint。
3.使用CheckpointStore保存用户画像。
4.使用threadId作为长期记忆唯一命名空间。
5.使用sessionId作为长期记忆归属键。
6.将两种Store继承同一个通用无类型Store接口。

# 四、MemoryCategory

在`agent-core`长期记忆领域包中必须新增或复用：

`MemoryCategory`

枚举值必须包含：

- `PROFILE`
- `PREFERENCE`
- `FACT`
- `RULE`

语义：

1.PROFILE：较稳定的用户身份或背景信息。
2.PREFERENCE：用户明确表达的偏好。
3.FACT：与用户相关、可长期复用的事实。
4.RULE：用户希望Agent长期遵守的个人规则。

要求：

1.不得使用自由字符串代替枚举。
2.不得包含SHORT_TERM或CHECKPOINT。
3.不得包含模型供应商信息。
4.不得通过category决定数据权限。
5.所有类别的数据仍按userId隔离。

# 五、MemoryEntry

在`agent-core`必须新增或最小完善不可变：

`MemoryEntry`

字段必须包含：

- `memoryId`
- `userId`
- `namespace`
- `key`
- `value`
- `category`
- `metadata`
- `version`
- `createdAt`
- `updatedAt`

要求：

1.memoryId不能为空。
2.userId不能为空。
3.namespace不能为空。
4.key不能为空。
5.value不能为空。
6.category不能为空。
7.metadata为不可变`Map<String,String>`。
8.version必须大于等于0。
9.createdAt和updatedAt不能为空。
10.updatedAt不得早于createdAt。
11.所有字符串进行trim。
12.不得包含sessionId。
13.不得包含threadId作为长期记忆归属。
14.不得包含完整UserSession。
15.不得包含RunContext。
16.不得包含AgentMessage列表。
17.不得包含Spring AI、LangGraph4j或Servlet类型。
18.不得将密码、API Key和credentialHash作为合法记忆值。
19.模型必须保持不可变。
20.更新记忆时保留memoryId和createdAt。

# 六、MemoryStoreKey

在`agent-core`必须新增不可变：

`MemoryStoreKey`

字段必须包含：

- `userId`
- `namespace`
- `key`

要求：

1.三个字段均不能为空。
2.统一trim。
3.必须实现稳定的值相等语义。
4.不得包含sessionId。
5.不得包含threadId。
6.不得使用字符串拼接作为Store唯一内部Key。
7.不得使用`userId+":"+key`等容易冲突的字符串。
8.适合作为ConcurrentHashMap的Key。
9.不得依赖Spring。

# 七、MemoryStore接口

在`agent-core`必须存在：

`MemoryStore`

接口至少提供：

```java
MemoryEntry put(MemoryEntry entry);

Optional<MemoryEntry> get(
        String userId,
        String namespace,
        String key
);

Collection<MemoryEntry> list(
        String userId,
        String namespace
);

boolean delete(
        String userId,
        String namespace,
        String key
);
```

要求：

1.`put`采用明确upsert语义。
2.相同`userId+namespace+key`视为同一记忆。
3.首次写入创建记录。
4.再次写入更新value、category、metadata和updatedAt。
5.更新时version加1。
6.更新时memoryId和createdAt保持不变。
7.get不存在返回`Optional.empty()`。
8.list返回不可变快照。
9.list只返回指定用户和namespace的数据。
10.delete不存在时返回false。
11.delete只删除精确复合Key。
12.不得提供不带userId的get、list和delete。
13.不得提供“列出全部用户记忆”的公共方法。
14.接口不得依赖Spring。
15.接口不得依赖数据库。
16.接口不得执行模型。
17.接口不得访问CheckpointStore。
18.不得返回null。

如现有接口方法名不同但语义完全等价，可保留兼容方法；最终输出必须说明实际接口。

# 八、MemoryIdGenerator

检查现有通用ID生成器。

若没有适合长期记忆的ID生成能力，在`agent-core`必须新增：

`MemoryIdGenerator`

方法：

```java
String generate();
```

在`agent-infrastructure`提供默认实现。

要求：

1.可使用UUID v4。
2.不得使用数据库自增假设。
3.不得包含userId。
4.不得包含namespace和key。
5.不得使用sessionId。
6.不得在日志打印ID与完整value的组合。
7.已有通用安全ID生成器可直接复用时，不得重复新增。

# 九、InMemoryMemoryStore

在`agent-infrastructure`必须新增：

`InMemoryMemoryStore implements MemoryStore`

要求：

1.使用`ConcurrentHashMap<MemoryStoreKey,MemoryEntry>`。
2.不得使用static Map。
3.不得使用普通HashMap配合不完整同步。
4.put使用原子操作实现upsert。
5.首次put保存传入memoryId和createdAt。
6.更新时保留原memoryId和createdAt。
7.更新时version=旧version+1。
8.updatedAt使用传入的新时间或Store约定的时间，但行为必须明确。
9.get返回不可变MemoryEntry。
10.list只扫描或索引指定userId和namespace。
11.list返回稳定顺序，建议按key升序。
12.list返回不可变列表。
13.delete只删除精确MemoryStoreKey。
14.不同用户相同namespace和key必须保存为两条独立记录。
15.不得因用户A更新而影响用户B。
16.不得返回内部Map视图。
17.不得缓存当前用户。
18.不得使用ThreadLocal。
19.同一个Bean支持并发读写。
20.不得访问UserStore、SessionStore或CheckpointStore。
21.不得记录完整value和metadata。
22.不得实现持久化文件写入。
23.不得自动过期或清理记忆。

# 十、长期记忆命名空间

本批采用：

```text
userId
└─namespace
   └─key
```

默认namespace建议为：

`profile`

允许的示例：

- `profile`
- `preferences`
- `facts`
- `rules`

要求：

1.namespace属于逻辑分组，不属于权限边界替代品。
2.所有操作仍必须携带userId。
3.不同namespace中的相同key允许共存。
4.namespace和key使用稳定安全字符。
5.禁止路径穿越语义：
- `../`
- `..\`
- 绝对路径
  6.不得将namespace映射为文件系统目录。
  7.不得通过namespace访问其他用户数据。
  8.不得使用threadId作为长期namespace前缀。

# 十一、MemoryWriteCommand

在`agent-application`必须新增不可变：

`MemoryWriteCommand`

字段必须包含：

- `namespace`
- `key`
- `value`
- `category`
- `metadata`

要求：

1.不得包含userId。
2.不得包含memoryId。
3.不得包含version。
4.不得包含createdAt和updatedAt。
5.不得包含sessionId或threadId。
6.userId必须来自已认证UserSession。
7.metadata为不可变Map。
8.不得包含HTTP对象。
9.不得包含模型响应。
10.不得包含任意Agent消息列表。

# 十二、MemoryView

在`agent-application`必须新增不可变：

`MemoryView`

字段至少包含：

- `memoryId`
- `namespace`
- `key`
- `value`
- `category`
- `metadata`
- `version`
- `createdAt`
- `updatedAt`

要求：

1.不得返回userId给普通前端结果，当前用户已经明确。
2.不得返回sessionId。
3.不得返回内部StoreKey。
4.不得返回密码或credentialHash。
5.集合和Map保持不可变。
6.用于后续API和前端，不依赖Spring Web。

# 十三、LongTermMemoryApplicationService

在`agent-application`必须新增纯Java：

`LongTermMemoryApplicationService`

依赖：

- `MemoryStore`
- `MemoryIdGenerator`
- `Clock`
- 长期记忆配置或验证器

至少提供：

```java
MemoryView put(
    UserSession operator,
    MemoryWriteCommand command
);

Optional<MemoryView> get(
    UserSession operator,
    String namespace,
    String key
);

Collection<MemoryView> list(
    UserSession operator,
    String namespace
);

boolean delete(
    UserSession operator,
    String namespace,
    String key
);
```

职责：

1.operator不能为空。
2.userId只能取`operator.userId`。
3.不得从command读取userId。
4.统一验证namespace、key、value和metadata。
5.读取现有MemoryEntry。
6.首次写入生成memoryId。
7.首次写入version=0。
8.首次写入createdAt=updatedAt=Clock.instant()。
9.更新时复用原memoryId和createdAt。
10.更新时由Store执行原子version增加。
11.映射为MemoryView。
12.get不存在返回Optional.empty。
13.list只查询当前用户。
14.delete只删除当前用户精确Key。
15.不得访问CheckpointStore。
16.不得调用模型。
17.不得自动提取用户偏好。
18.不得注入Agent Prompt。
19.不得使用ThreadLocal。
20.保持无状态和线程安全。

# 十四、输入校验

必须统一校验：

## namespace

1.不能为空。
2.最大长度建议64。
3.只允许字母、数字、`_`、`-`和`.`。
4.不得包含空格和路径分隔符。

## key

1.不能为空。
2.最大长度建议128。
3.只允许稳定可读字符。
4.不得包含换行。
5.不得包含路径穿越。

## value

1.不能为空。
2.最大长度由配置决定。
3.默认建议4096字符。
4.不得只包含空白。
5.不得包含明显的credentialHash字段。
6.不得将完整模型Prompt作为记忆写入。
7.不得将完整聊天记录作为单条记忆写入。

## metadata

1.条目数量受配置限制。
2.key和value长度受限制。
3.禁止敏感key：
- password
- credential
- token
- apiKey
- secret
- authorization
- sessionId
  4.敏感key匹配忽略大小写。
  5.发现敏感key时明确拒绝，不得仅脱敏后保存。
  6.metadata不能包含嵌套对象。

可以实现纯Java：

`MemoryEntryValidator`

不得在Application Service中散落全部校验代码。

# 十五、错误码

检查现有`AgentErrorCode`。

缺失时最小新增，名称固定：

- `MEMORY_NOT_FOUND`
- `INVALID_MEMORY_ENTRY`
- `MEMORY_STORE_FAILED`

语义：

1.输入非法使用`INVALID_MEMORY_ENTRY`。
2.直接要求必须存在的记忆不存在时使用`MEMORY_NOT_FOUND`。
3.Store内部不可恢复异常使用`MEMORY_STORE_FAILED`。
4.list为空不是错误。
5.delete不存在返回false，不默认抛异常。
6.不得把记忆错误转换为Checkpoint错误。
7.不得创建多个同义错误码。
8.错误信息不得包含完整value和metadata。

# 十六、长期记忆配置

必须增加或复用类型安全配置：

```yaml
agent:
  memory:
    enabled: true
    backend: in-memory
    default-namespace: profile
    max-namespace-length: 64
    max-key-length: 128
    max-value-length: 4096
    max-metadata-entries: 16
    max-metadata-key-length: 64
    max-metadata-value-length: 256
```

要求：

1.配置类位于基础设施或现有配置模块。
2.所有限制必须大于0。
3.default-namespace必须符合namespace规则。
4.backend本批只支持`in-memory`。
5.不得自动创建数据库配置。
6.不得包含API Key。
7.不得在Application Service中硬编码另一套默认值。
8.enabled=false时不提供长期记忆业务Bean。
9.MemoryStore自定义Bean存在时允许替换内存实现。
10.配置非法时启动失败并给出明确原因。

# 十七、Spring Bean装配

在`agent-infrastructure`必须装配：

- `MemoryStore→InMemoryMemoryStore`
- `MemoryIdGenerator`或复用现有生成器
- `MemoryEntryValidator`
- `LongTermMemoryApplicationService`

要求：

1.core和application实现保持纯Java。
2.使用`@Configuration`和`@Bean`。
3.默认MemoryStore使用`@ConditionalOnMissingBean`。
4.受`agent.memory.enabled=true`控制。
5.不得在bootstrap主类手工new。
6.不得产生Bean循环依赖。
7.无模型配置时MemoryStore仍能装配。
8.不得注入ModelInvocationGateway。
9.不得注入CheckpointStore到长期记忆服务。
10.不得创建Fake持久化实现。
11.不得创建数据库和Redis Bean。
12.不得在Sample关闭时自动关闭MemoryStore。

# 十八、框架查询能力

必须扩展现有框架查询能力，优先复用：

```text
GET /api/framework/memory
```

若已有统一框架能力接口，则不要创建重复Controller。

至少展示：

- `enabled`
- `longTermMemoryAvailable`
- `backend`
- `defaultNamespace`
- `namespaceIsolation=userId`
- `supportsPut=true`
- `supportsGet=true`
- `supportsList=true`
- `supportsDelete=true`
- `shortTermStore=CheckpointStore`
- `longTermStore=MemoryStore`
- `storesSeparated=true`

要求：

1.不得暴露用户记忆内容。
2.不得暴露MemoryStore内部Map。
3.不得暴露Bean完整类名。
4.不得暴露用户数量和记忆数量。
5.不得暴露API Key。
6.接口无需登录即可沿用现有框架查询策略。
7.无模型配置时可查询。
8.enabled=false时明确显示不可用。

本批不得提供普通用户记忆HTTP管理接口。

# 十九、多用户隔离

必须验证以下语义：

1.用户A写入：
`preferences/language=Python`
2.用户B写入：
`preferences/language=Java`
3.两条记录必须同时存在。
4.A读取只能得到Python。
5.B读取只能得到Java。
6.A删除自己的记录不能删除B的记录。
7.相同key在不同namespace可以共存。
8.相同namespace和key在不同userId下完全隔离。
9.不得根据username作为Store隔离Key。
10.不得使用sessionId作为长期归属。
11.用户重新登录后仍通过同一userId读取长期记忆。
12.不同threadId不影响长期记忆读取。
13.本批暂不通过Agent运行链验证不同threadId。

# 二十、更新和版本语义

对同一`userId+namespace+key`连续put：

首次：

```text
version=0
memoryId=M1
createdAt=T1
updatedAt=T1
```

第二次：

```text
version=1
memoryId=M1
createdAt=T1
updatedAt=T2
```

要求：

1.memoryId保持不变。
2.createdAt保持不变。
3.updatedAt更新。
4.version严格增加1。
5.不得创建重复记录。
6.并发更新使用Store原子操作。
7.不得由Application Service先读再无条件覆盖造成丢失更新。
8.最终更新值以实际原子操作结果为准。
9.本批不实现客户端version条件更新。
10.不得使用version作为权限凭证。

# 二十一、敏感信息限制

长期记忆不得保存：

1.密码。
2.credentialHash。
3.Session ID。
4.X-Session-Id。
5.API Key。
6.Authorization Header。
7.私钥。
8.完整RunContext。
9.完整UserSession。
10.完整Checkpoint。
11.完整聊天消息历史。
12.完整Prompt。
13.模型原始响应对象。
14.异常堆栈。

日志允许记录：

- userId
- namespace
- key
- category
- operation
- version
- success

日志禁止记录：

- value
- 完整metadata
- sessionId
- 密钥和Token

# 二十二、本批禁止实现

禁止：

1.修改CheckpointStore。
2.短期会话续接。
3.按threadId加载历史。
4.保存ReactAgentState到MemoryStore。
5.把长期记忆注入ReasonNode。
6.让LLM自动提取偏好。
7.记忆写入工具。
8.记忆检索工具。
9.Memory Controller。
10.前端记忆页面。
11.跨会话演示。
12.用户主动写记忆HTTP接口。
13.数据库、Redis或文件持久化。
14.向量检索。
15.Embedding。
16.记忆相似度搜索。
17.记忆过期和清理任务。
18.记忆冲突自动合并。
19.SSE。
20.测试脚本、README和Git操作。

# 二十三、边界场景

必须通过代码检查或现有可执行能力确认：

## 首次写入

1.创建MemoryEntry。
2.version=0。
3.createdAt等于updatedAt。
4.能够立即get。

## 更新记忆

1.相同复合Key执行upsert。
2.memoryId不变。
3.version增加。
4.updatedAt更新。
5.list中不出现重复记录。

## 用户隔离

1.A和B写相同namespace和key。
2.两者value互不覆盖。
3.各自list只返回自己的记录。

## namespace隔离

同一用户写入：

- `profile/language`
- `preferences/language`

预期两条独立记录。

## 空列表

1.不存在记忆时返回不可变空集合。
2.不抛MEMORY_NOT_FOUND。

## 删除

1.首次删除返回true。
2.再次删除返回false。
3.不影响其他用户相同key。

## 非法输入

1.空namespace拒绝。
2.空key拒绝。
3.空value拒绝。
4.value超长拒绝。
5.敏感metadata key拒绝。
6.路径穿越namespace拒绝。
7.错误响应不包含完整value。

## 并发更新

同一复合Key并发put时：

1.不产生两个记录。
2.MemoryStoreKey唯一。
3.version更新行为明确。
4.Store内部结构不损坏。

不得为验证新增隐藏Controller或测试脚本。

# 二十四、编译验收

执行：

```bash
mvn clean compile -DskipTests
mvn -pl agent-bootstrap -am package -DskipTests
```

必须逐项确认：

1.全部模块编译通过。
2.agent-bootstrap打包通过。
3.`MemoryCategory`真实存在。
4.`MemoryEntry`真实存在。
5.`MemoryStoreKey`真实存在。
6.`MemoryStore`真实存在。
7.`InMemoryMemoryStore`真实存在。
8.`MemoryWriteCommand`真实存在。
9.`MemoryView`真实存在。
10.`LongTermMemoryApplicationService`真实存在。
11.`MemoryEntryValidator`真实存在。
12.MemoryStore与CheckpointStore保持独立。
13.长期记忆使用userId隔离。
14.相同key的不同用户互不覆盖。
15.put具有明确upsert和version语义。
16.集合及metadata不可变。
17.无模型配置时Memory Bean可装配。
18.框架查询显示Store分层。
19.没有修改ReAct、Supervisor和Checkpoint主链。
20.没有Agent记忆注入、API和前端。
21.没有数据库或Redis。
22.git diff没有无关修改。

# 二十五、最终输出

最终输出必须逐项对应实施清单，只输出：

1.新增和修改文件清单。
2.短期和长期记忆的分层边界。
3.`MemoryCategory`定义。
4.`MemoryEntry`字段与不可变规则。
5.`MemoryStoreKey`隔离语义。
6.`MemoryStore`最终接口。
7.`InMemoryMemoryStore`线程安全和upsert实现。
8.version更新语义。
9.`LongTermMemoryApplicationService`调用流程。
10.输入和敏感信息校验。
11.多用户与namespace隔离方式。
12.配置及Spring Bean装配。
13.框架查询能力。
14.边界场景实际检查结果。
15.编译和打包结果。
16.未完成验证及准确原因。
17.发现但未处理的问题。

不得将计划描述为已完成，不得仅输出“编译通过”，不得遗漏强制交付类型。

编译或打包失败时继续修复；无法修复时说明准确阻塞原因，不得伪造成功。

===PHASE8_BATCH1_END===