# TASK-0041 diff 评审报告（代码评审）

- 评审对象：提交 `9a6cda6`（BASE=`3cdde58`，23 文件全 `calc/**`）——双触发执行器（周期调度+到达驱动+节流幂等+汇总算子+失败隔离）
- 评审角色：fresh 代码评审（L1-execute §13「diff 评审」），仅依据 brief 指向的文件与工作区源码
- 评审日期：2026-11-24
- 评审手段：通读 diff 全 2452 行 + 对照 `calc/src/**` 源码（0040 图结构/GraphBuilder/DataItem/契约）+ 契约库（CONV-02 质量码表、config-snapshot-schema）；**实测复现**（`mvn test`、对编译产物用 javac/jshell 验证比较运算符、env 门控真 Redis live 测试）

---

## 结论

**不通过（存在 Important 级缺陷，需退回一轮）**。

发现 **3 项 Important、5 项 Minor、0 项 Critical**。其中 `ArithmeticEvaluator` 的 `>=`/`<=` 比较方向被实测证明反转（静默产出错误计算结果），`ValueStore` 存在真实的 HashMap 并发读写数据竞争，两项均为 §13 所述「明显质量缺陷（错误处理、边界、安全）」，触发 §12 退回路径。其余主线（拓扑执行、ResultSink 边界、失败隔离打码 7、RESP2 订阅、范围核对、测试计数）经核验成立。

---

## 一、发现清单

| 编号 | 严重级 | 出处（文件:行） | 一句话说明 |
| --- | --- | --- | --- |
| F-01 | **Important** | `calc/src/main/java/net/otlab/calc/executor/ArithmeticEvaluator.java:108,110` | `>=`/`<=` 比较操作数反转，`10>=3` 实测得 0.0、`3<=10` 得 0.0，公式项静默产出反向布尔值。 |
| F-02 | **Important** | `calc/src/main/java/net/otlab/calc/executor/ValueStore.java:19,22,27` | `put()` 为 `synchronized`、`get()` 未同步，底层 `HashMap` 被调度线程与订阅线程并发读写，存在数据竞争/扩容死循环风险（javadoc 的「线程安全」声明不成立）。 |
| F-03 | **Important** | `calc/src/main/java/net/otlab/calc/executor/ExecutorRuntime.java:54,63,64` + `DualTriggerExecutor.java:83` + 测试 `DualTriggerExecutorContractTest.java:2114-2125` | 「同窗保留最后正确结果」未兑现：首个事件即计算并交付，同窗后续事件只写 store 不重算，产出结果反映的是窗口**首个**值；契约测试更新为 9.0 后从未断言交付值，留下验收盲区。 |
| F-04 | Minor | `calc/src/main/java/net/otlab/calc/executor/PeriodicScheduler.java:29,41,53` | 类 javadoc 称「用 WindowIdempotencyGuard 保证同窗只触发一次」，但该 `guard` 字段赋值后从未使用（实际靠 `lastFiredBucket` 去重）；文档与实现漂移 + 死字段。 |
| F-05 | Minor | `calc/src/main/java/net/otlab/calc/executor/ResultSink.java:27,31` + `ExecutorRuntime.java:123,128` | 默认装配的 `InMemoryResultSink.deliveries` 为无界 `CopyOnWriteArrayList`，每次执行都 add，生产启用后长期运行无界增长；`/status.view()` 每次还要整体拷贝该列表（O(n)）。 |
| F-06 | Minor | `calc/src/main/java/net/otlab/calc/executor/CalcPeriod.java`（`fromCode` 全仓无生产调用者）；`DualTriggerExecutor.java:247-249` | 0081 口径 `fromCode`（含 MONTH/YEAR）只在测试被调，生产路径用 `getCalcPeriod()` 秒数直算，MONTH/YEAR 日历周期从未在运行时实例化——声明的 MONTH/YEAR 能力实为未接线（受快照契约 `calcPeriod` 为 integer 秒所限）。 |
| F-07 | Minor | `calc/src/main/java/net/otlab/calc/executor/ExecutorRuntime.java:79,119` + `web/CalcStatusController.java:68` | `/calc/v1/status.executor` 暴露占位值 `tickSeconds="see-config"`（启动日志亦为字面「配置」）而非真实配置；并将内部 `lastVisitOrder`（图拓扑）与 `lastResults`（计算值）加入既有无鉴权状态口——暴露面与既有 status 同源（内网面），但占位值与口径需修正。 |

---

## 二、发现详述与实证

### F-01（Important）比较运算符反转

