# aichat-claw 插件架构设计

> 状态：待评审
> 日期：2026-03-21
> 编写：锋哥（Backend）
> 关联需求：`04-需求与设计/需求/REQ-002-aiclaw-AI助理功能需求设计.md`
> 关联决策：`07-决策记录/ADR-001-aiclaw架构决策.md`

---

## 一、背景与动机

当前 aichat-node（原 aichat-plugins）通过 openclaw 的 HTTP Chat Completions API（`/v1/chat/completions`）与 AI 通信。此方案存在以下问题：

| # | 问题 | 影响 |
|---|------|------|
| 1 | **无上下文** | 每次 HTTP 请求无状态，openclaw 的会话上下文、记忆、compaction 机制无法生效 |
| 2 | **无法调用 openclaw 能力** | Skills、Agent Tools、工作流等 openclaw 核心能力被 HTTP API 屏蔽 |
| 3 | **无法注入 HuLa 能力** | 未来需要让 AI 主动查找好友、发消息、发朋友圈，HTTP 模式无法实现 |
| 4 | **单 claw 绑定** | 架构只考虑了 openclaw，未来 zeroclaw/kimiclaw 等接入缺乏统一抽象 |

---

## 二、架构方案：aichat-node + aichat-claw 双层架构

### 2.1 架构全景

```
┌──────────┐       ┌───────────────┐       ┌──────────────────────────────────┐
│ HuLa-    │       │  aichat-node  │       │         openclaw 进程             │
│ Server   │←WS──→│  (统一桥接)    │←WS RPC→│  ┌────────────────────────────┐  │
│          │       │               │        │  │ aichat-claw (Plugin)       │  │
│          │       │ · 多claw路由   │        │  │ · Channel: 消息收发        │  │
│          │       │ · 流式缓冲    │        │  │ · Tools: HuLa能力注入      │  │
│          │       │ · ACK/去重    │        │  │ · 上下文/记忆/skills自动   │  │
│          │       │ · 统一协议    │        │  └────────────────────────────┘  │
└──────────┘       │               │        └──────────────────────────────────┘
                   │               │
                   │               │←(未来)→ zeroclaw gateway
                   │               │←(未来)→ kimiclaw gateway
                   └───────────────┘
```

### 2.2 各层职责

| 层 | 项目 | 职责 | 不做什么 |
|---|------|------|---------|
| **HuLa-Server** | Java 微服务 | IM 消息存储、WS 推送、用户管理、流式中继（StreamProcessor） | 不感知 claw 类型 |
| **aichat-node** | Node.js 进程 | 统一 WS 协议与 HuLa-Server 通信；根据 `adapterType` 路由到对应 claw；流式消息缓冲转发；ACK/去重/防抖 | 不处理 AI 逻辑，不管上下文 |
| **aichat-claw** | openclaw Plugin | 注册 Channel 接收消息；注册 Agent Tools 暴露 HuLa 能力；消息走 openclaw 完整 agent pipeline | 不直接连 HuLa-Server |

### 2.3 保留 aichat-node 中间层的理由

| 理由 | 说明 |
|------|------|
| **多 claw 路由** | 同一台机器可能同时运行 openclaw + zeroclaw，node 根据 aiclaw 的 `adapterType` 路由 |
| **统一协议** | 不同 claw 的通信协议不同（openclaw 用 WS RPC，zeroclaw 可能用 gRPC），node 统一为 HuLa WS 协议 |
| **流式缓冲** | 统一处理流式消息的缓冲、分批、超时，减少对 HuLa-Server 的影响 |
| **连接管理** | 统一处理与 HuLa-Server 的 WS 连接、心跳、重连、ACK |

---

## 三、通信协议

### 3.1 aichat-node ↔ HuLa-Server（不变）

复用现有 WS 协议：
- 连接：`ws://host/api/ws/ws?token=<连接token>&clientId=<机器码>`
- 心跳：`{type: 2, data: ""}`
- ACK：`{type: 15, data: {msgId, timestamp}}`
- 流式：`STREAM_START(17)` / `STREAM_DELTA(18)` / `STREAM_END(19)`

### 3.2 aichat-node ↔ openclaw gateway（WS RPC）

使用 openclaw 原生 WebSocket RPC 协议（`ws://localhost:18789`）：

**发送消息（请求）：**
```json
{
  "type": "req",
  "id": "msg-001",
  "method": "agent",
  "params": {
    "message": "你好",
    "sessionKey": "aiclaw-140789091499520"
  }
}
```

