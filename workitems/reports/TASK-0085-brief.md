# 实施 brief：TASK-0085（BUG-0006 微改——补 operator 角色 iot:tenant:admin 授权）

> 微改（单模块、无接口/数据模型/业务规则变化）。BASE=f928df9。**跳过 diff 评审**（L1-execute §13 微改档），仅验收命令。续用 TASK-0004 执行者上下文（agent_615cc6d1）。

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0085.md`（含风险核对三行）
- BUG 详情：`bugs/BUG-0006.md`（现象/根因/复现/修复方案）
- 首评发现：`reports/TASK-0004-review.md` F-1（Minor deferred，本微改闭合）
- 缺口权威证据：
  - `resources/sql/upgrade/20261124_operator_isolation.sql`（当前 operator 授权段：4000-4007/171/173/2/3/4，**缺 4017**）
  - `resources/sql/upgrade/20261121_tenant_id_columns.sql:53/55`（menu 4017=iot:tenant:admin，历史上仅授 role 1）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/SysTenantController.java:103-104`（`@SaCheckPermission("iot:tenant:admin")` on POST /{id}/init-admin）
- 守卫/鉴权链（不改，只测）：`backend/.../system/config/OperatorIsolationGuard.java` + `SaTokenInterceptorConfig.java` + SaInterceptor（注解鉴权）

## 接口与决策（逐条结论）

1. **补授权**：在 operator 授权脚本补幂等一行 `INSERT INTO sys_role_menu (role_id, menu_id) SELECT 3, 4017 WHERE NOT EXISTS (SELECT 1 FROM sys_role_menu WHERE role_id=3 AND menu_id=4017)`。
   - **脚本选择**：优先补进原 `20261124_operator_isolation.sql`（同一批尚未被用户应用到真库——DDL 全局只产出未执行，报告已声明）；若判断原脚本可能已被部分环境应用，则新建 `20261124_operator_tenant_admin_grant.sql` 幂等脚本更稳妥——**执行者自决并在报告说明理由**，两种都必须幂等。
2. **真实注解链测试**：新增 backend 测试，用**真实 SysTenantController**（非 dummy）验证 operator 账号经「守卫（路径放行）→ SaInterceptor（@SaCheckPermission("iot:tenant:admin") 注解鉴权）」双层后 `POST /iot/tenant/{id}/init-admin` 可达控制器逻辑（断言非 403；若 service 依赖重可用 mock service 但仍走真实控制器 + 真实注解鉴权链）。**关键**：构造"未补授权版本"该测试必须 403（红），补授权后 200/非403（绿）——TDD 红转绿留痕。
   - 补一条 SQL 幂等断言：operator 菜单授权数由 13→14（含 4017），H2 MODE=MySQL 二次执行计数不变。
3. **不误伤**：不改守卫白名单、不改 operator 角色语义、不改其他角色授权、不改 AlarmConfig。

## 验收命令（必须原样贴回）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（含新注解链测试 + SQL 幂等断言），贴 Results 段
2. SQL 幂等：H2 MODE=MySQL 或静态断言，**不在用户 phpStudy MySQL 执行**（报告声明）
3. 范围核对：`git diff --name-only f928df9..HEAD` 仅落 SQL 授权脚本 + 测试文件

## 报告

报告写到：`reports\TASK-0085-report.md`。五项齐全 + 红转绿证据（未授权必红/补后绿）+ SQL 幂等输出 + 脚本选择理由。微改跳过 diff 评审，报告注明。

## 契约

- no-subagents；只改本批范围；不置 TASK 已完成/不改 BUG 状态（由协调者收尾）；不改 workitems 索引；不在用户库执行 DDL。
- 授权幂等（WHERE NOT EXISTS）；测试走真实控制器注解链不用 dummy；不硬编码凭据/文档库路径。
