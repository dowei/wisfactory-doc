# TASK-0045 验证报告：指标聚合调度与物化存储 st_metric_value

- 执行者：TASK-0045 执行者（no-subagents；计算引擎道——只改 `calc/**`）
- **BASE（开工 HEAD）= `01236fd`**（2026-11-24 开工时主仓 main，brief「BASE=当前 HEAD」口径）；**diff 区间末 = `4dd8144`**（本任务提交）；区间内并存并行前端道 TASK-0059 提交 `86a1c09`（改 `frontend/**`，本任务全程未碰/未 add/未提交）
- 交付提交：**`4dd8144`**（`TASK-0045: 指标聚合调度与物化存储 st_metric_value（自实现调度/窗口/聚合/完整度/四元组tag）`；**21 文件全部 `calc/**`，本提交越界 0**）
- 执行日期：2026-11-24
- 环境事实：Redis 宿主 phpStudy 3.0.504@6379（PING +PONG，本任务未写业务键）；TDengine 192.168.31.145:6041 REST 仍故障（`curl -m 6 ... /rest/sql --data-raw "SELECT 1"` → HTTP=000、time=6.006s，服务器侧，TD-0053 登记），真机写入+读回列为环境恢复后交付条件，**不宣称已验真**；8095 归 calc（起前探测 FREE / 用毕即停 / 终态 FREE）；假后端 18045 用毕即停；wsl docker 未碰；并行前端道 0059 在途（不碰 frontend）。
- 结论先行：**验收 3 条全部通过**——mvn 220 测试 0F0E3Skip BUILD SUCCESS（195 基线逐类零回归 + 25 新增）；live 冒烟全序列 `RESULT fail=0`（比值/日均/级联/完整度/无数据/月末窗口/四元组 tag/同 ts 覆盖，逐条原始输出）；范围核对本提交 21 文件全落 `calc/**`、越界 0。**调度取向（自实现非 XXL-Job）**与**契约字面差异（st_metric_value 四元组 tag、完整度不入物理列）**按 brief「登记字面差异」口径记录，见 §三/§四。

---

## 一、改动文件清单（提交 `4dd8144`，21 项，全部 `calc/**`）

### 1.1 新增——指标聚合包 `calc/src/main/java/net/otlab/calc/metric/`（11）

| 文件 | 职责 |
| --- | --- |
| `MetricWindow.java` | 自然日历窗口（HOUR/DAY/MONTH/YEAR 对齐真实日历边界，非固定秒数）；`previous()` 对 MONTH 恒返回上一完整自然月（月末一次、无「本月至今」半成品，ADR-0010）；`due(now,graceDays)` 宽限到期判据；窗口唯一键=(tenant,metricCode,periodStart,calendarType)。 |
| `MetricValueStore.java` | 已物化读数源（0042 ResultSink 数据 + Redis，注入缝）：value/quality/ts；GOOD(0) 计入到达率，非 GOOD/停用剔除；前缀展开读级联子结果；null/空白前缀 fail-closed（live 抓到 NPE 后修）。 |
| `MetricDef.java` | 指标定义（映射快照 iot_metric）：aggType 1 平均/2 累计/3 最大/4 最小/5 比值；sourceKeys / ratioNumKey+ratioDenKey；srcType 级联层级标记。 |
| `MetricResult.java` | 聚合结果：value(null=无值不写 0)、completeness(0..1)、incomplete(阈值标记)、noData、ts(窗口起始)。完整度/标记不进冻结表列（字面差异见 §三）。 |
| `MetricAggregator.java` | 聚合口径：级联 src_type（下级物化结果逐级聚合）；比值分子分母分别聚合再相除（非逐条比值均值），分母 0 无值打标；DAY+avgType=1 固定「总量÷24」（23 小时到达→23.0≠24.0）；完整度=GOOD 到达/应到、非 GOOD+停用剔除、低于阈值打标；无数据出无值+完整度 0。 |
| `MetricValue.java` | 物化写载荷：ts/value/yoy/mom + 四元组 tag（tenant_id/metric_code/instance_code/period）；`withValue()` 同 ts 覆盖。 |
| `MetricValueWriter.java` | st_metric_value 真写口：逐语句 REST（0029 多语句静默空操作纪律）、库名全限定、USING TAGS 首写、taosAdapter code 解析（0049 F-01）；**四元组 tag 逐字按冻结契约，无 full:/full_key**（0049 F-02）；同 ts 覆盖写幂等；失败上抛由 Sink WARN 隔离。 |
| `MetricValueRecorder.java` | 记录式写口（照 0042 StorageRuntime.RecordingWriter）：TDengine REST 故障期捕获 INSERT SQL + 载荷供 /calc/v1/status 观测与 live 契约断言。 |
| `MetricSink.java` | 物化 Sink：逐条写失败 WARN 隔离不阻塞后续；无数据 value=null 仍写 NULL 行（不写 0 冒充）。 |
| `MetricScheduler.java` | 自实现单线程调度（照 0041 PeriodicScheduler 范式，daemon + fail-soft）：按窗口到期（end+graceDays）回溯找最近到期窗提交；同窗 windowKey 幂等去重；启动/停止/观测。 |
| `MetricRuntime.java` | 装配 聚合器+读数源+Sink+调度；定义装载、到期触发计算+物化写+观测登记；`/calc/v1/status.metric` 视图（windows/completeness/产出数/告警数/lastWriteSql）。 |

