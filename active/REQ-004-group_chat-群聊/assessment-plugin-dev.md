# REQ-004 可行性评估 — plugins 侧（aichat-node + aichat-claw）

> 评估人：plugin-dev
> 日期：2026-05-20
> 版本：基于需求.md v2.1 + review-v2.0-reviewer.md
> 结论：**技术上可行，无阻塞问题，但 aichat-node 防循环实现存在架构复杂度**

---

## 一、总体评估

| 维度 | 评估 | 说明 |
|------|------|------|
| 技术可行性 | ✅ 可行 | 无阻塞技术问题 |
| 与现有代码兼容性 | ⚠️ 需适配 | `ClawAdapter` 接口需扩展，MessageHandler 需重构 |
| 数据模型兼容性 | ✅ 兼容 | `im_message` 零变更，新表独立 |
| 防循环复杂度 | ⚠️ 较高 | aichat-node 作为无状态 CLI，持久化限流状态需要设计决策 |
| 预估工作量 | **9 人天** | plugins 侧（不含 server/frontend） |

---

## 二、Agent Loop 模型与 openclaw adapter 兼容性

### 2.1 当前模型 vs Agent Loop 模型

**当前**（`packages/node/src/handler/message.ts:105`）：
```
收到消息 → adapter.chat() → onChunk(STREAM_DELTA) → onDone(STREAM_END)
```
AI 的回复内容通过 `assistant` stream 直接作为聊天消息发出。

**REQ-004 Agent Loop**：
```
收到消息 → adapter.chat() → thinking 流式输出(THINKING_DELTA)
                         → agent 自主调用 Tool(hula_send_message) 发消息
                         → THINKING_END
```
AI 的"思考"仅用于展示，实际消息由 MCP Tool 独立发送。

### 2.2 兼容性分析

**✅ 兼容的部分：**

1. **openclaw gateway 的事件模型天然支持**。当前 `OpenclawAdapter`（`packages/node/src/claw/openclaw.ts:365-386`）已区分：
   - `stream='assistant'` → delta chunk（可映射为 thinking 内容）
   - `stream='lifecycle'` → end/error（可映射为 THINKING_END）

2. **`hula_send_message` Tool 已存在且可用**。`packages/claw/src/tools/send-message.ts:14-32` 已实现：
   - 参数：`roomId` + `content`
   - 调用 `HulaApiClient.sendMessage()` → `POST /api/im/chat/msg`
   - 返回 `{ ok: true, msgId }`

3. **sessionKey 已包含 roomId 上下文**。当前格式 `aiclaw-{selfUid}-room-{roomId}`，agent 通过 session 可知当前群。

**⚠️ 需要修改的部分：**

1. **`ClawAdapter` 接口语义变更**。当前 `StreamCallbacks`（`packages/node/src/claw/interface.ts`）：
   ```ts
   interface StreamCallbacks {
     onChunk: (chunk: string) => void;   // 当前 = 消息内容
     onDone: (fullContent: string) => void;
     onError: (error: Error) => void;
   }
   ```
   REQ-004 下 `onChunk` 的含义变为"thinking delta"，不再直接产生聊天消息。需要：
   - **方案 A**：新增 `ThinkingCallbacks` 接口（`onThinkingStart/Delta/End`），保持 `StreamCallbacks` 不变供未来使用
   - **方案 B**：重命名现有 callbacks 为 thinking 语义
   - **推荐方案 A**，因为 §3.1 明确"现有 STREAM 协议保留不动"

2. **`MessageHandler.sendToAI()` 重构**。当前（`message.ts:105-160`）：
   - 调用 `adapter.chat()` → 收到 chunk 直接 `STREAM_DELTA`
   - 收到 done 直接 `STREAM_END`
   REQ-004 下：
   - 调用 `adapter.chat()` → 收到 chunk 转发 `THINKING_DELTA`
   - 收到 done 转发 `THINKING_END`
   - **不再发送 STREAM 事件**（除非保留双协议）
   - Tool 发送的消息独立走 REST API，aichat-node 不感知

3. **aichat-claw 的 `HulaApiClient` Token 来源**。当前（`packages/claw/src/index.ts:19-22`）：
   ```ts
   const aiclawToken = config.hula?.aiclawToken || '';
   const hulaApi = new HulaApiClient(serverUrl, aiclawToken);
   ```
   当前使用单一全局 token。群聊场景下每个 aiclaw 有独立 token，需要确认：
   - openclaw 在执行 Tool 时是否将 agent 的 credential 注入 context？
   - 若否，aichat-claw 需要为每个 aiclaw 维护独立的 `HulaApiClient` 实例
   - **风险等级：中**。需要 server-dev / openclaw 侧确认 Tool context 传递机制。

---

## 三、THINKING Stream 上下文管理

### 3.1 当前实现

