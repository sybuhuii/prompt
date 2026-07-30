继续开发通用 Agent 框架。第一阶段前两批应已完成：

1. Maven 多模块工程骨架已建立。
2. `agent-core` 领域模型和接口已实现。
3. 根目录执行 Maven 编译能够通过。

当前只执行第一阶段第 3 批：实现 Agent 与 Tool 注册中心、运行时基础扩展点、Spring Boot 装配和最小查询入口。

本批仍然不要实现真正的模型调用、ReAct、Supervisor、权限校验和持久化。

## 一、实现 AgentRegistry

在 `agent-runtime` 中创建：

```text
com.ksyun.agent.runtime.registry
```

定义接口：

```java
public interface AgentRegistry {

    void register(AgentDefinition definition);

    Optional<AgentDefinition> find(String name);

    AgentDefinition getRequired(String name);

    Collection<AgentDefinition> list();

    boolean contains(String name);
}
```

实现：

```text
DefaultAgentRegistry
```

要求：

* 使用 `ConcurrentHashMap`
* 支持线程安全注册和查询
* Agent 名称不能为空
* 重复注册相同名称时明确拒绝，不允许静默覆盖
* `list()` 返回不可变快照
* 找不到 Agent 时，`getRequired()` 抛出带有 `AGENT_NOT_FOUND` 错误码的框架异常
* 不依赖 Spring 容器完成核心逻辑

## 二、实现 ToolRegistry

在同一包下定义：

```java
public interface ToolRegistry {

    void register(AgentTool tool);

    Optional<AgentTool> find(String name);

    AgentTool getRequired(String name);

    Collection<AgentTool> list();

    boolean contains(String name);
}
```

实现：

```text
DefaultToolRegistry
```

要求：

* 使用 `ConcurrentHashMap`
* 根据 `tool.definition().name()` 注册
* 工具名称不能为空
* 重复注册明确失败
* `list()` 返回不可变快照
* 找不到工具时抛出带有 `TOOL_NOT_FOUND` 错误码的框架异常

## 三、Provider 自动注册能力

实现：

```text
AgentProviderRegistrar
ToolProviderRegistrar
```

职责：

### AgentProviderRegistrar

接收多个 `AgentProvider`，将其返回的全部 `AgentDefinition` 注册到 `AgentRegistry`。

### ToolProviderRegistrar

接收多个 `ToolProvider`，将其返回的全部 `AgentTool` 注册到 `ToolRegistry`。

要求：

* Provider 返回 null 时按空集合处理或明确拒绝，但不能产生难以定位的空指针
* 注册失败时保留清晰错误信息
* 不吞掉重复名称异常

这套机制用于未来业务模块通过 SPI 注册 Agent 和 Tool。

## 四、运行 ID 生成器

在：

```text
com.ksyun.agent.runtime.run
```

定义：

```java
public interface RunIdGenerator {
    String nextRunId();
}
```

实现：

```text
UuidRunIdGenerator
```

生成不带空格且全局唯一性足够的 runId。

当前不需要引入雪花算法或分布式 ID 服务。

## 五、预留运行时接口

在：

```text
com.ksyun.agent.runtime
```

定义：

```java
public interface AgentRuntime
```

包含：

```java
AgentResult start(
        AgentTask task,
        RunContext context
);

AgentResult resume(
        String runId,
        ApprovalDecision decision,
        RunContext context
);
```

当前只定义接口，不创建伪造的完整实现，不要编写假的 ReAct 逻辑。

## 六、预留 Tool 执行扩展点

在：

```text
com.ksyun.agent.runtime.tool
```

定义以下接口。

### ToolExecutionChain

```java
ToolResult proceed(ToolInvocation invocation);
```

### ToolInterceptor

```java
ToolResult intercept(
        ToolInvocation invocation,
        ToolExecutionChain chain
);
```

### ToolInvocationGateway

```java
ToolResult invoke(ToolInvocation invocation);
```

本批只定义扩展点，不实现：

