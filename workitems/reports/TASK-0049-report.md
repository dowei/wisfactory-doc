# 实施报告：TASK-0049（状态判定执行——优先级求值+Redis+st_item_value 留痕）

- 执行者：TASK-0049 执行者（no-subagents；brief 驱动；计算引擎道——只改 `calc/**`）
- BASE：`7628007`（开工时主仓 main HEAD）
- 本任务提交：`fded29b`（`TASK-0049: 状态判定执行（优先级求值+Redis+st_item_value留痕+LWT）`；23 文件全部 `calc/**`，越界 0）
- 执行日期：2026-11-24
- 环境：Redis 宿主 phpStudy 3.0.504@6379（RESP2 实测可连、真 SET/SUBSCRIBE 实测 PASS）；TDengine 192.168.31.145:6041（TCP 通，但 REST 查询本会话实测 21s 后 Connection reset——见「未做项」）；8095 归 calc（起前探测 FREE、用毕即停、终态 FREE）；scratch `.tmp-t0049/` 测后已删；backend 道未碰；WSL docker 容器未碰。
- 结论先行：**验收 3 条全部通过**——mvn test 159 测试 0F0E3Skip BUILD SUCCESS（0041 的 140 基线逐类零回归 + 新增 19）；live 冒烟全序列绿（运行→故障优先→UNKNOWN 不硬套、Redis 写+未变不重写、st_item_value INSERT 逐语句留痕、LWT 离线重判、全停归零）；范围核对 23 文件全落 calc/**。

---

## 1. 改动文件清单（23 文件，全部在 calc/**）

### 1.1 新增——状态判定包 `calc/src/main/java/net/otlab/calc/status/`（13）

| 文件 | 职责 |
| --- | --- |
| `StatusRule.java` / `StatusRuleSet.java` / `StatusDirectBinding.java` / `StatusDecision.java` | 规则/字典/直绑映射/判定结果的不可变承载（稳定序号字典单一来源、UNKNOWN 哨兵、来源枚举 DIRECT/RULE/LWT/UNKNOWN） |
| `StatusEvaluator.java` | 求值引擎（§1.5B）：直绑短路 → LWT 离线优先 → priority 升序首命中 → UNKNOWN+质量码 3；单规则失败隔离；复用 0041 ArithmeticEvaluator |
| `StatusEvaluatorRuntime.java` | 接入层：ResultSink 装饰器桥接 0041 执行器输出 + LWT 事件回调 → 求值 → Redis 写 + st_item_value 留痕；判定失败隔离、未变不重写、观测 view |
| `RedisStateWriter.java` / `Resp2StateWriter.java` | 当前状态写口接口 + 真 RESP2 实现：`iot:st:{tenant}:{instance}` JSON（statusCode/statusName/color/seq/quality/ts）、未变不重写、失败不覆盖旧值；接口附 NoOp/Recording 实现 |
| `StatusTraceSink.java` / `TdengineTraceWriter.java` | 留痕边界 + taosAdapter REST 真实现：STATUS 数据项写 st_item_value（value=稳定序号 quality=0/UNKNOWN -1+质量码）、逐语句独立下发、子表首写 `INSERT...USING...TAGS`（含 tenant_id）、未变不重写、失败 WARN 不阻塞 |
| `StatusRuleProvider.java` / `StatusRuleSetParser.java` | 规则源缝 + env 种子 JSON 解析（冻结快照不装载 iot_status_rule，演进位归发布链） |
| `RedisLwtEventSource.java` | LWT 离线事件订阅（ADR-0010）：真 SUBSCRIBE `iot:gwstatus:{tenant}`，RESP2 零依赖、断线退避重连、fail-soft，只订阅不写状态 |

### 1.2 新增——契约测试 `calc/src/test/java/net/otlab/calc/status/`（4，共 19 项）

| 文件 | 用例数 | 覆盖 |
| --- | --- | --- |
| `StatusEvaluatorContractTest.java` | 8 | priority 升序故障先于运行 / 直绑短路 / 枚举未命中落规则链 / UNKNOWN 不硬套 / LWT 离线优先 / 规则求值失败隔离+停用跳过 / 稳定序号单一来源 |
| `StatusRuntimeContractTest.java` | 6 | 状态变化写 Redis+trace / 未变不重写双写口 / 状态变化追加 / LWT 触发重判 / 判定失败不覆盖旧值 / 留痕失败不阻塞 |
| `TdengineTraceWriterContractTest.java` | 3 | INSERT 逐语句 USING TAGS value=seq quality=0 / 未变不重写 / 失败上抛由调用方 WARN |
| `StatusRedisLiveTest.java` | 2（env 门控 `CALC_STATUS_REDIS_TEST_ADDR`） | 真 RESP2 写状态 JSON + 未变不重写 / 真 SUBSCRIBE LWT→PUBLISH→回调 |

### 1.3 修改（6）

| 文件 | 说明 |
| --- | --- |
| `CalcEngineApplication.java` | statusRuntime Bean（默认关闭）+ 共享 ValueStore Bean + executorRuntime 接 sink 装饰器/共享 store + 启动 Runner 起状态运行时 |
| `config/CalcProperties.java` | 新增 8 项 status/tdengine env（默认全关/安全值——0040/0041 既有行为零变化） |
| `executor/ExecutorRuntime.java` | 新增 sink 装饰器/共享 store 构造重载，0041 原构造委托、行为不变（仅 final store 字段初始化迁移到构造器） |
| `web/CalcStatusController.java` | `/calc/v1/status` 增 `statusEvaluator` 观测面（null-safe） |
| `resources/application.yml` | 新增 env 映射 |
| `README.md` | status/ 目录、7 项 env、状态判定链路语义与规则源缝边界段 |

---

## 2. 验收命令原始输出

### 2.1 验收 1：`cd calc && mvn test` → BUILD SUCCESS、0 失败 0 错误（140 基线 + 新增 19）

终验（提交 `fded29b` 对应源码，HEAD=fded29b）Results 段逐类原始输出：

```
[INFO] Running net.otlab.calc.ack.HttpAckSenderTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.483 s -- in net.otlab.calc.ack.HttpAckSenderTest
[INFO] Running net.otlab.calc.CalcEngineApplicationTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.054 s -- in net.otlab.calc.CalcEngineApplicationTest
[INFO] Running net.otlab.calc.config.CalcPropertiesTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.016 s -- in net.otlab.calc.config.CalcPropertiesTest
[INFO] Running net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.calc.executor.ArithmeticEvaluatorContractTest
[INFO] Running net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.010 s -- in net.otlab.calc.executor.CalcPeriodContractTest
[INFO] Running net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.040 s -- in net.otlab.calc.executor.DualTriggerExecutorContractTest
[INFO] Running net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.005 s -- in net.otlab.calc.executor.PeriodicSchedulerContractTest
[INFO] Running net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[WARNING] Tests run: 1, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0 s -- in net.otlab.calc.executor.RedisArrivalSubscriberLiveTest
[INFO] Running net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.010 s -- in net.otlab.calc.executor.ValueStoreConcurrencyContractTest
[INFO] Running net.otlab.calc.graph.CycleDetectorTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.006 s -- in net.otlab.calc.graph.CycleDetectorTest
[INFO] Running net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.020 s -- in net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Running net.otlab.calc.graph.GraphBuilderTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.036 s -- in net.otlab.calc.graph.GraphBuilderTest
[INFO] Running net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.047 s -- in net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Running net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.161 s -- in net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Running net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.008 s -- in net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Running net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.031 s -- in net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Running net.otlab.calc.status.StatusEvaluatorContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.status.StatusEvaluatorContractTest
[INFO] Running net.otlab.calc.status.StatusRedisLiveTest
[WARNING] Tests run: 2, Failures: 0, Errors: 0, Skipped: 2, Time elapsed: 0.001 s -- in net.otlab.calc.status.StatusRedisLiveTest
[INFO] Running net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.011 s -- in net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Running net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 159, Failures: 0, Errors: 0, Skipped: 3
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.379 s
[INFO] Finished at: 2026-11-24T02:56:35+08:00
[INFO] ------------------------------------------------------------------------
EXIT=0
```

**0041 的 140 基线零回归**：140 = 8+1+7+4+5+10+2+1(skip)+2+7+17+23+11+13+9+20，逐类数值与 0041 修订轮报告一致；3 Skipped = 0041 env 门控 1 + 本任务 env 门控 2（真 Redis 另跑 PASS，见下）。新增 19（status 8+6+3+2）。

**env 门控真 Redis 单跑**（`CALC_STATUS_REDIS_TEST_ADDR=127.0.0.1:6379 mvn -q -Dtest=StatusRedisLiveTest test`）：EXIT=0，日志原始输出：

```
02:51:27.794 [main] INFO net.otlab.calc.status.RedisLwtEventSource -- [CALC-STATUS] LWT 事件订阅启动 channel=iot:gwstatus:t49live addr=127.0.0.1:6379
02:51:28.347 [main] INFO net.otlab.calc.status.Resp2StateWriter -- [CALC-STATUS] Redis 当前状态已写 key=iot:st:t49live:EQ1 value={"statusCode":"RUNNING","statusName":"RUNNING","color":"#00FF00","seq":1,"quality":0,"ts":1795211995000}
02:51:28.351 [main] INFO net.otlab.calc.status.Resp2StateWriter -- [CALC-STATUS] Redis 当前状态已写 key=iot:st:t49live:EQ1 value={"statusCode":"FAULT","statusName":"FAULT","color":"#FF0000","seq":3,"quality":0,"ts":1795211997000}
```

真 SET `iot:st:t49live:EQ1`（camelCase JSON + 稳定序号）、未变不重写（GET 前后一致）、真 SUBSCRIBE `iot:gwstatus:t49live` → PUBLISH → 回调逐项 PASS；测试 finally DEL 清理，宿主零残留。

### 2.2 验收 2：live 冒烟全序列（每步原始输出）

假 backend（Python HTTPServer，18049，真快照信封 SHA-256 可重算）+ calc 起（8095，status-enabled=true，规则种子 env 注入，TDengine REST 指向记录式假适配器 18049 捕获真实 INSERT SQL）。

**步骤 0：起前端口探测**

```
=== ports before ===
8095/18049 FREE
```

**步骤 1：calc 起 + 装配日志**

```
$ java -jar calc-engine.jar（CALC_STATUS_ENABLED=true / CALC_STATUS_INSTANCE_CODE=T49.EQ001 / CALC_STATUS_RULES_JSON={dict,rules,direct}）
INFO ... CalcEngineApplication : Started CalcEngineApplication in 1.569 seconds
INFO ... RedisArrivalSubscriber : [CALC-ARRIVAL] 新值事件订阅启动 channel=iot:newvalue:t49 addr=127.0.0.1:6379 db=0
INFO ... ExecutorRuntime : [CALC-RUNTIME] 双触发执行器已启动 tenant=t49 channel=iot:newvalue:t49
INFO ... RedisLwtEventSource : [CALC-STATUS] LWT 事件订阅启动 channel=iot:gwstatus:t49 addr=127.0.0.1:6379
INFO ... StatusEvaluatorRuntime : [CALC-STATUS] 状态判定已装配 tenant=t49 instance=T49.EQ001 redis=iot:st:t49:T49.EQ001 lwt=iot:gwstatus:t49
INFO ... SnapshotReconciler : [CALC-RECONCILE] 依赖图已切换 version=49 hash=1f6f88b4… nodes=2 edges=0 topo=2
=== port 8095 ===
TCP 0.0.0.0:8095 LISTENING 82152
```

规则种子：priority 10 = `T49.FAULT_FLAG == 1`→FAULT(seq3)、priority 20 = `T49.CURRENT > 0`→RUNNING(seq1)，字典 RUNNING=1/FAULT=3/OFFLINE=4。图含两个直绑点位输入节点 T49.CURRENT / T49.FAULT_FLAG（到达事件写共享 ValueStore → 求值读取）。

**步骤 2：运行判定 + Redis 写 + st_item_value 留痕**

```
$ python redis_helper.py pub iot:newvalue:t49 '{"pointCode":"T49.CURRENT","ts":1795211995000,"value":5,"quality":0}'
$ python redis_helper.py pub iot:newvalue:t49 '{"pointCode":"T49.FAULT_FLAG","ts":1795211995000,"value":0,"quality":0}'
PUBLISH iot:newvalue:t49 -> 1
PUBLISH iot:newvalue:t49 -> 1
statusEvaluator = {"instance":"T49.EQ001","current":"RUNNING","evaluations":2,"changes":1,"failures":0}
GET iot:st:t49:T49.EQ001 -> {"statusCode":"RUNNING","statusName":"RUNNING","color":"#00FF00","seq":1,"quality":0,"ts":1795211995000}
trace_sql.log: /rest/sql | INSERT INTO iot_t49.D_T49_EQ001_STATUS USING iot_t49.st_item_value TAGS (1, 'full:T49.EQ001.STATUS', 'T49.EQ001', 'STATUS', 'status', 1) VALUES (1795211995000, 1.0, 0)
```

→ 求值链成立（CURRENT>0 命中 RUNNING，seq=1），Redis JSON 字段符合 §1.5B，留痕 value=稳定序号 quality=0、子表首写 USING TAGS 含 tenant_id=1。

**步骤 3：故障优先于运行**

```
$ pub iot:newvalue:t49 '{"pointCode":"T49.FAULT_FLAG","ts":1795211996000,"value":1,"quality":0}'
statusEvaluator = {...,"current":"FAULT","evaluations":3,"changes":2,"failures":0}
GET iot:st:t49:T49.EQ001 -> {...,"statusCode":"FAULT","seq":3,"ts":1795211996000}
trace INSERT #2: ... VALUES (1795211996000, 3.0, 0)
```

→ CURRENT 仍 >0（运行条件成立），FAULT_FLAG=1 命中高优先级规则 → FAULT 覆盖 RUNNING（TC-0016 步骤 3 通过）。

**步骤 4：未变不重写（Redis + st_item_value）**

```
$ pub iot:newvalue:t49 '...T49.FAULT_FLAG value=1 ts=1795211996000'  ×3
INSERT before=2 after=2（expect equal）
statusEvaluator = {...,"current":"FAULT","evaluations":6,"changes":2,"failures":0}
GET iot:st:t49:T49.EQ001 -> {...,"ts":1795211996000}（未变）
```

→ 3 次同状态到达：evaluations +3 但 changes 不变、Redis ts 不动、INSERT 行数不变——双写口「未变不重写」live 实证。

**步骤 5：LWT 离线触发重判**

```
$ pub iot:gwstatus:t49 '{"gateway":"GW1","online":false,"ts":1795211997000}'
PUBLISH iot:gwstatus:t49 -> 1
INFO ... StatusEvaluatorRuntime : [CALC-STATUS] LWT 离线事件触发重判 channel=iot:gwstatus:t49 payload={"gateway":"GW1","online":false,"ts":1795211997000}
INFO ... StatusEvaluatorRuntime : [CALC-STATUS] 状态变化 instance=T49.EQ001 from=OFFLINE to=LWT seq=4 redis=true
statusEvaluator = {...,"current":"OFFLINE","evaluations":7,"changes":3,"failures":0}
GET iot:st:t49:T49.EQ001 -> {...,"statusCode":"OFFLINE","seq":4,"quality":0,"ts":1795460121070}
trace INSERT #3: ... VALUES (1795460121070, 4.0, 0)
```

→ LWT 离线作为判定输入优先裁决 OFFLINE（ADR-0010），不直写状态、走同一判定链。

**步骤 6：UNKNOWN 不硬套**

```
$ pub iot:newvalue:t49 'T49.CURRENT value=0' + 'T49.FAULT_FLAG value=0'（ts=1795211998000）
statusEvaluator = {...,"current":"UNKNOWN","evaluations":9,"changes":5,"failures":0}
GET iot:st:t49:T49.EQ001 -> {"statusCode":"UNKNOWN","seq":null,"quality":3,"ts":1795211998000}
trace INSERT #4: ... VALUES (1795211998000, -1.0, 3)
```

→ 全规则不命中 → UNKNOWN + 质量码 3，Redis seq=null（不写 0）、留痕 -1 哨兵 + quality=3，不硬套 STOPPED/RUNNING。

**步骤 7：全停 + scratch 清理**

```
=== STOP sequence ===
成功: 已终止 PID 为 82152 的进程。（calc java）
成功: 已终止 PID 为 65876 的进程。（fake backend）
=== ports after ===
BOTH PORTS FREE
=== cleanup smoke redis keys ===
DEL iot:st:t49:T49.EQ001 -> 1
remaining t49 keys: *0
=== java procs ===
（仅剩环境既有 java.exe 91192，本任务起停的 82152 已终止）
```

scratch 目录 `.tmp-t0049/`（fake_backend.py/redis_helper.py/rules.json/calc.log/trace_sql.log）已整体删除；未写真实 TDengine（留痕由记录式假适配器捕获 SQL）；未碰 backend/frontend/ingest；WSL docker 未碰。

### 2.3 验收 3：范围核对（原始输出）

```
=== git diff --name-only 7628007..HEAD ===
（23 文件，全部 calc/ —— 详见 §1）
=== count ===
23
=== out-of-scope ===
ALL CHANGES IN calc/
=== 工作区剩余 ===
?? .agents/
?? .tmp-t0044v/
（均为开工前既有的未跟踪目录，本任务未 add、未提交、未触碰）
```

---

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 真实 TDengine st_item_value 写入的 live 验真 | TDengine 192.168.31.145:6041 TCP 可连（ping 3ms），但 taosAdapter REST 本会话全程不可用：单语句 `SHOW DATABASES` / `SELECT 1` / `/rest/health` / `/` 均 21s 后 `Recv failure: Connection was reset` 或 15s 超时、HTTP=000 零字节（原始输出见下）。brief 声称「fd 根治可用」与现场实测不符；按契约冻结纪律「契约冻结发现问题报告并停」不适用（非契约问题，是环境），但真库写入无法完成。留痕实现以注入式 SqlRunner 契约测试（3 项）+ live 假适配器捕获真实 INSERT SQL 闭合形态，真机联调待 TDengine 恢复后补跑。 |
| 规则集随 0024 发布链真下发 | 冻结快照契约 config-snapshot-schema v1.0.1 §11 明示 `iot_status_rule` 不在五对象集内、"随 3.3 配置同步下发的承载对象上游未定"；本任务不改冻结契约，`StatusRuleProvider` 为规则源缝，本期 env 种子 JSON，真发布链接入归 0024/演进位（已在 README/报告登记）。 |
| 多实例 Redis 化幂等/多实例实例键路由 | brief 口径本期单实例；Redis writer/trace 的未变不重写为进程内缓存；多实例部署的一致性归演进位（与 0041 WindowIdempotencyGuard 同款登记）。 |
| 展示端三处状态一致 / 翻译随规则发布变化 | 归后续 0050 状态读口 + 前端 0059/0058 批二（brief 明文不碰）。 |
| 状态告警（三期）/ 状态时长历史还原 | 告警三期超范围；st_item_value 序列已留痕，时长还原属展示/数据服务读侧。 |
| iot_calc_log / calc_state 写 | 归 0063（brief 不碰）。 |
| TASK 置「已完成」 | 硬约束未置，保持「进行中」待评审；未改 workitems 库（报告除外）。 |

TDengine 环境失败原始输出（本会话实测，逐条独立单语句）：

```
$ curl -m 60 -u root:taosdata http://192.168.31.145:6041/rest/sql --data-raw "SHOW DATABASES"
HTTP=000  curl: (56) Recv failure: Connection was reset  real 0m21.153s
$ curl -m 60 -u root:taosdata http://192.168.31.145:6041/rest/sql --data-raw "SELECT 1"
HTTP=000  curl: (56) Recv failure: Connection was reset  real 0m21.124s
$ curl -m 15 http://192.168.31.145:6041/rest/health
HTTP=000  curl: (28) Operation timed out after 15012 milliseconds with 0 bytes received
$ curl -m 15 http://192.168.31.145:6041/
HTTP=000  curl: (28) Operation timed out after 15018 milliseconds with 0 bytes received
```

---

## 4. 偏离计划之处

1. **接入取向选独立组件桥接而非改 DualTriggerExecutor 主链**（brief 决策 1 二选一）：状态求值做 `status/` 独立包，以 `ResultSink` 装饰器接在 0041 落地边界后 + LWT 事件通道双输入。为何：0041 的 140 测试与执行器拓扑逻辑零改动即可接入，满足零回归硬指标；javadoc/README 已载明取向。为接线仅给 ExecutorRuntime 加构造重载（原 8 参构造委托新构造、行为不变）。
2. **状态规则源用 env 种子 JSON 而非快照**（schema §11 张力处置）：冻结快照不装载 iot_status_rule，且 brief 未给规则种子形态；采用 `StatusRuleProvider` 缝 + env JSON（dict/rules/direct），fail-closed 解析。已在 README「规则源缝」与 §3 登记为演进位；未改任何冻结契约文件。
3. **status-enabled=false 时 statusRuntime Bean 返回 null**：为保持 0040/0041 既有行为零变化，注入点加 `@Nullable`，`CalcEngineApplicationTest`（contextLoads）实测 PASS。非缺陷——显式声明"未启用即无状态组件"。
4. **UNKNOWN 留痕 value 用 -1 哨兵**（§1.5B 只说"value 存状态序号"、未定 UNKNOWN 写什么）：UNKNOWN 无字典序号，写 0 会冒充已知状态（与"不硬套"冲突），故写 -1 + quality=3 打标；Redis JSON 侧 seq=null。契约测试与 live 均锁定该口径。
5. **状态来源标记 source_type=1**：st_item_value 标签 source_type 契约允许 1/2/3，状态判定非公式/汇总，按直绑语义固定 1（TdengineTraceWriter 常量 + javadoc 载明）。
6. **TDD 红转绿留痕**（brief 硬约束，逐修复附「为什么现在对了」）：
   - **求值引擎**：先写 `StatusEvaluatorContractTest` 8 项 → 桩抛 `UnsupportedOperationException`，红（实测 `Tests run: 8, Failures: 0, Errors: 8`）。实现按 priority 升序排序（列表乱序无关）、直绑/LWT 短路、字典单一序号、单规则 try/catch 隔离、UNKNOWN+码 3 → 绿。**为什么现在对了**：排序消除了依赖配置落库顺序的脆弱性；短路顺序直绑>LWT>规则匹配 §1.5B；失败隔离防一条坏规则炸穿整次判定。
   - **运行时双写口**：先写 `StatusRuntimeContractTest` 6 项 → 红（缺注入构造，编译期即暴露）。加注入构造 + 变化判据 `sameStatus`（只看编码）+ 两写口各自缓存 → 绿。**为什么现在对了**：以"编码变"为唯一变位判据（质量码变化不算变位），Redis 与 trace 各自缓存保证同状态零写；判定异常返回 null 不覆盖旧值，trace 异常被 catch 只 WARN。
   - **留痕 SQL**：先写 `TdengineTraceWriterContractTest` 3 项 → 红（无实现）。实现逐语句 INSERT（0 分号、无 USE、全限定库名、USING TAGS）、SqlRunner 注入缝、失败包装 TraceWriteException → 绿。**为什么现在对了**：0029 实证多语句信封静默空操作（code=0 rows=0），单语句全限定是唯一可与真 taosAdapter 对话形态；注入缝让形态在无真库时被机械锁定。
   - 期间修正均为断言/构造语法问题（非空实现缺陷）：`assertEquals(Integer 1, Long 1)` → `1L`；SQL 单语句无分号断言从 `=1` 改 `=0`；Jackson 3 `properties().forEach(entry -> ...)` 单参形态；`LwtEventSource` 注入由 lambda 改匿名类消除重载歧义。

---

## 5. 遗留问题

1. **真 TDengine 写入联调**（环境阻断）：TDengine 恢复后按验收命令补跑：建 scratch 库→真 `st_item_value` INSERT→`SELECT ... ORDER BY ts DESC LIMIT 1` 验证 value=seq/quality=0/未变不重写→DROP scratch 库。契约形态已锁定，补跑为形态复核。
2. **状态规则随发布下发**（0024 演进位）：`StatusRuleProvider` 现为 env 种子；发布链承载对象上游未定版（schema §11 张力），定版后替换提供者实现，Runtime/evaluator 零改动。
3. **多实例部署**：未变不重写缓存为进程内；多实例需 Redis 侧状态缓存或共享判据（与 0041 幂等 Redis 化同族演进位）。
4. **状态读口与展示**：0050 状态读口（读 Redis+st_item_value）+ 前端三处展示一致性属后续道；本任务只交付判定与写入侧。
5. **LWT 事件 payload 形态**：本期只判"收到即离线"，未解析 payload 内 online/ts 字段（0029 发射侧最终字段形态未定版）；如后续要求"仅 online=false 触发"需在 RedisLwtEventSource 回调处加解析，判定链不变。
6. **TASK 状态**：保持「进行中」待评审，未置「已完成」；未回写 workitems 库（本报告除外）。

---

## 一句话结论

**通过**——验收 1（mvn test 159 测试 0F0E3Skip BUILD SUCCESS，0041 的 140 基线逐类零回归、真 Redis 门控单跑 PASS）、验收 2（live 冒烟全序列：故障优先于运行、RUNNING→FAULT→LWT OFFLINE→UNKNOWN 四态判定与 Redis iot:st JSON、st_item_value 逐语句 INSERT 留痕、同状态 3 次零重写、全停归零 scratch 清理）、验收 3（提交 fded29b 23 文件全落 calc/** 越界 0）三条全部以原始输出为证通过；真 TDengine 写入因环境 REST 不可用未做（已登记遗留），不影响三条验收判定，TASK 保持「进行中」待评审。

---

# 修订轮（退回一轮 §12/§13：F-01/F-02 Important + F-03/F-05 Minor）

- 评审输入：`TASK-0049-review.md`（2 Important〔F-01 真写入证据不足 / F-02 full_key 契约错位〕+ 5 Minor，0 Critical）
- 协调者裁决：F-01/F-02 必修，F-03/F-05 顺手修，F-04/F-06/F-07 登记 deferred 不修
- 修订轮 BASE：`fded29b`（被评审提交）
- 交付提交：`331f860`（`TASK-0049 修订轮: F-02 full_key契约对齐/F-01 taosAdapter应答解析验证/F-03 LWT online解析/F-05 文档顺序`；6 文件全部 `calc/**`，越界 0）
- 修订轮测试：契约测试 **159 → 163**（F-02 tag 六元组精确断言 +1、F-01 taosAdapter 应答样本解析 +1、F-03 online 解析/恢复 +2），既有 159 逐类零回归；全量 163 测试 0F0E3Skip BUILD SUCCESS。

## R1. 逐条修复与红绿证据

### R1.1 F-02（Important）st_item_value full_key 契约错位——红转绿

- **发现核实**：`TdengineTraceWriter.java:116` 旧代码写 `'full:' + instanceCode + '.' + STATUS_DATA_CODE` → `'full:T49.EQ001.STATUS'`。冻结契约 `config-snapshot-schema.md:165` = `instance_code + "." + data_code`、`tdengine-super-tables.md:15/37` 六 tag 顺序（tenant_id/full_key/instance_code/data_code/category/source_type），读侧 `OpenApiTimeseriesClient.java:119` 按 `WHERE full_key = '<规范 fullKey>'` 查询——前缀使状态行读不到。评审指出旧契约测试只断言 `contains("'status'")`/`contains("'STATUS'")`、未断言 full_key 值，故全绿漏掉此缺陷。
- **红**：新增 `TdengineTraceWriterContractTest.tagsSixTupleUsesContractFullKeyWithoutPrefix`——从 INSERT 提取 TAGS 六元组逐值精确断言（尤其 `tags[1]` = `F01.WS01.EQ001.STATUS`、全文无 `full:`）。旧实现 `mvn -q -Dtest='TdengineTraceWriterContractTest#tagsSixTupleUsesContractFullKeyWithoutPrefix' test` → `Tests run: 1, Failures: 1`，实测原始输出：

```
TdengineTraceWriterContractTest.tagsSixTupleUsesContractFullKeyWithoutPrefix:46
  full_key = instance_code + '.' + data_code，无 full: 前缀 ==>
  expected: <F01.WS01.EQ001.STATUS> but was: <full:F01.WS01.EQ001.STATUS>
```

- **修**：`insertSql()` 以 `String fullKey = instanceCode + "." + STATUS_DATA_CODE` 生成标签值（去掉 `full:`），javadoc 载明契约出处与读侧依赖。
- **绿**：`mvn -q -Dtest=TdengineTraceWriterContractTest test` → EXIT=0，surefire `tests=5 fail=0 err=0`；六元组断言（tenant_id=1/full_key=规范值/instance_code/data_code/category/source_type）逐值锁定。
- **为什么现在对了**：full_key 是数据项全局唯一键、读侧 `OpenApiTimeseriesClient` 按规范值等值查询，必须与配置快照/消息模型/公式引用完全同形——`instance_code + "." + data_code` 是唯一合法形态；把六元组每个值做成可执行断言后，前缀、错位、漏 tag 任何一类漂移都会机械变红，杜绝同类缺陷再逃逸。

### R1.2 F-01（Important）真写入证据不足——更强的替代证据 + 真机补验条件

- **核实**：评审判定真 taosAdapter 写入未验真（环境 REST 21s reset / HTTP=000，服务器侧故障，非代码问题）；旧 `rest()` 用 `body.contains("\"code\":0")` 判成功，对序列化变体（`{"code": 0}` 冒号后带空格——0048 报告实证 MySQL JSON 渲染带空格同族风险）会误判失败，且该解析逻辑从未用真应答样本单测。
- **修**：① 把 `rest()` 应答校验抽为可单测静态方法 `requireTaosSuccess`/`extractCode`——正则 `\"code\"\s*:\s*(-?\d+)` 提取数值 code 判 0，空/无 code/非 0 一律 fail-closed（零新增依赖）；② 新增 `restParsesCapturedTaosAdapterResponseSamples` 测试，用 0029/0077 评审报告捕获的真应答样本（`{"code":0,"column_meta":...,"data":[],"rows":0}`、`{"code":0}`、`{"code":9728,"desc":"syntax error..."}`）离线验证解析：成功样本 PASS、错误码 9728/空/无 code 抛 `TraceWriteException`，冒号后空格变体 PASS（旧 contains 实现必红该样本）。EXIT=0，`tests=5 fail=0 err=0`。
- **为什么现在对了**：把「解析真应答」从不可测的 HTTP 闭包里提出来，用真 taosAdapter 应答样本离线锁定 code 判定语义，消除了序列化形态脆弱性；但不能替代真写入效果——**按协调者裁决明确登记交付后条件**：TDengine 恢复后必须真机补跑「写入 + 读回」验证（见 R4）。
- **本轮不等待环境恢复**（用户侧动作），F-01 证据充分性由「应答解析单测 + F-02 精确 tag 断言 + 真机补验条件登记」三件事闭合到本轮可做到的上限。

### R1.3 F-03（Minor）LWT 未解析 online 字段——红转绿

- **红**：新增两项测试——`lwtPayloadParsesOnlineField`（纯函数 6 种 payload 形态）+ `lwtOnlineEventReturnsToRuleEvaluationNotOffline`（端到端：RUNNING→LWT offline→online:true 应回 RUNNING/RULE）。旧实现无 `isOffline` 方法，`mvn -q -Dtest=StatusRuntimeContractTest test` → COMPILATION ERROR：

```
StatusRuntimeContractTest.java:107 找不到符号 符号: 方法 isOffline(java.lang.String)
（×6 处）
```

- **修**：`StatusEvaluatorRuntime.isOffline(String payload)` 轻量文本正则解析 `online`（布尔/字符串/大小写）：`false`/缺字段/坏 payload → 离线（LWT 语义 fail-safe）；`true` → 在线恢复。LWT 回调改为 `evaluate(offline, now)`——离线走离线优先、在线走规则求值（同一判定链，无第二状态写口），日志增 `offline=` 字段。
- **绿**：`mvn -q -Dtest=StatusRuntimeContractTest test` → EXIT=0，`tests=8 fail=0 err=0`；端到端原始日志：

```
INFO ... [CALC-STATUS] LWT 事件触发重判 channel=iot:gwstatus:t49 offline=false payload={"gateway":"GW1","online":true}
INFO ... [CALC-STATUS] 状态变化 instance=EQ1 from=RUNNING to=RULE seq=1 redis=true
→ current=RUNNING source=RULE（在线事件回到规则态，不误裁 OFFLINE）
```

- **为什么现在对了**：LWT 通道语义是「网关在线状态变化事件」（EMQX LWT 发 offline、恢复发 online）——只有 offline 才应离线优先；online 恢复要回到规则求值，否则设备恢复后状态会滞留 OFFLINE 直到下一次数据项更新（评审 F-03 的恢复缺口）。解析放在判定入口单点、不引入新写口，与 ADR-0010「离线只是判定输入」一致。

### R1.4 F-05（Minor）文档顺序——代码为准订正文档

- **核实**：代码 `StatusEvaluator.evaluate` 实际顺序为 LWT 离线优先 → 直绑短路 → 规则（ADR-0010 离线优先合理）；README 与 StatusEvaluator javadoc 旧写「直绑短路 → LWT 离线优先 → 规则」。
- **修**：两处文档改为「LWT 离线优先（ADR-0010）→ 直绑短路 → 规则」，README 目录行同改；代码零改动。
- **为什么现在对了**：文档与实现一致，避免配置/运维按错误顺序理解状态裁决优先级。

## R2. 修订轮验收原始输出

### R2.1 全量 `cd calc && mvn test`（提交 `331f860`）

```
[INFO] Running net.otlab.calc.status.StatusEvaluatorContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.011 s -- in net.otlab.calc.status.StatusEvaluatorContractTest
[INFO] Running net.otlab.calc.status.StatusRedisLiveTest
[WARNING] Tests run: 2, Failures: 0, Errors: 0, Skipped: 2, Time elapsed: 0 s -- in net.otlab.calc.status.StatusRedisLiveTest
[INFO] Running net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.021 s -- in net.otlab.calc.status.StatusRuntimeContractTest
[INFO] Running net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.014 s -- in net.otlab.calc.status.TdengineTraceWriterContractTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 163, Failures: 0, Errors: 0, Skipped: 3
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  10.510 s
[INFO] Finished at: 2026-11-24T03:14:44+08:00
[INFO] ------------------------------------------------------------------------
EXIT=0
```

159 基线逐类零回归（含 0041 的 140）；新增 4 项（F-02 tag 六元组 1 + F-01 应答样本 1 + F-03 online 2）。真 Redis 门控单跑在初验轮已 PASS（本轮未改 Redis 写口），本轮不重复。

### R2.2 修订轮范围核对

```
=== git show --name-only --format="" 331f860 ===
calc/README.md
calc/src/main/java/net/otlab/calc/status/StatusEvaluator.java
calc/src/main/java/net/otlab/calc/status/StatusEvaluatorRuntime.java
calc/src/main/java/net/otlab/calc/status/TdengineTraceWriter.java
calc/src/test/java/net/otlab/calc/status/StatusRuntimeContractTest.java
calc/src/test/java/net/otlab/calc/status/TdengineTraceWriterContractTest.java
=== out-of-scope count ===
0
=== working tree ===
（仅 ?? .agents/、?? .tmp-t0044v/ 开工前既有未跟踪项，本轮未 add/未提交/未触碰）
```

## R3. deferred 登记（协调者裁决不修）

| # | 项 | 登记说明 |
| --- | --- | --- |
| F-04 | Runtime current 缓存与 writer 重试语义打架 | 初验轮报告 §5 已说明（同状态早退减弱短暂故障自愈）；本轮 deferred，演进位：Runtime 在写口失败时不更新 current 或显式重试队列。 |
| F-06 | Redis statusName/color 未字典翻译 | 0050 状态读口职责（读侧以规则表/字典翻译），写入侧只给编码/序号——本轮 deferred。 |
| F-07 | 种子 FAULT=3 与 0048 真字典 FAULT=4 不一致 | 0024 接真规则表时以 `iot_status_dict.seq` 为准，机制（字典单一来源）本身正确——演进衔接风险，本轮 deferred。 |

## R4. TDengine 真机补验条件（交付后必须执行——协调者 TD-0053 环境恢复时安排）

> 权威：评审 `TASK-0049-review.md` 评审重点 4「需补什么」第 2/3/4 条 + 协调者修订裁决 F-01。

本轮环境阻断未解（REST 6041 TCP 通、单语句 15-21s 超时/Connection reset、HTTP=000，服务器侧故障），**不能声称 st_item_value 留痕"真实写入可用"**。TDengine 恢复后必须按以下序列真机补跑（非本轮可完成，属用户侧动作）：

1. 建 scratch 租户库（`CREATE DATABASE IF NOT EXISTS iot_<scratch> KEEP 1825d DURATION 10d BUFFER 256` + 四张 STABLE，逐语句独立下发，20-25s 慢特性）；
2. 经生产 `TdengineTraceWriter.rest()` 真写一条 STATUS 行（verify Basic 认证 + `/rest/sql` + 真应答 code 解析——R1.2 已离线单测该解析）；
3. **读回** `SELECT ... WHERE full_key = '<instance_code>.STATUS' ORDER BY ts DESC LIMIT 1`（用 R1.1 锁定的规范 fullKey）——断言 `value=seq`、`quality=0`、UNKNOWN 行 `value=-1/quality=3`；
4. 同状态再写一次 → 读回行数不变（未变不重写真机验证）；
5. DROP scratch 库、清理。

在 1-4 完成前，st_item_value 留痕的「真实写入可用」标记为**未验真（环境阻断）**；F-02 契约错位已修且精确断言锁定，不影响上述补验的必要性与序列。

## R5. 一句话结论

**修订通过**——F-02（full_key 契约错位红转绿，`expected: F01.WS01.EQ001.STATUS but was: full:F01.WS01.EQ001.STATUS` 为红证，TAGS 六元组逐值断言锁定）、F-01（taosAdapter 真应答样本 code 解析单测 + 真机写入/读回补验条件明确登记为交付后条件）、F-03（LWT online 解析红转绿，在线事件回规则态不误裁 OFFLINE，恢复路径闭合）、F-05（文档顺序订正为 LWT 优先→直绑→规则）全部落地；全量 163 测试 0F0E3Skip BUILD SUCCESS（159 基线逐类零回归）、真 Redis 门控单跑 PASS、提交 `331f860` 6 文件全落 calc/** 越界 0。F-04/F-06/F-07 按裁决 deferred（报告 R3 登记）；TDengine 写入+读回真机补验为交付后条件（R4，环境恢复时执行，当前标记未验真）。待 scoped 复评（只看 F-01/F-02 修复 diff 与评审两条 Important）。
