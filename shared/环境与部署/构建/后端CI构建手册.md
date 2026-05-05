# 后端构建与运行手册（HuLa-Server + aichat-plugins）

> 面向：DevOps（石哥）、tester
> 责任人：backend（峰哥）
> 更新日期：2026-05-05
> 目标：后端开发容器编译 + Runtime 运行环境搭建说明

---

## 一、后端服务架构速览

HuLa-Server 是 Spring Cloud 微服务架构，实际运行 **4 个核心服务**，通过 API 网关对外暴露统一入口：

| 服务模块 | 容器内端口 | 对外 | 说明 |
|---------|-----------|------|------|
| `luohuo-gateway-server` | 18760 | ✅ 对外唯一入口 | API 网关，路由+鉴权 |
| `luohuo-oauth-server`   | 18761 | ❌ 内部 | 认证服务（登录/Token） |
| `luohuo-ws-server`      | 18762 | ❌ 内部 | WebSocket 长连接 |
| `luohuo-im-server`      | 18763 | ❌ 内部 | IM 业务（消息/好友/群） |
| ~~`luohuo-system-server`~~ | 18764 | — | **不使用**（AI/LLM 能力层） |

前端只需访问 **网关（18760）**，其余端口是微服务内部路由，不对外暴露。

---

## 二、环境要求

| 依赖 | 版本 | 说明 |
|------|------|------|
| Docker | ≥ 24 | 宿主机无需安装 Java/Maven/Node，全部走容器 |
| Docker Compose | ≥ 2.20 | |
| Git | 任意 | 拉取源码 |
| 外部 MySQL | 10.38.10.10:3306 | 数据库 `hula`，用户 `openclaw/a` |
| 外部 Redis | 10.38.10.10:6379 | 密码 `a` |
| 外部 RocketMQ | 10.38.10.10:9876 | NameServer，无 ACL |

---

## 三、源码结构与构建顺序

```
HuLa-Server/
├── luohuo-util/        ← ⚠️ 工具模块，必须最先构建并 install 到本地 Maven
│   └── luohuo-parent/
└── luohuo-cloud/       ← 主多模块项目
    ├── luohuo-dependencies-parent/
    ├── luohuo-public/
    ├── luohuo-gateway/luohuo-gateway-server/   ← 网关
    ├── luohuo-oauth/luohuo-oauth-server/       ← 认证
    ├── luohuo-ws/luohuo-ws-server/             ← WebSocket
    ├── luohuo-im/luohuo-im-server/             ← IM 业务
    ├── luohuo-base/
    ├── luohuo-system/     ← 不使用（LLM 能力层）
    ├── luohuo-ai/
    └── luohuo-support/
```

**⚠️ 关键构建顺序**：`luohuo-util` 是私有工具包，不在 `luohuo-cloud` 的 Maven reactor 中，也不在 Maven Central。**必须先单独 install，后续模块才能解析其依赖。**

---

## 四、容器体系

当前采用 **Dev 容器编译 + Runtime 容器运行** 两阶段模式。

### 4.1 Dev 容器：hula-server-dev

- 镜像：`maven:3.9-eclipse-temurin-21`（见 `infra/docker/hula-server/Dockerfile.dev`）
- 启动方式：`sleep infinity`，开发人员 `docker exec` 进入编译
- 源码挂载：`../../HuLa-Server:/workspace/HuLa-Server`
- Maven 缓存：Docker volume `maven-cache`

**编译命令**（在宿主机执行 `docker exec`）：

```bash
# 1. 先安装 luohuo-util（首次或工具模块有变动时）
docker exec hula-server-dev bash -lc "
  cd /workspace/HuLa-Server/luohuo-util
  mvn clean install -DskipTests -q
"

# 2. 编译 4 个核心服务（约 3-5 分钟）
docker exec hula-server-dev bash -lc "
  cd /workspace/HuLa-Server/luohuo-cloud
  mvn -pl luohuo-gateway/luohuo-gateway-server,luohuo-oauth/luohuo-oauth-server,luohuo-ws/luohuo-ws-server,luohuo-im/luohuo-im-server -am clean package -DskipTests -q
"
```

### 4.2 Runtime 容器：hula-server-runtime

- 镜像：`eclipse-temurin:21-jre`（见 `infra/docker/hula-server/Dockerfile.runtime`）
- 容器名：`hula-server-runtime`
- Profile：`run`（不随 `up-dev.sh` 启动，需单独 `run-runtime.sh`）
- JAR 来源：从 HuLa-Server workspace **只读挂载**，容器启动时由 `start-runtime.sh` 复制到 `/runtime/jars/`
- 更新方式：`docker restart hula-server-runtime`（自动取最新编译产物）

