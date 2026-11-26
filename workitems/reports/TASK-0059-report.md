# TASK-0059 实施报告（设备详情页——状态区/实时/曲线/来源追溯 · 前端道）

- 任务：TASK-0059（状态保持「进行中」，未回写 workitems 库；本报告为唯一例外写入）
- 执行者：前端道执行者（no-subagents；brief 驱动）
- BASE：`e387c3b`（开工时 main HEAD，`git rev-parse --short HEAD` 实测）
- 本任务提交：`86a1c09`（main，注明 TASK-0059；`git add` 只含 frontend/，13 文件）
- 并行说明：BASE 之后、本任务提交之前，并行 calc 道 TASK-0042 修订轮提交 `01236fd`（2 个 calc 文件）插入主仓历史；本任务未 rebase/改写他人历史，仅按线性历史落自己的提交。`git diff e387c3b..HEAD` 含该 2 个 calc 文件，均非本任务产生。
- 安全红线：前端不持有 appKey/secret、不直连 openapi/v1 HMAC 接口（详见 §3 数据源缺口登记）。

## 一、改动文件清单（13 个，全在 frontend/）

| 文件 | 作用 |
| --- | --- |
| `frontend/src/api/display/display-api.ts` | 展示端数据服务扩展：在 0058 批一基础上新增 9 个管理面端点类型与封装——0050 状态读口 `GET /api/v1/iot/status/current`、`/history`；组件/实时/曲线/追溯数据源 `GET /iot/item/list`、`/iot/point/list`、`/iot/point/{id}/curve`、`/iot/item/{id}/impact`、`/iot/instance/{id}/items/{itemId}/effective-params`、`/iot/metric/list`。全部登录态端点，零 appKey/HMAC、零占位假数据 |
| `frontend/src/router/display.ts` | 新增第 4 条静态路由 `/display/device`（name=DisplayDeviceDetail，hideInMenu、需登录），instanceId 走 query 参数不增路径段，下钻计数保持 3 |
| `frontend/src/views/display/workshop/index.vue` | 车间页设备色块接下钻：`onOpenDevice(dev)` → `DisplayDeviceDetail`，携带 instanceId + factoryId/workshopId（供详情页返回）；禁用设备不跳转 |
| `frontend/src/views/display/device/index.vue` | 设备详情页：四区装配（状态/实时/曲线/追溯）+ 组件清单；一次性并行拉三个清单数据源；「刷新数据」按钮一次刷新全部区（重算后自动刷新的简单方案：手动刷新，见 §四）；无权设备由后端各端点裁剪，前端不伪造 |
| `frontend/src/views/display/device/device-status-zone.vue` | 状态区：大色块用 0050 读口 `statusName`/`color`（不硬编码）+ `changeTime` 持续时长（本地按秒重算、不重复请求）+ 近 24h 状态历史时间线；UNKNOWN/LWT 离线按读口径原样展示；加载失败走「状态数据接入中」空态 |
| `frontend/src/views/display/device/device-realtime-zone.vue` | 实时数据区：当前值/单位/质量标记/更新时间/来源；质量码 1/2 超限标红、3 离线灰显（ADR-0011 TD-0022）；当前值取绑定点位近 24h 曲线末点（管理面端点），公式/汇总类项明确标「需实时计算（数据接入中）」，不造假 |
| `frontend/src/views/display/device/device-curve-zone.vue` | 历史曲线：ECharts 多点位叠加（多选）、时间范围选择（1h/24h/7d/自定义）、超限时刻红线 markLine + 断线时段灰色 markArea（按质量码）；降采样标注常驻图上方；无数据走「数据接入中」空态 |
| `frontend/src/views/display/device/device-trace-zone.vue` | 来源追溯：按数据项折叠展开——有效取值/公式/汇总方式/周期 + 影响预览引用边（OUT 输入项、IN 依赖方）逐条列出 instance/dataCode/关系；汇总子级贡献值与「本次计算输入值」因无读口标「待计算值接口接入」，不写死链、不造假 |
| `frontend/src/views/display/device/device-component-zone.vue` | 组件清单：数据项按 category（数据类别）、点位按 dataType（瞬时/累计/状态/开关）、指标按 period 分组折叠；「查看来源」跳追溯面板定位 |
| `frontend/src/views/display/device/device-utils.ts` | 纯函数：质量码 0-10 语义映射（ADR-0009/0010）、超限红/离线灰派生、按跨度分档降采样（原始/60s/300s/3600s）+ 强制标注（ADR-0010）、时长/时间格式化；未知码按 warn 不冒充正常 |
| `frontend/src/views/display/display.less` | 四区页面样式（状态块/时间线/实时表/曲线/追溯/组件清单） |
| `frontend/tests/unit/display/device-utils.spec.ts` | 8 用例：质量码映射、超限红/离线灰不叠加、降采样分档与标注必现、空输入空态、时长 UNKNOWN 与格式化 |
| `frontend/tests/unit/display/display-routes.spec.ts` | 由 3 用例改 3 用例：四路由清单/命名、下钻≤3（设备详情 query 不增段）、需登录+hideInMenu+不混用 /iot/（旧断言「无设备路由」随 0059 落地更新） |

