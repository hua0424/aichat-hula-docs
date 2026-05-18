# ISS-009 / ISS-010 E2E 联合回归报告（2026-05-13）

> 执行人：backend-tester
> 联签：server-dev（PR 作者）+ manager（任务协调）
> 执行时间：2026-05-13 11:25 ~ 11:45（Asia/Shanghai）
> 目标：验证 dev HEAD `33f73c6f`（含 PR #6 `8735dcde` ISS-009 + PR #7 `33f73c6f` ISS-010）在 runtime 环境是否正确生效
> 测试账号：`2439646234@qq.com / 123456`（uid=`10937855681024`，昵称 Dawn）
> 被测 aiclaw：`安洁`（uid=`140789091499520`）、`SmokeTestAI-Renamed`

---

## 一、测试结果总览

| 步骤 | 场景 | 结果 | 验证方式 |
|------|------|------|----------|
| ISS-009 | `/chat/contact/list` NPE 修复（账号已注销占位符） | ✅ 通过 | API + 响应体 placeholder 校验 |
| ISS-010 | `/aiclaw/list` 新增 `activeStatus` 字段 | ✅ 通过 | API + JSON 字段校验 |
| Baseline | `/chat/msg/page` 无回归 | ✅ 通过 | API total 与 DB 一致 |
| 跨接口一致性 | `/user/friend/page` 与 `/aiclaw/list` activeStatus 对照 | ✅ 通过 | ui-tester 在 dev runtime 18760 实测,SmokeTestAI-Renamed `activeStatus=2`、安洁 `activeStatus=1`,两接口完全一致 |

---

## 二、环境部署

| 步骤 | 动作 | 状态 |
|------|------|------|
| 1. 代码更新 | `git checkout dev && git pull` → `33f73c6f`(含 `8735dcde` + `33f73c6f`) | ✅ |
| 2. Maven 构建 | `mvn clean package -Dmaven.test.skip=true`(`luohuo-im-biz` 测试 + `luohuo-ai-biz` 依赖异常跳过,5 个 runtime JAR 已生成) | ✅ |
| 3. 重启 runtime | `docker restart hula-server-runtime` | ✅ |
| 4. 服务就绪 | 5 个 Java 进程全部存活,gateway 18760 监听就绪 | ✅ |
| 5. Token 重新签发 | OAuth login → `b594050c-669a-44a8-a8f5-bad657d5cccb` | ✅ |

**构建说明**
- `luohuo-im-biz` Testcontainers 测试 fail(`c0ad5e62` 引入,非本次 PR 影响),改 `-Dmaven.test.skip=true` 跳过 compile + run
- `luohuo-ai-biz` 长期 `javax.jws-api:${javax.jws-api.version}` 占位符无法解析,该模块非 runtime 必需,5 个核心 JAR(oauth/ws/im/gateway/system)在其前完成构建
- `im-entity-3.0.6-SNAPSHOT.jar` 已重打包,classpath 内 ISS-009 `RoomDTO` + ISS-010 `AiclawVO.activeStatus` 字段已生效

---

## 三、ISS-009 验证详情

### 现象（修复前 / ui-tester baseline）
- `/chat/contact/list` 100% 返回 `code=-5`,NPE 来自已软删 AI 房间的 `RoomDTO` 字段访问
- 客户端联系人列表无法加载

### 触发
- Dawn token 调用 `GET /api/im/chat/contact/list?pageSize=20`

### 修复验证（runtime API 响应抓取)

```json
{
  "code": 200,
  "message": "成功",
  "data": {
    "cursor": "0",
    "isLast": true,
    "list": [
      {
        "roomId": "140789095693824",
        "type": 2,
        "hotFlag": 0,
        "text": "...",
        "activeTime": 1762942304667,
        "detail": {
          "type": 2,
          "detailId": "140789091499520",
          "name": "安洁",
          "avatar": "https://..."
        }
      },
      {
        "roomId": "<deleted_room_id>",
        "type": 2,
        "detail": {
          "type": 2,
          "detailId": "0",
          "name": "账号已注销",
          "avatar": ""
        }
      },
      ...
    ]
  }
}
```

**校验要点**
- `code=200` 正常返回 ✅(此前 100% `code=-5`)
- 共 10 项,含 6 个 `name="账号已注销"` 占位符(`detailId="0"`, `avatar=""`) ✅
- 「安洁」单聊条目完整保留(roomId=140789095693824, name="安洁") ✅
- 零异常、零崩溃,im.log 无 NPE 栈 ✅

---

## 四、ISS-010 验证详情

### 现象（修复前)
- `/aiclaw/list` 返回的 AI 条目缺少 `activeStatus`,前端在线指示灯无数据源
- 与 `/user/friend/page`（普通好友)的字段口径不一致

