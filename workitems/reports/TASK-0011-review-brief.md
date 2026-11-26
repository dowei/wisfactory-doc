# 评审 brief：TASK-0011 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；模板域 + 删除保护 + 校验器接线，中高体量，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0011-diff.txt`（TASK-0011 单提交 6126aa1，21 文件约 3200 行；协调者越界反查：20 文件落声明路径 + IotDataDictService javadoc-only 为 brief 明文授权例外）
- 实施 brief（DDL 权威页码、接口清单、接线裁决、降级裁决）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0011-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0011.md`
- 实施报告（偏离 2/3/4/6/7 与「与上游差异」头注——供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0011-report.md`
- 关联决策：`workitems\decisions\ADR-0011.md`（TD-0011 default_calc_period 补字段）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：三表 DDL 与附录 A 逐字性（+default_calc_period 补字段与「与上游差异」头注）、接口九个与权限一一对应（4024-4029 六行↔注解双向守卫）、字典校验接线（一次收集全项调 validateForTemplateRef、三态拒整批、首失败项保序、500 上限先于查库）、data_code/template_code 不可改、删除保护两类引用汇总、平台预置降级（接口仅租户模板）、公式占位校验边界；②明显质量缺陷（错误处理、边界、安全——重点：iot_model_instance 最小实体/Mapper 的引用计数查询是否会被租户插件正确过滤（防跨租户引用计数失真）、批量保存的事务边界与并发兜底、删除保护批内缺失 fail-closed、Edit status 白名单语义、XXE/注入面（无文件解析不适用，核 SQL 参数化）、DTO 与 VO 的字段暴露面）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0011-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
