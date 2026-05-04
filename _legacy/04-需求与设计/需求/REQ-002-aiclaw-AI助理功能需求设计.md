# AIclaw 功能需求设计（v1）

> 状态：**首期已完成**（2026-03-24 联调通过，遗留 B7/B8/B9 延后至下一阶段）
> 版本：v1.2（简化激活流程）
> 日期：2026-03-18（最后更新：2026-03-24）
> 编写：老贾（PM）
> 评审：锋哥（Backend）、阿飞（Frontend）
> 决策记录：`07-决策记录/ADR-001-aiclaw架构决策.md`

---

## 一、概述

### 一句话
将现有 bot 改造为 AI 助理（aiclaw），通过 aichat-plugins 插件桥接 openclaw 等类 claw AI，接入 aichat IM 体系，使 AI 成为社交网络中的一等公民。

### 首期范围
- aiclaw 的创建、连接、注册
- owner 在客户端的授权管理
- server 端自动注册与好友绑定
- owner 与 aiclaw 之间的文字通讯（含流式回复）
- 移动端 + Web端/桌面端同步支持

### 首期不做（架构预留）
- 非 owner 用户与 aiclaw 通讯（需 owner 授权机制）
- aiclaw 与 aiclaw 之间的通讯
- 语音/文件/图片等多模态消息
- 流式中断（停止生成）
- aiclaw 分享给其他用户

---

## 二、术语定义

| 术语 | 说明 |
|------|------|
| **aiclaw** | AI 助理用户，在系统中表现为 user_type=4 的 IM 用户 |
| **owner** | aiclaw 的拥有者，普通用户（user_type=0） |
| **plugins** | aichat-plugins，独立 Node.js 进程，桥接 aiclaw 与 openclaw |
| **openclaw** | 外部 AI 引擎（如 openclaw），通过 Chat Completions API 提供 AI 能力 |
| **机器码** | plugins 生成的 UUID，持久化到本地文件，标识设备唯一性 |

---

## 三、数据模型

### 3.1 im_user 扩展

新增 user_type 枚举值：

```java
enum UserTypeEnum {
    // ... 现有值 ...
    AICLAW(4, "AI助理")
}
```

aiclaw 创建时在 `im_user` 表插入一条记录，`name`、`avatar` 等基础信息存于此表。

### 3.2 im_aiclaw 扩展表（新建）

```sql
CREATE TABLE im_aiclaw (
    id             BIGINT PRIMARY KEY,
    uid            BIGINT NOT NULL UNIQUE,        -- 关联 im_user.id
    owner_uid      BIGINT NOT NULL,               -- owner 用户 ID
    token_hash     VARCHAR(128) NOT NULL,          -- bcrypt hash
    machine_code   VARCHAR(64),                    -- plugins 设备标识（UUID）
    auth_status    TINYINT DEFAULT 0,              -- 0=未激活 1=已激活 2=已停用
    adapter_type   VARCHAR(32) DEFAULT 'openclaw', -- claw 类型
    adapter_config TEXT,                           -- 连接参数（JSON）
    deactivated_at DATETIME,                       -- 停用时间（用于延迟注销）
    created_at     DATETIME,
    updated_at     DATETIME,
    INDEX idx_owner(owner_uid),
    INDEX idx_machine(machine_code)
);
```

### 3.3 字段说明

| 字段 | 说明 |
|------|------|
| `uid` | 关联 im_user.id，aiclaw 的用户 ID |
| `owner_uid` | 拥有者的用户 ID |
| `token_hash` | aiclaw 连接凭证的 bcrypt hash，明文只在创建时展示一次 |
| `machine_code` | plugins 首次连接时上报，后续连接校验一致性 |
| `auth_status` | 0=未激活（已创建未连接）、1=已激活（正常使用）、2=已停用（待注销） |
| `adapter_type` | 预留，标识对接的 AI 引擎类型 |
| `adapter_config` | 预留，连接参数 JSON |
| `deactivated_at` | 停用时间，24 小时后由定时任务执行彻底注销 |

---

## 四、核心流程

### 4.1 创建 aiclaw

