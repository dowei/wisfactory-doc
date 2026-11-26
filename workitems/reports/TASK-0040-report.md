# TASK-0040 验收报告：依赖图构建与快照消费切换——计算引擎服务引导

> **中断恢复接手**——前一执行者完成代码 + 115 测试后额度中断，本次补 live 冒烟 + 报告。原有 45 个 Java 文件逻辑未改；本次仅补跑验收、live 真链冒烟、提交与报告。

- 任务：TASK-0040（标准任务 5 人天，计算引擎道开线）
- BASE：`537366e`（开工时主仓 HEAD，git log 核对无误）
- 提交：`04e6acd`（`TASK-0040: 计算引擎引导（快照消费/依赖图/原子切换/ACK 闭环）`，49 文件，5011 insertions）
- 执行日期：2026-11-23
- 环境事实：MySQL 8.0.12 @127.0.0.1:3307 / Redis 3.0.504 @6379（宿主 phpStudy，已迁移，实测正常）；backend dev jar `backend/ot-admin/target/ot-admin-1.0.0.jar`（8098，启动 17.6s）；calc jar `calc/target/calc-engine.jar`（8095，启动 7.5s）；TDengine 192.168.31.145:6041 可用但本批不写（无 scratch 库）。

---

## 一、改动文件清单（验收 3：改动范围核对）

`git diff --name-only 537366e..HEAD` 共 49 项，**全部落在 `calc/**`**（+ 新增 `calc/.gitignore`、`calc/README.md`），无 backend / frontend / ingest / workitems / 契约文件变更：

```
calc/.gitignore
calc/README.md
calc/pom.xml
calc/src/main/java/net/otlab/calc/CalcEngineApplication.java
calc/src/main/java/net/otlab/calc/ack/AckCommand.java
calc/src/main/java/net/otlab/calc/ack/AckSender.java
calc/src/main/java/net/otlab/calc/ack/HttpAckSender.java
calc/src/main/java/net/otlab/calc/config/CalcProperties.java
calc/src/main/java/net/otlab/calc/config/ServerPortCustomizer.java
calc/src/main/java/net/otlab/calc/graph/CycleDetector.java
calc/src/main/java/net/otlab/calc/graph/DependencyGraph.java
calc/src/main/java/net/otlab/calc/graph/EdgeType.java
calc/src/main/java/net/otlab/calc/graph/FailureReason.java
calc/src/main/java/net/otlab/calc/graph/FormulaExpressionValidator.java
calc/src/main/java/net/otlab/calc/graph/GraphBuildException.java
calc/src/main/java/net/otlab/calc/graph/GraphBuilder.java
calc/src/main/java/net/otlab/calc/graph/GraphEdge.java
calc/src/main/java/net/otlab/calc/graph/GraphNode.java
calc/src/main/java/net/otlab/calc/graph/NodeKind.java
calc/src/main/java/net/otlab/calc/runtime/AppliedGraph.java
calc/src/main/java/net/otlab/calc/runtime/ConfigChangedSubscriber.java
calc/src/main/java/net/otlab/calc/runtime/GraphHolder.java
calc/src/main/java/net/otlab/calc/runtime/ReconcileCounters.java
calc/src/main/java/net/otlab/calc/runtime/SnapshotReconciler.java
calc/src/main/java/net/otlab/calc/snapshot/DataItem.java
calc/src/main/java/net/otlab/calc/snapshot/FetchedSnapshot.java
calc/src/main/java/net/otlab/calc/snapshot/FormulaRef.java
calc/src/main/java/net/otlab/calc/snapshot/Hashes.java
calc/src/main/java/net/otlab/calc/snapshot/HttpSnapshotFetcher.java
calc/src/main/java/net/otlab/calc/snapshot/Metric.java
calc/src/main/java/net/otlab/calc/snapshot/RawJsonExtractor.java
calc/src/main/java/net/otlab/calc/snapshot/SnapshotBody.java
calc/src/main/java/net/otlab/calc/snapshot/SnapshotFetchException.java
calc/src/main/java/net/otlab/calc/snapshot/SnapshotFetcher.java
calc/src/main/java/net/otlab/calc/snapshot/SnapshotFormatException.java
calc/src/main/java/net/otlab/calc/snapshot/SnapshotParser.java
calc/src/main/java/net/otlab/calc/web/CalcStatusController.java
calc/src/main/resources/application.yml
calc/src/test/java/net/otlab/calc/CalcEngineApplicationTest.java
calc/src/test/java/net/otlab/calc/ack/HttpAckSenderTest.java
calc/src/test/java/net/otlab/calc/config/CalcPropertiesTest.java
calc/src/test/java/net/otlab/calc/graph/CycleDetectorTest.java
calc/src/test/java/net/otlab/calc/graph/FormulaExpressionValidatorTest.java
calc/src/test/java/net/otlab/calc/graph/GraphBuilderTest.java
calc/src/test/java/net/otlab/calc/runtime/SnapshotReconcilerTest.java
calc/src/test/java/net/otlab/calc/snapshot/HttpSnapshotFetcherContractTest.java
calc/src/test/java/net/otlab/calc/snapshot/RawJsonExtractorTest.java
calc/src/test/java/net/otlab/calc/snapshot/SnapshotParserContractTest.java
calc/src/test/java/net/otlab/calc/support/TestSnapshots.java
```

