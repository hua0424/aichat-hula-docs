# `@aichat/node` 部署改造方案

> 作者：plugin-dev
> 日期：2026-05-12
> 关联问题：[ISS-001](../../风险与问题/问题清单.md) — aiclaw 无 AI 回复
> 关联报告：[2026-05-12 runtime 冒烟测试](../../风险与问题/2026-05-12-runtime冒烟测试-server-dev.md)
> 状态：草案，待 server-dev / backend-tester 对齐
> 适用范围：runtime 联调环境（`hula-server-runtime` + `aichat-plugins-dev` 容器）

---

## 一、背景与裁决

ISS-001 在 manager 处的裁决为 **方案 A：贴 REQ-002 双层架构**：
- `@aichat/node`（Node.js）负责 WS 桥接 + ACK / 去重 / 防抖 + 流式转发；
- `@aichat/claw`（openclaw 原生插件）只暴露工具，不直连 HuLa-Server。

经审计，`packages/node` 已经把 ISS-001 列出的三处缺口（① WS 连接 ② inbound 适配 ③ openclaw 回复闭环）实现完毕（详见 §六）。**剩下的问题是这个进程从来没在容器里被拉起**：

| 项 | 当前状态 | 目标状态 |
|----|----------|----------|
| `aichat-plugins-dev` 容器 command | 只起 `openclaw gateway` + 复制 claw extension | 同时启动 `aichat start` |
| 容器内 `/root/.aichat` | 不存在 | 持久化卷 + 已激活（安洁 token） |
| HuLa-Server 目标地址 | env `HULA_SERVER_URL=http://hula-server-dev:18760`（dev，未使用） | `ws://hula-server-runtime:18760/api/ws/ws`（runtime） |
| `aiclaw token` 存放 | 计划放 `aichat-claw` 配置 `hula.aiclawToken` | 改放 `~/.aichat/credentials.jsonc`（由 `aichat activate` 写入） |
| `packages/node/src/config.ts` `DEFAULT_SERVER_URL` | `ws://localhost:18760/ws/ws`（缺 `/api`） | 通过 `~/.aichat/config.jsonc` 覆盖，不改源码（避免触动 claw-dev 的局部开发） |

---

## 二、目标拓扑（runtime 联调）

```
                       aichat-dev-net
 ┌──────────────────────────────────────────────────────────────────┐
 │                                                                  │
 │   ┌──────────────────── aichat-plugins-dev ──────────────────┐  │
 │   │                                                          │  │
 │   │   openclaw gateway  ── ws://localhost:18789 ──┐          │  │
 │   │                                                │          │  │
 │   │   aichat (node)  ──→ ws gateway (RPC, 安洁 device 签名)  │  │
 │   │     │                                                    │  │
 │   │     └─ ws://hula-server-runtime:18760/api/ws/ws          │  │
 │   │         ?token=<安洁 connectionToken>                     │  │
 │   │         &clientId=<machineCode>                           │  │
 │   │                                                          │  │
 │   │   挂载：openclaw-home → /root/.openclaw                  │  │
 │   │         aichat-home   → /root/.aichat（新增）            │  │
 │   └──────────────────────────────────────────────────────────┘  │
 │                                                                  │
 │   hula-server-runtime  (gateway 18760，宿主机 18080)             │
 └──────────────────────────────────────────────────────────────────┘
```

要点：
- node 不依赖宿主，只走容器网络 `hula-server-runtime:18760`；
- gateway URL 内嵌 `/api`，与前端联调一致（[联调环境说明](../联调/联调环境说明.md)）；
- 不新增 service，沿用 `aichat-plugins-dev`（gateway 与 node 必须同容器，因为 node 走 `ws://localhost:18789` 接 gateway）。

---

## 三、Compose 改造（具体 diff）

涉及文件：`/home/hua/projects/backend_workspace/aichat/infra/compose/docker-compose.dev.yml`（service `aichat-plugins-dev`）。

### 3.1 卷

```yaml
volumes:
  - ../../aichat-plugins:/workspace/aichat-plugins
  - plugins-node-modules:/workspace/aichat-plugins/node_modules
  - openclaw-home:/root/.openclaw
  - aichat-home:/root/.aichat          # 新增：node 凭证与配置
```

底部 `volumes:` 列表新增一行：

```yaml
volumes:
  ...
  aichat-home:
```

### 3.2 env / environment

新增（其余保持）：

```yaml
environment:
  TZ: Asia/Shanghai
  AICHAT_HOME: /root/.aichat
  # node 直接读 ~/.aichat/config.jsonc 中的 server.url（见 §4.1），
  # 这里不依赖 HULA_SERVER_URL；保留旧 env 不会冲突。
```

