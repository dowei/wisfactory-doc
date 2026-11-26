# TASK-0060 diff 评审报告（代码评审）

> 评审者：fresh 子代理（no-subagents；只依据 review-brief 必读 + 工作区源码判断，未向协调者追问）
> 口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」
> 对象：实现提交 `bf84526`（BASE `4dd8144`；区间内仅此 1 提交，9 文件全落 `frontend/**`）
> 评审日期：2026-11-24
> 契约依据：`resources/contract/tdengine-super-tables.md` 冻结稿 v1.0.0（`st_metric_value` = ts/value/yoy/mom 4 列 + 四元组 tag，无完整度物理列）
> 契约约束：fresh、no-subagents、只评审不修改任何文件（本报告为唯一写入）

## 结论

**通过（0 Critical / 0 Important / 2 Minor，均不阻断）**——7 项评审重点逐项核对无 Critical/Important。
**安全红线（重点 1）成立**：全 diff 零真实凭据、零 HMAC/openapi 直连、不自造代理，源码级复核管理面确无指标值/完整度登录态读口，空态 + 缺口登记诚实。
**完整度空态处置（重点 3）成立**：无物理列、无端点的完整度一律 null → 显式「接入中（读口待接入）」，不静默判完整、不造假数字，80% 阈值与 0045 同口径。
独立复跑：pnpm test 64 文件 601 用例全绿、typecheck 0 error；范围核对 9 文件全 `frontend/**`、越界 0。2 条 Minor（工厂维度下拉不触发重过滤、period 注释与实现不符）按 §13 记 deferred，不进修订轮。

---

## 一、对「评审重点 1（安全红线）」的明确判定

**判定：通过，安全红线成立，非 Important。**

源码级复核事实（均非采信执行者报告自述）：

1. **全 diff 扫描**：`git diff 4dd8144..bf84526` 中 `appKey/appSecret/createHmac/secret//openapi/v1` 命中全部落在**注释、文件头说明、页面缺口告警文案**（diff L18/50/98/99/125/460 等），描述的是「不持有/不直连/不自造代理」；无任何真实凭据常量、无 `crypto.createHmac`、无 `fetch/axios/XMLHttpRequest` 到 openapi 的调用。新增行单独过滤结果同上（仅注释/文案命中）。
2. **新增目录 import 全景**：`frontend/src/views/display/metrics/**` 仅 import `vue`、`echarts`、`@ant-design/icons-vue/ReloadOutlined`、`display-api`（getRequest 封装）、`use-display-tree`、本地组件/`metric-utils`/`device-utils`、`display.less`。零直接 HTTP、零 openapi SDK、零 crypto。
3. **唯一取数**：`index.vue::loadAll` 经 `displayApi.queryMetricList(f.id)` → `getRequest('/iot/metric/list', {instanceId})`（`display-api.ts:344-350`），管理面登录态端点、后端数据权限裁剪。
4. **读口缺口源码实证**（复核越界反查）：
   - `IotMetricController.java:58-137` 仅有 `/list`、`/{id}`、POST、`/generate`、PUT、`/changeStatus`、DELETE——全是**指标定义 CRUD，无值读口**；
   - 指标值唯一读口 `OpenApiController.java:86 GET /openapi/v1/metrics/query`，控制器类级 `@Anonymous`（:44，不进 Sa-Token 登录态），但被 `OpenApiAuthFilter.java:50 PATH_PREFIX="/openapi/v1/"` 全覆盖 HMAC 鉴权——浏览器直连需前端持有 appKey/secret，安全红线禁止；
   - 完整度无物理列，仅由 calc 内存视图 `/calc/v1/status.metric`（0045 已实证，内网观测面）承载，ot-admin 无对应登录态端点。
5. **处置**：值/完整度/趋势 `valueRows=[]`、`completenessMap=new Map()` 恒空，页面常驻 `a-alert` 缺口登记（`readGap=true`），报告 §3.1 写明「需要什么端点」（管理态 `/iot/metric/values` + `/iot/metric/completeness`，复用数据权限裁剪、禁走 HMAC）。不自造代理、不造假。

结论：安全红线与 0059 先例同 discipline，判定**通过**。

---

## 二、对「评审重点 3（完整度空态处置）」的明确判定

**判定：通过，空态处置诚实、不静默、不造假，非 Important。**

