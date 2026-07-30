# 修复 Agent Web 前端审计问题

## 一、任务目标

修复当前 `agent-web` 实现中已经确认的功能、类型、架构和验收偏差。

本次不是重新设计，也不是增加新功能。只修复下文列出的明确问题，并完成真实验证。

必须遵守：

1. 根目录 `AGENTS.md`
2. `.kscc-prompts/web.md`
3. 后端真实 Controller、DTO 和枚举
4. 本提示词中的修复要求

禁止：

- 修改 `web.md`
- 修改 `AGENTS.md`
- 修改无关后端代码
- 创建 Fake、Mock、固定回答或假流式输出
- 调用 `/api/dev/**`
- 用类型断言掩盖接口不一致
- 为通过测试删除或弱化测试
- 执行 Git commit、push 或远程操作
- 把计划描述成修复结果

---

## 二、开始前必须检查

修改前完整读取：

```text
AGENTS.md
.kscc-prompts/web.md
agent-web/package.json
agent-web/tsconfig.json
agent-web/tsconfig.app.json
agent-web/src/types/index.ts
agent-web/src/api/
agent-web/src/chat/
agent-web/src/stores/
agent-web/src/queries/
agent-web/src/components/chat/
agent-web/src/components/layout/
agent-web/src/components/sidebar/
agent-web/src/components/ui/
agent-web/src/views/
```

同时读取并核对真实后端定义：

```text
agent-api/src/main/java/com/ksyun/agent/api/controller/
agent-api/src/main/java/com/ksyun/agent/api/dto/
agent-core/src/main/java/com/ksyun/agent/core/run/RunStatus.java
agent-core/src/main/java/com/ksyun/agent/core/approval/ApprovalStatus.java
agent-core/src/main/java/com/ksyun/agent/core/tool/ToolRiskLevel.java
```

以真实代码字段为准，不得根据当前前端类型反推后端协议。

---

## 三、建立本轮修复 Ledger

开始修改前创建或更新：

```text
.kscc-prompts/web-execution-ledger.md
```

为本次修复建立以下固定 ID：

| ID | 修复内容 |
|---|---|
| FIX-001 | 取消请求真实接入 AbortSignal |
| FIX-002 | ChatTransport 接入 Chat Store |
| FIX-003 | Zod 接入真实 API 响应边界 |
| FIX-004 | Markdown 代码块真实使用 CodeBlock |
| FIX-005 | Shiki 类型安全和复制功能 |
| FIX-006 | 审批 QueryClient 生命周期修复 |
| FIX-007 | 401 集中清理认证状态并跳转登录 |
| FIX-008 | 前后端枚举严格一致 |
| FIX-009 | 消除 `any` 和不安全双重断言 |
| FIX-010 | Reka UI/shadcn-vue 基础组件真实使用 |
| FIX-011 | Lucide 图标真实使用或移除依赖 |
| FIX-012 | 不可达组件和真实数据链路说明 |
| FIX-013 | Playwright 配置与关键流程 |
| FIX-014 | 桌面、移动端、深色主题实际验证 |
| FIX-015 | 最终类型检查、测试和生产构建 |

Ledger 每项必须记录：

- 状态
- 修改文件
- 实现符号
- 验证方式
- 验证结果
- 是否仍有偏差

没有代码证据和验证结果，不得标记为完成。

---

## 四、FIX-001：修复取消请求

### 当前问题

`chat.ts` 创建了 `AbortController`，但没有把 `signal` 传给：

- `invokeAgent`
- `invokeSupervisor`
- API Client

`jsonTransport.ts` 也创建了另一个 `AbortController`，但同样没有传递 `signal`。

因此点击“取消请求”目前只改变前端状态，不能取消真实 `fetch`。

### 修复要求

建立唯一、清晰的取消链路：

```text
ChatComposer
→ Chat Store
→ ChatTransport
→ agents.ts/supervisors.ts
→ API Client
→ fetch(signal)
```

要求：

1. Chat Store 不得再直接调用 `invokeAgent` 和 `invokeSupervisor`。
2. Chat Store 必须通过 `ChatTransport` 调用后端。
3. `JsonTransport` 创建并持有当前请求的 `AbortController`。
4. `JsonTransport` 必须把 `controller.signal` 传入 API 层。
5. `agents.ts` 和 `supervisors.ts` 必须接收可选 `AbortSignal`。
6. API Client 最终必须把 `signal` 传给 `fetch`。
7. `cancelRequest()` 必须调用当前 Transport 的 `abort()`。
8. 不得同时存在两套互不关联的 AbortController。
9. 请求结束后必须清理 Controller。
10. 新请求开始前不得残留旧 Controller。

