# 评审报告：TASK-0045 diff 评审（代码评审）

- 评审角色：fresh 代码评审子代理（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」；按本 brief 逐条核对）
- 被评审对象：实现提交 `4dd8144`（父 `86a1c09`；`86a1c09..4dd8144` 限 `calc/`，21 文件）
- 评审日期：2026-11-24
- 契约依据：`resources/contract/tdengine-super-tables.md`（冻结稿 v1.0.0，**逐字核实**）；详细设计 V1.1 §1.7/§2.2（PDF 抽取）
- 契约约束：fresh、no-subagents、只评审不修改任何文件（本报告为唯一写入）

## 结论先行

**通过（0 Critical / 0 Important / 4 Minor，均不阻断）**。8 条评审重点逐项核对均无 Critical/Important 缺陷；「评审重点 3（契约差异裁决）」经逐字核实契约原文——**执行者裁决正确**：`st_metric_value` 冻结契约为**四元组 tag + 4 列**，brief 的「六元组 / calendar_type tag / 完整度列」属 `st_item_value` 词汇混用，执行者坚持契约并登记字面差异是正确做法（详见 §三）。TDengine 真机未验真按同 0042 口径证据充分、判通过，但不得宣称已验真。

---

## 一、发现清单

| # | 严重级 | 出处 | 一句话说明 |
| --- | --- | --- | --- |
| M-1 | Minor | `MetricAggregator.aggregate`（diff L152-157）+ `aggregateValue` case 1（L188） | 普通 AVG 型（aggType=1 且非 DAY）走「对到达值取均值」而非「按应到数归一化」；当前无 HOUR/非 DAY 平均型的契约/验收场景，且口径与 branch 注释「1 平均」一致，属演进位语义未定版，非本期缺陷。 |
| M-2 | Minor | `MetricRuntime.onDue`（diff L432-433）写死 `yoy=null, mom=null`；`MetricResult`（L289-305）无 yoy/mom 承载 | 冻结表 yoy/mom 列已就位，物化行恒写 null；报告 §三-6 已登记为后续看板道（0053/0060）职责，本任务不假值填充合理，但需留意「日指标字段齐全」（TC-0015 step 2 含同比/环比）在端到端尚未闭合。 |
| M-3 | Minor | `MetricObserveController`（diff L1152/1163-1180）新增无鉴权内网写口 `POST /calc/v1/metric/observe` | brief 只要求 `/calc/v1/status` 观测视图，新增驱动缝扩大内网可注入面（可写进程内物化值）；沿 `/calc/v1/**` 既有无鉴权观测面边界、且非契约命名空间，属可接受但应在部署面限制访问。 |
| M-4 | Minor | 执行者报告 live L5（报告 L182）`M_MON ts=2026-11-01` 与实测 `1790812800000` 实为 `2026-10-01`；另一路 `1795392000000` 实为 `2026-11-23` | 报告日期标签与毫秒 ts 存在约 1 天/1 月偏差（ts 本身与窗口月初语义自洽：MONTH 窗以 UTC 月初对齐，`previous` 对 2026-11-24 返回上一完整月 2026-10-01）。属报告文字标注瑕疵，不影响代码口径；建议订正标签避免误导。 |

> 无 Critical、无 Important。

---

## 二、逐项评审重点核对

### 评审重点 1：聚合正确性 —— 通过