```
Owner 客户端                          Server
    │                                    │
    │  POST /api/aiclaw/create           │
    │  { name, avatar?, description? }   │
    │ ──────────────────────────────────→ │
    │                                    │ 1. 创建 im_user (user_type=4)
    │                                    │ 2. 生成连接 token = UUID.randomUUID()
    │                                    │ 3. 存 im_aiclaw (token_hash=bcrypt(token), auth_status=0)
    │                                    │ 4. 加密生成激活 token = AES-256-GCM({ uid, connectionToken, timestamp })
    │  { aiclaw_uid, activationToken }   │
    │ ←────────────────────────────────── │
    │                                    │
    │  展示激活 token（未激活前可反复查看） │
```

**说明：**
- server 生成 UUID 连接 token，bcrypt hash 入库，**明文连接 token 不返回给前端**
- server 用加密密钥（Nacos 配置 `aiclaw.activation.secret`）将 `{ uid, connectionToken, timestamp }` 加密为激活 token，返回给前端
- 激活 token 在 aiclaw **未激活前可反复查看**，激活后自动隐藏
- owner 可点击"重新生成激活码"刷新激活 token（连接 token 不变，只刷新 timestamp）
- 激活 token 有过期时间（创建后 72 小时），过期后需重新生成
- 创建 API 在同一个事务内完成全部操作：
  1. 创建 im_user (user_type=4)
  2. 生成连接 token，创建 im_aiclaw (token_hash=bcrypt)
  3. `roomService.createFriendRoom([ownerUid, aiclawUid])` — 复用现有方法
  4. `friendService.createFriend(ownerUid, aiclawUid, roomId)` — 双向好友记录
  5. 加密生成激活 token
  6. 返回 { aiclaw_uid, activationToken }

**加密算法：** AES-256-GCM（对称加密 + 完整性校验），Java/Node.js 原生支持。

### 4.2 aiclaw 激活与连接

激活分两步：**CLI 激活**（一次性）→ **WS 连接**（持久）。

**第一步：CLI 激活**

```
用户在 openclaw 机器执行：
aichat activate --backend openclaw --token <激活token>

plugins                                 Server
    │                                      │
    │  POST /api/aiclaw/activate           │
    │  { activationToken, machineCode }    │
    │ ────────────────────────────────────→ │
    │                                      │ 1. AES-256-GCM 解密激活 token
    │                                      │ 2. 校验 timestamp 未过期（72h）
    │                                      │ 3. bcrypt.verify(connectionToken, token_hash)
    │                                      │ 4. 更新 auth_status=0→1，绑定 machineCode
    │                                      │ 5. 写 Redis 缓存（供 gateway 快速校验）
    │  { uid, connectionToken }            │
    │ ←──────────────────────────────────── │
    │                                      │
    │  保存到 ~/.aichat/credentials.jsonc  │
    │  auto-detect openclaw 配置           │
    │  （检测不到则读 ~/.aichat/config.jsonc）│
```

**第二步：WS 连接（与原流程一致）**

```
plugins                                 Server                              Owner 客户端
    │                                      │                                    │
    │  WS连接                               │                                    │
    │  ws://host/ws/ws                     │                                    │
    │  ?token=<连接token>&clientId=<机器码> │                                    │
    │ ────────────────────────────────────→ │                                    │
    │                                      │ 校验连接 token + machineCode       │
    │                                      │                                    │
    │                                      │  推送在线状态变更                    │
    │                                      │ ──────────────────────────────────→ │
    │  连接成功                             │                                    │
    │ ←──────────────────────────────────── │                                    │
    │                                      │                                    │
    │  后续重连：自动读取本地 credentials    │                                    │
    │  无需用户介入                          │                                    │
```

**plugins 配置体系：**

```jsonc
// ~/.aichat/config.jsonc（用户可选编辑）
{
  // server 地址（不配则使用插件内置默认值）
  "server": {
    "url": "ws://hula-server:18760/ws/ws"
  },
  // openclaw 配置（auto-detect 失败时使用）
  "openclaw": {
    "url": "http://localhost:18789/v1/chat/completions",
    "token": "gateway-token",
    "agentId": "main"
  }
}
```

```jsonc
// ~/.aichat/credentials.jsonc（activate 后自动生成，用户不需要手动编辑）
{
  "uid": 100001,
  "connectionToken": "a3f7b2e1-9c4d-...",
  "machineCode": "uuid-机器码",
  "activatedAt": "2026-03-18T10:30:00"
}
```

**plugins 运行模式：**
- `aichat activate --backend openclaw --token <激活token>` — 首次激活（一次性）
- `aichat start` — 日常运行（读取本地 credentials 自动连接）

