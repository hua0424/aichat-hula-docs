# REQ-004 群聊 — server 端详细设计

> Owner: server-dev  
> 输入: [需求.md](需求.md) v2.1 + design-tasks.md §2.1  
> 日期: 2026-05-20  
> 状态: v1.7（M4-2 完结：RetryPushConsumer 死会话保护 + cleanupSession 修复 + RateLimitChecker Redis 缓存 + thinkingId fallback + API 路径补全）

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
    status           TINYINT DEFAULT 0 COMMENT '状态：0=进行中 1=成功 2=错误 3=超时',
    error_code       VARCHAR(64) DEFAULT NULL COMMENT '错误码（rate_limit_exceeded / daily_limit_exceeded / short_reply_skip / timeout）',
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

**回写路径（S5）— 谁在何时写入 thinking_msg_rel：**

```
aiclaw agent loop 中调用 hula_send_message Tool
    │ 携带 thinkingId（从 THINKING_START 广播中提取）
    ▼
aichat-claw Plugin → REST POST /chat/msg
    │ Body 包含 { ..., extra: { thinkingId: "xxx" } }
    ▼
Server MessageController
    │ 1. 写入 im_message（正常消息落库）
    │ 2. 读取 extra.thinkingId
    │ 3. 写入 im_aiclaw_thinking_msg_rel(thinking_id, msg_id)
    │ 4. 更新 im_aiclaw_thinking.has_response = 1
    ▼
完成：thinking 与消息的关联建立
```

**与 plugin-dev 对齐点：**
- `hula_send_message` REST API 的 request body 新增可选字段 `extra?: { thinkingId?: string }`
- server 收到后识别 `extra.thinkingId`，如存在则执行关联写入
- 若 thinkingId 为空或不存在，跳过关联（兼容非 thinking 场景的普通消息）

> ⚠️ **待 plugin-dev 确认**：`hula_send_message` schema 是否接受 `extra.thinkingId` 字段？

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

**完整 API 路径（经 Gateway 路由）**：

| 内部路径 | Gateway 路径 | 说明 |
|---------|-------------|------|
| `GET /aiclaw/group/config` | `GET /api/im/aiclaw/group/config` | 查询群配置 |
| `PUT /aiclaw/group/config` | `PUT /api/im/aiclaw/group/config` | 更新群配置 |
| `POST /thinking/start` | 不暴露（ws-server 内部调用） | 创建 thinking 记录 |
| `POST /thinking/delta` | 不暴露 | 追加 delta |
| `POST /thinking/end` | 不暴露 | 结束 thinking |
| `POST /thinking/error` | 不暴露 | 标记 thinking 错误 |
| `GET /thinking/room/{roomId}/members` | 不暴露 | 查询群成员列表 |

> Gateway 路由规则：`/api/im/**` → `lb://luohuo-im-server`，StripPrefix=1

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

**Token 有效期与刷新机制（I1）：**
- `connectionToken` 有效期：参考现有用户登录 token 机制（如 7 天）
- 刷新方式：aichat-node 在 token 过期前调用 `POST /aiclaw/refresh-token` 获取新 token
- 失效处理：token 过期后 aiclaw 自动进入 standby 状态，待重新激活后恢复
- 注：具体有效期由现有认证体系决定，设计阶段不单独定义

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
    │ WS THINKING_START(20) { fromUid, roomId, triggerMsgId }
    ▼
