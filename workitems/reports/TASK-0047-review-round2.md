# TASK-0047 scoped 复评 round2 报告（代码评审）

- 评审者：fresh 代码评审子代理（no-subagents；只评审、只写本报告，未改动任何源码 / workitems）
- 口径：L1-execute §13「diff 评审」复评条款；**scoped**——只看修复 diff `f19edc1..ea34d02`（限 `calc/`，10 文件）与原评审 F-01/F-02/F-03，不重开原评审已通过部分
- 证据来源：brief 指定 4 份文件（diff 包 / 原评审 / 修订轮报告 / 实施 brief）+ 工作区 `D:\workspace\WisFactory\calc\src\**` 源码；独立执行 `mvn test` + surefire XML 聚合核对测试数；`git log`/`git show`/`git diff` 核对提交范围与隔离
- 独立核验动作：`grep -rn "t47-live" calc/src/` → NONE；`mvn test`（退出码 0，surefire 聚合 275/0/0/3）；git 范围与提交内容核对（见下）
- live 冒烟：本评审未独立起服务复跑（有状态、依赖 8095 + 注入缝 + env 阈值），live 结论依修订轮报告原始输出 + 契约测试佐证，判「报告自证充分、未独立验证」，不作为发现

---

## 一、三问逐条结论

### 问1 · F-03（租户硬编码 + 冒烟值泄漏）是否真修复 —— **通过**

**结论**：真修复，生产代码零硬编码租户/冒烟值残留，tenant 从请求贯穿到计算与状态键，跨租户查 task 返回 404。

**出处与核验**：
- 生产代码零 `"t47-live"`：全仓 `grep -rn "t47-live" calc/src/` → **NONE**（`t47-live`/`t47live` 变体均 NONE）；修复 diff 中 `CalcEngineApplication.java` 删除 `recomputeWindow("t47-live",...)`，改 `ctx.tenant()`（`CalcEngineApplication.java:269`）。全 diff 10 文件扫描未见其他租户字面量或冒烟值注入生产路径（唯一残留 `1L` 是 `CalcProperties.tenantId=1L`，属 f19edc1 既有单租户配置默认、不在本次修复 diff 内，且经 brief 登记为演进位，非冒烟泄漏）。
- tenant 贯穿链路（源码逐跳确认）：
  - 请求入口 `CalcRecalculateController.recalculate` 取 `req.tenant()` → `taskManager.submit(req.tenant()...)`（`CalcRecalculateController.java:72-73`）；
  - `CalcTaskManager.submit` 归一化空→`default`，写入 `CalcTask.tenant` 与 taskId（`CalcTaskManager.java:168-172`）；
  - `run()` → `long tenantId = tenantIdOf(tenant)`（`:226`，取代旧 `1L`），写入 `RecalcContext(taskId,triggerType,true,tenant,tenantId,...)`（`:227`）；
  - 生产 lambda 用 `ctx.tenant()` 调 `metricRuntime.recomputeWindow(ctx.tenant(),...)`（`CalcEngineApplication.java:269`）；`MetricRuntime.recomputeWindow` 把 tenant 用于 `new MetricWindow(tenant,...)`、calcLog 键与 `doCompute`（`MetricRuntime.java:247-266`），确认 tenant 真正参与计算，非仅传参丢弃；
  - 状态键 `onSuccess/onFailure(ctx.tenantId(),...)` 写注入 store（`CalcEngineApplication.java:272-282`），按 tenantId 分区。
- `getTask(tenant, taskId)` 归属校验：`CalcTaskManager.java:301-308`（仅 `effTenant.equals(t.tenant())` 返回，否则 null）；HTTP 层 `CalcRecalculateController.java:92-96` 对 null 返回 404「task 不存在或不属于该 tenant」。契约测试 `CalcTenantIsolationContractTest.crossTenantTaskQueryIsRejected`（:70-82）断言跨租户 `"tenantB"`/`"default"` 均 null；HTTP 测试 `CalcRecalculateControllerTest.java:93` 断言跨租户 `?tenant=otherTenant` → 404。
- `recentTasks` 按租户过滤：`CalcStatusController.java:127-128`（`.filter(t -> effTenant.equals(t.tenant()))`），状态页只显示本租户任务。
- live 实证（依报告原始输出）：tenantA status `tenantId=1` / tenantB `tenantId=2`（键隔离）；跨租户 `GET /calc/v1/task/MANUAL-tenantA-000001?tenant=tenantB` → 404。
- **对旧实现必红**：`CalcTenantIsolationContractTest.java:20-21` javadoc 明示旧实现 `RecalcContext` 无 tenant 字段、无 tenant-aware `getTask`、`run()` 写死 1L——编译期 + 真实断言双重锁定。

### 问2 · F-01（状态 2 空接线）是否真修复 —— **通过**

**结论**：真修复，env 阈值经装配真正传入状态机并在生产状态查询路径调用 `evaluateLag` 产生 calc_status=2；`CalcStatusVo.from` 阈值参数用于 lagging 标志；测试对旧实现必红。

