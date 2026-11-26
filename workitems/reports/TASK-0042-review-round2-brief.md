# 评审 brief：TASK-0042 scoped 复评 round2（代码评审）

> 修订轮复评（§13：只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为终审。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-diff-round2.txt`（e387c3b..01236fd 限 calc/，2 文件）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-review.md`（I-1 DELTA 复合表达式截断 + M-1/2/3）
- 修订轮报告段：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-report.md`「修订轮」段
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-brief.md`
- 需要上下文时读工作区源码（D:\workspace\WisFactory\calc\...）

## 越界反查（协调者已比对，评审者复核）

- 修复 diff 2 文件（DualTriggerExecutor + OperatorIntegrationContractTest）全落 calc/executor；区间仅一提交无交错；backend/frontend/ingest/契约零触碰。

## 复评要点（只答三问）

1. **I-1 是否真修复**：evaluateDeltaFormula 从 `find()` 改 `matches()` 严格全串匹配纯 `DELTA(key)`——三种复合形态（DELTA(x)/3600 / 2*DELTA(x) / DELTA(x)+1）是否真 fail-closed（sample=null、单节点隔离、不发假 0）；纯 DELTA(x) 仍走原生算子绿；负向测试对旧实现必红（旧返回 success+原始差分）
2. **是否引入新 Critical/Important**：严格全串匹配是否误伤合法形态（如 DELTA(x) 带空白/大小写变体——校验器白名单允许哪些形态需核对）；异常路径与 0041 失败隔离纪律一致（quality 打码而非假 0）；195 测试零回归（194 基线+1）
3. **Minor 处置确认**：M-1（满量程字段缺，deferred 需 CONV-03 或点位档案）、M-2（INTEGRAL ts 跳变无法判定）、M-3（BASE 双字段订正）是否如实登记、未隐藏；round1 重点 7（TDengine 真机未验真证据充分）结论不因本修订改变

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-review-round2.md`。逐条回答三问（结论+出处+一句话说明）；新发现按 Critical/Important/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
