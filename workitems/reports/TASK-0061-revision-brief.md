# TASK-0061 diff 评审退回修订 brief（一轮退回，仅 I-1）

> 首评 `TASK-0061-review.md` 判 1 Important（I-1）+ 4 Minor（deferred）。本轮只修 I-1，其余 Minor 记 deferred 不改。续用原执行者上下文（agent_a727f843）。

## 必读（只给路径，不粘正文）

- 首评报告：`reports\TASK-0061-review.md`（I-1 原文 + 6 项特别关注点判定 + Minor 清单）
- 实施 brief + 报告：`reports\TASK-0061-brief.md`（D2/D3 巡检三类清单要求）、`TASK-0061-report.md`
- diff：`reports\TASK-0061-diff.txt`
- 待修文件（核心）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/inspect/scheduler/InspectTaskRunner.java`
- 现成聚合纯函数：`.../inspect/vo/InspectPayloads.java`（`groupCalcErrors(List<CalcLogVO>)` :68、`approximateLowCompleteness(List<CalcStateVO>)` :93、`onlineRateText` :30）
- 数据源可行性证据：`.../inspect/support/JdbcInspectSources.java`（已证明系统级 JdbcTemplate 按 tenantId 直接查表可行）、`resources/sql/upgrade/20261124_iot_calc_schema.sql:22-54`（iot_calc_state/log 均有 `tenant_id` 列）
- Calc VO：`backend/ot-admin/.../iot/calcread/CalcLogVO.java`、`CalcStateVO.java`（只读复用其结构，不改 calcread 模块）

## I-1（Important，必须修）：巡检任务必须真正产出三类清单，不得恒空集/占位

**首评认定（已独立核实，成立）**：`InspectTaskRunner.java` 中 `safeCalcErr()`/`safeCompleteness()` 恒 `return List.of()`，`onlineSnap = List.of()`，TENANT_DB_MISMATCH 无条件写死「待核验」；`InspectPayloads.groupCalcErrors()`/`approximateLowCompleteness()` 全 `src/main/java` 无生产调用（仅单测调用）。真机每轮巡检只落空集+占位，看板三 tab 长期空，未满足 REQ-0031 验收 2 / TC-0020 步骤 3（巡检任务一次运行产出三类清单+标签租户比对清单）。

**根因是纯实现缺陷，不是计划缺陷**：runner 构造器已注入 `JdbcTemplate`（可直接按 tenant_id 查表），执行者在注释里称「需 0063 显式租户重载 CalcReadService，当前 fail-soft 占位」——但 **CalcReadService 需要会话租户是因为它走会话裁剪；巡检是系统级任务，完全可以直接用已注入的 JdbcTemplate 查 `iot_calc`.`iot_calc_state` / `iot_calc_log` 表的 tenant_id 列**（DDL :24/:39 明确有该列），绕开会话租户问题。brief D2 已明确要求三类清单产出，只有 TENANT_DB_MISMATCH 因 TDengine REST 故障允许「待核验」占位；CALC_ERR/COMPLETENESS/在线率用空集捷径不符合 D2。

**修复方案（协调者裁决，可执行）**：
1. **CALC_ERR 清单**：在 InspectTaskRunner 用 JdbcTemplate 查 `iot_calc.iot_calc_log`（`WHERE tenant_id=? AND result_status=2 AND started_at>=?` 取最近窗口，如最近 24h 可配），映射成 `List<CalcLogVO>`，调用现成的 `InspectPayloads.groupCalcErrors(logs)` 按数据项/原因聚合 → 落 `TYPE_CALC_ERR`。查询异常才 fail-soft 空集 + WARN（真实降级，不是无条件空集）。
2. **完整度低清单**：查 `iot_calc.iot_calc_state`（`WHERE tenant_id=?`），映射成 `List<CalcStateVO>`，调用 `InspectPayloads.approximateLowCompleteness(states)`（按滞后/异常停算近似，页面已注明口径）→ 落 `TYPE_COMPLETENESS`。
3. **在线率快照**：按 tenant_id 从点位表/状态源聚合计数（分子在线/分母总数，**禁逐点扫 TDengine**——可用 `iot:st` Redis 键计数或 point 表 status 聚合，仿 JdbcInspectSources 的只读口径），落 `TYPE ONLINE_RATE` 快照（供 D1 看板 onlineRate 读）。无点位时 total=0 → 看板 N/A（保持 REQ-0031 验收 6）。若确实无可聚合数据源，至少要用真实查询得出 total=0，而不是写死空集。
4. **TENANT_DB_MISMATCH**：继续按 brief D2.4 允许「待核验」占位（TDengine REST 故障），保持不变。

**测试口径（TDD 红转绿）**：
- 新增契约测试（H2 或 mock JdbcTemplate）：给定 calc_log 含 2 个 fullKey 各若干 `result_status=2` + 不同 error_msg，`run()` 落库的 `TYPE_CALC_ERR` payload 经 `groupCalcErrors` 聚合出**非空、按原因分组**的清单；给定 calc_state 含滞后/停算行，`TYPE_COMPLETENESS` 落**非空**清单；给定点位/状态样本，`ONLINE_RATE` 落真实分子/分母。
- 断言 runner 确实调用聚合函数（非空结果），旧版恒空集必红。
- 保留原有 fail-soft 测试（数据源异常→空集+WARN），但区分「真实降级」与「无条件空集」。

## Minor（deferred，本轮不改，报告如实登记）

- M-1 `safeCalcState()` 非 allTenant 一刀切空集 → 租户管理员成功率恒 N/A（fail-closed 无泄露，可用性受损，演进位）
- M-2 history 分页 `LIMIT page*size` + subList 跨页重复/漏（前端当前固定 pageNum=1 未暴露）
- M-3 报告「0 ingest 引用」措辞与 diff 包 2 处 ingest 注释不符（0 文件改动属实，仅报告措辞）
- M-4 单一 iot:instance:query 码无法区分运营看健康/租户管理员看明细（权限分层演进位）

## 验收命令（沿用原始，必须原样贴回）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（含新增 I-1 清单产出契约测试），贴 Results 段
2. frontend 不受影响（本轮只改 backend InspectTaskRunner + 测试），确认 `pnpm build:test` 仍绿即可（可选，若本轮未碰前端可注明跳过）
3. 范围核对：`git diff --name-only 60c4568..HEAD` 只落 backend inspect（InspectTaskRunner.java + 相关测试），不碰 frontend/contract/ingest/既有模块

## 交付

- 修订提交（注明 TASK-0061 修订轮 I-1）；报告追加「修订轮」段：I-1 修复（真实取数+聚合调用+非空断言红转绿）、全量 Results、为什么现在对了、Minor deferred 登记。
- no-subagents；只改 backend inspect 范围；不置 TASK 已完成；报告写到 `reports\TASK-0061-report.md`（追加段）。
