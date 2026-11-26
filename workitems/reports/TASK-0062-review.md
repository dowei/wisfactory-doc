# TASK-0062 diff 评审（首评）

- 评审角色：独立代码评审（fresh / no-subagents / 只评审不修改，除本报告外未改任何文件、未推进状态、未派子代理）
- 评审对象：`2755060` 单提交隔离 diff 包（`TASK-0062-diff.txt`，26 文件，BASE=`24b5c44`，区间干净单提交）
- 口径：L1-execute §13「diff 评审」；标准任务首评，1 退回 + 1 复评上限
- 结论：**退回执行者一轮**——发现 1 项 Critical、1 项 Important；其余按 Minor 记 deferred。

---

## 一句话总评

告警记录表 DDL、生成边界（GRACEFUL 不生成 / 缓存积压不造假）、列表/确认/处理端点（跨租户 403、状态守门、写口 @OperateLog）、前端告警页、菜单种子 id=172 与范围/测试/越界反查均独立核实通过；但**真实生成链路存在阻断性缺陷：`AlarmStore.upsert()` 用 `jdbc.update(sql, args...)` 不回填自增主键，导致 `record.getId()==null`，而站内提醒出口 `iot_alarm_notice.alarm_id` 为 NOT NULL——每次告警生成时提醒写入必然失败并被 fail-soft 吞掉，站内出口在真机路径上一条都不落库**，REQ-0031 验收 3/TC-0020 步骤 4（停适配器 5 分钟内告警通知到达指定角色）不成立，构成 Critical 退回项；另有执行者关键判断「工程无全局 MetaObjectHandler」与源码事实不符的事实错误（Important）。

---

## 发现清单

### [Critical] C-1 · 真实生成链路站内提醒始终写不进去（`upsert` 不回填 id + notice.alarm_id NOT NULL），验收 3 不成立

