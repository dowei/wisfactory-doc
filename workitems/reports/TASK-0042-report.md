# TASK-0042 报告：DELTA/INTEGRAL 算子与同构存储

- 执行者：TASK-0042 执行者（no-subagents；计算引擎道——只改 `calc/**`）
- BASE：`331f860`（开工时主仓 main HEAD，按实施 brief「BASE=当前 HEAD」记录）；**diff/评审 BASE=`e49fea8`**（本任务提交 `e387c3b` 的实际父节点，并行 TASK-0050 提交；M-3 订正——两值并存，范围核对以提交哈希为准，详见修订轮 R2-M3）
- 交付提交：`e387c3b`（`TASK-0042: DELTA/INTEGRAL 原生算子与同构存储（st_item_value+Redis缓存+六元组tag+无full前缀）`；**24 文件全部 `calc/**`，本提交越界 0**）
- 执行日期：2026-11-24
- 并行事实：区间内 `e49fea8` 为 backend 道 TASK-0050（改 `backend/**` 状态读口），由并行执行者提交——本任务全程**未碰 backend/、未跑 backend mvn**，未 add/未提交其任何文件。
- 环境事实：Redis 宿主 phpStudy 3.0.504@6379（真 SET/GET/KEYS/DEL 实测）；8095 归 calc（起前探测 FREE、用毕即停、终态 FREE）；TDengine 192.168.31.145:6041 REST 仍故障（`curl -m 6 ... /rest/sql --data-raw "SELECT 1"` → HTTP=000、time=6.01s，服务器侧，TD-0053 登记），真机写入+读回列为环境恢复后交付条件，**不宣称已验真**；WSL docker 容器未碰。
- 结论先行：**验收 3 条全部通过**——mvn 194 测试 0F0E3Skip BUILD SUCCESS（0049 的 163 基线逐类零回归 + 新增 31）；live 冒烟全序列绿（算子输出五异常+三场景 22 断言、Redis→同构写 SQL 端到端 quality 0/8/9、Redis 点位缓存 JSON、全停归零）；范围核对本提交 24 文件全落 `calc/**`。

---

## 一、改动文件清单（本提交 24 项，全部 `calc/**`）

### 1.1 新增——原生算子包 `calc/src/main/java/net/otlab/calc/operator/`（4）

| 文件 | 职责 |
| --- | --- |
| `DeltaOperator.java` | DELTA 差分原生算子：正常差分/回退打码（码8）/换表 PARAM 分段（ADR-0010 新基线）/溢出归零（码1）/跨周期标记（码9）/坏样本保留上游码；按 key 独立基线，ConcurrentHashMap 线程安全。 |
| `IntegralOperator.java` | INTEGRAL 梯形积分原生算子：真实 ts 间隔梯形积分逐段累加/缺口不积分（质量非0 跳过、重建基线）/停机零增量（码0）与断线区分；`isHandwritable()=false`（ADR-0010 不开放手写）。 |
| `EnergyDerivation.java` | 能源计量模板派生：绑累计→DELTA、绑流量→INTEGRAL；绑定方向正反翻转正负号（只改方向即换向）；模板默认/覆盖三方合并口径（实例 null 继承模板）。 |
| `OperatorEvaluator.java` | 算子桥：DELTA 公式调用接原生差分（质量码透传）、INTEGRAL 预置派生入口、换表分段登记；DELTA 可引用/INTEGRAL 禁手写的判定缝。 |

### 1.2 新增——同构存储包 `calc/src/main/java/net/otlab/calc/storage/`（5）

| 文件 | 职责 |
| --- | --- |
| `ItemValue.java` | 同构写载荷（ts/value/quality + 六元组 tag 维度），DIRECT/FORMULA/ROLLUP 仅 source_type 不同。 |
| `TdengineItemValueWriter.java` | st_item_value 真写口：逐语句 REST（0029 多语句静默空操作）、库名全限定、USING TAGS 子表首写、taosAdapter code 解析（0049 F-01）；**full_key=instance_code+'.'+data_code 无 full: 前缀**、六元组 tag 顺序锁定；SqlRunner 注入缝。 |
| `RedisItemValueCache.java` | 数据项点位缓存：`iot:item:{tenant}:{fullKey}` JSON `{fullKey,value,quality,ts}`，RESP2 零依赖客户端（0041/0049 同款字节帧解析），失败 WARN 不阻塞。 |
| `HomogeneousResultSink.java` | 同构写 Sink：结果按 source_type 写 st_item_value + Redis 缓存，失败结果不写假 0，单条写失败 WARN 隔离、装饰器链可下接状态判定。 |
| `StorageRuntime.java` | 运行时装配：算子桥 + 同构写 Sink 装饰器 + 记录式写口（捕获 INSERT SQL 供契约断言/观测），`/calc/v1/status.storage` 观测面。 |

### 1.3 修改（7）

