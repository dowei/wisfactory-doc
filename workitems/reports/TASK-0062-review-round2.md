# TASK-0062 scoped 复评（round2 · 终审）

- 角色：独立代码评审（fresh / no-subagents / 只评审不修改，唯一例外：本报告）
- 评审对象：`2755060..7886325` 隔离 diff（`TASK-0062-diff-round2.txt`，3 文件，单提交）
- 口径：L1-execute §13「diff 评审」；scoped 复评仅覆盖 C-1/I-1 修复 + 回归确认
- 终审约束：0062 已用尽「1 退回 + 1 复评」，本轮为唯一复评；闭合 C-1 与 I-1 且无新 C/I 即通过，否则不通过并列新发现

---

## 结论：**通过**

C-1（Critical）与 I-1（Important）均已闭合，本轮**无新增 Critical / Important**。按终审契约判 **通过**。3 Minor 维持 deferred，附带核查项（create_time/created_time 同值）已核实通过，均不阻断。

---

## 一句话总评

修订 `7886325` 在 `AlarmStore.upsert()` 写入后按唯一键 `(tenant_id, alarm_type, object_code, occurred_at)` 查回 `id` 并 `a.setId(id)`，独立核实新增路径与 ON DUPLICATE 命中路径都能拿到指向刚写行的真实主键，去重语义（不回退已确认状态、只刷新 detail/level/update_time/update_by）保持不变，`createAndNotify()` 的 `upsert→notify` 顺序下 `notify()` 必拿非 null 且正确的 id；新 E2E 用例不手工 `setId`、走真生成入口，评审在修复前代码（临时 worktree `2755060`）独立复现红（`NULL not allowed for column "ALARM_ID"`、notice 0 行），修后绿；I-1 两处注释已改为准确表述且实现零改动；范围恰 3 文件、区间干净，backend 全量 2516 用例 0F0E。

---

## C-1 闭合判定：**已闭合**

