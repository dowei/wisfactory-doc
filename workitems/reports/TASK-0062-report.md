# TASK-0062 实施报告：系统事件告警记录闭环（生成/列表/确认/站内出口）

- 任务：TASK-0062（FEAT-0020 最后一块）
- 执行：单 subagent，BASE=24b5c44，HEAD（执行前）=24b5c44
- 提交：`27550604c09f424ab3a327b77434de6e93ecd241`（26 文件，全部新增，范围核对 ALL IN SCOPE，未 push）
- 结论（一句话）：**告警记录表 DDL + 三类系统事件告警生成（网关离线事件消费 + 数据超时 @Scheduled 近似扫描）+ 列表/确认/处理端点（跨租户 403、写口审计）+ 独立站内提醒出口 + 前端告警列表页全部落地，backend 2513 测试 0 失败 0 错误、frontend 构建/lint/typecheck 全绿，DDL 只产出未执行，范围零越界。**

## 一、交付内容与决策落地

### D1 · 告警记录表（协调者定版，补上游修订⑩空缺）
- 新建 `resources/sql/upgrade/20261124_iot_alarm.sql`：`iot_alarm_record` 字段按 brief D1 全部齐全——
  `id/tenant_id/alarm_type/object_type/object_code/alarm_level/title/detail/status/handler_id/handle_note/occurred_at/confirmed_at/handled_at/created_time`；
  索引 `idx_tenant_status_time (tenant_id,status,occurred_at)`、`idx_tenant_type (tenant_id,alarm_type)`；
  唯一键 `uk_alarm_dedup (tenant_id, alarm_type, object_code, occurred_at)`。
- **审计四件套选择：显式列**（`create_time/create_by/update_time/update_by/deleted`），不套 MP `@TableField(fill=)`。
  理由：工程内无全局 `MetaObjectHandler`（已核 `backend/ot-core`/`ot-admin` 全量），MP 自动填充不会真正落值；
  本域走 JdbcTemplate（`AlarmStore`）显式写列，确认/处理真留操作人（REQ-0031 验收 5）。与 0061 巡检结果表
  （系统级无操作员、不套审计）刻意不同，报告登记此差异。
- 去重语义：`INSERT ... ON DUPLICATE KEY UPDATE` 只刷新 `detail/alarm_level/update_time/update_by`，
  **不改 status/handler_id/confirmed_at/handled_at**——重复事件不把已确认告警打回待确认（H2 MySQL 模式真 SQL 契约测试覆盖）。
- DDL 配套新表 `iot_alarm_notice`（站内提醒出口，见 D3）；菜单种子 id=172 挂 110 系统监控，
  组件 `monitor/alarm/index`，权限 `iot:instance:query`，`WHERE NOT EXISTS` 幂等 + role 1 授权（仿 0061 写法）。

### D2 · 告警生成（仅系统事件，ADR-0013 增补 6）
- **GATEWAY_OFFLINE**：新 `GatewayOfflineAlarmListener` 只读消费既有 `GatewayOfflineEvent`（未改网关域一行代码），
  title「网关离线」、detail 含 `reason + gatewayId + occurredAt`、level=WARN；仿 `GatewayOfflineEventListener`
  的 fail-closed（缺字段 WARN 忽略）+ try/catch 不外溢；**GRACEFUL 正常下线不生成告警**（listener + service 双层守卫）。
- **DATA_TIMEOUT**：新 `AlarmTimeoutScanner`（`@Scheduled`，复用 `PublishSchedulingConfig` 的 `@EnableScheduling`，
  周期 `ot.alarm.data-timeout-scan-seconds` 默认 300 秒）扫描 0063 `iot_calc.iot_calc_state` 中
  `calc_status IN (2,3)` 且 `last_calc_time` 逾阈值的项，**按数据项 full_key 逐项生成 ERROR 告警**
  （聚合为一条会丢数据项定位，故逐项；唯一键去重不重复刷）。只读源不可用 fail-soft 降级。
