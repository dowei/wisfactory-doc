# TASK-0086 报告：calc trial 试算端点（同引擎 · 沙箱隔离 · 缺数明示）

## 0. 结论（一句话）

已在 calc 独立 Maven 工程交付 `POST /internal/v1/calc/trial`（8095）：复用真算 ArithmeticEvaluator /
OperatorEvaluator / DeltaOperator 同引擎求值，沙箱隔离不落库，缺数 value=null + missing 明示不用 0；
19 个新增契约测试 + 既有 288 基线零回归（calc mvn test 307/0/0/3）。

## 1. 元数据

| 项 | 值 |
| --- | --- |
| 任务 | TASK-0086（计算引擎 trial 试算端点） |
| BASE | `815cabf` |
| 提交 | `33aef05c0375bbcc84c1dc1cc9f989cb88ff03c3` |
| 改动文件数 | **7**（全落声明范围） |
| 关联 | FEAT-0012 / REQ-0016 / TC-0012 / ADR-0005 / ADR-0013 增补 5 |
| 下游 | 为 TASK-0037（转发 trial）、TASK-0038（窗口曲线增强）解开「转发目标不存在」死锁 |

## 2. 改动文件清单（git diff --name-only 815cabf..HEAD）

```
calc/README.md
calc/src/main/java/net/otlab/calc/trial/TrialEngine.java
calc/src/main/java/net/otlab/calc/trial/TrialRequest.java
calc/src/main/java/net/otlab/calc/trial/TrialResponse.java
calc/src/main/java/net/otlab/calc/web/TrialController.java
calc/src/test/java/net/otlab/calc/trial/TrialControllerHttpTest.java
calc/src/test/java/net/otlab/calc/trial/TrialEngineContractTest.java
```

范围核对：backend / frontend / ingest / resources / resources/contract 零触碰；calc 既有
executor/operator/graph/metric/status/state/web 控制器实现只读复用（未修改）。无 git add -A、
无 push、未置 TASK 已完成、未改 workitems 索引。

## 3. 五项交付（D1–D4 + 契约）

### D1 端点形态
- `POST /internal/v1/calc/trial`（calc 8095，独立 Spring Boot 应用），风格仿 CalcRecalculateController
  （`@RestController` + `ResponseEntity<Map<String,Object>>`）；语法/白名单拒绝返回 HTTP 400，其余 200。
- 请求：`formula` + `variables`（number 或 `{value,quality,ts}` sample）+ 可选 `history` /
  `atTime` / `window{start,end,stepMs}`。
- 响应：`value` / `quality` / `missing` / `reason` / 可选 `curve`（逐点 ts/value/quality/missing/reason）
  / `engineVersion`（同口径标识 `calc-engine/1.0.0/trial-v1`，响应字段声明）。
- `engineVersion` 选择：以响应字段标注（而非 Header），与管理面 preview 响应规范同源，便于日志与断言。

### D2 同引擎同口径（核心）
- 单点公式经 `ArithmeticEvaluator.evaluate(formula, resolver, functionResolver)`；resolver 对缺失返回
  null → 求值器抛异常 → 捕获转成缺数响应；**无第二套算术/比较/三元实现**。
- `functionResolver.supports` 交给 `OperatorEvaluator`；DELTA 从 `history` 取一窗样本，按 ts 升序
  逐点调用 `operators.delta(key, ValueSample, null, null, false)` → 原生 DeltaOperator 五异常口径
  （正常/回退码 8/换表/溢出码 1/跨周期码 9/坏样本）原样透传 quality。
- 复合形态 `DELTA(METER)/2` 经同一 evaluator 求值（函数结果作为算术子表达式），与真算 0042
  `evaluateDeltaFormula` 仅接受纯 `DELTA(key)` 相比，trial 侧在同引擎内自然支持复合（本任务 D4 明示
  "DELTA 用两点历史→差分值正确"，复合支持为同引擎红利，不另写求值）。
- INTEGRAL 手写由 `FormulaExpressionValidator` 白名单拒绝（与真算/管理面 validate 一致），未知函数同拒。

### D3 沙箱隔离与不落库（硬约束）
- `TrialEngine` 仅依赖纯计算组件：`ArithmeticEvaluator` / `OperatorEvaluator` / `DeltaOperator` /
  `FormulaExpressionValidator`；**不注入** CalcStateStore / JdbcTemplate / MetricEventPublisher /
  调度器 / Redis / 真库查询。
