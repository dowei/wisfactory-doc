# 评审 brief：TASK-0045 diff 评审（代码评审）

> 实现提交 4dd8144（BASE=e49fea8，隔离区间 86a1c09..4dd8144 剔除并行 0059 frontend；21 文件 calc/**）——指标聚合调度与物化 st_metric_value（解锁 0060 指标看板）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0045-diff.txt`（86a1c09..4dd8144 限 calc/，21 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0045-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0045-report.md`（含三处字面差异登记：XXL-Job→自实现/四元组 tag/TDengine 真机未验真）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0045.md`
- 决策：FEAT-0015/REQ-0025/TC-0015/ADR-0010（月指标月末一次）
- **契约权威**：`resources/contract/tdengine-super-tables.md`（st_metric_value 四元组 tag：tenant_id/metric_code/instance_code/period + 4 列 ts/value/yoy/mom——执行者据此裁决 brief 的"六元组/日历口径 tag/完整度列"为词汇混用）
- 上游：reports\TASK-0044-report.md（指标查询+快照纳入）+ TASK-0042-report.md（st_item_value 写纪律）+ TASK-0041-report.md（PeriodicScheduler 范式）+ calc/src/**
- 详细设计 §1.7 指标聚合口径（PDF pypdf 可提取）

## 越界反查（协调者已比对，评审者复核）

- 21 文件全落 `calc/**`；区间内 0059 frontend 提交（86a1c09）已隔离剔除；backend/frontend/ingest/契约零触碰（契约文件未改实证）。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **聚合正确性（最关键）**：比值型分子分母**分别聚合再相除**（非逐条比值再平均——live 实证 30/7≠逐条均值）；日级平均=总量÷24（live 23.0≠24.0）；级联 SUM；月窗口 ts=月初（ADR-0010 月末一次）；**无数据 value=NULL+完整度 0 不写 0**（CONV-02 不写假值）；同 ts 覆盖幂等
2. **完整度口径**：非 GOOD 质量码计缺失、停用剔除；60% 有效数据带不完整标记（live 实证）；阈值 80% 可配；**完整度没有物理列（契约四列无此列）——由内存结果+/calc/v1/status 承载**——核实该裁决清晰、不丢失可观测性
3. **契约差异裁决核实（重点）**：执行者称冻结契约 st_metric_value 只有四元组 tag（tenant_id/metric_code/instance_code/period）+4 列，brief 的"六元组/calendar_type tag/完整度列"是 st_item_value 词汇混用——**评审者必须读 resources/contract/tdengine-super-tables.md 逐字核实**：若执行者正确，这是正确坚持契约；若契约其实有 calendar_type tag/完整度列而执行者漏写，则 Important
4. **调度裁决**：自实现（照 0041 PeriodicScheduler）而非 XXL-Job——核实不改 backend sys_job、不引 XXL-Job 依赖、env 改频能力；月/年窗口自然日历边界正确（非固定秒数）
5. **物化写纪律**：逐语句 REST（0029 多语句信封静默空操作）、full_key 规范无 `full:` 前缀（0049 F-02）、tag 逐值精确断言；st_metric_value 与 st_item_value 写入并存不冲突
6. **TDengine 真机未验真**：REST 故障下契约测试+假适配器证据充分性（与 0042 同口径）——**证据充分判通过，真机补验列环境恢复后条件但不得宣称已验真**
7. **测试有效性**：220 测试 0F（195 基线+25）；live RESULT fail=0 全序列与断言对应；4 处 TDD 红转绿（含 live NPE 修复）留痕；区间混入 0059 提交的双 BASE 登记清晰
8. **零回归与范围**：0041/0042/0049 既有测试零回归；commit 仅 21 文件 calc/**

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0045-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。**特别对第 3 点（契约差异裁决）给出明确判定**。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
