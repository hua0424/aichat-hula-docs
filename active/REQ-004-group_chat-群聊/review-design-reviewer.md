# REQ-004 详细设计评审 — reviewer 反馈

> 评审人：reviewer
> 日期：2026-05-20
> 评审对象：
> - `design-server.md` v1.1 (server-dev, c0792a7)
> - `design-plugin.md` v1.1 (plugin-dev, 1f4b54b)
> - `design-frontend.md` v1.1 (frontend-dev, caf3343)
> - `design-tasks.md` (manager, 35d7bc3)
> 结论：**有条件通过**，需处理 1 个 Must-fix + 5 个 Should-fix + 6 个 Info

---

## 总评

三份设计文档整体质量高，架构清晰、代码骨架详尽、跨组对齐项 X1-X6 均有明确结论。Agent Loop 模型在三方的实现路径上一致：plugin 触发 → openclaw agent → thinking 事件 → MCP Tool 发消息 → server 落库推送 → 前端展示。im_message 零改动的硬约束在三份文档中被严格执行。ISS-015 兼容性在前端设计中有 10 项零交互核查表，充分。

以下是逐项评审结果。

---

## 跨组一致性验证（X1-X6）

### X1: THINKING WS payload — ⚠️ 有一处不一致

三方对 payload 字段基本对齐，但 `thinkingId` 的来源描述存在矛盾：

| 文档 | thinkingId 描述 |
|------|----------------|
| **server §3.3.2** | "server 生成的 thinking ID（THINKING_START 创建后回传）" |
| **plugin §A.2** | "server 生成的 thinking 记录 ID（START 后由 server 返回）" |
| **plugin §F.2** | "thinkingId 由 server 在 START 时返回" |
| **frontend §3.1** | "由 **aichat-node 生成**，全局唯一" ← ❌ |
| **frontend X1** | "thinkingId 由 server 生成（Long 自增主键 → String）" ← ✅ |

**问题**：frontend §3.1 注释写"由 aichat-node 生成"，但同一文档的 X1 结论和 server/plugin 文档均确认"由 server 生成"。

**结论**：frontend §3.1 的注释是笔误，X1 结论正确。需要修正注释。

### X2: autoReply — ✅ 三方一致

- server: extra 不落库，WS payload only
- plugin: `hula_send_message` 传递 `extra: { autoReply: true }`
- frontend: `(data as any).extra?.autoReply` 检测 + 内存 Set 标记

三方完全一致。

### X3: 防循环分层 — ✅ 两方一致

- server: Redis 滑动窗口（频率/日限）
- plugin: AntiLoopGuard 内存层（互触发/短回复/退避）
- frontend: 不参与防循环

分层清晰，职责明确。

### X4: aichat-claw Token — ⚠️ 开放项

plugin-dev 列出方案 A（openclaw context）+ 方案 B（实例池），待 openclaw 行为验证。这是合理的开放项，有保底方案。

### X5: 群配置 WS 通知 — ⚠️ payload 字段有差异

| 文档 | 格式 | config 嵌套 |
|------|------|------------|
| **server §3.3.3 WSGroupConfigChange** | 扁平结构（rateLimitPerMinute/dailyLimit 等直接在顶层） | 无嵌套 |
| **plugin §F.3 GroupConfigUpdateDTO** | 嵌套结构（config: { rateLimitPerMinute, ... }） | 有 `config` 嵌套 |
| **frontend §3.1 AiclawGroupConfigUpdatePayload** | 嵌套结构（config: AiclawGroupConfig） | 有 `config` 嵌套 |

**问题**：server 用扁平结构，plugin 和 frontend 用嵌套结构（`config: {}` 包裹）。开发时会出现字段路径不一致（如 server 的 `payload.rateLimitPerMinute` vs plugin 的 `payload.config.rateLimitPerMinute`）。

**Must-fix #1**：统一 payload 格式。建议 server 改为嵌套结构（与 plugin + frontend 一致），因为嵌套结构语义更清晰（config 对象完整替换）。

### X6: 上下文窗口 — ✅ 已关闭

openclaw session-memory hook 已启用，sessionKey 自带 history。三方确认不需要 server API。

---

## Must-fix

### M1: X5 群配置 WS 通知 payload 格式不一致（见上方 X5 分析）

server `WSGroupConfigChange` 是扁平结构，plugin/frontend 是嵌套结构。三方必须统一。

