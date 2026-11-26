# 评审 brief：TASK-0075 diff 评审（BUG 修复类加严）

## 角色

- 代码评审（BUG 修复类任务加严 diff 评审：核验两项加严证据 + diff 质量）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0075-diff.txt`（TASK-0075 单提交 9754e0e，4 文件约 436 行；协调者越界反查：全部落在 TASK 风险核对三行声明范围——切面/实体 + 增量 SQL + 测试）
- BUG 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\bugs\BUG-0003.md`
- 复现 TC：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\testcases\TC-0024.md`
- TASK 详情（含微改风险核对三行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0075.md`
- 实施报告（含偏离 1「三列放宽」决策与其报错 SQL 证据，供交叉印证）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0075-report.md`

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13（BUG 修复类加严）
- **加严核验两项（逐项给结论）**：
  1. 全仓同契约点扫描：sys_operate_log 全库引用（写入路径/查询路径/实体/脚本）是否都被审到，user_id 可空化对既有查询/导出/联动有无破坏（selectMyList 的 NULL 身份行语义、导出列等）；
  2. 回归测试先红后绿：红=修复前 Field 'user_id' doesn't have a default value + 零新增（证据是否与 BUG-0003 复现同款）；绿=落库 user_id=NULL + 认证态回归 user_id=1 不变。
- 评审两样：①与 BUG-0003 修复方案/TASK 风险核对的一致性——重点：偏离 1「三列放宽」（user_id/user_type/user_name）是否有充分依据与副作用评估（NOT NULL 移除对其他写入方的影响、查询兼容）、增量脚本幂等与 01_base 不改纪律、切面 null 容忍的落库字段完整性（尝试上下文记录什么、不造什么数）；②明显质量缺陷（测试是否真实覆盖未认证路径、守卫与异常路径、事务边界）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0075-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条）
- 一句话说明

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
