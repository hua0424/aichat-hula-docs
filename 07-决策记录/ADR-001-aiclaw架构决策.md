# ADR-20260315-AIclaw 架构决策

- 状态：Accepted
- 日期：2026-03-15
- 决策人：华血（审批）、老贾（PM）
- 参与方：锋哥（Backend）、阿飞（Frontend）

## 背景

将现有 bot 改造为 AI 助理（aiclaw），通过 aichat-plugins 插件桥接 openclaw 等外部 AI 引擎，接入 aichat IM 体系。需要确定用户模型、连接协议、消息路由、流式传输等核心架构方案。

## 决策清单

### D1：用户模型 — 新建 im_aiclaw 扩展表 + user_type=4

**备选：** A）在 im_user 表加字段 B）新建扩展表

**决策：方案 B** — 新建 `im_aiclaw` 扩展表，`im_user` 只新增 `user_type=4` 枚举值。

**依据：** aiclaw 独有字段（owner_uid, token_hash, machine_code, auth_status 等）放 im_user 会污染通用模型，查询场景也不同。扩展性更好（后续可加 adapter_type 等）。

---

### D2：Token 方案 — UUID + bcrypt，独立于 OAuth

**备选：** A）复用 OAuth/Sa-Token B）HMAC-SHA256 C）UUID + bcrypt

**决策：方案 C** — token = UUID.randomUUID()，存储 bcrypt hash，不复用 OAuth。

**依据：** aiclaw token 是长期设备凭证，不是会话 token。OAuth 体系的刷新/过期/设备类型等概念不适用。UUID + bcrypt 最简单，无需管理 server_secret。

---

### D3：plugins 连接协议 — 复用现有 WS 通道

**备选：** A）独立 TCP/HTTP 通道 B）仅 MQ C）复用 WS

**决策：方案 C** — plugins 作为 aiclaw 用户的一个 WS "设备"接入，复用 `ws://host/api/ws/ws?token=<TOKEN>&clientId=<机器码>`。

**依据：** 一举获得认证、心跳、在线状态、Session 管理、消息推送能力，零额外开发。这是整个架构中影响最大的决策。

---

### D4：消息路由 — 复用现有推送链路，不引入额外 MQ topic

**备选：** A）新增 aiclaw-msg MQ topic B）复用现有推送链路

**决策：方案 B** — 用户消息通过现有 `PushService → SessionManager.sendToDevice()` 推送到 plugins 的 WS 连接。

**依据：** D3 决策后，plugins 已是 WS 设备，消息推送链路自动覆盖。额外 MQ topic 纯属冗余。离线消息由现有离线存储机制处理。

**注意：** 之前讨论的 MQ topic/tag/connect_token 设计全部作废。

---

### D5：流式消息 — WS 三阶段协议（start/delta/end），仅 end 落库

**决策：**
- 新增 `STREAM_START`、`STREAM_DELTA`、`STREAM_END` 三种 WS 事件类型
- start/delta 只走实时 WS，不落库
- end 携带完整内容，作为标准 type=1 消息持久化
- msgId 在 stream_start 时预生成（雪花 ID）
- stream_end 含 status 字段（complete/error）

**依据：** 消息存储模型零改动，改动集中在 WS 传输层（增量变更）。stream_delta 高频小包不适合走 MQ。

---

### D6：消息类型 — type=1 文本，前端按 user_type 判断 AI 样式

**备选：** A）type=1 + userType=4 判断渲染 B）新增 type=17 AI 消息类型

**决策：方案 A** — 消息本质是文本，落库 type=1。前端用 `fromUser.userType === AICLAW` 判断是否启用 AI 渲染样式。

**依据：** 消息内容结构与普通文本完全一致，新增类型会导致反序列化冗余。与现有 Bot 判断逻辑（account === UserType.BOT）思路一致。

---

### D7：机器码 — UUID 持久化到本地文件

**备选：** A）MAC/CPU 硬件 ID B）UUID 持久化

**决策：方案 B** — plugins 首次启动生成 UUID，持久化到本地文件。

**依据：** 硬件 ID 在虚拟机/容器环境不稳定。UUID 持久化简单可控，换机器自然变更，触发重新授权流程。

---

### D8：机器码变更授权 — 新增独立 WS 事件，不复用好友申请

**决策：** 新增 `AICLAW_AUTH_REQUEST` WS 事件类型，独立于好友申请机制。

**依据：** 好友申请有同意/拒绝/过期等复杂状态机，机器码授权只需"确认/拒绝"，语义不同，单独实现更干净。

---

### D9：好友关系 — 可删除（强确认 + 24h 恢复期）

**决策：**
- owner 可删除 aiclaw 好友关系，需输入密码强确认
- 删除后 24 小时恢复期（aiclaw 停止服务，可恢复）
- 24 小时后 XXL-Job 定时任务执行彻底注销

