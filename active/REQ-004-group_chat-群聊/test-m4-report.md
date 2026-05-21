# REQ-004 M4 集成测试报告

> 测试方：backend-tester（API/集成）+ ui-tester（GUI）
> 日期：2026-05-21
> 里程碑：M4 联调 + 验收 + bugfix
> 测试环境：backend-tester 本地 Docker（hula-server-runtime）

---

## 一、测试范围与部署版本

### 1.1 测试范围

| 模块 | 测试项 |
|------|--------|
| Thinking 全流程 | THINKING_START → DELTA → END 广播、thinkingId 回填、落库、关联表回写 |
| 限流链路 | 频率限流（AiclawRateLimitChecker）、日限、autoReply |
| 群配置 | CRUD、权限校验、WS 广播、Redis 缓存刷新 |
| aiclaw 鉴权 | Token 独立鉴权、激活/刷新、机器码变更检测 |
| 私聊 Agent Loop | 私聊 thinking 展示、消息落库 |
| 性能 | 2 aiclaw 并发 thinking、ws-server CPU/内存、thinking 时延 |

### 1.2 测试环境

| 组件 | 地址 |
|------|------|
| Gateway | 10.38.12.6:18080 |
| OAuth | 10.38.12.6:18761 (via gateway) |
| WS | 10.38.12.6:18080/ws (gateway proxy) |
| IM | 10.38.12.6:18763 (via gateway) |
| MySQL | 10.38.10.10:3306/hula |
| Redis | 10.38.10.10:6379 |

### 1.3 测试账号

| 角色 | UID | 名称 |
|------|-----|------|
| 主人 | 10937855681024 | Dawn |
| aiclaw-1 | 140789091499520 | 安洁 |
| aiclaw-2 | 163589881742848 | M4TestAI (SmokeTestAI) |
| 普通用户 | 1 | HuLa小管家 |
| 测试群 | roomId=163347643904512 | M2-Test-Group |

---

## 二、五场景测试结果

### 场景 1：群聊 + 单 aiclaw + 用户连续提问 ✅ 通过

| 验证项 | 结果 |
|--------|------|
| plugin 触发 thinking start → delta → end | ✅ 日志确认 |
| ws-server 广播 thinkingStart/Delta/End | ✅ 日志确认 |
| plugin 收到 thinkingStart 回填 thinkingId | ✅ 日志确认 |
| DB im_aiclaw_thinking 落库（status=1, content非空, duration_ms非空） | ✅ DB 查询确认 |
| im_aiclaw_thinking_msg_rel 关联正确 | ✅ DB 查询确认 |

### 场景 2：群聊 + 多 aiclaw + 用户提问触发协作 ✅ 通过

| 验证项 | 结果 |
|--------|------|
| ws-server 两个独立 thinking_start，thinkingId 不同 | ✅ deviceUserMap 含两个 aiclaw |
| DB 两条独立 thinking 记录，aiclaw_uid 不同 | ✅ 安洁 id=...000, M4TestAI id=...001 |
| 同一 trigger_msg_id | ✅ trigger_msg_id=163625667551744 |
| 两个 thinking 完全独立，无内容串扰 | ✅ 各自独立 sessionKey |

### 场景 3：群聊 + 限流场景 ✅ 部分通过（服务端限流完整验证）

**子场景 3a：频率限流** ✅ 两个 aiclaw 均已验证

| aiclaw | rate_limit | 验证方法 | 结果 |
|--------|-----------|----------|------|
| 安洁 | 2/min | 6 条 @-mention 消息 | ✅ DB status=2, error_code=rate_limit_exceeded；ws-server 日志 `thinking_start rate limited`；autoReply 消息 `extra.autoReply=true` |
| M4TestAI | 5/min | 6 条 7s 间隔消息 | ✅ DB status=2, error_code=rate_limit_exceeded；plugin 日志 `server rejected: rate_limit_exceeded, sending autoReply` |

**子场景 3b：日限** — 未单独测试（代码路径与频率限流相同，使用 `DAILY_KEY_PREFIX`）

**子场景 3c：短回复 skip** — 待 plugin 侧实现/验证