**流式响应（事件流）：**
```json
{ "type": "event", "event": "agent", "payload": { "content": "嗨", "done": false } }
{ "type": "event", "event": "agent", "payload": { "content": "，老大", "done": false } }
{ "type": "event", "event": "agent", "payload": { "done": true, "fullContent": "嗨，老大" } }
```

**认证：**
- 连接时使用 `gateway.auth.token`（从 `~/.openclaw/openclaw.json` 自动读取）

### 3.3 aichat-claw ↔ openclaw 内部

aichat-claw 作为 openclaw 的原生 Plugin，通过内部 API 通信：
- 消息路由：Channel → Gateway → Agent pipeline（内部事件驱动）
- Tool 执行：Agent 调用 → Plugin 注册的 Tool → 执行 → 返回结果
- 运行时依赖：`api.runtime.sessions`、`api.runtime.channels`、`api.runtime.tools`

---

## 四、消息流详解

### 4.1 用户发消息给 aiclaw

```
用户客户端
    │  WS: 发送消息
    ▼
HuLa-Server
    │  存库 → MQ → PushService
    ▼
aichat-node (WS 收到 receiveMessage)
    │  ACK 回执 → 去重 → 防抖
    │  根据 adapterType 路由
    ▼
openclaw gateway (WS RPC: method="agent")
    │  路由到 agent pipeline
    ▼
openclaw agent
    │  上下文组装（历史消息 + 记忆 + compaction）
    │  调用 LLM (kimi/gpt/...)
    │  可能调用 Tools（hula_find_friend 等）
    ▼
流式响应 (event stream)
    │
    ▼
aichat-node (收到流式 events)
    │  转为 STREAM_START/DELTA/END
    │  发送给 HuLa-Server WS
    ▼
HuLa-Server (StreamProcessor)
    │  中继给用户 WS + stream_end 落库
    ▼
用户客户端（逐字渲染）
```

### 4.2 AI 主动操作（Agent Tools）

```
用户说："帮我给张三发个消息说明天开会"
    │
    ▼
openclaw agent 分析意图
    │  决定调用 hula_find_friend(keyword="张三")
    ▼
aichat-claw Plugin Tool 执行
    │  调用 HuLa-Server REST API: GET /api/im/friend/search?key=张三
    │  返回: [{uid: 123, name: "张三"}]
    ▼
agent 继续决策
    │  调用 hula_send_message(friendUid=123, content="明天开会")
    ▼
aichat-claw Plugin Tool 执行
    │  调用 HuLa-Server REST API: POST /api/im/chat/msg
    │  返回: {success: true}
    ▼
agent 回复用户
    │  "已经给张三发送了消息：明天开会"
    ▼
（流式响应 → aichat-node → HuLa-Server → 用户）
```

---

## 五、aichat-claw Plugin 设计

### 5.1 插件清单

```json
// aichat-claw/openclaw.plugin.json
{
  "id": "aichat-claw",
  "name": "AiChat HuLa Integration",
  "version": "0.1.0",
  "description": "连接 openclaw 与 HuLa IM 系统，注入 HuLa 通讯能力",
  "configSchema": {
    "type": "object",
    "properties": {
      "hulaServerUrl": {
        "type": "string",
        "description": "HuLa-Server API 地址"
      },
      "aiclawToken": {
        "type": "string",
        "description": "aiclaw 连接凭证（用于调用 HuLa-Server API）"
      }
    }
  }
}
```

### 5.2 注册入口

```typescript
// aichat-claw/src/index.ts
import { registerChannel } from './channel/index.js';
import { registerTools } from './tools/index.js';

export default function register(api: OpenClawPluginApi) {
  const config = api.config;
  const logger = api.logger;

  logger.info('aichat-claw plugin loading...');

  // 注册 HuLa Channel（消息收发）
  registerChannel(api, config);

  // 注册 Agent Tools（HuLa 能力注入）
  registerTools(api, config);

  logger.info('aichat-claw plugin loaded');
}
```

### 5.3 Agent Tools 清单

| Tool | 描述 | 参数 |
|------|------|------|
| `hula_send_message` | 主动给 HuLa 好友发消息 | `friendUid`, `content` |
| `hula_find_friend` | 搜索 HuLa 好友 | `keyword` |
| `hula_add_friend` | 发送好友申请 | `targetUid`, `message` |
| `hula_list_friends` | 获取好友列表 | 无 |
| `hula_post_feed` | 发布朋友圈动态 | `content`, `images?` |
| `hula_get_chat_history` | 获取与某好友的聊天记录 | `friendUid`, `limit?` |

