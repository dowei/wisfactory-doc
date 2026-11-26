# TASK-0047 diff 评审报告（代码评审）

- 评审者：fresh 代码评审子代理（no-subagents；只评审、只写本报告，未改动任何源码/workitems）
- 评审口径：L1-execute §13「diff 评审」；BASE `ebd443d` → 实现提交 `f19edc1`，20 文件 `calc/**`
- 证据来源：brief 指定的 diff 包 / 实施 brief / 报告 / TASK/FEAT/REQ/TC/ADR / 工作区 `calc/src/**` 源码；并独立执行 `mvn test` 核对测试数；`git show` / `git diff --name-only` 核对范围
- 未做的事：未独立复跑 live 冒烟（需起 8095 + 注入指标定义 + env 阈值，属有状态时序验证），live 结论依报告原始输出 + 契约测试佐证；详细设计 §1.9A 的 PDF 文本层不可读（`pdftotext` 输出乱码），字段「逐字一致」无法独立核验（见「无法判定」）

---

## 一、六个评审重点逐项结论

| 重点 | 结论 | 依据 / 限缩 |
| --- | --- | --- |
| 1 异步补算正确性 | **通过** | `CalcTaskManager.java:93-110` 独立 `ThreadPoolExecutor`（默认 2 workers + 有界队列 64 + 自定义拒绝处理），在线 `DualTriggerExecutor` 不共用池；`submit` 先 put task 再 `pool.execute`，taskId 生成与计算解耦，`mvn test` 中 `recalculateReturnsTaskIdImmediatelyAndProgressIsQueryable` 与 live S2（64ms/27ms）佐证立返；`topoOrder` 复用 `GraphBuilder.topoSort`（`:786-814`），契约测试乱序提交断言 M_LOW<M_HIGH<M_TOP；留痕缝 `RecalcContext(isRecalc=true)` + `recomputeWindow` 的 `calcLog.append(triggerType,...)` 逐字段由 `manualRecalcLogsTriggerTypeManualAndIsRecalcOne` 锁定。限流拒绝语义正确（但有计数瑕疵，见 F-04）。 |
| 2 停算保护 | **通过（限缩）** | `CalcProperties.calcErrorStopThreshold`（env 可配，默认 5）经 `CalcEngineApplication.calcTaskManager` 传入状态机；`CalcStateMachine.onFailure` 达阈值置 3、`onSuccess` 清零恢复、水位 `Math.max` 单调不回退，均由 6 个契约测试逐项锁定；子树隔离在 `CalcTaskManager.run` + `findStoppedUpstream` 以拓扑传递闭包实现，`consecutiveFailuresStopOnlySubtreeNotSiblings` 三轮序列与 live S5/S6（坏根停算 + M_BAD_CHILD 隔离 + M_OK2 兄弟持续正常 + 第三轮跳过）一致。**限缩**：`calc_status=2 滞后` 与滞后 env 阈值在生产链路未接线（F-01），不影响停算 3 与水位结论。 |
| 3 CalcStateStore 缝 0063 可替换性 | **明确判定：部分成立——接口边界成立，但「零改动替换」不成立**。详见 §二 F-02。 | 接口类型边界干净、无内存特有类型；但 `CalcTaskManager:87` 硬编码 `new InMemoryCalcStateStore()`（无注入/工厂），契约测试 4 例全部直接 new 内存实现，0063 必须改构造与夹具。"状态字段与 §1.9A 逐字一致"因 PDF 文本层不可读判「无法判定」（字段集合与 REQ-0023 叙述一致）。 |
| 4 管理面入口最小化（ADR-0010） | **通过** | diff 中新增路由仅 `/calc/v1/recalculate`、`/calc/v1/task/{taskId}`，status 并入既有 `CalcStatusController` 的 `calcState` 段，无 `/internal/v1/calc/**`、无鉴权、无前端改动；`CalcRecalculateController` / `CalcStatusController` / `CalcStatusVo` / `CalcTaskProgressVo` 四处 javadoc 明确登记 0061 权限码 + 审计 + 页面接线义务。与 ADR-0010 TD-0024「内部接口一期、管理面入口最小化」一致。 |
| 5 测试有效性 | **通过** | 独立执行 `mvn test`，surefire 聚合 **265 run / 0 failures / 0 errors / 3 skipped**，与报告 §验收 1 逐类数值一致（244 基线 + 21 新增；3 skip 为既有 env 门控）。5 处红转绿留痕（编译期 / 2 真实断言 / Spring 撞映射 / 端口争用 / live 拓扑偏序）均有「为什么现在对了」且与 diff 终态吻合：isRecalc 断言已改 `true`、三轮停算序列已落测试、status 已并段、测试端口 `:18095`（本次实测 Tomcat 起于 18095）、拓扑断言改偏序。live fail=0 序列与断言对应，未独立复跑（见上文）。 |
| 6 零回归与范围 | **通过** | `git show --name-only f19edc1` = 20 文件全 `calc/**`；`git diff --name-only ebd443d..f19edc1` = 20 行，越界 0；区间仅一提交。244 基线逐类与 0046 修订轮报告一致（仅 CalcPropertiesTest 10→11）。并行 0053 的 backend 改动未入提交（工作区存在但本任务未 add）。 |