- **比值型分子分母分别聚合再相除**：`aggregateRatio`（diff L196-215）分别对 `ratioNumKey`/`ratioDenKey` 取 `goodValuesWithPrefix` 后 `sum()/sum()`，非逐条比值再平均。契约测试 `ratioAggregatesNumeratorAndDenominatorSeparatelyThenDivides` 以 10+20 / 2+5 = 30/7≠4.5 锁定；live L1 实证 `4.2857`（diff/报告一致）。与详细设计 §1.7「分⼦分⺟先各⾃聚合再相除，分⺟为 0 时结果为空并打标记」逐字一致。分母 0 → `value=null` + `incomplete=true`（L211-213），非 Infinity/假 0。
- **日级平均 = 总量÷24**：`aggregateAverageDiv24`（diff L222-233）固定用 `expectedCount`（DAY 平均传 24）作分母，`value = total/24`；`MetricRuntime.expectedCount`（L437-444）对 DAY+aggType=1 硬编码 24。测试断言 23×24/24=23.0 ≠ 到达均值 24.0，live L2 实证 23.0≠24.0，完整度 23/24。口径正确。
- **级联 SUM**：`aggregateValue` case 2 = sum（L190）；`readValues`（L162-182）用显式键 + 同前缀展开物化子结果并去重。测试 `sumAccumulateAggregatesAllChildren`（2+4+6=12）、`cascadeAggregationUsesSrcTypeAndMaterializedSource`（3+5=8，srcType=2）与 live L3（1..6=21）覆盖。级联 src_type 仅作标记、来源取下级物化结果，符合「读已物化结果不现算」。
- **月窗口 ts=月初（ADR-0010 月末一次）**：`MetricWindow.forInstant` MONTH → `withDayOfMonth(1).truncatedTo(DAYS)`，`previous` → 上一完整自然月（diff L1034-1037、L1048-1063）；`isMonthEndOnce()`；调度 `shiftBack` 用 `minusMonths(1)` 自然日历对齐（L610-620）。契约测试与 live L5 覆盖，无「本月至今」中间快照。
- **无数据 value=NULL + 完整度 0 不写 0**：`noData` 时 `value=null`（L156-157、L204-211），`MetricSink.write` 仍写 NULL 行（L666-678），契约测试 `noDataOutputsNoValueAndZeroCompletenessNotZeroValue` + live L4 实证 `VALUES (..., null, null, null)`。符合 CONV-02 不写假值。
- **同 ts 覆盖幂等**：子表由 metric_code+instance_code 决定、主键 ts=窗口起始（`insertSql` diff L979-991），`withValue` 仅变 value；契约测试 `primaryKeyColumnsAreTsOnlySameTsOverwriteIsIdempotent` 与 live L7（同子表同 ts、value 变化）覆盖。调度侧 `fired` 集合按 `windowKey` 去重（L582-595）。
- 结论：重点 1 全部满足。M-1 为非 DAY 平均口径的演进位提示，不构成阻断。

### 评审重点 2：完整度口径 —— 通过

- **非 GOOD 计缺失、停用剔除**：`MetricValueStore.goodValue`（diff L837-843）仅 `quality==0` 且未停用才计入；`allGood`/`goodValuesWithPrefix` 同步过滤（L851-872）。契约测试以质量码 8/3 + `disable` 构造 7/10=0.7，live L3 实证 0.6 带不完整标记。
- **60% 有效数据带不完整标记**：`incomplete = noData || value==null || completeness<minCompleteness`（L158、L213、L231）；测试与 live 均断言 60%<80% → 带标记。
- **阈值 80% 可配**：`metric-completeness-min` env `CALC_METRIC_COMPLETENESS_MIN` 默认 0.8（`CalcProperties`/`application.yml` diff L73-74、L1202），`CalcPropertiesTest.task0045MetricDefaults` 断言默认值。
- **完整度无物理列、由内存结果 + /calc/v1/status 承载**：`MetricResult` 记录 completeness/incomplete/noData（L305-306），`MetricRuntime.recordObservation`/`view`（L447-475）与 `CalcStatusController`（status.metric）暴露；`TdengineMetricValueWriterContractTest` 明确断言 SQL 不含 `completeness`/`quality` 越界列（L1596-1597），live L8 status 实证含各指标 completeness/incomplete/noData。**可观测性未丢失**——裁决清晰，符合契约。
- 结论：重点 2 满足。M-4（live 日期标签）属报告文字瑕疵，不影响口径。

### 评审重点 3：契约差异裁决（重点）—— 裁决正确，执行者坚持契约无误

**逐字核实 `resources/contract/tdengine-super-tables.md`（冻结稿 v1.0.0）：**

- §2.2 超级表设计表（第 16 行）：
  `st_metric_value | ts TIMESTAMP（窗口起始）, value DOUBLE, yoy DOUBLE, mom DOUBLE | tenant_id、metric_code、instance_code、period | 每个指标一张 | 计算引擎（指标调度）`
- 附录 B DDL（第 38-39 行）：
  `CREATE STABLE IF NOT EXISTS st_metric_value (ts TIMESTAMP, value DOUBLE, yoy DOUBLE, mom DOUBLE) TAGS (tenant_id BIGINT, metric_code BINARY(96), instance_code BINARY(96), period BINARY(8));`

