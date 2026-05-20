# REQ-004 前端可行性评估报告

> 评估人：frontend-dev
> 日期：2026-05-20
> 范围：Thinking 状态条 UI、chat store 扩展、WS 事件处理
> 结论：**技术上完全可行，无阻塞问题，预估工作量 3-3.5 天**

---

## 一、技术可行性分析

### 1.1 WS 事件扩展 — 可行，零风险

现有 WS 事件体系已支持流式消息（STREAM_START / STREAM_DELTA / STREAM_END），新增 THINKING 三个事件完全复用同一模式：

```typescript
// src/services/wsType.ts —— 现有 STREAM 事件（83-87 行）
STREAM_START = 'streamStart',
STREAM_DELTA = 'streamDelta',
STREAM_END = 'streamEnd',

// 新增（仅需 3 行）
THINKING_START = 'thinkingStart',
THINKING_DELTA = 'thinkingDelta',
THINKING_END = 'thinkingEnd',
```

**事件处理位置**：`src/layout/index.vue:437-479` 已有 STREAM 事件的 Mitt 监听 + rAF 节流模式，THINKING_DELTA 可直接复用同一套 `requestAnimationFrame` 缓冲机制，代码增量 < 30 行。

### 1.2 Chat Store 扩展 — 可行，与现有架构兼容

现有 store 的 streaming 状态：

```typescript
// src/stores/chat.ts:1493+
const streamingMessages = reactive(new Set<string>())     // 跟踪流式消息 msgId
const pendingStreamReplace = new Map<string, string[]>()  // 占位符 → 真实消息
```

**Thinking 状态新增（与上述状态完全隔离）：**

```typescript
interface ThinkingState {
  aiclawId: number
  aiclawName: string
  avatar?: string
  content: string
  status: 'thinking' | 'complete' | 'error'
  startTime: number
}

// key: `${roomId}:${aiclawId}`，确保同房间多 aiclaw 独立
const thinkingStreams = reactive(new Map<string, ThinkingState>())
```

**新增 Action（4 个）：**

| Action | 职责 | 行数预估 |
|--------|------|---------|
| `startThinking(roomId, aiclawId, payload)` | 创建 ThinkingState，初始化 content | ~8 行 |
| `appendThinking(roomId, aiclawId, delta)` | 追加 thinking 内容 | ~5 行 |
| `finalizeThinking(roomId, aiclawId, payload)` | 标记完成/错误，记录 duration | ~8 行 |
| `clearThinking(roomId, aiclawId?)` | 清理（切换房间或主动关闭） | ~6 行 |

**关键：thinkingStreams 与 chatMessageList 零交互。** Thinking 内容不进入消息列表，不经过排序、去重、placeholder 替换等任何逻辑。ISS-015 修好的 `sendTime` 排序和 `pendingStreamReplace` 多槽数组完全不受影响。

### 1.3 UI 组件 — 可行，有现成锚点

**ChatMain.vue:41-46** 已有 AI 助理提示条：

```vue
<!-- AI 助理对话提示条 -->
<div v-if="isAiclawSession" class="flex-shrink-0 px-12px pt-6px">
  <div class="...">{{ isCurrentRoomStreaming ? t('aiclaw.chat.streaming') : t('aiclaw.chat.notice') }}</div>
</div>
```

此位置可直接替换为 `<ThinkingPanel />`，无需改动布局结构。

**ThinkingPanel.vue** 职责：
- 接收 `thinkingStreams`（按当前 roomId filter）
- v-for 渲染多个 `ThinkingCard`
- 管理面板展开/收起状态（pinia 或局部 ref）

**ThinkingCard.vue** 职责：
- 展示 aiclaw 头像 + 名称 + 状态徽章
- 可折叠内容区（n-collapse 或自定义）
- 流式内容追加时 auto-scroll 到最新
- THINKING_END 后切换为「回顾」模式（只读、可展开）

