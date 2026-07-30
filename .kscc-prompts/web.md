# Agent Platform 前端重构任务

## 一、任务目标

重新设计并实现 `agent-web` 前端，将当前测试性质的页面升级为一套可持续维护的 AI Agent 产品界面。

整体交互参考 ChatGPT，但不要复制 ChatGPT 的品牌、Logo、文案和像素级样式。目标是实现类似的产品结构和交互体验：

- 左侧会话与功能导航
- 中间沉浸式聊天区域
- 底部固定输入框
- Agent、Supervisor、工具执行、人工审批融入聊天流程
- 同时保留登录、权限、用户管理、角色管理等平台能力
- 支持桌面端和移动端
- 支持浅色与深色主题

本次用户明确改变了仓库原有前端约束：

> 允许且要求使用 TypeScript。原有“前端禁止 TypeScript”的约束对本次任务不再适用。

除 TypeScript 这一条外，必须继续遵守仓库根目录 `AGENTS.md` 中的其他架构、安全、构建和范围约束。

---

## 二、开始修改前必须检查

实现前先完整检查以下内容，以仓库真实代码为准，不得凭记忆猜测：

1. 根目录 `AGENTS.md`
2. `agent-web/package.json`
3. `agent-web/package-lock.json`
4. `agent-web/vite.config.js`
5. `agent-web/src/router/index.js`
6. `agent-web/src/api/` 下全部接口封装
7. `agent-web/src/stores/` 下全部状态管理
8. `agent-web/src/views/` 和 `components/` 下现有页面
9. 后端 `agent-api` 中所有 Controller 和 HTTP DTO
10. 登录、Session、Agent、Supervisor、HITL、用户和角色管理接口
11. 后端接口当前是否支持 SSE 或其他流式响应
12. 后端实际返回的错误结构、运行状态和 metadata/evidence 结构

必须先确认真实接口字段，再定义前端 TypeScript 类型。

禁止：

- 猜测后端不存在的接口
- 创建固定响应、Mock Agent、Fake 流式输出
- 绕过认证或权限
- 让浏览器直接调用模型
- 把 `sessionId` 放进 URL 或请求 Body
- 在前端保存密码、API Key、密码 Hash
- 为了界面演示伪造工具调用、审批状态或 Agent 执行结果

---

## 三、技术栈

使用以下技术栈：

### 核心技术

- Vue 3
- Vite
- TypeScript
- Vue Router
- Pinia
- TanStack Vue Query
- Tailwind CSS 4
- shadcn-vue
- Reka UI
- Lucide Vue Next
- Zod
- markdown-it
- DOMPurify
- Shiki，按需加载语言
- Vitest
- Vue Test Utils
- Playwright

继续使用现有 npm 和 `package-lock.json`，不要切换 pnpm、yarn 或 bun。

不要引入：

- React
- Next.js
- Nuxt
- Axios
- Element Plus
- Ant Design Vue
- Vuetify
- 另一套重复的状态管理框架
- 另一套完整 UI 组件库

### 组件体系

优先使用 shadcn-vue 提供的基础组件源码：

- Button
- Textarea
- Tooltip
- Dialog
- Alert Dialog
- Dropdown Menu
- Popover
- Sheet
- Scroll Area
- Avatar
- Command
- Skeleton
- Toast
- Tabs

shadcn-vue 只作为基础组件来源。以下核心产品组件必须根据本项目语义自行实现：

- ChatMessage
- ChatComposer
- MarkdownContent
- CodeBlock
- ToolCallCard
- ApprovalCard
- AgentSelector
- SupervisorDispatchCard
- RunStatusIndicator

不得为了简单而把整个聊天界面拼成传统后台表单。

---

## 四、TypeScript 改造要求

将 `agent-web` 完整改造为 TypeScript 项目：

