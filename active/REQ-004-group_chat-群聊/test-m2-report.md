# REQ-004 M2 增量提测报告

> 测试方：backend-tester
> 日期：2026-05-20
> 里程碑：M2 Agent Loop + Thinking 落库 + 展示
> server commit：`b4ed3241`（含编译修复）
> plugin commit：`6495ea5`
> frontend commit：`896f1b848`

---

## 一、部署状态

| 项目 | 状态 |
|------|------|
| server 构建 | ✅ 通过（M2 + M1-fix 编译修复） |
| im.jar 部署 | ✅ 已重启 runtime 容器，服务正常启动 |
| ws.jar 部署 | ✅ 已重启 runtime 容器，服务正常启动 |
| dev MySQL DDL | ✅ M1 已执行，M2 无新增 DDL |

---

## 二、测试项与结果

### 2.1 入群自动同意（S-M2-4）

**测试方法：** 代码审查 + 逻辑验证

**代码审查结果：**
- `RoomAppServiceImpl.addMember()` 第 1074-1082 行：
  - ✅ 调用 `getAiclawUidsOfUser(uid)` 获取邀请人 aiclaw 列表
  - ✅ `autoAgreeUids = validUids ∩ aiclawUids` 交集计算正确
  - ✅ 自动同意走 `batchAddAiclawMembers()`，跳过 UserApply
  - ✅ 非 aiclaw 继续走原有邀请流程
- `batchAddAiclawMembers()` 第 1139-1168 行：
  - ✅ 已在群中则跳过
  - ✅ 事务内写入 group_member + 创建 contact
  - ✅ 缓存失效（memberList/exceptMemberList/Presence）
  - ✅ 发布 `GroupMemberAddEvent`

**API 实测限制：**
- 测试用户现有群为「官方群聊」，无法移除成员重新测试
- 新建测试群时，aiclaw 作为初始成员被直接加入（走 group create 路径，非 addMember 路径）
- **结论：代码逻辑正确，因测试数据限制未做端到端 API 验证**

---

### 2.2 Thinking 落库（S-M2-1/2）

**测试方法：** DB 直接操作 + 代码审查

**DB schema 验证：**
- ✅ `im_aiclaw_thinking` 表结构正确，支持 INSERT/UPDATE
- ✅ 手动 INSERT 测试通过：
  ```sql
  INSERT INTO im_aiclaw_thinking (...)
  VALUES (999999999001, 140789091499520, 163347643904512, 12345, 'Test content', 0, 0, NOW(), 10937855681024)
  ```
- ✅ 手动 UPDATE 测试通过（content 追加 + duration_ms 回填）
- ✅ 雪花 ID 字段（BIGINT）存储正常

**ThinkingService 代码审查：**
- ✅ `create()`：生成雪花 ID → builder 构造 → insert
- ✅ `appendDelta()`：selectById → content 拼接 → updateById
- ✅ `finalize()`：selectById → durationMs 回填 → updateById
- ⚠️ `appendDelta()` 中 seq 不持久化到 DB（仅 WS 顺序校验，符合设计）

**内部 HTTP 端点测试：**
- `/thinking/start` 等端点部署在 IM 服务（port 18763），设计为 ws-server 内部调用
- 直接调用返回 `ContextUtil 不存在租户编号`（内部服务缺少网关透传的租户上下文）
- **此为微服务架构预期行为，非缺陷**

---

### 2.3 WS 推送路由（S-M2-3）

**测试方法：** 代码审查

**ThinkingProcessor 代码审查：**
- ✅ `handleStart()`：HTTP 调用 IM 创建 thinking → 缓存 ThinkingContext → 广播 thinkingStart（含 thinkingId）
- ✅ `handleDelta()`：校验 activeThinking → HTTP 追加 delta → 广播 thinkingDelta
- ✅ `handleEnd()`：移除 activeThinking → HTTP finalize → 广播 thinkingEnd（含 status）
- ✅ `pushToMembers()`：使用 `pushService.sendPushMsg()` 向群成员列表广播
- ✅ `@Scheduled(fixedRate = 60000)`：5 分钟超时扫描，自动推送 `thinkingEnd(status="error")`
- ✅ `activeThinkings` 使用 `ConcurrentHashMap`，线程安全

---

### 2.4 hula_send_message + thinking_msg_rel 回写（S-M2-5）

**测试方法：** API 调用 + DB 验证

**测试结果：**

