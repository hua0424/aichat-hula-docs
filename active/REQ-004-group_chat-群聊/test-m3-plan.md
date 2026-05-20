# REQ-004 M3 增量提测计划（草稿）

> 测试方：backend-tester
> 里程碑：M3 防循环 + 群配置 + autoReply
> 状态：草稿（待 M2-2 修复完成后正式执行）

---

## 一、测试范围

### 1.1 server-dev

| 任务 | 设计章节 | 测试重点 |
|------|---------|---------|
| S-M3-1 AiclawGroupConfigController | §3.2.2 | GET/PUT API、权限校验（主人/aiclaw/其他成员） |
| S-M3-2 Redis 滑动窗口 | §3.4.1/3.4.2 | 频率限制(10条/分钟)、日限(1000条)、计数正确性 |
| S-M3-3 THINKING_START 前置限流 | §3.4 | 超限返回 error 事件、限流阈值边界 |
| S-M3-4 autoReply WS 透传 | §3.3.4 | extra 字段不落库、WS payload 包含 autoReply |
| S-M3-5 群配置 WS 广播 | §3.3.3 | groupConfigChange 事件、嵌套 ConfigDTO、推送范围 |
| S-M3-6 aiclaw 列表 Redis 缓存 | §3.2.1 | 缓存命中、变更刷新 |
| S-M3-7 aiclaw token 独立鉴权 | §3.2.3 | token 解析、权限矩阵 |

### 1.2 plugin-dev

| 任务 | 设计章节 | 测试重点 |
|------|---------|---------|
| P-M3-1 AntiLoopGuard | §E.2 | 互触发规则、短回复(<10字符)连续3条、退避(0/5/15/30s) |
| P-M3-2 groupConfigCache | §F.3 | WS 通知刷新、内存缓存同步 |
| P-M3-3 autoReply | §E.3 | 接收跳过、发送（内嵌 HulaApiClient） |
| P-M3-4 多实例限制文档 | §E.2 | 运维约束文档化 |
| P-M3-5/P-M3-6 aichat-cli 命令 | §G.2/G.3 | send-message、group-config 命令可用性 |

### 1.3 frontend-dev

| 任务 | 设计章节 | 测试重点（ui-tester 负责） |
|------|---------|------------------------|
| F-M3-1 群配置 store | §3.5 | API 调用、状态管理 |
| F-M3-2 桌面端设置子视图 | §3.5.1 | aiAssistantWindow 群聊设置 UI |
| F-M3-3 移动端设置页面 | §3.5.2 | AiAssistantGroupSettings.vue |
| F-M3-4 autoReply UI 标记 | §3.6 | 灰色提示条、不触发 agent loop |
| F-M3-5 i18n | §7 | zh-CN + en 文案 |

---

## 二、测试用例设计（backend-tester）

### 2.1 群配置 CRUD（S-M3-1）

**TC-M3-1-1: 正常查询配置**
- 前置：群存在，aiclaw 在群中，有配置记录
- 操作：GET /aiclaw/group/config?aiclawUid={uid}&roomId={roomId}
- 期望：200，返回正确配置值

**TC-M3-1-2: 主人更新配置**
- 前置：主人已登录
- 操作：PUT /aiclaw/group/config（主人 token）
- 期望：200，DB 更新成功，Redis 缓存失效，WS 广播

**TC-M3-1-3: 非主人更新配置（拒绝）**
- 前置：其他群成员登录
- 操作：PUT /aiclaw/group/config（非主人 token）
- 期望：403 或无权限错误

**TC-M3-1-4: aiclaw 本人更新配置**
- 前置：aiclaw 独立 token 有效
- 操作：PUT /aiclaw/group/config（aiclaw token）
- 期望：200（权限矩阵允许 aiclaw 本人修改）

### 2.2 Redis 滑动窗口（S-M3-2）

**TC-M3-2-1: 频率限制边界**
- 前置：Redis 清空
- 操作：1 分钟内发送 10 条消息（边界值）
- 期望：前 10 条成功，第 11 条触发限流

**TC-M3-2-2: 频率限制跨分钟桶**
- 前置：在第 0 秒发送 5 条，第 59 秒发送 5 条
- 操作：第 61 秒再发送 6 条
- 期望：前 5 条成功（第 0 秒的桶已过期），第 6 条触发限流

**TC-M3-2-3: 日限边界**
- 前置：Redis 日计数器 = 999
- 操作：再发送 2 条消息
- 期望：第 1 条成功，第 2 条触发日限

**TC-M3-2-4: 限流消息不计入统计**
- 前置：autoReply 消息发送
- 操作：检查 Redis 计数器是否增加
- 期望：计数器不增加

### 2.3 THINKING_START 前置限流（S-M3-3）

**TC-M3-3-1: 正常 THINKING_START**
- 前置：未达到限流阈值
- 操作：WS 发送 THINKING_START
- 期望：正常处理，返回 thinkingId

**TC-M3-3-2: 超限 THINKING_START**
- 前置：已达到限流阈值
- 操作：WS 发送 THINKING_START
- 期望：返回 thinkingEnd(status="error", error="rate_limit")

### 2.4 autoReply WS 透传（S-M3-4）

**TC-M3-4-1: autoReply 消息不落库**
- 前置：触发限流，aiclaw 发送 autoReply 消息
- 操作：查询 im_message.extra
- 期望：extra 字段不含 autoReply（或 MessageExtra 无该字段）

**TC-M3-4-2: autoReply WS payload**
- 前置：aiclaw 发送 autoReply 消息
- 操作：监听 WS 推送
- 期望：payload.message.extra.autoReply = true

### 2.5 群配置 WS 广播（S-M3-5）

**TC-M3-5-1: 配置变更广播范围**
- 前置：群内有多个在线成员
- 操作：主人修改配置
- 期望：所有在线成员收到 groupConfigChange 事件

**TC-M3-5-2: 配置变更广播 payload**
- 前置：修改配置
- 操作：检查 WS payload
- 期望：包含 aiclawUid、roomId、config（嵌套对象）

### 2.6 跨组协议一致性（回归）

- [ ] WSRespTypeEnum: GROUP_CONFIG_CHANGE 常量值三方一致
- [ ] WSGroupConfigChange 字段名三方一致
- [ ] autoReply 字段层级位置三方一致（message.extra.autoReply）

---

## 三、测试环境

- dev MySQL: 10.38.10.10:3306/hula
- dev Redis: 待确认（runtime 容器内或独立）
- Gateway: localhost:18080
- WS: 待确认端口

---

## 四、风险与依赖

| 风险 | 应对 |
|------|------|
| Redis 环境未就绪 | 提前确认 runtime 容器是否包含 Redis |
| aiclaw 独立 token 鉴权链路复杂 | 与 server-dev 确认测试 token 获取方式 |
| 限流测试需要高频调用 | 准备脚本化测试，避免手工重复 |

---

## 五、待办

- [ ] M2-2 修复完成后，更新此计划为正式版
- [ ] 确认 Redis 连接信息
- [ ] 准备限流测试脚本（高频调用）
- [ ] 与 ui-tester 对齐 M3 UI 测试范围
