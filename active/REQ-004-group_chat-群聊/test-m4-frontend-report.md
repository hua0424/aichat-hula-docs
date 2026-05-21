# REQ-004 M4 前端白盒联调报告

> 测试方：frontend-dev
> 日期：2026-05-21
> 里程碑：M4 端到端集成测试 + Bug 修复
> frontend commit：`53fa48a52`
> 前置 commit：`d56b7e4a4`（M4 API URL 对齐 + 响应格式处理）

---

## 一、联调环境

| 项目 | 值 |
|------|------|
| Gateway | `http://10.38.12.6:18080` |
| WS URL | `ws://10.38.12.6:18080/api/ws/ws?token=...&clientId=...` |
| 测试账号 | Dawn（13275346112/123456，uid=10937855681024） |
| 测试群 | roomId=163347643904512 |
| aiclaw | 安洁 uid=140789091499520, M4TestAI uid=163589881742848 |
| 测试工具 | Python 3 websockets 库 + curl |

---

## 二、联调场景与结果

### 2.1 场景 1：ThinkingStart 广播链路 ✅

**验证方式**：Python WS client 监听 + HTTP 触发 aiclaw thinking

**结果**：
- thinkingStart：收到 2 个并发 aiclaw 的独立 thinkingId ✅
- thinkingDelta：123 个 chunk，seq 1→123，内容完整 ✅
- thinkingEnd：durationMs + status=complete ✅
- WS payload 字段完全匹配 wsType.ts 类型定义 ✅

### 2.2 场景 2：AICLAW_GROUP_CONFIG_UPDATE WS 广播 ✅

**验证方式**：Python WS client 监听 + PUT API 触发配置变更

**步骤**：
1. WS 连接 `/api/ws/ws?token=xxx&clientId=xxx`，发送 type=3 auth
2. HTTP PUT `im/aiclaw/group/config`（rateLimitPerMinute 改为 15）
3. 等待 WS 广播

**结果**：
```
type: "groupConfigChange"
data:
  aiclawUid: 140789091499520
  roomId: 163347643904512
  config:
    rateLimitPerMinute: 15
    dailyLimit: 100
    respondToAi: 1        ← server 返回 integer
    mentionRequired: 0     ← server 返回 integer
```
- PUT 成功（`success: true`）✅
- WS 广播在 1s 内到达 ✅
- 恢复原配置（rate=5）成功 ✅

### 2.3 场景 3：autoReply extra 字段透传 ✅（server 端确认）

**验证方式**：backend-tester 跑限流场景，确认 server 发送格式

**server 端确认**（backend-tester）：
```
body={content="发言受限：发言频率限制，已自动跳过本次响应"}
extra={autoReply=true}
```

**前端代码链路确认**：
1. `layout/index.vue`：检测 `data.message.extra?.autoReply === true` ✅
2. `chat.ts`：`markMessageAsAutoReply(msgId)` 存入 reactive Set ✅
3. `ChatMain.vue`：`v-if="chatStore.isAutoReplyMessage(item.message.id)"` 渲染标签 ✅
4. i18n：zh-CN "自动回复" / en "Auto-reply" ✅

---

## 三、发现的 Bug 与修复

### 3.1 Bug M4-FE-1：WS adapter 层遗漏 REQ-004 事件映射（P0）

**严重性**：P0 — M2/M3 的 thinking + groupConfigChange 功能完全失效

**现象**：
- WS 收到 thinkingStart/Delta/End、groupConfigChange 事件
- 但 adapter 不分发到 Mitt event bus
- layout/index.vue 的 handler 永远不会触发

**根因**：三层 WS adapter 均未添加 REQ-004 新增的 4 个事件类型映射

**影响范围**：
- Web 模式（webSocketWeb.ts）：switch 缺 4 个 case
- Tauri 桌面（webSocketRust.ts）：listener 缺 4 个
- Rust 后端（client.rs）：match arm 缺 4 个

**修复**（commit `53fa48a52`）：

| 文件 | 新增映射 |
|------|---------|
| `webSocketWeb.ts` | `thinkingStart` → THINKING_START |
| `webSocketWeb.ts` | `thinkingDelta` → THINKING_DELTA |
| `webSocketWeb.ts` | `thinkingEnd` → THINKING_END |
| `webSocketWeb.ts` | `groupConfigChange` → AICLAW_GROUP_CONFIG_UPDATE |
| `webSocketRust.ts` | `ws-thinking-start` → THINKING_START |
| `webSocketRust.ts` | `ws-thinking-delta` → THINKING_DELTA |
| `webSocketRust.ts` | `ws-thinking-end` → THINKING_END |
| `webSocketRust.ts` | `ws-group-config-change` → AICLAW_GROUP_CONFIG_UPDATE |
| `client.rs` | `thinkingStart` → emit ws-thinking-start |
| `client.rs` | `thinkingDelta` → emit ws-thinking-delta |
| `client.rs` | `thinkingEnd` → emit ws-thinking-end |
| `client.rs` | `groupConfigChange` → emit ws-group-config-change |