`MessageHandler` 使用简单的布尔状态（`message.ts:25`）：
```ts
private streaming = false;           // 是否正在流式回复
private pendingMessages: string[];   // 流式期间排队消息
```

### 3.2 REQ-004 下的挑战

**并发场景**：同一群内有多个 aiclaw，或多个用户同时发消息触发不同 aiclaw 的思考。

**需求**：
- 每个 thinking 会话独立跟踪（start → delta → end）
- 同一 aiclaw 同一时间只应有一个 active thinking（但不同 aiclaw 可并行）
- thinking 与消息发送（Tool 调用）是并行的

### 3.3 实现方案

将 `streaming: boolean` 替换为：
```ts
interface ThinkingSession {
  sessionKey: string;        // aiclaw-{uid}-room-{roomId}
  msgId: string;             // 触发消息的 msgId
  startTime: number;
  seq: number;
}
private thinkingSessions = new Map<string, ThinkingSession>();
```

**变更点：**
- `sendToAI()` 前检查 `thinkingSessions.has(sessionKey)`，避免重复触发
- `onThinkingStart` 时创建 session，发送 `THINKING_START`
- `onChunk` 时发送 `THINKING_DELTA`
- `onDone/onError` 时删除 session，发送 `THINKING_END`
- `pendingMessages` 队列保持，但入队条件改为 `thinkingSessions.size > 0`

**与现有 activeStreams 冲突？** 无冲突。REQ-004 的 THINKING 事件（20/21/22）与现有 STREAM 事件（17/18/19）是独立协议。§3.1 明确"STREAM 协议保留不动"。两者的状态机互不干扰。

**复杂度：中等**。主要是将单 session 布尔状态改为 Map 管理。

---

## 四、防循环限制在 aichat-node 层的复杂度

### 4.1 需求汇总（§2.4）

| 规则 | 默认值 | 检查时机 | 实现难点 |
|------|--------|----------|----------|
| 频率限制 | 10 条/分钟 | 调用 agent 前 | 中：需要滑动窗口计数器 |
| 每日上限 | 1000 条 | 调用 agent 前 | **高**：需要跨进程/跨实例持久化 |
| AI 互触发 | true | 调用 agent 前 | 低：检查 sender userType |
| 短回复跳过 | 最近 3 条均 <10 字符 | agent 决定发送时 | 中：需要跟踪最近回复长度 |
| 指数退避 | 0s→+5s→+15s→+30s | 投递前附加延迟 | **高**：延迟施加点与 Tool 调用路径不重合 |

### 4.2 核心问题：aichat-node 是无状态 CLI

当前 aichat-node 的设计假设：
- 单进程、无外部依赖（无 Redis、无 DB）
- 限流/配额状态若仅存内存，重启即丢失
- 若用户运行多个 aichat-node 实例（如多设备），状态不共享

**每日上限 1000 条**：
- 纯内存实现：重启后配额重置，可被绕过
- 文件持久化（`~/.aichat/rate-limit.json`）：单实例可用，多实例不共享
- Redis：需要新增依赖和配置
- **推荐**：若 server 侧已有 `im_aiclaw_group_config` 和消息落库，**server 侧做配额检查更可靠**（通过每日消息计数 SQL）

**指数退避"投递前附加延迟"**：
REQ-004 原文："AI-to-AI 退避由 aichat-node 在投递前附加延迟"

但实际的"投递"有两种路径：
1. **Tool 调用路径**：agent → `hula_send_message` → aichat-claw → REST API → server
   - aichat-node **不在此路径上**，无法控制延迟
2. **触发路径**：收到消息 → aichat-node 决定是否调用 `adapter.chat()`
   - 延迟可在此施加（收到消息后等待 N 秒再触发 agent）

**解读**："投递前"应指"触发 agent loop 前"，即收到 AI 消息后延迟再调用 `adapter.chat()`。这需要：
- 收到消息后先检查是否为 AI 消息
- 若是，检查该群连续 AI-to-AI 轮数
- 根据轮数计算延迟，用 `setTimeout` 推迟 `adapter.chat()` 调用
- 人类消息到来时重置计数器

**实现复杂度：中高**。需要：
- 维护每个群的 `aiRoundCounter: { count: number, lastAiUid: number }`
- 维护每个群的 `lastMessageFromHuman: boolean` 用于重置
- 延迟期间新消息到达的并发处理（延迟队列管理）

### 4.3 autoReply 标记（reviewer I1）

限流触发时发送的说明消息需要标记 `autoReply: true`，其他 aiclaw 跳过。

实现：
- `hula_send_message` Tool 增加可选参数 `extra?: Record<string, unknown>`
- 限流说明消息通过 Tool 发送时传入 `{ autoReply: true }`
- aichat-node 收到消息后检查 `message.body.urlContentMap?.autoReply`，为 true 则跳过 agent loop
- **复杂度：低**

