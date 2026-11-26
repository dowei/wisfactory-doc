# TASK-0085 执行报告：BUG-0006 微改——补 operator 角色 iot:tenant:admin 授权

- BASE：`f928df9`
- HEAD（本任务提交）：`815cabfc7ed129d2071ec45627ea6cafd4a15738`（短 `815cabf`）
- 性质：微改（单模块、无接口/数据模型/业务规则变化），**跳过 diff 评审**（L1-execute §13 微改档，按 brief 要求）
- TASK 状态：保持「进行中」；BUG-0006 状态未改（由协调者收尾）
- 一句话结论：operator 角色补授 `iot:tenant:admin`（menu 4017）幂等授权后，真实 SysTenantController 注解链测试由红转绿，SQL H2 二次执行授权数 14 稳定；2559 项 ot-admin 单测全绿，DDL 只产出未执行。

---

## 1 · 改动文件清单（3 个，全部落声明范围）

| 文件 | 性质 |
| --- | --- |
| `D:/workspace/WisFactory/resources/sql/upgrade/20261124_operator_isolation.sql` | 改：operator 授权段补 1 行幂等 `sys_role_menu(3,4017)`；注释与核对期望 13→14 |
| `D:/workspace/WisFactory/backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorRoleSeedSqlTest.java` | 改：H2 授权数断言 13→14；夹具补 menu 4017；新增 4017 授权存在断言 |
| `D:/workspace/WisFactory/backend/ot-admin/src/test/java/net/otlab/admin/module/iot/controller/SysTenantControllerOperatorInitAdminTest.java` | 新：真实 SysTenantController 注解链测试（operator 经守卫+SaInterceptor 到 init-admin） |

`git diff --name-only f928df9..HEAD` 恰为上述 3 项。守卫 `OperatorIsolationGuard` / 策略 `OperatorIsolationPolicy` / `SaTokenInterceptorConfig` / 角色语义 / 其他角色授权 / `AlarmConfig` 全部零改动。工作副本 `.agents/`、`.tmp-t0044v/` 为任务前已存在未跟踪目录，未纳入提交、未改动。

## 2 · 脚本选择理由（brief 接口决策第 1 条要求自决并说明）

**选择：补进原 `20261124_operator_isolation.sql`，不新建脚本。**

理由：
1. TASK-0004 报告 §9 与代码评审报告 §四.5 均已核实：该脚本「只产出、不在用户库执行」，**没有任何环境应用过它**（无 DDL 执行痕迹）。因此不存在「部分环境已应用旧版、补原脚本会漏执行」的风险，brief 给出的「新建脚本更稳妥」触发条件不成立。
2. 保持单一权威的运营方角色种子脚本：role 定义、菜单授权（含 4017）、幂等口径集中一处，避免两份 upgrade 脚本并存导致部署方执行顺序疑惑。
3. 新增行仍是幂等 `INSERT ... SELECT 3, 4017 WHERE NOT EXISTS (...)`：即便某个环境意外先应用了旧版，重新执行补后的同一脚本即补齐 4017，无重复插入风险（H2 二次执行实测计数不变）。
4. 改动粒度最小：仅 1 行 SQL + 注释，符合微改契约。

---

## 3 · 验收命令原始输出（原样贴回）

### 3.1 `cd backend && mvn -pl ot-admin test`（brief 验收命令 1）

```
[INFO] Running net.otlab.admin.module.iot.controller.SysTenantControllerOperatorInitAdminTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.220 s -- in net.otlab.admin.module.iot.controller.SysTenantControllerOperatorInitAdminTest
[INFO] Running net.otlab.admin.module.system.config.OperatorRoleSeedSqlTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.047 s -- in net.otlab.admin.module.system.config.OperatorRoleSeedSqlTest
...
[INFO] Results:
[INFO] 
[INFO] Tests run: 2559, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:06 min
```

2559 = 2558（TASK-0004 基线）+ 1（本任务新增注解链测试）。零失败、零错误。

### 3.2 SQL 幂等（brief 验收命令 2）

不在用户 phpStudy MySQL（8.0.12@3307）执行 DDL。用 H2 `MODE=MySQL` 真执行 + 静态断言双重验证：

`OperatorRoleSeedSqlTest`（5/5 绿，含 H2 用例 `scriptIdempotentOnH2`）实测：
- 第 1 次（CREATED）：`sys_role WHERE role_code='operator'` = 1 行；`sys_role_menu WHERE role_id=3` = **14** 行（原 13 + 4017）；`WHERE role_id=3 AND menu_id=4017` = 1 行；`sys_menu id=173` = 1 行
- 第 2 次（EXISTS）：四项计数均不变（1 / 14 / 1 / 1），无重复插入、无主键冲突

静态断言同时锁定：4017 授权行使用 `WHERE NOT EXISTS (role_id=3 AND menu_id=4017)`；脚本头「只产出不执行」声明；每条语句独占一行（sql-exec.sh `;\n` 切分兼容）。

### 3.3 范围核对（brief 验收命令 3）

```
$ git diff --name-only f928df9..HEAD
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/controller/SysTenantControllerOperatorInitAdminTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/system/config/OperatorRoleSeedSqlTest.java
resources/sql/upgrade/20261124_operator_isolation.sql
```

