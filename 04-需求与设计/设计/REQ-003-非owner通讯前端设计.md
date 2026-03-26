# REQ-003 非 owner 与 aiclaw 通讯 — 前端设计

> 状态：待评审
> 日期：2026-03-26
> 编写：阿飞（Frontend）
> 关联需求：`04-需求与设计/需求/REQ-003-非owner与aiclaw通讯.md`
> 关联后端设计：`04-需求与设计/设计/REQ-003-非owner通讯后端设计.md`

---

## 一、改动总览

| # | 功能 | 改动范围 | 优先级 |
|---|------|---------|--------|
| F14 | 好友申请通知区分"加我" vs "加我的 AI 助理" | MobileApplyList + 桌面端通知组件 | P0 |
| F15 | aiclaw 好友详情显示"主人：xxx" | PersonalInfo | P0 |
| F16 | aiclaw 管理详情页（对外人设编辑） | 新建路由页 + API | P0 |
| F17 | aiclaw 对话记录查看（独立只读页面） | 新建两级页面 | P0 |
| F18 | aiclaw 好友管理（列表/移除/关系说明） | 新建组件 + API | P1 |
| F19 | 搜索添加 aiclaw（搜索结果 AI 标识） | AddFriends 搜索结果适配 | P1 |

---

## 二、F14 好友申请通知区分

### 2.1 数据依赖

后端在 `NoticeVO`（前端类型 `NoticeItem`）中新增 `receiverUserType` 字段。

前端类型变更（`services/types.ts`）：

```typescript
export interface NoticeItem {
  // ... 现有字段 ...
  /** 接收者用户类型（4=aiclaw），用于区分通知文案 */
  receiverUserType?: number
}
```

### 2.2 展示逻辑

**移动端 `MobileApplyList.vue`：**

当 `eventType === NoticeType.ADD_ME`（收到好友申请）时，判断 `receiverUserType`：

```
if (item.receiverUserType === 4) {
  // 文案："xxx 想添加你的 AI 助理 {receiverName} 为好友"
  // 显示 aiclaw 的头像 + 名称
} else {
  // 现有文案："xxx 请求添加你为好友"
}
```

审批操作（同意/拒绝）不变，仍然传 `{ applyId, state }`，后端内部根据 target userType 分支处理。

**桌面端通知组件：** 同移动端逻辑。

### 2.3 i18n 文案

```json
{
  "aiclaw": {
    "friendApply": {
      "title": "{name} 想添加你的 AI 助理 {aiclawName} 为好友",
      "approve": "同意",
      "reject": "拒绝"
    }
  }
}
```

---

## 三、F15 aiclaw 好友详情显示"主人"

### 3.1 数据依赖

后端 `UserInfoResp` 新增：
- `userType: number`
- `ownerInfo?: { uid: string; name: string; avatar: string }`（仅 userType=4 时返回）

前端 `MsgUserType`（或 `UserInfoType`）需要相应扩展。

### 3.2 展示位置

**`PersonalInfo.vue`** 的用户信息区（账号/ID 展示行附近），新增条件渲染：

```vue
<!-- aiclaw 主人信息 -->
<div v-if="userDetailInfo?.userType === 4 && userDetailInfo?.ownerInfo" class="flex items-center gap-4px">
  <span class="text-12px text-#999">主人：</span>
  <n-avatar round :size="16" :src="userDetailInfo.ownerInfo.avatar" />
  <span class="text-12px">{{ userDetailInfo.ownerInfo.name }}</span>
</div>
```

### 3.3 条件

- 仅非 owner 用户查看 aiclaw 详情时显示。owner 自己查看不需要显示"主人"（因为就是自己）
- 判断是否为 owner：`userDetailInfo?.ownerInfo?.uid === userStore.userInfo?.uid`，如果是 owner 则不显示此行

---

## 四、F16 aiclaw 管理详情页

### 4.1 路由设计

