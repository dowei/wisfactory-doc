# 评审 brief：TASK-0083 scoped 复评 round2（代码评审）

> 修订轮复评（§13：只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为终审。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-diff-round2.txt`（4d4012e..7628007 限 backend/，单提交 2 文件；区间内 0041 calc 修订提交已隔离剔除）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-review.md`（F-1 接线时序 Important + F-2/F-3 Minor）
- 修订轮报告段：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-report.md`「修订轮」段
- 实施 brief（**决策 1 已按 round1 计划缺陷纠正——follow 在 assemble 之前**）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-brief.md`
- 上游实证链：0040 calc 运行时只读快照文件（HttpSnapshotFetcher→SnapshotReconciler→DualTriggerExecutor）+ 0024 两段式落位 promote
- 需要上下文时读工作区源码（D:\workspace\WisFactory\backend\...）

## 越界反查（协调者已比对，评审者复核）

- 修复 diff 2 文件（IotPublishServiceImpl + IotPublishAutoFollowTest）全落授权范围；0041 calc 修订（73b8587）已隔离剔除；0081 内核/SQL/yml/ot-core/前端/ingest/calc 零触碰。

## 复评要点（只答三问）

1. **F-1 是否真修复**：followUnlockedInstances 是否真从 assemble 之后前移到 confirm-gate 之后/assemble 之前——**当期发布快照即含跟随新值**（dev 实证 v13 当期快照 1/2-0-13.json 含 /1000 新值与 /100 锁定旧值、无 v14）；失败整批回滚/零半态/网关早返回/T3 skip/乐观锁 4002 是否保持；**原先的静默缺口（applied 推进但快照旧值、无第二次发布永不生效）是否彻底消除**
2. **F-2 是否真修复**：follow 筛选加 status=1，与 0016 清单/确认门/影响预览同口径——核实在 SQL/内存筛选哪一层、边界（status 为 null/其他值）是否合理
3. **修复 diff 是否引入新 Critical/Important**：follow 前移后与 0024 两段式落位（stage/promote）、快照哈希计算、publish_log 派生、NotifySender afterCommit 的时序协调——有无新的半态/哈希失配/重复通知；2403 测试零回归（2401 基线+2）；F-3 仅登记不改（确认 deferred 留痕）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-review-round2.md`。逐条回答三问（结论+出处+一句话说明）；新发现按 Critical/Important/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