1. **80% 阈值同口径**：`metric-utils.ts:62 COMPLETENESS_MIN = 0.8`，与 0045 `CalcProperties.completeness-min` 默认 0.8 一致；`metric-utils.spec.ts` 有 `expect(COMPLETENESS_MIN).toBe(0.8)` 硬断言。
2. **无物理列 → 空态登记**：`completenessMap` 恒空，`completenessOf()` 返回 null；`completenessDisplay(null)` → `{value:null, incomplete:false, text:'完整度数据接入中（读口待接入）'}`（`metric-utils.ts:61-64`）。卡片渲染该文案（`metric-card.vue:51-52`），**不显示任何伪造百分比、不显示「完整」**。此处 `incomplete:false` 仅用于控制橙色「完整度不足」角标是否出现（null 时不出角标），并非判定为完整——展示文案明确「接入中」，故不构成「静默判完整」。
3. **不造假标记**：低于阈值 `value < 0.8` 才置 `incomplete=true` 并显式文案「· 低于 80% 阈值」+ 角标 + 橙色上边框；无端点时不伪造低标记、也不伪造正常标记。
4. **单测锁约束**：`completenessDisplay` 三条用例分别锁阈值 0.8、低于阈值带标记、null→「读口待接入」不静默判完整。真锁约束，非快照。

结论：完整度空态处置**通过**。唯一遗留是「管理面完整度读口缺失」本身（环境/归属缺口，报告 §3.1 已登记等裁决），属 deferred，非执行者缺陷。

---

## 三、发现清单

| # | 严重级 | 出处 | 一句话说明 |
| --- | --- | --- | --- |
| M-1 | Minor | `index.vue:49`（a-select v-model）+ `:86/:134`（selectedFactoryId 仅在 loadAll 使用）+ `:159`（仅 onMounted 调用） | 工厂维度下拉无 `@change`、无 `watch(selectedFactoryId)`：选中单厂后卡片不重新过滤（仍显示全部工厂指标定义），而 `factoryHint` 却提示「仅展示所选工厂指标定义」——UI 提示与实际不符。须再点刷新才生效。当前值恒空态、指标定义仍为权限内真数据、无数据完整性/安全影响，且与 0059 M-2（选择器不裁剪）同级，故记 Minor/deferred。修法：`@change="loadAll()"` 或 `watch(selectedFactoryId, () => loadAll())`。 |
| M-2 | Minor | `index.vue:116-121`（periodSections 注释与实现） | 注释称「未知 period 归 DAY 段，避免指标丢失」，实现仅 `?? 'DAY'` 处理 null/undefined；非空非法 period 不匹配四段中任一段而被静默丢弃。后端 `IotMetricServiceImpl.PERIOD_VALUES=Set.of(HOUR,DAY,MONTH,YEAR)` 校验保证不会出现非法值，故无实际影响，仅注释不准确。 |

> 无 Critical、无 Important。

---

## 四、评审重点逐条核对

| # | 重点 | 判定 | 证据 |
| --- | --- | --- | --- |
| 1 | 安全红线 | **通过**（见 §一明确判定） | 全 diff 零真实凭据/零 HMAC/零 openapi 直连/零代理；取数仅 `getRequest('/iot/metric/list')`；管理面无值读口、值仅在 HMAC openapi、完整度仅 calc 内网——源码逐项核实；空态 + 缺口告警 + 端点登记齐备 |
| 2 | 同源纪律 | **通过** | `MetricValueRow`（display-api.ts:238-248）逐字对齐冻结契约 4 列 ts/value/yoy/mom + 四元组 tag（unit 显式标注为非物理展示字段）；value/yoy/mom 全程只读传入、前端零现算；`trendDisplay` 仅做比率→文案/方向色派生；涨跌方向色集中 `COLOR_UP/DOWN/FLAT` 一处登记并 javadoc 标注 ADR-0011 租户可配演进位 |
| 3 | 完整度空态 | **通过**（见 §二明确判定） | 0.8 阈值与 0045 同口径且单测锁定；null→「接入中（读口待接入）」，不静默判完整、不造假数字/标记 |
| 4 | 趋势图 | **通过** | ECharts 多指标曲线按 `metricCode\|instanceCode` 分组叠加（`metric-trend.vue:529-567`）+ 周期切换过滤（`buildMetricTrend` 按 period）；去年同期仅取真实同 ts 行（`comparisonTs` 识别 + `oneYearBefore` 取真实 value，yoy 比率不反推），无对比行标注「不造假对比曲线」；降采样复用 `pickDownsampleTier` 且标注常驻（ADR-0010）；空行 → `hasData=false` 空态；卸载 `dispose()` + 移除 resize 监听 |
| 5 | 上看板标记/过滤/空态/路由 | **通过（带 M-1/M-2）** | `mark` prop + `--marked` 放大/置顶样式已留演进位（接口未交付前缺省）；工厂维度过滤控件与按周期分组渲染均在，但下拉不触发重过滤（M-1）、period 注释不准（M-2）；空态统一「指标数据接入中」；`/display/metrics` 静态注册、hideInMenu、无 meta.public，经 `router/index.ts:244` 既有 `displayRouters` push 挂 OtLayout（:274-275），总览页入口按钮 `router.push({name:'DisplayMetrics'})` |
| 6 | 测试有效性 | **通过** | 本评审独立复跑：`pnpm test` 64 文件 601 用例全绿（display 子集 5 文件 28 用例）、`pnpm typecheck` 0 error；`display-routes.spec.ts` 真锁 5 路由路径/命名/段数/需登录/hideInMenu；`metric-utils.spec.ts` 10 用例真锁方向色与 null 不冒充 0、0.8 阈值与读口缺口空态、按周期过滤/同期真实行不反推/降采样标注必现/同 ts 覆盖——非纯快照。live 8098 缺席按 brief「忙则跳过登记」，报告有 curl HTTP=000 + netstat 无监听原始输出，登记合理；本评审未起服务，环境真值标**无法判定** |
| 7 | 零回归与范围 | **通过** | 独立复核 `git diff --name-only 4dd8144..bf84526` = 9 文件全 `frontend/**`、`grep -v '^frontend/'` 为空；区间仅 1 提交；全量 601 用例含 0058/0059 既有 display 测试（device-utils 8 / display-routes 3 / display-slot-card 2 / display-tree 5）零回归；backend/calc/ingest/契约/workitems 零触碰 |

