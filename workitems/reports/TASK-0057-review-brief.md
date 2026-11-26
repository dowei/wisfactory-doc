# 评审 brief：TASK-0057 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；编辑器校验规则与后端 Expander 同口径是硬约束）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0057-diff.txt`（TASK-0057 单提交 587d5e6，6 文件约 1050 行；协调者越界反查：全部落在 brief 声明范围 frontend/iot 域与 tests）
- 实施 brief（交付三件/接线义务/语法口径权威/验收命令）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0057-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0057.md`
- 实施报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0057-report.md`
- 语法口径权威：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/RelativeRefExpander.java`（0013——前端规则必须与其同源）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：校验规则表与 RelativeRefExpander 同口径（{TOKEN} 字符集/前缀白名单 SELF./PARENT./长度 2000/括号配平/花括号配对）、引用选择器禁手输+上下文自动包装、接线义务占位（影响预览/试算/循环依赖三件不得实现假功能）、template-item-modal 接入不破坏既有编辑流；②明显质量缺陷（错误处理、边界——重点：校验规则的边界（嵌套花括号/空 token/超长输入的 UI 行为）、选择器数据源为空态、组件 props 契约清晰度（0036/0016 复用依据）、vitest 断言真实性）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0057-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