### 3.3 command（替换原段落）

```yaml
command: >-
  bash -lc '
    set -e
    mkdir -p /root/.openclaw/extensions /root/.aichat

    # 1) 同步 aichat-claw extension（保持原行为）
    if [ ! -f /root/.openclaw/extensions/aichat-claw/openclaw.plugin.json ]; then
      cp -r /workspace/aichat-plugins/packages/claw /root/.openclaw/extensions/aichat-claw
      ( cd /root/.openclaw/extensions/aichat-claw && npm install )
    fi

    # 2) 启动 openclaw gateway 前台
    openclaw gateway 2>&1 | tee /tmp/openclaw-gateway.log &

    # 3) 等 gateway 起来再起 node（gateway 监听 18789）
    for i in $(seq 1 30); do
      (echo > /dev/tcp/127.0.0.1/18789) >/dev/null 2>&1 && break
      sleep 1
    done

    # 4) 守护启动 aichat（exit 后 5s 重启，便于看日志）
    cd /workspace/aichat-plugins/packages/node
    while true; do
      if [ ! -f /root/.aichat/credentials.jsonc ]; then
        echo "[aichat] credentials missing, waiting for activation…"
        sleep 30
        continue
      fi
      node dist/cli.js start 2>&1 | tee -a /tmp/aichat-node.log
      echo "[aichat] exited, retry in 5s"
      sleep 5
    done &

    wait -n
  '
```

> 说明：
> - 引号改成单引号是为了避免 `$(...)` 被 compose 提前展开；
> - 把 `sleep infinity` 换成 `wait -n`，任一前台子进程退出时 compose 重启容器（同时配 `restart: unless-stopped` 也行，但当前 dev 项目没启用，按现状不加）；
> - `credentials.jsonc` 缺失时只警告不退出，给 backend-tester 留激活窗口。

---

## 四、首次激活（一次性，backend-tester 执行）

### 4.1 写 `~/.aichat/config.jsonc`

容器拉起后：

```bash
docker exec aichat-plugins-dev bash -lc 'cat > /root/.aichat/config.jsonc <<EOF
{
  "server": {
    "url": "ws://hula-server-runtime:18760/api/ws/ws"
  },
  "claws": {
    "openclaw": {
      "gatewayUrl": "ws://localhost:18789"
    }
  }
}
EOF'
```

> `claws.openclaw.token` 不需要填：node 端 auto-detect 会从 `/root/.openclaw/openclaw.json` 读 `gateway.auth.token`（[config.ts:119-127](../../../../packages/node/src/config.ts)）。

### 4.2 取「安洁」激活 Token

> **安全约定：activation token 永远不写入本文档 / 任何 repo**。本节使用占位 `<ACTIVATION_TOKEN>`；真实 token 由 server-dev / owner 通过点对点通道（claude-peers `send_message` 或私聊）发给执行人，落到容器内 `~/.aichat/credentials.jsonc` 后即用即弃。

「安洁」(`uid=140789091499520`) 的 aiclaw activation token 来源：

- DB 现状（server-dev 2026-05-12 已 refresh）：`auth_status=0, machine_code=NULL`，新 activation token 由 server-dev 通过点对点通道（claude-peers `send_message`）发给 backend-tester。
- 实现说明：服务端控制器路径是 `POST /api/im/aiclaw/{uid}/refresh-activation`,联调文档里的 `reset-token` 是其别名,语义等价（旧 token 作废 → 生成新 token → `auth_status` 回 0 → `machine_code` 清空）。

### 4.3 跑激活

```bash
docker exec aichat-plugins-dev bash -lc '
  cd /workspace/aichat-plugins/packages/node &&
  node dist/cli.js activate --backend openclaw --token <ACTIVATION_TOKEN>
'
```

成功后会生成 `/root/.aichat/credentials.jsonc`，包含：

```json
{
  "uid": 140789091499520,
  "connectionToken": "...",
  "machineCode": "<UUID>",
  "activatedAt": "2026-05-12T..."
}
```

下一轮 supervisor 循环（最长等 30s）就会自动启动 `aichat start`。

### 4.4 验证

```bash
docker exec aichat-plugins-dev tail -f /tmp/aichat-node.log
```

期望看到（顺序）：
1. `[openclaw] Connected to gateway v…`
2. `[hula-ws] Connecting to ws://hula-server-runtime:18760/api/ws/ws…`
3. `[hula-ws] Connected`
4. （等 owner 在客户端发条消息）`[message] dispatch receiveMessage …`、`[stream] start/delta×N/end`

---

## 五、Owner 端联调脚本（server-dev 执行）