* SchemaValidationInterceptor
* ToolPermissionInterceptor
* ApprovalInterceptor
* AuditInterceptor
* 真实工具执行链

这些内容留到后续阶段。

## 七、预留模型调用网关

在：

```text
com.ksyun.agent.runtime.model
```

定义：

```java
public interface ModelInvocationGateway {

    ModelResponse invoke(
            ModelRequest request,
            RunContext context
    );
}
```

本批不要调用 Spring AI。

## 八、Application 查询服务

在 `agent-application` 中创建：

```text
com.ksyun.agent.application.framework
```

实现：

```text
FrameworkQueryService
```

依赖：

```text
AgentRegistry
ToolRegistry
```

提供查询：

```java
Collection<AgentDefinition> listAgents();

Collection<ToolDefinition> listTools();
```

注意：

* application 只依赖 runtime/core 的接口
* 不直接访问 Spring 容器
* 不实现注册逻辑

## 九、Spring Boot 配置装配

在 `agent-infrastructure` 中创建配置类，负责把以下默认实现注册成 Bean：

```text
DefaultAgentRegistry
DefaultToolRegistry
UuidRunIdGenerator
FrameworkQueryService
```

同时扫描所有：

```text
AgentProvider
ToolProvider
```

并在应用启动阶段通过 Registrar 注册。

要求：

* 没有 Provider 时应用仍然可以正常启动
* 不允许 infrastructure 中出现具体业务 Agent
* 不实现 Spring AI 和 LangGraph4j 配置
* 不实现任何 Store 的内存版本

## 十、最小 API

在 `agent-api` 中创建一个简单的框架查询 Controller，例如：

```text
GET /api/framework/agents
GET /api/framework/tools
GET /api/framework/health
```

要求：

### `/api/framework/agents`

返回当前已注册 Agent 的基础元信息。

### `/api/framework/tools`

返回当前已注册 Tool 的基础元信息。

不得直接暴露：

* systemPrompt 完整内容
* 内部实现类名
* 权限敏感数据

可以定义专门的响应 DTO。

### `/api/framework/health`

返回：

```json
{
  "status": "UP",
  "framework": "agent-platform"
}
```

该接口只用于验证多模块装配成功。

## 十一、Bootstrap 启动

确认 `agent-bootstrap` 中的：

```text
AgentPlatformApplication
```

能够扫描：

```text
com.ksyun.agent
```

启动模块需要正确引入：

```text
agent-api
agent-application
agent-runtime
agent-infrastructure
```

不要在启动类中手工 new 大量组件。

## 十二、本批不要实现的内容

禁止提前实现：

* Spring AI ChatClient
* LangGraph4j StateGraph
* ReAct 图
* Supervisor 图
* 示例 Agent
* 示例 Tool
* SessionStore 内存实现
* CheckpointStore 内存实现
* MemoryStore 内存实现
* RBAC
* ACL
* HITL
* SSE
* 前端

## 十三、验收要求

依次执行：

```bash
mvn clean compile -DskipTests
```

然后执行：

```bash
mvn -pl agent-bootstrap -am package -DskipTests
```

确保：

1. 全部模块编译通过。
2. `agent-bootstrap` 能完成打包。
3. Spring Bean 不存在循环依赖。
4. 没有 Provider 时项目仍能成功装配。
5. 注册中心可以处理空注册状态。
6. 模块依赖方向没有被破坏。
7. 没有引入当前阶段不需要的 Agent 框架实现。

若环境允许，启动应用并验证三个接口能够访问；若无法启动，至少必须保证完整编译和打包通过。

## 十四、输出要求

完成后：

1. 列出新增和修改文件。
2. 说明 AgentRegistry 和 ToolRegistry 的线程安全与重复注册策略。
3. 说明 Provider 自动注册流程。
4. 说明 Spring Bean 的装配关系。
5. 给出实际 Maven 编译和打包结果。
6. 发现错误后继续修复直到通过。

不要生成或修改 README、使用说明、升级记录、验收报告等文档。

不要生成测试脚本。
