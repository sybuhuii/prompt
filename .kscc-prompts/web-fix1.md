# Agent Platform 前端交互模型纠正及遗留问题修复任务

## 一、任务目标

修复当前前端错误的产品交互模型，并处理上一轮审查发现的全部遗留问题。

本项目的正确业务关系是：

```text
用户 → Supervisor → Agent → Tool
```

必须明确：

- 用户只与 Supervisor 交互。
- Supervisor 负责理解用户请求、拆分任务并调度 Agent。
- Agent 是系统内部执行单元，用户不知道系统中有哪些 Agent，也不能选择 Agent。
- 前端不得向普通用户展示 Agent 列表、Agent 选择器、Agent 模式或 Agent 调用入口。
- 后端 Agent 接口目前只用于测试 Agent 功能。本次不要删除或修改这些后端测试接口，但生产前端不得调用它们。
- 登录后不能再出现“请选择 Agent 或 Supervisor”的提示。

本次同时修复：

1. 用户错误地直接选择和调用 Agent。
2. Markdown 代码块顺序被破坏。
3. 取消请求后立即发送新消息时的竞态。
4. 401、登出和切换用户时未清理查询缓存。
5. API 错误响应没有经过 Zod 校验。
6. E2E 测试存在无效断言。
7. Shiki 打包了大量不需要的语言和主题。
8. 修复台账位置和内容不符合要求。
9. Playwright 验证没有真正完成。

---

## 二、强制执行规则

开始修改前，必须完整阅读：

1. 仓库根目录 `AGENTS.md`
2. 根目录或已指定位置的 `web.md`
3. 当前前端源码
4. 当前后端真实 Controller、请求 DTO、响应 DTO 和枚举
5. 当前 `package.json`、锁文件、Vite、Vitest、Playwright 配置
6. 已有修复台账

以仓库真实代码为准，禁止凭记忆猜测接口、字段、路由、状态枚举或组件名称。

不得：

- 跳过本提示词中的任何要求。
- 只修改文档而不修改真实实现。
- 通过类型断言掩盖契约问题。
- 创建 Fake Agent、Fake Supervisor 或固定回复。
- 使用 Mock 冒充真实端到端验证。
- 修改后端 Agent/Supervisor 业务架构。
- 删除后端用于测试 Agent 的接口。
- 引入第二套状态管理、HTTP 客户端或聊天传输链路。
- 添加无关功能、无关重构或依赖升级。
- 伪造构建、测试、浏览器验证结果。

如本提示词与真实后端能力冲突，必须先报告具体冲突及代码证据，不得自行虚构后端能力。

---

## 三、先建立执行清单

修改代码前，在：

```text
.kscc-prompts/web-execution-ledger.md
```

建立或更新执行台账。

台账至少包含：

| 编号 | 要求 | 涉及文件 | 实现方式 | 验证方式 | 状态 | 证据 |
|---|---|---|---|---|---|---|

本次所有要求必须逐项登记。

状态只能使用：

- `待处理`
- `处理中`
- `已完成`
- `已验证`
- `阻塞`

只有代码完成且对应验证通过后，才能标记为 `已验证`。

删除或停止使用错误位置的：

```text
agent-web/LEDGER.md
```

最终台账必须位于 `.kscc-prompts/web-execution-ledger.md`。确保 `.kscc-prompts/` 已被 `.gitignore` 忽略，不要提交本地提示词和台账。

---

# 四、核心产品交互模型修复

## 4.1 删除用户可见的 Agent 模式

检查并修改所有相关位置，包括但不限于：

- 路由
- Sidebar
- ChatView
- ChatComposer
- Chat Store
- Chat Transport
- Query hooks
- 类型定义
- 空状态
- 标题和提示语
- 本地线程索引
- 权限导航
- E2E 测试

普通用户界面必须满足：

