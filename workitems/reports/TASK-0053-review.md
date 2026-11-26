# TASK-0053 代码评审报告（diff 评审）

- 评审角色：代码评审子代理（fresh，口径 L1-execute §13「diff 评审」）
- 评审对象：实现提交 `a230f91`（BASE `ebd443d`，隔离 0047 并行 `f19edc1`；22 文件 `backend/ot-admin` openapi 模块 + 测试）
- 评审依据：仅 brief 指到的文件（`TASK-0053-diff.txt` / `TASK-0053-brief.md` / `TASK-0053-report.md` / TASK-0053 + FEAT-0017 + REQ-0028 + TC-0017 + ADR-0010 / `resources/contract/unified-message-model.md` v1.1.0 §3.5 / 0046/0051/0054/0029 报告）与工作区源码；fresh 独立判断，未向协调者追问，判断不了处标「无法判定」，未派 subagent。
- 评审日期：2026-11-24

## 评审前独立核验（非仅采信报告）

| 核验项 | 方法 | 结果 |
| --- | --- | --- |
| 提交范围 | `git show --name-only a230f91` | 22 文件，全部 `backend/ot-admin/**`，非 backend 路径 0；与报告 §一/§二-4 一致。✓ |
| 冻结契约未改 | `git log -- resources/contract/unified-message-model.md` + 提交文件表 | 契约文件仅见于初始冻结提交 `470d7f5`，`a230f91` 未触碰。✓ |
| 测试 2445/0 | 实跑 `mvn install -pl ot-core -DskipTests -q && mvn test -pl ot-admin` | **Tests run: 2445, Failures: 0, Errors: 0, Skipped: 0 / BUILD SUCCESS**（日志中其它模块的 ERROR 行为既有用例的预期异常日志，非测试失败）。✓ |
| EMQX 1883 退化 | 独立裸 MQTT 3.1.1 CONNECT 报文探测 `192.168.31.145` | 1883/8883/8083/18083 TCP 均 OPEN；1883 CONNECT 后 `recv` 超时、无 CONNACK——报告 §二-2 环境退化复现成立。✓ |
| Paho/CGLIB 修复 | 读实际源码 | `SubscriptionConfig` 为 `@Configuration(proxyBeanMethods=false)` + public 构造；`ot-admin/pom.xml` 的 Paho 1.2.5 依赖来自旧提交 `2e4a1b9`（TASK-0026），非本任务新增。✓ |

---

## 评审重点逐项结论

### 重点 1 字段名定版（契约义务）—— 通过
- `ItemChangeEventVO` / `MetricProducedEventVO` 的字段 + javadoc + `toMap()` 有序序列化 + `OpenApiEventCodec` 的 4/9 字段校验表 + `ItemChangeEventContractTest` / `MetricProducedEventContractTest` 逐字段断言（snake_case 不出现、窗口毫秒），把契约 §与上游差异第 7/8 条「待 TASK-0053 定版」闭合为 **camelCase + 毫秒时间戳**，与 0046 生产侧 `MetricProducedEvent` javadoc「最终风格归 0053 定版」、修订轮已按 camelCase/毫秒发出的口径一致（0046 报告修订轮 §四-2 / `ebd443d`）。
- 冻结契约文件本体零改、`CONV-03` 升版走裁决已如实登记（报告 §五-1）。**「定版通过 VO+javadoc+测试实现、不改冻结契约本体」成立。**
- 实证一致：0046 发出的 14 键中 `metricCode/instanceKey/windowStart/windowEnd/quality` 均为 camelCase/毫秒，0053 消费侧 `METRIC_REQUIRED` 与 VO 同键；两侧可真实对接。

