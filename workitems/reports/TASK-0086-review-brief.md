# 评审 brief：TASK-0086 diff 评审（代码评审）

> 实现提交 33aef05（BASE=815cabf，区间干净单提交，7 文件全 calc）——计算引擎 trial 试算端点。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0086-diff.txt`（33aef05 单提交，7 文件，861 行）
- 实施 brief + 报告：reports\TASK-0086-brief.md / TASK-0086-report.md（含 D1-D4 落地 + TDD 红转绿 + 缺数/沙箱/同口径证据）
- TASK 详情：`tasks/TASK-0086.md`（前置 TASK-0040，下游 0037/0038）
- 关联对象：FEAT-0012/REQ-0016/TC-0012 + ADR-0005 + ADR-0013 增补 5
- 复用引擎 API（只读基线，本任务不改）：`calc/src/main/java/net/otlab/calc/executor/ArithmeticEvaluator.java`、`operator/OperatorEvaluator.java`、`operator/DeltaOperator.java`、`graph/FormulaExpressionValidator.java`、`executor/ValueSample.java`
- 端点范式：`calc/web/CalcRecalculateController.java`（既有 POST 端点风格）

## 越界反查

- 提交 33aef05 恰 7 文件（TrialEngine/Request/Response + TrialController + 2 测试 + README），全落 calc/**；backend/frontend/ingest/resources/contract/calc 既有实现零触碰；区间 815cabf..HEAD 干净单提交（并行 0087 尚未提交）。

## 评审重点

1. **同引擎同口径（D2，核心）**：是否统一经 `ArithmeticEvaluator.evaluate(expr, resolver, functionResolver)` 求值、DELTA 经 `OperatorEvaluator`→`DeltaOperator` 逐点差分、手写 INTEGRAL 由 `FormulaExpressionValidator` 白名单拒绝；**核实没有另写一套求值逻辑**（grep 新代码里有无独立表达式解析/差分实现）。同 formula+variables 经 trial 与直接调引擎结果一致的测试是否有效。
2. **沙箱隔离与不落库（D3，硬约束）**：OperatorEvaluator/DeltaOperator 是否每请求/每曲线点 new（baselines 实例状态隔离）；trial 是否**无** CalcStateStore/JdbcTemplate 写口、不发指标事件、不触发调度、不写 iot_calc_state/log；HTTP 测试用 @MockitoBean 断言 upsert 零调用是否有效；连续两次试算不受前次 DELTA 基线影响的隔离测试是否真实。
3. **缺数明示不用 0（REQ-0016 验收 4）**：缺标识符 → value=null + missing=[key] + reason 明确（不是 0/不是静默）；曲线缺数点严格 ts 匹配、不沿用旧值；quality 码语义正确。
4. **端点契约（D1）**：POST /internal/v1/calc/trial、单点 atTime + window 曲线；语法/白名单拒绝返回 400；请求/响应 DTO 字段与 brief 一致；风格与既有 calc 控制器对齐。
5. **测试有效性与零回归**：calc `mvn test` 307 tests/0F/0E/3Skipped（288 基线 + 19 新增）、`mvn package -DskipTests` BUILD SUCCESS；TDD 红转绿真实、非恒真；既有 288 零回归。
6. **范围**：7 文件全 calc、越界 0、区间干净。

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0086-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。**特别对第 1 点（同引擎无另写）与第 2 点（沙箱不落库、不污染在线状态）给出明确判定**。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
