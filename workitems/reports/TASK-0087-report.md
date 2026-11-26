# TASK-0087 执行报告：管理面数据项与指标值读口（登录态，补 FEAT-0019 缺口）

- 执行者：单执行者（no-subagents）
- BASE：`815cabf`
- 提交：`578753fde9fead5a16f7b5067b3faa061ab08525`（父提交 `33aef05` 为 TASK-0086 并行提交，文件集不交）
- 改动文件数：**10**（backend ot-admin 主代码 5 + 测试 3 + 修改既有 openapi service 契约/实现 2）
- TASK 状态：保持「进行中」（执行者不置已完成、不改 workitems 索引）

## 一句话结论

四个登录态只读端点已交付，复用 HMAC 同一份查询核心（仅身份→scope 翻译分离）、会话 `resolve(userId)` 数据权限裁剪、读不审、完整度无源诚实空集不造假；`cd backend && mvn -pl ot-admin test` 全绿（2576 测试，0 失败 0 错误 = 2559 基线 + 17 新增），范围核对通过。

## 1. 改动文件清单（10 个，全部落在 brief 声明范围内）

新增（管理面 read 包 `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/displayread/`，允许「iot 域新 read 包」）：
1. `IotDisplayReadService.java` — 四端点服务接口
2. `MetricValueVO.java` — 指标值展示 VO（对齐前端 MetricValueRow：ts/value/yoy/mom/period）
3. `MetricCompletenessVO.java` — 完整度响应（available/reason/rows，诚实空态）
4. `controller/IotDisplayReadController.java` — 四 GET 端点（登录态 + 权限码 + 读不审）
5. `impl/IotDisplayReadServiceImpl.java` — 会话 resolve 翻译 + 委托共享核心 + 403/空态语义

修改（openapi 查询组件「仅必要可见性调整」，brief 允许）：
6. `openapi/service/OpenApiQueryService.java` — 新增 3 个显式 `VisibleScope` 共享核心重载
7. `openapi/service/impl/OpenApiQueryServiceImpl.java` — HMAC 三入口委托到共享核心；原查询逻辑原样下沉，行为零改变

新增测试：
8. `test/.../iot/displayread/controller/IotDisplayReadControllerTest.java`（6 用例）
9. `test/.../iot/displayread/impl/IotDisplayReadServiceImplTest.java`（10 用例）
10. `test/.../openapi/service/impl/OpenApiQuerySharedCoreTest.java`（1 用例，复用一致性硬证据）

范围核对（`git diff --name-only 815cabf..HEAD` 共 17 行 = 本任务 10 + TASK-0086 并行提交 7 行 calc）：
- 本任务 10 行全部为 `backend/ot-admin/src/.../iot/displayread/**` 与 `openapi/service/**`，全部在声明范围。
- TASK-0086 并行提交 `33aef05` 的 7 行 `calc/**` 属其独立文件集（brief 明确「与 TASK-0086 文件集不交，可并行」），本任务未触碰。
- `frontend/**`、`ingest/**`、`resources/**`（含 contract）、HMAC 鉴权逻辑（`openapi/auth/**`、`openapi/filter/**`、`requireIdentity`、`OpenApiAuthFilter`）、既有六接口行为：本提交零触碰（已 grep 实查）。

## 2. 验收命令原始输出（原样贴回）

命令：`cd backend && mvn -pl ot-admin test`

```
[INFO] Running net.otlab.admin.module.iot.displayread.controller.IotDisplayReadControllerTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.241 s -- in net.otlab.admin.module.iot.displayread.controller.IotDisplayReadControllerTest
[INFO] Running net.otlab.admin.module.iot.displayread.impl.IotDisplayReadServiceImplTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.140 s -- in net.otlab.admin.module.iot.displayread.impl.IotDisplayReadServiceImplTest
[INFO] Running net.otlab.admin.module.openapi.service.impl.OpenApiQuerySharedCoreTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.128 s -- in net.otlab.admin.module.openapi.service.impl.OpenApiQuerySharedCoreTest
...
[INFO] Results:
[INFO] 
[INFO] Tests run: 2576, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:01 min
[INFO] Finished at: 2026-11-24T11:58:19+08:00
[INFO] ------------------------------------------------------------------------
```

说明：2559 基线 + 17 新增（6 控制器 + 10 服务 + 1 复用一致性），零回归；`OperateLogCoverageGuardTest`/`IotPermissionsCompleteGuardTest` 等既有守卫测试同轮通过（读不审、权限码已注册均未触发）。

## 3. TDD 红转绿留痕（每修复附「为什么现在对了」）

