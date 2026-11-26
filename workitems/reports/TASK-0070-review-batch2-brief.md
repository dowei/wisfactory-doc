# 评审 brief：TASK-0070 批次二 diff 评审（代码评审）

> 前端批次二（提交 e492973，BASE=88dce89，12 文件——发布页影响预览段/待升级清单页/快照查看入口）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」；前端三门前置已由执行者实测 0 error + 真登录全链走查含 5002 三支路与内容漂移复现，评审者不复跑构建）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-diff-batch2.txt`（88dce89..e492973，12 文件）
- 批次 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-brief-batch2.md`（三件组接口决策/5002 交互锚点/顺手清理三项）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-report-batch2.md`（含 BASE 修正说明与冒烟遗留登记）
- 批一评审报告（范式与 Minor 基线）：`reports\TASK-0070-review-batch1.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0070.md`
- 后端契约源（核对页面调用与 API 形态）：iot/template/controller/IotModelTemplateController.java（impact-preview/upgrade/list/upgrade/confirm）+ entity/dto/TemplateImpactPreviewVO.java、TemplateChangeEntryVO.java + IotModelInstanceController.java（lock）+ iot/publish/controller/IotPublishController.java（publish 的 confirmToken、snapshot）+ ot-core ResultCode（5002）
- 决策权威：workitems 库 `decisions\ADR-0008.md`（三档政策——UI 文案核对基准）

## 越界反查（协调者已比对，评审者复核）

- 88dce89..e492973 共 12 文件全落 `frontend/src/**`，无越界。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **previewToken 协议交互完整性**：预览→携 token 发布→5002 三支路（未预览/凭据过期/内容漂移）→预览态重置→重预览——分支覆盖与用户引导文案；无 T2 不带 token 直发的判定口径（前端如何知道「无 diff/T1」——依据预览响应哪一字段）
2. **diff 展示与 ADR-0008 一致性**：逐条目档位徽标 T1/T2/T3 映射、T3「逐台迁移不随本次发布」提示、affected/locked 分列——文案与 ADR 语义逐条对照
3. **待升级清单页**：upgrade/list 行集与 0016 响应字段对齐（instanceCode/applied/current/locked/tierSummary/diff）；批量确认 instanceIds[]+targetVersion 组装；确认后行消失/空态；锁定开关交互与 0012 口径（locked=1 编辑受限）呼应
4. **快照抽屉**：哈希展示+复制、快照体折叠树/原文切换（RawValue 原始字节透传后的体形态——0107 修复后出站体=原始字节，前端渲染是否按原样处理）、4071 权限位
5. **同框架同组件**：与 0056/批一范式一致；新弹窗 defineExpose 完整（冒烟修过 1 处——核有无第四处遗漏）
6. **顺手清理三项**：R-2 accept 收窄/R-3 文案/R-6 权限位——是否按 brief 落地且未越界
7. 空态/加载态/错误态三态；组件卸载清理；分页交互

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-review-batch2.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