`CalcEngineApplication.java`（storageRuntime Bean + 算子/同构写 Sink 注入执行器，默认关）、`config/CalcProperties.java`（operators-enabled/homogeneous-storage-enabled/tenant-id 三项 env，默认关）、`executor/ArithmeticEvaluator.java`（函数求值器注入缝，默认 null 保持 0041 行为）、`executor/DualTriggerExecutor.java`（DELTA 原生求值分支，默认无算子桥时既有行为不变）、`executor/ExecutorRuntime.java`（算子+存储 Sink 构造重载）、`web/CalcStatusController.java`（storage 观测面，null-safe）、`resources/application.yml`（env 映射）。

### 1.4 新增——契约测试（8）

| 文件 | 用例数 | 覆盖 |
| --- | --- | --- |
| `operator/DeltaOperatorContractTest.java` | 6 | 正常差分/回退码8/换表分段/溢出码1/跨周期码9/坏样本，恢复后从最近样本差分。 |
| `operator/IntegralOperatorContractTest.java` | 4 | 真实间隔梯形/缺口不积分/停机零增量区分断线/INTEGRAL 不开放手写。 |
| `operator/EnergyDerivationContractTest.java` | 5 | 累计绑→DELTA、流量绑→INTEGRAL、方向换向取负、模板三方合并默认/覆盖。 |
| `executor/OperatorIntegrationContractTest.java` | 3 | DELTA 经执行器原生求值且基线跨周期连续/无算子桥 0041 行为不变/INTEGRAL 白名单禁手写。 |
| `storage/TdengineItemValueWriterContractTest.java` | 7 | 六元组 tag 精确断言无 full: 前缀/三来源同构/异常码原样写/逐语句单 SQL/注入缝捕获/taosAdapter 应答解析/失败上抛。 |
| `storage/HomogeneousResultSinkContractTest.java` | 5 | 双写口/失败不写假0/存储失败隔离/Redis 失败隔离/null metadata 兜底（live 红转绿）。 |
| `config/CalcPropertiesTest.java`（修改） | +1 | TASK-0042 三项 env 默认值零回归断言。 |

新增用例合计 31（6+4+5+3+7+5+1），163 基线 → 194。

---

## 二、验收命令原始输出

### 验收 1：`cd calc && mvn test` → BUILD SUCCESS、0 失败 0 错误

终验（提交 `e387c3b` 对应源码，HEAD=e387c3b）Results 段逐类原始输出（CR 行尾已去）：

```
[INFO] Running net.otlab.calc.ack.HttpAckSenderTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.598 s -- in net.otlab.calc.ack.HttpAckSenderTest
[INFO] Running net.otlab.calc.CalcEngineApplicationTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 6.701 s -- in net.otlab.calc.CalcEngineApplicationTest
[INFO] Running net.otlab.calc.config.CalcPropertiesTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.093 s -- in net.otlab.calc.config.CalcPropertiesTest
[INFO] Running net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.025 s -- in net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Running net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.060 s -- in net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Running net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.091 s -- in net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Running net.otlab.calc.executor.OperatorIntegrationContractTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.022 s -- in net.otlab.calc.executor.OperatorIntegrationContractTest
[INFO] Running net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.010 s -- in net.otlab.calc.executor.PeriodicSchedulerContractTest
[WARNING] Tests run: 1, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0 s -- in net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[INFO] Running net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.027 s -- in net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Running net.otlab.calc.graph.CycleDetectorTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.calc.graph.CycleDetectorTest
[INFO] Running net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.050 s -- in net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Running net.otlab.calc.graph.GraphBuilderTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.073 s -- in net.otlab.calc.graph.GraphBuilderTest
[INFO] Running net.otlab.calc.operator.DeltaOperatorContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.calc.operator.DeltaOperatorContractTest
[INFO] Running net.otlab.calc.operator.EnergyDerivationContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.014 s -- in net.otlab.calc.operator.EnergyDerivationContractTest
[INFO] Running net.otlab.calc.operator.IntegralOperatorContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.014 s -- in net.otlab.calc.operator.IntegralOperatorContractTest
[INFO] Running net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.089 s -- in net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Running net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.366 s -- in net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Running net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.031 s -- in net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Running net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.073 s -- in net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Running net.otlab.calc.status.StatusEvaluatorContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.055 s -- in net.otlab.calc.status.StatusEvaluatorContractTest
[WARNING] Tests run: 2, Failures: 0, Errors: 0, Skipped: 2, Time elapsed: 0 s -- in net.otlab.calc.status.StatusRedisLiveTest
[INFO] Running net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.040 s -- in net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Running net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.027 s -- in net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] Running net.otlab.calc.storage.HomogeneousResultSinkContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.016 s -- in net.otlab.calc.storage.HomogeneousResultSinkContractTest
[INFO] Running net.otlab.calc.storage.TdengineItemValueWriterContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.026 s -- in net.otlab.calc.storage.TdengineItemValueWriterContractTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 194, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  16.649 s
EXIT=0
```