**判定：契约明确且仅有 —— 列 4 个（ts/value/yoy/mom）+ tag 4 个（tenant_id/metric_code/instance_code/period）。无 calendar_type 标签、无完整度/quality 列。** `calendar_type` 只出现在详细设计 §1.7 的 `iot_metric` **定义表**（MySQL）中（「1 自然日历 2 班次日历预留，一期只实现自然口径」），是设计字段/窗口接口概念，**不是** st_metric_value 时序表 tag；「六元组 + full_key」属 `st_item_value`（契约第 15 行：tenant_id/full_key/instance_code/data_code/category/source_type）。backend 读侧 `OpenApiTimeseriesClient`（第 207-210 行）逐字 `SELECT ts, value, yoy, mom ... st_metric_value WHERE tenant_id=... AND metric_code=... AND \`period\`=...`，仅 4 列、按 2 个 tag 过滤，进一步佐证。

因此：
- **执行者的裁决（四元组 tag、完整度不入物理列、无 full: 前缀）正确**，是对冻结契约的正确坚持。
- brief 决策 4 / 实施 brief 第 17、25、27 行的「六元组 tag / calendar_type tag / 结果行含完整度字段」为 `st_item_value` 词汇混用，**不构成契约升级依据**（CONV-03 需契约变更方可扩列/扩 tag，执行者未擅改契约文件，正确）。
- `MetricValueWriter.insertSql`（diff L979-991）逐字落 4 元组 tag + 4 列、无 full:/无 completeness，契约测试与 live L6 逐值断言一致。
- **结论：此点判「执行者正确，无缺陷」**。若将来确需在物理行增加 calendar_type/完整度，须先升版契约（CONV-03）另开任务，与本任务无关。

### 评审重点 4：调度裁决 —— 通过

- **自实现而非 XXL-Job**：`MetricScheduler`（diff L477-626）照 0041 `PeriodicScheduler` 范式，单线程 daemon + fail-soft（`scheduleAtFixedRate` + try/catch RuntimeException）；未引 XXL-Job 依赖（calc/pom.xml 无 xxl 项，grep 仅命中 javadoc/log 文字）、未改 backend sys_job（`86a1c09..4dd8144` 无 backend 改动）。与实施 brief 第 15 行「调度裁决」一致，属登记的字面差异非缺陷。
- **env 改频能力**：`CALC_SCHEDULER_TICK_SECONDS` 复用（live 起带 60s），`start(tickSeconds)` 校验 ≥1（L396-398、L541-560）。
- **自然日历边界**：`MetricWindow` HOUR/DAY/MONTH/YEAR 用 `truncatedTo`/`withDayOfMonth(1)`/`withDayOfYear(1)`/`plusMonths`/`plusYears`（L1019-1046），`shiftBack` 用 `minusHours/Days/Months/Years`（L610-621），非固定秒数。测试逐边界断言通过。
- 结论：重点 4 满足。

### 评审重点 5：物化写纪律 —— 通过

- **逐语句 REST**：`MetricValueWriter.write`（L967-976）每次 `runner.execute(insertSql)` 单条下发；`rest()` 工厂逐条 POST `/rest/sql`；契约测试 `writerUsesPerStatementRestAndParsesTaosCode` 断言 2 条写=2 次调用、`assertFalse(sql.contains(";"))`（L1598）防 0029 多语句信封静默空操作。
- **full_key 规范无 `full:` 前缀**：指标行以 metric_code 标签定位（L905-906），测试断言 SQL 不含 `full`（L1595）；与 0049 F-02 纪律一致（0042 写口 likewise 对 st_item_value 用无前缀 fullKey，本任务不写该列）。
- **tag 逐值精确断言**：`insertSqlHasFourFrozenTagsInExactOrder` 断言 `TAGS (1, 'M_AVG_DAY', 'I01', 'DAY')` 与列顺序 `VALUES (ts, value, yoy, mom)`；live L6 逐值核对。
- **taosAdapter code 解析**：`requireTaosSuccess` 正则 `"code"\\s*:\\s*(-?\\d+)`（L952-965）容错冒号后空格、非 0/无 code fail-closed，与 0049 F-01 同款；测试覆盖。
- **st_metric_value 与 st_item_value 并存不冲突**：前者用独立 `MetricValueWriter`/`st_metric_value`、独立 Sink；两者写入口分离、默认后端均 opt-in（metric 默认关），不共享写口状态。契约测试无回归。
- 结论：重点 5 满足。

### 评审重点 6：TDengine 真机未验真 —— 证据充分，判通过（附条件）