**关键 JVM 参数**（在 `start-runtime.sh` 中注入）：

```bash
java \
  -Dluohuo.nacos.ip=nacos \
  -Dluohuo.nacos.port=8848 \
  -DNACOS_NAMESPACE=bfa0d426-e281-4da0-b830-c3962ed883d2 \  # Runtime 独立命名空间
  -Dluohuo.nacos.local-ip=$(hostname -i) \
  -Dserver.port=18760 \
  -jar /runtime/jars/gateway.jar
```

### 4.3 aichat-plugins Dev 容器

- 镜像：`node:24-bookworm`（见 `infra/docker/aichat-plugins/Dockerfile.dev`）
- 内置 openclaw@latest（AI 引擎网关），启动时 symlink aichat-claw plugin 并运行 openclaw gateway
- 源码挂载：`../../aichat-plugins:/workspace/aichat-plugins`
- openclaw gateway Control UI 端口：宿主机 `18790` → 容器 `18789`

---

## 五、Nacos 初始化（⚠️ 首次部署必做）

服务启动前，Nacos 必须完成命名空间创建和配置上传，否则所有服务无法启动。
**Nacos volume 持久化后，重新部署时无需重复初始化。**

### 5.1 创建 dev 命名空间

```bash
curl -X POST "http://<nacos-host>:8848/nacos/v1/console/namespaces" \
  --user nacos:nacos \
  -d "customNamespaceId=bfa0d426-e281-4da0-b830-c3962ed883d1&namespaceName=dev&namespaceDesc=dev"
```

命名空间 ID 固定为 `bfa0d426-e281-4da0-b830-c3962ed883d1`，所有服务的 `bootstrap.yml` 中已硬编码此 ID。

### 5.2 必须上传的 6 个配置文件

```bash
NS="bfa0d426-e281-4da0-b830-c3962ed883d1"
NACOS="http://<nacos-host>:8848/nacos/v1/cs/configs"

curl -s -X POST $NACOS --user nacos:nacos \
  --data-urlencode "dataId=<配置文件名>" \
  --data-urlencode "group=DEFAULT_GROUP" \
  --data-urlencode "tenant=$NS" \
  --data-urlencode "type=yaml" \
  --data-urlencode "content=<YAML内容>"
```

| 配置文件 | 来源 | 注意事项 |
|---------|------|---------|
| `common.yml` | 见下方 §5.3 | 内容与源码版有多处差异，不可直接用源文件 |
| `redis.yml` | 见下方 §5.4 | 需替换 host/password |
| `mysql.yml` | 见下方 §5.4 | 需替换 host/db/user/password/driver |
| `rocketmq.yml` | 见下方 §5.4 | 需替换 nameserver 地址 |
| `sa-token.yml` | 源码原样上传 | `HuLa-Server/luohuo-cloud/luohuo-support/luohuo-boot-server/src/main/resources/config/dev/sa-token.yml` |
| `luohuo-gateway-server.yml` | 见下方 §5.5 | ⚠️ 网关路由+/api前缀，源码不含此文件，必须手动上传 |

### 5.3 common.yml（与源码差异最大，须完整核对）

以下为 CI 环境所需的 `common.yml` 关键字段，完整内容向峰哥获取：

**① ⚠️ 不能包含 `server.port`**

```yaml
server:
  # 不在此处配置 server.port！
  # 原因：Spring Cloud bootstrap 加载的 Nacos 配置，优先级高于启动命令的 -Dserver.port
  # 各服务端口通过运行时环境变量 SERVER_PORT 或启动脚本单独指定
  shutdown: GRACEFUL
  # ... 其余 server 配置
```

**② WS 服务线程池（缺失则 WS 服务启动崩溃）**

```yaml
luohuo:
  thread:
    core-size: 4        # SessionManager 线程池核心线程数
    max-size: 16        # 最大线程数
    queue-capacity: 1000  # 队列容量（必须 > 0）
    keep-alive: 60      # 空闲超时（秒）
```

**③ 第三方登录占位符（缺失则 OAuth 服务启动崩溃）**

```yaml
# dev/测试环境用占位符即可，不影响账号密码登录
gitcode:
  client-id: dev-placeholder
  client-secret: dev-placeholder
  redirect-uri: http://<前端域名>/oauth/callback/gitcode
gitee:
  client-id: dev-placeholder
  client-secret: dev-placeholder
  redirect-uri: http://<前端域名>/oauth/callback/gitee
github:
  client-id: dev-placeholder
  client-secret: dev-placeholder
  redirect-uri: http://<前端域名>/oauth/callback/github
```

