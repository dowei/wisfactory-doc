# TASK-0059 diff 评审报告（代码评审）

> 评审者：fresh 子代理（no-subagents；只依据 review-brief 必读 + 工作区源码判断，未向协调者追问）
> 口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」
> 对象：`86a1c09`（BASE `e387c3b`；区间内 0042 `01236fd` 已隔离），13 文件全落 `frontend/**`
> 评审日期：2026-11-24

## 结论

**通过（无 Critical / Important）**——7 项评审重点均核对，安全红线、状态同源、缺口处置诚实性、降采样留痕、追溯无死链、下钻≤3、测试与三门口均成立。发现 3 条 Minor（曲线 2 条、追溯定位 1 条），按 §13 记 deferred，不进修订轮。

### 对「评审重点 3（API 缺口处置合理性）」的明确判定

**判定：诚实且合理，通过，非 Important。**

核验事实（工作区源码实证，非采信报告自述）：

1. 管理面确无「按设备取数据项实时/历史值」等价端点。`backend/.../module/openapi/controller/OpenApiController.java:63/71/100` 的 `items/realtime`、`items/history`、`points/history` 均挂在 `/openapi/v1/**`，且 `OpenApiAuthFilter.java:50` 明确 `PATH_PREFIX = "/openapi/v1/"` 全覆盖 HMAC 鉴权——浏览器不可直连；管理面仅 `IotStatusReadController`(`/api/v1/iot/status/*`，0050) 与 `IotPointServiceImpl.getPointCurve24h`(`/iot/point/{id}/curve`，0023 fail-soft 近 24h)。brief §3 / 报告 §3 的缺口事实陈述与源码一致。
2. 前端实现未造假、未写死链：绑定类项（sourceType=1）当前值取 `pointCurve` 末点且来源列如实标注「点位 #X（近 24h 曲线末点）」（`device-realtime-zone.vue:766,779`）；公式/汇总类项值列明确「需实时计算（数据接入中）」/「…端点待接入」（同文件 :766/:781-782），不是伪装真值。后端曲线 SQL `PointCurveQueryClient.java:67` 为 `ORDER BY ts ASC`，故“末点=最新”的派生成立，非时序倒序误读。
3. 曲线 >24h 跨度明示「数据源仅提供近 24h，长跨度曲线待历史端点接入」（`device-curve-zone.vue:34`），不静默伪造长跨度数据。
4. 前端零 appKey/secret、零 `/openapi/v1/**` fetch：对 `frontend/src/api/display/`、`views/display/`、`router/display.ts` 的 grep 仅命中注释中的安全红线说明，无任何实际凭据或 HMAC 直连代码。

缺口登记可执行性：报告 §3（:124-126）已写清**需要什么端点**——管理面登录态的「按实例批量取数据项实时值（v/q/ts）」与「按 fullKey + interval/agg 的历史值」等价端点，并写明必须复用 0050/iot 的数据权限裁剪、不得走 HMAC；数据源封装集中在 `display-api.ts`，替换面小。**「归哪个任务」尚未指定**，报告表述为「缺口登记，等裁决」——这与 brief 决策 3「无等价端点→登记缺口等裁决」一致，指派属协调者裁决权，执行者按契约未改 workitems 库，不构成执行者缺陷；记为 deferred 观察项，建议协调者在裁决时补挂一个后端任务并写入验收标准（见「deferred」条 D-4）。

如需补位，后端端点形态（供裁决参考，非本批要求）：`GET /iot/instance/{id}/items/realtime`（返回 v/q/ts，按数据权限裁剪）与 `GET /iot/item/history`（fullKey + interval/agg）；计算值/贡献值真值另需计算值读口（0063 留痕到位后）。

## 发现清单

### Critical
无。

### Important
无。

### Minor

- **M-1（曲线断线标注带在原始粒度档不可见）** — `frontend/src/views/display/device/device-curve-zone.vue:128`。`m.xAxis + selectedTier.bucketMs || 60_000` 因 `+` 优先于 `||` 实算为 `(m.xAxis + bucketMs) || 60000`；原始档（≤2h，`bucketMs=0`）得 `m.xAxis`（非零时间戳），兜底 `60000` 永不生效，markArea 起止 xAxis 相同→零宽灰带，断线时段标注不可见。修法：加括号 `m.xAxis + (selectedTier.bucketMs || 60_000)`。影响「近 1 小时」档的断线可视化，不涉数据正确性/超限红线标注。

- **M-2（时间范围选择器未真正裁剪显示窗口）** — `frontend/src/views/display/device/device-curve-zone.vue:96-106`。`from/to` 仅用于 `pickDownsampleTier`，曲线数据 `downsampleSeries(curve.dataPoints, …)` 未按 `from` 过滤；选「近 1 小时」实际仍渲染近 24h 全量点，只是标注变成「原始粒度」。数据真实、时间轴自证，非造假，但选择器主功能与文案不符；24h 默认档正确，7d/自定义档已诚实标注仅 24h。建议在 `load()` 中按 `from` 过滤 `points`（或改标签为“粒度选择”）。

- **M-3（组件清单「查看来源」未展开追溯面板，traceItemId 为死参数）** — `frontend/src/views/display/device/index.vue:127-131` + `frontend/src/views/display/device/device-trace-zone.vue:92`。`onTrace()` 调 `loadDetail()` 预取详情后仅 `router.push({ query: { traceItemId } })`，但 `device-trace-zone.vue` 从未读 `traceItemId`、`activeKey` 也未被设置，面板仍折叠；报告 §1 声称「跳追溯面板定位」实际不发生（内容可手动展开后查看，因 `loadDetail` 预取，故非死链）。建议由父级向 trace 组件传 `activeItemId` 并写 `activeKey`，或删除无效 query 写入。