- `main.js` 改为 `main.ts`
- `vite.config.js` 改为 `vite.config.ts`
- Router、Store、API、Composable、工具函数全部改为 `.ts`
- Vue 组件使用 `<script setup lang="ts">`
- 增加正确的 `tsconfig` 配置
- 启用严格类型检查
- 开启 `noUncheckedIndexedAccess`
- 增加 `vue-tsc --noEmit` 类型检查命令
- 禁止使用大面积 `any`
- 禁止使用 `as unknown as Xxx` 绕过类型检查
- 对无法确定的数据使用 `unknown`，经过校验后再使用
- 为消息、事件、运行状态使用可辨识联合类型

建议的核心类型：

```ts
type ChatRole = 'user' | 'assistant' | 'system'

type ChatMessage =
  | UserChatMessage
  | AssistantChatMessage
  | ToolCallChatMessage
  | ToolResultChatMessage
  | ApprovalChatMessage
  | SupervisorDispatchMessage
  | ErrorChatMessage

type RunState =
  | { status: 'idle' }
  | { status: 'submitting' }
  | { status: 'running'; runId?: string }
  | { status: 'suspended'; runId: string }
  | { status: 'completed'; runId: string }
  | { status: 'failed'; runId?: string; errorCode?: string; message: string }
```

类型名称和字段必须根据后端真实 DTO 调整，不得机械照抄示例。

---

## 五、API 与状态管理

### 1. API 层

所有 HTTP 调用集中在 `src/api/`，组件不得直接调用 `fetch`。

建议结构：

```text
src/api/
├── client.ts
├── auth.ts
├── framework.ts
├── agents.ts
├── supervisors.ts
├── approvals.ts
├── users.ts
├── roles.ts
├── context.ts
├── contracts/
└── errors.ts
```

要求：

- Session Header 注入集中处理
- 401、403 和网络错误集中处理
- 支持 AbortSignal
- 返回结构化错误，禁止到处抛裸字符串
- 不在日志中输出 Session、密码、完整消息或敏感参数
- 认证状态只能从现有正式认证流程恢复
- `/api/dev/**` 不得被正式前端调用

如果后端当前提供 OpenAPI 文档：

- 使用 `openapi-typescript` 生成接口类型
- 使用 `openapi-fetch` 调用普通 JSON 接口
- 生成文件放到明确的 `generated` 目录
- 生成文件不得手工编辑

如果后端当前没有 OpenAPI 文档：

- 不得为了前端任务擅自修改后端依赖
- 根据真实 Controller 和 DTO 编写前端 contracts
- 不安装无实际输入源的 OpenAPI 工具
- 在最终结果中说明后续可以接入类型自动生成

### 2. Pinia

Pinia 只管理客户端状态：

- 当前用户与权限
- 当前工作区模式
- 当前 Agent/Supervisor
- 当前会话和消息
- 当前运行状态
- 输入草稿
- 侧边栏展开状态
- 主题和用户界面偏好

不得把所有服务端查询结果长期复制到 Pinia。

### 3. TanStack Vue Query

用于管理服务端数据：

- Agent 列表
- Supervisor 列表
- Tool 列表
- 待审批列表
- 用户列表
- 角色列表
- 能力查询
- 缓存失效与重新获取

为不同数据设置合理的 `staleTime`，不要依赖默认的频繁自动重新请求。

### 4. Zod

TypeScript 只能提供编译期类型，外部数据仍需运行时校验。

Zod 只用于系统边界：

- 流式事件
- 动态 metadata
- URL 参数
- 从浏览器存储恢复的数据
- 人工审批 payload

不要在组件内部对已经类型安全的数据重复校验。

---

## 六、流式响应要求

先检查后端是否已经存在真实 SSE 或流式接口。

### 如果后端已经支持流式响应

使用：

- 原生 `fetch`
- `ReadableStream`
- `TextDecoder`
- `AbortController`
- Zod 事件校验

不得使用原生 `EventSource`，因为当前系统需要：

- POST Body
- `X-Session-Id` Header
- 请求取消能力

建议把流式解析封装为：

```text
src/chat/
├── chatTransport.ts
├── streamParser.ts
├── streamSchemas.ts
└── streamEvents.ts
```

流式协议必须以真实后端事件为准。

### 如果后端当前不支持流式响应

必须遵循：