### AbortError 处理

API Client 不得把 `AbortError` 转换成普通“网络连接失败”。

必须明确区分：

- 用户主动取消
- 网络断开
- 后端 HTTP 错误
- 未知前端错误

取消后：

- 不显示“网络连接失败”
- 不宣称后端 Agent 已停止
- 可以显示“已取消等待”
- 助手占位消息不能永久停留在“思考中”
- 后续响应不得覆盖已经取消的前端会话状态
- Store 必须回到明确状态

如果后端不支持服务器端取消，文案必须保持为“取消等待”或“已停止等待响应”。

### 测试

新增或修复测试，至少覆盖：

- `signal` 被传入 API Client
- `abort()` 会终止当前 fetch
- AbortError 不被映射为网络错误
- 取消后占位消息状态正确
- 取消后不会错误进入 completed/failed

---

## 五、FIX-002：ChatTransport 必须真实接入

当前 `ChatTransport` 和 `jsonTransport` 属于未接入死代码。

修复后必须满足：

```text
Chat Store → ChatTransport → API
```

禁止：

```text
Chat Store → 直接 import agents.ts/supervisors.ts
```

要求：

- Transport 只负责传输和取消
- Store 负责消息与运行状态
- API 模块负责 HTTP DTO 映射
- 不创建 `ChatTransportV2`
- 不创建重复 Transport
- 当前只保留真实可用的 JSON Transport
- 不创建 FakeSseTransport 或未使用的 SseTransport

---

## 六、FIX-003：Zod 接入真实 API 边界

### 当前问题

`schemas.ts` 只定义了 Schema，并且只在测试中使用。正式 API 调用通过：

```ts
return data as T
```

强制断言，没有运行时校验。

### 修复要求

普通接口响应必须在进入 Store、Query 或组件前完成 Zod 校验。

至少覆盖：

- 登录响应
- 当前用户响应
- Agent 调用响应
- Supervisor 调用响应
- 待审批列表响应
- 待审批详情响应
- 审批恢复响应
- Agent 列表
- Supervisor 列表
- Tool 列表
- 用户列表
- 角色列表
- 后端错误结构

可以选择以下一种架构：

### 方案 A：API 模块解析

```text
client.request() 返回 unknown
→ agents.ts 使用 agentInvokeResponseSchema.parse()
→ 返回经过验证的数据
```

### 方案 B：Client 接收 Schema

```text
client.request(url, schema)
→ schema.parse(response)
→ 返回 z.infer 类型
```

选择一种即可，不得同时实现两套重复机制。

要求：

1. HTTP Client 不得继续使用无校验的 `data as T`。
2. 对外部 JSON 使用 `unknown`。
3. Zod 校验失败必须转换成结构化前端错误。
4. 错误信息不得泄漏完整响应或敏感数据。
5. 静态类型优先从 `z.infer` 推导，避免 Schema 和 interface 双份漂移。
6. 如果保留 interface，必须确保测试验证两者一致。
7. 不得使用 `as unknown as Xxx` 绕过。

### 测试

至少覆盖：

- 合法响应通过
- 缺少必填字段失败
- 枚举值错误失败
- metadata 保持 `Record<string, unknown>`
- 错误响应不泄漏原始数据
- API 模块确实调用 Schema，而不是只测试 Schema 文件

---

## 七、FIX-004：Markdown 代码块必须真实使用 CodeBlock

### 当前问题

`MarkdownContent.vue` 导入了 `CodeBlock.vue`，但：

- 模板没有使用
- markdown-it 的 fence renderer 没有接入
- Shiki 不会执行
- 复制按钮不会显示

### 修复要求

Markdown 中的围栏代码块必须真实渲染为 `CodeBlock.vue`。

要求：

- 普通 Markdown 继续使用 markdown-it
- 原始 HTML 保持禁用
- 最终 HTML 继续经过 DOMPurify
- 代码块由 Vue 组件接管
- 必须显示语言标签
- 必须显示复制按钮
- 未识别语言回退纯文本
- 代码内容必须安全转义
- 不允许通过字符串拼接注入事件处理器
- 不允许直接把不可信模型输出交给 `v-html`
- 不引入第二套 Markdown 解析器

可以采用：

