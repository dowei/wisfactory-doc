# 评审 brief：FEAT-0022 计划评审（修正后复评）

## 角色

- 计划评审（增强类轻量六项：1 覆盖 / 4 依赖图 / 5 验收可判定 / 6 边界覆盖 / 8 粒度 / 10 TC 覆盖）

## 背景

- 首轮评审 `reports/FEAT-0022-plan-review.md` 发现一项条件回炉：核实发现后端 templateId 创建后不可变、无「已建实例套模板」端点，TASK-0099 原「放开组织节点模板生成」前提有误。
- 已按核实结论修正需求链地基，本次为修正后复评。

## 必读（只给路径，不粘正文）

- 本 brief：`reports/FEAT-0022-plan-review-2-brief.md`
- REQ：`requirements/REQ-0036.md`
- FEAT：`features/FEAT-0022.md`
- ADR：`decisions/ADR-0017.md`（含增补裁决 ⑤：templateId 保持不可变、已建组织节点靠实例级增删补数据；①降级为验证既有能力）
- TASK 集：`tasks/TASK-0099.md`（已重写为验证性任务）、`TASK-0100.md`、`TASK-0101.md`、`TASK-0102.md`、`TASK-0103.md`
- TC：`testcases/TC-0027.md`、`TC-0028.md`、`TC-0029.md`
- 首轮评审：`reports/FEAT-0022-plan-review.md`（了解修正背景，非本次判定依据）

## 依据

- `.agents/protocol/checklists/plan-review.md` 的增强类轻量六项（1/4/5/6/8/10）

## 复评重点（在六项基础上，重点确认首轮发现是否已闭环）

1. 首轮条件回炉项：**套模板入口** —— REQ/FEAT/ADR/TASK-0099/TC-0027 是否已一致改为「新建时选模板 + 已建节点靠实例级增删 + templateId 不可变」，不再有「已建实例事后套模板」的不可执行步骤。
2. 首轮「可自决修改」10 项是否闭环：full_key 层级对照（TC-0027/0028/0029）、删除指标/改数据项公式汇总（TC-0028/TASK-0102）、node_type=3（TC-0029）、后端验收命令指明测试类（TASK-0099/0100/0101）、前端去掉硬编码基线（TASK-0102）、TASK-0103 缺陷来源与命令、并发边界口径、无模板空态（TC-0029）、粒度建议。
3. 六项本身仍需逐项给结论。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\FEAT-0022-plan-review-2.md`。结论三堆：通过 / 可自决修改 / 计划缺陷（回炉），每项发现带出处与一句话说明。明确写：首轮条件回炉项是否已闭环（是/否 + 证据）。

## 契约

- **fresh**：只依据「必读」与「依据」判断，不向任何人追问；判断不了标「无法判定」。
- **no-subagents**：不得再派子代理。
- **只评审不修改**：不改任何文件，产出进报告。
