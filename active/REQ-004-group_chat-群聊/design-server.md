# REQ-004 群聊 — server 端详细设计

> Owner: server-dev  
> 输入: [需求.md](需求.md) v2.1 + design-tasks.md §2.1  
> 日期: 2026-05-20  
> 状态: v1.1（对齐 manager + plugin-dev + frontend-dev 决策后修订）

---

## 一、概述

本设计覆盖 REQ-004 中 server 端（HuLa-Server / luohuo-im）的全部后端变更，包括：

- 3 张新表（`im_aiclaw_group_config` / `im_aiclaw_thinking` / `im_aiclaw_thinking_msg_rel`）
- 群邀请 API 适配 aiclaw 自动入群
- 群级配置 REST API + Redis 缓存 + WS 失效通知
- THINKING WS 事件（20/21/22）server 端处理与推送路由
- XXL-Job thinking 过期清理任务
- 防循环 DB 权威层（频率/日限统计）

**im_message / MessageTypeEnum 零改动**，所有变更仅围绕新表和新增 WS 类型展开。

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              HuLa-Server                                 │
│                                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌──────────┐ │
│  │ RoomController│    │ AiclawGroup  │    │ ThinkingEvent│    │ XXL-Job  │ │
│  │  /room/group │    │ ConfigController│   │ Handler      │    │ CleanJob │ │
│  │  /member     │    │  /aiclaw/group │   │  (20/21/22)  │    │          │ │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └────┬─────┘ │
│         │                  │                   │                 │       │
│  ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐          │       │
│  │ RoomAppService│   │ AiclawGroup │    │ ThinkingService│       │       │
│  │ .addMember() │   │ ConfigService│   │ .saveDelta()  │       │       │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘          │       │
│         │                  │                   │                 │       │
│  ┌──────▼──────────────────▼───────────────────▼─────────────────▼─────┐ │
│  │                         DAO / Mapper Layer                          │ │
│  │  UserApplyDao  GroupMemberDao  AiclawGroupConfigDao                │ │
│  │  AiclawThinkingDao  AiclawThinkingMsgRelDao  MessageDao            │ │
│  └──────┬──────────────────┬──────────────────┬────────────────────────┘ │
│         │                  │                  │                          │
│  ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐                  │
│  │   MySQL     │    │   Redis     │    │ PushService │                  │
│  │  (im_*)     │    │  (Config   │    │  WS 广播    │                  │
│  │             │    │   Cache)    │    │             │                  │
│  └─────────────┘    └─────────────┘    └─────────────┘                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                           ┌────────┴────────┐
                           ▼                 ▼
                     aichat-node        HuLa Client
                     (plugin-dev)       (frontend-dev)
