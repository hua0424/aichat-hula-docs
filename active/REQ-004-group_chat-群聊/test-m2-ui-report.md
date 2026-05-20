# REQ-004 M2 UI 测试报告

> 测试人员：ui-tester
> 日期：2026-05-20
> 前端 commit：`896f1b848`（group_chat 分支）
> 测试方式：代码静态分析 + 有限 GUI 验证

---

## 1. 测试概览

| 项目 | 内容 |
|------|------|
| 测试范围 | ThinkingPanel / ThinkingCard / ChatMain.vue 集成 / 归档抽屉 / 回归测试 |
| 测试输入 | design-frontend.md v1.2、需求.md v2.1、commit 896f1b848 |
| 平台 | Windows 11 桌面端 |
| GUI 环境 | Tauri dev 模式（login 窗口可启动，主界面因凭据缺失未进入） |

---

## 2. 测试结果摘要

| 级别 | 数量 | 状态 |
|------|------|------|
| P0 - 严重 | 1 | 需立即修复 |
| P1 - 重要 | 1 | 建议本轮修复 |
| P2 - 中等 | 2 | 建议确认后修复 |
| P3 - 低 | 2 | 可选优化 |
| 通过 | 4 | 静态分析通过 |

**总体结论**：M2 前端代码存在 1 个 P0 竞态条件 bug，需修复后方可提测通过。其余问题建议在联调前处理。

---

## 3. 详细发现

### P0 — finalizeThinking setTimeout 竞态条件（严重）

**文件**：`src/stores/chat.ts:1768-1785`

**问题描述**：
`finalizeThinking` 在 thinking 完成后设置 30s 延迟归档：

```typescript
setTimeout(() => {
  archiveThinking(state)
  thinkingStreams.delete(key)
}, 30_000)
```

如果同一个 aiclaw 在这 30s 内再次触发 thinking（收到新消息 → THINKING_START）：
1. `startThinking` 会用新状态覆盖 `thinkingStreams` 中同一 `key`
2. 30s 后旧 `setTimeout` 执行，`thinkingStreams.delete(key)` 会**误删新的 thinking 状态**
3. 用户将看到 ThinkingCard 突然消失

**复现路径**：
1. 群聊中 aiclaw 完成一次 thinking（THINKING_END）
2. 30s 内群内再次有消息触发该 aiclaw thinking
3. 新 ThinkingCard 显示后，在旧 30s 超时点突然消失

**修复建议**：
```typescript
setTimeout(() => {
  const current = thinkingStreams.get(key)
  if (current === state) {
    archiveThinking(state)
    thinkingStreams.delete(key)
  }
}, 30_000)
```

---

### P1 — ThinkingCard 暗色模式文字颜色失效

**文件**：`src/components/rightBox/chatBox/ThinkingCard.vue:96-98`

**问题描述**：
```typescript
const textColorClass = computed(() => {
  return '#333 dark:[--text-color]'
})
```

`:class` 绑定接收字符串时，Vue 会将其作为**类名列表**解析。`'#333'` 含 `#` 字符，不是有效的 CSS 类名；`dark:[--text-color]` 虽可能被 UnoCSS 识别，但整个绑定方式不合法。

**影响**：暗色模式下 ThinkingCard 头部文字颜色不会自适应切换。

**修复建议**：
```vue
<span class="text-(12px font-500) truncate text-#333 dark:text-[--text-color]">
```

或删除 `textColorClass` computed，直接在模板中写入颜色类。

---

### P2 — showThinkingPanel 与设计文档行为不一致

**文件**：`src/components/rightBox/chatBox/ChatMain.vue:285-295`

**问题描述**：
实际代码中群聊场景下 `showThinkingPanel` 仅在以下情况返回 `true`：
- `chatStore.isCurrentRoomThinking`（有活跃 thinking）
- `chatStore.thinkingArchive.get(roomId)?.length`（有归档记录）

设计文档 §3.4.3 描述的行为包含 `hasAiclawMemberInCurrentGroup()` 检查：群聊中**只要有 aiclaw 成员**即预留 ThinkingPanel 区域。

**影响**：群聊首次打开时 ThinkingPanel 不显示，收到消息后突然出现，可能导致消息列表跳动（layout shift）。

**建议**：与 frontend-dev 确认设计意图，如按设计文档实现需补充 `hasAiclawMemberInCurrentGroup()` 检查。

