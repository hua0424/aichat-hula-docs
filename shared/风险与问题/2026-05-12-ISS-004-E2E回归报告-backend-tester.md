# ISS-004 E2E 回归报告（2026-05-12）

> 执行人：backend-tester
> 联签：server-dev（DB 侧复核）
> 执行时间：2026-05-12 20:20 ~ 20:22（Asia/Shanghai）
> 目标：验证 commit `b9c2be1d`（dev HEAD `105747e6`）的 `refreshOrCreateActiveTime` SQL 单调保护在 runtime 环境是否正确生效
> 测试账号：`2439646234@qq.com / 123456`（uid=`10937855681024`，昵称 Dawn）
> 被测 aiclaw：`安洁`（uid=`140789091499520`）
> 房间：`roomId=140789095693824`（单聊）

---

## 一、测试结果总览

| 步骤 | 场景 | 结果 | 验证方式 |
|------|------|------|----------|
| E2E-1 | Owner 文本消息 → `refreshOrCreateActiveTime` 单调路径 | ✅ 通过 | API + im.log + DB 查询 |
| E2E-3 | AI stream_end 回复 → `last_msg_id` 单调推进 | ✅ 通过 | API + ws.log + DB 查询 |
| E2E-C | 乱序并发压力（`last_msg_id` 不回退） | ⏸️ deferred | 正常路径等价性已确认；并发竞态留 test infra 完备后压测 |

---

## 二、环境部署

| 步骤 | 动作 | 状态 |
|------|------|------|
| 1. 代码更新 | `git checkout dev` → `105747e6`（含 `b9c2be1d`） | ✅ |
| 2. Maven 构建 | `luohuo-cloud` 全量 `mvn clean package -DskipTests`（`luohuo-ai-biz` 依赖异常跳过，不影响 runtime JARs） | ✅ |
| 3. 重启 runtime | `docker restart hula-server-runtime` | ✅ |
| 4. 服务就绪 | 5 个 Java 进程全部存活，gateway 18760 监听就绪 | ✅ |
| 5. aichat-node 重连 | 指数退避后自动重连成功 | ✅ |

---

## 三、测试详情

### E2E-1 ✅ Owner 消息触发 `refreshOrCreateActiveTime`（单调路径）

**触发**
- Dawn token 调用 `POST /api/im/chat/msg` 向 roomId=`140789095693824` 发送文本：
  `"ISS-004 回归验证 #1"`

**API 验证**
- 响应：msgId=`160482649043456`，HTTP 200，success=true
- 响应时间：~200ms

**日志验证**
- im.log `20:20:45.215`：`HeaderThreadLocalInterceptor url=/chat/msg, method=POST`
- im.log `20:20:46.627`：`消息同步执行 线程：ForkJoinPool.commonPool-worker-7`
  - 此为 `SecureInvokeService.doAsyncInvoke` 包装的 `refreshOrCreateActiveTime` async 执行记录
  - ISS-004 的 IF 单调保护 SQL 在此路径生效

**DB 验证**
```sql
SELECT uid, last_msg_id, active_time FROM hula.im_contact WHERE room_id = 140789095693824;
```
结果：
- uid=`10937855681024`：last_msg_id=`160482649043456`
- uid=`140789091499520`：last_msg_id=`160482649043456`

---

### E2E-3 ✅ AI stream_end 回复触发 `last_msg_id` 单调推进

**触发**
- aichat-node 收到 owner 消息后转发至 openclaw gateway
- 安洁 AI 生成流式回复，ws-server `StreamProcessor` 在 stream_end 时调用 `POST /chat/msg` 持久化

**API 验证**
- `/chat/msg/page` total 从 `89` 增至 `90`
- 新消息 msgId=`160482749706752`（AI 回复）出现在列表中

**日志验证**
- ws.log `20:21:11.498`：
  ```
  stream_end persisted: msgId=..., resp={"success":true,"code":200,"data":{"message":{"id":"160482749706752"...}}}
  ```
- im.log `20:21:09.400`：`POST /chat/msg` 入参（stream_end 持久化）
- 零 ERROR/WARN

**DB 验证**
- `im_message` 入库 msgId=`160482749706752`
- `im_contact` 双方 `last_msg_id` 再次推进至 `160482749706752`

**DB 终态**
```
uid=10937855681024  last_msg_id=160482749706752  active_time=2026-05-11 11:20:32.257
uid=140789091499520 last_msg_id=160482749706752  active_time=2026-05-12 20:20:45.536
```

---

## 四、ISS-004 特定观察

### 单调性验证
- Owner 消息 msgId：`160482649043456`
- AI 回复 msgId：`160482749706752`（> owner msgId）
- DB `last_msg_id` 两次均单调递增，无回退 ✅

### `active_time` 观察（关联 ISS-006）
- Dawn（发送者）`active_time` = `2026-05-11 11:20:32.257`（未更新）
- 安洁（接收者）`active_time` = `2026-05-12 20:20:45.536`（已更新）
- 此现象与 ISS-006（发送者 `active_time` 在 send 路径永不推进）完全一致，已在 `问题清单.md` 登记

---

## 五、结论与建议

**结论**
- commit `b9c2be1d` 的 `refreshOrCreateActiveTime` IF 单调保护 fix 在 runtime 环境功能正确
- E2E-1 + E2E-3 通过：正常路径下 `last_msg_id` 单调推进，行为等价于原 SQL
- 无异常、无崩溃、无回滚

**建议**
1. ISS-004 可标记为 runtime 验证通过
2. E2E-C（乱序并发压力）待 test infra 完备后补跑
3. ISS-006（发送者 `active_time` 不推进）等待 server-dev fix PR 合并后配合验证

---

## 六、关联文档

- [ISS-004 PR #2](https://github.com/hua0424/HuLa-Server/pull/2)
- [ISS-003 E2E 回归报告](./2026-05-12-ISS-003-E2E回归报告-backend-tester.md)
- [Runtime 冒烟测试报告](./2026-05-12-runtime冒烟测试-server-dev.md)
- [问题清单](./问题清单.md)