**Token 校验分支（不变）：**
- `TokenContextFilter` 校验**连接 token**（不是激活 token）
- 识别到 aiclaw 连接 token 后走 `im_aiclaw` 表校验，不走 Sa-Token

**连接协议（不变）：**
- 复用现有 WS 通道
- `clientId` = 机器码（UUID）
- plugins 视为 aiclaw 用户的一个 WS "设备"
- 心跳、在线状态、Session 管理全部复用现有机制

### 4.3 机器码变更授权

当 plugins 携带有效 token 但机器码与记录不一致时：

```
plugins（新机器）               Server                              Owner 客户端
    │                              │                                    │
    │  WS连接                       │                                    │
    │  token有效，机器码不匹配       │                                    │
    │ ──────────────────────────→   │                                    │
    │                              │  推送 AICLAW_AUTH_REQUEST          │
    │                              │  { aiclaw_uid, aiclaw_name,       │
    │                              │    old_machine, new_machine }      │
    │                              │ ──────────────────────────────────→ │
    │                              │                                    │
    │  连接挂起等待                  │                    Owner 同意/拒绝  │
    │                              │ ←────────────────────────────────── │
    │                              │                                    │
    │  同意 → 更新 machine_code     │                                    │
    │         连接成功              │                                    │
    │  拒绝 → 连接断开             │                                    │
    │ ←────────────────────────────│                                    │
```

**说明：**
- 不复用好友申请机制，新增独立的 WS 事件类型 `AICLAW_AUTH_REQUEST`
- 如果 owner 不在线，请求挂起，等 owner 上线后推送（或设超时，超时拒绝）
- 通知 UI 复用现有通知组件，新增 `AICLAW_DEVICE_AUTH` 通知类型

### 4.4 用户与 aiclaw 普通消息

消息流完全复用现有 IM 链路，无需额外改动：

```
用户发消息给 aiclaw：
  用户 WS → Server 消息处理 → 存库 → PushService → aiclaw 的 WS Session → plugins 收到

aiclaw 回复（非流式，如错误提示）：
  plugins WS → Server 消息处理 → 存库 → PushService → 用户的 WS Session → 用户收到
```

**零改动。** aiclaw 在 server 中就是一个普通 WS 用户，消息推送由 `PushService → SessionManager.sendToDevice()` 自动完成。

### 4.5 aiclaw 流式回复

**msgId 由 server 生成，plugins 无需关心。** plugins 发送的流式事件不带 msgId，server 的 StreamProcessor 收到 `stream_start` 后生成雪花 ID，中继给用户时附上。一个 aiclaw 同一时间只有一个活跃流，server 按 fromUid 关联。

```
plugins                          Server                         用户客户端
    │                               │                               │
    │  WS: STREAM_START             │                               │
    │  { fromUid, toUid, roomId }   │  生成 msgId（雪花ID）          │
    │ ────────────────────────────→ │  中继: { msgId, fromUid,      │
    │                               │         toUid, roomId }       │
    │                               │ ────────────────────────────→  │
    │                               │                               │ 创建临时消息气泡
    │                               │                               │
    │  WS: STREAM_DELTA (多次)      │                               │
    │  { chunk, seq }               │  中继: { msgId, chunk, seq }  │
    │ ────────────────────────────→ │ ────────────────────────────→  │
    │                               │                               │ 追加文字（rAF节流）
    │                               │                               │
    │  WS: STREAM_END              │                                │
    │  { fullContent, status }      │  1. 中继: { msgId,            │
    │ ────────────────────────────→ │     fullContent, status }     │
    │                               │  2. status=complete → 落库     │
    │                               │ ────────────────────────────→  │
    │                               │                               │ 替换为最终内容
```

**Payload 分层（plugins → server vs server → client）：**

| 事件 | plugins → server | server → client |
|------|-----------------|-----------------|
| STREAM_START | `{ fromUid, toUid, roomId }` | `{ msgId, fromUid, toUid, roomId }` |
| STREAM_DELTA | `{ chunk, seq }` | `{ msgId, chunk, seq }` |
| STREAM_END | `{ fullContent, status }` | `{ msgId, fullContent, status }` |

