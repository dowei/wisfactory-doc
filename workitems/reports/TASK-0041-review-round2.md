# TASK-0041 scoped 复评 round2 报告（代码评审）

- 评审对象：修复提交 `73b8587`（区间 `9a6cda6..73b8587` 限 `calc/`，单提交 12 文件；并行 `4d4012e` 为 TASK-0083 backend，已隔离剔除）
- 评审口径：L1-execute §13「diff 评审」复评条款——只看修复 diff 与原发现 F-01/F-02/F-03（含 F-04~F-07 处置），不重开原评审已通过部分；修订上限 1 退回+1 复评，本轮终审
- 评审角色：fresh 代码评审——仅依据 brief 必读文档与工作区源码判断，未向协调者追问
- 评审日期：2026-11-24
- 状态核对：工作区 HEAD=`73b8587`（=交付提交），`git status` 无 `calc/` 改动
- 独立复核手段：通读 12 文件修复 diff 与对应工作区源码；`grep` 全部调用点；独立复跑 `cd calc && mvn test`

---

## 结论

**通过**（三问全过，无新增 Critical/Important）。

F-01（比较方向）、F-02（ValueStore 线程安全）、F-03（同窗保留末值 + 幂等 + 覆盖交付）三项 Important 均真修复，红绿证据成立；F-04 死字段清理零波及，F-06 边界登记诚实准确；独立复跑 140 测试 0F0E1Skip（134 基线+6），0040 图结构零影响。新发现 4 项均为 **Minor**（文档漂移/边界登记，非阻断），不足以再退回。

---

## 一、复评三问逐条

### 问 1：F-01/F-02 是否真修复 —— **真修复（两项均通过）**

**1.1 F-01 `>=`/`<=` 比较方向。结论：真修复。**
- 出处：`calc/src/main/java/net/otlab/calc/executor/ArithmeticEvaluator.java:103-123`（diff 第 105-116 行）；测试 `calc/src/test/java/net/otlab/calc/executor/ArithmeticEvaluatorContractTest.java:25-58`；报告 R1.1 红证据。
- 一句话说明：已改为先存右值 `double r = parseAdditive()` 再按「左 OP 右」求值（`v >= r` / `v <= r`），`10>=3=1.0`、`3<=10=1.0`、三元分支断言成立；`>=`、`<=`、三元 3 项对旧实现必红（报告 R1.1 三条失败实证），`>`/`<` 1 项是回归守卫（旧实现本就正确），我独立复跑 4/4 绿。

**1.2 F-02 ValueStore 并发。结论：真修复。**
- 出处：`calc/src/main/java/net/otlab/calc/executor/ValueStore.java:1-50`（diff 对应段）；测试 `calc/src/test/java/net/otlab/calc/executor/ValueStoreConcurrencyContractTest.java:31-106`；报告 R1.2。
- 一句话说明：`HashMap` → `ConcurrentHashMap`，去掉误导的 `synchronized`，javadoc 与实现一致并点明「调度线程与订阅线程并发读写」场景；结构守卫（反射断言 `get` 同步或底层 `ConcurrentMap`，旧实现必红）+ 4 写×4 读×500 真并发压力（`CountDownLatch` 门控、2000 值全可见、无异常）双测均绿，我独立复跑 2/2 通过。

### 问 2：F-03 是否真修复 —— **真修复（通过）**

- 出处：`WindowIdempotencyGuard.java:23-31,52-73`；`ResultSink.java:25-70`；`DualTriggerExecutor.java:82-160`；`ExecutorRuntime.java:55-72,115-118`；测试 `DualTriggerExecutorContractTest.java:93-139`；报告 R2.2。
- 三机制逐项核对：
  - **GUARD DUPLICATE→REPLACE**：`Decision` 枚举仅存 `FIRST`/`REPLACE`，`tryAcquire` 同窗返回 `REPLACE`（:57-73），仍受 `maxEntries`/TTL 有界约束。成立。
  - **ResultSink windowKey 覆盖交付**：新增 `accept(String,Trigger,List)`，`Delivery` 增 `windowKey` 字段；`InMemoryResultSink` 覆写为按 windowKey `deliveries.set(i, fresh)` 原地替换、无匹配才 `add`（:45-55），同窗不追加。成立。
  - **执行器每次到达重算覆盖**：`execute` 去早退，`guard.tryAcquire` 结果仅记录日志，FIRST/REPLACE 均用最新 store 重算并按 windowKey 交付（:84-88,152-156）。成立。
- **最终交付值是否为同窗末值**：是。单测同秒 10 值 `[1..8,3,9]` 末值 9 → `A.V=P1+1=10.0`（断言非首值 2.0），我独立复跑通过；报告 R2.2 live 9 条末值 9 → `A.V=P1*2=18.0`、`deliveries=1`，与代码路径一致。
  - 精确说明：brief 括注「单测 9.0→A.V=18.0」与实测不符——单测公式为 `P1+1`（末值 10.0），18.0 属 live 公式 `P1*2`；两者都等于「末值×系数」，不影响末值语义结论。
  - live 数字取自报告 R2.2 原始输出，**未独立复跑**（需外部 Redis + 假后端，复评契约限定只读判断）；末值机制已由单测与源码独立闭合。
