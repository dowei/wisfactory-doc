# 评审 brief：TASK-0004 diff 评审（代码评审）

> 实现提交 f928df9（BASE=57aeb36，区间干净单提交，9 文件）——运营方硬隔离与审计埋点。FEAT-0001 最后一块。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0004-diff.txt`（f928df9 单提交，9 文件，986 行）
- 实施 brief + 报告：reports\TASK-0004-brief.md / TASK-0004-report.md（含白名单全清单逐条理由 + 越权红线/审计证据 + §0 偏差登记）
- TASK 详情 + FEAT-0001/REQ-0001（**验收 6：运营方直调任一租户业务数据接口 403**）/TC-0001 + ADR-0011（TD-0040/0042）
- 挂载点基线：`backend/.../system/config/SaTokenInterceptorConfig.java`（addInterceptors 登录拦截器 + SaInterceptor + buildExcludePaths）
- 复用审计（不改）：`backend/ot-core/.../event/PermissionDeniedEvent.java` + `backend/.../system/listener/PermissionDeniedAuditListener.java` + GlobalExceptionHandler（BUG-0044 范式）
- 白名单目标端点（只读复用）：`iot/inspect/controller/IotInspectController.java`（dashboard/history/latest）、`iot/calcread/controller/IotCalcReadController.java`（state/log）、`iot/controller/SysTenantController.java`（实际路径）
- 角色种子范式：`resources/sql/01_base.sql:561-562`

## 越界反查

- 提交 9 文件（3 main：OperatorIsolationGuard/Policy/SaTokenInterceptorConfig + 5 test + 1 SQL），全落声明范围；frontend/ingest/calc/contract/既有 iot 业务实现零触碰。

## 评审重点

1. **白名单准确性（安全关键，头号重点）**：`OperatorIsolationPolicy` 默认拒绝是否正确——运营方（hasRole operator）**只能**放行健康用量（inspect 三端点 + calc state/log，**仅 GET**）、租户管理、系统自管理；其余 iot 业务接口一律 403。执行者按**实际控制器路径**（`/user /role /menu /profile /auth`）而非 brief 示例（`/sys/*` `/login` `/getInfo`）——必须独立核实这些实际路径是否与真实 @RequestMapping 完全一致、有无过宽通配导致业务接口被误放行、有无过窄导致登录/首页/getInfo 被误拒（运营方无法登录或 401 循环）。**逐条比对白名单 vs 真实控制器路由**。新增业务接口是否自动受控（白名单制而非黑名单）。
2. **守卫挂载与不误伤**：`OperatorIsolationGuard` 挂在登录校验后、SaInterceptor 前，preHandle 阶段生效（控制器前）；仅对 operator 生效，admin（role 1）/租户管理员不被误伤（测试锁三类角色）；403 用 NotRoleException → code=403、文案不泄露资源标识（与 0061/0062 越权口径一致）。
3. **审计埋点**：越权被拒复用 GlobalExceptionHandler → PermissionDeniedEvent → PermissionDeniedAuditListener 落库（未重造）；守卫若自定义异常/直接 response 是否仍能触发事件——报告需有审计落库证据；既有监听器零改动。
4. **角色种子与幂等**：role_code='operator'、id=3（避开 1/2）、幂等 WHERE NOT EXISTS；platform_admin 别名口径登记不污染 AlarmConfig；H2 MODE=MySQL 两次执行计数不变（角色 1 行/授权 13 行/菜单 173 一行）；**DDL 未在用户 phpStudy MySQL 执行**。
5. **测试有效性与零回归**：backend 2558 用例 0F0E（2534 基线 + 24 新增）；越权红线全套（每类业务接口代表 403 + 白名单 200 + 三类角色不误伤 + 审计 + SQL 幂等）有断言且非恒真；TDD 红转绿真实（GREEN-1 信封修正 / GREEN-2 反射发现 PUT 漏白名单补齐 / GREEN-3 H2 计数校正）；既有测试零回归。
6. **范围**：9 文件全落范围、区间干净、越界 0。

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0004-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。**特别对第 1 点白名单准确性（实际路径 vs 真实路由、有无过宽/过窄、新增接口自动受控）给出明确判定**——这是安全边界，不能只看测试绿灯。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