> 首期实现 `hula_send_message` 和 `hula_find_friend`，其余后续迭代。

---

## 六、aichat-node 重构

### 6.1 统一 claw 适配器接口

```typescript
// aichat-node/src/claw/interface.ts
export interface ClawAdapter {
  /** claw 类型标识 */
  readonly type: string;  // 'openclaw' | 'zeroclaw' | ...

  /** 连接到 claw gateway */
  connect(): Promise<void>;

  /** 断开连接 */
  disconnect(): Promise<void>;

  /** 发送消息，通过回调接收流式响应 */
  chat(
    message: string,
    sessionId: string,
    callbacks: StreamCallbacks,
  ): Promise<void>;

  /** 连接状态 */
  get isConnected(): boolean;
}

export interface StreamCallbacks {
  onChunk: (chunk: string) => void;
  onDone: (fullContent: string) => void;
  onError: (error: Error) => void;
}
```

### 6.2 openclaw 适配器（WS RPC）

```typescript
// aichat-node/src/claw/openclaw.ts
export class OpenclawAdapter implements ClawAdapter {
  readonly type = 'openclaw';
  private ws: WebSocket | null = null;
  private gatewayUrl: string;
  private gatewayToken: string;

  async connect() {
    // 连接 openclaw gateway WS RPC
    this.ws = new WebSocket(this.gatewayUrl);
    // 认证握手
  }

  async chat(message, sessionId, callbacks) {
    // 发送 RPC 请求: method="agent"
    // 监听 event stream，逐个 chunk 回调
  }
}
```

### 6.3 多 claw 路由器

```typescript
// aichat-node/src/router.ts
export class ClawRouter {
  private adapters: Map<string, ClawAdapter> = new Map();

  register(adapter: ClawAdapter) {
    this.adapters.set(adapter.type, adapter);
  }

  /** 根据 adapterType 路由到对应 claw */
  getAdapter(adapterType: string): ClawAdapter {
    const adapter = this.adapters.get(adapterType);
    if (!adapter) throw new Error(`Unknown claw type: ${adapterType}`);
    return adapter;
  }
}
```

### 6.4 与现有代码的关系

| 现有文件 | 变化 |
|---------|------|
| `ws/client.ts` | 保留，重命名为 `server/hula-ws.ts` |
| `ws/protocol.ts` | 保留，移入 `stream/protocol.ts` |
| `handler/message.ts` | 保留，ACK/去重/防抖逻辑不变 |
| `utils/debounce.ts` | 保留 |
| `auth/machine.ts` | 保留 |
| `config.ts` | 保留 `~/.aichat/` 配置体系 |
| `cli.ts` | 保留 `activate` / `start` 命令 |
| `ai/openclaw.ts` | **替换**为 `claw/openclaw.ts`（HTTP SSE → WS RPC） |
| `ai/engine.ts` | **替换**为 `claw/interface.ts`（统一适配器接口） |
| `commands/start.ts` | 重构，初始化 router + 适配器 |

---

## 七、目录结构

```
aichat-plugins/                  # 现有仓库，重构为 monorepo
├── packages/
│   ├── node/                    # aichat-node（统一桥接层）
│   │   ├── package.json
│   │   └── src/
│   │       ├── cli.ts                  # aichat activate / start
│   │       ├── config.ts               # ~/.aichat/ 配置体系
│   │       ├── server/
│   │       │   └── hula-ws.ts          # HuLa-Server WS 连接
│   │       ├── claw/
│   │       │   ├── interface.ts        # ClawAdapter 统一接口
│   │       │   └── openclaw.ts         # openclaw WS RPC 适配器
│   │       ├── router.ts               # 多 claw 路由器
│   │       ├── stream/
│   │       │   ├── buffer.ts           # 流式缓冲
│   │       │   └── protocol.ts         # STREAM_START/DELTA/END
│   │       ├── handler/
│   │       │   └── message.ts          # ACK/去重/防抖
│   │       ├── auth/
│   │       │   └── machine.ts          # 机器码
│   │       └── utils/
│   │           └── debounce.ts         # 防抖合并
│   │
│   └── claw/                    # aichat-claw（openclaw Plugin）
│       ├── openclaw.plugin.json        # 插件清单
│       ├── package.json
│       └── src/
│           ├── index.ts                # Plugin 注册入口
│           ├── channel/
│           │   ├── config.ts           # Channel 配置适配器
│           │   ├── outbound.ts         # 消息发送
│           │   └── gateway.ts          # 连接生命周期
│           └── tools/
│               ├── index.ts            # Tools 批量注册
│               ├── send-message.ts     # hula_send_message
│               ├── find-friend.ts      # hula_find_friend
│               ├── add-friend.ts       # hula_add_friend
│               ├── list-friends.ts     # hula_list_friends
│               └── post-feed.ts        # hula_post_feed
│
├── package.json                 # monorepo root (workspace)
├── tsconfig.json
└── README.md
```

