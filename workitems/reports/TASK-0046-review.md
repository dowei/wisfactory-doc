# TASK-0046 diff 评审报告（代码评审）

- 评审角色：fresh 代码评审子代理（口径 `.agents/protocol/L1-execute.md` §13「diff 评审」）
- 评审对象：BASE `bf84526` → 交付提交 `f8421c7`（单提交，19 文件 `calc/**`）
- 评审依据（仅此及工作区源码）：review-brief、diff 包、实施 brief、执行者报告、TASK-0046 详情、FEAT-0015/REQ-0025/TC-0015、ADR-0007/0010/0011、0045/0041/0042 报告、`resources/contract/tdengine-super-tables.md`，以及评审中为核对载荷而回溯的 `resources/contract/unified-message-model.md` v1.1.0 与 `push-payload-evaluation.md`（ADR-0007 明文引用「统一消息模型」契约，属工作区源码，未向协调者追问）
- 评审日期：2026-11-24
- fresh 声明：未共享协调者上下文；判断不了的项标「无法判定」

## 独立复核的原始证据（非仅采信报告）

1. **范围**：`git show --name-only f8421c7` 与 `git diff --name-only bf84526..f8421c7` 均为 19 项，`grep -v '^calc/'` 为空 → 越界 0，确认协调者「19 文件全落 calc/**、单提交无交错」。
2. **测试**：独立执行 `cd calc && mvn test` → `Tests run: 240, Failures: 0, Errors: 0, Skipped: 3` / `BUILD SUCCESS`；surefire XML 聚合 33 类 = 240/0/0/3，与报告逐类数值一致。
3. **EMQX 退化**：评审时点独立探测 `192.168.31.145:1883` → TCP 可连（`/dev/tcp` OPEN），但裸 MQTT 3.1.1 CONNECT 报文 `recv` 超时、无 CONNACK → 报告所述退化复现成立。

---

## 重点逐条判定

### 重点 1 迟到修正正确性 —— 基本通过，级联一项不通过

- 宽限期判据方向：`MetricRuntime.onLateArrival` 用 `end <= now <= end+graceMillis`（`MetricRuntime.java:180-184`），即「窗口已结束且 now 在宽限期内」才算迟到，超窗 fail-closed；TDD 第 3 处红转绿（原 `now>=end+grace` 方向错误）修复留痕与代码一致。✓
- 窗口键幂等：`recalcFired` 集合按 `w.key()=tenant|metricCode|startMillis|calendarType` 去重（`:170,186-188`），同窗重复到达只算一次、只留一条 BACKFILL；契约测试覆盖。✓
- 同 ts 覆盖写：`doCompute` 以 `w.startMillis()` 为 ts 构造 `MetricValue` 交 `sink.write`，TDengine 同 ts 覆盖；live L1→L2 的 `lastWriteSql` 显示 ts=1795305600000 不变、value 100→150。✓
- **级联（不通过，见发现 F-01）**：`cascadeRecalc`（`:202-215`）**未复用 0040 拓扑序**，仅为对 `defs` 的单趟平扫、单层传播、并用 `k.contains(changed.metricCode())` 子串匹配；更关键的是级联 `doCompute` 走 `MetricAggregator.readValues` 从**原始样本 `values` 表**读数（`MetricAggregator.java:55-74`、`MetricValueStore.java:63-99`），而下级指标重算结果写的是另一张 `materialized` 表（`MetricValueStore.java:26-44`），二者不互通 → **修正后的下级值实际未传播到上级指标**。该路径 6 个契约测试零覆盖。
- 失败隔离：事件通道 fail-soft（`MetricEventPublisher.safePublish`）、sink WARN 隔离均已实现；级联 `doCompute` 无逐项 try/catch，单个上级计算抛异常会冒泡（并入 F-01）。

### 重点 2 BACKFILL 留痕 —— 通过（Minor 见 F-07）

- §1.9A 全字段：`CalcLogEntry(taskId, triggerType, fullKey, periodStart, periodEnd, isRecalc, inputSummary, resultStatus)` 齐备，`fullKey=tenant|metricCode|instanceCode|period`、`isRecalc=1`、`triggerType=BACKFILL`；`backfillAuditHasAllRequiredFields` 逐字段断言，live L2 `calcLog` 回显一致。✓
- TDengine 不可用内存兜底 + 契约测试口径：`CalcLogStore` 纯内存（CopyOnWriteArrayList+AtomicLong），`auditFallsBackToMemoryWhenTdengineUnavailable` 断言；真机物理落库已登记演进位。✓
- 留痕写失败不阻塞计算：留痕在 `doCompute` 之后调用，计算先落地；内存表无失败模式（无显式 try/catch，见 F-07）。

### 重点 3 yoy/mom 伴随值 —— 通过（Minor 见 F-05/F-06）

- 从已物化结果读：`CompanionValueCalculator.calculate` 读 `store.materialized(...)`，不现算原始点；同比为「本期 ts 减 1 年」、环比为回退一周期，`compareTs` 自然日历实现，TDD 同比窗多回退一周期的修复与代码一致（`CompanionValueCalculator.java:91-102`）。✓
- 写既有 yoy/mom 列、不改契约：live `lastWriteSql` 为 4 列 `(ts,value,yoy,mom)`，无扩列；0045 deferred M-2 补齐。✓
- 缺同期/分母 0 → null+原因不写 0：`calculate` 返回 null 并附 `yoyNote/momNote`，测试覆盖；CONV-02 口径成立。✓
- 级联指标 yoy/mom 随主值同步：主值路径成立；级联路径受 F-01 影响，实际同步无法保证 → 该项**部分无法判定**（依赖级联修复）。
- live 实证 0.25/-0.2 → 0.875/0.2 同行覆盖：与 live SQL 一致。✓

### 重点 4 事件生产侧 ADR-0007 载荷逐字一致 —— 【明确判定：不能判定为「逐字一致」，存在 Important 偏差】

逐项核对结论：

| 核验项 | 结论 |
| --- | --- |
| 主题 `iot/{tenant}/metric/out` / `iot.metric.result.{tenant}` | ✓ `MetricProducedEvent.mqttTopic/kafkaTopic` 逐字锁定，契约测试 `topicsAreTenantIsolated` 覆盖 |
| 无 `is_recalc` / `calcTime` | ✓ **评审者核实：这两个字段确实不应在 ADR-0007 载荷里** —— ADR-0010 TD-0030 明文「不加 is_recalc/calc_time」，冻结契约 `push-payload-evaluation.md` 扩展字段表亦无此二者；实现正确省略，执行者声称成立 |
| 双通道同载荷、租户双隔离（主题+载荷）、QoS1 fire-and-forget、Kafka env 默认关 | ✓ 契约测试 + 代码一致 |
| hash（SHA-256） | ✓ `computeHash()` 使用 SHA-256、64 字符十六进制；`payloadHasAllAdr0007FinalizedFields` 断言长度 |
| **「逐字对齐 ADR-0007 14 字段」** | ✗ **不成立/无法逐字核对** |

理由（见 F-02）：
1. ADR-0007 正文只写「载荷在统一消息模型上扩展 metric_code、period、窗口起止、value、yoy、mom」，**从未定义「14 字段」清单**；冻结契约 `unified-message-model.md` v1.1.0 §3.5 推送行列出的扩展字段为 **9 个**（metric_code、instance_key〔即 full_key〕、period、window_start、window_end、value、yoy、mom、**quality**）。报告自称「载荷 14 字段（ADR-0007 定版）」无权威出处。
2. 实现的 14 字段相对冻结契约扩展表：**缺 `quality`**（契约 9 扩展字段之一，CONV-02 核心语义）；用 `instanceCode`（值 "I01"）代替契约的 `instance_key/full_key`。
3. camelCase 命名与 windowStart/windowEnd 的毫秒类型，恰是 `unified-message-model.md` v1.1.0「与上游差异」第 7/8 条与 `push-payload-evaluation.md` 遗留 1/2 明文「**待 TASK-0053 落地时定版**」的项；报告 §五-1 自认「0053 消费侧须对齐，若协调者另定命名风格须升契约修订号」——即尚未定版，不能称「逐字一致」。

**判定：载荷与 ADR-0010（无时间/重算标记字段）一致、与 ADR-0007 意图方向一致，但不存在可供「逐字核对」的 14 字段权威定义，且与冻结推送契约存在字段级偏差（缺 quality、instanceCode≠instance_key/full_key、命名/类型属未定版项）。「逐字一致」主张不成立，需在 0053 耦合前闭合（F-02，Important）。**

### 重点 5 同窗口重发不变（ADR-0010）—— 通过（Minor 见 F-06）

- 载荷无 is_recalc/calc_time；字段集/身份字段（metricCode/instanceCode/period/windowStart/windowEnd/tenant/completeness/incomplete/eventType）重发一致，value/yoy/mom/hash 随修正变化——与 ADR-0010 TD-0030「按指标全键+窗口幂等覆盖、不加时间标记字段」的合理解读一致，报告 §四-4 的精确解读可采纳。✓
- 契约测试 `resendSameWindowPayloadIsByteIdentical` 仅证明序列化确定性（两个全同对象），真正的修正重发场景由 live 步骤 3 覆盖；测试命名/断言可改进（F-06）。
- 对接提示：hash 含 value/yoy/mom，修正后 hash 变化，0053 必须以「指标全键+窗口」而非 hash 为幂等键（F-06）。

### 重点 6 EMQX 环境替代证据充分性 —— 【明确判定：充分（在既定假后端先例与 TDengine 同口径下成立），非真机验证】

按 brief 给的口径「协议层真验证 + 生产栈真实 + 环境恢复后真机复验条件清晰」逐项判定：

1. **退化事实独立复现**：评审时点 TCP 1883 OPEN、MQTT CONNECT 无 CONNACK（本次独立裸协议探测超时），与报告登记一致，非实现缺陷。✓
2. **协议层真验证**：本地 MQTT 3.1.1 夹具 broker + 外部订阅者用真 MQTT 协议（裸 socket）+ calc 生产侧用 Paho 1.2.5 真客户端，完成真 pub/sub 全序列，两条事件实收、租户 B 零消息——是协议层真实交换，非 mock 返回值。✓
3. **生产栈真实**：`PahoMqttChannel` 即生产类，仅 `CALC_METRIC_MQTT_BROKER` env 切换地址，代码零改；与 TDengine 记录式写口同源纪律。✓
4. **恢复复验条件清晰**：报告 §三末 #1 给出 env 切换 + 订阅断言 + 租户 B 断言，broker 恢复后免改代码复验。✓

结论：**替代证据对本任务「事件生产侧」的代码正确性判定是充分的**；缺口是夹具非 EMQX（未验 EMQX 的匿名 allow/ACL/并发/重连等 broker 特有能力），执行者未宣称真 EMQX 实收、已如实登记，真机复验仍为用户侧清晰的待办。无 Critical/Important 发现。
（旁证瑕疵：夹具首版曾无法解析 SUBSCRIBE/QoS1 PUBLISH，属测试夹具自身问题，已修正后实收，不伤生产栈结论；live topic 显示 `ot/...` 少首字符为夹具解析显示伪影，应用侧主题常量由契约测试逐字锁定。）

### 重点 7 测试有效性 —— 通过

- 240 测试 0F0E3Skip、BUILD SUCCESS 已独立复现；220 基线逐类零回归（仅 CalcPropertiesTest 9→10），新增 20（伴生 6+迟到 6+事件 7+配置 1）计数自洽。
- 3 处 TDD 红转绿均有具体失败现象与「为什么现在对了」（同比窗回退、recorder 载荷未登记、宽限判据方向+幂等）；红态属历史时点，无法逐条重演，按报告留痕采信。
- live 全序列与断言基本对应；级联路径零测试（F-01）。

### 重点 8 零回归与范围 —— 通过

- 19 文件全 `calc/**`、单提交、backend/frontend/ingest/契约零触碰，独立 git 核对确认；0041/0042/0045 既有测试逐类计数未变。

---

## 发现清单

| 级别 | 出处 | 一句话说明 |
| --- | --- | --- |
| **Important** | F-01 · `D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\metric\MetricRuntime.java:202-215`；`MetricAggregator.java:55-74`；`MetricValueStore.java:26-44,63-99` | 级联重算用 `defs` 单趟平扫单层传播、子串匹配、未复用 0040 拓扑序，且从原始样本 `values` 表而非下级重算结果 `materialized` 表读数，修正后的下级值不会传到上级；该路径零测试覆盖——「级联指标重算传播沿依赖图向上」未真正实现。 |
| **Important** | F-02 · `D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\event\MetricProducedEvent.java:302-320`；`D:\workspace\WisFactory\resources\contract\unified-message-model.md` §3.5 推送行（line 61）；`push-payload-evaluation.md` line 25/77-79；ADR-0007 line 9 | 「逐字对齐 ADR-0007 14 字段」无权威出处（ADR-0007 未定 14 字段、冻结契约定 9 扩展字段），载荷缺契约扩展字段 `quality`、用 `instanceCode` 代替 `instance_key/full_key`，camelCase/毫秒类型属契约明文「待 TASK-0053 定版」项——0053 对接前须确认，否则消费侧契约错位。（`is_recalc/calcTime` 确实应缺席，此项实现正确。） |
| Minor | F-03 · `CompanionValueCalculator.java` line 85 注释 vs line 97/102 代码 | `compareTs` 方法注释仍写「同比 = 上一周期再减 1 年」，与 TDD 修复后的正确实现（本期直接 minusYears(1)）矛盾，注释误导。 |
| Minor | F-04 · `D:\workspace\WisFactory\calc\src\test\java\net\otlab\calc\metric\MetricCompanionValueContractTest.java`（calculate 调用处）/`MetricRuntime.java:132-136` | 缺同期/分母 0 的「原因」`yoyNote/momNote` 被 `CompanionValues` 返回后在 `doCompute` 丢弃，只把 null 写入 yoy/mom 列，原因既不落库也不进事件——「null+原因」只交付了 null（与 4 列冻结契约一致、compareNote 进载荷属 0053 遗留，故记 Minor）。 |
| Minor | F-05 · `D:\workspace\WisFactory\calc\src\test\java\net\otlab\calc\event\MetricProducedEventContractTest.java:1015-1021` | `resendSameWindowPayloadIsByteIdentical` 用两个完全相同对象断言逐字一致，只证明序列化确定性，未覆盖「值变化后的同窗重发」；应补字段集/身份字段一致、仅值伴随值 hash 变化的断言。 |
| Minor | F-06 · `D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\event\MetricProducedEvent.java:326-342`；报告 §五-1 | hash 含 value/yoy/mom，修正后 hash 变化；报告允许 0053「按全键+窗口（或 hash）幂等」，若 0053 误用 hash 为键会把修正重发当新事件，须明确只按「指标全键+窗口」幂等。 |
| Minor | F-07 · `MetricRuntime.java:191-194,209-212` | `calcLog.append` 无 try/catch，当前靠内存 CopyOnWriteArrayList 不会失败；未来接真 REST 落地时须补 fail-soft（「留痕写失败不阻塞计算」的显式保证）。 |

---

## 结论

**不通过（按 §13 走退回/修订路径）**：存在 2 项 Important（F-01 级联重算未按依赖图传播修正值且零测试；F-02 ADR-0007 载荷「逐字一致」主张不成立、与冻结推送契约存在字段级偏差）。其余重点（范围/零回归、留痕、yoy/mom 主值路径、ADR-0010 解读、测试有效性）通过。

对用户特别要求的两个明确判定：

- **重点 4（ADR-0007 载荷逐字一致）：不能判定为逐字一致**——无「14 字段」权威定义，与冻结契约缺 `quality`、`instanceCode`≠`instance_key/full_key`、命名/类型属 0053 未定版项；但 `is_recalc/calcTime` 确实应缺席且实现正确。
- **重点 6（EMQX 替代证据充分性）：充分**——退化独立复现、协议层真验证、生产栈真实、复验条件清晰；夹具非真 EMQX 的缺口已如实登记，不构成 Critical/Important。

建议：F-01/F-02 修订后做 scoped 复评（只看修复 diff 与原发现）；F-03~F-07 记 TASK 任务结果 deferred，收尾进 LOG。评审未修改任何文件。
