# ISS-015 E2E 回归报告（2026-05-15）

> 执行人：backend-tester
> 联签：server-dev（PR 作者）+ manager（任务协调）
> 执行时间：2026-05-15 11:50 ~ 11:55（Asia/Shanghai）
> 目标：验证 dev HEAD `04d94c55`（PR #9 ISS-015）AI 流式回复 sendTime 是否取 stream_start 而非 stream_end
> 测试账号：`2439646234@qq.com / 123456`（uid=`10937855681024`，昵称 Dawn）
> 被测 aiclaw：`安洁`（uid=`140789091499520`）
> 房间：`roomId=140789095693824`（单聊）

---

## 一、测试结果总览

| 步骤 | 场景 | 结果 | 验证方式 |
|------|------|------|----------|
| 部署 | dev HEAD `04d94c55` 上 runtime | ✅ 通过 | 4 个 JAR 重打 + 重启 |
| 登录 | OAuth anyTenant/login 获取新 token | ✅ 通过 | API + token 校验 |
| E2E-1 | Owner 消息触发 AI 流式回复 | ✅ 通过 | API + ws.log |
| **E2E-2** | **AI reply sendTime ≈ stream_start, 非 stream_end** | ✅ **通过** | **ws.log 时间戳 + DB + API 三方交叉** |
| 消息排序 | `/chat/msg/page` 时序正确 | ✅ 通过 | API total + list 顺序 |

---

## 二、环境部署

| 步骤 | 动作 | 状态 |
|------|------|------|
| 1. 代码更新 | `git checkout dev && git pull` → `04d94c55`（含 ISS-015 PR #9） | ✅ |
| 2. Maven 构建 | `docker exec hula-server-dev mvn -B -Dmaven.test.skip=true clean install`（luohuo-im/ws/oauth/gateway 4 个 JAR 10:42 重打） | ✅ |
| 3. 重启 runtime | `docker restart hula-server-runtime` | ✅ |
| 4. 服务就绪 | 5 个 Java 进程注册 nacos,gateway 18080 监听 | ✅ |
| 5. Token 重新签发 | OAuth login → `bcbdbb8c-1ccc-4d46-997c-99bbf759b45a` | ✅ |

**构建说明**
- `luohuo-im-server.jar` / `luohuo-ws-server.jar` 含 ISS-015 改动（MessageAdapter / ChatMessageReq / StreamProcessor）,mtime `May 15 10:42` ✅
- `luohuo-ai-biz` `javax.jws-api` 占位符 fail → `luohuo-system` 子树 SKIPPED,system-server.jar 仍为 3月30 历史缓存(§十 已知,manager 已决合并 ISS-012 一起扫)

---

## 三、登录恢复（供后续测试复用）

旧 token `b594050c-669a-44a8-a8f5-bad657d5cccb`（May 13 签发）已过期,本次验证重新获取 token。

**可用 curl（backend-tester 留存）**

```bash
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: bHVvaHVvX3dlYjp7dW9odV93ZWJfc2VjcmV0}" \
  -d '{
    "account": "2439646234@qq.com",
    "password": "123456",
    "deviceType": "PC",
    "systemType": "2",
    "grantType": "PASSWORD",
    "isAutoLogin": false,
    "asyncData": true
  }' \
  "http://localhost:18080/api/oauth/anyTenant/login"
```

> ⚠️ 之前多次 trial-and-error 的错误根源:body 字段名应为 `deviceType`(不是 terminal/clientType/loginType)、`systemType` 为字符串 `"2"`、`grantType` 为字符串 `"PASSWORD"`,且必须带 `Authorization: base64('luohuo_web:luohuo_web_secret')` header。

---

## 四、ISS-015 验证详情

### 现象（修复前）

AI 流式回复的 `sendTime` 取 `stream_end` 时刻,长流式回复（10s+）会导致:
- 消息列表中 AI 回复时间戳比 owner 消息晚十几秒
- 排序可能出现"owner 消息 → 其他消息 → AI 回复"的夹塞现象

