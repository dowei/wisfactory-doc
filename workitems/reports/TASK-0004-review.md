# TASK-0004 代码评审报告（首评）

- 评审对象：提交 `f928df9`（BASE=`57aeb36`，区间干净单提交，9 文件，986 行 diff 包）
- 评审口径：L1-execute §13「diff 评审」；fresh / no-subagents / 只评审不修改（仅本报告）
- 评审日期：2026-11-24
- 契约：标准任务首评（1 退回 + 1 复评上限）；C/I → 退回执行者一轮，Minor → deferred

---

## 一、结论

**通过（PASS）**。无 Critical / Important 级问题；发现 1 项 Minor（deferred，见 §三）。白名单安全边界经逐条源码比对（非仅凭测试绿灯）判定准确：默认拒绝成立、实际控制器路径与白名单完全一致、无过宽通配误放行业务接口、无过窄导致运营方登录/getInfo 被误拒、新增业务接口自动受控。守卫挂载、403 口径、审计复用、角色种子幂等、范围约束均核实通过。

---

## 二、头号重点：白名单准确性（安全边界）逐条独立判定

不依赖测试结论，直接比对 `OperatorIsolationPolicy` 与真实控制器 `@RequestMapping` / 方法映射，并核对 `StpUtil.hasRole("operator")` 的真实取值来源。

### 2.0 角色判定链路（守卫是否真的只对 operator 生效）

- `OperatorIsolationGuard` 用 `StpUtil.hasRole("operator")` → `SaTokenConfig.getRoleList`（`.../system/config/SaTokenConfig.java:36-39`）→ `SysUserServiceImpl.getRoleListByUserId`（`:647-670`）→ 返回 `SysRole::getRoleCode` 列表。
- 因此 `operator` 精确匹配种子 `role_code='operator'`（非 role_name「运营方」）。守卫实际采用 `StpUtil.hasRole`（`OperatorIsolationGuard.java:34`），与 brief 所述 `SecurityUtils.hasRole` 等价（后者仅委托 `StpUtil.hasRole`，`SecurityUtils.java:36-38`）。**判定：正确。**

### 2.1 逐条比对（白名单项 vs 真实路由）

| 策略项 | 真实控制器源码 | 比对结论 |
| --- | --- | --- |
| GET `/api/v1/iot/inspect/dashboard\|history\|latest`（精确集、仅 GET） | `IotInspectController.java:33` 类级 `@RequestMapping("/api/v1/iot/inspect")`；`:43/:50/:61` 三个 `@GetMapping("/dashboard"/"/history"/"/latest")`，均 `@SaCheckPermission("iot:instance:query")`，**零写口** | 完全一致；精确集非宽前缀，`/api/v1/iot/inspect/*` 其他子路径默认拒绝；仅 GET，写方法拒绝 |
| GET `/api/v1/iot/calc/state\|log`（精确集、仅 GET） | `IotCalcReadController.java:37` 类级 `/api/v1/iot/calc`；`:47/:54` 两个 `@GetMapping("/state"/"/log")`，零写口 | 完全一致；`/api/v1/iot/calc/state/extra`、`/api/v1/iot/calc` 均不命中 |
| `/iot/tenant/` 前缀全方法 + 根路径 `POST`/`PUT` | `SysTenantController.java:33` 类级 `/iot/tenant`；方法：`:40 GET /list`、`:50 GET /{id}`、`:58 POST(根)`、`:67 PUT(根)`、`:76 PUT /{id}/status`、`:85 DELETE /{ids}`、`:103 POST /{id}/init-admin` | 全部真实路由被覆盖；前缀带尾斜杠故 `/iot/tenantx` 不误放行；GET 根无映射且被正确拒绝（GREEN-2 已修正 PUT 根路径） |
| `/user/` `/role/` `/menu/` `/profile/` `/auth/` 前缀 + 各自根路径 | `SysUserController.java:48` `/user`；`SysRoleController.java:39` `/role`；`SysMenuController.java:26` `/menu`；`SysProfileController.java:35` `/profile`；`LoginController.java:57` `/auth` | 全部为真实类级 `@RequestMapping`；各方法均落在对应前缀下；无类级/方法级多路径映射（全仓 `@RequestMapping({...})` 为 0） |

