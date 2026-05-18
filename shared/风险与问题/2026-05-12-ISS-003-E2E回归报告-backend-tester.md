# ISS-003 E2E 回归报告（2026-05-12）

> 执行人：backend-tester（日志侧确认）+ server-dev（DB 侧验证 + 触发测试）
> 执行时间：2026-05-12 19:35 ~ 19:37（Asia/Shanghai）
> 目标：验证 commit `3663bc6c` 的 `syncContactLastMsgId` fix 在 runtime 环境是否正确生效
> 测试账号：`2439646234@qq.com / 123456`（uid=`10937855681024`，昵称 Dawn）
> 被测 aiclaw：`安洁`（uid=`140789091499520`，adapter_type=`openclaw`，auth_status=`1`）
> 房间：`roomId=140789095693824`（单聊）

---

## 一、测试结果总览

| 步骤 | 场景 | 结果 | 验证方式 |
|------|------|------|----------|
| E2E-1 | Owner 文本消息 → contact.last_msg_id 同步 | ✅ 通过 | server-dev DB 查询 + backend-tester 日志复核 |
| E2E-3 | AI stream_end 回复 → contact.last_msg_id 同步 | ✅ 通过 | server-dev DB 查询 + backend-tester 日志复核 |
| E2E-4 | 消息撤回 → contact.last_msg_id 回退 | ⏸️ deferred | 非 ISS-003 阻塞项，后续安排 |
| E2E-5 | 新成员入群 → contact.last_msg_id 初始化 | ⏸️ deferred | 非 ISS-003 阻塞项，后续安排 |

---

## 二、测试详情

### E2E-1 ✅ Owner 文本消息触发 contact 同步

**触发**
- server-dev 以 Dawn token 调用 `POST /api/im/chat/msg` 向 roomId=`140789095693824` 发送文本：
  `"ISS-003 E2E 验证 #1"`

**DB 验证（server-dev）**
- `im_message` 入库 msgId=`160471311837696`
- `im_contact` 两行（uid=`10937855681024` 与 uid=`140789091499520`）的 `last_msg_id` 均同步推进至 `160471311837696`

**日志验证（backend-tester）**
- im.log：`19:35:41.440` `POST /chat/msg` 入参；`19:35:43.345` 消息处理完成；零 ERROR/WARN
- ws.log：`19:36:00.103/19:36:00.320` `PushConsumer` 推送消息至 aichat-node；零 ERROR/WARN

---

### E2E-3 ✅ AI stream_end 回复触发 contact 同步

**触发**
- aichat-node 收到 owner 消息后转发至 openclaw gateway
- 安洁 AI 生成流式回复，ws-server `StreamProcessor` 在 stream_end 时调用 `POST /chat/msg` 持久化

**DB 验证（server-dev）**
- `im_message` 入库 msgId=`160471492192768`
- `im_contact` 两行 `last_msg_id` 均再次推进至 `160471492192768`

**日志验证（backend-tester）**
- im.log：`19:36:25.465` `POST /chat/msg` 入参（stream_end 持久化）；零 ERROR/WARN
- ws.log：`19:36:26.898` `stream_end persisted: success=true, code=200, data.message.id=160471492192768`；零 ERROR/WARN

**关于 syncContactLastMsgId 的日志说明**
- `syncContactLastMsgId` 为 `ChatServiceImpl#sendMsg` 内同步 inline 调用，位于 `@Transactional` 事务中，与 `im_message` 插入同事务
- 该方法无 `@Async`，正常路径无 INFO 日志（仅异常分支打 `log.warn`），因此日志侧「静默」为预期行为
- im.log / ws.log 在 19:35~19:37 窗口无任何 ERROR/WARN/rollback，进程（pid 57 im-server 等）全部存活，无 crash/restart

---

## 三、环境状态

| 组件 | 状态 | 备注 |
|------|------|------|
| hula-server-runtime | ✅ Up | 5 个 Java 进程全部存活（oauth/system/ws/im/gateway） |
| nacos | ✅ Up | standalone，healthcheck 通过 |
| aichat-plugins-dev | ✅ Up | openclaw gateway + aichat-node 双进程运行，已激活 |
| aichat-node ↔ runtime WS | ✅ Connected | `ws://hula-server-runtime:18760/api/ws/ws` |

---

## 四、结论与建议

**结论**
- commit `3663bc6c` 的 `syncContactLastMsgId` fix 在 runtime 环境功能正确
- E2E-1（owner 消息）与 E2E-3（AI stream_end 回复）双侧 `im_contact.last_msg_id` 均正确同步推进
- 零异常、零崩溃、零回滚

**建议**
1. 关闭 ISS-003
2. E2E-4（消息撤回）、E2E-5（新成员入群）另起 follow-up 安排，不阻塞本次 closure

---

## 五、关联文档

- [ISS-003 问题记录](./问题清单.md)
- [Runtime 冒烟测试报告（2026-05-12）](./2026-05-12-runtime冒烟测试-server-dev.md)
- [aichat-node 部署改造方案](../环境与部署/配置/aichat-node部署改造方案.md)