说明：新增 `calc/.gitignore` 镜像 backend 的 target 忽略规则（根 `.gitignore` 无 `target/` 通配），否则 `git add calc/**` 会把 `calc/target/` 下 jar/class/surefire 产物一并提交；加入后 `git add calc/` 仅暂存源码 49 项，构建产物未入库。

**范围核对原始输出**：

```
=== HEAD after commit ===
04e6acd TASK-0040: 计算引擎引导（快照消费/依赖图/原子切换/ACK 闭环）
537366e TASK-0064 高危权限点清单注册与菜单种子（机械差集收口）
=== scope check BASE..HEAD ===
calc/.gitignore
...（上列 49 项）...
=== count ===
49
=== out-of-scope check ===
ALL CHANGES IN calc/
```

工作区中另有并行道产生的未跟踪文件（`.agents/`、`backend/ot-admin/src/test/.../OperateLogCoverageGuardTest.java`、`resources/scripts/audit-operatelog-coverage.py`），本次 **未 add、未提交**（严格遵守 `git add calc/**`，绝不 `git add -A`）。

---

## 二、每条验收命令原始输出

### 验收 1：`cd calc && mvn test`（重跑留证）

原命令：`cd /d/workspace/WisFactory/calc && mvn test`。结果 **BUILD SUCCESS，Tests run: 115, Failures: 0, Errors: 0, Skipped: 0**（前一执行者成果复核通过，未改代码逻辑）。原始输出（关键段，完整日志已留存）：

```
[INFO] --- surefire:3.5.6:test (default-test) @ calc-engine ---
[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running net.otlab.calc.ack.HttpAckSenderTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.639 s -- in net.otlab.calc.ack.HttpAckSenderTest
[INFO] Running net.otlab.calc.CalcEngineApplicationTest
... Spring Boot 4.1.0 启动日志（Tomcat started on port 8095）...
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 6.654 s -- in net.otlab.calc.CalcEngineApplicationTest
[INFO] Running net.otlab.calc.config.CalcPropertiesTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.026 s -- in net.otlab.calc.config.CalcPropertiesTest
[INFO] Running net.otlab.calc.graph.CycleDetectorTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.027 s -- in net.otlab.calc.graph.CycleDetectorTest
[INFO] Running net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.053 s -- in net.otlab.calc.graph.FormulaExpressionValidatorTest
[INFO] Running net.otlab.calc.graph.GraphBuilderTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.099 s -- in net.otlab.calc.graph.GraphBuilderTest
[INFO] Running net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.073 s -- in net.otlab.calc.runtime.SnapshotReconcilerTest
[INFO] Running net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.315 s -- in net.otlab.calc.snapshot.HttpSnapshotFetcherContractTest
[INFO] Running net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.021 s -- in net.otlab.calc.snapshot.RawJsonExtractorTest
[INFO] Running net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.088 s -- in net.otlab.calc.snapshot.SnapshotParserContractTest
[INFO] Results:
[INFO] Tests run: 115, Failures: 0, Errors: 0, Skipped: 0
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  24.321 s
[INFO] Finished at: 2026-11-23T09:53:09+08:00
[INFO] ------------------------------------------------------------------------
EXIT=0
```