**④ 邮件配置（缺失则 IM 服务启动崩溃）**

```yaml
spring:
  mail:
    host: smtp.163.com
    port: 465
    username: <邮箱账号>
    password: <SMTP授权码>
    properties:
      mail.smtp.ssl.enable: true
      mail.smtp.auth: true
```

### 5.4 中间件配置（Docker 版）

**redis.yml**

```yaml
luohuo:
  redis:
    ip: <redis-host>
    port: 6379
    password: <redis-password>
    database: 0
spring:
  data:
    redis:
      host: ${luohuo.redis.ip}
      password: ${luohuo.redis.password}
      port: ${luohuo.redis.port}
      database: ${luohuo.redis.database}
```

**mysql.yml**（关键差异：数据库名 `hula`，驱动去掉 p6spy）

```yaml
luohuo:
  mysql: &db-mysql
    username: <db-user>
    password: <db-password>
    driverClassName: com.mysql.cj.jdbc.Driver    # ← 不用 P6SpyDriver
    url: jdbc:mysql://<mysql-host>:3306/hula?serverTimezone=Asia/Shanghai&characterEncoding=utf8&useUnicode=true&useSSL=false&autoReconnect=true&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true
  database:
    p6spy: false    # ← 必须关闭
    # ... 其余配置保持源码默认值
```

> 完整 mysql.yml 内容参考峰哥 `infra/README.md`（含所有 druid 连接池参数）。

**rocketmq.yml**

```yaml
luohuo:
  rocketmq:
    enabled: true
    ip: <rocketmq-nameserver-host>
    port: 9876
    access-key: ""
    secret-key: ""
rocketmq:
  name-server: ${luohuo.rocketmq.ip}:${luohuo.rocketmq.port}
  producer:
    group: chatGroup
```

### 5.5 luohuo-gateway-server.yml（⚠️ 网关路由配置）

**此文件源码中不存在，必须手动上传到 Nacos。** 它定义了网关的 `/api` 前缀和各服务的路由规则。

```yaml
server:
  servlet:
    context-path: /api    # ContextPathFilter 读取此值剥离 /api 前缀

spring:
  cloud:
    gateway:
      routes:
        - id: oauth
          uri: lb://luohuo-oauth-server
          predicates:
            - Path=/oauth/**
          filters:
            - StripPrefix=1
        - id: im
          uri: lb://luohuo-im-server
          predicates:
            - Path=/im/**
          filters:
            - StripPrefix=1
        - id: ws
          uri: lb://luohuo-ws-server
          predicates:
            - Path=/ws/**
          filters:
            - StripPrefix=1
        - id: base
          uri: lb://luohuo-base-server
          predicates:
            - Path=/base/**
          filters:
            - StripPrefix=1
        - id: system
          uri: lb://luohuo-system-server
          predicates:
            - Path=/system/**
          filters:
            - StripPrefix=1
        - id: ai
          uri: lb://luohuo-ai-server
          predicates:
            - Path=/ai/**
          filters:
            - StripPrefix=1
```

**请求链路**：`/api/oauth/anyTenant/login` → ContextPathFilter 剥离 `/api` → `/oauth/anyTenant/login` → 路由匹配 `Path=/oauth/**` → 转发到 `luohuo-oauth-server`，`StripPrefix=1` 去掉 `/oauth` → 最终到达 `/anyTenant/login`

> 新增服务模块时需在此文件添加对应路由条目。源码副本：`config/dev/gateway-server.yml`

⚠️ **Nacos 上传 API 必须使用 `tenant` 参数**（不是 `namespaceId`），否则配置会上传到 public 命名空间：
```bash
# ✅ 正确
curl -X POST $NACOS -d "tenant=$NS&dataId=luohuo-gateway-server.yml&group=DEFAULT_GROUP&type=yaml" \
  --data-urlencode "content=$(cat gateway-server.yml)"
```

---

## 六、Runtime 启动与编排

### 6.1 启动流程

Runtime 容器（`hula-server-runtime`）通过 `start-runtime.sh` 脚本启动全部 4 个服务：

1. 从 workspace 只读挂载复制 JAR 到 `/runtime/jars/`
2. 按顺序启动：oauth → system（跳過）→ ws → im → gateway
3. 通过 `tail -F` 持续输出日志保持容器运行

### 6.2 启动命令

