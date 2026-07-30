# 前端严重回归修复：会话切换丢失与 Tailwind 样式未生效

## 一、任务性质

这是对上一轮实现的返工，不是新增功能。

当前有两个已经复现的严重问题：

1. 切换到另一个对话，再切换回来，原会话消息全部消失。
2. 页面几乎显示成浏览器默认样式，没有达到参考图效果。

问题截图：

```text
C:\Users\KC\AppData\Local\Temp\codex-clipboard-6259534e-6f0b-42f4-9d05-60b46307850a.png
```

目标设计图：

```text
C:\Users\KC\AppData\Local\Temp\codex-clipboard-5b70485e-8842-4c5f-9c56-964a2da696d3.png
```

必须先检查真实代码，再实施修复，不得只调整文字或宣称验证通过。

---

## 二、已经确认的根因

### 2.1 会话消息被主动清空

当前文件：

```text
agent-web/src/stores/chat.ts
```

当前 `switchThread()` 中存在：

```ts
currentThreadId.value = threadId
messages.value = []
```

项目只保存了线程摘要，没有按会话保存和恢复消息。因此切换会话时，消息被永久清空。

不能只删除 `messages.value = []`，否则会导致不同会话共享同一份消息。

必须实现真正的多会话消息隔离。

### 2.2 Tailwind CSS 没有进入样式入口

当前文件：

```text
agent-web/src/styles/globals.css
```

目前只有：

```css
@import "./tokens.css";
```

项目使用 Tailwind CSS 4 和 `@tailwindcss/vite`，但没有在 CSS 入口引入 Tailwind：

```css
@import "tailwindcss";
```

因此 Vue 模板中的以下工具类基本没有生成：

```text
flex
h-dvh
w-[260px]
rounded-lg
border
px-3
py-2
text-sm
bg-[var(...)]
```

截图中出现默认蓝色链接、默认按钮、图标尺寸异常、布局失效，正是 Tailwind 工具类未生效的表现。

---

# 三、会话状态重新设计

## 3.1 每个会话必须拥有独立消息

不要继续使用全局唯一的：

```ts
const messages = ref<ChatMessage[]>([])
```

作为所有会话共同的数据源。

应建立明确的会话模型，例如：

```ts
interface Conversation {
  conversationId: string
  threadId?: string
  title: string
  messages: ChatMessage[]
  draft: string
  createdAt: number
  lastMessageAt: number
}
```

注意：

- `conversationId` 是前端稳定 ID。
- `threadId` 是后端首次响应后返回的真实线程 ID。
- 新建对话在获得后端 `threadId` 前，也必须拥有稳定的 `conversationId`。
- Sidebar 应使用 `conversationId` 选择会话。
- 调用后端时使用当前会话的 `threadId`。
- 不要把 `conversationId` 冒充后端 `threadId`。
- 不要使用 Supervisor 名称作为会话 ID。

建议状态：

```ts
const conversations = ref<Conversation[]>([])
const activeConversationId = ref<string | null>(null)

const activeConversation = computed(...)
const messages = computed(() => activeConversation.value?.messages ?? [])
```

如果现有组件要求可修改的消息引用，可使用明确的 Store 方法：

```ts
addMessage(conversationId, message)
updateMessage(conversationId, messageId, patch)
```

不要通过不安全的可写 computed 或类型断言掩盖问题。

## 3.2 新建对话

点击“新建对话”时：

1. 取消当前前端等待。
2. 创建新的 `conversationId`。
3. 新会话的 `threadId` 初始为空。
4. 消息列表为空。
5. 标题为“新对话”。
6. 将新会话设为当前会话。
7. 不得删除已有会话。
8. 不得清空其他会话的消息。
9. 不自动发送虚假欢迎消息。

## 3.3 首次收到响应

后端首次返回真实 `threadId` 后：

- 将该 `threadId` 绑定到发起请求的会话。
- 不要根据当前页面状态把响应写入另一个会话。
- 标题使用该会话第一条用户消息的安全截断版本。
- 不要使用系统回答作为会话标题。
- 更新 `lastMessageAt`。
- Sidebar 中不得重复插入同一个会话。

## 3.4 切换会话

切换时：

- 根据 `conversationId` 修改当前会话。
- 展示该会话自己的消息。
- 不得执行 `messages = []`。
- 不得把 A 会话的消息显示到 B 会话。
- 不得把 A 会话的 `threadId` 用于 B 会话请求。
- 不得删除任何已有消息。
- 每个会话的草稿应独立保存，或切换时明确保存当前草稿。
- 滚动位置可以回到底部，但不能影响消息内容。