| 端 | 路由 | 组件 |
|----|------|------|
| 移动端 | `/mobile/mobileMy/aiAssistant/:uid` | `mobile/views/my/AiAssistantDetail.vue`（新建） |
| 桌面端 | 管理页右侧面板（选中列表项时显示） | `views/aiAssistantWindow/AiAssistantDetail.vue`（新建） |

移动端从管理列表点击 aiclaw 项进入详情页，HeaderBar 显示 aiclaw 名称 + 返回按钮。

### 4.2 页面结构

详情页分为三个区块（移动端纵向排列，桌面端可用 tab）：

```
┌─────────────────────────────────┐
│ ← 安洁（AI 助理名称）            │  HeaderBar
├─────────────────────────────────┤
│ 头像 + 名称 + 状态 + 基本信息    │  顶部概要
├─────────────────────────────────┤
│ 📝 对外人设                      │
│ ┌─────────────────────────────┐ │
│ │ textarea（可编辑）           │ │
│ │ placeholder: "定义 AI 助理  │ │
│ │ 与其他用户聊天时的行为..."   │ │
│ └─────────────────────────────┘ │
│               [保存]             │
├─────────────────────────────────┤
│ 👥 好友管理                    → │  点击进入 F18 好友列表页
├─────────────────────────────────┤
│ 💬 对话记录                    → │  点击进入 F17 对话列表页
└─────────────────────────────────┘
```

### 4.3 对外人设编辑

- textarea，最大长度 500 字符
- 下方"保存"按钮，调用 `PUT /api/im/aiclaw/{uid}/persona`
- body: `{ publicPersona: "..." }`
- 空内容表示清除人设
- 保存成功后 toast 提示
- 进入页面时通过 aiclaw 详情接口获取当前 `publicPersona` 值

### 4.4 API 枚举新增

```typescript
// enums/index.ts
AICLAW_SET_PERSONA = 'aiclawSetPersona',

// webImRequest.ts
aiclawSetPersona: { method: 'PUT', path: 'im/aiclaw/{uid}/persona' },
```

---

## 五、F17 aiclaw 对话记录查看

### 5.1 路由设计

| 端 | 路由 | 组件 |
|----|------|------|
| 移动端 | `/mobile/mobileMy/aiAssistant/:uid/conversations` | `AiclawConversations.vue`（新建） |
| 移动端 | `/mobile/mobileMy/aiAssistant/:uid/conversations/:friendUid` | `AiclawConversationDetail.vue`（新建） |

### 5.2 第一级：对话列表

展示所有与该 aiclaw 有过对话的用户，按最后消息时间倒序。

```
┌─────────────────────────────────┐
│ ← 对话记录                       │  HeaderBar
├─────────────────────────────────┤
│ 🟢 小明                         │
│    最近：你好，帮我看看这段代码   │
│                     2026-03-25  │
├─────────────────────────────────┤
│ ⚪ 小红                         │
│    最近：谢谢！                  │
│                     2026-03-24  │
├─────────────────────────────────┤
│         暂无对话记录              │  空状态
└─────────────────────────────────┘
```

每项显示：
- 用户头像 + 名称
- 最后一条消息内容（截断）+ 时间
- 点击进入第二级

**API：** `GET /api/im/aiclaw/{uid}/conversations`

### 5.3 第二级：聊天记录详情（只读）

展示 aiclaw 与该用户的完整聊天记录，复用现有消息气泡组件但**去掉输入框和操作菜单**。

```
┌─────────────────────────────────┐
│ ← 小明 的对话                    │  HeaderBar
├─────────────────────────────────┤
│    ┌──────────┐                 │
│    │ 你好     │          小明   │  用户消息（右侧）
│    └──────────┘                 │
│                                 │
│  安洁  ┌──────────────────┐     │
│        │ 你好！有什么可以  │     │  aiclaw 消息（左侧）
│        │ 帮助你的？       │     │
│        └──────────────────┘     │
│                                 │
│         ── 没有更多了 ──         │
└─────────────────────────────────┘
```

