# 评审 brief：TASK-0098 问卷功能软下线

## 角色

代码评审（L1-execute §13 diff 评审）

## 必读（只给路径，不粘正文）

- diff 包：`D:/Documents/dowei/2.项目资料/智慧工厂/workitems/reports/TASK-0098-diff.txt`
- 改动文件清单：`D:/Documents/dowei/2.项目资料/智慧工厂/workitems/reports/TASK-0098-diff-names.txt`
- 实施 brief：`D:/Documents/dowei/2.项目资料/智慧工厂/workitems/reports/TASK-0098-brief.md`
- TASK 详情：`D:/Documents/dowei/2.项目资料/智慧工厂/workitems/tasks/TASK-0098.md`

## 依据

- `D:/workspace/WisFactory/.agents/protocol/L1-execute.md` 的「diff 评审」口径（§13）：评审与 TASK 验收标准 / brief 接口的一致性 + 明显质量缺陷（错误处理、边界、安全）；发现分 Critical / Important / Minor。
- 越界反查已由协调者完成：diff-names 中 5 个文件全部落在 brief 声明的改动范围内（4 个 tracked + 1 个 untracked 新增 SQL）。评审者只需判断范围内改动是否正确，**越界不是本轮关注点**。

## 本轮重点核对项（逐条给结论）

1. **SQL 脚本正确性**：`resources/sql/upgrade/20261125_disable_survey.sql` 是否恰好两条幂等 UPDATE、`module_code` 列名是否与真实 DDL 一致（可查 `resources/sql/01_base.sql` 的 sys_module 建表）、反向回滚注释是否把两列改回 1、有无 DROP/DELETE 或触碰 sys_role_menu / survey_* 表。
2. **路由摘除完整性**：`routers.ts` 与 `index.ts` 是否真正摘除全部 survey 静态路由挂载（四组 + surveyRouters），public 免登录放行是否随路由摘除而失效（代码里 `meta.public` 是通用机制，无独立白名单常量）。
3. **测试有效性**：`build-routes.spec.ts` / `router-guard.spec.ts` 的修改是否真实断言了「survey 路由不再挂载」与「/s/:key 无 token 跳登录」，而不是为了让输出匹配而弱化断言。
4. **残留风险**：是否有被遗漏的 survey 入口仍可达（例如其他静态路由文件、layout 菜单回退分支）——只报事实，不扩大范围。

## 报告

报告写到：`D:/Documents/dowei/2.项目资料/智慧工厂/workitems/reports/TASK-0098-review.md`。发现清单每条带：严重级（Critical / Important / Minor）、出处（文件/行或 TASK 字段）、一句话说明；无发现写「无发现」并给总体结论（通过 / 不通过）。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
