# aichat-plugins 技术方案

> 状态：已评审
> 日期：2026-03-14
> Owner：backend（锋哥）
> 关联：`07-决策记录/ADR-20260313-openclaw替换为aichat-plugins.md`

---

## 一、定位与目标

aichat-plugins 是 HuLa-Server 与外部 AI 助手（openclaw 等）之间的**通讯桥接服务**。

**核心职责：**
- 接收 HuLa IM 中用户发给 bot 的消息
- 转发到对应的 AI 助手（当前为 openclaw）
- 收集 AI 的流式响应，分批推回 HuLa IM，实现类似 ChatGPT 的逐步渲染体验

**设计原则：**
- 对 AI 助手零侵入 —— 仅通过其公开 API 集成
- 多 claw 适配 —— 统一 adapter 接口，新增 claw 只需实现一个 adapter
- 跨平台 —— 纯 Node.js，支持 Linux/macOS/Windows，`npm install` 即可运行

---

## 二、整体架构

```
┌──────────┐    WebSocket     ┌─────────────┐    RocketMQ     ┌──────────────────┐    HTTP/SSE     ┌──────────┐
│  用户端  │ ←──────────────→ │ HuLa-Server │ ──────────────→ │  aichat-plugins  │ ──────────────→ │ openclaw │
│ (Web/App)│                  │  (WS 网关)   │ ←────────────── │  (Node.js 进程)  │ ←────────────── │ (gateway)│
└──────────┘                  └─────────────┘   HuLa IM API   └──────────────────┘   SSE stream    └──────────┘
```

### 数据流

**上行（用户 → AI）：**
1. 用户通过已有 WebSocket 连接发送消息给 bot 用户（`im_user.user_type = 2`）
2. HuLa-Server 识别目标为 bot，将消息投递到 RocketMQ
3. aichat-plugins 消费 MQ 消息
4. 调用 openclaw Chat Completions API（`POST /v1/chat/completions`，`stream: true`）

**下行（AI → 用户）：**
1. openclaw 以 SSE 流式返回 token
2. aichat-plugins 分批收集（每 3-5 个 token 或每 200-500ms）
3. 每批调用 HuLa-Server IM 内部 API，发送一条"部分回复"消息
4. HuLa-Server 通过用户已有的 WebSocket 连接推送给客户端
5. 客户端逐步拼接渲染，流式完成后标记为完整消息

### 关键决策

| 决策点 | 选型 | 理由 |
|--------|------|------|
| 部署形态 | 独立 Node.js 进程 | 对 claw 零侵入；未来多 claw 用同一架构 |
| 上行通信 | RocketMQ | 解耦，不占 WebSocket 连接资源 |
| 下行通信 | HuLa-Server IM API | 复用用户已有 WebSocket，无需额外连接 |
| 流式方案 | 分批推送部分回复 | 无需 bot 直连，体验接近实时 |
| 与 openclaw 的集成 | Chat Completions API | OpenAI 兼容格式，支持 SSE，零侵入 |
| 技术栈 | TypeScript + Node.js | openclaw 生态亲和，跨平台，npm 分发 |

---

## 三、openclaw 集成接口

> 详细文档见 `10-附录/外部接口调查/openclaw-API调查-20260314.md`

**使用接口：** `POST /v1/chat/completions`（OpenAI Chat Completions 兼容）

```bash
curl -N http://<openclaw-gateway>:18789/v1/chat/completions \
  -H 'Authorization: Bearer <GATEWAY_TOKEN>' \
  -H 'Content-Type: application/json' \
  -d '{"model":"openclaw","stream":true,"messages":[{"role":"user","content":"你好"}]}'
```

- **认证**：`Authorization: Bearer <token>`（openclaw gateway 配置中设定）
- **流式响应**：SSE 格式，`data: <json>` 逐行返回，`data: [DONE]` 结束
- **Agent 选择**：通过 `x-openclaw-agent-id` header 或 `model: "agent:<name>"` 指定
- **安全约束**：token 具有 operator 级别权限，aichat-plugins 与 openclaw 必须在同一内网

---

## 四、Bot 用户模型

复用 HuLa-Server 现有的用户体系：

| 字段 | 值 | 说明 |
|------|------|------|
| `im_user.user_type` | `2` | 机器人类型（1=系统用户，2=机器人，3=普通用户） |

每个 AI 助手在 HuLa IM 中注册为一个 `user_type=2` 的用户，拥有头像、昵称等属性。用户与 bot 的对话走标准 IM 消息流程，HuLa-Server 在消息路由时根据 `user_type` 判断是否转发到 RocketMQ。

---

## 五、项目目录结构