注：验收命令 brief 写 `mvn install && mvn test`；本次按中断恢复指令只重跑 `mvn test`（`install` 产物 jar 已存在且源码未变，`test` 已覆盖编译+全量用例）。此点记为「偏离计划之处」。

### 验收 2：live 真链冒烟全序列（每步原始输出）

**步骤 0：启动前端口探测**

```
=== port 8095/8098 listeners ===
NONE
=== backend jar ===
-rw-r--r-- 1 dowil 197609 133649940 Nov 23 05:31 backend/ot-admin/target/ot-admin-1.0.0.jar
=== calc jar ===
-rw-r--r-- 1 dowil 197609 19942467 Nov 23 08:01 calc/target/calc-engine.jar
=== java procs ===
NO RELEVANT JAVA PROC
```

**步骤 1：backend dev 起（8098，注入 `OT_GATEWAY_INTERNAL_TOKEN=tok-0040-smoke`）**

```
$ cd backend/ot-admin && OT_GATEWAY_INTERNAL_TOKEN=tok-0040-smoke java -jar target/ot-admin-1.0.0.jar --spring.profiles.active=dev
...
2026-11-23 09:56:04.885 [main] INFO  org.springframework.boot.tomcat.TomcatWebServer - Tomcat started on port 8098 (http) with context path '/'
2026-11-23 09:56:04.902 [main] INFO  net.otlab.admin.OtAdminApplication - Started OtAdminApplication in 17.617 seconds (process running for 18.241)
=== port check ===
  TCP    0.0.0.0:8098           0.0.0.0:0              LISTENING       61996
  TCP    [::]:8098              [::]:0                 LISTENING       61996
```

**步骤 2：快照真拉取 + 哈希可重算校验（内部令牌端点）**

无令牌 → fail-closed（内部端点纪律），带令牌 → code=200：

```
=== snapshot endpoint (no token) ===
HTTP 200
{"code":401,"data":null,"message":"内部令牌无效","timestamp":1795398971780}
=== snapshot endpoint (with token) ===
HTTP 200
code 200 msg 操作成功
version 6 hash b89df2060ed1a23446e6d624b5588230f7608e1d9111f5e0337c22bd05ab5b2e uri 1/2-0-3.json schema 1.0.1
snapshot type dict keys ['schemaVersion', 'tenantId', 'scopeType', 'scopeId', 'version', 'dataItems', 'formulas', 'metrics']
```

对 data.snapshot 原文逐字节重算 SHA-256（独立脚本，模拟 calc RawJsonExtractor + Hashes.sha256Hex）：

```
envelope hash: b89df2060ed1a23446e6d624b5588230f7608e1d9111f5e0337c22bd05ab5b2e
recomputed  : b89df2060ed1a23446e6d624b5588230f7608e1d9111f5e0337c22bd05ab5b2e
MATCH: True
snapshot bytes: 6158
```

既有库快照数据（scopeType=2，租户 default=id1，21 行中计算域 6 行）：

