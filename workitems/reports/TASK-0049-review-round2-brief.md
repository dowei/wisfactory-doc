# 评审 brief：TASK-0049 scoped 复评 round2（代码评审）

> 修订轮复评（§13：只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为终审。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-diff-round2.txt`（fded29b..331f860 限 calc/，单提交 6 文件）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-review.md`（F-01 真写入证据不足 + F-02 full_key `full:` 前缀契约错位 + F-03/04/05/06/07 Minor）
- 修订轮报告段：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-report.md`「修订轮」段（R1 红绿/R4 真机补验条件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-brief.md`
- 契约：`resources\contract\config-snapshot-schema.md:165`（full_key=instance_code+"."+data_code）+ 读侧 OpenApiTimeseriesClient
- 需要上下文时读工作区源码（D:\workspace\WisFactory\calc\...）

## 越界反查（协调者已比对，评审者复核）

- 修复 diff 6 文件全落 calc/status 域+README；区间仅一提交无交错；backend/frontend/ingest/契约零触碰。

## 复评要点（只答三问）

1. **F-02 是否真修复**：full_key 从 `'full:{instance}.STATUS'` 改为 `{instance}.STATUS`——六元组 tag（tenant_id/full_key/instance_code/data_code/category/source_type）**逐值精确断言**是否真实落测试（非仅 SQL 形态）；与 config-snapshot-schema.md:165 及读侧 OpenApiTimeseriesClient 查询口径逐字一致
2. **F-01 是否充分处置**：taosAdapter 应答解析抽出 requireTaosSuccess/extractCode + 真应答样本离线单测（code:0/9728/空格变体）——旧 contains 实现对空格变体必红是否成立；**真机写入+读回验证登记为环境恢复后交付条件（报告 R4）**是否清晰、不误导"已验真"；环境 REST 仍故障（协调器实测 15s 超时）非代码问题
3. **修复 diff 是否引入新 Critical/Important**：F-03 isOffline 解析 online 字段的 6 形态与恢复路径（offline=false→RULE）是否正确、不破坏"LWT 离线优先"；F-05 文档顺序订正与代码/ADR-0010 一致；163 测试零回归（159 基线+4）；F-04/F-06/F-07 仅 deferred 留痕未改代码

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-review-round2.md`。逐条回答三问（结论+出处+一句话说明）；新发现按 Critical/Important/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论（真机补验条件作为遗留项，不阻断代码收尾）。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
