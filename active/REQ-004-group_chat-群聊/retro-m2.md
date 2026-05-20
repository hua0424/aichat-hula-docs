# REQ-004 M2 里程碑回顾

> 主持：manager
> 日期：2026-05-20
> 里程碑：M2 Agent Loop + Thinking 落库 + 展示
> 预计工作量：6.5 人天（并行约 3 自然日）
> 实际历时：**约 30 分钟**（同日内完成）

---

## 一、完成情况

### 1.1 各方提交

| Owner | 任务 | Commit | 状态 |
|-------|------|--------|------|
| server-dev | S-M2-1 ~ S-M2-4 | HuLa-Server `e1556f61` | ✅ |
| plugin-dev | P-M2-1 ~ P-M2-7 | aichat-plugins `6495ea5` | ✅ |
| frontend-dev | F-M2-1 ~ F-M2-4 | HuLa `896f1b848` | ✅ |

### 1.2 产出明细

**server-dev**：
- ThinkingService：雪花 ID 生成 + create/appendDelta/finalize 落库
- ThinkingProcessor：WS 20/21/22 入口处理 + HTTP 调 IM 服务
- WS 推送路由：THINKING_START 生成 thinkingId 后广播 + 5 分钟超时清理 + 缓存群成员列表
- addMember 改造：识别邀请人 aiclaw 列表 + 自动入群（GroupMember + 会话 + 缓存 + 事件）

**plugin-dev**：
- MessageHandler：streaming → thinkingSessions Map 重构，triggerAgentLoop
- thinkingId 回填：handleThinkingStartBroadcast 通过 `fromUid===selfUid + triggerMsgId` 双重匹配
- ThinkingSession 5 分钟超时清理（setTimeout + onError + 自动 THINKING_END）
- OpenclawAdapter：assistant → onThinkingDelta，lifecycle end → onThinkingEnd(durationMs)
- hula_send_message 扩展 extra 参数（含 thinkingId + autoReply 预留）
- HulaApiClientPool：多 aiclaw 独立 token 实例池（aichat-claw 内）
- userType=4（AICLAW）判断；M2 暂跳过 AI 消息（M3 加入 respondToAi 后开放）

**frontend-dev**：
- chat.ts thinkingStreams Map + thinkingArchive + 6 个 actions/methods
- autoReplyMessages Set + markMessageAsAutoReply（M3 预留）
- layout/index.vue THINKING 事件 Mitt 监听 + rAF 节流（独立于 streamDeltaBuffers）
- ThinkingCard.vue 流式光标 + 状态徽章 + 折叠 + 30s 归档
- ThinkingPanel.vue 多卡片容器 + 归档抽屉（n-drawer）
- ChatMain.vue 集成（D1 决策：群聊 + 私聊统一），showThinkingPanel 双场景覆盖
- i18n zh-CN + en 文案

---

## 二、跨组对齐核查

### 2.1 WS Payload 字段一致性（plugin-dev 自测已确认）

| 协议项 | plugin 字段 | server 字段 | 一致性 |
|--------|------------|-------------|--------|
| THINKING_START 入参 | fromUid, roomId, triggerMsgId | fromUid, roomId, triggerMsgId | ✅ |
| THINKING_DELTA 入参 | thinkingId, chunk, seq, roomId | thinkingId, chunk, seq, roomId | ✅ |
| THINKING_END 入参 | thinkingId, durationMs, status, error, roomId | thinkingId, durationMs, status, error, roomId | ✅ |
| WSThinkingStartResp | { thinkingId, fromUid, roomId, triggerMsgId } | { thinkingId, fromUid, roomId, triggerMsgId } | ✅ |

### 2.2 thinkingId 回填路径

- server `WSThinkingStartResp` 含 thinkingId，广播到群成员
- plugin `handleThinkingStartBroadcast` 通过 `fromUid===selfUid + triggerMsgId` 双重匹配
- frontend 从 ThinkingStartPayload 接收 thinkingId 创建 ThinkingState

### 2.3 hula_send_message extra.thinkingId

- plugin Tool schema 接受 `extra?: Record<string, unknown>` ✅
- 透传到 REST body 路径：`body.extra.thinkingId` ✅
- 与 design-server.md §S5 完全一致

### 2.4 aiclawName/aiclawAvatar 来源

- server 端：不在 payload 中携带（v1.2 决策）
- frontend 端：三级降级 `payload.aiclawName → groupStore.getUserInfo(uid)?.name → 'AI'`
- frontend `startThinking` 实现已 ready

---

## 三、设计与实现差异

### 3.1 server: WSReqType THINKING_START 通过 HTTP 调用 IM 服务落库

server-dev 实际实现：ThinkingProcessor 收到 WS 事件后通过 HTTP 调用 IM 服务（而非直接调本地 ThinkingService），这是 HuLa-Server 微服务架构的正常做法（ws-server 与 im-server 解耦）。设计文档 §3.3.2 简化描述为"直接落库"，实际是合理的微服务实现，**无问题**。

### 3.2 plugin: ThinkingSession 5 分钟超时已实现

S3 修订项落地。

### 3.3 frontend: showThinkingPanel 双场景