### 重点 2 数据项订阅 —— 部分通过（见 F-02；字段/scope/HMAC 本身通过）
- 4 字段逐字 camelCase：`ITEM_REQUIRED={fullKey,ts,value,quality}`（`OpenApiEventCodec.java:26`）、`ItemChangeEventVO.toMap()` 键序逐字、契约测试 `payloadIsExactlyFourContractFields` 锁键序、缺一即拒——成立。✓
- 订阅时 scope 校验：`SubscriptionRegistry.subscribeItem` 经 `requireScope`（`resolveClientScope(clientId,tenantId,scopeJson)` 显式租户重载，0051 同口径三态）+ `instanceMapper.selectByCodesForTenant` + `scope.instanceIds().contains(r.getId())`，不在授权范围即 6003 拒绝（diff `SubscriptionRegistry.java:867-889`）——成立。✓
- HMAC 链走 0051 既有不改：`SubscriptionController` 位于 `/openapi/v1/subscribe/**`（过滤器 `PATH_PREFIX=/openapi/v1/` 全覆盖）、类级 `@Anonymous` + `requireIdentity` fail-closed；过滤器/审计代码未改——成立。✓
- 但 brief 决策 1 明示「事件源=0029 新值事件（经数据服务转发或桥接……不自造新事件）」与 TASK 验收「产出后 N 秒内收到载荷」这一端到端链路，本提交无生产/桥接实现，见 F-02。

### 重点 3 指标事件消费（幂等非 hash）—— 【明确判定：通过（关键项正确，非 hash 成立）】
这是本任务关键项，给出显式判定：
- **实现口径正确**：幂等键 `MetricProducedEventVO.idempotencyKey() = tenant|metricCode|instanceKey|period|windowStart|windowEnd`（指标全键 + 窗口），**不含 hash**（diff `MetricProducedEventVO.java:323-327`）；`SubscriptionEventStore.acceptMetric` 以 `event.idempotencyKey()` 作 `metrics` Map 的 upsert 键（diff `SubscriptionEventStore.java:690-696,708-715`），`hash` 仅作为 VO 传输字段落缓冲，从不参与去重。✓
- **测试正确区分**：`EventRingBufferContractTest.idempotentByFullKeyAndWindowNotHash` 用同键同窗、值 `100.0→150.0`、hash `hash-v1→hash-v2` 两条事件，断言 `metricCount()==1`、`latest.value==150.0`、`latest.hash=="hash-v2"`——若以 hash 为键则 count=2，必红。✓
- **live 实证对应**：报告 S4（diff 报告 §二-2）首条 `value=100.0/hash=h1` → 修正重发 `value=150.0/hash=h2` 同窗口（1795305600000/1795392000000），结果 `count=1`、以 150.0 为准。✓
- **与上游闭环**：0046 修订轮 F-06 强化登记「hash 含 value/yoy/mom，修正后变化；0053 必须以指标全键+窗口幂等，不得用 hash」，ADR-0010 TD-0030 同口径；消费侧严格承接，未复发 F-06 风险。✓
- 9 扩展字段齐备（含 quality），MQTT+Kafka 双通道（后者 env 默认关）、同窗重发以最新值为准、无订阅者不报错——均通过。
- **结论：重点 3 通过；幂等严格按「指标全键+窗口」非 hash，实现/单测/live 三重一致，无偏差。**

### 重点 4 租户隔离（brief 所列判据：摄入侧）—— 通过；另见 F-01
- 摄入侧双校验成立：`SubscriptionConsumer.onMetricMessage` 先 `parseMetricTopic` 得主题 tenant、`hasMetricSubscription(topicTenant)` 命中订阅，再比较 `topicTenant.equals(event.getTenant())`，不一致丢弃 + WARN（diff `SubscriptionConsumer.java:625-647`）；数据项侧按主题 tenant+instance 匹配订阅 + `fullKey` 实例命名空间前缀校验（diff `:599-622`）。live S5（租户 B 载荷跨主题丢弃 + WARN）实证；scope 三态与 0051 `resolveClientScope` 显式租户重载同源。✓
- 就 brief 重点 4 明列判据（主题+载荷双校验、租户 B 载荷丢弃、scope 三态）而言**通过**。但同批新增的对外事件读口存在跨租户读取面，属重点 4 判据之外的新发现，单列 F-01（不影响本重点按所列判据通过的结论）。

