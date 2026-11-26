# TASK-0046 scoped 复评 round2 报告（代码评审 · 终审）

- 角色：fresh 代码评审子代理（口径 `.agents/protocol/L1-execute.md` §13「diff 评审」复评条款）
- 复评范围（严格 scoped）：仅修复 diff `f8421c7..ebd443d`（限 `calc/`，9 文件）+ 原评审 F-01/F-02；不重开原评审已通过部分，未派 subagent，未修改任何文件
- 依据：brief 指定「必读」四项 + 工作区源码（`D:\workspace\WisFactory\calc\...`）+ 契约权威 `resources/contract/unified-message-model.md` v1.1.0 / `push-payload-evaluation.md` / `tdengine-super-tables.md`
- 日期：2026-11-24
- fresh 声明：未共享协调者上下文；判断不了的项标「无法判定」

## 独立复核的基础事实（非仅采信执行者报告）

1. **范围与区间**：`git log --oneline f8421c7..ebd443d` 仅一提交（ebd443d）；`git diff --name-only f8421c7..ebd443d` 9 项，`grep -v '^calc/'` 为空 → 越界 0，无交错。✓
2. **测试**：独立执行 `cd calc && mvn test -q` → `EXIT=0`；聚合 surefire XML（34 类）→ `tests=244 failures=0 errors=0 skipped=3`，GraphBuilderTest 23 / MetricAggregatorContractTest 8 / LateArrivalRecalcContractTest 6 / MetricCompanionValueContractTest 6 基线计数未变，新增 CascadeRecalcContractTest 2、MetricProducedEventContractTest 7（原 7→9）。✓
3. **契约权威核对**：§3.5 推送行（`unified-message-model.md:61`）明文 9 扩展字段；`push-payload-evaluation.md:40-54` 字段级比对表无冲突；CONV-02 质量码 0=正常 / 7=汇总不完整（`tdengine-super-tables.md:47-59`）。✓

---

## 复评要点逐条判定（只答三问）

### 问 1：F-01 是否真修复 —— **真修复（通过）**

- **出处**：`D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\metric\MetricCascade.java`（全文）；`MetricRuntime.java:126-137, 208-233`；`MetricAggregator.java:76-99`；`MetricValueStore.java:46-60`；`graph\GraphBuilder.java:290-340`；测试 `D:\workspace\WisFactory\calc\src\test\java\net\otlab\calc\metric\CascadeRecalcContractTest.java`。
- **一句话说明**：级联链已改为「MetricCascade 映射 0040 图（METRIC 节点 + METRIC_SOURCE 边）→ 复用 `GraphBuilder.topoSort` 取传递闭包拓扑序 → 下层先算并登记 materialized、上层经 `aggregateCascade`/`materializedCascadeValue` 从下级物化结果读新值」，三级链 M_LOW←M_HIGH←M_TOP 补传后均=7 的端到端断言对旧实现（单层平扫 + 从 values 表读值 → 上级 null）必红、对修复必绿。

逐子项核对：

| 子项 | 判定 | 依据 |
| --- | --- | --- |
| 映射为 0040 图结构 | ✓ | `MetricCascade.buildGraph` 用 `NodeKind.METRIC` 建节点、`EdgeType.METRIC_SOURCE` 建「上级→下级」边（`from=def.metricCode, to=child`），并复用 `GraphBuilder.topoSort` / `dependenciesOf` 及 `DependencyGraph` 承载——与 0040 同结构同类，未另造类型 |
| 拓扑序复用 `GraphBuilder.topoSort` | ✓ | `MetricCascade.java:55` 直接调 `GraphBuilder.topoSort(nodes, edges)`；边向 from→to 使上级未决依赖数=下级数，Kahn 输出 M_LOW→M_HIGH→M_TOP（被依赖者在前），与 0040 同一实现 |
| 从下级 materialized 读值 | ✓ | `doCompute`（`MetricRuntime.java:129-132`）对 srcType=4 走 `aggregateCascade`；`aggregateCascade`（`MetricAggregator.java:84-91`）对每 sourceKey 调 `source.materializedCascadeValue(tenant, child, instanceCode, def.period(), ts)`；后者（`MetricValueStore.java:53-59`）以前缀 `tenant|child|instance|period|` + `endsWith("|"+ts)` 精确命中物化键。下级 `doCompute` 先 `recordMaterialized` 同 ts 覆盖，上级读到的是修正新值 |
| 两静态方法 public 化零行为改动（逐行核对） | ✓ | `git diff GraphBuilder.java` 仅两 hunk：`@@ -287,8 +287,9 @@` 与 `@@ -325,8 +326,8 @@`，改动仅为 Javadoc 增行 + `static`→`public static`；`topoSort`（`:292-327`）与 `dependenciesOf`（`:330-340`）方法体逐行未变；独立跑 GraphBuilderTest 23 项全绿 |
| 每层 try/catch 失败隔离不破坏既有语义 | ✓ | `cascadeRecalc`（`MetricRuntime.java:218-232`）逐 def `try { doCompute; append SUCCESS/NO_DATA } catch (RuntimeException) { append FAILED }`——单上级异常不冒泡阻断兄弟；对非级联路径无改动，成功语义（同窗覆盖、幂等由 recalcFired 保证）不变 |
| 级联图构造对 0040 图结构零破坏 | ✓ | `MetricCascade` 只**新建** DependencyGraph 并复用 0040 公开方法/节点边类；`GraphBuilder.build()` 及 0040 图构建逻辑零改动，仅可见性放大 |

