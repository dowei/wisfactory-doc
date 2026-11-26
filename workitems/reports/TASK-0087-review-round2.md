# TASK-0087 复评报告（round2 终审 · scoped diff 评审）

- 评审角色：独立代码评审（fresh、no-subagents、只评审不修改；唯一例外：本报告）
- 评审口径：L1-execute §13「diff 评审」；scoped 复评只覆盖 I-1 修复 + 回归确认
- 评审对象：隔离修订 `578753f..2d0f1a9`（HEAD=`2d0f1a9f4c9b576084c2d1bcf35c156a5f7bf4b6`），2 文件
- 评审日期：2026-11-24
- 终审约束：0087 已用尽「1 退回 + 1 复评」上限；闭合 I-1 且无新 C/I → 通过，否则不通过

## 一句话结论

**通过（PASS）**。I-1 闭合：三处共享核心 scope 判定（`:182/:285/:395`）均改为 allTenant 感知，与 `:118` 范式一致；`:395` 保留 `instanceId==null` 始终拒绝；`:275` 附带修复必要且正确（实测单独回退该 hunk 会让 allTenant 指标读口 NPE/不闭合）。新增契约测试走真实共享核心、非恒真、非 mock 捷径，修复前必红（`Failures:1` BUILD FAILURE）、修后绿（1/0/0）。HMAC 零回归、范围 2 文件越界 0、全量 2577 用例 0F0E。**无新 Critical / Important / Minor 发现。**

## 发现清单

| 严重级 | 出处 | 说明 |
| --- | --- | --- |
| — | — | 无 C/I/M 发现。 |

（观察项，不构成发现、不影响结论：见 §6。）

---

## 1. 范围与越界反查（独立核实）

- `git rev-parse HEAD` = `2d0f1a9f4c9b576084c2d1bcf35c156a5f7bf4b6`；`git log --oneline 578753f..HEAD` 恰 1 条提交（修订轮 I-1）。
- `git diff --name-only 578753f..HEAD` 恰 2 文件：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/service/impl/OpenApiQueryServiceImpl.java`
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/displayread/impl/IotDisplayReadAllTenantContractTest.java`（新增）
- `git show --stat 2d0f1a9`：2 files changed, 165 insertions(+), 5 deletions(-)。
- `git diff --stat 578753f..HEAD` 与传入的 `TASK-0087-diff-round2.txt`（209 行）一致。
- 工作区 `git status --short` 仅 `?? .agents/`、`?? .tmp-t0044v/` 两个未跟踪目录（评审前既存的环境产物，非本轮提交、非本报告创建）。
- frontend / calc / ingest / resources / contract / HMAC 鉴权（openapi/auth、openapi/filter、requireIdentity、OpenApiAuthFilter）零触碰。越界 0，符合「最小修复」。

## 2. I-1 真修复核实（回到源码逐处独立核对）

证据源：`backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/service/impl/OpenApiQueryServiceImpl.java`。`VisibleScope.java:36-38` 确认 `VisibleScope.allTenant(boolean)` 的 `instanceIds` 恒 `Set.of()`；`DataPermServiceImpl.computeUserScope():117-121`（scope=ALL 角色 → `VisibleScope.allTenant(...)`）与 `:241-242`（`clientScope(...)` → `new VisibleScope(false, Set.of(), visibleIds, false)`）分别确认管理面可出 allTenant=true、HMAC 恒 allTenant=false。

| 位置 | 方法 | HEAD 实际代码 | 判定 |
| --- | --- | --- | --- |
| `:183`（对应报告 :182） | `itemsRealtime` | `if (item == null \|\| (!scope.allTenant() && !scope.instanceIds().contains(item.getInstanceId())))` | allTenant=true 跳过集合裁剪，正确 |
| `:277`（对应报告 :275） | `metricsQuery` 装载回显实例集 | `if (scope.allTenant() \|\| scope.instanceIds().contains(row.getInstanceId()))` | allTenant=true 收全部实例进 `instanceIds`，正确（必要性见 §3） |
| `:288`（对应报告 :285） | `metricsQuery` 逐码裁剪 | `if (metric == null \|\| (!scope.allTenant() && !scope.instanceIds().contains(metric.getInstanceId())))` | allTenant=true 跳过集合裁剪，正确 |
| `:399`（对应报告 :395） | `requireInstanceVisible(Long, Long, VisibleScope, String)` | `if (instanceId == null \|\| (!scope.allTenant() && !scope.instanceIds().contains(instanceId)))` | allTenant=true 跳过集合检查；`instanceId == null` 仍放在 `\|\|` 前段、无条件拒绝，正确保留 |

