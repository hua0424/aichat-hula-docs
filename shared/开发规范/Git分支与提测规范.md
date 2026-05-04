# Git分支与提测规范

## 结论
采用三层分支模型：`master`（生产）/`dev`（测试集成）/`feature/*`（功能开发），测试环境基线分支为 `dev`。

## 分支定义
- `master`：生产稳定分支，仅接受经过验收的变更
- `dev`：测试集成分支，测试环境默认发布来源
- `feature/*`：功能开发分支（示例：`feature/chat-message-search`）
- `hotfix/*`：生产问题紧急修复分支
- `release/*`（可选）：版本冻结与发布前验证

## 合并策略
1. `feature/*` → PR → `dev`
2. `dev` 通过 QA 验收后 → PR → `master`
3. `hotfix/*` 修复后需回合并 `master` 与 `dev`

## 提测规范
- 必须提供：
  - 变更摘要
  - 影响范围
  - 测试点
  - 回滚说明
- CI 产物必须包含可追溯镜像 tag（建议：`dev-<shortsha>`）
- 未通过门禁（编译/单测/关键检查）不得提测

## 提交规范（建议）
- 提交信息建议遵循 Conventional Commits
  - `feat:` 新功能
  - `fix:` 修复
  - `refactor:` 重构
  - `docs:` 文档
  - `chore:` 构建/工具

## 保护规则（建议）
- `master` / `dev` 禁止直接 push
- 必须 PR + 至少 1 名评审通过
- 必须通过 CI 后方可合并

## 目录与工作区规范
- 开发人员：各自独立 clone 工作目录（不得共用同一工作树）
- 运维人员：独立部署目录，仅管理 compose 与配置
- QA：只连接测试环境，不直接操作源码目录