```
id	tenant_id	scope_type	scope_id	version	snapshot_hash	snapshot_uri	publish_status	publish_time
24	1	2	0	1	9d4ea7328e2d16dcec70392549e547ce03aa4eb597df948893f9359aaeea7b15	1/2-0-1.json	1	2026-11-22 23:49:48
29	1	2	0	2	4d5e05b141a95b9548c258fe7d5a30284382713c2a4f89af9298b551f4d55e85	1/2-0-2.json	1	2026-11-23 02:15:17
30	1	2	0	3	b89df2060ed1a23446e6d624b5588230f7608e1d9111f5e0337c22bd05ab5b2e	1/2-0-3.json	1	2026-11-23 02:15:30
32	1	2	0	4	f66845910da9427643e220f9fab1431701a8e3a12b8e7a7aaba7d184d4b958fb	1/2-0-4.json	1	2026-11-23 04:40:09
33	1	2	0	5	e82b8d3d7a00716d0f3cec8fb3cfebb2f0ffb33586ed76e56152e8eca02835ec	1/2-0-5.json	1	2026-11-23 04:43:33
34	1	2	0	6	b89df2060ed1a23446e6d624b5588230f7608e1d9111f5e0337c22bd05ab5b2e	1/2-0-3.json	1	2026-11-23 06:28:16
```

**步骤 3：造冒烟版本行 + calc 起（8095，对账周期 10s）**

既有 6 个真实快照体（v2~v5 文件）均含前序冒烟遗留的悬空引用项 `F01.WS13.R2DQ.R2_DEAD2 → R2_DEAD1`（引用目标不存在），若直接消费，calc 会正确 fail-closed。为验收「构建成功 → ACK 1」段，另建两个经 backend 真端点/真存储提供的版本行：
- v40 `1/2-0-40-clean.json`（从真实 v3 体移除悬空项 id=8 及其引用行，hash `9f212812086a710987f5a48f269f970ea2957cb9ffa01e2e330d07e3f5a2b09c`，18 数据项）；
- v41 `1/2-0-40-bad.json`（在 clean 体上新增一条指向不存在目标的引用边 ref#9999，hash `9d0f72a82b09ccec421ce13fc8eda4c0660c3138af23bcbf4fb57aba54629747`），并各配 `(version_id, target_type=3, target_id='calc', ack_status=0)` 下发记录行。

原命令：

```
$ cd calc && CALC_BACKEND_BASE_URL=http://127.0.0.1:8098 CALC_INTERNAL_TOKEN=tok-0040-smoke CALC_TENANT=default CALC_SNAPSHOT_RECONCILE_SECONDS=10 java -jar target/calc-engine.jar
...
2026-11-23T09:58:46.815+08:00  INFO 63532 --- [calc-engine] [main] o.s.boot.tomcat.TomcatWebServer : Tomcat started on port 8095 (http) with context path '/'
2026-11-23T09:58:46.857+08:00  INFO 63532 --- [calc-engine] [main] net.otlab.calc.CalcEngineApplication : Started CalcEngineApplication in 7.532 seconds
2026-11-23T09:58:46.868+08:00  INFO 63532 --- [calc-engine] [main] n.otlab.calc.runtime.SnapshotReconciler : [CALC-RECONCILE] 对账循环启动（启动即拉 + 每 10s 对账；config.changed 订阅缝占位 TD-0057）
2026-11-23T09:58:47.231+08:00  INFO 63532 --- [shot-reconciler] n.otlab.calc.runtime.SnapshotReconciler : [CALC-RECONCILE] 依赖图已切换 version=40 hash=9f212812086a710987f5a48f269f970ea2957cb9ffa01e2e330d07e3f5a2b09c nodes=19 edges=13 topo=19
```

**步骤 4：依赖图构建成功 + 观测口 + ACK 真落库**