| 测试项 | 结果 | 说明 |
|--------|------|------|
| 发送消息带 `extra.thinkingId` | ✅ API 接受 | `POST /api/im/chat/msg` 请求 body 接受 `extra` 字段 |
| `im_message.extra` 存储 | ❌ **未实现** | `extra.thinkingId` 未写入 `im_message` 表 |
| `im_aiclaw_thinking_msg_rel` 写入 | ❌ **未实现** | 关联表未自动插入 |
| `im_aiclaw_thinking.has_response` 更新 | ❌ **未实现** | has_response 未自动更新为 1 |

**验证过程：**
1. 发送消息：`{"roomId":"1","msgType":1,"body":{"content":"test"},"extra":{"thinkingId":"999999999001"}}`
2. 消息发送成功，返回 msgId=163351385225728
3. DB 查询 `im_message`：extra 字段不含 `thinkingId`
4. DB 查询 `im_aiclaw_thinking_msg_rel`：无新记录
5. DB 查询 `im_aiclaw_thinking.has_response`：仍为 0

**代码审查：**
- 搜索 server 全库，无代码在消息发送流程中读取 `extra.thinkingId` 并写入关联表
- ChatController / MessageService 未处理 `extra.thinkingId`

**结论：`thinking_msg_rel` 关联回写逻辑在 server 端未实现。**

---

### 2.5 跨组协议一致性回归

**测试方法：** 代码对照

| 协议项 | Server | Plugin | Frontend | 一致性 |
|--------|--------|--------|----------|--------|
| THINKING_START | 20 | 20 | — | ✅ |
| THINKING_DELTA | 21 | 21 | — | ✅ |
| THINKING_END | 22 | 22 | — | ✅ |
| thinkingStart | "thinkingStart" | "thinkingStart" | "thinkingStart" | ✅ |
| thinkingDelta | "thinkingDelta" | "thinkingDelta" | "thinkingDelta" | ✅ |
| thinkingEnd | "thinkingEnd" | "thinkingEnd" | "thinkingEnd" | ✅ |
| groupConfigChange | "groupConfigChange" | "groupConfigChange" | "groupConfigChange" | ✅ |
| WSThinkingEnd.status | String("complete"/"error") | String | "complete" \| "error" | ✅ |
| thinkingId | String | string | string | ✅ |

---

## 三、问题汇总

| 编号 | 问题 | 级别 | 阻塞 M3？ | 归属 |
|------|------|------|----------|------|
| M2-1 | `AiclawThinking.builder().id()` 编译失败（`SuperEntity` 字段不在 Lombok `@Builder` 中） | **P1** | 是（已修复 `b4ed3241`） | server-dev |
| M2-2 | `thinking_msg_rel` 关联回写未实现：消息发送时未读取 `extra.thinkingId`，未写入关联表，未更新 `has_response` | **P1** | 是 | server-dev |
| M2-3 | 新建测试群（roomId=163347643904512）出现 `TooManyResultsException`，message API 对该群报错 | P3 | 否（现有数据问题） | 数据层 |
| M2-4 | Thinking 内部端点直接调用缺租户上下文（`ContextUtil 不存在租户编号`） | P3 | 否（内部服务设计，ws→im 调用有上下文） | 架构设计 |

---

## 四、测试结论

| 测试项 | 结果 |
|--------|------|
| 入群自动同意代码逻辑 | ✅ **通过**（代码审查） |
| Thinking 落库 DB 层 | ✅ **通过**（DB 操作 + 代码审查） |
| WS 推送路由代码逻辑 | ✅ **通过**（代码审查） |
| thinking_msg_rel 关联回写 | ❌ **未实现** |
| 跨组协议一致性 | ✅ **通过** |

### 总体结论

**M2 部分通过，发现 1 个 P1 未实现项（M2-2）。**

`thinking_msg_rel` 关联表回写是 M2 设计文档 §3.1.3 / §S5 明确要求的 server 端职责，但当前代码中消息发送流程（ChatController/MessageService）未处理 `extra.thinkingId`：
- 未读取请求 body 中的 `extra.thinkingId`
- 未写入 `im_aiclaw_thinking_msg_rel`
- 未更新 `im_aiclaw_thinking.has_response = 1`

**建议：** 在 `MessageController` 或 `MessageService` 的消息发送流程中，检查 `extra.thinkingId`，如存在则：
1. 调用 `AiclawThinkingMsgRelMapper.insertIgnore(rel)`
2. 调用 `AiclawThinkingMapper.updateHasResponse(thinkingId, 1)`

修复后做轻量复测即可（发送带 thinkingId 的消息，验证 DB 两张表）。

---

## 五、签名

| 角色 | 确认 |
|------|------|
| backend-tester | 测试执行完成，报告已提交 |
| 下一步 | 等待 server-dev 实现 M2-2 `thinking_msg_rel` 关联回写 |
