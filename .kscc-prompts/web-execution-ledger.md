# Agent Platform 前端执行台账

## 执行日期：2026-07-30

---

## 1. 单 Supervisor 自动获取

**修复前：** 用户需要手动选择 Supervisor，页面展示 SupervisorSelector 和 Supervisor 技术名称。

**修复后：**
- ChatStore 新增 `initSupervisor()` 方法，自动调用 `GET /api/framework/supervisors` 获取列表，使用第一个（也是唯一一个）Supervisor
- `_supervisorName` 为内部变量（下划线前缀），不暴露给 UI
- `isReady` computed 替代 `hasSupervisor`，标识系统是否就绪
- ChatView 的 `onMounted` 自动调用 `chatStore.initSupervisor()`
- SupervisorSelector.vue 已删除

**代码位置：** `src/stores/chat.ts:56-65`, `src/views/chat/ChatView.vue:36-38`

---

## 2. 不展示 Supervisor/Agent 技术名称

**修复前：** ChatMessage 显示 `supervisorName` 和 `agentName`，侧边栏线程显示 `supervisorName`，ChatComposer 显示 Supervisor 名称。

**修复后：**
- ChatMessage 助手消息不显示 Supervisor/Agent 名称，以"系统"身份展示
- ChatComposer 不显示 Supervisor 名称
- 侧边栏线程不显示 Supervisor 名称
- ThreadEntry 类型移除 `supervisorName` 字段
- AssistantChatMessage 类型移除 `supervisorName` 和 `agentName` 字段

**代码位置：** `src/components/chat/ChatMessage.vue`, `src/components/chat/ChatComposer.vue`, `src/components/sidebar/Sidebar.vue`, `src/types/index.ts:280-284`

---

## 3. 危险工具审批弹窗

**修复前：** 审批在聊天流中展示为卡片，用户需要点击"前往审批"按钮跳转独立页面。

**修复后：**
- 新增 `ApprovalPopup.vue` 组件，基于 Reka UI Dialog
- ChatView 中检测 `pendingApproval` 消息，立即弹出审批弹窗
- 弹窗显示操作名称、风险等级、原因
- 支持"通过"和"拒绝"操作
- 不展示 Supervisor/Agent 技术名称
- ApprovalCard 保留在聊天流中作为历史记录
- 审批操作直接调用 `approvalApi.decideAndResume`，不需要跳转页面

**代码位置：** `src/components/chat/ApprovalPopup.vue`, `src/views/chat/ChatView.vue:42-78`

---

## 4. 审批 API 调用链

```
ChatView.handleApprove/handleReject
  → approvalApi.decideAndResume(runId, approvalId, action, comment)
    → POST /api/hitl/approvals/{runId}/decide-and-resume
      → Zod Schema 校验响应
        → 更新 chatStore 消息状态
          → invalidateQueries(['hitl', 'approvals'])
```

---

## 5. 遗留问题修复

| 问题 | 修复 | 位置 |
|------|------|------|
| SupervisorSelector 仍存在 | 已删除 | `src/components/chat/SupervisorSelector.vue` (removed) |
| ChatMessage 展示技术名称 | 移除 supervisorName/agentName 显示 | `src/components/chat/ChatMessage.vue` |
| 侧边栏展示 supervisorName | 移除 | `src/components/sidebar/Sidebar.vue` |
| ThreadEntry 含 supervisorName | 移除字段 | `src/types/index.ts`, `src/api/contracts/schemas.ts` |
| AssistantChatMessage 含 supervisorName/agentName | 移除字段 | `src/types/index.ts` |
| E2E 期望看到 Supervisor 选择器 | 更新为不得出现选择器 | `e2e/app.spec.ts` |
| threadEntrySchema 允许旧字段 | 添加 .strict() | `src/api/contracts/schemas.ts` |

---

## 6. 测试与构建

| 检查项 | 结果 | 命令 |
|--------|------|------|
| TypeScript 类型检查 | ✅ 0 错误 | `npx vue-tsc --noEmit` |
| 单元测试 | ✅ 34/34 通过 | `npx vitest run` |
| 生产构建 | ✅ 成功 | `npx vite build` |
| 技术名称搜索 | ✅ UI 中无 supervisorName/agentName/general_supervisor | `grep` |
| 选择器搜索 | ✅ 无 SupervisorSelector/AgentSelector | `grep` |

---

## 7. 浏览器验证

需要运行中的后端。桌面端/移动端/深色主题验证依赖 `npm run dev` + 后端。

---

## 8. 未验证内容

1. **Playwright 浏览器未安装** — 需 `npx playwright install`
2. **真实后端 E2E** — 需运行中的后端和凭据环境变量
3. **审批弹窗真实触发** — 需后端配置危险工具触发 HITL
4. **Shiki 包体积** — WASM 语言文件仍较大

---

## 10. 执行台账

✅ `.kscc-prompts/web-execution-ledger.md` 已更新