- **出处**：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/store/AlarmStore.java` 的 `upsert()`（diff 第 832–845 行）——`jdbc.update("INSERT INTO iot_alarm_record ...", args...)` 无 `GeneratedKeyHolder`，`AlarmRecord.id` 不被回填；`service/impl/AlarmRecordServiceImpl.java` 的 `createAndNotify()`（第 674–683 行）在 `store.upsert(a)` 后直接 `notifier.notify(tenantId, a)`；`service/impl/AlarmNoticeOutboxNotifier.java` 的 `notify()`（第 566–575 行）把 `record.getId()` 写入 `iot_alarm_notice.alarm_id`；DDL `resources/sql/upgrade/20261124_iot_alarm.sql:2413` 该列 `bigint NOT NULL`（H2 测试建表亦为 `alarm_id BIGINT NOT NULL`，`AlarmNoticeOutboxNotifierTest.java:1418`）。
- **说明**：`AlarmRecordServiceImpl.base()` 不设 id，`upsert()` 也不回填 id，故生产路径交给 notifier 的 record 永远 `id=null`。notifier 的 INSERT 因 `alarm_id` 为 NULL 违反 NOT NULL 约束而抛异常，被自身 `catch (RuntimeException)` 吞掉只打 WARN；上层 `createAndNotify()` 的 catch 也不会感知。结果：**告警主记录正常落库，但站内提醒出口（iot_alarm_notice）在真实告警生成路径上 0 行**。报告的 D3「生成告警即写一条站内提醒」「通知到达角色」只在 `AlarmNoticeOutboxNotifierTest` 里人为 `a.setId(11L)` 的隔离单测中成立，属测试桩掩盖了真实链路的断裂。我用编译后的真实 `AlarmStore` + `AlarmNoticeOutboxNotifier`（H2 MySQL 模式、照测试同款 DDL）端到端复现：`upsert returned=1, record.getId() after upsert=null`；notifier WARN「NULL not allowed for column "ALARM_ID"」；`alarms in iot_alarm_record=1, notices actually written=0`。REQ-0031 验收 3「停适配器 → 站内告警/通知 5 分钟内到达指定角色」与 TC-0020 步骤 4 直接不成立。
- **判定**：阻断主链路交付目标（站内通知出口是 D3 三件交付之一、验收标准硬项），按 §13 Critical → 退回一轮。
- **建议修复方向**：让 `upsert()` 回填主键（`GeneratedKeyHolder` 或插入后 `latest(tenant,type,code,occurredAt)` 取 id 再 `a.setId(...)`；注意 ON DUPLICATE KEY 命中分支也要拿到既有行 id），确保 `createAndNotify()` 传给 notifier 的 record 带真实 id；补一条「service 生成 → notifier → iot_alarm_notice 有行且 alarm_id 指向告警」的端到端契约测试（不手工 setId），并保留 fail-soft 但让“提醒实际未落库”可观测（计数/告警日志）。

### [Important] I-1 · 执行者关键判断「工程无全局 MetaObjectHandler，MP fill 不落值」与源码事实不符

- **出处**：`AlarmRecord.java` 头注（diff 第 133–136 行）、`AlarmStore.java` 头注（第 779–781 行）、`TASK-0062-report.md` 第 15–18 行的「已核 backend/ot-core/ot-admin 全量，工程内无全局 MetaObjectHandler」；实际源码 `backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java:30` 的 `public class MybatisPlusConfig implements MetaObjectHandler`，第 57–60 行 `globalConfig.setMetaObjectHandler(this)` 注入，`OtAdminApplication.java:16` 的 `@ComponentScan(basePackages={"net.otlab.admin","net.otlab.core"})` 确认该类被加载；`BaseEntity.java:26–35` 确为 `@TableField(fill=INSERT/INSERT_UPDATE)`。
- **说明**：工程**确有**全局 MetaObjectHandler（ot-core 的 `MybatisPlusConfig`）且已被 ot-admin 扫描装配，报告「无全局 MetaObjectHandler」的断言为事实错误。评审独立核实后，这个错误**没有导致实现本身出错**——本域走 JdbcTemplate 原生 SQL（`AlarmStore`），`AlarmRecord` 是普通 POJO 非 `@TableName`/`BaseEntity` 实体，MP 自动填充对它和原生 SQL 均不生效，因此**显式写 create_time/update_time/create_by/update_by 的列是正确的、必要的**，确认/处理留痕列也确实落库（H2 真 SQL 契约测试 `AlarmStoreTest.confirmAndHandleAudit/auditAndTenantIsolation` 已核实）。但执行者给出的核心理由不成立，报告口径误导；若后续有人据「无 MetaObjectHandler」这一错误前提删掉显式写列，会直接破坏留痕。C-1 修复时一并改正报告与源码注释即可（保留显式写列的实现）。
- **判定**：关键判断失真、影响结论可信度，按 §13 Important 记录随退回轮一并纠正；非独立功能缺陷。
- **建议修复方向**：把报告/注释改为准确表述（「工程有全局 MetaObjectHandler，但本域为 JdbcTemplate 原生 SQL + 普通 POJO，MP fill 不适用，故显式写审计列」），无需改动显式列实现本身。

### [Minor] M-1 · `upsert` 的 ON DUPLICATE 分支刷新 `alarm_level`，与「重复事件只改 detail/时间」的登记措辞略有出入

- **出处**：`AlarmStore.upsert()` 第 840–841 行 `ON DUPLICATE KEY UPDATE detail=VALUES(detail), alarm_level=VALUES(alarm_level), update_time=..., update_by=...`；报告 D1 第 19–20 行称「只刷新 detail/alarm_level/update_time/update_by」（报告写全了，brief D1 第 37 行只说「只更新不改状态流转」）。
- **说明**：刷新级别不违反状态不回退硬约束，且报告已如实登记，不构成缺陷；仅提示语义上重复事件若带不同 level 会覆盖原级别，确认符合预期即可。deferred。

### [Minor] M-2 · `check` 权限码沿用 `iot:instance:query` 用于写口，权限模型粒度不足

- **出处**：`IotAlarmController.java` 三个端点均 `@SaCheckPermission("iot:instance:query")`（第 91/105/113 行）；报告 D3 第 43–44 行与 0061 评审 M-4 同口径。
- **说明**：brief D3 明确允许（0064 无 `alarm:*` 位，避免造无人持有的权限位导致全角色 403），实现与 brief 一致；但确认/处理是写操作却与只读复用同码，属权限模型层缺口，非本任务错误。deferred，随后续权限设计处理。

### [Minor] M-3 · `handle` 时间戳用服务端 `System.currentTimeMillis()`，与 `occurred_at` 时区口径无冲突但需确认客户端展示一致

- **出处**：`AlarmRecordServiceImpl.confirm/handle` 第 707/728 行 `new Timestamp(System.currentTimeMillis())`；前端 `formatAlarmTime(ms)` 用本地时区 `new Date(ms)` 格式化。
- **说明**：毫秒时间戳两端一致，无实际缺陷；仅提示部署环境 DB/应用时区需统一以免 datetime 列与前端展示偏差。deferred。

---

## 逐条评审重点核对（均独立核实源码/契约原文，不依赖报告自述）

1. **告警记录表（D1）**：
   - 字段/索引/唯一键：`20261124_iot_alarm.sql` 建表含 brief D1 全部字段（含 `created_time` 与审计 `create_time` 并存）、`idx_tenant_status_time(tenant_id,status,occurred_at)`、`idx_tenant_type(tenant_id,alarm_type)`、`uk_alarm_dedup(tenant_id,alarm_type,object_code,occurred_at)`，DDL 契约测试逐项锁定。**通过**。
   - 审计四件套：`AlarmStore` 显式写 `create_time/create_by/update_time/update_by` + 确认/处理写 `handler_id/confirmed_at/handled_at/handle_note`，H2 真 SQL 测试核实落库；「工程无 MetaObjectHandler」前提失实（见 I-1），但显式列实现正确。**实现通过，理由纠正见 I-1**。
   - **重复事件不回退已确认状态：成立**——`upsert` ON DUPLICATE 只刷 detail/level/update_time/update_by，不碰 status/handler_id/confirm/handle 时间；`AlarmStoreTest.dedupKeepsStatus` 真 SQL 断言已确认(1)告警重复事件后仍为 1。**通过**。
   - **DDL 逐行切分 + 只产出未执行**：以 `sql-exec.sh` 同口径 `;\s*\n` 切分实测 4 段、0 同行多语句；DDL 仅由 `IotAlarmDdlSplitContractTest` 静态读取，全仓无执行脚本引用。**通过**。

2. **告警生成边界（D2，ADR-0013 增补 6）**：
   - 仅三类系统事件：`AlarmType` 枚举严格三值，`fromName` 未知名返回 null fail-closed；前端 `ALARM_TYPES` 同三值，无 `THRESHOLD`。**通过**。
   - **GRACEFUL 正常下线不生成：成立**——`GatewayOfflineAlarmListener` 在 reason 判断与 `AlarmRecordServiceImpl.raiseGatewayOffline` 服务层各守一道（listener 第 333–335 行 + service 第 634–636 行）；`GatewayOfflineAlarmListenerTest.gracefulSkipped` 与 `AlarmRecordServiceContractTest.gracefulDoesNotRaise` 双层断言。事件源 `GatewayOfflineReason`（TIMEOUT/LWT_ABNORMAL/GRACEFUL）契约原文核实一致。**通过**。
   - fail-closed：`GatewayOfflineEvent` 四字段任一缺失 WARN 忽略（listener 第 328–332 行）+ 消费 try/catch 不外溢（第 336–342 行），仿 `GatewayOfflineEventListener` 范式，测试覆盖 4 类缺字段与异常兜底。**通过**。
   - **DATA_TIMEOUT**：`AlarmTimeoutScanner.scanDataTimeout()` 用 `@Qualifier("calcReadJdbcTemplate")`（该 bean 在 0063 `CalcReadDataSourceConfig:47-48` 核实存在）查 0063 `iot_calc.iot_calc_state` 的 `calc_status IN(2,3) AND last_calc_time < cutoff`，按 full_key 逐项生成 ERROR 告警，单行失败/源不可用 fail-soft 降级；`@Scheduled(fixedDelayString/initialDelayString)` 复用 `PublishSchedulingConfig` 的 `@EnableScheduling`（唯一开关，未重复建）。**通过**。
   - **缓存积压无真源不造假：成立**——`scanCacheBacklog()` 仅 log + `return 0`，不调 `raiseCacheBacklog`，测试 `cacheBacklogNoFakeData` 断言 service 零触达；报告登记「待核验」占位。**通过**。
   - 业务阈值/通知矩阵/升级策略未混入，阈值一期只驱动展示标红（前端 `alarmLevelColor` 仅 ERROR/WARN，无阈值规则）。**边界遵守通过**。

3. **端点与权限（D3，REQ-0031 验收 4/5）**：
   - 列表 tenant 会话注入：`AlarmRecordServiceImpl.requireTenant()` 取 `TenantContext.getTenantId()`（唯一入口，签名核实），`AlarmStore.list/count` 均显式带 tenant_id，请求无 tenant 参数；`AlarmRecordServiceContractTest.listUsesSessionTenant` 断言 eq(TENANT)。**通过**。
   - **跨租户 403：成立**——`confirm/handle` 先 `store.byId(会话 tenant, id)`，查无返回 403（`ResultCode.FORBIDDEN=403` 核实），`AlarmStore.confirm/handle` 的 WHERE 亦带 tenant_id 条件（0 行并发 409）；`crossTenantForbidden` 断言 403 且写口零触达。**通过**。
   - 状态流转守卫：`AlarmStatus.canConfirm/canHandle` + 服务层非法跃迁 409、空 note 400、并发 0 行 409，不静默改；测试覆盖 2→确认/处理 409。**通过**。
   - **写口 @OperateLog 留痕：成立**——confirm/handle 方法均挂 `@OperateLog(title=..., businessType=UPDATE)`，list 不挂；`IotAlarmControllerTest.annotationsLocked` 反射逐方法锁定，全局 `OperateLogCoverageGuardTest.allManagementWriteEndpointsAudited` 复跑通过（新控制器写口全挂审计，无回归）。服务端另写 handler_id/update_by/时间戳双留痕。**通过**。
   - **独立 iot_alarm_notice 出口不污染 sys_notice：落实（但真链路写不进，见 C-1）**——全包 grep 无任何 `sys_notice`/`SysNotice` 写引用，仅注释声明不用；`AlarmNoticeOutboxNotifier` INSERT 目标为独立表，DDL 独立建表；无邮件/企微外发代码（grep 无命中），配置 `ot.alarm.notice-target-role` 预留。**隔离选择通过；有效性因 C-1 不通过**。
   - 权限码沿用 `iot:instance:query`（见 M-2）。

4. **前端告警页（D4）**：
   - 筛选（状态/类型/时间窗）+ 列表（标题/级别/对象/时间/状态）+ 确认/处理备注弹窗齐备；状态/级别/类型标签配色；空态 `a-empty`?——列表用 `a-table` 的 `:locale="{ emptyText: '暂无系统事件告警（无事件源时诚实空态，不造假数据）' }"`（ant-design-vue 表格空态，等价诚实空态文案，brief D4 要求「空态 a-empty」，实现用 table locale 空文案，语义一致但非字面 `a-empty` 组件，记 Minor 级登记不作缺陷）；操作按钮按状态禁用与后端 409 同口径。
   - **API 只调管理面端点、不碰 appKey/HMAC/代理**：`alarm-api.ts` 仅 `getRequest('/api/v1/iot/alarm/list')` / `postRequest(confirm)` / `postParamRequest(handle)`，全量 grep 无 appKey/secret/openapi/HMAC/代理。**通过**。
   - **菜单种子 id=172 幂等、挂 110、避开 0061 的 171**：DDL `INSERT ... SELECT 172,110,1,... 'monitor/alarm/index', 'iot:instance:query' ... WHERE NOT EXISTS` + sys_role_menu role=1 授权同幂等；`01_base.sql` 核实 110「系统监控」存在、168-170 已用、0061 用 171、172 在基线无占用；sys_menu 列清单逐列与 01_base 对齐。**通过**。

5. **测试有效性与零回归**：
   - 独立复跑 `mvn -pl ot-admin test`：**2513 tests, 0 failures, 0 errors**（告警 8 类共 27 用例逐类 3/4/3/2/7/3/1/4... 全绿，与报告一致）；`OperateLogCoverageGuardTest` 全绿。
   - 前端独立复跑：`pnpm test` 66 文件/610 passed（含 alarm-helper 4 用例）；`pnpm lint` 9 warning/0 error 全在 5 个存量无关文件，4 个 alarm 文件 0 warning；`pnpm typecheck` 0 输出退出 0；`pnpm build:test` 成功（仅既有 chunk 体积 warning）。
   - TDD 红转绿留痕（报告第三节）逐条对应测试断言与实现修复（GRACEFUL 双层守卫、store.count 桩、DDL created_time、confirm/handle mock 桩等），可信。
   - 调试残留：diff 新增行 grep `System.out/err|console.log|debugger|TODO|FIXME|printStackTrace` 为 0。**通过**。
   - 0061/0063/gateway 既有代码零改动（范围反查佐证），零回归。

6. **范围**：`git show --stat 2755060` = 26 文件全新增，`git diff --name-only 24b5c44..HEAD` 恰 26 文件全落 brief 声明范围；`ingest/**`、`resources/contract/**`、`iot/inspect`（0061）、`iot/gateway` 既有事件代码、`iot/calcread|statusread|metric|publish` 既有模块、`sys_notice` 公告表、framework upstreams 零触碰；`.agents/`/`.tmp-t0044v/` 为执行前已有 untracked，不属本任务。**通过**。

---

## 越界反查

`git diff --name-only 24b5c44..HEAD` 恰 26 文件、单提交 `2755060`、区间干净无夹入，全部落在 brief 改动范围声明。`AlarmConfig`/`AlarmStore`/扫描器只读引用既有 `calcReadJdbcTemplate` 与 `GatewayOfflineEvent`（未改其一行），站内出口走新表不写 `sys_notice`。**结论：无越界。**

---

## 契约遵守

- fresh / no-subagents / 只评审不修改（除本报告外未改任何文件、未推进状态、未派子代理）：已遵守。
- 对 brief 特别关注点的明确判定：
  - **仅系统事件边界 + GRACEFUL 不生成 + 缓存积压不造假数据：通过**（三层核实：枚举严格三值、listener+service 双层 GRACEFUL 守卫及双测试、scanCacheBacklog 空跑零调用）。
  - **跨租户 403 + 写口审计留痕：通过**（会话 tenant + byId 判存 403 + WHERE 带 tenant；confirm/handle 挂 @OperateLog 且全局覆盖守卫通过）。
  - **重复事件不回退已确认状态：通过**（H2 真 SQL 契约断言）。
  - **站内出口独立表不污染 sys_notice：落实**（零 sys_notice 写引用），但**有效性未通过**（C-1：真链路写不进）。
  - **DDL 只产出未执行、逐行切分：通过**。
  - **两个关键判断**：(1)「无全局 MetaObjectHandler」**属实性不成立**（见 I-1，源码证实有全局 handler），但显式 JdbcTemplate 审计列实现正确；(2)「独立 iot_alarm_notice 不污染 sys_notice」**落实**（隔离成立），但因 C-1 真链路无有效提醒。

---

## 处置建议

按 §13：C-1 为 Critical、I-1 为 Important → 退回执行者一轮修复：优先修 `upsert` 主键回填 + 端到端站内提醒契约测试（C-1），并改正报告/注释中关于 MetaObjectHandler 的事实错误（I-1，保留显式列实现）。复评 scoped 只看 C-1/I-1 修复 diff、新增端到端测试与既有 25 项无回归。M-1～M-3 记 TASK `任务结果` 的 deferred，随收尾进 LOG，不进修订轮。本轮评 1 Critical / 1 Important / 3 Minor。