- 不显示 Agent 列表。
- 不显示 Agent 名称。
- 不提供 Agent/Supervisor 模式切换。
- 不出现“选择 Agent”。
- 不出现“Agent 模式”。
- 不允许用户构造 Agent 调用请求。
- 不调用 Agent 列表接口来生成用户界面。
- 不调用 Agent invoke 接口发送用户消息。
- 用户消息只能发送给 Supervisor。
- Agent 调度信息默认不能暴露给用户。
- 不得通过 Supervisor 返回的内部 metadata 向用户泄漏成员 Agent 名称、内部任务拆分、Prompt、权限或实现细节。

完成后执行全局搜索，确认生产源码中不存在用户可达的 Agent 选择和调用链。

允许保留后端测试 Agent 接口，但前端生产调用链必须是：

```text
ChatView
  → Chat Store
  → ChatTransport
  → Supervisor API
  → HTTP Client
  → 后端 Supervisor Controller
```

不得存在：

```text
用户
  → Agent selector
  → Agent invoke API
```

如果前端原有 Agent API 文件仅被测试或不再使用：

- 优先删除无用的生产前端调用代码。
- 不要保留误导性的死代码。
- 不要删除后端接口。
- 不要创建 `V2`、`NewChatStore`、`SupervisorOnlyStore` 等重复实现。

## 4.2 登录后的默认行为

登录成功后，不得显示：

```text
请选择 Agent 或 Supervisor
```

应按以下规则实现：

### 只有一个可用 Supervisor

- 自动选中该 Supervisor。
- 自动进入聊天工作区。
- 显示正常的欢迎空状态和输入框。
- 不要求用户额外选择。
- 不自动发送假消息。
- 不伪造 Supervisor 回复。

### 有多个可用 Supervisor

- 可以展示 Supervisor 选择器。
- 选择器只能展示 Supervisor，不能展示 Agent。
- 说明文字应面向业务能力，不暴露内部 Agent 编排。
- 用户选择后进入对应 Supervisor 会话。

### 没有可用 Supervisor

显示明确且安全的空状态，例如：

```text
当前没有可用的智能协作服务，请联系管理员。
```

不得：

- 回退到 Agent 模式。
- 显示 Agent 列表。
- 使用固定 Supervisor 名称。
- 创建假的 Supervisor。
- 允许在未确定 Supervisor 时发送消息。

### Supervisor 列表加载失败

- 展示明确的加载失败状态。
- 提供重试按钮。
- 不把加载失败显示成“没有 Supervisor”。
- 不自动改用 Agent API。

## 4.3 会话模型

前端线程记录不应再保存用户可切换的 `agent/supervisor` 模式。

根据真实代码做最小修改，使线程至少只表达：

- `threadId`
- Supervisor 标识
- 标题
- 创建时间
- 最后消息时间

如果浏览器中可能存在旧版本线程数据：

- 使用 Zod 对旧数据做兼容读取或安全丢弃。
- 不得未经校验直接恢复。
- 不得因为旧的 `mode: "agent"` 数据导致页面崩溃。
- 旧 Agent 会话不能继续调用 Agent 接口。
- 必要时安全清除不再兼容的旧会话索引。

---

# 五、Markdown 渲染修复

当前实现把普通正文一次性渲染，然后把所有代码块追加到末尾，破坏 Markdown 原始顺序。

例如：

````markdown
第一段

```ts
const a = 1
```

第二段

```java
class Demo {}
```

第三段
````

必须严格按以下顺序显示：

```text
第一段
TypeScript 代码块
第二段
Java 代码块
第三段
```

实现要求：

- 根据 markdown-it token 生成有序渲染片段。
- 片段类型至少区分安全 HTML 和代码块。
- Vue 模板按片段原始顺序渲染。
- 普通 HTML 必须经过 DOMPurify。
- Markdown 原始 HTML 保持禁用。
- 外部链接必须添加 `target="_blank"` 和 `rel="noopener noreferrer"`。
- 代码内容不得通过不安全的 HTML 字符串注入。
- 不使用正则表达式从完整 HTML 中提取代码块。
- 删除无用的 `inlineHtml`、未使用变量和占位符实现。
- 多个代码块、无语言代码块和未知语言代码块都必须正确显示。
- 增加针对“正文—代码—正文—代码—正文”顺序的组件测试。
- 增加 XSS 安全测试，确认事件属性和危险标签不会进入 DOM。