- 继续调用现有正式 JSON 接口
- 请求期间显示真实的运行中状态
- 返回后一次性展示真实结果
- 不得用定时器逐字输出制造假流式效果
- 不得生成假的 ToolCall、推理步骤或审批事件
- 可以抽象 `ChatTransport` 接口，但只实现当前真实可用的 Transport
- 不要创建未使用的 FakeSseTransport
- 在最终结果中明确说明：实现了流式扩展边界，但当前后端接口仍是非流式

AbortController 只能表达“前端取消等待/中断请求”，如果后端没有真正的运行取消能力，界面不得宣称服务器上的 Agent 已停止执行。

---

## 七、页面与路由设计

### 1. 登录页

保留正式登录逻辑，重新设计为简洁产品化页面：

- 居中登录卡片
- 产品名称和简短描述
- 用户名、密码输入
- 登录中状态
- 清晰但不泄漏内部信息的错误提示
- Enter 提交
- 密码不得写入持久化存储
- 不展示默认示例密码

### 2. 主聊天页

主页面作为产品核心，支持两种运行模式：

- Agent
- Supervisor

模式和目标选择应集成在聊天界面中，不再使用大块测试表单。

主页面结构：

```text
┌─────────────────────────────────────────────────────────┐
│ 左侧导航      │                  顶部栏                  │
│               ├─────────────────────────────────────────┤
│ 新建对话      │                                         │
│ 当前会话      │              消息内容区域               │
│ Agent         │                                         │
│ Supervisor    │                                         │
│ 待审批        ├─────────────────────────────────────────┤
│ 管理功能      │              固定输入区域               │
│ 用户菜单      │                                         │
└─────────────────────────────────────────────────────────┘
```

### 3. 左侧栏

包含：

- 新建对话
- 当前浏览器会话期间创建的线程
- Agent 工作区
- Supervisor 工作区
- 待审批入口及数量
- 用户与角色管理入口
- 当前用户菜单
- 主题切换
- 退出登录

权限不足时不展示管理入口，但后端仍然必须作为最终安全边界。

如果后端没有“会话列表”或“历史消息查询”接口：

- 不得假装服务端支持历史会话
- 不得构造假的历史记录
- 只维护当前前端生命周期内真实创建的线程索引
- 不要默认把完整聊天内容永久保存到 `localStorage`
- 刷新后无法恢复完整历史时，要用合理空状态表达

### 4. 消息区

视觉要求：

- 内容最大宽度约 760～820px
- 助手消息以自然文档形式显示，不使用夸张气泡
- 用户消息可以使用紧凑的浅色气泡
- 支持 Markdown、表格、列表、引用和代码块
- 代码块支持语言标签、复制按钮和按需高亮
- 模型输出进入 DOM 前必须经过 DOMPurify
- 外部链接安全打开
- 超长单词、URL 和代码不得撑破页面
- 消息操作按钮悬停或聚焦时出现
- 支持复制回答
- 暂不实现后端不支持的重新生成、编辑重发和分支功能

### 5. 输入区

实现接近 ChatGPT 的圆角输入容器：

- Textarea 自动增长
- 设置最大高度，超出后内部滚动
- Enter 发送
- Shift+Enter 换行
- 输入为空时禁止发送
- 请求进行中防止重复提交
- 支持取消前端请求
- 显示当前 Agent 或 Supervisor
- 移动端适配安全区域
- 不实现后端不支持的附件上传和语音输入

### 6. 工具调用卡片

根据后端真实结果展示：

- 工具名称
- 当前状态
- 成功或失败
- 简洁结果摘要
- 可折叠详情
- 结构化错误码
- 不展示完整敏感参数
- 不展示内部堆栈、Java 类名和模型原始响应

普通工具失败应作为对话中的可理解状态展示，不要把所有工具失败都转成全屏错误。

### 7. 人工审批卡片

高风险工具触发审批时，聊天中显示醒目的审批卡片：

- 风险级别
- 工具名称
- 操作摘要
- 待审批状态
- 跳转审批页按钮
- 审批通过、拒绝后的最终状态

不得在前端自行判断审批是否可以绕过。

### 8. Supervisor 展示