```
$ curl -s http://127.0.0.1:8095/calc/v1/status
{"status":"running","version":40,"hash":"9f212812086a710987f5a48f269f970ea2957cb9ffa01e2e330d07e3f5a2b09c","appliedAt":"2026-11-23T01:58:47.231597400Z","nodeCount":19,"edgeCount":13,"lastAttempt":{"version":40,"ok":true,"error":null,"at":"2026-11-23T01:58:47.231597400Z"},"counters":{"ackOk":1,"pulls":1,"replaced":1,"ackFail":0,"skipped":0,"buildFailures":0,"fetchFailures":0}}

$ bash backend/scripts/sql-exec.sh "SELECT id, version_id, target_type, target_id, ack_status, ack_time, error_msg, update_time FROM iot_config_publish_log WHERE version_id=40 AND target_type=3 AND target_id='calc'"
id	version_id	target_type	target_id	ack_status	ack_time	error_msg	update_time
21	40	3	calc	1	2026-11-23 09:58:48	null	2026-11-23 09:58:48
DONE

=== calc log ACK line ===
2026-11-23T09:58:47.793+08:00  INFO 63532 --- [shot-reconciler] n.otlab.calc.runtime.SnapshotReconciler : [CALC-RECONCILE] ACK 生效已回执 version=40 targetType=3 targetId=calc
```

ACK 成功行真实落库：`id=21, ack_status=1, ack_time` 已写，`error_msg=null`。

**步骤 5：造坏快照 → 拉取/构建失败 → 旧图保持 + ACK 失败**

```
$ bash backend/scripts/sql-exec.sh "INSERT INTO iot_config_version (...) VALUES (41, 1, 2, 0, 41, '9d0f72...', '1/2-0-40-bad.json', 1, NOW(), NOW())"
OK(1)
$ bash backend/scripts/sql-exec.sh "INSERT INTO iot_config_publish_log (version_id, target_type, target_id, ack_status, create_time) VALUES (41, 3, 'calc', 0, NOW())"
OK(1)
=== wait for reconcile cycle (10s) ===
=== calc log ===
2026-11-23T09:59:07.151+08:00 ERROR 63532 --- [shot-reconciler] n.otlab.calc.runtime.SnapshotReconciler : [CALC-RECONCILE] 新版本构建失败（保持旧图 version=40 继续可用）version=41 原因: 悬空引用: ref#9999 F01.WS13.R2OKQ.ELEC_CONS -> F01.WS13.R2DQ.SMOKE_MISSING_TARGET（引用目标不在快照——上游逻辑删未清边或装载缺失）
2026-11-23T09:59:07.217+08:00 ERROR 63532 --- [shot-reconciler] n.otlab.calc.runtime.SnapshotReconciler : [CALC-RECONCILE] ACK 失败回执未受理 version=41 msg=依赖图构建失败(DANGLING_REF): ...
2026-11-23T09:59:17.302+08:00 ERROR 63532 --- [shot-reconciler] n.otlab.calc.runtime.SnapshotReconciler : [CALC-RECONCILE] 新版本构建失败（保持旧图 version=40 继续可用）version=41 原因: 悬空引用: ref#9999 ...
2026-11-23T09:59:17.367+08:00  INFO 63532 --- [shot-reconciler] n.otlab.calc.runtime.SnapshotReconciler : [CALC-RECONCILE] ACK 失败已回执 version=41 msg=依赖图构建失败(DANGLING_REF): ...
2026-11-23T09:59:27.403+08:00 ERROR 63532 --- [shot-reconciler] n.otlab.calc.runtime.SnapshotReconciler : [CALC-RECONCILE] 新版本构建失败（保持旧图 version=40 继续可用）version=41 原因: 悬空引用: ref#9999 ...
2026-11-23T09:59:27.448+08:00  INFO 63532 --- [shot-reconciler] n.otlab.calc.runtime.SnapshotReconciler : [CALC-RECONCILE] ACK 失败已回执 version=41 msg=依赖图构建失败(DANGLING_REF): ...
=== status (old graph should remain v40) ===
{"status":"running","version":40,"hash":"9f212812086a710987f5a48f269f970ea2957cb9ffa01e2e330d07e3f5a2b09c","appliedAt":"2026-11-23T01:58:47.231597400Z","nodeCount":19,"edgeCount":13,"lastAttempt":{"version":41,"ok":false,"error":"悬空引用: ref#9999 F01.WS13.R2OKQ.ELEC_CONS -> F01.WS13.R2DQ.SMOKE_MISSING_TARGET（引用目标不在快照——上游逻辑删未清边或装载缺失）","at":"2026-11-23T01:59:27.403490200Z"},"counters":{"ackOk":3,"pulls":5,"replaced":1,"ackFail":1,"skipped":1,"buildFailures":3,"fetchFailures":0}}
```

