# 评审 brief：TASK-0064 diff 评审（代码评审）

> 实现提交 537366e（BASE=7771181，单提交，4 文件 630 行）——权限点机械差集收口（种子 SQL+比对脚本+守卫测试扩展）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0064-diff.txt`（7771181..537366e，4 文件：守卫测试/两个比对脚本/升级 SQL）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0064-brief.md`（机械差集四步/高危核验/角色初始化/一一对应断言）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0064-report.md`（含 40 项提取/两处抽取伪影修正/首轮 grep 漏四段式注解的修正记录）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0064.md`
- 权威：详细设计 §3.1/§3.1续/§3.4/§3.5 接口表权限列（D:\Documents\dowei\2.项目资料\智慧工厂\factory-data-platform-detail-design-v1.1.pdf p21-23——评审者可 pypdf 复核抽样）
- 上游范式：菜单种子幂等（0024/0052 脚本）+ 守卫测试范式（IotPublishMenuPermissionGuardTest/0052 菜单守卫测试）

## 越界反查（协调者已比对，评审者复核）

- 4 文件全落声明范围（升级 SQL+守卫测试+两个脚本）；`resources/scripts/` 为 brief 授权的临时产物落点。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **机械差集正确性**：40 项提取清单 vs 库内实态——差集恰 3 项（iot:point:import/export、iot:item:bind）是否真实（评审者 pypdf 抽样复核设计权限列 3-5 项 + 对照现库种子全集）；两处抽取伪影修正（连字 conﬁg→config/换行拆行展开）是否正确且无类似残留
2. **首轮 grep 漏四段式修正**：`iot:template:item:edit` 四段式注解曾被漏——修正后的提取/比对口径是否覆盖全部注解形态（三段/四段/INSERT...SELECT）
3. **守卫测试全局口径**：「全模块 @SaCheckPermission ⊆ 菜单种子登记集」——测试实现是否真全局（包扫描/反射还是 grep）、误报/漏报面、执行者提到的「BUG-0046 防复发」表述与台账 BUG 编号一致性（台账仅 BUG-0001~0005——核实该表述出处，疑为笔误）
4. **种子 SQL**：4079-4081 挂 4011/4010 合理性+超管授权三行+幂等 INSERT IGNORE+头注登记（段位顺延）
5. **脚本质量**：export-design-perms.py / perm-diff-check.sh 可重复执行（幂等）且不误报；脚本落 resources/scripts/ 的合规性
6. 既有测试零回归（2370 全量内；0052/0024 守卫测试不受全局口径扩展影响）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0064-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