┌─────────────────┐
│ WebSocketHandler │ 接收 WS 消息
│ (现有 Handler)   │
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│ ThinkingStart Handler (M4 修正)                          │
│ 1. 创建 thinking 记录（落库），先生成 thinkingId          │
│    （确保限流拒绝时 thinkingId 已存在，用于 plugin 路由） │
│ 2. 【频率校验】Redis 检查 rate limit，超限 →              │
│    - 更新 thinking 记录 status=2, error_code             │
│    - 推 thinkingEnd { thinkingId, error: "rate_limit_..."}│
│      给 plugin（含有效 thinkingId，plugin 可路由 autoReply）│
│    - 不触发 adapter.chat()                               │
│ 3. 限流通过 → 广播 thinkingStart { thinkingId, ... }      │
│    到群内所有成员（含该 aiclaw 自己）                      │
└────────────────────────┬─────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    群成员A(前端)   群成员B(前端)   群成员C(aiclaw=fromUid)
         │               │               │
         │               │               ▼
         │               │    plugin 匹配 fromUid === selfUid
         │               │    提取 thinkingId 存入 session
         │               │               │
         │               │               ▼
         │               │    DELTA/END 携带 thinkingId
         │               │               │
         ▼               ▼               ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ 前端展示        │     │ 前端展示        │     │ ThinkingDelta   │
│ thinking 状态条 │     │ thinking 状态条 │     │ /End Handler    │
└─────────────────┘     └─────────────────┘     │ 1. 追加/结束    │
                                                │ 2. 广播到群成员 │
                                                └─────────────────┘
```

**推送路由（推送给谁）：**
- THINKING_START/DELTA/END 推送给群内**所有在线成员**（包括人类用户和其他 aiclaw）
- 每个事件 payload 携带 `fromUid` 字段，供前端区分是哪个 aiclaw 的 thinking

**thinkingId 回传机制（S1）：**
- THINKING_START 是 plugin→server 单向请求，server 生成 thinkingId 后无法直接回传给 plugin
- **方案**：server 将 thinkingId 放入 thinkingStart **广播 payload**（server→client 方向），群内所有成员（含该 aiclaw 自己）都能收到
- plugin 侧收到 thinkingStart 后，通过匹配 `fromUid === selfUid` 识别是自己的 session，从中提取 `thinkingId`
- 后续 THINKING_DELTA/END 的入参携带该 `thinkingId`，server 通过 thinkingId 反查 roomId 和 thinking 记录

**thinkingId fallback 索引（M4 收尾）：**
- 由于 race condition（openclaw 在 thinkingStart 广播到达前就产生 delta），plugin 可能发出 `thinkingId` 为空的 THINKING_DELTA/END
- **server-side fallback**：`ThinkingProcessor` 维护二级索引 `ConcurrentHashMap<aiclawUid:roomId, thinkingId>`
  - `handleStart` 创建 thinking 时写入索引
  - `handleDelta`/`handleEnd` 收到空 thinkingId 时，用 `req.getRoomId()` + `aiclawUid` 反查
  - 反查成功则正常处理，仍失败才丢弃
  - `handleEnd` / 超时扫描清理索引
- **plugin-side 双重保险**：aichat-node 内部维护 `pendingDeltas` 缓冲，待 thinkingId 回填后补发

**频率校验前置（S2）—— M4 修正：**
- THINKING_START Handler 中**先创建 thinking 记录生成 thinkingId**，再做频率校验（Redis 滑动窗口计数器）
- 若超限，更新 thinking 记录 `status=2, error_code="rate_limit_exceeded"`，然后推 `THINKING_END` 错误事件给该 aiclaw：
  ```json
  { "type": 22, "data": { "thinkingId": "186xxxxxxxx", "status": "error", "error": "rate_limit_exceeded" } }
  ```
- plugin 端通过 `thinkingId` 匹配 active session，再按 `error` code 触发对应 `sendAutoReply`
- 此举避免 agent loop 启动后（GPU/CPU 消耗）才发现被限流，与 plugin-dev 的 AntiLoopGuard 配合形成双层过滤
- thinking 记录保留 `status` + `error_code`，便于运维统计限流发生频率

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
    private String status;         // "complete" | "error"（M1-fix 新增）
    private String error;          // 异常详情，正常结束为空
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
    private String status;         // "complete" | "error"（M1-fix 新增）
    private String error;          // 异常详情，正常结束为空
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
    private String aiclawUid;       // 哪个 aiclaw 的配置变了
    private String roomId;
    private ConfigDTO config;

    @Data
    public static class ConfigDTO {
        private Integer rateLimitPerMinute;
        private Integer mentionRequired;
        private Integer dailyLimit;
        private Integer respondToAi;
    }
}
```

