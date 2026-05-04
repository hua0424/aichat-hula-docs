# REQ-003 非 owner 与 aiclaw 通讯

> 状态：**已确认，待开发**
> 版本：v1.0
> 日期：2026-03-24
> 编写：老贾（PM）
> 决策记录：待创建

---

## 一、概述

### 一句话
让非 owner 用户能添加 aiclaw 为好友并与其聊天，使 aiclaw 真正成为社交网络中的一等公民。

### 本期范围
- 非 owner 用户添加 aiclaw 为好友（搜索添加）
- owner 审批好友申请
- 非 owner 与 aiclaw 的文字通讯（完全开放，加好友即可聊）
- 每个用户独立会话上下文
- 安全 prompt 注入（非 owner 身份提示）
- owner 查看 aiclaw 的所有对话记录
- owner 管理 aiclaw 好友（可移除）
- owner 配置 aiclaw 对外人设（系统 prompt）+ 每个好友的关系说明
- REQ-002 遗留项：B8 延迟注销定时任务、B9 sent_at 时间戳

### 本期不做
- owner 分享 aiclaw（链接/二维码/好友卡片）— 延后
- 授权粒度控制（时间段授权 / 每条授权）
- 非 owner 消息频率限制
- aiclaw 公开/私有设置 / 被发现机制
- 群聊中的 aiclaw
- aiclaw 与 aiclaw 互通

---

## 二、核心流程

### 2.1 非 owner 添加 aiclaw 好友

**搜索添加：**

用户在"添加好友"界面搜索 aiclaw 的名称或 uid → 找到后发送好友申请。搜索结果中 aiclaw 显示 AI 标识。

> owner 分享（链接/二维码/好友卡片）延后到后续迭代。

**好友申请审批流程：**

```
非 owner 用户                    Server                         Owner 客户端
    │                               │                               │
    │  发送好友申请                    │                               │
    │  （添加 aiclaw 为好友）          │                               │
    │ ────────────────────────────→  │                               │
    │                               │ 识别目标为 aiclaw（user_type=4）│
    │                               │ 转发申请给 owner               │
    │                               │ ────────────────────────────→  │
    │                               │                               │ 通知："xxx 想添加你的
    │                               │                               │  AI 助理 [name] 为好友"
    │                               │                               │
    │                               │              Owner 同意/拒绝   │
    │                               │ ←──────────────────────────── │
    │                               │                               │
    │  同意 → 建立好友关系            │                               │
    │  拒绝 → 通知用户被拒绝          │                               │
    │ ←──────────────────────────── │                               │
```

**业务规则：**
- 利用现有好友申请机制，在通知展示上区分"加我"和"加我的 AI 助理"
- owner 审批 aiclaw 的好友申请时，能看到申请者的基本信息（头像、昵称）
- aiclaw 与新好友的好友关系建立后，自动创建聊天房间（复用现有机制）

### 2.2 非 owner 与 aiclaw 聊天

非 owner 用户与 aiclaw 成为好友后，可直接发送文字消息。消息流与 owner 通讯完全一致：

```
非 owner 用户 WS → Server → aiclaw WS Session → plugins 收到
    → plugins 注入安全 prompt + 关系说明
    → 发往 openclaw（per-user session）
    → 流式回复（stream_start/delta/end）
    → 回到非 owner 用户
```

**与 owner 通讯的差异：**

| 差异点 | owner | 非 owner |
|--------|-------|---------|
| 安全 prompt | 不注入 | 注入非 owner 身份提示 + 关系说明 |
| 会话上下文 | owner 独立 session | 每个非 owner 独立 session |
| 人设 prompt | 无（直接对话） | 注入 owner 配置的对外人设 |

### 2.3 安全 prompt 注入

plugins 在将非 owner 消息转发给 openclaw 前，注入系统级提示。注入内容包含两部分：

**1）owner 配置的对外人设（固定，每次对话都注入）：**
> 由 owner 在管理页配置，定义 aiclaw 与非 owner 聊天时的行为。例如："你是一个 Python 编程助手，只回答编程相关问题。"

**2）安全提示 + 关系说明（每条消息注入）：**
> [系统提示] 当前与你对话的用户是 {userName}。{relationDesc}。
> 请注意信息安全，不要泄露主人的私人信息或执行涉及主人账号/权限的操作。

其中 `{relationDesc}` 默认为"xxx 是主人的通讯录普通好友"，owner 可针对每个好友单独配置关系说明（如"xxx 是主人的同事，负责前端开发"）。