Supervisor 模式需要区分：

- Supervisor 分析/运行状态
- 子任务分派
- 被选择的 Agent
- 子 Agent 结果
- 最终汇总

只能展示后端真实返回的数据，不展示模型详细思维链。

### 9. 管理页面

保留并重新设计：

- 用户管理
- 角色管理
- 权限拒绝页
- HITL 审批页

管理页面可以使用更紧凑的表格风格，但必须与聊天产品共享：

- 色彩变量
- Button
- Dialog
- Form
- Toast
- 空状态
- 加载状态
- 错误状态

---

## 八、视觉设计规范

整体风格：

- 简洁
- 克制
- 中性
- 高信息密度但不拥挤
- 不使用大面积渐变
- 不使用玻璃拟态
- 不使用夸张阴影
- 不使用营销落地页式卡片堆叠
- 不照搬 ChatGPT Logo 和品牌配色

建议：

- 左侧栏宽度约 260px
- 主内容居中
- 背景以中性灰和低对比边框为主
- 圆角保持统一层级
- 使用 CSS 变量定义设计 Token
- 浅色和深色主题都必须具有足够对比度
- 使用系统字体栈，中文优先保证清晰度
- 动画控制在 120～200ms
- 尊重 `prefers-reduced-motion`
- 所有可点击元素必须有 hover、focus-visible、disabled 状态

设计 Token 至少包括：

```text
background
foreground
muted
muted-foreground
border
sidebar
sidebar-foreground
accent
accent-foreground
destructive
warning
success
ring
radius
```

---

## 九、响应式与可访问性

必须支持：

- 宽屏桌面
- 普通笔记本
- 平板
- 手机

移动端要求：

- 左侧栏改为 Sheet 抽屉
- 输入区固定在底部
- 不出现横向滚动
- 代码块允许自身横向滚动
- Dialog 不超出屏幕
- 触摸目标尺寸合理

可访问性要求：

- 全键盘操作
- 正确使用语义化 HTML
- Dialog、Dropdown 等使用 Reka UI 的可访问能力
- 图标按钮必须包含可访问名称
- 加载和状态变化提供适当的 `aria-live`
- 不能仅靠颜色表达成功、失败和审批状态
- Focus 样式不得被删除

---

## 十、建议目录结构

可以根据真实代码调整，但职责必须清晰：

```text
agent-web/src/
├── api/
│   ├── contracts/
│   ├── client.ts
│   ├── errors.ts
│   └── ...
├── chat/
│   ├── chatTransport.ts
│   ├── streamEvents.ts
│   └── streamParser.ts
├── components/
│   ├── chat/
│   ├── approvals/
│   ├── sidebar/
│   ├── layout/
│   └── ui/
├── composables/
│   ├── useAutoScroll.ts
│   ├── useChat.ts
│   ├── useChatStream.ts
│   ├── useTheme.ts
│   └── useTextareaAutosize.ts
├── queries/
├── router/
├── stores/
├── types/
├── utils/
├── views/
│   ├── chat/
│   ├── auth/
│   ├── approvals/
│   └── admin/
├── styles/
│   ├── globals.css
│   ├── markdown.css
│   └── tokens.css
├── App.vue
└── main.ts
```

避免创建无意义的：

- Manager
- Helper
- Facade
- V2
- NewComponent
- AnotherStore

---

## 十一、滚动与性能

聊天滚动行为必须正确：

- 首次进入滚动到底部
- 用户位于底部附近时，消息更新自动跟随
- 用户主动向上滚动后，不强制拉回底部
- 出现新消息时显示“回到底部”按钮
- Markdown 渲染不要在每个微小更新中重复进行昂贵高亮
- Shiki 只在代码块存在时异步加载
- 流式期间可以节流渲染
- 完成后再执行完整代码高亮
- 当前没有性能证据时，不要过早引入复杂虚拟列表

---

## 十二、错误与状态设计

必须实现统一状态：

- 页面加载中
- 空状态
- 提交中
- Agent 运行中
- 工具执行中
- 等待审批
- 已完成
- 已取消等待
- 网络失败
- Session 过期
- 权限不足
- 模型不可用
- 线程忙
- 线程挂起
- 业务错误

