# ISS-003 方案评审 — reviewer 反馈

> 评审人：reviewer
> 日期：2026-05-12
> 评审对象：[ISS-003-写入路径同步last-msg-id方案.md](ISS-003-写入路径同步last-msg-id方案.md)（commit 90ca42a）
> 结论：**有条件通过**，需处理 2 个 must-fix + 3 个 should-fix

---

## 总评

方案方向正确——同事务写 `contact.last_msg_id` 是最小侵入、最可靠的修复。改动收敛，不碰 MQ/接口契约，回滚干净。以下逐项回复 server-dev 的 5 个问题，并补充代码审查发现。

---

## Q1：§三选型 — 方案 A vs B

**同意选 A。** 补充一点论证：

方案 B 的 `skipPush=true` 不发 MQ 是致命缺陷，但更深层的问题是：即使补发 MQ，异步 Consumer 与主事务之间存在**不可控的窗口期**——`sendMsg` commit 后到 Consumer 消费前，客户端调 `/chat/msg/page` 仍会漏返。方案 A 同事务内完成，窗口期为零。

**但有一个边界需注意**：方案 A 依赖 `@Transactional` 的正常提交。如果 `sendMsg` 后续逻辑（如 `isTemp` 计数器更新）抛异常导致回滚，`refreshLastMsgId` 也会一起回滚——这是正确行为。但需确认 `sendMsg` 方法上没有 `@Async` 或其他导致事务边界不一致的注解。当前代码确认无此问题。

---

## Q2：§四 diff — syncContactLastMsgId 调用时序

**当前时序（`checkAndSaveMsg` 之后、`MessageSendEvent` 之前）是正确的，但需微调。**

分析各时序选项：

| 插入点 | 评价 |
|--------|------|
| `checkAndSaveMsg` 之前 | ❌ msgId 尚未生成，无法调用 |
| `checkAndSaveMsg` 之后、`isTemp` 之前（**建议**） | ✅ 最早可用点，语义清晰：消息落库后立即推进游标 |
| `isTemp` 之后、`publishEvent` 之前（**方案当前**） | ✅ 可行，但与 `isTemp` 逻辑无依赖关系，放在前面更直观 |
| `publishEvent` 之后 | ⚠️ 仍同事务，但语义上给人一种"事件已发再推进"的错觉 |

**建议**：将 `syncContactLastMsgId` 调用移到 `checkAndSaveMsg` 之后、`isTemp` 判断之前。理由：

1. 逻辑上 `last_msg_id` 推进是消息落库的附带动作，与临时会话计数器无关，应尽早完成
2. 如果 `isTemp` 分支抛异常（虽然当前不会），`last_msg_id` 不应回滚——但放在前面也一起回滚，两种位置行为一致，放前面更直观
3. 代码可读性：核心路径（save → sync last_msg_id → event）线性排列

**同时注意风险 #5**：`roomCache.get(roomId)` 在 `check()` 方法内已被调用过一次。`syncContactLastMsgId` 内又调一次。虽然缓存命中开销极小，但更优雅的做法是让 `check()` 返回 Room 对象（或将其存入方法局部变量），避免二次缓存查找。这是非阻塞优化，可后续做。

---

## Q3：§四 ContactDao 新方法 — lambdaUpdate 子条件

**`and(w -> w.isNull().or().lt())` 写法在 MyBatis-Plus 中是标准用法，项目内也有先例。**

但更关键的问题是：**你其实不需要写 lambdaUpdate，项目已有更地道的做法。**

看 `ContactMapper.xml` 中的 `refreshOrCreateActive`：

```sql
INSERT INTO im_contact (room_id, uid, last_msg_id, active_time, is_del) VALUES (...)
ON DUPLICATE KEY UPDATE
  last_msg_id = IF(#{msgId} < last_msg_id, VALUES(last_msg_id), last_msg_id)
```

这是 `INSERT ... ON DUPLICATE KEY UPDATE` 模式，天然具备"只前进不回退"语义，且**同时处理了 Contact 行不存在的情况**（自动 INSERT）。