### 3.2 Bug M4-FE-2：updateAiclawGroupConfig 未做 integer→boolean 转换（P1）

**现象**：WS 广播 `respondToAi=1`（integer）直接存入 store Map，n-switch 组件期望 boolean

**修复**：和 `loadAiclawGroupConfigs` 一致，做 `=== true || === 1` 转换
```typescript
const raw = config as Record<string, unknown>
const normalized: AiclawGroupConfig = {
  respondToAi: raw.respondToAi === true || raw.respondToAi === 1,
  mentionRequired: raw.mentionRequired === true || raw.mentionRequired === 1,
  // ...
}
```

### 3.3 前置修复（commit `d56b7e4a4`，M4 早期）

| Bug | 描述 |
|-----|------|
| API URL 不匹配 | 前端用动态路径 `im/aiclaw/{uid}/group-config/list`，实际 API 是 `im/aiclaw/group/config` + 查询参数 |
| autoReply 检测路径 | 权威位置从 `(data as any).extra` 改为 `data.message.extra` |
| 非列表 API | 设计假设有 list 接口，实际是单条查询，改为遍历 roomIds 逐个获取 |

---

## 四、经验与教训

### 4.1 WS adapter 三层同步维护

**问题**：新增 WS 事件类型时，必须同步修改 3 处（webSocketWeb.ts、webSocketRust.ts、client.rs），任一遗漏导致该平台功能完全失效。

**教训**：
- WS adapter 是横切关注点，必须和 wsType.ts 枚举严格同步
- PR review 时必须检查：wsType.ts 新增枚举 → 3 层 adapter 是否都有映射
- 建议后续增加自动化检查：遍历 WsResponseMessageType 枚举，验证每个值至少在一个 adapter 中有映射

### 4.2 server integer boolean 与 frontend boolean

**问题**：server 返回 `respondToAi=1`/`mentionRequired=0`（integer），frontend TypeScript 类型声明为 `boolean`。

**教训**：
- 凡是 server 返回的 "boolean" 字段，在 store 层统一做 `=== true || === 1` 转换
- WS 广播和 HTTP API 响应都可能出现这个问题
- 转换代码需要放在 Mitt handler（WS）和 imRequest callback（HTTP）两个路径中

### 4.3 端到端联调不可替代

**问题**：WS adapter 遗漏在以下检查中都不可见：
- 后端 API 测试（事件 server 确实发了）
- UI 静态分析（layout handler 写了但收不到事件）
- Python WS client（直接读 payload，绕过 adapter）

**教训**：只有真实 Tauri/Web 运行 + 触发事件才能发现 adapter 层 bug。M4 的"协议链路"白盒联调价值远超单元测试。

---

## 五、待办

| 项目 | 状态 | 备注 |
|------|------|------|
| ui-tester GUI 回归测试 | 待完成 | 需要 pull `53fa48a52` 重新构建 Tauri |
| autoReply GUI 实际触发 | 待确认 | server 确认字段存在，等 ui-tester 验证 UI 标签 |
| `data.extra` fallback 清理 | 后续 cleanup | autoReply 检测的兼容降级路径可在下个迭代移除 |
| video call WS 事件映射 | 不在本迭代 | ScreenSharing/NetworkPoor/MediaControl 等 6 个既有遗漏 |

---

## 六、修改文件清单

| 文件 | 变更 |
|------|------|
| `src/services/webSocketWeb.ts` | +4 switch case（thinkingStart/Delta/End, groupConfigChange） |
| `src/services/webSocketRust.ts` | +4 Tauri listener（ws-thinking-start/delta/end, ws-group-config-change） |
| `src-tauri/src/websocket/client.rs` | +4 match arm（thinkingStart/Delta/End, groupConfigChange） |
| `src/stores/chat.ts` | 修复 updateAiclawGroupConfig integer→boolean 转换 |
| `src/layout/index.vue` | （前置修复）autoReply 检测路径 + AICLAW_GROUP_CONFIG_UPDATE handler |
| `src/utils/webImRequest.ts` | （前置修复）group config API URL 对齐 |
| `src-tauri/src/im_request_client.rs` | （前置修复）Rust URL 映射同步 |
| `src/enums/index.ts` | （前置修复）新增 group config enum entries |