> 验证命令 4（范围核对）：`git diff-tree --no-commit-id --name-only -r 86a1c09 | grep -v '^frontend/'` 输出为空（ZERO），本任务 13 文件全落 frontend/**（src/** 与前端单测）；并行 calc 0042 的 2 文件未 add、未提交、未触碰。

## 二、验收命令原始输出

### 1）pnpm 三门（全 0 error）

```
===== pnpm build:test =====
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported ...
✓ 2986 modules transformed.
✓ built in 1m 13s
BUILD_EXIT=0
```
（存量告警同 0058：enterprise-employee-list.vue 常量赋值警告、vue3-json-viewer icon.svg 提示、chunk>500kB；本批新增 device 分包均正常产出）

```
===== pnpm lint =====
$ eslint .
✖ 9 problems (0 errors, 9 warnings)
LINT_EXIT=0
```
（9 条 warning 全部位于未改动的存量测试文件，与 0058 基线同款：edit-lock-api.spec.ts / setting-drawer-persist.spec.ts / axios.spec.ts / redpack-list-helper.spec.ts / redpack-columns.spec.ts；本任务新增文件经 `npx eslint --fix` 后复检 0 error 0 warning）

```
===== pnpm typecheck =====
$ vue-tsc --noEmit
TYPECHECK_EXIT=0
```

补充全量单测（brief 测试口径要求同源/质量/降采样/追溯/空态断言）：

```
===== pnpm test =====
$ vitest run
✓ tests/unit/display/device-utils.spec.ts (8 tests)
✓ tests/unit/display/display-routes.spec.ts (3 tests)
✓ tests/unit/display/display-slot-card.spec.ts (2 tests)
✓ tests/unit/display/display-tree.spec.ts (5 tests)
 Test Files  63 passed (63)
      Tests  591 passed (591)
TEST_EXIT=0
```
（基线 62 文件 583 用例 + 本批新增 8 用例；display 子集单测先跑 `npx vitest run tests/unit/display/` → 4 文件 18 用例全绿）

### 2）联调冒烟（8098 空闲，实做；用毕即停）

起前探测：`netstat :8098` 无 LISTENING（`NO_8098_LISTENER`）；jar/DB(3307)/Redis(6379) 均在位。启动 `scripts\dev.cmd start`（PID=93100，8098 LISTENING），冒烟脚本 `/tmp/t0059_smoke.py`（管理面登录态，逐个调用详情页实际消费端点），终轮原始输出（EXIT=0）：

```
OK   | L0 登录 200 http=200 code=200
OK   | L1 实例树非空 root_nodes=9
OK   | L2 找到设备实例(nodeType=4) id=20 code=EQ1A
OK   | L3 状态 current 200 status=UNKNOWN color=None quality=3
OK   | L4 状态 history 200 points=0
OK   | L5 数据项清单 200 items=2
OK   | L6 点位清单 200 points=0
SKIP | L7 点位曲线（无绑定点位）
OK   | L8 引用反查 impact 200 itemId=22
OK   | L9 有效参数/公式 200 itemId=22
OK   | L10 指标清单 200 metrics=0

SMOKE RESULT ok=10 fail=0
SMOKE_EXIT=0
```

补测点曲线端点（补 L7 缺席）：`/tmp/t0059_smoke_curve.py` 取点位列表总 5 条，对 pointId=9001 调 `/iot/point/9001/curve`：

```
POINT_LIST code=200 total=5 sample=[(9001, 'T21_SMOKE_DEPT4'), (9002, 'T21_SMOKE_DEPT2'), (12019, 'T24_P_SMOKE_01'), ...]
CURVE http=200 code=200 pointId=9001 dataPoints=0 sample=[]
CURVE_CHECK_DONE
EXIT=0
```

逐条对应：L3 返回 UNKNOWN/quality=3（0050 无状态空态口径，前端按读口原样展示，不现算）；L4 历史空集（TDengine 环境故障，0050 报告 TD-0053 已登记，前端走「暂无状态变化记录」空态）；L5/L6 清单真数据形状确认；L8/L9 impact/effective-params 200（追溯面板数据源可用）；L7 补测曲线端点 200 + fail-soft 空集（0023 既有口径，前端走「数据接入中」空态，不造假曲线）。

用毕即停：`scripts\dev.cmd stop` → `已停止 PID=93100` → `netstat :8098 LISTENING` 无输出（`PORT_8098_FREE`）。冒烟脚本在 gitignored `/tmp/`，未入库。

### 3）范围核对

```
$ git rev-parse --short HEAD        # 开工 BASE
e387c3b
$ git rev-parse --short HEAD        # 本任务提交后
86a1c09
$ git diff-tree --no-commit-id --name-only -r 86a1c09
（13 文件，详见 §1，全部 frontend/**）
$ git diff-tree --no-commit-id --name-only -r 86a1c09 | grep -v '^frontend/'
0 — ZERO — ALL IN FRONTEND
```

## 三、未做项与原因 / 数据源缺口登记

| 未做项 / 缺口 | 原因（含安全红线证据） | 处置 |
| --- | --- | --- |
| **数据项「实时当前值」专用端点（管理面等价 0051 `openapi/v1/items/realtime`）** | 全仓 grep：管理面只有点位近 24h 曲线 `GET /iot/point/{id}/curve`（0023），无「按设备批量取数据项当前值 v/q/ts」端点；唯一等价端点在 `openapi/v1/items/realtime`（0051 报告 §0：appKey+HMAC 签名，浏览器不可直连） | 按 brief 不持有 appKey、不自造代理：**绑定类项当前值由点位曲线末点派生**（管理面登录态端点）；公式/汇总类项显式「需实时计算（数据接入中）」。**专用端点缺口登记，等裁决** |
| **数据项历史曲线端点（管理面等价 `openapi/v1/items/history`）** | 同上，管理面无按 fullKey 带 interval/agg 的历史值端点；`openapi/v1/items/history` 需 HMAC | 曲线走点位 curve 端点（近 24h 窗口），跨度 >24h 明示「数据源仅提供近 24h，长跨度待历史端点接入」，不静默降采样、不造假；缺口登记等裁决 |
| **汇总子级「贡献值」与公式「本次计算输入值」真值** | 0036 impact/effective-params 只给引用关系与公式配置，不含运行时值；0051 `points/history` 需 HMAC；计算留痕 `iot_calc_log` 归 0063 未交付 | 追溯面板展示引用边（instance/dataCode/关系）+ 公式文本，数值列标「贡献值待计算值接口接入」；不写死链、不造假 |
| **计算留痕（iot_calc_log）入口** | brief 决策 5 明文归 0063 未交付 | 不提供入口（避免死链），追溯面板脚注明示；报告登记 |
| **状态历史真机读回** | 0050 报告 TD-0053：TDengine 192.168.31.145 REST 故障，读口 fail-soft 空集 | 前端按空态展示；非本任务范围，不影响本任务验收 |
| **上看板标记 mark 真值** | 0058 批一已登记接口未交付 | 本批未改动该逻辑（沿用 0058 可选字段），不重复登记 |
| **TASK 置「已完成」** | 硬约束：不置完成、不改 workitems 库（报告除外） | TASK 保持「进行中」，待评审 |

数据纪律声明：四区所有无数据源槽位均为「数据接入中」/「暂无…」空态，**无一处硬编码假数字、无未交付页死链、无 appKey/secret 或其他 HMAC 凭据**。

## 四、偏离计划之处

1. **实时值/曲线数据源口径**：brief 决策 3/4 写「走管理面已有的点位实时值接口（grep 确认 0023/0051 管理侧形态）」并预设 0051 管理侧有封装。grep 实证管理面只有点位近 24h 曲线（0023 `GET /iot/point/{id}/curve`，fail-soft），无数据项实时/历史等价端点；按 brief「管理面无等价端点→报告登记缺口等裁决，不自造代理」执行——绑定类当前值/曲线取点位曲线末点/序列，公式汇总类走空态。**属 brief 明确允许的缺口路径，非越界自造**。
2. **下钻入口形态**：brief 决策 1 写「/display/device?instanceId=（从车间页设备色块点击进入）」，与 0058 批一车间页「设备行只展示不跳转」不同。本批按 brief 决策 1 接上入口（第 3 次点击），未改设备卡片布局与色块样式（只加 click handler），下钻计数仍 ≤3。
3. **重算后自动刷新方案**：brief 决策 7 要求「轮询或手动刷新按钮，选简单方案 javadoc 说明」。本批选手动刷新按钮（详情页头部 + 实时/曲线区各一个）：状态区持续时长仅本地按秒重算（不重复请求），数据变更由用户点「刷新数据」一次性重拉全部区；理由：页面为低频查看型，短轮询会放大后端负载且与 0051「以订阅代替高频轮询」口径相悖，WebSocket 订阅链未交付。javadoc/代码注释已说明（状态区 onMounted 注释、详情页头部按钮）。
4. **单测位置**：brief 改动范围写「frontend/src/** 与路由配置」，本批在 `frontend/tests/unit/display/**` 新增/改 2 个单测文件以覆盖 brief 测试口径（同源/质量/降采样/追溯/空态/下钻）。仍全部位于 frontend/ 内、随 frontend/ 提交，与 0058 批一同款，未越出前端道。
5. **无实质性技术偏离**：未改 backend/calc/ingest/契约/workitems（报告除外），只 add frontend/，未置 TASK 已完成。

## 五、遗留问题

1. **专用实时/历史值端点待裁决**（§3 首两项）：若管理面后续补齐「按设备取数据项实时/历史值」等价端点（或授权临时代理方案），实时区公式/汇总类项、7d/自定义曲线可替换为真数据；当前实现的数据源封装集中在 `display-api.ts`，替换面小、不影响组件布局。
2. **计算留痕（0063）与计算值读口到位后**：追溯面板「贡献值待计算值接口接入」占位可填真值并接留痕入口；本批不写死链。
3. **TDengine 环境恢复后**（0050 TD-0053）：状态历史时间线与点位曲线真机读回可由协调者复跑联调（fail-soft 契约与 SQL 形态已由 0050/0023 单测锁定，前端空态已验证）。
4. **工作区并发未提交项**：`.agents/`、`.tmp-t0044v/`、`.tmp-t0045-design.txt` 属其他任务/环境目录，本任务未触碰/未提交；并行 calc 0042 提交 `01236fd` 与本任务文件集不相交，协调者合并时留意。
5. **TASK 状态**：保持「进行中」待评审，未置「已完成」；未回写 workitems 库（本报告除外）。

---

一句话结论：**通过**——pnpm build:test/lint/typecheck 三门 exit 0（lint 0 error），pnpm test 63 文件 591 用例全绿（含本批 8 条新断言）；`git diff-tree 86a1c09` 13 文件全落 frontend/**；8098 空闲起停各一次，联调冒烟 10/10 + 曲线端点补测 200 全通过、用毕端口复核 FREE；设备详情页四区与组件清单真接入、下钻≤3、状态三来源与 0050 同源、质量标记渲染、曲线降采样标注常驻、追溯展开、空态无假数据、前端零 appKey/HMAC 凭据均成立；数据项专用实时/历史值端点缺失按 brief 登记缺口等裁决，不影响本批验收。
