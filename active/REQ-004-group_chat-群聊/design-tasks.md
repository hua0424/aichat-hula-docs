# REQ-004 详细设计任务清单

> 编写：manager
> 日期：2026-05-20
> 阶段：详细设计（设计评审前置）
> 输入：[需求.md](需求.md) v2.1 + 三轮 reviewer 评审 + 三份可行性评估

---

## 一、本阶段目标

需求方案已评审通过（v2.1），三个开发已完成可行性评估，**现在进入"具体怎么做"的详细设计阶段**。

> ⚠️ **本阶段只产出设计文档，不开始编码**。所有代码改动需在设计文档评审通过、任务分发后再启动。

每位 developer 输出一份独立的设计文档，落到 `teamdocs/active/REQ-004-group_chat-群聊/` 目录下：

| Owner | 文档 | 覆盖范围 |
|-------|------|----------|
| server-dev | `design-server.md` | HuLa-Server（im 服务）后端设计 |
| plugin-dev | `design-plugin.md` | aichat-node + aichat-claw 设计 |
| frontend-dev | `design-frontend.md` | HuLa 客户端前端设计 |

---

## 二、各 owner 设计文档产出要求

### 2.1 server-dev — `design-server.md`

**A. 数据库**
- [ ] `im_aiclaw_group_config` / `im_aiclaw_thinking` / `im_aiclaw_thinking_msg_rel` 完整 DDL（含字符集、引擎、索引、注释）
- [ ] Liquibase / Flyway 迁移脚本位置
- [ ] 与现有 `im_aiclaw` 表的关联关系
- [ ] XXL-Job 清理任务设计（thinking 30d/180d 过期，扫描批量 + 异步删除）

**B. 接口**
- [ ] `POST /room/group/member` 自动同意逻辑改造点（识别"被邀 uid 属于邀请人 aiclaw"的判定条件）
- [ ] 群配置 REST：CRUD 接口签名（query/update）、权限校验（仅 aiclaw 主人）
- [ ] 鉴权：aiclaw token 在群配置接口的传递与校验

**C. WS 事件**
- [ ] plugins → server：THINKING_START/DELTA/END (20/21/22) 入口处理类
- [ ] server → client：thinkingStart/Delta/End 推送路由（如何决定推送给哪些群成员客户端）
- [ ] 群配置变更通知 WS 事件类型 + payload
- [ ] **autoReply 字段的载体**：是放 `im_message.extra` 还是仅在 WS payload 中（需与 plugin-dev 对齐）

**D. 防循环（DB 权威层）**
- [ ] 频率限制（10/分钟）的统计 SQL + 索引设计（高频群性能考量）
- [ ] 每日上限（1000）的统计方式（基于 `im_message` 还是独立计数表）
- [ ] 限流触发后的响应给 aichat-node 的协议（HTTP 状态码 / WS 错误事件）

**E. 缓存**
- [ ] Redis 缓存 key 设计（`im_aiclaw_group_config:{aiclaw_uid}:{room_id}`）、过期策略
- [ ] 配置更新后失效 + WS 广播机制

**F. 待澄清问题**
- [ ] 上下文窗口策略（默认最近 50 条）由 server 提供还是 node 自行维护

**预估工作量**：6.5–7 人天（开发阶段，不含本设计阶段）

---

### 2.2 plugin-dev — `design-plugin.md`

**A. aichat-node 重构**
- [ ] `MessageHandler` 从 `streaming: boolean` 改为 `thinkingSessions: Map<sessionKey, ThinkingSession>` 的状态机设计
- [ ] `ClawAdapter` 新增 `ThinkingCallbacks` 接口（保留现有 `StreamCallbacks`）
- [ ] OpenclawAdapter 中 `stream='assistant'` 事件 → `onThinkingDelta` 的映射
- [ ] 并发场景：同房间多 aiclaw 同时思考的 session 隔离

**B. MCP Tool**
- [ ] `hula_send_message` 扩展 `extra?: Record<string, unknown>` 参数（承载 autoReply 标记）
- [ ] 群聊 roomId 校验（确认 aiclaw 已入群）
- [ ] **aichat-claw 的 token 上下文方案**：openclaw Tool execution context 是否传递 agent credential（R1，需与 server-dev 共同确认）
  - 若是：直接使用
  - 若否：aichat-claw 按 aiclaw 维护 `HulaApiClient` 实例池

**C. 防循环（内存层）**
- [ ] AI 互触发开关检查时机（调用 adapter.chat() 前）
- [ ] 短回复跳过：最近 3 条回复长度跟踪结构 + 触发逻辑
- [ ] autoReply 标记的接收与跳过逻辑
- [ ] 指数退避：每群 `aiRoundCounter`，延迟队列实现（setTimeout + 取消逻辑）
- [ ] 人类消息到达时计数器重置

**D. WS 协议**
- [ ] THINKING_START/DELTA/END payload 完整字段（与 server-dev 对齐）
- [ ] 群配置更新 WS 通知接收 + 本地缓存刷新

**E. aichat-cli**
- [ ] `send-message` 命令（roomId / uid）
- [ ] `group-config` 命令（查询/修改）
- [ ] 认证：aiclaw token 来源（`~/.aichat/credentials.jsonc`）

