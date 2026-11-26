# 实施 brief：TASK-0061（运维监控页：四类指标 + 巡检清单 + 历史回看）

> 标准任务（跨 backend 巡检域 + frontend 监控页，预估 4 人天）。BASE=640c9bd。一个执行者做完整、统一测一次。前置 0047✅、0063✅（calc 只读端点已就绪）。

## 批次

- TASK-0061（本批 1 个）
- 验收粒度：一个 subagent 跑一次整体验收（backend mvn test + frontend 构建/检查）
- BASE：`640c9bd`（派发前代码仓 `git rev-parse --short HEAD`）
- 改动范围（文件/目录级，并行准入与越界反查用）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/inspect/**`（新建巡检域：entity/mapper/service/controller/scheduler + VO/DTO）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/inspect/**`
  - `resources/sql/upgrade/20261124_iot_inspect.sql`（巡检结果表 DDL + sys_module 菜单种子，**只产出不执行**）
  - `frontend/src/views/monitor/iot-inspect/**`（新建运维监控页 + helper）
  - `frontend/src/api/monitor/**`（如无合适 api 目录则新建 iot-inspect-api.ts）
  - 允许对 `resources/sql/01_base.sql` 仅做"菜单种子同步注释"（可选，若 upgrade 种子已足够则不碰）
  - **不得触碰**：ingest/**（0084 修订在途）、resources/contract/**（冻结）、iot/publish|calcread|statusread|metric 既有代码（只读调用其端点）、framework upstreams

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0061.md`
- 关联对象：`features/FEAT-0020.md`、`requirements/REQ-0031.md`、`testcases/TC-0020.md`、`decisions/ADR-0014.md`（TD-0047 巡检落库 90 天）
- 数据源端点（只读复用，不改）：
  - `backend/ot-admin/.../iot/calcread/controller/IotCalcReadController.java` → `GET /api/v1/iot/calc/state|log`（0063，CalcStateVO 含 calcStatus/errorCount/lastCalcTime/watermark；CalcLogVO 含 resultStatus/errorMsg/fullKey/periodStart/periodEnd）
  - `backend/ot-admin/.../iot/statusread/controller/IotStatusReadController.java` → `GET /api/v1/iot/status/current|history`（0050）
  - `backend/ot-admin/.../iot/metric/controller/IotMetricController.java` → `GET /iot/metric/list`（0045）
  - `frontend/src/api/display/display-api.ts`（读端点封装范式；安全红线注释必读）
- 调度范式：`backend/ot-admin/.../iot/publish/config/PublishSchedulingConfig.java`（@EnableScheduling）+ `.../publish/notify/PublishAckTimeoutScanner.java`（@Scheduled fixedDelayString + try/catch 不抛 + 幂等条件更新）
- 数据权限范式：`backend/ot-admin/.../iot/calcread/impl/CalcReadServiceImpl.java`（visible() 裁剪 + allTenant）+ `.../iot/dataperm/**`
- 前端页面范式：`frontend/src/views/monitor/actuator/health/index.vue`（a-card/v-privilege/空态 a-empty）、`frontend/src/views/monitor/cache/index.vue`
- 前端路由：`frontend/src/router/index.ts:163`（import.meta.glob 动态注册）+ `resources/sql/01_base.sql:643-689`（sys_module 菜单种子，110 系统监控 / 168-170 缓存监控示例）

## 接口与决策（逐条结论，无选项）

### D1 · 四类指标（监控页看板，全程无日志检索）
1. **通道健康**：`GET /iot/channel/list` 既有端点取通道状态（若既有端点不含状态则用 channel 实体的 status 字段）；展示各通道 status；无通道显示空态。
2. **点位在线率（聚合快照，禁止逐点扫 TDengine）**：以**巡检任务产出的聚合快照**为数据源（见 D2），读巡检结果表里的最近一次 `online_rate`（分子/分母/百分比）；新租户/无点位时显示 **N/A 而非 100%**（REQ-0031 验收 6，不误导）。禁止在监控页或巡检任务里逐点查 TDengine。
3. **计算成功率**：基于 0063 `GET /api/v1/iot/calc/state` 的 calcStatus（0 正常/2 滞后/3 异常停算）+ `GET /log` 的 resultStatus 计算成功率（成功/(成功+失败)）；无数据时显示空态/N/A，不造 100%。
4. **指标产出情况**：基于 `GET /iot/metric/list` + 指标完整度（若 0045/0046 metric VO 含 completeness 字段则用之；无则展示"指标清单 + 应产出/实际产出"从 calc state 派生的粗粒度计数，并在页面注明口径）；无数据显示空态。

### D2 · 巡检任务（backend，Spring @Scheduled，重入保护）
- 复用 `PublishSchedulingConfig` 的 `@EnableScheduling`（若已启用则不再新建 config，直接在 inspect 包写 `@Component` + `@Scheduled`）；周期 `ot.inspect.interval-seconds`（默认 3600，fixedDelayString + initialDelay SpEL 同 0025 范式）。
- **重入保护**：用 Redis `SETNX iot:inspect:lock:{tenant|global} value NX EX <ttl>`（ttl = 周期秒数×2 兜底），拿到锁才执行、finally 释放（比对 value 防误删）；Redis 不可用时 fail-soft 跳过本轮并 WARN（不阻断），与 ingest fail-soft 范式一致。
- 巡检产出**三类清单 + 一类比对**（REQ-0031）：
  1. 当前点位离线清单（含离线时长）——数据源：statusread current + 点位配置（point status），**聚合计数**，不逐点扫 TDengine；
  2. CALC_ERR 统计按数据项/原因——数据源：0063 `GET /calc/log?fullKey=&from=&to=` 的 errorMsg/resultStatus 聚合；
  3. 完整度低于阈值指标清单——数据源：metric 完整度（若无字段则用 calc state 滞后/停算近似，页面注明口径）；
  4. 标签租户与库名 `iot_<tenant_code>` 不一致比对清单（ADR-0002 巡检项）——数据源：租户表 + TDengine/库名契约（**TDengine REST 当前故障，此项允许 fail-soft 产出"待核验"占位项而非报错**，报告登记）。
- 巡检任务只写巡检结果表（D3），不改写任何业务数据。

### D3 · 巡检结果落库（90 天归档，ADR-0014）
- 新建 `iot_inspect_result` 表（DDL 写 `resources/sql/upgrade/20261124_iot_inspect.sql`）：字段至少含 id/tenant_id/inspect_time/inspect_type（OFFLINE_LIST / CALC_ERR / COMPLETENESS / TENANT_DB_MISMATCH）/payload_json（清单 JSON）/item_count/created_time；索引 (tenant_id, inspect_time, inspect_type)。**不套审计四件套**（巡检是系统级定时任务，无操作员上下文，仿 publish_log 无 create_by）。
- 归档：巡检写入前删除 `inspect_time < now - 90 day` 的旧行（按期归档，可配 `ot.inspect.retention-days` 默认 90），条件删除幂等。
- **DDL 只产出不执行**：不得在用户 phpStudy MySQL 执行；报告写"DDL 应用条件"（用户按 `sql-exec.sh` 每条语句单独调用执行，语句分行兼容 `;\n` 切分——参考 TASK-0063 修订的教训：PREPARE/EXECUTE/DEALLOCATE 等不得同行多语句）。

### D4 · 管理面只读端点（frontend 数据源）
- `GET /api/v1/iot/inspect/dashboard` → 四类指标聚合（tenant 从会话，按数据权限裁剪；无数据显示 N/A/空态字段，不造数）。
- `GET /api/v1/iot/inspect/history?type=&from=&to=` → 历史巡检清单（90 天内，分页+按天聚合；跨租户明细 → 403，数据权限 fail-closed）。
- `GET /api/v1/iot/inspect/latest?type=` → 最近一次巡检清单（监控页当前视图）。
- 权限码沿用 `iot:instance:query`（0064 无 inspect:* 位，仿 0063 calcread 沿用口径）；**读不审**（0079 范式，不挂 @OperateLog）；零写口（无 POST/PUT 暴露给前端——巡检是调度任务内部写，不开放手动触发 API）。
- 数据权限：tenant_id 从会话强制注入，禁止请求参数指定 tenant 越权；跨租户 fullKey/实例明细按 DataPermService.visible() 裁剪，无权返回空集/403。

### D5 · 前端监控页（frontend/src/views/monitor/iot-inspect/）
- 新建运维监控页（Vue 3，仿 actuator/cache 页的 a-card/v-privilege 风格）：四个指标卡（通道健康/点位在线率/计算成功率/指标产出）+ 三类巡检清单 tab + 标签租户比对清单 + 历史回看（日期范围选择 + 列表）。
- 空态纪律：无数据显示 `a-empty` / N/A，**绝不造假数据**；在线率无点位显示 N/A 不是 100%（REQ-0031 验收 6）。
- API 封装在 `frontend/src/api/monitor/iot-inspect-api.ts`，只调 D4 管理面端点（走已登录态 axios），**不持有 appKey/secret、不直连 `/openapi/v1/**`、不自造代理**（安全红线，仿 display-api.ts 头注）。
- 菜单种子：在 `resources/sql/upgrade/20261124_iot_inspect.sql` 加 sys_module INSERT（挂在 110「系统监控」下，如 `(171, 110, 1, 'IoT运维监控', 2, '/monitor/iot-inspect', 'monitor/iot-inspect/index', NULL, NULL, 'DashboardOutlined', 8, 1, 1, 0, 0, ...)`，具体编号避开已用 168-170）+ 授权段（若权限码沿用 iot:instance:query 则无需新权限种子，menu 权限字段给 iot:instance:query）。若 01_base.sql 的菜单编号段需要同步注释，可选追加注释，不改数据行。

## 验收命令（必须原样贴回输出）

1. `cd backend && mvn -pl ot-admin test -q`（或工程既有全量命令，先读 README/pom 确认）→ 0 失败 0 错误（贴 Results 段）；巡检调度/落库/归档/数据权限/403 红线有契约测试（TDD 红转绿留痕）
2. `cd frontend && pnpm build:test && pnpm lint && pnpm typecheck`（读 package.json 确认脚本名）→ 全绿；监控页构建通过、无 TS 错误
3. DDL 静态检查：`20261124_iot_inspect.sql` 每条语句独占行（兼容 sql-exec.sh 的 `;\n` 切分）、无同行多语句；**未在用户库执行**（报告声明）
4. 范围核对：`git diff --name-only 640c9bd..HEAD` 全落 D1 改动范围声明；ingest/contract/既有模块零触碰

## 报告

报告写到：`reports\TASK-0061-report.md`。五项齐全 + 一句话结论 + TDD 红转绿（每修复附"为什么现在对了"）+ DDL 未执行声明 + 空态/租户 403 证据。

## 契约

- no-subagents；只改本批改动范围；不置 TASK 已完成；不改 workitems 索引；DDL 只在文件产出不执行。
- 不造假数据、不硬编码凭据/文档库路径、不碰 appKey/secret、不直连 openapi HMAC。
- 空态纪律：在线率 N/A 非 100%；无数据 a-empty。
- 跨租户 403（REQ-0031 验收 5）。
- 测试或验收失败先定位根因，禁止为匹配输出改测试/改期望。
