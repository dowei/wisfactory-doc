# 实施 brief：TASK-0004（运营方硬隔离与审计埋点）

> 标准任务（backend 权限隔离 + SQL 种子）。BASE=57aeb36。一个执行者做完整、统一测一次。前置 0002✅。FEAT-0001 最后一块（其余 0001/0002/0003/0072/0075/0077/0079 全完成）。

## 批次

- TASK-0004（本批 1 个）
- 验收粒度：一个 subagent 跑一次整体验收（backend mvn test）
- BASE：`57aeb36`
- 改动范围（文件/目录级，越界反查用）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/system/**`（SaToken 配置挂运营方守卫、运营方角色判定工具、拦截器）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/system/**`
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（如越权红线测试需扩，仅新增测试文件）
  - `resources/sql/upgrade/20261124_operator_isolation.sql`（运营方角色种子 + 营养菜单授权 + 健康用量白名单菜单，**只产出不执行**）
  - **不得触碰**：frontend/**、ingest/**、calc/**、resources/contract/**（冻结）、iot/inspect|alarm|calcread|dataitem 既有业务代码（只读复用其健康用量端点作白名单，不改）、既有 system 业务类除必要守卫挂载外、framework upstreams

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0004.md`
- 关联对象：`features/FEAT-0001.md`、`requirements/REQ-0001.md`（**验收 6：运营方直调任一租户业务数据接口 403**）、`testcases/TC-0001.md`、`decisions/ADR-0011.md`（TD-0040/0042 定版）
- 拦截器挂载点：`backend/ot-admin/src/main/java/net/otlab/admin/module/system/config/SaTokenInterceptorConfig.java`（addInterceptors 现有两个拦截器 + buildExcludePaths）
- 租户/用户上下文：`backend/ot-core/src/main/java/net/otlab/core/tenant/TenantContext.java`、`.../utils/SecurityUtils.java`（hasRole）
- 审计事件（已有，勿重造）：`backend/ot-core/src/main/java/net/otlab/core/event/PermissionDeniedEvent.java` + `backend/ot-admin/src/main/java/net/otlab/admin/module/system/listener/PermissionDeniedAuditListener.java`（BUG-0044 范式，越权在控制器前拦截时由鉴权层发事件落库）
- 白名单数据源（运营方可看的健康/用量，只读复用）：`backend/.../iot/inspect/controller/IotInspectController.java`（GET /dashboard|history|latest）、`.../iot/calcread/controller/IotCalcReadController.java`（GET /state|log）；租户管理页 `.../iot/controller/SysTenantController.java`
- 角色种子范式：`resources/sql/01_base.sql:561-562`（sys_role 1 admin / 2 user）、`resources/sql/upgrade/20261124_iot_alarm.sql`（菜单幂等 WHERE NOT EXISTS）
- 越权审计守卫测试范式：`backend/.../iot/dataitem/config/IotDataItemMenuPermissionGuardTest.java`（反射扫描 @RestController @SaCheckPermission）

## 接口与决策（逐条结论，无选项）

### D1 · 运营方角色与判定
- 新增运营方角色种子：`sys_role` 加 `role_code='operator'`、`role_name='运营方'`、`data_scope` 合适值；与告警通知目标角色 `platform_admin`（`AlarmConfig` 默认）语义对齐——**统一为 role_code='operator'**，若担心影响既有 platform_admin 配置，可在种子里加别名说明但不改 AlarmConfig（报告登记口径）。
- 运营方判定：`SecurityUtils.hasRole("operator")`（SaToken 会话角色）。角色种子 ID 避开 1/2（如 3），幂等 INSERT（WHERE NOT EXISTS by role_code）。

### D2 · 全局硬隔离守卫（核心，REQ-0001 验收 6）
- 在 `SaTokenInterceptorConfig` 的登录拦截器之后（或同一拦截器内）挂**运营方业务数据隔离守卫**：
  - 若当前登录者 `hasRole("operator")`：仅放行**白名单路径**，其余路径一律 `StpUtil.checkRole` 失败式拒绝 → 403。
  - **白名单**（运营方可访问）：
    1. 健康/用量：`/api/v1/iot/inspect/**`（GET）、`/api/v1/iot/calc/state|log`（GET）
    2. 租户管理：`/iot/tenant/**`、`/sys/tenant/**`（按实际 SysTenantController 路径，运营方需开户/启停/配额）
    3. 系统基础：`/sys/user/**`、`/sys/role/**`、`/sys/menu/**`、`/login/**`、`/logout`、`/getInfo`、`/sys/profile/**`（运营方账户自管理）
    4. 框架匿名/公共路径沿用 buildExcludePaths，不受影响
  - **拒绝范围**：所有 iot 业务配置/数据接口（instance/item/point/dataitem/metric/channel/gateway/template/publish/statusread/alarm/... 的增删改查）、openapi 管理、inspect/alarm 的写口（确认/处理是租户管理员职责，运营方只读健康）——以**路径白名单制**实现（默认拒绝，非枚举黑名单），保证新增业务接口自动受控。
  - 白名单用**精确路径 + 前缀 + HTTP 方法**组合，避免通配过宽（如 `/api/v1/iot/inspect/**` 仅 GET；`/iot/tenant/**` 按实际方法）；报告列全白名单清单并说明逐条理由。
  - 守卫必须在**控制器之前**生效（拦截器 preHandle），运营方直调业务接口返回 403（code=403，与既有越权口径一致）。
- **不得误伤**：超级管理员 admin（role 1）与租户管理员（普通 user/tenant_admin）不受此守卫影响——守卫只对 operator 角色生效。用测试锁定三类角色行为。

### D3 · 审计埋点
- 运营方越权被拒时，发 `PermissionDeniedEvent`（复用 BUG-0044 范式，GlobalExceptionHandler 已对 NotRole/NotPermissionException 发布）；若守卫用自定义异常/直接 response，需显式 publish 事件确保留痕。
- 核实 `PermissionDeniedAuditListener` 能把事件落库（既有实现，不改），守卫触发后审计表有记录。报告给证据。

### D4 · 测试口径（越权红线）
- TDD 新增契约/集成测试（MockMvc 或反射）：
  1. 运营方 token 直调**每个**代表性业务接口（instance/item/point/dataitem/metric/channel/gateway/template/publish/status/alarm 各取一）→ 403，审计事件发布；
  2. 运营方调白名单健康端点（inspect/dashboard、calc/state）→ 200；
  3. 租户管理员/超管调业务接口 → 不被守卫误伤（仍按各自 @SaCheckPermission 口径）；
  4. 角色种子 SQL 二次运行幂等（CREATED/EXISTS 口径）。
- **全局覆盖守卫**：扩展类似 IotDataItemMenuPermissionGuardTest 的反射扫描，确认新增业务接口未被白名单误纳入（或白名单精确可控）。

## 验收命令（必须原样贴回输出）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（越权红线全套 + 白名单 + 审计 + 幂等有测试），贴 Results 段
2. SQL 二次运行幂等验证（按 sql-exec.sh 口径，只验证脚本幂等逻辑，**不在用户 phpStudy MySQL 执行 DDL**——若需真库验证用 H2 或静态断言，报告说明）
3. 范围核对：`git diff --name-only 57aeb36..HEAD` 全落改动范围声明；frontend/ingest/calc/contract/既有 iot 业务模块零触碰

## 报告

报告写到：`reports\TASK-0004-report.md`。五项齐全 + 一句话结论 + TDD 红转绿 + DDL 未执行声明 + **白名单完整清单与逐条理由** + 越权红线全绿证据 + 审计落库证据。

## 契约

- no-subagents；只改本批改动范围；不置 TASK 已完成；不改 workitems 索引；DDL 只产出不执行。
- **默认拒绝**：运营方只能看健康/用量与租户管理，其余业务接口一律 403；白名单制而非黑名单。
- 不误伤 admin/租户管理员；审计埋点复用既有 PermissionDeniedEvent，不重造。
- 不硬编码凭据/文档库路径；越权 403 文案不泄露资源标识（与既有 0061/0062 口径一致）。
- 测试失败先定位根因，禁止为匹配输出改测试/改期望。