- **GW_CACHE_BACKLOG**：一期无 Redis/点位积压指标 → 扫描入口 `scanCacheBacklog()` 返回 0，
  **不生成假数据**（诚实空态，本页登记「待核验」占位）；真指标接入后走 `raiseCacheBacklog` 显式生成。
- **边界声明**：未做业务阈值告警事件化、通知矩阵、升级策略（三期 PB-14）；阈值一期只驱动展示标红。

### D3 · 管理面端点（列表/确认/处理 + 站内出口）
- `GET /api/v1/iot/alarm/list?status=&type=&from=&to=&pageNum=&pageSize=` 分页列表（tenant 会话强制注入，
  `AlarmStore.list/count` 均显式带 tenant_id，数据权限裁剪）；
  `POST /api/v1/iot/alarm/{id}/confirm` 0→1 写 `confirmed_at + handler_id`；
  `POST /api/v1/iot/alarm/{id}/handle?note=` 0→2 或 1→2 写 `handled_at + handle_note + handler_id`（处理人覆盖最后操作）。
- 状态流转守卫：非法跃迁（2→0、重复确认/处理）返回 **409**，空 note 400，跨租户明细/确认/处理 **403**（`AlarmRecordServiceContractTest`、`AlarmStoreTest` 证据）；
  写口挂 `@OperateLog(title="系统事件告警确认/处理", businessType=UPDATE)`（写要审，与 0061 读不审相反），
  全局 `OperateLogCoverageGuardTest` 复跑通过；权限码沿用 `iot:instance:query`
  （0064 无 `alarm:*` 位，确认/处理不另造无人持有的 `alarm:update` 种子，避免全角色 403，报告登记选择）。
- **站内出口选择：独立轻量表 `iot_alarm_notice`（不复用 sys_notice）**。理由：sys_notice 是全员公告表
  （`notice_type` 1通知/2公告，前端公告列表默认无 type 过滤），新写 type=3 会污染公告列表与字典；
  独立表零污染、按 `tenant_id + target_role + read_status` 可查。生成告警即写一条（目标角色
  `ot.alarm.notice-target-role` 默认 platform_admin，运维/平台管理员），**一期不实际外发邮件/企微**（配置预留不外发）。
  提醒写失败 fail-soft 仅 WARN，不阻断告警主记录。

### D4 · 前端告警列表页
- 新建 `frontend/src/views/monitor/alarm/index.vue`（Vue 3，仿 0061 monitor 风格）：筛选（状态/类型/时间）+
  列表（标题/级别/对象/时间/状态）+ 确认/处理（备注弹窗输入）；`v-privilege="'iot:instance:query'"`；
  状态/级别标签配色、空态 `a-empty` 文案「无事件源时诚实空态，不造假数据」；操作按钮按状态可用性禁用
  （与后端 409 同口径），跨租户由后端 403/裁剪，前端不暴露越权入口。
- `frontend/src/api/monitor/alarm-api.ts` 只调 D3 管理面端点（已登录态 axios：
  `getRequest/postRequest/postParamRequest`），**不持 appKey/secret、不直连 openapi HMAC、不自造代理**。
- 菜单种子见 D1（DDL 内 id=172）。

## 二、验收命令与输出（原样贴回）

### 1. backend
```
$ cd backend && mvn -pl ot-admin test
[INFO] Tests run: 2513, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```
告警域契约测试类（全绿）：
```
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- IotAlarmControllerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- GatewayOfflineAlarmListenerTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- AlarmTimeoutScannerTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 -- AlarmEnumsTest
Tests run: 7, Failures: 0, Errors: 0, Skipped: 0 -- AlarmRecordServiceContractTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- IotAlarmDdlSplitContractTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 -- AlarmNoticeOutboxNotifierTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- AlarmStoreTest
（另：OperateLogCoverageGuardTest 3 tests 0 失败，新写口审计接线未回归）
```