```
aichat-plugins/
├── package.json              # 项目配置，bin 入口
├── tsconfig.json             # TypeScript 配置
├── .env.example              # 环境变量示例
├── src/
│   ├── index.ts              # 入口：启动 MQ consumer + 健康检查 HTTP
│   ├── config.ts             # 统一配置（环境变量 → 类型化配置对象）
│   ├── consumer/
│   │   └── message-consumer.ts   # RocketMQ 消费者：收到消息 → 分发到 adapter
│   ├── adapter/
│   │   ├── base.ts           # Adapter 接口定义（send / stream）
│   │   └── openclaw.ts       # openclaw Chat Completions 实现（SSE 消费）
│   ├── publisher/
│   │   └── hula-client.ts    # HuLa-Server IM API 客户端：推送回复消息
│   └── utils/
│       └── logger.ts         # 日志工具
├── test/
│   └── ...
└── docs/
    └── ...
```

**核心模块职责：**

| 模块 | 职责 | 输入 | 输出 |
|------|------|------|------|
| `consumer/message-consumer.ts` | 消费 RocketMQ 消息，解析用户消息，分发到对应 adapter | MQ 消息 | adapter 调用 |
| `adapter/base.ts` | 定义统一的 claw adapter 接口 | - | 接口类型 |
| `adapter/openclaw.ts` | 调用 openclaw `/v1/chat/completions`，消费 SSE 流 | 用户消息 | 流式 token 回调 |
| `publisher/hula-client.ts` | 调用 HuLa-Server IM API，推送部分回复 / 完成标记 | token 批次 | HTTP 请求 |
| `config.ts` | 环境变量集中管理 | `process.env` | 类型化配置 |

**Adapter 接口设计：**

```typescript
interface ClawAdapter {
  name: string;
  sendMessage(params: {
    messages: Array<{ role: string; content: string }>;
    agentId?: string;
    onChunk: (text: string) => void;   // 每批 token 回调
    onDone: () => void;                // 流式完成回调
    onError: (err: Error) => void;
  }): Promise<void>;
}
```

新增 claw 只需实现此接口。例如未来的 `zeroclaw.ts`、`localclaw.ts` 等。

---

## 六、配置项

| 环境变量 | 说明 | 示例值 |
|----------|------|--------|
| `ROCKETMQ_NAMESRV` | RocketMQ NameServer 地址 | `10.38.10.10:9876` |
| `ROCKETMQ_GROUP` | 消费者组名 | `aichat-plugins-group` |
| `ROCKETMQ_TOPIC` | 订阅的消息 topic | `bot-message` |
| `HULA_SERVER_URL` | HuLa-Server 网关地址（内网） | `http://hula-server-dev:18760` |
| `HULA_SERVER_TOKEN` | 调用 HuLa IM API 的认证 token | - |
| `OPENCLAW_URL` | openclaw gateway 地址 | `http://127.0.0.1:18789` |
| `OPENCLAW_TOKEN` | openclaw gateway 认证 token | - |
| `OPENCLAW_AGENT_ID` | 默认 agent ID | `main` |
| `STREAM_BATCH_MS` | 流式分批间隔（毫秒） | `300` |
| `STREAM_BATCH_CHARS` | 流式分批最小字符数 | `20` |

---

## 七、开发环境

| 项目 | 说明 |
|------|------|
| 容器名 | `aichat-plugins-dev` |
| 基础镜像 | `node:22-bookworm` |
| 运行方式 | `sleep infinity`，容器内手动 `npm run dev` |
| 源码挂载 | `/workspace/aichat-plugins`（宿主机双向同步） |
| `node_modules` | Docker volume 持久化（`plugins-node-modules`） |
| 网络 | `aichat-dev-net`，可直接访问 `hula-server-dev`、`nacos` 等容器 |
| 启动脚本 | `infra/scripts/up-dev.sh`（与 nacos、hula-server-dev 一并启动） |

---

## 八、与 HuLa-Server 的协作约定（待开发）

以下接口需要在 HuLa-Server 侧新增或确认，是联调的前置条件：

| 接口 | 方向 | 说明 | 状态 |
|------|------|------|------|
| Bot 消息投递到 MQ | HuLa-Server → RocketMQ | 用户发给 `user_type=2` 的消息，投递到约定 topic | 待开发 |
| IM 消息推送 API | aichat-plugins → HuLa-Server | 以 bot 身份发送消息给用户（含部分回复/完成标记） | 待确认 |
| Bot 注册/管理 API | 管理端 → HuLa-Server | 创建/配置 bot 用户，绑定 claw adapter 类型和连接参数 | 待设计 |
| Bot 心跳上报 | aichat-plugins → HuLa-Server | 定期上报 bot 在线状态，供前端显示 | 待设计 |

> **MQ Topic 与消息格式**需要与 HuLa-Server 后端协商确定，是接口契约的核心内容。

---

## 九、未来扩展（仅记录方向，不在当前迭代实现）

- **多 claw 支持**：通过 adapter 模式扩展，bot 用户绑定不同的 adapter 类型
- **对话上下文管理**：维护 bot 与用户的对话历史，传入 `messages` 数组
- **openclaw 补丁插件**：当 API 不够用时，开发轻量 openclaw 插件暴露额外能力
- **Bot 管理界面**：在 HuLa-Admin 中管理 bot 用户、查看对话记录、配置参数
