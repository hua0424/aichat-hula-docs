# REQ-003 非 owner 与 aiclaw 通讯 — 后端设计

> 状态：已评审，2026-03-26 补充前后端联调对齐细节
> 日期：2026-03-24（更新：2026-03-26）
> 编写：锋哥（Backend）
> 关联需求：`04-需求与设计/需求/REQ-003-非owner与aiclaw通讯.md`
> 关联架构：`04-需求与设计/设计/REQ-002-aichat-claw插件架构设计.md`

---

## 一、数据模型变更

### 1.1 im_aiclaw 表扩展

新增一个字段：

```sql
ALTER TABLE im_aiclaw ADD COLUMN public_persona TEXT DEFAULT NULL COMMENT '对外人设（系统 prompt）';
```

- 为空时不注入人设，只注入安全提示
- 修改后对新消息立即生效（plugins 每次从 WS payload 取最新值）

### 1.2 新建 im_aiclaw_friend_ext 表

存储 owner 为 aiclaw 的每个非 owner 好友配置的关系说明。**不复用 im_user_friend.remark**，因为 remark 是前端展示的好友备注，与 prompt 注入用的关系说明语义不同。

```sql
CREATE TABLE im_aiclaw_friend_ext (
  id          BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
  aiclaw_uid  BIGINT(20)          NOT NULL COMMENT 'aiclaw 的 im_user.id',
  friend_uid  BIGINT(20)          NOT NULL COMMENT '非 owner 好友的 im_user.id',
  relation_desc VARCHAR(200)      DEFAULT NULL COMMENT '关系说明，注入安全 prompt。为空时使用默认文案',
  create_time DATETIME(3)         NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  update_time DATETIME(3)         NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  create_by   BIGINT(20)          DEFAULT NULL,
  update_by   BIGINT(20)          DEFAULT NULL,
  is_del      TINYINT(1)          NOT NULL DEFAULT 0,
  tenant_id   BIGINT(20)          NOT NULL DEFAULT 1,
  PRIMARY KEY (id),
  UNIQUE KEY uk_aiclaw_friend (aiclaw_uid, friend_uid),
  KEY idx_aiclaw_uid (aiclaw_uid)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='aiclaw 好友扩展（关系说明）';
```

### 1.3 会话上下文隔离：无需存储

openclaw 原生通过 `sessionKey` 在 gateway 服务端维护独立的对话上下文。plugins 为每个用户构造不同的 sessionKey 即可实现隔离，**不需要新建表，不需要 Redis 存储**。

**sessionKey 构造规则：**

| 场景 | sessionKey 格式 | 说明 |
|------|----------------|------|
| owner 与 aiclaw 对话 | `aiclaw-{aiclawUid}-owner` | 沿用现有（改为语义更清晰的格式） |
| 非 owner 与 aiclaw 对话 | `aiclaw-{aiclawUid}-peer-{senderUid}` | 每个用户独立 session |

openclaw gateway 存储位置：`~/.openclaw/agents/<agentId>/sessions/<sessionKey>.jsonl`，自带每日凌晨 4:00 重置 + 可配空闲超时。

### 1.4 B24（sent_at）取消

`im_message.create_time` 由 MyBatis Plus 的 `LuohuoMetaObjectHandler` 在 INSERT 时自动填充 `LocalDateTime.now()`，即 server 收到 WS 消息并入库的时刻。与用户实际发送时间的误差仅为网络延迟（通常 < 50ms），plugins 防抖直接用 `create_time` 即可。**B24 无需开发。**

---

## 二、接口设计

### 2.1 新增 HTTP API

所有接口统一前缀 `/api/im/aiclaw/`，经网关路由到 IM 服务。

| 方法 | 路径 | 说明 | 鉴权 |
|------|------|------|------|
| GET | `/api/im/aiclaw/{uid}/conversations` | aiclaw 的对话列表（按好友分组，含最后一条消息） | owner 登录态 |
| GET | `/api/im/aiclaw/{uid}/conversations/{friendUid}/messages` | aiclaw 与某用户的聊天记录（分页） | owner 登录态 |
| GET | `/api/im/aiclaw/{uid}/friends` | aiclaw 的好友列表（不含 owner 本人） | owner 登录态 |
| DELETE | `/api/im/aiclaw/{uid}/friends/{friendUid}` | 移除 aiclaw 的某个好友 | owner 登录态 |
| PUT | `/api/im/aiclaw/{uid}/friends/{friendUid}/relation` | 设置某个好友的关系说明 | owner 登录态 |
| PUT | `/api/im/aiclaw/{uid}/persona` | 设置 aiclaw 对外人设 | owner 登录态 |

### 2.2 对话记录接口实现方式

**复用现有消息查询 Service**，加 owner 身份校验包装层：