## 3.5 异步请求隔离

每次请求必须捕获：

```ts
conversationId
requestId
AbortController
```

请求完成后：

- 响应必须写回发起请求的会话。
- 用户切换到其他会话时，旧响应不得写入当前会话。
- 旧请求的 `finally` 不得清理新请求的 Controller。
- 取消 A 后发送 B，A 的异常不得把 B 状态改成 `idle`。
- 审批消息也必须归属于发起请求的会话。

可以在切换会话时取消旧请求，但取消后必须保留该会话已有消息。

## 3.6 浏览器存储

首先保证当前前端生命周期内切换会话不会丢失。

如果现有设计要求刷新页面后继续显示会话，则使用经过 Zod 校验的 `sessionStorage` 或已有存储机制保存安全的展示数据。

不得保存：

- Session ID
- 密码
- API Key
- 权限集合
- 原始工具参数
- 完整 metadata
- 内部 Agent 信息

401 和登出必须清除全部会话数据。

如果后端没有历史消息查询接口，必须在最终输出中明确说明：

```text
当前会话历史由前端保存；后端暂未提供历史消息查询接口。
```

不得伪造后端历史查询功能。

---

# 四、恢复 Tailwind CSS

修改：

```text
agent-web/src/styles/globals.css
```

确保 Tailwind 4 被正确引入，例如：

```css
@import "tailwindcss";
@import "./tokens.css";
```

必须以当前 Tailwind 4 和 `@tailwindcss/vite` 的真实版本为准。

不得：

- 使用旧版 `@tailwind base/components/utilities` 写法，除非当前真实版本明确需要。
- 手写大量重复 CSS 来绕过 Tailwind 未生效问题。
- 删除 Vite 中的 Tailwind 插件。
- 通过截图假装样式已生效。
- 只执行构建而不进行真实浏览器检查。

修复后必须确认：

- `flex` 实际产生 `display: flex`。
- `h-dvh` 实际占满窗口高度。
- Sidebar 宽度实际约为设计值。
- Lucide 图标尺寸受到 `w-4 h-4` 等类控制。
- 按钮不再使用浏览器默认样式。
- 页面不存在默认蓝色下划线导航。
- 边框、背景、间距、圆角和悬停效果真实生效。

---

# 五、按参考图重新完成视觉设计

Tailwind 恢复后，继续按照参考图调整，不能停留在“样式能显示”。

## 5.1 左侧栏

桌面端：

- 宽度约 `300px`，可根据实际页面微调。
- 顶部显示产品标识“智能协作”。
- “新建对话”为完整描边按钮。
- 会话按“今天、昨天、更早”分组。
- 每条会话显示图标、标题和时间。
- 当前会话使用淡紫背景和左侧强调线。
- 底部显示待审批、权限允许的管理入口、主题、用户信息、退出登录。
- 用户区域和退出入口的视觉层级应接近参考图。

不得显示：

- Supervisor 名称
- Agent 名称
- Agent/Supervisor 选择器

## 5.2 顶部栏

增加清晰的会话顶部栏：

- 显示当前会话标题。
- 新会话显示“新对话”。
- 可显示“服务可用”状态。
- 不显示 `general_supervisor`。
- 不显示“切换 Agent”。
- 与左侧栏有清晰边界。
- 高度、留白、字体层次参考设计图。

## 5.3 消息区域

- 主内容居中，设置合理最大宽度。
- 用户消息靠右，使用淡紫色圆角气泡。
- 系统消息靠左，配统一系统头像。
- 系统内容使用白色或浅色内容区域。
- 消息之间有充分垂直间距。
- 时间信息弱化显示。
- Markdown、列表和代码块排版清晰。
- 工具调用使用紧凑状态卡片。
- 不要把消息贴在页面边缘。

## 5.4 输入区

- 固定在聊天区域底部。
- 使用白色大圆角输入容器。
- 宽度与消息主区域一致。
- 有柔和边框和阴影。
- 发送按钮使用绿色或设计系统主色。
- Enter 发送，Shift+Enter 换行。
- 下方显示：

```text
内容由 AI 生成，请仔细甄别
```

- 没有真实功能的附件、礼物、联网按钮不得照搬。

## 5.5 空状态