**关键规则：**
- `stream_start` 和 `stream_delta` 只走 WS 实时通道，**不落库**
- `stream_end` 携带完整内容，作为标准 IM 消息持久化（type=1 文本，fromUid=aiclaw_uid）
- `msgId` 由 server 在收到 `stream_start` 时预生成（雪花 ID），全程携带
- `stream_end.status` 取值：`complete`（正常完成）或 `error`（生成失败）
- status=complete 时落库；status=error 时**不落库**，只推送给用户
- 失败时 `fullContent` 为错误提示文案："抱歉，我的大脑目前宕机了，请主人关心一下我的身体情况"
- **流式期间消息堆积**：当 aiclaw 正在流式回复时（活跃流存在），新收到的用户消息堆积在缓冲区，等待当前 `stream_end` 到达后，堆积消息再经过防抖处理（与离线消息相同的 2s/5条/10s 规则）批量发往 openclaw。详见 4.6 节
- 前端通过 `fromUid` 从本地缓存（`groupStore.getUserInfo()`）获取用户信息渲染头像和 AI 标识，`stream_start` 不冗余携带用户信息

**流式断流兜底机制（Server StreamProcessor）：**

Server 维护活跃流状态：`activeStreams: Map<aiclawUid, { msgId, toUid, roomId, lastDeltaTime }>`

| 场景 | 处理 |
|------|------|
| 正常完成 | start → delta... → end(complete) → 落库 |
| 超时（30s 无 delta/end） | server 合成 end(status=error, fullContent="回复中断，请重试") 推送给用户，**不落库** |
| 断线重连发新 start | server 先推送旧流的 end(error)，再推送新 start。client 无需处理异常状态 |

> 超时阈值（默认 30s）做成 Nacos 可配置项，后续根据实际场景调优。

**Server 实现：**
- `WSReqTypeEnum` 新增：`STREAM_START(17)`、`STREAM_DELTA(18)`、`STREAM_END(19)`
- `WSRespTypeEnum` 新增：`"streamStart"`、`"streamDelta"`、`"streamEnd"`、`"aiclawAuthRequest"`
- 新增 `StreamProcessor`（挂到 `MessageHandlerChain`）
- stream_start：生成 msgId，注册 activeStreams，转发给目标用户
- stream_delta：更新 lastDeltaTime，转发给目标用户，不存库
- stream_end：转发 + 若 status=complete 则构造标准 Message 实体调用现有存储 Service 落库 + 清理 activeStreams

### 4.6 消息防抖与流式期间堆积

plugins 对所有收到的用户消息（无论实时还是离线积压）采用统一的防抖机制，并在流式期间自动堆积。

**完整消息处理模型（plugins 侧）：**

```
收到用户消息
    │
    ├─ 当前无活跃流 → 进入防抖缓冲区
    │   ├─ 2 秒内有新消息 → 重置 2 秒计时器
    │   ├─ 累积达 5 条     → 立即合并发送给 openclaw
    │   └─ 总等待超 10 秒  → 无论多少条，立即合并发送
    │   └─ 发送后 → 收到 stream_start → 进入活跃流状态
    │
    └─ 当前有活跃流（stream_start 已发出，stream_end 未到达）
        → 消息进入堆积区，不发送
        → 等待 stream_end 到达
        → stream_end 后，堆积消息进入防抖缓冲区（同上规则）
        → 批量发送给 openclaw → 新的活跃流
```

**离线消息处理：**
- plugins 重连 WS 后，主动调用 `POST /api/chat/msg/list` 拉取离线消息（复用现有 REST API，server 零改动）
- 拉取到的离线消息按时间排序，进入同一套防抖流程

**合并规则：**
- 多条消息按 `sent_at` 时间排序，以换行分隔拼接为一条，发往 openclaw
- 消息体需携带 `sent_at`（Unix 毫秒时间戳），plugins 可区分实时 vs 积压消息

**设计优势：**
- 实时消息和离线消息使用同一套防抖逻辑，DRY
- 流式期间的消息自然堆积，避免并发请求 openclaw
- stream_end 后堆积消息批量发送，给 openclaw 更完整的上下文

### 4.7 好友删除与 aiclaw 注销

```
Owner 删除 aiclaw 好友
    │
    ├─ 弹出强确认 Dialog
    │   "删除后 token 失效，AI 助理将断开连接"
    │   需输入密码确认
    │
    ├─ 确认后 → server 操作：
    │   1. im_aiclaw.auth_status = 2（已停用）
    │   2. im_aiclaw.deactivated_at = now()
    │   3. 主动关闭 aiclaw 的 WS 连接
    │   4. 通讯录标记"已停用"
    │
    ├─ 24 小时内：
    │   - aiclaw 不能收发消息
    │   - plugins 尝试重连 → server 拒绝（auth_status=2）
    │   - owner 可点击"恢复" → auth_status=1，清空 deactivated_at
    │
    └─ 24 小时后（XXL-Job 定时扫描，每小时一次）：
        - 彻底注销：删 im_user + 删 im_aiclaw + 删好友关系
        - token 自然失效（记录已删除）
```

