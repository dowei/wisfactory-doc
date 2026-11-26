# TASK-0061 scoped 复评 round2（终审）

- 评审角色：独立代码评审（fresh / no-subagents / 只评审不修改，唯一例外=本报告）
- 评审对象：隔离修订 `24b5c44`（区间 `60c4568..24b5c44`，1 提交、2 文件，全 backend iot/inspect）
- 口径：L1-execute §13「diff 评审」；scoped 复评只覆盖 I-1 修复 + 回归确认；0061 已用尽「1 退回 + 1 复评」，本轮为终审
- 判定口径：闭合 I-1 且无新 C/I → 通过，否则如实不通过

## 一句话总评

修订轮把三类清单的**旧版恒空集捷径真正删除**，改为用已注入的 JdbcTemplate 按租户遍历的 `tenant_id` 直查 `iot_calc.iot_calc_log`/`iot_calc_state` 并映射 VO 后调用现成聚合纯函数，ONLINE_RATE 改为一条 SQL 聚合真实分子/分母；我独立重跑证明新增契约测试在旧版 `60c4568` 上 3 断言必红、在修订版全绿（非恒真、非 mock 捷径），范围恰 2 文件、backend 2486 用例 0F0E、4 Minor 如实 deferred 未改。I-1 闭合，无新 C/I。

## 明确判定

| 判定项 | 结论 |
| --- | --- |
| I-1（巡检清单恒空集）是否闭合 | **是（闭合）** |
| 是否有新 Critical / Important | **无**（0 C / 0 I） |
| 终审结论 | **通过** |

## 复评重点逐项核对（均回到源码/实测独立核实，非采信报告自述）

### 重点 1 · I-1 真修复

1. **旧版恒 `List.of()` 是否真移除——真移除**。取 `git worktree` 检出修订前的 `60c4568` 独立核对：`InspectTaskRunner.java` 旧版幸福路径确有 `onlineSnap = List.of()`（:68）、`safeCalcErr()` 恒 `return List.of()`（:82）、`safeCompleteness()` 恒 `return List.of()`（:90）。修订版 `24b5c44` 源码（`:89-134`）改为真实查询 + 聚合；全文件残留的 3 处 `List.of()`（:110/:132/:159）**全部位于 `catch (RuntimeException)` 降级分支**，不再是幸福路径捷径。
2. **已注入 JdbcTemplate 按 tenant_id 直查——成立**。`safeCalcErr()` `:92-102` 执行 `SELECT full_key, result_status, error_msg FROM iot_calc.iot_calc_log WHERE tenant_id=? AND result_status=2 AND started_at>=?`；`safeCompleteness()` `:117-125` 执行 `SELECT full_key, calc_status FROM iot_calc.iot_calc_state WHERE tenant_id=?`。权威 DDL `resources/sql/upgrade/20261124_iot_calc_schema.sql:24`（state）与 `:39`（log）确证两表均有 `tenant_id` 列，`:46` 确证 `result_status=2` 为失败，`:28` 确证 `calc_status 2 滞后 / 3 异常停算`。
3. **聚合函数是否真被生产代码调用——真调用**。全仓 `grep groupCalcErrors|approximateLowCompleteness` 结果：生产代码调用点仅 `InspectTaskRunner.java:103`（`groupCalcErrors(logs)`）与 `:126`（`approximateLowCompleteness(states)`），其余为 `InspectPayloads.java:68/:93` 定义本身。直接消除首评「全 `src/main/java` 仅定义、无生产调用」的认定。`groupCalcErrors` 按 `errorMsg` 归组计数、`approximateLowCompleteness` 按 `calcStatus>=2` 产出条目并保留「calc 状态近似」basis，均为真实非平凡聚合（`InspectPayloads.java:68-109`）。
4. **ONLINE_RATE 真实聚合、禁逐点扫 TDengine——成立**。`safeOnlineSnapshot()` `:139-142` 一条 SQL `SELECT SUM(CASE WHEN status=1 THEN 1 ELSE 0 END) AS online, COUNT(*) AS total FROM iot_point WHERE tenant_id=? AND deleted=0`；无 `SELECT` 逐点、`无 TDengine 调用`，全仓无新增 TDengine REST 依赖。`total=0` 由下游 `onlineRateText()`（`InspectPayloads.java:30-35`）判 N/A，符合 REQ-0031 验收 6；`saveOnlineRate()`（`:173-184`）落库 payload `[{online,total,basis}]`，消费端 `JdbcInspectSources.onlineRate()`（`:76-101`）取最近一次快照解析，读写口径一致。
5. **TENANT_DB_MISMATCH 保持占位——成立**，`:76-77` 仍为 `pendingVerification(tenant.code(), expectedDbName(...))`，与 brief D2.4 授权一致。
6. 三类清单（CALC_ERR / COMPLETENESS / TENANT_DB_MISMATCH）落库见 `:82-84`；ONLINE_RATE 落库见 `:85`。REQ-0031 验收 2「巡检任务一次运行产出三类清单 + 标签租户比对清单」的 I-1 缺口已消除。