- markdown-it token 分析后由 Vue 分段渲染
- 自定义安全 token renderer
- 其他能够真实使用 `CodeBlock.vue` 的单一方案

但最终必须证明 `CodeBlock.vue` 在真实 Markdown 代码围栏中被挂载，而不是仅在测试中直接挂载。

### 测试

至少覆盖：

- 普通段落
- Markdown 代码围栏
- 指定语言代码围栏
- 未知语言回退
- HTML/脚本注入被清理
- 代码复制按钮存在
- 代码内容不会执行

---

## 八、FIX-005：Shiki 类型安全

当前存在：

```ts
lang: lang.value as any
```

必须删除。

要求：

- 使用 Shiki 提供的明确类型
- 对语言名称做白名单或兼容映射
- 未支持语言回退为 `text`
- 不得用 `any`
- 不得用双重断言
- Shiki 只在真实代码块存在时加载
- 不要导入完整语言集合到首屏
- 完成后检查构建产物，避免所有 Shiki 语言进入主包

复制状态可以使用短时定时器，但必须在组件卸载时清理 Timer，避免卸载后更新状态。

复制失败不得静默伪装成功，应提供可访问但克制的失败反馈。

---

## 九、FIX-006：审批 QueryClient 生命周期

### 当前问题

`invalidateApprovals()` 在事件处理期间调用 `useQueryClient()`，可能脱离 Vue setup 注入上下文。

### 修复要求

必须在组件 setup 阶段获取 QueryClient。

推荐：

```ts
const queryClient = useQueryClient()

await decideAndResume(...)

await queryClient.invalidateQueries({
  queryKey: ['hitl', 'approvals']
})
```

或让一个在 setup 中调用的 Composable 返回 mutation。

要求：

- 删除全局 `invalidateApprovals()` 中的 `useQueryClient()`
- 删除未使用的 QueryClient 变量
- 审批通过和拒绝后都刷新列表
- 失败时保留当前审批项
- 不得因一个审批项操作影响其他项的输入状态

同时修复审批页当前所有列表项共用一个 `comment` 的问题：

- 每个审批项应有独立拒绝原因
- 或打开独立 Dialog 输入拒绝原因
- 不得在操作 A 时把操作 B 的拒绝原因带过去

优先使用 Reka UI/shadcn-vue Dialog 或 AlertDialog。

---

## 十、FIX-007：401 集中认证失效处理

### 当前问题

API Client 收到 401 后只清除 Session Storage，没有：

- 清空 Pinia `currentUser`
- 把认证状态设为未登录
- 跳转 `/login`
- 保留合理 redirect
- 告知登录已过期

### 修复要求

建立集中且无循环依赖的认证失效机制。

可以选择：

- 在 API Client 注册 `onUnauthorized` 回调
- 集中 Session 服务发布认证失效事件
- 其他单一、清晰的机制

要求：

1. API Client 不直接 import Vue 组件。
2. 避免 API Client、Store、Router 循环依赖。
3. 应用启动时注册一次 401 处理器。
4. 401 时：
    - 清除 Session
    - 清空认证 Store
    - 清理与用户相关的 Query Cache
    - 跳转 `/login`
    - 添加 `expired=1` 或安全 redirect
5. 登录接口自身的 401 不得重复触发过期跳转逻辑。
6. 403 不得清除 Session。
7. 403 继续表示“已认证但无权限”。
8. 不得把 sessionId 放进 URL 或 Body。

### 测试

至少覆盖：

- 正式接口 401 清理 Session
- Pinia 认证状态被清空
- 跳转登录页
- 登录接口 401 不进入过期循环
- 403 不清理认证状态

---

## 十一、FIX-008：严格修正枚举

必须根据真实 Java 枚举修正前端。

当前真实值为：

```text
RunStatus:
CREATED
RUNNING
INTERRUPTED
SUSPENDED
COMPLETED
FAILED
```

```text
ApprovalStatus:
PENDING
APPROVED
REJECTED
```

```text
ToolRiskLevel:
SAFE
LOW
MEDIUM
HIGH
```

前端必须：

- 删除不存在的 `EXPIRED`
- 删除不存在的 `CRITICAL`
- 增加真实存在的 `SAFE`
- 同步修正 Zod Schema
- 同步修正风险等级显示
- 同步修正相关测试
- 搜索整个前端，确保不存在旧值

不得为了“未来可能支持”预先添加后端不存在的枚举值。

---

