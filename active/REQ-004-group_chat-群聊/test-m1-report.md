# REQ-004 M1 增量提测报告

> 测试方：backend-tester
> 日期：2026-05-20
> 里程碑：M1 协议层 + 基础设施
> server commit：`8ffd0bf3`
> plugin commit：`ccd61e4`
> frontend commit：`01e152d69`

---

## 一、DDL 执行与验证

### 1.1 执行结果

在 dev MySQL（`10.38.10.10:3306/hula`）执行 `docs/sql/req-004-group-chat.sql`，3 张表全部建表成功。

| 表名 | 状态 | 字符集 |
|------|------|--------|
| `im_aiclaw_group_config` | ✅ 创建成功 | utf8mb4 |
| `im_aiclaw_thinking` | ✅ 创建成功 | utf8mb4 |
| `im_aiclaw_thinking_msg_rel` | ✅ 创建成功 | utf8mb4 |

### 1.2 索引验证

**im_aiclaw_group_config：**
- ✅ `PRIMARY KEY (id)`
- ✅ `UNIQUE KEY uk_aiclaw_room (aiclaw_uid, room_id)`
- ✅ `KEY idx_room_id (room_id)`

**im_aiclaw_thinking：**
- ✅ `PRIMARY KEY (id)`
- ✅ `KEY idx_aiclaw_room (aiclaw_uid, room_id)`
- ✅ `KEY idx_trigger_msg (trigger_msg_id)`
- ✅ `KEY idx_create_time (create_time)`
- ✅ `KEY idx_has_response_create (has_response, create_time)`

**im_aiclaw_thinking_msg_rel：**
- ✅ `PRIMARY KEY (thinking_id, msg_id)`（联合主键）

### 1.3 与 design-server.md v1.2 的差异

| 项目 | 设计文档 | 实际 DDL | 影响 |
|------|---------|---------|------|
| `id` 自增 | `AUTO_INCREMENT` | 无 `AUTO_INCREMENT`，注释标注「雪花ID」 | 与 Entity `@TableId(type = IdType.INPUT)` 一致，应用层生成 ID ✅ |
| 时间精度 | `DATETIME(3)` | `DATETIME`（无精度） | 无功能影响，时间戳精度为秒级 |
| `im_aiclaw_thinking` 字段 | 设计文档无 `create_by` | 实际有 `create_by` | 扩展字段，无负面影响 |

**结论：DDL 验证通过 ✅**

---

## 二、Entity 与 Mapper 验证

### 2.1 代码位置

| 类 | 路径 |
|----|------|
| `AiclawGroupConfig` | `luohuo-im-entity/.../entity/AiclawGroupConfig.java` |
| `AiclawThinking` | `luohuo-im-entity/.../entity/AiclawThinking.java` |
| `AiclawThinkingMsgRel` | `luohuo-im-entity/.../entity/AiclawThinkingMsgRel.java` |
| `AiclawGroupConfigMapper` | `luohuo-im-biz/.../mapper/AiclawGroupConfigMapper.java` |
| `AiclawThinkingMapper` | `luohuo-im-biz/.../mapper/AiclawThinkingMapper.java` |
| `AiclawThinkingMsgRelMapper` | `luohuo-im-biz/.../mapper/AiclawThinkingMsgRelMapper.java` |

### 2.2 基础 CRUD 能力

3 个 Mapper 均继承 `BaseMapper<T>`，具备以下能力：
- `insert(T)` —— 插入
- `selectById(Serializable)` —— 按主键查询
- `updateById(T)` —— 按主键更新
- `deleteById(Serializable)` —— 按主键删除

### 2.3 发现的问题

#### ⚠️ P2：`AiclawThinkingMsgRel` 实体与复合主键不匹配

**问题描述：**
- `AiclawThinkingMsgRel` 继承 `SuperEntity<Long>`，包含 `id`、`createBy`、`isDel` 字段
- 但数据库表 `im_aiclaw_thinking_msg_rel` 的**主键是复合主键** `(thinking_id, msg_id)`，**无 `id` 列**
- `BaseMapper.selectById()` / `updateById()` 依赖单字段 `@TableId`，对复合主键行为未定义

**影响：**
- `selectById(id)` 无法正确查询（DB 无 `id` 列）
- `updateById(entity)` 无法正确更新
- 需改用自定义 Mapper XML 或 `@TableId` + `@TableField` 特殊处理

**建议：**
- 方案 A：`AiclawThinkingMsgRel` 不继承 `SuperEntity`，改为独立 POJO，手动标注 `@TableId` 在 `thinkingId` 上（MyBatis-Plus 不支持复合主键原生方案）
- 方案 B：增加自定义 Mapper XML，手写按复合主键查询/更新的方法
- 方案 C：新增无业务意义的自增 `id` 列，将 `(thinking_id, msg_id)` 改为 `UNIQUE KEY`

> 参考：MyBatis-Plus 官方文档明确说明「不支持复合主键」，建议使用业务层封装或方案 C。

#### ⚠️ P2：`AiclawThinking` 实体含 DB 不存在的字段

**问题描述：**
- `AiclawThinking` 继承 `Entity<Long>`，包含 `updateTime`、`updateBy`
- 但 `im_aiclaw_thinking` 表**无 `update_time` / `update_by` 列**

**影响：**
- MyBatis-Plus `FieldFill.INSERT_UPDATE` 自动填充时，INSERT/UPDATE 语句会包含不存在的列，导致 SQL 执行失败
- 例如 `insert(AiclawThinking)` 会生成 `INSERT INTO im_aiclaw_thinking (... , update_time, update_by) VALUES (...)`，报错 `Unknown column 'update_time'`