**关键设计决策：**
- **只读**：不显示输入框、不显示消息操作菜单（回复/转发/撤回等）
- **消息气泡**：复用 `renderMessage` 组件，但传入 `readOnly` 模式
- **分页加载**：向上滚动加载更多历史消息
- **消息方向**：aiclaw 的消息显示在左侧，用户消息显示在右侧（从 owner 视角看对话）

**API：** `GET /api/im/aiclaw/{uid}/conversations/{friendUid}/messages`
- 分页参数：`pageSize`、`cursor`
- 响应格式与 `POST /api/im/chat/msg/list` 一致

### 5.4 API 枚举新增

```typescript
// enums/index.ts
AICLAW_CONVERSATIONS = 'aiclawConversations',
AICLAW_CONVERSATION_MESSAGES = 'aiclawConversationMessages',

// webImRequest.ts
aiclawConversations: { method: 'GET', path: 'im/aiclaw/{uid}/conversations' },
aiclawConversationMessages: { method: 'GET', path: 'im/aiclaw/{uid}/conversations/{friendUid}/messages' },
```

---

## 六、F18 aiclaw 好友管理

### 6.1 路由设计

| 端 | 路由 | 组件 |
|----|------|------|
| 移动端 | `/mobile/mobileMy/aiAssistant/:uid/friends` | `AiclawFriends.vue`（新建） |

### 6.2 页面结构

```
┌─────────────────────────────────┐
│ ← 好友管理                       │  HeaderBar
├─────────────────────────────────┤
│ 小明                             │
│ 关系：主人的同事，负责前端开发     │  灰字，未设置时不显示
│              [编辑关系] [移除]    │
├─────────────────────────────────┤
│ 小红                             │
│ 关系：未设置                      │
│              [编辑关系] [移除]    │
└─────────────────────────────────┘
```

### 6.3 交互

**编辑关系说明：**
- 点击"编辑关系"→ 弹出 Dialog，包含一个输入框（`n-input`，maxlength=200）
- placeholder："例如：xxx 是主人的同事，负责前端开发"
- 确认后调用 `PUT /api/im/aiclaw/{uid}/friends/{friendUid}/relation`
- body: `{ relationDesc: "..." }`

**移除好友：**
- 点击"移除"→ 弹出确认 Dialog："移除后该用户将无法再与 AI 助理通讯，确定吗？"
- 确认后调用 `DELETE /api/im/aiclaw/{uid}/friends/{friendUid}`
- 成功后从列表中移除该项

### 6.4 API 枚举新增

```typescript
// enums/index.ts
AICLAW_FRIENDS = 'aiclawFriends',
AICLAW_REMOVE_FRIEND = 'aiclawRemoveFriend',
AICLAW_SET_RELATION = 'aiclawSetRelation',

// webImRequest.ts
aiclawFriends: { method: 'GET', path: 'im/aiclaw/{uid}/friends' },
aiclawRemoveFriend: { method: 'DELETE', path: 'im/aiclaw/{uid}/friends/{friendUid}' },
aiclawSetRelation: { method: 'PUT', path: 'im/aiclaw/{uid}/friends/{friendUid}/relation' },
```

---

## 七、F19 搜索添加 aiclaw

### 7.1 改动范围

**首期只做搜索添加**，不做分享链接/二维码/卡片。

**`AddFriends.vue`（移动端搜索页面）：**

搜索结果中，当 `userType === 4` 时：
- 名称旁显示 AI 标识徽章（复用 `aiclaw.badge` 即 "AI"）
- 底部显示"AI 助理"标签（区别于普通用户）

```vue
<template v-if="item.userType === 4">
  <span class="text-#7c5cfc text-11px font-500 px-4px py-1px rounded-4px bg-#7c5cfc15">AI</span>
</template>
```

### 7.2 数据依赖

