# 评审 brief：TASK-0047 scoped 复评 round2（代码评审）

> 修订轮复评（§13：只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为终审。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0047-diff-round2.txt`（f19edc1..ea34d02 限 calc/，10 文件）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0047-review.md`（F-03 租户硬编码+冒烟值泄漏 / F-01 状态2 空接线 / F-02 CalcStateStore 硬编码 + F-04/05 Minor）
- 修订轮报告段：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0047-report.md`「修订轮」段
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0047-brief.md`
- 需要上下文时读工作区源码（D:\workspace\WisFactory\calc\...）

## 越界反查

- 修复 diff 10 文件全落 calc/**；区间仅一提交；backend 0053 改动已隔离；frontend/ingest/契约零触碰。

## 复评要点（只答三问）

1. **F-03 是否真修复**：全仓 grep 确认生产代码零 `"t47-live"` 字面量；tenant 从请求贯穿 RecalcContext/计算/状态键；`getTask(tenant,taskId)` 校验归属跨租户 404——live 实证 tenantA/B 状态 tenantId=1/2、跨查 404；**是否仍有其他硬编码租户/冒烟值残留**（全 diff 扫描）
2. **F-01 是否真修复**：env `CALC_CALC_LAG_THRESHOLD_MS` 真正传入状态机（非仅 log）；状态查询前调 evaluateLag 产生 calc_status=2；CalcStatusVo.from 阈值参数用于 lagging 标志——live 200ms 阈值实证 calc_status=2/滞后/lagging=true；对旧实现必红
3. **F-02 是否真修复 + 无新 C/I**：CalcStateStore 改 @Bean+构造注入（不再 new InMemory）；`@TestFactory` 动态测试同套契约跑内存与 JDBC 替身——**0063 换实现零代码改动是否真达成**；275 测试零回归（265 基线+10）；F-04/05 deferred 留痕真实

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0047-review-round2.md`。逐条回答三问（结论+出处+一句话说明）；新发现按 C/I/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