> 注：与 plugin-dev / frontend-dev 对齐，采用嵌套结构 `config: { ... }`，便于前端直接替换配置对象。

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
// ChatMessageResp.Message 新增 extra 字段（仅 WS payload，不落库）
@Data
public static class Message {
    // ... 现有字段（id, roomId, sendTime, type, body, messageMarks）...
    private Map<String, Object> extra;  // 扩展字段，autoReply 放在里面
}
```

**autoReply 在 WS payload 中的层级位置（I3 明确，M4 精确化）：**
- `extra` 是 `ChatMessageResp.Message` 的**字段**（与 `body`、`messageMarks` 同级）
- WS payload 中精确路径为 **`data.message.extra.autoReply`**（不是 `data.extra.autoReply`）
- 不是放在 body 内部
- 示例（完整 WS payload）：
  ```json
  {
    "type": "receiveMessage",
    "data": {
      "fromUser": { "uid": "10001", "name": "aiclaw-bot" },
      "message": {
        "id": "12345",
        "roomId": "30001",
        "type": 1,
        "body": { "content": "触发频率限制，本分钟内无法回复" },
        "extra": { "autoReply": true, "thinkingId": "186xxx" }
      }
    }
  }
  ```

**边界说明：**
1. `im_message` 表零改动（不存储 extra）
2. `extra` 字段仅在 WS 推送时存在，REST API 返回的消息 JSON 中也可能包含（用于 MCP Tool 调用场景）
3. autoReply 仅用于实时防循环，历史消息无需追溯
4. 限流说明消息**不计入**限流统计

> ✅ **跨组对齐（X2）已确认**：server-dev + plugin-dev + frontend-dev 三方一致。

#### 3.3.5 WS Session 清理修复（M4 收尾）

**问题**：WS 断开后 `USER_DEVICE_SESSION_MAP` / `SESSION_USER_MAP` / `SESSION_CLIENT_MAP` 未清理，导致死会话残留、推送路由无效、RetryPushConsumer 反复重推。

**根因**：`SessionManager.cleanupSession()` 的 guard 条件 `if (!session.isOpen())` 导致从 `ReactiveWebSocketHandler.doFinally` 调用时**永远跳过清理**（doFinally 时 session 仍 open）。

```java
// BUG: doFinally 时 session 仍 open → !isOpen() = false → 整个 cleanup 被跳过
public void cleanupSession(WebSocketSession session) {
    if (session != null && !session.isOpen()) {  // ← 永远 false
        session.close(...).doAfterTerminate(() -> { /* 清理 maps */ }).subscribe();
    }
}
```

**修复**：始终执行清理，如 session 仍 open 则先 close 再清理 maps：

```java
public void cleanupSession(WebSocketSession session) {
    if (session == null) return;
    Mono<Void> closeMono = session.isOpen()
            ? session.close(CloseStatus.GOING_AWAY)
            : Mono.empty();
    closeMono.subscribeOn(Schedulers.boundedElastic())
            .doAfterTerminate(() -> {
                // 始终清理 maps（SESSION_CLIENT_MAP / SESSION_USER_MAP / USER_DEVICE_SESSION_MAP）
                String clientId = SESSION_CLIENT_MAP.remove(sessionId);
                Long uid = SESSION_USER_MAP.remove(sessionId);
                if (clientId != null && uid != null) {
                    boolean isLastSession = cleanDeviceSession(uid, clientId, sessionId);
                    if (isLastSession) {
                        nacosSessionRegistry.removeDeviceRoute(uid, clientId);
                        syncOnline(uid, clientId, false);
                    }
                }
            }).subscribe();
}
```

**影响范围**：所有 WS 客户端断连场景（正常 close / 心跳超时 / 网络异常 / token 过期），修复前均存在 map 残留。

> ⚠️ RetryPushConsumer 实际有 `maxReconsumeTimes = 3`（RocketMQ 限制），并非无限重试。但死会话导致每条消息都触发 1 次无效 retry，放大了表象。

---

#### 3.3.6 RetryPushConsumer 死会话保护（M4-2 修正）

**问题**：`RetryPushConsumer` 在收到延迟重试消息时，对所有 uid 统一检查 in-flight set 并重新推送。若用户已下线（无活跃 WS 会话），重推必然失败，浪费 `maxReconsumeTimes = 3` 的宝贵重试次数。

**修复**：重试前增加用户在线状态检查，已下线则直接清理 in-flight set，不再重推：

```java
public void onMessage(NodePushDTO message) {
    String onlineUsersKey = PresenceCacheKeyBuilder.globalOnlineUsersKey().getKey();

    deviceUserMap.values().forEach(uid -> {
        // M4-2: 死会话保护 — 用户已下线则直接清理 in-flight，不再浪费重试
        Boolean isOnline = cachePlusOps.zIsMember(onlineUsersKey, uid);
        if (!Boolean.TRUE.equals(isOnline)) {
            log.info("用户已下线，跳过重试并清理 in-flight: uid={}, hashId={}", uid, message.getHashId());
            cachePlusOps.sRem(PassageMsgCacheKeyBuilder.build(uid), message.getHashId());
            return;
        }

        Boolean exist = cachePlusOps.sIsMember(PassageMsgCacheKeyBuilder.build(uid), message.getHashId());
        if (exist) {
            // 仅在线用户才重推
            pushService.sendPushMsg(message.getWsBaseMsg(), Arrays.asList(uid), message.getUid());
            // ... contactDao.refreshOrCreateActive
        }
    });
}
```

**设计要点**：
- 使用 `globalOnlineUsersKey`（ZSET）判断用户是否在线，与 `SessionManager.syncOnline()` 维护的权威状态一致
- 用户已下线时主动 `sRem` 清理 in-flight set，避免该消息 hash 长期残留
- 仅对仍在线的用户执行 `sendPushMsg` + `contactDao.refreshOrCreateActive`

---

### 3.4 防循环 DB 权威层

> **M4 收尾更新**：`AiclawRateLimitChecker` 已改为从 Redis 群配置缓存读取 `rateLimitPerMinute` / `dailyLimit`，fallback 到默认值（10/1000）。不再 hardcode。详见 §3.4.1。

#### 3.4.1 频率限制（默认 10 条/分钟，可配置）

**主校验：Redis 滑动窗口计数器**

**限流阈值来源（M4 收尾修正）**：`AiclawRateLimitChecker.check()` 从 Redis 群配置缓存读取 `rateLimitPerMinute` / `dailyLimit`，不再 hardcode 默认值。

```java
// AiclawRateLimitChecker — 限流配置读取链路
private Config resolveConfig(Long aiclawUid, Long roomId) {
    try {
        String cacheKey = "im:aiclaw:group:config:" + aiclawUid + ":" + roomId;
        Object cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            JSONObject json = JSONUtil.parseObj(cached.toString());
            return new Config(
                json.getInt("rateLimitPerMinute", DEFAULT_RATE_LIMIT),  // 10
                json.getInt("dailyLimit", DEFAULT_DAILY_LIMIT));        // 1000
        }
    } catch (Exception e) {
        log.warn("Failed to resolve aiclaw config from Redis: ...", e);
    }
    return new Config(DEFAULT_RATE_LIMIT, DEFAULT_DAILY_LIMIT);  // fallback
}
```

配置缓存由 `AiclawGroupConfigServiceImpl` 维护：
- `getConfig()` 读 DB 后写入 Redis（TTL 30min）
- `updateConfig()` 更新 DB 后刷新 Redis 缓存

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

#### 3.4.4 短回复 skip（服务端权威层）

**背景**：aichat-node 与 aichat-claw 不在同一进程（node 为 WS 客户端，claw 为 openclaw 插件），plugin 端无法可靠维护短回复状态 → 上收至 server 权威层。

**触发时机**：`ChatServiceImpl.sendMsg()` 在 `checkAndSaveMsg()` **之前**执行，避免短回复落库。

**算法**：
```java
// 仅当发送者为 aiclaw (userType=4) 且在群聊中时检查
if (sender.userType != 4 || !room.isRoomGroup()) return;

