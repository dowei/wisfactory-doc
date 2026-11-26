# 实施 brief：TASK-0089（展示端接入管理面值读口——设备实时区与指标看板接真数据）

> 标准任务（纯 frontend）。BASE=2d0f1a9。一个执行者做完整、统一测一次。前置 0087✅（四只读端点已交付）。FEAT-0019 收尾任务。

## 批次

- TASK-0089（本批 1 个）
- 验收粒度：一个 subagent 跑一次整体验收（frontend pnpm 三门 + test）
- BASE：`2d0f1a9`
- 改动范围（文件/目录级，越界反查用）：
  - `frontend/src/api/display/display-api.ts`（加四封装 + 类型）
  - `frontend/src/views/display/device/device-realtime-zone.vue`（实时区取数替换）
  - `frontend/src/views/display/metrics/index.vue`（指标看板取数替换）
  - `frontend/tests/unit/display/**`（读口封装/空态/安全红线测试）
  - **不得触碰**：组件与纯函数（metric-card.vue/metric-trend.vue/metric-utils.ts/trendDisplay/completenessDisplay/buildMetricTrend/device-utils.ts/display-factory-card.vue 等）、backend/**/calc/**/ingest/**/resources/contract/**、display-api.ts 既有封装签名（只增不改）

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0089.md`
- 关联对象：`features/FEAT-0019.md`、`requirements/REQ-0029.md`、`REQ-0030.md`、`testcases/TC-0019.md`
- 0087 端点契约（权威，逐字对齐）：
  - `backend/.../displayread/controller/IotDisplayReadController.java:50/59/77/88`
  - `GET /iot/instance/{instanceId}/items/realtime?fullKeys=a,b,c` → `Result<Map<fullKey, {v:number|null, q:int, ts:long}>>`（OpenApiRealtimeItemVO；scope 外实例 403/外键静默裁剪）
  - `GET /iot/item/history?fullKey=&start=&end=&interval=&agg=` → `Result<[{ts,value,quality}...]>`
  - `GET /iot/metric/values?metricCodes=&period=&start=&end=` → `Result<[{metricCode,instanceCode,period,ts,value,yoy,mom,unit,compareNote}...]>`（MetricValueVO；value/yoy/mom 同源不现算，null 不冒充 0）
  - `GET /iot/metric/completeness?metricCodes=&period=` → `Result<{available:boolean, reason?, rows:[{metricCode,completeness:number|null,incomplete:boolean|null}]}>`（**当前 available=false 诚实空集**）
- 替换点：
  - `frontend/src/views/display/device/device-realtime-zone.vue:98-136`（现绑定类走 pointCurve 末点派生、公式/汇总类空态；要改成走 realtime 端点真值）
  - `frontend/src/views/display/metrics/index.vue:105-151`（现 valueRows/completenessMap 恒空 + readGap 告警；要接 metric/values + metric/completeness）
- 前端安全红线 + 空态纪律：`frontend/src/api/display/display-api.ts` 头注（不持 appKey/secret、不直连 openapi HMAC、不自造代理）、0059/0060 review 报告

## 接口与决策（逐条结论，无选项）

### D1 · display-api.ts 加四封装（只增不改既有签名）
- `itemsRealtime(instanceId, fullKeys): Promise<Record<string, {v:number|null,q:number,ts:number|null}>>` —— getRequest('/iot/instance/{id}/items/realtime', { fullKeys: fullKeys.join(',') })，asArray/空对象容错。
- `itemHistory(fullKey, start, end, interval?, agg?): Promise<IotHistoryPoint[]>` —— getRequest('/iot/item/history', ...)，asArray。
- `metricValues(metricCodes, period, start, end): Promise<MetricValueRow[]>` —— getRequest('/iot/metric/values', ...)，asArray；**复用现有 MetricValueRow 类型**（display-api.ts :235 附近，字段 ts/value/yoy/mom/metricCode/instanceCode/period/unit 已对齐 0087 VO）。
- `metricCompleteness(metricCodes, period): Promise<{available:boolean, reason?:string, rows:MetricCompletenessRow[]}>` —— getRequest('/iot/metric/completeness', ...)，rows asArray；**复用现有 MetricCompletenessRow 类型**。
- 全部走管理面已登录态 axios（getRequest），不触 appKey/secret/HMAC。

### D2 · 设备实时区替换（device-realtime-zone.vue）
- load 逻辑：先 queryItemList 拿数据项 → 收集各项 fullKey → 一次 `displayApi.itemsRealtime(instanceId, fullKeys)` 拿真值；每项按 fullKey 取 {v,q,ts}。
- 绑定类/公式/汇总类**统一走 realtime 真值**（0087 端点对所有 sourceType 返回 v/q/ts）；保留 `realtimeDisplayState(quality, value, item)` 的状态判定（同源不现算）与超限标红/离线灰显；来源文本改为如实标注「实时值读口」（不再写"曲线末点"）。
- 空态：realtime 无该项 / v=null → 显示「数据接入中」，**不展示假数据**；读口整体失败（catch）走空态 + console.error，不炸页面。

### D3 · 指标看板替换（metrics/index.vue）
- `loadAll` 内：queryMetricList 拿定义后，用设备/工厂维度下的 metricCodes + period + 时间窗调 `displayApi.metricValues` → 映射填入 `valueRows`（复用现有 buildMetricTrend/metric-utils 纯函数，不改它们）；调 `displayApi.metricCompleteness` → 若 `available=true` 把 rows 的 completeness 填入 `completenessMap`，**若 available=false 保持 completenessMap 空 + readGap 告警**（"完整度接入中（读口待接入）"文案不变，不静默判完整）。
- 值/同比/环比同源展示（value/yoy/mom 直接来自读口，不现算）；无值行/null 伴随值保持空态不冒充 0。
- **不改组件/纯函数**（brief 前置约定）——只换 loadAll 里的数据来源与 readGap 判定（available=false 时 readGap 保持 true）。

### D4 · 测试口径
- display-api 四封装各：正确 URL/参数、响应容错（null/undefined→空）、安全红线（全文件零 appKey/secret/HMAC 直连）。
- 实时区：realtime 真值渲染（绑定+公式类都出真值）、v=null 空态、读口失败空态、来源标注正确。
- 指标看板：metricValues→valueRows 映射正确（同源不现算）、completeness available=true 填 map / available=false 保持空态+告警、null 不冒充 0。
- 既有 display 单测零回归（0058/0059/0060 的 610+ 用例）。

## 验收命令（必须原样贴回输出）

1. `cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test` → 0 错误 0 失败（含新增读口封装/空态/安全红线测试 + 既有 610+ 零回归），贴原始输出
2. 范围核对：`git diff --name-only 2d0f1a9..HEAD` 全落改动范围声明；组件/纯函数/backend/calc/ingest/contract 零触碰

## 报告

报告写到：`reports\TASK-0089-report.md`。五项齐全 + 一句话结论 + 安全红线证据（零 appKey/secret/HMAC）+ 空态诚实证据（无源不造假）+ 组件纯函数零改动证据。

## 契约

- no-subagents；只改本批改动范围；不置 TASK 已完成；不改 workitems 索引。
- 安全红线：只调管理面登录态 API、不持 appKey/secret、不直连 /openapi/v1/**、不自造代理。
- 空态纪律：无数据/读口不可用显示"数据接入中"，在线率/完整度不造假，null 不冒充 0；完整度 available=false 时保持告警空态。
- 组件/纯函数零改动；同源不现算；测试失败先定位根因，禁止为匹配输出改测试/改期望。
