# 实施 brief：TASK-0070 批次一（下发状态页 + 批量导入向导 + 凭据管理页）

> 0070 整任务含七页组，其中四页组后端 API 未交付（待升级清单=0016、调用方=0052、适配器=0033+、重算入口=0047）——**协调者分两批派发**：本批=批次一（API 已交付三页组），批次二待依赖交付后另行 brief（发布页影响预览段=0015 本批交付后接入）。TASK-0070 保持「进行中」不收尾，直至两批全齐。

## 批次

- 批次一（backend API 已交付：0024/0025 发布域、0019 凭据域、0022 导入域）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `frontend/src/**`（views/iot + api/iot 新增页面与客户端，照 0055/0056 既有范式）
  - 前端路由/菜单挂载配置（照 0056 既有挂载方式）
  - 不碰 backend/**、ingest/**、契约文件、workitems（报告除外）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0070.md`（本批只做其中三页组）
- 关联：workitems 库 `features\FEAT-0018.md`、`testcases\TC-0018.md`
- 上游交付范式：`reports\TASK-0056-report.md` + `reports\TASK-0055-report.md`（控制台框架/第一组页/组件与 API 客户端范式——**同框架同组件**是 0070 的明文要求）
- 后端 API 契约源（开工先读 controller 形态）：`D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\iot\publish\controller\IotPublishController.java`（GET /iot/publish/list 版本列表 + GET /iot/publish/log/list 下发状态三态/timedOut/publishState）、iot/credential/controller/**（凭据列表掩码/view 等）、点位导入导出接口（grep iot 点位导入 controller——0022 交付）
- 菜单种子现状：4015 配置发布页 + 4070-4073（0024）+ 4074（0025）已在后端种子——前端路由与菜单对齐照 0056 范式

## 本批三页组的接口与决策（逐条结论）

1. **下发状态页**（挂发布/版本上下文）：按 versionId 或 scope 查询下发记录列表——每目标行（目标类型/标识/ack_status 三态徽标/ack_time/error_msg）+ timedOut 标红 + 版本级 publishState 汇总（全部生效/存在失败/下发中）；重发按钮（POST /iot/publish/resend，iot:publish:resend）带确认框，已生效行按钮置灰；空态/分页/乐观锁交互照 0056 范式。
2. **批量导入向导**（点位批量导入）：上传 Excel → 逐行回执展示（成功/失败行明细）→ 错误行下载（后端 0022 交付的回执形态照 controller 为准）→ 结果汇总；向导步骤条交互，失败不中断逐行回执展示。
3. **凭据管理页**：凭据列表（只展示类型/掩码/key_version/状态——**永不展示密文**）；查看明文入口（单设权限 iot:credential:view，无权限角色隐藏入口照 0019 口径）带二次确认 + 操作留痕提示；轮换/吊销动作带确认框。
- 错误码/HTTP 形态照前端既有 api/iot 客户端封装（0056 范式），不自造请求层。

## 本批不做（登记后续批次，不建占位代码）

发布页影响预览+二次确认段（等 TASK-0015 交付）、待升级清单页（0016）、调用方管理页（0052）、适配器管理页（0033+）、指标重算入口（0047）。

## 验收命令

环境：pnpm 三门（build:test/lint/typecheck，package.json scripts）；backend 8098 可能被其他在途 agent 占用——**启动前探测：空闲则做一轮页面走查冒烟**（backend dev 起→frontend dev（8099）→三页走查：下发状态空态+凭据掩码+导入向导回执→全停）；忙则跳过联调冒烟并在报告登记「联调冒烟待补」（协调者后续窗口补）。

1. `pnpm install`（如依赖未变则跳过并说明）→ `pnpm build:test` → 0 error
2. `pnpm lint` → 0 error（warning 数对照开工基线）
3. `pnpm typecheck` → 0 error
4. 联调冒烟（视 8098 空闲情况，见上）
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落 frontend/src/** 与路由配置内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-report-batch1.md`。五项：改动文件清单 / 验收命令原始输出 / 未做项与原因（含批次二登记清单）/ 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；**提交只 add `frontend/**`**（绝不 `git add -A`，主仓同期有其他道提交）；提交注明 TASK-0070 batch1；不改 workitems 库（唯一例外：写报告）；不把 TASK 置「已完成」。
- 与 TASK-0056 同框架同组件（明文要求）；不自造 UI 组件库/请求层。