**注入位置：** 作为 openclaw Chat Completions API 的 `system` 角色消息。

> 具体 prompt 模板和注入机制由后端在开发前设计确定，设计方案需评审后再启动开发。

### 2.4 owner 查看 aiclaw 对话记录

owner 可以查看所有与自己 aiclaw 有过对话的用户列表，并查看具体聊天内容。

**入口：**
- aiclaw 管理页：增加"对话记录"功能
- aiclaw 聊天界面：增加入口

> 具体前端入口位置和交互形式由阿飞设计。

**业务规则：**
- owner 只能查看，不能代替 aiclaw 回复（aiclaw 的回复由 AI 生成）
- 按用户分组展示对话列表
- 显示每个用户的最后一条消息和时间

### 2.5 owner 管理 aiclaw 好友

owner 拥有对 aiclaw 好友的管辖权：

- **查看好友列表**：在 aiclaw 管理页查看所有与 aiclaw 为好友的用户
- **移除好友**：owner 可主动移除 aiclaw 的某个好友（解除好友关系），被移除的用户无法再与 aiclaw 通讯
- **配置关系说明**：owner 可为每个好友设置关系说明文本，注入到安全 prompt 中

### 2.6 owner 配置 aiclaw 对外人设

owner 可在 aiclaw 管理页配置一段"对外人设"文本（系统 prompt），定义 aiclaw 与非 owner 用户聊天时的行为和定位。

**业务规则：**
- 每个 aiclaw 一个对外人设配置
- 可随时修改，修改后对新消息立即生效（已进行中的对话下一条消息开始使用新人设）
- 为空时不注入人设 prompt，只注入安全提示

**示例：**
- "你是一个 Python 编程助手，只回答编程相关问题，拒绝其他话题。"
- "你是一个翻译助手，帮助用户进行中英文翻译。"
- "你是一个友善的聊天伙伴，可以闲聊但不讨论政治和宗教话题。"

### 2.7 aiclaw 好友详情展示

非 owner 用户查看 aiclaw 的好友详情时，显示：

- 名称、头像、简介（owner 设置的）
- AI 标识（与 REQ-002 一致）
- **主人：xxx（昵称）** — 显示 owner 信息
- 在线状态

---

## 三、REQ-002 遗留项

### 3.1 B8：24h 延迟注销定时任务

owner 停用 aiclaw 后（auth_status=2），24 小时内可恢复。超过 24 小时需自动清理。

**业务规则：**
- 定时任务每小时扫描一次 `auth_status=2 且 deactivated_at < now()-24h` 的记录
- 彻底注销：删除 im_user + 删除 im_aiclaw + 删除所有好友关系（含非 owner 好友）+ 清理 Redis 缓存
- 注销后，所有与该 aiclaw 的聊天记录保留（不删消息），但好友关系解除

> 具体定时任务技术方案（XXL-Job）由后端设计确定。

### ~~3.2 B9：消息体 sent_at 时间戳~~ — 已取消

~~消息体中增加发送时间戳，供 plugins 的防抖机制区分实时消息和离线积压消息。~~

**取消原因：** 经后端核实，`im_message.create_time` 已是服务端收到消息的时刻（误差 < 50ms），plugins 防抖直接用 `create_time` 即可，无需新增字段。

---

## 四、数据模型变更

### 4.1 im_aiclaw 表扩展

| 新增字段 | 类型 | 说明 |
|---------|------|------|
| `public_persona` | TEXT | 对外人设（系统 prompt），可为空 |

### 4.2 新建表 im_aiclaw_friend_ext

存储 owner 为每个非 owner 好友配置的关系说明。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | BIGINT | 主键 |
| `aiclaw_uid` | BIGINT | aiclaw 用户 ID |
| `friend_uid` | BIGINT | 非 owner 好友用户 ID |
| `relation_desc` | VARCHAR(200) | 关系说明，为空时使用默认文案 |
| 通用审计字段 | | create_time, update_time, create_by, update_by, is_del, tenant_id |

### 4.3 会话上下文隔离

非 owner 与 aiclaw 的会话隔离由 openclaw 原生 sessionKey 机制实现（gateway 服务端存储），plugins 为每个用户构造不同的 sessionKey。无需新建表或 Redis 存储。

---

## 五、接口变更