`status` 始终 `version=40`（旧图保持、持续可用），`lastAttempt.ok=false` 且 error 精确记录悬空引用原因；`counters.replaced=1` 未变、`buildFailures=3`——配置错误未打停计算（语义在图层确立）。

ACK 失败真实落库：

```
$ bash backend/scripts/sql-exec.sh "SELECT id, version_id, target_type, target_id, ack_status, ack_time, error_msg, update_time FROM iot_config_publish_log WHERE version_id=41 AND target_type=3 AND target_id='calc'"
id	version_id	target_type	target_id	ack_status	ack_time	error_msg	update_time
22	41	3	calc	2	null	依赖图构建失败(DANGLING_REF): 悬空引用: ref#9999 F01.WS13.R2OKQ.ELEC_CONS -> F01.WS13.R2DQ.SMOKE_MISSING_TARGET（引用目标不在快照——上游逻辑删未清边或装载缺失）	2026-11-23 09:59:27
DONE

=== 成功行仍 status=1（未被倒退）===
id	version_id	ack_status	ack_time
21	40	1	2026-11-23 09:58:48
DONE
```

**步骤 6：全停归零 + 冒烟数据恢复**

```
=== taskkill calc 63532 ===
成功: 已终止 PID 为 63532 的进程。
=== taskkill backend 61996 ===
成功: 已终止 PID 为 61996 的进程。
=== ports after ===
BOTH PORTS ZERO
=== jps after ===
NO RELEVANT JAVA PROC
```

数据库/文件恢复（删除 v40/v41 版本行与下发记录、删除两 fixture 文件），基线核对：

```
=== verify baseline restored ===
max_ver_id	n
34	21
DONE
max_log_id	n
20	20
DONE
id	version	snapshot_hash
34	6	b89df2060ed1a23446e6d624b5588230f7608e1d9111f5e0337c22bd05ab5b2e
DONE
id	version_id	ack_status	error_msg
20	34	2	ACK 超时（超过 300 秒未回执）
DONE
=== file hash unchanged ===
b89df2060ed1a23446e6d624b5588230f7608e1d9111f5e0337c22bd05ab5b2e
=== fixture files gone ===
NO FIXTURE FILES
```

dev 库数据恢复至冒烟前基线（max 版本行 id=34、max 下发记录 id=20，原行 ack 状态不变），临时 `.smoke-t0040/` 已删除；无 scratch 库、未写 TDengine。

---

## 三、未做项与原因

1. **`mvn install` 段单独留证**：按中断恢复指令只重跑 `mvn test`（jar 已存在且源码未变，`test` 已覆盖编译 + 全量用例）；`install` 的产物 jar 在 `calc/target/` 已存在并被本次 live 冒烟实际消费。
2. **执行器/算子（0041/0042）、iot_calc_log/state 写（0041+/0063）、st_metric_value（0045）、状态判定（0049）**：brief 决策 7 明确不在本批范围，计划内未做，非缺口。
3. **config.changed 真订阅（TD-0057）、NotifySender 真联调（TD-0056）**：本期拉取驱动（启动全量拉取 + 可配对账周期），订阅缝 `ConfigChangedSubscriber` 为 NoOp 占位，计划内未做（详见遗留问题）。
4. **契约冻结变更**：三份冻结契约未改；实测 ACK 端点形态、快照解析、哈希重算均与契约一致，未发现设计级缺陷。
5. **prod 档启动回归**：brief 验收未列 prod 段（仅 dev live 冒烟），未做。

