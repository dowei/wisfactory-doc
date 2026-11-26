# TASK-0060 实施报告：指标看板（卡片/同比环比/趋势叠加）· 前端道

- 任务：TASK-0060（协调者已认领，状态保持「进行中」——本任务未置「已完成」）
- 执行者：前端道执行者（no-subagents）
- 执行日期：2026-11-24
- **BASE（开工 HEAD）= `4dd8144`**（`git rev-parse --short HEAD`，主仓 main）；**提交 = `bf84526`**（注明 TASK-0060，`git add` 只含 `frontend/`）
- 范围：`frontend/**`（src 6 + 前端单测 2 + 路由/总览入口），未碰 backend/、calc/、ingest/、契约、workitems 库（本报告唯一例外）
- 结论先行：pnpm 三门 0 error、`pnpm test` 64 文件 601 用例全绿（含本批 10 条纯函数新断言）；范围核对 `git diff --name-only 4dd8144..bf84526` 9 文件全落 `frontend/**`、越界 0。**安全红线成立**：前端零 appKey/secret、零 HMAC/openapi 直连、不自造代理；管理面无指标值/完整度登录态读口→值/完整度/趋势空态 + 缺口登记，不静默、不造假。live 8098 走查因无监听按 brief 登记缺席。详见 §二/§三。

---

## 一、改动文件清单（提交 `bf84526`，9 项，全部 `frontend/**`）

| 文件 | 作用 |
| --- | --- |
| `frontend/src/api/display/display-api.ts`（改） | 新增 `MetricValueRow`（对齐 st_metric_value 冻结契约 4 列 value/yoy/mom/ts + 四元组标签）、`MetricCompletenessRow` 类型；注释登记读口缺口与演进位。**只加类型，不写未交付端点调用** |
| `frontend/src/router/display.ts`（改） | 静态注册 `/display/metrics`（`name=DisplayMetrics`，hideInMenu、需登录，同 0058/0059 范式），挂 OtLayout 下 |
| `frontend/src/router/index.ts`（未改） | 路由接线沿用既有 `displayRouters` push，无需改动 |
| `frontend/src/views/display/metrics/index.vue`（新） | 看板主页：工厂维度过滤（全部/单厂）、按周期分组的卡片网格、趋势区、页面级「管理面指标值读口缺口」告警；指标定义取 `/iot/metric/list` 真数据，值/完整度恒空态；总览页按钮直达入口 |
| `frontend/src/views/display/metrics/metric-card.vue`（新） | 指标卡片：本期值 + 同比/环比涨跌方向色 + 完整度阈值标记 + 上看板标记（ADR-0011）；无值行→「指标数据接入中」空态；卡片值与结果表同数（同源，不现算） |
| `frontend/src/views/display/metrics/metric-trend.vue`（新） | 趋势图：ECharts 多指标叠加 + 去年同期虚线对比、周期切换、降采样常驻标注（ADR-0010）；无数据→空态，不造曲线 |
| `frontend/src/views/display/metrics/metric-utils.ts`（新） | 纯函数：`trendDisplay`（涨绿跌红，租户可配演进位登记 ADR-0011）、`completenessDisplay`（0.8 阈值标记）、`buildMetricTrend`（按 period 过滤/同 ts 去年同期真实行对比/降采样分档标注） |
| `frontend/src/views/display/overview/index.vue`（改） | 总览页头部加「指标看板」按钮（hideInMenu 下提供可发现入口，router.push 命名路由，不新增下钻层级） |
| `frontend/tests/unit/display/metric-utils.spec.ts`（新） | 10 用例：方向色/未知不冒充 0、完整度 80% 阈值与读口缺口、趋势按周期过滤/同期真实行/降采样标注/同 ts 覆盖 |
| `frontend/tests/unit/display/display-routes.spec.ts`（改） | 路由断言由 4 条更新为 5 条（含 `/display/metrics`），段数 [1,3,4,2,2]、需登录/hideInMenu 不混用 /iot/ 保持 |

> 范围核对：`git diff --name-only 4dd8144..bf84526` 输出即上表 9 个文件（含 src、路由与 `tests/unit/display/**` 前端单测），全部在 `frontend/`。`/display/metrics` 通过 router/index.ts 既有 `displayRouters` push 自动生效，故未改 router/index.ts。

## 二、验收命令原始输出

### 1）pnpm 三门 + 单测（全 0 error / 全绿）

