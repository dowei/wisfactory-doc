# 实施 brief：TASK-0087（管理面数据项与指标值读口——补 FEAT-0019 缺口）

> 标准任务（backend 只读端点）。BASE=815cabf。一个执行者做完整、统一测一次。与 TASK-0086（calc，后台并行）文件集/测试环境不交，可并行。前置 0054✅。

## 批次

- TASK-0087（本批 1 个）
- 验收粒度：一个 subagent 跑一次整体验收（backend mvn test）
- BASE：`815cabf`
- 改动范围（文件/目录级，越界反查用）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/**`（管理面只读控制器 + DTO；仅必要的查询组件可见性调整）
  - 或 `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/**`（若放 iot 域，如 statusread 旁新 read 包）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/**` 或对应 iot 测试包
  - **不得触碰**：frontend/**、calc/**、ingest/**、resources/**、resources/contract/**（冻结）、既有 HMAC 鉴权逻辑（OpenApiAuthFilter/requireIdentity）、既有 openapi 六接口行为、既有 iot 业务模块
  - 本任务**不改前端**（展示端接入另批）

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0087.md`
- 关联对象：`features/FEAT-0019.md`、`requirements/REQ-0028.md`、`REQ-0029.md`、`testcases/TC-0017.md`、`TC-0019.md`
- 缺口来源（要修的）：`reports/TASK-0059-review.md` D-4（按设备取数据项实时/历史值端点形态）、`TASK-0060-review.md` D-3（管理态 /iot/metric/values + /iot/metric/completeness）、`frontend/src/api/display/display-api.ts` 头注 + 指标段注释（前端消费契约）
- **可复用查询能力（不复制）**：`backend/.../openapi/support/OpenApiTimeseriesClient.java`（st_item_value/st_point/st_metric_value 查询、fail-soft 空集/白名单/行数上限/最新一条口径）、`.../service/impl/OpenApiQueryServiceImpl.java`（itemsRealtime :151 / itemsHistory / metricsQuery :223 的查询与裁剪逻辑）
- 数据权限（会话口径，必须复用）：`backend/.../iot/dataperm/DataPermService.java`（`resolve(userId)` 会话版，**不是** resolveClientScope）、`IotStatusReadController.java`（0050 登录态 + 数据权限裁剪范式）
- 完整度数据源：`calc/src/main/java/net/otlab/calc/web/MetricObserveController.java:38`（GET /calc/v1/metric 内存视图，calc 8095 内网）；`backend/.../iot/calcread/`（0063 只读 calc 客户端范式）
- 权限码：iot:instance:query（沿用 0061/0063/0062 口径）；读不审（0079 范式）

## 接口与决策（逐条结论，无选项）

### D1 · 四个只读端点（登录态）
1. `GET /iot/instance/{instanceId}/items/realtime?fullKeys=a,b,c`（批量≤200；返回 `Map<fullKey, {value,quality,ts}>`）——数据源 st_item_value 最新值（复用 OpenApiTimeseriesClient.queryItemLatest / itemsRealtime 逻辑）。
2. `GET /iot/item/history?fullKey=&start=&end=&interval=&agg=`（跨度/降采样/聚合与 HMAC items/history 同口径；返回时序点集）——复用 OpenApiTimeseriesClient。
3. `GET /iot/metric/values?metricCodes=&period=&start=&end=`（返回 metricCode → [{ts,value,yoy,mom}]）——复用 st_metric_value 查询（OpenApiTimeseriesClient metricsQuery 同款），**value/yoy/mom 同源展示不现算**。
4. `GET /iot/metric/completeness?metricCodes=&period=`（完整度）——数据源评估：calc `/calc/v1/metric` 内存视图在 calc 8095 内网，管理面可达性需评估；若可达则加只读客户端拉视图映射成 {metricCode, completeness}；**若不可达/无来源则诚实返回空集 + 明确空态字段（如 `available:false, reason:"完整度读口接入中"`），绝不返回伪造 0/1 或静默判完整**（REQ-0016 不造假口径）。报告说明评估结论。

### D2 · 鉴权与数据权限（与 HMAC 侧严格分离）
- 四个端点走**登录态**：`StpUtil.checkLogin()`（SaToken 拦截器已有）+ `@SaCheckPermission("iot:instance:query")` + 读不审（不加 @OperateLog）。
- **数据权限用会话版**：`DataPermService.resolve(SecurityUtils.getUserId())` 得 VisibleScope，按可见 instanceIds/fullKey 裁剪——**不是** HMAC 的 `resolveClientScope`（那是 appKey 调用方口径）。与 IotStatusReadController 完全同口径。
- scope 外/无权实例：集合类端点返回空集（不回显存在性），单资源类（如指定 instanceId 无权）403，文案不泄露资源标识。跨租户由多租户插件 + 会话 tenant_id 强制隔离。
- **不触碰 HMAC**：不引入/不读取 appKey/secret，不复用 requireIdentity（那是 HMAC 身份），不写 /openapi/v1/** 路径。

### D3 · 复用策略（禁止复制查询逻辑）
- 优先把 `OpenApiQueryServiceImpl` / `OpenApiTimeseriesClient` 的查询能力以 public 方法或提取共享组件方式复用（两个调用方：HMAC 六接口 + 管理面四端点同一份逻辑）。
- 差异只在"身份→scope"翻译：HMAC 走 resolveClientScope，管理面走 resolve(userId)——把差异约束在控制器/薄适配层，查询核心共享。
- 继承 fail-soft 空集、编码白名单、行数上限与截断、最新一条口径（OpenApiTimeseriesClient 已有），不另造。

### D4 · 测试口径（TDD 红转绿）
- 四端点各：登录态正常取数（mock timeseries client 返回样本）→ 200 且值/质量/时间正确；scope 外 → 空集/403；无权限 → 403；跨租户 → 隔离；输入校验（fullKeys 超 200 / 空 / 非法编码 → 400/空集不炸）。
- 完整度：有源时映射正确；无源时 `available:false` 空态、不造假（断言 value 不为 0/1）。
- 复用性：管理面与 HMAC 同参数查询返回一致结果（共享逻辑证据）。
- 既有 2559 基线零回归。

## 验收命令（必须原样贴回输出）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（含新增四端点契约测试 + 完整度空态 + 数据权限 + 既有 2559 零回归），贴 Results 段
2. 范围核对：`git diff --name-only 815cabf..HEAD` 全落改动范围声明；frontend/calc/ingest/resources/contract/HMAC 鉴权逻辑零触碰

## 报告

报告写到：`reports\TASK-0087-report.md`。五项齐全 + 一句话结论 + TDD 红转绿 + 数据权限裁剪证据 + 完整度无源诚实空集证据 + 复用策略说明（与 HMAC 共享逻辑但身份翻译分离）。

## 契约

- no-subagents；只改本批改动范围；不置 TASK 已完成；不改 workitems 索引。
- 登录态 + 会话数据权限（resolve(userId)），不碰 appKey/secret/HMAC；读不审；跨租户隔离。
- 完整度无源诚实空集不造假；value/yoy/mom 同源不现算；复用既有查询能力不复制。
- 不硬编码凭据/文档库路径；测试失败先定位根因，禁止为匹配输出改测试/改期望。