### 2. frontend
```
$ cd frontend && pnpm build:test
✓ built in 1m 8s
（仅有既有第三方 css "2of" 警告，与本改动无关）

$ cd frontend && pnpm lint
✖ 9 problems (0 errors, 9 warnings)   -- 均属既有 unrelated 文件，本改动 4 个文件 0 warning

$ cd frontend && pnpm typecheck
$ vue-tsc --noEmit                    -- 无输出，通过

$ cd frontend && pnpm test（附加）
✓ tests/unit/monitor/alarm-helper.spec.ts (4 tests)
Test Files 66 passed (66), Tests 610 passed (610)
```

### 3. DDL 静态检查
```
$ python -I -c "<按 sql-exec.sh 的 ;\n 切分复刻检查>"
segments: 4
segments with inline semicolon: []
CREATE TABLE IF NOT EXISTS: iot_alarm_record / iot_alarm_notice
INSERT INTO sys_menu / sys_role_menu（均 WHERE NOT EXISTS）
IotAlarmDdlSplitContractTest 3 tests 0 失败（机械锁定同样口径）
```
**DDL 未执行声明**：`20261124_iot_alarm.sql` **只产出，未在用户 phpStudy MySQL（8.0.12@3307）或任何环境执行**。
应用条件：由部署方按环境通过 `backend/scripts/sql-exec.sh` 逐条单独调用（脚本按 `;\n` 切分，
本文件每条语句独占一行、无同行多语句）；先建表，再执行两条幂等菜单种子；
应用后可用报告尾部核对 SQL 验证（`SHOW CREATE TABLE` / `SELECT ... sys_menu WHERE id=172`）。

### 4. 范围核对
```
$ git status --short
?? backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/
?? backend/ot-admin/src/test/java/net/otlab/admin/module/iot/alarm/
?? frontend/src/api/monitor/alarm-api.ts
?? frontend/src/views/monitor/alarm/
?? frontend/tests/unit/monitor/alarm-helper.spec.ts
?? resources/sql/upgrade/20261124_iot_alarm.sql
（git diff --name-only 24b5c44..HEAD 在提交前为空；全部为新增文件，零 tracked 修改）
```
所有新增文件均落 brief 改动范围声明；`ingest/**`、`resources/contract/**`、`iot/inspect/**`（0061）、
`iot/gateway` 既有事件代码、`iot/calcread|statusread|metric|publish` 既有模块、`sys_notice` 公告表、
framework upstreams **零触碰**。（`.agents/`、`.tmp-t0044v/` 为执行前已有的 untracked，不属本任务、不提交。）

## 三、TDD 红转绿留痕

1. **RED**：先写 8 个契约测试类（值域/状态流转、store H2 真 SQL 去重与审计、service 越权 403/409、
   listener fail-closed/GRACEFUL、scanner 近似扫描/空态、DDL 静态、前端 helper）。首次
   `mvn -pl ot-admin test` 编译失败（找不到 AlarmType/AlarmStore/IotAlarmController 等符号）——预期红。
2. **GREEN 过程（每修复附「为什么现在对了」）**：
   - 实现 AlarmType/Status/Record/Store/Service/Controller/Listener/Scanner/Notifier/Config 后重跑，
     6 项失败：①`gracefulDoesNotRaise`——直接调 service 传 GRACEFUL 仍生成 → **service 层补 GRACEFUL 守卫**（纵深防御，为什么现在对了：listener 与服务双层过滤，任何调用路径都不生成）；
     ②`listUsesSessionTenant` total=0 → **测试补 stub store.count**（为什么现在对了：VO.total 来自独立 count 查询，mock 默认返回 0 是测试桩不全，不是实现错）；
     ③`listScopedAndPaged` status/type 过滤断言错 → **修正测试数据 status 并改类型期望为 2**（为什么现在对了：两行同类型本就应返回 2，原期望与数据矛盾）；
     ④`alarmTableShape` 缺 created_time → **DDL 增 created_time 列**（为什么现在对了：brief D1 明列该字段，与审计 create_time 并存）；
     ⑤⑥ confirm/handle 409 → **测试补 mock 写口返回 1 + 修正 verify 的 tenant 实参为会话租户**（为什么现在对了：mock 默认 int=0 触发并发冲突分支是桩不全；verify 用会话 tenant 与实现一致）。
   - 前端 `postRequest(url)` 单参 TS2554 → **改用 `postParamRequest` 传 note、`postRequest(url, undefined)`**（为什么现在对了：axios 封装签名要求 data 实参，postParamRequest 正是 @RequestParam 形态）；
     helper/index.vue 两处 prettier warning → **按提示压行**（为什么现在对了：eslint 告警清零，本改动文件 0 warning）。