```java
// ArithmeticEvaluator.java:104-116
if (consume(">=")) {
    v = parseAdditive() >= v ? 1 : 0;   // ← 计算的是 right >= left
} else if (consume("<=")) {
    v = parseAdditive() <= v ? 1 : 0;   // ← 计算的是 right <= left
} else if (consume(">")) {
    double r = parseAdditive();
    v = v > r ? 1 : 0;                  // 正确：先取 r 再比较
}
```

实测（编译 `target/classes`，javac 驱动同一 `ArithmeticEvaluator.evaluate`）：

```
10>=3 = 0.0 (expect 1.0)
3>=10 = 1.0 (expect 0.0)
10<=3 = 1.0 (expect 0.0)
3<=10 = 0.0 (expect 1.0)
10>3  = 1.0 (correct)
10<3  = 0.0 (correct)
```

`>=`/`<=` 被 `FormulaExpressionValidator` 明确列入白名单（`FormulaExpressionValidator.java:117,137,194`），且 `ArithmeticEvaluator` 是 `sourceType=2` 公式项的实际生产求值器，故此 bug 从快照可解析的合法配置直达生产结果——任何含 `>=`/`<=` 的公式（含三元条件 `A>=10?1:0`）会静默产出反向结果，且不会抛异常、不触发失败隔离，属于最危险的「静默错算」。不在任何契约测试的断言面（四法汇总测试只用四则运算），故 134 测试全绿也未暴露。修法：仿 `>`/`<` 分支先存右值再比较。

### F-02（Important）ValueStore 并发数据竞争

```java
private final Map<String, ValueSample> values = new HashMap<>();  // :19
public ValueSample get(String key) { return values.get(key); }     // :22 —— 无 synchronized
public synchronized void put(String key, ValueSample sample) { ... } // :27
```

`PeriodicScheduler` 的 daemon 线程与 `RedisArrivalSubscriber` 的 daemon 线程都会进入 `DualTriggerExecutor.execute` → `computeItem`/`childSamples`，经 `store.get()` 读；订阅回调 `acceptNewValue` 与执行器落结果经 `store.put()` 写。即 `HashMap` 被跨线程同时读写：轻则可见性问题（读到旧值/中间态），重则扩容期间并发读写导致链表成环、线程 CPU 打满。`put()` 的 javadoc「synchronized 写 + 读用 volatile 语义的同步块」不成立——`get()` 既未同步也无 volatile。修法：`get()` 同步或用 `ConcurrentHashMap`；`ValueSample` 为不可变 record，无其他逃逸风险。

### F-03（Important）「保留最后正确结果」语义未兑现/未验证

TASK 任务描述与 FEAT-0013 业务规则均要求「同项同窗口幂等**保留最后正确结果**」。实际链路（`ExecutorRuntime.java:54-64`）：

```
每个到达事件：acceptNewValue(pointCode, sample)  // 总是写 store
             → executor.execute(...)              // 窗口内第 2+ 个事件在 DualTriggerExecutor:83 被 DUPLICATE 直接返回空
```

结果：第 1 个事件用其自身值完成计算并交付；后续同窗事件虽更新了 store，但不再重算、不再交付。因此**对外产出的是窗口首值的结果，而非末值的结果**——与「保留最后正确结果」字面语义相悖。契约测试 `throttleSameWindowOnlyComputesOnce`（`:2114-2125`）在首算后 `store.put("P1", 9.0, ...)` 刻意模拟「最后一条值更新」，却只断言 `duplicates==9` 与 `sink.deliveries().size()==1`，从未断言交付结果的 value 是否反映 9.0，使该验收点成为盲区。若产品口径确为「只算一次、首值为准」，需修订验收条文与报告措辞；若确为「末值为准」，需改为窗口末值触发计算（如延迟合并/覆盖交付）。此项涉及验收语义，建议由协调者对照 REQ-0020/FEAT-0013 定夺口径后裁决。

### F-04（Minor）调度器 guard 死字段与文档漂移

`PeriodicScheduler.java:29` javadoc 写「用 `WindowIdempotencyGuard` 保证同一周期窗只触发一次」，实际 `guard` 字段（`:41,53`）从未被读写，去重完全由 `lastFiredBucket`（`:116`）完成。执行者报告 §四-2 已正确说明「幂等统一交给执行器守卫」，故代码行为无误，但类注释误导维护者，且构成未使用字段。

### F-05（Minor）默认 ResultSink 无界增长

`InMemoryResultSink.deliveries`（`ResultSink.java:27`）为无界 `CopyOnWriteArrayList`，`accept()`（`:31`）每次执行都 add；`clear()` 在生产无人调用。`ExecutorRuntime` 固定装配该实现（`:123,128`），1 秒 tick + 多周期项 + 到达事件下随时间线性膨胀，且 `view()` 每次全量 `List.copyOf(deliveries)` 再取末尾，轮询越久越慢。属演进位边界，但默认实现即生产实现，建议加环形界（如保留最近 N 次）或明确登记为仅测试用途。