- `OperatorEvaluator` 在 `evaluatePoint` / 每个曲线点调用 `operatorFactory.get()` 新建（构造器默认
  `OperatorEvaluator::new`），baselines 生命周期仅限本次求值；跨请求/跨试算互不干扰，更不影响在线算子。
- 曲线点按 `window.stepMs` 逐点求值（严格 ts 匹配取数，缺数段不沿用旧值冒充），每点独立 missing 标注。

### D4 测试口径（TDD 红转绿）
19 个新增契约测试：`TrialEngineContractTest`（14）+ `TrialControllerHttpTest`（5）。

### 契约纪律
no-subagents；只改 calc 改动范围；提交只 add calc 7 个文件（未 add -A）；未 push；不置 TASK 已完成；
未改 workitems 索引；未硬编码凭据/文档库路径。

## 4. TDD 红转绿留痕（每修复附「为什么现在对了」）

| 轮次 | 观察 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- |
| **红 1**（`mvn test -Dtest=Trial*...`） | 12 compilation errors：`TrialEngine` 类不存在 + `vars()` varargs 类型错 | 先记录红态原始输出，再补测试 helper `vars(Object...)`，新建 `TrialEngine`/`TrialController` 实现 | 红在「无实现」而非「写错期望」——说明测试确实在驱动新行为，未为匹配输出改期望 |
| **绿 1 部分** | 19 tests / 6 red：4×NPE（value=null）、1×曲线 t=3000 返回 138、1×Δ 曲线 t=1000 得 7 | 诊断：`check.identifiers()` 把仅存在于 history 的 METER 当缺数；`snapshotAt` 用「≤t 最新」填充致缺数段沿用旧值；wl 请求响应是 curve 不是 value | 修 1：history keys 视为「已提供」；修 2：曲线点严格 `s.ts()==t`，无点即 missing；修 3：区分单点 atTime 请求与 window 请求的测试构造——缺数段语义与 D4 一致后才绿 |
| **绿 1 再部分** | 1 red：DELTA 第二样本 quality=4 期望差分 8 实得 0 | 修测试为三点历史（t=3000 quality=4）并断言 BAD_INPUT 口径（value=0 + quality=4，不冒充差分 92） | 真算 DeltaOperator 五异常口径第一优先是坏样本过滤；trial 同引擎必须透传该口径，**不是**把上游坏值参与差分——现在断言与真算一致 |
| **绿 2** | 19/19 PASS | — | 见下方 Acceptance 原始输出 |

## 5. 缺数不冒充 0 证据

- `TrialEngineContractTest.missingIdentifierReportsNullValueAndNamedKeyNeverZero`：`A*2+B` 只给 A →
  `value=null`、`missing=[B]`、`reason` 含 B、`quality=6`、rejected=false（HTTP 200 业务缺数，非语法错）。
- `missingAllVariablesDoesNotReturnZero`：`A+B` 无变量 → A、B 均进 missing。
- `TrialControllerHttpTest.trialMissingIsExplicitNotZero`：HTTP 响应含 `"value":null` + `"B"` + `"quality":6`。
- 窗口曲线缺数点 `windowCurveEmitsPointPerStepAndMarksMissingSegment`：t=3000/4000 METER 无样本 →
  `CurvePoint.value=null`、`missing=[METER]`、`quality=6`（严格 ts 匹配，不沿用 t=2000 旧值）。

## 6. 沙箱隔离证据

- `TrialEngineContractTest.sandboxCreatesFreshOperatorPerRequestAndReplaysIdentically`：注入计数工厂，
  同一 `DELTA(METER)` atTime=2000 连试两次 → 两次 value=5.0 且 `created=2`（每请求 new OperatorEvaluator，
  第二次不受第一次基线污染）。
- `TrialEngineContractTest.deltaWindowCurveEachPointIsolated`：曲线三点 t=1000/2000/3000 分别为
  0 / 5 / 7（每点独立沙箱，不串扰）。
- `TrialControllerHttpTest.trialDoesNotWriteCalcStateStore`：`@MockitoBean CalcStateStore` 替换 Bean，
  trial 发 DELTA 请求后 `verify(calcStateStore, never()).upsert(any())` 通过 —— 端点不写 iot_calc_state/log。
- 静态：`TrialController` 构造只依赖 `TrialEngine`；`TrialEngine` 字段只有 `Supplier<OperatorEvaluator>`，
  无写口/事件/调度/Redis 依赖。

## 7. 同口径证据

- `sameEngineParityWithDirectArithmeticEvaluator`：`A*2+B` 经 trial 与直接调
  `ArithmeticEvaluator.evaluate("A*2+B", resolver)` 结果相等（delta < 1e-9）。