---

# 六、取消请求竞态修复

当前 Chat Store 中，旧请求的 `catch/finally` 可能在新请求开始后：

- 把新请求状态重置为 `idle`。
- 清空新请求的 AbortController。
- 导致新请求无法取消。
- 用旧响应覆盖当前会话状态。

必须修复为请求级隔离。

建议方式：

- 每次发送生成唯一 `requestId`，或捕获本次局部 `controller`。
- 只有当前活动请求才能修改全局 `runState`。
- 只有在 `activeController === currentRequestController` 时才能清理 Controller。
- 旧请求取消后的 `catch/finally` 不得影响新请求。
- 切换会话和新建会话后，旧请求返回不得写入新会话消息。
- 取消只表达“前端停止等待”，不得宣称后端 Agent/Supervisor 已停止执行。
- 不保留无实际行为的 `ChatTransport.abort()`。
- 如果 Controller 由 Store 持有，就从接口中删除无意义的 `abort()`。
- 保持唯一取消链路：

```text
Chat Store AbortController
  → Transport signal
  → API signal
  → fetch signal
```

必须新增测试：

1. 发送请求 A。
2. 取消 A。
3. 立即发送请求 B。
4. A 随后抛出 AbortError。
5. B 必须仍处于运行状态。
6. B 的 Controller 必须仍可取消。
7. A 不得覆盖 B 的状态和消息。

还要测试：

- 新建会话后旧请求完成。
- 切换线程后旧请求完成。
- 网络错误与主动取消的状态不同。
- SUSPENDED 优先于普通失败判断。

---

# 七、401、登出及用户缓存隔离

必须显式创建并持有唯一的 Vue Query `QueryClient`，将其传给 `VueQueryPlugin`。

集中 401 处理必须按以下顺序执行：

1. 取消当前用户的进行中查询。
2. 清空 Vue Query 缓存。
3. 清空当前聊天请求和用户相关 Pinia 状态。
4. 清除 Session。
5. 清空当前用户。
6. 跳转登录页，并保留安全的 redirect。
7. 登录接口自身返回 401 时，只展示登录失败，不触发“Session 过期”跳转。

主动登出同样必须：

- 即使后端登出请求失败，也不能把旧用户数据留给下一位登录用户。
- 清理 Query Cache。
- 清理聊天消息、线程索引和运行状态。
- 清理用户相关 Store。
- 清理 Session。
- 跳转登录页。
- 不记录或输出 sessionId。

登录为另一个用户时，不得看到上一用户的：

- 审批列表
- 用户列表
- 角色列表
- Supervisor 查询结果中带用户权限差异的数据
- 聊天消息
- 会话索引
- 错误状态

增加测试覆盖：

- 普通 401 清理 Session、用户状态和 Query Cache。
- 登录接口 401 不触发过期跳转。
- 403 不清 Session。
- 主动登出清理全部用户数据。
- 用户 A 登出后用户 B 不复用用户 A 的缓存。

---

# 八、错误响应 Zod 校验

已经存在 `apiErrorResponseSchema`，必须真实使用。

禁止：

```ts
const errorData = data as ApiErrorResponse
```

必须：

- 对非 2xx JSON 响应使用 `apiErrorResponseSchema.safeParse`。
- 校验成功后转换为结构化 `ApiError`。
- 校验失败时使用通用、安全的错误信息。
- 不把后端原始响应、内部类名、堆栈或敏感数据展示给用户。
- 非 JSON 错误响应也必须安全处理。
- 401 的集中行为保持独立。
- AbortError 不得转换为网络错误。
- 网络断开继续使用独立错误类型或明确状态。

增加测试：

- 合法错误响应。
- 缺少 `errorCode`。
- 缺少 `message`。
- 非 JSON 错误响应。
- 401、403。
- AbortError。
- 普通网络错误。

同时检查所有 Zod Schema 与后端真实 DTO 一致。

