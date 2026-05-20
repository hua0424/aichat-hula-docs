# REQ-004 M3 里程碑回顾

> 主持：manager
> 日期：2026-05-20
> 里程碑：M3 防循环 + 群配置 + autoReply
> 预计工作量：6.5 人天（并行约 3 自然日）
> 实际历时：**约 2 小时**（同日内完成，含设计调整）

---

## 一、完成情况

### 1.1 各方提交

| Owner | 任务 | Commit | 状态 |
|-------|------|--------|------|
| server-dev | S-M3-1 ~ S-M3-7 + 短回复 skip 升级 | HuLa-Server M3 commit + short-reply commit | ✅ |
| plugin-dev | P-M3-1 ~ P-M3-6 + gap-2 收尾 | aichat-plugins `f6729f1` + `ba76c09` | ✅ |
| frontend-dev | F-M3-1 ~ F-M3-5 | HuLa `69a8cd2d5` | ✅ |
| 文档迭代 | design-server v1.4 + design-plugin v1.5 | teamdocs `eff5912` + `205f798` | ✅ |

### 1.2 产出明细

**server-dev**：
- AiclawGroupConfigController CRUD + 权限校验（GET 群成员，PUT 主人/aiclaw 本人）
- Redis 滑动窗口限流（频率/日限）— im-biz 实现
- THINKING_START 前置限流校验 — ws-biz 实现
- 群配置变更 WS 广播（嵌套 ConfigDTO）
- aiclaw 主人关系 Redis 缓存 + 变更刷新
- aiclaw token 独立鉴权（Gateway + Controller 权限矩阵）
- **新增**：短回复 skip 在 `ChatServiceImpl.sendMsg()` 中实现，配置读 `im_aiclaw_group_config.short_reply_threshold/lookback`
- error code 标准化：`rate_limit_exceeded` / `daily_limit_exceeded` / `short_reply_skip`

**plugin-dev**：
- AntiLoopGuard 实现互触发/退避两规则（短回复升级到 server 后移除）
- groupConfigCache + WS 通知刷新
- autoReply 收发：内嵌轻量 HulaApiClient + sendAutoReply
- 多实例限制文档化（ANTI_LOOP_MULTI_INSTANCE.md）
- aichat-cli send-message + group-config 命令
- **gap-2 收尾**：handleThinkingEndBroadcast 中 switch 三个 error code 触发对应 autoReply

**frontend-dev**：
- chat.ts aiclawGroupConfigs Map + 4 actions
- ImUrlEnum + webImRequest.ts + im_request_client.rs URL 映射
- 桌面端 aiAssistantWindow groupSettings RightView + 配置表单
- 移动端 AiclawGroupSettings.vue 全屏页面 + 路由
- autoReply UI 标记（消息气泡上方"自动回复"标签）
- i18n zh-CN + en（10 个 group_settings keys + auto_reply_tag）

---

## 二、本里程碑发生的设计调整

### 2.1 短回复 skip 升级（plugin → server）

**发现**：plugin-dev 在 P-M3-1 实现 AntiLoopGuard 时验证了进程边界：
- aichat-node（WS 客户端）与 aichat-claw（openclaw 插件）**不在同一进程**
- aichat-claw 通过 hula_send_message Tool 直接走 REST 发消息到 server，aichat-node 不在发送路径上
- → plugin 端 shouldSkipShortReply 实现了但无法 wire up

**决策**：升级到 server 权威层（ChatServiceImpl.sendMsg 中查最近 N 条消息长度）

**实施**：
- server: 新增 `ChatServiceImpl.checkShortReplySkip()` + `msgMapper.selectRecentMsgLengths()`
- plugin: shouldSkipShortReply/recordReply 保留为 dead code（下次 cleanup PR 移除）
- 文档：design-server v1.4 §3.4.4 / design-plugin v1.4 移除 §E.2 short reply 段

### 2.2 autoReply 触发条件标准化

**发现**：plugin-dev 实现 P-M3-3 时发现 sendAutoReply 缺少触发入口（guard.check() 只返回 allow/delay，没有 block）

