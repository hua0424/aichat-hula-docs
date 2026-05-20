# REQ-004 v2.0 评审 — reviewer 反馈

> 评审人：reviewer
> 日期：2026-05-20
> 评审对象：[需求.md](需求.md) v2.0（commit 6d63c2a）
> 结论：**通过，附 2 个 should-fix + 3 个 info**

---

## 总评

v2.0 的 Agent Loop 模型比 v1.1 的两阶段决策更自然——符合 openclaw agent 的实际执行模型（思考 → Tool Call → 继续思考 → ... → 结束），且支持一次 thinking 产生多条回复（1:N 关联），架构上更合理。

对照现有代码验证了关键路径的可行性：
- `hula_send_message` Tool 已有 REST API 实现（`aichat-claw/src/hula-api.ts` → `POST /api/im/chat/msg`），可直接复用
- openclaw adapter 的事件驱动模型（`stream='assistant'` → deltas, `stream='lifecycle'` → completion）可自然映射为 THINKING 事件
- MsgSendConsumer 已支持 aiclaw 消息路由（通过 `findAiclawUid` + `fillAiclawExt`），群聊场景可复用

im_aiclaw_thinking 独立表 + 多对多关联表（`im_aiclaw_thinking_msg_rel`）的设计合理，MessageTypeEnum 不变，现有查询零影响。这些都是好的。

以下是发现的问题。

---

## Should-fix

### S1：§5.1 与 §2.3.3 矛盾——"流式输出" vs "一次性出现"

§2.3.1 图和 §2.3.3 明确 MCP Tool 发送的消息**一次性投递完整消息**：

```
│  调用 MCP Tool:     │──→ plugin → 直接发送完整消息
│  hula_send_message  │    → 前端：正常消息气泡，一次性出现
```

但 §5.1 写的是：

> agent 发送的正式消息正常出现在消息列表中（标准聊天气泡，**流式输出**）

"流式输出"与"一次性出现"是互斥的。开发人员看到这两处会产生歧义。

**建议**：§5.1 改为"标准聊天气泡，完整消息一次性出现"，与 §2.3.3 保持一致。

### S2：respond_to_ai=true + 移除安全网，循环风险较高

v2.0 将 `respond_to_ai` 默认改为 `true`，同时移除了 v1.1 的深度检测和 AI 占比检测。仅保留 rate_limit（10/min）+ daily_limit（1000）+ 短回复跳过。

**风险评估**：两个 respond_to_ai=true 的 aiclaw 在同一群内：
- 用户发一条消息 → A 回复 → B 看到 A 的回复 → B 回复 → A 看到 B 的回复 → ...
- 每个一分钟 10 条 = 两人交替 5 轮/分钟
- daily_limit 1000 = 可持续约 100 分钟的 AI-to-AI 对话
- 期间产生约 1000 条 AI 消息，对群内人类用户极其干扰

"短回复跳过"只在回复 < 10 字符时生效，如果 aiclaw 在做实质性协作讨论（这是 respond_to_ai=true 的目标场景），不会触发。

**建议**：不恢复硬性深度限制（3 轮太短，确实阻碍协作），但增加**指数退避**作为软性安全网：

| 连续 AI-to-AI 轮数 | 触发前延迟 |
|-------------------|-----------|
| 1-5 轮 | 0s（正常） |
| 6-10 轮 | +5s |
| 11-20 轮 | +15s |
| 20+ 轮 | +30s |

- **人类消息中断间自动重置**计数器
- 不硬性阻止对话，但自然减速
- 由 aichat-node 跟踪每个群内连续 AI-to-AI 轮数（维护一个简单的环形缓冲区即可）

这样既保留协作能力，又防止高频 AI 对话淹没群聊。

---

## Info

### I1：限流自动回复可能触发其他 aiclaw 的 agent loop

§2.4 规定"触发限制时，aiclaw 自动回复一条消息说明原因（如'今日发言已达上限，暂停回复'）"。

问题：这条自动回复走正常消息通道（REST API → 入库 → MQ → 推送全员），其他 aiclaw 会收到这条消息。由于 `respond_to_ai=true`，其他 aiclaw 的 agent loop 会被触发，即使原 aiclaw 已经停止。

**极端场景**：群内 5 个 aiclaw，3 个达到 daily_limit，每个触发时都发自动回复 → 每条自动回复又触发另外 4 个 aiclaw → 连锁反应。

**建议**：
- 方案 A：自动回复消息的 `extra` 字段标记 `autoReply: true`，aichat-node 收到后跳过此类消息的 agent loop 触发
- 方案 B：自动回复不走正常消息通道，改用 WS 直接推送（不入库），仅展示给人类用户
- 方案 C：自动回复仅在私聊场景发送，群聊场景静默跳过（更简洁）

### I2：`mention_required` 字段从 schema 中移除但 §2.2 仍提及

§4.1 `im_aiclaw_group_config` 表没有 `mention_required` 字段，§5.2 配置项也没有列出 @ 触发。但 §2.2 仍写"默认不需要 @ 即可触发"，暗示 @ 触发功能存在。

如果 @ 触发功能确实移除（简化本期范围），§2.2 应删除相关描述。
如果保留，schema 需补回 `mention_required` 字段。

### I3：`im_aiclaw_thinking_msg_rel` 缺少 `create_time`

关联表只有 `(thinking_id, msg_id)` 两个字段。虽然通过 `im_aiclaw_thinking.create_time` 可间接获取时间，但如果一次 thinking 产生多条回复，无法区分各条回复的先后顺序（agent loop 中可能先发一条、思考一会、再发一条）。

**建议**：加 `seq INT NOT NULL COMMENT '回复顺序'` 或 `create_time DATETIME(3)` 字段。非阻塞，设计阶段决定。

---

## 评审总结

| 级别 | 编号 | 内容 | 阻塞? |
|------|------|------|-------|
| Should-fix | S1 | §5.1"流式输出"与 §2.3.3"一次性出现"矛盾 | 否 |
| Should-fix | S2 | respond_to_ai=true 缺少安全网，建议增加指数退避 | 否 |
| Info | I1 | 限流自动回复可能触发其他 aiclaw 的 agent loop | 否 |
| Info | I2 | mention_required 从 schema 移除但 §2.2 仍提及 | 否 |
| Info | I3 | thinking_msg_rel 关联表缺少时序字段 | 否 |

**结论：通过。** Agent Loop 模型架构合理，数据模型与现有结构兼容。S1 是文档一致性问题需修正，S2 的指数退避建议不阻塞但强烈推荐——没有安全网的 respond_to_ai=true 在多 aiclaw 群聊中风险较高。
