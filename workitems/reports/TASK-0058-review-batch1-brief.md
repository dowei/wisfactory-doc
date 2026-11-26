# 评审 brief：TASK-0058 批次一 diff 评审（代码评审）

> 前端批次一（提交 9aed9a4，BASE=5758d5f，13 文件）——展示端工程与模型树+总览/工厂/车间页骨架。三门已由执行者实测 0 error+583 用例全绿，评审者不复跑构建。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0058-diff-batch1.txt`（5758d5f..9aed9a4，13 文件）
- 批次 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0058-brief.md`（API 现实分批/数据纪律不造假/上看板标记/下钻≤3）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0058-report-batch1.md`（含静态路由偏离登记/联调冒烟缺席登记）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0058.md`
- 关联：FEAT-0019/REQ-0029/TC-0019/ADR-0011（workitems 库）
- 范式基准：reports\TASK-0055-report.md + frontend/src/views/iot/** 与 api/iot/**（0056/0070 范式）；router/index.ts 既有挂载（iotUpgradeListRouter 范式）

## 越界反查（协调者已比对，评审者复核）

- 13 文件全落 frontend/src/** 与 frontend/tests/**；无 backend/ingest/calc/workitems 触碰。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **数据纪律（首要）**：全 diff 扫描确认**无假数据/硬编码假数字**——状态数/实时值/指标/能耗槽位均为骨架+「实时数据接入中」空态；上看板 mark 字段为可选且无标记走普通卡片；未交付页（0059/0060）无死链入口
2. **模型树真接入**：display-api.ts 只封装已交付 `GET /iot/instance/tree`（无未交付端点假调用）；use-display-tree lazy load 与后端数据权限裁剪的边界（前端不二次过滤但空态明确）；无权厂/节点不出现
3. **三级页面与下钻**：总览/工厂/车间三页路由接线（/display/** 独立），下钻点击数≤3（工厂→车间→设备入口；设备详情批次二）；hideInMenu 静态路由范式与 iotUpgradeListRouter 对照——执行者登记的偏离（未走 sys_menu 动态菜单）是否合理
4. **上看板标记（ADR-0011）**：mark 可选字段的渲染逻辑（放大/置顶）与无标记降级；不填值时 UI 行为正确
5. **同框架同组件**：OtLayout/OtLoading/样式体系与 0055/0056 一致；不自造 UI 库/请求层
6. **测试有效性**：3 个 display 单测（routes/slot-card/tree）是否真锁住关键约束（空态无假数据/路由可达/树过滤），而非仅渲染快照
7. 组件卸载清理/内存泄漏（composable 的响应式清理）；lint 9 warning 为存量且本批无新增 warning

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0058-review-batch1.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