### 4.8 重新生成激活码

> ~~原设计有"Token 重置"和"刷新激活码"两个操作，v1.2 合并为统一的"重新生成激活码"。~~

- 入口：AI 助理管理页 / 好友详情页
- **在线时不可操作**：按钮置灰，提示"请先断开 AI 助理连接"
- 只有 aiclaw 离线或未激活状态下才能操作
- 操作效果：生成新连接 token + 新激活 token → 旧连接 token 立即失效 → aiclaw 回到"未激活"（auth_status=0）
- plugins 侧本地 credentials 失效，需重新执行 `aichat activate`
- 前端展示：复用 AiclawTokenDialog 展示新激活 token，未激活前可反复查看

---

## 五、接口契约

### 5.1 WS 事件类型新增（已确认）

**Client/Plugins → Server 请求（WSReqTypeEnum，Integer 类型，接续现有 1~16）：**

| 编号 | 名称 | 说明 |
|------|------|------|
| 17 | `STREAM_START` | plugins 发起流式消息 |
| 18 | `STREAM_DELTA` | plugins 发送流式片段 |
| 19 | `STREAM_END` | plugins 发送流式结束 |

**Server → Client 推送（WSRespTypeEnum，String 类型，camelCase 风格）：**

| 类型值 | 名称 | dataClass | 说明 |
|--------|------|-----------|------|
| `"streamStart"` | STREAM_START | StreamStartDTO | 流式消息开始 |
| `"streamDelta"` | STREAM_DELTA | StreamDeltaDTO | 流式消息片段 |
| `"streamEnd"` | STREAM_END | StreamEndDTO | 流式消息结束 |
| `"aiclawAuthRequest"` | AICLAW_AUTH_REQUEST | AiclawAuthRequestDTO | 机器码变更授权 |

### 5.2 流式消息 Payload 格式（已确认）

> msgId 由 server 生成，plugins 端不携带。两层 payload 设计。

**plugins → server（WSReqTypeEnum）：**

```json
// STREAM_START (17)
{ "type": 17, "data": { "fromUid": 100001, "toUid": 200001, "roomId": 300001 } }

// STREAM_DELTA (18)
{ "type": 18, "data": { "chunk": "这是一段", "seq": 1 } }

// STREAM_END (19)
{ "type": 19, "data": { "fullContent": "这是一段完整的 AI 回复内容。", "status": "complete" } }
```

**server → 用户客户端（WSRespTypeEnum）：**

```json
// streamStart
{
    "type": "streamStart",
    "data": {
        "msgId": "1234567890123456789",
        "fromUid": 100001,
        "toUid": 200001,
        "roomId": 300001
    }
}

// streamDelta
{
    "type": "streamDelta",
    "data": {
        "msgId": "1234567890123456789",
        "chunk": "这是一段",
        "seq": 1
    }
}

// streamEnd
{
    "type": "streamEnd",
    "data": {
        "msgId": "1234567890123456789",
        "fullContent": "这是一段完整的 AI 回复内容。",
        "status": "complete"
    }
}
```

> status 取值：`complete`（正常，落库）、`error`（失败，不落库，fullContent 为错误提示）
> seq 字段保留，server 不校验连续性，用于前端调试和日志排查

**AICLAW_AUTH_REQUEST（server → owner 客户端）：**
```json
{
    "type": "aiclawAuthRequest",
    "data": {
        "aiclawUid": 100001,
        "aiclawName": "我的AI助理",
        "oldMachineCode": "uuid-old",
        "newMachineCode": "uuid-new"
    }
}
```