- `deltaCompositeExpressionUsesSameEvaluator`：`DELTA(METER)/2` = 3，函数结果与算术共享同一 evaluator。
- `deltaUsesHistoryWindowPointwiseAndTransportsQuality`：正常差分 8 / 坏样本码 4 原样透传，与
  `DeltaOperatorContractTest` / `OperatorIntegrationContractTest` 口径一致。
- `engineVersionIsDeclaredForParityClaim`：响应含 `engineVersion=calc-engine/1.0.0/trial-v1` 声明。

## 8. 验收命令原始输出

### 8.1 `cd calc && mvn test`（post-commit 原样 Results 段）

```
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.747 s -- in net.otlab.calc.ack.HttpAckSenderTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.484 s -- in net.otlab.calc.CalcEngineApplicationTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.036 s -- in net.otlab.calc.config.CalcPropertiesTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.040 s -- in net.otlab.calc.event.MetricProducedEventContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.011 s -- in net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.052 s -- in net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.calc.executor.OperatorIntegrationContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.008 s -- in net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.006 s -- in net.otlab.calc.graph.CycleDetectorTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.025 s -- in net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.036 s -- in net.otlab.calc.graph.GraphBuilderTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.026 s -- in net.otlab.calc.metric.CascadeRecalcContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.metric.LateArrivalRecalcContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.011 s -- in net.otlab.calc.metric.MetricAggregatorContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.metric.MetricCompanionValueContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.007 s -- in net.otlab.calc.metric.MetricSchedulerContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.005 s -- in net.otlab.calc.metric.MetricWindowContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.008 s -- in net.otlab.calc.metric.TdengineMetricValueWriterContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.007 s -- in net.otlab.calc.operator.DeltaOperatorContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.operator.EnergyDerivationContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.operator.IntegralOperatorContractTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.046 s -- in net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.244 s -- in net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.037 s -- in net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.006 s -- in net.otlab.calc.state.CalcStateMachineContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.010 s -- in net.otlab.calc.state.CalcStateStoreContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.calc.state.CalcStateStoreReplaceabilityContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.012 s -- in net.otlab.calc.state.CalcStatusViewContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.016 s -- in net.otlab.calc.state.IotCalcSchemaDdlContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 6.308 s -- in net.otlab.calc.state.JdbcCalcStateStoreContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.status.StatusEvaluatorContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.020 s -- in net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.storage.HomogeneousResultSinkContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.012 s -- in net.otlab.calc.storage.TdengineItemValueWriterContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.277 s -- in net.otlab.calc.task.CalcTaskManagerContractTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.044 s -- in net.otlab.calc.task.CalcTenantIsolationContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.161 s -- in net.otlab.calc.trial.TrialControllerHttpTest
[INFO] Tests run: 14, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.041 s -- in net.otlab.calc.trial.TrialEngineContractTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.823 s -- in net.otlab.calc.web.CalcRecalculateControllerTest
[INFO] Results:
[INFO]
[INFO] Tests run: 307, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] BUILD SUCCESS
```

基线核对：新增 19（14+5），既有 288（307−19），既有零回归。

### 8.2 `cd calc && mvn package -DskipTests`

```
[INFO] BUILD SUCCESS
[INFO] Total time:  2.772 s
```

### 8.3 范围核对

```
$ git diff --name-only 815cabf..HEAD
calc/README.md
calc/src/main/java/net/otlab/calc/trial/TrialEngine.java
calc/src/main/java/net/otlab/calc/trial/TrialRequest.java
calc/src/main/java/net/otlab/calc/trial/TrialResponse.java
calc/src/main/java/net/otlab/calc/web/TrialController.java
calc/src/test/java/net/otlab/calc/trial/TrialControllerHttpTest.java
calc/src/test/java/net/otlab/calc/trial/TrialEngineContractTest.java
```

## 9. 遗留 / 边界说明

- trial 不查真库/Redis（brief D3 明示不在本任务范围）；0037 管理面负责取数后传入 variables/history；
  0038 增强才考虑从历史存储取数。
- 曲线点严格 ts 匹配取数（缺数段不沿用旧值）；单点 `atTime` 取 ≤t 最新样本，两种语义分别测试锁定。
- 工作区 `backend/` 在本次执行期间出现并发的他人改动（`OpenApiQueryService.java` + `displayread/`），
  已识别并完整保留、未 add、未提交（只提交 calc 7 文件）；本执行者未触碰 backend/frontend/ingest/resources。