### 5.1 新增 HTTP API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/im/aiclaw/{uid}/conversations` | 获取 aiclaw 的所有对话列表（owner 调用） |
| GET | `/api/im/aiclaw/{uid}/conversations/{friendUid}/messages` | 获取 aiclaw 与某用户的聊天记录（owner 调用） |
| GET | `/api/im/aiclaw/{uid}/friends` | 获取 aiclaw 的好友列表（owner 调用） |
| DELETE | `/api/im/aiclaw/{uid}/friends/{friendUid}` | 移除 aiclaw 的某个好友（owner 调用） |
| PUT | `/api/im/aiclaw/{uid}/friends/{friendUid}/relation` | 设置某个好友的关系说明（owner 调用） |
| PUT | `/api/im/aiclaw/{uid}/persona` | 设置 aiclaw 对外人设（owner 调用） |

### 5.2 变更的现有逻辑

| 变更点 | 说明 |
|--------|------|
| 好友申请处理 | server 识别目标为 aiclaw 时，将申请转发给 owner 审批（而非 aiclaw 本身） |
| aiclaw 好友详情 | 返回 owner 信息（uid、昵称）用于前端显示"主人：xxx" |
| WS 消息推送给 aiclaw | 消息 payload 新增 aiclaw 扩展字段，携带 senderName、isOwner、publicPersona、relationDesc，供 plugins 注入安全 prompt |

---

## 六、前端变更

### 6.1 好友申请通知

- owner 收到的好友申请通知区分"加我"和"加我的 AI 助理 [name]"
- 审批界面显示申请者信息 + 目标 aiclaw 名称

### 6.2 aiclaw 管理页扩展

- 增加"对外人设"编辑区域（文本框）
- 增加"对话记录"入口
- 增加"好友管理"入口（查看/移除/配置关系说明）

### 6.3 aiclaw 好友详情

- 非 owner 用户查看 aiclaw 详情时显示"主人：xxx（昵称）"

### 6.4 聊天窗口

- 非 owner 与 aiclaw 聊天时，顶部提示条不变（"AI 助理对话 · 回复由 AI 生成"）
- 流式渲染复用 REQ-002 实现，零改动

> 具体前端交互和入口设计由阿飞确定。

---

## 七、待开发前设计确认项

以下关键设计需要开发人员在任务启动前完成方案设计，经评审后再开始编码：

| # | 设计项 | 负责人 | 状态 | 输出位置 |
|---|--------|--------|------|---------|
| 1 | 好友申请转发机制（server 识别 aiclaw → 转 owner）的具体实现方案 | 锋哥 | **已完成** | `04-需求与设计/设计/REQ-003-非owner通讯后端设计.md` |
| 2 | 安全 prompt 注入模板 + per-user session 的 plugins 侧实现方案 | 锋哥 | **已完成** | 同上 |
| 3 | owner 查看对话记录的前端入口和交互设计 | 阿飞 | **已完成** | `04-需求与设计/设计/REQ-003-非owner通讯前端设计.md` |
| 4 | XXL-Job 延迟注销的定时任务实现方案 | 锋哥 | **已完成** | 同上 |

---

## 八、任务拆解（粗粒度）

### 后端（锋哥）

| # | 任务 | 优先级 |
|---|------|--------|
| B16 | 好友申请转发：aiclaw 好友申请 → owner 审批 | P0 |
| B17 | aiclaw 好友详情返回 owner 信息 | P0 |
| B18 | owner 查看 aiclaw 对话列表 + 聊天记录 API | P0 |
| B19 | owner 管理 aiclaw 好友（列表/移除）API | P1 |
| B20 | aiclaw 对外人设配置（public_persona 字段 + API） | P0 |
| B21 | 好友关系说明（relation_desc 存储 + API） | P1 |
| B22 | plugins 安全 prompt 注入 + per-user session | P0 |
| B23 | XXL-Job 24h 延迟注销定时任务（B8 遗留） | P1 |
| ~~B24~~ | ~~消息体 sent_at 时间戳（B9 遗留）~~ — 已取消（由 im_message.create_time 覆盖） | |

### 前端（阿飞）

| # | 任务 | 优先级 |
|---|------|--------|
| F14 | 好友申请通知区分"加我" vs "加我的 AI 助理" | P0 |
| F15 | aiclaw 好友详情显示"主人：xxx" | P0 |
| F16 | aiclaw 管理页：对外人设编辑 | P0 |
| F17 | aiclaw 管理页：对话记录查看 | P0 |
| F18 | aiclaw 管理页：好友管理（列表/移除/关系说明） | P1 |
| F19 | 搜索结果中 aiclaw 显示 AI 标识 | P1 |