```
===== pnpm lint =====
$ eslint .
✖ 9 problems (0 errors, 9 warnings)
（9 条 prettier/prettier warning 全部位于未改动的存量测试文件：
  edit-lock-api.spec.ts / setting-drawer-persist.spec.ts / axios.spec.ts /
  redpack-list-helper.spec.ts / redpack-columns.spec.ts ——与 0058/0059 基线同）
LINT_EXIT=0

===== pnpm typecheck =====
$ vue-tsc --noEmit
TYPECHECK_EXIT=0

===== pnpm test =====
$ vitest run
✓ tests/unit/display/device-utils.spec.ts (8 tests)
✓ tests/unit/display/display-routes.spec.ts (3 tests)
✓ tests/unit/display/metric-utils.spec.ts (10 tests)   ← 本批新增
✓ tests/unit/display/display-slot-card.spec.ts (2 tests)
✓ tests/unit/display/display-tree.spec.ts (5 tests)
 Test Files  64 passed (64)
      Tests  601 passed (601)
TEST_EXIT=0

===== pnpm build:test =====
$ vite build --base=/admin/ --mode test
...（存量 chunk>500kB 警告等均为既有）
✓ built in 1m 5s
BUILD_EXIT=0
```

本批新增文件经 `npx eslint <files> --fix` 后复检：0 error 0 warning（.less 无匹配配置 ignored）；typecheck 全程 0 error。

### 2）live 走查（缺席登记，按 brief「8098 忙则跳过登记」）

```
$ curl -s -o /dev/null -w "HTTP_CODE=%{http_code}\n" --max-time 5 http://127.0.0.1:8098/
HTTP_CODE=000
CURL_EXIT=7
$ netstat -ano | grep ":8098" | grep -i listen
NO_8098_LISTENER
```

结论：8098 无监听（协调者未启动后端），**live 走查（展示端指标看板页→卡片/趋势/完整度标记/空态→全停）缺席待补**。执行者不擅自启动 backend、不跨道触碰。待补项：后端就绪后执行「登录→/display/metrics→指标定义卡片（真数据）/值-完整度-趋势空态与缺口告警→全停」。

### 3）安全红线 + 范围核对（原始输出）

```
===== 安全红线：新指标文件零凭据/零 HMAC 直连代码 =====
$ grep -rnE "appKey|appSecret|createHmac|/openapi/v1/(metrics|items|points)" \
    frontend/src/views/display/metrics/ （排除注释行）
（无命中）
$ grep -rn "getRequest\|fetch(\|axios" frontend/src/views/display/metrics ... | grep -i openapi
NO_DIRECT_OPENAPI_CALL
说明：appKey/secret/HMAC 字样仅出现在文件头注释/页面缺口告警文案中（说明"不持有/不直连"），
无任何真实凭据常量、无 crypto.createHmac、无 /openapi/v1 请求。管理面取数只用 displayApi
封装的 getRequest('/iot/metric/list')（登录态）。

===== 范围核对 =====
$ git diff --name-only 4dd8144..bf84526
（9 文件，全部 frontend/**，详见 §一）
$ git diff --name-only 4dd8144..bf84526 | grep -v '^frontend/'
ZERO_OUT_OF_SCOPE
$ git status --short（提交后）
?? .agents/            ← 开工既存未跟踪目录，本任务未触碰
?? .tmp-t0044v/        ← 开工既存临时目录，本任务未触碰
$ git rev-parse --short HEAD
bf84526
```

## 三、未做项与原因

### 3.1 管理面指标值读口缺口（登记，非本道可补）

| 缺口 | 实证（源码级，非采信报告） | 影响 | 登记处置 |
| --- | --- | --- | --- |
| **管理面无指标值登录态读口** | `backend/.../iot/metric/controller/IotMetricController.java` 仅 `/iot/metric/{list/{id}/add/generate/edit/changeStatus/remove}`，全是**指标定义** CRUD；全仓 GET 映射中无「按 metricCode/period/时间窗取 st_metric_value 值」的登录态端点 | 卡片本期值、同比(yoy)、环比(mom)、趋势曲线无法取真值 | 值恒空态（类型与纯函数已就位），页面告警登记，不自造代理、不造假 |
| **指标值仅在 HMAC 对外接口** | `OpenApiController.java:86 GET /openapi/v1/metrics/query` 是唯一值读口；`OpenApiAuthFilter.java:50 PATH_PREFIX="/openapi/v1/"` 全覆盖 HMAC 鉴权，`@Anonymous` 不进登录态 | 浏览器直连需前端持有 appKey/secret——**安全红线禁止** | 前端零凭据、零直连；缺口登记等裁决（同 0059 纪律） |
| **完整度无物理列、无管理面读口** | 0045 报告 §三-2：st_metric_value 列仅 ts/value/yoy/mom，完整度由内存视图 `/calc/v1/status.metric`（calc 内网观测面，非 ot-admin 登录态）承载；ot-admin 无对应端点 | 低于 80% 完整度标记无法取真值 | 完整度 null→「完整度数据接入中（读口待接入）」，不静默判完整、不造假标记 |

**需要什么端点（供协调者裁决补挂后端任务）**：管理面登录态 `GET /iot/metric/values`（入参 metricCodes + period + start/end，返回 metricCode/ts/value/yoy/mom，复用 `/iot/metric/list` 的数据权限裁剪）与 `GET /iot/metric/completeness`（或把完整度并入前者响应），**不得走 HMAC openapi**。读口到位后只替换 `metrics/index.vue::loadAll` 的取数处（当前保留 `valueRows`/`completenessMap` 演进位），卡片/趋势/纯函数无需改动。归属任务未在报告指定（按 brief「缺口登记等裁决」，指派属协调者权）。