特别检查：

- `AgentInfoResponse.contextManagement` 不得只使用宽泛的 `z.record(z.unknown())`，应匹配真实嵌套字段。
- 已有枚举字段应使用对应 enum schema。
- 尽量从 Zod Schema 使用 `z.infer` 导出响应类型，避免 Schema 与手写 interface 双份漂移。

---

# 九、E2E 测试修复

当前测试中存在没有实际验证能力的断言，必须修复。

禁止以下类型的宽松断言：

```ts
expect(stillOnLogin || hasError).toBeTruthy()
expect(isDenied || isUsers).toBeTruthy()
expect(typeof isDark).toBe('boolean')
```

这些断言无法证明功能正确。

## 9.1 登录测试

Enter 提交必须验证至少一个真实结果：

- 登录请求确实被发送。
- 合法凭据进入预期路由。
- 非法凭据显示真实错误。
- 按钮点击和 Enter 使用同一提交逻辑。

## 9.2 路由权限测试

必须使用权限明确的真实测试账户，并确定断言：

- 无权限用户访问管理页，必须进入权限不足页。
- 有权限用户访问管理页，必须停留在管理页。
- 不得用两个互相矛盾的结果做 OR 断言。

## 9.3 主题测试

必须验证：

- 切换前的确定主题状态。
- 点击后 `html` 的主题 class 或属性发生确定变化。
- 页面背景色、前景色等关键 CSS 变量随主题变化。
- 再次切换后恢复预期状态。

## 9.4 用户只与 Supervisor 交互

增加真实 E2E：

- 登录后不显示 Agent 选择器。
- 页面中不存在“请选择 Agent 或 Supervisor”。
- 页面中不展示 Agent 列表。
- 单 Supervisor 时自动进入聊天工作区。
- 多 Supervisor 时只展示 Supervisor 选择。
- 用户发送消息时只请求 Supervisor invoke 接口。
- 前端不得请求 Agent invoke 接口。
- 无 Supervisor 时显示正确空状态。
- Supervisor 加载失败时显示重试状态。
- 桌面和移动端都验证。

如没有真实后端或测试凭据：

- 可以明确跳过依赖真实后端的用例。
- 必须在最终结果中列出跳过数量和准确原因。
- 不得把跳过写成通过。
- 不得使用 Mock Supervisor 冒充真实完整 E2E。
- 单元级 UI 测试可以 Mock 网络边界，但必须明确它不是完整 E2E。

Playwright 浏览器依赖必须安装完成后再执行验证。若环境或网络阻止安装，必须报告阻塞原因，不得写“E2E 已通过”。

---

# 十、Shiki 构建优化

当前使用 `shiki/bundle/web`，生产构建生成大量不需要的语言和主题资源，并出现大分块警告。

必须检查当前实际 Shiki 版本提供的 API，然后实现真正受控加载：

- 只打包产品实际支持的语言。
- 只加载实际使用的主题。
- 不生成全部 Shiki 语言和主题分块。
- 未知语言回退为转义后的纯文本代码块。
- 不使用不安全类型断言绕过任意语言加载。
- 删除未使用变量。
- Shiki 代码必须延迟加载，不能阻塞登录页和普通首屏。
- Chat 页面可以异步加载高亮能力。
- 复制功能在高亮失败时仍然可用。

最低支持语言建议以项目真实需要为准，例如：

- Java
- TypeScript
- JavaScript
- JSON
- Bash
- YAML
- SQL
- Markdown
- Plain text

不得凭空扩大语言列表。

构建后记录：

- 主入口 JS 大小。
- Chat 页面分块大小。
- Shiki 相关分块。
- 是否仍有超过 500 kB 的警告。
- 优化前后差异。

不能简单提高 `chunkSizeWarningLimit` 来隐藏警告。

---

# 十一、UI 和无障碍要求

保持现有 ChatGPT 风格设计，但纠正产品语义。

登录后的空聊天状态应表达：

- 用户正在与智能协作服务或 Supervisor 交流。
- 用户可以直接描述目标。
- 系统会在内部完成任务规划和执行。