**红（先写测试）**：先落 16 个新契约测试（控制器 6 + 服务 10），编译期红——`OpenApiQueryServiceImpl 不是抽象的，未覆盖 metricsQuery(Long, List, String, Long, Long, VisibleScope)`（接口已加共享核心重载，实现未加），运行期目标类 `IotDisplayReadController` / `IotDisplayReadServiceImpl` 尚不存在。原始红输出要点：

```
[ERROR] ...OpenApiQueryServiceImpl不是抽象的, 并且未覆盖...metricsQuery(java.lang.Long,java.util.List<java.lang.String>,java.lang.String,java.lang.Long,java.lang.Long,net.otlab.admin.module.iot.dataperm.VisibleScope)
```

**逐修复 · 为什么现在对了**：

1. **接口/实现不同步红** → `OpenApiQueryServiceImpl` 增加三个显式 scope 重载（realtime/history/metrics），HMAC 旧签名保留并委托。
   *为什么现在对了*：编译不再报未覆盖；HMAC 六接口走旧签名→内部 `resolveScope(identity)`→共享核心，对外行为（scope 6003、400 校验、fail-soft、X-Truncated）逐字未改，既有 0051/0050 测试全绿佐证。
2. **共享核心中 `identity.getTenantId()` 的替换** → 核心内部统一用入参 `tenantId`，HMAC 入口传 `identity.getTenantId()`、管理面传会话 `TenantContext.getTenantId()`。
   *为什么现在对了*：租户值来源单一且显式，跨租户不可能通过请求参数注入；两个调用方同参走同一份核心，复用一致性测试以 spy 锁定委托链（同参同结果）。
3. **管理面服务/控制器不存在红** → 新增 `IotDisplayReadServiceImpl` + `IotDisplayReadController`（四 GET）。
   *为什么现在对了*：16 个契约测试全绿——正常取数 v/q/ts、scope 裁剪、403 通用文案、跨租户隔离（签名无 tenant 入参）、400 原样透传、完整度空态、权限注解与读不审断言逐项通过。
4. **复用一致性用例一次失败（verify 期望 1 实际 2）** → 修正为 `times(2)`（HMAC 委托 1 次 + 显式核心直呼 1 次）。
   *为什么现在对了*：断言的是「同一核心方法被两条入口共用」这一真实事实，而非人为规定的调用次数；修后该用例绿，成为 D3 共享逻辑的机械证据。
5. **完整度无源** → 以 `MetricCompletenessVO.unavailable(reason)` 返回，不接 calc HTTP、不写伪造值。
   *为什么现在对了*：无源时 `available=false` + 空 rows，测试断言 rows 中无 0/1 完整度，也不会静默把「无数据」判为完整。

**绿**：`mvn -pl ot-admin test` → `Results: Tests run: 2576, Failures: 0, Errors: 0, Skipped: 0` / `BUILD SUCCESS`。

## 4. 数据权限裁剪证据（会话 resolve，不是 resolveClientScope）

- 服务实现只调 `dataPermService.resolve(SecurityUtils.getUserId())`（第 122 行注释与实现同口径），与 `IotCalcReadController` / `CalcReadServiceImpl:91,120` 完全同款；全类无 `resolveClientScope` 调用（仅注释声明分离）。
- 租户取 `TenantContext.getTenantId()`（会话 tenant_id），传给共享核心的 mapper 显式 tenant 条件与 TDengine 库名派生；跨租户隔离由多租户插件 + 会话强制，服务签名不含 tenant 入参（测试 `crossTenantIsolation` 锁定）。
- 集合类（realtime/metric values）：scope 外键静默裁剪（核心原有语义，不回显存在性）；单资源类（路径指定 instanceId 无权、item/history scope 外 6003）：转 403 通用文案「无权访问该资源」，测试断言文案不含 fullKey。
- 无租户上下文（异步/漏设）：`requireTenant()` fail-closed 403，不返回任何数据。
- 四端点均 `@SaCheckPermission("iot:instance:query")`（沿用 0061/0063/0062，已注册 4031），无 `@OperateLog`（0079 读不审），控制器零写动词映射（测试反射锁定）。

## 5. 完整度读口可达性评估结论 + 诚实空集证据

**评估结论：不可达，采用诚实空集。**

