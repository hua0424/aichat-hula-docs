# IM 服务前端接口规范

> 责任人：backend（锋哥）
> 面向：前端开发人员（阿飞）
> 服务路由前缀：`/api/im/**` → luohuo-im-server
> 关联文档：[REQ-001 联调接口契约](REQ-001-联调接口契约.md)

---

## 一、公共说明

### 1.1 协议与基础地址

详见 [REQ-001 联调接口契约](REQ-001-联调接口契约.md) 第一章。本文档所有接口均路由到 **IM 服务**（`/api/im/**`）。

| 环境 | API Base |
|------|---------|
| runtime（前端联调） | `http://localhost:18080/api/im` |
| 容器内 | `http://hula-server-runtime:18760/api/im` |

### 1.2 请求格式

- Content-Type：`application/json`
- GET 请求无 Body，参数通过 URL path 或 query 传递
- 字符编码：UTF-8

### 1.3 鉴权方式

所有需要鉴权的接口，在请求 Header 中携带：

```
Token: <TOKEN_VALUE>
```

Token 通过登录接口（`POST /api/oauth/anyTenant/login`）获取，由 Sa-Token 校验。

### 1.4 统一响应体 R\<T>

所有接口响应统一包装为 `R<T>` 格式：

```json
{
  "code": 200,
  "success": true,
  "msg": "",
  "data": { },
  "path": "/api/im/...",
  "version": "3.0.0",
  "timestamp": 1711234567890
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | Integer | 状态码 |
| `success` | Boolean | 是否成功 |
| `msg` | String | 提示消息，成功时通常为空 |
| `data` | T | 业务数据，各接口不同 |
| `path` | String | 请求路径 |
| `version` | String | 版本号 |
| `timestamp` | Long | 响应时间戳（13 位） |

> 以下各接口的"成功响应"均指 `data` 字段的内容，外层 `R<T>` 包装省略不再重复。

### 1.5 错误码表

| code | 含义 | 说明 |
|------|------|------|
| `200` | 成功 | |
| `-1` | 系统繁忙 | 服务端内部错误 |
| `-9` | 参数验证异常 | 入参校验失败，`msg` 中有具体字段提示 |
| `-10` | 操作异常 | 业务逻辑错误（如权限不足、资源不存在） |
| `406` | Token 已过期 | HTTP 状态码仍为 200，需判断 `code` 字段 |

### 1.6 游标翻页通用说明

部分接口使用游标翻页，入参和出参格式统一：

**入参（Query 参数）：**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `pageSize` | Integer | 否 | 20 | 每页条数，最大 100 |
| `cursor` | String | 否 | null | 游标值，首次不传，翻页时传上次返回的 cursor |

**出参结构：**

```json
{
  "cursor": "1711234567890",
  "isLast": false,
  "list": [ ],
  "total": 100
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `cursor` | String | 下次翻页带上此参数 |
| `isLast` | Boolean | 是否最后一页 |
| `list` | Array | 数据列表 |
| `total` | Long | 总数 |

---

## 二、接口列表

### aiclaw 管理

#### 2.1 获取 aiclaw 列表

| 项目 | 值 |
|------|-----|
| 路径 | `GET /api/im/aiclaw/list` |
| 说明 | 获取当前用户创建的所有 AI 助理列表 |
| 鉴权 | 登录态（Token Header） |

**关键逻辑：**
- 返回当前登录用户（owner）名下所有 aiclaw
- `publicPersona` 字段用于管理详情页回显对外人设文本

**成功响应 `data`：**

```json
[
  {
    "uid": 100001,
    "name": "安洁",
    "avatar": "https://example.com/avatar.jpg",
    "description": "Python 编程助手",
    "authStatus": 1,
    "adapterType": "openclaw",
    "publicPersona": "你是一个 Python 编程助手，只回答编程相关问题。",
    "createTime": "2026-03-20T10:30:00"
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `uid` | Long | AI 助理用户 ID |
| `name` | String | 名称 |
| `avatar` | String | 头像 URL |
| `description` | String | 简介 |
| `authStatus` | Integer | 授权状态：0=未激活 1=已激活 2=已停用 |
| `adapterType` | String | claw 类型（如 openclaw） |
| `publicPersona` | String | 对外人设文本，可为 null |
| `createTime` | String | 创建时间（ISO 8601） |

---

#### 2.2 设置 aiclaw 对外人设

| 项目 | 值 |
|------|-----|
| 路径 | `PUT /api/im/aiclaw/{uid}/persona` |
| 说明 | 设置 aiclaw 与非 owner 用户聊天时的系统 prompt（对外人设） |
| 鉴权 | owner 登录态（Token Header） |

**关键逻辑：**
- 修改后对新消息**立即生效**（已进行中的对话，下一条消息开始使用新人设）
- 传空字符串清除人设，清除后只注入安全提示

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `uid` | Long | aiclaw 的用户 ID |

**请求体：**

```json
{
  "publicPersona": "你是一个 Python 编程助手，只回答编程相关问题。"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `publicPersona` | String | 是 | 对外人设文本，最大 500 字符，空串表示清除 |

**成功响应 `data`：** null

**错误场景：**

| 场景 | code | msg |
|------|------|-----|
| uid 不是当前用户的 aiclaw | -10 | 无权操作该 AI 助理 |
| publicPersona 超长 | -9 | 对外人设不能超过 500 字符 |

---

### aiclaw 对话记录（owner）

#### 2.3 获取 aiclaw 对话列表

| 项目 | 值 |
|------|-----|
| 路径 | `GET /api/im/aiclaw/{uid}/conversations` |
| 说明 | 获取 aiclaw 与所有非 owner 好友的对话列表，按最后消息时间倒序 |
| 鉴权 | owner 登录态（Token Header） |

**关键逻辑：**
- 只返回**有过消息**的好友，无对话的好友不出现
- 按最后消息时间倒序排列
- owner 与 aiclaw 自己的对话不在此列表中

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `uid` | Long | aiclaw 的用户 ID |

**成功响应 `data`：**

```json
{
  "list": [
    {
      "friendUid": 200001,
      "friendName": "小明",
      "friendAvatar": "https://example.com/xiaoming.jpg",
      "lastMessage": {
        "content": "你好，帮我看看这段代码",
        "sendTime": 1711234567890,
        "type": 1
      },
      "roomId": 300001
    },
    {
      "friendUid": 200002,
      "friendName": "小红",
      "friendAvatar": "https://example.com/xiaohong.jpg",
      "lastMessage": {
        "content": "谢谢！",
        "sendTime": 1711148167890,
        "type": 1
      },
      "roomId": 300002
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `list` | Array | 对话列表 |
| `list[].friendUid` | Long | 好友用户 ID |
| `list[].friendName` | String | 好友昵称 |
| `list[].friendAvatar` | String | 好友头像 URL |
| `list[].lastMessage.content` | String | 最后一条消息内容 |
| `list[].lastMessage.sendTime` | Long | 发送时间（13 位时间戳） |
| `list[].lastMessage.type` | Integer | 消息类型（见附录 B） |
| `list[].roomId` | Long | 聊天房间 ID |

**错误场景：**

| 场景 | code | msg |
|------|------|-----|
| uid 不是当前用户的 aiclaw | -10 | 无权操作该 AI 助理 |

---

#### 2.4 获取 aiclaw 与某用户的聊天记录

| 项目 | 值 |
|------|-----|
| 路径 | `GET /api/im/aiclaw/{uid}/conversations/{friendUid}/messages` |
| 说明 | 获取 aiclaw 与指定好友的聊天记录，游标翻页，格式与现有消息接口一致 |
| 鉴权 | owner 登录态（Token Header） |

**关键逻辑：**
- 复用现有消息分页查询，响应格式与 `GET /api/im/chat/msg/page` 完全一致
- **owner 只读查看**，不产生已读标记
- 消息按时间倒序（最新在前），向上滚动加载更早消息

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `uid` | Long | aiclaw 的用户 ID |
| `friendUid` | Long | 好友的用户 ID |

**查询参数：**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `pageSize` | Integer | 否 | 20 | 每页条数 |
| `cursor` | String | 否 | null | 游标，首次不传 |

**请求示例：**

```
GET /api/im/aiclaw/100001/conversations/200001/messages?pageSize=20&cursor=1711234567890
```

**成功响应 `data`：**

```json
{
  "cursor": "1711234500000",
  "isLast": false,
  "list": [
    {
      "fromUser": {
        "uid": 200001
      },
      "message": {
        "id": 50001,
        "roomId": 300001,
        "sendTime": "2026-03-25T14:30:00",
        "type": 1,
        "body": {
          "content": "你好，帮我看看这段代码"
        },
        "messageMarks": {}
      }
    },
    {
      "fromUser": {
        "uid": 100001
      },
      "message": {
        "id": 50002,
        "roomId": 300001,
        "sendTime": "2026-03-25T14:30:05",
        "type": 1,
        "body": {
          "content": "好的，请把代码发给我看看。"
        },
        "messageMarks": {}
      }
    }
  ],
  "total": 128
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `cursor` | String | 下次翻页游标 |
| `isLast` | Boolean | 是否最后一页 |
| `list[].fromUser.uid` | Long | 发送者用户 ID（aiclaw uid 或好友 uid） |
| `list[].message.id` | Long | 消息 ID |
| `list[].message.roomId` | Long | 房间 ID |
| `list[].message.sendTime` | String | 发送时间（ISO 8601） |
| `list[].message.type` | Integer | 消息类型（见附录 B） |
| `list[].message.body` | Object | 消息体，结构随 type 变化。type=1 时为 `{ "content": "文本内容" }` |
| `list[].message.messageMarks` | Object | 消息标记（key 为标记类型，value 为 `{ count, userMarked }`） |
| `total` | Long | 消息总数 |

**错误场景：**

| 场景 | code | msg |
|------|------|-----|
| uid 不是当前用户的 aiclaw | -10 | 无权操作该 AI 助理 |
| friendUid 不是该 aiclaw 的好友 | -10 | 该用户不是 AI 助理的好友 |

---

### aiclaw 好友管理（owner）

#### 2.5 获取 aiclaw 好友列表

| 项目 | 值 |
|------|-----|
| 路径 | `GET /api/im/aiclaw/{uid}/friends` |
| 说明 | 获取 aiclaw 的好友列表（不含 owner 本人），含关系说明 |
| 鉴权 | owner 登录态（Token Header） |

**关键逻辑：**
- 返回 aiclaw 的所有好友，**不含 owner 本人**
- 每项附带 `relationDesc`（来自 im_aiclaw_friend_ext 表），用于好友管理页展示和编辑

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `uid` | Long | aiclaw 的用户 ID |

**成功响应 `data`：**

```json
[
  {
    "uid": 200001,
    "name": "小明",
    "avatar": "https://example.com/xiaoming.jpg",
    "account": "xiaoming",
    "activeStatus": 1,
    "userType": 3,
    "relationDesc": "主人的同事，负责前端开发"
  },
  {
    "uid": 200002,
    "name": "小红",
    "avatar": "https://example.com/xiaohong.jpg",
    "account": "xiaohong",
    "activeStatus": 2,
    "userType": 3,
    "relationDesc": null
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `uid` | Long | 好友用户 ID |
| `name` | String | 好友昵称 |
| `avatar` | String | 好友头像 URL |
| `account` | String | 好友账号 |
| `activeStatus` | Integer | 在线状态：1=在线 2=离线 |
| `userType` | Integer | 用户类型（见附录 A） |
| `relationDesc` | String | 关系说明，null 表示未设置（plugins 使用默认文案） |

**错误场景：**

| 场景 | code | msg |
|------|------|-----|
| uid 不是当前用户的 aiclaw | -10 | 无权操作该 AI 助理 |

---

#### 2.6 移除 aiclaw 好友

| 项目 | 值 |
|------|-----|
| 路径 | `DELETE /api/im/aiclaw/{uid}/friends/{friendUid}` |
| 说明 | 移除 aiclaw 的指定好友，解除好友关系 |
| 鉴权 | owner 登录态（Token Header） |

**关键逻辑：**
- 级联操作：软删除双向好友关系 → 更新聊天房间状态 → 删除 aiclaw_friend_ext 记录 → 通知 plugins 清理该用户的 openclaw session → 清理 Redis 缓存
- **消息记录保留**，不删除历史聊天内容
- 被移除的用户无法再与 aiclaw 通讯

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `uid` | Long | aiclaw 的用户 ID |
| `friendUid` | Long | 要移除的好友用户 ID |

**成功响应 `data`：** null

**错误场景：**

| 场景 | code | msg |
|------|------|-----|
| uid 不是当前用户的 aiclaw | -10 | 无权操作该 AI 助理 |
| friendUid 不是该 aiclaw 的好友 | -10 | 该用户不是 AI 助理的好友 |

---

#### 2.7 设置好友关系说明

| 项目 | 值 |
|------|-----|
| 路径 | `PUT /api/im/aiclaw/{uid}/friends/{friendUid}/relation` |
| 说明 | 为 aiclaw 的某个好友设置关系说明，注入到安全 prompt 中 |
| 鉴权 | owner 登录态（Token Header） |

**关键逻辑：**
- 关系说明会被注入到 aiclaw 与该用户聊天时的安全 prompt 中
- 未设置时 plugins 使用默认文案"`{用户名}是主人的通讯录好友`"
- 传空字符串清除关系说明（恢复使用默认文案）

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `uid` | Long | aiclaw 的用户 ID |
| `friendUid` | Long | 好友的用户 ID |

**请求体：**

```json
{
  "relationDesc": "主人的同事，负责前端开发"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `relationDesc` | String | 是 | 关系说明，最大 200 字符，空串表示清除 |

**成功响应 `data`：** null

**错误场景：**

| 场景 | code | msg |
|------|------|-----|
| uid 不是当前用户的 aiclaw | -10 | 无权操作该 AI 助理 |
| friendUid 不是该 aiclaw 的好友 | -10 | 该用户不是 AI 助理的好友 |
| relationDesc 超长 | -9 | 关系说明不能超过 200 字符 |

---

### 用户信息

#### 2.8 用户详情

| 项目 | 值 |
|------|-----|
| 路径 | `GET /api/im/user/getById/{id}` |
| 说明 | 获取用户详细信息。当目标用户为 aiclaw 时，额外返回 owner 信息 |
| 鉴权 | 登录态（Token Header） |

**关键逻辑：**
- 当 `userType=4`（AI 助理）时，响应中附加 `ownerInfo` 字段
- 普通用户（`userType=3`）不返回 `ownerInfo`

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | Long | 用户 ID |

**成功响应 `data`（aiclaw 用户示例）：**

```json
{
  "uid": 100001,
  "account": "aiclaw_anjie",
  "name": "安洁",
  "avatar": "https://example.com/anjie.jpg",
  "sex": 0,
  "resume": "Python 编程助手",
  "modifyNameChance": 0,
  "wearingItemId": null,
  "itemIds": [],
  "userStateId": null,
  "avatarUpdateTime": null,
  "context": false,
  "num": 0,
  "linkedGitee": false,
  "linkedGithub": false,
  "linkedGitcode": false,
  "userType": 4,
  "ownerInfo": {
    "uid": 10001,
    "name": "华血",
    "avatar": "https://example.com/owner.jpg"
  }
}
```

**新增字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `userType` | Integer | 用户类型（见附录 A）。普通用户为 3，AI 助理为 4 |
| `ownerInfo` | Object | 仅 userType=4 时返回，其他类型不返回此字段 |
| `ownerInfo.uid` | Long | owner 的用户 ID |
| `ownerInfo.name` | String | owner 的昵称 |
| `ownerInfo.avatar` | String | owner 的头像 URL |

---

#### 2.9 用户搜索

| 项目 | 值 |
|------|-----|
| 路径 | `GET /api/im/user/search` |
| 说明 | 搜索用户，结果含 aiclaw 用户。前端根据 userType 展示 AI 标识 |
| 鉴权 | 登录态（Token Header） |

**关键逻辑：**
- 搜索范围包含 aiclaw 用户（userType=4），不做过滤
- 前端根据 `userType=4` 显示 AI 标识徽章

**查询参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | String | 是 | 搜索关键词（匹配名称或账号） |

**成功响应 `data`：**

```json
[
  {
    "uid": 200001,
    "name": "小明",
    "avatar": "https://example.com/xiaoming.jpg",
    "account": "xiaoming",
    "userType": 3
  },
  {
    "uid": 100001,
    "name": "安洁",
    "avatar": "https://example.com/anjie.jpg",
    "account": "aiclaw_anjie",
    "userType": 4
  }
]
```

**新增字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `userType` | Integer | 用户类型（见附录 A）。前端据此判断是否显示 AI 标识 |

---

### 通知

#### 2.10 通知列表

| 项目 | 值 |
|------|-----|
| 路径 | `GET /api/im/room/notice/page` |
| 说明 | 获取好友申请/群申请等通知列表，游标翻页。新增 receiverUserType 用于区分 aiclaw 好友申请 |
| 鉴权 | 登录态（Token Header） |

**关键逻辑：**
- 当 `receiverUserType=4` 时，表示有人想添加当前用户的 AI 助理为好友
- 前端据此显示不同通知文案："xxx 想添加你的 AI 助理 {receiverName} 为好友"
- 审批操作（同意/拒绝）调用方式不变，仍传 `{ applyId, state }`

**查询参数：** 游标翻页参数（见 §1.6）

**成功响应 `data`（游标翻页）：**

```json
{
  "cursor": "1711234567890",
  "isLast": false,
  "list": [
    {
      "id": 5001,
      "eventType": 1,
      "type": 2,
      "applyId": 8001,
      "operateId": null,
      "roomId": null,
      "content": "你好，想添加你的 AI 助理为好友",
      "status": 0,
      "createTime": "2026-03-25T14:30:00",
      "read": 0,
      "senderId": 200001,
      "senderName": "小明",
      "senderAvatar": "https://example.com/xiaoming.jpg",
      "receiverId": 100001,
      "receiverName": "安洁",
      "receiverAvatar": "https://example.com/anjie.jpg",
      "receiverUserType": 4
    }
  ],
  "total": 5
}
```

**新增字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `receiverUserType` | Integer | 接收者（被申请人）的用户类型。为 4 时表示目标是 aiclaw |

**前端文案判断：**

| receiverUserType | 通知文案 |
|------------------|---------|
| 4（aiclaw） | "{senderName} 想添加你的 AI 助理 {receiverName} 为好友" |
| 其他 | "{senderName} 请求添加你为好友"（现有文案） |

---

## 三、附录

### A. userType 枚举值

| 值 | 含义 |
|----|------|
| 1 | 系统用户 |
| 2 | 机器人 |
| 3 | 普通用户 |
| 4 | AI 助理（aiclaw） |

### B. 消息 type 枚举值

| 值 | 含义 |
|----|------|
| 1 | 正常文本 |
| 2 | 撤回消息 |
| 3 | 图片消息 |
| 4 | 文件消息 |
| 5 | 语音消息 |
| 6 | 视频消息 |

---

## 四、变更日志

| 日期 | 关联需求 | 变更内容 |
|------|---------|---------|
| 2026-03-26 | REQ-003 | 初始版本。新增 7 个接口：aiclaw 列表（补 publicPersona）、设置对外人设、对话列表、聊天记录、好友列表、移除好友、设置关系说明。3 个现有接口新增字段：UserInfoResp 补 userType + ownerInfo、UserSearchResp 补 userType、NoticeVO 补 receiverUserType |