## 评审重点逐条核对

| # | 重点 | 判定 | 证据 |
| --- | --- | --- | --- |
| 1 | 安全红线 | **通过** | grep 全 diff 范围：appKey/secret/HMAC/openapi 仅出现在注释；所有请求经 `getRequest` 走管理面登录态端点；`OpenApiAuthFilter.java:50` HMAC 仅覆盖 `/openapi/v1/`，前端零命中 |
| 2 | 状态区同源 | **通过** | `device-status-zone.vue` 仅调 `statusCurrent`/`statusHistory`（0050 读口）；statusName/color/时间线取服务端翻译，无硬编码状态名/色、无现算；大色块 `current.color ? … : --unknown`；`duration` 由 `changeTime` 派生、null→「未知」；UNKNOWN/quality=3/LWT 按读口原样展示。已核 VO 字段：`StatusCurrentVO`/`StatusHistoryPointVO` 与前端 TS interface 一一对应 |
| 3 | 实时区与缺口处置 | **通过（见上明确判定）** | 绑定类项取曲线末点并显式标源；公式/汇总类走空态不造假；缺口登记含端点形态与 HMAC 证据；仅「归哪个任务」待裁决（deferred） |
| 4 | 历史曲线 | **通过（带 M-1/M-2）** | 多选（`mode="multiple"`）、4 档时间范围、超限 markLine(1/2 红)/断线 markArea(3 灰) 齐备（MarkArea 缺陷见 M-1）；降采样分档 `pickDownsampleTier` + `downsampleSeries` 常驻标注 `annotation`，原始档默认「未降采样」，不静默降采样（ADR-0010 留痕口径）；TDengine fail-soft 空集以 `.catch(()=>null)` + `hasData=false` 走「数据接入中」空态，组件卸载 `dispose()`+移除 resize 监听。时间范围裁剪不实见 M-2 |
| 5 | 来源追溯 | **通过** | `itemImpact`（OUT 输入/IN 依赖）+ `itemEffectiveParams`（sourceType/formula/rollup/calcPeriod）展开；贡献值列显式「贡献值待计算值接口接入」；计算留痕脚注明示未交付、不写死链；接口 200 形状已由后端 `IotItemImpactVO`/`IotItemEffectiveParamsVO` 与 0059 冒烟 L8/L9 佐证 |
| 6 | 下钻与空态 | **通过（带 M-3）** | `/display/device?instanceId=` 静态注册、`name=DisplayDeviceDetail`、hideInMenu、需登录；车间页 `onOpenDevice` 加 click（禁用设备 `status===0` 跳过），携带 instanceId+factoryId/workshopId，`goBack` 正确回车间/总览；路径段计数 1/3/4/2（query 不增段）≤3；四区无数据源统一「数据接入中」/「暂无…」空态，零假数据；定位缺陷见 M-3 |
| 7 | 测试有效性 | **通过** | 本评审独立复跑：`pnpm build:test`=0、`pnpm lint`=0 errors（9 warnings 全在未改存量文件）、`pnpm typecheck`=0、`pnpm test` 63 文件 591 用例全绿（display 子集 18）；新增 8 用例覆盖质量码映射/超限红离线灰/降采样分档与标注必现/空输入/时长 UNKNOWN。live 8098 冒烟 10/10 + 曲线端点补测 200 依报告（:73-103），本评审未起服务独立复跑该环境项，标**无法判定**——但所依赖端点契约已与后端源码逐项核对一致；卸载清理（状态区清 interval、曲线 dispose/removeEventListener）成立 |

## 越界反查（复核）

`git diff-tree --no-commit-id --name-only -r 86a1c09` 输出 13 文件，全部 `frontend/**`（src 11 + 前端单测 2）；`grep -v '^frontend/'` 为空（ZERO）。BASE 后并行 calc `01236fd` 未 add/未提交/未触碰；backend/calc/ingest/契约/workitems 零改动。与 brief 声明集及 review-brief「越界反查」一致。

## deferred / 观察项（不进修订轮）

- **D-1（M-1）**：原始粒度档断线灰带零宽——建议下个前端小修补括号，附回归。
- **D-2（M-2）**：时间范围选择不裁剪数据——补 `from` 过滤或正名。
- **D-3（M-3）**：traceItemId 死参数 / 面板未自动展开——补 `activeKey` 联动或删无效 query。
- **D-4（重点 3 延伸）**：专用实时/历史值端点的归属任务未在报告指定（报告「等裁决」符合 brief）；建议协调者裁决时新建或指定后端任务并写验收标准，之后替换 `display-api.ts` 的封装即可，组件布局不受影响。
- **O-1（非本批缺陷，无法判定）**：brief/报告/代码注释将「降采样留痕」记作 ADR-0010、「同源」记作 ADR-0011，但 `workitems/decisions/ADR-0010.md`/`ADR-0011.md` 实际为 intake 第二/三批定版（ADR-0010 内含 TD-0030「重算痕迹以 iot_calc_log 为准」）。该标号用法在本批上游文档一致出现，非执行者引入；如为项目术语简称建议在术语表澄清，不影响本批代码验收。
- **O-2（无法判定）**：live 8098 冒烟与「数据项专用端点」缺口的环境态真值（如 TDengine 恢复后曲线/状态历史读回）本评审未起服务复跑，依报告与后端契约判定。

## 一句话结论

**通过**——security/同源/缺口诚实性/降采样留痕/追溯无死链/下钻≤3/591 用例与三门口全部核实成立，无 Critical/Important；3 条 Minor 记 deferred，第 3 点 API 缺口处置判定为诚实合理、非 Important，缺口登记清晰到端点形态，归属任务待协调者裁决补挂。