- **幂等是否仍成立**：成立。`sink.deliveries().size()==1`（单测断言 + 我复跑）；中间值仅重算覆盖、不构成独立「产出」；对外同窗只有一次最终交付。`throttled` 语义由「丢弃」改为「同窗合并重算次数」并已 javadoc 登记（:115），仅计数、无行为影响。
- **ResultSink 覆盖交付是否引入新语义问题**：同 windowKey 旧结果被原地替换、无同窗旧结果残留；不同 windowKey 各自一条交付，符合预期。发现一处边缘并发竞态（见新发现 Minor-4）与三处文档漂移（Minor-1/2/3），均非阻断。

### 问 3：修复 diff 是否引入新 Critical/Important —— **无新增 C/I（通过）**

- **F-04 死字段清理是否波及其他调用点**：无波及。`PeriodicScheduler` 构造器由 4 参改 3 参，全部 3 个调用点（`ExecutorRuntime.java:49`、`PeriodicSchedulerContractTest.java:42,71`）已同步；`grep` 无 4 参残留，无 `Decision.DUPLICATE` 残留（`DUPLICATE_NODE` 为图构建无关项）。通过。
- **F-06 MONTH/YEAR 边界说明是否误导**：不误导，诚实登记。`CalcPeriod.java:62-71` 与 `PeriodicScheduler.java:31-33` javadoc 明示「快照 `calcPeriod` 为整数秒、MONTH/YEAR 日历窗生产未接线（deferred）、HOUR/DAY 以秒经快照可达」；核实 `fromCode` 在 `calc/src/main` 无生产调用者、`DataItem.calcPeriod` 为 `Long` 秒、契约 `config-snapshot-schema.md:173` 为 `integer` 秒，说明准确。通过。
- **140 测试零回归（134 基线+6）**：我独立复跑 `cd calc && mvn test` → `Tests=140, Failures=0, Errors=0, Skipped=1`（env 门控 live 测试），`BUILD SUCCESS`；134 基线逐类零回归，新增 6 项（F-01 4 + F-02 2）+ F-03 既有项强化。通过。
- **并发改动对 0040 图结构零影响**：零影响。`git show --name-only --format="" 73b8587` 为 12 文件、越界计数 0，全落 `executor/` 主码与测试，未触碰 `graph/` 包；执行器仍按原 `getTopoOrder()`/`node()`/`getEdges()` 使用 0040 图；0040 基线逐类（CycleDetector/FormulaExpressionValidator/GraphBuilder/SnapshotReconciler/HttpSnapshotFetcher/RawJsonExtractor/SnapshotParser）在我复跑中全绿。通过。

---

## 二、新发现分级

| 编号 | 级别 | 出处 | 说明 |
| --- | --- | --- | --- |
| Minor-1 | Minor | `WindowIdempotencyGuard.java:10-11` | 类级 javadoc 仍写「同一 windowKey 只允许第一次触发执行，后续到达判为重复（节流合并 + 幂等——只产出一次结果）」，与修复后 REPLACE 语义（后续到达重算替换）矛盾；方法级（:52-56）与枚举（:23-31）已更新，类级漏改。非阻断文档漂移。 |
| Minor-2 | Minor | `DualTriggerExecutor.java:80` | `execute` 的 `@return` 仍写「幂等重复返回空列表」，修复后同窗重复返回重算的非空结果（:84-88 无早退，:152-159 均交付）。非阻断文档漂移。 |
| Minor-3 | Minor | `ResultSink.java:28` | 默认方法 javadoc 称「默认实现为替换」，但接口 default `accept(String,...)`（:31-33）实际委派到无键 `accept`（内存实现为 append）；真正替换仅在 `InMemoryResultSink` 覆写（:45-55）。对后续 Sink 实现者有轻微误导；当前唯一实现行为正确。非阻断。 |
| Minor-4 | Minor | `ResultSink.java:45-55` | `accept(String,...)` 为「扫描—set/add」读改写、非原子。若调度线程与订阅线程对同一 windowKey 并发交付（需某数据项同时配 `calcPeriod>0` 且收到到达事件、窗键对齐），首条交付写入前两线程可能各加一条，出现同窗 2 条交付。单通道内（订阅回调串行、调度单线程）不触发；当前仅内存假实现受影响，真实 Sink 为演进位。非阻断，登记即可。 |

无 Critical、无 Important。

---

## 三、契约执行声明

- **fresh**：仅依据 brief「必读」文档与工作区源码判断，未向协调者追问；无法独立复跑的 live 端到端数字已明确标注来源（报告 R2.2）而未标「通过实证」。
- **no-subagents**：未派任何 subagent。
- **只评审不修改**：未修改任何文件；独立 `mvn test` 仅生成 gitignore 的 `target/` 构建产物；本报告为唯一产出。