D1 决策落地。`isAiclawSession || (isGroup && hasAiclawInGroup)` 双分支覆盖私聊 + 群聊。

### 3.4 plugin: M2 暂跳过所有 AI 消息

`userType === 4` 时直接跳过，M3 加入 respondToAi 配置后再开放。这是合理的渐进策略，避免 M2 阶段未实现防循环时出现 AI 互触发死循环。

---

## 四、M1 经验教训复用情况

| 经验 | 复用情况 |
|------|----------|
| 跨组协议常量值对齐 | ✅ plugin-dev 主动自测 + 列对照表 |
| Entity-DB schema 一致性 | ✅ server-dev M1-fix 后无新增 Entity，沿用现有 |
| 字段语义由权威方填充 | ✅ status 字段由 plugin → server 路径完整 |
| 每天开工前必拉 teamdocs | ⚠️ plugin-dev 自测时短暂误判（design-server v1.3 已加 status 但其本地未拉最新），及时纠正 |

**沉淀 1**：自测时如发现"设计未提及但应该有的字段"，第一步先 `git pull` 排除文档不同步问题，第二步再上报。

**沉淀 2（M2 提测踩坑）**：M1-fix 复测时 backend-tester 提到"代码变更简单无新增依赖，编译风险极低；如需可临时启动 dev 容器验证"，但**未实际编译**。M2 提测时立即踩到 Lombok `@Builder` vs `SuperEntity` 父类 id 字段不兼容的编译错误（`ThinkingService.create().id(thinkingId)` 编译失败）。

→ **规则**：今后涉及 **Entity 基类切换** 的修复（如 `Entity` ↔ `SuperEntity` ↔ 独立 POJO），**强制走一次实际编译验证**，不依赖"风险评估"。代价小（一次 Maven 编译），收益大（避免下一阶段提测被阻塞）。

修复：server-dev commit `b4ed3241`，`.id(thinkingId)` 链式调用拆成 `build()` 后 `setId(thinkingId)`。

---

## 五、设计与代码同步状态

| 文档 | 当前版本 | 同步状态 |
|------|---------|---------|
| design-server.md | v1.3 | ✅ M1-fix 后已同步 |
| design-plugin.md | v1.3 | ✅ M1-3/M1-4 已同步 |
| design-frontend.md | v1.2 | ⚠️ 是否需要 v1.3 同步 status 字段说明？（frontend 早已定义 status，不影响） |

建议 frontend-dev 在 M3 启动前可以做一个 design-frontend.md 微调，明确"status 字段由 server 推送"，与 design-server v1.3 / design-plugin v1.3 完全对齐。**不阻塞 M3 启动**。

---

## 六、提测准入

### 6.1 准入清单

- [x] 三方所有 M2 任务完成
- [x] 三方自测覆盖（WS payload + thinkingId 回填 + 隔离边界 + 私聊 D1）
- [x] 跨组对齐项无歧义
- [x] WS 协议常量值三方一致（M1 验证 + M2 新增字段一致）

### 6.2 提测项

| 测试方 | 内容 |
|--------|------|
| backend-tester | aiclaw 自动入群、thinking 落库、关联表回写、WS 推送路由、5 分钟超时清理 |
| ui-tester | 群聊 thinking UI 展示、多 aiclaw 并发、归档抽屉、桌面端 + 移动端、私聊 thinking 展示（D1 验证） |

### 6.3 端到端验证场景

1. **群聊 + 单 aiclaw 提问**：用户发消息 → aiclaw 入 agent loop → thinking 流式显示 → 正式消息发出
2. **群聊 + 多 aiclaw 并发**：多 aiclaw 同时 thinking，前端独立卡片展示
3. **私聊 + aiclaw thinking**：D1 决策验证
4. **aiclaw 邀请入群**：owner 邀请自己的 aiclaw，无 UserApply 待处理记录
5. **超时清理**：模拟 openclaw 卡死，5 分钟后自动 THINKING_END(error)

---

## 七、决议

✅ **M2 通过，启动 backend-tester + ui-tester 增量提测，提测通过后启动 M3。**

待办：
1. backend-tester 启动 M2 提测（API + WS + DB 落库）
2. ui-tester 启动 M2 提测（首次介入：thinking UI 展示）
3. 提测发现问题立即修复（参考 M1 经验，P0/P1 阻塞 M3，P2/P3 可并行修复）
4. M3 启动前 frontend-dev 可选做 design-frontend.md v1.3 微调

---

## 八、M3 启动预告

**M3 范围**：防循环 + 群配置 + autoReply（~5d）
- server: AiclawGroupConfigController + Redis 滑动窗口 + THINKING_START 前置限流 + autoReply WS 透传 + 群配置广播
- plugin: AntiLoopGuard（互触发/短回复/退避）+ groupConfigCache + autoReply 收发 + aichat-cli 命令
- frontend: 群配置 store + 桌面/移动端配置 UI + autoReply 标记 + i18n

**M3 启动前提**：
- M2 backend-tester + ui-tester 提测通过
- 修复所有 M2 提测发现的 P0/P1 缺陷

预计 M3 启动时间：M2 提测完成后立即启动。
