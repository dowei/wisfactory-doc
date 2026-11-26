# 评审 brief：TASK-0002 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；本任务是框架多租户化，风险等级最高，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0002-diff.txt`（BASE 04fc3a5 → 3f3b3c7，33 文件约 2500 行；协调者越界反查：全部落在 brief 声明范围）
- 实施 brief（含接口契约与裁决）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0002-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0002.md`
- 实施报告（含中断恢复盘点、根因链 R1-R6、第二执行者独立复核附录——供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0002-report.md`

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` 的「diff 评审」口径（§13）
- 评审两样：①与 TASK 验收标准 / brief 接口契约的一致性——重点：**白名单式 ignoreTable 是否真正 fail-closed**（六表 + iot_ 前缀之外必须全部忽略，含大小写/反引号表名归一）、**旁路仅限三处 Mapper 方法且显式条件完整**（@InterceptorIgnore 的三处登录/预检/角色查询条件必须自带租户维）、tenant_id 服务层覆盖是否覆盖全部写入口（user/dept/role/dict/config + 关联行）、登录租户解析顺序与禁用拒登、init-admin 路径租户例外与 4002、升级脚本幂等与存量回填；②明显质量缺陷（错误处理、边界、安全——重点：注入面（tenant_code 直拼 SQL？）、会话伪造面、旁路滥用、JSqlParser 覆盖盲区、事务边界、并发 init-admin）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0002-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条）
- 一句话说明

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
