# REQ-004 需求评审 — reviewer 反馈

> 评审人：reviewer
> 日期：2026-05-20
> 评审对象：[需求.md](需求.md)（commit 5918c57）
> 结论：**有条件通过**，需处理 3 个 must-fix + 5 个 should-fix

---

## 总评

需求方向正确——群聊 + Think/Tool Call 消息机制升级是 REQ-002/003 遗留的自然延伸。架构上复用了现有的三阶段流式协议（STREAM_START/DELTA/END）的设计模式，将其扩展为 THINKING 三阶段，思路清晰。

但文档中存在 **1 个硬性冲突（type 枚举值）** 和 **2 个架构级缺陷（存储模型 + 流式响应退化）**，必须在设计阶段修正。以下是逐项分析。

---

## Must-fix

### M1：消息类型 type=5 与现有 SOUND 枚举冲突

**问题**：§3.3 和 §4.2 提出新增 `type = 5 (THINKING)`，但 `MessageTypeEnum` 中 `type=5` 已被 `SOUND("语音")` 占用：

```java
// MessageTypeEnum.java — 当前值
TEXT(1), RECALL(2), IMG(3), FILE(4), SOUND(5), VIDEO(6), ...
```

如果强行复用 5，已有语音消息的查询、展示、路由逻辑全部会异常。

**建议**：
- 方案 A（推荐）：将 THINKING 放到新编号 `19`（当前最大值为 LOCATION(18)），并在文档中修正
- 方案 B：不使用 im_message 存储思考内容，改用独立表 `im_aiclaw_thinking`（见 M2）

**如果采用方案 B**，im_message 不需要新增 type，彻底避免类型冲突和查询污染。

---

### M2：im_message.content varchar(1024) 无法存储完整思考内容

**问题**：§2.3.3 计划将思考内容落库到 im_message，但现有表结构：

```sql
`content` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL
```

aiclaw 的完整思考过程（chain-of-thought reasoning）通常 2000~10000 字符，远超 1024 限制。即使放 `extra` JSON 字段也有同样的实际长度问题（虽然 JSON 类型无硬限制，但会影响查询性能）。

**更关键的是**：思考记录和聊天消息的生命周期不同——
- 聊天消息是用户间通信，永久保留
- 思考记录是 AI 处理日志，属于运维/审计数据，有清理需求

将两者混在 im_message 中会导致：
1. 所有现有消息查询 API（`/api/chat/msg/list` 等）需要加 `WHERE type != THINKING` 过滤
2. 思考记录的清理（如 N 天后删除）会影响 im_message 表的碎片率
3. 消息统计、未读计数等逻辑全部需要排除 thinking 类型

**建议**：新建独立表 `im_aiclaw_thinking`：

```sql
CREATE TABLE im_aiclaw_thinking (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    aiclaw_uid      BIGINT NOT NULL,
    room_id         BIGINT NOT NULL,
    trigger_msg_id  BIGINT COMMENT '触发本次思考的消息 ID',
    content         TEXT NOT NULL COMMENT '完整思考内容',
    decision        VARCHAR(16) NOT NULL COMMENT 'respond|skip|rate_limited',
    reason          VARCHAR(512) COMMENT '决策原因',
    duration_ms     INT COMMENT '思考耗时(毫秒)',
    response_msg_id BIGINT COMMENT 'decision=respond 时对应的正式消息 ID',
    create_time     DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    INDEX idx_aiclaw_room (aiclaw_uid, room_id),
    INDEX idx_trigger_msg (trigger_msg_id),
    INDEX idx_create_time (create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb44;
```

**好处**：
- im_message 零改动，现有查询逻辑不受影响
- 思考内容用 TEXT 类型，无长度限制
- 独立生命周期，可按 `create_time` 过期清理
- 不影响消息未读计数、消息列表等核心 IM 逻辑
- 彻底解决 M1 的 type 枚举冲突（不需要新 type）

**前端展示**：聊天记录中展示 thinking 时，通过 `trigger_msg_id` 关联查询，或者在 WS 推送 thinking_start/delta/end 时不走 im_message，走独立的 thinking WS 事件。

---

### M3：Think+Tool Call 模式下群聊回复失去流式能力

**问题**：当前 aiclaw 私聊的回复流程是三阶段流式输出：

