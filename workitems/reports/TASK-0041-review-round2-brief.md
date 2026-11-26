# 评审 brief：TASK-0041 scoped 复评 round2（代码评审）

> 修订轮复评（§13：只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为终审。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-diff-round2.txt`（9a6cda6..73b8587 限 calc/，单提交 12 文件；区间内 0083 backend 提交已隔离剔除）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-review.md`（F-01 比较反转/F-02 ValueStore 并发/F-03 末值语义 + Minor F-04~F-07 处置）
- 修订轮报告段：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-report.md`「修订轮」段（R1 红绿证据/R2.2 交付值实证）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-brief.md`
- 需要上下文时读工作区源码（D:\workspace\WisFactory\calc\...）

## 越界反查（协调者已比对，评审者复核）

- 修复 diff 12 文件全落 calc/executor 与测试；并行道 0083 的 backend 改动已隔离剔除。

## 复评要点（只答三问）

1. **F-01/F-02 是否真修复**：①>=/<= 比较方向纠正（10>=3→1.0、3<=10→1.0），四法双向+三元分支测试对旧实现必红；②ValueStore HashMap→ConcurrentHashMap，javadoc 一致，4写×4读并发压力测试真实有效（非假并发）
2. **F-03 是否真修复**：WindowIdempotencyGuard DUPLICATE→REPLACE + ResultSink windowKey 覆盖交付 + 执行器每次到达重算覆盖——**最终交付值是否为同窗末值**（单测 9.0→A.V=18.0、live 9 条末值9→18.0=9×2）；幂等是否仍成立（deliveries=1，中间值不算"产出"）；ResultSink 覆盖交付是否引入新语义问题（如重复 windowKey 的旧结果残留）
3. **修复 diff 是否引入新 Critical/Important**：F-04 死字段清理是否波及其他调用点；F-06 MONTH/YEAR 边界说明是否误导（生产不可达诚实登记）；140 测试零回归（134 基线+6）；并发改动对 0040 图结构零影响

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-review-round2.md`。逐条回答三问（结论+出处+一句话说明）；新发现按 Critical/Important/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