---

### P2 — AICLAW_GROUP_CONFIG_UPDATE 枚举值潜在不一致

**文件**：`src/services/wsType.ts:97`

**问题描述**：
```typescript
AICLAW_GROUP_CONFIG_UPDATE = 'groupConfigChange'
```

设计文档 §3.1 中描述为 `'aiclawGroupConfigUpdate'`。虽然 commit `01e152d69` 已修正为 `'groupConfigChange'`，但需与 server-dev 确认后端实际发送的 WS type 字符串，避免两端不匹配导致事件丢失。

---

### P3 — Transition mode="out-in" 切换空白

**文件**：`src/components/rightBox/chatBox/ThinkingPanel.vue:2-25`

**问题描述**：`mode="out-in"` 在活跃 thinking 列表与归档入口切换时，先等待旧元素退出再进入新元素，可能产生短暂空白。

**建议**：如 UX 要求更流畅，可改为 `mode="default"` 或移除 mode 属性。

---

### P3 — startThinking 未清理旧 setTimeout

**文件**：`src/stores/chat.ts:1727-1732`

**问题描述**：当 aiclaw 有新 thinking 覆盖旧的未完成 thinking 时：
```typescript
if (existing && existing.status === 'thinking') {
  existing.status = 'error'
  existing.errorMsg = 'Superseded by new thinking'
  existing.endTime = Date.now()
  archiveThinking(existing)
}
```

旧 thinking 被立即归档，但如果之前 `finalizeThinking` 已为该 thinking 设置了 30s `setTimeout`，该超时仍在运行，可能导致 30s 后误操作。

**建议**：使用 `clearTimeout` 机制或唯一标识符避免过期超时执行。

---

## 4. 回归测试

| 检查项 | 结果 | 说明 |
|--------|------|------|
| ISS-015 修复影响 | 通过 | `thinkingStreams` 与 `chatMessageList` / `streamingMessages` / `pendingStreamReplace` 完全隔离，不共享状态 |
| STREAM 协议私聊流式 | 通过 | 现有 `streamDeltaBuffers` / `streamRafId` 与 thinking 对应变量独立，互不影响 |
| `chatMessageList` 行为 | 通过 | thinking 内容不进入消息列表，不经过排序/去重/placeholder 替换逻辑 |
| `normalizeMsgSendTime` | 通过 | thinking 使用独立的 `startTime` / `endTime`，不调用消息时间戳归一化 |
| `pushMsg()` 隔离 | 通过 | thinking 不调用 `pushMsg`，不触发会话未读计数等副作用 |

---

## 5. GUI 验证记录

| 检查项 | 结果 | 说明 |
|--------|------|------|
| 应用启动 | 通过 | Tauri dev 编译成功，login 窗口 336x499 正常显示 |
| 登录窗口高度 | 通过 | 490px 修复生效，底部栏完整显示无截断 |
| 主界面进入 | 阻塞 | 无登录凭据（localStorage 无 token），无法进入主界面测试 ThinkingPanel |
| ThinkingPanel 渲染 | 未测试 | 需登录后进入含 aiclaw 的群聊/私聊 |
| ThinkingCard 交互 | 未测试 | 需 WS thinkingStart/Delta/End 事件触发 |
| 归档抽屉 | 未测试 | 需上述前提条件 |

**GUI 测试受限原因**：
1. 本地 dev 环境无保存的登录凭据
2. ThinkingPanel 功能依赖后端 WS 事件（`thinkingStart` / `thinkingDelta` / `thinkingEnd`），需 server-dev / plugin-dev 配合

---

## 6. 建议

1. **立即修复 P0**：`finalizeThinking` 的 setTimeout 竞态条件，建议采用闭包内引用校验方案
2. **本轮修复 P1**：ThinkingCard 暗色模式文字颜色绑定
3. **联调前确认 P2**：
   - 与 frontend-dev 确认 `showThinkingPanel` 行为是否按设计文档实现
   - 与 server-dev 确认 `groupConfigChange` WS type 字符串
4. **GUI 完整测试**：获取测试账号密码后，进入含 aiclaw 的群聊进行端到端验证

---

## 7. 附件

- 前端 commit：`896f1b848` (`REQ-004(M2): frontend thinking UI — store + events + components`)
- 设计文档：`design-frontend.md` v1.2
