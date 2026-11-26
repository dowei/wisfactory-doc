# 评审 brief：TASK-0073 diff 评审

## 角色

- 代码评审（L1-execute §13 diff 评审口径）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0073-diff.txt`（= `git diff d971c10..cff0642`，本任务唯一提交 cff0642 的完整 diff；已做越界反查：15 个文件全部落声明范围 `ingest/**`，无声明集之外文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0073-brief.md`（含改动范围声明、接口与决策、验收命令）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0073.md`

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` 的「diff 评审」口径（§13）
- 评审看两样：**与 TASK 验收标准 / brief 接口的一致性**，和**明显质量缺陷**（错误处理、边界、安全）。发现分级 Critical / Important / Minor。
- 契约符合性是本任务的核心验收标准：diff 中解析样例、质量码映射、写语句形态如需对照原文，契约文件在 `D:\workspace\WisFactory\resources\contract\`（unified-message-model.md v1.1.0 / config-snapshot-schema.md v1.0.1 / tdengine-super-tables.md v1.0.0），CONV-02 在 `D:\Documents\dowei\2.项目资料\智慧工厂\workitems\conventions.md`。注意：契约本身是冻结只读权威，发现「实现与契约不符」按一致性缺陷分级；发现「契约本身留白/未定」不是缺陷，标注为留白即可。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0073-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条 / 哪个 TASK 哪个字段）
- 一句话说明

结尾给一行总结论：`VERDICT: PASS`（无 Critical/Important）或 `VERDICT: FAIL`（有 Critical/Important）。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
