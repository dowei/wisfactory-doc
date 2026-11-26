# TASK-0004 执行报告：运营方硬隔离与审计埋点

- BASE：`57aeb36`
- HEAD（本任务提交）：`f928df9b6cc201469dd048274b5bf1f565e3fa51`（短 `f928df9`）
- 执行者：单执行者（no-subagents）
- TASK 状态：保持「进行中」（按契约不置已完成、不改索引）
- 一句话结论：运营方角色种子 + preHandle 白名单守卫（默认拒绝）+ 既有 PermissionDeniedEvent 审计链路全部落地，2558 项 ot-admin 单测全绿，越权红线/白名单/不误伤/审计落库/SQL 幂等均有实测证据；DDL 只产出未执行。

---

## 0 · 接口与决策（D1-D4）执行对照

| 决策 | 要求 | 落地 |
| --- | --- | --- |
| D1 | 角色种子 role_code='operator'、id 避开 1/2、幂等 INSERT；platform_admin 语义对齐不改 AlarmConfig | `resources/sql/upgrade/20261124_operator_isolation.sql`：id=3 / '运营方' / data_scope=1，`WHERE NOT EXISTS (role_code='operator')`；脚本头登记 platform_admin 别名口径，未改 AlarmConfig |
| D2 | 全局硬隔离守卫，preHandle，默认拒绝白名单制，403 code=403 文案不泄露资源标识 | `OperatorIsolationGuard` 在 `SaTokenInterceptorConfig` 登录校验后、SaInterceptor 前挂载；抛 `NotRoleException("operator:business-isolation")`，由 GlobalExceptionHandler 统一 403 + 「无权访问该资源」 |
| D3 | 复用既有 PermissionDeniedEvent + PermissionDeniedAuditListener，不重造 | 守卫抛 Sa-Token 原生异常，GlobalExceptionHandler 发布事件（BUG-0044 范式），既有 Listener 落 sys_operate_log；测试实锚验证 |
| D4 | TDD 红转绿；越权红线全套；反射扫描全局守卫；SQL 二次运行幂等 | 5 个测试类、24 个用例；反射扫描 iot/openapi 全部 @RestController；H2 真执行脚本两次验幂等 |

### 路径口径对齐（重要偏差登记）

brief D2 白名单示例使用 `/sys/user/**`、`/sys/role/**`、`/login/**`、`/getInfo` 等前缀；**工程实际控制器路径**为：

- `SysUserController` → `/user/**`
- `SysRoleController` → `/role/**`
- `SysMenuController` → `/menu/**`
- `SysProfileController` → `/profile/**`
- `LoginController` → `/auth/login|logout|info`（登录/登出本就是 @Anonymous，由既有 buildExcludePaths / 注解跳过，守卫不介入）
- `SysTenantController` → `/iot/tenant/**`（brief 已授权「按实际路径」）

按 brief「白名单按实际 SysTenantController 路径」的同一原则，全部白名单按**真实控制器路径**实现，并在下方清单逐条标注。若将来前端统一加 `/sys` 网关前缀，由网关在转发前剥除（contextPath 已在守卫中剥除处理）。

---

## 1 · 改动文件清单（9 个，全部落声明范围）

| 文件 | 性质 |
| --- | --- |
| `D:/workspace/WisFactory/backend/ot-admin/src/main/java/net/otlab/admin/module/system/config/SaTokenInterceptorConfig.java` | 改：挂载守卫（1 处 insert） |
| `D:/workspace/WisFactory/backend/ot-admin/src/main/java/net/otlab/admin/module/system/config/OperatorIsolationPolicy.java` | 新：白名单策略（默认拒绝） |
| `D:/workspace/WisFactory/backend/ot-admin/src/main/java/net/otlab/admin/module/system/config/OperatorIsolationGuard.java` | 新：preHandle 守卫 |
| `D:/workspace/WisFactory/backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorIsolationPolicyTest.java` | 新：白名单/拒绝单测（10 例） |
| `D:/workspace/WisFactory/backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorIsolationGuardTest.java` | 新：MockMvc 越权红线/白名单/不误伤/审计（7 例） |
| `D:/workspace/WisFactory/backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorIsolationGlobalGuardTest.java` | 新：反射扫描全局守卫（1 例） |
| `D:/workspace/WisFactory/backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorIsolationMountTest.java` | 新：挂载点契约（1 例） |
| `D:/workspace/WisFactory/backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorRoleSeedSqlTest.java` | 新：SQL 静态+H2 幂等（5 例） |
| `D:/workspace/WisFactory/resources/sql/upgrade/20261124_operator_isolation.sql` | 新：角色种子 + 菜单授权（只产出不执行） |

