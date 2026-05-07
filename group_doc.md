# AIChat（HuLa）项目组团队通讯录

## 成员列表

| Peer ID | 角色 | 职责说明 |
|---------|------|---------|
| manager | manager | 项目经理，总负责人，PRD 文档，需求分析，任务拆解、分派与跟踪，团队协调 |
| frontend-dev | developer | 前端开发，负责 HuLa 客户端与管理端界面开发、设计实现 plan |
| server-dev | developer | 后端开发，负责 HuLa-Server 微服务代码开发与设计实现 plan |
| plugin-dev | developer | 插件开发，负责 aichat-plugins 通讯插件代码开发与设计实现 plan |
| backend-tester | tester | 后端测试，负责 API 接口测试、集成测试、Runtime 环境 CI/CD 发布，质量保障 |
| ui-tester | tester | 前端测试，负责 HuLa Windows / Android 客户端界面测试与验收 |
| reviewer | reviewer | 评审人员，负责对 manager 的 PRD、架构设计、数据库设计、接口设计、任务拆分与安排进行评审，对 developer 提交的代码进行 Code Review |

## 项目组公共信息说明

AIChat 项目组通过 `claude-peers` 网络协作。团队按"前端 / 后端 / 插件 / 测试 / 评审 / 管理"六个职能划分。

团队共享文档目录：各 peer 工作目录下的 `aichat-hula-docs/`（Git 仓库：https://github.com/hua0424/aichat-hula-docs.git），需各自 `git pull` 同步最新。

目录结构：
- `shared/` — 共享知识库（项目总览、环境部署、开发规范、决策记录）
- `active/` — 活跃需求（一个 REQ 一个目录，含需求、设计、任务文档）
- `archive/` — 已归档需求
- `看板.md` — 全局任务看板

---

## 职责详情

### manager（项目经理 / 技术经理）

- **Peer ID**：`manager`
- **Host / CWD**：ai-dev-group · `/home/hua/projects/peer_workspace/aichat/manager`
- **职责范围**：
  - 项目总体负责人，冲突决策，争议处理，任务协调与转述
  - 需求分析与 PRD 撰写，系统设计，任务规划，进度跟踪
  - 数据库表结构、接口规范的统一定义与协调
  - 团队协调、任务分配与跨仓库联动
  - 管理与维护 `aichat-hula-docs` 共享文档库，确保信息一致
- **沟通建议**：需求澄清、跨仓库决策、接口规范、表结构修改优先与 manager 对齐

---

### frontend-dev（前端开发）

- **Peer ID**：`frontend-dev`
- **Host / CWD**：ai-dev-group · `/home/hua/projects/peer_workspace/aichat/frontend`
- **负责仓库**：
  - `HuLa` — 即时通讯客户端（Vue3 + Vite7 + Tauri + TS）
  - `HuLa-Admin` — 后台管理端（Vue3 + Vite + TS）
- **关键约定**：
  - 前端通过 API 网关（`18760`）与后端交互
  - WebSocket 连接：`ws://<host>:18080/api/ws/ws?token=<TOKEN>&clientId=<ID>`
  - CORS 白名单、端口配置见 `shared/环境与部署/联调/`
- **沟通建议**：界面交互、接口对接、构建问题与 manager 或 server-dev 对齐

---

### server-dev（后端开发）

- **Peer ID**：`server-dev`
- **Host / CWD**：ai-dev-group · `/home/hua/projects/peer_workspace/aichat/server`
- **负责仓库**：`HuLa-Server`（Spring Cloud 2024 + Spring Boot3 + Java21 + Maven）
- **微服务范围**（仅运行 4 个核心服务）：
  - `luohuo-gateway-server`（18760）— API 网关，路由+鉴权
  - `luohuo-oauth-server`（18761）— 认证服务（登录/Token）
  - `luohuo-ws-server`（18762）— WebSocket 长连接
  - `luohuo-im-server`（18763）— IM 业务（消息/好友/群）
- **不使用**：`luohuo-system-server`（AI/LLM 能力层）、`luohuo-ai-server`
- **关键约定**：
  - 编译在 `hula-server-dev` 容器内完成（Maven）
  - Nacos 配置管理，命名空间 `dev` / `dev-runtime`
  - 外部中间件：MySQL 10.38.10.10:3306 / Redis 10.38.10.10:6379 / RocketMQ 10.38.10.10:9876
- **沟通建议**：表结构变更、接口设计、微服务边界问题与 manager 对齐

---

### plugin-dev（插件开发）

- **Peer ID**：`plugin-dev`
- **Host / CWD**：ai-dev-group · `/home/hua/projects/peer_workspace/aichat/plugin`
- **负责仓库**：`aichat-plugins`（Node.js 通讯插件，仓库：https://github.com/hua0424/aichat-plugins.git）
- **职责范围**：
  - aichat 与 openclaw 等 AI 引擎的通讯桥接
  - openclaw gateway 插件开发与维护
  - 插件与 HuLa-Server 的 WS 连接管理
- **关键约定**：
  - 开发容器 `aichat-plugins-dev`（node:24-bookworm，内置 openclaw）
  - openclaw gateway Control UI 端口：宿主机 18790
  - 通过 WS 连接到 HuLa-Server Gateway:18760
