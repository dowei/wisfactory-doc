# 评审 brief：TASK-0043 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；口径标注与比值约束是指标语义正确性的关口，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0043-diff.txt`（TASK-0043 单提交 fb574cf，17 文件约 3370 行；协调者越界反查：全部落在 brief 声明范围 metric 域 + redline 测试增补授权例外）
- 实施 brief（DDL 逐字/CRUD 契约/口径标注硬验收/比值型约束/批量生成占位/裁剪与写防线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0043-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0043.md`
- 实施报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0043-report.md`
- 决策：`workitems\decisions\ADR-0009.md`（TD-0027/TD-0039）、`ADR-0011.md`（TD-0028）、`ADR-0014.md`（编码不入字典）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：DDL 与 p33 逐字性、口径标注强制（agg_type/period 合法枚举）、比值型（agg_type=5）分子分母齐备且同实例存在启用、非比值型 ratio_* 置 NULL、calendar_type 锁 1、metric_code 唯一双道闸+字符集+不入字典、批量生成整批事务逐项校验、**模板自动生成占位口径**（请求清单批量而非模板载体）、裁剪+写路径防线（0021 修订轮范式）、启停事件只发布、上看板占位登记、菜单 4058-4062 守卫；②明显质量缺陷（错误处理、边界、安全——重点：ratio_* 引用的存在性/启用性校验完整性、批量生成并发唯一兜底、裁剪条件 NULL fail-closed、SQL 参数化、事件事务边界）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0043-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
