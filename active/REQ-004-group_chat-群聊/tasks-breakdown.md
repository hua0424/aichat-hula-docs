# REQ-004 任务分解与开发执行计划

> 编写：manager
> 日期：2026-05-20
> 阶段：任务分发
> 状态：进行中
> 输入：design-server.md v1.2 + design-plugin.md v1.2 + design-frontend.md v1.2 + review-design-reviewer.md（v1.2 confirm 通过）

---

## 一、总体策略

### 1.1 分支策略

- **统一在 `group_chat` 分支开发**（三个仓库 + teamdocs 子模块均使用 `group_chat`）
- 完成所有里程碑并测试通过后，由 **manager 报请 owner 决定**是否合并到 `dev`
- 不创建 feature 分支，所有变更直接 push 到 `group_chat`
- 每个里程碑结束时打 tag：`req-004-m1`、`req-004-m2`、`req-004-m3`、`req-004-m4`

### 1.2 里程碑划分

总开发量 18.6 人天，拆分为 4 个里程碑，**每个里程碑结束后**：
1. 三方完成自测
2. manager 主持里程碑回顾，对照设计文档核查实现是否一致
3. 必要时调整后续设计（reviewer 参与设计调整评审）
4. backend-tester / ui-tester 做**增量提测**
5. 重大争议停下来与 owner 确认后再继续

```
M1 协议层 + 基础设施 (~3d)
       ↓
       ↓ 里程碑回顾 + 增量提测
       ↓
M2 Agent Loop + Thinking 展示 (~5d)
       ↓
       ↓ 里程碑回顾 + 增量提测
       ↓
M3 防循环 + 群配置 + autoReply (~5d)
       ↓
       ↓ 里程碑回顾 + 增量提测
       ↓
M4 联调 + 验收 + bugfix (~3-5d)
       ↓
    需求闭环
```

### 1.3 增量提测原则

- 每个里程碑结束都提测（M1/M2/M3/M4）
- backend-tester：API 接口 + WS 协议层验证
- ui-tester：桌面端 + 移动端 UI 行为验收
- 测试通过才进入下一个里程碑
- bug 修复优先于推进新里程碑

### 1.4 重大争议处理

里程碑回顾或开发过程中如出现以下情况，**立即停下来与 owner 确认**：
- 设计与实现冲突，需要修改设计核心思路
- 跨组接口需要重新对齐
- 工作量预估偏差超过 30%
- 引入了原需求/设计未提到的新依赖

---

## 二、里程碑 M1：协议层 + 基础设施（~3 工作日）

### 2.1 目标

打通 WS 协议层 + 数据库层，让三方能进入"空跑联调"状态。

### 2.2 任务清单

#### server-dev（~1.0d）

| # | 任务 | 设计章节 | 工作量 |
|---|------|---------|--------|
| S-M1-1 | 3 张新表 DDL + 手动 SQL 文件落到 `docs/sql/` | design-server §3.1 | 0.5d |
| S-M1-2 | Entity + Mapper 骨架（AiclawGroupConfig / AiclawThinking / AiclawThinkingMsgRel） | §4.1 | 0.3d |
| S-M1-3 | WSReqTypeEnum / WSRespTypeEnum 扩展（20/21/22 + groupConfigChange） | §3.3.1 | 0.2d |

#### plugin-dev（~1.0d）

| # | 任务 | 设计章节 | 工作量 |
|---|------|---------|--------|
| P-M1-1 | WS 协议扩展：`protocol.ts` 新增 THINKING_* + groupConfigUpdate | design-plugin §F.1 | 0.5d |
| P-M1-2 | `ClawAdapter` 接口扩展：`ThinkingCallbacks` 接口定义 | §B.2 | 0.3d |
| P-M1-3 | `OpenclawAdapter.PendingChat` 结构补充 `startTime` 字段 | §C.3 | 0.2d |

#### frontend-dev（~0.5d）

| # | 任务 | 设计章节 | 工作量 |
|---|------|---------|--------|
| F-M1-1 | `wsType.ts` 新增 enum + 4 个 Payload 类型 + AiclawGroupConfig 类型 | design-frontend §3.1 | 0.3d |
| F-M1-2 | `MsgType` 扩展可选 `extra` 字段 | §3.6 | 0.2d |