### F-06（Minor）MONTH/YEAR 为未接线能力

`CalcPeriod.fromCode`（含 SECOND/MINUTE/HOUR/DAY/MONTH/YEAR 与 0081 同款口径 + fail-closed）在 `calc/src/main/java` 全仓无调用者，仅 `CalcPeriodContractTest` 直接调用。生产实际路径：快照 `calcPeriod`（integer 秒，契约 `config-snapshot-schema.md:173` 定义 0=到达驱动）→ `PeriodicScheduler.tick` 用 `item.getCalcPeriod()*1000` → `DualTriggerExecutor.resultTimestamp` 用 `fromSecondsLong`（`:247-249`）。故 MONTH/YEAR 日历窗（`boundaryOf` 的 `ChronoUnit` 分支）运行时永不触发；brief 决策 1 所述「HOUR/DAY/MONTH/YEAR 口径一致」在 calc 侧只有 HOUR/DAY（以秒表示）经快照可达，MONTH/YEAR 停留于单测声明。鉴于冻结快照契约以整数秒表达周期、无法表示自然月/年，此项更宜记为口径声明与实际能力的差距（演进位），非阻断缺陷——但报告不应将 MONTH/YEAR 计入「已生效」能力。

### F-07（Minor）status 观测占位值与暴露面

`ExecutorRuntime.view()` 产出 `tickSeconds="see-config"`（`:119`）字面占位串，启动日志 `tick="配置"`（`:79`）同样未填真实 tick；实际 tick 已在 Bean 装配处约束（`CalcEngineApplication.java:96` 默认 1s）。`/calc/v1/status`（无鉴权，内网面）新增 `executor` 视图（`CalcStatusController.java:68`），含 `lastVisitOrder`（依赖图拓扑）与 `lastResults`（计算值）——与既有 status 口同源、面内暴露属 FEAT-0015 状态页设计方向，不构成新增越权，但「是否泄露内部状态」需部署层确认内网约束；占位值应改为真实配置。

---

## 三、按 brief「评审重点」逐条核对

| # | 评审重点 | 结论 |
| --- | --- | --- |
| 1 | 两通道统一引擎 / 0040 图结构复用 / ResultSink 边界 | **通过**。两通道共用 `execute(Trigger, DependencyGraph)`（非两套逻辑）；直接复用 0040 `DependencyGraph.getTopoOrder()`（Kahn、被依赖者在前）+ `GraphEdge`/`NodeKind`，`selectTargets` 周期=整图、到达=`downstreamOf` 可达子图；`ResultSink` 仅内存假实现 + 日志，全链路 grep 无 TDengine/MySQL 业务表写入，边界清晰。 |
| 2 | 周期调度（整点判据 / 结果 ts 对齐 / 启停异常 / 默认关闭） | **基本通过**。bucket 变更触发 + 结果 ts=窗右边界（`resultTimestamp`），`PeriodicSchedulerContractTest` 锁定；daemon 单线程 + `safeTick` fail-soft；`executor-enabled=false` 默认关闭，134 测试中 0040 既有 115 逐类零回归（实测确认）；`calcStartupRunner` 仅在 enabled 时 start。启停细节见 F-04（死 guard）、F-07（占位 tick）。 |
| 3 | 到达驱动（RESP2 订阅 / 同秒节流 / 有界 TTL / 多实例演进位） | **基本通过**。env 门控真 Redis live 测试实测 `Tests run:1 Failures:0 Skipped:0`（真 SUBSCRIBE→PUBLISH 五键逐字段一致），RESP2 字节级 `RespReader` 解析正确、断线指数退避、close 幂等；`WindowIdempotencyGuard` 有界 FIFO + TTL 正确（单测覆盖）。「保留最后正确结果」语义问题见 F-03。 |
| 4 | 失败隔离（quality=6 / 不发假 0 / 下游标记 / CONV-02 不冲突） | **通过**。`CalcResult.failure(sample=null)`；`markFailed` 产失败结果并入 `failed` 集，`hardDependencies` 排除 ROLLUP 边使子级缺数走 quality=7 而非失败传播；实测单测 `failureOnlyAffectsDownstreamNotOtherBranches` 与 `formulaFunctionNotSupportedIsIsolatedAsFailure` 覆盖；quality 6/7 与 CONV-02 质量码表（`tdengine-super-tables.md:55-56`）一致，无冲突。 |
| 5 | 汇总算子（四法 / quality=7 / 开关关闭 / 动态子级） | **通过**。`RollupOperator` SUM/AVG/MAX/MIN + q7/disabled/missing 排除 + 全缺打码 7 不写 0；`childSamples` 每次求值实时遍历 ROLLUP 边读 store（动态子级）；`enabled==0` 排除。四法与打码由 `rollupFourMethods` 等测试锁定。 |
| 6 | ArithmeticEvaluator 白名单 / CalcPeriod 口径一致性 | **不通过（F-01）+ 部分（F-06）**。非法表达式 fail-closed 成立（拒绝函数调用/尾随残留，非 eval 注入）；但 `>=`/`<=` 方向反转（F-01，实测）。MONTH/YEAR 生产未接线（F-06）。 |
| 7 | 并发与资源（线程生命周期 / 内存界 / 优雅停机） | **部分**。调度/订阅线程均 daemon、有 start/stop/close；`guard` 有界、TTL 生效；报告记录 8095/18041 全停实证。缺陷：`ValueStore` 并发读race（F-02）、ResultSink 无界（F-05）。优雅停机机制本身（Spring destroy 调 `ExecutorRuntime.close`）源码中未见显式 `@PreDestroy`/`destroyMethod` 声明——`ExecutorRuntime` 实现 `AutoCloseable` 且为 `@Bean`，依赖 Spring 对 `AutoCloseable`/close 的默认推断，报告「全停实证」以进程终止（kill PID）验证，线程为 daemon，故无挂死；但 Bean 生命周期的显式停机钩子缺失，记为**无法判定**是否依赖容器推断。 |
| 8 | 测试有效性（134 测试 / live 冒烟 / 红转绿） | **通过（数字与实测相符）**。本地复跑 `mvn test`：Tests=134 / Failures=0 / Errors=0 / Skipped=1，与报告一致；0040 基线 115 逐类零回归；env 门控 live 实测真 Redis 通过（本报告复跑确认）。4 处红转绿留痕见报告 §四，逻辑自洽。唯一保留：F-03 的「保留末值」断言盲区。 |
| 9 | 观测视图（泄露内部状态 / 性能） | **部分（F-07）**。暴露面与既有无鉴权 status 同源（内网面），属 FEAT-0015 设计方向；占位值与 O(n) 拷贝需修正。 |

