# TASK-0089 diff 评审报告（标准任务首评）

> 评审对象：提交 `9459e34`（BASE `2d0f1a9`，6 文件纯 frontend，744 行 diff 包）。
> 角色：独立 fresh 评审（no-subagents，只读源码，不改代码/不推进状态）。
> 本轮为原评审因 API 限流中断后的重做；所有结论均回到源码与 git 独立核实，未采信报告自述。

## 结论

**退回执行者一轮（1 Important，0 Critical）**。6 项评审重点中安全红线、空态诚实（含 `available=false`）、端点契约、组件/纯函数零改动、范围、测试与零回归均判定通过；但指标看板「本期值」取数存在一处会实际显示错误周期数值的质量缺陷（见 I-1），按 L1-execute §13 / §12 走退回路径。复评仅看 I-1 的修复 diff，不重开全场。

---

## 一、发现清单

| 编号 | 严重级 | 出处（绝对/仓库相对路径 + 位置） | 一句话说明 |
| --- | --- | --- | --- |
| I-1 | **Important** | `D:\workspace\WisFactory\frontend\src\views\display\metrics\index.vue:121-124`（`valueOf()`，本 diff 未改但与本次取数激活直接交互）＋ 本 diff 同文件 `:279-291`（366 天时间窗 + `valueResults.flat()`）＋ `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/support/OpenApiTimeseriesClient.java:43-48`（`ORDER BY ts ASC`） | `valueOf()` 按 `metricCode` 取**首行**，而 `/iot/metric/values` 对 366 天窗口返回按 `ts` **升序**的多行，卡片「本期值」会显示窗口内**最早（约一年前 / 去年对比行）**的值而非本期值——本次把 `valueRows` 从恒空改为真值后该潜伏缺陷被激活，且 0060 review D-4 已预警「读口返回多行需补 period/instanceCode 约束」未被闭合。 |

无 Critical。无 Minor 需 deferred（报告自述的 menu-cache-frame 偶发超时见第五节，按环境抖动登记，不构成发现）。

### I-1 证据链（逐条实测核实）

1. **后端返回顺序为 ts 升序**：`OpenApiTimeseriesClient.queryMetrics()` SQL 为 `... AND ts >= startMs AND ts <= endMs ORDER BY ts ASC LIMIT FETCH_LIMIT`，`OpenApiQueryServiceImpl.metricsQuery()` 按 `for (code : codes)` 逐码 `result.addAll(page.rows())` 保序；`IotDisplayReadServiceImpl.metricValues()` 仅做字段映射（`windowStart→ts`、period 回填），不改顺序。因此同一 `metricCode` 的行在结果集中为 `ts ASC`。
2. **本 diff 的查询窗口必然含多行**：`index.vue` 新增 `start = end - 366*24*60*60*1000`，注释明确「覆盖去年同 ts 对比行」；`buildMetricTrend()` 亦按「结果集中是否存在 `ts+1年` 同伴行」识别对比行。366 天窗口覆盖去年同 ts 行（实测：now-1year ≥ now-366d，成立）。
3. **消费端取首行**：`valueOf()` = `valueRows.value.find((r) => r.metricCode === metric.metricCode) ?? null`，`valueRows.value = valueResults.flat()` 无排序/无截断。`metric-card.vue` 模板 `v-if="value && value.value != null"` 直接显示该行 `value.value` 作为「本期值」，yoy/mom 同源取同一行。
4. **后果**：`find()` 返回 `ts ASC` 的首行 = 窗口最早行 = 去年对比行/一年前的值，被标为「本期值」显示。数值真实但周期错误，属核心交付物（指标卡片本期值）的错误结果，无绕过手段。
5. **测试漏检**：`metrics-board-read-ports.spec.ts` 全部 `metricValues.mockResolvedValue` 每个 metricCode 仅 1 行（`TS = 2026-11-20` 单行），未覆盖「同码多行 + ts 升序」的真实形态，故 20 测试全绿未暴露此问题。
6. **非本轮新增代码但由本 diff 激活**：`valueOf()` 在 BASE `2d0f1a9` 即存在且本次未改；改前 `valueRows.value = []` 恒空（卡片恒空态），`valueOf()` 恒返回 null，无实际影响（0060 review D-4 据此记 deferred）。本次 diff 使其首次处理真实多行数据，故为本次改动引入的实际缺陷。
7. **归属/修复方向**：在 `index.vue` 内按「同 metricCode 取最新非对比行（ts 最大且不属于 `buildMetricTrend` 的去年对比集）」或按 `period + instanceCode` 约束定位本期行；`metric-card`/`metric-utils` 等组件与纯函数仍不改。修复后可补一条「同码多行 ts 升序 → 卡片显示最新行」的断言。

---

## 二、评审重点逐项判定

### 重点 1 · 安全红线 —— **通过（头号重点，明确判定）**

回到源码独立扫描（非采信报告）：

