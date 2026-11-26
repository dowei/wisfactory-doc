# 评审 brief：TASK-0020 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0020-diff.txt`（TASK-0020 单提交 36bcd1f，17 文件约 1530 行；协调者越界反查：全部落在 brief 声明范围。**背景**：该提交由被额度中断的会话完成实现，恢复执行者做了全量验收重跑——评审针对 diff 本身）
- 实施 brief（职责边界/事件契约/offline_reason 补列裁决/列表筛选）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0020-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0020.md`
- 实施报告（恢复接手口径，供交叉印证）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0020-report.md`
- 决策：`workitems\decisions\ADR-0009.md`（TD-0029 LWT 事件链）、ADR-0010（判定归计算引擎）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：职责边界（不判定不写 Redis——diff 中不得出现状态 Redis 写入/超时判定逻辑）、事件契约三 reason 枚举、offline_reason 补列与「与上游差异」头注、0017 回写白名单扩清除动作的最小性、list 筛选参数、fail-closed（未知 gatewayId WARN 忽略）、禁用与离线正交；②明显质量缺陷（错误处理、边界、安全——重点：监听器异常不炸主流程、事件消费的幂等性（重复事件）、心跳回写清因与离线事件的竞态、SQL 参数化、0041-0045 守卫不受影响）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0020-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
