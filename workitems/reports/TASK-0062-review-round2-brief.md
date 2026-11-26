# 评审 brief：TASK-0062 scoped 复评 round2（修订轮）

> 隔离修订 diff：2755060..7886325，3 文件全 backend iot/alarm（AlarmStore + AlarmRecord 注释 + 真链路 E2E 测试）。首评 C-1（Critical）+ I-1（Important）退回一轮，本轮为唯一复评。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」；scoped 复评只覆盖 C-1/I-1 修复 + 回归确认）

## 必读（只给路径，不粘正文）

- **本轮隔离 diff**：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0062-diff-round2.txt`（3 文件，202 行）
- 首评报告：`reports\TASK-0062-review.md`（C-1 H2 复现 + I-1 原文 + 3 Minor）
- 退回 brief：`reports\TASK-0062-revision-brief.md`（裁决 + 修复方案）
- 修订报告段：`reports\TASK-0062-report.md`（追加的「修订轮」段）
- 首评 diff + brief：`reports\TASK-0062-diff.txt` / `TASK-0062-brief.md`
- 权威证据：`backend/ot-core/.../MybatisPlusConfig.java:30/59`（全局 MetaObjectHandler 确实存在）、`resources/sql/upgrade/20261124_iot_alarm.sql:52`（alarm_id NOT NULL）、`backend/.../alarm/service/impl/AlarmNoticeOutboxNotifier.java`（notify 用 record.getId()）

## 越界反查

- 修订 7886325 恰 3 文件全 backend iot/alarm；区间 2755060..7886325 只此一提交；frontend/contract/ingest/既有模块零触碰。

## 复评重点

1. **C-1（Critical）真修复**：`AlarmStore.upsert()` 是否在写入后按唯一键 `SELECT id` 回填 `a.setId(id)`，且**新增路径与命中重复键路径都能拿到真实 id**；去重语义（重复事件不回退已确认状态：只刷新 detail/level/update_time/update_by，不改 status/handler/确认处理时间）是否保持不变；`AlarmRecordServiceImpl:93-94` 的 upsert→notify 顺序下，`notify()` 拿到的 `a.getId()` 必定非 null 且指向刚写的记录。
2. **E2E 测试有效性（红转绿）**：`AlarmGenerationEndToEndTest`（3 用例）是否**不手工 setId**、走真生成入口，断言告警主记录 1 行 + notice 1 行且 `alarm_id` 指向主记录；评审独立在修复前代码上应能复现红（`NULL not allowed for ALARM_ID`、notice 0 行）、修后绿；非恒真、非 mock 捷径。
3. **I-1 措辞纠正**：`AlarmRecord.java:9` / `AlarmStore.java:23` 注释是否改为「本域走 JdbcTemplate 原生 SQL、不经 MP mapper，显式写审计四件套（框架 MetaObjectHandler 仅对 MP mapper 生效）」——不再声称"工程无全局 MetaObjectHandler"；**实现代码（审计列写法）是否零改动**。
4. **3 Minor deferred + 附带核查**：3 Minor 如实登记未改；`create_time`/`created_time` 两列同值核实结论是否记录；backend 2516 用例 0F0E（2513 基线 + 3 新增）、既有零回归；范围仅 3 文件。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0062-review-round2.md`。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。对 C-1/I-1 是否闭合、是否有新 C/I 给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