### 2.2 过宽（误放行业务接口）判定：**无**

- 全仓 `@RestController` 类级映射中，命中白名单前缀的**仅有**上述 5 个 system 控制器 + `SysTenantController`；无第二个控制器落在 `/user`、`/role`、`/menu`、`/profile`、`/auth`、`/iot/tenant/` 下（逐一 grep 全 backend main 确认）。
- 所有 iot 业务接口挂载在互不相交前缀：`/iot/instance`、`/iot/item`、`/iot/point`、`/iot/metric`、`/iot/channel`、`/iot/gateway`、`/iot/template`、`/iot/publish`、`/iot/statusdict`、`/iot/statusrule`、`/iot/dict`、`/api/v1/iot/alarm`、`/api/v1/iot/status`、`/openapi/*` 等——**无一命中白名单前缀**。
- 前缀 `/user/` 带尾斜杠，`/userProfile` 之类不会误命中；`/iot/tenant/` 同理不误放 `/iot/tenantx`。
- 健康端点用精确 `Set.of(...)` + 仅 GET，非 `/api/v1/iot/**` 宽前缀。

### 2.3 过窄（误拒登录/首页/getInfo）判定：**无 401/403 循环**

- 登录链路：`/auth/login`、`/auth/logout`、`/auth/captchaImage`、`/auth/login/test` 均为 `@Anonymous`（`LoginController.java:104/188/228/251`），未登录时守卫 `!StpUtil.isLogin()` 直接放行；`/auth/info`（`:206`，登录态）由 `/auth/` 前缀放行。
- **前端实际 getInfo 路径核对（brief 点名的误拒风险）**：`frontend/src/api/system/user-api.ts:62` 的 `getInfo()` 实际请求 **`/user/info`**，映射 `SysUserController.java:145 @GetMapping("/info")`，**无 `@SaCheckPermission`**，由 `/user/` 前缀放行。启动引导的菜单/模块/权限均随 `/user/info` 一次返回（`SysUserController.java:156-167`），不依赖 `/module/*` 等未白名单接口。
- brief 示例的 `/getInfo`、`/login`、`/logout`、`/sys/*` 在本工程**无对应映射**（全仓 grep 确认），执行者按实际路径实现属正确纠偏，不构成过窄。
- `buildExcludePaths()`（`/favicon.ico`、`/error`，dev 追加文档路径）在三个拦截器上保持一致；守卫脚本扫描采用的是真实配置，无遗漏。

### 2.4 新增接口自动受控判定：**成立**

- 策略为 default-deny（`isAllowed` 末尾 `return false`），新增 iot 业务接口未登记即 403，无需维护黑名单。
- `OperatorIsolationGlobalGuardTest` 反射扫描 `net.otlab.admin.module.iot` 与 `...openapi` 全部 `@RestController`（实测 **27 个控制器 / 127 条方法映射**，断言阈值 ≥15/≥40 真实且非恒真），确认放行集合恰为文档白名单子集，`/iot/instance/list`、`/api/v1/iot/alarm/list` 确认不在放行集合。

**第 1 点总判定：白名单准确、默认拒绝安全边界成立，无 C/I 级问题。**

---

## 三、发现清单

| # | 严重级 | 出处 | 一句话说明 |
| --- | --- | --- | --- |
| 1 | Minor（deferred） | `resources/sql/upgrade/20261124_operator_isolation.sql:20-35`（仅授予菜单 4000-4007、171、173、2/3/4）对比 `SysTenantController.java:103-104` 的 `@SaCheckPermission("iot:tenant:admin")` 与菜单 4017（`20261121_tenant_id_columns.sql:53`，历史上仅授予 role 1） | 租户管理白名单声称覆盖「初始化管理员」全链路，但 operator 角色未获 `iot:tenant:admin`（菜单 4017）权限，守卫放行后仍会被下游 SaInterceptor 以 403 拒绝，运营方无法经此接口完成 REQ-0001 验收 1 的「初始化管理员」动作——fail-closed、非安全越界，属种子授权完整性缺口，建议后续补 `sys_role_menu(3,4017)`。 |