## 十二、FIX-009：消除不安全类型断言

搜索并处理：

```text
any
as any
as unknown as
Record<string, unknown> 强制写入具体类型
```

重点修复：

- `ChatMessage.vue` 工具结果分支中的 `(message as any)`
- `CodeBlock.vue` 的语言 `as any`
- Chat Store 中通过 `Record<string, unknown>` 写入 status
- API Client 的 `data as T`

要求：

- 使用可辨识联合类型自动缩窄
- 为工具结果建立类型安全 computed
- 对 `'status' in response` 后的类型使用明确 type guard
- 不得通过重新命名 `UnsafeAny` 等方式逃避检查
- 第三方类型确实无法表达时，必须使用最小范围 `unknown` 并校验

最终静态搜索不得发现业务代码中的显式 `any`。

---

## 十三、FIX-010：真实使用 Reka UI 与 shadcn-vue 基础组件

### 当前问题

虽然安装了 `reka-ui`，但源码没有实际 import。`components/ui` 下组件也没有被业务页面使用。

### 修复要求

不得只保留依赖并在报告中宣称已使用。

至少真实接入：

- 移动端侧边栏：Sheet/Dialog 语义
- 用户/角色编辑：Dialog
- 密码重置或拒绝审批：Dialog/AlertDialog
- 用户菜单：Dropdown Menu
- 图标按钮：Tooltip
- 页面操作按钮：统一 Button
- 输入区域：统一 Textarea 或明确的产品组件封装

要求：

- 优先使用已有 `components/ui`
- 必要时按 shadcn-vue 模式增加源码组件
- 底层使用 Reka UI
- 不得再手写缺少焦点管理的 `fixed inset-0` 假 Dialog
- Dialog 支持 Esc
- Dialog 支持焦点锁定
- Dialog 有正确标题和描述
- 移动端 Sheet 有遮罩、关闭按钮和无障碍语义
- 图标按钮必须有 accessible name

修复后必须能够搜索到业务代码真实 import：

```text
reka-ui
@/components/ui/
```

未使用的 UI 组件应删除，不得保留死代码。

---

## 十四、FIX-011：Lucide 图标

当前声明使用 Lucide，但源码没有实际引用，界面使用了字符：

```text
☰
+
↑
■
↓
```

修复要求：

- 使用当前实际安装且可用的 Lucide Vue 包
- 菜单、新建、发送、取消、向下滚动、主题、退出等使用一致图标
- 图标按钮保留文字替代或 `aria-label`
- 如果确认当前包已弃用，应使用官方当前包并更新依赖
- 不得同时保留两个 Lucide 包
- 如果最终决定不使用 Lucide，则删除依赖并在报告中明确，不得继续宣称使用

---

## 十五、FIX-012：不可达组件与真实能力

检查：

- `ToolCallCard`
- `SupervisorDispatchCard`
- `CodeBlock`
- `ApprovalCard`
- 所有 ChatMessage 联合类型

要求：

1. `CodeBlock` 必须按前文真实接入。
2. `ApprovalCard` 必须由真实 `SUSPENDED` 响应触发。
3. `ToolCallCard` 只能在后端真实返回安全工具信息时显示。
4. `SupervisorDispatchCard` 只能在后端真实返回安全分派信息时显示。
5. 不得从模型正文推测 ToolCall 或 Supervisor 分派。
6. 不得构造假工具轨迹。
7. 后端当前没有提供相应数据时：
    - 保留组件需要有明确的真实扩展边界和测试
    - 或删除不可达组件
    - 最终报告必须说明“尚无真实数据链路”
8. 不得再次宣称未接入组件已经完成真实调用链。

### HITL 状态判断

必须先根据真实 `RunStatus` 判断 `SUSPENDED`，不得仅根据 `success` 决定完成。

正确优先级应类似：

```text
status == SUSPENDED
→ 显示挂起和审批

否则 success == true
→ completed

否则
→ failed
```

必须验证后端 `DefaultReactSuspendNode` 返回：

```text
success=false
status=SUSPENDED
```

并按真实语义处理。

---

## 十六、FIX-013：Playwright

`web.md` 已明确授权新增必要前端测试。

增加：

- `@playwright/test`
- `playwright.config.ts`
- `test:e2e` 脚本
- 独立 E2E 目录

至少实现：

1. 登录页渲染
2. Enter 提交行为
3. 未登录访问正式页面跳转登录
4. 权限不足页面路由
5. 桌面布局基础检查
6. 移动端菜单打开与 Esc 关闭
7. 深色主题切换