```

---

## 三、详细设计

### 3.1 数据库设计

#### 3.1.1 im_aiclaw_group_config —— 群级 aiclaw 配置

```sql
CREATE TABLE im_aiclaw_group_config (
    id                    BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键',
    aiclaw_uid            BIGINT NOT NULL COMMENT 'aiclaw 的 uid',
    room_id               BIGINT NOT NULL COMMENT '群聊 room_id',
    rate_limit_per_minute INT UNSIGNED DEFAULT 10 COMMENT '频率限制（条/分钟），0=无限制',
    mention_required      TINYINT UNSIGNED DEFAULT 0 COMMENT '是否需要 @ 触发：0=否，1=是',
    daily_limit           INT UNSIGNED DEFAULT 1000 COMMENT '每日发言上限',
    respond_to_ai         TINYINT UNSIGNED DEFAULT 1 COMMENT '是否响应其他 aiclaw：0=否，1=是',
    short_reply_threshold INT UNSIGNED DEFAULT 10 COMMENT '短回复字符阈值（用于规则4）',
    short_reply_lookback  INT UNSIGNED DEFAULT 3 COMMENT '短回复检查最近 N 条',
    is_del                TINYINT UNSIGNED DEFAULT 0 COMMENT '逻辑删除：0=正常，1=删除',
    create_time           DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    update_time           DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    UNIQUE KEY uk_aiclaw_room (aiclaw_uid, room_id),
    INDEX idx_room_id (room_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='aiclaw 群聊配置表';
```

**与现有表关系：**
- `aiclaw_uid` → 逻辑关联 `im_aiclaw.uid`（无外键，aiclaw 表在业务服务）
- `room_id` → 逻辑关联 `im_room.id`
- `uk_aiclaw_room` 保证每个 aiclaw 在每个群中只有一条配置记录

#### 3.1.2 im_aiclaw_thinking —— thinking 记录

```sql
CREATE TABLE im_aiclaw_thinking (
    id               BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键',
    aiclaw_uid       BIGINT NOT NULL COMMENT '产生 thinking 的 aiclaw uid',
    room_id          BIGINT NOT NULL COMMENT '所属群聊 room_id',
    trigger_msg_id   BIGINT COMMENT '触发本次 thinking 的消息 ID（im_message.id）',
    content          TEXT NOT NULL COMMENT '完整思考文本',
    duration_ms      INT COMMENT '处理耗时（毫秒），THINKING_END 时回填',
    has_response     TINYINT UNSIGNED DEFAULT 0 COMMENT '是否产生了回复消息：0=否，1=是',
    is_del           TINYINT UNSIGNED DEFAULT 0 COMMENT '逻辑删除',
    create_time      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    INDEX idx_aiclaw_room (aiclaw_uid, room_id),
    INDEX idx_trigger_msg (trigger_msg_id),
    INDEX idx_create_time (create_time),
    INDEX idx_has_response_create (has_response, create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='aiclaw thinking 记录表';
```

**设计要点：**
- `has_response` 标记用于 XXL-Job 清理时快速区分 30d/180d 过期策略
- `content` 使用 TEXT 类型，thinking 内容通常较长（几百到几千字符）
- 无外键约束，避免影响 `im_message` 写入性能

#### 3.1.3 im_aiclaw_thinking_msg_rel —— thinking 与消息关联

```sql
CREATE TABLE im_aiclaw_thinking_msg_rel (
    thinking_id      BIGINT NOT NULL COMMENT 'thinking 记录 ID',
    msg_id           BIGINT NOT NULL COMMENT '关联的 im_message.id',
    create_time      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '关联建立时间',
    PRIMARY KEY (thinking_id, msg_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='thinking 与回复消息关联表';
```

**设计要点：**
- 联合主键避免重复关联
- 无单独自增 ID，减少索引开销
- `create_time` 用于按时间排序关联消息

#### 3.1.4 数据库迁移

项目当前**不使用 Liquibase / Flyway**，数据库变更通过 `docs/sql/` 目录下的手动 SQL 文件管理。

**新增文件**：`docs/sql/im_aiclaw_group_chat.sql`

```sql
-- REQ-004 aiclaw 群聊扩展表
-- 创建日期：2026-05-20

CREATE TABLE IF NOT EXISTS `im_aiclaw_group_config` (
  ...
);

CREATE TABLE IF NOT EXISTS `im_aiclaw_thinking` (
  ...
);

CREATE TABLE IF NOT EXISTS `im_aiclaw_thinking_msg_rel` (
  ...
);
```

> 注：参考现有 `docs/sql/im_aiclaw.sql` 格式和命名规范。

---

### 3.2 接口设计

#### 3.2.1 POST /room/group/member —— aiclaw 自动同意改造

**现有逻辑**（`RoomAppServiceImpl.addMember` 第 1049-1107 行）：
1. 校验房间存在、自己是群成员
2. 过滤已进群和已邀请的用户
3. 创建 `UserApply` 邀请记录（状态 UNTREATED，需被邀请人手动同意）
4. 推送 WS 通知给被邀请人和群管理员

**改造后逻辑：**

```java
public void addMember(Long uid, MemberAddReq request) {
    // ... 现有校验逻辑不变 ...

    // 新增：识别被邀请人中的 aiclaw
    Set<Long> aiclawUids = aiclawService.getAiclawUidsOfUser(uid); // 查询邀请人的 aiclaw 列表
    Set<Long> autoAgreeUids = new HashSet<>(validUids);
    autoAgreeUids.retainAll(aiclawUids);  // 被邀 uid 属于邀请人 aiclaw → 自动同意
    validUids.removeAll(autoAgreeUids);   // 剩余 uid 走原有邀请流程

    // 自动同意：直接入群，不走 UserApply
    if (!autoAgreeUids.isEmpty()) {
        batchAddGroupMembers(roomGroup, autoAgreeUids);
        // 发送入群事件
        publishGroupMemberAddEvent(roomGroup, autoAgreeUids);
    }

    // 原有流程：非 aiclaw 的邀请走 UserApply
    if (!validUids.isEmpty()) {
        // ... 原有 invite 逻辑 ...
    }
}
```

**判定条件：**
- 调用 `aiclawService.getAiclawUidsOfUser(inviterUid)` 获取邀请人拥有的所有 aiclaw uid
- 被邀请 uid 在该集合中 → 自动同意
- 需要 aiclaw 服务提供查询接口（或 Redis 缓存）

> ⚠️ **待 manager 决策（X4）**：`aiclawService.getAiclawUidsOfUser()` 是调用远程 RPC 还是查本地缓存？aiclaw 列表变更频率低，建议 Redis 缓存 + 变更通知刷新。

#### 3.2.2 群级配置 REST 接口

**Controller**: `AiclawGroupConfigController`

```java
@RestController
@RequestMapping("/aiclaw/group/config")
public class AiclawGroupConfigController {

    /**
     * 查询 aiclaw 在指定群的配置
     * GET /aiclaw/group/config?aiclawUid={uid}&roomId={roomId}
     */
    @GetMapping
    public R<AiclawGroupConfigResp> getConfig(@RequestParam Long aiclawUid,
                                                 @RequestParam Long roomId);

    /**
     * 更新 aiclaw 群配置（仅 aiclaw 主人）
     * PUT /aiclaw/group/config
     */
    @PutMapping
    public R<Void> updateConfig(@Valid @RequestBody AiclawGroupConfigUpdateReq request);
}
```

**权限校验：**
- `GET`：群成员可查询（用于前端展示）
- `PUT`：仅 aiclaw 主人（通过 aiclaw_uid → owner_uid 映射校验）

**Request / Response DTO：**

```java
@Data
public class AiclawGroupConfigUpdateReq {
    @NotNull private Long aiclawUid;
    @NotNull private Long roomId;
    @Min(0) private Integer rateLimitPerMinute;
    @Range(0, 1) private Integer mentionRequired;
    @Min(0) private Integer dailyLimit;
    @Range(0, 1) private Integer respondToAi;
}

@Data
public class AiclawGroupConfigResp {
    private Long aiclawUid;
    private Long roomId;
    private Integer rateLimitPerMinute;
    private Integer mentionRequired;
    private Integer dailyLimit;
    private Integer respondToAi;
}
```

#### 3.2.3 鉴权：aiclaw 独立 token

群配置接口支持**aiclaw 独立 token**鉴权（不依赖主人 token）：

**Token 解析链路：**
```
Authorization: Bearer {connectionToken}
    → 解析出 aiclawUid
    → 查 im_aiclaw 获取 owner_uid
    → 校验：调用者 uid == owner_uid 或 调用者 uid == aiclawUid
```

**权限矩阵：**
| 接口 | 主人 | aiclaw 本人 | 其他群成员 |
|------|------|------------|-----------|
| GET /aiclaw/group/config | ✅ | ✅ | ✅（只读） |
| PUT /aiclaw/group/config | ✅ | ✅ | ❌ |

**实现要点：**
- aichat-node 的 `connectionToken`（activate 流程获取）即为 aiclaw token
- 鉴权层新增 `AiclawTokenAuthenticationFilter`，识别 aiclaw token 并注入 `AiclawAuthentication`
- 群配置 PUT 接口使用 `@PreAuthorize("hasAiclawPermission(#request.aiclawUid)")` 注解校验

---

### 3.3 WS 事件设计

#### 3.3.1 枚举扩展

**WSReqTypeEnum**（plugins → server）：

```java
public enum WSReqTypeEnum {
    // ... 现有 ...
    STREAM_START(17, "流式消息开始"),
    STREAM_DELTA(18, "流式消息片段"),
    STREAM_END(19, "流式消息结束"),

    // 新增
    THINKING_START(20, "thinking 开始"),
    THINKING_DELTA(21, "thinking 增量"),
    THINKING_END(22, "thinking 结束"),
    ;
}
```

**WSRespTypeEnum**（server → client）：

```java
public enum WSRespTypeEnum {
    // ... 现有 ...
    STREAM_START("streamStart", "流式消息开始", WSStreamStart.class),
    STREAM_DELTA("streamDelta", "流式消息片段", WSStreamDelta.class),
    STREAM_END("streamEnd", "流式消息结束", WSStreamEnd.class),

    // 新增
    THINKING_START("thinkingStart", "thinking 开始", WSThinkingStart.class),
    THINKING_DELTA("thinkingDelta", "thinking 增量", WSThinkingDelta.class),
    THINKING_END("thinkingEnd", "thinking 结束", WSThinkingEnd.class),
    GROUP_CONFIG_CHANGE("groupConfigChange", "群配置变更", WSGroupConfigChange.class),
    ;
}
```

#### 3.3.2 THINKING 事件处理流程

```
plugins (aichat-node)
    │ WS THINKING_START(20)
    ▼
┌─────────────────┐
│ WebSocketHandler │ 接收 WS 消息
│ (现有 Handler)   │
└────────┬────────┘
         │ 根据 type=20/21/22 分发
         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ ThinkingStart   │     │ ThinkingDelta   │     │ ThinkingEnd     │
│ Handler         │     │ Handler         │     │ Handler         │
│                 │     │                 │     │                 │
│ 1. 创建 thinking│     │ 1. 追加 content │     │ 1. 回填 duration│
│    记录（落库）  │     │ 2. 广播 DELTA   │     │ 2. 标记完成     │
│ 2. 广播 START   │     │    到群成员     │     │ 3. 广播 END     │
│    到群成员     │     │                 │     │    到群成员     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 ▼
                    PushService.sendPushMsg()
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
                群成员A       群成员B       群成员C
                (含前端)      (含前端)      (aiclaw)
```

**推送路由（推送给谁）：**
- THINKING_START/DELTA/END 推送给群内**所有在线成员**（包括人类用户和其他 aiclaw）
- 每个事件 payload 携带 `aiclawUid` 字段，供前端区分是哪个 aiclaw 的 thinking

**DTO 设计：**

> 与 plugin-dev / frontend-dev 对齐定稿：WS payload 中消息 ID 统一使用 **String**（与现有 `ChatMessageResp.Message.id` 风格一致），Entity 中仍为 Long。

```java
// WSReqTypeEnum 20 → 入参（plugin → server）
@Data
public class WSThinkingStart {
    private String fromUid;        // aiclaw uid
    private String roomId;
    private String triggerMsgId;   // 触发消息 ID
}

// WSReqTypeEnum 21 → 入参（plugin → server）
@Data
public class WSThinkingDelta {
    private String thinkingId;     // server 生成的 thinking ID
    private String chunk;          // 增量内容
    private Integer seq;           // 序号（用于顺序校验）
    private String roomId;         // 可选冗余，方便调试
}

// WSReqTypeEnum 22 → 入参（plugin → server）
@Data
public class WSThinkingEnd {
    private String thinkingId;     // server 生成的 thinking ID
    private Integer durationMs;    // 处理耗时
    private String error;          // 异常信息（可选）
    private String roomId;         // 可选冗余，方便调试
}

// WSRespTypeEnum → 响应（server → client）
@Data
public class WSThinkingStartResp {
    private String thinkingId;     // server 生成的 thinking ID（THINKING_START 创建后回传）
    private String fromUid;        // aiclaw uid
    private String roomId;
    private String triggerMsgId;
}

@Data
public class WSThinkingDeltaResp {
    private String thinkingId;
    private String fromUid;
    private String roomId;
    private String chunk;
    private Integer seq;
}

@Data
public class WSThinkingEndResp {
    private String thinkingId;
    private String fromUid;
    private String roomId;
    private Integer durationMs;
    private String error;
}
```

> ⚠️ **跨组对齐（X1）**：THINKING_START payload 字段确认：
> - `fromUid` vs `aiclawUid`：统一使用 `aiclawUid` 更清晰
> - `triggerMsgId`：必须携带，用于前端关联"哪条消息触发的 thinking"
> - `thinkingId`：server 生成后回传，供后续 DELTA/END 关联

#### 3.3.3 群配置变更 WS 通知

当群配置通过 PUT 接口更新后，server 向群内所有成员（或仅相关 aiclaw 的 aichat-node）推送配置变更通知：

```java
@Data
public class WSGroupConfigChange {
    private Long aiclawUid;       // 哪个 aiclaw 的配置变了
    private Long roomId;
    private Integer rateLimitPerMinute;
    private Integer mentionRequired;
    private Integer dailyLimit;
    private Integer respondToAi;
}
```

推送目标：
- 群内所有在线成员的客户端（frontend 刷新配置展示）
- aiclaw 对应的 aichat-node（plugin-dev 侧刷新内存限流配置）

#### 3.3.4 autoReply 字段载体

**Manager 决策：采用 extra 包裹方案（WS payload only，不落库）**

- aichat-node 调用 `hula_send_message` REST API 时传 `extra: { autoReply: true, reason: "rate_limit" }`
- server 接收后**不落库**，仅在 WS 推送时透传到 payload
- 前端通过 `extra.autoReply` 识别并跳过触发 agent loop

**实现：**

```java
// ChatMessageResp.Message 新增 extra 字段
@Data
public static class Message {
    // ... 现有字段 ...
    private Map<String, Object> extra;  // 扩展字段，autoReply 放在里面
}
```

**边界说明：**
1. `im_message` 表零改动（不存储 extra）
2. autoReply 仅用于实时防循环，历史消息无需追溯
3. 限流说明消息**不计入**限流统计

> ✅ **跨组对齐（X2）已确认**：server-dev + plugin-dev + frontend-dev 三方一致。

---

### 3.4 防循环 DB 权威层

#### 3.4.1 频率限制（10 条/分钟）

**主校验：Redis 滑动窗口计数器**

```
Key: im:aiclaw:rate:{aiclawUid}:{roomId}:{yyyyMMddHHmm}  → int
TTL: 2min
```

每次发言时 INCR，查询时累加最近 2 个 bucket。

**兜底：DB 对账（非实时，低频）**

```sql
-- 日终对账用，不作为实时校验
SELECT COUNT(*) FROM im_message
WHERE from_uid = #{aiclawUid}
  AND room_id = #{roomId}
  AND type = 1
  AND create_time >= DATE_SUB(NOW(), INTERVAL 1 MINUTE)
  AND is_del = 0;
```

**索引策略：**
- **不加 im_message 新索引**（保护高频写入表性能）
- 如需对账，利用现有 `idx_from_uid`（如有）或全表扫描（日终低频可接受）

#### 3.4.2 每日上限（1000 条）

**主校验：Redis 日计数器**

```
Key: im:aiclaw:daily:{aiclawUid}:{roomId}:{yyyyMMdd}  → int
TTL: 25h
```

每日一个 bucket，每次发言 INCR，达到阈值拒绝。

**兜底：DB 对账**

```sql
SELECT COUNT(*) FROM im_message
WHERE from_uid = #{aiclawUid}
  AND room_id = #{roomId}
  AND type = 1
  AND DATE(create_time) = CURDATE()
  AND is_del = 0;
```

#### 3.4.3 限流触发后的响应

当 aichat-node 调用 server 接口（如 MCP Tool 发送消息）触发限流时：

**HTTP 响应：**
```json
{
  "code": 429,
  "msg": "rate_limit_exceeded",
  "data": {
    "limitType": "per_minute|daily",
    "retryAfter": 45
  }
}
```

**aiclaw 自动回复限流说明消息：**
- aichat-node 收到 429 后，自动构造限流说明消息
- 消息 WS payload 携带 `autoReply: true`
- 该消息**不计入**限流统计（避免恶性循环）

---

### 3.5 缓存设计

#### 3.5.1 Redis Key 设计

```
# 群级配置缓存
im:aiclaw:group:config:{aiclawUid}:{roomId}  → JSON(AiclawGroupConfig)
TTL: 1h（配置变更时主动失效）

# aiclaw 主人关系缓存（用于权限校验）
im:aiclaw:owner:{aiclawUid}  → ownerUid
TTL: 24h

# 防循环计数器（频率）
im:aiclaw:rate:{aiclawUid}:{roomId}:{yyyyMMddHHmm}  → int
TTL: 2min

# 防循环计数器（日限）
im:aiclaw:daily:{aiclawUid}:{roomId}:{yyyyMMdd}  → int
TTL: 25h
```

**CacheKeyBuilder 实现：**

```java
@Component
public class AiclawGroupConfigCacheKeyBuilder implements CacheKeyBuilder {
    @Override
    public String getModular() { return "im"; }
    @Override
    public String getTable() { return "aiclaw_group_config"; }
    @Override
    public String getField() { return "aiclaw_uid.room_id"; }

    public CacheKey key(Long aiclawUid, Long roomId) {
        return key(aiclawUid, roomId);
    }
}
```

#### 3.5.2 配置更新后失效 + WS 广播

```java
@Transactional
public void updateConfig(AiclawGroupConfigUpdateReq req) {
    // 1. 更新 DB
    configDao.updateById(...);

    // 2. 失效 Redis 缓存
    cacheOps.del(configKeyBuilder.key(req.getAiclawUid(), req.getRoomId()));

    // 3. WS 广播配置变更
    pushService.sendPushMsg(
        WsAdapter.buildGroupConfigChange(req),
        groupMemberCache.getMemberUidList(req.getRoomId())
    );
}
```

---

### 3.6 已关闭问题

#### F1. 上下文窗口策略（默认最近 50 条）— ❌ 不需要 server 提供

**结论（plugin-dev 排查后）**：
- openclaw gateway 通过 sessionKey 自身维护 conversation history
- aichat-node 不需要传上下文，openclaw 内部处理
- **server 不需要提供上下文查询 API**，节省 ~0.5d 工作量

---

## 四、代码骨架

### 4.1 新增 Entity

```java
@Data
@TableName("im_aiclaw_group_config")
public class AiclawGroupConfig {
    @TableId(type = IdType.AUTO)
    private Long id;
    private Long aiclawUid;
    private Long roomId;
    private Integer rateLimitPerMinute;
    private Integer mentionRequired;
    private Integer dailyLimit;
    private Integer respondToAi;
    private Integer isDel;
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
}

@Data
@TableName("im_aiclaw_thinking")
public class AiclawThinking {
    @TableId(type = IdType.AUTO)
    private Long id;
    private Long aiclawUid;
    private Long roomId;
    private Long triggerMsgId;
    private String content;
    private Integer durationMs;
    private Integer hasResponse;
    private Integer isDel;
    private LocalDateTime createTime;
}

@Data
@TableName("im_aiclaw_thinking_msg_rel")
public class AiclawThinkingMsgRel {
    private Long thinkingId;
    private Long msgId;
    private LocalDateTime createTime;
}
```

### 4.2 新增 Controller

```java
@RestController
@RequestMapping("/aiclaw/group/config")
@Validated
public class AiclawGroupConfigController {

    @GetMapping
    public R<AiclawGroupConfigResp> getConfig(...);

    @PutMapping
    public R<Void> updateConfig(...);
}
```

### 4.3 新增 XXL-Job

```java
@Component
@Slf4j
public class AiclawThinkingCleanJob {

    @Resource
    private AiclawThinkingDao thinkingDao;

    /**
     * 清理过期 thinking 记录
     * 无回复：30 天；有回复：180 天
     */
    @XxlJob("cleanAiclawThinking")
    public void clean() {
        // 1. 清理无回复的 thinking（> 30 天）
        int count1 = thinkingDao.deleteNoResponseBefore(
            LocalDateTime.now().minusDays(30));
        XxlJobHelper.log("Deleted {} no-response thinking records", count1);

        // 2. 清理有回复的 thinking（> 180 天）
        int count2 = thinkingDao.deleteWithResponseBefore(
            LocalDateTime.now().minusDays(180));
        XxlJobHelper.log("Deleted {} with-response thinking records", count2);
    }
}
```

### 4.4 Thinking 事件 Handler

```java
@Component
public class ThinkingEventHandler {

    @Resource
    private ThinkingService thinkingService;
    @Resource
    private PushService pushService;

    public void handleStart(WSThinkingStart start) {
        Long thinkingId = thinkingService.create(start);
        WSThinkingStartResp resp = ...;
        pushService.sendPushMsg(resp, getRoomMemberUids(start.getRoomId()));
    }

    public void handleDelta(WSThinkingDelta delta, Long aiclawUid, Long roomId) {
        thinkingService.appendDelta(aiclawUid, roomId, delta.getChunk());
        WSThinkingDeltaResp resp = ...;
        pushService.sendPushMsg(resp, getRoomMemberUids(roomId));
    }

    public void handleEnd(WSThinkingEnd end, Long aiclawUid, Long roomId) {
        thinkingService.finalize(aiclawUid, roomId, end.getDurationMs());
        WSThinkingEndResp resp = ...;
        pushService.sendPushMsg(resp, getRoomMemberUids(roomId));
    }
}
```

---

## 五、跨组对齐项结论

| 编号 | 议题 | 结论 | 状态 |
|------|------|------|------|
| X1 | THINKING WS payload 字段 | payload 含 `fromUid`/`thinkingId`/`roomId`/`triggerMsgId`/`seq`/`durationMs`/`error?`；WS 中 ID 统一 String 类型 | ✅ 已对齐 |
| X2 | autoReply 载体 | **extra 包裹方案**（`extra: { autoReply: true }`），WS payload only，不落库 | ✅ 已对齐 |
| X3 | 防循环分层 | server 提供 Redis 计数器（频率/日限）+ DB 对账；aichat-node 做内存层（互触发/短回复/退避） | ✅ 已对齐 |
| X4 | aichat-claw Token 上下文 | aiclaw 独立 token（`connectionToken`）鉴权；群配置 PUT 校验 owner/aiclaw 本人 | ✅ 已对齐 |
| X5 | 群配置 WS 通知 payload | `WSGroupConfigChange` 含全部配置字段，推送群内**所有在线成员** + aichat-node | ✅ 已对齐 |
| X6 | 上下文窗口归属 | **不需要 server 提供**，openclaw 自身维护 conversation history | ✅ 已关闭 |

---

## 六、风险与降级方案

| 风险 | 等级 | 影响 | 降级方案 |
|------|------|------|---------|
| 高频群 COUNT(*) 性能问题 | 🔶 中 | 防循环 SQL 慢查询 | 切换为 Redis 滑动窗口计数器，DB 仅做对账 |
| 多 aiclaw 并发 thinking 串流 | 🔶 中 | 前端显示混乱 | thinking WS payload 必须含 `aiclawUid`，前端按 uid 隔离 thinking 状态 |
| 新表 DDL 执行失败 | 🟢 低 | 部署阻塞 | Liquibase 支持回滚；灰度环境先验证 |
| aiclaw 自动入群误判 | 🟢 低 | 非 aiclaw 被自动入群 | 查询 aiclaw 列表时加 Redis 缓存 + DB 双校验 |
| thinking content 过大 | 🟢 低 | TEXT 字段存储压力 | 设置 content 上限（如 64KB），超限截断 |

---

## 七、工作量分解

| 子任务 | 工作量 | 说明 |
|--------|--------|------|
| **DDL + Liquibase 迁移** | 0.5d | 3 张表 + 索引 + 迁移脚本 |
| **Entity + Mapper + Service 骨架** | 0.5d | MyBatis-Plus 生成 3 套 |
| **aiclaw 自动入群改造** | 0.5d | addMember 条件分支 + aiclaw 列表查询 |
| **群配置 REST API** | 1d | CRUD + 权限校验 + Redis 缓存 + WS 通知 |
| **THINKING WS 协议层** | 1d | Enum 扩展 + 3 个 Handler + DTO + 落库 |
| **thinking_msg_rel 关联** | 0.5d | MCP Tool 消息回写关联 |
| **XXL-Job 清理任务** | 0.5d | 定时清理 + 调度中心配置 |
| **防循环 DB 层** | 0.5d | Redis 计数器 + 限流响应协议（不加 DB 索引） |
| **autoReply 标记透传** | 0.3d | WS payload extra 扩展 |
| **aiclaw token 鉴权** | 0.5d | 新增鉴权 Filter + 权限注解 |
| **联调测试** | 1.5d | E2E 含多 aiclaw 并发 |
| **设计评审修订** | 0.5d | 按 reviewer 意见调整 |
| **合计** | **~6.5d** | 开发阶段（不含本设计阶段） |

---

## 附录：文件清单

| 文件 | 路径 |
|------|------|
| design-server.md | `teamdocs/active/REQ-004-group_chat-群聊/` |
| im_aiclaw_group_config DDL | `luohuo-im/luohuo-im-biz/src/main/resources/db/changelog/` |
| AiclawGroupConfig.java | `luohuo-im/luohuo-im-entity/...` |
| AiclawThinking.java | `luohuo-im/luohuo-im-entity/...` |
| AiclawGroupConfigController.java | `luohuo-im/luohuo-im-controller/...` |
| AiclawThinkingCleanJob.java | `luohuo-im/luohuo-im-biz/...` |
| WSReqTypeEnum.java | `luohuo-model/...` |
| WSRespTypeEnum.java | `luohuo-model/...` |
| ThinkingEventHandler.java | `luohuo-im/luohuo-im-biz/...` |