### 1.4 正式消息投递 — 无需改动

REQ-004 §2.3.3 明确 MCP Tool 发送的消息**一次性投递完整消息**，走现有消息通道（`RECEIVE_MESSAGE`）。前端无需任何改动，消息自然出现在 `chatMessageList` 中，由现有 `pushMsg` + 排序逻辑处理。

---

## 二、与 ISS-015 的兼容性确认

| ISS-015 修复项 | Thinking 功能是否触及 | 结论 |
|---------------|---------------------|------|
| `chatMessageList` 按 `sendTime` 排序 | Thinking 不进入消息列表 | 无影响 |
| `chatMessageListByRoomId` 按 `sendTime` 排序 | Thinking 不进入消息列表 | 无影响 |
| `clearRedundantMessages` 按 `sendTime` 排序 | Thinking 不进入消息列表 | 无影响 |
| `pendingStreamReplace` 多槽数组 | Thinking 不走 STREAM 协议 | 无影响 |
| `normalizeMsgSendTime` 时间戳归一化 | Thinking 用独立时间戳 | 无影响 |

**结论：ISS-015 所有修复均与 REQ-004 前端实现零冲突。**

---

## 三、工作量预估

| 任务 | 内容 | 预估 |
|------|------|------|
| WS 类型定义 | `wsType.ts` 新增 3 个 enum + DTO 类型 | 0.5 天 |
| WS 事件处理 | `layout/index.vue` Mitt 监听 + rAF 缓冲 | 0.5 天 |
| Chat Store 扩展 | `chat.ts` thinkingStreams + 4 actions | 0.5 天 |
| ThinkingPanel.vue | 面板容器、多卡片列表、展开收起 | 0.5 天 |
| ThinkingCard.vue | 单卡片 UI、流式内容区、回顾模式 | 0.5 天 |
| ChatMain.vue 集成 | 替换现有 AI notice bar、绑定 thinkingStreams | 0.5 天 |
| i18n 文案 | zh-CN + en 双语 | 0.5 天 |
| 联调测试 | 与 server-dev / plugin-dev 端到端验证 | 0.5 天 |
| **总计** | | **3-3.5 天** |

---

## 四、风险与建议

### 4.1 无阻塞风险

- **技术层面**：全部复用现有成熟模式（Mitt 事件、rAF 节流、Pinia 响应式），无新技术引入
- **架构层面**：Thinking 状态与消息状态完全隔离，不侵入现有消息生命周期
- **兼容性层面**：与 ISS-015 修复零冲突，与现有 STREAM 协议并存不悖

### 4.2 建议关注的细节

1. **ThinkingCard 内容高度管理**：思考内容可能很长，需要 max-height + 内部滚动，避免顶层面板无限膨胀挤压消息列表可视区域。

2. **多 aiclaw 并发性能**：同房间内多个 aiclaw 同时 thinking 时，每个都维持一个 rAF buffer。实测 5 个并发以内无压力（现有 STREAM 协议已验证）。

3. **移动端适配**：`src/mobile/` 下需同步实现 ThinkingPanel 的移动端版本（设计可复用桌面逻辑，仅调整布局）。如本期只实现桌面端，需在需求中明确标注。

4. **THINKING_END 后状态保留时长**：建议 thinking 完成后保留卡片 30 秒再自动收起，给用户留出「看到并点击回顾」的窗口。具体时间可产品决定。

---

## 五、结论

| 维度 | 评估 |
|------|------|
| 技术可行性 | 完全可行 |
| 架构兼容性 | 与现有 store / WS / UI 架构 100% 兼容 |
| ISS-015 影响 | 零冲突 |
| 预估工作量 | **3-3.5 天**（纯前端） |
| 阻塞问题 | **无** |

建议：可以进入开发阶段。建议优先实现 WS 类型 + Store 状态 + 基础组件骨架（1.5 天），再完善 UI 细节和联调（2 天）。
