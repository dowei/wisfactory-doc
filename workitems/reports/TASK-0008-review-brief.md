# 评审 brief：TASK-0008 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0008-diff.txt`（BASE 88caf08 → c1b2615，13 文件约 1370 行；协调者越界反查：全部落在 brief 声明范围）
- 实施 brief（含 DDL 逐字、接口契约、路径迁移裁决）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0008-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0008.md`
- 实施报告（含两处申报取舍与基线数字更正，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0008-report.md`
- CONV-01：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\conventions.md`（业务数据字典与 sys_dict 不得混用）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` 的「diff 评审」口径（§13）
- 评审两样：①与 TASK 验收标准 / brief 接口契约的一致性——重点：DDL 与附录 A 逐字性、路径迁移完整性（/tenant→/iot/tenant 是否有漏改点：控制器/测试/前端 api/文档注释）、4002 双道闸、编辑锁 data_code/data_type、DELETE=禁用语义（不物理删不逻辑删、编码永不复用）、菜单 4018-4022 五行不含 import、租户隔离（写侧覆盖+查询插件注入）；②明显质量缺陷（错误处理、边界、安全——重点：data_code/category/data_type 值域校验的合理性、分页、mass assignment、迁移是否破坏既有租户接口的调用方（前端 api 与 0055 交付页面））

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0008-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条）
- 一句话说明

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