// 读取群配置（无记录用默认值：threshold=10, lookback=3）
int threshold = config.shortReplyThreshold;   // 短回复阈值（字符数）
int lookback  = config.shortReplyLookback;    // 检查最近 N 条

// 查询该 aiclaw 在该群最近 N 条消息的内容长度
List<Integer> lengths = messageMapper.selectRecentMsgLengths(
    aiclawUid, roomId, lookback);

// 最近 N 条全部 < threshold → 拒绝
if (lengths.size() >= lookback && lengths.stream().allMatch(l -> l < threshold)) {
    // 有 thinkingId 时 WS 广播 thinkingEnd(error=short_reply_skip)
    // 然后抛 BizException("short_reply_skip")
}
```

**SQL**：
```sql
SELECT CHAR_LENGTH(content) AS msg_length
FROM im_message
WHERE from_uid = #{fromUid} AND room_id = #{roomId} AND status = 0
ORDER BY create_time DESC, id DESC
LIMIT #{limit}
```

**拒绝路径**：
1. `sendMsg()` 中抛 `BizException("short_reply_skip")`
2. 若 `request.extra.thinkingId` 存在，先 `PushService.sendPushMsg()` 广播 `thinkingEnd(status=error, error="short_reply_skip")` 给群成员
3. aichat-node 收到 `thinkingEnd(error=short_reply_skip)` 后触发 `sendAutoReply`

**Error Code 约定**（供 plugin-dev 参考）：
| Error Code | 场景 | 触发位置 | thinkingId |
|-----------|------|---------|------------|
| `rate_limit_exceeded` | 频率超限（10条/分钟） | ws-biz ThinkingProcessor | **有**（M4 修正：先 create thinking 再限流检查） |
| `daily_limit_exceeded` | 日限超限（1000条） | ws-biz ThinkingProcessor | **有**（同上） |
| `short_reply_skip` | 连续短回复跳过 | im-biz ChatServiceImpl | 有（从 `request.extra.thinkingId` 透传） |

> M4 关键修正：限流拒绝场景下 thinkingEnd 必须携带有效 `thinkingId`，plugin 端才能通过 `thinkingId` 匹配 active session 并触发对应 `sendAutoReply`。因此 ThinkingProcessor 在限流检查**之前**先调用 `createThinkingViaHttp` 生成 thinking 记录，拒绝时再 `markErrorViaHttp` 更新 `status=2` + `error_code`。

---

### 3.5 缓存设计

#### 3.5.1 Redis Key 设计

> I4：Redis key 分隔符统一使用英文冒号 `:`，uid/roomId 等变量值中**不应出现冒号**。若业务值可能含冒号，使用 `String.join(":", prefix, uid, roomId)` 前先对值做校验或转义。

```
# 群级配置缓存
im:aiclaw:group:config:{aiclawUid}:{roomId}  → JSON(AiclawGroupConfig)
TTL: 30min（配置变更时主动刷新）

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