### 重点 5 不写业务表 —— 通过
- `SubscriptionEventStore` 仅用两个内存 `LinkedHashMap`（metrics/items）做环形缓冲 + 幂等登记（diff `SubscriptionEventStore.java:681-753`）；`subscribe/` 包全包检索无任何 Mapper 写 / INSERT / `st_metric_value` 写路径（`SubscriptionRegistry` 的 tenant/instance mapper 仅只读用于 scope 解析）。指标结果以 0045 物化表为准、事件仅触发信号、无重复落库路径——成立，ADR-0010 遵循。✓

### 重点 6 审计与订阅管理 —— 通过
- 订阅开通/关闭挂 `@OperateLog`（INSERT/DELETE，`SubscriptionController.java:55,64,72,80`），0079/0052 范式；类在 `/openapi/v1/subscribe/**` 复用 0051 HMAC 过滤器 + `iot_api_call_log` 403 类全量审计，过滤器代码零改（live 审计 4 类订阅路径 200 落库实证）。✓

### 重点 7 环境替代证据充分性 —— 【明确判定：充分（与 0046 同口径成立，非真机验证）】
逐项判定：
1. **退化独立复现**：本人独立探 EMQX 1883——TCP OPEN、MQTT CONNECT 无 CONNACK（同 0046 报告 §二与 0046 评审重点 6 的复现），环境退化属实、非实现缺陷。✓
2. **协议层真验证**：本地 MQTT 3.1.1 夹具 broker，消费侧 `PahoSubscriptionBridge` 用真 Paho 1.2.5 客户端连夹具订阅两路通配符，发布侧裸 MQTT 3.1.1 真协议 PUBLISH，S3-S6 真收数据项 4 字段 / 指标 9 字段、全键窗口幂等、租户丢弃、无订阅者不报错——是真协议交换，非 mock 返回值。✓
3. **生产栈真实**：消费侧生产代码即 Paho 桥 + `SubscriptionConsumer` + `OpenApiEventCodec`，非测试替身；仅 broker 地址由 `OT_OPENAPI_SUB_MQTT_BROKER` env 切换（默认 `tcp://192.168.31.145:1883`），恢复后同一交付免改码复验，补验条件清晰（报告 §三末）。✓
4. **Kafka**：无 broker、env 默认关（`kafka-enabled=false`），`KafkaSubscriptionBridge` 条件装配演进位占位、统一调 `onMetricMessage`；契约测试覆盖 `iot.metric.result.{tenant}` 主题解析同口径，未冒充真 broker 消费。✓
5. 缺口如实登记（夹具非真 EMQX，未验 ACL/并发/重连等 broker 特有能力，未宣称真机实收）。与 0046「充分（协议层真验证+生产栈真实+复验条件清晰），非真机验证」同口径。
- **结论：重点 7 替代证据充分性判定为「充分」——对本任务消费侧代码正确性判定充分，不构成 Critical/Important；夹具非真 EMQX 的缺口已如实登记、复验条件清晰。**

### 重点 8 测试有效性 —— 通过
- 2445/0/0/0 已由本人独立复跑复现；新增 22 用例（2+4+5+9+2）逐类计数与报告一致，0051/0054 既有类（HmacAuthStrategyTest 22 / OpenApiQueryServiceImplTest 31 / OpenApiDictSyncMapperSqlTest 3 / OpenApiAuthFilterTest 7 等）逐类零回归。✓
- 4 处 TDD 红转绿留痕具体（编译期缺符号 / 测试嵌套类型 / `@Configuration` CGLIB 构造修复 `proxyBeanMethods=false`+public 构造 / 冒烟 QoS1 假 packet-id），每项附「为什么现在对了」，与源码吻合；红态属历史时点不可逐条重演，按留痕采信（同 0046 评审口径）。✓
- live S1-S6 与断言一一对应；dev `@Profile("dev")` 观测面不进 prod，prod 档默认零连接尝试 + ERROR=0 + 签名 200/篡改 403+6001 实证。✓