**0049 的 163 基线零回归**：163 = 8+1+7+4+5+10+2+1(skip)+2+7+17+23+11+13+9+20+8+2(skip)+8+5（0049 修订轮报告逐类一致）；3 Skipped = 0041 env 门控 1 + 0049 env 门控 2（前轮真 Redis 单跑已 PASS，本轮未改这些写口）。新增 31 项（算子 6+4+5、集成 3、存储 7+5、配置 1）。真 Redis 点位缓存另在 live 冒烟中实测写入（见验收 2 步骤 5）。

### 验收 2：live 冒烟全序列（每步原始输出）

假后端（Python HTTPServer，18042，真快照信封 SHA-256 可重算，计算范围 scopeType=2/scopeId=0，节点 `F01.WS01.EQ001.FLOW = DELTA(F01.WS01.EQ001.METER)` + 算子桥/同构写启用）+ calc 起（8095，`CALC_OPERATORS_ENABLED=true CALC_HOMOGENEOUS_STORAGE_ENABLED=true`）。

**步骤 0：起前端口探测 + TDengine 环境探测**

```
=== ports before ===
18042/8095 FREE
=== Redis ping ===
+PONG
=== TDengine REST probe (single, 6s) ===
curl -m 6 -u root:taosdata http://192.168.31.145:6041/rest/sql --data-raw "SELECT 1"
HTTP=000 time=6.014095   → 服务器侧故障（TD-0053），真机写入列为环境恢复后条件
```

**步骤 1：假后端 + calc 起，真哈希可重算 + 图切换 + 记录式写口装配**

```
=== fake backend ===
HASH=5e1ee34979c700fcd50e8af57b0d25bd1a2b35e7712aebb4b7579f1967f6af15  （首次 arrival 版）
（最终 3600s 周期版：HASH=e836aa47750f0f47e7006f6bc7c937f8c3b9f96bd82ef49662f915d6dff3203f；data.snapshot 逐字重算 == 信封 snapshotHash，一致）

$ java -jar calc-engine.jar（CALC_EXECUTOR_ENABLED=true / CALC_OPERATORS_ENABLED=true /
  CALC_HOMOGENEOUS_STORAGE_ENABLED=true / CALC_TENANT=t42 / CALC_TENANT_ID=1）
INFO ... CalcEngineApplication : Started CalcEngineApplication in 1.599 seconds
INFO ... RedisArrivalSubscriber : [CALC-ARRIVAL] SUBSCRIBE 已发送 channel=iot:newvalue:t42
INFO ... ExecutorRuntime       : [CALC-RUNTIME] 双触发执行器已启动 tenant=t42 channel=iot:newvalue:t42
INFO ... SnapshotReconciler    : [CALC-RECONCILE] 依赖图已切换 version=42 hash=e836aa47750f0f47e7006f6bc7c937f8c3b9f96bd82ef49662f915d6dff3203f nodes=2 edges=1 topo=2
=== port 8095 ===
TCP 0.0.0.0:8095 LISTENING 61892
```

**步骤 2：算子输出断言（生产 jar 内类直跑，五异常 + 三场景 + 派生换向 + SQL 契约，22/22 PASS）**

```
$ java -cp .;calc-engine.jar LiveProbe   （脚本内均为生产类：DeltaOperator / IntegralOperator / EnergyDerivation / StorageRuntime.RecordingWriter）
=== LIVE PROBE (production classes from calc-engine.jar) ===
PASS DELTA 正常差分 15-10=5 [exp=5.0 got=5.0]
PASS DELTA 正常 quality=0 [exp=0.0 got=0.0]
PASS DELTA 回退不写负值 [exp=0.0 got=0.0]
PASS DELTA 回退码8 [exp=8.0 got=8.0]
PASS DELTA 跨周期不差分 [exp=0.0 got=0.0]
PASS DELTA 跨周期码9 [exp=9.0 got=9.0]
PASS DELTA 溢出归零 [exp=0.0 got=0.0]
PASS DELTA 溢出码1 [exp=1.0 got=1.0]
PASS DELTA 换表PARAM分段建新基线
PASS INTEGRAL 真实间隔 (2+4)/2*3s=9 [exp=9.0 got=9.0]
PASS INTEGRAL 缺口不积分 [exp=0.0 got=0.0]
PASS INTEGRAL 缺口码3 [exp=3.0 got=3.0]
PASS INTEGRAL 停机零增量
PASS INTEGRAL 停机码0（与断线严格区分） [exp=0.0 got=0.0]
PASS 派生绑累计→DELTA
PASS 派生正绑 +1 [exp=1.0 got=1.0]
PASS 派生反绑 -1（只改方向即换向） [exp=-1.0 got=-1.0]
PASS 派生反绑取负 [exp=-5.0 got=-5.0]
SQL=INSERT INTO iot_t42.D_F01_WS01_EQ001_FLOW USING iot_t42.st_item_value TAGS (1, 'F01.WS01.EQ001.FLOW', 'F01.WS01.EQ001', 'FLOW', 'OTHER', 2) VALUES (4000, 5.0, 0)
PASS full_key 无 full: 前缀
PASS SQL 无 full: 前缀
PASS 六元组 tag 精确匹配
PASS 逐语句单 SQL 无分号
RESULT pass=22 fail=0
```