### 重点 2 · fail-soft 语义与 tenant_id 来源

- **异常才降级、与无条件空集明确区分——成立**。`safeCalcErr()`/`safeCompleteness()` 的空集仅在 `catch (RuntimeException e)` 内返回并伴 `log.warn`（`:107-111` / `:130-133`）；ONLINE_RATE 异常降级 `total=0` + WARN（`:147-150`）。正常路径必然执行查询与聚合，无「先返回空集再解释」的捷径。契约测试第 4 例 `calcQueryFailureDegradesToEmpty` 专测 drop 表后空集 + 其它清单照常落库（测试源码 `:341-356`）。
- **tenant_id 强制来自租户遍历而非请求参数——成立**。`run()` `:59-63` 由 `activeTenants()`（`SELECT id, tenant_code FROM sys_tenant WHERE status=1 AND deleted=0`，`:155-156`）产出 `TenantRow`，逐租户把 `tenant.id()` 传入三个查询与落库；系统级调度线程无会话、无 HTTP 请求参数入口，不存在借用会话/请求 tenant 的路径。

### 重点 3 · 测试有效性（TDD 红转绿）——独立实测

不采信报告中粘贴的 RED/GREEN 文本，做了两次真实执行：

- **GREEN（修订版 HEAD `24b5c44`）**：强制全量重编译后跑 `mvn -pl ot-admin test -Dtest=InspectTaskRunnerContractTest`，结果 `Tests run: 4, Failures: 0, Errors: 0, Skipped: 0`、`BUILD SUCCESS`。
- **RED（修订前 `60c4568`）**：在临时 `git worktree`（用后即删）检出旧版 runner、仅放入同一份新契约测试后运行，结果 `Tests run: 4, Failures: 3, Errors: 0, Skipped: 0`、`BUILD FAILURE`，失败点与首评 I-1 一一对应：
  - `calcErrProducesNonEmptyGroupedList:110` expected `<true>` but was `<false>`（CALC_ERR 恒空集）；
  - `completenessProducesNonEmptyList:128` expected `<2>` but was `<0>`（COMPLETENESS 恒空集）；
  - `onlineRateProducesRealCounts:143` expected `not <null>`（ONLINE_RATE 旧版不落库）；
  - 第 4 例 `calcQueryFailureDegradesToEmpty` 在新旧两版均绿——符合其「降级语义不随修复变化」的设计，非红转绿判别项。
- **断言落到落库 payload、非恒真、非 mock 捷径**：测试用 H2 内存库建真表并 `INSERT` 样本（`InspectTaskRunnerContractTest.java:236-285`），经 `runner.run()` 后用 `store.latest(...)` 取落库行断言：CALC_ERR 3 条失败日志（超时×2、除零×1）+ 1 条成功过滤 → `itemCount>0` 且 payload 含两类原因与 `\"count\":2`（`:289-305`）；COMPLETENESS 3 条状态（滞后/停算/正常）→ `itemCount=2` 且含「calc 状态近似」（`:309-321`）；ONLINE_RATE 3 点位（2 在线/1 离线）→ payload 含 `\"online\":2`、`\"total\":3`（`:325-337`）。无 `mock JdbcTemplate`、无 `assertThat(true)` 类恒真式。
- 说明：brief 复评重点 3 所述「4 项非空清单断言」实为 3 项数据非空断言 + 1 项 fail-soft 降级断言；三项数据断言均逐条核实为真实且旧版必红。

