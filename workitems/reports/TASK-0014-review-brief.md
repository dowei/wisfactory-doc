# 评审 brief：TASK-0014 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；小体量任务从严但聚焦）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0014-diff.txt`（TASK-0014 单提交 7fa7cb9，13 文件约 1560 行；协调者越界反查：全部落在 brief 声明范围 dataitem 域与 iot 测试）
- 实施 brief（覆盖契约/运行时继承三态/时间线钩子占位/数据权限接线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0014-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0014.md`
- 实施报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0014-report.md`
- 组件契约：`workitems\decisions\ADR-0011.md`（TD-0022 valid/alarm 分工）、0012 修订轮写路径范式（`reports\TASK-0012-review-round2.md`）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：覆盖仅写实例行四列（partial null=清除）、运行时继承三态（INSTANCE/TEMPLATE/NONE 不复制落库）、min<=max 校验、时间线钩子占位（接口+NoOp+javadoc 接线点，不建表）、数据权限 resolve 校验（0012 修订轮范式）、权限复用 iot:item:*；②明显质量缺陷（错误处理、边界、安全——重点：decimal 范围校验、并发覆盖、钩子接口契约清晰度（0039 接线依据）、测试断言真实性）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0014-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