- 数据源事实：`calc/src/main/java/net/otlab/calc/web/MetricObserveController.java:38` 的 `GET /calc/v1/metric` 是 calc 8095 内网**观测面**（类注释明确「非契约命名空间、无鉴权、不作为生产数据源入口」），`MetricRuntime.view()` 的 `windows.completeness` 为内存视图（进程内状态，无持久化物理列）。
- 契约事实：`resources/contract/tdengine-super-tables.md` 冻结的 `st_metric_value` 只有 (ts, value, yoy, mom)，无完整度列；`resources/sql/upgrade/20261124_iot_calc_schema.sql` 的 iot_calc 库仅 state/log 两表，无完整度结果表。
- 管理面事实：`backend/ot-admin` 无 calc HTTP 基址配置（grep `calc.*base-url|8095|/calc/v1` 仅命中注释与日志）、无 HTTP 客户端连 calc；0063 只读数据源只连 iot_calc schema 的 MySQL 表，拿不到 calc 进程内存视图。运行期 `curl -m 2 http://127.0.0.1:8095/calc/v1/metric` 无响应（000）。
- 因此不新建客户端、不连内网观测面（security + 契约双重不当），返回 `available=false` + reason「完整度读口接入中：calc 内网观测面未对管理面开放，暂无可用数据源」+ 空 rows；测试断言无 0/1 伪造值、无静默判完整。后续若 calc 提供管理面只读数据口，可在此服务内替换为真实映射，接口形状不变。

## 6. 复用策略说明（与 HMAC 共享逻辑，身份翻译分离）

- 共享逻辑：`OpenApiQueryService` 新增三个显式 `VisibleScope` 核心方法（`itemsRealtime / itemsHistory / metricsQuery` 的 `Long tenantId, ..., VisibleScope scope` 重载），HMAC 六接口（identity 入口）与展示端四端点都调用它们——查询逻辑只有一份，无复制。
- 差异仅在薄适配层身份翻译：HMAC 入口 `OpenApiQueryServiceImpl` 用 `resolveClientScope(clientId, tenantId, scopeJson)`（appKey 调用方口径）；管理面 `IotDisplayReadServiceImpl` 用 `resolve(userId)`（会话口径）。两者翻译出的 `VisibleScope` 结构相同，下游核心完全一致。
- 继承语义：fail-soft 空集、编码白名单、行数上限/截断（X-Truncated）、「最新一条」、value/yoy/mom 同源、null 伴随值不冒充 0 —— 全部由共享核心/客户端提供，管理面未另造。
- 机械证据：`OpenApiQuerySharedCoreTest` 以 spy 断言 HMAC identity 入口内部调用展示端复用的同一显式 scope 核心方法，且两条入口同参同结果。

## 7. 未做项与原因

- 前端接入（`display-api.ts` 四方法替换取数处）：brief 明确「本任务不改前端，接入另批」——未做。
- 完整度真实数据源接入：不可达（见 §5），按 brief 诚实空集；非遗漏。
- calc / ingest / resources / contract / HMAC 鉴权改动：brief 明令不触碰——未做。
- TASK 收尾（置已完成/回填哈希/LOG/索引）：执行者契约禁止——未做。

## 8. 偏离计划之处

- 无实质偏离。两点说明：
  1. 范围核对 `git diff --name-only 815cabf..HEAD` 含 TASK-0086 并行提交的 7 行 calc 文件（brief 已预告可并行、文件集不交）；本任务自身提交严格 10 文件全在声明范围。
  2. realtime 端点路径含 `{instanceId}`，按 D2「指定 instanceId 无权 403」实现（非集合空集），比纯集合端点更 fail-closed，与 TC-0019 步骤 7「工厂 B 设备直调 A 账号 403」同向。

## 9. 遗留问题

- 完整度读口待 calc 提供管理面只读数据口后再接真实值（当前诚实空集，前端 `completenessDisplay(null)` 已按「数据接入中」空态呈现）。
- 四端点真实 TDengine/Redis 联调（0042/0023 fail-soft 空集如实）归 0068 环境恢复后联调，不在本任务验收范围。

---

## 修订轮（I-1：allTenant scope 判定修复，BASE=578753f）

> 评审退回一轮：主体 578753f 验收全过但 diff 评审发现 Important I-1——共享核心三处 `!scope.instanceIds().contains(...)` 缺 allTenant 检查；管理面接入 `resolve(userId)` 后超管/运营方的 allTenant scope（instanceIds 恒空集）会被误判为 scope 外，四读口取不到任何数据。本轮只修 I-1。

### R1. I-1 三处修复（`OpenApiQueryServiceImpl`，与同文件 :118 范式一致）

| 位置 | 方法 | 修改前 | 修改后 |
| --- | --- | --- | --- |
| :182 | `itemsRealtime` | `item == null \|\| !scope.instanceIds().contains(item.getInstanceId())` | `item == null \|\| (!scope.allTenant() && !scope.instanceIds().contains(item.getInstanceId()))` |
| :285 | `metricsQuery` | `metric == null \|\| !scope.instanceIds().contains(metric.getInstanceId())` | `metric == null \|\| (!scope.allTenant() && !scope.instanceIds().contains(metric.getInstanceId()))` |
| :395 | `requireInstanceVisible` | `instanceId == null \|\| !scope.instanceIds().contains(instanceId)` | `instanceId == null \|\| (!scope.allTenant() && !scope.instanceIds().contains(instanceId))`，保留 instanceId==null 始终拒绝 |