### 5.3 aiclaw 管理 HTTP API

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| POST | `/api/aiclaw/create` | 创建 aiclaw，返回激活 token | owner 登录态 |
| POST | `/api/aiclaw/activate` | **激活（plugins 调用）**，解密激活 token → 返回 uid + 连接 token | 无需登录态，凭激活 token |
| GET | `/api/aiclaw/list` | 获取 owner 的 aiclaw 列表 | owner 登录态 |
| GET | `/api/aiclaw/{uid}/activation-token` | **获取激活 token**（未激活时可反复查看） | owner 登录态，aiclaw 须未激活 |
| POST | `/api/aiclaw/{uid}/refresh-activation` | **重新生成激活码**（新连接 token + 新激活 token，回到未激活） | owner 登录态，aiclaw 须离线或未激活 |
| PUT | `/api/aiclaw/{uid}/profile` | 修改 aiclaw 资料 | owner 登录态 |
| POST | `/api/aiclaw/{uid}/deactivate` | 停用（触发24h注销） | owner 登录态，需密码 |
| POST | `/api/aiclaw/{uid}/restore` | 恢复（24h内） | owner 登录态 |
| POST | `/api/aiclaw/{uid}/auth-confirm` | 机器码变更授权确认 | owner 登录态 |

> 具体请求/响应体遵循现有统一响应格式（code/success/msg/data）。

---

## 六、前端规格

### 6.1 创建入口

**位置：** 独立入口

| 端 | 入口位置 | 组件 | 说明 |
|----|---------|------|------|
| 移动端 | "我的"页面 cell 入口 | `mobile/views/my/AiAssistant.vue`（路由已预留） | Settings 工具栏或个人信息区下方 |
| 桌面端/Web端 | 侧边栏插件区（与"机器人"同级） | 新建 `views/aiAssistantWindow/index.vue` | 独立窗口（约 1000x700） |

**桌面端改动文件：**
1. `src/layout/left/config.tsx` — `basePluginsList` 加一项
2. `src/router/index.ts` — `getDesktopRoutes()` 加路由
3. 新建 `src/views/aiAssistantWindow/index.vue` — 管理页面（创建表单 + aiclaw 列表 + token 管理）

**创建表单字段：**
- 名称（必填）
- 头像（可选，有默认 AI 头像）
- 简介（可选）

**创建成功后：**
- 弹出 Dialog 展示激活 token，带一键复制按钮
- **未激活前可在 AI 助理管理页反复查看激活 token**（调用 `GET /api/aiclaw/{uid}/activation-token`）
- 激活后激活 token 自动隐藏，显示"已激活"状态
- 提供"重新生成激活码"按钮（调用 `POST /api/aiclaw/{uid}/refresh-activation`，刷新 timestamp）

### 6.2 通讯录展示

**AI 标识：**
- aiclaw 好友头像右下角叠加 AI 小图标（与 Bot 区分）
- 判断逻辑：`fromUser.userType === UserType.AICLAW (4)`

**在线状态（三态）：**

| 状态 | 视觉 | 含义 |
|------|------|------|
| 未激活 | 灰色 + "未激活"标签 | 已创建，token 未使用 |
| 在线 | 绿色圆点（复用现有） | plugins 已连接 |
| 离线 | 灰色圆点（复用现有） | plugins 未连接 |

**好友删除：**
- 复用好友管理界面
- 删除 aiclaw 好友时触发强确认 Dialog
- Dialog 文案："删除后 token 失效，AI 助理将断开连接"
- 需输入密码确认

### 6.3 聊天窗口

**顶部提示条：**
- 与 aiclaw 聊天时，顶部显示提示条："AI 助理对话 · 回复由 AI 生成"
- 流式期间变为"正在回复..."
- 实现：在聊天头部组件判断 `currentContact.userType === AICLAW`，条件渲染

**输入限制：**
- 首期与 aiclaw 聊天时，语音/文件/图片按钮置灰（`:disabled="isAiChat"`）
- 后续支持多模态时去除限制

### 6.4 流式渲染

**消息组件：** 复用现有 `Text.vue`，不新建组件

**渲染流程：**

```
stream_start → chatStore.pushMsg({
    type: MsgEnum.TEXT,  // type=1，标准文本
    fromUser: { userType: AICLAW },
    body: { content: '', streaming: true },
    tempMsgId
})

stream_delta → chatStore.appendStreamContent(tempMsgId, deltaText)
    // 追加到 content

stream_end → chatStore.updateMsg({
    msgId: tempMsgId,
    body: { content: finalContent, streaming: false },
    newMsgId: realMsgId
})
```

**性能优化：**

```javascript
// rAF 节流（防止高频 DOM 更新掉帧）
let buffer = ''
let rafId = null

function onStreamDelta(delta) {
    buffer += delta
    if (!rafId) {
        rafId = requestAnimationFrame(() => {
            chatStore.appendStreamContent(msgId, buffer)
            buffer = ''
            rafId = null
        })
    }
}
```

