# 评审 brief：TASK-0056 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；前端 CRUD 页组——重点接口契约一致性与权限位）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0056-diff.txt`（TASK-0056 单提交 424031b，30 文件约 4680 行；协调者越界反查：全部落在 brief 声明范围 frontend/iot 域）
- 实施 brief（六域页面范围/后端联调不在本轮裁决/交互范式/静态验收门）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0056-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0056.md`
- 实施报告（含页面-接口映射核对表，供交叉印证）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0056-report.md`
- 接口权威（对照源）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/**/controller/*.java`（六域 controller——评审应抽样核对映射表的准确性）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：页面-接口映射核对表与 controller 实际注解/路径/参数一致（抽样验证不少于六域各两接口）、v-privilege 权限位与菜单种子逐一对齐、表单校验与 DTO 注解对齐、停用/删除二次确认、乐观锁预留登记、点位/指标占位维持、tenant 页升级未破坏 0055 已验证的权限显隐骨架、状态规则入口占位；②明显质量缺陷（错误处理、边界——重点：API 封装的响应解包一致性（code 非 200 处理）、conn_config 编辑器安全（JSON 注入/超大文本）、类型定义与后端 DTO 字段对齐、组件复用度、内存泄漏（drawer/modal 挂载））

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0056-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