**预估工作量**：8 人天（开发阶段）

---

### 2.3 frontend-dev — `design-frontend.md`

**A. 组件**
- [ ] `ThinkingPanel.vue`：容器、按 roomId 过滤 thinkingStreams、多卡片列表、展开/收起
- [ ] `ThinkingCard.vue`：头像 + 名称 + 状态徽章、可折叠内容区、流式 auto-scroll、回顾模式
- [ ] 在 `ChatMain.vue:41-46` 的集成位置（替换现有 AI notice 条）

**B. 状态管理**
- [ ] `chat.ts` 新增 `thinkingStreams: Map<string, ThinkingState>` 完整数据结构
- [ ] 4 个 Action 实现：`startThinking` / `appendThinking` / `finalizeThinking` / `clearThinking`
- [ ] **与现有 `streamingMessages` / `pendingStreamReplace` 的隔离边界确认**（关键：保证 ISS-015 修复不被破坏）

**C. WS 事件**
- [ ] `wsType.ts` 新增 enum + DTO 类型
- [ ] `layout/index.vue` Mitt 监听 + rAF 节流（复用现有 STREAM 模式）
- [ ] 多 aiclaw 并发 thinking 时的 rAF buffer 隔离方案

**D. 群配置入口**
- [ ] aiclaw 管理页 → "群聊设置" 入口位置
- [ ] 配置项 UI：频率限制（数字输入）、每日上限、AI 互触发（开关）、是否需要 @ 触发
- [ ] 配置更新后 WS 通知的 UI 反馈

**E. UX 细节**
- [ ] THINKING_END 后状态条保留时长（建议默认 30s，可产品决定）
- [ ] ThinkingCard 内容 max-height + 内部滚动
- [ ] **移动端范围**：本期是否实现 `src/mobile/` 下的 ThinkingPanel？设计文档中需明确

**F. i18n**
- [ ] zh-CN / en 文案清单

**预估工作量**：3–3.5 人天（开发阶段）

---

## 三、关键跨组对齐项

下列问题需要多位 owner 共同确认，建议在设计期间通过 `send_message` 直接沟通：

| 编号 | 议题 | 关联 owner | 备注 |
|------|------|-----------|------|
| X1 | THINKING_START/DELTA/END WS payload 完整字段 | server-dev + plugin-dev | 含 fromUid、roomId、triggerMsgId、seq、durationMs 等 |
| X2 | autoReply 字段载体（im_message.extra vs WS payload only） | server-dev + plugin-dev + frontend-dev | 影响数据库设计与前端跳过逻辑 |
| X3 | 防循环分层最终方案 | server-dev + plugin-dev | server 承担 DB 权威（频率/日限），node 承担内存层（互触发/短回复/退避） |
| X4 | aichat-claw Token 上下文方案（R1） | plugin-dev + server-dev | openclaw Tool execution context 是否提供 agent credential |
| X5 | 群配置 WS 通知 payload 格式 | server-dev + plugin-dev + frontend-dev | 一次变更需推送 node 与所有相关客户端 |
| X6 | 上下文窗口（默认 50 条）由 server 还是 node 维护 | server-dev + plugin-dev | 影响内存与查询性能 |

**对齐方式建议**：每个议题由其中一位 owner 起草初稿写入自己的设计文档，通过 `send_message` 通知相关方 review，达成一致后各自定稿。

---

## 四、时间线

> 今天：2026-05-20（周三）

| 节点 | 日期 | 内容 |
|------|------|------|
| 设计启动 | 05-20 | manager 分派任务，三位开发开始设计 |
| 跨组对齐 | 05-21 ~ 05-22 | X1–X6 跨组议题对齐 |
| 设计初稿提交 | **05-25（周一）** | 三份设计文档 push 到 teamdocs，通知 manager |
| reviewer 评审 | 05-26（周二） | reviewer 评审三份设计 + 跨组一致性检查 |
| 修订 | 05-27 ~ 05-28 | 按评审意见修订 |
| 终稿通过 | **05-29（周五）** | 设计阶段结束，进入任务分发 |

---

## 五、产出格式建议

每份设计文档建议结构：

```
1. 概述（关联需求条目）
2. 整体架构图 / 数据流图（mermaid 可）
3. 详细设计（按本文档列出的产出要求展开）
4. 接口/数据模型/代码骨架
5. 与跨组对齐项的最终一致结论
6. 风险与降级方案
7. 工作量分解（细化到 0.5 天粒度的子任务）
```

---

## 六、注意事项

1. **只设计、不编码**：本阶段产出的是设计文档，**不要**提交任何代码改动到对应仓库。
2. **基于需求 v2.1**：所有设计须与最新 `需求.md`（v2.1）和三轮评审意见一致。
3. **不确定项写"待 manager 决策"**：任何设计中无法独立决断的开放问题，列出选项并标注，由 manager 在评审时裁决，不要等待。
4. **提交方式**：写入 `teamdocs/active/REQ-004-group_chat-群聊/design-{owner}.md`，git commit & push 到 group_chat 分支后通过 `send_message` 通知 manager。
5. **每日同步**：每天开始工作时调用 `set_summary` 更新进度。