**出处与核验**：
- env 阈值真正传入状态机（非仅 log）：`CalcEngineApplication.calcTaskManager` 读 `properties.getCalcLagThresholdMs()` → `lagMs`（`:258-259`），经 **6 参全注入构造** `new CalcTaskManager(fn, calcStateStore, threshold, lagMs, workers, capacity)`（`:260,285`）传入；`CalcTaskManager` 完整构造 `this.lagThresholdMs = lagThresholdMs` 并 `new CalcStateMachine(stateStore, errorStopThreshold, lagThresholdMs)`（`CalcTaskManager.java:100-106`）；`CalcStateMachine` 持有 `this.lagThresholdMs` 于 `evaluateLag`（`CalcStateMachine.java:23-28,94-113`）。旧实现的「仅 `if (lagMs!=15min) log.info` + 4 参构造写死 5min」已删除。env 映射链完整：`application.yml:63` `calc-lag-threshold-ms: ${CALC_CALC_LAG_THRESHOLD_MS:900000}` → `CalcProperties.getCalcLagThresholdMs()`（:148-149）→ 上述装配。
- 状态查询前调 evaluateLag 产生 calc_status=2：`CalcStatusController.status()` 在 fullKey 分支 `calcTaskManager.evaluateLag(tenantId, fullKey, now)`（`:114`）、无 fullKey 分支对每 state 调 `evaluateLag`（`:119-124`）；生产缝 `CalcTaskManager.evaluateLag`（`:314-316`）委派状态机。`CalcStateMachine.evaluateLag` 以 `now-watermark > lagThresholdMs` 置 calc_status=2、不覆盖停算 3（`:102-112`）。原评审指出「全仓 `calc/src/main` 仅命中方法定义、零生产调用」的空接线已消除。
- `CalcStatusVo.from` 阈值参数真正生效：`CalcStatusVo.java:25-31` 新增 `boolean lagging = s.watermark()>0 && lag > Math.max(0,lagThresholdMs)`，字段进 record 与 `toMap()`（`:17,44`），不再为死参数。测试 `CalcStatusViewContractTest.statusVoConsumesLagThresholdForLaggingFlag`（:82-93）：同 state 用阈值 1000ms→`lagging=false`、200ms→`lagging=true`，并断言 map `lagging` 值。
- live 实证（依报告原始输出）：`CALC_CALC_LAG_THRESHOLD_MS=200` 启动日志「阈值=2、滞后阈值=200ms」；`GET /calc/v1/status?fullKey=M_LOW&tenant=tenantA` 返回 `calcStatus=2 / calcStatusText=滞后 / lagMillis=86401447 / lagging=true / stopped=false`。
- **对旧实现必红**：`CalcTaskManagerContractTest.statusPathAppliesLagThresholdEnvAndProducesCalcStatus2`（+:2 项之一）构造 200ms 阈值、`evaluateLag(...,700L)` 断言 `calcStatus=2`，javadoc 明示旧实现零调用/写死 5min 必红；`CalcStatusViewContractTest` 阈值翻转断言对旧死参数必红。

### 问3 · F-02（CalcStateStore 硬编码）是否真修复 + 无新 C/I —— **通过**

**结论**：真修复——`@Bean` + 构造注入替代 `new InMemoryCalcStateStore()`；`@TestFactory` 同套契约跑内存与 JDBC 替身两实现；「0063 换实现零代码改动」达成（消费侧零改动）；275 测试零回归；F-04/05 deferred 留痕真实。无新 C/I。

