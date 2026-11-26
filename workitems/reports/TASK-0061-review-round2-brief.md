# 评审 brief：TASK-0061 scoped 复评 round2（修订轮）

> 隔离修订 diff：60c4568..24b5c44，2 文件全 backend iot/inspect（InspectTaskRunner + 契约测试）。首评 I-1（Important，巡检清单恒空集）退回一轮，本轮为唯一复评。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」；scoped 复评只覆盖 I-1 修复 + 回归确认）

## 必读（只给路径，不粘正文）

- **本轮隔离 diff**：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0061-diff-round2.txt`（2 文件，357 行）
- 首评报告：`reports\TASK-0061-review.md`（I-1 原文 + 4 Minor deferred）
- 退回 brief：`reports\TASK-0061-revision-brief.md`（协调者裁决 + 修复方案）
- 修订报告段：`reports\TASK-0061-report.md`（追加的「七、修订轮」段）
- 首评 diff + brief：`reports\TASK-0061-diff.txt` / `TASK-0061-brief.md`（D2 三类清单要求）
- 权威证据：`resources/sql/upgrade/20261124_iot_calc_schema.sql:22-54`（iot_calc_state/log 有 tenant_id 列）、`backend/.../inspect/vo/InspectPayloads.java`（groupCalcErrors :68 / approximateLowCompleteness :93）、`backend/.../inspect/support/JdbcInspectSources.java`（系统级按 tenantId 查表范式）

## 越界反查

- 修订 24b5c44 恰 2 文件全 backend iot/inspect；区间 60c4568..24b5c44 只此一提交（0084 修订 3e09775 在 60c4568 之前已隔离）；frontend/contract/ingest/既有模块零触碰。

## 复评重点

1. **I-1（Important）真修复**：`InspectTaskRunner` 是否用已注入的 JdbcTemplate 按 tenant_id 直接查 `iot_calc.iot_calc_log`（result_status=2）/`iot_calc.iot_calc_state`，映射 CalcLogVO/CalcStateVO 后调用现成 `groupCalcErrors`/`approximateLowCompleteness` 产出**非空** CALC_ERR/COMPLETENESS 清单；ONLINE_RATE 是否用 iot_point status 一条 SQL 聚合真实分子/分母（**禁逐点扫 TDengine**、无点位 total=0→N/A）；TENANT_DB_MISMATCH 保持待核验占位。核实**无无条件空集捷径**——旧版恒 `List.of()` 是否真正移除。
2. **fail-soft 语义**：查询异常才降级空集+WARN 是否与"无条件空集"明确区分（dim response 的边界）；派生的 tenant_id 是否强制来自租户遍历而非请求参数（系统级任务无会话）。
3. **测试有效性（TDD 红转绿）**：`InspectTaskRunnerContractTest` 4 项非空清单断言是否真实（CALC_ERR 3 log rows→2 reason groups、COMPLETENESS 3 state rows→2 low items、online=2/total=3），旧恒空集版必红、修后绿；非恒真、非 mock 捷径。java SQL 用 H2 或 mock JdbcTemplate 均可，但断言要落到落库 payload 内容。
4. **Minor deferred 与回归**：4 Minor（M-1 非 allTenant 空集/M-2 分页/M-3 报告措辞/M-4 权限码）是否如实登记 deferred 未改；backend 2486 用例 0F0E（2482 基线+4 新增）、既有测试零回归；范围仅 2 文件。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0061-review-round2.md`。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。对 I-1 是否闭合、是否有新 C/I 给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
