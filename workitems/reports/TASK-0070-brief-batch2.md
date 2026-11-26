# 实施 brief：TASK-0070 批次二（发布页影响预览段 + 待升级清单页 + 快照查看入口）

> 0070 整任务第二批。批次一（下发状态/导入/凭据三页组）已交付并评审通过（reports\TASK-0070-review-batch1.md，Minor 7 条 deferred）。**本批 API 依赖已全部交付**：0015 影响预览+5002 协议、0016 待升级清单/确认/锁定开关、0024 快照查看。调用方（0052）/适配器（0033+）/重算（0047）三页组仍缺 API——归批次三，不在本批。

## 批次

- 批次二（TASK-0070 整任务保持「进行中」，两批全齐后协调者收尾）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `frontend/src/**`（发布页升级 + 待升级清单新页 + 快照查看入口，照 0056/0070 批一范式）
  - 前端路由/菜单挂载配置
  - 不碰 backend/**、ingest/**、workitems（报告除外）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0070.md`
- 批次一报告与评审：`reports\TASK-0070-report-batch1.md` + `reports\TASK-0070-review-batch1.md`（范式基准与 Minor 7 条——批二顺手可清的 R-2/R-3/R-6 若涉及同文件可清，R-7 等后端信封字段限制的不清）
- 上游 API 契约源（开工先读 controller）：iot/template/controller/IotModelTemplateController.java（GET /iot/template/impact-preview、GET /iot/template/upgrade/list、POST /iot/template/upgrade/confirm）+ IotModelInstanceController（PUT /iot/instance/lock）+ iot/publish/controller/IotPublishController.java（POST /iot/publish/publish 的 confirmToken 字段、GET /iot/publish/snapshot）+ iot/template/entity/dto/TemplateImpactPreviewVO.java、TemplateChangeEntryVO.java
- 上游报告：`reports\TASK-0015-report.md`（previewToken 协议/5002）+ `reports\TASK-0016-report.md`（清单/确认/锁定语义）
- 决策：workitems 库 `decisions\ADR-0008.md`（三档政策——UI 文案必须体现：T1 随发布生效/T2 强制确认+锁定不跟随/T3 逐台迁移）
- 契约：5002=下发未确认（ResultCode）——发布确认交互的错误处理锚点

## 本批三件组的接口与决策（逐条结论）

1. **发布页影响预览+二次确认段**（升级 views/iot/publish/index.vue）：发布动作前置——先调 impact-preview（templateId）→ 展示 diff（逐条目带档位徽标 T1/T2/T3）+ affectedInstanceCount/lockedInstanceCount 分列 → 用户确认后携 previewToken 调 publish；**无 diff（T1/无模板变更）直接发布不带 token**；5002 响应 → 引导用户先做影响预览（文案锚定「下发未确认」）→ token 过期/内容漂移（再次 5002）→ 重新预览；T3 条目在 diff 中明确标注「逐台迁移，不随本次发布」。
2. **待升级清单页**（新页挂模板/实例上下文，路由照 0056 范式）：upgrade/list 行（instanceCode/applied vs current 版本/locked 徽标/diff 展开逐条目档位）→ 逐台确认 + 勾选批量确认（confirm 接口 instanceIds[]+targetVersion）→ 确认后行消失/清单空态；**T3 条目确认走逐台迁移语义的 UI 提示**（每台迁移结果展示留待 0081 交付后增强，本批先承接确认动作）；锁定开关入口（PUT /iot/instance/lock）放清单行操作列。
3. **快照查看入口**（4071 iot:publish:query 已有种子）：版本列表行操作列加「查看快照」→ 抽屉展示 GET /iot/publish/snapshot 返回体（version/snapshotHash/snapshotUri/schemaVersion + 快照体折叠树/原文切换）；哈希展示 + 复制按钮。
- 快捷清理（仅涉同文件顺手项）：R-2 accept 收窄 .xlsx、R-3 吊销行轮换文案补「复活启用」提示、R-6 导入下一步挂权限位——不涉其他文件。

## 本批不做（登记批次三）

调用方管理页（0052）、适配器管理页（0033+）、指标重算入口（0047）——API 未交付。

## 验收命令

环境：pnpm 三门；backend 8098 启动前探测（忙则跳过联调冒烟登记待补——批一口径）。

1. `pnpm build:test` → 0 error；`pnpm lint` → 0 error；`pnpm typecheck` → 0 error
2. 联调冒烟（视 8098）：发布页预览→5002→带 token 发布全链；待升级清单行/diff 展开/单台+批量确认；快照抽屉哈希展示；全停
3. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落 frontend/src/** 与路由配置

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-report-batch2.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；**提交只 add frontend/**；提交注明 TASK-0070 batch2；不改 workitems 库（唯一例外：写报告）；不把 TASK 置「已完成」。
- 同框架同组件；UI 文案体现 ADR-0008 三档政策。