```
STREAM_START → STREAM_DELTA(n) → STREAM_END → persistStreamMessage
```

前端实时看到逐字输出。

REQ-004 提议的新流程：

```
THINKING_START → THINKING_DELTA(n) → THINKING_END（决策）
    ↓ decision=respond
调用 hula_send_message Tool → aichat-node → HuLa-Server REST API
    → 存储为完整消息 → MQ推送
```

**关键问题**：`hula_send_message` 工具当前实现（`aichat-claw/src/tools/send-message.ts`）是调用 `/api/im/chat/msg` 发送一条完整消息。这意味着 aiclaw 的正式回复从**流式输出退化为一次性完整消息**。

用户在群聊中看到的效果：
- 旧体验：逐字流式输出 → 流畅
- 新体验："AI 正在思考..."（流式思考） → 思考结束 → **整条消息突然出现**

**这是显著的 UX 退化**。

**建议**：

方案 A（推荐）：**Tool Call 触发二次流式输出**
- thinking 结束后，如果 decision=respond，aiclaw 不是调用 REST API 一次发完整消息
- 而是进入第二轮流式输出：复用现有 STREAM_START/DELTA/END 协议
- 即：THINKING 阶段 → 思考展示 → RESPONDING 阶段（复用现有 STREAM 协议）→ 流式输出正式回复
- 最终 STREAM_END 时落库为 type=1(TEXT) 正式消息

方案 B：**Tool Call 支持流式语义**
- 改造 `hula_send_message` 工具，增加 `stream: true` 参数
- 工具内部走 WS STREAM 协议而非 REST API
- aiclaw 的回复内容流式传递

无论哪种方案，核心要求是：**正式回复不能失去流式能力**。

---

## Should-fix

### S1：私聊默认应保证总是回复

§2.3.2 提出私聊也用 Think+Tool Call 模式。架构统一是合理的，但需明确私聊的默认行为约束：

**问题**：如果 aiclaw 在私聊中 decision=skip（决定不回复），用户体验极差——用户直接对 aiclaw 说话，aiclaw 不回复。

**建议**：在需求中明确：
- 私聊场景下，aiclaw 的 system prompt 应包含"对用户的每条消息都必须回复"的指令
- 或在 aichat-node 层面硬编码：私聊模式下若 decision=skip，仍强制触发 respond
- thinking 展示是增强功能（让用户看到 AI 的思考过程），不应成为回复决策的门槛

### S2：im_aiclaw_group_config 缺少索引和默认值考量

**问题**：§4.1 建表语句：

```sql
rate_limit_per_minute INT DEFAULT 0 COMMENT '每分钟发言限制，0=无限制',
```

- `INT` 类型支持负数，应加 `CHECK (rate_limit_per_minute >= 0)` 或在代码层校验
- 缺少 `tenant_id` 字段（如果系统是多租户的，参考现有 im_aiclaw 表的处理）
- 缺少 `deleted` / `is_del` 软删除字段（参考 im_contact 等表的模式）

**建议**：
```sql
rate_limit_per_minute INT UNSIGNED DEFAULT 0 COMMENT '每分钟发言限制，0=无限制',
respond_to_ai TINYINT UNSIGNED DEFAULT 0 COMMENT '是否响应其他 aiclaw 的消息',
is_del TINYINT UNSIGNED DEFAULT 0 COMMENT '是否删除',
```

### S3：防循环机制不够完备

§2.4 的防循环依赖 `respond_to_ai=false` 默认值，但：

**场景**：群内有 aiclaw A（owner-1 设置 respond_to_ai=true）和 aiclaw B（owner-2 也设置 respond_to_ai=true）
- 用户发消息 → A 回复 → B 看到 A 的回复且 respond_to_ai=true → B 回复 → A 看到 B 的回复且 respond_to_ai=true → 循环
- rate_limit_per_minute 限制的是单个 aiclaw 的频率，但不阻止两个 aiclaw 交替回复

**建议**：
- 在服务器端（aichat-node）增加**对话深度检测**：追踪连续 aiclaw-to-aiclaw 回复链深度，超过阈值（如 3 轮）强制中断
- 或者更简单：当检测到最近 N 条群消息都是 aiclaw 发出的时，跳过本轮触发
- 此逻辑应在设计文档中明确，不能仅靠 owner 自觉配置