**建议**：server 改为嵌套结构：
```java
@Data
public class WSGroupConfigChange {
    private Long aiclawUid;
    private Long roomId;
    private ConfigDTO config;  // 嵌套对象

    @Data
    public static class ConfigDTO {
        private Integer rateLimitPerMinute;
        private Integer mentionRequired;
        private Integer dailyLimit;
        private Integer respondToAi;
    }
}
```

---

## Should-fix

### S1: server §3.3.2 — THINKING_START 的 thinkingId 回传机制未完整定义

server 设计描述了 THINKING_START 时"创建 thinking 记录（落库）"和"广播 START 到群成员"，但**thinkingId 的回传机制**不明确：

1. plugin 发送 `THINKING_START(20)` → server 创建记录 → 生成 thinkingId
2. server 需要把 thinkingId 返回给 plugin（用于后续 DELTA/END 关联）
3. 但 WS 协议是 server→client 推送模式，**如何回传给发起方 plugin？**

当前 WSReqTypeEnum 20 是 plugin→server 单向，没有定义 server→plugin 的响应通道。

**建议**：
- 方案 A（推荐）：在 server 的 THINKING_START 处理中，**将 thinkingId 写入广播 payload**。plugin 收到广播后，通过 `fromUid === selfUid` 匹配自己的 thinking session，提取 thinkingId。
- 方案 B：复用现有 WS 双向通信机制（如果 server 端 WebSocket 支持回复）。

需要在设计中明确这一机制。plugin §F.2 已写了 `→ server 返回: { thinkingId, ... }`，但 server 端没有描述对应的实现。

### S2: server §3.4 — 防循环频率限制在发送时校验，不在 THINKING 时校验

server 设计将频率限制放在"发言时"（aichat-node 调用 REST API 发消息时返回 429），但此时 agent loop 已经在运行（消耗 GPU/CPU 资源）。

更高效的方案是在 **THINKING_START 时就校验频率**——如果已超限，直接返回错误，避免启动无意义的 agent loop。

**建议**：server 在 THINKING_START 处理时增加限流检查，超限时返回错误事件（如 `THINKING_END { error: "rate_limit_exceeded" }`），plugin 收到后不再触发 adapter.chat()。

### S3: plugin §A.2 — thinkingSessions Map 可能内存泄漏

`thinkingSessions` 在 `onThinkingEnd` 时删除条目，但如果 openclaw agent 长时间不结束（如卡死、网络断连），thinkingSessions 会持续积累。

**建议**：增加 thinking session 超时机制（如 5 分钟），超时后自动发送 THINKING_END(error) 并清理。类似 server 端 StreamProcessor 的 30s 超时机制。

### S4: plugin §E.2 — AntiLoopGuard 的 aiRoundCount 跨实例问题

AntiLoopGuard 维护在 aichat-node 单实例内存中。如果同一 aiclaw 的 aichat-node 有多个实例（水平扩展），各实例的 `aiRoundCount` 不同步，退避效果减弱。

**评估**：当前 aichat-node 可能是单实例部署（每个 aiclaw 一个 node 进程），但如果未来需要多实例，这里需要 Redis 同步。

**建议**：在设计中加一条注释标注此限制。如果确认单实例部署，可以忽略。

### S5: server §3.1.2 — im_aiclaw_thinking 缺少 thinkingId → msg_id 的回写机制

当 MCP Tool 发送消息后，需要建立 thinking → msg 的关联（写入 `im_aiclaw_thinking_msg_rel`）。但 server 端设计只描述了"thinking 事件处理"，没有描述关联回写的触发点。

**关键问题**：谁负责写 `im_aiclaw_thinking_msg_rel`？
- MCP Tool 发消息时，plugin 端不知道 thinkingId（或者知道？）
- Server 端在收到消息时不知道 thinkingId

**建议**：明确关联建立的路径：
1. plugin 发送消息时在 REST API body 中携带 `thinkingId`
2. Server 收到后同时写入 `im_aiclaw_thinking_msg_rel` 和更新 `im_aiclaw_thinking.has_response = 1`
3. 在 server 设计中补充这一流程

---

## Info

### I1: server §3.2.3 — aiclaw token 鉴权的 Token 来源

server 写"connectionToken（activate 流程获取）即为 aiclaw token"。但需要确认：connectionToken 是否有有效期？过期后如何刷新？如果 aiclaw 重启后 token 失效，aichat-node 需要重新 activate。

### I2: plugin §A.4 — userType 判断 aiclaw 的值

plugin 写 `data.fromUser.userType === 2`（机器人），但 `UserTypeEnum` 中 `AICLAW = 4`。如果 aiclaw 的 userType 是 4，则判断条件应为 `=== 4`。需确认 openclaw gateway 返回的 userType 是否映射为 2。

