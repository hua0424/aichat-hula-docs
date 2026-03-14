# CI流水线设计（最终版）

## 结论
本项目采用“**容器内构建 + 本地私有镜像仓库 + 测试/生产双标签体系 + 测试发布人工审批**”流程。

- 宿主机不安装 Java/Node 构建工具链；仅需 Docker/Compose/Git
- 后端构建基线固定 **JDK 21 + Maven 3.9+**
- 合并后不做 lint/unit test 强制门禁（按当前管理策略执行）

---

## 1. 适用范围
- `HuLa`（客户端）
- `HuLa-Admin`（管理端）
- `HuLa-Server`（服务端）

---

## 2. 分支与触发策略

### 2.1 触发规则
1. **PR 阶段**：由 Manager（老贾）人工 Review（不强制自动化门禁）
2. **合并到 develop 后**：触发镜像构建并推送测试标签
3. **生产发布时**：打生产版本标签（`vX.Y.Z`）并构建生产镜像

### 2.2 增量构建规则（只改动才重建）
- `HuLa/**` 变更 → 重建 `hula-client`
- `HuLa-Admin/**` 变更 → 重建 `hula-admin`
- `HuLa-Server/**` 变更 → 重建 `hula-server`
- 未改动服务不重建

---

## 3. 镜像命名与标签规范

## 3.1 镜像命名
- `<local-registry>/aichat/hula-client`
- `<local-registry>/aichat/hula-admin`
- `<local-registry>/aichat/hula-server`

> `local-registry` 示例：`registry.local:5000`

## 3.2 标签规范（仅两类）
- **测试标签**：`test-<shortsha>`
- **生产标签**：`vX.Y.Z`

---

## 4. CI标准流水线（简版）

### Stage A：识别改动服务
- 根据 git diff 识别本次需要构建的服务列表

### Stage B：容器内构建
- 每个服务在对应 Builder 镜像内执行构建
- 产物进入 Docker 多阶段构建流程

### Stage C：镜像打包与推送
- 构建 Runtime 镜像
- 打 `test-<shortsha>` 或 `vX.Y.Z`
- 推送至本地私有仓库

### Stage D：测试环境发布（手动）
1. 运维修改 `docker-compose.yml`（仅替换 tag）
2. 提交发布申请
3. **华血审批通过后**执行：
   - `docker compose pull`
   - `docker compose up -d`
4. 验证健康状态并通知测试

---

## 5. 发布与回滚

## 5.1 发布原则
- 测试环境仅消费镜像，不拉源码
- 每次部署必须可追溯到 commit

## 5.2 回滚步骤
1. 确认上一稳定 tag
2. 回改 compose 中镜像 tag
3. `docker compose up -d`
4. 验证核心链路

---

## 6. 职责分工
- **Manager（老贾）**：PR Review、方案把控、节点评审
- **DevOps（石哥）**：流水线维护、镜像仓库、部署执行
- **Backend/Frontend**：维护 Dockerfile 与服务构建脚本
- **QA（思思）**：测试验收与缺陷闭环
- **华血**：测试环境发布审批

---

## 7. 当前不做项（明确）
- 合并后自动 lint 强制门禁
- 合并后自动 unit test 强制门禁
- 外部密钥系统接入（先采用 `.env` 分层）

---

## 8. 后续扩展建议（可选）
- 后续可在不改变主流程前提下，逐步补回：
  - 合并后烟雾测试
  - 单测报告归档
  - 镜像漏洞扫描
