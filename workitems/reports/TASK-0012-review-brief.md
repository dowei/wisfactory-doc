# 评审 brief：TASK-0012 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；实例树是数据权限组件首个真实消费方，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0012-diff.txt`（TASK-0012 单提交 c9770ef，15 文件约 2725 行；协调者越界反查：全部落在 brief 声明范围——template 服务守卫为 brief 明文授权例外）
- 实施 brief（创建语义/树裁剪裁决/锁定语义/M-6 守卫/增量脚本）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0012-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0012.md`
- 实施报告（含 remark 收窄口径登记，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0012-report.md`
- 组件契约：`workitems\decisions\ADR-0016.md`（裁决 2/3/4/6——空/403、未绑部门 fail-closed、仅本人数据类空集、点位/实例过滤形态）+ `reports\TASK-0005-inputs.md`（§4 步骤 3-5）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：创建语义（TD-0008 设备/组件级强制模板且模板 node_type 一致、工厂/车间 dept 必填、短码字符集与全码长度、ancestors 维护、applied_template_version 初值）、编辑不可改五字段（TD-0010 挪树禁止）与 locked 编辑限制、删除子实例保护、**树裁剪正确性**（allTenant 全树/可见集+根路径骨架/无权兄弟不返回/createByOnly 空树/禁用节点照常返回带 status）、列表 id IN 裁剪、详情 403（SCOPE_FORBIDDEN 首消费点）、M-6 守卫红转绿、增量脚本判存幂等与 4030-4035 权限一一对应；②明显质量缺陷（错误处理、边界、安全——重点：短码拼接的注入面（全码拼进查询）、树裁剪的内存与递归深度、ancestors 与 parent 的事务一致性、组件 VisibleScope 消费的空集语义、跨租户（插件注入 vs 显式条件交互）、重复 code 并发兜底）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0012-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
