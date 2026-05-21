# REQ-004 v1.1 二轮评审 — reviewer 反馈

> 评审人：reviewer
> 日期：2026-05-20
> 评审对象：[需求.md](需求.md) v1.1（commit 8202285）
> 结论：**通过，可进入设计阶段**

---

## Must-fix 修复验证

| # | 问题 | 修复情况 | 验证结果 |
|---|------|----------|----------|
| M1 | type=5 冲突 | §3.3 + §4.3 明确"MessageTypeEnum 不做任何变更"，思考内容用独立表 | ✅ 彻底解决 |
| M2 | varchar(1024) + 存储模型 | §4.2 新建 `im_aiclaw_thinking`（TEXT），含 `trigger_msg_id`/`response_msg_id`/`duration_ms`；§2.3.3 过期策略（30d/180d）| ✅ 完整且合理 |
| M3 | 流式能力退化 | §2.3.1 清晰描述两阶段：THINKING 展示 → 复用 STREAM 协议流式回复；§2.5 注明"回复阶段不走 CLI REST API，走 WS STREAM" | ✅ 关键补充到位 |

## Should-fix 纳入验证

| # | 纳入情况 | 验证结果 |
|---|----------|----------|
| S1 私聊必须回复 | §2.3.2 三条硬约束 + aichat-node 层面强制 | ✅ |
| S2 UNSIGNED + is_del | §4.1 全部 UNSIGNED，新增 is_del + daily_limit | ✅ |
| S3 防循环 | §2.4 五层防护（频率/日限/AI互触发/深度检测/占比检测） | ✅ 超出预期 |
| S4 WSRespTypeEnum | §3.1 新增 thinkingStart/thinkingDelta/thinkingEnd | ✅ |
| S5 CLI 认证 | §2.5 aiclaw token + 权限范围限制 | ✅ |

## Info 项落实

| # | 落实情况 |
|---|----------|
| I1 上下文窗口 | §2.2 明确最近 N 条（默认 50），设计阶段细化 |
| I2 工具复用 | §2.5 + §六 明确评估策略，设计阶段决定 |
| I3 stream context 路由 | §3.2 注释说明 aiclawUid 关联，设计阶段细化 |
| I4 配置实时生效 | §2.2 Redis 缓存 + WS 通知 |

## 亮点

1. **§2.5 的"两阶段回复不走 CLI"区分**：主动发言走 CLI REST API，回复阶段走 WS STREAM——清晰分离两种消息路径，避免混淆
2. **§2.4 五层防循环**：深度检测 + AI 占比检测超出了我原始建议的范围，防护充分
3. **§2.1 核实了群管理 API**：定位到具体代码文件和方法，可行性已验证

## 设计阶段注意事项（非阻塞）

1. **im_aiclaw_thinking 与 STREAM 流的 response_msg_id 关联时机**：STREAM_END 时 persistStreamMessage 生成 msgId，此时需回填 thinking 记录的 `response_msg_id`。aichat-node 需要在 STREAM_END 回调中执行此更新。
2. **深度检测的实现**：需查询群内最近 N 条消息判断"连续 aiclaw 回复链"，注意查询性能（高频群建议缓存最近消息列表）。
3. **THINKING_DELTA 的 context 路由**：Server 需维护独立的 thinking stream context map（与现有 activeStreams 分开），因为同一 aiclaw 可能在 thinking 阶段时也有活跃的 stream。

---

**结论：v1.1 修复完整，需求文档质量达标，批准进入设计阶段。**
