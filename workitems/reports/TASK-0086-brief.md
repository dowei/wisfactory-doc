# 实施 brief：TASK-0086（计算引擎 trial 试算端点）

> 标准任务（calc 独立 Maven 项目）。BASE=815cabf。一个执行者做完整、统一测一次。前置 0040✅。为 0037/0038 解开"转发目标不存在"死锁。

## 批次

- TASK-0086（本批 1 个）
- 验收粒度：一个 subagent 跑一次整体验收（calc mvn test）
- BASE：`815cabf`
- 改动范围（文件/目录级，越界反查用）：
  - `calc/src/main/java/net/otlab/calc/web/TrialController.java`（新建端点）
  - `calc/src/main/java/net/otlab/calc/trial/**`（新建试算域：请求/响应 DTO + 试算服务 + 沙箱求值）
  - `calc/src/test/java/net/otlab/calc/trial/**`
  - `calc/README.md`（可选，端点说明）
  - **不得触碰**：backend/**、frontend/**、ingest/**、resources/**（业务资源）、resources/contract/**（冻结）、calc 既有 executor/operator/graph/metric/status/state/web 控制器实现（只读复用其 API，不改）

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0086.md`
- 关联对象：`features/FEAT-0012.md`、`requirements/REQ-0016.md`、`testcases/TC-0012.md`、`decisions/ADR-0005.md`、`ADR-0013 增补 5`（窗口曲线试算）
- 下游消费：`tasks/TASK-0037.md`（validate/preview 转发 trial）、`TASK-0038.md`（窗口曲线增强，本端点是其承载体）；`reports/TASK-0037`尚不存在——0037 未派发
- 可复用引擎 API（只读复用，不修改）：
  - `calc/src/main/java/net/otlab/calc/executor/ArithmeticEvaluator.java`（`evaluate(expr, Resolver)` / `evaluate(expr, Resolver, FunctionResolver)`；四则/比较/逻辑/三元；函数调用经 FunctionResolver 注入）
  - `calc/src/main/java/net/otlab/calc/operator/OperatorEvaluator.java`（`supports("DELTA")` / `delta(key, ValueSample, expectedPeriodMs, maxRange, paramChangeEvent)`；INTEGRAL 仅预置派生不开放手写）
  - `calc/src/main/java/net/otlab/calc/operator/DeltaOperator.java`（`evaluate(Input)` → Result(value, quality, kind, reason, ts)；五异常口径）
  - `calc/src/main/java/net/otlab/calc/executor/ValueSample.java`（value/quality/ts 三键）
  - `calc/src/main/java/net/otlab/calc/graph/FormulaExpressionValidator.java`（语法校验，可复用给 validate）
- 端点范式：`calc/src/main/java/net/otlab/calc/web/CalcRecalculateController.java`（POST /calc/v1/recalculate + ResponseEntity<Map>）、`MetricObserveController.java`（observe/recalc 请求响应风格）
- calc 端口 8095（application.yml），独立 Spring Boot 应用

## 接口与决策（逐条结论，无选项）

### D1 · 端点形态
- `POST /internal/v1/calc/trial`（内部端点，calc 8095；与管理面 preview 同请求响应规范，详细设计 §3.1）。
- 请求体（JSON）至少支持：
  - `formula` string（表达式，如 `DELTA(F01.WS01.EQ001.M01)` 或 `A*2+B`）
  - `variables` map<string, number|sample>（标识符 → 值/样本；RD-缺数语义：未提供的标识符 = 缺数）
  - `history` 可选 map<string, list<sample>>（DELTA/INTEGRAL 取一窗历史的输入；sample = {value, quality, ts}）
  - `atTime` 可选 long（指定时刻，preview 用；不落库）
  - `window` 可选对象 {start, end, stepMs}（窗口曲线试算，0038 用；本任务先支持基础曲线点集输出）
- 响应体（JSON）：
  - `value` number|null（单点结果；缺数时 null 不用 0）
  - `quality` int（沿用质量码语义；成功 0、缺数/失败按既有码）
  - `missing` list<string>（**缺数标识符清单，明确返回，REQ-0016 验收 4**）
  - `reason` string（失败/缺数原因说明）
  - `curve` 可选 list<{ts, value, quality, missing}>（窗口试算时逐点输出；缺数段每点明示）
  - 同口径声明：响应头或字段标注 `engineVersion` 或与真算一致的口径标识（报告说明选择）

### D2 · 同引擎求值（核心，禁止另写一套）
- 单点公式：`ArithmeticEvaluator.evaluate(formula, resolver, functionResolver)`。
  - resolver：从 `variables` 取值，缺失返回 null → 求值器抛异常 → 捕获后转成"缺数"响应（missing 含标识符、value=null、reason 明确），**绝不返回 0 冒充**。
  - functionResolver：`supports` 交给 `OperatorEvaluator`；DELTA 用窗口历史按时间顺序构造 ValueSample 逐点调用 `delta(...)` 得到差分；INTEGRAL 不开放手写（白名单拒绝，与 FormulaExpressionValidator 一致）。
- 语法校验：可复用 `FormulaExpressionValidator` 对 formula 先 validate，语法/不支持函数错误转成 400 类响应（与管理面 validate 口径对齐）。
- **窗口曲线**：按 window {start,end,stepMs} 逐点求值（每点用该时刻前的历史窗），产出 curve 点集；每点独立缺数标注；**沙箱**——DELTA 基线等算子状态每请求新建实例（不共享真算的跨周期 baselines），保证试算不污染在线计算。

### D3 · 沙箱隔离与不落库（硬约束）
- trial 服务**不注入**任何 CalcStateStore/JdbcTemplate 写口、不发指标产出事件、不触发调度、不写 iot_calc_state/log。
- DELTA/INTEGRAL 算子实例每请求 new（OperatorEvaluator 内部 baselines 是实例状态）——试算的基线建立只在本次请求生命周期内，跨请求/跨试算互不干扰，更不影响在线算子的 baselines。
- 只读复用：允许读 `variables`/`history` 入参（调用方提供），**trial 端点自己去查真库/Redis 不在本任务范围**（0037 管理面负责取数后传入；0038 增强才考虑从历史存储取）。

### D4 · 测试口径（TDD 红转绿）
- 单点：四则/括号/三元求值正确；缺标识符 → 响应 missing 含该 key、value=null、reason 明确（**不是 0**）；DELTA 用两点历史 → 差分值正确且 quality 透传；手写 INTEGRAL → 拒绝。
- 窗口曲线：给定 window + 历史，curve 点集按 stepMs 输出，某段缺数 → 该点 missing 非空、value=null。
- 沙箱：同一公式连试两次，第二次结果不受第一次 DELTA 基线影响（证明状态隔离）；trial 调用后 calc 状态存储/日志无新增（mock 或断言无写口调用）。
- 同口径：同一 formula+variables 经 trial 与直接调 ArithmeticEvaluator 结果一致。
- 既有 calc 288 契约测试零回归。

## 验收命令（必须原样贴回输出）

1. `cd calc && mvn test` → 0 失败 0 错误（新增 trial 契约测试 + 既有 288 基线零回归），贴 Results 段
2. `cd calc && mvn package -DskipTests` 或工程既有构建命令 → BUILD SUCCESS
3. 范围核对：`git diff --name-only 815cabf..HEAD` 全落改动范围声明；backend/frontend/ingest/resources/contract/calc 既有实现零触碰

## 报告

报告写到：`reports\TASK-0086-report.md`。五项齐全 + 一句话结论 + TDD 红转绿（每修复附"为什么现在对了"）+ 缺数不冒充 0 证据 + 沙箱隔离证据 + 同口径证据。

## 契约

- no-subagents；只改本批改动范围；不置 TASK 已完成；不改 workitems 索引。
- 同引擎同口径（复用 ArithmeticEvaluator/OperatorEvaluator/DeltaOperator，禁止另写求值）；沙箱隔离（算子每请求 new、不落库、不发事件、不污染在线状态）；缺数明示不用 0；手写 INTEGRAL 拒绝。
- 不硬编码凭据/文档库路径；测试失败先定位根因，禁止为匹配输出改测试/改期望。