---

## 四、契约、范围与测试核验（辅助结论）

- **越界反查**：`git diff --name-only 3cdde58..9a6cda6` 实测 23 文件、`grep -v '^calc/'` 为空；backend/frontend/ingest/契约零触碰，与 brief 声明一致，**通过**。
- **CONV-02 质量码**：0 正常 / 6 计算失败 / 7 汇总不完整 / 9 跨周期，实现取值与 `resources/contract/tdengine-super-tables.md:47-61` 唯一口径一致；CALC_ERR 不落时序行（失败仅 `CalcResult.error()`）符合口径；无冲突。
- **calcPeriod 口径**：backend 0081 `mapCalcPeriod`（`IotDataItemServiceImpl.java:1159-1184`，null/空白→0、纯数字→秒、SECOND/MINUTE/HOUR/DAY、坏码 fail-closed）与 `CalcPeriod.fromCode` 一致，但后者生产未调用（见 F-06）。
- **0029 发射侧**：`iot:newvalue:{tenant}` 五键 `{tenant,pointCode,ts,value,quality}` 解析与 live payload 一致；tenant 为空退化为通配 `iot:newvalue:*`（`RedisArrivalSubscriber.java:1287`），属实现选择，报告未特别登记但在范围内。
- **测试计数实证**：本地 `mvn test` 汇总 `Tests=134 Failures=0 Errors=0 Skipped=1`，与执行者报告逐类一致（含 1 项 env 门控 Skip）；真 Redis live `CALC_REDIS_TEST_ADDR=127.0.0.1:6379` 单跑实测 `Skipped=0`。

---

## 五、处置建议（供协调者）

1. **F-01、F-02、F-03 为 Important**，按 L1-execute §13/§12 走**退回一轮**：将本报告原样退回执行者修复（比较方向 + ValueStore 线程安全 + 「保留最后正确结果」口径实现或文档对齐并补断言），修订轮后做 scoped 复评（只看修复 diff 与原发现）。
2. **F-04～F-07 为 Minor**：记 TASK `任务结果` 的 deferred，收尾进 LOG，不进修订轮（其中 F-07 的占位值建议在退回轮顺手清理，成本极低）。
3. 验收命令数字（134 测试 / live 冒烟 / 范围）与 0040 基线零回归经独立复跑确认属实，无需因测试层面退回；退回原因聚焦上述实现缺陷与验收语义盲区。
