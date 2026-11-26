# 评审 brief：TASK-0081 scoped 复评 round2（代码评审）

> 修订轮复评（§13：只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为终审。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-diff-round2.txt`（b491e34..5758d5f，单提交 2 文件）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-review.md`（I-1 calc_period 坏码静默降级 / m-1 SecurityUtils 依赖登记 deferred / m-2 T1 VO 误报补插成功）
- 修订轮报告段：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-report.md`「修订轮」段
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-brief.md`
- 需要上下文时读工作区源码（D:\workspace\WisFactory\backend\...）

## 越界反查（协调者已比对，评审者复核）

- 修复 diff 2 文件（IotUpgradeApplyServiceImpl + 测试）全落授权范围；组装器/SQL/yml/ot-core/前端/ingest/calc 零触碰。

## 复评要点（只答三问）

1. **I-1 是否真修复**：新默认非法 period 码（WEEKLY）经 requireMapPeriod 抛 PARAM_ERROR → confirm 事务整批回滚（对齐 0013 fail-closed）；旧默认仍 safeMapPeriod 宽容保留历史升级——两侧语义是否都正确；红→绿负向测试是否真实区分力（对旧实现必红）
2. **m-2 是否真修复**：T1 补插 rowUpdated/动作文案由 regenerateForInstance 返回值决定（返回 0 如实报"未补插"）——核实返回值语义确实反映实际落行数（而非其他含义）
3. **修复 diff 是否引入新 Critical/Important**：requireMapPeriod 的异常类型/事务回滚传播（PARAM_ERROR 是否被 confirm 事务正确捕获为整批回滚而非吞掉）；既有 2384 测试零回归（2387 全绿）；m-1 仅登记不改（确认 0083 复用核实项已留痕）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-review-round2.md`。逐条回答三问（结论+出处+一句话说明）；新发现按 Critical/Important/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
