# 实施 brief：TASK-0046（迟到修正重算与同比环比伴随值）

## 批次

- TASK-0046（本批共 1 个，标准任务 3 人天；计算引擎道 calc/**——前置 0045✅，指标链收官任务，解锁 0047 手动补算/0053 推送订阅）
- BASE：开工 HEAD（git rev-parse --short HEAD 记录进报告）
- 改动范围（越界即违约）：calc/**；不碰 backend/frontend/ingest/契约/workitems（报告除外）

## 必读

- TASK 详情 + FEAT-0015/REQ-0025/TC-0015/ADR-0009（稳定序号）/ADR-0010（修正后同窗口重发）
- 上游：reports\TASK-0045-report.md（窗口接口/宽限期 3 天/聚合物化/完整度/st_metric_value 4 列 ts/value/yoy/mom——**yoy/mom 恒 null 为 0045 deferred M-2，本任务补上**）+ TASK-0041-report.md（双触发+ResultSink）+ TASK-0042-report.md（算子异常语义）+ calc/src/**
- 契约：resources/contract/tdengine-super-tables.md（st_metric_value 冻结 4 列 ts/value/yoy/mom+4 元组 tag——0045 已核实，本任务**不改契约**，yoy/mom 写入既有列）
- 详细设计 §1.9A（iot_calc_log BACKFILL 留痕链）+ §1.7（同比环比伴随值）

## 接口与决策（逐条结论）

1. **迟到检测与触发**：0045 宽限期（默认 3 天 env CALC_METRIC_GRACE_DAYS）内到达的数据触发**受影响窗口重算**——按数据项→所属指标→依赖图向上级联（复用 0040 图拓扑序），同窗口覆盖（同 ts 覆盖写 0045 既有语义）；重算触发来源：接入链补传（0029 原始 ts 落库）+ Redis 新值事件；不重复触发（窗口键幂等，0041 WindowIdempotencyGuard 同款）
2. **覆盖修正与留痕**：重算结果覆盖 st_metric_value 同 ts 行（is_recalc 语义由留痕记录承载，结果表 4 列无 is_recalc 列——不扩契约）；**iot_calc_log 留痕**按 §1.9A：task_id 落库、trigger_type=BACKFILL、full_key=指标全键、period_start/end、is_recalc=1、input_summary、结果状态——写独立 schema（如 iot_calc 库，按设计"独立 schema 与 iot_calc_state 同域"）；**TDengine 不可用时留痕走内存+契约测试**（同 0045 环境口径，真机补验列条件）
3. **yoy/mom 伴随值**：同比（去年同期同窗口）/环比（上一周期同窗口）从 st_metric_value 已物化结果读取（不现算），写入主值同行的 yoy/mom 列——**补 0045 deferred M-2**：缺同期数据时 yoy/mom=null+完整度标记（不造假）；级联指标的 yoy/mom 随主值重算同步更新
4. **修正后事件同窗口重发（ADR-0010 关键——本任务承担 0053 的事件生产侧）**：每次重算完成（含首次产出与迟到修正）发**指标产出事件**：
   - MQTT `iot/{tenant}/metric/out` + Kafka `iot.metric.result.{tenant}`（双通道；EMQX 192.168.31.145 真 broker 可用——1883 匿名 allow 曾实证；Kafka 本环境无 broker——**Kafka 侧留接口+契约测试，运行时按 env 开关默认关**，登记演进位）
   - **载荷权威（2026-11-24 round1 评审纠正——原"ADR-0007 14 字段"为 brief 笔误，以本节为准）**：冻结契约 `resources/contract/unified-message-model.md` v1.1.0「3.5」章推送行——**统一消息模型基础字段 + 9 个扩展字段**：metric_code、instance_key（即 full_key）、period、window_start、window_end、value、yoy、mom、**quality**——共 9 扩展字段，**quality 必须包含**（原实现缺 quality 是偏离）。字段名风格原文明确「待 TASK-0053 落地时定版」（契约 §修订说明第 7 条）——本批采用 camelCase/毫秒时间戳属允许范围，但须 javadoc 登记"最终风格归 0053 定版"。ADR-0007 只规定"在统一消息模型上扩展这些字段"，未定义 14 字段清单。
   - **修正后同窗口重发载荷不变**（ADR-0010——除 value/yoy/mom/hash 随重算变化外，身份字段/窗口字段/quality 语义保持一致，不引入 is_recalc/calcTime 等非契约字段）
   - 事件源与 0053 的订阅侧对接（0053 后续派——消费本任务发出的真实事件）
5. **租户隔离**：事件主题含 tenant、载荷含 tenant，**租户 B 收不到租户 A 事件**（订阅侧 0053 也校验，本任务生产侧先保证主题隔离）
6. **无订阅者不报错**：MQTT QoS1 publish fire-and-forget（0048 fail-soft 纪律），Kafka 无 broker 时 env 关闭不发
7. **不碰**：0047 手动补算接口（后续）、0053 订阅侧（后续消费本批事件）、backend 任何代码、契约冻结

### 测试口径（TDD）

Contract 测试（迟到宽限内触发/窗口幂等/级联重算/st_metric_value 同 ts 覆盖+yoy/mom 写入/BACKFILL 留痕字段/is_recalc=1/事件载荷 ADR-0007/同窗口重发载荷不变/租户主题隔离/无订阅者不报错）；既有 0041/0042/0045 测试零回归；红转绿留痕。

## 验收命令

环境事实：Redis 宿主 phpStudy 3.0.504@6379；EMQX 192.168.31.145:1883 匿名 allow 可用（已实证）；TDengine REST 故障（TD-0053，真机补验列条件）；8095 归 calc（探测/用毕即停）。

1. `cd calc && mvn test` → BUILD SUCCESS、0 失败 0 错误（220 基线+新增，Results 段贴回）
2. live 冒烟（每步原始输出贴回）：calc 起（8095）→ 补传迟到数据 → 窗口重算+st_metric_value 覆盖+yoy/mom 写入断言 → BACKFILL 留痕字段断言 → **外部 MQTT 订阅 iot/{tenant}/metric/out 真收事件**（载荷逐字段+同窗口重发载荷不变）→ 租户 B 主题无消息 → 全停
3. 范围核对：git diff --name-only BASE..HEAD 全落 calc/**

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-report.md。五项齐全 + 事件生产侧交付登记（供 0053 消费）+ TDengine/Kafka 补验条件 + 一句话结论。

## 契约

- no-subagents；只改 calc/**；提交只 add calc/**；不把 TASK 置「已完成」；不改 workitems（报告除外）；契约冻结发现问题报告并停。
- TDD 红转绿，每个修复附「为什么现在对了」。