### 问 2：F-02 是否真修复 —— **真修复（通过）**

- **出处**：`D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\event\MetricProducedEvent.java:11-25, 47, 66-67, 74-75, 102, 110`；测试 `D:\workspace\WisFactory\calc\src\test\java\net\otlab\calc\event\MetricProducedEventContractTest.java:51-89`；契约 `resources\contract\unified-message-model.md:61, 89-90`、`push-payload-evaluation.md:40-54`、`tdengine-super-tables.md:47-59`。
- **一句话说明**：`toPayloadMap()` 已逐字段齐备冻结契约 §3.5 推送行 9 扩展字段（含原缺的 `quality`、原缺的 `instanceKey`），quality 按 CONV-02 正常 0 / 不完整·无数据 7，javadoc 如实登记「camelCase/毫秒时间戳最终归 0053 定版」「中台按指标全键+窗口幂等、非 hash」，旧载荷对 9 字段断言必红成立。

逐字段对照 §3.5 推送行（9 扩展字段）：

| 契约字段（§3.5 snake_case） | 载荷键（本批 camelCase） | 齐备 |
| --- | --- | --- |
| metric_code | metricCode | ✓ |
| instance_key（即 full_key） | instanceKey（与 instanceCode 同值，过渡双字段并存） | ✓ |
| period | period | ✓ |
| window_start | windowStart | ✓ |
| window_end | windowEnd | ✓ |
| value | value | ✓ |
| yoy | yoy | ✓ |
| mom | mom | ✓ |
| quality | quality（CONV-02：0 正常 / 7 汇总不完整） | ✓ |

补充判定：

- **quality 语义**：构造函数 `this.quality = incomplete ? 7 : 0`（`:66-67`），`incomplete` 由聚合器「无数据或完整度<阈值」得出；7=汇总不完整、0=正常与 CONV-02 枚举一致，不写 0 冒充正常；测试 `qualityReflectsCompletenessAndNoData` 断言不完整/无数据=7。✓
- **javadoc 如实登记**：`:17-20` 明文「camelCase/毫秒时间戳——契约第 7/8 条『待 TASK-0053 落地时定版』…最终字段名风格与时间类型归 0053 定版」，与契约 `unified-message-model.md:89-90` 及 `push-payload-evaluation.md:77-79` 遗留 1/2 一致；`:24-25` 明文「按指标全键+窗口幂等覆盖（非 hash——hash 随修正变化）」，与 ADR-0010 TD-0030 及契约推送行幂等口径一致。✓（原 round1「逐字对齐 ADR-0007 14 字段」表述已作废，本批不再主张。）
- **instanceCode 过渡双字段**：`instanceKey` 与 `instanceCode` 同值（测试均断言 "I01"），javadoc/测试明示 0053 消费侧以 instanceKey 为准、instanceCode 为过渡。契约把命名/类型列为 0053 未定版项，本批属允许范围并已留痕，不构成字段缺失。✓
- **旧载荷必红成立**：`git show f8421c7:...MetricProducedEvent.java` 的 `toPayloadMap` 无 `instanceKey`/`quality` 键；新测试 `payloadContainsAllNineFrozenContractExtensionFields`（断言两键存在）、`qualityReflectsCompletenessAndNoData`（断言 quality 值）对旧载荷分别触发断言失败/NPE——必红成立，与报告 RED 证据一致。✓

