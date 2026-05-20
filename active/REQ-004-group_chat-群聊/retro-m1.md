# REQ-004 M1 里程碑回顾

> 主持：manager
> 日期：2026-05-20
> 里程碑：M1 协议层 + 基础设施
> 预计工作量：2.5 人天（并行约 1.5 自然日）
> 实际历时：**约 30 分钟**（同日内完成）

---

## 一、完成情况

### 1.1 各方提交

| Owner | 任务 | Commit | 状态 |
|-------|------|--------|------|
| server-dev | S-M1-1 / S-M1-2 / S-M1-3 | HuLa-Server `8ffd0bf3` | ✅ |
| plugin-dev | P-M1-1 / P-M1-2 / P-M1-3 | aichat-plugins `ccd61e4` | ✅ |
| frontend-dev | F-M1-1 / F-M1-2 | HuLa `21d2d1f06` | ✅（待对齐 1 处常量名） |

### 1.2 产出明细

**server-dev**：
- 3 张表 DDL 落到 `docs/sql/req-004-group-chat.sql`
- 3 套 Entity + Mapper（统一 BIGINT 主键 + 雪花 ID）
- WSReqTypeEnum / WSRespTypeEnum 扩展 + WSGroupConfigChange 嵌套 ConfigDTO + DTO ID 用 String

**plugin-dev**：
- `protocol.ts` 新增 WSReqType 20/21/22 + 4 个 WSRespType 字符串 + 7 个 payload 类型
- `ClawAdapter` 接口扩展 `ThinkingCallbacks`（保留 `StreamCallbacks` 兼容）
- `OpenclawAdapter.PendingChat` 补 `startTime` 字段
- `pnpm build` + `pnpm lint` 通过

**frontend-dev**：
- `wsType.ts` 新增 4 个 WsResponseMessageType + 5 个 Payload 类型 + AiclawGroupConfig
- `MsgType` 扩展可选 `extra` 字段
- TypeScript 类型检查通过

---

## 二、跨组对齐核查

### 2.1 WS 协议常量值核查（三方对照表）

| 协议项 | server | plugin | frontend | 一致性 |
|--------|--------|--------|----------|--------|
| THINKING_START | 20 | 20 | — | ✅ |
| THINKING_DELTA | 21 | 21 | — | ✅ |
| THINKING_END | 22 | 22 | — | ✅ |
| Resp: thinking start | `"thinkingStart"` | `"thinkingStart"` | `"thinkingStart"` | ✅ |
| Resp: thinking delta | `"thinkingDelta"` | `"thinkingDelta"` | `"thinkingDelta"` | ✅ |
| Resp: thinking end | `"thinkingEnd"` | `"thinkingEnd"` | `"thinkingEnd"` | ✅ |
| Resp: 群配置变更 | `"groupConfigChange"` | `"groupConfigChange"`（已自改） | `"aiclawGroupConfigUpdate"` | ⚠️ **需 frontend-dev 改** |

### 2.2 ID 类型核查

- server 端 thinkingId / msgId 等 ID 字段在 WS DTO 中用 **String** 序列化（防 JS 精度丢失）✅
- plugin-dev 端 ThinkingSession.thinkingId 用 string ✅
- frontend-dev 端 ThinkingStartPayload.thinkingId 用 string ✅

### 2.3 嵌套结构核查（M1 修订项）

- server `WSGroupConfigChange` 包含 `config: ConfigDTO` 嵌套 ✅
- plugin `GroupConfigUpdateDTO`（v1.2 设计）含 `config` 嵌套 ✅
- frontend `AiclawGroupConfigUpdatePayload` 含 `config` 嵌套 ✅

---

## 三、发现的问题

### 3.1 P-M1 自测发现：群配置 WS 类型字符串名跨组不一致

**问题**：v1.2 设计评审时 reviewer 抓了 payload 嵌套/扁平（M1），但**字符串常量值**没列入检查项，三方在编码阶段才发现：
- server: `groupConfigChange`
- plugin (v1.2 设计): `groupConfigUpdate` → 编码时自改为 `groupConfigChange`
- frontend (v1.2 设计 + M1 实现): `aiclawGroupConfigUpdate`

**处理**：
- 决策统一为 **`groupConfigChange`**（server 权威，字符串简洁）
- plugin-dev 已自查并修改
- frontend-dev 待修改（plugin-dev 直接联系处理）
- 不需要 manager 中转 — 属于跨组 daily 范畴

**根因**：评审阶段只看了结构未看常量值。

### 3.2 经验沉淀

reviewer 已记录到后续 review 检查清单（feedback 已确认）：
- WS 事件类型字符串（Req/Resp 双方向）
- REST API 路径
- Redis key 模式
- Token / extra 字段名
- DTO 字段名 camelCase vs snake_case

下次设计 confirm review 时，常量值一致性会作为强制项。

---

## 四、设计与实现差异

**无重大差异**。三方实现严格遵循 design-server v1.2 / design-plugin v1.2 / design-frontend v1.2 + M1 修订项。

仅 1 处编码细节调整（plugin-dev）：
- 设计文档原写 `groupConfigUpdate`，编码时为对齐 server 改为 `groupConfigChange`，并同步回写设计文档（待 plugin-dev push 文档调整）

---

## 五、提测准入

### 5.1 准入清单

- [x] 三方所有 M1 任务完成
- [x] 三方各自构建/类型检查通过
- [x] WS 协议常量值核查表完成（待 frontend-dev 改 1 处后全绿）
- [x] 跨组对齐项无歧义
- [x] 提交规范符合（commit 信息含 `REQ-004(M1)`）

### 5.2 提测项

| 测试方 | 内容 |
|--------|------|
| backend-tester | DDL 在 dev MySQL 执行验证 + Entity 单元测试 + WS 枚举值校验（人工触发 dummy 事件） |
| ui-tester | 本里程碑无 UI 改动，跳过 |

---

## 六、决议

✅ **M1 通过，但需先解决一处常量字符串对齐（frontend-dev `aiclawGroupConfigUpdate` → `groupConfigChange`），通过后启动 M2。**

待办：
1. frontend-dev 修改 `wsType.ts` 中 `aiclawGroupConfigUpdate` → `groupConfigChange`
2. 通知 backend-tester 启动 M1 提测
3. 提测通过后 → 启动 M2 里程碑

---

## 七、M2 启动预告

**M2 范围**：Agent Loop + Thinking 落库 + 展示（~5d）
- server: ThinkingService + 推送路由 + invite API 改造
- plugin: MessageHandler 重构 + Adapter 映射 + hula_send_message 扩展 + userType=4 修正
- frontend: chat.ts thinkingStreams + Mitt 监听 + ThinkingCard + ThinkingPanel

**M2 启动前提**：
- frontend-dev 完成常量字符串修复
- backend-tester M1 提测通过
- 三方确认无其他遗漏

预计 M2 启动时间：常量修复 + 提测完成后立即启动。