延续 [冒烟测试](../../风险与问题/2026-05-12-runtime冒烟测试-server-dev.md) §一的 1~6 步，重点关注：

| 步 | 期望 |
|----|------|
| 4. `POST /api/im/chat/msg` | 200，DB `im_message` 写入；同时 runtime 日志出现 `pushService.sendPushMsg`（在线集合应含 `140789091499520`） |
| 5. WS streamStart / streamDelta / streamEnd | 触发；`status=complete`；`fullContent` 入库 |
| 6. `GET /api/im/chat/msg/page` | total 与 DB 一致（ISS-002 修复后应稳定） |

任何一步异常 → 抓 `tail -200 /tmp/aichat-node.log` + runtime `im.log` 的相同时间戳片段挂回问题清单。

---

## 六、`@aichat/node` 现状审计（参考，不必读全）

确认实现完整的关键文件（路径相对 `packages/node/src/`）：

| 缺口 | 文件 / 锚点 | 完成情况 |
|------|-------------|----------|
| WS 连接 / 心跳 / 重连 | `server/hula-ws.ts` | 25s 心跳、1→30s 指数重连、token+clientId URL 拼装 |
| ACK + 去重 + 自消息过滤 + 防抖 | `handler/message.ts` | Set(500) 去重；type≠1 跳过；`MessageDebouncer(2s/5/10s)` |
| openclaw WS RPC（v3 device 签名 → agent 调用 → 流式事件） | `claw/openclaw.ts` | challenge / connect / `method:"agent"` / runId 路由 / tick 看门狗 |
| 三段流式上行 | `handler/message.ts` `sendToAI()` | `STREAM_START` → `STREAM_DELTA{chunk,seq}` → `STREAM_END{fullContent,status}` |
| CLI | `cli.ts` + `commands/{activate,start}.ts` | `aichat activate --token … / aichat start` |

未发现需要 plugin-dev 当前改动的代码。**唯一软性问题**：`DEFAULT_SERVER_URL` 默认值缺 `/api` 前缀（[config.ts:49](../../../../packages/node/src/config.ts)），方案里通过 `~/.aichat/config.jsonc` 覆盖，不入侵代码。等本次联调过后再决定是否修默认值（避免影响 claw-dev 的本地环境）。

---

## 七、回滚

部署回滚（不留尾巴）：
1. `git checkout` compose 文件回到改造前；
2. `docker compose -p aichat-backend-dev up -d aichat-plugins-dev`；
3. （可选）`docker volume rm aichat-backend-dev_aichat-home`；
4. `aichat-claw` extension 残留无害，可不动。

被测影响：回滚后 ISS-001 复现，不构成功能回退。

---

## 八、未对齐项 / 风险

| # | 内容 | 责任方 | 阻塞? |
|---|------|--------|------|
| 1 | 「安洁」activation token 通过点对点通道下发（不入 repo） | server-dev → backend-tester | ✅ 阻塞激活 |
| 2 | 是否同意「同容器 supervisor」而非新增 `aichat-node-runtime` service | backend-tester | 否，若不同意我改用新增 service 方案，工作量+1 |
| 3 | runtime Nacos 命名空间是 `bfa0d426-…-883d2`（与 dev 隔离），node 不进 Nacos 不受影响。**仅核对一下不要被误配。** | backend-tester | 否 |
| 4 | `~/.aichat/credentials.jsonc` 含 connectionToken（敏感），`aichat-home` 卷不要泄漏到镜像 / 仓库 | backend-tester | 否 |
| 5 | `packages/claw/openclaw.plugin.json` 源码缺 `activation.onStartup:true`（容器副本已 patch）。本次不阻塞，但建议 plugin-dev 在 ISS-001 关闭后补一个 PR | plugin-dev | 否 |
| 6 | 是否对节点崩溃要告警（telemetry / push） | manager | 否，未来再讨论 |

---

## 九、后续动作清单

- [ ] **server-dev**：通过 claude-peers 点对点通道把「安洁」activation token 发给 backend-tester（不进任何 repo）；
- [ ] **backend-tester**：评审 §3 compose diff 并执行（compose 文件在 backend workspace），确认 4.1 / 4.3 步可正常激活；
- [ ] **plugin-dev**（我）：激活流程跑通后，确认日志符合 §4.4 期望；不通则 hotfix；
- [ ] **server-dev**：复跑冒烟测试 §一 步骤 4-6，更新 ISS-001 至 Closed；
- [ ] **plugin-dev**：另起 PR 修 `packages/claw/openclaw.plugin.json` 的 `activation.onStartup` 与未来的 `DEFAULT_SERVER_URL` 默认值（不阻塞本次联调）。
