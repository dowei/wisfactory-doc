# 评审 brief：TASK-0024 scoped 复评 round2（代码评审）

> 修订轮复评（§13：复评是 scoped 的——只看修复 diff 与原发现，不重开全场）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」的复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-diff-round2.txt`（BASE=dca3dc9，HEAD=da830e4；两提交 4b7bbe3=TDD 红 / da830e4=修复绿；4 文件 +354/-82）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-review.md`（C-1 并发竞态摧毁胜者快照文件 + I-1 并发测试弱化；Minor 8 条已记 deferred 不在复评范围）
- 修订轮报告（执行者自述与证据）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-revision-report.md`
- 实施 brief（修复目标口径 D2 文件名契约等）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-brief.md`
- 需要上下文时可读工作区源文件与原评审报告引用的代码（`D:\workspace\WisFactory\backend\...`）

## 越界反查（协调者已比对，评审者复核）

- `git diff --name-only dca3dc9..HEAD` 共 4 文件，全部落在修订授权范围（publish service impl / storage / 对应两测试）内，无越界。

## 复评要点（只答三问）

1. **C-1 是否真修复**：两段式落位（stage 唯一暂存 → INSERT 胜者后 promote 原子落位正式名）是否结构性消除两个破坏向量（败者覆盖正式名 / 败者删除正式名）；promote 失败路径是否保证「不存在已提交行指向缺失文件」（事务回滚时序）；孤儿暂存文件权衡是否已登记。
2. **I-1 是否真修复**：并发测试是否真两线程栅栏对齐 ×30 轮、断言是否覆盖「恰好一 200 一 5001 + 胜者行哈希=文件内容哈希 + 败者零残留 + 取回核对」；测试对 uk_scope_version 的 synchronized+DuplicateKeyException 仿真是否等价于物理唯一键语义（不弱化、不放水）。
3. **修复 diff 是否引入新 Critical/Important**：只针对这 4 文件的 diff 本身（并发正确性、事务边界、资源清理、测试有效性），不重开原评审已通过的 8 项。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-review-round2.md`。逐条回答三问（每条：结论 + 出处 + 一句话说明）；新发现按 Critical/Important/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