真实 Agent/Supervisor 流程：

- 只能连接真实后端
- 凭据从环境变量读取
- 不得硬编码示例密码
- 后端或凭据不存在时明确跳过，并报告原因
- 不得用 Mock Agent 冒充真实 E2E

不要提交 Playwright 浏览器二进制。

---

## 十七、FIX-014：真实页面验证

生产构建成功不等于页面可用。

完成修改后，必须启动本地前端并实际检查：

### 桌面端

- 登录卡片居中
- 登录页样式生效
- 侧边栏宽度正常
- 聊天内容最大宽度正常
- 输入框固定在底部
- 页面无横向溢出

### 移动端

至少检查：

```text
375 × 812
```

验证：

- 侧边栏默认隐藏
- Sheet 可以打开
- Esc 可以关闭
- 焦点不会落到遮罩后
- 输入框不溢出
- 代码块自身滚动
- 页面无横向滚动

### 深色主题

验证：

- 背景
- 文本
- 边框
- 输入框
- Dialog
- Tool/Approval 卡片
- Focus Ring

检查浏览器控制台，不能存在运行时错误和 Vue 警告。

不得只通过源码推断页面正常。

---

## 十八、测试补充

除前述专项测试外，至少保留并修复：

- API 错误映射
- Zod 边界校验
- 自动滚动判断
- Auth Store
- Chat Store 状态转换
- ChatComposer Enter/Shift+Enter
- Approval 状态
- ToolCall 类型安全渲染
- 401/403 行为
- Abort 行为

测试必须验证正式实现，而不是复制一份测试专用逻辑。

---

## 十九、实施顺序

严格按以下顺序：

1. 读取真实代码和 Ledger
2. 修正前后端枚举
3. 修正 API Client 和 Zod 边界
4. 修正 ChatTransport 与 AbortSignal
5. 修正 Chat Store 状态转换
6. 修正 Markdown 和 CodeBlock
7. 修正审批 QueryClient
8. 修正 401 集中处理
9. 消除 `any` 和不安全断言
10. 接入 Reka UI/shadcn-vue
11. 接入 Lucide
12. 处理不可达组件
13. 增加 Playwright 配置
14. 补充测试
15. 实际页面检查
16. 完成后统一执行最终验证
17. 更新 Ledger
18. 输出最终结果

禁止提前执行最终构建。全部代码修改完成后再统一验证。

---

## 二十、最终验证

所有修改完成后执行一次完整验证：

```bash
npm run typecheck
npm run test -- --run
npm run build
```

如果 Playwright 环境可用：

```bash
npm run test:e2e
```

同时执行静态搜索：

```text
/api/dev/
as any
as unknown as
data as T
Fake
Mock
CRITICAL
EXPIRED
reka-ui
lucide
CodeBlock
jsonTransport
AbortController
safeParse
.parse(
```

要求：

- `/api/dev/` 不得出现在正式 API 调用
- 不得存在业务 `as any`
- 不得存在双重断言
- 不得存在 `data as T`
- 不得存在 Fake 或固定回答
- 不得存在错误枚举
- `reka-ui` 必须有真实业务引用
- Lucide 必须有真实业务引用，或者依赖已删除
- `CodeBlock` 必须有真实调用
- `jsonTransport` 必须由 Chat Store 使用
- AbortSignal 必须最终进入 `fetch`
- Zod Schema 必须由正式 API 调用

验证失败时继续修复，不得把失败描述为通过。

---

## 二十一、最终输出格式

最终回复严格按以下顺序：

1. 总体结论
2. FIX-001 至 FIX-015 状态表
3. 新增和修改文件
4. ChatTransport 与 AbortSignal 完整调用链
5. Zod 正式接入位置
6. Markdown 到 CodeBlock 的真实渲染链
7. 401 集中处理链
8. 审批刷新修复
9. 枚举一致性结果
10. `any` 和不安全断言搜索结果
11. Reka UI/shadcn-vue 实际使用位置
12. Lucide 实际使用位置
13. 不可达组件处理结果
14. Playwright 配置和执行结果
15. 桌面端、移动端和深色主题验证
16. 类型检查结果
17. 单元测试结果
18. 生产构建结果
19. 未完成验证及准确原因
20. 发现但未处理的非本次问题
21. Ledger 路径

任何 FIX 仍为失败时，不得宣称“全部修复完成”。