### 4.4 建议：防循环分层

考虑到 aichat-node 的无状态特性，建议将防循环**分层实现**：

| 层级 | 负责规则 | 原因 |
|------|----------|------|
| aichat-node（轻量） | AI 互触发开关、短回复跳过、autoReply 跳过 | 无需持久化，纯内存逻辑 |
| HuLa-Server（权威） | 频率限制、每日上限 | 有 DB 持久化，可精确统计 |
| aichat-node（附加） | 指数退避延迟 | 纯内存计数器，失效可接受 |

若 manager 坚持全部由 aichat-node 实现，需要引入文件/Redis 持久化机制，增加约 **2 人天**工作量。

---

## 五、预估工作量

### 5.1 aichat-node（~6 人天）

| 任务 | 天数 | 说明 |
|------|------|------|
| WS 协议扩展：THINKING_START/DELTA/END | 0.5 | 在 `protocol.ts` 新增类型，与 server-dev 对齐 payload 格式 |
| `ClawAdapter` 接口扩展 | 0.5 | 新增 `ThinkingCallbacks`，保持 `StreamCallbacks` 兼容 |
| `MessageHandler` 重构：thinking 会话管理 | 1.5 | 替换 `streaming` 布尔为 Map，处理并发 thinking |
| 防循环：AI 互触发 + 短回复 + autoReply | 1.0 | 纯内存逻辑，相对直接 |
| 防循环：频率/日限（若全放 node） | 1.5 | 含持久化设计，若分层则减至 0.5 |
| 防循环：指数退避 | 1.0 | 延迟队列 + 计数器管理 |
| 群配置 WS 通知处理 | 0.5 | 接收配置更新通知，刷新本地缓存 |
| aichat-cli 扩展 | 0.5 | `group-config` 命令 |
| 集成测试 | 1.0 | 与 server 联调 |

### 5.2 aichat-claw + openclaw adapter（~2 人天）

| 任务 | 天数 | 说明 |
|------|------|------|
| `OpenclawAdapter` 支持 thinking 事件 | 1.0 | 将 assistant stream 映射为 thinking callbacks |
| `hula_send_message` Tool 增强 | 0.5 | 支持 `extra` 字段（autoReply 标记）、验证群聊 roomId |
| aichat-claw Token 管理（若需） | 0.5 | 确认 openclaw Tool context 机制 |

### 5.3 总计

- **plugins 侧总计：8 人天**（若防循环全放 node + 有持久化）
- **plugins 侧总计：6.5 人天**（若防循环分层，server 负责频率/日限）

建议按 **8 人天** 预留缓冲。

---

## 六、风险与建议

### 6.1 风险清单

| 风险 | 等级 | 说明 | 缓解 |
|------|------|------|------|
| R1：Tool Token 上下文传递 | 中 | aichat-claw 是否知道当前 aiclaw 的 token | 与 server-dev 确认 openclaw Tool execution context |
| R2：aichat-node 多实例限流 | 中 | 多设备运行 aichat-node 时限流状态不共享 | 建议 server 侧做权威配额检查 |
| R3：指数退避施加点歧义 | 低 | "投递前" vs "触发前"的理解差异 | 与 manager 确认：延迟施加在调用 adapter.chat() 前 |
| R4：thinking 与 Tool 消息时序 | 低 | THINKING_END 可能在 Tool 消息发送后到达 | 前端需支持 thinking 条与消息气泡并行展示（§5.1 已设计） |

### 6.2 关键建议

1. **防循环分层**：建议频率限制和每日上限由 HuLa-Server 做权威检查（基于 `im_message` 表每日计数），aichat-node 做轻量前置检查。这样 aichat-node 保持无状态，server 侧已有 DB 可精确统计。

2. **ClawAdapter 接口**：采用方案 A（新增 `ThinkingCallbacks`，保留 `StreamCallbacks`），为未来双协议并行留余地。

3. **aichat-claw Token**：优先确认 openclaw 是否在 Tool execute context 中提供 agent credential。若提供，可直接使用；若否，aichat-claw 需改为按 aiclaw 维护 `HulaApiClient` 实例池。

4. **开发顺序**：
   - Phase 1：WS 协议 + adapter 接口扩展（1 天）
   - Phase 2：MessageHandler thinking 重构（2 天）
   - Phase 3：防循环逻辑（2 天）
   - Phase 4：aichat-claw Tool 增强（1 天）
   - Phase 5：集成测试（2 天）

---

## 七、阻塞问题结论

**无阻塞问题。**

所有需求在技术上均可实现。最大的复杂度来自 aichat-node 的防循环持久化状态管理，但这可以通过**分层设计**（server 侧做权威配额）来降低。若 manager 坚持全部由 aichat-node 实现，需要引入文件/Redis 持久化，工作量从 6.5 天增至 8 天。