`git diff --name-only 57aeb36..HEAD` 全量即上表 9 项；frontend / ingest / calc / resources/contract / iot/inspect|alarm|calcread|dataitem 既有业务实现代码零触碰（只读复用健康端点路径与菜单 ID）。

---

## 2 · 白名单完整清单与逐条理由

守卫唯一放行入口：`OperatorIsolationPolicy.isAllowed(method, path)`。**凡未列入下表，一律 403（默认拒绝，非黑名单）**。

| # | HTTP 方法 | 路径（实际控制器路径） | 放行理由 |
| --- | --- | --- | --- |
| 1 | GET | `/api/v1/iot/inspect/dashboard` | 租户健康看板只读聚合（FEAT-0020/IotInspectController），运营方查看租户健康 |
| 2 | GET | `/api/v1/iot/inspect/history` | 健康巡检历史回看，只读 |
| 3 | GET | `/api/v1/iot/inspect/latest` | 最新巡检清单，只读 |
| 4 | GET | `/api/v1/iot/calc/state` | 用量/计算状态只读（IotCalcReadController） |
| 5 | GET | `/api/v1/iot/calc/log` | 用量计算日志只读 |
| 6 | POST | `/iot/tenant` | 运营方新建租户（REQ-0001 一期运营方手工开户） |
| 7 | PUT | `/iot/tenant` | 运营方编辑租户主数据 |
| 8 | GET/PUT/POST/DELETE | `/iot/tenant/{id|ids}/**`（list / {id} / {id}/status / {id}/init-admin / {ids}） | 租户查询/详情/启停/初始化管理员/删除，运营方租户全生命周期管理（SysTenantController 实际挂载） |
| 9 | 全部方法 | `/user/**`（含 `/user`） | 运营方账户自管理：用户列表/详情/新增/改密/角色分配 |
| 10 | 全部方法 | `/role/**`（含 `/role`） | 运营方角色管理（自管理所需最小集） |
| 11 | 全部方法 | `/menu/**`（含 `/menu`） | 运营方菜单/权限查看与分配 |
| 12 | 全部方法 | `/profile/**` 与 `/profile` | 运营方个人中心（资料/头像/密码） |
| 13 | 全部方法 | `/auth/**` | 认证入口（login/logout/info/captchaImage）；login/logout/captcha 本就 @Anonymous，info 为登录态自信息 |
| — | 任意 | 其他所有路径（框架匿名/公共路径除外） | **默认拒绝**：iot 业务配置/数据（instance/item/point/dataitem/metric/channel/gateway/template/publish/status/alarm）、openapi 管理、健康端点写口等一律 403 |

白名单精确性控制：
- 健康端点用**精确路径集合**，不是 `/api/v1/iot/**` 宽前缀；`/api/v1/iot/calc/state/extra`、`/api/v1/iot/calc`（无子路径）等均不命中。
- 健康端点仅 GET；POST/PUT/DELETE 一概拒绝（巡检写口/告警确认归租户管理员）。
- 租户管理限 `/iot/tenant` 根与 `/iot/tenant/` 前缀；`/iot/tenantx` 不命中。
- 反射全局守卫扫描 iot/openapi 所有 @RestController 的实际路由，确认「被白名单放行的路由」恰好落在上述清单，新增业务接口自动落默认拒绝侧。

---

## 3 · 验收命令原始输出（原样贴回）

### 3.1 `cd backend && mvn -pl ot-admin test`（brief 验收命令 1）

```
[INFO] Running net.otlab.admin.module.system.config.OperatorIsolationGlobalGuardTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.229 s -- in net.otlab.admin.module.system.config.OperatorIsolationGlobalGuardTest
[INFO] Running net.otlab.admin.module.system.config.OperatorIsolationGuardTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.140 s -- in net.otlab.admin.module.system.config.OperatorIsolationGuardTest
[INFO] Running net.otlab.admin.module.system.config.OperatorIsolationMountTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.003 s -- in net.otlab.admin.module.system.config.OperatorIsolationMountTest
[INFO] Running net.otlab.admin.module.system.config.OperatorIsolationPolicyTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.006 s -- in net.otlab.admin.module.system.config.OperatorIsolationPolicyTest
[INFO] Running net.otlab.admin.module.system.config.OperatorRoleSeedSqlTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.257 s -- in net.otlab.admin.module.system.config.OperatorRoleSeedSqlTest
...
[INFO] Results:
[INFO] 
[INFO] Tests run: 2558, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:08 min
```

零失败、零错误。README 同义命令 `mvn test -pl ot-admin -am` 未另跑（brief 指定命令为准；ot-core 已随依赖参与编译，2558 覆盖 core 测试）。

### 3.2 SQL 二次运行幂等（brief 验收命令 2）

不在用户 phpStudy MySQL（8.0.12@3307）执行 DDL。用两重验证：