---

## 八、配置体系

### 8.1 aichat-node 配置（~/.aichat/config.jsonc）

```jsonc
{
  // HuLa-Server 连接
  "server": {
    "url": "ws://192.168.8.84:18080/api/ws/ws"
  },
  // claw 适配器配置
  "claws": {
    "openclaw": {
      "gatewayUrl": "ws://localhost:18789",
      // token 自动从 ~/.openclaw/openclaw.json 读取
    }
    // 未来:
    // "zeroclaw": { "gatewayUrl": "ws://localhost:28789" }
  }
}
```

### 8.2 aichat-claw 插件配置（openclaw.json 中）

```jsonc
// ~/.openclaw/openclaw.json
{
  "plugins": {
    "entries": {
      "aichat-claw": {
        "enabled": true,
        "config": {
          "hulaServerUrl": "http://192.168.8.84:18080/api",
          "aiclawToken": "连接凭证（用于调 HuLa API）"
        }
      }
    }
  }
}
```

---

## 九、对比当前架构的改进

| 维度 | 当前 | 目标 |
|------|------|------|
| AI 上下文 | 无（HTTP 无状态） | openclaw 自动管理（会话历史 + compaction） |
| 记忆 | 无 | openclaw memory 系统自动生效 |
| Skills | 无 | openclaw skills 全部可用 |
| Agent Tools | 无 | HuLa 能力注入（找好友、发消息、发朋友圈等） |
| AI 主动操作 | 不可能 | Agent 可主动调用 HuLa 能力 |
| 多 claw 支持 | 只支持 openclaw | 统一 ClawAdapter 接口 |
| 通信方式 | HTTP SSE（无状态） | WS RPC（持久连接 + session） |
| 首 token 延迟 | 高（agent 工具调用阻塞） | 更低（gateway event stream 实时推送） |

---

## 十、实施计划

```
Phase 1：基础重构（aichat-node + aichat-claw 骨架）
  ├─ monorepo 初始化（packages/node + packages/claw）
  ├─ ClawAdapter 接口定义
  ├─ openclaw WS RPC 适配器
  ├─ aichat-claw Plugin 骨架（注册入口 + 空 Channel）
  └─ 验证：消息通过 WS RPC 走 agent pipeline，上下文生效

Phase 2：Agent Tools 注入
  ├─ hula_send_message / hula_find_friend
  ├─ aichat-claw Plugin 注册 Tools
  └─ 验证：AI 能主动调用 HuLa 能力

Phase 3：多 claw 路由
  ├─ router.ts + ClawAdapter 接口验证
  ├─ aichat-node 路由逻辑
  └─ 预留 zeroclaw adapter 接口

Phase 4：高级功能
  ├─ hula_post_feed（朋友圈）
  ├─ hula_add_friend（加好友）
  ├─ AI 主动发起对话
  └─ 多 aiclaw 共享一个 node 进程
```

---

## 十一、风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| openclaw WS RPC 协议变更 | 适配器失效 | 锁定 openclaw 版本；RPC 客户端做兼容层 |
| agent 响应时间长（工具调用链） | 首 token 延迟 | StreamProcessor 超时延长至 90s；前端"思考中"状态 |
| aichat-claw Plugin 加载失败 | openclaw 启动异常 | Plugin 做错误隔离；失败时 fallback 到 HTTP SSE |
| 多 claw 并发资源竞争 | 内存/CPU | 每个 claw adapter 独立连接池；node 进程资源限制 |

---

## 十二、参考资料

- OpenClaw 开发指南：`reference/OpenClaw-Dev-Guide/`
- OpenClaw 源代码：`reference/openclaw/`
- openclaw Plugin SDK：`reference/openclaw/src/plugins/`
- openclaw Channel 类型定义：`reference/openclaw/src/channels/plugins/types.plugin.ts`
- openclaw Gateway 协议：`reference/openclaw/src/gateway/protocol/`
