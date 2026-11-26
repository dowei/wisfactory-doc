# 评审报告：TASK-0086 diff 评审（代码评审）

> 角色：独立代码评审（fresh、no-subagents、只评审不修改）。口径：L1-execute §13「diff 评审」。
> 结论：**通过（无 Critical / 无 Important）**；3 条 Minor 记 deferred，不进修订轮。

## 0. 一句话结论

7 文件全落 calc、单提交越界 0；**同引擎无另写求值/差分逻辑**、**沙箱每请求 new 且不落库/不污染在线状态**两项头号重点经源码独立核实成立；缺数明示不用 0、端点契约与测试均闭合；评审通过。

## 1. 评审输入与独立性

| 项 | 内容 |
| --- | --- |
| 评审对象 | 提交 `33aef05c0375bbcc84c1dc1cc9f989cb88ff03c3`（单提交），BASE `815cabf` |
| 输入 | `reports/TASK-0086-review-brief.md` + `reports/TASK-0086-diff.txt` + `reports/TASK-0086-brief.md` + `reports/TASK-0086-report.md` + `tasks/TASK-0086.md` + REQ-0016 / TC-0012 / ADR-0013 / ADR-0005 |
| 独立核实方式 | 回到 `D:\workspace\WisFactory\calc` 读新代码与复用引擎源码；grep 独立解析/差分痕迹；独立重跑 `mvn test` / `mvn package`；核对 Bean 装配与 auth 配置。**未采信报告自述作为结论依据** |
| 严重级口径 | Critical / Important → 退回一轮（上限 1 退回 + 1 复评）；Minor → deferred（收尾进 LOG，不进修订轮） |

## 2. 越界反查与范围

