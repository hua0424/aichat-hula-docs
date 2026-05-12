# ISS-003：`/chat/msg` 写入路径同步 `im_contact.last_msg_id` 方案

> 作者：server-dev
> 日期：2026-05-12
> 优先级：P1（manager 2026-05-12 18:35 升级 + Owner 指派）
> 关联：[ISS-002 复测后系统性根因](2026-05-12-runtime冒烟测试-server-dev.md#二、发现的问题) / [问题清单 #ISS-003](问题清单.md)
> 状态：草案，待 manager + reviewer 异步评审

---

## 一、问题陈述

`/api/im/chat/msg/page` 漏返新消息的根因（ISS-002 复测后定位）：

- 服务端**写入路径**只插入 `im_message`，从不更新发送/接收双方的 `im_contact.last_msg_id`；
- 唯一的 `last_msg_id` 更新逻辑在 [ChatServiceImpl#updateContactLastMsgIds](../../../luohuo-cloud/luohuo-im/luohuo-im-biz/src/main/java/com/luohuo/flex/im/core/chat/service/impl/ChatServiceImpl.java)（line 196-231），且**只在 `getMsgList` 拉取路径上异步触发**——纯依赖 WS 推送的客户端永远不会触发它；
- 因此每条新消息发出后，所有持有 stale `last_msg_id` 的客户端调用 `/chat/msg/page` 都会被 `Message.id ≤ contact.last_msg_id` 条件过滤掉。

2026-05-12 18:28 端到端复测复现：

| 指标 | 复测前 | 复测后（owner→安洁 各 1 条） |
|------|-------|-----|
| DB `im_message` count | 84 | 86 |
| owner `contact.last_msg_id` | 160442597630976 | **160442597630976（未更新）** |
| `/chat/msg/page` total（owner 视角） | 84 | **84（漏返 2 条）** |

backend-tester 之前对 owner 端 `last_msg_id` 的手工修正是 one-shot 兜底；写入路径不修，每次新消息都会再次复现。

---

## 二、写入路径全景与缺口

### 2.1 入口收口

所有 `im_message` 插入新行都收口在：

```
POST /api/im/chat/msg
  → ChatController.send → ChatServiceImpl.sendMsg(req, uid)   [@Transactional]
    → MsgHandlerFactory.getStrategyNoNull(msgType)
      → AbstractMsgHandler#checkAndSaveMsg → 各策略 saveMsg → MessageDao.save
```

涵盖的策略：`TextMsgHandler` / `ImgMsgHandler` / `FileMsgHandler` / `SoundMsgHandler` / `VideoMsgHandler` / `EmojisMsgHandler` / `MergeMsgHandler` / `MapMsgHandler` / `SystemMsgHandler` / `NoticeMsgHandler` / `BotMsgHandler` / `AudioCallMsgHandler` / `VideoCallMsgHandler`。

### 2.2 撤回路径（**天然豁免**）

```
POST /api/im/chat/msg/recall
  → ChatServiceImpl.recallMsg → RecallMsgHandler.recall
    → MessageDao.updateById(type=RECALL)   # 仅 UPDATE，不插入新行
    → MessageRecallEvent（独立事件，不进 sendMsg 路径）
```

→ 不修改 `last_msg_id`，符合 manager「撤回不要更新」要求。

### 2.3 aiclaw 流式回复（stream_end）

```
aichat-node → WS STREAM_END(19) → StreamProcessor.handleEnd
  → persistStreamMessage(ctx, fullContent)
    → HTTP POST /chat/msg  (body: {skip:true, skipPush:true, msgType:1, content})
      → 再次进 ChatServiceImpl.sendMsg
```

→ stream_end 路径**复用** `sendMsg`，因此只要在 `sendMsg` 内同事务加 `contact.last_msg_id` 同步，**stream_end 自动覆盖**。

> 顺带观察（**不在 ISS-003 范围**）：stream_end 因 `skipPush=true` 不发 `MessageSendEvent`，导致 `MsgSendConsumer` 不触发 → `Room.last_msg_id` 没被更新（[RoomDao.refreshActiveTime](../../../luohuo-cloud/luohuo-im/luohuo-im-biz/src/main/java/com/luohuo/flex/im/core/chat/dao/RoomDao.java) line 23）。会导致最近会话列表显示 owner 发的最后一条而不是 AI 回复。manager 已记下，后续单独立 issue。

### 2.4 现状一览

| 写入入口 | 是否更新 `Contact.last_msg_id` | 备注 |
|---------|------|------|
| owner 单聊/群聊发文本/图/文件等 | ❌ 不更新 | ISS-003 主修复 |
| owner 撤回消息 | ❌ 不更新（也不应更新） | 已豁免 |
| aiclaw stream_end 落库 | ❌ 不更新 | 修了 §2.1 自动覆盖 |
| 客户端 `getMsgList` 拉取（登录/同步） | ✅ 异步更新 receiveUid 的 contact | 现状保留 |

---

## 三、方案 A：同事务 `contactDao.refreshLastMsgId`

### 3.1 选型理由

| 维度 | 方案 A（同事务） | 方案 B（MQ Consumer 兜底） |
|------|-----------------|---------------------------|
| 原子性 | ✅ 与 `Message` 插入同 `@Transactional`，不会半提交 | ❌ MQ 异步，存在窗口期 |
| stream_end 覆盖 | ✅ skipPush=true 也走 sendMsg | ❌ skipPush=true 不发 MQ |
| 跨服务依赖 | ✅ 不依赖 RocketMQ 可用性 | ❌ MQ 故障时静默失败 |
| 性能成本 | 单聊 1 行 / 群聊 1 条 batch UPDATE（在 `(room_id, uid)` 索引下毫秒级） | 与方案 A 相同 |
| 复杂度 | ✅ 改动收敛在 `ChatServiceImpl` + `ContactDao` | ❌ 需触动 MQ DTO + Consumer + skipPush 语义 |

→ 选 A。

### 3.2 不动 skipPush 语义

manager 提示「**先聚焦 ISS-003 范围**」。本次不重构 `skipPush` 的事件触发逻辑，`Room.last_msg_id` 缺口另起 issue。

---

## 四、变更点（代码 diff 草稿）

### 4.1 新增 `ContactDao.refreshLastMsgId`

文件：`luohuo-cloud/luohuo-im/luohuo-im-biz/src/main/java/com/luohuo/flex/im/core/chat/dao/ContactDao.java`

```java
/**
 * 同步房间内指定成员的会话 last_msg_id（只前进，不回退）
 *
 * 单条 UPDATE：WHERE room_id = ? AND uid IN (...) AND (last_msg_id IS NULL OR last_msg_id < ?)
 * 配合 (room_id, uid) 索引可在毫秒级完成；条件保证并发乱序写入时 last_msg_id 单调递增。
 */
public void refreshLastMsgId(Long roomId, Long msgId, List<Long> uidList) {
    if (CollectionUtil.isEmpty(uidList) || msgId == null || roomId == null) {
        return;
    }
    lambdaUpdate()
            .eq(Contact::getRoomId, roomId)
            .in(Contact::getUid, uidList)
            .and(w -> w.isNull(Contact::getLastMsgId).or().lt(Contact::getLastMsgId, msgId))
            .set(Contact::getLastMsgId, msgId)
            .update();
}
```

### 4.2 `ChatServiceImpl.sendMsg` 追加同步调用

文件：`luohuo-cloud/luohuo-im/luohuo-im-biz/src/main/java/com/luohuo/flex/im/core/chat/service/impl/ChatServiceImpl.java`

```diff
 @Override
 @Transactional
 public Long sendMsg(ChatMessageReq request, Long uid) {
     check(true, request.isSkip(), request.isTemp(), request.getRoomId(), uid);
     AbstractMsgHandler<?> msgHandler = MsgHandlerFactory.getStrategyNoNull(request.getMsgType());
     Long msgId = msgHandler.checkAndSaveMsg(request, uid);

+    // 同事务推进房间内所有活跃成员的 contact.last_msg_id（含 sender）
+    // 修复 ISS-003：写入路径不更新 contact.last_msg_id，导致 /chat/msg/page 漏返新消息
+    syncContactLastMsgId(request.getRoomId(), msgId);
+
     // 临时会话单独处理一下消息计数器
     if (request.isTemp()) {
         ...
     }

     // 发布消息发送事件（skipPush=true 时仅存库不推送，用于流式消息 stream_end 落库）
     if (!request.isSkipPush()) {
         SpringUtils.publishEvent(new MessageSendEvent(this, new ChatMsgSendDto(msgId, uid)));
     }
     return msgId;
 }

+/**
+ * 推进 contact.last_msg_id：单聊为双方 uid，群聊为所有未踢出成员。
+ * 与 {@link #sendMsg} 共享事务，避免与 im_message 插入产生窗口期。
+ */
+private void syncContactLastMsgId(Long roomId, Long msgId) {
+    Room room = roomCache.get(roomId);
+    if (room == null) {
+        return;
+    }
+    List<Long> memberUids;
+    if (Objects.equals(room.getType(), RoomTypeEnum.GROUP.getType())) {
+        memberUids = groupMemberCache.getMemberUidList(roomId);  // 全员（含 sender），未踢出
+    } else if (Objects.equals(room.getType(), RoomTypeEnum.FRIEND.getType())) {
+        RoomFriend rf = roomFriendDao.getByRoomId(roomId);
+        memberUids = List.of(rf.getUid1(), rf.getUid2());
+    } else {
+        return;  // 热点房间（HotRoom）等其他类型不参与 contact 维度的游标
+    }
+    contactDao.refreshLastMsgId(roomId, msgId, memberUids);
+}
```

### 4.3 受影响文件清单

| 文件 | 改动 |
|------|------|
| `ContactDao.java` | 新增 `refreshLastMsgId` 方法 |
| `ChatServiceImpl.java` | `sendMsg` 内插一行调用 + 私有方法 `syncContactLastMsgId` |
| **无其他改动** | 不动 MQ、不动 listener、不动 skipPush 语义、不动撤回路径 |

---

## 五、测试设计

### 5.1 单元测试（`ChatServiceImplTest`）

| # | 用例 | 期望 |
|---|------|------|
| UT-1 | 单聊发文本：owner→friend，给定 msgId=200，roomFriend.uid1=owner、uid2=friend | `refreshLastMsgId(roomId, 200, [owner, friend])` 被调用一次 |
| UT-2 | 群聊发文本：roomId 对应 RoomTypeEnum.GROUP | 用 `getMemberUidList` 返回值调用 `refreshLastMsgId` |
| UT-3 | 房间不存在（`roomCache.get` 返回 null） | `refreshLastMsgId` 不被调用，方法静默返回 |
| UT-4 | `roomFriendDao.getByRoomId` 返回 null（数据异常） | 抛出 NPE 或安全返回（按 fail-fast 原则可抛 IllegalStateException，但保留原有行为） |

### 5.2 DAO 层测试（`ContactDaoTest`，集成 MySQL）

| # | 用例 | 期望 |
|---|------|------|
| DT-1 | 现有 contact.last_msg_id=100，refresh 到 200 | last_msg_id 更新为 200 |
| DT-2 | 现有 contact.last_msg_id=200，refresh 到 150（乱序） | last_msg_id 不变（保持 200，`<` 条件未命中） |
| DT-3 | 现有 contact.last_msg_id=NULL | last_msg_id 更新为 200 |
| DT-4 | 房间内有踢出用户（不在 uidList） | 该用户 last_msg_id 不变（`uid IN (...)` 未命中） |
| DT-5 | uidList 为空 | 方法直接 return，不执行 SQL |

### 5.3 端到端联调测试（接 runtime + aichat-node 桥接）

复用 [冒烟测试 §五 ISS-001 闭环复测](2026-05-12-runtime冒烟测试-server-dev.md#五iss-001-闭环复测2026-05-12-18-27-18-28) 流程，但增加 page 校验：

| # | 操作 | 期望 |
|---|------|------|
| E2E-1 | owner 在 `roomId=140789095693824` 发文本 → 拿到 `owner.msgId` | 立即查 `im_contact WHERE uid=owner, roomId=...`：last_msg_id = owner.msgId |
| E2E-2 | 同时查 `im_contact WHERE uid=安洁`：last_msg_id = owner.msgId | ✅ |
| E2E-3 | 等 ~20s 收到 streamEnd | 查 `im_contact WHERE uid=owner`：last_msg_id = 安洁.msgId（覆盖更新） |
| E2E-4 | `GET /chat/msg/page?roomId=...&pageSize=100`（owner 视角） | total = DB count，无漏返 |
| E2E-5 | owner 撤回 owner 自己刚发的消息 | `im_contact.last_msg_id` 保持在 E2E-3 的 stream_end msgId（撤回不动） |

### 5.4 并发场景

| # | 用例 | 期望 |
|---|------|------|
| CC-1 | 两个事务并发 sendMsg：T1 拿到 msgId=100，T2 拿到 msgId=101，T1 先 commit | 最终 contact.last_msg_id = 101（不会被 T2 的 100 回退） |
| CC-2 | T1 拿到 msgId=101，T2 拿到 msgId=100，T2 后 commit | 同上，最终 = 101（`<` 条件保证） |

`<` 条件保证幂等 + 单调递增，无需额外锁。

---

## 六、回滚策略

1. **代码回滚**：直接 `git revert` 本次 commit；
2. **数据状态**：本次方案只**前进** `last_msg_id`，不会破坏旧数据。回滚后 contact 表的 last_msg_id 会继续保持「修复期间已推进的高位」，对后续 `/chat/msg/page` 而言反而是改进（漏返概率降低），无副作用；
3. **不需要数据清理**：与 ISS-002 时 backend-tester 的「one-shot 修正 + 清 Redis」不同，本方案在数据层无遗留。

---

## 七、风险与未对齐项

| # | 内容 | 责任方 | 阻塞? |
|---|------|--------|------|
| 1 | `groupMemberCache.getMemberUidList` 是否完全排除「踢出」用户？需确认 `GroupMember` 表删除/软标记语义。**实现期间需在 PR 描述里写明** | server-dev | 否 |
| 2 | Contact 行缺失场景：新加群成员未自动 create Contact 行 → UPDATE 0 行 → 该成员后续 page 漏返。属于既有问题（群创建 / 成员加入路径），不在 ISS-003 范围 | manager | 否 |
| 3 | 大群（>500 成员）的 batch UPDATE 性能：`(room_id, uid)` 索引下 lambdaUpdate `in` 子句性能可接受；如生产出现慢查询可改成分批 | reviewer | 否 |
| 4 | `Room.last_msg_id` 在 skipPush=true（stream_end）路径下不更新 → 最近会话列表显示错误。**另起 issue** | manager | 否 |
| 5 | `roomCache.get(roomId)` 在 sendMsg 入口已被 `check` 调用，理论上结果可复用避免二次缓存查找。本次为可读性新加一次调用 | server-dev | 否（可后续优化） |

---

## 八、后续动作清单

- [ ] **manager / reviewer**：异步评审本方案（无窗口期约束）；评论或 send_message 反馈即可；
- [ ] **server-dev**（我）：评审通过后 commit + 提 PR：
  - [ ] `ContactDao#refreshLastMsgId` + UT
  - [ ] `ChatServiceImpl#syncContactLastMsgId` + UT
  - [ ] DAO 集成测试（含 DT-1 ~ DT-5）
  - [ ] 跑全量已有测试，确保未破坏现有路径
- [ ] **server-dev**：合并后联系 backend-tester 在 runtime 复测 E2E-1 ~ E2E-5，回归 ISS-002 / ISS-003 关闭场景；
- [ ] **server-dev**：把 `Room.last_msg_id` skipPush=true 缺口登记为新 issue（独立 PR，不阻塞本次）。

---

## 九、变更范围摘要（给 reviewer 一句话）

仅两处改动：`ContactDao` 新增 1 个 lambdaUpdate 方法，`ChatServiceImpl#sendMsg` 在 `msgHandler.checkAndSaveMsg` 之后插一行 + 1 个 private helper。同事务、只前进、撤回豁免、stream_end 自动覆盖。无 MQ / 配置 / 接口契约改动。