**步骤 3：到达驱动 DELTA 异常序列（真 Redis PUBLISH → 执行器 → 同构写 SQL，每事件读 `/calc/v1/status.storage`）**

```
$ python driver.py（PUBLISH iot:newvalue:t42 {"pointCode":"F01.WS01.EQ001.METER",...}；calcPeriod=3600s 供跨周期判定）
基线 value=100 ts=1000 | writes=1 | SQL ... TAGS (1,'F01.WS01.EQ001.FLOW','F01.WS01.EQ001','FLOW','OTHER',2) VALUES (1000, 0.0, 0)
正常 105-100=5          | writes=2 | SQL ... VALUES (2000, 5.0, 0)
回退 102<105            | writes=3 | SQL ... VALUES (3000, 0.0, 8)     ← 回退码8、不写负值
（跨周期小间隔不误判）   | writes=4 | SQL ... VALUES (9000, 1.0, 0)
恢复 108-103=5          | writes=5 | SQL ... VALUES (10000, 5.0, 0)
真实跨周期 500@100000 → 505@6000000（gap=5.9M ms > 3600s×1.5 容差）：
基线                    | writes=6 | SQL ... VALUES (100000, 392.0, 0)
跨周期                  | writes=7 | SQL ... VALUES (6000000, 0.0, 9)  ← 跨周期码9、不差分
```

→ 五异常 live 端到端成立：值/质量码原样落 SQL（0/5/8/9），六元组 tag 逐行精确、full_key 无前缀、逐语句单 INSERT。

**步骤 4：同构写 SQL 契约断言（步骤 2/3 原始输出）**

```
SQL 全文：INSERT INTO iot_t42.D_F01_WS01_EQ001_FLOW USING iot_t42.st_item_value
  TAGS (1, 'F01.WS01.EQ001.FLOW', 'F01.WS01.EQ001', 'FLOW', 'OTHER', 2) VALUES (6000000, 0.0, 9)
六元组逐值：tenant_id=1 / full_key=F01.WS01.EQ001.FLOW（无 full: 前缀）/ instance_code=F01.WS01.EQ001 /
  data_code=FLOW / category=OTHER / source_type=2（FORMULA，同源同构）
0 个分号；无 USE；库名 iot_t42 全限定；子表首写 USING st_item_value TAGS
```

**步骤 5：Redis 点位缓存 + 派生换向**

```
$ KEYS iot:item:t42:*    → *1 / $32
$ GET iot:item:t42:F01.WS01.EQ001.FLOW
  → {"fullKey":"F01.WS01.EQ001.FLOW","value":0.0,"quality":9,"ts":6000000}   ← 与跨周期 SQL 同值同码
派生换向：见步骤 2「派生反绑 -1」「派生反绑取负」两断言（只改方向正负号翻转，算子/data_code 不变）
```

**步骤 6：全停归零 + 冒烟数据清理**

```
=== STOP sequence ===
成功: 已终止 PID 为 61892 的进程。（calc）
成功: 已终止 PID 为 70208 的进程。（fake backend）
=== ports after ===
BOTH FREE
=== cleanup smoke redis keys ===
keys found: 1  → DEL iot:item:t42:F01.WS01.EQ001.FLOW :1
=== leftover procs on smoke ports ===
NONE
```