### 1.2 新增——观测驱动口 `calc/src/main/java/net/otlab/calc/web/MetricObserveController.java`（1）

`GET /calc/v1/metric`（观测视图）+ `POST /calc/v1/metric/observe`（live 冒烟/演进位驱动缝：注入指标定义+物化值+时刻触发一次到期窗计算）。非契约命名空间（沿用 /calc/v1/** 内网观测面）；不作为生产数据源入口，真实触发仍由调度线程发起。

### 1.3 修改（5）

`CalcEngineApplication.java`（metricRuntime Bean + runner 启动自实现调度，默认关）、`config/CalcProperties.java`（metric-aggregation-enabled / grace-days / completeness-min 三项 env，默认关/3/0.8）、`web/CalcStatusController.java`（status.metric 观测面，null-safe）、`resources/application.yml`（env 映射）+ 测试 `config/CalcPropertiesTest.java`（+1 默认值零回归断言）。

### 1.4 新增——契约测试（4 类 24 项 + 配置 1 = 25）

| 文件 | 用例数 | 覆盖 |
| --- | --- | --- |
| `metric/MetricWindowContractTest.java` | 7 | HOUR/DAY/MONTH/YEAR 自然日历边界；月末一次（`previous` 月中→上一完整月）；宽限期到期判据；窗口唯一键含 tenant/metric/periodStart/calendarType。 |
| `metric/MetricAggregatorContractTest.java` | 8 | 级联 SUM；比值分子分母分别聚合再相除（30/7≠4.5）；分母 0 无值；日均=总量÷24（23.0≠24.0）；完整度到达率含非 GOOD/停用剔除（0.7）+低阈值打标；无数据不写 0；级联 src_type；比值缺键 fail-closed（live 抓 NPE 后补）。 |
| `metric/TdengineMetricValueWriterContractTest.java` | 5 | 四元组 tag 逐值精确+列顺序（冻结契约逐字）；无 full:/无 quality/completeness 越界列；同 ts 覆盖幂等；逐语句两次 REST；失败 WARN 不阻塞；taosAdapter code 解析 fail-closed。 |
| `metric/MetricSchedulerContractTest.java` | 4 | 宽限期回溯最近到期窗+同窗幂等；宽限 0 即到期；月窗月末一次月内不重复；status 视图含窗口/完整度/产出数。 |
| `config/CalcPropertiesTest.java`（修改） | +1 | TASK-0045 三项 env 默认值零回归断言。 |

新增用例合计 25，195 基线 → 220。

---

## 二、验收命令原始输出

### 验收 1：`cd calc && mvn test` → BUILD SUCCESS、0 失败 0 错误

终验（提交 `4dd8144` 对应源码）Results 段逐类原始输出（CR 行尾已去）：

```
[INFO] Running net.otlab.calc.ack.HttpAckSenderTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.900 s -- in net.otlab.calc.ack.HttpAckSenderTest
[INFO] Running net.otlab.calc.CalcEngineApplicationTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.437 s -- in net.otlab.calc.CalcEngineApplicationTest
[INFO] Running net.otlab.calc.config.CalcPropertiesTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.037 s -- in net.otlab.calc.config.CalcPropertiesTest
[INFO] Running net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.007 s -- in net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Running net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Running net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.041 s -- in net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Running net.otlab.calc.executor.OperatorIntegrationContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.014 s -- in net.otlab.calc.executor.OperatorIntegrationContractTest
[INFO] Running net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.006 s -- in net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Running net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[WARNING] Tests run: 1, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0.001 s -- in net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[INFO] Running net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.010 s -- in net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Running net.otlab.calc.graph.CycleDetectorTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.006 s -- in net.otlab.calc.graph.CycleDetectorTest
[INFO] Running net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.028 s -- in net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Running net.otlab.calc.graph.GraphBuilderTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.045 s -- in net.otlab.calc.graph.GraphBuilderTest
[INFO] Running net.otlab.calc.metric.MetricAggregatorContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.metric.MetricAggregatorContractTest
[INFO] Running net.otlab.calc.metric.MetricSchedulerContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.015 s -- in net.otlab.calc.metric.MetricSchedulerContractTest
[INFO] Running net.otlab.calc.metric.MetricWindowContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.007 s -- in net.otlab.calc.metric.MetricWindowContractTest
[INFO] Running net.otlab.calc.metric.TdengineMetricValueWriterContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.011 s -- in net.otlab.calc.metric.TdengineMetricValueWriterContractTest
[INFO] Running net.otlab.calc.operator.DeltaOperatorContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.004 s -- in net.otlab.calc.operator.DeltaOperatorContractTest
[INFO] Running net.otlab.calc.operator.EnergyDerivationContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.011 s -- in net.otlab.calc.operator.EnergyDerivationContractTest
[INFO] Running net.otlab.calc.operator.IntegralOperatorContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.007 s -- in net.otlab.calc.operator.IntegralOperatorContractTest
[INFO] Running net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.052 s -- in net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Running net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.214 s -- in net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Running net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.007 s -- in net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Running net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.043 s -- in net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Running net.otlab.calc.status.StatusEvaluatorContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.018 s -- in net.otlab.calc.status.StatusEvaluatorContractTest
[INFO] Running net.otlab.calc.status.StatusRedisLiveTest
[WARNING] Tests run: 2, Failures: 0, Errors: 0, Skipped: 2, Time elapsed: 0.001 s -- in net.otlab.calc.status.StatusRedisLiveTest
[INFO] Running net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.020 s -- in net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Running net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.011 s -- in net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] Running net.otlab.calc.storage.HomogeneousResultSinkContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.008 s -- in net.otlab.calc.storage.HomogeneousResultSinkContractTest
[INFO] Running net.otlab.calc.storage.TdengineItemValueWriterContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.024 s -- in net.otlab.calc.storage.TdengineItemValueWriterContractTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 220, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  6.329 s
EXIT=0
```

**195 基线逐类零回归**：195 = 8+1+8+4+5+10+4+2+1(skip)+2+7+17+23+6+5+4+11+13+9+20+8+2(skip)+8+5+5+7；本轮每类数值与 0042 修订轮报告 Results 段逐字一致。3 Skipped = 0041 env 门控 1 + 0049 env 门控 2（未改这些写口）。新增 25 项（窗口 7 + 聚合 8 + 写口 5 + 调度 4 + 配置 1）。

**TDD 红转绿留痕（逐修复附「为什么现在对了」）**：

1. **窗口/聚合/写口/调度契约测试**：先写测试 → 编译期红（`找不到符号 MetricWindow/MetricAggregator/...`，实测 `[ERROR] COMPILATION ERROR` 30+ 行）；实现后 → 真实断言红 3 项：① completeness expected 0.7 but was 0.9（测试用 3 参 `put` 把质量码当 ts 写，4 参修正）② avg expected 23.0 but was 3.0（sourceKeys 给精确 3 键而非前缀，改前缀 `I.H` + 聚合器展开物化子结果）③ scheduler expected 1 but was 0（宽限 3 天>日窗长 1 天时紧邻上一窗永不到期，调度器改「向前回溯找最近到期窗」，测试同步改为尾追+幂等）。修后 32/32 绿。为什么现在对了：把日历边界/月末/比值口径/日均口径/到达率/四元组 tag 全部做成可执行断言，前缀 bug、假均值、宽限死区在数学上无法逃逸。
2. **live 抓到比值缺键 NPE（真红）**：`POST /calc/v1/metric/observe` 用 aggType=5 但 ratioNumKey/ratioDenKey 为 null → HTTP 500，日志 `NullPointerException: Cannot invoke "String.length()" because "prefix" is null`（`MetricValueStore.goodValuesWithPrefix`）。先补回归测试 `ratioMissingNumDenKeysFailsClosedNoDataNotThrow` → `Errors: 1` 真红；修 `goodValuesWithPrefix` 对 null/blank 前缀返回空集（fail-closed 无数据，不抛）→ 绿。为什么现在对了：配置缺失是「无数据」而非「异常」——fail-closed 产出无值+完整度 0 行，不静默崩溃、不阻塞调度线程；null 前缀不再作为 `startsWith` 入参。

### 验收 2：live 冒烟全序列（每步原始输出）

假后端（Python HTTPServer，18045，真快照信封 SHA-256 可重算，快照空图 version=45，hash=`c3d5fa772602e429bb8a6485a09dd086d051ddc2b5fe90b4b71117d70b77ad19`）+ calc 起（8095，`CALC_METRIC_AGGREGATION_ENABLED=true CALC_TENANT=t45 CALC_TENANT_ID=1 CALC_METRIC_GRACE_DAYS=3 CALC_METRIC_COMPLETENESS_MIN=0.8 CALC_SCHEDULER_TICK_SECONDS=60`）。

**步骤 0：起前端口 + calc 启动（自实现调度确认）**

```
8095/18045 FREE（起前探测）；Redis PING +PONG
TDengine REST probe: curl -m 6 -u root:taosdata http://192.168.31.145:6041/rest/sql --data-raw "SELECT 1"
  → HTTP=000 time=6.006102（服务器侧故障 TD-0053，真机写入列环境恢复后条件）

$ java -jar calc-engine.jar
INFO ... TomcatWebServer : Tomcat started on port 8095 (http)
INFO ... CalcEngineApplication : Started CalcEngineApplication in 1.613 seconds
INFO ... MetricScheduler : [CALC-METRIC-SCHED] 指标聚合调度启动 tick=60s graceDays=3（自实现，非 XXL-Job）
=== port 8095 === TCP 0.0.0.0:8095 LISTENING 80076
=== status.metric 起前 ===
{"windows":{},"produced":0,"lastCompleteness":null,"lastIncompleteFlags":{},"schedulerRunning":true,"graceDays":3,"completenessMin":0.8,"warnings":0,"writes":0,"lastWriteSql":null}
```

**步骤 1–8：完整 live 冒烟序列（最终干净一轮，逐条原始输出）**

```
=== L1 比值型：分子分母分别聚合再相除 ===
PASS 比值 30/7=4.2857 | INSERT INTO iot_t45.D_M_RATIO_I01 USING iot_t45.st_metric_value TAGS (1, 'M_RATIO', 'I01', 'DAY') VALUES (1795392000000, 4.285714285714286, null, null)
PASS 非逐条比值均值(4.5)
PASS 完整度=1.0

=== L2 平均口径：日平均=总量÷24（23 小时到达）===
PASS 均值=23*24/24=23.0 | INSERT INTO iot_t45.D_M_AVG_I01 USING iot_t45.st_metric_value TAGS (1, 'M_AVG', 'I01', 'DAY') VALUES (1795392000000, 23.0, null, null)
PASS 非到达均值 24.0
PASS 完整度=23/24 | 0.9583333333333334

=== L3 级联 src_type + 60% 完整度标记 ===
PASS 级联 SUM 1..6=21 | INSERT INTO iot_t45.D_M_CASCADE_I01 USING iot_t45.st_metric_value TAGS (1, 'M_CASCADE', 'I01', 'DAY') VALUES (1795392000000, 21.0, null, null)
PASS 完整度=0.6
PASS 60%<80% 带不完整标记

=== L4 分母 0 / 无数据：无值完整度 0，不写 0 ===
PASS value=NULL 非 0 | INSERT INTO iot_t45.D_M_EMPTY_I01 USING iot_t45.st_metric_value TAGS (1, 'M_EMPTY', 'I01', 'DAY') VALUES (1795392000000, null, null, null)
PASS 完整度=0
PASS 带无数据/不完整标记

=== L5 月末窗口口径：MONTH 窗起=月初，无中间快照 ===
PASS MONTH ts=2026-11-01 (=1790812800000) | INSERT INTO iot_t45.D_M_MON_I01 USING iot_t45.st_metric_value TAGS (1, 'M_MON', 'I01', 'MONTH') VALUES (1790812800000, 300.0, null, null)
PASS period tag=MONTH

=== L6 st_metric_value 四元组 tag 契约逐值断言（brief「六元组」为 st_item_value 词汇，见 §三）===
PASS SQL 形态/四元组标签
PASS tenant_id=1 / metric_code=M_MON / instance_code=I01 / period=MONTH
PASS 无 full: 前缀
PASS 逐语句无分号

=== L7 同 ts 覆盖写幂等：重算 M_RATIO 两次，窗口键/ts 同，value 变化 ===
PASS 同子表+同 ts
  s1: INSERT INTO iot_t45.D_M_RATIO_I01 USING iot_t45.st_metric_value TAGS (1, 'M_RATIO', 'I01', 'DAY') VALUES (1795392000000, 4.285714285714286, null, null)
  s2: INSERT INTO iot_t45.D_M_RATIO_I01 USING iot_t45.st_metric_value TAGS (1, 'M_RATIO', 'I01', 'DAY') VALUES (1795392000000, 8.0, null, null)
PASS value 可被覆盖(两次值不同)

=== L8 最终 status.metric ===
{"windows":{"M_RATIO":{...,"completeness":1.0,"incomplete":false,"noData":false},
           "M_AVG":{...,"completeness":0.9583333333333334,"incomplete":false},
           "M_MON":{"period":"MONTH","completeness":1.0,"ts":1790812800000,"incomplete":false},
           "M_CASCADE":{...,"completeness":0.6,"incomplete":true},
           "M_EMPTY":{...,"completeness":0.0,"incomplete":true,"noData":true}},
 "produced":7,"writes":7,"warnings":0,"schedulerRunning":true,"graceDays":3,"completenessMin":0.8,
 "lastWriteSql":"INSERT INTO iot_t45.D_M_RATIO_I01 USING iot_t45.st_metric_value TAGS (1, 'M_RATIO', 'I01', 'DAY') VALUES (1795392000000, 8.0, null, null)"}
RESULT fail=0
```

**步骤 9：全停归零**

```
=== STOP ===
calc stopped（PID 80076）
fake backend stopped（PID 18045 对应进程）
=== ports after ===
8095/18045 FREE
=== 宿主服务未动 ===
TCP 0.0.0.0:3307 LISTENING 65616
TCP 127.0.0.1:6379 LISTENING 5508
```

临时冒烟目录 `D:\workspace\WisFactory\.tmp-t0045\` 测后已整体删除；未写真 TDengine（SQL 由记录式写口捕获，真机写入列环境恢复后条件）；未碰 backend/frontend/ingest；WSL docker 未碰。

### 验收 3：范围核对（原始输出）

```
=== 本任务提交 ===
git show --name-only --format="" 4dd8144 → 21 项
=== out-of-scope in my commit ===
0
=== 区间说明（双 BASE，并行前端道在途）===
开工 HEAD（brief BASE）= 01236fd
86a1c09 TASK-0059 设备详情页：状态区/实时/曲线/来源追溯（前端道）——并行提交，本任务未碰/未 add/未提交
4dd8144 TASK-0045 本任务提交（父节点 86a1c09）
git diff --name-only 01236fd..HEAD 共 34 项 = 21 calc/**（本任务）+ 13 frontend/**（TASK-0059）
=== calc 工作区 ===
git status --short calc/ → 空
=== 未跟踪（开工既存/并行，本任务未触碰）===
?? .agents/   ?? .tmp-t0044v/
```

评审请按提交哈希 `4dd8144` 判断归属（21 项全 `calc/**`），不按区间首末（区间混入并行前端 0059 的 13 项）。

---

## 三、未做项与原因

1. **调度取向：自实现调度线程（非 XXL-Job）——brief「调度裁决」明定**：TASK 原文「XXL-Job 调度」在 calc 独立 Spring Boot 4 工程不可行（无 XXL-Job 依赖、无 backend sys_job 访问）。按 0041 PeriodicScheduler 先例在 calc 内自实现单线程 daemon 调度（javadoc 载明取向，tick/grace 由 env 可配），**未引入 XXL-Job 依赖、未改 backend sys_job**。依 brief 第 15 行「登记字面差异，不视为偏离缺陷」——本项即字面差异登记，非实现缺口。
2. **st_metric_value 契约字面差异：四元组 tag 而非「六元组/calendar_type tag」，完整度不入物理列**：brief 决策 4 写「标签含 metric_code/instance_code/period/日历口径」「六元组 tag 逐值精确断言」「结果行含完整度字段」，但冻结契约 `resources/contract/tdengine-super-tables.md` v1.0.0 与详细设计 §1.7/§2.2 明确：st_metric_value 列=`(ts TIMESTAMP〔窗口起始〕, value DOUBLE, yoy DOUBLE, mom DOUBLE)`、标签=`(tenant_id, metric_code, instance_code, period)`（4 元组）；**无 calendar_type 标签、无完整度/quality 列**。backend `OpenApiTimeseriesClient` 读侧亦逐字读 4 列、按 metric_code+period 标签过滤。**裁决：brief 自身「按超级表契约逐字」为最高约束**——物化行严格落冻结契约 4 元组 + 4 列；`full:` 前缀（0049 F-02 st_item_value 教训）本就不适用于指标行（metric_code 标签即定位键）；calendar_type 保留在窗口唯一键与 `MetricValue.calendarType()` 内存载荷，完整度/incomplete/noData 由 `MetricResult` 与 `/calc/v1/status.metric` 承载（live L8 实证）。按 brief「契约冻结发现问题报告并停」的字面差异处置：本差异经上游冻结稿+设计 PDF+backend 读侧三方核实**非契约冲突**（brief 措辞混用 st_item_value 概念），未改契约文件，登记于此；若评审要求物理列扩完整度/tag 扩日历口径，须先升版契约（CONV-03）后另开任务。
3. **真机 TDengine st_metric_value 写入 + 读回（验真）**：REST 6041 仍故障（HTTP=000、6s/15s timeout/reset，服务器侧 TD-0053 登记）；按 brief 决策 6 与 0042/0049 同口径，真机写入列为环境恢复后交付条件，**当前未验真、不宣称真实写入可用**。
4. **指标定义/物化值从快照自动装载**：本交付 `MetricRuntime.setDefinitions()`/`source()` 为注入缝，live 由 `/calc/v1/metric/observe` 驱动缝供数；真实 trigger 由调度线程按定义周期发起，但定义从 `iot_metric` 快照装载、值从 0042 ResultSink 数据回填的自动接线属演进位（与 0041 MONTH/YEAR 生产未接线同族）——契约快照已含 metrics 对象（0044 已验快照纳入），接线不改变本任务交付的窗口/聚合/完整度/物化写口径。
5. **完整度「应到周期值」精确分母**：当前按 `expectedCount` 入参（DAY 平均固定 24，其余按 sourceKeys 数）计算；点位/数据项停用时段剔除已支持（`MetricValueStore.disable`），完整「应到集」从快照/档案下发的精确接线为演进位（本任务口径与 TDD 断言自洽）。
6. **yoy/mom 同比环比伴随值**：冻结表列已就位（st_metric_value 列含 yoy/mom），本任务物化写按「无值写 null」留位；同比/环比计算归看板/后续道（brief 决策 5 不写 Redis 指标缓存、0053/0060 后续消费），未在本任务假值填充。
7. **0046 迟到修正重算**：宽限期接口已提供（`MetricWindow.due`/grace env），迟到数据触发 BACKFILL 重算归 TASK-0046；本任务只提供窗口接口与同 ts 覆盖写幂等能力。
8. **TASK 状态**：按硬约束未把 TASK-0045 置「已完成」，保持「进行中」待评审；未改 workitems 库（本报告唯一例外）。

**TDengine 真机补验条件（环境恢复后必须执行，当前属用户侧动作）**：

1. 建 scratch 租户库（`CREATE DATABASE IF NOT EXISTS iot_t45scratch KEEP 1825d DURATION 10d BUFFER 256` + st_metric_value STABLE，逐语句独立下发，20-25s 慢特性）；
2. 经生产 `MetricValueWriter.rest()` 真写一条 st_metric_value 行（验证 Basic 认证 + `/rest/sql` + 真应答 code 解析——契约测试已离线锁定解析）；
3. **读回** `SELECT ts, value, yoy, mom FROM iot_t45scratch.st_metric_value WHERE metric_code='M_AVG' AND \`period\`='DAY' ORDER BY ts DESC LIMIT 1`（period 保留字反引号，BUG-0004 同口径）——断言 value（含比值 4.2857/日均 23.0）、yoy/mom、四元组 tag 与写侧一致；
4. 同窗口再写一次 → 读回行数/同 ts 覆盖口径验证幂等；
5. DROP scratch 库、清理。

在 1-4 完成前，st_metric_value 的「真实写入可用」标记为**未验真（环境阻断）**。

---

## 四、偏离计划之处

1. **调度取向字面差异（非缺陷）**：self-managed 调度线程替代 XXL-Job——brief「调度裁决」明定取向并登记为字面差异，见 §三-1；非计划外偏离。
2. **st_metric_value tag/列字面差异（非契约冲突）**：四元组 tag 而非 brief 措辞「六元组/calendar_type tag」、完整度不入物理列——按冻结契约逐字实现，三方核实（冻结稿+详细设计+backend 读侧），登记见 §三-2；若评审认定 brief 措辞优先，则须先升版契约（CONV-03）后另开任务，本任务不改冻结契约。
3. **TDD 红转绿留痕（逐修复附「为什么现在对了」）**：见 §验收 1 的 TDD 段——3 项真实断言失败（测试夹具 3 参/4 参 put 误用、sourceKeys 精确键 vs 前缀、宽限>窗长死区）+ 1 项 live NPE（比值缺键），每项红→修→绿与原因已逐条记录。
4. **回溯最近到期窗（调度语义细化）**：初版调度只看「上一窗」，宽限 3 天>日窗 1 天时上一窗永不到期（契约测试真红 `expected 1 but was 0`）；改为向前回溯找最近 `end+grace<=now` 的窗并提交一个（不批量补算，批量补算归 0046）。为什么现在对了：宽限期语义是「数据到达容差」，不是「跳过日指标产出」；回溯单窗保证每次到期都有产出且幂等，批量补算的复杂度留给迟到修正道。
5. **观测驱动口 `/calc/v1/metric/observe`（新增面）**：brief 只要求 `/calc/v1/status` 增观测视图，但真机 TDengine 故障 + 数据定义未接线使 live 无法端到端驱动；新增 /calc/v1/** 内网观测驱动缝（非契约命名空间、无鉴权、与既有 status 面同源）供 live 注入定义+物化值触发一次计算。为什么现在对了：沿 0041/0042 既有「内网观测面」边界，不污染 /internal/v1/** 契约路径、不写真库，契约断言仍由记录式写口/生产类完成。
6. **并行提交 HEAD 前移（双 BASE 登记）**：开工 HEAD=`01236fd`，期间前端道 0059 提交 `86a1c09`（13 项 frontend/**）——本任务未参与、未 add/未提交；本任务提交 `4dd8144` 父节点为 `86a1c09`。范围核对以提交哈希为准（21 项全 calc/**），区间 `01236fd..HEAD` 混入并行前端 13 项属并发事实（同 0042 修订轮 M-3 双 BASE 处置）。
7. **假后端 18045 而非 backend 8098**：brief 允许 live 用假端点 + 真 Redis；并行前端道 0059 在途且 backend 本轮不涉及，用独立假端点提供受控空快照（仍经 calc 真 HTTP 客户端 + 真 SHA-256 校验 + 真 ACK 闭环，非 mock calc 侧），不引入 backend 依赖。8095/18045 起前探测 FREE、用毕即停、终态 FREE。

---

## 五、遗留问题

1. **真 TDengine 写入联调（环境阻断）**：条件见 §三末；黄河 6041 恢复前 st_metric_value 真实写入标记未验真。
2. **指标定义/物化值自动接线**：`setDefinitions`/`source` 为注入缝，从快照 iot_metric 与 0042 ResultSink 的自动回填为演进位（口径已由本任务锁定，接线不改实现）。
3. **完整应到集精确分母**：当前 expectedCount 入参口径自洽（DAY 平均 24 硬口径），停用时段剔除已支持；按点位/数据项档案下发的精确应到集为演进位。
4. **yoy/mom 计算**：列已留位（null），同比环比生产值归后续看板/推送道（0053/0060），本任务不假值填充。
5. **批量迟到补算**：宽限接口与同 ts 覆盖已就位，BACKFILL 批量重算归 0046。
6. **多实例一致性**：记录式写口与进程内 ValueStore 为单实例口径，多实例共享判据（Redis 化）为演进位（同 0041/0042/0049 演进位）。
7. **区间范围说明**：`01236fd..HEAD` 含并行 TASK-0059 的 13 项 frontend/**；评审按提交 `4dd8144` 判断本任务归属。
8. **TASK 状态**：保持「进行中」待评审，未置「已完成」；未回写 workitems 库（本报告除外）。

---

## 一句话结论

**通过**——验收 1（`cd calc && mvn test` 220 测试 0F0E3Skip BUILD SUCCESS，195 基线逐类零回归 + 25 新增，红转绿 4 处含 live NPE 修复留痕），验收 2（live 冒烟全序列 `RESULT fail=0`：比值 30/7≠逐条均值、日均 23.0≠24.0、级联 SUM、60% 完整度带标记、无数据 value=NULL 完整度 0 不写假 0、MONTH 窗口 ts=月初无中间快照、st_metric_value 四元组 tag 逐值精确且无 full:/无分号、同 ts 覆盖幂等、全停归零），验收 3（提交 `4dd8144` 21 文件全落 `calc/**` 越界 0、calc 工作区干净）三条全部以原始输出为证通过；调度自实现（非 XXL-Job）与四元组 tag/完整度不入物理列按 brief「登记字面差异」处置、真 TDengine 写入因 REST 故障列环境恢复后条件，均不阻断三条验收判定，TASK 保持「进行中」待评审。