**Must-fix #1**：当前 `refreshLastMsgId` 方案用 `lambdaUpdate`，**只能 UPDATE 已存在的行**。如果群成员的 Contact 行缺失（§七风险 #2 已提及），UPDATE 0 行，该成员的 `last_msg_id` 永远不会被推进。这不是"既有问题可忽略"——写入路径修复后，拉取路径的 `updateContactLastMsgIds` 也会被触发（因为 WS 推送后客户端会调 `getMsgList`），两者可能产生竞态。

**建议方案**：在 `ContactMapper.xml` 新增一个 `refreshLastMsgId` SQL，复用 `INSERT ... ON DUPLICATE KEY UPDATE` 模式：

```xml
<insert id="refreshLastMsgId">
    INSERT INTO im_contact (room_id, uid, last_msg_id, is_del) VALUES
    <foreach collection="memberUidList" item="uid" separator=",">
        (#{roomId},#{uid},#{msgId},0)
    </foreach>
    ON DUPLICATE KEY UPDATE
    last_msg_id = IF(#{msgId} > last_msg_id OR last_msg_id IS NULL, #{msgId}, last_msg_id)
</insert>
```

这样一行 SQL 同时解决：
- Contact 行不存在 → 自动创建
- `last_msg_id` 为 NULL → 直接更新
- 乱序 msgId → 只前进不回退
- 无需 `and(w -> w.isNull().or().lt())` 子条件

如果坚持用 lambdaUpdate（不想新增 XML），则至少要加一行日志 `log.warn` 当 `update()` 返回 0 行时，以便排查 Contact 行缺失问题。

---

## Q4：§五测试设计 — 并发场景

**CC-1 / CC-2 覆盖了乱序 commit 场景，足够。** 不需要加 SERIALIZABLE 死锁场景。

理由：
1. 当前 `sendMsg` 事务隔离级别是 MySQL 默认 `REPEATABLE READ`，`refreshLastMsgId` 是单行 UPDATE + 条件判断，不会产生 gap lock 冲突
2. 死锁通常发生在两个事务交叉锁不同行（A→B vs B→A），而 `refreshLastMsgId` 对同一 room 内所有成员的 Contact 行是**同一条 SQL** 批量更新，锁获取顺序一致，不会产生死锁
3. SERIALIZABLE 隔离级别在生产环境不使用，测试没有实际意义

**Should-fix**：建议补充一个测试用例：
- **CC-3**：同一房间两个用户几乎同时发消息，验证 `last_msg_id` 最终为两者中较大的 msgId（覆盖群聊并发场景）

---

## Q5：§七风险 #1 — 踢出语义

**Must-fix #2：`groupMemberCache.getMemberUidList(roomId)` 返回的是包含已踢出成员的全量列表，不符合预期。**

代码实证：

```java
// GroupMemberCache.java line 37-43
@Cacheable(cacheNames = "luohuo:member:all", key = "#roomId")
public List<Long> getMemberUidList(Long roomId) {
    RoomGroup roomGroup = roomGroupDao.getByRoomId(roomId);
    if (Objects.isNull(roomGroup)) {
        return null;
    }
    return groupMemberDao.getMemberUidList(roomGroup.getId(), null);  // null = 不过滤 deFriend
}
```

`getMemberUidList(groupId, null)` 中 `null` 表示不过滤 `deFriend`，即**包含被踢出/屏蔽的成员**。

而 `GroupMember` 实体没有"删除"字段——踢出操作是 `removeByGroupId`（物理删除行），不是软标记。但 `deFriend` 屏蔽是软标记。

**关键问题**：方案中注释写"全员（含 sender），未踢出"，但实际调用返回的是**包含已屏蔽成员**的列表。这意味着被踢出后又被 `removeByGroupId` 删除的成员不会出现在列表中（正确），但主动屏蔽群的成员（`deFriend=true`）会出现在列表中，他们的 `contact.last_msg_id` 也会被推进。

**这是否是期望行为？** 取决于产品需求：
- 如果屏蔽群的人取消屏蔽后也应该能看到期间的消息 → 应该推进 `last_msg_id` → 当前行为正确
- 如果屏蔽群的人取消屏蔽后不应看到屏蔽期间的消息 → 不应推进 → 需改用 `getMemberExceptUidList`