**（a）静态断言**（`OperatorRoleSeedSqlTest`，全绿）：
- `WHERE NOT EXISTS` 存在且判存条件为 `role_code = 'operator'`
- 角色种子 + 菜单授权至少两条幂等 INSERT
- 每条语句独占一行，兼容 sql-exec.sh 按 `;\n` 切分
- 脚本头含「只产出、不在用户库执行」声明

**（b）H2 真执行**（`scriptIdempotentOnH2`，1 例绿）：
```
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- in OperatorRoleSeedSqlTest
```
H2 `MODE=MySQL` 最小夹具（sys_role/sys_menu/sys_role_menu），按 sql-exec.sh 口径切分全脚本连续执行两次：
- 第 1 次（CREATED）：`sys_role WHERE role_code='operator'` = 1 行；`sys_role_menu WHERE role_id=3` = 13 行；`sys_menu id=173` = 1 行
- 第 2 次（EXISTS）：三项计数均不变（1 / 13 / 1），无重复插入、无主键冲突

### 3.3 范围核对（brief 验收命令 3）

```
$ git diff --name-only 57aeb36..HEAD
backend/ot-admin/src/main/java/net/otlab/admin/module/system/config/OperatorIsolationGuard.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/config/OperatorIsolationPolicy.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/config/SaTokenInterceptorConfig.java
backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorIsolationGlobalGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorIsolationGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorIsolationMountTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorIsolationPolicyTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorRoleSeedSqlTest.java
resources/sql/upgrade/20261124_operator_isolation.sql
```

9 个文件，全部在 brief 改动范围内；frontend / ingest / calc / resources/contract / iot 既有业务实现零触碰。工作副本中 `.agents/`、`.tmp-t0044v/` 为任务开始前已存在的未跟踪目录，未纳入本次提交、未改动。

---

## 4 · TDD 红转绿留痕（每修复附「为什么现在对了」）