### 重点 9 零回归与范围 —— 通过
- `git show --name-only a230f91` 22 文件全落 `backend/ot-admin` openapi 模块 + 测试；0047 `f19edc1`（calc）独立提交隔离，calc/frontend/ingest/契约零触碰；越界 0。✓

---

## 发现清单

> 严重级：Critical（阻断）/ Important（应修）/ Minor（建议）。出处为工作区绝对路径 + 关键行。

| 级别 | 出处 | 一句话说明 |
| --- | --- | --- |
| **Important** | `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\openapi\controller\SubscriptionController.java:87-99`；`D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\openapi\subscribe\SubscriptionServiceImpl.java:60-70`；`D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\openapi\subscribe\SubscriptionEventStore.java:28-29,92-99` | 事件读口 `GET /openapi/v1/subscribe/{metric,item}/events` 只 `requireIdentity(request)` 校验 HMAC 却丢弃身份，直接返回**全局**环形缓冲（`store.metricRing()/itemRing()` 不按 tenant/appKey 过滤）；缓冲按全键/窗口或 fullKey 建键、不按租户分区，因此任一租户 B 的合法 appKey 可读到租户 A 已落缓冲的事件载荷——摄入侧精心做的租户双校验在读口被绕过，违反 REQ-0028 验收 3 / FEAT-0017「主题按租户隔离」；live S5 仅测摄入侧丢弃，未覆盖两合法 appKey 间的读隔离。 |
| **Important** | `D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-brief.md:17`（决策 1）；`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0053.md:14,24`；`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-report.md:146,241-242`（live S3 / 未做项） | 全仓检索仅见 `iot/open/{t}/{i}/item` 的**订阅**常量（`PahoSubscriptionBridge`/`SubscriptionRegistry`），无任何组件把 0029 的 `iot:newvalue:{tenant}` 新值事件转发/桥接为该主题的生产者；live S3 由裸协议脚本直接 PUBLISH 注入事件，未走 0029 事件链。故 brief 决策 1「事件源=0029 新值事件（经数据服务转发或桥接……不自造新事件）」与「产出后 N 秒内收到载荷」的端到端链路未被交付/验证，报告的未做项亦未登记此缺口——需协调者澄清该桥接归属（本任务 vs 后续任务），否则数据项订阅在生产环境无事件来源。 |

说明：F-02 涉及归属边界，fresh 评审者无法从 brief/工作项判定该桥接是否明确划归本任务（brief 决策 1 文字倾向本任务，但评审重点 1-9 未列此项，报告未做项亦未登记），故按「不满足 TASK 验收 + 报告漏登记」列为 Important 并显式标注需归属澄清，不臆断为确定的范围违约。

---

## 评审结论

**不通过（2 Important）**——9 个评审重点中，重点 1/4（所列判据）/5/6/8/9 全部通过，**重点 3（幂等非 hash）明确判定通过**、**重点 7（环境替代证据充分性）明确判定充分（非真机验证）**；但存在 2 项重点覆盖范围之外/关联链路的 Important：事件读口缺租户/调用方过滤导致跨租户可读（F-01）、数据项事件 0029→`iot/open` 生产桥接缺失且端到端未验（F-02）。建议 F-01 必须修复（读口按 `identity.tenantId` / scope / appKey 过滤缓冲或按租户分区缓冲，并补两合法 appKey 读隔离测试）；F-02 先由协调者澄清归属，若属本任务则补桥接实现与端到端实证，若不属本任务则须在报告未做项中显式登记依赖与验收条件。修复/澄清后建议 scoped 复审。