- `git show 9459e34:frontend/src/api/display/display-api.ts` 剔除注释后扫描 `appKey|appSecret|createHmac|HMAC|secret|/openapi/v1|fetch(` → 唯一命中 `import { getRequest } from '/@/lib/axios'`（管理面登录态 axios），零凭据、零 HMAC、零 openapi 直连、零原生 fetch/自造代理。
- 两个视图代码区（剔除注释/文案）扫描同款模式：`device-realtime-zone.vue` 零命中；`metrics/index.vue` 唯一命中为 `<a-alert>` 的 `description` 文案「前端零 appKey/secret、零 HMAC 直连、不自造代理」——声明性文字、非可执行实现，且语义为「不持有/不直连」，不构成混淆或误导。
- 四封装全部 `getRequest('/iot/...')`：文件内 `getRequest(` 共 13 处（9 既有 + 4 新增），无 `postRequest`/`http.get`/`axios.` 旁路。
- **既有签名未改**：`git diff 2d0f1a9 9459e34 -- display-api.ts | grep '^-[^-]'` → 零删除行；diff 仅在接口 `MetricCompletenessRow` 之后追加类型与在 `displayApi` 对象末尾追加四方法。

判定：安全红线成立，无 C/I。

### 重点 2 · 空态诚实 —— **通过（头号重点第二条，明确判定）**

- **realtime 无该项 / v=null**：`device-realtime-zone.vue` 中 `live = realtime[fullKey] ?? null`、`rawValue = live?.v ?? null`，`valueText = rawValue == null ? '数据接入中' : \`${rawValue}${unit}\``——0 不作为 null 替身，`v=null` 走「数据接入中」；`qualityLabel = live ? state.qualityLabel : '—'`，无该项时质量标记不伪造；`updateTime = live?.ts != null ? formatTs : '—'`。`itemsRealtime()` 对 `data=null/非对象/数组` 归一 `{}`。
- **读口失败**：`itemsRealtime(...)` 外层 catch → `rows=[]` + `console.error`，页面走 `v-else`「数据接入中（无实时值数据源，不展示假数据）」。
- **指标值 null 不冒充 0**：`metricValues()` 原样 `asArray` 透传，`MetricValueRow.value/yoy/mom` 为 `number|null`；`metric-card.vue`（未改）`v-if="value && value.value != null"` 空态 + `trendDisplay(null)` → `'—'`（`metric-utils.ts` 未改）。看板 `valueOf()` 返回 null 时卡片空态。
- **完整度 `available=false`（当前 0087 诚实空集）**：`metricCompleteness()` 对 `data=null/非对象` fail-closed 返回 `{available:false, rows:[]}`；`index.vue::loadAll()` 仅 `available === true` 才 `completeness.set(...)` 且 `row.completeness != null` 才写入，`anyAvailable` 为 false 时 `completenessMap` 保持空、`readGap = !anyAvailable = true`，`<a-alert>` 保持「完整度接入中（读口待接入）」告警。无伪造百分比、无伪造角标、无静默判完整。**明确判定：通过。**
- 局部降级：`metricValues(...).catch(()=>[])`、`metricCompleteness(...).catch(()=>({available:false,rows:[]}))` 独立 fail-soft，任一失败只让对应数据走空态；外层 catch 亦 `readGap = true`。

### 重点 3 · 端点契约对齐 0087 —— **通过**

| 封装 | 本 diff URL / 参数 | 0087 controller（权威） | 判定 |
| --- | --- | --- | --- |
| itemsRealtime | `getRequest('/iot/instance/${id}/items/realtime', {fullKeys: join(',')})` | `@GetMapping("/iot/instance/{instanceId}/items/realtime")` + `@RequestParam("fullKeys") String` | 对齐 |
| itemHistory | `/iot/item/history` `{fullKey,start,end,interval,agg}` | 同路径 + 同名 5 参数 | 对齐 |
| metricValues | `/iot/metric/values` `{metricCodes,period,start,end}` | 同路径 + 同名 4 参数 | 对齐 |
| metricCompleteness | `/iot/metric/completeness` `{metricCodes,period}` | 同路径 + 同名 2 参数 | 对齐 |

响应字段：`RealtimeItemVO{v:number|null,q:number,ts:number|null}` 对齐 `OpenApiRealtimeItemVO(v/q/ts)`；`IotHistoryPoint{ts,v:number|null,q}` 对齐 `OpenApiHistoryPointVO(ts/v/q)`；`MetricCompletenessVO{available,reason?,rows}` 对齐后端 `MetricCompletenessVO`；`MetricValueRow`/`MetricCompletenessRow` 为既有类型，字段与 `MetricValueVO`/`MetricCompletenessVO.Row` 对齐。容错 `asArray` 与 realtime 空对象/fail-closed 分支均实测用例覆盖。

说明：`itemHistory` 在代码注释中写 `→ [{ts,v,q}]`，brief §必读处写 `[{ts,value,quality}]`；以 controller 实际返回的 `OpenApiHistoryPointVO(ts/v/q)` 为准，实现与代码注释一致。该封装当前未被两视图调用（预留封装），不影响本次交付；仅措辞差异，非缺陷。

### 重点 4 · 组件/纯函数零改动 —— **通过**