### I3: frontend §3.6 — autoReply 消息的 extra 字段获取路径

frontend 从 `(data as any).extra?.autoReply` 获取。需确认 server 推送的 receiveMessage WS payload 中，extra 字段是放在 `data` 顶层还是 `data.message` 内。server §3.3.4 写"WS payload 携带 extra"，但未明确 payload 结构层级。

### I4: server §3.5.1 — Redis Key 命名冲突风险

`im:aiclaw:rate:{aiclawUid}:{roomId}:{yyyyMMddHHmm}` — 如果 roomId 是 Long，转为字符串后可能与 uid 混淆。建议用更明确的分隔符或前缀。

### I5: server §六 风险 — "Liquibase 支持回滚"与 §3.1.4 "不使用 Liquibase / Flyway"矛盾

§3.1.4 明确"不使用 Liquibase / Flyway"，但 §六风险表中写"Liquibase 支持回滚"。需要统一描述。

### I6: design-tasks.md 时间线 — 设计初稿提交日期已提前完成

design-tasks.md 写"设计初稿提交 05-25（周一）"，但实际 05-20（当天）已全部提交并完成跨组对齐。时间线可更新为实际进度。

---

## 评审总结

### 跨组一致性

| 编号 | 议题 | 状态 | 备注 |
|------|------|------|------|
| X1 | THINKING payload 字段 | ✅ 基本一致 | frontend §3.1 注释笔误需修正 |
| X2 | autoReply 载体 | ✅ 三方一致 | — |
| X3 | 防循环分层 | ✅ 两方一致 | — |
| X4 | aichat-claw Token | ⚠️ 开放项 | 有方案 B 保底，不阻塞 |
| X5 | 群配置 WS payload | ❌ 不一致 | **Must-fix M1**：server 扁平 vs plugin/frontend 嵌套 |
| X6 | 上下文窗口 | ✅ 已关闭 | — |

### 问题清单

| 级别 | 编号 | 内容 | 阻塞? |
|------|------|------|-------|
| **Must-fix** | M1 | X5 群配置 WS 通知 payload 格式不一致（server 扁平 vs plugin/frontend 嵌套） | 是 |
| Should-fix | S1 | thinkingId 回传机制未在 server 端明确（THINKING_START 时如何将 ID 返回给 plugin） | 否 |
| Should-fix | S2 | 频率限制应在 THINKING_START 时校验，而非仅在校验时 | 否 |
| Should-fix | S3 | thinkingSessions 需增加超时清理机制 | 否 |
| Should-fix | S4 | AntiLoopGuard 内存状态在多实例部署时不同步（标注限制即可） | 否 |
| Should-fix | S5 | thinking_msg_rel 关联回写路径未在设计中文档化 | 否 |
| Info | I1 | aiclaw token 有效期和刷新机制 | 否 |
| Info | I2 | userType 判断值 2 vs 4（AICLAW）需确认 | 否 |
| Info | I3 | autoReply extra 在 WS payload 中的层级位置需明确 | 否 |
| Info | I4 | Redis Key 命名中的分隔符建议 | 否 |
| Info | I5 | server §六 Liquibase 描述与 §3.1.4 矛盾 | 否 |
| Info | I6 | design-tasks 时间线可更新 | 否 |

### ISS-015 兼容性

frontend §3.2.5 的 10 项零交互核查表覆盖了所有关键状态（`chatMessageList`、`messageMap`、`streamingMessages`、`pendingStreamReplace`、`pushMsg`、`normalizeMsgSendTime`、`clearRedundantMessages`、`startStream/appendStreamContent/finalizeStream`），验证充分。thinking 的 4 个新 action 独立实现，不调用任何现有消息处理函数。**ISS-015 兼容性无风险。**

### 数据库设计评估

三张新表 DDL 设计合理：
- `im_aiclaw_group_config`：UNIQUE KEY 保证一对一，字段类型 UNSIGNED 防负数，idx_room_id 支持按群查询
- `im_aiclaw_thinking`：TEXT 类型无长度限制，idx_has_response_create 支持过期清理高效查询
- `im_aiclaw_thinking_msg_rel`：联合主键避免重复，无自增 ID 减少开销
- 无外键约束，不影响 im_message 写入性能
- 字符集 utf8mb4 与现有表一致

**S5 需补充**：thinking_msg_rel 的写入触发点（见 Should-fix S5）。

---

**结论：M1（X5 payload 格式）修复后可进入任务分发阶段。** S1-S5 建议在设计文档中补充说明，但不阻塞任务分发。三份文档整体设计质量优秀，开发可行。