```
AiclawConversationController
  → 校验当前用户是 aiclaw 的 owner
  → 通过 aiclaw_uid 查 im_room_friend 找到所有聊天房间
  → 复用 MessageService 的消息分页查询
```

不新建独立的消息查询逻辑，避免重复造轮子。

**对话列表响应格式**（2026-03-26 与前端对齐确认）：

```json
{
  "list": [
    {
      "friendUid": "xxx",
      "friendName": "小明",
      "friendAvatar": "url",
      "lastMessage": { "content": "...", "sendTime": 1711234567890, "type": 1 },
      "roomId": "xxx"
    }
  ]
}
```

### 2.3 现有接口改造（2026-03-26 补充）

以下现有接口需要补充字段，以支持前端识别 aiclaw 用户和展示 owner 信息：

| 接口 | 返回类 | 改动 | 用途 |
|------|--------|------|------|
| `GET /user/getById/{id}` | `UserInfoResp` | 新增 `userType` 字段；当 `userType=4` 时附加 `ownerInfo: { uid, name, avatar }` | 非 owner 查看 aiclaw 好友详情时显示"主人：xxx" |
| `GET /user/search` | `UserSearchResp` | 新增 `userType` 字段 | 搜索结果中前端根据 `userType=4` 显示 AI 标识 |
| `GET /room/notice/page` | `NoticeVO` | 新增 `receiverUserType` 字段 | 前端区分"加我为好友"和"加我的 AI 助理为好友"通知文案 |

> **说明：**`GET /user/friend/page` 返回的 `FriendResp` 已包含 `userType` 字段，无需改动。

### 2.4 移除好友的级联操作

`DELETE /api/im/aiclaw/{uid}/friends/{friendUid}` 完整级联：

```
1. 校验当前用户是 aiclaw 的 owner
2. 软删除 im_user_friend（双向记录，uid↔friend_uid）
3. 更新 im_room_friend 状态（de_friend 置为屏蔽）
4. 删除 im_aiclaw_friend_ext 对应记录
5. 通知 plugins 清理 openclaw session（通过 WS 事件或 MQ）
6. 清理 Redis 中该好友关系的缓存
7. 消息记录保留，不删除
```

---

## 三、好友申请转发机制

### 3.1 方案：target_id 存 aiclaw_uid，业务层转发 owner

**现有流程：**
```
申请人 → im_user_apply(target_id=被申请人uid) → 被申请人审批
```

**aiclaw 场景改造：**
```
申请人 → im_user_apply(target_id=aiclaw_uid) → server 识别 target 是 aiclaw
  → JOIN im_aiclaw 查 owner_uid → WS 推送通知给 owner
  → owner 审批（同意/拒绝）
  → 同意后：建立 申请人↔aiclaw 的好友关系 + 创建聊天房间 + 初始化 im_aiclaw_friend_ext
```

**改造点：**

| 模块 | 改动 |
|------|------|
| `ApplyService.handlerApply(FriendApplyReq)` | 无改动，target_id 正常存 aiclaw_uid |
| `UserApplyListener.notifyFriend()` | 查 target 的 user_type，若为 AICLAW(4) 则查 im_aiclaw.owner_uid，WS 推送通知给 owner（而非 aiclaw） |
| `ApplyService.handlerApply(ApplyReq)` | 见下方 approve 详细改造 |
| `NoticeVO` | 新增 `receiverUserType` 字段，前端据此区分通知文案 |

**approve() 详细改造（2026-03-26 补充）：**

现有代码在审批同意时，用审批人 uid 与申请人建立好友关系：
```java
// 现有逻辑（ApplyServiceImpl 第 249-252 行）
RoomFriend roomFriend = roomService.createFriendRoom(Arrays.asList(uid, invite.getUid()));
friendService.createFriend(roomFriend.getRoomId(), uid, invite.getUid());
```

aiclaw 场景下，审批人是 owner，但好友关系应建在 **申请人 ↔ aiclaw** 之间。改造逻辑：

```
1. 通过 applyId 查出 invite，取 invite.getTargetId()
2. 查 target 的 userType：
   - 若 userType != AICLAW(4)：走原有逻辑，uid（审批人）↔ invite.getUid()（申请人）
   - 若 userType == AICLAW(4)：
     a. 校验 uid（审批人）== im_aiclaw.owner_uid（必须是 owner 才能审批）
     b. 好友关系建在 invite.getUid()（申请人）↔ invite.getTargetId()（aiclaw）
     c. 创建 im_aiclaw_friend_ext 记录（relation_desc 留空，后续 owner 可设置）
     d. 缓存更新也对应调整为 申请人 ↔ aiclaw 的好友缓存
```

**前端调用方式不变**，仍然传 `{ applyId, state }`，后端根据 target 的 userType 内部分支处理。

