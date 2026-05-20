# REQ-004 M3 UI 测试报告

> 测试人员：ui-tester
> 日期：2026-05-20
> 前端 commit：`69a8cd2d5`（group_chat 分支）
> 测试方式：代码静态分析

---

## 1. 测试概览

| 项目 | 内容 |
|------|------|
| 测试范围 | 桌面端群聊设置 / 移动端群聊设置 / 配置变更广播 / autoReply 标签 / 限流 UI / i18n / 回归 |
| 测试输入 | design-frontend.md v1.3、commit 69a8cd2d5 |
| 平台 | Windows 11 桌面端（静态分析） |

---

## 2. 测试结果摘要

| 级别 | 数量 | 状态 |
|------|------|------|
| P0 - 严重 | 0 | — |
| P1 - 重要 | 1 | 需修复 |
| P2 - 中等 | 0 | — |
| P3 - 低 | 0 | — |
| 通过 | 6 | ✅ |

---

## 3. 详细发现

### P1 — layout/index.vue 缺少 AICLAW_GROUP_CONFIG_UPDATE WS 事件处理

**文件**：`src/layout/index.vue`

**问题描述**：
`layout/index.vue` 中的 Mitt WS 事件监听器包含：
- `RECEIVE_MESSAGE` ✅
- `STREAM_START` / `STREAM_DELTA` / `STREAM_END` ✅
- `THINKING_START` / `THINKING_DELTA` / `THINKING_END` ✅
- `AICLAW_AUTH_REQUEST` ✅

但**缺少** `AICLAW_GROUP_CONFIG_UPDATE`（即 `groupConfigChange`）的 handler。

**影响**：
主人修改群配置后，群内其他在线成员前端**无法收到 WS 通知**，不会触发 `chatStore.updateAiclawGroupConfig()` 更新本地缓存，也不会显示 toast 提示。

**修复建议**：
在 `layout/index.vue` 中追加：

```typescript
useMitt.on(WsResponseMessageType.AICLAW_GROUP_CONFIG_UPDATE, (data: AiclawGroupConfigUpdatePayload) => {
  chatStore.updateAiclawGroupConfig(data.aiclawUid, data.roomId, data.config)
  // 非主人时显示配置变更提示
  if (data.aiclawUid !== currentUserUid) {
    const aiclawName = chatStore.getAiclawName?.(data.aiclawUid) || 'AI'
    window.$message?.info(t('aiclaw.group_settings.config_changed', { name: aiclawName }))
  }
})
```

---

## 4. 通过项

### 4.1 桌面端群聊设置 ✅

**文件**：`src/views/aiAssistantWindow/index.vue`

- aiclaw 列表 → 选择后 detail 视图有"群聊设置"入口（第140-145行）
- `rightView === 'groupSettings'` 子视图存在（第290-341行）
- 配置表单：频率限制（n-input-number, min=0, max=100）、每日上限（n-input-number, min=0, max=10000）、响应其他 AI（n-switch）、@ 触发（n-switch）
- 保存按钮调用 `chatStore.saveAiclawGroupConfig()`，loading 状态按 roomId 追踪
- 空状态展示机器人图标 + "该 AI 助理尚未加入任何群聊"

### 4.2 移动端群聊设置 ✅

**文件**：`src/mobile/views/my/AiclawGroupSettings.vue` / `src/mobile/views/my/AiAssistantDetail.vue` / `src/router/index.ts`

- `AiclawGroupSettings.vue` 完整页面：HeaderBar + 配置表单 + 保存按钮 + 空状态 + loading
- `AiAssistantDetail.vue` 第76-82行有"群聊设置"入口，路由跳转 `/mobile/mobileMy/aiAssistant/${uid}/groupSettings`
- `router/index.ts` 第287-289行：`path: 'aiAssistant/:uid/groupSettings'` 路由已注册

### 4.3 autoReply 标签 ✅

**文件**：`src/layout/index.vue` / `src/components/rightBox/chatBox/ChatMain.vue`

- `layout/index.vue` 第373-377行：收到 WS `RECEIVE_MESSAGE` 时检测 `extra?.autoReply === true`，调用 `chatStore.markMessageAsAutoReply()`
- `ChatMain.vue` 第90-93行：消息气泡上方条件渲染 `auto-reply-tag`，文案 `t('aiclaw.auto_reply_tag')`，样式 `text-(10px #999)`

### 4.4 i18n ✅

**文件**：`locales/zh-CN/aiclaw.json` / `locales/en/aiclaw.json`

- `group_settings` 全部 10 个 key 双语完整：`title`, `empty`, `rate_limit`, `rate_limit_hint`, `daily_limit`, `respond_to_ai`, `mention_required`, `mention_required_hint`, `save`, `save_success`, `save_failed`, `config_changed`
- `auto_reply_tag` 双语完整
- `detail.group_settings` 双语完整

### 4.5 限流场景 UI 反馈 ✅

**文件**：`src/components/rightBox/chatBox/ThinkingCard.vue`

- error 状态显示 `thinking.errorMsg` 或默认 "思考出错"（第22-24行）
- 文字颜色使用 `[--danger-text]`，暗色模式自适应
- 频率限制的错误信息可由后端通过 `THINKING_END` payload 的 `errorMsg` 字段传递，前端正确展示

### 4.6 M2 回归 ✅

- `ThinkingPanel.vue` / `ThinkingCard.vue` / `chat.ts` 中 M2 相关逻辑（thinkingStreams / archive / setTimeout 竞态修复）未被 M3 改动影响
- `ChatMain.vue` 中 `showThinkingPanel` 的 M2 修复（userType === 4 常驻显示）保留完好

---

## 5. GUI 测试

| 检查项 | 结果 | 说明 |
|--------|------|------|
| 应用启动 | — | 同 M2，dev 编译可用 |
| 登录 | 阻塞 | 无凭据，无法进入主界面 |
| 群聊设置 UI | 未测试 | 需登录 + aiclaw 列表加载 |
| 配置保存 | 未测试 | 需后端 API 可用 |
| WS 通知 | 未测试 | 需多客户端联调 |
| autoReply 标签 | 未测试 | 需限流场景触发 |

**深度 GUI 测试**：等待测试账号到位后执行（manager 已安排）。

---

## 6. 建议

1. **立即修复 P1**：在 `layout/index.vue` 中补全 `AICLAW_GROUP_CONFIG_UPDATE` WS handler，否则群配置变更广播功能不工作
2. 其余 6 项通过静态分析，GUI 端到端验证待账号到位后补测

---

## 7. 附件

- 前端 commit：`69a8cd2d5` (`feat(REQ-004/M3): aiclaw group config + autoReply UI`)