当前“向系统发送消息开始对话”过于简陋。

应设计成完整空状态：

- 系统图标。
- 标题，例如“有什么可以帮你？”
- 简短说明。
- 与输入框形成清晰视觉层次。
- 不显示 Supervisor 或 Agent 技术概念。

---

# 六、同时修复已发现的条件渲染错误

检查：

```text
agent-web/src/views/chat/ChatView.vue
```

当前“加载中”和“系统不可用”使用了相同条件：

```vue
v-if="!chatStore.isReady && chatStore.supervisorLoaded"
v-else-if="!chatStore.isReady && chatStore.supervisorLoaded"
```

第二个分支永远无法进入。

必须建立明确状态：

- `loading`
- `ready`
- `unavailable`

要求：

- 未完成请求时显示加载。
- 请求成功且唯一 Supervisor 可用时进入聊天。
- 返回空列表、多个 Supervisor 或请求失败时显示不可用。
- “重试”必须真正重新请求。
- 当前 `initSupervisor()` 不能因为 `supervisorLoaded` 已经是 `true` 而导致重试无效。
- 多个 Supervisor 时不得擅自使用第一个。
- 用户界面不得暴露 Supervisor 名称。

---

# 七、测试要求

至少新增或修复以下测试。

## 7.1 会话切换

```text
创建会话 A
→ 发送消息并获得响应
→ 创建会话 B
→ 发送另一条消息
→ 切换回 A
→ A 的用户消息和系统回复仍完整存在
→ A 不包含 B 的消息
→ 再切换回 B
→ B 的消息仍完整存在
```

还要覆盖：

- 新建会话不删除旧会话。
- 不同会话拥有不同 `threadId`。
- 响应写回发起请求的会话。
- 切换期间旧响应不污染当前会话。
- 401 和登出清除所有会话。

## 7.2 样式生效

在真实浏览器中断言：

- Sidebar 的 computed `display` 为 `flex`。
- Sidebar 实际宽度符合设计。
- 页面根布局横向排列。
- 输入框位于底部。
- 用户消息靠右。
- 系统消息靠左。
- 按钮具有圆角、边框和设计颜色。
- 页面无横向溢出。
- 不存在浏览器默认蓝色链接样式。

只检查 class 字符串存在不算通过，必须检查真实 computed style 或截图。

## 7.3 视觉检查

分别生成并检查：

- 1440×900 桌面截图。
- 390×844 移动端截图。
- 空会话截图。
- 有多轮消息和代码块的截图。
- 审批弹窗截图。
- 深色主题截图。

必须与目标设计图逐项对照：

- 布局
- 间距
- 字体层级
- 消息对齐
- 颜色
- 边框
- 圆角
- 输入区
- 会话列表

不得只说“接近设计图”。

---

# 八、真实验证

完成全部修改后执行：

```bash
npm run typecheck
npm test -- --run
npm run build
npm run test:e2e
```

并进行真实浏览器验证。

最终必须报告：

- 每条命令退出码。
- 测试通过、失败和跳过数量。
- Tailwind 是否真实生效。
- 桌面和移动端截图位置。
- 会话 A/B 往返切换结果。
- 刷新后的会话行为。
- 控制台错误。
- 网络错误。
- 未完成验证及准确原因。

---

# 九、禁止事项

- 禁止只删除 `messages.value = []`。
- 禁止所有会话共享同一消息数组。
- 禁止使用固定假历史。
- 禁止 Mock 数据冒充真实 E2E。
- 禁止只修改 CSS 文案。
- 禁止只看构建成功就宣称 UI 正确。
- 禁止忽略参考图。
- 禁止暴露 Supervisor 和 Agent 内部名称。
- 禁止修改后端测试 Agent 接口。
- 禁止伪造测试结果。

---

# 十、最终输出

```markdown
## 1. 根因
说明会话丢失和样式失效的真实原因。

## 2. 修改文件
列出全部新增、修改和删除文件。

## 3. 会话数据模型
说明 conversationId、threadId、messages 和切换流程。

## 4. 视觉实现
逐项说明与参考图的对应关系。

## 5. 测试结果
列出命令、退出码和测试数量。

## 6. 浏览器验证
列出桌面、移动端和会话切换验证证据。

## 7. 未验证内容
准确列出未验证项和原因。

## 8. 执行台账
确认 `.kscc-prompts/web-execution-ledger.md` 已更新。
```