### 2.3 M1 验收标准

- ✅ 3 张表 DDL 在 dev MySQL 上执行成功
- ✅ server / plugin / frontend 三方 WS 协议常量值一致（20/21/22 + `thinkingStart/Delta/End` + `groupConfigChange`）
- ✅ plugin 能通过新协议发送 dummy THINKING_START，server 能解析（暂不需要广播）
- ✅ frontend 能识别新 WS 事件类型（暂不需要 UI 展示）
- ✅ Entity / DTO 类型在三端定义一致

### 2.4 M1 提测内容

- backend-tester：DDL 执行 + Entity 单元测试 + WS 枚举值校验
- ui-tester：本里程碑无 UI 改动，跳过

---

## 三、里程碑 M2：Agent Loop + Thinking 展示（~5 工作日）

### 3.1 目标

完整跑通 Agent Loop 模型：plugin 收到群消息 → 触发 agent → THINKING 流式 → MCP Tool 发消息 → 三方展示。

### 3.2 任务清单

#### server-dev（~2.0d）

| # | 任务 | 设计章节 | 工作量 |
|---|------|---------|--------|
| S-M2-1 | ThinkingService：creat/appendDelta/finalize（含 thinkingId 生成） | design-server §3.3.2 | 0.5d |
| S-M2-2 | ThinkingEventHandler：处理 20/21/22 入口 + 落库 | §3.3.2 / §4.4 | 0.5d |
| S-M2-3 | WS 推送路由：thinking 事件按 roomId 广播给群成员（带 thinkingId） | §3.3.2 / S1 修订 | 0.5d |
| S-M2-4 | `POST /room/group/member` 改造：识别 aiclaw uid → 自动同意入群 | §3.2.1 | 0.5d |

#### plugin-dev（~3.0d）

| # | 任务 | 设计章节 | 工作量 |
|---|------|---------|--------|
| P-M2-1 | `MessageHandler` 重构：streaming boolean → thinkingSessions Map | design-plugin §A.2 / §A.4 | 1.0d |
| P-M2-2 | thinkingId 回填机制：通过 fromUid===selfUid 匹配回填 | §A.4 / S1 修订 | 0.3d |
| P-M2-3 | ThinkingSession 5 分钟超时清理 | §A.2 / S3 修订 | 0.3d |
| P-M2-4 | `OpenclawAdapter` assistant→thinking 映射 | §C.2 | 0.5d |
| P-M2-5 | `hula_send_message` Tool 扩展：`extra.thinkingId` 字段 | §D.1 / S5 修订 | 0.5d |
| P-M2-6 | userType=4（AICLAW）判断逻辑 | §A.4 / I2 修订 | 0.2d |
| P-M2-7 | aichat-claw HulaApiClient 实例池（方案 B 保底） | §D.4 | 0.2d |

#### frontend-dev（~1.5d）

| # | 任务 | 设计章节 | 工作量 |
|---|------|---------|--------|
| F-M2-1 | `chat.ts` thinkingStreams + 4 个 actions + archive 机制 | design-frontend §3.2 | 0.5d |
| F-M2-2 | `layout/index.vue` Mitt 监听 + rAF 节流（THINKING_* 三事件） | §3.3 | 0.3d |
| F-M2-3 | `ThinkingCard.vue` 组件（头部 + 内容 + 流式光标 + 折叠） | §3.4.2 | 0.4d |
| F-M2-4 | `ThinkingPanel.vue` 容器 + 归档抽屉 | §3.4.1 | 0.3d |

### 3.3 M2 验收标准

- ✅ 主人邀请 aiclaw 入群，aiclaw 自动同意（无 UserApply 待处理记录）
- ✅ 群内用户发消息，aiclaw 进入 agent loop 触发 thinking
- ✅ Thinking 内容流式出现在前端 ThinkingPanel，多 aiclaw 并发时独立卡片
- ✅ Thinking 内容落库到 `im_aiclaw_thinking`（带 thinkingId）
- ✅ Agent 通过 `hula_send_message` 发出的正式消息正常显示
- ✅ `im_aiclaw_thinking_msg_rel` 关联表写入正确
- ✅ THINKING_END 后卡片折叠，30s 后归档