### 触发

Dawn token 调用 `POST /api/im/chat/msg` 向 roomId=`140789095693824` 发送文本:
`"ISS-015 stream_start sendTime 验证 #1"`

### 时间线（三方交叉证据）

| 事件 | 时间戳 (CST) | 来源 | 说明 |
|------|-------------|------|------|
| T0: Owner 消息 sendTime | `11:51:46.038` | POST /chat/msg 响应 + API | 基准 |
| T1: PushConsumer 收到 owner 消息 | `11:51:50.894` | ws.log | 路由到 aiclaw-node |
| **T2: AI reply sendTime** | **`11:51:52.994`** | **ws.log stream_end JSON + DB + API** | **≈ T1 (stream_start 附近)** |
| T3: stream_end persisted | `11:52:04.320` | ws.log | 流式生成完成 |

**时间差分析**

| 区间 | 差值 | 结论 |
|------|------|------|
| T1 → T2 | **2.100 s** | AI sendTime 紧跟 PushConsumer 接收,即 **stream_start 附近** |
| T2 → T3 | **11.326 s** | AI sendTime **远早于** stream_end |
| T0 → T2 | 6.956 s | 正常 owner → AI 响应间隔 |

### DB 验证

```sql
SELECT id, room_id, from_uid, create_time, type, LEFT(content, 40)
FROM im_message WHERE id IN (161441722802688, 161441794105856);
```

| id | room_id | from_uid | create_time | type | content |
|----|---------|----------|-------------|------|---------|
| 161441722802688 | 140789095693824 | 10937855681024 | 2026-05-15 11:51:46.038 | 1 | ISS-015 stream_start sendTime 验证 #1 |
| 161441794105856 | 140789095693824 | 140789091499520 | 2026-05-15 11:51:52.994 | 1 | 老大，ISS-015 验证收到 🐱… |

### API 验证（`/chat/msg/page`）

- total=`102`（含本次 2 条新增）
- 消息列表时序:owner(11:51:46.038) → AI(11:51:52.994) ✅ 无夹塞

### ws.log 关键摘录

```
11:51:50.894 PushConsumer.onMessage 收到节点消息:
  fromUser.uid=10937855681024, message.id=161441722802688
  deviceUserMap={...=140789091499520}

11:52:04.320 StreamProcessor.stream_end persisted:
  msgId=2055134042755624960
  resp.data.message.id=161441794105856
  resp.data.message.sendTime=1778817112994  ← T2
```

---

## 五、结论与建议

**结论**
- dev HEAD `04d94c55` 的 ISS-015 fix 在 runtime 环境功能正确
- AI 流式回复的 `sendTime` 已改为取 **stream_start** 时刻（≈ PushConsumer 收到 owner 消息后 2s）,而非 **stream_end** 时刻（再晚 11s）
- 消息列表时序正确,无夹塞
- 零异常、零崩溃、零回滚

**建议**
1. ISS-015 标记为 runtime 验证通过,可关闭
2. ui-tester 跑 GUI 两轮对话复测（manager 已安排）,重点观察消息气泡时间戳与排序
3. system-server.jar 历史缓存问题继续跟 ISS-012 一起扫（§十 已留痕）
4. 新 token `bcbdbb8c-1ccc-4d46-997c-99bbf759b45a` 有效期约 30 天,供后续测试复用

---

## 六、关联文档

- [ISS-015 PR #9](https://github.com/hua0424/HuLa-Server/pull/9)
- [ISS-013 PR #8 E2E 回归报告](./2026-05-13-ISS-009-010-E2E回归报告-backend-tester.md) — §十 javax.jws-api 根因实证
- [ISS-009/010 E2E 联合回归报告](./2026-05-13-ISS-009-010-E2E回归报告-backend-tester.md)
- [Runtime 冒烟测试报告](./2026-05-12-runtime冒烟测试-server-dev.md)
