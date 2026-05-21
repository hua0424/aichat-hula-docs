# CR-M1: WS 设备指纹从 URL Query 迁移至 Header/SubProtocol

> Owner: server-dev + plugin-dev 协作
> 优先级: P1（REQ-004 合并 dev 后首个 follow-up PR）
> 状态: 设计完成，待三方 dev 对齐后启动实现

---

## 问题

当前 WS 连接建立时，`clientId`（设备指纹）通过 URL query string 传递：

```
ws://gateway/websocket?clientId=abc123
```

**风险**：
- `clientId` 暴露在浏览器历史记录、代理日志、服务端 access log
- HTTP Referrer 可能将含 `clientId` 的 URL 泄漏给第三方
- 服务端 `ReactiveWebSocketHandler.extractClientId()` 从 URI 解析，URI 在各层基础设施中都有记录

## 目标

将 `clientId` 从 URL query string 迁移到 WS handshake header，兼容期内同时支持两种来源，逐步淘汰 query 方式。

---

## 方案

### 1. 传输方式变更

| 来源 | 当前 | 目标 |
|------|------|------|
| URL query `?clientId=xxx` | 唯一方式 | 兼容期 fallback |
| `Sec-WebSocket-Protocol` header | 未使用 | **首选方式** |

**为什么用 `Sec-WebSocket-Protocol`**：
- 标准 WS 握手头，所有 WS 客户端库（浏览器 WebSocket、nodejs ws、Java WebSocketClient）原生支持
- 不会像 URL 那样泄漏到 access log、browser history
- Spring WebFlux `WebSocketSession.getHandshakeInfo().getHeaders()` 可直接读取
- 可携带多个 protocol 值，格式：`Sec-WebSocket-Protocol: aiclaw-v1, clientId_abc123`

### 2. Server 端改造（ws-biz）

**文件**: `ReactiveWebSocketHandler.java`

```java
private String extractClientId(WebSocketSession session) {
    // 1. 优先从 Sec-WebSocket-Protocol 读取
    List<String> protocols = session.getHandshakeInfo().getHeaders()
            .getOrEmpty("Sec-WebSocket-Protocol");
    for (String proto : protocols) {
        // 支持格式: "clientId_abc123" 或 "aiclaw-v1, clientId_abc123"
        for (String part : proto.split(",")) {
            String trimmed = part.trim();
            if (trimmed.startsWith("clientId_")) {
                return trimmed.substring("clientId_".length());
            }
        }
    }

    // 2. 兼容期 fallback：从 URL query 读取（deprecated，后续移除）
    String queryClientId = UriComponentsBuilder.fromUriString(
                    session.getHandshakeInfo().getUri().toString())
            .build().getQueryParams().getFirst("clientId");
    if (queryClientId != null) {
        log.warn("[CR-M1] clientId from URL query is deprecated: session={}",
                session.getId());
    }
    return queryClientId;
}
```

**兼容性**：
- Phase 1（启动后 0~30 天）：同时支持 header + query，query 打 warn log
- Phase 2（30 天后）：移除 query fallback，仅支持 header，query 方式返回 `CloseStatus.BAD_DATA`

### 3. Client 端改造

**plugin-dev / frontend-dev 需同步修改 WS 连接建立代码**：

```javascript
// 浏览器端示例
const ws = new WebSocket(url, ['aiclaw-v1', `clientId_${deviceFingerprint}`]);

// 或 nodejs ws 示例
const ws = new WebSocket(url, {
  headers: { 'Sec-WebSocket-Protocol': `aiclaw-v1, clientId_${deviceFingerprint}` }
});
```

### 4. Gateway 侧

`WebSocketHeaderFilter` 已确保 WS 升级头正确传递，无需额外改动。Gateway 只需保证 `Sec-WebSocket-Protocol` header 透传到 upstream ws-server。

---

## 验收标准

- [ ] `ReactiveWebSocketHandler` 能从 `Sec-WebSocket-Protocol` 正确解析 clientId
- [ ] 兼容期内 query fallback 工作正常并打印 deprecation warn
- [ ] 所有现有 WS 客户端（前端、plugin）迁移到 header 方式
- [ ] query fallback 移除后，仅 header 方式可建立连接

## 关联

- server-dev: `ReactiveWebSocketHandler.java` 改造
- plugin-dev: aiclaw-plugin WS 连接代码改造
- frontend-dev: HuLa App WS 连接代码改造