**出处与核验**：
- `@Bean` + 构造注入：`CalcEngineApplication.calcStateStore()`（`:233-236`）提供默认 `InMemoryCalcStateStore` Bean；`calcTaskManager(..., CalcStateStore calcStateStore, ...)`（`:247-250`）方法注入，传给 6 参构造（`:285`）；`CalcTaskManager` 新增完整注入构造（`:100-109`），4 个兼容重载委托 `new InMemoryCalcStateStore()` 仅为测试/兼容保留，**生产装配走注入路径**。原评审 `CalcTaskManager.java:87` 的硬编码已移入兼容重载，生产 bean 链路无硬编码。
- 消费侧零改动：`CalcTaskManager.stateStore` 与 `CalcStateMachine.store` 均为接口类型（`CalcTaskManager.java:62`、`CalcStateMachine.java:21`），`CalcStatusController` 经 `stateStore()` getter 读接口；0063 换 JDBC 只需替换 `calcStateStore()` Bean（javadoc 明示 `@Primary` 或 `@ConditionalOnProperty`，`:229-231`），管理器/状态机/VO/控制器不改动。
- `@TestFactory` 同套契约两实现：`CalcStateStoreReplaceabilityContractTest`（新文件）`JdbcLikeStateStore` 仅实现接口（`:27-43`），`sameContractRunsOnAnyImplementationWithoutProductionChange`（`:50-81`）对内存实现 + 替身各跑 2 条契约（upsert/get 与租户隔离、同 uk 覆盖不追加），共 4 动态测试；断言 `assertNull(store.get(2L,"T.A"))`、`assertEquals(1, store.all().size())` 均为真实断言，非 `assertInstanceOf` 恒真。若测试直接 new 内存实现则替身必红——证明零改动替换成立。补充 `CalcTaskManagerContractTest.injectedStateStoreIsUsedByManagerAndMachine`（+:1）断言 `store.get(tenantIdOf("tenantA"),"M.A") != null`，证明写入落在注入 store 而非内部硬编码实例。
- **275 测试零回归（独立核验）**：本评审独立执行 `mvn test`（退出码 0），surefire 41 报告聚合 **tests=275 / failures=0 / errors=0 / skipped=3**（3 skip 为 0041/0049 既有 env 门控）。新增类计数：Replaceability 4 / TenantIsolation 3 / CalcStatusViewContract 5（原 4 +1）/ CalcTaskManagerContract 7（原 5 +2）/ CalcRecalculateControllerTest 1。275 = 265 基线 + 10 新增，与原评审基线一致，无回归。
- **F-04/05 deferred 留痕真实**：报告「修订轮二」明确「F-04/F-05（Minor，deferred 不修，报告登记）」（:391）；源码核对 F-04 拒绝计数翻倍仍在（handler `:131` + catch `:184` 各 `rejected.incrementAndGet()`）、F-05 `topoOrder` 环回退仍 `return new ArrayList<>(fullKeys)`（`:216-219`）——与「deferred 不修」声明一致，留痕真实，无隐瞒。两项均属原评审 Minor，按 scoped 复评口径不作为新发现重开。
- 范围核对：`git show --name-only ea34d02` = 10 文件全 `calc/**`、越界 0；`git log` 显示 `f19edc1..ea34d02` 间夹入 `a230f91`（TASK-0053，22 文件全 `backend/ot-admin/**`），但其为独立提交、与本修复无文件交集，属「backend 0053 改动已隔离」，符合 brief 越界反查。

---

## 二、新发现（scoped：仅修复 diff 与原 F-01/02/03 相关）

无 Critical / Important。

| 严重级 | 出处（绝对路径:行） | 一句话说明 |
| --- | --- | --- |
| Minor | `D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\web\CalcStatusController.java:113-117` | F-01 修复的副作用：`evaluateLag` 对**未登记** fullKey 返回默认 `calcStatus=1` 状态（`CalcStateMachine.evaluateLag:96-98`），fullKey 分支不再像旧实现「未登记则不出行」，会为任意传入 fullKey 产出一条 status-1 行。属只读观测面的行为放宽（无写入、无跨租户），不影响三问结论；如需严格「仅登记节点可见」可在 fullKey 分支先判 `store.get` 非 null 再 evaluateLag。 |

说明：该 Minor 为 F-01 接线引入的可观察行为变化（旧实现 fullKey 未登记不出行，新实现出行），不属原评审已通过部分的重开，且不构成阻断；按 scoped 复评「新发现分级」如实登记。

---

## 三、无法判定

1. **live 冒烟全序列（独立复跑）**：本评审未独立起 calc 服务（8095 + 指标定义注入 + env 阈值 200ms + 有状态时序），F-03 的 tenantA/B `tenantId=1/2`、跨查 404、F-01 的 `calc_status=2/滞后/lagging=true` 等 live 实证依修订轮报告原始输出 + 契约测试佐证，判「报告自证充分、未独立验证」。
2. **详细设计 §1.9A 字段「逐字一致」**：PDF 文本层不可读（原评审已登记），复评不重开，仍无法判定；不影响三问的租户/滞后/注入结论。
3. **0063 真 JDBC 实现 100% 零改动**：本轮契约测试用的是内存中的 `JdbcLikeStateStore` 替身（非真实 JDBC/DB），证明的是「依赖接口、消费侧零改动 + 同套契约可复跑」；真实 JDBC 实现是否有额外契约（事务/连接/ON UPDATE 语义）需 0063 落地时验证，属演进位，不影响本轮 F-02 判定。

---

## 四、结论

**通过（scoped 复评 round2）**——三问全过：F-03 真修复（生产零 `t47-live`、tenant 贯穿请求→上下文→计算→状态键、跨租户查 task 404、无其他硬编码租户/冒烟值残留）、F-01 真修复（env 阈值经 6 参构造传入状态机、生产状态查询调 evaluateLag 产生 calc_status=2、CalcStatusVo 阈值参数用于 lagging、对旧实现必红）、F-02 真修复且无新 C/I（@Bean + 构造注入、@TestFactory 同套契约跑内存/JDBC 替身、0063 换实现消费侧零改动、275 测试 0F0E3Skip 零回归、F-04/05 deferred 留痕真实）。

修复提交 `ea34d02` 10 文件全 `calc/**` 越界 0，并行 TASK-0053 的 `backend/**` 改动已隔离；仅登记 1 项 Minor（未登记 fullKey 在 fullKey 分支产出 status-1 行），不阻断。修订上限 1 退回 + 1 复评，本轮为终审；按 §13 复评通过，TASK 状态由协调者处置。本评审未修改任何源码或 workitems（仅写本报告）。