---

## 二、发现清单

| 严重级 | 出处（绝对路径:行） | 一句话说明 |
| --- | --- | --- |
| **Important** | `D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\CalcEngineApplication.java:246-276`；`D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\state\CalcStateMachine.java:94-113`；`D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\state\CalcStatusVo.java:20-24`；`D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\web\CalcStatusController.java:105,109` | F-01 滞后阈值 env 读入后仅用于 log、未传入状态机（实际经 4 参构造写死 5 分钟），且 `evaluateLag`（calc_status=2）在生产链路零调用，`CalcStatusVo.from` 的 `lagThresholdMs` 入参被忽略——calc_status=2 永不产生、`CALC_CALC_LAG_THRESHOLD_MS` 是空开关。 |
| **Important** | `D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\task\CalcTaskManager.java:87`；`D:\workspace\WisFactory\calc\src\test\java\net\otlab\calc\state\CalcStateStoreContractTest.java:21,45,64,75` | F-02 接口边界清晰但「0063 换 JDBC 零改动替换」不成立：内存实现在 `CalcTaskManager` 构造中硬编码（无注入/工厂），契约测试 4 例全部直接 new 内存实现（非可重跑的抽象契约）。 |
| **Important** | `D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\CalcEngineApplication.java:256`；`D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\task\CalcTaskManager.java:201`；`D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\web\CalcStatusController.java:101` | F-03 生产 lambda 把 `recomputeWindow` 租户写死为字面量 `"t47-live"`（live 冒烟值泄漏），task/状态页 `tenantId` 固定 1L——请求 tenant 被忽略，REQ-0023 租户分区隔离在生产路径不成立；报告 §五-6 只登记了 1L，未登记该硬编码。 |
| Minor | `D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\task\CalcTaskManager.java:106-108,159` | F-04 队列满拒绝时 handler 与 submit 的 catch 各执行一次 `rejected.incrementAndGet()`，拒绝计数翻倍；契约测试只断言 `>=1` 故未暴露。 |
| Minor | `D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\task\CalcTaskManager.java:808-814` | F-05 `topoOrder` 捕获环异常后返回提交序，注释称 fail-closed 实为 fail-open：依赖成环时绕过「下级先算」，与仍按 dependencies 执行的子树隔离不一致（低概率配置错误路径）。 |

### 发现详述

**F-01（Important，滞后状态生产链路空接线）**
`calcLagThresholdMs` 在 `CalcProperties` 与 `application.yml` 声明（env `CALC_CALC_LAG_THRESHOLD_MS`，默认 900000ms），在 `calcTaskManager()` 中读入局部变量 `lagMs`，但唯一用途是 `if (lagMs != 15min) log.info(...)`；构造 manager 用的是 4 参重载 `new CalcTaskManager(fn, threshold, workers, capacity)`，该重载把状态机滞后阈值写死为 `5L*60L*1000L`（5min）。全仓 `grep evaluateLag` 在 `calc/src/main` 仅命中方法定义本身，无任何生产调用；`CalcStatusVo.from(s, lagThresholdMs, now)` 的 `lagThresholdMs` 形参在函数体内从未引用（lag 仅 `now - watermark`）。后果：§决策3 / 报告 §1.1 声称的「calc_status 2 滞后」在状态页与状态存储中永不出现（只有 1/3），运维拿不到滞后状态判定，env 阈值为空配置。建议：去掉误导性 env 或把 `evaluateLag` 接入状态维护路径并将 env 真正传入状态机，同时移除 `CalcStatusVo.from` 的死参数。

**F-02（Important，重点 3 明确判定）**
分两层判定：
- **接口层成立**：`CalcStateStore`（`get/upsert/byTenant/all`）只暴露 `CalcState`（全为基本类型 + String 的不可变值对象）与 `List`，无 ConcurrentHashMap/内存特有类型外泄；`CalcTaskManager.stateStore` 与 `CalcStateMachine.store` 字段类型均为接口，`CalcStatusController` 经 `stateStore()` getter 读接口——消费侧无改动。
- **「零改动替换」不成立**：① `CalcTaskManager.java:87` 在构造函数内 `new InMemoryCalcStateStore()`，无构造注入、setter 或工厂位；`CalcEngineApplication.calcTaskManager()` 又是在 bean 方法里 `new CalcTaskManager(...)`，与 `CalcStateStore` 无 DI 关系。0063 换 JDBC 必须修改 `CalcTaskManager`（新增接收 `CalcStateStore` 的构造/工厂）并改装配处，不是「换实现零改动」。② `CalcStateStoreContractTest` 4 个用例全部 `new InMemoryCalcStateStore()`（`:21,:45,:64,:75` 及 `:30,:37,:49,:61` 等），报告 §三-1 / §四补验条件声称「0063 跑同套契约测试对 JDBC 实现复验」——该套件并未参数化，无法不改一行就对 JDBC 实现复跑，`assertInstanceOf(CalcStateStore.class, store)` 是恒真断言。③ `CalcStateMachine` 用 `System.currentTimeMillis()` 生成 `updateTime`，与 0063 DDL 的 `ON UPDATE CURRENT_TIMESTAMP` 语义可能冲突（次要）。
- **结论：明确判定为「接口抽象到位、但零改动可替换性主张不成立」**——正确表述应为「0063 需新增实现 + 在 CalcTaskManager/装配处注入 + 参数化契约测试」，属小改动但非文档所称零改动。该实现差距本身不阻塞本任务三条验收命令，但作为被重点点名的替换缝须纠正声明或补齐注入位。