- `git diff --name-status 815cabf..HEAD`：恰 7 文件，`M calc/README.md` + `A` 6 文件（TrialEngine/Request/Response/TrialController/2 测试），全落 calc/**，与 brief 改动范围声明一致。
- `git log`：`815cabf..HEAD` 干净单提交（HEAD=`33aef05`）；无第二提交、无 merge。
- 工作区当前有并行未提交改动（`OpenApiQueryService.java` / `OpenApiQueryServiceImpl.java` / `displayread/` / `.tmp-t0087-acceptance.log` 等），均不在本提交内，非本任务范围；本评审不计为越界。
- **判定：范围合规，越界 0。**

## 3. 头号重点判定（明确结论）

### 3.1 重点 1 · 同引擎同口径——**成立，未另写一套求值/差分**

独立核实（读源码，不只信报告）：

- 求值唯一入口：`TrialEngine.java:79`（单点）与 `:102`（曲线点）均为 `ArithmeticEvaluator.evaluate(req.formula(), ctx::resolve, ctx::callFunction)`，与复用基线 `ArithmeticEvaluator.java:51` 的 `evaluate(expr, Resolver, FunctionResolver)` 为同一方法。四则/括号/比较/逻辑/三元全部由该求值器解析，TrialEngine 无 `parseAdditive/parseTernary` 等解析方法。
- DELTA 唯一差分口：`TrialEngine.java:236` 调 `operators.delta(deltaKey, new ValueSample(...), null, null, false)` → `OperatorEvaluator.java:36` → `DeltaOperator.evaluate(...)`。TrialEngine **无 `new DeltaOperator()`**、无手写 `current - prev`、无基线 Map（`grep baselines` 仅命中复用引擎）。
- 白名单先闸门：`TrialEngine.java:63-64` 调 `FormulaExpressionValidator.validate(...)`，失败 → `reject(...)` → HTTP 400；手写 INTEGRAL/未知函数被 `FormulaExpressionValidator.java:24-27, 156-161` 的白名单/BANNED 拒绝。
- grep 新代码全部 `Pattern`/`Matcher` 仅一处：`TrialEngine.java:44, 251` 的 `DELTA_ARG` 正则 + `extractDeltaKey()`，只提取 DELTA 单参 key（因 FunctionResolver 只给已求值 double、不给标识符名，必须绑定 history key）；**仅识别参数名，不求值、不差分**，与真算 `DualTriggerExecutor.evaluateDeltaFormula` 用同形态正则识别 `DELTA(key)` 对齐，不构成第二套逻辑。
- 同口径测试有效：`TrialEngineContractTest.java:194-201`（`sameEngineParityWithDirectArithmeticEvaluator`）同一 `A*2+B` 经 trial 与直接调 `ArithmeticEvaluator` 相等（1e-9），非恒真断言；`:168-176`（`DELTA(METER)/2`=3）证明函数结果共享求值器。

**判定：同引擎同口径成立——未发现另写的表达式解析或差分实现。**

### 3.2 重点 2 · 沙箱隔离与不落库——**成立**

独立核实：

- 每请求/每点 new：`TrialEngine.java:77`（单点）与 `:100`（每曲线点）均 `new EvalContext(..., operatorFactory.get(), ...)`，默认工厂 `:92` `OperatorEvaluator::new`；`OperatorEvaluator` 内部 `delta = new DeltaOperator()`（`OperatorEvaluator.java:23`），baselines 是实例状态（`DeltaOperator.java:47`），因此每求值新建、跨请求/跨点不共享。
- 注入面干净：`TrialController` 仅依赖 `TrialEngine`；`TrialEngine` 字段仅 `Supplier<OperatorEvaluator>`（`:48`）。无 CalcStateStore / JdbcTemplate / MetricEventPublisher / 调度器 / Redis / 真库查询引用；grep 新代码无 `state`/`event`/`schedule`/`jdbc` 写口。
- @MockitoBean 断言有效：`TrialControllerHttpTest.java:48-49` `@MockitoBean CalcStateStore calcStateStore` 替换真实 Bean；生产 Bean 由 `CalcEngineApplication.java:235` `@Bean calcStateStore(...)` 无条件注册（默认 InMemoryCalcStateStore），故 mock 命中真实Bean 缝而非空气。`:94` `verify(calcStateStore, never()).upsert(any(CalcState.class))` 在发 DELTA 请求后断言零调用，测试实测通过（独立重跑 5/5 PASS），**断言有效**。
- 连续两次隔离真实：`TrialEngineContractTest.java:178-189` 注入计数 factory，两次同请求均 value=5.0 且 `created.get()==2`；`TrialControllerHttpTest.sandboxRepeatsIdenticallyOverHttp` 连发两次 body 相等。测试非恒真（若共享基线，第二次基线会污染），有效。

**判定：沙箱每请求 new、不落库、不发事件、不污染在线 baselines——成立。**

## 4. 其余评审重点

**重点 3 · 缺数明示不用 0（REQ-0016 验收 4）——通过。**
`TrialEngine.java:81/125`（单点）与 `:106/111`（曲线点）缺数走 `TrialResponse.failure(...)`（`TrialResponse.java:55-56`）→ value=null、quality=6（`:42` `QUALITY_MISSING`）、missing 列具体 key、reason 含 key，非 0/非静默；语法拒绝单独 `reject()`（HTTP 400）与业务缺数（HTTP 200）严格区分。曲线点 `snapshotAt` 用 `s.ts() == t`（`:133` strict），缺数段不沿用旧值；单点 `atTime` 用 `s.ts() <= t` 取最新（`:133`），两种语义分别测试锁定，合理。质量码 6 = CONV-02 计算失败，语义正确。测试 `missingIdentifierReportsNullValueAndNamedKeyNeverZero`（`TrialEngineContractTest.java:45-52`）、HTTP `trialMissingIsExplicitNotZero`、窗口曲线缺数段断言均非空且非恒真。

**重点 4 · 端点契约（D1）——通过。**
`TrialController.java:42` `@PostMapping("/internal/v1/calc/trial")`，`@RestController` + `ResponseEntity<Map<String,Object>>`（`:43-46`），风格与 `CalcRecalculateController.java:41/54-55` 对齐；rejected→400（`:46`），其余 200。DTO 字段 formula/variables/history/atTime/window 与响应 value/quality/missing/reason/curve/engineVersion 与 brief §3.1 一致；`engineVersion` 响应字段声明（`TrialResponse.java:38`）。无路由冲突（grep 唯一）。

**重点 5 · 测试有效性与零回归——通过（独立重跑）。**
独立执行 `cd calc && mvn test` 得 `Tests run: 307, Failures: 0, Errors: 0, Skipped: 3` + `BUILD SUCCESS`；`mvn package -DskipTests` 得 `BUILD SUCCESS`。新增 19（TrialEngineContractTest 14 + TrialControllerHttpTest 5），既有 288 零回归。TDD 红转绿留痕见报告 §4：红因「无实现」而非改期望；绿 1 三处修复（history key 视为已提供 / 曲线严格 ts / 复合表达式构造）均有根因，断言非恒真。

**重点 6 · 范围——通过。** 见 §2。

## 5. 发现清单

| # | 严重级 | 出处 | 一句话说明 |
| --- | --- | --- | --- |
| F-1 | Minor(deferred) | `TrialEngine.java:93-97, 193-251`（evaluateCurve + callFunction 循环） | 窗口曲线每点对 DELTA 从 history 全量重建基线循环求值，O(点数²)；本批为试算端点、数据量有限，正确性不受影响，仅性能演进位。 |
| F-2 | Minor(deferred) | `TrialEngine.java:236`（`operators.delta(key, sample, null, null, false)`） | trial 调 DELTA 固定传 `expectedPeriodMs=null`，故真算的「跨周期打码 9」口径在 trial 不触发；不等于同口径的 DELTA 差分主路径（正常/回退/坏样本/换表/溢出均经原生算子），但跨周期 anomaly 场景试算结果与真算可能有差异，建议后续由调用方传入期望周期补齐。 |

## 6. 判定汇总

| 评审项 | 判定 |
| --- | --- |
| 头号重点 1：同引擎同口径、无另写求值/差分 | **成立，无另写** |
| 头号重点 2：沙箱每请求 new、不落库、不污染在线状态、@MockitoBean 零调用断言有效 | **成立，断言有效** |
| 重点 3：缺数明示不用 0、严格 ts、quality 码 | 通过 |
| 重点 4：端点契约与风格 | 通过 |
| 重点 5：测试有效性与零回归（独立重跑 307/0/0/3） | 通过 |
| 重点 6：范围（7 文件全 calc、越界 0、区间干净） | 通过 |
| 严重级汇总 | Critical 0 / Important 0 / Minor 2（deferred） |

**最终结论：通过（无 C/I，不触发退回；2 条 Minor 记 deferred 收尾进 LOG）。**
