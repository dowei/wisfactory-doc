# TASK-0061 执行报告：运维监控页（四类指标 + 巡检清单 + 历史回看）

- 执行者：TASK-0061 执行者（no-subagents）
- BASE：`640c9bd`（派发前 HEAD）；执行期间 HEAD 由并行 TASK-0084 推进到 `3e09775`，本次提交记录见末节
- 状态：TASK 保持「进行中」（按 brief 契约不置已完成、不改 workitems 索引）
- 结论（一句话）：四类指标看板 + 三类巡检清单 + 标签租户比对 + 90 天历史回看已按 D1-D5 落地，backend 2482 测试全绿 / frontend build+lint+typecheck+606 单测全绿，DDL 只产出未执行，在线率 N/A 与跨租户 403 有契约测试证据。

## 一、改动清单（25 个新文件，均在 brief D1 声明范围内）

### backend（iot/inspect 新建巡检域，14 主 + 6 测试）
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/inspect/`
  - `entity/IotInspectResult.java`（巡检结果行，无审计四件套）
  - `vo/InspectItemVO.java`、`vo/InspectPayloads.java`（清单条目 + 聚合纯函数 + OnlineSnapshot）、`vo/DashboardVO.java`
  - `service/InspectReadService.java`、`impl/InspectReadServiceImpl.java`（只读端点服务，会话 tenant 强制注入 + fail-closed）
  - `store/InspectResultStore.java`（落库/最近一次/历史/90 天归档，条件删除幂等）
  - `controller/IotInspectController.java`（dashboard/latest/history 三 GET，权限 `iot:instance:query`，零写口，读不审）
  - `scheduler/InspectScheduler.java`（锁键/TTL 纯函数）、`scheduler/InspectTaskScheduler.java`（@Scheduled + Redis SETNX 重入锁 + fail-soft）、`scheduler/InspectTaskRunner.java`（三类清单 + 比对 + 落库归档任务体）
  - `config/InspectConfig.java`（store Bean + 归档期配置）、`support/InspectSources.java` + `support/JdbcInspectSources.java`（看板三数据源只读 SPI + 实现）
- `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/inspect/`（6 个契约测试类，21 用例）

### frontend（新建监控页 + API + 单测）
- `frontend/src/views/monitor/iot-inspect/index.vue`（四类指标卡 + 四 tab 清单 + 历史回看）
- `frontend/src/views/monitor/iot-inspect/helper.ts`（纯函数与类型，空态纪律）
- `frontend/src/api/monitor/iot-inspect-api.ts`（只封装 D4 三端点，安全红线头注）
- `frontend/tests/unit/monitor/iot-inspect-helper.spec.ts`（5 用例）

### resources
- `resources/sql/upgrade/20261124_iot_inspect.sql`（巡检结果表 DDL + sys_menu 171 + 超管授权；**只产出不执行**）

未触碰：`ingest/**`（0084 并行修订）、`resources/contract/**`、`iot/publish|calcread|statusread|metric` 既有代码、framework upstreams、01_base.sql（`git status` 核对见第五节）。

## 二、五项验收命令与原始输出

### 1. backend 单测
命令：`cd backend && mvn -pl ot-admin test`（brief 允许的工程既有全量命令；`-q` 变体 `mvn -pl ot-admin test -q` 退出码 0）
```
[INFO] Results:
[INFO] Tests run: 2482, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  01:36 min
```
其中本任务新增 21 用例（surefire 报告原始行）：
```
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- IotInspectControllerTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- InspectReadServiceContractTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- InspectResultStoreTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- IotInspectDdlSplitContractTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 -- InspectLockContractTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- InspectAggregatorTest
```

### 2. frontend 构建/检查（脚本名按 package.json 实际：build:test / lint / typecheck）
命令：`cd frontend && pnpm build:test && pnpm lint && pnpm typecheck`
```
$ pnpm build:test
✓ built in 55.82s                       # exit=0（仅有存量 chunk>500kB / css-syntax 既有 warning）
$ pnpm lint
✖ 9 problems (0 errors, 9 warnings)      # exit=0；9 条全为存量文件（survey/edit-lock、layout/setting-drawer、lib/axios 等），本任务 4 文件 0 warning
$ pnpm typecheck
$ vue-tsc --noEmit                       # exit=0，无 TS 错误
```
补充：`pnpm test`（vitest 全量）`Test Files 65 passed / Tests 606 passed`，含本任务 helper 5 用例。

### 3. DDL 静态检查
`resources/sql/upgrade/20261124_iot_inspect.sql` 按 `sql-exec.sh` 的 `;\n` 切分忠实复刻（同 IotCalcDdlSplitContractTest 口径）：
```
切分段数: 3                 # CREATE TABLE + 2 × INSERT
段内残余分号(同行多语句): 0
同行多语句行数: 0
```
JUnit 契约 `IotInspectDdlSplitContractTest`（3 用例）一并锁表结构 4 类 inspect_type + 复合索引 + 无审计列 + 菜单种子。

### 4. 范围核对
`git status --short` 本次工作树仅 25 个新文件（第一节清单），全部在 brief D1 声明范围内；`git diff` 无已跟踪文件被本次修改。
说明：执行期间并行任务 TASK-0084 提交了 `3e09775`（5 个 ingest 文件），因此 `git diff --name-only 640c9bd..HEAD` 会显示 ingest 5 文件——**该提交非本任务所为**；本任务未触碰 ingest/**（0084 在途）、contract/**、既有 publish/calcread/statusread/metric 模块。

### 5. 契约/空态/403 证据
- 在线率 N/A 非 100%：`InspectAggregatorTest.onlineRateNaWhenNoPoints`（`onlineRateText(0,0)="N/A"` / `(3,0)="N/A"`）、`InspectReadServiceContractTest.dashboardEmptyIsNa`（空看板 onlineRate/calcSuccessRate 均 N/A、channels/metrics 空）；前端 `iot-inspect-helper.spec.ts` 同口径 5 用例。
- 跨租户 403：`InspectReadServiceContractTest.crossTenantForbidden`（requestedTenant=999 ≠ 会话 tenant=1 → BusinessException code=403）、`restrictedScopeHistoryDenied`（受限 scope 无可见实例 → history 空集 fail-closed）；`IotInspectControllerTest.crossTenantForbiddenPropagates`（403 透传）。
- 零写口 / 读不审：`IotInspectControllerTest.readonlyGuardedAndNoAudit`（3 GET / 0 写映射 / 0 @OperateLog / 权限码 iot:instance:query）。
- 重入保护：`InspectLockContractTest`（`iot:inspect:lock:global`、ttl=周期×2）+ InspectTaskScheduler 中 Redis `setIfAbsent(key,token,ttl)` / finally 比对 value 才 DEL / Redis 故障 fail-soft WARN 跳过。
- 落库归档：`InspectResultStoreTest`（90 天旧行删除、新行保留、二次归档删 0 幂等、跨租户/类型隔离）。

## 三、TDD 红转绿留痕（每修复附「为什么现在对了」）

| 轮次 | 测试 | 红的原因 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- | --- |
| 1 | `IotInspectDdlSplitContractTest`（先写） | DDL 文件不存在，`NoSuchFileException`（3 用例全红） | 产出 `20261124_iot_inspect.sql`（建表 + 菜单 + 授权，每条语句独占行） | 文件存在且 3 段切分、无同行分号、结构字段/索引/4 类型齐全，契约通过 |
| 2 | 6 个 backend 契约测试（先写） | 编译失败：inspect 域 VO/service/controller/store 类全部不存在 | 实现 14 个主类（VO/聚合/store/读服务/控制器/调度/装配） | 所有符号齐备，21 用例可运行 |
| 3 | backend 首轮运行 | ① 控制器测试反射按 `dashboard(String)` 取方法 → `NoSuchMethodException`（测试侧签名笔误）；② 跨租户测试把「受限 data scope」误当「跨租户」（用 null 请求 tenant 调守卫，impl 对 null 放行）；③ 归档用 MySQL `DATE_SUB(NOW(),INTERVAL ? DAY)` 在 H2 MySQL 模式语法错误、被 fail-soft 吞成删 0 | ① 测试改按 `dashboard()` 反射（改测试代码错误，不是改期望）；② 测试改为传不同 tenant 断言 403 + 另增受限 scope 空集用例（区分两个契约）；③ 归档截止时间改在 Java 侧算好作绑定参数 `inspect_time < ?`，方言无关 | ① 反射匹配真实签名；② 403 由“请求 tenant ≠ 会话 tenant”触发、fail-closed 由“受限无实例”触发，两条独立可验；③ MySQL/H2 均接受参数化 timestamp，归档真正删除 2 条旧行且幂等，不再依赖方言 |
| 4 | 前端 helper 测试（先写） | helper.ts 不存在，vitest 解析失败（no tests） | 实现 helper.ts（N/A/空态纯函数 + 类型） | 5 用例全绿 |
| 5 | 前端 lint（新文件） | 新 4 文件 prettier/prettier warning | `eslint --fix` 仅作用于本任务 4 文件 | 新文件 0 warning；剩余 9 warning 全为存量文件，未改 |

## 四、DDL 应用条件（未在用户库执行声明）

- **本脚本只产出文件，未在用户 phpStudy MySQL（8.0.12@3307）执行任何语句**，未连任何用户库。
- 部署方操作步骤：逐条语句单独调用 `backend/scripts/sql-exec.sh`（该通道按 `;\n` 切分，脚本已保证每条独占一行）；先 `CREATE TABLE`，再两条 `INSERT ... WHERE NOT EXISTS`（可重复执行）。
- 应用后核对：`SHOW CREATE TABLE iot_inspect_result`；`SELECT id,parent_id,perms FROM sys_menu WHERE id=171`（期望 110 / `iot:instance:query`）；`SELECT * FROM sys_role_menu WHERE menu_id=171`（期望 `(1,171)`）。
- 运行时配置：`ot.inspect.interval-seconds`（默认 3600，@Scheduled fixedDelay + initialDelay）、`ot.inspect.retention-days`（默认 90）。

## 五、已知边界与登记事项

1. **`TENANT_DB_MISMATCH` 当前为「待核验」占位**：brief D2.4 明确 TDengine REST 当前故障，允许 fail-soft 产出占位项而非报错；条目 `result=待核验`、`expectedDb=iot_<tenantCode>`，真机 REST 恢复后仅替换取数处（聚合纯函数不变）。
2. **CALC_ERR / COMPLETENESS 巡检任务体为空集占位**：巡检调度线程无会话租户，0063 CalcReadService 按会话租户裁剪（`TenantContext.getTenantId()` 为 null → fail-closed）；任务体按 fail-soft 产出空清单，不做逐点扫库。看板侧（有会话）按 fullKey 逐键取 calc 日志实时计算成功率，不受此限。要真机产出明细需 0063 增加显式租户重载（不属本任务范围，未改 0063）。
3. **在线率快照源**：看板从巡检结果表最近一次 `ONLINE_RATE` 聚合快照读分子/分母；本任务未落 ONLINE_RATE 行（点位在线器属 statusread/接入侧，未在本批范围），故真机未接入前看板在线率显示 N/A（不造假），符合 REQ-0031 验收 6。
4. HEAD 漂移：执行中并行 TASK-0084 提交 `3e09775`（ingest/**），本任务提交基于当时 HEAD、仅 add 本任务 25 个文件，未 revert/覆盖 0084 变更。

## 六、提交

- 提交命令只 `git add` 第一节 25 个具体路径（不含 `.agents/`、`.tmp-t0044v/`、0084 的 ingest 文件），不 `git add -A`、不 push。
- **提交结果**：`git commit` 已完成，提交哈希 `60c4568`（完整 `60c456858736277705afe4f44c7bbd8ebf258c7a`）；`git show --pretty=format: --name-only HEAD` 恰 25 个文件，全部在声明范围，ingest/contract/既有模块零命中；未 push；TASK 状态保持「进行中」。

---

# 七、修订轮（diff 评审退回 I-1，一轮）

> 依据 `reports/TASK-0061-revision-brief.md` 与首评 `TASK-0061-review.md`。首评 1 Important（I-1）+ 4 Minor；本轮**只修 I-1**，4 Minor 记 deferred 不改。no-subagents；只改 backend inspect 范围；不置 TASK 已完成。

## 7.1 I-1 原文与根因

首评认定：`InspectTaskRunner.safeCalcErr()`/`safeCompleteness()` 恒 `return List.of()`、`onlineSnap=List.of()`，`InspectPayloads.groupCalcErrors()`/`approximateLowCompleteness()` 全 `src/main/java` 无生产调用（仅单测），真机每轮只落空集 + 「待核验」占位，未满足 REQ-0031 验收 2 / TC-0020 步骤 3。根因是纯实现缺陷：runner 已注入 JdbcTemplate 却以「CalcReadService 需会话租户」为由走空集捷径；而系统级调度可直接按 tenant_id 查 `iot_calc.iot_calc_state/log`（DDL :24/:39 明确有 tenant_id 列），绕开会话裁剪。

## 7.2 修复内容（真实取数 + 现成聚合调用）

仅改 `InspectTaskRunner.java`（+ 新增契约测试）：
- **CALC_ERR**：JdbcTemplate 直查 `iot_calc.iot_calc_log WHERE tenant_id=? AND result_status=2 AND started_at>=?`（窗口 `ot.inspect.calc-error-window-hours`，默认 24h），映射 `CalcLogVO` → 调 `InspectPayloads.groupCalcErrors(logs)` 按原因分组落 `TYPE_CALC_ERR`；仅查询异常才 fail-soft 空集 + WARN。
- **COMPLETENESS**：直查 `iot_calc.iot_calc_state WHERE tenant_id=?`，映射 `CalcStateVO` → 调 `InspectPayloads.approximateLowCompleteness(states)`（滞后/停算近似，basis 保留口径标注）落 `TYPE_COMPLETENESS`；异常才空集。
- **ONLINE_RATE**：`iot_point WHERE tenant_id=? AND deleted=0` 用 `SUM(CASE WHEN status=1 ...)/COUNT(*)` 一条 SQL 聚合真实分子/分母（**禁逐点扫 TDengine**），落 `ONLINE_RATE` 快照供看板读取；无点位 total=0 → 看板 N/A（REQ-0031 验收 6）；异常降级 total=0 + WARN。
- **TENANT_DB_MISMATCH**：保持不变，继续「待核验」占位（brief D2.4 TDengine REST 故障授权）。
- 未引入 CalcReadService/DataPermService 注入（不改 calcread 既有模块），未碰任何业务表写口。

## 7.3 TDD 红转绿（每修复附「为什么现在对了」）

| 轮次 | 测试 | 红因 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- | --- |
| 1 | `InspectTaskRunnerContractTest`（先写 4 用例） | 旧版 3 断言失败：CALC_ERR item_count=0（期望非空）、COMPLETENESS=0（期望 2）、ONLINE_RATE latest=null（期望快照），原样复现 I-1 | 实现真实取数 + 映射 + 调现成聚合 + 落 ONLINE_RATE | 给定样本日志（2 原因）聚合出非空分组、状态（滞后+停算）产出 2 条、点位（2 在线/1 离线）落 online=2/total=3；断言全部通过 |
| 2 | fail-soft 用例 | 无需修（旧版也绿） | 保留真实降级语义：drop 掉 iot_calc 表后查询抛错 → WARN 空集，其余清单照常 | 区分「真实降级」与「无条件空集」——旧版无数据时也空集无法通过非空用例，现仅异常时空集 |

新契约测试原始 RED：
```
InspectTaskRunnerContractTest.calcErrProducesNonEmptyGroupedList:110 期望 true 实际 false
InspectTaskRunnerContractTest.completenessProducesNonEmptyList:128 期望 2 实际 0
InspectTaskRunnerContractTest.onlineRateProducesRealCounts:143 期望 not null 实际 null
Tests run: 4, Failures: 3, Errors: 0
```
新契约测试原始 GREEN：
```
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- InspectTaskRunnerContractTest
[INFO] BUILD SUCCESS
```
修复运行日志证据（真实取数计数）：
```
[INSPECT] CALC_ERR tenant=1: 3 log row(s) -> 2 reason group(s)
[INSPECT] COMPLETENESS tenant=1: 3 state row(s) -> 2 low item(s)
[INSPECT] ONLINE_RATE 落库 online=2/total=3（契约断言）
[INSPECT] CALC_ERR 降级空集 tenant=1: bad SQL grammar ...（drop 表后的真实降级 + WARN）
```

## 7.4 验收命令与原始输出

1. `cd backend && mvn -pl ot-admin test`（原样命令）
```
[INFO] Results:
[INFO] Tests run: 2486, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  01:31 min
```
（2482 → 2486，新增 4 用例即 I-1 契约；0 失败 0 错误。）
2. frontend：本轮未碰前端（只改 InspectTaskRunner + 测试），按修订 brief 第 2 条可跳过——原首轮 `pnpm build:test`/`lint`/`typecheck` 仍绿，未重新运行。
3. 范围核对：`git status --short` 相对 `60c4568` 仅 2 文件：
```
 M backend/ot-admin/src/main/java/net/otlab/admin/module/iot/inspect/scheduler/InspectTaskRunner.java
?? backend/ot-admin/src/test/java/net/otlab/admin/module/iot/inspect/scheduler/InspectTaskRunnerContractTest.java
```
`git diff --name-only 60c4568..HEAD` 空（尚未提交）；frontend/contract/ingest/既有模块零触碰。

## 7.5 Minor（deferred，本轮不改，如实登记）

- **M-1**：`safeCalcState()` 非 allTenant 一刀切空集 → 租户管理员成功率恒 N/A（fail-closed 无泄露，可用性受损，演进位）。
- **M-2**：history 分页 `LIMIT page*size` + `subList` 跨页重复/漏（前端当前固定 pageNum=1，未暴露）。
- **M-3**：报告「0 ingest 引用」措辞与 diff 包 2 处 ingest 注释不符（0 文件改动属实，仅措辞）。
- **M-4**：单一 `iot:instance:query` 码无法区分运营看健康/租户管理员看明细（权限分层演进位）。

## 7.6 修订提交

- 本修订轮只 `git add` 上述 2 个具体路径，不 `git add -A`、不 push；提交信息注明「TASK-0061 修订轮 I-1」。
- **修订提交结果**：`git commit` 已完成，哈希 `24b5c44`（完整 `24b5c4434784885d6ac3eafba4a8c4d856fb39d5`），提交信息注明「TASK-0061 修订轮 I-1」；`git show --pretty=format: --name-only HEAD` 恰 2 文件（InspectTaskRunner.java + InspectTaskRunnerContractTest.java），逐路径核对全在 backend iot/inspect 范围；frontend/contract/ingest/既有模块零修改（测试类仅 import CalcLogVO/CalcStateVO 只读 VO 结构，不改 calcread 模块）；未 push；TASK 状态保持「进行中」。