```bash
cd /home/hua/projects/backend_workspace/aichat/infra

# 构建镜像（仅首次）
./scripts/build.sh

# 启动运行环境
./scripts/run-runtime.sh

# 更新到最新版本（dev 环境编译后）
docker restart hula-server-runtime
```

### 6.3 Docker Compose 编排（runtime 部分）

```yaml
hula-server-runtime:
  build:
    context: ../..
    dockerfile: infra/docker/hula-server/Dockerfile.runtime
  container_name: hula-server-runtime
  volumes:
    - ../../HuLa-Server:/workspace/HuLa-Server:ro   # 只读挂载，获取编译后的 JAR
    - runtime-logs:/runtime/logs
  environment:
    TZ: Asia/Shanghai
    NACOS_IP: nacos
    NACOS_PORT: "8848"
    NACOS_NAMESPACE: ${HULA_RUNTIME_NAMESPACE:-bfa0d426-e281-4da0-b830-c3962ed883d2}
  ports:
    - "${HULA_SERVER_PORT:-18080}:18760"   # 网关对外（前端联调端口）
  depends_on:
    nacos:
      condition: service_healthy
  restart: on-failure:3
  profiles: ["run"]
  networks: [aichat-dev-net]
```

### 6.4 Nacos 健康检查配置

```yaml
  nacos:
    image: nacos/nacos-server:v2.3.2
    environment:
      MODE: standalone
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:8848/nacos/v1/console/health/readiness >/dev/null || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 20
      start_period: 30s
```

---

## 七、部署后验证

### 7.1 Nacos 服务注册检查（4 个服务全部注册才算正常）

```bash
curl -s "http://<nacos-host>:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=20&namespaceId=bfa0d426-e281-4da0-b830-c3962ed883d1" \
  --user nacos:nacos

# 期望返回：
# {"count":4,"doms":["luohuo-gateway-server","luohuo-oauth-server","luohuo-ws-server","luohuo-im-server"]}
```

### 7.2 网关联通性验证

```bash
# 应返回 HTTP 200（业务层错误码不是 200 没关系，HTTP 层 200 代表网关已路由到 oauth）
curl -s -o /dev/null -w "%{http_code}" \
  http://<gateway-host>:18760/api/oauth/anyTenant/login \
  -X POST -H "Content-Type: application/json" \
  -d '{"account":"test","password":"test","clientId":"test","systemType":2,"deviceType":"PC","grantType":"PASSWORD"}'
```

### 7.3 WebSocket 连通性验证

1. 先调登录接口取得 `token`
2. 用 `wscat` 或浏览器 DevTools 连接：
   `ws://<gateway-host>:18760/api/ws/ws?token=<TOKEN>&clientId=<任意字符串>`
3. 无 token / token 无效时，连接会被服务端主动关闭

---

## 八、常见故障速查

| 现象 | 原因 | 解决 |
|------|------|------|
| 服务启动时所有服务抢同一端口，端口冲突 | Nacos `common.yml` 中含 `server.port` | 删除 `common.yml` 中的 `server.port` |
| WS 服务启动即崩溃，日志含 `LinkedBlockingQueue(null)` | `luohuo.thread.queue-capacity` 未配置（默认 0，非法） | 在 Nacos `common.yml` 补充 `luohuo.thread.*` 四个字段 |
| OAuth 服务启动崩溃，日志含 `gitcode.client-id` / `gitee.client-id` | 第三方 OAuth 配置缺失 | 在 Nacos `common.yml` 补充 gitcode/gitee/github 占位符 |
| IM 服务启动崩溃，日志含 `JavaMailSender` bean not found | 邮件配置缺失 | 在 Nacos `common.yml` 补充 `spring.mail.*` |
| 服务启动但 Nacos 只显示 1 个服务 | 其他服务启动失败但静默退出 | 逐一检查各服务日志尾部 |
| 网关返回 406 | 对应微服务（oauth/ws/im）未在 Nacos 注册 | 确认 4 个服务都已注册并健康 |
| Maven 构建失败：`com.luohuo.basic:luohuo-parent` not found | `luohuo-util` 未先 install | 在 hula-server-dev 容器中先执行 `luohuo-util` 的 `mvn clean install` |

---

## 九、责任与联系

| 事项 | 负责人 |
|------|--------|
| Dockerfile 维护、Nacos 配置内容 | 峰哥（backend） |
| CI 流水线搭建、镜像仓库、部署脚本 | 石哥（DevOps） |
| 测试环境发布审批 | 华血 |
| Nacos `common.yml` 完整内容获取 | 联系峰哥，或查看 `infra/README.md` 中的 Docker 版配置 |