**建议：**
- 方案 A：`AiclawThinking` 改为继承 `SuperEntity<Long>`（去掉 updateTime/updateBy）
- 方案 B：DDL 增加 `update_time` / `update_by` 列

#### ℹ️ 备注：`AiclawGroupConfig` 完全匹配

- `AiclawGroupConfig` 继承 `Entity<Long>`，所有字段（含 updateTime/updateBy）均与 DB 列一一对应 ✅

---

## 三、WS 枚举值三方一致性校验

### 3.1 请求方向（plugin → server）

| 协议项 | Server `WSReqTypeEnum` | Plugin `protocol.ts` | 一致性 |
|--------|------------------------|----------------------|--------|
| THINKING_START | `20` | `20` | ✅ |
| THINKING_DELTA | `21` | `21` | ✅ |
| THINKING_END | `22` | `22` | ✅ |

> 注：Frontend 不发送 THINKING 事件，故 `WsRequestMsgType` 无需包含 20/21/22。

### 3.2 响应方向（server → client/plugin）

| 协议项 | Server `WSRespTypeEnum` | Plugin `protocol.ts` | Frontend `wsType.ts` | 一致性 |
|--------|------------------------|----------------------|----------------------|--------|
| thinkingStart | `"thinkingStart"` | `"thinkingStart"` | `"thinkingStart"` | ✅ |
| thinkingDelta | `"thinkingDelta"` | `"thinkingDelta"` | `"thinkingDelta"` | ✅ |
| thinkingEnd | `"thinkingEnd"` | `"thinkingEnd"` | `"thinkingEnd"` | ✅ |
| groupConfigChange | `"groupConfigChange"` | `"groupConfigChange"` | `"groupConfigChange"` | ✅ |

> Frontend 枚举常量名为 `AICLAW_GROUP_CONFIG_UPDATE`，但**字符串值为 `"groupConfigChange"`**，与 server/plugin 一致。retro-m1.md 指出的常量名问题已修复（commit `01e152d69`）。

### 3.3 WS DTO 结构核查

| DTO | Server 字段 | Plugin Payload | Frontend Payload | 说明 |
|-----|------------|----------------|------------------|------|
| `WSThinkingStart` | `fromUid: String, roomId: String, triggerMsgId: String` | `fromUid: number, roomId: number, triggerMsgId: string` | `fromUid: number, roomId: number, triggerMsgId?: string` | Server 用 String 防精度丢失，plugin/frontend 用 number（JS 安全整数范围内无精度问题） |
| `WSThinkingDelta` | `thinkingId: String, chunk: String, seq: Integer, roomId: String` | `thinkingId?: string, chunk: string, seq: number` | `thinkingId: string, roomId: number, chunk: string, seq: number` | 结构一致 |
| `WSThinkingEnd` | `thinkingId: String, durationMs: Integer, error: String, roomId: String` | `thinkingId?: string, durationMs: number, error?: string` | `thinkingId: string, roomId: number, durationMs?: number, status: 'complete'\|'error', errorMsg?: string` | Frontend 多了 `status` 字段（设计文档 §3.3.2 未提及），属前端内部状态标记，不影响协议交互 |
| `WSGroupConfigChange.ConfigDTO` | `rateLimitPerMinute, mentionRequired, dailyLimit, respondToAi` | `rateLimitPerMinute, mentionRequired, dailyLimit, respondToAi` | `rateLimitPerMinute, dailyLimit, respondToAi, mentionRequired` | 字段完全一致，嵌套结构 `config: {...}` 一致 ✅ |

**结论：WS 枚举值三方一致性验证通过 ✅**

---

## 四、问题汇总

| 编号 | 问题 | 级别 | 阻塞 M2？ | 建议处理方 |
|------|------|------|----------|-----------|
| M1-1 | `AiclawThinkingMsgRel` 实体含 `id` 字段但 DB 为复合主键 `(thinking_id, msg_id)` | P2 | 是（影响关联表写入） | server-dev |
| M1-2 | `AiclawThinking` 实体含 `updateTime`/`updateBy` 但 DB 无对应列 | P2 | 是（影响 thinking 落库） | server-dev |
| M1-3 | Plugin `ThinkingStartPayload.fromUid/roomId` 类型为 `number`，Server DTO 为 `String` | P3 | 否（JS 安全整数范围内） | plugin-dev（可选对齐） |
| M1-4 | Frontend `ThinkingEndPayload` 多了 `status` 字段 | P3 | 否（前端内部字段） | frontend-dev（确认设计） |

---

## 五、测试结论

| 测试项 | 结果 |
|--------|------|
| DDL 执行与验证 | ✅ **通过** |
| Entity / Mapper 结构验证 | ⚠️ **有条件通过**（存在 2 个 P2 问题需修复） |
| WS 枚举值三方一致性 | ✅ **通过** |

### 总体结论

**M1 提测部分通过**。WS 协议层和 DDL 均验证通过，但存在 **2 个 P2 级 Entity-DB 映射问题**，建议 server-dev 在 M2 启动前修复：

1. `AiclawThinkingMsgRel` 的复合主键与 `SuperEntity.id` 冲突
2. `AiclawThinking` 的 `updateTime`/`updateBy` 字段与缺失的 DB 列冲突

修复后无需重新提测，backend-tester 可在 M2 测试期间一并验证。

---

## 六、签名

| 角色 | 确认 |
|------|------|
| backend-tester | 测试执行完成，报告已提交 |
| 下一步 | 等待 manager / server-dev 对 M1-1 / M1-2 的修复决策 |