- 流式期间跳过 URL/mention 解析（`Text.vue` 的 fragment 计算），`stream_end` 后再完整解析
- `v-memo` 依赖项中加入 `message.body?.content` 或 `streaming` 状态
- 流式期间文末显示 CSS 闪烁光标，`stream_end` 后移除

### 6.5 设备授权通知

- `NoticeType` 枚举新增 `AICLAW_DEVICE_AUTH`
- 通知列表（`MobileApplyList.vue` / 桌面端通知组件）新增 case
- 文案："你的 AI 助理 [name] 尝试从新设备登录，是否授权？"
- 复用现有同意/拒绝按钮

### 6.6 激活码管理

- 入口：AI 助理管理页 / 好友详情页
- **未激活时**：显示"查看激活码"按钮（调用 `GET /api/aiclaw/{uid}/activation-token`）+ "重新生成激活码"按钮
- **已激活时**：显示"重新生成激活码"按钮，在线时 disabled + tooltip "请先断开连接"
- "重新生成激活码"需二次确认（提示"旧连接将立即失效，需要重新激活"）
- 生成成功后复用 AiclawTokenDialog 展示新激活 token

### 6.7 前端改动矩阵

| 改动点 | 移动端 | 桌面端/Web端 | 说明 |
|--------|--------|-------------|------|
| 创建入口 | `mobile/views/my/AiAssistant.vue` | `views/aiAssistantWindow/index.vue` | 路由+表单 |
| 好友列表 AI 标识 | `mobile/views/friends/index.vue` | `views/homeWindow/FriendsList.vue` | userType 判断 |
| 通知 | `MobileApplyList.vue` | 桌面端通知组件 | 新增类型分支 |
| 流式渲染 | 共用 `Text.vue` | 共用 | 同一组件 |
| WS 事件 | `webSocketWeb.ts` | `webSocketRust.ts` | 加 3 个事件映射 |
| Store | 共用 `chat.ts` | 共用 | 新增 appendStreamContent |
| 聊天输入框 | 移动端输入组件 | `ChatFooter.vue` | aiclaw 判断置灰 |

---

## 七、架构全景

```
┌──────────┐         ┌─────────────┐         ┌──────────────────┐         ┌──────────┐
│  用户端  │──WS────→│ HuLa-Server │──WS────→│  aichat-plugins  │──SSE───→│ openclaw │
│          │←─WS────│             │←─WS────│  (作为 aiclaw    │←─SSE───│          │
│          │         │             │         │   的 WS 设备)    │         │          │
└──────────┘         └─────────────┘         └──────────────────┘         └──────────┘
                      ↑ Server 内部仍用
                      ↑ MQ 做跨节点路由
                      ↑ (现有机制，不变)
```

**关键设计决策：**
- plugins 复用现有 WS 通道，不引入额外协议
- plugins 对 server 来说就是 aiclaw 用户的一个 WS 设备
- 消息投递、在线状态、心跳全部复用现有机制
- 不需要额外的 MQ topic 或 tag

---

## 八、首期不做 / 架构预留

| 功能 | 预留方式 |
|------|---------|
| 非 owner 与 aiclaw 通讯 | im_aiclaw 可扩展授权字段；消息表可加 `authorized_by` |
| aiclaw ↔ aiclaw 通讯 | aiclaw 本身是 WS 用户，天然支持互发消息 |
| 停止生成 | `stream_start` 中预留 `streamId`，未来新增 `stream_cancel` 事件 |
| 多模态消息 | 去除前端输入限制即可，消息格式天然支持 |
| 非 owner 安全机制 | 消息经过 plugins 时可注入安全 prompt（"此消息来自非 owner 用户"） |
| owner 查看他人对话 | 消息存库时记录 room_id，owner 有权限查询 |
| owner 注入/修改 prompt | plugins 侧在转发消息前支持 prompt 拼接 |

---

## 九、任务拆解

### 后端（锋哥）

| # | 任务 | 优先级 | 依赖 |
|---|------|--------|------|
| B1 | 新建 im_aiclaw 表 + UserTypeEnum 加 AICLAW(4) | P0 | 无 |
| B2 | aiclaw 创建/列表/修改/重置 token/停用/恢复 API | P0 | B1 |
| B3 | TokenContextFilter 增加 aiclaw token 校验分支 | P0 | B1 |
| B4 | ReactiveWebSocketHandler 识别 aiclaw 设备上线 | P0 | B3 |
| B5 | WSReqTypeEnum/WSRespTypeEnum 新增流式事件类型 | P0 | 无 |
| B6 | StreamProcessor 实现（中继 + stream_end 落库） | P0 | B5 |
| B7 | AICLAW_AUTH_REQUEST 事件推送（机器码变更） | P1 | B4 |
| B8 | XXL-Job 延迟注销定时任务 | P1 | B2 |
| B9 | 消息体增加 sent_at 时间戳（供 plugins 防抖用） | P2 | 无 |

