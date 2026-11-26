# 评审 brief：TASK-0063 scoped 复评 round2（修订轮）

> 隔离修订 diff：fb8f66d..640c9bd，4 文件（DDL + CalcReadServiceImpl javadoc + 2 契约测试）；区间内 0084（2fc2a4d，ingest/**）与 0063 主体（fb8f66d）已隔离。0063 已用尽「1 退回 + 1 复评」上限，本轮为终审。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」；scoped 复评只覆盖修订范围 + 回归确认）

## 必读（只给路径，不粘正文）

- **本轮隔离 diff**：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0063-diff-round2.txt`（fb8f66d..640c9bd，4 文件）
- round1 评审报告：`reports\TASK-0063-review.md`（F-1 Important / F-2 Minor 原文与判定）
- round1 评审 brief：`reports\TASK-0063-review-brief.md`
- 修订报告段：`reports\TASK-0063-report.md`（追加的「修订轮」段：切分实证、javadoc 订正、全量 Results、红转绿）
- 实施 brief：`reports\TASK-0063-brief.md`
- 权威 SQL 通道：`backend/scripts/sql-exec.sh`（按 `;\n` 切分）
- DDL 全文：`resources/sql/upgrade/20261124_iot_calc_schema.sql`（工作区最终态）

## 越界反查

- 修订区间 fb8f66d..640c9bd 只含 4 文件，全落本任务声明范围（DDL + calcread 包）；0084 的 2fc2a4d 为并行 ingest 提交，不在本 diff；backend 其余模块/frontend/calc/契约零触碰；DDL 仍未在用户 phpStudy MySQL 执行。

## 复评重点

1. **F-1（Important）真修复**：DDL `:64/:73` 的 rw/ro 建号段 `PREPARE/EXECUTE/DEALLOCATE` 是否逐条独立成行，与 `sql-exec.sh` 的 `;\n` 切分契约一致；`IotCalcDdlSplitContractTest` 是否真实复刻切分算法、对旧脚本必须红、修后绿；是否新增「每行最多一条语句」机械断言覆盖脚本全文；只读探针是否实证通道可用且**未执行任何 DDL / 未创建任何对象**。
2. **F-2（Minor）订正**：`CalcReadServiceImpl` javadoc 是否改为「单段 fullKey 仅 allTenant 可见、受限角色 fail-closed 裁掉」，与 `visible()` 实装和真实写入口径一致；`CalcReadDataPermContractTest` 是否走真实 `visible()` 路径（非 deps=null 捷径）锁定受限角色裁单段/越权行、allTenant 全通过。
3. **回归确认**：backend 2461 测试 0F0E0Skip（2457 基线 + 4 新增）、calc 288 测试 0F0E3Skip（基线不变）的报告证据是否可信；0047/0050/0053 既有测试零回归。
4. **终审红线**：是否引入新的 Critical/Important；DDL 是否仍保持「只产出不执行」纪律；范围是否仍仅 4 文件、无新增越界。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0063-review-round2.md`。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。对 F-1/F-2 是否闭合与是否有新 C/I 给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