**依据：** 不可删除过于僵硬，但需要防误操作。24h 恢复期兼顾安全与灵活性。

---

### D10：离线消息 — 现有离线存储 + plugins 主动拉取 + 防抖合并

**决策：** 消息由现有离线机制存储，plugins 上线后主动调用 `POST /api/chat/msg/list` 拉取离线消息（复用现有 REST API，server 零改动）。plugins 侧做防抖合并（2s 缓冲 / 5 条上限 / 10s 最大等待）后发往 openclaw。

---

### D11：首期客户端范围 — 移动端 + Web端/桌面端同步

**决策：** 首期同时支持移动端和 Web端/桌面端。

**依据：** 核心代码（WS 层、Store 层、Text.vue）是平台共用的，额外工作量主要在入口和好友列表两处。

---

### D12：msgId 生成 — server 端生成，plugins 不携带

**决策：** plugins 发送的流式事件（start/delta/end）不携带 msgId，由 server 的 StreamProcessor 在收到 stream_start 时生成雪花 ID，中继给用户时附上。

**依据：** 一个 aiclaw 同一时间只有一个活跃流，server 按 fromUid 关联即可。plugins 实现最简化，无需关心 ID 生成。

---

### D13：流式断流兜底 — server 端超时检测与异常恢复

**决策：** Server 的 StreamProcessor 维护活跃流状态（`activeStreams: Map<aiclawUid, ...>`），处理三种异常场景：
- 30s 无 delta/end → 合成 error end 推送用户，不落库
- 断线后重连发新 start → 先推送旧流 error end，再推送新 start
- 超时阈值做成 Nacos 可配置项（默认 30s）

**依据：** server 端统一处理断流，client 无需实现异常状态判断，保证顺序始终正确。

---

### D14：流式期间消息处理 — 统一防抖+堆积，不做严格串行

**决策：** plugins 对所有用户消息（实时和离线）采用统一防抖机制（2s/5条/10s）。活跃流期间新消息堆积，stream_end 到达后堆积消息经防抖批量发送给 openclaw。

**备选：** 严格串行排队（前一个流式回复结束前不处理新消息）

**依据：** 统一防抖逻辑更简洁（DRY），堆积后批量发送给 openclaw 提供更完整上下文，避免串行队列的实现复杂度。

---

### D15：激活流程简化 — 双 token 模型 + CLI 一键激活（v1.2 新增）

**决策：**
- 创建时返回**加密的激活 token**（AES-256-GCM 加密 `{ uid, connectionToken, timestamp }`），不返回明文连接 token
- 激活 token 未激活前可反复查看，激活后隐藏，72 小时过期
- plugins 通过 `aichat activate --backend openclaw --token <激活token>` 一条命令完成激活
- server 新增 `POST /api/aiclaw/activate` 接口，解密激活 token → 校验 → 返回 uid + 连接 token
- plugins 保存连接 token 到 `~/.aichat/credentials.jsonc`，后续 `aichat start` 自动连接
- server 地址由插件内置默认值提供，可通过 `~/.aichat/config.jsonc` 覆盖
- openclaw 配置优先 auto-detect 本地安装，检测不到则读 `~/.aichat/config.jsonc`

**备选：** 手动编辑 .env 文件配置 5 个参数后启动

**依据：** 原流程需要手动编辑 .env 填写 WS_URL/AICLAW_TOKEN/AICLAW_UID/OPENCLAW_URL/OPENCLAW_TOKEN 五个参数，对非技术用户门槛高。新方案用户只需复制粘贴一个 token 即可完成全部配置。加密激活 token 比明文连接 token 安全性更高（一次性使用、有过期时间、明文连接 token 从不暴露给用户）。

---

## 影响

- **正向：** 架构极简（纯 WS，无额外协议），改动量可控（增量变更为主），plugins 实现大幅简化
- **负向：** plugins 对 WS 连接稳定性有依赖，断线重连机制需可靠
- **风险与缓解：** openclaw 会话状态由 openclaw 自行管理，重启可能丢失上下文 → 首期接受，后续可在 plugins 侧增加上下文缓存

## 后续动作

- [x] 锋哥确认流式 payload 格式（2026-03-17，采纳，msgId 由 server 生成）
- [x] 锋哥确认 aiclaw 好友关系创建时机与 room_id 生成（2026-03-17，创建 API 同事务内完成）
- [x] 锋哥确认 plugins 拉取离线消息机制（2026-03-17，主动拉取 REST API）
- [x] 阿飞确认桌面端 AI 助理入口具体位置（2026-03-17，侧边栏插件区）
- [x] 锋哥分配 WSReqTypeEnum/WSRespTypeEnum 具体编号（2026-03-17，Req 17/18/19，Resp camelCase）
- [ ] 需求文档评审通过后更新看板任务