- 与 `:118` 已存在的正确范式 `if (!scope.allTenant() && scope.instanceIds().isEmpty())` 同构：先判 `allTenant()` 再走集合语义。受限 scope（allTenant=false、含 denied()）行为逐字未变。
- `scope == null` 收敛未改：`:166/:228/:263` 三处共享核心入口仍 `scope = VisibleScope.denied()`；`IotDisplayReadServiceImpl.sessionScope():125-126` 仍 `scope == null ? denied() : scope`。null-scope fail-closed 路径保持。
- 未改 `resolveScope`（`:387-389`，仍 `resolveClientScope(...)`）、`:329 points/history`（仍 `resolveScope(identity).instanceIds().contains(...)`）、`:118 modelsTree`、`:151/:196/:243` 三个 HMAC identity 入口委托、dictSync、status 读口。

## 3. `:275` 附带修复：必要且正确（独立实测）

执行者称「不修则 allTenant 下 instanceCode 仍空、指标读口不闭合」。独立核实结论：**成立**。

- 代码链：`metricsQuery` 在 `:282` 用装载好的 `instanceIds` 调 `loadInstanceCodes(tenantId, instanceIds)`（`:301-310`，内部 `instanceMapper.selectByIdsForTenant`），得到 `instanceCodeById`；`:292` 把 `instanceCodeById.get(metric.getInstanceId())` 传给 `timeseriesClient.queryMetrics(...)`；后者在 `OpenApiTimeseriesClient.java:227` 用 `vo.setInstanceCode(instanceCode)` 回显，最终由 `IotDisplayReadServiceImpl.metricValues():96-100` 映射进 `MetricValueVO`。
- allTenant 时 `instanceIds` 恒空集；若沿用旧判定，`:302` 的 `instanceIds.isEmpty()` 为真 → `loadInstanceCodes` 直接 `return Map.of()` → `instanceCodeById.get(101)` 为 null → 指标值即使查到，`instanceCode` 回显为空，映射链不闭合。
- **实测反证**：在 HEAD 基础上仅把 `:277` 这一处回退为旧判定（保留 :183/:288/:399 三处修复），跑 `mvn -pl ot-admin -Dtest=IotDisplayReadAllTenantContractTest test` → `Tests run: 1, Failures: 0, Errors: 1` / BUILD FAILURE，栈为 `NullPointerException: Cannot invoke "OpenApiPage.rows()" because "page" is null` at `OpenApiQueryServiceImpl.java:293`。根因正是 `instanceCode` 传 null 使测试中 `eq("F01")` 的 mock 不命中。（生产 `queryMetrics` 不会返回 null，表现是 instanceCode 空而非 NPE；测试因精确 mock 而更早暴露。无论表现形态，不修 :275 则指标读口确实不闭合。）
- 恢复 HEAD 源文件后 sha256 回到 `8c86b8ea...d92a`，`git status` 无 tracked 改动。
- 判定：`:275` 属 I-1 同一缺陷的必要组成（allTenant 感知在指标路径的完整落地），非越界、非多余改动。

## 4. HMAC 零回归核实

- `DataPermService.resolveClientScope(Long, Long, String)` javadoc（`DataPermService.java:76`）明确 `allTenant 恒 false`；实现 `DataPermServiceImpl.resolveClientScope():204-242` 全部返回 `clientScope(...)` = `new VisibleScope(false, Set.of(), visibleIds, false)` 或 `denied()`，无 `allTenant(true)` 出口。
- 因此新增的 `!scope.allTenant()` 分支对 HMAC 恒为 true，必然落入原集合裁剪逻辑，行为逐字不变；而 HMAC 六接口本就不走 `:277` 的 allTenant 分支（scope 非 allTenant）。
- `resolveScope` 与 `:329 points/history` 未改（已与 `git show 578753f:...` 对照确认）。
- 回归测试类在库且全绿：`OpenApiQueryServiceImplTest`（六接口契约 + scope 裁剪三态 + 6003 + 限值）、`DataPermClientScopeTest` / `DataPermClientScopeTenantTest` / `ClientScopeRedlineTest` / `HmacAuthStrategyTest` / `OpenApiAuthFilterTest` / `OperateLogCoverageGuardTest` / `IotPermissionsCompleteGuardTest` 等（全量见 §5）。

