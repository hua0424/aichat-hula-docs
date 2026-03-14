# ADR-20260313-openclaw 替换为 aichat-plugins

## 状态
已决定（2026-03-13）

## 背景
原项目使用 openclaw 作为核心交付链插件工程，负责与 HuLa-Server 的连接与通讯适配。随着业务演进，需要支持与多种 openclaw 变体的通讯对接，原 openclaw 项目不再满足需求。

## 决定
1. **废弃 openclaw 项目**：从开发工作目录中移除，不再维护
2. **引入 aichat-plugins 项目**：新仓库 https://github.com/hua0424/aichat-plugins.git，用于开发 aichat 与各种 openclaw 变体的通讯插件
3. **分支策略对齐**：aichat-plugins 已建立 `master`（生产）/ `dev`（测试集成）分支

## 影响
- CI 流水线需调整：石哥需将 openclaw 相关构建替换为 aichat-plugins
- Docker Compose 编排需调整：`openclaw-dev-runtime` 容器替换为 `aichat-plugins-dev-runtime`
- 环境变量需调整：`OPENCLAW_PORT` / `OPENCLAW_GATEWAY_TOKEN` → `AICHAT_PLUGINS_PORT` / `AICHAT_PLUGINS_GATEWAY_TOKEN`

## 已更新文档
- `00-总览/项目总览.md`
- `01-项目结构/目录盘点.md`
- `02-环境与部署/后端构建环境设计与登记-20260310.md`
- `02-环境与部署/环境登记与责任分工.md`
- `03-开发规范/Git分支与提测规范.md`（同步更新分支命名：master/dev）
- `05-任务与进度/看板.md`
- `README.md`

## 待办（运维协同）
- [ ] 石哥：调整 CI 流水线，将 openclaw 构建替换为 aichat-plugins
- [ ] 石哥：更新 docker-compose 编排中的 openclaw 相关服务
- [ ] 石哥：更新测试环境 `.env` 变量命名