不得向用户展示内部实现细节，例如：

- Agent 成员列表
- Agent 调度名称
- 内部 Prompt
- 工具权限集合
- RunContext
- 内部节点名称
- Java 类名
- 完整 metadata

移动端侧边栏和 Dialog 继续使用 Reka UI，必须保留：

- Esc 关闭
- 遮罩
- 焦点锁定
- 可访问标题
- 键盘操作
- 明确的 `aria-label`

图标继续统一使用当前已经选定的 Lucide Vue 包，不得混入 emoji 或第二套图标库。

---

# 十二、验证顺序

完成全部修改后，严格按仓库 `AGENTS.md` 要求执行真实验证。

至少执行：

```bash
npm run typecheck
npm test -- --run
npm run build
npm run test:e2e
```

不要在尚未完成修改时反复运行最终构建；完成全部修改后再执行最终验证。

还必须进行真实浏览器检查：

## 桌面端

- 登录页
- 登录成功后的默认页面
- 单 Supervisor 自动进入
- 多 Supervisor 选择
- 无 Supervisor
- Supervisor 加载失败和重试
- 发送消息
- 取消后立即再次发送
- Markdown 多代码块顺序
- 代码复制
- 401
- 403
- 主动登出
- 深色主题
- 审批 Dialog
- 管理页面权限

## 移动端

- 无横向溢出
- 输入框可用
- 键盘操作正常
- Sheet 打开和关闭
- Esc 关闭
- 遮罩关闭
- Supervisor 选择
- 聊天消息不被遮挡
- 代码块可横向滚动

真实浏览器验证时检查：

- 页面控制台错误
- 网络失败
- 未处理 Promise rejection
- Vue warning
- 无效 DOM 嵌套
- 横向溢出
- 请求是否错误地调用 Agent invoke API

---

# 十三、完成标准

只有同时满足以下条件才能声明任务完成：

- 用户界面完全移除 Agent 选择和 Agent 直接调用。
- 登录后不再显示“请选择 Agent 或 Supervisor”。
- 用户只能与 Supervisor 交互。
- 单 Supervisor 自动进入聊天。
- 多 Supervisor 只展示 Supervisor 选择。
- 无 Supervisor 和加载失败有不同状态。
- 生产前端没有 Agent invoke 调用链。
- Markdown 代码块保持原始顺序。
- 取消后立即发送不存在竞态。
- 401 和登出会清理全部用户缓存及状态。
- 错误响应经过 Zod 校验。
- E2E 断言具有确定验证能力。
- Shiki 不再输出大量无关语言和主题资源。
- 类型检查通过。
- 单元测试通过。
- 生产构建通过。
- Playwright 实际执行；失败或跳过必须如实报告。
- 执行台账位于正确路径并逐项包含代码和验证证据。

---

# 十四、最终输出格式

最终回复必须严格包含以下章节：

```markdown
## 1. 修改文件
逐个列出新增、修改和删除的文件。

## 2. 用户交互模型
说明用户如何进入 Supervisor 会话，以及 Agent 为什么不再对用户可见。

## 3. 核心调用链
列出从页面到 Supervisor 后端接口的真实调用链。

## 4. 各问题修复证据
按本任务编号逐项说明代码位置和实现方式。

## 5. 测试结果
列出每条实际执行的命令、退出码和结果。

## 6. 浏览器验证
分别列出桌面端和移动端真实验证结果。

## 7. 构建产物
列出主入口、Chat 分块、Shiki 分块及大分块警告情况。

## 8. 未完成或跳过项
准确说明所有未验证内容及原因。

## 9. 非本批问题
仅报告发现但未处理的非阻断问题。

## 10. 执行台账
确认 `.kscc-prompts/web-execution-ledger.md` 已逐项更新。
```

禁止：

- 把计划写成完成结果。
- 把跳过写成通过。
- 只说“所有验证通过”而不提供命令和结果。
- 使用与真实类名、文件名或接口不符的名称。
- 忽略本提示词中的任何一项。