**复用现有 WS 推送机制**，不新增通知类型。

---

## 四、安全 prompt 注入机制

### 4.1 WS 消息 payload 扩展

server 将非 owner 用户的消息推送给 aiclaw（plugins）时，在 WS payload 中携带以下额外信息：

```json
{
  "msgId": 12345,
  "roomId": 100,
  "fromUid": 200,
  "content": "你好",
  "type": 1,
  "createTime": 1711234567890,

  "aiclaw": {
    "senderName": "小明",
    "isOwner": false,
    "publicPersona": "你是一个 Python 编程助手，只回答编程相关问题。",
    "relationDesc": "小明是主人的同事，负责前端开发"
  }
}
```

- owner 发送消息时：`aiclaw.isOwner = true`，不带 publicPersona 和 relationDesc
- 非 owner 发送消息时：server 查 im_aiclaw + im_aiclaw_friend_ext 填充完整信息
- `aiclaw` 字段仅在消息目标为 aiclaw 用户时附加

### 4.2 plugins 侧处理

plugins 收到消息后，根据 `aiclaw.isOwner` 决定是否注入安全 prompt：

```typescript
// 构造 extraSystemPrompt
if (!aiclaw.isOwner) {
  const parts: string[] = [];

  // 1. 注入对外人设（如有）
  if (aiclaw.publicPersona) {
    parts.push(aiclaw.publicPersona);
  }

  // 2. 注入安全提示 + 关系说明
  const relation = aiclaw.relationDesc || `${aiclaw.senderName}是主人的通讯录好友`;
  parts.push(
    `[系统提示] 当前与你对话的用户是${aiclaw.senderName}。${relation}。` +
    `请注意信息安全，不要泄露主人的私人信息或执行涉及主人账号/权限的操作。`
  );

  extraSystemPrompt = parts.join('\n\n');
}

// 构造 sessionKey（per-user 隔离）
const sessionKey = aiclaw.isOwner
  ? `aiclaw-${selfUid}-owner`
  : `aiclaw-${selfUid}-peer-${fromUid}`;

// 调用 openclaw
adapter.chat(message, sessionKey, { extraSystemPrompt });
```

---

## 五、B8 延迟注销定时任务

### 5.1 实现方案

使用 XXL-Job 定时任务，每小时执行一次：

```
触发条件：auth_status = 2 AND deactivated_at < NOW() - INTERVAL 24 HOUR AND is_del = 0
```

### 5.2 注销级联操作

```
1. 查 im_aiclaw 的 uid
2. 删除 im_aiclaw 记录（is_del = 1）
3. 删除 im_user 记录（is_del = 1）
4. 查 im_user_friend 找到 aiclaw 的所有好友关系，逐一软删除
5. 更新对应 im_room_friend 状态
6. 删除 im_aiclaw_friend_ext 所有相关记录
7. 通知 plugins 断开该 aiclaw 连接 + 清理所有 openclaw session
8. 清理 Redis 缓存（user、aiclaw、token、好友关系）
9. 消息记录保留，不删除
```

---

## 六、设计确认项跟踪

| # | 设计项 | 方案 | 确认人 | 确认日期 |
|---|--------|------|--------|---------|
| 1 | 好友关系说明存储 | 新建 im_aiclaw_friend_ext 表 | 老大 | 2026-03-24 |
| 2 | 会话上下文隔离 | openclaw sessionKey 原生隔离，无需存储 | 老大 | 2026-03-24 |
| 3 | 好友申请转发 | target_id 存 aiclaw_uid，业务层转 owner | 老大 | 2026-03-24 |
| 4 | 安全 prompt 注入 | server 查好放 WS payload，plugins 用 extraSystemPrompt | 老大 | 2026-03-24 |
| 5 | 对话记录接口 | 复用现有消息查询 Service + owner 鉴权包装 | 老大 | 2026-03-24 |
| 6 | 移除好友 session 清理 | 清理 openclaw session | 老大 | 2026-03-24 |
| 7 | 好友申请通知 | 复用现有 WS 推送，NoticeVO 新增 receiverUserType，前端改文案 | 老大 | 2026-03-24 |
| 8 | B24 sent_at | 取消，直接用 create_time | 老大 | 2026-03-24 |
| 9 | 现有接口改造 | UserInfoResp 补 userType+ownerInfo，UserSearchResp 补 userType，NoticeVO 补 receiverUserType | 前端联调对齐 | 2026-03-26 |
| 10 | 对话列表响应格式 | 按前端确认的 list 格式（friendUid/Name/Avatar + lastMessage + roomId） | 前端联调对齐 | 2026-03-26 |
| 11 | approve() 改造 | aiclaw 场景好友关系建在 申请人↔aiclaw 之间，前端调用方式不变 | 前端联调对齐 | 2026-03-26 |