| 轮次 | 动作 | 结果 / 证据 |
| --- | --- | --- |
| RED | 先写 3 类测试：`OperatorIsolationPolicyTest`（10）、`OperatorIsolationGuardTest`（7）、`OperatorRoleSeedSqlTest`（4） | `mvn -pl ot-admin -Dtest=... test` 退出码 1，编译失败：`找不到符号 OperatorIsolationPolicy / OperatorIsolationGuard`，脚本文件不存在 → 证明测试先行、非事后补写 |
| GREEN-1 | 实现 Policy / Guard / 挂载 / SQL 后重跑 | 21 例中 17 通过、4 失败：白名单与不误伤用例报 `No value at JSON path "$.code"` |
| 修复 1 | **为什么现在对了**：失败根因是测试夹具的 dummy 控制器返回裸字符串 `"ok"`，而生产控制器统一返回 `Result` 信封（`$.code`）。把 dummy 返回改为 `Result.success()`，与生产响应契约一致——不是为匹配输出改断言，而是修夹具使被测链路真实。 | 重跑：21/21 通过，`OperatorIsolationGuardTest` 7 例全绿 |
| GREEN-2 | 补全局覆盖守卫反射扫描 + 挂载点契约 | 发现 `SysTenantController` 根路径 `PUT /iot/tenant` 未纳入白名单 |
| 修复 2 | **为什么现在对了**：旧策略只放行根路径 POST；而租户编辑实际映射为根路径 PUT。把白名单改为「POST/PUT 根路径 + /iot/tenant/** 前缀全方法」，与控制器真实路由一一对应，既不放宽到 GET 根（无映射），也不误拦编辑接口。 | `OperatorIsolationGlobalGuardTest` 1 例绿，扫描 iot/openapi 全部 @RestController 无意外放行 |
| GREEN-3 | 补 SQL H2 真执行幂等 + 全量验收 | H2 首轮期望授权数写错为 14 |
| 修复 3 | **为什么现在对了**：逐条数脚本授权＝4000-4007 八项 + 171 + 173 + 2/3/4 三项 = 13。断言改为 13 与脚本真实授权集一致，不是改实现迁就测试。 | H2 测试 5/5 绿；全量 `mvn -pl ot-admin test` 2558/2558 绿 |

---

## 5 · 越权红线全绿证据

`OperatorIsolationGuardTest`（7 例全绿）关键断言：

1. **11 类业务代表接口 403**：GET `/iot/instance/list`、`/iot/item/list`、`/iot/point/list`、`/api/v1/iot/dataitem/history`、`/iot/metric/list`、`/iot/channel/list`、`/iot/gateway/list`、`/iot/template/list`、`/iot/publish/list`、`/api/v1/iot/status/current`、`/api/v1/iot/alarm/list` → 每个 `$.code == 403`、`$.message == "无权访问该资源"`（不泄露资源标识）。
2. **写接口 403**：POST `/iot/instance`、PUT `/iot/point/1`、DELETE `/iot/gateway/1`、POST `/api/v1/iot/alarm/1/confirm` → 403。
3. **白名单 200**：GET `/api/v1/iot/inspect/dashboard`、GET `/api/v1/iot/calc/state`、GET `/iot/tenant/list`、POST `/iot/tenant`、GET `/user/list`、GET `/auth/info` → `$.code == 200`，且 `publishEvent` 从未调用（无误拒）。
4. **不误伤 admin（role 'admin'）**：GET `/iot/instance/list` → 200（守卫不介入）。
5. **不误伤租户管理员（role 'user','tenant_admin'）**：同上 → 200。
6. **审计事件发布**：11 次越权每次均 `publishEvent(PermissionDeniedEvent)`；事件 `requestUri`、`httpMethod='GET'`、`loginId=10`、`requiredCodes=['operator:business-isolation']` 断言通过。
7. **全局反射扫描**：`OperatorIsolationGlobalGuardTest` 扫描 iot/openapi 全部 @RestController（≥15 个控制器、≥40 条路由），被放行路由恰好为文档白名单子集；`/iot/instance/list`、`/api/v1/iot/alarm/list` 确认不在放行集合。

**审计落库证据**（`deniedEventPersistedByExistingListener`）：守卫事件 → 既有 `PermissionDeniedAuditListener.onPermissionDenied()` → `OperateLogService.saveOperateLog()` 被调用一次，捕获的 `SysOperateLog`：`status=0`、`module='权限校验'`、`url='/iot/instance/list'`、`requestMethod='GET'`、`content` 含路径、`failReason` 含 `operator:business-isolation`、`createTime` 非空。未重造事件类、未改既有 Listener。

---

## 6 · 未做项与原因

- 未在用户 phpStudy MySQL 执行 DDL：brief 契约明令「只产出不执行」；已用 H2 + 静态断言验幂等。
- 未改 `AlarmConfig` 默认 `platform_admin`：brief D1 授权「可加别名说明但不改」；脚本头登记 role_code 统一口径，部署方按需把 `ot.alarm.notice-target-role` 设为 `operator`。
- 未新增 `/api/v1/iot/calc/state|log` 以外的用量菜单业务页实现：只产出 `sys_menu id=173`（IoT用量查看）种子，前端页面不在本任务改动范围。
- 未跑 `-am` 全链路：brief 指定验收命令为 `mvn -pl ot-admin test`；已原样执行且全绿。
- 未置 TASK 已完成、未改 workitems 索引、未 push：按契约由协调者收尾。

## 7 · 偏离计划之处

1. 白名单 system 路径按**实际控制器路径**（`/user /role /menu /profile /auth`）实现，而非 brief 示例的 `/sys/*`、`/login`、`/getInfo`；理由见 §0 路径口径对齐，`SysTenantController` 项 brief 已授权按实际路径，其余同原则处理。
2. 角色 `data_scope=1`（全部）：运营方跨租户看健康/用量与租户主数据，业务数据由守卫硬隔离；如部署方希望进一步收窄，可在角色管理界面调整，不影响守卫语义。
3. 新增 `sys_menu id=173`（IoT用量查看，复用 `iot:instance:query` 权限码，不新增权限点）作为健康用量白名单菜单；brief 要求「健康用量白名单菜单」种子，0061 的 171 已覆盖巡检，故补一条用量入口。

## 8 · 遗留问题

- 授权工作流二期（PB-06 / TD-0042）不在本期：一期仅硬隔离 + 可审计；运营方临时看业务数据需走人工授权流程（二期实现）。
- 脚本 `sys_menu id=173` 的 `component='monitor/iot-usage/index'` 为预留前端页路径；若前端未交付该页，菜单点击会 404（不影响后端守卫/审计/幂等验收）。建议 FEAT-0020 后续任务对齐前端路由。
- 告警通知目标角色默认仍为 `platform_admin`：需在部署环境显式设置 `ot.alarm.notice-target-role=operator` 才能真正复用运营方角色（不改代码前提下由配置完成）。

## 9 · DDL 未执行声明

`resources/sql/upgrade/20261124_operator_isolation.sql` **仅产出，未在任何用户数据库（含 phpStudy MySQL 8.0.12@3307）执行**。幂等性通过 H2 内存库真执行两次 + 静态断言验证，证据见 §3.2。部署方须按 `backend/scripts/sql-exec.sh` 口径逐条执行。

---

## 10 · TASK 详情计划子任务勾选情况

TASK-0004 详情的三个计划子任务（角色菜单 / 接口拦截 / 审计埋点）本次全部实质完成；按契约执行者不手写 TASK 详情、不改索引，勾选与关闭由协调者收尾时据本报告回填。