> 说明：该 Minor 不影响本任务核心验收 6（运营方直调业务数据 403），不改变安全边界结论，按契约 deferred，不触发退回。测试用 dummy 控制器无 `@SaCheckPermission`，故该链路缺口被绿灯掩盖——建议后续集成测试补一条「operator 真实注解链」用例。

---

## 四、其余评审重点核实

1. **守卫挂载与不误伤**（`SaTokenInterceptorConfig.java:52-107` 实读）：拦截器顺序 = 登录校验/账号状态复核 → `OperatorIsolationGuard` → `SaInterceptor`，均 `/**` + 同款 excludePaths，守卫 `preHandle` 早于控制器。守卫仅 `hasRole("operator")` 介入（`:34`），admin/租户管理员直接放行。`OperatorIsolationGuardTest` 三类角色用例（admin=`"admin"`、租户管理员=`"user","tenant_admin"`）实跑通过。**通过。**
2. **403 口径与不泄露资源标识**：守卫抛 `NotRoleException("operator:business-isolation")`（策略常量，非路径），`GlobalExceptionHandler.java:133-137` → `ResultCode.FORBIDDEN(403)` + 固定文案「无权访问该资源」，与 0061/0062 口径一致。审计 `failReason` 含角色码与异常信息但不含被请求资源路径对外返回。**通过。**
3. **审计复用（未重造）**：守卫抛 Sa-Token 原生异常 → `GlobalExceptionHandler.publishPermissionDenied`（`:143-175`）构建 `PermissionDeniedEvent(requiredCodes=[role], loginId, uri, method, ip, ua, ...)` → `publishEvent` → 既有 `PermissionDeniedAuditListener.onPermissionDenied`（零改动）→ `OperateLogService.saveOperateLog`。测试 `deniedEventPersistedByExistingListener` 用真实 Listener + mock 服务断言 `status=0 / module=权限校验 / url / method / failReason 含 role 码 / createTime`。**通过。**
4. **角色种子与幂等**：`role_code='operator'`、`id=3`（避开 1/2）、`data_scope=1`、`WHERE NOT EXISTS by role_code` 幂等；菜单授权 4000-4007 + 171 + 173 + 2/3/4 = 13 条，`sys_menu 173` 按 id 判存幂等。H2 `MODE=MySQL` 两次执行计数稳定（1 / 13 / 1），与断言一致。platform_admin 别名：`AlarmConfig.java:31` 默认仍 `platform_admin` 未改，脚本头已登记别名、报告 §8 说明部署时设 `ot.alarm.notice-target-role=operator`。**通过。**
5. **DDL 未执行**：脚本头明确「只产出、不在用户库执行」；git 仅新增脚本文件，无对 phpStudy MySQL 的执行痕迹；幂等由 H2 内存库 + 静态断言验证。**通过。**
6. **测试有效性与零回归**：独立复跑 5 个新增测试类全部绿（Policy 10 + Guard 7 + GlobalGuard 1 + Mount 1 + RoleSeedSql 5 = 24，0F0E）；执行者报告全量 `mvn -pl ot-admin test` 2558/2558（2534 基线 + 24），与新增数吻合。TDD 红转绿留痕（GREEN-1 Result 信封 / GREEN-2 PUT 根漏白 / GREEN-3 H2 计数 13）属实且「为什么现在对了」合理。**通过。**
7. **范围与越界反查**：`git diff --name-only 57aeb36..HEAD` 与 diff 包一致，恰 9 文件（3 main + 5 test + 1 SQL），全落声明范围；frontend/ingest/calc/contract 及既有 iot 业务实现零触碰；工作副本 `.agents/`、`.tmp-t0044v/` 为任务前已存在未跟踪目录，未纳入提交。**通过。**

---

## 五、契约遵守

- 首评口径执行：1 项 Minor 记 deferred，未触发退回；无 C/I。
- 评审仅覆盖隔离的 9 文件 diff（f928df9 单提交），区间干净，未改代码、未推进状态、未派子代理。
- 报告位置：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0004-review.md`。

**结论：通过（1 项 Minor deferred，不阻塞）。**