**F-03（Important，租户维度硬编码）**
`CalcEngineApplication.calcTaskManager()` 的计算 lambda 调用 `metricRuntime.recomputeWindow("t47-live", ...)`——这是 live 冒烟用租户字面量，进入生产代码后无论 recalculate 请求体传什么 tenant，指标都在 `"t47-live"` 名下计算并写 log；`CalcTaskManager.run()` 的 `long tenantId = 1L` 与 `CalcStatusController.status()` 的 `long tenantId = 1L` 使状态存储键恒为 `(1, fullKey)`。结果：请求中的 `tenant` 仅存进 `CalcTask.tenant` 快照，不参与计算与状态；按 REQ-0023 核心规则「租户 A 补算风暴不影响租户 B」在生产路径无法成立（内存 store 的 `tenant_id` 列永远是 1）。报告 §五-6 已把「tenant_id 暂固定 1L」登记为演进位，但 `"t47-live"` 硬编码未登记，且它会使 live 之外的任何真实租户计算串到测试租户。建议把 tenant 从 submit 上下文贯穿到 `recomputeWindow`，并移除字面量。

**F-04（Minor）** 有界队列拒绝路径：`start()` 的拒绝 handler 先 `rejected.incrementAndGet()` 再抛 `RejectedExecutionException`，`submit()` 的 `catch` 再 `rejected.incrementAndGet()`，同一次拒绝被计 2 次。`boundedQueueRejectsBacklogWithoutBlockingOnlineExecutor` 只断言 `rejectedCount() >= 1`，未锁定精确值。

**F-05（Minor）** `topoOrder` 在 `GraphBuilder.topoSort` 抛异常（环/内部矛盾）时 `return new ArrayList<>(fullKeys)`（提交序）。注释写「fail-closed」，实际是绕过依赖顺序执行（fail-open）；同时 `findStoppedUpstream` 仍按 `dependencies` 做带 `seen` 的闭包遍历，二者对同一依赖图的处理不一致。仅依赖配置成环时触发。

---

## 三、无法判定

1. **状态字段与 §1.9A「逐字一致」**：指定详细设计 PDF 的文本层不可读（`pdftotext` 输出为乱码，关键字 `iot_calc_state` / `calc_status` / `1.9A` 命中 0），无法做逐字字段/类型比对。`CalcState` 字段集合（tenant_id / full_key / last_calc_time / watermark / calc_status 1-2-3 / error_count / last_error / update_time）与 REQ-0023 正文叙述一致，但「逐字一致」主张**无法判定**。
2. **live 冒烟全序列**：本评审未独立起服务复跑（有状态、依赖 8095 与注入缝），依报告原始输出 + 契约测试佐证，判「报告自证充分、未独立验证」，不作为发现。
3. **TDengine 真机补验（TD-0053）**：环境仍故障，报告按条件登记为演进位，本评审不判定。

---

## 四、结论

**不通过（按 §13 走修订/复审路径）**——三条验收命令独立核对均通过（265/0/0/3、范围 20 文件全 `calc/**`、diff 与报告一致），重点 1/4/5/6 通过、重点 2 通过（限缩），但存在 **3 项 Important**：
- F-01 滞后状态（calc_status=2）与 env 阈值在生产链路空接线；
- F-02 重点 3 的「CalcStateStore 零改动可替换」明确判定为不成立（接口层成立、装配与测试夹具层不成立），须纠正声明或补注入位/参数化契约；
- F-03 生产代码把租户写死为 `"t47-live"` 且 tenant_id 固定 1L，请求 tenant 被忽略。

两项 Minor（F-04 拒绝计数翻倍、F-05 环回退 fail-open）建议顺带修。修复/纠偏上述 Important 后，本 diff 的核心实现与测试证据足以支持通过；0061/0063 接线义务与 TDengine 真机补验按报告登记，不构成本次阻断。TASK 状态保持「进行中」由协调者处置；本评审未修改任何源码或 workitems（仅写本报告）。