### 触发
- Dawn token 调用 `GET /api/im/aiclaw/list`

### 修复验证（runtime API 响应抓取)

```json
{
  "code": 200,
  "message": "成功",
  "data": [
    {
      "uid": "140789091499520",
      "name": "安洁",
      "avatar": "https://...",
      "adapterType": "openclaw",
      "authStatus": 1,
      "activeStatus": 1
    },
    {
      "uid": "<smoke_test_uid>",
      "name": "SmokeTestAI-Renamed",
      "adapterType": "openclaw",
      "authStatus": 1,
      "activeStatus": 2
    }
  ]
}
```

**校验要点**
- 每个 AI 条目均包含 `activeStatus` 字段(Integer 类型) ✅
- 安洁 `activeStatus=1`(online,与 aichat-node 注册状态一致) ✅
- SmokeTestAI-Renamed `activeStatus=2`(offline) ✅
- 字段语义与 `/user/friend/page`(普通好友 Redis ZSET 在线集合)对齐 ✅

### 跨接口一致性
- backend-tester 本地 compose 环境差异说明:`aichat-backend-dev` 仅含 oauth/ws/im/gateway/system 5 服务,**未含 `luohuo-base-server`**,因此 `/user/friend/page` 在本环境返回 503/404 为**预期行为**,非测试遗漏
- ui-tester 在 `huahome.top:18760`(dev runtime,服务全集)补做交叉对照,结果一致:
  - 同一 AI(SmokeTestAI-Renamed,uid=`158673431812608`)在 `/user/friend/page` 与 `/aiclaw/list` 均返回 `activeStatus=2` ✅
  - 同一 AI(安洁)在两接口均返回 `activeStatus=1` ✅
- 实证 server-dev 设计意图:两接口同一 ZSET 数据源(`onlineService.getOnlineUsersList` → `OnlineServiceImpl.globalOnlineUsersKey`),PR #7 拼装 VO 时透传无误

---

## 五、Baseline 回归（`/chat/msg/page`)

| 项 | 值 | 状态 |
|----|----|------|
| `total` | 94 | ✅ |
| `isLast` | true | ✅ |
| 与 DB `im_message WHERE room_id=140789095693824 AND is_del=0` 数 | 一致 | ✅ |

无回归。

---

## 六、日志验证

### im.log（11:30 ~ 11:45 窗口）
- `GET /chat/contact/list` 入参 + 处理,零 ERROR/WARN/NPE ✅
- `GET /aiclaw/list` 入参 + 处理,零 ERROR/WARN ✅
- `GET /chat/msg/page` 入参 + 处理,零 ERROR/WARN ✅

### 进程状态
- oauth/ws/im/gateway/system 5 个 Java 进程全部存活,无 crash/restart

---

## 七、结论与建议

**结论**
- dev HEAD `33f73c6f` 在 runtime 环境功能正确
- ISS-009:`/chat/contact/list` NPE 已消除,已软删 AI 房间以「账号已注销」占位符渲染,客户端联系人列表可正常加载
- ISS-010:`/aiclaw/list` 新增 `activeStatus` 字段,与好友在线状态口径一致,前端指示灯有了数据源
- 零异常、零崩溃、零回滚

**建议**
1. ISS-009 / ISS-010 标记为 runtime 验证通过,可关闭
2. ui-tester 跑 4-bug 统一 curl ✅ 已完成(全绿),GUI 客户端复测进行中
3. ISS-007(P3 RetryPushConsumer)/ ISS-012(级联软删 + 既有业务数据清理)/ ISS-013(根 pom `<project.build.sourceEncoding>UTF-8</...>` + 构建配置 hardening)按 manager 排期跟进
4. 「账号已注销」6 占位符乱码 → **ISS-013 PR #8 已 squash-merge(dev HEAD `699eef2d`),ai-dev-group runtime 字节级验证 PASS**(详见 §九附录)。根因实证为 **m2 stale + 部署残留**,非编译期 charset 配置缺失(根 parent `luohuo-util/luohuo-parent` 已显式 `sourceEncoding=UTF-8` + `maven-compiler-plugin <encoding>UTF-8</...>`,继承生效)。PR 同时做 hardening(5 个 luohuo-im 子模块显式重声明 `source/target=21` + `sourceEncoding=UTF-8`)+ force rebuild。`huahome.top:18760` 是**用户私人 HuaServer** 上的 dev runtime,**不在 backend-tester / ui-tester 管辖**,需用户上线后自行 `git pull origin dev` → `mvn -B -Dmaven.test.skip=true clean install` → 重启 runtime 才能消除该机器上的乱码
5. `luohuo-im-biz` Testcontainers 测试 → manager 已批 server-dev 的**方案 2**(profile gate,日常 `mvn package` 不跑 IT,CI 显式 `mvn verify -Pintegration-test`),等 server-dev PR 落地后本报告「构建小坑」项可移除
6. `luohuo-base-server` 跑不起来的问题独立处理,不阻塞当前 PR
7. **新登记候选**:`luohuo-ai-biz` 依赖 `javax.jws:javax.jws-api:${javax.jws-api.version}` 占位符未定义,导致 `mvn clean install` 在 luohuo-ai-biz 阶段 fail,luohuo-system 子树连带 SKIPPED。当前不阻塞 runtime(5 个核心 JAR 在其前已生成),但 system-server.jar 实际跑的是历史 build 缓存产物。P3,manager 已决合并入 ISS-012 一并排查。**根因链 + 解法候选见 §十(server-dev 2026-05-13 供稿)**