---

## 五、越界反查（复核）

`git rev-parse HEAD` = `bf84526035fb2d2…`（= `bf84526`）；`git log 4dd8144..bf84526` 仅 1 提交（TASK-0060）；`git diff --name-only 4dd8144..bf84526` 输出 9 文件，全部 `frontend/**`（src 6 + tests 2 + 路由/总览已在 src 6 内），`grep -v '^frontend/'` → ZERO。与 brief 声明集「9 文件 frontend/**」及 review-brief「越界反查」一致。工作区仅 `?? .agents/`、`?? .tmp-t0044v/`（开工既存未跟踪，本任务未触碰）。

---

## 六、deferred / 观察项（不进修订轮）

- **D-1（M-1）**：工厂维度下拉补 `@change="loadAll()"` 或 watch，使选择即时生效；同步修正 `factoryHint` 与实际展示的一致性。
- **D-2（M-2）**：订正 `periodSections` 注释（把「未知 period 归 DAY 段」改为「null/undefined 归 DAY 段」），或补真正的未知 period 兜底以避免将来后端校验放宽时丢指标。
- **D-3（读口缺口归属，重点 1/3 延伸）**：管理面指标值/完整度登录态读口尚未指定归属任务（报告 §3.1「等裁决」符合 brief）；建议协调者裁决时补挂后端任务并写验收标准（端点形态报告已给），之后仅替换 `metrics/index.vue::loadAll` 取数处，组件/纯函数不动。
- **D-4（未来读口落位提示）**：`valueOf()` 当前仅按 `metricCode` 取首行（`index.vue`），依据 `uk_tenant_metric_code` 租户内唯一成立；但读口若返回同 metricCode 的多周期/多实例行，需补 `period/instanceCode` 约束以免绑定到错误周期的行。当前 `valueRows` 恒空，无实际影响。
- **O-1（术语，无法判定/非本批引入）**：注释将「同源」记 ADR-0011、「降采样」记 ADR-0010，而 `workitems/decisions/ADR-0010.md`/`ADR-0011.md` 实为 intake 第二/三批定版。该标号用法在 0045/0059 上游文档一致出现（见 TASK-0059-review O-1），非本批引入，建议术语表澄清，不影响验收。
- **O-2（无法判定）**：live 8098 走查与 TDengine 6041 真机读回的环境态真值本评审未起服务复跑，依报告与后端契约判定；TD-0053 环境阻断按同 0042/0045/0059 口径登记。

---

## 一句话结论

**通过**——安全红线（重点 1）与完整度空态处置（重点 3）均给出明确通过判定：零凭据/零 HMAC/零代理、缺口空态+登记诚实，80% 阈值同口径且不静默不造假；同源纪律、趋势同期真实行与降采样留痕、上看板演进位、路由挂载均核实成立；pnpm 601 用例与 typecheck 独立复跑全绿、9 文件全 frontend/** 越界 0；无 Critical/Important，2 条 Minor 记 deferred 不进修订轮，TASK 保持「进行中」待协调者收尾。
