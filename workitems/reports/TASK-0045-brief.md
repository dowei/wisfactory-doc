# 实施 brief：TASK-0045（指标聚合调度与物化存储 st_metric_value）

## 批次

- TASK-0045（本批共 1 个，标准任务 4 人天；计算引擎道 calc/**——前置 0044✅/0041✅，指标结果写入方，解锁 0060 指标看板 + 0053 推送订阅 + 0046 迟到修正）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `calc/**`（指标聚合模块：窗口生成/聚合物化/完整度 + 调度）
  - 不碰 backend/frontend/ingest/契约文件/workitems（报告除外）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0045.md`
- 决策：FEAT-0015/REQ-0025/TC-0015/ADR-0010（月指标月末一次产出）
- **调度裁决（协调者，开工必读）**：TASK 原文写"XXL-Job 调度"，但 calc 是**独立 Spring Boot 4 工程**（无 XXL-Job 依赖、无 backend sys_job 访问）——按 0041 PeriodicScheduler 先例，**calc 内自实现调度线程**（@Scheduled 或自实现，javadoc 说明取向），配置/env 提供改频能力；**不引入 XXL-Job 依赖、不改 backend sys_job**——登记与任务描述的字面差异，不视为偏离缺陷
- 上游交付（开工先读）：reports\TASK-0044-report.md（指标定义查询+快照纳入——本任务聚合口径的数据源）+ TASK-0042-report.md（st_item_value 同构写+六元组 tag+逐语句 REST——st_metric_value 写入同款纪律）+ TASK-0041-report.md（PeriodicScheduler 调度范式）+ calc/src/** 现有 executor/operator/status 代码
- 契约：`resources/contract/` 超级表 v1.0.0（st_metric_value 标签：metric_code/instance_code/period/calendar_type——tengine-super-tables.md）+ 详细设计 §1.7 指标聚合口径
- 0044 验证发现：对外 metrics/query 返回空集=st_metric_value 无写入方——**本任务正是补这个写入方**

## 接口与决策（逐条结论，不再讨论）

1. **窗口生成接口**：自然日历实现（HOUR/DAY/MONTH/YEAR 对齐真实日历边界，非固定秒数）；月指标**月末一次产出**（ADR-0010）；窗口以 (tenant, metricCode, periodStart, calendarType) 唯一标识；自实现调度线程按窗口到期触发，宽限期 3 天可配（env `CALC_METRIC_GRACE_DAYS` 默认 3）——窗口结束后宽限期内到达的数据可纳入（与 0046 迟到重算接缝，本任务只提供窗口接口）
2. **聚合计算**：小时→日→月→年逐级聚合（级联 src_type）；**比值型分子分母分别聚合再相除**（不是逐条比值再平均）；**平均口径=总量÷24**（日级小时均）；读数走已物化结果不现算（0042/0049 ResultSink 数据 + Redis）；**无数据出无值+完整度 0，不写 0**（CONV-02 不写假值）
3. **完整度**：应到周期值到达率——非 GOOD 质量码计缺失、停用实例/数据项剔除；阈值 80% 可配（env `CALC_METRIC_COMPLETENESS_MIN` 默认 0.8）；低于阈值的结果行**带不完整标记**（不静默）
4. **物化写 st_metric_value**：标签含 metric_code/instance_code/period/calendar_type（按超级表契约逐字）；**full_key 规范无 `full:` 前缀**（0049 F-02 教训）、六元组 tag 逐值精确断言、逐语句 REST（0029 多语句信封静默空操作纪律）；结果行含完整度字段；同 ts 覆盖写（重算幂等）；failure WARN 不阻塞
5. **结果可观测**：/calc/v1/status 增指标聚合观测视图（窗口/完整度/产出数）；不写 Redis 指标缓存（0053 推送/0060 看板后续消费 st_metric_value 直接查）
6. **TDengine 环境**：REST 6041 当前故障（15s 超时/reset，服务器侧，TD-0053 登记）——写路径做契约测试（假适配器捕获 INSERT SQL+六元组 tag 逐值断言）、真机写入+读回列为环境恢复后交付条件，不阻塞代码收尾（与 0042/0049 同口径）
7. **不碰**：0046 迟到修正重算（后续）、0053 推送（后续）、0060 指标看板（前端后续）、backend 任何代码、契约冻结

### 测试口径（TDD）

Contract 测试（窗口自然日历边界/月末一次/宽限期/级联聚合/比值分子分母分别聚合/平均÷24/完整度口径含非 GOOD 剔除与阈值标记/无数据不写 0/st_metric_value 六元组 tag 精确断言）；既有 0041/0042/0049 测试零回归；红转绿留痕。

## 验收命令

环境事实（2026-11-24）：Redis 宿主 phpStudy 3.0.504@6379；TDengine REST 故障（真机补验列条件，TD-0053）；8095 归 calc（探测/用毕即停）；并行前端道 0059 在途（不碰 frontend）。

1. `cd calc && mvn test` → BUILD SUCCESS、0 失败 0 错误（195 基线+新增，Results 段贴回）
2. live 冒烟（每步原始输出贴回）：calc 起（8095）→ 构造指标数据（比值型+平均型+级联型）→ 观察聚合产出（st_metric_value INSERT SQL 契约断言）→ 完整度阈值标记/无数据不写 0 → 月末窗口口径 → 全停；真机读回列环境恢复后条件
3. 范围核对：`git diff --name-only <BASE>..HEAD` 全落 calc/**

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0045-report.md`。五项齐全 + 调度裁决登记 + TDengine 真机补验条件 + 一句话结论。

## 契约

- no-subagents；只改 calc/**；提交只 add calc/**；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）；契约冻结发现问题报告并停。
- TDD 红转绿，每个修复附「为什么现在对了」。