`git diff 2d0f1a9 9459e34` 逐一核实：`metric-card.vue`、`metric-trend.vue`、`metric-utils.ts`（含 `trendDisplay`/`completenessDisplay`/`buildMetricTrend`/`valueOf` 之外的纯函数）、`device-utils.ts`、`display-factory-card.vue` 全部 `UNCHANGED`；backend/calc/ingest/contract 在提交内零文件。看板仅替换 `loadAll()` 取数处与 `readGap` 判定，模板仍绑定 `metric-card`/`metric-trend` 与 `valueOf`/`completenessOf`。注：I-1 涉及的 `valueOf()` 虽在改动文件 `index.vue` 内，但该行本身未改，故不破坏本条「零改动」判定；其修复亦应限于 `index.vue`。

### 重点 5 · 测试有效性与零回归 —— **通过**

本评审独立实测（非仅采信报告）：
- `pnpm exec vitest run` 三份新测试：`3 passed / 20 passed`（display-read-ports 10、device-realtime-zone 5、metrics-board-read-ports 5）。
- `pnpm typecheck`（`vue-tsc --noEmit`）：exit 0、无输出。
- `pnpm exec eslint`（6 个改动文件）：exit 0、零 warning/error。
- `pnpm test` 全量：`Test Files 69 passed (69) / Tests 630 passed (630)`，exit 0；= 610 基线 + 20 新增，既有 0058/0059/0060 display 测试零回归。
- 测试真锁 URL/参数/空态/安全扫描，非快照：`display-read-ports.spec.ts` 逐字断言四 URL 与参数对象、`data=null/非数组` 容错、源码剔除注释后扫 `appKey|secret|HMAC|/openapi/v1|fetch(`；`device-realtime-zone.spec.ts` 断言 `pointCurve` 零调用 + `itemsRealtime(42,[...fullKeys])`、v=null 不含 `0℃`/`null`、读口失败空态；`metrics-board-read-ports.spec.ts` 断言 value/yoy/mom 原样、available=true 填 map/低于阈值标记、available=false 不出现「完整度 100%」。
- 报告登记的 `menu-cache-frame.spec.ts` 偶发 5000ms 超时：本轮全量复跑未复现（69/69 绿），按环境负载抖动登记合理，未改测试掩盖。

覆盖缺口：见 I-1 第 5 点（同码多行 ts 升序未覆盖），该缺口即为 Important 发现的成因，不单列发现。

### 重点 6 · 范围 / 越界反查 —— **通过**

- 提交身份：`git cat-file -t 9459e34 = commit`；父 `2d0f1a9`；HEAD = `ee767db`。
- `git diff-tree --no-commit-id --name-status -r 9459e34` = 6 行：3 改（display-api.ts、device-realtime-zone.vue、metrics/index.vue）+ 3 增（三份 spec），全部落 brief 声明范围；backend/calc/ingest/contract/组件/纯函数零触碰。
- diff 包 `TASK-0089-diff.txt` 行数 744 = `git diff 2d0f1a9 9459e34` 行数 744，内容一致（隔离可信）。
- 并行 0037 隔离：`git diff --name-only 9459e34 HEAD` = 17 文件，全部 `backend/ot-admin/.../dataitem/**` + `application.yml`，零 frontend 文件——0037 backend 变更未混入本 diff，本评审只覆盖 0089 frontend。

---

## 三、契约逐条核对

- **fresh / no-subagents / 只评审不修改（报告除外）**：本轮新会话重做，未派子代理，未改任何代码；仅写本报告。
- **标准任务首评**：1 Important → 退回执行者一轮，复评上限 1 轮且 scoped 到 I-1；无 Minor 需 deferred。
- **退回路径（L1-execute §12）**：失败输出原样退回执行者重做，不现场改验收标准、不自行修代码；TASK 保持「进行中」不置完成。

---

## 四、给执行者的修复要点（I-1）

1. 仅改 `frontend/src/views/display/metrics/index.vue`：`valueOf()` 在同 `metricCode` 的多行中选定**本期/最新**行（如按 `ts` 取最大且排除 `buildMetricTrend` 认定的去年对比行；或按 `period + instanceCode` 精确约束），不触碰 `metric-card.vue`/`metric-utils.ts`/`buildMetricTrend` 等组件与纯函数。
2. 补一条测试：同一 `metricCode` 返回 `ts` 升序多行（含去年对比行）时，卡片显示最新本期值而非首行旧值。
3. 复评只审该修复 diff 与 I-1，不重开全场；其余五项已通过结论沿用。

---

## 一句话结论

**退回一轮**——安全红线（零凭据/零 HMAC/零代理、签名未改）与完整度 `available=false` 空态（不静默判完整、不造假）两项头号重点均明确通过，端点契约、组件/纯函数零改动、范围隔离、20 新增 + 610 基线全绿亦通过；但 I-1（指标卡片「本期值」按 `metricCode` 取 ts 升序首行，接入真值后会显示一年前/去年对比行的旧值）为 Important 质量缺陷，按 §13/§12 退回执行者一轮，0 Critical。
