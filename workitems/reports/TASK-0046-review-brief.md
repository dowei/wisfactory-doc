# 评审 brief：TASK-0046 diff 评审（代码评审）

> 实现提交 f8421c7（BASE=bf84526，单提交 19 文件 calc/**）——迟到修正重算与同比环比伴随值 + 指标产出事件生产侧（解锁 0047 手动补算/0053 订阅消费）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-diff.txt`（bf84526..f8421c7，19 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-report.md`（含 EMQX 退化/本地夹具 broker 替代实证登记）
- TASK 详情 + FEAT-0015/REQ-0025/TC-0015/ADR-0010（同窗口重发）/ADR-0007（事件载荷）
- 上游：reports\TASK-0045-report.md（聚合物化+宽限期+yoy/mom deferred M-2）+ TASK-0041/0042-report.md + calc/src/**
- 契约：resources/contract/tdengine-super-tables.md（st_metric_value 冻结 4 列 ts/value/yoy/mom——**不改契约**）
- ADR-0007 指标产出事件载荷（workitems decisions 目录）——逐字核对 14 字段

## 越界反查（协调者已比对，评审者复核）

- 19 文件全落 `calc/**`；区间仅一提交无交错；backend/frontend/ingest/契约零触碰。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **迟到修正正确性**：宽限期窗口键幂等触发（不重复重算）、同 ts 覆盖写、级联 srcType=4 沿依赖图向上（复用 0040 拓扑序）、失败隔离；宽限判据方向（"窗口结束+3 天内到达"才算迟到，非反向）
2. **BACKFILL 留痕**：iot_calc_log §1.9A 全字段（task_id/trigger_type=BACKFILL/full_key=指标全键/period_start-end/is_recalc=1/input_summary/结果状态）；留痕写失败不阻塞计算；TDengine 不可用时的内存+契约测试口径
3. **yoy/mom 伴随值**：从已物化结果读（不现算 ADR-0011）、写 st_metric_value 既有 yoy/mom 列（不改契约 4 列）、缺同期/分母 0→null+原因**不写 0**（CONV-02）；级联指标 yoy/mom 随主值同步；live 实证 0.25/-0.2→0.875/0.2 同行覆盖
4. **事件生产侧（本任务承担 0053 生产侧）**：MQTT `iot/{tenant}/metric/out` + Kafka `iot.metric.result.{tenant}`，**载荷逐字对齐 ADR-0007 14 字段+hash**——live 实证"字段集/身份字段一致、无 is_recalc/calcTime"（评审者核实这两个字段确实不应在 ADR-0007 载荷里）；双通道同载荷、租户双隔离（主题+载荷）、QoS1 fire-and-forget 无订阅者不报错、Kafka env 默认关
5. **同窗口重发不变（ADR-0010 关键）**：修正后重发载荷与首次产出**逐字段一致**（除 hash 反映新值外无非预期变化）；事件由 0053 订阅侧消费——生产侧字段形态清晰可对接
6. **环境退化处置**：EMQX 1883 TCP 可连但 MQTT 握手无 CONNACK（brief 曾"已实证"时点退化）——用本地 MQTT 3.1.1 夹具 broker 完成真协议层 pub/sub 验证、生产侧 Paho 真实栈 env 可切、恢复后免改代码复验。**评估该替代证据充分性**（与 TDengine 同口径：协议层真验证+生产栈真实，环境恢复后真机复验条件清晰）
7. **测试有效性**：240 测试 0F（220 基线+20）；3 处 TDD 红转绿（同比窗回退/recorder 载荷登记/宽限判据方向+幂等）留痕真实；live 全序列与断言对应
8. **零回归与范围**：0041/0042/0045 既有测试零回归；commit 仅 19 文件 calc/**

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。**特别对第 4 点（ADR-0007 载荷逐字一致）与第 6 点（EMQX 环境替代证据充分性）给出明确判定**。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