## 5. 测试有效性与回归实测（红转绿独立复现）

测试文件：`IotDisplayReadAllTenantContractTest.java`（新增，156 行）。

- **真实性**：`setUp()` 构造真实 `OpenApiQueryServiceImpl` + 真实 `IotDisplayReadServiceImpl`，仅 mock mapper/store/DataPermService；`when(dataPermService.resolve(eq(7L))).thenReturn(VisibleScope.allTenant(false))`（`:140`）；`StpUtil.login("7")` + session `tenant_id=1` 使 `SecurityUtils.getUserId()`=7、`TenantContext.getTenantId()`=1（已核对 `SecurityUtils.java:16-18`、`TenantContext.java:37-53`）。走的是管理面 `sessionScope()` → 共享核心真实判定链，非 mock 捷径。
- **非恒真**：断言为 `assertFalse(realtime.isEmpty())`、`assertEquals(23.5, v)`、`assertEquals(1, metrics.size())`、`assertEquals(412.6, value)`、`assertEquals(0.083, yoy)`、`assertNotNull(ts)`（`:148-153`）；依赖 fake mapper 返回行并通过核心裁剪，旧版必失败。
- **红（独立复现）**：临时把源文件回退到 `578753f` 版本（仅反向应用 source hunk）后跑定向测试 → `Tests run: 1, Failures: 1, Errors: 0`、`BUILD FAILURE`；失败点 `:148`：`allTenant 超管经管理面实时读口必须取到数据 … expected: <false> but was: <true>`（realtime 空集）。与修订报告 R2 自述一致。
- **绿（独立复现）**：恢复 HEAD 后跑定向测试 → `Tests run: 1, Failures: 0, Errors: 0, Skipped: 0`、`BUILD SUCCESS`（12:37:29）。
- **全量回归（独立复跑）**：`cd backend && mvn -pl ot-admin test` → `Tests run: 2577, Failures: 0, Errors: 0, Skipped: 0` / `BUILD SUCCESS`（12:41:30）。主体基线 2576 + 新增 1，与报告 R4/R6 自述一致。
- 注：定向测试「红」由 realtime 断言先触发；`:395` 单目标路径未在本轮新增测试直接断言，但其修复经源码逐处核对正确、且 HMAC/scope 外既有测试与全量回归未受影响（见 §6 观察项）。

## 6. 观察项（不构成发现，不影响结论）

1. `:395` 的 allTenant 分支在新增契约测试中未被独立触发（测试覆盖 realtime `:183` 与 metric `:277/:288`）。该修复与 `:183/:288` 同构、已由源码核对，且 `itemsHistory` scope 外 6003、`points/history` 6003 等既有回归全绿；属于测试覆盖收窄而非逻辑缺陷。鉴于终审无退回额度、核心缺陷已闭合，不作为阻塞项。
2. `:275` 单独回退时的失败形态是 mock 不命中导致的 NPE（非生产语义的「instanceCode 为空」），系测试用 `eq("F01")` 精确匹配所致；测试 thereby 更早暴露问题，不影响「必须修」的结论，也不需改测试。
3. 工作区两个未跟踪目录为评审前既存环境产物，与本轮提交无关；评审未创建、未删除任何项目文件（报告除外），源码 sha256 已于实验后复原。

## 7. 最终判定

- I-1 是否闭合：**是**（三处 + 必要的 :275 附带修复均 allTenant 感知，`:395` null 拒绝保留）。
- `:275` 附带修复是否必要且正确：**必要且正确**（实测反证见 §3）。
- 是否有新 C/I：**否**（无 C/I/M）。
- 结论：**通过（PASS）**。本终审仅输出本报告，不改代码、不推进 TASK 状态、不派子代理。