3. **GREEN**：最终 backend 2513/0/0、frontend build/lint/typecheck/单测全绿（见第二节）。

## 四、契约遵守

- **仅系统事件**：网关异常离线（TIMEOUT/LWT_ABNORMAL）、数据超时未达、网关缓存积压；GRACEFUL 不生成；
  业务阈值告警/通知矩阵/升级策略归三期不做，阈值一期只驱动展示标红。
- **不造假数据**：缓存积压无真指标 → 诚实空态/待核验（返回 0，报告登记）；在线/告警无数据显示空态文案。
- **DDL 只产出不执行**；菜单种子幂等；前端不碰 appKey/secret/HMAC；不硬编码凭据/文档库路径。
- **跨租户 403 + 写口留痕**：`AlarmRecordServiceContractTest.crossTenantForbidden` 验证 403 且不触达写库；
  `AlarmStoreTest.confirmAndHandleAudit/auditAndTenantIsolation` 验证 handler_id/update_by/时间戳落库；
  `IotAlarmControllerTest.annotationsLocked` 验证写口 @OperateLog、读口不挂；
  `OperateLogCoverageGuardTest` 全局复跑通过。
- no-subagents；只改本批改动范围；未置 TASK 已完成；未改 workitems 索引；未 push。

## 五、改动文件清单（26 个新增文件）

backend main（13）：
`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/` 下
`config/AlarmConfig.java`、`controller/IotAlarmController.java`、`entity/AlarmRecord.java`、
`entity/AlarmStatus.java`、`entity/AlarmType.java`、`listener/GatewayOfflineAlarmListener.java`、
`scheduler/AlarmTimeoutScanner.java`、`service/AlarmNotifier.java`、`service/AlarmRecordService.java`、
`service/impl/AlarmNoticeOutboxNotifier.java`、`service/impl/AlarmRecordServiceImpl.java`、
`store/AlarmStore.java`、`vo/AlarmListVO.java`。

backend test（8）：同路径 `src/test/java/.../iot/alarm/` 下
`controller/IotAlarmControllerTest.java`、`entity/AlarmEnumsTest.java`、
`listener/GatewayOfflineAlarmListenerTest.java`、`scheduler/AlarmTimeoutScannerTest.java`、
`service/impl/AlarmNoticeOutboxNotifierTest.java`、`service/impl/AlarmRecordServiceContractTest.java`、
`store/AlarmStoreTest.java`、`store/IotAlarmDdlSplitContractTest.java`。

frontend（4）：`src/api/monitor/alarm-api.ts`、`src/views/monitor/alarm/index.vue`、
`src/views/monitor/alarm/helper.ts`、`tests/unit/monitor/alarm-helper.spec.ts`。

SQL（1）：`resources/sql/upgrade/20261124_iot_alarm.sql`。

---
**待办交接（不属于本任务代码改动）**：本 TASK 未置「已完成」（按契约由协调者收尾）；
FEAT-0020 全部子任务（0061/0062/0063）已交付，建议协调者推进 FEAT-0020 完成。
报告路径：`reports/TASK-0062-report.md`。

---
---

# 修订轮（diff 评审退回一轮：1 Critical + 1 Important）