临时冒烟目录 `D:\workspace\WisFactory\.tmp-t0042\`（fake_backend.py / driver.py / LiveProbe.java / 日志）测后已整体删除；未写真实 TDengine（SQL 由记录式写口捕获，真机写入列环境恢复后条件）；未碰 backend/frontend/ingest；WSL docker 未碰。

### 验收 3：范围核对（原始输出）

```
=== git show --name-only --format="" e387c3b ===
calc/README.md
calc/src/main/java/net/otlab/calc/CalcEngineApplication.java
calc/src/main/java/net/otlab/calc/config/CalcProperties.java
calc/src/main/java/net/otlab/calc/executor/{ArithmeticEvaluator,DualTriggerExecutor,ExecutorRuntime}.java
calc/src/main/java/net/otlab/calc/operator/{DeltaOperator,EnergyDerivation,IntegralOperator,OperatorEvaluator}.java
calc/src/main/java/net/otlab/calc/storage/{HomogeneousResultSink,ItemValue,RedisItemValueCache,StorageRuntime,TdengineItemValueWriter}.java
calc/src/main/java/net/otlab/calc/web/CalcStatusController.java
calc/src/main/resources/application.yml
calc/src/test/java/net/otlab/calc/config/CalcPropertiesTest.java
calc/src/test/java/net/otlab/calc/executor/OperatorIntegrationContractTest.java
calc/src/test/java/net/otlab/calc/operator/{DeltaOperatorContractTest,EnergyDerivationContractTest,IntegralOperatorContractTest}.java
calc/src/test/java/net/otlab/calc/storage/{HomogeneousResultSinkContractTest,TdengineItemValueWriterContractTest}.java
=== count ===
24
=== out-of-scope in MY commit ===
0
=== git status --short calc/ ===
（空）
=== 区间 BASE..HEAD 说明 ===
git diff --name-only 331f860..HEAD 共 39 项，其中 15 项 backend/** 为并行 TASK-0050 提交 e49fea8 所改，
与本任务无关；本任务提交 e387c3b 24 项全部 calc/**，越界 0。
=== 工作区未跟踪（开工前既有，本任务未触碰） ===
?? .agents/   ?? .tmp-t0044v/
```

---

## 三、未做项与原因

1. **真机 TDengine st_item_value 写入 + 读回（验真）**：REST 6041 仍故障（HTTP=000、15s 超时/6s reset，服务器侧，TD-0053 登记）；按 brief 决策 6 与 0049 同口径，真机写入列为环境恢复后交付条件，**当前未验真、不宣称真实写入可用**。补验条件见 §五末。
2. **DELTA 溢出量程（满量程）在 live 端的真值注入**：冻结快照 DataItem 无 full-scale 专属字段（`validMax` 为有效范围非满量程），算子支持 `maxRange` 入参且契约测试已锁定「超量程→0+码1」，但执行器接线本期未从点位档案取满量程（点位不随计算快照下发，派生规则 7）——登记演进位；live 端溢出场景由生产类直跑探针（步骤 2）验真。
3. **INTEGRAL 经执行器预置公式的端到端 live**：ADR-0010 INTEGRAL 不开放手写、仅随能源计量预置公式提供；本期快照无预置 INTEGRAL 数据项样本，场景由算子契约测试 + 生产类直跑探针（步骤 2）锁定，执行器主链只接 DELTA 引用（预置 INTEGRAL 由 `OperatorEvaluator.integral()` 提供、待 0081 三方合并预置公式落地后接线）。
4. **多实例幂等/缓存一致性**：记录式写口与点位缓存未变不重写为进程内；多实例 Redis 化归演进位（与 0041/0049 同族登记）。
5. **真 Aviator 表达式引擎接入**：`ArithmeticEvaluator` 增量支持 DELTA 函数调用，嵌套/多算子组合的真 Aviator 归 FEAT-0012；本期能源计量预置为单 DELTA 调用（brief 口径）。
6. **TASK 状态**：按硬约束未把 TASK-0042 置「已完成」，保持「进行中」待评审；未改 workitems 库（本报告唯一例外）。

**TDengine 真机补验条件（环境恢复后必须执行，当前属用户侧动作）**：

1. 建 scratch 租户库（`CREATE DATABASE IF NOT EXISTS iot_<scratch> KEEP 1825d DURATION 10d BUFFER 256` + 四张 STABLE，逐语句独立下发，20-25s 慢特性）；
2. 经生产 `TdengineItemValueWriter.rest()` 真写一条 st_item_value 行（验证 Basic 认证 + `/rest/sql` + 真应答 code 解析——契约测试已离线锁定该解析）；
3. **读回** `SELECT ... WHERE full_key = 'F01.WS01.EQ001.FLOW' ORDER BY ts DESC LIMIT 1`（用已锁定的规范 fullKey）——断言 `value`/`quality`（含 8/9 异常码）、`source_type=2`、六元组 tag 与写侧一致；
4. 同 fullKey 再写一次 → 读回行数/幂等口径按 st_item_value 同窗口覆盖语义验证；
5. DROP scratch 库、清理。

在 1-4 完成前，st_item_value 的「真实写入可用」标记为**未验真（环境阻断）**。

---

## 四、偏离计划之处

1. **接入取向选默认关闭的 opt-in 构造/Bean，而非改 0041 主链默认行为**（brief 决策 1/3 + 零回归硬指标）：`DualTriggerExecutor` 加带 `OperatorEvaluator` 的重载构造，原构造保持「函数调用隔离失败」（0041 既有测试 `formulaFunctionNotSupportedIsIsolatedAsFailure` 逐字不变仍绿）；算子/同构写由 `CALC_OPERATORS_ENABLED` / `CALC_HOMOGENEOUS_STORAGE_ENABLED` 控制（默认 false）。为什么现在对了：单点 opt-in 使 163 基线逐类零回归，同时 live 可按 env 打开全链验证；既有部署不引入新后台线程/写口。
2. **TDD 红转绿留痕（逐修复附「为什么现在对了」）**：
   - **DELTA/INTEGRAL 算子**：先写 10 项契约测试 → 编译期红（`程序包 DeltaOperator 不存在`）；实现五异常/三场景原生算子 → 绿（10/10）。为什么现在对了：把回退/换表/溢出/跨周期/坏样本做成可执行断言，负值与冒充单周期流量在数学上不可能通过；INTEGRAL 用真实 ts 间隔而非固定周期，缺口段累计不变、恢复重建基线，断线与停机由 Kind+quality 严格分离。
   - **同构写 SQL**：先写 7 项契约测试 → 编译期红；实现逐语句 REST + full_key 规范 + 六元组 tag → 绿（7/7）。为什么现在对了：六元组逐值断言使 full: 前缀/错位/漏 tag 任一漂移机械变红，杜绝 0049 F-02 同类缺陷再逃逸；taosAdapter code 正则解析（冒号后空格容错）离线锁定真应答。
   - **StorageRuntime 空 metadata NPE（live 驱动红转绿）**：live 首轮到达事件被订阅器吞掉——日志 `[CALC-ARRIVAL] 事件解析失败 ... because "this.metadata" is null`（真红，writes=0）；修 `HomogeneousResultSink` 构造器 `metadata==null → defaultMetadata()` 兜底 + 补回归测试 `nullMetadataFallsBackToDefaultProjection`，live 重跑 writes 逐事件递增（绿）。为什么现在对了：默认投影把 fullKey 末段拆为 dataCode、instance 前缀为 instanceCode，构造期即消除空指针，避免失败在异步订阅器里被 fail-soft 静默吞掉。
   - **测试夹具断言笔误**：派生换向测试初版把 reverse.apply(5) 断言成 +5（真红 expected 5 was -5），按语义订正为 -5（绿）——属断言错误、非实现缺陷。
3. **冒烟载体：calcPeriod 从 1s 改为 3600s**：初版 1s 周期项每秒 tick 触发 DELTA，与到达序列交织污染基线（5 次到达产出 7 行、质量码被周期 tick 覆盖）。改为 3600s（保留期望周期供跨周期判定、烟窗内不 tick）后，5 次到达恰 5 行、每事件质量码可逐条断言。属冒烟载体构造修正，未改 calc 代码/契约。
4. **假后端端点用 18042 而非 backend 8098**：brief 允许 live 冒烟用 Python 假端点 + 真 Redis；并行 0050 在途改 backend/** 且 backend 道不跑 mvn，故用独立假端点提供受控快照（仍经 calc 真 HTTP 客户端 + 真 SHA-256 校验 + 真 ACK 消费，非 mock calc 侧），不引入 backend 依赖。
5. **Window 观测 `tickSeconds="see-config"`**：沿用 0041 既有占位（本次未改，避免与 0041 基线测试冲突）；登记 deferred，归 FEAT-0015 状态页完善。

---

## 五、遗留问题

1. **真 TDengine 写入联调**（环境阻断）：条件见 §三末；在黄河 6041 恢复前标记未验真。
2. **DELTA 满量程来源**：点位档案满量程未随计算快照下发（派生规则 7），`maxRange` 入参已由算子/契约测试支持，执行器取值为演进位；管理面若需 live 溢出验真需在快照契约升版（CONV-03）增加 fullScale 字段或另行走点位档案读侧。
3. **INTEGRAL 预置公式端到端**：待 0081 三方合并把能源计量预置 INTEGRAL 数据项下发后接线（`OperatorEvaluator.integral()` 已备）；白名单禁手写已锁定。
4. **真实 ResultSink 写口切换**：`StorageRuntime` 默认记录式写口（TDengine 环境所限），真 `TdengineItemValueWriter.rest()` / `RedisItemValueCache` 已实现并经契约测试/live Redis 写实测，真机切换只需修改 app Bean 注入（演进位）。
5. **多实例一致性**：同构写/缓存幂等为进程内口径，多实例部署需 Redis 侧共享判据（同 0041/0049 演进位）。
6. **区间范围说明**：BASE..HEAD 含并行 TASK-0050 的 backend 15 项；验收核对以本任务提交 `e387c3b` 为准（24 项全 calc/**），评审时请按提交哈希而非区间首末判断归属。
7. **TASK 状态**：保持「进行中」待评审，未置「已完成」；未回写 workitems 库（本报告除外）。

---

## 一句话结论

**通过**——验收 1（`mvn test` 194 测试 0F0E3Skip BUILD SUCCESS，0049 的 163 基线逐类零回归；31 新测试红转绿，含 live 驱动的 null-metadata NPE 修复留痕），验收 2（live 冒烟全序列：生产类探针 22/22 断言覆盖 DELTA 五异常+INTEGRAL 三场景+派生换向+六元组 tag/full_key SQL 契约；真 Redis→执行器→st_item_value INSERT 端到端 quality 0/5/8/9 逐事件断言；Redis 点位缓存 JSON 与 SQL 同码；全停归零无残留），验收 3（提交 `e387c3b` 24 文件全落 `calc/**`、本提交越界 0、工作区 calc/ 干净）三条全部以原始输出为证通过；真 TDengine 写入+读回因 REST 故障未做（已登记环境恢复后补验条件），不影响三条验收判定，TASK 保持「进行中」待评审。

---

# 修订轮（退回一轮 §12/§13：1 Important〔I-1〕+ 3 Minor）

- 评审输入：`TASK-0042-review.md`（1 Important I-1 DELTA 复合表达式静默错值；3 Minor M-1 溢出码主链不可达 / M-2 INTEGRAL ts 跳变缺口 / M-3 报告 BASE 字段口径；0 Critical；评审重点 7 TDengine 未验真判定「充分，不阻断」）。
- 协调者裁决：**I-1 必修（fail-closed 复合表达式）**；M-1/M-2 登记 deferred；M-3 报告订正。
- 修订轮 BASE：`e387c3b`（被评审提交）
- 交付提交：`01236fd`（`TASK-0042 修订轮: I-1 DELTA复合表达式fail-closed（不丢后缀运算/不发假值）`；2 文件全部 `calc/**`，越界 0）
- 修订轮测试：**194 → 195**（+1 I-1 负向测试），194 基线逐类零回归；全量 195 测试 0F0E3Skip BUILD SUCCESS。

## R1. I-1（Important）DELTA 复合表达式静默错值 —— 红转绿

- **核实**：`DualTriggerExecutor.java:192-195,229-246`：`containsFunction` 广谱命中含 `DELTA(` 的公式后，`evaluateDeltaFormula` 用 `Pattern.compile("DELTA\\s*\\(\\s*([\\w.]+)\\s*\\)").matcher(formula).find()` 只取首个 DELTA(x) 即 `CalcResult.success(key,2,sample)`，丢弃表达式其余部分。0040 `FormulaExpressionValidator` 白名单已接受单参 DELTA 调用（`DELTA(x)/3600` 合法——REQ-0019「用量÷周期时长=平均流量」口径），旧实现会静默算成原始差分且质量码丢失——属静默错值，非隔离失败，违背 CONV-02「不写假值」与 0041「函数调用隔离失败」纪律。
- **红**：新增 `OperatorIntegrationContractTest.compositeDeltaExpressionFailsClosedNotSilentlyWrongValue`（3 种复合形态：`DELTA(x)/3600` / `2*DELTA(x)` / `DELTA(x)+1`，断言隔离失败 `sample=null`、错误原因指向 DELTA 形态）。旧实现 `mvn -q -Dtest='OperatorIntegrationContractTest#compositeDeltaExpressionFailsClosedNotSilentlyWrongValue' test` → `Tests run: 1, Failures: 1`，实测原始输出：

```
[ERROR] net.otlab.calc.executor.OperatorIntegrationContractTest.compositeDeltaExpressionFailsClosedNotSilentlyWrongValue
  复合 DELTA 表达式须隔离失败（不发假值）: DELTA(EQ1.METER.V)/3600 ==> expected: <false> but was: <true>
  at OperatorIntegrationContractTest.java:101
[INFO] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
```

  （`failed=0`、结果 `isSuccess()=true`——旧实现确实写出原始差分，真红。）

- **修**：`evaluateDeltaFormula` 由 `matcher(formula).find()` 改为 `matcher(formula.trim()).matches()` **全串严格匹配纯 `DELTA(key)` 单形态**（首尾空白容忍）；非纯形态抛 `IllegalStateException("DELTA 公式形态不支持（仅接受纯 DELTA(key)，复合表达式归 FEAT-0012 表达式引擎）: "+formula)`，由执行器既有单节点失败隔离（sample=null、不发假 0）。javadoc 同步订正——删除旧「支持 DELTA(x) 作为更大表达式的一部分」的错误自述，写明 I-1 口径与 REQ-0019/CONV-02 违背根因。
- **绿**：`mvn -q -Dtest=OperatorIntegrationContractTest test` → EXIT=0，surefire `tests=4 fail=0 err=0`；实测日志逐条新形态均隔离：

```
ERROR ... [CALC-EXEC] 单节点计算失败（隔离，不阻塞整图）key=EQ1.FLOW.V reason=DELTA 公式形态不支持（仅接受纯 DELTA(key)，复合表达式归 FEAT-0012 表达式引擎）: DELTA(EQ1.METER.V)/3600
ERROR ... [CALC-EXEC] 单节点计算失败（隔离，不阻塞整图）key=EQ1.FLOW.V reason=...: 2*DELTA(EQ1.METER.V)
ERROR ... [CALC-EXEC] 单节点计算失败（隔离，不阻塞整图）key=EQ1.FLOW.V reason=...: DELTA(EQ1.METER.V)+1
（三种复合形态均 failed=1、sample=null；纯 DELTA(EQ1.METER.V) 用例 deltaRunsNativelyThroughExecutorWithBaselineCarriedAcrossPeriods 仍绿，差分 5.0/quality=0）
```

- **为什么现在对了**：`matches()` 要求整个公式串（trim 后）恰为 `DELTA(key)`，`DELTA(x)/3600` 的 `/3600`、`2*DELTA(x)` 的前缀、`DELTA(x)+1` 的后缀都使全串匹配失败，不可能再被截断成「原始差分 + success」；失败沿执行器既有单节点隔离路径产出 `CalcResult.failure`（sample=null、error 非空），下游同构写跳过失败结果不发假 0（CONV-02）。复合表达式求值归 FEAT-0012 表达式引擎（本轮不引 FEAT-0012——范围外），fail-closed 是本期唯一在范围内的正确处置：宁可隔离失败，也不静默错值。

## R2. Minor 处置与取舍

| 编号 | 处置 | 说明 |
| --- | --- | --- |
| M-1（溢出码 1 主链不可达） | **deferred（不硬修）** | 核实：`DualTriggerExecutor.evaluateDeltaFormula` 调 `operators.delta(argKey, current, periodMs, null, false)`，`maxRange` 恒 null；冻结快照 `config-snapshot-schema.md` 数据项对象只有 `validMin/validMax`（有效范围，语义「打码不丢数」非「满量程」），点位对象不随计算范围（scopeType=2）下发；REQ-0019 溢出口径明示「按上限-旧值+新值计算（**需点位档案配满量程值**）」。接线需从点位档案读满量程或升版快照契约增 fullScale 字段——前者越出 calc 职责（点位归 ingest/管理面），后者触 CONV-03 契约冻结变更（报告并停），均超修订范围。**取舍**：保持 deferred，溢出由算子契约测试 6 项 + live 生产类探针（DELTA 溢出码1 PASS）覆盖，执行器主链在点位满量程读侧定版后接线；报告 §三未做项 2/§五遗留 2 已登记同源演进位。 |
| M-2（INTEGRAL ts 跳变缺口） | **deferred** | 评审已标「无法判定」：上游缺口是否恒带 quality=3 补标，工作区资料无法确认；INTEGRAL 主链未接线（ADR-0010 预置公式演进位），当前实现按上游质量码判缺口，若上游契约保证补标则无缺陷。待上游缺口补标契约定版后决定是否需要 ts 跳变容差保护；本轮不改算子语义（避免在没有契约依据时引入新口径）。 |
| M-3（报告 BASE 字段口径） | **报告订正** | 初验报告首部仅记 BASE=`331f860`（实施 brief 口径「开工时 HEAD」），而 diff 包/评审 brief/提交父节点均为 `e49fea8`（并行 TASK-0050 提交，`331f860` 实为 TASK-0049 修订轮）。本修订轮已将首部改为双字段并列：`BASE:331f860`（开工 HEAD）+ `diff/评审 BASE:e49fea8`（本任务提交实际父节点），并注明范围核对以各提交哈希为准——保留两口径各自事实、不再混淆。 |

## R3. 修订轮验收原始输出

### R3.1 全量 `cd calc && mvn test`（提交 `01236fd`）

```
[INFO] Running net.otlab.calc.executor.OperatorIntegrationContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.018 s -- in net.otlab.calc.executor.OperatorIntegrationContractTest
...（其余 25 类逐类数值与初验报告 Results 段一致：8+1+8+4+5+10+2+1(skip)+2+7+17+23+6+5+4+11+13+9+20+8+2(skip)+8+5+5+7）
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 195, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  14.459 s
EXIT=0
```

194 基线逐类零回归（初验 194 的全部 26 类数值原样保持）；新增 1 项（I-1 负向测试 `compositeDeltaExpressionFailsClosedNotSilentlyWrongValue`，内含 3 种复合形态循环断言）；3 Skipped 仍为 0041 env 门控 1 + 0049 env 门控 2（未改这些写口）。真 Redis / live 冒烟本轮未重跑（I-1 修复仅收紧 DELTA 公式形态判定，纯 DELTA 用例仍绿、同构写与缓存未动；初验 live 全序列证据仍成立）。

### R3.2 修订轮范围核对

```
=== git show --name-only --format="" 01236fd ===
calc/src/main/java/net/otlab/calc/executor/DualTriggerExecutor.java
calc/src/test/java/net/otlab/calc/executor/OperatorIntegrationContractTest.java
=== out-of-scope count ===
0
=== 工作区并行改动（未触碰） ===
M frontend/src/api/display/display-api.ts + ?? frontend/src/views/display/device/（并行前端道在途，本修订轮未 add/未提交/未触碰）
```

## R4. 一句话结论

**修订通过**——I-1（DELTA 复合表达式 `DELTA(x)/3600` 静默错值：红证 `expected: <false> but was: <true>`、`failed=0` 写出原始差分；修为 `matches()` 严格纯 DELTA(key) 全串匹配 + fail-closed，3 种复合形态均隔离失败 sample=null，纯 DELTA 仍绿），M-1/M-2 按评审与协调者裁决登记 deferred（满量程需点位档案/CONV-03 契约升级、INTEGRAL 缺口补标契约待定），M-3 报告 BASE 订正为 `e49fea8`；全量 195 测试 0F0E3Skip BUILD SUCCESS（194 基线逐类零回归）、提交 `01236fd` 2 文件全落 `calc/**` 越界 0；TASK 保持「进行中」待 scoped 复评（只看 I-1 修复 diff 与原 Important）。

