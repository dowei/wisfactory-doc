# TASK-0047 验证报告：手动补算接口与计算状态页数据源

- 执行者：TASK-0047 执行者（no-subagents；**计算引擎道 calc/**——只改 `calc/**`）
- **BASE（开工 HEAD）= `ebd443d`**（`git rev-parse --short HEAD`，2026-11-24 main；brief「BASE=当前 HEAD」口径）；**交付提交 = `f19edc1`**（注明 TASK-0047，`git add -A calc/` 只含 `calc/**`）
- 交付提交 20 文件全部 `calc/**`，`git diff --name-only ebd443d..HEAD` 越界 0（详见 §一 / §验收 3）
- 执行日期：2026-11-24
- 结论先行：**验收 3 条全部通过**——① `cd calc && mvn test` 265 测试 0F0E3Skip BUILD SUCCESS（244 基线逐类零回归 + 21 新增）；② live 冒烟全序列通过（8095 起→recalculate 异步 taskId 立返 64ms/27ms→进度查询逐项 MANUAL 结果+拓扑序→status 水位推进/calc_status=1→连续失败阈值=2 坏根停算+下游子树隔离+兄弟正常→第三轮停算跳过、全停归零）；③ 范围核对 20 文件全落 `calc/**`、越界 0。
- 环境事实与取舍：8095 归 calc（起前探测 FREE / 用毕即停 / 终态 FREE，JVM PID 60580 用毕 `taskkill` 终止）；Redis 宿主 phpStudy 3.0.504@6379（本任务未写业务键）；TDengine REST 仍故障（TD-0053，本任务沿用记录式写口、不写真库，真机补验列条件）；**并行 backend 道 TASK-0053 在途**——本任务全程未碰 `backend/`、未跑 backend mvn（工作区 backend/** 改动为 0053 在途，未 add/未提交/未触碰）；WSL docker 未碰；契约冻结未改。

---

## 一、改动文件清单（提交 `f19edc1`，20 项，全部 `calc/**`）

### 1.1 新增——计算状态包 `calc/src/main/java/net/otlab/calc/state/`（5）

| 文件 | 职责 |
| --- | --- |
| `CalcStateStore.java` | 计算状态存储**接口**（§1.9 iot_calc_state 字段语义；唯一键 tenant_id+full_key 由 upsert 保证）；**0063 换 JDBC 实现零改动替换缝**（照 ingest/calc 假实现范式——调用点只依赖接口）。 |
| `InMemoryCalcStateStore.java` | 接口内存实现（ConcurrentHashMap，uk 覆盖语义、租户隔离、稳定排序只读视图）；并发安全（补算后台线程 + status 只读线程）。 |
| `CalcState.java` | 状态行（tenant_id/full_key/last_calc_time/watermark/calc_status 1-2-3/error_count/last_error/update_time）；INT→文本翻译（ADR-0009 TD-0005）；流畅 Builder。 |
| `CalcStateMachine.java` | 水位推进（max 单调不回退）+ 连续错误阈值停算（达阈值 calc_status=3，中间成功归零）+ 滞后判据（evaluateLag→2）+ 上游停算子树隔离标记（stopByUpstream）；阈值 env 可配（默认 5）。 |
| `CalcStatusVo.java` | **状态页只读数据源 VO**（水位/状态/滞后毫秒/连续错误/最近错误/停算布尔），`toMap()` 供 0061 消费；只读无触发口。 |

### 1.2 新增——补算任务包 `calc/src/main/java/net/otlab/calc/task/`（3）

| 文件 | 职责 |
| --- | --- |
| `CalcTaskManager.java` | **异步任务核心**：submit 立返 taskId、独立线程池（默认 2 workers）+ 有界队列（默认 64，超限拒绝不堆积拖垮在线计算）、0040 `GraphBuilder.topoSort` 拓扑序执行（复用 0046 MetricCascade 图结构）、逐项 MANUAL 留痕缝（RecalcContext.triggerType/isRecalc/taskId/stateMachine）、连续失败停算 + 沿依赖图向下游子树隔离（兄弟不波及）、进度快照（CalcTask 不可变对象替换）。 |
| `CalcTask.java` | 任务进度快照（QUEUED/RUNNING/COMPLETED/FAILED + total/done/failed + 逐项 ItemResult 成功/失败明细）；§1.9A task_id 落库持久化缝。 |
| `CalcTaskProgressVo.java` | 任务进度**只读 VO**（§3.4 GET task/{taskId}），`toMap()` 供 0061 消费。 |

### 1.3 新增——内网接口 `calc/src/main/java/net/otlab/calc/web/CalcRecalculateController.java`（1）

内网三接口：`POST /calc/v1/recalculate`（body fullKeys[]+startTime/endTime+triggerType → 异步 taskId 立返）、`GET /calc/v1/task/{taskId}`（进度+逐项结果）、status 数据源落在既有 `CalcStatusController` 的 `calcState` 段（见 1.4，避免 `/calc/v1/status` 重复映射）。**管理面入口最小化（ADR-0010）**：本口落 calc 内网 `/calc/v1/**`、无鉴权；契约 §3.4 `/internal/v1/calc/**` 的管理面转发 + 权限 + 审计归 0061（javadoc 登记接线义务）。

### 1.4 修改（6）

| 文件 | 修改内容 |
| --- | --- |
| `CalcEngineApplication.java` | 新增 `calcTaskManager` Bean（recalculate-enabled=true 才装配，零回归）：计算函数接 `MetricRuntime.recomputeWindow`（复用 0046 doCompute + 拓扑级联 + MANUAL 留痕），成功/失败驱动 CalcStateMachine 维护 iot_calc_state 内存态；启动 runner 启动后台线程池。 |
| `config/CalcProperties.java` | 5 项 env：recalculate-enabled（默认 false）/ calc-error-stop-threshold（默认 5）/ recalc-queue-capacity（默认 64）/ recalc-workers（默认 2）/ calc-lag-threshold-ms（默认 900000）。 |
| `metric/MetricRuntime.java` | 新增 `recomputeWindow`（TASK-0047 手动补算执行缝：按 fullKey 对窗口重算，复用 doCompute + 0046 级联闭包，逐项 triggerType MANUAL/BACKFILL + is_recalc=1 留痕；异常上抛交 task 层停算判定）。 |
| `web/CalcStatusController.java` | `GET /calc/v1/status?fullKey=` 增 `calcState` 只读数据源段（水位/状态/滞后/连续错误/近期任务）；fullKey 可选过滤；fullKey 参数新增不改变既有 status 响应结构（零回归）。 |
| `resources/application.yml` | 5 项 env 映射。 |
| `test/config/CalcPropertiesTest.java` | +1 项 TASK-0047 默认值零回归断言。 |

### 1.5 新增——契约测试（5 类 21 项）

| 文件 | 用例数 | 覆盖 |
| --- | --- | --- |
| `state/CalcStateStoreContractTest.java` | 4 | 内存实现是接口实例、租户隔离、uk 覆盖不追加、按租户稳定排序、未登记返回 null。 |
| `state/CalcStateMachineContractTest.java` | 6 | 成功推进水位+重置错误+恢复、连续错误阈值停算（仅该 fullKey）、连续非累计、停算持续至显式恢复、滞后判据、水位单调不回退。 |
| `state/CalcStatusViewContractTest.java` | 4 | VO 字段/文本翻译/滞后/停算布尔、无水位、任务进度 VO 逐项结果、Map 序列化供 0061。 |
| `task/CalcTaskManagerContractTest.java` | 5 | taskId 立返+进度查询、拓扑序从物化结果执行（乱序提交→下级先算）、MANUAL+isRecalc+taskId 留痕、连续失败停算子树隔离兄弟不受影响、有界队列拒绝限流。 |
| `web/CalcRecalculateControllerTest.java` | 1 | HTTP 三接口端到端（recalculate 立返 taskId → task 进度逐项 → status calcState 只读源 → 不存在 task 404）。 |

新增用例合计 21（4+6+4+5+1+配置 1 已计入 = 见下）：实际新增 = 状态 4+6+4=14、任务 5、HTTP 1、配置 1 = 21；244 基线 → 265。

---

## 二、验收命令原始输出

### 验收 1：`cd calc && mvn test` → BUILD SUCCESS、0 失败 0 错误

终验（提交 `f19edc1` 对应源码）逐类 Results 段（CR 行尾已去）：

```
[INFO] Running net.otlab.calc.ack.HttpAckSenderTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.176 s -- in net.otlab.calc.ack.HttpAckSenderTest
[INFO] Running net.otlab.calc.CalcEngineApplicationTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 5.802 s -- in net.otlab.calc.CalcEngineApplicationTest
[INFO] Running net.otlab.calc.config.CalcPropertiesTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.093 s -- in net.otlab.calc.config.CalcPropertiesTest
[INFO] Running net.otlab.calc.event.MetricProducedEventContractTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.073 s -- in net.otlab.calc.event.MetricProducedEventContractTest
[INFO] Running net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.033 s -- in net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Running net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Running net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.126 s -- in net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Running net.otlab.calc.executor.OperatorIntegrationContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.033 s -- in net.otlab.calc.executor.OperatorIntegrationContractTest
[INFO] Running net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.calc.executor.PeriodicSchedulerContractTest
[WARNING] Tests run: 1, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0 s -- in net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[INFO] Running net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.023 s -- in net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Running net.otlab.calc.graph.CycleDetectorTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.graph.CycleDetectorTest
[INFO] Running net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.059 s -- in net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Running net.otlab.calc.graph.GraphBuilderTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.061 s -- in net.otlab.calc.graph.GraphBuilderTest
[INFO] Running net.otlab.calc.metric.CascadeRecalcContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.032 s -- in net.otlab.calc.metric.CascadeRecalcContractTest
[INFO] Running net.otlab.calc.metric.LateArrivalRecalcContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.014 s -- in net.otlab.calc.metric.LateArrivalRecalcContractTest
[INFO] Running net.otlab.calc.metric.MetricAggregatorContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.016 s -- in net.otlab.calc.metric.MetricAggregatorContractTest
[INFO] Running net.otlab.calc.metric.MetricCompanionValueContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.015 s -- in net.otlab.calc.metric.MetricCompanionValueContractTest
[INFO] Running net.otlab.calc.metric.MetricSchedulerContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.012 s -- in net.otlab.calc.metric.MetricSchedulerContractTest
[INFO] Running net.otlab.calc.metric.MetricWindowContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.012 s -- in net.otlab.calc.metric.MetricWindowContractTest
[INFO] Running net.otlab.calc.metric.TdengineMetricValueWriterContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.018 s -- in net.otlab.calc.metric.TdengineMetricValueWriterContractTest
[INFO] Running net.otlab.calc.operator.DeltaOperatorContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.012 s -- in net.otlab.calc.operator.DeltaOperatorContractTest
[INFO] Running net.otlab.calc.operator.EnergyDerivationContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.026 s -- in net.otlab.calc.operator.EnergyDerivationContractTest
[INFO] Running net.otlab.calc.operator.IntegralOperatorContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.calc.operator.IntegralOperatorContractTest
[INFO] Running net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.080 s -- in net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Running net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.558 s -- in net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Running net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.012 s -- in net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Running net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.080 s -- in net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Running net.otlab.calc.state.CalcStateMachineContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.022 s -- in net.otlab.calc.state.CalcStateMachineContractTest
[INFO] Running net.otlab.calc.state.CalcStateStoreContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.017 s -- in net.otlab.calc.state.CalcStateStoreContractTest
[INFO] Running net.otlab.calc.state.CalcStatusViewContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.021 s -- in net.otlab.calc.state.CalcStatusViewContractTest
[INFO] Running net.otlab.calc.status.StatusEvaluatorContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.035 s -- in net.otlab.calc.status.StatusEvaluatorContractTest
[WARNING] Tests run: 2, Failures: 0, Errors: 0, Skipped: 2, Time elapsed: 0 s -- in net.otlab.calc.status.StatusRedisLiveTest
[INFO] Running net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.026 s -- in net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Running net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.039 s -- in net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] Running net.otlab.calc.storage.HomogeneousResultSinkContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.027 s -- in net.otlab.calc.storage.HomogeneousResultSinkContractTest
[INFO] Running net.otlab.calc.storage.TdengineItemValueWriterContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.022 s -- in net.otlab.calc.storage.TdengineItemValueWriterContractTest
[INFO] Running net.otlab.calc.task.CalcTaskManagerContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.257 s -- in net.otlab.calc.task.CalcTaskManagerContractTest
[INFO] Running net.otlab.calc.web.CalcRecalculateControllerTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.024 s -- in net.otlab.calc.web.CalcRecalculateControllerTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 265, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  15.976 s
EXIT=0
```

**244 基线逐类零回归**：244 = 0046 修订轮报告终验全部类（ack 8 / app 1 / config 10→11〔本任务 +1〕/ event 9 / executor 4+5+10+4+2+1skip+2 / graph 7+17+23 / metric cascade 2+late 6+agg 8+companion 6+sched 4+window 7+writer 5 / operator 6+5+4 / runtime 11 / snapshot 13+9+20 / status 8+2skip+8+5 / storage 5+7）；本轮每类数值与 0046 修订轮报告逐字一致（仅 CalcPropertiesTest 10→11 因本任务 +1 断言）。3 Skipped = 0041 env 门控 1 + 0049 env 门控 2（未改这些写口）。新增 21 项（状态 14 + 任务 5 + HTTP 1 + 配置 1）。

**TDD 红转绿留痕（每个修复附「为什么现在对了」）**：

1. **RED（编译期）**：先写 5 个契约测试类 + 配置断言 → `mvn test` `[ERROR] COMPILATION ERROR`（CalcStateStore / CalcStateMachine / CalcTaskManager / CalcTask / CalcStatusVo / CalcTaskProgressVo / CalcProperties 新 getter 等 60+ 处「找不到符号」）。GREEN：实现上述类与方法后消除编译错误。
2. **RED（真实断言，CalcTaskManagerContractTest 30 项 2 失败）**：① `manualRecalcLogsTriggerTypeManualAndIsRecalcOne` expected `isRecalc=1` but was `isRecalc=true`（测试字符串期望与 boolean 表示不符）；② `consecutiveFailuresStopOnlySubtreeNotSiblings` expected aFails=2 but was 1（阈值=2 的连续失败跨两轮才达阈值，测试误设单轮断言）。GREEN 修复与原因：① 断言改为 `isRecalc=true`，并 javadoc 明示「is_recalc=1 落库由 0063 映射」——为什么现在对了：内存态用 boolean 表达 is_recalc 语义，DDL TINYINT(0/1) 映射是 0063 落库职责，契约测试断言内存真值而非物理列字面；② 测试改为 3 轮（轮1 失败1次未停→轮2 达2次停算+子树跳过→轮3 整体跳过），断言与「连续错误阈值」定义一致——为什么现在对了：error_count 是连续累计，阈值 2 需两次失败才触发，单轮内 M_A 只失败一次（M_A_CHILD 因未达阈值仍执行、计的是自己的 count），三轮序列真实模拟生产重试节奏。
3. **RED（Spring 上下文）**：首个全量运行 `CalcRecalculateControllerTest` + `CalcEngineApplicationTest` 均 `Failed to load ApplicationContext`，根因 `Ambiguous mapping: GET /calc/v1/status`（新控制器与既有 CalcStatusController 撞映射）。GREEN：删新控制器的 status 映射，把 calc-state 只读数据源并入既有 `CalcStatusController.status()` 的 `calcState` 段（fullKey 参数）——为什么现在对了：单一映射端点承载全部 status 观测面（含新增 calcState），避免同路径双 handler；既有 zodiac 响应结构不破坏，零回归。
4. **RED（端口）**：Second full run `PortInUseException: 8095`——`ServerPortCustomizer` 按 `calc.http-addr` 默认 `:8095` 覆盖 RANDOM_PORT，前一轮 TIME_WAIT 阻断重绑。GREEN：新测试 `@TestPropertySource` 设 `calc.http-addr=:18095`——为什么现在对了：测试夹具显式指定专用端口，避开默认 8095 的时序争用；既有测试与生产默认 8095 不变（零回归）。
5. **RED（live 拓扑断言）**：live S3 拓扑断言写死 `M_LOW,M_HIGH,M_TOP,M_OK`，实测 `M_LOW,M_OK,M_HIGH,M_TOP`。修正为校验依赖偏序（index(M_LOW)<index(M_HIGH)<index(M_TOP)）——为什么现在对了：`GraphBuilder.topoSort` 的 Kahn 序中无依赖的 M_OK 进入首批 ready 集是合法拓扑序，偏序断言才正确表达「下级先算」约束。

### 验收 2：live 冒烟全序列（每步原始输出）

**步骤 0：起前端口 + calc 启动（recalculate/metric 双开关 + 阈值 2）**

```
8095 起前探测 FREE；Redis 宿主 phpStudy 3.0.504@6379（本任务未写业务键）
$ java -jar target/calc-engine.jar   # env: CALC_RECALCULATE_ENABLED=true CALC_METRIC_AGGREGATION_ENABLED=true CALC_ERROR_STOP_THRESHOLD=2 CALC_RECALC_WORKERS=2 CALC_RECALC_QUEUE_CAPACITY=32 CALC_TENANT=t47live
INFO ... TomcatWebServer : Tomcat started on port 8095 (http) with context path '/'
INFO ... CalcEngineApplication : Started CalcEngineApplication in 4.377 seconds
INFO ... CalcEngineApplication : [CALC] 指标聚合调度已启用（自实现调度，非 XXL-Job；graceDays=3）
INFO ... CalcEngineApplication : [CALC] 手动补算异步任务已启用（独立线程池 + 有界队列；阈值=2）
```

**步骤 S0：起前 status（无状态）**

```
GET /calc/v1/status → HTTP 200
{"status":"empty","lastAttempt":{"ok":false,"error":"拉取失败: snapshot: 快照拉取失败（后端不可达）: null",...},"counters":{...},"executor":{...},"calcState":{"enabled":true,"states":[],"recentTasks":[]}}
```

**步骤 S1：注入指标定义 + 物化基线（观测驱动缝；拓扑 M_LOW←M_HIGH←M_TOP + 兄弟 M_OK）**

```
POST /calc/v1/metric/observe → HTTP 200
{"windows":{"M_OK":{"completeness":1.0,"noData":false,...},"M_HIGH":{...},"M_TOP":{...},"M_LOW":{...}},"produced":4,...}
```

**步骤 S2：recalculate 异步提交（taskId 立返）**

```
POST /calc/v1/recalculate {"tenant":"t47live","fullKeys":["M_LOW","M_HIGH","M_TOP","M_OK"],"startTime":...,"endTime":...,"reason":"TASK-0047 live smoke","triggerType":"MANUAL"}
→ HTTP 200 {"ok":true,"taskId":"MANUAL-t47live-000001","accepted":4}
taskId=MANUAL-t47live-000001 elapsed_ms=64  （<1000ms，异步立返）
（复跑一轮：taskId=MANUAL-t47live-000002 elapsed_ms=27）
```

**步骤 S3：进度查询（逐项 MANUAL 结果 + 拓扑序）**

```
GET /calc/v1/task/MANUAL-t47live-000002 → status=COMPLETED total=4 done=4 failed=0
items=[{"fullKey":"M_LOW","periodStart":1795299537615,"periodEnd":1795385937615,"success":true,"error":null},
       {"fullKey":"M_OK","...","success":true},
       {"fullKey":"M_HIGH","...","success":true},
       {"fullKey":"M_TOP","...","success":true}]
拓扑偏序断言：index(M_LOW)<index(M_HIGH)<index(M_TOP) → PASS（下级先算，0040 拓扑序复用）
逐项 triggerType=MANUAL + is_recalc=1（CalcLogStore 同窗留痕由契约测试逐字段锁定）
```

**步骤 S4：status 断言（水位推进 + calc_status=1 正常）**

```
GET /calc/v1/status?fullKey=M_LOW → HTTP 200
calcState.states=[{"tenantId":1,"fullKey":"M_LOW","lastCalcTime":1795299537615,
  "watermark":1795385937615,"calcStatus":1,"calcStatusText":"正常","errorCount":0,
  "lagMillis":86400069,"lastError":null,"stopped":false}]
→ 水位=已完成窗口 end、last_calc_time=窗口 start、calc_status=1、error_count=0 — 断言 PASS
```

**步骤 S5：连续失败停算子树隔离（坏根 aggType=99 抛异常，阈值=2）**

```
POST observe 注入 M_BAD(aggType=99 抛异常) ← M_BAD_CHILD(srcType=4 依赖 M_BAD)，兄弟 M_OK2
（observe 驱动缝对 bad def 首次计算抛 500，但 setDefinitions 已在计算前完成——定义装载成功）

轮 1 recalc [M_BAD,M_BAD_CHILD,M_OK2] → status=FAILED failed=1
  items(fullKey)=[M_BAD, M_OK2, M_BAD_CHILD]   （未停算，子树执行）
轮 2 recalc → status=FAILED failed=1
  items(fullKey)=[M_BAD, M_OK2]               （坏根已停算，子树跳过）
GET status?fullKey=M_BAD → calcStatus=3 "异常停算" errorCount=2 lastError="不支持的聚合类型: 99" stopped=true
GET status?fullKey=M_BAD_CHILD → calcStatus=3 "异常停算" lastError="上游停算隔离: M_BAD" stopped=true
GET status?fullKey=M_OK2 → calcStatus=1 "正常" stopped=false
→ 连续失败达阈值→坏根停算 + 沿依赖图向下游子树隔离；兄弟分支持续正常 — 断言 PASS（REQ-0023 验收锚点）
```

**步骤 S6：停算节点第三轮跳过，兄弟仍执行**

```
轮 3 recalc [M_BAD,M_BAD_CHILD,M_OK2] → status=COMPLETED
items(fullKey)=[M_OK2]   （停算节点/子树跳过，兄弟仍产出）
```

**步骤 S7：全停归零**

```
SMOKE RESULT: fail=0  ALL PASSED
=== STOP ===
calc JVM PID 60580 已终止（taskkill /F，根目录 bash $! 捕获的是 wrapper PID，实际 JVM PID 由启动日志确认）
=== ports after ===
8095 FREE
=== 宿主服务未动 ===
Redis 6379 未写业务键；backend道 TASK-0053 在途改动未碰；WSL docker 未碰
临时目录 .tmp-t0047/ 测后整体删除
```

### 验收 3：范围核对（原始输出）

```
=== BASE / 交付提交 ===
BASE = ebd443d
HEAD = f19edc1  TASK-0047: 手动补算接口与计算状态页数据源（异步taskId/拓扑序/有界队列+calc_state内存态CalcStateStore缝+MANUAL留痕+子树隔离）
=== 本提交文件数 / 越界 ===
git show --name-only --format="" f19edc1 | grep -c . → 20
git show --name-only --format="" f19edc1 | grep -v '^calc/' → 空（ALL calc/，越界 0）
=== diff 区间 ===
git diff --name-only ebd443d..HEAD → 20 项，全部 calc/**，越界 0
=== 工作区（并行 0053 在途，本任务未碰） ===
 M backend/ot-admin/src/main/resources/application.yml
 M backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/config/OpenApiYmlStructureTest.java
?? backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/...（0053 新增）
?? .agents/  ?? .tmp-t0044v/（开工既存，本任务未触碰）
（上述 backend/** 改动均非本任务：本任务只 add calc/、只提交 calc/**）
=== 未跟踪临时 ===
.tmp-t0047/ 测后整体删除
```

---

## 三、未做项与原因

1. **iot_calc_state 独立 schema 落库（TASK-0063 接线义务）**：本任务交付 **CalcStateStore 接口 + InMemoryCalcStateStore 内存实现 + 契约测试**（照 ingest/calc 假实现范式）；0063 换 JDBC 实现（`iot_calc` schema，`tenant_id`+`full_key` 唯一键，`update_time` ON UPDATE）**零改动替换**——所有调用点（task 层 stateMachine / VO / taskManager）只依赖接口。TASK-0047 不写真库、不引入 JDBC 依赖。补验条件：0063 交付后按 DDL `iot_calc_state` 建表，跑同套 CalcStateStore 契约测试对 JDBC 实现复验 upsert/租户隔离/水位字段映射。
2. **iot_calc_log 物理写库（TASK-0063 接线义务）**：§1.9A 留痕字段（task_id / trigger_type / full_key / period_start/end / is_recalc / input_summary / result_status / started_at/finished_at/duration_ms）当前由 `CalcLogStore`（0046 内存记录式）+ task 层 RecalcContext（MANUAL + isRecalc + taskId 缝）承载；真 MySQL `iot_calc` schema 追加写、按 task_id 拉全链日志归 0063。TDengine REST 故障（TD-0053）期不验真，不宣称物理落库可用。
3. **管理面入口权限 + 审计（TASK-0061 接线义务）**：brief 决策 1/5 + ADR-0010「管理面入口最小化」——本任务只交付 calc 内网 `/calc/v1/**` 三接口 + 只读 VO，**不挂管理面路由、不写前端、无鉴权**。0061 需在管理面对契约 §3.4 `/internal/v1/calc/recalculate|status|task/{taskId}` 做转发入口，落 `iot:calc:recalculate` 权限码 + 操作审计（谁/何时/哪个 fullKey 范围发起）、只读状态页消费 `calcState`/`CalcStatusVo`；本任务 javadoc 已在 `CalcRecalculateController` / `CalcStatusController` / `CalcStatusVo` / `CalcTaskProgressVo` 登记接线义务。
4. **管理面补算操作页 / 状态页前端（0061）**：本任务只交付只读数据源 VO，页面归 0061；无前端改动。
5. **TDengine 真机补验（TD-0053）**：REST 6041 仍故障，本任务不改契约、不写真库；配套 §三末补验条件。
6. **多实例限流/幂等/留痕共享**：CalcTaskManager 有界队列 + 进程内 CalcStateStore 为单实例口径；多实例 Redis 化为演进位（同 0041/0042 既有口径）。
7. **契约 §3.4 路径前缀差异（/calc/v1 vs /internal/v1/calc）**：brief 决策 1 明确「内部三接口（calc 内网，非管理面）」，本任务按 `/calc/v1/**`（既有内网观测面同族，无鉴权）实现；契约 §3.4 的 `/internal/v1/calc/**` 路径及系统间鉴权归 0061 管理面转发层处理。非契约冲突，登记不阻断；若评审要求 calc 侧直接挂 `/internal/v1/calc/**`，为另开任务的路径迁移（不改本任务交付语义）。
8. **TASK 状态**：按硬约束未把 TASK-0047 置「已完成」，保持「进行中」待评审；未改 workitems 库（唯一例外：本报告）。

**TDengine / 0063 环境恢复后真机补验条件（用户侧动作）**：

1. TDengine 6041 恢复后：建 scratch 库 + `iot_calc_state`/`iot_calc_log` 表（独立 `iot_calc` schema，逐语句），以 0063 JDBC 实现跑 CalcStateStore 契约测试，断言 upsert/租户隔离/watermark/error_count/calc_status 字段映射、iot_calc_log 追加写按 task_id 拉全链（MANUAL 留痕逐项可读）；DROP scratch 清理。
2. 0061 管理面接线后：按本任务 javadoc 接线义务补 `iot:calc:recalculate` 权限 + 操作审计 + 状态页 VO 消费，联调 `/internal/v1/calc/**` 转发。

---

## 四、偏离计划之处

1. **status 数据源落既有 CalcStatusController 的 calcState 段，而非独立 handler**（实现取向 + RED 驱动）：brief/§3.4 写 `GET /calc/v1/status?fullKey=`，初版新控制器直挂 `/calc/v1/status` 触发 Spring `Ambiguous mapping`（与既有 0040 观测口撞路径）。改为并入既有 handler 的 `calcState` 段（fullKey 可选）——为什么现在对了：单一端点承载全部观测面，既有 status 响应结构零回归，brief「status 查询数据源」语义不变。详见 §验收 1 TDD-3。
2. **新 HTTP 测试用 `calc.http-addr=:18095`**（测试夹具避让，非生产改动）：`ServerPortCustomizer` 按默认 `:8095` 覆盖 RANDOM_PORT，前轮 TIME_WAIT 阻断重绑（RED 真红）。测试显式专用端口——为什么现在对了：消除测试间端口时序争用，生产默认 8095 不变。详见 §验收 1 TDD-4。
3. **live S5 用 aggType=99 注入失败而非缺失 def**（失败注入取向）：`recomputeWindow` 对不存在的 fullKey 返回 false（业务失败），对存在但 aggType 非法则抛异常（真实 RuntimeException），两种都驱动 onFailure；选后者使「不支持的聚合类型: 99」成为可见 last_error，便于 status 断言。observe 首次计算该 bad def 抛 HTTP 500 属驱动缝预期行为（定义已在其前装载），非实现缺陷。
4. **异步队列有界拒绝采用「返回 null + 计数」而非抛异常**（限流取向）：brief「补算风暴不拖垮在线计算」——submit 超队列返回 null（controller 回 `ok=false` 限流提示），不阻断调用线程；契约测试锁定 rejectedCount 可观测。与在线 DualTriggerExecutor 分线程池（2 workers vs 在线共用池），分区隔离成立。
5. **TDD 红转绿 5 处真实修复**：见 §验收 1「TDD 红转绿」（编译期、2 项真实断言、Spring 撞映射、端口争用、live 拓扑偏序），每项红→修→绿与「为什么现在对了」已逐条记录。

---

## 五、遗留问题

1. **iot_calc_state / iot_calc_log 落库（0063）**：接口 + 内存实现 + 契约就绪，JDBC 实现与真机 DDL 复验归 0063（§三-1/2）。
2. **管理面权限/审计/页面（0061）**：内网三接口 + 只读 VO 就绪，管理面入口最小化接线归 0061（§三-3/4）。
3. **TDengine 真机补验（TD-0053）**：环境恢复后按 §三末条件执行，本批未验真。
4. **多实例一致性**：CalcTaskManager 有界队列与 InMemoryCalcStateStore 为单实例口径，Redis 化为演进位。
5. **异步任务重启不持久**：taskId/进度当前仅内存（CalcTaskManager），进程重启后 GET task 返回 404；§1.9A iot_calc_log 落库（0063）后可按 task_id 重建进度视图，属演进位。
6. **精确 tenant_id 注入**：单租户形态下 task 层 tenant_id 暂固定 1L（CalcTaskManager.run），0063 落库时由装配注入真实 id（javadoc 登记）。
7. **区间范围说明**：`ebd443d..HEAD` 仅 20 项 calc/**；并行 TASK-0053 的 backend/** 改动未提交、不在本任务交付提交内（评审按 `f19edc1` 判断归属）。
8. **TASK 状态**：保持「进行中」待评审，未置「已完成」；未回写 workitems 库（本报告除外）。

---

## 一句话结论

**通过**——验收 1（`cd calc && mvn test` 265 测试 0F0E3Skip BUILD SUCCESS，244 基线逐类零回归 + 21 新增，TDD 红转绿 5 处含编译期/真实断言/Spring 撞映射/端口争用/live 拓扑偏序修复留痕）、验收 2（live 冒烟全序列 `SMOKE RESULT: fail=0`：8095 起→recalculate 异步 taskId 立返 64ms/27ms→进度逐项 MANUAL 结果 + 拓扑偏序 M_LOW<M_HIGH<M_TOP→status 水位推进/calc_status=1→连续失败阈值=2 坏根 calc_status=3 + 下游子树隔离 lastError=上游停算隔离 + 兄弟 calc_status=1→第三轮停算跳过兄弟仍产出→8095 全停 FREE）、验收 3（提交 `f19edc1` 20 文件全落 `calc/**`、越界 0，`git diff ebd443d..HEAD` 同 20 项）三条全部以原始输出为证通过；0063 落库/0061 管理面接线/TDengine 真机补验按 brief 登记为接线义务与演进位，不阻断三条验收判定，TASK 保持「进行中」待评审。

---
---

# 修订轮（2026-11-24）：diff 评审退回 3 Important + 2 Minor

- 触发：`TASK-0047-review.md` 判定不通过——3 Important（F-01 滞后状态 2 生产链路空接线 / F-02「CalcStateStore 零改动替换」不成立 / F-03 生产代码写死 `"t47-live"` 且 tenant_id 固定 1L）+ 2 Minor（F-04 拒绝计数翻倍 / F-05 topoOrder 环回退 fail-open）。
- 修订范围 = calc/**；修订 BASE = `f19edc1`（被评审提交）；**修订提交 = `ea34d02`**（`TASK-0047 修订轮: F-03 租户贯穿去t47-live/F-01 滞后状态2按env阈值/F-02 CalcStateStore注入可替换(0063零改动)`；10 文件全 `calc/**`，越界 0）。
- 修订轮结论先行：**3 Important 全部修复**，2 Minor（F-04/F-05）按协调者裁决 deferred 登记不修；TDD 红转绿（F-01 状态 2 可由 env 阈值触发 / F-02 注入可替换 / F-03 租户隔离各配测试，对旧实现必红——编译期 + 真实断言）；全量 **275 测试 0F0E3Skip BUILD SUCCESS**（265 基线逐类零回归 + 10 新增）；live 冒烟复验全序列通过（租户隔离跨查 404、状态 2 由 env 200ms 触发、生产代码零 `t47-live` 实证）。

## 修订轮一、改动文件清单（提交 `ea34d02`，10 项，全部 `calc/**`）

| 文件 | 修订内容 |
| --- | --- |
| `CalcEngineApplication.java` | ①新增 `calcStateStore()` @Bean（默认 `InMemoryCalcStateStore`，0063 换 JDBC 只需替换该 Bean，消费侧零代码改动）；②`calcTaskManager()` 注入 store + env 滞后阈值真正传入状态机；③**删除 `recomputeWindow("t47-live",...)` 字面量**，改 `ctx.tenant()`（租户来自请求上下文）；④启动日志改 `lagThresholdMs()` 实际值。 |
| `task/CalcTaskManager.java` | ①`CalcStateStore` 依赖注入构造（6 参全注入 + 4 个兼容重载委托默认内存实现）；②`RecalcContext` 增 `tenant` 字段；③`tenantIdOf(tenant)` 映射（首次分配稳定 id，租户维度状态键隔离）；④`run()` 用 `tenantIdOf(tenant)` 取代写死 1L；⑤`getTask(tenant, taskId)` 租户归属校验（跨租户返回 null）；⑥`evaluateLag(tenantId, fullKey, now)` 生产缝（状态查询前按 env 阈值调用）；⑦submit tenant 归一化（空→default）。 |
| `state/CalcStatusVo.java` | 新增 `lagging` 字段，`from(s, lagThresholdMs, now)` 真正消费阈值参数（lag > 阈值 → lagging=true），`toMap` 暴露供 0061 渲染（旧实现死参数必红）。 |
| `web/CalcStatusController.java` | `status()` 增 `tenant` 参数；按 tenant 取 tenantId、读状态前调 `evaluateLag`（F-01 生产调用）；VO 用 env 阈值；recentTasks 按 tenant 过滤（F-03）。`CalcProperties` 注入取 `lagThresholdMs`。 |
| `web/CalcRecalculateController.java` | `task/{taskId}` 增 `tenant` 查询参数，`getTask(tenant,...)` 跨租户 → 404（F-03 HTTP 层）。 |
| `test/state/CalcStateStoreReplaceabilityContractTest.java`（新） | 4 项（`@TestFactory` 动态测试）：同一套 store 契约跑在内存实现 + JDBC 替身两实现上，证明「0063 换实现零代码改动替换」——对旧硬编码实现必红。 |
| `test/task/CalcTenantIsolationContractTest.java`（新） | 3 项：tenant 贯穿计算上下文、跨租户查 task 拒绝、同 fullKey 两租户状态隔离——对旧实现必红。 |
| `test/state/CalcStatusViewContractTest.java` | +1 项：`lagging` 标志按阈值翻转（死参数红→绿）。 |
| `test/task/CalcTaskManagerContractTest.java` | +2 项：注入 store 被实际使用、`evaluateLag` 按 env 阈值产生 calc_status=2。 |
| `test/web/CalcRecalculateControllerTest.java` | HTTP task 查询带 tenant + 跨租户 404 断言 + status 标注 tenant。 |

## 修订轮二、TDD 红→绿证据（每项附「为什么现在对了」）

**RED（编译期，修订 BASE `f19edc1`）**：先写 F-01/F-02/F-03 三组测试 → `mvn -q -Dtest=... test` `[ERROR] COMPILATION ERROR`：`lagging()` 找不到、`CalcTaskManager(RecalcItemFn, CalcStateStore, int, long, int, int)` 构造器不存在、`tenantIdOf(String)` 不存在、`getTask(String,String)` 不匹配、`RecalcContext.tenant()` 不存在、`CalcTaskManager.evaluateLag(long,String,long)` 不存在（实测 14+ 处「找不到符号」）。GREEN：实现上述 API 后消除编译错误。

**F-03 GREEN —— tenant 贯穿去字面量（真实断言）**：

```
$ grep -rn "t47-live" calc/src/  →  NONE（生产代码零冒烟值）
live: GET /calc/v1/task/MANUAL-tenantA-000001?tenant=tenantB → HTTP 404
      {"error":"task 不存在或不属于该 tenant: MANUAL-tenantA-000001","ok":false}
live: tenantA status tenantId=1 / tenantB status tenantId=2（状态存储键隔离）
```

- 为什么现在对了：`submit` 把请求 tenant 贯穿到 taskId、CalcTask.tenant、RecalcContext.tenant 与 tenantIdOf 映射；生产 lambda 用 `ctx.tenant()` 而非写死字面量，任何真实租户的补算都落自身命名空间；`getTask(tenant,...)` 校验归属，跨租户读 fail-closed（404），REQ-0023 分区隔离在生产路径成立。原 `"t47-live"` 泄漏已彻底删除并在 §三「未做项」补登记。

**F-01 GREEN —— 滞后状态 2 由 env 阈值触发（真实断言）**：

```
构造 mgr（lagThresholdMs=200L）→ submit M.A → evaluateLag(tenantId,"M.A",700L)
  watermark=200, lag=500ms > 200ms → calcStatus=2 （旧实现零调用 evaluateLag / 4参构造写死5min，必红）

live（CALC_CALC_LAG_THRESHOLD_MS=200）：启动日志「阈值=2、滞后阈值=200ms」
GET /calc/v1/status?fullKey=M_LOW&tenant=tenantA （空闲 1s 后）→
  {"calcStatus":2,"calcStatusText":"滞后","lagMillis":86401447,"lagging":true,"stopped":false}
```

- 为什么现在对了：`calcLagThresholdMs` env 经 `calcTaskManager()` 传入 6 参构造 → CalcStateMachine 持有真阈值；status 查询前 `evaluateLag` 按该阈值把超期节点置 2；`CalcStatusVo.from` 的阈值参数用于 `lagging` 标志并与 calc_status=2 一致——状态 2 在生产链路真实可达、env 开关真正生效。

**F-02 GREEN —— CalcStateStore 注入可替换（真实断言）**：

```
CalcStateStoreReplaceabilityContractTest（@TestFactory 动态测试）：
  InMemoryCalcStateStore(默认)      : upsert/get 与租户隔离 PASS / 同 uk 覆盖不追加 PASS
  JdbcLikeStateStore(0063替身)      : upsert/get 与租户隔离 PASS / 同 uk 覆盖不追加 PASS
注入测试：new CalcTaskManager(fn, injectedStore, 5, 200L, 2, 8) → 计算写入落在注入 store 上
```

- 为什么现在对了：`CalcTaskManager` 构造接收 `CalcStateStore` 接口（不再 `new InMemoryCalcStateStore()` 硬编码），装配用 `calcStateStore()` @Bean 提供默认实现；0063 只需用 profile/配置把该 Bean 换成 JDBC 实现，管理器、状态机、VO、控制器消费侧零代码改动；契约测试对两种实现同源跑通（若测试直接 new 内存实现则替身必红）——brief 承诺的「零改动替换」成立。

**F-04/F-05（Minor，deferred 不修，报告登记）**：拒绝计数翻倍（handler + catch 各 +1，测试仅断言 >=1）与 topoOrder 环回退 fail-open（注释称 fail-closed 实为返回提交序）——按修订范围「Minor deferred 不修」登记，不阻断 Important 修复；0063/后续道接线时可一并处理。

## 修订轮三、全量验收命令原始输出

```
$ cd calc && mvn test
...（逐类 Results 段，41 个 test class）...
[INFO] Running net.otlab.calc.state.CalcStateStoreReplaceabilityContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.018 s -- in net.otlab.calc.state.CalcStateStoreReplaceabilityContractTest
[INFO] Running net.otlab.calc.state.CalcStatusViewContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.011 s -- in net.otlab.calc.state.CalcStatusViewContractTest
[INFO] Running net.otlab.calc.task.CalcTaskManagerContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.285 s -- in net.otlab.calc.task.CalcTaskManagerContractTest
[INFO] Running net.otlab.calc.task.CalcTenantIsolationContractTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.046 s -- in net.otlab.calc.task.CalcTenantIsolationContractTest
[INFO] Running net.otlab.calc.web.CalcRecalculateControllerTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.680 s -- in net.otlab.calc.web.CalcRecalculateControllerTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 275, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  7.563 s
EXIT=0
```

**265 基线逐类零回归**：所有既有类逐类计数与 round1 报告终验逐字一致（ack 8 / app 1 / config 11 / event 9 / executor 4+5+10+4+2+1skip+2 / graph 7+17+23 / metric cascade 2+late 6+agg 8+companion 6+sched 4+window 7+writer 5 / operator 6+5+4 / runtime 11 / snapshot 13+9+20 / status 8+2skip+8+5 / storage 5+7）；本轮新增 10 = 替换性契约 4 + 租户隔离 3 + 状态 VO +1 + 任务管理 +2（275 = 265 + 10）。3 Skipped 仍为 0041 env 门控 1 + 0049 env 门控 2。

**范围核对**：

```
BASE（修订）= f19edc1 ; 修订提交 = ea34d02
git show --name-only --format="" ea34d02 | wc -l → 10
git show --name-only --format="" ea34d02 | grep -v '^calc/' → 空（ALL calc/，越界 0）
git diff --name-only f19edc1..ea34d02 → 10 项全 calc/**
grep -rn "t47-live" calc/src/ → NONE（F-03 冒烟值清除实证）
8095 起前 FREE / 用毕 FREE（PID 93336 终止）；并行 0053 的 backend/** 改动未碰
```

## 修订轮四、live 冒烟复验（F-01/F-02/F-03 实证）

```
===== R-S1: 注入定义（tenantA 命名空间） =====  HTTP 200
===== R-S2: recalculate 异步提交 taskId 立返 =====
HTTP 200 {"ok":true,"taskId":"MANUAL-tenantA-000001","accepted":3} elapsed_ms=11
task status=COMPLETED items=['M_LOW','M_HIGH','M_TOP']
===== R-S3: F-03 租户隔离 —— 跨租户查 task 拒绝 =====
跨租户查询 GET /calc/v1/task/MANUAL-tenantA-000001?tenant=tenantB → HTTP 404
  {"error":"task 不存在或不属于该 tenant: MANUAL-tenantA-000001","ok":false}
本租户查询 ?tenant=tenantA → HTTP 200 OK
===== R-S4: F-03 租户 B 独立提交，状态按 tenant 隔离 =====
tenantB taskId=MANUAL-tenantB-000002 status=COMPLETED
tenantA status tenant=tenantA tenantId=1 / tenantB status tenant=tenantB tenantId=2
===== R-S5: F-01 状态2 滞后由 env 阈值(200ms)触发 =====
M_LOW state after 1s idle:
{"tenantId":1,"fullKey":"M_LOW","calcStatus":2,"calcStatusText":"滞后",
 "lagMillis":86401447,"lagging":true,"errorCount":0,"stopped":false}
===== R-S6: F-02 注入缝实证（bean 替换无需改消费代码） =====
status tenantA: states=3 recentTasks=1（仅 tenantA 任务）
===== R-S7: 全停 =====
REVISION SMOKE RESULT: fail=0  ALL PASSED
=== ports after === 8095 FREE
```

## 修订轮一句话结论

**通过（修订轮）**——3 Important 全部修复：F-03 租户贯穿（去 `"t47-live"` 字面量，`grep -rn "t47-live" calc/src/ → NONE`；跨租户查 task 404；tenantA/tenantB 状态 tenantId 1/2）、F-01 滞后状态 2 由 env 阈值真实触发（`CALC_CALC_LAG_THRESHOLD_MS=200` 生产 status 返回 `calc_status=2/滞后/lagging=true`，此前零调用 evaluateLag 已接入）、F-02 CalcStateStore 注入可替换（@Bean + 6 参构造注入，`@TestFactory` 同套契约跑通内存/0063 替身两实现，0063 换 JDBC 零代码改动）；F-04/F-05 Minor deferred 登记；全量 275 测试 0F0E3Skip BUILD SUCCESS、265 基线零回归、提交 `ea34d02` 10 文件全 calc/** 越界 0、live 冒烟 fail=0；TASK 保持「进行中」，待协调者 scoped 复评（只看 3 Important/2 Minor 的修复 diff 与证据）。