错误处理要求：

- 401：清理认证状态并跳转登录
- 403：显示权限不足，不要伪装为资源不存在
- 404：显示明确空状态
- 409：表达线程忙或线程挂起
- 503：表达模型或服务不可用
- 不把内部错误详情直接显示给用户
- Toast 用于短暂操作反馈
- 页面级错误使用持久错误区域
- 消息执行错误应尽量显示在对应对话上下文中

---

## 十三、测试要求

允许为本次前端重构新增必要测试，但只覆盖关键逻辑，不创建庞大测试工程。

至少覆盖：

### Vitest

- 流式事件解析器（如果存在真实流式接口）
- Zod 边界校验
- API 错误映射
- Chat Store 状态转换
- 自动滚动核心判断
- 权限路由判断

### Vue Test Utils

- ChatComposer Enter 与 Shift+Enter
- ToolCallCard 状态展示
- ApprovalCard 状态展示
- 未授权管理入口隐藏

### Playwright

至少保留适合真实环境运行的关键流程配置：

- 登录
- 进入聊天页
- 选择 Agent
- 发送消息
- 权限不足跳转

如果真实接口或运行环境不足，不得使用固定假响应冒充端到端验证。应说明哪些测试未运行及原因。

---

## 十四、实施顺序

按以下顺序实施，避免同时大范围失控：

1. 检查现有前后端真实接口
2. 完成 TypeScript 和 Vite 基础迁移
3. 建立设计 Token、Tailwind 和基础 UI 组件
4. 重构认证、API Client 和错误处理
5. 引入 Pinia 和 TanStack Vue Query，明确状态边界
6. 实现整体 App Layout 和响应式侧边栏
7. 实现主聊天页、消息模型和输入框
8. 接入真实 Agent 调用
9. 接入真实 Supervisor 调用
10. 接入工具调用和运行状态展示
11. 接入 HITL 审批
12. 重构用户和角色管理页面
13. 增加必要测试
14. 执行类型检查、测试和生产构建
15. 修复全部阻断错误后再结束

不要提前修改后端来配合假设中的前端设计。遇到必须由后端提供的新能力时，先完成可用的前端边界，并在最终结果中准确报告缺失能力。

---

## 十五、验收标准

最终必须满足：

- 项目全部使用 TypeScript
- `vue-tsc` 类型检查通过
- 不存在大面积 `any`
- 正式前端不调用 `/api/dev/**`
- 登录和 Session 逻辑正常
- Agent 调用使用真实后端接口
- Supervisor 调用使用真实后端接口
- HITL 审批使用真实后端接口
- 权限页面和导航逻辑正常
- 工具调用不会绕过后端 Gateway
- 模型输出经过安全 Markdown 渲染
- 没有 Fake Agent、固定答案或假流式输出
- 桌面端和移动端布局可用
- 浅色和深色主题可用
- 键盘操作和 Focus 状态可用
- 错误、空状态和加载状态完整
- 前端生产构建成功
- 不修改无关后端代码
- 不执行 Git commit、push 或修改远程仓库

---

## 十六、验证命令

根据实际 `package.json` 补充脚本，并至少执行：

```bash
npm install
npm run typecheck
npm run test -- --run
npm run build
```

如果配置了 Playwright 且环境具备条件，再执行：

```bash
npm run test:e2e
```

不要伪造验证结果。

---

## 十七、最终输出要求

最终回复必须明确包含：

1. 新增和修改的文件
2. TypeScript 迁移结果
3. 最终使用的前端技术栈
4. 页面和路由变化
5. 核心组件结构
6. API、Pinia、TanStack Query 的职责划分
7. Agent 和 Supervisor 的真实调用链
8. HITL 审批交互
9. 流式响应是否被后端真实支持
10. 安全 Markdown 渲染方案
11. 响应式与可访问性实现
12. 类型检查结果
13. 单元测试结果
14. 生产构建结果
15. 未完成验证及准确原因
16. 发现但未处理的非本次问题

禁止把计划描述成已完成结果。