### 问 3：修复 diff 是否引入新 Critical/Important —— **无新 Critical/Important（通过）**

- **出处**：修复 diff 9 文件（见问 1/2 各出处）；`TASK-0046-report.md`「修订轮」段；`D:\workspace\WisFactory\calc\src\main\java\net\otlab\calc\metric\CompanionValueCalculator.java:84-100`。
- **一句话说明**：独立复跑 244 测试 0F0E3Skip、240 基线逐类零回归，F-03 注释订正与实现一致，F-04~07 deferred 留痕真实，EMQX 替代证据结论不因本修订改变，级联图构造对 0040 图零破坏，未发现新增 C/I。

| 子项 | 判定 | 依据 |
| --- | --- | --- |
| 244 测试零回归 | ✓ | 独立 `mvn test` EXIT=0；surefire 聚合 244/0/0/3（基线 240 + 级联 2 + 事件 2），34 类；3 Skip 仍为 0041 env 门控 1 + 0049 env 门控 2 |
| F-03 注释订正 | ✓ | `CompanionValueCalculator.compareTs` javadoc（`:84-88`）改为「同比=本期 ts 直接 minusYears(1)、环比=回退一周期」，与实现（`:100` `mode==YOY? t.minusYears(1):prev`）一致，F-03 Minor 闭合 |
| F-04~07 deferred 留痕真实 | ✓ | 报告「修订轮五」逐条登记（F-04 随 0053、F-05 已由 live 步骤 3 覆盖、F-06 javadoc 强化按全键+窗口、F-07 真库接线时补），均为 Minor deferred；修复 diff 未引入与之矛盾的实现 |
| EMQX 环境替代证据结论不因本修订改变 | ✓ | 修复 diff 未触碰 `PahoMqttChannel`/`MetricEventPublisher`/装配与 broker env（仅载荷 map + javadoc），round1「重点 6 充分（协议层真验证、生产栈真实、复验条件清晰）」结论的传输面依据不受影响；载荷字段变化由单测覆盖；真机复验仍为用户侧条件 |
| 级联图构造对 0040 图结构零破坏 | ✓ | 见问 1 末子项；未改 `GraphBuilder.build()`，GraphBuilderTest 23 项零回归 |

**新发现（按级别分级）**：无 Critical、无 Important。以下 1 项 Minor（观察项，不影响通过结论）：

| 级别 | 出处 | 一句话说明 |
| --- | --- | --- |
| Minor（新·观察） | `CascadeRecalcContractTest.java:86-106`（`cascadeIsolationOneBadParentDoesNotBlockSibling`） | 该用例名为「单上级异常隔离」，但所造 M_BAD 的 sourceKey "M_MISSING" 不在 defs 中 → `buildGraph` 不建边 → M_BAD 不在 M_LOW 受影响闭包内、且 `aggregateCascade` 返回 noData 不抛异常，故 try/catch 的 FAILED 分支实际未被触发；用例证明的是「非受影响节点不阻塞兄弟」而非「抛异常被隔离」。修复代码的 try/catch 本身存在且正确，仅测试未覆盖异常分支，建议后续补一def 抛 RuntimeException 的用例；非阻塞。 |

---

## 结论

**通过（scoped 复评 round2 · 终审）**。三问全过且无新 Critical/Important：

- **F-01 真修复**：级联改走 0040 拓扑序（`GraphBuilder.topoSort`）+ 从下级 materialized 读新值，三级链端到端旧红新绿；两静态方法 public 化经逐行核对为零行为改动；每层 try/catch 隔离不破坏既有语义。
- **F-02 真修复**：载荷逐字段齐备冻结契约 §3.5 的 9 扩展字段（含 quality、instanceKey），quality 按 CONV-02 0/7，javadoc 如实登记 0053 定版与「全键+窗口幂等、非 hash」；旧载荷必红成立。
- **无新 C/I**：244 测试 0F0E3Skip 独立复现、240 基线零回归，F-03 订正/F-04~07 deferred 留痕真实，EMQX 结论不变，0040 图结构零破坏。

按 §13 修订轮复评条款，F-01/F-02 两项 Important 均已闭合，可结束退回/修订路径。评审者未修改任何文件。
