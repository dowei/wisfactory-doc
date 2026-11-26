# 实施 brief：TASK-0098 问卷功能软下线

## 批次

- TASK-0098（本批共 1 个）
- 验收粒度：一个 subagent 跑一次本批整体验收。
- BASE：57804b47（派发前代码仓 `git -C /d/workspace/WisFactory rev-parse --short HEAD` 的结果）
- 改动范围（文件/目录级，越界即 Important 起）：
  - `resources/sql/upgrade/20261125_disable_survey.sql`（新）
  - `frontend/src/router/routers.ts`（改：不展开 surveyRouters）
  - `frontend/src/router/index.ts`（改：摘除 survey 静态路由 push 与 /s/:key 等 public 免登录白名单）
  - `frontend/tests/unit/router/build-routes.spec.ts`（改：修正 survey 路由挂载断言）
  - `frontend/tests/unit/router/router-guard.spec.ts`（改：修正 public 白名单断言）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:/Documents/dowei/2.项目资料/智慧工厂/workitems/tasks/TASK-0098.md`
- 关联对象：无（本任务为独立工程清理，任务关联留空，不对应 REQ/FEAT）

## 接口与决策

- SQL 脚本固定命名 `resources/sql/upgrade/20261125_disable_survey.sql`，只含两条 UPDATE：`sys_module.status=0 WHERE code='survey' AND deleted=0` 与 `sys_menu.visible=0 WHERE module_id=3 AND deleted=0`；文件末尾用注释附反向回滚 SQL（把两值改回 1）。脚本必须幂等（重复执行结果一致）。
- 路由摘除后 `/survey/**` 静态路由不再挂载；`/survey/write`、`/s/:key`、`/survey/form/view`、`/survey/public/result` 不再出现在 public 免登录白名单——直链应返回 404 或跳转登录。
- 必须使用 TASK 正文中列明的精确文件路径，不得改其他文件。

## 歧义裁决（逐条给结论，不给选项）

1. **survey 代码文件是否删除**：不删。前端 `views/api/components/store/assets/composables/plugin/types/utils/survey` 全部保留，仅摘除路由入口。理由：避免与只读上游分叉、后端服务仍存活。
2. **后端是否改动**：完全不动。`backend/.../module/survey`、wechat handler、SysJobServiceImpl、i18n 全部保留。
3. **SQL 历史脚本是否改动**：不改 `02_survey.sql`、不改任何 `upgrade/*survey*.sql`，只新增一份软下线脚本。
4. **sys_role_menu / survey_* 表**：不动、不 DROP，保留全部数据与授权，确保可回滚。
5. **验收加严口径**：本任务 SQL 仅两条 UPDATE，不涉及表结构，不会导致应用启动失败；故以「SQL 幂等性验证（两次执行结果一致，给出静态论证或实测）」替代 prod 档真实启动验证，不要求真实 prod 启动。前端四门仍照常全跑。

## 验收命令

- 命令（在 `/d/workspace/WisFactory/frontend` 下）：
  - `pnpm build:test`
  - `pnpm lint`
  - `pnpm typecheck`
  - `pnpm test`
  - 另附：SQL 脚本内容自检（确认只含两条 UPDATE + 反向回滚注释、无 DROP/DELETE）+ 幂等性论证。
- 期望：四条前端命令 EXIT=0（lint 允许既有 prettier warning，但 0 error）；SQL 脚本符合上述约束。

## 报告

报告写到：`D:/Documents/dowei/2.项目资料/智慧工厂/workitems/reports/TASK-0098-report.md`。必须包含五项：

1. 改动文件清单
2. 每条验收命令的原始输出（不要只写「通过」）
3. 未做项与原因
4. 偏离计划之处
5. 遗留问题

## 契约

- **no-subagents**：不得再派 subagent。
- 只改本批 TASK 范围内的文件；不得把 `状态` 置为「已完成」，那由协调者收尾。
- 不手写索引行；不得改本批之外的对象。
- 测试或验收失败先定位根因再修——禁止为了让输出匹配而改测试、改期望或打补丁。
- 每个修复要能说明「为什么现在对了」，写进报告。

## 收尾回写

- 在 TASK 详情（绝对路径 `D:/Documents/dowei/2.项目资料/智慧工厂/workitems/tasks/TASK-0098.md`）里勾选 `计划子任务`、填 `修改内容` / `任务结果`（含实测输出）；状态保持 `进行中` 不自行改。
- LOG 与 git 哈希由协调者跑 `close-task.sh` 生成。