---

## 四、偏离计划之处

1. **成功路径使用临时冒烟版本行而非直接消费既有快照**：库内 6 个既有 scopeType=2 快照体（v2~v5 文件）均含前序冒烟遗留的悬空引用项 `R2_DEAD2 → R2_DEAD1`（实测 4 个文件逐一检查均命中），直接消费只能走「构建失败 + ACK 2」路径。为覆盖「构建成功 + ACK 1」及「失败保旧图」全序列，按 brief 验收 2 允许的「造坏快照（SQL 改版本行 content 或临时挡端点——任选可行法）」同源思路，新增 v40（修复体）/v41（坏体）两个版本行 + 快照文件 + 下发记录，均经 **backend 真端点/真存储**提供（非 mock），且成功段哈希重算一致。冒烟结束已全量恢复基线（见验收 2 步骤 6）。
2. **新增 `calc/.gitignore`**：brief 仅写 `calc/**`，新顶层 Maven 工程若无本地 target 忽略规则，`git add calc/**` 会把构建产物入库；补该文件属 calc/** 范围内必要工程文件（镜像 backend/.gitignore），未越界。
3. **未单独执行 `mvn install`**：见未做项 1。
4. **ACK 首次回执曾出现一次「未受理」**：v41 首次失败回执时计数 `ackFail=1`（疑似 backend ACK 超时扫描器并发窗口），下一对账周期即 `ACK 失败已回执`（DB 行 id=22 最终 ack_status=2 + errorMsg 完整）。未改代码；作为遗留观察项登记。

---

## 五、遗留问题

1. **config.changed 订阅缝（TD-0057 延续登记）**：`ConfigChangedSubscriber` 为 NoOp 占位（javadoc 载明接线义务），本批使用拉取驱动（启动全量拉取 + `CALC_SNAPSHOT_RECONCILE_SECONDS` 默认 300s 对账）兜底；NotifySender 真联调未通（TD-0056），待 0026 通知链真通后由下游接线替换该 Bean，调用点不扩散。**本批只登记 TD，不改 workitems 库**。
2. **ACK 回执首次偶发未受理（观察项）**：live 冒烟中 v41 首次失败回执日志出现一次 `ACK 失败回执未受理`（`counters.ackFail=1`），次周期成功。`HttpAckSender.send` 在业务码非 200 / 网络失败时返回 false 并由调用方计数告警（fail-soft 设计），未打停对账主链；原因疑为 backend `PublishAckTimeoutScanner` 与 calc 首次回执争用同一行（窗口极短）。建议后续道若接入 config.changed 高频触发时，观察 ackFail 计数趋势，必要时给 ACK 加重试退避（本批不改逻辑）。
3. **库内既有快照含悬空引用种子**：`F01.WS13.R2DQ.R2_DEAD2 → R2_DEAD1` 为早期冒烟遗留数据，calc 对其 fail-closed 属正确行为；若后续真计算道直接消费这些版本会得到构建失败 + ACK 2，需由配置治理侧清理或补齐引用边（非计算引擎缺陷）。
4. **TASK 状态**：按硬约束未将 TASK-0040 置「已完成」，保持「进行中」，由协调者安排评审后裁决。

---

## 六、一句话结论

中断恢复接手完成：115 单测重跑全绿（0F0E）、live 真链冒烟全序列通过（真快照哈希重算一致、成功 ACK ack_status=1 真落库、坏快照保持旧图且 ACK ack_status=2+errorMsg 真落库、端口归零、数据恢复基线），calc/** 已提交 `04e6acd`（49 文件全在范围内），遗留 config.changed 订阅缝按 TD-0057 登记待续，TASK 保持「进行中」待评审。