仅 SQL 授权脚本 + 测试文件，守卫/角色语义/其他角色/AlarmConfig/前后端业务代码零触碰。

---

## 4 · TDD 红转绿留痕（未授权必红 / 补后绿）

| 轮次 | 动作 | 实测输出 |
| --- | --- | --- |
| RED-0 | 写测试后发现 `setUp()` 未声明 `throws IOException`（编译错）：`未报告的异常错误 java.io.IOException`，BUILD FAILURE | 修夹具签名（非改期望），重跑 |
| RED | 未补 SQL 授权时跑 `SysTenantControllerOperatorInitAdminTest,OperatorRoleSeedSqlTest` | `Tests run: 6, Failures: 2`：<br>① `前置条件：operator 菜单授权集必须含 iot:tenant:admin（menu 4017）——SQL 未补授权时本例必红 ==> expected: <true> but was: <false>`<br>② `首次执行运营方菜单授权应为 14 条... ==> expected: <14> but was: <13>`<br>BUILD FAILURE（退出码 1） |
| GREEN | 在原 SQL 补 `INSERT INTO sys_role_menu ... SELECT 3, 4017 WHERE NOT EXISTS (...)`，SQL 测试 13→14 同步 | `Tests run: 6, Failures: 0, Errors: 0, Skipped: 0`，BUILD SUCCESS |
| 为什么现在对了 | 注解链测试的 operator 权限集**从 SQL 实读派生**（role 3 的 `sys_role_menu` 授权 ID ∩ `sys_menu` perms，4017→`iot:tenant:admin`）。补授权后 `SaInterceptor` 在 `POST /iot/tenant/7/init-admin` 的 `@SaCheckPermission("iot:tenant:admin")` 处放行，真实控制器返回 `Result.success(9001)`（`$.code=200`, `$.data=9001`）；未补时权限集缺该码必 403/前置失败。测试与 SQL 内容直接绑定， dummy 控制器再也无法掩盖此缺口。 | 全量 `mvn -pl ot-admin test` 2559/2559 绿 |

## 5 · 真实注解链测试有效性（brief 接口决策第 2 条）

`SysTenantControllerOperatorInitAdminTest` 装配：
- **真实控制器**：`new SysTenantController()`（非 dummy），仅注入 mock `SysTenantService`（`initTenantAdmin` 返回 9001）
- **真实守卫**：`OperatorIsolationGuard`（路径 `/iot/tenant/7/init-admin` 在白名单内 → 放行）
- **真实注解鉴权**：`new SaInterceptor(handle -> {})` 处理 `@SaCheckPermission("iot:tenant:admin")`
- **真实登录态**：`StpUtil.login(3)`，`StpInterface.getRoleList` 返回 `["operator"]`、`getPermissionList` 返回 SQL 派生的权限集
- **断言**：`$.code=200`、`$.data=9001`；权限集前置断言强制 SQL 补授权

与 TASK-0004 的 dummy 测试形成互补，闭合评审 F-1 指出的「dummy 无注解掩盖缺口」。

---

## 6 · 未做项与原因

- 未在用户 phpStudy MySQL 执行 DDL：brief 契约明令；用 H2 内存库 + 静态断言验幂等。
- 未新建独立 upgrade 脚本：理由见 §2（原脚本从未应用，补原脚本即充分且更集中）。
- 未改守卫白名单/角色语义/其他角色/AlarmConfig：brief 约束「不误伤、不扩权」，本微改仅补一条授权。
- 未改 TASK/BUG 状态、索引：按契约由协调者收尾。
- 微改跳过 diff 评审：按 brief 与 L1-execute §13 微改档，仅验收命令。

## 7 · 偏离计划之处

无实质偏离。brief 允许「原脚本补一行 或 新建幂等脚本」二选一，本报告 §2 给出选择理由。测试「未补授权必红」实测以前置断言红（权限集中缺 `iot:tenant:admin`）体现，而非直接断言 HTTP 403——直接原因同一：`SaInterceptor` 注解鉴权会因缺码抛 `NotPermissionException`→403；前置断言更明确定位到 SQL 授权缺口，补后走完整链返回 200。

## 8 · 遗留问题

- menu 4017 行本身由 `20261121_tenant_id_columns.sql:53` 登记（历史上仅授 role 1），本次只补 operator（role 3）授权；其他角色如需该权限仍由界面/单独脚本分配，符合「不改其他角色」约束。
- 若某部署环境在 TASK-0004 后已手工执行过旧版 `20261124_operator_isolation.sql`，只需重跑补后的同一脚本（幂等）即可补上 4017，无需额外迁移脚本。

---

## 9 · DDL 未执行声明

`resources/sql/upgrade/20261124_operator_isolation.sql`（含本次补的 4017 授权行）**仅产出，未在任何用户数据库（含 phpStudy MySQL 8.0.12@3307）执行**。幂等性由 H2 `MODE=MySQL` 真执行两次（计数 1 / 14 / 1 / 1 稳定）+ 静态断言验证，证据见 §3.2。部署方须按 `backend/scripts/sql-exec.sh` 口径逐条执行。

## 10 · TASK 详情计划子任务

TASK-0085 两个计划子任务（补授权 / 注解链测试）本次全部实质完成；按契约执行者不手写 TASK 详情、不改 BUG 状态、不改索引，由协调者收尾时据本报告回填。