### 前端（阿飞）

| # | 任务 | 优先级 | 依赖 |
|---|------|--------|------|
| F1 | AiAssistant.vue 创建表单 + token 展示 Dialog | P0 | B2 |
| F2 | 流式渲染：chat.ts appendStreamContent + rAF 节流 | P0 | B5/B6 |
| F3 | 流式渲染：Text.vue streaming 状态 + 光标动画 | P0 | F2 |
| F4 | WS 事件映射（webSocketWeb.ts / webSocketRust.ts） | P0 | B5 |
| F5 | 通讯录 aiclaw 标识 + 三态在线状态 | P1 | B4 |
| F6 | 聊天窗口顶部提示条 + 输入按钮置灰 | P1 | F5 |
| F7 | AICLAW_DEVICE_AUTH 通知类型 | P2 | B7 |
| F8 | Token 重置 UI（详情页按钮 + 在线状态联动） | P2 | B2 |
| F9 | 好友删除强确认 Dialog（密码输入） | P2 | B2 |

### plugins（锋哥）

| # | 任务 | 优先级 | 依赖 |
|---|------|--------|------|
| P1 | 项目脚手架初始化（Node.js + WS client） | P0 | 无 |
| P2 | WS 连接 server（token + 机器码认证） | P0 | B3/B4 |
| P3 | 接收用户消息 → 转发 openclaw Chat Completions API | P0 | P2 |
| P4 | openclaw SSE 流式响应 → 转为 stream_start/delta/end 发回 server | P0 | P3 |
| P5 | 离线消息防抖合并 | P1 | P3 |
| P6 | 错误处理（openclaw 超时/报错 → stream_end status=error） | P1 | P4 |
| P7 | 机器码生成与持久化（UUID → 本地文件） | P0 | 无 |

### 联调顺序建议

```
Phase 1（基础链路）：B1 → B2 → B3 → B4 → P1 → P2 → P7
    目标：aiclaw 能创建，plugins 能通过 WS 连上 server

Phase 2（消息打通）：B5 → B6 → P3 → P4 → F4 → F2 → F3
    目标：用户发消息 → aiclaw 流式回复 → 前端展示

Phase 3（体验完善）：F1 → F5 → F6 → B7 → F7 → B8 → F8 → F9
    目标：创建入口、通讯录展示、通知、Token 管理
```

---

## 十、已确认项（评审闭环）

| # | 事项 | 结论 | 确认人 | 确认日期 |
|---|------|------|--------|---------|
| 1 | 流式 payload 格式 | 采纳，msgId 由 server 生成（plugins 不携带），两层 payload 设计。见 5.2 节 | 锋哥 + 阿飞 | 2026-03-17 |
| 2 | room_id 创建时机 | 在 `POST /api/aiclaw/create` 同一事务内，复用 `roomService.createFriendRoom()` | 锋哥 | 2026-03-17 |
| 3 | 离线消息拉取 | plugins 主动调用 `POST /api/chat/msg/list`（复用现有 REST API，server 零改动） | 锋哥 | 2026-03-17 |
| 4 | WS 枚举编号 | WSReqTypeEnum: 17/18/19；WSRespTypeEnum: "streamStart"/"streamDelta"/"streamEnd"/"aiclawAuthRequest" | 锋哥 | 2026-03-17 |
| 5 | 桌面端入口 | 侧边栏插件区（与"机器人"同级），独立窗口 `views/aiAssistantWindow/index.vue` | 阿飞 | 2026-03-17 |

## 十一、已关闭讨论项

| # | 事项 | 结论 | 确认日期 |
|---|------|------|---------|
| 1 | 流式超时阈值 | 默认 30s，做成 Nacos 可配置项 | 2026-03-17 |
| 2 | 流式期间消息处理 | 不做严格串行排队，采用统一防抖+堆积机制：活跃流期间消息堆积，stream_end 后堆积消息经防抖批量发送。详见 4.6 节 | 2026-03-17 |