- 附带同方法内兄弟判定 `:275`（`metricsQuery` 装载实例编码回显的 `instanceIds` 集合）同样改为 `scope.allTenant() || scope.instanceIds().contains(row.getInstanceId())`——否则 allTenant 下回显 map 仍为空、指标值拿不到 instanceCode，三处修复不闭合。
- `resolveScope`/HMAC 身份翻译路径未改；`:329 points/history` 的 `resolveScope(identity)` 判定保持原样（HMAC scope 恒 `allTenant=false`，无管理面入口复用该方法）。

### R2. allTenant=true 管理面读口契约测试（TDD 红转绿留痕）

- 新增 `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/displayread/impl/IotDisplayReadAllTenantContractTest.java`：真实 `OpenApiQueryServiceImpl` 共享核心 + 假 mapper/store 端到端；mock `DataPermService.resolve(userId)` → `VisibleScope.allTenant(false)`，经管理面 `IotDisplayReadService.itemsRealtime` / `metricValues` 取数，断言无异常 + 非空 realtime（v=23.5）+ 非空 metric values（value=412.6、yoy=0.083、ts 非空）。
- **红（修复前）**：`mvn -pl ot-admin -Dtest=IotDisplayReadAllTenantContractTest test` → `Failures: 1`，`allTenant 超管经管理面实时读口必须取到数据 … expected: <false> but was: <true>`（realtime 空集，`instanceIds` 空集裁剪所致），`BUILD FAILURE`。
- **绿（修复后）**：同命令 → `Tests run: 1, Failures: 0, Errors: 0` / `BUILD SUCCESS`。

### R3. 为什么现在对了

- `VisibleScope.allTenant()` 的 `instanceIds` 恒 `Set.of()`（`VisibleScope.java:36-38`），语义是「不做任何数据权限条件」（`DataPermConditionAssembler`/`TdengineTagConditionRenderer` 同款 `allTenant` 分支）。修复后三处统一先判 `scope.allTenant()`：true 时跳过集合检查（视为可见），false 时仍按可见集裁剪，与 `:118 modelsTree` 的正确范式一致，数据权限语义恢复（超管/运营方可查全租户数据，fail-closed 不再误伤合法请求）。
- `:395` 的 `instanceId == null` 始终拒绝保留，null-scope（`resolve` 返回 null → `sessionScope()` 收敛为 `denied()`，核心 scope 为 null 也收敛为 `denied()`）路径不变；`:275` 回显 map 补全后，指标 instanceCode 与值同源，管理面 VO 映射链完整。

### R4. HMAC 侧零回归证据

- HMAC 六接口入口仍只走 `resolveClientScope`（`DataPermService.resolveClientScope` 的 `allTenant` 恒 false，见 `DataPermService.java:76` javadoc），三处新判定 `!scope.allTenant()` 分支对 HMAC 恒定进入原集合裁剪逻辑，行为逐字未变；`resolveScope` 未改，`:329 points/history` 未改。
- 全量验收 `cd backend && mvn -pl ot-admin test`：`Results: Tests run: 2577, Failures: 0, Errors: 0, Skipped: 0` / `BUILD SUCCESS`（2576 主体基线 + 1 新增 allTenant 契约；原有 scope 外/denied/跨租户/HMAC 六接口/守卫测试全部同轮通过）。
- 范围核对：`git diff --name-only 578753f..HEAD`（提交后）仅 `openapi/service/impl/OpenApiQueryServiceImpl.java` 与 `iot/displayread/impl/IotDisplayReadAllTenantContractTest.java`——frontend/calc/ingest/resources/contract/HMAC 鉴权零触碰。

### R5. 附带核查（评审线索）

- `distinct` 用法：`normalizeKeys` 以 `LinkedHashSet` 保序去重，HMAC 与管理面同走共享核心，本次未改；无新行为差异。
- HMAC 六接口既有测试：`OpenApiQueryServiceImplTest` / `OpenApiStatusQueryTest` / `ClientScopeRedlineTest` 等在本轮全量验收中全绿，无回归。

### R6. 修订验收原始输出（Results 段）

命令：`cd backend && mvn -pl ot-admin test`

```
[INFO] Results:
[INFO] 
[INFO] Tests run: 2577, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:12 min
[INFO] Finished at: 2026-11-24T12:32:45+08:00
[INFO] ------------------------------------------------------------------------
```

- TASK 状态保持「进行中」（修订执行者不置已完成、不改 workitems 索引）；提交信息注明「TASK-0087 修订轮 I-1」；不 push。
