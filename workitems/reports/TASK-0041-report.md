# TASK-0041 报告：双触发执行器（周期调度 + 到达驱动 + 节流幂等）

- 任务：TASK-0041（标准任务 4 人天；计算引擎道，只改 `calc/**`）
- 执行者角色：TASK-0041 执行者（no-subagents；brief 驱动）
- BASE：`3cdde58`（开工时主仓 main HEAD）
- 交付提交：`9a6cda6`（`TASK-0041: 双触发执行器（周期调度+到达驱动+节流幂等+汇总算子+ResultSink）`；23 文件全部 `calc/**`，越界 0）
- 执行日期：2026-11-24
- 验收结论：3/3 验收通过（mvn 134 测试 0F0E1Skip〔1 为 env 门控测试，真 Redis 另测 PASS〕；live 冒烟全序列通过；范围核对全落 calc/**）
- 环境事实：Redis 宿主 phpStudy 3.0.504@6379（真连接实测，Redisson/RESP2 可连）；8095 归 calc（启动前探测、用毕即停，终态 FREE）；TDengine 本任务不写；backend 道 TASK-0083 在途（全程未碰 backend/、未跑 backend mvn）；WSL docker 容器未碰。

---

## 一、改动文件清单（验收 3：范围核对）

`git diff --name-only 3cdde58..HEAD` 共 **23 项，全部落 `calc/**`**；`grep -v '^calc/'` 输出为空（`ALL CHANGES IN calc/`）。工作区 `git status --short calc/` 为空（已全部提交）。backend/** 与 `.tmp-0083/` 等为并行 TASK-0083 在途改动，本任务未 add、未提交、未触碰。

**新增（17 项）——`calc/src/main/java/net/otlab/calc/executor/`（13 项）：**

| 文件 | 职责 |
| --- | --- |
| `CalcPeriod.java` | 周期换算：同 backend 0081 `mapCalcPeriod` 口径（calc 侧独立实现，不依赖 backend）+ MONTH/YEAR；周期边界（period_start/period_end） |
| `Trigger.java` | 两通道统一入参（PERIODIC / ARRIVAL + 窗口幂等键） |
| `ValueSample.java` / `ValueStore.java` | 统一值样本（value/quality/ts，CONV-02）+ 进程内假实现值存储（真实源演进位） |
| `CalcResult.java` / `ResultSink.java` | 结果形态锁定（fullKey+来源标记 1/2/3+样本，失败无假值）+ 落地边界（内存假实现+日志，不写业务表） |
| `WindowIdempotencyGuard.java` | 同秒合并节流 + 幂等：有界 FIFO TTL 登记（多实例 Redis 化演进位） |
| `RollupOperator.java` | 汇总四法 SUM/AVG/MAX/MIN + quality=7 打标 + 开关关闭不纳入 + 动态子级 |
| `ArithmeticEvaluator.java` | 最小确定性公式求值（四则/括号/比较/逻辑/三元；DELTA/INTEGRAL 归 0042，越界=单节点隔离失败） |
| `DualTriggerExecutor.java` | 两通道统一引擎：拓扑序执行、失败隔离（quality=6、不发假 0、不阻塞他支）、汇总/公式/直绑分派、ResultSink 交付 |
| `PeriodicScheduler.java` | 自实现单线程调度（javadoc 载明取向：静态 @Scheduled 无法表达快照动态周期），按 bucket 变更整点触发、结果边界对齐 |
| `RedisArrivalSubscriber.java` | 真订阅 `iot:newvalue:{tenant}`（0029 发射侧通道），RESP2 零依赖客户端，断线指数退避重连、fail-soft |
| `ExecutorRuntime.java` | 装配/生命周期/观测（executor-enabled 总开关，默认关闭保持 0040 行为） |

**新增（4 项）——测试：**

| 文件 | 覆盖 |
| --- | --- |
| `executor/CalcPeriodContractTest.java`（5） | 周期换算同 0081 口径、MONTH/YEAR、固定/月/年边界、执行器右边界对齐 |
| `executor/DualTriggerExecutorContractTest.java`（10） | 拓扑顺序、到达影响面收敛、同秒节流幂等、有界 TTL 守卫、单节点失败隔离、汇总四法、打码 7、开关关闭、全缺 quality=7 |
| `executor/PeriodicSchedulerContractTest.java`（2） | 整点触发/边界对齐/到达项不参与/同窗幂等、图未装载不触发 |
| `executor/RedisArrivalSubscriberLiveTest.java`（1，env 门控） | 真 Redis SUBSCRIBE→PUBLISH 五键→回调逐字段一致 |

**修改（6 项）：**`CalcEngineApplication.java`（执行器 Bean + 启动装配）、`config/CalcProperties.java`（6 项新 env）、`web/CalcStatusController.java`（status 增 executor 观测视图）、`application.yml`（新 env 映射）、`test/config/CalcPropertiesTest.java`（默认值断言）、`README.md`（executor 目录/配置/语义/边界段）。

---

## 二、验收命令原始输出

### 验收 1：`cd calc && mvn test` → BUILD SUCCESS，0 失败 0 错误

完整命令：`cd /d/workspace/WisFactory/calc && mvn test`。终验（提交后 HEAD `9a6cda6` 对应源码、配置一致）Results 段与 BUILD 行原始输出（CR 行尾已去）：

```
[INFO] Running net.otlab.calc.ack.HttpAckSenderTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.557 s -- in net.otlab.calc.ack.HttpAckSenderTest
[INFO] Running net.otlab.calc.CalcEngineApplicationTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 5.103 s -- in net.otlab.calc.CalcEngineApplicationTest
[INFO] Running net.otlab.calc.config.CalcPropertiesTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.052 s -- in net.otlab.calc.config.CalcPropertiesTest
[INFO] Running net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.030 s -- in net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Running net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.078 s -- in net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Running net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Running net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[WARNING] Tests run: 1, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0.001 s -- in net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[INFO] Running net.otlab.calc.graph.CycleDetectorTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.015 s -- in net.otlab.calc.graph.CycleDetectorTest
[INFO] Running net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.039 s -- in net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Running net.otlab.calc.graph.GraphBuilderTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.056 s -- in net.otlab.calc.graph.GraphBuilderTest
[INFO] Running net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.065 s -- in net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Running net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.349 s -- in net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Running net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.024 s -- in net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Running net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.075 s -- in net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 134, Failures: 0, Errors: 0, Skipped: 1
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  9.062 s
[INFO] Finished at: 2026-11-24T01:40:xx+08:00
[INFO] ------------------------------------------------------------------------
EXIT=0
```

**0040 基线零回归**：115 基线（8+1+6+7+17+23+11+13+9+20）逐类全绿，新增 19 项（calc +1、executor 5+10+2+1=18，其中 1 项 env 门控 Skip）。134 = 115 + 19（含 1 Skip）。

**env 门控真 Redis 集成单跑**（`CALC_REDIS_TEST_ADDR=127.0.0.1:6379 mvn -q -Dtest=RedisArrivalSubscriberLiveTest test`）：EXIT=0，surefire `tests=1 fail=0 err=0`——真 SUBSCRIBE `iot:newvalue:t41live…` → 真 PUBLISH 五键 payload → 回调收到 `P41_LIVE / 42.5 / 0 / ts` 逐字段一致（无门控时 mvn test 自动 Skip，故 Results 段记 Skipped:1）。

### 验收 2：live 冒烟全序列（每步原始输出）

**步骤 0：启动前端口探测**

```
=== ports before ===
18041/8095 FREE
```

**步骤 1：假 backend 起（18041；真快照信封 + SHA-256 可重算哈希）+ calc 起（8095）**

```
=== fake backend ===
HASH=98d7335de52b158e6405736b0a83798111e8d3c66af22c5e2008b2ec9e40534e
（raw data.snapshot 逐字节重算 == 信封 snapshotHash，一致）

$ cd calc && CALC_BACKEND_BASE_URL=http://127.0.0.1:18041 CALC_INTERNAL_TOKEN=tok-t41 \
  CALC_TENANT=default CALC_EXECUTOR_ENABLED=true CALC_REDIS_ADDR=127.0.0.1:6379 \
  CALC_SCHEDULER_TICK_SECONDS=1 CALC_SNAPSHOT_RECONCILE_SECONDS=3600 java -jar target/calc-engine.jar
INFO ... CalcEngineApplication : Started CalcEngineApplication in 4.456 seconds
INFO ... PeriodicScheduler : [CALC-SCHED] 周期调度启动 tick=1s（整点触发 + 窗边界对齐）
INFO ... RedisArrivalSubscriber : [CALC-ARRIVAL] 新值事件订阅启动 channel=iot:newvalue:default addr=127.0.0.1:6379 db=0
INFO ... CalcEngineApplication : [CALC] 双触发执行器已启用（周期调度 + Redis 到达订阅 iot:newvalue:default）
INFO ... RedisArrivalSubscriber : [CALC-ARRIVAL] SUBSCRIBE 已发送 channel=iot:newvalue:default
INFO ... SnapshotReconciler : [CALC-RECONCILE] 依赖图已切换 version=41 hash=98d7335d… nodes=10 edges=7 topo=10
=== port 8095 ===
TCP 0.0.0.0:8095 LISTENING 92084
```

假 backend 快照图（10 节点 / 7 边）：`P1 → T41.A.V(P1*2) → T41.B.V(A+1)` 到达链；`T41.ROLLUP(SUM, calcPeriod=1)` 汇总 `T41.01/02/03.ROLLUP` 三子级；`P_MISSING → T41.BAD.V(P_MISSING+1) → T41.TAIL.V(BAD+1)` 失败链。

**步骤 2：到达驱动触发 + 拓扑顺序**

```
$ python smoke_helper.py pub default P1 5 1795211995000
PUBLISH :2
INFO ... DualTriggerExecutor : [CALC-EXEC] 执行完成 trigger=ARRIVAL nodes=3 results=2 failed=0 topo=[P1, T41.A.V, T41.B.V]
（status.executor.lastVisitOrder = ["P1","T41.A.V","T41.B.V"]——被依赖者在前，拓扑序正确）
```

**步骤 3：同秒合并节流（10 条新值 → 只触发一次）**

```
$ python smoke_helper.py pubburst default P2 1795212000000
PUBLISHED 10 events same second
INFO ... DualTriggerExecutor : [CALC-EXEC] 节流/幂等命中（同窗只算一次，保留最后正确结果）window=P2@1795212000 trigger=ARRIVAL
  （×9；throttle log lines: 9）
status.executor: arrivals=11 throttled=10 deliveries=2（10 条同秒 → deliveries 仅 +1）
```

**步骤 4：周期调度按短周期（1s）实跑一轮 + 结果时间戳对齐边界**

```
INFO ... PeriodicScheduler : [CALC-SCHED] 周期触发 key=T41.ROLLUP period=1000ms boundary=1795455375000 bucket=1795455374
INFO ... PeriodicScheduler : [CALC-SCHED] 周期触发 key=T41.ROLLUP period=1000ms boundary=1795455376000 bucket=1795455375
INFO ... DualTriggerExecutor : [CALC-EXEC] 执行完成 trigger=PERIODIC nodes=10 results=1 failed=0 topo=[T41.01.ROLLUP, T41.02.ROLLUP, T41.03.ROLLUP, P1, P_MISSING, T41.ROLLUP, T41.A.V, T41.BAD.V, T41.B.V, T41.TAIL.V]
（周期项每 bucket 触发一次；到达项 T41.A.V/B.V/BAD.V/TAIL.V 不产出——results=1 仅 ROLLUP；
 结果 ts=1795455487000=周期窗右边界）
```

**步骤 5：注入单节点失败观察隔离（P_MISSING 无值 → BAD.V → TAIL.V）**

```
$ python smoke_helper.py pub default P_MISSING 7 1795212001000
PUBLISH :2
ERROR ... DualTriggerExecutor : [CALC-EXEC] 单节点失败（隔离，不发假 0）key=T41.BAD.V reason=无值输入: P_MISSING
ERROR ... DualTriggerExecutor : [CALC-EXEC] 单节点失败（隔离，不发假 0）key=T41.TAIL.V reason=上游节点失败（依赖链传播）
（BAD/TAIL sample=null，不写假 0；A.V/B.V 等其他分支不受影响继续产出）
```

**步骤 6：汇总结果断言（SUM 四法全部由契约测试锁定，live 验 SUM + 打码 7）**

```
# 子级 2/4/8 注入后读 status.executor.lastResults：
[{"fullKey":"T41.ROLLUP","sourceType":3,"value":14.0,"quality":0,"ts":1795455487000}]
  → SUM=2+4+8=14，来源标记 3（ROLLUP），quality=0，ts=周期边界

# 将 T41.03.ROLLUP 以 quality=7 重发后：
[{"fullKey":"T41.ROLLUP","sourceType":3,"value":6.0,"quality":7,"ts":1795455496000}]
  → q7 子级贡献 8 不纳入（2+4=6），保留打码 7（CONV-02 汇总不完整）
```

契约测试 `rollupFourMethods` 机械锁定 SUM=14 / AVG=14/3 / MAX=8 / MIN=2；`rollupExcludesQuality7AndDisabledChildrenWithMark` 锁定 q7 打标与开关关闭不纳入；`rollupAllChildrenMissingOutputsQuality7NotZero` 锁定全缺 quality=7 不写 0。

**步骤 7：全停归零 + 冒烟数据清理**

```
=== STOP sequence ===
成功: 已终止 PID 为 92084 的进程。（calc）
成功: 已终止 PID 为 86944 的进程。（fake backend）
=== ports after ===
BOTH PORTS FREE
=== cleanup smoke redis keys ===
DEL T41.01.ROLLUP :0  / T41.02.ROLLUP :0 / T41.03.ROLLUP :0 / P1 :0 / P2 :0 / P_MISSING :0 / T41.BAD.V :0
（:0 = 无残留——calc 只把事件值写入进程内 ValueStore，不写 Redis 业务键；冒烟未污染宿主 Redis）
=== no java procs ===
NONE
```

临时假后端/辅助脚本目录 `D:\task41-smoke\` 测后已删除；未写 TDengine；未碰 backend/、未跑 backend mvn；WSL docker 容器未碰。

### 验收 3：范围核对（原始输出）

```
=== git diff --name-only 3cdde58..HEAD ===
calc/README.md
calc/src/main/java/net/otlab/calc/CalcEngineApplication.java
calc/src/main/java/net/otlab/calc/config/CalcProperties.java
calc/src/main/java/net/otlab/calc/executor/{13 项}
calc/src/main/java/net/otlab/calc/web/CalcStatusController.java
calc/src/main/resources/application.yml
calc/src/test/java/net/otlab/calc/config/CalcPropertiesTest.java
calc/src/test/java/net/otlab/calc/executor/{4 项}
=== count ===
23
=== out-of-scope check ===
ALL CHANGES IN calc/
=== calc working tree ===
（空）
```

---

## 三、未做项与原因

1. **DELTA / INTEGRAL 算子（0042）**：brief 决策 6 明确不在本批范围。`ArithmeticEvaluator` 遇函数调用抛 `UnsupportedOperationException`，由执行器按单节点失败隔离（不打假 0），接口位留待 0042 接入。
2. **指标聚合 / st_metric_value 写（0045）**：brief 决策 5 明示 ResultSink 边界——执行器不写 TDengine/MySQL 业务表；Metric 节点本执行器不计算（0045 边界）。
3. **数据项结果表 / Redis 数据项缓存真实 Sink**：本批只交付 `ResultSink` 接口 + 内存假实现 + 日志 + 契约测试锁定结果形态；真实 Sink 归计算引擎后续，演进位登记。
4. **多实例 Redis 化幂等登记**：brief 决策 2 明示本期为进程内有界 TTL 假实现，多实例 Redis 化为演进位（接口不变、实现可替换）。
5. **新值事件消费对接的 payload 再定版**：0029 发射侧五键 `{tenant,pointCode,ts,value,quality}` 已定版且 live 实测一致；本批按该形态消费，无契约冻结变更。
6. **config.changed 真订阅（TD-0057）/ NotifySender 真联调（TD-0056）**：仍为 0040 既有 NoOp 占位 + 拉取对账兜底，本批不触碰（到达驱动走 Redis 新值事件独立通道，与 config.changed 正交）。
7. **状态判定（0049）、iot_calc_log/state 写（0063）**：brief 决策 6 明示归后续道；失败原因经日志/`CalcResult.error()` 暴露，落表归 0063。
8. **契约冻结变更**：三份冻结契约未改；CONV-02 质量码（0 正常/6 计算失败/7 汇总不完整/9 跨周期）、统一模型 ts/value/quality 三键、0029 新值事件通道逐字对齐，未发现设计级缺陷。
9. **TASK 状态**：按硬约束未将 TASK-0041 置「已完成」，保持「进行中」待评审；未改 workitems 库（报告除外）。

---

## 四、偏离计划之处

1. **周期触发判据从「nowMillis % period == 0」改为「bucket 变更」**（live 驱动红转绿）：初版按整点瞬间精确命中，live 中 tick=1s 的墙钟时刻几乎不可能恰在 epoch 秒整点，13 次 tick 零触发（真红）。修正为「当前周期窗 bucket 与上次的 bucket 不同即触发一次」，tick 粒度保证新窗开始后的下个 tick 命中；结果时间戳仍由执行器按周期边界对齐（契约测试时间戳断言不变）。为什么现在对了：整点触发的业务语义是「每个周期窗算一次」，bucket 比较精确表达该语义且对墙钟抖动鲁棒，同窗幂等由守卫兜底。
2. **调度器与执行器共用守卫导致的重复节流**（live 驱动红转绿）：初版调度器先 `guard.tryAcquire(windowKey)`、执行器再用同一守卫判同键 → 每次周期触发都被执行器判 DUPLICATE（live 日志 5 次全「节流命中」、零结果）。修正为调度器只用 bucket 去重、幂等统一交给执行器守卫。为什么现在对了：幂等责任单点收敛（执行器），调度器只管触发时机，职责不重叠。
3. **丢失子级不记失败传播**（TDD 红转绿）：初版缺值点位/子级记入失败集导致汇总项被「上游失败」传播而整节点失败，与 quality=7 语义冲突（契约测试红 2 项）。修正为：缺数点位不记失败传播（公式求值/汇总按无值口径自然处理），汇总边排除在硬依赖传播外（quality=7 而非失败）。为什么现在对了：CONV-02「缺失打码不写假 0」要求汇总不完整是结果打标而非计算失败，职责分离后失败隔离与汇总打码各自成立。
4. **Redis RESP2 解析从 BufferedReader 混读改为字节级解析器**（live 驱动红转绿）：初版字符/行混读在 RESP 批量串尾 CRLF 与整数回复（SUBSCRIBE 确认 `:1`）上连续两次「消息头损坏」（真红两轮）。修正为字节级 `RespReader`（正确区分 `$`/`:`/`+`/`-` 与嵌套数组）。为什么现在对了：RESP 是有帧长度的二进制协议，字节级按帧读取是唯一可靠形态；真 Redis SUBSCRIBE 全链测试从 FAIL 转 PASS 为证。
5. **新增 `calc/src/main/java/net/otlab/calc/executor/` 包与 6 项 env**：brief 写「只改 calc/**」，新增包/配置全在范围内；executor 默认关闭（`CALC_EXECUTOR_ENABLED=false`）确保 0040 既有 115 测试与启动行为零变化（新增应用装配测试仍 PASS）。
6. **fake backend 真快照哈希构造**：初版信封 `data.snapshot` 经 Python `json.dumps` 再序列化，RawJsonExtractor 提取的逐字节原文与预置 hash 不一致（fail-closed 拒收，真红）；修正为把紧凑 JSON 原文逐字嵌入信封字符串（重算一致）后 graph 正常切换。属冒烟载体构造修正，未改 calc 代码/契约。
7. **假后端端点用 18041 而非 backend 8098**：brief 允许 live 冒烟用 Python RESP 客户端连宿主 Redis/自造触发源；并行 0083 在途改 backend/** 且 backend 道不跑 mvn，故用独立假端点提供受控快照（仍经 calc 真 HTTP 客户端 + 真哈希校验 + 真 ACK 回执消费，非 mock calc 侧），不引入 backend 依赖。

---

## 五、遗留问题

1. **多实例幂等 Redis 化（演进位）**：`WindowIdempotencyGuard` 为进程内有界 TTL 实现；多实例部署需 Redis `SET NX EX` 跨实例登记，接口不变。单实例语义已由契约测试 + live（同秒 10 条→1 次）锁定。
2. **真实 ResultSink 替换（演进位）**：内存假实现仅验证结果形态与交付边界；写 st_item_value / Redis 数据项缓存的真实 Sink 归后续道，`ResultSink` 为实现替换缝，调用点不扩散。
3. **公式求值器口径（0042/FEAT-0012 衔接）**：`ArithmeticEvaluator` 是最小确定性求值（无 Aviator），DELTA/INTEGRAL 调用按单节点失败隔离（质量码 6）；真 Aviator 与内置算子由 0042/FEAT-0012 接入时替换该组件，失败隔离语义不变。
4. **周期调度 tick 粒度**：默认 1s，最小周期项配置若 < 1s 则无法在其边界内触发（本批周期项以秒为单位，详见 §1.6 `calc_period` 整秒口径）；亚秒周期需求未在契约范围。
5. **执行器 6 项新 env 未纳入 prod runbook**：默认关闭不影响既有部署；启用时需按 README 配置表注入（dev 冒烟已逐项实测），runbook 固化归部署道。
6. **TASK 状态**：保持「进行中」，由协调者安排评审后裁决（硬约束：未置「已完成」）。

---

## 六、一句话结论

**通过**——验收 1（`mvn test` 134 测试 0F0E1Skip〔1 为 env 门控测试，单跑真 Redis PASS〕，0040 基线 115 逐类零回归；「为什么现在对了」红转绿 4 处留痕），验收 2（live 冒烟全序列：拓扑链 `topo=[P1,A.V,B.V]`、同秒 10 条→1 次计算 9 次节流、周期 1s 每窗触发且结果 ts=窗右边界、失败链 BAD/TAIL 隔离不发假 0、汇总 SUM=14/q7 重发=6.0 quality=7、真 Redis SUBSCRIBE 五键逐字段一致、8095/18041 全停归零无残留），验收 3（提交 `9a6cda6` 23 文件全落 calc/**、越界 0、工作区干净）三条全部以原始输出为证通过；遗留均为演进位/后续道边界，无阻断缺陷，TASK 保持「进行中」待评审。

---

# 修订轮（退回轮 §12/§13：F-01 + F-02 + F-03，顺手 F-04）

- 评审输入：`TASK-0041-review.md`（3 Important〔F-01 比较反转 / F-02 ValueStore 并发 / F-03 同窗末值盲区〕+ 5 Minor，0 Critical）
- 协调者裁决：F-01/F-02/F-03 必修，F-04 死字段清理顺手做；F-05/F-06/F-07 登记 deferred（F-06 补 javadoc 边界说明）
- 修订轮 BASE：`9a6cda6`（被评审提交；区间内并行 `4d4012e` 为 TASK-0083 backend/**，未触碰）
- 交付提交：`73b8587`（`TASK-0041 修订轮: F-01 比较方向/F-02 ValueStore 并发/F-03 同窗保留末值+F-04 死字段`；12 文件全部 `calc/**`，`git show --name-only --format="" HEAD | grep -v '^calc/' | wc -l` = 0）
- 修订轮测试：契约测试 **134 → 140**（新增 F-01 4 项 + F-02 2 项；F-03 强化既有项），既有 134 逻辑零回归；env 门控真 Redis 单跑仍 PASS（tests=1 fail=0 err=0 skip=0）。

## R1. 逐条修复与红绿证据

### R1.1 F-01（Important）`>=`/`<=` 操作数反转——红转绿

- **红**：新增 `ArithmeticEvaluatorContractTest`（4 项，覆盖 `>= <= > <` 四法 × 正反两向 + 三元条件）。旧实现 `mvn -q -Dtest=ArithmeticEvaluatorContractTest test` → `Tests run: 4, Failures: 3`，实测原始输出：

```
ArithmeticEvaluatorContractTest.greaterOrEqualComparesLeftAgainstRight:27 10>=3 真 ==> expected: <1.0> but was: <0.0>
ArithmeticEvaluatorContractTest.lessOrEqualComparesLeftAgainstRight:36 10<=3 假 ==> expected: <0.0> but was: <1.0>
ArithmeticEvaluatorContractTest.ternaryUsesCorrectComparisonDirection:55 3>=10 假→0 ==> expected: <0.0> but was: <1.0>
```

- **修**：`ArithmeticEvaluator.java:parseComparison` 仿 `>`/`<` 分支——先把右值存入局部变量 `double r = parseAdditive()`，再 `v >= r` / `v <= r`（旧代码 `parseAdditive() >= v` 算的是 right >= left）。
- **绿**：`mvn -q -Dtest=ArithmeticEvaluatorContractTest test` → EXIT=0，surefire `tests=4 fail=0 err=0`；`> <` 两法 4 项亦全绿（未回归）。
- **为什么现在对了**：比较表达式左值 `v` 在进入 while 前由 `parseAdditive()` 取得，运算符后必须再取右值、按「左 OP 右」求值；局部 `r` 固定了求值顺序，消除旧实现把右值当左操作数导致的静默反向布尔（含三元条件分支）。该 bug 从 `FormulaExpressionValidator` 白名单合法配置直达生产结果，修复后 4 项断言（含 `A>=B`/`B>=A`/`A<=B`/`B<=A` 标识符左右向）全部锁定方向正确。

### R1.2 F-02（Important）ValueStore HashMap 并发读写——红转绿

- **红**：新增 `ValueStoreConcurrencyContractTest`（2 项）：① 结构性契约守卫（反射断言 `get` 为 synchronized 或底层 map 为 `ConcurrentMap`——覆盖协调者裁决的两种合法修法）；② 真实并发压力（4 写 × 4 读 × 500 次，断言无异常、最终值全可见）。旧实现 `mvn -q -Dtest=ValueStoreConcurrencyContractTest test` → `Tests run: 2, Failures: 1`：

```
ValueStoreConcurrencyContractTest.threadSafetyContractIsBackedByImplementation:44
ValueStore 须线程安全：get 同步 或 底层 ConcurrentMap（旧 HashMap 无同步 get 必红） ==> expected: <true> but was: <false>
```

- **修**：`ValueStore.java` 底层 `HashMap` → `ConcurrentHashMap`；`get/put/put(value,ts)/snapshot/clear` 全部为线程安全操作（去掉误导性的 `synchronized` 修饰），javadoc 改为与实现一致并点明「周期调度线程与 Redis 到达订阅线程并发读写」场景（F-02 评审点）。
- **绿**：`mvn -q -Dtest=ValueStoreConcurrencyContractTest test` → EXIT=0，surefire `tests=2 fail=0 err=0`；并发压力项无异常、2000 个写入值最终全可见。
- **为什么现在对了**：`ConcurrentHashMap` 的 get/put 在 JMM 下对并发读写提供 happen-before 与原子性，消除了旧实现「put synchronized + get 无同步」对同一 HashMap 的无互斥竞争（扩容期间可能链表成环/CPU 打满）；`ValueSample` 为不可变 record，`snapshot()` 返回 `HashMap` 副本的不可变视图，无二次逃逸。结构性守卫把线程安全契约固化为可执行断言，避免 javadoc 与实现再次漂移。

### R1.3 F-03（Important）同窗保留最后正确结果——红转绿（含交付值实证）

- **红**：强化 `DualTriggerExecutorContractTest.throttleSameWindowOnlyComputesOnce`：模拟真实到达链——同秒 10 条值 `[1,2,3,4,5,6,7,8,3,9]`，每条先写 store 再触发；断言 ① 后续到达重算替换（返回非空）② ResultSink 同窗只 1 条交付 ③ **交付 A.V 值 = 10.0（末值 9+1），不得是首值 2.0**。旧实现 `mvn -q -Dtest='DualTriggerExecutorContractTest#throttleSameWindowOnlyComputesOnce' test` → `Tests run: 1, Failures: 1`：

```
throttleSameWindowOnlyComputesOnce:120 同窗后续到达重算替换（空返回=旧实现未保留末值） ==> expected: <0> but was: <9>
```

（旧实现同窗后续 9 次直接返回空——首值计算交付后再不重算，正是评审 F-03 实证。）

- **修**（三处协同，口径不变仅实现纠偏）：
  1. `WindowIdempotencyGuard.Decision.DUPLICATE` → `REPLACE`：同窗重复到达不再「丢弃」，而是重算并替换（仍受 maxEntries/TTL 有界约束）；
  2. `ResultSink` 新增 `accept(String windowKey, Trigger, List<CalcResult>)` 默认方法，`InMemoryResultSink` 覆写为按 windowKey 覆盖交付记录（`Delivery` 增 windowKey 字段）——对外同窗始终只有 1 条最终交付；演进位真实 Sink 按同键覆盖写；
  3. `DualTriggerExecutor.execute` 去早退：FIRST/REPLACE 都用最新 store 重算，按 windowKey 交付；`ExecutorRuntime` 的 `throttled` 计数语义由「丢弃」改为「同窗合并重算次数」（值可见但不再代表丢事件）。
- **绿**：`mvn -q -Dtest=DualTriggerExecutorContractTest test` → EXIT=0，surefire `tests=10 fail=0 err=0`；`guardIsBoundedAndExpires` 同步更新为 REPLACE 断言（有界 FIFO/TTL 语义不变）。
- **为什么现在对了**：「同窗多次触发只产出一次结果」中的「产出一次」落在 ResultSink 的窗键唯一交付上——中间值的重算只覆盖不追加；「保留最后正确结果」落在每次到达都重算并覆盖上——最终交付的 sample 必反映末值。单元测试锁死末值 10.0（9+1）；live 另做端到端实证（见 R2.2）。
  - 修复过程中另抓到一处自身回归：初版 `InMemoryResultSink.accept(String,...)` 用无键构造器创建 `Delivery`（windowKey=null）导致覆盖永不命中（`Tests run: 10, Failures: 1 … expected: <1> but was: <10>`）；改用 `new Delivery(windowKey, trigger, results)` 后绿——`Delivery` 的 windowKey 是覆盖匹配的锚点。

### R1.4 Minor 顺手/F-06 边界

- **F-04（死字段）**：`PeriodicScheduler` 删除赋值后从未使用的 `guard` 字段与构造器参数（3 处调用点同步更新）；类 javadoc 改为真实机制（`lastFiredBucket` 触发去重 + 执行器守卫按 windowKey 幂等），消除文档漂移。`PeriodicSchedulerContractTest` 仍 2/2 绿。
- **F-06（deferred + javadoc）**：`CalcPeriod.fromCode`（含 MONTH/YEAR 日历窗）生产路径无调用者——快照契约 `calcPeriod` 为整数秒（0=到达驱动，`config-snapshot-schema.md:173`），无法表达自然月/年。按裁决**不本轮实现**，在 `CalcPeriod.fromCode` 与 `PeriodicScheduler` 类 javadoc 明示边界：HOUR/DAY 以秒表示经快照可达，MONTH/YEAR 仅声明、生产未接线，登记 deferred。
- **F-05/F-07（deferred）**：`InMemoryResultSink.deliveries` 无界增长、status 中 `tickSeconds="see-config"` 占位值与 `lastVisitOrder/lastResults` 暴露面——按裁决仅登记 deferred，不本轮改（F-07 暴露面与既有内网无鉴权 status 同源，非新增越权；占位值修正可与 FEAT-0015 状态页一道处理）。

## R2. 修订轮验收原始输出

### R2.1 全量 `cd calc && mvn test`（HEAD `73b8587`）

```
[INFO] Running net.otlab.calc.ack.HttpAckSenderTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.775 s -- in net.otlab.calc.ack.HttpAckSenderTest
[INFO] Running net.otlab.calc.CalcEngineApplicationTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.477 s -- in net.otlab.calc.CalcEngineApplicationTest
[INFO] Running net.otlab.calc.config.CalcPropertiesTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.016 s -- in net.otlab.calc.config.CalcPropertiesTest
[INFO] Running net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.022 s -- in net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Running net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.017 s -- in net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Running net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.050 s -- in net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Running net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.008 s -- in net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Running net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[WARNING] Tests run: 1, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0 s -- in net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[INFO] Running net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Running net.otlab.calc.graph.CycleDetectorTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.calc.graph.CycleDetectorTest
[INFO] Running net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.021 s -- in net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Running net.otlab.calc.graph.GraphBuilderTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.029 s -- in net.otlab.calc.graph.GraphBuilderTest
[INFO] Running net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.071 s -- in net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Running net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.196 s -- in net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Running net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.011 s -- in net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Running net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.036 s -- in net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 140, Failures: 0, Errors: 0, Skipped: 1
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  15.021 s
[INFO] ------------------------------------------------------------------------
EXIT=0
```

134 基线逐类零回归（含 0040 的 115）；新增 6 项（F-01 4 + F-02 2）+ F-03 既有项强化。env 门控真 Redis 单跑另行确认：`CALC_REDIS_TEST_ADDR=127.0.0.1:6379 mvn -q -Dtest=RedisArrivalSubscriberLiveTest test` → EXIT=0，surefire `tests=1 fail=0 err=0 skip=0`。

### R2.2 F-03 交付值实证（live 端到端）

```
# 假后端（18042）单链 P1 -> A.V=P1*2；calc 起（8095），executor-enabled=true
INFO ... Started CalcEngineApplication
INFO ... [CALC-RECONCILE] 依赖图已切换 version=41 hash=06758bd1… nodes=2 edges=1 topo=2

# 同秒 PUBLISH 9 条新值（值 1..9，末值 9），全部经真 Redis pub/sub 到达订阅器
published 9 events, last value=9

# 读 /calc/v1/status.executor
deliveries = 1                 （同窗只产出一次交付——幂等成立）
lastResults = [{"fullKey":"A.V","sourceType":2,"value":18.0,"quality":0,"ts":1795211995000}]
arrivals = 9
→ A.V=18.0 = 末值 9 × 2（非首值 2.0 × 2=4.0）——「保留最后正确结果」live 实证
```

单元测试另锁 `末值 P1=9.0 → A.V=10.0`（公式 `P1+1`）为最终交付值。两处证据共同闭合 F-03 验收盲区。

### R2.3 修订轮范围核对

```
=== git show --name-only --format="" 73b8587 ===
calc/src/main/java/net/otlab/calc/executor/{ArithmeticEvaluator,CalcPeriod,DualTriggerExecutor,ExecutorRuntime,PeriodicScheduler,ResultSink,ValueStore,WindowIdempotencyGuard}.java
calc/src/test/java/net/otlab/calc/executor/{ArithmeticEvaluatorContractTest,ValueStoreConcurrencyContractTest,DualTriggerExecutorContractTest,PeriodicSchedulerContractTest}.java
=== out-of-scope count ===
0
=== ports after ===
FREE（8095/18042 全停；临时 D:\t41r-smoke\ 已删除；宿主 Redis 键零残留）
```

## R3. 修订轮一句话结论

**修订通过**——F-01（`>=`/`<=` 左右方向红转绿，4 项覆盖四法双向+三元）、F-02（ValueStore 改 ConcurrentHashMap + 结构守卫/并发压力双测红转绿）、F-03（同窗 REPLACE + ResultSink 窗键覆盖，单测末值 10.0 与 live 末值 18.0 双实证，幂等=1 交付保持）、F-04（死字段清理+文档对齐）全部以红绿证据落地；F-05/F-06/F-07 按裁决登记 deferred（F-06 javadoc 已注明 MONTH/YEAR 生产未接线边界）；全量 140 测试 0F0E1Skip BUILD SUCCESS（134 基线零回归）、真 Redis PASS、提交 `73b8587` 12 文件全落 calc/** 越界 0。待 scoped 复评（只看修复 diff 与原 3 条 Important/5 条 Minor）。