#### 3.5.2 配置更新后刷新 + WS 广播

```java
@Transactional
public void updateConfig(AiclawGroupConfigUpdateReq req) {
    // 1. 更新 DB
    configDao.updateById(...);

    // 2. 刷新 Redis 缓存（更新而非删除，避免穿透）
    AiclawGroupConfigResp cachedResp = BeanUtil.copyProperties(config, AiclawGroupConfigResp.class);
    stringRedisTemplate.opsForValue().set(
        buildConfigCacheKey(aiclawUid, roomId), JSONUtil.toJsonStr(cachedResp), CONFIG_CACHE_TTL);

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
| X4 | aichat-claw Token 上下文 | aiclaw 独立 token（`connectionToken`）鉴权；群配置 PUT 校验 owner/aiclaw 本人 | ⚠️ 已知限制：OpenclawAdapter 未传 thinkingId 给 openclaw gateway → `has_response=0`，需 4 层联动修，后续迭代 |
| X5 | 群配置 WS 通知 payload | `WSGroupConfigChange` 含全部配置字段，推送群内**所有在线成员** + aichat-node | ✅ 已对齐 |
| X6 | 上下文窗口归属 | **不需要 server 提供**，openclaw 自身维护 conversation history | ✅ 已关闭 |

---

## 六、风险与降级方案

| 风险 | 等级 | 影响 | 降级方案 |
|------|------|------|---------|
| 高频群 COUNT(*) 性能问题 | 🔶 中 | 防循环 SQL 慢查询 | 切换为 Redis 滑动窗口计数器，DB 仅做对账 |
| 多 aiclaw 并发 thinking 串流 | 🔶 中 | 前端显示混乱 | thinking WS payload 必须含 `aiclawUid`，前端按 uid 隔离 thinking 状态 |
| WS session 断连清理失败 | 🔴 高 | 死会话残留、推送路由无效、retry 浪费 | M4 已修：`cleanupSession` 去掉 `!isOpen()` guard，始终清理 maps |
| runtime JAR 版本管理 | 🔶 中 | 部署旧版 JAR 导致功能缺失 | 建议加 sha256 / version stamp 验证部署流程 |
| 新表 DDL 执行失败 | 🟢 低 | 部署阻塞 | 手动 SQL 文件在灰度环境先验证；执行前备份 |
| aiclaw 自动入群误判 | 🟢 低 | 非 aiclaw 被自动入群 | 查询 aiclaw 列表时加 Redis 缓存 + DB 双校验 |
| thinking content 过大 | 🟢 低 | TEXT 字段存储压力 | 设置 content 上限（如 64KB），超限截断 |

---

## 七、工作量分解

| 子任务 | 工作量 | 说明 |
|--------|--------|------|
| **DDL + SQL 文件** | 0.5d | 3 张表 + 索引 + docs/sql/ 脚本 |
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

| 文件 | 路径 | 说明 |
|------|------|------|
| design-server.md | `teamdocs/active/REQ-004-group_chat-群聊/` | 本文档 |
| im_aiclaw_group_config DDL | `luohuo-im/luohuo-im-biz/src/main/resources/db/changelog/` | 群配置表 DDL |
| AiclawGroupConfig.java | `luohuo-im/luohuo-im-entity/...` | 群配置 Entity |
| AiclawThinking.java | `luohuo-im/luohuo-im-entity/...` | Thinking Entity |
| AiclawGroupConfigController.java | `luohuo-im/luohuo-im-controller/.../chat/` | 群配置 REST API |
| AiclawGroupConfigServiceImpl.java | `luohuo-im/luohuo-im-biz/.../impl/` | 群配置 Service（含 Redis 缓存） |
| AiclawRateLimitChecker.java | `luohuo-ws/luohuo-ws-biz/.../service/` | 限流校验器（读 Redis 配置） |
| ThinkingProcessor.java | `luohuo-ws/luohuo-ws-biz/.../processor/` | Thinking WS 事件处理（含 thinkingId fallback） |
| ThinkingController.java | `luohuo-im/luohuo-im-controller/...` | Thinking REST API（ws-server 内部调用） |
| ThinkingService.java | `luohuo-im/luohuo-im-biz/.../service/` | Thinking 业务逻辑 |
| SessionManager.java | `luohuo-ws/luohuo-ws-biz/.../websocket/` | WS 会话管理（含 cleanupSession 修复） |
| WSReqTypeEnum.java | `luohuo-model/...` | WS 请求类型枚举 |
| WSRespTypeEnum.java | `luohuo-model/...` | WS 响应类型枚举 |
| AiclawThinkingCleanJob.java | `luohuo-im/luohuo-im-biz/...` | XXL-Job 清理任务 |