- **沟通建议**：插件架构、openclaw 集成、WS 协议问题与 manager 或 server-dev 对齐

---

### backend-tester（后端测试）

- **Peer ID**：`backend-tester`
- **Host / CWD**：ai-dev-group · `/home/hua/projects/peer_workspace/aichat/tester`
- **职责范围**：
  - 后端 API 接口测试、自动化测试用例编写
  - Runtime 环境管理（通过 `run-runtime.sh` 启动集成测试环境）
  - CI/CD 发布流程执行 — 测试通过后由 tester 发布到测试环境
  - 质量保障，缺陷跟踪与回归测试
  - 联调测试案例编写与维护
- **测试环境**：
  - API Base: `http://<host>:18080/api`
  - WebSocket: `ws://<host>:18080/api/ws/ws`
  - Nacos 控制台: `http://<host>:8848/nacos`
- **沟通建议**：测试环境部署、接口验证、缺陷复现优先与 manager 和对应 developer 对齐

---

### ui-tester（前端 UI 测试）

- **Peer ID**：`ui-tester`
- **Host / CWD**：HuaServer · `D:\projects\hx\HuLa`
- **职责范围**：
  - HuLa Windows 桌面客户端测试
  - HuLa Android 客户端测试
  - 前端界面功能验收、交互体验测试
  - 前端 Bug 复现与跟踪
- **沟通建议**：前端缺陷、界面交互问题与 manager 和 frontend-dev 对齐

---

### reviewer（评审 / Code Review）

- **Peer ID**：`reviewer`
- **Host / CWD**：ai-dev-group · `/home/hua/projects/peer_workspace/aichat/reviewer`
- **职责范围**：
  - 评审 manager 的 PRD、架构设计、数据库设计、接口设计、任务列表与计划
  - 对开发同事提交的代码进行 Code Review（质量、安全、性能、最佳实践）
  - **只评审，不直接修改**；评审结果统一向 manager 汇报
- **使用建议**：
  - 重要需求在派发 developer 执行前，请 reviewer 先过一遍 PRD 和任务清单
  - 每个 REQ 开发阶段完成后请求 Code Review
  - reviewer 给出的 Major / Blocker 级别结论需由 manager 确认后再分发给 developer

---

## 角色通用职责

### developer（frontend-dev / server-dev / plugin-dev）
- 按 manager 分配的设计文档实现功能
- 每个功能点完成后提交 PR，通过 `send_message` 通知 manager
- 及时调用 `set_summary` 更新当前工作状态，方便其他成员了解进度
- 只接收 manager 的新工作安排，需 manager 确认才能启动新工作
- 与其他成员有争议时提交 manager 进行裁决
- 按 manager 要求及时回填 `aichat-hula-docs` 中的任务跟踪文档

### tester（backend-tester / ui-tester）
- 按 manager 分配的测试计划执行测试
- 发现问题及时记录到 `shared/风险与问题/问题清单.md`
- 测试完成后输出测试报告，通知 manager 和对应 developer
- 后端测试负责 Runtime 环境部署与 CI/CD 发布

### reviewer
- 不修改代码，只做审查
- 审查结论只发给 manager，由 manager 分发

---

## 协作流程

1. **需求启动**：manager 在 `aichat-hula-docs/active/` 下新建 REQ 目录（按 `TEMPLATE-REQ.md` 骨架），编写需求文档、设计文档、任务拆解与跟踪文档
2. **设计评审**：manager 主动通知 `reviewer` 评审 PRD 和任务清单，评审结果反馈 manager
3. **任务分发**：manager 检查评审结果决定是否修改，循环迭代直到通过。评审完成后通过 `send_message` 通知对应 peer 执行任务，说明文档在 `aichat-hula-docs` 中的位置
4. **开发执行**：developer 在各自工作目录进行开发，完成后提交 PR，更新任务跟踪文档
5. **测试验证**：backend-tester 部署 Runtime 测试环境，执行接口测试和集成测试；ui-tester 执行客户端验收测试
6. **代码评审**：开发 + 单元测试完成后，manager 通知 `reviewer` 进行 Code Review，评审结果反馈 manager 决定是否修改
7. **归档总结**：REQ 完成后，manager 将 REQ 目录从 `active/` 移至 `archive/`，提炼可复用知识回写到 `shared/`，更新 `看板.md`

## 沟通规范

- **短消息**（通知、确认、简短问题）：直接用 `send_message` 发送
- **大段内容**（设计方案、PRD、Review 意见、测试报告）：写入 `aichat-hula-docs` 对应目录，再通过 `send_message` 发送文件路径，例如：
  ```
  REQ-004 设计文档已更新，请 review：active/REQ-004-xxx/设计-后端.md
  ```
- **同步进度**：每天开始工作时用 `set_summary` 更新自己的摘要，方便其他成员通过 `list_peers` 了解进度
- **文档协作**：各 peer 本地 `aichat-hula-docs/` 需及时 `git pull` 同步，修改后 `git commit && git push`
- **看板更新**：任务状态变更时及时更新 `看板.md`
- **身份确认**：不确定自己当前 peer ID 时，用 `whoami` 确认