- 依据：`reports/TASK-0062-revision-brief.md` + 首评 `reports/TASK-0062-review.md`
- 基准提交：`2755060`；修订提交：见本段末尾（提交信息注明「TASK-0062 修订轮 C-1」）
- 范围：仅 backend `iot/alarm`（`AlarmStore.java`、`AlarmRecord.java` 注释、新增端到端测试），
  frontend / DDL / ingest / contract / 既有模块零触碰；no-subagents；未置 TASK 已完成、未改索引、未 push。

## R-1 · C-1 修复（真链路站内提醒落库）

**根因（首评复现成立）**：旧 `AlarmStore.upsert()` 用 `jdbc.update(INSERT ... ON DUPLICATE KEY UPDATE)` 不回填
自增主键，`AlarmRecord.getId()` 新增后仍为 null；`AlarmRecordServiceImpl.createAndNotify()` 先 upsert 再
`notifier.notify(tenantId, a)`，而 `AlarmNoticeOutboxNotifier` 用 `record.getId()` 写
`iot_alarm_notice.alarm_id`（DDL NOT NULL）→ 真链路每次提醒 INSERT 违反非空约束，被自身 fail-soft 吞掉，
notice 表 0 行。首评以编译后真实类 H2 端到端复现（`upsert returned=1, id=null`、notice 0 行）。

**修复（brief 裁决方案 2：原生 upsert + 唯一键查回 id）**：`upsert()` 写入后按唯一键
`(tenant_id, alarm_type, object_code, occurred_at)` 执行 `SELECT id ... AND deleted=0` 查回真实主键并
`a.setId(id)`——新增与 ON DUPLICATE 命中两条路径都拿到 id，且指向刚写的那一行；**去重语义不变**
（ON DUPLICATE 分支仍只刷 detail/alarm_level/update_time/update_by，不碰 status/handler_id/确认处理时间）。
`AlarmNoticeOutboxNotifier` 与 `AlarmRecordServiceImpl` 的调用链未改（id 非 null 自然生效）；
fail-soft 语义保留，但不再是「必然失败」。

**TDD 红转绿（新测试 `AlarmGenerationEndToEndTest`，不手工 setId，走真实生成入口）**：
- **RED**（修复前实跑输出，原样摘录）：
  ```
  [WARN] [ALARM-NOTICE] 站内提醒写入失败: NULL not allowed for column "ALARM_ID"
  Tests run: 3, Failures: 2, Errors: 0 -- AlarmGenerationEndToEndTest
  gatewayOfflineGeneratesAlarmAndNoticeWithRealId: 期望 notice 1 行 but was 0
  ```
- **GREEN**（修复后）：
  ```
  Tests run: 3, Failures: 0, Errors: 0 -- AlarmGenerationEndToEndTest
  ```
  三个用例：①网关离线生成 → alarm_record 1 行 + alarm_notice 1 行，`alarm_id` 等于主记录 id、
  target_role=platform_admin；②重复事件仍去重、已确认状态不回退（唯一键查回后零回归）；
  ③附带核查 create_time/created_time 同值非 NULL（见 R-3）。

## R-2 · I-1 措辞纠正（不改实现）

- **报告原措辞失实**（本报告首版 D1 第 16 行）：「工程内无全局 MetaObjectHandler（已核 ot-core/ot-admin 全量），
  MP 自动填充不会真正落值」——与源码不符：`backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java:30`
  明确 `implements MetaObjectHandler`、`:59` `globalConfig.setMetaObjectHandler(this)` 注入全局。
- **纠正后口径（权威）**：工程**有**全局 MetaObjectHandler；但本域 `AlarmRecord` 是普通 POJO，
  `AlarmStore` 走 JdbcTemplate 原生 SQL、不经 MP mapper，框架 fill 对本域不适用，故显式写审计四件套列
  是正确且必要的。确认/处理另写 handler_id/confirmed_at/handled_at/handle_note 的实现保持原样未动。
- 源码注释已按此改写：`AlarmRecord.java` 头注、`AlarmStore.java` 头注（仅注释，零实现改动）。
- DDL 文件 `20261124_iot_alarm.sql:13` 同类旧措辞属 `resources/sql/**`，不在本轮「只改 backend iot/alarm」范围，
  留待后续与 DDL 其他维护一并订正（如实登记，不影响实现）。

