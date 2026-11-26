# 评审 brief：TASK-0029 scoped 复评 round2（代码评审）

> 修订轮复评（§13：复评是 scoped 的——只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为终审。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-diff-round2.txt`（cfed417..88dce89，单提交 9 文件）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-review.md`（I1 同步网络 sink + Minor 7 条中 M1-M4 打包修复、M5/M6/M7 裁决 deferred）
- 修订轮报告段：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-report.md`「修订轮」段
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-brief.md`
- 需要上下文时直接读工作区源文件（D:\workspace\WisFactory\ingest\...）

## 越界反查（协调者已比对，评审者复核）

- 修复 diff 9 文件全落 ingest/**（授权范围）。

## 复评要点（只答三问）

1. **I1 是否真修复**：有界异步交接（OnStatus 非阻塞入队 select-default 满队丢弃+计数 / 独立发射 goroutine 逐条短超时 3s / ctx 取消退出）——分发回调内是否零网络调用；非阻塞契约测试（慢发布者 gate 下 3 条 OnStatus <100ms）是否真实锁住非阻塞语义；丢弃计数与 Done()/Dropped() 可观测性；选异步交接而非短超时 ctx 的取舍论证是否成立（paho 心跳叠加论证）
2. **M1-M4 是否真修复**：M1 攒批断言改真护栏（元组数计数+1，≥2 防御口径——修订轮反检出装载探针行合并行为，注释留痕是否清晰）；M2 背书删除+死代码清理+四表逐字锁零损失；M3 RESP 三重上限（bulk 1MiB/数组 1024/回复行 1MiB）红→绿；M4 日志措辞一致
3. **修复 diff 是否引入新 Critical/Important**：异步发射的顺序语义变化（状态事件不再严格有序？ADR-0010 消费方语义是否受影响）、goroutine 生命周期（ctx 取消/泄漏）、丢弃策略（满队丢弃是否丢失关键状态——状态事件幂等/最新态语义）、契约测试 205→211 的既有零改动

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-review-round2.md`。逐条回答三问（每条：结论+出处+一句话说明）；新发现按 Critical/Important/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