### S4：WS 事件编号 20/21/22 与 THINKING 命名应与 STREAM 协调

现有 WSReqTypeEnum：
```java
STREAM_START(17), STREAM_DELTA(18), STREAM_END(19)
```

REQ-004 提议：
```
THINKING_START(20), THINKING_DELTA(21), THINKING_END(22)
```

编号连续，没有问题。但结合 M3 的建议（正式回复仍用 STREAM 协议），THINKING 事件只承载思考阶段的展示，职责更清晰。

**建议**：明确 THINKING 事件只负责思考阶段，正式回复仍复用 STREAM 协议。同时考虑是否需要 WSRespTypeEnum 新增 `thinkingStart/thinkingDelta/thinkingEnd` 三个响应类型（当前文档 §3.1 只定义了 plugins → server 方向，server → client 方向用 `thinkingStart` 等字符串类型——需确认 WSRespTypeEnum 是否需要同步新增）。

### S5：aichat-cli 认证机制未定义

§2.5 描述 aichat-cli 通过 HuLa-Server REST API 发送消息，但未说明认证方式。

现有的 `hula_send_message` 工具（在 aichat-claw 中）使用 aiclaw 的已有 WS 会话发送消息。但 CLI 是独立进程，没有 WS 会话。

**建议**：
- CLI 需要一个 API Token 或 session 认证机制
- 可以复用 aiclaw 的 token（存储在 CLI 配置文件中）
- 或者 aichat-node 提供 token 签发接口
- 安全要求：Token 应有权限范围限制（只能发消息，不能执行其他操作）
- 在设计阶段需明确认证方案

---

## 待确认关键问题的回复

### Q1：Think + Tool Call 统一私聊是否影响体验？

**结论：可行但需约束。** 私聊统一为 Think+Tool Call 在架构上简化了实现（一套代码路径），但必须保证：

1. 私聊中 aiclaw 默认总是回复（不应出现 skip）
2. Thinking 展示是"增值体验"而非"回复门槛"
3. 正式回复必须保持流式输出能力（见 M3）
4. Thinking 阶段的延迟需控制——如果 thinking 过长（超过 5s 无输出），前端应显示进度提示

**建议**：在需求文档中增加"私聊行为约束"小节，明确私聊模式下 decision 不能为 skip。

### Q2：思考内容落库粒度？

**结论：每次思考都落库，但建议独立表 + 过期策略。**（详见 M2）

- **落库粒度**：每次触发都记录完整 thinking content
- **存储**：独立 `im_aiclaw_thinking` 表，TEXT 类型
- **过期策略**：
  - `decision=skip` 的记录：保留 30 天后自动清理
  - `decision=respond` 的记录：保留 180 天
  - 通过 `response_msg_id` 可关联到正式消息
- **高频场景**：每条群消息都触发所有 aiclaw 的 thinking。100 人群中有 5 个 aiclaw，每条消息产生 5 条 thinking 记录。日活 1000 条消息 = 5000 条 thinking/天。这在存储上可接受，但建议加监控。

### Q3：默认无限制是否有安全风险？

**结论：有风险，建议调整默认值。**

- `mention_required=false`（默认）→ 每条消息都触发所有 aiclaw → **可接受**（这是群聊 AI 助理的预期行为）
- `rate_limit_per_minute=0`（默认无限制）→ **有风险**：如果 aiclaw 的 prompt 导致它对每条消息都回复，高频群中将严重刷屏
- `respond_to_ai=false`（默认）→ **正确**：防止 aiclaw 互触发

**建议**：
- `rate_limit_per_minute` 默认值改为 `10`（每分钟最多 10 条），而非 0
- 文档中明确说明"无限制"的含义和潜在风险
- 增加**每日总量限制**作为安全兜底（如每日 500 条），即使 rate_limit 设为 0 也生效

### Q4：MCP vs Tool 优先级？

**结论：同意先 openclaw Tool，再 MCP Server。**

- openclaw Tool 是当前核心路径，优先实现
- MCP Server 是扩展层，给未来非 openclaw agent 使用
- 建议在 CLI 设计上做好抽象：CLI 是核心原语，Tool 和 MCP 是两种 transport 封装（文档已正确描述此点）

