# 评审 brief：TASK-0013 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；相对引用展开与依赖图是计算正确性的地基，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0013-diff.txt`（TASK-0013 单提交 fb13384，17 文件约 2480 行；协调者越界反查：全部落在 brief 声明范围——instance 服务删除保护接线为授权例外）
- 实施 brief（生成两触发/字段映射/相对引用展开/full_key/汇总参与口径/领域事件/删除保护接线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0013-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0013.md`
- 实施报告（含相对引用语法的 PDF 提取口径与映射说明，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0013-report.md`
- 决策：`workitems\decisions\ADR-0008.md`（一档口径）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：生成事务内一致（创建实例同事务）、重生成幂等三态（跳过/补/保留历史）、字段映射完整性（含 default_calc_period→calc_period 的 ADR-0011 兑现）、相对引用展开正确性（SELF./PARENT. 语法与 PDF §1.6 对照、PARENT 顶级失败整批）、ref 依赖图逐引用落库、full_key 冗余与双 uk、新增项不新增上级引用（ADR-0008 一档）、领域事件只发布不建监听、config_version 置 NULL、删除保护接线一处、菜单 4036-4040 一一对应；②明显质量缺陷（错误处理、边界、安全——重点：RelativeRefExpander 的语法边界（嵌套引用/循环引用/畸形表达式）、生成链的并发（同实例重生成并发/uk 兜底）、事务边界（事件发布时机与事务回滚交互——afterCommit or 同事务？）、实例删除保护计数口径（活跃行）、SQL 参数化）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0013-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