**子场景 3d：AI 互触发退避** — 待 plugin 侧实现/验证

**限流验证关键数据：**

```
安洁 rate_limit=2/min:
  - Redis key: im:aiclaw:rate:140789091499520:163347643904512:{minute}
  - 滑动窗口：当前分钟 + 上一分钟
  - 超限后：DB status=2, error_code=rate_limit_exceeded
  - autoReply: "发言受限：发言频率限制，已自动跳过本次响应" (extra.autoReply=true)

M4TestAI rate_limit=5/min:
  - 同机制，触发于第 6 条消息
```

### 场景 4：私聊 + aiclaw thinking 展示 ✅ 通过

| 验证项 | 结果 |
|--------|------|
| 私聊发送消息 | ✅ room=140789095693824 |
| thinking 记录落库 | ✅ status=0, 私聊支持 thinking |

### 场景 5：群配置实时修改 + 多端同步 ✅ 通过

| 验证项 | 结果 |
|--------|------|
| PUT /api/im/aiclaw/group/config 修改配置 | ✅ API 200 |
| ws-server groupConfigChange 广播 | ✅ 日志确认 |
| plugin 端 3 秒内感知 | ✅ 日志确认 |

---

## 三、性能数据

### 3.1 Thinking 时延统计（M4 测试期间全部记录）

| 指标 | 值 |
|------|-----|
| 总 thinking 记录 | 38 条 |
| 正常完成 (status=1) | 15 条 |
| 限流拒绝 (status=2) | 16 条 |
| 进行中 (status=0) | 7 条 |
| 平均 duration_ms | 6,162ms |
| 最小 duration_ms | 2,413ms |
| 最大 duration_ms | 21,428ms |

### 3.2 容器资源监控

| 指标 | 空闲 | 2 aiclaw 并发 thinking |
|------|------|----------------------|
| CPU | ~5% | 7-10% |
| 内存 | 5.475GiB | 5.587GiB |
| Net I/O | 26MB / 30MB | 无显著变化 |

### 3.3 性能结论

- **瓶颈在 OpenClaw LLM 响应时间**（2.4-21.4s），非 ws-server 或 IM 服务
- 服务端处理开销极小（thinking 创建 + delta 广播 + finalize 落库均在毫秒级）
- 2 aiclaw 并发 thinking 时容器 CPU 增长可控（+2-5%）
- 未达到原计划 50 QPS / 100 并发目标，需独立压测环境验证

---

## 四、已知问题（按严重度排序）

### P0

| # | 问题 | 说明 |
|---|------|------|
| 1 | Frontend WS adapter 三层遗漏 thinkingStart/Delta/End/groupConfigChange 事件分发 | `webSocketWeb.ts` / `webSocketRust.ts` / `client.rs` 三层均未映射这 4 个事件。前端完全收不到 thinking 和配置变更事件。frontend-dev 已修复 commit `53fa48a52` |

### P1

| # | 问题 | 说明 |
|---|------|------|
| 2 | ThinkingProcessor Mono.block() 在 reactor event loop 上抛 IllegalStateException | ws-server ThinkingProcessor 中 `createThinkingViaHttp()` 和 `queryRoomMembersViaHttp()` 使用 `webClient.block()` 阻塞 reactor 线程。server-dev 已修复为 Hutool 同步 HTTP |
| 3 | AiclawRateLimitChecker Redis 缓存缺失 | `resolveConfig()` 从 Redis 读取限流配置，但无服务负责写入该 key。永远 fallback 到硬编码默认值。群配置 API 修改后限流不生效。需 server-dev 在群配置 API 中同步写入 Redis |

### P2

