# Runtime 冒烟测试报告（2026-05-12）

> 执行人：server-dev
> 执行时间：2026-05-12 16:12 ~ 16:21
> 目标：验证 runtime 重新部署 + openclaw 更新后，登录 → aiclaw 列表 → 与"安洁"聊天的核心 API 链路是否正常
> 测试账号：`2439646234@qq.com / 123456`（uid=`10937855681024`）
> 被测 aiclaw：`安洁`（uid=`140789091499520`，adapter_type=`openclaw`，auth_status=`1`）
> 房间：`roomId=140789095693824`（单聊）

## 一、测试结果总览

| 步骤 | API | 结果 | 备注 |
|------|------|------|------|
| 1. 登录 | `POST /api/oauth/anyTenant/login` | ✅ 通过 | token 正常返回，expire=2591998s |
| 2. aiclaw 列表 | `GET /api/im/aiclaw/list` | ✅ 通过 | 返回 2 条，含"安洁" |
| 3. 单聊会话详情 | `GET /api/im/chat/contact/detail/friend?id=140789091499520&roomType=2` | ✅ 通过 | 拿到 roomId |
| 4. 发送消息 | `POST /api/im/chat/msg` | ✅ HTTP 200 | DB `im_message` 已写入（id=160420296516608、160422209119232） |
| 5. AI 流式回复 | WebSocket streamStart/Delta/End | ❌ **未触发** | 见问题 #1 |
| 6. 消息分页 | `GET /api/im/chat/msg/page` | ✅ 已闭环（2026-05-12 17:54 复测） | 见问题 #2 |

## 二、发现的问题

### 问题 #1：aiclaw 无 AI 回复（阻断 P0）

**现象**
- 16:12:59 / 16:20:35 各发一条文本到房间 `140789095693824`，DB 入库正常；
- runtime `/runtime/logs/im.log` 仅出现 `HeaderThreadLocalInterceptor url=/chat/msg, method=POST` 与 `消息同步执行 ForkJoinPool.commonPool-worker-N`，**没有任何 aiclaw push / streamStart / streamEnd 日志**；
- 历史上 2026-05-11 11:20 起的 `"你好"`、`"在吗"`、本次 2 条测试消息均无 AI 回复；只有 2026-04 之前的旧记录有 `🐱` 回复。

**根因猜测**
- runtime 仅有 `hula-server-runtime` 容器在跑，对应的 `aichat-plugins-runtime`（openclaw 桥接，需以 aiclaw token 登录 runtime WS）**不存在**：
  - `docker ps`：只有 `hula-server-runtime`（Up 3 days）+ `aichat-nacos`；
  - 现存的 `aichat-plugins-dev` 容器（在调用我的工具时被 SIGKILL 退出）`env` 里 `HULA_SERVER_URL=http://hula-server-dev:18760`，本身就只接 dev，不接 runtime；
  - compose `infra/compose/docker-compose.dev.yml` 中也只定义了 `aichat-plugins-dev` 这一个 plugins 服务。
- `MsgSendConsumer` 单聊分支会 `if (onlineUsersList.contains(aiclawUid)) pushService.sendPushMsg(...)`；如果 aiclaw 在线集合不含该 uid，消息只入库、不推送到桥接，AI 自然不回复，符合本次现象。

**复现步骤**
1. `POST /api/oauth/anyTenant/login` 以 `2439646234@qq.com / 123456` 取 token；
2. `POST /api/im/chat/msg` 向 `roomId=140789095693824` 发文本；
3. 观察 DB `im_message` 已写入但无 aiclaw 行；订阅 WS 不出现 `streamStart`。

**建议处置**
1. 由 backend-tester 拉起 / 补齐 `aichat-plugins-runtime` 容器（或者其他 openclaw 桥接进程），命中 hula-server-runtime（容器内 18760，外暴露 18080），用"安洁"的 aiclaw token 完成 WS 注册；
2. 上线后 server-dev 复测本报告步骤 4/5；
3. 长期建议：在 `shared/环境与部署/配置/服务端依赖清单与中间件拓扑.md` 中补一节"plugins runtime 容器拓扑"，避免与 dev 混淆。

### 问题 #2：`/chat/msg/page` 漏返刚入库的消息（P1，已闭环 2026-05-12 17:54）

**现象**
- DB：`SELECT COUNT(*) FROM im_message WHERE room_id=140789095693824 AND is_del=0` = **83**；
- API：`GET /api/im/chat/msg/page?roomId=140789095693824&pageSize=100` 返回 `total=81, isLast=true`，缺失就是本次新发的 2 条；
- 这 2 条消息在 `POST /chat/msg` 的响应里已经带上 `id`，且 DB 已有记录，唯独 page 接口看不到。

**根因（backend-tester 定位）**
- `im_contact.last_msg_id` 未随新消息更新 —— `/chat/msg/page` 走的是基于 `im_contact.last_msg_id` 的游标分页，COUNT 被旧游标限住，所以新增到 `im_message` 的两条消息查不到；
- 同时 Redis 中存在 72 条 `luohuo:msg:*` 缓存键残留，需要一并清理。

**处置（backend-tester 执行）**
1. 清理 72 条 `luohuo:msg:*` Redis 缓存键；
2. 把 `im_contact.last_msg_id` 更新到最新 `160442597630976`。

**复测结果（server-dev，2026-05-12 17:54）**
- DB `im_message` count = **84**
- `GET /api/im/chat/msg/page?roomId=140789095693824&pageSize=100` → `total=84, isLast=true`，`items.length=84`
- DB 与 API 完全一致 ✅

**长期建议（待评审）**
- 写入路径加保护：消息落库的同时同步 `im_contact.last_msg_id`，或在 `/chat/msg/page` 入口加一层 DB 与游标的一致性兜底。具体实现需求评审后再上 backlog。

## 三、其它观察

- 历史消息中（2026-04-01 起）出现大量 `RetryPushConsumer.ack失败重新发送消息` 日志，建议关注 WS 推送可靠性问题，看是否与问题 #2 同源。
- `hula-server-runtime` 容器内日志里频繁打印 `IPUtils.getClientIp` / `TokenContextFilter.parseToken`，并夹杂 Netty 堆栈片段（疑似断开重连），但未影响 HTTP 接口。可在后续优化时排查。

## 四、后续动作清单

- [x] backend-tester：清理 Redis room 消息缓存 + 更新 `im_contact.last_msg_id`，page 接口 total 与 DB 一致（2026-05-12 17:54 完成）。
- [x] server-dev：复测 P1 闭环（2026-05-12 17:54 完成，DB=API=84）。
- [ ] backend-tester：runtime 侧 `aichat-claw` 插件已加载（7 plugins），安洁 token 已配置 —— 但 WS inbound 闭环未实现，**转 plugin-dev 补桥接实现**；
- [ ] plugin-dev：补全 `aichat-claw` 在 runtime 的 WebSocket inbound + agent 触发逻辑（接 hula-server-runtime 18760/容器内）；
- [ ] server-dev：plugin-dev 完成后复测 streamStart/Delta/End 推送链路。