**建议**：在方案中明确标注使用 `getMemberUidList`（含屏蔽成员）是**有意为之**，并说明原因。如果确实应该排除屏蔽成员，改用 `getMemberExceptUidList`。

---

## 额外发现

### E1：`getMemberUidList` 返回 null 时的 NPE

```java
// 方案中的 syncContactLastMsgId
memberUids = groupMemberCache.getMemberUidList(roomId);
// ...
contactDao.refreshLastMsgId(roomId, msgId, memberUids);
```

`getMemberUidList` 在 `roomGroup` 为 null 时返回 `null`。虽然 `refreshLastMsgId` 内有 `CollectionUtil.isEmpty(uidList)` 判断可以处理 null，但建议在 `syncContactLastMsgId` 中也做 null 判断或加日志，避免静默跳过异常情况。

### E2：单聊 `roomFriendDao.getByRoomId` 返回 null 的防御

方案中 `syncContactLastMsgId` 单聊分支：

```java
RoomFriend rf = roomFriendDao.getByRoomId(roomId);
memberUids = List.of(rf.getUid1(), rf.getUid2());
```

如果 `rf` 为 null，直接 NPE。建议加 null 检查 + 日志。UT-4 也提到了这个问题但方案说"按 fail-fast 原则可抛 IllegalStateException"——在 `@Transactional` 内抛异常会导致整个 `sendMsg` 回滚，消息丢失。**必须做防御性返回而不是抛异常。**

### E3：`refreshLastMsgId` 与 `refreshOrCreateActiveTime` 的关系

现有 `ContactDao.refreshOrCreateActiveTime`（由 `MsgSendConsumer` 调用）也会更新 `last_msg_id`：

```sql
ON DUPLICATE KEY UPDATE
  last_msg_id = VALUES(last_msg_id),
  active_time = VALUES(active_time)
```

ISS-003 修复后，写入路径（`sendMsg` 同事务）和推送路径（`MsgSendConsumer` 异步）都会更新 `last_msg_id`。两者可能对同一行产生并发写入，但由于都只做"前进"更新，结果一致。不过 `refreshOrCreateActiveTime` 中的 `last_msg_id = VALUES(last_msg_id)` **没有乱序保护**（无条件覆盖），如果 Consumer 消费延迟，可能用旧 msgId 覆盖新 msgId。

**Should-fix**：建议 `refreshOrCreateActiveTime` 也加上 `IF` 条件保护，与 `refreshOrCreateActive` 保持一致：

```sql
last_msg_id = IF(VALUES(last_msg_id) > last_msg_id, VALUES(last_msg_id), last_msg_id)
```

这不在 ISS-003 范围内，但应登记为 follow-up issue。

### E4：Room.last_msg_id skipPush 缺口

方案 §二.3 已记录，另起 issue。**同意不在本次修复**。但建议在代码中加一行注释标记，方便后续定位：

```java
// TODO(ISS-00X): Room.last_msg_id not updated when skipPush=true (stream_end path)
```

---

## 评审总结

| 级别 | 编号 | 内容 | 阻塞? |
|------|------|------|-------|
| **Must-fix** | M1 | `refreshLastMsgId` 用 lambdaUpdate 无法处理 Contact 行缺失，建议改用 `INSERT ... ON DUPLICATE KEY UPDATE` | 是 |
| **Must-fix** | M2 | `getMemberUidList` 返回包含屏蔽成员的列表，与方案注释"未踢出"不一致，需明确意图 | 是 |
| Should-fix | S1 | `syncContactLastMsgId` 调用位置建议移到 `isTemp` 之前 | 否 |
| Should-fix | S2 | 单聊 `roomFriendDao.getByRoomId` 返回 null 时需防御性返回，不能抛异常 | 否 |
| Should-fix | S3 | `refreshOrCreateActiveTime` SQL 缺少乱序保护，登记 follow-up issue | 否 |
| Info | I1 | 补充 CC-3 群聊并发测试用例 | 否 |
| Info | I2 | `getMemberUidList` 返回 null 时加日志 | 否 |
| Info | I3 | `Room.last_msg_id` skipPush 缺口加 TODO 注释 | 否 |

**结论**：M1 + M2 修复后可合入。