后端 `UserSearchResp` 新增 `userType` 字段。前端搜索结果类型需要扩展：

```typescript
// 搜索结果项新增 userType
type SearchUserItem = {
  uid: string
  name: string
  avatar: string
  account: string
  userType?: number  // 新增
}
```

### 7.3 添加好友流程

用户点击 aiclaw 搜索结果 → 进入 `ConfirmAddFriend.vue` → 填写验证消息 → 发送好友申请。

流程与添加普通用户完全一致，前端无需特殊处理。后端会将申请转发给 owner 审批。

---

## 八、新建文件清单

| 文件 | 用途 |
|------|------|
| `src/mobile/views/my/AiAssistantDetail.vue` | 移动端 aiclaw 管理详情页（人设+入口） |
| `src/mobile/views/my/AiclawConversations.vue` | 移动端 aiclaw 对话列表页 |
| `src/mobile/views/my/AiclawConversationDetail.vue` | 移动端 aiclaw 对话记录详情页（只读） |
| `src/mobile/views/my/AiclawFriends.vue` | 移动端 aiclaw 好友管理页 |
| `src/views/aiAssistantWindow/AiAssistantDetail.vue` | 桌面端 aiclaw 管理详情面板 |

---

## 九、改动文件清单

| 文件 | 改动说明 |
|------|---------|
| `src/services/types.ts` | NoticeItem 新增 `receiverUserType`；搜索结果类型新增 `userType` |
| `src/mobile/components/my/MobileApplyList.vue` | 好友申请通知文案区分 aiclaw |
| `src/mobile/components/my/PersonalInfo.vue` | aiclaw 详情显示"主人：xxx" |
| `src/mobile/views/friends/AddFriends.vue` | 搜索结果 AI 标识 |
| `src/mobile/views/my/AiAssistant.vue` | 列表项点击跳转到详情页 |
| `src/router/index.ts` | 新增 4 条移动端路由 |
| `src/enums/index.ts` | 新增 API 枚举 |
| `src/utils/webImRequest.ts` | 新增 API 映射 |
| `locales/zh-CN/aiclaw.json` | 新增 i18n 文案 |
| `locales/en/aiclaw.json` | 新增 i18n 文案 |

---

## 十、实现顺序建议

```
Phase 1（P0，依赖后端 B16/B17/B20 就绪）：
  F15（好友详情显示主人）→ F14（好友申请通知）→ F16（管理详情页+人设编辑）

Phase 2（P0，依赖后端 B18 就绪）：
  F17（对话记录查看）

Phase 3（P1，依赖后端 B19/B21 就绪）：
  F18（好友管理）→ F19（搜索添加 aiclaw）
```

---

## 十一、待确认项

| # | 事项 | 状态 | 确认日期 |
|---|------|------|---------|
| 1 | 后端 NoticeVO 新增 `receiverUserType` | 已确认，锋哥补充 | 2026-03-26 |
| 2 | 后端 UserInfoResp 新增 `userType` + `ownerInfo` | 已确认，锋哥补充 | 2026-03-26 |
| 3 | 后端 UserSearchResp 新增 `userType` | 已确认，锋哥补充 | 2026-03-26 |
| 4 | `GET /api/im/aiclaw/{uid}/conversations` 响应格式 | 已对齐（见后端设计文档 §2.2） | 2026-03-26 |
| 5 | `publicPersona` 回显方式 | 已确认：直接在 `GET /api/im/aiclaw/list` 返回项中附带 `publicPersona` 字段 | 2026-03-26 |
| 6 | 对话消息分页格式 | 已确认：与 `POST /api/im/chat/msg/list` 一致，复用 `CursorPageBaseResp`（cursor/isLast/list/total） | 2026-03-26 |
| 7 | approve() 前端调用方式 | 已确认：不变，传 `{ applyId, state }`，后端内部判断 aiclaw 场景 | 2026-03-26 |