### Q5：thinking 消息对现有消息查询的影响？

**如果采用 M2 建议（独立表）**：现有查询 API 完全不受影响，无需任何改动。

**如果仍然用 im_message**：
- 必须在所有消息查询中默认过滤 `type != THINKING`
- `/api/chat/msg/list` 应默认不返回 thinking
- 新增参数 `includeThinking=true` 可选返回
- 未读计数、消息统计等需排除 thinking 类型
- **强烈建议不采用此方案**

---

## 额外发现

### E1：群消息分发需考虑上下文窗口

§2.2 提到"aiclaw 收到群内所有消息，积累到自身会话上下文中"。高频群聊中，一天可能有数千条消息。

**建议**：
- 明确上下文窗口策略（最近 N 条？最近 N 小时？token 限制？）
- 群聊上下文构建方式：与私聊的 sessionKey 不同，群聊需要聚合多个用户的发言
- 当上下文超出 token 限制时的截断/摘要策略需在设计阶段定义

### E2：§六实施依赖 — REQ-003 Tool 基础设施状态

文档写"待开发"，但实际上 aichat-claw 中已有 `hula_send_message` 和 `hula_find_friend` 两个工具的基础实现。需要明确：

- 现有工具实现是否可以直接复用？
- 还是需要重构为 aichat-cli 独立工具？
- 两者之间的关系是什么？（CLI 调 REST API vs Tool 走 WS）

### E3：THINKING WS 事件缺少 fromUid/roomId 关联

§3.2 的 THINKING_START payload 包含 `fromUid` 和 `roomId`，但 `THINKING_DELTA` 只有 `chunk` 和 `seq`：

```json
{ "type": 21, "data": { "chunk": "正在分析...", "seq": 1 } }
```

服务端 StreamProcessor 需要通过 `aiclawUid` 关联活跃 stream context 来路由 delta 到正确的用户。当前 STREAM 协议也是这样做的，但需确认 aichat-node 侧的 THINKING stream context 管理是否需要独立的状态映射（因为 thinking 和 streaming 可能在同一个 aiclaw 上同时发生）。

### E4：群级配置修改的实时生效

§2.4 提到"plugins/aichat-node 在调用 openclaw 前检查群配置"。但配置修改后如何实时生效？

- 如果 aichat-node 启动时缓存配置 → 修改后需通知缓存失效
- 如果每次都查数据库 → 性能问题
- 建议使用 Redis 缓存 + MQ/WS 通知失效

---

## 评审总结

| 级别 | 编号 | 内容 | 阻塞? |
|------|------|------|-------|
| **Must-fix** | M1 | `type=5` 与 `SOUND(5)` 枚举冲突，需换编号或用独立表 | 是 |
| **Must-fix** | M2 | `varchar(1024)` 无法存储完整思考内容，且混入 im_message 污染查询；建议独立 `im_aiclaw_thinking` 表 | 是 |
| **Must-fix** | M3 | Tool Call 发送完整消息导致回复失去流式能力，UX 退化；建议 Tool Call 触发二次 STREAM 流式输出 | 是 |
| Should-fix | S1 | 私聊场景 aiclaw 必须总是回复，需在需求中明确约束 | 否 |
| Should-fix | S2 | 建表语句 rate_limit 应为 UNSIGNED，补充 is_del 等字段 | 否 |
| Should-fix | S3 | 防循环缺少对话深度检测，仅靠 respond_to_ai 不够 | 否 |
| Should-fix | S4 | WSRespTypeEnum 需同步新增 thinking 相关类型 | 否 |
| Should-fix | S5 | aichat-cli 认证机制未定义 | 否 |
| Info | I1 | 上下文窗口策略需在设计阶段明确（E1） | 否 |
| Info | I2 | 现有 hula_send_message 工具与 aichat-cli 的关系需明确（E2） | 否 |
| Info | I3 | THINKING_DELTA 的 stream context 路由需确认（E3） | 否 |
| Info | I4 | 群级配置修改的实时生效机制需设计（E4） | 否 |

**结论**：M1（枚举冲突）、M2（存储模型）、M3（流式能力退化）修复后可进入设计阶段。建议 M2 和 M1 一起解决——独立表方案同时消除了枚举冲突和存储限制问题。