### 3.4 M2 提测内容

- backend-tester：API 入群自动同意、thinking 落库、关联表回写、WS 路由
- ui-tester：群聊 thinking UI 展示、多 aiclaw 并发、归档抽屉、桌面/移动端

---

## 四、里程碑 M3：防循环 + 群配置 + autoReply（~5 工作日）

### 4.1 目标

实现 5 层防循环 + 群配置管理界面 + autoReply 跳过机制。

### 4.2 任务清单

#### server-dev（~2.5d）

| # | 任务 | 设计章节 | 工作量 |
|---|------|---------|--------|
| S-M3-1 | `AiclawGroupConfigController`：GET/PUT REST API + 权限校验 | design-server §3.2.2 | 0.5d |
| S-M3-2 | Redis 滑动窗口计数器（频率/日限） | §3.4 | 0.5d |
| S-M3-3 | THINKING_START 前置限流校验（超限返回 error 事件） | §3.4 / S2 修订 | 0.3d |
| S-M3-4 | autoReply WS payload 透传（不落库） | §3.3.4 / I3 修订 | 0.3d |
| S-M3-5 | 群配置 WS 广播：嵌套 ConfigDTO 推送给群内所有成员 | §3.3.3 / M1 修订 | 0.4d |
| S-M3-6 | aiclaw 列表 Redis 缓存 + 变更通知刷新 | §3.2.1 / X4 决策 | 0.3d |
| S-M3-7 | aiclaw token 独立鉴权链路 | §3.2.3 / I1 | 0.2d |

#### plugin-dev（~2.5d）

| # | 任务 | 设计章节 | 工作量 |
|---|------|---------|--------|
| P-M3-1 | AntiLoopGuard：互触发 / 短回复 / 退避三规则 | design-plugin §E.2 | 1.0d |
| P-M3-2 | groupConfigCache + WS 通知刷新 | §F.3 | 0.3d |
| P-M3-3 | autoReply 接收跳过 + 发送（内嵌 HulaApiClient） | §E.3 / M1 决策 | 0.5d |
| P-M3-4 | 多实例 AntiLoopGuard 状态限制文档化（运维约束） | §E.2 / S4 修订 | 0.1d |
| P-M3-5 | aichat-cli `send-message` 命令 | §G.2 | 0.3d |
| P-M3-6 | aichat-cli `group-config` 命令 | §G.3 | 0.3d |

#### frontend-dev（~1.5d）

| # | 任务 | 设计章节 | 工作量 |
|---|------|---------|--------|
| F-M3-1 | aiclaw 群配置 store + API 调用（loadConfigs/save/update） | design-frontend §3.5 | 0.3d |
| F-M3-2 | 桌面端 aiAssistantWindow 群聊设置子视图 | §3.5.1 | 0.5d |
| F-M3-3 | 移动端 AiAssistantGroupSettings.vue 页面 | §3.5.2 | 0.3d |
| F-M3-4 | autoReply 消息 UI 标记（灰色提示条） | §3.6 | 0.2d |
| F-M3-5 | i18n 文案（zh-CN + en） | §7 | 0.2d |

### 4.3 M3 验收标准

- ✅ 主人在 aiclaw 管理页能修改群配置（频率/日限/AI 互触发/@ 触发）
- ✅ 配置修改实时生效（aichat-node 内存缓存刷新）
- ✅ 配置变更广播到群内所有成员，UI 有 toast 提示
- ✅ 高频发言触发限流，aiclaw 自动发限流说明消息
- ✅ 限流说明消息标记 autoReply，其他 aiclaw 不响应
- ✅ AI-to-AI 协作时退避机制按表生效（0/5/15/30s）
- ✅ 短回复（<10 字符）连续 3 条触发跳过
- ✅ aichat-cli 命令可用

### 4.4 M3 提测内容

- backend-tester：群配置 CRUD、限流路径、autoReply 透传、Redis 滑动窗口
- ui-tester：群配置页面（桌面+移动）、配置变更 toast、autoReply 标记

---

## 五、里程碑 M4：联调 + 验收 + bugfix（~3-5 工作日）

### 5.1 目标

三方端到端联调，所有 M1-M3 功能在真实环境运行无误，完成验收。