**决策**：通过 server 推送的 thinkingEnd error code 触发

**实施**：
- server: 限流场景推 `thinkingEnd { status: "error", error: "rate_limit_exceeded" }` 等
- plugin: handleThinkingEndBroadcast 中 switch error code 触发对应 sendAutoReply
- 文档：design-plugin v1.5 新增 autoReply 触发条件表 + error code 表

### 2.3 经验复用

| 经验沉淀 | 复用情况 |
|---------|---------|
| 跨组协议常量值对齐（M1） | ✅ error code 字符串三方对齐 |
| Entity-DB schema 一致性（M1） | ✅ 无新 Entity 改动 |
| 设计-代码 Entity 基类切换必须实际编译验证（M2） | ✅ 无 Entity 基类调整 |
| 跨模块设计要求必须逐条对照核查（M2） | ✅ S-M3-4 autoReply 透传 server/plugin 闭环验证 |
| SuperEntity 表 DDL 必须含 tenant_id（M2） | ✅ im_aiclaw_group_config DDL 已在 M3 启动前补齐 tenant_id（commit c262b806） |
| 每天开工前必拉 teamdocs（M2） | ✅ 三方均确认按规则执行 |

---

## 三、跨组对齐核查

### 3.1 群配置 WS 广播 payload（X5 复核）

| 项 | server | plugin | frontend | 一致性 |
|----|--------|--------|----------|--------|
| WS type | `groupConfigChange` | `groupConfigChange` | `groupConfigChange` | ✅ |
| 嵌套结构 | `WSGroupConfigChange.config: ConfigDTO` | 嵌套 `config` | 嵌套 `config` | ✅ |
| 字段：rateLimitPerMinute | ✅ | ✅ | ✅ | ✅ |
| 字段：dailyLimit | ✅ | ✅ | ✅ | ✅ |
| 字段：respondToAi | ✅ | ✅ | ✅ | ✅ |
| 字段：mentionRequired | ✅ | ✅ | ✅ | ✅ |

### 3.2 群配置 REST API

| 项 | server 实现 | plugin CLI | frontend 调用 | 一致性 |
|----|------------|-----------|---------------|--------|
| 路径 | `/aiclaw/group/config` | `/aiclaw/group/config` | `AICLAW_GROUP_CONFIG_LIST` / `_UPDATE` 映射 | ✅ |
| GET 权限 | 群成员可访问 | aichat-cli group-config 查询 | aiclaw 管理页加载 | ✅ |
| PUT 权限 | 主人 / aiclaw 本人 | aichat-cli group-config 更新 | aiclaw 主人保存 | ✅ |

### 3.3 autoReply 触发链路

```
[场景：发送频率超限]
plugin 调 hula_send_message
  → server REST /api/im/chat/msg → ChatServiceImpl.sendMsg
  → 频率限流校验失败 → 推 WS thinkingEnd { status: "error", error: "rate_limit_exceeded" }
  → plugin handleThinkingEndBroadcast 收到 → sendAutoReply("发言频率限制，已自动跳过本次响应")
  → 调用内嵌 HulaApiClient.sendMessage(roomId, content, extra: { autoReply: true })
  → server REST /api/im/chat/msg → MsgSendConsumer 透传 extra.autoReply 到 WS payload
  → 其他 aiclaw plugin 收到 RECEIVE_MESSAGE → 检测 extra.autoReply=true 跳过 agent loop
  → frontend ChatMain 展示"自动回复"标签
```

✅ 三方端到端完整。

---

## 四、设计与实现差异

### 4.1 server: 短回复 skip 实现位置

设计文档 v1.4 §3.4.4 描述放在 `ChatServiceImpl.sendMsg()` 在 `msgHandler.checkAndSaveMsg()` 之前。实际实现路径与设计一致 ✅。

### 4.2 plugin: AntiLoopGuard.shouldSkipShortReply 残留

由于设计调整，shouldSkipShortReply 和 recordReply 成为 dead code。

**处理**：M3 不删除（避免冲突影响其他子任务），M4 联调前做 cleanup commit 移除。

