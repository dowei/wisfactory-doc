# 评审 brief：TASK-0010 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0010-diff.txt`（TASK-0010 单提交 9a4c6ef，4 文件约 513 行；协调者越界反查：全部落在 brief 声明范围）
- 实施 brief（三态校验器契约、文案区分、纯查询口径、租户双保险口径）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0010-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0010.md`
- 实施报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0010-report.md`
- 既有实现（口径基准）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java`（importDicts 的 IN 预检与状态语义）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：三态语义（OK/NOT_REGISTERED/DISABLED，deleted 行=未登记）、批量一次 IN 预检、文案区分未登记/已停用、纯查询不抛异常、tenantId 显式双保险与插件注入的交互（brief 裁决 3 允许实测调整，须有说明）；②明显质量缺陷（边界——空集合/超大集合入参、SQL 注入面、接口 javadoc 契约完整性（0011 接线依据）、测试断言真实性）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0010-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