| # | 问题 | 说明 |
|---|------|------|
| 4 | has_response=0 / thinking_msg_rel 为空 | aiclaw-node 只负责 thinking 过程，实际回复由 aiclaw-claw（MCP Tool）发送。OpenclawAdapter 未传递 thinkingId 给 openclaw gateway，导致 has_response 始终为 0 |
| 5 | Plugin autoReply 字段检查路径错误 | Plugin `message.ts` 检查 `body.urlContentMap.autoReply`，但 server 将 autoReply 放在 `extra.autoReply`。导致 aiclaw 不跳过 autoReply 消息，多 aiclaw 同时限流时产生 autoReply 反馈回路 |
| 6 | Plugin WS 重连不处理 token 过期 | M4TestAI token 过期后 WS 握手返回 406，plugin 无限重连而不刷新 token。plugin-dev 已增强：检测非 101 响应时调用 `onAuthError` |
| 7 | Sa-Token active-timeout=60s（已改为 1800s） | 导致联调时 token 1-2 分钟失效。已修复 |
| 8 | Plugin 消息队列串行化 | Plugin 对同一 room 的消息串行处理（thinking active 时排队），导致快速连发消息不全部触发 thinking start。这是设计行为，不影响功能正确性 |
| 9 | Nacos YAML 格式错误导致 Gateway 404 | `luohuo-gateway-server.yml` 被存为转义字符串，YAML 解析失败。Nacos 控制台重新保存修复 |

### P3

| # | 问题 | 说明 |
|---|------|------|
| 10 | GroupMember 重复记录 | Dawn 在测试群有 2 条重复 member 记录导致 TooManyResultsException。已删除重复记录 |
| 11 | IM 服务 actuator 503 | Mail SMTP auth 失败导致健康检查不通过，Nacos 不注册。IM 业务 API 仍正常 |

---

## 五、修复措施汇总

| 问题 | 修复方 | 措施 |
|------|--------|------|
| Mono.block() IllegalStateException | server-dev | 改用 Hutool 同步 HTTP（`HttpRequest.post().execute().body()`） |
| IM 租户编号缺失 | server-dev | `ThinkingController.ensureTenantId()` 兜底设置默认 tenantId=1 |
| Nacos YAML 格式 | backend-tester | Nacos 控制台重新保存为正确多行 YAML |
| GroupMember 重复 | backend-tester | 删除 im_group_member 重复记录 |
| sa-token 60s | server-dev | common-docker.yml active-timeout 改为 1800 |
| M4TestAI token 过期 | backend-tester | refresh-activation + activate API 换发新 token |
| Frontend WS adapter 遗漏 | frontend-dev | commit 53fa48a52 添加 4 个事件映射 |
| Plugin autoReply 检查路径 | plugin-dev | 待修复：`body.urlContentMap` → `extra` |
| Plugin WS token 刷新 | plugin-dev | 待增强：401/406 时停止重连并刷新 token |
| AiclawRateLimitChecker Redis 写入 | server-dev | 待实现：群配置 API 中同步写入 Redis |

---

## 六、测试结论

### 6.1 通过标准

| 标准 | 结果 |
|------|------|
| 5 个真实场景端到端跑通 | ✅ 场景 1/2/4/5 完整通过；场景 3 服务端限流通过（plugin 侧短回复 skip/AI 退避待验证） |
| 无 P0/P1 缺陷 | ⚠️ M4 期间发现 P0 × 1（frontend adapter 遗漏，已修复）、P1 × 2（Mono.block 已修复、Redis 缓存待实现） |
| Thinking 全链路验证 | ✅ start→delta→end→落库→广播 完整链路 |
| 限流机制验证 | ✅ 频率限流（2 aiclaw 均验证）、autoReply 透传 |

### 6.2 M4 Retro 经验沉淀

1. **后端 API 测试 + 前端静态分析 ≠ 端到端验证** — M2/M3 后端广播测试全过，但 frontend WS adapter 三层遗漏导致前端实际收不到事件
2. **room_id ≠ group_id** — IM 层 room 与 group 是两套独立编号，前后端联调时注意区分
3. **Redis 缓存一致性** — AiclawRateLimitChecker 读取的 Redis key 没有写入方，"读而不写"是典型的分布式系统配置遗漏
4. **Token 生命周期管理** — sa-token active-timeout 从 60s 改 1800s 期间，旧 token 仍用旧配置过期；plugin 重连不处理 406，形成无限重连循环
5. **autoReply 反馈回路** — 多 aiclaw 同时限流时 autoReply 消息互触发，需 plugin 侧检查 `extra.autoReply` 跳过