### 3.2 TDengine 补验条件（TD-0053，环境阻断，非本道可解）

- 0045 报告已登记 TDengine REST 192.168.31.145:6041 服务器侧故障（HTTP=000/6s timeout），st_metric_value 真机写入+读回列环境恢复后条件；本任务未触碰 TDengine、未写真库。
- 前端侧真机读回验证条件（环境恢复后）：后端 8098 就绪 + 管理面指标值读口交付（§3.1）+ st_metric_value 有真实写入行 → 执行「登录→/display/metrics→卡片值与结果表逐条比对（同源断言）→完整度标记→趋势同期叠加→全停」。当前这些条件均不具备，故 live 走查与值同源真机验证一并缺席登记。
- 本批不造假数据：未硬编码任何指标值/完整度/趋势点；所有未接数据槽位为空态。

### 3.3 其余未做项

| 项 | 原因 |
| --- | --- |
| 涨跌颜色租户可配 | ADR-0011 明定二期独立展示偏好；本期固定涨绿跌红，演进位在 `metric-utils.trendDisplay` 集中登记 |
| 上看板标记真值 | ADR-0011 标记字段接口未交付；卡片已预留 `mark` prop 与放大/置顶样式，接口交付填值即可，渲染逻辑不动 |
| 趋势「同 ts 去年同期行」对比的真机数据 | 依赖 §3.1 读口 + §3.2 真机数据；纯函数已按真实行口径实现并有单测，真机行缺席时 UI 空态并标注「不造假对比曲线」 |
| TASK 状态 | 按硬约束未置「已完成」，保持「进行中」待评审；未回写 workitems 库（本报告唯一例外） |

## 四、偏离计划之处

1. **live 走查缺席**：brief 验收命令 2 明示「8098 探测，忙则跳过登记」——实测无监听（§二.2），按计划内允许缺席，非执行偏离。
2. **单测文件位置**：brief 改动范围写「frontend/src/** 与路由配置」，本批在 `frontend/tests/unit/display/**` 新增 1 个单测文件并改 1 个既有单测，以覆盖纯函数/路由断言（同 0058/0059 先例）。仍全部在 `frontend/` 内、随 frontend/ 提交，未越出前端道，显式标注。
3. **路由挂载方式**：沿用 0058/0059 的静态路由段（挂 OtLayout、hideInMenu、无需 sys_menu 种子），总览页加按钮提供可发现入口；未改 router/index.ts（既有 displayRouters push 自动生效）。若协调者要求看板走独立菜单/独立布局外壳需另行裁决，不影响功能与验收。
4. **「值与结果表同数」的当前形态**：因管理面读口缺口（§3.1），本批交付的是**同源结构与纯函数**（值行类型逐字对齐冻结契约、同源定位按 metricCode、不现算），真值比对需待读口交付。这不是实现偏离，而是 brief「无读口→空态+登记、不自造代理」纪律的直接落地。
5. **无实质技术偏离**：未改 backend/calc/ingest/契约/workitems、未置 TASK 已完成、未造假数据、未写死链、只 add frontend/。

## 五、遗留问题

1. **管理面指标值/完整度登录态读口（最高优先）**：缺口形态见 §3.1，需协调者裁决补挂后端任务并写验收标准；前端替换面集中在 `metrics/index.vue::loadAll`。
2. **真机值同源与趋势同期走查（环境阻断）**：条件见 §3.2；TDengine 6041 恢复 + 读口交付前，卡片真值/完整度标记/趋势叠加的真机验证标记未做。
3. **租户涨跌色配置、上看板标记真值**：二期演进位，接口/配置交付后接入，当前样式与纯函数已是可扩展结构。
4. **区间范围说明**：BASE `4dd8144` 与提交 `bf84526` 之间仅本任务 1 个提交（前端道独占、无其他前端任务在途）；`git diff 4dd8144..bf84526` 9 文件全 frontend/，无并发混入。
5. **开工既存未跟踪目录**：`.agents/`、`.tmp-t0044v/` 本任务未触碰，提交后仍在，协调者合并时可按需清理。

---

一句话结论：**通过**——pnpm build:test/lint/typecheck 三门 exit 0（lint 0 error，9 warning 全在未改存量文件）、`pnpm test` 64 文件 601 用例全绿（含本批 10 条新断言），`git diff --name-only 4dd8144..bf84526` 9 文件全落 `frontend/**` 越界 0；安全红线经源码级核实成立（前端零凭据/零 HMAC/零 openapi 直连、不自造代理），管理面无指标值/完整度读口按 brief 走空态+缺口登记且缺口形态与端点建议清晰；live 8098 走查因无监听按 brief 登记缺席、真机值同源待读口与 TDengine 环境恢复后补验，均不阻断本批骨架与数据纪律验收，TASK 保持「进行中」待评审。