### 5.2 任务清单

| 任务 | Owner | 内容 |
|------|-------|------|
| M4-1 | 三方 | 多 aiclaw 群聊端到端联调 |
| M4-2 | 三方 | 私聊 Agent Loop 模型联调（D1 决策范围） |
| M4-3 | backend-tester | 集成测试 + 性能压测（thinking 高并发） |
| M4-4 | ui-tester | 桌面端 + 移动端完整 UI 验收 |
| M4-5 | 三方 | bug 修复 |
| M4-6 | manager | 测试报告汇总 |

### 5.3 M4 验收标准

- ✅ 5 个真实场景跑通：
  1. 群聊 + 单 aiclaw + 用户连续提问
  2. 群聊 + 多 aiclaw + 用户提问触发协作
  3. 群聊 + 限流场景（频率/日限/AI 互触发/短回复/退避）
  4. 私聊 + aiclaw thinking 展示
  5. 群配置实时修改 + 多端同步
- ✅ 无 P0/P1 缺陷
- ✅ ISS-015 修复未受影响（回归测试）
- ✅ 性能指标达标（待 tester 提出基准）

---

## 六、各开发总工作量再核算

| Owner | M1 | M2 | M3 | M4 | 总计 |
|-------|----|----|----|----|------|
| server-dev | 1.0d | 2.0d | 2.5d | 1.0d | 6.5d ✓ |
| plugin-dev | 1.0d | 3.0d | 2.5d | 1.5d | 8.0d ✓ |
| frontend-dev | 0.5d | 1.5d | 1.5d | 0.6d | 4.1d ✓ |
| **总计** | **2.5d** | **6.5d** | **6.5d** | **3.1d** | **18.6d** |

按 3 人并行执行：
- M1 历时：~1.5 自然日
- M2 历时：~3 自然日
- M3 历时：~3 自然日
- M4 历时：~2 自然日
- **预计总历时：~9.5 自然日**（含里程碑回顾时间）

---

## 七、协作约定

### 7.1 daily 同步

每天上班时三位开发用 `set_summary` 更新当前进度，例如：
```
[REQ-004 M2] 已完成 P-M2-1 MessageHandler 重构，正在做 P-M2-2 thinkingId 回填，预计今天结束。
```

### 7.2 阻塞响应

- 跨组依赖阻塞：直接 `send_message` 给对方，抄送 manager
- 设计与实现冲突：先 `send_message` 给 manager
- 重大争议：暂停推进，向 manager 报告，manager 决定是否升级给 owner

### 7.3 提交规范

git commit 信息建议格式：
```
REQ-004(M1): <模块> <动作>

例如：
REQ-004(M2): server WS 添加 ThinkingEventHandler
REQ-004(M3): plugin AntiLoopGuard 互触发规则实现
```

### 7.4 里程碑回顾

每个里程碑结束后，manager 主持简短回顾（半天内）：
1. 实现与设计的差异（如有）
2. 是否需要调整后续设计
3. 提测准入检查
4. 决定是否进入下一里程碑

---

## 八、风险与应对

| 风险 | 应对 |
|------|------|
| openclaw Tool credential 行为与预期不符（X4） | plugin-dev 已有保底方案 B（实例池），不阻塞 |
| 高频群 thinking 落库性能问题 | M2 提测时压测，必要时引入异步落库 |
| 移动端 ThinkingPanel 复用桌面端有边缘 case | M2 ui-tester 提测时重点验证 |
| 三方 thinkingId 类型不一致（Long vs string） | M1 提测时三方对照测试用例 |
| Redis 滑动窗口在跨实例时计数偏差 | 标注限制，单实例部署不影响 |

---

## 附录：文档导航

- 需求：[需求.md](需求.md) v2.1
- 设计任务清单：[design-tasks.md](design-tasks.md)
- 后端设计：[design-server.md](design-server.md) v1.2
- 插件设计：[design-plugin.md](design-plugin.md) v1.2
- 前端设计：[design-frontend.md](design-frontend.md) v1.2
- 评审报告：[review-design-reviewer.md](review-design-reviewer.md)
- 可行性评估：[feasibility-frontend.md](feasibility-frontend.md) / [assessment-plugin-dev.md](assessment-plugin-dev.md)