- **出处**：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/store/AlarmStore.java` 的 `upsert()`（修订后第 81–103 行）；调用链 `service/impl/AlarmRecordServiceImpl.java:91-100`（`upsert(a)` 后紧接 `notifier.notify(tenantId, a)`，未改）、`service/impl/AlarmNoticeOutboxNotifier.java:32-41`（用 `record.getId()` 写 `iot_alarm_notice.alarm_id`，未改）；DDL 权威 `resources/sql/upgrade/20261124_iot_alarm.sql:52` 该列 `NOT NULL`。
- **独立核实（回到源码，不依赖报告自述）**：
  1. **两条路径都回填真实 id**：`upsert()` 先执行 `INSERT ... ON DUPLICATE KEY UPDATE`，随后 `SELECT id ... WHERE tenant_id=? AND alarm_type=? AND object_code=? AND occurred_at=? AND deleted=0`，命中即 `a.setId(id)`。唯一键与查回条件完全对齐（DDL `uk_alarm_dedup(tenant_id,alarm_type,object_code,occurred_at)`，`:43`），新增与命中重复键都会命中同一行。评审另写探针测试（临时 worktree `7886325`，用后已删除）断言：①新增路径 `upsert` 前 `id=null`、后 `id` 等于库中行 id；②构造重复键 upsert（改 detail/level、DB 中先置 status=1/handler=42）后 `id` 仍等于既有行 id。绿（1 test 0F0E）。
  2. **去重语义未变**：ON DUPLICATE KEY UPDATE 仍只刷 `detail/alarm_level/update_time/update_by`，不碰 `status/handler_id/confirmed_at/handled_at`。探针测试断言重复事件后 status 仍 1、handler 仍 42、detail/level 被刷新、行数仍 1，绿。E2E 用例 `duplicateEventStillDedupedAfterIdBackfill` 亦断言确认态不被打回。查回 SELECT 的 `AND deleted=0` 在新增/命中即时读取场景无副作用——grep 确认本域无任何 `deleted=1` 软删写入路径（仅 `AlarmRecord.setDeleted`/ROW_MAPPER 读取），故不会漏查。
  3. **notify 拿到非 null 且正确 id**：`createAndNotify()` 顺序为 `store.upsert(a); notifier.notify(tenantId, a);`，`upsert` 内已回填，notifier 用 `record.getId()` 写 notice。E2E 用例断言 notice 1 行且 `alarm_id` 等于告警主记录 id、`target_role=platform_admin`，修后绿；探针测试对连续两次生成（含重复键）断言 notice 2 行且无 `alarm_id IS NULL`，绿。
  4. **fail-soft 保留**：notifier 的 `catch (RuntimeException)` 仅 WARN 不抛，未改动；修复前“必然失败”变为修后正常落库。
- **E2E 红转绿（评审独立复现，非照抄报告）**：
  - **RED**（临时 worktree `2755060` 修复前代码，拷入同一 E2E 测试文件实跑）：`Tests run: 3, Failures: 2, Errors: 0`。`gatewayOfflineGeneratesAlarmAndNoticeWithRealId` 期望 notice 1 行 but was 0；WARN 原文含 `NULL not allowed for column "ALARM_ID"`（H2 `[23502-240]`）；`createdTimeMirrorsCreateTime` 亦因两列默认值独立取值相差 18ms 而红。
  - **GREEN**（当前 `7886325`）：`mvn -pl ot-admin test -Dtest=AlarmGenerationEndToEndTest` → surefire `Tests run: 3, Failures: 0, Errors: 0`（2.203s）。
  - **测试有效性**：测试**不手工 setId**（全文 `setId` 仅出现在注释“不手工 setId”中，无代码调用），走真实生成入口 `raiseGatewayOffline → createAndNotify → upsert + notifier`，自建 H2 MySQL 模式两表 DDL（notice.alarm_id 为 `BIGINT NOT NULL`），断言主记录 1 行 + notice 1 行 + alarm_id 指向主记录；非 mock 捷径、非恒真（修前确红）。用例②验证去重不回退状态，用例③验证两时间列同值。
- **判定**：Critical 根因消除，真链路站内提醒可落库，REQ-0031 验收 3 / TC-0020 步骤 4 恢复成立，闭合。

## I-1 闭合判定：**已闭合（仅措辞）**

- **出处**：`entity/AlarmRecord.java` 头注、`store/AlarmStore.java` 头注（修订 diff 第 6–12、25–29 行）。
- **权威证据复核**：`backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java:30` 确为 `public class MybatisPlusConfig implements MetaObjectHandler`，`:59` `globalConfig.setMetaObjectHandler(this)` 注入全局——首评事实认定成立，工程确有全局 handler。
- **注释核实**：两处旧措辞“工程无全局 MetaObjectHandler，MP @TableField(fill=) 不会真正自动填充/不依赖 MP 自动填充”已改为“本域走 JdbcTemplate 原生 SQL、不经过 MP mapper，故显式写审计列——框架的 MetaObjectHandler（ot-core MybatisPlusConfig）仅对 MP mapper 生效，对本域不适用”，不再声称“工程无全局 MetaObjectHandler”，与源码事实一致。`git diff 2755060..7886325` 逐行核对：`AlarmRecord.java` 只改注释（5 行注释块）；`AlarmStore.java` 头注改注释，其余改动属 C-1 的 upsert 与 created_time（不属审计列写法变更）。`confirm()/handle()` 的显式审计列写法、`upsert()` 的 create_time/create_by/update_time/update_by 写法除新增 `created_time` 同写外未改（新增列属附带核查项，非 I-1 实现改动）。判定 I-1 闭合。
- **附带登记（不阻断）**：`resources/sql/upgrade/20261124_iot_alarm.sql:13` 的同类旧措辞仍在，但属 `resources/sql/**`，不在本轮“只改 backend iot/alarm”范围，修订报告已如实登记留后续订正；不影响实现与 I-1 闭合。

---

## 发现清单

本轮**无新增 Critical / Important / Minor（缺陷）**。以下为 deferred 与核查登记：

- **[Deferred·M-1]** `AlarmStore.upsert()` ON DUPLICATE 分支刷新 `alarm_level`，首评措辞只提 detail/时间——不违反状态不回退硬约束，报告 D1 已如实登记刷新字段集，本轮未改，维持 deferred。出处：`AlarmStore.java:89-90`；首评 `TASK-0062-review.md` M-1。
- **[Deferred·M-2]** 确认/处理写口沿用 `iot:instance:query`（与只读同码）——brief D3 明确允许（无 `alarm:*` 位，避免全角色 403），属权限模型层缺口，本轮未改，维持 deferred。出处：`IotAlarmController.java` 三端点；首评 M-2。
- **[Deferred·M-3]** confirm/handle 时间戳用服务端 `System.currentTimeMillis()`，前端按本地时区格式化——毫秒两端一致，仅提示部署环境时区统一，本轮未改，维持 deferred。出处：`AlarmRecordServiceImpl.confirm/handle`；首评 M-3。
- **[核查·通过]** create_time / created_time 两列同值：修订 `upsert()` 将同一 `now` 显式写入 `create_time` 与 `created_time`（`VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,0)` 对应 `now, now, createBy, now, createBy`），新增/命中均如此；E2E 用例 `createdTimeMirrorsCreateTime` 修后绿（两列毫秒值相等、非 NULL、>0）；生产 DDL 两列仍各带 `DEFAULT CURRENT_TIMESTAMP` 兜底（`:36-37`），应用时由显式写覆盖。首评复现的 RED（两列相差 18ms）在修后消失。结论：口径不分裂，无 NULL，通过。
- **[核查·通过]** backend 全量：`mvn -pl ot-admin test` → `Tests run: 2516, Failures: 0, Errors: 0, Skipped: 0`，`BUILD SUCCESS`（2513 基线 + 3 新增 E2E；告警域 9 类测试含新增 `AlarmGenerationEndToEndTest` 3 用例全绿）。日志中的 `ERROR` 行来自故意走异常分支的既有测试（如 TencentSms/Minio/GlobalExceptionHandler），对应用例本身 0F0E，非失败。
- **[核查·通过]** frontend 本轮未碰（范围仅 backend iot/alarm），按 brief 第 2 条注明跳过，无新回归面。

---

## 逐条复评重点核对

1. **C-1 真修复**：通过——按唯一键 `SELECT id` 回填 `a.setId(id)`；新增/命中两条路径都拿到真实 id（独立探针测试绿）；去重语义未变（状态/处理人/确认处理时间不回退，只刷 detail/level/update_time/update_by，探针与 E2E 双证）；`upsert→notify` 顺序下 `notify()` 的 `record.getId()` 必非 null 且指向刚写行（E2E 断言 alarm_id 等于主记录 id）。
2. **E2E 测试有效性（红转绿）**：通过——不手工 setId、走真生成入口、断言主记录 1 行 + notice 1 行且 alarm_id 指向主记录；评审在修复前代码独立复现红（notice 0 行 + `NULL not allowed for "ALARM_ID"`），修后绿；非 mock 捷径、非恒真。
3. **I-1 措辞纠正**：通过——两处注释改为“本域 JdbcTemplate 原生 SQL、不经 MP mapper，框架 MetaObjectHandler 仅对 MP mapper 生效”，不再声称“工程无全局 MetaObjectHandler”；实现（审计列写法）零改动。
4. **3 Minor deferred + 附带核查**：通过——如实登记未改；两时间列同值核实通过；2516 用例 0F0E；既有零回归；范围仅 3 文件。

---

## 越界反查

`git rev-parse HEAD` = `788632567b89bf51d82daa7aad746c5606c3efa9`；`git log --oneline 2755060..7886325` 恰 1 提交；`git diff --name-only 2755060..7886325` 恰 3 文件，全部在 `backend/ot-admin/.../iot/alarm/` 下（`AlarmRecord.java` 注释、`AlarmStore.java` upsert+注释、新增 `AlarmGenerationEndToEndTest.java`），`--stat` 147 insertions / 9 deletions。**frontend / contract / ingest / DDL / 既有模块零触碰**。区间干净无夹入提交。主工作树 `git status --short` 仅原有 `?? .agents/`、`?? .tmp-t0044v/`（执行前已存在的 untracked，不属本任务）；评审所用临时 worktree 均已 `git worktree remove --force` 清除，未改主工作树任何代码。

---

## 契约遵守

- fresh / no-subagents / 只评审不修改（唯一例外：本报告）：已遵守。
- 终审约束：C-1 与 I-1 均闭合，无新 C/I → 明确判定 **通过**。
- 未改代码、未推进 TASK 状态、未派子代理；报告写到指定路径 `reports/TASK-0062-review-round2.md`。