- REST 6041 故障实证（报告 HTTP=000、6.006s，TD-0053 登记）与 0042 同口径。
- 证据面：契约测试（假 runner 捕获 INSERT、逐值 tag/列断言、fail-closed code 解析）+ 记录式 `MetricValueRecorder`/`MetricSink` 假写口 + live `RESULT fail=0` 全序列（比值/日均/级联/完整度/无数据/月末/四元组/同 ts 覆盖）+ `lastWriteSql` 观测，充分覆盖本任务范围内的写侧正确性。
- 结论：**证据充分判通过，真机补验列为环境恢复后条件（报告 §三末 5 步）**；报告明确「不宣称已验真」，符合要求。不得在结论中宣称真实写入可用。

### 评审重点 7：测试有效性 —— 通过

- **220 测试 0F**：195 基线逐类零回归（每类数值与 0042 修订轮 Results 逐字一致）+ 25 新增（窗口 7 + 聚合 8 + 写口 5 + 调度 4 + 配置 1）；报告 Results 段逐类贴回，`Tests run: 220, Failures: 0, Errors: 0, Skipped: 3`。3 Skipped 为 0041/0049 env 门控（未改这些写口）。
- **live RESULT fail=0**：L1-L8 逐条原始输出与断言对应（含 `RESULT fail=0`、全停归零）。
- **4 处 TDD 红转绿留痕**：3 项真实断言红（completeness 0.7 vs 0.9、avg 23.0 vs 3.0、scheduler 1 vs 0）+ 1 项 live NPE（比值缺键 `goodValuesWithPrefix` null 前缀）→ 均附「为什么现在对了」，且 fail-closed 修复合理（null/blank 前缀返回空集而非抛 NPE，符合「配置缺失=无数据」）。
- **双 BASE 登记清晰**：开工 HEAD=`01236fd`、diff/评审 BASE=`e49fea8`、并行 0059 提交 `86a1c09`（13 项 frontend/**，父链实证）；评审按提交 `4dd8144` 判断归属。口径清楚。
- 结论：重点 7 满足。M-4（live 日期标签瑕疵）归报告订正类，不影响测试有效性判定。

### 评审重点 8：零回归与范围 —— 通过

- **commit 仅 21 文件 calc/**：实证 `git show --name-only 4dd8144` = 21 项，非 calc 计数 0；`git status --short calc/` 空（工作区干净）。
- **0041/0042/0049 既有测试零回归**：195 基线逐类数值与 0042 修订轮一致，无改动这些写口。
- **契约零触碰**：`86a1c09..4dd8144` 无 `resources/**`、`backend/**` 改动（冻结契约文件未改）。
- 结论：重点 8 满足。

---

## 三、契约差异裁决（重点 3）明确判定

**判定：执行者正确，无缺陷。**

- `resources/contract/tdengine-super-tables.md` v1.0.0 逐字：st_metric_value = **4 列（ts TIMESTAMP〔窗口起始〕/ value DOUBLE / yoy DOUBLE / mom DOUBLE）+ 4 元组 tag（tenant_id / metric_code / instance_code / period）**；无 calendar_type tag、无完整度/quality 列。附录 B DDL 复核一致。
- 「六元组 / calendar_type tag / 完整度列」是 `st_item_value`（tenant_id/full_key/instance_code/data_code/category/source_type）与设计 §1.7 `iot_metric` 定义表（含 calendar_type）的概念混用，不适用于 st_metric_value 物理表。backend 读侧 `OpenApiTimeseriesClient` 只读 4 列、按 metric_code+period 过滤，佐证契约即 4 元组。
- 执行者按「brief 决策 4『按超级表契约逐字』为最高约束」坚持 4 元组 + 4 列，未改冻结契约，登记字面差异并声明「若需扩列/扩 tag 须先升版契约 CONV-03 另开任务」——**处置完全正确**。
- calendar_type 保留在窗口唯一键（`MetricWindow.key()` 含 calendarType）与 `MetricValue.calendarType()` 内存载荷；完整度/incomplete/noData 由 `MetricResult` 与 `/calc/v1/status.metric` 承载，可观测性未丢失。

---

## 四、最终结论

**通过**。0 Critical、0 Important；4 Minor（M-1 非 DAY 平均语义演进位 / M-2 yoy/mom 恒 null 留位 / M-3 内网观测驱动口无鉴权 / M-4 live 日期标签瑕疵）均不阻断本任务验收，按登记/后续道处理即可。8 条评审重点逐项核对无实质偏差，其中**评审重点 3 契约差异裁决明确判执行者正确（四元组 tag + 4 列，brief 六元组/日历 tag/完整度列为词汇混用）**。TDengine 真机写入未验真为环境阻断（TD-0053），证据充分判通过、真机补验列条件，不宣称已验真。TASK-0045 应保持「进行中」待协调者裁决，未置「已完成」。
