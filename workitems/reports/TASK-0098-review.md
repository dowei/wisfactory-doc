# TASK-0098 代码评审报告（fresh 评审）

## 评审输入与范围

- 评审角色：fresh 代码评审（L1-execute §13 diff 评审），只读评审 brief 所列路径及 brief 明确允许核对的 `resources/sql/01_base.sql`；不共享协调者上下文、未派 subagent、未修改任何文件。
- 输入：review-brief / diff 包 / diff-names / 实施 brief / TASK-0098 详情 / L1-execute.md §13 / `resources/sql/01_base.sql`（brief 第 1 项显式授权）。
- 依据级别：与 TASK 验收标准 / 实施 brief 接口的一致性 + 明显质量缺陷；发现分 Critical / Important / Minor。

## 本轮重点核对项逐条结论

### 1. SQL 脚本正确性（`resources/sql/upgrade/20261125_disable_survey.sql`）

**结论：符合。**

- 非注释行恰为两条 UPDATE：① `sys_module SET status=0 WHERE module_code='survey' AND deleted=0`；② `sys_menu SET visible=0 WHERE module_id=3 AND deleted=0`。两条均为「设为固定值」的 UPDATE，重复执行终态相同（status/visible 恒为 0，受影响行数可为 0），满足幂等；`AND deleted=0` 不触及软删除行。
- 列名与真实 DDL 一致：`01_base.sql:139` 的 `sys_module` 编码列名为 `module_code`（表无 `code` 列）、`:143` `status` 为 0禁用/1启用、`:149` 有 `deleted`；`01_base.sql:163` 的 `sys_menu` 有 `module_id`、`:172` `visible` 为 0隐藏/1显示、`:181` 有 `deleted`。脚本 WHERE/SET 列名全部命中。TASK 正文与实施 brief 中写的 `code='survey'` 为笔误，实现者按实际 schema 改用 `module_code`，并在 TASK「修改内容」与 SQL 头注释留痕（若照抄 `code` 会报 Unknown column）；该偏离是修正而非缺陷。
- 反向回滚注释（`diff.txt` 行 333-334）把 `status` 与 `visible` 均改回 1，WHERE 条件与正向一致；TASK 详情记载 survey 30 行菜单基线为 visible=1，故回滚可精确复原。
- 全文无 DROP / DELETE 语句（仅注释文字提及）；未触碰 `sys_role_menu` 与任何 `survey_*` 表。

### 2. 路由摘除完整性

**结论：符合。**

- `routers.ts`：删除 `import surveyRouters from './survey'` 及 `routerArray` 中的 `...surveyRouters`，即 survey 默认导出的全部静态路由（含 `/s/:key` 公开填写页）整体不再注册。
- `index.ts`：删除四组 survey 静态路由的 import，并移除 4 处 push（form workbench / recycle / category / template preview）。brief 要求的「四组 + surveyRouters」全部摘除。
- public 免登录：diff 中未见独立白名单常量的变更；按 brief 第 2 项与 TASK「修改内容/任务结果」，守卫为通用 `meta.public` 机制，无独立白名单常量——survey 路由（含 `/s/:key`、`/survey/write`、`/survey/form/view`、`/survey/public/result`）整体移除后自然失去放行，直链成为未匹配路径，由守卫按无 token 跳登录。符合 brief 第 23 行接口与 TASK 验收标准。

### 3. 测试有效性

**结论：断言真实，但存在回归防护弱化（见 Minor 发现 1、2）。**

- `build-routes.spec.ts`：新增 `assertSurveyRoutesRemoved()`，对 5 个 survey 路由名断言 `hasRoute(...)=false`、对 5 条 `/survey/**` 路径断言 `resolve(...).name === '404'`，并在空模块 / survey 模块 / admin 模块 / 菜单迁移四种状态下调用；同时保留 `/survey/form` 菜单驱动路由仍可达的断言（符合「软下线不改数据层」口径）。断言针对目标行为，未见删除或放宽为满足输出而做的断言。
- `router-guard.spec.ts`：公开页用例改用通用 `/public/demo`（`meta.public`）保留守卫放行分支覆盖；新增「`/s/:key` 无 token 直链跳 `/login` 且调用 logout」断言。断言真实。
- 但两文件将 `/@/router/survey` mock 导出置空（`[]` / `undefined`），使上述断言与生产 survey 路由定义解耦；`build-routes.spec.ts` 注释亦明示「即便被误 push 也不会注册出有效路由」。若未来误恢复 survey 路由挂载，测试仍会通过——属回归防护弱化，不影响本次功能正确性。

### 4. 残留风险

**结论：范围内未见残留；范围外无法判定。**

- 范围内事实：`routers.ts` 与 `index.ts` 已无 survey 的 import/push；survey 源码按实施 brief 歧义裁决 1 全部保留但不再有路由入口；`build-routes.spec.ts` 菜单迁移用例显示菜单数据仍可注册 `/survey/form`，该可达性依赖 SQL 将 `visible=0` 后由菜单过滤消除，属既定「软下线不改数据层」口径，非遗漏。
- 所列路径之外（其他静态路由文件、layout 菜单回退分支是否仍藏有 survey 入口）：受 fresh 只读范围约束，未读取这些文件，**无法判定**。

## 发现清单

1. **Minor** — 出处：`frontend/tests/unit/router/build-routes.spec.ts`（diff 行 89-94，survey mock 段）。将 survey mock 四个导出置为 `[]`/`undefined`，使「不再挂载」断言与生产 survey 路由定义解耦，误恢复 push 时测试仍绿；回归防护弱化，非本次功能缺陷。
2. **Minor** — 出处：`frontend/tests/unit/router/router-guard.spec.ts`（diff 行 267-281，survey mock 段）。survey 默认导出置空后，`/s/:key` 跳登录断言依赖的是 mock 的路由表而非生产 `routers.ts`，若生产误恢复公开路由该用例仍通过；同上属回归防护弱化。

Critical：0；Important：0；Minor：2。

## 总体结论

**通过。** 两条 SQL UPDATE 列名与真实 DDL 一致、幂等、可回滚且无破坏性语句；四组 survey 静态路由与 `surveyRouters` 均已摘除，`/s/:key` 随路由移除自动失去 public 放行；测试真实断言了「不再挂载」与「无 token 跳登录」。两项 Minor 为回归防护层面的测试设计问题，不阻塞本次验收，建议记入 deferred。
