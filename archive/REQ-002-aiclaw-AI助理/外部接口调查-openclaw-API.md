# OpenClaw 外部接口调查

> 调查日期：2026-03-14
> 文档版本：基于 https://docs.openclaw.ai/ 当日内容
> 源码仓库：https://github.com/openclaw/openclaw

---

## 结论

**推荐集成接口：Chat Completions API**（OpenAI 兼容格式），支持流式 SSE 响应，零侵入，认证简单。

---

## 可用接口一览

| 接口 | 路径 | 方法 | 用途 | 流式 | 认证 |
|------|------|------|------|------|------|
| **Chat Completions API** | `/v1/chat/completions` | POST | 发送对话，获取 AI 回复 | SSE (`stream: true`) | Bearer Token |
| Webhook (wake) | `/hooks/wake` | POST | 触发系统事件 | 无 | Bearer Token |
| Webhook (agent) | `/hooks/agent` | POST | 触发 agent 执行任务并返回结果 | 无 | Bearer Token |
| Gateway WebSocket | WebSocket JSON-RPC | WS | 完整 operator 级控制协议 | 有 | Token + 设备密钥对 |
| CLI `message send` | 命令行 | - | 向指定 channel 发消息 | 无 | 本地配置 |

---

## Chat Completions API（推荐）

### 基本信息

- **端点**：`POST http://<gateway-host>:<port>/v1/chat/completions`
- **默认地址**：`http://127.0.0.1:18789/v1/chat/completions`
- **格式**：OpenAI Chat Completions 兼容
- **认证**：`Authorization: Bearer <token>`（token 在 openclaw gateway 配置中设定）

### 请求参数

**必须 Headers：**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**可选 Headers：**
```
x-openclaw-agent-id: <agentId>    # 指定 agent，默认 "main"
x-openclaw-session-key: <key>     # 会话路由控制
```

**Request Body：**

| 字段 | 类型 | 必须 | 说明 |
|------|------|------|------|
| `model` | string | 否 | agent 标识，如 `"openclaw:main"` 或 `"agent:beta"` |
| `messages` | array | 是 | 消息数组，每项含 `role`（user/assistant/system）和 `content` |
| `stream` | boolean | 否 | `true` 启用 SSE 流式响应 |
| `user` | string | 否 | 用户标识，用于派生稳定的 session key |

### 非流式请求示例

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{"model":"openclaw","messages":[{"role":"user","content":"hi"}]}'
```

### 流式请求示例

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{"model":"openclaw","stream":true,"messages":[{"role":"user","content":"hi"}]}'
```

### 流式响应格式

- Content-Type：`text/event-stream`
- 每行格式：`data: <json>`
- 结束标记：`data: [DONE]`

---

## Webhook API

### /hooks/wake — 触发系统事件

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{"text": "System event description", "mode": "now"}'
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `text` | string（必须） | 事件描述 |
| `mode` | string | `"now"` 立即执行，`"next-heartbeat"` 下次心跳时执行 |

### /hooks/agent — 触发 Agent 执行

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "message": "Run this task",
    "name": "TaskName",
    "agentId": "hooks",
    "model": "openai/gpt-5.2-mini",
    "deliver": true,
    "channel": "telegram",
    "to": "chat_id"
  }'
```

- `deliver: true` 时结果路由到指定 channel
- 支持 `model`、`timeout`、thinking level 等配置

---

## Gateway WebSocket 协议

- **协议**：WebSocket + JSON text frames
- **帧类型**：
  - Request：`{type:"req", id, method, params}`
  - Response：`{type:"res", id, ok, payload|error}`
  - Event：`{type:"event", event, payload, seq?, stateVersion?}`
- **认证流程**：challenge-response + 设备密钥对
- **权限**：operator/node 角色，细粒度 scope 控制
- **适用场景**：完整管控（管理面板、自动化运维），不适合简单的消息收发集成

---

## CLI 命令

### message send — 发送消息

```bash
openclaw message send --channel <provider> --target <destination> \
  --message "text" [--media <file>] [--reply-to <id>]
```

**支持 channel**：whatsapp, telegram, discord, googlechat, slack, mattermost, signal, imessage, msteams

### message read — 读取消息（仅部分 channel）

```bash
openclaw message read --channel discord --target <channel_id> \
  [--limit N] [--before <id>] [--after <id>]
```

---

## 流式响应机制说明

openclaw 内部有两层流式：

1. **Block streaming（channel 层）**：将完成的文本块作为消息发送，不是 token 级别
2. **Preview streaming（Telegram/Discord/Slack）**：生成过程中更新临时预览消息

**Chat Completions API 的流式是独立于上述机制的**，直接返回 token 级别的 SSE 流，这是我们集成使用的方式。

---

## 安全注意事项

- Chat Completions API 的 token 具有 operator 级别权限，**必须在私有网络内使用**
- 不要将 gateway 端口暴露到公网
- aichat-plugins 与 openclaw 部署在同一内网或同一主机
