# 实施 brief：TASK-0062（系统事件告警记录闭环：生成/列表/确认/站内出口）

> 标准任务（backend iot/alarm 新域 + frontend 告警列表页）。BASE=24b5c44。一个执行者做完整、统一测一次。前置 0061✅（巡检域已交付，文件集串行完毕）、0063✅。FEAT-0020 最后一块，收尾后推 FEAT-0020 完成。

## 批次

- TASK-0062（本批 1 个）
- 验收粒度：一个 subagent 跑一次整体验收（backend mvn test + frontend 构建/检查）
- BASE：`24b5c44`
- 改动范围（文件/目录级，越界反查用）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/**`（新建 alarm 域：entity/mapper/service/controller/listener/vo + DDL 对应）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/alarm/**`
  - `resources/sql/upgrade/20261124_iot_alarm.sql`（告警记录表 DDL + 菜单种子 id=172，**只产出不执行**）
  - `frontend/src/views/monitor/alarm/**`（新建告警列表页 + helper）
  - `frontend/src/api/monitor/alarm-api.ts`
  - `frontend/tests/unit/monitor/alarm-*.spec.ts`（如需）
  - **不得触碰**：ingest/**、resources/contract/**（冻结）、iot/inspect（0061 刚交付）、iot/gateway 既有事件代码（只读消费其事件，不改）、iot/calcread|statusread|metric|publish 既有模块、sys_notice（公告表，不用作告警出口）、framework upstreams

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0062.md`
- 关联对象：`features/FEAT-0020.md`、`requirements/REQ-0031.md`、`testcases/TC-0020.md`、`decisions/ADR-0013.md`（增补 6：系统事件告警最小闭环边界）
- 事件源（只读消费，不改）：
  - `backend/ot-admin/.../iot/gateway/event/GatewayOfflineEvent.java` + `GatewayOfflineEventListener.java` + `GatewayOfflineReason.java`（TIMEOUT/LWT_ABNORMAL/GRACEFUL）+ `IotGatewayInternalController.java`（POST /internal/v1/gateway/offline-event 注入端点，dev 冒烟用）
  - 数据超时未达 / 网关缓存积压：一期无现成事件源，**用调度扫描近似**（见 D2），不新建 ingest 改动
- 只读端点范式：`iot/inspect/controller/IotInspectController.java`（0061 刚交付，沿用 iot:instance:query + 读不审 + tenant 会话注入）、`iot/calcread/controller/IotCalcReadController.java`（0063）
- 调度范式：`iot/inspect/scheduler/InspectScheduler.java` + `iot/publish/notify/PublishAckTimeoutScanner.java`
- 前端范式：`frontend/src/views/monitor/iot-inspect/index.vue`（0061 监控页风格）、`frontend/src/api/monitor/iot-inspect-api.ts`

## 接口与决策（逐条结论，无选项）

### D1 · 告警记录表（协调者定版，补上游修订⑩空缺）
- 新建 `iot_alarm_record`（放业务库非 iot_calc；DDL 写 `resources/sql/upgrade/20261124_iot_alarm.sql`）：
  - 字段至少：`id` bigint auto、`tenant_id` bigint、`alarm_type` varchar(32)（GATEWAY_OFFLINE / DATA_TIMEOUT / GW_CACHE_BACKLOG）、`object_type` varchar(32)（GATEWAY/INSTANCE）、`object_code` varchar(96)（对象编码，如 gateway_code）、`alarm_level` varchar(16)（WARN/ERROR，一期用 WARN=离线、ERROR=停算类）、`title` varchar(256)、`detail` varchar(1000)（原因/上下文 JSON 或文本）、`status` tinyint（0 待确认 1 已确认 2 已处理）、`handler_id` bigint null、`handle_note` varchar(500) null、`occurred_at` datetime（事件发生时间）、`confirmed_at` datetime null、`handled_at` datetime null、`created_time` datetime。
  - 索引：`idx_tenant_status_time (tenant_id, status, occurred_at)`、`idx_tenant_type (tenant_id, alarm_type)`。
  - **审计四件套**：沿用 BaseEntity 口径（create_time/create_by/update_time/update_by/deleted）——告警确认/处理需留痕操作人（REQ-0031 验收 5「确认处理留痕」），与巡检结果表（系统级无操作员，不套）不同；用 MyBatis-Plus BaseEntity 或显式列均可，报告说明选择。
  - 幂等：同 (tenant_id, alarm_type, object_code, occurred_at) 唯一键防重复告警（uk）；重复事件只更新不改状态流转（不把已确认的告警打回待确认）。
- **DDL 只产出不执行**：不得在用户 phpStudy MySQL 执行；报告写应用条件（sql-exec.sh 每条语句单独调用，逐行切分兼容 `;\n`）。

### D2 · 告警生成（仅系统事件，ADR-0013 增补 6 边界）
- **网关离线**（GATEWAY_OFFLINE）：新增 `@EventListener` 消费 `GatewayOfflineEvent`，生成告警记录（title「网关离线」、detail 含 reason + gatewayId + occurredAt、alarm_level=WARN）。仿 `GatewayOfflineEventListener` 的 fail-closed（缺字段 WARN 忽略）+ try/catch 不外溢异常。注意：GRACEFUL 正常下线**不生成告警**（只有 TIMEOUT/LWT_ABNORMAL 是异常离线）。
- **数据超时未达**（DATA_TIMEOUT）/ **网关缓存积压**（GW_CACHE_BACKLOG）：一期无现成事件源，**用 Spring @Scheduled 扫描近似**（复用 @EnableScheduling，周期可配如 5 分钟）：
  - DATA_TIMEOUT：扫描 0063 `iot_calc.iot_calc_state` 中 calc_status=2（滞后）/3（停算）且 last_calc_time 逾阈值的项，按数据项生成告警（或聚合为一条，报告说明口径）；
  - GW_CACHE_BACKLOG：若 Redis/点位无现成积压指标，产出**「待核验」占位告警或不生成**（不许造假数据），报告登记——这类一期允许无真数据源的诚实空态。
  - 两者都需**去重**（靠 D1 唯一键），不重复刷告警。
- **边界严格**：不做业务阈值告警事件化/通知矩阵/升级策略（三期）；阈值一期只驱动展示标红。报告声明边界遵守。

### D3 · 管理面端点（列表/确认/处理 + 站内出口）
- `GET /api/v1/iot/alarm/list?status=&type=&from=&to=&page=` → 分页列表（tenant 从会话强制注入，数据权限裁剪；跨租户明细 403）。
- `POST /api/v1/iot/alarm/{id}/confirm` → 状态 0→1，记 confirmed_at + handler_id（当前登录用户，SaHolder/SysUserContext）。
- `POST /api/v1/iot/alarm/{id}/handle` → 状态 1→2（或 0→2 允许直接处理），记 handled_at + handle_note + handler_id。
- 状态流转守卫：非法跃迁（如 2→0）返回 409/400，不静默改；越权（非本租户/无权）403。
- **写口挂 @OperateLog**（确认/处理是写操作，0079 范式：写要审；与 0061 只读端点读不审相反）。权限码沿用 `iot:instance:query`（0064 无 alarm:* 位，仿 0061/0063 口径），确认/处理可加 `iot:instance:update`（若已注册）或沿用 query + 登录态，报告说明选择。
- **站内通知出口**：确认/处理留痕已覆盖"站内出口"最小语义——另在告警**生成**时向目标角色写一条站内提醒记录（复用 SysNoticeService.insertNotice 或独立轻量通知表均可，报告说明；sys_notice 是全员公告表，若用它需明确 notice_type 区分，不污染公告列表）。一期**不实际外发**邮件/企微（配置预留不外发）。

### D4 · 前端告警列表页（frontend/src/views/monitor/alarm/）
- 新建告警列表页（Vue 3，仿 0061 monitor 风格）：筛选（状态/类型/时间）+ 列表（标题/级别/对象/时间/状态）+ 确认/处理操作按钮 + 处理备注输入。
- 权限 `v-privilege` 沿用 iot:instance:query；状态标签配色；空态 a-empty；跨租户数据由后端 403/裁剪，前端不暴露越权入口。
- API 封装 `frontend/src/api/monitor/alarm-api.ts`，只调 D3 管理面端点（已登录态 axios），**不持 appKey/secret、不直连 openapi HMAC、不自造代理**。
- 菜单种子：在 DDL 加 sys_menu INSERT（id=172，挂 110「系统监控」下，path `/monitor/alarm`，component `monitor/alarm/index`，perms iot:instance:query）+ sys_role_menu 授权（role 1），仿 0061 `20261124_iot_inspect.sql:31-41` 的幂等写法（WHERE NOT EXISTS）。

## 验收命令（必须原样贴回输出）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（告警生成/去重/状态流转/越权 403/审计留痕有契约测试，TDD 红转绿留痕），贴 Results 段
2. `cd frontend && pnpm build:test && pnpm lint && pnpm typecheck`（读 package.json 确认脚本名）→ 全绿
3. DDL 静态检查：`20261124_iot_alarm.sql` 逐行切分、无同行多语句；**未在用户库执行**（报告声明）
4. 范围核对：`git diff --name-only 24b5c44..HEAD` 全落改动范围声明；ingest/contract/既有模块零触碰

## 报告

报告写到：`reports\TASK-0062-report.md`。五项齐全 + 一句话结论 + TDD 红转绿 + DDL 未执行声明 + 边界遵守声明（仅系统事件、不外发）+ 越权 403 与留痕证据。

## 契约

- no-subagents；只改本批改动范围；不置 TASK 已完成；不改 workitems 索引；DDL 只产出不执行。
- 仅系统事件告警（设备/网关离线、数据超时未达、网关缓存积压），业务阈值告警/通知矩阵/升级策略归三期不做。
- 阈值一期只驱动展示标红；不造假数据（无事件源时诚实空态/待核验）。
- 跨租户 403；写操作留痕；不硬编码凭据/文档库路径；前端不碰 appKey/secret/HMAC。
- 测试失败先定位根因，禁止为匹配输出改测试/改期望。