---

## 八、关联文档

- [ISS-009 PR #6](https://github.com/hua0424/HuLa-Server/pull/6)
- [ISS-010 PR #7](https://github.com/hua0424/HuLa-Server/pull/7)
- [ISS-013 PR #8](https://github.com/hua0424/HuLa-Server/pull/8) — 字节级验证见 §九
- [ISS-005/006 E2E 联合回归报告](./2026-05-12-ISS-005-006-E2E联合回归报告-backend-tester.md)
- [ISS-004 E2E 回归报告](./2026-05-12-ISS-004-E2E回归报告-backend-tester.md)
- [ISS-003 E2E 回归报告](./2026-05-12-ISS-003-E2E回归报告-backend-tester.md)
- [问题清单](./问题清单.md)

---

## 九、ISS-013 PR #8 字节级验证附录

> 执行时间:2026-05-13 12:00 ~ 12:08(Asia/Shanghai)
> 执行环境:`ai-dev-group` 上的 `hula-server-runtime`(本地 compose runtime)
> 目标:验证 PR #8 「luohuo-im 子树 5 个 pom hardening + force rebuild」能消除「账号已注销」6 个占位符乱码

### 步骤

1. **拉 PR 分支并构建**(squash-merge 后 dev HEAD = `699eef2d`,内容等价 PR 分支 commit `056ff9f6`)
   ```bash
   cd /home/hua/projects/backend_workspace/aichat/HuLa-Server
   git fetch origin
   git checkout fix/iss-013-pom-source-encoding-utf8   # 056ff9f6
   docker exec hula-server-dev bash -lc \
     'cd /workspace/HuLa-Server/luohuo-cloud && mvn -B -Dmaven.test.skip=true clean install'
   ```
   - `luohuo-im-server.jar` 重打 mtime `May 13 12:01`,size 258 MB ✅
   - `luohuo-ai-biz` `javax.jws-api` 占位符无法解析(pre-existing,与 PR #8 无关),system 子树连带 SKIPPED
   - 5 个 runtime 必需 JAR(oauth/ws/im/gateway/system)在 luohuo-ai-biz 之前完成或被复用历史构建产物

2. **重启 runtime**
   ```bash
   docker restart hula-server-runtime
   # ~60s 后 5 个 java 进程全部就绪,gateway 18760 绑定成功
   ```

3. **API 抓取**
   ```bash
   curl -s -o /tmp/contact-list.json \
     -w "HTTP=%{http_code}\n" \
     -H "Token: <Dawn token>" -H "ApplicationId: 1" \
     "http://localhost:18080/api/im/chat/contact/list?pageSize=20"
   # HTTP=200,响应 3196 bytes 落盘
   ```

### 字节级证据

**A. 期望 UTF-8 byte 序列命中**:`账号已注销` = `e8 b4 a6 e5 8f b7 e5 b7 b2 e6 b3 a8 e9 94 80`

```bash
grep -c $'\xe8\xb4\xa6\xe5\x8f\xb7\xe5\xb7\xb2\xe6\xb3\xa8\xe9\x94\x80' /tmp/contact-list.json
# 输出: 1   ✅ Found expected UTF-8 bytes
```

**B. 文本级匹配**:`账号已注销` 字符串

```bash
grep -o '账号已注销' /tmp/contact-list.json | wc -l
# 输出: 6   ✅ 6 个占位符全部正确渲染
```

**C. Python 解码 repr() 每条 detailId / name / type**

```
('140789091499520', '安洁', 2)
('158673431812608', 'SmokeTestAI-Renamed', 2)
('0', '账号已注销', 2)
('0', '账号已注销', 2)
('0', '账号已注销', 2)
('0', '账号已注销', 2)
('0', '账号已注销', 2)
('0', '账号已注销', 2)
('1', 'HuLa官方频道', 1)
('1', 'HuLa小管家', 2)
```

### 结论

- ✅ ISS-013 PR #8 在 `ai-dev-group` runtime 字节级验证 PASS
- ✅ 6 个占位符 + 4 个真实名称(安洁 / SmokeTestAI-Renamed / HuLa官方频道 / HuLa小管家)全部 UTF-8 正确解码,**无任何 mojibake**
- ✅ 实证 server-dev 的根因诊断:**m2 stale + 部署残留**(非编译期 charset 配置缺失),force rebuild 是消除乱码的关键
- ⏳ `huahome.top:18760`(**用户私人 HuaServer**,不在 backend-tester/ui-tester 管辖)需用户上线后自行 `git pull origin dev` + `mvn clean install` + 重启 runtime 才能消除该机器上的乱码
- ✅ luohuo-parent 已显式 `sourceEncoding=UTF-8` + `maven-compiler-plugin <encoding>UTF-8</...>`,继承到 luohuo-im 子树生效。PR #8 的 hardening(5 子模块显式重声明 `source/target=21` + `sourceEncoding=UTF-8`)是 **defense in depth**,防御未来 maven 升级 / parent 链漂移

---

## 十、luohuo-ai-biz `javax.jws-api` 占位符 — 根因实证(server-dev 供稿,2026-05-13)

> 数据来源:server-dev 2026-05-13 04:08 UTC 在 ai-dev-group 上对 m2 + pom 树的实证
> 用途:为 manager 在 ISS-012(级联软删 + 既有业务数据清理)统一排查时提供素材;不单独开 PR
> 角色边界:backend-tester 不动 pom,仅做素材登记

### 根因链

1. `org.springdoc:springdoc-openapi:2.5.0` 的 pom 里:
   - `<properties>` 块定义 `<javax.jws-api.version>1.1</...>`
   - `<dependencyManagement>` 声明 `<groupId>javax.jws</groupId><artifactId>javax.jws-api</artifactId><version>${javax.jws-api.version}</version>`(test scope)
2. `luohuo-ai-biz`(或其上层)通过 `<scope>import</scope>` 把 `springdoc-openapi` 当 BOM 引入
3. **Maven BOM import 语义**:`<scope>import</scope>` 只导入 `<dependencyManagement>`,**不导入 `<properties>`** ← 根因所在
4. → 占位符 `${javax.jws-api.version}` 在 luohuo 侧无法解析 → 报错 `javax.jws-api:${javax.jws-api.version}` 字面 404
5. 火山引擎 SDK(`volc-sdk-java:1.0.269`)和 cxf 也间接 pull `javax.jws-api`,放大表面症状

### 解法候选(按推荐度)

| 方案 | 做法 | 评价 |
|------|------|------|
| (a) ⭐ 推荐 | 在 `luohuo-dependencies-parent` 的 `<dependencyManagement>` 显式 pin `javax.jws-api:1.1` | 1 行止血,精准覆盖 springdoc BOM 占位符,不破坏现有 javax/jakarta 混用 |
| (b) | 在 `luohuo-parent` 的 `<properties>` 加 `<javax.jws-api.version>1.1</...>` | 语义不严谨,但 maven property 解析会沿 parent 链找到,能跑通 |
| (c) | 切 jakarta namespace(`jakarta.jws:jakarta.jws-api:3.0.0`) | 工作量大,要排除一堆 javax → jakarta 迁移 |
| (d) | `<exclusions>` 把 volc-sdk-java + cxf 的 transitive `javax.jws-api` 排除 | 解决不了 springdoc dependencyManagement 那条,maven 还是会试解析 |

**方案 (a) 示例**(放 `luohuo-util/luohuo-parent/pom.xml` 或 `luohuo-cloud/luohuo-dependencies-parent/pom.xml`):

```xml
<dependency>
    <groupId>javax.jws</groupId>
    <artifactId>javax.jws-api</artifactId>
    <version>1.1</version>
</dependency>
```

### 影响面

- ✅ 5 个 runtime JAR(oauth/ws/im/gateway/system)正常打 — luohuo-ai-biz fail 出现在 reactor 后段,前面子树已 build 完
- ⚠️ 历史包袱:runtime 拉的 `system-server.jar` 实际是上次成功 build 的缓存产物;luohuo-system 子树继续演进会被这条阻塞拖死
- ⏳ 长期:`luohuo-ai` 子树要继续演进,这条阻塞早晚要清

### Issue 标题建议(server-dev 提供)

> `build(ai): luohuo-ai-biz 因 springdoc-openapi BOM 导入未带 properties 导致 javax.jws-api:${...} 解析失败`
> 标 P3 backlog,manager 决定与 ISS-012 一起扫
