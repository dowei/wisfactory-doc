# TASK-0062 diff 评审退回修订 brief（一轮退回：C-1 必修 + I-1 措辞）

> 首评 `TASK-0062-review.md` 判 1 Critical（C-1）+ 1 Important（I-1）+ 3 Minor（deferred）。本轮修 C-1（真修复）+ I-1（仅措辞，不改实现）。续用原执行者上下文（agent_3f0460c2）。

## 必读（只给路径，不粘正文）

- 首评报告：`reports\TASK-0062-review.md`（C-1 的 H2 端到端复现证据 + I-1 原文 + 3 Minor）
- 实施 brief + 报告：`reports\TASK-0062-brief.md`（D1 唯一键去重 / D3 站内出口）、`TASK-0062-report.md`
- diff：`reports\TASK-0062-diff.txt`
- 待修文件（核心）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/store/AlarmStore.java`（upsert :76 不回填主键）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/service/impl/AlarmNoticeOutboxNotifier.java`（notify 用 record.getId()）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/service/impl/AlarmRecordServiceImpl.java`（:93-94 生成链调用）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/entity/AlarmRecord.java` + AlarmStore.java 的「无全局 MetaObjectHandler」注释（I-1 措辞）
- 权威证据：`backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java:30/59`（**明确 implements MetaObjectHandler 并注入**）、`resources/sql/upgrade/20261124_iot_alarm.sql:52`（iot_alarm_notice.alarm_id NOT NULL）

## C-1（Critical，必修）：真链路站内提醒写不进库

**首评认定（已独立核实，成立）**：`AlarmStore.upsert()` 用 `jdbc.update(sql, args...)` 不回填自增主键，新增后 `record.getId()` 仍是 null；`AlarmRecordServiceImpl:93-94` 先 `upsert(a)` 再 `notifier.notify(tenantId, a)`，而 `AlarmNoticeOutboxNotifier.notify()` 用 `record.getId()` 写 `iot_alarm_notice.alarm_id`（DDL :52 为 NOT NULL）→ 真链路每次提醒 INSERT 必然失败、被 fail-soft 吞掉。评审用编译后真实类在 H2 端到端复现：`upsert returned=1, record.getId()=null`，notice 表 0 行；报告里「生成即写一条站内提醒」只在测试手工 `setId(11L)` 才成立。REQ-0031 验收 3 / TC-0020 步骤 4 不成立。

**修复方案（协调者裁决）**：让 `upsert()` 在新增后回填主键到 `a.setId(...)`，使后续 notifier 拿到真实 id。两种可选实现，任选其一但要正确：
1. 用 `SimpleJdbcInsert` + `usingGeneratedKeyColumns("id")` 执行 INSERT（新增路径），拿到 key；重复事件走 ON DUPLICATE KEY UPDATE 时另行按唯一键查回 id（或仍用 `latest(tenantId,type,objectCode)` 查回）——注意 upsert 语义要保持「重复事件不回退已确认状态」不变。
2. 保留原生 `INSERT ... ON DUPLICATE KEY UPDATE`，插入后用 `SELECT id FROM iot_alarm_record WHERE tenant_id=? AND alarm_type=? AND object_code=? AND occurred_at=?` 查回 id 并 `a.setId(id)`（利用唯一键，新增/命中都能拿到 id）。
- **关键**：不论哪种，必须保证 `notify()` 被调用时 `a.getId() != null` 且指向刚 upsert 的那条记录。
- 加一条**真链路端到端契约测试**（不手工 setId）：调用 `generate()`/生成入口 → 断言 `iot_alarm_record` 有 1 行 **且 `iot_alarm_notice` 有对应 1 行**（alarm_id 指向主记录 id）。旧版（id 不回填）必红，修后绿。
- 保留原有去重测试（重复事件不回退状态）零回归；fail-soft 语义（提醒写失败不阻断主记录）保留，但不再是"必然失败"。

## I-1（Important，仅措辞纠正，不改实现逻辑）

**首评认定（成立但不影响正确性）**：执行者在 `AlarmRecord.java:9`、`AlarmStore.java:23` 注释及报告称「工程无全局 MetaObjectHandler，MP @TableField(fill=) 不落值」——**失实**：`backend/ot-core/.../MybatisPlusConfig.java:30` 明确 `implements MetaObjectHandler`、`:59` 注入全局。但评审独立核实后承认**显式 JdbcTemplate 审计列实现本身正确**（本域用普通 POJO + 原生 SQL，不经 MP mapper，fill 确实不适用）。

**处理**：只改注释与报告措辞——改为「本域走 JdbcTemplate 原生 SQL、不经过 MP mapper，故显式写审计四件套（框架的 MetaObjectHandler 仅对 MP mapper 生效，对本域不适用）」。**不改任何实现代码**（upsert/confirm/handle 的审计列写法保持）。报告修订轮段纠正该判断。

## 3 Minor（deferred，本轮不改，报告登记）

- M-1 / M-2 / M-3：见首评报告原文，如实登记 deferred。

## 附带核查项（非缺陷，顺手确认）

- DDL `iot_alarm_record` 同时有 `create_time` 与 `created_time` 两列（注释称兼容查询习惯）——确认两列都被写入且同值（AlarmStore upsert 写 create_time；检查 created_time 是否有 DEFAULT 兜底），不产生 NULL/口径分裂即可；若发现不一致在报告登记，不作为本轮阻断。

## 验收命令（沿用原始，必须原样贴回）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（含新增真链路提醒落库契约测试），贴 Results 段
2. frontend 本轮未碰（仅改 backend alarm），确认 `pnpm build:test` 仍绿即可，或注明跳过并说明未碰前端
3. 范围核对：`git diff --name-only 2755060..HEAD` 只落 backend iot/alarm（upsert/notifier/service/注释 + 测试），frontend/contract/ingest/既有模块零触碰

## 交付

- 修订提交（注明 TASK-0062 修订轮 C-1）；报告追加「修订轮」段：C-1 修复（主键回填 + 真链路提醒落库红转绿）、I-1 措辞纠正、全量 Results、Minor deferred、附带核查结论。
- no-subagents；只改 backend iot/alarm 范围；不置 TASK 已完成；报告写到 `reports\TASK-0062-report.md`（追加段）。