### 4.3 frontend: groupSettings 子视图

完整实现桌面端 + 移动端两套，超出设计文档 §3.5 的最小集要求（设计文档原本只描述了桌面端，移动端是 M3 自然补完）。frontend 自决无问题 ✅。

---

## 五、提测准入

### 5.1 准入清单

- [x] 三方所有 M3 任务完成
- [x] 设计调整及时更新到文档（design-server v1.4 / design-plugin v1.5）
- [x] error code 标准化三方对齐
- [x] groupConfigChange WS 协议三方对齐
- [x] autoReply 端到端链路自测通过

### 5.2 提测项

| 测试方 | 内容 |
|--------|------|
| backend-tester | 群配置 REST CRUD + Redis 滑动窗口 + 短回复 skip + autoReply 透传 + 配置变更广播 + Redis aiclaw 主人关系 |
| ui-tester | 群配置页面（桌面 + 移动） + 配置变更 toast + autoReply 标签 + 限流场景 UI 反馈 + 深度 GUI（需测试账号） |

### 5.3 端到端验证场景

1. **群配置 CRUD**：主人在管理页修改配置 → server 落库 + Redis 缓存失效 + WS 广播
2. **频率限流触发**：aiclaw 高频发言 → server 拒绝 → plugin autoReply → 群内显示"自动回复"
3. **每日上限触发**：aiclaw 当日累计超 1000 → 同上
4. **短回复 skip**：aiclaw 最近 3 条均 < 10 字符 → server 拒绝 → plugin 仅日志不触发 autoReply
5. **AI 互触发开关**：respondToAi=false 时，aiclaw 不响应其他 aiclaw 消息
6. **AI-to-AI 退避**：连续 AI-to-AI 6/11/21 轮时延迟 5/15/30s

---

## 六、决议

✅ **M3 通过，启动 backend-tester + ui-tester 增量提测。**

待办：
1. backend-tester 启动 M3 提测（群配置 CRUD + 限流 + autoReply）
2. ui-tester 启动 M3 提测（群配置 UI + autoReply 标记 + 深度 GUI）
3. server-dev / backend-tester 配合准备 ui-tester 深度 GUI 测试所需账号
4. 提测发现 P0/P1 阻塞 M4，P2/P3 可并行修复

---

## 七、M3 新增经验沉淀

### 沉淀 5：进程边界对设计的隐式约束

**事件**：plugin-dev 实现 P-M3-1 时发现"aichat-node ↔ aichat-claw 不在同一进程"对 AntiLoopGuard 的 wire up 有硬约束。设计阶段没显式标注两者的进程关系，导致 §E.2 设计的短回复 skip 在 plugin 内存层"看起来合理但实际无法接入"。

→ **规则**：设计文档涉及"跨组件状态共享"时，必须明确**组件的进程/部署关系**。可在设计文档加一节"组件部署拓扑"，标注哪些组件同进程、哪些跨进程、跨进程通信方式（HTTP/WS/MQ/IPC）。

后续 confirm review 检查清单增加：「设计中提到的"共享状态"，是否经过进程边界检查」。

### 沉淀 6：dead code 的处理时机

**事件**：plugin AntiLoopGuard.shouldSkipShortReply 因设计调整成为 dead code。

→ **规则**：里程碑中产生的 dead code，**不在本里程碑内做 cleanup**（避免与其他变更产生 merge 冲突或测试干扰），**在下一里程碑启动前做 cleanup commit**。本次 M3 → M4 之间 plugin-dev 做 cleanup。

---

## 八、M4 启动预告

**M4 范围**：端到端联调 + 全场景验收 + bugfix（~3-5d）
- 5 个真实场景验证（见 §5.3）
- backend-tester 集成测试 + 性能压测（thinking 高并发）
- ui-tester 桌面 + 移动端完整 UI 验收
- 三方协同 bug 修复
- manager 测试报告汇总
- plugin-dev 在 M4 启动前做 M3 dead code cleanup

**M4 启动前提**：
- M3 backend-tester + ui-tester 提测通过
- 修复所有 M3 提测发现的 P0/P1 缺陷