### 重点 4 · Minor deferred 与回归

- **4 Minor 如实登记未改**：`TASK-0061-report.md` 修订轮 §7.5 保留 M-1（`safeCalcState()` 非 allTenant 空集）、M-2（history 分页 `LIMIT page*size`+`subList`）、M-3（「0 ingest 引用」措辞）、M-4（单一 `iot:instance:query` 码）为 deferred；本轮 2 文件 diff 未触碰相关代码，与 first-review 原文一致。
- **测试与零回归**：surefire 原始报告全量汇总 `Tests=2486, Failures=0, Errors=0, Skipped=0`（基线 2482 + 新增 4）；新增类独立报告 `Tests run: 4, Failures: 0, Errors: 0`。本轮未碰前端，按修订 brief 第 2 条可跳过，既有测试无回归。
- **范围仅 2 文件**：见下节越界反查。

## 越界反查

- `git log --oneline 60c4568..24b5c44` 恰 1 提交（`24b5c44`）；`git diff --name-only 60c4568..24b5c44` 与 `git show --name-only 24b5c44` 均恰 2 文件：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/inspect/scheduler/InspectTaskRunner.java` 与 `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/inspect/scheduler/InspectTaskRunnerContractTest.java`，全在 backend iot/inspect。
- 父链：`24b5c44^ = 60c4568`，`60c4568^ = 3e09775`（TASK-0084 修订），0084 在更早提交已隔离、不在本轮范围；HEAD = `24b5c44`，工作树仅存量未跟踪目录 `.agents/`、`.tmp-t0044v/`，无评审引入改动。
- 结论：**无越界**——frontend/contract/ingest/既有模块零触碰（测试类仅 import `CalcLogVO`/`CalcStateVO` 只读结构，不改 calcread 模块）。

## 发现清单

| 严重级 | 数量 | 说明 |
| --- | --- | --- |
| Critical | 0 | 无 |
| Important | 0 | 无（I-1 已闭合） |
| Minor（新增） | 0 | 无 |
| Minor（deferred，non-blocking） | 4 | M-1/M-2/M-3/M-4，出处与说明见 `reports/TASK-0061-review.md` §发现清单，修订轮原样登记于 `reports/TASK-0061-report.md` §7.5，本轮未改、不阻断 |

**非阻断观察（不计为 C/I，不影响通过判定，供后续跟踪）**：`resources/sql/upgrade/20261122_iot_point_domain.sql:97` 把 `iot_point.status` 注释为「0 禁用 1 启用」，本轮 ONLINE_RATE 以 `status=1` 作为「在线」近似（basis 已在产物中如实标注口径）；更权威的在线态源为 `iot:st` Redis statusread，但本轮 brief 复评重点 1 明确点名「用 iot_point status 一条 SQL 聚合」，属协调者裁决范围内的近似实现。另：系统级直查跨 schema `iot_calc` 依赖部署账号具备相应 SELECT 授权（`20261124_iot_calc_schema.sql:79` 的 `iot_calc_ro` 仅示例最小授权）——若生产主账号无跨 schema 读权限，查询会走异常降级为空集，建议部署时核对授权；此为部署前置条件，不改变本轮 scoped 评审结论。

## 契约遵守

- fresh / no-subagents / 只评审不修改（除本报告外未改任何文件、未推进状态、未派子代理）：已遵守。临时 RED 验证 worktree 用后即删，评审对象仓库 HEAD 与工作树未变。
- 0061 终审口径已按 brief 执行：I-1 闭合 + 无新 C/I → **通过**；4 Minor 仍为 deferred，不进入本轮阻断项。