## R-3 · 附带核查：create_time / created_time 两列同值

- 首版 DDL 两列各带 `DEFAULT CURRENT_TIMESTAMP`，H2 实测两次独立取值相差 24ms（首版 E2E 用例 RED 暴露），
  属口径分裂隐患。本轮 `upsert()` 改为将同一个 `now` 显式写入两列（`create_time=?` 与 `created_time=?`），
  新增/命中路径均如此；三列（含 update_time）同源。
- E2E 用例 `createdTimeMirrorsCreateTime` 断言两列非 NULL 且毫秒值相等，GREEN 通过。
- 生产 DDL 两列默认值仍各自 `CURRENT_TIMESTAMP` 作为兜底（应用后由应用显式写同值覆盖），无需改 DDL。

## R-4 · 3 Minor deferred（如实登记，本轮不改）

- **M-1**：`upsert` ON DUPLICATE 分支刷新 `alarm_level`（原措辞只提 detail/时间）——不违反状态不回退硬约束，
  报告 D1 已如实登记刷新字段集，deferred。
- **M-2**：确认/处理写口沿用 `iot:instance:query`（与只读同码）——brief D3 明确允许（无 alarm:* 位，
  避免造无人持有的权限位致全角色 403），属权限模型层缺口，deferred 随后续权限设计处理。
- **M-3**：confirm/handle 时间戳用服务端 `System.currentTimeMillis()`，前端按本地时区格式化——毫秒两端一致，
  提示部署环境 DB/应用时区统一即可，deferred。

## R-5 · 修订轮验收输出（原样贴回）

### 1. backend
```
$ cd backend && mvn -pl ot-admin test
[INFO] BUILD SUCCESS
（surefire 汇总：Tests run: 2516, Failures: 0, Errors: 0, Skipped: 0 — 由 2513 增至 2516，新增 3 条 E2E）
```
告警域契约测试类（全绿，含新增类）：
```
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- IotAlarmControllerTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 -- AlarmEnumsTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- GatewayOfflineAlarmListenerTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- AlarmTimeoutScannerTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- AlarmGenerationEndToEndTest（新增）
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- IotAlarmDdlSplitContractTest
Tests run: 7, Failures: 0, Errors: 0, Skipped: 0 -- AlarmRecordServiceContractTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- AlarmStoreTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 -- AlarmNoticeOutboxNotifierTest
```

### 2. frontend（跳过，说明）
本轮**未碰任何前端文件**（范围仅 backend iot/alarm），`pnpm build:test/lint/typecheck` 首轮已全绿，
按修订 brief 第 2 条注明跳过；无前端回归面。

### 3. 范围核对
```
$ git diff --name-only 2755060..HEAD
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/entity/AlarmRecord.java  （仅注释）
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/store/AlarmStore.java    （upsert 主键回填 + created_time + 注释）
?? backend/ot-admin/src/test/java/net/otlab/admin/module/iot/alarm/service/impl/AlarmGenerationEndToEndTest.java（新增）
```
frontend / resources/contract / ingest / iot/inspect / iot/gateway 既有代码 / 既有 iot 模块零触碰；
`.agents/`、` .tmp-t0044v/` 为执行前已有 untracked，不属本任务、不提交。

## R-6 · 修订结论

C-1 真修复（主键回填 + 真链路提醒落库端到端红转绿）、I-1 措辞纠正（保留显式审计列实现）、
3 Minor 如实 deferred、create_time/created_time 同值核实通过；全量 backend 2516 测试 0 失败 0 错误，
范围零越界。未置 TASK 已完成；FEAT-0020 收尾仍由协调者推进。
修订提交哈希：`788632567b89bf51d82daa7aad746c5606c3efa9`（提交信息含「TASK-0062 修订轮 C-1」，3 文件，范围零越界，未 push）。

