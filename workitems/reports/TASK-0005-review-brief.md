# 评审 brief：TASK-0005 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；本任务是跨系统共用的权限组件，风险等级高，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-diff.txt`（TASK-0005 单提交 384d58f，24 文件约 2400 行，已隔离同期 Go 道提交；协调者越界反查：全部落在 brief 声明范围）
- 实现口径权威：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\decisions\ADR-0016.md`（七项裁决——评审逐条对照）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0005.md`
- 设计素材（§4 全文与五种 scope 语义）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-inputs.md`
- 实施报告（供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-report.md`

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` 的「diff 评审」口径（§13）
- 评审两样：①与 ADR-0016 七项裁决的一致性——重点：子树上限截断与 [DATAPERM] 日志格式、空/403 策略常量与判定辅助、仅本人适用集（配置类 create_by 分支/时序空集）、多角色并集与 scope=4 的 OR 合并、中台 scope JSON 解析三态（null 拒绝/subtree false 精确/true 展开）、点位双条件 NULL 不特判（fail-closed）、TDengine 标签渲染用 instance_key/dept_id、VisibleScopeCache 接口+NoOp 接缝；②明显质量缺陷（错误处理、边界、安全——重点：组件 Mapper 的 SQL 注入面与租户隔离（组件只读 Mapper 是否会绕过租户插件或反之）、ancestors 前缀匹配的正确性（前缀匹配 vs 逗号边界——`F01` 不应命中 `F01.WS01` 之外的 `F011`）、多角色并集的空集语义、自包含设计是否泄漏对业务服务的依赖、配置键唯一性）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条）
- 一句话说明

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
